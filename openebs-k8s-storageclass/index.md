# OpenEBS to Kubernetes StorageClass



> OpenEBS是一个开源存储平台，为DevOps和容器环境提供持久和集装箱 块存储。
> 块存储一般用于 Mysql data, redis data, jenkins data, RabbitMQ data, 等数据类的存储
> 官方 github https://github.com/openebs/openebs



# OpenEBS to Kubernetes StorageClass


## 部署 OpenEBS

> 本文 基于 kubernetes 集群中部署这个 存储


```
# 下载 yaml 文件

wget https://openebs.github.io/charts/openebs-operator.yaml

# 最后 StorageClass 部分，可配置容量，等，可配置多个 StorageClass，如下:


apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
   name: openebs-standard
provisioner: openebs.io/provisioner-iscsi
parameters:
  openebs.io/storage-pool: "default"
  openebs.io/jiva-replica-count: "2"
  openebs.io/volume-monitor: "true"
  openebs.io/capacity: 5G

```






```
# 按照官方文档只需要直接导入 yaml 既可

kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml



# 查看创建

[root@kubernetes-64 openebs]# kubectl get pod
NAME                                   READY     STATUS    RESTARTS   AGE
maya-apiserver-5fc675bc6-wl2t7         1/1       Running   0          48s
openebs-provisioner-576cbb68d4-bdpwv   1/1       Running   0          48s



# 查看 StorageClass

[root@kubernetes-64 openebs]# kubectl get StorageClass
NAME               PROVISIONER                    AGE
openebs-standard   openebs.io/provisioner-iscsi   2m

```


### 测试 应用



>  这里注意，所有的node 必须安装 iscsi ，因为 openebs 是基于 iscsi 挂载的



```
[root@kubernetes-64 openebs]#  yum install -y iscsi-initiator-utils

```



```
[root@kubernetes-64 openebs]# vi test-nginx.yaml


kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: openebs-standard
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G
      
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-dm
spec:
  replicas: 1
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
          volumeMounts:
            - name: openebs-volume
              mountPath: "/usr/share/nginx/html"
      volumes:
      - name: openebs-volume
        persistentVolumeClaim:
          claimName: test-claim


```

```
# 导入配置

[root@kubernetes-64 openebs]# kubectl apply -f test-nginx.yaml 
deployment.extensions "nginx-dm" created




# 查看服务

[root@kubernetes-64 openebs]# kubectl get pods
NAME                                                             READY     STATUS    RESTARTS   AGE
maya-apiserver-5fc675bc6-wl2t7                                   1/1       Running   0          1h
nginx-dm-7554876d67-j2kz4                                        1/1       Running   0          21m
openebs-provisioner-576cbb68d4-bdpwv                             1/1       Running   0          1h
pvc-89c8c04b-2be9-11e8-93d3-44a8420b9988-ctrl-865fbf6bb6-f2n9q   2/2       Running   0          54m
pvc-89c8c04b-2be9-11e8-93d3-44a8420b9988-rep-f97764db5-d9pj8     1/1       Running   0          54m
pvc-89c8c04b-2be9-11e8-93d3-44a8420b9988-rep-f97764db5-lvzjn     1/1       Running   0          54m


```




