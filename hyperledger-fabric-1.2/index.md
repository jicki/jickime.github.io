# hyperledger-fabric v 1.2 多机 多节点



> fabric v1.2 , 多机，多节点，多 Orderer ，生产环境配置。

# 部署 hyperledger-fabric v1.2



## 环境规划

> 相关hostname 必须配置 dns 
>
> 关于 orderer 集群
>
> 当orderer 向peer节点提交Transaction的时候，peer节点会得到或返回一个读写集结果，该结果会发送给orderer节点进行共识和排序，此时如果orderer节点突然down掉，就会使请求服务失效而引发的数据丢失等问题，且目前的sdk对orderer发送的Transaction的回调会占用极长的时间，当大批量数据导入的时候该回调可认为不可用。

|节点标识|hostname|IP|开放端口|系统|
|--------|--------|--|--------|----------|---|--|
|orderer0节点|orderer0.jicki.cn|192.168.100.67|7050|CentOS 7 x64|
|orderer1节点|orderer1.jicki.cn|192.168.100.15|7050|CentOS 7 x64|
|orderer2节点|orderer2.jicki.cn|192.168.100.91|7050|CentOS 7 x64|
|peer0节点|peer0.org1.jicki.cn|192.168.100.15|7051, 7052, 7053|CentOS 7 x64|
|peer0节点|peer0.org2.jicki.cn|192.168.100.91|7051, 7052, 7053|CentOS 7 x64|
|zk0节点|zookeeper0|192.168.100.67|2181|CentOS 7 x64|
|zk1节点|zookeeper1|192.168.100.15|2181|CentOS 7 x64|
|zk2节点|zookeeper2|192.168.100.91|2181|CentOS 7 x64|
|kafka0节点|kafka0|192.168.100.67|9092|CentOS 7 x64|
|kafka1节点|kafka1|192.168.100.15|9092|CentOS 7 x64|
|kafka2节点|kafka2|192.168.100.91|9092|CentOS 7 x64|



> 所有节点 docker 互为单机，不做网络集群，连接使用 hostname  or  IP 
>
> 所有需要配置 hosts  域名
>

```
# 所有节点均需要配置

vi /etc/hosts

# fabric hostname

192.168.100.67 orderer0.jicki.cn
192.168.100.15 orderer1.jicki.cn
192.168.100.91 orderer2.jicki.cn
192.168.100.15 peer0.org1.jicki.cn
192.168.100.91 peer0.org2.jicki.cn
192.168.100.67 zookeeper0
192.168.100.15 zookeeper1
192.168.100.91 zookeeper2
192.168.100.67 kafka0
192.168.100.15 kafka1
192.168.100.91 kafka2

# fabric hostname


```









## 官方地址

> 文档以官方文档为主 http://hyperledger-fabric.readthedocs.io/en/release-1.2/prereqs.html

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
 Version:           18.06.0-ce
 API version:       1.38
 Go version:        go1.10.3
 Git commit:        0ffa825
 Built:             Wed Jul 18 19:08:18 2018
 OS/Arch:           linux/amd64
 Experimental:      false

Server:
 Engine:
  Version:          18.06.0-ce
  API version:      1.38 (minimum version 1.12)
  Go version:       go1.10.3
  Git commit:       0ffa825
  Built:            Wed Jul 18 19:10:42 2018
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
docker-compose version 1.22.0, build f46880f
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

mkdir -p /opt/gopath/src

cp -r github.com /opt/gopath/src


# 文件如下:

[root@localhost fabric]# ls -lt
总用量 580
drwxr-xr-x  7 root root    138 6月   5 09:29 vendor
-rw-r--r--  1 root root    301 6月   5 09:29 tox.ini
drwxr-xr-x  2 root root     56 6月   5 09:29 unit-test
drwxr-xr-x  7 root root    134 6月   5 09:29 test
drwxr-xr-x  2 root root   4096 6月   5 09:29 scripts
-rw-r--r--  1 root root    316 6月   5 09:29 settings.gradle
drwxr-xr-x  3 root root     91 6月   5 09:29 sampleconfig
drwxr-xr-x  2 root root   4096 6月   5 09:29 release_notes
drwxr-xr-x 11 root root   4096 6月   5 09:29 protos
drwxr-xr-x  3 root root     30 6月   5 09:29 release
drwxr-xr-x  9 root root    154 6月   5 09:29 peer
drwxr-xr-x  3 root root     23 6月   5 09:29 proposals
drwxr-xr-x  6 root root    126 6月   5 09:29 orderer
drwxr-xr-x  6 root root   4096 6月   5 09:29 msp
drwxr-xr-x  9 root root    130 6月   5 09:29 images
drwxr-xr-x  2 root root     29 6月   5 09:29 gotools
drwxr-xr-x  2 root root   4096 6月   5 09:29 idemix
drwxr-xr-x 15 root root   4096 6月   5 09:29 gossip
drwxr-xr-x 10 root root   4096 6月   5 09:29 examples
drwxr-xr-x  4 root root     48 6月   5 09:29 events
drwxr-xr-x  4 root root    137 6月   5 09:29 docs
drwxr-xr-x  4 root root   4096 6月   5 09:29 devenv
-rw-r--r--  1 root root   3356 6月   5 09:29 docker-env.mk
drwxr-xr-x 20 root root   4096 6月   5 09:29 core
drwxr-xr-x 23 root root   4096 6月   5 09:29 common
drwxr-xr-x 10 root root   4096 6月   5 09:29 bddtests
-rw-r--r--  1 root root     13 6月   5 09:29 ci.properties
drwxr-xr-x  8 root root   4096 6月   5 09:29 bccsp
-rw-r--r--  1 root root  11358 6月   5 09:29 LICENSE
-rwxr-xr-x  1 root root  17068 6月   5 09:29 Makefile
-rw-r--r--  1 root root   5678 6月   5 09:29 README.md
-rw-r--r--  1 root root 475471 6月   5 09:29 CHANGELOG.md
-rw-r--r--  1 root root    597 6月   5 09:29 CODE_OF_CONDUCT.md
-rw-r--r--  1 root root    664 6月   5 09:29 CONTRIBUTING.md


```


## 生成 Hyperledger Fabric 证书


> 证书生成只需要生成一次，这里只在第一个节点配置


```
# 下载官方证书生成软件(均为二进制文件)
# 官方离线下载地址为 https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/


# 选择相应版本 CentOS 选择 linux-amd64-1.2.0  Mac 选择 darwin-amd64-1.2.0


# 下载地址为: https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.2.0/hyperledger-fabric-linux-amd64-1.2.0.tar.gz



mkdir /opt/jicki/

cd /opt/jicki

wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.2.0/hyperledger-fabric-linux-amd64-1.2.0.tar.gz

tar zxvf hyperledger-fabric-linux-amd64-1.2.0.tar.gz

# 解压后是 一个 bin 与 一个 config 目录

[root@localhost jicki]# tree bin/
bin/
├── configtxgen
├── configtxlator
├── cryptogen
├── get-docker-images.sh
├── orderer
└── peer

