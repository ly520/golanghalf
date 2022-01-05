# Mutex的实现

```go

// 互斥锁
type Mutex struct {
    /* 
     * 状态信息，该装态字段是一个复合字段，具有多个意义，即：
     * 1、mutexLocked: 持有锁标记
     * 2、mutexWoken：唤醒标记
     * 3、mutexStarving：饥饿标记
     * 4、mutexWaiterShift：阻塞的waiter数量
     * */
	state int32  
	sema  uint32
}

type Locker interface {
	Lock()
	Unlock()
}

const (
	mutexLocked = 1 << iota // 持有锁标记
	mutexWoken // 唤醒标记
	mutexStarving // 饥饿标记, 如果 waiter 获取不到锁的时间超过阈值 1 毫秒，那么，这个 Mutex 就进入到了饥饿模式。
	mutexWaiterShift = iota // 阻塞的waiter数量

    /* 
    * 互斥公平性
    * 互斥可以有两种操作模式：正常和饥饿。
    * 
    * 在正常模式下，等待的协程按FIFO顺序排队，但醒来的协程并不拥有互斥锁，而是与新到达的goroutines争夺锁的所有权。
    * 新到的goroutines有一个优势——它们已经在CPU上运行，而且可能有很多，所以一个被唤醒的协程很有可能无法获得锁。
    * 在这种情况下，它在等待队列的前面排队。如果服务生在超过1ms的时间内无法获取互斥锁，它会将互斥锁切换到饥饿模式。
    * 在饥饿模式下，互斥锁的所有权直接从解锁的goroutine传递给队列前面的协程。
    * 新到达的goroutine不会尝试获取互斥锁，即使它看起来已解锁，也不会尝试旋转。相反，他们将自己排在等待队列的末尾。
    * 
    * 如果协程接收到互斥锁的所有权，并看到（1）它是队列中的最后一个服务生，或（2）它等待的时间少于1毫秒，
    * 它会将互斥体切换回正常操作模式。
    * 
    * 正常模式具有相当好的性能，因为goroutine可以连续多次获取互斥，即使有阻塞的侍者。
    * 
    * 饥饿模式对于预防尾潜伏期的病理病例非常重要。
    */
	starvationThresholdNs = 1e6
)
```
## 1、加锁

### 正常模式和饥饿模式
Mutex 可能处于两种操作模式下：正常模式和饥饿模式。
> 接下来我们分析一下 Mutex 对饥饿模式和正常模式的处理。
请求锁时调用的 Lock 方法中一开始是 fast path，这是一个幸运的场景，当前的
goroutine 幸运地获得了锁，没有竞争，直接返回，否则就进入了 lockSlow 方法。这样
的设计，方便编译器对 Lock 方法进行内联，你也可以在程序开发中应用这个技巧。

> 正常模式下，waiter 都是进入先入先出队列，被唤醒的 waiter 并不会直接持有锁，而是要
和新来的 goroutine 进行竞争。新来的 goroutine 有先天的优势，它们正在 CPU 中运
行，可能它们的数量还不少，所以，在高并发情况下，被唤醒的 waiter 可能比较悲剧地获
取不到锁，这时，它会被插入到队列的前面。如果 waiter 获取不到锁的时间超过阈值 1 毫
秒，那么，这个 Mutex 就进入到了饥饿模式。

如果拥有 Mutex 的 waiter 发现下面两种情况的其中之一，它就会把这个 Mutex 转换成正
常模式:
 >此 waiter 已经是队列中的最后一个 waiter 了，没有其它的等待锁的 goroutine 了；

 >此 waiter 的等待时间小于 1 毫秒。

正常模式拥有更好的性能，因为即使有等待抢锁的 waiter，goroutine 也可以连续多次获
取到锁。
饥饿模式是对公平性和性能的一种平衡，它避免了某些 goroutine 长时间的等待锁。在饥
饿模式下，优先对待的是那些一直在等待的 waiter。

