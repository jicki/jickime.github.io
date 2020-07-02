# Ceph RBD CephFS



# Ceph RBD  CephFS


##环境准备 

```
(这里只做基础测试, ceph-manager , ceph-mon, ceph-osd 一共三台)

10.6.0.140 = ceph-manager

10.6.0.187 = ceph-mon-1

10.6.0.188 = ceph-osd-1

10.6.0.94 = node-94
```


## 初始化环境

注： ceph 对时间要求很严格， 一定要同步所有的服务器时间

在 manager 上面修改 /etc/hosts :

```
10.6.0.187 ceph-mon-1
10.6.0.188 ceph-osd-1
10.6.0.94 node-94
```

修改各服务器上面的 hostname (说明：ceph-deploy工具都是通过主机名与其他节点通信)

```
hostnamectl --static set-hostname ceph-manager
hostnamectl --static set-hostname ceph-mon-1
hostnamectl --static set-hostname ceph-osd-1
hostnamectl --static set-hostname node-94
```
配置manager节点与其他节点ssh key 访问

```
[root@ceph-manager ~]# ssh-keygen

# 将key 发送到各节点中

[root@ceph-manager ~]#ssh-copy-id ceph-mon-1
[root@ceph-manager ~]#ssh-copy-id ceph-osd-1
```

## 安装 ceph


在manager节点安装 ceph-deploy

```
[root@ceph-manager ~]#yum -y install centos-release-ceph
[root@ceph-manager ~]#yum makecache
[root@ceph-manager ~]#yum -y install ceph-deploy ntpdate
```

在其他各节点安装 ceph 的yum源

```
[root@ceph-mon-1 ~]# yum -y install centos-release-ceph
[root@ceph-mon-1 ~]# yum makecache

[root@ceph-osd-1 ~]# yum -y install centos-release-ceph
[root@ceph-osd-1 ~]# yum makecache
```


## 创建ceph 目录

```
[root@ceph-manager ~]#mkdir -p /etc/ceph
[root@ceph-manager ~]#cd /etc/ceph
```

## 创建监控节点：

```
[root@ceph-manager /etc/ceph]#ceph-deploy new ceph-mon-1
```

执行完毕会生成 ceph.conf ceph.log ceph.mon.keyring 三个文件

编辑 ceph.conf 增加 osd 节点数量
在最后增加：

```
osd pool default size = 1
```

使用ceph-deploy在所有机器安装ceph

```
[root@ceph-manager /etc/ceph]# ceph-deploy install ceph-manager ceph-mon-1 ceph-osd-1
```

如果出现错误，也可以到各节点中直接 yum -y install ceph ceph-radosgw 进行安装

```
yum -y install ceph ceph-radosgw
```


## 初始化监控节点

```
[root@ceph-manager /etc/ceph]# ceph-deploy mon create-initial
```
 

osd 节点创建存储空间

```
[root@ceph-osd-1 ~]# mkdir -p /opt/osd1
```

在管理节点上启动 并 激活 osd 进程

```
[root@ceph-manager ~]# ceph-deploy osd prepare ceph-osd-1:/opt/osd1
[root@ceph-manager ~]# ceph-deploy osd activate ceph-osd-1:/opt/osd1
```

把管理节点的配置文件与keyring同步至其它节点

```
[root@ceph-manager ~]# ceph-deploy admin ceph-mon-1 ceph-osd-1
```

查看集群健康状态 (HEALTH_OK 表示OK)

```
[root@ceph-manager ~]# ceph health
HEALTH_OK
```
 

## 客户端配置

客户端 挂载: ceph 有多种挂载方式, rbd 块设备映射， cephfs 挂载 等

注:
在生产环境中，客户端应该对应pool的权限，而不是admin 权限

```
[root@ceph-manager ~]# ssh-copy-id node-94
```

## 安装ceph

```
[root@ceph-manager ~]# ceph-deploy install node-94
```

或者 登陆 node-94 执行 

```
yum -y install ceph ceph-radosgw
```

如果ssh 非22端口，会报错 可使用 scp 传

```
scp -P端口 ceph.conf node-94:/etc/ceph/
scp -P端口 ceph.client.admin.keyring node-94:/etc/ceph/
```


## 创建pool

```
[root@ceph-manager ~]# ceph osd pool create press 100
pool 'press' created
```


