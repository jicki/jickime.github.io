# Go Viper 配置管理库



{{< figure src="/img/posts/golang/viper-logo.png" title="Viper" >}}

# Viper

* Viper 是 Go 语言项目中相对完善的一个配置文件解决方案。 Viper 可以处理所有类型的配置需求和格式。

  * **注: 目前 Viper 是大小写不敏感的。**

* Viper 特性:

  * 设置默认值

  * 支持 `JSON`、`TOML`、`YAML`、`INT`、`HCL`、`envfile`、`Java properties` 格式的配置文件中读取配置信息。

  * 实时监控与重新读取加载配置文件, 既热加载。( 可选 )

  * 环境变量读取配置。

  * 支持从远程配置系统 (etcd、zk、Consul) 读取并监控配置信息变化。

  * 命令行标准输入读取配置。

  * `buffer` 里读取配置。

  * 支持直接通过代码 `Set` 配置值。 

* Viper 读取配置的优先级

  1. 显示调用 `Set` 设置值 ( 代码中通过 Viper.Set 设置值 )

  2. 程序命令行参数 ( flag - 程序运行时指定的参数 ) 

  3. 环境变量

  4. 配置文件

  5. 远程 `key/value` 存储系统

  6. 默认值



## 安装 Viper


* `go get` and `go module`


```shell

go get -u github.com/spf13/viper


```


## Viper 实践


> 设置默认值, 在项目中设置默认值非常的有必要。


```go
viper.SetDefault("ConfDir", "./conf/")

```

---

> 读取配置文件

* Viper 支持搜索多个路径, 但是目前 Viper 只支持读取一个配置文件。

* Viper 没有默认的配置文件搜索路径, 需要程序自行定义。



```go
func main() {

	// 读取配置文件
	// SetConfigFile 设置完整配置文件
	//viper.SetConfigFile("config.yaml")

	// SetConfigName 设置配置文件名称 - 不需要定义文件扩展名
	viper.SetConfigName("config")

	// SetConfigType 设置配置文件类型 - 当设置配置文件时必须设置此项
	viper.SetConfigType("yaml")

	// AddConfigPath 添加配置文件读取目录, 支持添加多个
	// 这里注意 搜索路径是从上到下搜索, 如果下面有配置写入会生成
	viper.AddConfigPath("./conf/")
        viper.AddConfigPath("./")

	// ReadInConfig 查找并读取配置文件
	if err := viper.ReadInConfig(); err != nil {
		panic(fmt.Errorf("Read Config Error %s \n", err))
	}
}
```


---


> 写入配置文件

* 写入配置文件一般用于程序运行时需要更新配置文件时使用。


```go

func main() {

	// 写入配置文件

	// WriteConfig 将当前配置 写入/覆盖 AddConfigPath 和 SetConfigName 设置的预定路径。
	if err := viper.WriteConfig(); err != nil {
		fmt.Printf("WriteConfig Failed Error %s \n", err)
		return
	}

	// SafeWriteConfig 将当前配置 写入设置的预定路径, 如果存在不会覆盖 而是报错。
	if err := viper.SafeWriteConfig(); err != nil {
		fmt.Printf("Safe WriteConfig Failed Error %s \n", err)
		return
	}

	// WriteConfigAs 将当前配置 写入/覆盖 到自定义的路径中
	if err := viper.WriteConfigAs("./config"); err != nil {
		fmt.Printf("WriteConfig As Failed Error %s \n", err)
		return
	}

	// SafeWriteConfigAs 将当前配置 写入 到自定义的路径中, 如果文件存在不会覆盖 而是报错。
	if err := viper.SafeWriteConfigAs("./config"); err != nil {
		fmt.Printf("Safe WriteConfig  As Failed Error %s \n", err)
		return
	}
}

```


---


> 监控并重新读取配置文件

* Viper 支持在运行时实时读取配置文件, 并且动态加载配置。


```go

func main(){

	// WatchConfig 监控配置文件变化
	viper.WatchConfig()

	// OnConfigChange 是监控配置文件有变化后调用的一个回调函数
	viper.OnConfigChange(func(e fsnotify.Event) {
		fmt.Println("Config file changed:", e.Name)
	})
}


```


---

> 应用使用例子


