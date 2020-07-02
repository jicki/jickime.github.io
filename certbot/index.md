# Let's Encrypt 证书 申请



> Let's Encrypt 是一个免费、开放，自动化的证书颁发机构，由 ISRG（Internet Security Research Group）运作。
>
ISRG 是一个关注网络安全的公益组织，其赞助商从非商业组织到财富100强公司都有，包括 Mozilla、Akamai、Cisco、Facebook，密歇根大学等等。
>
ISRG 以消除资金，技术领域的障碍，全面推进加密连接成为互联网标配为自己的使命。
>
Let's Encrypt 项目于2012年由 Mozilla 的两个员工发起，2014年11年对外宣布公开，2015年12月3日开启公测。



# Certbot

> Certboot 是官方提供的一个 申请 Let's Encrypt 的工具。
>
> 官方文档 https://certbot.eff.org/docs/


## 安装 Certbot


```
# 下载 二进制文件

wget https://dl.eff.org/certbot-auto

chmod a+x ./certbot-auto

./certbot-auto --help

```


```
# 初始化环境

./certbot-auto -n 

会初始化生成环境，会创建 virtualenv env

# 注: 如果系统存在两个版本 virtualenv 会出现问题

# cerbot 会使用 yum  与 pip 下载 virtualenv

# 请使用 pip install virtualenv

```



## 生成证书

> 生成证书的方式有多种，webroot, nginx, apache, standalone, DNS 的方式


### standalone 模式

```
# 独立模式 --standalone

./certbot-auto certonly --standalone --email jicki@qq.com --agree-tos -d jicki.cn -d www.jicki.cn

# 独立模式需要 占用本机的 80 以及 443 端口 用来认证 证书，

# 所以需要先关闭 本机 服务

```


### webroot 模式

```
# 网站目录 --webroot 模式

# 不同域名需要配置再不同的 --webroot 目录下

./certbot-auto certonly --agree-tos --email jicki@qq.com --webroot -w /var/www/html/ -d jicki.cn -d www.jicki.cn -w /var/www/wiki -d wiki.jicki.cn


# --webroot 模式 不需要关闭正在运行的服务, 但是会在 网站文件目录下 创建一个 .well-known 目录
对于这个目录需要配置外部禁止访问。
# 这里面注意，配置反向代理的https这模式不适用。 


# nginx 配置 在相关域名下配置


    location ~ \.well-known{
        allow all;
    }
    
    location ^~ /.well-known/acme-challenge/ {
        alias         /var/www/html/;
        try_files     $uri =404;
    }

```


### nginx apache 模式

```
./certbot-auto --nginx

./certbot-auto --apache

# nginx 与 apache 两种模式 会自动修改 nginx 与 apache 配置文件，
# 所以对 nginx 与 apache 的安装有要求，配置文件必须在固定位置。

```



### Dns 模式

> dns 模式支持 范域名 的证书

```
./certbot-auto --server https://acme-v02.api.letsencrypt.org/directory -d "*.jicki.cn" --manual --preferred-challenges dns-01 certonly



# 这里执行命令后~需要 交互输入 一些 配置 如下:


Saving debug log to /var/log/letsencrypt/letsencrypt.log
Plugins selected: Authenticator manual, Installer None
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for jicki.cn

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: y     

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Please deploy a DNS TXT record under the name
_acme-challenge.jicki.cn with the following value:

tdoCC636Cel1wQPY-LB-FURPvNSloFhBdWyEoqkQZNU

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
Waiting for verification...





## 这里面提示在 dns 里配置一下 认证

Please deploy a DNS TXT record under the name
_acme-challenge.jicki.cn with the following value:

tdoCC636Cel1wQPY-LB-FURPvNSloFhBdWyEoqkQZNU

Before continuing, verify the record is deployed.



主机记录: _acme-challenge.jicki.cn
记录类型: TXT
记录值: tdoCC636Cel1wQPY-LB-FURPvNSloFhBdWyEoqkQZNU



# 配置完以后~~等待认证
```



## 配置 https


```
#  上面生成好证书以后会将证书生成在 /etc/letsencrypt 目录下

```


