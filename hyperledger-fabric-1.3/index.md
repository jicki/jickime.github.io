# hyperledger-fabric v 1.3


> fabric v1.3 , 单机 多节点 kafka 手动部署, 所有服务均 开启 SSL 认证。



# 部署 hyperledger-fabric v1.3



## 环境规划

> 相关hostname 必须配置 dns 
>
> 关于 orderer 集群
>
> 当orderer 向peer节点提交Transaction的时候，peer节点会得到或返回一个读写集结果，该结果会发送给orderer节点进行共识和排序，此时如果orderer节点突然down掉，就会使请求服务失效而引发的数据丢失等问题，且目前的sdk对orderer发送的Transaction的回调会占用极长的时间，当大批量数据导入的时候该回调可认为不可用。

|节点标识|hostname|IP|开放端口|系统|
|--------|--------|--|--------|----------|---|--|
|orderer0节点|orderer0.jicki.cn|192.168.100.100|7050|CentOS 7 x64|
|orderer1节点|orderer1.jicki.cn|192.168.100.100|7050|CentOS 7 x64|
|orderer2节点|orderer2.jicki.cn|192.168.100.100|7050|CentOS 7 x64|
|peer0节点|peer0.org1.jicki.cn|192.168.100.100|7051, 7052, 7053|CentOS 7 x64|
|peer0节点|peer0.org2.jicki.cn|192.168.100.100|7051, 7052, 7053|CentOS 7 x64|
|zk0节点|zookeeper0|192.168.100.100|2181|CentOS 7 x64|
|zk1节点|zookeeper1|192.168.100.100|2181|CentOS 7 x64|
|zk2节点|zookeeper2|192.168.100.100|2181|CentOS 7 x64|
|kafka0节点|kafka0|192.168.100.100|9092|CentOS 7 x64|
|kafka1节点|kafka1|192.168.100.100|9092|CentOS 7 x64|
|kafka2节点|kafka2|192.168.100.100|9092|CentOS 7 x64|




```
# 配置一下 hosts 后面需要调用


# fabric

192.168.168.100 orderer0.jicki.cn
192.168.168.100 orderer1.jicki.cn
192.168.168.100 orderer2.jicki.cn
192.168.168.100 peer0.org1.jicki.cn
192.168.168.100 peer0.org2.jicki.cn

# fabric


```


## 官方地址

> 文档以官方文档为主 http://hyperledger-fabric.readthedocs.io/en/release-1.3/prereqs.html

```
# 官网 github
https://github.com/hyperledger/fabric
```

## 环境准备

> 所有机器 安装 Docker (用于 fabric 服务启动) 



```
# 导入 yum 源

# 安装 yum-config-manager

yum -y install yum-utils

# 导入
yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
    
    
# 安装 docker

yum -y install docker-ce
    
```


```

# 启动 docker 

systemctl daemon-reload
systemctl start docker
systemctl enable docker

```


```
# 查看 docker 版本

docker version
Client:
 Version:           18.06.1-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        e68fc7a
 Built:             Tue Aug 21 17:23:03 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.1-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       e68fc7a
  Built:            Tue Aug 21 17:25:29 2018
  OS/Arch:          linux/amd64
  Experimental:     false
```




* 安装 Docker-compose (用于 docker 容器服务统一管理 编排)


```
# 安装 pip

curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py

python get-pip.py

```


```
# 安装 docker-compose

pip install docker-compose --ignore-installed requests

```


```
docker-compose version
docker-compose version 1.32.0, build f46880f
docker-py version: 3.4.1
CPython version: 2.7.5
OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013
```


* Golang (用于 fabric cli 服务的调用， ca 服务证书生成 )


```
mkdir -p /opt/golang
mkdir -p /opt/gopath

# 国外地址
wget https://storage.googleapis.com/golang/go1.10.linux-amd64.tar.gz

# 国内地址
wget https://studygolang.com/dl/golang/go1.10.linux-amd64.tar.gz


# 解压
tar zxvf go1.10.linux-amd64.tar.gz

# 配置环境变量

vi /etc/profile

添加如下

# golang env
export PATH=$PATH:/opt/golang/go/bin
export GOPATH=/opt/gopath

# 生效配置

source /etc/profile


# 查看配置

go version
go version go1.10 linux/amd64

```



## Hyperledger Fabric 源码

> fabric 源码用于 cli 智能合约安装时的依赖, 这里只用于第一个节点


```
# 下载 Fabric 源码, 源码中 import 的路径为github.com/hyperledger/fabric ,所以我们要按照这个路径

mkdir -p /opt/gopath/src/github.com/hyperledger

cd /opt/gopath/src/github.com/hyperledger


git clone https://github.com/hyperledger/fabric


# 查看分支
git branch -a


# 查看本地分支
git branch


# 切换分支
git checkout -b release-1.3 remotes/origin/release-1.3


# 文件如下:

[root@localhost fabric]# ls -lt
总用量 1240
drwxr-xr-x  8 root root    153 11月  5 11:41 vendor
drwxr-xr-x  4 root root     96 11月  5 11:41 token
-rw-r--r--  1 root root    301 11月  5 11:41 tox.ini
drwxr-xr-x  2 root root     56 11月  5 11:41 unit-test
-rw-r--r--  1 root root   3816 11月  5 11:41 testingInfo.rst
-rw-r--r--  1 root root 438053 11月  5 11:41 test-pyramid.png
drwxr-xr-x  2 root root   4096 11月  5 11:41 scripts
-rw-r--r--  1 root root    316 11月  5 11:41 settings.gradle
drwxr-xr-x  4 root root    110 11月  5 11:41 sampleconfig
drwxr-xr-x  3 root root     22 11月  5 11:41 release
drwxr-xr-x  2 root root   4096 11月  5 11:41 release_notes
drwxr-xr-x 14 root root   4096 11月  5 11:41 protos
drwxr-xr-x 11 root root   4096 11月  5 11:41 peer
drwxr-xr-x  6 root root    126 11月  5 11:41 orderer
drwxr-xr-x  6 root root   4096 11月  5 11:41 msp
drwxr-xr-x 12 root root   4096 11月  5 11:41 integration
drwxr-xr-x  8 root root    112 11月  5 11:41 images
drwxr-xr-x  2 root root   4096 11月  5 11:41 idemix
drwxr-xr-x 16 root root   4096 11月  5 11:41 gossip
-rw-r--r--  1 root root   2944 11月  5 11:41 gotools.mk
drwxr-xr-x  8 root root    126 11月  5 11:41 examples
drwxr-xr-x  5 root root    156 11月  5 11:41 docs
drwxr-xr-x  7 root root   4096 11月  5 11:41 discovery
-rw-r--r--  1 root root   3356 11月  5 11:41 docker-env.mk
drwxr-xr-x  4 root root   4096 11月  5 11:41 devenv
drwxr-xr-x 22 root root   4096 11月  5 11:41 core
drwxr-xr-x 24 root root   4096 11月  5 11:41 common
drwxr-xr-x  4 root root     46 11月  5 11:41 cmd
-rw-r--r--  1 root root     14 11月  5 11:41 ci.properties
drwxr-xr-x  8 root root   4096 11月  5 11:41 bccsp
-rw-r--r--  1 root root   3205 11月  5 11:41 Gopkg.toml
-rw-r--r--  1 root root  11358 11月  5 11:41 LICENSE
-rwxr-xr-x  1 root root  18176 11月  5 11:41 Makefile
-rw-r--r--  1 root root   6482 11月  5 11:41 README.md
-rw-r--r--  1 root root 670001 11月  5 11:41 CHANGELOG.md
-rw-r--r--  1 root root    597 11月  5 11:41 CODE_OF_CONDUCT.md
-rw-r--r--  1 root root    664 11月  5 11:41 CONTRIBUTING.md
-rw-r--r--  1 root root  26423 11月  5 11:41 Gopkg.lock


```


## 生成 Hyperledger Fabric 证书


> 证书生成只需要生成一次，这里只在第一个节点配置


```
# 下载官方证书生成软件(均为二进制文件)
# 官方离线下载地址为 https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/


# 选择相应版本 CentOS 选择 linux-amd64-1.3.0  Mac 选择 darwin-amd64-1.3.0


# 下载地址为: https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.3.0/hyperledger-fabric-linux-amd64-1.3.0.tar.gz



mkdir /opt/jicki/

cd /opt/jicki

wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.3.0/hyperledger-fabric-linux-amd64-1.3.0.tar.gz

tar zxvf hyperledger-fabric-linux-amd64-1.3.0.tar.gz



# 解压后是 一个 bin 与 一个 config 目录

[root@localhost jicki]# tree bin/
bin/
├── configtxgen
├── configtxlator
├── cryptogen
├── discover
├── get-docker-images.sh
├── idemixgen
├── orderer
└── peer

0 directories, 8 files
 
 
# 为方便使用 我们配置一个 环境变量

vi /etc/profile


# fabric env
export PATH=$PATH:/opt/jicki/bin


# 使文件生效
source /etc/profile

```


```
# 拷贝 configtx.yaml  与 crypto-config.yaml 。


cd /opt/jicki/

cp /opt/gopath/src/github.com/hyperledger/fabric/examples/e2e_cli/crypto-config.yaml .

cp /opt/gopath/src/github.com/hyperledger/fabric/examples/e2e_cli/configtx.yaml .

```

```
# 这里修改相应 jicki.cn 为 jicki.cn

sed -i 's/example\.com/jicki\.me/g' *.yaml


# 编辑 crypto-config.yaml 增加 orderer 为三个如下:


OrdererOrgs:
  - Name: Orderer
    Domain: jicki.cn
    CA:
        Country: CN
        Province: GuangDong
        Locality: ShenZhen
    Specs:
      - Hostname: orderer0
      - Hostname: orderer1
      - Hostname: orderer2
      
PeerOrgs:
  - Name: Org1
    Domain: org1.jicki.cn
    EnableNodeOUs: true
    CA:
        Country: CN
        Province: GuangDong
        Locality: ShenZhen
    Template:
      Count: 2
    Users:
      Count: 1
  - Name: Org2
    Domain: org2.jicki.cn
    EnableNodeOUs: true
    CA:
        Country: CN
        Province: GuangDong
        Locality: ShenZhen
    Template:
      Count: 2
    Users:
      Count: 1
      
```




