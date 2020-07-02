# Flannel 网络模型


# Flannel

![flannel][1]


**Flannel 是 `CoreOS` 公司针对 `Kubernetes` 设计的一个基于`CNI`标准的网络工具, 其目在于帮助 `Kuberentes` 实现一个简单的3层网络。**

- - -

**常用网络术语**

* **2层网络**

  * `OSI` 网络模型中的数据链路层.
  * 处理网络上两个相邻节点之间的帧传递.
  * 以太网就工作在第2层, `MAC`地址表示为子层.


* **3层网络**

  * `OSI` 网络模型中的网络层.
  * 处理主机之间路由数据包.
  * IPv4、IPv6、ICMP 工作在第3层.


* **VXLAN**

  * VXLAN代表虚拟可扩展的`LAN`.
  * VXLAN是一种封装和覆盖协议, 可在现有网络上运行. 
  * VXLAN虚拟化与VLAN类似, 但提供更大的灵活性和功能, VLAN只有4096个网络ID.
  * VXLAN用于通过在UDP数据报中封装第2层以太网帧来实现大型网络部署.

* **Overlay**

  * Overlay网络是建立在现有网络之上的虚拟逻辑网络.
  * Overlay网络通常用于在现有网络之上提供有用的抽象, 并分离和保护不同的逻辑网络.

* **数据封装**

  * 数据封装是指在附加层中封装网络数据包以提供其他上下文和信息的过程.
  * 在Overlay网络中, 数据封装被用于从虚拟网络转换到底层地址空间, 从而能路由到不同的位置. 


* **网状网络(Mesh NetWork)**

  * 网状网络是指每个节点连接到许多其他节点以协作路由、并实现更大连接的网络.
  * 网状网络允许通过多个路径进行路由, 从而提供更可靠的网络.
  * 网状网络的缺点是每个附加节点都会增加大量开销.


* **BGP**

  * BGP代表"边界网关协议", 用于管理边缘路由器之间数据包的路由方式.
  * BGP通过考虑可用路径, 路由规则和特定网络策略, 帮助弄清楚如何将数据包从一个网络发送到另一个网络. 
  * BGP有时被用作CNI插件中的路由机制, 而不是封装的覆盖网络.




## Flannel 工作原理


**Flannel网络模型**

* `udp`:&emsp;使用用户态udp封装,默认使用`8285`端口, 由于是在用户态 封包与解包, 性能上有较大的损耗.

* `vxlan`:&emsp; vxlan封装, 需要配置`VNI`, `Port` ( 默认端口`8247` ) 和 `BGP`.

* `host-gw`:&emsp; 直接路由模式, 将容器网络的路由信息直接更新到主机的路由表中. 仅适用于二层直接可达的网络中, 推荐在网络自由的环境中使用, 效率较高.

* `aws-vpc`:&emsp; 使用 `Amazon VPC route table` 自动创建路由, 仅适用于 AWS 中的 EC2 使用.

* `gce`:&emsp; 使用`Google Compute Engine Network`创建路由, 所有`instance`需要开启`IP forwarding`, 仅适用于 GCE 上运行.

* `Ali-vpc`:&emsp; 使用阿里云 `VPC route table` 创建路由, 仅适用于阿里云ECS上运行.




### UDP 模式


* UDP 模式相对来说比较简单, 采用UDP模式时, 需要在`flannel` 的配置文件中指定`Backend.Type`为`UDP`, 也可通过直接修改`flannel`的`ConfigMap`的方式实现。


* UDP模式中, `flannel` 进程在启动时会通过打开`/dev/net/tun` 的方式生产一个`tun`设备, `tun`设备可简单理解为Linux提供的一种内核网络与用户空间（应用程序）通信的机制, 及应用可以通过直接读写`tun`设备的方式收发`RAW IP`包。

  * `flannel` 启动后会生成一个 `flannel0` 的网络接口(网口), 通过 `ip addr` 可查看.

  *  通过 `ip -d link show flannel0` 可查看此网络接口为 `tun` 设备.

  * `flannel0` 网络接口的 `MTU` 值为 1472 , 相对于物理网口 `eth0` 等少了28个字节.

  * `udp` 模式默认监听端口为 `8285`。



**主机通信**


