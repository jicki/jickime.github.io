# Go学习笔记


# Go语言基础

## 函数

* `函数` 是组织好的、可重复使用的、用于执行指定任务的代码块。
* Go语言中支持`函数`、`匿名函数`和`闭包`, 并且函数在Go语言中属于 "一等公民" 。

### 函数定义

* Go语言中定义函数使用`func`关键字.

```shell
// 函数定义 (参数 与 返回值 可以是多个)
func 函数名(参数)(返回值){
    函数体
}

```

* 名词解析:
  1. 函数名:  由字母、数字、下划线组成。但函数名的第一个字母不能是数字。在同一个包内, 函数名也称不能重名。
  2. 参数: 参数由`参数变量`和`参数变量类型`组成, 多个参数之间使用`,`分隔。
  3. 返回值: 返回值由`返回值变`量和其`变量类型`组成, 也可以只写返回值的类型, 多个返回值必须用`()`包裹, 并用`,`分隔。
  4. 函数体: 实现指定功能的代码块。

```go
// 不包含参数与返回值的函数
func sayHello() {
	fmt.Println("Hello ~")
}

func main() {
	// 函数的调用
	sayHello()
}
```

* 输出:

```go
Hello ~
```

### 函数参数

* 函数参数由`参数变量`和`参数变量类型`组成, 如果相邻变量的类型相同, 则可以省略类型。
* 函数参数类型相同时可以简写, 如: `func sum(x, y int)`
* Go语言是没有 默认参数的

```go
// 带参数的函数
func sayHi(name string) {
	fmt.Println("Hi ", name)
}
func main() {
	// 带参数的函数调用
	sayHi("康师傅")
}
```

* 输出:

```go
Hi  康师傅
```

```go
// 带多个参数和返回值的函数
func intSum(x, y int) int {
    sum := x + y
    // 使用 return 关键字 返回这个值
	return sum
}

func main() {
    // 多个参数和返回值函数调用
	ret := intSum(10, 11)
	fmt.Println(ret)
}
```

* 输出:

```go
21
```

### 函数可变参数

* `可变参数` 是指函数的参数数量不固定, Go语言中的可变参数通过在参数名后加`...`来标识。
* `可变参数` 要放在参数最后面, 如: `func a(x int, y...int)`

```go
// 带可变参数的函数
// 可变参数的类型为 切片
func allSum(a ...int) int {
	sum := 0
	// 遍历 a 切片中的元素
	for _, arg := range a {
		// 将所有传入的参数相加
		sum = sum + arg
	}
	return sum
}

func main() {
	// 可变参数的函数调用
	r1 := allSum()
	r2 := allSum(10, 20)
	r3 := allSum(20, 30, 40)
	fmt.Println(r1)
	fmt.Println(r2)
	fmt.Println(r3)
}

```

* 输出:

```go
0
30
90
```

### 函数返回值

* Go语言中通过`return`关键字向外输出返回值。
* Go语言中函数支持多返回值, 函数如果有多个返回值时必须用`()`将所有返回值包裹起来。
* 函数定义时可以给返回值命名, 并在函数体中直接使用这些变量, 最后通过`return`关键字返回。
* 多返回值也支持类型简写

```go
// 带多个返回值的函数
func calc(x, y int) (sub, sum int) {
	sum = x + y
	sub = x - y
	return
}

func main() {
	// 多返回值的函数调用 (有多少个返回值就必须用多少个参数去接收)
	sub, sum := calc(100, 200)
	fmt.Println(sub, sum)
}
```

* 输出:

```go
-100 300
```

### defer语句

* Go语言中的`defer`语句会将其后面跟随的语句进行延迟处理。在`defer`归属的函数即将返回时, 将延迟处理的语句按`defer`定义的逆序进行执行, 也就是说, 先被`defer`的语句最后被执行, 最后被`defer`的语句，最先被执行。

* 由于`defer`语句延迟调用的特性，所以`defer`语句能非常方便的处理资源释放问题。比如: 资源清理、文件关闭、解锁及记录时间等。

* 在Go语言的函数中`return`语句在底层并不是原子操作, 它分为给返回值赋值和RET指令两步。而`defer`语句执行的时机就在返回值赋值操作后, RET指令执行前。

```shell

// 函数 return 语句 底层实现

                [返回值 = x]
[return x] ->       ⇊
                [RET 指令]

// defer 语句执行
                [返回值 = x]
                    ⇊
[return x] ->   [运行 defer]
                    ⇊
                [RET 指令]
```

