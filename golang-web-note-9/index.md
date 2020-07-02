# Cookie 与 Session


# Go Web 编程

* HTTP 协议是无状态的，对于一个浏览器发出的多次请求，WEB 服务器无法区分, 是不是来源于同一个浏览器, 所以诞生了Cookie 与 Session  使某个域名下的所有网页能够共享某些数据.

## Cookie

* `Cookie` 实际上是一小段的文本信息`（key-value)` 形式, `Cookie` 是纯文本格式，不包含任何可执行的代码.

* 客户端向服务器发起请求，如果服务器需要记录该用户状态，就使用`Response`向客户端浏览器颁发一个`Cookie`。客户端浏览器会把`Cookie`保存起来。当浏览器再请求该网站时，浏览器把请求的网址连同该`Cookie`一同提交给服务器。服务器检查该`Cookie`，以此来辨认用户状态。


### Cookie 机制

1. 客户端`User-Agent`发送一个请求到服务器。

2. 服务器发送一个`HttpResponse`响应到客户端，其中包含`Set-Cookie`的头部。

3. 客户端保存`Cookie`, 之后向服务器发送请求时, `HttpRequest`请求中会包含一个`Cookie`的头部。

4. 服务器返回响应数据。 


### Cookie 特点

1. 客户端发送请求的时候, 会携带服务端`HttpResponse` 之前`Set-Cookie`的`Cookie`信息。

2. 服务端可以设置`Cookie`数据`key/value`信息。 

3. `Cookie`是针对单个域名的，不同域名之间的`Cookie`是独立的。 

4. `Cookie`数据可以配置过期时间，过期的`Cookie`数据会被系统清除。



### Golang 使用 Cookie

* Go语言中 `net/http` 标准库定义了 `Cookie` 。

```go
type Cookie struct {
    Name       string
    Value      string
    Path       string
    Domain     string
    Expires    time.Time
    RawExpires string
    // MaxAge=0 表示未设置Max-Age属性
    // MaxAge<0 表示立刻删除该Cookie，等价于"Max-Age: 0"
    // MaxAge>0 表示存在Max-Age属性，单位是秒
    MaxAge   int
    Secure   bool
    HttpOnly bool
    Raw      string
    Unparsed []string // 未解析的"key/value"对的原始文本
}
```



* 使用 `net/http` 标准库中的 `SetCookie` 函数, 设置 `Cookie`. 

```go
func SetCookie(w ResponseWiter, cookie *Cookie)
```


* 获取 `Cookie`, `Request` 对象拥有两个获取`Cookie`的方法和一个添加`Cookie`的方法

  * 获取 `Cookie` 方法

```go
// 1. 解析并返回该请求的Cookie头设置的所有Cookie
func (r *Request) Cookies() []*Cookie

// 2. 返回请求中名为name的Cookie，如果未找到该Cookie会返回nil, ErrNoCookie。
func (r *Request) Cookie(name string) (*Cookie, error)
```

  * 添加Cookie的方法

```go
// AddCookie向请求中添加一个Cookie。
func (r *Request) AddCookie(c *Cookie)
```



### Gin 框架 Cookie

* 利用 `c *gin.Context`  `c.SetCookie` 设置`Cookie`  `c.Cookie` 获取 `Cookie`

* 例子:

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

// 创建一个 User 的结构体
type UserInfo struct {
	UserName string `form:"username"`
	Password string `form:"password"`
}

func autoCookie(c *gin.Context) gin.HandlerFunc {

}

func loginHandler(c *gin.Context) {
	if c.Request.Method == "POST" {
		var u UserInfo
		if err := c.ShouldBind(&u); err != nil {
			c.HTML(http.StatusOK, "login.html", gin.H{
				"ShouldBindErr": "用户名密码禁止为空",
			})
			return
		}
		if u.UserName == "jicki" && u.Password == "123456" {
			// 设置 Session
			session := sessions.Default(c)
			// 清除旧的
			session.Clear()
			session.Set("username", u.UserName)
			_ = session.Save()
			// 登录成功跳转到 home 页
			c.Redirect(http.StatusFound, "/index")
		} else {
			c.HTML(http.StatusOK, "login.html", gin.H{
				"ShouldBindErr": "用户名密码错误",
			})
			return
		}
	} else {
		c.HTML(http.StatusOK, "login.html", nil)
	}

}

