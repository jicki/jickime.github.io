# docker swarm 集群



# Docker Swarm 集群

## 环境说明

```
IP 10.6.17.11  管理节点

IP 10.6.17.12   节点A

IP 10.6.17.13   节点B

IP 10.6.17.14   节点C
```


## 安装 zookeeper

安装过程略

```
zk://10.6.17.11:2181
```
 

## 修改docker启动参数

修改 -H 添加 tcp 端口

```
sed -i 's/-H fd:\/\//-H tcp:\/\/10.6.17.11:2375 --cluster-store=zk:\/\/10.6.17.11:2181\/store --cluster-advertise=10.6.17.11:2375/g' /lib/systemd/system/docker.service
```
 
## 重启docker

```
systemctl daemon-reload

systemctl restart docker
```
 

## 创建 master 节点

```
docker -H tcp://10.6.17.11:2375 run --name master --restart=always -d -p 8888:2375 swarm manage zk://10.6.17.11:2181/swarm
```



## 添加node节点到集群


A:

```
docker -H tcp://10.6.17.12:2375 run --name node_1 --restart=always -d swarm join --addr=10.6.17.12:2375 zk://10.6.17.11/swarm
```
 

B:

```
docker -H tcp://10.6.17.13:2375 run --name node_2 --restart=always -d swarm join --addr=10.6.17.13:2375 zk://10.6.17.11/swarm
```


C:

```
docker -H tcp://10.6.17.14:2375 run --name node_3 --restart=always -d swarm join --addr=10.6.17.14:2375 zk://10.6.17.11/swarm
```
 


## 列出节点信息

```
docker -H tcp://10.6.17.11:8888 run --rm swarm list zk://10.6.17.11:2181/swarm

10.6.17.12:2375

10.6.17.13:2375

10.6.17.14:2375
```
 

## 管理集群

```
[root@localhost docker]# docker -H tcp://10.6.17.11:8888 ps -a

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS               NAMES

3629e76de67c        swarm               "/swarm join --addr=1"   4 minutes ago       Up 4 minutes        2375/tcp            localhost.localdomain/node_1

325b71855b86        swarm               "/swarm join --addr=1"   6 minutes ago       Up About a minute   2375/tcp            localhost.localdomain/node_3

b888bbbfe594        swarm               "/swarm join --addr=1"   6 minutes ago       Up 6 minutes        2375/tcp            localhost.localdomain/node_2
```



# FAQ

```
# swarm 集群中，出现加入 network 的错误，于某个 node 中。

Cannot start container: subnet sandbox join failed for "10.0.0.0/24": error creating vxlan interface: file exists

network sandbox join failed: error creating vxlan interface: file exists

# 参考解决方案

umount /var/run/docker/netns/*
rm -rf /var/run/docker/netns/*

# 重启 docker
systemctl restart docker

#如果还是不行 重启 这台 node 服务器


```

