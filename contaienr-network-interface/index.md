# Container Network Interface - CNI



# CNI

> CNI 既 Container Network Interface 的简写.

* **CNI 是 Kubernetes 网络的一个标准**

  * `CNI` 提供 `POD` 的网络连接.

  * `CNI` 提供分配IP地址的能力, 为`POD` 提供 ipv4 与 ipv6 的 DHCP 功能.

  * `CNI` 还提供了 `NetWrok Policy` 的能力, 定义 `POD` 与 `POD` 之间的网络规则.


- - -

* **CNI运行原理**

  * 根据`CNI`标准, 网络组件需要在 `CNI` 下创建二进制执行文件, 用于配置网络环境.

  * 当 `kubernetes` 创建/删除 `pod` 时, 调用二进制执行文件进行相应的网络配置. 

  * `CNI` 下的二进制执行文件会包含如下参数

    * `Container ID`&emsp; 容器ID号, 需要用于指定此容器配置网络, 也有一些`CNI`组件需要利用容器ID进行数据索引.

    * `Network namespace path`&emsp; 当前 `container` 真正的 `network namespace` 路径 . 每个 `container` 都会拥有至少一个 `network namespace`. `namespace` 是通过 `Linux kernel` 操作的, 因此大部分 `CNI` 组件都会根据这个路径, 通过 `network namespace` 进行相关的网络配置.

    * `Network configuration`&emsp; 用于定义`CNI`组件的相关配置. 包含两部分组成 `CNI`标准 和 网络组件额外定义.

    * `Name of the interface inside the container`&emsp; 在当前`container` 中创建网卡名称, 如: `eth0` 等.

 
- - -

**network namespace**

* 如上面所说每个 `container` 都至少拥有一个 `network namespace`, 但两个 `container` 可以共用一个相同的 `network namespace` .  

```
# 创建一个容器 n1
[root@k8s-node-1 ~]# docker run -d --name n1 alpine sleep 3600s


# 查看容器相关网络
[root@k8s-node-1 ~]# docker exec -it n1 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

```


```
# 创建另一个容器 n2 并配置网络 --net=container:{容器id}
# 容器id 是 n1 的容器id
[root@k8s-node-1 ~]# docker run -d --net=container:b27fa0d5db21 --name n2 alpine sleep 3600s


# 查看容器相关网络
[root@k8s-node-1 ~]# docker exec -it n2 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:02
          inet addr:172.17.0.2  Bcast:172.17.255.255  Mask:255.255.0.0
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```


* 可以看到如上两个容器的网络都是同一个, 这里就可以想到 `kubernetes` 的 `pod` 的网络.

* 实际 `pod` 中有更灵活的设置, 详见如下架构图

![cni][1]



- - -


**Network Configuration**

* `Network Configuration` 就是一个 `json` 格式的配置文件. 如上我们讲过 这个配置文件包含两部分

  * `CNI` 标准的配置 - 根据官方 `CNI Spec` 包含如下配置项

    * `cniVersion`&emsp; 定义`CNI`的版本, `CNI` 组件必须相同, 否则报错.

    * `name`&emsp; 唯一的名称标识.

    * `type`&emsp; `CNI`组件定义的名称, 上层应用会根据这个名称去找对应的`CNI`组件. 然后调用`CNI`组件的二进制文件进行网络配置. 这个配置非常重要.

    * `args`&emsp; 额外的配置, 主要用于上层应用调用`CNI`时的一些额外参数. 

    * `ipMasq`&emsp; 标注当前`CNI`组件是否支持 `SNAT`. 只做标识, 实际`SNAT`是在`CNI`组件中实现.

    * `dns`&emsp; `CNI`组件的 DNS 配置, 如: `nameserver`、`search`、`domain`、`options` 等. 注: 在`kubernetes`中, 会忽略这里的 DNS 配置, 因为 `kubernetes` 会在自身设置 DNS 配置.

    * `ipam`&emsp; 既 `IP Address Management Plugin`, 负责IP地址的分配. 一般为`host-local` 和 `dhcp` .


