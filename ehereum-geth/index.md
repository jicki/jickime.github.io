# Ehereum-geth



> 基于 eth 私有链文档 to docker


# 部署私有链


## 创建 创世区块


```
# 创世区块文件说明

{
  "config": {
        //区块链的ID，你随便给一个就可以
        "chainId": 21,
        //下面三个参数暂时不知道干啥的
        "homesteadBlock": 0,
        "eip155Block": 0,
        "eip158Block": 0
    },
  //用来预置账号以及账号的以太币数量，应该也就是所谓的预挖
  //前面是钱包地址，后面为 数量， 必须先创建钱包账号
  //"alloc": {
  //"0x0000000000000000000000000000000000000001": {"balance": "111111111"},
  //"0x0000000000000000000000000000000000000002": {"balance": "222222222"}
  //}
  "alloc"      : {},
  //币基地址，也就是默认的钱包地址，因为我没有地址，所以全0，为空
  //后面运行Geth后创建新账户时，如果Geth发现没有币基地址，会默认将第一个账户的地址设置为币基地址
  //也就是矿工账号
  "coinbase"   : "0x0000000000000000000000000000000000000000",
  //挖矿难度，你可以随便控制.
  "difficulty" : "0x4000",
  //附加信息，随便填个文本或不填也行，类似中本聪在比特币创世块中写的报纸新闻
  "extraData"  : "",
  //gas最高限制，以太坊运行交易，合约等所消耗的gas最高限制，这里设置为最高
  "gasLimit"   : "0xffffffff",
  //64位随机数，用于挖矿，注意他和mixhash的设置需要满足以太坊黄皮书中的要求
  //直接用我这个也可以
  "nonce"      : "0x0000000000000042",
  //与nonce共同用于挖矿，注意他和nonce的设置需要满足以太坊黄皮书中的要求
  "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
  //上一个区块的Hash值，因为是创世块，石头里蹦出来的，没有在它前面的，所以是0
  "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
  //创世块的时间戳，这里给0就好
  "timestamp"  : "0x00"
}

```


## 初始化 init

> 这里geth 环境部署在 docker 中，使用 command 命令初始化


```
# 首先创建一个目录

mkdir -p /opt/blockchain


# 将创世区块文件拷贝到目录下面 如下创世区块文件为 CustomGenesis.json


geth --datadir "/opt/blockchain" init CustomGenesis.json



# 初始化完成生成文件

-rw-r--r-- 1 root root  657 May 11 17:08 CustomGenesis.json
drwxr-xr-x 5 root root 4096 Jul 25 14:14 geth
srw------- 1 root root    0 Jul 25 09:14 geth.ipc
drwx------ 2 root root 4096 May 19 11:56 keystore


```

### 预挖币


```
1. 首先需要创建一个钱包

2. 停止 geth 服务 

3. 删除 blockchain 目录下 geth geth.ipc 文件 保留 keystore 目录 以及 CustomGenesis.json 创世文件

4. 修改 CustomGenesis.json 文件 增加 如下：

  //"alloc": {
  //"0x0000000000000000000000000000000000000001": {"balance": "111111111"},
  //"0x0000000000000000000000000000000000000002": {"balance": "222222222"}
  //}

5. 重新初始化一次

```


## 启动区块服务

> 在未创建钱包时，不能默认启动挖矿

```
# 所有权限
geth --datadir "/opt/blockchain" --nodiscover --identity "myetherum" --miner --minerthreads 1 --rpc --rpcaddr "0.0.0.0" --rpcport "5020" --rpcapi "db,eth,net,web3,admin,miner,personal,rpc" --rpccorsdomain "*" --networkid 9527 console


# 部分权限
geth --datadir "/opt/blockchain" --nodiscover --identity "myetherum" --miner --minerthreads 1 --rpc --rpcaddr "0.0.0.0" --rpcport "5020" --rpcapi "eth,net,web3,rpc" --rpccorsdomain "*" --networkid 9527 console


```

## 操作命令


```
# 连接终端
geth attach http://127.0.0.1:8545


# 查看节点信息
admin.nodeInfo


# 连接其他节点(enode 从 admin.nodeInfo 中查看)
admin.addPeer("enode://64bd4d3bebab176f59c40ac11d2a24dd2f82663f77eb6bf77843f3f48c2505ac34c8eb1bd8cfd53c1290421faee047feb31459bd4231c068efbdbf25accf744d@172.17.0.4:30303")


# 查看连接信息
admin.peers


# 创建钱包
personal.newAccount("123456789")


# 启动挖矿
miner.start()


# 停止挖矿
miner.stop()


# 查看主钱包 币数量
web3.fromWei(eth.getBalance(eth.coinbase), "ether")


# 查看节点下的钱包账户
eth.accounts


# 查看交易所需gas
eth.gasPrice

```


## 智能合约

> 在线编辑合约 http://remix.ethereum.org/

```
# 进入终端 必须有 personal 权限，用于解锁账号

1. 开启挖矿，创建合约必须开启挖矿


2. 需要解锁主钱包账号 
   personal.unlockAccount(eth.coinbase,'钱包密码')


3. 复制 SOL 文件生成的合约文件到终端。

```


## docker-compose.yaml


> docker-compose 直接启动区块

```
version: '2'
services:
  ethereum:
    image: ethereum/client-go:alltools-latest
    hostname: ethereum
    container_name: ethereum
    restart: always
    ports:
    - "8545:8545"
    volumes:
    - /opt/blockchain:/opt/blockchain
    #command: geth --datadir "/opt/blockchain" init /opt/blockchain/CustomGenesis.json
    #command: geth --datadir "/opt/blockchain" --identity "oboetherum" --mine --minerthreads 1 --rpc --rpcaddr "0.0.0.0" --rpcport "8545" --rpcapi "admin,miner,personal,eth,net,web3,rpc" --rpccor
sdomain "*" --networkid 62222
    command: geth --datadir "/opt/blockchain" --identity "oboetherum" --mine --minerthreads 1 --rpc --rpcaddr "0.0.0.0" --rpcport "8545" --rpcapi "eth,net,web3,rpc" --rpccorsdomain "*" --networki
d 62222
```

