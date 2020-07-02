# Open-Falcon 监控




# 环境准备

> 官方文档 https://book.open-falcon.org/zh_0_2

## 安装 golang

```
wget https://storage.googleapis.com/golang/go1.6.4.linux-amd64.tar.gz

tar zxvf go1.6.4.linux-amd64.tar.gz

mv go /opt/local/

# 增加环境变量

vi /etc/profile


# Golang ENV
export GOROOT=/opt/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=/opt/local/golang

go version
go version go1.6.4 linux/amd64

```


## 安装 Mysql 5.7

```
# 初始化依赖

yum -y install cmake ncurses ncurses-devel bison bison-devel boost boost-devel


# 创建 mysql 用户以及相关目录

/usr/sbin/groupadd mysql
/usr/sbin/useradd -g mysql mysql
mkdir -p /opt/local/mysql/data
mkdir -p /opt/local/mysql/binlog
mkdir -p  /opt/local/mysql/logs
mkdir -p /opt/local/mysql/relaylog
mkdir -p /var/lib/mysql
mkdir -p /opt/local/mysql/etc


# 下载 源码包

wget ftp://ftp.mirrorservice.org/sites/ftp.mysql.com/Downloads/MySQL-5.7/mysql-5.7.19.tar.gz

tar zxvf mysql-5.7.19.tar.gz

cd mysql-5.7.19

cmake -DCMAKE_INSTALL_PREFIX="/opt/local/mysql" -DDEFAULT_CHARSET=utf8 \
-DMYSQL_DATADIR="/opt/local/mysql/data/" -DCMAKE_INSTALL_PREFIX="/opt/local/mysql" \
-DINSTALL_PLUGINDIR=plugin -DWITH_INNOBASE_STORAGE_ENGINE=1 -DDEFAULT_COLLATION=utf8_general_ci \
-DENABLED_LOCAL_INFILE=1 -DENABLED_PROFILING=1 -DWITH_ZLIB=system \
-DWITH_EXTRA_CHARSETS=none -DMYSQL_MAINTAINER_MODE=OFF -DEXTRA_CHARSETS=all \
-DWITH_PERFSCHEMA_STORAGE_ENGINE=1 -DWITH_MYISAM_STORAGE_ENGINE=1 \
-DDOWNLOAD_BOOST=1 -DWITH_BOOST=/usr/local/boost


make -j `cat /proc/cpuinfo | grep processor| wc -l`

make install


# 创建相关目录，授权

chmod +w /opt/local/mysql
chown -R mysql:mysql /opt/local/mysql
chmod +w /var/lib/mysql
chown -R mysql:mysql /var/lib/mysql
cp /opt/local/mysql/support-files/mysql.server  /etc/init.d/mysqld
chmod 755 /etc/init.d/mysqld
echo 'basedir=/opt/local/mysql/' >> /etc/init.d/mysqld
echo 'datadir=/opt/local/mysql/data' >>/etc/init.d/mysqld


# 创建关联

ln -s /opt/local/mysql/lib/mysql /usr/lib/mysql
ln -s /opt/local/mysql/include/mysql /usr/include/mysql
ln -s /opt/local/mysql/bin/mysql /usr/bin/mysql
ln -s /opt/local/mysql/bin/mysqldump /usr/bin/mysqldump
ln -s /opt/local/mysql/bin/myisamchk /usr/bin/myisamchk
ln -s /opt/local/mysql/bin/mysqld_safe /usr/bin/mysqld_safe
ln -s /tmp/mysql.sock /var/lib/mysql/mysql.sock


# 初始化数据库

vi /opt/local/mysql/etc/my.cnf

[client]
default-character-set=utf8mb4

[mysqld]
########basic settings########
server-id = 1
port = 3306
user = mysql
bind_address = 127.0.0.1
autocommit = 1
character_set_server=utf8mb4
collation-server=utf8mb4_unicode_ci
skip-character-set-client-handshake
init_connect='SET collation_connection = utf8mb4_unicode_ci'
init_connect='SET NAMES utf8mb4'
skip_name_resolve = 1
max_connections = 800
max_connect_errors = 1000
datadir = /opt/local/mysql/data
pid-file = /opt/local/mysql/mysql.pid
transaction_isolation = READ-COMMITTED
explicit_defaults_for_timestamp = 1
join_buffer_size = 134217728
tmp_table_size = 67108864
tmpdir = /tmp
max_allowed_packet = 16777216
sql_mode = "STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER"
interactive_timeout = 1800
wait_timeout = 1800
read_buffer_size = 16777216
read_rnd_buffer_size = 33554432
sort_buffer_size = 33554432
########log settings########
log_error = /opt/local/mysql/logs/mysqld.log
slow_query_log = 1
slow_query_log_file = /opt/local/mysql/logs/slow.log
log_queries_not_using_indexes = 1
log_slow_admin_statements = 1
log_slow_slave_statements = 1
log_throttle_queries_not_using_indexes = 10
expire_logs_days = 90
long_query_time = 2
min_examined_row_limit = 100
########replication settings########
master_info_repository = TABLE
relay_log_info_repository = TABLE
log_bin = /opt/local/mysql/binlog/mysql-bin
sync_binlog = 1
gtid_mode = on
enforce_gtid_consistency = 1
log_slave_updates
binlog_format = row
relay_log = /opt/local/mysql/relaylog/relay-bin
relay_log_recovery = 1
binlog_gtid_simple_recovery = 1
slave_skip_errors = ddl_exist_errors
########innodb settings########
innodb_page_size = 16384
innodb_buffer_pool_size = 8G
innodb_buffer_pool_instances = 8
innodb_buffer_pool_load_at_startup = 1
innodb_buffer_pool_dump_at_shutdown = 1
innodb_lru_scan_depth = 2000
innodb_lock_wait_timeout = 5
innodb_io_capacity = 4000
innodb_io_capacity_max = 8000
innodb_flush_method = O_DIRECT
innodb_file_format = Barracuda
innodb_file_format_max = Barracuda
innodb_log_group_home_dir = /opt/local/mysql/relaylog
innodb_undo_directory = /opt/local/mysql/binlog
innodb_undo_logs = 128
innodb_undo_tablespaces = 3
innodb_flush_neighbors = 1
innodb_log_file_size = 4G
innodb_log_buffer_size = 16777216
innodb_purge_threads = 4
innodb_large_prefix = 1
innodb_thread_concurrency = 64
innodb_print_all_deadlocks = 1
innodb_strict_mode = 1
innodb_sort_buffer_size = 67108864
############mysql 5.7 ##################
innodb_buffer_pool_dump_pct = 40
innodb_page_cleaners = 4
innodb_undo_log_truncate = 1
innodb_max_undo_log_size = 2G
innodb_purge_rseg_truncate_frequency = 128
binlog_gtid_simple_recovery=1
log_timestamps=system
transaction_write_set_extraction=MURMUR32
show_compatibility_56=on
########semi sync replication settings########
#plugin_dir=/opt/local/mysql/lib/plugin
#plugin_load = "rpl_semi_sync_master=semisync_master.so;rpl_semi_sync_slave=semisync_slave.so"
#loose_rpl_semi_sync_master_enabled = 1
#loose_rpl_semi_sync_slave_enabled = 1
#loose_rpl_semi_sync_master_timeout = 5000



rm -rf /etc/my.cnf

cd /opt/local/mysql/bin/

/opt/local/mysql/bin/mysqld --initialize --user=mysql --basedir=/opt/local/mysql --datadir=/opt/local/mysql/data

# 查看 mysql 密码

cat /opt/local/mysql/logs/mysqld.log |grep password


# 启动 mysql

service mysqld start

chkconfig mysqld on

# 设置安全配置

/opt/local/mysql/bin/mysql_secure_installation -uroot -p


# 登陆 mysql

mysql -uroot -p

```


