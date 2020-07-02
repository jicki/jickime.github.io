# openshift origin 3.7



# Openshift Origin 3


## 环境说明

|类型|主机名|IP|系统|内核|
|:---:|:---:|:---:|:---:|:---|
|Master|ops-master-64|172.16.1.64|CentOS 7.4|Kernel 4.4.x|
|Master|ops-master-65|172.16.1.65|CentOS 7.4|Kernel 4.4.x|
|Node|ops-node-66|172.16.1.66|CentOS 7.4|Kernel 4.4.x|


## 系统要求

|类型|CPU|内存|空间|系统|
|:--:|:--:|:--:|:--:|:--:|
|Master|2vCPU|8G|40G|CentOS 7.2 以上|
|Node|1vCPU|8G|20G|CentOS 7.2 以上|



## 初始化环境


```
hostnamectl --static set-hostname hostname

ops-master-64: 172.16.1.64
ops-master-65: 172.16.1.65
ops-node-64:   172.16.1.66

```


```
# 配置 hostname 通信

vi /etc/hosts

# OpenShift hostname 
172.16.1.64 ops-master-64
172.16.1.65 ops-master-65
172.16.1.66 ops-node-66
# OpenShift hostname

```


```
# 启用 NetworkManager

systemctl start NetworkManager
systemctl enable NetworkManager

```

```
# 停止, 禁用防火墙

systemctl stop firewalld
systemctl diable firewalld

```


```
# 清除之前安装的 dnsmasq

yum remove -y dnsmasq

```


```
# 清除之前安装的 docker

yum remove -y docker-engine-selinux  docker-engine  docker-ce

```



```
# 安装 iptables

yum -y install iptables iptables-services



# 配置 iptables，特别重要的是 ssh 非 22 端口的服务器

iptables-save > /etc/sysconfig/iptables

vi /etc/sysconfig/iptables

# sample configuration for iptables service
# you can edit this manually or use system-config-firewall
# please do not ask us to add additional ports/services to this default configuration
*filter
:INPUT ACCEPT [0:0]
:FORWARD ACCEPT [0:0]
:OUTPUT ACCEPT [0:0]
-A INPUT -m state --state RELATED,ESTABLISHED -j ACCEPT
-A INPUT -p icmp -j ACCEPT
-A INPUT -i lo -j ACCEPT
# 需要开放的端口写法
-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT
# 需要开放的端口写法
-A INPUT -j REJECT --reject-with icmp-host-prohibited
-A FORWARD -j REJECT --reject-with icmp-host-prohibited
COMMIT



# 使配置生效

iptables-restore /etc/sysconfig/iptables


# 启动 iptables

systemctl start iptables
systemctl enable iptables

```


```
# 配置 selinux  安装时默认会打开


vi /etc/selinux/config 


SELINUX=permissive
SELINUXTYPE=targeted

```




## 安装所需依赖

> 在所有节点安装所需依赖

```
yum install -y wget git net-tools bind-utils iptables-services bridge-utils bash-completion python-yaml centos-release-openshift-origin dnsmasq

```



# 安装docker

## yum 安装 docker

```
# 使用官方 yum 原生配置

yum -y install docker

```


## 更改docker 配置


```
# 编辑如下配置

vi /etc/sysconfig/docker

# OPTIONS 修改为如下配置

OPTIONS='--insecure-registry=10.254.0.0/16 --graph=/opt/docker --registry-mirror=http://b438f72b.m.daocloud.io --log-opt max-size=50m --log-opt max-file=5'



# 编辑文件系统 kernel 3.10  设置为 overlay , 4.0 以上设置为 overlay2

vi /etc/sysconfig/docker-storage

DOCKER_STORAGE_OPTIONS="--storage-driver overlay2"

```

## 启动 docker

```
# 重新读取配置，启动 docker
systemctl daemon-reload
systemctl start docker
systemctl enable docker

```


# 安装 etcd 集群


## 安装服务

