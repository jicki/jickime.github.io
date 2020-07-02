# KubeSphere 2.1.1 to kubernetes



# KubeSphere

> 以应用为中心的容器平台, Golang 语言开发并且完全开源。


## 什么是 KubeSphere

* KubeSphere 是一款面向云原生设计的开源项目，在目前主流容器调度平台 Kubernetes 之上构建的分布式多租户容器管理平台，提供简单易用的操作界面以及向导式操作方式，在降低用户使用容器调度平台学习成本的同时，极大降低开发、测试、运维的日常工作的复杂度。



## KubeSphere 特点

* 极简体验，向导式 UI
  *  面向开发、测试、运维友好的用户界面，向导式用户体验，降低 Kubernetes 学习成本的设计理念

  *  用户基于应用模板可以一键部署一个完整应用的所有服务，UI 提供全生命周期管理


* 业务高可靠与高可用
  * `自动弹性伸缩`：部署 (Deployment) 支持根据访问量进行动态横向伸缩和容器资源的弹性扩缩容，保证集群和容器资源的高可用  

  * `提供健康检查`：支持为容器设置健康检查探针来检查容器的健康状态，确保业务的可靠性 


* 容器化 DevOps 持续交付
  * `DevOps`：基于 Jenkins 的可视化 CI/CD 流水线编辑，无需对 Jenkins 进行配置，同时内置丰富的 CI/CD 流水线模版 

  * `Source to Image (s2i)`: 从已有的代码仓库中获取代码，并通过 s2i 自动构建镜像完成应用部署并自动推送至镜像仓库，无需编写 Dockerfile

  * `端到端的流水线设置`: 支持从仓库 (GitHub / SVN / Git)、代码编译、镜像制作、镜像安全、推送仓库、版本发布、到定时构建的端到端流水线设置

  * `安全管理`: 支持代码静态分析扫描以对 DevOps 工程中代码质量进行安全管理

  * `日志`: 日志完整记录 CI / CD 流水线运行全过程   


* 开箱即用的微服务治理
  * `灵活的微服务框架`: 基于 Istio 微服务框架提供可视化的微服务治理功能，将 Kubernetes 的服务进行更细粒度的拆分，支持无侵入的微服务治理

  * `完善的治理功能`: 支持灰度发布、熔断、流量监测、流量管控、限流、链路追踪、智能路由等完善的微服务治理功能 


* 灵活的持久化存储方案
  * 支持 GlusterFS、CephRBD、NFS 等开源存储方案，支持有状态存储

  * NeonSAN CSI 插件对接 QingStor NeonSAN，以更低时延、更加弹性、更高性能的存储，满足核心业务需求

  * QingCloud CSI 插件对接 QingCloud 云平台各种性能的块存储服务


* 灵活的网络方案支持
  * 支持 Calico、Flannel 等开源网络方案

  * 分别开发了 QingCloud 云平台负载均衡器插件 和适用于物理机部署 Kubernetes 的 负载均衡器插件 Porter

  * `商业验证的 SDN 能力`：可通过 QingCloud CNI 插件对接 QingCloud SDN，获得更安全、更高性能的网络支持  

* 多维度监控日志告警

  * KubeSphere 全监控运维功能可通过可视化界面操作，同时，开放标准接口对接企业运维系统，以统一运维入口实现集中化运维

  * `可视化秒级监控`: 秒级频率、双重维度、十六项指标立体化监控；提供服务组件监控，快速定位组件故障

  * 提供按节点、企业空间、项目等资源用量排行

  * 支持基于多租户、多维度的监控指标告警，目前告警策略支持集群节点级别和工作负载级别等两个层级

  * 提供多租户日志管理，在 KubeSphere 的日志查询系统中，不同的租户只能看到属于自己的日志信息



## KubeSphere 架构

![图片2][2]




![图片1][1]



