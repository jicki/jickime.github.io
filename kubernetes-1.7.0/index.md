# kubernetes 1.7.0 + flannel 二进制部署



# kubernetes 1.7.0 + flannel

> 基于 二进制 文件部署
> 本地化 kube-apiserver, kube-controller-manager , kube-scheduler


## 环境说明

```
k8s-master-1: 10.6.0.140
k8s-master-2: 10.6.0.187
k8s-node-1:   10.6.0.188
```


## 初始化环境

```
hostnamectl --static set-hostname hostname

10.6.0.140 - k8s-master-1
10.6.0.187 - k8s-master-2
10.6.0.188 - k8s-node-1
```



```
#编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts

10.6.0.140 k8s-master-1
10.6.0.187 k8s-master-2
10.6.0.188 k8s-node-1
```


# 创建 验证

> 这里使用 CloudFlare 的 PKI 工具集 cfssl 来生成 Certificate Authority (CA) 证书和秘钥文件。


## 安装 cfssl


```
mkdir -p /opt/local/cfssl

cd /opt/local/cfssl

wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
mv cfssl_linux-amd64 cfssl

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
mv cfssljson_linux-amd64 cfssljson

wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64
mv cfssl-certinfo_linux-amd64 cfssl-certinfo

chmod +x *
```


## 创建 CA 证书配置

```
mkdir /opt/ssl

cd /opt/ssl

/opt/local/cfssl/cfssl print-defaults config > config.json

/opt/local/cfssl/cfssl print-defaults csr > csr.json


```


```
# config.json 文件

{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}

```



```
# csr.json 文件

{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}


```

## 生成 CA 证书和私钥

```

cd /opt/ssl/

/opt/local/cfssl/cfssl gencert -initca csr.json | /opt/local/cfssl/cfssljson -bare ca


[root@k8s-master-1 ssl]# ls -lt
总用量 20
-rw-r--r-- 1 root root 1005 7月   3 17:26 ca.csr
-rw------- 1 root root 1675 7月   3 17:26 ca-key.pem
-rw-r--r-- 1 root root 1363 7月   3 17:26 ca.pem
-rw-r--r-- 1 root root  210 7月   3 17:24 csr.json
-rw-r--r-- 1 root root  292 7月   3 17:23 config.json

```


## 分发证书


```
# 创建证书目录
mkdir -p /etc/kubernetes/ssl

# 拷贝所有文件到目录下
cp * /etc/kubernetes/ssl

# 这里要将文件拷贝到所有的k8s 机器上

scp * 10.6.0.187:/etc/kubernetes/ssl/

scp * 10.6.0.188:/etc/kubernetes/ssl/
```






# etcd 集群

> etcd 是k8s集群的基础组件，这里感觉没必要创建双向认证。


## 安装 etcd

```
yum -y install etcd3
```


## 修改 etcd 配置

```
# etcd-1

# 修改配置文件，/etc/etcd/etcd.conf 需要修改如下参数：

mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf-bak

vi /etc/etcd/etcd.conf


ETCD_NAME=etcd1
ETCD_DATA_DIR="/var/lib/etcd/etcd1.etcd"
ETCD_LISTEN_PEER_URLS="http://10.6.0.140:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.6.0.140:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.6.0.140:2380"
ETCD_INITIAL_CLUSTER="etcd1=http://10.6.0.140:2380,etcd2=http://10.6.0.187:2380,etcd3=http://10.6.0.188:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.6.0.140:2379"

```

```
# etcd-2

# 修改配置文件，/etc/etcd/etcd.conf 需要修改如下参数：

mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf-bak

vi /etc/etcd/etcd.conf

ETCD_NAME=etcd2
ETCD_DATA_DIR="/var/lib/etcd/etcd2.etcd"
ETCD_LISTEN_PEER_URLS="http://10.6.0.187:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.6.0.187:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.6.0.187:2380"
ETCD_INITIAL_CLUSTER="etcd1=http://10.6.0.140:2380,etcd2=http://10.6.0.187:2380,etcd3=http://10.6.0.188:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.6.0.187:2379"

```