## 安装 redis

```
# 下载 redis

wget http://download.redis.io/releases/redis-3.2.10.tar.gz

tar zxvf redis-3.2.10.tar.gz 

cd redis-3.2.10

make

make install

cd  utils

./install_server.sh

...... 输入相关信息 ......

cd /etc/init.d/

mv redis_6379  redis


# 创建 目录

mkdir -p /opt/local/redis/{data,logs,conf}

cd /opt/local/redis/conf

vi redis.conf


bind 127.0.0.1
protected-mode yes
port 6379
tcp-backlog 2048
timeout 0
tcp-keepalive 300
daemonize yes
supervised no
pidfile /var/run/redis.pid
loglevel notice
logfile "/opt/local/redis/logs/redis.log"
maxmemory 10gb
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename redis_dump.rdb
dir /opt/local/redis/data
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



# 启动 redis

chkconfig redis on

service redis start

```



# 配置 Open-Falcon


## 初始化环境

```
mkdir -p $GOPATH/src/github.com/open-falcon
cd $GOPATH/src/github.com/open-falcon
git clone https://github.com/open-falcon/falcon-plus.git

```


## 导入数据库

```
cd $GOPATH/src/github.com/open-falcon/falcon-plus/scripts/mysql/db_schema/
mysql -u root -p < 1_uic-db-schema.sql
mysql -u root -p < 2_portal-db-schema.sql
mysql -u root -p < 3_dashboard-db-schema.sql
mysql -u root -p < 4_graph-db-schema.sql
mysql -u root -p < 5_alarms-db-schema.sql

```

