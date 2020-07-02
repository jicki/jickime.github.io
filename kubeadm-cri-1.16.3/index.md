# kubeadm cri to 1.16.3


> kubeadm cri to 1.16.3


# kubernetes 1.16.3

> 本文基于 kubeadm 方式部署，kubeadm 在1.13 版本以后正式进入 GA.
>
> 目前国内各大厂商都有 kubeadm 的镜像源，对于部署 kubernetes 来说是大大的便利.
>
> 从官方对 kubeadm 的更新频繁度来看，kubeadm 应该是后面的趋势，毕竟二进制部署确实麻烦了点.



# 1. 环境说明


|系统|IP|Containerd|Kernel|作用|
|-|-|-|-|-|
|CentOS 7 x64|172.16.0.3|18.09.6|4.4.205|K8s-Master|
|CentOS 7 x64|172.16.0.10|18.09.6|4.4.205|K8s-Node|


## 1.1 初始化环境

### 1.1.1 配置 hosts

```
hostnamectl --static set-hostname hostname

k8s-node-1  172.16.0.3
k8s-node-2  172.16.0.10
```



```
#编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts

172.16.0.3 k8s-node-1
172.16.0.10 k8s-node-2
```


### 1.1.2 关闭防火墙

```
sed -ri 's#(SELINUX=).*#\1disabled#' /etc/selinux/config
setenforce 0
systemctl disable firewalld
systemctl stop firewalld

```

### 1.1.3 关闭虚拟内存

```
# 临时关闭

swapoff -a

```

```
# 永久关闭

vi /etc/fstab 

注释掉关于 swap 的一段

```



### 1.1.4 添加内核配置

```
vi /etc/sysctl.conf

net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
vm.swappiness=0


# 生效配置

sysctl -p

```


### 1.1.5 配置IPVS模块

> kube-proxy 使用 ipvs 方式负载 ，所以需要内核加载 ipvs 模块, 否则只会使用 iptables 方式


```
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
#!/bin/bash
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF



# 授权
chmod 755 /etc/sysconfig/modules/ipvs.modules 


# 加载模块
bash /etc/sysconfig/modules/ipvs.modules


# 查看加载
lsmod | grep -e ip_vs -e nf_conntrack_ipv4

# 输出如下:
-----------------------------------------------------------------------
nf_conntrack_ipv4      20480  0 
nf_defrag_ipv4         16384  1 nf_conntrack_ipv4
ip_vs_sh               16384  0 
ip_vs_wrr              16384  0 
ip_vs_rr               16384  0 
ip_vs                 147456  6 ip_vs_rr,ip_vs_sh,ip_vs_wrr
nf_conntrack          110592  2 ip_vs,nf_conntrack_ipv4
libcrc32c              16384  2 xfs,ip_vs
-----------------------------------------------------------------------
```


### 1.1.6 配置yum源

> 使用 阿里 的 yum 源


```

cat << EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF




# 更新 yum

yum makecache
```



# 2. 安装 crictl和containerd

## 安装 containerd

```shell
# 安装 yum-config-manager

yum -y install yum-utils

# 导入
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo

# 安装 containerd.io
yum update && yum install containerd.io
```


```shell

# 导出完整配置文件
mv /etc/containerd/config.toml /etc/containerd/config.toml-bak

containerd config default > /etc/containerd/config.toml


# 修改配置文件
vi /etc/containerd/config.toml

# 按需修改如下配置

root = "/opt/containerd"

sandbox_image = "registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.1"

    [plugins.cri.registry]
      [plugins.cri.registry.mirrors]
        [plugins.cri.registry.mirrors."docker.io"]
          endpoint = ["https://registry-1.docker.io"]
        [plugins.cri.registry.mirrors."daocloud.io"]
          endpoint = ["http://b438f72b.m.daocloud.io"]
```

```shell
# 修改 systemctl 文件，配置 modprobe 加载

vi /usr/lib/systemd/system/containerd.service

在
ExecStartPre=-/sbin/modprobe overlay

下面添加:

ExecStartPre=-/sbin/modprobe br_netfilter

```



```shell
systemctl enable containerd
systemctl start containerd
systemctl status containerd

```



## 配置 crictl

```shell
# 配置文件

cat <<EOF > /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

```

### 测试安装