0 directories, 6 files
 
 
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
│   └── jicki.cn
│       ├── ca
│       │   ├── ca.jicki.cn-cert.pem
│       │   └── ea7afe99d56e6c2e839113909651f8b3531a6447e4615eed208c7aa86b211ad0_sk
│       ├── msp
│       │   ├── admincerts
│       │   │   └── Admin@jicki.cn-cert.pem
│       │   ├── cacerts
│       │   │   └── ca.jicki.cn-cert.pem
│       │   └── tlscacerts
│       │       └── tlsca.jicki.cn-cert.pem
│       ├── orderers
│       │   ├── orderer0.jicki.cn
│       │   │   ├── msp
│       │   │   │   ├── admincerts
│       │   │   │   │   └── Admin@jicki.cn-cert.pem
│       │   │   │   ├── cacerts
│       │   │   │   │   └── ca.jicki.cn-cert.pem
│       │   │   │   ├── keystore
│       │   │   │   │   └── c4ff1a8f55af9e4108596a26471a2a748eeb370b00300abddac692d5c5f72f57_sk
│       │   │   │   ├── signcerts
│       │   │   │   │   └── orderer0.jicki.cn-cert.pem
│       │   │   │   └── tlscacerts
│       │   │   │       └── tlsca.jicki.cn-cert.pem
│       │   │   └── tls
│       │   │       ├── ca.crt
│       │   │       ├── server.crt
│       │   │       └── server.key
│       │   ├── orderer1.jicki.cn
│       │   │   ├── msp
│       │   │   │   ├── admincerts
│       │   │   │   │   └── Admin@jicki.cn-cert.pem
│       │   │   │   ├── cacerts
│       │   │   │   │   └── ca.jicki.cn-cert.pem
│       │   │   │   ├── keystore
│       │   │   │   │   └── f057fd116621213b9d06cf05d1b04b3b53057b4df4338ca5bb3b59e36acf074a_sk
│       │   │   │   ├── signcerts
│       │   │   │   │   └── orderer1.jicki.cn-cert.pem
│       │   │   │   └── tlscacerts
│       │   │   │       └── tlsca.jicki.cn-cert.pem
│       │   │   └── tls
│       │   │       ├── ca.crt
│       │   │       ├── server.crt
│       │   │       └── server.key
│       │   └── orderer2.jicki.cn
│       │       ├── msp
│       │       │   ├── admincerts
│       │       │   │   └── Admin@jicki.cn-cert.pem
│       │       │   ├── cacerts
│       │       │   │   └── ca.jicki.cn-cert.pem
│       │       │   ├── keystore
│       │       │   │   └── a0395ee7823af3ab3ae56fa4569051a5525324f506cd38b0d0dcaa3e484ae084_sk
│       │       │   ├── signcerts
│       │       │   │   └── orderer2.jicki.cn-cert.pem
│       │       │   └── tlscacerts
│       │       │       └── tlsca.jicki.cn-cert.pem
│       │       └── tls
│       │           ├── ca.crt
│       │           ├── server.crt
│       │           └── server.key
│       ├── tlsca
│       │   ├── 4164a39aaee758c85e9d4458bae4d51f0d01e58116003cd5782dbbaff68b1545_sk
│       │   └── tlsca.jicki.cn-cert.pem
│       └── users
│           └── Admin@jicki.cn
│               ├── msp
│               │   ├── admincerts
│               │   │   └── Admin@jicki.cn-cert.pem
│               │   ├── cacerts
│               │   │   └── ca.jicki.cn-cert.pem
│               │   ├── keystore
│               │   │   └── 30d0b23ab85ede3d4424a336d839067bf7cad70f538fcf87d2a9c5083b5482f4_sk
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
    │   │   ├── 98671ea7c42ccdeb73327d7c67f759e487497f13349b97a135392f660cf6747f_sk
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
    │   │   │   │   │   └── c99b65a414bc877998bb52c7ba80faf7e01d6512e8e50253daccd93e273bbb24_sk
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
    │   │       │   │   └── f1a8262cd8c78642d0aa50eb9ac2bae98949ce937063fb8e0d0c03fb7b16e4ce_sk
    │   │       │   ├── signcerts
    │   │       │   │   └── peer1.org1.jicki.cn-cert.pem
    │   │       │   └── tlscacerts
    │   │       │       └── tlsca.org1.jicki.cn-cert.pem
    │   │       └── tls
    │   │           ├── ca.crt
    │   │           ├── server.crt
    │   │           └── server.key
    │   ├── tlsca
    │   │   ├── c4e8130e739a69e6f9628e1e98fa54326ddfdcc93ef8568ffb72ba521069f3d1_sk
    │   │   └── tlsca.org1.jicki.cn-cert.pem
    │   └── users
    │       ├── Admin@org1.jicki.cn
    │       │   ├── msp
    │       │   │   ├── admincerts
    │       │   │   │   └── Admin@org1.jicki.cn-cert.pem
    │       │   │   ├── cacerts
    │       │   │   │   └── ca.org1.jicki.cn-cert.pem
    │       │   │   ├── keystore
    │       │   │   │   └── c8edb858abc7a4589458419d9ca74125830115e1e3544cc85c7a4a91f8b84b52_sk
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
    │           │   │   └── c720e53e25331b60bd8b33f2e19642e1ebef092afcc5f4c07f6efab91eceb298_sk
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
        │   ├── 5e398a55ab9e021cc89f60dfc16ae21def5b35495f0621d8aa55a2463ba051f6_sk
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
        │   │   │   │   └── a4eb0cf3769173e73cfcaa74ec938ef2def877b0ae0efe4e95be4a7588621da0_sk
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
        │       │   │   └── 26e3d15b68a9d2b022e6560607b3556e0150df3526826ff4a81c585714d5d41c_sk
        │       │   ├── signcerts
        │       │   │   └── peer1.org2.jicki.cn-cert.pem
        │       │   └── tlscacerts
        │       │       └── tlsca.org2.jicki.cn-cert.pem
        │       └── tls
        │           ├── ca.crt
        │           ├── server.crt
        │           └── server.key
        ├── tlsca
        │   ├── 0979917773701392b0e7f1924ae711cedfae0dfbb4d82bd5a90c18207855fe8e_sk
        │   └── tlsca.org2.jicki.cn-cert.pem
        └── users
            ├── Admin@org2.jicki.cn
            │   ├── msp
            │   │   ├── admincerts
            │   │   │   └── Admin@org2.jicki.cn-cert.pem
            │   │   ├── cacerts
            │   │   │   └── ca.org2.jicki.cn-cert.pem
            │   │   ├── keystore
            │   │   │   └── ba8e549c298b47d1889d51d75e3a01724b8d2b068da369c7319f2cd586c49e70_sk
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
                │   │   └── e183353072b166455f40b2b65ec168f52518101ee7097345cd8ff8864a71698e_sk
                │   ├── signcerts
                │   │   └── User1@org2.jicki.cn-cert.pem
                │   └── tlscacerts
                │       └── tlsca.org2.jicki.cn-cert.pem
                └── tls
                    ├── ca.crt
                    ├── client.crt
                    └── client.key

125 directories, 123 files
```

```
# 这里为了方便， 将整个证书目录拷贝到其他节点里去 
# 一般来说只需要拷贝对应的

scp -r crypto-config 192.168.100.15:/opt/jicki

scp -r crypto-config 192.168.100.91:/opt/jicki



```





## 生成 Hyperledger Fabric 创世区块


> 只在第一个节点配置


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
        AnchorPeers:
            - Host: peer0.org1.jicki.cn
              Port: 7051
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

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2.jicki.cn/msp
        AnchorPeers:
            - Host: peer0.org2.jicki.cn
              Port: 7051
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

Capabilities:
    Global: &ChannelCapabilities
        V1_1: true

    Orderer: &OrdererCapabilities
        V1_1: true

    Application: &ApplicationCapabilities
        V1_2: true

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
        # 设置最大的区块大小。每个区块最大有Orderer.AbsoluteMaxBytes个字节
        # 假定这里设置的值为 99MB，记住这个值，这会影响怎样配置Kafka代理。
        AbsoluteMaxBytes: 99 MB
        # 设置每个区块建议的大小。Kafka对于相对小的消息提供更高的吞吐量。
        # 区块大小最好不要超过1MB。
        PreferredMaxBytes: 512 KB
    Kafka:
        Brokers:
        # 这里如果非容器内建议使用 [ip:port]配置kafka 集群。
        # 如下为 单机版本集群内的配置。
        # kafka 集群最少为3节点, 建议配置4个以上。
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


2018-07-24 14:17:28.681 CST [common/tools/configtxgen] main -> WARN 001 Omitting the channel ID for configtxgen is deprecated.  Explicitly passing the channel ID will be required in the future, defaulting to 'testchainid'.
2018-07-24 14:17:28.681 CST [common/tools/configtxgen] main -> INFO 002 Loading configuration
2018-07-24 14:17:28.726 CST [msp] getMspConfig -> INFO 003 Loading NodeOUs
2018-07-24 14:17:28.727 CST [msp] getMspConfig -> INFO 004 Loading NodeOUs
2018-07-24 14:17:28.727 CST [common/tools/configtxgen] doOutputBlock -> INFO 005 Generating genesis block
2018-07-24 14:17:28.728 CST [common/tools/configtxgen] doOutputBlock -> INFO 006 Writing genesis block



# 创世区块 是在 orderer 服务中使用

[root@localhost jicki]# ls -lt channel-artifacts/
总用量 16
-rw-r--r-- 1 root root 12484 7月  24 14:17 genesis.block
```