```
# 然后这里使用 cryptogen 软件来生成相应的证书了

[root@localhost jicki]# cryptogen generate --config=./crypto-config.yaml
org1.jicki.cn
org2.jicki.cn

# 生成一个 crypto-config 证书目录

[root@payment jicki]# tree crypto-config
crypto-config
├── ordererOrganizations
│   └── jicki.cn
│       ├── ca
│       │   ├── 1b50e9236c72ccbdf474ecd39ceb300f90b983b8044981895ee8ba4fda466c38_sk
│       │   └── ca.jicki.cn-cert.pem
│       ├── msp
│       │   ├── admincerts
│       │   │   └── Admin@jicki.cn-cert.pem
│       │   ├── cacerts
│       │   │   └── ca.jicki.cn-cert.pem
│       │   └── tlscacerts
│       │       └── tlsca.jicki.cn-cert.pem
│       ├── orderers
│       │   ├── orderer0.jicki.cn
│       │   │   ├── msp
│       │   │   │   ├── admincerts
│       │   │   │   │   └── Admin@jicki.cn-cert.pem
│       │   │   │   ├── cacerts
│       │   │   │   │   └── ca.jicki.cn-cert.pem
│       │   │   │   ├── keystore
│       │   │   │   │   └── 4dd2ea13c77739776e2e70a82516fce9836c25c92ab6d762333b468b8824fdfd_sk
│       │   │   │   ├── signcerts
│       │   │   │   │   └── orderer0.jicki.cn-cert.pem
│       │   │   │   └── tlscacerts
│       │   │   │       └── tlsca.jicki.cn-cert.pem
│       │   │   └── tls
│       │   │       ├── ca.crt
│       │   │       ├── server.crt
│       │   │       └── server.key
│       │   ├── orderer1.jicki.cn
│       │   │   ├── msp
│       │   │   │   ├── admincerts
│       │   │   │   │   └── Admin@jicki.cn-cert.pem
│       │   │   │   ├── cacerts
│       │   │   │   │   └── ca.jicki.cn-cert.pem
│       │   │   │   ├── keystore
│       │   │   │   │   └── 2a55c127a61858ddf36cdffe66782f6a3976955ed6ccafa7dd602b9f393cee77_sk
│       │   │   │   ├── signcerts
│       │   │   │   │   └── orderer1.jicki.cn-cert.pem
│       │   │   │   └── tlscacerts
│       │   │   │       └── tlsca.jicki.cn-cert.pem
│       │   │   └── tls
│       │   │       ├── ca.crt
│       │   │       ├── server.crt
│       │   │       └── server.key
│       │   └── orderer2.jicki.cn
│       │       ├── msp
│       │       │   ├── admincerts
│       │       │   │   └── Admin@jicki.cn-cert.pem
│       │       │   ├── cacerts
│       │       │   │   └── ca.jicki.cn-cert.pem
│       │       │   ├── keystore
│       │       │   │   └── d85da317a39fc3c95c5ddce183bf492e1310decee79dd215ef4cfd4f7130d0c4_sk
│       │       │   ├── signcerts
│       │       │   │   └── orderer2.jicki.cn-cert.pem
│       │       │   └── tlscacerts
│       │       │       └── tlsca.jicki.cn-cert.pem
│       │       └── tls
│       │           ├── ca.crt
│       │           ├── server.crt
│       │           └── server.key
│       ├── tlsca
│       │   ├── e03e3c85138178da8a03c3cc00aecb0627aa299bdd0b92c4bc82591cd47a2672_sk
│       │   └── tlsca.jicki.cn-cert.pem
│       └── users
│           └── Admin@jicki.cn
│               ├── msp
│               │   ├── admincerts
│               │   │   └── Admin@jicki.cn-cert.pem
│               │   ├── cacerts
│               │   │   └── ca.jicki.cn-cert.pem
│               │   ├── keystore
│               │   │   └── fac87a53e4255e8007bf947bda23464b40f46107ebcae48400e33ae5b778ac32_sk
│               │   ├── signcerts
│               │   │   └── Admin@jicki.cn-cert.pem
│               │   └── tlscacerts
│               │       └── tlsca.jicki.cn-cert.pem
│               └── tls
│                   ├── ca.crt
│                   ├── client.crt
│                   └── client.key
└── peerOrganizations
    ├── org1.jicki.cn
    │   ├── ca
    │   │   ├── aa167e5dcb62e9e1068487484c08f0bc0a13e13cebf53a0fa2f64038a1a76662_sk
    │   │   └── ca.org1.jicki.cn-cert.pem
    │   ├── msp
    │   │   ├── admincerts
    │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │   │   ├── cacerts
    │   │   │   └── ca.org1.jicki.cn-cert.pem
    │   │   ├── config.yaml
    │   │   └── tlscacerts
    │   │       └── tlsca.org1.jicki.cn-cert.pem
    │   ├── peers
    │   │   ├── peer0.org1.jicki.cn
    │   │   │   ├── msp
    │   │   │   │   ├── admincerts
    │   │   │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │   │   │   │   ├── cacerts
    │   │   │   │   │   └── ca.org1.jicki.cn-cert.pem
    │   │   │   │   ├── config.yaml
    │   │   │   │   ├── keystore
    │   │   │   │   │   └── 7d20eedce4f8f3417bb1d4b723a0018b701fb2485afd727a2e1f34c09148e4eb_sk
    │   │   │   │   ├── signcerts
    │   │   │   │   │   └── peer0.org1.jicki.cn-cert.pem
    │   │   │   │   └── tlscacerts
    │   │   │   │       └── tlsca.org1.jicki.cn-cert.pem
    │   │   │   └── tls
    │   │   │       ├── ca.crt
    │   │   │       ├── server.crt
    │   │   │       └── server.key
    │   │   └── peer1.org1.jicki.cn
    │   │       ├── msp
    │   │       │   ├── admincerts
    │   │       │   │   └── Admin@org1.jicki.cn-cert.pem
    │   │       │   ├── cacerts
    │   │       │   │   └── ca.org1.jicki.cn-cert.pem
    │   │       │   ├── config.yaml
    │   │       │   ├── keystore
    │   │       │   │   └── f4b8c4a0ebe3b34fe08cc2d33ea0c8f1e8a9817ffacc42d0c97323f0ddd57958_sk
    │   │       │   ├── signcerts
    │   │       │   │   └── peer1.org1.jicki.cn-cert.pem
    │   │       │   └── tlscacerts
    │   │       │       └── tlsca.org1.jicki.cn-cert.pem
    │   │       └── tls
    │   │           ├── ca.crt
    │   │           ├── server.crt
    │   │           └── server.key
    │   ├── tlsca
    │   │   ├── 9c549e33c1d594a0f5eaedac0e1e6c753b9c3b2368d27d907d5a4ad59faa06d0_sk
    │   │   └── tlsca.org1.jicki.cn-cert.pem
    │   └── users
    │       ├── Admin@org1.jicki.cn
    │       │   ├── msp
    │       │   │   ├── admincerts
    │       │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │       │   │   ├── cacerts
    │       │   │   │   └── ca.org1.jicki.cn-cert.pem
    │       │   │   ├── keystore
    │       │   │   │   └── a9666f561d211e7b7cc170bfe854721431a1038b7914a67555d82dcf6b9eaaf8_sk
    │       │   │   ├── signcerts
    │       │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │       │   │   └── tlscacerts
    │       │   │       └── tlsca.org1.jicki.cn-cert.pem
    │       │   └── tls
    │       │       ├── ca.crt
    │       │       ├── client.crt
    │       │       └── client.key
    │       └── User1@org1.jicki.cn
    │           ├── msp
    │           │   ├── admincerts
    │           │   │   └── User1@org1.jicki.cn-cert.pem
    │           │   ├── cacerts
    │           │   │   └── ca.org1.jicki.cn-cert.pem
    │           │   ├── keystore
    │           │   │   └── 543ee272a207c159538c412a2d97f83863031027014a17d2e4a6e3de931cfa3d_sk
    │           │   ├── signcerts
    │           │   │   └── User1@org1.jicki.cn-cert.pem
    │           │   └── tlscacerts
    │           │       └── tlsca.org1.jicki.cn-cert.pem
    │           └── tls
    │               ├── ca.crt
    │               ├── client.crt
    │               └── client.key
    └── org2.jicki.cn
        ├── ca
        │   ├── bcbc85a3992715502c7e7e7d21b792678a9d03668a52bb18a0c10862d45222c3_sk
        │   └── ca.org2.jicki.cn-cert.pem
        ├── msp
        │   ├── admincerts
        │   │   └── Admin@org2.jicki.cn-cert.pem
        │   ├── cacerts
        │   │   └── ca.org2.jicki.cn-cert.pem
        │   ├── config.yaml
        │   └── tlscacerts
        │       └── tlsca.org2.jicki.cn-cert.pem
        ├── peers
        │   ├── peer0.org2.jicki.cn
        │   │   ├── msp
        │   │   │   ├── admincerts
        │   │   │   │   └── Admin@org2.jicki.cn-cert.pem
        │   │   │   ├── cacerts
        │   │   │   │   └── ca.org2.jicki.cn-cert.pem
        │   │   │   ├── config.yaml
        │   │   │   ├── keystore
        │   │   │   │   └── ad97de341e17baae8d25d34d1d23808cb77b5eceaf1ab53586cbf751ff9157cb_sk
        │   │   │   ├── signcerts
        │   │   │   │   └── peer0.org2.jicki.cn-cert.pem
        │   │   │   └── tlscacerts
        │   │   │       └── tlsca.org2.jicki.cn-cert.pem
        │   │   └── tls
        │   │       ├── ca.crt
        │   │       ├── server.crt
        │   │       └── server.key
        │   └── peer1.org2.jicki.cn
        │       ├── msp
        │       │   ├── admincerts
        │       │   │   └── Admin@org2.jicki.cn-cert.pem
        │       │   ├── cacerts
        │       │   │   └── ca.org2.jicki.cn-cert.pem
        │       │   ├── config.yaml
        │       │   ├── keystore
        │       │   │   └── a38c440f80e16a95a808744f4b6409c4f2fc5d718c06047bd4b13e5d71b8c868_sk
        │       │   ├── signcerts
        │       │   │   └── peer1.org2.jicki.cn-cert.pem
        │       │   └── tlscacerts
        │       │       └── tlsca.org2.jicki.cn-cert.pem
        │       └── tls
        │           ├── ca.crt
        │           ├── server.crt
        │           └── server.key
        ├── tlsca
        │   ├── 6a4bf592dd12463266fad5f8ef683ad90062ea1913f734a9149046b6269ca73b_sk
        │   └── tlsca.org2.jicki.cn-cert.pem
        └── users
            ├── Admin@org2.jicki.cn
            │   ├── msp
            │   │   ├── admincerts
            │   │   │   └── Admin@org2.jicki.cn-cert.pem
            │   │   ├── cacerts
            │   │   │   └── ca.org2.jicki.cn-cert.pem
            │   │   ├── keystore
            │   │   │   └── d402b684b807080653511978a51fcd2326668156cd8a10fb612b80bc49c9b354_sk
            │   │   ├── signcerts
            │   │   │   └── Admin@org2.jicki.cn-cert.pem
            │   │   └── tlscacerts
            │   │       └── tlsca.org2.jicki.cn-cert.pem
            │   └── tls
            │       ├── ca.crt
            │       ├── client.crt
            │       └── client.key
            └── User1@org2.jicki.cn
                ├── msp
                │   ├── admincerts
                │   │   └── User1@org2.jicki.cn-cert.pem
                │   ├── cacerts
                │   │   └── ca.org2.jicki.cn-cert.pem
                │   ├── keystore
                │   │   └── b7d167da7a974720943d33bd5641d0b306273565a7e715538d3edf1e58f5d3e6_sk
                │   ├── signcerts
                │   │   └── User1@org2.jicki.cn-cert.pem
                │   └── tlscacerts
                │       └── tlsca.org2.jicki.cn-cert.pem
                └── tls
                    ├── ca.crt
                    ├── client.crt
                    └── client.key

125 directories, 123 files
```



## 生成 Hyperledger Fabric 创世区块



