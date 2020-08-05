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



## Gin 框架使用例子


### 翻译器

```go
package validator

import (
	"fmt"

	"github.com/gin-gonic/gin/binding"
	"github.com/go-playground/locales/en"
	"github.com/go-playground/locales/zh"
	ut "github.com/go-playground/universal-translator"
	"github.com/go-playground/validator/v10"
	enTranslations "github.com/go-playground/validator/v10/translations/en"
	zhTranslations "github.com/go-playground/validator/v10/translations/zh"
)

// validator 翻译器

// 定义一个全局的翻译器 trans
var trans ut.Translator

// 初始化翻译器
func InitTrans(locale string) (err error) {
	// 更改Gin框架中validator引擎的属性,实现翻译
	if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
		// 中文
		zhT := zh.New()
		// 英文
		enT := en.New()

		uni := ut.New(enT, zhT, enT)

		// locale 取决于 http 请求中的 'Accept-Language' 决定
		var ok bool
		trans, ok = uni.GetTranslator(locale)
		if !ok {
			return fmt.Errorf("uni.GetTranslator(%s) Failed", locale)
		}
		// 注册翻译器
		switch locale {
		case "en":
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		case "zh":
			err = zhTranslations.RegisterDefaultTranslations(v, trans)
		default:
			err = enTranslations.RegisterDefaultTranslations(v, trans)
		}
		return
	}
	return
}
```



```go

// 定义请求参数的结构体
type ParamSignUp struct {
	UserName   string `json:"username" binding:"required"`
	Password   string `json:"password" binding:"required"`
	RePassword string `json:"re_password" binding:"required"`
}

func main() {
        // 初始化 validator 翻译器

	if err := validator.InitTrans("zh"); err!=nil {
            fmt.Println("InitTrans Failed Error", err)
            return
	}
	r := gin.Default()

	r.POST("/signup", func(c *gin.Context) {
		var p ParamSignUp
		if err := c.ShouldBind(&p); err != nil {
                   // 判断这个错误是否属于 vilidator 类型校检错误。
		   verr, ok := err.(vilidator.ValidationErrors)
                   if !ok {
		   // 如果不是 vilidator 类型错误就返回正常的错误类型
                      c.JSON(http.StatusOK, gin.H{
                                "Msg": err.Error(),
                    })
		    return
		  }
                  // 如果是 vilidator 类型错误
		   c.JSON(http.StatusOK, gin.H{
		           "Msg": verr.Translate(trans),
		})
		return
	     }
	})
	r.Run()
}

```






