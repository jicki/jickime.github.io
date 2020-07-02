# sersync 实时同步文件





sersync 实时同步文件


sersync 主要用于服务器同步，web镜像等功能。sersync是使用c++编写，在结合rsync同步的时候，节省了运行时耗和网络资源。因此更快。sersync配置起来很简单。另外本项目相比较其他脚本开源项目，使用多线程进行同步，尤其在同步较大文件时，能够保证多个服务器实时保持同步状态，同步及时快速。

 

## 测试环境

```
10.8.8.9
10.8.8.10 
centos 5.8 x64
```

10.8.8.9 为 sersync服务器


## 服务端配置 

安装 sersync 服务
[项目地址][1] 

下载最新64位版本

```
wget http://sersync.googlecode.com/files/sersync2.5.4_64bit_binary_stable_final.tar.gz
mkdir -p /opt/sersync
tar zxvf sersync2.5.4_64bit_binary_stable_final.tar.gz -C /opt/sersync
```

编辑配置文件

cd /opt/sersync/GNU-Linux-x86/
vi confxml.xml


修改以下部分

```
    <sersync>
        <localpath watch="/opt/htdocs/">
            <remote ip="10.8.8.10" name="system"/>
            <!--<remote ip="192.168.8.39" name="tongbu"/>-->
            <!--<remote ip="192.168.8.40" name="tongbu"/>-->
        </localpath>
```

修改好以后保存...

安装 rsync , sersync 是直接调用 rsync 服务

```
yum -y install rsync
```

启动 sersync

```
/opt/sersync/GNU-Linux-x86/sersync2 -d -o /opt/sersync/GNU-Linux-x86/confxml.xml
```


 

## 客户端配置

 
安装服务

```
yum -y install rsync
```

编辑配置文件

mkdir /opt/local/rsync
cd /opt/local/rsync/
vi  /opt/local/rsync/rsyncd.conf

```
#Global Settings
uid = root #以什么身份运行rsync
gid = root
use chroot = no #不使用chroot
max connections = 100 #最大连接数
log file = /opt/local/rsync/log/rsyncd.log #指定rsync的日志文件，而不将日志发送给syslog
pid file = /opt/local/rsync/rsyncd.pid #指定rsync的pid文件
lock file = /opt/local/rsync/rsync.lock #指定支持max connections参数的锁文件，默认值是/var/run/rsyncd.lock
comment = hello world
```
 
## rsync 配置文件说明

```
[system] # 这里是认证的模块名，在client端需要指定

path = /opt/htdocs/ # 需要做镜像的目录

read only = no # yes只读 值为NO意思为可读可写模式，数据恢复用NO

hosts allow = 10.8.8.9 10.8.8.10 #允许访问的服务器IP

hosts deny = * #黑名单

list = true # 允许列文件

ignore errors = yes # 可以忽略一些无关的IO错误
```


启动rsync 服务

```
rsync --daemon --config=/opt/local/rsync/rsyncd.conf 
```
 
  [1]: http://code.google.com/p/sersync/downloads/list

