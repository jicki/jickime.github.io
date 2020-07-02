# kubernetes jenkins



> jenkins 所有的资源全部在 kubernetes 中进行，并配置 kubernetes 云
> 针对 jenkins slave 做动态资源伸缩，构建时申请 pod, 操作完毕后自动删除 pod

# 基于 Kubernetes Jenkins slave 动态资源伸缩 pod


## 环境说明

> 请自行配置 kubernetes 集群, kubernetes 版本为 1.9.1

```
172.16.1.64 - kubernetes master
172.16.1.65 - kubernetes master or node
172.16.1.66 - kubernetes node
```


## 先决条件


> 后端存储, 用于存放 jenkins home 目录，以及 slave 构建依赖包目录


```
# 配置jenkins 后端存储, 如 nfs, gfs, cephfs 等， 请自行配置，或者参考 之前的文章。

# 基于单独的 nfs 最为后端 文档在这里 https://jicki.cn/2017/01/01/kubernetes-nfs/ 

```


## 创建 jenkins master



```
# 首先创建一个 namespace

vi jenkins-namespace.yaml


apiVersion: v1
kind: Namespace
metadata:
  name: jenkins

```

```
# 导入文件

kubectl apply -f jenkins-namespace.yaml 
namespace "jenkins" created


# 查看

[root@kubernetes-64 jenkins]# kubectl get namespace
NAME            STATUS    AGE
default         Active    26d
ingress-nginx   Active    15d
jenkins         Active    23s

```




```
# 创建 jenkins 的 rbac , jenkins 与 api-service 通讯必须要 rabc 认证


vi jenkins-rbac.yaml

---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: jenkins
  name: jenkins-admin
  namespace: jenkins

---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
  namespace: jenkins
  labels:
    k8s-app: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: jenkins


```


```
# 导入文件

[root@kubernetes-64 jenkins]# kubectl apply -f jenkins-rbac.yaml 
serviceaccount "jenkins-admin" created
clusterrolebinding "jenkins-admin" created

# 查看
[root@kubernetes-64 jenkins]# kubectl get serviceaccount -n jenkins
NAME            SECRETS   AGE
default         1         3h
jenkins-admin   1         1m


```



```
vi jenkins-deployment.yaml


apiVersion: apps/v1beta1
kind: Deployment  
metadata:  
  name: jenkins
  namespace: jenkins
spec:  
  replicas: 1  
  strategy:  
    type: RollingUpdate  
    rollingUpdate:  
      maxSurge: 2  
      maxUnavailable: 0  
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
        nfs:  
          server: 172.16.1.66
          path: "/opt/data/jenkins"


---
kind: Service  
apiVersion: v1  
metadata:  
  labels:  
      app: jenkins  
  name: jenkins
  namespace: jenkins
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
  namespace: jenkins
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
# 导入 文件

[root@kubernetes-64 jenkins]# kubectl apply -f jenkins-deployment.yaml 
deployment "jenkins" created
service "jenkins" created
ingress "jenkins" created



# 查看 服务

[root@kubernetes-64 jenkins]# kubectl get pods -n jenkins
NAME                     READY     STATUS    RESTARTS   AGE
jenkins-c5f45c84-8vlcp   1/1       Running   0          6m
[root@kubernetes-64 jenkins]# 
[root@kubernetes-64 jenkins]# kubectl get svc -n jenkins
NAME      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)              AGE
jenkins   ClusterIP   10.254.47.35   <none>        8080/TCP,50000/TCP   6m
[root@kubernetes-64 jenkins]# 
[root@kubernetes-64 jenkins]# kubectl get ingress -n jenkins   
NAME      HOSTS              ADDRESS   PORTS     AGE
jenkins   jenkins.jicki.cn             80        6m



# 查看 jenkins 初始密码

[root@kubernetes-64 jenkins]# kubectl logs pods/jenkins-c5f45c84-8vlcp -n jenkins


*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

b2458cfb46724268a4db025c54f172ab

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************
```


## 配置 jenkins


```
# 登录 web ui

http://jenkins.jicki.cn

```

![jenkins1][1]
![jenkins2][2]



```
# Master 需要安装的插件

1. Pipeline
2. kubernetes plugin 
3. Build WIth Parameters    (构建 输入参数的插件)
4. Git Parameter Plug-In  ( git 分支 构建选择)


```


![jenkins3][3]


## 配置 kubernetes 云


```
# 点击【系统管理】-【系统设置】-【新增一个云】-【Kubernetes】
```

![jenkins4][4]

![jenkins5][5]



### 基于 pipeline 模式

```
# 创建一个基于 pipeline 的配置  cloud 字段配置为 kubernetes 云的 name .


podTemplate(label: 'jenkins-pod', cloud: 'Jenkins-Cloud') {
    node('jenkins-pod') {
        stage('Run shell') {
            sh 'echo hello world'
        }
    }
}

```


```
# 点击立即构建, 控制台输出如下:


Started by user 小炒肉
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] podTemplate
[Pipeline] {
[Pipeline] node
Still waiting to schedule task
jenkins-slave-bkcfk-vpzz0 is offline
Running on jenkins-slave-bkcfk-vpzz0 in /home/jenkins/workspace/test-k8s
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Run shell)
[Pipeline] sh
[test-k8s] Running shell script
+ echo hello world
hello world
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // podTemplate
[Pipeline] End of Pipeline
Finished: SUCCESS

```

