# tomcat 优化配置 以及说明




# tomcat 优化配置 以及说明


## 并发优化
 

JVM调优

以下为1G物理内存tomcat配置: 

编辑 catalina.sh 文件

```
JAVA_OPTS="-server -Xms512M -Xmx512M -Xss256K"
```


参数说明

```
-server:                一定要作为第一个参数，在多个CPU时性能佳

-Xms：                  初始Heap大小，使用的最小内存,cpu性能高时此值应设的大一些

-Xmx：                  java heap最大值，使用的最大内存

-Xms 与 -Xmx  两个值是分配JVM的最小和最大内存，取决于硬件物理内存的大小，建议均设为物理内存的一半。

-Xss：                  每个线程的Stack大小
```
 


以下为32G物理内存tomcat配置: 

```
JAVA_OPTS="-server -Xms20480m -Xmx20480m -Xss1024K"
```



## 开启 apr 模式


安装apr 以及 tomcat-native

```
yum -y install apr apr-devel
```

进入tomcat/bin目录，比如

```
cd /opt/local/tomcat/bin/
tar xzfv tomcat-native.tar.gz
cd tomcat-native-1.1.32-src/jni/native/
./configure --with-apr=/usr/bin/apr-1-config
make && make install
```

安装成功后还需要对tomcat设置环境变量，方法是在catalina.sh文件中增加1行：

```
CATALINA_OPTS="-Djava.library.path=/usr/local/apr/lib"
```
 
修改8080端对应的conf/server.xml

查找  protocol="org.apache.coyote.http11.Http11AprProtocol"


```
     <Connector executor="tomcatThreadPool"

               port="8080" 

               protocol="org.apache.coyote.http11.Http11AprProtocol"

               connectionTimeout="20000"

               enableLookups="false"

               redirectPort="8443"

               URIEncoding="UTF-8" />
```


PS:启动以后查看日志 显示如下表示开启 apr 模式

```
INFO: APR capabilities: IPv6 [true], sendfile [true], accept filters [false], random [true].
```


## FAQ

日志乱码的解决办法

编辑 catalina.sh 文件

```
JAVA_OPTS="$JAVA_OPTS -Djavax.servlet.request.encoding=UTF-8 -Dfile.encoding=UTF-8 -Duser.timezone=GMT+8"
```

