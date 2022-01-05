# RWMutex 读写锁

```go

type RWMutex struct {
	w           Mutex  // 为 writer 的竞争锁而设计
	writerSem   uint32 // writer信号量,为了阻塞设计的信号量
	readerSem   uint32 // reader信号量,为了阻塞设计的信号量
	readerCount int32  // reader的数量
	readerWait  int32  // 等待writer完成的reader的数量
}

const rwmutexMaxReaders = 1 << 30

```

## 读锁

```go 
/* ***************************************************************

在通过以下方式向并发检测器指示关系之前发生：
-解锁->锁定：readerSem
-解除锁定->解除锁定：readerSem
-RUnlock->Lock:writerSem
这些函数暂时禁用对并发同步事件的处理，以便向并发检测器提供更精确的模型。
例如， RLock中的atomic.AddInt32似乎不应该提供acquire-release语义，
这将错误地同步并发阅读器，从而可能丢失并发。

RLock锁定rw进行读取。
它不应用于递归读取锁定；被阻止的锁调用会使新reader无法获取锁。请参阅有关RWMutex类型的文档。

* **************************************************************/

func (rw *RWMutex) RLock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}
    /* *******************************
     * 对 reader 计数加 1。readerCount可能为负数，这是因为，readerCount 这个字段有双重含义：
     *
     * 1、没有 writer 竞争或持有锁时，readerCount 和我们正常理解的 reader 的计数是一样的；
     * 2、如果有 writer 竞争锁或者持有锁时，那么，readerCount 不仅仅承担着 reader
     *    的计数功能，还能够标识当前是否有 writer 竞争或持有锁，在这种情况下，请求锁的
     *    reader 的处理进入runtime_SemacquireMutex(&rw.readerSem, false, 0)，阻塞等待锁的释放
     */
	if atomic.AddInt32(&rw.readerCount, 1) < 0 {
		// rw.readerCount是负值的时候，意味着此时有writer等待请求锁，因为writer优先
		runtime_SemacquireMutex(&rw.readerSem, false, 0)
	}

	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
	}
}

// RUnlock撤消单个RLock调用；
// 它不会影响其他同时阅读的读者。
// 如果rw在RUnlock入口读取时未锁定，则为运行时错误。
func (rw *RWMutex) RUnlock() {
	if race.Enabled {
		_ = rw.w.state
		race.ReleaseMerge(unsafe.Pointer(&rw.writerSem))
		race.Disable()
	}
    // 要将 Reader 的计数减去 1,
    // 如果它是负值，就表示当前有 writer 竞争锁，在这种情况下，还会调用 rUnlockSlow 方法，检查是不是
    // reader 都释放读锁了，如果读锁都释放了，那么可以唤醒请求写锁的 writer 了。
	if r := atomic.AddInt32(&rw.readerCount, -1); r < 0 {
		// 有等待的writer
		rw.rUnlockSlow(r) 
	}
	if race.Enabled {
		race.Enable()
	}
}

func (rw *RWMutex) rUnlockSlow(r int32) {
	if r+1 == 0 || r+1 == -rwmutexMaxReaders {
		race.Enable()
		throw("sync: RUnlock of unlocked RWMutex")
	}
	// A writer is pending.
	if atomic.AddInt32(&rw.readerWait, -1) == 0 {
		// 最后一个reader了，writer终于有机会获得锁了
		runtime_Semrelease(&rw.writerSem, false, 1)
	}
}

```
> 当一个或者多个 reader 持有锁的时候，竞争锁的 writer 会等待这些 reader 释放完，才可
能持有这把锁。当 writer 请求锁的时候，是无法改变既有的reader 持有锁的现实的，也不会强制这些 reader 释放锁，它的优先权只是限定后来的reader 不要和它抢。

>所以，rUnlockSlow 将持有锁的 reader 计数减少 1 的时候，会检查既有的 reader 是不是
都已经释放了锁，如果都释放了锁，就会唤醒 writer，让 writer 持有锁。

---

## 写锁

>RWMutex 是一个多 writer 多 reader 的读写锁，所以同时可能有多个 writer 和 reader。
那么，为了避免 writer 之间的竞争，RWMutex 就会使用一个 Mutex 来保证 writer 的互斥。

> 一旦一个 writer 获得了内部的互斥锁，就会反转 readerCount 字段，把它从原来的正整
数 readerCount(>=0) 修改为负数（readerCount-rwmutexMaxReaders），让这个字段
保持两个含义（既保存了 reader 的数量，又表示当前有 writer）

```go


func (rw *RWMutex) Lock() {
	if race.Enabled {
		_ = rw.w.state
		race.Disable()
	}

	// 首先解决其他writer竞争问题
	rw.w.Lock()

	// 反转readerCount，告诉reader有writer竞争锁,记录当前活跃的 reader 数量，
    // 所谓活跃的reader，就是指持有读锁还没有释放的那些 reader。
	r := atomic.AddInt32(&rw.readerCount, -rwmutexMaxReaders) + rwmutexMaxReaders

	// 如果当前有活跃的reader持有锁，那么需要等待
    /********************************
     * 如果 readerCount 不是 0，就说明当前有持有读锁的 reader，RWMutex 需要把这个当
     * 前 readerCount 赋值给 readerWait 字段保存下来， 同时，这个 writer 进入
     * 阻塞等待状态。
     * 每当一个 reader 释放读锁的时候（调用 RUnlock 方法时），readerWait 字段就减 1，直
     * 到所有的活跃的 reader 都释放了读锁，才会唤醒这个 writer。
    */
	if r != 0 && atomic.AddInt32(&rw.readerWait, r) != 0 {
		runtime_SemacquireMutex(&rw.writerSem, false, 0)
	}

	if race.Enabled {
		race.Enable()
		race.Acquire(unsafe.Pointer(&rw.readerSem))
		race.Acquire(unsafe.Pointer(&rw.writerSem))
	}
}
```

#### 解写锁

> 当一个 writer 释放锁的时候，它会再次反转 readerCount 字段。可以肯定的是，因为当
前锁由 writer 持有，所以，readerCount 字段是反转过的，并且减去了
rwmutexMaxReaders 这个常数，变成了负数。所以，这里的反转方法就是给它增加
rwmutexMaxReaders 这个常数值。
既然 writer 要释放锁了，那么就需要唤醒之后新来的 reader，不必再阻塞它们了，让它们
开开心心地继续执行就好了。
在 RWMutex 的 Unlock 返回之前，需要把内部的互斥锁释放。释放完毕后，其他的
writer 才可以继续竞争这把锁。

```go

func (rw *RWMutex) Unlock() {
	if race.Enabled {
		_ = rw.w.state
		race.Release(unsafe.Pointer(&rw.readerSem))
		race.Disable()
	}

	// 告诉reader没有活跃的writer了
	r := atomic.AddInt32(&rw.readerCount, rwmutexMaxReaders)
	if r >= rwmutexMaxReaders {
		race.Enable()
		throw("sync: Unlock of unlocked RWMutex")
	}

	// 唤醒阻塞的reader们
	for i := 0; i < int(r); i++ {
		runtime_Semrelease(&rw.readerSem, false, 0)
	}
    
	// Allow other writers to proceed.
	rw.w.Unlock()
	if race.Enabled {
		race.Enable()
	}
}

```


