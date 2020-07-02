# docker 1.12 swarm 集群



# docker 1.12 swarm

## 新特性

1. docker swarm：集群管理，子命令有init, join, leave, update

2. docker service：服务创建，子命令有create, inspect, update, remove, tasks

3. docker node：节点管理，子命令有accept, promote, demote, inspect, update, tasks, ls, rm

4. docker stack/deploy：试验特性，用于多应用部署， 类似与 docker-compose 中的特性。

 

## 安装 docker

```
wget -qO- https://get.docker.com/ | sh
```


```
[root@swarm-manager ~]# docker version
Client:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   23cf638
 Built:        
 OS/Arch:      linux/amd64

Server:
 Version:      1.12.1
 API version:  1.24
 Go version:   go1.6.3
 Git commit:   23cf638
 Built:        
 OS/Arch:      linux/amd64
```


## btrfs 文件系统

> 修改 docker 默认的存储系统 为 btrfs

官方文档 https://docs.docker.com/engine/userguide/storagedriver/btrfs-driver/


前提:

	系统 必须支持 btrfs (cat /proc/filesystems | grep btrfs)
	
	
	需要 一块未使用的分区或者未使用的硬盘

```
PS:
	如果系统正在使用，没有未使用的分区或者硬盘，可使用虚拟方式创建

    #选择一个分区，如 /opt
	mkdir /opt/btrfsimg
	
	#创建一个 100G 的 img 文件
	dd if=/dev/zero of=/opt/btrfsimg/btrfs.img bs=1024 count=102400000

	#格式化为 btrfs 
    mkfs.btrfs -L 'btrfs' /opt/btrfsimg/btrfs.img
	
	#查看类型
	blkid /opt/btrfsimg/btrfs.img
	
	#停止 docker 
	systemctl stop docker.service
	
	#清空 /var/lib/docker
	rm -rf /var/lib/docker/*
	
	#挂载 btrfs.img
	mount -o loop /opt/btrfsimg/btrfs.img /var/lib/docker
	
	#将挂载加入 fstab
	echo "/opt/btrfsimg/btrfs.img /var/lib/docker btrfs defaults 0 0" >> /etc/fstab
	
	#挂载
	mount -a

	#docker 启动项 增加 --storage-driver=btrfs
	sed -i 's/dockerd/dockerd --storage-driver=btrfs --insecure-registry 172.16.1.26:5000/g' /lib/systemd/system/docker.service
	
	#重启 docker
	systemctl daemon-reload
	systemctl start docker.service

	#docekr info 查看
	Storage Driver: btrfs
	Build Version: Btrfs v3.19.1
```


 

# swarm 命令

我们首先来看看 1.12 中 新特性里面的 内置 swarm 命令 （swarmkit采用raft协议构建集群）

```
[root@swarm-manager ~]# docker swarm --help

Usage:  docker swarm COMMAND

Manage Docker Swarm

Options:
      --help   Print usage

Commands:
  init        Initialize a swarm
  join        Join a swarm as a node and/or manager
  join-token  Manage join tokens
  update      Update the swarm
  leave       Leave a swarm

Run 'docker swarm COMMAND --help' for more information on a command.
```
 

## init 初始化

```
[root@swarm-manager ~]# docker swarm init --advertise-addr 10.6.0.140
Swarm initialized: current node (2jqcgbfehfibna79brhc2mwns) is now a manager.

To add a worker to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-5rq3nqvkbk4j4mlyfiykei02qczg9ajoct9u4ig7u6f7bn8kxi-94zjynj4723hwv91oakqh3cda \
    10.6.0.140:2377

To add a manager to this swarm, run the following command:
    docker swarm join \
    --token SWMTKN-1-5rq3nqvkbk4j4mlyfiykei02qczg9ajoct9u4ig7u6f7bn8kxi-7c0h0rv5rg1ufx1j2ztvk0rw4 \
    10.6.0.140:2377
```

执行 docker node ls  可以查看 Swarm 的集群情况   （只能在 manager 中执行）

```
[root@swarm-manager ~]#docker node ls
ID                           NAME                   MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
0vwpni05mew2j84i6gjet44iu *  swarm-manager          Accepted    Ready   Active        Leader
```

```
[root@swarm-manager ~]# netstat -lan|grep 2377
```

可以看到 群集开放了一这个 2377 的端口。

默认绑定 0.0.0.0:2377 ，当然我们也可以使用 docker swarm init --listen-addr <MANAGER-IP>:<PORT> 进行绑定ip

2377 这个端口是用于 Swarm 中 node 节点加入使使用的。

 

## join 加入 Swarm