```shell
# 测试拉取镜像

crictl pull nginx:alpine

Image is up to date for sha256:a624d888d69ffdc185ed3b9c9c0645e8eaaac843ce59e89f1fbe45b0581e4ef6

# 查看

crictl images

IMAGE                     TAG                 IMAGE ID            SIZE
docker.io/library/nginx   alpine              a624d888d69ff       8.78MB

```

# 3. 部署 kubernetes 


## 3.1 安装相关软件

> 所有软件安装都通过 yum 安装


```

# kubernetes 相关 (Master)
yum -y install kubelet-1.16.3 kubeadm-1.16.3 kubectl-1.16.3

# kubernetes 相关 (Node)
yum -y install kubelet-1.16.3 kubeadm-1.16.3

# ipvs 相关

yum -y install ipvsadm ipset

```

```
# 配置 kubelet 自动启动 (暂时不需要启动)

systemctl enable kubelet.service

```

* 修改源码, 增加证书 10年期限

```
# 下载源码

git clone https://github.com/kubernetes/kubernetes

Cloning into 'kubernetes'...
remote: Enumerating objects: 219, done.
remote: Counting objects: 100% (219/219), done.
remote: Compressing objects: 100% (128/128), done.
remote: Total 1087208 (delta 112), reused 91 (delta 91), pack-reused 1086989
Receiving objects: 100% (1087208/1087208), 668.66 MiB | 486.00 KiB/s, done.
Resolving deltas: 100% (777513/777513), done.


```


```
# 查看分支
cd kubernetes

git branch -a

```

```
查看当前的分支
git branch


```

```
# 切换到相关的分支
git checkout remotes/origin/release-1.16

```


* 修改 cert.go 文件

```
# 打开文件
vi staging/src/k8s.io/client-go/util/cert/cert.go

# 如下 默认已经是10年,可不修改,也可以修改99年,但是不能超过100年

NotAfter:              now.Add(duration365d * 10).UTC(),


```

* 修改 constants.go 文件

```
# 打开文件
vi cmd/kubeadm/app/constants/constants.go

# 如下 默认是 1年, 修改为 10 年 

CertificateValidity = time.Hour * 24 * 365

# 修改为

CertificateValidity = time.Hour * 24 * 365 * 10
```


* 重新编译 kubeadm


```
make all WHAT=cmd/kubeadm GOFLAGS=-v

```

* 拷贝 覆盖 kubeadm


```
# 编译后生成目录为 _output/local/bin/linux/amd64

cp _output/local/bin/linux/amd64/kubeadm /usr/bin/kubeadm


cp: overwrite ‘/usr/bin/kubeadm’? y

```


## 3.3 修改 kubeadm 配置信息



```
# 导出 配置 信息

kubeadm config print init-defaults > kubeadm-init.yaml

```



```shell
# 修改相关配置，本文配置信息如下
# 这里特别要说明 criSocket: /run/containerd/containerd.sock


apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 172.16.0.3
  bindPort: 6443
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  name: k8s-node-1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "172.16.0.3:6443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.16.3
networking:
  dnsDomain: cluster.local
  podSubnet: 10.254.64.0/18
  serviceSubnet: 10.254.0.0/18
scheduler: {}
---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"


```


## 3.4 初始化集群 


```
kubeadm init --config kubeadm-init.yaml

```


```
# 输出如下:

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of control-plane nodes by copying certificate authorities 
and service account keys on each node and then running the following as root:

  kubeadm join 172.16.0.3:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:02260b45ddd43ab04ee54b3d77f9677a6eb7166b5501acf3ee61f069eceb626e \
    --control-plane       

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 172.16.0.3:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:02260b45ddd43ab04ee54b3d77f9677a6eb7166b5501acf3ee61f069eceb626e 

```


```
# 拷贝权限文件

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

```


```
# 查看集群状态(1.16 版本 AGE 都显示 unknown )

[root@k8s-node-1 yaml]# kubectl get cs
NAME                 AGE
scheduler            <unknown>
controller-manager   <unknown>
etcd-0               <unknown>

```


> 至此，集群初始化完成,  Master 部署完成。


## 3.5 加入 kubernetes 集群

> 如上有 kubeadm init 后有两条 kubeadm join 命令,  --experimental-control-plane 为 加入 Master
>
> 另外token 有时效性，如果提示 token 失效，请自行创建一个新的 token.
>
> kubeadm token create --print-join-command    创建新的 join token


