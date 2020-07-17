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




### Redis Pipeline


* `Pipeline` 管道技术, 指的是客户端允许将多个请求依次发给服务器, 过程中而不需要等待请求的回复, 在最后再一并读取结果即可。


> Pipeline 例子


```go
func redisPipeline() {
	pipe := Rdb.Pipeline()
	pipe.Set("key1", "value1", 0)
	pipe.Set("key2", "value2", 0)
	pipe.Set("key3", "value3", 0)
	pipe.Get("key1")

	cmder, err := pipe.Exec()
	if err != nil {
		fmt.Printf("Pipeline Exec Failed Error %v\n", err)
		return
	}
	for _, cmd := range cmder {
		fmt.Printf("Pipe Exec: [%v] \n", cmd)
	}
}

func main() {
	if err := initRdb(); err != nil {
		fmt.Printf("initRdb Failed Error %v\n", err)
		return
	}
	fmt.Println("Connect Redis Client Success")
	defer Rdb.Close()
	redisPipeline()
}
```

```shell
# 输出结果
Connect Redis Client Success
Pipe Exec: [set key1 value1: OK] 
Pipe Exec: [set key2 value2: OK] 
Pipe Exec: [set key3 value3: OK] 
Pipe Exec: [get key1: value1] 
```




### Redis 事务操作


#### TxPipeline


* Redis 是单线程, 因此单个命令始终是原子的, 但是来自不同客户端的两个命令可以依次执行, 但是 `Multi/exec` 能够确保在`multi/exec`两个语句之间的命令没有其他客户端正在执行命令。基于这样的场景下我们需要使用`TxPipeline`来解决。

  * `TxPipeline` 类似于 `Pipeline` 的方式,  不过 `TxPipeline` 内部会使用 `Multi/exec` 封装排列的命令。


```go

func redisTxPipeline() {
	pipe := Rdb.TxPipeline()
	pipe.Set("key1", "value1", 0)
	pipe.Set("key2", "value2", 0)
	pipe.Set("key3", "value3", 0)
	pipe.Get("key1")

	cmder, err := pipe.Exec()
	if err != nil {
		fmt.Printf("Pipeline Exec Failed Error %v\n", err)
		return
	}

	for _, cmd := range cmder {
		fmt.Printf("Pipe Exec: [%v] \n", cmd)
	}
}

func main() {
	if err := initRdb(); err != nil {
		fmt.Printf("initRdb Failed Error %v\n", err)
		return
	}
	fmt.Println("Connect Redis Client Success")
	defer Rdb.Close()

	redisTxPipeline()
}
```

```shell
# 输出结果

Connect Redis Client Success
Pipe Exec: [set key1 value1: OK] 
Pipe Exec: [set key2 value2: OK] 
Pipe Exec: [set key3 value3: OK] 
Pipe Exec: [get key1: value1] 

```




#### WATCH

* 某些场景下, 我们除了要使用 `Multi/Exec` 命令外, 还需要配合 `WATCH` 命令使用。

* 如: 在使用 `WATCH` 命令监视某个键之后, 直到执行 `Exec` 命令的这段时间内, 如果有其他用户抢先对被监视的键进行了替换、更新、删除等操作, 那么当用户尝试执行`Exec` 的时候, 事务将返回一个错误, 用户可以根据这个错误选择重试事务或者放弃事务。


```go
func txWatch() {
	key := "watch_count"
	err := Rdb.Watch(func(tx *redis.Tx) error {
		n, err := tx.Get(key).Int()
		if err != nil && err != redis.Nil {
			return err
		}
		_, err = tx.Pipelined(func(pipe redis.Pipeliner) error {
			pipe.Set(key, n+1, 0)
			return nil
		})
		return err
	}, key)
	if err != nil {
		fmt.Printf("WATCH Failed Error %v\n", err)
		return
	}
	fmt.Printf("WATCH Get watch_count: %v \n", Rdb.Get(key).Val())
}
```

