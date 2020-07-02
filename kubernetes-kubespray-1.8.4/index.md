# kubernetes 1.8.4 to kubespray



# kubernetes 1.8.4 to kubespray

> 官方 github  https://github.com/kubernetes-incubator/kubespray



# 环境准备

## 初始化环境

|节点|IP|
|----|----|
|172.16.1.64|node1|
|172.16.1.65|node2|
|172.16.1.66|node3|



## 设置hostname


```
hostnamectl --static set-hostname hostname
```

|IP|hostname|
|----|----|
|172.16.1.64|node1|
|172.16.1.65|node2|
|172.16.1.66|node3|



## 配置 hosts

```
vi /etc/hosts
```

|IP|hostname|
|----|----|
|172.16.1.64|node1|
|172.16.1.65|node2|
|172.16.1.66|node3|


## 修改内核

```
vi /etc/sysctl.conf

# docker
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1

```

## 配置SSH Key 登陆

```
# 确保本机也可以 ssh 连接，否则下面部署失败

ssh-keygen -t rsa -N ""

ssh-copy-id -i /root/.ssh/id_rsa.pub 172.16.1.65

ssh-copy-id -i /root/.ssh/id_rsa.pub 172.16.1.66

```



# 配置 kubespray 


## 安装基础软件

```
# 安装 git
yum -y install git

# 安装 centos 额外的yum源
yum install -y epel-release

# make 缓存
yum clean all && yum makecache

# 安装 软件
yum install -y python-pip python34 python-netaddr python34-pip ansible

```


## 下载源码

```
# git clone
git clone https://github.com/kubernetes-incubator/kubespray

```

## 下载镜像

> 老问题，国外镜像被墙, 我已经上传至 docker 官方的 hub 中, 这里需要自行安装 docker .

```
jicki/hyperkube:v1.8.4_coreos.0
jicki/flannel:v0.9.1
jicki/flannel-cni:v0.3.0
jicki/etcd:v3.2.4
jicki/routereflector:v0.4.0
jicki/cluster-proportional-autoscaler-amd64:1.1.1
jicki/k8s-dns-sidecar-amd64:1.14.7 
jicki/k8s-dns-kube-dns-amd64:1.14.7 
jicki/k8s-dns-dnsmasq-nanny-amd64:1.14.7
jicki/kubernetes-dashboard-init-amd64:v1.0.1
jicki/kubernetes-dashboard-amd64:v1.7.1
jicki/pause-amd64:3.0

```


```
images=(cluster-proportional-autoscaler-amd64:1.1.1 k8s-dns-sidecar-amd64:1.14.7 k8s-dns-kube-dns-amd64:1.14.7 k8s-dns-dnsmasq-nanny-amd64:1.14.7 pause-amd64:3.0 hyperkube:v1.8.4_coreos.0 flannel-cni:v0.3.0 flannel:v0.9.1 etcd:v3.2.4 routereflector:v0.4.0 kubernetes-dashboard-init-amd64:v1.0.1 kubernetes-dashboard-amd64:v1.7.1)
for imageName in ${images[@]} ; do
  docker pull jicki/$imageName
done

```


## 修改配置

