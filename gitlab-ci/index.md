# GitLab CI/CD Docker



# Gitlab CI 介绍

> GitLab CI 是 GitLab 为了提升其在软件开发工程中作用，完善 DevOps 理念所加入的 CI/CD 基础功能。可以便捷的融入软件开发环节中。通过 GitLab CI 可以定义完善的 CI/CD Pipeline。
>
> GitLab CI 是默认包含在 GitLab 中的，如果代码在 GitLab 进行托管，可以很容易的进行集成
>
> GitLab CI 的前端界面比较美观，容易被人接受
>
> 构建日志相对完整，容易追踪错误
>
> 使用 YAML 进行配置，任何人都可以很方便的使用

# Gitlab 名词解释

> gitlab 中的名词，我们在 .gitlab-ci.yaml 中会经常使用到

```
1. Pipeline 
    Pipeline 相当于一个构建任务，里面可以包含多个流程，如依赖安装、编译、测试、部署等, 任何提交或者 Merge Request 的合并都可以触发 Pipeline.
```


```
2. Stages
    Stage 表示构建的阶段，即 Pipeline 中的包含的流程.

    所有 Stages 按顺序执行，即当一个 Stage 完成后，下一个 Stage 才会开始.

    任一 Stage 失败，后面的 Stages 将永不会执行，Pipeline 整个过程失败.

    只有当所有 Stages 完成后，Pipeline 才会成功.
```

```
3. Jobs
    Job 是 Stage 中的任务.

    相同 Stage 中的 Jobs 会并行执行.

    任一 Job 失败，那么 Stage 失败，Pipeline 失败.

    相同 Stage 中的 Jobs 都执行成功时，该 Stage 成功.

```


# Gitlab-ce 搭建

> Gitlab 支持本地部署, 支持 docker 部署, 支持 kubernetes 部署.
>
> 为了方便还是使用 docker 部署最简单, 使用 docker-compose 直接启动.


```
# 创建 gitlab 的 docker-compose.yaml 文件


version: '2'
services:
  gitlab:
    image: gitlab/gitlab-ce
    restart: always
    container_name: gitlab
    hostname: gitlab
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        external_url 'http://gitlab.jicki.cn'
    ports:
      - '80:80'
      - '443:443'
      - '8022:22'
    volumes:
      - './data/gitlab/config:/etc/gitlab'
      - './data/gitlab/logs:/var/log/gitlab'
      - './data/gitlab/data:/var/opt/gitlab'

```

```
# 使用 docker-compose up -d 启动服务


# 查看启动的服务

[root@localhost compose]# docker-compose ps
 Name        Command          State                                   Ports                             
--------------------------------------------------------------------------------------------------------
gitlab   /assets/wrapper   Up (healthy)   0.0.0.0:8022->22/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp

```


```
浏览器 http://gitlab.jicki.cn 访问 gitlab

首次访问会提示配置 初始密码，最少8位
```

![图1][1]
![图2][2]

```
# 创建一个 用户，作为后续 gitlab-ci 的项目关联 用户.
```

![图3][3]

![图4][4]

![图5][5]


```
# 创建一个 Groups, 作为后续 整体团队关联 Project 项目.

# 添加 上面创建的用户到此 Groups 中.

```

![图6][6]

![图7][7]

```
# 创建一个 Project 项目.



```

![图8][8]

![图9][9]

![图10][10]



# 生成java项目

> 使用 https://start.spring.io/ 生成一个基于 Gradle 的java 项目


![图11][11]


```
# 将生成的项目 上传到 Gitlab 的 TestProject 项目中

# 生成的项目
[root@localhost testproject]# ll
总用量 28
-rw-r--r-- 1 root root  446 6月  19 06:32 build.gradle
drwxr-xr-x 3 root root   21 6月  19 06:32 gradle
-rwxr-xr-x 1 root root 5305 6月  19 06:32 gradlew
-rw-r--r-- 1 root root 2269 6月  19 06:32 gradlew.bat
-rw-r--r-- 1 root root  338 6月  19 06:32 HELP.md
-rw-r--r-- 1 root root   11 6月  19 14:34 README.md
-rw-r--r-- 1 root root   96 6月  19 06:32 settings.gradle
drwxr-xr-x 4 root root   30 6月  19 06:32 src

```


