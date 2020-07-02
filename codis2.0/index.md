# Codis-2.0 版本 编译安装




# codis 2.0 版本编译安装


## 安装相关依赖

```
yum install -y git
```

安装 go 语言, 请务必安装1.7.x 版本

```
wget https://storage.googleapis.com/golang/go1.7.5.linux-amd64.tar.gz
tar zxvf go1.7.5.linux-amd64.tar.gz
mv go /opt/local/
```

配置环境变量

```
vi /etc/profile
```

增加

```
export GOROOT=/opt/local/go
export PATH=$PATH:$GOROOT/bin
export GOPATH=/opt/local/codis
```

让配置生效

```
source /etc/profile
```
 
检查

```
go version
```
 

## 安装codis

```
go get -u -d github.com/CodisLabs/codis

cd /opt/local/codis/src/github.com/CodisLabs/codis

make

make -j -C extern/redis-2.8.21/

```


Hint: It's a good idea to run 'make test' ;)    表示安装完成


完成安装会在 

```
codis/src/github.com/CodisLabs/codis/bin 
```

生成  四个文件

```
assets
codis-config
codis-proxy
codis-server   
```
 

 复制文件 方便管理

```
mkdir -p /opt/local/codis/{logs,config,data}/
cd /opt/local/codis/src/github.com/CodisLabs/codis/bin
cp -rf * /opt/local/codis/bin
cd /opt/local/codis/src/github.com/CodisLabs/codis/
cp config.ini /opt/local/codis/config/
cd /opt/local/codis/src/github.com/CodisLabs/codis/extern/redis-2.8.21/src
cp redis-cli /opt/local/codis/bin/redis-cli-2.8.21
ln -s /opt/local/codis/bin/redis-cli-2.8.21 /opt/local/codis/redis-cli

```
 

## 配置 codis 

修改配置文件

```
cd /opt/local/codis/config

vi config.ini
```
 
修改IP，配置WEB管理 默认 18087 端口  多proxy 配置VIP地址，但只能启动一个dashboard ,程序掉线再启动其他

```
dashboard_addr
```

( zk  配置 zookeeper 地址，单机配置一个，群集配置多个~以，号隔开 ）

```
proxy_id=codis_proxy_1   # 配置为唯一
```
 
```
product=     # zookeeper 节点信息
```
 


redis-server 配置文件

```
cd /opt/local/codis/src/github.com/CodisLabs/codis/extern/redis-test/config

cp 6379.conf  6380.conf /opt/local/codis/config/
```
 
 
修改里面的配置端口与配置 需要修改的配置如下：

```
daemonize yes      #后台模式运行

pidfile  var/run/redis_6379.pid   #pid 文件

port 6379          #运行端口

timeout 50         #请求超时时间,默认0

logfile "/opt/local/codis/logs/codis_6379.log"   #日志文件

maxmemory 20gb     #最大内存设置

save 900 1      #打开保存快照的条件( 第一个*表示多长时间 , 第三个*表示执行多少次写操作 ) 

save 300 10

save 60 10000

dbfilename 6379.rdb                     #数据快照保存的名字

dir /opt/local/codis/data        #数据快照的保存目录

appendfilename "6379_appendonly.aof"    #Redis更加高效的数据库备份及灾难恢复方式。

appendfsync everysec         # ( always: always表示每次有写操作都进行同步. everysec: 表示对写操作进行累积，每秒同步一次 )
```
 

## dashboard 设置

创建 dashboard 启动脚本

```
vi /opt/local/codis/start_dashboard.sh
```

```
#!/bin/sh

nohup /opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini -L /opt/local/codis/logs/dashboard.log dashboard --addr=:18087 --http-log=/opt/local/codis/logs/requests.log &>/dev/null &
```
 
启动 dashboard 

```
/opt/local/codis/start_dashboard.sh
```
 


初始化 slot , 该命令会在zookeeper上创建slot相关信息

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini slot init
```
 

## codis-server 设置

启动codis-server服务

```
/opt/local/codis/bin/codis-server /opt/local/codis/config/6379.conf
/opt/local/codis/bin/codis-server /opt/local/codis/config/6380.conf
```
 
添加 Redis Server Group

( 每一个 Server Group 作为一个 Redis 服务器组存在, 只允许有一个 master, 可以有多个 slave, group id 仅支持大于等于1的整数 )


第一组

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 1 172.16.32.78:6379 master

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 1 172.16.32.79:6380 slave
```
 
第二组

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 2 172.16.32.79:6379 master

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 2 172.16.32.78:6380 slave
```
 
第三组

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 3 172.16.32.80:6379 master

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 3 172.16.32.81:6380 slave
```
 
第四组

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 4 172.16.32.81:6379 master

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini server add 4 172.16.32.80:6380 slave
```
 

设置 server group 服务的 slot 范围
 
```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini slot range-set 0 255 1 online

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini slot range-set 256 511 2 online

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini slot range-set 512 767 3 online

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini slot range-set 768 1023 4 online
```
 
 
 
slot 数据迁移

Codis 支持动态的根据实例内存, 自动对slot进行迁移, 以均衡数据分布.

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini slot rebalance
```
 
执行此命令需要满足下列条件：

 1. 所有的codis-server都必须设置了maxmemory参数
 2. 所有的 slots 都应该处于 online 状态, 即没有迁移任务正在执行
 3. 所有 server group 都必须有 Master


另外 codis 还支持比例迁移。

如: 将slot id 为 [0-511] 的slot的数据, 迁移到 server group 2 上, --delay 参数表示每迁移一个 key 后 sleep 的毫秒数, 默认是 0, 用于限速.

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini slot migrate 0 511 2 --delay=10
```
 
迁移的过程对于上层业务来说是安全且透明的, 数据不会丢失, 上层不会中止服务.


注意：迁移的过程中打断是可以的, 但是如果中断了一个正在迁移某个slot的任务, 下次需要先迁移掉正处于迁移状态的 slot, 否则无法继续 (即迁移程序会检查同一时刻只能有一个 slot 处于迁移状态).

 

 
## codis-proxy 设置

创建 start_proxy.sh，启动codis-proxy服务

```
vi /opt/local/codis/start_proxy.sh
```

```
#!/bin/sh
echo "shut down codis_proxy_1.."
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini proxy offline codis_proxy_1
echo "done"
echo "start new proxy..."
nohup /opt/local/codis/bin/codis-proxy --log-level info -c /opt/local/codis/config/config.ini -L /opt/local/codis/logs/proxy.log  --cpu=4 --addr=0.0.0.0:19000 --http-addr=0.0.0.0:11000 &>/dev/null &
echo "done"
```


启动 proxy

```
/opt/local/codis/start_proxy.sh
```
 
设置 proxy 为 online 状态

```
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini proxy online codis_proxy_1

/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini proxy online codis_proxy_2
 
/opt/local/codis/bin/codis-config -c /opt/local/codis/config/config.ini proxy online codis_proxy_3
```
 


## 访问dashboard

```
http://localhost:18087/admin/
```
 

 
## codis-server 主从切换配置

 


codis-ha 是一个通过 是一个通过 codis开放的 api 实现自动切换主从

配置如下：

```
go get github.com/ngaut/codis-ha

cd /opt/local/codis/src/github.com/ngaut/codis-ha

go build
```
 

启动：

```
cd /opt/local/codis/bin

./codis-ha --codis-config="127.0.0.1:18087" --productName="mycodis"

```

```
https://github.com/wlibo666/codis-ha
```

这个 codis-ha 添加了很多新功能，包括邮件报警，等功能。