```
# etcd-3

# 修改配置文件，/etc/etcd/etcd.conf 需要修改如下参数：

mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf-bak

vi /etc/etcd/etcd.conf


ETCD_NAME=etcd3
ETCD_DATA_DIR="/var/lib/etcd/etcd3.etcd"
ETCD_LISTEN_PEER_URLS="http://10.6.0.188:2380"
ETCD_LISTEN_CLIENT_URLS="http://10.6.0.188:2379,http://127.0.0.1:2379"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.6.0.188:2380"
ETCD_INITIAL_CLUSTER="etcd1=http://10.6.0.140:2380,etcd2=http://10.6.0.187:2380,etcd3=http://10.6.0.188:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="http://10.6.0.188:2379"

```



> 修改 etcd 启动文件 /usr/lib/systemd/system/etcd.service

```
sed -i 's/\\\"${ETCD_LISTEN_CLIENT_URLS}\\\"/\\\"${ETCD_LISTEN_CLIENT_URLS}\\\" --listen-client-urls=\\\"${ETCD_LISTEN_CLIENT_URLS}\\\" --advertise-client-urls=\\\"${ETCD_ADVERTISE_CLIENT_URLS}\\\" --initial-cluster-token=\\\"${ETCD_INITIAL_CLUSTER_TOKEN}\\\" --initial-cluster=\\\"${ETCD_INITIAL_CLUSTER}\\\" --initial-cluster-state=\\\"${ETCD_INITIAL_CLUSTER_STATE}\\\"/g' /usr/lib/systemd/system/etcd.service

``` 

## 启动 etcd

> 分别启动 所有节点的 etcd 服务

```
systemctl enable etcd

systemctl start etcd

systemctl status etcd
```


## 验证 etcd 集群状态

> 查看 etcd 集群状态：

```
etcdctl cluster-health

# 出现 cluster is healthy 表示成功
```


> 查看 etcd 集群成员：

```
etcdctl member list


4cccc71dfdfc0646: name=etcd1 peerURLs=http://10.6.0.140:2380 clientURLs=http://10.6.0.140:2379 isLeader=true
5ffd5d99530e9fe6: name=etcd2 peerURLs=http://10.6.0.187:2380 clientURLs=http://10.6.0.187:2379 isLeader=false
dcd6fb81996f77c3: name=etcd3 peerURLs=http://10.6.0.188:2380 clientURLs=http://10.6.0.188:2379 isLeader=false

```


# Flannel 网络


## 安装 flannel 

> 这边其实由于内网，就没有使用SSL认证，直接使用了

```
yum -y install flannel
```


> 清除网络中遗留的docker 网络 (docker0, flannel0 等)

```
ifconfig
```

> 如果存在 请删除之，以免发生不必要的未知错误

```
ip link delete docker0
....

```


## 配置 flannel

> 设置 flannel 所用到的IP段

```
etcdctl --endpoint http://10.6.0.140:2379 set /flannel/network/config '{"Network":"10.233.0.0/16","SubnetLen":25,"Backend":{"Type":"vxlan","VNI":1}}'
```


> 接下来修改 flannel 配置文件

```
vim /etc/sysconfig/flanneld

# 旧版本:
FLANNEL_ETCD="http://10.6.0.140:2379,http://10.6.0.187:2379,http://10.6.0.188:2379"   # 修改为 集群地址
FLANNEL_ETCD_KEY="/flannel/network/config"        # 修改为 上面导入配置中的  /flannel/network
FLANNEL_OPTIONS="--iface=em1"                     # 修改为 本机物理网卡的名称


# 新版本:
FLANNEL_ETCD="http://10.6.0.140:2379,http://10.6.0.187:2379,http://10.6.0.188:2379"   # 修改为 集群地址
FLANNEL_ETCD_PREFIX="/flannel/network"        # 修改为 上面导入配置中的  /flannel/network
FLANNEL_OPTIONS="--iface=em1"                     # 修改为 本机物理网卡的名称

```


## 启动 flannel

```
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```
 
 

# 安装 docker


```
# 导入 yum 源

# 安装 yum-config-manager

yum -y install yum-utils

# 导入
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo


# 更新 repo
yum makecache

# 安装

yum install docker-ce
```
 

## 更改docker 配置

