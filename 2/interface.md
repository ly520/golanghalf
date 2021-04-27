# 接口

## 基础

#### 鸭子类型
>鸭子类型可以描述为：“当看到一只鸟走起来像鸭子、游泳起来像鸭子、叫起来也像鸭子，那么这只鸟就可以被称为鸭子。”
鸭子类型的关注点在于行为，而不再类型和属性。

go语言的接口和python一样也是实现的鸭子类型，接口是多个方法签名的集合。只要结构体拥有某个接口所声明的所有方法，就表示它 "实现" 了该接口，无须像java、C#一样在类型上显式声明实现了哪个接口。

!> 注意： 接口是一种类型，包含了类型信息和一个数据指针，当对象赋值给接口时，数据指针会指向该对象。其定义如下：

```go

    // 接口
    type iface struct {
        tab  *itab          // 接口表
        data unsafe.Pointer // 数据指针
    }

    // 空接口
    type eface struct {
        _type *_type         // 类型信息
        data  unsafe.Pointer // 数据指针
    }

```
#### `iface` ：
`iface`是go语言接口的定义：
 * `tab  *itab`：接口表存储元数据信息，包括接口类型、动态类型，以及实现接口的⽅方法指针。⽆无论是反
射还是通过接口调⽤用⽅方法，都会⽤用到这些信息。
* `data unsafe.Pointer`：数据指针，数据指针持有的是⺫⽬目标对象的只读复制品，复制完整对象或指针。因此对象赋值给接口的时候会引起复制。

#### `eface`
`eface` 是空接口的实现，没有方法，只包含了类型信息和数据指针

 #### go语言接口特性：
* 仅有方法： 接口只有方法声明，没有实现，没有数据字段。
* 嵌套：接口可以匿名嵌入其他接口，或嵌入到结构中。
* 对象赋值：对象赋值给接口时，会发生拷贝，而接口内部存储的是指向这个复制品的指针，既无法修改复制品的状态，也无法获取对象的指针。
* nil判断：只有当接口存储的类型和对象都为nil时，接口才等于nil。
* 接收者转换：接口调用不会做receiver的自动转换。
* 匿名：接口同样支持匿名字段方法。
* 多态：接口能够实现类似面向对象的多态性。
* 空接口：空接口可以作为任何类型数据的容器。
* 多实现：一个类型可实现多个接口。
* 命名习惯：go语言接口命名习惯以 `er` 结尾。
* 完全实现：如果实现了接口的一个方法，那么就必须实现接口的所有方法
  
 ```go
    package main

    import "fmt"

    func main() {
        var a Animal
        a = new(Cat)
        fmt.Println(a.GetName())
        fmt.Println(GetAnimalName(a))
        fmt.Println(Run(a))

        fmt.Println("--------------------")

        a = new(Dog)
        fmt.Println(a.GetName())
        fmt.Println(GetAnimalName(a))
        fmt.Println(Run(a))
    }

    func GetAnimalName(a Animal) string {
        return a.GetName()
    }

    func Run(a Animal) string {
        return a.Run()
    }

    type Animal interface {
        GetName() string
        Run() string
    }

    type Cat struct {
        Name string
    }

    func (a Cat) GetName() string {
        a.Name = "cat's name"
        return a.Name
    }

    func (Cat) Run() string {
        return "cat run"
    }

    type Dog struct {
        Cat
    }

    func (Dog) Run() string {
        return "dog run"
    }

    func (d Dog) GetName() string {
        return "dog's name"
    }

 ```


!> 当方法接收者是 `类型` 时，接口变量既可以接受类型也可以接受指针

!> 当方法接收者是 `指针` 时，接口变量则只接受指针；如果赋值一个类型会引发编译错误

```go 
    package main

    import "fmt"

    func main() {
        var a Animal
        a = Cat{} // 编译错误：cannot use (Cat literal) (value of type Cat) as Animal value in assignment: missing method GetName (GetName has pointer receiver)
        fmt.Println(a.GetName())

    }

    type Animal interface {
        GetName() string
        Run() string
    }

    type Cat struct {
        Name string
    }

    func (a *Cat) GetName() string {
        a.Name = "cat's name"
        return a.Name
    }

    func (*Cat) Run() string {
        return "cat run"
    }

```

#### 空接口

即：没有任何方法的接口,所有对象都实现了空接口。方法如果以空接口作为参数，那么这个方法可以传入任何类型的值。

```go 
func Move(a interface{}) {
	// todo
}
```

## 类型断言

一个接口的值（简称接口值）是由一个具体类型和具体类型的值两部分组成的。这两部分分别称为接口的动态类型和动态值。只有当这两部分的值都为nil时，接口的值才为nil。

断言语法：
``` Bash
    v, ok := x.(T)

    x：表示类型为interface{}的变量
    T：表示断言x可能是的类型。
    v：v转化为T之后的变量
    ok：断言是否成功
```

```go
func main() {
	var d interface{} = Dog{Cat: Cat{Name: "sdf"}}
	at, ok := d.(Dog)
	fmt.Printf("%v :%T \n", ok, at)
}
```
type switch:

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
