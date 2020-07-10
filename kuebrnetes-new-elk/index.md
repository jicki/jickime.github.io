# Elasticsearch Log-pilot Kibana


# Kubernetes ELK

> 旧 ELK = Elasticsearch + Logstash + Kibana
> 
> 新 ELK = Elasticsearch + Log-pilot + Kibana



## Log-pilot 介绍

> Log-Pilot 是阿里开源的一个智能容器日志采集工具，它不仅能够高效便捷地将容器日志采集输出到多种存储日志后端，同时还能够动态地发现和采集容器内部的日志文件。 
> 
> Github :  https://github.com/AliyunContainerService/log-pilot


* 在 kubernetes 下, Log-Pilot 可以依据环境变量 `aliyun_logs_$name = $path` 动态地生成日志采集配置文件, 其中包含两个变量:

*  1. `$name` 是我们自定义的一个字符串, 它在不同的场景下指代不同的含义, 在本场景中, 将日志采集到 ElasticSearch 的时候, 这个 `$name` 表示的是 `Index`。

*  2. `$path` 支持两种输入形式, `stdout` 和容器内部日志文件的路径, 对应日志标准输出和容器内的日志文件。
    * 第一种 约定关键字 `stdout` 表示的是采集容器的标准输出日志, 如本例中我们要采集 Nginx 容器日志, 那么我们通过配置标签 `aliyun_logs_nginx-logs=stdout` 来采集 nginx 标准输出日志。
    * 第二种 容器内部日志文件的路径, 也支持通配符的方式, 通过配置环境变量 `aliyun_logs_access=/var/log/nginx/*.log` 来采集 Nginx 容器内部的日志。当然如果你不想使用 aliyun 这个关键字, Log-Pilot 也提供了环境变量 `PILOT_LOG_PREFIX` 可以指定自己的声明式日志配置前缀, 比如 `PILOT_LOG_PREFIX: "aliyun,custom"`。


* Log-Pilot 还支持多种日志解析格式, 通过 `aliyun_logs_$name_format=<format>` 标签就可以告诉 Log-Pilot 在采集日志的时候, 同时以什么样的格式来解析日志记录, 支持的格式包括：none、json、csv、nginx、apache2 和 regxp。
* Log-Pilot 同时支持自定义 `tag` 我们可以在环境变量里配置 `aliyun_logs_$name_tags="K1=V1,K2=V2"` 那么在采集日志的时候也会将 `K1=V1` 和 `K2=V2` 采集到容器的日志输出中。自定义 `tag` 可帮助您给日志产生的环境打上 `tag` 方便进行日志统计、日志路由和日志过滤。


## 部署 ELK

* 环境介绍: 
    * CentOS 7 x64
    * Docker-ce 18.09.6
    * Kubernetes v1.14.3
    * Kubernetes 5 * Node ( 3 Master  2 Node ) 




### 部署 Elasticsearch

> 使用谷歌官方的 https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch
> 
> 


```
# 先下载 yaml 文件

wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/es-service.yaml

```

```
# 这里修改的东西有点
# 创建 一个 es-statefulset.yaml 文件 :

vi es-statefulset.yaml



# RBAC authn and authz
apiVersion: v1
kind: ServiceAccount
metadata:
  name: elasticsearch-logging
  namespace: kube-system
  labels:
    k8s-app: elasticsearch-logging
    addonmanager.kubernetes.io/mode: Reconcile
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: elasticsearch-logging
  labels:
    k8s-app: elasticsearch-logging
    addonmanager.kubernetes.io/mode: Reconcile
rules:
- apiGroups:
  - ""
  resources:
  - "services"
  - "namespaces"
  - "endpoints"
  verbs:
  - "get"
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: kube-system
  name: elasticsearch-logging
  labels:
    k8s-app: elasticsearch-logging
    addonmanager.kubernetes.io/mode: Reconcile
subjects:
- kind: ServiceAccount
  name: elasticsearch-logging
  namespace: kube-system
  apiGroup: ""
roleRef:
  kind: ClusterRole
  name: elasticsearch-logging
  apiGroup: ""
---
# Elasticsearch deployment itself
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: elasticsearch-logging
  namespace: kube-system
  labels:
    k8s-app: elasticsearch-logging
    version: v6.6.1
    addonmanager.kubernetes.io/mode: Reconcile
    kubernetes.io/cluster-service: "true"
spec:
  serviceName: elasticsearch-logging
  replicas: 3
  selector:
    matchLabels:
      k8s-app: elasticsearch-logging
      version: v6.7.2
  template:
    metadata:
      labels:
        k8s-app: elasticsearch-logging
        version: v6.7.2
    spec:
      tolerations:
      - effect: NoSchedule
        key: node-role.kubernetes.io/master
      serviceAccountName: elasticsearch-logging
      containers:
      - image: jicki/elasticsearch:v6.6.1
        name: elasticsearch-logging
        resources:
          # need more cpu upon initialization, therefore burstable class
          limits:
            cpu: 1000m
          requests:
            cpu: 100m
        ports:
        - containerPort: 9200
          name: db
          protocol: TCP
        - containerPort: 9300
          name: transport
          protocol: TCP
        securityContext:
          capabilities:
            add:
              - IPC_LOCK
              - SYS_RESOURCE
        volumeMounts:
        - name: elasticsearch-logging
          mountPath: /data
        env:
        - name: "http.host"
          value: "0.0.0.0"
        - name: "network.host"
          value: "_eth0_"
        - name: "bootstrap.memory_lock"
          value: "false"
        - name: "cluster.name"
          value: "docker-cluster"
        - name: "NAMESPACE"
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      volumes:
      - name: elasticsearch-logging
        emptyDir: {}
      # Elasticsearch requires vm.max_map_count to be at least 262144.
      # If your OS already sets up this number to a higher value, feel free
      # to remove this init container.
      initContainers:
      - image: alpine:3.6
        command: ["/sbin/sysctl", "-w", "vm.max_map_count=262144"]
        name: elasticsearch-logging-init
        securityContext:
          privileged: true

```