```
[root@swarm-node-1 ~]#docker swarm join --token SWMTKN-1-5rq3nqvkbk4j4mlyfiykei02qczg9ajoct9u4ig7u6f7bn8kxi-94zjynj4723hwv91oakqh3cda 10.6.0.140:2377

This node joined a Swarm as a worker.
```


这里在 node-1 里面执行了 join 命令，加入了 10.6.0.140 这个 manager 这个 Swarm 集群里

 
```
[root@swarm-manager ~]#docker node ls           
ID                           NAME                   MEMBERSHIP  STATUS  AVAILABILITY  MANAGER STATUS
0vwpni05mew2j84i6gjet44iu *  swarm-manager          Accepted    Ready   Active        Leader
4mqsmp0gzlqeicit98ce8wh2q    swarm-node-1           Accepted    Ready   Active
```

这里可以看到 node-1 已经加入到 swarm 的集群里面来了。

 


## update 命令

```
[root@swarm-manager ~]#docker swarm update
Swarm updated.
```


## leave 命令

退出集群

```
[root@swarm-node-1 ~]#docker swarm leave 10.6.0.140:2377  
```

## inspect 命令

查询 Swarm 集群 的整体信息。 （只能在 manager 中执行）

```
[root@swarm-manager ~]#docker swarm inspect
[
    {
        "ID": "c052zw5ll0ugw08shg2xf7ajp",
        "Version": {
            "Index": 11
        },
        "CreatedAt": "2016-06-23T02:09:18.935434519Z",
        "UpdatedAt": "2016-06-23T02:09:19.155114277Z",
        "Spec": {
            "Name": "default",
            "AcceptancePolicy": {
                "Policies": [
                    {
                        "Role": "worker",
                        "Autoaccept": true
                    },
                    {
                        "Role": "manager",
                        "Autoaccept": false
                    }
                ]
            },
            "Orchestration": {
                "TaskHistoryRetentionLimit": 10
            },
            "Raft": {
                "SnapshotInterval": 10000,
                "LogEntriesForSlowFollowers": 500,
                "HeartbeatTick": 1,
                "ElectionTick": 3
            },
            "Dispatcher": {
                "HeartbeatPeriod": 5000000000
            },
            "CAConfig": {
                "NodeCertExpiry": 7776000000000000
            }
        }
    }
]
```
 

 
# service 命令

```
[root@swarm-manager ~]#docker service --help

Usage:  docker service COMMAND

Manage Docker services

Options:
      --help   Print usage

Commands:
  create      Create a new service
  inspect     Inspect a service
  tasks       List the tasks of a service
  ls          List services
  rm          Remove a service
  scale       Scale one or multiple services
  update      Update a service

Run 'docker service COMMAND --help' for more information on a command.
```
 
## docker create 命令
既 创建 一个 服务。

```
[root@swarm-manager ~]#docker service create --help

Usage:  docker service create [OPTIONS] IMAGE [COMMAND] [ARG...]

Create a new service

Options:
      --constraint value             Placement constraints (default [])
      --endpoint-mode string         Endpoint mode(Valid values: VIP, DNSRR)
  -e, --env value                    Set environment variables (default [])
      --help                         Print usage
  -l, --label value                  Service labels (default [])
      --limit-cpu value              Limit CPUs (default 0.000)
      --limit-memory value           Limit Memory (default 0 B)
      --mode string                  Service mode (replicated or global) (default "replicated")
  -m, --mount value                  Attach a mount to the service
      --name string                  Service name
      --network value                Network attachments (default [])
  -p, --publish value                Publish a port as a node port (default [])
      --replicas value               Number of tasks (default none)
      --reserve-cpu value            Reserve CPUs (default 0.000)
      --reserve-memory value         Reserve Memory (default 0 B)
      --restart-condition string     Restart when condition is met (none, on_failure, or any)
      --restart-delay value          Delay between restart attempts (default none)
      --restart-max-attempts value   Maximum number of restarts before giving up (default none)
      --restart-window value         Window used to evalulate the restart policy (default none)
      --stop-grace-period value      Time to wait before force killing a container (default none)
      --update-delay duration        Delay between updates
      --update-parallelism uint      Maximum number of tasks updated simultaneously
  -u, --user string                  Username or UID
  -w, --workdir string               Working directory inside the container
```
 

docker service create 里面有非常多的 参数。 这里有很详细的使用说明。


下面我们来 创建一个 service 试试看

首先pull 一个 nginx 镜像 下来 用于 测试

```
[root@swarm-manager ~]#docker pull nginx
```

创建 2 个 nginx :

```
[root@swarm-manager ~]#docker service create --name nginx --replicas 2 -p 80:80/tcp nginx
```

## ls 命令
查看 服务 启动情况。

