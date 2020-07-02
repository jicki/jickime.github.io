# Go学习笔记


# Go语言基础

## HTML 基础

* 超文本标记语言(Hypertext Markup Language, Html) 是一种用于创建网页的标记语言
* 本质上是浏览器可识别的规则,我们按照规则写网页,浏览器根据规则渲染我们的网页.对于不同的浏览器,对同一个标签可能会有不同的解析.(既兼容性)
* 网页文件的后缀(扩展名): html 或 htm

### Web本质

#### C/S 架构

* C/S 架构 -> 软件开发

* 优势: 可定制化高,用户体验好.
* 劣势: 开发成本高,适配不同的平台,有新功能需要客户端升级.

#### B/S 架构

* B/S 架构 -> Web开发

* 优势: 开发成本低.
* 劣势: 复杂功能没办法很好的实现.

### HTML 文档结构

#### html基础结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>网页标题</title>
</head>
<body>
    <h1>用户具体看到的内容</h1>
    </body>
</html>
```

* `<!DOCTYPE html>` 声明为 html5 文档,必须在HTML 文档的第一行,位于`<html>`标签之前.
* `<html></html>` 是文档的开始标记和结束标记,是HTML页面的根元素, 在它们之间是文档的 头部`<head>` 和 主体`<body>` .
* `<head></head>` 定义了HTML文档的开头部分. 它们之间的内容不会在浏览器窗口中显示.包含了文档的元素`meta` 数据.
* `<title></title>` 定义了网页标题,在浏览器的标题栏中显示.
* `<body></body>` 之间的文本是具体的WEB内容,在网页的主体中显示.
* `<meta charset="UTF-8">` 定义了网页的编码,在中文网页中不定义有可能会出现乱码.

#### HTML标签格式

* HTML标签是由 尖括号<> 包裹的关键字,如 `<html>`, `<div>` 等.
* HTML标签通常都是 成对的出现, 比如 `<div> </div>` 第一个标签开始,第二个为标签结束,结束的标签语会带 斜杠`/` .
* 有一部分标签是单独的, 比如 `<br/>` , `<hr/>` , `<img src="1.jpg" />` 等.
* 标签里可以附带 若干个属性参数, 也可以不带属性.

#### HTML标签语法

* 成对标签:  <标签名 A属性="A属性值1;A属性值2" B属性="B属性值1;B属性值2;B属性值3"...>标签内容</标签名>
* 单标签: <标签名 属性1="属性值1" 属性2="属性值2".../>

#### HTML重要属性

* `id`: 定义标签的唯一ID, HTML标签文档数中唯一
* `class`: HTML元素定义一个或多个类名 (classname) CSS样式类名
* `style`: 规定元素的行内样式(CSS样式)

#### HTML常用标签

|标签|意义|
|-|-|
|`<title> </title>`|定义网页标题|
|`<style> </style>`|定义内部样式表|
|`<scrpit> </scrpit>`|定义JS代码或引入外部JS文件|
|`<link/>`|引入外部样式表文件|
|`<meta/>`|定义网页原信息|

#### body内常用标签

|标签|意义|
|-|-|
|`<b> </b>`| 加粗字体 |
|`<i> </i>`| 斜体 |
|`<u> </u>`| 下划线 |
|`<s> </s>`| 删除线 |
|`<p> </p>`| 段落标签 |
|`<h1> </h1>`| 标题1 |
|`<h2> </h2>`| 标题2 |
|`<h3> </h3>`| 标题3 |
|`<h4> </h4>`| 标题4 |
|`<h5> </h5>`| 标题5 |
|`<h6> </h6>`| 标题6 |
|`<br>`| 换行 |
|`<hr>`| 水平线 |
|`<!-- -->`| 注释 |

#### 特殊字符

|标签|意义|
|-|-|
|`&nbsp;`| 空格 |
|`&gt;`| > |
|`&lt;`| < |
|`&amp;`| & |
|`&yen;`| ¥ |
|`&copy;`| 版权 |
|`&reg;`| 注册 |

#### div标签 与 span标签

* `<div>` 标签用来定义一个`块级元素`,并无实际的意义.主要通过CSS样式为其赋予不同的表现.
* `<span>` 标签用来定义`内联元素`,并无实际意义.主要通过CSS样式为其赋予不同的表现.

* 块级元素 与 内联元素 的区别:
* 所谓`块级元素`, 是以另起一行开始渲染的元素, `内联元素`则不需要另起一行.如果单独在网页中插入这两个元素,不会对页面产生任何的影响.
* 在标签嵌套中,通常`块元素`可以包含`内联元素`或某些`块级元素` , 但是`内联元素`不能包含`块级元素`,它只能包含其它`内联元素`.
* `<p>`标签 不能包含 `块级元素`, `<p>` 标签内 也不能包含 `<p>` 标签.

#### img标签

* `<img src="图片路径" alt="图片未加载成功时的提示" title="鼠标悬浮时提示信息" width="宽" height="高(宽高属性只设置一个会等比例缩放)">`

#### a标签

* 超链接标签
* 所谓的`超链接`是指从一个网页指向一个目标的连接关系, 这个目标可以是另一个网页, 也可以是相同网页上的不同位置, 还可以是一个图片, 一个电子邮件地址, 一个文件, 甚至是一个应用程序.

* `<a href="https://baidu.com" target="_blank">显示的内容</a>`
* `href` 的几种属性:
  * 绝对URL - 指向另一个站点(href=<http://baidu.com>)
  * 相对URL - 指向当前站点内的路径 (href="index.html")
  * 锚URL - 指向页面中的锚(href="#top")

* target 属性:
  * `_blank` 表示在浏览器打开新的标签显示网页.
  * `_self` 表示在浏览器当前的标签中显示网页.

#### 列表

```html
<!--
无序列表: 
  type属性:
      disc   - 实心原点(默认)
      circle - 空心圆圈
      square - 实心方块
      none   - 无样式
