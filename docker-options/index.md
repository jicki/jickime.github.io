# docker 基础设置



# docker 基础



## 升级内核

```
# 导入 Key

rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org



# 安装 Yum 源

rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm



# 更新 kernel

yum --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel 


# 配置 内核优先

grub2-set-default 0

```



## 开启内核namespace支持

```
# 执行如下
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"



# 必须重启系统

reboot

```



## 修改内核参数

```
cat<<EOF > /etc/sysctl.d/docker.conf
# 要求iptables不对bridge的数据进行处理
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
EOF



# 生效配置
sysctl --system
```



```
# 检查系统

curl -s https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh | bash

```

## docker install

```
# 指定安装,并指定安装源

export VERSION=19.03
curl -fsSL "https://get.docker.com/" | bash -s -- --mirror Aliyun

```



## docker 设置


```
# 第一种方式, 增加daemon.json


mkdir -p /etc/docker/
cat>/etc/docker/daemon.json<<EOF
{
  "bip": "172.17.0.1/16",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://dockerhub.azk8s.cn","https://gcr.azk8s.cn","https://quay.azk8s.cn"],
  "data-root": "/opt/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  }
}
EOF
```


```
# kubernetes docker

mkdir -p /etc/docker/
cat>/etc/docker/daemon.json<<EOF
{ 
  "bip": "172.17.0.1/16",
  "exec-opts": ["native.cgroupdriver=systemd"],
  "registry-mirrors": ["https://dockerhub.azk8s.cn","https://gcr.azk8s.cn","https://quay.azk8s.cn"],
  "data-root": "/opt/docker",
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "5"
  },
  "dns-search": ["default.svc.cluster.local", "svc.cluster.local", "localdomain"],
  "dns-opts": ["ndots:2", "timeout:2", "attempts:2"]
}
EOF
```


```
# 第二种方式，增加 opts

mkdir -p /etc/systemd/system/docker.service.d/


# 增加 docker.service 文件

cat >> /etc/systemd/system/docker.service << EOF

[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target docker-storage-setup.service
Wants=docker-storage-setup.service

[Service]
Type=notify
Environment=GOTRACEBACK=crash
ExecReload=/bin/kill -s HUP $MAINPID
Delegate=yes
KillMode=process
ExecStart=/usr/bin/dockerd \
          $DOCKER_OPTS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $DOCKER_DNS_OPTIONS \
          $INSECURE_REGISTRY
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=1min
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF



# 增加配置文件

cat >> /etc/systemd/system/docker.service.d/docker-options.conf << EOF
[Service]
Environment="DOCKER_OPTS=--insecure-registry=10.254.0.0/16 \
    --registry-mirror=https://registry.docker-cn.com \
    --exec-opt native.cgroupdriver=systemd \
    --data-root=/opt/docker --log-opt max-size=50m --log-opt max-file=5"
EOF

```

