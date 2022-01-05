# Cond

``` go 

type Cond struct {
	noCopy noCopy

	// 锁，在wait时需要加锁
	L Locker

	notify  notifyList // 等待队列
	checker copyChecker
}

func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}

// Wait atomically unlocks c.L and suspends execution
// of the calling goroutine. After later resuming execution,
// Wait locks c.L before returning. Unlike in other systems,
// Wait cannot return unless awoken by Broadcast or Signal.
//
// Because c.L is not locked when Wait first resumes, the caller
// typically cannot assume that the condition is true when
// Wait returns. Instead, the caller should Wait in a loop:
//
//    c.L.Lock()
//    for !condition() {
//        c.Wait()
//    }
//    ... make use of condition ...
//    c.L.Unlock()
//

// 会把调用者 Caller 放入 Cond 的等待队列中并阻塞，
// 直到被 Signal 或者Broadcast 的方法从等待队列中移除并唤醒。
// 
// 调用 Wait 方法时必须要持有 c.L 的锁。
func (c *Cond) Wait() {
	c.checker.check()

    // 增加到等待队列中
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()

    // 阻塞休眠直到被唤醒
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}

// Signal wakes one goroutine waiting on c, if there is any.
//
// It is allowed but not required for the caller to hold c.L
// during the call.

// 允许调用者 Caller 唤醒一个等待此 Cond 的 goroutine。如果此时没有等待的 goroutine，
// 显然无需通知 waiter；如果 Cond 等待队列中有一个或者多个等待的goroutine，
// 则需要从等待队列中移除第一个 goroutine 并把它唤醒。
// 在其他编程语言中，比如 Java 语言中，Signal 方法也被叫做 notify 方法。
func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}

// Broadcast wakes all goroutines waiting on c.
//
// It is allowed but not required for the caller to hold c.L
// during the call.

// 允许调用者 Caller 唤醒所有等待此 Cond 的 goroutine。如果此时没有等待的 goroutine，
// 显然无需通知 waiter；
// 如果 Cond 等待队列中有一个或者多个等待的goroutine，则清空所有等待的 goroutine，并全部唤醒。
// 在其他编程语言中，比如 Java语言中，Broadcast 方法也被叫做 notifyAll 方法。
func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}

// copyChecker 是一个辅助结构，可以在运行时检查 Cond 是否被复制使用。
type copyChecker uintptr

func (c *copyChecker) check() {
	if uintptr(*c) != uintptr(unsafe.Pointer(c)) &&
		!atomic.CompareAndSwapUintptr((*uintptr)(c), 0, uintptr(unsafe.Pointer(c))) &&
		uintptr(*c) != uintptr(unsafe.Pointer(c)) {
		panic("sync.Cond is copied")
	}
}

```