```
# 创建服务
[root@kubernetes-1 elk]# kubectl apply -f .
service/elasticsearch-logging created
serviceaccount/elasticsearch-logging created
clusterrole.rbac.authorization.k8s.io/elasticsearch-logging created
clusterrolebinding.rbac.authorization.k8s.io/elasticsearch-logging created
statefulset.apps/elasticsearch-logging created

```

```
# 查看服务

[root@kubernetes-1 elk]# kubectl get statefulset,svc,pods -n kube-system |grep elasticsearch

statefulset.apps/elasticsearch-logging   3/3     119s
service/elasticsearch-logging   ClusterIP   10.254.25.18    <none>        9200/TCP                 17h

pod/elasticsearch-logging-0                    1/1     Running   0          119s
pod/elasticsearch-logging-1                    1/1     Running   0          116s
pod/elasticsearch-logging-2                    1/1     Running   0          114s
```

### 部署 Log-pilot

> log-pilot 会在每个 node 里创建一个 服务，用于接收 docker 的日志。



```

# 创建一个 log-pilot.yaml 文件

vi log-pilot.yaml

apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  name: log-pilot
  namespace: kube-system
  labels:
    k8s-app: log-pilot
spec:
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        k8s-app: log-pilot
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        effect: NoSchedule
      serviceAccountName: elasticsearch-logging
      containers:
      - name: log-pilot
        image: registry.cn-hangzhou.aliyuncs.com/acs/log-pilot:0.9.7-filebeat
        env:
          - name: "LOGGING_OUTPUT"
            value: "elasticsearch"
          - name: "ELASTICSEARCH_HOSTS"
            value: "elasticsearch-logging:9200"
          - name: "NODE_NAME"
            valueFrom:
              fieldRef:
                fieldPath: spec.nodeName
        volumeMounts:
        - name: sock
          mountPath: /var/run/docker.sock
        - name: logs
          mountPath: /var/log/filebeat
        - name: state
          mountPath: /var/lib/filebeat
        - name: root
          mountPath: /host
          readOnly: true
        - name: localtime
          mountPath: /etc/localtime
        securityContext:
          capabilities:
            add:
            - SYS_ADMIN
      terminationGracePeriodSeconds: 30
      volumes:
      - name: sock
        hostPath:
          path: /var/run/docker.sock
      - name: logs
        hostPath:
          path: /var/log/filebeat
      - name: state
        hostPath:
          path: /var/lib/filebeat
      - name: root
        hostPath:
          path: /
      - name: localtime
        hostPath:
          path: /etc/localtime

```


```
# 创建 log-pilot 服务

[root@kubernetes-1 elk]# kubectl apply -f log-pilot.yaml 
daemonset.extensions/log-pilot created

```

```
# 查看服务

[root@kubernetes-1 elk]# kubectl get pods -n kube-system -o wide |grep log-pilot
log-pilot-59f22                            1/1     Running   0          30s     10.254.91.122    kubernetes-4   <none>           <none>
log-pilot-9hqxj                            1/1     Running   0          30s     10.254.105.92    kubernetes-1   <none>           <none>
log-pilot-mvjcr                            1/1     Running   0          30s     10.254.90.151    kubernetes-2   <none>           <none>
log-pilot-nml7n                            1/1     Running   0          30s     10.254.96.72     kubernetes-5   <none>           <none>
log-pilot-wr766                            1/1     Running   0          30s     10.254.101.27    kubernetes-3   <none>           <none>

```


### 部署 Kibana

> 使用谷歌官方的 https://github.com/kubernetes/kubernetes/tree/master/cluster/addons/fluentd-elasticsearch
> 



```
# 下载 yaml 文件

wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/kibana-deployment.yaml


wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/fluentd-elasticsearch/kibana-service.yaml


```

