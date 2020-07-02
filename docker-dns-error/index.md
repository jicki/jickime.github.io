# docker swarm name service 通信错误



> docker swarm 之间使用 name service 通信，容器重启后IP变动导致的通信故障

## 故障说明

kafka 配置

```
zookeeper.connect=zookeeper-1:2181,zookeeper-2:2181,zookeeper-3:2181
```

当 zookeeper 容器重启后 overlay 分配给 zookeeper 的IP变动， 导致kakfa 出现故障。

重启 kafka 容器以后 恢复，但是 程序 连接 kafka 时间使用的 配置是 kafka name service

这时候 程序出现连接不上 kafka, 需要重启 程序。


## 解决方案


修改 JVM DNS 缓存, JVM默认 DNS 缓存时间是永远有效


两种方式设置dns缓存的方法：

```

# 1.在JAVA_OPTS里设置
-Dsun.net.inetaddr.ttl=3 -Dsun.net.inetaddr.negative.ttl=1

# 2.修改property
System.setProperty("sun.net.inetaddr.ttl", "3");
System.setProperty("sun.net.inetaddr.negative.ttl", "1");
```

sun.net.inetaddr.ttl=3  表示 DNS 缓存时间为 3 秒
sun.net.inetaddr.negative.ttl = 1 表示开启 DNS 缓存时间，默认为10秒。(0表示禁止缓存, -1表示永久缓存)




docker 指定 ip

docker network 创建网络时指定 subnets IP段

docker run 时 指定 ip (指定尽量最后面的IP地址，避免IP被占用)


```
docker network create --driver overlay --subnet=10.1.0.0/16 my-net

[root@swarm-master ~]# docker network inspect my-net
[
    {
        "Name": "my-net",
        "Id": "83333c68bdddf95fd9f398735733258722950969c2a76a3929e70c16d845ffe4",
        "Scope": "global",
        "Driver": "overlay",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.1.0.0/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]


# 创建容器

docker run -d --name jicki --net=my-net --ip 10.1.10.10 alpine ping www.qq.com


[root@swarm-master ~]# docker exec jicki ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN 
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
239: eth0@if240: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1450 qdisc noqueue state UP 
    link/ether 02:42:0a:01:0a:0a brd ff:ff:ff:ff:ff:ff
    inet 10.1.10.10/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe01:a0a/64 scope link 
       valid_lft forever preferred_lft forever
241: eth1@if242: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UP 
    link/ether 02:42:ac:12:00:0d brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.13/16 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:d/64 scope link 
       valid_lft forever preferred_lft forever
	   

docker run -d --name jicki2 --net=my-net alpine ping www.qq.com

# 测试通信
[root@swarm-master ~]# docker exec jicki ping jicki2
PING jicki2 (10.1.0.2): 56 data bytes
64 bytes from 10.1.0.2: seq=0 ttl=64 time=0.149 ms
64 bytes from 10.1.0.2: seq=1 ttl=64 time=0.096 ms
64 bytes from 10.1.0.2: seq=2 ttl=64 time=0.081 ms
64 bytes from 10.1.0.2: seq=3 ttl=64 time=0.080 ms
64 bytes from 10.1.0.2: seq=4 ttl=64 time=0.090 ms
64 bytes from 10.1.0.2: seq=5 ttl=64 time=0.089 ms
```

