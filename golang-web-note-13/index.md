# Gin templates


# Go Web 编程


## gin 框架 templates


* templates 的基础语法以及概念在前面的文章中有记录

* [HTML 基础概念](https://jicki.cn/golang/go/2000/01/01/golang-study-note-8 "HTML 基础")


### gin 下的例子

* go 代码

```go
package main

import (
	"fmt"
	"html/template"
	"net/http"

	"github.com/gin-gonic/gin"
)

type User struct {
	Name  string
	Age   int
	Hobby []string
}

// template 语法
func main() {
	s1 := User{
		Name: "张大仙",
		Age:  20,
		Hobby: []string{
			"王者",
			"英雄联盟",
			"说骚话",
		},
	}
	s2 := User{
		Name: "魔教教主",
		Age:  30,
		Hobby: []string{
			"英雄联盟",
			"刺激战场",
			"说骚话",
		},
	}
	r := gin.Default()

	// 导入 自定义函数到 模板中
	// 导入 自定义函数必须在 加载模板 之前,否则会报错
	r.SetFuncMap(template.FuncMap{
		"custom": func(name string) template.HTML {
			return template.HTML(fmt.Sprintf("hello %s", name))
		},
	})

	// 加载模板
	r.LoadHTMLGlob("templates/*")
	// html 基础
	r.GET("/basic", func(c *gin.Context) {
		c.HTML(http.StatusOK, "basic.tmpl", gin.H{
			"s1": s1,
			"s2": s2,
		})
	})
	// html 自定义函数
	r.GET("/custom", func(c *gin.Context) {
		c.HTML(http.StatusOK, "custom.tmpl", gin.H{
			"s1": s1,
		})
	})
	// html 嵌套
	r.GET("/nest", func(c *gin.Context) {
		c.HTML(http.StatusOK, "nest.tmpl", nil)
	})
	_ = r.Run(":8888")
}
```

{% raw %}

* html 代码


* basic.tmpl

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <!-- 标题 -->
    <title>HTML基础</title>
</head>
<body>
    <!-- .变量 的使用-->
    <p>欢迎光临 {{ .s1.Name }}</p>
    <p>姓名: {{ .s1.Name }}</p>
    <p>年龄: {{ .s1.Age }}</p>
    <!-- 变量定义 与 判断语句 -->
    {{ $age := .s1.Age}}
    {{ if lt $age 22}}
        <p> {{ .s1.Name }} 好好学习 </p>
    {{else}}
        <p> {{ .s1.Name }} 好好工作 </p>
    {{end}}
    <!-- range 循环 -->
    <p>爱好:
        <br>
        {{ range $index, $hobby := .s1.Hobby }}
        {{ $index }} - {{ $hobby }} </p>
    {{end}}
    <hr>
    <!-- with 的使用 -->
    {{ with .s2 }}
    <p>欢迎光临 {{ .Name }}</p>
    <p>姓名: {{ .Name }}</p>
    <p>年龄: {{ .Age }}</p>
    <!-- 变量定义 与 判断语句 -->
    {{ $age := .Age }}
    {{ if lt $age 22 }}
        <p> {{ .Name }} 好好学习 </p>
    {{else}}
        <p> {{ .Name }} 好好工作 </p>
    {{end}}
    <!-- range 循环 -->
    <p>爱好:
        <br>
        {{ range $index, $hobby := .Hobby }}
        {{ $index }} - {{ $hobby }} </p>
    {{end}}
    {{end}}
    <hr>
    <!-- index 索引取值 -->
    <p>index 取 s2 Hobby 索引为[2]的值 = {{index .s2.Hobby 2}} </p>
</body>
</html>
```

* custom.tmpl

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <!-- 标题 -->
    <title>HTML 进阶</title>
</head>
<body>
    {{ with .s1 }}
        <!-- 引用 自定义函数 -->
    <p> {{ custom .Name }}</p>
    {{end}}
</body>
</html>
```


* nest.tmpl


```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <!-- 标题 -->
    <title>HTML 嵌套</title>
</head>
<body>
    <!-- 嵌套另一个单独的模板文件 -->
    {{template "nesting" }}
<hr>
    {{template "temp"}}
</body>
</html>
<!-- define 自定义模板 -->
{{ define "temp"}}
    <ol>
        <li>青铜</li>
        <li>白银</li>
        <li>黄金</li>
    </ol>
{{end}}

```


* nesting.tmpl


```html

{{ define "nesting" }}
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <!-- 标题 -->
    <title>HTML 嵌套的模板</title>
</head>
<body>
    <p>HTML 嵌套模板</p>
    <ul>
        <li>钻石</li>
        <li>星耀</li>
        <li>王者</li>
    </ul>
</body>
</html>
{{end}}

```

### 自定义模板渲染符

* 自定义模板渲染符可使用 `r.Delims("左边符号", "右边符号")` 来重新定义。

* go 代码

```go
package main

import (
	"fmt"
	"html/template"
	"net/http"

	"github.com/gin-gonic/gin"
)

type User struct {
	Name  string
	Age   int
	Hobby []string
}

// template 语法
func main() {
	s1 := User{
		Name: "张大仙",
		Age:  20,
		Hobby: []string{
			"王者",
			"英雄联盟",
			"说骚话",
		},
	}
	s2 := User{
		Name: "魔教教主",
		Age:  30,
		Hobby: []string{
			"英雄联盟",
			"刺激战场",
			"说骚话",
		},
	}
	r := gin.Default()

	// 自定义渲染分隔符
	// 自定义渲染分隔符必须在 导入模板之前定义
	r.Delims("{[{", "}]}")
	// 加载模板
	r.LoadHTMLGlob("templates/*")
	// html 自定义渲染符号
	r.GET("/delimit", func(c *gin.Context) {
		c.HTML(http.StatusOK, "delimit.tmpl", gin.H{
			"name": s1.Name,
			"age":  s1.Age,
		})
	})
	_ = r.Run(":8888")
}

```

* html 

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <!-- 标题 -->
    <title>HTML 自定义分隔符</title>
</head>
<body>
    <!-- 自定义分隔符 -->
    <p>姓名:  {[{ .name }]}</p>
    <p>年龄:  {[{ .age }]}</p>
</body>
</html>

```


### 多模板继承

* Gin 框架下的模板都是单模板,多模板继承需要使用 第三方的包。

* `go get -u github.com/gin-contrib/multitemplate`

* go 代码

```go
package main

import (
	"fmt"
	"html/template"
	"net/http"

	"github.com/gin-contrib/multitemplate"

	"github.com/gin-gonic/gin"
)
type User struct {
	Name  string
	Age   int
	Hobby []string
}

// 创建一个方法用来处理多模板继承
func createMyRender() multitemplate.Renderer {
	r := multitemplate.NewRenderer()
        // 添加两个多模板继承, 初始模板必须写在前面。
	r.AddFromFiles("index", "templates/inherit.tmpl", "templates/i1.tmpl")
	r.AddFromFiles("home", "templates/inherit.tmpl", "templates/i2.tmpl")
	return r
}

// html 继承
func main() {
	s1 := User{
		Name: "张大仙",
		Age:  20,
		Hobby: []string{
			"王者",
			"英雄联盟",
			"说骚话",
		},
	}
	s2 := User{
		Name: "魔教教主",
		Age:  30,
		Hobby: []string{
			"英雄联盟",
			"刺激战场",
			"说骚话",
		},
	}
	r := gin.Default()
	// 加载模板
	r.LoadHTMLGlob("templates/*")
        
	// 接收多模板函数定义的返回值
	r.HTMLRender = createMyRender()
        // 定义一个组
	inGroup := r.Group("/inherit")
	{

		inGroup.GET("/index", func(c *gin.Context) {
                        // 这里 c.HTML 写入的"index" 模板为 createMyRender 函数定义的名称
			c.HTML(http.StatusOK, "index", gin.H{
				"name": s1.Name,
			})
		})

		inGroup.GET("/home", func(c *gin.Context) {
			// 这里 c.HTML 写入的"home" 模板为 createMyRender 函数定义的名称
			c.HTML(http.StatusOK, "home", gin.H{
				"name": s2.Name,
			})
		})
	}

	_ = r.Run(":8888")
}
```

* html 模板

* inherit.tmpl

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8">
    <!-- 标题 -->
    <title>HTML 继承</title>
</head>
<body>
<div id="container" style="width:100%">

    <div id="header" style="background-color:lightseagreen;">
        <h1 style="margin-bottom:0;text-align:center;">HTML 继承</h1></div>

    <div id="menu" style="background-color:lightcyan;height:200px;width:100px;float:left;">
        <b></b>
        <br>
        Golang
        <br>
        Gin
        <br>
        HTML
    </div>
    <div id="content" style="text-align:center;">
        {{ block "content" . }} {{ end }}
    </div>

    <div id="footer" style="background-color:cornflowerblue;clear:both;text-align:center;">
    联系方式: https://jicki.cn</div>
</div>
</body>
</html>

```

* i1.tmpl

```html
{{/*继承 inherit 模板*/}}
{{template "inherit"}}

{{/* 重新定义 inherit 模板中的内容 */}}
{{define "content"}}
    <h1>这是index页面</h1>
    <p>hello {{ .name }}</p>
{{end}}

```


* i2.tmpl

```html
{{/*继承 inherit 模板*/}}
{{template "inherit"}}

{{/* 重新定义 inherit 模板中的内容 */}}
{{define "content"}}
    <h1>这是home页面</h1>
    <p>hello {{ .name }}</p>
{{end}}

```

{% endraw %}

