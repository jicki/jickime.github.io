# 基于 docker-compose 的docker编排




# docker 编排

## 说明

docker-compose 是一款开源的docker 简化复杂容器环境的管理工具 。

docker-compose 在结合Swarm 与 docker 进程化容器部署可以很方便的部署一套环境。

具体的流程如下：
![此处输入图片的描述][1]
 
 
## docker-compose 安装

docker-compose 是用 python 写的，所以我们安装

使用 

```
pip install docker-compose
```

升级 ssl_match_hostname 大于等于3.5.1 版本，才能使用 docker-compose

```
pip install backports.ssl_match_hostname --upgrade
```
 


## 编写 yaml 文件

以下是 V2 版本的 docker-compose.yaml 

docker-compose  v2 版本 标签
官方文档：
https://docs.docker.com/compose/compose-file/#version-2


```
version: '2'                 # 使用compose v2 版本
networks:                    # 定义一个网络
        network-cn:          # 定义网络名称
                external:                             
                        name: ovrcn      # 使用自行创建的网络，如果不设置这个，会自动创建一个别的网络，与你原来网络不在一个网络中。
services:                                             # 服务组名称
        nginx-1:
                image: nginx                          # 镜像
                networks:
                        network-cn:                   # 自定义的网络名
                               aliases:   
                                        - nginx       # overlay 网络的网络别名
                hostname: nginx                       # 容器里面的 hostname
                container_name: nginx-1               # 创建容器时的容器名称
                ports:                                # 映射端口
                - "80:80"
                - "443:443"
                environment:                          # --env 配置
                - constraint:node==swarm-node-28
                volumes:                              # 挂载目录
                - /opt/data/nginx/logs:/opt/local/nginx/logs
```


编写完 docker-compose.yaml  
使用 

```
docker-compose up -d
```


使用  docker-compose ps 命令可以查看 容器的启动情况。

```
docker-compose ps
 
  Name     Command               State                Ports         

-----------------------------------------------------------------------------------

nginx-1    /opt/local/nginx/sbin/ngin ...  Up      172.16.1.28:443->443/tcp, 172.16.1.28:80->80/tcp
```




## docker-compose 约束标签

```
# 约束两个nginx相关的容器不在同一个节点中
affinity:container!=nginx-*


# 约束两个服务，如master 与 slave 不会调度到同一个节点中 
affinity:service!=slave


# 约束 两个容器调度到不同的 可用区域中
availability:az==2

```











  [1]: https://jicki.cn/img/posts/docker-compose/1.png