```
# 下面来生成一个 peer 服务 中使用的 tx 文件 TwoOrgsChannel 名称为 configtx.yaml 中 Profiles 字段下的，这里必须指定上面的 channelID


[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel


2018-07-24 14:18:02.261 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-07-24 14:18:02.306 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-07-24 14:18:02.307 CST [msp] getMspConfig -> INFO 003 Loading NodeOUs
2018-07-24 14:18:02.308 CST [msp] getMspConfig -> INFO 004 Loading NodeOUs
2018-07-24 14:18:02.311 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 005 Writing new channel tx



[root@localhost jicki]# ls -lt channel-artifacts/
总用量 20
-rw-r--r-- 1 root root   346 7月  24 14:18 channel.tx
-rw-r--r-- 1 root root 12484 7月  24 14:17 genesis.block

```


```
# 定义组织 生成锚节点更新文件


# Org1MSP

[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP


2018-07-24 14:18:02.261 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-07-24 14:18:02.306 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-07-24 14:18:02.307 CST [msp] getMspConfig -> INFO 003 Loading NodeOUs
2018-07-24 14:18:02.308 CST [msp] getMspConfig -> INFO 004 Loading NodeOUs
2018-07-24 14:18:02.311 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 005 Writing new channel tx



# Org2MSP


[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP

2018-07-24 14:19:21.947 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-07-24 14:19:21.993 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 002 Generating anchor peer update
2018-07-24 14:19:21.993 CST [common/tools/configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Writing anchor peer update


```


```
# 拷贝 channel-artifacts 目录到其他节点

scp -r channel-artifacts 192.168.100.15:/opt/jicki

scp -r channel-artifacts 192.168.100.91:/opt/jicki


```



## 配置 Zookeeper Kafka 集群

> zookeeper 与 kafka 集群 必须在 所有服务器之间启动。


### 第一个节点 (192.168.100.67)

> ZOO_SERVERS 本机因为在 docker 里，需要配置为 0.0.0.0:2888:3888


```
vi docker-compose-zk-kafka.yaml


version: '2'
services:
  zookeeper0:
    container_name: zookeeper0
    hostname: zookeeper0
    image: zookeeper
    restart: always
    environment:
      - ZOO_MY_ID=1
      - ZOO_SERVERS=server.1=0.0.0.0:2888:3888 server.2=zookeeper1:2888:3888 server.3=zookeeper2:2888:3888
    #volumes:
    # 存储数据与日志
    #- ./data/zookeeper0/data:/data
    #- ./data/zookeeper0/datalog:/datalog
    ports:
      - 192.168.100.67:2181:2181 
      - 192.168.100.67:2888:2888
      - 192.168.100.67:3888:3888
      extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      
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
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper0:2181,zookeeper1:2181,zookeeper2:2181
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
    #- ./data/kafka0/data:/data
    #- ./data/kafka0/data:/logs
    ports:
      - 192.168.100.67:9092:9092
    depends_on:
      - zookeeper0
    extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      - "kafka0:192.168.100.67"
      - "kafka1:192.168.100.15"
      - "kafka2:192.168.100.91"
      
```



### 第二个节点 (192.168.100.15)

```
vi docker-compose-zk-kafka.yaml


version: '2'
services:
  zookeeper1:
    container_name: zookeeper1
    hostname: zookeeper1
    image: zookeeper
    restart: always
    environment:
      - ZOO_MY_ID=2
      - ZOO_SERVERS=server.1=zookeeper0:2888:3888 server.2=0.0.0.0:2888:3888 server.3=zookeeper2:2888:3888
    #volumes:
    # 存储数据与日志
    #- ./data/zookeeper1/data:/data
    #- ./data/zookeeper1/datalog:/datalog
    ports:
      - 192.168.100.15:2181:2181
      - 192.168.100.15:2888:2888
      - 192.168.100.15:3888:3888
     extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      
  kafka1:
    container_name: kafka1
    hostname: kafka1
    image: hyperledger/fabric-kafka
    restart: always
    environment:
      - KAFKA_BROKER_ID=2
      - KAFKA_MIN_INSYNC_REPLICAS=2
      - KAFKA_DEFAULT_REPLICATION_FACTOR=3
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper0:2181,zookeeper1:2181,zookeeper2:2181
      - KAFKA_MESSAGE_MAX_BYTES=103809024
      - KAFKA_REPLICA_FETCH_MAX_BYTES=103809024
      - KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE=false
      - KAFKA_LOG_RETENTION_MS=-1
    #volumes:
    # 存储数据与日志.
    #- ./data/kafka1/data:/data
    #- ./data/kafka1/data:/logs
    ports:
      - 192.168.100.15:9092:9092
    depends_on:
      - zookeeper1
    extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      - "kafka0:192.168.100.67"
      - "kafka1:192.168.100.15"
      - "kafka2:192.168.100.91"
      
```


### 第三个节点 (192.168.100.91)

```      
vi docker-compose-zk-kafka.yaml


version: '2'
services:
  zookeeper2:
    container_name: zookeeper2
    hostname: zookeeper2
    image: zookeeper
    restart: always
    environment:
      - ZOO_MY_ID=3
      - ZOO_SERVERS=server.1=zookeeper0:2888:3888 server.2=zookeeper1:2888:3888 server.3=0.0.0.0:2888:3888
    #volumes:
    # 存储数据与日志
    #- ./data/zookeeper2/data:/data
    #- ./data/zookeeper2/datalog:/datalog
    ports:
      - 192.168.100.91:2181:2181
      - 192.168.100.91:2888:2888
      - 192.168.100.91:3888:3888
    extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      
  kafka2:
    container_name: kafka2
    hostname: kafka2
    image: hyperledger/fabric-kafka
    restart: always
    environment:
      - KAFKA_BROKER_ID=3
      - KAFKA_MIN_INSYNC_REPLICAS=2
      - KAFKA_DEFAULT_REPLICATION_FACTOR=3
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper0:2181,zookeeper1:2181,zookeeper2:2181
      - KAFKA_MESSAGE_MAX_BYTES=103809024
      - KAFKA_REPLICA_FETCH_MAX_BYTES=103809024
      - KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE=false
      - KAFKA_LOG_RETENTION_MS=-1
    #volumes:
    # 存储数据与日志.
    #- ./data/kafka2/data:/data
    #- ./data/kafka2/data:/logs
    ports:
      - 192.168.100.91:9092:9092
    depends_on:
      - zookeeper2
    extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      - "kafka0:192.168.100.67"
      - "kafka1:192.168.100.15"
      - "kafka2:192.168.100.91"
      
```


```
# 分别启动服务

docker-compose -f docker-compose-zk-kafka.yaml up -d




```




## 配置 Hyperledger Fabric Orderer


### 第一个节点 (192.168.100.67)

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
      - ORDERER_GENERAL_TLS_ENABLED=false
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]

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
    ports:
      - 7050:7050
    extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      - "kafka0:192.168.100.67"
      - "kafka1:192.168.100.15"
      - "kafka2:192.168.100.91"
      
```      


### 第二个节点 (192.168.100.15)



```
# 创建文件 docker-compose-orderer.yaml

# 创建于 /opt/jicki 目录下

vi  docker-compose-orderer.yaml


version: '2'
services:
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
      - ORDERER_GENERAL_TLS_ENABLED=false
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
      
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
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer1.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer1.jicki.cn/tls/:/var/hyperledger/orderer/tls
    ports:
      - 7050:7050
    extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      - "kafka0:192.168.100.67"
      - "kafka1:192.168.100.15"
      - "kafka2:192.168.100.91"
      
```


### 第三个节点 (192.168.100.91)

```   
# 创建文件 docker-compose-orderer.yaml

# 创建于 /opt/jicki 目录下

vi  docker-compose-orderer.yaml


version: '2'
services:
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
      - ORDERER_GENERAL_TLS_ENABLED=false
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
      
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
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer2.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer2.jicki.cn/tls/:/var/hyperledger/orderer/tls
    ports:
      - 7050:7050
    extra_hosts:
      - "zookeeper0:192.168.100.67"
      - "zookeeper1:192.168.100.15"
      - "zookeeper2:192.168.100.91"
      - "kafka0:192.168.100.67"
      - "kafka1:192.168.100.15"
      - "kafka2:192.168.100.91"
      
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
 
 
 
## 配置 Hyperledger Fabric peer
 
> 节点下都必须启动一个数据存储，如 file 或者 couchdb 等


### 第二个节点 (192.168.100.15) 


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
      - CORE_PEER_TLS_ENABLED=false
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
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
    depends_on:
      - couchdb0
    extra_hosts:
      - "couchdb0:192.168.100.15"
      - "orderer0.jicki.cn:192.168.100.67"
      - "orderer1.jicki.cn:192.168.100.15"
      - "orderer2.jicki.cn:192.168.100.91"