```
cd kubespray/inventory/group_vars

vi k8s-cluster.yml

# k8s-cluster 为控制一些基础信息的配置文件。

# all.yml 控制一些需要详细配置的信息 

#  这里打开  kubelet_load_modules: true


# API 负载均衡，否在默认都连接到第一台master (坑爹)
loadbalancer_apiserver_localhost: true


# 修改 api 密码

vi roles/kubespray-defaults/defaults/main.yaml

kube_api_pwd: xxxx



# 修改 flannel 网络的模式 默认是 vxlan 修改为 host-gw
# 注 host-gw 模式 服务器必须 二层互通

vi roles/network_plugin/flannel/defaults/main.yml

flannel_backend_type: "vxlan"

修改

flannel_backend_type: "host-gw"


## 修改 证书过期时间  -days xxxx

vi /opt/kubespray/roles/kubernetes/secrets/files/make-ssl.sh

openssl req -x509 -new -nodes -key ca-key.pem -days 10000 -out ca.pem -subj "/CN=kube-ca" > /dev/null 2>&1


openssl x509 -req -in ${name}.csr -CA ca.pem -CAkey ca-key.pem -CAcreateserial -out ${name}.pem -days 3650 -extensions v3_req -extfile ${CONFIG} > /dev/null 2>&1



## 修改所有 images 的地址为个人的仓库地址

roles/download/defaults/main.yml
roles/dnsmasq/templates/dnsmasq-autoscaler.yml.j2
roles/kubernetes-apps/ansible/defaults/main.yml


sed -i 's/gcr\.io\/google_containers/jicki/g' roles/download/defaults/main.yml

sed -i 's/gcr\.io\/google_containers/jicki/g' roles/dnsmasq/templates/dnsmasq-autoscaler.yml.j2

sed -i 's/gcr\.io\/google_containers/jicki/g' roles/kubernetes-apps/ansible/defaults/main.yml





roles/download/defaults/main.yml
roles/kubernetes-apps/local_volume_provisioner/defaults/main.yml


sed -i 's/quay\.io\/coreos/jicki/g' roles/download/defaults/main.yml

sed -i 's/quay\.io\/calico/jicki/g' roles/download/defaults/main.yml

sed -i 's/quay\.io\/external_storage/jicki/g' roles/kubernetes-apps/local_volume_provisioner/defaults/main.yml

```


## 初始化


```

cd kubespray

CONFIG_FILE=inventory/inventory.cfg python3 contrib/inventory_builder/inventory.py 172.16.1.64 172.16.1.65 172.16.1.66


# 初始化以后会生成一个 inventory.cfg 文件，如果SSH 端口不同，需要打开修改

cd kubespray/inventory

vi inventory.cfg


[all]
node1    ansible_host=172.16.1.64 ansible_port=33 ip=172.16.1.64
node2    ansible_host=172.16.1.65 ansible_port=33 ip=172.16.1.65
node3    ansible_host=172.16.1.66 ansible_port=33 ip=172.16.1.66

[kube-master]
node1
node2

[kube-node]
node1
node2
node3

[etcd]
node1
node2
node3

[k8s-cluster:children]
kube-node
kube-master

[calico-rr]

[vault]
node1
node2
node3

```



## 部署集群

```
ansible-playbook -i inventory/inventory.cfg cluster.yml -b -v --private-key=~/.ssh/id_rsa


```

```
PLAY RECAP ***************************************************************************************************************************************************************
localhost                  : ok=2    changed=0    unreachable=0    failed=0   
node1                      : ok=289  changed=24   unreachable=0    failed=0   
node2                      : ok=242  changed=13   unreachable=0    failed=0   
node3                      : ok=216  changed=0    unreachable=0    failed=0   

Friday 08 December 2017  14:52:54 +0800 (0:00:00.039)       0:05:44.482 ******* 
```

```
# 查看 flannel 模式 FLANNEL_MTU = 1450 是 vxlan FLANNEL_MTU = 1500 是 host-gw

[root@node1 kubespray]# cat /var/run/flannel/subnet.env 
FLANNEL_NETWORK=10.233.64.0/18
FLANNEL_SUBNET=10.233.66.1/24
FLANNEL_MTU=1500
FLANNEL_IPMASQ=true

```




## 查看集群


```
[root@node1 kubespray]# kubectl get nodes
NAME      STATUS    ROLES         AGE       VERSION
node1     Ready     master,node   6m        v1.8.4+coreos.0
node2     Ready     master,node   6m        v1.8.4+coreos.0
node3     Ready     node          6m        v1.8.4+coreos.0

```


```
[root@node1 kubespray]# kubectl get pods -n kube-system
NAME                                    READY     STATUS    RESTARTS   AGE
kube-apiserver-node1                    1/1       Running   0          22m
kube-apiserver-node2                    1/1       Running   0          22m
kube-controller-manager-node1           1/1       Running   0          18m
kube-controller-manager-node2           1/1       Running   0          18m
kube-dns-7ff67768b6-mx28d               3/3       Running   0          17m
kube-dns-7ff67768b6-xkjl5               3/3       Running   0          17m
kube-flannel-9fmr4                      2/2       Running   0          17m
kube-flannel-l9w4s                      2/2       Running   0          17m
kube-flannel-vktd9                      2/2       Running   0          17m
kube-proxy-node1                        1/1       Running   0          22m
kube-proxy-node2                        1/1       Running   0          22m
kube-proxy-node3                        1/1       Running   0          22m
kube-scheduler-node1                    1/1       Running   0          18m
kube-scheduler-node2                    1/1       Running   0          18m
kubedns-autoscaler-dcb9b446b-m87sx      1/1       Running   0          17m
kubernetes-dashboard-77c965686c-tq4zw   1/1       Running   0          17m
nginx-proxy-node3                       1/1       Running   0          21m

```




