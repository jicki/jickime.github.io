# Jenkins Kubernetes Pipeline


# 1. Jenkins Kubernetes

> Jenkins作为最为流行的持续集成工具，有着丰富的用户群、强大的扩展能力、丰富的插件，是目前最为常见的CI/CD工具。在Jenkins 加强其Pipeline功能后，更是可以通过丰富的step库，实现各种复杂的流程。同时随着Docker的流行，Jenkins内也增加了对Docker的支持，可以实现容器内流程的执行, 随着 kubernetes 集群的流行，Jenkins 也有对于 Kubernetes 的插件，实现 pods 动态资源伸缩，构建时创建 pods, 完成时删除 pods, 既用既销。


# 2. Jenkins
> 这里 默认是已经部署好了 kubernetes 集群.


## 2.1 环境说明

|节点标识|hostname|系统|IP|
|-|-|-|-|
|K8S-Master-1|kubernetes-1|CentOS 7x64|192.168.168.11|
|K8S-Master-2|kubernetes-2|CentOS 7x64|192.168.168.12|
|K8S-Master-3|kubernetes-3|CentOS 7x64|192.168.168.13|
|K8S-Node-1|kubernetes-4|CentOS 7x64|192.168.168.14|
|K8S-Node-2|kubernetes-5|CentOS 7x64|192.168.168.15|


## 2.2 配置 NFS 共享存储

> 在部署 Jenkins 之前，我们需要先创建一个 共享存储，用于存放 Jenkins 生成的数据文件，以及配置文件。
> 
> 这里面我们选择 K8S-Node-2 这台服务器部署 NFS 服务。



### 2.2.1 安装NFS软件

```
#  安装 NFS 服务的软件

[root@kubernetes-5 ~]# yum -y install nfs-utils rpcbind

已安装:
  nfs-utils.x86_64 1:1.3.0-0.61.el7                                                                  rpcbind.x86_64 0:0.2.0-47.el7                                                                 

作为依赖被安装:
  gssproxy.x86_64 0:0.7.0-21.el7            keyutils.x86_64 0:1.5.8-3.el7       libbasicobjects.x86_64 0:0.1.1-32.el7    libcollection.x86_64 0:0.7.0-32.el7    libevent.x86_64 0:2.0.21-4.el7     
  libini_config.x86_64 0:1.3.1-32.el7       libnfsidmap.x86_64 0:0.25-19.el7    libpath_utils.x86_64 0:0.2.1-32.el7      libref_array.x86_64 0:0.1.5-32.el7     libtirpc.x86_64 0:0.2.4-0.15.el7   
  libverto-libevent.x86_64 0:0.2.5-4.el7    quota.x86_64 1:4.01-17.el7          quota-nls.noarch 1:4.01-17.el7           tcp_wrappers.x86_64 0:7.6-77.el7      

完毕！
```


```
# 其他所有 k8s 用于调度的服务器都必须安装 nfs 客户端

[root@kubernetes-1 ~]# yum -y install nfs-utils
[root@kubernetes-2 ~]# yum -y install nfs-utils
[root@kubernetes-3 ~]# yum -y install nfs-utils
[root@kubernetes-4 ~]# yum -y install nfs-utils

```


### 2.2.2 配置 NFS

> 在 NFS 服务器（K8S-Node-2）中创建 目录
> /opt/nfs/jenkins

```
[root@kubernetes-5 ~]# mkdir -p /opt/nfs/jenkins

```

```
# 修改配置文件
[root@kubernetes-5 ~]# vi /etc/exports

增加

/opt/nfs/jenkins       192.168.168.0/24(rw,sync,no_root_squash)

```

```
# 启动 NFS 服务

[root@kubernetes-5 ~]# systemctl enable rpcbind.service    
[root@kubernetes-5 ~]# systemctl enable nfs-server.service

[root@kubernetes-5 ~]# systemctl start rpcbind.service    
[root@kubernetes-5 ~]# systemctl start nfs-server.service

[root@kubernetes-5 ~]# systemctl status rpcbind.service    
[root@kubernetes-5 ~]# systemctl status nfs-server.service
```


```
# 查看服务
[root@kubernetes-5 ~]# showmount -e 192.168.168.15
Export list for 192.168.168.15:
/opt/nfs/jenkins 192.168.168.0/24

```

### 2.2.3 配置 NFS Client Provisioner

