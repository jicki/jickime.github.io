# Go学习笔记


# Go语言基础

## Go语言包(package)

* 在工程化的Go语言开发项目中, Go语言的源码复用是建立在包（package）基础之上的。

* `包（package`）是多个Go源码的集合, 是一种高级的代码复用方案, Go语言为我们提供了很多内置包,如`fmt`、`os`、`io`等。

* 我们可以根据自己的需要创建自己的包。一个包可以简单理解为一个存放.go文件的文件夹。该文件夹下面的所有go文件都要在代码的第一行添加`package 包名`, 声明该文件归属的包。
  1. 一个文件夹下面只能有一个包, 同样一个包的文件不能在多个文件夹下。
  2. 包名可以不和文件夹的名字一样, 包名不能包含`-`符号。
  3. 包名为`main`的包为应用程序的入口包，编译时不包含`main`包的源代码时不会得到可执行文件。

### 包(package)的可见性

* 如果想在一个包中引用另外一个包里的标识符（如变量、常量、类型、函数等）时, 该标识符必须是对外可见的（public）。在Go语言中只需要将标识符的`首字母大写`就可以让标识符对外可见了。
* `首字母小写` 只能在当前包中调用。

### 包的导入

* 要在代码中引用其他包的内容, 需要使用`import`关键字导入使用的包。具体语法 `import "包的路径"`

* 注意事项:
  1. import导入语句通常放在文件开头包声明语句的下面。
  2. 导入的包名需要使用双引号包裹起来。
  3. 包名是从$GOPATH/src/后开始计算的，使用/进行路径分隔。
  4. Go语言中禁止循环导入包。
  5. Go语言中导入的包必须要使用,否则会报错。

### 包的别名

* 在导入包名的时候, 我们还可以为导入的包设置别名。具体语法格式 `import 别名 "包的路径"`

```go

import (
	"fmt"
	// 导入自己写的包
	js "github.com/jicki/package/calc"
)

func main() {
	ret := js.Add(10, 20)
	fmt.Println(ret)
}
```

### 匿名导入包

* 如果只希望导入包, 而不使用包内部的数据时, 可以使用匿名导入包。具体的格式 `import _ "包的路径"`

### init()函数

* 在Go语言程序执行时导入包语句会自动触发包内部`init()`函数的调用。需要注意的是: `init()` 函数没有参数也没有返回值。 `init()`函数在程序运行时自动被调用执行, 不能在代码中主动调用它。

```go
# 包初始化时执行的顺序

[ 全局声明 ]  # 变量声明,既 var a = 100 等
    ↓
[ init() ]
    ↓
[ main() ]

```

### init()函数执行顺序

* Go语言包会从`main`包开始检查其导入的所有包, 每个包中又可能导入了其他的包。Go编译器由此构建出一个树状的包引用关系, 再根据引用顺序决定编译顺序, 依次编译这些包的代码。

* `init()` 多用于执行一些初始化工作,如: 日志格式化, 加载配置文件, 打开数据库连接等。

* 多个包含 `init()` 函数的 包在运行时, 被最后导入的包会最先初始化并调用其init()函数。

```go
                # 导入包的顺序如下:
          import        import         import
[ main() ] ---> [ 函数A ] ---> [ 函数B ] ---> [ 函数C ]

                # 初始化时执行顺序如下:
[ main.init() ] <--- [ A.init() ] <--- [ B.init() ] <--- [ c.init() ]


```

* 实际运行例子:

```go
package a

import (
	"fmt"

	_ "github.com/jicki/golang基础/go包/package/b"
)

func init() {
	fmt.Println(" 执行 a 包 的 init() 函数")
}
```

```go
package b

import (
	"fmt"

	_ "github.com/jicki/golang基础/go包/package/c"
)

func init() {
	fmt.Println(" 执行 b 包 的 init() 函数")
}
```

```go
package c

import "fmt"

func init() {
	fmt.Println(" 执行 c 包 的 init() 函数")
}
```

```go
package main

import (
	"fmt"
	// 导入自己写的包
	_ "github.com/jicki/golang基础/go包/package/a"
)

func init() {
	fmt.Println("执行 main 包 的 init() 函数")
}

func main() {
	fmt.Println("执行 main 包 的 main() 函数")
}
```

* 输出

```go
 执行 c 包 的 init() 函数
 执行 b 包 的 init() 函数
 执行 a 包 的 init() 函数
 执行 main 包 的 init() 函数
 执行 main 包 的 main() 函数

```

### 练习题

* 编写一个clac包实现加减乘除四个功能函数, 在snow这个包中导入并使用加减乘除四个函数实现数学运算。

```go
# clac 包
func Add(x, y int) int {
	return x + y
}

func Sub(x, y int) int {
	return x - y
}

func Multi(x, y int) int {
	return x * y
}

func Divide(x, y int) int {
	return x / y
}
```

```go
# snow 包

package snow

import (
	"fmt"

	"github.com/jicki/package/package_work/clac"
)

func Sum(char string, x, y int) int {
	sum := 0
	switch char {
	case "+":
		sum = clac.Add(x, y)
	case "-":
		sum = clac.Sub(x, y)
	case "*":
		sum = clac.Multi(x, y)
	case "/":
		sum = clac.Divide(x, y)
	default:
		fmt.Println("char 输入有误!")
	}
	return sum
}
```

```go
# main.go

import (
	"fmt"

	"github.com/jicki/package/package_work/snow"
)

func main() {
	ret := snow.Sum("+", 100, 200)
	fmt.Println(ret)
}

```