```



### 第三个节点 (192.168.100.91) 


```
vi  docker-compose-peer.yaml


version: '2'
services:
  couchdb1:
    container_name: couchdb1
    image: hyperledger/fabric-couchdb
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    ports:
      - "5984:5984"
    #volumes:
      # 数据持久化，用于存储链码值
      #- ./data/couchdb1/data:/opt/couchdb/data
      
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
      - CORE_PEER_TLS_ENABLED=false
      - CORE_PEER_TLS_CERT_FILE=/etc/hyperledger/fabric/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/etc/hyperledger/fabric/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/etc/hyperledger/fabric/tls/ca.crt
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls:/etc/hyperledger/fabric/tls
        # 数据持久化, 存储安装，以及实例化智能合约的数据
        - ./data/peer0org2:/var/hyperledger/production
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    ports:
      - 7051:7051
      - 7052:7052
      - 7053:7053
    depends_on:
      - couchdb1
    extra_hosts:
      - "couchdb1:192.168.100.91"
      - "orderer0.jicki.cn:192.168.100.67"
      - "orderer1.jicki.cn:192.168.100.15"
      - "orderer2.jicki.cn:192.168.100.91"

```
      
### 启动服务

```
docker-compose -f docker-compose-peer.yaml up -d


# 查看 docker logs 输出如下, 既为成功


# 第二个节点

2018-08-07 02:05:20.638 UTC [discovery] NewService -> INFO 122 Created with config TLS: false, authCacheMaxSize: 1000, authCachePurgeRatio: 0.750000
2018-08-07 02:05:20.638 UTC [nodeCmd] registerDiscoveryService -> INFO 123 Discovery service activated
2018-08-07 02:05:20.638 UTC [nodeCmd] serve -> INFO 124 Starting peer with ID=[name:"peer0.org1.jicki.cn" ], network ID=[jicki], address=[peer0.org1.jicki.cn:7051]
2018-08-07 02:05:20.638 UTC [nodeCmd] serve -> INFO 125 Started peer with ID=[name:"peer0.org1.jicki.cn" ], network ID=[jicki], address=[peer0.org1.jicki.cn:7051]



# 第三个节点

2018-08-07 02:07:23.628 UTC [discovery] NewService -> INFO 133 Created with config TLS: false, authCacheMaxSize: 1000, authCachePurgeRatio: 0.750000
2018-08-07 02:07:23.628 UTC [nodeCmd] registerDiscoveryService -> INFO 134 Discovery service activated
2018-08-07 02:07:23.628 UTC [nodeCmd] serve -> INFO 135 Starting peer with ID=[name:"peer0.org2.jicki.cn" ], network ID=[jicki], address=[peer0.org2.jicki.cn:7051]
2018-08-07 02:07:23.628 UTC [nodeCmd] serve -> INFO 136 Started peer with ID=[name:"peer0.org2.jicki.cn" ], network ID=[jicki], address=[peer0.org2.jicki.cn:7051]


```
      


## 配置 Hyperledger Fabric cli 客户端
   
 
> 客户端服务，只需要配置一个既可，这里配置再 第一个节点中，用于调用创建 channel 与 智能合约



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
      - CORE_PEER_TLS_ENABLED=false
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
    extra_hosts:
      - "orderer0.jicki.cn:192.168.100.67"
      - "orderer1.jicki.cn:192.168.100.15"
      - "orderer2.jicki.cn:192.168.100.91"
      - "peer0.org1.jicki.cn:192.168.100.15"
      - "peer0.org2.jicki.cn:192.168.100.91"
```


### 启动服务

```
docker-compose -f docker-compose-cli.yaml up -d

```






## Hyperledger Fabric 创建 Channel



```
# 上面我们创建了 cli 容器，我们可以直接进入 容器里操作


[root@localhost jicki]# docker exec -it cli bash
root@77962643125a:/opt/gopath/src/github.com/hyperledger/fabric/peer# 


# 执行 创建命令 (未启动 认证)

peer channel create -c mychannel -f ./channel-artifacts/channel.tx --orderer orderer0.jicki.cn:7050


# 以下为启用认证

peer channel create -o orderer0.jicki.cn:7050 -c mychannel -f ./channel-artifacts/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem




# 输出如下:

2018-08-07 02:20:49.574 UTC [grpc] Printf -> DEBU 094 parsed scheme: ""
2018-08-07 02:20:49.574 UTC [grpc] Printf -> DEBU 095 scheme "" not registered, fallback to default scheme
2018-08-07 02:20:49.574 UTC [grpc] Printf -> DEBU 096 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-08-07 02:20:49.574 UTC [grpc] Printf -> DEBU 097 ClientConn switching balancer to "pick_first"
2018-08-07 02:20:49.574 UTC [grpc] Printf -> DEBU 098 pickfirstBalancer: HandleSubConnStateChange: 0xc420281c80, CONNECTING
2018-08-07 02:20:49.575 UTC [grpc] Printf -> DEBU 099 pickfirstBalancer: HandleSubConnStateChange: 0xc420281c80, READY
2018-08-07 02:20:49.575 UTC [channelCmd] InitCmdFactory -> INFO 09a Endorser and orderer connections initialized
2018-08-07 02:20:49.775 UTC [msp] GetDefaultSigningIdentity -> DEBU 09b Obtaining default signing identity
2018-08-07 02:20:49.775 UTC [msp] GetDefaultSigningIdentity -> DEBU 09c Obtaining default signing identity
2018-08-07 02:20:49.775 UTC [msp/identity] Sign -> DEBU 09d Sign: plaintext: 0AD1060A1508051A06088184A4DB0522...2FB3601B298B12080A021A0012021A00 
2018-08-07 02:20:49.775 UTC [msp/identity] Sign -> DEBU 09e Sign: digest: DE4E562C31BC2BA09C70978648AAE7EC6F81BC49BF5D6AE8760FD64B84F3541C 
2018-08-07 02:20:49.777 UTC [cli/common] readBlock -> INFO 09f Received block: 0




# 创建以后生成文件 mychannel.block

total 24
-rw-r--r-- 1 root root 15382 Aug  7 02:20 mychannel.block
drwxr-xr-x 2 root root  4096 Aug  6 09:16 channel-artifacts
drwxr-xr-x 4 root root  4096 Aug  6 06:45 crypto

```



## Hyperledger Fabric 加入 Channel


> 我们这边有2个 peer 所以需要分别加入, 后续有多少个 peer 都需要加入到 Channel 中

```
# peer0.org1.jicki.cn 加入 此 channel 中，首先需要查看如下 环境变量


echo $CORE_PEER_LOCALMSPID
echo $CORE_PEER_ADDRESS
echo $CORE_PEER_MSPCONFIGPATH
echo $CORE_PEER_TLS_ROOTCERT_FILE


# 加入 channel

peer channel join -b mychannel.block


# 输出如下: 

2018-08-07 02:31:52.810 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:31:52.811 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:31:52.811 UTC [msp] GetDefaultSigningIdentity -> DEBU 037 Obtaining default signing identity
2018-08-07 02:31:52.811 UTC [grpc] Printf -> DEBU 038 parsed scheme: ""
2018-08-07 02:31:52.811 UTC [grpc] Printf -> DEBU 039 scheme "" not registered, fallback to default scheme
2018-08-07 02:31:52.811 UTC [grpc] Printf -> DEBU 03a ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:31:52.811 UTC [grpc] Printf -> DEBU 03b ClientConn switching balancer to "pick_first"
2018-08-07 02:31:52.811 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4203cb530, CONNECTING
2018-08-07 02:31:52.812 UTC [grpc] Printf -> DEBU 03d pickfirstBalancer: HandleSubConnStateChange: 0xc4203cb530, READY
2018-08-07 02:31:52.812 UTC [channelCmd] InitCmdFactory -> INFO 03e Endorser and orderer connections initialized
2018-08-07 02:31:52.813 UTC [msp/identity] Sign -> DEBU 03f Sign: plaintext: 0A98070A5C08011A0C089889A4DB0510...131231D39FB81A080A000A000A000A00 
2018-08-07 02:31:52.813 UTC [msp/identity] Sign -> DEBU 040 Sign: digest: E9D23EC60B2EDFA54AD140AB09299DCF3FA11A657D85743E58A15BC02FE38743 
2018-08-07 02:31:52.902 UTC [channelCmd] executeJoin -> INFO 041 Successfully submitted proposal to join channel


```


