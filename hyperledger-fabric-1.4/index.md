# hyperledger-fabric v 1.4


> fabric v1.4 , 单机 多节点 kafka 手动部署, 所有服务均 开启 SSL 认证。



# 部署 hyperledger-fabric v1.4



## 环境规划

> 相关hostname 必须配置 dns 
>
> 关于 orderer 集群
>
> 当orderer 向peer节点提交Transaction的时候，peer节点会得到或返回一个读写集结果，该结果会发送给orderer节点进行共识和排序，此时如果orderer节点突然down掉，就会使请求服务失效而引发的数据丢失等问题，且目前的sdk对orderer发送的Transaction的回调会占用极长的时间，当大批量数据导入的时候该回调可认为不可用。

|节点标识|hostname|IP|开放端口|系统|
|--------|--------|--|--------|----------|---|--|
|orderer0节点|orderer0.jicki.cn|192.168.100.100|7050|CentOS 7 x64|
|peer0节点|peer0.org1.jicki.cn|192.168.100.100|7051, 7052, 7053|CentOS 7 x64|
|peer0节点|peer0.org2.jicki.cn|192.168.100.100|7051, 7052, 7053|CentOS 7 x64|
|zk0节点|zookeeper0|192.168.100.100|2181|CentOS 7 x64|
|zk1节点|zookeeper1|192.168.100.100|2181|CentOS 7 x64|
|zk2节点|zookeeper2|192.168.100.100|2181|CentOS 7 x64|
|kafka0节点|kafka0|192.168.100.100|9092|CentOS 7 x64|
|kafka1节点|kafka1|192.168.100.100|9092|CentOS 7 x64|
|kafka2节点|kafka2|192.168.100.100|9092|CentOS 7 x64|


## 官方地址

> 文档以官方文档为主 http://hyperledger-fabric.readthedocs.io/en/release-1.4/prereqs.html

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
Client: Docker Engine - Community
 Version:           19.03.4
 API version:       1.40
 Go version:        go1.12.10
 Git commit:        9013bf583a
 Built:             Fri Oct 18 15:52:22 2019
 OS/Arch:           linux/amd64
 Experimental:      false

Server: Docker Engine - Community
 Engine:
  Version:          19.03.4
  API version:      1.40 (minimum version 1.12)
  Go version:       go1.12.10
  Git commit:       9013bf583a
  Built:            Fri Oct 18 15:50:54 2019
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.2.10
  GitCommit:        b34a5c8af56e510852c35414db4c1f4fa6172339
 runc:
  Version:          1.0.0-rc8+dev
  GitCommit:        3e425f80a8c931f88e6d94a8c831b9d5aa481657
 docker-init:
  Version:          0.18.0
  GitCommit:        fec3683
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

curl -L "https://github.com/docker/compose/releases/download/1.25.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose


```



```
docker-compose version
docker-compose version 1.24.1, build 4667896
docker-py version: 3.7.3
CPython version: 2.7.5
OpenSSL version: OpenSSL 1.0.2k-fips  26 Jan 2017
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
git checkout -b release-1.4 remotes/origin/release-1.4


# 文件如下:

[root@localhost fabric]# ls -lt
total 1420
drwxr-xr-x  3 root root   4096 Oct 29 11:19 sampleconfig
drwxr-xr-x  9 root root   4096 Oct 29 09:48 vendor
drwxr-xr-x  8 root root   4096 Oct 29 09:48 token
-rw-r--r--  1 root root    495 Oct 29 09:48 tox.ini
drwxr-xr-x  2 root root   4096 Oct 29 09:48 unit-test
-rw-r--r--  1 root root   3816 Oct 29 09:48 testingInfo.rst
-rw-r--r--  1 root root 438053 Oct 29 09:48 test-pyramid.png
drwxr-xr-x  2 root root   4096 Oct 29 09:48 scripts
-rw-r--r--  1 root root    316 Oct 29 09:48 settings.gradle
drwxr-xr-x  3 root root   4096 Oct 29 09:48 release
drwxr-xr-x  2 root root   4096 Oct 29 09:48 release_notes
drwxr-xr-x 14 root root   4096 Oct 29 09:48 protos
drwxr-xr-x 11 root root   4096 Oct 29 09:48 peer
drwxr-xr-x  6 root root   4096 Oct 29 09:48 orderer
drwxr-xr-x  6 root root   4096 Oct 29 09:48 msp
drwxr-xr-x 13 root root   4096 Oct 29 09:48 integration
drwxr-xr-x  2 root root   4096 Oct 29 09:48 idemix
drwxr-xr-x  8 root root   4096 Oct 29 09:48 images
-rw-r--r--  1 root root   2999 Oct 29 09:48 gotools.mk
drwxr-xr-x 17 root root   4096 Oct 29 09:48 gossip
drwxr-xr-x  7 root root   4096 Oct 29 09:48 examples
drwxr-xr-x  5 root root   4096 Oct 29 09:48 docs
drwxr-xr-x  7 root root   4096 Oct 29 09:48 discovery
-rw-r--r--  1 root root   3355 Oct 29 09:48 docker-env.mk
drwxr-xr-x  4 root root   4096 Oct 29 09:48 devenv
drwxr-xr-x 24 root root   4096 Oct 29 09:48 core
drwxr-xr-x 28 root root   4096 Oct 29 09:48 common
drwxr-xr-x  4 root root   4096 Oct 29 09:48 cmd
-rw-r--r--  1 root root     14 Oct 29 09:48 ci.properties
drwxr-xr-x  9 root root   4096 Oct 29 09:48 bccsp
-rw-r--r--  1 root root  29937 Oct 29 09:48 Gopkg.lock
-rw-r--r--  1 root root   3849 Oct 29 09:48 Gopkg.toml
-rw-r--r--  1 root root  11358 Oct 29 09:48 LICENSE
-rwxr-xr-x  1 root root  17474 Oct 29 09:48 Makefile
-rw-r--r--  1 root root   7391 Oct 29 09:48 README.md
-rw-r--r--  1 root root   1035 Oct 29 09:48 SECURITY.md
-rw-r--r--  1 root root 804634 Oct 29 09:48 CHANGELOG.md
-rw-r--r--  1 root root    597 Oct 29 09:48 CODE_OF_CONDUCT.md
-rw-r--r--  1 root root    661 Oct 29 09:48 CONTRIBUTING.md

```


## 生成 Hyperledger Fabric 证书


> 证书生成只需要生成一次，这里只在第一个节点配置


```
# 下载官方证书生成软件(均为二进制文件)
# 官方离线下载地址为 https://github.com/hyperledger/fabric/releases


# 选择相应版本 CentOS 选择 linux-amd64-1.4.0  Mac 选择 darwin-amd64-1.4.0


# 下载地址为: https://github.com/hyperledger/fabric/releases/download/v1.4.0/hyperledger-fabric-linux-amd64-1.4.0.tar.gz


mkdir /opt/jicki/

cd /opt/jicki

wget https://github.com/hyperledger/fabric/releases/download/v1.4.0/hyperledger-fabric-linux-amd64-1.4.0.tar.gz

tar zxvf hyperledger-fabric-linux-amd64-1.4.0.tar.gz



# 解压后是 一个 bin 与 一个 config 目录

[root@localhost jicki]# tree
.
└── bin
    ├── configtxgen
    ├── configtxlator
    ├── cryptogen
    ├── discover
    ├── get-docker-images.sh
    ├── idemixgen
    ├── orderer
    └── peer

1 directory, 8 files
 
 
# 为方便使用 我们配置一个 环境变量

vi /etc/profile


# fabric env
export PATH=$PATH:/opt/jicki/bin


# 使文件生效
source /etc/profile

```

```
# 创建 cryptogen.yaml 文件


OrdererOrgs:
  - Name: Orderer
    Domain: jicki.cn
    CA:
        Country: CN
        Province: GuangDong
        Locality: ShenZhen
    Specs:
      - Hostname: orderer0
      
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

[root@localhost jicki]# cryptogen generate --config=./cryptogen.yaml
org1.jicki.cn
org2.jicki.cn

# 生成一个 crypto-config 证书目录

