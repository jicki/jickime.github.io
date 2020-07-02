# hyperledger-fabric v2.0


> fabric v2.0 , 单机 多节点 RAFT 手动部署, 所有服务均 开启 SSL 认证。


# hyperledger fabric



## Hyperleger fabric 是什么?

* Linux基金会于2015年创建了Hyperledger项目，以推进跨行业的区块链技术。Hyperledger Fabric是Hyperledger中的区块链项目之一。像其他区块链技术一样，它具有分类帐，使用智能合约，并且是参与者用来管理其交易的系统。Hyperledger Fabric与其他区块链系统最大的不同体现在私有和许可。

![图1][1]


* Hyperledger Fabric 是一个（平台），提供分布式账本解决方案的平台，Hyperledger Fabric 由模块化架构支撑，并具备极佳的`保密性`、`可伸缩性`、`灵活性`、`可扩展性`，Hyperledger Fabric 被设计成（模块直接插拔，适用多种场景），支持不同的模块组件直接拔插启用，并能适应在经济生态系统中错综复杂的各种场景。




## Hyperledger Fabric 交易流程

![图2][2]






## 部署 Hyperledger Fabric v2.0

> 官网 github ---  https://github.com/hyperledger/fabric



### 准备阶段

* docker-ce

* docker-compose

* Hyperledger Fabric images



### 安装 docker


* 升级内核

```shell

# 导入 Key

rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org



# 安装 Yum 源

rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm



# 更新 kernel

yum --enablerepo=elrepo-kernel install -y kernel-lt kernel-lt-devel


# 配置 内核优先

grub2-set-default 0

```


* 开启内核 namespace 支持

```
# 执行如下
grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"



# 必须重启系统

reboot

```




* 修改内核参数

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


* 检查环境状态

```shell
curl -s https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh | bash

```



* 安装 docker

```
# 指定安装,并指定安装源

export VERSION=19.03
curl -fsSL "https://get.docker.com/" | bash -s -- --mirror Aliyun

```


* docker 额外配置

```shell
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


* 启动 docker


```shell
# 配置开机启动
systemctl enable docker


# 启动 docker
systemctl start docker

```


### 安装 docker-compose


```
# 国内下载点
curl -L https://get.daocloud.io/docker/compose/releases/download/1.25.4/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose



# github 下载
curl -L "https://github.com/docker/compose/releases/download/1.25.4/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

chmod +x /usr/local/bin/docker-compose
```



### Hyperledger Fabric images

* 由于 image 镜像较大,下载过程会比较久, 所以这里提前将 images 下载到本地。


```shell
#!/bin/bash

FABRIC_TAG=2.0

function pullImages() {
  docker pull hyperledger/fabric-couchdb:0.4.20
  docker pull hyperledger/fabric-ca:1.4.6
  for image in peer orderer ccenv tools; do
    echo "Pull image hyperledger/fabric-$image:$FABRIC_TAG"
    docker pull hyperledger/fabric-"$image":"$FABRIC_TAG"
    sleep 1
  done
}

echo "Pull images for hyperledger fabric network."

pullImages

```

```shell
[root@localhost jicki]# docker images
REPOSITORY                   TAG                 IMAGE ID            CREATED             SIZE
hyperledger/fabric-tools     2.0                 5c9a03790913        3 weeks ago         512MB
hyperledger/fabric-peer      2.0                 5c7e5946f3dc        3 weeks ago         57.2MB
hyperledger/fabric-orderer   2.0                 92bd220edcdd        3 weeks ago         39.7MB
hyperledger/fabric-ccenv     2.0                 800087268d9b        3 weeks ago         529MB
hyperledger/fabric-ca        1.4.6               3b96a893c1e4        3 weeks ago         150MB

```




### 创建 crypto-config.yaml 文件


* Fabric 网络成员加密配置

* 如下配置 3个 orderer 节点, 2个Org组织, 每个组织下有 2个Peer节点。

```
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


### 创建 configtxgen.yaml 文件

* 网络初始化以及通道配置



