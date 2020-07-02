# Gin 框架


# Go Web 编程

## Gin 框架

>官方 中文文档 https://gin-gonic.com/zh-cn/docs/


## Gin 简介

* Gin 是一个用 Go (Golang) 编写的 HTTP web 框架。 使用 `httprouter`, 因此是一个拥有很好性能的API框架。


## Gin 特性

* 快速
  * 基于 Radix 树的路由，内存占用小。没有使用反射。API 性能可以直观的测试出来。

* 支持中间件
  * 传入的 HTTP 请求可以由一系列中间件和最终操作来处理。 例如：Logger，Authorization，GZIP，最终操作 DB。

* Crash 处理
  * Gin 可以 捕获 发生在 HTTP 请求中的 panic 并 recover 它。这样，你的服务器将始终可用。例如，你可以向 Sentry 报告这个 panic .

* JSON 验证
  * Gin 可以解析并验证请求的 JSON，例如检查所需值的存在。

* 路由组
  * 更好地组织路由。是否需要授权，不同的 API 版本,  此外，这些组可以无限制地嵌套而不会降低性能。

* 错误管理
  * Gin 提供了一种方便的方法来收集 HTTP 请求期间发生的所有错误。使用中间件可以将错误写入日志文件，数据库里。

* 内置渲染
  * Gin 为 JSON，XML 和 HTML 渲染提供了易于使用的 API。

* 可扩展性
  * 可以很简单创建中间件。

## 安装使用

* 通过 go get 并使用 import 导入包, 既可。

```shell
go get -u github.com/gin-gonic/gin
```

```go

import "github.com/gin-gonic/gin"

```

## JSON 格式

* c.JSON(状态码, JSON序列化的数据gin.H{"":"",})

```go
package main

import (
	"github.com/gin-gonic/gin"
	"log"
)

func main() {
	// 创建一个 gin实例,返回一个 *engine 路由引擎
	r := gin.Default()
	// 创建一个GET 方法 的 /hello 的路由
	// func 使用 1. 匿名函数方式,拼接json
	r.GET("/hello",func(c *gin.Context){
		// 使用 JSON格式,方式, 状态码为 200
		// gin.H 是返回一个map
		c.JSON(200,gin.H{
			"message":"hello world",
		})
	})
        // 2. 结构体的形式返回
	r.GET("/json", func(c *gin.Context) {
                // 定义一个结构体
		stu := struct {
			// 首字母必须大写,否则序列化不出值
			Name string
			Age  int
		}{Name: "jicki", Age: 20}
		// 传入状态码, 结构体
		c.JSON(http.StatusOK, stu)
	})
	// 启动 gin 服务
	if err := r.Run(":8888");err!=nil{
		log.Fatal(err.Error())
	}
}
```

```shell
# 访问 http://127.0.0.1:8888/hello

# 显示一个 json 格式的数据

{"message":"hello world"}


# 访问 http://127.0.0.1:8888/json

{"Name":"jicki","Age":20}
```

## XML 格式输出

* c.XML(状态码, 具体的结构体类型)

```go

package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()
	r.GET("/xml", func(c *gin.Context) {
		// 定义一个结构体
		type Student struct {
			// 首字母必须大写,否则序列化不出值
			Name string
			Age  int
		}
		stu1 := Student{
			Name: "学生",
			Age:  20,
		}
		// XML 需要传入具体的结构体类型
		c.XML(http.StatusOK, stu1)
	})

	// 启动
	if err := r.Run(":8888"); err != nil {
		log.Fatalf("Server Run Failed err:%v\n", err)
		return
	}
}


```



* 输出

```shell
<Student>
  <Name>学生</Name>
  <Age>20</Age>
</Student>
```


## YAML 格式

```go
package main

import (
        "log"
        "net/http"

        "github.com/gin-gonic/gin"
)

func main() {
        r := gin.Default()
        r.GET("/yaml", func(c *gin.Context) {
                // 定义一个结构体
                stu := struct {
                        // 首字母必须大写,否则序列化不出值
                        Name string
                        Age  int
                }{Name: "jicki", Age: 20}
                // 传入 结构体
                c.YAML(http.StatusOK, stu)
        })
        // 启动
        if err := r.Run(":8888"); err != nil {
                log.Fatalf("Server Run Failed err:%v\n", err)
                return
        }
}
```


## RESTful API

* 基于Gin 的 RESTful API 写法 (GET,POST,PUT,DELETE)

