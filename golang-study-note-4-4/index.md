# Go学习笔记


# Go语言基础

## 接口(interface)

* 接口(interface) 定义了一个对象的行为规范, 只定义规范不实现, 由具体的对象来实现规范的细节。

### 接口类型

* 在Go语言中接口（interface）是一种类型, 一种抽象的类型。

* `interface`是一组方法`method`的集合, 是`duck-type programming`的一种体现。接口做的事情就像是定义一个协议（规则）, 只要一台机器有洗衣服和甩干的功能, 我就称它为洗衣机。不关心属性（数据）, 只关心行为（方法）。

* 请牢记接口（interface）是一种类型。

* 我们编程过程中会经常遇到:
  1. 一个网上商城可能使用支付宝、微信、银联等方式去在线支付, 我们能不能把它们当成"支付方式"来处理呢？
  2. 三角形, 四边形, 圆形都能计算周长和面积, 我们能不能把它们当成"图形"来处理呢？
* Go语言中为了解决类似上面的问题, 就设计了接口这个概念。接口区别于我们之前所有的具体类型, 接口是一种抽象的类型。当你看到一个接口类型的值时, 你不知道它是什么, 唯一知道的是通过它的方法能做什么。

### 接口的定义

* Go语言提倡面向接口编程。

* 每个接口由数个方法组成, 接口的定义格式如下:
  1. 接口名: 使用`type`将接口定义为自定义的类型名。Go语言的接口在命名时, 一般会在单词后面添加`er`, 如有写操作的接口叫`Writer`, 有字符串功能的接口叫`Stringer`等。接口名最好要能突出该接口的类型含义。
  2. 方法名: 当方法名首字母是大写且这个接口类型名首字母也是大写时, 这个方法可以被接口所在的包`（package）`之外的代码访问。
  3. 参数列表、返回值列表: 参数列表和返回值列表中的参数变量名可以省略。

```go
type 接口类型名 interface{
    方法名1( 参数列表1 ) 返回值列表1
    方法名2( 参数列表2 ) 返回值列表2
    …
}

```

### 实现接口的条件

* 一个对象只要全部实现了接口中的方法, 那么就实现了这个接口。换句话说, 接口就是一个需要实现的方法列表。

```go
// 定义两个结构体
type cat struct{}
type dog struct{}

// 定义一个 sayer 接口, 有一个say()方法
type sayer interface {
	say()
}

// 分别定义say方法,这里实现了接口sayer中的方法,那么就实现了接口
func (c cat) say() {
	fmt.Println("喵喵喵~")
}
func (d dog) say() {
	fmt.Println("汪汪汪~")
}

func main() {
	var s sayer // 定义一个接口
	c1 := cat{}
	// 可以把c1这个实例直接赋值给接口
	s = c1
	s.say()
	d1 := dog{}
	// 可以把d1这个实例直接赋值给接口
	s = d1
	s.say()
}

```

* 输出:

```go
喵喵喵~
汪汪汪~
```

### 值接收者和指针接收者实现接口的区别

* 使用值接收者实现接口和使用指针接收者实现接口有什么区别呢？
  * 使用指针接收者实现接口: 只有类型指针能够保存到接口变量中。
  * 使用值接收者实现接口: 类型的值类型和指针类型都能够保存到接口变量中。

```go
// 定义一个接口 mover, 包含一个 move()方法
type mover interface {
	move()
}

// 定义一个结构体
type dog struct {
	name string
}

// 给 dog 实现一个方法 move

// 使用值接收者实现接口: 类型的值类型和指针类型都能够保存到接口变量中。
func (d dog) move() {
	fmt.Printf("%s 在奔跑~\n", d.name)
}

// 使用指针接收者实现接口: 只有类型指针能够保存到接口变量中。
//func (d *dog) move() {
//	fmt.Printf("%s 在奔跑~", d.name)
//}

func main() {
	// 定义一个接口变量
	var m mover
	// 初始化一个d1 值类型的实例
	d1 := dog{
		name: "旺财",
	}
	// 初始化一个d2 指针类型的实例
	d2 := &dog{
		name: "旺财",
	}
	// 值类型实例d1赋值给 接口变量m
	m = d1
	// 指针类型实例d2赋值给 接口变量m
	m = d2
	m.move()
	fmt.Println(m)
}
```

