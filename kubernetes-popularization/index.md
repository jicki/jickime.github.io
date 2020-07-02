# kubernetes 基础知识


# Kubernetes

> Kubernetes 是 Google 基于 Borg 开源的容器编排调度，用于管理容器集群自动化部署、扩容以及运维的开源平台。作为云原生计算基金会 CNCF（Cloud Native Computing Foundation）最重要的组件之一（CNCF 另一个毕业项目 Prometheus ），它的目标不仅仅是一个编排系统，而是提供一个规范，可以让你来描述集群的架构，定义服务的最终状态，Kubernetes 可以帮你将系统自动地达到和维持在这个状态，Kubernetes 也可以对容器(Docker)进行集群管理和服务编排（Docker Swarm 类似的功能）,对于大多开发者来说，以容器的方式运行一个程序是一个最基本的需求，跟多的是 Kubernetes 能够提供路由网关、水平扩展、监控、备份、灾难恢复等一系列运维能力。


* 本文用于科普 `Kubernetes` 的一些基础知识以及概念。


## Kubernetes 核心组件

![图1][1]


* `Kubernetes` 集群由两种角色组成:

  * `Master` 集群调度节点, 管理集群。(kube-apiserver, kube-controller-manager, kube-scheduler, etcd)

  * `Node` 引用程序实际运行的工作节点。(kubelet, kube-proxy)



### Master 端

* `kube-apiserver` 集群控制入口, 提供了 `HTTP Rest` 接口的关键服务进程，是 `Kubernetes` 里所有资源的增删改查等操作的唯一入口, 也是集群控制的入口。

  * 提供集群管理的 REST API 接口，包括认证授权、数据校验以及集群状态变更等。

  * 提供其他模块之间的数据交互和通信的枢纽（其他模块通过 API Server 查询或修改数据，只有 API Server 才直接操作 etcd）



* `kube-controller-manager` 服务运行控制器, 处理常规任务的后台线程 比如故障检测、自动扩展、滚动更新等。kube-controller-manager 由一系列的控制器组成包括如下控制器

  * `Replication Controller`

  * `Node Controller`

  * `CronJob Controller`

  * `Daemon Controller`

  * `Deployment Controller`

  * `Endpoint Controller`

  * `Garbage Collector`

  * `Namespace Controller`

  * `Job Controller`

  * `Pod AutoScaler`

  * `RelicaSet`

  * `Service Controller`

  * `ServiceAccount Controller`

  * `StatefulSet Controller`

  * `Volume Controller`

  * `Resource quota Controller`  


* `kube-scheduler` 负责 Pod 资源调度。监视新创建没有分配到Node的Pod, 为Pod选择一个Node。调度方式:

  * 公平调度

  * 资源高效利用

  * QoS

  * affinity 和 anti-affinity (约束)

  * 数据本地化（data locality）

  * 内部负载干扰（inter-workload interference）

  * deadlines



* `etcd` 用于共享配置和服务发现，存储 Kubernetes 集群所有的网络配置和对象的状态信息。


### Node 端


* `kubelet` 负责 Pod 对应的容器的创建，启动等任务，同时与Master节点密切协作, kubelet 提供了一系列`PodSpecs`集合规范，并确保这些`PodSpecs`中描述的容器运行正常。 kubelet是主要的节点代理，它会监视已分配给节点的pod, 具体功能

  * 安装 Pod 所需的volume。

  * 下载 Pod 的Secrets。

  * 监控 Pod 中运行的 docker（或experimentally，rkt）容器。 

  * 定期执行容器健康检查。

  * 回传 pod 的状态到其他 kubernetes 服务中。

  * 回传 节点 的状态到其他 kubernetes 服务中。


* `kube-proxy` 实现 Kubernetes SVC 的通信与负载均衡机制的重要组件。通过在主机上维护网络规则并执行连接转发来实现Kubernetes服务抽象。



## Kubernetes 基本对象与术语


### Pod

* Pod 是一组紧密关联的容器集合，它们共享 `PID`、`IPC`、`Network` 和 `UTS namespace`，是 Kubernetes 调度的基本单位。Pod 的设计理念是支持多个容器在一个 Pod 中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。



### Label

* Label 是识别 Kubernetes 对象的标签，以 `key/value` 的方式附加到对象上（key最长不能超过63字节，value 可以为空，也可以是不超过253字节的字符串）。 Label 不提供唯一性，并且实际上经常是很多对象（如 Pods）都使用相同的 label 来标志具体的应用。 Label 定义好后其他对象可以使用 Label Selector 来选择一组相同 label 的对象（比如 Service 用 label 来选择一组 Pod）。Label Selector 支持以下几种方式:

  * 等于和不等于, 如app=nginx和env!=production

  * 集合，如env in (production, test)

  * 多个label（它们之间是AND关系），如app=nginx, env=test