> NFS Client Provisioner 是一个自动配置NFS的服务, 它使现有和已经配置的NFS服务器，通过持久卷声明支持Kubernetes持久卷的动态配置。
> 
> 持久卷命名规则是  ${namespace}-${pvcName}-${pvName}



```
# 下载官方 yaml 文件

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/rbac.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/deployment.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/class.yaml

```


```
# 修改 deployment.yaml 文件，配置对应 NFS IP 与 path

apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.168.15
            - name: NFS_PATH
              value: /opt/nfs/jenkins
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.168.15
            path: /opt/nfs/jenkins
```


```
# 导入 服务 文件

[root@kubernetes-1 nfs-client]# kubectl apply -f .
storageclass.storage.k8s.io/managed-nfs-storage created
serviceaccount/nfs-client-provisioner created
deployment.extensions/nfs-client-provisioner created
serviceaccount/nfs-client-provisioner unchanged
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created

```


```
# 查看 storageclass

[root@kubernetes-1 nfs-client]# kubectl get storageclass
NAME                  PROVISIONER      AGE
managed-nfs-storage   fuseim.pri/ifs   39s


# 查看 pods
[root@kubernetes-1 nfs-client]# kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-cc5f6f9f6-d8njv   1/1     Running   0          53s

```


## 2.3 部署Jenkins

> 官方 github https://github.com/jenkinsci/kubernetes-plugin


```
# 下载 Jenkins 官方的 yml 文件


wget https://raw.githubusercontent.com/jenkinsci/kubernetes-plugin/master/src/main/kubernetes/jenkins.yml


wget https://raw.githubusercontent.com/jenkinsci/kubernetes-plugin/master/src/main/kubernetes/service-account.yml

```


```
# 修改 jenkins.yml 文件, 修改后文件如下:
# 主要是修改  volumeClaimTemplates: 
# Ingress 部分的域名



# jenkins

---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: jenkins
  labels:
    name: jenkins
spec:
  serviceName: jenkins
  replicas: 1
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      name: jenkins
      labels:
        name: jenkins
    spec:
      terminationGracePeriodSeconds: 10
      serviceAccountName: jenkins
      containers:
        - name: jenkins
          image: jenkins/jenkins:lts-alpine
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
            - containerPort: 50000
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 0.5
              memory: 500Mi
          env:
            - name: LIMITS_MEMORY
              valueFrom:
                resourceFieldRef:
                  resource: limits.memory
                  divisor: 1Mi
            - name: JAVA_OPTS
              # value: -XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -XX:MaxRAMFraction=1 -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85
              value: -Xmx$(LIMITS_MEMORY)m -XshowSettings:vm -Dhudson.slaves.NodeProvisioner.initialDelay=0 -Dhudson.slaves.NodeProvisioner.MARGIN=50 -Dhudson.slaves.NodeProvisioner.MARGIN0=0.85 -Dorg.csanchez.jenkins.plugins.kubernetes.clients.cacheExpiration=60
          volumeMounts:
            - name: jenkins-home
              mountPath: /var/jenkins_home
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12 # ~2 minutes
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            failureThreshold: 12 # ~2 minutes
      securityContext:
        fsGroup: 1000
  volumeClaimTemplates:
  - metadata:
      name: jenkins-home
      annotations:
        volume.beta.kubernetes.io/storage-class: "managed-nfs-storage"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: jenkins
spec:
  # type: LoadBalancer
  selector:
    name: jenkins
  # ensure the client ip is propagated to avoid the invalid crumb issue when using LoadBalancer (k8s >=1.7)
  #externalTrafficPolicy: Local
  ports:
    -
      name: http
      port: 80
      targetPort: 8080
      protocol: TCP
    -
      name: agent
      port: 50000
      protocol: TCP

---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: jenkins
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    kubernetes.io/tls-acme: "true"
    # "413 Request Entity Too Large" uploading plugins, increase client_max_body_size
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
    nginx.ingress.kubernetes.io/proxy-request-buffering: "off"
    # For nginx-ingress controller < 0.9.0.beta-18
    ingress.kubernetes.io/ssl-redirect: "true"
    # "413 Request Entity Too Large" uploading plugins, increase client_max_body_size
    ingress.kubernetes.io/proxy-body-size: 50m
    ingress.kubernetes.io/proxy-request-buffering: "off"
spec:
  rules:
  - http:
      paths:
      - path: /
        backend:
          serviceName: jenkins
          servicePort: 80
    host: jenkins.jicki.cn
  tls:
  - hosts:
    - jenkins.jicki.cn
    secretName: tls-jenkins

```



