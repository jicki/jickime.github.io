# Codis 3.1




> 官方 配置文档, 使用说明 https://github.com/CodisLabs/codis/blob/release3.2/doc/tutorial_zh.md


# 环境配置

## 安装 git

```
yum install -y git autoconf
```


## 安装 golang，(使用 1.5.x 版本)

```
wget https://storage.googleapis.com/golang/go1.5.4.linux-amd64.tar.gz
tar zxvf go1.5.4.linux-amd64.tar.gz
mkdir /opt/local
mv go /opt/local/
```

## 配置环境变量

```
vi /etc/profile

# 添加如下:

# golang ENV
export GOROOT=/opt/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=/opt/local/codis


# 立即生效

source /etc/profile

# 检查版本

go version
go version go1.5.4 linux/amd64

echo $GOPATH
/opt/local/codis
```

# 安装 Codis

## 下载 Codis 源码

```
mkdir -p $GOPATH/src/github.com/CodisLabs

cd $GOPATH/src/github.com/CodisLabs

git clone https://github.com/CodisLabs/codis.git -b release3.1

```

## 编译 Codis 源码

```
cd $GOPATH/src/github.com/CodisLabs/codis
make
make -j -C extern/redis-3.2.4/

# 输入如下信息：

go build -i -o bin/codis-dashboard ./cmd/dashboard
go build -i -o bin/codis-proxy ./cmd/proxy
go build -i -o bin/codis-admin ./cmd/admin
go build -i -o bin/codis-ha ./cmd/ha
go build -i -o bin/codis-fe ./cmd/fe


# 查看 安装以后的版本

cat bin/version

version = 2017-03-08 14:07:13 +0800 @b1919d11593dfd1f47a2461837233dfc8fc78002 @3.1.5-26-gb1919d1
compile = 2017-04-05 18:13:46 +0800 by go version go1.5.4 linux/amd64


# 复制文件，方便管理

mkdir -p /opt/local/codis/{bin,logs,data}/

cp -rf $GOPATH/src/github.com/CodisLabs/codis/bin/* /opt/local/codis/bin

cp -rf $GOPATH/src/github.com/CodisLabs/codis/config /opt/local/codis/


```





# 配置 Codis

## Codis Dashboard

```
cd /opt/local/codis/config/

vim dashboard.toml
```


```
# 修改配置文件


##################################################
#                                                #
#                  Codis-Dashboard               #
#                                                #
##################################################

# Set Coordinator, only accept "zookeeper" & "etcd" & "filesystem".
coordinator_name = "zookeeper"
coordinator_addr = "127.0.0.1:2181"

# Set Codis Product Name/Auth.
product_name = "codis-demo"
product_auth = ""

# Set bind address for admin(rpc), tcp only.
admin_addr = "0.0.0.0:18080"

# Set configs for redis sentinel.
sentinel_quorum = 2
sentinel_parallel_syncs = 1
sentinel_down_after = "30s"
sentinel_failover_timeout = "5m"
sentinel_notification_script = ""
sentinel_client_reconfig_script = ""
```



```
# 启动 Dashboard

nohup /opt/local/codis/bin/codis-dashboard --ncpu=4 --config=/opt/local/codis/config/dashboard.toml \
    --log=/opt/local/codis/logs/dashboard.log --log-level=WARN &

```


## Codis Proxy

```
cd /opt/local/codis/config/

vim proxy.toml
```