## 编译 程序

```
cd $GOPATH/src/github.com/open-falcon/falcon-plus/

# 编译所有的模块
make all


# 编译指定模块
make agent


# 打包
make pack


# 创建目录

mkdir /opt/local/open-falcon

mv open-falcon-v0.2.1.tar.gz /opt/local/open-falcon

cd /opt/local/open-falcon

tar zxvf open-falcon-v0.2.1.tar.gz 

[root@localhost open-falcon]# ls
agent  aggregator  alarm  api  gateway  graph  hbs  judge  nodata  open-falcon  plugins  public  transfer

```


## 配置 Transfer


> transfer是数据转发服务。它接收agent上报的数据，然后按照哈希规则进行数据分片、并将分片后的数据分别push给graph&judge等组件。


```
cd /opt/local/open-falcon/transfer/config

vi cfg.json

{
    "debug": true,
    "minStep": 30,
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6060"
    },
    "rpc": {
        "enabled": true,
        "listen": "0.0.0.0:8433"
    },
    "socket": {
        "enabled": false,
        "listen": "0.0.0.0:4444",
        "timeout": 3600
    },
    "judge": {
        "enabled": true,
        "batch": 200,
        "connTimeout": 1000,
        "callTimeout": 5000,
        "maxConns": 32,
        "maxIdle": 32,
        "replicas": 500,
        "cluster": {
            "judge-00" : "127.0.0.1:6080"
        }
    },
    "graph": {
        "enabled": true,
        "batch": 200,
        "connTimeout": 1000,
        "callTimeout": 5000,
        "maxConns": 32,
        "maxIdle": 32,
        "replicas": 500,
        "cluster": {
            "graph-00" : "127.0.0.1:6070"
        }
    },
    "tsdb": {
        "enabled": false,
        "batch": 200,
        "connTimeout": 1000,
        "callTimeout": 5000,
        "maxConns": 32,
        "maxIdle": 32,
        "retry": 3,
        "address": "127.0.0.1:8088"
    }
}


```


## 配置 Graph

> graph是存储绘图数据的组件。graph组件 接收transfer组件推送上来的监控数据，同时处理api组件的查询请求、返回绘图数据。

```
# 创建数据 目录

mkdir -p /opt/data/6070


cd /opt/local/open-falcon/graph/config

vi cfg.json

{
    "debug": false,
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6071"
    },
    "rpc": {
        "enabled": true,
        "listen": "0.0.0.0:6070"
    },
    "rrd": {
        "storage": "/opt/data/6070"
    },
    "db": {
        "dsn": "root:123456@tcp(127.0.0.1:3306)/graph?loc=Local&parseTime=true",
        "maxIdle": 4
    },
    "callTimeout": 5000,
    "migrate": {
            "enabled": false,
            "concurrency": 2,
            "replicas": 500,
            "cluster": {
                    "graph-00" : "127.0.0.1:6070"
            }
    }
}


```



## 配置 Api 组件

> api组件，提供统一的restAPI操作接口。比如：api组件接收查询请求，根据一致性哈希算法去相应的graph实例查询不同metric的数据，然后汇总拿到的数据，最后统一返回给用户。

