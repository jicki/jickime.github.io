# mycat 安装配置



# mycat 


[Mycat 项目地址][1]

## 安装程序

1. Mycat 需要jdk 环境，首先安装 jdk


2. 下载 mycat server

```
tar zxvf Mycat-server-1.3.0.3-release-20150321221622-linux.tar

mv Mycat-server-1.3.0.3-release-20150321221622-linux /opt/local/mycat
```

3. 创建mycat用户，改变目录权限为mycat


```
useradd mycat
chown –R mycat:mycat /opt/local/mycat
```

4. 修改 schema.xml

```
vi /opt/local/conf/schema.xml
```
 

配置参数说明

Schema 中 主要配置 mycat 数据库 ，mysql 表 ，分片规则，分片类型
```
         <schema name="TESTDB"checkSQLschema="false" sqlMaxLimit="100">

                   <!-- auto sharding by id(long) -->

                   <tablename="travelrecord" dataNode="dn1,dn2,dn3"rule="auto-sharding-long" />
```
 

mycat 数据库 TESTDB

mysql 表 travelrecord

mysql节点dn1,dn2,dn3 

分片规则  auto-sharding-long

rule分片规则 具体在 conf/rule.xml 中定义

 
```
<dataNodename="dn1" dataHost="localhost1" database="db1"/>

<dataNodename="dn2" dataHost="localhost1" database="db2"/>

<dataNodename="dn3" dataHost="localhost1" database="db3"/>

         <dataHostname="localhost1" maxCon="1000" minCon="10"balance="0"

                   writeType="0"dbType="mysql" dbDriver="native">
```
 
以上为mysql节点 信息 

dn1 ,dn2 , dn3 为分片的mysql 节点, 既分片会存放到 3个mysql 或者群集中

db1   db2   db3 为 mysql 数据库中 三个表

 
Mysql节点 连接，用户名，密码:

```
<writeHost host="hostM1" url="127.0.0.1:3306"user="root"
                       password="123456 ">

``` 


5. 修改  /opt/local/conf/server.xml

```
<propertyname="serverPort">8066</property> <propertyname="managerPort">9066</property>

         <user name="test">

                   <propertyname="password">test</property>

                   <propertyname="schemas">TESTDB</property>

         </user>
```

serverPortMycat登录端口默认为 8066    

managerPort管理端口 默认为 9066

username 为登录mycat 用户

password 为登录 密码

schemas 为上面schema name= 中设定的 mycat 数据库名

 



6. Mysql 创建 数据库

```
CREATE database db1;
CREATEdatabase db2;
CREATE database db3;
```
 

 
7. 启动 mycat

```
/opt/local/mycat/bin/mycat start
```
 


  [1]: https://github.com/MyCATApache/Mycat-download

