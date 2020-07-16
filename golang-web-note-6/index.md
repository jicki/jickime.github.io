# Go 操作 Redis


# Go Web 编程

## Golang go-Redis

> 官方 github https://github.com/go-redis/redis

### docs 文档

> https://godoc.org/github.com/go-redis/redis


### Install

```shell
go get -u github.com/go-redis/redis
```

### import

```go
import "github.com/go-redis/redis"
```


### 初始化连接


#### 单 Redis 节点

```go

// 创建一个全局的 redis 客户端实例 Rdb
var Rdb *redis.Client

func initRdb() (err error) {
        // 这里是赋值, 而不是 创建一个变量, 所以只需要使用 = 号
	Rdb = redis.NewClient(&redis.Options{
		Addr: "127.0.0.1:6379",
		// 留空为没密码
		Password: "",
		// 默认的 DB 为 0
		DB: 0,
		// 连接池大小
		PoolSize: 100,
	})
	// 测试 redis 是否可以连接
	_, err = Rdb.Ping().Result()
	return err
}
```


#### Redis 哨兵模式

```go

// 创建一个全局的 redis 客户端实例 Rdb
var Rdb *redis.Client

func initRdb() (err error) {
        // 这里是赋值, 而不是 创建一个变量, 所以只需要使用 = 号
	Rdb = redis.NewFailoverClient(&redis.FailoverOptions{
		MasterName:    "master",
		SentinelAddrs: []string{"127.0.0.1:6379", "127.0.0.1:6380", "127.0.0.1:6381"},
	})
        // 测试 redis 是否可以连接
        _, err = Rdb.Ping().Result()
        return err
}

```


#### Redis Cluster 集群模式

* 集群模式的 客户端实例与单节点是不一样的 `*redis.ClusterClient`

```go

// 创建一个全局的 redis 客户端实例 Rdb
var Rdb *redis.ClusterClient


func initRdb() (err error) {
        // 这里是赋值, 而不是 创建一个变量, 所以只需要使用 = 号
        Rdb = redis.NewClusterClient(&redis.ClusterOptions{
                Addrs: []string{"127.0.0.1:7000", "127.0.0.1:7001", "127.0.0.1:7002", "127.0.0.1:7003"},
        })
        // 测试 redis 是否可以连接
        _, err = Rdb.Ping().Result()
        return err
}

```


### 连接redis

```go

func main() {
	if err := initRdb(); err != nil {
		fmt.Printf("initRdb Failed Error %v\n", err)
		return
	}
	fmt.Println("Connect Redis Client Success")
	defer Rdb.Close()
}
```


### 操作 Redis



#### Set/Get 操作


> Set 操作的函数

```go

func redisSet(key string, value interface{}, expr time.Duration) {
	err := Rdb.Set(key, value, expr).Err()
	if err != nil {
		fmt.Printf("redisSet Failed Error %v\n", err)
		return
	}
	fmt.Printf("Redis Set key: [%s] value: [%v] Success \n", key, value)
}

func main() {
	if err := initRdb(); err != nil {
		fmt.Printf("initRdb Failed Error %v\n", err)
		return
	}
	fmt.Println("Connect Redis Client Success")
	defer Rdb.Close()

	redisSet("RedisKey", "Value1", 0)

}

```

* 输出:

```shell

Connect Redis Client Success
Redis Set key: [RedisKey] value: [Value1] Success 

```


> Get 操作的函数


```go
func redisGet(key string) {
	value, err := Rdb.Get(key).Result()
	// 应该首先使用 redis.Nil 判断 key 是否存在
	if err == redis.Nil {
		fmt.Printf("Key: [%s] is Null \n", key)
		// 这里再判断 err != nil
	} else if err != nil {
		fmt.Printf("Redis Get Key Failed Error %v \n", err)
	} else {
		fmt.Printf("Key: [%s]  value: [%v] \n", key, value)
	}
}

func main() {
	if err := initRdb(); err != nil {
		fmt.Printf("initRdb Failed Error %v\n", err)
		return
	}
	fmt.Println("Connect Redis Client Success")
	defer Rdb.Close()
	// Get 一个不存在的 key
	redisGet("redisKey")
}

```


```shell
# 输出结果

Connect Redis Client Success
Key: [redisKey] is Null 

```



