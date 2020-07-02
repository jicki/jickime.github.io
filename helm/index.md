# helm 包管理



> Helm 是 Kubernetes 的包管理器，可以帮我们简化 kubernetes 的操作，一键部署应用。


# helm

## 安装依赖

```
# 首先要在所有 node 里安装 socat 软件

yum -y install socat

```


## 下载文件

```
# 下载 安装包 (googleapis.com 有可能访问不到，多试几次，或者自己翻墙下载)

wget https://storage.googleapis.com/kubernetes-helm/helm-v2.11.0-linux-amd64.tar.gz

tar zxvf helm-v2.11.0-linux-amd64.tar.gz 

cd linux-amd64/

mv helm /usr/local/bin/


# 验证服务

[root@kubernetes-1 ~]# helm version
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}

```


## 初始化 helm

```
# 初始化 helm

[root@kubernetes-1 ~]# helm init --skip-refresh

Creating /root/.helm/repository/repositories.yaml 
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com 
Adding local repo with URL: http://127.0.0.1:8879/charts 
$HELM_HOME has been configured at /root/.helm.

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.

Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
To prevent this, run `helm init` with the --tiller-tls-verify flag.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!

```


## 替换 repo

```
# helm 的 repo 默认是使用 谷歌的 repo ，国内访问不到，修改为 阿里云 的 repo

[root@kubernetes-1 ~]# helm repo list
NAME    URL                                             
stable  https://kubernetes-charts.storage.googleapis.com
local   http://127.0.0.1:8879/charts 



[root@kubernetes-1 ~]# helm repo remove stable
"stable" has been removed from your repositories



[root@kubernetes-1 ~]# helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
"stable" has been added to your repositories


[root@kubernetes-1 ~]# helm repo update
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 



[root@kubernetes-1 ~]# helm repo list
NAME    URL                                                   
local   http://127.0.0.1:8879/charts                          
stable  https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts


```


## 替换 tiller 镜像

```
# 默认 tiller 会使用 gcr.io 中的镜像，修改为自己的否则下载不到


helm init --upgrade --tiller-image=jicki/tiller:v2.11.0

```




## 修改 tiller rbac 为admin

```


# 创建 rbac 并修改为 admin 权限

kubectl create serviceaccount --namespace=kube-system tiller

kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller

kubectl patch deploy --namespace=kube-system tiller-deploy -p '{"spec":{"template":{"spec":{"serviceAccount":"tiller"}}}}'

```



```
# 查看 kubernetes pods

[root@kubernetes-1 ~]# kubectl get pods -n kube-system | grep tiller
tiller-deploy-7fc579878c-67vf8             1/1     Running   0          4m44s

```



```
# 验证tiller

[root@kubernetes-1 ~]# helm version
Client: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.11.0", GitCommit:"2e55dbe1fdb5fdb96b75ff144a339489417b146b", GitTreeState:"clean"}

```



## helm 命令

> Helm 一些常用命令


```
# 查询 charts 包

[root@kubernetes-1 ~]# helm search mysql
NAME                            CHART VERSION   APP VERSION     DESCRIPTION                                                 
stable/mysql                    0.3.5                           Fast, reliable, scalable, and easy to use open-source rel...
stable/percona                  0.3.0                           free, fully compatible, enhanced, open source drop-in rep...
stable/percona-xtradb-cluster   0.0.2           5.7.19          free, fully compatible, enhanced, open source drop-in rep...
stable/gcloud-sqlproxy          0.2.3                           Google Cloud SQL Proxy                                      
stable/mariadb                  2.1.6           10.1.31         Fast, reliable, scalable, and easy to use open-source rel...
```


```
# 查询 包 的具体信息


[root@kubernetes-1 ~]# helm inspect stable/mysql
description: Fast, reliable, scalable, and easy to use open-source relational database
  system.
engine: gotpl
home: https://www.mysql.com/
icon: https://www.mysql.com/common/logos/logo-mysql-170x115.png
keywords:
......

```


```
# 部署 程序

helm install stable/mysql
```



```
# 查询支持的选项,用于set 参数


[root@kubernetes-1 ~]# helm inspect values stable/mysql
## mysql image version
## ref: https://hub.docker.com/r/library/mysql/tags/
##
image: "mysql"
imageTag: "5.7.14"

## Specify password for root user
##
## Default: random 10 character string
# mysqlRootPassword: testing
.......

```

```
# 根据上面 查询 values 的mysqlRootPassword ，修改为 jicki


helm install --name mysql --set mysqlRootPassword=jicki  stable/mysql
````


```
# 升级 设置，或者版本


helm upgrade db-mysql --set mysqlRootPassword=passwd --set image="jicki/mysql" --set imageTag="5.7.20" stable/mysql

```

```
# 回滚 到upgrade 之前的版本


helm rollback db-mysql 1

```


```
# 添加 额外 repo


helm repo add my-repo https://kubernetes-charts.jicki.cn/

```


```
# 查询 repo 列表


helm repo list
```

