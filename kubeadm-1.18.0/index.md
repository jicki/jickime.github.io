# kubeadm v1.18.0 HA


> kubeadm v1.18.0


# kubernetes 1.18.0

> 本文基于 kubeadm 方式部署，kubeadm 在1.13 版本以后正式进入 GA.
>
> 目前国内各大厂商都有 kubeadm 的镜像源，对于部署 kubernetes 来说是大大的便利.
>
> 从官方对 kubeadm 的更新频繁度来看，kubeadm 应该是后面的趋势，毕竟二进制部署确实麻烦了点.



# 1. 环境说明


|系统|IP|Containerd|Kernel|hostname|备注|
|-|-|-|-|-|-|
|Aws Linux|10.18.77.61|19.03.6-ce|4.14.171|k8s-node-3|Master|
|Aws Linux|10.18.77.117|19.03.6-ce|4.14.171|k8s-node-1|Master or node|
|Aws Linux|10.18.77.218|19.03.6-ce|4.14.171|k8s-node-2|Master or node|

## 1.1 初始化环境

### 1.1.1 配置 hosts

```
hostnamectl --static set-hostname hostname
hostnamectl --transient set-hostname hostname

k8s-node-1  10.18.77.61
k8s-node-2  10.18.77.117
k8s-node-3  10.18.77.218
```



```
#编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts

10.18.77.61   k8s-node-1
10.18.77.117  k8s-node-2
10.18.77.218  k8s-node-3
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
# 开启内核 namespace 支持

grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"


```



```
# 修改内核参数

cat<<EOF > /etc/sysctl.d/docker.conf
net.ipv4.ip_forward=1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
vm.swappiness=0
EOF


# 生效配置

sysctl --system



# 重启系统
reboot

```


```
# 添加 kubernetes 内核优化

cat<<EOF > /etc/sysctl.d/kubernetes.conf
# conntrack 连接跟踪数最大数量
net.netfilter.nf_conntrack_max = 10485760 
# 允许送到队列的数据包的最大数目
net.core.netdev_max_backlog = 10000
# ARP 高速缓存中的最少层数
net.ipv4.neigh.default.gc_thresh1 = 80000
# ARP 高速缓存中的最多的记录软限制
net.ipv4.neigh.default.gc_thresh2 = 90000
# ARP 高速缓存中的最多记录的硬限制
net.ipv4.neigh.default.gc_thresh3 = 100000
EOF


# 生效配置

sysctl --system

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

## 2.1 检查系统

```
curl -s https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh | bash
```



## 2.2 安装 docker


```
# 清除缓存
yum makecache


yum -y install docker

```



* 因为 aws linux 不支持如下安装: 如下支持 ubuntu, debain, centos, rhel

```

# 指定安装,并指定安装源

export VERSION=19.03
curl -fsSL "https://get.docker.com/" | bash -s -- --mirror Aliyun

```




## 2.3 配置 docker


```
mkdir -p /etc/docker/

cat>/etc/docker/daemon.json<<EOF
{
  "bip": "172.17.0.1/16",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://9jwx2023.mirror.aliyuncs.com"],
  "data-root": "/opt/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "dns-search": ["default.svc.cluster.local", "svc.cluster.local", "localdomain"],
  "dns-opts": ["ndots:2", "timeout:2", "attempts:2"]
}
EOF

```


## 2.4 启动docker


```shell
systemctl enable docker
systemctl start docker
systemctl status docker

docker info

```


# 3. 部署 kubernetes 


## 3.1 安装相关软件

> 所有软件安装都通过 yum 安装


```
# kubernetes 相关 (Master)
yum -y install tc kubelet-1.18.0 kubeadm-1.18.0 kubectl-1.18.0


# kubernetes 相关 (Node)
yum -y install tc kubelet-1.18.0 kubeadm-1.18.0


# ipvs 相关
yum -y install ipvsadm ipset 

```

```
# 配置 kubelet 自动启动 (暂时不需要启动)

systemctl enable kubelet.service

```

* 配置 kubectl 命令补全


```
# 安装 bash-completion

yum -y install bash-completion


# Linux 默认脚本路径为 /usr/share/bash-completion/bash_completion

 
# 配置 bashrc
vi ~/.bashrc

# 添加如下:


# kubectl
source /usr/share/bash-completion/bash_completion
source <(kubectl completion bash)
 

# 生效配置
source ~/.bashrc

```


## 3.2 修改证书期限

* 默认基本证书的有效期为1年


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
git checkout remotes/origin/release-1.18

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

  * 拷贝到所有的 master 中

```
# 编译后生成目录为 _output/local/bin/linux/amd64

cp _output/local/bin/linux/amd64/kubeadm /usr/bin/kubeadm


cp: overwrite ‘/usr/bin/kubeadm’? y

```


## 3.3 修改 kubeadm 配置信息

* 打印 `kubeadm` init 的 yaml 配置
 
  * `kubeadm config print init-defaults`

  * `kubeadm config print init-defaults --component-configs KubeletConfiguration` 

  * `kubeadm config print init-defaults --component-configs KubeProxyConfiguration`


```
# 导出 配置 信息

kubeadm config print init-defaults > kubeadm-init.yaml