```shell

---
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

        OrdererEndpoints:
            - orderer0.jicki.cn:7050
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
            Endorsement:
                Type: Signature
                Rule: "OR('Org1MSP.peer')"

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
            Endorsement:
                Type: Signature
                Rule: "OR('Org2MSP.peer')"

        AnchorPeers:
            - Host: peer0.org2.jicki.cn
              Port: 7051
Capabilities:
    Channel: &ChannelCapabilities
        V2_0: true

    Orderer: &OrdererCapabilities
        V2_0: true

    Application: &ApplicationCapabilities
        V2_0: true

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
        LifecycleEndorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"
        Endorsement:
            Type: ImplicitMeta
            Rule: "MAJORITY Endorsement"

    Capabilities:
        <<: *ApplicationCapabilities

Orderer: &OrdererDefaults
    OrdererType: etcdraft
    BatchTimeout: 2s
    BatchSize:
        MaxMessageCount: 10
        AbsoluteMaxBytes: 99 MB
        PreferredMaxBytes: 512 KB
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
    TwoOrgsChannel:
        Consortium: SampleConsortium
        <<: *ChannelDefaults
        Application:
            <<: *ApplicationDefaults
            Organizations:
                - *Org1
                - *Org2
            Capabilities:
                <<: *ApplicationCapabilities

    SampleMultiNodeEtcdRaft:
        <<: *ChannelDefaults
        Capabilities:
            <<: *ChannelCapabilities
        Orderer:
            <<: *OrdererDefaults
            OrdererType: etcdraft
            EtcdRaft:
                Consenters:
                - Host: orderer0.jicki.cn
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/tls/server.crt
                - Host: orderer1.jicki.cn
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/jicki.cn/orderers/orderer1.jicki.cn/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/jicki.cn/orderers/orderer1.jicki.cn/tls/server.crt
                - Host: orderer2.jicki.cn
                  Port: 7050
                  ClientTLSCert: crypto-config/ordererOrganizations/jicki.cn/orderers/orderer2.jicki.cn/tls/server.crt
                  ServerTLSCert: crypto-config/ordererOrganizations/jicki.cn/orderers/orderer2.jicki.cn/tls/server.crt
            Addresses:
                - orderer0.jicki.cn:7050
                - orderer1.jicki.cn:7050
                - orderer2.jicki.cn:7050

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



### 生成证书和创世区块


```shell
# 下载证书生成工具

cd /opt/jicki

wget https://github.com/hyperledger/fabric/releases/download/v2.0.0/hyperledger-fabric-linux-amd64-2.0.0.tar.gz

```

```shell
[root@localhost jicki]# tar zxvf hyperledger-fabric-linux-amd64-2.0.0.tar.gz 
bin/
bin/configtxgen
bin/orderer
bin/peer
bin/discover
bin/idemixgen
bin/configtxlator
bin/cryptogen
config/
config/configtx.yaml
config/core.yaml
config/orderer.yaml
```


```shell
# 为方便使用 我们配置一个 环境变量

vi /etc/profile


# fabric env
export PATH=$PATH:/opt/jicki/bin


# 使文件生效
source /etc/profile
```


```shell
# 首先需要创建一个文件夹
mkdir -p /opt/jicki/channel-artifacts

```


* 生成证书

```shell
# 然后这里使用 cryptogen 软件来生成相应的证书了

[root@localhost jicki]# cryptogen generate --config=./crypto-config.yaml
org1.jicki.cn
org2.jicki.cn

```

* 生成区块

> `-profile` 在 configtx.yaml 文件中查找 Profiles: 下面字段

```shell

# generate genesis block for orderer
# 生成启动 orderer 所需要的创世区块文件
# 这里特别注意 orderer 的channelID 与 下面创建的 channelID 千万不能相同。


configtxgen -profile SampleMultiNodeEtcdRaft \
 -outputBlock ./channel-artifacts/genesis.block \
 -channelID orderer-channel

