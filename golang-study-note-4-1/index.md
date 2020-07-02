# Go学习笔记


# Go语言基础

## 类型别名和自定义类型

### 自定义类型

* 在Go语言中有一些基本的数据类型, 如`string`、`整型`、`浮点型`、`布尔`等数据类型, Go语言中可以使用`type`关键字来定义自定义类型。

* 自定义类型是定义了一个全新的类型。我们可以基于内置的基本类型定义, 也可以通过`struct`定义。

* 类型定义和类型别名的区别: 自定义类型,定义以后输出的类型会输出为 新定义的类型, 而类型别名,在定义以后输出的类型仍然是原类型。

```go
// Myint 基于 int 的自定义类型
type Myint int

func main() {
    // 定义一个 i 变量 类型为 Myint 类型
	var i Myint
	fmt.Printf("type: %T value: %v \n", i, i)
}
```

* 输出:

```go
type: main.Myint value: 0
```

### 类型别名

* 类型别名规定: `TypeAlias`只是`Type`的别名, 本质上`TypeAlias`与`Type`是同一个类型。
* 类型别名定义: `type TypeAlias = Type`。
* `rune`和`byte`就是类型别名, 它们其实是 `type byte = uint8`, `type rune = int32`

```go
// 类型别名
// Aliasint 是 int 的类型别名
type Aliasint = int

func main(){
	// 定义一个 a 变量 类型为 Aliasint 类型
	var a Aliasint
	fmt.Printf("type: %T value: %v \n", a, a)
}
```

* 输出:

```go
type: int value: 0
```

## 结构体

* Go语言中的基础数据类型可以表示一些事物的基本属性, 但是当我们想表达一个事物的全部或部分属性时, 这时候再用单一的基本数据类型明显就无法满足需求了, Go语言提供了一种自定义数据类型, 可以封装多个基本数据类型, 这种数据类型叫`结构体`, 英文名称`struct`。 也就是我们可以通过`struct`来定义自己的类型了。

* Go语言中通过`struct`来实现面向对象。

### 结构体的定义

* 使用`type`和`struct`关键字来定义结构体。

```go
type 类型名 struct {
    字段名 字段类型
    字段名 字段类型
    …
}
```

* 说明:
  * 类型名: 标识自定义结构体的名称, 在同一个包内不能重复。
  * 字段名: 表示结构体字段名。结构体中的字段名必须唯一。
  * 字段类型: 表示结构体字段的具体类型。

```go
// 定义 struct 结构体
type person struct {
	name, city string
	age        int8
}
```

* 如上定义了一个`person`的自定义类型, 它有`name`、`city`、`age`三个字段, 分别表示姓名、城市和年龄。这样我们使用这个person结构体就能够很方便的在程序中表示和存储人信息了。

* 语言内置的基础数据类型是用来描述一个值的, 而结构体是用来描述一组值的。比如一个人有名字、年龄和居住城市等, 本质上是一种聚合型的数据类型。

### 结构体实例化

* 只有当结构体实例化时, 才会真正地分配内存。也就是必须实例化后才能使用结构体的字段。

* 结构体本身也是一种类型, 我们可以像声明内置类型一样使用`var`关键字声明结构体类型。 如: `var 结构体实例 结构体类型`

```go
type person struct {
	name, city string
	age        int8
}

func main() {
	// 定义了一个 p1 变量,类型为 person 类型
	var p1 person
	// 通过 . 的形式进行 赋值
	p1.name = "盘古"
	p1.city = "宇宙"
	p1.age = 99
	fmt.Printf("%#v \n", p1)
	// 通过 . 的形式进行 访问
	fmt.Printf("名字: %v 城市: %v 年龄: %v \n", p1.city, p1.city, p1.age)
}
```

* 输出:

```go
main.person{name:"盘古", city:"宇宙", age:99}

名字: 宇宙 城市: 宇宙 年龄: 99
```

### 匿名结构体

* 在定义一些临时数据结构等场景下可以使用匿名结构体。

