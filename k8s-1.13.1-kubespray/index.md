# k8s 1.13.1 to kubespray



# Kubernetes 1.13.1

> kubespray Deploy a Production Ready Kubernetes Cluster
>
> kubespray 利用 Ansible 自动部署 更新 kubernetes 集群


## 环境说明

```
kubernetes-1: 192.168.0.247
kubernetes-2: 192.168.0.248
kubernetes-3: 192.168.0.249

```

## 初始化环境

```
hostnamectl --static set-hostname hostname

kubernetes-1: 192.168.0.247
kubernetes-2: 192.168.0.248
kubernetes-3: 192.168.0.249

```


## 配置ssh key 认证

```
# 确保本机也可以 ssh 连接，否则下面部署失败

ssh-keygen -t rsa -N ""

ssh-copy-id -i /root/.ssh/id_rsa.pub 192.168.0.247

ssh-copy-id -i /root/.ssh/id_rsa.pub 192.168.0.248

ssh-copy-id -i /root/.ssh/id_rsa.pub 192.168.0.249
```



## 更新内核

> 更新内核为 4.4.x , CentOS 默认为3.10.x 

```
# 导入 Key

rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org


# 安装 Yum 源

rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm


# 更新 kernel

yum --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel 


# 配置 内核优先

grub2-set-default 0


# 重启系统

```



## 增加内核配置

```
vi /etc/sysctl.conf

# docker
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1



```




# kubespray


## 安装依赖


```
# 安装 centos 额外的yum源
rpm -ivh https://dl.fedoraproject.org/pub/epel/7/x86_64/Packages/e/epel-release-7-11.noarch.rpm


# make 缓存
yum clean all && yum makecache


# 安装 git

yum -y install git



# ansible 必须 >= 2.7

# 安装 软件
yum install -y python-pip python34 python-netaddr python34-pip ansible

```


## 下载源码


```
# git clone 源码

cd /opt/

git clone https://github.com/kubernetes-sigs/kubespray

```



## 安装 kubespray 依赖

```
cd /opt/kubespray

# 安装依赖

pip install -r requirements.txt


Successfully installed MarkupSafe-1.1.0 hvac-0.7.1 jinja2-2.10 netaddr-0.7.19 pbr-5.1.1

```


## 配置 kubespray


### hosts 配置

```
# 复制一份 自己的配置

cd /opt/kubespray

cp -rfp inventory/sample inventory/jicki


# 修改配置 hosts.ini

cd /opt/kubespray/inventory/jicki

rm -rf hosts.ini


vi hosts.ini

[all]
kubernetes-1 ansible_host=kubernetes-1 ansible_port=32 ip=192.168.0.247 etcd_member_name=etcd1
kubernetes-2 ansible_host=kubernetes-2 ansible_port=32 ip=192.168.0.248 etcd_member_name=etcd2
kubernetes-3 ansible_host=kubernetes-3 ansible_port=32 ip=192.168.0.249 etcd_member_name=etcd3

[kube-master]
kubernetes-1
kubernetes-2

[etcd]
kubernetes-1
kubernetes-2
kubernetes-3

[kube-node]
kubernetes-2
kubernetes-3

[k8s-cluster:children]
kube-master
kube-node

[calico-rr]

```


### etcd 配置

```
cd /opt/kubespray/inventory/jicki/group_vars

rm -rf etcd.yml


vi etcd.yml 

etcd_compaction_retention: 1
#etcd_metrics: basic
# etcd 内存限制 默认 512M
etcd_memory_limit: "2G"
# etcd data 文件大小，默认2G
etcd_quota_backend_bytes: "8G"
# etcd ssl 认证
# etcd_peer_client_auth: true

```


### 全局配置 all.yaml


```
cd /opt/kubespray/inventory/jicki/group_vars/all

vi all.yml

# 修改如下配置:

loadbalancer_apiserver_localhost: true

# 加载内核模块，否则 ceph, gfs 等无法挂载客户端
kubelet_load_modules: true


vi docker.yml

# 修改如下配置: 

docker_daemon_graph: "/opt/docker"


docker_registry_mirrors:
   - https://registry.docker-cn.com
   - https://mirror.aliyuncs.com
```


