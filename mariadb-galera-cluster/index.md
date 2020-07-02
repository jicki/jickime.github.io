# Mariadb galera 群集




# Mariadb galera 群集

## 环境说明:

```
CentOS 7 x64  *  3
IP : 192.168.0.100
IP : 192.168.0.101
IP : 192.168.0.102
```
 

## 安装相关软件

配置mariadb YUM 源

这里 mariadb-galera 使用 源码安装，其他使用yum 安装

http://yum.mariadb.org/10.0.15/rhel7-amd64/rpms/
下载软件 MariaDB-Galera-10.0.15
 
```
wget http://mirrors.neusoft.edu.cn/mariadb/mariadb-galera-10.0.15/source/mariadb-galera-10.0.15.tar.gz
```
 
创建yum 文件

vi /etc/yum.repos.d/mariadb.repo

```
[mariadb]
name = MariaDB
baseurl = http://yum.mariadb.org/10.0.15/centos7-amd64/
enabled = 1
gpgkey=https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

安装依赖软件

```
yum makecache
yum -y install galera MariaDB-client socat cmake lsof perl-DBD-MySQL perl-Digest-MD5 perl-Time-HiRes rsync libev zlib-devel
```

percona-xtrabackup 同步软件，请使用2.2.x 版本

```
rpm -ivh percona-xtrabackup-2.2.8-5059.el7.x86_64.rpm
```

 
安装 mariadb

```
/usr/sbin/groupadd mysql
/usr/sbin/useradd -g mysql mysql
mkdir -p /opt/local/mysql/data
mkdir -p /opt/local/mysql/binlog
mkdir -p  /opt/local/mysql/logs
mkdir -p /opt/local/mysql/relaylog
mkdir -p /var/lib/mysql
mkdir -p /opt/local/mysql/wsrep

tar zxvf mariadb-galera-10.0.15.tar.gz
cd mariadb-10.0.15
cmake -DCMAKE_INSTALL_PREFIX="/opt/local/mysql" -DDEFAULT_CHARSET=utf8 -DMYSQL_DATADIR="/opt/local/mysql/data/" -DCMAKE_INSTALL_PREFIX="/opt/local/mysql" -DINSTALL_PLUGINDIR=plugin -DWITH_INNOBASE_STORAGE_ENGINE=1 -DDEFAULT_COLLATION=utf8_general_ci -DENABLE_DEBUG_SYNC=0 -DENABLED_LOCAL_INFILE=1 -DENABLED_PROFILING=1 -DWITH_ZLIB=system -DWITH_EXTRA_CHARSETS=none -DMYSQL_MAINTAINER_MODE=OFF -DEXTRA_CHARSETS=all -DWITH_FAST_MUTEXES=ON -DWITH_PERFSCHEMA_STORAGE_ENGINE=1 -DWITH_MYISAM_STORAGE_ENGINE=1

make -j `cat /proc/cpuinfo | grep processor| wc -l`
make install
```

授权，并初始化数据库

```
chmod +w /opt/local/mysql
chown -R mysql:mysql /opt/local/mysql
chmod +w /var/lib/mysql
chown -R mysql:mysql /var/lib/mysql
./scripts/mysql_install_db --defaults-file=/opt/local/mysql/my.cnf --basedir=/opt/local/mysql --datadir=/opt/local/mysql/data --user=mysql
cp ./support-files/mysql.server  /etc/init.d/mysqld
chmod 755 /etc/init.d/mysqld
echo 'basedir=/opt/local/mysql/' >> /etc/init.d/mysqld
echo 'datadir=/opt/local/mysql/data' >>/etc/init.d/mysqld
```


启动服务

```
service mysqld start
chkconfig mysqld on
```


创建链接

```
ln -s /opt/local/mysql/lib/mysql /usr/lib/mysql
ln -s /opt/local/mysql/include/mysql /usr/include/mysql
ln -s /opt/local/mysql/bin/mysql /usr/bin/mysql
ln -s /opt/local/mysql/bin/mysqldump /usr/bin/mysqldump
ln -s /opt/local/mysql/bin/myisamchk /usr/bin/myisamchk
ln -s /opt/local/mysql/bin/mysqld_safe /usr/bin/mysqld_safe
ln -s /tmp/mysql.sock /var/lib/mysql/mysql.sock
```

设置密码，并加固数据库安全

```
/opt/local/mysql/bin/mysqladmin -u root password 'rldb123'
/opt/local/mysql/bin/mysql_secure_installation
```

 

授权用于集群同步的用户和密码

```
/opt/local/mysql/bin/mysql -uroot -p
GRANT ALL PRIVILEGES ON *.* TO 'sst'@'localhost' IDENTIFIED BY 'rldb123';
GRANT ALL PRIVILEGES ON *.* TO 'sst'@'同步ip' IDENTIFIED BY 'rldb123';
FLUSH PRIVILEGES;
```

 

配置wsrep.cnf文件

```
cp /opt/local/mysql/support-files/wsrep.cnf /opt/local/mysql/wsrep
```

```
vi /opt/local/mysql/wsrep/wsrep.cnf
```

修改如下

```
wsrep_provider=/usr/lib64/galera/libgalera_smm.so
wsrep_cluster_address="gcomm://"
wsrep_sst_auth=sst:123456
wsrep_sst_method=xtrabackup
wsrep_node_address=192.168.0.100
```
 
注意:

```
"gcomm://" 是特殊的地址,仅仅是Galera cluster初始化启动时候使用。
如果集群启动以后，我们关闭了第一个节点，那么再次启动的时候必须先修改，"gcomm://"为其他节点的集群地址
例如 wsrep_cluster_address="gcomm://192.168.0.101:4567"
```
 
修改 mairadb 配置文件

```
vi /opt/local/mysql/my.cnf
```

在[client] 下面 增加

```
!includedir /opt/local/mysql/wsrep
```

配置完成以后，重启mariadb

```
service mysqld restart
```

```
netstat -tulpn |grep -e 4567 -e 3306
```
 
登录mysql 查看 global 的 status 

```
/opt/local/mysql/bin/mysql -uroot -p
```

```
show global status like '%state%';
```


接下来就是 其他节点的添加！  步骤大致相同

构造新节点的操作步骤如下:

wsrep_cluster_address的配置不一样:

```
wsrep_cluster_address="gcomm://Node-A-IP,Node-B-IP"   # 这里指向是指上一层的集群地址
```

重起MariaDB

```
service mysqld restart
```

所有节点配置完成以后,更改节点1的 wsrep_cluster_address="gcomm://" 为其他IP
 

加入 仲裁人 (可选)
"仲裁人"节点上没有数据,它在集群中的作用就是在集群发生分裂时进行仲裁，集群中可以有多个"仲裁人"节点。

"仲裁人"节点加入集群的方法如下:

```
garbd -a gcomm://192.168.0.100:4567 -g my_wsrep_cluster -d
```
注释：参数说明: -d：以daemon模式运行 -a：集群地址 -g： 集群名称

 
检查集群状况：

```
SHOW VARIABLES LIKE 'wsrep_cluster_address';
show status like 'wsrep%';
```

 


## FAQ 

配置相同，出现无法启动同步的时候，通常都是因为权限问题。

所有节点都执行授权.

```
chmod +w /opt/local/mysql

chown -R mysql:mysql /opt/local/mysql
```

然后再启动同步..
启动完成以后，再次执行授权命令..
通常是因为 data 目录下面的 ib_logfile0 ib_logfile1 ib_logfile2 为root 权限 所以不能同步，导致的。

