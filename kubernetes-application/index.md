# kubernetes 实战应用部署



> kubernetes 一些应用的部署， yaml 文件编写


# 1 kubernetes 应用部署


## 1.1 部署一个 nginx  rc 


> 编写 一个 nginx yaml

```
apiVersion: v1 
kind: ReplicationController 
metadata: 
  name: nginx-rc 
spec: 
  replicas: 2 
  selector: 
    name: nginx 
  template: 
    metadata: 
      labels: 
        name: nginx 
    spec: 
      containers: 
        - name: nginx 
          image: nginx 
          imagePullPolicy: IfNotPresent
          ports: 
            - containerPort: 80
```

```
[root@k8s-master ~]#kubectl get rc
NAME       DESIRED   CURRENT   READY     AGE
nginx-rc   2         2         2         2m


[root@k8s-master ~]#kubectl get pod -o wide
NAME             READY     STATUS    RESTARTS   AGE       IP          NODE
nginx-rc-2s8k9   1/1       Running   0          10m       10.32.0.3   k8s-node-1
nginx-rc-s16cm   1/1       Running   0          10m       10.40.0.1   k8s-node-2
```


> 编写一个 nginx service 让集群内部容器可以访问 (ClusterIp)


```
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
[root@k8s-master ~]#kubectl create -f nginx-svc.yaml 
service "nginx-svc" created


[root@k8s-master ~]#kubectl get svc -o wide
NAME         CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes   10.0.0.1      <none>        443/TCP   2d        <none>
nginx-svc    10.6.164.79   <none>        80/TCP    29s       name=nginx

```



> 编写一个 curl 的pods

```
apiVersion: v1
kind: Pod
metadata:
  name: curl
spec:
  containers:
  - name: curl
    image: radial/busyboxplus:curl
    command:
    - sh
    - -c
    - while true; do sleep 1; done
```


```
# 测试pods 内部通信
[root@k8s-master ~]#kubectl exec curl curl nginx
```



```
# 在任何node节点中，可使用ip访问

[root@k8s-node-1 ~]# curl 10.6.164.79
[root@k8s-node-2 ~]# curl 10.6.164.79

```


> 编写一个 nginx service 让外部可以访问 (NodePort)

```
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc-node
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  type: NodePort
  selector:
    name: nginx
```


```
[root@k8s-master ~]#kubectl get svc -o wide
NAME             CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE       SELECTOR
kubernetes       10.0.0.1       <none>        443/TCP   2d        <none>
nginx-svc        10.6.164.79    <none>        80/TCP    29m       name=nginx
nginx-svc-node   10.12.95.227   <nodes>       80/TCP    17s       name=nginx


[root@k8s-master ~]#kubectl describe svc nginx-svc-node |grep NodePort
Type:                   NodePort
NodePort:               <unset> 32669/TCP
```



```
# 使用 ALL node节点物理IP + 端口访问

http://localhost:32669
```




## 1.2 部署一个 zookeeper 集群 


> 编写 一个 zookeeper-cluster.yaml


```
apiVersion: extensions/v1beta1
kind: Deployment 
metadata: 
  name: zookeeper-1
spec: 
  replicas: 1
  template: 
    metadata: 
      labels: 
        name: zookeeper-1 
    spec: 
      containers: 
        - name: zookeeper-1
          image: zk:alpine 
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "1"
          - name: NODES
            value: "0.0.0.0,zookeeper-2,zookeeper-3"
          ports:
          - containerPort: 2181

---

apiVersion: extensions/v1beta1 
kind: Deployment
metadata:
  name: zookeeper-2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: zookeeper-2
    spec:
      containers:
        - name: zookeeper-2
          image: zk:alpine
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "2"
          - name: NODES
            value: "zookeeper-1,0.0.0.0,zookeeper-3"
          ports:
          - containerPort: 2181

---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: zookeeper-3
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: zookeeper-3
    spec:
      containers:
        - name: zookeeper-3
          image: zk:alpine
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "3"
          - name: NODES
            value: "zookeeper-1,zookeeper-2,0.0.0.0"
          ports:
          - containerPort: 2181
---

apiVersion: v1 
kind: Service 
metadata: 
  name: zookeeper-1 
  labels:
    name: zookeeper-1
spec: 
  ports: 
    - name: client
      port: 2181
      protocol: TCP
    - name: followers
      port: 2888
      protocol: TCP
    - name: election
      port: 3888
      protocol: TCP
  selector: 
    name: zookeeper-1

---

apiVersion: v1 
kind: Service 
metadata: 
  name: zookeeper-2
  labels:
    name: zookeeper-2
spec: 
  ports: 
    - name: client
      port: 2181
      protocol: TCP
    - name: followers
      port: 2888
      protocol: TCP
    - name: election
      port: 3888
      protocol: TCP
  selector: 
    name: zookeeper-2

---

apiVersion: v1 
kind: Service 
metadata: 
  name: zookeeper-3
  labels:
    name: zookeeper-3
spec: 
  ports: 
    - name: client
      port: 2181
      protocol: TCP
    - name: followers
      port: 2888
      protocol: TCP
    - name: election
      port: 3888
      protocol: TCP
  selector: 
    name: zookeeper-3
	
```