```
# 修改 build.gradle , 主要添加 dependencies 下的几项

buildscript {
        repositories {
                mavenCentral()
        }
        dependencies {
                classpath 'org.springframework.boot:spring-boot-gradle-plugin:1.5.21.RELEASE'
        }
}

plugins {
        id 'java'
}

apply plugin: 'org.springframework.boot'

group = 'java'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
        mavenCentral()
}

dependencies {
        compile 'org.springframework.boot:spring-boot-starter'
        compile 'org.springframework.boot:spring-boot-starter-web'
        compile 'org.springframework.boot:spring-boot-starter-thymeleaf'
        testCompile 'org.springframework.boot:spring-boot-starter-test'
}


```


```
# 在 src/main/java/java/TestProject 目录下
# 创建一个 HomeController.java 文件 , 内容如下:


package java.java.TestProject;

import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;

@Controller
public class HomeController {

    @RequestMapping("/")
    public String home(){
        return "index";
    }
}

```


```
# 在 src/main/resources/templates 目录下
# 创建一个 index.html 文件, 内容如下:


<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8"/>
    <title>Title</title>
</head>
<body>
<h1>Test Java Gradle project</h1>
</body>
</html>

```

```
# 上传到 gitlab 中

[root@localhost testproject]# git add -A

[root@localhost testproject]# git commit -m "add java project"

[root@localhost testproject]# git push
Username for 'http://gitlab.jicki.cn': jicki
Password for 'http://jicki@gitlab.jicki.cn': 
Counting objects: 36, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (15/15), done.
Writing objects: 100% (20/20), 50.06 KiB | 0 bytes/s, done.
Total 20 (delta 4), reused 0 (delta 0)
To http://gitlab.jicki.cn/java/testproject.git
   9a25098..d95ec5f  master -> master
```


```
# Gitlab 中项目 Project 内容如下:

```

![图12][12]



# GitLab CI 配置

> Gitlab CI 的使用 需要使用 Runner 服务, Runner 主要负责CI任务的执行
>
> 这里 Runner 服务使用 docker 运行 docker-compose 启动 


```
services:
  gitlab-runner:
    container_name: gitlab-runner
    image: gitlab/gitlab-runner:alpine
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data/gitlab-runner/config/config.toml:/etc/gitlab-runner/config.toml
```


```
# 启动之前需要先创建 一个空的 config.toml 文件

[root@localhost config]# touch config.toml

# 启动
[root@localhost compose]# docker-compose up -d
gitlab is up-to-date
Creating gitlab-runner ... done



# 查看服务
[root@localhost compose]# docker-compose ps
    Name                   Command                  State                                   Ports                             
------------------------------------------------------------------------------------------------------------------------------
gitlab          /assets/wrapper                  Up (healthy)   0.0.0.0:8022->22/tcp, 0.0.0.0:443->443/tcp, 0.0.0.0:80->80/tcp
gitlab-runner   /usr/bin/dumb-init /entryp ...   Up   
```

```
# 查看日志

[root@localhost compose]# docker logs -f gitlab-runner
Runtime platform                                    arch=amd64 os=linux pid=6 revision=ac2a293c version=11.11.2
Starting multi-runner from /etc/gitlab-runner/config.toml ...  builds=0
Running in system-mode.                            
                                                   
Configuration loaded                                builds=0
listen_address not defined, metrics & debug endpoints disabled  builds=0
[session_server].listen_address not defined, session endpoints disabled  builds=0
```


## 注册 Runner

> Runner 注册以后必须要使用命令进行 激活，才能连接到gitlab-ce 服务里面去


```
# 在注册 Runner 之前，我们需要在 gitlab 里面查看 Runner 的 Token

# 一会我们注册的时候需要使用到这个 Token

```

![图13][13]


```
# 进入容器注册 Runner


[root@localhost compose]# docker exec -it gitlab-runner gitlab-runner register

# 输出如下:

Runtime platform                                    arch=amd64 os=linux pid=19 revision=ac2a293c version=11.11.2
Running in system-mode.                            
                                                   
Please enter the gitlab-ci coordinator URL (e.g. https://gitlab.com/):
http://gitlab.jicki.cn
Please enter the gitlab-ci token for this runner:
pr5VscoaY2fC_8WiSdc7
Please enter the gitlab-ci description for this runner:
[d1f701bae725]: TestProject
Please enter the gitlab-ci tags for this runner (comma separated):
JavaProject
Registering runner... succeeded                     runner=pr5Vscoa
Please enter the executor: parallels, shell, docker+machine, kubernetes, docker, docker-ssh, ssh, virtualbox, docker-ssh+machine:
docker
Please enter the default Docker image (e.g. ruby:2.1):
alpine
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded! 

```


