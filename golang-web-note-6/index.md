# Go 操作 Redis


# Go Web 编程

## Golang go-redis

> Go 操作 redis 
> 
> 官方 github https://github.com/go-redis/redis


### Install

```shell
go get -u github.com/go-redis/redis
```

### import

```go
import "github.com/go-redis/redis"
```


### 连接redis

```go
package main

import (
	"fmt"

	"github.com/go-redis/redis"
)

func main() {
	// 创建一个 redis 连接实例
	client := redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",
		DB:       0,
	})

	// 连接 redis,并使用 Result() 返回一个 PONG 表示成功, 和一个 err
	pong, err := client.Ping().Result()
	if err != nil {
		panic(err)
	}
	// 打印 pong 返回
	fmt.Println(pong)
}
```


### 操作例子

```go
package main

import (
	"fmt"
	"video/util"

	"github.com/go-redis/redis"
)

// redis 连接的函数, 返回一个 redis client 的指针实例
func NewRedis(addr, password string, db int) (client *redis.Client, err error) {
	// 创建一个 redis 连接实例
	client = redis.NewClient(&redis.Options{
		Addr:     addr,
		Password: password,
		DB:       db,
	})

	_, err = client.Ping().Result()

	if err != nil {
		util.Log().Panic("连接Redis不成功 %v\n", err)
	}

	return
}

func main() {
	client, err := NewRedis("127.0.0.1:6379", "", 0)
	if err != nil {
		panic(err)
	}
	// 调用 redis Set 命令, 0 是 expiration key 的过期时间
	err = client.Set("key01", "value01", 0).Err()
	if err != nil {
		panic(err)
	}

	// 调用 redis Get 命令
	value, err := client.Get("key01").Result()
	if err != nil {
		panic(err)
	}
	fmt.Printf("key01:%#v\n", value)

	// 利用 redis.Nil 判断 key 是否存在
	v2, err := client.Get("key_").Result()
	if err == redis.Nil {
		fmt.Println("key_ 不存在")
	} else if err != nil {
		panic(err)
	} else {
		fmt.Printf("key_:%#v\n", v2)
	}
}

```

* 输出:

```shell

key01:"value01"
key_ 不存在

```

### docs 文档

> https://godoc.org/github.com/go-redis/redis