```go
// defer 延迟执行
func main() {
	// 多个 defer 是遵循 先入后出 的顺序
	fmt.Println("Start --")
	defer fmt.Println("第一条")
	defer fmt.Println("第二条")
	defer fmt.Println("第三条")
	fmt.Println("End --")
}

```

* 输出:

```go
Start --
End --
第三条
第二条
第一条
```

### defer 题目分析

```go
// defer 的题目
func calc(index string, a, b int) int {
	ret := a + b
	fmt.Println(index, a, b, ret)
	return ret
}

func main() {
    // defer 题目
    // defer 先入后出 的顺序
	x := 1
	y := 2
	// 第一个参数为 "AA" , 第二个参数为 x = 1
	// 第三个参数为 calc("A", 1, 2) 的返回值 = 3
	// 最后整体的返回值是 1 + 3 = 4
	// 输出 AA 1 3 4
    // calc("A", x, y) 这里的输出为 A 1 2 3
	defer calc("AA", x, calc("A", x, y))
	x = 10
	// 第一个参数为 "BB", 第二个参数为 x = 10
	// 第三个参数为 calc("B", 10, 2) 的返回值 = 12
	// 最后整体的返回值是 10 + 12 = 22
	// 输出 BB 10 12 22
	// calc("B", x, y) 这里输出为 B 10 2 12
	defer calc("BB", x, calc("B", x, y))
	y = 20
}
```

* 输出:

```go
A 1 2 3
B 10 2 12
BB 10 12 22
AA 1 3 4
```

### 函数 变量作用域

* 函数 `全局变量`与`局部变量`同时存在时, 优先使用 函数内定义的`局部变量`, 然后在使用`全局变量`。

```go
// 定义全局变量
var intG = 10

func testFunc() {
	// 定义一个局部变量
	intG := 100
	// 输出这个变量 (函数会优先使用 局部变量)
	fmt.Println(intG)
}

func main() {
	// 调用函数
	testFunc()
}
```

* 输出:

```go
100
```

### 函数类型与变量

* 函数类型变量

```go

func testFunc() {
	// 定义一个局部变量
	intG := 100
	// 输出这个变量
	fmt.Println(intG)
}

func main() {
	// 调用函数
	testFunc()
	// 函数可以作为一个变量来赋值
	abc := testFunc
	fmt.Printf("%T \n", abc)
	abc()
}
```

* 输出:

```go
func()
100
```

### 高阶函数

* 高阶函数分为`函数作为参数`和`函数作为返回值`两部分。

* 函数作为参数

```go
func sum(x, y int) int {
	return x + y
}

func sub(x, y int) int {
	return x - y
}

// 函数变量也可以是 函数类型
func calc(x, y int, op func(int, int) int) int {
	return op(x, y)
}

func main() {
	// 将 sum 这个函数当成 calc 的变量传入
	r1 := calc(100, 200, sum)
	fmt.Println(r1)
	// 将 sub 这个函数当成 calc 的变量传入
	r2 := calc(100, 200, sub)
	fmt.Println(r2)
}
```

* 输出:

```go
300
-100
```

* 函数作为返回值

```go
func sum(x, y int) int {
	return x + y
}

func sub(x, y int) int {
	return x - y
}

// 函数作为函数的返回值传入函数中
func do(s string) (func(int, int) int, error) {
	switch s {
	case "+":
		return sum, nil
	case "-":
		return sub, nil
	default:
		err := errors.New("无法识别的操作符")
		return nil, err
	}
}

func main() {
	r2, err := do("+")
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(r2(100, 200))
	r3, err := do("-")
	if err != nil {
		fmt.Println(err)
	}
	fmt.Println(r3(400, 200))
}
```

* 输出

```go
300
200
```

### 匿名函数和闭包

* 匿名函数 - 匿名函数就是没有函数名的函数。
* 匿名函数多用于实现回调函数和闭包。

```go
func(参数)(返回值){
    函数体
}
```

```go
// 匿名函数
func main() {
	// 定义匿名函数与调用
	// 1. 匿名函数调用,赋值给变量
	r1 := func() {
		fmt.Println("匿名函数1")
	}
	r1()
	// 2. 匿名函数定义的时候就直接调用
	func() {
		fmt.Println("匿名函数2")
	}()
}
```

* 输出:

```go
匿名函数1
匿名函数2
```

