# logrus 日志系统


# Go Web 编程

## logrus

* Logrus is a structured logger for Go (golang), completely API compatible with the standard library logger.

### Logrus 特性

* 结构化日志

* 完全兼容标准日志库, 多个日志级别: `Trace`, `Debug`, `Info`, `Warning`, `Error`, `Fataland`, `Panic` 。

* 支持Field，可以输出附加信息

* 兼容golang 原生 Log-logger

* 支持 `TextFormat` 和 `JsonFormat` 输出, 也支持自定义格式化日志格式。

* 支持Hook

* 线程安全


### 安装

```shell
go get -u github.com/sirupsen/logrus

```


### import

```shell
import "github.com/sirupsen/logrus"

```


### 兼容log日志库方式

```go

package main

import (
	"net/http"
	"os"

	// 模块的别名
	log "github.com/sirupsen/logrus"
)

func init() {
	// 设置日志格式为json格式, 并且配置 时间格式
	log.SetFormatter(&log.JSONFormatter{TimestampFormat: "2006-01-02 15:04:05.000"})

	// 设置将日志输出到标准输出（默认的输出为stderr,标准错误）
	// 日志消息输出可以是任意的io.writer类型
	log.SetOutput(os.Stdout)

	// 设置日志级别  (warn级别以上的日志才会输出)
	log.SetLevel(log.WarnLevel)

        // 记录函数名,以及行数(会极大的消耗性能)
        // log.SetReportCaller(true)
}

func main() {
	// Info 级别日志
	log.WithFields(log.Fields{
		"Code": http.StatusOK,
		"Msg":  "啦啦啦啦啦啦啦啦啦啦~~",
	}).Info("这是一条INFO日志")

	log.WithFields(log.Fields{
		"Code": http.StatusOK,
		"Msg":  "啦啦啦啦啦啦啦啦啦啦~~",
	}).Warn("这是一条 Warning 日志")

	log.WithFields(log.Fields{
		"Code": http.StatusOK,
		"Msg":  "啦啦啦啦啦啦啦啦啦啦~~",
	}).Fatal("这是一条 Fatal 日志")
}

```

* 输出:

```shell

{"Code":200,"Msg":"啦啦啦啦啦啦啦啦啦啦~~","level":"warning","msg":"这是一条 Warning 日志","time":"2019-12-16 14:35:18.164"}
{"Code":200,"Msg":"啦啦啦啦啦啦啦啦啦啦~~","level":"fatal","msg":"这是一条 Fatal 日志","time":"2019-12-16 14:35:18.164"}

```


### 自己创建Logger实例

* logrus 可以配置多个 Logger 实例, 应对多个地方输出, 通常我们会定义全局的 Logger 实例。

 
```go
package main

import (
	"net/http"
	"os"

	"github.com/sirupsen/logrus"
)

// logrus提供了New() 函数来创建一个logrus的实例.
// 可以创建任意数量的logrus实例.
var log = logrus.New()

func main() {
	// 设置logrus实例的输出到任意io.writer
	log.Out = os.Stdout
	// 配置输出格式以及时间格式
	log.Formatter = &logrus.JSONFormatter{TimestampFormat: "2006-01-02 15:04:05.000"}
	// 设置日志级别
	log.SetLevel(logrus.WarnLevel)

	// 固定 Fields
	entry := log.WithFields(logrus.Fields{
		"Code": http.StatusOK,
		"Msg":  "啦啦啦啦啦啦啦啦啦啦~~",
	})
	// Info 级别日志
	entry.Info("这是一条 Info 日志")
	// Warn 级别日志
	entry.Warn("这是一条 Info 日志")
	// Fatal 级别日志
	entry.Fatal("这是一条 Info 日志")
}
```

* 输出:

```shell
{"Code":200,"Msg":"啦啦啦啦啦啦啦啦啦啦~~","level":"warning","msg":"这是一条 Info 日志","time":"2019-12-16 15:08:39.459"}
{"Code":200,"Msg":"啦啦啦啦啦啦啦啦啦啦~~","level":"fatal","msg":"这是一条 Info 日志","time":"2019-12-16 15:08:39.459"}

```


### Hook 接口方式

* `logrus`最令人心动的功能就是其可扩展的 `hook` 机制了,通过在初始化时为`logrus`添加`hook`,`logrus` 可以实现各种扩展功能.

* `logrus`的`hook`接口定义如下,其原理是每次写入日志时拦截,修改`logrus.Entry`.

```go

// logrus在记录Levels()返回的日志级别的消息时会触发 hook ,
// 按照Fire方法定义的内容修改logrus.Entry.
type Hook interface {
    Levels() []Level
    Fire(*Entry) error
}

```

* `hook`的使用很简单,在初始化前调用`log.AddHook(hook)`添加相应的`hook`即可.

* `logrus`官方仅仅内置了`syslog`的`hook`. 但Github也有很多第三方的hook可供使用.



## Gin 框架 使用 logrus

* logrus 在 Gin 框架中,可使用`log`方式将Gin框架输出的日志也输出到`logrus`实例中, 也可以使用Gin 中间件的方式灵活替代日志输出.

```go
package main

import (
	"net/http"
	"os"

	"github.com/gin-gonic/gin"
	"github.com/sirupsen/logrus"
)

// 创建一个 全局 logrus 实例
var log = logrus.New()

// 初始化 log 配置
func init() {
	// 配置 日志输出格式以及时间格式
	log.Formatter = &logrus.JSONFormatter{TimestampFormat: "2006-01-02 15:04:05.000"}

	// 打开文件
	file, err := os.OpenFile("./logs/gin.log", os.O_CREATE|os.O_APPEND|os.O_WRONLY, 0644)
	if err != nil {
		log.Fatalf("Open File Failed err: %v", err)
	}
	// 将日志 输出到文件
	log.Out = file

	// 配置 gin 框架的模式为 线上模式
	gin.SetMode(gin.ReleaseMode)

	// 将 gin 框架默认输出 logrus 实例中
	gin.DefaultWriter = log.Out

	// 配置 日志输出级别
	log.SetLevel(logrus.InfoLevel)
}

func main() {
	// 创建一个 gin 实例
	r := gin.Default()
	r.GET("/index", func(c *gin.Context) {
		log.WithFields(logrus.Fields{
			"Code": http.StatusOK,
			"Path": "index",
		}).Info("这是 Index 页面..")

		// 页面 JSON 的数据
		c.JSON(200, gin.H{
			"message": "This Index",
		})
	})
	_ = r.Run(":8888")
}

```




