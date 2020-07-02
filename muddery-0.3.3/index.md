# Muddery 部署 配置文档



# Muddery 简介


> 程序源代码：https://github.com/muddery/muddery
>
> Muddery是一个用Python编写的在线文字游戏（如MUD）框架，所有的代码都是开源的，采用BSD许可证发布。它使用Evennia（一个MUD游戏框架）作为其内核。

## Muddery 特点

* Muddery具有以下特点:

 1. 使用Python开发，可以跨平台使用，只需要花几分钟时间就能够安装它。
 2. 支持多人在线游戏，游戏内容主要以文字形式展现，但也可以扩展加入多媒体的内容。
 3. 内建有基本的任务系统、事件系统、对话系统等，便于游戏的创建。
 4. 自带有网页版的游戏编辑器，可以在网页上构建游戏世界。
 5. 自带网页客户端，可以轻松地发布游戏。
 6. 完全使用点击式的游戏操作模式，便于在智能手机、平板设备上使用。
 

# Muddery 安装部署


## 安装所需软件

> 这里使用 CentOS 7.x 64 系统



```
1. # Python , CentOS 默认都会自带 python 2.7

2. 安装 pip

curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py

python get-pip.py


3. 安装 git

yum -y install git


4. 安装 python-devel

yum -y install python-devel


5. 安装 virtualenv

pip install virtualenv

```

## 配置 virtualenv


```
# 创建 虚拟环境目录

mkdir -p /opt/muddery

# 创建 一个 虚拟环境

virtualenv pyenv


# 如果 系统安装了 python 3.x 版本 请使用 使用 2.7 版本来创建 virtualenv 环境

virtualenv -p /usr/bin/python2 pyenv



# 创建虚拟环境以后会生成 pyenv 目录


# 激活/进入 虚拟环境

source pyenv/bin/activate


# 激活以后~会变成:   

(pyenv) [root@localhost muddery]# 

```



## 配置 muddery


```
# 下载源码

(pyenv) [root@localhost muddery]# git clone https://github.com/muddery/muddery.git


# 进入目录

(pyenv) [root@localhost muddery]# cd /opt/muddery/muddery



# 下载 muddery 所需要的依赖

(pyenv) [root@localhost muddery]# pip install -e .


# 相关依赖在 requirements.txt 文件中

```



## 初始化 模板



```
# muddery 支持创建 英文 与 中文 版本的模板


# 进入我们的目录

cd /opt/muddery

# 创建一个模板

# 英文版 ( mygame 为 模板目录 )

muddery --init mygame


# 中文版 ( mygame 为 模板目录  cn 为 中文 )

muddery --init mygame cn

```


```
# 模板配置文件在 mygame/server/conf/settings.py 中

# 配置文件中可以修改 启动端口, 语言 等参数


# 服务器内部管理端口 admin web-ui
WEBSERVER_PORTS = [(8000, 5001)]

# websocket 端口, 用于 webclient 连接
WEBSOCKET_CLIENT_PORT = 8001

# AMP 连接端口
AMP_PORT = 5000

# 语言设置, en 与 cn
LANGUAGE_CODE = 'zh-cn'

```


```

# 修改数据库 Muddery 默认使用Sqlite3数据库。

# Sqlite3 数据库只适用于测试， 修改为 mysql 或者其他

# 编辑 mygame/server/conf/settings.py  文件


vi mygame/server/conf/settings.py

# 在 最下面 添加 如下

######################################################################
# Evennia Database config
######################################################################

# Database config syntax:
# ENGINE - path to the the database backend. Possible choices are:
#            'django.db.backends.sqlite3', (default)
#            'django.db.backends.mysql',
#            'django.db.backends.postgresql_psycopg2' (see Issue 241),
#            'django.db.backends.oracle' (untested).
# NAME - database name, or path to the db file for sqlite3
# USER - db admin (unused in sqlite3)
# PASSWORD - db admin password (unused in sqlite3)
# HOST - empty string is localhost (unused in sqlite3)
# PORT - empty string defaults to localhost (unused in sqlite3)
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mygame',
        'USER': 'root',
        'PASSWORD': '123456789',
        'HOST': '127.0.0.1',
        'PORT': '3306'
        },
    'worlddata': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'mygame_worlddata',
        'USER': 'root',
        'PASSWORD': '123456789',
        'HOST': '127.0.0.1',
        'PORT': '3306'
        }}
        
# Database's router
DATABASE_ROUTERS = ['muddery.worlddata.db.database_router.DatabaseAppsRouter']

DATABASE_APPS_MAPPING = {
    'worlddata': 'worlddata',
}

```