```
# 修改配置

vi /usr/lib/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS $DOCKER_OPTS $DOCKER_DNS_OPTIONS
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target

```

```
# 修改其他配置

cat >> /usr/lib/systemd/system/docker.service.d/docker-options.conf << EOF
[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.254.0.0/16 --graph=/opt/docker --registry-mirror=http://b438f72b.m.daocloud.io"
EOF

```





```
# 重新读取配置，启动 docker 
systemctl daemon-reload
systemctl start docker
```
 
 

## 查看docker网络

```
ifconfig

docker0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
        inet 10.233.19.1  netmask 255.255.255.128  broadcast 0.0.0.0
        ether 02:42:c1:2c:c5:be  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

em1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.6.0.140  netmask 255.255.255.0  broadcast 10.6.0.255
        inet6 fe80::d6ae:52ff:fed1:f0c9  prefixlen 64  scopeid 0x20<link>
        ether d4:ae:52:d1:f0:c9  txqueuelen 1000  (Ethernet)
        RX packets 16286600  bytes 1741928233 (1.6 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 15841272  bytes 1566357399 (1.4 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

flannel.1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.233.19.0  netmask 255.255.255.255  broadcast 0.0.0.0
        inet6 fe80::d9:e2ff:fe46:9cdd  prefixlen 64  scopeid 0x20<link>
        ether 02:d9:e2:46:9c:dd  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 26 overruns 0  carrier 0  collisions 0


```







# 安装 kubectl 工具


## Master 端


```
# 首先安装 kubectl

wget https://dl.k8s.io/v1.7.0/kubernetes-client-linux-amd64.tar.gz

tar -xzvf kubernetes-client-linux-amd64.tar.gz

cp kubernetes/client/bin/* /usr/local/bin/

chmod a+x /usr/local/bin/kube*


# 验证安装

kubectl version
Client Version: version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.0", GitCommit:"d3ada0119e776222f11ec7945e6d860061339aad", GitTreeState:"clean", BuildDate:"2017-06-29T23:15:59Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}

```

## 创建 admin 证书

> kubectl 与 kube-apiserver 的安全端口通信，需要为安全通信提供 TLS 证书和秘钥。


```
cd /opt/ssl/

vi admin-csr.json


{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

```


```
# 生成 admin 证书和私钥
cd /opt/ssl/

/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/config.json \
  -profile=kubernetes admin-csr.json | /opt/local/cfssl/cfssljson -bare admin


# 查看生成

[root@k8s-master-1 ssl]# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem

cp admin*.pem /etc/kubernetes/ssl/

```


## 配置 kubectl kubeconfig 文件

```
# 配置 kubernetes 集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://10.6.0.140:6443


# 配置 客户端认证

kubectl config set-credentials admin \
  --client-certificate=/etc/kubernetes/ssl/admin.pem \
  --embed-certs=true \
  --client-key=/etc/kubernetes/ssl/admin-key.pem
  


kubectl config set-context kubernetes \
  --cluster=kubernetes \
  --user=admin


kubectl config use-context kubernetes

```


## 分发 kubectl config 文件

```
# 将上面配置的 kubeconfig 文件分发到其他机器

# 其他服务器创建目录

mkdir /root/.kube

scp /root/.kube/config 10.6.0.187:/root/.kube/

scp /root/.kube/config 10.6.0.188:/root/.kube/

```



# 部署 kubernetes Master 节点

> Master 需要部署 kube-apiserver , kube-scheduler , kube-controller-manager 这三个组件。



## 安装 组件


```
# 从github 上下载版本

cd /tmp

wget https://dl.k8s.io/v1.7.0/kubernetes-server-linux-amd64.tar.gz

tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/

```


## 创建 kubernetes 证书

```
/opt/ssl

vi kubernetes-csr.json

{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.6.0.140",
    "10.254.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}


## 这里 hosts 字段中 三个 IP 分别为 127.0.0.1 本机， 10.6.0.140 为 Master 的IP， 10.254.0.1 为 kubernetes SVC 的 IP， 一般是 部署网络的第一个IP , 如: 10.254.0.1 ， 在启动完成后，我们使用   kubectl get svc ， 就可以查看到

```


## 生成 kubernetes 证书和私钥


