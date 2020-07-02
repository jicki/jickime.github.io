# Go学习笔记


# Go语言基础

## 标识符与关键字

### 标识符

* 在编程语言中标识符就是程序员定义的具有特殊意义的词, 比如 变量名、常量名、函数名等. Go语言中标识符由字母数据和`_`(下划线)组成,并且只能以字母和`_`开头。比如 `Abc`、`_Abc`、`a123`

### 关键字

* 关键字是指编程语言中预先定义了具有特殊含义的标识符。关键字和保留字都不建议用作变量名。

* Go语言中有25个关键字:

```shell
break     default      func    interface  select
case      defer        go      map        struct
chan      else         goto    package    switch
const     fallthrough  if      range      type
continue  for          import  return     var
```

* Go语言中的37个保留字:

```shell
常量: true  false  iota  nil

类型:  
    int      int8      int16        int64
    uint     uint8     uint16       uint32   uint64  uintptr
    float32  float64   complex128   complex64
    bool     byte      rune         string   error

方法(functions):
    make      len    cap   new   append   copy   close   delete
    complex   real   imag
    panic     recover

```

### 变量

* 变量的含义: 程序运行过程中的数据都是保存在内存中，我们想要在代码中操作某个数据时就需要去内存上找到这个变量，但是如果我们直接在代码中通过内存地址去操作变量的话，代码的可读性会非常差而且还容易出错，所以我们就利用变量将这个数据的内存地址保存起来，以后直接通过这个变量就能找到内存上对应的数据了。

* 变量类型: 变量 的功能是存储数据。不同的变量保存的数据类型可能会不一样。常见变量的数据类型有: 整型、浮点型、布尔型等。 Go语言中的每一个变量都有自己的类型，并且变量必须经过声明才能开始使用。

* 变量声明: Go语言中的变量需要声明后才能使用，同一作用域内不支持重复声明。 并且Go语言的变量声明后必须使用, 否则会报错。

* 变量声明格式: `var 变量名 变量类型`, 函数外的变量声明必须都是以 var 开头进行声明。

* 变量赋值: `var 变量名 变量类型 = 值` `var 变量名1,变量名2 = 变量1的值, 变量2的值`。

```go
package main

import "fmt"

func main() {
	// 标准声明
	var name string
	var age int
	var isOk bool
	// 批量声明
	var (
		a string
		b int
		c bool
	)
	// 变量赋值
	name = "名字"
	age = 19
	isOk = false
	a = "A"
	b = 12
	c = true
	// 短变量声明,只能在函数内使用
	d := "dddd"
	// 输出
	fmt.Println(name, age, isOk, a, b, c, d)
}
```

### 常量

* 相对于变量，常量是恒定不变的值，多用于定义程序运行期间不会改变的那些值。 常量的声明和变量声明非常类似，只是把var换成了const，常量在定义的时候必须赋值。

* 常量定义与赋值: `const 常量名 = 值`

```go
// 常量定义
const a = 3333
const b = 4444

// 批量定义
const (
	c = 5555
	d = 6666
)

// 常量如果不写值会继承上一个的值
const (
	e = 7777
	f
	g
)
```

* `iota` 关键字: `iota` 是go语言中的常量计数器, 只能在常量表达式中使用。

* `iota` 在`const`关键字出现时将被重置为0。`const`中每新增一行常量声明将使`iota`计数一次(`iota` 可理解为`const`语句块中的行索引)。 使用`iota`能简化定义，在定义枚举时很有用。

```go
const (
	a = iota //0
	b        //1
	c        //2
	d        //3
)
```

* `iota` 在常量使用 `_`(忽略)仍然会占用一位。

```go
const (
    a = iota //0
    b        //1
    _
    d        //3
)
```

```go
const (
	a1 = iota //0
	a2        //1
	a3 = 100  //100
	a4 = iota //3
)
```

```go
const (
	b1, b2 = iota + 1, iota + 2 // 1, 2
	b3, b4                      // 2, 3
	b5, b6                      // 3, 4
)
```