```go
func main() {
	// 定义一个匿名结构体
	var p2 struct {
		name string
		age  int8
	}
	p2.name = "女娲"
	p2.age = 99
	fmt.Printf("%#v \n", p2)
}
```

* 输出:

```go
struct { name string; age int8 }{name:"女娲", age:99}
```

### 指针类型的结构体

* 通过使用`new`关键字对结构体进行实例化, 得到的是结构体的内存地址。

```go
//结构体指针

type person struct {
	name, city string
	age        int8
}

func main() {
	// 定义一个结构体指针
	var p3 = new(person)
	fmt.Printf("%T \n", p3)
	// 结构体可以直接对结构体的指针使用,不需要使用*符号
	p3.name = "小企鹅"
	p3.city = "北极"
	p3.age = 5
	fmt.Printf("%#v \n", p3)
}
```

* 输出:

```go
*main.person
&main.person{name:"小企鹅", city:"北极", age:5}
```

### 取结构体的地址实例化

* 使用`&`对结构体进行取地址操作相当于对该结构体类型进行了一次`new`实例化操作。

```go
type person struct {
	name, city string
	age        int8
}
func main() {
	// 取结构体地址进行实例化
	p4 := &person{}
	fmt.Printf("p4 = %T \n", p4)
	p4.name = "北极熊"
	p4.city = "北极"
	p4.age = 10
    fmt.Printf("%#v \n", p4)
}
```

* 输出:

```go
p4 = *main.person
&main.person{name:"北极熊", city:"北极", age:10}
```

### 结构体初始化

* 没有初始化的结构体, 其成员变量都是对应其类型的零值。
  1. 使用键值对初始化 - 使用键值对对结构体进行初始化时, 键对应结构体的字段, 值对应该字段的初始值。也可以对结构体指针进行键值对初始化。当某些字段没有初始值的时候, 该字段可以不写。此时, 没有指定初始值的字段的值就是该字段类型的零值。
  2. 使用值的列表初始化 - 初始化结构体的时候可以简写, 也就是初始化的时候不写键, 直接写值。但是需要注意的是 (1. 必须初始化结构体的所有字段。2. 初始值的填充顺序必须与字段在结构体中的声明顺序一致。3. 方式不能和键值初始化方式混用。)

* 键值对初始化

```go
type person struct {
	name, city string
	age        int
}

func main() {
	// 1. 键值对初始化
	p5 := person{
		name: "玉帝",
		city: "天宫",
		age:  9999,
	}
	fmt.Printf("%#v \n", p5)
	// 取地址初始化
	p6 := &person{
		name: "王母",
		city: "天宫",
		age:  9998,
	}
	fmt.Printf("%#v \n", p6)
}
```

* 输出:

```go
main.person{name:"玉帝", city:"天宫", age:9999}
&main.person{name:"王母", city:"天宫", age:9998}
```

* 值的列表初始化

```go
type person struct {
	name, city string
	age        int
}
func main() {
    // 2. 值的列表初始化
    // 值列表初始化 必须按照 struct 定义的顺序写, 必须写全部字段
	p7 := person{
		"二郎神",
		"天宫",
		999,
	}
	fmt.Printf("%#v \n", p7)
}
```

* 输出:

```go
main.person{name:"二郎神", city:"天宫", age:999}
```

### 结构体内存布局

* 结构体占用一块连续的内存。

```go
type test struct {
	a int8
	b int8
	c int8
	d int8
}
func main() {
	n := test{
		1, 2, 3, 4,
	}
	fmt.Printf("n.a %p\n", &n.a)
	fmt.Printf("n.b %p\n", &n.b)
	fmt.Printf("n.c %p\n", &n.c)
	fmt.Printf("n.d %p\n", &n.d)
}
```

* 输出:

```go
n.a 0xc0000a0004
n.b 0xc0000a0005
n.c 0xc0000a0006
n.d 0xc0000a0007
```

### 结构体的匿名字段

* 结构体允许其成员字段在声明时没有字段名而只有类型, 这种没有名字的字段就称为匿名字段。
* 匿名字段不能有重复的类型.