func indexHandler(c *gin.Context) {
	c.HTML(http.StatusOK, "index.html", nil)
}

func homeHandler(c *gin.Context) {
	// 判断是否有 Cookie
	cookie, err := c.Cookie("username")
	if err != nil {
		c.Redirect(http.StatusFound, "/login")
		return
	}
	c.HTML(http.StatusOK, "home.html", gin.H{
		"username": cookie,
	})
}

func main() {
	r := gin.Default()
	r.LoadHTMLGlob("templates/*")

	r.GET("/index", indexHandler)
	r.GET("/login", loginHandler)
	r.POST("/login", loginHandler)
	r.GET("/home", homeHandler)

	_ = r.Run(":8888")
}
```

* Html 文件

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>首页</title>
</head>
<body>
    <h1>首页</h1>
</body>
</html>
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录页面</title>
</head>
<body>
    <form action="" method="POST" enctype="application/x-www-form-urlencoded" >
        <div>
            <label> 用户名:
                <input type="text" name="username">
            </label>
        </div>
        <div>
            <label> 密码:
                <input type="password" name="password">
            </label>
        </div>
        <div>
            <input type="submit">
        </div>
        <p style="color:red">{{ .ShouldBindErr }}</p>
    </form>
</body>
</html>
```


```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Home</title>
</head>
<body>
    <h1>{{ .username }}家目录</h1>
</body>
</html>

```

## Session 

* `Session` 与 `Cookie` 都是会话保持的机制, `Session` 是记录客户状态的机制, 不同的是`Cookie` 保存在客户端浏览器中，而 `Session` 保存在服务器上。

### Session 原理

1. 浏览器向服务器发送登录请求(post), 携带账号和密码。

2. 登录成功, 服务器记录登录的状态, `req.session.user = user`; 服务器记录这些信息。

3. 服务器返回的响应头中携带服务器生成的 `Session ID` 并将 `Session ID` 记录到`Cookie`中，作为身份标识。

4. 浏览器再次访问服务器的时候会通过`Cookie`携带`Session ID`。

5. 服务器获取浏览器发送的`Session ID`后, 在服务器查找`Session ID`, 如果找不到, 返回未登录状态。

6. 如果找到 `Session ID` , 根据 `Session ID` 查找对应的对象, 返回登录成功。


### Gin 框架 Session

* Gin middleware for Session management `https://github.com/gin-contrib/sessions`

* Gin 框架可以使用基于 Gin 中间件的 第三方模块 处理 `Session`。

* `gin-sessions` 支持多种后端存储 `Session`

  1. cookie-based

  2. Redis

  3. Memcached

  4. MongoDB

  5. Memstore


* Download and install:

```shell

go get -u github.com/gin-contrib/sessions

```

* import:

```shell
import "github.com/gin-contrib/sessions"

```


*  例子:

