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



## 基础类型

## 变量作用域

## 初始化