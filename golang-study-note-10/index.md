# Golang flag 库


# Go语言基础

## Golang flag 库

* Go语言内置 - `flag`包实现了命令行参数的解析, `flag`包使得开发命令行工具更为简单。

* `flag`参数类型

  * `flag`包 - 支持的命令行参数类型有`bool`、`int`、`int64`、`uint`、`uint64`、`float`、`float64`、`string`、`duration`。


|flag参数|有效值|
|-|-|
|字符串flag|合法字符串|
|整数flag|1234、0664、0x1234等类型, 也可以是负数。|
|浮点数flag|合法浮点数|
|bool类型flag|1, 0, t, f, T, F, true, false, TRUE, FALSE, True, False|
|时间段flag|任何合法的时间段字符串。如:"300ms"、"-1.5h"、"2h45m"。合法的单位。如: "s"、"ms"、"h"。|

### 定义命令行flag参数

* `flag.Type()`

  * `flag.Type(flag名, 默认值, 帮助信息)*Type` 

  * 注意: 如下: name、age、married、delay 均为对应类型的指针。

```go

package main

import (
	"flag"
	"fmt"
)

func main() {
	// 分别是 名称, 具体内容, 帮助信息
	name := flag.String("name", "张大仙", "姓名")
	age := flag.Int("age", 20, "年龄")
	married := flag.Bool("married", false, "婚否")
	delay := flag.Duration("delay", 0, "时间间隔")
	// 解析命令行参数
	flag.Parse()
	// 输出
	fmt.Println(*name, *age, *married, *delay)
}

```

```shell
# go run main.go -name "张小仙" -age 22 -married true 

张小仙 22 true 0s

```

* `flag.TypeVar()`

  * `flag.TypeVar(Type指针, flag名, 默认值, 帮助信息)` 

```go

package main

import (
	"flag"
	"fmt"
	"time"
)

func cmdFlag() {
	var (
		name    string
		age     int
		married bool
		delay   time.Duration
	)
	flag.StringVar(&name, "name", "张大仙", "姓名")
	flag.IntVar(&age, "age", 20, "年龄")
	flag.BoolVar(&married, "married", false, "婚否")
	flag.DurationVar(&delay, "delay", 0, "时间间隔")
	flag.Parse()
	fmt.Println(name, age, married, delay)
}

func main() {
	cmdFlag()
}


```

```shell
# go run main.go -age=50 -married=true -delay=20s

张大仙 50 true 20s

```



* `flag.Parse()`

  * `flag.Parse()` 是用来对定义好的 `flag` 命令参数进行解析。

  * 支持的命令行参数格式有以下几种:

    * `-flag xxx` - (使用空格, 一个-符号)
 
    * `--flag xxx` - (使用空格, 两个-符号)

    * `-flag=xxx` - (使用等号, 一个-符号) 

    * `--flag=xxx` - (使用等号，两个-符号)

  * 其中, 布尔类型的参数必须使用等号的方式指定。

  * `Flag`解析在第一个非`flag`参数（单个"-"不是flag参数）之前停止, 或者在终止符"–"之后停止。


### 其他 flag 函数


* `flag.Args()` - 返回命令行参数后的其他参数, 以`[]string`类型。

* `flag.NArg()` - 返回命令行参数后的其他参数个数。

* `flag.NFlag()` - 返回使用的命令行参数个数。



```go
package main

import (
	"flag"
	"fmt"
	"time"
)

func cmdFlag() {
	var (
		name    string
		age     int
		married bool
		delay   time.Duration
	)
	flag.StringVar(&name, "name", "张大仙", "姓名")
	flag.IntVar(&age, "age", 20, "年龄")
	flag.BoolVar(&married, "married", false, "婚否")
	flag.DurationVar(&delay, "delay", 0, "时间间隔")
	flag.Parse()

	fmt.Printf("额外使用了%d个命令行外参数\n", flag.NArg())
	fmt.Printf("使用了%d个命令行参数\n", flag.NFlag())
	fmt.Println(name, age, married, delay, flag.Args())

}

func main() {
	cmdFlag()
}
```

```shell
# go run main.go -age 20 "我来了" "我走了" 

额外使用了2个命令行外参数
使用了1个命令行参数
张大仙 20 false 0s [我来了 我走了]

```


### flag 应用


```go
package main

import (
	"flag"
	"fmt"
	"time"
)

func cmdFlag() {
	var (
		password string
	)
	flag.StringVar(&password, "pw", "123456", "密码")
	flag.Parse()

	//判断输入的值
	if password == "654321" {
		fmt.Println("密码正确")
		fmt.Printf("当前时间为 %v \n", time.Now().Format("2006-01-02 15:04:05"))
	} else {
		fmt.Println("密码不正确,请确认后再试。")
	}

}

func main() {
	cmdFlag()
}
```


