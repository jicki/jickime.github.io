# Go学习笔记


# Go语言基础

## 1. 反射

* 反射是程序执行时检查其所拥有的结构. 尤其是类型的一种能力.
当 `【类型断言】` 无法满足时, 我们就使用 `【反射】` 获取 `动态类型` 和 `动态值` .

* 反射一般应用于 各类 web 框架, 配置文件解析库, ORM框架

* 反射的优点: 反射是一个强大并富有表现力的工具, 可以让代码更加灵活.

* 反射的缺点: 
    1. 基于反射的代码极其脆弱, 反射中类的类型错误会在程序运行时才会引发panic. 
    2. 在代码中大量使用 反射, 会使代码很难理解. 
    3. 反射的性能低下, 基于反射实现的代码通常比正常代码效率低一到两个数量级


1. 在反射中 `动态类型` 还划分为 `类型(Type)` 与 `种类(Kind)` , 因为在 Go 语言中我们可以使用 `type` 关键字构造很多自定义类型, `种类(Kind)` 是指最底层的类型, 在反射中, 需要区分 `指针` `结构体` 等 大品种的类型时, 就会使用 `种类(Kind)` .


```go
// 种类(Kind) 的类型包括如下
type Kind uint

const (
	Invalid Kind = iota
	Bool
	Int
	Int8
	Int16
	Int32
	Int64
	Uint
	Uint8
	Uint16
	Uint32
	Uint64
	Uintptr
	Float32
	Float64
	Complex64
	Complex128
	Array
	Chan
	Func
	Interface
	Map
	Ptr   //指针
	Slice
	String
	Struct
	UnsafePointer
)
```


### 1. reflect 包

* Go语言中反射的相关功能由内置的 `reflect` 包提供, 任意接口值在反射中都可以理解为由 `reflect.Type` 和 `reflect.Value` 两部分组成, 并且 `reflect.TypeOf` 和 `reflect.ValueOf` 两个函数来获取任意对象的Value和Type.


#### TypeOf
* Go语言中, 使用 `reflect.TypeOf()` 函数可以获得任意值类型对象 (reflect.Type), 程序通过类型对象可以访问任意值的类型信息.


```go
// 获取 任意值 的类型
func reflectType(x interface{}) {
	v := reflect.TypeOf(x)      // 获取 x 的动态信息
	fmt.Printf("Type: %v\n", v) // 打印 v 的类型
}

func main() {
	reflectType(11)           // 输出 Type: int
	reflectType("字符串")        //输出 Type: string
	reflectType([10]string{}) //输出 Type: [10]string
}

```


```go
// 使用 TypeOf(x).Name() 以及 TypeOf(x).Kind() 方法
// 获取动态类型的 具体类型(Type) 以及 种类(Kind)
func reflectType(x interface{}) {
	v := reflect.TypeOf(x) // 获取 x 的动态信息
	// 获取类型的  类型(Type) 以及 种类(Kind)
	fmt.Printf("Type name: %v, Kind: %v\n", v.Name(), v.Kind())
}

type cat struct {
	name string
}

func main() {
	c1 := cat{
		name: "小花",
	}
	i := 100
	/* 输出如下: 
	( 引用类型的 name 为 空 )
	Type name: cat, Kind: struct
	Type name: , Kind: ptr
	Type name: , Kind: slice
	Type name: , Kind: map
	*/
	reflectType(c1)
	reflectType(&i)
	reflectType([]string{})
	reflectType(map[string]int{})
}

```


#### ValueOf

* reflect.ValueOf() 函数 返回的是 `reflect.value` 类型, 包含了原始值的相关信息. `relect.value` 与 原始值之间可以相互转换.


`relect.value` 类型 获取原始值的方法有如下:

方法 | 说明 |
-|-|
Interface() interface{} | 将值以 interface{} 类型返回, 可通过类型断言转换指定类型  |
Int() int64 | 将值以 int 类型返回, 所有符号整型可以用此类型返回 |
Uint() uint64 | 将值以 uint64 类型返回, 所有无符号整型可以用此类型返回 |
Float() float64 | 将值以 float64 类型返回, 所有 浮点数 可用此类型返回|
Bool() bool | 将值以 (布尔值)bool 类型返回|
Bytes() []bytes| 将值以字节数组 []bytes 类型返回|
String() string| 将值以 (字符串)string 类型返回|



