# jenkins build android



> 利用jenkins 对 android 做自动化build  apk, 并使用 360 进行加固。


# 环境配置


## 配置 java 环境

```
# 下载 oracle jdk 1.8



```



## 配置 android sdk


```
# 下载 https://developer.android.com/studio/index.html

# 拉倒最下面下载 sdk-tools

如：

https://dl.google.com/android/repository/sdk-tools-linux-4333796.zip


# 创建目录

mkdir -p /opt/android-sdk

cd /opt/android-sdk

wget https://dl.google.com/android/repository/sdk-tools-linux-4333796.zip


# 创建 env

vi /etc/profile

# anddrid env
export ANDROID_HOME=/opt/android-sdk
export PATH=$ANDROID_HOME/tools:$ANDROID_HOME/platform-tools:$PATH

# 生效配置

source /etc/profile


# 更新 sdk 工具

cd /opt/android-sdk

bin/sdkmanager --update

bin/sdkmanager --licenses

# 执行 打包所需的 版本

tools/android update sdk

tools/android update sdk --no-ui --filter build-tools-25.0.2,android-25,extra-android-m2repository

```



## 配置 jenkins

```
# 配置一个全局变量

系统管理 --> 系统设置 --> 全局属性 --> 环境变量 --> 新增

键	ANDROID_HOME
值	/opt/android-sdk

```


### 添加360 加固

```
# 这里保存官方文档 以防忘记

http://bbs.360.cn/forum.php?mod=viewthread&tid=6294457


# 下载 Linux 版本的加固

链接：http://pan.baidu.com/s/1jI7OyME （提取码：y3k1）

```