```

```shell
# 输出如下:
2020-03-23 19:01:00.548 CST [common.tools.configtxgen] main -> INFO 001 Loading configuration
2020-03-23 19:01:00.576 CST [common.tools.configtxgen.localconfig] completeInitialization -> INFO 002 orderer type: etcdraft
2020-03-23 19:01:00.576 CST [common.tools.configtxgen.localconfig] completeInitialization -> INFO 003 Orderer.EtcdRaft.Options unset, setting to tick_interval:"500ms" election_tick:10 heartbeat_tick:1 max_inflight_blocks:5 snapshot_interval_size:16777216 
2020-03-23 19:01:00.576 CST [common.tools.configtxgen.localconfig] Load -> INFO 004 Loaded configuration: /opt/jicki/configtx.yaml
2020-03-23 19:01:00.578 CST [common.tools.configtxgen] doOutputBlock -> INFO 005 Generating genesis block
2020-03-23 19:01:00.578 CST [common.tools.configtxgen] doOutputBlock -> INFO 006 Writing genesis block
```



```shell
# generate channel configuration transaction
# 生成用于配置自定义 channel 的交易文件
# 这里特别注意 configuration 的channelID 不能与上面 Orderer的channelID相同.

configtxgen \
    -profile TwoOrgsChannel \
    -outputCreateChannelTx ./channel-artifacts/channel.tx \
    -channelID mychannel

```

```shell
# 输出如下:

2020-03-23 19:02:33.902 CST [common.tools.configtxgen] main -> INFO 001 Loading configuration
2020-03-23 19:02:33.929 CST [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /opt/jicki/configtx.yaml
2020-03-23 19:02:33.929 CST [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 003 Generating new channel configtx
2020-03-23 19:02:33.932 CST [common.tools.configtxgen] doOutputChannelCreateTx -> INFO 004 Writing new channel tx

```




```shell
# 生成用于配置 anchor peer 的交易文件


# Org1

configtxgen \
    -profile TwoOrgsChannel \
    -outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx \
    -channelID mychannel \
    -asOrg Org1MSP



# Org2

configtxgen \
  -profile TwoOrgsChannel \
  -outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx \
  -channelID mychannel \
  -asOrg Org2MSP


```




```shell
# 输出如下:

2020-03-23 19:04:41.859 CST [common.tools.configtxgen] main -> INFO 001 Loading configuration
2020-03-23 19:04:41.888 CST [common.tools.configtxgen.localconfig] Load -> INFO 002 Loaded configuration: /opt/jicki/configtx.yaml
2020-03-23 19:04:41.889 CST [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 003 Generating anchor peer update
2020-03-23 19:04:41.890 CST [common.tools.configtxgen] doOutputAnchorPeersUpdate -> INFO 004 Writing anchor peer update

```




### 启动 fabric


* docker-compose.yaml 文件

```shell
version: '2'
services:
  orderer0.jicki.cn:
    container_name: orderer0.jicki.cn
    image: hyperledger/fabric-orderer:2.0
    environment:
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_LISTENPORT=7050
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt, /etc/hyperledger/crypto/peerOrg1/tls/ca.crt, /etc/hyperledger/crypto/peer1rg1/tls/ca.crt, /etc/hyperledger/crypto/peerOrg2/tls/ca.crt, /etc/hyperledger/crypto/peer1rg2/tls/ca.crt]
      - ORDERER_GENERAL_CLUSTER_CLIENTCERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_CLUSTER_CLIENTPRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_CLUSTER_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    # 数据持久化,以及存储
    - ./data/orderer0:/var/hyperledger/production
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/tls/:/var/hyperledger/orderer/tls
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/:/etc/hyperledger/crypto/peerOrg1
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer1.org1.jicki.cn/:/etc/hyperledger/crypto/peer1rg1
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/:/etc/hyperledger/crypto/peerOrg2
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer1.org2.jicki.cn/:/etc/hyperledger/crypto/peer1rg2
    networks:
      default:
        aliases:
          - jicki
    ports:
      - 7050:7050
  orderer1.jicki.cn:
    container_name: orderer1.jicki.cn
    image: hyperledger/fabric-orderer:2.0
    environment:
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_LISTENPORT=7050
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt, /etc/hyperledger/crypto/peerOrg1/tls/ca.crt, /etc/hyperledger/crypto/peer1rg1/tls/ca.crt, /etc/hyperledger/crypto/peerOrg2/tls/ca.crt, /etc/hyperledger/crypto/peer1rg2/tls/ca.crt]
      - ORDERER_GENERAL_CLUSTER_CLIENTCERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_CLUSTER_CLIENTPRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_CLUSTER_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    # 数据持久化,以及存储
    - ./data/orderer1:/var/hyperledger/production
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer1.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer1.jicki.cn/tls/:/var/hyperledger/orderer/tls
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/:/etc/hyperledger/crypto/peerOrg1
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer1.org1.jicki.cn/:/etc/hyperledger/crypto/peer1rg1
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/:/etc/hyperledger/crypto/peerOrg2
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer1.org2.jicki.cn/:/etc/hyperledger/crypto/peer1rg2
    networks:
      default:
        aliases:
          - jicki
    ports:
      - 8050:7050
  orderer2.jicki.cn:
    container_name: orderer2.jicki.cn
    image: hyperledger/fabric-orderer:2.0
    environment:
      - CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=jicki_default
      - ORDERER_GENERAL_LOGLEVEL=debug
      - ORDERER_GENERAL_LISTENADDRESS=0.0.0.0
      - ORDERER_GENERAL_LISTENPORT=7050
      - ORDERER_GENERAL_GENESISMETHOD=file
      - ORDERER_GENERAL_GENESISFILE=/var/hyperledger/orderer/orderer.genesis.block
      - ORDERER_GENERAL_LOCALMSPID=OrdererMSP
      - ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/orderer/msp
      # enabled TLS
      - ORDERER_GENERAL_TLS_ENABLED=true
      - ORDERER_GENERAL_TLS_PRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_TLS_CERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_TLS_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt, /etc/hyperledger/crypto/peerOrg1/tls/ca.crt, /etc/hyperledger/crypto/peer1rg1/tls/ca.crt, /etc/hyperledger/crypto/peerOrg2/tls/ca.crt, /etc/hyperledger/crypto/peer1rg2/tls/ca.crt]
      - ORDERER_GENERAL_CLUSTER_CLIENTCERTIFICATE=/var/hyperledger/orderer/tls/server.crt
      - ORDERER_GENERAL_CLUSTER_CLIENTPRIVATEKEY=/var/hyperledger/orderer/tls/server.key
      - ORDERER_GENERAL_CLUSTER_ROOTCAS=[/var/hyperledger/orderer/tls/ca.crt]
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric
    command: orderer
    volumes:
    # 数据持久化,以及存储
    - ./data/orderer2:/var/hyperledger/production
    - ./channel-artifacts/genesis.block:/var/hyperledger/orderer/orderer.genesis.block
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer2.jicki.cn/msp:/var/hyperledger/orderer/msp
    - ./crypto-config/ordererOrganizations/jicki.cn/orderers/orderer2.jicki.cn/tls/:/var/hyperledger/orderer/tls
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/:/etc/hyperledger/crypto/peerOrg1
    - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer1.org1.jicki.cn/:/etc/hyperledger/crypto/peer1rg1
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/:/etc/hyperledger/crypto/peerOrg2
    - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer1.org2.jicki.cn/:/etc/hyperledger/crypto/peer1rg2
    networks:
      default:
        aliases:
          - jicki
    ports:
      - 9050:7050

  couchdb0:
    container_name: couchdb0
    image: hyperledger/fabric-couchdb:0.4.20
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
    image: hyperledger/fabric-couchdb:0.4.20
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    volumes:
      # 数据持久化，用于存储链码值
      - ./data/couchdb1/data:/opt/couchdb/data
    networks:
      default:
        aliases:
          - jicki

  couchdb2:
    container_name: couchdb2
    image: hyperledger/fabric-couchdb:0.4.20
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    volumes:
      # 数据持久化，用于存储链码值
      - ./data/couchdb2/data:/opt/couchdb/data
    networks:
      default:
        aliases:
          - jicki

  couchdb3:
    container_name: couchdb3
    image: hyperledger/fabric-couchdb:0.4.20
    environment:
      - COUCHDB_USER=
      - COUCHDB_PASSWORD=
    volumes:
      # 数据持久化，用于存储链码值
      - ./data/couchdb3/data:/opt/couchdb/data
    networks:
      default:
        aliases:
          - jicki

  peer0.org1.jicki.cn:
    container_name: peer0.org1.jicki.cn
    image: hyperledger/fabric-peer:2.0
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
      - CORE_CHAINCODE_EXECUTETIMEOUT=300s
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

  peer1.org1.jicki.cn:
    container_name: peer1.org1.jicki.cn
    image: hyperledger/fabric-peer:2.0
    environment:
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb1:5984
      
      - CORE_PEER_ID=peer1.org1.jicki.cn
      - CORE_PEER_NETWORKID=jicki
      - CORE_PEER_ADDRESS=peer1.org1.jicki.cn:7051
      - CORE_PEER_CHAINCODELISTENADDRESS=peer1.org1.jicki.cn:7052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org1.jicki.cn:7051
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
      - CORE_CHAINCODE_EXECUTETIMEOUT=300s
      - GODEBUG=netdns=go
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer1.org1.jicki.cn/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org1.jicki.cn/peers/peer1.org1.jicki.cn/tls:/etc/hyperledger/fabric/tls
        # 数据持久化, 存储安装，以及实例化智能合约的数据
        - ./data/peer1org1:/var/hyperledger/production
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


  peer0.org2.jicki.cn:
    container_name: peer0.org2.jicki.cn
    image: hyperledger/fabric-peer:2.0
    environment:
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb2:5984

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
      - CORE_CHAINCODE_EXECUTETIMEOUT=300s
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
      - 9051:7051
      - 9052:7052
      - 9053:7053
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - couchdb2

  peer1.org2.jicki.cn:
    container_name: peer1.org2.jicki.cn
    image: hyperledger/fabric-peer:2.0
    environment:
      - CORE_LEDGER_STATE_STATEDATABASE=CouchDB
      - CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS=couchdb3:5984

      - CORE_PEER_ID=peer1.org2.jicki.cn
      - CORE_PEER_NETWORKID=jicki
      - CORE_PEER_ADDRESS=peer1.org2.jicki.cn:7051
      - CORE_PEER_CHAINCODELISTENADDRESS=peer1.org2.jicki.cn:7052
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer1.org2.jicki.cn:7051
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
      - CORE_CHAINCODE_EXECUTETIMEOUT=300s
      - GODEBUG=netdns=go
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer1.org2.jicki.cn/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org2.jicki.cn/peers/peer1.org2.jicki.cn/tls:/etc/hyperledger/fabric/tls
        # 数据持久化, 存储安装，以及实例化智能合约的数据
        - ./data/peer1org2:/var/hyperledger/production
    working_dir: /opt/gopath/src/github.com/hyperledger/fabric/peer
    command: peer node start
    ports:
      - 10051:7051
      - 10052:7052
      - 10053:7053
    networks:
      default:
        aliases:
          - jicki
    depends_on:
      - couchdb3
  org1.cli:
    container_name: org1.cli
    image: hyperledger/fabric-tools:2.0
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - GODEBUG=netdns=go
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # - CORE_LOGGING_LEVEL=ERROR
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=org1.cli
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

  org2.cli:
    container_name: org2.cli
    image: hyperledger/fabric-tools:2.0
    tty: true
    environment:
      - GOPATH=/opt/gopath
      - GODEBUG=netdns=go
      - CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock
      # - CORE_LOGGING_LEVEL=ERROR
      - CORE_LOGGING_LEVEL=DEBUG
      - CORE_PEER_ID=org2.cli
      - CORE_PEER_ADDRESS=peer0.org2.jicki.cn:7051
      - CORE_PEER_LOCALMSPID=Org2MSP
      - CORE_PEER_TLS_ENABLED=true
      - CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/server.crt
      - CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/server.key
      - CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt
      - CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp
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
```




### 创建 Channel


```shell
#!/bin/bash

