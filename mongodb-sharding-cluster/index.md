# mongodb sharding cluster




# mongodb sharding cluster


## 环境说明

分别在3台机器运行一个mongod实例（称为mongod shard11，mongod shard12，mongod shard13）组织replica set1，作为cluster的shard1

分别在3台机器运行一个mongod实例（称为mongod shard21，mongod shard22，mongod shard23）组织replica set2，作为cluster的shard2

每台机器运行一个mongod实例，作为3个config server

每台机器运行一个mongs进程，用于客户端连接


```

Server1  10.3.0.100      

Mongod shard11:27017

Mongod shard21:17017

Mongod config1:20000

Mongos1:20001

```


```
Server1 10.3.0.101      

Mongod shard12:27017

Mongod shard22:17017

Mongod config1:20000

Mongos1:20001

```

```
Server3 10.3.0.102      

Mongod shard13:27017

Mongod shard23:17017

Mongod config1:20000

Mongos1:20001

```


## 安装 mongodb


1. 创建用户 mongodb (如果用root启用mongodb,可略此步)

```
useradd -u 1001 mongodb
```
 

2. 下载 mongodb-linux-x86_64-2.2.0

```
tar zxvf mongodb-linux-x86_64-2.2.0.tgz

mv mongodb-linux-x86_64-2.2.0 /opt/local/mongodb
```
 

3. 创建 数据目录 日志目录 配置目录 (server 1)

```
mkdir -p /opt/local/mongodb/data/shard

mkdir -p /opt/local/mongodb/data/logs

mkdir -p /opt/local/mongodb/data/config
```
 

4. 创建 数据目录 日志目录 配置目录 (server 2)

```
mkdir -p /opt/local/mongodb/data/shard12

mkdir -p /opt/local/mongodb/data/shard22

mkdir -p /opt/local/mongodb/data/logs

mkdir -p /opt/local/mongodb/data/config
```
 

5. 创建 数据目录 日志目录 配置目录 (server 3)

```
mkdir -p /opt/local/mongodb/data/shard13

mkdir -p /opt/local/mongodb/data/shard23

mkdir -p /opt/local/mongodb/data/logs

mkdir -p /opt/local/mongodb/data/config
```
 

6. 修改 目录 所有者

```
chown -R mongodb:mongodb /opt/local/mongodb
```



7.  创建配置文件

```
vi mongodb.conf


port=27017                                                  #端口号

fork=true                                                   #以守护进程的方式运行，创建服务器进程

logpath=/opt/local/mongodb/data/logs/shard.log              #日志输出文件路径

logappend=true                                              #日志输出方式

dbpath=/opt/local/mongodb/data/shard/                       #数据库路径

shardsvr=true                                               #设置是否分片

maxConns=10000                                              #数据库的最大连接数

replSet=shard1                                              #设置副本集名称

oplogSize=5000                                              #设置oplog的大小(MB)


```
 

## 启动 mongodb 27017


server 1

```
/opt/local/mongodb/bin/mongod --shardsvr --maxConns 10000 --replSet shard1 --port 27017 --dbpath /opt/local/mongodb/data/shard11/ --oplogSize 500 --logpath=/opt/local/mongodb/data/logs/shard11.log --logappend --fork
```


server 2

```
/opt/local/mongodb/bin/mongod --shardsvr --maxConns 10000 --replSet shard1 --port 27017 --dbpath /opt/local/mongodb/data/shard12/ --oplogSize 500 --logpath=/opt/local/mongodb/data/logs/shard12.log --logappend --fork
```
 

server 3

```
/opt/local/mongodb/bin/mongod --shardsvr --maxConns 10000 --replSet shard1 --port 27017 --dbpath /opt/local/mongodb/data/shard13/ --oplogSize 500 --logpath=/opt/local/mongodb/data/logs/shard13.log --logappend --fork
```
 

 
## 初始化replica set shard1

用mongo连接其中一个mongod，执行:

```
/opt/local/mongodb/bin/mongo --host 10.3.0.100:27017
```

```
> config= {_id: 'shard1', members: [ {_id:0,host:'10.3.0.100:27017'},

... {_id:1,host:'10.3.0.101:27017'},

... {_id:2,host:'10.3.0.102:27017'},]

... }

{...}
> rs.initiate(config);

{

"info" : "Config now saved locally. Should come online in about a minute.",

"ok" : 1

}

```

## 启动 mongodb 17017

