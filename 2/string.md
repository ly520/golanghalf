# 字符串
## 基础
!> go的字符串是不可变的字节序列。
基本操作：
* len() 函数
* `+` 拼接字符串
* 单引号`' `表示rune字符
* 双引号`" `表示字节序列
* 反引号 ` 表示原始字符串，会忽略转义符，可多行显示

go提供了通过下标访问的方式：
```go 
	a := "hello world"
	fmt.Println(len(a))
	fmt.Printf("%d,%d\r\n", a[2], a[7])
	fmt.Printf("%s,%s\r\n", string(a[2]), string(a[7]))
```  
输出:
```bash
11
108,111
l,o
```
由于go字符串是字节序列，所以下标访问到的是`byte` 值,需要类型转换之类才能获得字符值。

## 中文字符串
看下面的代码：
```go
func main() {
	a := "窗前明月光"
	fmt.Println(len(a))
	fmt.Printf("%d,%d\r\n", a[2], a[7])
	fmt.Printf("%s,%s\r\n", string(a[2]), string(a[7]))
}
```
输出：

``` bash 
15
151,152
,
```
!> go语言使用utf8编码。中文字符占3个字节，通过下标访问到的是字节，无法访问原字符。此时需要使用rune数组

修改上面的代码：

```go
func main() {
	a := "窗前明月光"
	r := []rune(a)
	fmt.Println(len(r))
	fmt.Printf("%d,%d\r\n", r[2], r[4])
	fmt.Printf("%s,%s\r\n", string(r[2]), string(r[4]))
}
```
输出：

``` bash 
5
26126,20809
明,光
```
此时输出符合预期。

## Unicode
> Unicode 它囊括了世界上所有文书体系的全部字符，还有重音符和其他变音符，控制码（如制表符和回车符)，以及许多特有文字，对它们各自赋予一个叫Unicode码点的标准数字。在Go的术语中，这些字符记号称为文字符号(`rune`)。

简单说就是以`int32`的数字来表示世界上所有的字符，这就是go中的`rune`。

## 常用操作
### 常用包
|包名|说明|
|----|----|
|strings|提供搜索、替换、比较、修整、切分、连接字符串、ToUpper、ToLower....等函数,生成并返回一个新字符串|
|bytes|由于字符串不可变，因此按增量方式构建字符串会导致多次内存分配和复制。这种情况下，使用bytes.Buffer类型会更高效。|
|strconv|的函数，主要用于转换布尔值、整数、浮点数为与之对应的字符串形式，或者把字符串转换为布尔值、整数、浮点数，另外还有为字符串添加/去除引号的函数。|
|unicode|包含判别文字符号值特性的函数，如IsDigit、IsLetter、Isupper和IsLower。每个函数以单个文字符号值作为参数，并返回布尔值。若文字符号值是英文字母，转换函数(如Toupper和ToLower)将其转换成指定的大小写。所有函数都遵循Unicode标准对字母数字等的分类原则。|

### strings包常用函数

|函数|说明|
|----|----|
|len(str)|	求长度|
|+或fmt.Sprintf	|拼接字符串|
|strings.Split	|分割|
|strings.Contains|	判断是否包含|
|strings.HasPrefix,strings.HasSuffix|	前缀/后缀判断|
|strings.Index(),strings.LastIndex()|	子串出现的位置|
|strings.Join(a[]string, sep string)|	join操作|
|strings.Fields(s string) []string|按一个或多个空格切分字符串|

!> 由于字符串是不可变的，`strings`对字符串操作是都会重新分配内存，而造成性能低下，因此最好使用`byte`包中的`byte.Buffer`来操作字符串
``` go
func BenchmarkString(b *testing.B) {
	s := "hello "
	for i := 0; i < b.N; i++ {
		s += " golang"
	}

}

func BenchmarkBuffer(b *testing.B) {
	var buf bytes.Buffer
	buf.WriteString("hello ")
	for i := 0; i < b.N; i++ {
		buf.WriteString(" golang")
	}
}
```
执行结果：
```bash
 go test -v -bench="." -benchmem

BenchmarkString
BenchmarkString-4          82735             53751 ns/op          293508 B/op          1 allocs/op
BenchmarkBuffer
BenchmarkBuffer-4       90417729                14.19 ns/op           26 B/op          0 allocs/op
```
由上可以看出性能差距十分明显。

## 字符串转换
字符串和数字的格式化通过`strconv`包完成。
|函数|说明|
|----|----|
|strconv.Atoi()|ASCII to integer， 字符串转化为整数|
|strconv.ItoA()|integer to ASCII， 整数转化成字符串|
|FormatInt、FormatUint|按指定的进制转化|
|ParseInt|按进制转化字符串|

## 字符串格式化占位符
|占位符|说明|
|-------|------|
|%v |相应值的默认格式。在打印结构体时，“加号”标记（%+v）会添加字段名|
|%#v| 相应值的 Go 语法表示|
|%T |相应值的类型的 Go 语法表示|
|%% |字面上的百分号，并非值的占位符|
|%t |单词 true 或 false。|
|%b |二进制表示|
|%c |相应 Unicode 码点所表示的字符|
|%d |十进制表示|
|%o |八进制表示|
|%q |单引号围绕的字符字面值，由 Go 语法安全地转义|
|%x |十六进制表示，字母形式为小写 a-f|
|%X |十六进制表示，字母形式为大写 A-F|
|%U |Unicode 格式：U+1234，等同于 “U+%04X”|
|%b |无小数部分的，指数为二的幂的科学计数法，与 strconv.FormatFloat 的 ‘b’ 转换格式一致。|
|%e |科学计数法，例如 -1234.456e+78|
|%E |科学计数法，例如 -1234.456E+78|
|%f |有小数点而无指数，例如 123.456|
|%g |根据情况选择 %e 或 %f 以产生更紧凑的（无末尾的 0）输出|
|%G |根据情况选择 %E 或 %f 以产生更紧凑的（无末尾的 0）输出|
|%s |字符串或切片的无解译字节|
|%q |双引号围绕的字符串，由 Go 语法安全地转义|
|%x |十六进制，小写字母，每字节两个字符|
|%X |十六进制，大写字母，每字节两个字符|
|%p |指针：十六进制表示，前缀 0x|