### Namespace

* Namespace 是对一组资源和对象的抽象集合，比如可以用来将系统内部的对象划分为不同的项目组或用户组。常见的 pods, services, deployments 等都是属于某一个 namespace 的（默认是default），而 Node, PersistentVolumes 等则不属于任何 namespace。



### ReplicaSet

* ReplicaSet 它的主要作用是确保 `Pod` 以你指定的副本数运行,  如果有容器异常退出, 会自动创建新的 `Pod` 来替代, 而异常多出来的容器也会自动回收。 支持集合式的 `selector`。ReplicaSet 可以独立使用, 但建议使用 `Deployment` 来自动管理 ReplicaSet, 这样就无需担心跟其他机制的不兼容问题（比如 ReplicaSet 不支持 rolling-update 但 Deployment 支持）, 并且 `Deployment` 还支持版本记录、回滚、暂停升级等高级特性。



### Deployment

* Deployment 确保任意时间都有指定数量的 `Pod` 在运行。如果为某个 `Pod` 创建了 Deployment 并且指定3个副本，它会创建3个 Pod，并且持续监控它们。如果某个 Pod 不响应，那么 Deployment 会替换它，保持总数为3。如果之前不响应的 Pod 恢复了，现在就有4个 Pod 了，那么 Deployment 会将其中一个终止保持总数为3。如果在运行中将副本总数改为5，Deployment 会立刻启动2个新 Pod，保证总数为5。Deployment 还支持回滚和滚动升级。

* 创建 Deployment 需要指定:
  
  * Pod 模板 用于配置 pod 生成的属性和副本。

  * Label 标签 Deployment 需要监控的 Pod 的标签。 


### StatefulSet

* StatefulSet 是为了解决有状态服务的问题 (对应 Deployments 和 ReplicaSets是为无状态服务而设计), 其应用场景包括

  * 稳定的持久化存储，即 Pod 重新调度后还是能访问到相同的持久化数据，基于 PVC 来实现。

  * 稳定的网络标志，即 Pod 重新调度后其 `PodName` 和 `HostName` 不变，基于 Headless Service（即没有Cluster IP的Service）来实现  
 
  * 有序部署，有序扩展，即 Pod 是有顺序的，在部署或者扩展的时候要依据定义的顺序依次依次进行（即从0到N-1，在下一个Pod运行之前所有之前的 Pod 必须都是 Running 和 Ready 状态），基于 init containers来实现。

  * 有序收缩，有序删除（即从N-1到0）


### DaemonSet

* DaemonSet 保证在每个Node上都运行一个容器副本，常用来部署一些集群的日志、监控或者其他系统管理应用。


### Service

* Service 是应用服务的抽象，通过 labels 为应用提供负载均衡和服务发现。匹配 labels 的Pod IP 和端口列表组成 endpoints，由 kube-proxy 负责将服务 IP 负载均衡到这些endpoints 上。每个 Service 都会自动分配一个 cluster IP（仅在集群内部可访问的虚拟地址）和 DNS 名，其他容器可以通过该地址或 DNS 来访问服务，而不需要了解后端容器的运行。



### Job

* Job 负责批量处理短暂的一次性任务 (short lived one-off tasks)，即仅执行一次的任务，它保证批处理任务的一个或多个Pod成功结束。

* Kubernetes 支持以下几种 Job

  * 非并行 Job：通常创建一个 Pod 直至其成功结束

  * 固定结束次数的 Job：设置 `.spec.completions`, 创建多个 Pod，直到 `.spec.completions` 个 Pod 成功结束

  * 带有工作队列的并行 Job：设置 `.spec.Parallelism` 但不设置 `.spec.completions`，当所有Pod结束并且至少一个成功时，Job 就认为是成功



### CronJob

* CronJob 即定时任务，就类似于 Linux 系统的 crontab，在指定的时间周期运行指定的任务。


### Horizontal Pod Autoscaler（HPA）

* Horizontal Pod Autoscaling 可以根据 CPU 使用率或应用自定义 metrics 自动扩展 Pod 数量（支持replication controller、deployment和replica set）, 从而合理的扩展性能与使用资源。


### PersistentVolume (PV)

* PersistentVolume（PV）是集群中由管理员配置的一段网络存储。 它是集群中的资源，就像节点是集群资源一样。 PV是容量插件，如Volumes，但其生命周期独立于使用PV的任何单个pod。 此API对象捕获存储实现的详细信息，包括NFS，iSCSI或特定于云提供程序的存储系统。

* PersistentVolume 支持的类型:

  * GCEPersistentDisk

  * AWSElasticBlockStore

  * AzureFile

  * AzureDisk

  * FC (Fibre Channel)

  * Flexvolume

  * Flocker

  * NFS

  * iSCSI

  * RBD (Ceph Block Device)

  * CephFS

  * Cinder (OpenStack block storage)

  * Glusterfs

  * VsphereVolume

  * Quobyte Volumes

  * HostPath

  * Portworx Volumes

  * ScaleIO Volumes

  * StorageOS


