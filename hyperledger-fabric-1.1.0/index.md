# hyperledger-fabric v1.1.0



> 单机版 手动部署 fabric v1.1.0 实例为1.0 版本的, 使用以及说明。

# 部署 hyperledger-fabric v1.1.0

## 官方地址

> 文档以官方文档为主 http://hyperledger-fabric.readthedocs.io/en/release-1.1/prereqs.html

```
# 官网 github
https://github.com/hyperledger/fabric
```

## 环境准备

* 安装 Docker (用于 fabric 服务启动) 
    

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
 Version:      17.05.0-ce
 API version:  1.29
 Go version:   go1.7.5
 Git commit:   89658be
 Built:        Thu May  4 22:06:25 2017
 OS/Arch:      linux/amd64

Server:
 Version:      17.05.0-ce
 API version:  1.29 (minimum version 1.12)
 Go version:   go1.7.5
 Git commit:   89658be
 Built:        Thu May  4 22:06:25 2017
 OS/Arch:      linux/amd64
 Experimental: false
```




* 安装 Docker-compose (用于 docker 容器服务统一管理 编排)


```
# 安装 pip

curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py

python get-pip.py

```


```
# 安装 docker-compose

pip install docker-compose

```


```
docker-compose version
docker-compose version 1.16.1, build 6d1ac219
docker-py version: 2.5.1
CPython version: 2.7.5
OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013
```


* Golang (用于 fabric cli 服务的调用， ca 服务证书生成 )


```
mkdir -p /opt/golang
mkdir -p /opt/gopath

# 国外地址
curl -O https://storage.googleapis.com/golang/go1.10.linux-amd64.tar.gz

# 国内地址
curl -O https://studygolang.com/dl/golang/go1.10.linux-amd64.tar.gz


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


## 环境规划

> 相关hostname 必须配置 dns 


|节点标识|hostname|IP|开放端口|系统|
|--------|--------|--|--------|----------|---|--|
|orderer节点|orderer.jicki.cn|192.168.168.100|7050|CentOS 7 x64|
|ca节点|ca0.org1.jicki.cn|192.168.168.100|7054|CentOS 7 x64|
|peer节点|peer0.org1.jicki.cn|192.168.168.100|7051, 7052, 7053|CentOS 7 x64|




## Hyperledger Fabric 源码


```
# 下载 Fabric 源码, 源码中 import 的路径为github.com/hyperledger/fabric ,所以我们要按照这个路径

mkdir -p /opt/jicki/github.com/hyperledger

cd /opt/jicki/github.com/hyperledger

git clone https://github.com/hyperledger/fabric


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


```
# 下载官方证书生成软件(均为二进制文件)
# 官方离线下载地址为 https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/


# 选择相应版本 CentOS 选择 linux-amd64-1.1.0  Mac 选择 darwin-amd64-1.1.0


# 下载地址为: https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.1.0/hyperledger-fabric-linux-amd64-1.1.0.tar.gz

cd /opt/jicki

wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.1.0/hyperledger-fabric-linux-amd64-1.1.0.tar.gz

tar zxvf hyperledger-fabric-linux-amd64-1.1.0.tar.gz

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
# 下载1.0 版本的 configtx.yaml  与 crypto-config.yaml 。


cd /opt/jicki/

wget https://raw.githubusercontent.com/hyperledger/fabric/v1.0.0/examples/e2e_cli/crypto-config.yaml

wget https://raw.githubusercontent.com/hyperledger/fabric/v1.0.0/examples/e2e_cli/configtx.yaml


# 这里修改相应 jicki.cn 为 jicki.cn

sed -i 's/example\.com/jicki\.me/g' *.yaml


# 然后这里使用 cryptogen 软件来生成相应的证书了

[root@localhost jicki]# cryptogen generate --config=./crypto-config.yaml
org1.jicki.cn
org2.jicki.cn



# 生成一个 crypto-config 证书目录



# 这里使用 configtxgen 来创建 创世区块


# 首先需要创建一个文件夹
mkdir -p /opt/jicki/channel-artifacts


# 创建 创世区块  TwoOrgsOrdererGenesis 名称为 configtx.yaml 中 Profiles 字段下的

[root@localhost jicki]# configtxgen -profile TwoOrgsOrdererGenesis -outputBlock ./channel-artifacts/genesis.block
2018-06-05 10:26:12.714 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-06-05 10:26:12.726 CST [common/tools/configtxgen] doOutputBlock -> INFO 002 Generating genesis block
2018-06-05 10:26:12.726 CST [common/tools/configtxgen] doOutputBlock -> INFO 003 Writing genesis block

# 创世区块 是在 orderer 服务中使用

[root@localhost jicki]# ls -lt channel-artifacts/
总用量 12
-rw-r--r-- 1 root root 8951 6月   5 10:26 genesis.block



# 下面来生成一个 peer 服务 中使用的 tx 文件 TwoOrgsChannel 名称为 configtx.yaml 中 Profiles 字段下的
# 这里配置的 channelID 为 mychannel , 后续使用到

[root@localhost jicki]# configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/mychannel.tx -channelID mychannel
2018-06-05 10:30:25.279 CST [common/tools/configtxgen] main -> INFO 001 Loading configuration
2018-06-05 10:30:25.291 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 002 Generating new channel configtx
2018-06-05 10:30:25.323 CST [common/tools/configtxgen] doOutputChannelCreateTx -> INFO 003 Writing new channel tx



[root@localhost jicki]# ls -lt channel-artifacts/
总用量 16
-rw-r--r-- 1 root root  308 6月   5 10:30 mychannel.tx
-rw-r--r-- 1 root root 8951 6月   5 10:26 genesis.block

```


## 配置 Hyperledger Fabric Orderer


```
# 创建文件 docker-compose-orderer.yaml

# 创建于 /opt/jicki 目录下

vi  docker-compose-orderer.yaml



version: '2'

services:

  orderer.jicki.cn:
    container_name: orderer.jicki.cn
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
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer.jicki.cn/tls/:/var/hyperledger/orderer/tls
    networks:
      default:
        aliases:
          - jicki
    ports:
      - 7050:7050

```




## 配置 Hyperledger Fabric Peer





```
# 创建 peer yaml 文件

# 创建于 /opt/jicki 目录下

# 注: 如下目录中相关的 ca 证书请替换为 各自生成的, 本文目录为 /opt/jicki/crypto-config/peerOrganizations/org1.jicki.cn/ca

# cli 部分  ./chaincode/go/:/opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go
# 为 智能合约的目录 我们约定为这个目录 需要预先创建 

