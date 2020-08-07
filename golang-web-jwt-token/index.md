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


## Go 使用 JWT

* Go 语言使用 `jwt-go` 第三方库实现 `JWT` 的生成 和 解析。

```go
package jwt

import (
	"errors"
	"time"

	"github.com/dgrijalva/jwt-go"
)

// 定义 Token 的过期时间  2 小时
const TokenExpireDuration = time.Hour * 2

// 定义一个 密钥 Secret
var mySecret = []byte("小炒肉的秘密")

// 定义一个结构体用于自定义的 payload 信息
type MyClaims struct {
	UserID   int64  `json:"user_id"`
	UserName string `json:"username"`
	// jwt.standardClaims 包含 payload 官方提供的7个字段
	jwt.StandardClaims
}

// GenToken: 生成 Token 的函数
func GenToken(userID int64, username string) (string, error) {
	// 创建一个 MyClaims 的实例 payload
	c := MyClaims{
		UserID:   userID,
		UserName: username,
		StandardClaims: jwt.StandardClaims{
			// 过期时间
			ExpiresAt: time.Now().Add(TokenExpireDuration).Unix(),
			// 签发人
			Issuer: "DouHu",
		},
	}
	// 创建  Token ( jwt.SigningMethodES256 是加密算法 )
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, c)

	// 使用自己定义 密钥 Secret 进行加密返回完整的 token
	return token.SignedString(mySecret)
}

// ParseToken: 解析 JWT 的函数
func ParseToken(tokenString string) (*MyClaims, error) {
	// 定义一个
	var mc = new(MyClaims)
	// 解析 Token 将传入的 tokenString 解析到 &MyClaims 这个结构体中去
	token, err := jwt.ParseWithClaims(tokenString, mc, func(token *jwt.Token) (interface{}, error) {
		return mySecret, nil
	})
	if err != nil {
		return nil, err
	}
	if token.Valid {
		return mc, nil
	}
	return nil, errors.New("ParseToken invalid token")
}

```





## Gin JWT 认证

* 在 Gin 框架中, 实现一个 认证 JWT Token 的中间件。后续在需要认证的路由中直接嵌入这个中间件就可以。


```go

// 认证 JWT Token 中间件
func JWTAuthMiddleware() func(c *gin.Context) {
	return func(c *gin.Context) {
		// 客户端携带 Token 的三种方式 1.请求头. 2. 请求体. 3. URL中.
		// 请求头中: 既 Header: Authorization 下 Bearer  后
		// Authorization: Bearer token(header).token(payload).token(signature)
		authHeader := c.Request.Header.Get("Authorization")
		// 如果为空
		if authHeader == "" {
			c.JSON(http.StatusOK, gin.H{
				"Code": 2003,
				"Msg":  "Header Authorization 为空",
				"Data": "nil",
			})
			c.Abort()
			return
		}
		// 如果 authHeader 不为空, Split 以空格 切割为2部分
		parts := strings.SplitN(authHeader, " ", 2)
		// 如果切割后 不止2部分, 或者 第1 部分未 包含 Bearer
		if !(len(parts) == 2 && parts[0] == "Bearer") {
			c.JSON(http.StatusOK, gin.H{
				"Code": 2004,
				"Msg":  "Header Bearer 格式有误",
				"Data": "nil",
			})
			c.Abort()
			return
		}
		// 解析 parts 第2部分的 JWT Token
                // jwt.ParseToken() 是上面我们实现的一个解析Jwt Token的函数
		mc, err := jwt.ParseToken(parts[1])
		if err != nil {
			c.JSON(http.StatusOK, gin.H{
				"Code": 2005,
				"Msg":  "无效的 Token",
				"Data": "nil",
			})
			c.Abort()
			return
		}
		// 将解析后获取的信息, 保存到 上下文 (c *gin.Context) 中
		c.Set("userID", mc.UserID)
		// 执行下一个函数
		c.Next()
	}
}

```


