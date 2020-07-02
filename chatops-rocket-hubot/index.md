# ChatOps RocketChat Hubot


# ChatOps

**什么是ChatOps**

* ChatOps 以 IM（沟通平台）为中心，通过一系列的机器人去对接后台的各种服务，我们只需在聊天窗口中与机器人对话，即可与后台服务进行交互，整个工作的展开就像是使唤一个智能助手那样简单自然。

![图1][1]

**ChatOps的好处**

* 方便简单友好。只需要在IM前台与预设好的机器人对话即可完成与后台工具、系统的交互，在移动环境下无须再与众多复杂的工具直接对接，大大提升移动办公的可行性。

* DevOps文化。用与机器人对话这种简单的方式降低 DevOps 的接受门槛，让这种自动化办公的理念更容易地扩展到团队的每一个角落。

* 公开透明。所有的工作消息都在同一个聊天平台中沉淀并公开给所有相关成员，可以消除沟通壁垒，工作历史有迹可循，团队合作更加顺畅。

* 上下文共享。减少因工作台切换等对消息的截断，保证消息的完整性，让工作承接有序，各角色，各工具都成为完成工作流中的一环，打造真正流畅的工作体验。
  

**ChatOps 的实践经验**

* ChatOps 主要由四个部分组成：自动化的理念、一个沟通承载平台、一系列连接人与工具的机器人，以及一些后台工具和服务（基础设施）。它不仅可以应用在技术团队中，还可以发展为适应不同种类团队的方法模型，这也是 ChatOps 这个概念提出的背景之一。随着全行业的发展和人力成本的攀升，ChatOps 也可以说是应用于全行业的 DevOps。


**GitHub 工作**

* Github 的员工 60% 都是在家里远程办公的， 新员工入职培训就是查看老员工跟其他人的聊天信息，感受别人是如何进行工作。

* Github 使用了 机器人执行大部分的运维脚本，将各种重复、以及繁琐的工作交给机器人去处理。

* 在聊天工具(IM)中和同事聊天、查看运维信息、查看监控等, 所有相关人员都可以看到具体的情况。

* 聊天工具(IM)中保留了相关员工的操作记录，执行了那些操作，做过什么事情，都会记录下来。



## hubot

> hubot 是github 开源的一款 机器人
>
> https://hubot.github.com/


`docker`


```
部署 docker 略 ..
```




`RocketChat`

> RocketChat 是一款开源的 聊天室
>
> 官方 github https://github.com/RocketChat/Rocket.Chat
> 
> 官方主页 https://rocket.chat/


```
vi docker-compose.yaml


version: '2'
services:
  mongo:
    image: mongo:4
    hostname: mongo
    container_name: mongo
    restart: always
    volumes:
    - ./data/rocket/mongo/data:/data/db:z
    command: mongod --smallfiles --oplogSize 1024 --replSet rs1 --storageEngine=mmapv1

  rocket:
    image: rocketchat/rocket.chat
    hostname: rocket
    container_name: rocket
    restart: always
    environment:
    - DEPLOY_METHOD=docker
    - NODE_ENV=production
    - MONGO_URL=mongodb://mongo:27017/rocketchat
    - MONGO_OPLOG_URL=mongodb://mongo:27017/local?replSet=rs01
    - HOME=/tmp
    - PORT=3000
    - ROOT_URL=http://localhost:3000
    - Accounts_AvatarStorePath=/app/uploads
    volumes:
    - ./data/rocket/uploads:/app/uploads
    ports:
    - "3000:3000"
    depends_on:
      - mongo

```

```
# 这里先启动一下 mongo


docker-compose up -d mongo



# 这里mongo 还需要执行一些设置

docker exec -d mongo bash -c 'echo -e "replication:\n  replSetName: \"rs01\"" | tee -a /etc/mongod.conf && mongo --eval "printjson(rs.initiate())"'


```


```
# 启动 rocketchat

docker-compose up -d rocket


```

```
# 查看 rocketchat 的 logs 确保没报错

docker logs rocket


➔ System ➔ startup
➔ +----------------------------------------------+
➔ |                SERVER RUNNING                |
➔ +----------------------------------------------+
➔ |                                              |
➔ |  Rocket.Chat Version: 1.2.1                  |
➔ |       NodeJS Version: 8.11.4 - x64           |
➔ |      MongoDB Version: 4.0.10                 |
➔ |       MongoDB Engine: mmapv1                 |
➔ |             Platform: linux                  |
➔ |         Process Port: 3000                   |
➔ |             Site URL: http://localhost:3000  |
➔ |     ReplicaSet OpLog: Enabled                |
➔ |          Commit Hash: 7475d7628a             |
➔ |        Commit Branch: HEAD                   |
➔ |                                              |
➔ +----------------------------------------------+

```


