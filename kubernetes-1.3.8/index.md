# docker k8s 1.3.8 + flannel




# docker k8s + flannel

## 说明

kubernetes 是谷歌开源的 docker 集群管理解决方案。

项目地址： 
http://kubernetes.io/


## 环境说明

```
node-1: 10.6.0.140
node-2: 10.6.0.187
node-3: 10.6.0.188
```
kubernetes 集群，包含 master 节点，与 node 节点。


## 初始化环境

```
hostnamectl --static set-hostname hostname

10.6.0.140 - k8s-master
10.6.0.187 - k8s-node-1
10.6.0.188 - k8s-node-2
```

编辑 /etc/hosts 文件，配置hostname 通信

vi /etc/hosts 添加:

```
10.6.0.140 k8s-master
10.6.0.187 k8s-node-1
10.6.0.188 k8s-node-2
```


## 安装 etcd 集群

etcd 是k8s集群的基础组件。

```
yum -y install etcd
```
 

修改配置文件，/etc/etcd/etcd.conf 需要修改如下参数：

```
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

其他etcd集群中： ETCD_NAME , 以及IP 需要变动



修改 etcd 启动文件 /usr/lib/systemd/system/etcd.service

```
sed -i 's/\\\"${ETCD_LISTEN_CLIENT_URLS}\\\"/\\\"${ETCD_LISTEN_CLIENT_URLS}\\\" --listen-client-urls=\\\"${ETCD_LISTEN_CLIENT_URLS}\\\" --advertise-client-urls=\\\"${ETCD_ADVERTISE_CLIENT_URLS}\\\" --initial-cluster-token=\\\"${ETCD_INITIAL_CLUSTER_TOKEN}\\\" --initial-cluster=\\\"${ETCD_INITIAL_CLUSTER}\\\" --initial-cluster-state=\\\"${ETCD_INITIAL_CLUSTER_STATE}\\\"/g' /usr/lib/systemd/system/etcd.service
``` 


分别启动 所有节点的 etcd 服务

```
systemctl enable etcd

systemctl start etcd

systemctl status etcd
```


查看 etcd 集群状态：

```
etcdctl cluster-health
```

出现 cluster is healthy 表示成功



查看 etcd 集群成员：

```
etcdctl member list
```


## flannel 网络


安装 flannel 

```
yum -y install flannel
```


清除网络中遗留的docker 网络 (docker0, flannel0 等)

```
ifconfig
```

如果存在 请删除之，以免发生不必要的未知错误

```
ip link delete docker0
....
```


设置 flannel 所用到的IP段

```
etcdctl --endpoint http://10.6.0.140:2379 set /flannel/network/config '{"Network":"10.10.0.0/16","SubnetLen":25,"Backend":{"Type":"vxlan","VNI":1}}'
```

接下来修改 flannel 配置文件

```
vim /etc/sysconfig/flanneld


FLANNEL_ETCD="http://10.6.0.140:2379,http://10.6.0.187:2379,http://10.6.0.188:2379"   # 修改为 集群地址
FLANNEL_ETCD_KEY="/flannel/network/config"        # 修改为 上面导入配置中的  /flannel/network
FLANNEL_OPTIONS="--iface=em1"                     # 修改为 本机物理网卡的名称
```


启动 flannel

```
systemctl enable flanneld
systemctl start flanneld
systemctl status flanneld
```
 

下面还需要修改 docker 的启动文件 /usr/lib/systemd/system/docker.service

在 ExecStart 参数 dockerd 后面增加

```
ExecStart=/usr/bin/dockerd $DOCKER_NETWORK_OPTIONS
```

重新读取配置，启动 docker 

```
systemctl daemon-reload
systemctl start docker
```
 
查看网络接管

```
ifconfig
```


可以看到 docker0 与 flannel.1 已经在我们设置的IP段内了，表示已经成功

 


# 安装 kubernetes


安装k8s 首先是 Master 端安装

## 下载 rpm 包

```
http://upyun.mritd.me/kubernetes/kubernetes-1.3.8-1.x86_64.rpm
```

```
rpm -ivh kubernetes-1.3.8-1.x86_64.rpm
```

安装完 kubernetes 以后


## 配置 apiserver


编辑配置文件

```
vim /etc/kubernetes/apiserver