* 输出:

```go
旺财 在奔跑~
&{旺财}
```

### 类型与接口的关系

* 一个类型实现多个接口
  * 一个类型可以同时实现多个接口, 而接口间彼此独立, 不知道对方的实现。 例如: 狗可以叫, 也可以动。我们就分别定义Sayer接口和Mover接口.
* 多个类型实现同一接口
  * Go语言中不同的类型还可以实现同一接口 首先我们定义一个Mover接口, 它要求必须由一个move方法。

```go
type mover interface {
	move()
}

type sayer interface {
	say()
}

// 定义一个结构体
type dog struct {
	name string
}

// 给 dog 实现一个方法 move
func (d dog) move() {
	fmt.Printf("%s 在奔跑~\n", d.name)
}

// 给 dog 实现一个方法 say
func (d dog) say() {
	fmt.Printf("%s 在叫~\n", d.name)
}

func main() {
	d1 := dog{
		name: "旺财",
	}
	var m mover // 定义一个 mover 类型的变量
	var s sayer // 定义一个 sayer 类型的变量
	m = d1      // d1 可以赋值给 mover 接口
	s = d1      // d1 可以赋值给 sayer 接口
	m.move()
	s.say()
}

```

* 输出:

```go
旺财 在奔跑~
旺财 在叫~
```

### 接口嵌套

* 接口与接口间可以通过嵌套创造出新的接口。

```go
// 接口的嵌套: 相当于实现了mover 与 sayer的接口方法
type animal interface {
	mover
	sayer
}

type mover interface {
	move()
}

type sayer interface {
	say()
}

// 定义一个结构体
type dog struct {
	name string
}

// 给 dog 实现一个方法 move
func (d dog) move() {
	fmt.Printf("%s 在奔跑~\n", d.name)
}

// 给 dog 实现一个方法 say
func (d dog) say() {
	fmt.Printf("%s 在叫~\n", d.name)
}

func main() {
	d1 := dog{
		name: "旺财",
	}
	var a animal // 定义一个 animal 类型的变量
	a = d1       // d1 可以赋值给 animal 接口
	a.say()
	a.move()
}

```

* 输出:

```go
旺财 在叫~
旺财 在奔跑~
```

### 空接口

* 空接口的定义
  * 空接口是指没有定义任何方法的接口。因此任何类型都实现了空接口。
  * 空接口类型的变量可以存储任意类型的变量。

```go
func main() {
	// 定义空接口
	var x interface{}
	// 空接口 可以接受任意类型
	x = "string"
	fmt.Printf("%#v\n", x)
	x = 100
	fmt.Printf("%#v\n", x)
	x = false
	fmt.Printf("%#v\n", x)
	x = []string{}
	fmt.Printf("%#v\n", x)
	x = struct{}{}
	fmt.Printf("%#v\n", x)
}
```

* 输出:

```go
"string"
100
false
[]string{}
struct {}{}
```

### 空接口的应用

* 1.空接口作为函数的参数
  * 使用空接口实现可以接收任意类型的函数参数。

```go
// 空接口作为函数参数
func show(a interface{}) {
	fmt.Printf("type:%T value:%v\n", a, a)
}
```

* 2.空接口作为map的值
  * 使用空接口实现可以保存任意值的字典。

```go
func main(){
	// 定义一个 map value 的类型为 interface
	studentInfo := make(map[string]interface{})
	// 这里可以赋值为 任何类型
	studentInfo["name"] = "张大仙"
	studentInfo["age"] = 18
	studentInfo["hobby"] = []string{"王者荣耀", "讲骚话"}
    fmt.Printf("%#v\n", studentInfo)
    }
```

* 输出:

```go
map[string]interface {}{"age":18, "hobby":[]string{"王者荣耀", "讲骚话"}, "name":"张大仙"}

```

### 类型断言