```
# Runner registered successfully

# 登录 gitlab 后台查看 注册的 Runner 服务

```

![图14][14]


```
# 配置完成以后~查看 config.toml 生成的配置文件
# 我这里增加了一些配置注释说明
# 配置信息的文档地址 https://docs.gitlab.com/runner/configuration/advanced-configuration.html


[root@localhost config]# cat config.toml
// 限制同时运行多少个 jobs , 配置为0 为不限制
concurrent = 1
// 配置多就检查一下 jobs,最低为3,设置3以下，也为3.
check_interval = 0

[session_server]
  //jobs 完成以后保持 会话 的时间,默认为 1800 (30分钟).
  session_timeout = 1800

// 以下为 注册 runner 时填写的信息
[[runners]]
  name = "TestProject"
  url = "http://gitlab.jicki.cn"
  token = "V5tiX7JbKmhJx2gSDDcb"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "alpine"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
```



```
# 以上 config.toml 为默认生成的配置文件
# 以下为修改以后，的 config.toml
# 这里主要修改了一些 runner 中需要使用到的配置
# runners 中添加
#  // builds_dir 构建将存储在所选执行程序的上下文中的目录
#  builds_dir = "/gitlab/runner-builds"
#  // cache_dir 构建缓存的目录将存储在所选执行程序
#  cache_dir = "/gitlab/runner-cache"
#  由于我们 Runner 使用docker 运行，所以以上目录配置，需要额外增加配置 volumes 这个配置
#  volumes = ["/data/gitlab-runner:/gitlab","/var/run/docker.sock:/var/run/docker.sock","/data/gradle:/data/gradle","/data/sonar_cache:/root/.sonar"]
# extra_hosts 为额外的 dns 配置, 如果有dns 服务器就不需要
# extra_hosts = ["gitlab.jicki.cn:192.168.168.102"]




concurrent = 1
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "TestProject"
  url = "http://gitlab.jicki.cn"
  token = "V5tiX7JbKmhJx2gSDDcb"
  executor = "docker"
  builds_dir = "/gitlab/runner-builds"
  cache_dir = "/gitlab/runner-cache"
  [runners.custom_build_dir]
  [runners.docker]
    tls_verify = false
    image = "alpine"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/opt/data/gitlab-runner:/gitlab","/var/run/docker.sock:/var/run/docker.sock","/opt/data/gradle:/root/.gradle","/opt/data/sonar_cache:/root/.sonar"]
    extra_hosts = ["gitlab.jicki.cn:192.168.168.102"]
    shm_size = 0
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]

```



```
# 修改完配置以后, 删除容器，再重新启动既可.

# gitlab-runner 容器为无状态的服务.

```


# 创建初始镜像

> 本项目语言为 java , 使用docker 运行 java 项目的时候，需要一个初始化的环境，这里预先构建一个 alpine 基于 openjdk 的初始镜像, dockerfile 如下:
>
> 初始镜像需要预先构建，然后上传到 image 仓库中，方便以后各node pull 下来.


```
FROM alpine

ENV JAVA_HOME /usr/lib/jvm/java-1.8-openjdk
ENV PATH $PATH:/usr/lib/jvm/java-1.8-openjdk/jre/bin:/usr/lib/jvm/java-1.8-openjdk/bin

RUN apk add --update bash curl tar wget ca-certificates unzip \
    openjdk8 font-adobe-100dpi ttf-dejavu fontconfig \
    && rm -rf /var/cache/apk/*

CMD ["bash"]
```


```
# 这里将镜像构建为 jicki/openjdk:1.8-alpine

[root@localhost compose]# docker build -t="jicki/openjdk:1.8-alpine" .


[root@localhost compose]# docker run --rm jicki/openjdk:1.8-alpine java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (IcedTea 3.12.0) (Alpine 8.212.04-r0)
OpenJDK 64-Bit Server VM (build 25.212-b04, mixed mode)
```



# 创建项目镜像

> 项目镜像按照每个项目运行参数不同会有不同，也可以创建一个基于 ARG 变量的 项目镜像模板 通过传参数 构建出不一样的项目镜像.
>
> 根据以下 镜像模板，这里 ARG 只有一个,PROJECT_BUILD_FINALNAME,  这里代表 java 项目包build 生成的 名称, 在build  java 项目生成 jar 包以后，获取到这个 名称, 传递到这个 参数既可.