server `

```
/opt/local/mongodb/bin/mongod --shardsvr --maxConns 10000 --replSet shard2 --port 17017 --dbpath /opt/local/mongodb/data/shard21/ --oplogSize 100 --logpath=/opt/local/mongodb/data/logs/shard21.log --logappend --fork
```
 

 

server 2

```
/opt/local/mongodb/bin/mongod --shardsvr --maxConns 10000 --replSet shard2 --port 17017 --dbpath /opt/local/mongodb/data/shard22/ --oplogSize 100 --logpath=/opt/local/mongodb/data/logs/shard22.log --logappend --fork
```
 

 

server 3

```
/opt/local/mongodb/bin/mongod --shardsvr --maxConns 10000 --replSet shard2 --port 17017 --dbpath /opt/local/mongodb/data/shard23/ --oplogSize 100 --logpath=/opt/local/mongodb/data/logs/shard23.log --logappend --fork
```
 

 

## 初始化replica set shard2


用mongo连接其中一个mongod，执行:

```
/opt/local/mongodb/bin/mongo --host 10.3.0.100:17017
```

```
> config= {_id: 'shard2', members: [ {_id:0,host:'10.3.0.100:17017'},

... {_id:1,host:'10.3.0.101:17017'},

... {_id:2,host:'10.3.0.102:17017'},]

... }

{...}

> rs.initiate(config);

{

"errmsg" : "couldn't initiate : set name does not match the set name host 10.3.0.101:17017 expects",

"ok" : 0

}

>

```

## 启动 config server



server 1

```
/opt/local/mongodb/bin/mongod --configsvr --dbpath /opt/local/mongodb/data/config/ --port 20000 --logpath /opt/local/mongodb/data/logs/config.log --logappend --fork
```
 

server 2

```
/opt/local/mongodb/bin/mongod --configsvr --dbpath /opt/local/mongodb/data/config/ --port 20000 --logpath /opt/local/mongodb/data/logs/config.log --logappend --fork
```
 

server 3

```
/opt/local/mongodb/bin/mongod --configsvr --dbpath /opt/local/mongodb/data/config/ --port 20000 --logpath /opt/local/mongodb/data/logs/config.log --logappend --fork
```
 

## 配置 mongs

在server1,server2,server3 上分别执行：

```
/opt/local/mongodb/bin/mongos --configdb 10.3.0.100:20000,10.3.0.101:20000,10.3.0.102:20000 --port 20001 --chunkSize 5 --logpath /opt/local/mongodb/data/logs/mongos.log --logappend --fork
```

Configuring the Shard Cluster

连接到其中一个mongos进程，并切换到admin数据库做以下配置

连接到mongs，并切换到admin

```
/opt/local/mongodb/bin/mongo 10.3.0.100:20001/admin

MongoDB shell version: 2.2.0

connecting to: 10.3.0.100:20001/admin

mongos> db

admin

```
 

 

## 加入shards


如里shard是单台服务器:

```
db.runCommand( { addshard : “<serverhostname>[:<port>]” } ) 
```
 

如果shard是replica sets:

```
db.runCommand( { addshard : “replicaSetName/<serverhostname>[:port]” } )
```



shard 1

```
db.runCommand( { addshard : "shard1/10.3.0.100:27017,10.3.0.101:27017,10.3.0.102:27017", maxsize:204800});

{ "shardAdded" : "shard1", "ok" : 1 }

```

 

shard 2

```
mongos> db.runCommand( { addshard : "shard2/10.3.0.100:17017,10.3.0.101:17017,10.3.0.102:17017", maxsize:204800});

{ "shardAdded" : "shard2", "ok" : 1 }
```

 
## 验证 shards


```
mongos> db.runCommand( { listshards : 1 } )

{

"shards" : [

{

"_id" : "shard1",

"host" : "shard1/10.3.0.100:27017,10.3.0.101:27017,10.3.0.102:27017"

},

{

"_id" : "shard2",

"host" : "shard2/10.3.0.100:17017,10.3.0.101:17017,10.3.0.102:17017"

}

],

"ok" : 1

}
```


## 删除 shards

```
db.runCommand( { removeshard : "shard1/10.3.0.100:27017,10.3.0.101:27017"} );
```
 

 

## 激活分片

命令：

```
> db.runCommand( { enablesharding : “<dbname>” } );
```


通过执行以上命令，可以让数据库跨shard，如果不执行这步，数据库只会存放在一个shard，一旦激活数据库分片，数据库中不同的collection将被存放在不同的shard上，但一个collection仍旧存放在同一个shard上，要使单个collection也分片，

还需单独对collection作些操作



## 增添节点:

以一个新建的mongodb服务为例,

连接 mongo

```
/opt/local/mongodb/bin/mongo --host 10.3.0.100:27017

MongoDB shell version: 2.2.0

connecting to: 10.3.0.100:27017/test

shard1:PRIMARY>rs.add(“IP:port”);
```

## 删除节点：

必须在主节点上操作：

```
PRIMARY>rs.remove(“127.0.0.1:27020”)
```


查看同步状态

```
rs.status()
```

从有读的权限：

在主控上执行:

```
PRIMARY> db.getMongo().setSlaveOk();
```

```
SECONDARY> rs.slaveOk();
```

## 增加验证 auth


群集如果加了 auth 验证，群集之前互相取不到 主 就无法验证...必须要增加 keyFile 验证才行...

先创建auth 验证


```
> use admin

