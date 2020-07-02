# Golang to Kafka


# Go Web 编程


## Kafka

* `Apache Kafka` 是由著名职业社交公司`Linkedln`开发的, 最初是被设计用于解决`Linkedln` 公司内部海量日志传输等问题。`Kafka` 使用 `Scala` 语言编写。 2011年`Linkedln` 将`Kafka`开源 并进入 `Apache` 孵化器, 2012年10月正式毕业,成为`Apache` 顶级项目。

### 消息队列通信模型


* 点对点模式 `(Queue)`

  * 生产者 生产消息发送到 `queue` 中,  消费者从 `queue` 中取出消息并且消费消息。 一条消息被消费以后, `queue` 中就没有了, 不会有重复消费。


* 发布/订阅 `(topic)`

  * 生产者 (发布消息) 将消息发布到 `topic` 中, 同时有多个 消费者 (订阅topic) 消费这条消息。相对于 点对点`(queue)`方式, 发布到 `topic` 中的消息会被 所有 订阅了该 `topic` 的消费者进行消费。



### Kafka 介绍

* `Kafka` 是一个分布式数据流服务, 可以运行在单台服务器上, 也可以在多个服务器中部署成集群模式。它提供了发布和订阅的功能, 使用者可以发送数据到 `Kafka` 中, 也可以从 `Kafka` 中读取数据。

* `Kafka` 特点:

  * 高吞吐量、低延迟 - 每秒可以生产约 25 万消息 (50 MB) , 每秒处理 55 万消息 (110 MB)。

  * 持久化数据存储 - 可进行持久化操作。将消息持久化到磁盘, 因此可用于批量消费, 例如 `ETL`, 以及实时应用程序。通过将数据持久化到硬盘以及 `replication` 防止数据丢失。

  * 高容错 - 分布式系统易于扩展, 所有的 `producer`、`broker` 和 `consumer` 都可以配置多个, 均为分布式的。无需停机即可扩展机器。 消息被处理的状态是在 `consumer` 端维护, 而不是由 `server` 端维护。当失败时能自动平衡。



### Kafka 架构

* `Producer` - 生产者

* `Kafka Cluster` - Kafka 集群

  * `Broker` - 每一个`Kafka Server` 可以成为一个`Broker`, 多个`Broker`就是 `Kafka Cluster`。(单机服务器也可以部署多个`Broker`, 多个 `Broker` 连接到同一个 `Zookeeper` 中,就形成了 `Kafka Cluster` )。

  * `Topic` - 消息类别名, 一个`Topic`存放一类的消息。每个`Topic`都有一个或者多个订阅者( consumer ), `Producer` 负责将消息推送到 `Topic`, 然后由 订阅了该 `Topic` 的 `consumer` 从该 `Topic` 中读取消息。一个 `Broker` 可以创建一个或多个`Topic` , 同一个`Topic` 可以分布在同一个 `Kafka Cluster` 下的多个`Broker` 中。

  * `Partition` - `Kafka` 为每个 `Topic` 维护多个 `Partition` 分区 ( 既数据分片 ), 每个分区都会映射到同一个 逻辑日志文件中。当一条 消息 被发布到 `Topic` 上, 这条消息会分布在其中一个 `Partition` 中, 并且 `Broker` 会将这条 消息 追到逻辑 日志文件的最后一个 `segment` 中。 

    * 每个 `Partition` 都是一个有序的、不可变的结构化的提交日志记录的序列。在每个`Partition`中每一条日志记录都会被分配一个序号——通常称为`offset`, `offset`在`Partition`内是唯一的。逻辑日志文件会被化分为多个文件`segment`（每个`segment`的大小一样的）。

    * `Broker` 集群将会保留所有已发布的 消息 `records`, 不管这些消息是否已被消费。保留时间依赖于一个可配的保留周期。

    * `Partition` 是分布式的存在于一个`Kafka Cluster`的多个`Broker`上。每个`Partition`会被复制多份存在于不同的`Broker`上。这样做是为了容灾。具体会复制几份, 会复制到哪些`Broker`上, 都是可以配置的。经过相关的复制策略后, 每个`Topic`在每个`Broker`上会驻留一到多个`Partition`。

    * 对于同一个`Partition`, 它所在任何一个`Broker`, 都有能扮演两种角色: `leader`、`follower`。

    * `Partition` 在服务器中表现形式为一个一个的文件夹,每个`Partition` 包含多个 `segment` 文件。每组的 `segment` 文件中 包含三种文件, `.index` 文件， `.log` 文件, `.timeindex` 文件, `.log` 文件是存储 具体 消息 的, `.index` 与 `.timeindex` 文件是 索引文件,用于检索与查找具体的消息。

* `Consumer` - 消费者, `consumer` 可以是一个,也可以形成一个 `consumer Group` ,每个组包含多个 `consumer`, 共同消费订阅的`Topic`消息, 提高效率。




### Kafka 生产消费流程

1. `Producer` 首先连接 `Kafka Cluster` 并获取 `Partition` 的信息, 查找具体的 `Leader` 。

2. `Producer` 将 消息 发送给 具体的 `Partition` 的 `Leader`。

3. `Partition` 的 `Leader` 将消息写入 本地磁盘中。

4. `Partition` 的 `follower` 此时会拉取 `Leader` 的消息。

5. `Partition` 的 `follower` 将消息写入 本地磁盘中, 并发送 ACK 给 `Leader`。

6. `Partition` 的 `Leader` 收到 所有 `follower` 的 ACK 后 给 `Producer` 发送 ACK。


* 注意: ACK = `RequiredAcks` , 一共包含三种确认方式, 分别是 `0`  不需要 ACK 确认。   `1` 只需要 `Leader` 确认既可。 `ALL`或`-1` 表示 既需要 `Leader` 确认 也需要 `follower` 确认。 