```
/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/config.json \
  -profile=kubernetes kubernetes-csr.json | /opt/local/cfssl/cfssljson -bare kubernetes

# 查看生成

[root@k8s-master-1 ssl]# ls -lt kubernetes*
-rw-r--r-- 1 root root 1245 7月   4 11:25 kubernetes.csr
-rw------- 1 root root 1679 7月   4 11:25 kubernetes-key.pem
-rw-r--r-- 1 root root 1619 7月   4 11:25 kubernetes.pem
-rw-r--r-- 1 root root  436 7月   4 11:23 kubernetes-csr.json


# 拷贝到目录
cp -r kubernetes* /etc/kubernetes/ssl/

```

## 配置 kube-apiserver

> kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token  一致，如果一致则自动为 kubelet生成证书和秘钥。


```
# 生成 token

[root@k8s-master-1 ssl]# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
11849e4f70904706ab3e631e70e6af0d


# 创建 token.csv 文件

/opt/ssl

vi token.csv

11849e4f70904706ab3e631e70e6af0d,kubelet-bootstrap,10001,"system:kubelet-bootstrap"


# 拷贝

cp token.csv /etc/kubernetes/

```


## 创建 kube-apiserver.service 文件

```
一、 开启了 RBAC 

# 自定义 系统 service 文件一般存于 /etc/systemd/system/ 下

vi /etc/systemd/system/kube-apiserver.service

[Unit]
Description=kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --advertise-address=10.6.0.140 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/lib/audit.log \
  --authorization-mode=RBAC \
  --bind-address=10.6.0.140 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=http://10.6.0.140:2379,http://10.6.0.187:2379,http://10.6.0.188:2379 \
  --event-ttl=1h \
  --kubelet-https=true \
  --insecure-bind-address=10.6.0.140 \
  --runtime-config=rbac.authorization.k8s.io/v1alpha1 \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=30000-32000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --experimental-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/token.csv \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```

```
二、 关闭了 RBAC 

# 自定义 系统 service 文件一般存于 /etc/systemd/system/ 下

vi /etc/systemd/system/kube-apiserver.service

[Unit]
Description=kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --advertise-address=10.6.0.140 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/lib/audit.log \
  --bind-address=10.6.0.140 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/kubernetes.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/kubernetes-key.pem \
  --etcd-servers=http://10.6.0.140:2379,http://10.6.0.187:2379,http://10.6.0.188:2379 \
  --event-ttl=1h \
  --kubelet-https=true \
  --insecure-bind-address=10.6.0.140 \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=30000-32000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --experimental-bootstrap-token-auth \
  --token-auth-file=/etc/kubernetes/token.csv \
  --v=2
Restart=on-failure
RestartSec=5
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target




```

```
# 这里面要注意的是 --service-node-port-range=30000-32000
# 这个地方是 映射外部端口时 的端口范围，随机映射也在这个范围内映射，指定映射端口必须也在这个范围内。
```





 
## 启动 kube-apiserver

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver

```


## 配置 kube-controller-manager


```
# 创建 kube-controller-manager.service 文件

vi /etc/systemd/system/kube-controller-manager.service


[Unit]
Description=kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://10.6.0.140:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.254.0.0/16 \
  --cluster-cidr=10.233.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target


```



## 启动 kube-controller-manager


```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```


## 配置 kube-scheduler

```
# 创建 kube-cheduler.service 文件

vi /etc/systemd/system/kube-scheduler.service


[Unit]
Description=kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --address=127.0.0.1 \
  --master=http://10.6.0.140:8080 \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```

## 启动 kube-scheduler

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler

```
 

## 验证 Master 节点

```
[root@k8s-master-1 opt]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"} 

```
 


# 部署 kubernetes Node 节点

```

 (首先部署 10.6.0.187)
 
```

> Node 节点 需要部署的组件有 docker  flannel  kubectl  kubelet  kube-proxy 这几个组件。



## 配置 kubectl

```
wget https://dl.k8s.io/v1.7.0/kubernetes-client-linux-amd64.tar.gz

tar -xzvf kubernetes-client-linux-amd64.tar.gz