### 3.5.1 验证 Master 节点


> 这里 STATUS 显示 NotReady 是因为 没有安装网络组件


```
# 查看 node

[root@k8s-node-1 yaml]# kubectl get nodes
NAME         STATUS     ROLES    AGE     VERSION
k8s-node-1   NotReady   master   4m41s   v1.16.3

```

## 3.6 部署 Node 节点


```
kubeadm join 172.16.0.3:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:02260b45ddd43ab04ee54b3d77f9677a6eb7166b5501acf3ee61f069eceb626e

```



```
# 输出如下:

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.


```

### 3.6.1 验证 所有 节点

> 这里 STATUS 显示 NotReady 是因为 没有安装网络组件

```
[root@k8s-node-1 yaml]# kubectl get nodes
NAME         STATUS     ROLES    AGE     VERSION
k8s-node-1   NotReady   master   4m41s   v1.16.3
k8s-node-2   NotReady   <none>   2m46s   v1.16.3

```


* 查看证书


```
# 更新证书
# kubeadm alpha certs renew all

# 查看证书时间
kubeadm alpha certs check-expiration

```









## 3.7 安装网络组件


> Calico 网络
>
> 官方文档 https://docs.projectcalico.org/v3.10/introduction


### 3.7.1  下载 Calico yaml

```
# 下载 yaml 文件

wget https://docs.projectcalico.org/v3.10/manifests/calico.yaml


```


### 3.7.2  修改 Calico 配置

> 这里只需要修改 分配的 CIDR 就可以

```
vi calico.yaml

# 修改 pods 分配的 IP 段

            - name: CALICO_IPV4POOL_CIDR
              value: "10.254.64.0/18"
              
```



```
# 导入 yaml 文件

[root@k8s-node-1 calico]# kubectl apply -f calico.yaml 
configmap/calico-config unchanged
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org unchanged
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org unchanged
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers unchanged
clusterrole.rbac.authorization.k8s.io/calico-node unchanged
clusterrolebinding.rbac.authorization.k8s.io/calico-node unchanged
daemonset.apps/calico-node configured
serviceaccount/calico-node unchanged
deployment.apps/calico-kube-controllers unchanged
serviceaccount/calico-kube-controllers unchanged


# 查看服务

[root@k8s-node-1 calico]# kubectl get pods -n kube-system |grep calico
calico-kube-controllers-6b64bcd855-r2llv   1/1     Running   0          18m
calico-node-2jgz4                          1/1     Running   0          18m
calico-node-sblfx                          1/1     Running   0          18m


[root@k8s-node-1 calico]# crictl ps |grep calico
73dd289b349f6       8f87d09ab8119       18 minutes ago      Running             calico-kube-controllers   0                   fd39d95666bd1
502550c1b5d0f       4a88ba569c297       18 minutes ago      Running             calico-node               0                   e9801794bb446

```



## 3.8 检验整体集群



### 3.8.1 查看 状态

> 所有的 STATUS 都为 Ready 
>


```

[root@k8s-node-1 calico]# kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-node-1   Ready    master   51m   v1.16.3
k8s-node-2   Ready    <none>   49m   v1.16.3

```



* 查看 etcd 状态


```
# 这里目前只有一个 etcd 节点,多个节点 就写多个就可以
export ETCDCTL_API=3


# 1
etcdctl -w table \
   --endpoints=https://172.16.0.3:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   endpoint status




# 2
etcdctl -w table \
   --endpoints=https://172.16.0.3:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   endpoint health




# 3
etcdctl -w table \
   --endpoints=https://172.16.0.3:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   member list

```






### 3.8.2 查看 pods 状态


```
[root@k8s-node-1 calico]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-6b64bcd855-r2llv   1/1     Running   0          21m
kube-system   calico-node-2jgz4                          1/1     Running   0          21m
kube-system   calico-node-sblfx                          1/1     Running   0          21m
kube-system   coredns-67c766df46-6zk8j                   1/1     Running   0          52m
kube-system   coredns-67c766df46-rpwd5                   1/1     Running   0          52m
kube-system   etcd-k8s-node-1                            1/1     Running   1          51m
kube-system   kube-apiserver-k8s-node-1                  1/1     Running   1          51m
kube-system   kube-controller-manager-k8s-node-1         1/1     Running   0          51m
kube-system   kube-proxy-tshbd                           1/1     Running   0          52m
kube-system   kube-proxy-vsn2l                           1/1     Running   0          50m
kube-system   kube-scheduler-k8s-node-1                  1/1     Running   1          51m

```