## 数据类型

* Go语言中有丰富的数据类型，除了基本的整型、浮点型、布尔型、字符串外，还有数组、切片、结构体、函数、map、通道（channel）等。

### 整型

* 整型分为以下两个大类： 按长度分为：`int8`、`int16`、`int32`、`int64` 对应的无符号整型：`uint8`、`uint16`、`uint32`、`uint64` 其中，`uint8`就是我们熟知的`byte`型，`int16`对应C语言中的`short`型，`int64`对应C语言中的`long`型。

|类型|描述|
|-|-|
|int8|有符号 8位整型 (-128 到 127)|
|int16|有符号 16位整型 (-32768 到 32767)|
|int32|有符号 32位整型 (-2147483648 到 2147483647)|
|int64|有符号 64位整型 (-9223372036854775808 到 9223372036854775807)|
|uint8|无符号8位整型(0 到 255)|
|uint16|无符号16位整型(0 到 65535)|
|uint32|无符号32位整型(0 到 4294967295)|
|uint64|无符号 64位整型 (0 到 18446744073709551615)|

### 特殊类型

* `int`, `uint` 在不同平台中有差异。

|类型|描述|
|-|-|
|uint|32位操作系统上就是uint32，64位操作系统上就是uint64|
|int|32位操作系统上就是int32，64位操作系统上就是int64|
|uintptr|无符号整型，用于存放一个指针|

* 获取对象的长度的内建`len()`函数返回的长度可以根据不同平台的字节长度进行变化。实际使用中，`slice`切片或 `map` 的元素数量等都可以用`int`来表示。在涉及到二进制传输、读写文件的结构描述时，为了保持文件的结构不会受到不同编译目标平台字节长度的影响，不要使用`int`和`uint`。

### 数字字面量语法（Number literals syntax）

* 数字字面量语法，便于开发者以二进制、八进制或十六进制浮点数的格式定义数字。`v := 0b00101101`， 代表二进制的 101101，相当于十进制的 45。 `v := 0o377`，代表八进制的 377，相当于十进制的 255。 `v := 0x1p-2`，代表十六进制的 1 除以 2²，也就是 0.25。 而且还允许我们用 `_` 来分隔数字, 比如说: `v := 123_456` 等于 123456。

### 八进制和十六进制

* `fmt` 包中可转换成这类进制的值。

```go
	// 十进制
	var a int = 10
	// %b 表示转换成二进制
	fmt.Printf("%b \n", a)
	// %d 表示转换成十进制
	fmt.Printf("%d \n", a)

	// 八进制
	var b int = 077
	// %o 表示转换成八进制
	fmt.Printf("%o \n", b)
	// %d 表示转换成十进制
	fmt.Printf("%d \n", b)

	// 十六进制
	var c int = 0xff
	// 直接打印值
	fmt.Println(c)
	// %x 表示转换成十六进制
	fmt.Printf("%x \n", c)

```

### 浮点型

* Go语言支持两种浮点型数: `float32` 和`float64`。这两种浮点型数据格式遵循IEEE 754标准: `float32` 的浮点数的最大范围约为 3.4e38, 可以使用常量定义: `math.MaxFloat32`. `float64` 的浮点数的最大范围约为 `1.8e308`, 可以使用一个常量定义: `math.MaxFloat64`。

```go
func main() {
	// 打印 32位 最大浮点数 3.4028234663852886e+38
	fmt.Println(math.MaxFloat32)
	// 打印 64位 最大浮点数 1.7976931348623157e+308
	fmt.Println(math.MaxFloat64)
}
```

### 复数

* 复数有实部和虚部, `complex64`的实部和虚部为32位, `complex128`的实部和虚部为64位。

```go
	var c1 complex64
	c1 = 1 + 2i
	var c2 complex128
	c2 = 2 + 3i
	fmt.Println(c1)
	fmt.Println(c2)
```

### 布尔类型

* Go语言中以`bool`类型进行声明布尔型数据，布尔型数据只有`true`（真）和`false`（假）两个值。
* 布尔类型变量的默认值为false。布尔类型不能进行数值运算,也无法与其他类型进行转换。