* PersistentVolume 服务状态

  * `Available`  资源尚未被claim使用

  * `Bound`  卷已经被绑定到claim了

  * `Released`  claim被删除，卷处于释放状态，但未被集群回收。

  * `Failed`  卷自动回收失败


### PersistentVolumeClaim（PVC）

* PersistentVolumeClaim（PVC）是由用户进行存储的请求。 它类似于pod。 Pod消耗节点资源，PVC消耗PV资源。Pod可以请求特定级别的资源（CPU和内存）。声明可以请求特定的大小和访问模式（例如，可以一次读/写或多次只读）。


* PVC和PV是一一对应的。

* PVC 与 PV 的生命周期

  * PV是群集中的资源。PVC是对这些资源的请求，并且还充当对资源的检查。

  * PV和PVC之间的相互作用遵循生命周期：`Provisioning` ——-> `Binding` ——–> `Using` -——> `Releasing` -——> `Recycling`

    
* Provisioning (准备) 通过集群外的存储系统或者云平台来提供存储持久化支持。

  * 静态提供Static：集群管理员创建多个PV。 它们携带可供集群用户使用的真实存储的详细信息。 它们存在于Kubernetes API中，可用于消费。

  * 动态提供Dynamic：当管理员创建的静态PV都不匹配用户的PersistentVolumeClaim时，集群可能会尝试为PVC动态配置卷。 此配置基于StorageClasses：PVC必须请求一个类，并且管理员必须已创建并配置该类才能进行动态配置。 要求该类的声明有效地为自己禁用动态配置。



* Binding (绑定) 用户创建pvc并指定需要的资源和访问模式。在找到可用pv之前，pvc会保持未绑定状态。

* Using (使用) 用户可在pod中像volume一样使用pvc。

* Releasing (释放) 用户删除pvc来回收存储资源，pv将变成"released"状态。由于还保留着之前的数据，这些数据需要根据不同的策略来处理，否则这些存储资源无法被其他pvc使用。

* Recycling (回收)  pv 可以设置三种回收策略：保留（Retain），回收（Recycle）和删除（Delete）。

  * 保留策略 -  允许人工处理保留的数据。

  * 删除策略 -  将删除pv和外部关联的存储资源，需要插件支持。

  * 回收策略 -  将执行清除操作，之后可以被新的pvc使用，需要插件支持。



### StorageClass

* StorageClass为管理员提供了一种描述他们提供的存储的"类"的方法。 不同的类可能映射到服务质量级别，或备份策略，或者由群集管理员确定的任意策略。 Kubernetes本身对于什么类别代表是不言而喻的。 这个概念有时在其他存储系统中称为"配置文件"。




### Command、Args、Entrypoint、Cmd

* `Command`、`Args` 是属于 `kubernetes` 的命令

* `Entrypoint`、`Cmd` 是属于 `Docker` 的命令


* 在`kubernetes` 于 `Docker image` 中都分别包含命令的情况下:

  * 如果 `command` 和 `args` 均没有写, 那么用 `Docker image` 默认的配置。

  * 如果 `command` 写了, 但 `args` 没有写, 那么`Docker` 默认的配置会被忽略而且仅仅执行`kubernetes` 的 `command`（不带任何参数的）。

  * 如果 `command` 没写, 但 `args` 写了, 那么`Docker` 默认配置的 `Entrypoint` 命令行会被执行, 但是调用的参数是 `kubernetes` 的 `args`。

  * 如果 `command` 和 `args` 都写了, 那么`Docker` 默认的配置会被忽略, 而使用 `kubernetes` 的配置。








## Pod 的生命周期

* Pod对象自从其创建开始至其终止退出的时间范围称为其生命周期。

  1. 创建主容器（main container）为 `必需`的操作。

  2. 初始化容器（init container）。

  3. 容器启动后钩子（post start hook）。

  4. 容器的存活性探测（liveness probe）。

  5. 就绪性探测（readiness probe）。

  6. 容器终止前钩子（pre stop hook）

  7. 其他Pod的定义操作。

![图2][2]



### 初始化容器(init container)

* 初始化容器（init container）即应用程序的主容器启动之前要运行的容器，常用于为主容器执行一些预置操作，它们具有两
种典型特征。

  1. 初始化容器必须运行完成直至结束，若某初始化容器运行失败，那么kubernetes需要重启它直到成功完成。（注意：如果pod的`spec.restartPolicy`字段值为 `Never`，那么运行失败的初始化容器不会被重启。）

  2. 每个初始化容器都必须按定义的顺序串行运行。


### pod phase

