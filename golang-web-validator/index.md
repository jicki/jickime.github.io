# Go Web validator 参数校验


# Validator

* Go Struct and Field validation, including Cross Field, Cross Struct, Map, Slice and Array diving。

  * `github` -  https://github.com/go-playground/validator 


## validator Fields


| Tag | Description |
|-|-|
|eqcsfield |	Field Equals Another Field (relative)|
|eqfield |	Field Equals Another Field|
|fieldcontains |NOT DOCUMENTED IN doc.go|
|fieldexcludes |NOT DOCUMENTED IN doc.go|
|gtcsfield |	Field Greater Than Another Relative Field|
|gtecsfield |	Field Greater Than or Equal To Another Relative Field|
|gtefield |	Field Greater Than or Equal To Another Field|
|gtfield |      Field Greater Than Another Field|
|ltcsfield |	Less Than Another Relative Field|
|ltecsfield |	Less Than or Equal To Another Relative Field|
|ltefield |	Less Than or Equal To Another Field|
|ltfield |	Less Than Another Field|
|necsfield |	Field Does Not Equal Another Field (relative)|
|nefield |      Field Does Not Equal Another Field|


* 注意: 一般来说 Struct 的结构体 `tag` 使用 `validate` 进行约束. 在 `Gin` 框架中, 使用 `binding` 就可以约束。 


```go

// 定义请求参数的结构体
type ParamSignUp struct {
	UserName   string `json:"username" binding:"required"`
	Password   string `json:"password" binding:"required"`
	RePassword string `json:"re_password" binding:"required"`
}

func main() {
	r := gin.Default()

	r.POST("/signup", func(c *gin.Context) {
		var p ParamSignUp
		if err := c.ShouldBind(&p); err != nil {
			c.JSON(http.StatusOK, gin.H{
				"Msg": err.Error(),
			})
			return
		}
	})
	r.Run()
}

```

