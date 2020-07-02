# MariaDB GTID 复制同步



# MariaDB GTID 复制同步

## 说明

**GTID：Global Transaction ID,全局事务ID，在整个主从复制架构中任何两个事物ID是不能相同的。全局事务ID是Mster服务器生成一个128位的UUID+事物的ID号组成的，UUID标示主服务器的身份，此UUID在整个主从复制架构中是绝对唯一，而且即使更换主服务器后UUID也不会改变而是继承当前主服务器的UUID身份。**

 

## 环境说明

```
master  IP ： 10.6.0.96
slave   IP :  10.6.0.138
```

配置本地hosts

```
vim /etc/hosts

10.6.0.96 master.mysql

10.6.0.138 slave.mysql
```
 

## 确认配置文件


```
binlog_format                         二进制日志的格式，有row、statement和mixed几种类型；

log-slave-updates、

report-port

report-host：                         用于启动GTID及满足附属的其它需求；

master-info-repository

relay-log-info-repository             启用此两项，可用于实现在崩溃时保证二进制及从服务器安全的功能；

sync-master-info                      启用之可确保无信息丢失；

slave-parallel-workers                设定从服务器的SQL线程数；0表示关闭多线程复制功能；

binlog-checksum

master-verify-checksum

slave-sql-verify-checksum             启用复制有关的所有校验功能；

binlog-rows-query-log-events          启用之可用于在二进制日志记录事件相关的信息，可降低故障排除的复杂度；

log-bin                               启用二进制日志，这是保证复制功能的基本前提；

server-id                             同一个复制拓扑中的所有服务器的id号必须惟一.
```
 


## master 配置


//修改如下内容

```
log-bin=/opt/local/mysql/binlog/mysql-bin          #二进制日志文件目录

server-id       = 1                                #从服务器不能跟此id重复

binlog_format=ROW                                  #二进制日志文件格式

innodb_file_per_table=1                            #innodb表空间独立

log-slave-updates=true                             #从master取得并执行的二进制日志写入自己的二进制日志文件中
```


//添加以下内容

```
binlog-do-db=mysql                             #指定只同步mysql库                          

master-info-repository=TABLE                       #用于实现在崩溃时保证二进制及从服务器安全的功能；

relay-log-info-repository=TABLE                    #用于实现在崩溃时保证二进制及从服务器安全的功能；

sync-master-info=1                                 #启用之可确保无信息丢失

slave-parallel-threads=2                           #设定从服务器的SQL线程数；0表示关闭多线程复制功能

binlog-checksum=CRC32                              #启用复制有关的所有校验功能

master-verify-checksum=1                           #启用复制有关的所有校验功能

slave-sql-verify-checksum=1                        #启用复制有关的所有校验功能

binlog-rows-query-log_events=1                     #启用之可用于在二进制日志记录事件相关的信息，可降低故障排除的复杂度；

report-host=master.mysql                           #master 主机名，必须能ping通

report-port=3306                                   #端口
```
 
重启 mysql

```
service mysqld restart
```
 
创建同步帐号

```
mysql -uroot -p

grant replication slave,replication client on *.* to "rep"@'10.6.0.138' identified by 'rep12345';

flush privileges;
```
 

## slave 配置

 
//修改如下内容

```
log-bin=/opt/local/mysql/binlog/mysql-bin          #二进制日志文件目录

server-id       = 10                               #从服务器不能跟此id重复

binlog_format=ROW                                  #二进制日志文件格式

innodb_file_per_table=1                            #innodb表空间独立

log-slave-updates=true                             #从master取得并执行的二进制日志写入自己的二进制日志文件中

relay-log=/opt/local/mysql/relaylog/s74-relay-bin  
```
 

//添加以下内容

```
replicate-do-db=mysql                           #指定只同步mysql库

master-info-repository=TABLE                       #用于实现在崩溃时保证二进制及从服务器安全的功能；

relay-log-info-repository=TABLE                    #用于实现在崩溃时保证二进制及从服务器安全的功能；

sync-master-info=1                                 #启用之可确保无信息丢失

slave-parallel-threads=2                           #设定从服务器的SQL线程数；0表示关闭多线程复制功能

binlog-checksum=CRC32                              #启用复制有关的所有校验功能

master-verify-checksum=1                           #启用复制有关的所有校验功能

slave-sql-verify-checksum=1                        #启用复制有关的所有校验功能

binlog-rows-query-log_events=1                     #启用之可用于在二进制日志记录事件相关的信息，可降低故障排除的复杂度；

report-host=slave.mysql                            #slave 主机名，必须能ping通

report-port=3306                                   #端口
```
 

在slave服务器使用主mysql上创建的账号密码登陆

 
```
mysql -uroot -p 

change master to master_host='10.6.0.96',master_user='rep',master_password='rep12345',master_use_gtid=current_pos;

start slave;
```
 

查看是否启用 gtid:

```
show processlist;
```
 

## 检查同步状态：

 
```
show slave status\G;
```
 



## Position 复制

```
show master status;      
```

记录 Position

```
+------------------+----------+--------------+------------------+

| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB |

+------------------+----------+--------------+------------------+

| mysql-bin.000008 |     1062 |              |                  |

+------------------+----------+--------------+------------------+
```
 
在 slave 上面 执行

```
CHANGE MASTER TO MASTER_HOST='10.6.0.96',MASTER_USER='rep',MASTER_PASSWORD='rep12345',MASTER_LOG_FILE='mysql-bin.000008',MASTER_LOG_POS=1062;

start slave;
```

查看状态

```
show slave status \G;
```