```
# 编辑 kibana-deployment.yaml

# 这里我们是使用 ingress 暴露端口 
# SERVER_BASEPATH 这个变量 注释掉
# 如果我们使用 kubectl proxy 访问，就不要注释
# 并且我们添加一个 CLUSTER_NAME

          - name: CLUSTER_NAME
            value: docker-cluster

         # - name: SERVER_BASEPATH
         #   value: /api/v1/namespaces/kube-system/services/kibana-logging/proxy

```




```
# 配置一个 ingress

vi kibana-ingress.yaml


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kibana-ingress
  namespace: kube-system
spec:
  rules:
  - host: kibana.jicki.cn
    http:
      paths:
      - backend:
          serviceName: kibana-logging
          servicePort: 5601

```





```
# 创建 Kibana 服务

[root@kubernetes-1 elk]# kubectl apply -f kibana-deployment.yaml
deployment.apps/kibana-logging created


[root@kubernetes-1 elk]# kubectl apply -f kibana-service.yaml 
service/kibana-logging created


[root@kubernetes-1 elk]# kubectl apply -f kibana-ingress.yaml 
ingress.extensions/kibana-ingress created

```


```
# 查看服务

[root@kubernetes-1 elk]# kubectl get pods,svc,ingress -n kube-system|grep kibana   
pod/kibana-logging-5bf6bbccf9-jv4d2            1/1     Running   0          4m31s

service/kibana-logging          ClusterIP   10.254.31.15    <none>        5601/TCP                 4m26s

ingress.extensions/kibana-ingress         kibana.jicki.cn                80        47s

```



### 创建一个 Nginx 服务

> 创建一个标准的 Nginx 服务, 主要是 env 添加了两种,上面我们介绍过
> 
> 一 是 基于 docker stdout 输出日志 (aliyun_logs_catalina)
> 
> 二 是 基于 程序指定输出到指定的目录中 (aliyun_logs_access)
> 
> ( elasticsearch )环境变量中的 $name 表示 Index，这里 $name 即是 catalina 和 access,  这里用于 Kibana 查询日志

```
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
      tolerations:
        - key: "node-role.kubernetes.io/master"
          effect: "NoSchedule"
      containers: 
        - name: nginx 
          image: nginx:alpine 
          imagePullPolicy: IfNotPresent
          env:
           - name: aliyun_logs_nginx-log
             value: "stdout"
          # - name: aliyun_logs_access
          #   value: "/var/log/nginx/*.log"
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
# 创建 nginx 服务

[root@kubernetes-1 nginx]# kubectl apply -f nginx-deployment.yaml 
deployment.apps/nginx-dm created
service/nginx-svc created

```

```
# 查看服务

[root@kubernetes-1 nginx]# kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
nginx-dm-64c8d4bb84-5fj62                1/1     Running   0          3s
nginx-dm-64c8d4bb84-79hwc                1/1     Running   0          3s
nginx-dm-64c8d4bb84-zhp49                1/1     Running   0          3s


```


```
# 查看 elasticsearch 里面的索引, 既 index

# 查看 elasticsearch svc 
[root@kubernetes-1 elk]# kubectl get svc -n kube-system
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
elasticsearch-logging   ClusterIP   10.254.25.18    <none>        9200/TCP                 17h



# 查看索引

[root@kubernetes-1 elk]# curl 10.254.25.18:9200/_cat/indices?v
health status index                uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   .kibana_1            Q8rg1kKsTxSuApXL2SJSqg   1   1          1            0      7.4kb          3.7kb
green  open   nginx-log-2019.07.02 TuXuOOONTL-aIZt2wq_hHQ   5   1         20            0      319kb        166.2kb

```






### 配置 Kibana 查询

> 上面已经部署 Kibana 服务, 并且通过 ingress 暴露了 http 服务.



```
# 访问 kibana 

```


# 配置 Java 日志多行匹配


## filebeat 配置 


```
# 修改镜像内的模板文件 添加 multiline


# 源码中路径 log-pilot/assets/filebeat/filebeat.tpl

# 镜像中路径 /pilot/filebeat.tpl 


{{range .configList}}
- type: log
  enabled: true
  paths:
      - {{ .HostDir }}/{{ .File }}
  multiline.pattern: '^[0-9]{4}-[0-9]{2}-[0-9]{2}'
  multiline.negate: true
  multiline.match: after
  multiline.timeout: 10s
  multiline.max_lines: 10000


```


![图1][1]

![图2][2]

![图3][3]

![图4][4]

![图5][5]

![图6][6]


  [1]: https://jicki.cn/img/posts/elk/new-1.png
  [2]: https://jicki.cn/img/posts/elk/new-2.png
  [3]: https://jicki.cn/img/posts/elk/3.png 
  [4]: https://jicki.cn/img/posts/elk/4.png 
  [5]: https://jicki.cn/img/posts/elk/5.png 
  [6]: https://jicki.cn/img/posts/elk/6.png 

