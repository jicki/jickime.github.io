# jenkins gradle android



# Jenkins 构建 Android 包

* 使用 Jenkins 构建 Android  APK 包

  * 使用 腾讯 乐固 进行加固

  * 加固以后进行重新签名

  * 重新签名的 APK 上传到 蒲公英平台

  * 发送通知到 企业微信群中


## 安装部署 Jenkins

* 这一部分就省略了


## 配置 java 环境

* 部署 Java 环境 - 这一部分也省略了, 可选择 openjdk 或 oraclejdk

```
java -version
java version "1.8.0_241"
Java(TM) SE Runtime Environment (build 1.8.0_241-b07)
Java HotSpot(TM) 64-Bit Server VM (build 25.241-b07, mixed mode)
```





## 配置 gradle

* 部署 gradle 也省略了.

```
gradle -version

------------------------------------------------------------
Gradle 5.6.4
------------------------------------------------------------

Build time:   2019-11-01 20:42:00 UTC
Revision:     dd870424f9bd8e195d614dc14bb140f43c22da98

Kotlin:       1.3.41
Groovy:       2.5.4
Ant:          Apache Ant(TM) version 1.9.14 compiled on March 12 2019
JVM:          1.8.0_241 (Oracle Corporation 25.241-b07)
OS:           Linux 4.4.215-1.el7.elrepo.x86_64 amd64
```


## 配置 Android SDK

```
# 创建目录

mkdir -p /opt/android

cd /opt/android

wget https://dl.google.com/android/repository/sdk-tools-linux-4333796.zip

unzip sdk-tools-linux-4333796.zip

```


```
# 配置 env

vi /etc/profile

# anddrid env
export ANDROID_HOME=/opt/android
export PATH=$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools:$PATH




# 生效配置
source /etc/profile
```


### 更新 Android SDK


```
# 首先执行更新操作

sdkmanager --update

```


```
# 更新 SDK 版本

android update sdk


# 更新一些指定 tools, 根据自己的版本

android update sdk --no-ui --filter build-tools-29.0.3,android-29,extra-android-m2repository


```


```
# 最后更新 licenses

sdkmanager --licenses

```


### 配置腾讯云乐固SDK

```
# 下载 乐固 SDK

mkdir -p /opt/android/ms-client

cd /opt/android/ms-client

wget https://leguimg.qcloud.com/ms-client/java-tool/1.0.3/ms-shield.jar

```


```
# 升级版本

java –jar ms-shield.jar –update

```




## 配置 Jenkins


### 配置 全局变量

> 必须配置, 否则读取不到 sdk 文件


```
# 系统管理 --> 
    系统设置 --> 
      全局属性 --> 
        环境变量 --> 
          新增 -->
            键值对列表 -->
              键: ANDROID_HOME
              值: /opt/android
```


### 配置全局工具

* 配置 `JAVA` - 配置 `Gradle`




## 创建 Jenkins 工程


* 创建一个 自定义风格的工程


* 配置 构建 --> `shell`

  * 主要配置 乐固、签名以及上传蒲公英


```
#!/bin/sh

set -e
# 蒲公英 KEY
PGYKEY=
# 蒲公英 APIKEY
PGYAPIKEY=
# 腾讯云乐固认证 
SECRET=
# 腾讯云乐固认证 KEY
SECRETKEY=

echo "================加固开始======================="
rm -rf /opt/legu

mkdir -p /opt/legu

java -Dfile.encoding=utf-8 -jar /opt/android/ms-client/ms-shield.jar -sid ${SECRET} -skey ${SECRETKEY} -uploadPath /apk/release/app-product-release.apk  -downloadPath /opt/legu

echo "================加固结束======================="

ls -lt /opt/legu/app-product-release_legu.apk



echo "================开始签名======================="

/opt/android/build-tools/29.0.3/apksigner sign \
  --ks /opt/android/sign/xx.jks \
  --ks-key-alias xxx \
  --ks-pass pass:xxxxx \
  --key-pass pass:xxxxx \
  --out /opt/legu/app-product-release-sign_legu.apk /opt/legu/app-product-release_legu.apk

echo "================签名结束======================="

ls -lt /opt/legu/app-product-release-sign_legu.apk



echo "================上传蒲公英====================="

  echo "upload online apk to 蒲公英"

  RESULT=$(curl -F "file=@/opt/legu/app-product-release-sign_legu.apk" -F "uKey=${PGYKEY}" -F "_api_key=${PGYAPIKEY}" https://qiniu-storage.pgyer.com/apiv1/app/upload)

echo "================上传蒲公英结束====================="

echo "================下载地址=========================="

# 返回的是 json 格式的 所以格式化一下
echo $RESULT|jq .

```


* 构建后的信息

```

================下载地址==========================
{
  "code": 0,
  "message": "",
  "data": {
    "appKey": "xxxxxxx",
    "userKey": "xxxxxxxxx",
    "appType": "2",
    "appIsLastest": "1",
    "appFileSize": "xxxxxxx",
    "appName": "jicki",
    "appVersion": "1.8.0",
    "appVersionNo": "13",
    "appBuildVersion": "55",
    "appIdentifier": "xxx.xx.xxx",
    "appIcon": "d9b32400cd42dcc1",
    "appDescription": "",
    "appUpdateDescription": "",
    "appScreenshots": "",
    "appShortcutUrl": "jicki",
    "appCreated": "2020-04-24 18:20:08",
    "appUpdated": "2020-04-24 18:20:08",
    "appQRCodeURL": "https://www.pgyer.com/app/qrcodeHistory/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  }
}
Finished: SUCCESS
```



## 配置企业微信通知


* 企业微信中, 创建一个群聊

  * 进入群 -> 右击群标签, 添加群机器人

![微信机器人][1]


* 添加机器人名称

![微信机器人][2]


* 复制 Webhook

![微信机器人][3] 




* 安装 jenkins 插件

  * 插件名称为 `Qy Wechat Notification` , 在 jenkins 插件管理中心安装




* 配置机器人 Webhook

  * 进入 jenkins 的 Job

  * 选择 构建后的操作 --> 选择 `企业微信通知`


![微信机器人][4]

![微信机器人][5]




* 企业微信 构建信息

![微信机器人][6]








  [1]: https://jicki.cn/img/posts/jenkins/wechat-bot.png
  [2]: https://jicki.cn/img/posts/jenkins/wechat-bot2.png
  [3]: https://jicki.cn/img/posts/jenkins/wechat-bot3.png
  [4]: https://jicki.cn/img/posts/jenkins/wechat-bot4.png
  [5]: https://jicki.cn/img/posts/jenkins/wechat-bot5.png
  [6]: https://jicki.cn/img/posts/jenkins/wechat-bot6.png