```
# 配置 rocket ，增加 hubot 机器人

# 增加 hubot 机器人
```


![图4][2]

![图4][3]


```
# 在 rocket-chat 中默认已经有一个 rocket.cat 的机器人账号
# 只需要修改一下密码就可以
```



```
# 创建 hubot-rocketchat

vi docker-compose.yaml


  hubot:
    image: rocketchat/hubot-rocketchat
    hostname: hubot
    container_name: hubot
    restart: always
    environment:
    - ROCKETCHAT_URL=http://192.168.168.11:3000
    - ROCKETCHAT_ROOM=''
    - LISTEN_ON_ALL_PUBLIC=true
    - ROCKETCHAT_USER=rocket.cat
    - ROCKETCHAT_PASSWORD=123456
    #- ROCKETCHAT_AUTH=password
    - BOT_NAME=rocket.cat
    # Scrpits 可到这里查看 https://www.npmjs.com/search?q=keywords:hubot-scripts
    - EXTERNAL_SCRIPTS=hubot-pugme,hubot-help
    volumes:
    # Scrpits 目录，自己编写的脚本放于此处既可
    - ./data/rocket/scripts:/home/hubot/scripts

```


```
# 启动机器人

docker-compose up -d hubot

```


```
# 编辑信息

cd ./data/rocket/scripts

# 里面有太多例子了，清除掉只留下如下:

vi example.coffee

module.exports = (robot) ->
  robot.catchAll (res) ->
    res.send "对不起，我不认识这条命令，请查看 help"
  robot.hear /hi/i, (res) ->
    res.reply "hello"
  robot.hear /吃饭了吗/, (res) ->
    res.send '猫只吃鱼'

```

```
# 修改了任何脚本都需要重启机器人

docker restart hubot

```


`简单测试:`


![图4][4]




## 添加 hubot 插件

> hubot-script-shellcmd 是执行 shell 命令的一个插件
>
> 插件官方 github https://github.com/coderofsalvation/hubot-script-shellcmd


```
# 在 docker-compose.yaml 的 EXTERNAL_SCRIPTS 中增加
hubot-script-shellcmd
# 将这个路径挂载到我们本地 方便修改


    environment:
    - EXTERNAL_SCRIPTS=hubot-pugme,hubot-help,hubot-script-shellcmd
    volumes:
    - ./data/rocket/bash:/home/hubot/bash  

```


```
# 创建一下目录

mkdir -p ./data/rocket/bash/handlers


# 增加完毕以后重启 hubot 机器人

docker restart hubot


# 重启以后，将生成的 bash 文件拷贝到我们的目录
# 这里一定要拷贝，否则执行会卡住


cd ./data/rocket/bash

docker cp hubot:/home/hubot/node_modules/hubot-script-shellcmd/bash/handler .

cd ./data/rocket/bash/handlers

docker cp hubot:/home/hubot/node_modules/hubot-script-shellcmd/bash/handlers/helloworld .

docker cp hubot:/home/hubot/node_modules/hubot-script-shellcmd/bash/handlers/update .

```


`测试shellcmd`

![图5][5]


`创建 bash`

```
cd ./data/rocket/bash/handlers

# 新建一个 bash ,并且授权必须有执行权限


vi date


#!/bin/bash

time=$(date "+%Y-%m-%d %H:%M:%S")

echo "${time}"

exit 0

```

![图6][6]


```
# 创建脚本不需要重启 hubot 机器人

```



`操作docker`

> 基于 shellcmd 插件，我们可以操作一下 docker



```
# 添加如下挂载, 并且这里镜像中运行用户必须使用 root

    user: root

    volumes:
    - /usr/bin/docker:/usr/bin/docker
    - /var/run/docker.sock:/var/run/docker.sock:z

```

```
# 添加 一个 bash

cd ./data/rocket/bash/handlers

vi dockerversion

#!/bin/bash

docker version

exit 0

```


![图7][7]


  [1]: https://jicki.cn/img/posts/chatops/1.png
  [2]: https://jicki.cn/img/posts/chatops/2.png
  [3]: https://jicki.cn/img/posts/chatops/3.png 
  [4]: https://jicki.cn/img/posts/chatops/4.png 
  [5]: https://jicki.cn/img/posts/chatops/5.png 
  [6]: https://jicki.cn/img/posts/chatops/6.png 
  [7]: https://jicki.cn/img/posts/chatops/7.png 