```
# 导入文件

[root@kubernetes-1 jenkins]# kubectl apply -f .
statefulset.apps/jenkins created
service/jenkins created
ingress.extensions/jenkins created
serviceaccount/jenkins created
role.rbac.authorization.k8s.io/jenkins created
rolebinding.rbac.authorization.k8s.io/jenkins created

```



```
# 查看 pods , svc, ingress

[root@kubernetes-1 jenkins]# kubectl get pods
NAME                                     READY   STATUS    RESTARTS   AGE
jenkins-0                                1/1     Running   0          2m42s



[root@kubernetes-1 jenkins]# kubectl get svc
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)            AGE
jenkins      ClusterIP   10.254.28.67   <none>        80/TCP,50000/TCP   2m52s


[root@kubernetes-1 jenkins]# kubectl get ingress
NAME            HOSTS              ADDRESS   PORTS     AGE
jenkins         jenkins.jicki.cn             80, 443   3m2s

```


```
# 测试访问 https://jenkins.jicki.cn

```

![图1][1]


```
# 查询密码 

[root@kubernetes-1 jenkins]# kubectl logs pods/jenkins-0 


.....

*************************************************************
*************************************************************
*************************************************************

Jenkins initial setup is required. An admin user has been created and a password generated.
Please use the following password to proceed to installation:

ecaf26a8ca154e359b2fecd519aa753e

This may also be found at: /var/jenkins_home/secrets/initialAdminPassword

*************************************************************
*************************************************************
*************************************************************


```

![图2][2]

![图3][3]


```
# 至此 jenkins 安装部署完成
```



## 2.4 配置 Jenkins

> Jenkins 支持 kubernetes 需要 安装 kubernetes plugin 这个插件



### 2.4.1 安装kubernetes 插件

> 系统管理 --> 插件管理 --> 可选插件 --> kubernetes plugin


![图4][4]

![图5][5]


### 2.4.2 配置 kubernetes 插件

> 系统管理 --> 系统设置 --> 云 --> 新增一个云 --> kubernetes


![图6][6]

![图7][7]

![图8][8]


```
# 配置说明: 

1. Name 处默认为 kubernetes，也可以修改为其他名称，如果这里修改了，下边在执行 Job 时指定 podTemplate() 参数 cloud 为其对应名称，否则会找不到，cloud 默认值取：kubernetes


2. Kubernetes URL 处我填写了 https://kubernetes.default 这里我填写了 Kubernetes Service 对应的 DNS 记录，通过该 DNS 记录可以解析成该 Service 的 Cluster IP，注意：也可以填写 https://kubernetes.default.svc.cluster.local , 或者直接填写外部 Kubernetes 的地址 https://<ClusterIP>:<Ports>。


3. Jenkins URL 处我填写了 http://jenkins.default，跟上边类似，也是使用 Jenkins Service 对应的 DNS 记录.

```


> 连接测试

![图9][9]


## 2.5 Pipeline

> pipeline 是一套jenkins官方提供的插件，它可以用来在jenkins 中实现和集成连续交付。
> 

![图10][10]

> 如上图所示 pipeline 定义了一个流程，流程中完成过一个 CI/CD 流程的步骤.

* Pipeline 我们可以分为 step , node , stage
* node: 是 pipleline里groovy的一个概念，node可以给定参数用来选择 agent，node里的steps将会运行在node选择的agent上, job里可以有多个node，将job的steps按照需求运行在不同的 agent 中.
* stage: 是 pipeline里groovy里引入的一个虚拟的概念，是一些step的集合，通过stage我们可以将job的所有steps划分为不同的stage，使得整个job像管道一样更容易维护.
* step: 理解为 job, 是 pipeline 的一个任务,是最小的单位.



## 2.5.1 创建 Pipeline

> Pipeline 也是 Jenkins 的一个插件，需要先安装 pipeline 插件, 安装 jenkins 的时候选择默认插件安装，会默认安装这个插件.
> 
> 新建任务 --> 输入任务名称 --> 选择 流水线(pipeline)


![图11][11]



```
# 在 流水线 中选择 Pipeline script


def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: 'label', cloud: 'kubernetes') {
    node('label') {
        stage('Run shell') {
            sh 'echo hello world'
        }
    }
}

```