* 空接口可以存储任意类型的值, 那我们如何获取其存储的具体数据呢?

* 接口值
  * 一个接口的值（简称接口值）是由一个具体类型和具体类型的值两部分组成的。这两部分分别称为接口的动态类型和动态值。

```go
var x interface{}
x = false

# 这里面 x 空接口包含两个部分
# 1. 动态类型 = bool 类型
# 2. 动态值 = false
```

* 想要判断空接口中的值这个时候就可以使用类型断言。 其语法格式: `x.(T)` , 该语法返回两个参数，第一个参数是x转化为T类型后的变量, 第二个值是一个布尔值, 若为`true`则表示断言成功, 为`false`则表示断言失败。
  * `x`: 表示类型为`interface{}`的变量
  * `T`: 表示断言x可能是的类型。

```go
// 类型断言
func main() {
	var x interface{}
	x = false
	// 1. 使用 ok 来做类型断言
	// 如果 !ok 的情况下,v的值是类型的零值
	v, ok := x.(string)
	if !ok {
		fmt.Printf("不是String类型 v: %v\n", v)
	} else {
		fmt.Printf("是String类型 v: %v\n", v)
	}

	// 2. 使用switch case 来做类型断言
	switch v := x.(type) {
	case string:
		fmt.Printf("String类型 v: %v\n", v)
	case int:
		fmt.Printf("Int类型 v: %v\n", v)
	case bool:
		fmt.Printf("布尔类型 v: %v\n", v)
	default:
		fmt.Printf("其他类型 v: %v\n", v)
	}

}
```

* 输出:

```go
不是String类型 v:
布尔类型 v: false

```

### 注意事项

* 关于接口需要注意的是, 只有当有两个或两个以上的具体类型必须以相同的方式进行处理时才需要定义接口。不要为了接口而写接口, 那样只会增加不必要的抽象, 导致不必要的运行时损耗。

### 接口练习题

* 使用接口的方式实现一个既可以往终端写日志也可以往文件写日志的简易日志库。

* mylogger.go

```go
package mylogger

import "strings"

// 自定义一个 日志级别 的类型
type Level uint16

// 定义 日志级别 的常量
const (
	DebugLevel Level = iota
	InfoLevel
	WarningLevel
	ErrorLevel
	FatalLevel
)

// 定义一个 日志的 接口
type Logger interface {
	Debug(format string, args ...interface{})
	Info(format string, args ...interface{})
	Warning(format string, args ...interface{})
	Error(format string, args ...interface{})
	Fatal(format string, args ...interface{})
	Close()
}

// 获取对应 日志级别的 字符串
func getLevelStr(level Level) string {
	switch level {
	case DebugLevel:
		return "DEBUG"
	case InfoLevel:
		return "INFO"
	case WarningLevel:
		return "WARNING"
	case ErrorLevel:
		return "ERROR"
	case FatalLevel:
		return "FATAL"
	default:
		return "DEBUG"
	}
}

// 获取用户传入的 字符串,解析日志级别
func parseLogLevel(levelStr string) Level {
	// 1. 将字符串转换成全小写
	levelStr = strings.ToLower(levelStr)
	switch levelStr {
	case "debug":
		return DebugLevel
	case "info":
		return InfoLevel
	case "warning":
		return WarningLevel
	case "error":
		return ErrorLevel
	case "fatal":
		return FatalLevel
	default:
		return DebugLevel
	}
}
```

* file.go

