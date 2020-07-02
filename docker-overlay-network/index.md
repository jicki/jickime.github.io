# Docker overlay 网络




# Docker overlay 网络

## 说明


Overlay网络是指在不改变现有网络基础设施的前提下，通过某种约定通信协议，把二层报文封装在IP报文之上的新的数据格式。 这样不但能够充分利用成熟的IP路由协议进程数据分发，而且在Overlay技术中采用扩展的隔离标识位数，能够突破VLAN的4000数量限制， 支持高达16M的用户，并在必要时可将广播流量转化为组播流量，避免广播数据泛滥。 因此，Overlay网络实际上是目前最主流的容器跨节点数据传输和路由方案。 Overlay网络的实现方式可以有许多种，其中IETF（国际互联网工程任务组）制定了三种Overlay的实现标准 1. 虚拟可扩展LAN（VXLAN） 2. 采用通用路由封装的网络虚拟化（NVGRE） 3. 无状态传输协议（SST） Docker内置的Overlay网络是采用IETF标准的VXLAN方式，并且是VXLAN中普遍认为最适合大规模的云计算虚拟化环境的SDN Controller模式。 Docker的Overlay网络功能与其Swarm集群是紧密整合的，因此为了使用Docker的内置跨节点通信功能，最简单的方式就是采纳Swarm作为集群的解决方案。


## overlay 条件

在 docker 1.9 中，要使用 Swarm + overlay 网络架构，还需要以下几个条件：

1. 所有Swarm节点的Linux系统内核版本不低于3.16  (在 docker 1.10 后面版本中，已经支持内核3.10，升级内核实在是一个麻烦事情)

2. 需要一个额外的配置存储服务，例如Consul、Etcd或ZooKeeper

3. 所有的节点都能够正常连接到配置存储服务的IP和端口

4. 所有节点运行的Docker后台进程需要使用『--cluster-store』和『--cluster-advertise』参数指定所使用的配置存储服务地址


## 环境说明

```
服务器3台 如下：

10.6.17.12

10.6.17.13

10.6.17.14
```
 
```
docker version

Client:

 Version:      1.10.0-rc1

 API version:  1.22

 Go version:   go1.5.3

 Git commit:   677c593

 Built:        Fri Jan 15 20:50:15 2016

 OS/Arch:      linux/amd64
```



## 修改主机名

```
10.6.17.12  = hostnamectl --static set-hostname swarm-master

10.6.17.13  = hostnamectl --static set-hostname swarm-node-1

10.6.17.14  = hostnamectl --static set-hostname swarm-node-2
```



上面的4个条件中，第一个条件在docker 1.10 RC 版本中已经默认就满足了。


下面我们来创建第二个条件中的 配置存储服务，配置存储服务按照大家的使用习惯，自己选择一个配置存储。


由于我们java 项目一直在使用 ZooKeeper ，所以这边选择 ZooKeeper 作为存储服务，为了方便测试，这边只配置 单机的 ZooKeeper 服务

 
 
## 配置 swarm 集群

```
[10.6.17.12]# sed -i 's/-H fd:\/\//-H tcp:\/\/10.6.17.12:2375 --cluster-store=zk:\/\/10.6.17.12:2181/store --cluster-advertise=10.6.17.12:2376/g' /lib/systemd/system/docker.service

 
[10.6.17.13]# sed -i 's/-H fd:\/\//-H tcp:\/\/10.6.17.13:2375 --cluster-store=zk:\/\/10.6.17.12:2181/store --cluster-advertise=10.6.17.13:2376/g' /lib/systemd/system/docker.service


[10.6.17.14]# sed -i 's/-H fd:\/\//-H tcp:\/\/10.6.17.14:2375 --cluster-store=zk:\/\/10.6.17.12:2181/store --cluster-advertise=10.6.17.14:2376/g' /lib/systemd/system/docker.service
```
 

```
systemctl daemon-reload   

systemctl restart docker.service
```


首先我们选择 10.6.17.12 这台机器做为 master 节点 创建 swarm：

```
[10.6.17.12]# docker -H tcp://10.6.17.12:2375 run --name master --restart=always -d -p 8888:2375 swarm manage zk://10.6.17.12:2181/swarm
```
 

在其他两台Docker业务容器运行的节点上运行Swarm Agent服务：

```
[10.6.17.13]# docker -H tcp://10.6.17.13:2375 run --name node_1 --restart=always -d swarm join --addr=10.6.17.13:2375 zk://10.6.17.12:2181/swarm
```

```
[10.6.17.14]# docker -H tcp://10.6.17.14:2375 run --name node_2 --restart=always -d swarm join --addr=10.6.17.14:2375 zk://10.6.17.12:2181/swarm
```
 

查看所有节点上的信息：

```
[10.6.17.12]# docker -H tcp://10.6.17.12:8888 ps -a
 

CONTAINER ID        IMAGE               COMMAND                  CREATED                  STATUS                  PORTS               NAMES

5fc7753caa2c        swarm               "/swarm join --addr=1"   Less than a second ago   Up Less than a second   2375/tcp            swarm-node-1/node_1

330b964ba732        swarm               "/swarm join --addr=1"   Less than a second ago   Up Less than a second   2375/tcp            swarm-node-2/node_2
```

至此 swarm 集群已经搭建完成了。

 

## 创建 overlay 网络

Swarm提供与Docker服务完全兼容的API，因此可以直接使用docker命令进行操作。

注意上面命令中创建Master服务时指定的外部端口号8888，它就是用来连接Swarm服务的地址。

现在我们就可以创建一个Overlay类型的网络了：

```
[10.6.17.12]# docker -H tcp://10.6.17.12:8888 network create --driver=overlay ovr0
```
 