```
# peer1.org2.jicki.cn 加入 此 channel 中，这里配置一下环境变量

export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_ADDRESS=peer0.org2.jicki.cn:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp


# 加入 channel

peer channel join -b mychannel.block



# 输入如下:

2018-08-07 02:32:23.864 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:32:23.864 UTC [msp] Validate -> DEBU 036 MSP Org2MSP validating identity
2018-08-07 02:32:23.865 UTC [msp] GetDefaultSigningIdentity -> DEBU 037 Obtaining default signing identity
2018-08-07 02:32:23.865 UTC [grpc] Printf -> DEBU 038 parsed scheme: ""
2018-08-07 02:32:23.865 UTC [grpc] Printf -> DEBU 039 scheme "" not registered, fallback to default scheme
2018-08-07 02:32:23.865 UTC [grpc] Printf -> DEBU 03a ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-08-07 02:32:23.865 UTC [grpc] Printf -> DEBU 03b ClientConn switching balancer to "pick_first"
2018-08-07 02:32:23.865 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc420ca8bf0, CONNECTING
2018-08-07 02:32:23.866 UTC [grpc] Printf -> DEBU 03d pickfirstBalancer: HandleSubConnStateChange: 0xc420ca8bf0, READY
2018-08-07 02:32:23.866 UTC [channelCmd] InitCmdFactory -> INFO 03e Endorser and orderer connections initialized
2018-08-07 02:32:23.867 UTC [msp/identity] Sign -> DEBU 03f Sign: plaintext: 0A98070A5C08011A0C08B789A4DB0510...131231D39FB81A080A000A000A000A00 
2018-08-07 02:32:23.867 UTC [msp/identity] Sign -> DEBU 040 Sign: digest: 620FDAF5EA0CAC6FB6FEC4B16910082B6E471EA03688AB0095A71DC58F280E0B 
2018-08-07 02:32:23.955 UTC [channelCmd] executeJoin -> INFO 041 Successfully submitted proposal to join channel

```


## Hyperledger Fabric 锚节点

>  锚节点通过广播的方式通知有新节点加入


```
# 使用Org1的管理员身份更新锚节点配置 

# 同样需要先配置变量

export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_ADDRESS=peer0.org1.jicki.cn:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/users/Admin@org1.jicki.cn/msp



# 未开启认证的方式

peer channel update -o orderer0.jicki.cn:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx



# 开启认证的方式

peer channel update -o orderer0.jicki.cn:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem



# 输出如下:

2018-08-07 02:45:16.093 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:45:16.093 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:45:16.093 UTC [msp] GetDefaultSigningIdentity -> DEBU 037 Obtaining default signing identity
2018-08-07 02:45:16.093 UTC [grpc] Printf -> DEBU 038 parsed scheme: ""
2018-08-07 02:45:16.093 UTC [grpc] Printf -> DEBU 039 scheme "" not registered, fallback to default scheme
2018-08-07 02:45:16.093 UTC [grpc] Printf -> DEBU 03a ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-08-07 02:45:16.093 UTC [grpc] Printf -> DEBU 03b ClientConn switching balancer to "pick_first"
2018-08-07 02:45:16.093 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4201fe5e0, CONNECTING
2018-08-07 02:45:16.094 UTC [grpc] Printf -> DEBU 03d pickfirstBalancer: HandleSubConnStateChange: 0xc4201fe5e0, READY
2018-08-07 02:45:16.094 UTC [channelCmd] InitCmdFactory -> INFO 03e Endorser and orderer connections initialized
2018-08-07 02:45:16.094 UTC [msp] GetDefaultSigningIdentity -> DEBU 03f Obtaining default signing identity
2018-08-07 02:45:16.094 UTC [msp] GetDefaultSigningIdentity -> DEBU 040 Obtaining default signing identity
2018-08-07 02:45:16.094 UTC [msp/identity] Sign -> DEBU 041 Sign: plaintext: 0A9A060A074F7267314D5350128E062D...2A0641646D696E732A0641646D696E73 
2018-08-07 02:45:16.094 UTC [msp/identity] Sign -> DEBU 042 Sign: digest: B9EBDD95115ECC8722C7E5D917CE7AB428AF8F7921A55D01D988488B24EBCA64 
2018-08-07 02:45:16.094 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:45:16.094 UTC [msp] GetDefaultSigningIdentity -> DEBU 044 Obtaining default signing identity
2018-08-07 02:45:16.094 UTC [msp/identity] Sign -> DEBU 045 Sign: plaintext: 0AD1060A1508021A0608BC8FA4DB0522...223072D3C517446F35F9431FCE53B9AD 
2018-08-07 02:45:16.094 UTC [msp/identity] Sign -> DEBU 046 Sign: digest: 6B483C98171C37B2FBD318E27AC73DD68277C3719CAA04719225672B028ABABD 
2018-08-07 02:45:16.094 UTC [grpc] Printf -> DEBU 047 parsed scheme: ""
2018-08-07 02:45:16.094 UTC [grpc] Printf -> DEBU 048 scheme "" not registered, fallback to default scheme
2018-08-07 02:45:16.094 UTC [grpc] Printf -> DEBU 049 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-08-07 02:45:16.094 UTC [grpc] Printf -> DEBU 04a ClientConn switching balancer to "pick_first"
2018-08-07 02:45:16.095 UTC [grpc] Printf -> DEBU 04b pickfirstBalancer: HandleSubConnStateChange: 0xc4201fef50, CONNECTING
2018-08-07 02:45:16.095 UTC [grpc] Printf -> DEBU 04c pickfirstBalancer: HandleSubConnStateChange: 0xc4201fef50, READY
2018-08-07 02:45:16.214 UTC [channelCmd] update -> INFO 04d Successfully submitted channel update

```