### k8s 功能配置

```
cd /opt/kubespray/inventory/jicki/group_vars/k8s-cluster

vi k8s-cluster.yml

# 按照自己的需求修改

```


### 修改镜像

> 默认镜像 从 gcr.io/google-containers 下载, 修改为国内的地址


```
# 下载国内镜像

docker pull jicki/kube-proxy:v1.13.1
docker pull jicki/kube-controller-manager:v1.13.1
docker pull jicki/kube-scheduler:v1.13.1
docker pull jicki/kube-apiserver:v1.13.1
docker pull jicki/coredns:1.2.6
docker pull jicki/cluster-proportional-autoscaler-amd64:1.3.0
docker pull jicki/kubernetes-dashboard-amd64:v1.10.0
docker pull jicki/etcd:3.2.24
docker pull jicki/node:v3.1.3
docker pull jicki/ctl:v3.1.3
docker pull jicki/kube-controllers:v3.1.3
docker pull jicki/cni:v3.1.3
docker pull jicki/pause:3.1
docker pull jicki/pause-amd64:3.1

```


```
# 修改下载的url

inventory/jicki/group_vars/k8s-cluster/k8s-cluster.yml

kube_image_repo: "gcr.io/google-containers"
修改为
kube_image_repo: "jicki"


sed -i 's/gcr\.io\/google_containers/jicki/g' roles/download/defaults/main.yml
sed -i 's/quay\.io\/coreos/jicki/g' roles/download/defaults/main.yml
sed -i 's/quay\.io\/calico/jicki/g' roles/download/defaults/main.yml


```



## 安装k8s集群


> 注： 创建集群会需要从 storage.googleapis.com 下载 二进制文件 如 kubeadm, hyperkube 等

```
cd /opt/kubespray

ansible-playbook -i inventory/jicki/hosts.ini --become --become-user=root cluster.yml





PLAY RECAP ****************************************************************************************************************************************************************************************
kubernetes-1               : ok=355  changed=72   unreachable=0    failed=0   
kubernetes-2               : ok=343  changed=70   unreachable=0    failed=0   
kubernetes-3               : ok=285  changed=50   unreachable=0    failed=0   
localhost                  : ok=1    changed=0    unreachable=0    failed=0   

Friday 21 December 2018  13:48:23 +0800 (0:00:00.059)       0:09:56.520 ******* 
=============================================================================== 
kubernetes/preinstall : Update package management cache (YUM) ---------------------------------------------------------------------------------------------------------------------------- 108.02s
download : container_download | download images for kubeadm config images ----------------------------------------------------------------------------------------------------------------- 90.71s
bootstrap-os : Install pip for bootstrap -------------------------------------------------------------------------------------------------------------------------------------------------- 46.12s
gather facts from all instances ----------------------------------------------------------------------------------------------------------------------------------------------------------- 26.64s
kubernetes/master : kubeadm | Initialize first master ------------------------------------------------------------------------------------------------------------------------------------- 23.33s
kubernetes/master : kubeadm | Init other uninitialized masters ---------------------------------------------------------------------------------------------------------------------------- 21.14s
etcd : reload etcd ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ 10.83s
kubernetes/kubeadm : Restart all kube-proxy pods to ensure that they load the new configmap ------------------------------------------------------------------------------------------------ 9.07s
etcd : wait for etcd up -------------------------------------------------------------------------------------------------------------------------------------------------------------------- 7.29s
etcd : Gen_certs | Write etcd master certs ------------------------------------------------------------------------------------------------------------------------------------------------- 6.76s
kubernetes-apps/ansible : Kubernetes Apps | Start Resources -------------------------------------------------------------------------------------------------------------------------------- 5.10s
etcd : Refresh Time Fact ------------------------------------------------------------------------------------------------------------------------------------------------------------------- 4.52s
container-engine/docker : Ensure old versions of Docker are not installed. | RedHat -------------------------------------------------------------------------------------------------------- 3.91s
kubernetes/master : kubeadm | write out kubeadm certs -------------------------------------------------------------------------------------------------------------------------------------- 3.89s
kubernetes/preinstall : Install packages requirements -------------------------------------------------------------------------------------------------------------------------------------- 3.74s
kubernetes-apps/ansible : Kubernetes Apps | Lay Down CoreDNS Template ---------------------------------------------------------------------------------------------------------------------- 3.50s
network_plugin/calico : Calico | Copy cni plugins ------------------------------------------------------------------------------------------------------------------------------------------ 3.09s
etcd : Gen_certs | Gather etcd master certs ------------------------------------------------------------------------------------------------------------------------------------------------ 2.54s
kubernetes-apps/network_plugin/calico : Start Calico resources ----------------------------------------------------------------------------------------------------------------------------- 2.51s
container-engine/docker : ensure docker packages are installed ----------------------------------------------------------------------------------------------------------------------------- 2.08s
```





