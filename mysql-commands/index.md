# mysql 命令



# mysql 命令


##增加/删除用户。


格式：grant select on 数据库.* to 用户名@登录主机 identified by "密码"

增加一个用户test1密码为abc，让他可以在任何主机上登录，并对所有数据库有查询、插入、修改、删除的权限

```
grant select,insert,update,delete on *.* to test1@"%" Identified by "abc";
```


创建所有权限的帐号:

```
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' IDENTIFIED BY '12345678';
```

指定用户拥有创建表的权限 -  (index 为创建索引权限)

```
grant select,insert,update,delete,create,drop,index on mydb.* to test@localhost identified by "test";
```
 

删除指定用户

```
Delete FROM user Where User='test' and Host='localhost';
```


## 显示命令

```
mysql> select version();        查看MySQL的版本号

mysql> select current_date();        查看MySQL的当前日期

mysql> select version(),current_date(); 同时查看MySQL的版本号和当前日期

mysql> show processlist;   显示语句执行时间

mysqladmin -uroot -p status              查看当前连接数(Threads就是连接数.)
```
 

显示当前数据库服务器中的数据库列表：

```
mysql> SHOW DATABASES;
```


显示数据库中的数据表：

```
mysql> USE 库名；

mysql> SHOW TABLES;
```


显示数据表的结构：

```
mysql> DESCRIBE 表名;
```


建立数据库：

```
mysql> CREATE DATABASE 库名;
```


建立数据表：

```
mysql> USE 库名;

mysql> CREATE TABLE 表名 (字段名 VARCHAR(20), 字段名 CHAR(1));
```


删除数据库：

```
mysql> DROP DATABASE 库名;
```


删除数据表：

```
mysql> DROP TABLE 表名；
```


将表中记录清空：

```
mysql> DELETE FROM 表名;
```

显示表中的记录：

```
mysql> SELECT * FROM 表名;
```

插入记录：

```
mysql> INSERT INTO 表名 VALUES ("hyq","M");
```

更新表中数据：

```
mysql-> UPDATE 表名 SET 字段名1='a',字段名2='b' WHERE 字段名3='c';
```


## 运维命令


导入.sql文件命令：

```
mysql> USE 数据库名;

mysql> SOURCE /opt/mysql.sql;
```
 

命令行修改root密码：

```
mysql> UPDATE mysql.user SET password=PASSWORD('新密码') WHERE User='root';

mysql> FLUSH PRIVILEGES;
```

 
导出整个数据库

```
mysqldump -u 用户名 -p 数据库名 > 导出的文件名
```


导出一个表

```
mysqldump -u 用户名 -p 数据库名 表名> 导出的文件名
```


导出一个数据库结构

```
mysqldump -u user_name -p -d --add-drop-table database_name > outfile_name.sql
```

带语言参数导出

```
mysqldump -uroot -p --default-character-set=latin1 --set-charset=gbk --skip-opt database_name > outfile_name.sql
```