* Pod 的 status 属性是一个 `PodStatus` 对象，拥有一个 phase 字段。它简单描述了 Pod 在其生命周期的阶段。

|pod phase|描述|
|-|-|
|挂起(Pending)|kubernetes 通过`apiserver`创建了pod 资源对象并存入etcd中, 但它尚未被调度完成, 或者仍处于从仓库下载镜像的过程中|
|运行中(Running)| Pod已经被调度至某节点，并且所有容器都已经被kubelet创建完成，至少一个容器正在运行。|
|成功(Succeeded)| Pod中的所有容器都已经成功并且不会被重启。|
|失败(Failed)| Pod中的所有容器都已终止了，并且至少有一个容器是因为失败终止。即容器以非0状态退出或者被系统禁止。|
|未知(Unknown)| `ApiServer` 无法正常获取到Pod对象的状态信息，通常是由于无法与所在工作节点的kubelet通信所致。|



### Pod conditions

* Pod 的 status 属性是一个 `PodStatus` 对象, 里面包含 PodConditions 数组，代表 Condition 是否通过。

* PodCondition 属性描述

|字段|描述|
|-|-|
|lastProbeTime|最后一次探测 Pod Condition 的时间戳。|
|lastTransitionTime|上次 Condition 从一种状态转换到另一种状态的时间。|
|message|上次 Condition 状态转换的详细描述。|
|reason|Condition 最后一次转换的原因。|
|status|Condition 状态类型，可以为 True False Unknown|
|type|Condition类型(PodScheduled, Ready, Initialized, Unschedulable, ContainersReady)|

* Condition Type 说明

  * `PodScheduled` Pod 已被调度到一个节点 

  * `Ready` Pod 能够提供请求，应该被添加到负载均衡池中以提供服务

  * `Initialized` 所有 init containers 成功启动

  * `Unschedulable` 调度器不能正常调度容器，例如缺乏资源或其他限制

  * `ContainersReady` Pod 中所有容器全部就绪


### Container probes

* Probe 是在容器上 kubelet 的定期执行的诊断，kubelet 通过调用容器实现的 Handler 来诊断。

  * `Success` 容器诊断通过。

  * `Failure` 容器诊断失败。

  * `Unknown` 诊断失败，因此不应采取任何措施。

* Handlers 包含如下三种

  * `ExecAction` 在容器内部执行指定的命令，如果命令以状态代码 0 退出，则认为诊断成功。

  * `TCPSocketAction` 对指定 IP 和端口的容器执行 TCP 检查，如果端口打开，则认为诊断成功。

  * `HTTPGetAction` 对指定 IP + port + path 路径上的容器的执行 HTTP Get 请求。如果响应的状态代码大于或等于 200 且小于 400，则认为诊断成功。


* kubelet 可以选择性地对运行中的容器进行两种探测器执行和响应。

  * `livenessProbe`  存活性探测, 探测容器是否正在运行，如果活动探测失败，则 kubelet 会杀死容器，并且容器将受其 重启策略 的约束。如果不指定活动探测, 默认状态是 Success。


  * `readinessProbe`  就绪性探测, 探测容器是否已准备好为请求提供服务，如果准备情况探测失败，则控制器会从与 Pod 匹配的所有服务的端点中删除 Pod 的 IP 地址。初始化延迟之前的默认准备状态是 `Failure`, 如果容器未提供准备情况探测，则默认状态为 Success。


* 以下为 Nginx 应用的两种探针配置示例

```shell
apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 2
  selector:
    matchLabels:
      name: nginx
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http
          # readinessProbe - 检测pod 的 Ready 是否为 true
          readinessProbe:
            tcpSocket:
              port: 80
            # 启动后5s 开始检测
            initialDelaySeconds: 5  
            # 检测 间隔为 10s
            periodSeconds: 10
          # livenessProbe - 检测 pod 的 State 是否为 Running
          livenessProbe:
            httpGet:
              path: /
              port: 80
            # 启动后 15s 开始检测
            # 检测时间必须在 readinessProbe 之后
            initialDelaySeconds: 15
            # 检测 间隔为 20s
            periodSeconds: 20
```



### Pod RestartPolicy (重启策略)

* PodSpec 中有一个`restartPolicy`字段，可能的值为Always、OnFailure和Never。默认为Always。restartPolicy适用于Pod中的所有容器。而且它仅用于控制在同一节点上重新启动Pod对象的相关容器。

* 首次需要重启的容器，将在其需要时立即进行重启，随后再次需要重启的操作将由kubelet延迟一段时间后进行，且反复的重启操作的延迟时长依次为10秒、20秒、40秒... 300秒是最大延迟时长。

* Pod，一旦绑定到一个节点，Pod对象将永远不会被重新绑定到另一个节点，它要么被重启，要么终止，直到节点发生故障或被删除。


### Pod 创建过程