-->

<ul type="square">
    <li>无序列表1</li>
    <li>无序列表2</li>
    <li>无序列表3</li>
    <li>无序列表4</li>
    <li>无序列表5</li>
</ul>

```

```html
<!--
有序列表:
  type属性: 
    1 数字列表(默认)
    A 大写字母
    a 小写字母
    I 大写罗马数字
    i 小写罗马数字
  start属性:
    从哪一个为开始,如 start="2"
-->
<ol type="1" start="2">
    <li>有序列表1</li>
    <li>有序列表2</li>
    <li>有序列表3</li>
    <li>有序列表4</li>
    <li>有序列表5</li>
</ol>

```

```html
<!--
标题列表:
  

-->

<dl>
    <dt>标题列表1</dt>
        <dd>标题列表1内容</dd>
    <dt>标题列表2</dt>
        <dd>标题列表2内容1</dd>
        <dd>标题列表2内容2</dd>
        <dd>标题列表2内容3</dd>
</dl>

```

#### 表格

* 表格是一个二维数据空间, 一个表格由若干行组成,一个行又有若干单元格组成,单元格里可以包含文件、列表、团案、表单、数字符号、预置文本和其它的表格等内容.
* 表格最重要的目的是显示表格类数据.表格类数据是指最适合组织为表格格式(既按行和列组织)的数据.

```html
<!--表格的基本结构-->

<table border="2">
    <thead>
    <tr>
        <th>属性</th>
        <th>意义</th>
    </tr>
    </thead>
    <tbody>
    <tr>
        <td>border</td>
        <td>表格边框</td>
    </tr>
    <tr>
        <td>cellpadding</td>
        <td>内边框</td>
    </tr>
    <tr>
        <td>cellspacing</td>
        <td>外边框</td>
    </tr>
    <tr>
        <td>width</td>
        <td>像素百分比</td>
    </tr>
    <tr>
        <td>rowspan</td>
        <td>单元格横跨多少行</td>
    </tr>
    <tr>
        <td>colspan</td>
        <td>单元格横跨多少列(合并单元格)</td>
    </tr>  
    </tbody>