```
cd /opt/local/open-falcon/api/config

vi cfg.json 
# 主要修改 "salt": "" 为加密字串

{
        "log_level": "debug",
        "db": {
                "falcon_portal": "root:123456@tcp(127.0.0.1:3306)/falcon_portal?charset=utf8&parseTime=True&loc=Local",
                "graph": "root:123456@tcp(127.0.0.1:3306)/graph?charset=utf8&parseTime=True&loc=Local",
                "uic": "root:123456@tcp(127.0.0.1:3306)/uic?charset=utf8&parseTime=True&loc=Local",
                "dashboard": "root:123456@tcp(127.0.0.1:3306)/dashboard?charset=utf8&parseTime=True&loc=Local",
                "alarms": "root:123456@tcp(127.0.0.1:3306)/alarms?charset=utf8&parseTime=True&loc=Local",
                "db_bug": true
        },
        "graphs": {
                "cluster": {
                        "graph-00": "127.0.0.1:6070"
                },
                "max_conns": 100,
                "max_idle": 100,
                "conn_timeout": 1000,
                "call_timeout": 5000,
                "numberOfReplicas": 500
        },
        "metric_list_file": "./api/data/metric",
        "web_port": "0.0.0.0:8080",
        "access_control": true,
        "signup_disable": false,
        "salt": "pleaseinputwhichyouareusingnow",
        "skip_auth": false,
        "default_token": "default-token-used-in-server-side",
        "gen_doc": false,
        "gen_doc_path": "doc/module.html"
}


```


## 部署 Heartbeat 服务

> 心跳服务器，所有agent都会连到HBS，每分钟发一次心跳请求。

```
cd /opt/local/open-falcon/hbs/config

vi cfg.json


{
    "debug": true,
    "database": "root:123456@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true",
    "hosts": "",
    "maxConns": 20,
    "maxIdle": 15,
    "listen": ":6030",
    "trustable": [""],
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6031"
    }
}


```



## 部署 Judge 服务

> Judge用于告警判断，agent将数据push给Transfer，Transfer不但会转发给Graph组件来绘图，还会转发给Judge用于判断是否触发告警。

```
cd /opt/local/open-falcon/judge/config

vi cfg.json

{
    "debug": true,
    "debugHost": "nil",
    "remain": 11,
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6081"
    },
    "rpc": {
        "enabled": true,
        "listen": "0.0.0.0:6080"
    },
    "hbs": {
        "servers": ["127.0.0.1:6030"],
        "timeout": 300,
        "interval": 60
    },
    "alarm": {
        "enabled": true,
        "minInterval": 300,
        "queuePattern": "event:p%v",
        "redis": {
            "dsn": "127.0.0.1:6379",
            "maxIdle": 5,
            "connTimeout": 5000,
            "readTimeout": 5000,
            "writeTimeout": 5000
        }
    }
}

```

## 部署一个 mail 服务

>  发送警告邮件需要部署一个 mail 服务  用于发送邮件, 这里边部署一个简单的
>  mail-provider 地址 https://github.com/zzlyzq/mail-provider

```
# 下载 依赖模块

cd /opt/local/golang/src/github.com/open-falcon

git clone https://github.com/zzlyzq/mail-provider

cd /opt/local/golang/src/github.com/open-falcon/mail-provider

# 下载依赖
go get 

./control build (编译)

./control pack (打包)

mkdir /opt/local/open-falcon/mail-provider

mv falcon-mail-provider-0.0.1.tar.gz /opt/local/open-falcon/mail-provider

cd /opt/local/open-falcon/mail-provider

tar zxvf falcon-mail-provider-0.0.1.tar.gz


# 修改配置文件 里的 smtp 为自己的地址
# QQ邮箱，请开启 smtp 的功能，在QQ邮箱后台开启

vi cfg.json

{
    "debug": true,
    "http": {
        "listen": "0.0.0.0:4000",
        "token": ""
    },
    "smtp": {
        "addr": "smtp.qq.com:587",
        "username": "jicki@qq.com",
        "password": "123456",
        "from": "jicki@qq.com"
    }
}



# 运行程序

./control start

# 查看日志
./control tail



# 测试, 在测试时 token 暂时设置为空


curl http://127.0.0.1:4000/sender/mail -d "tos=jicki@qq.com&subject=xx&content=yy"

```


## 部署一个 微信网关


> 微信网关 git  https://github.com/Yanjunhui/chat

