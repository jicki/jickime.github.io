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


## 初始化 Zap Logger

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





### 自定义 New() 方法

 
* `func New(core zapcore.Core, options ...Option) *Logger `

  * `zapcore.Core` 主要的一些参数其中的三个配置项

    * `Encoder` - 编码器, 1. 更改输出日志格式.  如: `zapcore.NewJSONEncoder(zap.NewProductionEncoderConfig())` 或 `zapcore.NewConsoleEncoder(zap.NewProductionEncoderConfig())` 。 2. 更改日志时间的格式. 如: `encoderConfig := zap.NewProductionEncoderConfig() ` 、 `encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder`。

    * `WriteSyncer` - 配置日志输出方式. 如需要输出到文件需使用 `zapcore.AddSync()` 函数,并将打开的文件句柄传入函数中. `file, _ := os.OpenFile("./HttpGet.log", os.O_CREATE|os.O_APPEND|os.O_RDWR, 0644)` 

    * `LogLevel` - 日志级别. ( Debug、Info、Warn、Error、DPanic、Panic、Fatal ) 


  * `options` 配置, 可以增加一些额外的输出

    * `zap.AddCaller()` - 函数调用信息记录。日志中输出如: `"caller":"zap/main.go:47"` 此类信息。



> 自定义 New 例子

```go

// 定义一个全局的 zap.Logger 实例 Logger
var Logger *zap.Logger

// 定义一个全局的 zap.SugaredLogger 实例 SugarLogger
var SugarLogger *zap.SugaredLogger

// 初始化 zap Logger 函数
func InitLogger() {
	encoder := NewEncoder()
	writeSync := NewLogWriter()
	core := zapcore.NewCore(encoder, writeSync, zapcore.DebugLevel)
	Logger = zap.New(core, zap.AddCaller())
	SugarLogger = Logger.Sugar()
}

// InitLogger: zap.New() 函数的 code 部分 Encoder 配置项
func NewEncoder() zapcore.Encoder {
	//return zapcore.NewConsoleEncoder(zap.NewProductionEncoderConfig())
	encoderConfig := zap.NewProductionEncoderConfig()
	encoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder

	return zapcore.NewJSONEncoder(encoderConfig)
}

// InitLogger: zap.New() 函数的 code 部分 LogWriter 配置项
func NewLogWriter() zapcore.WriteSyncer {
	file, _ := os.OpenFile("./HttpGet.log", os.O_CREATE|os.O_APPEND|os.O_RDWR, 0644)
	return zapcore.AddSync(file)
}

func simpleHttpGet(url string) {
	resp, err := http.Get(url)
	if err != nil {
		SugarLogger.Errorf("Get [%s]  Error: %s \n", url, err)
	} else {
		SugarLogger.Infof("Get [%s] Success StatusCode: %d \n", url, resp.StatusCode)
		_ = resp.Body.Close()
	}
}

func main() {
	InitLogger()

	simpleHttpGet("http://www.baidu.com")
	// Sync 是将内存中的数据保存到本地磁盘
	defer SugarLogger.Sync()
}
```


```shell
# 输出文件的日志


{"level":"info","ts":"2020-07-21T14:38:01.090+0800","msg":"Get [http://www.baidu.com] Success StatusCode: 200 \n"}
{"level":"info","ts":"2020-07-21T14:48:29.474+0800","caller":"zap/main.go:47","msg":"Get [http://www.baidu.com] Success StatusCode: 200 \n"}

```



## Lumberjack 切割日志

> lumberjack is a log rolling package for Go 

* `Zap` 本身并不支持日志的切割, 需要配合第三方库实现。




### 安装


```shell
go get -u github.com/natefinch/lumberjack

```


### zap 结合lumberjack

* `zap` 使用 `lumberjack` 需要在 `zapcore.NewCore()` 函数中的 `WriteSyncer` 方法中集成。 

> NewLogWriter 函数

```go
// InitLogger: zap.New() 函数的 code 部分 LogWriter 配置项
func NewLogWriter() zapcore.WriteSyncer {
	lumberjackLogger := &lumberjack.Logger{
		// 日志文件路径
		Filename: "./HttpGet.log",
		// 最大切割文件大小 单位是 MB
		MaxSize: 500,
		// 最多备份数量
		MaxBackups: 5,
		// 最多保存天数
		MaxAge: 30,
		// 是否进行压缩
		Compress: false,
	}
	return zapcore.AddSync(lumberjackLogger)
}
```


* 切割日志的格式为: `HttpGet-2020-07-21T07-35-29.789.log` 。 





