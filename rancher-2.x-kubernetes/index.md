# Rancher 2.x to kubernetes


# Rancher

> Rancher 官方文档 https://docs.rancher.cn/


## 什么是 Rancher

* Rancher是一套容器管理平台，它可以帮助组织在生产环境中轻松快捷的部署和管理容器。 Rancher可以轻松地管理各种环境的Kubernetes，满足IT需求并为DevOps团队提供支持。

* Kubernetes不仅已经成为的容器编排标准，它也正在迅速成为各类云和虚拟化厂商提供的标准基础架构。Rancher用户可以选择使用Rancher Kubernetes Engine(RKE)创建Kubernetes集群，也可以使用GKE，AKS和EKS等云Kubernetes服务。 Rancher用户还可以导入和管理现有的Kubernetes集群。

* Rancher支持各类集中式身份验证系统来管理Kubernetes集群。例如，大型企业的员工可以使用其公司Active Directory凭证访问GKE中的Kubernetes集群。IT管理员可以在用户，组，项目，集群和云中设置访问控制和安全策略。 IT管理员可以在单个页面对所有Kubernetes集群的健康状况和容量进行监控。

* Rancher为DevOps工程师提供了一个直观的用户界面来管理他们的服务容器，用户不需要深入了解Kubernetes概念就可以开始使用Rancher。 Rancher包含应用商店，支持一键式部署Helm和Compose模板。Rancher通过各种云、本地生态系统产品认证，其中包括安全工具，监控系统，容器仓库以及存储和网络驱动程序。下图说明了Rancher在IT和DevOps组织中扮演的角色。每个团队都会在他们选择的公共云或私有云上部署应用程序。

![图1][1]




## Rancher 架构

* 大多数Rancher2.0软件运行在Rancher Server节点上,Rancher Server包括用于管理整个Rancher部署的所有组件。

* 下图说明了Rancher2.0 的运行架构。该图描绘了管理两个Kubernetes集群的Rancher server安装:

  * 一个由RKE创建。
  * 一个由GKE创建。


![图2][2]





## Rancher 组件介绍


* Rancher API服务器

  * Rancher API server 建立在嵌入式Kubernetes API服务器和etcd数据库之上。

    * `Rancher API server`:  管理与外部身份验证提供程序(如Active Directory或GitHub)对应的用户身份

    * `认证授权`: Rancher API server管理访问控制和安全策略

    * `项目`: 项目是集群中的一组多个命名空间和访问控制策略的集合

    * `节点`: Rancher API server跟踪所有集群中所有节点的标识