```
[root@swarm-manager ~]#docker service ls
ID            NAME   REPLICAS  IMAGE  COMMAND
1b9a58mlz330  nginx  1/2       nginx  
```



## overlay 网络


```
# 创建一个 覆盖 所有集群的 overlay  网络

docker network create --driver overlay --opt encrypted --subnet=10.0.9.0/24 my-net

#使用 --opt encrypted 标识
```

```
# 创建 service 时添加 --endpoint-mode dnsrr  使用dns 做为服务发现，否则跨主机之间无法通讯


# 例:
docker service create --replicas 3 --name my-nginx --network my-net --endpoint-mode dnsrr nginx:alpine
```


```
# 使用 nslookup my-nginx   查询dns 情况

-----------------------------------------------------------------------------------------------------------
nslookup: can't resolve '(null)': Name does not resolve

Name:      my-nginx
Address 1: 10.0.9.3 b177080c9e65
Address 2: 10.0.9.2 my-nginx.1.0p2gn3h3ujoghub8ilyyvbenq.my-net
Address 3: 10.0.9.4 my-nginx.3.axmsfkrxd89gbp75j00cu5qqw.my-net
-----------------------------------------------------------------------------------------------------------
```






## tasks 命令 

查看 nginx 的情况。

```
[root@swarm-manager ~]#docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE           DESIRED STATE  NODE
56er48j3hin9ysdi3sb1chbn1  nginx.1  nginx    nginx  Preparing 2 minutes  Running        swarm-node-1
e7vtvpkbstznoi8ogihaao1f5  nginx.2  nginx    nginx  Running 2 minutes    Running        swarm-manager
``` 

Swarm模式下的引擎拥有自组织与自修复特性，意味着它们能够识别我们定义的应用，并在出现差错时持续检查并修复环境。

举例来说，如果大家关闭某台运行有Nginx实例的设备，则另一节点上会自动启动一套新的容器。

如果关闭Swarm内半数设备所使用的网络交换机，则另外一半设备会顶替而上，接管对应工作负载。

```
[root@swarm-manager ~]#docker service ls
ID            NAME   REPLICAS  IMAGE  COMMAND
1b9a58mlz330  nginx  2/2       nginx
``` 

```
[root@swarm-manager ~]#docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE          DESIRED STATE  NODE
56er48j3hin9ysdi3sb1chbn1  nginx.1  nginx    nginx  Running 32 minutes  Running        swarm-node-1
e7vtvpkbstznoi8ogihaao1f5  nginx.2  nginx    nginx  Running 32 minutes  Running        swarm-manager
``` 


## scale 命令
批量生成已有容器。

直接对 nginx=10 既可让 nginx 的容器生成10个。

```
[root@swarm-manager ~]#docker service scale nginx=10
nginx scaled to 10
```

使用 tasks 可以看到，已经在 2个 节点中生成了10个 nginx 容器

```
[root@swarm-manager ~]#docker service tasks nginx
ID                         NAME      SERVICE  IMAGE  LAST STATE            DESIRED STATE  NODE
56er48j3hin9ysdi3sb1chbn1  nginx.1   nginx    nginx  Running 43 minutes    Running        swarm-node-1
e7vtvpkbstznoi8ogihaao1f5  nginx.2   nginx    nginx  Running 43 minutes    Running        swarm-manager
9vqxcmskj1nawo8wl0fqr32j2  nginx.3   nginx    nginx  Preparing 20 seconds  Running        swarm-manager
0vbqoyestm7ob6r1zq9jwj6il  nginx.4   nginx    nginx  Running 20 seconds    Running        swarm-node-1
13jf9mkl4k5e57pq4hoeb68ru  nginx.5   nginx    nginx  Running 20 seconds    Running        swarm-node-1
a0tk6ni6a02diuo5u3t870qk7  nginx.6   nginx    nginx  Running 20 seconds    Running        swarm-manager
cwplvo5wfqp3rn5ynvxv9wv90  nginx.7   nginx    nginx  Running 20 seconds    Running        swarm-manager
7feil5xqc5hdkseasthkq2nyx  nginx.8   nginx    nginx  Running 20 seconds    Running        swarm-node-1
8jt5yovxoz7t89edinb9ydao1  nginx.9   nginx    nginx  Starting 20 seconds   Running        swarm-node-1
dst4ydun1upham0o7e8a9hj3w  nginx.10  nginx    nginx  Running 20 seconds    Running        swarm-manager
```
 


当我们想 缩容 时间， 也可以使用 scale nginx=2 让容器变成2个。

```
[root@swarm-manager ~]#docker service scale nginx=2
nginx scaled to 2
``` 


在运行 nginx=2 时可以看到 容器已经缩小为 2个 。

当我们使用 docker ps 查看，会发现容器被 stop 而非 rm 。