```
# 这里使用 configtxgen 来创建 创世区块


# 首先需要创建一个文件夹
mkdir -p /opt/jicki/channel-artifacts


# 修改 configtx.yaml 文件增加 orderer 多节点 以及 peer 多节点, 删除了 &Org3 节点

# 完整 configtx.yaml 如下: 
# configtx.yaml 文件格式 请千万注意 空格 与 tab 键 里的缩进，否则会报错。

Organizations:

    - &OrdererOrg
        Name: OrdererOrg
        ID: OrdererMSP
        MSPDir: crypto-config/ordererOrganizations/jicki.cn/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Writers:
                Type: Signature
                Rule: "OR('OrdererMSP.member')"
            Admins:
                Type: Signature
                Rule: "OR('OrdererMSP.admin')"

    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: crypto-config/peerOrganizations/org1.jicki.cn/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.peer', 'Org1MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org1MSP.admin', 'Org1MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org1MSP.admin')"

        AnchorPeers:
            - Host: peer0.org1.jicki.cn
              Port: 7051

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2.jicki.cn/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.peer', 'Org2MSP.client')"
            Writers:
                Type: Signature
                Rule: "OR('Org2MSP.admin', 'Org2MSP.client')"
            Admins:
                Type: Signature
                Rule: "OR('Org2MSP.admin')"

        AnchorPeers:
            - Host: peer0.org2.jicki.cn
              Port: 7051

Capabilities:
    Channel: &ChannelCapabilities
        V1_3: true

    Orderer: &OrdererCapabilities
        V1_1: true

    Application: &ApplicationCapabilities
        V1_3: true
        V1_2: false
        V1_1: false

Application: &ApplicationDefaults

    Organizations:

    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"

    Capabilities:
        <<: *ApplicationCapabilities

Orderer: &OrdererDefaults

    OrdererType: kafka

    Addresses:
        - orderer0.jicki.cn:7050
        - orderer1.jicki.cn:7050
        - orderer2.jicki.cn:7050

    BatchTimeout: 2s

    BatchSize:
        MaxMessageCount: 10
        AbsoluteMaxBytes: 99 MB
        PreferredMaxBytes: 512 KB

    Kafka:
        Brokers:
            - kafka0:9092
            - kafka1:9092
            - kafka2:9092

    Organizations:

    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"
        BlockValidation:
            Type: ImplicitMeta
            Rule: "ANY Writers"

    Capabilities:
        <<: *OrdererCapabilities

Channel: &ChannelDefaults
    Policies:
        Readers:
            Type: ImplicitMeta
            Rule: "ANY Readers"
        Writers:
            Type: ImplicitMeta
            Rule: "ANY Writers"
        Admins:
            Type: ImplicitMeta
            Rule: "MAJORITY Admins"

    Capabilities:
        <<: *ChannelCapabilities

Profiles:
    TwoOrgsOrdererGenesis:
        <<: *ChannelDefaults
        Orderer:
            <<: *OrdererDefaults
            Organizations:
                - *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                    - *Org1
                    - *Org2
    TwoOrgsChannel:
        Consortium: SampleConsortium
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org1
                - *Org2

```






```
# 创建 创世区块  TwoOrgsOrdererGenesis 名称为 configtx.yaml 中 Profiles 字段下的

[root@localhost jicki]# configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block


2018-11-05 12:06:21.780 CST [common/tools/configtxgen] main -> WARN 001 Omitting the channel ID for configtxgen for output operations is deprecated.  Explicitly passing the channel ID will be required in the future, defaulting to 'testchainid'.
2018-11-05 12:06:21.780 CST [common/tools/configtxgen] main -> INFO 002 Loading configuration
2018-11-05 12:06:21.829 CST [common/tools/configtxgen] doOutputBlock -> INFO 003 Generating genesis block
2018-11-05 12:06:21.829 CST [common/tools/configtxgen] doOutputBlock -> INFO 004 Writing genesis block



# 创世区块 是在 orderer 服务中使用

[root@localhost jicki]# ls -lt channel-artifacts/
总用量 16
-rw-r--r-- 1 root root 12479 11月  5 12:06 genesis.block
```


```
# 下面来生成一个 peer 服务 中使用的 tx 文件 TwoOrgsChannel 名称为 configtx.yaml 中 Profiles 字段下的，这里必须指定上面的 channelID


[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel


2018-11-05 12:08:11.241 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-11-05 12:08:11.286 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-11-05 12:08:11.288 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 003 Writing new channel tx


[root@localhost jicki]# ls -lt channel-artifacts/
总用量 20
-rw-r--r-- 1 root root   346 11月  5 12:08 channel.tx
-rw-r--r-- 1 root root 12479 11月  5 12:06 genesis.block

```


```
# 定义组织 生成锚节点更新文件


# Org1MSP

[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP


2018-11-05 12:08:47.653 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-11-05 12:08:47.701 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 002 Generating anchor peer update
2018-11-05 12:08:47.702 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Writing anchor peer update



# Org2MSP


[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP


2018-11-05 12:09:00.134 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-11-05 12:09:00.179 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 002 Generating anchor peer update
2018-11-05 12:09:00.179 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Writing anchor peer update


```


## 配置 Zookeeper Kafka 集群

> 启动 zookeeper 与 kafka 集群


```
vi docker-compose-zk-kafka.yaml


version: '2'
services:
  zookeeper1:
    container_name: zookeeper1
    hostname: zookeeper1
    image: hyperledger/fabric-zookeeper
    restart: always
    environment:
      - ZOO_MY_ID=1
      - ZOO_SERVERS=server.1=zookeeper1:2888:3888 server.2=zookeeper2:2888:3888 server.3=zookeeper3:2888:3888
    #volumes:
    # 存储数据与日志
    #- ./data/zookeeper1/data:/data
    #- ./data/zookeeper1/datalog:/datalog
    networks:
      default:
        aliases:
          - jicki

  zookeeper2:
    container_name: zookeeper2
    hostname: zookeeper2
    image: hyperledger/fabric-zookeeper
    restart: always
    environment:
      - ZOO_MY_ID=2
      - ZOO_SERVERS=server.1=zookeeper1:2888:3888 server.2=zookeeper2:2888:3888 server.3=zookeeper3:2888:3888
    #volumes:
    # 存储数据与日志
    #- ./data/zookeeper2/data:/data
    #- ./data/zookeeper2/datalog:/datalog
    networks:
      default:
        aliases:
          - jicki

  zookeeper3:
    container_name: zookeeper3
    hostname: zookeeper3
    image: hyperledger/fabric-zookeeper
    restart: always
    environment:
      - ZOO_MY_ID=3
      - ZOO_SERVERS=server.1=zookeeper1:2888:3888 server.2=zookeeper2:2888:3888 server.3=zookeeper3:2888:3888
    #volumes:
    # 存储数据与日志
    #- ./data/zookeeper3/data:/data
    #- ./data/zookeeper3/datalog:/datalog
    networks:
      default:
        aliases:
          - jicki


  kafka0:
    container_name: kafka0
    hostname: kafka0
    image: hyperledger/fabric-kafka
    restart: always
    environment:
      - KAFKA_BROKER_ID=1
      # 设置一个M值,数据提交时会写入至少M个副本(这里M=2)（这些数据会被同步并且归属到in-sync 副本集合或ISR）M 必须小于 如下 N 值,并且大于1,既最小为2。
      - KAFKA_MIN_INSYNC_REPLICAS=2
      # 设置一个N值, N代表着每个channel都保存N个副本的数据到Kafka的代理上。N 必须大于如上 M 值, 既 N 值最小值为 3。
      - KAFKA_DEFAULT_REPLICATION_FACTOR=3
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper1:2181,zookeeper2:2181,zookeeper3:2181
      # 如下99为configtx.yaml中会设置最大的区块大小(参考configtx.yaml中AbsoluteMaxBytes参数)
      # 每个区块最大有Orderer.AbsoluteMaxBytes个字节
      # 99 * 1024 * 1024 B
      - KAFKA_MESSAGE_MAX_BYTES=103809024
      # 每个通道获取的消息的字节数 如上一样
      - KAFKA_REPLICA_FETCH_MAX_BYTES=103809024
      # 数据一致性在区块链环境中是至关重要的, 我们不能从in-sync 副本（ISR）集合之外选取channel leader , 否则我们将会面临对于之前的leader产生的offsets覆盖的风险
      - KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE=false
      # 关闭基于时间的日志保留方式并且避免分段到期。
      - KAFKA_LOG_RETENTION_MS=-1
    #volumes:
    # 存储数据与日志.
    #- ./data/kafka1/data:/data
    #- ./data/kafka1/data:/logs
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - zookeeper1
      - zookeeper2
      - zookeeper3

  kafka1:
    container_name: kafka1
    hostname: kafka1
    image: hyperledger/fabric-kafka
    restart: always
    environment:
      - KAFKA_BROKER_ID=2
      - KAFKA_MIN_INSYNC_REPLICAS=2
      - KAFKA_DEFAULT_REPLICATION_FACTOR=3
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper1:2181,zookeeper2:2181,zookeeper3:2181
      - KAFKA_MESSAGE_MAX_BYTES=103809024
      - KAFKA_REPLICA_FETCH_MAX_BYTES=103809024
      - KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE=false
      - KAFKA_LOG_RETENTION_MS=-1
    #volumes:
    # 存储数据与日志.
    #- ./data/kafka1/data:/data
    #- ./data/kafka1/data:/logs
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - zookeeper1
      - zookeeper2
      - zookeeper3
          
  kafka2:
    container_name: kafka2
    hostname: kafka2
    image: hyperledger/fabric-kafka
    restart: always
    environment:
      - KAFKA_BROKER_ID=3
      - KAFKA_MIN_INSYNC_REPLICAS=2
      - KAFKA_DEFAULT_REPLICATION_FACTOR=3
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper1:2181,zookeeper2:2181,zookeeper3:2181
      - KAFKA_MESSAGE_MAX_BYTES=103809024
      - KAFKA_REPLICA_FETCH_MAX_BYTES=103809024
      - KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE=false
      # 关闭基于时间的日志保留方式并且避免分段到期。
      - KAFKA_LOG_RETENTION_MS=-1
    #volumes:
    # 存储数据与日志.
    #- ./data/kafka2/data:/data
    #- ./data/kafka2/data:/logs
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - zookeeper1
      - zookeeper2
      - zookeeper3
      
```


```
# 分别启动服务

docker-compose -f docker-compose-zk-kafka.yaml up -d

```




## 配置 Hyperledger Fabric Orderer