CLI_NAME=cli

ORDERER=orderer0.jicki.cn:7050

CORE_PEER_TLS_ENABLED=true

CAFILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/ordererOrganizations/jicki.cn/orderers/orderer0.jicki.cn/msp/tlscacerts/tlsca.jicki.cn-cert.pem

# Channel 通道ID 不能与 之前 Orderer 之前创建的通道相同,否则报错
# config update for existing channel did not pass initial checks: implicit policy evaluation failed - 0 sub-policies were satisfied, but this policy requires 1 of the 'Writers' sub-policies to be satisfied: permission denied

CHANNEL=mychannel

docker exec -it -e "ORDERER=$ORDERER" -e "CORE_PEER_TLS_ENABLED=$CORE_PEER_TLS_ENABLED" -e "CAFILE=$CAFILE" -e "CHANNEL=$CHANNEL" "$(docker ps -q -f status=running --filter name=$CLI_NAME)" sh -c 'peer channel create -o $ORDERER -c $CHANNEL -f ./channel-artifacts/channel.tx --tls $CORE_PEER_TLS_ENABLED --cafile $CAFILE'

```



### 加入 Channel (Join)


* 每个 peer 节点都需要单独加入到 Channel 中，这里有4个peer节点。


* 首先是 org1 组织

```shell
# org1 下的 peer0 节点

