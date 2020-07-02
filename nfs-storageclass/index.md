# NFS - StorageClass



#  基于 StorageClass 的 NFS 动态卷

> 

## 基本信息

```
172.16.1.64  K8s-Master and node

172.16.1.65  K8s-Master and node

172.16.1.66  K8s-Master and node and NFS Server

```




## NFS 服务

```
# 安装 NFS

yum -y install nfs-utils rpcbind
```


```
# k8s 所有节点 安装 NFS 客户端

yum -y install nfs-utils

```


## 配置 NFS 目录与权限

```
vi /etc/exports

增加

/opt/nfsdata   172.16.1.0/24(rw,sync,no_root_squash)

```

## 启动 NFS 服务

```
systemctl enable rpcbind.service    
systemctl enable nfs-server.service

systemctl start rpcbind.service    
systemctl start nfs-server.service


# 查看信息

showmount -e 172.16.1.66

Export list for 172.16.1.66:
/opt/nfsdata 172.16.1.0/24
```



## 配置 NFS Client Provisioner

```
# 官网镜像地址
quay.io/external_storage/nfs-client-provisioner:latest


# 个人镜像地址

jicki/nfs-client-provisioner:latest
```





```
# 配置一个 rbac.yaml

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
  
---

kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: nfs-client-provisioner-runner
rules:
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]
  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]

---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: run-nfs-client-provisioner
subjects:
  - kind: ServiceAccount
    name: nfs-client-provisioner
    namespace: default
roleRef:
  kind: ClusterRole
  name: nfs-client-provisioner-runner
  apiGroup: rbac.authorization.k8s.io

```



```
# 配置一个 deployment 服务


kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccount: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: jicki/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 172.16.1.66
            - name: NFS_PATH
              value: /opt/nfsdata
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.16.1.66
            path: /opt/nfsdata

```


## 创建 服务

```
kubectl apply -f .
serviceaccount "nfs-client-provisioner" created
clusterrole "nfs-client-provisioner-runner" created
clusterrolebinding "run-nfs-client-provisioner" created
deployment "nfs-client-provisioner" created


# 查看服务

kubectl get pods |grep nfs
nfs-client-provisioner-8cdb56f4d-l8vmr   1/1       Running   0          26s

```

## 创建 StorageClass 


```
# nfs-storageclass

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storage 
provisioner: fuseim.pri/ifs  # fuseim.pri/ifs 是 nfs-client-provisioner 服务中的一个 env

```




```
# 导入文件
kubectl apply -f nfs-storageclass.yaml 
storageclass "nfs-storage" created


#  查看服务
kubectl get storageclass
NAME          PROVISIONER
nfs-storage   fuseim.pri/ifs

```


# 测试

## 创建一个 nginx StatefulSet


```
# nginx-statefulset

apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 2
  volumeClaimTemplates:
  - metadata:
      name: html 
      annotations:
        volume.beta.kubernetes.io/storage-class: "nfs-storage" # 这里配置 上面创建的 storageclass 的名称
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 2Gi 
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - mountPath: "/usr/share/nginx/html/"
          name: html

```



```
# 导入nginx-statefulset
kubectl apply -f nginx-statefulset.yaml 
statefulset "web" created


# 查看服务
kubectl get pods|grep web
web-0                                    1/1       Running   0          1m
web-1                                    1/1       Running   0          1m



# 查看 pvc

kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
html-web-0   Bound     pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    1m
html-web-1   Bound     pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    1m

```



# 验证 测试 挂载


## 缩容 测试

```
# 查看服务

kubectl get pods|grep web
web-0                                    1/1       Running   0          1m
web-1                                    1/1       Running   0          1m



# 查看 nfs-server 中的目录

ll
总用量 8
drwxrwxrwx 2 root root 4096 10月 18 10:19 default-html-web-0-pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2
drwxrwxrwx 2 root root 4096 10月 18 10:19 default-html-web-1-pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2



# 写入文件到挂载目录中

kubectl exec -it web-1 -- touch /usr/share/nginx/html/jicki.txt


kubectl exec -it web-1 -- ls -lt /usr/share/nginx/html
total 0
-rw-r--r--    1 root     root             0 Oct 18 02:24 jicki.txt


# nfs server 中的文件

ls default-html-web-1-pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2/
jicki.txt


# 收缩  pod = 1个 既 缩掉 web-1

kubectl scale statefulset web --replicas=1
statefulset "web" scaled



# 查看 pod 
kubectl get pods |grep web
web-0                                    1/1       Running   0          7m


# 查看 pvc (发现 web-1 的 pvc 仍然在, 而且是 Bound 状态)

kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
html-web-0   Bound     pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    8m
html-web-1   Bound     pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    8m



# 恢复 web-1

kubectl scale statefulset web --replicas=2
statefulset "web" scaled


# 查看服务

kubectl get pods |grep web
web-0                                    1/1       Running   0          13m
web-1                                    1/1       Running   0          22s


# 查看pvc 

kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
html-web-0   Bound     pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    13m
html-web-1   Bound     pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    13m


# 查看之前创建的文件 ( 发现恢复了 )


[root@k8s-master-64 yaml]# kubectl exec -it web-1 -- ls -lt /usr/share/nginx/html
total 0
-rw-r--r--    1 root     root             0 Oct 18 02:24 jicki.txt

```


## 删除 statefulset 测试


```
# delete 掉

 kubectl delete -f nginx-statefulset.yaml 
statefulset "web" deleted


# 查看 statefulset

kubectl get statefulset
No resources found.


# 查看 pvc (发现 仍然 存在)

kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
html-web-0   Bound     pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    16m
html-web-1   Bound     pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    16m


# nfs server 中的文件 (数据仍然存在)

ls default-html-web-1-pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2/
jicki.txt


# 重新创建 相同名称的 statefulset

kubectl get pods |grep web
web-0                                    1/1       Running   0          3m
web-1                                    1/1       Running   0          3m


# 查看 文件，可以看到 文件仍然存在

kubectl exec -it web-1 -- ls -lt /usr/share/nginx/html
total 0
-rw-r--r--    1 root     root             0 Oct 18 02:24 jicki.txt


```


## 删除 pvc 测试


```
# 收缩 replicas = 1 

kubectl scale statefulsets web --replicas=1

# 查看 pvc

 kubectl get pvc
NAME         STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
html-web-0   Bound     pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    22h
html-web-1   Bound     pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2   2Gi        RWO            nfs-storage    22h


# 删除 pvc 

kubectl delete pvc/html-web-1
persistentvolumeclaim "html-web-1" deleted


# nfs server (原目录更改为 archived 开头)

archived-default-html-web-1-pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2
default-html-web-0-pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2


# 重新扩容 replicas = 2

kubectl scale statefulsets web --replicas=2
statefulset "web" scaled


# nfs server (生成新的目录)


archived-default-html-web-1-pvc-bc3478ac-b3aa-11e7-b194-80d4a5d413e2  default-html-web-1-pvc-4a64a8dc-b469-11e7-b194-80d4a5d413e2
default-html-web-0-pvc-bb0c0ada-b3aa-11e7-b194-80d4a5d413e2

```