设置pool 的pgp_num

```
[root@ceph-manager ~]# ceph osd pool set press pgp_num 100
```

查看创建的pool

```
[root@ceph-manager ~]# ceph osd lspools
0 rbd,1 press,
```

设置副本数为2 (osd 必须要大于或者等于副本数，否则报错, 千万注意)

```
[root@ceph-manager ~]# ceph osd pool set press size 2
```


创建一个100G 名为 image 镜像

```
[root@ceph-manager ~]# rbd create -p press --size 100000 image
```

查看一下镜像:

```
[root@ceph-manager ~]# rbd -p press info image
rbd image 'image':
size 100000 MB in 25000 objects
order 22 (4096 kB objects)
block_name_prefix: rb.0.104b.74b0dc51
format: 1
```
 


## 客户端块存储挂载：
在node-94 上面 map 镜像

```
[root@node-94 ~]# rbd -p press map image
/dev/rbd0
```

格式化 image

```
[root@node-94 ~]# mkfs.xfs /dev/rbd0
```


创建挂载目录

```
[root@node-94 ~]# mkdir /opt/rbd
```

挂载 rbd

```
[root@node-94 ~]# mount /dev/rbd0 /opt/rbd

[root@node-94 ~]# time dd if=/dev/zero of=haha bs=1M count=1000
```
 

取消 map 镜像

```
[root@node-94 ~]# umount /opt/rbd
[root@node-94 ~]# rbd unmap /dev/rbd0
```
 
## cephFS 文件系统

客户端 cephFS 文件系统 (cephFS 必须要有2个osd 才能运行，请注意)：

使用 cephFS 集群中必须有 mds 服务

创建 mds 服务 (由于机器有限就在 mon 的服务器上面 创建 mds 服务)

```
[root@ceph-manager ~]# ceph-deploy mds create ceph-mon-1
```
 

创建2个pool 做为文件系统的data 与 metadata

```
[root@ceph-manager ~]# ceph osd pool create cephfs_data 99
pool 'cephfs_data' created


[root@ceph-manager ~]# ceph osd pool create cephfs_metadata 99
pool 'cephfs_metadata' created
```

创建 文件系统：

```
[root@ceph-manager ~]# ceph fs new jicki cephfs_metadata cephfs_data
new fs with metadata pool 6 and data pool 5
```

查看所有文件系统：

```
[root@ceph-manager ~]# ceph fs ls
name: jicki, metadata pool: cephfs_metadata, data pools: [cephfs_data ]
```

删除一个文件系统

```
[root@ceph-manager ~]# ceph fs rm jicki --yes-i-really-mean-it
```
 

## 客户端挂载 cephFS

安装 ceph-fuse：

```
[root@node-94 ~]# yum install ceph-fuse -y
```

创建挂载目录:

```
[root@node-94 ~]# mkdir -p /opt/jicki
[root@node-94 ~]# ceph-fuse /opt/jicki
[root@node-94 ~]# df -h|grep ceph
ceph-fuse 1.6T 25G 1.6T 2% /opt/jicki
```
 


# ceph 相关命令：


## manager 篇

查看实时的运行状态信息：

```
[root@ceph-manager ~]# ceph -w
```

查看状态信息：

```
[root@ceph-manager ~]# ceph -s
```

查看存储空间：

```
[root@ceph-manager ~]# ceph df
```

删除某个节点的所有的ceph数据包：

```
[root@ceph-manager ~]# ceph-deploy purge ceph-mon-1
[root@ceph-manager ~]# ceph-deploy purgedata ceph-mon-1
```

为ceph创建一个admin用户并为admin用户创建一个密钥，把密钥保存到/etc/ceph目录下：

```
[root@ceph-manager ~]# ceph auth get-or-create client.admin mds 'allow' osd 'allow *' mon 'allow *' -o /etc/ceph/ceph.client.admin.keyring
```

为osd.ceph-osd-1创建一个用户并创建一个key

```
[root@ceph-manager ~]# ceph auth get-or-create osd.ceph-osd-1 mon 'allow rwx' osd 'allow *' -o /etc/ceph/keyring
```

为mds.ceph-mon-1创建一个用户并创建一个key

```
[root@ceph-manager ~]# ceph auth get-or-create mds.ceph-mon-1 mon 'allow rwx' osd 'allow *' mds 'allow *' -o /etc/ceph/keyring
```