### 闭包

* 闭包指的是一个函数和与其相关的引用环境组合而成的实体。简单来说: `闭包=函数+引用环境(外层变量的引用)`。

```go
// 函数闭包
// 定义一个函数,它的返回值是一个函数类型
func a() func() {
	name := "闭包"
	//把函数作为返回值,并传入一个变量
	return func() {
		fmt.Println("func a:", name)
	}
}
func main() {
	// 将a()这个函数赋值给 变量r1
	// 闭包 = "函数 + 外层变量的引用" 所以 r1 就是一个闭包
	r1 := a()
	// 此时调用r1 这个变量 相当于调用 a()函数内的匿名函数
	r1()
}
```

* 输出:

```go
func a: 闭包
```

```go
// 定义一个函数并且包含一个变量name,返回值是一个函数类型
func b(name string) func() {
	return func() {
		fmt.Println("func b:", name)
	}
}

func main() {
	// 此时调用 r2 这个变量 加上参数
	r2 := b("闭包b")
	r2()
}
```

* 输出:

```go
func b: 闭包b
```

* 闭包的应用一

```go
// 定义一个函数 返回值是一个匿名函数
// 匿名函数包含一个参数和一个返回值
func suffixFunc(suffix string) func(string) string {
	// 返回 这个匿名函数
	return func(name string) string {
		// 如果 name 参数 不包含 suffix 这个上层参数
		if !strings.HasSuffix(name, suffix) {
			// 返回 name 参数 + suffix 参数
			return name + suffix
		}
		// 如果包含就直接返回这个 name 参数
		return name
	}
}

func main() {
	//suffixFunc 赋值给 变量 ret, suffixFunc 传入一个参数
	ret := suffixFunc(".txt")

	// 调用 suffixFunc 内部的匿名函数
	r1 := ret("小说")
	fmt.Println(r1)

	ret = suffixFunc(".avi")
	r2 := ret("视频")
	fmt.Println(r2)
}
```

* 输出:

```go
小说.txt
视频.avi
```

* 闭包的应用二

```go
// 闭包的应用 2

// 定义一个函数,有一个变量和两个函数返回值
func calc(base int) (func(int) int, func(int) int) {
	// 定义个匿名函数有一个参数和一个返回值
	sum := func(i int) int {
		// 闭包: 引用外层变量 base
		// base = base + i
		base += i
		// 返回 base
		return base
	}
	// 定义个匿名函数有一个参数和一个返回值
	sub := func(i int) int {
		// 闭包: 引用外层变量 base
		// base = sum 函数的返回值 base - i
		base -= i
		// 返回 base
		return base
	}
	// 返回如上两个匿名函数的变量
	return sum, sub
}

func main() {
	//调用函数 有两个返回值x,y 并传入calc参数 base = 200
	// x = sum 这个匿名函数
	// y = sub 这个匿名函数
	x, y := calc(200)
	// 调用 x 这个匿名函数,传入一个参数100
	ret1 := x(100)
	// 调用 y 这个匿名函数,传入一个参数200
	ret2 := y(200)
	// 分别打印
	fmt.Printf("x:sum = %d  y:sub = %d \n", ret1, ret2)
}
```

* 输出:

```go
x:sum = 300  y:sub = 100
```

### 内置函数

|内置函数|说明|
|-|-|
|close|主要用来关闭channel|
|len|用来求长度，比如string、array、slice、map、channel|
|new|用来分配内存，主要用来分配值类型，比如int、struct。返回的是指针|
|make|用来分配内存，主要用来分配引用类型，比如chan、map、slice|
|append|用来追加元素到数组、slice中|
|panic和recover|用来做错误处理|

* panic/recover
  * Go语言中 版本小于等于 `1.12` 是没有异常机制, 但是使用`panic/recover`模式来处理错误。`panic`可以在任何地方引发。
  * `recover()`必须搭配`defer`使用。
  * `defer`一定要在可能引发`panic`的语句之前定义。

* 引发 panic

```go
func a() {
	fmt.Println("func in a")
}

func b() {
	panic("panic in b")
}

func c() {
	fmt.Println("func in c")
}

// panic && recover
func main() {
	// 分别调用 a b c 函数
	a()
	b()
	c()
}
```

* 输出:

```go
func in a
panic: panic in b

goroutine 1 [running]:
main.b(...)
        /Users/jicki/go/src/github.com/jicki/panic_recover/panic.go:10
main.main()
        /Users/jicki/go/src/github.com/jicki/panic_recover/panic.go:21 +0x96
exit status 2
```

