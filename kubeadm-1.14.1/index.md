# kubeadm HA to 1.14.1


> kubeadm HA to 1.14.1


# kubernetes 1.14.1

> 本文基于 kubeadm 方式部署，kubeadm 在1.13 版本以后正式进入 GA.
>
> 目前国内各大厂商都有 kubeadm 的镜像源，对于部署 kubernetes 来说是大大的便利.
>
> 从官方对 kubeadm 的更新频繁度来看，kubeadm 应该是后面的趋势，毕竟二进制部署确实麻烦了点.



# 1. 环境说明


|系统|IP|Docker|Kernel|作用|
|-|-|-|-|-|
|CentOS 7 x64|192.168.168.11|18.09.6|4.4.179|K8s-Master|
|CentOS 7 x64|192.168.168.12|18.09.6|4.4.179|K8s-Master|
|CentOS 7 x64|192.168.168.13|18.09.6|4.4.179|K8s-Master|
|CentOS 7 x64|192.168.168.14|18.09.6|4.4.179|K8s-Node|
|CentOS 7 x64|192.168.168.15|18.09.6|4.4.179|K8s-Node|


## 1.1 初始化环境

### 1.1.1 配置 hosts

```
hostnamectl --static set-hostname hostname

kubernetes-1  192.168.168.11
kubernetes-2  192.168.168.12
kubernetes-3  192.168.168.13
kubernetes-4  192.168.168.14
kubernetes-5  192.168.168.15
```



```
#编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts

192.168.168.11 kubernetes-1
192.168.168.12 kubernetes-2
192.168.168.13 kubernetes-3
192.168.168.14 kubernetes-4
192.168.168.15 kubernetes-5  
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



# 2. 安装 docker

> The list of validated docker versions has changed. 1.11.1 and 1.12.1 have been removed. The current list is 1.13.1, 17.03, 17.06, 17.09, 18.06, 18.09.


```
# 安装 yum-config-manager

yum -y install yum-utils

# 导入
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo


# 更新 repo
yum makecache

# 查看yum 版本

yum list docker-ce.x86_64  --showduplicates |sort -r


# 官方目前版本为18.09.x 这里直接安装最新版本就行

yum -y install docker-ce


# 查看安装

docker version
Client:
 Version:           18.09.6
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        481bc77156
 Built:             Sat May  4 02:34:58 2019
 OS/Arch:           linux/amd64
 Experimental:      false

```
 

## 2.1 更改docker 配置


```
# 添加配置

vi /etc/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker-storage-setup.service
Wants=docker-storage-setup.service

[Service]
Type=notify
Environment=GOTRACEBACK=crash
ExecReload=/bin/kill -s HUP $MAINPID
Delegate=yes
KillMode=process
ExecStart=/usr/bin/dockerd \
          $DOCKER_OPTS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $DOCKER_DNS_OPTIONS \
          $INSECURE_REGISTRY
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=1min
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target

```

```
# 修改其他配置 第一种


mkdir -p /etc/systemd/system/docker.service.d/


vi /etc/systemd/system/docker.service.d/docker-options.conf

# 添加如下 :   (注意 environment 必须在同一行，如果出现换行会无法加载)

# docker 版本 17.03.2 之前配置为 --graph=/opt/docker

# docker 版本 17.04.x 之后配置为 --data-root=/opt/docker 

# 10.254.0.0/16 是 kubernetes 预分配的IP，下面会用于 k8s svc 的IP段分配
 