```
# yum 安装 etcd

yum -y install etcd

```


```
# 启动服务

systemctl enable etcd

systemctl start etcd

systemctl status etcd

```


```

# 如果报错 请使用
journalctl -f -t etcd  和 journalctl -u etcd 来定位问题

```




# 安装 ansible 


```
# 安装 centos 额外的yum源
yum install -y epel-release

# make 缓存
yum clean all && yum makecache

# 安装 软件
yum install -y python-pip python34 python-netaddr python34-pip ansible pyOpenSSL

```

## 配置SSH Key 登陆

```
# 确保本机也可以 ssh 连接，否则下面部署失败

ssh-keygen -t rsa -N ""

ssh-copy-id -i /root/.ssh/id_rsa.pub 172.16.1.65

ssh-copy-id -i /root/.ssh/id_rsa.pub 172.16.1.66

```


# 下载 OpenShift

> 官方有 github 里有 openshift-ansible https://github.com/openshift/openshift-ansible



```
# 下载最新 releases 版本

cd /opt

[root@ops-master-64 opt]# git clone https://github.com/openshift/openshift-ansible
正克隆到 'openshift-ansible'...
remote: Counting objects: 86508, done.
remote: Compressing objects: 100% (12/12), done.
remote: Total 86508 (delta 3), reused 9 (delta 0), pack-reused 86494
接收对象中: 100% (86508/86508), 21.95 MiB | 152.00 KiB/s, done.
处理 delta 中: 100% (52747/52747), done.
```

## 配置参数


```
# 目录 openshift-ansible/inventory

# 包含如下文件

[root@ops-master-64 inventory]# ls
hosts.example                     hosts.glusterfs.mixed.example   hosts.glusterfs.registry-only.example         hosts.openstack
hosts.glusterfs.external.example  hosts.glusterfs.native.example  hosts.glusterfs.storage-and-registry.example  README.md


```


```
# 编辑配置文件 hosts.example 里面有完整的参数可供参考

# 新建一个 新的

[root@ops-master-64 inventory]# vi hosts



# This is an example of an OpenShift-Ansible host inventory

[OSEv3:children]
masters
nodes
etcd

# Set variables common for all OSEv3 hosts
[OSEv3:vars]

# SSH 时使用的用户
ansible_user=root

# 日志级别，0 是指记录错误与警告，默认是2
debug_level=0

# 安装版本 origin = 开源版  openshift-enterprise = 企业版
openshift_deployment_type=origin

# 安装版本
openshift_release=v3.7

# 跳过如下检测
openshift_disable_check=docker_storage,memory_availability,docker_image_availability

# htpasswd auth
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider', 'filename': '/etc/origin/master/htpasswd'}]

# host group for masters
[all]
ops-master-64 ansible_port=99
ops-master-65 ansible_port=99
ops-node-66   ansible_port=99


[masters]
ops-master-64
ops-master-65

[etcd]
ops-master-64
ops-master-65
ops-node-66

[nodes]
ops-master-64
ops-master-65
ops-node-66 openshift_node_labels="{'region': 'primary', 'zone': 'default'}"

```


# 部署 Openshift


## 初始化安装环境

```
# 首先清理一次环境

cd /opt/openshift-ansible

ansible-playbook -i inventory/hosts playbooks/adhoc/uninstall.yml -b -v --private-key=~/.ssh/id_rsa



PLAY [lb] ****************************************************************************************************************************************************************
skipping: no hosts matched

PLAY RECAP ***************************************************************************************************************************************************************
ops-master-64              : ok=24   changed=6    unreachable=0    failed=0   
ops-master-65              : ok=57   changed=10   unreachable=0    failed=0   
ops-node-66                : ok=45   changed=9    unreachable=0    failed=0   


```



## 开始部署集群

```
cd /opt/openshift-ansible

ansible-playbook -i inventory/hosts playbooks/deploy_cluster.yml -b -v --private-key=~/.ssh/id_rsa



```