```go
// 结构体的匿名字段
type Person struct {
	string
	int
}

func main() {
	// 调用以及访问结构体的匿名字段
	p1 := Person{
		"张大仙",
		18,
	}
	// 通过访问 类型名字 来访问字段
	fmt.Println(p1.string, p1.int)
}
```

* 输出:

```go
张大仙 18
```

### 嵌套结构体

* 一个结构体中可以嵌套包含另一个结构体或结构体指针。

```go
// 嵌套结构体
type Address struct {
	province, city string
}

type Person struct {
	name, gender string
	age          int
	// 嵌套 Address 结构体
	// 第一个Address是名称 第二个Address是类型
	Address Address
}

func main() {
	//嵌套结构体的初始化
	p1 := Person{
		name:   "张大仙",
		gender: "女",
		age:    30,
		Address: Address{
			province: "广东",
			city:     "深圳",
		},
	}
	fmt.Printf("%#v \n", p1)
	//嵌套结构体的字段访问
	fmt.Println(p1.name, p1.Address)
}
```

* 输出:

```go
main.Person{name:"张大仙", gender:"女", age:30, Address:main.Address{province:"广东", city:"深圳"}}
张大仙 {广东 深圳}
```

### 嵌套匿名结构体

* 如上例子修改一下嵌套的结构体只写类型名

```go
// 嵌套结构体
type Address struct {
	province, city string
}

type Person struct {
	name, gender string
	age          int
	// 嵌套 匿名结构体
	// 只写类型名
	Address
}

func main() {
	//嵌套结构体的初始化
	p1 := Person{
		name:   "张大仙",
		gender: "女",
		age:    30,
		Address: Address{
			province: "广东",
			city:     "深圳",
		},
	}
	fmt.Printf("%#v \n", p1)
	//嵌套匿名结构体的访问, 可直接访问匿名结构体里的字段
	// 如下两种方式都可以访问结构体里的字段
	fmt.Println(p1.Address.city, p1.city)
}
```

* 输出:

```go
main.Person{name:"张大仙", gender:"女", age:30, Address:main.Address{province:"广东", city:"深圳"}}
深圳 深圳
```

* 匿名结构体字段冲突

```go
package main

import "fmt"

// 嵌套匿名结构体字段冲突
type Address struct {
	province, city string
	updateTime     string
}
type Email struct {
	addr       string
	updateTime string
}

type Person struct {
	name, gender string
	age          int
	// 嵌套 匿名结构体 Address
	Address
	// 嵌套 匿名结构体 Email
	Email
}

func main() {
	// 初始化一个Person 实例
	p1 := Person{
		name:   "张大仙",
		gender: "女",
		age:    20,
		Address: Address{
			province:   "广东",
			city:       "深圳",
			updateTime: "2019-10-30",
		},
		Email: Email{
			addr:       "zhangdaxian@huya.com",
			updateTime: "2019-10-29",
		},
	}

	// 当匿名结构体存在冲突字段的时候,需要区分
	fmt.Println("Address:", p1.Address.updateTime)
	fmt.Println("Email:", p1.Email.updateTime)
}

```

* 输出:

```go
Address: 2019-10-30
Email: 2019-10-29
```

### 构造函数

* Go语言的结构体是没有构造函数, 我们可以自己实现。构造函数就是 构造一个结构体实例的函数。
* 如下实现了一个person的构造函数。 因为struct是值类型, 如果结构体比较复杂的话, 值拷贝性能开销会比较大, 所以该构造函数返回的是结构体指针类型。

```go
type person struct {
	name, city string
	age        int
}

// 构造函数

// 构造函数一般约定名称前为 new 开头
// 这里返回值使用的是 取 person 内存地址指针的值
func newPerson(name, city string, age int) *person {
    // 返回 内存地址的 person
	return &person{
		name: name,
		city: city,
		age:  age,
	}
}
func main() {
	// 构造多个 person
	p8 := newPerson("托塔天王", "天宫", 999)
    fmt.Printf("%#v \n", p8)
    p9 := newPerson("太白金星", "天宫", 9999)
	fmt.Printf("%#v \n", p9)
}
```