###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--insecure-bind-address=10.6.0.140"

# The port on the local server to listen on.
KUBE_API_PORT="--port=8080"

# Port minions listen on
KUBELET_PORT="--kubelet-port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://10.6.0.140:2379,http://10.6.0.187:2379,http://10.6.0.188:2379"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

# Add your own!
KUBE_API_ARGS=""
```
 

 

## 启动 所有服务

```
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl enable kube-apiserver
systemctl enable kube-controller-manager
systemctl enable kube-scheduler
systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
```
 


node 端安装

```
wget http://upyun.mritd.me/kubernetes/kubernetes-1.3.8-1.x86_64.rpm

rpm -ivh kubernetes-1.3.8-1.x86_64.rpm
```


## node 配置 kubelet

编辑配置文件

```
vim /etc/kubernetes/kubelet


###
# kubernetes kubelet (minion) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=10.6.0.187"

# The port for the info server to serve on
KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=k8s-node-1"

# location of the api-server
KUBELET_API_SERVER="--api-servers=http://10.6.0.140:8080"

# Add your own!
KUBELET_ARGS="--pod-infra-container-image=docker.io/kubernetes/pause:latest"
```
 

注： KUBELET_HOSTNAME 这个配置中 配置的为 hostname 名称，主要用于区分 node 在集群中的显示
名称 必须能 ping 通，所以前面在 /etc/hosts 中要做配置


## node 配置 k8s config

下面修改 kubernetes 的 config 文件

```
vim /etc/kubernetes/config


###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=http://10.6.0.140:8080"
```
 

 

## node 启动 所有服务

```
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kubelet
systemctl enable kube-proxy
systemctl status kubelet
systemctl status kube-proxy
```
 


## 测试

```
[root@k8s-master ~]#kubectl --server="http://10.6.0.140:8080" get node
NAME         STATUS     AGE
k8s-master   NotReady   50m
k8s-node-1   Ready      1m
k8s-node-2   Ready      57s
```

 


# 双向 TLS 认证配置

kubernetes 提供了多种安全认证机制 Token 或用户名密码的单向 tls 认证, 基于 CA 证书 双向 tls 认证。


## 创建 ca 证书

master 中 创建 openssl ca 证书

```
mkdir /etc/kubernetes/cert

# 必须授权

chown kube:kube -R /etc/kubernetes/cert

cd /etc/kubernetes/cert

# 生成私钥

openssl genrsa -out k8sca-key.pem 2048

openssl req -x509 -new -nodes -key k8sca-key.pem -days 10000 -out k8sca.pem -subj "/CN=kube-ca"
```


## 配置 apiserver 证书


复制openssl 的配置文件到cert目录中

```
cp /etc/pki/tls/openssl.cnf .
```


编辑 配置文件，支持IP认证

```
vim openssl.cnf


# 在 distinguished_name 上面添加  req_extensions          = v3_req

[ req ]
.....
req_extensions          = v3_req
distinguished_name      = req_distinguished_name
.....


# 在 [ v3_req ]  下面添加  subjectAltName = @alt_names

[ v3_req ]

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names


## 添加 如下所有内容

[ alt_names ]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = ${K8S_SERVICE_IP}  # kubernetes server ip
IP.2 = ${MASTER_HOST}     # master ip(如果都在一台机器上写一个就行)
```
 

 
## 签署 apiserver 证书

然后开始签署 apiserver 相关的证书

```
openssl genrsa -out apiserver-key.pem 2048

openssl req -new -key apiserver-key.pem -out apiserver.csr -subj "/CN=kube-apiserver" -config openssl.cnf