* 创建一个 pod 的过程

  1. 用户通过kubectl或其他API客户端提交了Pod Spec给API Server。

  2. API Server尝试着将Pod对象的相关信息存入etcd中，待写入操作执行完成，API Server即会返回确认信息至客户端。

  3. API Server开始检测etcd中的状态变化。

  4. 所有的kubernetes组件均使用`watch`机制来跟踪检查API Server上的相关的变动。

  5. kube-scheduler（调度器）通过其`watcher`觉察到API Server创建了新的Pod对象但尚未绑定至任何工作节点。

  6. kube-scheduler（调度器）为Pod对象挑选一个工作节点并将结果信息更新至API Server。

  7. 调度结果信息由API Server更新至etcd存储系统，而且API Server也开始反映此Pod对象的调度结果。

  8. Pod被调度到的目标工作节点上的kubelet尝试在当前节点上调用Docker启动容器，并将容器的结果状态返回送至API Server。 

  9. API Server将Pod状态信息存入etcd系统中。 

  10. 在etcd确认写入操作成功完成后，API Server将确认信息发送至相关的kubelet，事件将通过它被接受。




## Kubernetes 资源调度与限制

* 在 Kubernetes 体系中，资源默认是被多租户多应用共享使用的，应用与租户间不可避免地存在资源竞争问题。

* 在 Kubernetes 中支持分别从 `Namespace`、`Pod` 和 `Container` 三个级别对资源进行管理。

* 在 Kubernetes 将 cpu  1 Core (核) = 1000m, `m` 这个单位表示 千分之一核, 2000m 表示 两个完整的核心, 也可以写成`2` 或者 `2.0`。



### ResourceQuota 

* `Namespace` 级别, 可以通过创建 `ResourceQuota` 对象对`Namespace`进行绑定, 提供一个总体资源使用量限制。

  1. 可以设置该命名空间中 Pod 可以使用到的计算资源（CPU、内存）、存储资源总量上限。

  2. 可以限制该 Namespace 中某种类型对象（如 Pod、RC、Service、Secret、ConfigMap、PVC 等）的总量上限。



* ResourceQuota 示例

```shell
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 30Gi
    requests.storage: 500Gi
    requests.ephemeral-storage: 10Gi
    limits.cpu: "40"
    limits.memory: 60Gi
    limits.ephemeral-storage: 20Gi
    pods: "10"
    services: "5"
    replicationcontrollers: "20"
    resourcequotas: "1"
    secrets: "10"
    configmaps: "10"
    persistentvolumeclaims: "10"
    services.nodeports: "50"
    services.loadbalancers: "10"

```

* `requests` kubernetes会根据Request的值去查找有足够资源的node来调度此pod, 既超过了 Request 限制的值, Pod 将不会被调度到此node中。 

* `limits`  对应资源量的上限, 既最多允许使用这个上限的资源量, 由于cpu是可压缩的, 进程是无法突破上限的, 而memory是不可压缩资源, 当进程试图请求超过limit限制时的memory, 此进程就会被kubernetes杀掉。




### LimitRange

* LimitRange 对象设置 Namespace 中 Pod 及 Container 的默认资源配额和资源限制。

```shell
apiVersion: v1
kind: LimitRange
metadata:
  name: limit
spec:
  limits:
  - type: Pod
    max:
      cpu: "10"
      memory: 100Gi
    min:
      cpu: 200m
      memory: 6Mi
    maxLimitRequestRatio:
      cpu: "2"
      memory: "4"
  - type: Container
    max:
      cpu: "2"
      memory: 1Gi
    min:
      cpu: 100m
      memory: 3Mi
    default:
      cpu: 300m
      memory: 200Mi
    defaultRequest:
      cpu: 200m
      memory: 100Mi
    maxLimitRequestRatio:
      cpu: "2"
      memory: "4"
  - type: PersistentVolumeClaim
    max:
      storage: 10Gi
    min:
      storage: 5Gi
```

* `pod` 与 `Container` 以及 `pvc` 类型可分开定义 `LimitRange` 分配资源。


* `limits` 字段下面的 `default` 字段表示每个 Pod 的默认的 limits 配置，所以任何没有分配资源的 limits 的 Pod 都会被自动分配 200Mi limits 的内存和 300m limits 的 CPU。 


* `defaultRequest` 字段表示每个 Pod 的默认 requests 配置，所以任何没有分配资源的 requests 的 Pod 都会被自动分配 100Mi requests 的内存和 200m requests 的 CPU。


* `max` 与 `min` 字段分别限制 type 类型下的 服务最大与最小的限制值。 



### ResourceRequests/ResourceLimits


* 在 Container 级别可以对两种计算资源进行管理 `CPU` 和 `内存`。

* `ResourceRequests` 表示容器希望被分配到的可完全保证的资源量，Requests 的值会被提供给 Kubernetes 调度器，以便优化基于资源请求的容器调度。

