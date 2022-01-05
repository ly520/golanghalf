# Waitgroup 

```go

 type WaitGroup struct {
    // 避免复制使用的一个技巧，可以告诉vet工具违反了复制使用的规则
	noCopy noCopy

    // 64bit(8bytes)的值分成两段，高32bit是计数值，低32bit是waiter的计数
    // 另外32bit是用作信号量的
    // 因为64bit值的原子操作需要64bit对齐，但是32bit编译器不支持，所以数组中的元素在不同的
    // 总之，会找到对齐的那64bit作为state，其余的32bit做信号量
	state1 [3]uint32
}

// 得到state的地址和信号量的地址
func (wg *WaitGroup) state() (statep *uint64, semap *uint32) {
	if uintptr(unsafe.Pointer(&wg.state1))%8 == 0 {
        // 如果地址是64bit对齐的，数组前两个元素做state，后一个元素做信号量
		return (*uint64)(unsafe.Pointer(&wg.state1)), &wg.state1[2]
	} else {
        // 如果地址是32bit对齐的，数组后两个元素用来做state，它可以用来做64bit的原子操作
		return (*uint64)(unsafe.Pointer(&wg.state1[1])), &wg.state1[0]
	}
}

```
#### 64位系统
>因为对 64 位整数的原子操作要求整数的地址是 64 位对齐的，所以针对 64 位和 32 位环
境的 state 字段的组成是不一样的。

>在 64 位环境下，state1 的第一个元素是 waiter 数，第二个元素是 WaitGroup 的计数
值，第三个元素是信号量。

#### 32位系统
>在 32 位环境下，如果 state1 不是 64 位对齐的地址，那么 state1 的第一个元素是信号
量，后两个元素分别是 waiter 数和计数值。

---

## Add
>Add 方法主要操作的是 state 的计数部分。你可以为
计数值增加一个 delta 值，内部通过原子操作把这个值加到计数值上。需要注意的是，这
个 delta 也可以是个负数，相当于为计数值减去一个值，Done 方法内部其实就是通过
Add(-1) 实现的。

```go
func (wg *WaitGroup) Add(delta int) {
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		if delta < 0 {
			// Synchronize decrements with Wait.
			race.ReleaseMerge(unsafe.Pointer(wg))
		}
		race.Disable()
		defer race.Enable()
	}

    // 高32bit是计数值v，所以把delta左移32，增加到计数上
	state := atomic.AddUint64(statep, uint64(delta)<<32)
	v := int32(state >> 32) // 当前计数值
	w := uint32(state)
	if race.Enabled && delta > 0 && v == int32(delta) {
        // 第一个增量必须与Wait同步。
        // 需要将其建模为读取，因为可以有多个并发工作组。计数器从0转换。

        // 如果计数值v为0并且waiter的数量w不为0，那么state的值就是waiter的数量
        // 将waiter的数量设置为0，因为计数值v也是0,所以它们俩的组合*statep直接设置为0即可。
		race.Read(unsafe.Pointer(semap))
	}
	if v < 0 {
		panic("sync: negative WaitGroup counter")
	}
	if w != 0 && delta > 0 && v == int32(delta) {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	if v > 0 || w == 0 {
		return
	}

    // 当waiters>0时，此goroutine已将计数器设置为0。
    // 现在不可能同时发生状态突变：
    // -Adds不能与等待同时发生，
    // -如果Wait看到counter==0，它不会增加waiters。
    // 还是做一个廉价的健康检查来检测WaitGroup的误用。
	if *statep != state {
		panic("sync: WaitGroup misuse: Add called concurrently with Wait")
	}
	// Reset waiters count to 0.
	*statep = 0
	for ; w != 0; w-- {
		runtime_Semrelease(semap, false, 0)
	}
}

func (wg *WaitGroup) Done() {
	wg.Add(-1)
}
```

## Wait
>Wait 方法的实现逻辑是：不断检查 state 的值。如果其中的计数值变为了 0，那么说明所
有的任务已完成，调用者不必再等待，直接返回。如果计数值大于 0，说明此时还有任务
没完成，那么调用者就变成了等待者，需要加入 waiter 队列，并且阻塞住自己。

```go 
func (wg *WaitGroup) Wait() {
	statep, semap := wg.state()
	if race.Enabled {
		_ = *statep // trigger nil deref early
		race.Disable()
	}
	for {
		state := atomic.LoadUint64(statep)
		v := int32(state >> 32) // 当前计数值
		w := uint32(state) // waiter的数量
		if v == 0 {
			// 如果计数值为0, 调用这个方法的goroutine不必再等待，继续执行它后面的逻辑即可
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}
			return
		}

		// 否则把waiter数量加1。期间可能有并发调用Wait的情况，所以最外层使用了一个for循环
		if atomic.CompareAndSwapUint64(statep, state, state+1) {
			if race.Enabled && w == 0 {
                // 等待必须与第一次添加同步。
                // 需要将其建模为与read-in Add进行写竞争。
                // 因此，我只能为第一个服务员写作，
                // 否则，并发等待将相互竞争。

				race.Write(unsafe.Pointer(semap))
			}

            // 阻塞休眠等待
			runtime_Semacquire(semap) 

			if *statep != 0 {
				panic("sync: WaitGroup is reused before previous Wait has returned")
			}
			if race.Enabled {
				race.Enable()
				race.Acquire(unsafe.Pointer(wg))
			}

            // 被唤醒，不再阻塞，返回
			return
		}
	}
}
```