```
# 创建文件 docker-compose-orderer.yaml

# 创建于 /opt/jicki 目录下

vi  docker-compose-orderer.yaml


version: '2'
services:
  orderer0.jicki.cn:
    container_name: orderer0.jicki.cn
    image: hyperledger/fabric-orderer
    environment:
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      # - ORDERER_GENERAL_LOGLEVEL=error
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_LISTENPORT=7050
      #- ORDERER_GENERAL_GENESISPROFILE=AntiMothOrdererGenesis
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      #- ORDERER_GENERAL_LEDGERTYPE=ram
      #- ORDERER_GENERAL_LEDGERTYPE=file
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt, /etc/hyperledger/crypto/peerOrg1/tls/ca.crt, /etc/hyperledger/crypto/peerOrg2/tls/ca.crt]

      # KAFKA
      - ORDERER_KAFKA_RETRY_LONGINTERVAL=10s
      - ORDERER_KAFKA_RETRY_LONGTOTAL=100s
      - ORDERER_KAFKA_RETRY_SHORTINTERVAL=1s
      - ORDERER_KAFKA_RETRY_SHORTTOTAL=30s
      - ORDERER_KAFKA_VERBOSE=true
      - ORDERER_KAFKA_BROKERS=[kafka0:9092,kafka1:9092,kafka2:9092]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/tls/:/var/hyperledger/orderer/tls
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/:/etc/hyperledger/crypto/peerOrg1
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/:/etc/hyperledger/crypto/peerOrg2
    networks:
      default:
        aliases:
          - jicki
    ports:
      - 7050:7050
    depends_on:
      - kafka0
      - kafka1
      - kafka2

  orderer1.jicki.cn:
    container_name: orderer1.jicki.cn
    image: hyperledger/fabric-orderer
    environment:
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      # - ORDERER_GENERAL_LOGLEVEL=error
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_LISTENPORT=7050
      #- ORDERER_GENERAL_GENESISPROFILE=AntiMothOrdererGenesis
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      #- ORDERER_GENERAL_LEDGERTYPE=ram
      #- ORDERER_GENERAL_LEDGERTYPE=file
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt, /etc/hyperledger/crypto/peerOrg1/tls/ca.crt, /etc/hyperledger/crypto/peerOrg2/tls/ca.crt]

      # KAFKA
      - ORDERER_KAFKA_RETRY_LONGINTERVAL=10s
      - ORDERER_KAFKA_RETRY_LONGTOTAL=100s
      - ORDERER_KAFKA_RETRY_SHORTINTERVAL=1s
      - ORDERER_KAFKA_RETRY_SHORTTOTAL=30s
      - ORDERER_KAFKA_VERBOSE=true
      - ORDERER_KAFKA_BROKERS=[kafka0:9092,kafka1:9092,kafka2:9092]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/tls/:/var/hyperledger/orderer/tls
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/:/etc/hyperledger/crypto/peerOrg1
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/:/etc/hyperledger/crypto/peerOrg2
    networks:
      default:
        aliases:
          - jicki
    ports:
      - 8050:7050
    depends_on:
      - kafka0
      - kafka1
      - kafka2
      
  orderer2.jicki.cn:
    container_name: orderer2.jicki.cn
    image: hyperledger/fabric-orderer
    environment:
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      # - ORDERER_GENERAL_LOGLEVEL=error
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_LISTENPORT=7050
      #- ORDERER_GENERAL_GENESISPROFILE=AntiMothOrdererGenesis
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      #- ORDERER_GENERAL_LEDGERTYPE=ram
      #- ORDERER_GENERAL_LEDGERTYPE=file
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt, /etc/hyperledger/crypto/peerOrg1/tls/ca.crt, /etc/hyperledger/crypto/peerOrg2/tls/ca.crt]

      # KAFKA
      - ORDERER_KAFKA_RETRY_LONGINTERVAL=10s
      - ORDERER_KAFKA_RETRY_LONGTOTAL=100s
      - ORDERER_KAFKA_RETRY_SHORTINTERVAL=1s
      - ORDERER_KAFKA_RETRY_SHORTTOTAL=30s
      - ORDERER_KAFKA_VERBOSE=true
      - ORDERER_KAFKA_BROKERS=[kafka0:9092,kafka1:9092,kafka2:9092]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/tls/:/var/hyperledger/orderer/tls
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/:/etc/hyperledger/crypto/peerOrg1
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/:/etc/hyperledger/crypto/peerOrg2
    networks:
      default:
        aliases:
          - jicki
    ports:
      - 9050:7050
    depends_on:
      - kafka0
      - kafka1
      - kafka2
      
```


### 启动服务

> 所有节点启动服务

```
docker-compose -f docker-compose-orderer.yaml up -d


# 启动完毕，查看docker logs 如下表示成功



2018-08-07 01:42:14.431 UTC [orderer/consensus/kafka] processMessagesToBlocks -> DEBU 2fd [channel: testchainid] Successfully unmarshalled consumed message, offset is 0. Inspecting type...
2018-08-07 01:42:14.431 UTC [orderer/consensus/kafka] processConnect -> DEBU 2fe [channel: testchainid] It's a connect message - ignoring
2018-08-07 01:42:14.431 UTC [orderer/consensus/kafka] processMessagesToBlocks -> DEBU 2ff [channel: testchainid] Successfully unmarshalled consumed message, offset is 1. Inspecting type...
2018-08-07 01:42:14.431 UTC [orderer/consensus/kafka] processConnect -> DEBU 300 [channel: testchainid] It's a connect message - ignoring
2018-08-07 01:42:14.431 UTC [orderer/consensus/kafka] processMessagesToBlocks -> DEBU 301 [channel: testchainid] Successfully unmarshalled consumed message, offset is 2. Inspecting type...
2018-08-07 01:42:14.431 UTC [orderer/consensus/kafka] processConnect -> DEBU 302 [channel: testchainid] It's a connect message - ignoring

```
 
## 配置 Hyperledger Fabric ca 节点

> 配置两个 ca 节点


```
vi docker-compose-ca.yaml

version: '2'
services:
  ca.org1.jicki.cn:
    container_name: ca.org1.jicki.cn
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca-org1
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/aa167e5dcb62e9e1068487484c08f0bc0a13e13cebf53a0fa2f64038a1a76662_sk
      - FABRIC_CA_SERVER_TLS_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_TLS_KEYFILE=/etc/hyperledger/fabric-ca-server-config/aa167e5dcb62e9e1068487484c08f0bc0a13e13cebf53a0fa2f64038a1a76662_sk
    ports:
      - "7054:7054"
    command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/aa167e5dcb62e9e1068487484c08f0bc0a13e13cebf53a0fa2f64038a1a76662_sk -b admin:adminpw -d'
    volumes:
      - ./crypto-config/peerOrganizations/org1.jicki.cn/ca/:/etc/hyperledger/fabric-ca-server-config
    depends_on:
      - orderer0.jicki.cn
      - orderer1.jicki.cn
      - orderer2.jicki.cn
      

  ca.org2.jicki.cn:
    container_name: ca.org2.jicki.cn
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca-org2
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org2.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/bcbc85a3992715502c7e7e7d21b792678a9d03668a52bb18a0c10862d45222c3_sk
      - FABRIC_CA_SERVER_TLS_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org2.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_TLS_KEYFILE=/etc/hyperledger/fabric-ca-server-config/bcbc85a3992715502c7e7e7d21b792678a9d03668a52bb18a0c10862d45222c3_sk
    ports:
      - "8054:7054"
    command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org2.jicki.cn-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/bcbc85a3992715502c7e7e7d21b792678a9d03668a52bb18a0c10862d45222c3_sk -b admin:adminpw -d'
    volumes:
      - ./crypto-config/peerOrganizations/org2.jicki.cn/ca/:/etc/hyperledger/fabric-ca-server-config
    depends_on:
      - orderer0.jicki.cn
      - orderer1.jicki.cn
      - orderer2.jicki.cn

```

 
 
## 配置 Hyperledger Fabric peer
 
> peer 节点下都必须启动一个数据存储，如 file 或者 couchdb 等


```      
vi  docker-compose-peer.yaml

version: '2'
services:
  couchdb0:
    container_name: couchdb0
    image: hyperledger/fabric-couchdb
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - "5984:5984"
    #volumes:
      # 数据持久化，用于存储链码值
      #- ./data/couchdb0/data:/opt/couchdb/data
    networks:
      default:
        aliases:
          - jicki

  couchdb1:
    container_name: couchdb1
    image: hyperledger/fabric-couchdb
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - "6984:5984"
    #volumes:
      # 数据持久化，用于存储链码值
      #- ./data/couchdb1/data:/opt/couchdb/data
    networks:
      default:
        aliases:
          - jicki
          
  peer0.org1.jicki.cn:
    container_name: peer0.org1.jicki.cn
    image: hyperledger/fabric-peer
    environment:
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb0:5984
      
      - CORE_PEER_ID=peer0.org1.jicki.cn
      - CORE_PEER_NETWORKID=jicki
      - CORE_PEER_ADDRESS=peer0.org1.jicki.cn:7051
      - CORE_PEER_CHAINCODELISTENADDRESS=peer0.org1.jicki.cn:7052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.jicki.cn:7051
      - CORE_PEER_LOCALMSPID=Org1MSP

      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki
      # - CORE_LOGGING_LEVEL=ERROR
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      - CORE_PEER_GOSSIP_SKIPHANDSHAKE=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=false
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls:/etc/hyperledger/fabric/tls
        # 数据持久化, 存储安装，以及实例化智能合约的数据
        #- ./data/peer0org1:/var/hyperledger/production
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    ports:
      - 7051:7051
      - 7052:7052
      - 7053:7053
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - couchdb0
      - ca.org1.jicki.cn
      - ca.org2.jicki.cn
      - orderer0.jicki.cn
      - orderer1.jicki.cn
      - orderer2.jicki.cn

  peer0.org2.jicki.cn:
    container_name: peer0.org2.jicki.cn
    image: hyperledger/fabric-peer
    environment:
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb1:5984
      
      - CORE_PEER_ID=peer0.org2.jicki.cn
      - CORE_PEER_NETWORKID=jicki
      - CORE_PEER_ADDRESS=peer0.org2.jicki.cn:7051
      - CORE_PEER_CHAINCODELISTENADDRESS=peer0.org2.jicki.cn:7052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.jicki.cn:7051
      - CORE_PEER_LOCALMSPID=Org2MSP

      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki
      # - CORE_LOGGING_LEVEL=ERROR
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      - CORE_PEER_GOSSIP_SKIPHANDSHAKE=true
      - CORE_PEER_GOSSIP_USELEADERELECTION=true
      - CORE_PEER_GOSSIP_ORGLEADER=false
      - CORE_PEER_PROFILE_ENABLED=false
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls:/etc/hyperledger/fabric/tls
        # 数据持久化, 存储安装，以及实例化智能合约的数据
        #- ./data/peer0org2:/var/hyperledger/production
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    ports:
      - 8051:7051
      - 8052:7052
      - 8053:7053
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - couchdb1
      - ca.org1.jicki.cn
      - ca.org2.jicki.cn
      - orderer0.jicki.cn
      - orderer1.jicki.cn
      - orderer2.jicki.cn

```
      
### 启动服务

```
docker-compose -f docker-compose-peer.yaml up -d

```
      


## 配置 Hyperledger Fabric cli 客户端
   
 
> 客户端服务，只需要配置一个既可，用于调用创建 channel 与 智能合约


```
vi  docker-compose-cli.yaml


version: '2'
services:
  cli:
    container_name: cli
    image: hyperledger/fabric-tools
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # - CORE_LOGGING_LEVEL=ERROR
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=cli
      - CORE_PEER_ADDRESS=peer0.org1.jicki.cn:7051
      - CORE_PEER_LOCALMSPID=Org1MSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/users/Admin@org1.jicki.cn/msp
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    volumes:
        - /var/run/:/host/var/run/
        - /opt/golang/go:/opt/go
        - /opt/gopath:/opt/gopath
        - ./peer:/opt/gopath/src/github.com/hyperledger/fabric/peer
        - ./chaincode/go/:/opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    networks:
      default:
        aliases:
          - jicki
```


### 启动服务

```
docker-compose -f docker-compose-cli.yaml up -d

```






## Hyperledger Fabric 创建 Channel



```
# 上面我们创建了 cli 容器，我们可以直接进入 容器里操作


[root@localhost jicki]# docker exec -it cli bash
root@0b55c64a9853:/opt/gopath/src/github.com/hyperledger/fabric/peer# 

```


```
# 执行 创建命令 (未启动 认证)

peer channel create -c mychannel -f ./channel-artifacts/channel.tx --orderer orderer0.jicki.cn:7050


# 提示如下表示认证不通过
Error: failed to create deliver client: rpc error: code = Unavailable desc = all SubConns are in TransientFailure, latest connection error: <nil>
```





