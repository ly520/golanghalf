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

## 无类型常量
go语言常量特殊性：这样的常量没有固定的从属于某一个类型，它的值比基本类型精度要高，且算术精度高于原生机器精度。

从属类型待定的常量共有6种，分别是无类型布尔、无类型整数、无类型文字符号、无类型浮点数、无类型复数、无类型字符串。例如内建类型：(/builtin/builtin.go)
```go
// true and false are the two untyped boolean values.
// true and false 是两个无类型bool值
const (
	true  = 0 == 0 // Untyped bool.
	false = 0 != 0 // Untyped bool.
)
```
无类型常量会延迟确定其从属类型，我们能够将吴类型常量赋值给不同的基础类型，而无需类型转换，这看起来有点像是动态语言的特性：
```go
func main() {
	var a float32 = math.Pi
	var b float64 = math.Pi
	var c complex64 = math.Pi
	var d complex128 = math.Pi

	fmt.Println(a) // 3.1415927
	fmt.Println(b) // 3.141592653589793
	fmt.Println(c) // (3.1415927+0i)
	fmt.Println(d) // (3.141592653589793+0i)
}
```
!> 只有常量才能是无类型的，当转换为另外的类型时，必须邀请目标类型能表示原始值，以下会出现编译错误，因为`int32`和`int64`无法表示`Pi`的值
```go 
	var e int32 = math.Pi 
	var f int64 = math.Pi
```

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

`make()`函数是go语言专门用来创建`切片 slice`和`通道 channel`以及`map`的专用函数，与`new()`不用，`make()`返回的是引用类型

!> `new()`函数返回的是变量指针，`make()`返回的是类型本身
```go 
func main() {
	fmt.Printf("%T\r\n", make([]int, 4)) //  []int ，返回类型本身
	fmt.Printf("%T\r\n", new([4]int)) // *[4]int , 返回指针
}
```
new和make的函数官方签名和描述如下：
``` go 
// The make built-in function allocates and initializes an object of type
// slice, map, or chan (only). Like new, the first argument is a type, not a
// value. Unlike new, make's return type is the same as the type of its
// argument, not a pointer to it. The specification of the result depends on
// the type:
//	Slice: The size specifies the length. The capacity of the slice is
//	equal to its length. A second integer argument may be provided to
//	specify a different capacity; it must be no smaller than the
//	length. For example, make([]int, 0, 10) allocates an underlying array
//	of size 10 and returns a slice of length 0 and capacity 10 that is
//	backed by this underlying array.
//	Map: An empty map is allocated with enough space to hold the
//	specified number of elements. The size may be omitted, in which case
//	a small starting size is allocated.
//	Channel: The channel's buffer is initialized with the specified
//	buffer capacity. If zero, or the size is omitted, the channel is
//	unbuffered.
/*
翻译为：
内建函数 make 分配并且初始化 一个 slice, 或者 map 或者 chan 对象。 并且只能是这三种对象。 和 new 一样，第一个参数是 类型，不是一个值。 但是make 的返回值就是这个类型（即使一个引用类型），而不是指针。 具体的返回值，依赖具体传入的类型。

Slice : 第二个参数 size 指定了它的长度，此时它的容量和长度相同。
你可以传入第三个参数 来指定不同的容量值，但是必须不能比长度值小。
比如: make([]int, 0, 10)
Map: 根据size 大小来初始化分配内存，不过分配后的 map 长度为0。 如果 size 被忽略了，那么会在初始化分配内存的时候 分配一个小尺寸的内存。
Channel: 管道缓冲区依据缓冲区容量被初始化。如果容量为 0 或者被 忽略，管道是没有缓冲区的。
*/
func make(t Type, size ...IntegerType) Type

// The new built-in function allocates memory. The first argument is a type,
// not a value, and the value returned is a pointer to a newly
// allocated zero value of that type.
/*
翻译为：
new 内置函数分配内存。第一个参数是类型，而不是值，返回的值是指向该类型的新分配的零值的指针。
*/
func new(Type) *Type
```

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