```go
func main() {
	r := gin.Default()
	// HTTP 四个请求方式

	// 基于 GET 请求
	r.GET("/hello",func(c *gin.Context){
		// gin.H 是 map[string]interface{} 的一种快捷方式
		c.JSON(http.StatusOK, gin.H{
			"Message":"GET",
		})
	})
	// 基于 POST 请求
	r.POST("/hello",func(c *gin.Context){
		c.JSON(http.StatusOK,gin.H{
			"Message":"POST",
		})
	})
	// 基于 PUT 请求
	r.PUT("/hello",func(c *gin.Context){
		c.JSON(http.StatusOK,gin.H{
			"Message":"PUT",
		})
	})
	// 基于 DELETE 请求
	r.DELETE("/hello",func(c *gin.Context){
		c.JSON(http.StatusOK,gin.H{
			"Message":"DELETE",
		})
	})
	// 启动服务
	if err := r.Run(":8888");err!=nil{
		fmt.Printf("Server Run Failed err: %v\n",err)
		return
	}
}
```

### Any 请求方式

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// Any 请求,判断请求方式

func indexHandler(c *gin.Context) {
	UserName := c.PostForm("username")
	PassWord := c.PostForm("password")
	// 利用 Request.Method 判断 请求方式
	if c.Request.Method == "POST" {
		c.JSON(http.StatusOK, gin.H{
			"Code":     http.StatusOK,
			"username": UserName,
			"password": PassWord,
		})
		// 其他请求方式直接返回到 login.html
	} else {
		c.HTML(http.StatusOK, "login.html", nil)
	}
}

func main() {
	r := gin.Default()
	r.LoadHTMLGlob("templates/*")
	r.Any("/index", indexHandler)
	_ = r.Run(":8888")
}

```

* HTML页面

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录页面</title>
</head>
<body>
<form action="/index" method="post">
    用户名: <input type="text" name="username" >
    密码: <input type="password" name="password">
    <input type="submit" value="Submit">
</form>
</body>
</html>

```






## Gin 框架的渲染

### HTML渲染

* c.HTML(状态码, 模板文件, gin.H{模板内容}) 

* Gin 可以渲染 单个 或者 多个 html 文件, 也可以渲染 整个 html 目录. `LoadHTMLFiles` 和 `LoadHTMLGlob`。

* r.LoadHTMLFiles("/模板文件路径/1.html", "/模板文件路径/2.html")

* r.LoadHTMLGlob("/模板文件路径/*")  

* r.LoadHTMLGlob("/模板文件路径/**/*"), `/**/*` 需要在 html 文件中引入 {{define 路径/x.html}} 声明.

```go
func main() {
	r := gin.Default()
	// 使用 LoadHTMLGlob() 或者 LoadHTMLFiles()
	// LoadHTMLFiles 指定单个或多个 文件
	// r.LoadHTMLFiles("./templates/index.html", "./templates/login.html")
	// LoadHTMLGlob 指定目录 ,使用通配符
	r.LoadHTMLGlob("./templates/*")
	r.GET("/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", gin.H{
			// 将内容映射到 index.html 对应的标签中.
			"title":  "Gin WebSite",
			"status": http.StatusOK,
		})
	})
	// 启动
	if err := r.Run(":8888"); err != nil {
		log.Fatalf("Server Run Failed err:%v\n", err)
		return
	}
}
```

### Static 静态文件

* 当我们渲染的HTML文件中引用了静态文件时，我们只需要按照以下方式在渲染页面前调用`gin.Static`方法加载。

* 静态资源文件包含 (css, js, image 等)

* r.Static("/代码中配置的路径","实际文件的路径")

* HTML 文件index.html


```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ .title }}</title>
</head>
<body>
<h1>{{ .title }}</h1>
<p>Status: {{ .status }}</p>
<img src={{ .img }}  width="100" height="150" alt="golang"/>
</body>
</html>

```

* main

```go
func main() {
	r := gin.Default()
	r.LoadHTMLGlob("./templates/*")
	//Static 相当于挂载目录, images 是挂载后的路径(代码), static 是实际的路径
	r.Static("/images", "./static")
	r.GET("/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "index.html", gin.H{
			"title":  "Gin WebSite",
			"status": http.StatusOK,
			"img":    "/images/111.png",
		})
	})
	// 启动
	if err := r.Run(":8888"); err != nil {
		log.Fatalf("Server Run Failed err:%v\n", err)
		return
	}
}

```

## 参数解析三种方法

