# Once

``` go
// Once is an object that will perform exactly one action.
//
// Once不可复制
type Once struct {    
    // 它是结构中的第一个，因为它在热路径中使用。
    // 热路径内联在每个调用站点。
    // 在某些体系结构（amd64/386）上，将“完成”放在第一位可以实现更紧凑的指令，
    // 而在其他体系结构上可以实现更少的指令（用于计算偏移量）。
	done uint32
	m    Mutex // 互斥锁
}

// Do在且仅当Do首次被调用一次时调用函数f。换句话说，给定var一次如果once.Do(f)被多次调用，
// 只有第一次调用才会调用f，即使每次调用中f的值不同。要执行每个函数，都需要一个新的Once实例。
// Do用于必须只运行一次的初始化。由于f是niladic，可能需要使用函数文字来捕获Do调用的函数的参数：
// 
// config.once.Do(func() { config.init(filename) })
// 
// 因为在对f的一个调用返回之前，没有对Do的调用返回，所以如果f导致调用Do，它将死锁。
// 如果f panic，Do认为它已经回来；Do的未来调用返回而不调用f。
func (o *Once) Do(f func()) {
	// Note: Here is an incorrect implementation of Do:
	//
	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
	//		f()
	//	}
	//
	// Do guarantees that when it returns, f has finished.
	// This implementation would not implement that guarantee:
	// given two simultaneous calls, the winner of the cas would
	// call f, and the second would return immediately, without
	// waiting for the first's call to f to complete.
	// This is why the slow path falls back to a mutex, and why
	// the atomic.StoreUint32 must be delayed until after f returns.

/* ****************************************************************
 *  注意：以下是Do的不正确实现：
 * 
 * 	//	if atomic.CompareAndSwapUint32(&o.done, 0, 1) {
 * 	//		f()
 * 	//	}
 * 这个实现有一个很大的问题，就是如果参数 f 执行很慢的
 * 话，后续调用 Do 方法的 goroutine 虽然看到 done 已经设置为执行过了，但是获取某些
 * 初始化资源的时候可能会得到空的资源，因为 f 还没有执行完
 * 这就是为什么慢路径返回到互斥体，以及为什么原子路径返回到互斥锁,
 * atomic.StoreUint32必须延迟到f返回之后。
 */

	if atomic.LoadUint32(&o.done) == 0 {
		// Outlined slow-path to allow inlining of the fast-path.
		o.doSlow(f)
	}
}

func (o *Once) doSlow(f func()) {
	o.m.Lock()
	defer o.m.Unlock()

    // 双检查的机制（double-checking），再次判断 o.done 是否为 0，如果为0，
    // 则是第一次执行，执行完毕后，就将 o.done 设置为 1，然后释放锁。
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}

```

>所以，一个正确的 Once 实现要使用一个互斥锁，这样初始化的时候如果有并发的
goroutine，就会进入doSlow 方法。互斥锁的机制保证只有一个 goroutine 进行初始
化，同时利用双检查的机制（double-checking），再次判断 o.done 是否为 0，如果为
0，则是第一次执行，执行完毕后，就将 o.done 设置为 1，然后释放锁。
即使此时有多个 goroutine 同时进入了 doSlow 方法，因为双检查的机制，后续的
goroutine 会看到 o.done 的值为 1，也不会再次执行 f。
这样既保证了并发的 goroutine 会等待 f 完成，而且还不会多次执行 f。


# 自定义Once

``` go 

// 一个功能更加强大的Once
type Once struct {
	m    sync.Mutex
	done uint32
}

// 传入的函数f有返回值error，如果初始化失败，需要返回失败的error
// Do方法会把这个error返回给调用者
func (o *Once) Do(f func() error) error {
	if atomic.LoadUint32(&o.done) == 1 {
		//fast path
		return nil
	}
	return o.slowDo(f)
}

// 如果还没有初始化
func (o *Once) slowDo(f func() error) error {
	o.m.Lock()
	defer o.m.Unlock()

	var err error

	// 双检查，还没有初始化
	if o.done == 0 {
		err = f()

		// 初始化成功才将标记置为已初始化
		if err == nil {
			atomic.StoreUint32(&o.done, 1)
		}
	}
	return err
}

```