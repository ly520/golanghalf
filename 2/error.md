# 错误处理

## 1、错误处理
go语言采用和C语言类似的以返回值来处理错误，得益于go语言的多返回值，错误处理也很简单。

go语言定义了一个类型`error`，这是一个接口，其定义如下：
```go
// The error built-in interface type is the conventional interface for
// representing an error condition, with the nil value representing no error.
/**
 翻译：
 error内置接口类型是表示错误条件的常规接口，nil值表示无错误。
*/
type error interface {
	Error() string
}
```
我们在定义函数的时候，通常会为函数增加一个`error`类型的返回值，以处理程序中的错误：
```go
package main

import (
	"errors"
	"fmt"
)

func main() {
	if result, err := sub(1, 2); err != nil {
		fmt.Println(result, err)
	} else {
		fmt.Println("差：", result)
	}

}

func sub(a, b int) (result int, err error) {
	result = a - b
	if result < 0 {
		err = errors.New("结果小于零")
	}

	return result, err
}
```

我们可以定义一个结构体，然后实现`error`接口，一次来实现自己的错误类型,下面代码由官方示例而来：
```go
    package errors_test

    import (
        "fmt"
        "time"
    )

    // MyError 实现error接口
    type MyError struct {
        When time.Time
        What string
    }

    // 实现error接口的Error方法
    func (e MyError) Error() string {
        return fmt.Sprintf("%v: %v", e.When, e.What)
    }

    func oops() error {
        // MyError实现了error接口，所以能够直接返回。
        return MyError{
            time.Date(1989, 3, 15, 22, 30, 0, 0, time.UTC),
            "the file system has gone away",
        }
    }

    func main() {
        if err := oops(); err != nil {
            switch t := err.(type) {
            case MyError:
                fmt.Println("MyError来了：", t)
            default:
                fmt.Println("default error")
            }
        }
    }
```
输出：
```bash
MyError来了： 1989-03-15 22:30:00 +0000 UTC: the file system has gone away
```

内建的errors包：
|函数|说明|
|-------|------|
|New(text string) error|新建一个`error`|
|Unwrap(err error) error|如果`err`的类型包含返回错误的`Unwrap`方法，则`Unwrap`返回调用`err`上的`Unwrap`方法的结果。否则，`Unwrap`返回`nil`。即：获得一个嵌套的`error`，`Unwrap`能够获得最外层的`error`|
|Is(err, target error) bool |判断`error`是否包含其中：1、如果`err`和`target`是同一个，那么返回`true`；2、如果`err` 是一个`wrap error`,`target`也包含在这个嵌套`error`链中的话，那么也返回`true`。即：包含或相等都返回`true`。|
|As(err error, target interface{}) bool|将`error`类型转化为目标类型，`As`会遍历`err`嵌套链，从里面找到类型符合的`error`，然后把这个`error`赋予`target`,这样我们就可以使用转换后的`target`了。|


## 2、异常处理
go语言没有`C++`和`java`等语言的`try...catch...finally`结构，而是采用`panic、defer、recover`来处理异常。

panic:
```go 
// The panic built-in function stops normal execution of the current
// goroutine. When a function F calls panic, normal execution of F stops
// immediately. Any functions whose execution was deferred by F are run in
// the usual way, and then F returns to its caller. To the caller G, the
// invocation of F then behaves like a call to panic, terminating G's
// execution and running any deferred functions. This continues until all
// functions in the executing goroutine have stopped, in reverse order. At
// that point, the program is terminated with a non-zero exit code. This
// termination sequence is called panicking and can be controlled by the
// built-in function recover.
/*
 * 翻译：
 * panic内置函数停止当前goroutine的正常执行。当函数F调用panic时，F的正常执行立即停止。任何被F延迟执行的函数都以通常的方式运行，然后F返回给它的调用者。对调用方G来说，F的调用行为就像对panic的调用，终止G的执行并运行任何延迟的函数。直到执行goroutine中的所有函数按相反顺序停止为止。此时，程序以非零退出代码终止。此终止序列称为panic，可由内置函数recover控制。 
*/
func panic(v interface{})
```
recover:
``` go 
// The recover built-in function allows a program to manage behavior of a
// panicking goroutine. Executing a call to recover inside a deferred
// function (but not any function called by it) stops the panicking sequence
// by restoring normal execution and retrieves the error value passed to the
// call of panic. If recover is called outside the deferred function it will
// not stop a panicking sequence. In this case, or when the goroutine is not
// panicking, or if the argument supplied to panic was nil, recover returns
// nil. Thus the return value from recover reports whether the goroutine is
// panicking.
/*
 * 翻译：
 * recover内置函数允许程序管理异常的goroutine的行为。在延迟函数（而不是它调用的任何函数）内执行recover调用，通过恢复正常执行来终止异常，并检索传递给panic的错误值。如果在defer函数外部调用recover，它将不会终止异常。在这种情况下，或者当goroutine没有异常时，或者如果提供给panic的参数为nil，recover返回nil。因此，recover的返回值判断goroutine是否发生异常。
*/
func recover() interface{}
```
!> `panic`、`recover`、`defer`的使用规则：

* `defer`语句必须在`panic`之前
* `recover` 只有在 `defer`函数中才有效，且必须实在第一层函数，否则`recover`无法捕获异常
* `recover` 处理异常后，程序不会恢复到 `panic` 的语句出，而是跑到 `defer` 之后语句逻辑处
* 多个 `defer` 会形成 `defer` 栈，后定义的 `defer` 语句会被最先调用：后入先出
* `defer`中引发的`panic`可以被之后`defer`中的`recover`捕获

错误1：`panic`置于`defer`之前，`recover`无法捕获`panic`，程序无法恢复
 ```go 
    func main() {

        panicFunc()

        defer func() {
            if err := recover(); err != nil {
                fmt.Println(err)
            }
        }()

        fmt.Println("main")
    }

    func panicFunc() {
        panic("panic in panicFunc")
    }
 ```
错误2：`recover`处于嵌套的`defer`函数内部，无法捕获`panic`，程序无法恢复

```go

func main() {

	defer func() {

		func() {
			if err := recover(); err != nil {
				fmt.Println(err)
			}
		}()
	}()
	panicFunc()

	fmt.Println("main")
}

func panicFunc() {
	panic("panic in panicFunc")
}
```