* `ResourceLimits` 表示容器能用的资源上限，这个上限的值会影响在节点上发生资源竞争时的解决策略。



```shell
apiVersion: v1
kind: Pod
metadata:
  name: busybox
spec:
  containers:
   - name: busybox
     image: busybox
     resources:
       requests:
         memory: "100Mi"
         cpu: "200m"
       limits:
         memory: "200Mi"
         cpu: "250m"
```


### kubernetes 与 Cgroup

> Kubernetes 对内存资源的限制实际上是通过 cgroup 来控制的，cgroup 是容器的一组用来控制内核如何运行进程的相关属性集合。针对内存、CPU 和各种设备都有对应的 cgroup。cgroup 是具有层级的，这意味着每个 cgroup 拥有一个它可以继承属性的父亲，往上一直直到系统启动时创建的 root cgroup。


* 内存 限制

  * Kubernetes 通过 cgroup 和 OOM killer 来限制 Pod 的内存资源，当超过内存限制值以后, Kubernetes 会选择好几个进程作为 `OOM killer` 候选人, 其中最重要的进程是标注为 `pause` 的进程, 用来为业务容器创建共享的 `network` 和 `namespace`, 其 `oom_score_adj` 值为 `-998`，可以确保不被杀死。`oom_score_adj` 值越低就越不容易被杀死, 因为业务容器内`pause` 之外的所有其他进程的 `oom_score_adj` 值都相同，所以谁的内存使用量最多，`oom_score` 值就越高，也就越容易被杀死。



* CPU 限制

  * 在 Kubernetes 中设置的 cpu requests 最终会被 cgroup 设置为 `cpu.shares` 属性的值， cpu limits 会被带宽控制组设置为`cpu.cfs_period_us` 和 `cpu.cfs_quota_us` 属性的值。与内存一样，cpu requests 主要用于在调度时通知调度器节点上至少需要多少个 cpu shares 才可以被调度。

  * cpu requests 与 内存 requests 不同，设置了 cpu requests 会在 cgroup 中设置一个属性，以确保内核会将该数量的 shares 分配给进程。


  * cpu limits 与 内存 limits 也有所不同。如果容器进程使用的内存资源超过了内存使用限制，那么该进程将会成为 oom-killing 的候选者。但是容器进程基本上永远不能超过设置的 CPU 配额，所以容器永远不会因为尝试使用比分配的更多的 CPU 时间而被驱逐。系统会在调度程序中强制进行 CPU 资源限制，以确保进程不会超过这个限制。



## Kubernetes 网络模型

* Kubernetes 中每个Pod 都拥有一个独立的IP地址，而且假定所有Pod 都在一个可以直接连通的、扁平的网络空间中，不管是否运行在同一Node上都可以通过Pod的IP来访问。

* Kubernetes 中Pod的IP是最小粒度IP。同一个Pod内所有的容器共享一个网络堆栈，该模型称为IP-per-Pod模型。


### Kubernetes 通信

* 同一个 Node 下, 同一个 pod 内

* 同一个Pod的容器共享同一个网络命名空间, 它们之间的访问可以用 Pod IP 地址 + 容器端口就可以访问。


![图3][3] 



* 同一个 Node 下, 不同的 pod 之间通信


![图4][4]




* 不同的 Node 之间, pod 与 pod 使用 网络组件进行通信

![图5][5]




## Kubernetes RBAC

* 基于角色的访问控制（Role-Based Access Control, 即 "RBAC"）: 使用 “rbac.authorization.k8s.io” API Group 实现授权决策，允许管理员通过 Kubernetes API 动态配置策略。

![图6][6]



### 角色 ( ClusterRole 与 Role )

* `Role（角色）` 是一系列权限的集合，例如一个角色可以包含读取 Pod 的权限和列出 Pod 的权限。
  * Role 只能用来给某个特定 namespace 中的资源作鉴权。



* 如下是 role 对象的定义的一个 demo 授予 default 这个 namespaces 下的 pods 的权限

```shell
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: demo-role
rules:
- apiGroups: [""]  # 空字符串""表明使用 core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list", "create", "delete"]
```



* 大多数资源由代表其名字的字符串表示，例如 ”pods”，就像它们出现在相关API endpoint 的URL中一样。然而，有一些Kubernetes API还 包含了”子资源”，比如 pod 的 logs。

* 在Kubernetes中，pod logs endpoint的URL格式为：`GET /api/v1/namespaces/{namespace}/pods/{name}/log`

```shell
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: pod-and-pod-logs-reader
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list"]
```



* 通过 `resourceNames` 列表，角色可以针对不同种类的请求根据资源名引用资源实例。

* 当指定了 `resourceNames` 列表时，不同动作 种类的请求的权限，如使用 ”get”、”delete”、”update”以及”patch”等动词的请求，将被限定到资源列表中所包含的资源实例上。