```


* 文中配置的 `127.0.0.1` 均为后续配置的 `Nginx Api` 代理ip

  * `advertiseAddress: 10.18.77.218` 与 `bindPort: 5443` 为程序绑定的地址与端口 

  * `controlPlaneEndpoint: "127.0.0.1:6443"` 为实际访问 ApiServer 的地址

  *  这里这样配置是为了维持 Apiserver 的HA, 所以每个机器上部署一个 `Nginx` 做4层代理 ApiServer

```
# 修改相关配置，本文配置信息如下


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
  # ApiServer 程序绑定的 ip, 填写网卡实际ip
  advertiseAddress: 10.18.77.61
  # ApiServer 程序绑定的端口,修改为5443是为怕跟下面不冲突
  bindPort: 5443
nodeRegistration:
  criSocket: /var/run/dockershim.sock
  name: k8s-node-1
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
---
apiServer:
  # apiserver相关配置
  extraArgs:
    # 审计日志相关配置
    audit-log-maxage: "20"
    audit-log-maxbackup: "10"
    audit-log-maxsize: "100"
    audit-log-path: "/var/log/kube-audit/audit.log"
    audit-policy-file: "/etc/kubernetes/audit-policy.yaml"
    audit-log-format: json
  # 开启审计日志配置, 所以需要将宿主机上的审计配置
  extraVolumes:
  - name: "audit-config"
    hostPath: "/etc/kubernetes/audit-policy.yaml"
    mountPath: "/etc/kubernetes/audit-policy.yaml"
    readOnly: true
    pathType: "File"
  - name: "audit-log"
    hostPath: "/var/log/kube-audit"
    mountPath: "/var/log/kube-audit"
    pathType: "DirectoryOrCreate"
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
# Api Server 实际访问地址
controlPlaneEndpoint: "127.0.0.1:6443"
controllerManager: {}
dns:
  type: CoreDNS
etcd:
  local:
    # Etcd Volume 本地路径,最好修改为独立的磁盘
    dataDir: /var/lib/etcd
imageRepository: registry.cn-hangzhou.aliyuncs.com/google_containers
kind: ClusterConfiguration
kubernetesVersion: v1.18.0
networking:
  dnsDomain: cluster.local
  # K8s Pod ip地址的取值范围
  podSubnet: 10.253.0.0/16
  # K8s Svc ip地址的取值范围
  serviceSubnet: 10.254.0.0/16
scheduler: {}
---
# kubelet 相关配置
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
clusterDNS:
# coredns 默认ip地址
- 10.96.0.10
# 如下为 NodeLocal DNSCache 默认主机地址
#- 169.254.20.10
clusterDomain: cluster.local
---
# kube-proxy 相关配置
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  minSyncPeriod: 5s
  syncPeriod: 5s
  # 加权轮询调度
  scheduler: "wrr"

```


* 创建审计策略文件 

```
vi /etc/kubernetes/audit-policy.yaml


apiVersion: audit.k8s.io/v1 # This is required.
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  - level: RequestResponse
    resources:
    - group: ""
      resources: ["pods"]
  - level: Metadata
    resources:
    - group: ""
      resources: ["pods/log", "pods/status"]

  - level: None
    resources:
    - group: ""
      resources: ["configmaps"]
      resourceNames: ["controller-leader"]

  - level: None
    users: ["system:kube-proxy"]
    verbs: ["watch"]
    resources:
    - group: "" # core API group
      resources: ["endpoints", "services"]

  - level: None
    userGroups: ["system:authenticated"]
    nonResourceURLs:
    - "/api*" # Wildcard matching.
    - "/version"

  - level: Request
    resources:
    - group: "" # core API group
      resources: ["configmaps"]
    namespaces: ["kube-system"]

  - level: Metadata
    resources:
    - group: "" # core API group
      resources: ["secrets", "configmaps"]

  - level: Request
    resources:
    - group: "" # core API group
    - group: "extensions" # Version of group should NOT be included.

  - level: Metadata
    omitStages:
      - "RequestReceived"


```



## 3.4 配置 Nginx Proxy


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
        server 10.18.77.61:5443;
        server 10.18.77.117:5443;
        server 10.18.77.218:5443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
EOF

```

* 授权

```
# 更新权限
chmod +r /etc/nginx/nginx.conf
```



* 创建系统 `systemd.service` 文件


```
cat << EOF >> /etc/systemd/system/nginx-proxy.service
[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 127.0.0.1:6443:6443 \\
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


* 启动 Nginx Proxy

```
# 启动 Nginx

systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy
systemctl status nginx-proxy

```



## 3.5 初始化集群 

* `--upload-certs` 会在加入 master 节点的时候自动拷贝证书

```
kubeadm init --config kubeadm-init.yaml --upload-certs

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

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join 127.0.0.1:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:ed09a75d84bfbb751462262757310d0cf3d015eaa45680130be1d383245354f8 \
    --control-plane --certificate-key 93cb0d7b46ba4ac64c6ffd2e9f022cc5f22bea81acd264fb4e1f6150489cd07a

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 127.0.0.1:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:ed09a75d84bfbb751462262757310d0cf3d015eaa45680130be1d383245354f8 

```


```
# 拷贝权限文件

mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

```


```
# 查看集群状态

