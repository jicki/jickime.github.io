# Go Jwt With Token



# JWT

* `JWT`全称`JSON Web Token`是一种跨域认证解决方案, 属于一个开放的标准, 它规定了一种 `Token` 实现方式, 目前多用于前后端分离项目和`OAuth2.0`业务场景下。

  * `JWT` 本身没有定义任何技术实现, 它只是定义了一种基于`Token`的会话管理的规则, 涵盖`Token`需要包含的标准内容和`Token`的生成过程, 特别适用于分布式的单点登录(OSS) 场景。



* `JWT` 优势

  * `JWT` 拥有基于`Token`的会话管理方式所拥有的一切优势, 不依赖 `Cookie`, 防止`CSRF`攻击, 也可以在禁止 `Cookie` 的浏览器环境中使用。

  * 服务端不需要存储 `Session`, 服务端认证鉴权业务扩展方便, 避免存储`Session` 所需要配置的如: `Redis`等组件, 降低系统架构复杂度。

* `JWT` 劣势

  * 由于`Token`鉴权有效期存储于 `JWT` 中, 所以 `JWT` `Token` 一段签发, 就会在有效期内一直有效, 无法在服务端进行中止, 只能通过客户端进行废除。 




## JWT 组成

* 一个 JWT 有三个部分组成, 以`.` 进行分割. 这三部分都是`单独`经过 `Base64` 编码。

  * `Header` 头部

    * `header` 中存储了所使用的加密算法和`Token`类型. JSON 格式: `{"alg": "HS256", "typ": "JWT"}`     

  * `Payload` 负载

    * `payload` 中包含官方提供的7个字段. (也可以自定义字段) JSON 格式: 

      * `iss(issuer)`: 签名人

      * `exp(expiration time)`: 过期时间

      * `sub(subject)`: 主题
 
      * `aud(audience)`: 受众

      * `nbf(Not Before)`: 生效时间

      * `iat(Issued At)`: 签发时间

      * `jti(JWT ID)`: 编号。

  * `Signature` 签名

    * `signature` 是对前面两部分(`Header`、`Payload`) 进行签名, 防止数据篡改。

      * 签名的过程首先需要有一个定义的密钥(Secret)。然后使用`Header` 里定义的算法进行签名。(默认为 HMAC SHA256) 


