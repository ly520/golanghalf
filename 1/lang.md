# 变量
## 定义变量
golang变量通过var关键字定义，golang是静态编译型语言，不允许在运行时改变变量的类型，在定义时需要指定变量类型，变量会自动被初始化零值。golang支持类型推断，如果变量给出初始化值，那么可以省略类型：
``` go
var i int
var a int = 64
var s="hello world"
```
golang 可以同时定义多个变量
``` go
var a,b,c int
var s,l="hello","world"
var (
    g int
    f float32
    ss string
)
```

## 常量
常量表示不可变量，通过`const`定义，`const`用法和`var`相同：
```go
const x int = 3
const y int = 0
const (
	z int    = 4
	s string = "golang"
)
```
!> 常量定义必须给与常量值，常量在编译期会被展开，即以常量值替代标识符，直接同于计算。如上代码，编译之后`x、y、z、s` 会被替代为各自的值，作为指令数据使用。因而常量是无法获得地址的。、

## 枚举
go语言没有`C#`那种明确指出的`enum`,但是我们可以使用`iota`实现一组自增常量来实现：
```go 
const (
	man    = iota // 0
	women         // 1
	unkown        // 2
)
```
`iota` 默认值为零，然后依次递增，如果不想从零开始，可以忽略第一个值：
```go 

const (
	_      = iota // 0
	man           // 1
	women         // 2
	unkown        // 3
)
```
`iota`也可以融入计算表达式：
```go
const (
	_, _ = iota, iota * 10 // 0,0
	x, a                   // 1,10
	y, b                   // 2,10
	z, c                   // 3,30
)
```
显示指定枚举值：
```go
const (
	_ = iota // 0
	a        // 1
	b        // 2
	c = 100  // 100 显示指定
	d        // 100 与上一行相同
	e = iota // 3 恢复iota自增
	f        // 4 自增
)
```

## 基础类型
| 类型                          | 默认值 | 说明                            |
| ----------------------------- | ------ | ------------------------------- |
| int8、int16、int32、int64     | 0      | 整型数值                        |
| uint8、uint16、uint32、uint64 | 0      | 无符号整型                      |
| float32、float64              | 0.0    | 浮点数                          |
| uint32、uint64                | 0      | 整型数值                        |
| rune                          | 0      | Unicode Code Point、int32的别名 |
| byte                          | 0      | uint8的别名                     |
| complex64、complex128         |        | 复数                            |
| bool                          | false  | 布尔值                          |
|                               |        |                                 |
| string                        | ""     | 字符串                          |
| array                         |        | 数组                            |
| struct                        |        | 结构体                          |
| function                      | nil    | 函数                            |
| interface                     | nil    | 接口                            |
| map                           | nil    | 字典、引用类型                  |
| slice                         | nil    | 切片、引用类型                  |
| channel                       | nil    | 管道、引用类型                  |
| pointer                       | nil    | 指针                            |

## 指针
go语言支持指针，通过`*`操作符定义。指针指向变量的存储地址，因此指针的值就是变量的地址。可以通过取址运算符取得变量的地址：
```go 
func fn() *int {
	i := 30

	return &i
}
func main() {

	var name string = "golang"
	fmt.Printf("%p\r\n", &name)
	fmt.Printf("%p\r\n", fn())
}
```
输出
``` shell
0xc00003a240
0xc000012078
```

## new()、make()函数
`new()`用以创建变量，并返回变量的地址，通常用来创建结构体变量：
```go 
i := new(int)
s := new(string)
```

`make()`函数是go语言专门用来创建`切片 slice`和`通道 channel`以及`map`的专用函数，与`new()`不用，他返回的是变量指针,即`make()`返回的值引用类型


!> `new()`函数返回的是变量地址，`make()`返回的是变量指针

## 类型声明
类型通过`type`关键字声明，go语言可以声明三种不同的类型：
* interface 接口
* struct 结构
* function 函数
```go 
type Student struct {
	Name   string
	Age    int
	IsMale bool
}

type Sty interface {
	SayHi()
	Work(name string) int
}

type ShowStudent func(name string, age int, isMale bool) (string, int)

```


## 变量作用域
go语言支持两种作用域：
* 块级作用域：局部变量、形式参数
* 全局变量
!> go语言的`:=`操作符会定义新的变量，即便是同名变量，只要不在同一个作用域中，就会定义新的变量：
```go
func fn() (string, error) {
	return "test scope of variable", nil
}

func main() {
	var name string
	if name, err := fn(); nil == err {
		fmt.Printf("error: %p,%s\r\n", &name, name)
	}
	fmt.Printf("test:%p,%s\r\n", &name, name)
	fmt.Println("Hello, 世界")
}

```

输出：
``` shell
error: 0xc00003a250,test scope of variable
test:0xc00003a240,
Hello, 世界
```
从结果可以看出两个`name`的地址是不同的，`if`中的`name`变量通过`:=`操作符定义，虽然名称同为`name`，但是由于`var`定义的`name`与`if`中的`name`作用域不同，`var name string` 作用于`main`方法，而`if`中的`name`只作用于`if`语句块，因此`:=`会重新定义一个变量替代前面的`name`。

>写过js程序的朋友肯定都知道，js中十分令人头疼的问题便是全局变量，几乎是一个地狱，命名冲突、难以维护、逐渐臃肿、朔源困难...，而go语言的这个特性便是解决全局问题的一个有效手段。