![图12][12]


![图13][13]



```
# 添加多个 stage 

def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: 'label', cloud: 'kubernetes') {
    node('label') {
        stage('Git Pull') {
            sh 'echo Git Pull Job'
        }
        stage('Build Job') {
            sh 'echo Gradle Job Build'
        }
        stage('Docker Build') {
            sh 'echo Dokcer Build Image'
        }   
        stage('Push Docker Imges') {
            sh 'echo Docker Push Imges To Registry'
        } 
        stage('Update Deployment') {
            sh 'echo Update Deployment Image'
        }
        stage('Send Message') {
            sh 'echo Send Message to Wechat'
        }       
    }
}

```

![图14][14]



### 2.5.2 基于自己的 slave 镜像构建

> 在使用 pipeline 的时候默认会启动 官方的 image
>
>  jenkins/jnlp-slave:alpine 去运行我们的流程
> 
> 当然我们也可以构建自己的 slave 镜像，指定运行.



```
# 首先需要下载官方的 jenkins-slave 文件

wget https://raw.githubusercontent.com/jenkinsci/docker-jnlp-slave/master/jenkins-slave

```

```
# 这里使用我个人的镜像
# 镜像基于 alpine  软件包含  oracle 1.8 与 gradle 4.5 构建
# 镜像名称为 jicki/slave:3.16

# 创建 dockerfile


vi dockerfile



FROM jicki/slave:3.16

COPY jenkins-slave /usr/local/bin/jenkins-slave

USER root
RUN chmod +x /usr/local/bin/jenkins-slave

USER jenkins
ENTRYPOINT ["jenkins-slave"]

```

```
# 构建镜像
docker build -t="jicki/jenkins-jnlp" .

```


```
# 配置 Pipeline 


def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: 'label', cloud: 'kubernetes', containers: [
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
    node('label') {
        stage('Task-1') {
            stage('show Java version') {
                sh 'java -version'
            }
            stage('show Gradle version') {
                sh 'gradle -version'
            }            
        }
    }
}

```

![图15][15]



### 2.5.3 完整的 java 项目

> 通过 Pipeline Scm 插件， CheckOut git 代码，然后执行 Gradle 打包





```
# 以下为配置的 Pipeline


def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: 'label', cloud: 'kubernetes', containers: [
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
    node('label') {
        stage('Task-1') {
            stage('Git CheckOut') {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CleanBeforeCheckout']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '4f419afa-8f63-45d8-85af-66381f17b924', url: 'http://192.168.168.14:3000/jicki/demo.git']]])
                echo 'Checkout'
            }
            stage('show Java version') {
                sh 'java -version'
            }
            stage('show Gradle version') {
                sh 'gradle -version'
            }  
            stage('Gradle Build') {
                echo 'Gradle Building'
                sh 'gradle clean build jar --stacktrace --debug'
                echo 'Show Build jar'
                sh 'ls -lt /home/jenkins/workspace/${JOB_NAME}/build/libs/*.jar'
            }
        }
    }
}

```

```
# 这里面备注一下 Pipeline SCM 插件

checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CleanBeforeCheckout']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '4f419afa-8f63-45d8-85af-66381f17b924', url: 'http://192.168.168.14:3000/jicki/demo.git']]])

# credentialsId 标签 这里是 Jenkins 里创建的一个 凭据, 这个凭据 的用户名密码为 Git 代码库的用户密码.
# 创建完毕以后, 编辑里可以查看到 Id
```


![图16][16]

![图17][17]


> 下面增加一个 docker build 流程

```
# 编辑以上 jicki/jenkins-jnlp 镜像, 增加 docker
```