```
# 以下为启用认证
peer channel create -o orderer0.jicki.cn:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem




# 输出如下:

2018-11-05 04:29:11.416 UTC [grpc] DialContext -> DEBU 060 parsed scheme: ""
2018-11-05 04:29:11.416 UTC [grpc] DialContext -> DEBU 061 scheme "" not registered, fallback to default scheme
2018-11-05 04:29:11.416 UTC [grpc] watcher -> DEBU 062 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-11-05 04:29:11.416 UTC [grpc] switchBalancer -> DEBU 063 ClientConn switching balancer to "pick_first"
2018-11-05 04:29:11.417 UTC [grpc] HandleSubConnStateChange -> DEBU 064 pickfirstBalancer: HandleSubConnStateChange: 0xc420389240, CONNECTING
2018-11-05 04:29:11.417 UTC [grpc] HandleSubConnStateChange -> DEBU 065 pickfirstBalancer: HandleSubConnStateChange: 0xc420389240, READY
2018-11-05 04:29:11.417 UTC [channelCmd] InitCmdFactory -> INFO 066 Endorser and orderer connections initialized
2018-11-05 04:29:11.618 UTC [msp] GetDefaultSigningIdentity -> DEBU 067 Obtaining default signing identity
2018-11-05 04:29:11.618 UTC [msp] GetDefaultSigningIdentity -> DEBU 068 Obtaining default signing identity
2018-11-05 04:29:11.618 UTC [msp/identity] Sign -> DEBU 069 Sign: plaintext: 0AD1060A1508051A0608978EFFDE0522...3561A3D8F91812080A021A0012021A00 
2018-11-05 04:29:11.618 UTC [msp/identity] Sign -> DEBU 06a Sign: digest: 8E158641AEF7FC331BDF972FB71525AA5B119D2ABD26A6B4932A6BE6E0179D4F 
2018-11-05 04:29:11.622 UTC [cli/common] readBlock -> INFO 06b Received block: 0




# 创建以后生成文件 mychannel.block

total 16
-rw-r--r-- 1 root root 15407 Nov  5 04:29 mychannel.block
drwxr-xr-x 2 root root   111 Nov  5 04:09 channel-artifacts
drwxr-xr-x 4 root root    69 Nov  5 03:53 crypto

```



## Hyperledger Fabric 加入 Channel


> 我们这边有2个 peer 所以需要分别加入, 后续有多少个 peer 都需要加入到 Channel 中

```
# peer0.org1.jicki.cn 加入 此 channel 中，首先需要查看如下 环境变量


echo $CORE_PEER_LOCALMSPID
echo $CORE_PEER_ADDRESS
echo $CORE_PEER_MSPCONFIGPATH
echo $CORE_PEER_TLS_ROOTCERT_FILE
```



```
# 加入 channel (未开启认证)

peer channel join -b mychannel.block

```


```
# 加入 channel (开启认证)

peer channel join -b mychannel.block -o orderer0.jicki.cn:7050 --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem


# 输出如下: 

2018-11-07 03:12:21.050 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-07 03:12:21.050 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-11-07 03:12:21.050 UTC [msp] GetDefaultSigningIdentity -> DEBU 037 Obtaining default signing identity
2018-11-07 03:12:21.051 UTC [grpc] DialContext -> DEBU 038 parsed scheme: ""
2018-11-07 03:12:21.051 UTC [grpc] DialContext -> DEBU 039 scheme "" not registered, fallback to default scheme
2018-11-07 03:12:21.051 UTC [grpc] watcher -> DEBU 03a ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-07 03:12:21.051 UTC [grpc] switchBalancer -> DEBU 03b ClientConn switching balancer to "pick_first"
2018-11-07 03:12:21.051 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc42053dd30, CONNECTING
2018-11-07 03:12:21.056 UTC [grpc] HandleSubConnStateChange -> DEBU 03d pickfirstBalancer: HandleSubConnStateChange: 0xc42053dd30, READY
2018-11-07 03:12:21.056 UTC [channelCmd] InitCmdFactory -> INFO 03e Endorser and orderer connections initialized
2018-11-07 03:12:21.057 UTC [msp/identity] Sign -> DEBU 03f Sign: plaintext: 0A97070A5B08011A0B0895B089DF0510...FD5719255DE61A080A000A000A000A00 
2018-11-07 03:12:21.057 UTC [msp/identity] Sign -> DEBU 040 Sign: digest: 4F2C696D018EC053894B7911B363A9D58394B6DF795CBDBEFDF52A1775AF87F3 
2018-11-07 03:12:21.163 UTC [channelCmd] executeJoin -> INFO 041 Successfully submitted proposal to join channel

```


```
# peer1.org2.jicki.cn 加入 此 channel 中，这里配置一下环境变量


export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_ADDRESS=peer0.org2.jicki.cn:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp
```


```
# 加入 channel (未开启认证)

peer channel join -b mychannel.block

```



```

# 加入 channel (开启认证)

peer channel join -b mychannel.block -o orderer0.jicki.cn:7050 --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem


# 输入如下:

2018-11-07 03:13:15.350 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-07 03:13:15.350 UTC [msp] Validate -> DEBU 036 MSP Org2MSP validating identity
2018-11-07 03:13:15.351 UTC [msp] GetDefaultSigningIdentity -> DEBU 037 Obtaining default signing identity
2018-11-07 03:13:15.351 UTC [grpc] DialContext -> DEBU 038 parsed scheme: ""
2018-11-07 03:13:15.351 UTC [grpc] DialContext -> DEBU 039 scheme "" not registered, fallback to default scheme
2018-11-07 03:13:15.352 UTC [grpc] watcher -> DEBU 03a ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-11-07 03:13:15.352 UTC [grpc] switchBalancer -> DEBU 03b ClientConn switching balancer to "pick_first"
2018-11-07 03:13:15.352 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc420311d00, CONNECTING
2018-11-07 03:13:15.356 UTC [grpc] HandleSubConnStateChange -> DEBU 03d pickfirstBalancer: HandleSubConnStateChange: 0xc420311d00, READY
2018-11-07 03:13:15.356 UTC [channelCmd] InitCmdFactory -> INFO 03e Endorser and orderer connections initialized
2018-11-07 03:13:15.357 UTC [msp/identity] Sign -> DEBU 03f Sign: plaintext: 0A9C070A5C08011A0C08CBB089DF0510...FD5719255DE61A080A000A000A000A00 
2018-11-07 03:13:15.357 UTC [msp/identity] Sign -> DEBU 040 Sign: digest: 9D4D6D52E6C028072A3053D31AD56AC4D334C323F0464E4026A15AFDDB6AB720 
2018-11-07 03:13:15.466 UTC [channelCmd] executeJoin -> INFO 041 Successfully submitted proposal to join channel

```



## Hyperledger Fabric 实例化测试

>  在上面我们已经拷贝了官方的例子，在 chaincode 下, 下面我们来测试一下


### 安装智能合约


```
# cli 部分  ./chaincode/go/:/opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go
# 为 智能合约的目录 我们约定为这个目录 需要预先创建 


mkdir -p /opt/jicki/chaincode/go

cd /opt/jicki/chaincode/go

# 创建以后~我们拷贝官方的 例子进来，方便后面进行合约测试

cp -r /opt/gopath/src/github.com/hyperledger/fabric/examples/chaincode/go/example0* /opt/jicki/chaincode/go/


# 官方这里有5个例子

[root@localhost jicki]# ls -lt chaincode/go/
总用量 0
drwxr-xr-x 3 root root 75 11月  5 12:32 example05
drwxr-xr-x 3 root root 75 11月  5 12:32 example03
drwxr-xr-x 3 root root 75 11月  5 12:32 example04
drwxr-xr-x 3 root root 47 11月  5 12:32 example01
drwxr-xr-x 3 root root 75 11月  5 12:32 example02



# 如上我们挂载的地址为 github.com/hyperledger/fabric/jicki/chaincode/go


# 注: 这里面的 example02 的 package 为 example02 会报错

Error: could not assemble transaction, err Proposal response was not successful, error code 500, msg failed to execute transaction 819b581ce88604e9b6651764324876f2ca7a47d7aeb7ee307f273af867a4a134: error starting container: error starting container: API error (404): oci runtime error: container_linux.go:247: starting container process caused "exec: \"chaincode\": executable file not found in $PATH"


# 将 chaincode.go  chaincode_test.go 中  package 修改成 main 然后在最下面增加 main()函数

package example02 修改为 package main


# 在最后添加如下:


func main() {
        err := shim.Start(new(SimpleChaincode))
        if err != nil {
                fmt.Printf("Error starting Simple chaincode: %s", err)
        }
}

```



```
# 安装指定合约到 所有的 peer 节点中，每个节点都必须安装一次

# 同样需要先配置变量

export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_ADDRESS=peer0.org1.jicki.cn:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/users/Admin@org1.jicki.cn/msp



# 安装 合约

peer chaincode install -n example2 -p github.com/hyperledger/fabric/jicki/chaincode/go/example02 -v 1.0   



# 输出如下:

2018-11-05 04:34:32.461 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-05 04:34:32.461 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-11-05 04:34:32.462 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-05 04:34:32.462 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-05 04:34:32.463 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:34:32.463 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-05 04:34:32.463 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc42054f730, CONNECTING
2018-11-05 04:34:32.464 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc42054f730, READY
2018-11-05 04:34:32.466 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-05 04:34:32.466 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-05 04:34:32.466 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:34:32.466 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-05 04:34:32.466 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4205ad630, CONNECTING
2018-11-05 04:34:32.467 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4205ad630, READY
2018-11-05 04:34:32.468 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-05 04:34:32.468 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 044 Using default escc
2018-11-05 04:34:32.468 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 045 Using default vscc
2018-11-05 04:34:32.545 UTC [golang-platform] getCodeFromFS -> DEBU 046 getCodeFromFS github.com/hyperledger/fabric/jicki/chaincode/go/example02
2018-11-05 04:34:34.204 UTC [golang-platform] func1 -> DEBU 047 Discarding GOROOT package fmt
2018-11-05 04:34:34.204 UTC [golang-platform] func1 -> DEBU 048 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-11-05 04:34:34.204 UTC [golang-platform] func1 -> DEBU 049 Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-11-05 04:34:34.204 UTC [golang-platform] func1 -> DEBU 04a Discarding GOROOT package strconv
2018-11-05 04:34:34.204 UTC [golang-platform] func1 -> DEBU 04b skipping dir: /opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/cmd
2018-11-05 04:34:34.204 UTC [golang-platform] GetDeploymentPayload -> DEBU 04c done
2018-11-05 04:34:34.204 UTC [container] WriteFileToPackage -> DEBU 04d Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode.go
2018-11-05 04:34:34.206 UTC [container] WriteFileToPackage -> DEBU 04e Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode_test.go
2018-11-05 04:34:34.206 UTC [msp/identity] Sign -> DEBU 04f Sign: plaintext: 0A97070A5B08031A0B08DA90FFDE0510...C7F84F000000FFFFEB6FF30D00280000 
2018-11-05 04:34:34.206 UTC [msp/identity] Sign -> DEBU 050 Sign: digest: 608ADA9B6683C70D87397C3F254A354AEA4EEC4E69C67C5A67A8D25CF0291AFA 
2018-11-05 04:34:34.223 UTC [chaincodeCmd] install -> INFO 051 Installed remotely response:<status:200 payload:"OK" > 

```