[root@localhost jicki]# docker exec -it org1.cli bash


export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/users/Admin@org1.jicki.cn/msp
export CORE_PEER_ADDRESS=peer0.org1.jicki.cn:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/ca.crt


peer channel join \
    -b mychannel.block \
    -o orderer0.jicki.cn:7050 \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA

```


```shell
# org1 下的 peer1 节点

export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/users/Admin@org1.jicki.cn/msp
export CORE_PEER_ADDRESS=peer1.org1.jicki.cn:7051
export CORE_PEER_LOCALMSPID="Org1MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer1.org1.jicki.cn/tls/ca.crt



peer channel join \
    -b mychannel.block \
    -o orderer0.jicki.cn:7050 \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA

```


* 接下来是 org2 组织

```shell
# org2 组织下的 peer0 节点


export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp
export CORE_PEER_ADDRESS=peer0.org2.jicki.cn:7051
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt


peer channel join \
    -b mychannel.block \
    -o orderer0.jicki.cn:7050 \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA

```


```shell
# org2 组织下的 peer1 节点


export CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/users/Admin@org2.jicki.cn/msp
export CORE_PEER_ADDRESS=peer1.org2.jicki.cn:7051
export CORE_PEER_LOCALMSPID="Org2MSP"
export CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer1.org2.jicki.cn/tls/ca.crt


peer channel join \
    -b mychannel.block \
    -o orderer0.jicki.cn:7050 \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA

```


* 输出日志

```
# 加入成功都会输出如下信息

2020-03-24 04:14:54.947 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2020-03-24 04:14:54.950 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2020-03-24 04:14:54.954 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2020-03-24 04:14:55.092 UTC [channelCmd] executeJoin -> INFO 004 Successfully submitted proposal to join channel

```


### 更新组织的 anchor peer



* org1 组织



```shell

docker exec -it org1.cli bash


peer channel update \
    -o orderer0.jicki.cn:7050 \
    -c mychannel \
    -f ./channel-artifacts/Org1MSPanchors.tx \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA


```




* org2 组织


```shell

docker exec -it org2.cli bash


peer channel update \
    -o orderer0.jicki.cn:7050 \
    -c mychannel \
    -f ./channel-artifacts/Org2MSPanchors.tx \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA

```

* 输出如下信息


```shell
2020-03-24 04:19:46.008 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2020-03-24 04:19:46.011 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2020-03-24 04:19:46.011 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2020-03-24 04:19:46.027 UTC [channelCmd] update -> INFO 004 Successfully submitted channel update
```



### 安装 ChainCode


```shell
# git clone 下载一个 fabric-samples


git clone https://github.com/hyperledger/fabric-samples

Cloning into 'fabric-samples'...
remote: Enumerating objects: 4891, done.
remote: Total 4891 (delta 0), reused 0 (delta 0), pack-reused 4891
Receiving objects: 100% (4891/4891), 1.72 MiB | 369.00 KiB/s, done.
Resolving deltas: 100% (2478/2478), done.

```


* 拷贝 chaincode 到 cli 的目录下

```shell
# 创建一个目录
mkdir -p ./data/cli/chaincode/go/mycc


