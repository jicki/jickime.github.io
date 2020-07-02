# rancher docker 集群管理于编排



rancher docker 集群管理于编排

# 官方 github
https://github.com/rancher/rancher

# 环境说明

```
10.6.0.140
10.6.0.187
10.6.0.188
```
```
#修改主机名:
10.6.0.140 = hostnamectl --static set-hostname reancher-manager
10.6.0.187 = hostnamectl --static set-hostname reancher-node-1
10.6.0.188 = hostnamectl --static set-hostname reancher-node-2
```

#  安装 rancher

## 初始化 rancher

```
[root@reancher-manager ~]#mkdir -p /opt/rencher/mysql

[root@reancher-manager ~]#docker run -d --name rencher --restart=always -v /opt/rencher/mysql:/var/lib/mysql -p 8080:8080 rancher/server

[root@reancher-manager ~]#docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                              NAMES
c8209da7add0        rancher/server      "/usr/bin/s6-svscan /"   12 minutes ago      Up 12 minutes       3306/tcp, 0.0.0.0:8080->8080/tcp   rencher
```

## mysql 说明

```
使用自己的mysql数据库，可使用如下参数：

docker run -d --restart=always -p 8080:8080 \
-e CATTLE_DB_CATTLE_MYSQL_HOST=<hostname or IP of MySQL instance> \
-e CATTLE_DB_CATTLE_MYSQL_PORT=<port> \
-e CATTLE_DB_CATTLE_MYSQL_NAME=<Name of database> \
-e CATTLE_DB_CATTLE_USERNAME=<Username> \
-e CATTLE_DB_CATTLE_PASSWORD=<Password> \
rancher/server:v1.1.3
```


## rancher WEB UI

```
http://10.6.0.140:8080
```
![此处输入图片的描述][1]

![此处输入图片的描述][2]

![此处输入图片的描述][3]


## 安装 rancher-agent

```
# node 节点
docker run -d --privileged -v /var/run/docker.sock:/var/run/docker.sock -v /var/lib/rancher:/var/lib/rancher rancher/agent:v1.0.2 http://10.6.0.140:8080/v1/scripts/8944D0EC8BCFEB4F127C:1472544000000:BIX8IC8bWsRbx60NMhka4AmxmpQ
```

```
[root@reancher-node-1 ~]# docker ps -a
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS                     PORTS               NAMES
6c7f533527ec        rancher/agent:v1.0.2   "/run.sh run"            4 minutes ago       Up 4 minutes                                   rancher-agent
7566aa61cdbe        rancher/agent:v1.0.2   "/run.sh state"          4 minutes ago       Exited (0) 4 minutes ago                       rancher-agent-state
032d85c88779        rancher/agent:v1.0.2   "/run.sh http://10.6."   5 minutes ago       Exited (0) 4 minutes ago                       fervent_morse

```
![此处输入图片的描述][4]
![此处输入图片的描述][5]



## 添加 容器
![此处输入图片的描述][6]



## 部署 stack 与 service
![此处输入图片的描述][7]
![此处输入图片的描述][8]
![此处输入图片的描述][9]
![此处输入图片的描述][10]
![此处输入图片的描述][11]
![此处输入图片的描述][12]
![此处输入图片的描述][13]
![此处输入图片的描述][14]


## 生成 Load Balance
![此处输入图片的描述][15]
![此处输入图片的描述][16]
![此处输入图片的描述][17]
![此处输入图片的描述][18]
![此处输入图片的描述][19]
![此处输入图片的描述][20]


## 访问应用
![此处输入图片的描述][21]
![此处输入图片的描述][22]


# rancher 网络

> Rancher 中  网络+负载均衡  实现 与 说明

>依赖镜像：rancher/agent-instance:v0.8.3
>Rancher 网络是 采用SDN技术所建容器为虚拟ip地址，各host之间容器采用ipsec隧道实现跨主机通信，使用的是udp的500和4500端口。
>启动任务时，在各个host部署容器之前会起一个Network  Agent容器，负责组建网络环境。   
![此处输入图片的描述][23]


# 破坏测试
> 破坏性测试 (以下为别人测试)

server 是以容器方式运行，Mysql数据库保存了任务数据以及任务逻辑关系


## 破坏server端

1.

操作：在server端和agent端正常运行状态下，stop掉server容器

结果：业务不受影响。start重启容器后恢复管理功能。


2.

操作：将server端容器rm删除掉, Mysql数据未保存，重新再起一个server容器。

结果：1.当前业务不受影响

2.新server仍然能够识别和管理各个agent，因为agent端是连server的ip端口，ip不变就能连上

3.agent端原有的任务容器的命名和逻辑关系没有了。

3.

操作：将server端容器rm删除掉（将mysql数据/var/lib/mysql 映射至宿主机），重新再起一个server容器。

结果：新起的容器能够识别任务状态，命名，逻辑关系。恢复到之前的状态。

 

## 破坏agent端

4.

操作：host命令行下删除掉agent容器

结果：不影响当前业务状态，server端显示host失联，无法对该agent下发任务进行扩容和缩容。

重新启动agent后恢复正常。

5.

操作：server控制端删除agent端的业务容器（例如删除nginx容器）

结果：删除后数秒内，在另一个host上重新启动一个新的业务容器。

6.

操作：host命令行下删除agent端的业务容器（例如删除nginx容器）

结果：删除后数秒内，在当前host上重新启动一个新的业务容器。

7.

操作：host命令行下删除掉agent容器后，再删除一个业务容器

结果：server端因为与agent失联，导致无法更新该host上的容器变化，没有新启动任何容器。


  [1]: https://jicki.cn/img/posts/rancher/1.png
  [2]: https://jicki.cn/img/posts/rancher/2.png
  [3]: https://jicki.cn/img/posts/rancher/3.png
  [4]: https://jicki.cn/img/posts/rancher/4.png
  [5]: https://jicki.cn/img/posts/rancher/5.png
  [6]: https://jicki.cn/img/posts/rancher/6.png
  [7]: https://jicki.cn/img/posts/rancher/7.png
  [8]: https://jicki.cn/img/posts/rancher/8.png
  [9]: https://jicki.cn/img/posts/rancher/9.png
  [10]: https://jicki.cn/img/posts/rancher/10.png
  [11]: https://jicki.cn/img/posts/rancher/11.png
  [12]: https://jicki.cn/img/posts/rancher/12.png
  [13]: https://jicki.cn/img/posts/rancher/13.png
  [14]: https://jicki.cn/img/posts/rancher/14.png
  [15]: https://jicki.cn/img/posts/rancher/15.png
  [16]: https://jicki.cn/img/posts/rancher/16.png
  [17]: https://jicki.cn/img/posts/rancher/17.png
  [18]: https://jicki.cn/img/posts/rancher/18.png
  [19]: https://jicki.cn/img/posts/rancher/19.png
  [20]: https://jicki.cn/img/posts/rancher/20.png
  [21]: https://jicki.cn/img/posts/rancher/21.png
  [22]: https://jicki.cn/img/posts/rancher/22.png
  [23]: https://jicki.cn/img/posts/rancher/23.png