### 3.8.3 查看 svc 的状态


```
[root@k8s-node-1 calico]# kubectl get svc --all-namespaces
NAMESPACE     NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.254.0.1    <none>        443/TCP                  53m
kube-system   kube-dns     ClusterIP   10.254.0.10   <none>        53/UDP,53/TCP,9153/TCP   53m

```


### 3.8.3 查看 IPVS 的状态


```
[root@k8s-node-1 calico]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 172.16.0.3:6443              Masq    1      5          0         
TCP  10.254.0.10:53 rr
  -> 10.254.69.130:53             Masq    1      0          0         
  -> 10.254.69.131:53             Masq    1      0          0         
TCP  10.254.0.10:9153 rr
  -> 10.254.69.130:9153           Masq    1      0          0         
  -> 10.254.69.131:9153           Masq    1      0          0         
UDP  10.254.0.10:53 rr
  -> 10.254.69.130:53             Masq    1      0          0         
  -> 10.254.69.131:53             Masq    1      0          0 


```

# 4. 测试集群


## 4.1 创建一个 nginx deployment

```

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
          volumeMounts:
            - name: tz-config
              mountPath: /etc/localtime
              readOnly: true
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
      volumes:
        - name: tz-config
          hostPath:
            path: /etc/localtime
---

apiVersion: v1 
kind: Service
metadata: 
  name: nginx-svc 
spec: 
  ports: 
    - port: 80
      name: http
      targetPort: 80
      protocol: TCP 
  selector: 
    name: nginx

```


```
# 导入文件

[root@k8s-node-1 yaml]# kubectl apply -f nginx-deployment.yaml 
deployment.apps/nginx-dm created
service/nginx-svc created

```




```
# 查看服务

[root@k8s-node-1 yaml]# kubectl get pods -o wide
NAME                       READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
nginx-dm-9947b45c5-7wvlq   1/1     Running   0          6m25s   10.254.76.66   k8s-node-2   <none>           <none>
nginx-dm-9947b45c5-dlbv8   1/1     Running   0          6m25s   10.254.76.65   k8s-node-2   <none>           <none>


# 查看 svc
[root@k8s-node-1 yaml]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE     SELECTOR
kubernetes   ClusterIP   10.254.0.1      <none>        443/TCP   61m     <none>
nginx-svc    ClusterIP   10.254.52.255   <none>        80/TCP    6m42s   name=nginx


# 查看node-2 服务

[root@k8s-node-2 ~]# crictl ps |grep nginx
0209fb1b4d88b       a624d888d69ff       About a minute ago   Running             nginx               0                   735222980d8a8
f6016fae8ced0       a624d888d69ff       About a minute ago   Running             nginx               0                   500132de59fb2

```


```
#  node-1 访问 svc

[root@k8s-node-1 yaml]# curl 10.254.52.255
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>


```

```
# node-2 访问 svc

[root@k8s-node-2 ~]# curl 10.254.52.255
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```



```
# 查看 ipvs 规则

[root@k8s-node-1 yaml]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 172.16.0.3:6443              Masq    1      5          0         
TCP  10.254.0.10:53 rr
  -> 10.254.69.130:53             Masq    1      0          0         
  -> 10.254.69.131:53             Masq    1      0          0         
TCP  10.254.0.10:9153 rr
  -> 10.254.69.130:9153           Masq    1      0          0         
  -> 10.254.69.131:9153           Masq    1      0          0         
TCP  10.254.52.255:80 rr
  -> 10.254.76.65:80              Masq    1      0          1         
  -> 10.254.76.66:80              Masq    1      0          0         
UDP  10.254.0.10:53 rr
  -> 10.254.69.130:53             Masq    1      0          0         
  -> 10.254.69.131:53             Masq    1      0          0  

```



## 4.2 验证 dns 的服务


```
# 创建一个 pod

apiVersion: v1
kind: Pod
metadata:
  name: alpine
spec:
  containers:
  - name: alpine
    image: alpine
    command:
    - sleep
    - "3600"

```