```
# 一个完整的 jenkins 例子


kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: jenkins-claim
  annotations:
    volume.beta.kubernetes.io/storage-class: openebs-standard
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5G

---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: jenkins
  name: jenkins-admin
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
  labels:
    k8s-app: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: default
---
apiVersion: apps/v1beta1
kind: Deployment  
metadata:  
  name: jenkins
spec:  
  replicas: 1  
  template:  
    metadata:  
      labels:  
        app: jenkins  
    spec:  
      securityContext:
        fsGroup: 1000
      serviceAccount: "jenkins-admin"
      containers:  
      - name: jenkins
        image: jenkins/jenkins
        imagePullPolicy: IfNotPresent  
        ports:  
        - containerPort: 8080  
          name: web  
          protocol: TCP  
        - containerPort: 50000  
          name: agent  
          protocol: TCP  
        volumeMounts:  
        - name: jenkinshome
          mountPath: /var/jenkins_home
        env:  
        - name: JAVA_OPTS  
          value: "-Xms512m -Xmx1024m -XX:PermSize=512m -XX:MaxPermSize=1024m -Duser.timezone=Asia/Shanghai"
        - name: TRY_UPGRADE_IF_NO_MARKER
          value: "true"
      volumes:  
      - name: jenkinshome
        persistentVolumeClaim:
          claimName: jenkins-claim

---
kind: Service  
apiVersion: v1  
metadata:  
  labels:  
      app: jenkins  
  name: jenkins
spec:  
  ports:  
  - port: 8080  
    targetPort: 8080  
    name: web  
  - port: 50000  
    targetPort: 50000  
    name: agent  
  selector:
    app: jenkins
    
    
---
apiVersion: extensions/v1beta1  
kind: Ingress  
metadata:  
  name: jenkins
spec:  
  rules:  
  - host: jenkins.jicki.cn
    http:  
      paths:  
      - path: /  
        backend:  
          serviceName: jenkins  
          servicePort: 8080


```


```
# 导入文件

[root@kubernetes-64 openebs]# kubectl apply -f jenkins-openebs.yaml 
persistentvolumeclaim "jenkins-claim" created
serviceaccount "jenkins-admin" unchanged
clusterrolebinding.rbac.authorization.k8s.io "jenkins-admin" configured
deployment.apps "jenkins" created
service "jenkins" created
ingress.extensions "jenkins" created




# 查看服务

[root@kubernetes-64 openebs]# kubectl get pods
NAME                                                             READY     STATUS    RESTARTS   AGE
jenkins-847bf4f5b-knr8j                                          1/1       Running   0          21m
maya-apiserver-5fc675bc6-zh4r9                                   1/1       Running   0          1h
openebs-provisioner-576cbb68d4-s2dqt                             1/1       Running   0          1h
pvc-6ea040f0-2c21-11e8-93d3-44a8420b9988-ctrl-79f9fb7854-l7vtw   2/2       Running   0          21m
pvc-6ea040f0-2c21-11e8-93d3-44a8420b9988-rep-5c55bb6788-85g56    1/1       Running   0          21m
pvc-6ea040f0-2c21-11e8-93d3-44a8420b9988-rep-5c55bb6788-vmr9t    1/1       Running   0          21m


```



```
# 查看 密码

[root@kubernetes-64 openebs]# kubectl logs jenkins-847bf4f5b-knr8j


```



### 验证存储

```
# 在 web ui 里创建一个 job


# 查看 pod ，位于 kubernetes-66 中

[root@kubernetes-64 openebs]# kubectl get pods -o wide
NAME                                                             READY     STATUS    RESTARTS   AGE       IP             NODE
jenkins-847bf4f5b-cfghx                                          1/1       Running   0          4m        10.254.78.6    kubernetes-66



# 删除 pods

[root@kubernetes-64 openebs]# kubectl delete pods/jenkins-847bf4f5b-cfghx
pod "jenkins-847bf4f5b-cfghx" deleted


# 查看重建状态 转移至 kubernetes-65 中

[root@kubernetes-64 openebs]# kubectl get pods -o wide                   
NAME                                                             READY     STATUS              RESTARTS   AGE       IP             NODE
jenkins-847bf4f5b-4dtqx                                          0/1       ContainerCreating   0          6s        <none>         kubernetes-65
jenkins-847bf4f5b-cfghx                                          0/1       Terminating         0          5m        10.254.78.6    kubernetes-66




[root@kubernetes-64 openebs]# kubectl get pods -o wide
NAME                                                             READY     STATUS    RESTARTS   AGE       IP             NODE
jenkins-847bf4f5b-4dtqx                                          1/1       Running   0          38s       10.254.126.8   kubernetes-65



# 验证~ jenkins 数据是否存在

```

