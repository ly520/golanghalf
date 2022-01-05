# Channel

## channel的数据结构
```go
type hchan struct {
    // 循环队列元素的数量；代表 chan 中已经接收但还没被取走的元素的个数。内建函数 len 可以返回这个字段的值。
	qcount   uint  

    // 循环队列的大小；chan 使用一个循环队列来存放元素，循环队列很适合这种生产者 - 消费者的场景         
	dataqsiz uint           
	buf      unsafe.Pointer // 循环队列的指针；存放元素的循环队列的 buffer。
    
    // chan 中元素的类型和 size。因为 chan 一旦声明，它的元素类型是固定的，即普通类型或者指针类型，所以元素大小也是固定的。
	elemsize uint16 // chan中元素的大小
	elemtype *_type // chan中元素的类型

	closed   uint32 // chan 是否已close
    
    // 处理发送数据的指针在 buf 中的位置。
    // 一旦接收了新的数据，指针就会加上elemsize，移向下一个位置。
    // buf 的总大小是 elemsize 的整数倍，而且 buf 是一个循环列表
	sendx    uint   // send 在buf中的索引

    // 处理接收请求时的指针在 buf 中的位置。
    // 一旦取出数据，此指针会移动到下一个位置。
	recvx    uint   // receive 在buf中的索引

    // chan 是多生产者多消费者的模式，
    // 如果消费者因为没有数据可读而被阻塞了，就会被加入到 recvq 队列中。
	recvq    waitq  // receiver 的等待队列

    // 如果生产者因为 buf 满了而阻塞，会被加入到 sendq 队列中
	sendq    waitq  // sender 的等待队列
   
	lock mutex // 互斥锁 保护所有字段
}
```

## Chan初始化

```go

func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// compiler checks this but be safe.
	if elem.size >= 1<<16 {
		throw("makechan: invalid channel element type")
	}
	if hchanSize%maxAlign != 0 || elem.align > maxAlign {
		throw("makechan: bad alignment")
	}

	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
		// chan的size或者元素的size是0，不必创建buf
		c = (*hchan)(mallocgc(hchanSize, nil, true))

		// Race 检查器使用这个地址来同步
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
        // 元素不是指针，分配一块连续的内存给hchan数据结构和buf
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))

        // hchan数据结构后面紧接着就是buf
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 元素包含指针，那么单独分配buf
		c = new(hchan)
		c.buf = mallocgc(mem, elem, true)
	}

    // 元素大小、类型、容量都记录下来
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	c.dataqsiz = uint(size)
	lockInit(&c.lock, lockRankHchan)

	if debugChan {
		print("makechan: chan=", c, "; elemsize=", elem.size, "; dataqsiz=", size, "\n")
	}
	return c
}

```

## Send
Go 在编译发送数据给 chan 的时候，会把 send 语句转换成 chansend1 函数，
chansend1 函数会调用 chansend

```go

func chansend1(c *hchan, elem unsafe.Pointer) {
	chansend(c, elem, true, getcallerpc())
}

func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
	// 1、如果 chan 是 nil 的话，就把调用者 goroutine park（阻塞休眠），
    // 调用者就永远被阻塞住了，所以，throw("unreachable") 这一行是不可能执行到的代码。
    if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	if debugChan {
		print("chansend: chan=", c, "\n")
	}

	if raceenabled {
		racereadpc(c.raceaddr(), callerpc, funcPC(chansend))
	}

    // 2、如果chan没有被close,并且chan满了，直接返回
    // 当你往一个已经满了的 chan 实例发送数据时，并且不想阻塞当前调用，那么就直接返回。
	if !block && c.closed == 0 && full(c) {
		return false
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock) // 加锁

    // 3、如果 chan 已经被 close 了，再往里面发送数据的话会 panic。
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}

    // 4、如果等待队列中有等待的 receiver，那么这段代码就把它从队列中弹出，然后
    // 直接把数据交给它（通过 memmove(dst, src, t.size)），
    // 而不需要放入到 buf 中，速度可以更快一些

    // 从接收队列中出队一个等待的receiver
	if sg := c.recvq.dequeue(); sg != nil {
        // 找到一个等待的接收者。将值传递给接收器。
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}

    // 5、buf还没满,说明当前没有 receiver，需要把数据放入到 buf 中，放入之后，就成功返回了
	if c.qcount < c.dataqsiz {
		// Space is available in the channel buffer. Enqueue the element to send.
        // 通道缓冲区中有可用空间。使要发送的元素排队。
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			racenotify(c, c.sendx, nil)
		}
		typedmemmove(c.elemtype, qp, ep)
		c.sendx++
		if c.sendx == c.dataqsiz {
			c.sendx = 0
		}
		c.qcount++
		unlock(&c.lock)
		return true
	}

    // 6、buf满了。chansend1不会进入if块里，因为chansend1的block=true
	if !block {
		unlock(&c.lock)
		return false
	}

	// Block on the channel. Some receiver will complete our operation for us.
    // 阻塞channel,一些接收者会帮我们完成操作。
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	c.sendq.enqueue(mysg)
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	KeepAlive(ep)

	// someone woke us up.
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	closed := !mysg.success
	gp.param = nil
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	mysg.c = nil
	releaseSudog(mysg)
	if closed {
		if c.closed == 0 {
			throw("chansend: spurious wakeup")
		}
		panic(plainError("send on closed channel"))
	}
	return true
}

```

