# Hyperledger Fabric to kubernetes


# Hyperledger Fabric



# 环境说明


## 系统说明

|IP|Hostname|标识|
|-|-|-|
|192.168.0.247|kubernetes-1|K8S-Master and Kubectl-Cli|
|192.168.0.248|kubernetes-2|K8S-Master and Node|
|192.168.0.249|kubernetes-3|K8S-Node and NFS Server|


## 服务说明

|名称|版本|
|-|-|
|CentOS|7 x64|
|Hyperledger Fabric|1.4|
|kubernetes|1.13.2|
|docker|18.06.1-ce|


# Fabric 环境配置



## docker 配置

> 由于 实例化 chaincode 需要由 docker 创建容器
>
> 而 这个容器需要跟 k8s 里的 peer  svc 通信, 所以需要额外配置

```
# 配置 docker 配置

修改 docker 选项 DOCKER_OPTS 

在 DOCKER_OPTS 中 增加 --dns 添加 k8s 中 dns 的 ip


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get svc -n kube-system    

NAME                   TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                  AGE
coredns                ClusterIP   10.233.0.3     <none>        53/UDP,53/TCP,9153/TCP   4d23h


# 我这个环境中 coredns IP 为 10.233.0.3

# 所以 添加

--dns 10.233.0.3

```






## cryptogen 下载

> cryptogen 用于简化生成 fabric 所需要的所有证书


```
# 官方离线下载地址为 
https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/

# 选择相应版本 CentOS 选择 linux-amd64-1.4.0
# Mac 选择 darwin-amd64-1.4.0

# 创建工作目录 ( 后续所有文件都存于此目录中 )

mkdir -p /opt/jicki

cd /opt/jicki

wget https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.4.0/hyperledger-fabric-linux-amd64-1.4.0.tar.gz



# 解压文件
[root@kubernetes-1 /opt/jicki]# tar zxvf hyperledger-fabric-linux-amd64-1.4.0.tar.gz


# 删除 config 这个文件夹,  这里用不到

[root@kubernetes-1 /opt/jicki]# rm -rf config


# 查看文件
[root@kubernetes-1 /opt/jicki]# tree
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


## Fabric 源码下载



```
[root@kubernetes-1 /opt/jicki]# git clone https://github.com/hyperledger/fabric

```



## 生成 证书


```
# 创建 cryptogen.yaml 文件, 1.4版本 crypto-config.yaml 文件更改为 cryptogen.yaml

vi cryptogen.yaml

OrdererOrgs:
  - Name: Orderer
    Domain: orgorderer1
    CA:
        Country: CN
        Province: GuangDong
        Locality: ShenZhen
    Specs:
      - Hostname: orderer0
      
PeerOrgs:
  - Name: Org1
    Domain: org1
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
    Domain: org2
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
# 生成证书

[root@kubernetes-1 /opt/jicki]# cryptogen generate --config=./cryptogen.yaml
org1
org2



# 查看生成目录结构

[root@kubernetes-1 /opt/jicki]# tree -d crypto-config/      
crypto-config/
├── ordererOrganizations
│   └── orgorderer1
│       ├── ca
│       ├── msp
│       │   ├── admincerts
│       │   ├── cacerts
│       │   └── tlscacerts
│       ├── orderers
│       │   └── orderer0.orgorderer1
│       │       ├── msp
│       │       │   ├── admincerts
│       │       │   ├── cacerts
│       │       │   ├── keystore
│       │       │   ├── signcerts
│       │       │   └── tlscacerts
│       │       └── tls
│       ├── tlsca
│       └── users
│           └── Admin@orgorderer1
│               ├── msp
│               │   ├── admincerts
│               │   ├── cacerts
│               │   ├── keystore
│               │   ├── signcerts
│               │   └── tlscacerts
│               └── tls
└── peerOrganizations
    ├── org1
    │   ├── ca
    │   ├── msp
    │   │   ├── admincerts
    │   │   ├── cacerts
    │   │   └── tlscacerts
    │   ├── peers
    │   │   ├── peer0.org1
    │   │   │   ├── msp
    │   │   │   │   ├── admincerts
    │   │   │   │   ├── cacerts
    │   │   │   │   ├── keystore
    │   │   │   │   ├── signcerts
    │   │   │   │   └── tlscacerts
    │   │   │   └── tls
    │   │   └── peer1.org1
    │   │       ├── msp
    │   │       │   ├── admincerts
    │   │       │   ├── cacerts
    │   │       │   ├── keystore
    │   │       │   ├── signcerts
    │   │       │   └── tlscacerts
    │   │       └── tls
    │   ├── tlsca
    │   └── users
    │       ├── Admin@org1
    │       │   ├── msp
    │       │   │   ├── admincerts
    │       │   │   ├── cacerts
    │       │   │   ├── keystore
    │       │   │   ├── signcerts
    │       │   │   └── tlscacerts
    │       │   └── tls
    │       └── User1@org1
    │           ├── msp
    │           │   ├── admincerts
    │           │   ├── cacerts
    │           │   ├── keystore
    │           │   ├── signcerts
    │           │   └── tlscacerts
    │           └── tls
    └── org2
        ├── ca
        ├── msp
        │   ├── admincerts
        │   ├── cacerts
        │   └── tlscacerts
        ├── peers
        │   ├── peer0.org2
        │   │   ├── msp
        │   │   │   ├── admincerts
        │   │   │   ├── cacerts
        │   │   │   ├── keystore
        │   │   │   ├── signcerts
        │   │   │   └── tlscacerts
        │   │   └── tls
        │   └── peer1.org2
        │       ├── msp
        │       │   ├── admincerts
        │       │   ├── cacerts
        │       │   ├── keystore
        │       │   ├── signcerts
        │       │   └── tlscacerts
        │       └── tls
        ├── tlsca
        └── users
            ├── Admin@org2
            │   ├── msp
            │   │   ├── admincerts
            │   │   ├── cacerts
            │   │   ├── keystore
            │   │   ├── signcerts
            │   │   └── tlscacerts
            │   └── tls
            └── User1@org2
                ├── msp
                │   ├── admincerts
                │   ├── cacerts
                │   ├── keystore
                │   ├── signcerts
                │   └── tlscacerts
                └── tls
```


## 生成初始化Fabric文件


```
# 创建 configtx.yaml 文件

vi configtx.yaml


Organizations:

    - &OrdererOrg
        Name: Orgorderer1MSP
        ID: Orgorderer1MSP
        MSPDir: crypto-config/ordererOrganizations/orgorderer1/msp
        Policies:
            Readers:
                Type: Signature
                Rule: "OR('Orgorderer1MSP.member')"
            Writers:
                Type: Signature
                Rule: "OR('Orgorderer1MSP.member')"
            Admins:
                Type: Signature
                Rule: "OR('Orgorderer1MSP.admin')"

    - &Org1
        Name: Org1MSP
        ID: Org1MSP
        MSPDir: crypto-config/peerOrganizations/org1/msp
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
            - Host: peer0.org1
              Port: 7051

    - &Org2
        Name: Org2MSP
        ID: Org2MSP
        MSPDir: crypto-config/peerOrganizations/org2/msp
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
            - Host: peer0.org2
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
        - orderer0.orgorderer1:7050

    BatchTimeout: 2s

    BatchSize:
        MaxMessageCount: 10
        AbsoluteMaxBytes: 99 MB
        PreferredMaxBytes: 512 KB

    Kafka:
        Brokers:
            - kafka-0.broker.kafka:9092
            - kafka-1.broker.kafka:9092
            - kafka-2.broker.kafka:9092
            - kafka-3.broker.kafka:9092

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
                - kafka-0.broker.kafka:9092
                - kafka-1.broker.kafka:9092
                - kafka-2.broker.kafka:9092
                - kafka-3.broker.kafka:9092

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


# 创建 创世区块  TwoOrgsOrdererGenesis 是 
# configtx.yaml 中 Profiles 字段下的名称

[root@kubernetes-1 /opt/jicki]# configtxgen -profile TwoOrgsOrdererGenesis \
 -outputBlock ./channel-artifacts/genesis.block


```

```
# 下面来生成一个 peer 服务 中使用的 tx 文件 TwoOrgsChannel 名称为 
configtx.yaml 中 Profiles 字段下的，这里必须指定上面的 channelID