mkdir -p /opt/jicki/chaincode/go

cd /opt/jicki/chaincode/go

# 创建以后~我们拷贝官方的 例子进来，方便后面进行合约测试

cp -r /opt/jicki/github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example0* /opt/jicki/chaincode/go/


# 官方这里有5个例子

[root@localhost jicki]# ls -lt chaincode/go/
总用量 0
drwxr-xr-x 2 root root 81 6月   5 11:06 chaincode_example02
drwxr-xr-x 2 root root 69 6月   5 11:06 chaincode_example03
drwxr-xr-x 2 root root 69 6月   5 11:06 chaincode_example04
drwxr-xr-x 2 root root 81 6月   5 11:06 chaincode_example05
drwxr-xr-x 2 root root 43 6月   5 11:06 chaincode_example01







vi  docker-compose-peer.yaml


version: '2'

services:

  couchdb:
    container_name: couchdb
    image: hyperledger/fabric-couchdb
    # Comment/Uncomment the port mapping if you want to hide/expose the CouchDB service,
    # for example map it to utilize Fauxton User Interface in dev environments.
    ports:
      - "5984:5984"
    volumes:
      # 数据持久化，用于存储链码值
      - /opt/jicki/couchdb/data:/opt/couchdb/data

  ca0.org1.jicki.cn:
    container_name: ca0.org1.jicki.cn
    image: hyperledger/fabric-ca
    environment:
      - FABRIC_CA_HOME=/etc/hyperledger/fabric-ca-server
      - FABRIC_CA_SERVER_CA_NAME=ca
      - FABRIC_CA_SERVER_TLS_ENABLED=false
      - FABRIC_CA_SERVER_TLS_CERTFILE=/etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem
      - FABRIC_CA_SERVER_TLS_KEYFILE=/etc/hyperledger/fabric-ca-server-config/ebed980e19743358897c6c7e9069dbc789a872097af8d142bdc69f0ac60c89d5_sk
    ports:
      - "7054:7054"
    command: sh -c 'fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org1.jicki.cn-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/ebed980e19743358897c6c7e9069dbc789a872097af8d142bdc69f0ac60c89d5_sk -b admin:adminpw -d'
    volumes:
      - ./crypto-config/peerOrganizations/org1.jicki.cn/ca/:/etc/hyperledger/fabric-ca-server-config

  peer0.org1.jicki.cn:
    container_name: peer0.org1.jicki.cn
    image: hyperledger/fabric-peer
    environment:
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb:5984

      - CORE_PEER_ID=peer0.org1.jicki.cn
      - CORE_PEER_NETWORKID=jicki
      - CORE_PEER_ADDRESS=peer0.org1.jicki.cn:7051
      - CORE_PEER_CHAINCODELISTENADDRESS=peer0.org1.jicki.cn:7052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org1.jicki.cn:7051
      - CORE_PEER_LOCALMSPID=Org1MSP

      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # the following setting starts chaincode containers on the same
      # bridge network as the peers
      # https://docs.docker.com/compose/networking/
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
        - /opt/jicki/peer0:/var/hyperledger/production
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    ports:
      - 7051:7051
      - 7052:7052
      - 7053:7053
    depends_on:
      - couchdb
    networks:
      default:
        aliases:
          - jicki

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
        - ./chaincode/go/:/opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go
        - ./crypto-config:/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/
        - ./channel-artifacts:/opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
    depends_on:
      - peer0.org1.jicki.cn

```



## 启动 Hyperledger Fabric 服务


```
# 启动 orderer 服务

[root@localhost jicki]# docker-compose -f docker-compose-orderer.yaml up -d
Creating network "jicki_default" with the default driver
Pulling orderer.jicki.cn (hyperledger/fabric-orderer:latest)...
latest: Pulling from hyperledger/fabric-orderer
1be7f2b886e8: Pull complete
6fbc4a21b806: Pull complete
c71a6f8e1378: Pull complete
4be3072e5a37: Pull complete
06c6d2f59700: Pull complete
4d536120d8a5: Pull complete
0baaf9ec263e: Pull complete
770563795186: Pull complete
61d33418a569: Pull complete
b1b98004e7c6: Pull complete
Digest: sha256:7a0a6ca2bbddff69ddf63615cdddfe46bf5f2fe7c55530a092d597d99bd2a4bb
Status: Downloaded newer image for hyperledger/fabric-orderer:latest
Creating orderer.jicki.cn ... 
Creating orderer.jicki.cn ... done


```



```
# 启动 peer 服务

[root@localhost jicki]# docker-compose -f docker-compose-peer.yaml up -d
Creating ca0.org1.jicki.cn ... 
Creating couchdb ... 
Creating couchdb ... donee ... done
Creating ca0.org1.jicki.cn
Creating peer0.org1.jicki.cn ... 
Creating peer0.org1.jicki.cn ... done
Creating cli ... 
Creating cli ... done


```


```
# 下载的镜像

[root@payment jicki]# docker images
REPOSITORY                                  TAG                   IMAGE ID            CREATED             SIZE
hyperledger/fabric-couchdb                  latest                35228d48a25a        2 months ago        1.56GB
hyperledger/fabric-ca                       latest                72617b4fa9b4        2 months ago        299MB
hyperledger/fabric-tools                    latest                b7bfddf508bc        2 months ago        1.46GB
hyperledger/fabric-orderer                  latest                ce0c810df36a        2 months ago        180MB
hyperledger/fabric-peer                     latest                b023f9be0771        2 months ago        187MB

```




```
# 查看启动的服务

[root@localhost jicki]# docker ps -a
77962643125a        hyperledger/fabric-tools                          "/bin/bash"              About a minute ago   Up About a minute                                                                                                                                       cli
8334cd884f99        hyperledger/fabric-peer                           "peer node start"        About a minute ago   Up About a minute       0.0.0.0:7051-7053->7051-7053/tcp                                                                                                peer0.org1.jicki.cn
7faf1692b1c8        hyperledger/fabric-ca                             "sh -c 'fabric-ca-..."   About a minute ago   Up About a minute       0.0.0.0:7054->7054/tcp                                                                                                          ca0.org1.jicki.cn
c3d581ca1957        hyperledger/fabric-couchdb                        "tini -- /docker-e..."   About a minute ago   Up About a minute       4369/tcp, 9100/tcp, 0.0.0.0:5984->5984/tcp                                                                                      couchdb
435f6268ed57        hyperledger/fabric-orderer                        "orderer"                39 minutes ago       Up 39 minutes           0.0.0.0:7050->7050/tcp    