## 验证k8s集群



```
# 查看集群状态

[root@kubernetes-1 ~]# kubectl get nodes -o wide
NAME           STATUS   ROLES         AGE   VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION                CONTAINER-RUNTIME
kubernetes-1   Ready    master        20m   v1.13.1   192.168.0.247   <none>        CentOS Linux 7 (Core)   4.4.168-1.el7.elrepo.x86_64   docker://18.6.1
kubernetes-2   Ready    master,node   19m   v1.13.1   192.168.0.248   <none>        CentOS Linux 7 (Core)   4.4.168-1.el7.elrepo.x86_64   docker://18.6.1
kubernetes-3   Ready    node          19m   v1.13.1   192.168.0.249   <none>        CentOS Linux 7 (Core)   4.4.168-1.el7.elrepo.x86_64   docker://18.6.1



# 查看集群组件

[root@kubernetes-1 ~]# kubectl get all --all-namespaces
NAMESPACE     NAME                                           READY   STATUS    RESTARTS   AGE
kube-system   pod/calico-kube-controllers-7c499f67b4-tldlj   1/1     Running   0          18m
kube-system   pod/calico-node-6psl6                          1/1     Running   0          18m
kube-system   pod/calico-node-dh5rv                          1/1     Running   0          18m
kube-system   pod/calico-node-mhb2m                          1/1     Running   0          18m
kube-system   pod/coredns-6fd7dbf94c-vvf5h                   1/1     Running   0          18m
kube-system   pod/coredns-6fd7dbf94c-zns9w                   1/1     Running   0          17m
kube-system   pod/dns-autoscaler-5b547856bc-t6brh            1/1     Running   0          17m
kube-system   pod/kube-apiserver-kubernetes-1                1/1     Running   0          19m
kube-system   pod/kube-apiserver-kubernetes-2                1/1     Running   0          19m
kube-system   pod/kube-controller-manager-kubernetes-1       1/1     Running   0          19m
kube-system   pod/kube-controller-manager-kubernetes-2       1/1     Running   0          19m
kube-system   pod/kube-proxy-dgt8r                           1/1     Running   0          17m
kube-system   pod/kube-proxy-skmf9                           1/1     Running   0          18m
kube-system   pod/kube-proxy-wj584                           1/1     Running   0          17m
kube-system   pod/kube-scheduler-kubernetes-1                1/1     Running   0          19m
kube-system   pod/kube-scheduler-kubernetes-2                1/1     Running   0          19m
kube-system   pod/kubernetes-dashboard-d7978b5cc-2s8bw       1/1     Running   0          17m
kube-system   pod/nginx-proxy-kubernetes-3                   1/1     Running   0          19m

NAMESPACE     NAME                           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes             ClusterIP   10.233.0.1     <none>        443/TCP                  20m
kube-system   service/coredns                ClusterIP   10.233.0.3     <none>        53/UDP,53/TCP,9153/TCP   18m
kube-system   service/kubernetes-dashboard   ClusterIP   10.233.47.27   <none>        443/TCP                  17m

NAMESPACE     NAME                         DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR                 AGE
kube-system   daemonset.apps/calico-node   3         3         3       3            3           <none>                        18m
kube-system   daemonset.apps/kube-proxy    3         3         3       3            3           beta.kubernetes.io/os=linux   20m

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           18m
kube-system   deployment.apps/coredns                   2/2     2            2           18m
kube-system   deployment.apps/dns-autoscaler            1/1     1            1           17m
kube-system   deployment.apps/kubernetes-dashboard      1/1     1            1           17m

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-7c499f67b4   1         1         1       18m
kube-system   replicaset.apps/coredns-6fd7dbf94c                   2         2         2       18m
kube-system   replicaset.apps/dns-autoscaler-5b547856bc            1         1         1       17m
kube-system   replicaset.apps/kubernetes-dashboard-d7978b5cc       1         1         1       17m



# 查看 ipvs

[root@kubernetes-1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.233.0.1:443 rr
  -> 192.168.0.247:6443           Masq    1      0          0         
  -> 192.168.0.248:6443           Masq    1      0          0         
TCP  10.233.0.3:53 rr
  -> 10.233.90.131:53             Masq    1      0          0         
  -> 10.233.101.2:53              Masq    1      0          0         
TCP  10.233.0.3:9153 rr
  -> 10.233.90.131:9153           Masq    1      0          0         
  -> 10.233.101.2:9153            Masq    1      0          0         
TCP  10.233.47.27:443 rr
  -> 10.233.101.1:8443            Masq    1      0          0         
UDP  10.233.0.3:53 rr
  -> 10.233.90.131:53             Masq    1      0          0         
  -> 10.233.101.2:53              Masq    1      0          0         

```



