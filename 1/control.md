# 控制结构

## If
go语言的if分支和其他语言基本一致，其语法为：
```  
//单条件判断
if condition {
    //do something
}


//多条件判断
if condition {

} else if condition {
    //do something
} else {
    //do something

}
````
```go 
 func main() {
	b := true
	if b {
		fmt.Println("true")
	} else {
		fmt.Println("false")
	}
}
```

!> go 的独特指出在于if条件可以跟一个语句，然后再判断。go语言函数支持多返回值，而我们经常要判断函数的执行是否成功、是否返回了错误，这将非常有用，大大提升了可读性。

```
if statement;condition{
    //do something
}

```
```go
func main() {
	if ok, _ := cond(); ok {
		fmt.Println("true")
	} else {
		fmt.Println("false")
	}
}

func cond() (bool, string) {
	return true, "golang"
}

```

## for
go语言仅支持for循环，有多重表现形式：
* for {}：无限循环
* for i:=0; i < n; i++ {} :普通递增的有限循环
* for k,v:= range container : for ...range循环又来遍历slice、map和数组,字符串也可以遍历
* break 跳出循环
* continue 结束当前循环，继续下一次循环

 ```go
 
func main() {
	// 1、循环变量
	for i := 0; i < 10; i++ {
		if i == 2 {
			continue
		}

		if i == 9 {
			break
		}
	}
	// 2、for range
	// slice
	a := []int{1, 3, 4, 5, 6, 7, 8}
	for i := range a {
		fmt.Println(i)
	}

	// map
	b := map[int]string{
		1: "hello",
		2: "world",
		3: "golang",
	}
	for k, v := range b {
		fmt.Println(k, v)
	}

	// 3、无限循环
	for {
		fmt.Println("无限")
	}
}
 ```
!> `range`循环会复制对象，因此不建议直接遍历数组，改而使用slice，或集合的指针：

 ``` go
 func main() {
	a := [3]int{1, 2, 3}

	for i, v := range a {
		if i == 0 {
			a[1], a[2] = 500, 600
			fmt.Println(a)
		}
		a[i] = v + 100 // v 是从a中复制出来的数据，因此上面的更改不影响v的值，所以最终输出为100,101,102
	}
	fmt.Println(a)
}

```

 输出：

``` bash
	[1 222 333]
	[101 102 103]
```

## switch
go语言的`switch`语句和其他语言几乎一致，可以像`java`、`C#`一样工作，不同之处在于：
* 不需要`break`: `go`语言的`switch`中的`case`是自带`break`的，我们不需要编写`break`，编译器会帮我们完成这个工作。
* 花括号：`case`语句块是可以带上花括号的，而且这里的花括号可以独立一行
* `case`表达式：`go`语言`case`是支持表达式的
* 多表达式：`case`表达式支持多表达式,以逗号分隔
* `fallthrough`：`fallthrough`语句用于标明执行完当前`case`语句之后按顺序执行下一个`case` 语句。`fallthrough` 必须是 `case` 语句块中的最后一条语句。如果它出现在语句块的中间，编译器将会报错：fallthrough statement out of place。
* 无表达式：`switch`表达式不是必须的，可以不写，等同于 `switch true`

```go
package main

import (  
    "fmt"
)

func main() {
	i := 2
	switch i {
	case 1:
		fmt.Println("Thumb")
	case 2, 4, 5: // 多表达式
		fmt.Println("Index")
		fallthrough
	case 3:
		{
			fmt.Println("Middle")
		}
	default:
		fmt.Println("default")
	}

}
```
无switch表达式：
```go 
package main

import (  
    "fmt"
)

func main() {  
    num := 75
    switch { // switch 无表达式
    case num >= 0 && num <= 50:
        fmt.Println("num is greater than 0 and less than 50")
    case num >= 51 && num <= 100:
        fmt.Println("num is greater than 51 and less than 100")
    case num >= 101:
        fmt.Println("num is greater than 100")
    }
}
```
## type switch
`Type Switch` 是 `Go` 语言中一种特殊的 `switch `语句，它比较的是类型而不是具体的值。它判断接口变量的类型，然后根据具体类型再做相应处理。

!> `Type Switch` 语句的 `case` 子句中不能使用`fallthrough`。类型断言语法：`变量名.(type)`

基础语法如下：
```go 
switch x.(type) {
case Type1:
     // todo
case Type2:
     // todo
default:
     // todo
}
```
判断接口的类型：
```go
package main

import (
	"fmt"
)

type Animal interface {
	shout() string
}

type Dog struct{}

func (self Dog) shout() string {
	return fmt.Sprintf("wang wang")
}

type Cat struct{}

func (self Cat) shout() string {
	return fmt.Sprintf("miao miao")
}

func main() {
	var animal Animal = Dog{}

	switch animal.(type) {
	case Dog:
		fmt.Println("animal'type is Dog")
	case Cat:
		fmt.Println("animal'type is Cat")
	default:
		fmt.Println("animal'type is unkown")

	}
}

```
### switch表达式也可以声明变量,这将十分方便的使用接口的值