```
[root@k8s-master ~]#kubectl create -f zookeeper-cluster.yaml --record



[root@k8s-master ~]#kubectl get pods -o wide
NAME                           READY     STATUS    RESTARTS   AGE       IP          NODE
zookeeper-1-2149121414-cfyt4   1/1       Running   0          51m       10.32.0.3   k8s-node-1
zookeeper-2-2653289864-0bxee   1/1       Running   0          51m       10.40.0.1   k8s-node-2
zookeeper-3-3158769034-5csqy   1/1       Running   0          51m       10.40.0.2   k8s-node-2


[root@k8s-master ~]#kubectl get deployment -o wide    
NAME          DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
zookeeper-1   1         1         1            1           51m
zookeeper-2   1         1         1            1           51m
zookeeper-3   1         1         1            1           51m


[root@k8s-master ~]#kubectl get svc -o wide
NAME          CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE       SELECTOR
zookeeper-1   10.8.111.19    <none>        2181/TCP,2888/TCP,3888/TCP   51m       name=zookeeper-1
zookeeper-2   10.6.10.124    <none>        2181/TCP,2888/TCP,3888/TCP   51m       name=zookeeper-2
zookeeper-3   10.0.146.143   <none>        2181/TCP,2888/TCP,3888/TCP   51m       name=zookeeper-3



```



## 1.3 部署一个 kafka 集群 


> 编写 一个 kafka-cluster.yaml


```

apiVersion: extensions/v1beta1
kind: Deployment 
metadata: 
  name: kafka-deployment-1
spec: 
  replicas: 1
  template: 
    metadata: 
      labels: 
        name: kafka-1 
    spec: 
      containers: 
        - name: kafka-1
          image: kafka:alpine 
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "1"
          - name: ZK_NODES
            value: "zookeeper-1,zookeeper-2,zookeeper-3"
          ports:
          - containerPort: 9092

---

apiVersion: extensions/v1beta1 
kind: Deployment
metadata:
  name: kafka-deployment-2
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: kafka-2  
    spec:
      containers:
        - name: kafka-2
          image: kafka:alpine
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "2"
          - name: ZK_NODES
            value: "zookeeper-1,zookeeper-2,zookeeper-3"
          ports:
          - containerPort: 9092

---

apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kafka-deployment-3
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: kafka-3  
    spec:
      containers:
        - name: kafka-3
          image: kafka:alpine
          imagePullPolicy: IfNotPresent
          env:
          - name: NODE_ID
            value: "3"
          - name: ZK_NODES
            value: "zookeeper-1,zookeeper-2,zookeeper-3"
          ports:
          - containerPort: 9092

---

apiVersion: v1 
kind: Service 
metadata: 
  name: kafka-1 
  labels:
    name: kafka-1
spec: 
  ports: 
    - name: client
      port: 9092
      protocol: TCP
  selector: 
    name: kafka-1

---

apiVersion: v1 
kind: Service 
metadata: 
  name: kafka-2
  labels:
    name: kafka-2
spec: 
  ports: 
    - name: client
      port: 9092
      protocol: TCP
  selector: 
    name: kafka-2

---

apiVersion: v1 
kind: Service 
metadata: 
  name: kafka-3
  labels:
    name: kafka-3
spec: 
  ports: 
    - name: client
      port: 9092
      protocol: TCP
  selector: 
    name: kafka-3

```