switched to db admin

> db.addUser('sa','sa')

{

"_id" : ObjectId("4e2914a585178da4e03a16c3"),

"user" : "sa",

"readOnly" : false,

"pwd" : "75692b1d11c072c6c79332e248c4f699"

}

>
```


然后在每个server里创建 key 文件....

```
echo "1234567890111111111" /opt/local/mongodb/data/config/key

chown mongodb:mongodb key

chmod 600 key
```


然后分别启动 mongod

server 1

```
/opt/local/mongodb/bin/mongod --shardsvr --replSet shard1 --port 27017 --dbpath /opt/local/mongodb/data/shard11/ --oplogSize 100 --logpath=/opt/local/mongodb/data/logs/shard11.log --logappend --fork --keyFile /opt/local/mongodb/data/config/key
```
 

server 2

```
/opt/local/mongodb/bin/mongod --shardsvr --replSet shard1 --port 27017 --dbpath /opt/local/mongodb/data/shard12/ --oplogSize 100 --logpath=/opt/local/mongodb/data/logs/shard12.log --logappend --fork --keyFile /opt/local/mongodb/data/config/key
```
 

server 3

```
/opt/local/mongodb/bin/mongod --shardsvr --replSet shard1 --port 27017 --dbpath /opt/local/mongodb/data/shard13/ --oplogSize 100 --logpath=/opt/local/mongodb/data/logs/shard13.log --logappend --fork --keyFile /opt/local/mongodb/data/config/key
```


再启动 config

server 1

```
/opt/local/mongodb/bin/mongod --configsvr --dbpath /opt/local/mongodb/data/config/ --port 20000 --logpath /opt/local/mongodb/data/logs/config.log --logappend --fork --keyFile /opt/local/mongodb/data/config/key
```
 

server 2

```
/opt/local/mongodb/bin/mongod --configsvr --dbpath /opt/local/mongodb/data/config/ --port 20000 --logpath /opt/local/mongodb/data/logs/config.log --logappend --fork --keyFile /opt/local/mongodb/data/config/key
```
 

server 3

```
/opt/local/mongodb/bin/mongod --configsvr --dbpath /opt/local/mongodb/data/config/ --port 20000 --logpath /opt/local/mongodb/data/logs/config.log --logappend --fork --keyFile /opt/local/mongodb/data/config/key
```


最后分别启动mongos

```
/opt/local/mongodb/bin/mongos --configdb 10.3.0.100:20000,10.3.0.101:20000,10.3.0.102:20000 --port 20001 --chunkSize 5 --logpath /opt/local/mongodb/data/logs/mongos.log --logappend --fork --keyFile /opt/local/mongodb/data/config/key
```

最后验证 auth

```
./mongo

MongoDB shell version: 2.2.0

connecting to: test

> show dbs

Wed Sep 5 01:51:44 uncaught exception: listDatabases failed:{ "errmsg" : "need to login", "ok" : 0 }

> use admin

switched to db admin

> db.auth('sa','sa')

1

shard1:SECONDARY> db.serverStatus()
```


注意:

添加auth 以后...导入数据 等一系列操作...都必须验证用户...

如：

```
/opt/local/mongodb/bin/mongorestore -u sa -p sa --drop /opt/1
```
 
其他都需要进行操作！否则！报错！！！



## 强制切换主

```
cfg = rs.conf()

cfg.members[0].priority = 0.5

cfg.members[1].priority = 0.5

cfg.members[2].priority = 1

rs.reconfig(cfg)
```
 

## 限制 mongodb 占用内存

```
ulimit -s 4096 && ulimit -m 31457280 && sudo -u mongodb numactl --interleave=all /opt/local/mongodb/bin/mongod --shardsvr --maxConns 10000 --replSet shard1 --port 27017 --dbpath /opt/local/mongodb/data/shard16/ --oplogSize 1000 --logpath=/opt/local/mongodb/data/logs/shard16.log --logappend --fork
```







## 数据库命令

显示当前数据库服务器上的数据库

```
show dbs
```

切换到指定数据库pagedb的上下文，可以在此上下文中管理pagedb数据库以及其中的集合等

```
use pagedb
```
 

显示数据库中所有的集合（collection）

```
show collections
```


查看数据库服务器的状态。

```
db.serverStatus()  
```


查询指定数据库统计信息

```
use fragment
db.stats()
```
 

查询指定数据库包含的集合名称列表

```
db.getCollectionNames()
```
 
 
复制数据库

```
db.copyDatabase("111","222") 
```


删除数据库

```
db.dropDatabase()
```
 
创建集合

```
db.createCollection(name, { size : ..., capped : ..., max : ... } )
```
 

删除集合

```
db.mycoll.drop()
```
 


查询一条记录

使用findOne()函数，参数为查询条件，可选，系统会随机查询获取到满足条件的一条记录（如果存在查询结果数量大于等于1）示例如下所示：

```
db.storeCollection.findOne({'version':'3.5'})  