```go
func reflectValue(x interface{}) {
	v := reflect.ValueOf(x) // 获取 函数参数 x 的值信息
	k := v.Kind()           // 获取 值信息 对应的 种类(kind)
	// switch 类型断言 判断传入时的类型, 强制转换成 传入时的类型
	// 当类型传入 接口时，类型会转换成 接口类型.
	// 想获取传入时的类型,就必须要进行如下判断并强制转换为原来的类型
	switch k {
	case reflect.Int64:
		fmt.Printf("type = int64 , value = %d \n", 
		int64(v.Int()))
	case reflect.Float32:
		fmt.Printf("type = float32 , value = %f \n", 
		float32(v.Float()))
	case reflect.Float64:
		fmt.Printf("type = float64 , value = %f \n", 
		float64(v.Float()))
	}
}

func main() {
	var a float32 = 3.14
	var b int64 = 100
	reflectValue(a)
	reflectValue(b)
}

```


通过反射 修改设置变量的值


```go
// 在反射中 使用 Elem() 方法获取指针对应的值
func reflectValue(x interface{}) {
	v := reflect.ValueOf(x)
	// 利用 类型断言 判断 Kind 种类 是否等于 传入的类型
	if v.Elem().Kind() == reflect.Int64 {
		// 利用 SetInt 方法修改 传入的值
		v.Elem().SetInt(190)
	}
}

func main() {
	var a int64 = 100
	reflectValue(&a)
	fmt.Println(a)
}
```

IsNil() 与 IsValid()

* IsNil() 常用于判断 指针是否为空, IsValid() 常用于判断返回值是否有效.


```go
func main() {
	var a *int
	var b *int
	c := 100
	b = &c
	// 输出 a IsNil: true
	fmt.Println("a IsNil:", reflect.ValueOf(a).IsNil())
	// 输出 b IsNil: false
	fmt.Println("b IsNil:", reflect.ValueOf(b).IsNil())
}
```




`IsNil()`

* func(v Value) IsNil() bool{}

* IsNil() 用来判断 v 的值是否为 nil ( 引用类型 在没有初始化的时候值都为 nil ), v 持有的值分类必须是 通道(channel)、函数、接口、映射、指针、切片之一; 否则会引起 panic
  

`IsValid()`

* func(v Value) IsValid() bool{}

* IsValid() 返回 v 是否有一个值. 如果 v 是零值 返回 false, 此时 v 除了 IsValid、String、Kind 之外的方法都会引起 panic



```go

func main() {
	var a *int
	// 输出 a nil IsValid: true
	fmt.Println("a nil IsValid:", reflect.ValueOf(a).IsValid())

	// 创建一个匿名结构体 b
	b := struct{}{}
	// FieldByName() 判断结构体的 字段 是否 存在
	// 输出 是否存在的结构体字段: false
	fmt.Println("是否存在的结构体字段:", 
	reflect.ValueOf(b).FieldByName("123").IsValid())
	// MethodByName() 判断结构体的 方法 是否存在
	// 输出 是否存在的结构体方法: false
	fmt.Println("是否存在的结构体方法:", 
	reflect.ValueOf(b).MethodByName("123").IsValid())

}

```


### 结构体反射

* `reflect.TypeOf()` 获取反射对象信息,如果是类型是 结构体 , 可以通过反射 `reflect.Type()` 的 `NumField()` 和 `Field()` 方法获取结构体成员的详细信息. 



`reflect.Type` 获取结构体成员的方法: 

|方法|说明|
|-|-|
|Field(i int) StructField|根据索引,返回索引对应的结构体字段信息|
|NumField() int|返回结构体成员字段数量|
|FieldByName(name string) (StructField, bool)|根据指定字符串返回字符串对应结构体字段信息|
|FieldByIndex(index []int) StructField|多层成员访问, 根据 []int 提供的每个结构体的字段索引,返回字段信息|
|FieldByNameFunc(match func(string) bool)|根据传入的匹配函数查找需要的字段|
|NumMethod() int|返回该类型的方法 集中方法的数目|
|Method(int) Method|返回该方法集中的第N个方法|
|MethodByName(string)(Method, bool)|根据方法名返回该类型集中的方法|