* 输出:

```go
&main.person{name:"托塔天王", city:"天宫", age:999}
&main.person{name:"太白金星", city:"天宫", age:9999}
```

### 结构体练习题

```go
type student struct {
	name string
	age  int
}

func main() {
	// 定义一个 map key 值为 string  value 为 struct
	m := make(map[string]*student)
	// 定义一个 类型为 struct 的切片,里面有三组元素
	stus := []student{
		{name: "小王子", age: 18},
		{name: "娜扎", age: 23},
		{name: "大王八", age: 9000},
	}

	for _, stu := range stus {
		// stu = {小王子 18}  {娜扎 23} {大王八 9000}
		// stu.name 分别 = 小王子, 娜扎, 大王八
		// &stu 是取地址, slice 是引用类型, 指向同一个内存地址
		m[stu.name] = &stu
		// 所以这里 map[大王八:0xc000092020 娜扎:0xc000092020 小王子:0xc000092020]
	}

	for k, v := range m {
		// m = map[大王八:0xc000092020 娜扎:0xc000092020 小王子:0xc000092020]
		// k = "大王八", "娜扎", "小王子"
		// v = "0xc000092020" 同一个内存地址
		fmt.Println(k, "=>", v.name)
	}
}
```

* 输出:

```go
小王子 => 大王八
娜扎 => 大王八
大王八 => 大王八
```

## 方法和接收者

* Go语言中的方法`（Method）`是一种作用于特定类型变量的函数。这种特定类型变量叫做接收者`（Receiver)` , 接收者的概念就类似于其他语言中的`this`或者`self`。

* 方法与函数的区别是, 函数不属于任何类型, 方法属于特定的类型。

* 方法的定义格式:
  * 接收者变量: 接收者中的参数变量名在命名时, 官方建议使用接收者类型名的第一个小写字母, 而不是self、this之类的命名。例如, `Person`类型的接收者变量应该命名为 `p`, `Connector`类型的接收者变量应该命名为`c`等。
  * 接收者类型: 接收者类型和参数类似, 可以是指针类型和非指针类型。
  * 方法名、参数列表、返回参数: 具体格式与函数定义相同。

```go
func (接收者变量 接收者类型) 方法名(参数列表) (返回参数) {
    函数体
}
```

```go
// 方法与接受者

// Person 定义一个结构体
type Person struct {
	name string
	age  int8
}

// NewPerson  Person 结构体的构造函数
func NewPerson(name string, age int8) *Person {
	return &Person{
		name: name,
		age:  age,
	}
}

// Dream 为Person类型定义的方法
func (p Person) Dream() {
	fmt.Printf("%v 的梦想是做一条咸鱼 \n", p.name)
}

func main() {
	// 实例化 Person
	p1 := NewPerson("八戒", 20)
	// 调用方法
	p1.Dream()
}
```

* 输出:

```go
八戒 的梦想是做一条咸鱼
```

### 指针类型的接收者

* 指针类型的接收者由一个结构体的指针组成, 由于指针的特性, 调用方法时修改接收者指针的任意成员变量, 在方法结束后, 修改都是有效的。这种方式就十分接近于其他语言中面向对象中的`this`或者`self`。

* 例如 我们为Person添加一个SetAge方法, 来修改实例变量的年龄。

```go
// Person 定义一个结构体
type Person struct {
	name string
	age  int8
}

// NewPerson  Person 结构体的构造函数
func NewPerson(name string, age int8) *Person {
	return &Person{
		name: name,
		age:  age,
	}
}

// 指针接受者 接受者的类型是指针类型
// SetAge 修改age 的方法
func (p *Person) SetAge(newAge int8) {
	p.age = newAge
}

func main() {
	// 实例化 Person
	p1 := NewPerson("八戒", 20)
	// 修改前的值
	fmt.Println(p1.age)
	// 调用指针类型的方法
	p1.SetAge(99)
	// 修改后的值
	fmt.Println(p1.age)
}

```