* 值得注意的是，如果设置了 resourceNames，则请求所使用的动词不能是 list、watch、create或者delete  collection。由于资源名不会出现在 create、list、watch和delete  collection 等API请求的URL中，所以这些请求动词不会被设置了resourceNames 的规则所允许，因为规则中的 resourceNames 部分不会匹配这些请求。


* 如下 如果需要限定一个角色绑定主体只能 ”get” 或者 ”update” 指定的一个 configmap

```shell
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  namespace: default
  name: configmap-updater
rules:
- apiGroups: [""]
  resources: ["configmap"]
  resourceNames: ["my-configmap"]
  verbs: ["update", "get"]

```





* `ClusterRole` 对象可以授予与 Role 对象相同的权限，但由于它们属于集群范围对象，也可以使用它们授予对以下几种资源的访问权限

  * 集群范围资源（例如节点，即 node）

  * 非资源类型 endpoint（例如 /healthz）

  * 授权多个 Namespace


* 如下是 ClusterRole 定义可用于授予用户对某一个 namespace，或者 所有 namespace 的 secret（取决于其绑定方式）的读访问权限

```shell
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  # ClusterRole 是集群范围对象，没有 "namespace" 区分
  name: demo-clusterrole
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "watch", "list", "create", "delete"]
```



### 角色绑定 ( ClusterRoleBinding 与 RoleBinding )


* `RoleBinding` 把 Role 或 ClusterRole 中定义的各种权限映射到 `User`，`Service Account` 或者 `Group`，从而让这些用户继承角色在 namespace 中的权限。



* 如下是 RoleBinding 引用在同一命名空间内定义的Role对象。

```shell
# 以下角色绑定定义将允许用户 "jane" 从 "default" 命名空间中读取pod
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: jane
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io

```


* `RoleBinding` 对象也可以引用一个 ClusterRole 对象用于在 RoleBinding 所在的命名空间内授予用户对所引用的ClusterRole 中定义的命名空间资源的访问权限。这一点允许管理员在整个集群范围内首先定义一组通用的角色，然后再在不同的命名空间中复用这些角色。



* 如下 RoleBinding 引用的是一个 ClusterRole 对象，但是用户”dave”（即角色绑定主体）还是只能读取”development” 命名空间中的 secret（即RoleBinding所在的命名空间）

```shell
# 以下角色绑定允许用户"dave"读取"development"命名空间中的secret。
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-secrets
  namespace: development # 这里表明仅授权读取"development"命名空间中的资源。
subjects:
- kind: User
  name: dave
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```




* `ClusterRoleBinding` 让用户继承 ClusterRole 在整个集群中的权限。


* 如下使用 ClusterRoleBinding 在集群级别和所有命名空间中授予权限。下面示例中所定义的 ClusterRoleBinding 允许在用户组 ”manager” 中的任何用户都可以读取集群中任何命名空间中的 secret 。

```shell
# 以下`ClusterRoleBinding`对象允许在用户组"manager"中的任何用户都可以读取集群中任何命名空间中的secret。
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: read-secrets-global
subjects:
- kind: Group
  name: manager
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: secret-reader
  apiGroup: rbac.authorization.k8s.io
```


### 默认角色 与 默认角色绑定

* API Server 会创建一组默认的 ClusterRole 和 ClusterRoleBinding 对象。这些默认对象中有许多包含 system: 前缀，表明这些资源由 Kubernetes基础组件”拥有”。对这些资源的修改可能导致非功能性集群（non-functional cluster）。

* `system:node` ClusterRole 对象。这个角色定义了 kubelet 的权限。如果这个角色被修改，可能会导致kubelet 无法正常工作。

* 所有默认的 ClusterRole 和 ClusterRoleBinding 对象都会被标记为 kubernetes.io/bootstrapping=rbac-defaults。






### 用户角色

* 通过命令 `kubectl get clusterrole` 查看到并不是所有都是以 system:前缀，它们是面向用户的角色。这些角色包含超级用户角色（cluster-admin），即旨在利用 ClusterRoleBinding（cluster-status）在集群范围内授权的角色， 以及那些使用 RoleBinding（admin、edit和view）在特定命名空间中授权的角色。

  * `cluster-admin`  超级用户权限，允许对任何资源执行任何操作。在 ClusterRoleBinding 中使用时，可以完全控制集群和所有命名空间中的所有资源。在 RoleBinding 中使用时，可以完全控制 RoleBinding 所在命名空间中的所有资源，包括命名空间自己。

  * `admin` 管理员权限，利用 RoleBinding 在某一命名空间内部授予。在 RoleBinding 中使用时，允许针对命名空间内大部分资源的读写访问， 包括在命名空间内创建角色与角色绑定的能力。但不允许对资源配额（resource quota）或者命名空间本身的写访问。

  * `edit`  允许对某一个命名空间内大部分对象的读写访问，但不允许查看或者修改角色或者角色绑定。

  * `view`  允许对某一个命名空间内大部分对象的只读访问。不允许查看角色或者角色绑定。由于可扩散性等原因，不允许查看 secret 资源。 