* 使用 recover 恢复错误

```go
func a() {
	fmt.Println("func in a")
}

func b() {
	// recover 必须跟 defer 配合使用
	// 定义一个匿名函数,使用 recover 尝试恢复错误
	defer func() {
		err := recover()
		if err != nil {
			fmt.Println("recover error in b")
		}
	}()
	// recover 必须要在可能引发 panic 之前执行
	panic("panic in b")
}

func c() {
	fmt.Println("func in c")
}

// panic && recover
func main() {
	// 分别调用 a b c 函数
	a()
	b()
	c()
}

```

* 输出:

```go
func in a
recover error in b
func in c
```

### 函数练习题

```go
/*
你有50枚金币，需要分配给以下几个人：Matthew,Sarah,Augustus,Heidi,Emilie,Peter,Giana,Adriano,Aaron,Elizabeth。
分配规则如下：
a. 名字中每包含1个'e'或'E'分1枚金币
b. 名字中每包含1个'i'或'I'分2枚金币
c. 名字中每包含1个'o'或'O'分3枚金币
d: 名字中每包含1个'u'或'U'分4枚金币
写一个程序，计算每个用户分到多少金币，以及最后剩余多少金币？
程序结构如下，请实现 ‘dispatchCoin’ 函数
*/
```

```go
var (
	coins        = 50
	users        = []string{"Matthew", "Sarah", "Augustus", "Heidi", "Emilie", "Peter", "Giana", "Adriano", "Aaron", "Elizabeth"}
	distribution = make(map[string]int, len(users))
)

func dispatchCoin() int {
	// 遍历 users 切片,获取每个名字的元素
	for _, name := range users {
		nameCoin := 0
		// 遍历切片中的元素,获取到每个元素的字母的 byte
		for _, str := range name {
			// 循环判断 str 中的 byte
			switch str {
			case 'e', 'E':
				nameCoin += 1
			case 'i', 'I':
				nameCoin += 2
			case 'o', 'O':
				nameCoin += 3
			case 'u', 'U':
				nameCoin += 4
			}
		}
		// 将判断后的金币数量 赋值到 map 字典中
		distribution[name] = nameCoin
		// 金币总数 = 金币总数 - 每个元素的 nameCoin
		coins -= nameCoin
	}
	// 最后返回这个 金币 coins 总数
	fmt.Println(distribution)
	return coins
}

func main() {
	left := dispatchCoin()
	fmt.Println("剩下:", left)
}
```

* 输出:

```go
map[Aaron:3 Adriano:5 Augustus:12 Elizabeth:4 Emilie:6 Giana:2 Heidi:5 Matthew:1 Peter:2 Sarah:0]
剩下: 10
```

## 指针

* Go语言中的指针区别于C/C++中的指针, 不能进行偏移和运算, 是安全指针。
* Go语言中的指针需要先知道3个概念: 指针地址、指针类型和指针取值。
* Go语言中的函数传参都是值拷贝, 当我们想要修改某个变量的时候, 我们可以创建一个指向该`变量地址`的`指针变量`。传递数据使用指针, 而无须拷贝数据。`类型指针`不能进行偏移和运算。Go语言中的指针操作非常简单, 只需要记住两个符号: `&（取地址）` 和 `*（根据地址取值）`。
* 取地址操作符`&`和取值操作符 `*`是一对互补操作符, `&`取出地址, `*`根据地址取出地址指向的值。
* 变量、指针地址、指针变量、取地址、取值的相互关系和特性如下:
  1. 对变量进行取地址 `&` 操作, 可以获得这个变量的指针变量。
  2. 指针变量的值是指针地址。
  3. 对指针变量进行取值`*`操作, 可以获得指针变量指向的原变量的值。

### 指针地址和指针类型

* 每个变量在运行时都拥有一个地址, 这个地址代表变量在内存中的位置。Go语言中使用`&`字符放在变量前面对变量进行"取地址"操作。 Go语言中的值类型（int、float、bool、string、array、struct）都有对应的指针类型, 如: `*int`、`*int64`、`*string` 等。

* 取变量指针的语法: `ptr := &v`
  * v: 代表被取地址的变量, 类型为T
  * ptr: 用于接收地址的变量, ptr的类型就为*T, 称做T的指针类型。*代表指针。