```
# Nginx 配置 https


{
  listen     80;
  listen     443 ssl;
  listen [::]:443 ssl ipv6only=on;
  server_name jicki.cn www.jicki.cn;
  root /var/www/html;
  index index.html index.htm index.php;
  access_log /var/logs/nginx/jicki.log main;
  
  # ssl setting
  ssl_protocols TLSv1.2 TLSv1.1 TLSv1;
  ssl_certificate /etc/letsencrypt/live/www.jicki.cn/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/www.jicki.cn/privkey.pem;
  ssl_trusted_certificate /etc/letsencrypt/live/www.jicki.cn/chain.pem;
  ssl_session_cache    shared:SSL:1m;
  ssl_session_timeout  5m;
  server_tokens off;
  ssl_prefer_server_ciphers on;
  fastcgi_param   HTTPS               on;
  fastcgi_param   HTTP_SCHEME         https;

  # 强制跳转到 https
  if ($scheme = http) {
       return 301 https://$server_name$request_uri;
     }

  # 禁止 webroot 模式目录
  location ~ \.well-known{
        allow all;
    }
  # 禁止 webroot 模式目录
  location ^~ /.well-known/acme-challenge/ {
        alias         /var/www/html/;
        try_files     $uri =404;
    }

```


## 配置续签证书


```
crontab -e

添加如下: 每周1检测一次

30 2 * * 1   /opt/certbot/certbot-auto renew  >> /var/log/certbot-renew.log


```


# docker 方式


> docker 方式只做简单的介绍，需要懂docker的人使用，不懂docker 建议使用上面方式



```
# 这里放一个 脚本


#!/bin/bash

case $1 in

"make")

        docker stop nginx

        docker run --rm -p 80:80 -p 443:443 \
        -v /opt/data/nginx/ssl/:/etc/letsencrypt \
        certbot/certbot certonly \
        --standalone -m jicki@qq.com --agree-tos \
        -d www.jicki.cn -d jicki.cn

        docker start nginx

        ;;
"renew")

        docker stop nginx

        docker run --rm -p 80:80 -p 443:443 \
        -v /opt/nginx/ssl/:/etc/letsencrypt \
        -v /var/log/letsencrypt:/var/log/letsencrypt \
        certbot/certbot renew \
        --standalone

        docker start nginx
        ;;
*)
        echo "Please choose make/renew"
        ;;
esac
```

> acem.sh 也是一个签发工具，这个对于 泛域名配置 会比较简单，可以自动添加到dns记录里

```
第一次成功之后，acme.sh会记录下App_Key跟App_Secret，并且生成一个定时任务，每天凌晨0：00自动检测过期域名并且自动续期。


# 泛域名 最好使用 acem.sh 这个容器来配置
# 这里面 App_Key 与 App_Secret 是dns商里面的一个 api 
# acme.sh 支持很多个 dns 商



# 如下是 aliyun 的配置
docker run --rm  -it  \
  -v /opt/nginx/ssl:/acme.sh  \
  -e Ali_Key="xxxxxx" \
  -e Ali_Secret="xxxx" \
  neilpang/acme.sh  --issue --dns dns_ali -d jicki.cn -d *.jicki.cn



#  DNSPod 配置如下:
docker run --rm  -it  \
  -v /opt/nginx/ssl:/acme.sh  \
  -e DP_Id="xxxxxx" \
  -e DP_Key="xxxx" \
  neilpang/acme.sh  --issue --dns dns_dp -d jicki.cn -d *.jicki.cn



# GoDaddy 配置如下:
docker run --rm  -it  \
  -v /opt/nginx/ssl:/acme.sh  \
  -e GD_Key="xxxxxx" \
  -e GD_Secret="xxxx" \
  neilpang/acme.sh  --issue --dns dns_gd -d jicki.cn -d *.jicki.cn


```

# Kubernetes 方式

> Kubernetes 通过 Cert manager 进行自动申请 Let's Encrypt 。
>
> github 地址 https://github.com/jetstack/cert-manager


## 部署 cert-manager

> 这里官方使用 helm 来直接部署


### 安装 helm 

> Helm 是 Kubernetes 的包管理器，可以帮我们简化 kubernetes 的操作，一键部署应用。
>
> helm 部署 请参考 https://jicki.cn/kubernetes/docker/2018/12/07/helm/



### 安装 cert-manager