[root@payment jicki]# tree crypto-config
crypto-config
├── ordererOrganizations
│   └── jicki.cn
│       ├── ca
│       │   ├── 87fcad73e61dbc5e267d0b56e991e2ef445407ddf89924debc299cf42dde53aa_sk
│       │   └── ca.jicki.cn-cert.pem
│       ├── msp
│       │   ├── admincerts
│       │   │   └── Admin@jicki.cn-cert.pem
│       │   ├── cacerts
│       │   │   └── ca.jicki.cn-cert.pem
│       │   └── tlscacerts
│       │       └── tlsca.jicki.cn-cert.pem
│       ├── orderers
│       │   └── orderer0.jicki.cn
│       │       ├── msp
│       │       │   ├── admincerts
│       │       │   │   └── Admin@jicki.cn-cert.pem
│       │       │   ├── cacerts
│       │       │   │   └── ca.jicki.cn-cert.pem
│       │       │   ├── keystore
│       │       │   │   └── 8b8f847af6be4f902f4a85236155a0b9ee37d17edee74ee56212d84cb4b52219_sk
│       │       │   ├── signcerts
│       │       │   │   └── orderer0.jicki.cn-cert.pem
│       │       │   └── tlscacerts
│       │       │       └── tlsca.jicki.cn-cert.pem
│       │       └── tls
│       │           ├── ca.crt
│       │           ├── server.crt
│       │           └── server.key
│       ├── tlsca
│       │   ├── d13f753996d547371bcecc387472e88b95cc790dbcbb59a914f6aa05531e8a18_sk
│       │   └── tlsca.jicki.cn-cert.pem
│       └── users
│           └── Admin@jicki.cn
│               ├── msp
│               │   ├── admincerts
│               │   │   └── Admin@jicki.cn-cert.pem
│               │   ├── cacerts
│               │   │   └── ca.jicki.cn-cert.pem
│               │   ├── keystore
│               │   │   └── 7dfa64d80276527ed1c4ffd030c8b1f4fda213c85396ac3e06794d3957d825bc_sk
│               │   ├── signcerts
│               │   │   └── Admin@jicki.cn-cert.pem
│               │   └── tlscacerts
│               │       └── tlsca.jicki.cn-cert.pem
│               └── tls
│                   ├── ca.crt
│                   ├── client.crt
│                   └── client.key
└── peerOrganizations
    ├── org1.jicki.cn
    │   ├── ca
    │   │   ├── 3a233ccbd6706cee4c66a320e43bedad3e72b6f68e831ce121f35760eb1ac275_sk
    │   │   └── ca.org1.jicki.cn-cert.pem
    │   ├── msp
    │   │   ├── admincerts
    │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │   │   ├── cacerts
    │   │   │   └── ca.org1.jicki.cn-cert.pem
    │   │   ├── config.yaml
    │   │   └── tlscacerts
    │   │       └── tlsca.org1.jicki.cn-cert.pem
    │   ├── peers
    │   │   ├── peer0.org1.jicki.cn
    │   │   │   ├── msp
    │   │   │   │   ├── admincerts
    │   │   │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │   │   │   │   ├── cacerts
    │   │   │   │   │   └── ca.org1.jicki.cn-cert.pem
    │   │   │   │   ├── config.yaml
    │   │   │   │   ├── keystore
    │   │   │   │   │   └── eecd3dec5e6ba609a931d94bf0a1fb9defe047e68e1437fd1fec5fcfbe7dea23_sk
    │   │   │   │   ├── signcerts
    │   │   │   │   │   └── peer0.org1.jicki.cn-cert.pem
    │   │   │   │   └── tlscacerts
    │   │   │   │       └── tlsca.org1.jicki.cn-cert.pem
    │   │   │   └── tls
    │   │   │       ├── ca.crt
    │   │   │       ├── server.crt
    │   │   │       └── server.key
    │   │   └── peer1.org1.jicki.cn
    │   │       ├── msp
    │   │       │   ├── admincerts
    │   │       │   │   └── Admin@org1.jicki.cn-cert.pem
    │   │       │   ├── cacerts
    │   │       │   │   └── ca.org1.jicki.cn-cert.pem
    │   │       │   ├── config.yaml
    │   │       │   ├── keystore
    │   │       │   │   └── d5a32a61b164604e6352a783f843e00c46ca2dfcefbc1b78c3ea14536483169b_sk
    │   │       │   ├── signcerts
    │   │       │   │   └── peer1.org1.jicki.cn-cert.pem
    │   │       │   └── tlscacerts
    │   │       │       └── tlsca.org1.jicki.cn-cert.pem
    │   │       └── tls
    │   │           ├── ca.crt
    │   │           ├── server.crt
    │   │           └── server.key
    │   ├── tlsca
    │   │   ├── d17dbbb6206ef972706c17f25b77bd9482b9e1606cfa88ef90dbba179d4a86f7_sk
    │   │   └── tlsca.org1.jicki.cn-cert.pem
    │   └── users
    │       ├── Admin@org1.jicki.cn
    │       │   ├── msp
    │       │   │   ├── admincerts
    │       │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │       │   │   ├── cacerts
    │       │   │   │   └── ca.org1.jicki.cn-cert.pem
    │       │   │   ├── keystore
    │       │   │   │   └── 5420790271603cc3da1ec2a4e0c45e30fb6ebb00a001021b9e0a4d29ad4d19cc_sk
    │       │   │   ├── signcerts
    │       │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │       │   │   └── tlscacerts
    │       │   │       └── tlsca.org1.jicki.cn-cert.pem
    │       │   └── tls
    │       │       ├── ca.crt
    │       │       ├── client.crt
    │       │       └── client.key
    │       └── User1@org1.jicki.cn
    │           ├── msp
    │           │   ├── admincerts
    │           │   │   └── User1@org1.jicki.cn-cert.pem
    │           │   ├── cacerts
    │           │   │   └── ca.org1.jicki.cn-cert.pem
    │           │   ├── keystore
    │           │   │   └── 43e76b981378f4820bdc3cf7a690e42c018e0cb69cf097dba4bbe0d8f8188cfa_sk
    │           │   ├── signcerts
    │           │   │   └── User1@org1.jicki.cn-cert.pem
    │           │   └── tlscacerts
    │           │       └── tlsca.org1.jicki.cn-cert.pem
    │           └── tls
    │               ├── ca.crt
    │               ├── client.crt
    │               └── client.key
    └── org2.jicki.cn
        ├── ca
        │   ├── 7ae61d566e35ebdcffa42a90e385d6698ffa457b69c3a56b94dfd77d8a2cfe96_sk
        │   └── ca.org2.jicki.cn-cert.pem
        ├── msp
        │   ├── admincerts
        │   │   └── Admin@org2.jicki.cn-cert.pem
        │   ├── cacerts
        │   │   └── ca.org2.jicki.cn-cert.pem
        │   ├── config.yaml
        │   └── tlscacerts
        │       └── tlsca.org2.jicki.cn-cert.pem
        ├── peers
        │   ├── peer0.org2.jicki.cn
        │   │   ├── msp
        │   │   │   ├── admincerts
        │   │   │   │   └── Admin@org2.jicki.cn-cert.pem
        │   │   │   ├── cacerts
        │   │   │   │   └── ca.org2.jicki.cn-cert.pem
        │   │   │   ├── config.yaml
        │   │   │   ├── keystore
        │   │   │   │   └── 753d7217f9cbabdffbc69b74a38efedb70ab80b183cf50d78669fe0d19503aed_sk
        │   │   │   ├── signcerts
        │   │   │   │   └── peer0.org2.jicki.cn-cert.pem
        │   │   │   └── tlscacerts
        │   │   │       └── tlsca.org2.jicki.cn-cert.pem
        │   │   └── tls
        │   │       ├── ca.crt
        │   │       ├── server.crt
        │   │       └── server.key
        │   └── peer1.org2.jicki.cn
        │       ├── msp
        │       │   ├── admincerts
        │       │   │   └── Admin@org2.jicki.cn-cert.pem
        │       │   ├── cacerts
        │       │   │   └── ca.org2.jicki.cn-cert.pem
        │       │   ├── config.yaml
        │       │   ├── keystore
        │       │   │   └── 3ab2f929ddbe609cb587a55ac8f5bca4d47734db2c5bfed0221ef90ea73cb994_sk
        │       │   ├── signcerts
        │       │   │   └── peer1.org2.jicki.cn-cert.pem
        │       │   └── tlscacerts
        │       │       └── tlsca.org2.jicki.cn-cert.pem
        │       └── tls
        │           ├── ca.crt
        │           ├── server.crt
        │           └── server.key
        ├── tlsca
        │   ├── 413aa8963bb8d193e91d7b54ccc699f32f293726e342ca102ea648e73d74813e_sk
        │   └── tlsca.org2.jicki.cn-cert.pem
        └── users
            ├── Admin@org2.jicki.cn
            │   ├── msp
            │   │   ├── admincerts
            │   │   │   └── Admin@org2.jicki.cn-cert.pem
            │   │   ├── cacerts
            │   │   │   └── ca.org2.jicki.cn-cert.pem
            │   │   ├── keystore
            │   │   │   └── 08f6865e8d30d2d326e9d04be8eea92e25b39128afba02652fa0f4ee3f0dc35a_sk
            │   │   ├── signcerts
            │   │   │   └── Admin@org2.jicki.cn-cert.pem
            │   │   └── tlscacerts
            │   │       └── tlsca.org2.jicki.cn-cert.pem
            │   └── tls
            │       ├── ca.crt
            │       ├── client.crt
            │       └── client.key
            └── User1@org2.jicki.cn
                ├── msp
                │   ├── admincerts
                │   │   └── User1@org2.jicki.cn-cert.pem
                │   ├── cacerts
                │   │   └── ca.org2.jicki.cn-cert.pem
                │   ├── keystore
                │   │   └── d5ac39f341b6e0ea2e50646dcabd132dcb002da31760a02e376d55669a30e885_sk
                │   ├── signcerts
                │   │   └── User1@org2.jicki.cn-cert.pem
                │   └── tlscacerts
                │       └── tlsca.org2.jicki.cn-cert.pem
                └── tls
                    ├── ca.crt
                    ├── client.crt
                    └── client.key

