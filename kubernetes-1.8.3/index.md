# kubernetes 1.8.3



# kubernetes 1.8.3

> 基于 二进制 文件部署
> 本地化 kube-apiserver, kube-controller-manager , kube-scheduler
> 我这边配置  既是 master 也是 nodes

## 环境说明

```
k8s-master-64: 172.16.1.64
k8s-master-65: 172.16.1.65
k8s-master-66: 172.16.1.66
```


## 初始化环境

```
hostnamectl --static set-hostname hostname

k8s-master-64: 172.16.1.64
k8s-master-65: 172.16.1.65
k8s-master-66: 172.16.1.66
```



```
#编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts

k8s-master-64: 172.16.1.64
k8s-master-65: 172.16.1.65
k8s-master-66: 172.16.1.66
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

```


```
# config.json 文件

vi  config.json


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

vi csr.json

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


[root@k8s-master-64 ssl]# ls -lt
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
cp *.pem /etc/kubernetes/ssl
cp ca.csr /etc/kubernetes/ssl

# 这里要将文件拷贝到所有的k8s 机器上

scp *.pem 172.16.1.65:/etc/kubernetes/ssl/
scp *.csr 172.16.1.65:/etc/kubernetes/ssl/

scp *.pem 172.16.1.66:/etc/kubernetes/ssl/
scp *.csr 172.16.1.66:/etc/kubernetes/ssl/

```



# 安装 docker

> 所有服务器预先安装 docker-ce ，官方1.8 中提示， 目前 k8s 支持最高 Docker versions 1.11.2, 1.12.6, 1.13.1, and 17.03.2

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

# 查看yum 版本

yum list docker-ce.x86_64  --showduplicates |sort -r



# 安装指定版本 docker-ce 17.03 被 docker-ce-selinux 依赖, 不能直接yum 安装 docker-ce-selinux

wget https://download.docker.com/linux/centos/7/x86_64/stable/Packages/docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm


rpm -ivh docker-ce-selinux-17.03.2.ce-1.el7.centos.noarch.rpm


yum -y install docker-ce-17.03.2.ce


# 查看安装

docker version
Client:
 Version:      17.03.2-ce
 API version:  1.27
 Go version:   go1.7.5
 Git commit:   f5ec1e2
 Built:        Tue Jun 27 02:21:36 2017
 OS/Arch:      linux/amd64

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

mkdir -p /usr/lib/systemd/system/docker.service.d/


vi /usr/lib/systemd/system/docker.service.d/docker-options.conf

# 添加如下 :   (注意 environment 必须在同一行，如果出现换行会无法加载)