```go
// 加锁
// 如果锁已经在使用中，则调用goroutine会阻塞，直到互斥锁可用为止。

func (m *Mutex) Lock() {
	// Fast path: 幸运之路，一下就获取到了锁
	if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
		if race.Enabled {
			race.Acquire(unsafe.Pointer(m))
		}
		return
	}
	// Slow path：缓慢之路，尝试自旋竞争或饥饿状态下饥饿goroutine竞争
	m.lockSlow()
}

func (m *Mutex) lockSlow() {
	var waitStartTime int64 // 记录此 goroutine 请求锁的初始时间
	starving := false // 此goroutine的饥饿标记
	awoke := false // 唤醒标记
	iter := 0 // 自旋次数
	old := m.state // 当前的锁的状态
	for {
        // 对正常模式的处理：
		// 锁是非饥饿状态，锁还没被释放，尝试自旋
		if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // 自旋
            // 尝试设置mutexWoken标志，以通知Unlock不要唤醒其他被阻止的goroutine。
			if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
				atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
				awoke = true
			}
			runtime_doSpin()
			iter++
			old = m.state // 再次获取锁的状态，之后会检查是否锁被释放
			continue
		}
		new := old

		// 不获取饥饿的互斥锁，新到达的goroutine必须排队。

        // 非饥饿状态下抢锁。把 state 的锁的那一位，置为加锁状态，
        // 后续 CAS 如果成功就可能获取到了锁。
		if old&mutexStarving == 0 {
			new |= mutexLocked // 非饥饿状态，加锁
		}

        // 饥饿状态
        // 如果锁已经被持有或者锁处于饥饿状态，我们最好的归宿就是等待，
        // 所以 waiter 的数量加 1
		if old&(mutexLocked|mutexStarving) != 0 {
			new += 1 << mutexWaiterShift // waiter数量加1
		}

        // 如果此 goroutine 已经处在饥饿状态，并且锁还被持有，那么，
        // 我们需要把此 Mutex 设置为饥饿状态。
		if starving && old&mutexLocked != 0 {
			new |= mutexStarving  // 设置饥饿状态
		}

        // 清除 mutexWoken 标记，因为不管是获得了锁还是进入休眠，
        // 我们都需要清除 mutexWoken 标记
		if awoke {
            // goroutine已从睡眠中唤醒，因此无论哪种情况，我们都需要重置标志。
			if new&mutexWoken == 0 {
				throw("sync: inconsistent mutex state")
			}
			new &^= mutexWoken // 新状态清除唤醒标记
		}

        // 尝试成功设置新状态
		if atomic.CompareAndSwapInt32(&m.state, old, new) {
            // 原来锁的状态已释放，并且不是饥饿状态，正常请求到了锁，返回
			if old&(mutexLocked|mutexStarving) == 0 {
				break // 用CAS锁定互斥锁
			}

            // 处理饥饿状态
            // 如果以前就在队列里面，加入到队列头

            // 判断是否第一次加入到 waiter 队列。一开始不对waitStartTime初始化，
            // 就是为了在这里用来做条件判断: waitStartTime == 0 则表示第一次
			queueLifo := waitStartTime != 0
			if waitStartTime == 0 {
				waitStartTime = runtime_nanotime()
			}

            // 阻塞等待
            // 将此 waiter 加入到队列，如果是首次，加入到队尾，先进先出。
            // 如果不是首次，那么加入到队首，这样等待最久的 goroutine 优先能够获取到锁。此 goroutine 会进行休眠.
			runtime_SemacquireMutex(&m.sema, queueLifo, 1)

            // 唤醒之后检查锁是否应该处于饥饿状态
            // 行判断此 goroutine 是否处于饥饿状态。注意，执行这一句的时候，它已经被唤醒了
			starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
			old = m.state

            // 如果锁已经处于饥饿状态，直接抢到锁，返回
			if old&mutexStarving != 0 {
                // 如果这个goroutine被唤醒并且互斥锁处于饥饿模式，
                // 所有权已移交给我们，但互斥锁处于某种不一致的状态：
                // 未设置mutexLocked，我们仍被视为等待中。解决这个问题。
				if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
					throw("sync: inconsistent mutex state")
				}

                // 有点绕，加锁并且将waiter数减1
                // 设置一个标志，这个标志稍后会用来加锁，而且还会将 waiter 数减 1。
				delta := int32(mutexLocked - 1<<mutexWaiterShift)
				if !starving || old>>mutexWaiterShift == 1 {
                    // 退出饥饿模式。
                    // 关键是在这里做，并考虑等待时间。
                    // 饥饿模式非常低效，两个goroutine一旦将互斥切换到饥饿模式，就可以无限锁定。

                    // 设置标志delta，在没有其它的 waiter 或者此 goroutine 等待还没超过 1 毫秒，
                    // 则会将 Mutex 转为正常状态
					delta -= mutexStarving // 最后一个waiter或者已经不饥饿了
				}
                // 将这个标识delta 应用到 state 字段上。
				atomic.AddInt32(&m.state, delta)
				break
			}
			awoke = true
			iter = 0
		} else {
			old = m.state
		}
	}

	if race.Enabled {
		race.Acquire(unsafe.Pointer(m))
	}
}

```

## 2、解锁

```go

func (m *Mutex) Unlock() {
	if race.Enabled {
		_ = m.state
		race.Release(unsafe.Pointer(m))
	}

	// Fast path: drop lock bit.
	new := atomic.AddInt32(&m.state, -mutexLocked)  //去掉锁标志
	if new != 0 {
		// Outlined slow path to allow inlining the fast path.
		// To hide unlockSlow during tracing we skip one extra frame when tracing GoUnblock.

        // 勾勒出慢路径以允许内联快速路径。
        // 为了在跟踪过程中隐藏unlockSlow，我们在跟踪GoUnblock时跳过了一个额外的帧。
		m.unlockSlow(new)
	}
}

func (m *Mutex) unlockSlow(new int32) {
    // 原本就没有加锁
	if (new+mutexLocked)&mutexLocked == 0 {
		throw("sync: unlock of unlocked mutex")
	}
	if new&mutexStarving == 0 {
		old := new
		for {
            // 如果 Mutex 处于正常状态，如果没有 waiter，或者已经有在处理的情况了，
            // 那么释放就好，不做额外的处理
			if old>>mutexWaiterShift == 0 || old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
				return
			}
			// 唤醒一个协程
            // 否则，waiter 数减 1，mutexWoken 标志设置上，通过 CAS 更新 state 的值
			new = (old - 1<<mutexWaiterShift) | mutexWoken
			if atomic.CompareAndSwapInt32(&m.state, old, new) {
				runtime_Semrelease(&m.sema, false, 1)
				return
			}
			old = m.state
		}
	} else {
        // 饥饿模式：将互斥锁的所有权移交给下一个等待的协程，并给出我们的时间片，
        // 以便下一个等待的协程可以立即开始运行。
        // 注意：mutexLocked未设置，等待的协程将在醒来后设置。
        // 但是如果设置了mutexStarving，互斥锁仍然被认为是锁定的，所以新的goroutines将不会获得它。

        // 如果 Mutex 处于饥饿状态，则直接唤醒等待队列中的 waiter。
		runtime_Semrelease(&m.sema, true, 1)
	}
}

```