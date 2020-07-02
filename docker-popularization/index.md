# docker 基础知识



# Docker

* 本文用于科普 docker 的一些基础知识以及概念

* LXC = Linux Container

## 一、什么是docker

* Docker 是一个开放源代码软件项目, 让应用程序部署在软件货柜下的工作可以自动化进行, 借此在 Linux 操作系统上, 提供一个额外的软件抽象层, 以及操作系统层虚拟化的自动管理机制。Docker 利用 Linux 核心中的资源分离机制，例如 `cgroups`,以及 Linux 核心名字空间`(LXC Namespace)`，来创建独立的容器。


## 二、资源限制与隔离

* 资源隔离 (Namespace):

  * Linux 实现了 6 项资源隔离 

|Namespace|系统调用参数|隔离内容|支持内核版本|
|-|-|-|-|
|UTS|`CLONE_NEWUTS`|主机名和域名|2.6.19|
|IPC|`CLONE_NEWIPC`|信号量、消息队列和共享内存|2.6.19|
|PID|`CLONE_NEWPID`|进程编号|2.6.24|
|NetWork|`CLONE_NEWNET`|网络设备、网络栈、端口等|2.6.29|
|Mount|`CLONE_NEWNS`|挂载点(文件系统)|2.4.19|
|User|`CLONE_NEWUSER`|用户和用户组|3.8|


* 资源限制 (cgroups):

  * Linux 通过 cgroups 实现 cpu、内存、IO的限制以及网络的分配, (Linux 系统目录 /sys/fs/cgroup/ )

  * 限制内存以及虚拟内存并关闭oom-kill  `docker run --memory 500M --memory-swap 200M --oom-kill-disable `

  * 绑定CPU核心 `docker run --cpu-cpus 1` , `docker run --cpuset-cpus="0-2"` 
 
  * 限制CPU `docker run -c 2048 --cpu-period=50000 --cpu-quota=50000` 

    * `--cpu-quota` cpu调度范围    
  
    * `--cpu-period` cpu调度周期内使用时间
 
    * `-c` cpu 核心ms值

  * 限制磁盘IO

    * `--device-read-bps` 限制此设备上的读速度(bytes per second), 单位可以是kb、mb或者gb

    * `--device-read-iops` 通过每秒读IO次数来限制指定设备的读速度

    * `--device-write-bps` 限制此设备上的写速度（bytes per second），单位可以是kb、mb或者gb。

    * `--device-write-iops` 通过每秒写IO次数来限制指定设备的写速度。

    * `--blkio-weight` 容器默认磁盘IO的加权值，有效值范围为10-100。

    * `--blkio-weight-device` 针对特定设备的IO加权控制。其格式为`DEVICE_NAME:WEIGHT`


 
## 三、容器 VS 虚拟机 

  * 相比传统的虚拟化技术(vm), 使用Docker在CPU, Memory, Disk IO, Network IO  上的性能损耗都有同样水平甚至更优的表现。Container的快速创建、启动、销毁受到很多赞誉。

  * 容器是直接使用宿主机的内核的, 而虚拟机(VM)是虚拟化一个完整操作系统。所以单数量上一台物理机能跑的容器是远大于虚拟机的数量。 


|特性|容器|虚拟机|
|-|-|-|
|启动|秒级|分钟级|
|使用容量|一般为MB|一般为GB|
|性能|接近宿主机性能|弱于宿主机性能|
|系统支持量|单机可支持上千容器|单机可支持几十个|


## 四、docker 基础概念


### 镜像(image)

* Docker镜像是一个特殊的文件系统，提供容器运行时所需的程序、库、资源、配置等文件，另外还包含了一些为运行时准备的一些配置参数（如匿名卷、环境变量、用户等）。

* 镜像是一个静态的概念，不包含任何动态数据，其内容在构建之后也不会被改变。


* 镜像是分层结构

  * docker利用存储驱动 overlay2 的存储技术实现了镜像的分层结构, 分层结构中内容相同的层只占用一个空间, 意思就是只有不同的结构层才会占用存储空间, 相同的层都是直接引用。



* 构建一个镜像

  * `docker build [option] [-t <image>:<tag>]  <path> ` 命令构建镜像

  * `dockerfile` 
  
    * `FROM`:  指定基础镜像

    * `ENV`:  `key = value` 格式 - 设置一个环境变量, 可以被dockerfile里后续的指令使用, 也在容器运行过程中保持。 

    * `ARG`:  `key = value` 格式 `docker build --build-arg <varname>=<value>`  将该变量传递给构建器替换默认变量。
    * `RUN`:  命令操作层,每一个RUN 会新增一层,所以目的相同的RUN尽量合并在同一个RUN里减少大小。

    * `COPY`:  从本地复制 文件/文件夹 到镜像中。

    * `ADD`: 从本地复制 文件/文件夹 到镜像中, 支持压缩文件自解压和网络中下载文件。

    * `WORKDIR`: 声明工作目录,可以支持多个目录, 会自动创建目录。

    * `USER`: 指定构建时 USER 后续操作的 用户。 

    * `EXPOSE`: 声明需要使用的端口, 仅用于声明与显示并不会实际映射端口到外部。`docker ps`  PORTS 栏 会显示声明的端口。

    * `VOLUME`: 容器运行时应该尽量保持容器存储层不发生写操作, 对于数据库类需要保存动态数据的应用, 其数据库文件应该保存于卷(volume)中, 为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 Dockerfile 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。


    * `CMD`:  CMD 指令允许用户指定容器的默认执行的命令。此命令会在容器启动且 `docker run` 没有指定其他命令时运行。 CMD 可以用作 ENTRYPOINT 默认参数, 或者用作 Docker 的默认命令。

    * `ENTRYPOINT`: ENTRYPOINT 的 Exec 格式用于设置容器启动时要执行的命令及其参数, 同时可通过CMD命令或者命令行参数提供额外的参数。ENTRYPOINT 中的参数始终会被使用, 这是与CMD命令不同的一点。


### 容器(Container)

* Docker的容器就是 docker 通过镜像启动一个实例就叫 docker 容器。

* `docker run <images name>` 命令启动容器。


### 仓库(Repository)


