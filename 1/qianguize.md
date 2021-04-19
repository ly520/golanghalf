# golang 概览-潜规则

## 1、概览
``` go
package main

import "fmt"

var f string

func init() {
	f = "go "
}

func main() {
	f += "world!"
	ss := "golang 初体验"
	fmt.Printf("Hello %s \r\n%s", f, ss)
}

```
输出结果：
``` bash
Hello go world!
golang 初体验
```


从上面的程序我们可以看出go程序的基本结构：
* `package` 用来定义包名，go语言值函数式语言，变量、常量和函数是go程序的基本单元，他们通过包组织在一起，这个面向对象语言通过类来组织代码块是不同的。
* `import` 用来引用包，`"fmt"`不是包名而是包路径，go内置包的路径都很简单，几乎都是一个词语，第三方包，如beego则是`"github.com/astaxie/beego"`。
* `var` 用以定义变量，`string`是变量类型，`:=`用来定义并初始化变量，是一个语法糖，用来简化变量定义与初始化。
* `init` 函数，特殊函数，包完成初始化之后自动调用，会在`main`函数之前被调用。
* `main` 函数，go程序的入口

> 以上就是go程序的基本结构了，这其中还有一些潜规则需要注意，这些潜规则是和其他语言十分不同的

## 潜规则

!> 1、`{` 左花括号不能独立一行
``` go
func main() {
	t := true
	if t {
		fmt.Println("true")
	} else { // 这里的左花括号也不能独立一行
		fmt.Println("false")
	}
}
```
!> 不过也有例外，`select`的`case`分支的代码块中`{`可以独立一行,但是`case`的花括号不是必须的：
```go 
// 该程序交替打印数字和字母
package main

import (
	"fmt"
	"sync"
)

func main() {
	char_ch, num_ch := make(chan bool), make(chan bool)
	wg := sync.WaitGroup{}

	go func() {
		i := 1
		for {
			select {
			case <-num_ch:
				{ // 这里的{可以独立一行，但是{}在这里不是必须的
					fmt.Println(i)
					i++
					fmt.Println(i)
					i++

					char_ch <- true
				}
			default:
				{
					break
				}
			}

		}
	}()
	wg.Add(1)

	go func(wait *sync.WaitGroup) {
		i := 'A'
		for {
			select {
			case <-char_ch:
				if i >= 'Z' {
					wg.Done()
					return
				}
				fmt.Println(string(i))
				i++
				fmt.Println(string(i))
				i++

				num_ch <- true
			default:
				break
			}

		}
	}(&wg)

	num_ch <- true // 给num_ch一个初始值
	wg.Wait()
}


```

!> 2、golang单行代码不需要分号`;`,编译器会自动添加。一行有句代码，则最后一句不需要分号。

```go 
func main() {
	if b := t(); b == true { // t()后面的分号不能省
		fmt.Println("true")
	} else {
		fmt.Println("false")
	}
}

func t() bool {
	return true
}
```

!> 3、 go区分大小写且与访问级别相关：小写开头的变量或函数是私有的，仅包内可访问。大写开头则是公共的，包内外皆可访问。

!> 4、init函数是自动调用的，且没有参数和返回值。init函数每个包可以有多个，同一个文件也可以有多个，go将按规则一次访问。

!> 5、go自带单元测试工具，单元测试的代码文件必须以`"_test.go"`结束，如：`main_test.go`会被识别问单元测试代码文件。
* 单元测试函数名必须以`"Test"`开始，且参数为`*testing.T`类型
* 性能测试函数名必须以`"Benchmark"`开始，且参数为`*testing.B`类型
``` go 
// 单元测试
func TestFirst(t *testing.T) {

}
// 性能测试
func BenchmarkFirst(b *testing.B) {

}

```

!> 6、go程序包名与文件无需相同，但一个文件夹只能包含一个包名。如果一个文件夹中的不同文件使用不同的包名，那么会编译失败。