```go

func InitViper() {
	// 设置默认值
	viper.SetDefault("ConfDir", "./conf/")

	// 读取配置文件
	// SetConfigFile 设置完整配置文件
	//viper.SetConfigFile("config.yaml")

	// SetConfigName 设置配置文件名称 - 不需要定义文件扩展名
	viper.SetConfigName("config")
	// SetConfigType 设置配置文件类型 - 当设置配置文件时必须设置此项
	viper.SetConfigType("yaml")
	// AddConfigPath 添加配置文件读取目录, 支持添加多个
	viper.AddConfigPath("./conf/")

	// ReadInConfig 查找并读取配置文件
	if err := viper.ReadInConfig(); err != nil {
		panic(fmt.Errorf("Read Config Error %s \n", err))
	}

	// WatchConfig 监控配置文件变化
	viper.WatchConfig()

	// OnConfigChange 是监控配置文件有变化后调用的一个回调函数
	viper.OnConfigChange(func(e fsnotify.Event) {
		fmt.Println("配置文件发生变化:", e.Name)
	})

	// 写入配置文件
	// SafeWriteConfig 将当前配置 写入设置的预定路径, 如果存在不会覆盖 而是报错。
	if err := viper.SafeWriteConfig(); err != nil {
		fmt.Printf("Safe WriteConfig Failed Error %s \n", err)
		return
	}

}

func main() {
	InitViper()

	r := gin.Default()

	r.GET("/version", func(c *gin.Context) {
		c.String(http.StatusOK, viper.GetString("version"))
	})

	addr := fmt.Sprintf("%s:%s", viper.GetString("host"), viper.GetString("port"))

	if err := r.Run(addr); err != nil {
		fmt.Printf("Gin Run Failed Error %v \n", err)
		return
	}
}

```


---

>  别名 Alias

* 将配置项中某个键设置别名。


```go
// 将lodu 与 Verbose 注册别名, 既关联在一起
viper.RegisterAlias("loud", "Verbose")


// 设置其中一个配置项的 值
viper.Set("loud", true)

// 如下 Get 两个配置项的值都会等于 true
viper.GetBool("loud")
viper.GetBool("Verbose")

```


---

> 环境变量


* `SetEnvPrefix()` - 定义以什么为前缀的 环境变量

  * 这里设置的 环境变量前缀, 强制转换为 大写 。

* `BindEnv()` - 绑定一个环境变量

* `AllowEmptyEnv(bool)` - 是否允许为空


**注: 环境变量的使用建议全部使用大写, viper 也会强制转换为大写**

```go

func viperEnv() {
	viper.SetEnvPrefix("VIP")

	if err := viper.BindEnv("NAME"); err != nil {
		fmt.Printf("BindEnv Failed Error %v \n", err)
		return
	}
	if err := os.Setenv("VIP_NAME", "小炒肉"); err != nil {
		fmt.Printf("Os Set Env Failed Error %v \n", err)
		return
	}
	fmt.Printf("ViperEnv Get VIP_NAME = [%v] \n", viper.Get("NAME"))
}


func main() {

	viperEnv()
}
```


```shell
# 输出结果

ViperEnv Get VIP_NAME = [小炒肉] 
```


---


> 使用 Flags

* Viper 支持 `Cobra` 库的 `Pflag`

```go
func viperPFlag() {
	// pflag 设置
	pflag.Int("flagname", 123, "message flag name")
	// 生效配置
	pflag.Parse()

	// 绑定 pflag
	if err := viper.BindPFlags(pflag.CommandLine); err != nil {
		fmt.Printf("Bind PFlags Failed Error %v \n", err)
		return
	}
	fmt.Printf("flagname = [%v] \n", viper.GetInt("flagname"))
}

func main() {

	viperPFlag()
}

```

```shell
# 运行命令 go run main.go --flagname 99998 输出结果

flagname = [99998] 

```

---


### 定义配置源


```go

func yamlExample() {
	viper.SetConfigType("yaml")

	// 定义一个 yaml 的配置源,这里注意 yaml 对格式支持很严格,要对齐首行
	var yamlConfig = []byte(`
host: "127.0.0.1"
port: 9999
version: "v1.0"
`)
	// 通过 io.Reader 读取 buffer 内的配置
	config := bytes.NewBuffer(yamlConfig)
	if err := viper.ReadConfig(config); err != nil {
		fmt.Printf("Read Buffer Config Failed Error %s \n", err)
		return
	}
	// Get 配置信息
	fmt.Printf("Version: %s \n", viper.Get("version"))
}

func main() {

	yamlExample()
}

```


```shell
# 输出结果

Version: v1.0 

```


### 远程 Key/Value 存储

* 读取 `Key/Value` 存储中的配置信息。 

* 需要 匿名导入 `viper/remote` 包如: `import _ "github.com/spf13/viper/remote"`



> etcd


```go
viper.AddRemoteProvider("etcd", "http://127.0.0.1:4001", "/config/")

// 设置 etcd 中存储类型。
viper.SetConfigType("json")

err := viper.ReadRemoteConfig()

```


---


> Consul


```go
viper.AddRemoteProvider("consul", "http://127.0.0.1:8500", "MY_CONSUL_KEY")

viper.SetConfigType("json")

err := viper.ReadRemoteConfig()

```