```
# 项目 dockerfile 如下:


FROM jicki/openjdk:1.8-alpine

ARG PROJECT_BUILD_FINALNAME

ENV TZ 'Asia/Shanghai'
ENV PROJECT_BUILD_FINALNAME ${PROJECT_BUILD_FINALNAME}


COPY build/libs/${PROJECT_BUILD_FINALNAME}.jar /${PROJECT_BUILD_FINALNAME}.jar

CMD ["bash","-c","java -jar /${PROJECT_BUILD_FINALNAME}.jar"]

```


```
# 将项目 dockerfile 上传到 相对应的 git Project 里去

```



# Gradle 配置修改

> java 基于 Gradle 的项目, 我们知道都有一个配置文件, build.gradle , 这里因为以上我们需要获取 项目镜像模板 中的 PROJECT_BUILD_FINALNAME 参数, 这里需要修改一下这个文件, 使 Gradle build 项目 jar 的时候将这个参数传递到env文件里去.



```
# 完整的 build.gradle 如下:



buildscript {
        repositories {
                mavenCentral()
        }
        dependencies {
                classpath 'org.springframework.boot:spring-boot-gradle-plugin:1.5.21.RELEASE'
        }
}

plugins {
        id 'java'
}

apply plugin: 'org.springframework.boot'

group = 'java'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '1.8'

repositories {
        mavenCentral()
}

dependencies {
        compile 'org.springframework.boot:spring-boot-starter'
        compile 'org.springframework.boot:spring-boot-starter-web'
        compile 'org.springframework.boot:spring-boot-starter-thymeleaf'
        testCompile 'org.springframework.boot:spring-boot-starter-test'
}

bootRepackage {

    mainClass = 'java.java.TestProject.TestProjectApplication'
    executable = true

    doLast {
        File envFile = new File("build/tmp/PROJECT_ENV")

        println("Create ${archivesBaseName} ENV File ===> " + envFile.createNewFile())
        println("Export ${archivesBaseName} Build Version ===> ${version}")
        envFile.write("export PROJECT_BUILD_FINALNAME=${archivesBaseName}-${version}\n")

        println("Generate Docker image tag...")
        envFile.append("export BUILD_DATE=`date +%Y%m%d%H%M%S`\n")
        envFile.append("export IMAGE_NAME=jicki/test:`echo \${CI_BUILD_REF_NAME} | tr '/' '-'`-`echo \${CI_COMMIT_SHA} | cut -c1-8`-\${BUILD_DATE}\n")
        envFile.append("export LATEST_IMAGE_NAME=jicki/test:latest\n")
    }
}

```

> 以上配置 主要添加 bootRepackage 下面的所有选项, 这里如果懂 gradle 以及一点 bash 的话, 很容易就看懂以上配置.




# 创建Gitlab-ci 的配置

> Gitlab  CI 的配置文件, 为 .gitlab-ci.yml , 每个项目下面都需要一个 .gitlab-ci.yaml 做为自动化 ci 的配置文件.
> 
> 官方关于 ci yaml 文档为 https://docs.gitlab.com/ee/ci/yaml/




```
# 由于我们这里需要 build java gradle 项目, 在使用 自动化的时候, 我们还需要 用于构建 项目 jar 镜像 以及 docker build 的镜像, 镜像也需要预先创建好, 并上传到仓库中.
```



```
# 用于 gradle build  jar 的 dockerfile


FROM jicki/openjdk:1.8-alpine

ENV GRADLE_VERSION=4.5
ENV GRADLE_HOME=/opt/gradle
ENV GRADLE_FOLDER=/root/.gradle

# Change to tmp folder
WORKDIR /tmp

# Download and extract gradle to opt folder
RUN wget https://downloads.gradle.org/distributions/gradle-${GRADLE_VERSION}-bin.zip \
    && unzip gradle-${GRADLE_VERSION}-bin.zip -d /opt \
    && ln -s /opt/gradle-${GRADLE_VERSION} /opt/gradle \
    && rm -f gradle-${GRADLE_VERSION}-bin.zip \
    && ln -s /opt/gradle/bin/gradle /usr/bin/gradle \
    && apk add libstdc++

# Create .gradle folder
RUN mkdir -p $GRADLE_FOLDER

# Mark as volume
VOLUME  $GRADLE_FOLDER
```


```
# 构建镜像

[root@localhost ~]# docker build -t="jicki/gradle:alpine" .


[root@localhost ~]# docker run --rm jicki/gradle:alpine gradle -version

------------------------------------------------------------
Gradle 4.5
------------------------------------------------------------

Build time:   2018-01-24 17:04:52 UTC
Revision:     77d0ec90636f43669dc794ca17ef80dd65457bec

Groovy:       2.4.12
Ant:          Apache Ant(TM) version 1.9.9 compiled on February 2 2017
JVM:          1.8.0_212 (IcedTea 25.212-b04)
OS:           Linux 4.4.181-1.el7.elrepo.x86_64 amd64

```