```
# 修改配置文件

##################################################
#                                                #
#                  Codis-Proxy                   #
#                                                #
##################################################

# Set Codis Product Name/Auth.
product_name = "codis-demo"
product_auth = ""

# Set bind address for admin(rpc), tcp only.
admin_addr = "0.0.0.0:11080"

# Set bind address for proxy, proto_type can be "tcp", "tcp4", "tcp6", "unix" or "unixpacket".
proto_type = "tcp4"
proxy_addr = "0.0.0.0:19000"

# Set jodis address & session timeout, only accept "zookeeper" & "etcd".
jodis_name = "zookeeper"
jodis_addr = "127.0.0.1:2181"
jodis_timeout = "20s"
jodis_compatible = false

# Set datacenter of proxy.
proxy_datacenter = ""

# Set max number of alive sessions.
proxy_max_clients = 1000

# Set max offheap memory size. (0 to disable)
proxy_max_offheap_size = "1024mb"

# Set heap placeholder to reduce GC frequency.
proxy_heap_placeholder = "256mb"

# Proxy will ping backend redis (and clear 'MASTERDOWN' state) in a predefined interval. (0 to disable)
backend_ping_period = "5s"

# Set backend recv buffer size & timeout.
backend_recv_bufsize = "128kb"
backend_recv_timeout = "30s"

# Set backend send buffer & timeout.
backend_send_bufsize = "128kb"
backend_send_timeout = "30s"

# Set backend pipeline buffer size.
backend_max_pipeline = 1024

# Set backend never read replica groups, default is false
backend_primary_only = false

# Set backend parallel connections per server
backend_primary_parallel = 1
backend_replica_parallel = 1

# Set backend tcp keepalive period. (0 to disable)
backend_keepalive_period = "75s"

# If there is no request from client for a long time, the connection will be closed. (0 to disable)
# Set session recv buffer size & timeout.
session_recv_bufsize = "128kb"
session_recv_timeout = "30m"

# Set session send buffer size & timeout.
session_send_bufsize = "64kb"
session_send_timeout = "30s"

# Make sure this is higher than the max number of requests for each pipeline request, or your client may be blocked.
# Set session pipeline buffer size.
session_max_pipeline = 1024

# Set session tcp keepalive period. (0 to disable)
session_keepalive_period = "75s"

# Set session to be sensitive to failures. Default is false, instead of closing socket, proxy will send an error response to client.
session_break_on_failure = false

# Set metrics server (such as http://localhost:28000), proxy will report json formatted metrics to specified server in a predefined period.
metrics_report_server = ""
metrics_report_period = "1s"

# Set influxdb server (such as http://localhost:8086), proxy will report metrics to influxdb.
metrics_report_influxdb_server = ""
metrics_report_influxdb_period = "1s"
metrics_report_influxdb_username = ""
metrics_report_influxdb_password = ""
metrics_report_influxdb_database = ""


```


```
# 启动 codis-proxy

nohup /opt/local/codis/bin/codis-proxy --ncpu=4 --config=/opt/local/codis/config/proxy.toml \
    --log=/opt/local/codis/logs/proxy.log --log-level=WARN &

# 日志输出如下：

proxy waiting online ...

# 必须添加到集群中，才正常。
# 可使用 Codis-fe 界面添加 或者使用 Codis-admin 添加

```

## Codis-Server

```
# 启动 server 与 启动 redis 方法相同

nohup /opt/local/codis/bin/codis-server /opt/local/codis/config/redis.conf &


# 启动 Codis-Server 以后需要使用 Codis-fe  或者 Codis-admin 添加到集群

```


```
# 配置文件


bind 127.0.0.1
protected-mode yes
port 6379
tcp-backlog 2048
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /var/run/redis_6379.pid
loglevel notice
logfile "/opt/local/codis/logs/redis.log"
maxmemory 10gb
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename redis_dump.rdb
dir /opt/local/codis/data
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
appendonly no
appendfilename "appendonly.aof"
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
aof-load-truncated yes
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes

```








## Codis-FE

```
# 启动 codis-fe

nohup /opt/local/codis/bin/codis-fe --ncpu=4 --log=/opt/local/codis/logs/fe.log --log-level=WARN --zookeeper=127.0.0.1:2181 --listen=127.0.0.1:8080 &
```


## Codis-admin

```
# 添加 codis-proxy

/opt/local/codis/bin/codis-admin --dashboard=127.0.0.1:18080 --create-proxy -x 127.0.0.1:11080

# 日志输入如下：
jodis create node /jodis/codis-demo/proxy-15fba3007f3c6ee0887749681cb82307
proxy is working ... 
```



```
# 修复异常退出的 Codis-dashboard 
# dashboard 非正常退出，或者 kill -9 时使用

/opt/local/codis/bin/codis-admin --remove-lock --product=codis-demo --zookeeper=127.0.0.1:2181

```




```
# 修复异常退出的 Codis-proxy
# proxy 非正常退出，或者 kill -9 时使用

/opt/local/bin/codis-admin --dashboard=127.0.0.1:18080 --remove-proxy --addr=127.0.0.1:11080 --force

```