```
cd /opt/local/open-falcon

git clone https://github.com/Yanjunhui/chat

cd chat/

chmod +x control.sh

# 需要修改 配置文件

cat config.conf


#http 服务端口
[http]
port = 4567

#微信接口信息
[weixin]
CorpID = ww6424d33203e90e20
AgentId = 1000002
Secret = FoST_8RQSTjZwH_CN3aQW6UKksjCSI9mizFqD7HKhrw
EncodingAESKey = K2M3WMhRHIOH4I1Ww5jxpllGrgY01nvBjUgTvcJEEHX


# 启动
./control.sh start
./control.sh status



## 注意: 

要收到 im 报警信息，必须要在 个人用户里面 填写 微信相关资料

微信相关帐号是  登陆微信公众号 --> 通讯里， 里面用户的 帐号

不是个人微信帐号，填写个人帐号，是收不到报警的。

```


## 部署 Alarm 服务

> alarm模块是处理报警event的，judge产生的报警event写入redis，alarm从redis读取处理，并进行不同渠道的发送。

```
cd /opt/local/open-falcon/alarm/config

vi cfg.json


{
    "log_level": "debug",
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:9912"
    },
    "redis": {
        "addr": "127.0.0.1:6379",
        "maxIdle": 5,
        "highQueues": [
            "event:p0",
            "event:p1",
            "event:p2"
        ],
        "lowQueues": [
            "event:p3",
            "event:p4",
            "event:p5",
            "event:p6"
        ],
        "userIMQueue": "/queue/user/im",
        "userSmsQueue": "/queue/user/sms",
        "userMailQueue": "/queue/user/mail"
    },
    "api": {
        "im": "http://127.0.0.1:4567/send",
        "sms": "http://127.0.0.1:10086/sms",
        "mail": "http://127.0.0.1:4000/sender/mail",
        "dashboard": "http://127.0.0.1:8081",
        "plus_api":"http://127.0.0.1:8080",
        "plus_api_token": "default-token-used-in-server-side"
    },
    "falcon_portal": {
        "addr": "root:123456@tcp(127.0.0.1:3306)/alarms?charset=utf8&loc=Asia%2FChongqing",
        "idle": 10,
        "max": 100
    },
    "worker": {
        "im": 10,
        "sms": 10,
        "mail": 50
    },
    "housekeeper": {
        "event_retention_days": 7,
        "event_delete_batch": 100
    }
}

```


## 配置 Nodata 服务

> nodata用于检测监控数据的上报异常。nodata和实时报警judge模块协同工作，过程为: 配置了nodata的采集项超时未上报数据，nodata生成一条默认的模拟数据；用户配置相应的报警策略，收到mock数据就产生报警。采集项上报异常检测，作为judge模块的一个必要补充，能够使judge的实时报警功能更加可靠、完善。


```
cd /opt/local/open-falcon/nodata/config

vi cfg.json 

{
    "debug": true,
    "http": {
        "enabled": true,
        "listen": "0.0.0.0:6090"
    },
    "plus_api":{
        "connectTimeout": 500,
        "requestTimeout": 2000,
        "addr": "http://127.0.0.1:8080",
        "token": "default-token-used-in-server-side"
    },
    "config": {
        "enabled": true,
        "dsn": "root:123456@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true&wait_timeout=604800",
        "maxIdle": 4
    },
    "collector":{
        "enabled": true,
        "batch": 200,
        "concurrent": 10
    },
    "sender":{
        "enabled": true,
        "connectTimeout": 500,
        "requestTimeout": 2000,
        "transferAddr": "127.0.0.1:6060",
        "batch": 500
    }
}


```


## 配置 Aggregator 服务

> 集群聚合模块。聚合某集群下的所有机器的某个指标的值，提供一种集群视角的监控体验。


```
cd /opt/local/open-falcon/aggregator/config

vi cfg.json

{
    "debug": true,
    "http": {
        "enabled": false,
        "listen": "0.0.0.0:6055"
    },
    "database": {
        "addr": "root:123456@tcp(127.0.0.1:3306)/falcon_portal?loc=Local&parseTime=true",
        "idle": 10,
        "ids": [1, -1],
        "interval": 55
    },
    "api": {
        "connect_timeout": 500,
        "request_timeout": 2000,
        "plus_api": "http://127.0.0.1:8080",
        "plus_api_token": "default-token-used-in-server-side",
        "push_api": "http://127.0.0.1:1988/v1/push"
    }
}


```


