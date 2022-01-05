# semaphore 信号量

```go
package semaphore // import "golang.org/x/sync/semaphore"

import (
	"container/list"
	"context"
	"sync"
)

type waiter struct {
	n     int64
	ready chan<- struct{} // 获取信号量时关闭
}

// NewWeighted 为并发访问创建具有给定最大组合权重的新加权信号量。
func NewWeighted(n int64) *Weighted {
	w := &Weighted{size: n}
	return w
}

// Weighted 提供了一种将并发访问绑定到资源的方法。
// 调用者可以请求具有给定权重的访问。
type Weighted struct {
	size    int64 // 最大资源数
	cur     int64 // 当前已被使用的资源
	mu      sync.Mutex // 互斥锁，对字段的保护
	waiters list.List // 等待队列
}

// Acquire获取权重为n的信号量，阻塞直到资源可用或ctx完成。
// 成功时，返回nil。失败时，返回ctx.Err（）并保持信号量不变。
// 如果ctx已经完成，则Acquire仍可能在不阻塞的情况下成功。
func (s *Weighted) Acquire(ctx context.Context, n int64) error {
	s.mu.Lock()
    // fast path, 如果有足够的资源，就不考虑ctx.Done的状态，将cur加上n就返回
	if s.size-s.cur >= n && s.waiters.Len() == 0 {
		s.cur += n
		s.mu.Unlock()
		return nil
	}

    // 如果是不可能完成的任务：请求的资源数大于能提供的最大的资源数
	if n > s.size {
		// Don't make other Acquire calls block on one that's doomed to fail.
        // 不要在一个注定要失败的范围里进行其他Acquire调用。
		s.mu.Unlock()

        // 依赖ctx的状态返回，否则一直等待
		<-ctx.Done()
		return ctx.Err()
	}

    
    // 创建了一个ready chan,以便被通知唤醒
	ready := make(chan struct{})

    // 把调用者加入到等待队列中
	w := waiter{n: n, ready: ready}
	elem := s.waiters.PushBack(w)
	s.mu.Unlock()

    // 等待
	select {
        // context的Done被关闭
	case <-ctx.Done():
		err := ctx.Err()
		s.mu.Lock()
		select {
            // 如果被唤醒了，忽略ctx的状态
		case <-ready:
			// Acquired the semaphore after we were canceled.  Rather than trying to
			// fix up the queue, just pretend we didn't notice the cancelation.
			err = nil
		default:
			isFront := s.waiters.Front() == elem
			s.waiters.Remove(elem)
            // 通知其它的waiters,检查是否有足够的资源
			if isFront && s.size > s.cur {
				s.notifyWaiters()
			}
		}
		s.mu.Unlock()
		return err

    // 被唤醒了
	case <-ready:
		return nil
	}
}

func (s *Weighted) acquireSlow(ctx context.Context, n int64) error{
        // 如果是不可能完成的任务：请求的资源数大于能提供的最大的资源数
	if n > s.size {
		// Don't make other Acquire calls block on one that's doomed to fail.
        // 不要在一个注定要失败的范围里进行其他Acquire调用。
		s.mu.Unlock()

        // 依赖ctx的状态返回，否则一直等待
		<-ctx.Done()
		return ctx.Err()
	}

    
    // 创建了一个ready chan,以便被通知唤醒
	ready := make(chan struct{})

    // 把调用者加入到等待队列中
	w := waiter{n: n, ready: ready}
	elem := s.waiters.PushBack(w)
	s.mu.Unlock()

    // 等待
	select {
        // context的Done被关闭
	case <-ctx.Done():
		err := ctx.Err()
		s.mu.Lock()
		select {
            // 如果被唤醒了，忽略ctx的状态
		case <-ready:
			// Acquired the semaphore after we were canceled.  Rather than trying to
			// fix up the queue, just pretend we didn't notice the cancelation.
			err = nil
		default:
			isFront := s.waiters.Front() == elem
			s.waiters.Remove(elem)
            // 通知其它的waiters,检查是否有足够的资源
			if isFront && s.size > s.cur {
				s.notifyWaiters()
			}
		}
		s.mu.Unlock()
		return err
        
    // 被唤醒了
	case <-ready:
		return nil
	}
}

// TryAcquire acquires the semaphore with a weight of n without blocking.
// On success, returns true. On failure, returns false and leaves the semaphore unchanged.
func (s *Weighted) TryAcquire(n int64) bool {
	s.mu.Lock()
	success := s.size-s.cur >= n && s.waiters.Len() == 0
	if success {
		s.cur += n
	}
	s.mu.Unlock()
	return success
}

// Release 方法将当前计数值减去释放的资源数 n，并唤醒等待队列中的调用者，
// 看是否有足够的资源被获取。
func (s *Weighted) Release(n int64) {
	s.mu.Lock()
	s.cur -= n
	if s.cur < 0 {
		s.mu.Unlock()
		panic("semaphore: released more than held")
	}
	s.notifyWaiters()
	s.mu.Unlock()
}

// notifyWaiters 方法就是逐个检查等待的调用者，如果资源不够，或者是没有等待者了，就返回
func (s *Weighted) notifyWaiters() {
	for {
		next := s.waiters.Front()
		if next == nil {
			break // 没有被阻塞的任务了
		}

		w := next.Value.(waiter)

        //避免饥饿，这里还是按照先入先出的方式处理
		if s.size-s.cur < w.n {
            // 没有足够的token给下一个waiter。我们可以继续（试着找一个要求少一点的waiter），
            // 但工作量太大，可能会导致请求量大的waiter挨饿；
            // 取而代之的是，我们让所有剩下的waiter都被阻塞了。
            // 
            // 考虑一个信号量作为读写锁，用n个token，n个reader和一个writer。每个reader可以Acquire(1)以获得读锁。
            // writer可以Acquire(N) 以获得写入锁，不包括所有reader。
            // 如果我们允许reader在队列中跳到前面，writer就会饿死 --- 每个reader总是有一个可用的token。
			break
		}

		s.cur += w.n
		s.waiters.Remove(next)
		close(w.ready)
	}
}

```

>notifyWaiters 方法是按照先入先出的方式唤醒调用者。当释放 100 个资源的时候，如果
第一个等待者需要 101 个资源，那么，队列中的所有等待者都会继续等待，即使有的等待
者只需要 1 个资源。这样做的目的是避免饥饿，否则的话，资源可能总是被那些请求资源
数小的调用者获取，这样一来，请求资源数巨大的调用者，就没有机会获得资源了