* 输出:

```go
20
99
```

### 值接收者

* 当方法作用于值类型接收者时, Go语言会在代码运行时将接收者的值复制一份。在值类型接收者的方法中可以获取接收者的成员值, 但修改操作只是针对副本, 无法修改接收者变量本身。

* 函数与方法传参的时候都是复制了一份, 所以修改只是修改了复制的那一份, 没有修改到本身。

```go
// Person 定义一个结构体
type Person struct {
	name string
	age  int8
}

// NewPerson  Person 结构体的构造函数
func NewPerson(name string, age int8) *Person {
	return &Person{
		name: name,
		age:  age,
	}
}


// 值接收者 接收者的类型是值类型
// SetName 修改 name 的方法
func (p Person) SetName(newName string) {
	p.name = newName
}

func main() {
	// 实例化 Person
	p1 := NewPerson("八戒", 20)

	// 修改前的值
	fmt.Println(p1.name)
	// 调用值类型的修改方法
	p1.SetName("悟空")
	// 修改后的值
	fmt.Println(p1.name)
}
```

* 输出:

```go
八戒
八戒
```

* 什么时候应该使用指针类型接收者
  1. 需要修改接收者中的值
  2. 接收者是拷贝代价比较大的大对象
  3. 保证一致性, 如果有某个方法使用了指针接收者, 那么其他的方法也应该使用指针接收者。

### 任意类型添加方法

* 在Go语言中, 接收者的类型可以是任何类型, 不仅仅是结构体, 任何类型都可以拥有方法。
* 举个例子: 我们基于内置的`int`类型使用`type`关键字可以定义新的自定义类型, 然后可以为我们的自定义类型添加方法。
* 注意事项: 非本地类型不能定义方法, 也就是说我们不能给别的包的类型定义方法。

```go
// 任意类型添加方法

// 给任意类型定义方法必须是包内的类型
type myString string

// 为 myString 类型定义一个方法 sayHello
func (m myString) sayHello() {
	fmt.Println("Hello I'am myString!")
}

func main() {
	//定义一个m 类型是 myString
	m := myString("string")
	// 调用方法
	m.sayHello()
}
```

* 输出:

```go
Hello I'am myString!
```

### 结构体"继承"

* Go语言中是没有继承这个概念的, 但是使用结构体也可以实现其他编程语言中面向对象的继承。

```go
// 结构体的 "继承"
// 利用 结构体嵌套的方法实现
type Animal struct {
	Name string
}

// 定义一个 Animal Move 的方法
func (a *Animal) Move() {
	fmt.Printf("%s 走来走去 \n", a.Name)
}

// 定义一个 Dog 的结构体
type Dog struct {
	Feet int
	// 嵌套 匿名结构体 Animal的指针
	*Animal
}

// 定义一个 Dog wang 的方法
func (d Dog) Wang() {
	fmt.Printf("%s 汪汪汪的叫 \n", d.Name)
}

// 定义一个 Cat 的结构体
type Cat struct {
	Feet int
	// 嵌套 匿名结构体 Animal的指针
	*Animal
}

// 定义一个 Cat mao 的方法
func (c Cat) Mao() {
	fmt.Printf("%s 喵喵喵的叫 \n", c.Name)
}

func main() {
	// 实例化一个 dog
	dog1 := Dog{
		Feet: 4,
		Animal: &Animal{
			Name: "旺财",
		},
	}
	// 实例化一个 cat
	cat1 := Cat{
		Feet: 4,
		Animal: &Animal{
			Name: "小花",
		},
	}
	// 调用方法
	dog1.Wang()
	cat1.Mao()
	// Dog 与 Cat 也可以调用 嵌套结构的方法 Move
	// 这就相当于实现了 继承
	dog1.Move()
	cat1.Move()
}
```

* 输出:

```go
旺财 汪汪汪的叫
小花 喵喵喵的叫
旺财 走来走去
小花 走来走去
```

### 结构体字段的可见性

