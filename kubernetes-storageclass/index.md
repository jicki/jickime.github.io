# Kubernetes Storage Class




# Kubernetes Storage Class

> StorageClass 可以定义多个 StorageClass 对象，并可以分别指定存储插件、设置参数，用于提供不同的存储卷。这样的设计让集群管理员能够在同一个集群内，定义和提供不同类型的、不同参数的卷（相同或者不同的存储系统）。这样的设计还确保了最终用户在无需了解太多的情况下，有能力选择不同的存储选项。本质上是为底层存储提供者描绘了蓝图，以及各种参数。


# Ceph RBD StorageClass

> Ceph RBD 集群部署 这里就略过了，可查看我之前的文章 https://jicki.cn/2017/05/09/kubernetes-ceph-rbd


## 创建 ceph pool


```
# 预先 创建 需要用到的 pool = data

[root@ceph-node-1 ~]# ceph osd pool create data 128


# 128 为 pg 数 的计算公式一般为

若少于5个OSD， 设置pg_num为128。
5~10个OSD，设置pg_num为512。
10~50个OSD，设置pg_num为4096。
超过50个OSD，可以参考计算公式。

pg数 = osd 数量 * 100 / pool 复制份数 / pool 数量

# 查看 pool 复制份数, 既 ceph.conf 里设置的 osd_pool_default_size

ceph osd dump |grep size|grep rbd

# 当 osd  pool复制数  pool 数量 变更时，应该重新计算并变更 pg 数
# 变更 pg_num 的时候 应该将 pgp_num 的数量一起变更，否则无法报错

ceph osd pool set rbd pg_num 256
ceph osd pool set rbd pgp_num 256


# 查看 创建的 pool

[root@ceph-node-1 ceph-cluster]# ceph osd lspools
0 rbd,1 data,


# 查看 pool 具体参数

[root@ceph-node-1 ceph-cluster]# ceph osd dump |grep pool

pool 0 'rbd' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 64 pgp_num 64 last_change 27 flags hashpspool stripe_width 0
pool 1 'data' replicated size 2 min_size 1 crush_ruleset 0 object_hash rjenkins pg_num 128 pgp_num 128 last_change 102 flags hashpspool stripe_width 0


# 查看 pool 下的 image

[root@ceph-node-1 ceph-cluster]# rbd -p data list
kubernetes-dynamic-pvc-fbd43f54-96a6-11e7-ac97-44a8420b9988


[root@ceph-node-1 ceph-cluster]# rbd -p data info kubernetes-dynamic-pvc-fbd43f54-96a6-11e7-ac97-44a8420b9988
rbd image 'kubernetes-dynamic-pvc-fbd43f54-96a6-11e7-ac97-44a8420b9988':
        size 51200 MB in 12800 objects
        order 22 (4096 kB objects)
        block_name_prefix: rb.0.5f29.74b0dc51
        format: 1
```

## 创建 secret 认证


```
# 创建 ceph-secret （如果之前创建过 secret  , delete 掉）

# 获取 client.admin 的值
[root@ceph-node-1 ceph-cluster]# ceph auth get-key client.admin
AQBpUAxZGmDBBxAAVbgeRss9jv39dE0biTE7qQ==

# 创建 secret ( type 类型必须为 kubernetes.io/rbd )

kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" --from-literal=key='AQBpUAxZGmDBBxAAVbgeRss9jv39dE0biTE7qQ==' --namespace=kube-system

kubectl create secret generic ceph-secret --type="kubernetes.io/rbd" --from-literal=key='AQBpUAxZGmDBBxAAVbgeRss9jv39dE0biTE7qQ==' --namespace=default



# 查看 状态
[root@k8s-master-25 storageclass]# kubectl get secret -n kube-system |grep ceph
ceph-secret                                kubernetes.io/rbd                     1         49s

```


## 配置 storageclass