</table>

```

|属性|意义|
|-|-|
|`border`|表格边框|
|`cellpadding`|内边框|
|`cellspacing`|外边框|
|`width`|像素百分比|
|`rowspan`|单元格横跨多少行|
|`colspan`|单元格横跨多少列(合并单元格)|

#### form标签

* 表单 用于向服务器传输数据,从而实现用户与WEB服务器的交互.
* 表单 能够包含input系列标签, 比如 文本字段、复选框、提交按钮 等.
* 表单 可以包含 textarea、select、fieldset 和 label 标签.
  
* 想要在HTML里面通过点击 `form` 表单的 `sumbit` 按钮提交数据:
  * 所有获取用户输入的标签必须放在 `form` 标签内.
  * 所有获取用户输入的标签必须要有 `name` 属性.
  * 必须要有 `sumbit` 按钮并且 `form` 表单要有 `action` 属性.

* 表单属性:

|属性|描述|
|-|-|
|`accept-charset`|规定在被提交表单中使用的字符集(默认: 页面字符集)|
|`action`|规定向何处提交表单的地址(URL)提交页面.|
|`autocomplete`|规定浏览器应该自动完成表单(默认开启)|
|`enctype`|规定被提交数据的编码(默认: url-encoded)|
|`method`|规定在提交表单时所用的HTTP方法(默认: GET)|
|`name`|规定识别表单的方法(对于 DOM 使用: document.forms.name)|
|`novalidate`|规定浏览器不验证表单|
|`target`|规定 action 属性中地址的目标(默认: self)|

* 表单元素
* 基本概念:
  * HTML表单是HTML元素中较为复杂的部分,表单往往和脚本、动态页面、数据处理等功能相结合,因此它是制作动态网站很重要的内容.
  * 表单一般用来收集用户的输入信息
* 表单工作原理:
  * 访问者在浏览有表单的网页时,可填写必须的信息,然后按某个按钮提交.这些信息通过 Internet 传送到服务器上.
  * 服务器上专门的程序对这些数据进行处理,如果有错误会返回错误信息,并要求纠正错误.当数据完整无误后,服务器反馈一个输入完成的信息.

* method: GET与POST方法的场景
  * GET:  获取页面 , 搜索引擎检索.
  * POST: 提交`form`表单时, 有敏感数据时.

#### input标签

* `<input>` 元素会根据不同的 type 属性, 变化多种形态.

|type属性|表现形式|对应代码|
|-|-|-|
|`text`|单行输入文本|`<input type="text" />`|
|`password`|密码输入框|`<input type="password" />`|
|`date`|日志输入框|`<input type="date" />`|
|`checkbox`|复选框|`<input type="checkbox" checked="checked" />`|
|`radio`|单选框|`<input type="radio" />`|
|`submit`|提交按钮|`<input type="submit" value="提交" />`|
|`reset`|重置按钮|`<input type="reset" value="重置" />`|
|`button`|普通按钮|`<input type="button" value="普通按钮" />`|
|`hidden`|隐藏输入框|`<input type="hiddent" />`|
|`file`|文本选择框|`<input type="file" />`|

* 属性说明:
  * `name`: 表单提交时的 `键` ,注意跟`id`区分.
  * `value`: 表单提交时对应的值(type="button" "reset" "submit" 时为按钮上显示的文本内容)
  * `checked`: `radio` 和 `checkbox` 默认被选中的项
  * `readonly`: `text` 和 `password` 设置只读
  * `disabled`: 所有 `input` 均适用.

#### select标签

* 属性说明: `<select type="属性">`
  * `multiple`: 布尔值属性, 设置后为多选, 否则默认单选.
  * `disabled`: 禁用
  * `selected`: 默认选中该选项
  * `value`: 定义提交时的选项值

```html
            <select name="addr" id="s1">
                <option value="bj">北京</option>
                <option value="sh">上海</option>
                <option value="sz">深圳</option>
                <option value="gz">广州</option>
            </select>
