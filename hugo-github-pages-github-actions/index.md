# Github Actions 构建 Hugo


{{< figure src="/img/post-bg-alitrip.jpg" title="Hugo" >}}

# Hugo

* Hugo 是基于 Go 语言开发的静态网站构建程序.



## 流程


{{< mermaid >}}
graph LR;
    A[小炒肉] -->|Git Push| B(Github hugo source)
    B -->|自动构建流程| C{Github Actions}
    C -->|生成静态文件并推送| D(Github jicki.github.io)
{{< /mermaid >}}



## 创建Github仓库 


1. 创建 `hugo` 仓库, 并设置为 Private  此仓库用于存放 hugo 源代码.

![图片1][1]




2. 创建 `jicki.github.io` 仓库, 此仓库用于 Github Pages .


![图片2][2]



## 创建 SSH Key

* `SSH Key` 主要用于 `hugo` 仓库 自动构建  `Github Actions` 生成静态文件后认证自动推送到 `jicki.github.io` 仓库.


```shell

ssh-keygen -t rsa -b 4096 -C "jicki@qq.com" -f key/id_rsa_hugo

```


```shell
[root@jicki key]# ls -lt
total 8
-rw------- 1 root root 3243 Jun 30 16:39 id_rsa_hugo
-rw-r--r-- 1 root root  738 Jun 30 16:39 id_rsa_hugo.pub
```


## 配置 SSH Key


* 配置 `jicki.github.io` 仓库 Deploy keys .

  * 使用 `id_rsa_hugo.pub` 文件

![图片3][3]




* 配置 `hugo` 仓库 Secrets

  * 使用 `id_rsa_hugo` 文件


![图片4][4]




## 初始化 hugo


* 这里需要初始化一次 hugo 所以需要有 hugo 客户端, 请自行解决.



### clone 仓库

```shell

git clone https://github.com/jicki/hugo

```

```shell

cd hugo

# 生成 hugo 源码

hugo new site .


```

### 配置 hugo

* 此处请自行配置 hugo 的相关设定, 如安装 主题, 配置 `config.toml` 等.




### push 代码到 仓库


```shell
git add -A

git commit -m "Add hugo code"

git push
```



## 配置 Github Actions

![图片5][5]

![图片6][6]


![图片7][7]



* `main.yml` 文件内容

```shell

name: Deploy Hugo Site to Github Pages on Master Branch
on:
  push:
    branches:
      - master
jobs:
  build-deploy:
    runs-on: ubuntu-18.04
    steps:
      - uses: actions/checkout@v1  # v2 does not have submodules option now
       # with:
       #   submodules: true

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v2
        with:
          hugo-version: '0.73.0'
          # extended: true

      - name: Build
        run: hugo --minify

      - name: Deploy
        uses: peaceiris/actions-gh-pages@v3
        with:
          deploy_key: ${{ secrets.ACTIONS_DEPLOY }} # 这里的 ACTIONS_DEPLOY 则是上面设置 Private Key 的变量名
          external_repository: jicki/jicki.github.io # Pages 远程仓库
          publish_dir: "public"  # hugo 默认生成目录为 public 这里一定要配置
          keep_files: false # remove existing files
          publish_branch: master  # deploying branch
          commit_message: ${{ github.event.head_commit.message }}

```




## Actions Workflows


* 每次推送代码以及更新文件到 `hugo` 仓库会自动触发 `Github Actions` 

![图片8][8]




## 测试访问


* 浏览器访问 `https://jicki.github.io/`  查看具体的效果.


![图片9][9]




  [1]: https://jicki.cn/img/posts/hugo/1.png
  [2]: https://jicki.cn/img/posts/hugo/2.png
  [3]: https://jicki.cn/img/posts/hugo/3.png
  [4]: https://jicki.cn/img/posts/hugo/4.png
  [5]: https://jicki.cn/img/posts/hugo/5.png
  [6]: https://jicki.cn/img/posts/hugo/6.png
  [7]: https://jicki.cn/img/posts/hugo/7.png
  [8]: https://jicki.cn/img/posts/hugo/8.png
  [9]: https://jicki.cn/img/posts/hugo/9.png