```
# 使用Org2的管理员身份更新锚节点配置 

# 同样需要先配置变量

export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_ADDRESS=peer0.org2.jicki.cn:7051
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt
export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp


# 未开启认证的方式

peer channel update -o orderer0.jicki.cn:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx



# 开启认证的方式

peer channel update -o orderer0.jicki.cn:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem


# 输出如下:

2018-08-07 02:45:36.285 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:45:36.285 UTC [msp] Validate -> DEBU 036 MSP Org2MSP validating identity
2018-08-07 02:45:36.286 UTC [msp] GetDefaultSigningIdentity -> DEBU 037 Obtaining default signing identity
2018-08-07 02:45:36.286 UTC [grpc] Printf -> DEBU 038 parsed scheme: ""
2018-08-07 02:45:36.286 UTC [grpc] Printf -> DEBU 039 scheme "" not registered, fallback to default scheme
2018-08-07 02:45:36.286 UTC [grpc] Printf -> DEBU 03a ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-08-07 02:45:36.286 UTC [grpc] Printf -> DEBU 03b ClientConn switching balancer to "pick_first"
2018-08-07 02:45:36.286 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc42025a560, CONNECTING
2018-08-07 02:45:36.286 UTC [grpc] Printf -> DEBU 03d pickfirstBalancer: HandleSubConnStateChange: 0xc42025a560, READY
2018-08-07 02:45:36.287 UTC [channelCmd] InitCmdFactory -> INFO 03e Endorser and orderer connections initialized
2018-08-07 02:45:36.287 UTC [msp] GetDefaultSigningIdentity -> DEBU 03f Obtaining default signing identity
2018-08-07 02:45:36.287 UTC [msp] GetDefaultSigningIdentity -> DEBU 040 Obtaining default signing identity
2018-08-07 02:45:36.287 UTC [msp/identity] Sign -> DEBU 041 Sign: plaintext: 0A9A060A074F7267324D5350128E062D...2A0641646D696E732A0641646D696E73 
2018-08-07 02:45:36.287 UTC [msp/identity] Sign -> DEBU 042 Sign: digest: 1A11594F70194B6D24D4D5416310F049D0D292447F3A9624AC3612F36E2CA424 
2018-08-07 02:45:36.287 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:45:36.287 UTC [msp] GetDefaultSigningIdentity -> DEBU 044 Obtaining default signing identity
2018-08-07 02:45:36.287 UTC [msp/identity] Sign -> DEBU 045 Sign: plaintext: 0AD1060A1508021A0608D08FA4DB0522...C2F2ED2EF0B45937BFCDDE5333EED203 
2018-08-07 02:45:36.287 UTC [msp/identity] Sign -> DEBU 046 Sign: digest: 8130CD8476B357A487219831107C06F93DC8AE12DF9CB2EB04BB199802467E58 
2018-08-07 02:45:36.287 UTC [grpc] Printf -> DEBU 047 parsed scheme: ""
2018-08-07 02:45:36.287 UTC [grpc] Printf -> DEBU 048 scheme "" not registered, fallback to default scheme
2018-08-07 02:45:36.287 UTC [grpc] Printf -> DEBU 049 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-08-07 02:45:36.287 UTC [grpc] Printf -> DEBU 04a ClientConn switching balancer to "pick_first"
2018-08-07 02:45:36.287 UTC [grpc] Printf -> DEBU 04b pickfirstBalancer: HandleSubConnStateChange: 0xc42025aed0, CONNECTING
2018-08-07 02:45:36.287 UTC [grpc] Printf -> DEBU 04c pickfirstBalancer: HandleSubConnStateChange: 0xc42025aed0, READY
2018-08-07 02:45:36.314 UTC [channelCmd] update -> INFO 04d Successfully submitted channel update

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
total 20
drwxr-xr-x 3 root root 4096 Aug  7 10:46 example01
drwxr-xr-x 3 root root 4096 Aug  7 10:46 example02
drwxr-xr-x 3 root root 4096 Aug  7 10:46 example03
drwxr-xr-x 3 root root 4096 Aug  7 10:46 example04
drwxr-xr-x 3 root root 4096 Aug  7 10:46 example05



# 如上我们挂载的地址为 github.com/hyperledger/fabric/jicki/chaincode/go


# 注: 这里面的 example02 的 package 为 example02 会报错

Error: could not assemble transaction, err Proposal response was not successful, error code 500, msg failed to execute transaction 819b581ce88604e9b6651764324876f2ca7a47d7aeb7ee307f273af867a4a134: error starting container: error starting container: API error (404): oci runtime error: container_linux.go:247: starting container process caused "exec: \"chaincode\": executable file not found in $PATH"


# 将 chaincode.go  chaincode_test.go 中  package 修改成 main 然后在最下面增加 main()函数

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

2018-08-07 02:48:05.296 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:48:05.296 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:48:05.297 UTC [grpc] Printf -> DEBU 037 parsed scheme: ""
2018-08-07 02:48:05.297 UTC [grpc] Printf -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-08-07 02:48:05.297 UTC [grpc] Printf -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:48:05.297 UTC [grpc] Printf -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-08-07 02:48:05.297 UTC [grpc] Printf -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc42049c2f0, CONNECTING
2018-08-07 02:48:05.298 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc42049c2f0, READY
2018-08-07 02:48:05.298 UTC [grpc] Printf -> DEBU 03d parsed scheme: ""
2018-08-07 02:48:05.298 UTC [grpc] Printf -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-08-07 02:48:05.298 UTC [grpc] Printf -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:48:05.298 UTC [grpc] Printf -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-08-07 02:48:05.299 UTC [grpc] Printf -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4202ce020, CONNECTING
2018-08-07 02:48:05.299 UTC [grpc] Printf -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4202ce020, READY
2018-08-07 02:48:05.300 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:48:05.300 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 044 Using default escc
2018-08-07 02:48:05.300 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 045 Using default vscc
2018-08-07 02:48:05.300 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 046 java chaincode disabled
2018-08-07 02:48:05.338 UTC [golang-platform] getCodeFromFS -> DEBU 047 getCodeFromFS github.com/hyperledger/fabric/jicki/chaincode/go/example02
2018-08-07 02:48:05.450 UTC [golang-platform] func1 -> DEBU 048 Discarding GOROOT package fmt
2018-08-07 02:48:05.450 UTC [golang-platform] func1 -> DEBU 049 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-08-07 02:48:05.450 UTC [golang-platform] func1 -> DEBU 04a Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-08-07 02:48:05.450 UTC [golang-platform] func1 -> DEBU 04b Discarding GOROOT package strconv
2018-08-07 02:48:05.450 UTC [golang-platform] func1 -> DEBU 04c skipping dir: /opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/cmd
2018-08-07 02:48:05.451 UTC [golang-platform] GetDeploymentPayload -> DEBU 04d done
2018-08-07 02:48:05.451 UTC [container] WriteFileToPackage -> DEBU 04e Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode.go
2018-08-07 02:48:05.451 UTC [container] WriteFileToPackage -> DEBU 04f Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode_test.go
2018-08-07 02:48:05.452 UTC [msp/identity] Sign -> DEBU 050 Sign: plaintext: 0A98070A5C08031A0C08E590A4DB0510...C7F84F000000FFFFEB6FF30D00280000 
2018-08-07 02:48:05.452 UTC [msp/identity] Sign -> DEBU 051 Sign: digest: 4CBC4EA6119D83478DB9214C41CAFB439A4D5CE4C955B0989F4AA07B94238B1B 
2018-08-07 02:48:05.469 UTC [chaincodeCmd] install -> INFO 052 Installed remotely response:<status:200 payload:"OK" > 

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

2018-08-07 02:48:24.440 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:48:24.441 UTC [msp] Validate -> DEBU 036 MSP Org2MSP validating identity
2018-08-07 02:48:24.441 UTC [grpc] Printf -> DEBU 037 parsed scheme: ""
2018-08-07 02:48:24.441 UTC [grpc] Printf -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-08-07 02:48:24.441 UTC [grpc] Printf -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-08-07 02:48:24.441 UTC [grpc] Printf -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-08-07 02:48:24.441 UTC [grpc] Printf -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc420229510, CONNECTING
2018-08-07 02:48:24.442 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc420229510, READY
2018-08-07 02:48:24.442 UTC [grpc] Printf -> DEBU 03d parsed scheme: ""
2018-08-07 02:48:24.442 UTC [grpc] Printf -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-08-07 02:48:24.442 UTC [grpc] Printf -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org2.jicki.cn:7051 0  <nil>}]
2018-08-07 02:48:24.442 UTC [grpc] Printf -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-08-07 02:48:24.443 UTC [grpc] Printf -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4202b8a20, CONNECTING
2018-08-07 02:48:24.443 UTC [grpc] Printf -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4202b8a20, READY
2018-08-07 02:48:24.444 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:48:24.444 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 044 Using default escc
2018-08-07 02:48:24.444 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 045 Using default vscc
2018-08-07 02:48:24.444 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 046 java chaincode disabled
2018-08-07 02:48:24.481 UTC [golang-platform] getCodeFromFS -> DEBU 047 getCodeFromFS github.com/hyperledger/fabric/jicki/chaincode/go/example02
2018-08-07 02:48:24.588 UTC [golang-platform] func1 -> DEBU 048 Discarding GOROOT package fmt
2018-08-07 02:48:24.588 UTC [golang-platform] func1 -> DEBU 049 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-08-07 02:48:24.588 UTC [golang-platform] func1 -> DEBU 04a Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-08-07 02:48:24.588 UTC [golang-platform] func1 -> DEBU 04b Discarding GOROOT package strconv
2018-08-07 02:48:24.588 UTC [golang-platform] func1 -> DEBU 04c skipping dir: /opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/cmd
2018-08-07 02:48:24.588 UTC [golang-platform] GetDeploymentPayload -> DEBU 04d done
2018-08-07 02:48:24.588 UTC [container] WriteFileToPackage -> DEBU 04e Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode.go
2018-08-07 02:48:24.589 UTC [container] WriteFileToPackage -> DEBU 04f Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/example02/chaincode_test.go
2018-08-07 02:48:24.589 UTC [msp/identity] Sign -> DEBU 050 Sign: plaintext: 0A98070A5C08031A0C08F890A4DB0510...C7F84F000000FFFFEB6FF30D00280000 
2018-08-07 02:48:24.589 UTC [msp/identity] Sign -> DEBU 051 Sign: digest: 8322AEA3AA9478AC330B78C685B29F2FD85505EBC838DB13A71BEB58AFD822B9 
2018-08-07 02:48:24.606 UTC [chaincodeCmd] install -> INFO 052 Installed remotely response:<status:200 payload:"OK" > 

```

### 实例化 Chaincode


> 这里无论多少个 peer 节点, 实例化只需要实例化一次，就可以。


