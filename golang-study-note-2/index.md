# Go学习笔记


# Go语言基础

## 运算符

* 运算符用于在程序运行时执行数学或逻辑运算。

* Go 语言内置的运算符有:
  * 算术运算符
  * 关系运算符
  * 逻辑运算符
  * 位运算符
  * 赋值运算符

### 算数运算符

* `++`, `--` 不是运算符,是一个单独的语句. `a++` a = a + 1, `a--` a = a - 1

|运算符|描述|
|-|-|
|`+`|相加|
|`-`|相减|
|`*`|相乘|
|`/`|相除|
|`%`|求余|

```go
func main() {
	// 算法运算符
	a := 10
	b := 20
	fmt.Println(a + b)
	fmt.Println(b - a)
	fmt.Println(a * b)
	fmt.Println(b / a)
	fmt.Println(a % b)
}
```

### 关系运算符

|运算符|描述|
|-|-|
|`==`|检查两个值是否相等，如果相等返回 True 否则返回 False|
|`!=`|检查两个值是否不相等，如果不相等返回 True 否则返回 False|
|`>`|检查左边值是否大于右边值，如果是返回 True 否则返回 False|
|`>=`|检查左边值是否大于等于右边值，如果是返回 True 否则返回 False|
|`<`|检查左边值是否小于右边值，如果是返回 True 否则返回 False|
|`<=`|检查左边值是否小于等于右边值，如果是返回 True 否则返回 False|

```go
func main() {
	a := 10
	b := 20
	// 关系运算符 (返回布尔值)
	fmt.Println(a > b)
	fmt.Println(a != b)
    fmt.Println(a <= b)
}
```

### 逻辑运算符

|运算符|描述|
|-|-|
|`&&`|逻辑 AND 运算符。 如果两边的操作数都是 True，则为 True，否则为 False。|
|`||`|逻辑 OR 运算符。 如果两边的操作数有一个 True，则为 True，否则为 False。|
|`!`|逻辑 NOT 运算符。 如果条件为 True，则为 False，否则为 True。|

```go
func main() {
	// 逻辑运算符 (返回布尔值)
	fmt.Println(10 < 5 && 5 > 3)
	fmt.Println(10 < 5 && 5 > 10)
	// ! 表示取反
	fmt.Println(!(20 < 10))
    fmt.Println(5 < 10 || 10 > 5)
}
```

### 位运算符

* 位运算符对整数在内存中的二进制位进行操作。

|运算符|描述|
|-|-|
|`&`|参与运算的两数各对应的二进位相与。（两位均为1才为1）|
|`|`|参与运算的两数各对应的二进位相或。（两位有一个为1就为1）|
|`^`|参与运算的两数各对应的二进位相异或，当两对应的二进位不一样时，结果为1。|
|`<<`|左移n位就是乘以2的n次方。"a<<b" 是把a的各二进位全部左移b位，高位丢弃，低位补0。|
|`>>`|右移n位就是除以2的n次方。"a>>b" 是把a的各二进位全部右移b位。|

```go
func main() {
    //位运算符
	c := 1 // 001
	d := 5 // 101
	// & 上下两个都为1才为1
	fmt.Println(c & d) // 001
	// | 上下有1 就为1
	fmt.Println(c | d) // 101
	// ^ 上下不一样 就为1
    fmt.Println(c ^ d) // 100
    // << 左移2位 1 = 001 左移2位 100
	fmt.Println(1 << 2) // 100
	// >> 右移2位 4 = 100 右移2位 001
    fmt.Println(4 >> 2) // 001
    // 1 左移10位 10000000000
	fmt.Println(1 << 10)
}
```

### 赋值运算符

|运算符|描述|
|-|-|
|`=`|简单的赋值运算符，将一个表达式的值赋给一个左值|
|`+=`|相加后再赋值|
|`-=`|相减后再赋值|
|`*=`|相乘后再赋值|
|`/=`|相除后再赋值|
|`%=`|求余后再赋值|
|`<<=`|左移后赋值|
|`>>=`|右移后赋值|
|`&=`|按位与后赋值|
|`|=`|按位或后赋值|
|`^=`|按位异或后赋值|

```go
func main() {
	// 赋值运算符
	var e int
	// 赋值
	e = 10
	// e = e + 10
	e += 10
	fmt.Println(e)
}
```

## 流程控制

* 流程控制是每种编程语言控制逻辑走向和执行次序的重要部分, 流程控制可以说是一门语言的"经脉"。

* Go语言中最常用的流程控制有`if`和`for`, 而`switch`和`goto`主要是为了简化代码、降低重复代码而生的结构,属于扩展类的流程控制。

### if && else

* `if`条件判断基本写法:

```go
if 条件表达式1 {
	执行语句1
} else if 条件表达式2 {
	执行语句2
} else {
	执行语句3
}

```

* 当条件表达式1的结果为true时, 执行语句1, 否则判断条件表达式2, 如果满足则执行语句2, 都不满足时, 则执行语句3。if判断中的else if和else都是可选的, 可以根据实际需要进行选择。

* Go语言规定与`if`匹配的左括号`{`必须与if和表达式放在同一行, `{`放在其他位置会触发编译错误. 同理与`else`匹配的`{`也必须与`else`写在同一行, `else`也必须与上一个`if`或`else` , `if`右边的大括号在同一行。