109 directories, 107 files
```



## 生成 Hyperledger Fabric 创世区块



```
# 这里使用 configtxgen 来创建 创世区块


# 首先需要创建一个文件夹
mkdir -p /opt/jicki/channel-artifacts

# 完整 configtx.yaml 如下: 
# configtx.yaml 文件格式 请千万注意 空格 与 tab 键 里的缩进，否则会报错。

Organizations:

    - &OrdererOrg
        Name: OrdererMSP
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
            Capabilities:
                <<: *OrdererCapabilities
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
            Capabilities:
                <<: *ApplicationCapabilities

    SampleDevModeKafka:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: kafka
            Kafka:
                Brokers:
                - kafka0:9092
                - kafka1:9092
                - kafka2:9092

            Organizations:
            - *OrdererOrg
            Capabilities:
                <<: *OrdererCapabilities
        Application:
            <<: *ApplicationDefaults
            Organizations:
            - <<: *OrdererOrg
        Consortiums:
            SampleConsortium:
                Organizations:
                - *Org1
                - *Org2
```






```
# 首先需要创建一个文件夹

mkdir -p /opt/jicki/channel-artifacts


# 创建 创世区块  TwoOrgsOrdererGenesis 名称为 configtx.yaml 中 Profiles 字段下的

[root@localhost jicki]# configtxgen -profile TwoOrgsOrdererGenesis \
 -outputBlock ./channel-artifacts/genesis.block