|组件|功能说明|
|-|-|
|ks-account|提供用户、权限管理相关的 API|
|ks-apiserver|整个集群管理的 API 接口和集群内部各个模块之间通信的枢纽，以及集群安全控制|
|ks-apigateway|负责处理服务请求和处理 API 调用过程中的所有任务|
|ks-console|提供 KubeSphere 的控制台服务|
|ks-controller-manager|实现业务逻辑的，例如创建企业空间时，为其创建对应的权限；或创建服务策略时，生成对应的 Istio 配置等|
|Metrics-server|Kubernetes 的监控组件，从每个节点的 Kubelet 采集指标信息|
|Prometheus|提供集群、节点、工作负载、API 对象等相关监控数据与服务|
|Elasticsearch|提供集群的日志索引、查询、数据管理等服务，在安装时也可对接您已有的 ES 减少资源消耗|
|Fluent Bit|提供日志接收与转发，可将采集到的⽇志信息发送到 ElasticSearch、Kafka|
|Jenkins|提供 CI/CD 流水线服务|
|SonarQube|可选安装项，提供代码静态检查与质量分析|
|Source-to-Image|将源代码自动将编译并打包成 Docker 镜像，方便快速构建镜像|
|Istio|提供微服务治理与流量管控，如灰度发布、金丝雀发布、熔断、流量镜像等|
|Jaeger|收集 Sidecar 数据，提供分布式 Tracing 服务|
|OpenPitrix|提供应用模板、应用部署与管理的服务|
|Alert|提供集群、Workload、Pod、容器级别的自定义告警服务|
|Notification|通用的通知服务，目前支持邮件通知|
|redis|将 ks-console 与 ks-account 的数据存储在内存中的存储系统|
|MySQL|集群后端组件的数据库，监控、告警、DevOps、OpenPitrix 共用 MySQL 服务|
|PostgreSQL|SonarQube 和 Harbor 的后端数据库|
|OpenLDAP|负责集中存储和管理用户账号信息与对接外部的 LDAP|
|存储|内置 CSI 插件对接云平台存储服务，可选安装开源的 NFS/Ceph/Gluster 的客户端|
|网络|可选安装 Calico/Flannel 等开源的网络插件，支持对接云平台 SDN|


## KubeSphere 部署

> KubeSphere 支持独立部署与 Linux 主机中, 也支持直接部署于 Kubernetes 集群中。


* 本文中将`KubeSphere` 部署于 Kubernetes 集群中。

  * 集群需求:

    * Kubernetes 版本: `1.15.x`、`1.16.x`、`1.17.x`；

    * Helm版本： 2.10.0 ≤ Helm Version ＜ 3.0.0； 关于 helm 的tiller init 安装, 使用国内的镜像 `helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.16.5 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts`

    * 集群已有默认的存储类型（StorageClass）。 必须要有默认的 StorageClass, 可以安装 nfs,gfs,cephfs, openobs 等。
    * CSR signing 功能在 kube-apiserver 中被激活。

    * 集群能够访问外网。 


### Kubernetes 集群信息

|HostName|IP|Docker|Kernel|kuberneres|作用|
|-|-|-|-|-|-|
|k8s-node-1|10.18.77.61|19.03.6-ce|4.14.173|1.18.0|Master节点|
|k8s-node-2|10.18.77.117|19.03.6-ce|4.14.173|1.18.0|Master节点 or Node节点|
|k8s-node-3|10.18.77.218|19.03.6-ce|4.14.173|1.18.0|Master节点 or Node节点 or nfs Server|



### NFS StorageClass


* NFS Server 节点安装服务

```
[root@k8s-node-3 opt]# yum -y install nfs-utils rpcbind
```


* 所有 K8S 节点都需要安装 `nfs-utils` 依赖, 否则无法挂载


```
yum -y install nfs-utils

```



* 配置 NFS 目录与权限


```
# 添加 nfs 存储目录
[root@k8s-node-3 opt]# mkdir -p /opt/nfsdata


# 增加权限以及目录到 nfs
[root@k8s-node-3 opt]# vi /etc/exports

增加

/opt/nfsdata   10.18.77.0/24(rw,sync,no_root_squash)
```



* 启动 NFS 服务

```
systemctl enable rpcbind.service    
systemctl enable nfs-server.service

systemctl start rpcbind.service    
systemctl start nfs-server.service


# 查看信息
[root@k8s-node-3 opt]# showmount -e 10.18.77.218
Export list for 10.18.77.218:
/opt/nfsdata 10.18.77.0/24
```



* 部署 `StorageClass`

  * 此版本为古老的版本, k8s 新版的存储驱动必须符合 CSI 接口实现。

  * 官方有提供一个基于 CSI 接口的 `https://github.com/kubernetes-csi/csi-driver-nfs`  但是目前并不支持 `StorageClass`  


```
# 下载 yaml 文件

for file in class.yaml deployment.yaml rbac.yaml;
do 
  wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/$file;
done

```


* 修改 deployment.yaml 文件


```
# vi deployment.yaml

# 修改如下部分

          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 10.18.77.218
            - name: NFS_PATH
              value: /opt/nfsdata
      volumes:
        - name: nfs-client-root
          nfs:
            server: 10.18.77.218
            path: /opt/nfsdata
```




* 创建服务

```
[root@k8s-node-1 nfs]# kubectl apply -f .
storageclass.storage.k8s.io/managed-nfs-storage created
deployment.apps/nfs-client-provisioner created
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
```



* 查看服务

```
[root@k8s-node-1 nfs]# kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-576d645cbc-6546h   1/1     Running   0          59s
```


```
[root@k8s-node-1 nfs]# kubectl get storageclass
NAME                  PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   fuseim.pri/ifs   Delete          Immediate           false                  74s

```