# 拷贝到目录下
cp fabric-samples/chaincode/abstore/go/* ./data/cli/chaincode/go/mycc/ 

```


* 打包 chaincode

```shell
# 打包 chaincode

docker exec -it org1.cli bash

peer lifecycle chaincode package mycc.tar.gz \
    --path /opt/gopath/src/github.com/hyperledger/fabric/jicki/chaincode/go/mycc \
    --lang golang \
    --label mycc



# 打包完以后会在当前目录生成 mycc.tar.gz 文件

```

* 安装 chaincode 每个组织在任意peer节点安装一次既可。

* 在 peer0.org1 上安装 chaincode

```shell
docker exec -it org1.cli bash


# 安装
peer lifecycle chaincode install mycc.tar.gz


# 查看安装情况
peer lifecycle chaincode queryinstalled

```


* 在 peer0.org2 上安装 chaincode


```shell
docker exec -it org2.cli bash


# 安装
peer lifecycle chaincode install mycc.tar.gz


# 查看安装情况
peer lifecycle chaincode queryinstalled

```


### chaincode 投票

* 每次安装新的 chaincode  组织都需要对其进行投票。


* org1 投票

  * `peer lifecycle chaincode queryinstalled` 查看 Package ID: 部分为 `CC_PACKAGE_ID`

```shell
docker exec -it org1.cli bash


# 获取 CC_PACKAGE_ID 
bash-5.0# peer lifecycle chaincode queryinstalled
Installed chaincodes on peer:
Package ID: mycc:14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be, Label: mycc


# 设置变量
export CC_PACKAGE_ID=mycc:14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be


# 进行投票

peer lifecycle chaincode approveformyorg \
    --channelID mychannel \
    --name mycc \
    --version 1 \
    --init-required \
    --package-id $CC_PACKAGE_ID \
    --sequence 1 \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA

```

```shell
# 输出如下:

2020-03-24 06:24:19.615 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2020-03-24 06:24:19.619 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2020-03-24 06:24:19.634 UTC [cli.lifecycle.chaincode] setOrdererClient -> INFO 003 Retrieved channel (mychannel) orderer endpoint: orderer0.jicki.cn:7050
2020-03-24 06:24:21.880 UTC [chaincodeCmd] ClientWait -> INFO 004 txid [fb01bb2497d00ea1b4e63ef1f9afad125f892dd84717c0712e649716f3092493] committed with status (VALID) at 


```

```shell
# 查看投票情况

peer lifecycle chaincode checkcommitreadiness \
  --channelID mychannel \
  --name mycc \
  --version 1 \
  --init-required \
  --sequence 1 \
  --tls $CORE_PEER_TLS_ENABLED \
  --cafile $ORDERER_CA \
  --output json
```




* org2 投票


```shell
docker exec -it org2.cli bash


# 获取 CC_PACKAGE_ID
bash-5.0# peer lifecycle chaincode queryinstalled
Installed chaincodes on peer:
Package ID: mycc:14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be, Label: mycc


# 设置变量
export CC_PACKAGE_ID=mycc:14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be


# 进行投票

peer lifecycle chaincode approveformyorg \
    --channelID mychannel \
    --name mycc \
    --version 1 \
    --init-required \
    --package-id $CC_PACKAGE_ID \
    --sequence 1 \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA

```

```shell
# 输出如下:

2020-03-24 06:26:25.523 UTC [cli.lifecycle.chaincode] setOrdererClient -> INFO 003 Retrieved channel (mychannel) orderer endpoint: orderer0.jicki.cn:7050
2020-03-24 06:26:27.675 UTC [chaincodeCmd] ClientWait -> INFO 004 txid [067cce8f1c3ca9fa8f9239ad78542e30fee5b83d68ae054a8e9b76a01393bb08] committed with status (VALID) at
```


```shell
# 查看投票情况

peer lifecycle chaincode checkcommitreadiness \
  --channelID mychannel \
  --name mycc \
  --version 1 \
  --init-required \
  --sequence 1 \
  --tls $CORE_PEER_TLS_ENABLED \
  --cafile $ORDERER_CA \
  --output json
```

```shell
# 投票情况

{
        "approvals": {
                "Org1MSP": true,
                "Org2MSP": true
        }
}

```



### 提交确认 ( submit )


```shell
docker exec -it org1.cli bash


# 提交 commit
peer lifecycle chaincode commit \
    -o orderer0.jicki.cn:7050 \
    --channelID mychannel \
    --name mycc \
    --version 1 \
    --sequence 1 \
    --init-required \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA \
    --peerAddresses peer0.org1.jicki.cn:7051 \
    --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/ca.crt \
    --peerAddresses peer0.org2.jicki.cn:7051 \
    --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt

```

* 输出如下信息

```shell

2020-03-24 07:14:12.187 UTC [chaincodeCmd] ClientWait -> INFO 003 txid [131d247fb9035ccb832b15ab34b64fd03fa74d1d5db762388a837b31c4c29e93] committed with status (VALID) at peer0.org1.jicki.cn:7051
2020-03-24 07:14:12.199 UTC [chaincodeCmd] ClientWait -> INFO 004 txid [131d247fb9035ccb832b15ab34b64fd03fa74d1d5db762388a837b31c4c29e93] committed with status (VALID) at peer0.org2.jicki.cn:7051

```


* 分别检查是否成功提交 (commit)


* org1

```shell
# org1

docker exec -it org1.cli bash


peer lifecycle chaincode querycommitted \
    --channelID mychannel \
    --name mycc

```

```shell
# 输出如下:

Committed chaincode definition for chaincode 'mycc' on channel 'mychannel':
Version: 1, Sequence: 1, Endorsement Plugin: escc, Validation Plugin: vscc, Approvals: [Org1MSP: true, Org2MSP: true]

```

* org2

```shell
# org2

docker exec -it org2.cli bash


peer lifecycle chaincode querycommitted \
    --channelID mychannel \
    --name mycc

```

```shell
# 输出如下:
Committed chaincode definition for chaincode 'mycc' on channel 'mychannel':
Version: 1, Sequence: 1, Endorsement Plugin: escc, Validation Plugin: vscc, Approvals: [Org1MSP: true, Org2MSP: true]

```



### 操作 chaincode


* Invoke 操作


```shell
docker exec -it org1.cli bash


# Invoke 操作 ( 初始化 a, b 的值为100 )

peer chaincode invoke \
    -o orderer0.jicki.cn:7050 \
    --tls $CORE_PEER_TLS_ENABLED \
    --cafile $ORDERER_CA \
    -C mychannel \
    -n mycc \
    --peerAddresses peer0.org1.jicki.cn:7051 \
    --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.jicki.cn/peers/peer0.org1.jicki.cn/tls/ca.crt \
    --peerAddresses peer0.org2.jicki.cn:7051 \
    --tlsRootCertFiles /opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.jicki.cn/peers/peer0.org2.jicki.cn/tls/ca.crt \
    --isInit \
    -c '{"Args":["Init","a","100","b","100"]}'

```

```shell
# 输出:
2020-03-24 07:20:59.683 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 003 Chaincode invoke successful. result: status:200 
```



* Query 操作


```shell

peer chaincode query \
    -C mychannel \
    -n mycc \
    -c '{"Args":["query","a"]}'

```

```shell
# 输出:

100

```




### 查看 docker container 

* 如下为安装 chaincode 以后,生成的容器

```shell

[root@localhost jicki]# docker ps -a
CONTAINER ID        IMAGE                                                                                                                                                              COMMAND                  CREATED             STATUS              PORTS                                                                       NAMES
439369210b83        jicki-peer0.org2.jicki.cn-mycc-14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be-4b4600a44f861b0308c82385aacc3c53c711443e78b5bbfda58137dc13822a5a   "chaincode -peer.add…"   9 minutes ago       Up 9 minutes                                                                                    jicki-peer0.org2.jicki.cn-mycc-14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be
471657d67a7a        jicki-peer0.org1.jicki.cn-mycc-14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be-80a4542fb02360e50a162b4d72841d190b64ed34b9b4a06234a09ed2a8c4d392   "chaincode -peer.add…"   9 minutes ago       Up 9 minutes                                                                                    jicki-peer0.org1.jicki.cn-mycc-14332eb253d2983dce11dc45734ea312538a70db1b61d79439fe8a42fb7572be

```





  [1]: https://jicki.cn/img/posts/fabric/fabric.png
  [2]: https://jicki.cn/img/posts/fabric/fabric-2.png