`StructField` 类型

* `StructField` 类型用来描述结构体中的一个字段信息
* `StructField` 的定义:

```go
type StructField struct {
	Name 		string  	// 字段名字
	pkgPath 	string  	// 非导出字段的包路径
	Type 		Type		// 字段类型
	Tag 		StructTag	// 字段标签
	Offset  	uintptr		// 字段在结构体中的字节偏移量
	Index   	[]int		// 用于Type.FieldByIndex 的索引切片
	Anonymous 	bool		// 是否为匿名字段
}

```


* 例子1


```go

// 定义一个 student 的结构体
type student struct {
	Name  string `json:"name"` //为 json 输出定义一个 tag
	Score int    `json:"score"`
}

func main() {
	stu1 := student{
		Name:  "小学生",
		Score: 100,
	}
	// 反射 获取 stu1 的类型
	t := reflect.TypeOf(stu1)
	// t.Name() 获取 stu1 类型,  t.Kind() 获取 stu1 类型的种类.
	fmt.Println(t.Name(), t.Kind()) // 输出 student struct

	// 遍历结构体
	// 1. 通过结构体索引获取所有字段信息.
	// t.NumField() 会返回结构体成员的数量
	for i := 0; i < t.NumField(); i++ {
		//t.Field() 根据索引返回索引的字段信息
		field := t.Field(i)
		fmt.Printf("Key: %s index: %d  Type: %v  json Tag: %v \n",
			field.Name, field.Index, field.Type, 
			field.Tag.Get("json"))
		/* 输出:
		Key: Name index: [0]  Type: string  json Tag: name
		Key: Score index: [1]  Type: int  json Tag: score
		*/
	}
	// 2. 通过字段名的方式,获取指定结构体字段信息
	// t.FieldByName("字段名") 会返回指定字段名的结构体信息
	if soreField, ok := t.FieldByName("Score"); ok {
		fmt.Printf("Key: %s Index: %d Type: %v json tag: %v\n",
			soreField.Name, soreField.Index, soreField.Type, 
			soreField.Tag.Get("json"))
		/* 输出:
		Key: Score Index: [1] Type: int json tag: score
		*/
	}
}
```



* 例子2

```go

type student struct {
	Name  string
	Score int
}

// Study 创建学习方法
func (s student) Study() string {
	msg := "好好学习，天天向上"
	fmt.Println(msg)
	return msg
}

func (s student) Sleep() string {
	msg := "好好休息，劳逸结合"
	fmt.Println(msg)
	return msg
}

// 创建一个函数
func printMethod(x interface{}) {
	// 反射 获取传入参数 x 的 动态类型 x 与 动态值 v
	t := reflect.TypeOf(x)
	v := reflect.ValueOf(x)

	// t.NumMethod() 返回该类型方法的数目
	// 获取 Method 结构体类型必须是 值类型
	fmt.Printf("Method Num: %d\n", t.NumMethod())
	for i := 0; i < v.NumMethod(); i++ {
		// v.Method(i).Type() 获取 方法 的类型 以及返回值 类型
		methodType := v.Method(i).Type()
		// t.Method(i).Name 获取 方法的 名称
		methodName := t.Method(i).Name
		fmt.Printf("Method Name: %s\n", methodName)
		fmt.Printf("Method Type: %v\n", methodType)
		/* 输出
		Method Num: 2
		Method Name: Sleep
		Method Type: func() string
		Method Name: Study
		Method Type: func() string
		*/
	}
	// 通过反射调用获取的方法,反射调用方法传递参数必须是 []reflect.Value 类型
	var args = []reflect.Value{}
	// 通过 .Call 方法 调用方法.
	v.MethodByName("Study").Call(args)
	// Method(i int) i 为方法的 index
	v.Method(0).Call(args)
	/* 输出:
	好好学习，天天向上
	好好休息，劳逸结合
	*/
}

// 打印结构体里的方法
func main() {
	stu1 := student{
		Name:  "张三丰",
		Score: 100,
	}
	printMethod(stu1)
}

```