```

#### label标签

* 定义 `<label>` 标签为 `input` 元素定义标注(标记)
* 说明: `label` 元素不会向用户呈现任何特殊效果, `<label>` 标签的 `for` 属性值应当与相关元素的 `id`属性值相同.

```html
    <label for="i1">用户名: </label>
    <input id="i1" name="username" type="text" placeholder="用户名">
```

#### go获取form表单数据

```go
func index(w http.ResponseWriter, r *http.Request) {
    data, err := ioutil.ReadFile("./form.html")
    if err != nil {
        fmt.Println("open file err: ", err)
        return
    }
    w.Write(data)
}

func reg(w http.ResponseWriter, r *http.Request) {
    // 获取html表单提交的信息
    // r: 代表请求的相关内容
    // 获取请求的方法
    fmt.Println("请求方法: ", r.Method)
    // 解析表单数据
    r.ParseForm()
    // 获取表单数据
    fmt.Printf("表单数据 %#v \n", r.Form)
    // 获取表单数据中指定数据,通过html中<input> name 字段
    fmt.Printf("用户名:%#v  密码: %#v \n ", r.Form["username"], r.Form["password"])
    // 返回注册信息到页面上
    fmt.Fprintf(w, "头像: %v 用户名: %v 密码: %v 性别: %v 生日: %v  爱好: %v  居住地址: %v 个人介绍: %v",
        r.Form["avatar"], r.Form["username"], r.Form["password"], r.Form["gender"], r.Form["birthday"],
        r.Form["like"], r.Form["addr"], r.Form["info"])
    // w.Write([]byte("注册用户"))
}

func main() {
    http.HandleFunc("/web", index)
    http.HandleFunc("/reg", reg)
    http.ListenAndServe("127.0.0.1:8888", nil)
}
```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>注册用户</title>
</head>
<body>
    <h1>注册用户</h1>
    <form action="http://127.0.0.1:8888/reg" method="POST">
        <div>
            <label for="img">头像: </label>
            <input type="file" name="avatar" accept="image/*">
        </div>
        <div>
            <label for="i1">用户名: </label>
            <input id="i1" name="username" type="text" placeholder="用户名">
        </div>
        <div>
            <label for="pw">密&nbsp;&nbsp;&nbsp;码:</label>
            <input id="pwd" name="password" type="password">
        </div>
        <div>
            <label>性别: </label>
            <input name="gender" value="male" type="radio">男
            <input name="gender" value="female" type="radio">女
        </div>
        <div>
            <label>生日: </label>
            <input name="birthday" value="" type="date">
        </div>
        <div>
            <label>爱好: </label>
            <input name="like" type="checkbox" value="basketball">篮球
            <input name="like" type="checkbox" value="football">足球
        </div>
        <div>
            <label for="s1">居住地址: </label>
            <select name="addr" id="s1">
                <option value="bj">北京</option>
                <option value="sh">上海</option>
                <option value="sz">深圳</option>
                <option value="gz">广州</option>
            </select>
        </div>
        <div>
            <label>个人介绍: </label>
            <textarea name="info" cols="40" rows="5"></textarea>
        </div>
        <div>
            <input type="submit" value="提交">
            <input type="reset"  value="重置">
        </div>
    </form>
</body>
</html>
```

### template 语法

* 模板渲染 本质就是一种字符串替换,一种高级的字符串替换.

#### 模拟模板