* 结构体中字段大写开头表示可公开访问, 小写表示私有（仅在定义当前结构体的包中可访问）。

```go
type Person struct {
	// 字段首字母大写开头的表示外部可访问
	Name string
	Age int
}
```

### 结构体JSON序列化

* JSON `(JavaScript Object Notation)` 是一种轻量级的数据交换格式。易于人阅读和编写。同时也易于机器解析和生成。
* JSON键值对是用来保存`JS`对象的一种方式, `键/值` 对组合中的键名写在前面并用双引号`""`包裹，使用冒号`:`分隔, 然后紧接着值, 多个键值之间使用英文`,`分隔。

```go
package main

import (
	"encoding/json"
	"fmt"
)

// 定义一个 Student 的结构体
type student struct {
	ID   int
	Name string
}

// 定义一个 student 的构造函数
func newStudent(id int, name string) student {
	return student{
		ID:   id,
		Name: name,
	}
}

// 定义一个 class 的结构体
type class struct {
	Title string
	// 定义个student字段,类型是 student结构体类型的切片
	Students []student
}

// 结构体 JSON 序列化
func main() {
	// 实例化一个 class 的对象 c1
	c1 := class{
		Title: "王者荣耀",
		// student 字段是一个切片,所以需要先初始化才能使用
		Students: make([]student, 0, 20),
	}
	// 批量录入学生信息
	for i := 1; i < 10; i++ {
		tmpStu := newStudent(i, fmt.Sprintf("召唤师%02d", i))
		// 切片使用 append 函数追加 元素
		c1.Students = append(c1.Students, tmpStu)
	}
	fmt.Printf("%#v \n", c1)
	// 利用 json.Marshal 包序列化 这一组数据
	// 序列化因为调用的是外部的包，所以上面的 struct 跟 字段必须大写
	data, err := json.Marshal(c1)
	// 格式化 json 输出
	//data, err := json.MarshalIndent(c1, "", "")
	if err != nil {
		fmt.Println("json marshal failed err:", err)
		return
	}
	// data 变量保存的数据类型为 byte 类型(uint8)
	// %s 会转换成 string 类型打印
	fmt.Printf("%s \n", data)
}
```

* 输出:

```go
main.class{Title:"王者荣耀", Students:[]main.student{main.student{ID:1, Name:"召唤师01"}, main.student{ID:2, Name:"召唤师02"}, main.student{ID:3, Name:"召唤师03"}, ma:4, Name:"召唤师04"}, main.student{ID:5, Name:"召唤师05"}, main.student{ID:6, Name:"召唤师06"}, main.student{ID:7, Name:"召唤师07"}, main.student{ID:8, Name:"召唤师08t{ID:9, Name:"召唤师09"}}}

{"Title":"王者荣耀","Students":[{"ID":1,"Name":"召唤师01"},{"ID":2,"Name":"召唤师02"},{"ID":3,"Name":"召唤师03"},{"ID":4,"Name":"召唤师04"},{"ID":5,"Name":"召唤师05"},{"ID":6,"Name":"召唤师06"},{"ID":7,"Name":"召唤师07"},{"ID":8,"Name":"召唤师08"},{"ID":9,"Name":"召唤师09"}]}
```

### 结构体JSON 反序列化

* JSON格式的字符串 反序列化成 结构体

```go
// 定义一个 Student 的结构体
type student struct {
	ID   int
	Name string
}

// 定义一个 student 的构造函数
func newStudent(id int, name string) student {
	return student{
		ID:   id,
		Name: name,
	}
}

// 定义一个 class 的结构体
type class struct {
	Title string
	// 定义个student字段,类型是 student结构体类型的切片
	Students []student
}
// 结构体 JSON 反序列化
func main() {
	// Json 反序列化  Json 字符串 --> 结构体
	jsonStr := `{"Title": "王者荣耀","Students": [{"ID": 1,"Name": "召唤师01"},{"ID": 2,"Name": "召唤师02"}]}`
	// 定义一个 class 实例化对象 c2
	c2 := class{}
	// 这里需要传入 &c2 必须要传入指针, 否则无法写入数据
	err = json.Unmarshal([]byte(jsonStr), &c2)
	if err != nil {
		fmt.Println("json Unmarshal failed err:", err)
	}
	fmt.Printf("%#v \n", c2)
}

```