### Golang 操作 Kafka

* Go语言操作 `Kafka` 使用 `sarama` 这个库。

* install

```shell
go get -u github.com/shopify/sarama
```



* 发送消息到 kafka


```go
package main

import (
	"flag"
	"fmt"

	"github.com/Shopify/sarama"
)

func main() {
	// 初始化 Kafka Client 配置 实例
	kafkaConf := sarama.NewConfig()

	// 配置 Producer Ack 确认(分别为 NoResponse, WaitForLocal, WaitForAll)
	// WaitForAll 表示 leader 与 follower 都需要确认.
	kafkaConf.Producer.RequiredAcks = sarama.WaitForAll

	// 配置 Producer 写入哪一个 分区。
	// NewRandomPartitioner 表示 随机写入一个分区。
	kafkaConf.Producer.Partitioner = sarama.NewRandomPartitioner

	// 配置 Producer 成功以后返回 确认。
	kafkaConf.Producer.Return.Successes = true

	// 配置连接 Kafka Server
	client, err := sarama.NewSyncProducer([]string{"127.0.0.1:9092"}, kafkaConf)
	if err != nil {
		fmt.Printf("Producer 连接失败: %v\n", err)
		return
	}
	// 关闭 连接
	defer client.Close()

	// 创建一条 需要写入到 kafka 的信息。
	// The request attempted to perform an operation on an invalid topic.
	// 如上报错: Topic 必须符合 [a-zA-Z0-9\\._\\-] 规则。
	var value string
	flag.StringVar(&value, "value", "", "消息内容")
	flag.Parse()
	if len(value) == 0 {
		fmt.Println("消息不能为空,使用 -value='' 输入消息内容 ")
		return
	}
	msg := &sarama.ProducerMessage{
		Topic: "TestMessage",
		Key:   nil,
		Value: sarama.StringEncoder(value),
	}

	// 发送信息到 kafka
	Part, offset, err := client.SendMessage(msg)
	if err != nil {
		fmt.Printf("SendMessage Failed: %v\n", err)
		return
	}
	fmt.Printf("Send Message partition: %v, offset: %v \n", Part, offset)
}

```


```shell
# go run main.go -value="这是一条测试信息"

Send Message partition: 0, offset: 5

```



* consumer 消费消息

```go
package main

import (
	"flag"
	"fmt"
	"sync"

	"github.com/Shopify/sarama"
)

var (
	topic string
	wg    sync.WaitGroup
)

func main() {
	// 创建一个 consumer 实例, 连接到 Kafka Server
	consumer, err := sarama.NewConsumer([]string{"127.0.0.1:9092"}, nil)
	if err != nil {
		fmt.Printf("consumer 连接失败, err:%v\n", err)
		return
	}
	defer consumer.Close()

	flag.StringVar(&topic, "topic", "", "topic名称")
	flag.Parse()
	if len(topic) == 0 {
		fmt.Println("topic 不能为空, 使用 -topic='' 指定topic")
		return
	}
	// 获取指定 Topic 的分区列表 id
	partitionList, err := consumer.Partitions(topic)
	if err != nil {
		fmt.Printf("获取 Topic Partitions分区失败: err%v\n", err)
		return
	}
	// 打印 partition 列表的 id ( []int32 )
	fmt.Println(partitionList)

	// 遍历 partition 列表
	for partition := range partitionList {
		// 针对每个分区创建一个对应的分区消费者
		// OffsetNewest: 从当前的偏移量开始消费, OffsetOldest: 从最老的偏移量开始消费
		pc, err := consumer.ConsumePartition(topic, int32(partition), sarama.OffsetNewest)

		if err != nil {
			fmt.Printf("failed to start consumer for partition %d,err:%v \n", partition, err)
			return
		}
		// Possible resource leak, 'defer' is called in a for loop.
		defer pc.AsyncClose()
		// 异步从每个分区消费信息
		wg.Add(1)
		go func(sarama.PartitionConsumer) {
			for msg := range pc.Messages() {
				fmt.Printf("Partition:%d Offset:%d Key:%v Value:%v \n", msg.Partition, msg.Offset, string(msg.Key), string(msg.Value))
			}
			wg.Done()
		}(pc)
	}
	wg.Wait()
}

```

* 启动后会一直等待接收消息

```shell
# go run main.go -topic="TestMessage"

Partition:0 Offset:13 Key: Value:这是条测试消息 
Partition:0 Offset:14 Key: Value:这是条测试消息 

```


### tail 库的使用

* 使用 `hpcloud/tail` 第三方库, 实现了 类似于 Linux 命令中 `tail -f` 的效果。

* `install`  `go get -u github.com/hpcloud/tail` 


* 一个例子:

```go
package main

import (
	"fmt"
	"time"

	"github.com/hpcloud/tail"
)

func main() {
	file := "./access.log"
	// 初始化 tail 实例
	config := tail.Config{
		Location: &tail.SeekInfo{
			Offset: 0,
			Whence: 2,
		},
		ReOpen:      true,
		MustExist:   false,
		Poll:        true,
		Pipe:        false,
		RateLimiter: nil,
		Follow:      true,
		MaxLineSize: 0,
		Logger:      nil,
	}
	tails, err := tail.TailFile(file, config)
	if err != nil {
		fmt.Printf("tail file err: %v\n", err)
		return
	}
	var (
		msg *tail.Line
		ok  bool
	)
	for {
		// 一直往这里写数据
		msg, ok = <-tails.Lines
		if !ok {
			fmt.Printf("tail file close reopen, filename: %s \n",
				tails.Filename)
			time.Sleep(time.Second)
			continue
		}
		fmt.Println("msg:", msg.Text)
	}
}

```