## 启动所有服务

```
cd /opt/local/open-falcon

./open-falcon start

./open-falcon check
        falcon-graph         UP           71646 
          falcon-hbs         UP           71658 
        falcon-judge         UP           71670 
     falcon-transfer         UP           71678 
       falcon-nodata         UP           71686 
   falcon-aggregator         UP           71695 
        falcon-agent         UP           71706 
      falcon-gateway         UP           71715 
          falcon-api         UP           71724 
        falcon-alarm         UP           71738 

```




# 配置 Agent 

> agent用于采集机器负载监控指标，比如cpu.idle、load.1min、disk.io.util等等，每隔60秒push给Transfer。agent与Transfer建立了长连接，数据发送速度比较快，agent提供了一个http接口/v1/push用于接收用户手工push的一些数据，然后通过长连接迅速转发给Transfer。

```

cd /opt/local/open-falcon/agent/config

# Agent 配置文件


{
    "debug": true,
    "hostname": "",
    "ip": "",
    "plugin": {
        "enabled": false,
        "dir": "./plugin",
        "git": "https://github.com/open-falcon/plugin.git",
        "logs": "./logs"
    },
    "heartbeat": {
        "enabled": true,
        "addr": "127.0.0.1:6030",
        "interval": 60,
        "timeout": 1000
    },
    "transfer": {
        "enabled": true,
        "addrs": [
            "127.0.0.1:8433"
        ],
        "interval": 60,
        "timeout": 1000
    },
    "http": {
        "enabled": false,
        "listen": ":1988",
        "backdoor": false
    },
    "collector": {
        "ifacePrefix": ["eth", "em"],
        "mountPoint": []
    },
    "default_tags": {
    },
    "ignore": {
        "cpu.busy": true,
        "df.bytes.free": true,
        "df.bytes.total": true,
        "df.bytes.used": true,
        "df.bytes.used.percent": true,
        "df.inodes.total": true,
        "df.inodes.free": true,
        "df.inodes.used": true,
        "df.inodes.used.percent": true,
        "mem.memtotal": true,
        "mem.memused": true,
        "mem.memused.percent": true,
        "mem.memfree": true,
        "mem.swaptotal": true,
        "mem.swapused": true,
        "mem.swapfree": true
    }
}




```


```
# agent 脚本  添加 hostname = IP

#!/bin/bash
mkdir /opt/local
cd /opt/local
wget http://172.16.1.100/agent.tar.gz
tar zxvf agent.tar.gz
rm -rf agent.tar.gz
IPADDR=`ifconfig em1|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"`
sed -i 's/\"hostname\"\:.*$/\"hostname\"\: \"'$IPADDR'\"\,/g' open-falcon/agent/config/cfg.json
cat open-falcon/agent/config/cfg.json
cd open-falcon
./open-falcon start agent
./open-falcon check agent

```






# 配置前端 dashboard


## 初始化依赖

```
yum install -y python-virtualenv
yum install -y python-devel
yum install -y openldap-devel
yum install -y mysql-devel
yum groupinstall "Development tools"

```



## 安装配置 dashboard

```
cd /opt/local/open-falcon

git clone https://github.com/open-falcon/dashboard.git

cd dashboard

# 创建 python 独立运行环境

virtualenv ./env

# 安装 python 依赖模块

# 执行安装

./env/bin/pip install -r pip_requirements.txt -i http://mirrors.aliyun.com/pypi/simple/



# 修改配置

# 修改里面的 mysql 配置

rrd/config.py

ALARM_DB_PASS

```


## 启动 dashboard

```
# debug 模式

./env/bin/python wsgi.py


# 正常模式

bash control start


# 查看日志
bash control tail 

```


## 登陆 WEB UI

```

http://172.16.1.100:8081/


注册  root 帐号  为 admin 帐号

```




# 监控报警


## 配置 报警名单

```
# 首先配置 用户组

dashboard --> Welcome root -->  Teams 

Add+  -- > Create Team

# 创建一个 ICT 组

名称： ICT
简介： 运维组
成员： root