```
# 创建一个 storageclass 文件

vi test-storageclass.yaml


apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: ceph-rbd-test
provisioner: kubernetes.io/rbd
parameters:
  monitors: 172.16.1.37:6789,172.16.1.38:6789,172.16.1.39:6789
  adminId: admin
  adminSecretName: ceph-secret
  adminSecretNamespace: kube-system
  pool: data
  userId: admin
  userSecretName: ceph-secret





# 配置说明: (主要是 parameters 字段下)

monitors: Ceph Mon 的地址，以,隔开

adminId:  Ceph客户端用于创建块设备的用户(不是k8s用户)，默认为 admin 用户；

adminSecretNamespace: admin 的 namespaces

adminSecret：admin的SecretID

pool： RBD的pool存储池

userId: 用于块设备映射的用户ID，默认可以和admin一致

userSecretName： Ceph-Secret的ID



# 导入 yaml 文件

[root@k8s-master-25 storageclass]# kubectl apply -f test-storageclass.yaml 
storageclass "ceph-rbd-test" created


# 查看

[root@k8s-master-25 storageclass]# kubectl get StorageClass
NAME            TYPE
ceph-rbd-test   kubernetes.io/rbd

```


## 创建 pvc

```
# 创建基于 storageclass 的 pvc


vi storageclass-pvc.yaml


kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: storageclass-pvc
  annotations: 
    volume.beta.kubernetes.io/storage-class: ceph-rbd-test
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi



# 导入 yaml 文件

[root@k8s-master-25 storageclass]# kubectl apply -f storageclass-pvc.yaml 
persistentvolumeclaim "storageclass-pvc" created


# 查看

[root@k8s-master-25 storageclass]# kubectl get pvc
NAME               STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS    AGE
storageclass-pvc   Bound     pvc-b1faef32-96a4-11e7-bf40-44a8420b9988   50Gi       RWO           ceph-rbd-test   30m

```


## 创建测试 StatefulSet

> storageclass 主要用于 有状态服务 statefulset 的volume 使用


```
# 创建一个测试 的 StatefulSet 
# svc 中必须配置  clusterIP: None , 因为 StatefulSet 需要 Headless 服务


apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx"
  replicas: 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
      annotations:
        volume.beta.kubernetes.io/storage-class: ceph-rbd-test
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi
          
---

apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx



## 查看 pods

[root@k8s-master-25 storageclass]# kubectl get pods
NAME      READY     STATUS    RESTARTS   AGE
web-0     1/1       Running   0          1m
web-1     1/1       Running   0          1m
web-2     1/1       Running   0          1m
web-3     1/1       Running   0          1m



## 查看自动创建的 pvc

[root@k8s-master-25 storageclass]# kubectl get pvc
NAME        STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS    AGE
www-web-0   Bound     pvc-c927b383-96cb-11e7-bf40-44a8420b9988   10Gi       RWO           ceph-rbd-test   2m
www-web-1   Bound     pvc-cdfb501e-96cb-11e7-bf40-44a8420b9988   10Gi       RWO           ceph-rbd-test   1m
www-web-2   Bound     pvc-d27843f0-96cb-11e7-bf40-44a8420b9988   10Gi       RWO           ceph-rbd-test   1m
www-web-3   Bound     pvc-d571ef22-96cb-11e7-bf40-44a8420b9988   10Gi       RWO           ceph-rbd-test   1m



## 查看 svc 

[root@k8s-master-25 storageclass]# kubectl get svc -l app=nginx
NAME      CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
nginx     None         <none>        80/TCP    4m



## 测试 dns 以及 Headless 的 dns 服务

[root@k8s-master-25 storageclass]# kubectl exec -it alpine -- nslookup nginx
nslookup: can't resolve '(null)': Name does not resolve

Name:      nginx
Address 1: 10.233.104.72 web-2.nginx.default.svc.cluster.local
Address 2: 10.233.209.135 web-3.nginx.default.svc.cluster.local
Address 3: 10.233.235.68 web-0.nginx.default.svc.cluster.local
Address 4: 10.233.246.196 web-1.nginx.default.svc.cluster.local



[root@k8s-master-25 storageclass]# kubectl exec -it alpine -- nslookup web-0.nginx
nslookup: can't resolve '(null)': Name does not resolve

Name:      web-0.nginx
Address 1: 10.233.235.68 web-0.nginx.default.svc.cluster.local

```