查看ceph集群中的认证用户及相关的key

```
[root@ceph-manager ~]# ceph auth list
```

删除集群中的一个认证用户

```
[root@ceph-manager ~]# ceph auth del osd.0
```

查看集群健康状态详细信息

```
[root@ceph-manager ~]# ceph health detail
```

查看ceph log日志所在的目录

```
[root@ceph-manager ~]# ceph-conf --name mds.ceph-manager --show-config-value log_file
```
 

## mon 篇

查看mon的状态信息

```
[root@ceph-manager ~]# ceph mon stat
```

查看mon的选举状态

```
[root@ceph-manager ~]# ceph quorum_status --format json-pretty
```

看mon的映射信息

```
[root@ceph-manager ~]# ceph mon dump
```
 

删除一个mon节点

```
[root@ceph-manager ~]# ceph mon remove ceph-mon-1
```

获得一个正在运行的mon map，并保存在mon-1-map.txt文件中

```
[root@ceph-manager ~]# ceph mon getmap -o mon-1-map.txt
```

查看mon-1-map.txt

```
[root@ceph-manager ~]# monmaptool --print mon-1-map.txt
```

把上面的mon map注入新加入的节点

```
[root@ceph-manager ~]# ceph-mon -i ceph-mon-3 --inject-monmap mon-1-map.txt
```

查看mon的socket

```
[root@ceph-manager ~]# ceph-conf --name mon.ceph-mon-1 --show-config-value admin_socket
```

查看mon的详细状态

```
[root@ceph-mon-1 ~]# ceph daemon mon.ceph-mon-1 mon_status
```

删除一个mon节点

```
[root@ceph-manager ~]# ceph mon remove ceph-mon-1
```


## msd 篇

查看msd状态

```
[root@ceph-manager ~]# ceph mds dump
```

删除一个mds节点

```
[root@ceph-manager ~]# ceph mds rm 0 mds.ceph-mds-1
```
 

 

## osd 篇

查看ceph osd运行状态

```
[root@ceph-manager ~]# ceph osd stat
```

查看osd映射信息

```
[root@ceph-manager ~]# ceph osd stat
```

查看osd的目录树

```
[root@ceph-manager ~]# ceph osd tree
```

down掉一个osd硬盘 (ceph osd tree 可查看osd 的硬盘信息，下面为down osd.0 节点)

```
[root@ceph-manager ~]# ceph osd down 0
```

在集群中删除一个osd硬盘

```
[root@ceph-manager ~]# ceph osd rm 0
```

在集群中删除一个osd 硬盘 并 crush map 清除map信息

```
[root@ceph-manager ~]# ceph osd crush rm osd.0
```

在集群中删除一个osd的host节点

```
[root@ceph-manager ~]# ceph osd crush rm ceph-osd-1
```

查看最大osd的个数 

```
[root@ceph-manager ~]# ceph osd getmaxosd
```

设置最大的osd的个数（当扩大osd节点的时候必须扩大这个值）

```
[root@ceph-manager ~]# ceph osd setmaxosd 10
```

设置osd crush的权重 ceph osd crush set <ID> <WEIGHT> <NAME> ID WEIGHT NAME 使用 ceph osd tree 查看

```
[root@ceph-manager ~]# ceph osd crush set 1 3.0 host=ceph-osd-1
```

设置osd 的权重 ceph osd reweight <ID> <REWEIGHT>

```
[root@ceph-manager ~]# ceph osd reweight 1 0.5
```

把一个osd节点踢出集群

```
[root@ceph-manager ~]# ceph osd out osd.1
```

把踢出的osd重新加入集群

```
[root@ceph-manager ~]# ceph osd in osd.1
```

暂停osd （暂停后整个集群不再接收数据）

```
[root@ceph-manager ~]# ceph osd pause
```
 

再次开启osd （开启后再次接收数据） 

```
[root@ceph-manager ~]# ceph osd unpause
```
 

## PG 篇

查看pg组的映射信息

```
[root@ceph-manager ~]# ceph pg dump |more
```
 

查看一个PG的map

```
[root@ceph-manager ~]# ceph pg map 0.3f
```

查看PG状态

```
[root@ceph-manager ~]# ceph pg stat
```

查询一个pg的详细信息

```
[root@ceph-manager ~]# ceph pg 0.39 query
```
 