```


## 配置 nodata 监控 agent

```
# 创建 HostGrop

dashboard --> HostGroups

添加一个名称 dev-server 的 HostGroups

点击 hosts -- > Add Host  添加  dev 相关服务器 

```


```
# 创建 nodata
# 监控 client 的 agent ，如果 agent 掉了， 那么无法上传数据，所以直接配置
模板是不行的， 必须配置 nodata 在 agent 抓不到数据的时候 值为 -1 . 

dashboard --> nodata

Add nodata

name: nodata.agent

endpoint选择:   机器分组  ---  dev-server

metric: agent.alive

type: GAUGE

周期: 60

数据上报中断时，补发如下值: -1


```




```
# 创建 策略配置

dashboard --> Templates

创建一个 名称为 agent-alive 的模板


# 模板基础信息:

name: agent-alive

模板策略列表:

metric: agent.alive   note: 无法连接agent

if [all(#3)] < 0 : alarm(); callback();

save 保存

# 模板报警配置:

之前设置的 ICT



# Save
```


```
# 绑定 模板

dashboard --> HostGroups

查找  dev-server   点击  templates

查找  agent-alive   选择 + Bind

```


```
# 测试

关闭一个 agent 的进程


等待60秒 查看 Alarm-Dashboard


等待收取邮件~


```


## 配置 系统监控指标

```
dashboard --> Templates

Add 添加一个 系统指标模块

监控指标如下:
```

![sys.png-60.9kB][1]


# FAQ 

## 修改 报警模板

```
cd open-falcon/falcon-plus/modules/alarm/cron


# 报警内容:

cat builder.go



package cron

import (
        "fmt"

        "github.com/open-falcon/falcon-plus/common/model"
        "github.com/open-falcon/falcon-plus/common/utils"
        "github.com/open-falcon/falcon-plus/modules/alarm/g"
)

func BuildCommonSMSContent(event *model.Event) string {
        return fmt.Sprintf(
                "[P%d][%s][%s][][%s %s %s %s %s%s%s][O%d %s]",
                event.Priority(),
                event.Status,
                event.Endpoint,
                event.Note(),
                event.Func(),
                event.Metric(),
                utils.SortedTags(event.PushedTags),
                utils.ReadableFloat(event.LeftValue),
                event.Operator(),
                utils.ReadableFloat(event.RightValue()),
                event.CurrentStep,
                event.FormattedTime(),
        )
}

func BuildCommonIMContent(event *model.Event) string {
        return fmt.Sprintf(
                "[报警级别: %d][报警状态: %s][报警Host: %s][报警内容: %s][报警时间: %s]",
                event.Priority(),
                event.Status,
                event.Endpoint,
                event.Note(),
                //event.Func(),
                //event.Metric(),
                //utils.SortedTags(event.PushedTags),
                //utils.ReadableFloat(event.LeftValue),
                //event.Operator(),
                //utils.ReadableFloat(event.RightValue()),
                //event.CurrentStep,
                event.FormattedTime(),
        )
}

func BuildCommonMailContent(event *model.Event) string {
        link := g.Link(event)
        return fmt.Sprintf(
                "报警状态: %s\r\n报警级别: %d\r\n报警Host: %s\r\n报警事件: %s\r\n事件标签: %s\r\n报警表达式: %s: %s%s%s\r\n报警内容: %s\r\n最大报警次数: %d   当前报警次数: %d\r\n报警时间: %s\r\n报警模板: %s\r\n",
                event.Status,
                event.Priority(),
                event.Endpoint,
                event.Metric(),
                utils.SortedTags(event.PushedTags),
                event.Func(),
                utils.ReadableFloat(event.LeftValue),
                event.Operator(),
                utils.ReadableFloat(event.RightValue()),
                event.Note(),
                event.MaxStep(),
                event.CurrentStep,
                event.FormattedTime(),
                link,
        )
}

func GenerateSmsContent(event *model.Event) string {
        return BuildCommonSMSContent(event)
}

func GenerateMailContent(event *model.Event) string {
        return BuildCommonMailContent(event)
}

func GenerateIMContent(event *model.Event) string {
        return BuildCommonIMContent(event)
}


```

  [1]: https://jicki.cn/img/posts/openfalcon/sys.png