[root@k8s-node-1 kubeadm]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-0               Healthy   {"health":"true"} 
```


## 3.6 加入 kubernetes 集群

> 如上有 kubeadm init 后有两条 kubeadm join 命令,  --control-plane 为 加入 Master
>
> 另外token 有时效性，如果提示 token 失效，请自行创建一个新的 token.
>
> kubeadm token create --print-join-command    创建新的 join token


### 3.6.1 加入 其他 Master 节点

> 我这里三个服务器都是 Master 节点,所有都加入 --control-plane 的选项


* 创建审计策略文件

```
# 其他两台服务器创建

ssh k8s-node-2 "mkdir -p /etc/kubernetes/"

ssh k8s-node-3 "mkdir -p /etc/kubernetes/"

```


* 拷贝策略文件

```
# k8s-node-2 节点
scp /etc/kubernetes/audit-policy.yaml k8s-node-2:/etc/kubernetes/

# k8s-node-3 节点
scp /etc/kubernetes/audit-policy.yaml k8s-node-3:/etc/kubernetes/

```


* 分别 join master


```
# 先测试 api server 连通性
curl -k https://127.0.0.1:6443


# 返回如下信息:

{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {
    
  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {
    
  },
  "code": 403

```


* 增加额外的配置,用于区分不用的 master 中的 `apiserver-advertise-address` 与 `apiserver-bind-port`

```
# k8s-node-2

kubeadm join 127.0.0.1:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:ed09a75d84bfbb751462262757310d0cf3d015eaa45680130be1d383245354f8 \
    --control-plane --certificate-key 93cb0d7b46ba4ac64c6ffd2e9f022cc5f22bea81acd264fb4e1f6150489cd07a \
    --apiserver-advertise-address 10.18.77.117 \
    --apiserver-bind-port 5443


# k8s-node-3

kubeadm join 127.0.0.1:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:ed09a75d84bfbb751462262757310d0cf3d015eaa45680130be1d383245354f8 \
    --control-plane --certificate-key 93cb0d7b46ba4ac64c6ffd2e9f022cc5f22bea81acd264fb4e1f6150489cd07a \
    --apiserver-advertise-address 10.18.77.218 \
    --apiserver-bind-port 5443

```



* 拷贝 `config` 配置文件

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config


```


### 3.6.2 验证 Master 节点


> 这里 STATUS 显示 NotReady 是因为 没有安装网络组件


```
# 查看 node

[root@k8s-node-1 kubeadm]# kubectl get nodes
NAME         STATUS     ROLES    AGE     VERSION
k8s-node-1   NotReady   master   106m    v1.18.0
k8s-node-2   NotReady   master   2m18s   v1.18.0
k8s-node-3   NotReady   master   63s     v1.18.0

```



### 3.6.3 配置 Master to node

* 这里主要是让 master 直接可以运行 pods

  * 执行命令: `kubectl taint node node-name node-role.kubernetes.io/master-`

  * 禁止 master 运行pod `kubectl taint nodes node-name node-role.kubernetes.io/master=:NoSchedule`

  * 增加 `ROLES` 标签: `kubectl label nodes localhost node-role.kubernetes.io/node=`

  * 删除 `ROLES` 标签: `kubectl label nodes localhost node-role.kubernetes.io/node-`

    * `ROLES` 标签可以添加任意的值, 如: `kubectl label nodes localhost node-role.kubernetes.io/jicki=`
 
## 3.7 部署 Node 节点

* node 节点, 直接 `join` 就可以

```
kubeadm join 127.0.0.1:6443 --token abcdef.0123456789abcdef \
    --discovery-token-ca-cert-hash sha256:ed09a75d84bfbb751462262757310d0cf3d015eaa45680130be1d383245354f8

```



```
# 输出如下:

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.


```

### 3.7.1 验证 所有 节点

> 这里 STATUS 显示 NotReady 是因为 没有安装网络组件

```
[root@k8s-node-1 yaml]# kubectl get nodes
NAME         STATUS     ROLES    AGE     VERSION
k8s-node-1   NotReady   master   106m    v1.18.0
k8s-node-2   NotReady   master   2m18s   v1.18.0
k8s-node-3   NotReady   master   63s     v1.18.0
k8s-node-4   NotReady   <none>   2m46s   v1.18.0
k8s-node-5   NotReady   <none>   2m46s   v1.18.0
k8s-node-6   NotReady   <none>   2m46s   v1.18.0

```


### 3.7.2 查看验证证书

* 这里如果后续替换的话, 所有 master 节点都需要执行如下更新命令

```
# 更新证书
kubeadm alpha certs renew all

# 查看证书时间
kubeadm alpha certs check-expiration

```

```
[root@k8s-node-1 kubeadm]# kubeadm alpha certs check-expiration
CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
admin.conf                 Mar 07, 2119 06:22 UTC   98y                                     no      
apiserver                  Mar 07, 2119 06:22 UTC   98y             ca                      no      
apiserver-etcd-client      Mar 07, 2119 06:22 UTC   98y             etcd-ca                 no      
apiserver-kubelet-client   Mar 07, 2119 06:22 UTC   98y             ca                      no      
controller-manager.conf    Mar 07, 2119 06:22 UTC   98y                                     no      
etcd-healthcheck-client    Mar 07, 2119 06:22 UTC   98y             etcd-ca                 no      
etcd-peer                  Mar 07, 2119 06:22 UTC   98y             etcd-ca                 no      
etcd-server                Mar 07, 2119 06:22 UTC   98y             etcd-ca                 no      
front-proxy-client         Mar 07, 2119 06:22 UTC   98y             front-proxy-ca          no      
scheduler.conf             Mar 07, 2119 06:22 UTC   98y                                     no      

CERTIFICATE AUTHORITY   EXPIRES                  RESIDUAL TIME   EXTERNALLY MANAGED
ca                      Mar 28, 2030 04:30 UTC   9y              no      
etcd-ca                 Mar 28, 2030 04:30 UTC   9y              no      
front-proxy-ca          Mar 28, 2030 04:30 UTC   9y              no      

```


## 3.8 安装网络组件


> Flannel 网络组件
>

### 3.8.1  下载 Flannel yaml

```
# 下载 yaml 文件

wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml


```


### 3.8.2  修改 Flannel 配置

> 这里只需要修改 分配的 CIDR 就可以

```
vi kube-flannel.yml

# 修改 pods 分配的 IP 段, 与模式 vxlan

# "Type": "vxlan" , 云上一般都不支持 host-gw 模式,一般只用于 2层网络。

# 主要是如下部分

data:
  cni-conf.json: |
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
  net-conf.json: |
    {
      "Network": "10.253.0.0/16",
      "Backend": {
        "Type": "vxlan"
      }
    }
---         
```



```
# 导入 yaml 文件

[root@k8s-node-1 flannel]# kubectl apply -f kube-flannel.yml 
podsecuritypolicy.policy/psp.flannel.unprivileged created
clusterrole.rbac.authorization.k8s.io/flannel created
clusterrolebinding.rbac.authorization.k8s.io/flannel created
serviceaccount/flannel created
configmap/kube-flannel-cfg created
daemonset.apps/kube-flannel-ds-amd64 created
daemonset.apps/kube-flannel-ds-arm64 created
daemonset.apps/kube-flannel-ds-arm created
daemonset.apps/kube-flannel-ds-ppc64le created
daemonset.apps/kube-flannel-ds-s390x created

```


```
# 查看服务

[root@k8s-node-1 flannel]# kubectl get pods -n kube-system -o wide |grep kube-flannel
kube-flannel-ds-amd64-2tw6q          1/1     Running   0          88s    10.18.77.61    k8s-node-1   <none>           <none>
kube-flannel-ds-amd64-8nrtd          1/1     Running   0          88s    10.18.77.218   k8s-node-3   <none>           <none>
kube-flannel-ds-amd64-frmk9          1/1     Running   0          88s    10.18.77.117   k8s-node-2   <none>           <none>

```

* 优化 Coredns 配置

  * 根据 node 情况增加 `replicas` 数量

  * 最好可以 约束 coredns 的 pod 调度到不同的 node 中。`kubectl edit deploy coredns -n kube-system` 

```
kubectl scale deploy/coredns --replicas=3 -n kube-system

```

* 使用 `NodeLocal DNSCache`

  * 官方文档 `https://kubernetes.io/zh/docs/tasks/administer-cluster/nodelocaldns/`

  * `NodeLocal DNSCache` - 通过在集群节点上作为 `DaemonSet` 运行 dns 缓存代理来提高集群 DNS 性能。

  * `NodeLocal DNSCache` - 集群中的 Pods 将可以访问在同一节点上运行的 dns 缓存代理，从而避免了`iptables DNAT` 规则和连接跟踪。 本地缓存代理将查询 kube-dns 服务以获取集群主机名的缓存缺失（默认为 cluster.local 后缀）。 


* `NodeLocal DNSCache` 架构图

![NodeLocal][2]


* 部署 `NodeLocal DNSCache`

  * 建议在 `kubeadm init` 阶段以后就配置整体 dns

  * 如果在旧的集群部署 `NodeLocal DNSCache` 原来的所有应用组件建议重新部署,包括`网络组建`, 否则会遇到很多莫名其妙问题。

  * 如果使用 `istio` 的话, 会出现一些问题, 暂时还不兼容 `istio` , 或者是我配置上有问题。


```
# 下载 YAML

wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml



# 修改配置
sed -i 's/k8s\.gcr\.io/jicki/g' nodelocaldns.yaml

sed -i 's/__PILLAR__LOCAL__DNS__/10\.254\.0\.10/g' nodelocaldns.yaml 

sed -i 's/__PILLAR__DNS__SERVER__/169\.254\.20\.10/g' nodelocaldns.yaml

sed -i 's/__PILLAR__DNS__DOMAIN__/cluster\.local/g' nodelocaldns.yaml


# __PILLAR__DNS__SERVER__  -设置为 coredns svc 的 IP。
# __PILLAR__LOCAL__DNS__   -设置为本地链接IP（默认为169.254.20.10）。
# __PILLAR__DNS__DOMAIN__  -设置为群集域（默认为cluster.local）。



# 创建服务

[root@k8s-node-1 kubeadm]# kubectl apply -f nodelocaldns.yaml 


```

```
# 查看服务

[root@k8s-node-1 kubeadm]# kubectl get pods -n kube-system |grep node-local-dns
node-local-dns-mfxdk                 1/1     Running   0          3m12s


[root@k8s-node-1 kubeadm]# kubectl get svc -n kube-system kube-dns-upstream
NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kube-dns-upstream   ClusterIP   10.254.45.66   <none>        53/UDP,53/TCP   23m


# 查看本地开放端口
[root@k8s-node-1 kubeadm]# netstat -lan|grep 169.254.20.10
tcp        0      0 169.254.20.10:53        0.0.0.0:*               LISTEN     
udp        0      0 169.254.20.10:53        0.0.0.0:*                          

```

* 修改 kubelet 使用 `NodeLocal DNSCache`

  * kubeadm 部署的集群, kubelet 的配置在 /var/lib/kubelet/config.yaml 中


```
vi /var/lib/kubelet/config.yaml

# 修 改
clusterDNS:
- 10.254.0.10

# 修改为 本机 ip
clusterDNS:
- 169.254.20.10

```


* 重启 kubelet

  * 这里也可以在 kubeadm init 的阶段就配置好 NodeLocal 的ip

```
# 重启 kubelet 应用dns

systemctl daemon-reload && systemctl restart kubelet

```






## 3.9 检验整体集群



### 3.9.1 查看 状态

> 所有的 STATUS 都为 Ready 
>


```
[root@k8s-node-1 flannel]# kubectl get nodes
NAME         STATUS   ROLES    AGE    VERSION
k8s-node-1   Ready    master   131m   v1.18.0
k8s-node-2   Ready    master   27m    v1.18.0
k8s-node-3   Ready    master   26m    v1.18.0

```

* 查看 etcd 状态


```
# 这里目前只有一个 etcd 节点,多个节点 就写多个就可以
export ETCDCTL_API=3


# 1
etcdctl -w table \
   --endpoints=https://k8s-node-1:2379,https://k8s-node-2:2379,https://k8s-node-3:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   endpoint status


+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|        ENDPOINT         |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| https://k8s-node-1:2379 | 930e2b9d17050efd |   3.4.3 |  2.4 MB |      true |      false |         8 |      23258 |              23258 |        |
| https://k8s-node-2:2379 |  94853f1a64b6f05 |   3.4.3 |  2.4 MB |     false |      false |         8 |      23258 |              23258 |        |
| https://k8s-node-3:2379 | c4a2be5275d5ce12 |   3.4.3 |  2.4 MB |     false |      false |         8 |      23258 |              23258 |        |
+-------------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+



# 2
etcdctl -w table \
   --endpoints=https://k8s-node-1:2379,https://k8s-node-2:2379,https://k8s-node-3:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   endpoint health


+-------------------------+--------+-------------+-------+
|        ENDPOINT         | HEALTH |    TOOK     | ERROR |
+-------------------------+--------+-------------+-------+
| https://k8s-node-1:2379 |   true | 13.300955ms |       |
| https://k8s-node-3:2379 |   true |  14.65399ms |       |
| https://k8s-node-2:2379 |   true | 17.387096ms |       |
+-------------------------+--------+-------------+-------+



# 3
etcdctl -w table \
   --endpoints=https://k8s-node-1:2379,https://k8s-node-2:2379,https://k8s-node-3:2379 \
   --cacert=/etc/kubernetes/pki/etcd/ca.crt \
   --cert=/etc/kubernetes/pki/etcd/server.crt \
   --key=/etc/kubernetes/pki/etcd/server.key \
   member list


+------------------+---------+------------+---------------------------+---------------------------+------------+
|        ID        | STATUS  |    NAME    |        PEER ADDRS         |       CLIENT ADDRS        | IS LEARNER |
+------------------+---------+------------+---------------------------+---------------------------+------------+
|  94853f1a64b6f05 | started | k8s-node-2 | https://10.18.77.117:2380 | https://10.18.77.117:2379 |      false |
| 930e2b9d17050efd | started | k8s-node-1 |  https://10.18.77.61:2380 |  https://10.18.77.61:2379 |      false |
| c4a2be5275d5ce12 | started | k8s-node-3 | https://10.18.77.218:2380 | https://10.18.77.218:2379 |      false |
+------------------+---------+------------+---------------------------+---------------------------+------------+

```






### 3.9.2 查看 pods 状态



```
[root@k8s-node-1 flannel]# kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-546565776c-9zbqz             1/1     Running   0          137m
kube-system   coredns-546565776c-lz5fs             1/1     Running   0          137m
kube-system   etcd-k8s-node-1                      1/1     Running   0          138m
kube-system   etcd-k8s-node-2                      1/1     Running   0          34m
kube-system   etcd-k8s-node-3                      1/1     Running   0          33m
kube-system   kube-apiserver-k8s-node-1            1/1     Running   0          138m
kube-system   kube-apiserver-k8s-node-2            1/1     Running   0          34m
kube-system   kube-apiserver-k8s-node-3            1/1     Running   0          33m
kube-system   kube-controller-manager-k8s-node-1   1/1     Running   1          138m
kube-system   kube-controller-manager-k8s-node-2   1/1     Running   0          34m
kube-system   kube-controller-manager-k8s-node-3   1/1     Running   0          33m
kube-system   kube-flannel-ds-amd64-2tw6q          1/1     Running   0          9m11s
kube-system   kube-flannel-ds-amd64-8nrtd          1/1     Running   0          9m11s
kube-system   kube-flannel-ds-amd64-frmk9          1/1     Running   0          9m11s
kube-system   kube-proxy-9qv4l                     1/1     Running   0          34m
kube-system   kube-proxy-f29dk                     1/1     Running   0          137m
kube-system   kube-proxy-zgjnf                     1/1     Running   0          33m
kube-system   kube-scheduler-k8s-node-1            1/1     Running   1          138m
kube-system   kube-scheduler-k8s-node-2            1/1     Running   0          34m
kube-system   kube-scheduler-k8s-node-3            1/1     Running   0          33m
```


### 3.9.3 查看 svc 的状态

```
[root@k8s-node-1 flannel]# kubectl get svc --all-namespaces
NAMESPACE     NAME         TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                  AGE
default       kubernetes   ClusterIP   10.254.0.1    <none>        443/TCP                  138m
kube-system   kube-dns     ClusterIP   10.254.0.10   <none>        53/UDP,53/TCP,9153/TCP   138m

```


### 3.9.3 查看 IPVS 的状态


```
[root@k8s-node-1 flannel]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.254.0.1:443 wrr
  -> 10.18.77.61:5443             Masq    1      2          0         
  -> 10.18.77.117:5443            Masq    1      0          0         
  -> 10.18.77.218:5443            Masq    1      0          0         
TCP  10.254.0.10:53 wrr
  -> 10.254.64.3:53               Masq    1      0          0         
  -> 10.254.65.4:53               Masq    1      0          0         
TCP  10.254.0.10:9153 wrr
  -> 10.254.64.3:9153             Masq    1      0          0         
  -> 10.254.65.4:9153             Masq    1      0          0         
TCP  10.254.28.93:80 wrr
  -> 10.254.65.5:80               Masq    1      0          1         
  -> 10.254.66.3:80               Masq    1      0          2         
UDP  10.254.0.10:53 wrr
  -> 10.254.64.3:53               Masq    1      0          0         
  -> 10.254.65.4:53               Masq    1      0          0  

```

# 4.  测试集群


## 4.1 创建一个 nginx deployment

```
apiVersion: apps/v1
kind: Deployment 
metadata: 
  name: nginx-dm
  labels:
    app: nginx
spec: 
  replicas: 3
  strategy:
    # 配置滚动升级策略
    type: RollingUpdate
    rollingUpdate:
      # 生成1个新的pod完成后再删除1个旧的pod
      maxSurge: 1
      # 设置最多容忍2个pods处于无法提供服务的状态
      maxUnavailable: 2
  # 控制 pod 处于就绪状态的观察时间
  # pod 在这段时间内都正常运行, 才认为新 pod 可用, 将老的 pod 删除掉。
  minReadySeconds: 120
  selector:
    matchLabels:
      app: nginx
  template: 
    metadata: 
      labels: 
        app: nginx 
        version: v1.0.0
    spec: 
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80
              name: http
          # 资源的限制
          resources:
            limits:
              cpu: 1000m
              memory: 500Mi
            requests:
              # 1 cpu = 1000m
              cpu: 0.5
              memory: 250Mi 
          volumeMounts:
            - name: tz-config
              mountPath: /etc/localtime
              readOnly: true
          # readinessProbe - 检测pod 的 Ready 是否为 true
          # 就绪探针 如果探针判断失败,则不会有流量发往到这个pod。 
          readinessProbe:
            tcpSocket:
              port: 80
            # 启动后5s 开始检测
            initialDelaySeconds: 5  
            # 检测 间隔为 10s
            periodSeconds: 10
            # 探针探测失败后, 最少连续探测成功多少次才被认定为成功 
            successThreshold: 1
            # 探测成功后, 最少连续探测失败多少次才被认定为失败
            failureThreshold: 1
          # livenessProbe - 检测 pod 的 State 是否为 Running
          # 活性探测 如果探针判断失败, 则会重启这个 pod。
          livenessProbe:
            httpGet:
              path: /
              port: 80
            # 启动后 15s 开始检测
            # 检测时间必须在 readinessProbe 之后
            initialDelaySeconds: 15
            # 检测 间隔为 20s
            periodSeconds: 20
            # 探针探测失败后, 最少连续探测成功多少次才被认定为成功
            successThreshold: 1
            # 探测成功后, 最少连续探测失败多少次才被认定为失败
            failureThreshold: 3
      volumes:
        - name: tz-config
          hostPath:
            path: /etc/localtime
---

apiVersion: v1 
kind: Service
metadata: 
  name: nginx-svc 
  labels:
    app: nginx
spec: 
  ports: 
    - port: 80
      name: http
      targetPort: 80
      protocol: TCP 
  selector: 
    app: nginx
```


```
# 导入文件

[root@k8s-node-1 kubeadm]# kubectl apply -f nginx-deployment.yaml
deployment.apps/nginx-dm created
service/nginx-svc created

```



```
# 查看服务
[root@k8s-node-1 kubeadm]# kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dm-8665b6b679-lf72f   1/1     Running   0          37s
nginx-dm-8665b6b679-mqn5f   1/1     Running   0          37s


# 查看 svc
[root@k8s-node-1 kubeadm]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE    SELECTOR
kubernetes   ClusterIP   10.254.0.1      <none>        443/TCP   146m   <none>
nginx-svc    ClusterIP   10.254.23.158   <none>        80/TCP    54s    name=nginx


```


* 访问 svc 与 

```
#  node-1 访问 svc

[root@k8s-node-1 yaml]# curl 10.254.28.93
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

[root@k8s-node-2 ~]# curl 10.254.28.93
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
TCP  10.254.0.1:443 wrr
  -> 10.18.77.61:5443             Masq    1      2          0         
  -> 10.18.77.117:5443            Masq    1      0          0         
  -> 10.18.77.218:5443            Masq    1      0          0         
TCP  10.254.0.10:53 wrr
  -> 10.254.64.3:53               Masq    1      0          0         
  -> 10.254.65.4:53               Masq    1      0          0         
TCP  10.254.0.10:9153 wrr
  -> 10.254.64.3:9153             Masq    1      0          0         
  -> 10.254.65.4:9153             Masq    1      0          0         
TCP  10.254.28.93:80 wrr
  -> 10.254.65.5:80               Masq    1      0          10        
  -> 10.254.66.3:80               Masq    1      0          10        
UDP  10.254.0.10:53 wrr
  -> 10.254.64.3:53               Masq    1      0          0         
  -> 10.254.65.4:53               Masq    1      0          0 

```



## 4.2 验证 dns 的服务


```
# 测试
[root@k8s-node-1 kubeadm]# kubectl get pods
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dm-8665b6b679-28zbw   1/1     Running   0          7m54s
nginx-dm-8665b6b679-h5rhn   1/1     Running   0          7m54s


# kubernetes 服务

[root@k8s-node-1 kubeadm]# kubectl exec -it nginx-dm-8665b6b679-28zbw -- nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.254.0.1 kubernetes.default.svc.cluster.local


# nginx-svc 服务

[root@k8s-node-1 kubeadm]# kubectl exec -it nginx-dm-8665b6b679-28zbw -- nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.27.199 nginx-svc.default.svc.cluster.local

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
        imagePullPolicy: IfNotPresent
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
        kubernetes.io/arch: "amd64"
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
serviceaccount/metrics-server unchanged
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


* 查看 `pods` 的信息

```
[root@k8s-node-1 metrics]# kubectl top pods -n kube-system
NAME                                 CPU(cores)   MEMORY(bytes)   
coredns-546565776c-9zbqz             2m           5Mi             
coredns-546565776c-lz5fs             2m           5Mi             
etcd-k8s-node-1                      27m          75Mi            
etcd-k8s-node-2                      25m          76Mi            
etcd-k8s-node-3                      23m          75Mi            
kube-apiserver-k8s-node-1            21m          272Mi           
kube-apiserver-k8s-node-2            19m          277Mi           
kube-apiserver-k8s-node-3            23m          279Mi           
kube-controller-manager-k8s-node-1   12m          37Mi            
kube-controller-manager-k8s-node-2   2m           12Mi            
kube-controller-manager-k8s-node-3   2m           12Mi            
kube-flannel-ds-amd64-f2ck7          2m           8Mi             
kube-flannel-ds-amd64-g6tp6          2m           8Mi             
kube-flannel-ds-amd64-z2cvb          2m           9Mi             
kube-proxy-9qv4l                     12m          9Mi             
kube-proxy-f29dk                     11m          9Mi             
kube-proxy-zgjnf                     10m          9Mi             
kube-scheduler-k8s-node-1            3m           9Mi             
kube-scheduler-k8s-node-2            2m           8Mi             
kube-scheduler-k8s-node-3            2m           10Mi            
metrics-server-7ff8dccd5b-jsjkk      2m           13Mi            

```

* 查看 `node` 信息

```
[root@k8s-node-1 metrics]# kubectl top nodes
NAME         CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%   
k8s-node-1   110m         5%     1100Mi          28%       
k8s-node-2   97m          4%     1042Mi          27%       
k8s-node-3   94m          4%     1028Mi          26% 
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
# 修改 副本数
spec:
  replicas: 2


# 配置 node affinity 
# 配置 hostNetwork
# 配置 dnsPolicy: ClusterFirstWithHostNet


# 在 如下之间添加
    spec:
      serviceAccountName: nginx-ingress-serviceaccount


# 添加完如下:
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/hostname
                operator: In
                values:
                - k8s-node-2
                - k8s-node-3
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
                - k8s-node-2
                - k8s-node-3
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


* 添加一个 svc 用于解决如下错误问题
  * `err services "ingress-nginx" not found `


```
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
spec:
  type: ClusterIP
  ports:
  - name: http
    port: 80
    targetPort: 80
    protocol: TCP
  - name: https
    port: 443
    targetPort: 443
    protocol: TCP
  selector:
    app: ingress-nginx
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
limitrange/ingress-nginx created

```


### 6.2.4 查看服务状态


```
[root@k8s-node-1 ingress]# kubectl get pods -n ingress-nginx -o wide
NAME                                        READY   STATUS    RESTARTS   AGE     IP             NODE         NOMINATED NODE   READINESS GATES
nginx-ingress-controller-5d5b986984-lxsng   1/1     Running   0          2m16s   10.18.77.218   k8s-node-3   <none>           <none>
nginx-ingress-controller-5d5b986984-t8tvx   1/1     Running   0          53s     10.18.77.117   k8s-node-2   <none>           <none>
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
[root@k8s-node-1 kubeadm]# kubectl apply -f nginx-ingress.yaml 
ingress.extensions/nginx-ingress created


# 查看 ingress
[root@k8s-node-1 kubeadm]# kubectl get ingress
NAME            CLASS    HOSTS            ADDRESS   PORTS   AGE
nginx-ingress   <none>   nginx.jicki.cn             80      34s


```




### 6.2.6 测试访问


```
[root@k8s-node-1 kubeadm]# curl -I nginx.jicki.cn
HTTP/1.1 200 OK
Server: nginx/1.17.8
Date: Mon, 30 Mar 2020 08:54:56 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Vary: Accept-Encoding
Last-Modified: Tue, 03 Mar 2020 17:36:53 GMT
ETag: "5e5e95b5-264"
Accept-Ranges: bytes

```

# 7. Dashboard

> 官方 https://github.com/kubernetes/dashboard


## 7.1 Dashboard 介绍

> Dashboard 是 Kubernetes 集群的 通用 WEB UI 
> 它允许用户管理集群中运行的应用程序并对其进行故障排除，以及管理集群本身。


## 7.2 部署 Dashboard

> 注意 dashboard 1.10.x 版本 不支持 kubernetes 1.16.x 以上的必须使用 2.0 版本否则报错
> 
> 404 the server could not find the requested resource 
> 
> 目前 Dashboard 已经进入 rc6 阶段


### 7.2.1 下载 yaml 文件


```
# 下载 yaml 文件

https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-rc6/aio/deploy/recommended.yaml

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
[root@k8s-node-1 dashboard]# kubectl get pods -n kubernetes-dashboard |grep dashboard
dashboard-metrics-scraper-779f5454cb-8m5p5   1/1     Running   0          19s
kubernetes-dashboard-64686c4bf9-bwvvj        1/1     Running   0          19s


# svc 服务
[root@k8s-node-1 dashboard]# kubectl get svc -n kubernetes-dashboard |grep dashboard
dashboard-metrics-scraper   ClusterIP   10.254.39.66    <none>        8000/TCP   43s
kubernetes-dashboard        ClusterIP   10.254.53.202   <none>        443/TCP    44s

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

```


* 部署好 dashboard 以后会生成一个 自签的证书

  * `kubernetes-dashboard-certs` 后面 ingress 会使用到这个证书

```
[root@k8s-node-1 dashboard]# kubectl get secret -n kubernetes-dashboard
NAME                               TYPE                                  DATA   AGE
default-token-nnn5x                kubernetes.io/service-account-token   3      6m32s
kubernetes-dashboard-certs         Opaque                                0      6m32s
kubernetes-dashboard-csrf          Opaque                                1      6m32s
kubernetes-dashboard-key-holder    Opaque                                2      6m32s
kubernetes-dashboard-token-7plmf   kubernetes.io/service-account-token   3      6m32s

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
    secretName: kubernetes-dashboard-certs
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
NAME                   CLASS    HOSTS                ADDRESS   PORTS     AGE
kubernetes-dashboard   <none>   dashboard.jicki.cn             80, 443   2m53s

```


### 7.2.6 测试访问


```
[root@k8s-node-1 dashboard]# curl -I -k https://dashboard.jicki.cn
HTTP/2 200 
server: nginx/1.17.8
date: Mon, 30 Mar 2020 09:41:02 GMT
content-type: text/html; charset=utf-8
content-length: 1287
vary: Accept-Encoding
accept-ranges: bytes
cache-control: no-store
last-modified: Fri, 13 Mar 2020 13:43:54 GMT
strict-transport-security: max-age=15724800; includeSubDomains

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



### 7.2.8

> 浏览器访问

![图1][1]







## FAQ

* `Failed to get system container stats for "/system.slice/docker.service": failed to get cgroup stats` 错误 

* 推测是由于 kubernetes 版本与 docker 版本不兼容导致的问题

```
# 打开10-kuberadm.conf 文件
vi /usr/lib/systemd/system/kubelet.service.d/10-kubeadm.conf

# 添加如下:

Environment="KUBELET_CGROUP_ARGS=--cgroup-driver=systemd --runtime-cgroups=/systemd/system.slice --kubelet-cgroups=/systemd/system.slice"
```

```
# 加载配置
systemctl daemon-reload

# 重启 kubelet
systemctl restart kubelet
```


### 修改 node 名称


```
vi /var/lib/kubelet/kubeadm-flags.env

# 修改其中的 --hostname-override= 变量


# 重启 kubelet

systemctl daemon-reload 

systemctl restart kubelet


# 删除旧的 node

kubectl delete no nod-name


# 查看 csr

[root@k8s-node-1 kubeadm]# kubectl get csr
NAME        AGE   SIGNERNAME                                    REQUESTOR               CONDITION
csr-nzhlq   17s   kubernetes.io/kube-apiserver-client-kubelet   system:node:localhost   Pending


# 通过 csr
[root@k8s-node-1 kubeadm]# kubectl certificate approve csr-nzhlq 



# 通过以后再查看 node
[root@k8s-node-1 kubeadm]# kubectl get nodes
NAME         STATUS     ROLES    AGE   VERSION
k8s-node-1   NotReady   <none>   8s    v1.18.0


# 需要等待一段时间等待状态
[root@k8s-node-1 kubeadm]# kubectl get nodes
NAME         STATUS   ROLES    AGE   VERSION
k8s-node-1   Ready    <none>   63s   v1.18.0

```

  [1]: https://jicki.cn/img/posts/dashboard/dashboard-jicki.png
  [2]: https://jicki.cn/img/posts/dashboard/nodelocaldns.jpg