```


## Hyperledger Fabric 创建Channel 加盟


```
# 上面我们创建了 cli 容器，我们可以直接进入 容器里操作


[root@localhost jicki]# docker exec -it cli bash
root@77962643125a:/opt/gopath/src/github.com/hyperledger/fabric/peer# 


# 执行 创建命令

peer channel create -o orderer.jicki.cn:7050 -c mychannel -t 50 -f ./channel-artifacts/mychannel.tx


# 输出如下:

2018-06-05 03:58:27.812 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 03:58:27.812 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 03:58:27.813 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-06-05 03:58:27.813 UTC [msp] GetLocalMSP -> DEBU 004 Returning existing local MSP
2018-06-05 03:58:27.813 UTC [msp] GetDefaultSigningIdentity -> DEBU 005 Obtaining default signing identity
2018-06-05 03:58:27.814 UTC [msp] GetLocalMSP -> DEBU 006 Returning existing local MSP
2018-06-05 03:58:27.814 UTC [msp] GetDefaultSigningIdentity -> DEBU 007 Obtaining default signing identity
2018-06-05 03:58:27.814 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0A96060A074F7267314D5350128A062D...53616D706C65436F6E736F727469756D 
2018-06-05 03:58:27.816 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: B98D97859C943C258DC17F79C1AFDB97259206D8459A523B0D2063DD6A6EDE78 
2018-06-05 03:58:27.817 UTC [msp] GetLocalMSP -> DEBU 00a Returning existing local MSP
2018-06-05 03:58:27.817 UTC [msp] GetDefaultSigningIdentity -> DEBU 00b Obtaining default signing identity
2018-06-05 03:58:27.817 UTC [msp] GetLocalMSP -> DEBU 00c Returning existing local MSP
2018-06-05 03:58:27.817 UTC [msp] GetDefaultSigningIdentity -> DEBU 00d Obtaining default signing identity
2018-06-05 03:58:27.817 UTC [msp/identity] Sign -> DEBU 00e Sign: plaintext: 0ACD060A1508021A0608E394D8D80522...38F301FCA102E04B2C4451A710338CE6 
2018-06-05 03:58:27.817 UTC [msp/identity] Sign -> DEBU 00f Sign: digest: 5DF2DEECA8C542FB555728C1AF37A7135D2E73EC5A70C01DA110FA5FB7A4350B 
2018-06-05 03:58:27.855 UTC [msp] GetLocalMSP -> DEBU 010 Returning existing local MSP
2018-06-05 03:58:27.855 UTC [msp] GetDefaultSigningIdentity -> DEBU 011 Obtaining default signing identity
2018-06-05 03:58:27.855 UTC [msp] GetLocalMSP -> DEBU 012 Returning existing local MSP
2018-06-05 03:58:27.855 UTC [msp] GetDefaultSigningIdentity -> DEBU 013 Obtaining default signing identity
2018-06-05 03:58:27.855 UTC [msp/identity] Sign -> DEBU 014 Sign: plaintext: 0ACD060A1508021A0608E394D8D80522...7AC84BD8C96D12080A021A0012021A00 
2018-06-05 03:58:27.855 UTC [msp/identity] Sign -> DEBU 015 Sign: digest: C895D56A7881A4C5DB5BB30224F3F481B5F164E328F1220F63453B280888D492 
2018-06-05 03:58:27.856 UTC [channelCmd] readBlock -> DEBU 016 Got status: &{NOT_FOUND}
2018-06-05 03:58:27.856 UTC [msp] GetLocalMSP -> DEBU 017 Returning existing local MSP
2018-06-05 03:58:27.856 UTC [msp] GetDefaultSigningIdentity -> DEBU 018 Obtaining default signing identity
2018-06-05 03:58:27.857 UTC [channelCmd] InitCmdFactory -> INFO 019 Endorser and orderer connections initialized
2018-06-05 03:58:28.057 UTC [msp] GetLocalMSP -> DEBU 01a Returning existing local MSP
2018-06-05 03:58:28.057 UTC [msp] GetDefaultSigningIdentity -> DEBU 01b Obtaining default signing identity
2018-06-05 03:58:28.058 UTC [msp] GetLocalMSP -> DEBU 01c Returning existing local MSP
2018-06-05 03:58:28.058 UTC [msp] GetDefaultSigningIdentity -> DEBU 01d Obtaining default signing identity
2018-06-05 03:58:28.058 UTC [msp/identity] Sign -> DEBU 01e Sign: plaintext: 0ACD060A1508021A0608E494D8D80522...0EE8DDD220F812080A021A0012021A00 
2018-06-05 03:58:28.058 UTC [msp/identity] Sign -> DEBU 01f Sign: digest: 3025E2B47E9058D643182FD23B581EA8504099AD34B3CDF7B672D62FC8CEDC0B 
2018-06-05 03:58:28.062 UTC [channelCmd] readBlock -> DEBU 020 Received block: 0
2018-06-05 03:58:28.062 UTC [main] main -> INFO 021 Exiting.....




# 创建以后生成文件 mychannel.block

# ls -lt
total 12
-rw-r--r-- 1 root root 11804 Jun  5 03:58 mychannel.block
drwxr-xr-x 2 root root    57 Jun  5 02:30 channel-artifacts
drwxr-xr-x 4 root root    69 Jun  5 02:20 crypto

```


```
# 加入 此 channel 中

peer channel join -b mychannel.block


# 输出如下:


2018-06-05 04:00:28.379 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:00:28.379 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:00:28.381 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2018-06-05 04:00:28.382 UTC [msp/identity] Sign -> DEBU 004 Sign: plaintext: 0A94070A5C08011A0C08DC95D8D80510...EF2ADBC833421A080A000A000A000A00 
2018-06-05 04:00:28.384 UTC [msp/identity] Sign -> DEBU 005 Sign: digest: 285C7FC1C3276EBE9B90898D4CF6643E59741F578EC09FE2210311D56D191F81 
2018-06-05 04:00:28.485 UTC [channelCmd] executeJoin -> INFO 006 Successfully submitted proposal to join channel
2018-06-05 04:00:28.485 UTC [main] main -> INFO 007 Exiting.....

```



## Hyperledger Fabric 实例化测试

>  在上面我们已经拷贝了官方的例子，在 chaincode 下, 下面我们来测试一下


### 安装智能合约


```
# 如上我们挂载的地址为 github.com/hyperledger/fabric/jicki/chaincode/go


# 指定安装路径 安装

peer chaincode install -n mychannel -p github.com/hyperledger/fabric/jicki/chaincode/go/chaincode_example02 -v 1.0     



# 输出如下:


2018-06-05 04:03:55.916 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:03:55.916 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:03:55.917 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-06-05 04:03:55.917 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-06-05 04:03:55.917 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode disabled
2018-06-05 04:03:55.963 UTC [golang-platform] getCodeFromFS -> DEBU 006 getCodeFromFS github.com/hyperledger/fabric/jicki/chaincode/go/chaincode_example02
2018-06-05 04:03:56.105 UTC [golang-platform] func1 -> DEBU 007 Discarding GOROOT package fmt
2018-06-05 04:03:56.105 UTC [golang-platform] func1 -> DEBU 008 Discarding provided package github.com/hyperledger/fabric/core/chaincode/shim
2018-06-05 04:03:56.105 UTC [golang-platform] func1 -> DEBU 009 Discarding provided package github.com/hyperledger/fabric/protos/peer
2018-06-05 04:03:56.105 UTC [golang-platform] func1 -> DEBU 00a Discarding GOROOT package strconv
2018-06-05 04:03:56.106 UTC [golang-platform] GetDeploymentPayload -> DEBU 00b done
2018-06-05 04:03:56.106 UTC [container] WriteFileToPackage -> DEBU 00c Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/chaincode_example02/chaincode_example02.go
2018-06-05 04:03:56.107 UTC [container] WriteFileToPackage -> DEBU 00d Writing file to tarball: src/github.com/hyperledger/fabric/jicki/chaincode/go/chaincode_example02/chaincode_example02_test.go
2018-06-05 04:03:56.108 UTC [msp/identity] Sign -> DEBU 00e Sign: plaintext: 0A93070A5B08031A0B08AC97D8D80510...3F79FC370000FFFF2584A7C4002A0000 
2018-06-05 04:03:56.108 UTC [msp/identity] Sign -> DEBU 00f Sign: digest: 00863649A2C1F4DCF7913323182FF13ADEA767FC30C25D83179D1ECFBEF06180 
2018-06-05 04:03:56.124 UTC [chaincodeCmd] install -> DEBU 010 Installed remotely response:<status:200 payload:"OK" > 


```


### 实例化 Chaincode


```

peer chaincode instantiate -o orderer.jicki.cn:7050 -C mychannel -n mychannel -c '{"Args":["init","A","10","B","10"]}' -P "OR ('Org1MSP.member')" -v 1.0



# 输出如下:


2018-06-05 04:05:46.217 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:05:46.217 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:05:46.220 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-06-05 04:05:46.221 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-06-05 04:05:46.221 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode disabled
2018-06-05 04:05:46.221 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0A9E070A6608031A0B089A98D8D80510...314D53500A04657363630A0476736363 
2018-06-05 04:05:46.221 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: 461D88DFC289669A30F4B32BCCAE9AC22EF7F473A991796D71C515FE3C9E710E 
2018-06-05 04:06:07.391 UTC [msp/identity] Sign -> DEBU 008 Sign: plaintext: 0A9E070A6608031A0B089A98D8D80510...743A646439C8D2AC5A7A105153A92C88 
2018-06-05 04:06:07.391 UTC [msp/identity] Sign -> DEBU 009 Sign: digest: 16B5C05D014187EE64710445C16DFD912C9BFFA637AB098A392D3FD8DA23D293 
2018-06-05 04:06:07.393 UTC [main] main -> INFO 00a Exiting.....

```


### 操作智能合约


```
# query 查询方法


# 查询 A 账户里的余额

peer chaincode query -C mychannel -n mychannel -c '{"Args":["query","A"]}'


# 输出如下:

2018-06-05 04:07:20.190 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:07:20.190 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:07:20.190 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-06-05 04:07:20.190 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-06-05 04:07:20.191 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode disabled
2018-06-05 04:07:20.191 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0AA3070A6B08031A0B08F898D8D80510...6E6E656C1A0A0A0571756572790A0141 
2018-06-05 04:07:20.191 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: 26BD0EFDB8015795543FDFBEF605A71870580A6DCE9B4E1285B009642116594F 
Query Result: 10
2018-06-05 04:07:20.206 UTC [main] main -> INFO 008 Exiting.....


# 可以看到 返回 Query Result: 10 账户上有10个币




# 查询 B 账户里的余额

peer chaincode query -C mychannel -n mychannel -c '{"Args":["query","B"]}'


# 输出如下:

2018-06-05 04:09:08.238 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:09:08.238 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:09:08.238 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-06-05 04:09:08.238 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-06-05 04:09:08.239 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode disabled
2018-06-05 04:09:08.239 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0AA3070A6B08031A0B08E499D8D80510...6E6E656C1A0A0A0571756572790A0142 
2018-06-05 04:09:08.239 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: 8DEB3D74945B3C274B215FB16B0B5EA468EB0B2BB8121949796EE0A7DD31EAC7 
Query Result: 10
2018-06-05 04:09:08.256 UTC [main] main -> INFO 008 Exiting.....


# 可以看到 返回 Query Result: 10 账户上有10个币

```


```
# invoke 转账方法


# 从A账户 转账 5个币 到 B 账户


peer chaincode invoke -C mychannel -n mychannel -c '{"Args":["invoke", "A", "B", "5"]}'


# 输出如下:


