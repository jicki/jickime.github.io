# Kata to docker 集成



# Kata 简介

![图1][1]

* Katacontainer 是 OpenStack 基金会于 2017 KubeCon 峰会上正式发布，在2018年5月份 OpenStack 温哥华峰会上对外发布1.0版本，并且在那届峰会上还有好几个关于 katacontainer 的演讲。我对 KataContainers 的具体实现原理不清楚，只知道它是一个轻量虚拟机实现，可以无缝地与容器生态系统(实现 OCI 接口)进行集成。
* katacontainer(包括 Google 的 gVisor)的主要优势是能够对接遗留系统以及提供比传统容器更好的安全性。我在本文后面的实验也可以证明，katacontainer 可以与传统的 runc 并存，为不同性质的应用负载提供了强大的灵活性。


# 安装 Docker


```
# 安装 yum-config-manager
yum -y install yum-utils

# 导入
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo


# 更新 repo
yum makecache


# 安装 docker
yum -y install docker-ce

```


## 配置 docker

```
# 添加配置

vi /etc/systemd/system/docker.service

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
```


```
# 创建额外配置目录

mkdir -p /etc/systemd/system/docker.service.d/

```

```
# 添加额外配置

vi /etc/systemd/system/docker.service.d/docker-options.conf

[Service]
Environment="DOCKER_OPTS=\
    --registry-mirror=http://b438f72b.m.daocloud.io \
    --exec-opt native.cgroupdriver=systemd \
    --data-root=/opt/docker --log-opt max-size=50m --log-opt max-file=5"
```


```

# 重新读取配置，启动 docker, 并设置自动启动 

systemctl daemon-reload
systemctl start docker
systemctl enable docker

```


```
# 验证 安装

docker info
Client:
 Debug Mode: false

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 19.03.2
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
 Logging Driver: json-file
 Cgroup Driver: systemd
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 894b81a4b802e4eb2a91d1ce216b8817763c29fb
 runc version: 425e105d5a03fabd737a126ad93d62a9eeede87f
 init version: fec3683
 Security Options:
  seccomp
   Profile: default
 Kernel Version: 4.4.194-1.el7.elrepo.x86_64
 Operating System: CentOS Linux 7 (Core)
 OSType: linux
 Architecture: x86_64
 CPUs: 2
 Total Memory: 3.859GiB
 Name: localhost
 ID: 4JQ2:WFL6:YSBJ:TFS4:E5H4:3TGD:2CY4:FRYC:7DCC:5RX2:E5SO:MJQW
 Docker Root Dir: /opt/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Registry Mirrors:
  http://b438f72b.m.daocloud.io/
 Live Restore Enabled: false
```



# 安装 Kata


```
# 下载 yum 源
curl http://download.opensuse.org/repositories/home:/katacontainers:/releases:/x86_64:/master/CentOS_7/home:katacontainers:releases:x86_64:master.repo -o /etc/yum.repos.d/katacontainers.repo


# 更新yum 缓存

yum makecache

# 安装 kata

yum -y install kata-runtime kata-proxy kata-shim


```




```
# 检查安装,并确认硬件支持

kata-runtime kata-check


# 输出 最后一句 System can currently create Kata Containers 证明可以正常支持。

```


## 配置 kata 

```
# 拷贝 kata 配置文件

cp /usr/share/defaults/kata-containers/configuration.toml /etc/kata-containers/configuration.toml

```



```
# 添加一个 config 配置 docker 运行 kata
# 这里配置 默认还是使用 docker 的 runc 作为容器 也可以将 --default-runtime 配置为 kata 
# 运行容器的时候使用 --runtime kata-runtime 指定 使用 kata-runntime 
# 如: docker run -d --name kata-nginx -p 80:80 --runtime kata-runtime nginx:alpine


vi /etc/systemd/system/docker.service.d/kata-containers.conf

[Service]
Type=simple
ExecStart=
ExecStart=/usr/bin/dockerd -D --default-runtime runc --add-runtime kata-runtime=/usr/bin/kata-runtime

```

```
# 生效配置,重启 docker

systemctl daemon-reload
systemctl restart docker
```


## 测试 kata

```
# 启动容器
docker run -d --name kata-nginx -p 80:80 --runtime kata-runtime nginx:alpine



# 查看运行的 kata-runntime

kata-runtime list

```

  [1]: https://jicki.cn/img/posts/kata/kata.png

