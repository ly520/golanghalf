# 结构

## 基础
结构体是一系列字段的集合，用来描述某一类型的数据。

### 结构定义
```
  type 结构名称 struct {
        字段名   类型
        字段名   类型
        …
    }
```
```go
type Car struct {
	Name           string
	Weight, Height float32
	IsRipe         bool
	Color          string
	Seat           int
}
```
#### 组合嵌套
go语言的结构体能通过组合嵌套的方式实现继承
``` go 
type Car struct {
	Name   string
	Weight float32
	IsRipe bool
	Color  string
	Seat   int
	size   Size // 具名方式
	Wheel       // 匿名方式
}

type Size struct {
	Height float32
	Width  float32
	Length float32
}

type Wheel struct {
	Size
	Name, Brand string
}

``` 
## 初始化

```go
package main

import "fmt"

func main() {
	// 1、基础实例化
	var c Car
	c.Name = "Car"
	c.IsRipe = true
	fmt.Printf("基础实例化:%+v\n", c)

	// 2、字面量实例化
	car := Car{
		Name:   "BMW",
		Weight: 132.25,
		IsRipe: false,
		Color:  "Red",
		Seat:   5,
	}
	fmt.Printf("字面量实例化:%+v\n", car)

	// 3、简洁实例化，顺序必须按结构定义的顺序，并且每个字段都必须赋值
	car2 := Car{
		"Berry",
		132.25,
		false,
		"Black",
		4,
	}
	fmt.Printf("简洁实例化:%+v\n", car2)

	// 4、指针
	car3 := &Car{
		Name:   "BMW33",
		Weight: 132.25,
		IsRipe: false,
		Color:  "Red",
		Seat:   5,
	}
	fmt.Printf("指针:%+v\n", car3)

	// 5、new函数
	cc := new(Car)
	cc.Name = "Linken"
	fmt.Printf("new函数:%+v\n", cc)

	// 6、匿名结构体
	var a struct {
		Name string
		Age  int
	}
	a.Name = "wwww"
	fmt.Printf("匿名结构体:%+v\n", a)

	// 7、匿名结构体,定义并初始化
	p := struct {
		Name  string
		IsMan bool
	}{
		Name:  "jack",
		IsMan: true,
	}
	fmt.Printf("匿名结构体,定义并初始化:%+v\n", p)

	// 8、组合结构
	wh := Wheel{
		Name:  "hanma",
		Brand: "BBA",
		Width: 12.256,
		Size: Size{
			Height: 4655.23,
			Width:  12.32,
			Length: 5.32,
		},
	}
	fmt.Printf("组合结构:%+v\n", wh)

	// 9、组合结构字段访问
	s := wh.Size.Height
	fmt.Printf("组合结构字段访问:%v\n", s)

}

type Car struct {
	Name   string
	Weight float32
	IsRipe bool
	Color  string
	Seat   int
}

type Size struct {
	Height float32
	Width  float32
	Length float32
}

type Wheel struct {
	Size
	Name, Brand string
	Width       float32
}

```
输出：
``` 
基础实例化:{Name:Car Weight:0 IsRipe:true Color: Seat:0}
字面量实例化:{Name:BMW Weight:132.25 IsRipe:false Color:Red Seat:5}
简洁实例化:{Name:Berry Weight:132.25 IsRipe:false Color:Black Seat:4}
指针:&{Name:BMW33 Weight:132.25 IsRipe:false Color:Red Seat:5}
new函数:&{Name:Linken Weight:0 IsRipe:false Color: Seat:0}
匿名结构体:{Name:wwww Age:0}
匿名结构体,定义并初始化:{Name:jack IsMan:true}
组合结构:{Size:{Height:4655.23 Width:12.32 Length:5.32} Name:hanma Brand:BBA Width:12.256}
组合结构字段访问:4655.23
```

## 下划线
结构体的定义可以使用下划线，意为：在使用字面量初始化时，强制使用键值对的方式，不可使用简洁的方式,否则会引发编译错误
```go
type Car struct {
	_      int
	Name   string
	Weight float32
	IsRipe bool
	Color  string
	Seat   int
}

func main() {
    // 字面量实例化
	car := Car{
		Name:   "BMW",
		Weight: 132.25,
		IsRipe: false,
		Color:  "Red",
		Seat:   5,
	}
	fmt.Printf("字面量实例化:%+v\n", car)

	//  编译错误，定义时使用了'_'，不可使用简洁方式初始化
	car2 := Car{
		"Berry",
		132.25,
		false,
		"Black",
		4,
	}
}
```

## Tag
有时候我们在定义结构的时候，需要一些额外的描述，这些额外描述可以通过tag实现，我们能够通过反射获取到tag的信息：
```go
type Car struct {
	Name   string  `hello`
	Weight float32 "haha"
	IsRipe bool    `isripe`
	Color  string  "color"
	Seat   int     "seat"
}

func main() {
	t := reflect.TypeOf(Car{})
	f1, _ := t.FieldByName("Name")
	fmt.Println(f1.Tag)
	f4, _ := t.FieldByName("Weight")
	fmt.Println(f4.Tag)
	f5, _ := t.FieldByName("Color")
	fmt.Println(f5.Tag)
}
```
输出：
```
hello
haha
color
```
> Tag 定义使用反引号和双引号都是可以的

 通常使用`Tag`都会使用结构化的键值对来定义，以便于解析,格式为： `key1:"value1" key2:"value2" key3:"value3"`:
  * 使用反引号定义tag
  * key不需要引号
  * value需要使用双引号
  * key与key之间通过空格分隔
 
 如果`Tags`格式没问题的话，我们可以通过`Lookup`或者`Get`来获取键值对的值。`Lookup`回传两个值:对应的值和是否找到

 ```go
    package main

    import (
        "fmt"
        "reflect"
    )

    func main() {
        t := reflect.TypeOf(Car{})
        f, _ := t.FieldByName("Name")
        fmt.Println("tag:", f.Tag)

        v, ok := f.Tag.Lookup("db")
        fmt.Printf("lookup db:%s, %t\n", v, ok)

        v, ok = f.Tag.Lookup("type")
        fmt.Printf("lookup type:%s, %t\n", v, ok)

        v, ok = f.Tag.Lookup("defualt")
        fmt.Printf("lookup defualt:%s, %t\n", v, ok)

        v = f.Tag.Get("defualt")
        fmt.Printf("get defualt:%s\n", v)
    }
 ```

 输出：

``` bash
    tag: db:"name" type:"varchar(50)" defualt:"golang"
    lookup db:name, true
    lookup type:varchar(50), true
    lookup defualt:golang, true
    get defualt:golang
```
