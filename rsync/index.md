# Centos rsync 文件同步




Centos rsync 文件同步 

# 服务器端配置

## 安装依赖
```
yum -y install xinetd
```

CentOS默认已经安装了rsync 服务.. 输入 rsync 命令可查看是否安装.

## 编辑配置文件
vi /etc/xinetd.d/rsync

修改如下代码 disable = yes 改成 disable = no
并 启动 xinetd

```
service rsync
{
disable = yes
socket_type = stream
wait = no
user = root
server = /usr/bin/rsync
server_args = –daemon
log_on_failure += USERID
}
```

```
/etc/init.d/xinetd start 或 service xinetd start
```

防火墙打开...端口 默认端口是873 
 
 
创建 rsync 配置文件 
 
mkdir /opt/local/rsync
vi  /opt/local/rsync/rsyncd.conf （这个文件不存在自己创建）

```
#Global Settings
uid = root #以什么身份运行rsync
gid = root
use chroot = no #不使用chroot
max connections = 20 #最大连接数
secrets file = /opt/local/rsync/rsyncd.secrets #密码文件位置，认证文件设置，设置用户名和密码
log file = /opt/local/rsync/log/rsyncd.log #指定rsync的日志文件，而不将日志发送给syslog
pid file = /opt/local/rsync/rsyncd.pid #指定rsync的pid文件
lock file = /opt/local/rsync/rsync.lock #指定支持max connections参数的锁文件，默认值是/var/run/rsyncd.lock
comment = hello world
#motd file = /opt/local/rsync/rsyncd.motd #欢迎信息文件名称和存放位置（此文件没有，可以自行添加）
 
[backup] # 这里是认证的模块名，在client端需要指定
path = /data/resource # 需要做镜像的目录
auth users = rsync # 授权帐号。认证的用户名，如果没有这行，则表明是匿名，多个用户用,分隔
read only = no # yes只读 值为NO意思为可读可写模式，数据恢复用NO
hosts allow = * #允许访问的服务器IP
hosts deny = * #黑名单
list = true # 允许列文件
#ignore errors # 可以忽略一些无关的IO错误
#exclude = cache/111/ cache/222/ #忽略的目录
```

创建密码认证文件.
vi /opt/local/rsync/rsyncd.secrets

```
rsync:111111 #用户名:密碼
```

给文件正确的权限

```
# chown root:root /opt/local/rsync/rsyncd.secrets
# chmod 600  /opt/local/rsync/rsyncd.secrets  #（必须是600）
```

启动rsync  

```
rsync --daemon --config=/opt/local/rsync/rsyncd.conf
``` 
 
 
# 客户端配置


CentOS默认已经安装了rsync 服务,如果没有请执行

```
yum -y install rsync
```

创建密码认证文件

vi /data/rsync/rsyncd.pas
 
加入密码

```
rsyncpass
```

注意，客户端的密码文件只需要密码，而不需要用户名！
 
更改密码文件的权限

```
chmod 0600 /data/rsync/rsyncd.pas
```
 
执行异步同步操作

``` 
rsync -vzrtopgu --progress --delete --password-file=/data/rsync/rsyncd.pas  rsync@10.6.0.2::backup /data/resourcebak/
``` 

## rsync 参数说明

命令行中-vzrtopg 

```
v 是verbose，

z 是压缩传输，

r 是recursive，

topg 都是保持文件原有属性如属主、时间的参数。

u 是只同步已经更新的文件，避免没有更新的文件被重复更新一次，不过要注意两者机器的时钟的同步。

–progress 是指显示出详细的进度情况，

–delete 是指如果服务器端删除了这一文件，那么客户端也相应把文件删除，保持真正的一致。

后面的rsync@10.6.0.2::backup中，之后的backup是模块名， 也就是在/opt/local/rsync/rsyncd.conf中自定义的名称，rsync是指定模块中指定的可以同步的用户名。

最后的/data/resourcebak/是备份到本地的目录名。

在这里面，还可以用-e ssh的参数建立起加密的连接。
可以用–password-file=/password/path/file来指定密码文件，这样就可以在脚本中使用而无需交互式地输入验证密码了，这里需要注意的是这份密码文件权限属性要设得只有属主可读。
```