```
# dockerfile

FROM frolvlad/alpine-java

ENV GRADLE_VERSION=5.4.1 \
    GRADLE_HOME=/opt/gradle \
    GRADLE_FOLDER=/root/.gradle \
    DOCKER_CHANNEL=stable \
    DOCKER_VERSION=18.09.6 \
    HOME=/home/jenkins

USER root

# Change to tmp folder
WORKDIR /tmp

# Download and extract gradle to opt folder
RUN wget https://downloads.gradle.org/distributions/gradle-${GRADLE_VERSION}-bin.zip \
    && unzip gradle-${GRADLE_VERSION}-bin.zip -d /opt \
    && ln -s /opt/gradle-${GRADLE_VERSION} /opt/gradle \
    && rm -f gradle-${GRADLE_VERSION}-bin.zip \
    && ln -s /opt/gradle/bin/gradle /usr/bin/gradle \
    && apk add --no-cache libstdc++ git libltdl wget \
    ca-certificates \
    && echo 'hosts: files dns' > /etc/nsswitch.conf \
    && mkdir -p $GRADLE_FOLDER \
    && addgroup -S -g 10000 jenkins \
    && adduser -D -S -G jenkins -u 10000 jenkins

LABEL Description="This is a base image, which provides the Jenkins agent executable (slave.jar)" Vendor="Jenkins project" Version="3.19"
ARG VERSION=3.19
ARG AGENT_WORKDIR=/home/jenkins/agent

RUN apk add --no-cache --virtual .build-deps \
    curl

RUN curl --create-dirs -sSLo /usr/share/jenkins/slave.jar https://repo.jenkins-ci.org/public/org/jenkins-ci/main/remoting/${VERSION}/remoting-${VERSION}.jar \
  && chmod 755 /usr/share/jenkins \
  && chmod 644 /usr/share/jenkins/slave.jar \
  && apk del .build-deps

ENV AGENT_WORKDIR=${AGENT_WORKDIR}
RUN mkdir /home/jenkins/.jenkins && mkdir -p ${AGENT_WORKDIR}

# Mark as volume
VOLUME  $GRADLE_FOLDER
VOLUME /home/jenkins/.jenkins
VOLUME ${AGENT_WORKDIR}

RUN set -eux; \
        \
# this "case" statement is generated via "update.sh"
        apkArch="$(apk --print-arch)"; \
        case "$apkArch" in \
# amd64
                x86_64) dockerArch='x86_64' ;; \
# arm32v6
                armhf) dockerArch='armel' ;; \
# arm32v7
                armv7) dockerArch='armhf' ;; \
# arm64v8
                aarch64) dockerArch='aarch64' ;; \
                *) echo >&2 "error: unsupported architecture ($apkArch)"; exit 1 ;;\
        esac; \
        \
        if ! wget -O docker.tgz "https://download.docker.com/linux/static/${DOCKER_CHANNEL}/${dockerArch}/docker-${DOCKER_VERSION}.tgz"; then \
                echo >&2 "error: failed to download 'docker-${DOCKER_VERSION}' from '${DOCKER_CHANNEL}' for '${dockerArch}'"; \
                exit 1; \
        fi; \
        \
        tar --extract \
                --file docker.tgz \
                --strip-components 1 \
                --directory /usr/local/bin/ \
        ; \
        rm docker.tgz; \
        \
        dockerd --version; \
        docker --version

# Down Docker File
RUN wget https://raw.githubusercontent.com/docker-library/docker/cdcff675cecbae122cbae49ed6e17fa78bb6116a/18.09/modprobe.sh  -O /usr/local/bin/modprobe
RUN wget https://raw.githubusercontent.com/docker-library/docker/cdcff675cecbae122cbae49ed6e17fa78bb6116a/18.09/docker-entrypoint.sh -O /usr/local/bin/docker-entrypoint.sh

# Down jenkins-slave file
RUN wget https://raw.githubusercontent.com/jenkinsci/docker-jnlp-slave/master/jenkins-slave -O /usr/local/bin/jenkins-slave

# Chomd
RUN chmod +x /usr/local/bin/jenkins-slave \
    && chmod +x /usr/local/bin/modprobe \
    && chmod +x /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["jenkins-slave"]

```


```
# build

docker build -t="jicki/jenkins-jnlp:docker"


```

> 这里 gradle 项目 需要增加一点 变量, 修改 build.gradle 文件

