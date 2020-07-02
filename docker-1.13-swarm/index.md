# docker 1.13 swarm 集群



# 1.13 Changelog

```
#简单介绍如下几点:

# XFS 默认文件系统更新为 overlay2

Server Version: 1.13.0
Storage Driver: overlay
 Backing Filesystem: xfs


# 添加 service 容器互通 ping 支持
 
# service 创建 添加 hostname

docker service create --hostname
 
 
# 正式支持 docker stack
# 支持docker-compose version 3 
docker stack deploy


```

# 安装 docker 1.13


```
wget -qO- https://get.docker.com/ | sh


#修改 配置

sed -i 's/dockerd/dockerd --graph=\/opt\/docker\/images/g' /lib/systemd/system/docker.service

systemctl daemon-reload
systemctl restart docker.service
systemctl enable docker.service
```


# 创建 swarm 


## Swarm init

```
[root@swarm-node-1 ~]#docker swarm init --advertise-addr 10.6.0.140
Swarm initialized: current node (hf0h3qnlsicf5x6bltubjxgtg) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-26a59cyiceqq93w79abuyuaifkr2pxbsj41k9gas0ttxfrcw4x-an3tg2qodlh8noc8fx4zwniic \
    10.6.0.140:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.

```


## Swarm join

```
[root@swarm-node-2 ~]#docker swarm join \
>     --token SWMTKN-1-26a59cyiceqq93w79abuyuaifkr2pxbsj41k9gas0ttxfrcw4x-an3tg2qodlh8noc8fx4zwniic \
>     10.6.0.140:2377
This node joined a swarm as a worker.




[root@swarm-node-3 ~]#docker swarm join \
>     --token SWMTKN-1-26a59cyiceqq93w79abuyuaifkr2pxbsj41k9gas0ttxfrcw4x-an3tg2qodlh8noc8fx4zwniic \
>     10.6.0.140:2377
This node joined a swarm as a worker.
```



```
[root@swarm-node-1 ~]#docker node ls
ID                           HOSTNAME      STATUS  AVAILABILITY  MANAGER STATUS
e5b6fq2gfv3fex37q06974kg6    swarm-node-2  Ready   Active        
hf0h3qnlsicf5x6bltubjxgtg *  swarm-node-1  Ready   Active        Leader
zfniuw09nktpqdyswulnxemqt    swarm-node-3  Ready   Active 
```



## overlay network

```
# 创建一个 overlay 网络

[root@swarm-node-1 ~]#docker network create --driver overlay --opt encrypted --subnet=10.0.9.0/24 my-net
vxdbvrbvxcpep6sk6mtfy3nlp



[root@swarm-node-1 ~]#docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
vxdbvrbvxcpe        my-net              overlay             swarm

```


## docker service create

```
docker service create --replicas 3 --name my-nginx --network my-net --endpoint-mode dnsrr nginx:alpine

```




## docker stack deploy

```
[root@swarm-node-1 ~]#docker-compose version
docker-compose version 1.10.0, build 4bd6f1a
docker-py version: 2.0.2
CPython version: 2.7.5
OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013


# 编写一个 docker-compose.yaml

version: '3'
networks:
        network-pro:
                external:
                        name: procdution
services:
        nginx-1:
                image: nginx:alpine
                networks:
                        network-pro:
                               aliases:
                                        - nginx
                deploy:
                        replicas: 3
                        update_config:
                                parallelism: 2
                                delay: 10s
                        restart_policy:
                                condition: on-failure
                                delay: 5s
                                max_attempts: 5
                                window: 120s
                hostname: nginx-1
                container_name: nginx-1
                ports:
                - "80:80"
                volumes:
                - /opt/upload/nginx-1/logs:/var/log/nginx/

# 创建 service

docker stack deploy -c docker-compose.yaml nginx

Creating service nginx_nginx-1

```




## 创建一个zk 集群


```
# 编写一个 zk.yaml

version: '3'
networks:
        network-pro:
                external:
                        name: procdution
services:
        zookeeper-1:
                image: zk:alpine
                networks:
                        network-pro:
                               aliases:
                                        - zookeeper-1
                deploy:
                        update_config:
                                parallelism: 2
                                delay: 10s
                        restart_policy:
                                condition: on-failure
                                delay: 5s
                                max_attempts: 5
                                window: 120s
                hostname: zookeeper-1
                container_name: zookeeper-1
                environment:
                - NODE_ID=1
                - NODES=zookeeper-1,zookeeper-2,zookeeper-3

        zookeeper-2:
                image: zk:alpine
                networks:
                        network-pro:
                               aliases:
                                        - zookeeper-2
                deploy:
                        update_config:
                                parallelism: 2
                                delay: 10s
                        restart_policy:
                                condition: on-failure
                                delay: 5s
                                max_attempts: 5
                                window: 120s
                hostname: zookeeper-2
                container_name: zookeeper-2
                environment:
                - NODE_ID=2
                - NODES=zookeeper-1,zookeeper-2,zookeeper-3

        zookeeper-3:
                image: zk:alpine
                networks:
                        network-pro:
                               aliases:
                                        - zookeeper-3
                deploy:
                        update_config:
                                parallelism: 2
                                delay: 10s
                        restart_policy:
                                condition: on-failure
                                delay: 5s
                                max_attempts: 5
                                window: 120s
                hostname: zookeeper-3
                container_name: zookeeper-3
                environment:
                - NODE_ID=3
                - NODES=zookeeper-1,zookeeper-2,zookeeper-3

# 创建 service

[root@swarm-node-1 ~]#docker stack deploy -c zk.yaml zk

[root@swarm-node-1 ~]#docker stack ls
NAME   SERVICES
nginx  2
zk     3


[root@swarm-node-1 ~]#docker service ls
ID            NAME            MODE        REPLICAS  IMAGE
0ivc88rl1krj  nginx_nginx-2   replicated  1/1       nginx:alpine
luysdxxywxpg  zk_zookeeper-3  replicated  1/1       zk:alpine
mgi1acc4nz2p  nginx_nginx-1   replicated  1/1       nginx:alpine
wtkwruzhzg0w  zk_zookeeper-1  replicated  1/1       zk:alpine
ztlj74u7ir7f  zk_zookeeper-2  replicated  1/1       zk:alpine




```