```go
// 指针
func main() {
	// a int类型
	a := 100
	// & 取地址 b *int类型 (int类型的指针)
	b := &a
	// a 的值是 10 , a 的指针是 a 自己的指针
	fmt.Printf("a 的值: [%v] a 的指针: [%p]\n", a, &a)
	// b 的值是 a 的指针, b 的指针是 b 自己的地址
	fmt.Printf("b 的值: [%v] b 的指针: [%p]\n", b, &b)
}
```

* 输出:

```go
a 的值: [100] a 的指针: [0xc0000aa008]
b 的值: [0xc0000aa008] b 的指针: [0xc0000a6018]
```

### 指针取值

* 在对普通变量使用`&`操作符取地址后会获得这个变量的指针, 然后可以对指针使用`*`操作，也就是指针取值。

```go
func main() {
	// a int类型
	a := 100
	// & 取地址 b *int类型 (int类型的指针)
	b := &a
	fmt.Printf("a 的值: [%v] a 的指针: [%p]\n", a, &a)
	fmt.Printf("b 的值: [%v] b 的指针: [%p]\n", b, &b)
	// 指针取值  c 的值 等于 b 指针的值
	c := *b
	fmt.Printf("c 的值: [%v] c 的指针: [%p]\n", c, &c)
}
```

* 输出:

```go
a 的值: [100] a 的指针: [0xc0000aa008]
b 的值: [0xc0000aa008] b 的指针: [0xc0000a6018]
c 的值: [100] c 的指针: [0xc0000aa018]
```

### 指针传值

```go
// 定义一个函数包含一个int类型的参数 x
func modify1(x int) {
	// 将 x 修改为 100
	x = 100
}

// 定义一个函数包含一个*int (指针类型)类型的参数 y
func modify2(y *int) {
	// 将 指针地址的值 修改为 100
	*y = 100
}

func main() {
	// a int类型
	a := 10

	// 调用 modify1 将 a 传到 函数中
	modify1(a)
	fmt.Printf("modify1 = %d \n", a)

	// 调用 modify2 将 a 的地址传到 函数中
	modify2(&a)
	fmt.Printf("modify2 = %d \n", a)

}

```

* 输出:

```go
modify1 = 10
modify2 = 100
```

### new和make

* Go语言中`new`和`make`是内建的两个函数, 主要用来分配内存。

* new
  * `new`是一个内置的函数，它的函数签名:  `func new(Type) *Type` , `Type`表示类型, `new`函数只接受一个参数, 这个参数是一个类型. `*Type`表示类型指针, `new`函数返回一个指向该类型内存地址的指针。
  * `new`函数不太常用, 使用`new`函数得到的是一个类型的指针,并且该指针对应的值为该类型的零值。

```go
func main() {
	a := new(int)
	b := new(bool)
	fmt.Printf("a 的类型: %T a 的值: %v \n", a, *a)
	fmt.Printf("b 的类型: %T b 的值: %v \n", b, *b)

	// 定义一个 指针类型 的变量 c
	var c *int
	// 定义了以后需要使用 new 来初始化这个变量
	c = new(int)
	// 不初始化会报 nil 空指针的错误
	*c = 100
	fmt.Println(*c)
}
```

* 输出:

```go
a 的类型: *int a 的值: 0
b 的类型: *bool b 的值: false
100
```

* `make` 也是用于内存分配的，区别于`new`, 它只用于`slice`、`map`以及`chan`的内存创建, 而且它返回的类型就是这三个类型本身, 而不是他们的指针类型, 因为这三种类型就是引用类型, 所以就没有必要返回他们的指针了。
* `make` 函数的函数签名 `func make(t Type, size ...IntegerType) Type` 
* `make` 函数是无可替代的, 我们在使用`slice`、`map`以及`channel`的时候, 都需要使用`make`进行初始化, 然后才可以对它们进行操作。

```go
func main() {
	// make
	//  定义一个 map
	var m map[string]int
	// 使用 make 进行初始化
	m = make(map[string]int, 10)
	// 必须初始化以后才能操作,否则报错
	m["Map"] = 1
	fmt.Println(m)
}
```

* 输出:

```go
map[Map:1]
```

### new 与 make 的区别

1. 二者都是用来做内存分配的。
2. `make`只用于`slice`、`map`以及`channel`的初始化, 返回的还是这三个引用类型本身。
3. `new`用于类型的内存分配, 并且内存对应的值为类型零值, 返回的是指向类型的指针。