### 基于 自由风格模式


```

# 点击【系统管理】-【系统设置】-【新增一个云】-【Kubernetes】- 【 Kubernetes Pod Template 】
  - 【 Container Template 】

具体参数 如下：

```

![jenkins6][6]
![jenkins7][7]



```
# 添加一个 自动风格的 pj

```

![jenkins8][8]
![jenkins9][9]
![jenkins10][10]
![jenkins11][11]
![jenkins12][12]


## 制作 slave 镜像

> 官方默认的 jenkins-slave 里面不存在任何的工具，依赖，项目环境等，我们需要制作一个属于自己项目的 jenkins-slave 镜像。

> 官方 jenkins-slave dockerfile 地址 https://github.com/jenkinsci/docker-jnlp-slave
> 官方 slave  https://github.com/jenkinsci/docker-slave





```
# 个人的 oracle-jdk-8 镜像 java -version = 1.8.0_161

docker pull jicki/oracle-jdk:8

# 个人的 gradle 镜像 

docker pull jicki/gradle:4.5

```



```
# 基于 jicki/gradle:4.5 的一个 jenkins slave


docker pull jicki/slave:3.16

```



```

# 获取官方的git，主要是 jenkins-slave 脚本文件

git clone https://github.com/jenkinsci/docker-jnlp-slave



# 重新编辑 dockerfile 官方默认使用 openjdk-1.8

vi  dockerfile

FROM jicki/slave:3.16

COPY jenkins-slave /usr/local/bin/jenkins-slave

USER root
RUN chmod +x /usr/local/bin/jenkins-slave

USER jenkins
ENTRYPOINT ["jenkins-slave"]
    
```

```
docker build -t="jicki/jenkins-jnlp" .

```




```
# 测试自己的镜像


podTemplate(label: 'jicki-jenkins', cloud: 'Jenkins-Cloud', containers: [
    containerTemplate(
        name: 'jnlp', 
        image: 'jicki/jenkins-jnlp', 
        alwaysPullImage: true, 
        args: '${computer.jnlpmac} ${computer.name}'),

  ],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
],) 
{
    node('jicki-jenkins') {
        stage('Task-1') {
            stage('show java version') {
                sh 'java -version'
            }
        }
    }
    node('jicki-jenkins') {
        stage('Task-2') {
            stage('show gradle version') {
                sh 'gradle -version'
            }
        }
    }
}

```


```

Started by user 小炒肉
Running in Durability level: MAX_SURVIVABILITY
[Pipeline] podTemplate
[Pipeline] {
[Pipeline] node
Still waiting to schedule task
Waiting for next available executor on jenkins-slave-9v0g5-73tng
Running on jenkins-slave-9v0g5-73tng in /home/jenkins/workspace/test-2
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Task-1)
[Pipeline] stage
[Pipeline] { (show java version)
[Pipeline] sh
[test-2] Running shell script
+ java -version
java version "1.8.0_161"
Java(TM) SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.161-b12, mixed mode)
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] node
Still waiting to schedule task
jenkins-slave-9v0g5-nxdf8 is offline
Running on jenkins-slave-9v0g5-nxdf8 in /home/jenkins/workspace/test-2
[Pipeline] {
[Pipeline] stage
[Pipeline] { (Task-2)
[Pipeline] stage
[Pipeline] { (show gradle version)
[Pipeline] sh
[test-2] Running shell script
+ gradle -version

------------------------------------------------------------
Gradle 4.5
------------------------------------------------------------

Build time:   2018-01-24 17:04:52 UTC
Revision:     77d0ec90636f43669dc794ca17ef80dd65457bec

Groovy:       2.4.12
Ant:          Apache Ant(TM) version 1.9.9 compiled on February 2 2017
JVM:          1.8.0_161 (Oracle Corporation 25.161-b12)
OS:           Linux 4.4.111-1.el7.elrepo.x86_64 amd64

[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // stage
[Pipeline] }
[Pipeline] // node
[Pipeline] }
[Pipeline] // podTemplate
[Pipeline] End of Pipeline
Finished: SUCCESS


```


  [1]: https://jicki.cn/img/posts/jenkins/jenkins1.png
  [2]: https://jicki.cn/img/posts/jenkins/jenkins2.png
  [3]: https://jicki.cn/img/posts/jenkins/jenkins3.png
  [4]: https://jicki.cn/img/posts/jenkins/jenkins4.png
  [5]: https://jicki.cn/img/posts/jenkins/jenkins5.png
  [6]: https://jicki.cn/img/posts/jenkins/jenkins6.png
  [7]: https://jicki.cn/img/posts/jenkins/jenkins7.png
  [8]: https://jicki.cn/img/posts/jenkins/jenkins8.png
  [9]: https://jicki.cn/img/posts/jenkins/jenkins9.png
  [10]: https://jicki.cn/img/posts/jenkins/jenkins10.png
  [11]: https://jicki.cn/img/posts/jenkins/jenkins11.png
  [12]: https://jicki.cn/img/posts/jenkins/jenkins12.png

