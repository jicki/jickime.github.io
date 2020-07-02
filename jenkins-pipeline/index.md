# Jenkins Pipeline 语法


# Jenkins Pipeline 的核心概念

* Stage 阶段
  * 一个Pipeline可以划分成若干个Stage，每个Stage代表一组操作，例如：“Build”，“Test”，“Deploy”。 

* Node 节点
  * 一个Node就是一个Jenkins节点，或者是Master，或者是Agent，是执行Step的具体运行环境。 

* Step 步骤
  * Step是最基本的操作单元，小到创建一个目录，大到构建一个Docker镜像，由各类Jenklins Plugin提供，例如：sh 'make' 。



## Pipeline 变量传递

> 变量的传递非常重要, 不同的变量定义 必须用在不同阶段中, 否则报错.


1. 自定义变量(局部变量)


```
# 注意 变量的引用 一定要用双引号, 单引号识别为字符串。
def username = 'Jenkins'
echo "Hello Mr.${username}"
```


```
def username = 'Jenkins'
pipeline {
   agent none
   stages {
       stage('Stage1') {
           agent { label "jnlp-dev-android" }
           steps {
               timestamps {
                   echo '这是 jnlp-dev-android 第一个被执行的 stage.'
                   echo "Hello Mr.${username}"
               }
           }
       }
       stage('Stage2') {
            agent { label "jnlp-test-java" }
            steps {
                timestamps {
                    echo "在 agent jnlp-test-java 上执行的并行任务 1."
                    echo "Hello Mr.${username}"
                    echo "在 agent jnlp-test-java 上执行的并行任务 1 结束."
                }
           }
        }
   }
}
```


2. 环境变量(局部)


```
withEnv(['MY_HOME=/opt']){
    sh 'echo $MY_HOME'
}
```


```
pipeline {
   agent none
   stages {
       stage('Stage1') {
           agent { label "jnlp-dev-android" }
           steps {
               timestamps {
                   echo '这是 jnlp-dev-android 第一个被执行的 stage.'
                   withEnv(['MY_HOME=/opt']){
                       sh 'echo $MY_HOME'
                   }
               }
           }
       }
       stage('Stage2') {
            agent { label "jnlp-test-java" }
            steps {
                timestamps {
                    echo "在 agent jnlp-test-java 上执行的并行任务 1."
                    echo "在 agent jnlp-test-java 上执行的并行任务 1 结束."
                }
           }
        }
   }
}
```



3. 环境变量(全局)


```
environment {MY_HOME='/opt'}
echo "MY_HOME is ${env.MY_HOME}"
```


```
pipeline {
   environment {MY_HOME='/opt'}
   agent none
   stages {
       stage('Stage1') {
           agent { label "jnlp-dev-android" }
           steps {
               timestamps {
                   echo "这是 jnlp-dev-android 第一个被执行 开始."
                   echo "MY_HOME is ${env.MY_HOME}"
                   echo "这是 jnlp-dev-android 第一个被执行 结束."
               }
           }
       }
       stage('Stage2') {
            agent { label "jnlp-test-java" }
            steps {
                timestamps {
                    echo "在 agent jnlp-test-java 第二个被执行 开始."
                    echo "MY_HOME is ${env.MY_HOME}"
                    echo "在 agent jnlp-test-java 第二个被执行 结束."
                }
           }
        }
   }
}
```



4. 参数化构建(全局)

  * 参数化构建需用安装额外的插件

```
parameters {string(name:'Jenkins',defaultValue:'Hello',description:'How should I greet the world')}
ehco "${params.Greeting} World!"
```




## Pipeline 一些例子


### stage 间变量传递

```
pipeline {
   agent none
   stages {
       stage('Stage1') {
           agent { label "jnlp-dev-android" }
           steps {
               script {
                   echo "这是 jnlp-dev-android 第一个被执行 开始."
                   FOO = "test2"
                   env.BAR = "bar2"
                   echo "env.BAR = ${BAR}" // print BAR = bar2
                   echo "FOO = ${FOO}" // print FOO = test2
                   echo "这是 jnlp-dev-android 第一个被执行 结束."
               }
           }
       }
       stage('Stage2') {
            agent { label "jnlp-test-java" }
            steps {
                script {
                    echo "在 agent jnlp-test-java 第二个被执行 开始."
                    echo "env.BAR  =  ${BAR}" // print BAR = bar2
                    echo "FOO  = ${FOO}" // print FOO = test2
                    echo "在 agent jnlp-test-java 第二个被执行 结束."
                }
           }
        }
   }
}

```


### stage 中修改 全局环境变量

* 修改后只作用于 stages 这一阶段

```
pipeline {
   agent none
   environment {
       FOO = "bar"
   }
   stages {
       stage('Stage1') {
           agent { label "jnlp-dev-android" }
           steps {
               script {
                   echo "这是 jnlp-dev-android 第一个被执行 开始."
                   withEnv(["FOO=newbar"]) {
                       echo "FOO = ${env.FOO}" // print  FOO = newbar
                   }
                   echo "这是 jnlp-dev-android 第一个被执行 结束."
               }
           }
       }
       stage('Stage2') {
            agent { label "jnlp-test-java" }
            steps {
                script {
                    echo "在 agent jnlp-test-java 第二个被执行 开始."
                    echo "FOO = ${env.FOO}" // print  FOO = bar
                    echo "在 agent jnlp-test-java 第二个被执行 结束."
                }
           }
        }
   }
}
```



