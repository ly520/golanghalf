# 方法

## 基础

go语言的方法绑定在对象实例上，与java、C#、C++方法定义不同，go语言在定义方法时需要指定接收者。

* 接收者：go语言方法需要指定接收者
* 当前包：只能为当前包定义方法，go语言不支持类似C#语言的扩展方法
* 接收者参数：接收者参数可以任意命名，官方推荐使用使用结构名称的第一个字母，且小写；如果方法中不使用改变量，那么变量可以省略
* 参数类型：接收者参数类型可以是类型或者类型指针，不能是接口
* 无重载：go语言不支持方法重载，在一个接收者中，方法名必须唯一
* 可以使用实例或者指针调用方法，编译器会自动转换
* 大小写原则：方法同样遵守大小写原则，小写仅包内可访问

定义语法：
```
func (接收者参数 接收者类型) 方法名(参数列表)(返回值列表){
    // todo    
}
```

```go
    package main

    import "fmt"

    func main() {
        a := new(Animal)
        fmt.Println(a.name())
        fmt.Println(a.Name)

        a = a.New()
        fmt.Println(a.Name)
        fmt.Println(a.Run())
    }

    type Animal struct {
        Name string
    }

    func (a *Animal) New() *Animal {
        a.Name = "new animal"
        return a
    }

    func (a Animal) name() string {
        a.Name = "animal"
        return a.Name
    }

    func (Animal) Run() string {
        return "animal run"
    }   

```
输出：
```
animal

new animal
animal run
```

!> 注意：第二行输出为空！因为name()方法定义时使用的是类型，而不是指针，因此传递的是类型Animal的副本，所以在name()中修改对象的属性，是不影响原对象的。所以其属性Name依旧是零值，即空字符串。而New()方法传递的是对象指针，所以是能修改原对象信息的。

!> 如果希望修改对象的值那么接收者应该使用指针，反之可以使用类型。

!>如果有某个方法的接收者使用了指针，那么建议所有的方法都使用指针，以避免程序员理解错误而误用造成bug

## 内嵌的结构方法

结构对象可以直接访问内嵌的结构的方法，与只用自身的方法一样；如果存在同名方法，则调用自身方法，不会调用内嵌的结构方法。

```go
package main

import "fmt"

func main() {
	a := new(Animal)
	fmt.Println(a.name())
	fmt.Println(a.Name)

	a = a.New()
	fmt.Println(a.Name)
	fmt.Println(a.Run())

	d := new(Dog)
	fmt.Println("dog:", d.Run())
	fmt.Println("dog:", d.name())

	d = d.New()
	fmt.Println("dog:", d.Name)
	fmt.Println("dog:", d.Run())

}

type Animal struct {
	Name string
}

func (a *Animal) New() *Animal {
	a.Name = "new animal"
	return a
}

func (a Animal) name() string {
	a.Name = "animal"
	return a.Name
}

func (Animal) Run() string {
	return "animal run"
}

type Dog struct {
	Animal
}

func (d *Dog) New() *Dog {
	d.Name = "dog type, hello ketty"
	return d
}

func (Dog) Run() string {
	return "dog run"
}

```