## 测试集群

> 验证 dns 服务，以及 网络

```
# 创建一个 deployment


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
[root@node1 yaml]# kubectl apply -f nginx-deployment.yaml 
deployment "nginx-dm" created
service "nginx-svc" created



[root@node1 yaml]# kubectl get pods -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP             NODE
nginx-dm-55b58f68b6-7l5qr   1/1       Running   0          27s       10.233.65.3    node2
nginx-dm-55b58f68b6-dkhvl   1/1       Running   0          27s       10.233.64.17   node1
nginx-dm-55b58f68b6-xsvmr   1/1       Running   0          27s       10.233.66.3    node3



[root@node1 yaml]# kubectl get svc -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes   ClusterIP   10.233.0.1      <none>        443/TCP   30m       <none>
nginx-svc    ClusterIP   10.233.38.166   <none>        80/TCP    1m        name=nginx

```


```
[root@node1 yaml]# curl 10.233.38.166


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
# 创建一个 pods

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

```


```
[root@node1 yaml]# kubectl apply -f alpine-pods.yaml 
pod "alpine" created


[root@node1 yaml]# kubectl get pods
NAME                        READY     STATUS    RESTARTS   AGE
alpine                      1/1       Running   0          19s

```



```
[root@node1 yaml]# kubectl exec -it alpine nslookup nginx-svc
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx-svc
Address 1: 10.233.38.166 nginx-svc.default.svc.cluster.local



[root@node1 yaml]# kubectl exec -it alpine nslookup kubernetes
nslookup: can't resolve '(null)': Name does not resolve

Name:      kubernetes
Address 1: 10.233.0.1 kubernetes.default.svc.cluster.local


```




## 部署 ingress

> Kubernetes 暴露服务的方式目前只有三种：LoadBlancer Service、NodePort Service、Ingress； 什么是 Ingress ? Ingress 就是利用 Nginx Haproxy 等负载均衡工具来暴露 Kubernetes 服务。
官方 Nginx Ingress github https://github.com/kubernetes/ingress-nginx



```
# 下载镜像

jicki/nginx-ingress-controller:0.9.0
jicki/defaultbackend:1.4


```



```
# 下载 yaml

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/namespace.yaml 

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/default-backend.yaml

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/configmap.yaml

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/tcp-services-configmap.yaml

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/udp-services-configmap.yaml

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/rbac.yaml

wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/with-rbac.yaml

```


```
#  查看下载

[root@node1 nginx-ingress]# ls -lt
总用量 28
-rw-r--r--. 1 root root   89 12月  8 15:36 udp-services-configmap.yaml
-rw-r--r--. 1 root root 1865 12月  8 15:36 with-rbac.yaml
-rw-r--r--. 1 root root   63 12月  8 15:36 namespace.yaml
-rw-r--r--. 1 root root 2385 12月  8 15:36 rbac.yaml
-rw-r--r--. 1 root root   89 12月  8 15:36 tcp-services-configmap.yaml
-rw-r--r--. 1 root root  129 12月  8 15:36 configmap.yaml
-rw-r--r--. 1 root root 1131 12月  8 15:36 default-backend.yaml



# 修改 镜像下载地址

sed -i 's/gcr\.io\/google_containers/jicki/g' *
sed -i 's/quay\.io\/kubernetes-ingress-controller/jicki/g' *


```


```
# 为 node2  node3 打上相同的 label

[root@node1 nginx-ingress]# kubectl label nodes node2 ingress=nginx 
node "node2" labeled
[root@node1 nginx-ingress]# kubectl label nodes node3 ingress=nginx
node "node3" labeled


# 删除 label

kubectl label node node2 ingress-

kubectl label node node3 ingress-


# 查看 label

[root@node1 nginx-ingress]# kubectl get nodes --show-labels
NAME      STATUS    ROLES         AGE       VERSION           LABELS
node1     Ready     master,node   1h        v1.8.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/hostname=node1,node-role.kubernetes.io/master=true,node-role.kubernetes.io/node=true
node2     Ready     master,node   1h        v1.8.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=node2,node-role.kubernetes.io/master=true,node-role.kubernetes.io/node=true
node3     Ready     node          59m       v1.8.4+coreos.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,ingress=proxy,kubernetes.io/hostname=node3,node-role.kubernetes.io/node=true

```