这个命令被发送给了Swarm服务，Swarm会在所有Agent节点上添加一个属性完全相同的Overlay类型网络。
 
在每个节点上面 使用 docker network ls  可以查看 到已经有一个  ovr0  的 overlay 网络

```
docker network ls
```


在Swarm的网络里面，每个网络的名字都会加上节点名称作为前缀， 


如： 
```
   swarm-node-1/node_1    

   swarm-node-2/node_2
```
 
但Overlay类型的网络是没有这个前缀的，这也说明了这类网络是被所有节点共有的。

下面我们在Swarm中创建两个连接到Overlay网络的容器，并用Swarm的过滤器限制这两个容器分别运行在不同的节点上。

## 创建基于overlay的容器

nginx dockerfile

``` 
FROM centos
MAINTAINER jicki@qq.com
RUN yum -y update; yum clean all
RUN yum -y install epel-release; yum clean all
RUN yum -y install wget; yum clean all
ADD ./nginx.sh /root/
RUN /bin/bash /root/nginx.sh
RUN rm -rf /root/nginx.sh
RUN rm -rf /opt/local/nginx/conf/nginx.conf
ADD ./nginx.conf /opt/local/nginx/conf/
RUN mkdir -p /opt/local/nginx/conf/vhost
ADD ./docker.conf /opt/local/nginx/conf/vhost
RUN chown -R upload:upload /opt/htdocs/web
EXPOSE 80 443
CMD ["/opt/local/nginx/sbin/nginx", "-g", "daemon off;"]
```


```
[10.6.17.12]# docker -H tcp://10.6.17.12:8888 run --name nginx_web_1 --net ovr0 --env="constraint:node==swarm-node-1" -d -v /opt/data/nginx/logs:/opt/local/nginx/logs nginx

[10.6.17.12]# docker -H tcp://10.6.17.12:8888 run --name nginx_web_2 --net ovr0 --env="constraint:node==swarm-node-2" -d -v /opt/data/nginx/logs:/opt/local/nginx/logs nginx
```
 


## 测试网络

创建完两个容器以后，下面来来测试一下 ovr0 这个网络的连通性

```
[10.6.17.12]# docker -H tcp://10.6.17.12:8888 exec -it nginx_web_1 ping nginx_web_2

PING nginx_web_2 (10.0.0.3) 56(84) bytes of data.

64 bytes from nginx_web_2.ovr0 (10.0.0.3): icmp_seq=1 ttl=64 time=0.360 ms

64 bytes from nginx_web_2.ovr0 (10.0.0.3): icmp_seq=2 ttl=64 time=0.247 ms

64 bytes from nginx_web_2.ovr0 (10.0.0.3): icmp_seq=3 ttl=64 time=0.234 ms

64 bytes from nginx_web_2.ovr0 (10.0.0.3): icmp_seq=4 ttl=64 time=0.241 ms

64 bytes from nginx_web_2.ovr0 (10.0.0.3): icmp_seq=5 ttl=64 time=0.212 ms
```

如上所示 我们已经在Docker的Overlay网络上成功的进行了跨节点的数据通信。


测试两个 ssh 的服务，创建两个 容器，查看容器所属 IP 。

```
[10.6.17.12]# docker -H tcp://10.6.17.12:8888 run --name ssh-1 --net ovr0 --env="constraint:node==swarm-node-1" -d -p 8001:22 ssh

 

[10.6.17.12]# docker -H tcp://10.6.17.12:8888 run --name ssh-2 --net ovr0 --env="constraint:node==swarm-node-2" -d -p 8001:22 ssh
```
 
创建容器 IP 为  DHCP 分配， 按照从下向上分配， 重启不会改变overlay 的IP 。

首先创建 ssh-1 分配IP为 10.0.0.4    创建 ssh-2 分配IP为 10.0.0.5

销毁 ssh-1 再次创建 分配IP 为 10.0.0.4 

销毁 ssh-1  ssh-2  先创建 ssh-2 分配 IP 为 10.0.0.4  

 

## alias 


在 docker 1.10 后面的版本中 --net-alias=[]  的使用！！

在docker run 的时候 可指定相同的 alias ，可以实现 故障切换的效果。。

具体命令如：

 
```
[10.6.17.12]# docker -H tcp://10.6.17.12:8888 run --name nginx_web_1 --net ovr0 --net-alias="nginx" --env="constraint:node==swarm-node-1" -d -v /opt/data/nginx/logs:/opt/local/nginx/logs nginx

[10.6.17.12]# docker -H tcp://10.6.17.12:8888 run --name nginx_web_2 --net ovr0 --net-alias="nginx" --env="constraint:node==swarm-node-2" -d -v /opt/data/nginx/logs:/opt/local/nginx/logs nginx
```
 
当我们进入 机器里面的时候 使用 dig 查看 nginx A记录 看到的是一个，但是 一个容器 挂掉以后

A记录会自动绑定到另外一台机器中。


在 docker 1.11 后面的版本中 --net-alias=[] 已经支持 负载均衡。

当我们使用 dig 查看 A记录 时可以看到多个 A记录

 
# network disconnect

docker network disconnect  与  docker network connect 命令的使用！ 
使用这两个命令可达到 A B 测试 以及 快速 回滚 的效果。

```
docker network connect      ---->  加入 指定网络

docker network disconnect   ---->  退出 指定网络
```
 

具体命令使用：

```
docker network disconnect ovr0 nginx_web_2       nginx_web_2 这个容器退出 ovr0 这个网络。

docker network connect ovr0 nginx_web_2          nginx_web_2 这个容器重新加入 ovr0 这个网络。
```