```go
func index(w http.ResponseWriter, r *http.Request) {
    w.Write([]byte("欢迎光临index"))
}

func info(w http.ResponseWriter, r *http.Request) {
    data, err := ioutil.ReadFile("./info.html")
    if err != nil {
        fmt.Println("open file err: ", err)
        return
        }
    // 设置一个随机数,模拟不同访问s
    num := rand.Intn(10)
    // 先转换成字符串
    datastr := string(data)
    if num > 5 {
        // 字符的替换
        //{info} 是html中需要替换的变量
        //<li></li> 是替换后写入 html 中的标签数据
        //1 表示 替换次数
        datastr = strings.Replace(datastr, "{info}",
        "<li>《Golang》</li> \n <li>《Linux》</li> ", 1)
    } else {
        datastr = strings.Replace(datastr, "{info}",
        "<li>《三体》</li> \n <li>《大灰狼》</li> ", 1)
    }
    w.Write([]byte(datastr))
}

func main() {
    http.HandleFunc("/index", index)
    http.HandleFunc("/info", info)
    http.ListenAndServe("127.0.0.1:8888", nil)
}

```

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>个人中心</title>
</head>
<body>
    <div>
        <ul>
            {info}
        </ul>
    </div>
</body>
</html>
```

#### http/template 库

{% raw %}

* 模板语法:
  * 所有模板语法都必须包含在 \{\{ \}\} 中间.
  * \{\{\.\}\} 中的 `.` 表示当前对象.
  * 当我们传入一个结构体对象的时候,可以根据`.`来访问结构体对应的字段,如 \{\{\.Name\}\}
  * \{\{/* Go模板中的注释 */\}\} 执行时会忽略.可以多行,但是不能嵌套,需要紧贴分界符始止.

* pipeline:
  * `pipeline` 是指产生数据的操作. 比如\{\{.\}\}、\{\{\.Name\}\}等.
  * Go的模板语法中支持使用管道符 `|` 链接多个命令,用法和Linux下的管道类似,将`|`前面命令运行的结果(返回值)传递给后面的命令.
  * 注意: Go的模板语法中, `pipeline` 概念是传递数据,只要产生数据的都称为 `pipeline`.

* 变量:
  * `Action` 里可以初始化一个变量来捕获管道的执行结果.
  * 初始化语法: `$variable := pipeline` 其中 $variable 是变量名称. 声明变量的action不会产生任何输出.

```html
    <div>
        {{/* 这里是注释 */}}
        {{ $age := .Age }}
        <h1>{{ $age }}</h1>

        {{ $id := . }}
        <h1>{{ $id.ID }}</h1>
    </div>
```

#### 条件判断

* 条件判断
  * 条件判断 必须要以 \{\{end\}\} 来结束.
  * \{\{if 条件判断 arg1 arg2\}\}  输出  \{\{end\}\}
  * \{\{if 条件判断 arg1 arg2\}\} 输出 \{\{else\}\} 输出 \{\{end\}\}
  * \{\{if 条件判断 arg1 arg2\}\} 输出 \{\{else if 条件判断 arg3 arg4\}\} 输出 \{\{end\}\}

#### 比较函数

* 比较函数公式
  * 布尔函数会将任何类型的 零值 视为 假, 其余视为 真
  * `eq`: 如果 arg1 == arg2 返回 真
  * `ne`: 如果 arg1 != arg2 返回 真
  * `lt`: 如果 arg1 <  arg2 返回 真
  * `le`: 如果 arg1 <= arg2 返回 真
  * `gt`: 如果 arg1 >  arg2 返回 真
  * `ge`: 如果 arg1 >= arg2 返回 真
  * 为了简化 多参数 相等检测, `eq` 可接受2个或更多个参数, 会将第一个参数 分别于其他参数进行比较
  * `{{eq arg1 arg2 arg3}}` 这里既 arg1 分别于 arg2 arg3 分别比较.
  * 注: 比较函数只适用于 基础类型 (或重定义的基本类型, 如: type Celsius float32).

**例子:**

```html
    <div>
        {{/* 条件判断 */}}
        {{if gt .Age 20}}
    </div>
    <div>
        <h1>大于20岁</h1>
    </div>
    {{else}}
    <div>
        <h1>小于20岁</h1>
    </div>
    {{end}}