```
# 执行 helm install

helm install \
--name cert-manager \
--namespace kube-system \
--set ingressShim.defaultIssuerName=letsencrypt-prod \
--set ingressShim.defaultIssuerKind=ClusterIssuer \
--set image.repository=jicki/cert-manager-controller \
--set ingressShim.image.repository=jicki/cert-manager-ingress-shim \
stable/cert-manager


# 输出如下信息:

NAME:   cert-manager
LAST DEPLOYED: Fri Dec  7 14:46:20 2018
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1beta1/Deployment
NAME                       AGE
cert-manager-cert-manager  0s

==> v1/Pod(related)

NAME                                        READY  STATUS             RESTARTS  AGE
cert-manager-cert-manager-6b58f97c65-dl2j9  0/2    ContainerCreating  0         0s

==> v1/ServiceAccount

NAME                       AGE
cert-manager-cert-manager  0s

==> v1beta1/CustomResourceDefinition
certificates.certmanager.k8s.io    0s
clusterissuers.certmanager.k8s.io  0s
issuers.certmanager.k8s.io         0s

==> v1beta1/ClusterRole
cert-manager-cert-manager  0s

==> v1beta1/ClusterRoleBinding
cert-manager-cert-manager  0s


NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://github.com/jetstack/cert-manager/tree/v0.2.3/docs/api-types/issuer

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://github.com/jetstack/cert-manager/blob/v0.2.3/docs/user-guides/ingress-shim.md
```


```
# 这里 cert-manager 的 image 是国外地址

quay.io/jetstack/cert-manager-controller:v0.2.3
quay.io/jetstack/cert-manager-ingress-shim:v0.2.3


# 替换为官方最新版本，否则有可能报协议不匹配

jicki/cert-manager-controller:canary
jicki/cert-manager-ingress-shim:canary

```


```
# 查询服务可用的 values 修改 image 需要用到

helm inspect values stable/cert-manager



# 修改 image

[root@kubernetes-1 ~]# helm upgrade cert-manager --set image.repository=jicki/cert-manager-controller --set image.tag=canary --set ingressShim.image.repository=jicki/cert-manager-ingress-shim --set ingressShim.image.tag=canary stable/cert-manager




helm upgrade cert-manager --set image.repository=jicki/cert-manager-controller --set image.tag=canary --set ingressShim.image.repository=jicki/cert-manager-ingress-shim --set ingressShim.image.tag=canary stable/cert-manager
Release "cert-manager" has been upgraded. Happy Helming!
LAST DEPLOYED: Fri Dec  7 12:14:05 2018
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1/Pod(related)
NAME                                        READY  STATUS             RESTARTS  AGE
cert-manager-cert-manager-6b58f97c65-rk68q  2/2    Running            0         50m
cert-manager-cert-manager-766fb987fc-l5b7f  0/2    ContainerCreating  0         0s

==> v1/ServiceAccount

NAME                       AGE
cert-manager-cert-manager  59m

==> v1beta1/CustomResourceDefinition
certificates.certmanager.k8s.io    59m
clusterissuers.certmanager.k8s.io  59m
issuers.certmanager.k8s.io         59m

==> v1beta1/ClusterRole
cert-manager-cert-manager  59m

==> v1beta1/ClusterRoleBinding
cert-manager-cert-manager  59m

==> v1beta1/Deployment
cert-manager-cert-manager  59m


NOTES:
cert-manager has been deployed successfully!

In order to begin issuing certificates, you will need to set up a ClusterIssuer
or Issuer resource (for example, by creating a 'letsencrypt-staging' issuer).

More information on the different types of issuers and how to configure them
can be found in our documentation:

https://github.com/jetstack/cert-manager/tree/v0.2.3/docs/api-types/issuer

For information on how to configure cert-manager to automatically provision
Certificates for Ingress resources, take a look at the `ingress-shim`
documentation:

https://github.com/jetstack/cert-manager/blob/v0.2.3/docs/user-guides/ingress-shim.md

```



## 验证服务