## recv 接收

>在处理从 chan 中接收数据时，Go 会把代码转换成 chanrecv1 函数，如果要返回两个返
回值，会转换成 chanrecv2，chanrecv1 函数和 chanrecv2 会调用 chanrecv。

```go
func chanrecv1(c *hchan, elem unsafe.Pointer) {
	chanrecv(c, elem, true)
}

func chanrecv2(c *hchan, elem unsafe.Pointer) (received bool) {
	_, received = chanrecv(c, elem, true)
	return
}

// chanrecv通过通道c接收数据，并将接收到的数据写入ep。
// ep可能为零，在这种情况下，接收到的数据被忽略。
// 如果block==false且没有可用的元素，则返回（false，false）。
// 否则，如果c是闭合的，则为零*ep并返回（true，false）。
// 否则，用元素填充*ep并返回（true，true）。
// 非nil ep必须指向堆或调用方的堆栈。
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {
	if debugChan {
		print("chanrecv: chan=", c, "\n")
	}

    // 1、chan为nil，从 nil chan 中接收（读取、获取）数据时，调用者会被永远阻塞。
	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}

	// Fast path: 在未获得锁的情况下检查非阻塞操作是否失败。.
    // 2、block=false且c为空
	if !block && empty(c) {
		if atomic.Load(&c.closed) == 0 {
			return
		}
		if empty(c) {
			// The channel is irreversibly closed and empty.
			if raceenabled {
				raceacquire(c.raceaddr())
			}
			if ep != nil {
				typedmemclr(c.elemtype, ep)
			}
			return true, false
		}
	}

	var t0 int64
	if blockprofilerate > 0 {
		t0 = cputicks()
	}

	lock(&c.lock)

    // 3、c已经被close,且chan为空empty
    // 如果 chan 已经被 close 了，并且队列中没有缓存的元素，那么返回 true、false。
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}

    // 4、如果sendq队列中有等待发送的sender
    // 这个时候，如果 buf 中有数据，优先从buf 中读取数据，
    // 否则直接从等待队列中弹出一个 sender，把它的数据复制给这个receiver
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}

    // 5、没有等待的sender, buf中有数据
    // 这个是和 chansend 共用一把大锁，所以不会有并发的问题。
    // 如果 buf 有元素，就取出一个元素给 receiver
	if c.qcount > 0 {
		// Receive directly from queue
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			racenotify(c, c.recvx, nil)
		}
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		c.recvx++
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		c.qcount--
		unlock(&c.lock)
		return true, true
	}

    //  buf中没有元素，阻塞
    // 如果没有元素，那么当前的 receiver 就会被阻塞，
    // 直到它从 sender 中接收了数据，或者是 chan 被 close，才返回
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// no sender available: block on this channel.
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	c.recvq.enqueue(mysg)
	// Signal to anyone trying to shrink our stack that we're about
	// to park on a channel. The window between when this G's status
	// changes and when we set gp.activeStackChans is not safe for
	// stack shrinking.
	atomic.Store8(&gp.parkingOnChan, 1)
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)

	// someone woke us up
	if mysg != gp.waiting {
		throw("G waiting list is corrupted")
	}
	gp.waiting = nil
	gp.activeStackChans = false
	if mysg.releasetime > 0 {
		blockevent(mysg.releasetime-t0, 2)
	}
	success := mysg.success
	gp.param = nil
	mysg.c = nil
	releaseSudog(mysg)
	return true, success
}

```

## close

>通过 close 函数，可以把 chan 关闭，编译器会替换成 closechan 方法的调用。

>如果 chan 为 nil，close 会 panic；如果 chan 已经 closed，再次 close 也会 panic。
否则的话，如果 chan 不为 nil，chan 也没有closed，
就把等待队列中的 sender（writer）和 receiver（reader）从队列中全部移除并唤醒

```go

func closechan(c *hchan) {
    // 1、关闭nil的channel，则panic
	if c == nil {
		panic(plainError("close of nil channel"))
	}

    // chan不为nil则加锁
	lock(&c.lock)

    // 2、关闭一个 已经关闭的channel，则panic
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

    // 启用race，gotool跟踪
	if raceenabled {
		callerpc := getcallerpc()
		racewritepc(c.raceaddr(), callerpc, funcPC(closechan))
		racerelease(c.raceaddr())
	}

	c.closed = 1

	var glist gList

	// 释放所有的reader
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}

	// 释放所有的writer (它们会panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = unsafe.Pointer(sg)
		sg.success = false
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}

```