## API Server 认证机制

* 通常我们使用客户端 `kubectl` 来与`Kubernetes API Server` 交互, 它们之间的接口是REST调用。

* 为了确保 `Kubernetes` 集群的安全，`API Server` 都会对客户端进行身份认证，认证失败则无法调用API。

* Pod中访问`Kubernetes API Server`服务的时候，是以`Service`方式访问服务名为`kubernetes`的这个服务，而`kubernetes`服务又只在HTTPS 443上提供服务，那么如何进行身份认证呢? 

  * `Service Account Token`

* `Kubernetes` &emsp; 有两套账户系统分别是

  * 1.&emsp; `User Account` 是给用户使用的, 是全局性的, 主要用于 后端的用户数据库同步。

  * 2.&emsp; `Service Account` 是给 `Pod` 内的进程使用的, 是属于具体的 `Namespace` 的。


### Service Account

> 每个Namespace下有一个名为default的默认的Service Account对象，这个Service Account里面有一个名为Tokens的可以当作Volume一样被Mount到Podcast里的Secret,当Pod 启动时，这个Secret会自动被Mount到Pod的指定目录下，用来协助完成Pod中的进程访问API Server时的身份鉴权过程。


* `kubernetes` 开启了 `Service Account` 会在每个 `Namespace` 下面都会创建一个默认的 `default` 的`Service Account`。

```
[root@k8s-node-1 ~]# kubectl get sa --all-namespaces |grep default
default              default                                   1         13d
ingress-nginx        default                                   1         7d16h
istio-system         default                                   1         2d20h
jicki                default                                   1         2d18h
kube-system          default                                   1         13d
```



* 每个`Service Account` 下面都会拥有一个加密过的`secret` 作为`Token`。 这个`Token` 就是 `Service Account Token` 。这个 `Token` 才是真正在 `API Server` 验证(authentication)环节起作用的。


```
[root@k8s-node-1 ~]# kubectl get sa/default -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2020-04-03T10:06:15Z"
  name: default
  namespace: default
  resourceVersion: "372"
  selfLink: /api/v1/namespaces/default/serviceaccounts/default
  uid: 23344d2a-08e1-447c-a62e-73b821fcc5ce
secrets:
- name: default-token-9bcmt
```


* 当用户在该`Namespace` 下创建 `pod` 的时候都会默认使用这个`Service Account` 。


```
[root@k8s-node-1 ~]# kubectl get pods/nginx -o yaml
apiVersion: v1
kind: Pod
metadata:
 .....

spec:
  containers:
  - image: nginx:alpine
    imagePullPolicy: IfNotPresent
    name: nginx
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-9bcmt
      readOnly: true

 ....

  volumes:
  - name: default-token-9bcmt
    secret:
      defaultMode: 420
      secretName: default-token-9bcmt

```


* 查看 `secret` 的具体内容

  * `ca.crt` &emsp; 这个是`API Server`的 `CA`公钥证书, 用于`Pod` 中的 `Process` 对 `API Server` 的服务端数字证书进行校验时使用

  * `Namespace` 和 `Token` 被放到了容器内, 这样容器内就可以通过`https`的方式通过 SVC - `kubernetes` 请求访问`API Server` 。 

```
[root@k8s-node-1 ~]# kubectl get secrets default-token-9bcmt -o yaml
apiVersion: v1
data:
  ca.crt: {base64}
  namespace: ZGVmYXVsdA== 
  token: {base64}
kind: Secret

  ....

type: kubernetes.io/service-account-token
```



* `Secret`

  * `Kubernetes` 提供了 `Secret` 来处理敏感信息, 目前`Secret`的类型有3种

    * 1.&emsp; `Opaque` (default):&emsp; 任意字符串。 

    * 2.&emsp; `kubernetes.io/service-account-token`:&emsp; 作用于 `Service Account` 。

    * 3.&emsp; `kubernetes.io/dockercfg`:&emsp; 作用于`Docker registry`, 用于下载Docker镜像认证使用。 















  [1]: https://jicki.cn/img/posts/kubernetes/kubernetes.png

  [2]: https://jicki.cn/img/posts/kubernetes/pod.png

  [3]: https://jicki.cn/img/posts/kubernetes/node-pod.png

  [4]: https://jicki.cn/img/posts/kubernetes/node-pod-pod.png

  [5]: https://jicki.cn/img/posts/kubernetes/node-node-pod.gif

  [6]: https://jicki.cn/img/posts/kubernetes/rbac.png