```
[root@kubernetes-1 ~]# kubectl get pods -n kube-system --selector=app=cert-manager
NAME                                         READY   STATUS    RESTARTS   AGE
cert-manager-cert-manager-766fb987fc-l5b7f   2/2     Running   0          45s



# 查看具体信息

[root@kubernetes-1 ~]# kubectl describe pods/cert-manager-cert-manager-766fb987fc-l5b7f -n kube-system


Name:           cert-manager-cert-manager-766fb987fc-l5b7f
Namespace:      kube-system
Node:           kubernetes-2/192.168.0.248
Start Time:     Fri, 07 Dec 2018 12:14:06 +0800
Labels:         app=cert-manager
                pod-template-hash=766fb987fc
                release=cert-manager
Annotations:    <none>
Status:         Running
IP:             10.254.90.167
Controlled By:  ReplicaSet/cert-manager-cert-manager-766fb987fc
Containers:
  cert-manager:
    Container ID:   docker://7f6a6ed2257567c1a92dfe2ef583ddf275cc51bd8e5454ca694079f551aa6b17
    Image:          jicki/cert-manager-controller:canary
    Image ID:       docker-pullable://jicki/cert-manager-controller@sha256:e894e0965c974e526c489fc69e8536d55893610085c46f9ff59f6c75480f521d
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 07 Dec 2018 12:14:18 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from cert-manager-cert-manager-token-cpwvg (ro)
  ingress-shim:
    Container ID:   docker://21e37d4317c9083624b8fbe078433d53135d9f0715769110c362aeef69b2f9ed
    Image:          jicki/cert-manager-ingress-shim:canary
    Image ID:       docker-pullable://jicki/cert-manager-ingress-shim@sha256:d798681aae440fadde653559605f9d2d1c006da83caf6e86aced79faf3de2aa7
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 07 Dec 2018 12:14:32 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from cert-manager-cert-manager-token-cpwvg (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  cert-manager-cert-manager-token-cpwvg:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  cert-manager-cert-manager-token-cpwvg
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     <none>
Events:
  Type    Reason     Age   From                   Message
  ----    ------     ----  ----                   -------
  Normal  Scheduled  57s   default-scheduler      Successfully assigned kube-system/cert-manager-cert-manager-766fb987fc-l5b7f to kubernetes-2
  Normal  Pulling    57s   kubelet, kubernetes-2  pulling image "jicki/cert-manager-controller:canary"
  Normal  Pulled     45s   kubelet, kubernetes-2  Successfully pulled image "jicki/cert-manager-controller:canary"
  Normal  Created    45s   kubelet, kubernetes-2  Created container
  Normal  Started    45s   kubelet, kubernetes-2  Started container
  Normal  Pulling    45s   kubelet, kubernetes-2  pulling image "jicki/cert-manager-ingress-shim:canary"
  Normal  Pulled     31s   kubelet, kubernetes-2  Successfully pulled image "jicki/cert-manager-ingress-shim:canary"
  Normal  Created    31s   kubelet, kubernetes-2  Created container
  Normal  Started    31s   kubelet, kubernetes-2  Started container



# 查看生成的 crd

[root@kubernetes-1 ~]# kubectl get crd
NAME                                CREATED AT
certificates.certmanager.k8s.io     2018-12-07T03:14:36Z
clusterissuers.certmanager.k8s.io   2018-12-07T03:14:36Z
issuers.certmanager.k8s.io          2018-12-07T03:14:36Z

```



## 创建签发证书服务

> 创建一个基于上面 crd 中  certificates.certmanager.k8s.io 的 api 的服务

```
# 这里请特别注意 server: 这个地址, 官方版本是 0.2.3 使用v01的api 不要用 v02的~否则报错

vi letsencrypt-clusterissuer.yaml


apiVersion: certmanager.k8s.io/v1alpha1   
kind: ClusterIssuer   
metadata:   
  name: letsencrypt-prod   
  namespace: kube-system   
spec:   
  acme: 
    srver: https://acme-v01.api.letsencrypt.org/directory
    email: jicki@qq.com
    privateKeySecretRef:   
      name: letsencrypt-prod   
    http01: {}   


```



```
# 创建服务

[root@kubernetes-1 ~]# kubectl apply -f letsencrypt-clusterissuer.yaml 
clusterissuer.certmanager.k8s.io/letsencrypt-prod created
clusterissuer.certmanager.k8s.io/letsencrypt-staging created



# 查看

[root@kubernetes-1 ~]# kubectl get clusterissuer
NAME                  AGE
letsencrypt-prod      10s
letsencrypt-staging   10s

```


## 创建基于 ingress 的 https


```
# 查看 svc


[root@kubernetes-1 ~]# kubectl get svc -n kube-system
NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)         AGE
kubernetes-dashboard   ClusterIP   10.254.53.66   <none>        443/TCP         56d

```


```
# 编辑一个 ingress

[root@kubernetes-1 ~]# vi dashboard-ingress.yaml 


apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: kubernetes-dashboard
  namespace: kube-system
  annotations:
    kubernetes.io/ingress.class: "nginx"
    certmanager.k8s.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - dashboard.jicki.cn
    secretName: dashboard-tls
  rules:
  - host: dashboard.jicki.cn
    http:
      paths:
      - path: /
        backend:
          serviceName: kubernetes-dashboard
          servicePort: 443


```



