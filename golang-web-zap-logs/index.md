# Go Zap 日志库


# Go 项目中 使用 Zap 日志库

* `Zap` 日志库是 Uber 开源的一个 性能非常好, 结构化的日志库。


## 日志

* 日志系统需要具备的功能

  1. 能够将项目`事件`记录到文件中, 而不单单是应用程序控制台输出。

  2. 日志切割 - 能够根据文件大小、时间间隔等条件来切割日志文件。

  3. 支持不同的日志级别。如: `DEBUG`、`INFO`、`ERROR`等。

  4. 支持打印项目运行基础信息, 调用文件路径、函数名和行号, 日志时间等。






## 安装

```shell

go get -u go.uber.org/zap

```


## 配置 Zap Logger

* `Zap` 提供了两种类型的日志记录器 分别是 `Sugared Logger`  与 `Logger`。

  * `Sugared Logger` - 在不是很关键的上下文中使用. 它支持 结构化和`printf`风格的日志记录。

  * `Logger` - 在每微秒和每次内存分配都很重要的上下文中使用. 它速度比 `Sugared Logger` 速度更快, 内存分配次数更少, 但是它只支持强类型的结构化日志记录。




### Logger


* 通过调用 `zap.NewProduction()` 、 `zap.NewDevelopment()` 、 `zap.Example()` 创建一个 Logger 实例。

  * `NewProduction()` `NewDevelopment()` `Example()` 区别在于记录的信息不一样, 调用函数信息、日期、时间、日志级别等。

* 默认配置下, 以上三种都会将日志 打印到应用程序的 `Console` 界面中。 


> Logger 使用例子


```go
// 定义一个全局的 zap.Logger 实例 Logger
var Logger *zap.Logger

// 初始化 zap Logger 函数
func InitLogger() {
	Logger, _ = zap.NewProduction()
}

func simpleHttpGet(url string) {
	resp, err := http.Get(url)
	if err != nil {
		Logger.Error("Get Url Error",
			zap.String("url", url),
			zap.Error(err))
	} else {
		Logger.Info("Get Url Success",
			zap.String("Status", resp.Status),
			zap.String("url", url))
		_ = resp.Body.Close()
	}
}

func main() {
	InitLogger()

	// 模拟访问
	simpleHttpGet("http://www.google.com")
	simpleHttpGet("http://www.baidu.com")

	// Sync 是将内存中的数据保存到本地磁盘
	defer Logger.Sync()
}
```


```shell
# 输出结果

{"level":"error","ts":1595230339.586619,"caller":"zap/main.go:20","msg":"Get Url Error","url":"http://www.google.com","error":"Get \"http://www.google.com\": dial tcp 69.63.184.30:80: i/o timeout","stacktrace":"main.simpleHttpGet\n\t/Users/jicki/jicki/web/zap/main.go:20\nmain.main\n\t/Users/jicki/jicki/web/zap/main.go:35\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:203"}

{"level":"info","ts":1595230294.988758,"caller":"zap/main.go:24","msg":"Get Url Success","Status":"200 OK","url":"http://www.baidu.com"}

```


* `Logger.Info` 方法  `func (log *Logger) Info(msg string, fields ...Field)` 其中Info可以是其他的日志级别, fields 接收任意数量的`zapcore.Field` 的参数。

  * `zapcore.Field` 每一组都是 `key/value` 形式的结构化参数。
 


### Sugared Logger


> Sugared Logger 例子

```go
// 定义一个全局的 zap.Logger 实例 Logger
var Logger *zap.Logger

// 定义一个全局的 zap.SugaredLogger 实例 sugaredLogger
var sugaredLogger *zap.SugaredLogger

// 初始化 zap SugaredLogger 函数
func InitSugaredLogger() {
	// Logger 这里是赋值 不是 定义, 所以使用 = 赋值
	Logger, _ = zap.NewProduction()
	// 这里是 赋值 所以使用 =
	sugaredLogger = Logger.Sugar()
}

func sugaredHttpGet(url string) {
	// sugared Logger 支持 printf 的输出
	sugaredLogger.Debugf("Get request for %s \n", url)
	resp, err := http.Get(url)
	if err != nil {
		sugaredLogger.Errorf("Get for %s  Error %s \n", url, err)
	} else {
		sugaredLogger.Infof("Get for %s Success \n", url)
		_ = resp.Body.Close()
	}
}

func main() {
	InitSugaredLogger()

	// 调用日志
	sugaredHttpGet("http://www.google.com")
	sugaredHttpGet("http://www.baidu.com")
	// Sync 是将内存中的数据保存到本地磁盘
	defer sugaredLogger.Sync()
}

```

```shell
# 输出结果

{"level":"error","ts":1595236309.9537542,"caller":"zap/main.go:48","msg":"Get for http://www.google.com  Error Get \"http://www.google.com\": dial tcp 75.126.135.131:80: i/o timeout \n","stacktrace":"main.sugaredHttpGet\n\t/Users/jicki/jicki/web/zap/main.go:48\nmain.main\n\t/Users/jicki/jicki/web/zap/main.go:59\nruntime.main\n\t/usr/local/go/src/runtime/proc.go:203"}

{"level":"info","ts":1595236309.971819,"caller":"zap/main.go:50","msg":"Get for http://www.baidu.com Success \n"}

```

