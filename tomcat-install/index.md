# tomcat jdk 安装





> 简单记录一下...虽然安装很简单...

 
# Java JDK


```
首先下载配置安装 jdk 


http://www.oracle.com/technetwork/java/javase/downloads/jdk-6u25-download-346242.html


# 下载bin 或者 rpm 的安装  （需要注册个帐号）


# 32位

http://download.oracle.com/otn/java/jdk/8u25-b06/jdk-8u25-linux-i586-rpm.bin

 
# 64位

http://download.oracle.com/otn/java/jdk/8u25-b06/jdk-8u25-linux-x64-rpm.bin


chmod 777 jdk-8u25-linux-x64-rpm.bin


./jdk-8u25-linux-x64-rpm.bin
``` 

 
## 配置 jdk ENV


```
#安装完成以后...安装目录为 /usr/java/jdk1.8.0_25/

#接下来配置一下JDK的环境..


vi /etc/profile

#在最下面增加下面三行：


export JAVA_HOME=/usr/java/jdk1.8.0_25/

export PATH=$JAVA_HOME/bin:$JAVA_HOME/jre/bin:$PATH

export CLASSPATH=.:/usr/java/jdk1.8.0_25/lib:/usr/java/jdk1.8.0_25/jre/lib:$CLASSPATH

 

# 使设置生效:

source /etc/profile

```
 
## 验证 java
 
 
```
java -version

-----------------------------------------------------------------------

java version "1.8.0_25"

Java(TM) SE Runtime Environment (build 1.8.0_25-b06)

Java HotSpot(TM) Client VM (build 20.12-b01, mixed mode, sharing)

-----------------------------------------------------------------------

```

 

 

 

# tomcat

 
```
#下载最新稳定版本:


http://tomcat.apache.org/index.html

 
http://apache.dataguru.cn/tomcat/tomcat-7/v7.0.34/bin/apache-tomcat-7.0.34.tar.gz
```

 
## 安装

```
tar zxvf apache-tomcat-7.0.34.tar.gz


mv apache-tomcat-7.0.34 /opt/local/tomcat

```
 

## 配置tomcat环境


```
vi /etc/profile


#下面添加

export TOMCAT_HOME=/opt/local/tomcat/

 
#使设置生效

source /etc/profile
 

cd /opt/local/tomcat/bin/


chmod 777 *.sh

```


## 启动 Tomcat


```

/bin/bash catalina.sh start
```
 
## 测试

```
#访问

http://ip:8080 

```


## 配置JMX

```
vi catalina.sh   

# 在 CATALINA_OPTS 里面添加如下配置

CATALINA_OPTS="-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.local.only=false -Dcom.sun.management.jmxremote.authenticat
e=false -Dcom.sun.management.jmxremote.port=9999 -Dcom.sun.management.jmxremote.rmi.port=9999 -Djava.rmi.server.hostname=172.16.1.47 -Dcom.sun.management.jmxremote.ssl=false"


# 特别注意 -Djava.rmi.server.hostname=172.16.1.47

# -Djava.rmi.server.hostname=  在 docker 下必须填写 宿主机IP 否则连接不上


配置完毕 重启 tomcat


Windows 下安装 jdk 

在 java_home/bin/ 下面 找到 jconsole.exe 

选择远程连接， 输入 IP 与 端口 既可连接


```