[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.254.0.0/16 \
    --registry-mirror=https://registry.docker-cn.com \
    --exec-opt native.cgroupdriver=systemd \
    --data-root=/opt/docker --log-opt max-size=50m --log-opt max-file=5"




vi /etc/systemd/system/docker.service.d/docker-dns.conf


# 添加如下 : 

[Service]
Environment="DOCKER_DNS_OPTIONS=\
    --dns 114.114.114.114 \
    --dns-search default.svc.cluster.local \
    --dns-search svc.cluster.local --dns-search localdomain \
    --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2 "
    
```

```
# 修改配置 第二种

vi /etc/docker/daemon.json


{
  "insecure-registries": ["10.254.0.0/16"],
  "registry-mirrors": ["https://dockerhub.azk8s.cn","https://gcr.azk8s.cn","https://quay.azk8s.cn"],
  "exec-opts": ["native.cgroupdriver=systemd"],
  "data-root": "/opt/docker",
  "log-opts": {
  "max-size":"50m",
  "max-file":"5"
  },
  "dns-search": ["default.svc.cluster.local", "svc.cluster.local", "localdomain"],
  "dns-opts": ["ndots:2", "timeout:2", "attempts:2"]
}


```


```
# 重新读取配置，启动 docker, 并设置自动启动 

systemctl daemon-reload
systemctl start docker
systemctl enable docker

```


```
# 如果报错 请使用
journalctl -f -t docker  和 journalctl -u docker 来定位问题

```


# 3. 部署 kubernetes 


## 3.1 安装相关软件

> 所有软件安装都通过 yum 安装


```

# kubernetes 相关 (Master)
yum -y install kubelet-1.14.1 kubeadm-1.14.1 kubectl-1.14.1

# kubernetes 相关 (Node)
yum -y install kubelet-1.14.1 kubeadm-1.14.1

# ipvs 相关

yum -y install ipvsadm ipset

```



```
# 安装的软件列表

socat-1.7.3.2-2.el7.x86_64               
libnetfilter_queue-1.0.2-2.el7_2.x86_64  
libnetfilter_cttimeout-1.0.0-6.el7.x86_64
kubectl-1.14.1-0.x86_64                  
libnetfilter_cthelper-1.0.0-9.el7.x86_64 
conntrack-tools-1.4.4-4.el7.x86_64       
kubernetes-cni-0.7.5-0.x86_64            
kubelet-1.14.1-0.x86_64                  
cri-tools-1.12.0-0.x86_64                
kubeadm-1.14.1-0.x86_64
ipvsadm.x86_64 0:1.27-7.el7
ipset-6.38-3.el7_6.x86_64

```


```
# 这里由于 kubelet 的驱动默认是  cgroupfs 
# 但是 docker 在centos 7 下默认驱动是 systemd
# 这里必须修改为相同的，否则很多问题

方法一：修改 kubelet

vi /var/lib/kubelet/config.yaml

cgroupDriver: cgroupfs

# 修改为

cgroupDriver: systemd



方法二：修改 docker

vi /etc/systemd/system/docker.service.d/docker-options.conf

添加

--exec-opt native.cgroupdriver=cgroupfs \

```




```
# 配置 kubelet 自动启动 (暂时不需要启动)

systemctl enable kubelet.service

```



## 3.2 配置 Nginx to API Server 代理

> 这里可以使用 云上的LB, 物理的 LB 等等, 见仁见智,  我这里是觉得配置虚拟VIP太麻烦。
>
> 所以在每个 机器上面都配置一个 Nginx 反向代理 API Server
>
> 由于这里API Server 会使用 6443 端口，所以这里 Nginx 代理端口为 8443
>

### 3.2.1 创建 nginx 配置文件

```
# 创建配置目录
mkdir -p /etc/nginx

# 写入代理配置
cat << EOF >> /etc/nginx/nginx.conf
error_log stderr notice;

worker_processes auto;
events {
  multi_accept on;
  use epoll;
  worker_connections 1024;
}

stream {
    upstream kube_apiserver {
        least_conn;
        server 192.168.168.11:6443;
        server 192.168.168.12:6443;
        server 192.168.168.13:6443;
    }

    server {
        listen        0.0.0.0:8443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
EOF

```

### 3.2.2 授权配置文件

```
# 更新权限
chmod +r /etc/nginx/nginx.conf
```



### 3.2.3 创建系统 systemd.service 文件

```
cat << EOF >> /etc/systemd/system/nginx-proxy.service
[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 127.0.0.1:8443:6443 \\
                              -v /etc/nginx:/etc/nginx \\
                              --name nginx-proxy \\
                              --net=host \\
                              --restart=on-failure:5 \\
                              --memory=512M \\
                              nginx:alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
EOF
```


### 3.2.4 启动nginx

```
# 启动 Nginx

systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy
systemctl status nginx-proxy

```





## 3.3 修改 kubeadm 配置信息



```
# 导出 配置 信息

kubeadm config print init-defaults > kubeadm-init.yaml

```



```
# 修改相关配置 ，本文配置信息如下


apiVersion: kubeadm.k8s.io/v1beta1
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
  advertiseAddress: 127.0.0.1
  bindPort: 6443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: kubernetes-1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta1
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controlPlaneEndpoint: "127.0.0.1:8443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.14.1
networking:
  dnsDomain: cluster.local
  podSubnet: "10.254.64.0/18"
  serviceSubnet: "10.254.0.0/18"
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


[bootstrap-token] creating the "cluster-info" ConfigMap in the "kube-public" namespace
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

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

  kubeadm join 127.0.0.1:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:5dff9a228bf55f1e8a6a754f3c087017247734b06c24138c0474d448d8aa5569 \
    --experimental-control-plane          

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 127.0.0.1:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:5dff9a228bf55f1e8a6a754f3c087017247734b06c24138c0474d448d8aa5569

```



```
# 拷贝权限文件

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

```


```
# 查看集群状态

[root@kubernetes-1 ~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
controller-manager   Healthy   ok                  
scheduler            Healthy   ok                  
etcd-0               Healthy   {"health":"true"}   

```


> 至此，集群初始化完成，第一个 Master 部署完成。




## 3.5 部署其他 Master

> 这里需要将 第一个 Master 中的证书拷贝到其他 Master 机器中



```
# 查看 第一个 Master 证书目录

[root@kubernetes-1 ~]# ll /etc/kubernetes/pki/
总用量 56
-rw-r--r-- 1 root root 1237 5月   9 11:21 apiserver.crt
-rw-r--r-- 1 root root 1090 5月   9 11:21 apiserver-etcd-client.crt
-rw------- 1 root root 1679 5月   9 11:21 apiserver-etcd-client.key
-rw------- 1 root root 1675 5月   9 11:21 apiserver.key
-rw-r--r-- 1 root root 1099 5月   9 11:21 apiserver-kubelet-client.crt
-rw------- 1 root root 1679 5月   9 11:21 apiserver-kubelet-client.key
-rw-r--r-- 1 root root 1025 5月   9 11:21 ca.crt
-rw------- 1 root root 1679 5月   9 11:21 ca.key
drwxr-xr-x 2 root root  162 5月   9 11:21 etcd
-rw-r--r-- 1 root root 1038 5月   9 11:21 front-proxy-ca.crt
-rw------- 1 root root 1675 5月   9 11:21 front-proxy-ca.key
-rw-r--r-- 1 root root 1058 5月   9 11:21 front-proxy-client.crt
-rw------- 1 root root 1675 5月   9 11:21 front-proxy-client.key
-rw------- 1 root root 1679 5月   9 11:21 sa.key
-rw------- 1 root root  451 5月   9 11:21 sa.pub

```


### 3.5.1 创建证书目录



```

ssh kubernetes-2 "mkdir -p /etc/kubernetes/pki/etcd"

ssh kubernetes-3 "mkdir -p /etc/kubernetes/pki/etcd"

```


### 3.5.2 拷贝相应证书


```

scp /etc/kubernetes/pki/ca.* kubernetes-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* kubernetes-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* kubernetes-2:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* kubernetes-2:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf kubernetes-2:/etc/kubernetes/



scp /etc/kubernetes/pki/ca.* kubernetes-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/sa.* kubernetes-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/front-proxy-ca.* kubernetes-3:/etc/kubernetes/pki/
scp /etc/kubernetes/pki/etcd/ca.* kubernetes-3:/etc/kubernetes/pki/etcd/
scp /etc/kubernetes/admin.conf kubernetes-3:/etc/kubernetes/

```



### 3.5.3 加入 kubernetes 集群

> 如上有 kubeadm init 后有两条 kubeadm join 命令,  --experimental-control-plane 为 加入 Master
>
> 另外token 有时效性，如果提示 token 失效，请自行创建一个新的 token.
>
> kubeadm token create --print-join-command    创建新的 join token



```
kubeadm join 127.0.0.1:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:5dff9a228bf55f1e8a6a754f3c087017247734b06c24138c0474d448d8aa5569 \
    --experimental-control-plane


```



```
# 输出如下

This node has joined the cluster and a new control plane instance was created:

* Certificate signing request was sent to apiserver and approval was received.
* The Kubelet was informed of the new secure connection details.
* Control plane (master) label and taint were applied to the new node.
* The Kubernetes control plane instances scaled up.
* A new etcd member was added to the local/stacked etcd cluster.

To start administering your cluster from this node, you need to run the following as a regular user:

        mkdir -p $HOME/.kube
        sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        sudo chown $(id -u):$(id -g) $HOME/.kube/config

Run 'kubectl get nodes' to see this node join the cluster.

```




```
# 分别执行  创建配置文件

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

```




### 3.5.4 验证 Master 节点


> 这里 STATUS 显示 NotReady 是因为 没有安装网络组件


```
# 查看 node

[root@kubernetes-1 ~]# kubectl get nodes
NAME           STATUS     ROLES    AGE     VERSION
kubernetes-1   NotReady   master   11m     v1.14.1
kubernetes-2   NotReady   master   5m38s   v1.14.1
kubernetes-3   NotReady   master   2m39s   v1.14.1

```




## 3.6 部署 Node 节点


```
kubeadm join 127.0.0.1:8443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:5dff9a228bf55f1e8a6a754f3c087017247734b06c24138c0474d448d8aa5569

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

[root@kubernetes-1 ~]# kubectl get nodes
NAME           STATUS     ROLES    AGE     VERSION
kubernetes-1   NotReady   master   13m     v1.14.1
kubernetes-2   NotReady   master   8m20s   v1.14.1
kubernetes-3   NotReady   master   5m21s   v1.14.1
kubernetes-4   NotReady   <none>   45s     v1.14.1
kubernetes-5   NotReady   <none>   39s     v1.14.1

```




## 3.7 安装网络组件


> Calico 网络
>
> 官方文档 https://docs.projectcalico.org/v3.7/introduction


### 3.7.1  下载 Calico yaml

```
# 下载 yaml 文件

wget https://docs.projectcalico.org/v3.7/manifests/calico.yaml


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

[root@kubernetes-1 calico]# kubectl apply -f calico.yaml 
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.extensions/calico-node created
serviceaccount/calico-node created
deployment.extensions/calico-kube-controllers created
serviceaccount/calico-kube-controllers created



# 查看服务

[root@kubernetes-1 calico]# kubectl get pods -n kube-system |grep calico
calico-kube-controllers-8646dd497f-4d4hr   1/1     Running   0          28s
calico-node-455mf                          1/1     Running   0          28s
calico-node-b8mrr                          1/1     Running   0          28s
calico-node-fq9dj                          1/1     Running   0          28s
calico-node-sk77j                          1/1     Running   0          28s
calico-node-xxnb8                          1/1     Running   0          28s

```



## 3.8 检验整体集群



### 3.8.1 查看 状态

> 所有的 STATUS 都为 Ready 
>
> ROLES 为 三台 Master,  二台 <none> ，为 Node 节点


```


[root@kubernetes-1 ~]# kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
kubernetes-1   Ready    master   33m   v1.14.1
kubernetes-2   Ready    master   28m   v1.14.1
kubernetes-3   Ready    master   25m   v1.14.1
kubernetes-4   Ready    <none>   20m   v1.14.1
kubernetes-5   Ready    <none>   20m   v1.14.1

```


### 3.8.2 查看 pods 状态


```
[root@kubernetes-1 ~]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-8646dd497f-4d4hr   1/1     Running   0          3m33s
kube-system   calico-node-455mf                          1/1     Running   0          3m33s
kube-system   calico-node-b8mrr                          1/1     Running   0          3m33s
kube-system   calico-node-fq9dj                          1/1     Running   0          3m33s
kube-system   calico-node-sk77j                          1/1     Running   0          3m33s
kube-system   calico-node-xxnb8                          1/1     Running   0          3m33s
kube-system   coredns-d5947d4b-q4pst                     1/1     Running   0          35m
kube-system   coredns-d5947d4b-sz2bm                     1/1     Running   0          35m
kube-system   etcd-kubernetes-1                          1/1     Running   0          34m
kube-system   etcd-kubernetes-2                          1/1     Running   0          30m
kube-system   etcd-kubernetes-3                          1/1     Running   0          27m
kube-system   kube-apiserver-kubernetes-1                1/1     Running   0          35m
kube-system   kube-apiserver-kubernetes-2                1/1     Running   0          30m
kube-system   kube-apiserver-kubernetes-3                1/1     Running   0          26m
kube-system   kube-controller-manager-kubernetes-1       1/1     Running   1          34m
kube-system   kube-controller-manager-kubernetes-2       1/1     Running   0          30m
kube-system   kube-controller-manager-kubernetes-3       1/1     Running   0          26m
kube-system   kube-proxy-2ht7m                           1/1     Running   0          35m
kube-system   kube-proxy-bkg7z                           1/1     Running   0          22m
kube-system   kube-proxy-sdxnj                           1/1     Running   0          23m
kube-system   kube-proxy-tpc4x                           1/1     Running   0          27m
kube-system   kube-proxy-wvllx                           1/1     Running   0          30m
kube-system   kube-scheduler-kubernetes-1                1/1     Running   1          35m
kube-system   kube-scheduler-kubernetes-2                1/1     Running   0          30m
kube-system   kube-scheduler-kubernetes-3                1/1     Running   0          26m

```


### 3.8.3 查看 svc 的状态


```

[root@kubernetes-1 ~]# kubectl get svc --all-namespaces
NAMESPACE     NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.254.0.1    <none>        443/TCP                  43m
kube-system   kube-dns     ClusterIP   10.254.0.10   <none>        53/UDP,53/TCP,9153/TCP   43m

```






### 3.8.3 查看 IPVS 的状态


```
[root@kubernetes-1 ~]# ipvsadm -L -n

IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 192.168.168.11:6443          Masq    1      2          0         
  -> 192.168.168.12:6443          Masq    1      0          0         
  -> 192.168.168.13:6443          Masq    1      1          0         
TCP  10.254.0.10:53 rr
  -> 10.254.91.66:53              Masq    1      0          0         
  -> 10.254.105.65:53             Masq    1      0          0         
TCP  10.254.0.10:9153 rr
  -> 10.254.91.66:9153            Masq    1      0          0         
  -> 10.254.105.65:9153           Masq    1      0          0         
UDP  10.254.0.10:53 rr
  -> 10.254.91.66:53              Masq    1      0          0         
  -> 10.254.105.65:53             Masq    1      0          0         


```








# 4. 测试集群


## 4.1 创建一个 nginx deployment

```

apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 3
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
[root@kubernetes-1 nginx]# kubectl apply -f nginx-deployment.yaml 
deployment.apps/nginx-dm created
service/nginx-svc created

```




```
[root@kubernetes-1 nginx]# kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP             NODE           NOMINATED NODE   READINESS GATES
nginx-dm-76cf455886-4drbs   1/1     Running   0          23s   10.254.91.67   kubernetes-4   <none>           <none>
nginx-dm-76cf455886-blldj   1/1     Running   0          23s   10.254.96.66   kubernetes-5   <none>           <none>
nginx-dm-76cf455886-st899   1/1     Running   0          23s   10.254.96.65   kubernetes-5   <none>           <none>



[root@kubernetes-1 nginx]# kubectl get svc -o wide    
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes   ClusterIP   10.254.0.1     <none>        443/TCP   56m   <none>
nginx-svc    ClusterIP   10.254.2.177   <none>        80/TCP    41s   name=nginx

```


```
# 访问 svc


[root@kubernetes-1 nginx]# curl 10.254.2.177
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

[root@kubernetes-1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.2.177:80 rr
  -> 10.254.91.67:80              Masq    1      0          0         
  -> 10.254.96.65:80              Masq    1      0          0         
  -> 10.254.96.66:80              Masq    1      0          0    

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

[root@kubernetes-1 ~]# kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
alpine                      1/1     Running   0          3s
nginx-dm-76cf455886-4drbs   1/1     Running   0          11m
nginx-dm-76cf455886-blldj   1/1     Running   0          11m
nginx-dm-76cf455886-st899   1/1     Running   0          11m

```



```
# 测试

[root@kubernetes-1 ~]# kubectl exec -it alpine nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.254.0.1 kubernetes.default.svc.cluster.local





[root@kubernetes-1 ~]# kubectl exec -it alpine nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.2.177 nginx-svc.default.svc.cluster.local

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
[root@kubernetes-1 metrics]# kubectl apply -f metrics-server.yaml 
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
serviceaccount/metrics-server created
deployment.extensions/metrics-server created
service/metrics-server created

```


### 5.1.2 查看服务


```
[root@kubernetes-1 metrics]# kubectl get pods -n kube-system |grep metrics
metrics-server-5f7bbc8788-6rsj4            1/1     Running   0          54s

```


### 5.1.3 测试采集


```
[root@kubernetes-1 metrics]# kubectl top node
NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
kubernetes-1   159m         3%     1887Mi          11%       
kubernetes-2   153m         3%     1521Mi          9%        
kubernetes-3   150m         3%     1419Mi          8%        
kubernetes-4   113m         2%     1198Mi          7%        
kubernetes-5   76m          1%     1002Mi          6%   

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
# 配置 pods 份数

replicas: 3


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
                - kubernetes-1
                - kubernetes-2
                - kubernetes-3
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
                - kubernetes-1
                - kubernetes-2
                - kubernetes-3
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
[root@kubernetes-1 ingress]# kubectl apply -f mandatory.yaml 
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
[root@kubernetes-1 ingress]# kubectl get pods -n ingress-nginx -o wide
NAME                                        READY   STATUS    RESTARTS   AGE    IP               NODE           NOMINATED NODE   READINESS GATES
nginx-ingress-controller-7b5477f4f6-btvqm   1/1     Running   0          118s   192.168.168.13   kubernetes-3   <none>           <none>
nginx-ingress-controller-7b5477f4f6-sgnj8   1/1     Running   0          3m6s   192.168.168.12   kubernetes-2   <none>           <none>
nginx-ingress-controller-7b5477f4f6-xqldf   1/1     Running   0          67s    192.168.168.11   kubernetes-1   <none>           <none>

```



### 6.2.5 测试 ingress



```
# 查看之前创建的 Nginx

[root@kubernetes-1 ingress]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.254.0.1     <none>        443/TCP   25h
nginx-svc    ClusterIP   10.254.2.177   <none>        80/TCP    24h

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
[root@kubernetes-1 yaml]# kubectl apply -f nginx-ingress.yaml 
ingress.extensions/nginx-ingress created



# 查看 ingress

[root@kubernetes-1 yaml]# kubectl get ingress
NAME            HOSTS            ADDRESS   PORTS   AGE
nginx-ingress   nginx.jicki.cn             80      3s



```




### 6.2.6 测试访问


```
# kubernetes-1

[root@kubernetes-1 ~]# ping nginx.jicki.cn
PING nginx.jicki.cn (192.168.168.11) 56(84) bytes of data.
64 bytes from kubernetes-1 (192.168.168.11): icmp_seq=1 ttl=64 time=0.070 ms
64 bytes from kubernetes-1 (192.168.168.11): icmp_seq=2 ttl=64 time=0.031 ms
64 bytes from kubernetes-1 (192.168.168.11): icmp_seq=3 ttl=64 time=0.032 ms
64 bytes from kubernetes-1 (192.168.168.11): icmp_seq=4 ttl=64 time=0.044 ms



[root@kubernetes-1 ~]# curl -I nginx.jicki.cn
HTTP/1.1 200 OK
Server: nginx/1.15.10
Date: Fri, 10 May 2019 09:13:41 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Vary: Accept-Encoding
Last-Modified: Tue, 16 Apr 2019 21:22:22 GMT
ETag: "5cb6478e-264"
Accept-Ranges: bytes


```



```
# kubernetes-2


[root@kubernetes-1 ~]# ping nginx.jicki.cn
PING nginx.jicki.cn (192.168.168.12) 56(84) bytes of data.
64 bytes from kubernetes-2 (192.168.168.12): icmp_seq=1 ttl=63 time=0.468 ms
64 bytes from kubernetes-2 (192.168.168.12): icmp_seq=2 ttl=63 time=0.476 ms
64 bytes from kubernetes-2 (192.168.168.12): icmp_seq=3 ttl=63 time=1.15 ms
64 bytes from kubernetes-2 (192.168.168.12): icmp_seq=4 ttl=63 time=0.392 ms



[root@kubernetes-1 ~]# curl -I nginx.jicki.cn
HTTP/1.1 200 OK
Server: nginx/1.15.10
Date: Fri, 10 May 2019 09:21:06 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Vary: Accept-Encoding
Last-Modified: Tue, 16 Apr 2019 21:22:22 GMT
ETag: "5cb6478e-264"
Accept-Ranges: bytes


```


```
# kubernetes-3


[root@kubernetes-1 ~]# ping nginx.jicki.cn
PING nginx.jicki.cn (192.168.168.13) 56(84) bytes of data.
64 bytes from kubernetes-3 (192.168.168.13): icmp_seq=1 ttl=64 time=0.273 ms
64 bytes from kubernetes-3 (192.168.168.13): icmp_seq=2 ttl=64 time=0.098 ms
64 bytes from kubernetes-3 (192.168.168.13): icmp_seq=3 ttl=64 time=0.123 ms
64 bytes from kubernetes-3 (192.168.168.13): icmp_seq=4 ttl=64 time=0.092 ms



[root@kubernetes-1 ~]# curl -I nginx.jicki.cn
HTTP/1.1 200 OK
Server: nginx/1.15.10
Date: Fri, 10 May 2019 09:22:31 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Vary: Accept-Encoding
Last-Modified: Tue, 16 Apr 2019 21:22:22 GMT
ETag: "5cb6478e-264"
Accept-Ranges: bytes


```





# 7. Dashboard

> 官方 https://github.com/kubernetes/dashboard


## 7.1 Dashboard 介绍

> Dashboard 是 Kubernetes 集群的 通用 WEB UI 
> 它允许用户管理集群中运行的应用程序并对其进行故障排除，以及管理集群本身。


## 7.2 部署 Dashboard

> 就目前的版本 v1.10.1 仍然只支持 Heapster 作为监控数据采集显示.
>
> 暂时还不支持 Metrics-Server 做为 监控数据采集，这里就不配置 Heapster 了.
>

### 7.2.1 下载 yaml 文件

```
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml

```


### 7.2.2 修改yaml文件


```
# 这里主要是将 下载 images 的地址修改为 阿里 的地址


sed -i 's/k8s\.gcr\.io/registry\.cn-hangzhou\.aliyuncs\.com\/google_containers/g' kubernetes-dashboard.yaml

```




```

# 这里由于我上面 配置的 apiserver 端口为 8443 与这里的端口冲突了

# 这里 关于修改 默认端口的问题, 如下地方:

        ports:
        - containerPort: 8443
          protocol: TCP
          
.........
        livenessProbe:
          httpGet:
            scheme: HTTPS
            path: /
            port: 8443

..........
  ports:
    - port: 443
      targetPort: 8443


```

```
# 直接修改 上面端口 是不可以的， 必须在 args 下也更改

# 增加选项, 这里修改为 9443

        args:
          - --port=9443
          
```






### 7.2.3 apply 导入文件

```

[root@kubernetes-1 dashboard]# kubectl apply -f kubernetes-dashboard.yaml 
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
role.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
rolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-minimal created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created

```


### 7.2.4 查看服务状态


```
[root@kubernetes-1 dashboard]# kubectl get pods -n kube-system |grep dashboard
kubernetes-dashboard-5d9599dc98-kbhbw      1/1     Running   0          3m6s



[root@kubernetes-1 dashboard]# kubectl get svc -n kube-system |grep dashboard
kubernetes-dashboard   ClusterIP   10.254.28.8   <none>        443/TCP                  3m19s

```


### 7.2.5 暴露公网

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

kubectl create secret tls dashboard-secret --namespace=kube-system --cert dashboard.jicki.cn.pem --key dashboard.jicki.cn-key.key

```


```
# 查看 secret

[root@kubernetes-1 ssl]# kubectl get secret -n kube-system |grep dashboard
dashboard-secret                                 kubernetes.io/tls                     2      16s

```

```
[root@kubernetes-1 ssl]# kubectl describe secret/dashboard-secret -n kube-system
Name:         dashboard-secret
Namespace:    kube-system
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


vi dashboard-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
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

[root@kubernetes-1 dashboard]# kubectl apply -f dashboard-ingress.yaml 
ingress.extensions/kubernetes-dashboard created

```


```
# 查看 ingress


[root@kubernetes-1 dashboard]# kubectl get ingress -n kube-system
NAME                   HOSTS                ADDRESS   PORTS     AGE
kubernetes-dashboard   dashboard.jicki.cn             80, 443   21s

```


### 7.2.6 测试访问


```
[root@kubernetes-1 dashboard]# ping dashboard.jicki.cn
PING dashboard.jicki.cn (192.168.168.11) 56(84) bytes of data.
64 bytes from kubernetes-1 (192.168.168.11): icmp_seq=1 ttl=64 time=0.051 ms
64 bytes from kubernetes-1 (192.168.168.11): icmp_seq=2 ttl=64 time=0.031 ms


```


```
[root@kubernetes-1 dashboard]# curl -I -k https://dashboard.jicki.cn
HTTP/1.1 200 OK
Server: nginx/1.15.10
Date: Mon, 13 May 2019 05:57:28 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 990
Connection: keep-alive
Vary: Accept-Encoding
Accept-Ranges: bytes
Cache-Control: no-store
Last-Modified: Mon, 17 Dec 2018 09:04:43 GMT
Strict-Transport-Security: max-age=15724800; includeSubDomains

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
  namespace: kube-system
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
  namespace: kube-system

```

```
[root@kubernetes-1 dashboard]# kubectl apply -f dashboard-admin-rbac.yaml 
serviceaccount/kubernetes-dashboard-admin created
clusterrolebinding.rbac.authorization.k8s.io/kubernetes-dashboard-admin created

```


```
# 查看 secret

[root@kubernetes-1 dashboard]# kubectl -n kube-system get secret | grep kubernetes-dashboard-admin
kubernetes-dashboard-admin-token-c4lxz           kubernetes.io/service-account-token   3      20s

```


```
# 查看 token 部分


[root@kubernetes-1 dashboard]# kubectl describe -n kube-system secret/kubernetes-dashboard-admin-token-c4lxz
error: there is no need to specify a resource type as a separate argument when passing arguments in resource/name form (e.g. 'kubectl get resource/<resource_name>' instead of 'kubectl get resource resource/<resource_name>'
[root@kubernetes-1 dashboard]# kubectl describe -n kube-system secret/kubernetes-dashboard-admin-token-c4lxz 
Name:         kubernetes-dashboard-admin-token-c4lxz
Namespace:    kube-system
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard-admin
              kubernetes.io/service-account.uid: 43e785b2-7544-11e9-a1a6-000c29d28269

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  11 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1jNGx4eiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjQzZTc4NWIyLTc1NDQtMTFlOS1hMWE2LTAwMGMyOWQyODI2OSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.TM00Z9oL6YImxPAF1RbrcXjExMB-WqlgcRkpghAyXKTbWOE6IGbIA2WIpZr6VFt-A9hQ719v92C5D4T05MHwH9ZxmNvmR3TnTVwO3TfXkrs6B3wzbLK4mhtzbKOH1H3ZG20iQwukQuaacRhqqttY2b6IvL1fvlVTxYMaBOKpRuTQwt-jPgD--2DJpHOqVezAT-ZgDrpR0Qs0a478pJ283T4dBkVszmpjzsejw64UU8jM2ip1mGtkWp1OMJLufaxNQtyNmOBJGvEjCoxgRe8t91dK0BmC_8ouioTii2ppxeaSJkR7rFctEu-cMbWTIsfzcmQcHJcEDUUMdgt7l9qmYg


```

```
# 复制 token 如下部分:


token:      eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1jNGx4eiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjQzZTc4NWIyLTc1NDQtMTFlOS1hMWE2LTAwMGMyOWQyODI2OSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlLXN5c3RlbTprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.TM00Z9oL6YImxPAF1RbrcXjExMB-WqlgcRkpghAyXKTbWOE6IGbIA2WIpZr6VFt-A9hQ719v92C5D4T05MHwH9ZxmNvmR3TnTVwO3TfXkrs6B3wzbLK4mhtzbKOH1H3ZG20iQwukQuaacRhqqttY2b6IvL1fvlVTxYMaBOKpRuTQwt-jPgD--2DJpHOqVezAT-ZgDrpR0Qs0a478pJ283T4dBkVszmpjzsejw64UU8jM2ip1mGtkWp1OMJLufaxNQtyNmOBJGvEjCoxgRe8t91dK0BmC_8ouioTii2ppxeaSJkR7rFctEu-cMbWTIsfzcmQcHJcEDUUMdgt7l9qmYg



```



# 8. kubeadm upgrade

> 对于Kubeadm 部署的集群，可以使用命令直接平滑的升级，整个过程非常的简单，不会像二进制部署那样需要重新配置每个点的服务组件。


```
# 查看可升级的版本，以及升级的步骤，执行以下命令可查看，但是由于国内 k8s.io 被墙，所以国内无法获取可升级的版本。


[root@kubernetes-1 ~]# kubeadm upgrade plan


[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/config] Making sure the configuration is correct:
[upgrade/config] Reading configuration from the cluster...
[upgrade/config] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[upgrade] Fetching available versions to upgrade to
[upgrade/versions] Cluster version: v1.14.1
[upgrade/versions] kubeadm version: v1.14.1
I0625 09:26:20.514140   24089 version.go:96] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable.txt": Get https://dl.k8s.io/release/stable.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I0625 09:26:20.514167   24089 version.go:97] falling back to the local client version: v1.14.1
[upgrade/versions] Latest stable version: v1.14.1
I0625 09:26:30.567645   24089 version.go:96] could not fetch a Kubernetes version from the internet: unable to get URL "https://dl.k8s.io/release/stable-1.14.txt": Get https://dl.k8s.io/release/stable-1.14.txt: net/http: request canceled while waiting for connection (Client.Timeout exceeded while awaiting headers)
I0625 09:26:30.567672   24089 version.go:97] falling back to the local client version: v1.14.1
[upgrade/versions] Latest version in the v1.14 series: v1.14.1

Awesome, you're up-to-date! Enjoy!

```


## 8.1 升级服务组件


```
# 首先我们需要 升级 kubeadm kubelet kubectl 所有的Node 都需要升级
# 这里要千万注意，跨大版本升级千万要注意，1.14 升级到 1.15 的话，估计会出问题的。

# 查看可升级的版本

[root@kubernetes-1 ~]# yum makecache

[root@kubernetes-1 ~]# yum list kubeadm kubelet kubectl --showduplicates |sort -r

已加载插件：fastestmirror
已安装的软件包
可安装的软件包
 * updates: centos.ustc.edu.cn
Loading mirror speeds from cached hostfile
   .....
kubelet.x86_64                       1.15.0-0                        kubernetes 
kubelet.x86_64                       1.14.3-0                        kubernetes 
kubelet.x86_64                       1.14.2-0                        kubernetes 
kubelet.x86_64                       1.14.1-0                        kubernetes 
  ....
kubectl.x86_64                       1.15.0-0                        kubernetes 
kubectl.x86_64                       1.14.3-0                        kubernetes 
kubectl.x86_64                       1.14.2-0                        kubernetes 
kubectl.x86_64                       1.14.1-0                        kubernetes 
  ....
kubeadm.x86_64                       1.15.0-0                        kubernetes 
kubeadm.x86_64                       1.14.3-0                        kubernetes 
kubeadm.x86_64                       1.14.2-0                        kubernetes 
kubeadm.x86_64                       1.14.1-0                        kubernetes 

```

```
# 升级指定的版本, 

[root@kubernetes-1 ~]# yum -y install kubectl-1.14.3 kubelet-1.14.3 kubeadm-1.14.3


# 如果不小心升级到了大版本，可直接降级

[root@kubernetes-1 ~]# yum downgrade kubectl-1.14.3 kubelet-1.14.3 kubeadm-1.14.3

```



```
# 由于我们的 kubelet 是使用 systemctl 托管的服务 所以我们需要在所有Node重启一下

[root@kubernetes-1 ~]# systemctl daemon-reload
[root@kubernetes-1 ~]# systemctl restart kubelet

```





```
# 升级指定集群版本, 这里面由于我们导出了 kubeadm-init.yaml 文件，并做了修改，所以升级的时候也需要指定这个文件
# 当然在 kubeadm-init.yaml 里的(kubernetesVersion: v1.14.3)版本号也需要修改为升级的版本号

[root@kubernetes-1 ~]# kubeadm upgrade apply --config=kubeadm-init.yaml 
W0625 10:00:45.650868   20357 common.go:150] WARNING: overriding requested API server bind address: requested "127.0.0.1", actual "192.168.168.11"
[preflight] Running pre-flight checks.
[upgrade] Making sure the cluster is healthy:
[upgrade/config] Making sure the configuration is correct:
W0625 10:00:45.670849   20357 common.go:150] WARNING: overriding requested API server bind address: requested "127.0.0.1", actual "192.168.168.11"
[upgrade/version] You have chosen to change the cluster version to "v1.14.3"
[upgrade/versions] Cluster version: v1.14.1
[upgrade/versions] kubeadm version: v1.14.3
[upgrade/confirm] Are you sure you want to proceed with the upgrade? [y/N]: y
[upgrade/prepull] Will prepull images for components [kube-apiserver kube-controller-manager kube-scheduler etcd]
[upgrade/prepull] Prepulling image for component etcd.
[upgrade/prepull] Prepulling image for component kube-apiserver.
[upgrade/prepull] Prepulling image for component kube-controller-manager.
[upgrade/prepull] Prepulling image for component kube-scheduler.
[apiclient] Found 0 Pods for label selector k8s-app=upgrade-prepull-kube-controller-manager
[apiclient] Found 0 Pods for label selector k8s-app=upgrade-prepull-kube-scheduler
[apiclient] Found 3 Pods for label selector k8s-app=upgrade-prepull-kube-apiserver
[apiclient] Found 0 Pods for label selector k8s-app=upgrade-prepull-etcd
[apiclient] Found 3 Pods for label selector k8s-app=upgrade-prepull-kube-controller-manager
[apiclient] Found 3 Pods for label selector k8s-app=upgrade-prepull-kube-scheduler
[apiclient] Found 3 Pods for label selector k8s-app=upgrade-prepull-etcd
[upgrade/prepull] Prepulled image for component etcd.
[upgrade/prepull] Prepulled image for component kube-apiserver.
[upgrade/prepull] Prepulled image for component kube-controller-manager.
[upgrade/prepull] Prepulled image for component kube-scheduler.
[upgrade/prepull] Successfully prepulled the images for all the control plane components
[upgrade/apply] Upgrading your Static Pod-hosted control plane to version "v1.14.3"...
Static pod: kube-apiserver-kubernetes-1 hash: 6ad69de30854f1e2a7dded6373d4d803
Static pod: kube-controller-manager-kubernetes-1 hash: 9dd249309899307ea97eae283e64f30b
Static pod: kube-scheduler-kubernetes-1 hash: f6038c6cd5a85d4e6bb2b14e4743e83d
[upgrade/etcd] Upgrading to TLS for etcd
[upgrade/staticpods] Writing new Static Pod manifests to "/etc/kubernetes/tmp/kubeadm-upgraded-manifests168961842"
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-apiserver.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2019-06-25-10-01-54/kube-apiserver.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-apiserver-kubernetes-1 hash: 6ad69de30854f1e2a7dded6373d4d803
Static pod: kube-apiserver-kubernetes-1 hash: ba64314802dffba85edba8fc2d7f0718
[apiclient] Found 3 Pods for label selector component=kube-apiserver
[upgrade/staticpods] Component "kube-apiserver" upgraded successfully!
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-controller-manager.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2019-06-25-10-01-54/kube-controller-manager.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-controller-manager-kubernetes-1 hash: 9dd249309899307ea97eae283e64f30b
Static pod: kube-controller-manager-kubernetes-1 hash: 4421841c3f9656317e9ef5397b3b3137
[apiclient] Found 3 Pods for label selector component=kube-controller-manager
[upgrade/staticpods] Component "kube-controller-manager" upgraded successfully!
[upgrade/staticpods] Moved new manifest to "/etc/kubernetes/manifests/kube-scheduler.yaml" and backed up old manifest to "/etc/kubernetes/tmp/kubeadm-backup-manifests-2019-06-25-10-01-54/kube-scheduler.yaml"
[upgrade/staticpods] Waiting for the kubelet to restart the component
[upgrade/staticpods] This might take a minute or longer depending on the component/version gap (timeout 5m0s)
Static pod: kube-scheduler-kubernetes-1 hash: f6038c6cd5a85d4e6bb2b14e4743e83d
Static pod: kube-scheduler-kubernetes-1 hash: 529fe92a2e33decd793da38bcb616cd3
[apiclient] Found 3 Pods for label selector component=kube-scheduler
[upgrade/staticpods] Component "kube-scheduler" upgraded successfully!
[upload-config] storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.14" in namespace kube-system with the configuration for the kubelets in the cluster
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.14" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[addons] Applied essential addon: CoreDNS
[endpoint] WARNING: port specified in controlPlaneEndpoint overrides bindPort in the controlplane address
[addons] Applied essential addon: kube-proxy

[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.14.3". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.

```


```
# 查看升级版本

[root@kubernetes-1 ~]# kubectl get nodes
NAME           STATUS   ROLES    AGE   VERSION
kubernetes-1   Ready    master   46d   v1.14.3
kubernetes-2   Ready    master   46d   v1.14.3
kubernetes-3   Ready    master   46d   v1.14.3
kubernetes-4   Ready    <none>   46d   v1.14.3
kubernetes-5   Ready    <none>   46d   v1.14.3

```


```
# 查看原来的 服务

[root@kubernetes-1 ~]# kubectl get pods --all-namespaces
NAMESPACE       NAME                                        READY   STATUS    RESTARTS   AGE
default         nginx-dm-76cf455886-4drbs                   1/1     Running   3          46d
default         nginx-dm-76cf455886-blldj                   1/1     Running   3          46d
default         nginx-dm-76cf455886-st899                   1/1     Running   3          46d
ingress-nginx   nginx-ingress-controller-7b5477f4f6-btvqm   1/1     Running   3          45d
ingress-nginx   nginx-ingress-controller-7b5477f4f6-sgnj8   1/1     Running   3          45d
ingress-nginx   nginx-ingress-controller-7b5477f4f6-xqldf   1/1     Running   3          45d
kube-system     calico-kube-controllers-8646dd497f-4d4hr    1/1     Running   6          46d
kube-system     calico-node-455mf                           1/1     Running   3          46d
kube-system     calico-node-b8mrr                           1/1     Running   3          46d
kube-system     calico-node-fq9dj                           1/1     Running   3          46d
kube-system     calico-node-sk77j                           1/1     Running   2          46d
kube-system     calico-node-xxnb8                           1/1     Running   3          46d
kube-system     coredns-d5947d4b-q4pst                      1/1     Running   6          46d
kube-system     coredns-d5947d4b-sz2bm                      1/1     Running   5          46d
kube-system     etcd-kubernetes-1                           1/1     Running   2          46d
kube-system     etcd-kubernetes-2                           1/1     Running   2          46d
kube-system     etcd-kubernetes-3                           1/1     Running   2          46d
kube-system     kube-apiserver-kubernetes-1                 1/1     Running   0          17m
kube-system     kube-apiserver-kubernetes-2                 1/1     Running   2          46d
kube-system     kube-apiserver-kubernetes-3                 1/1     Running   2          46d
kube-system     kube-controller-manager-kubernetes-1        1/1     Running   0          16m
kube-system     kube-controller-manager-kubernetes-2        1/1     Running   3          46d
kube-system     kube-controller-manager-kubernetes-3        1/1     Running   2          46d
kube-system     kube-proxy-66lkl                            1/1     Running   0          16m
kube-system     kube-proxy-kzgsq                            1/1     Running   0          16m
kube-system     kube-proxy-p95qt                            1/1     Running   0          16m
kube-system     kube-proxy-rfjzx                            1/1     Running   0          16m
kube-system     kube-proxy-szbhc                            1/1     Running   0          16m
kube-system     kube-scheduler-kubernetes-1                 1/1     Running   0          16m
kube-system     kube-scheduler-kubernetes-2                 1/1     Running   3          46d
kube-system     kube-scheduler-kubernetes-3                 1/1     Running   2          46d
kube-system     kubernetes-dashboard-7f6ddd46-sfdk2         1/1     Running   6          42d
kube-system     metrics-server-588d84569d-nzbql             1/1     Running   4          46d

```

```
# 测试一下原来的服务是否正常



[root@kubernetes-1 ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.254.0.1     <none>        443/TCP   46d
nginx-svc    ClusterIP   10.254.2.177   <none>        80/TCP    46d


[root@kubernetes-1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 rr
  -> 192.168.168.11:6443          Masq    1      0          0         
  -> 192.168.168.12:6443          Masq    1      1          0         
  -> 192.168.168.13:6443          Masq    1      3          0         
TCP  10.254.0.10:53 rr
  -> 10.254.91.87:53              Masq    1      0          0         
  -> 10.254.105.68:53             Masq    1      0          0         
TCP  10.254.0.10:9153 rr
  -> 10.254.91.87:9153            Masq    1      0          0         
  -> 10.254.105.68:9153           Masq    1      0          0         
TCP  10.254.2.177:80 rr
  -> 10.254.91.86:80              Masq    1      0          0         
  -> 10.254.96.79:80              Masq    1      0          1         
  -> 10.254.96.80:80              Masq    1      0          0         
TCP  10.254.10.232:443 rr
  -> 10.254.91.84:9443            Masq    1      0          0         
TCP  10.254.28.6:443 rr
  -> 10.254.96.81:443             Masq    1      2          0         
UDP  10.254.0.10:53 rr
  -> 10.254.91.87:53              Masq    1      0          0         
  -> 10.254.105.68:53             Masq    1      0          0  


[root@kubernetes-1 ~]# 
[root@kubernetes-1 ~]# curl 10.254.2.177
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