```
# 安装 django 的 mysql 库

# 首先安装 mysql 支持


yum install mysql-devel



# 安装 python mysql 库 

pip install mysql-python




DEPRECATION: Python 2.7 will reach the end of its life on January 1st, 2020. Please upgrade your Python as Python 2.7 won't be maintained after that date. A future version of pip will drop support for Python 2.7.
Collecting mysql-python
  Using cached https://files.pythonhosted.org/packages/a5/e9/51b544da85a36a68debe7a7091f068d802fc515a3a202652828c73453cad/MySQL-python-1.2.5.zip
Building wheels for collected packages: mysql-python
  Building wheel for mysql-python (setup.py) ... done
  Stored in directory: /root/.cache/pip/wheels/07/d2/5f/314860e4cb53a44bf0ee0d051d4b34465e4b4fbe9de6d42f42
Successfully built mysql-python
Installing collected packages: mysql-python
Successfully installed mysql-python-1.2.5




# 登录 mysql 创建 两个数据库 mygame 与 mygame_worlddata


mysq -h 127.0.0.1 -uroot -p123456


MySQL [(none)]> create database mygame;
Query OK, 1 row affected (0.00 sec)



MySQL [(none)]> create database mygame_worlddata;
Query OK, 1 row affected (0.00 sec)





# 初始化 数据库 的表结构

(pyenv) [root@payment mygame]# cd /opt/mudgame/mygame


# 创建 数据库 与 基础数据
(pyenv) [root@payment mygame]# muddery --loaddata

Operations to perform:
  Apply all migrations: accounts, admin, auth, comms, contenttypes, database, flatpages, help, objects, scripts, server, sessions, sites, typeclasses, worlddata

....

Import local data success.
```





## 启动游戏

```
# 进入刚才的游戏模板目录

cd /opt/muddery/mygame


# 执行启动 (初次启动 会让 创建一个 超级管理员 )

(pyenv) [root@payment mygame]# muddery -i start

Create a superuser below. The superuser is Account #1, the 'owner' account of the server.


Username: 
Email address: 
Password: 
Password (again): 
Superuser created successfully.
Portal starting ...
... Portal started.
Server starting  ...
... Server started.
Evennia running.
----------------------- Evennia ---
Muddery Portal 0.8.0 (rev bab4b86)
    external ports:
        webserver-proxy: 8000
        webclient-websocket: 8001
    internal_ports (to Server):
        webserver: 5001
        amp: 5000

Muddery Server 0.8.0 (rev bab4b86)
    internal ports (to Portal):
        webserver: 5001
        amp : 5000
-----------------------------------

```



## 登录 web-ui


```
# 浏览器 访问

http://127.0.0.1:8000/


# 输入 第一次 muddery -i start 时 创建的 管理员账号密码

```
![muddery-web][1]




```
# 游戏编辑器

http://127.0.0.1:8000/editor/views/main.html
```
![muddery-editor][2]





## 编辑游戏

> 编辑器中各 选项的属性 请参考 文档 http://www.muddery.org/?cate=docs&content=documentations
>
> 点击 web ui 中的 编辑游戏 打开 游戏编辑器

```
1. 添加 区域

    首先我们点击 世界地图 --> 区域 --> Add 

    添加一个新手村 

```
![muddery-AREA][4]




```
2. 添加 房间

    点击 世界地图 --> 房间 --> Add

    添加一个房间 , 该房间创建于上面添加的 新手村区域中

```
![muddery-ROOM][5]



```
3. 添加 角色模板

     点击 角色 --> 角色模板 --> Add

     添加 角色模板.
     
```
![muddery-models][6]




```
4. 添加 世界NPC

   点击 角色 --> 世界NPC --> Add
   
   添加 世界NPC

```
![muddery-world-npc][7]


```
5. 添加 对话

   点击 对话 --> 对话列表 --> Add

   添加 对话

```
![muddery-dialogues][8]


```
# 添加对话内容 必须要保存了 对话才能添加

   点击 对话 --> 对话列表 --> 刚才添加的对话 --> Edit
   
   点击 Sentences --> Add

```

![muddery-dialogue-sentences][9]




> 添加完以上东西以后，再回去配置 游戏设置

```
6.  配置 游戏设置

    点击 基础设置 --> 游戏设置

```


```
7. 应用到游戏

    点击 管理 --> 应用
    
    Changes Applied. Please wait the server to restart.

    这里需要手动到 服务器上面 执行重启游戏服务

```


```
8.  登录游戏 

http://127.0.0.1:8000/webclient/main.html

```
![muddery-game][3]

![muddery-game-2][10]

![muddery-game-3][11]

![muddery-game-4][12]

![muddery-game-4][13]

  [1]: https://jicki.cn/img/posts/muddery/muddery-web.png
  [2]: https://jicki.cn/img/posts/muddery/muddery-editor.png
  [3]: https://jicki.cn/img/posts/muddery/muddery-game.png
  [4]: https://jicki.cn/img/posts/muddery/muddery-AREA.png
  [5]: https://jicki.cn/img/posts/muddery/muddery-ROOM.png
  [6]: https://jicki.cn/img/posts/muddery/muddery-models.png
  [7]: https://jicki.cn/img/posts/muddery/muddery-world-npc.png
  [8]: https://jicki.cn/img/posts/muddery/muddery-dialogues.png
  [9]: https://jicki.cn/img/posts/muddery/muddery-dialogue-sentences.png
  [10]: https://jicki.cn/img/posts/muddery/muddery-game-2.png
  [11]: https://jicki.cn/img/posts/muddery/muddery-game-3.png
  [12]: https://jicki.cn/img/posts/muddery/muddery-game-4.png
  [13]: https://jicki.cn/img/posts/muddery/muddery-game-5.png