```

#### range循环

* Go的模板语法中使用 `range` 关键字进行循环遍历,其中 `pipeline` 的值必须是数组、切片、字典或者通道.
  * `{{range $key, $value := .}}` 取值可以直接取,也可以加 `.` 里面的值

**例子:**

```html

    {{/* range 循环遍历 以下遍历Map */}}
    {{/*  map[int]struct */}}
    <hr>
    <table border="2">
        <thead>
            <tr>
                <th>序号</th>
                <th>ID</th>
                <th>姓名</th>
                <th>年龄</th>
            </tr>
        </thead>
        <tbody>
            {{range $index, $user := .}}
            <tr>
                <td>{{$index}}</td>
                <td>{{$user.ID}}</td>
                <td>{{$user.UserName}}</td>
                <td>{{$user.Age}}</td>
            </tr>
            {{end}}
        </tbody>
    </table>
```

#### with(局部变量)

* with语句: 其含义就是创建一个封闭的作用域, 在其范围内, 可以使用`.action`, 而与外面的`.`无关，只与with的参数有关;

```html
{{ with arg }}
    此时的点 . 就是arg
{{ end }}
```

#### 预定义函数

* 执行模板时, 函数从两个函数字典中查找: 首先是模板函数字典, 然后是全局函数字典. 一般不在模板内定函数,而是使用Funcs方法添加函数到模板里.
* 预定义的全局函数如下:
  * `and`: 函数返回它的第一个 empty 参数或者最后一个参数; `and x y`等价于`if x then y else x` 所有参数都会执行.
  * `or`: 函数返回它的第一个非 empty 参数或者最后一个参数; `or x y`等价于`if x then x else y` 所有参数都会执行.
  * `not`: 返回它的单个参数的布尔值是否定.
  * `len`: 返回它的参数的整数类型长度.
  * `index`: 执行结果为第一个参数以剩下的参数为索引/键指向的值.如:`index x 1 2 3` 返回 `x[1][2][3]` 的值.每个被索引的主题必须是数组、切片、字典.
  * `print`: 既 fmt.Sprint
  * `printf`: 既 fmt.Sprintf
  * `println`: 既 fmt.Sprintln
  * `html`: 返回其参数文本表示的 html 逸码等价表示.
  * `urlquery`: 返回其参数文本表示可嵌入URL查询的逸码等价表示.
  * `js`: 返回其参数文本表示的 JavaScrpit 逸码等价表示.
  * `call`: 执行结果是调用第一个参数的返回值,该参数必须是函数类型,其余参数作为调用该函数的参数;如: `call .x .y 1 2` 等价于Go语言里的 dot.x.y(1,2); 其中 y 是函数类型的字段或者字典的值,或者其他类似情况; `call` 的第一个参数的执行结果必须是函数类型的值(与预定函数print明显不同); 该函数类型值必须有1到2个返回值,如果有2个 则后一个必须是error接口类型;如果有2个返回值的方法返回error非nil,模板执行会中断并返回给调用模板执行者该错误;

```go
    //构建一个 map
    userMap := map[int]user{
        1: {1, "张三", 20},
        2: {2, "李四", 10},
        3: {3, "王五", 30},
    }
    t.Execute(w, userMap)
}
```

```html
    <div>
        {{/* 传入 . = userMap */}}
        {{/* 预定函数 */}}
        <p>Map长度: {{len .}}</p>
        {{/* with 与 printf */}}

        <p>{{with index . 1}}</p>
        {{printf "ID: %d 姓名: %s  年龄: %d" .ID .UserName .Age}}
        {{end}}
    </div>
```

#### 自定义模板函数

* 为模板添加一个自定义的 函数.

例子:

```go

type user struct {
    ID       int
    UserName string
    Age      int
}