openssl x509 -req -in apiserver.csr -CA k8sca.pem -CAkey k8sca-key.pem -CAcreateserial -out apiserver.pem -days 365 -extensions v3_req -extfile openssl.cnf
``` 

 
## 生成 node 证书

下面 生成 每个 node 的证书, 在 master 中生成，然后复制到各 node 中

apiserver 证书签署完成后还需要签署每个节点 node 的证书

```
cp openssl.cnf worker-openssl.cnf
```

编辑配置文件:

```
vim worker-openssl.cnf

## 主要修改如下[ alt_names ]内容 , 有多少个node 就写多少个IP配置：

[ alt_names ]
IP.1 = 10.6.0.187
IP.2 = 10.6.0.188
```


生成 k8s-node-1 私钥

```
openssl genrsa -out k8s-node-1-worker-key.pem 2048

openssl req -new -key k8s-node-1-worker-key.pem -out k8s-node-1-worker.csr -subj "/CN=k8s-node-1" -config worker-openssl.cnf

openssl x509 -req -in k8s-node-1-worker.csr -CA k8sca.pem -CAkey k8sca-key.pem -CAcreateserial -out k8s-node-1-worker.pem -days 365 -extensions v3_req -extfile worker-openssl.cnf
```



生成 k8s-node-2 私钥

```
openssl genrsa -out k8s-node-2-worker-key.pem 2048

openssl req -new -key k8s-node-2-worker-key.pem -out k8s-node-2-worker.csr -subj "/CN=k8s-node-2" -config worker-openssl.cnf

openssl x509 -req -in k8s-node-2-worker.csr -CA k8sca.pem -CAkey k8sca-key.pem -CAcreateserial -out k8s-node-2-worker.pem -days 365 -extensions v3_req -extfile worker-openssl.cnf
```



## 生成集群管理证书


所有 node 生成以后， 还需要生成集群管理证书

```
openssl genrsa -out admin-key.pem 2048

openssl req -new -key admin-key.pem -out admin.csr -subj "/CN=kube-admin"

openssl x509 -req -in admin.csr -CA k8sca.pem -CAkey k8sca-key.pem -CAcreateserial -out admin.pem -days 365
```
 

## apiserver 配置


编辑 master apiserver 配置文件

```
vim /etc/kubernetes/apiserver


###
# kubernetes system config
#
# The following values are used to configure the kube-apiserver
#

# The address on the local server to listen to.
KUBE_API_ADDRESS="--insecure-bind-address=10.6.0.140 --insecure-bind-address=127.0.0.1"

# The port on the local server to listen on.
KUBE_API_PORT="--secure-port=6443 --insecure-port=8080"

# Port minions listen on
KUBELET_PORT="--kubelet-port=10250"

# Comma separated list of nodes in the etcd cluster
KUBE_ETCD_SERVERS="--etcd-servers=http://10.6.0.140:2379,http://10.6.0.187:2379,http://10.6.0.188:2379"

# Address range to use for services
KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.254.0.0/16"

# default admission control policies
KUBE_ADMISSION_CONTROL="--admission-control=NamespaceLifecycle,NamespaceExists,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota"

# Add your own!
KUBE_API_ARGS="--tls-cert-file=/etc/kubernetes/cert/apiserver.pem --tls-private-key-file=/etc/kubernetes/cert/apiserver-key.pem --client-ca-file=/etc/kubernetes/cert/k8sca.pem --service-account-key-file=/etc/kubernetes/cert/apiserver-key.pem"
```
 

## controller manager 配置


接着编辑 controller manager 的配置

```
vim /etc/kubernetes/controller-manager

###
# The following values are used to configure the kubernetes controller-manager

# defaults from config and apiserver should be adequate

# Add your own!
KUBE_CONTROLLER_MANAGER_ARGS="--service-account-private-key-file=/etc/kubernetes/cert/apiserver-key.pem  --root-ca-file=/etc/kubernetes/cert/k8sca.pem --master=http://127.0.0.1:8080"
```
 

 


## 重启 所有服务

```
systemctl restart kube-apiserver
systemctl restart kube-controller-manager
systemctl restart kube-scheduler