* 标记一个默认的`StorageClass`

  * 集群内只能有一个 `StorageClass` 能够标记为默认。

```
# 注意 managed-nfs-storage 为 storageclass的名称

kubectl patch storageclass managed-nfs-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 输出:
storageclass.storage.k8s.io/managed-nfs-storage patched
```


```
# 再次查看 storageclass

[root@k8s-node-1 nfs]# kubectl get storageclass
NAME                            PROVISIONER      RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage (default)   fuseim.pri/ifs   Delete          Immediate           false                  6m19s
```


### 部署 KubeSphere


* 官方提供 `helm` 与 `yaml` 文件导入.


* 集群可用的资源符合 CPU > 1 Core，可用内存 > 2 G 。使用 minimal 版本

  * minimal 版本只开启了最基本的KubeSphere组件。

```
kubectl apply -f https://raw.githubusercontent.com/kubesphere/ks-installer/master/kubesphere-minimal.yaml
```


* 集群可用的资源符合 CPU ≥ 8 Core，可用内存 ≥ 16 G。使用完整版本

  * 完整的版本, 开启了所有的 KubeSphere组件。

```
kubectl apply -f https://raw.githubusercontent.com/kubesphere/ks-installer/master/kubesphere-complete-setup.yaml
```


* 查看安装过程

  * `ks-installer` 可以看到是利用 `ansible` 安装的。


```
kubectl logs -n kubesphere-system $(kubectl get pod -n kubesphere-system -l app=ks-install -o jsonpath='{.items[0].metadata.name}') -f

```


* 查看服务

```
[root@localhost kubesphere]# kubectl get pods,svc -n kubesphere-system
NAME                                         READY   STATUS    RESTARTS   AGE
pod/ks-account-596657f8c6-dbsm4              1/1     Running   0          31m
pod/ks-apigateway-78bcdc8ffc-jbghc           1/1     Running   0          31m
pod/ks-apiserver-5b548d7c5c-jzxnt            1/1     Running   0          31m
pod/ks-console-78bcf96dbf-2cdtx              1/1     Running   0          31m
pod/ks-controller-manager-696986f8d9-v5lzp   1/1     Running   0          31m
pod/ks-installer-75b8d89dff-f2hln            1/1     Running   0          32m
pod/openldap-0                               1/1     Running   0          31m
pod/redis-6fd6c6d6f9-mprdb                   1/1     Running   0          31m

NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/ks-account      ClusterIP   10.254.43.12    <none>        80/TCP     31m
service/ks-apigateway   ClusterIP   10.254.56.201   <none>        80/TCP     31m
service/ks-apiserver    ClusterIP   10.254.25.243   <none>        80/TCP     31m
service/ks-console      ClusterIP   10.254.58.101   <none>        80/TCP     31m
service/openldap        ClusterIP   None            <none>        389/TCP    31m
service/redis           ClusterIP   10.254.38.34    <none>        6379/TCP   31m
```


* 修改默认访问端口

  * 默认配置, 使用 `nodeport` 映射30880 端口。

  * 修改为 ingress 用域名访问

```
# 编辑 ks-console service
kubectl edit svc/ks-console -n kubesphere-system


# 去掉

    nodePort: 30880

    type: NodePort

```


* 配置域名访问

  * 配置ssl (可选)

```
# 使用自签的ssl证书

kubectl -n kubesphere-system create \
  secret tls tls-kubesphere-ingress \
  --cert=3258931_kubesphere.jicki.cn.pem \
  --key=3258931_kubesphere.jicki.cn.key

```


```
# 编写 kubesphere-ingress.yaml 文件

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ks-console-ingress
  namespace: kubesphere-system
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"
spec:
  rules:
  - host: kubesphere.jicki.cn
    http:
      paths:
      - backend:
          serviceName: ks-console
          servicePort: 80
  tls:
  - hosts:
    - kubesphere.jicki.cn
    secretName: tls-kubesphere-ingress
```

* 查看服务

```
[root@localhost kubesphere]# kubectl get ingress -n kubesphere-system
NAME                 HOSTS                 ADDRESS   PORTS     AGE
ks-console-ingress   kubesphere.jicki.cn             80, 443   14m
```


### 访问 KubeSphere

>
> 默认账号密码 用户名: admin  密码: P@88w0rd
>



![图3][3]


![图4][4]


![图5][5]


![图6][6]






  [1]: https://jicki.cn/img/posts/kubesphere/kubesphere.png
  [2]: https://jicki.cn/img/posts/kubesphere/kubesphere-1.png
  [3]: https://jicki.cn/img/posts/kubesphere/kubesphere-2.png
  [4]: https://jicki.cn/img/posts/kubesphere/kubesphere-3.png
  [5]: https://jicki.cn/img/posts/kubesphere/kubesphere-4.png
  [6]: https://jicki.cn/img/posts/kubesphere/kubesphere-5.png

