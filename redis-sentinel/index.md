# redis sentinel 集群监控




# redis sentinel 集群监控

## 环境说明

```
ip  172.16.1.31 26379  redis sentinel
ip  172.16.1.30 6379   主 1 
ip  172.16.1.31 6380   从 1
ip  172.16.1.31 6379   主 2
ip  172.16.1.30 6380   从 2
```
 

redis 主 服务器配置，按照默认的配置文件既可。

redis 从 服务器配置，需要在配置文件配置 slaveof 的配置,配置为主服务器IP 与 端口

配置完成以后，启动主服务，再启用从服务


查看主redis信息

```
redis-cli -h 172.16.1.30 info Replication  
```
 

## 配置 redis sentinel


redis 源码安装包 里面会包含 sentinel.conf 复制一份


编辑 sentinel.conf

```
#redis-0

sentinel announce-ip 172.16.1.31

port 26379

#master1

sentinel monitor master1 172.16.1.30 6379 1

sentinel down-after-milliseconds master1 5000

sentinel parallel-syncs master1 2

sentinel failover-timeout master1 900000

#master2

sentinel monitor master2 172.16.1.31 6379 1

sentinel down-after-milliseconds master2 5000

sentinel parallel-syncs master2 2

sentinel failover-timeout master2 900000
```
 


sentinel announce-ip 设置消息中使用指定的ip地址，而不是自动发现的本地地址。


sentinel monitor   设置redis 群集名字，IP ，端口 ， 1 表示 多少台 sentinel 决定故障，如果设置为2 表示需要2台sentinel 监控到故障才会进行切换



## 启动 sentinel 

```
/usr/local/bin/redis-sentinel /opt/local/redis/conf/sentinel.conf --sentinel
```