```go
package main

import (
	"fmt"
)

type Animal interface {
	shout() string
}

type Dog struct {
	name string
}

func (d Dog) shout() string {
	return fmt.Sprintf("wang wang")
}

type Cat struct{}

func (c Cat) shout() string {
	return fmt.Sprintf("miao miao")
}

type Horse struct{}

func (h Horse) shout() string {
	return fmt.Sprintf("yu yu ~")
}

func main() {
	var animal Animal = Dog{name: "kitty"}

	switch a := animal.(type) {
	case Dog:
		fmt.Println(a.name)
	case Cat:
		fmt.Println("Cat")
	case Horse:
		fmt.Println("Horse")
	default:
		fmt.Println("unkown")

	}
}

```

## goto
goto和C++中的用法完全一致
```go 
func main() {
	var n int = 30
	fmt.Println("ok1")
	if n > 20 {
		goto label
	}
	fmt.Println("ok2")

label:
	{
		fmt.Println("label")
		fmt.Println("label2")
		fmt.Println("label3")
	}
}
```

## select
select 专用于等待一个或者多个channel的输出,以简化channel的操作。select会监听case语句中channel的读写操作，当case中channel读写操作为非阻塞状态（即能读写）时，将会触发相应的动作。 

```
* 只用于`channel`： `select`中的`case`语句必须是一个`channel`操作
* `default`不阻塞： select中的default子句总是可运行的，即不会阻塞，通常用于超时处理。
* 随机执行一个：有多个case都可以运行时，`select`会随机公平地选出一个执行，其他不会执行。
* 执行`default`： 没有可运行的`case`语句时，且有`default`语句，那么就会执行`default`的动作。
* `select`将阻塞：没有可运行的`case`语句时，且没有`default`语句，`select`将阻塞，直到某个`case`通信可以运行
```

#### 基本用法
```go
    // select基本用法
    select {
    case <- chan1:
    // 如果chan1成功读到数据，则进行该case处理语句
    case chan2 <- 1:
    // 如果成功向chan2写入数据，则进行该case处理语句
    case <-time.After(2 * time.Second):
    // 两秒超时
    default:
    // 如果上面都没有成功，则进入default处理流程
```
#### 官方解释
> 1.For all the cases in the statement, the channel operands of receive operations and the channel and right-hand-side expressions of send statements are evaluated exactly once, in source order, upon entering the "select" statement. The result is a set of channels to receive from or send to, and the corresponding values to send. Any side effects in that evaluation will occur irrespective of which (if any) communication operation is selected to proceed. Expressions on the left-hand side of a RecvStmt with a short variable declaration or assignment are not yet evaluated.

>1、所有channel表达式都会被求值、所有被发送的表达式都会被求值。求值顺序：自上而下、从左到右.
结果是选择一个发送或接收的channel，无论选择哪一个case进行操作，表达式都会被执行。RecvStmt左侧短变量声明或赋值未被评估。

>If one or more of the communications can proceed, a single one that can proceed is chosen via a uniform pseudo-random selection. Otherwise, if there is a default case, that case is chosen. If there is no default case, the "select" statement blocks until at least one of the communications can proceed.

>2、如果有一个或多个IO操作可以完成，则Go运行时系统会随机的选择一个执行，否则的话，如果有default分支，则执行default分支语句，如果连default都没有，则select语句会一直阻塞，直到至少有一个IO操作可以进行.

> 3.Unless the selected case is the default case, the respective communication operation is executed.

>3、除非所选择的情况是默认情况，否则执行相应的通信操作。

>4.If the selected case is a RecvStmt with a short variable declaration or an assignment, the left-hand side expressions are evaluated and the received value (or values) are assigned.

>4、如果所选case是具有短变量声明或赋值的RecvStmt，则评估左侧表达式并分配接收值（或多个值）。

>5.The statement list of the selected case is executed.

>5、执行所选case中的语句


用例：主goroutine等待子goroutine完成，但是子goroutine无限运行，导致主goroutine会一直等待下去。而主线程想假如超过了一定的时没有返回的话，进行超时判断然后继续运行下去。
```go
package main

import (
	"fmt"
	"time"
)

var ch chan int = make(chan int, 1)

// 写入channel
func write() {
	time.Sleep(1 * time.Second)
	ch <- 1
}

// 读取channel
func read() {
	select {
	case ch1 := <-ch:
		fmt.Println(ch1)
		return
	case <-time.After(2 * time.Second):
		fmt.Println("read time out")
		return
	}
}

func main() {
	go write()
	read()
}

```