```
# 安装指定合约到 所有的 peer 节点中，每个节点都必须安装一次

# 同样需要先配置变量

export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_ADDRESS=peer0.org2.jicki.cn:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp



# 安装 合约

peer chaincode install -n example2 -p github.com/hyperledger/fabric/jicki/chaincode/go/example02 -v 1.0  



# 输出如下:

2018-11-05 04:34:52.684 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-05 04:34:52.684 UTC [msp] Validate -> DEBU 036 MSP Org2MSP validating identity
2018-11-05 04:34:52.685 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-05 04:34:52.685 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-05 04:34:52.685 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-11-05 04:34:52.685 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-05 04:34:52.685 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc420533700, CONNECTING
2018-11-05 04:34:52.686 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc420533700, READY
2018-11-05 04:34:52.688 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-05 04:34:52.688 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-05 04:34:52.688 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-11-05 04:34:52.688 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-05 04:34:52.689 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc420598d60, CONNECTING
2018-11-05 04:34:52.689 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc420598d60, READY
2018-11-05 04:34:52.690 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-05 04:34:52.690 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 044 Using default escc
2018-11-05 04:34:52.690 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 045 Using default vscc
2018-11-05 04:34:52.748 UTC [golang-platform] getCodeFromFS -> DEBU 046 getCodeFromFS github.com/hyperledger/fabric/jicki/chaincode/go/example02
2018-11-05 04:34:53.087 UTC [golang-platform] func1 -> DEBU 047 Discarding GOROOT package fmt
2018-11-05 04:34:53.087 UTC [golang-platform] func1 -> DEBU 048 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-11-05 04:34:53.087 UTC [golang-platform] func1 -> DEBU 049 Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-11-05 04:34:53.087 UTC [golang-platform] func1 -> DEBU 04a Discarding GOROOT package strconv
2018-11-05 04:34:53.088 UTC [golang-platform] func1 -> DEBU 04b skipping dir: /opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/cmd
2018-11-05 04:34:53.088 UTC [golang-platform] GetDeploymentPayload -> DEBU 04c done
2018-11-05 04:34:53.088 UTC [container] WriteFileToPackage -> DEBU 04d Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode.go
2018-11-05 04:34:53.089 UTC [container] WriteFileToPackage -> DEBU 04e Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode_test.go
2018-11-05 04:34:53.090 UTC [msp/identity] Sign -> DEBU 04f Sign: plaintext: 0A9B070A5B08031A0B08ED90FFDE0510...C7F84F000000FFFFEB6FF30D00280000 
2018-11-05 04:34:53.090 UTC [msp/identity] Sign -> DEBU 050 Sign: digest: 2EBCA1E2BB3349F501E9DBE66E02B986333AE1FD3FEFE0AE86C985437FA2905C 
2018-11-05 04:34:53.105 UTC [chaincodeCmd] install -> INFO 051 Installed remotely response:<status:200 payload:"OK" > 

```

### 实例化 Chaincode


> 这里无论多少个 peer 节点, 实例化只需要实例化一次，就可以。


```
# 实例化合约 (未认证)
peer chaincode instantiate -o orderer0.jicki.cn:7050 -C mychannel -n example2 -c '{"Args":["init","A","200","B","500"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" -v 1.0
```




```
# 实例化合约 (已认证)
peer chaincode instantiate -o orderer0.jicki.cn:7050 --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem -C mychannel -n example2 -c '{"Args":["init","A","200","B","500"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" -v 1.0




# 输出如下:

2018-11-07 03:17:30.242 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-07 03:17:30.242 UTC [msp] Validate -> DEBU 036 MSP Org2MSP validating identity
2018-11-07 03:17:30.243 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-07 03:17:30.243 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-07 03:17:30.244 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-11-07 03:17:30.244 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-07 03:17:30.244 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4205e5be0, CONNECTING
2018-11-07 03:17:30.248 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4205e5be0, READY
2018-11-07 03:17:30.249 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-07 03:17:30.249 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-07 03:17:30.249 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-11-07 03:17:30.249 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-07 03:17:30.249 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4205b0780, CONNECTING
2018-11-07 03:17:30.252 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4205b0780, READY
2018-11-07 03:17:30.253 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-07 03:17:30.254 UTC [grpc] DialContext -> DEBU 044 parsed scheme: ""
2018-11-07 03:17:30.254 UTC [grpc] DialContext -> DEBU 045 scheme "" not registered, fallback to default scheme
2018-11-07 03:17:30.254 UTC [grpc] watcher -> DEBU 046 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-11-07 03:17:30.254 UTC [grpc] switchBalancer -> DEBU 047 ClientConn switching balancer to "pick_first"
2018-11-07 03:17:30.254 UTC [grpc] HandleSubConnStateChange -> DEBU 048 pickfirstBalancer: HandleSubConnStateChange: 0xc42029ee50, CONNECTING
2018-11-07 03:17:30.257 UTC [grpc] HandleSubConnStateChange -> DEBU 049 pickfirstBalancer: HandleSubConnStateChange: 0xc42029ee50, READY
2018-11-07 03:17:30.257 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 04a Using default escc
2018-11-07 03:17:30.257 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 04b Using default vscc
2018-11-07 03:17:30.258 UTC [msp/identity] Sign -> DEBU 04c Sign: plaintext: 0AA6070A6608031A0B08CAB289DF0510...324D53500A04657363630A0476736363 
2018-11-07 03:17:30.258 UTC [msp/identity] Sign -> DEBU 04d Sign: digest: 4CADBFD99999E7E35A2C6C221EE15A5D6D1531AEC62B35B70F47B3C638049B67 
2018-11-07 03:17:30.267 UTC [msp/identity] Sign -> DEBU 04e Sign: plaintext: 0AA6070A6608031A0B08CAB289DF0510...A39B1E366186F53CCB32CCEC0B05E851 
2018-11-07 03:17:30.267 UTC [msp/identity] Sign -> DEBU 04f Sign: digest: 382322A56C5F45E0E6432CBFB026334DDA4106A3DFEABE7A14802614F04D5914 

```


### 操作智能合约


```
# query 查询方法


# 查询 A 账户里的余额

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'


# 输出如下:

2018-11-05 04:35:47.211 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-05 04:35:47.212 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-11-05 04:35:47.212 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-05 04:35:47.212 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-05 04:35:47.213 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:35:47.213 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-05 04:35:47.213 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4205355e0, CONNECTING
2018-11-05 04:35:47.214 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4205355e0, READY
2018-11-05 04:35:47.216 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-05 04:35:47.216 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-05 04:35:47.216 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:35:47.216 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-05 04:35:47.216 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4203d2790, CONNECTING
2018-11-05 04:35:47.217 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4203d2790, READY
2018-11-05 04:35:47.218 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-05 04:35:47.218 UTC [msp/identity] Sign -> DEBU 044 Sign: plaintext: 0AA6070A6A08031A0B08A391FFDE0510...706C65321A0A0A0571756572790A0141 
2018-11-05 04:35:47.219 UTC [msp/identity] Sign -> DEBU 045 Sign: digest: F4FF4D4EE24CDFE6D2EF369DEF6AA927717916A8076FAD3A32396C2EE39D70B6 
200


# 可以看到 返回 200
```


```
# 查询 B 账户里的余额

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'


# 输出如下:

2018-11-05 04:36:00.734 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-05 04:36:00.734 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-11-05 04:36:00.735 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-05 04:36:00.735 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-05 04:36:00.735 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:36:00.735 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-05 04:36:00.735 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4203e5470, CONNECTING
2018-11-05 04:36:00.737 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4203e5470, READY
2018-11-05 04:36:00.738 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-05 04:36:00.739 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-05 04:36:00.739 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:36:00.739 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-05 04:36:00.739 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc42039b0c0, CONNECTING
2018-11-05 04:36:00.740 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc42039b0c0, READY
2018-11-05 04:36:00.740 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-05 04:36:00.741 UTC [msp/identity] Sign -> DEBU 044 Sign: plaintext: 0AA7070A6B08031A0C08B091FFDE0510...706C65321A0A0A0571756572790A0142 
2018-11-05 04:36:00.741 UTC [msp/identity] Sign -> DEBU 045 Sign: digest: 32655FF22BB7DBA7FC4D7CF39D1E5D65C4212A61E76E809124BBE6A823304F98 
500

# 可以看到 返回 500

```


```
# invoke 转账方法


# 从A账户 转账 100 个币 到 B 账户 (未认证)


peer chaincode invoke -C mychannel -n example2 -c '{"Args":["invoke", "A", "B", "100"]}'

```




```
# 从A账户 转账 100 个币 到 B 账户 (开启认证)

peer chaincode invoke -C mychannel -n example2 -c '{"Args":["invoke", "A", "B", "100"]}' -o orderer0.jicki.cn:7050 --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem



# 输出如下:

2018-11-07 03:19:54.832 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-07 03:19:54.832 UTC [msp] Validate -> DEBU 036 MSP Org2MSP validating identity
2018-11-07 03:19:54.833 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-07 03:19:54.833 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-07 03:19:54.833 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-11-07 03:19:54.833 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-07 03:19:54.833 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4203b7ce0, CONNECTING
2018-11-07 03:19:54.837 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4203b7ce0, READY
2018-11-07 03:19:54.838 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-07 03:19:54.839 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-07 03:19:54.839 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-11-07 03:19:54.839 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-07 03:19:54.839 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4206c67f0, CONNECTING
2018-11-07 03:19:54.842 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4206c67f0, READY
2018-11-07 03:19:54.843 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-07 03:19:54.843 UTC [grpc] DialContext -> DEBU 044 parsed scheme: ""
2018-11-07 03:19:54.843 UTC [grpc] DialContext -> DEBU 045 scheme "" not registered, fallback to default scheme
2018-11-07 03:19:54.843 UTC [grpc] watcher -> DEBU 046 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-11-07 03:19:54.843 UTC [grpc] switchBalancer -> DEBU 047 ClientConn switching balancer to "pick_first"
2018-11-07 03:19:54.843 UTC [grpc] HandleSubConnStateChange -> DEBU 048 pickfirstBalancer: HandleSubConnStateChange: 0xc4201b2390, CONNECTING
2018-11-07 03:19:54.846 UTC [grpc] HandleSubConnStateChange -> DEBU 049 pickfirstBalancer: HandleSubConnStateChange: 0xc4201b2390, READY
2018-11-07 03:19:54.847 UTC [msp/identity] Sign -> DEBU 04a Sign: plaintext: 0AAB070A6B08031A0C08DAB389DF0510...6E766F6B650A01410A01420A03313030 
2018-11-07 03:19:54.847 UTC [msp/identity] Sign -> DEBU 04b Sign: digest: E9E5E9E5DC7C48C05F60D8314820AB7160C5303A4B037BAD03EE061AB056FC10 
2018-11-07 03:19:54.863 UTC [msp/identity] Sign -> DEBU 04c Sign: plaintext: 0AAB070A6B08031A0C08DAB389DF0510...52C77A337E38F86AFD2A4A02F3E9D10F 
2018-11-07 03:19:54.863 UTC [msp/identity] Sign -> DEBU 04d Sign: digest: 74B9DE72CE632AC6CA283A39CF95AADD3A252CC71EF5C213E21A14CFDAC07589 
2018-11-07 03:19:54.872 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> DEBU 04e ESCC invoke result: version:1 response:<status:200 > payload:"\n MM7\026q4R\244\332\217i^\345\"2L\227\336\336G`_m3m\233f$\206/\226\025\022d\nL\0220\n\010example2\022$\n\007\n\001A\022\002\010\001\n\007\n\001B\022\002\010\001\032\006\n\001A\032\0010\032\010\n\001B\032\003150\022\030\n\004lscc\022\020\n\016\n\010example2\022\002\010\001\032\003\010\310\001\"\017\022\010example2\032\0031.0" endorsement:<endorser:"\n\007Org2MSP\022\216\006-----BEGIN CERTIFICATE-----\nMIICFDCCAbqgAwIBAgIRANsTa6zjKyWSP3Tjh+32N5UwCgYIKoZIzj0EAwIwZzEL\nMAkGA1UEBhMCQ04xEjAQBgNVBAgTCUd1YW5nRG9uZzERMA8GA1UEBxMIU2hlblpo\nZW4xFjAUBgNVBAoTDW9yZzIuamlja2kubWUxGTAXBgNVBAMTEGNhLm9yZzIuamlj\na2kubWUwHhcNMTgxMTA1MDM0OTAwWhcNMjgxMTAyMDM0OTAwWjBhMQswCQYDVQQG\nEwJDTjESMBAGA1UECBMJR3VhbmdEb25nMREwDwYDVQQHEwhTaGVuWmhlbjENMAsG\nA1UECxMEcGVlcjEcMBoGA1UEAxMTcGVlcjAub3JnMi5qaWNraS5tZTBZMBMGByqG\nSM49AgEGCCqGSM49AwEHA0IABLB5tlyikwEe9Ec+n7kSYpwOWnsM+IMJ0xa14kdg\nN3N276hLr/EeBEuvfpgPwi7cl3nFnrMmRuT/zQbiafggdmCjTTBLMA4GA1UdDwEB\n/wQEAwIHgDAMBgNVHRMBAf8EAjAAMCsGA1UdIwQkMCKAILy8haOZJxVQLH5+fSG3\nkmeKnQNmilK7GKDBCGLUUiLDMAoGCCqGSM49BAMCA0gAMEUCIQDd7AZB9w3eY8Ms\nomoKMSYB0T3TS9iGJY+Pr25GLNpcpAIgJzxfN9X7HiPJShC0jYCCAUdCsEXB5t2l\njsWDuxFMujk=\n-----END CERTIFICATE-----\n" signature:"0E\002!\000\257#\"$2\233\032\304Y\245Nk\377`\321W\357\235J\225\r\247\370\030V\313N\314\347+\255\306\002 m\017\205\000\017\236g\217\336\211\357\205\372\006=&R\307z3~8\370j\375*J\002\363\351\321\017" > 
2018-11-07 03:19:54.872 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 04f Chaincode invoke successful. result: status:200 


