# HTTP 访问控制 CORS


# Go Web 编程

## HTTP 访问控制 CORS

* CORS - 跨域资源共享

  * 使用额外的 `HTTP` 头来告诉浏览器, 让运行在一个 `origin (domain)` 上的Web应用被准许访问来自不同源服务器上的指定的资源。

* 什么时候会出现跨域

  * 当一个资源从与该资源本身所在的服务器不同的域(域名)、协议(http/https)或端口(80/443等)请求一个资源时, 资源会发起一个跨域 HTTP 请求。
  
  * 如: https://www.jicki.cn 与 http://www.jicki.cn 属于协议不同, 跨域
  * 如: http://www.jicki.cn 与 http://www.jicki.cn:81 属于 端口不同, 跨域
  * 如: http://a1.jicki.cn 与 http://a2.jicki.cn 属于 域不同, 跨域


* 如何实现跨域请求

  1. 使用`Ajax`的 jsonp (使用该方式的缺点: 请求方式只能是`GET`请求)

  2. Nginx 反向代理 (将不同域 转换成 同域 的地址) 

  3. CORS ( 跨域资源共享 )


## CORS 解决跨域

> 浏览器会将`Ajax`请求分为两类，其处理方案略有差异：简单请求、特殊请求。


### 简单请求

* 简单请求: 只要同时满足以下两大条件，就属于简单请求。

  1. 请求方法是以(`HEAD`, `GET`, `POST`)三种方法之一
  
  2. HTTP的头信息不超出(`Accept`, `Accept-Language`, `Content-Language`, `Last-Event-ID`, `Content-Type`)几种字段


* 浏览器发现发现的`Ajax`请求是简单请求时，会在请求头中携带一个字段`Origin`。

* `Origin` 中会指出当前请求属于哪个域（协议+域名+端口）。服务会根据这个值决定是否允许其跨域。

* 如果服务器允许跨域，需要在返回的响应头中至少携带 `Access-Control-Allow-Origin`, `Access-Control-Allow-Credentials`, `Content-Type` 三个信息.

  * Access-Control-Allow-Origin - 可接受的域，是一个具体域名或者 `*` `*`代表任意域.

  * Access-Control-Allow-Credentials - 是否允许携带cookie，默认情况下, CORS 不会携带cookie, 除非这个值是true.


* 如果跨域请求要想操作cookie:

  1. 服务的响应头中需要携带Access-Control-Allow-Credentials 并且为 true .

  2. 浏览器发起`Ajax`需要指定`withCredentials` 并且为 true .

  3. 响应头中的 Access-Control-Allow-Origin 一定不能为`*`，必须是指定域名.


### 特殊请求

> 不符合简单请求的条件, 会被浏览器判定为特殊请求, 例如请求方式为PUT

* 特殊请求会在正式通信之前, 增加一次HTTP查询请求, 称为"预检" 请求（preflight）
  
  * 浏览器先询问服务器, 当前网页所在的域名是否在服务器的许可名单之中, 以及可以使用哪些HTTP动词和头信息字段。只有得到肯定答复, 浏览器才会发出正式的`XMLHttpRequest`请求, 否则就报错。


* 特殊请求 - 与简单请求相比, 除了三种信息外, 还会携带 `Access-Control-Request-Method` ,  `Access-Control-Request-Headers` 和 `Access-Control-Max-Age` 三个信息.

  * Access-Control-Request-Method - 特殊请求,的请求方式 如: PUT 

  * Access-Control-Request-Headers - 额外的 头信息.

  * Access-Control-Max-Age - 本次许可的有效时长, 单位是秒, 过期之前的`Ajax`请求就无需再次进行预检了.

* 如果浏览器得到上述响应, 则认定为可以跨域, 后续就跟简单请求的处理是一样的了


