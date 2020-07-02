# Golang 正则表达式


# 正则表达式

* 在 Golang 中 正则表达式 使用 `regexp` 包



## 常用正则语法

|语法|说明|表达式实例|
|-|-|-|-|
|`\d`|表示数字[0-9]|`a\dc`|
|`\D`|表示非数字的其他字符|`a\Dc`|
|`\w`|单词字符: 大小写字母、数字、下划线|`a\wc`|
|`\W`|非单词字符|`a\Wc`|
|`\s`|空白字符: `\t`、`\n`、`\r`、`\f` 其中之一|`a\sc`|
|`\S`|非空白字符|`a\Sc`|
|`.`|除了换行符之外的任意字符|`a.c`|
|`\.`| 表示符号 . |`abc\.com`|
|`+`|匹配前一个字符1次或无限次|`abc+`|
|`*`|匹配前一个字符0或无限次|`abc*`|
|`?`|匹配前一个字符0次或1次|`abc?`|
|`{m}`|匹配前一个字符m次|`ab{2}c`|
|`{m,n}`|匹配前一个字符m至n次。|`ab{1,2}c`|
|`{,n}`|匹配前面一个字符0至n次。|`ab{,2}c`|
|`{m,}`|匹配前面一个字符m至无限次|`ab{3,}`|
|`[...]`|匹配`[]`内的字符其中之一|`a[bcd]c`|
|`[\s\S]`|匹配任意字符|`12[\s\S]d`|
|`[a-z]`|匹配 a到z中的任意一个小写字符|`1[a-z]2`|
|`[^abc]`|匹配 除了abc之外的任意字符|`aa[^abc]bb`|
|`|`|代表左右表达式任意匹配一个|`abc|def`|
|`^`|匹配字符串开头|`^abc`|
|`$`|匹配字符串末尾|`abc$`|


## regexp 模块使用


```go
package main

import (
	"fmt"
	"regexp"
)

func main() {

	buf := "abc azc a7c aac 888 a9c  tac"

	//1) 解释规则, 解析正则表达式，如果成功返回解释器
	reg1 := regexp.MustCompile(`a[^a]c`)
	if reg1 == nil {
		fmt.Println("regexp err")
		return
	}

	//2) 根据规则提取关键信息
	result1 := reg1.FindAllStringSubmatch(buf, -1)
	fmt.Println("result1 = ", result1)
}
```

* 输出结果


```
result1 =  [[abc] [azc] [a7c] [a9c]]
```



## 模拟过滤页面


```go

package main

import (
	"fmt"
	"regexp"
)

func main() {
	//``   截取了一段页面原生字符串
	buf := `
<!DOCTYPE html>
<html lang="en">

<head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="google-site-verification" content="xBT4GhYoi5qRD5tr338pgPM5OWHHIDR6mNg1a3euekI" />
    <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
    <meta name="description" content="Your future depends on your dreams">
    <meta name="keywords"  content="小炒肉, 运维工程师, Jicki, DevOps, Docker, Kubernetes">
    <meta name="theme-color" content="#000000">
    
    <!-- Open Graph -->
    <meta property="og:title" content="About - 小炒肉 Blog | Jicki Blog">
    
    <meta property="og:type" content="website">
    <meta property="og:description" content="你是我的梦想">
    
    <meta property="og:image" content="https://jicki.cn/img/avatar-jicki.png">
    <meta property="og:url" content="https://jicki.cn/about/">
    <meta property="og:site_name" content="小炒肉 Blog | Jicki Blog">
    
    <title>About - 小炒肉 Blog | Jicki Blog</title>

<header class="intro-header" style="background-image: url('/img/about-bg.jpg')">
  <div class="container">
    <div class="row">
      <div class="col-lg-8 col-lg-offset-2 col-md-10 col-md-offset-1">
        
        <div class="site-heading">
        
          <h1>About</h1>
          <span class="subheading">你是我的梦想</span>
        </div>
      </div>
    </div>
  </div>
</header>


<!-- Main Content -->
<div class="container">
	<div class="row">
        

<!-- USE SIDEBAR -->
    <!-- PostList Container -->
    		<div class="
                col-lg-8 col-lg-offset-1
                col-md-8 col-md-offset-1
                col-sm-12
                col-xs-12
                postlist-container
            ">
    			<!-- Language Selector -->
<select class="sel-lang" onchange= "onLanChange(this.options[this.options.selectedIndex].value)">
    <option value="0" selected> 中文 Chinese </option>
    <option value="1"> 英文 English </option>
</select>

<!-- Chinese Version -->
<div class="zh post-container">
    
    <blockquote>
  <p>搞搞 docker, 弄弄 kubernetes,</p>

  <p>学学 golang, 写写 shell。</p>
</blockquote>

<h3 id="一-序">一、 序</h3>

<blockquote>
  <p><strong>Your future depends on your dreams</strong></p>
</blockquote>

<blockquote>
  <p><strong>不要活在别人的眼里，不要活在别人的嘴里</strong></p>
</blockquote>

<blockquote>
  <p><strong>要活在自己的心里，生活过的洒脱一点，不要为别人去活</strong></p>
</blockquote>

</body>

</html>
    `
	//解释正则表达式, 匹配标签内的字符
	reg1 := regexp.MustCompile(`<title>(?s:(.*?))</title>`)
	reg2 := regexp.MustCompile(`<p><strong>(?s:(.*?))</strong></p>`)
	if reg1 == nil {
		fmt.Println("MustCompile err")
		return
	}
	if reg2 == nil {
		fmt.Println("MustCompile err")
		return
	}

	//提取关键信息
	result1 := reg1.FindAllStringSubmatch(buf, -1)
	result2 := reg2.FindAllStringSubmatch(buf, -1)

	//过滤<></>
	for _, text := range result1 {
		//过滤不带标签的 不带<></>
		fmt.Println("text[1] = ", text[1])
	}
	//过滤<></>
	for _, text := range result2 {
		//过滤不带标签的 不带<></>
		fmt.Println("text[2] = ", text[1])
	}
}

```

* 输出如下:

```
text[1] =  About - 小炒肉 Blog | Jicki Blog
text[2] =  Your future depends on your dreams
text[2] =  不要活在别人的眼里，不要活在别人的嘴里
text[2] =  要活在自己的心里，生活过的洒脱一点，不要为别人去活

```

