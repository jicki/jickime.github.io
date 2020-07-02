# jenkins gradle docker 持续集成




# 部署 jenkins


> 基于  jenkins 持续集成 自动打包 构建镜像 更新版本



## 初始化环境

```
# 安装 open-jdk

yum -y install java


# 安装 kubectl

yum install -y kubectl


# 拷贝 master 证书到 本机

证书文件一般在 /etc/kubernetes/ssl 下

需要拷贝的证书有

ca.pem
admin.pem
admin-key.pem


# 执行如下脚本 

vi master.sh

#!/bin/bash

KUBE_API_SERVER="https://master-api"
CERT_DIR=${CERT_DIR-"."}

kubectl config set-cluster default-cluster --server=${KUBE_API_SERVER} \
    --certificate-authority=${CERT_DIR}/ca.pem 

kubectl config set-credentials default-admin \
    --certificate-authority=${CERT_DIR}/ca.pem \
    --client-key=${CERT_DIR}/admin-key.pem \
    --client-certificate=${CERT_DIR}/admin.pem      

kubectl config set-context default-system --cluster=default-cluster --user=default-admin
kubectl config use-context default-system



# 注意修改 master api 地址
# 这里一定要切换 jenkins 用户，否则执行报错

su jenkins
sh master.sh





# 安装 docker 这里略过了

# 配置 jenkins 的 docker 权限

vim /etc/sudoers

## Allow root to run any commands anywhere 
root    ALL=(ALL)       ALL
jenkins ALL=(ALL)       ALL


# 加到 root 组里
usermod -aG docker jenkins


# 这里要注意，如果你之前安装了jenkins 修改以后要重启jenkins



```


## 安装 jenkins

```
# 导入 jenkins yum 源
wget -O /etc/yum.repos.d/jenkins.repo http://pkg.jenkins.io/redhat-stable/jenkins.repo

# 导入 key
rpm --import http://pkg.jenkins.io/redhat-stable/jenkins.io.key


# 安装 jenkins
yum -y install jenkins

```

## 配置 jenkins

```
# 修改 默认目录

vi /etc/sysconfig/jenkins

JENKINS_HOME="/opt/jenkins"


# JVM 调优 (4G内存限制2G内存)

JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true -Xms1024m -Xmx2048m -XX:PermSize=512m -XX:MaxPermSize=1024m"


# 创建 目录 

mkdir -p /opt/jenkins

# 授权

chown -R jenkins:jenkins /opt/jenkins

```


## 启动 服务

```
systemctl start jenkins
chkconfig jenkins on

```

## 访问测试

```
http://myip:8080/


# 查看初始化密码

cat /opt/jenkins/secrets/initialAdminPassword
8e082d8cd85e4207a148bf2429e32f59
```



## 配置 用户密钥

```
# 打开 jenkins 用户 bash

vi /etc/passwd

jenkins 用户  

/var/lib/jenkins 修改为 /home/jenkins
/bin/false 修改为 /bin/bash


# 创建 home 目录

mkdir /home/jenkins

# 授权

chown -R jenkins:jenkins /home/jenkins

#  切换用户
su jenkins



# 生成key
ssh-keygen -t rsa -b 4096 -C "jenkins@git"


# 查看 公钥
cat /home/jenkins/.ssh/id_rsa.pub

```


## 添加 Credentials

```
# 登陆 web ui --> Credentials --> System
  --> Global Credentials --> Add Credentials

Kind: SSH Username with private key
    Scope: Global(Jenkins, nodes, items, all child tiems, etc)
    Username: jenkins-git
    Private Key: Form the Jenkins master ~/.ssh
    Passphrase: 
    ID: 
    Decription: jenins-git
    
```

## 添加 插件

```
# 登陆 web ui -->  系统管理 --> 插件管理


1. Pipeline
2. Gradle Plugin
3. Git plugin
4. Build WIth Parameters    (构建 输入参数的插件)
5. Email Extension Plugin   (邮件 发送 插件)
6. Multiple SCMs Plugin   (多 git 版本库 同时构建)
7. Git Parameter Plug-In  ( git 分支 构建选择)
8. description setter plugin ( 配置 Build History 显示具体信息 )
9. user build vars plugin  ( 显示 构建用户名 而非 id )
```


## 配置 全局组件

```
# 登陆 web ui --> 系统管理 --> Global Tool Configuration

JDK: (新增JDK)
    JDK: orace-jdk-8
    JAVA_HOME: /usr/java/jdk1.8.0_131/

Git: 
   Name: git-1.8.3
   Path to Git executable: /usr/bin/git
 
   
 Gradle:
    Name: gradle-2.5
    GRADLE_HOME: /opt/gradle
   
Docker:
    Name: docker-1.13.1
    Installation root: /opt/docker      (docker info |grep Root)

```


## 配置插件


