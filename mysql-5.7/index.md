# centos 7 源码安装 mysql 5.7



# centos 7 源码安装 mysql 5.7


## 创建 mysql 用户以及相关目录

```
/usr/sbin/groupadd mysql

/usr/sbin/useradd -g mysql mysql

mkdir -p /opt/local/mysql/data

mkdir -p /opt/local/mysql/binlog

mkdir -p  /opt/local/mysql/logs

mkdir -p /opt/local/mysql/relaylog

mkdir -p /var/lib/mysql

mkdir -p /opt/local/mysql/etc
```
 

## 下载软件

下载 Mysql 5.7 最新版本的 tar.gz 文件

[mysql 下载地址][1]


## 安装相关依赖

```
yum -y install cmake ncurses ncurses-devel bison bison-devel boost boost-devel
```


## 编译mysql

```
tar zxvf mysql-5.7.11.tar.gz

cd mysql-5.7.11

cmake -DCMAKE_INSTALL_PREFIX="/opt/local/mysql" -DDEFAULT_CHARSET=utf8 -DMYSQL_DATADIR="/opt/local/mysql/data/" -DCMAKE_INSTALL_PREFIX="/opt/local/mysql" -DINSTALL_PLUGINDIR=plugin -DWITH_INNOBASE_STORAGE_ENGINE=1 -DDEFAULT_COLLATION=utf8_general_ci -DENABLE_DEBUG_SYNC=0 -DENABLED_LOCAL_INFILE=1 -DENABLED_PROFILING=1 -DWITH_ZLIB=system -DWITH_EXTRA_CHARSETS=none -DMYSQL_MAINTAINER_MODE=OFF -DEXTRA_CHARSETS=all -DWITH_PERFSCHEMA_STORAGE_ENGINE=1 -DWITH_MYISAM_STORAGE_ENGINE=1 -DDOWNLOAD_BOOST=1 -DWITH_BOOST=/usr/local/boost

( -DDOWNLOAD_BOOST=1 会自动下载boost 到 DWITH_BOOST= 指定目录 或者自行下载，存放于指定目录 )

make -j `cat /proc/cpuinfo | grep processor| wc -l`

make install
```
 
## 创建相关目录，授权

```
chmod +w /opt/local/mysql

chown -R mysql:mysql /opt/local/mysql

chmod +w /var/lib/mysql

chown -R mysql:mysql /var/lib/mysql

cp /opt/local/mysql/support-files/mysql.server  /etc/init.d/mysqld

chmod 755 /etc/init.d/mysqld

echo 'basedir=/opt/local/mysql/' >> /etc/init.d/mysqld

echo 'datadir=/opt/local/mysql/data' >>/etc/init.d/mysqld
```
 

 
## 创建相关链接

```
ln -s /opt/local/mysql/lib/mysql /usr/lib/mysql

ln -s /opt/local/mysql/include/mysql /usr/include/mysql

ln -s /opt/local/mysql/bin/mysql /usr/bin/mysql

ln -s /opt/local/mysql/bin/mysqldump /usr/bin/mysqldump

ln -s /opt/local/mysql/bin/myisamchk /usr/bin/myisamchk

ln -s /opt/local/mysql/bin/mysqld_safe /usr/bin/mysqld_safe

ln -s /tmp/mysql.sock /var/lib/mysql/mysql.sock
```
 
## 初始化数据库

```
rm -rf /etc/my.cnf

cp /opt/local/mysql/support-files/my-default.cnf /opt/local/mysql/etc/my.cnf

cd /opt/local/mysql/bin/

./mysqld --initialize --user=mysql --basedir=/opt/local/mysql --datadir=/opt/local/mysql/data
```
 

初始化完毕会生成一个root 的 随机密码，请务必先记录一下。如果忘记了，请查看 (mysqld.log)



## 启动数据库

```
service mysqld start
```

## 数据库安全设置

```
/opt/local/mysql/bin/mysql_secure_installation -uroot -p
```

  [1]: ftp://ftp.mirrorservice.org/sites/ftp.mysql.com/Downloads/MySQL-5.7/