查看pg中stuck的状态 (如有非正常pg会显示)

```
[root@ceph-manager ~]# ceph pg dump_stuck unclean
[root@ceph-manager ~]# ceph pg dump_stuck inactive
[root@ceph-manager ~]# ceph pg dump_stuck stale
```
 

显示一个集群中的所有的pg统计

```
[root@ceph-manager ~]# ceph pg dump --format plain|more
```
 

恢复一个丢失的pg (og-id 为丢失的pg, 使用ceph pg dump_stuck inactive|unclean|stale 查找)

```
[root@ceph-manager ~]# ceph pg {pg-id} mark_unfound_lost revert
```
 


## pool 篇

查看ceph集群中的pool数量 

```
[root@ceph-manager ~]# ceph osd lspools
```

查看 PG组 号码：

```
[root@ceph-manager ~]# ceph osd pool get rbd pg_num
```

在ceph集群中创建一个pool

```
[root@ceph-manager ~]# ceph osd pool create test 100 (名称为 test, 100为PG组号码)
```

为一个ceph pool配置配额

```
[root@ceph-manager ~]# ceph osd pool set-quota test max_objects 10000
```

显示所有的pool

```
[root@ceph-manager ~]# ceph osd pool ls
```

在集群中删除一个pool

```
[root@ceph-manager ~]# ceph osd pool delete test test --yes-i-really-really-mean-it
```
 

显示集群中pool的详细信息

```
[root@ceph-manager ~]# rados df
```

给一个pool创建一个快照

```
[root@ceph-manager ~]# ceph osd pool mksnap test test-snap
```

删除pool的快照

```
[root@ceph-manager ~]# ceph osd pool rmsnap test test-snap
```

查看data池的pg数量

```
[root@ceph-manager ~]# ceph osd pool get test pg_num
```

设置data池的最大存储空间（默认是1T, 1T = 1000000000000, 如下为100T)

```
[root@ceph-manager ~]# ceph osd pool set test target_max_bytes 100000000000000
```

设置data池的副本数

```
[root@ceph-manager ~]# ceph osd pool set test size 3
```

设置data池能接受写操作的最小副本为2

```
[root@ceph-manager ~]# ceph osd pool set test min_size 2
```
 

查看集群中所有pool的副本尺寸

```
[root@ceph-manager ~]# ceph osd dump | grep 'replicated size'
```

设置一个pool的pg数量

```
[root@ceph-manager ~]# ceph osd pool set test pg_num 100
```

设置一个pool的pgp数量

```
[root@ceph-manager ~]# ceph osd pool set test pgp_num 100
```

查看ceph pool中的ceph object (volumes 为pool名称)（这里的object是以块形式存储的）

```
[root@ceph-manager ~]# rados ls -p volumes | more
```

创建一个对象object

```
[root@ceph-manager ~]# rados create test-object -p test
```
 

查看object

```
[root@ceph-manager ~]# rados -p test ls
```

删除一个对象

```
[root@ceph-manager ~]# rados rm test-object -p test
```

查看ceph中一个pool里的所有镜像 (volumes 为pool名称)

```
[root@ceph-manager ~]# rbd ls volumes
```
 

在test池中创建一个命名为images的1000M的镜像

```
[root@ceph-manager ~]# rbd create -p test --size 1000 images
```

查看刚创建的镜像信息

```
[root@ceph-manager ~]# rbd -p test info images
```
 

删除一个镜像

```
[root@ceph-manager ~]# rbd rm -p test images
```

调整一个镜像的尺寸

```
[root@ceph-manager ~]# rbd resize -p test --size 2000 images
```

给镜像创建一个快照 (池/镜像名@快照名)

```
[root@ceph-manager ~]# rbd snap create test/images@images1
```

删除一个镜像文件的一个快照

```
[root@ceph-manager ~]# rbd snap rm 快照池/快照镜像文件@具体快照
```
 

如果删除快照提示保护，需要先删除保护

```
[root@ceph-manager ~]# rbd snap unprotect 快照池/快照镜像文件@具体快照
```

删除一个镜像文件的所有快照

```
[root@ceph-manager ~]# rbd snap purge -p 快照池/快照镜像文件
```

把ceph pool中的一个镜像导出

```
[root@ceph-manager ~]# rbd export -p images --image <具体镜像id> /tmp/images.img
```