```
# 查看 创建的服务

[root@k8s-node-1 yaml]# kubectl get pods
NAME                       READY   STATUS    RESTARTS   AGE
alpine                     1/1     Running   0          47s
nginx-dm-9947b45c5-7wvlq   1/1     Running   0          10m
nginx-dm-9947b45c5-dlbv8   1/1     Running   0          10m

```



```
# 测试

# kubernetes 服务

[root@k8s-node-1 yaml]# kubectl exec -it alpine nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.254.0.1 kubernetes.default.svc.cluster.local




# nginx-svc 服务

[root@k8s-node-1 yaml]# kubectl exec -it alpine nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.52.255 nginx-svc.default.svc.cluster.local

```






# 5. 部署 Metrics-Server

> 官方 https://github.com/kubernetes-incubator/metrics-server


## 5.1  Metrics-Server 说明

> v1.11 以后不再支持通过 heaspter 采集监控数据，支持新的监控数据采集组件metrics-server，比heaspter轻量很多，也不做数据的持久化存储，提供实时的监控数据查询。



### 5.1.1 创建 Metrics-Server 文件


```
# vi metrics-server.yaml

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:aggregated-metrics-reader
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-server:system:auth-delegator
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: metrics-server-auth-reader
  namespace: kube-system
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: extension-apiserver-authentication-reader
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: system:metrics-server
rules:
- apiGroups:
  - ""
  resources:
  - pods
  - nodes
  - nodes/stats
  - namespaces
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:metrics-server
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:metrics-server
subjects:
- kind: ServiceAccount
  name: metrics-server
  namespace: kube-system
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-server
  namespace: kube-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    k8s-app: metrics-server
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  template:
    metadata:
      name: metrics-server
      labels:
        k8s-app: metrics-server
    spec:
      serviceAccountName: metrics-server
      volumes:
      # mount in tmp so we can safely use from-scratch images and/or read-only containers
      - name: tmp-dir
        emptyDir: {}
      containers:
      - name: metrics-server
        image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.6
        args:
          - --cert-dir=/tmp
          - --secure-port=4443
        ports:
        - name: main-port
          containerPort: 4443
          protocol: TCP
        securityContext:
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
        imagePullPolicy: Always
        volumeMounts:
        - name: tmp-dir
          mountPath: /tmp
        command:
          - /metrics-server
          - --kubelet-insecure-tls
          - --kubelet-preferred-address-types=InternalIP
      nodeSelector:
        beta.kubernetes.io/os: linux
---
apiVersion: v1
kind: Service
metadata:
  name: metrics-server
  namespace: kube-system
  labels:
    kubernetes.io/name: "Metrics-server"
    kubernetes.io/cluster-service: "true"
spec:
  selector:
    k8s-app: metrics-server
  ports:
  - port: 443
    protocol: TCP
    targetPort: main-port


```


```
# 导入服务

[root@k8s-node-1 metrics]# kubectl apply -f metrics-server.yaml
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
serviceaccount/metrics-server created
deployment.apps/metrics-server created
service/metrics-server created

```


### 5.1.2 查看服务


```
[root@k8s-node-1 metrics]# kubectl get pods -n kube-system |grep metrics
metrics-server-7b5b7fd65-v8sqc             1/1     Running   0          11s

```


### 5.1.3 测试采集

* 提示 error: metrics not available yet , 请等待一会采集后再查询

```
[root@k8s-node-1 metrics]# kubectl top node
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-node-1   139m         6%     1647Mi          44%       
k8s-node-2   71m          3%     810Mi           21% 

```

# 6. Nginx Ingress

> 官方地址 https://kubernetes.github.io/ingress-nginx/


## 6.1 Nginx Ingress 介绍

> 基于 Nginx 使用 Kubernetes ConfigMap 来存储 Nginx 配置文件



## 6.2 部署 Nginx ingress


### 6.2.1 下载 yaml 文件

```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml

```


### 6.2.2 修改 yaml 文件

```
# 替换 阿里 镜像下载地址

sed -i 's/quay\.io\/kubernetes-ingress-controller/registry\.cn-hangzhou\.aliyuncs\.com\/google_containers/g' mandatory.yaml

```