```
# 登陆 web ui --> 系统管理 --> 系统设置

Extended E-mail Notification: (没有标注 表示为空)

# SMTP server: 
xxxx.mail.com

# Default Content Type: 
HTML(text/html)

# Default Subject: 
Default Subject =  构建通知:$PROJECT_NAME - Build # $BUILD_NUMBER - $BUILD_STATUS!

# Default Content: 
<b style="font-size:12px">(本邮件是程序自动下发的，请勿回复，<span style="color:red">请相关人员fix it,重新提交到git 构建</span>)<br></b><hr>
<b style="font-size: 12px;">项目名称：$PROJECT_NAME<br></b><hr>
<b style="font-size: 12px;">构建编号：$BUILD_NUMBER<br></b><hr>
<b style="font-size: 12px;">GIT版本号：${GIT_REVISION}<br></b><hr>
<b style="font-size: 12px;">构建状态：$BUILD_STATUS<br></b><hr>
<b style="font-size: 12px;">触发原因：${CAUSE}<br></b><hr>
<b style="font-size: 12px;">构建日志地址：<a href="${BUILD_URL}console">${BUILD_URL}console</a><br></b><hr>
<b style="font-size: 12px;">构建地址：<a href="$BUILD_URL">$BUILD_URL</a><br></b><hr>
<b style="font-size: 12px;">变更集:${JELLY_SCRIPT,template="html"}<br></b><hr>


Default Triggers:
Failure - Any 
Success

```


## 创建项目

```
# 创建一个 自由风格 项目

Enter an item name:
java test



General:
参数化构建过程 --> 添加参数 --> String Parameter:
# 这里定义一个 参数，用来选择打包的项目，因为我这边一个项目下有多个war包，这里我是用 gradle 构建项目的，其他工具可以自行处理。

名字: models

默认值:(这里取settings.gradle 中的 include, 如下只是测试，只做参考) 
"java_1","java_2"

描述:
填写需要打包的项目
格式为： "项目名称"   多个项目以, 号隔开
如：  "java_1"    或者 "java_1","java_2" 


参数化构建过程  --> 添加参数 --> Git Parameter:

Name: 
    git_tag
Parameter Type:
    Branch or Tag

Default Value:
    origin/master


# 源码管理 ( 这里需要填写上面参数化构建 Git Parameter 名称)
  Git:

Branches to build
    Branch Specitfier (blank for 'any'):
            $git_tag

```


## Build Environment

```
# 添加 user 的 var

#勾选如下:
    Set jenkins user build variables


# 变量说明如下:

This plugin is used to set user build variables:
BUILD_USER -- full name of user started build,
BUILD_USER_FIRST_NAME -- first name of user started build,
BUILD_USER_LAST_NAME -- last name of user started build,
BUILD_USER_ID -- id of user started build.
BUILD_USER_EMAIL -- email of user started build.
```



## 构建

```
1. 选择一个 Execute shell
  Command: (这里是设置选择构建项目，因为有时候一个git版本库下有多个项目，需要分开来构建)
  sed -i '/include*/d' ${WORKSPACE}/settings.gradle
  echo "include $models" >> ${WORKSPACE}/settings.gradle
```


```
2. 选择Invoke Gradle Script
  Gradle Version:
    gradle-2.5
    
  Tasks:  clean war --stacktrace --debug
```

```
3. Execute Shell
  Command:
    mv ${WORKSPACE}/java_*/build/libs/*.war /opt/jenkins/war/
    ls -lt /opt/jenkins/war/
```

```    
4. Execute Shell
  Command:
  
  
#!/bin/sh

set -e

war_home=/opt/jenkins/war

date=`date +%y%m%d%M`

cd $war_home

rm -rf dockerfile

for war in $(ls);do

cat > dockerfile << EOF

FROM service/tomcat

add $war /opt/htdocs/webapp/

EOF

echo ">>> cat dockerfile <<<"

cat dockerfile

img_name="$war|sed -e 's/\.war//g;s/_/-/g'"

image_name=$(eval echo $img_name)

# echo $image_name

docker build -t="job/$image_name:$date" .

docker push job/$image_name:$date

done

docker images |grep "$date"

```


```
5. Execute Shell
  Command:
    rm -rf /opt/jenkins/war/*

```


# 构建后操作 (Post-build Actions)

```
# 构建后操作 (这里只写需要更改的)

Editable Email Notification:

Content Type: 
    HTML (text/html)

# 点击下面的 Advanced Settings..
Triggers:
    Failure - Any
        高级...
            Recipient List:
                jicki@qq.com
    
    Success
        高级...
            Recipient List:
                jicki@qq.com    

```


```
# 添加 Set build description

Regular expression:
	留空

Description:
	本次发布由 <span style="color:#E53333;"><strong>$BUILD_USER</strong></span> 发起，Git 分支<span style="color:#E53333;"><strong> $git_tag </strong></span>	

```