```
# 创建一个 deplpyment

vi nginx-deployment.yaml 


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
[root@kubernetes-1 ~]# kubectl apply -f nginx-deployment.yaml 
deployment.apps/nginx-dm created
service/nginx-svc created



[root@kubernetes-1 ~]# kubectl get pods -o wide
NAME                        READY   STATUS    RESTARTS   AGE   IP              NODE           NOMINATED NODE   READINESS GATES
nginx-dm-799879696c-5g9m6   1/1     Running   0          20s   10.233.90.133   kubernetes-2   <none>           <none>
nginx-dm-799879696c-b7htm   1/1     Running   0          20s   10.233.101.4    kubernetes-3   <none>           <none>
nginx-dm-799879696c-hzjfm   1/1     Running   0          20s   10.233.101.3    kubernetes-3   <none>           <none>




[root@kubernetes-1 ~]# kubectl get svc -o wide    
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes   ClusterIP   10.233.0.1     <none>        443/TCP   23m   <none>
nginx-svc    ClusterIP   10.233.42.25   <none>        80/TCP    26s   name=nginx
```

```
[root@kubernetes-1 ~]# ipvsadm -L -n
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.233.0.1:443 rr
  -> 192.168.0.247:6443           Masq    1      0          0         
  -> 192.168.0.248:6443           Masq    1      0          0         
TCP  10.233.0.3:53 rr
  -> 10.233.90.131:53             Masq    1      0          0         
  -> 10.233.101.2:53              Masq    1      0          0         
TCP  10.233.0.3:9153 rr
  -> 10.233.90.131:9153           Masq    1      0          0         
  -> 10.233.101.2:9153            Masq    1      0          0         
TCP  10.233.42.25:80 rr
  -> 10.233.90.133:80             Masq    1      0          1         
  -> 10.233.101.3:80              Masq    1      0          0         
  -> 10.233.101.4:80              Masq    1      0          1         
TCP  10.233.47.27:443 rr
  -> 10.233.101.1:8443            Masq    1      0          0         
UDP  10.233.0.3:53 rr
  -> 10.233.90.131:53             Masq    1      0          0         
  -> 10.233.101.2:53              Masq    1      0          0 


```





```
[root@kubernetes-1 ~]# curl -I http://10.233.42.25
HTTP/1.1 200 OK
Server: nginx/1.15.7
Date: Fri, 21 Dec 2018 06:10:48 GMT
Content-Type: text/html
Content-Length: 612
Last-Modified: Tue, 27 Nov 2018 22:24:28 GMT
Connection: keep-alive
ETag: "5bfdc41c-264"
Accept-Ranges: bytes

```