### QueryString 获取参数

* QueryString 查询字符串, 对 http 请求所带的数据进行解析

* http://localhost/search?name=xx&city=xx (`?`号后面为QueryString  多组 key-value `&`号分割)

```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// query_string
func queryString(c *gin.Context) {
	// DefaultQuery 当 name 不存在值时,会返回设置的值
	nameStr := c.DefaultQuery("name", "小炒肉")
	// Query  当 city 不存在时, 会返回 空字符串
	cityStr := c.Query("city")
	// 配置JSON格式化
	c.JSON(http.StatusOK, gin.H{
		"name": nameStr,
		"city": cityStr,
	})
}

func main() {
	r := gin.Default()

	//query string - 查询: http://127.0.0.1/search?name=小炒肉&city=深圳
	// ? 号后面为 query 值 key=value & 分割 key=value
	r.GET("/search", queryString)

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v", err)
	}
}
```

* 输出

```shell

{
  "city": "深圳",
  "name": "小炒肉"
}

```


### form 表单

* 请求数据通过表单进行 提交 `c.DefaultPostForm("key", "默认值")`

```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)
// form 表单
func formHandler(c *gin.Context) {
	// 当 name 不存在值时,会返回设置的值
	nameStr := c.DefaultPostForm("name", "小炒肉")
	// 当 city 不存在时, 会返回 空字符串
	cityStr := c.PostForm("city")
	// 配置JSON格式化输出
	c.JSON(http.StatusOK, gin.H{
		"name": nameStr,
		"city": cityStr,
	})
}

func main() {
	r := gin.Default()

	// form表单:  POST: http://127.0.0.1:8888/form (form-data, name=小炒肉 city=深圳)
	// POST 请求 form 表单获取数据.
	r.POST("/form", formHandler)

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v", err)
	}
}
```


### path 提交数据

*  `c.Param("key")`

* 通过路径参数 提交数据 http://127.0.0.1:8888/path/add/1 , http://127.0.0.1:8888/path/delete/2 


```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// path 路径
func pathHandler(c *gin.Context) {
	actionStr := c.Param("action")
	idStr := c.Param("id")
	c.JSON(http.StatusOK, gin.H{
		"action": actionStr,
		"id":     idStr,
	})
}

func main() {
	r := gin.Default()
	// path 参数: http://127.0.0.1:8888/path/add/1
	// 通过路径参数 提交数据
	r.GET("/path/:action/:id", pathHandler)

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v", err)
	}
}
```

* 输出:

```shell
# http://127.0.0.1:8888/path/add/1

{
  "action": "add",
  "id": "1"
}

# http://127.0.0.1:8888/path/delete/2

{
  "action": "delete",
  "id": "2"
}

```


## 参数绑定(ShouldBind)

* 为了能够更方便的获取参数, 基于请求的`Content-Type`识别请求数据类型并利用反射机制自动提取请求中`QueryString`、`form`表单、`JSON`、`XML`等参数到`struct`结构体中。Gin 框架 利用 `ShouldBind()`函数 强大的功能，它能够基于请求自动提取`JSON`、`form`表单和`QueryString`类型的数据，并把值绑定到指定的`struct`结构体对象。


* ShouldBind会按照下面的顺序解析请求中的数据完成绑定: 

  1. 如果是 `GET` 请求，只使用` Form `绑定引擎`（query）`。

  2. 如果是` POST `请求，首先检查`content-type` 是否为 `JSON` 或 `XML`，然后再使用 `Form（form-data）`。


### ShouldBind JSON


```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// 定义一个 用户结构体
type UserDB struct {
        // ShouldBind 通过对 binding 这个 tag 的参数进行 限制
	User     string `form:"user" json:"user" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}

// 绑定 json 格式
func ShouldBindJson(c *gin.Context) {
	// 创建一个 UserDB结构体类型的变量
	var userdb UserDB
	// 绑定 userdb , 通过 结构体 binding 的 tag 限制
	if err := c.ShouldBind(&userdb); err == nil {
		fmt.Printf("UserDB Info: %#v\n", userdb)
		c.JSON(http.StatusOK, gin.H{
			"user":     userdb.User,
			"password": userdb.Password,
		})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
}

func main() {
	r := gin.Default()
	// shouldBind json
	r.POST("/bindJson", ShouldBindJson)

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v", err)
	}
}