[root@kubernetes-1 /opt/jicki]# configtxgen -profile TwoOrgsChannel -outputCreateChannelTx ./channel-artifacts/channel.tx -channelID mychannel

```


```
# 定义组织 生成锚节点更新文件

# Org1MSP

[root@kubernetes-1 /opt/jicki]# configtxgen -profile TwoOrgsChannel \
-outputAnchorPeersUpdate ./channel-artifacts/Org1MSPanchors.tx -channelID mychannel -asOrg Org1MSP



# Org2MSP

[root@kubernetes-1 /opt/jicki]# configtxgen -profile TwoOrgsChannel \
-outputAnchorPeersUpdate ./channel-artifacts/Org2MSPanchors.tx -channelID mychannel -asOrg Org2MSP

```


```
# 查看生成文件

[root@kubernetes-1 /opt/jicki]# tree channel-artifacts/
channel-artifacts/
├── channel.tx
├── genesis.block
├── Org1MSPanchors.tx
└── Org2MSPanchors.tx

0 directories, 4 files

```




# 配置 NFS StorageClass

> NFS 用于 k8s 中 文件挂载，共享



## 安装NFS

```
# 服务端 安装 NFS 软件

[root@kubernetes-3 ~]# yum -y install nfs-utils rpcbind


# k8s 所有节点 安装 NFS 客户端

[root@kubernetes-1 ~]# yum -y install nfs-utils
[root@kubernetes-2 ~]# yum -y install nfs-utils

```



## 配置 NFS 

> 这里需要创建几个目录 /opt/nfs/data,  /opt/nfs/fabric/
>
> /opt/nfs/data 用于存放 zk, kafka 数据
>
> /opt/nfs/fabric 用于存放 fabric 所有的文件

```
# 创建目录

[root@kubernetes-3 ~]# mkdir -p /opt/nfs/{data,fabric}


# 修改配置文件

[root@kubernetes-3 ~]# vi /etc/exports

增加

/opt/nfs/data       192.168.0.0/24(rw,sync,no_root_squash)
/opt/nfs/fabric     192.168.0.0/24(rw,sync,no_root_squash)

```


```
# 启动 NFS 服务

[root@kubernetes-3 ~]# systemctl enable rpcbind.service    
[root@kubernetes-3 ~]# systemctl enable nfs-server.service

[root@kubernetes-3 ~]# systemctl start rpcbind.service    
[root@kubernetes-3 ~]# systemctl start nfs-server.service

[root@kubernetes-3 ~]# systemctl status rpcbind.service    
[root@kubernetes-3 ~]# systemctl status nfs-server.service



# 查看服务
[root@kubernetes-3 ~]# showmount -e 192.168.0.249
Export list for 192.168.0.249:
/opt/nfs/fabric 192.168.0.0/24
/opt/nfs/data   192.168.0.0/24

```


## 配置 NFS Client Provisioner

> 官方 说明 https://github.com/kubernetes-incubator/external-storage/tree/master/nfs-client

```
# 下载官方 yaml 文件

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/rbac.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/deployment.yaml

wget https://raw.githubusercontent.com/kubernetes-incubator/external-storage/master/nfs-client/deploy/class.yaml


```


```
# 修改 deployment.yaml 文件

vi deployment.yaml


apiVersion: v1
kind: ServiceAccount
metadata:
  name: nfs-client-provisioner
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  name: nfs-client-provisioner
spec:
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: nfs-client-provisioner
    spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: fuseim.pri/ifs
            - name: NFS_SERVER
              value: 192.168.0.249
            - name: NFS_PATH
              value: /opt/nfs/data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.0.249
            path: /opt/nfs/data

```


```
# 修改 class.yaml , 这里修改 name 名称

vi class.yaml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: zk-kafka-nfs-storage
provisioner: fuseim.pri/ifs 
parameters:
  archiveOnDelete: "false"

```






```
# 导入 yaml 文件

[root@kubernetes-1 /opt/yaml/nfs-client]# kubectl apply -f .
storageclass.storage.k8s.io/fabric-nfs-storage created
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
deployment.extensions/nfs-client-provisioner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created

```


```
# 查看服务

[root@kubernetes-1 /opt/yaml/nfs-client]# kubectl get storageclass
NAME                   PROVISIONER      AGE
zk-kafka-nfs-storage   fuseim.pri/ifs   27s


[root@kubernetes-1 /opt/yaml/nfs-client]# kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-578785c589-qlgnx   1/1     Running   0          2m24s
```



# 部署 fabric


## 创建 zookeeper


```
[root@kubernetes-1 /opt/jicki/k8s-yaml]# vi kafka-namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: kafka

```


```
# 创建 zk

[root@kubernetes-1 /opt/jicki/k8s-yaml]# vi zookeeper.yaml


apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: zoo
  namespace: kafka
spec:
  serviceName: "zoo"
  replicas: 4
  template:
    metadata:
      labels:
        app: zookeeper
    spec:
      terminationGracePeriodSeconds: 10
      containers:
        - name: zookeeper
          image: hyperledger/fabric-zookeeper:0.4.14
          ports:
            - containerPort: 2181
              name: client
            - containerPort: 2888
              name: peer
            - containerPort: 3888
              name: leader-election
          volumeMounts:
            - name: zkdata
              mountPath: /var/lib/zookeeper
  volumeClaimTemplates:
  - metadata:
      name: zkdata 
      annotations:
        volume.beta.kubernetes.io/storage-class: "zk-kafka-nfs-storage"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: zoo
  namespace: kafka
spec:
  ports:
  - port: 2888
    name: peer
  - port: 3888
    name: leader-election
  clusterIP: None
  selector:
    app: zookeeper

---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  namespace: kafka
spec:
  ports:
  - port: 2181
    name: client
  selector:
    app: zookeeper

```


```
# 导入 yaml 文件

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f namespace.yaml


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f zookeeper.yaml


```

```
# 查看服务

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get all -n kafka
NAME        READY   STATUS    RESTARTS   AGE
pod/zoo-0   1/1     Running   0          20m
pod/zoo-1   1/1     Running   0          9m11s
pod/zoo-2   1/1     Running   0          111s
pod/zoo-3   1/1     Running   0          108s

NAME                TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)             AGE
service/zoo         ClusterIP   None           <none>        2888/TCP,3888/TCP   21m
service/zookeeper   ClusterIP   10.233.22.96   <none>        2181/TCP            21m

NAME                   READY   AGE
statefulset.apps/zoo   4/4     20m


```



## 创建 kafka


```
vi kafka.yaml



apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: kafka
  namespace: kafka
spec:
  serviceName: "broker"
  replicas: 4
  template:
    metadata:
      labels:
        app: kafka
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: broker
        image: hyperledger/fabric-kafka:0.4.14
        ports:
          - containerPort: 9092
        env:
          - name: KAFKA_MESSAGE_MAX_BYTES
            value: "102760448"
          - name: KAFKA_REPLICA_FETCH_MAX_BYTES
            value: "102760448"
          - name: KAFKA_UNCLEAN_LEADER_ELECTION_ENABLE
            value: "false"
          - name: KAFKA_ZOOKEEPER_CONNECT
            value: zoo-0.zoo:2181,zoo-1.zoo:2181,zoo-2.zoo:2181,zoo-3.zoo:2181
          - name: KAFKA_PORT
            value: "9092"
          - name: GODEBUG
            value: netdns=go
          - name: KAFKA_ZOOKEEPER_CONNECTION_TIMEOUT_MS
            value: "30000"
          - name: KAFKA_LOG_DIRS
            value: /opt/kafka/data
          - name: KAFKA_LOG_RETENTION_MS
            value: "-1"
        volumeMounts:
        - name: kafkadata
          mountPath: /opt/kafka/data
  volumeClaimTemplates:
  - metadata:
      name: kafkadata 
      annotations:
        volume.beta.kubernetes.io/storage-class: "zk-kafka-nfs-storage"
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 20Gi


---
apiVersion: v1
kind: Service
metadata:
  name: kafka
  namespace: kafka
spec:
  ports:
  - port: 9092
  selector:
    app: kafka
    
---
apiVersion: v1
kind: Service
metadata:
  name: broker
  namespace: kafka
