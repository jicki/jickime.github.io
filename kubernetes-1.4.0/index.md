# docker kubernetes 1.4 部署



# k8s 1.4


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

配置 /etc/hosts

添加

```
10.6.0.140 k8s-master
10.6.0.187 k8s-node-1
10.6.0.188 k8s-node-2
```


# 安装kubernetes


## 安装依赖

```
yum install -y socat
```


## 增加yum 文件

```
cat <<EOF> /etc/yum.repos.d/k8s.repo
[kubelet]
name=kubelet
baseurl=http://files.rm-rf.ca/rpms/kubelet/
enabled=1
gpgcheck=0
EOF
```

## yum 安装程序

```
yum makecache

yum install -y kubelet kubeadm kubectl kubernetes-cni
```


由于 google 被墙, 所以使用 kubeadm init 创建 集群 的时候会出现卡住

国内已经有人将镜像上传至 docker hub 里面了


## 下载镜像

```
docker pull chasontang/kube-proxy-amd64:v1.4.0
docker pull chasontang/kube-discovery-amd64:1.0
docker pull chasontang/kubedns-amd64:1.7
docker pull chasontang/kube-scheduler-amd64:v1.4.0
docker pull chasontang/kube-controller-manager-amd64:v1.4.0
docker pull chasontang/kube-apiserver-amd64:v1.4.0
docker pull chasontang/etcd-amd64:2.2.5
docker pull chasontang/kube-dnsmasq-amd64:1.3
docker pull chasontang/exechealthz-amd64:1.1
docker pull chasontang/pause-amd64:3.0


# 下载以后使用 docker tag 命令将其做别名改为 gcr.io/google_containers


docker tag chasontang/kube-proxy-amd64:v1.4.0  gcr.io/google_containers/kube-proxy-amd64:v1.4.0
docker tag chasontang/kube-discovery-amd64:1.0 gcr.io/google_containers/kube-discovery-amd64:1.0
docker tag chasontang/kubedns-amd64:1.7  gcr.io/google_containers/kubedns-amd64:1.7
docker tag chasontang/kube-scheduler-amd64:v1.4.0  gcr.io/google_containers/kube-scheduler-amd64:v1.4.0
docker tag chasontang/kube-controller-manager-amd64:v1.4.0  gcr.io/google_containers/kube-controller-manager-amd64:v1.4.0
docker tag chasontang/kube-apiserver-amd64:v1.4.0  gcr.io/google_containers/kube-apiserver-amd64:v1.4.0
docker tag chasontang/etcd-amd64:2.2.5  gcr.io/google_containers/etcd-amd64:2.2.5
docker tag chasontang/kube-dnsmasq-amd64:1.3  gcr.io/google_containers/kube-dnsmasq-amd64:1.3
docker tag chasontang/exechealthz-amd64:1.1  gcr.io/google_containers/exechealthz-amd64:1.1
docker tag chasontang/pause-amd64:3.0  gcr.io/google_containers/pause-amd64:3.0


# 清除原来下载的镜像


docker rmi chasontang/kube-proxy-amd64:v1.4.0
docker rmi chasontang/kube-discovery-amd64:1.0
docker rmi chasontang/kubedns-amd64:1.7
docker rmi chasontang/kube-scheduler-amd64:v1.4.0
docker rmi chasontang/kube-controller-manager-amd64:v1.4.0
docker rmi chasontang/kube-apiserver-amd64:v1.4.0
docker rmi chasontang/etcd-amd64:2.2.5
docker rmi chasontang/kube-dnsmasq-amd64:1.3
docker rmi chasontang/exechealthz-amd64:1.1
docker rmi chasontang/pause-amd64:3.0

```
 

##  启动 kubelet

```
systemctl enable kubelet
systemctl start kubelet
``` 

 

## init 初始化集群

kubenetes 1.4 利用 kubeadm 创建 集群

```
[root@k8s-master ~]#kubeadm init --api-advertise-addresses=10.6.0.140


<master/tokens> generated token: "eb4d40.67aac8417294a8cf"
<master/pki> created keys and certificates in "/etc/kubernetes/pki"
<util/kubeconfig> created "/etc/kubernetes/kubelet.conf"
<util/kubeconfig> created "/etc/kubernetes/admin.conf"
<master/apiclient> created API client configuration
<master/apiclient> created API client, waiting for the control plane to become ready
<master/apiclient> all control plane components are healthy after 10.304645 seconds
<master/apiclient> waiting for at least one node to register and become ready
<master/apiclient> first node has registered, but is not ready yet
<master/apiclient> first node has registered, but is not ready yet
<master/apiclient> first node has registered, but is not ready yet
<master/apiclient> first node has registered, but is not ready yet
<master/apiclient> first node has registered, but is not ready yet
<master/apiclient> first node is ready after 3.004762 seconds
<master/discovery> created essential addon: kube-discovery, waiting for it to become ready
<master/discovery> kube-discovery is ready after 4.002661 seconds
<master/addons> created essential addon: kube-proxy
<master/addons> created essential addon: kube-dns

kubernetes master initialised successfully!

You can now join any number of machines by running the following on each node:

kubeadm join --token 8609e3.c2822cf312e597e1 10.6.0.140

```
 
查看 kubelet 状态

```
systemctl status kubelet
``` 

## 配置子节点

子节点 启动 kubelet 首先必须启动 docker

```
systemctl enable kubelet
systemctl start kubelet
``` 

加入集群

```
kubeadm join --token 8609e3.c2822cf312e597e1 10.6.0.140
``` 

查看 kubelet 状态

```
systemctl status kubelet
``` 

查看集群状态