```

### ShouldBind Form表单

* Form 表单与 Json 写法相同, ShouldBind 会根据 `Content-Type` 自行选择绑定器 


```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// 定义一个 用户结构体
type UserDB struct {
	User     string `form:"user" json:"user" binding:"required"`
	Password string `form:"password" json:"password" binding:"required"`
}

// 绑定 form 表单
func ShouldBindForm(c *gin.Context) {
	var userdb UserDB
	// Form 表单跟 JSON 一样写法
	// ShouldBind 会根据请求的Content-Type自行选择绑定器
	if err := c.ShouldBind(&userdb); err == nil {
		fmt.Printf("UserDB Info: %#v\n", userdb)
		c.JSON(http.StatusOK, gin.H{
			"user":     userdb.User,
			"password": userdb.Password,
		})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
}

func main() {
	r := gin.Default()

	// shouldBind form
	r.POST("bindForm", ShouldBindForm)

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v", err)
	}
}
```


### ShouldBind QueryString

```go
package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// 定义一个 用户结构体
type UserDB struct {
	User     string `form:"user" json:"user" binding:"required"`
	Password string `form:"password" json:"password" binding:"required,min=4,max=20"`
}
// 绑定 QueryString
func ShouldBindQuery(c *gin.Context) {
	var userdb UserDB
	// ShouldBind 会根据请求的Content-Type自行选择绑定器
	if err := c.ShouldBind(&userdb); err == nil {
		fmt.Printf("UserDB Info: %#v\n", userdb)
		c.JSON(http.StatusOK, gin.H{
			"user":     userdb.User,
			"password": userdb.Password,
		})
	} else {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
	}
}

func main() {
	r := gin.Default()

	// shouldBind QueryString
	// http://127.0.0.1:8888/bindQuery?user=小炒肉&password=123456
	r.GET("/bindQuery", ShouldBindQuery)

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v", err)
	}
}
```


### shouldBind 的例子

* index.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>{{ .title }}</title>
</head>
<body>
<h1>{{ .title }}</h1>
<p>Status: {{ .status }}</p>
<img src={{ .img }}  alt="golang"/>
</body>
</html>


```

* login.html

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录页面</title>
</head>
<body>
<form action="/user/login" method="post">
    用户名: <input type="text" name="username" >
    <br>
    密码: <input type="password" name="password">
    <input type="submit" value="Submit">
</form>
</body>
</html>


```

* main.go

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

type UserDB struct {
	UserName string `form:"username" binding:"required"`
	PassWord string `form:"password" binding:"required"`
}

func loginHandler(c *gin.Context) {
	// 判断请求方式为 POST
	if c.Request.Method == "POST" {
		// 定义一个 UserDB 结构体类型的变量
		var user UserDB
		// ShouldBind : Gin 会尝试根据 Content-Type 推断如何绑定。
		// 使用 ShouldBind 会自动解析 json, form, xml 等格式。
		if err := c.ShouldBind(&user); err != nil {
			c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
			return
		}
		// 模拟验证
		if user.UserName != "jicki" || user.PassWord != "123456" {
			c.JSON(http.StatusUnauthorized, gin.H{
				"Code": http.StatusUnauthorized,
				"Msg":  "unauthorized",
			})
			return
		}
		c.HTML(http.StatusOK, "index.html", gin.H{
			"title":  "首页",
			"status": "you are logged in",
			"img":    "/user/static/1.png",
		})

	} else {
		c.HTML(http.StatusOK, "login.html", nil)
	}
}

func main() {
	r := gin.Default()
	r.LoadHTMLGlob("templates/*")
	r.Static("/user/static", "./static")

	userGroup := r.Group("/user")
	{
		userGroup.GET("/login", loginHandler)
		userGroup.POST("/login", loginHandler)
	}

	_ = r.Run(":8888")
}

```


## 重定向


### HTML 重定向 

* HTTP 重定向 支持内部、外部重定向。

* `c.Redirect("状态码", "地址")`

```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// 重定向
func main() {
	r := gin.Default()

	// Redirect 301跳转

	// 外部跳转
	r.GET("/redirect1", func(c *gin.Context) {
		// StatusMovedPermanently error code 301
		c.Redirect(http.StatusMovedPermanently, "https://www.jicki.cn/")
	})

	// 内部跳转
	r.GET("/redirect2", func(c *gin.Context) {
		// StatusMovedPermanently error code 301
		c.Redirect(http.StatusMovedPermanently, "/redirect1")
	})

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v\n", err)
	}
}
```


### 路由重定向

* 路由重定向, 使用`HandleContext`