```
# 实例化合约 (未认证)
peer chaincode instantiate -o orderer0.jicki.cn:7050 -C mychannel -n example2 -c '{"Args":["init","A","200","B","500"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" -v 1.0



# 实例化合约 (已认证)
peer chaincode instantiate -o orderer0.jicki.cn:7050 --tls $CORE_PEER_TLS_ENABLED --cafile /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem -C mychannel -n example2 -c '{"Args":["init","A","100","B","50"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" -v 1.0




# 输出如下:

2018-08-07 02:49:00.544 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:49:00.544 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:49:00.545 UTC [grpc] Printf -> DEBU 037 parsed scheme: ""
2018-08-07 02:49:00.545 UTC [grpc] Printf -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-08-07 02:49:00.545 UTC [grpc] Printf -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:49:00.545 UTC [grpc] Printf -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-08-07 02:49:00.545 UTC [grpc] Printf -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4204ca910, CONNECTING
2018-08-07 02:49:00.546 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4204ca910, READY
2018-08-07 02:49:00.546 UTC [grpc] Printf -> DEBU 03d parsed scheme: ""
2018-08-07 02:49:00.546 UTC [grpc] Printf -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-08-07 02:49:00.546 UTC [grpc] Printf -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:49:00.546 UTC [grpc] Printf -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-08-07 02:49:00.547 UTC [grpc] Printf -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc420549310, CONNECTING
2018-08-07 02:49:00.548 UTC [grpc] Printf -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc420549310, READY
2018-08-07 02:49:00.548 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:49:00.548 UTC [grpc] Printf -> DEBU 044 parsed scheme: ""
2018-08-07 02:49:00.548 UTC [grpc] Printf -> DEBU 045 scheme "" not registered, fallback to default scheme
2018-08-07 02:49:00.548 UTC [grpc] Printf -> DEBU 046 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-08-07 02:49:00.548 UTC [grpc] Printf -> DEBU 047 ClientConn switching balancer to "pick_first"
2018-08-07 02:49:00.548 UTC [grpc] Printf -> DEBU 048 pickfirstBalancer: HandleSubConnStateChange: 0xc42006b380, CONNECTING
2018-08-07 02:49:00.548 UTC [grpc] Printf -> DEBU 049 pickfirstBalancer: HandleSubConnStateChange: 0xc42006b380, READY
2018-08-07 02:49:00.548 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 04a Using default escc
2018-08-07 02:49:00.548 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 04b Using default vscc
2018-08-07 02:49:00.549 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 04c java chaincode disabled
2018-08-07 02:49:00.549 UTC [msp/identity] Sign -> DEBU 04d Sign: plaintext: 0AA3070A6708031A0C089C91A4DB0510...324D53500A04657363630A0476736363 
2018-08-07 02:49:00.549 UTC [msp/identity] Sign -> DEBU 04e Sign: digest: 1852003550AFAEB4A9DD2F01A3F4DA89346E64FCC1B6056D45CCC297A9763A1F 
2018-08-07 02:49:13.285 UTC [msp/identity] Sign -> DEBU 04f Sign: plaintext: 0AA3070A6708031A0C089C91A4DB0510...46C5367A5E41CCC7C4704B72B4D5F2D4 
2018-08-07 02:49:13.285 UTC [msp/identity] Sign -> DEBU 050 Sign: digest: 8D57F12A907CA041F15725217B3EC56FC52C2E4A5A9FF8578FD90C18AC65F817 

```


### 操作智能合约


```
# query 查询方法


# 查询 A 账户里的余额

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'


# 输出如下:

2018-08-07 02:49:40.807 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:49:40.807 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:49:40.807 UTC [grpc] Printf -> DEBU 037 parsed scheme: ""
2018-08-07 02:49:40.807 UTC [grpc] Printf -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-08-07 02:49:40.807 UTC [grpc] Printf -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:49:40.807 UTC [grpc] Printf -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-08-07 02:49:40.808 UTC [grpc] Printf -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4201b5510, CONNECTING
2018-08-07 02:49:40.809 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4201b5510, READY
2018-08-07 02:49:40.809 UTC [grpc] Printf -> DEBU 03d parsed scheme: ""
2018-08-07 02:49:40.809 UTC [grpc] Printf -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-08-07 02:49:40.809 UTC [grpc] Printf -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:49:40.809 UTC [grpc] Printf -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-08-07 02:49:40.809 UTC [grpc] Printf -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc420232a20, CONNECTING
2018-08-07 02:49:40.809 UTC [grpc] Printf -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc420232a20, READY
2018-08-07 02:49:40.811 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:49:40.811 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 044 java chaincode disabled
2018-08-07 02:49:40.811 UTC [msp/identity] Sign -> DEBU 045 Sign: plaintext: 0AA7070A6B08031A0C08C491A4DB0510...706C65321A0A0A0571756572790A0141 
2018-08-07 02:49:40.811 UTC [msp/identity] Sign -> DEBU 046 Sign: digest: CD3360D1248F37AC518F8B9D55B46BA976526A6D115AE68A2EC3A93423C22D7C 
200


# 可以看到 返回 200
```


```
# 查询 B 账户里的余额

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'


# 输出如下:

2018-08-07 02:50:09.596 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:50:09.596 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:50:09.597 UTC [grpc] Printf -> DEBU 037 parsed scheme: ""
2018-08-07 02:50:09.597 UTC [grpc] Printf -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-08-07 02:50:09.597 UTC [grpc] Printf -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:50:09.597 UTC [grpc] Printf -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-08-07 02:50:09.597 UTC [grpc] Printf -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc42053fc70, CONNECTING
2018-08-07 02:50:09.597 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc42053fc70, READY
2018-08-07 02:50:09.598 UTC [grpc] Printf -> DEBU 03d parsed scheme: ""
2018-08-07 02:50:09.598 UTC [grpc] Printf -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-08-07 02:50:09.598 UTC [grpc] Printf -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:50:09.598 UTC [grpc] Printf -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-08-07 02:50:09.598 UTC [grpc] Printf -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc420247180, CONNECTING
2018-08-07 02:50:09.599 UTC [grpc] Printf -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc420247180, READY
2018-08-07 02:50:09.599 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:50:09.599 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 044 java chaincode disabled
2018-08-07 02:50:09.599 UTC [msp/identity] Sign -> DEBU 045 Sign: plaintext: 0AA7070A6B08031A0C08E191A4DB0510...706C65321A0A0A0571756572790A0142 
2018-08-07 02:50:09.599 UTC [msp/identity] Sign -> DEBU 046 Sign: digest: 1C28B767E687B1C2D9CCE05C14735F2C8D250BA3C8442F3EB077BADFCFB1D65B 
500


# 可以看到 返回 500

```


