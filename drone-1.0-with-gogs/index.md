# Drone 1.0 with Gogs CI/CD



# Drone搭建的私有CI/CD平台

> Drone 是基于 Go语言开发的一款用于 CI/CD DevOps自动化平台, 它基于 Docker 配置以及运行.
>
> 官方 github : https://github.com/drone/drone
> 官方文档: https://docs.drone.io/


## 环境说明

|IP|系统|Kernel|docker 版本|docker-compose 版本|
|-|-|-|-|-|
|192.168.168.102|CentOS 7|4.4.181|18.09.6|1.24.0|


## 安装 Drone

> docker 与 docker-compose 安装就略过了。
> 
> Drone 使用 docker-compose 直接安装既可。
>
> Drone 支持三种数据库默认为 sqllite3 , 还支持 Mysql 与 Postgres
>
> 这里就使用 mysql 
> 
> 这里关闭 firewalld 与 selinux

### mysql 的 compose

```
version: '2'
services:
  mysql:
    image: mysql:5.7
    hostname: mysql
    container_name: mysql
    restart: always
    volumes:
    - ./data/mysql/data:/var/lib/mysql
    - ./data/mysql/logs:/opt/local/mysql/logs
    - ./data/mysql/binlog:/opt/local/mysql/binlog
    - /etc/localtime:/etc/localtime
    environment:
    - MYSQL_ROOT_PASSWORD=123456
    - TZ=Asia/Shanghai
    ports:
      - "192.168.168.102:3306:3306"

```


### 创建一个 Gogs 


```
  gogs:
    image: gogs/gogs
    hostname: gogs
    container_name: gogs
    restart: always
    volumes:
    - /etc/localtime:/etc/localtime
    - ./data/gogs:/data
    ports:
      - "192.168.168.102:3000:3000"
    depends_on:
      - mysql

```




```
# 登录 mysql , 创建 gogs 以及 drone 数据库

mysql -uroot -p123456 -h 127.0.0.1

mysql> select version();
+-----------+
| version() |
+-----------+
| 5.7.26    |
+-----------+
1 row in set (0.00 sec)


mysql> CREATE DATABASE IF NOT EXISTS gogs CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
Query OK, 1 row affected (0.00 sec)


mysql> create database drone;
Query OK, 1 row affected (0.00 sec)

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| drone              |
| gogs               |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)


```


### 配置 gogs 

```
# 使用浏览器 访问 http://192.168.168.102:3000

```



```
# 这里配置相关数据库信息 并将localhost 修改为 192.168.168.102

```


```
# 登录 gogs 以后，创建一个测试的仓库

创建新的仓库:
     拥有者: jicki
     仓库名称: drone
     仓库描述: 测试 drone 仓库
     .gitignore: Actionscript
     授权许可: Apache License 2.0
     自述文档: Default
     使用选定的文件和模板初始化仓库

```

![图1][1]


### 配置 drone 的compose

> Drone 支持常见的Git仓库，例如 Github,Gitlab, Bitbucket以及Gogs等。
> 
> 这里使用 Gogs, 不同的 git 仓库 配置有所不同。




```
# 创建一个 secret 用于 Agent 与 Server 间的通讯

[root@localhost ~]# openssl rand -hex 16
dfde64837533976ee7dbf4a87e03ad1f

```





```
services:
  drone-server:
    image: drone/drone:latest
    hostname: drone-server
    container_name: drone-server
    restart: always
    environment:
      - DRONE_LOGS_DEBUG=true
      - DRONE_GIT_ALWAYS_AUTH=false
      - DRONE_AGENTS_ENABLED=true
      - DRONE_GOGS_SERVER=http://192.168.168.102:3000
      - DRONE_RPC_SECRET=dfde64837533976ee7dbf4a87e03ad1f
      - DRONE_RUNNER_CAPACITY=2
      - DRONE_SERVER_HOST=192.168.168.102:8000
      - DRONE_SERVER_PROTO=http
      - DRONE_TLS_AUTOCERT=false
    volumes:
      - /etc/localtime:/etc/localtime
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data/drone:/var/lib/drone/
    ports:
      - 8000:80
      - 6443:443
    depends_on:
      - mysql
  drone-agent:
    image: drone/agent:latest
    hostname: drone-agent
    container_name: drone-agent
    restart: always
    environment:
      - DRONE_DEBUG=true
      - DRONE_RPC_SERVER=http://192.168.168.102:8000
      - DRONE_RPC_SECRET=dfde64837533976ee7dbf4a87e03ad1f
      - DRONE_RUNNER_CAPACITY=2
    volumes:
      - /etc/localtime:/etc/localtime
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - drone-server

```


### 验证 Drone 服务

```
# 使用浏览访问

http://192.168.168.102:8000


# 这里特别注意，因为我们关联的是 gogs 的 git

# 所以这里登录 drone 的时候，必须使用 gogs 的账号密码

# 所以这里我们 部署 CI/CD 的时候~可以使用一个 管路员账号

# 或者使用一个有 所有组权限 的账号
```


![图2][2]





### 激活仓库


![图3][3]




![图4][4]

![图5][5]

![图6][6]




### 创建一个 .drone.yml 文件


```
# 创建一个 .drone.yml 并上传到 gogs 仓库中


# 这里是 单 Agent, 的 pipeline

# 1.0 版本支持 多 Agent, 多个 node 区分的 pipeline

vi .drone.yml

kind: pipeline
name: default

steps:
- name: frontend
  image: alpine
  commands:
  - echo "This My Drone CI Test!"
```


![图7][7]

![图8][8]

![图9][9]



```
# 创建一个 后端 golang

```

```
vi main.go


package main

import (
    "fmt"
)

func main() {
    fmt.Printf("hello world");
}

func hello() string {
    return "hello world";
}

```

```
vi main_test.go

package main

import "testing"

func TestHello(t *testing.T) {
    if hello() != "hello world" {
        t.Error("Testing error")
    }
}

```


```
# 编辑 .drone.yml 文件，增加后端

vi .drone.yml

kind: pipeline
name: default

steps:
- name: frontend
  image: alpine
  commands:
  - echo "This My Drone CI Test!"

- name: backend
  image: golang
  commands:
  - go build
  - go test

```

![图10][10]

![图11][11]

![图12][12]



  [1]: https://jicki.cn/img/posts/drone/1.png
  [2]: https://jicki.cn/img/posts/drone/2.png
  [3]: https://jicki.cn/img/posts/drone/3.png 
  [4]: https://jicki.cn/img/posts/drone/4.png 
  [5]: https://jicki.cn/img/posts/drone/5.png 
  [6]: https://jicki.cn/img/posts/drone/6.png 
  [7]: https://jicki.cn/img/posts/drone/7.png 
  [8]: https://jicki.cn/img/posts/drone/8.png 
  [9]: https://jicki.cn/img/posts/drone/9.png 
  [10]: https://jicki.cn/img/posts/drone/10.png 
  [11]: https://jicki.cn/img/posts/drone/11.png 
  [12]: https://jicki.cn/img/posts/drone/12.png 

