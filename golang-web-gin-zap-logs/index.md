# Gin 集成 Zap 日志库


# Gin Zap 集成


* Gin 框架中配置使用 `Zap` 日志库。

  * Gin 框架输出的日志, 包括项目中自己输出的日志, 以及 Gin 框架本身的日志输出。

  * Gin 框架的日志 `Logger` 是在我们调用 `gin.Default()` 的时候, 会加载两个中间件 `engine.Use(Logger(), Recovery())` 

    * `Logger()` - 中间件是将 Gin 框架中日志输出到标准输出.

    * `Recovery()` - 中间件是 程序出现 `panic` 的时候尝试恢复程序,然后将错误代码写入到日志中.


**Gin 框架中要集成 Zap 日志库 就需要重新实现 Logger() 和 Recovery() 两个中间件函数**


* 如下转载 七米 老师的代码

```go
// GinLogger 接收gin框架默认的日志
func GinLogger(logger *zap.Logger) gin.HandlerFunc {
	return func(c *gin.Context) {
		start := time.Now()
		path := c.Request.URL.Path
		query := c.Request.URL.RawQuery
		c.Next()

		cost := time.Since(start)
		logger.Info(path,
			zap.Int("status", c.Writer.Status()),
			zap.String("method", c.Request.Method),
			zap.String("path", path),
			zap.String("query", query),
			zap.String("ip", c.ClientIP()),
			zap.String("user-agent", c.Request.UserAgent()),
			zap.String("errors", c.Errors.ByType(gin.ErrorTypePrivate).String()),
			zap.Duration("cost", cost),
		)
	}
}

// GinRecovery recover掉项目可能出现的panic
func GinRecovery(logger *zap.Logger, stack bool) gin.HandlerFunc {
	return func(c *gin.Context) {
		defer func() {
			if err := recover(); err != nil {
				// Check for a broken connection, as it is not really a
				// condition that warrants a panic stack trace.
				var brokenPipe bool
				if ne, ok := err.(*net.OpError); ok {
					if se, ok := ne.Err.(*os.SyscallError); ok {
						if strings.Contains(strings.ToLower(se.Error()), "broken pipe") || strings.Contains(strings.ToLower(se.Error()), "connection reset by peer") {
							brokenPipe = true
						}
					}
				}

				httpRequest, _ := httputil.DumpRequest(c.Request, false)
				if brokenPipe {
					logger.Error(c.Request.URL.Path,
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
					// If the connection is dead, we can't write a status to it.
					c.Error(err.(error)) // nolint: errcheck
					c.Abort()
					return
				}

				if stack {
					logger.Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
						zap.String("stack", string(debug.Stack())),
					)
				} else {
					logger.Error("[Recovery from panic]",
						zap.Any("error", err),
						zap.String("request", string(httpRequest)),
					)
				}
				c.AbortWithStatus(http.StatusInternalServerError)
			}
		}()
		c.Next()
	}
}
```





## Gin New

* 将两个中间件函数添加到 Gin 框架实例中


```go

func main() {
	// r := gin.Default()
	r := gin.New()
        r.Use(GinLogger(), GinRecovery())

	r.GET("/", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"Code": http.StatusOK,
			"Msg":  "OK",
		})
	})
	_ = r.Run()
}

```



 