cp kubernetes/client/bin/* /usr/local/bin/

chmod a+x /usr/local/bin/kube*


# 验证安装

kubectl version
Client Version: version.Info{Major:"1", Minor:"7", GitVersion:"v1.7.0", GitCommit:"d3ada0119e776222f11ec7945e6d860061339aad", GitTreeState:"clean", BuildDate:"2017-06-29T23:15:59Z", GoVersion:"go1.8.3", Compiler:"gc", Platform:"linux/amd64"}

```




## 配置 kubelet

> kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求(certificatesigningrequests)。


```
# 先创建认证请求
# user 为 master 中 token.csv 文件里配置的用户
# 只需在一个node中创建一次就可以

kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

```

## 下载 二进制文件

```
cd /tmp

wget https://dl.k8s.io/v1.7.0/kubernetes-server-linux-amd64.tar.gz

tar zxvf kubernetes-server-linux-amd64.tar.gz 

cp -r kubernetes/server/bin/{kube-proxy,kubelet} /usr/local/bin/

```

## 创建 kubelet kubeconfig 文件

```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://10.6.0.140:6443 \
  --kubeconfig=bootstrap.kubeconfig

# 配置客户端认证

kubectl config set-credentials kubelet-bootstrap \
  --token=11849e4f70904706ab3e631e70e6af0d \
  --kubeconfig=bootstrap.kubeconfig


# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
  
  
# 配置默认关联
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig

# 拷贝生成的 bootstrap.kubeconfig 文件

mv bootstrap.kubeconfig /etc/kubernetes/

```


## 创建 kubelet.service 文件

```
# 创建 kubelet 目录

mkdir /var/lib/kubelet

vi /etc/systemd/system/kubelet.service


[Unit]
Description=kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --address=10.6.0.187 \
  --hostname-override=10.6.0.187 \
  --pod-infra-container-image=jicki/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --require-kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=10.254.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --v=2
ExecStopPost=/sbin/iptables -A INPUT -s 10.0.0.0/8 -p tcp --dport 4194 -j ACCEPT
ExecStopPost=/sbin/iptables -A INPUT -s 172.16.0.0/12 -p tcp --dport 4194 -j ACCEPT
ExecStopPost=/sbin/iptables -A INPUT -s 192.168.0.0/16 -p tcp --dport 4194 -j ACCEPT
ExecStopPost=/sbin/iptables -A INPUT -p tcp --dport 4194 -j DROP
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```


```
# 如上配置:
10.6.0.187       为本机的IP
10.254.0.2       预分配的 dns 地址
cluster.local.   为 kubernetes 集群的 domain
jicki/pause-amd64:3.0  这个是 pod 的基础镜像，既 gcr 的 gcr.io/google_containers/pause-amd64:3.0 镜像， 下载下来修改为自己的仓库中的比较快。
```

## 启动 kubelet

```

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet

```


## 配置 TLS 认证

```
# 查看 csr 的名称

[root@k8s-master-2 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-EUE41uO5bofZZ-7GKD_V31oHXsENKFXCkLPy6Dj35Sc   1m        kubelet-bootstrap   Pending


# 增加 认证

kubectl certificate approve node-csr-EUE41uO5bofZZ-7GKD_V31oHXsENKFXCkLPy6Dj35Sc

# 提示
certificatesigningrequest "node-csr-EUE41uO5bofZZ-7GKD_V31oHXsENKFXCkLPy6Dj35Sc" approved

```




## 验证 nodes

```
[root@k8s-master-1 ~]# kubectl get nodes
NAME         STATUS    AGE       VERSION
10.6.0.187   Ready     33s       v1.7.0


# 成功以后会自动生成配置文件与密钥

# 配置文件

ls /etc/kubernetes/kubelet.kubeconfig   
/etc/kubernetes/kubelet.kubeconfig


# 密钥文件

ls /etc/kubernetes/ssl/kubelet*
/etc/kubernetes/ssl/kubelet-client.crt  /etc/kubernetes/ssl/kubelet.crt
/etc/kubernetes/ssl/kubelet-client.key  /etc/kubernetes/ssl/kubelet.key

```


## 配置 kube-proxy


## 创建 kube-proxy 证书

```
# 证书方面由于我们node端没有装 cfssl
# 我们回到 master 端 机器 去配置证书，然后拷贝过来

[root@k8s-master-1 ~]# cd /opt/ssl


vi kube-proxy-csr.json

{
  "CN": "system:kube-proxy",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "ShenZhen",
      "L": "ShenZhen",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

```

## 生成 kube-proxy 证书和私钥

```
/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/etc/kubernetes/ssl/config.json \
  -profile=kubernetes  kube-proxy-csr.json | /opt/local/cfssl/cfssljson -bare kube-proxy
  
# 查看生成
ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem

# 拷贝到目录
cp kube-proxy*.pem /etc/kubernetes/ssl/

```

## 拷贝到Node节点

```
scp kube-proxy*.pem 10.6.0.187:/etc/kubernetes/ssl/

scp kube-proxy*.pem 10.6.0.188:/etc/kubernetes/ssl/

```


## 创建 kube-proxy kubeconfig 文件

```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://10.6.0.140:6443 \
  --kubeconfig=kube-proxy.kubeconfig


# 配置客户端认证

kubectl config set-credentials kube-proxy \
  --client-certificate=/etc/kubernetes/ssl/kube-proxy.pem \
  --client-key=/etc/kubernetes/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
  
  
# 配置关联

kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig



# 配置默认关联
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig

# 拷贝到目录
mv kube-proxy.kubeconfig /etc/kubernetes/

```


## 创建 kube-proxy.service 文件


```
# 创建 kube-proxy 目录

mkdir -p /var/lib/kube-proxy


vi /etc/systemd/system/kube-proxy.service

[Unit]
Description=kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=10.6.0.187 \
  --hostname-override=10.6.0.187 \
  --cluster-cidr=10.254.0.0/16 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

```


## 启动 kube-proxy

```

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```


## 部署其他Node 节点

```
 (第二个节点部署 10.6.0.188)

  .........省略了.......
  
        参照以上配置

```



## 测试集群


```
# 创建一个 nginx deplyment

apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 2
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
            
---

apiVersion: v1 
kind: Service
metadata: 
  name: nginx-svc 
spec: 
  ports: 
    - port: 80
      targetPort: 80
      protocol: TCP 
  selector: 
    name: nginx

```

```
[root@k8s-master-1 ~]# kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
nginx-dm-2214564181-lxff5   1/1       Running   0          14m
nginx-dm-2214564181-qm1bp   1/1       Running   0          14m


[root@k8s-master-1 ~]# kubectl get deployment
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-dm   2         2         2            2           14m


[root@k8s-master-1 ~]# kubectl get svc
NAME         CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.254.0.1      <none>        443/TCP   4h
nginx-svc    10.254.129.54   <none>        80/TCP    15m

```


```
# 在 node 里 curl

[root@k8s-node-1 ~]# curl 10.254.129.54
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



# 配置 KubeDNS

> 官方 github yaml 相关 https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/dns


## 下载镜像

```
# 官方镜像
gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.4
gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.4
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.4


# 我的镜像
jicki/k8s-dns-sidecar-amd64:1.14.4
jicki/k8s-dns-kube-dns-amd64:1.14.4
jicki/k8s-dns-dnsmasq-nanny-amd64:1.14.4

```



## 下载 yaml 文件

```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-cm.yaml


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-sa.yaml


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-controller.yaml.base


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kubedns-svc.yaml.base


# 修改后缀

mv kubedns-controller.yaml.base kubedns-controller.yaml

mv kubedns-svc.yaml.base kubedns-svc.yaml

```


## 系统预定义的 RoleBinding

> 预定义的 RoleBinding system:kube-dns 将 kube-system 命名空间的 kube-dns ServiceAccount 与 system:kube-dns Role 绑定， 该 Role 具有访问 kube-apiserver DNS 相关 API 的权限；


```
[root@k8s-master-1 kubedns]# kubectl get clusterrolebindings system:kube-dns -o yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  creationTimestamp: 2017-07-04T04:15:13Z
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-dns
  resourceVersion: "106"
  selfLink: /apis/rbac.authorization.k8s.io/v1beta1/clusterrolebindings/system%3Akube-dns
  uid: 60c1e0e1-606f-11e7-b212-d4ae52d1f0c9
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-dns
subjects:
- kind: ServiceAccount
  name: kube-dns
  namespace: kube-system
  
```

## 修改 kubedns-svc.yaml

```
# kubedns-svc.yaml 中 clusterIP: __PILLAR__DNS__SERVER__ 修改为我们之前定义的 dns IP 10.254.0.2

cat kubedns-svc.yaml

apiVersion: v1
kind: Service
metadata:
  name: kube-dns
  namespace: kube-system
  labels:
    k8s-app: kube-dns
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/name: "KubeDNS"
spec:
  selector:
    k8s-app: kube-dns
  clusterIP: 10.254.0.2
  ports:
  - name: dns
    port: 53
    protocol: UDP
  - name: dns-tcp
    port: 53
    protocol: TCP
    
```


## 修改 kubedns-controller.yaml

```
1. # 修改 --domain=__PILLAR__DNS__DOMAIN__.   为 我们之前 预定的 domain 名称 --domain=cluster.local.

2. # 修改 --server=/__PILLAR__DNS__DOMAIN__/127.0.0.1#10053  中 domain 为我们之前预定的 --server=/cluster.local./127.0.0.1#10053

3. # 修改 --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__, 中的 domain 为我们之前预定的  --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local.,

4. # 修改 --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,  中的 domain 为我们之前预定的  --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local.,

```

## 导入 yaml 文件

```
[root@k8s-master-1 kubedns]# kubectl create -f .
configmap "kube-dns" created
deployment "kube-dns" created
serviceaccount "kube-dns" created
service "kube-dns" created

```

## 查看 kubedns 服务

```
[root@k8s-master-1 kubedns]# kubectl get all --namespace=kube-system
NAME                           READY     STATUS    RESTARTS   AGE
po/kube-dns-1511229508-llfgs   3/3       Running   0          1m

NAME           CLUSTER-IP   EXTERNAL-IP   PORT(S)         AGE
svc/kube-dns   10.254.0.2   <none>        53/UDP,53/TCP   1m

NAME              DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deploy/kube-dns   1         1         1            1           1m

NAME                     DESIRED   CURRENT   READY     AGE
rs/kube-dns-1511229508   1         1         1         1m

```


## 验证 dns 服务


```
# 导入之前的 nginx-dm yaml文件

[root@k8s-master-1 ~]# kubectl get svc nginx-svc
NAME        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
nginx-svc   10.254.79.137   <none>        80/TCP    29s


# 创建一个 pods 来测试一下 nameserver

apiVersion: v1
kind: Pod
metadata:
  name: alpine
spec:
  containers:
  - name: alpine
    image: alpine
    command:
    - sh
    - -c
    - while true; do sleep 1; done



# 查看 pods
[root@k8s-master-1 ~]# kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
alpine                      1/1       Running   0          1m
nginx-dm-2214564181-4zjbh   1/1       Running   0          5m
nginx-dm-2214564181-tpz8t   1/1       Running   0          5m



# 测试

[root@k8s-master-1 ~]# kubectl exec -it alpine ping nginx-svc
PING nginx-svc (10.254.207.143): 56 data bytes


[root@k8s-master-1 ~]# kubectl exec -it alpine nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.207.143 nginx-svc.default.svc.cluster.local

```


# 部署 Ingress 与 Dashboard



## 部署 dashboard


> 官方 dashboard 的github https://github.com/kubernetes/dashboard


> 这里注意，以下部署的应用为 api-service 关闭了 RBAC 的， 在开启了 RBAC 的情况下，无论是 dashboard 与 nginx ingress 都需要修改，默认是有问题的。

## 下载 dashboard 镜像

```
# 官方镜像
gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.1

# 国内镜像
jicki/kubernetes-dashboard-amd64:v1.6.1
```


## 下载 yaml 文件

```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dashboard/dashboard-controller.yaml


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dashboard/dashboard-service.yaml

```


## 导入 yaml

```
[root@k8s-master-1 dashboard]# kubectl apply -f .
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created


# 查看 svc 与 pod

[root@k8s-master-1 dashboard]# kubectl get svc -n kube-system kubernetes-dashboard
NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   10.254.167.28   <none>        80/TCP    31s

```



## 部署 Nginx Ingress

> kubernetes 暴露服务的方式目前只有三种：LoadBlancer Service、NodePort Service、Ingress； 什么是 Ingress ? Ingress 就是利用 Nginx Haproxy 等负载均衡工具来暴露 kubernetes 服务。
>
>
> 官方 Nginx Ingress github https://github.com/kubernetes/ingress/tree/master/examples/deployment/nginx


```
# 下载镜像

# 官方镜像
gcr.io/google_containers/defaultbackend:1.0
gcr.io/google_containers/nginx-ingress-controller:0.9.0-beta.10

# 国内镜像
jicki/defaultbackend:1.0
jicki/nginx-ingress-controller:0.9.0-beta.10

```


```
# 部署 Nginx  backend , Nginx backend 用于统一转发 没有的域名 到指定页面。

curl -O https://raw.githubusercontent.com/kubernetes/ingress/master/examples/deployment/nginx/default-backend.yaml


# 直接导入既可, 这里不需要修改

[root@k8s-master-1 ingress]# kubectl apply -f default-backend.yml 
deployment "default-http-backend" created
service "default-http-backend" created


# 查看服务
[root@k8s-master-1 ingress]# kubectl get deployment -n kube-system default-http-backend
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
default-http-backend   1         1         1            1           36s

```


```
# 部署 Ingress Controller 组件

# 下载 yaml 文件

curl -O https://raw.githubusercontent.com/kubernetes/ingress/master/examples/daemonset/nginx/nginx-ingress-daemonset.yaml

```

```
# 导入 yaml 文件

[root@k8s-master-1 ingress]# kubectl apply -f nginx-ingress-daemonset.yaml 
daemonset "nginx-ingress-lb" created


# 查看服务
[root@k8s-master-1 ingress]# kubectl get daemonset -n kube-system
NAME               DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE-SELECTOR   AGE
nginx-ingress-lb   2         2         2         2            2           <none>          11s


```

```
# 创建一个 ingress

# 查看我们原有的 svc

[root@k8s-master-1 Ingress]# kubectl get svc nginx-svc
NAME        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx-svc   10.254.207.143   <none>        80/TCP    1d


# 创建 yaml 文件

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



# 导入 yaml

[root@k8s-master-1 ingress]# kubectl apply -f nginx-ingress.yaml 
ingress "nginx-ingress" created



# 查看 ingress

[root@k8s-master-1 Ingress]# kubectl get ingress
NAME            HOSTS            ADDRESS            PORTS     AGE
nginx-ingress   nginx.jicki.cn   10.6.0.187,10...   80        24s



# 测试访问

[root@k8s-master-1 ingress]# curl -I nginx.jicki.cn
HTTP/1.1 200 OK
Server: nginx/1.13.2
Date: Thu, 06 Jul 2017 04:21:43 GMT
Content-Type: text/html
Content-Length: 612
Connection: keep-alive
Last-Modified: Wed, 28 Jun 2017 18:27:36 GMT
ETag: "5953f518-264"
Accept-Ranges: bytes

```



```
# 配置一个 Dashboard Ingress

# 查看 dashboard 的 svc

[root@k8s-master-1 ingress]# kubectl get svc -n kube-system kubernetes-dashboard
NAME                   CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
kubernetes-dashboard   10.254.81.94   <none>        80/TCP    2h



# 编辑一个 yaml 文件


vi  dashboard-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: dashboard-ingress
  namespace: kube-system
spec:
  rules:
  - host: dashboard.jicki.cn
    http:
      paths:
      - backend:
          serviceName: kubernetes-dashboard
          servicePort: 80





# 查看 ingress

[root@k8s-master-1 dashboard]# kubectl get ingress -n kube-system
NAME                HOSTS                ADDRESS            PORTS     AGE
dashboard-ingress   dashboard.jicki.cn   10.6.0.187,10...   80        1m





# 测试访问

[root@k8s-master-1 dashboard]# curl -I dashboard.jicki.cn
HTTP/1.1 200 OK
Server: nginx/1.13.2
Date: Thu, 06 Jul 2017 06:32:00 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 848
Connection: keep-alive
Accept-Ranges: bytes
Cache-Control: no-store
Last-Modified: Tue, 16 May 2017 12:53:01 GMT

```