* 输出:

```go
main.class{Title:"王者荣耀", Students:[]main.student{main.student{ID:1, Name:"召唤师01"}, main.student{ID:2, Name:"召唤师02"}}}
```

### 结构体标签 (Tag)

* `Tag` 是结构体的元信息, 可以在运行的时候通过反射的机制读取出来。 `Tag`在结构体字段的后方定义, 由一对 `` 反引号包裹起来。

* 结构体标签由一个或多个键值对组成。键与值使用冒号`:`分隔, 值用双引号`""`括起来。键值对之间使用一个`空格`分隔。
* 注意事项: 为结构体编写`Tag`时, 必须严格遵守键值对的规则。结构体标签的解析代码的容错能力很差, 一旦格式写错, 编译和运行时都不会提示任何错误, 通过反射也无法正确取值。例如不要在`key`和`value`之间添加空格。

```go
// 定义一个 Student 的结构体
type student struct {
	ID   int
	Name string
}

// 定义一个 student 的构造函数
func newStudent(id int, name string) *student {
	return &student{
		ID:   id,
		Name: name,
	}
}

// 定义一个 class 的结构体
// 配置 tag, `输出格式:"tag名" 输出格式:"tag名"`
type class struct {
	Title string `json:"title" db:"title"`
	// 定义个student字段,类型是 student结构体类型的切片
	Students []student `json:"student_slice"`
}

func main() {
	// 实例化一个 class 的对象 c1
	c1 := class{
		Title: "King glory",
		// student 字段是一个切片,所以需要先初始化才能使用
		Students: make([]student, 0, 20),
	}
	// 批量录入学生信息
	for i := 1; i < 10; i++ {
		tmpStu := newStudent(i, fmt.Sprintf("player%02d", i))
		// 切片使用 append 函数追加 元素
		c1.Students = append(c1.Students, *tmpStu)
	}
	fmt.Printf("%#v \n", c1)
	// 利用 json.Marshal 包序列化 这一组数据
	// 序列化因为调用的是外部的包，所以上面的 struct 跟 字段必须大写
	data, err := json.Marshal(c1)
	// 格式化 json 输出
	//data, err := json.MarshalIndent(c1, "", "")
	if err != nil {
		fmt.Println("json marshal failed err:", err)
		return
	}
	// data 变量保存的数据类型为 byte 类型(uint8)
	// %s 会转换成 string 类型打印
	fmt.Printf("%s \n", data)
}
```

* 输出:

```go
main.class{Title:"King glory", Students:[]main.student{main.student{ID:1, Name:"player01"}, main.student{ID:2, Name:"player02"}, main.student{ID:3, Name:"player03"}, main.student{ID:4, Name:"player04"}, main.student{ID:5, Name:"player05"}, main.student{ID:6, Name:"player06"}, main.student{ID:7, Name:"player07"}, main.student{ID:8, Name:"player08"}, main.student{ID:9, Name:"player09"}}}
// 如下是打了 tag 以后 json 序列化后的输出
{"title":"King glory","student_slice":[{"ID":1,"Name":"player01"},{"ID":2,"Name":"player02"},{"ID":3,"Name":"player03"},{"ID":4,"Name":"player04"},{"ID":5,"Name":"player05"},{"ID":6,"Name":"player06"},{"ID":7,"Name":"player07"},{"ID":8,"Name":"player08"},{"ID":9,"Name":"player09"}]}

```

### 练习题

* 使用"面向对象"的思维方式编写一个学生信息管理系统。
  1. 学生有id、姓名、年龄、分数等信息
  2. 程序提供展示学生列表、添加学生、编辑学生信息、删除学生等功能