```
# 修改相关配置

vi with-rbac.yaml

# 这里配置 replicas: 2 , 上面配置了2个 labels
# 修改 yaml 文件 增加 hostNetwork   和 nodeSelector, 第二个 spec 下 增加。

    spec:
      hostNetwork: true
      serviceAccountName: nginx-ingress-serviceaccount
      nodeSelector:
        ingress: nginx

```



```
# 导入 yaml

[root@node1 nginx-ingress]# kubectl apply -f namespace.yaml 
namespace "ingress-nginx" created


[root@node1 nginx-ingress]# kubectl apply -f .
serviceaccount "nginx-ingress-serviceaccount" created
clusterrole "nginx-ingress-clusterrole" created
role "nginx-ingress-role" created
rolebinding "nginx-ingress-role-nisa-binding" created
clusterrolebinding "nginx-ingress-clusterrole-nisa-binding" created
configmap "tcp-services" created
configmap "udp-services" created
deployment "nginx-ingress-controller" created
configmap "nginx-configuration" created
deployment "default-http-backend" created
service "default-http-backend" created

```


```
#  查看服务

[root@node1 nginx-ingress]# kubectl get pods -n ingress-nginx
NAME                                       READY     STATUS    RESTARTS   AGE
default-http-backend-7f47b7d69b-wdj5j      1/1       Running   0          11s
nginx-ingress-controller-c646dfddc-6cw29   1/1       Running   0          11s
nginx-ingress-controller-c646dfddc-g9krl   1/1       Running   0          11s

```


```
# 编写 ingress yaml

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
# 查看服务

[root@node1 yaml]# kubectl get ingress
NAME            HOSTS            ADDRESS   PORTS     AGE
nginx-ingress   nginx.jicki.cn             80        1m

```


```
[root@node1 yaml]# curl nginx.jicki.cn

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



# FAQ 报错


## equalto 错误

```

# 升级 Jinja2

pip install --upgrade Jinja2

```




## Stop if swap enabled

```


# 报如下错误:

TASK [kubernetes/preinstall : Stop if swap enabled] **********************************************************************************************************************
Wednesday 06 December 2017  15:40:29 +0800 (0:00:00.064)       0:00:10.135 **** 
fatal: [node1]: FAILED! => {
    "assertion": "ansible_swaptotal_mb == 0", 
    "changed": false, 
    "evaluated_to": false, 
    "failed": true
}
fatal: [node2]: FAILED! => {
    "assertion": "ansible_swaptotal_mb == 0", 
    "changed": false, 
    "evaluated_to": false, 
▽   "failed": true
}
fatal: [node3]: FAILED! => {
    "assertion": "ansible_swaptotal_mb == 0", 
    "changed": false, 
    "evaluated_to": false, 
    "failed": true
}
        to retry, use: --limit @/opt/kubespray/cluster.retry



## 解决方法

# 在配置服务器上执行

cd /tmp

rm -rf node*

# 在每台服务器上执行

swapoff -a


# 永久关闭 swap

修改 /etc/fstab  里面 swap 的相关 mount

```


## 清理集群

```
ansible-playbook -i inventory/inventory.cfg reset.yml -b -v --private-key=~/.ssh/id_rsa

rm -rf /etc/kubernetes/
rm -rf /var/lib/kubelet
rm -rf /var/lib/etcd
rm -rf /var/lib/cni
rm -rf /usr/local/bin/kubectl
rm -rf /opt/cni
rm -rf /var/log/containers/
rm -rf /etc/systemd/system/calico-node.service
rm -rf /etc/systemd/system/kubelet.service
rm -rf /tmp/node*
rm -rf /usr/libexec/kubernetes
systemctl stop etcd.service
systemctl disable etcd.service
systemctl stop calico-node.service
systemctl disable calico-node.service
systemctl stop docker
rm -rf /opt/docker
systemctl start docker

```