```go
package mylogger

import (
	"fmt"
	"os"
	"path"
	"time"
)

// 往文件里写日志

// 定义 文件日志 的结构体
type FileLogger struct {
	// 定义日志记录的级别
	level Level
	// 日志文件名称
	filename string
	// 日志文件路径
	filepath string
	// 文件句柄
	file    *os.File
	errFile *os.File
	// 日志大小
	maxSize int64
}

// 文件日志的 构造函数 并初始化打开日志文件以及句柄
func NewFileLogger(levelStr, filename, filepath string, logSize int64) *FileLogger {
	logLevel := parseLogLevel(levelStr)
	fl := &FileLogger{
		level:    logLevel,
		filename: filename,
		filepath: filepath,
		maxSize:  logSize,
	}
	// 初始化 传入的文件名字以及路径打开文件
	fl.initFile()
	// 返回
	return fl
}

// 初始化文件的句柄的方法
func (f *FileLogger) initFile() {
	// 1. 拼接 日志文件 的路径
	logName := path.Join(f.filepath, f.filename)
	// 2. 打开文件,使用 os.OpenFile
	fileObj, err := os.OpenFile(logName, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		panic(fmt.Errorf("打开文件 %s 失败, %v", logName, err))
	}
	// 3. 将文件句柄赋值给 FileLogger 结构体中的 f.file
	f.file = fileObj
	// 4. 错误日志
	errLogName := fmt.Sprintf("%s.err", logName)
	// 5. 打开错误日志文件
	errFileObj, err := os.OpenFile(errLogName, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		panic(fmt.Errorf("打开文件 %s 失败, %v", errLogName, err))
	}
	// 6. 将文件句柄赋值给 FileLogger 结构体中的 f.errFile
	f.errFile = errFileObj
}

func (f *FileLogger) checkSplit(file *os.File) bool {
	// 获取当前日志文件的大小
	fileInfo, _ := file.Stat()
	fileSize := fileInfo.Size()
	// 判断传入的文件大小
	return fileSize >= f.maxSize
}

// 日志切割函数
func (f *FileLogger) splitLogFile(file *os.File) *os.File {
	// 获取 文件的完整路劲
	fileName := file.Name()
	backupName := fmt.Sprintf("%s_%v.back", fileName, time.Now().Unix())
	// 关闭原来的文件句柄
	file.Close()
	// 备份原来日志的文件
	_ = os.Rename(fileName, backupName)
	// 重新打开日志文件
	fileObj, err := os.OpenFile(fileName, os.O_CREATE|os.O_WRONLY|os.O_APPEND, 0644)
	if err != nil {
		panic(fmt.Errorf("打开文件 %s 失败, %v", fileName, err))
	}
	return fileObj
}

// 日志记录功能函数
func (f *FileLogger) log(level Level, format string, args ...interface{}) {
	// 获取 日志需要记录的信息
	// 0. 判断日志的级别, 判断传入参数level的日志级别
	if f.level > level {
		return
	}
	// 1. fmt.Sprint 字符串拼接
	msg := fmt.Sprintf(format, args...)
	// 2. 定义 要写入日志的格式: ( [时间][文件:行号][函数名][日志级别] msg )
	// 2.1 获取时间
	nowStr := time.Now().Format("2006-01-02 15:04:05.000")
	// 2.2 获取 文件,行号,函数名 等信息
	fileName, link, funcName := getCallerInfo(3)
	// 2.3 获取 用户传入的日志级别 字符串
	logLevelStr := getLevelStr(level)
	// 2.4 拼接成完整的日志信息
	logMsg := fmt.Sprintf("[%s][%s:%d][%s][%s] %s",
		nowStr, fileName, link, funcName, logLevelStr, msg)
	// 检查判断是否需要切分日志
	if f.checkSplit(f.file) {
		f.file = f.splitLogFile(f.file)
	}
	// 3. fmt.Fprintf 写入 f.file 日志文件
	_, err := fmt.Fprintln(f.file, logMsg)
	if err != nil {
		panic(fmt.Errorf("写入文件 %s 失败", f.filename))
	}
	// 4. 如果 level 等级是 error 或 fatal 级别还要写入 f.errFile 中
	if level >= ErrorLevel {
		if f.checkSplit(f.errFile) {
			f.errFile = f.splitLogFile(f.errFile)
		}
		_, err := fmt.Fprintln(f.errFile, logMsg)
		if err != nil {
			panic(fmt.Errorf("写入文件 %s 失败", f.filename))
		}
	}
}

// Debug 日志级别的方法  args ...表示 一个或多个参数
func (f *FileLogger) Debug(format string, args ...interface{}) {

	f.log(DebugLevel, format, args...)
}

// Info 日志级别的方法  args ...表示 一个或多个参数
func (f *FileLogger) Info(format string, args ...interface{}) {
	f.log(InfoLevel, format, args...)
}

// Warning 日志级别的方法  args ...表示 一个或多个参数
func (f *FileLogger) Warning(format string, args ...interface{}) {
	f.log(WarningLevel, format, args...)
}

// Warning 日志级别的方法  args ...表示 一个或多个参数
func (f *FileLogger) Error(format string, args ...interface{}) {
	f.log(ErrorLevel, format, args...)
}

// Fatal 日志级别的方法  args ...表示 一个或多个参数
func (f *FileLogger) Fatal(format string, args ...interface{}) {
	f.log(FatalLevel, format, args...)
}

// 关闭日志文件句柄的方法
func (f *FileLogger) Close() {
	f.file.Close()
	f.errFile.Close()
}

```