```
# 用于 docker build image 的 dockerfile


FROM docker

ARG TZ="Asia/Shanghai"

ENV TZ ${TZ}

RUN apk upgrade --update \
    && apk add bash curl tzdata wget ca-certificates git \
    && ln -sf /usr/share/zoneinfo/${TZ} /etc/localtime \
    && echo ${TZ} > /etc/timezone \
    && rm -rf /var/cache/apk/*

CMD ["/bin/bash"]

```


```
[root@localhost ~]# docker build -t="jicki/build:alpine" .



[root@localhost ~]# docker run --rm jicki/build:alpine docker version
Client: Docker Engine - Community
 Version:           18.09.6
 API version:       1.39
 Go version:        go1.10.8
 Git commit:        481bc77
 Built:             Sat May  4 02:33:34 2019
 OS/Arch:           linux/amd64
 Experimental:      false

```





```
# 以下为 这个项目的一个 .gitlab-ci.yaml 文件



# 调试开启
#before_script:
#  - pwd
#  - env

cache:
  key: $CI_PROJECT_NAME-$CI_COMMIT_REF_NAME-$CI_COMMIT_SHA
  paths:
    - build

stages:
  - build
  - deploy

auto-build:
  image: jicki/gradle:alpine
  stage: build
  script:
    - gradle --no-daemon clean assemble
  tags:
    - JavaProject

deploy:
  image: jicki/build:alpine
  stage: deploy
  script:
    - source build/tmp/PROJECT_ENV
    - echo "Build Docker Image ==> ${IMAGE_NAME}"
    - docker build -t ${IMAGE_NAME} --build-arg PROJECT_BUILD_FINALNAME=${PROJECT_BUILD_FINALNAME} .
#    - docker push ${IMAGE_NAME}
    - docker tag ${IMAGE_NAME} ${LATEST_IMAGE_NAME}
#    - docker push ${LATEST_IMAGE_NAME}
#    - docker rmi ${IMAGE_NAME} ${LATEST_IMAGE_NAME}
#    - kubectl --kubeconfig ${KUBE_CONFIG} set image deployment/test test=$IMAGE_NAME
  tags:
    - JavaProject
  only:
    - master
    - develop
    - /^chore.*$/
```



```
# 提交 .gitlab-ci.yaml 文件


# 查看 gitlab  ci 信息
```

![图15][15]

![图16][16]

![图17][17]

![图18][18]

![图19][19]

![图20][20]


```
# 查看生成的 项目镜像

[root@localhost testproject]# docker images
REPOSITORY                    TAG                              IMAGE ID            CREATED             SIZE
jicki/test                    master-4e4fcd5c-20190620122155   f2a4e4ab1ff1        6 minutes ago       160MB
```


  [1]: https://jicki.cn/img/posts/gitlab/1.png
  [2]: https://jicki.cn/img/posts/gitlab/2.png
  [3]: https://jicki.cn/img/posts/gitlab/3.png 
  [4]: https://jicki.cn/img/posts/gitlab/4.png 
  [5]: https://jicki.cn/img/posts/gitlab/5.png 
  [6]: https://jicki.cn/img/posts/gitlab/6.png 
  [7]: https://jicki.cn/img/posts/gitlab/7.png 
  [8]: https://jicki.cn/img/posts/gitlab/8.png 
  [9]: https://jicki.cn/img/posts/gitlab/9.png 
  [10]: https://jicki.cn/img/posts/gitlab/10.png 
  [11]: https://jicki.cn/img/posts/gitlab/11.png 
  [12]: https://jicki.cn/img/posts/gitlab/12.png 
  [13]: https://jicki.cn/img/posts/gitlab/13.png 
  [14]: https://jicki.cn/img/posts/gitlab/14.png 
  [15]: https://jicki.cn/img/posts/gitlab/15.png 
  [16]: https://jicki.cn/img/posts/gitlab/16.png 
  [17]: https://jicki.cn/img/posts/gitlab/17.png 
  [18]: https://jicki.cn/img/posts/gitlab/18.png 
  [19]: https://jicki.cn/img/posts/gitlab/19.png 
  [20]: https://jicki.cn/img/posts/gitlab/20.png 