```
# 配置 node affinity  与 hostNetwork

# 在 如下之间添加
    spec:
      serviceAccountName: nginx-ingress-serviceaccount


# 添加完如下:
    spec:
      hostNetwork: true
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - k8s-node-1
                - k8s-node-2
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app.kubernetes.io/name
                    operator: In
                    values: 
                    - ingress-nginx
              topologyKey: "kubernetes.io/hostname"
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      serviceAccountName: nginx-ingress-serviceaccount

```

```
# 如上 affinity 说明

      affinity:  # 声明 亲和性设置
        nodeAffinity: # 声明 为 Node 亲和性设置
          requiredDuringSchedulingIgnoredDuringExecution:  # 必须满足下面条件
            nodeSelectorTerms: # 声明 为 Node 调度选择标签
            - matchExpressions: # 设置node拥有的标签
              - key: kubernetes.io/hostname  #  kubernetes内置标签
                operator: In   # 操作符
                values:        # 值,既集群 node 名称
                - k8s-node-1
                - k8s-node-2
        podAntiAffinity:  # 声明 为 Pod 亲和性设置
          requiredDuringSchedulingIgnoredDuringExecution:  # 必须满足下面条件
            - labelSelector:  # 与哪个pod有亲和性，在此设置此pod具有的标签
                matchExpressions:  # 要匹配如下的pod的,标签定义
                  - key: app.kubernetes.io/name  # 标签定义为 空间名称(namespaces)
                    operator: In
                    values:              
                    - ingress-nginx
              topologyKey: "kubernetes.io/hostname"    # 节点所属拓朴域
      tolerations:    # 声明 为 可容忍 的选项
      - key: node-role.kubernetes.io/master    # 声明 标签为 node-role 选项
        effect: NoSchedule                     # 声明 node-role 为 NoSchedule 也可容忍
      serviceAccountName: nginx-ingress-serviceaccount

```






### 6.2.3  apply 导入 文件

```
[root@k8s-node-1 ingress]# kubectl apply -f mandatory.yaml
namespace/ingress-nginx created
configmap/nginx-configuration created
configmap/tcp-services created
configmap/udp-services created
serviceaccount/nginx-ingress-serviceaccount created
clusterrole.rbac.authorization.k8s.io/nginx-ingress-clusterrole created
role.rbac.authorization.k8s.io/nginx-ingress-role created
rolebinding.rbac.authorization.k8s.io/nginx-ingress-role-nisa-binding created
clusterrolebinding.rbac.authorization.k8s.io/nginx-ingress-clusterrole-nisa-binding created
deployment.apps/nginx-ingress-controller created

```


### 6.2.4 查看服务状态


```
[root@k8s-node-1 ingress]# kubectl get pods -n ingress-nginx -o wide
NAME                                        READY   STATUS    RESTARTS   AGE   IP            NODE         NOMINATED NODE   READINESS GATES
nginx-ingress-controller-7ff768bcfd-5q9ns   1/1     Running   0          95s   172.16.0.3    k8s-node-1   <none>           <none>
nginx-ingress-controller-7ff768bcfd-zxmv2   1/1     Running   0          96s   172.16.0.10   k8s-node-2   <none>           <none>
```



### 6.2.5 测试 ingress



```
# 查看之前创建的 Nginx

[root@k8s-node-1 ingress]# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.254.0.1      <none>        443/TCP   74m
nginx-svc    ClusterIP   10.254.52.255   <none>        80/TCP    19m

```

```
# 创建一个 nginx-svc 的 ingress


vi nginx-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx-ingress
spec:
  rules:
  - host: nginx.jicki.cn
    http:
      paths:
      - backend:
          serviceName: nginx-svc
          servicePort: 80

```




```
# 导入 yaml
[root@k8s-node-1 yaml]# kubectl apply -f nginx-ingress.yaml 
ingress.extensions/nginx-ingress created



# 查看 ingress

[root@k8s-node-1 yaml]# kubectl get ingress
NAME            HOSTS            ADDRESS   PORTS   AGE
nginx-ingress   nginx.jicki.cn             80      17s



```




### 6.2.6 测试访问


```
[root@k8s-node-1 yaml]# curl -I nginx.jicki.cn
HTTP/1.1 200 OK
Server: openresty/1.15.8.2
Date: Thu, 05 Dec 2019 05:41:07 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Vary: Accept-Encoding
Last-Modified: Tue, 19 Nov 2019 15:14:41 GMT
ETag: "5dd406e1-264"
Accept-Ranges: bytes

```