```go
# student.go

// 创建一个 student 的结构体
type student struct {
	id    int
	name  string
	class string
}

// 创建一个 student 的构造函数
func newStudent(id int, name, class string) *student {
	return &student{
		id:    id,
		name:  name,
		class: class,
	}
}

// 学员管理的结构体
type studentMgr struct {
	allStudents []*student
}

// 创建一个 studentMgr 的构造函数
func newStudentMgr() *studentMgr {
	return &studentMgr{
		allStudents: make([]*student, 0, 100),
	}
}

// 添加 学生的方法
func (s *studentMgr) addStudents(newStu *student) {
	// 使用 append 函数 往allStudents的切片中追加学生
	s.allStudents = append(s.allStudents, newStu)
}

// 编辑 学生的方法
func (s *studentMgr) editStudents(newStu *student) {
	for i, v := range s.allStudents {
		// 判断传入的 id 是否等于 学员学号
		if newStu.id == v.id {
			//根据切片的索引找到该学生信息,用新的学生信息覆盖
			s.allStudents[i] = newStu
			fmt.Println("修改学员成功")
			return
		}
	}
	fmt.Printf("没有找到 学号是: %d 的学生 \n", newStu.id)
}

// 删除 学生的方法
func (s *studentMgr) deleteStudents(newStu *student) {
	for i, v := range s.allStudents {
		// 判断传入的 id 是否等于 学员学号
		if newStu.id == v.id {
			// 利用append 函数删除切片中的元素 append(slice[:i],slice[i+1:]...)
			s.allStudents = append(s.allStudents[:i], s.allStudents[i+1:]...)
			fmt.Println("删除学员成功")
			return
		}
	}
	fmt.Printf("没有找到 学号是: %d 的学生 \n", newStu.id)
}

// 展示学生信息
func (s *studentMgr) showStudents() {
	// 使用 for range 循环 遍历 allStudents 切片
	for _, v := range s.allStudents {
		fmt.Printf("学号: %d 姓名: %s 班级: %s \n", v.id, v.name, v.class)
	}
}
```

```go
# main.go

/*
学员管理系统需求:
	1. 展示菜单
	2. 添加学员信息
	3. 编辑学员信息
	4. 展示所有学员信息
	5. 退出系统
*/
func showMenu() {
	fmt.Println("欢迎来到学员管理系统")
	fmt.Println("1. 添加学员信息")
	fmt.Println("2. 编辑学员信息")
	fmt.Println("3. 展示所有学员信息")
	fmt.Println("4. 删除学员")
	fmt.Println("5. 退出系统")

}

// 获取用户输入的函数
func getInput() *student {
	var (
		id    int
		name  string
		class string
	)
	fmt.Println("请按照要求输入学员信息: ")
	fmt.Print("请输入学号:")
	_, _ = fmt.Scanf("%d\n", &id)
	fmt.Print("请输入姓名:")
	_, _ = fmt.Scanf("%s\n", &name)
	fmt.Print("请输入班级:")
	_, _ = fmt.Scanf("%s\n", &class)
	// 输入完以后~调用student的构造函数
	stu := newStudent(id, name, class)
	fmt.Println(stu)
	return stu
}

func main() {
	// 实例化一个 studentMgr
	m1 := newStudentMgr()
	for {
		var input int
		// 1. 展示菜单
		showMenu()
		// 2. 获取用户输入:
		fmt.Print("Please enter number:")
		// 利用 fmt.Scan 捕获用户的输入 &input 变量需要传入指针
		_, err := fmt.Scanf("%d\n", &input)
		if err != nil {
			fmt.Println("Scan Failed err:", err)
		}
		fmt.Println("用户输入的是: ", input)
		// 3. 根据用户输入执行用户输入的操作
		switch input {
		case 1:
			stu := getInput()
			m1.addStudents(stu)
		case 2:
			stu := getInput()
			m1.editStudents(stu)
		case 3:
			m1.showStudents()
		case 4:
			stu := getInput()
			m1.deleteStudents(stu)
		case 5:
			os.Exit(0)
		default:
			fmt.Println("无效输入,请重新输入")
			continue
		}
	}
}
```