# 可以看到返回 invoke successful. result: status:200 成功
```


```
# 这里再查询 A 与 B 的账户

# A 账户余额 

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'  


# 输出如下:

2018-11-05 04:36:26.574 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-05 04:36:26.574 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-11-05 04:36:26.575 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-05 04:36:26.575 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-05 04:36:26.575 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:36:26.575 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-05 04:36:26.575 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc42066b6f0, CONNECTING
2018-11-05 04:36:26.577 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc42066b6f0, READY
2018-11-05 04:36:26.578 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-05 04:36:26.578 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-05 04:36:26.579 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:36:26.579 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-05 04:36:26.579 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4206dfdf0, CONNECTING
2018-11-05 04:36:26.579 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4206dfdf0, READY
2018-11-05 04:36:26.580 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-05 04:36:26.581 UTC [msp/identity] Sign -> DEBU 044 Sign: plaintext: 0AA7070A6B08031A0C08CA91FFDE0510...706C65321A0A0A0571756572790A0141 
2018-11-05 04:36:26.581 UTC [msp/identity] Sign -> DEBU 045 Sign: digest: 0BA86741E22D8C11EBADBB4C753849CF6EF9E3FFD0AFB713BB8E6DD5C80043D8 
100

```


```
# B 账户余额 

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'   



# 输出如下:

2018-11-05 04:36:39.259 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-11-02 03:49:00 +0000 UTC
2018-11-05 04:36:39.259 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-11-05 04:36:39.260 UTC [grpc] DialContext -> DEBU 037 parsed scheme: ""
2018-11-05 04:36:39.260 UTC [grpc] DialContext -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-11-05 04:36:39.260 UTC [grpc] watcher -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:36:39.260 UTC [grpc] switchBalancer -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-11-05 04:36:39.260 UTC [grpc] HandleSubConnStateChange -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4203b5470, CONNECTING
2018-11-05 04:36:39.261 UTC [grpc] HandleSubConnStateChange -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4203b5470, READY
2018-11-05 04:36:39.263 UTC [grpc] DialContext -> DEBU 03d parsed scheme: ""
2018-11-05 04:36:39.263 UTC [grpc] DialContext -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-11-05 04:36:39.263 UTC [grpc] watcher -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-11-05 04:36:39.263 UTC [grpc] switchBalancer -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-11-05 04:36:39.263 UTC [grpc] HandleSubConnStateChange -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4205e8d50, CONNECTING
2018-11-05 04:36:39.264 UTC [grpc] HandleSubConnStateChange -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4205e8d50, READY
2018-11-05 04:36:39.265 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-11-05 04:36:39.265 UTC [msp/identity] Sign -> DEBU 044 Sign: plaintext: 0AA6070A6A08031A0B08D791FFDE0510...706C65321A0A0A0571756572790A0142 
2018-11-05 04:36:39.265 UTC [msp/identity] Sign -> DEBU 045 Sign: digest: 6792CE0A74737C90A6ABC66A99C414B11E560D2AFEDBFFCD3C928CCE61B7597B 
600

```

```
# 查看 peer0.org1.jicki.cn 节点里 生成的容器

[root@localhost jicki]# docker ps -a
CONTAINER ID        IMAGE                                                                                                     COMMAND                  CREATED             STATUS              PORTS                                                                                         NAMES
526b6b46a60e        jicki-peer0.org1.jicki.cn-example2-1.0-58e362edb293b97212b572bd603337608795d8168a8a3632d2a14ac12fa6ae70   "chaincode -peer.add…"   About a minute ago   Up About a minute                                                                                                                                            jicki-peer0.org1.jicki.cn-example2-1.0

```


## Hyperledger Fabric 操作命令


### peer 命令


```
peer chaincode          # 对链进行操作
peer channel            # channel相关操作
peer logging            # 设置日志级别
peer node               # 启动、管理节点
peer version            # 查看版本信息

```

> upgrade 更新合约 更新合约相当于将合约重新实例化，并带有一个新的版本号。
>
> 更新合约之前，需要在所有的 peer节点 上安装(install)最新的合约，并使用新的版本号。

```
# 更新合约

# 首先安装(install)新的合约, 以本文为例, chaincode_example02, 初次安装版本号为 1.0 

peer chaincode install -n example2 -p github.com/hyperledger/fabric/jicki/chaincode/go/chaincode_example02 -v 1.1


# 更新版本为 1.1 的合约 (未开启认证)
peer chaincode upgrade -o orderer0.jicki.cn:7050 -C mychannel -n example2 -c '{"Args":["init","A","100","B","50"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" -v 1.1 


# 更新版本为 1.1 的合约 (开启认证)
peer chaincode upgrade -o orderer0.jicki.cn:7050 -C mychannel -n example2 -c '{"Args":["init","A","100","B","50"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" -v 1.1 -o orderer0.jicki.cn:7050 --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem


# 旧版本的合约, 目前，fabric不支持合约的启动与暂停。要暂停或删除合约，只能到peer上手动删除容器。

```






```
# 查看 已经创建的 通道 (channel)

peer channel  list


# 查看通道(channel) 的状态 -c(小写) 加 通道名称

peer channel getinfo -c mychannel


# 查看已经 安装的 智能合约(chincode)

peer chaincode  list --installed


# 查看已经 实例化的 智能合约(chincode) 需要使用 -C(大写) 加通道名称

peer chaincode -C mychannel list --instantiated

```




# 配置 Hyperledger Fabric balance-transfer


## 安装 node.js

```
# 安装NodeJS
 curl --silent --location https://rpm.nodesource.com/setup_8.x | sudo bash -
 
 yum install -y nodejs
 
 
 
# 验证

node -v
v8.11.4


npm -v
5.6.0


# 更改 npm 源为 taobao 源

npm install node-gyp --registry=https://registry.npm.taobao.org

npm install node-pre-gyp --registry=https://registry.npm.taobao.org

npm install grpc --registry=https://registry.npm.taobao.org

npm install --registry=https://registry.npm.taobao.org

npm rebuild


# 安装一下 jq 

 yum install jq -y


```

## 下载源码

```
cd /opt/jicki

git clone https://github.com/hyperledger/fabric-samples.git

mv fabric-samples/balance-transfer .

```

## 修改配置文件


```

mv network-config.yaml network-config.yaml-bak


# 增加 network-config 文件


vi network-config.yaml 


---
name: "balance-transfer"
x-type: "hlfv1"
description: "Balance Transfer Network"
version: "1.0"
channels:
  mychannel:
    orderers:
      - orderer0.jicki.cn
    peers:
      peer0.org1.jicki.cn:
        endorsingPeer: true
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: true

      #peer1.org1.jicki.cn:
      #  endorsingPeer: false
      #  chaincodeQuery: true
      #  ledgerQuery: true
      #  eventSource: false

      peer0.org2.jicki.cn:
        endorsingPeer: true
        chaincodeQuery: true
        ledgerQuery: true
        eventSource: true

      #peer1.org2.jicki.cn:
      #  endorsingPeer: false
      #  chaincodeQuery: true
      #  ledgerQuery: true
      #  eventSource: false

    chaincodes:
      - mycc:v0

organizations:
  Org1:
    mspid: Org1MSP

    peers:
      - peer0.org1.jicki.cn
      #- peer1.org1.jicki.cn

    certificateAuthorities:
      - ca-org1
    adminPrivateKey:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.jicki.cn/users/Admin@org1.jicki.cn/msp/keystore/a9666f561d211e7b7cc170bfe854721431a1038b7914a67555d82dcf6b9eaaf8_sk
    signedCert:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.jicki.cn/users/Admin@org1.jicki.cn/msp/signcerts/Admin@org1.jicki.cn-cert.pem

  Org2:
    mspid: Org2MSP
    peers:
      - peer0.org2.jicki.cn
      #- peer1.org2.jicki.cn

    certificateAuthorities:
      - ca-org2
    adminPrivateKey:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp/keystore/d402b684b807080653511978a51fcd2326668156cd8a10fb612b80bc49c9b354_sk
    signedCert:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp/signcerts/Admin@org2.jicki.cn-cert.pem

orderers:
  orderer0.jicki.cn:
    url: grpcs://localhost:7050

    grpcOptions:
      ssl-target-name-override: orderer0.jicki.cn

    tlsCACerts:
      path: artifacts/channel/crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/tls/ca.crt

peers:
  peer0.org1.jicki.cn:
    # this URL is used to send endorsement and query requests
    url: grpcs://localhost:7051

    grpcOptions:
      ssl-target-name-override: peer0.org1.jicki.cn
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/ca.crt

  #peer1.org1.jicki.cn:
  #  url: grpcs://localhost:7056
  #  grpcOptions:
  #    ssl-target-name-override: peer1.org1.jicki.cn
  #  tlsCACerts:
  #    path: artifacts/channel/crypto-config/peerOrganizations/org1.jicki.cn/peers/peer1.org1.jicki.cn/tls/ca.crt

  peer0.org2.jicki.cn:
    url: grpcs://localhost:8051
    grpcOptions:
      ssl-target-name-override: peer0.org2.jicki.cn
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt

  #peer1.org2.jicki.cn:
  #  url: grpcs://localhost:8056
  #  eventUrl: grpcs://localhost:8058
  #  grpcOptions:
  #    ssl-target-name-override: peer1.org2.jicki.cn
  #  tlsCACerts:
  #    path: artifacts/channel/crypto-config/peerOrganizations/org2.jicki.cn/peers/peer1.org2.jicki.cn/tls/ca.crt