# 7. Dashboard

> 官方 https://github.com/kubernetes/dashboard


## 7.1 Dashboard 介绍

> Dashboard 是 Kubernetes 集群的 通用 WEB UI 
> 它允许用户管理集群中运行的应用程序并对其进行故障排除，以及管理集群本身。


## 7.2 部署 Dashboard

> 注意 dashboard 1.10.x 版本 不支持 kubernetes 1.16.x 必须使用 2.0 版本否则报错
> 
> 404 the server could not find the requested resource 
> 

### 7.2.1 下载 yaml 文件

```
# 下载 yaml 文件

https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta6/aio/deploy/recommended.yaml


```

### 7.2.2 apply 导入文件

```
[root@k8s-node-1 dashboard]# kubectl apply -f recommended.yaml 
namespace/kubernetes-dashboard created
serviceaccount/kubernetes-dashboard created
service/kubernetes-dashboard created
secret/kubernetes-dashboard-certs created
secret/kubernetes-dashboard-csrf created
secret/kubernetes-dashboard-key-holder created
configmap/kubernetes-dashboard-settings created
role.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrole.rbac.authorization.k8s.io/kubernetes-dashboard created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/dashboard-metrics-scraper created
deployment.apps/dashboard-metrics-scraper created

```


### 7.2.3 查看服务状态


```
[root@k8s-node-1 yaml]# kubectl get pods -n kubernetes-dashboard |grep dashboard
dashboard-metrics-scraper-76585494d8-vgfmz   1/1     Running   0          9m22s
kubernetes-dashboard-b65488c4-cqnc8          1/1     Running   0          9m22s


# svc 服务

[root@k8s-node-1 yaml]# kubectl get svc -n kubernetes-dashboard |grep dashboard
dashboard-metrics-scraper   ClusterIP   10.254.17.210   <none>        8000/TCP   9m44s
kubernetes-dashboard        ClusterIP   10.254.7.84     <none>        443/TCP    9m44s

```


### 7.2.4 暴露公网

> 访问 kubernetes 服务，既暴露 kubernetes 内的端口到 外网，有很多种方案
>
> 1. LoadBlancer  ( 支持的公有云服务的负载均衡 )
>
> 2. NodePort (映射所有 node 中的某个端口，暴露到公网中)
>
> 3. Ingress ( 支持反向代理软件的对外服务, 如: Nginx , HAproxy 等)



```
# 由于我们已经部署了 Nginx-ingress 所以这里使用 ingress 来暴露出去

# Dashboard 这边 从 svc 上看只 暴露了 443 端口，所以这边需要生成一个证书
# 注: 这里由于测试，所以使用 openssl 生成临时的证书

```

```
# 生成证书

# 创建一个 基于 自身域名的 证书

openssl req -x509 -nodes -days 3650 -newkey rsa:2048 -keyout dashboard.jicki.cn-key.key -out dashboard.jicki.cn.pem -subj "/CN=dashboard.jicki.cn"


# 导入 域名的证书 到 集群 的 secret 中

kubectl create secret tls dashboard-secret --namespace=kubernetes-dashboard --cert dashboard.jicki.cn.pem --key dashboard.jicki.cn-key.key

```


```
# 查看 secret

[root@k8s-node-1 ssl]# kubectl get secret -n kubernetes-dashboard |grep dashboard
dashboard-secret                   kubernetes.io/tls                     2      22s

```

```
[root@k8s-node-1 ssl]# kubectl describe secret/dashboard-secret -n kubernetes-dashboard
Name:         dashboard-secret
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  <none>

Type:  kubernetes.io/tls

Data
====
tls.crt:  1119 bytes
tls.key:  1704 bytes

```


```
# 创建 dashboard ingress

# 这里面 annotations 中的 backend 声明,从 v0.21.0 版本开始变更, 一定注意
# nginx-ingress < v0.21.0 使用 nginx.ingress.kubernetes.io/secure-backends: "true"
# nginx-ingress > v0.21.0 使用 nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"


# 创建 ingress 文件

vi dashboard-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
  annotations:
    ingress.kubernetes.io/ssl-passthrough: "true"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
spec:
  tls:
  - hosts:
    - dashboard.jicki.cn
    secretName: dashboard-secret
  rules:
  - host: dashboard.jicki.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443

```

