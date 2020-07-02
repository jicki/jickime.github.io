# GitLab + Runner



# GitLab

## 编辑Yum源

```
cat <<EOF> /etc/yum.repos.d/gitlab-ce.repo
[gitlab-ce]
name=gitlab-ce
baseurl=https://mirrors.tuna.tsinghua.edu.cn/gitlab-ce/yum/el7/
repo_gpgcheck=0
gpgcheck=0
enabled=1
gpgkey=https://packages.gitlab.com/gpg.key
EOF
```

## 安装软件

```
# 安装依赖
yum install curl openssh-server openssh-clients postfix cronie

# 启动 postfix 邮件服务
systemctl start postfix
systemctl enable postfix

# 检查 postfix
systemctl status postfix

# 安装 GitLab 社区版
yum -y install gitlab-ce


# 启动程序
gitlab-ctl start


# 检测错误
gitlab-rake gitlab:check


# 初始化 GitLab
gitlab-ctl reconfigure
```

## 修改配置

```
vi /etc/gitlab/gitlab.rb

# 修改域名
external_url 'http://my_url/'

# 修改备份目录 备份周期7天 604800秒

gitlab_rails['backup_keep_time'] = 604800
gitlab_rails['backup_path'] = "/opt/gitlab/backups"

# 更换安装源
/opt/gitlab/embedded/bin/gem sources --add https://ruby.taobao.org/ --remove https://rubygems.org/

# 让配置生效, 任何配置修改必须执行 reconfigure
gitlab-ctl reconfigure
```


## 访问 gitlab

```
http://my_url/

# 最开始会让你创建密码
# 登陆帐号为 root
```



# 持续集成(GitLab-CI)

## 安装源

```
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-ci-multi-runner/script.rpm.sh | bash
```

## 安装 gitlab-ci-runner

```
yum install gitlab-ci-multi-runner -y

```


## 生成 runner token

```
http://my_url/admin/runners
```

## 配置 runner

```
[root@localhost ~]# gitlab-ci-multi-runner register
Running in system-mode.                            
                                                   
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://my_url/
Please enter the gitlab-ci token for this runner:
1RAM5gnaTjBec1ExYskF
Please enter the gitlab-ci description for this runner:
[localhost.localdomain]: my-runner       
Please enter the gitlab-ci tags for this runner (comma separated):
docker
WARNING: No TLS connection state                   
Registering runner... succeeded                     runner=1RAM5gna
Please enter the executor: parallels, shell, ssh, docker+machine, docker-ssh+machine, docker, docker-ssh, virtualbox, kubernetes:
docker
Please enter the default Docker image (eg. ruby:2.1):
ruby:2.1
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 
```



```
这里边要说明的是  

Please enter the executor: parallels, shell, ssh, docker+machine, docker-ssh+machine, docker, docker-ssh, virtualbox, kubernetes:

这一项。

这里的意思是~ 你这个 runner 是用来运行在什么情况下的。

如果物理机，一般是用 shell 来做 build .  如果是 docker 环境，就选择 docker ,  k8s集群 就选 kubernetes 。

```









## 查看生成

```
# 登陆
http://my_url/admin/runners

# Runners with last contact less than a minute ago: 1
# 可以看到有一条记录
```

## 信息修改与检查

```
 # runners 生成的信息存放于
 /etc/gitlab-runner/config.toml
```


## runners 使用

```
[root@localhost ~]# gitlab-runner -h
NAME:
   gitlab-runner - a GitLab Runner

USAGE:
   gitlab-runner [global options] command [command options] [arguments...]
   
VERSION:
   1.6.1 (c52ad4f)
   
AUTHOR(S):
   Kamil Trzciński <ayufan@ayufan.eu> 
   
COMMANDS:
   exec                 execute a build locally
   list                 List all configured runners
   run                  run multi runner service
   register             register a new runner
   install              install service
   uninstall            uninstall service
   start                start service
   stop                 stop service
   restart              restart service
   status               get status of a service
   run-single           start single runner
   unregister           unregister specific runner
   verify               verify all registered runners
   artifacts-downloader download and extract build artifacts (internal)
   artifacts-uploader   create and upload build artifacts (internal)
   cache-archiver       create and upload cache artifacts (internal)
   cache-extractor      download and extract cache artifacts (internal)
   help, h              Shows a list of commands or help for one command
   
GLOBAL OPTIONS:
   --debug                      debug mode [$DEBUG]
   --log-level, -l "info"       Log level (options: debug, info, warn, error, fatal, panic)
   --cpuprofile                 write cpu profile to file [$CPU_PROFILE]
   --help, -h                   show help
   --version, -v                print the version
```



## runner yaml 文件

官方地址

```
https://docs.gitlab.com/ce/ci/yaml/README.html
```



# 运维维护

## 定时备份

```
0 2 * * * gitlab-rake gitlab:backup:create
```


## 恢复数据

```
# 停止 unicorn
gitlab-ctl stop unicorn

# 停止 sidekiq
gitlab-ctl stop sidekiq

# 恢复数据 BACKUP= 指定恢复的时间戳
cd 备份目录

gitlab-rake gitlab:backup:restore BACKUP=1406691018
```





# FAQ 问题

## gitlab-ctl reconfigure 问题

```
ruby_block[supervise_redis_sleep] action run  卡住

#执行
ls -al /opt/gitlab/sv/redis/supervise
#提示
ls: cannot access /opt/gitlab/sv/redis/supervise: No such file or directory
# 重启 supervise
systemctl restart gitlab-runsvdir.service
```

## execute[create gitlab database user] 问题

```
gitlab-ctl stop
rm -rf /var/opt/gitlab/postgresql
gitlab-ctl start
gitlab-ctl reconfigure
```


## psql: could not connect to server: No such file or directory 问题

```
# 查看 错误
gitlab-ctl tail postgresql

# 发现 kernel with larger SHMMAX
增大 内核 shmmax 参数 或者 减少 gitlab 配置中 shared_buffers 参数

vi /etc/gitlab/gitlab.rb

postgresql['shared_buffers'] = "2560MB"

# 重启
gitlab-ctl restart
```