```go
func main() {
	var b bool
	// 默认值为 false
	fmt.Println(b)
	// 赋值
	b = true
	fmt.Println(b)
}

```

### 字符串(string)

* Go语言中的字符串以原生数据类型出现, 使用字符串就像使用其他原生数据类型（int、bool、float32、float64 等）一样. Go 语言里的字符串的内部实现使用UTF-8编码. 字符串的值为 `双引号(")` 中的内容, 可以在Go语言的源码中直接添加非ASCII码字符.

```go
func main() {
	// 字符串
	s1 := "hello"
	s2 := "哈喽"
	fmt.Println(s1)
	fmt.Println(s2)
}
```

* 字符串转移 - Go 语言的字符串常见转义符包含回车、换行、单双引号、制表符等，如下表所示。

|转义符|含义|
|-|-|
|`\r`|回车符(返回行首)|
|`\n`|换行符(直接跳到下一行的同列位置)|
|`\t`|制表符|
|`\'`|单引号|
|`\"`|双引号|
|`\\`|反斜杠|

```go
func main() {
    // 转义 如下反斜杠
	fmt.Println("c:\\golang\\bin\\go.exe")
}
```

* 多行字符串 - Go语言中要定义一个多行字符串时，就必须使用`反引号`字符

```go

func main() {
	// 多行字符串
	s3 := `
	第一行
	第二行
	第三行
	""
	''
	\
    `
}
```

* 字符串常用操作

|方法|介绍|
|-|-|
|`len(str)`|求长度|
|`+或fmt.Sprintf`|拼接字符串|
|`strings.Split`|分割|
|`strings.contains`|判断是否包含|
|`strings.HasPrefix`,`strings.HasSuffix`|前缀/后缀判断|
|`strings.Index()`,`strings.LastIndex()`|子串出现的位置|
|`strings.Join(a[]string, sep string)`|join操作|

```go
func main() {
	// 字符串
	s1 := "hello"
	s2 := "哈喽"
	s4 := fmt.Sprintf("%s -- %s", s1, s2)
	s5 := "how old are you"
	// 返回字符串长度
	fmt.Println(len(s5))
	// 字符串拼接
	fmt.Println(s1 + s2)
	fmt.Println(s4)
	// 字符串分割
	fmt.Println(strings.Split(s5, " "))
	// 切割后变成 slice 切片
	fmt.Printf("类型 = %T \n", strings.Split(s5, " "))

	// 判断字符串是否包含某个字符,返回 布尔值
	fmt.Println(strings.Contains(s5, "ow"))

	// 判断前缀,与后缀, 返回 布尔值
	fmt.Println(strings.HasPrefix(s5, "how"))
	fmt.Println(strings.HasSuffix(s5, "you"))

	// 判断字符串的位置 ,结果为 1 (从0开始)
	fmt.Println(strings.Index(s5, "o"))
	// 判断字符串最后出现的位置 , 结果为 13
	fmt.Println(strings.LastIndex(s5, "o"))

	// join 操作, 以为定义的符号连接字符串
	s6 := []string{"how", "old", "are", "you"}
	fmt.Println(s6)
	fmt.Println(strings.Join(s6, "-"))
}
```

### byte 与 rune 类型

* 组成每个字符串的元素叫做“字符”, 可以通过遍历或者单个获取字符串元素获得字符。 字符用单引号（’）包裹起来。

* byte: 其实就是`uint8`类型，或者叫 `byte` 型，代表了ASCII码的一个字符。

* rune: 其实就是 `int32`类型, 代表一个 UTF-8 字符。

```go
func main() {
	var c1 byte = 'c'
	var c2 rune = 'c'
	fmt.Printf("c1 = %v type = %T \nc2 = %v type = %T\n", c1, c1, c2, c2)

	s := "hello 世界"
	// for range 循环可以遍历 字符加符号的
	for _, v := range s {
		fmt.Printf("%c \n", v)
	}
}

```


