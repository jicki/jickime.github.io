# Redis twemproxy 集群



#Redis twemproxy 集群

## 环境说明

```
4台 redis 服务器
172.16.1.37:6379   - 1
172.16.1.36:6379   - 2
172.16.1.35:6379   - 3
172.16.1.34:6379   - 4
```
 

## 安装依赖

安装 autoconf 
centos 7 yum 安装既可， autoconf 版本必须 2.64以上版本

```
yum -y install autoconf
```
 
## 安装 twemproxy 

```
git clone https://github.com/twitter/twemproxy.git

autoreconf -fvi          #生成configure文件

./configure --prefix=/opt/local/twemproxy/ --enable-debug=log

make && make install

mkdir -p /opt/local/twemproxy/{run,conf,logs}

ln -s /opt/local/twemproxy/sbin/nutcracker /usr/bin/
```



## 配置 twemproxy


cd /opt/local/twemproxy/conf/


vi nutcracker.yml          #编辑配置文件

```
alpha:

  listen: 172.16.1.37:6380                        #监听端口

  hash: fnv1a_64                                  #key值hash算法，默认fnv1a_64

  distribution: ketama                            #分布算法      

  #ketama一致性hash算法；modula非常简单，就是根据key值的hash值取模；random随机分布

  auto_eject_hosts: true                          #摘除后端故障节点   

  redis: true                                     #是否是redis缓存，默认是false

  timeout: 400                                    #代理与后端超时时间，毫秒

  server_retry_timeout: 200000                    #摘除故障节点后重新连接的时间，毫秒

  server_failure_limit: 1                         #故障多少次摘除

  servers:

   - 172.16.1.34:6379:1 

   - 172.16.1.35:6379:1 

   - 172.16.1.36:6379:1 

   - 172.16.1.37:6379:1

```

检查配置文件是否正确 

```
nutcracker -t -c /opt/local/twemproxy/conf/nutcracker.yml       
```
 
## 启动twemproxy

```
nutcracker -d -c /opt/local/twemproxy/conf/nutcracker.yml -p /opt/local/twemproxy/run/redisproxy.pid -o /opt/local/twemproxy/logs/redisproxy.log
```