func info(w http.ResponseWriter, r *http.Request) {
    // 1. 打开模板文件
    htmlByte, err := ioutil.ReadFile("./info.html")
    if err != nil {
        fmt.Println("open html faild err", err)
        return
    }
    // 2. 自定义一个匿名函数
    helloFunc := func(arg string) (string, error) {
        return arg + "hello", nil
    }
    /* 3.1 template.New 创建一个 模板对象
       3.2 Funcs 将自定义函数添加到模板
       3.3 Parse 解析模板
    */
    t, err := template.New("info").Funcs(template.FuncMap{"hello": helloFunc}).Parse(string(htmlByte))
    if err != nil {
        fmt.Println("open html file faild! err", err)
        return
    }

//构建一个 map
    userMap := map[int]user{
        1: {1, "张三", 20},
        2: {2, "李四", 10},
        3: {3, "王五", 30},
    }
    t.Execute(w, userMap)
```

```html
    <div>
    {{/* 自定义函数 */}}
    {{with index . 1}}
        <p>{{ hello .UserName }}</p>
    {{end}}
    </div>
```

#### 嵌套template

* 在 template 中嵌套 其他的 template , 这个 template 可以是单独的文件, 也可以通过 `define` 定义的template.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="X-UA-Compatible" content="ie=edge">
    <title>嵌套 template </title>
</head>
<body>
    <h1>嵌套template语句</h1>
    <hr>
    {{/* 在template中调用 外部的html文件 */}}
    {{template "ul.html"}}
    {{/* 在template中调用 define中自定义的模板 */}}
    {{template "ol.html"}}
</body>
</html>

{{/* 在html中定义另一个html模板 */}}
{{define "ol.html"}}
<h1>定义的 ol.html</h1>
<ol>
    <li>第一</li>
    <li>第二</li>
    <li>第三</li>
</ol>
{{end}}
```

```html
{{/* ol.html */}}
<ul>
    <li>ul第一</li>
    <li>ul第二</li>
    <li>ul第三</li>
</ul>

```

```go
func index(w http.ResponseWriter, r *http.Request) {
	t, err := template.ParseFiles("./index.html", "ul.html")
	if err != nil {
		fmt.Println("open html faild err", err)
		return
	}
	t.Execute(w, t)
}

func main() {
	http.HandleFunc("/", index)
	http.ListenAndServe("127.0.0.1:8888", nil)
}
```

#### 链式操作

* 原理:
  * 每一次执行完方法以后返回操作的对象本身.

```go
// 链式操作
type student struct {
	name string
}

func (s student) study() student {
	fmt.Printf("%s 学习ing \n", s.name)
	// 执行完以后返回这个方法的本身
	return s
}

func (s student) sayhello() student {
	fmt.Printf("%s SayHello \n", s.name)
	return s
}

func main() {
	stu1 := student{"哈哈"}
	// 链式操作
	stu1.sayhello().study()
}
```

#### 模板继承

* block

  * `block` 是定义模板`{{define "name"}} T1 {{end}}`和执行`{{template "name" pipeline}}`缩写, 典型的用法是定义一组根模板, 然后通过在其中重新定义块模板进行自定义。

* `Go` 使用`template.ParseGlob`按照正则匹配规则解析模板文件, 然后通过`ExecuteTemplate`渲染指定的模板

```go
func index(w http.ResponseWriter, r *http.Request){
	tmpl, err := template.ParseGlob("templates/*.tmpl")
	if err != nil {
		fmt.Println("create template failed, err:", err)
		return
	}
	err = tmpl.ExecuteTemplate(w, "index.tmpl", nil)
	if err != nil {
		fmt.Println("render template failed, err:", err)
		return
	}
}
```

* base.tmpl

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <title>Go Templates</title>
</head>
<body>
<div class="container-fluid">
    {{block "content" . }}{{end}}
</div>
</body>
</html>

```


* index.tmpl

```html
{{/*继承 base.tmpl 模板 */}}
{{template "base.tmpl"}}

{{define "content"}}
    <div>Hello world!</div>
{{end}}
```

{% endraw %}

