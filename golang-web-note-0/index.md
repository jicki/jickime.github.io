# RESTful 框架


# Go Web 编程

## RESTful 框架

* `RESTful`架构，就是目前最流行的一种互联网软件架构。它结构清晰、符合标准、易于理解、扩展方便，所以正得到越来越多网站的采用。

  * REST这个词，是`Roy Thomas Fielding`在他2000年的博士论文中提出的, Fielding 将他对互联网软件的架构原则，定名为`REST`，即`Representational State Transfer`的缩写。中文翻译为 表现层状态转化 。

  * 如果一个架构符合REST原则，就称它为RESTful架构。


* 要理解`RESTful`架构，最好的方法就是去理解`Representational State Transfer`这个词组, REST的名称"表现层状态转化"中，省略了主语。"表现层"其实指的是"资源"（Resources）的"表现层"。"资源"是一种信息实体，它可以有多种外在表现形式。我们把"资源"具体呈现出来的形式，叫做它的"表现层"（Representation）。状态转化（State Transfer）, 客户端想要操作服务器，必须通过某种手段，让服务器端发生"状态转化"（State Transfer）。而这种转化是建立在表现层之上的，所以就是"表现层状态转化"。

  1. Resources(资源): 所谓"资源"，就是网络上的一个实体，或者说是网络上的一个具体信息, 每种资源对应一个特定的URI, 当需要获取某种资源时,访问资源对应的 URL 既可。

  2. Representation(表现层):  我们把"资源"具体呈现出来的形式，叫做它的 "表现层"（Representation）。URI只代表资源的实体，不代表它的形式。它的具体表现形式, 应该是在HTTP请求的头信息中用`Accept`和`Content-Type`字段指定，这两个字段才是对"表现层"的描述。

  3. State Transfer(状态转化): 互联网通信协议HTTP协议，是一个无状态协议。这意味着，所有的状态都保存在服务器端。因此，如果客户端想要操作服务器，必须通过某种手段，让服务器端发生"状态转化"（State Transfer）。而这种转化是建立在表现层之上的，所以就是"表现层状态转化"。


* 总结: 

  1. 每一个URI代表一种资源。

  2. 客户端和服务器之间，传递这种资源的某种表现层。

  3. 客户端通过四个HTTP动词，对服务器端资源进行操作，实现"表现层状态转化"。


## HTTP 四个动词-请求方法

* `GET` 请求 用来获取资源

* `POST` 请求 用来创建新的资源

* `PUT` 请求 用来更新资源

* `DELETE` 请求 用来删除资源


>  REST风格 的系统设计:

|请求方法|URI|含义|
|-|-|-|
|GET|/book|查询书籍信息|
|POST|/book|创建书籍|
|PUT|/book|更新书籍信息|
|DELETE|/book|删除书籍|


```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	r := gin.Default()
	// HTTP 四个请求方式
	
	// 基于 GET 请求
	r.GET("/book",func(c *gin.Context){
		c.JSON(http.StatusOK, gin.H{
			"Message":"查询书籍",
		})
	})
	// 基于 POST 请求
	r.POST("/book",func(c *gin.Context){
		c.JSON(http.StatusOK,gin.H{
			"Message":"创建书籍",
		})
	})
	// 基于 PUT 请求
	r.PUT("/book",func(c *gin.Context){
		c.JSON(http.StatusOK,gin.H{
			"Message":"更新书籍",
		})
	})
	// 基于 DELETE 请求
	r.DELETE("/book",func(c *gin.Context){
		c.JSON(http.StatusOK,gin.H{
			"Message":"删除书籍",
		})
	})
	// 启动服务
	if err := r.Run(":8888");err!=nil{
		fmt.Printf("Server Run Failed err: %v\n",err)
		return
	}
}

```