2018-06-05 04:10:23.536 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:10:23.536 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:10:23.536 UTC [msp/identity] Sign -> DEBU 003 Sign: plaintext: 0A94070A5C08011A0C08AF9AD8D80510...426C6F636B0A096D796368616E6E656C 
2018-06-05 04:10:23.536 UTC [msp/identity] Sign -> DEBU 004 Sign: digest: 39756FDF0F4C2116D88E1E929192F7F73A5693EA3AA7CB6D6088E8330F408847 
2018-06-05 04:10:23.543 UTC [common/channelconfig] NewStandardValues -> DEBU 005 Initializing protos for *channelconfig.ChannelProtos
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 006 Processing field: HashingAlgorithm
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 007 Processing field: BlockDataHashingStructure
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 008 Processing field: OrdererAddresses
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 009 Processing field: Consortium
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 00a Processing field: Capabilities
2018-06-05 04:10:23.543 UTC [common/channelconfig] NewStandardValues -> DEBU 00b Initializing protos for *channelconfig.ApplicationProtos
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 00c Processing field: Capabilities
2018-06-05 04:10:23.543 UTC [common/channelconfig] NewStandardValues -> DEBU 00d Initializing protos for *channelconfig.ApplicationOrgProtos
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 00e Processing field: AnchorPeers
2018-06-05 04:10:23.543 UTC [common/channelconfig] NewStandardValues -> DEBU 00f Initializing protos for *channelconfig.OrganizationProtos
2018-06-05 04:10:23.543 UTC [common/channelconfig] initializeProtosStruct -> DEBU 010 Processing field: MSP
2018-06-05 04:10:23.544 UTC [common/channelconfig] Validate -> DEBU 011 Anchor peers for org Org2MSP are 
2018-06-05 04:10:23.544 UTC [common/channelconfig] validateMSP -> DEBU 012 Setting up MSP for org Org2MSP
2018-06-05 04:10:23.544 UTC [msp] newBccspMsp -> DEBU 013 Creating BCCSP-based MSP instance
2018-06-05 04:10:23.544 UTC [msp] New -> DEBU 014 Creating Cache-MSP instance
2018-06-05 04:10:23.544 UTC [msp] Setup -> DEBU 015 Setting up MSP instance Org2MSP
2018-06-05 04:10:23.544 UTC [msp/identity] newIdentity -> DEBU 016 Creating identity instance for cert -----BEGIN CERTIFICATE-----
MIICODCCAd6gAwIBAgIRAJ6ejnxh35HTq6Mw0B7ku6gwCgYIKoZIzj0EAwIwbTEL
MAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNhbiBG
cmFuY2lzY28xFjAUBgNVBAoTDW9yZzIuamlja2kubWUxGTAXBgNVBAMTEGNhLm9y
ZzIuamlja2kubWUwHhcNMTgwNjA1MDIxNTQ0WhcNMjgwNjAyMDIxNTQ0WjBtMQsw
CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
YW5jaXNjbzEWMBQGA1UEChMNb3JnMi5qaWNraS5tZTEZMBcGA1UEAxMQY2Eub3Jn
Mi5qaWNraS5tZTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABHnTg44+BaU7xmqe
mlV/V/MAQQtdYaqB0UngYI/d2af6NVrpvHF3z8QRHfDXwTjPyAyUP7zEatYa2oAx
2P/TWnujXzBdMA4GA1UdDwEB/wQEAwIBpjAPBgNVHSUECDAGBgRVHSUAMA8GA1Ud
EwEB/wQFMAMBAf8wKQYDVR0OBCIEINgMW+fRS/ZyGJAzLmt6UzYeErQAI+NKm47H
Yzg9LQf3MAoGCCqGSM49BAMCA0gAMEUCIQDdSxYp/XOZD1No48Q5jPYHFzCEQ3Oy
ReKZb4wJrtR87QIgI3ePpK56oL16ivVOmDWVINch7v10cd6hzXFrwqWjizo=
-----END CERTIFICATE-----
2018-06-05 04:10:23.544 UTC [msp/identity] newIdentity -> DEBU 017 Creating identity instance for cert -----BEGIN CERTIFICATE-----
MIICEDCCAbegAwIBAgIRAMH1q/Lkt7ZEhmqvkw3WxQ8wCgYIKoZIzj0EAwIwbTEL
MAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNhbiBG
cmFuY2lzY28xFjAUBgNVBAoTDW9yZzIuamlja2kubWUxGTAXBgNVBAMTEGNhLm9y
ZzIuamlja2kubWUwHhcNMTgwNjA1MDIxNTQ0WhcNMjgwNjAyMDIxNTQ0WjBYMQsw
CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
YW5jaXNjbzEcMBoGA1UEAwwTQWRtaW5Ab3JnMi5qaWNraS5tZTBZMBMGByqGSM49
AgEGCCqGSM49AwEHA0IABNAQuJzs9Jp/taEnB+88ld9IKp3lEmFdCXJ/J8K7M5FA
Tyl+LVDsrcuPNZcU++C+l4gj4V/A3lx5W0CFFtPFIoCjTTBLMA4GA1UdDwEB/wQE
AwIHgDAMBgNVHRMBAf8EAjAAMCsGA1UdIwQkMCKAINgMW+fRS/ZyGJAzLmt6UzYe
ErQAI+NKm47HYzg9LQf3MAoGCCqGSM49BAMCA0cAMEQCIDZKd9vydrIuViS21vm4
lhfUP76UPFqh1ghCmwDSfmYuAiAxAK280gRxFFphtdIWpAo9h6EIaE+JJ8FCsytQ
W1gHtg==
-----END CERTIFICATE-----
2018-06-05 04:10:23.545 UTC [msp] Validate -> DEBU 018 MSP Org2MSP validating identity
2018-06-05 04:10:23.545 UTC [common/channelconfig] NewStandardValues -> DEBU 019 Initializing protos for *channelconfig.ApplicationOrgProtos
2018-06-05 04:10:23.545 UTC [common/channelconfig] initializeProtosStruct -> DEBU 01a Processing field: AnchorPeers
2018-06-05 04:10:23.545 UTC [common/channelconfig] NewStandardValues -> DEBU 01b Initializing protos for *channelconfig.OrganizationProtos
2018-06-05 04:10:23.545 UTC [common/channelconfig] initializeProtosStruct -> DEBU 01c Processing field: MSP
2018-06-05 04:10:23.545 UTC [common/channelconfig] Validate -> DEBU 01d Anchor peers for org Org1MSP are 
2018-06-05 04:10:23.545 UTC [common/channelconfig] validateMSP -> DEBU 01e Setting up MSP for org Org1MSP
2018-06-05 04:10:23.545 UTC [msp] newBccspMsp -> DEBU 01f Creating BCCSP-based MSP instance
2018-06-05 04:10:23.545 UTC [msp] New -> DEBU 020 Creating Cache-MSP instance
2018-06-05 04:10:23.545 UTC [msp] Setup -> DEBU 021 Setting up MSP instance Org1MSP
2018-06-05 04:10:23.546 UTC [msp/identity] newIdentity -> DEBU 022 Creating identity instance for cert -----BEGIN CERTIFICATE-----
MIICNzCCAd6gAwIBAgIRALGwrHSDvHRxDs7yh3TjpdEwCgYIKoZIzj0EAwIwbTEL
MAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNhbiBG
cmFuY2lzY28xFjAUBgNVBAoTDW9yZzEuamlja2kubWUxGTAXBgNVBAMTEGNhLm9y
ZzEuamlja2kubWUwHhcNMTgwNjA1MDIxNTQ0WhcNMjgwNjAyMDIxNTQ0WjBtMQsw
CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
YW5jaXNjbzEWMBQGA1UEChMNb3JnMS5qaWNraS5tZTEZMBcGA1UEAxMQY2Eub3Jn
MS5qaWNraS5tZTBZMBMGByqGSM49AgEGCCqGSM49AwEHA0IABGP2JGdlPBOHWTto
u60dAknaCMSTR8a3as2kDWx54sPonlQAV8gly/boL2I0DbnSGehDE9knMLNnq0pg
xifNYzujXzBdMA4GA1UdDwEB/wQEAwIBpjAPBgNVHSUECDAGBgRVHSUAMA8GA1Ud
EwEB/wQFMAMBAf8wKQYDVR0OBCIEIOvtmA4ZdDNYiXxsfpBp28eJqHIJevjRQr3G
nwrGDInVMAoGCCqGSM49BAMCA0cAMEQCIGWD4muVWNrtmBp5dtiqu1JvsYl6GT+b
zL9pvmzWkjzsAiAe4lQ/+REQK65y288HgtTmTbyW6gbuncMe/yRspTMnWQ==
-----END CERTIFICATE-----
2018-06-05 04:10:23.546 UTC [msp/identity] newIdentity -> DEBU 023 Creating identity instance for cert -----BEGIN CERTIFICATE-----
MIICETCCAbegAwIBAgIRAJ/0or+WtZwHIxuHaq8t5fAwCgYIKoZIzj0EAwIwbTEL
MAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNhbiBG
cmFuY2lzY28xFjAUBgNVBAoTDW9yZzEuamlja2kubWUxGTAXBgNVBAMTEGNhLm9y
ZzEuamlja2kubWUwHhcNMTgwNjA1MDIxNTQ0WhcNMjgwNjAyMDIxNTQ0WjBYMQsw
CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
YW5jaXNjbzEcMBoGA1UEAwwTQWRtaW5Ab3JnMS5qaWNraS5tZTBZMBMGByqGSM49
AgEGCCqGSM49AwEHA0IABNABvZUWsP4ZYd6Va26vLVnBloiG3FpjtBpTcjx1G5jo
fi43B18gOMiyI/sz9RSIJ8CtQbraLavPSMJbyNOoOzGjTTBLMA4GA1UdDwEB/wQE
AwIHgDAMBgNVHRMBAf8EAjAAMCsGA1UdIwQkMCKAIOvtmA4ZdDNYiXxsfpBp28eJ
qHIJevjRQr3GnwrGDInVMAoGCCqGSM49BAMCA0gAMEUCIQDT0Mx+EcGPeQ3oFtKL
cy5BX7j1Fjx7C66Alxh3QoSe+AIgWotX5bjVPi/6gbAiJWWVyga/VwxHKG+bsdVC
sz6Q3Fc=
-----END CERTIFICATE-----
2018-06-05 04:10:23.546 UTC [msp] Validate -> DEBU 024 MSP Org1MSP validating identity
2018-06-05 04:10:23.546 UTC [common/channelconfig] NewStandardValues -> DEBU 025 Initializing protos for *channelconfig.OrdererProtos
2018-06-05 04:10:23.546 UTC [common/channelconfig] initializeProtosStruct -> DEBU 026 Processing field: ConsensusType
2018-06-05 04:10:23.547 UTC [common/channelconfig] initializeProtosStruct -> DEBU 027 Processing field: BatchSize
2018-06-05 04:10:23.547 UTC [common/channelconfig] initializeProtosStruct -> DEBU 028 Processing field: BatchTimeout
2018-06-05 04:10:23.547 UTC [common/channelconfig] initializeProtosStruct -> DEBU 029 Processing field: KafkaBrokers
2018-06-05 04:10:23.547 UTC [common/channelconfig] initializeProtosStruct -> DEBU 02a Processing field: ChannelRestrictions
2018-06-05 04:10:23.547 UTC [common/channelconfig] initializeProtosStruct -> DEBU 02b Processing field: Capabilities
2018-06-05 04:10:23.547 UTC [common/channelconfig] NewStandardValues -> DEBU 02c Initializing protos for *channelconfig.OrganizationProtos
2018-06-05 04:10:23.547 UTC [common/channelconfig] initializeProtosStruct -> DEBU 02d Processing field: MSP
2018-06-05 04:10:23.547 UTC [common/channelconfig] validateMSP -> DEBU 02e Setting up MSP for org OrdererOrg
2018-06-05 04:10:23.547 UTC [msp] newBccspMsp -> DEBU 02f Creating BCCSP-based MSP instance
2018-06-05 04:10:23.547 UTC [msp] New -> DEBU 030 Creating Cache-MSP instance
2018-06-05 04:10:23.547 UTC [msp] Setup -> DEBU 031 Setting up MSP instance OrdererMSP
2018-06-05 04:10:23.547 UTC [msp/identity] newIdentity -> DEBU 032 Creating identity instance for cert -----BEGIN CERTIFICATE-----
MIICJDCCAcqgAwIBAgIRAOKDfKO2chUveMRxRhyH9PgwCgYIKoZIzj0EAwIwYzEL
MAkGA1UEBhMCVVMxEzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNhbiBG
cmFuY2lzY28xETAPBgNVBAoTCGppY2tpLm1lMRQwEgYDVQQDEwtjYS5qaWNraS5t
ZTAeFw0xODA2MDUwMjE1NDRaFw0yODA2MDIwMjE1NDRaMGMxCzAJBgNVBAYTAlVT
MRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQHEw1TYW4gRnJhbmNpc2NvMREw
DwYDVQQKEwhqaWNraS5tZTEUMBIGA1UEAxMLY2Euamlja2kubWUwWTATBgcqhkjO
PQIBBggqhkjOPQMBBwNCAAQx8FYRaMnfxEP8XT5kTx9r/Ykgy6W0vMkaZ+/0y79/
NVrbEl6BlQ7QMNQUUs/Y6zNk1f1PVq6jj1k9Np5J9EKRo18wXTAOBgNVHQ8BAf8E
BAMCAaYwDwYDVR0lBAgwBgYEVR0lADAPBgNVHRMBAf8EBTADAQH/MCkGA1UdDgQi
BCAWEky+Uck39P/A6ORyhPtqgXq3gX2lkNbgvjMlTRWKdTAKBggqhkjOPQQDAgNI
ADBFAiEAvt1tibAkeEQoHUSR54gsQGk7WSGfwiAKLSy9w6oniVoCID8xvAl4U02e
orJRHP0JJfJHUQQnr9rpbM/E1FSrXD+v
-----END CERTIFICATE-----
2018-06-05 04:10:23.547 UTC [msp/identity] newIdentity -> DEBU 033 Creating identity instance for cert -----BEGIN CERTIFICATE-----
MIICATCCAaegAwIBAgIQFS0xZIw5gAOc2r737SWNLzAKBggqhkjOPQQDAjBjMQsw
CQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy
YW5jaXNjbzERMA8GA1UEChMIamlja2kubWUxFDASBgNVBAMTC2NhLmppY2tpLm1l
MB4XDTE4MDYwNTAyMTU0NFoXDTI4MDYwMjAyMTU0NFowUzELMAkGA1UEBhMCVVMx
EzARBgNVBAgTCkNhbGlmb3JuaWExFjAUBgNVBAcTDVNhbiBGcmFuY2lzY28xFzAV
BgNVBAMMDkFkbWluQGppY2tpLm1lMFkwEwYHKoZIzj0CAQYIKoZIzj0DAQcDQgAE
qSC6oQtea8jWxOgZ3ybOFB42zpYSYzo8t/C1gz8sRSJVpAso/HGenGZZ3gpHIeIP
fWzUw1Wef2qbRWuXBCt7iqNNMEswDgYDVR0PAQH/BAQDAgeAMAwGA1UdEwEB/wQC
MAAwKwYDVR0jBCQwIoAgFhJMvlHJN/T/wOjkcoT7aoF6t4F9pZDW4L4zJU0VinUw
CgYIKoZIzj0EAwIDSAAwRQIhAJWqhpVLJa+w5H+5PhPVAMLZxZp/ydeTNGjJlSIP
anzyAiBhP1KC50lg2OPfBYwjUgogWoEgZBBFJq4Oo9WVDu2V0Q==
-----END CERTIFICATE-----
2018-06-05 04:10:23.548 UTC [msp] Validate -> DEBU 034 MSP OrdererMSP validating identity
2018-06-05 04:10:23.548 UTC [msp] Setup -> DEBU 035 Setting up the MSP manager (3 msps)
2018-06-05 04:10:23.548 UTC [msp] Setup -> DEBU 036 MSP manager setup complete, setup 3 msps
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 037 Proposed new policy Readers for Channel/Application/Org2MSP
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 038 Proposed new policy Writers for Channel/Application/Org2MSP
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 039 Proposed new policy Admins for Channel/Application/Org2MSP
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 03a Proposed new policy Readers for Channel/Application/Org1MSP
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 03b Proposed new policy Writers for Channel/Application/Org1MSP
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 03c Proposed new policy Admins for Channel/Application/Org1MSP
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 03d Proposed new policy Admins for Channel/Application
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 03e Proposed new policy Readers for Channel/Application
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 03f Proposed new policy Writers for Channel/Application
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 040 Proposed new policy Admins for Channel/Orderer/OrdererOrg
2018-06-05 04:10:23.548 UTC [policies] NewManagerImpl -> DEBU 041 Proposed new policy Readers for Channel/Orderer/OrdererOrg
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 042 Proposed new policy Writers for Channel/Orderer/OrdererOrg
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 043 Proposed new policy Admins for Channel/Orderer
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 044 Proposed new policy Readers for Channel/Orderer
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 045 Proposed new policy Writers for Channel/Orderer
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 046 Proposed new policy BlockValidation for Channel/Orderer
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 047 Proposed new policy Readers for Channel
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 048 Proposed new policy Writers for Channel
2018-06-05 04:10:23.549 UTC [policies] NewManagerImpl -> DEBU 049 Proposed new policy Admins for Channel
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 04a Adding to config map: [Group]  /Channel
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 04b Adding to config map: [Group]  /Channel/Orderer
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 04c Adding to config map: [Group]  /Channel/Orderer/OrdererOrg
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 04d Adding to config map: [Value]  /Channel/Orderer/OrdererOrg/MSP
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 04e Adding to config map: [Policy] /Channel/Orderer/OrdererOrg/Writers
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 04f Adding to config map: [Policy] /Channel/Orderer/OrdererOrg/Admins
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 050 Adding to config map: [Policy] /Channel/Orderer/OrdererOrg/Readers
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 051 Adding to config map: [Value]  /Channel/Orderer/ChannelRestrictions
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 052 Adding to config map: [Value]  /Channel/Orderer/ConsensusType
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 053 Adding to config map: [Value]  /Channel/Orderer/BatchSize
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 054 Adding to config map: [Value]  /Channel/Orderer/BatchTimeout
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 055 Adding to config map: [Policy] /Channel/Orderer/Readers
2018-06-05 04:10:23.549 UTC [common/configtx] addToMap -> DEBU 056 Adding to config map: [Policy] /Channel/Orderer/Writers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 057 Adding to config map: [Policy] /Channel/Orderer/BlockValidation
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 058 Adding to config map: [Policy] /Channel/Orderer/Admins
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 059 Adding to config map: [Group]  /Channel/Application
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 05a Adding to config map: [Group]  /Channel/Application/Org2MSP
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 05b Adding to config map: [Value]  /Channel/Application/Org2MSP/MSP
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 05c Adding to config map: [Policy] /Channel/Application/Org2MSP/Admins
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 05d Adding to config map: [Policy] /Channel/Application/Org2MSP/Readers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 05e Adding to config map: [Policy] /Channel/Application/Org2MSP/Writers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 05f Adding to config map: [Group]  /Channel/Application/Org1MSP
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 060 Adding to config map: [Value]  /Channel/Application/Org1MSP/MSP
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 061 Adding to config map: [Policy] /Channel/Application/Org1MSP/Readers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 062 Adding to config map: [Policy] /Channel/Application/Org1MSP/Writers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 063 Adding to config map: [Policy] /Channel/Application/Org1MSP/Admins
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 064 Adding to config map: [Policy] /Channel/Application/Readers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 065 Adding to config map: [Policy] /Channel/Application/Writers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 066 Adding to config map: [Policy] /Channel/Application/Admins
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 067 Adding to config map: [Value]  /Channel/OrdererAddresses
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 068 Adding to config map: [Value]  /Channel/Consortium
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 069 Adding to config map: [Value]  /Channel/HashingAlgorithm
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 06a Adding to config map: [Value]  /Channel/BlockDataHashingStructure
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 06b Adding to config map: [Policy] /Channel/Writers
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 06c Adding to config map: [Policy] /Channel/Admins
2018-06-05 04:10:23.550 UTC [common/configtx] addToMap -> DEBU 06d Adding to config map: [Policy] /Channel/Readers
2018-06-05 04:10:23.550 UTC [chaincodeCmd] InitCmdFactory -> INFO 06e Get chain(mychannel) orderer endpoint: orderer.jicki.cn:7050
2018-06-05 04:10:23.553 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 06f Using default escc
2018-06-05 04:10:23.553 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 070 Using default vscc
2018-06-05 04:10:23.553 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 071 java chaincode disabled
2018-06-05 04:10:23.553 UTC [msp/identity] Sign -> DEBU 072 Sign: plaintext: 0AA4070A6C08031A0C08AF9AD8D80510...06696E766F6B650A01410A01420A0135 
2018-06-05 04:10:23.553 UTC [msp/identity] Sign -> DEBU 073 Sign: digest: DEB9F3D773E6846F307010CB0B74D12F382948C9A74E99722E24AA0E8B85EA22 
2018-06-05 04:10:23.571 UTC [msp/identity] Sign -> DEBU 074 Sign: plaintext: 0AA4070A6C08031A0C08AF9AD8D80510...45F7B2AC73F72A9F1955EF3DD72ACD37 
2018-06-05 04:10:23.571 UTC [msp/identity] Sign -> DEBU 075 Sign: digest: F695C4F1347EA760A59E3E1D76A796625F6280D0FB82B3596605D16B609D6B3F 
2018-06-05 04:10:23.573 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> DEBU 076 ESCC invoke result: version:1 response:<status:200 message:"OK" > payload:"\n 3\021:I\033\214\205\225\242\\T\177D\030:EL\336v\350\213\204\367:\354M\217\253\000\251}\276\022f\nM\022\031\n\004lscc\022\021\n\017\n\tmychannel\022\002\010\001\0220\n\tmychannel\022#\n\007\n\001A\022\002\010\001\n\007\n\001B\022\002\010\001\032\006\n\001A\032\0015\032\007\n\001B\032\00215\032\003\010\310\001\"\020\022\tmychannel\032\0031.0" endorsement:<endorser:"\n\007Org1MSP\022\212\006-----BEGIN CERTIFICATE-----\nMIICEDCCAbagAwIBAgIQONKn7NxyKRszERjAoW/iJzAKBggqhkjOPQQDAjBtMQsw\nCQYDVQQGEwJVUzETMBEGA1UECBMKQ2FsaWZvcm5pYTEWMBQGA1UEBxMNU2FuIEZy\nYW5jaXNjbzEWMBQGA1UEChMNb3JnMS5qaWNraS5tZTEZMBcGA1UEAxMQY2Eub3Jn\nMS5qaWNraS5tZTAeFw0xODA2MDUwMjE1NDRaFw0yODA2MDIwMjE1NDRaMFgxCzAJ\nBgNVBAYTAlVTMRMwEQYDVQQIEwpDYWxpZm9ybmlhMRYwFAYDVQQHEw1TYW4gRnJh\nbmNpc2NvMRwwGgYDVQQDExNwZWVyMC5vcmcxLmppY2tpLm1lMFkwEwYHKoZIzj0C\nAQYIKoZIzj0DAQcDQgAExPvxGKgil1iaJjmDjnqsVgQVooziKPN4zn+IyczDEukJ\nEQlsD4JLbKHV+qfHFoGWOBMLnXSccHI8kMLgXg4cSqNNMEswDgYDVR0PAQH/BAQD\nAgeAMAwGA1UdEwEB/wQCMAAwKwYDVR0jBCQwIoAg6+2YDhl0M1iJfGx+kGnbx4mo\ncgl6+NFCvcafCsYMidUwCgYIKoZIzj0EAwIDSAAwRQIhAOSZmNgcSaGTo0t2Amvh\nMvsrghoSS3KT/6W8PVhPqf/9AiBBw5RBC6lelO7sxpjPjxkQgQfZAyJ2SOnh3HHb\n+22yqA==\n-----END CERTIFICATE-----\n" signature:"0D\002 z\017V\0213\0100\265\251J\303\254s\375o\016\223D\032\225;\037@\r\212'Vxq\217\361l\002 w\034\320I\036\335\260P\210U\372\020\346Pq\022E\367\262\254s\367*\237\031U\357=\327*\3157" > 
2018-06-05 04:10:23.573 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 077 Chaincode invoke successful. result: status:200 
2018-06-05 04:10:23.573 UTC [main] main -> INFO 078 Exiting.....