certificateAuthorities:
  ca-org1:
    url: https://localhost:7054
    httpOptions:
      verify: false
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org1.jicki.cn/ca/ca.org1.jicki.cn-cert.pem

    registrar:
      - enrollId: admin
        enrollSecret: adminpw
    caName: ca-org1

  ca-org2:
    url: https://localhost:8054
    httpOptions:
      verify: false
    tlsCACerts:
      path: artifacts/channel/crypto-config/peerOrganizations/org2.jicki.cn/ca/ca.org2.jicki.cn-cert.pem
    registrar:
      - enrollId: admin
        enrollSecret: adminpw
    caName: ca-org2

```



## 拷贝证书文件


```
cd /opt/jicki/balance-transfer/artifacts/

mv channel channel-bak

mkdir channel

# 拷贝自行生成的证书文件以及tx，等文件 


[root@localhost channel]# ls -lt
总用量 56
-rw-r--r-- 1 root root   281 11月  5 16:40 Org1MSPanchors.tx
-rw-r--r-- 1 root root   281 11月  5 16:40 Org2MSPanchors.tx
-rw-r--r-- 1 root root   346 11月  5 16:40 channel.tx
-rw-r--r-- 1 root root 12479 11月  5 16:40 genesis.block
-rw-r--r-- 1 root root 15407 11月  5 16:39 mychannel.block
-rw-r--r-- 1 root root   645 11月  5 16:39 crypto-config.yaml
-rw-r--r-- 1 root root  3859 11月  5 16:39 configtx.yaml
drwxr-xr-x 4 root root    69 11月  5 16:38 crypto-config

```



## 启动 balance-transfer


> runApp.sh 脚本里包含了 启动 ca 节点 以及 peer 节点的docker-compose
> 
> 如上我们已经启动过了，只需要直接运行 node app 既可，所以不需要用到此脚本


```
# 安装依赖

npm install

```


```
# 导入环境变量

PORT=4000 
HOST=192.168.168.100
```




```
# 启动

node app

```




## 测试 调用 API



```
# 运行 testAPIs.sh 可全部API跑一次

# 本文中只运行了 2个 peer 

# 所以需要 编辑 testAPIs.sh 修改文件中 删除 peer1.org1.jicki.cn  以及 peer1.org2.jicki.cn

# 修改  "channelConfigPath":"../artifacts/channel/mychannel.tx" 中 mychannel.tx 为 channel.tx

```



### 创建用户

```

# 创建用户

curl -s -X POST http://localhost:4000/users -H "content-type: application/x-www-form-urlencoded" -d 'username=jicki&orgName=Org1'


{"success":true,"secret":"","message":"jicki enrolled Successfully","token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NDE2MTIwNzAsInVzZXJuYW1lIjoiSmltIiwib3JnTmFtZSI6Ik9yZzEiLCJpYXQiOjE1NDE1NzYwNzB9.pQZAqkeRW7vtwvNSrSTy4SKsVt5yFGafzlWTQjjONWE"}



# 注册成功以后~~取 token 部分,配置成环境变量, 如下操作需用使用此 token

ORG1_TOKEN=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NDE2MTIwNzAsInVzZXJuYW1lIjoiSmltIiwib3JnTmFtZSI6Ik9yZzEiLCJpYXQiOjE1NDE1NzYwNzB9.pQZAqkeRW7vtwvNSrSTy4SKsVt5yFGafzlWTQjjONWE

```


### 创建 channel

```
# 创建 channel 
# 此处 channelName 如果存在会失败~ 报如下错误

response ::{"status":"BAD_REQUEST","info":"error authorizing update: error validating ReadSet: readset expected key [Group]  /Channel/Application at version 0, but got version 1"}




curl -s -X POST \
  http://localhost:4000/channels \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "channelName":"mychannel",
    "channelConfigPath":"../artifacts/channel/channel.tx"
}'




# 关于 channelName  需要跟 创建 channel.tx 时的 channelID 对应，否则报如下错误:


[2018-11-09 12:03:28.276] [DEBUG] Create-Channel -  response ::{"status":"BAD_REQUEST","info":"Failing initial channel config creation: mismatched channel IDs: 'mychannel' != 'youchannel'"}
[2018-11-09 12:03:28.277] [ERROR] Create-Channel - 
!!!!!!!!! Failed to create the channel 'youchannel' !!!!!!!!!


[2018-11-09 12:03:28.277] [ERROR] Create-Channel - Error: Failed to create the channel 'youchannel'



```



### 加入 channel

```
# 加入 ORG1 加入 channel 

# 如果已加入 channel 报如下错误:

[DEBUG] Join-Channel - Join Channel R E S P O N S E : [null,[{"status":500,"payload":{"type":"Buffer","data":[]},"isProposalResponse":true}]]
[ERROR] Join-Channel - Failed to join peer to the channel mychannel
[ERROR] Join-Channel - Failed to join all peers to channel. cause:Failed to join peer to the channel mychannel






curl -s -X POST \
  http://localhost:4000/channels/mychannel/peers \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.jicki.cn"]
}'

```



### 安装 chaincode



```
# 为 ORG1 安装 chaincode
# 已存在报如下错误:

[2018-11-07 16:10:44.011] [ERROR] install-chaincode - TypeError: proposalResponses.toJSON is not a function
    at Object.installChaincode (/opt/jicki/balance-transfer/app/install-chaincode.js:58:66)
    at <anonymous>
[2018-11-07 16:10:44.011] [ERROR] install-chaincode - Failed to install due to:TypeError: proposalResponses.toJSON is not a function





curl -s -X POST \
  http://localhost:4000/chaincodes \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.jicki.cn"],
    "chaincodeName":"mycc",
    "chaincodePath":"github.com/example_cc/go",
    "chaincodeType": "golang",
    "chaincodeVersion":"v0"
}'

```


### 实例化chaincode


```
# 如果已经实例化 报如下错误:

[2018-11-07 16:12:10.040] [ERROR] instantiate-chaincode - instantiate proposal was bad
[2018-11-07 16:12:10.040] [DEBUG] instantiate-chaincode - Failed to send Proposal and receive all good ProposalResponse
[2018-11-07 16:12:10.040] [ERROR] instantiate-chaincode - Failed to instantiate. cause:Failed to send Proposal and receive all good ProposalResponse





curl -s -X POST \
  http://localhost:4000/channels/mychannel/chaincodes \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.jicki.cn"],
    "chaincodeName":"mycc",
    "chaincodeVersion":"v0",
    "chaincodeType": "golang",
    "args":["a","100","b","200"]
}'


```





### 查询已创建的 channel


```

curl -s -X GET \
  "http://localhost:4000/channels?peer=peer0.org1.jicki.cn" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"



# 返回如下信息:

[2018-11-07 16:19:29.592] [DEBUG] Query - <<< channels >>>
[2018-11-07 16:19:29.592] [DEBUG] Query - [ 'channel id: mychannel' ]


```



### 查询 chaincode


```
#  %5B%22a%22%5D Escape 加密以后 等于 ["a"]
#  %5B%22b%22%5D Escape 加密以后 等于 ["b"]


# 查询 a 的值

curl -s -X GET \
  "http://localhost:4000/channels/mychannel/chaincodes/mycc?peer=peer0.org1.jicki.cn&fcn=query&args=%5B%22a%22%5D" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"

# 输出如下:

2018-11-07 16:33:29.266] [INFO] Query - a now has 100 after the move



# 查询 b 的值


curl -s -X GET \
  "http://localhost:4000/channels/mychannel/chaincodes/mycc?peer=peer0.org1.jicki.cn&fcn=query&args=%5B%22b%22%5D" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"



# 输出如下:

[2018-11-07 16:34:12.378] [INFO] Query - b now has 200 after the move


```





### 操作交易


```


# 从 A账号 转账 10 到 B账号中


# 必须配置 org1 与 org2 节点的 peers 否则报错:

[2018-11-07 17:17:53.267] [ERROR] invoke-chaincode - The invoke chaincode transaction was invalid, code:ENDORSEMENT_POLICY_FAILURE
[2018-11-07 17:17:53.268] [ERROR] invoke-chaincode - Error: The invoke chaincode transaction was invalid, code:ENDORSEMENT_POLICY_FAILURE
[2018-11-07 17:17:53.268] [ERROR] invoke-chaincode - Failed to invoke chaincode. cause:Error: The invoke chaincode transaction was invalid, code:ENDORSEMENT_POLICY_FAILURE





curl -s -X POST \
  http://localhost:4000/channels/mychannel/chaincodes/mycc \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json" \
  -d '{
    "peers": ["peer0.org1.jicki.cn", "peer0.org2.jicki.cn"],
    "fcn":"move",
    "args":["a","b","10"]
}'




# 输出如下:

POST invoke chaincode on peers of Org1 and Org2

Transaction ID is 70a5157704b950cca09a6a46f5be7fca61355b43ed83f3d9a5b633f3e38b3619

```







### 查询 chainInfo



```

curl -s -X GET \
  "http://localhost:4000/channels/mychannel?peer=peer0.org1.jicki.cn" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"
  
  
  
# 返回如下信息:

[2018-11-07 16:14:55.648] [DEBUG] Query - { height: Long { low: 5, high: 0, unsigned: true },
  currentBlockHash: 
   ByteBuffer {
     buffer: <Buffer 08 05 12 20 c0 c4 f3 65 a3 8a 66 d4 1e ff a4 45 3e 1c e6 2b 90 d3 38 4f 18 3f d5 b3 38 76 0e 26 30 32 e0 f9 1a 20 da 35 eb d4 1b 11 b2 7f 1a 07 c5 30 ... >,
     offset: 4,
     markedOffset: -1,
     limit: 36,
     littleEndian: true,
     noAssert: false },
  previousBlockHash: 
   ByteBuffer {
     buffer: <Buffer 08 05 12 20 c0 c4 f3 65 a3 8a 66 d4 1e ff a4 45 3e 1c e6 2b 90 d3 38 4f 18 3f d5 b3 38 76 0e 26 30 32 e0 f9 1a 20 da 35 eb d4 1b 11 b2 7f 1a 07 c5 30 ... >,
     offset: 38,
     markedOffset: -1,
     limit: 70,
     littleEndian: true,
     noAssert: false } }
     
```


### 查询已安装 chaincode


```

curl -s -X GET \
  "http://localhost:4000/chaincodes?peer=peer0.org1.jicki.cn&type=installed" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"



# 返回如下信息:

[2018-11-07 16:17:01.758] [DEBUG] Query - <<< Installed Chaincodes >>>
[2018-11-07 16:17:01.758] [DEBUG] Query - name: mycc, version: v0, path: github.com/example_cc/go

```



### 查询实例化 chaincode


```

curl -s -X GET \
  "http://localhost:4000/chaincodes?peer=peer0.org1.jicki.cn&type=instantiated" \
  -H "authorization: Bearer $ORG1_TOKEN" \
  -H "content-type: application/json"



# 返回如下信息:

[2018-11-07 16:18:06.494] [DEBUG] Query - <<< Installed Chaincodes >>>
[2018-11-07 16:18:06.494] [DEBUG] Query - name: mycc, version: v0, path: github.com/example_cc/go


```