```go
package main

import (
	"fmt"
	"net/http"

	"github.com/gin-contrib/sessions"
	"github.com/gin-contrib/sessions/cookie"
	"github.com/gin-gonic/gin"
)

// 创建一个 User 的结构体
type UserInfo struct {
	UserName string `form:"username"`
	Password string `form:"password"`
}

func homeHandler(c *gin.Context) {
	session := sessions.Default(c)
	username := session.Get("username")

	c.HTML(http.StatusOK, "home.html", gin.H{
		"username": username,
	})
}

func loginHandler(c *gin.Context) {
	if c.Request.Method == "POST" {
		var u UserInfo
		if err := c.ShouldBind(&u); err != nil {
			c.HTML(http.StatusOK, "login.html", gin.H{
				"ShouldBindErr": "用户名密码禁止为空",
			})
			return
		}
		if u.UserName == "jicki" && u.Password == "123456" {
			// 登录成功设置一个 Session
			SetSession(c, u)
			// 登录成功跳转到 home 页
			c.Redirect(http.StatusFound, "/index")
		} else {
			c.HTML(http.StatusOK, "login.html", gin.H{
				"ShouldBindErr": "用户名密码错误",
			})
			return
		}
	} else {
		c.HTML(http.StatusOK, "login.html", nil)
	}

}

func indexHandler(c *gin.Context) {
	c.HTML(http.StatusOK, "index.html", nil)
}

// 包装一个 Session 中间件, 并初始化session
func Session(secret string) gin.HandlerFunc {
	// 1. 创建一个 Cookie 实例 用于存储 Session
	store := cookie.NewStore([]byte(secret))
	// 2 创建一个 Redis 实例 用于存储 Session
	//store, _ := redis.NewStore(10, "tcp", "localhost:6379", "", []byte("secret"))
	// 3 创建一个 MemCached 实例 用于存储 Session
	//store := memcached.NewStore(memcache.New("localhost:11211"), "", []byte("secret"))
	// 4 创建一个 MongoDB  实例 用于存储 Session
	//session, err := mgo.Dial("localhost:27017/test")
	//if err != nil {
	//	// handle err
	//}
	//c := session.DB("").C("sessions")
	//store := mongo.NewStore(c, 3600, true, []byte("secret"))
	// 5 创建一个 memstore 实例 用于存储 Session
	//store := memstore.NewStore([]byte("secret"))
	//Also set Secure: true if using SSL, you should though
	store.Options(sessions.Options{HttpOnly: true, MaxAge: 7 * 86400, Path: "/"})
	return sessions.Sessions("gin-session", store)
}

// 包装 一个 设置 Session 的函数
func SetSession(c *gin.Context, user UserInfo) {
	session := sessions.Default(c)
	session.Clear()
	session.Set("username", user.UserName)
	err := session.Save()
	if err != nil {
		fmt.Printf("Save Session Failed: %s\n", err)
		return
	}
}

// 判断是否登录中间件
func AuthSessionMiddle() gin.HandlerFunc {
	return func(c *gin.Context) {
		session := sessions.Default(c)
		username := session.Get("username")
		if username == nil {
			c.Redirect(http.StatusFound, "/login")
			return
		}
		// 设置一个键值对
		c.Set("username", username)
		// 执行下一个 程序
		c.Next()
		return
	}
}

func main() {
	r := gin.Default()
	r.LoadHTMLGlob("templates/*")
	// 定义一个Session 加密串
	secret := "SetSessionPassword"
	r.GET("/index", indexHandler)
	// 开始使用 gin中间件 Sessions
	r.Use(Session(secret))

	r.GET("/login", loginHandler)
	r.POST("/login", loginHandler)
	r.GET("/home", AuthSessionMiddle(), homeHandler)

	_ = r.Run(":8888")
}

```


* html 文件

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>首页</title>
</head>
<body>
    <h1>首页</h1>
</body>
</html>

```


```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Home</title>
</head>
<body>
    <h1>{{ .username }}家目录</h1>
</body>
</html>

```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>登录页面</title>
</head>
<body>
    <form action="" method="POST" enctype="application/x-www-form-urlencoded" >
        <div>
            <label> 用户名:
                <input type="text" name="username">
            </label>
        </div>
        <div>
            <label> 密码:
                <input type="password" name="password">
            </label>
        </div>
        <div>
            <input type="submit">
        </div>
        <p style="color:red">{{ .ShouldBindErr }}</p>
    </form>
</body>
</html>

```


### gob 序列化

* 标准库`gob`是golang提供的 "私有" 的编解码方式, 它的效率会比json，xml等更高, 特别适合在Go语言程序间传递数据。


* 例子:

```go
package main

import (
	"bytes"
	"encoding/gob"
	"fmt"
)

type s struct {
	data map[string]interface{}
}

func gobDemo() {
	var s1 = s{
		data: make(map[string]interface{}, 8),
	}
	s1.data["count"] = 1
	// encode 编码
	// 创建一个 指针空间
	buf := new(bytes.Buffer)
	// 创建一个 编码器对象
	enc := gob.NewEncoder(buf)
	// 对 s1.data 进行编码
	err := enc.Encode(s1.data)
	if err != nil {
		fmt.Println("gob encode failed, err:", err)
		return
	}
	// 获取 编码后的 字节(Bytes)数据
	b := buf.Bytes()
	fmt.Println(b)
	var s2 = s{
		data: make(map[string]interface{}, 8),
	}
	// decode 解码
	// 创建一个 解码器对象
	dec := gob.NewDecoder(bytes.NewBuffer(b))
	// 对 s2.data 指针 进行解码
	err = dec.Decode(&s2.data)
	if err != nil {
		fmt.Println("gob decode failed, err", err)
		return
	}
	fmt.Println(s2.data)
	for _, v := range s2.data {
		fmt.Printf("value:%v, type:%T\n", v, v)
	}
}

func main() {
	gobDemo()
}

```