# 可以看到返回 invoke successful. result: status:200 成功


# 这里再查询 A 与 B 的账户


# A 账户余额 Query Result: 5

peer chaincode query -C mychannel -n mychannel -c '{"Args":["query","A"]}'  


# 输出如下:

2018-06-05 04:11:24.256 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:11:24.256 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:11:24.256 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-06-05 04:11:24.256 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-06-05 04:11:24.256 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode disabled
2018-06-05 04:11:24.257 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0AA3070A6B08031A0B08EC9AD8D80510...6E6E656C1A0A0A0571756572790A0141 
2018-06-05 04:11:24.257 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: 37E28D7C8C673D454A7BFE2483C896D6DAEB685BC873D413846FBAA656D1D8E2 
Query Result: 5
2018-06-05 04:11:24.273 UTC [main] main -> INFO 008 Exiting.....




# B 账户余额 Query Result: 15

peer chaincode query -C mychannel -n mychannel -c '{"Args":["query","B"]}'   



# 输出如下:

2018-06-05 04:11:46.861 UTC [msp] GetLocalMSP -> DEBU 001 Returning existing local MSP
2018-06-05 04:11:46.861 UTC [msp] GetDefaultSigningIdentity -> DEBU 002 Obtaining default signing identity
2018-06-05 04:11:46.861 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 003 Using default escc
2018-06-05 04:11:46.863 UTC [chaincodeCmd] checkChaincodeCmdParams -> INFO 004 Using default vscc
2018-06-05 04:11:46.863 UTC [chaincodeCmd] getChaincodeSpec -> DEBU 005 java chaincode disabled
2018-06-05 04:11:46.864 UTC [msp/identity] Sign -> DEBU 006 Sign: plaintext: 0AA4070A6C08031A0C08829BD8D80510...6E6E656C1A0A0A0571756572790A0142 
2018-06-05 04:11:46.864 UTC [msp/identity] Sign -> DEBU 007 Sign: digest: E3255F6FDA1F24F6BC6DD7C42075748CC27AC41900EECB07F884E2B608A8B709 
Query Result: 15
2018-06-05 04:11:46.878 UTC [main] main -> INFO 008 Exiting.....

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

peer chaincode install -n mychannel -p github.com/hyperledger/fabric/jicki/chaincode/go/chaincode_example02 -v 1.1


# 更新版本为 1.1 的合约
peer chaincode upgrade -o orderer.jicki.cn:7050 -C mychannel -n mychannel -c '{"Args":["init","A","10","B","10"]}' -P "OR ('Org1MSP.member')" -v 1.1


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