* 一个配置文件

```
{
  "cniVersion": "0.4.0",
  "name": "dbnet",
  "type": "bridge",
  // type (plugin) specific
  "bridge": "cni0",
  "ipam": {
    "type": "host-local",
    // ipam specific
    "subnet": "10.1.0.0/16",
    "gateway": "10.1.0.1"
  },
  "dns": {
    "nameservers": [ "10.1.0.1" ]
  }
}
```



## Kubernetes CNI


* `Kubernetes` 目前网络有两种配置, 通过 `kubelet` 中的 `--network-plugin` 来指定选择哪一种配置.

  * `cni`&emsp; 目前`kubernetes` 部署方式基本都使用 `cni` 配置网络. 如上所述, `cni`包含两个部分

    * `cni`组件的二进制文件 `--cni-bin-dir` 默认存放于 `/opt/cni/bin` 目录下.

    *  `cni`组件的 网络配置(Network Configuration) 或 网络配置列表(Network Configuration List) `--cni-conf-dir` 默认存放于 `etc/cni/net.d/` 目录中.


```
[root@k8s-node-1 ~]# ls -lt /opt/cni/bin/
total 36132
-rwxr-xr-x 1 root root 2973336 Mar 26  2019 bridge
-rwxr-xr-x 1 root root 7598064 Mar 26  2019 dhcp
-rwxr-xr-x 1 root root 2110208 Mar 26  2019 flannel
-rwxr-xr-x 1 root root 2288536 Mar 26  2019 host-device
-rwxr-xr-x 1 root root 2238208 Mar 26  2019 host-local
-rwxr-xr-x 1 root root 2621472 Mar 26  2019 ipvlan
-rwxr-xr-x 1 root root 2257808 Mar 26  2019 loopback
-rwxr-xr-x 1 root root 2650160 Mar 26  2019 macvlan
-rwxr-xr-x 1 root root 2613864 Mar 26  2019 portmap
-rwxr-xr-x 1 root root 2946664 Mar 26  2019 ptp
-rwxr-xr-x 1 root root 1951880 Mar 26  2019 sample
-rwxr-xr-x 1 root root 2103456 Mar 26  2019 tuning
-rwxr-xr-x 1 root root 2617328 Mar 26  2019 vlan
```



```
[root@k8s-node-1 ~]# ls -lt /etc/cni/net.d/
total 4
-rw-r--r-- 1 root root 292 Apr 26 14:05 10-flannel.conflist
```


```
[root@k8s-node-1 ~]# cat /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "cniVersion": "0.3.1",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```


**CNI工作流程**

* `Client` 发送创建 `Pod`的请求. `Pod`的创建中间流程忽略, 最后到 `kubelet` .

* `kubelet` 通过`dockershim` + `docker-containerd` 的方式创建 `container` .

* `Pod` 创建完毕以后, `kubelet` 通过 `CNI` 相关流程创建 `Pod` 网络.

  * `CNI`组件搜索 `--cni-conf-dir` 目录下的配置文件, 按照数字顺序查找合法的网络配置. `Flannel` 的配置文件为 `10-flannel.conflist` 
  * 读取配置文件, 通过 `type`  或者 `cni`组件. 
  * 通过 `type` 调用`--cni-bin-dir` 目录下的  `cni`组件的二进制文件.
  * 通过配置文件参数配置容器网络.
  * 配置完网络以后, 调用 `--cni-bin-dir` 目录下的 `portmap` 二进制文件, 将容器的IP和端口通过`iptables`映射到宿主机的端口上.




**Kubenet工作流程**


* `kubenet` 是个非常简单的 `L2 Bridge`,  其实也是通过 `CNI` 建立一个简单的网络, 由 `kubenet` 配置的网络只能用于单节点的网络通讯. 所以 `kubenet` 是用于测试.










  [1]: https://jicki.cn/img/posts/cni/cni.png