```go
// if else 判断
func main() {
	// 标准写法
	age := 20
	if age >= 30 {
		fmt.Println("年龄大于等于30岁")
	} else if age <= 19 {
		fmt.Println("年龄小于等于19")
	} else {
		fmt.Println("年龄介于20-30之间")
	}
	// if 语句块中也可以定义变量，只在if块中生效
	if score := 60; score >= 70 {
		fmt.Println("大于等于70")
	} else if score < 60 {
		fmt.Println("小于60")
	} else {
		fmt.Println("及格了")
	}
}

```

### for 循环

* Go 语言中的所有循环类型均可以使用`for`关键字来完成。

* 条件表达式返回`true`时循环体不停地进行循环, 直到条件表达式返回`false`时自动退出循环。
  
```go
for 初始语句;条件表达式;结束语句{
    循环体语句
}
```

```go
// for 循环
func main() {
	// 1. 标准的for循环语句
	for i := 0; i < 10; i++ {
		fmt.Println(i)
	}
	// 2. 循环语句块外定义变量，for语句中开头需要保留;号
	i := 1
	for ; i < 10; i++ {
		fmt.Println(i)
	}
	// 3. 循环体内只有条件表达式
	i = 10
	for i < 1 {
		fmt.Println(i)
		i--
	}
	// 4. 无限循环
	for {
		fmt.Println(i)
	}

	// 5. break 跳出循环
	for i := 0; i < 10; i++ {
		fmt.Println(i)
		if i == 5 {
			break
		}
	}
	// 6. continue 跳过本次循环,继续上一层循环
	for i := 0; i < 10; i++ {
		if i == 5 {
			continue
		}
		fmt.Println(i)
	}
}
```

### for range 循环

* Go语言中可以使用`for range`遍历数组、切片、字符串、map 及通道（channel）。 通过`for range`遍历的返回值有以下规律:
  1. 数组、切片、字符串返回索引和值。
  2. map返回键和值。
  3. 通道（channel）只返回通道内的值。

```go
// for range 循环
func main() {
	// 遍历数组, 返回 数组的 索引 与 值
	a1 := [5]int{1, 2, 3, 4, 5}
	for i, v := range a1 {
		fmt.Printf("index = %v, value = %v \n", i, v)
	}
	// 遍历map, 返回 map 的 key 与 value
	m1 := map[string]string{"第一列": "1", "第二列": "2", "第三列": "3"}
	for k, v := range m1 {
		fmt.Printf("Map key = %v, value = %v \n", k, v)
	}
}

```

### switch case 循环

* `switch`语句可方便地对大量的值进行条件判断。
* Go语言规定每个`switch`只能有一个`default`分支。
* `fallthrough`语法可以执行满足条件的case的下一个case, 是为了兼容C语言中的`case`设计的。

```go
func main() {
	score := 60
	switch {
	case score < 60:
		fmt.Println("不及格")
	case score >= 60 && score < 80:
		fmt.Println("及格")
	case score >= 80:
		fmt.Println("优秀")
	default:
		fmt.Println("无效的分数")
	}
	// 判断多个条件
	num := 5
	switch num {
	case 1, 3, 5, 7, 9:
		fmt.Println("奇数")
	case 2, 4, 6, 8, 10:
		fmt.Println("偶数")
	default:
		fmt.Println("无效的输入")
	}
	// fallthrough 满足条件以后会继续执行下一个
	num = 10
	switch {
	case num < 20:
		fmt.Println("小于20")
		fallthrough
	case num > 5:
		fmt.Println("大于5")
	case num > 10:
		fmt.Println("大于10")
	default:
		fmt.Println("其他")
	}
}

```

### goto语句

* `goto`语句通过标签进行代码间的无条件跳转。`goto`语句可以在快速跳出循环、避免重复退出上有一定的帮助。Go语言中使用`goto`语句能简化一些代码的实现过程。

* `goto` 语句必须配合 标签使用.

```go
// goto 语句
func main() {
	// for 循环多层嵌套
	for i := 0; i < 10; i++ {
		for j := 0; j < 10; j++ {
			if j == 5 {
				goto breakTag
			}
			fmt.Printf("i = %v j = %v \n", i, j)
		}
	}
	return
breakTag:
	fmt.Println("Goto跳转到此处")
}

```

### break(跳出循环)

* `break`语句可以结束`for`、`switch`和`select`的代码块。

* `break`语句还可以在语句后面添加标签，表示退出某个标签对应的代码块，标签要求必须定义在对应的`for`、`switch`和 `select`的代码块上。

```go
// break(跳出循环)
func main() {
	// for 循环
	for i := 0; i < 10; i++ {
		fmt.Println(i)
		if i == 2 {
			// 跳出for循环
			break
		}
	}
	// break 配合 标签
breakTag:
	for i := 0; i < 10; i++ {
		for j := 0; j < 10; j++ {
			if j == 2 {
				break breakTag
			}
			fmt.Printf("i = %v j = %v\n", i, j)
		}
	}
	fmt.Println("循环结束")
}

```

### continue(继续下次循环)

* `continue` 语句可以跳过前循环, 进行下一次的循环, 仅限在for循环内使用。

* 在`continue`语句后添加标签时, 表示开始标签对应的循环。

```go
// continue 继续下一个循环
func main() {
	for i := 0; i < 10; i++ {
		if i == 5 {
			continue
		}
		fmt.Println(i)
	}
}

```

### 九九乘法表

```go
func main() {
	for i := 1; i < 10; i++ {
		for j := 1; j <= i; j++ {
			fmt.Printf("%v * %v = %v| ", j, i, i*j)
		}
		fmt.Println()
	}
}
```