```
# 完整的 build.gradle 文件如下

buildscript {
    repositories {
        maven { url 'http://maven.aliyun.com/nexus/content/groups/public/' } // aliyun
        mavenLocal() 
    }
    dependencies {
        classpath("org.springframework.boot:spring-boot-gradle-plugin:1.5.10.RELEASE")
        classpath "io.spring.gradle:dependency-management-plugin:1.0.5.RELEASE"
    }
}
plugins {
    id 'java'
    id 'idea'
    id 'eclipse'
    id 'net.researchgate.release' version '2.5.0'
    id 'com.github.ben-manes.versions' version '0.14.0'
}

apply plugin: 'project-report'
apply plugin: 'maven-publish'
apply plugin: 'org.springframework.boot'
apply plugin: "io.spring.dependency-management"

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

jar {
    baseName = 'demo'
    version = ''
}

repositories {
        mavenCentral()
}

dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-web'
        testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

bootRepackage {
    doLast {
        File envFile = new File("build/tmp/PROJECT_ENV")
        println("Create ${archivesBaseName} ENV File ===> " + envFile.createNewFile())
        envFile.write("export PROJECT_BUILD_FINALNAME=${archivesBaseName}\n")
        println("Generate Docker image tag...")
        envFile.append("export BUILD_DATE=`date +%Y%m%d%H%M%S`\n")
        envFile.append("export IMAGE_NAME=jicki/demo:`echo \${CI_BUILD_REF_NAME} | cut -c1-8`\${BUILD_DATE}\n")
        envFile.append("export LATEST_IMAGE_NAME=jicki/demo:latest\n")
    }
}

```









> Pipeline 流程 , git pull, gradle build, docker build

```

def label = "mypod-${UUID.randomUUID().toString()}"
podTemplate(label: 'label', cloud: 'kubernetes', containers: [
    containerTemplate(
        name: 'jnlp', 
        image: 'jicki/jenkins-jnlp:docker', 
        alwaysPullImage: true,
        args: '${computer.jnlpmac} ${computer.name}'),
],
  volumes: [
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
    hostPathVolume(mountPath: '/root/.gradle', hostPath: '/opt/data/gradle'),
],)
{
    node('label') {
        stage('Task-1') {
            stage('Git CheckOut') {
                checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [[$class: 'CleanBeforeCheckout']], submoduleCfg: [], userRemoteConfigs: [[credentialsId: '4f419afa-8f63-45d8-85af-66381f17b924', url: 'http://192.168.168.14:3000/jicki/demo.git']]])
                echo 'Checkout'
            }
            stage('Gradle Build') {
                echo 'Gradle Building'
                sh 'gradle clean build jar --stacktrace --debug'
            }
            stage('Docker Build') {
                sh '''
                echo ----------------
                cat dockerfile
                echo ----------------
                source build/tmp/PROJECT_ENV
                docker build -t ${IMAGE_NAME} --build-arg PROJECT_BUILD_FINALNAME=${PROJECT_BUILD_FINALNAME} .
                docker images
                '''
            }
        }
    }
}

```


```
# git 项目底下的 dockerfile  , 以下为 dockerfile


FROM jicki/openjdk:1.8-alpine

ARG PROJECT_BUILD_FINALNAME

ENV TZ 'Asia/Shanghai'
ENV PROJECT_BUILD_FINALNAME ${PROJECT_BUILD_FINALNAME}


COPY build/libs/${PROJECT_BUILD_FINALNAME}.jar /${PROJECT_BUILD_FINALNAME}.jar

CMD ["sh","-c","java -jar /${PROJECT_BUILD_FINALNAME}.jar"]

```

![图18][18]

![图19][19]

  [1]: https://jicki.cn/img/posts/pipeline/1.png
  [2]: https://jicki.cn/img/posts/pipeline/2.png
  [3]: https://jicki.cn/img/posts/pipeline/3.png 
  [4]: https://jicki.cn/img/posts/pipeline/4.png 
  [5]: https://jicki.cn/img/posts/pipeline/5.png 
  [6]: https://jicki.cn/img/posts/pipeline/6.png 
  [7]: https://jicki.cn/img/posts/pipeline/7.png 
  [8]: https://jicki.cn/img/posts/pipeline/8.png 
  [9]: https://jicki.cn/img/posts/pipeline/9.png 
  [10]: https://jicki.cn/img/posts/pipeline/10.png 
  [11]: https://jicki.cn/img/posts/pipeline/11.png 
  [12]: https://jicki.cn/img/posts/pipeline/12.png 
  [13]: https://jicki.cn/img/posts/pipeline/13.png 
  [14]: https://jicki.cn/img/posts/pipeline/14.png 
  [15]: https://jicki.cn/img/posts/pipeline/15.png 
  [16]: https://jicki.cn/img/posts/pipeline/16.png 
  [17]: https://jicki.cn/img/posts/pipeline/17.png
  [18]: https://jicki.cn/img/posts/pipeline/18.png 
  [19]: https://jicki.cn/img/posts/pipeline/19.png

