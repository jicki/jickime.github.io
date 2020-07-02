# kafka 集群 多broker模式



# kafka 集群

## 环境说明

```
172.16.1.35   zookeeper   kafka

172.16.1.36   zookeeper   kafka

172.16.1.37   zookeeper   kafka
```

开放端口  2181  2888  3888   9092

 
## kafka 安装
略

 

## 修改配置

 1. 编辑  server.properties  文件  (以下为 172.16.1.35 的配置)

在默认的配置上，我只修改了4个地方。

broker.id = 
三个主机172.16.1.35,172.16.1.36,172.16.1.37
分别对应id为1，2，3

 
```
broker.id=1            
advertised.host.name=172.16.1.35   #配置为连接IP 否则会获取本地网卡IP
log.dirs=/opt/local/kafka/logs
zookeeper.connect=172.16.1.35:2181,172.16.1.36:2181,172.16.1.37:2181
```

非物理网卡，zk绑定IP 需要配置为 127.0.0.1 , 否则获取到的IP为 其他绑定IP

如：zookeeper.connect=127.0.0.1:2181,172.16.1.36:2181,172.16.1.37:2181


2. 编辑  consumer.properties 文件

配置  zookeeper.connect=  信息

如果 kafka 与 zookeeper 在同一台机器上，也可以不需要配置。


3. 编辑   producer.properties  文件

编辑 metadata.broker.list= 信息

配置为 多 broker 。

如 kafka_1:9092,kafka_2:9092,kafka_3:9092

 
4. 编辑  zookeeper.properties  文件

```
initLimit=5
syncLimit=2
server.1=0.0.0.0:2888:3888
server.2=172.16.1.36:2888:3888
server.3=172.16.1.37:2888:3888
dataDir=/tmp/zookeeper
```
 

非物理网卡，绑定IP 需要配置为 0.0.0.0 , 否则获取到的IP为 其他绑定IP


例
```
broker.id=1
server.1=0.0.0.0:2888:3888
server.2=172.16.1.36:2888:3888
server.3=172.16.1.37:2888:3888
```
 
```
broker.id=2
server.1=172.16.1.35:2888:3888
server.2=0.0.0.0:2888:3888
server.3=172.16.1.37:2888:3888
```


参数说明

initLimit：LF初始通信时限
集群中的follower服务器(F)与leader服务器(L)之间初始连接时能容忍的最多心跳数（tickTime的数量）。

syncLimit：LF同步通信时限
集群中的follower服务器与leader服务器之间请求和应答之间能容忍的最多心跳数（tickTime的数量）。

server.N=YYY:A:B
服务器名称与地址：集群信息（服务器编号，服务器地址，LF通信端口，选举端口）


分别将1，2，3写入三个主机的myid文件

echo "1" >> /tmp/zookeeper/myid


## 启动程序

分别启动三个服务器中的 zookeeper 和 kafka server 

```
/opt/local/kafka/bin/zookeeper-server-start.sh -daemon /opt/local/kafka/config/zookeeper.properties

/opt/local/kafka/bin/kafka-server-start.sh -daemon /opt/local/kafka/config/server.properties 
```
 


## kafka 命令


1、创建主题（Topic）

```
bin/kafka-topics.sh --zookeeper zk_host:port/chroot --create --topic my_topic_name --partitions 20 --replication-factor 3 --config x=y
```
 

 

2、查看所有主题

```
bin/kafka-topics.sh --list --zookeeper localhost:2181
```
 
 
3、查看指定主题：

```
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic 主题名
```
 

4、修改主题：

```
bin/kafka-topics.sh --zookeeper zk_host:port/chroot --alter --topic 主题名 --deleteConfig x
```
 

5 查看主题分区

```
./kafka-topics.sh --describe --zookeeper localhost
```