当我们使用 docker service rm nginx 的时候，所有的容器都会被 删除，请注意。

```
[root@swarm-manager ~]#docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE          DESIRED STATE  NODE
0vbqoyestm7ob6r1zq9jwj6il  nginx.4  nginx    nginx  Running 12 minutes  Running        swarm-node-1
13jf9mkl4k5e57pq4hoeb68ru  nginx.5  nginx    nginx  Running 12 minutes  Running        swarm-node-1
``` 

 

## update 命令

对 服务的启动 参数 进行 更新/修改。

上面我们新建了一个服务，命令为： 

```
[root@swarm-manager ~]#docker service create --name nginx --replicas 2 -p 80:80/tcp nginx
```

如果我们先新加入了一个 node 想让 nginx 分布在 3个 node 上面， 我们可以使用 update 命令。

```
[root@swarm-manager ~]#docker service update --replicas 3 nginx
nginx
``` 


更新完毕以后 我们可以查看到 REPLICAS 已经变成 3/3

```
[root@swarm-manager ~]#docker service ls
ID            NAME   REPLICAS  IMAGE  COMMAND
1b9a58mlz330  nginx  3/3       nginx  
``` 

docker service update 命令，也可用于直接 升级 镜像等。

```
[root@swarm-manager ~]#docker service tasks nginx
ID                         NAME     SERVICE  IMAGE  LAST STATE           DESIRED STATE  NODE
0vbqoyestm7ob6r1zq9jwj6il  nginx.4  nginx    nginx  Running 41 minutes   Running        swarm-node-1
340e1u31vadq3jtebzeddmatt  nginx.5  nginx    nginx  Preparing 5 seconds  Running        swarm-manager
```


上面我们使用了 nginx 镜像启动了 任务。 使用 update --image 可直接对 image 进行更新。

```
[root@swarm-manager ~]#docker service update --image nginx:new nginx 
nginx
```

可以看到 IMAGE 已经变成 nginx:new

```
[root@swarm-manager ~]#docker service tasks nginx
ID                         NAME     SERVICE  IMAGE      LAST STATE          DESIRED STATE  NODE
2ba3utpk6icf0w449kcwgxmnm  nginx.4  nginx    nginx:new  Running 49 seconds  Running        swarm-manager
5wmmneiueeool09fs8d2g1ncq  nginx.5  nginx    nginx:new  Running 49 seconds  Running        swarm-node-1
```

 

## 挂载目录, mount

```
docker service create --mount type=bind,target=/container_data/,source=/host_data/
```

例 - 本地目录：     target = 容器里面的路径， source = 本地硬盘路径

```
docker service create --name nginx --mount type=bind,target=/usr/share/nginx/html/,source=/opt/web/ --replicas 2 --publish 80:80/tcp nginx
```

```
docker service create --mount type=volume,source=<VOLUME-NAME>,target=<CONTAINER-PATH>,volume-driver=<DRIVER>,
```


例 - 挂载volume卷：  source  =  volume 名称 ,  traget = 容器里面的路径

```
docker service create --name nginx --mount type=volume,source=myvolume,target=/usr/share/nginx/html,volume-driver=local --replicas 2 --publish 80:80/tcp nginx
```
 
#  node 命令

```
[root@swarm-manager ~]#docker node --help

Usage:  docker node COMMAND

Manage Docker Swarm nodes

Options:
      --help   Print usage

Commands:
  demote      Demote one or more nodes from manager in the swarm
  inspect     Display detailed information on one or more nodes
  ls          List nodes in the swarm
  promote     Promote one or more nodes to manager in the swarm
  rm          Remove one or more nodes from the swarm
  ps          List tasks running on a node
  update      Update a node

Run 'docker node COMMAND --help' for more information on a command.
```
 
## accept 命令
用于 同意 申请加入 swarm 集群。


在使用 docker swarm init 的时候，如果使用了 --auto-accept none 的话，需要使用 docker node accept 来通过申请。

在没有通过申请之前，节点 MEMBERSHIP 状态为 Pending 状态。

--auto-accept 可以设置三种角色 分别为 (worker, manager, or none) 。

使用 docker node accept + 节点 ID 既可通过申请。


## promote 与 demote 命令


docker node promote 是 将 worker 普通节点，提升为 manager 节点。

docker node demote 是 将 manager 管理节点，降级为 worker 节点。



## inspect 命令

可查看节点的具体信息

## rm 命令 
可删除一个节点 


## tasks 命令
可查看节点中运行的 service 任务。

 

# docker stack/deploy 

目前 stack 还处于 测试阶段。

https://github.com/docker/docker/blob/master/experimental/docker-stacks-and-bundles.md


 

