# docker 镜像仓库 Harbor 部署




# docker 镜像仓库 Harbor

## 说明

```
Harbor 是 Vmware 公司开源的 企业级的 Docker Registry 管理项目

它主要 提供 Dcoker Registry 管理UI，可基于角色访问控制, AD/LDAP 集成，日志审核等功能，完全的支持中文。

Harbor 的所有组件都在 Dcoker 中部署，所以 Harbor 可使用 Docker Compose 快速部署。


注： 由于 Harbor 是基于 Docker Registry V2 版本，所以 docker 版本必须 > = 1.10.0 docker-compose >= 1.6.0

```

[harbor 项目地址][1]


## 下载 harbor


git 下载 源码。

```
git clone https://github.com/vmware/harbor
```


## 编辑配置文件


下载完以后 进入 harbor/Deploy 目录

初始化配置， 配置文件为harbor.cfg

```
## Configuration file of Harbor
# hostname 设置访问地址，支持IP，域名，主机名，禁止设置127.0.0.1
hostname = reg.mydomain.com

# 访问协议，可设置 http,https
ui_url_protocol = http

# 邮件通知, 配置邮件通知。
email_server = smtp.mydomain.com
email_server_port = 25
email_username = sample_admin@mydomain.com
email_password = abc
email_from = admin <sample_admin@mydomain.com>
email_ssl = false

# harbor WEB UI登陆使用的密码
harbor_admin_password = Harbor12345

# 认证方式，这里支持多种认证方式，默认是 db_auth ，既mysql数据库存储认证。
# 这里还支持 ldap 以及 本地文件存储方式。
auth_mode = db_auth

# ldap 服务器访问地址。
ldap_url = ldaps://ldap.mydomain.com
ldap_basedn = uid=%s,ou=people,dc=mydomain,dc=com

# mysql root 账户的 密码
db_password = root123
self_registration = on
use_compressed_js = on
max_job_workers = 3 
verify_remote_cert = on
customize_crt = on

# 一些显示的设置.
crt_country = CN
crt_state = State
crt_location = CN
crt_organization = organization
crt_organizationalunit = organizational unit
crt_commonname = example.com
crt_email = example@example.com
```


修改为配置文件以后 运行./prepare脚本更新配置, 出现如下信息表示 更新完毕.

```
./prepare

Generated configuration file: ./config/ui/env
Generated configuration file: ./config/ui/app.conf
Generated configuration file: ./config/registry/config.yml
Generated configuration file: ./config/db/env
Generated configuration file: ./config/jobservice/env
Clearing the configuration file: ./config/ui/private_key.pem
Clearing the configuration file: ./config/registry/root.crt
Generated configuration file: ./config/ui/private_key.pem
Generated configuration file: ./config/registry/root.crt
The configuration files are ready, please use docker-compose to start the service.
```

执行完毕会生成一个 docker-compose.yml  文件


配置 docker-compose.yml 文件中的 挂载目录，启动方式等选项。 


## 安装 docker-compose

```
pip install docker-compose 
```


## 生成容器

```
docker-compose up -d 
```

构建docker 容器
 
```
[root@localhost Deploy]# docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
deploy_jobservice   latest              be822b50163d        43 minutes ago      804.6 MB
deploy_mysql        latest              5015ce56c9d5        49 minutes ago      328.8 MB
deploy_ui           latest              8596c12dbeba        About an hour ago   808.1 MB
deploy_log          latest              6a74c6f52a2b        About an hour ago   187.9 MB
mysql               5.6                 5e0f1b09e25e        2 days ago          328.8 MB
ubuntu              14.04               0ccb13bf1954        12 days ago         187.9 MB
golang              1.6.2               8ecba0e9bd48        5 weeks ago         753.5 MB
nginx               1.9                 c8c29d842c09        10 weeks ago        182.7 MB
registry            2.4.0               8b162eee2794        3 months ago        171.1 MB
```
 
```
[root@localhost Deploy]# docker ps -a
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                                      NAMES
9704f42b05d5        deploy_jobservice        "/go/bin/harbor_jobse"   4 minutes ago       Up 4 minutes                                                   deploy_jobservice_1
0f8ff9b099d2        library/nginx:1.9        "nginx -g 'daemon off"   4 minutes ago       Up 4 minutes        0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   deploy_proxy_1
6b0159939874        deploy_ui                "/go/bin/harbor_ui"      4 minutes ago       Up 4 minutes        80/tcp                                     deploy_ui_1
6f2298da0f67        deploy_mysql             "docker-entrypoint.sh"   4 minutes ago       Up 4 minutes        3306/tcp                                   deploy_mysql_1
2dbca506e1ea        library/registry:2.4.0   "/bin/registry serve "   4 minutes ago       Up 4 minutes        5000/tcp, 0.0.0.0:5001->5001/tcp           deploy_registry_1
fc5b1a201c72        deploy_log               "/bin/sh -c 'cron && "   4 minutes ago       Up 4 minutes        0.0.0.0:1514->514/tcp                      deploy_log_1
```
 


完成以后，使用 http://userIP/ 访问 Harbor

使用 帐号 admin, 密码为 配置文件中 harbor_admin_password = Harbor12345 的密码 登陆

至此， Harbor 已经搭建完成，具体在 WEB UI 下面操作也是非常的简单，只有几个选项。


## 上传镜像

docker 需要上传 push 镜像，需要在 docker 中配置 --insecure-registry userIP 或者在nginx 中配置 https

配置完毕以后，重启 docker

使用 docker login userIP 登陆 Harbor