### 解析 conf 配置文件

```go
package main

import (
	"errors"
	"fmt"
	"io/ioutil"
	"reflect"
	"strconv"
	"strings"
)

/*
	1. 打开 xx.conf 文件
	2. 一行一行读取文件内容, 根据tag 找结构体对应的字段
	3. 根据找到的字段 赋值给结构体 指针(指针类型才可以修改)
	4. 利用 反射 设置对应的值

*/

//Config 配置文件 结构体 , 并设置 tag
type Config struct {
	FilePath string `conf:"file_path"`
	FileName string `conf:"file_name"`
	MaxSize  int64  `conf:"max_size"`
}

// 创建一个 赋值结构体的函数
func parseConf(fileName string, result interface{}) (err error) {
	// result 必须是 ptr
	// 利用反射 获取传入的变量的 动态类型 t
	t := reflect.TypeOf(result)
	// 利用反射 获取传入的变量的 动态值 v
	v := reflect.ValueOf(result)
	// 判断 (种类)Kind 是否为 Ptr (指针)类型
	if t.Kind() != reflect.Ptr {
		err = errors.New("result 必须为指针(ptr)类型 ")
		return
	}
	// 利用 Elem() 获取 Ptr(指针) 对应的信息
	telm := t.Elem()
	if telm.Kind() != reflect.Struct {
		err = errors.New("result 必须为指针结构体 ")
		return
	}

	// 读取文件 data = []byte 切片
	data, err := ioutil.ReadFile(fileName)
	if err != nil {
		err = fmt.Errorf("打开配置文件[%s]失败", fileName)
		return err
	}
	// 获取一个 []string 的切片
	s1 := strings.Split(string(data), "\n")
	//fmt.Println(s1)
	// 解析 []string 切片
	for index, line := range s1 {
		// 去除 空格
		line = strings.TrimSpace(line)
		// 忽略 空行 和 首字母为 # 的注释行
		if len(line) == 0 || strings.HasPrefix(line, "#") {
			continue
		}
		// 判断配置文件项是否正确的
		eqIndex := strings.Index(line, "=")
		if eqIndex == -1 {
			err = fmt.Errorf("line %d Config Error", index+1)
			return
		}
		// 检查是否按 = 分割每一行
		key := line[:eqIndex]
		value := line[eqIndex+1:]
		key = strings.TrimSpace(key)
		value = strings.TrimSpace(value)
		if len(key) == 0 {
			err = fmt.Errorf("line %d Config Error", index+1)
			return
		}
		// 利用 反射 给 result 指针 赋值
		// 遍历结构体的所有字段并且与 key 做比较
		// telm.NumField() 返回结构体成员的数量
		for i := 0; i < telm.NumField(); i++ {
			// 根据索引返回索引的字段信息
			field := telm.Field(i)
			// Get 结构体 指定 tag 的信息
			tag := field.Tag.Get("conf")
			// 判断配置文件中的 key 是否有结构体中的 tag 匹配
			if key == tag {
				// 获取结构体中字段的类型
				fieldType := field.Type
				// 类型断言, 并强制转换成 获取的字段类型
				switch fieldType.Kind() {
				case reflect.String:
					v.Elem().Field(i).SetString(value)
				case reflect.Int, reflect.Int8, reflect.Int16, reflect.Int32, reflect.Int64:
					// strconv 转换类型
					valueint, _ := strconv.ParseInt(value, 10, 64)
					v.Elem().Field(i).SetInt(valueint)
				}
			}
		}
	}
	return
}

func main() {
	// 初始化一个 结构体指针
	var str1 = &Config{}
	// 调用 parseConf() 函数,传入文件读取的数据到初始化的结构体中
	err := parseConf("./xx.conf", str1)
	if err != nil {
		fmt.Println(err)
	}
	// 打印赋值后的结构体
	fmt.Printf("%#v", str1)
}

```

