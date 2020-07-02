# tomcat 利用 cronolog 进行日志切割



# tomcat 利用 cronolog 切割日志


## 安装 软件

cronolog官网
http://cronolog.org/

```
wget http://cronolog.org/download/cronolog-1.6.2.tar.gz
cronolog-1.6.2.tar.gz
tar zxvf cronolog-1.6.2.tar.gz
cd cronolog-1.6.2
./configure && make && make install
```
 

## 配置 tomcat 


1. 编辑 catalina.sh 文件

查找

```
if [ -z "$CATALINA_OUT" ] ; then

  CATALINA_OUT=/opt/htdocs/logs/catalina.out

fi
```
 
修改为

```
if [ -z "$CATALINA_OUT" ] ; then

  CATALINA_OUT=/opt/htdocs/logs/catalina.%Y-%m-%d.out

fi
```

查找   

```
touch "$CATALINA_OUT"
```

修改为

```
#touch "$CATALINA_OUT"
```
  

查找 

```
"$CATALINA_OUT" 2>&1 "&"
```

有两处

```
      org.apache.catalina.startup.Bootstrap "$@" start \

      >> "$CATALINA_OUT" 2>&1 "&"
```
 

都修改

 ```

      org.apache.catalina.startup.Bootstrap "$@" start \

      | /usr/local/sbin/cronolog "$CATALINA_OUT" >> /dev/null &
```
 
重启 tomcat 服务，查看日志文件