spec:
  ports:
  - port: 9092
  clusterIP: None
  selector:
    app: kafka


```


```
# 导入 yaml 文件

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f kafka.yaml 
statefulset.apps/kafka created
service/kafka created
service/broker created

```


```
# 查看服务

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get pods -n kafka
NAME      READY   STATUS    RESTARTS   AGE
kafka-0   1/1     Running   0          2m26s
kafka-1   1/1     Running   0          100s
kafka-2   1/1     Running   0          53s
kafka-3   1/1     Running   0          43s



[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get svc -n kafka
NAME        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
broker      ClusterIP   None            <none>        9092/TCP            2m45s
kafka       ClusterIP   10.233.48.233   <none>        9092/TCP            2m45s



[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get statefulset -n kafka
NAME    READY   AGE
kafka   4/4     3m13s

```




## 创建 fabric 服务



### 初始化 

```
# 在 操作的机器上 挂载 nfs 目录 /opt/nfs/fabric
# 为了预先拷贝一些证书进去

mkdir -p /opt/nfs/fabric

mount -t nfs 192.168.0.249:/opt/nfs/fabric /opt/nfs/fabric
```


```
# 拷贝文件到挂载目录

# 创建两个目录 data 数据目录, 以及 证书，创世区块 等文件目录

mkdir -p /opt/nfs/fabric/{data,fabric}
```

```
# 拷贝 证书文件 创世区块 文件

mkdir ./channel-artifacts/chaincode

cp -r ./fabric/examples/chaincode/go/example* channel-artifacts/chaincode/

cp ./channel-artifacts/genesis.block ./crypto-config/ordererOrganizations/*

cp -r ./crypto-config /opt/nfs/fabric/fabric/

cp -r ./channel-artifacts /opt/nfs/fabric/fabric/

```

```
# 创建 fabric 运行时生成的共享数据文件夹

mkdir -p /opt/nfs/fabric/data/{orderer,peer}
mkdir -p /opt/nfs/fabric/data/orderer/orgorderer1/orderer0
mkdir -p /opt/nfs/fabric/data/peer/org{1,2}/ca
mkdir -p /opt/nfs/fabric/data/peer/org{1,2}/peer{0,1}/{couchdb,peerdata}

```



### 配置 orderer


```
# 创建 orderer1 的 namespaces 

vi orgorderer1-namespace.yaml


apiVersion: v1
kind: Namespace
metadata:
    name: orgorderer1

```

```
# 创建 orderer1 的 pv 与 pvc


vi orgorderer1-pv-pvc.yaml


apiVersion: v1
kind: PersistentVolume
metadata:
  name: orgorderer1-pv
  labels:
    app: orgorderer1-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/fabric/crypto-config/ordererOrganizations/orgorderer1
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: orgorderer1
 name: orgorderer1-pv
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 10Mi
 selector:
   matchLabels:
     app: orgorderer1-pv

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: orgorderer1-pvdata
  labels:
    app: orgorderer1-pvdata
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/data/orderer/orgorderer1
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: orgorderer1
 name: orgorderer1-pvdata
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 10Gi
 selector:
   matchLabels:
     app: orgorderer1-pvdata

```


```
# 创建 orderer1 Deployment 服务


vi orderer0.orgorderer1.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: orgorderer1
  name: orderer0-orgorderer1
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      labels:
        app: hyperledger
        role: orderer
        org: orgorderer1
        orderer-id: orderer0
    spec:
      containers:
      - name: orderer0-orgorderer1
        image: hyperledger/fabric-orderer:1.4.0
        env:
        - name: ORDERER_GENERAL_LOGLEVEL
          value: debug
        - name: ORDERER_GENERAL_LISTENADDRESS
          value: 0.0.0.0
        - name: ORDERER_GENERAL_GENESISMETHOD
          value: file
        - name: ORDERER_GENERAL_GENESISFILE
          value: /var/hyperledger/orderer/orderer.genesis.block
        - name: ORDERER_GENERAL_LOCALMSPID
          value: Orgorderer1MSP
        - name: ORDERER_GENERAL_LOCALMSPDIR
          value: /var/hyperledger/orderer/msp
        - name: ORDERER_GENERAL_TLS_ENABLED
          value: "false"
        - name: ORDERER_GENERAL_TLS_PRIVATEKEY
          value: /var/hyperledger/orderer/tls/server.key
        - name: ORDERER_GENERAL_TLS_CERTIFICATE
          value: /var/hyperledger/orderer/tls/server.crt
        - name: ORDERER_GENERAL_TLS_ROOTCAS
          value: '[/var/hyperledger/orderer/tls/ca.crt]'
        - name: ORDERER_KAFKA_VERSION
          value: 0.10.0.1
        - name: ORDERER_KAFKA_VERBOSE
          value: "true"
        - name: ORDERER_KAFKA_BROKERS
          value: "kafka-0.broker.kafka:9092,kafka-1.broker.kafka:9092,kafka-2.broker.kafka:9092,kafka-3.broker.kafka:9092"
        - name: GODEBUG
          value: netdns=go
        workingDir: /opt/gopath/src/github.com/hyperledger/fabric
        ports:
         - containerPort: 7050
        command: ["orderer"]
        volumeMounts:
         - mountPath: /var/hyperledger/orderer/msp
           name: certificate
           subPath: orderers/orderer0.orgorderer1/msp
         - mountPath: /var/hyperledger/orderer/tls
           name: certificate
           subPath: orderers/orderer0.orgorderer1/tls
         - mountPath: /var/hyperledger/orderer/orderer.genesis.block
           name: certificate
           subPath: genesis.block
         - mountPath: /var/hyperledger/production
           name: ordererdata
           subPath: orderer0
      volumes:
       - name: certificate
         persistentVolumeClaim:
             claimName: orgorderer1-pv
       - name: ordererdata
         persistentVolumeClaim:
             claimName: orgorderer1-pvdata

---
apiVersion: v1
kind: Service
metadata:
  name: orderer0
  namespace: orgorderer1
spec:
 selector:
   app: hyperledger
   role: orderer
   orderer-id: orderer0
   org: orgorderer1
 type: NodePort
 ports:
   - name: listen-endpoint
     protocol: TCP
     port: 7050
     targetPort: 7050
     nodePort: 32000

```

```
# 导入 yaml 文件 创建服务


# 创建 namespaces

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f orgorderer1-namespace.yaml 
namespace/orgorderer1 created


# 创建 pv pvc

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f orgorderer1-pv-pvc.yaml 
persistentvolume/orgorderer1-pv created
persistentvolumeclaim/orgorderer1-pv created
persistentvolume/orgorderer1-pvdata created
persistentvolumeclaim/orgorderer1-pvdata created



# 创建 deployment

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f orderer0.orgorderer1.yaml 
deployment.extensions/orderer0-orgorderer1 created
service/orderer0 created

```


```
# 查看创建的服务

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get pods -n orgorderer1
NAME                                    READY   STATUS    RESTARTS   AGE
orderer0-orgorderer1-6b896b94c4-5gs2c   1/1     Running   0          36s



[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS           REASON   AGE
orgorderer1-pv                             500Mi      RWX            Retain           Bound    orgorderer1/orgorderer1-pv                                       115s
orgorderer1-pvdata                         10Gi       RWX            Retain           Bound    orgorderer1/orgorderer1-pvdata                                   115s



[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get pvc -n orgorderer1
NAME                 STATUS   VOLUME               CAPACITY   ACCESS MODES   STORAGECLASS   AGE
orgorderer1-pv       Bound    orgorderer1-pv       500Mi      RWX                           2m9s
orgorderer1-pvdata   Bound    orgorderer1-pvdata   10Gi       RWX                           2m9s

```


### 配置 org1 服务


> 每个 org 包含 ca  cli  peer 三个服务



```
# 配置 org1 namespaces

vi org1-namespace.yaml


apiVersion: v1
kind: Namespace
metadata:
    name: org1

```



```
# 配置 org1  pv 与 pvc


vi org1-pv-pvc.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: org1-pv
  labels:
    app: org1-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/fabric/crypto-config/peerOrganizations/org1
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: org1
 name: org1-pv
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 10Mi
 selector:
   matchLabels:
     app: org1-pv

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: org1-pvdata
  labels:
    app: org1-pvdata
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/data/peer/org1
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: org1
 name: org1-pvdata
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 10Gi
 selector:
   matchLabels:
     app: org1-pvdata

```


```
# 配置 org1  ca
# FABRIC_CA_SERVER_TLS_KEYFILE 配置为自己生成的 sk
# sk 存在目录 crypto-config/peerOrganizations/org1/ca/



vi org1-ca.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: org1
  name: ca
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      labels:
       app: hyperledger
       role: ca
       org: org1
       name: ca
    spec:
     containers:
       - name: ca
         image: hyperledger/fabric-ca:1.4.0
         env:
         - name: FABRIC_CA_HOME
           value: /etc/hyperledger/fabric-ca-server
         - name: FABRIC_CA_SERVER_CA_NAME
           value: ca
         - name: FABRIC_CA_SERVER_TLS_ENABLED
           value: "false"
         - name: FABRIC_CA_SERVER_TLS_CERTFILE
           value: /etc/hyperledger/fabric-ca-server-config/ca.org1-cert.pem
         - name: FABRIC_CA_SERVER_TLS_KEYFILE
           value: /etc/hyperledger/fabric-ca-server-config/904aa978bf690c8198526a6410045cc308468ca7171d02af70ffc7d969eb4c7e_sk
         - name: GODEBUG
           value: netdns=go
         ports:
          - containerPort: 7054
         command: ["sh"]
         args: ["-c", " fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org1-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/904aa978bf690c8198526a6410045cc308468ca7171d02af70ffc7d969eb4c7e_sk -b admin:adminpw -d "]
         volumeMounts:
          - mountPath: /etc/hyperledger/fabric-ca-server-config
            name: certificate
            subPath: ca/
          - mountPath: /etc/hyperledger/fabric-ca-server
            name: cadata
            subPath: ca/
     volumes:
       - name: certificate
         persistentVolumeClaim:
            claimName: org1-pv
       - name: cadata
         persistentVolumeClaim:
            claimName: org1-pvdata

---
apiVersion: v1
kind: Service
metadata:
   namespace: org1
   name: ca
spec:
 selector:
   app: hyperledger
   role: ca
   org: org1
   name: ca
 type: NodePort
 ports:
   - name: endpoint
     protocol: TCP
     port: 7054
     targetPort: 7054
     nodePort: 30000

```



```
# 配置 org1  peer 服务 
# 这里有两个 peer 分别为 peer0, peer1


vi peer0.org1.yaml



apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: org1
  name: peer0-org1
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
       app: hyperledger
       role: peer
       peer-id: peer0
       org: org1
    spec:
      containers:
      - name: couchdb
        image: hyperledger/fabric-couchdb:0.4.10
        env:
         - name: COUCHDB_USER
           value: ""
         - name: COUCHDB_PASSWORD
           value: ""
        ports:
          - containerPort: 5984
        volumeMounts:
          - mountPath: /opt/couchdb/data
            name: peerdata
            subPath: peer0/couchdb
      - name: peer0-org1
        image: hyperledger/fabric-peer:1.4.0
        env:
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: ""
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: ""
        - name: CORE_VM_ENDPOINT
          value: "unix:///host/var/run/docker.sock"
        - name: CORE_LOGGING_LEVEL
          value: "DEBUG"
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_PEER_TLS_CERT_FILE
          value: "/etc/hyperledger/fabric/tls/server.crt"
        - name: CORE_PEER_TLS_KEY_FILE
          value: "/etc/hyperledger/fabric/tls/server.key"
        - name: CORE_PEER_TLS_ROOTCERT_FILE
          value: "/etc/hyperledger/fabric/tls/ca.crt"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_CHAINCODELISTENADDRESS
          value: "0.0.0.0:7052"
        - name: CORE_PEER_ID
          value: peer0.org1
        - name: CORE_PEER_ADDRESS
          value: peer0.org1:7051
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: peer0.org1:7051
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: peer0.org1:7051
        - name: CORE_PEER_LOCALMSPID
          value: Org1MSP
        workingDir: /opt/gopath/src/github.com/hyperledger/fabric/peer
        ports:
         - containerPort: 7051
         - containerPort: 7052
         - containerPort: 7053
        command: ["peer"]
        args: ["node","start"]
        volumeMounts:
         - mountPath: /etc/hyperledger/fabric/msp
           name: certificate
           subPath: peers/peer0.org1/msp
         - mountPath: /etc/hyperledger/fabric/tls
           name: certificate
           subPath: peers/peer0.org1/tls
         - mountPath: /var/hyperledger/production
           name: peerdata
           subPath: peer0/peerdata
         - mountPath: /host/var/run/
           name: run
      volumes:
       - name: certificate
         persistentVolumeClaim:
             claimName: org1-pv
       - name: peerdata
         persistentVolumeClaim:
             claimName: org1-pvdata
       - name: run
         hostPath:
           path: /var/run

---
apiVersion: v1
kind: Service
metadata:
   namespace: org1
   name: peer0
spec:
 selector:
   app: hyperledger
   role: peer
   peer-id: peer0
   org: org1
 type: NodePort
 ports:
   - name: externale-listen-endpoint
     protocol: TCP
     port: 7051
     targetPort: 7051
     nodePort: 30001

   - name: chaincode-listen
     protocol: TCP
     port: 7052
     targetPort: 7052
     nodePort: 30002

   - name: event-listen
     protocol: TCP
     port: 7053
     targetPort: 7053
     nodePort: 30003
     
```




```

vi peer1.org1.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: org1
  name: peer1-org1
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
       app: hyperledger
       role: peer
       peer-id: peer1
       org: org1
    spec:
      containers:
      - name: couchdb
        image: hyperledger/fabric-couchdb:0.4.10
        env:
         - name: COUCHDB_USER
           value: ""
         - name: COUCHDB_PASSWORD
           value: ""
        ports:
          - containerPort: 5984
        volumeMounts:
          - mountPath: /opt/couchdb/data
            name: peerdata
            subPath: peer1/couchdb
      - name: peer1-org1
        image: hyperledger/fabric-peer:1.4.0
        env:
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: ""
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: ""
        - name: CORE_VM_ENDPOINT
          value: "unix:///host/var/run/docker.sock"
        - name: CORE_LOGGING_LEVEL
          value: "DEBUG"
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_PEER_TLS_CERT_FILE
          value: "/etc/hyperledger/fabric/tls/server.crt"
        - name: CORE_PEER_TLS_KEY_FILE
          value: "/etc/hyperledger/fabric/tls/server.key"
        - name: CORE_PEER_TLS_ROOTCERT_FILE
          value: "/etc/hyperledger/fabric/tls/ca.crt"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_CHAINCODELISTENADDRESS
          value: "0.0.0.0:7052"
        - name: CORE_PEER_ID
          value: peer1.org1
        - name: CORE_PEER_ADDRESS
          value: peer1.org1:7051
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: peer1.org1:7051
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: peer1.org1:7051
        - name: CORE_PEER_LOCALMSPID
          value: Org1MSP
        workingDir: /opt/gopath/src/github.com/hyperledger/fabric/peer
        ports:
         - containerPort: 7051
         - containerPort: 7052
         - containerPort: 7053
        command: ["peer"]
        args: ["node","start"]
        volumeMounts:
         - mountPath: /etc/hyperledger/fabric/msp
           name: certificate
           subPath: peers/peer1.org1/msp
         - mountPath: /etc/hyperledger/fabric/tls
           name: certificate
           subPath: peers/peer1.org1/tls
         - mountPath: /var/hyperledger/production
           name: peerdata
           subPath: peer1/peerdata
         - mountPath: /host/var/run/
           name: run
      volumes:
       - name: certificate
         persistentVolumeClaim:
             claimName: org1-pv
       - name: peerdata
         persistentVolumeClaim:
             claimName: org1-pvdata
       - name: run
         hostPath:
           path: /var/run

---
apiVersion: v1
kind: Service
metadata:
   namespace: org1
   name: peer1
spec:
 selector:
   app: hyperledger
   role: peer
   peer-id: peer1
   org: org1
 type: NodePort
 ports:
   - name: externale-listen-endpoint
     protocol: TCP
     port: 7051
     targetPort: 7051
     nodePort: 30004

   - name: chaincode-listen
     protocol: TCP
     port: 7052
     targetPort: 7052
     nodePort: 30005

   - name: event-listen
     protocol: TCP
     port: 7053
     targetPort: 7053
     nodePort: 30006


```



```
# 导入 yaml 文件 创建服务

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org1-namespace.yaml 
namespace/org1 created


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org1-pv-pvc.yaml 
persistentvolume/org1-pv created
persistentvolumeclaim/org1-pv created
persistentvolume/org1-pvdata created
persistentvolumeclaim/org1-pvdata created


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org1-ca.yaml 
deployment.extensions/ca created
service/ca created

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f peer0.org1.yaml 
deployment.extensions/peer0-org1 created
service/peer0 created


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f peer1.org1.yaml 
deployment.extensions/peer1-org1 created
service/peer1 created

```


```
# 查看 创建的服务


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get all -n org1
NAME                              READY   STATUS    RESTARTS   AGE
pod/ca-75b9859c54-d5x9b           1/1     Running   0          120m
pod/peer0-org1-5767789cd7-7g2z7   2/2     Running   0          22m
pod/peer1-org1-866b6bcd45-c5mbp   2/2     Running   0          22m

NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                        AGE
service/ca      NodePort   10.233.5.5      <none>        7054:30000/TCP                                 120m
service/peer0   NodePort   10.233.57.55    <none>        7051:30001/TCP,7052:30002/TCP,7053:30003/TCP   36m
service/peer1   NodePort   10.233.31.189   <none>        7051:30004/TCP,7052:30005/TCP,7053:30006/TCP   34m

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ca           1/1     1            1           120m
deployment.apps/peer0-org1   1/1     1            1           36m
deployment.apps/peer1-org1   1/1     1            1           34m

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/ca-75b9859c54           1         1         1       120m
replicaset.apps/peer0-org1-5767789cd7   1         1         1       22m
replicaset.apps/peer0-org1-8c98c4c9     0         0         0       36m
replicaset.apps/peer1-org1-768c8d8f69   0         0         0       34m
replicaset.apps/peer1-org1-866b6bcd45   1         1         1       22m

```



### 配置 org2 服务


```
# 配置 org2 namespaces

vi org2-namespace.yaml


apiVersion: v1
kind: Namespace
metadata:
    name: org2
```



```
# 配置 org2 pv pvc

vi org2-pv-pvc.yaml


apiVersion: v1
kind: PersistentVolume
metadata:
  name: org2-pv
  labels:
    app: org2-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/fabric/crypto-config/peerOrganizations/org2
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: org2
 name: org2-pv
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 10Mi
 selector:
   matchLabels:
     app: org2-pv

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: org2-pvdata
  labels:
    app: org2-pvdata
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/data/peer/org2
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: org2
 name: org2-pvdata
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 10Gi
 selector:
   matchLabels:
     app: org2-pvdata

```


```
# 配置 org2  ca
# FABRIC_CA_SERVER_TLS_KEYFILE 配置为自己生成的 sk
# sk 存在目录 crypto-config/peerOrganizations/org2/ca/

vi org2-ca.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: org2
  name: ca
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      labels:
       app: hyperledger
       role: ca
       org: org2
       name: ca
    spec:
     containers:
       - name: ca
         image: hyperledger/fabric-ca:1.4.0
         env:
         - name: FABRIC_CA_HOME
           value: /etc/hyperledger/fabric-ca-server
         - name: FABRIC_CA_SERVER_CA_NAME
           value: ca
         - name: FABRIC_CA_SERVER_TLS_ENABLED
           value: "false"
         - name: FABRIC_CA_SERVER_TLS_CERTFILE
           value: /etc/hyperledger/fabric-ca-server-config/ca.org2-cert.pem
         - name: FABRIC_CA_SERVER_TLS_KEYFILE
           value: /etc/hyperledger/fabric-ca-server-config/44299d5dfb204e07b5274a7048269171876157d1efdab11a5ac631bd78d2fe0d_sk
         - name: GODEBUG
           value: netdns=go
         ports:
          - containerPort: 7054
         command: ["sh"]
         args: ["-c", " fabric-ca-server start --ca.certfile /etc/hyperledger/fabric-ca-server-config/ca.org2-cert.pem --ca.keyfile /etc/hyperledger/fabric-ca-server-config/44299d5dfb204e07b5274a7048269171876157d1efdab11a5ac631bd78d2fe0d_sk -b admin:adminpw -d "]
         volumeMounts:
          - mountPath: /etc/hyperledger/fabric-ca-server-config
            name: certificate
            subPath: ca/
          - mountPath: /etc/hyperledger/fabric-ca-server
            name: cadata
            subPath: ca/
     volumes:
       - name: certificate
         persistentVolumeClaim:
            claimName: org2-pv
       - name: cadata
         persistentVolumeClaim:
            claimName: org2-pvdata

---
apiVersion: v1
kind: Service
metadata:
   namespace: org2
   name: ca
spec:
 selector:
   app: hyperledger
   role: ca
   org: org2
   name: ca
 type: NodePort
 ports:
   - name: endpoint
     protocol: TCP
     port: 7054
     targetPort: 7054
     nodePort: 30100

```



```
# 配置 org2  peer 服务 
# 这里有两个 peer 分别为 peer0, peer1


vi peer0.org2.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: org2
  name: peer0-org2
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
       app: hyperledger
       role: peer
       peer-id: peer0
       org: org2
    spec:
      containers:
      - name: couchdb
        image: hyperledger/fabric-couchdb:0.4.10
        env:
        - name: COUCHDB_USER
          value: ""
        - name: COUCHDB_PASSWORD
          value: ""
        ports:
         - containerPort: 5984
        volumeMounts:
         - mountPath: /opt/couchdb/data
           name: peerdata
           subPath: peer0/couchdb
      - name: peer0-org2
        image: hyperledger/fabric-peer:1.4.0
        env:
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: ""
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: ""
        - name: CORE_VM_ENDPOINT
          value: "unix:///host/var/run/docker.sock"
        - name: CORE_LOGGING_LEVEL
          value: "DEBUG"
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_PEER_TLS_CERT_FILE
          value: "/etc/hyperledger/fabric/tls/server.crt"
        - name: CORE_PEER_TLS_KEY_FILE
          value: "/etc/hyperledger/fabric/tls/server.key"
        - name: CORE_PEER_TLS_ROOTCERT_FILE
          value: "/etc/hyperledger/fabric/tls/ca.crt"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_CHAINCODELISTENADDRESS
          value: "0.0.0.0:7052"
        - name: CORE_PEER_ID
          value: peer0.org2
        - name: CORE_PEER_ADDRESS
          value: peer0.org2:7051
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: peer0.org2:7051
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: peer0.org2:7051
        - name: CORE_PEER_LOCALMSPID
          value: Org2MSP
        workingDir: /opt/gopath/src/github.com/hyperledger/fabric/peer
        ports:
         - containerPort: 7051
         - containerPort: 7052
         - containerPort: 7053
        command: ["peer"]
        args: ["node","start"]
        volumeMounts:
         - mountPath: /etc/hyperledger/fabric/msp
           name: certificate
           subPath: peers/peer0.org2/msp
         - mountPath: /etc/hyperledger/fabric/tls
           name: certificate
           subPath: peers/peer0.org2/tls
         - mountPath: /var/hyperledger/production
           name: peerdata
           subPath: peer0/peerdata
         - mountPath: /host/var/run/
           name: run
      volumes:
       - name: certificate
         persistentVolumeClaim:
             claimName: org2-pv
       - name: peerdata
         persistentVolumeClaim:
             claimName: org2-pvdata
       - name: run
         hostPath:
           path: /var/run

---
apiVersion: v1
kind: Service
metadata:
   namespace: org2
   name: peer0
spec:
 selector:
   app: hyperledger
   role: peer
   peer-id: peer0
   org: org2
 type: NodePort
 ports:
   - name: externale-listen-endpoint
     protocol: TCP
     port: 7051
     targetPort: 7051
     nodePort: 30101

   - name: chaincode-listen
     protocol: TCP
     port: 7052
     targetPort: 7052
     nodePort: 30102

   - name: event-listen
     protocol: TCP
     port: 7053
     targetPort: 7053
     nodePort: 30103

```



```
# 配置 org2 的 peer1


vi peer1.org2.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: org2
  name: peer1-org2
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
       app: hyperledger
       role: peer
       peer-id: peer1
       org: org2
    spec:
      containers:
      - name: couchdb
        image: hyperledger/fabric-couchdb:0.4.10
        env:
        - name: COUCHDB_USER
          value: ""
        - name: COUCHDB_PASSWORD
          value: ""
        ports:
         - containerPort: 5984
        volumeMounts:
         - mountPath: /opt/couchdb/data
           name: peerdata
           subPath: peer1/couchdb
      - name: peer1-org2
        image: hyperledger/fabric-peer:1.4.0
        env:
        - name: CORE_LEDGER_STATE_STATEDATABASE
          value: "CouchDB"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_COUCHDBADDRESS
          value: "localhost:5984"
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_USERNAME
          value: ""
        - name: CORE_LEDGER_STATE_COUCHDBCONFIG_PASSWORD
          value: ""
        - name: CORE_VM_ENDPOINT
          value: "unix:///host/var/run/docker.sock"
        - name: CORE_LOGGING_LEVEL
          value: "DEBUG"
        - name: CORE_PEER_TLS_ENABLED
          value: "false"
        - name: CORE_PEER_GOSSIP_USELEADERELECTION
          value: "true"
        - name: CORE_PEER_GOSSIP_ORGLEADER
          value: "false"
        - name: CORE_PEER_PROFILE_ENABLED
          value: "true"
        - name: CORE_PEER_TLS_CERT_FILE
          value: "/etc/hyperledger/fabric/tls/server.crt"
        - name: CORE_PEER_TLS_KEY_FILE
          value: "/etc/hyperledger/fabric/tls/server.key"
        - name: CORE_PEER_TLS_ROOTCERT_FILE
          value: "/etc/hyperledger/fabric/tls/ca.crt"
        - name: CORE_PEER_ADDRESSAUTODETECT
          value: "true"
        - name: CORE_PEER_CHAINCODELISTENADDRESS
          value: "0.0.0.0:7052"
        - name: CORE_PEER_ID
          value: peer1.org2
        - name: CORE_PEER_ADDRESS
          value: peer1.org2:7051
        - name: CORE_PEER_GOSSIP_BOOTSTRAP
          value: peer1.org2:7051
        - name: CORE_PEER_GOSSIP_EXTERNALENDPOINT
          value: peer1.org2:7051
        - name: CORE_PEER_LOCALMSPID
          value: Org2MSP
        workingDir: /opt/gopath/src/github.com/hyperledger/fabric/peer
        ports:
         - containerPort: 7051
         - containerPort: 7052
         - containerPort: 7053
        command: ["peer"]
        args: ["node","start"]
        volumeMounts:
         - mountPath: /etc/hyperledger/fabric/msp
           name: certificate
           subPath: peers/peer1.org2/msp
         - mountPath: /etc/hyperledger/fabric/tls
           name: certificate
           subPath: peers/peer1.org2/tls
         - mountPath: /var/hyperledger/production
           name: peerdata
           subPath: peer1/peerdata
         - mountPath: /host/var/run/
           name: run
      volumes:
       - name: certificate
         persistentVolumeClaim:
             claimName: org2-pv
       - name: peerdata
         persistentVolumeClaim:
             claimName: org2-pvdata
       - name: run
         hostPath:
           path: /var/run

---
apiVersion: v1
kind: Service
metadata:
   namespace: org2
   name: peer1
spec:
 selector:
   app: hyperledger
   role: peer
   peer-id: peer1
   org: org2
 type: NodePort
 ports:
   - name: externale-listen-endpoint
     protocol: TCP
     port: 7051
     targetPort: 7051
     nodePort: 30104

   - name: chaincode-listen
     protocol: TCP
     port: 7052
     targetPort: 7052
     nodePort: 30105

   - name: event-listen
     protocol: TCP
     port: 7053
     targetPort: 7053
     nodePort: 30106

```


```
# 导入 yaml 文件 创建服务

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org2-namespace.yaml 
namespace/org2 created


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org2-pv-pvc.yaml 
persistentvolume/org2-pv created
persistentvolumeclaim/org2-pv created
persistentvolume/org2-pvdata created
persistentvolumeclaim/org2-pvdata created


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org2-ca.yaml 
deployment.extensions/ca created
service/ca created

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f peer0.org2.yaml 
deployment.extensions/peer0-org2 created
service/peer0 created


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f peer1.org2.yaml 
deployment.extensions/peer1-org2 created
service/peer1 created

```


```
# 查看 org2 的服务


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get all -n org2
NAME                              READY   STATUS    RESTARTS   AGE
pod/ca-594d86d8fc-fwg7r           1/1     Running   0          55s
pod/peer0-org2-68579cc9fd-gjmdw   2/2     Running   0          40s
pod/peer1-org2-7645b97c44-5f64k   2/2     Running   0          30s

NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                                        AGE
service/ca      NodePort   10.233.4.94     <none>        7054:30100/TCP                                 55s
service/peer0   NodePort   10.233.49.35    <none>        7051:30101/TCP,7052:30102/TCP,7053:30103/TCP   40s
service/peer1   NodePort   10.233.56.135   <none>        7051:30104/TCP,7052:30105/TCP,7053:30106/TCP   30s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/ca           1/1     1            1           55s
deployment.apps/peer0-org2   1/1     1            1           40s
deployment.apps/peer1-org2   1/1     1            1           30s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/ca-594d86d8fc           1         1         1       55s
replicaset.apps/peer0-org2-68579cc9fd   1         1         1       40s
replicaset.apps/peer1-org2-7645b97c44   1         1         1       30s

```




### 配置 cli 服务

> cli 服务，用于 命令行操作组织节点的环境。
>
> 这里每一个 org 配置一个 cli 服务，方便操作不同的 org 节点
>



```
# 创建 org1  cli 的 pv pvc 


vi org1-cli-pv-pvc.yaml


apiVersion: v1
kind: PersistentVolume
metadata:
    name: org1-cli-pv
    labels:
      app: org1-cli-pv
spec:
    capacity:
       storage: 500Mi
    accessModes:
       - ReadWriteMany
    nfs:
      path: /opt/nfs/fabric/fabric/channel-artifacts
      server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    namespace: org1
    name: org1-cli-pv
spec:
   accessModes:
     - ReadWriteMany
   resources:
      requests:
        storage: 10Mi
   selector:
     matchLabels:
       app: org1-cli-pv

```


```
# 创建 org1  cli  服务

vi org1-cli.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
   namespace: org1
   name: cli
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      labels:
       app: cli
    spec:
      containers:
        - name: cli
          image: hyperledger/fabric-tools:1.4.0
          env:
          - name: CORE_PEER_TLS_ENABLED
            value: "false"
          - name: CORE_VM_ENDPOINT
            value: unix:///host/var/run/docker.sock
          - name: GOPATH
            value: /opt/gopath
          - name: CORE_LOGGING_LEVEL
            value: DEBUG
          - name: CORE_PEER_ID
            value: cli
          - name: CORE_PEER_ADDRESS
            value: peer0.org1:7051
          - name: CORE_PEER_LOCALMSPID
            value: Org1MSP
          - name: CORE_PEER_MSPCONFIGPATH
            value: /etc/hyperledger/fabric/msp
          - name: GODEBUG
            value: netdns=go
          workingDir: /opt/gopath/src/github.com/hyperledger/fabric/peer
          command: ["/bin/bash", "-c", "--"]
          args: ["while true; do sleep 30; done;"]
          volumeMounts:
           - mountPath: /host/var/run/
             name: run
           - mountPath: /etc/hyperledger/fabric/msp
             name: certificate
             subPath: users/Admin@org1/msp
           - mountPath: /opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
             name: artifacts
      volumes:
        - name: certificate
          persistentVolumeClaim:
              claimName: org1-pv
        - name: artifacts
          persistentVolumeClaim:
              claimName: org1-cli-pv
        - name: run
          hostPath:
            path: /var/run

```


```
# 导入 yaml 文件

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org1-cli-pv-pvc.yaml 
persistentvolume/org1-cli-pv created
persistentvolumeclaim/org1-cli-pv created


[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org1-cli.yaml 
deployment.extensions/cli created

```



```
# 创建 org2  cli 的 pv pvc 


vi org2-cli-pv-pvc.yaml


apiVersion: v1
kind: PersistentVolume
metadata:
    name: org2-cli-pv
    labels:
      app: org2-cli-pv
spec:
    capacity:
       storage: 500Mi
    accessModes:
       - ReadWriteMany
    nfs:
      path: /opt/nfs/fabric/fabric/channel-artifacts
      server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
    namespace: org2
    name: org2-cli-pv
spec:
   accessModes:
     - ReadWriteMany
   resources:
      requests:
        storage: 10Mi
   selector:
     matchLabels:
       app: org2-cli-pv

```


```
# 创建 org2  cli  服务

vi org2-cli.yaml



apiVersion: extensions/v1beta1
kind: Deployment
metadata:
   namespace: org2
   name: cli
spec:
  replicas: 1
  strategy: {}
  template:
    metadata:
      labels:
       app: cli
    spec:
      containers:
        - name: cli
          image: hyperledger/fabric-tools:1.4.0
          env:
          - name: CORE_PEER_TLS_ENABLED
            value: "false"
          - name: CORE_VM_ENDPOINT
            value: unix:///host/var/run/docker.sock
          - name: GOPATH
            value: /opt/gopath
          - name: CORE_LOGGING_LEVEL
            value: DEBUG
          - name: CORE_PEER_ID
            value: cli
          - name: CORE_PEER_ADDRESS
            value: peer0.org2:7051
          - name: CORE_PEER_LOCALMSPID
            value: Org2MSP
          - name: CORE_PEER_MSPCONFIGPATH
            value: /etc/hyperledger/fabric/msp
          - name: GODEBUG
            value: netdns=go
          workingDir: /opt/gopath/src/github.com/hyperledger/fabric/peer
          command: ["/bin/bash", "-c", "--"]
          args: ["while true; do sleep 30; done;"]
          volumeMounts:
           - mountPath: /host/var/run/
             name: run
           - mountPath: /etc/hyperledger/fabric/msp
             name: certificate
             subPath: users/Admin@org2/msp
           - mountPath: /opt/gopath/src/github.com/hyperledger/fabric/peer/channel-artifacts
             name: artifacts
      volumes:
        - name: certificate
          persistentVolumeClaim:
              claimName: org2-pv
        - name: artifacts
          persistentVolumeClaim:
              claimName: org2-cli-pv
        - name: run
          hostPath:
            path: /var/run
```


```
# 导入 yaml 文件

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org2-cli-pv-pvc.yaml 
persistentvolume/org2-cli-pv created
persistentvolumeclaim/org2-cli-pv created



[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl apply -f org2-cli.yaml 
deployment.extensions/cli created

```


```
# 查看服务

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl get pods --all-namespaces

NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
org1          cli-68647b4c-6db4f                         1/1     Running   0          5m12s
org2          cli-5bd96dc66d-bhm65                       1/1     Running   0          18s

```



## 测试 集群


> 测试集群 
>
>1. 初始化 channel  2. 加入 channel  3. 更新为 锚节点 4. 安装 chaincode 5. 实例化 chaincode 6. 操作 chaincode



### org1 节点

```
# 登录  org1 cli

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl exec -it cli-68647b4c-6db4f -n org1 bash



1. 初始化 创建channel

peer channel create -o orderer0.orgorderer1:7050 -c mychannel -f ./channel-artifacts/channel.tx

```



> 拷贝 mychannel.block 文件到 channel-artifacts 共享目录

```
# 拷贝

cp mychannel.block channel-artifacts



# 加入 channel


peer channel join -b channel-artifacts/mychannel.block


2019-01-21 02:40:58.478 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 02:40:58.497 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 02:40:58.500 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-01-21 02:40:58.589 UTC [channelCmd] executeJoin -> INFO 004 Successfully submitted proposal to join channel

```



```
# 更新 锚节点 peer，每个 org 只需执行一次

# 更新对应的 org1 的

peer channel update -o orderer0.orgorderer1:7050 -c mychannel -f ./channel-artifacts/Org1MSPanchors.tx

```



### org2 节点

```
# 登录 org2 节点 cli

[root@kubernetes-1 /opt/jicki/k8s-yaml]# kubectl exec -it cli-5bd96dc66d-j2m6d -n org2 bash

```



```
# 加入 mychannel 

peer channel join -b channel-artifacts/mychannel.block


2019-01-21 02:47:14.529 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 02:47:14.549 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 02:47:14.551 UTC [channelCmd] InitCmdFactory -> INFO 003 Endorser and orderer connections initialized
2019-01-21 02:47:14.629 UTC [channelCmd] executeJoin -> INFO 004 Successfully submitted proposal to join channel



```




```
# 更新 锚节点 peer，每个 org 只需执行一次

# 更新对应的 org2 的

peer channel update -o orderer0.orgorderer1:7050 -c mychannel -f ./channel-artifacts/Org2MSPanchors.tx 


```





### 安装 chaincode

> 安装 chaincode 需要在 org1  org2 的 cli  分别安装


```
#  org1

kubectl exec -it cli-68647b4c-lhdhf -n org1 bash


# org2

kubectl exec -it cli-5bd96dc66d-j2m6d -n org2 bash


```


```
# 分别执行安装

peer chaincode install -n example2 -v 1.0 -p github.com/hyperledger/fabric/peer/channel-artifacts/chaincode/example02


```



### 实例化 chaincode

> 实例化 chaincode 只需要在 其中一个 org 执行就可以

```

peer chaincode instantiate -o orderer0.orgorderer1:7050 \
                             -C mychannel -n example2 -v 1.0 \
                             -c '{"Args":["init","A", "100", "B","200"]}' \
                             -P "OR ('Org1MSP.peer','Org2MSP.peer')"
                             
```


```
# 实例化以后~会在 docker 中生成 一个容器, 脱离于k8s 是由 docker 生成

[root@kubernetes-3 ~]# docker ps -a

CONTAINER ID        IMAGE                                                                                          COMMAND                  CREATED             STATUS                        PORTS               NAMES
26666283f935        dev-peer0.org1-example2-1.0-e8792315af3bc6f98b5be21c4c98ece1ab5c65537e361691af638304c0670cd5   "chaincode -peer.add…"   6 minutes ago       Up 6 minutes                                      dev-peer0.org1-example2-1.0

```



### 操作 chaincode

> 执行 转账，查询 操作

```
# 查询 A, B 账户的值


peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'


# 输出如下:
2019-01-21 03:45:18.469 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 03:45:18.485 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
100



peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'


# 输出如下:
2019-01-21 03:47:21.683 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 03:47:21.701 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
200


```


```
# invoke 转账

# 从A账户 转账 50 个币 到 B 账户

peer chaincode invoke -C mychannel -n example2 -c '{"Args":["invoke", "A", "B", "50"]}'

# 输出如下:
2019-01-21 03:48:43.787 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 03:48:43.806 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 03:48:43.816 UTC [chaincodeCmd] InitCmdFactory -> INFO 003 Retrieved channel (mychannel) orderer endpoint: orderer0.orgorderer1:7050
2019-01-21 03:48:43.830 UTC [chaincodeCmd] chaincodeInvokeOrQuery -> INFO 004 Chaincode invoke successful. result: status:200 



# 转账后在查询 A, B 的值

peer chaincode query -C mychannel -n example2 -c '{"Args":["query","A"]}'

# 输出:
2019-01-21 03:49:10.686 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 03:49:10.702 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
50



peer chaincode query -C mychannel -n example2 -c '{"Args":["query","B"]}'

# 输出:
2019-01-21 03:49:41.883 UTC [main] InitCmd -> WARN 001 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
2019-01-21 03:49:41.898 UTC [main] SetOrdererEnv -> WARN 002 CORE_LOGGING_LEVEL is no longer supported, please use the FABRIC_LOGGING_SPEC environment variable
250


```



# blockchain-explorer 部署

> blockchain-explorer 是 fabric 的区块浏览器


## 配置 postgres 


```
# 创建 数据目录 , 创建于 nfs 中

mkdir -p /opt/nfs/fabric/data/postgres

```


```
# 创建 namesapce

vi explorer-namespace.yaml


apiVersion: v1
kind: Namespace
metadata:
    name: explorer
```




```
# 创建 pv  pvc

vi explorer-pv-pvc.yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
  labels:
    app: postgres-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/data/postgres
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: explorer
 name: postgres-pv
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 10Gi
 selector:
   matchLabels:
     app: postgres-pv


---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: explorer-pv
  labels:
    app: explorer-pv
spec:
  capacity:
    storage: 500Mi
  accessModes:
    - ReadWriteMany
  nfs:
    path: /opt/nfs/fabric/fabric/crypto-config
    server: 192.168.0.249

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 namespace: explorer
 name: explorer-pv
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 100Mi
 selector:
   matchLabels:
     app: explorer-pv

```


```
# 创建 postgres deployment


vi postgres-deployment.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: explorer
  name: postgres-db
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: explorer-db
    spec:
      containers:
      - name: postgres
        image: postgres:10-alpine
        env:
        - name: TZ
          value: "Asia/Shanghai"
        - name: DATABASE_DATABASE
          value: "fabricexplorer"
        - name: DATABASE_USERNAME
          value: "jicki"
        - name: DATABASE_PASSWORD
          value: "password"
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgresdata
      volumes:
       - name: postgresdata
         persistentVolumeClaim:
           claimName: postgres-pv
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-db
  namespace: explorer
  labels:
    run: explorer-db
spec:
 selector:
   name: explorer-db
 ports:
  - protocol: TCP
    port: 5432
```



```
# 查看 服务

[root@kubernetes-1 /opt/jicki/k8s-yaml/explorer]# kubectl get pods -n explorer
NAME                           READY   STATUS    RESTARTS   AGE
postgres-db-7884d8c859-s8hqq   1/1     Running   0          32s


```



```
# 初始化数据库 
# 初始化数据库这边只能手动登录 pods 里操作一下, 导入表结构以及初始化数据。


kubectl get pods -n explorer
NAME                           READY   STATUS    RESTARTS   AGE
postgres-db-7884d8c859-s8hqq   1/1     Running   0          2m33s



kubectl exec -it postgres-db-7884d8c859-s8hqq -n explorer bash



cd /tmp/

wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/createdb.sh
wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/explorerpg.sql
wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/processenv.js
wget https://raw.githubusercontent.com/hyperledger/blockchain-explorer/master/app/persistence/fabric/postgreSQL/db/updatepg.sql


# 安装依赖软件

apk update
apk add jq
apk add nodejs
apk add sudo
rm -rf /var/cache/apk/*

# 执行创建脚本

chmod +x ./createdb.sh
./createdb.sh

# 看出输出 表示完成
You are now connected to database "fabricexplorer" as user "postgres".

退出pods
```






```
# 创建 explorer 配置文件 config.json


vi config.json

{
  "network-configs": {
    "network-1": {
      "version": "1.0",
      "clients": {
        "client-1": {
          "tlsEnable": false,
          "organization": "Org1MSP",
          "channel": "mychannel",
          "credentialStore": {
            "path": "./tmp/credentialStore_Org1/credential",
            "cryptoStore": {
              "path": "./tmp/credentialStore_Org1/crypto"
            }
          }
        }
      },
      "channels": {
        "mychannel": {
          "peers": {
            "peer0.org1": {},
            "peer1.org1": {},
            "peer0.org2": {},
            "peer1.org2": {}
          },
          "connection": {
            "timeout": {
              "peer": {
                "endorser": "6000",
                "eventHub": "6000",
                "eventReg": "6000"
              }
            }
          }
        }
      },
      "organizations": {
        "Org1MSP": {
          "mspid": "Org1MSP",
          "fullpath": false,
          "adminPrivateKey": {
            "path":
              "/fabric/peerOrganizations/org1/users/Admin@org1/msp/keystore"
          },
          "signedCert": {
            "path":
              "/fabric/peerOrganizations/org1/users/Admin@org1/msp/signcerts"
          }
        },
        "Org2MSP": {
          "mspid": "Org2MSP",
          "fullpath": false,
          "adminPrivateKey": {
            "path":
              "/fabric/peerOrganizations/org2/users/Admin@org2/msp/keystore"
          },
          "signedCert": {
            "path":
              "/fabric/peerOrganizations/org2/users/Admin@org2/msp/signcerts"
          }
        },
        "Orgorderer1MSP": {
          "mspid": "Orgorderer1MSP",
          "adminPrivateKey": {
            "path":
              "/fabric/ordererOrganizations/orgorderer1/users/Admin@orgorderer1/msp/keystore"
          }
        }
      },
      "peers": {
        "peer0.org1": {
          "tlsCACerts": {
            "path":
              "/fabric/peerOrganizations/org1/peers/peer0.org1/tls/ca.crt"
          },
          "url": "grpc://peer0.org1:7051",
          "eventUrl": "grpc://peer0.org1:7053",
          "grpcOptions": {
            "ssl-target-name-override": "peer0.org1"
          }
        },
        "peer1.org1": {
          "tlsCACerts": {
            "path":
              "/fabric/peerOrganizations/org1/peers/peer1.org1/tls/ca.crt"
          },
          "url": "grpc://peer1.org1:7051",
          "eventUrl": "grpc://peer1.org1:7053",
          "grpcOptions": {
            "ssl-target-name-override": "peer1.org1"
          }
        },
        "peer0.org2": {
          "tlsCACerts": {
            "path":
              "/fabric/peerOrganizations/org2/peers/peer0.org2/tls/ca.crt"
          },
          "url": "grpc://peer0.org2:7051",
          "eventUrl": "grpc://peer0.org2:7053",
          "grpcOptions": {
            "ssl-target-name-override": "peer0.org2"
          }
        },
        "peer1.org2": {
          "tlsCACerts": {
            "path":
              "/fabric/peerOrganizations/org2/peers/peer1.org2/tls/ca.crt"
          },
          "url": "grpc://peer1.org2:7051",
          "eventUrl": "grpc://peer1.org2:7053",
          "grpcOptions": {
            "ssl-target-name-override": "peer1.org2"
          }
        }
      },
      "orderers": {
        "orderer0.orgorderer1": {
          "url": "grpc://orderer0.orgorderer1:7050"
        }
      }
    },
    "network-2": {}
  },
  "configtxgenToolPath": "/fabric-path/workspace/fabric-samples/bin",
  "license": "Apache-2.0"
}


```


```
# 创建运行脚本 run.sh


vi run.sh


#!/bin/sh
mkdir -p /opt/explorer/app/platform/fabric/

mv /opt/explorer/app/platform/fabric/config.json /opt/explorer/app/platform/fabric/config.json.vanilla
cp /fabric/config.json /opt/explorer/app/platform/fabric/config.json

cd /opt/explorer
node $EXPLORER_APP_PATH/main.js && tail -f /dev/null

```



```
# 拷贝文件到 NFS 目录中 并 授权

cp config.json run.sh /opt/nfs/fabric/fabric/crypto-config/

chmod +x /opt/nfs/fabric/fabric/crypto-config/run.sh

```



```
# 创建 deployment 服务

vi explorer-deployment.yaml


apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  namespace: explorer
  name: explorer
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: explorer
    spec:
      containers:
      - name: explorer
        image: hyperledger/explorer:latest
        command: ["sh" , "-c" , "/fabric/run.sh"]
        env:
        - name: TZ
          value: "Asia/Shanghai"
        - name: DATABASE_HOST
          value: "postgres-db"
        - name: DATABASE_USERNAME
          value: "jicki"
        - name: DATABASE_PASSWORD
          value: "password"
        volumeMounts:
        - mountPath: /fabric
          name: certfile
      volumes:
       - name: certfile
         persistentVolumeClaim:
             claimName: explorer-pv

---
apiVersion: v1
kind: Service
metadata:
  name: explorer
  namespace: explorer
spec:
 selector:
   name: explorer
 ports:
   - name: webui
     protocol: TCP
     port: 8080


---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: explorer-ingress
  namespace: explorer
spec:
  rules:
  - host: explorer.jicki.cn
    http:
      paths:
      - backend:
          serviceName: explorer
          servicePort: 8080

```

![explorer][1]


  [1]: https://jicki.cn/img/posts/fabric/explorer.png