[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.254.0.0/16 --graph=/opt/docker --registry-mirror=http://b438f72b.m.daocloud.io --disable-legacy-registry"




vi /usr/lib/systemd/system/docker.service.d/docker-dns.conf


# 添加如下 : 

[Service]
Environment="DOCKER_DNS_OPTIONS=\
    --dns 10.254.0.2 --dns 114.114.114.114  \
    --dns-search default.svc.cluster.local --dns-search svc.cluster.local  \
    --dns-opt ndots:2 --dns-opt timeout:2 --dns-opt attempts:2  \

```





```
# 重新读取配置，启动 docker 
systemctl daemon-reload
systemctl start docker
systemctl enable docker

```





# etcd 集群

> etcd 是k8s集群最重要的组件， etcd 挂了，集群就挂了


## 安装 etcd

```
rpm -ivh etcd-3.2.9-1.x86_64.rpm
```

## 创建 etcd 证书


```
cd /opt/ssl/

vi etcd-csr.json

{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.16.1.64",
    "172.16.1.65",
    "172.16.1.66"
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

```


```
# 生成 etcd   密钥

/opt/local/cfssl/cfssl gencert -ca=/opt/ssl/ca.pem \
  -ca-key=/opt/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes etcd-csr.json | /opt/local/cfssl/cfssljson -bare etcd

```


```
# 查看生成

[root@k8s-master-64 ssl]# ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem



# 拷贝到etcd服务器

# etcd-1 
cp etcd*.pem /etc/kubernetes/ssl/

# etcd-2
scp etcd*.pem 172.16.1.65:/etc/kubernetes/ssl/

# etcd-3
scp etcd*.pem 172.16.1.66:/etc/kubernetes/ssl/



# 如果 etcd 非 root 用户，读取证书会提示没权限

chmod 644 /etc/kubernetes/ssl/etcd-key.pem

```



## 修改 etcd 配置



> 修改 etcd 配置文件 /etc/etcd/etcd.conf

```
# etcd-1

mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf-bak

vi /etc/etcd/etcd.conf

# [member]
ETCD_NAME=etcd1
ETCD_DATA_DIR="/var/lib/etcd/etcd1.etcd"
ETCD_WAL_DIR="/var/lib/etcd/wal"
ETCD_SNAPSHOT_COUNT="100"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://172.16.1.64:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.16.1.64:2379,http://127.0.0.1:2379"
ETCD_MAX_SNAPSHOTS="5"
ETCD_MAX_WALS="5"
#ETCD_CORS=""

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.1.64:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd1=https://172.16.1.64:2380,etcd2=https://172.16.1.65:2380,etcd3=https://172.16.1.66:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.1.64:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_STRICT_RECONFIG_CHECK="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"

# [proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"

# [security]
ETCD_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_PEER_AUTO_TLS="true"

# [logging]
#ETCD_DEBUG="false"
# examples for -log-package-levels etcdserver=WARNING,security=DEBUG
#ETCD_LOG_PACKAGE_LEVELS=""

``` 

```
# etcd-2

mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf-bak

vi /etc/etcd/etcd.conf

# [member]
ETCD_NAME=etcd2
ETCD_DATA_DIR="/var/lib/etcd/etcd2.etcd"
ETCD_WAL_DIR="/var/lib/etcd/wal"
ETCD_SNAPSHOT_COUNT="100"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://172.16.1.65:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.16.1.65:2379,http://127.0.0.1:2379"
ETCD_MAX_SNAPSHOTS="5"
ETCD_MAX_WALS="5"
#ETCD_CORS=""

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.1.65:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd1=https://172.16.1.64:2380,etcd2=https://172.16.1.65:2380,etcd3=https://172.16.1.66:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.1.65:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_STRICT_RECONFIG_CHECK="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"

# [proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"

# [security]
ETCD_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_PEER_AUTO_TLS="true"

# [logging]
#ETCD_DEBUG="false"
# examples for -log-package-levels etcdserver=WARNING,security=DEBUG
#ETCD_LOG_PACKAGE_LEVELS=""

``` 

```
# etcd-3

mv /etc/etcd/etcd.conf /etc/etcd/etcd.conf-bak

vi /etc/etcd/etcd.conf

# [member]
ETCD_NAME=etcd3
ETCD_DATA_DIR="/var/lib/etcd/etcd3.etcd"
ETCD_WAL_DIR="/var/lib/etcd/wal"
ETCD_SNAPSHOT_COUNT="100"
ETCD_HEARTBEAT_INTERVAL="100"
ETCD_ELECTION_TIMEOUT="1000"
ETCD_LISTEN_PEER_URLS="https://172.16.1.66:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.16.1.66:2379,http://127.0.0.1:2379"
ETCD_MAX_SNAPSHOTS="5"
ETCD_MAX_WALS="5"
#ETCD_CORS=""

# [cluster]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.16.1.66:2380"
# if you use different ETCD_NAME (e.g. test), set ETCD_INITIAL_CLUSTER value for this name, i.e. "test=http://..."
ETCD_INITIAL_CLUSTER="etcd1=https://172.16.1.64:2380,etcd2=https://172.16.1.65:2380,etcd3=https://172.16.1.66:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="k8s-etcd-cluster"
ETCD_ADVERTISE_CLIENT_URLS="https://172.16.1.66:2379"
#ETCD_DISCOVERY=""
#ETCD_DISCOVERY_SRV=""
#ETCD_DISCOVERY_FALLBACK="proxy"
#ETCD_DISCOVERY_PROXY=""
#ETCD_STRICT_RECONFIG_CHECK="false"
#ETCD_AUTO_COMPACTION_RETENTION="0"

# [proxy]
#ETCD_PROXY="off"
#ETCD_PROXY_FAILURE_WAIT="5000"
#ETCD_PROXY_REFRESH_INTERVAL="30000"
#ETCD_PROXY_DIAL_TIMEOUT="1000"
#ETCD_PROXY_WRITE_TIMEOUT="5000"
#ETCD_PROXY_READ_TIMEOUT="0"

# [security]
ETCD_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
ETCD_CLIENT_CERT_AUTH="true"
ETCD_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_AUTO_TLS="true"
ETCD_PEER_CERT_FILE="/etc/kubernetes/ssl/etcd.pem"
ETCD_PEER_KEY_FILE="/etc/kubernetes/ssl/etcd-key.pem"
ETCD_PEER_CLIENT_CERT_AUTH="true"
ETCD_PEER_TRUSTED_CA_FILE="/etc/kubernetes/ssl/ca.pem"
ETCD_PEER_AUTO_TLS="true"

# [logging]
#ETCD_DEBUG="false"
# examples for -log-package-levels etcdserver=WARNING,security=DEBUG
#ETCD_LOG_PACKAGE_LEVELS=""

``` 



## 启动 etcd

> 分别启动 所有节点的 etcd 服务

```
systemctl enable etcd

systemctl start etcd

systemctl status etcd
```


```
# 如果报错 请使用
journalctl -f -t etcd  和 journalctl -u etcd 来定位问题

```




## 验证 etcd 集群状态

> 查看 etcd 集群状态：

```
etcdctl --endpoints=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        cluster-health

member 35eefb8e7cc93b53 is healthy: got healthy result from https://172.16.1.66:2379
member 4576ff5ed626a66b is healthy: got healthy result from https://172.16.1.64:2379
member bf3bd651ec832339 is healthy: got healthy result from https://172.16.1.65:2379
cluster is healthy
```


> 查看 etcd 集群成员：

```
etcdctl --endpoints=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379\
        --cert-file=/etc/kubernetes/ssl/etcd.pem \
        --ca-file=/etc/kubernetes/ssl/ca.pem \
        --key-file=/etc/kubernetes/ssl/etcd-key.pem \
        member list


35eefb8e7cc93b53: name=etcd3 peerURLs=https://172.16.1.66:2380 clientURLs=https://172.16.1.66:2379 isLeader=false
4576ff5ed626a66b: name=etcd1 peerURLs=https://172.16.1.64:2380 clientURLs=https://172.16.1.64:2379 isLeader=true
bf3bd651ec832339: name=etcd2 peerURLs=https://172.16.1.65:2380 clientURLs=https://172.16.1.65:2379 isLeader=false

```

# 配置 Kubernetes 集群

> kubectl 安装在所有需要进行操作的机器上


## Master and Node 

> Master 需要部署 kube-apiserver , kube-scheduler , kube-controller-manager 这三个组件。
kube-scheduler 作用是调度pods分配到那个node里，简单来说就是资源调度。
kube-controller-manager 作用是 对 deployment controller , replication controller, endpoints controller, namespace controller, and serviceaccounts controller等等的循环控制，与kube-apiserver交互。


### 安装组件

```
# 从github 上下载版本

cd /tmp

wget https://dl.k8s.io/v1.8.3/kubernetes-server-linux-amd64.tar.gz

tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-apiserver,kube-controller-manager,kube-scheduler,kubectl,kube-proxy,kubelet} /usr/local/bin/

```


### 创建 admin 证书

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
  -config=/opt/ssl/config.json \
  -profile=kubernetes admin-csr.json | /opt/local/cfssl/cfssljson -bare admin


# 查看生成

[root@k8s-master-64 ssl]# ls admin*
admin.csr  admin-csr.json  admin-key.pem  admin.pem

cp admin*.pem /etc/kubernetes/ssl/

scp admin*.pem 172.16.1.65:/etc/kubernetes/ssl/

scp admin*.pem 172.16.1.66:/etc/kubernetes/ssl/
```


### 配置 kubectl kubeconfig 文件


>  生成证书相关的配置文件存储与 /root/.kube 目录中

```
# 配置 kubernetes 集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443


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




### 创建 kubernetes 证书

```
cd /opt/ssl

vi kubernetes-csr.json

{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.16.1.64",
    "172.16.1.65",
    "172.16.1.66",
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


## 这里 hosts 字段中 三个 IP 分别为 127.0.0.1 本机， 172.16.1.64 和 172.16.1.65 为 Master 的IP，多个Master需要写多个 10.254.0.1 为 kubernetes SVC 的 IP， 一般是 部署网络的第一个IP , 如: 10.254.0.1 ， 在启动完成后，我们使用   kubectl get svc ， 就可以查看到

```


### 生成 kubernetes 证书和私钥


```
/opt/local/cfssl/cfssl gencert -ca=/etc/kubernetes/ssl/ca.pem \
  -ca-key=/etc/kubernetes/ssl/ca-key.pem \
  -config=/opt/ssl/config.json \
  -profile=kubernetes kubernetes-csr.json | /opt/local/cfssl/cfssljson -bare kubernetes

# 查看生成

[root@k8s-master-64 ssl]# ls -lt kubernetes*
-rw-r--r-- 1 root root 1261 11月 16 15:12 kubernetes.csr
-rw------- 1 root root 1679 11月 16 15:12 kubernetes-key.pem
-rw-r--r-- 1 root root 1635 11月 16 15:12 kubernetes.pem
-rw-r--r-- 1 root root  475 11月 16 15:12 kubernetes-csr.json


# 拷贝到目录
cp kubernetes*.pem /etc/kubernetes/ssl/

scp kubernetes*.pem 172.16.1.65:/etc/kubernetes/ssl/

scp kubernetes*.pem 172.16.1.66:/etc/kubernetes/ssl/

```

### 配置 kube-apiserver

> kubelet 首次启动时向 kube-apiserver 发送 TLS Bootstrapping 请求，kube-apiserver 验证 kubelet 请求中的 token 是否与它配置的 token  一致，如果一致则自动为 kubelet生成证书和秘钥。


```
# 生成 token

[root@k8s-master-64 ssl]# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
d926457ebfe9b1feaa5957a3d73b8b73


# 创建 token.csv 文件

cd /opt/ssl

vi token.csv

d59a702004f33c659640bf8dd2717b64,kubelet-bootstrap,10001,"system:kubelet-bootstrap"


# 拷贝

cp token.csv /etc/kubernetes/

scp token.csv 172.16.1.65:/etc/kubernetes/

scp token.csv 172.16.1.66:/etc/kubernetes/
```


```
# 生成高级审核配置文件

cd /etc/kubernetes


cat >> audit-policy.yaml <<EOF
# Log all requests at the Metadata level.
apiVersion: audit.k8s.io/v1beta1
kind: Policy
rules:
- level: Metadata
EOF




# 拷贝

scp audit-policy.yaml 172.16.1.65:/etc/kubernetes/

scp audit-policy.yaml 172.16.1.66:/etc/kubernetes/

```



### 创建 kube-apiserver.service 文件

```
# 自定义 系统 service 文件一般存于 /etc/systemd/system/ 下
# 配置为 各自的本地 IP

vi /etc/systemd/system/kube-apiserver.service

[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
User=root
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota,NodeRestriction \
  --advertise-address=172.16.1.64 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --audit-policy-file=/etc/kubernetes/audit-policy.yaml \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --secure-port=6443 \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --enable-swagger-ui=true \
  --etcd-cafile=/etc/kubernetes/ssl/ca.pem \
  --etcd-certfile=/etc/kubernetes/ssl/etcd.pem \
  --etcd-keyfile=/etc/kubernetes/ssl/etcd-key.pem \
  --etcd-servers=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379 \
  --event-ttl=1h \
  --kubelet-https=true \
  --insecure-bind-address=127.0.0.1 \
  --insecure-port=8080 \
  --service-account-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=30000-32000 \
  --tls-cert-file=/etc/kubernetes/ssl/kubernetes.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kubernetes-key.pem \
  --enable-bootstrap-token-auth \
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
# k8s 1.8 添加 --authorization-mode=Node
# k8s 1.8 添加 --admission-control=NodeRestriction
# k8s 1.8 添加 --audit-policy-file=/etc/kubernetes/audit-policy.yaml

# 这里面要注意的是 --service-node-port-range=30000-32000
# 这个地方是 映射外部端口时 的端口范围，随机映射也在这个范围内映射，指定映射端口必须也在这个范围内。
```

 
### 启动 kube-apiserver

```
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver

```


### 配置 kube-controller-manager


```
# 创建 kube-controller-manager.service 文件

vi /etc/systemd/system/kube-controller-manager.service


[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=0.0.0.0 \
  --master=http://127.0.0.1:8080 \
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



### 启动 kube-controller-manager

```
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```


### 配置 kube-scheduler


```
# 创建 kube-cheduler.service 文件

vi /etc/systemd/system/kube-scheduler.service


[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/GoogleCloudPlatform/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --address=0.0.0.0 \
  --master=http://127.0.0.1:8080 \
  --leader-elect=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```

### 启动 kube-scheduler

```
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler

```
 

### 验证 Master 节点

```
[root@k8s-master-64 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"} 



[root@k8s-master-2 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-2               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}  



[root@k8s-master-3 ~]# kubectl get componentstatuses
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-0               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   

```
 

### 配置 kubelet

> kubelet 启动时向 kube-apiserver 发送 TLS bootstrapping 请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap 用户赋予 system:node-bootstrapper 角色，然后 kubelet 才有权限创建认证请求(certificatesigningrequests)。


```
# 先创建认证请求
# user 为 master 中 token.csv 文件里配置的用户
# 只需创建一次就可以

kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap

```

### 创建 kubelet kubeconfig 文件


```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=bootstrap.kubeconfig

# 配置客户端认证

kubectl config set-credentials kubelet-bootstrap \
  --token=d926457ebfe9b1feaa5957a3d73b8b73 \
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


### 创建 kubelet.service 文件

```
# 创建 kubelet 目录

> 配置为 node 本机 IP

mkdir /var/lib/kubelet

vi /etc/systemd/system/kubelet.service


[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --cgroup-driver=cgroupfs \
  --hostname-override=k8s-master-64 \
  --pod-infra-container-image=jicki/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=10.254.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --fail-swap-on=false \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --max-pods=512 \
  --v=2

[Install]
WantedBy=multi-user.target

```


```
# 如上配置:
k8s-master-64    本机hostname
10.254.0.2       预分配的 dns 地址
cluster.local.   为 kubernetes 集群的 domain
jicki/pause-amd64:3.0  这个是 pod 的基础镜像，既 gcr 的 gcr.io/google_containers/pause-amd64:3.0 镜像， 下载下来修改为自己的仓库中的比较快。
```

### 启动 kubelet

```

systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet

```

```
# 如果报错 请使用
journalctl -f -t kubelet  和 journalctl -u kubelet 来定位问题

```


### 配置 TLS 认证

```
# 查看 csr 的名称

[root@k8s-master-64 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-320ITerfRCS4yul0iKNpsYhgnYt9ubVdd1NuuxIhbKI   1m        kubelet-bootstrap   Pending
node-csr-PAaagu_xofoU55gGYWhLzHp4fro3EgEgFq-vHpZeoX8   1m        kubelet-bootstrap   Pending
node-csr-WMtKYQlQRtq5LO8_auYQ8oKT81HL9iIbxzbwYjcqVL0   1m        kubelet-bootstrap   Pending


# 增加 认证

kubectl get csr | grep Pending | awk '{print $1}' | xargs kubectl certificate approve

```



### 验证 nodes

```
[root@k8s-master-64 ~]# kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
k8s-master-64   Ready     <none>    13s       v1.8.3
k8s-master-65   Ready     <none>    13s       v1.8.3
k8s-master-66   Ready     <none>    13s       v1.8.3


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


### 创建 kube-proxy 证书

```
# 证书方面由于我们node端没有装 cfssl
# 我们回到 master 端 机器 去配置证书，然后拷贝过来

[root@k8s-master-64 ~]# cd /opt/ssl


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
  -config=/opt/ssl/config.json \
  -profile=kubernetes  kube-proxy-csr.json | /opt/local/cfssl/cfssljson -bare kube-proxy
  
# 查看生成
ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem

# 拷贝到目录
cp kube-proxy*.pem /etc/kubernetes/ssl/

scp kube-proxy*.pem 172.16.1.65:/etc/kubernetes/ssl/

scp kube-proxy*.pem 172.16.1.66:/etc/kubernetes/ssl/
```


### 创建 kube-proxy kubeconfig 文件

```
# 配置集群

kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
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


### 创建 kube-proxy.service 文件


> 配置为 各自的 IP

```
# 创建 kube-proxy 目录

mkdir -p /var/lib/kube-proxy


vi /etc/systemd/system/kube-proxy.service

[Unit]
Description=Kubernetes Kube-Proxy Server
Documentation=https://github.com/GoogleCloudPlatform/kubernetes
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=172.16.1.64 \
  --hostname-override=k8s-master-64 \
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


### 启动 kube-proxy

```

systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy

```

```
# 如果报错 请使用
journalctl -f -t kube-proxy  和 journalctl -u kube-proxy 来定位问题

```



##  Node 端 

> 单 Node 部分 需要部署的组件有 docker  calico  kubectl  kubelet  kube-proxy 这几个组件。
Node 节点 基于 Nginx 负载 API 做 Master HA


```
# master 之间除 api server 以外其他组件通过 etcd 选举，api server 默认不作处理；在每个 node 上启动一个 nginx，每个 nginx 反向代理所有 api server，node 上 kubelet、kube-proxy 连接本地的 nginx 代理端口，当 nginx 发现无法连接后端时会自动踢掉出问题的 api server，从而实现 api server 的 HA
```


![ HAMaster][2]


### 安装组件

```
cd /tmp

wget https://dl.k8s.io/v1.8.3/kubernetes-server-linux-amd64.tar.gz

tar -xzvf kubernetes-server-linux-amd64.tar.gz

cd kubernetes

cp -r server/bin/{kube-proxy,kubelet} /usr/local/bin/

```


```
# ALL node

mkdir -p /etc/kubernetes/ssl/

scp ca.pem kube-proxy.pem kube-proxy-key.pem  node-*:/etc/kubernetes/ssl/

```


### 配置 kubelet or kube-proxy

```
# kubelet

# 首先 创建 kubelet kubeconfig 文件


kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=bootstrap.kubeconfig


# 配置客户端认证

kubectl config set-credentials kubelet-bootstrap \
  --token=d926457ebfe9b1feaa5957a3d73b8b73 \
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


```

# 创建 kube-proxy kubeconfig 文件


kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
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


### 创建Nginx 代理

> 在每个 node 都必须创建一个 Nginx 代理， 这里特别注意， 当 Master 也做为 Node 的时候 不需要配置 Nginx-proxy 

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
        server 172.16.1.64:6443;
        server 172.16.1.65:6443;
        server 172.16.1.66:6443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
EOF

# 更新权限
chmod +r /etc/nginx/nginx.conf

```


```
# 配置 Nginx 基于 docker 进程，然后配置 systemd 来启动

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
                              nginx:1.13.5-alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
EOF

```



```
# 启动 Nginx

systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy
systemctl status nginx-proxy


# 重启 Node 的 kubelet 与 kube-proxy

systemctl restart kubelet
systemctl status kubelet

systemctl restart kube-proxy
systemctl status kube-proxy

```


### Master 配置 TLS 认证

```
# 查看 csr 的名称

[root@k8s-master-64 ~]# kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-5dohu3QPMPWkxp9HRiNv7fZSXTE3MbxCIdf6G95ehXE   1h        kubelet-bootstrap   Approved,Issued
node-csr-ZFWZ5q5m323w_Iv5eTQclDhfgwaZ0Go-pztEoBfCMJk   22s       kubelet-bootstrap   Pending
node-csr-_yztJwMNeZ9HPlofd0Eiy_4fQLqKCIjU9_IfQ5koDCk   1h        kubelet-bootstrap   Approved,Issued
node-csr-pf-Bb5Iqx6ccvVA67gLVT-G4Zl3Zl5FPUZS4d7V6rk4   2h        kubelet-bootstrap   Approved,Issued
node-csr-u6JUtu1mVbC6sb4bxDuGkZT7Flzehren0OkOlfNp_MA   16s       kubelet-bootstrap   Pending
node-csr-xZZVdiWl1UEnDaRNkp3AHzT_E8FX3A71-5hDgQOW08U   14s       kubelet-bootstrap   Pending


# 增加 认证

[root@k8s-master-64 ~]# kubectl certificate approve NAME
```

```
[root@k8s-master-64 ~]# kubectl get nodes
NAME          STATUS    ROLES     AGE       VERSION
k8s-master-64   Ready     <none>    13s       v1.8.3
k8s-master-65   Ready     <none>    13s       v1.8.3
k8s-master-66   Ready     <none>    13s       v1.8.3
k8s-node-67     Ready     <none>    13s       v1.8.3
k8s-node-68     Ready     <none>    13s       v1.8.3
k8s-node-69     Ready     <none>    13s       v1.8.3
```



# 配置 Calico 网络

> calico 因为有某些坑，采用 systemd 托管 calico 服务


## 安装 Calico 

> 官网地址 http://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/hosted

```
# 下载 yaml 文件

wget http://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/calico.yaml

wget http://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/rbac.yaml


# 下载 镜像

# 国外镜像 有墙
quay.io/calico/node:v2.6.0
quay.io/calico/cni:v1.11.0
quay.io/calico/kube-controllers:v1.0.0


# 国内镜像
jicki/node:v2.6.0
jicki/cni:v1.11.0
jicki/kube-controllers:v1.0.0

```

## 配置 calico

```
vi calico.yaml

# 注意修改如下选项:


  etcd_endpoints: "https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379"
  
    etcd_ca: "/calico-secrets/etcd-ca"  
    etcd_cert: "/calico-secrets/etcd-cert"
    etcd_key: "/calico-secrets/etcd-key"  


# 这里面要写入 base64 的信息



data:
  etcd-key: (cat /etc/kubernetes/ssl/etcd-key.pem | base64 | tr -d '\n')
  etcd-cert: (cat /etc/kubernetes/ssl/etcd.pem | base64 | tr -d '\n')
  etcd-ca: (cat /etc/kubernetes/ssl/ca.pem | base64 | tr -d '\n')


    - name: CALICO_IPV4POOL_CIDR
      value: "10.233.0.0/16"
      


```



```
#  calico-node DaemonSet 中

        # Runs calico/node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        # 注释掉 移动到 系统 systemd 中运行
        #- name: calico-node
        #  image: jicki/node:v2.6.1
        #  env:
        #    # The location of the Calico etcd cluster.
        #    - name: ETCD_ENDPOINTS
        #      valueFrom:
        #        configMapKeyRef:
        #          name: calico-config
        #          key: etcd_endpoints
        #    # Choose the backend to use.
        #    - name: CALICO_NETWORKING_BACKEND
        #      valueFrom:
        #        configMapKeyRef:
        #          name: calico-config
        #          key: calico_backend
        #    # Cluster type to identify the deployment type
        #    - name: CLUSTER_TYPE
        #      value: "k8s,bgp"
        #    # Disable file logging so `kubectl logs` works.
        #    - name: CALICO_DISABLE_FILE_LOGGING
        #      value: "true"
        #    # Set Felix endpoint to host default action to ACCEPT.
        #    - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
        #      value: "ACCEPT"
        #    # Configure the IP Pool from which Pod IPs will be chosen.
        #    - name: CALICO_IPV4POOL_CIDR
        #      value: "10.233.0.0/16"
        #    - name: CALICO_IPV4POOL_IPIP
        #      value: "always"
        #    # Disable IPv6 on Kubernetes.
        #    - name: FELIX_IPV6SUPPORT
        #      value: "false"
        #    # Set Felix logging to "info"
        #    - name: FELIX_LOGSEVERITYSCREEN
        #      value: "info"
        #    # Set MTU for tunnel device used if ipip is enabled
        #    - name: FELIX_IPINIPMTU
        #      value: "1440"
        #    # Location of the CA certificate for etcd.
        #    - name: ETCD_CA_CERT_FILE
        #      valueFrom:
        #        configMapKeyRef:
        #          name: calico-config
        #          key: etcd_ca
        #    # Location of the client key for etcd.
        #    - name: ETCD_KEY_FILE
        #      valueFrom:
        #        configMapKeyRef:
        #          name: calico-config
        #          key: etcd_key
        #    # Location of the client certificate for etcd.
        #    - name: ETCD_CERT_FILE
        #      valueFrom:
        #        configMapKeyRef:
        #          name: calico-config
        #          key: etcd_cert
        #    # Auto-detect the BGP IP address.
        #    - name: IP
        #      value: ""
        #    - name: FELIX_HEALTHENABLED
        #      value: "true"
        #  securityContext:
        #    privileged: true
        #  resources:
        #    requests:
        #      cpu: 250m
        #  livenessProbe:
        #    httpGet:
        #      path: /liveness
        #      port: 9099
        #    periodSeconds: 10
        #    initialDelaySeconds: 10
        #    failureThreshold: 6
        #  readinessProbe:
        #    httpGet:
        #      path: /readiness
        #      port: 9099
        #    periodSeconds: 10
        #  volumeMounts:
        #    - mountPath: /lib/modules
        #      name: lib-modules
        #      readOnly: true
        #    - mountPath: /var/run/calico
        #      name: var-run-calico
        #      readOnly: false
        #    - mountPath: /calico-secrets
        #      name: etcd-certs
        # 注释掉 移动到 系统 systemd 中运行
        # This container installs the Calico CNI binaries
        # and CNI network config file on each node.


```




## 导入 yaml 文件

```
[root@k8s-master-64 ~]# kubectl apply -f calico.yaml 
configmap "calico-config" created
secret "calico-etcd-secrets" created
daemonset "calico-node" created
deployment "calico-policy-controller" created
serviceaccount "calico-policy-controller" created
serviceaccount "calico-node" created


[root@k8s-master-64 ~]# kubectl apply -f rbac.yaml
clusterrole "calico-policy-controller" created
clusterrolebinding "calico-policy-controller" created
clusterrole "calico-node" created
clusterrolebinding "calico-node" created

```

## 配置 calico-node 

> 配置一个 service ，用系统 systemd 启动




```
# 每个需要安装 calico 网络的都必须配置, 并修改其中的不同的参数。
# 1. NODENAME=k8s-master-64   
# 2. IP=172.16.1.64

vi /etc/systemd/system/calico-node.service


[Unit]
Description=calico node
After=docker.service
Requires=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run   --net=host --privileged --name=calico-node \
                                -e ETCD_ENDPOINTS=https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379 \
                                -e ETCD_CA_CERT_FILE=/etc/kubernetes/ssl/ca.pem \
                                -e ETCD_CERT_FILE=/etc/kubernetes/ssl/etcd.pem \
                                -e ETCD_KEY_FILE=/etc/kubernetes/ssl/etcd-key.pem \
                                -e NODENAME=k8s-master-64 \
                                -e IP=172.16.1.64 \
                                -e IP6= \
                                -e AS= \
                                -e CALICO_IPV4POOL_CIDR=10.233.0.0/16 \
                                -e CALICO_IPV4POOL_IPIP=always \
                                -e CALICO_LIBNETWORK_ENABLED=true \
                                -e CALICO_NETWORKING_BACKEND=bird \
                                -e CALICO_DISABLE_FILE_LOGGING=true \
                                -e FELIX_IPV6SUPPORT=false \
                                -e FELIX_DEFAULTENDPOINTTOHOSTACTION=ACCEPT \
                                -e FELIX_LOGSEVERITYSCREEN=info \
                                -v /etc/kubernetes/ssl/ca.pem:/etc/kubernetes/ssl/ca.pem \
                                -v /etc/kubernetes/ssl/etcd.pem:/etc/kubernetes/ssl/etcd.pem \
                                -v /etc/kubernetes/ssl/etcd-key.pem:/etc/kubernetes/ssl/etcd-key.pem \
                                -v /var/run/calico:/var/run/calico \
                                -v /lib/modules:/lib/modules \
                                -v /run/docker/plugins:/run/docker/plugins \
                                -v /var/run/docker.sock:/var/run/docker.sock \
                                -v /var/log/calico:/var/log/calico \
                                jicki/node:v2.6.0
ExecStop=/usr/bin/docker rm -f calico-node
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target

```


## 启动 calico-node

```
systemctl daemon-reload
systemctl start calico-node
systemctl enable calico-node
systemctl status calico-node

```






##   配置  kubelet.service

```
vi /etc/systemd/system/kubelet.service

# 增加 如下配置

  --network-plugin=cni \


# 重新加载配置
systemctl daemon-reload
systemctl restart kubelet.service
systemctl status kubelet.service

```


## 验证 Calico

```

[root@k8s-master-64 calico]# kubectl get pods -n kube-system 
NAME                                       READY     STATUS    RESTARTS   AGE
calico-kube-controllers-85bd96b9bb-226r2   1/1       Running   0          14m
calico-node-4h2rs                          1/1       Running   0          14m
calico-node-v768t                          1/1       Running   0          14m
calico-node-xr5kv                          1/1       Running   0          14m

```


## 安装 Calicoctl

```
cd /usr/local/bin/

wget -c  https://github.com/projectcalico/calicoctl/releases/download/v1.6.1/calicoctl

chmod +x calicoctl



## 创建 calicoctl 配置文件

# 配置文件， 在 安装了 calico 网络的 机器下

mkdir /etc/calico

vi /etc/calico/calicoctl.cfg


apiVersion: v1
kind: calicoApiConfig
metadata:
spec:
  datastoreType: "etcdv2"
  etcdEndpoints: "https://172.16.1.64:2379,https://172.16.1.65:2379,https://172.16.1.66:2379"
  etcdKeyFile: "/etc/kubernetes/ssl/etcd-key.pem"
  etcdCertFile: "/etc/kubernetes/ssl/etcd.pem"
  etcdCACertFile: "/etc/kubernetes/ssl/ca.pem"




# 查看 calico 状态

[root@k8s-master-64 ~]# calicoctl node status
calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 172.16.1.65  | node-to-node mesh | up    | 09:24:50 | Established |
| 172.16.1.66  | node-to-node mesh | up    | 09:24:51 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

```


## 测试集群


```
# 创建一个 nginx deplyment

apiVersion: extensions/v1beta1 
kind: Deployment 
metadata: 
  name: nginx-dm
spec: 
  replicas: 3
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
[root@k8s-master-64 ~]# kubectl get pods -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP               NODE
nginx-dm-55b58f68b6-6q2lc   1/1       Running   0          2m        10.233.235.65    172.16.1.65
nginx-dm-55b58f68b6-x9hk5   1/1       Running   0          2m        10.233.28.129    172.16.1.64
nginx-dm-55b58f68b6-xbprt   1/1       Running   0          2m        10.233.21.1      172.16.1.66

```


```
# 在 node 里 curl

[root@k8s-master-64 ~]# curl 10.254.129.54
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
gcr.io/google_containers/k8s-dns-sidecar-amd64:1.14.7
gcr.io/google_containers/k8s-dns-kube-dns-amd64:1.14.7
gcr.io/google_containers/k8s-dns-dnsmasq-nanny-amd64:1.14.7


# 我的镜像
jicki/k8s-dns-sidecar-amd64:1.14.7
jicki/k8s-dns-kube-dns-amd64:1.14.7
jicki/k8s-dns-dnsmasq-nanny-amd64:1.14.7

```



## 下载 yaml 文件


```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kube-dns.yaml.base

# 修改后缀

mv kube-dns.yaml.base kube-dns.yaml

```

## 修改 kube-dns.yaml

```
1. # clusterIP: __PILLAR__DNS__SERVER__ 修改为我们之前定义的 dns IP 10.254.0.2

2. # 修改 --domain=__PILLAR__DNS__DOMAIN__.   为 我们之前 预定的 domain 名称 --domain=cluster.local.

3. # 修改 --server=/__PILLAR__DNS__DOMAIN__/127.0.0.1#10053  中 domain 为我们之前预定的 --server=/cluster.local./127.0.0.1#10053

4. # 修改 --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__, 中的 domain 为我们之前预定的  --probe=kubedns,127.0.0.1:10053,kubernetes.default.svc.cluster.local.,

5. # 修改 --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.__PILLAR__DNS__DOMAIN__,  中的 domain 为我们之前预定的  --probe=dnsmasq,127.0.0.1:53,kubernetes.default.svc.cluster.local.,

```

## 导入 yaml 文件

```
# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *

# 导入

[root@k8s-master-64 kubedns]# kubectl create -f kube-dns.yaml
service "kube-dns" created
serviceaccount "kube-dns" created
configmap "kube-dns" created
deployment "kube-dns" created

```

## 查看 kubedns 服务

```
[root@k8s-master-64 kubedns]# kubectl get pods -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
calico-kube-controllers-85bd96b9bb-226r2   1/1       Running   0          36m
calico-node-4h2rs                          1/1       Running   0          36m
calico-node-v768t                          1/1       Running   0          36m
calico-node-xr5kv                          1/1       Running   0          36m
kube-dns-787dd94b74-6fpk8                  3/3       Running   0          24s

```


## 配置 dns 自动扩容


```
# 官方镜像

gcr.io/google_containers/cluster-proportional-autoscaler-amd64:1.1.2-r2

# 个人镜像

jicki/cluster-proportional-autoscaler-amd64:1.1.2-r2

```



```
# 下载 yaml

wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml



# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *


# 导入服务

[root@k8s-master-64 dns]# kubectl apply -f dns-horizontal-autoscaler.yaml 
serviceaccount "kube-dns-autoscaler" created
clusterrole "system:kube-dns-autoscaler" created
clusterrolebinding "system:kube-dns-autoscaler" created
deployment "kube-dns-autoscaler" created


# 查看服务

[root@k8s-master-64 dns]# kubectl get pods -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
calico-kube-controllers-85bd96b9bb-226r2   1/1       Running   0          47m
calico-node-4h2rs                          1/1       Running   0          47m
calico-node-v768t                          1/1       Running   0          47m
calico-node-xr5kv                          1/1       Running   0          47m
kube-dns-787dd94b74-6fpk8                  3/3       Running   0          10m
kube-dns-787dd94b74-fx6nc                  3/3       Running   0          24s
kube-dns-autoscaler-856c97456-x6v7j        1/1       Running   0          36s


```



## 验证 dns 服务

> 在验证 dns 之前，在 dns 未部署之前创建的 pod 与 deployment 等，都必须删除，重新部署，否则无法解析


```

# 导入之前的 nginx-dm yaml文件


# 查看 pods

[root@k8s-master-64 ~]# kubectl get pods -o wide
NAME                       READY     STATUS    RESTARTS   AGE       IP               NODE
nginx-dm-d9d5db9dd-62k92   1/1       Running   0          16s       10.233.108.193   k8s-master-66
nginx-dm-d9d5db9dd-pgsrx   1/1       Running   0          16s       10.233.153.130   k8s-master-65
nginx-dm-d9d5db9dd-s4vtr   1/1       Running   0          16s       10.233.252.3     k8s-master-64


# 查看 svc

[root@k8s-master-64 ~]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes   ClusterIP   10.254.0.1       <none>        443/TCP   2h        <none>
nginx-svc    ClusterIP   10.254.102.240   <none>        80/TCP    33s       name=nginx





# 创建一个 pods 来测试一下 dns 

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
[root@k8s-master-64 ~]# kubectl get pods
NAME                       READY     STATUS    RESTARTS   AGE
alpine                     1/1       Running   0          7s
nginx-dm-d9d5db9dd-62k92   1/1       Running   0          1m
nginx-dm-d9d5db9dd-pgsrx   1/1       Running   0          1m
nginx-dm-d9d5db9dd-s4vtr   1/1       Running   0          1m


# 测试

[root@k8s-master-64 ~]# kubectl exec -it alpine nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.254.102.240 nginx-svc.default.svc.cluster.local

```


# 部署 Ingress 与 Dashboard



## 部署 dashboard && heapster


> 官方 dashboard 的github https://github.com/kubernetes/dashboard


## 下载 dashboard 镜像

```
# 官方镜像
gcr.io/google_containers/kubernetes-dashboard-amd64:v1.6.3

# 国内镜像
jicki/kubernetes-dashboard-amd64:v1.6.3
```


## 下载 yaml 文件

```
curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dashboard/dashboard-controller.yaml


curl -O https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dashboard/dashboard-service.yaml



# 因为开启了 RBAC 所以这里需要创建一个 RBAC 认证

vi dashboard-rbac.yaml


apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard
  namespace: kube-system

---

kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: dashboard
subjects:
  - kind: ServiceAccount
    name: dashboard
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io

```


## 导入 yaml

```
# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *




# dashboard-controller.yaml 增加 rbac 授权


# 在第二个 spec 下面 增加

    spec:
      serviceAccountName: dashboard



# 导入文件

[root@k8s-master-64 dashboard]# kubectl apply -f .
deployment "kubernetes-dashboard" created
serviceaccount "dashboard" created
clusterrolebinding "dashboard" created
service "kubernetes-dashboard" created




# 查看 svc 与 pod

[root@k8s-master-64 ~]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.254.0.2       <none>        53/UDP,53/TCP   9m
kubernetes-dashboard   ClusterIP   10.254.119.100   <none>        80/TCP          16s

```



## 部署 Nginx Ingress

> Kubernetes 暴露服务的方式目前只有三种：LoadBlancer Service、NodePort Service、Ingress； 什么是 Ingress ? Ingress 就是利用 Nginx Haproxy 等负载均衡工具来暴露 Kubernetes 服务。
>
>
> 官方 Nginx Ingress github https://github.com/kubernetes/ingress/tree/master/examples/deployment/nginx



## 配置 调度 node

```
# ingress 有多种方式 1.  deployment 自由调度 replicas
                     2.  daemonset 全局调度 分配到所有node里


#  deployment 自由调度过程中，由于我们需要 约束 controller 调度到指定的 node 中，所以需要对 node 进行 label 标签


# 默认如下:
[root@k8s-master-64 ingress]# kubectl get nodes
NAME            STATUS    ROLES     AGE       VERSION
k8s-master-64   Ready     <none>    21h       v1.8.3
k8s-master-65   Ready     <none>    21h       v1.8.3
k8s-master-66   Ready     <none>    21h       v1.8.3


# 对 65 与 66 打上 label

[root@k8s-master-64 ingress]# kubectl label nodes k8s-master-65 ingress=proxy
node "172.16.1.65" labeled
[root@k8s-master-64 ingress]# kubectl label nodes k8s-master-66 ingress=proxy
node "172.16.1.66" labeled


# 打完标签以后

[root@k8s-master-64 ingress]# kubectl get nodes --show-labels
NAME            STATUS    ROLES     AGE       VERSION   LABELS
k8s-master-64   Ready     <none>    21h       v1.8.3    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=k8s-master-64
k8s-master-65   Ready     <none>    21h       v1.8.3    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=k8s-master-65
k8s-master-66   Ready     <none>    21h       v1.8.3    beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=k8s-master-66

```




```
# 下载镜像

# 官方镜像
gcr.io/google_containers/defaultbackend:1.4
quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.9.0-beta.17

# 国内镜像
jicki/defaultbackend:1.4
jicki/nginx-ingress-controller:0.9.0-beta.17

```


```
# 部署 Nginx  backend , Nginx backend 用于统一转发 没有的域名 到指定页面。

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml


# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *


# 直接导入既可, 这里不需要修改

[root@k8s-master-64 ingress]# kubectl apply -f default-backend.yaml 
deployment "default-http-backend" created
service "default-http-backend" created



# 查看服务
[root@k8s-master-64 ingress]# kubectl get deployment -n kube-system default-http-backend
NAME                   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
default-http-backend   1         1         1            1           36s

```


```
# 部署 Ingress RBAC 认证

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml


```




```
# 部署 Ingress Controller 组件

# 下载 yaml 文件

curl -O https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml


# 替换所有的 images

sed -i 's/gcr\.io\/google_containers/jicki/g' *
sed -i 's/quay\.io\/kubernetes-ingress-controller/jicki/g' *

# 上面 对 两个 node 打了 label 所以配置 replicas: 2
# 修改 yaml 文件 增加 rbac 认证 , hostNetwork  还有 nodeSelector, 第二个 spec 下 增加。

spec:
  replicas: 2
  ....
    spec:
      hostNetwork: true
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        ingress: proxy
    ....


```


```
# 导入 yaml 文件

[root@k8s-master-64 ingress]# kubectl apply -f nginx-ingress-controller.yaml
deployment "nginx-ingress-controller" created


# 查看服务，可以看到这两个 pods 被分别调度到 65 与 29 中
[root@k8s-master-64 ingress]# kubectl get pods -n ingress-nginx -o wide
NAME                                        READY     STATUS    RESTARTS   AGE       IP             NODE
default-http-backend-7f47b7d69b-5df85       1/1       Running   0          1m        10.233.252.4   k8s-master-64
nginx-ingress-controller-745695d6cf-n74fn   1/1       Running   0          1m        172.16.1.66    k8s-master-66
nginx-ingress-controller-745695d6cf-xrml9   1/1       Running   0          1m        172.16.1.65    k8s-master-65


```

```
# 查看我们原有的 svc

[root@k8s-master-64 ingress]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)         AGE
kube-dns               ClusterIP   10.254.0.2       <none>        53/UDP,53/TCP   18m
kubernetes-dashboard   ClusterIP   10.254.119.100   <none>        80/TCP          9m


# 创建 yaml 文件

vi dashboard-ingress.yaml

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



# 导入 yaml

[root@k8s-master-64 dashboard]# kubectl apply -f dashboard-ingress.yaml 
ingress "dashboard-ingress" created



# 查看 ingress

[root@k8s-master-64 dashboard]# kubectl get ingress -n kube-system -o wide
NAME                HOSTS                ADDRESS                   PORTS     AGE
dashboard-ingress   dashboard.jicki.cn   172.16.1.65,172.16.1.66   80        35s


# 测试访问

[root@k8s-master-64 dashboard]# curl -I dashboard.jicki.cn
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

![ Dashboard][1]


# Other





## 特殊 env


```
# yaml 中的一些 特殊 env


    env:
    - name: MY_POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: MY_POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP



```




  [1]: https://jicki.cn/img/posts/kubernetes/dashboard.png
  [2]: https://jicki.cn/img/posts/kubernetes/hamaster.jpg