```
[root@swarm-manager ~]#docker login 10.6.0.192
Username (admin): admin
Password: 
Login Succeeded
```


查看 本地 images

```
[root@swarm-manager ~]#docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
mongodb             latest              8af05a33e512        3 weeks ago         958.4 MB
sath89/oracle-12c   latest              7effebcd18ee        11 weeks ago        5.692 GB
centos              latest              778a53015523        4 months ago        196.7 MB
``` 


tag 修改 image 的名字. 格式为: userip/项目名/image名字:版本号

```
[root@swarm-manager ~]#docker tag mongodb 10.6.0.192/jicki/mongodb:1.0

[root@swarm-manager ~]#docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
10.6.0.192/jicki/mongodb   1.0                 8af05a33e512        3 weeks ago         958.4 MB
mongodb                    latest              8af05a33e512        3 weeks ago         958.4 MB
sath89/oracle-12c          latest              7effebcd18ee        11 weeks ago        5.692 GB
centos                     latest              778a53015523        4 months ago        196.7 MB
```
 
push 镜像到 Harbor

```
[root@swarm-manager ~]#docker push 10.6.0.192/jicki/mongodb:1.0
The push refers to a repository [10.6.0.192/jicki/mongodb]
c1e4cd91bcd4: Pushed 
d9a948970255: Pushed 
dd9b001e77ee: Pushed 
625440e212f2: Pushed 
75fa23acbccb: Pushed 
fd269370dcf4: Pushed 
44e3199c59b3: Pushed 
db3474cfcfbc: Pushed 
5f70bf18a086: Pushed 
6a6c96337be1: Pushed 
1.0: digest: sha256:c7d2e619d86089ffef373819a99df1390c4f2df4aeec9c1f7945c55d63edc670 size: 2824
```
 

 

登陆 WEB UI ， 选择项目， 项目名称 jicki ， 进入 既可查看刚才上传的 image
![WEB UI][2]


至此， Harbor 都已经部署完成。

 
二、 配置Docker 镜像复制。
![此处输入图片的描述][3]
![此处输入图片的描述][4] 



配置 2个 Harbor

IP 1 = 10.6.0.192

IP 2 = 10.6.0.196


在 10.6.0.192 上面我们已经push 了一个 镜像，所以我们将这台当作 主节点，10.6.0.196 为从复制节点。

进入 WEB UI 选择 项目， 选择项目为 jicki , 然后选择 复制 选项。

![此处输入图片的描述][5]


点击 新增策略
![此处输入图片的描述][6]
![此处输入图片的描述][7] 


创建完毕以后，我们可以看 复制策略 已经有一栏。

复制任务里面 也已经有一个任务。
![此处输入图片的描述][8]
 

稍等一会，可以看到 复制任务里面 那个任务已经提示 完成。

![此处输入图片的描述][9]


登陆 10.6.0.196 的 WEB UI
![此处输入图片的描述][10]
 
我们可以看到， 镜像已经复制过来。而且连 日志操作 也会复制过来。





## harbor 升级


```
cd harbor/Deploy/

docker-compose down
```

删除原有的容器

 

备份整个目录

```
mv harbor/ /tm/harbor
```


重新 下载新的源码

```
git clone https://github.com/vmware/harbor
```
 

如果harbor 是迁移到其他服务器，请先执行数据备份

cd harbor/migration/

修改 migration.cfg 文件里面的 数据库 帐号密码

```
docker build -t migrate-tool .
```
 

运行一个临时数据库容器，注意：/data/database 为你设置的挂载数据库的目录 /path/to/backup 数据备份的目录

数据库备份：

```
docker run -ti --rm -v /data/database:/var/lib/mysql -v /path/to/backup:/harbor-migration/backup migrate-tool backup
```
 

数据库还原：

```
docker run -ti --rm -v /data/database:/var/lib/mysql migrate-tool up head
```
 

对比一下配置文件：

```
cd harbor/Deploy/

diff harbor.cfg /tmp/harbor/Deploy/harbor.cfg

diff docker-compose.yaml /tmp/harbor/Deploy/docker-compose.yaml
```


如果修改了端口 必须更新 nginx 里面的端口

```
harbor/Deploy/config/nginx/nginx.conf 
```
 

执行 ./prepare 生成新的配置文件

```
cd /harbor/Deploy/

./prepare
```
 

最后build 新的镜像，启动容器

```
cd /harbor/Deploy/

docker-compose up --build -d
```


登陆 WEB UI 检查是否OK

 

## FAQ 删除镜像，回收容量

```

# 首先确认 并打印是否有正在上传与下载的镜像

$ docker-compose stop

$ docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect --dry-run /etc/registry/config.yml


# 执行如下命令 GC 删除镜像

$ docker run -it --name gc --rm --volumes-from registry vmware/registry:2.6.2-photon garbage-collect  /etc/registry/config.yml


# 删除 重新启动

$ docker-compose start



```


  [1]: https://github.com/vmware/harbor
  [2]: https://jicki.cn/img/posts/harbor/1.png
  [3]: https://jicki.cn/img/posts/harbor/2.png
  [4]: https://jicki.cn/img/posts/harbor/3.png
  [5]: https://jicki.cn/img/posts/harbor/4.png
  [6]: https://jicki.cn/img/posts/harbor/5.png
  [7]: https://jicki.cn/img/posts/harbor/6.png
  [8]: https://jicki.cn/img/posts/harbor/7.png
  [9]: https://jicki.cn/img/posts/harbor/8.png
  [10]: https://jicki.cn/img/posts/harbor/9.png