* console.go

```go
package mylogger

import (
	"fmt"
	"os"
	"time"
)

// 终端打印日志

// 定义一个终端打印的结构体
type ConsoleLogger struct {
	// 定义日志级别
	level Level
}

// 创建一个 构造函数
func NewConsoleLogger(levelStr string) *ConsoleLogger {
	logLevel := parseLogLevel(levelStr)
	cl := &ConsoleLogger{
		level: logLevel,
	}
	// 返回
	return cl
}

// 日志记录功能函数
func (c *ConsoleLogger) log(level Level, format string, args ...interface{}) {
	// 获取 日志需要记录的信息
	// 0. 判断日志的级别, 判断传入参数level的日志级别
	if c.level > level {
		return
	}
	// 1. fmt.Sprint 字符串拼接
	msg := fmt.Sprintf(format, args...)
	// 2. 定义 要写入日志的格式: ( [时间][文件:行号][函数名][日志级别] msg )
	// 2.1 获取时间
	nowStr := time.Now().Format("2006-01-02 15:04:05.000")
	// 2.2 获取 文件,行号,函数名 等信息
	fileName, link, funcName := getCallerInfo(3)
	// 2.3 获取 用户传入的日志级别 字符串
	logLevelStr := getLevelStr(level)
	// 2.4 拼接成完整的日志信息
	logMsg := fmt.Sprintf("[%s][%s:%d][%s][%s] %s",
		nowStr, fileName, link, funcName, logLevelStr, msg)

	// 3. 日志输出到 屏幕
	_, err := fmt.Fprintln(os.Stdout, logMsg)
	if err != nil {
		panic("os.Stdout Error")
	}
}

// Debug 日志级别的方法  args ...表示 一个或多个参数
func (c *ConsoleLogger) Debug(format string, args ...interface{}) {
	c.log(DebugLevel, format, args...)
}

// Info 日志级别的方法  args ...表示 一个或多个参数
func (c *ConsoleLogger) Info(format string, args ...interface{}) {

	c.log(InfoLevel, format, args...)
}

// Warning 日志级别的方法  args ...表示 一个或多个参数
func (c *ConsoleLogger) Warning(format string, args ...interface{}) {
	c.log(WarningLevel, format, args...)
}

// Warning 日志级别的方法  args ...表示 一个或多个参数
func (c *ConsoleLogger) Error(format string, args ...interface{}) {
	c.log(ErrorLevel, format, args...)
}

// Fatal 日志级别的方法  args ...表示 一个或多个参数
func (c *ConsoleLogger) Fatal(format string, args ...interface{}) {
	c.log(FatalLevel, format, args...)
}

func (c *ConsoleLogger) Close() {
}

```

* main.go

```go
package main

import "github.com/jicki/logs_work/mylogger"

// 定义一个 日志的全局变量
var logger mylogger.Logger

func main() {
	// 实例化 文件输出
	logger = mylogger.NewFileLogger("debug", "./", "logs.log", 10*1024*1024)
	// 实例化 终端输出
	//logger = mylogger.NewConsoleLogger("debug")
	// 关闭文件
	defer logger.Close()
	// 写入日志的一些记录
	logger.Debug("Debug 日志!")
	logger.Info("Info 日志!")
	logger.Error("Error 日志!")
}

```