* 输出:

```shell
[14 255 129 4 1 2 255 130 0 1 12 1 16 0 0 18 255 130 0 1 5 99 111 117 110 116 3 105 110 116 4 2 0 2]
map[count:1]
value:1, type:int

```


### gin-session 的 gob 问题

* 使用`gin-session`的时候报错: 

  * securecookie: error - caused by: securecookie: error - caused by: gob: type not registered for interface: `自定义类型或高级对象`

* 需要解决以上错误,需要对`gob.Register(自定义类型或高级对象)` 类型进行注册。


* 错误例子:

```go
package main

import (
	"bytes"
	"encoding/gob"
	"encoding/json"
	"fmt"
	"log"
)

func CloneObject(a, b interface{}) []byte {
	// 创建一个 指针空间
	buff := new(bytes.Buffer)
	// 创建一个 编码器对象
	enc := gob.NewEncoder(buff)
	// 创建一个 解码器对象
	dec := gob.NewDecoder(buff)
	// 对 a 进行编码
	err := enc.Encode(a)
	if err != nil {
		log.Panic("e1: ", err)
	}
	// 获取 a 编码后的 字节(Bytes)数据
	b1 := buff.Bytes()
	// 对 b 进行解码
	err = dec.Decode(b)
	if err != nil {
		log.Panic("e2: ", err)
	}
	// 返回编码后的的 bytes 数据 b1
	return b1
}

func main() {
	// 定义 a 为空结构体类型
	var a interface{}
	// 初始化并赋值 a
	a = map[string]interface{}{"X": 1}
	// json 序列化 &a
	b2, err := json.Marshal(&a)
	// 打印序列化后的数据 b2
	fmt.Println(string(b2), err)

	// 定义 b 为空结构体类型
	var b interface{}
	// 使用 gob 对 &a &b 进行序列化与反序列化
	b1 := CloneObject(&a, &b)
	fmt.Println(string(b1))
}
```

* 输出:

```shell
{"X":1} <nil>
2019/12/23 15:13:49 e1: gob: type not registered for interface: map[string]interface {}
```


* 修改后的例子:


```go
package main

import (
	"bytes"
	"encoding/gob"
	"encoding/json"
	"fmt"
	"log"
)

func CloneObject(a, b interface{}) []byte {
	// 创建一个 指针空间
	buff := new(bytes.Buffer)
	// 创建一个 编码器对象
	enc := gob.NewEncoder(buff)
	// 创建一个 解码器对象
	dec := gob.NewDecoder(buff)
	// 对 a 进行编码
	err := enc.Encode(a)
	if err != nil {
		log.Panic("e1: ", err)
	}
	// 获取 a 编码后的 字节(Bytes)数据
	b1 := buff.Bytes()
	// 对 b 进行解码
	err = dec.Decode(b)
	if err != nil {
		log.Panic("e2: ", err)
	}
	// 返回编码后的的 bytes 数据 b1
	return b1
}

func main() {
	// 定义 a 为空结构体类型
	var a interface{}
	// 初始化并赋值 a
	a = map[string]interface{}{"X": 1}
	// json 序列化 &a
	b2, err := json.Marshal(&a)
	// 打印序列化后的数据 b2
	fmt.Println(string(b2), err)

	// 定义 b 为空结构体类型
	var b interface{}
	// 注册一下
	gob.Register(map[string]interface{}{})
	// 使用 gob 对 &a &b 进行序列化与反序列化
	b1 := CloneObject(&a, &b)
	fmt.Println(string(b1))
}

```


## Cookie 与 Session 优劣

1. `Cookie` 数据存放在客户端(浏览器等..), `Session` 数据放在服务器端(内存、关系型数据库、Redis、Memcache等)。

2. `Cookie` 不是很安全, 别人可以分析存放在本地的`Cookie` 并进行 `Cookie` 欺骗 考虑到安全应当使用`Session`。

3. `Session` 会在一定时间内保存在服务器上。当访问增多, 会比较占用你服务器的性能 考虑到减轻服务器性能方面, 应当使用 `Cookie` 。 

4. 单个`Cookie`保存的数据不能超过4KB, 很多浏览器都限制一个站点最多保存20个`Cookie`。

5. 将登陆信息等重要信息存放为 `Session`、其他信息如果需要保留, 可以放在`Cookie`中。

