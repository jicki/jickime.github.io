# kubernetes 1.4.5



> 更新 kubernetes 1.4.5 , 配置文档 



# 1 初始化环境


## 1.1 环境：

|节点|IP|
|----|----|
|node-1|10.6.0.140|
|node-2|10.6.0.187|
|node-3|10.6.0.188|


## 1.2 设置hostname

```
hostnamectl --static set-hostname hostname
```

|IP|hostname|
|----|----|
|10.6.0.140|k8s-master|
|10.6.0.187|k8s-node-1|
|10.6.0.188|k8s-node-2|




## 1.3 配置 hosts

```
vi /etc/hosts
```

|IP|hostname|
|----|----|
|10.6.0.140|k8s-master|
|10.6.0.187|k8s-node-1|
|10.6.0.188|k8s-node-2|




# 2 部署 kubernetes master



## 2.1 添加yum

```
------------------------------------------------------
cat <<EOF> /etc/yum.repos.d/k8s.repo
[kubelet]
name=kubelet
baseurl=http://files.rm-rf.ca/rpms/kubelet/
enabled=1
gpgcheck=0
EOF
------------------------------------------------------


yum makecache

yum install -y socat kubelet kubeadm kubectl kubernetes-cni

```


## 2.2 安装docker 


```
wget -qO- https://get.docker.com/ | sh


systemctl enable docker
systemctl start docker
```



## 2.3 下载镜像

```
images=(kube-proxy-amd64:v1.4.5 kube-discovery-amd64:1.0 kubedns-amd64:1.7 kube-scheduler-amd64:v1.4.5 kube-controller-manager-amd64:v1.4.5 kube-apiserver-amd64:v1.4.5 etcd-amd64:2.2.5 kube-dnsmasq-amd64:1.3 exechealthz-amd64:1.1 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.4.1)
for imageName in ${images[@]} ; do
  docker pull jicki/$imageName
  docker tag jicki/$imageName gcr.io/google_containers/$imageName
  docker rmi jicki/$imageName
done

```


## 2.4 启动 kubernetes

```
systemctl enable kubelet
systemctl start kubelet
```



## 2.5 创建集群

```
kubeadm init --api-advertise-addresses=10.6.0.140 --use-kubernetes-version v1.4.5
```


## 2.6 记录 token
```
kubernetes master initialised successfully!

You can now join any number of machines by running the following on each node:

kubeadm join --token=cf9fd4.7a477d4299305b93 10.6.0.140
```


## 2.7 检查 kubelet 状态

```
systemctl status kubelet
```



# 3 部署 kubernetes node


## 3.1 安装docker

```
wget -qO- https://get.docker.com/ | sh


systemctl enable docker
systemctl start docker
```



## 3.2 下载镜像

```
images=(kube-proxy-amd64:v1.4.5 kube-discovery-amd64:1.0 kubedns-amd64:1.7 kube-scheduler-amd64:v1.4.5 kube-controller-manager-amd64:v1.4.5 kube-apiserver-amd64:v1.4.5 etcd-amd64:2.2.5 kube-dnsmasq-amd64:1.3 exechealthz-amd64:1.1 pause-amd64:3.0 kubernetes-dashboard-amd64:v1.4.1)
for imageName in ${images[@]} ; do
  docker pull jicki/$imageName
  docker tag jicki/$imageName gcr.io/google_containers/$imageName
  docker rmi jicki/$imageName
done

```


## 3.3 启动 kubernetes

```
systemctl enable kubelet
systemctl start kubelet
```


## 3.4 加入集群


```
kubeadm join --token=cf9fd4.7a477d4299305b93 10.6.0.140
```

```
Node join complete:
* Certificate signing request sent to master and response
  received.
* Kubelet informed of new secure connection details.

Run 'kubectl get nodes' on the master to see this machine join.
```



## 3.5 查看集群状态


```
[root@k8s-master ~]#kubectl get nodes
NAME         STATUS    AGE
k8s-master   Ready     8m
k8s-node-1   Ready     1m
k8s-node-2   Ready     1m

```






# 4 设置 kubernetes

## 4.1 配置 POD 网络

```
kubectl apply -f https://git.io/weave-kube

daemonset "weave-net" created
```


## 4.2 查看系统服务状态


```
# kube-dns 必须配置完网络才能 Running 

[root@k8s-master ~]#kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY     STATUS    RESTARTS   AGE
kube-system   dummy-2088944543-io6ca               1/1       Running   0          22m
kube-system   etcd-k8s-master                      1/1       Running   0          22m
kube-system   kube-apiserver-k8s-master            1/1       Running   0          22m
kube-system   kube-controller-manager-k8s-master   1/1       Running   0          20m
kube-system   kube-discovery-982812725-rm6ut       1/1       Running   0          22m
kube-system   kube-dns-2247936740-htw22            3/3       Running   0          21m
kube-system   kube-proxy-amd64-lo0hr               1/1       Running   0          15m
kube-system   kube-proxy-amd64-t3qpn               1/1       Running   0          15m
kube-system   kube-proxy-amd64-wwj2z               1/1       Running   0          21m
kube-system   kube-scheduler-k8s-master            1/1       Running   0          21m
kube-system   weave-net-6k3ha                      2/2       Running   0          11m
kube-system   weave-net-auf0c                      2/2       Running   0          11m
kube-system   weave-net-bxj6d                      2/2       Running   0          11m
```



## 4.3 配置 flannel 网络

```
# 必须使用 --pod-network-cidr 指定网段(10.244.0.0/16 是因为 kube-flannel.yml 文件里面写了这个网段)

kubeadm init --api-advertise-addresses 10.6.0.140 --use-kubernetes-version v1.4.5 --pod-network-cidr 10.244.0.0/16
```

```
# 导入yml文件
kubectl create -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```






## 4.4 其他主机控制集群


```
# 备份master节点的 配置文件

/etc/kubernetes/admin.conf

# 保存至 其他电脑, 通过执行配置文件控制集群

kubectl --kubeconfig ./admin.conf get nodes

```


## 4.5 配置dashboard


```
#下载 yaml 文件, 直接导入会去官方拉取images

curl -O https://rawgit.com/kubernetes/dashboard/master/src/deploy/kubernetes-dashboard.yaml


#编辑 yaml 文件

vi kubernetes-dashboard.yaml

image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.4.0

修改为

image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.4.1


imagePullPolicy: Always

修改为

imagePullPolicy: IfNotPresent

```


```
kubectl create -f ./kubernetes-dashboard.yaml

deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created
```


```
# 查看 NodePort ，既外网访问端口

kubectl describe svc kubernetes-dashboard --namespace=kube-system

NodePort:               <unset> 31736/TCP

```


```
# 访问 dashboard

http://localhost:31736

```

![此处输入图片的描述][1]



# FAQ:


## kube-discovery error

```
	failed to create "kube-discovery" deployment [deployments.extensions "kube-discovery" already exists]
	

systemctl stop kubelet;
docker rm -f -v $(docker ps -q);
find /var/lib/kubelet | xargs -n 1 findmnt -n -t tmpfs -o TARGET -T | uniq | xargs -r umount -v;
rm -r -f /etc/kubernetes /var/lib/kubelet /var/lib/etcd;

systemctl start kubelet

kubeadm init

```


  [1]: https://jicki.cn/img/posts/kubernetes/1.png