systemctl status kube-apiserver
systemctl status kube-controller-manager
systemctl status kube-scheduler
```
 

## node 节点配置


先 copy 配置文件 到 node 节点


k8s-node-1 节点：

```
mkdir /etc/kubernetes/cert/

# 必须授权

chown kube:kube -R /etc/kubernetes/cert

# 拷贝如下三个文件到 /etc/kubernetes/cert/ 目录下

k8s-node-1-worker-key.pem 
k8s-node-1-worker.pem
k8sca.pem

```


k8s-node-2 节点：

```
mkdir /etc/kubernetes/cert/

# 必须授权
chown kube:kube -R /etc/kubernetes/cert

# 拷贝如下三个文件到 /etc/kubernetes/cert/ 目录下
k8s-node-2-worker-key.pem 
k8s-node-2-worker.pem
k8sca.pem
```

 
## node 节点 kubelet 配置


如下配置 所有 node 节点都需要配置

修改 kubelet 配置

```
vim /etc/kubernetes/kubelet


###
# kubernetes kubelet (minion) config

# The address for the info server to serve on (set to 0.0.0.0 or "" for all interfaces)
KUBELET_ADDRESS="--address=10.6.0.188"

# The port for the info server to serve on
KUBELET_PORT="--port=10250"

# You may leave this blank to use the actual hostname
KUBELET_HOSTNAME="--hostname-override=k8s-node-2"

# location of the api-server
KUBELET_API_SERVER="--api-servers=https://10.6.0.140:6443"

# Add your own!
KUBELET_ARGS="--tls-cert-file=/etc/kubernetes/cert/k8s-node-1-worker.pem --tls-private-key-file=/etc/kubernetes/cert/k8s-node-1-worker-key.pem --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml --pod-infra-container-image=docker.io/kubernetes/pause:latest"
```
 

 
## node 节点 config 配置


修改 config 配置


```
vim /etc/kubernetes/config


###
# kubernetes system config
#
# The following values are used to configure various aspects of all
# kubernetes services, including
#
#   kube-apiserver.service
#   kube-controller-manager.service
#   kube-scheduler.service
#   kubelet.service
#   kube-proxy.service
# logging to stderr means we get it in the systemd journal
KUBE_LOGTOSTDERR="--logtostderr=true"

# journal message level, 0 is debug
KUBE_LOG_LEVEL="--v=0"

# Should this cluster be allowed to run privileged docker containers
KUBE_ALLOW_PRIV="--allow-privileged=false"

# How the controller-manager, scheduler, and proxy find the apiserver
KUBE_MASTER="--master=https://10.6.0.140:6443"
```
 

## node 创建 kube-proxy

创建 kube-proxy 配置文件

```
vim /etc/kubernetes/worker-kubeconfig.yaml


apiVersion: v1
kind: Config
clusters:
- name: local
  cluster:
    certificate-authority: /etc/kubernetes/cert/k8sca.pem
users:
- name: kubelet
  user:
    client-certificate: /etc/kubernetes/cert/k8s-node-1-worker.pem
    client-key: /etc/kubernetes/cert/k8s-node-1-worker-key.pem
contexts:
- context:
    cluster: local
    user: kubelet
  name: kubelet-context
current-context: kubelet-context
```
 

 
## node 节点 配置 kube-proxy 证书


配置 kube-proxy 使其使用证书

```
vim /etc/kubernetes/proxy


###
# kubernetes proxy config

# default config should be adequate

# Add your own!
KUBE_PROXY_ARGS="--master=https://10.6.0.140:6443 --kubeconfig=/etc/kubernetes/worker-kubeconfig.yaml"
```
 

## node节点重启服务

```
systemctl restart kubelet
systemctl restart kube-proxy

systemctl status kubelet
systemctl status kube-proxy
```

至此，整个集群已经搭建完成，剩下的就是pod 的测试