```
# invoke 转账方法


# 从A账户 转账 20 个币 到 B 账户


peer chaincode invoke -C mychannel -n example2 -c '{"Args":["invoke", "A", "B", "100"]}'


# 输出如下:

2018-08-07 02:50:40.962 UTC [grpc] Printf -> DEBU 0bf parsed scheme: ""
2018-08-07 02:50:40.962 UTC [grpc] Printf -> DEBU 0c0 scheme "" not registered, fallback to default scheme
2018-08-07 02:50:40.962 UTC [grpc] Printf -> DEBU 0c1 ccResolverWrapper: sending new addresses to cc: [{orderer0.jicki.cn:7050 0  <nil>}]
2018-08-07 02:50:40.962 UTC [grpc] Printf -> DEBU 0c2 ClientConn switching balancer to "pick_first"
2018-08-07 02:50:40.962 UTC [grpc] Printf -> DEBU 0c3 pickfirstBalancer: HandleSubConnStateChange: 0xc4202e6bc0, CONNECTING
2018-08-07 02:50:40.963 UTC [grpc] Printf -> DEBU 0c4 pickfirstBalancer: HandleSubConnStateChange: 0xc4202e6bc0, READY
2018-08-07 02:50:40.963 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 0c5 java chaincode disabled
2018-08-07 02:50:40.963 UTC [msp/identity] Sign -> DEBU 0c6 Sign: plaintext: 0AA7070A6B08031A0C088092A4DB0510...6E766F6B650A01410A01420A03313030 
2018-08-07 02:50:40.963 UTC [msp/identity] Sign -> DEBU 0c7 Sign: digest: A9C3D9F911BBD38516C15CC0AABF05588F4EDCBE8686AFA59947A62EE12F41F4 
2018-08-07 02:50:40.972 UTC [msp/identity] Sign -> DEBU 0c8 Sign: plaintext: 0AA7070A6B08031A0C088092A4DB0510...8144630C48C565D11D1AA2562DB4FB70 
2018-08-07 02:50:40.972 UTC [msp/identity] Sign -> DEBU 0c9 Sign: digest: 0E9C53305451D3168CA61B8E525C416822E24A4C2C57ECE908C51C7A3F671B8D 
2018-08-07 02:50:40.978 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> DEBU 0ca ESCC invoke result: version:1 response:<status:200 > payload:"\n ~\303\035uF\270\254\024\3655\272|.\006\217\362}\242\340\311\270_\354}\353q\303Y\324\352J\000\022f\nN\0222\n\010example2\022&\n\007\n\001A\022\002\010\003\n\007\n\001B\022\002\010\003\032\010\n\001A\032\003100\032\010\n\001B\032\003600\022\030\n\004lscc\022\020\n\016\n\010example2\022\002\010\003\032\003\010\310\001\"\017\022\010example2\032\0031.0" endorsement:<endorser:"\n\007Org1MSP\022\216\006-----BEGIN CERTIFICATE-----\nMIICEzCCAbqgAwIBAgIRALNKLIQGXYydTA4WqXE0UUwwCgYIKoZIzj0EAwIwZzEL\nMAkGA1UEBhMCQ04xEjAQBgNVBAgTCUd1YW5nRG9uZzERMA8GA1UEBxMIU2hlblpo\nZW4xFjAUBgNVBAoTDW9yZzEuamlja2kubWUxGTAXBgNVBAMTEGNhLm9yZzEuamlj\na2kubWUwHhcNMTgwODA2MDY0MDU0WhcNMjgwODAzMDY0MDU0WjBhMQswCQYDVQQG\nEwJDTjESMBAGA1UECBMJR3VhbmdEb25nMREwDwYDVQQHEwhTaGVuWmhlbjENMAsG\nA1UECxMEcGVlcjEcMBoGA1UEAxMTcGVlcjAub3JnMS5qaWNraS5tZTBZMBMGByqG\nSM49AgEGCCqGSM49AwEHA0IABIJsThrTC3lNx4L97bB7YopOLaQNoCXhi/KAeMur\neYOchSQh3LbTb2dvf9hlW1DlN5y+QjrxAo9QNn6/fMk74uGjTTBLMA4GA1UdDwEB\n/wQEAwIHgDAMBgNVHRMBAf8EAjAAMCsGA1UdIwQkMCKAIExyclAVcPAhAWcUhubH\njfdv0sZOnU6wmqcDppjwTTFZMAoGCCqGSM49BAMCA0cAMEQCIEFjh/Q1281Puzjg\n7c5/BhCv8+HvDM8lQZ3kHV3CQzTsAiAJ2cWndgCLFoXqiCqo1xzEcEHNO1O9c9sn\ngnPbCBvW4g==\n-----END CERTIFICATE-----\n" signature:"0E\002!\000\241\244i\001\225.y1e\225@\261\003\322\275\326\013[\364\304Ri7I\304\024\361#\\_J`\002 e\310\335&\241\376\270EHh4\246\010Y\264\270\201Dc\014H\305e\321\035\032\242V-\264\373p" > 
2018-08-07 02:50:40.978 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 0cb Chaincode invoke successful. result: status:200



# 可以看到返回 invoke successful. result: status:200 成功
```


```
# 这里再查询 A 与 B 的账户

# A 账户余额 

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'  


# 输出如下:

2018-08-07 02:51:09.356 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:51:09.356 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:51:09.357 UTC [grpc] Printf -> DEBU 037 parsed scheme: ""
2018-08-07 02:51:09.357 UTC [grpc] Printf -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-08-07 02:51:09.357 UTC [grpc] Printf -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:51:09.357 UTC [grpc] Printf -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-08-07 02:51:09.357 UTC [grpc] Printf -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc420533510, CONNECTING
2018-08-07 02:51:09.358 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc420533510, READY
2018-08-07 02:51:09.358 UTC [grpc] Printf -> DEBU 03d parsed scheme: ""
2018-08-07 02:51:09.358 UTC [grpc] Printf -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-08-07 02:51:09.358 UTC [grpc] Printf -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:51:09.358 UTC [grpc] Printf -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-08-07 02:51:09.359 UTC [grpc] Printf -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc4205db310, CONNECTING
2018-08-07 02:51:09.359 UTC [grpc] Printf -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc4205db310, READY
2018-08-07 02:51:09.360 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:51:09.360 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 044 java chaincode disabled
2018-08-07 02:51:09.360 UTC [msp/identity] Sign -> DEBU 045 Sign: plaintext: 0AA7070A6B08031A0C089D92A4DB0510...706C65321A0A0A0571756572790A0141 
2018-08-07 02:51:09.360 UTC [msp/identity] Sign -> DEBU 046 Sign: digest: 4551936C8DDCEC3821AB6868B5D9162D86B31B32E1A34585CC3E70B06C717C83 
100
```


```
# B 账户余额 

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'   



# 输出如下:

2018-08-07 02:51:34.433 UTC [msp] setupSigningIdentity -> DEBU 035 Signing identity expires at 2028-08-03 06:40:54 +0000 UTC
2018-08-07 02:51:34.433 UTC [msp] Validate -> DEBU 036 MSP Org1MSP validating identity
2018-08-07 02:51:34.434 UTC [grpc] Printf -> DEBU 037 parsed scheme: ""
2018-08-07 02:51:34.434 UTC [grpc] Printf -> DEBU 038 scheme "" not registered, fallback to default scheme
2018-08-07 02:51:34.434 UTC [grpc] Printf -> DEBU 039 ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:51:34.434 UTC [grpc] Printf -> DEBU 03a ClientConn switching balancer to "pick_first"
2018-08-07 02:51:34.434 UTC [grpc] Printf -> DEBU 03b pickfirstBalancer: HandleSubConnStateChange: 0xc4202e7150, CONNECTING
2018-08-07 02:51:34.435 UTC [grpc] Printf -> DEBU 03c pickfirstBalancer: HandleSubConnStateChange: 0xc4202e7150, READY
2018-08-07 02:51:34.435 UTC [grpc] Printf -> DEBU 03d parsed scheme: ""
2018-08-07 02:51:34.435 UTC [grpc] Printf -> DEBU 03e scheme "" not registered, fallback to default scheme
2018-08-07 02:51:34.435 UTC [grpc] Printf -> DEBU 03f ccResolverWrapper: sending new addresses to cc: [{peer0.org1.jicki.cn:7051 0  <nil>}]
2018-08-07 02:51:34.435 UTC [grpc] Printf -> DEBU 040 ClientConn switching balancer to "pick_first"
2018-08-07 02:51:34.435 UTC [grpc] Printf -> DEBU 041 pickfirstBalancer: HandleSubConnStateChange: 0xc42050a660, CONNECTING
2018-08-07 02:51:34.436 UTC [grpc] Printf -> DEBU 042 pickfirstBalancer: HandleSubConnStateChange: 0xc42050a660, READY
2018-08-07 02:51:34.437 UTC [msp] GetDefaultSigningIdentity -> DEBU 043 Obtaining default signing identity
2018-08-07 02:51:34.437 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 044 java chaincode disabled
2018-08-07 02:51:34.437 UTC [msp/identity] Sign -> DEBU 045 Sign: plaintext: 0AA7070A6B08031A0C08B692A4DB0510...706C65321A0A0A0571756572790A0142 
2018-08-07 02:51:34.437 UTC [msp/identity] Sign -> DEBU 046 Sign: digest: EE474CDC8DD863E3F9163AE8A73CE43EF25C9FF417DD940EE133BDC1A6646CB8 
600

```

```
# 查看 peer0.org1.jicki.cn 节点里 生成的容器

[root@localhost jicki]# docker ps -a
CONTAINER ID        IMAGE                                                                                                     COMMAND                  CREATED             STATUS              PORTS                                                                                         NAMES
ddb72b9ca769        jicki-peer0.org1.jicki.cn-example2-1.0-58e362edb293b97212b572bd603337608795d8168a8a3632d2a14ac12fa6ae70   "chaincode -peer.add…"   2 minutes ago       Up 2 minutes                                                                                                      jicki-peer0.org1.jicki.cn-example2-1.0

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


# 更新版本为 1.1 的合约
peer chaincode upgrade -o orderer0.jicki.cn:7050 -C mychannel -n example2 -c '{"Args":["init","A","100","B","50"]}' -P "OR ('Org1MSP.member','Org2MSP.member')" -v 1.1


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


