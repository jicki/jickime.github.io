# CI/CD with Drone Kubernetes and Gogs



# Drone

* Drone is a Container-Native, Continuous Delivery Platform。 我们可以理解为 Drone 是基于 容器原生的一个 持续集成、持续交付的平台。 

* 官方网站是 `https://drone.io/`


## Mysql

* 对于 Drone 与 Gogs 都会使用到 Mysql

* 注: 无论如何并不建议在 容器化环境下 部署 数据层


---

* 部署 Mysql 的 Yaml 文件


* `Secret` 使用 `base64` 加密 (# echo -n "mysql"| base64 ) 


```shell
# mysql-namespaces.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: database
```

---

```shell
# mysql-secret.yaml

apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: database
data:
  root_pass: cmxkYkAxMjM=
```

---


```shell
# mysql-statefulset.yaml

---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: database
  labels:
    app: mysql
    service: database
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
      service: database
  template:
    metadata:
      labels:
        app: mysql
        service: database
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mysql
        image: mysql:8
        args:
        - --character-set-server=utf8mb4
        - --collation-server=utf8mb4_unicode_ci
        ports:
        - name: http
          containerPort: 3306
        env:
         - name: MYSQL_ROOT_PASSWORD
           valueFrom:
             secretKeyRef:
               name: mysql-secret
               key: root_pass
        resources:
          requests:
            cpu: "0.01"
        volumeMounts:
        - name: mysql-data-storage
          mountPath: /var/lib/mysql
        - name: tz-config
          mountPath: /etc/localtime
          readOnly: true
      restartPolicy: Always
      volumes:
      - name: mysql-data-storage
        hostPath:
          path: /opt/mysql/data
          type: Directory
      - name: tz-config
        hostPath:
          path: /etc/localtime

---
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: database
  labels:
    app: mysql
    service: database
spec:
  ports:
  - name: http
    protocol: TCP
    port: 3306
    targetPort: 3306
  selector:
    app: mysql
    service: database
  type: ClusterIP
```


---

```shell
# 查看验证服务

[root@jicki mysql]# kubectl get pods,statefulset,svc -n database
NAME          READY   STATUS    RESTARTS   AGE
pod/mysql-0   1/1     Running   0          6m1s

NAME                     READY   AGE
statefulset.apps/mysql   1/1     6m1s

NAME            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/mysql   ClusterIP   10.104.238.35   <none>        3306/TCP   6m1s

```



## Gogs

* Gogs 是一款 Go 语言开发轻量级的 Git 仓库, 是全完开源的。


### 部署 Gogs

* Gogs with Mysql to Kubernetes Deployment


* 在 WebUI 下初始化配置, 可能会遇到 `Error 1049: Unknown database 'gogs'` 需要在数据库先创建 用户 以及 数据库


* 相关 yaml 文件

```shell
# gogs-namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: git

```

---

```shell

# gogs-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: gogs
  namespace: git
  labels:
    app: gogs
spec:
  selector:
    matchLabels:
      app: gogs
  template:
    metadata:
      labels:
        app: gogs
    spec:
      containers:
      - name: gogs
        image: gogs/gogs
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 3000
          name: gogs
        - containerPort: 22
          name: ssh
        resources:
          requests:
            cpu: "0.01"
        volumeMounts:
        - name: gogs-data-storage
          mountPath: /data
        - name: tz-config
          mountPath: /etc/localtime
          readOnly: true
      restartPolicy: Always
      volumes:
      - name: gogs-data-storage
        hostPath:
          path: /opt/gogs/data
          type: Directory
      - name: tz-config
        hostPath:
          path: /etc/localtime

---
apiVersion: v1
kind: Service
metadata:
  name: gogs
  namespace: git
spec:
  ports:
  - name: gogs
    protocol: TCP
    port: 3000
    targetPort: 3000
  selector:
    app: gogs

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: gogs-ingress
  namespace: git
spec:
  rules:
  - host: gogs.jicki.cn
    http:
      paths:
      - backend:
          serviceName: gogs
          servicePort: 3000
```  



---


```shell
# 创建服务

[root@jicki gogs]# kubectl apply -f gogs-deployment.yaml
namespace/git created
deployment.apps/gogs created
service/gogs created
ingress.extensions/gogs-ingress created

```


---


* 初始化 gogs


{{< figure src="/img/posts/drone/drone-gogs.png" >}}


{{< figure src="/img/posts/drone/drone-gogs-1.png" >}}