```
# 查看 ingress 

[root@kubernetes-1 ~]# kubectl get ingress -n kube-system
NAME                        HOSTS                ADDRESS   PORTS     AGE
kubernetes-dashboard        dashboard.jicki.cn             80, 443   11s


# 查看 pods

[root@kubernetes-1 ~]# kubectl get pods -n kube-system
NAME                        HOSTS                ADDRESS   PORTS     AGE
cm-acme-http-solver-prcdn   dashboard.jicki.cn             80        8s


# 这个 cm-acme 是用来创建认证 证书的, 认证通过以后~会自动删除

```


```
# 查看具体信息

[root@kubernetes-1 ~]# kubectl describe ingress/kubernetes-dashboard -n kube-system
Name:             kubernetes-dashboard
Namespace:        kube-system
Address:          
Default backend:  default-http-backend:80 (<none>)
TLS:
  dashboard-tls terminates dashboard.jicki.cn
Rules:
  Host                Path  Backends
  ----                ----  --------
  dashboard.jicki.cn  
                      /   kubernetes-dashboard:443 (10.254.101.26:8443)
Annotations:
  certmanager.k8s.io/cluster-issuer:                 letsencrypt-prod
  ingress.kubernetes.io/ssl-passthrough:             true
  kubectl.kubernetes.io/last-applied-configuration:  {"apiVersion":"extensions/v1beta1","kind":"Ingress","metadata":{"annotations":{"certmanager.k8s.io/cluster-issuer":"letsencrypt-prod","ingress.kubernetes.io/ssl-passthrough":"true","kubernetes.io/ingress.class":"nginx","nginx.ingress.kubernetes.io/secure-backends":"true"},"name":"kubernetes-dashboard","namespace":"kube-system"},"spec":{"rules":[{"host":"dashboard.jicki.cn","http":{"paths":[{"backend":{"serviceName":"kubernetes-dashboard","servicePort":443},"path":"/"}]}}],"tls":[{"hosts":["dashboard.jicki.cn"],"secretName":"dashboard-tls"}]}}

  kubernetes.io/ingress.class:                  nginx
  nginx.ingress.kubernetes.io/secure-backends:  true
Events:
  Type    Reason             Age   From                      Message
  ----    ------             ----  ----                      -------
  Normal  CREATE             76s   nginx-ingress-controller  Ingress kube-system/kubernetes-dashboard
  Normal  CreateCertificate  76s   cert-manager              Successfully created Certificate "dashboard-tls"
  Normal  CREATE             62s   nginx-ingress-controller  Ingress kube-system/kubernetes-dashboard

```


```
# 查看生成证书

[root@kubernetes-1 ~]# kubectl get certificate -n kube-system
NAME            AGE
dashboard-tls   3m



[root@kubernetes-1 ~]# kubectl get secret -n kube-system
NAME                                  TYPE                                  DATA   AGE
dashboard-tls                         kubernetes.io/tls                     3      4m38s

```


```
# 查看具体日志 （所以需要域名可以正常使用）


[root@kubernetes-1 ~]# kubectl logs pods/cert-manager-7859bc8fd7-lhhgb -n cert-manager


I1206 07:05:06.579320       1 controller.go:176] ingress-shim controller: Finished processing work item "kube-system/kubernetes-dashboard"
I1206 07:05:06.579763       1 controller.go:148] certificates controller: Finished processing work item "kube-system/dashboard-tls"
I1206 07:05:06.579794       1 controller.go:142] certificates controller: syncing item 'kube-system/dashboard-tls'
I1206 07:05:06.585618       1 controller.go:181] orders controller: syncing item 'kube-system/dashboard-tls-3718435272'
I1206 07:05:06.585697       1 controller.go:148] certificates controller: Finished processing work item "kube-system/dashboard-tls"
I1206 07:05:06.585718       1 controller.go:142] certificates controller: syncing item 'kube-system/dashboard-tls'




# 如果域名未配置，会报错 (因为申请证书需要认证 域名下的 .well-known/acme-challenge 目录)

I1206 07:58:57.570703       1 http.go:110] could not reach 'http://dashboard.jicki.cn/.well-known/acme-challenge/0beMNTSzGirQygofZ2kyiexLjqPDSV3-XGUGpokFSNM': failed to GET 'http://dashboard.jicki.cn/.well-known/acme-challenge/0beMNTSzGirQygofZ2kyiexLjqPDSV3-XGUGpokFSNM': Get http://dashboard.jicki.cn/.well-known/acme-challenge/0beMNTSzGirQygofZ2kyiexLjqPDSV3-XGUGpokFSNM: dial tcp: lookup dashboard.jicki.cn on 10.254.0.2:53: no such host
```


