# CI/CD with Drone Kubernetes and Gogs



# Drone

* Drone is a Container-Native, Continuous Delivery Platform。 我们可以理解为 Drone 是基于 容器原生的一个 持续集成、持续交付的平台。 

* 官方网站是 `https://drone.io/`

* Drone 支持的Git仓库有

  1. `GitHub`
  2. `GitLab`
  3. `Gogs`
  4. `Gitea`
  5. `Bitbucket Cloud`
  6. `Bitbucket Server`


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





## Drone-Server


* `level=fatal msg="main: invalid configuration" error="Invalid port configuration. See https://discourse.drone.io/t/drone-server-changing-ports-protocol/4144"`

  * 这个问题是 k8s 与 drone 之间的命名问题, 官方竟然一直不解决或者明确说明。 

    * 解决方式一: 在创建 `deployment`、`StatefulSet`、`service` 不能创建名字为 `drone-server` 的服务。  

    * 解决方式二: 配置 `DRONE_SERVER_PORT=:80` 变量

```shell
# drone-namespace.yaml 

apiVersion: v1
kind: Namespace
metadata:
  name: drone
```


---

* 这里配置 `gogs_git_user` 需要单独在 gogs 里创建一个有所有组织权限的账户, 或者是管理员。


* `drone_agent_secret` 是用于 drone-agent 与 drone-server 进行通讯的 秘钥。

  * 使用 `openssl rand -hex 16` 进行生成。


```shell
# drone-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: drone
  namespace: drone
data:
  drone_gogs_server: http://gogs.jicki.cn
  drone_agents_enabled: 'true'
  drone_rpc_secret: 'ff7848cbd12a26c133fb6136301371c0'
  drone_db_driver: mysql
  drone_db_datasource: root:123456@tcp(mysql.database.svc.cluster.local:3306)/drone?parseTime=true
  drone_server_host: drone.jicki.cn
  drone_server_proto: http
```

---


```shell
# drone-server-statefulset.yaml

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: drone
  namespace: drone
  labels:
    app: drone
    service: cicd
spec:
  serviceName: drone
  replicas: 1
  selector:
    matchLabels:
      app: drone
      service: cicd
  template:
    metadata:
      labels:
        app: drone
        service: cicd
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: drone
        image: drone/drone
        ports:
        - name: http
          containerPort: 80
        - name: https
          containerPort: 443
        #securityContext:
        #  capabilities: {}
        #  privileged: true
        env:
        # Drone-ConfigMap
        - name: DRONE_AGENTS_ENABLED
          valueFrom:
            configMapKeyRef:
              name: drone
              key: drone_agents_enabled
        - name: DRONE_GOGS_SERVER
          valueFrom:
            configMapKeyRef:
              name: drone
              key: drone_gogs_server
        - name: DRONE_RPC_SECRET
          valueFrom:
            configMapKeyRef:
              name: drone
              key: drone_rpc_secret
        - name: DRONE_SERVER_HOST
          valueFrom:
            configMapKeyRef:
              name: drone
              key: drone_server_host
        - name: DRONE_SERVER_PROTO
          valueFrom:
            configMapKeyRef:
              name: drone
              key: drone_server_proto
        - name: DRONE_DATABASE_DRIVER
          valueFrom:
            configMapKeyRef:
              name: drone
              key: drone_db_driver
        - name: DRONE_DATABASE_DATASOURCE
          valueFrom:
            configMapKeyRef:
              name: drone
              key: drone_db_datasource
        volumeMounts:
        - name: drone-data-storage
          mountPath: /var/lib/drone
        - name: tz-config
          mountPath: /etc/localtime
          readOnly: true
      restartPolicy: Always
      volumes:
      - name: drone-data-storage
        hostPath:
          path: /opt/drone/data
          type: Directory
      - name: tz-config
        hostPath:
          path: /etc/localtime
---
apiVersion: v1
kind: Service
metadata:
  name: drone
  namespace: drone
  labels:
    app: drone
    service: cicd
spec:
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 80
  - name: https
    protocol: TCP
    port: 443
    targetPort: 443
  selector:
    app: drone
    service: cicd


---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: drone-ingress
  namespace: drone
spec:
  rules:
  - host: drone.jicki.cn
    http:
      paths:
      - backend:
          serviceName: drone
          servicePort: 80

```


---


```shell
# 导入yaml 文件

[root@jicki drone]# kubectl apply -f .
namespace/drone created
configmap/drone created
statefulset.apps/drone created
service/drone created
ingress.extensions/drone-ingress created

```



---


```shell
# 查看相关服务

[root@jicki drone]# kubectl get pods,svc,ingress -n drone
NAME          READY   STATUS    RESTARTS   AGE
pod/drone-0   1/1     Running   0          5m21s

NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
service/drone   ClusterIP   10.100.29.28   <none>        80/TCP,443/TCP   12m

NAME                               CLASS    HOSTS            ADDRESS         PORTS   AGE
ingress.extensions/drone-ingress   <none>   drone.jicki.cn   10.99.155.236   80      12m

```


---


### 访问WebUI

* 这里特别注意, 因为我们关联的是 gogs 的 git , 所以这里登录 `drone` 的时候, 使用 `gogs` 的账号密码。


{{< figure src="/img/posts/drone/drone-web-1.png" >}}

{{< figure src="/img/posts/drone/drone-web-2.png" >}}

{{< figure src="/img/posts/drone/drone-web-3.png" >}}




## drone-agent




