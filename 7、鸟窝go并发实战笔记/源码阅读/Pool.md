# Pool

>每次垃圾回收的时候，Pool 会把 victim 中的对象移除，然后把 local 的数据给 victim，
这样的话，local 就会被清空，而 victim 就像一个垃圾分拣站，里面的东西可能会被当做
垃圾丢弃了，但是里面有用的东西也可能被捡回来重新使用。

>victim 中的元素如果被 Get 取走，那么这个元素就很幸运，因为它又“活”过来了。但
是，如果这个时候 Get 的并发不是很大，元素没有被 Get 取走，那么就会被移除掉，因为
没有别人引用它的话，就会被垃圾回收掉。

```go 
package sync

import (
	"internal/race"
	"runtime"
	"sync/atomic"
	"unsafe"
)

// 1. sync.Pool 本身就是线程安全的，多个 goroutine 可以并发地调用它的方法存取对象；
// 2. sync.Pool 不可在使用之后再复制使用。
type Pool struct {
	noCopy noCopy

    // 垃圾回收时会回收victim，同时将local赋值给victim,清空local
	local     unsafe.Pointer // 当前的可用的元素， 指向 poolLocal 类型的对象
	localSize uintptr        // size of the local array

	victim     unsafe.Pointer // 指向 poolLocal 类型的对象
	victimSize uintptr        // size of victims array

    // 当调用Pool 的 Get 方法从池中获取元素，没有更多的空闲元素可返回时，
    // 就会调用这个 New 方法来创建新的元素。如果你没有设置 New 字段，
    // 没有更多的空闲元素可返回时，Get 方法将返回 nil，表明当前没有可用的元素。
	New func() interface{}
}

// Local per-P Pool appendix.
type poolLocalInternal struct {
    // 代表一个缓存的元素，而且只能由相应的一个 P 存取。
    // 因为一个 P 同时只能执行一个 goroutine，所以不会有并发的问题。
	private interface{} // 只能由相应的P使用。
    
    // 可以由任意的 P 访问，但是只有本地的 P 才能 pushHead/popHead，
    // 其它 P可以 popTail，相当于只有一个本地的 P 作为生产者（Producer），
    // 多个 P 作为消费者（Consumer），它是使用一个 local-free 的 queue 列表实现的。
	shared  poolChain   // Local P can pushHead/popHead; any P can popTail.
}

type poolLocal struct {
	poolLocalInternal

	// Prevents false sharing on widespread platforms with
	// 128 mod (cache line size) = 0 .
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}

// from runtime
func fastrand() uint32

var poolRaceHash [128]uint64

// poolRaceAddr returns an address to use as the synchronization point
// for race detector logic. We don't use the actual pointer stored in x
// directly, for fear of conflicting with other synchronization on that address.
// Instead, we hash the pointer to get an index into poolRaceHash.
// See discussion on golang.org/cl/31589.
func poolRaceAddr(x interface{}) unsafe.Pointer {
	ptr := uintptr((*[2]unsafe.Pointer)(unsafe.Pointer(&x))[1])
	h := uint32((uint64(uint32(ptr)) * 0x85ebca6b) >> 16)
	return unsafe.Pointer(&poolRaceHash[h%uint32(len(poolRaceHash))])
}

// Put adds x to the pool.
/***********************
 * Put 的逻辑：优先设置本地 private，如果 private 字段已经有值了，那么就把此
 * 元素 push 到本地队列中。
*/
func (p *Pool) Put(x interface{}) {
    // nil值直接丢弃
	if x == nil {
		return
	}
	if race.Enabled {
		if fastrand()%4 == 0 {
			// Randomly drop x on floor.
			return
		}
		race.ReleaseMerge(poolRaceAddr(x))
		race.Disable()
	}

    // 固定 P
	l, _ := p.pin()

    // 如果本地private没有值，直接设置这个值即可
	if l.private == nil {
		l.private = x
		x = nil
	}

    // 否则加入到本地队列中
	if x != nil {
		l.shared.pushHead(x)
	}

	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
	}
}

// Get从池中选择任意项，将其从池中删除，并将其返回给调用方。
// Get可以选择忽略该池并将其视为空。
// 调用方不应假定传递给Put的值与Get返回的值之间存在任何关系。
// 如果Get返回nil，而p.New为非nil，则Get返回调用p.New的结果。
func (p *Pool) Get() interface{} {
    /****************************************************************
    * 首先，从本地的 private 字段中获取可用元素，因为没有锁，
    * 获取元素的过程会非常快，如果没有获取到，就尝试从本地的 shared 获取一个，如果还没
    * 有，会使用 getSlow 方法去其它的 shared 中“偷”一个。最后，如果没有获取到，就尝
    * 试使用 New 函数创建一个新的。
    *********************************/
	if race.Enabled {
		race.Disable()
	}

    // 把当前goroutine固定在当前的P上
	l, pid := p.pin()
    
    // 优先从local的private字段取，快速
	x := l.private 

    // 取出对象后需要将其删除
	l.private = nil
    
    // 如果未取到对象
	if x == nil {
        // 从当前的local.shared弹出一个值，注意是从head读取并移除
		x, _ = l.shared.popHead()
		if x == nil {
             // 如果没有，则去偷一个
			x = p.getSlow(pid)
		}
	}

	runtime_procUnpin()
	if race.Enabled {
		race.Enable()
		if x != nil {
			race.Acquire(poolRaceAddr(x))
		}
	}

    // 如果没有获取到，则尝试使用New函数生成一个新的
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}

func (p *Pool) getSlow(pid int) interface{} {
	// See the comment in pin regarding ordering of the loads.
	size := runtime_LoadAcquintptr(&p.localSize) // load-acquire
	locals := p.local                            // load-consume

	// Try to steal one element from other procs.
    // 从其它proc中尝试偷取一个元素
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i+1)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

    // 如果其它proc也没有可用元素，那么尝试从vintim中获取
	size = atomic.LoadUintptr(&p.victimSize)
	if uintptr(pid) >= size {
		return nil
	}
	locals = p.victim
	l := indexLocal(locals, pid)

    // 同样的逻辑，先从vintim中的local private获取
	if x := l.private; x != nil {
		l.private = nil
		return x
	}

    // 从vintim其它proc尝试偷取
	for i := 0; i < int(size); i++ {
		l := indexLocal(locals, (pid+i)%int(size))
		if x, _ := l.shared.popTail(); x != nil {
			return x
		}
	}

    // 如果victim中都没有，则把这个victim标记为空，以后的查找可以快速跳过了
	atomic.StoreUintptr(&p.victimSize, 0)

	return nil
}

// pin pins the current goroutine to P, disables preemption and
// returns poolLocal pool for the P and the P's id.
// Caller must call runtime_procUnpin() when done with the pool.

// pin 方法会将此 goroutine 固定在当前的 P 上，避免查找元素期间被其它的 P 执行。
// 固定的好处就是查找元素期间直接得到跟这个 P 相关的 local。
// 有一点需要注意的是，pin 方法在执行的时候，
// 如果跟这个 P 相关的local 还没有创建，或者运行时 P 的数量被修改了的话，就会新创建 local
func (p *Pool) pin() (*poolLocal, int) {
	pid := runtime_procPin()
	// In pinSlow we store to local and then to localSize, here we load in opposite order.
	// Since we've disabled preemption, GC cannot happen in between.
	// Thus here we must observe local at least as large localSize.
	// We can observe a newer/larger local, it is fine (we must observe its zero-initialized-ness).
	s := runtime_LoadAcquintptr(&p.localSize) // load-acquire
	l := p.local                              // load-consume
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	return p.pinSlow()
}

func (p *Pool) pinSlow() (*poolLocal, int) {
	// Retry under the mutex.
	// Can not lock the mutex while pinned.
	runtime_procUnpin()
	allPoolsMu.Lock()
	defer allPoolsMu.Unlock()
	pid := runtime_procPin()
	// poolCleanup won't be called while we are pinned.
	s := p.localSize
	l := p.local
	if uintptr(pid) < s {
		return indexLocal(l, pid), pid
	}
	if p.local == nil {
		allPools = append(allPools, p)
	}
	// If GOMAXPROCS changes between GCs, we re-allocate the array and lose the old one.
	size := runtime.GOMAXPROCS(0)
	local := make([]poolLocal, size)
	atomic.StorePointer(&p.local, unsafe.Pointer(&local[0])) // store-release
	runtime_StoreReluintptr(&p.localSize, uintptr(size))     // store-release
	return &local[pid], pid
}

func poolCleanup() {

    /* *************************************************************
    * 在垃圾收集开始时，当世界停止时调用此函数。
    * 它不能分配，也可能不应该调用任何运行时函数。
    * 因为世界已停止，所以池用户不能位于固定部分中（实际上，这已将所有Ps固定）。
    * */
    // 删除所有池中的victim
    // 删除当前victim, STW所以不用加锁
	for _, p := range oldPools {
		p.victim = nil
		p.victimSize = 0
	}

	// 将local赋值给victim，清空local
	for _, p := range allPools {
		p.victim = p.local
		p.victimSize = p.localSize
		p.local = nil
		p.localSize = 0
	}

	// 将现有池赋值给oldPools,并清空现有池
	oldPools, allPools = allPools, nil
}

var (
	allPoolsMu Mutex

	// allPools is the set of pools that have non-empty primary
	// caches. Protected by either 1) allPoolsMu and pinning or 2)
	// STW.
	allPools []*Pool

	// oldPools is the set of pools that may have non-empty victim
	// caches. Protected by STW.
	oldPools []*Pool
)

func init() {
	runtime_registerPoolCleanup(poolCleanup)
}

func indexLocal(l unsafe.Pointer, i int) *poolLocal {
	lp := unsafe.Pointer(uintptr(l) + uintptr(i)*unsafe.Sizeof(poolLocal{}))
	return (*poolLocal)(lp)
}

// Implemented in runtime.
func runtime_registerPoolCleanup(cleanup func())
func runtime_procPin() int
func runtime_procUnpin()

// The below are implemented in runtime/internal/atomic and the
// compiler also knows to intrinsify the symbol we linkname into this
// package.

//go:linkname runtime_LoadAcquintptr runtime/internal/atomic.LoadAcquintptr
func runtime_LoadAcquintptr(ptr *uintptr) uintptr

//go:linkname runtime_StoreReluintptr runtime/internal/atomic.StoreReluintptr
func runtime_StoreReluintptr(ptr *uintptr, val uintptr) uintptr


```