```go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// 重定向
func main() {
	r := gin.Default()
	// HandleContext
	r.GET("/redirect3", func(c *gin.Context) {
		// 指定重定向的URL
		c.Request.URL.Path = "/redirect4"
		r.HandleContext(c)
	})
	r.GET("/redirect4", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"Redirect": "redirect4",
			"code":     http.StatusOK,
		})
	})

	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Error: %v\n", err)
	}
}
```

## Gin 路由 与 路由组


### 普通的路由请求

```go
import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func sayHello(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"Code": http.StatusOK,
		"Msg":  "hello 1",
	})
}

// 路由
func main() {
	r := gin.Default()

	// 普通的路由, 指定请求方式 GET (路径, 函数)
	r.GET("/hello1", sayHello)

	// 普通的路由, 指定请求方式 POST (路径, 匿名函数)
	r.POST("/hello2", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"Code": http.StatusOK,
			"Msg":  "hello 2",
		})
	})
	// 普通的路由, 匹配所有的请求方式
	r.Any("/hello3", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"Code": http.StatusOK,
			"Msg":  "hello 3",
		})
	})
	// 普通路由, 不匹配路径的路由默认返回
	r.NoRoute(func(c *gin.Context) {
		c.JSON(404, gin.H{
			"Code": 404,
			"Msg":  "404 Not Found",
		})
	})
	_ = r.Run(":8888")
}
```


### 路由组(Group)

* 我们可以将拥有共同URL前缀的路由划分为一个路由组。习惯性一对`{}`包裹同组的路由，这只是为了看着清晰，你用不用`{}`包裹功能上没什么区别。

* 通常我们将路由分组用在划分业务逻辑或划分API版本.


```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// 路由
func main() {
	r := gin.Default()
	// 路由组, 以组的形式定义功能组
	// Url  http://127.0.0.1:888/v1/student
	v1Group := r.Group("/v1")
	{
		v1Group.POST("/student", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"Code": 200,
				"Msg":  "v1 Version Create",
			})
		})
		v1Group.DELETE("/student", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"Code": 200,
				"Msg":  "v1 Version Delete",
			})
		})
		v1Group.GET("/student", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"Code": 200,
				"Msg":  "v1 Version Search",
			})
		})
		v1Group.PUT("/student", func(c *gin.Context) {
			c.JSON(200, gin.H{
				"Code": 200,
				"Msg":  "v1 Version Update",
			})
		})
	}
	_ = r.Run(":8888")
}
```


## Gin 上传文件


### 单个文件上传


```go

package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.LoadHTMLGlob("./upload/*")
	r.GET("/upload", func(c *gin.Context) {
		c.HTML(http.StatusOK, "upload.html", nil)
	})
	// 上传文件后的操作
	r.POST("/upload1", func(c *gin.Context) {
		// 获取上传后文件的 对象.
		// c.FormFile("Form 表单中 name="filename" 的名字")
		file, err := c.FormFile("filename")
		if err != nil {
			c.JSON(500, gin.H{
				"Code":  500,
				"Error": err.Error(),
			})
		}
		// 将文件保存到指定路径
		filePath := fmt.Sprintf("./%s", file.Filename)
		err = c.SaveUploadedFile(file, filePath)
		if err == nil {
			c.JSON(http.StatusOK, gin.H{
				"Code": http.StatusOK,
				"Data": filePath,
			})
		}
	})
	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Err: %v\n", err)
	}
}
```


* HTML 文件


```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Gin 上传文件</title>
</head>
<body>
    <form action="/upload1" method="post" enctype="multipart/form-data">
        单个文件: <input type="file" name="filename">
        <input type="submit">
    </form>
</body>
</html>

```


### 多个文件上传

* `c.MultipartForm()` 方法.

```go

package main

import (
	"fmt"
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

func main() {
	r := gin.Default()

	r.LoadHTMLGlob("./upload/*")
	r.GET("/upload", func(c *gin.Context) {
		c.HTML(http.StatusOK, "upload.html", nil)
	})
	// 多个文件上传文件后的操作
	r.POST("/upload2", func(c *gin.Context) {
		// 获取多个上传后的 对象.
		// c.MultipartForm()
		data, err := c.MultipartForm()
		if err != nil {
			c.JSON(500, gin.H{
				"Code":  500,
				"Error": err.Error(),
			})
		}
		// 将文件保存到指定路径
		// data.File["Form 表单中 name="filename" 的名字"]
		files := data.File["filename"]
		// 循环遍历所有文件
		for _, file := range files {
			filesPath := fmt.Sprintf("./%s", file.Filename)
			err = c.SaveUploadedFile(file, filesPath)
			if err == nil {
				c.JSON(http.StatusOK, gin.H{
					"Code": http.StatusOK,
					"Data": filesPath,
				})
			}
		}
	})
	err := r.Run(":8888")
	if err != nil {
		log.Fatalf("Server Run Err: %v\n", err)
	}
}

```