```
# 导入 yaml
[root@k8s-node-1 dashboard]# kubectl apply -f dashboard-ingress.yaml 
ingress.extensions/kubernetes-dashboard created
```


```
# 查看 ingress

[root@k8s-node-1 dashboard]# kubectl get ingress -n kubernetes-dashboard
NAME                   HOSTS                ADDRESS   PORTS     AGE
kubernetes-dashboard   dashboard.jicki.cn             80, 443   34s

```


### 7.2.6 测试访问


```
[root@k8s-node-1 dashboard]# curl -I -k https://dashboard.jicki.cn
HTTP/2 200 
server: openresty/1.15.8.2
date: Thu, 05 Dec 2019 06:39:03 GMT
content-type: text/html; charset=utf-8
content-length: 1262
vary: Accept-Encoding
strict-transport-security: max-age=15724800; includeSubDomains
accept-ranges: bytes
cache-control: no-store
last-modified: Thu, 14 Nov 2019 13:39:35 GMT


```



### 7.2.7 令牌 登录认证


```
# 创建一个 dashboard rbac 超级用户

vi dashboard-admin-rbac.yaml


---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-admin
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-admin
  namespace: kubernetes-dashboard

```

```
# 导入文件
[root@k8s-node-1 dashboard]# kubectl apply -f dashboard-admin-rbac.yaml 
serviceaccount/kubernetes-dashboard-admin created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-admin created

```


```
# 查看 secret

[root@k8s-node-1 dashboard]# kubectl get secret -n kubernetes-dashboard | grep kubernetes-dashboard-admin
kubernetes-dashboard-admin-token-9dkg4   kubernetes.io/service-account-token   3      38s

```


```
# 查看 token 部分

[root@k8s-node-1 dashboard]# kubectl describe -n kubernetes-dashboard secret/kubernetes-dashboard-admin-token-9dkg4
Name:         kubernetes-dashboard-admin-token-9dkg4
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard-admin
              kubernetes.io/service-account.uid: aee23b33-43a4-4fb4-b498-6c2fb029d63c

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlI4UlpGcTcwR2hkdWZfZWk1X0RUcVI5dkdraXFnNW8yYUV1VVRPQlJYMEkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi05ZGtnNCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImFlZTIzYjMzLTQzYTQtNGZiNC1iNDk4LTZjMmZiMDI5ZDYzYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.oyvo_bIM0Ukbs3ov8XbmJffpdK1nec7oKJBxu8V4vesPY_keQhNS9xiAw6zdF2Db2tiEzcpmN3SAgwGjfid5rlSQxGpNK3mkp1r60WSAhyU5e7RqwA9xRO-EtCZ2akrqFKzEn4j_7FGwbKbNsdRurDdOLtKU5KvFsFh5eRxvB6PECT2mgSugfHorrI1cYOw0jcQKE_hjVa94xUseYX12PyGQfoUyC6ZhwIBkRnCSNdbcb0VcGwTerwysR0HFvozAJALh_iOBTDYDUNh94XIRh2AHCib-KVoJt-e2jUaGH-Z6yniLmNr15q5xLfNBd1qPpZHCgoJ1JYz4TeF6udNxIA

```

```
# 复制 token 如下部分:

token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IlI4UlpGcTcwR2hkdWZfZWk1X0RUcVI5dkdraXFnNW8yYUV1VVRPQlJYMEkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi05ZGtnNCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImFlZTIzYjMzLTQzYTQtNGZiNC1iNDk4LTZjMmZiMDI5ZDYzYyIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.oyvo_bIM0Ukbs3ov8XbmJffpdK1nec7oKJBxu8V4vesPY_keQhNS9xiAw6zdF2Db2tiEzcpmN3SAgwGjfid5rlSQxGpNK3mkp1r60WSAhyU5e7RqwA9xRO-EtCZ2akrqFKzEn4j_7FGwbKbNsdRurDdOLtKU5KvFsFh5eRxvB6PECT2mgSugfHorrI1cYOw0jcQKE_hjVa94xUseYX12PyGQfoUyC6ZhwIBkRnCSNdbcb0VcGwTerwysR0HFvozAJALh_iOBTDYDUNh94XIRh2AHCib-KVoJt-e2jUaGH-Z6yniLmNr15q5xLfNBd1qPpZHCgoJ1JYz4TeF6udNxIA

```