# upgrades 版本


## update 1.13.2

> 优雅更新 版本


```
git fetch origin
git checkout origin/master

ansible-playbook -i inventory/jicki/hosts.ini --become --become-user=root upgrade-cluster.yml -e kube_version=v1.13.2


```


## FAQ 问题

```
# FAQ 1:

TASK [download : file_download | Download item] ***************************************************************************************************************************************************
Friday 21 December 2018  10:37:05 +0800 (0:00:00.250)       0:03:05.805 ******* 
FAILED - RETRYING: file_download | Download item (4 retries left).


# 多试几次~ 国内~是真的没办法, 国内网络一时可以一时不行，取决于它解析的IP

# 或者去其他地方下载了放过去

分别是:

https://storage.googleapis.com/kubernetes-release/release/v1.13.1/bin/linux/amd64/hyperkube

https://storage.googleapis.com/kubernetes-release/release/v1.13.1/bin/linux/amd64/kubeadm

https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz

# 下载到 所有服务器的  /tmp/releases 目录下

# 并且授权 执行权限 

chmod +x hyperkube kubeadm cni-plugins-amd64-v0.6.0.tgz
```


```
# 懒人~做法~人就是要懒，不想写脚本

echo "pull images"

docker pull gcr.io/google-containers/kube-proxy:v1.13.2
docker pull gcr.io/google-containers/kube-controller-manager:v1.13.2
docker pull gcr.io/google-containers/kube-scheduler:v1.13.2
docker pull gcr.io/google-containers/kube-apiserver:v1.13.2
docker pull gcr.io/google-containers/kubernetes-dashboard-amd64:v1.10.1


echo "tag images"

docker tag gcr.io/google-containers/kube-proxy:v1.13.2 jicki/kube-proxy:v1.13.2
docker tag gcr.io/google-containers/kube-controller-manager:v1.13.2 jicki/kube-controller-manager:v1.13.2
docker tag gcr.io/google-containers/kube-scheduler:v1.13.2 jicki/kube-scheduler:v1.13.2
docker tag gcr.io/google-containers/kube-apiserver:v1.13.2 jicki/kube-apiserver:v1.13.2
docker tag gcr.io/google-containers/kubernetes-dashboard-amd64:v1.10.1 jicki/kubernetes-dashboard-amd64:v1.10.1

echo "push images"

docker push jicki/kube-proxy:v1.13.2
docker push jicki/kube-controller-manager:v1.13.2
docker push jicki/kube-scheduler:v1.13.2
docker push jicki/kube-apiserver:v1.13.2
docker push jicki/kubernetes-dashboard-amd64:v1.10.1


echo "rmi images"

docker rmi jicki/kube-proxy:v1.13.2
docker rmi jicki/kube-controller-manager:v1.13.2
docker rmi jicki/kube-scheduler:v1.13.2
docker rmi jicki/kube-apiserver:v1.13.2
docker rmi jicki/kubernetes-dashboard-amd64:v1.10.1

docker rmi gcr.io/google-containers/kube-proxy:v1.13.2
docker rmi gcr.io/google-containers/kube-controller-manager:v1.13.2
docker rmi gcr.io/google-containers/kube-scheduler:v1.13.2
docker rmi gcr.io/google-containers/kube-apiserver:v1.13.2
docker rmi gcr.io/google-containers/kubernetes-dashboard-amd64:v1.10.1

```



## update 1.13.5


> 优雅更新 版本


```
git fetch origin
git checkout origin/master

ansible-playbook -i inventory/jicki/hosts.ini --become --become-user=root upgrade-cluster.yml -e kube_version=v1.13.5


```


## FAQ 问题