* html 文件

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Gin 上传文件</title>
</head>
<body>
    <form action="/upload2" method="post" enctype="multipart/form-data">
        多个文件: <input type="file" name="filename" multiple="multiple" />
        <input type="submit" />
    </form>
</body>
</html>

```


## Gin 中间件

* Gin框架允许开发者在处理请求的过程中，加入用户自己的钩子（Hook）函数。这个钩子函数就叫中间件，中间件适合处理一些公共的业务逻辑，比如登录校验、日志打印、耗时统计等。

* Gin中的中间件必须是一个`gin.HandlerFunc`类型。使用 `r.Use(HandlerFunc)` 调用中间件，可在全局, 路由组, 或者 单路由的某一个函数前。


```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func ShowIndexHandler(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"Code": 200,
		"Msg":  "首页",
	})
}

func ShowPingHandler(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"Code": 200,
		"Msg":  "购物页",
	})
}

func authLogin(c *gin.Context) {
	c.JSON(http.StatusOK, gin.H{
		"Code": 200,
		"Msg":  "Login OK",
	})
}

func userInfoHandler(c *gin.Context) {
	userInfo := struct {
		Name string
		Age  int
	}{Name: "小炒肉", Age: 20}

	c.JSON(http.StatusOK, gin.H{
		"Code": 200,
		"Data": userInfo,
	})
}

// gin 中间件
func main() {
	r := gin.Default()

	// 创建一个路由组
	ShopGroup := r.Group("/show")
	{
		ShopGroup.GET("/index", ShowIndexHandler)
		ShopGroup.GET("/show", ShowPingHandler)
	}

	// 创建另一个路由组
        // 可以在`r.Group("/user",authLogin)` 也可以写到第一行 
	UserGroup := r.Group("/user", authLogin)
	{
		// 路由组内 执行中间件
		// UserGroup.Use(authLogin)
		UserGroup.GET("/info", userInfoHandler)
	}

	_ = r.Run(":8888")

}

```


```go
package main

import (
	"fmt"
	"net/http"
	"time"

	"github.com/gin-gonic/gin"
)

func indexHandle(c *gin.Context) {
	fmt.Println("start indexHandle")

	c.JSON(http.StatusOK, gin.H{
		"Code": http.StatusOK,
		"Msg":  "index OK",
	})
}

// 定义一个中间件, 中间件必须包含 gin.Context
func middle(c *gin.Context) {
	// 设置一个值,传递到其他的 context 中
	c.Set("name", "小炒肉")

	fmt.Println("Middle in")
	// 统计函数开始的时间
	start := time.Now()
	// c.Next 执行下一个函数, 既 middle 后续的一个函数
	c.Next()
	// 计算 函数消耗的时间
	cost := time.Since(start)
	fmt.Printf("Cost = %v \n", cost)
	fmt.Println("Middle out")
}

// 定义一个中间件, 中间件必须包含 gin.Context
func m2(c *gin.Context) {
	fmt.Println("M2 in")
	// 接收 Context 中 set 的值
	name, ok := c.Get("name")
	if !ok {
		// 阻止 执行下一个函数
		c.Abort()
	} else {
		// c.Next 执行下一个函数
		fmt.Println(name)
		c.Next()
	}

	fmt.Println("M2 out")
}

// 判断是否登录的中间件 (一般都使用闭包的方式来写中间件)
func authMiddleWere(auth bool) gin.HandlerFunc {
	// 操作认证查询的工作
	// 比如查询数据库等
	return func(c *gin.Context) {
		// 判断是否登录
		//1. if 是登录用户
		//2. 为真 执行 c.Next()
		//3. 为假 执行 c.Abort()
		if auth {
			c.Next()
		} else {
			c.Abort()
		}
	}
}

func main() {
	r := gin.Default()

	// 全局中使用中间件, 注意顺序
	r.Use(middle, m2, authMiddleWere(true))

	r.GET("/index", indexHandle)

	_ = r.Run(":8888")
}

```