* 同主机通信

  * 容器A发送到同一个`subnet`的容器B时, 容器都处于同一个子网中, 此时容器A与容器B中的网络都桥接在`cni0`网络接口中, 这样就可以实现同主机的直接通信.


* 跨主机通信

  * 容器A发送 ICMP 到 `cni0` 接口.

  * `cni0` 匹配相应的路由规则, 将 `RAW` 包发送到 `flannel0`接口.

  * 因为`flannel0`为 `tun`设备, 所以`flannel` 会将 `RAW` 进行 UDP封包.

    * UDP报文 包含 UDP 头(8字节), IP信息头(20字节). 所以 MUT 值会比 物理接口 `eth0` 的MUT 值少 28 字节. 为了封包后添加的这28字节.

  * 将 UDP 封包后的 UDP报文 通过 物理接口 `eth0` 转发出去.

  * 容器B 从 物理接口 `eth0` 接收到 容器A 转发过来的 UDP报文.

  * 容器B 中的 `flannel` 将 UDP 报文进行解包, 获得 `RAW` 包.

  * 通过解包后的 `RAW` 包, 获取路由匹配规则, 通过路由规则将 `RAW`转发给 `cni0` 网桥.

  * `cni0` 网桥 将数据包 发送给桥接到 `cni0` 接口的容器B .


### Host-gw 模式

* `host-gw`模式下, 各节点之间的跨主机网络通信要通过节点上的路由表实现, 因此必须要通信双方所在的宿主机能够直接路由。这就要求`flannel` `host-gw`模式下集群中的所有节点必须在一个二层的直接网络内, 这限制使得`host-gw`模式无法适用于集群规模较大且需要对节点进行网段划分的场景。

`host-gw`模式下, 另外一个限制则是随着集群中节点规模的增大, `flannel`维护主机上成千上万条路由表的动态更新也是一个不小的压力, 因此在路由方式下, 路由表规则的数量是限制网络规模的一个重要因素. 

`host-gw`模式下, `flannel`的唯一作用就是负责主机上路由表的动态更新。




### VXLAN 模式


* `VXLAN` 模式使用比较简单, `flannel` 会在各节点生成一个 `flannel.1` 的 VXLAN 网卡(VTEP设备). 

* `VXLAN` 模式下封包与解包的工作是由内核进行的. `flannel`不转发数据, 仅动态设置`ARP`和`FDB`表项.

* `VXLAN` 模式下通信如下(跨主机的情况):

![flannel1][2]

  * 容器A 中的数据包先通过容器A 的路由表发送到 `cni0`.

  * `cni0` 通过匹配主机A中的路由表信息,将数据包发送到 `flannel.1` 接口.

  * `flannel.1` 是一个 `VTEP` 设备, 收到报文后按照 `VTEP`的配置进行封包. 根据`flannel.1`设备创建时设置的参数 `VNI`、`local IP`、`Port` 进行`VXLAN`封包。

  * 主机A通过物理网卡 `eth0` 发送封包到 主机B的物理网卡 `eth0`中.

  * 主机B的物理网卡`eth0` 再通过`VXLAN`默认端口8472 转发到 `VTEP` 设备`flannel.1` 进行解包.

  * 解包以后根据IP头匹配路由规则, 内核将包发送到`cni0`.

  * `cni0` 发送到桥接到此接口的容器B中.





**Flannel VXLAN工作模式**

> flannel 主动给子网添加远端主机路由的方式。同时, VTEP和网桥各自分配三层IP地址。当数据包到达目的主机后, 在内部进行三层寻址, 路由数目与主机数（而不是容器数）线性相关。官方声称同一个VXLAN子网下每个主机对应一个路由表项, 一个ARP表项和一个FDB表项。

* `Flannel` 创建VXLAN设备, 不再监听`L2Miss`和`L3Miss`事件.

  * `L2Miss`, 通过获取`VTEP`上的对外IP地址实现.

  * `L3Miss`, 通过查找`ARP`表`Mac`地址完成.

* `Flannel` 为远端主机创建静态`ARP`表项.

* `Flannel` 创建`FDB`转发表项, 包含`VTEP`、 `Mac`地址和远端`flannel`的对外IP.








  [1]: https://jicki.cn/img/posts/flannel/flannel-logo.png
  [2]: https://jicki.cn/img/posts/flannel/flannel-1.png