2019-10-29 12:22:06.715 CST [common.tools.configtxgen] main -> WARN 001 Omitting the channel ID for configtxgen for output operations is deprecated.  Explicitly passing the channel ID will be required in the future, defaulting to 'testchainid'.
2019-10-29 12:22:06.715 CST [common.tools.configtxgen] main -> INFO 002 Loading configuration
2019-10-29 12:22:06.743 CST [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: kafka
2019-10-29 12:22:06.743 CST [common.tools.configtxgen.localconfig] Load -> INFO 004 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:22:06.772 CST [common.tools.configtxgen.localconfig] completeInitialization -> INFO 005 orderer type: kafka
2019-10-29 12:22:06.772 CST [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 006 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:22:06.774 CST [common.tools.configtxgen] doOutputBlock -> INFO 007 Generating genesis block
2019-10-29 12:22:06.774 CST [common.tools.configtxgen] doOutputBlock -> INFO 008 Writing genesis block



# 创世区块 是在 orderer 服务中使用

[root@localhost jicki]# ls -lt channel-artifacts/
total 16
-rw-r--r-- 1 root root 13440 Oct 29 12:22 genesis.block
```


```
# 下面来生成一个 peer 服务 中使用的 tx 文件 TwoOrgsChannel 名称为 configtx.yaml 中 Profiles 字段下的，这里必须指定上面的 channelID


[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel


2019-10-29 12:25:41.961 CST [common.tools.configtxgen] main -> INFO 001 Loading configuration
2019-10-29 12:25:41.988 CST [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:25:42.016 CST [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: kafka
2019-10-29 12:25:42.016 CST [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 004 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:25:42.016 CST [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 005 Generating new channel configtx
2019-10-29 12:25:42.018 CST [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 006 Writing new channel tx


[root@localhost jicki]# ls -lt channel-artifacts/
total 20
-rw-r--r-- 1 root root   346 Oct 29 12:25 channel.tx
-rw-r--r-- 1 root root 13440 Oct 29 12:22 genesis.block

```


```
# 定义组织 生成锚节点更新文件

# Org1MSP

[root@localhost jicki]# configtxgen -profile TwoOrgsChannel \
-outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP


2019-10-29 12:26:33.769 CST [common.tools.configtxgen] main -> INFO 001 Loading configuration
2019-10-29 12:26:33.796 CST [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:26:33.824 CST [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: kafka
2019-10-29 12:26:33.824 CST [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 004 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:26:33.824 CST [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 005 Generating anchor peer update
2019-10-29 12:26:33.825 CST [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 006 Writing anchor peer update



# Org2MSP


[root@localhost jicki]# configtxgen -profile TwoOrgsChannel \
-outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP



2019-10-29 12:26:50.797 CST [common.tools.configtxgen] main -> INFO 001 Loading configuration
2019-10-29 12:26:50.825 CST [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:26:50.853 CST [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 orderer type: kafka
2019-10-29 12:26:50.853 CST [common.tools.configtxgen.localconfig] LoadTopLevel -> INFO 004 Loaded configuration: /opt/jicki/configtx.yaml
2019-10-29 12:26:50.853 CST [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 005 Generating anchor peer update
2019-10-29 12:26:50.853 CST [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 006 Writing anchor peer update


```


## 配置 fabric docker-compose.yaml


```
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
    volumes:
    # 存储数据与日志
    - ./data/zookeeper1/data:/data
    - ./data/zookeeper1/datalog:/datalog
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
    volumes:
    # 存储数据与日志
    - ./data/zookeeper2/data:/data
    - ./data/zookeeper2/datalog:/datalog
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
    volumes:
    # 存储数据与日志
    - ./data/zookeeper3/data:/data
    - ./data/zookeeper3/datalog:/datalog
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
      - GODEBUG=netdns=go
    volumes:
    # 存储数据与日志.
    - ./data/kafka1/data:/data
    - ./data/kafka1/data:/logs
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
      - GODEBUG=netdns=go
    volumes:
    # 存储数据与日志.
    - ./data/kafka1/data:/data
    - ./data/kafka1/data:/logs
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
      - GODEBUG=netdns=go
    volumes:
    # 存储数据与日志.
    - ./data/kafka2/data:/data
    - ./data/kafka2/data:/logs
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - zookeeper1
      - zookeeper2
      - zookeeper3

  orderer0.jicki.cn:
    container_name: orderer0.jicki.cn
    image: hyperledger/fabric-orderer:1.4.0
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
      - GODEBUG=netdns=go
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    # 数据持久化,以及存储
    - ./data/orderer0:/var/hyperledger/production
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

  ca.org1.jicki.cn:
    container_name: ca.org1.jicki.cn
    image: hyperledger/fabric-ca:1.4.0
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca-org1
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/3a233ccbd6706cee4c66a320e43bedad3e72b6f68e831ce121f35760eb1ac275_sk
      - FABRIC_CA_SERVER_TLS_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_TLS_KEYFILE=/etc/hyperledger/fabric-ca-server-config/3a233ccbd6706cee4c66a320e43bedad3e72b6f68e831ce121f35760eb1ac275_sk
      - GODEBUG=netdns=go
    ports:
      - "7054:7054"
    command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/3a233ccbd6706cee4c66a320e43bedad3e72b6f68e831ce121f35760eb1ac275_sk -b admin:adminpw -d'
    volumes:
      - ./crypto-config/peerOrganizations/org1.jicki.cn/ca/:/etc/hyperledger/fabric-ca-server-config
    depends_on:
      - orderer0.jicki.cn

  ca.org2.jicki.cn:
    container_name: ca.org2.jicki.cn
    image: hyperledger/fabric-ca:1.4.0
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca-org2
      - FABRIC_CA_SERVER_TLS_ENABLED=true
      - FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org2.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server-config/7ae61d566e35ebdcffa42a90e385d6698ffa457b69c3a56b94dfd77d8a2cfe96_sk
      - FABRIC_CA_SERVER_TLS_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org2.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_TLS_KEYFILE=/etc/hyperledger/fabric-ca-server-config/7ae61d566e35ebdcffa42a90e385d6698ffa457b69c3a56b94dfd77d8a2cfe96_sk
      - GODEBUG=netdns=go
    ports:
      - "8054:7054"
    command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org2.jicki.cn-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/7ae61d566e35ebdcffa42a90e385d6698ffa457b69c3a56b94dfd77d8a2cfe96_sk -b admin:adminpw -d'
    volumes:
      - ./crypto-config/peerOrganizations/org2.jicki.cn/ca/:/etc/hyperledger/fabric-ca-server-config
    depends_on:
      - orderer0.jicki.cn

  couchdb0:
    container_name: couchdb0
    image: hyperledger/fabric-couchdb:0.4.10
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    #ports:
    #  - "5984:5984"
    volumes:
      # 数据持久化，用于存储链码值
      - ./data/couchdb0/data:/opt/couchdb/data
    networks:
      default:
        aliases:
          - jicki

  couchdb1:
    container_name: couchdb1
    image: hyperledger/fabric-couchdb:0.4.10
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    #ports:
    #  - "6984:5984"
    volumes:
      # 数据持久化，用于存储链码值
      - ./data/couchdb1/data:/opt/couchdb/data
    networks:
      default:
        aliases:
          - jicki

  peer0.org1.jicki.cn:
    container_name: peer0.org1.jicki.cn
    image: hyperledger/fabric-peer:1.4.0
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
      - GODEBUG=netdns=go
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls:/etc/hyperledger/fabric/tls
        # 数据持久化, 存储安装，以及实例化智能合约的数据
        - ./data/peer0org1:/var/hyperledger/production
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
      - orderer0.jicki.cn

  peer0.org2.jicki.cn:
    container_name: peer0.org2.jicki.cn
    image: hyperledger/fabric-peer:1.4.0
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
      - GODEBUG=netdns=go
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls:/etc/hyperledger/fabric/tls
        # 数据持久化, 存储安装，以及实例化智能合约的数据
        - ./data/peer0org2:/var/hyperledger/production
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
      - ca.org2.jicki.cn
      - orderer0.jicki.cn
  cli:
    container_name: cli
    image: hyperledger/fabric-tools:1.4.0
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - GODEBUG=netdns=go
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
        - ./data/cli/peer:/opt/gopath/src/github.com/hyperledger/fabric/peer
        - ./data/cli/chaincode/go/:/opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - ca.org1.jicki.cn
      - ca.org2.jicki.cn
      - orderer0.jicki.cn
      - peer0.org1.jicki.cn
      - peer0.org2.jicki.cn
```


### 启动服务

```
docker-compose up -d

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

2019-10-29 05:47:10.407 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 05:47:10.423 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 05:47:10.426 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-10-29 05:47:10.479 UTC [cli.common] readBlock -> INFO 004 Got status: &{SERVICE_UNAVAILABLE}
2019-10-29 05:47:10.483 UTC [channelCmd] InitCmdFactory -> INFO 005 Endorser and orderer connections initialized
2019-10-29 05:47:10.685 UTC [cli.common] readBlock -> INFO 006 Got status: &{SERVICE_UNAVAILABLE}
2019-10-29 05:47:10.687 UTC [channelCmd] InitCmdFactory -> INFO 007 Endorser and orderer connections initialized
2019-10-29 05:47:10.890 UTC [cli.common] readBlock -> INFO 008 Received block: 0




# 创建以后生成文件 mychannel.block

total 24
-rw-r--r-- 1 root root 15632 Oct 29 05:47 mychannel.block
drwxr-xr-x 2 root root  4096 Oct 29 05:42 channel-artifacts
drwxr-xr-x 4 root root  4096 Oct 29 05:25 crypto

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

2019-10-29 06:35:49.083 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:35:49.097 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:35:49.101 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-10-29 06:35:49.293 UTC [channelCmd] executeJoin -> INFO 004 Successfully submitted proposal to join channel

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

2019-10-29 06:36:34.577 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:36:34.595 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:36:34.599 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-10-29 06:36:34.717 UTC [channelCmd] executeJoin -> INFO 004 Successfully submitted proposal to join channel

```



## Hyperledger Fabric 实例化测试

>  在上面我们已经拷贝了官方的例子，在 chaincode 下, 下面我们来测试一下


### 安装智能合约


```
# cli 部分  ./chaincode/go/:/opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go
# 为 智能合约的目录 我们约定为这个目录 需要预先创建 


mkdir -p /opt/jicki/data/cli/chaincode/go/

cd /opt/jicki/data/cli/chaincode/go/

# 创建以后~我们拷贝官方的 例子进来，方便后面进行合约测试

cp -r /opt/jicki/fabric/examples/chaincode/go/example0* /opt/jicki/data/cli/chaincode/go/


# 官方这里有5个例子

[root@localhost jicki]# ls -lt chaincode/go/
total 20
drwxr-xr-x 3 root root 4096 Oct 29 14:37 example01
drwxr-xr-x 3 root root 4096 Oct 29 14:37 example02
drwxr-xr-x 3 root root 4096 Oct 29 14:37 example03
drwxr-xr-x 3 root root 4096 Oct 29 14:37 example04
drwxr-xr-x 3 root root 4096 Oct 29 14:37 example05



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

2019-10-29 06:43:32.933 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:43:32.947 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:43:32.955 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2019-10-29 06:43:32.955 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2019-10-29 06:43:33.665 UTC [chaincodeCmd] install -> INFO 005 Installed remotely response:<status:200 payload:"OK" > 

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

2019-10-29 06:44:06.573 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:44:06.588 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:44:06.596 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2019-10-29 06:44:06.596 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2019-10-29 06:44:06.826 UTC [chaincodeCmd] install -> INFO 005 Installed remotely response:<status:200 payload:"OK" > 

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

2019-10-29 06:48:30.883 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:48:30.897 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:48:30.908 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2019-10-29 06:48:30.908 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc

```


### 操作智能合约


```
# query 查询方法


# 查询 A 账户里的余额

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'


# 输出如下:

2019-10-29 06:49:30.885 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:49:30.901 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
200


# 可以看到 返回 200
```


```
# 查询 B 账户里的余额

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'


# 输出如下:

2019-10-29 06:50:27.353 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:50:27.368 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
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

2019-10-29 06:51:12.681 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:51:12.695 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:51:12.721 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200 


# 可以看到返回 invoke successful. result: status:200 成功
```


```
# 这里再查询 A 与 B 的账户

# A 账户余额 

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'  


# 输出如下:

2019-10-29 06:51:43.325 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:51:43.340 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
100

```


```
# B 账户余额 

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'   



# 输出如下:

2019-10-29 06:51:55.761 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-10-29 06:51:55.775 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
600

```

```
# 查看 peer0.org1.jicki.cn 节点里 生成的容器

[root@localhost jicki]# docker ps -a
CONTAINER ID        IMAGE                                                                                                     COMMAND                  CREATED             STATUS              PORTS                                                                                         NAMES
1e3b763e3034        jicki-peer0.org2.jicki.cn-example2-1.0-3b619b4d039726a6e131bc22c779eab46f4f1325e0299f289761f6372e0fc252   "chaincode -peer.add…"   3 minutes ago       Up 3 minutes                                                                                 jicki-peer0.org2.jicki.cn-example2-1.0

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
      - orderer0.jicki.cn    peers:
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