* 集群控制 和 Agent

  * 集群控制器和集群代理实现管理Kubernetes集群所需的业务逻辑

    * 集群控制器实现Rancher安装所需的全局逻辑。&emsp; ( 1. 为集群和项目配置访问控制策略&emsp;  2. 通过调用以下方式配置集群 &emsp; 2.1 Docker machine驱动程序。&emsp; 2.2 RKE和GKE这样的Kubernetes引擎 )

    * 单独的集群代理实例实现相应集群所需的逻辑。&emsp; ( 1. 工作负载管理，例如每个集群中的pod创建和部署 &emsp;  2. 绑定并应用每个集群全局策略中定义的角色&emsp;  3. 集群与Rancher Server之间的通信:事件，统计信息，节点信息和运行状况 ）




* 认证代理

  * 该认证代理转发所有Kubernetes API调用。它集成了身份验证服务，如本地身份验证，Active Directory和GitHub。在每个Kubernetes API调用中，身份验证代理会对调用方进行身份验证，并在将调用转发给Kubernetes主服务器之前设置正确的Kubernetes模拟标头。

  * Rancher使用服务帐户与Kubernetes集群通信。




## Rancher 页面模块


* `全局层` -- 全局层主要对Rancher server 自身的基础配置, 比如Rancher Server URL、登录认证等。

  * `集群` -- 全局层的集群菜单，用于列出集群中所有的K8S集群。

  * `主机驱动` -- 用于与三方云平台API对接的中间件程序。

  * `应用商店-全局` -- 全局层的应用商店，负责应用商店的开关与添加。

  * `用户` -- 添加或者删除用户，或者修改用户的权限。

  * `系统设置` -- 全局下系统的基础配置，比如系统默认镜像仓库。

  * `安全`

    1. `角色` -- 一组权限的集合

    2. `Pod安全策略` -- Pod安全设置

    3. `登录认证` -- 用户的登录访问认证



* `集群层` -- 集群相关的配置

  1. `集群` -- 集群仪表盘，显示当前集群的资源状态

  2. `主机` -- 当前集群中添加的所有主机

  3. `存储` -- 存储卷, 持久卷。

  4. `项目与命名空间` -- 此集群拥有的项目和命名空间 ( deployment 与 namespaces )

  5. `集群成员` -- 集群用户

  6. `工具` -- 告警、通知、日志、CI/CD



* `项目层`

  * `工作负载`

    1. 工作负载服务 

    2. 负载均衡

    3. 服务发现

    4. 数据卷

    5. CI/CD

  * `应用商店` -- 项目

  * `资源`

    1. 告警

    2. 证书

    3. 配置映射

    4. 日志收集

    5. 镜像仓库

    6. 密文 (secret)

  * 命名空间

  * 项目成员


* 其他

  1. `API Keys`

  2. `主机模板`

  3. `喜好设定`



## 安装 Rancher

* 目前 Rancher 版本为 v2.3.4

### 系统需求

|系统|版本|docker型号|
|-|-|-|
|CentOS|7.5, 7.6, 7.7|Docker 17.03.2, 18.06.2, 18.09.x, 19.03.x|
|Oracle Linux|7.6|Docker 19.03.x|
|RancherOS|1.5.4|Docker 17.03.2, 18.06.2, 18.09.x (up to 18.09.8), 19.03.x|
|RHEL|7.5, 7.6, 7.7|RHEL Docker 1.13.x Docker 17.03.2, 18.06.2, 18.09.x, 19.03.x|
|Ubuntu|16.04, 18.04|Docker 17.03.2, 18.06.2, 18.09.x, 19.03.x|
|Windows Server|1809, 1903|Docker 19.03.x EE| 



### Kubernetes 版本

|系统|版本|组件|
|-|-|-|
|kubernetes|v1.17.0+|etcd: v3.4.3   flannel: v0.11.0  canal: v3.10.2  nginx-ingress-controller: 0.25.1|
|kubernetes|v1.16.3+|etcd: v3.3.15  flannel: v0.11.0  canal: v3.8.1  nginx-ingress-controller: 0.25.1|
|kubernetes|v1.15.6+|etcd: v3.3.10  flannel: v0.11.0  canal: v3.7.4  nginx-ingress-controller: 0.25.1|




### 单节点 Rancher

* Rancher 使用 docker 启动运行

```
# 启动
docker run -d  \
  --name=rancher \
  --restart=unless-stopped \
  -v /opt/rancher/auditlog:/var/log/auditlog \
  -e AUDIT_LEVEL=3 \
  -e AUDIT_LOG_PATH=/var/log/auditlog/rancher-api-audit.log \
  -e AUDIT_LOG_MAXAGE=20 \
  -e AUDIT_LOG_MAXBACKUP=20 \
  -e AUDIT_LOG_MAXSIZE=100 \
  -p 80:80 -p 443:443 \
  rancher/rancher

```




### 多节点 Rancher

* 多节点是基于 Kubernetes 集群部署 Rancher

* 由于我只有一台服务器所以k8s 只有一个节点,既是 Master 又是 node

* `kubectl taint node localhost node-role.kubernetes.io/master-` master 调度 pods

```
[root@localhost rancher]# kubectl get nodes
NAME        STATUS   ROLES    AGE   VERSION
localhost   Ready    master   17h   v1.17.4
```



#### 部署 Helm v3

* 官方推荐使用 Helm 来安装 Rancher 所以这里先安装一个 Helm v3

* 下载二进制文件 https://github.com/helm/helm/releases

```
mkdir /opt/helm

cd /opt/helm

wget https://get.helm.sh/helm-v3.1.2-linux-amd64.tar.gz

tar zxvf helm-v3.1.2-linux-amd64.tar.gz
linux-amd64/
linux-amd64/helm
linux-amd64/README.md
linux-amd64/LICENSE


mv linux-amd64/ bin

```


```
# 测试

[root@localhost ~]# /opt/helm/bin/helm version
version.BuildInfo{Version:"v3.1.2", GitCommit:"d878d4d45863e42fd5cff6743294a11d28a9abce", GitTreeState:"clean", GoVersion:"go1.13.8"}

```


* 添加`Chart` 仓库地址

```
# 添加 rancher 的源
/opt/helm/bin/helm repo add rancher-stable \
  https://releases.rancher.com/server-charts/stable


# 输出如下:
"rancher-stable" has been added to your repositories

```


#### 部署 cert-manager

* 部署 cert-manager 用于 rancher 的ssl 证书

```
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v0.14.0/cert-manager.yaml

```


* 验证`cert-manager`安装

```
[root@localhost rancher]# kubectl get pods -n cert-manager
NAME                                       READY   STATUS    RESTARTS   AGE
cert-manager-75f6cdcb64-ng7xp              1/1     Running   0          7m47s
cert-manager-cainjector-79788689f9-6gq8m   1/1     Running   0          7m47s
cert-manager-webhook-5b6c798c9-r6bkz       1/1     Running   0          7m47s

```


#### 导出 rancher YAML 文件

* 安装 rancher

  * 渲染 yaml 文件

    * 这里由于我不太使用 `helm` 所以我现在重新渲染成完整的 yaml 。


##### 使用 cert-manager


```
# 创建 namespaces , 这里 cattle-system 会存放后续 rancher 的 agent 等项目

kubectl create namespace cattle-system




# 安装 rancher 使用 cert-manager 做为证书

/opt/helm/bin/helm install rancher rancher-stable/rancher \
    --namespace cattle-system \
    --set hostname=rancher.jicki.cn \
    --debug --dry-run

```


*  clusterRoleBinding.yaml 文件

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rancher
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
subjects:
- kind: ServiceAccount
  name: rancher
  namespace: cattle-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```


* serviceAccount.yaml 文件

```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: rancher
  namespace: cattle-system
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
```


* deployment.yaml 文件

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: rancher
  namespace: cattle-system
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rancher
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: rancher
        release: rancher
    spec:
      serviceAccountName: rancher
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - rancher
              topologyKey: kubernetes.io/hostname
      containers:
      - image: rancher/rancher:v2.3.5
        imagePullPolicy: IfNotPresent
        name: rancher
        ports:
        - containerPort: 80
          protocol: TCP
        args:
        # Public trusted CA - clear ca certs
        - "--http-listen-port=80"
        - "--https-listen-port=443"
        - "--add-local=auto"
        env:
        - name: CATTLE_NAMESPACE
          value: cattle-system
        - name: CATTLE_PEER_SERVICE
          value: rancher
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 30
        resources:
          {}
        volumeMounts:
      volumes:
```


* service.yaml 文件


```
apiVersion: v1
kind: Service
metadata:
  name: rancher
  namespace: cattle-system
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: rancher
```


* ingress.yaml 文件

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: rancher
  namespace: cattle-system
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
  annotations:
    cert-manager.io/issuer: rancher
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"
spec:
  rules:
  - host: rancher.jicki.cn  # hostname to access rancher server
    http:
      paths:
      - backend:
          serviceName: rancher
          servicePort: 80
  tls:
  - hosts:
    - rancher.jicki.cn
    secretName: tls-rancher-ingress
```

* issuer-rancher.yaml 文件

```
apiVersion: cert-manager.io/v1alpha2
kind: Issuer
metadata:
  name: rancher
  namespace: cattle-system
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
spec:
  ca:
    secretName: tls-rancher
```


* 导入文件

```
[root@localhost rancher]# kubectl apply -f .

clusterrolebinding.rbac.authorization.k8s.io/rancher created
deployment.apps/rancher created
ingress.extensions/rancher created
issuer.cert-manager.io/rancher created
service/rancher created
serviceaccount/rancher created
```

* 查看服务


```
[root@localhost rancher]# kubectl get pods,svc,ingress,deployment -n cattle-system
NAME                           READY   STATUS    RESTARTS   AGE
pod/rancher-7689c58786-7gvsk   1/1     Running   1          44m
pod/rancher-7689c58786-fllcn   1/1     Running   2          44m
pod/rancher-7689c58786-vgk75   1/1     Running   2          44m

NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
service/rancher   ClusterIP   10.254.54.9   <none>        80/TCP    44m

NAME                         HOSTS              ADDRESS   PORTS   AGE
ingress.extensions/rancher   rancher.jicki.cn             80      44m

NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rancher   3/3     3            3           44m

```



##### 使用 自签 证书

```
# helm v3
# 创建 namespaces , 这里 cattle-system 会存放后续 rancher 的 agent 等项目

kubectl create namespace cattle-system
```


```

# 安装 rancher 使用 自签证书

/opt/helm/bin/helm install rancher rancher-stable/rancher \
    --namespace cattle-system \
    --set hostname=rancher.jicki.cn \
    --set ingress.tls.source=secret \
    --debug --dry-run

```


* clusterRoleBinding.yaml 文件

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: rancher
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
subjects:
- kind: ServiceAccount
  name: rancher
  namespace: cattle-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```


* serviceAccount.yaml 文件

```
kind: ServiceAccount
apiVersion: v1
metadata:
  name: rancher
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
```


* deployment.yaml 文件

```
kind: Deployment
apiVersion: apps/v1
metadata:
  name: rancher
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rancher
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: rancher
        release: rancher
    spec:
      serviceAccountName: rancher
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - rancher
              topologyKey: kubernetes.io/hostname
      containers:
      - image: rancher/rancher:v2.3.5
        imagePullPolicy: IfNotPresent
        name: rancher
        ports:
        - containerPort: 80
          protocol: TCP
        args:
        # Public trusted CA - clear ca certs
        - "--no-cacerts"
        - "--http-listen-port=80"
        - "--https-listen-port=443"
        - "--add-local=auto"
        env:
        - name: CATTLE_NAMESPACE
          value: cattle-system
        - name: CATTLE_PEER_SERVICE
          value: rancher
        livenessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /healthz
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 30
        resources:
          {}
        volumeMounts:
      volumes:
```

* service.yaml 文件

```
apiVersion: v1
kind: Service
metadata:
  name: rancher
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
spec:
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
  selector:
    app: rancher
```

* ingress.yaml 文件

```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: rancher
  labels:
    app: rancher
    chart: rancher-2.3.5
    heritage: Helm
    release: rancher
  annotations:
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "1800"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "1800"
spec:
  rules:
  - host: rancher.jicki.cn  # hostname to access rancher server
    http:
      paths:
      - backend:
          serviceName: rancher
          servicePort: 80
  tls:
  - hosts:
    - rancher.jicki.cn
    secretName: tls-rancher-ingress

```

* 导入文件

```
[root@k8s-node-1 rancher]# kubectl apply -f . -n cattle-system
clusterrolebinding.rbac.authorization.k8s.io/rancher unchanged
deployment.apps/rancher created
ingress.extensions/rancher created
service/rancher created
serviceaccount/rancher created

```

* 配置证书

  * 关于 阿里云 ssl 证书 这里下载 Nginx 的证书压缩包
    * `_.key`  文件是证书的私钥文件。
    * `_.pem` 文件是证书文件，一般包含两段内容。(public.crt + chain.crt)

* 导入证书到 k8s

```
kubectl -n cattle-system create \
  secret tls tls-rancher-ingress \
  --cert=3258931_rancher.jicki.cn.pem \
  --key=3258931_rancher.jicki.cn.key
```




#### 配置负载均衡入口 (可选)

> 因为我这里只有一台服务器使用的是 cert-manager 自签的证书,所以直接走的 ingress
>
> 线上购买的 ssl 签名, 需自行导入证书以及配置 nginx 或 slb 的证书。

* 可使用 nginx 或者 slb 等


* 配置 Nginx 作为代理

  * 创建 nginx 配置文件

{% raw %}
```
# 创建配置目录
mkdir -p /etc/nginx

# 写入代理配置
cat << EOF >> /etc/nginx/nginx.conf
worker_processes 2;
worker_rlimit_nofile 40000;

events {
    worker_connections 8192;
}

http {
    # Gzip Settings
    gzip on;
    gzip_disable "msie6";
    gzip_disable "MSIE [1-6]\.(?!.*SV1)";
    gzip_vary on;
    gzip_static on;
    gzip_proxied any;
    gzip_min_length 0;
    gzip_comp_level 8;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_types
      text/xml application/xml application/atom+xml application/rss+xml application/xhtml+xml image/svg+xml application/font-woff
      text/javascript application/javascript application/x-javascript
      text/x-json application/json application/x-web-app-manifest+json
      text/css text/plain text/x-component
      font/opentype application/x-font-ttf application/vnd.ms-fontobject font/woff2
      image/x-icon image/png image/jpeg;

    upstream rancher {
        server IP_NODE_1:80;
        server IP_NODE_2:80;
        server IP_NODE_3:80;
    }

    map $http_upgrade $connection_upgrade {
        default Upgrade;
        ''      close;
    }

    server {
        listen 443 ssl http2; # 如果是升级或者全新安装v2.2.2,需要禁止http2，其他版本不需修改。
        server_name rancher.jicki.cn;
        ssl_certificate tls.crt;
        ssl_certificate_key tls.key;

        location / {
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Forwarded-Port $server_port;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://rancher;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            # This allows the ability for the execute shell window to remain open for up to 15 minutes. 
            ## Without this parameter, the default is 1 minute and will automatically close.
            proxy_read_timeout 900s;
            proxy_buffering off;
        }
    }

    server {
        listen 80;
        server_name rancher.jicki.cn;
        return 301 https://$server_name$request_uri;
    }
}

EOF
```

* 授权配置文件

```
chmod +r /etc/nginx/nginx.conf
```


* 创建系统 systemd.service 文件

```
cat << EOF >> /etc/systemd/system/nginx-proxy.service
[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 127.0.0.1:80:80 \\
                              -p 127.0.0.1:443:443 \\
                              -v /etc/nginx:/etc/nginx \\
                              -v /etc/nginx/ssl:/etc/nginx/ssl \\
                              --name nginx-proxy \\
                              --net=host \\
                              --restart=on-failure:5 \\
                              --memory=512M \\
                              nginx:alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
EOF
```
{% endraw %}

* 启动nginx

```
systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy
systemctl status nginx-proxy
```

### 访问 Rancher 

* 如上配置, 已经添加 ingress 并且配置了域名为 rancher.jicki.cn


* 通过 浏览器访问 https://rancher.jicki.cn

![图3][3]


![图4][4]


![图5][5]


* 添加一个集群

![图6][6]


![图7][7]


![图8][8]


![图9][9]


![图10][10]


![图11][11]


![图12][12]


![图13][13]


![图14][14]




* k8s 集群服务的配置 

  * 我这里只有一台服务器, 所以都是基于rancher部署的那个导入的k8s集群。



![图15][15]


![图16][16]


![图17][17]


![图18][18]


![图19][19]









### 清除 rancher 




```
# 删除 yaml 文件
kubectl delete -f .


```




```
# 删除命名空间

# cattle-system
kubectl patch namespace cattle-system -p '{"metadata":{"finalizers":[]}}' --type='merge' -n cattle-system
kubectl delete namespace cattle-system --grace-period=0 --force


# cattle-global-data
kubectl patch namespace cattle-global-data -p '{"metadata":{"finalizers":[]}}' --type='merge' -n cattle-system
kubectl delete namespace cattle-global-data --grace-period=0 --force



# cattle-global-nt
kubectl patch namespace cattle-global-nt -p '{"metadata":{"finalizers":[]}}' --type='merge' -n cattle-system
kubectl delete namespace cattle-global-nt --grace-period=0 --force


#  local
kubectl patch namespace local -p '{"metadata":{"finalizers":[]}}' --type='merge' -n cattle-system

for resource in `kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get -o name -n local`; do kubectl patch $resource -p '{"metadata": {"finalizers": []}}' --type='merge' -n local; done

kubectl delete namespace local --grace-period=0 --force


# 其他相关 namespace, 具体名字根据自己环境 get namespace 查询
p-nw6f6           Active   46m
p-s8vtd           Active   46m
user-6jntr        Active   43m


kubectl delete namespace p-nw6f6 --grace-period=0 --force
kubectl delete namespace p-s8vtd --grace-period=0 --force
kubectl delete namespace user-6jntr --grace-period=0 --force

kubectl patch namespace p-nw6f6  -p '{"metadata":{"finalizers":[]}}' --type='merge' -n cattle-system
kubectl patch namespace p-s8vtd  -p '{"metadata":{"finalizers":[]}}' --type='merge' -n cattle-system
kubectl patch namespace user-6jntr  -p '{"metadata":{"finalizers":[]}}' --type='merge' -n cattle-system

```





  [1]: https://jicki.cn/img/posts/rancher/rancher.png
  [2]: https://jicki.cn/img/posts/rancher/rancher-1.png
  [3]: https://jicki.cn/img/posts/rancher/webui-1.png
  [4]: https://jicki.cn/img/posts/rancher/webui-2.png
  [5]: https://jicki.cn/img/posts/rancher/webui-3.png
  [6]: https://jicki.cn/img/posts/rancher/webui-4.png
  [7]: https://jicki.cn/img/posts/rancher/webui-5.png
  [8]: https://jicki.cn/img/posts/rancher/webui-6.png
  [9]: https://jicki.cn/img/posts/rancher/webui-7.png
  [10]: https://jicki.cn/img/posts/rancher/webui-8.png
  [11]: https://jicki.cn/img/posts/rancher/webui-9.png
  [12]: https://jicki.cn/img/posts/rancher/webui-10.png
  [13]: https://jicki.cn/img/posts/rancher/webui-11.png
  [14]: https://jicki.cn/img/posts/rancher/webui-12.png
  [15]: https://jicki.cn/img/posts/rancher/webui-13.png
  [16]: https://jicki.cn/img/posts/rancher/webui-14.png
  [17]: https://jicki.cn/img/posts/rancher/webui-15.png
  [18]: https://jicki.cn/img/posts/rancher/webui-16.png
  [19]: https://jicki.cn/img/posts/rancher/webui-17.png