{  

        "_id" : ObjectId("4ef970f23c1fc4613425accc"),  

        "version" : "3.5",  

        "segment" : "e3ol6"  

}  
```
 

创建索引

可以使用集合的ensureIndex(keypattern[,options])方法，

示例如下所示：

```
> use pagedb  

switched to db pagedb  

> db.page.ensureIndex({'title':1, 'url':-1})  

> db.system.indexes.find()  

{ "name" : "_id_", "ns" : "pagedb.page", "key" : { "_id" : 1 }, "v" : 0 }  

{ "name" : "_id_", "ns" : "pagedb.system.users", "key" : { "_id" : 1 }, "v" : 0}  

{ "_id" : ObjectId("4ef977633c1fc4613425accd"), "ns" : "pagedb.page", "key" : {"title" : 1, "url" : -1 }, "name" : "title_1_url_-1", "v" : 0 }  
```

上述，ensureIndex方法参数中，数字1表示升序，-1表示降序。

查询全部索引

```
db.system.indexes.find()
```

 
删除索引 

删除索引给出了两个方法：
第一个通过指定索引名称，第二个删除指定集合的全部索引。

```
db.mycoll.dropIndex(name)  

db.mycoll.dropIndexes()  
```


索引重建

可以通过集合的reIndex()方法进行索引的重建

示例如下所示：


```
> db.page.reIndex()  

{  

        "nIndexesWas" : 2,  

        "msg" : "indexes dropped for collection",  

        "ok" : 1,  

        "nIndexes" : 2,  

        "indexes" : [  

                {  

                        "name" : "_id_",  

                        "ns" : "pagedb.page",  

                        "key" : {  

                                "_id" : 1  

                        },  

                        "v" : 0  

                },  

                {  

                        "_id" : ObjectId("4ef977633c1fc4613425accd"),  

                        "ns" : "pagedb.page",  

                        "key" : {  

                                "title" : 1,  

                                "url" : -1  

                        },  

                        "name" : "title_1_url_-1",  

                        "v" : 0  

                }  

        ],  

        "ok" : 1  

}  
```
 

统计集合记录数

```
use fragment

db.baseSe.count()
```


查询指定数据库的集合当前可用的存储空间

```
use fragment

> db.baseSe.storageSize()

142564096
```
 

查询指定数据库的集合分配的存储空间

```
> db.baseSe.totalSize()

144096000
```


## 运维命令


1、备份全部数据库

```
mkdir testbak

cd testbak

mongodump
```

说明：默认备份目录及数据文件格式为./dump/[databasename]/[collectionname].bson


备份指定数据库

```
mongodump -d pagedb
```

说明：备份数据库pagedb中的数据。



备份一个数据库中的某个集合

```
mongodump -d pagedb -c page
```

说明：备份数据库pagedb的page集合。



恢复全部数据库

```
cd testbak

mongorestore --drop  /opt/dump
```

说明：将备份的所有数据库恢复到数据库，--drop指定恢复数据之前删除原来数据库数据，否则会造成回复后的数据中数据重复。




恢复某个数据库的数据

```
cd testbak

mongorestore -d pagedb --drop /opt/dump/pagedb
```

说明：将备份的pagedb的数据恢复到数据库。



恢复某个数据库的某个集合的数据

```
cd testbak

mongorestore -d pagedb -c page --drop
```

说明：将备份的pagedb的的page集合的数据恢复到数据库。


从MongoDB导出数据

```
mongoexport -d pagedb -c page -q {} -f _id,title,url,spiderName,pubDate --csv > pages.csv
```


说明：将pagedb数据库中page集合的数据导出到pages.csv文件，其中各选项含义：

-f 指定cvs列名为_id,title,url,spiderName,pubDate

-q 指定查询条件

注意：

如果上面的选项-q指定一个查询条件，需要使用单引号括起来，如下所示：

```
mongoexport -d page -c Article -q '{"spiderName": "mafengwoSpider"}' -f _id,title,content,images,publishDate,spiderName,url --jsonArray > mafengwoArticle.txt 
```



向MongoDB导入数据

```
mongoimport -d pagedb -c page --type csv --headerline --drop < csvORtsvFile.csv
```

说明：将文件csvORtsvFile.csv的数据导入到pagedb数据库的page集合中，使用cvs或tsv文件的列名作为集合的列名。需要注意的是，使用--headerline选项时，只支持csv和tsv文件。

--type支持的类型有三个：csv、tsv、json