```
[root@k8s-master ~]#kubectl get node
NAME         STATUS    AGE
k8s-master   Ready     1d
k8s-node-1   Ready     1d
k8s-node-2   Ready     1d
``` 

此时可看到 三个节点 都已经 Ready , 但是其实 Pod 只会运行在 node 节点

如果需要所有节点，包括master 也运行 Pod 需要运行

```
 kubectl taint nodes --all dedicated-
``` 

 


## 安装 POD 网络

这里使用官方推荐的 weave 网络

```
kubectl apply -f https://git.io/weave-kube
``` 


查看所有pod 状态

```
[root@k8s-master ~]#kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE
kube-system   etcd-k8s-master                      1/1       Running   1          49m
kube-system   kube-apiserver-k8s-master            1/1       Running   1          48m
kube-system   kube-controller-manager-k8s-master   1/1       Running   1          48m
kube-system   kube-discovery-1971138125-0oq58      1/1       Running   1          49m
kube-system   kube-dns-2247936740-ojzhw            3/3       Running   3          49m
kube-system   kube-proxy-amd64-1hhdf               1/1       Running   1          49m
kube-system   kube-proxy-amd64-4c2qt               1/1       Running   0          47m
kube-system   kube-proxy-amd64-tc3kw               1/1       Running   1          47m
kube-system   kube-scheduler-k8s-master            1/1       Running   1          48m
kube-system   weave-net-9mrlt                      2/2       Running   2          46m
kube-system   weave-net-oyguh                      2/2       Running   4          46m
kube-system   weave-net-zc67d                      2/2       Running   0          46m
```
 

 

## GlusterFS 作为 volume


官方详细说明：

https://github.com/kubernetes/kubernetes/tree/master/examples/volumes/glusterfs


1. 配置 GlusterFS 集群，以及设置好 GlusterFS 的 volume , node 客户端安装 glusterfs-client


2. k8s-master 创建一个 endpoints.

我这边 GlusterFS 有3个节点

```
vi glusterfs-endpoints.json


# 每一个 GlusterFS 节点，必须写一列. 端口随意填写(1-65535)


{
  "kind": "Endpoints",
  "apiVersion": "v1",
  "metadata": {
    "name": "glusterfs-cluster"
  },
  "subsets": [
    {
      "addresses": [
        {
          "ip": "10.6.0.140"
        }
      ],
      "ports": [
        {
          "port": 1
        }
      ]
    },
    {
      "addresses": [
        {
          "ip": "10.6.0.187"
        }
      ],
      "ports": [
        {
          "port": 1
        }
      ]
    },
    {
      "addresses": [
        {
          "ip": "10.6.0.188"
        }
      ],
      "ports": [
        {
          "port": 1
        }
      ]
    }
  ]
}
```
 

创建 endpoints

```
[root@k8s-master ~]#kubectl create -f glusterfs-endpoints.json 
endpoints "glusterfs-cluster" created
``` 


查看 endpoints

```
[root@k8s-master ~]#kubectl get endpoints
NAME                ENDPOINTS                                AGE
glusterfs-cluster   10.6.0.140:1,10.6.0.187:1,10.6.0.188:1   37s
``` 


3. k8s-master 创建一个 service.

```
vi glusterfs-service.json

# 这里注意之前填写的 port


{
  "kind": "Service",
  "apiVersion": "v1",
  "metadata": {
    "name": "glusterfs-cluster"
  },
  "spec": {
    "ports": [
      {"port": 1}
    ]
  }
}
```
 

创建 service 

```
[root@k8s-master ~]#kubectl create -f glusterfs-service.json 
service "glusterfs-cluster" created
``` 

查看 service

```
[root@k8s-master ~]#kubectl get service
NAME                CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
glusterfs-cluster   100.71.255.174   <none>        1/TCP     14s
``` 

 

4. k8s-master 创建一个 Pod 来测试挂载

```
vi glusterfs-pod.json


{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
        "name": "glusterfs"
    },
    "spec": {
        "containers": [
            {
                "name": "glusterfs",
                "image": "gcr.io/google_containers/pause-amd64:3.0",
                "volumeMounts": [
                    {
                        "mountPath": "/mnt/glusterfs",
                        "name": "glusterfsvol"
                    }
                ]
            }
        ],
        "volumes": [
            {
                "name": "glusterfsvol",
                "glusterfs": {
                    "endpoints": "glusterfs-cluster",
                    "path": "models",
                    "readOnly": false
                }
            }
        ]
    }
}
```

glusterfs 下 path 配置 glusterfs volume 的名称

readOnly: true (只读) and readOnly: false
 


查看 挂载的 volume

```
[root@k8s-node-2 ~]# mount | grep models
10.6.0.140:models on /var/lib/kubelet/pods/947390da-8f6a-11e6-9ade-d4ae52d1f0c9/volumes/kubernetes.io~glusterfs/glusterfsvol type fuse.glusterfs (rw,relatime,user_id=0,group_id=0,default_permissions,allow_other,max_read=131072)
``` 

 

## yaml 文件

编写一个 Deployment 的 yaml 文件

```
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
          - containerPort: 80
```
 

使用 kubectl create  进行创建

```
kubectl create -f nginx.yaml --record
``` 


查看 pod 

```
[root@k8s-master ~]#kubectl get pod
NAME                               READY     STATUS    RESTARTS   AGE
nginx-deployment-646889141-459i5   1/1       Running   0          9m
nginx-deployment-646889141-vxn29   1/1       Running   0          9m
``` 


查看 deployment

```
[root@k8s-master ~]#kubectl get deploy
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   2         2         2            2           10m
```

