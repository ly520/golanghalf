# SingleFlight

```go 

package singleflight

import "sync"

// 代表一个正在处理的请求，或者已经处理完的请求
type call struct {
	wg  sync.WaitGroup

    // 这个字段代表处理完的值，在waitgroup完成之前只会写一次
    // waitgroup完成之后就读取这个值
	val interface{}
	err error
}

// Group represents a class of work and forms a namespace in which
// units of work can be executed with duplicate suppression.
// group代表一个singleflight对象
type Group struct {
	mu sync.Mutex       // protects m
	m  map[string]*call // lazily initialized
}

// 这个方法执行一个函数，并返回函数执行的结果。你需要提供一个 key，对于同一个 key，在同一时间只有一个在执行，
// 同一个 key 并发的请求会等待。第一个执行的请求返回的结果，就是它的返回结果。
// 函数 fn 是一个无参的函数，返回一个结果或者error，而 Do 方法会返回函数执行的结果或者是 error，
func (g *Group) Do(key string, fn func() (interface{}, error)) (interface{}, error) {
	g.mu.Lock()
	if g.m == nil {
		g.m = make(map[string]*call)
	}

    // 如果已经存在相同的key
	if c, ok := g.m[key]; ok {
		g.mu.Unlock()

        // 等待这个key的第一个请求完成
		c.wg.Wait() 
		return c.val, c.err
	}

    // 第一个请求，创建一个call
	c := new(call)
	c.wg.Add(1)

    // 加入到key map中
	g.m[key] = c
	g.mu.Unlock()

    // 调用方法
	c.val, c.err = fn()
	c.wg.Done()

	g.mu.Lock()
	delete(g.m, key)
	g.mu.Unlock()

	return c.val, c.err
}


```