```
# FAQ 1:

TASK [download : file_download | Download item] ***************************************************************************************************************************************************
Friday 21 December 2018  10:37:05 +0800 (0:00:00.250)       0:03:05.805 *******
FAILED - RETRYING: file_download | Download item (4 retries left).


# 多试几次~ 国内~是真的没办法, 国内网络一时可以一时不行，取决于它解析的IP

# 或者去其他地方下载了放过去

分别是:

https://storage.googleapis.com/kubernetes-release/release/v1.13.5/bin/linux/amd64/hyperkube

https://storage.googleapis.com/kubernetes-release/release/v1.13.5/bin/linux/amd64/kubeadm

https://github.com/containernetworking/plugins/releases/download/v0.6.0/cni-plugins-amd64-v0.6.0.tgz

# 下载到 所有服务器的  /tmp/releases 目录下

# 并且授权 执行权限

chmod +x hyperkube kubeadm cni-plugins-amd64-v0.6.0.tgz
```


```
# 懒人~做法~人就是要懒，不想写脚本

echo "pull images"

docker pull gcr.io/google-containers/kube-proxy:v1.13.5
docker pull gcr.io/google-containers/kube-controller-manager:v1.13.5
docker pull gcr.io/google-containers/kube-scheduler:v1.13.5
docker pull gcr.io/google-containers/kube-apiserver:v1.13.5
docker pull quay.io/calico/kube-controllers:v3.4.0
docker pull quay.io/calico/node:v3.4.0
docker pull quay.io/calico/cni:v3.4.0
docker pull quay.io/calico/ctl:v3.4.0
docker pull quay.io/calico/routereflector:v0.6.1

echo "tag images"

docker tag gcr.io/google-containers/kube-proxy:v1.13.5 jicki/kube-proxy:v1.13.5
docker tag gcr.io/google-containers/kube-controller-manager:v1.13.5 jicki/kube-controller-manager:v1.13.5
docker tag gcr.io/google-containers/kube-scheduler:v1.13.5 jicki/kube-scheduler:v1.13.5
docker tag gcr.io/google-containers/kube-apiserver:v1.13.5 jicki/kube-apiserver:v1.13.5
docker tag quay.io/calico/kube-controllers:v3.4.0 jicki/kube-controllers:v3.4.0-amd64
docker tag quay.io/calico/node:v3.4.0 jicki/node:v3.4.0-amd64
docker tag quay.io/calico/cni:v3.4.0 jicki/cni:v3.4.0-amd64
docker tag quay.io/calico/ctl:v3.4.0 jicki/ctl:v3.4.0-amd64
docker tag quay.io/calico/routereflector:v0.6.1 jicki/routereflector:v0.6.1-amd64

echo "push images"

docker push jicki/kube-proxy:v1.13.5
docker push jicki/kube-controller-manager:v1.13.5
docker push jicki/kube-scheduler:v1.13.5
docker push jicki/kube-apiserver:v1.13.5
docker push jicki/node:v3.4.0-amd64
docker push jicki/cni:v3.4.0-amd64
docker push jicki/ctl:v3.4.0-amd64
docker push jicki/routereflector:v0.6.1-amd64
docker push jicki/kube-controllers:v3.4.0-amd64

echo "rmi images"

docker rmi jicki/kube-proxy:v1.13.5
docker rmi jicki/kube-controller-manager:v1.13.5
docker rmi jicki/kube-scheduler:v1.13.5
docker rmi jicki/kube-apiserver:v1.13.5
docker rmi gcr.io/google-containers/kube-proxy:v1.13.5
docker rmi gcr.io/google-containers/kube-controller-manager:v1.13.5
docker rmi gcr.io/google-containers/kube-scheduler:v1.13.5
docker rmi gcr.io/google-containers/kube-apiserver:v1.13.5
docker rmi quay.io/calico/kube-controllers:v3.4.0
docker rmi quay.io/calico/node:v3.4.0
docker rmi quay.io/calico/cni:v3.4.0
docker rmi quay.io/calico/ctl:v3.4.0
docker rmi quay.io/calico/routereflector:v0.6.1
docker rmi jicki/kube-controllers:v3.4.0-amd64
docker rmi jicki/node:v3.4.0-amd64
docker rmi jicki/cni:v3.4.0-amd64
docker rmi jicki/ctl:v3.4.0-amd64
docker rmi jicki/routereflector:v0.6.1-amd64

```

