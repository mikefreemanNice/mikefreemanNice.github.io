---
title: gitlab-ci发布项目
date: 2018-10-14 20:18:08
comments: false
categories: devops
tags: [devops]
description: gitlab-ci如何发布项目？
---

# 前言
这篇文章是即上篇文章的续写[http://www.wowfree.cn/2018/10/13/gitlab01/](http://www.wowfree.cn/2018/10/13/gitlab01/)，gitlab作为代码的私有仓库，被大部分公司所使用，gitlab8.0了提供ci功能，为了简单，一般会剔除掉jenkins的引用，本文主要介绍如何通过gitlab-ci编译、打包、发布项目，本文以Java的springboot项目为例。

# 简单实现
首先你需要新建一个springboot的项目，创建过程这里不赘述，如果你想在gitlab-ci发布它，首先需要在它的根路径创建一个.gitlab-ci.yml文件。简单配置如下：

```
image: maven:latest

stages:
  - build
  - package
  - deploy

build:
  stage: build
  script:
    - echo "=============== 开始编译构建任务 ==============="
    - mvn compile
  tags:
    - springdemo

package:
  stage: package
  tags:
  - springdemo
  script:
  - echo "=============== 开始打包任务 ==============="
  - mvn clean package -Dmaven.test.skip=true
  only:
  - master

deploy:
  stage: deploy
  tags:
  - springdemo
  script:
  - echo "=============== 开始部署任务 ==============="
  only:
  - master
```

- image的选择是你在注册gitlab-runner时选择的
- tags是gitlab-runner的tag，可以通过gitlab页面配置是否需要该tag，默认是需要的
- only是执行哪个分支，这里可以通过它来实现不同环境的部署。

# 进阶
如何实现项目的发布，首先我们要考虑的是gitlab运行在docker上，我们需要通过免密登录的方式发布到另一台机器，如何配置成了难点.  
官网给出的方案：[https://docs.gitlab.com/ee/ci/ssh_keys/README.html#ssh-keys-when-using-the-docker-executor](https://docs.gitlab.com/ee/ci/ssh_keys/README.html#ssh-keys-when-using-the-docker-executor) 。  
假设我们当前机器的ip为192.168.1.201，我们想发布到192.168.1.202上，首先需要在202的机器上生成公钥和撕咬，
## 配置192.168.1.202免密登录


```

vim /etc/ssh/sshd_config

```
将RSAAuthentication和PubkeyAuthentication注释去掉

```
ssh-keygen -t rsa

```

在~/.ssh的路径下面会看到公钥和私钥，将公钥配置到gitlab的root用户下的ssh key中。

## 配置gitlab.yml文件

在.gitlab.yml文件加入如下配置

```
before_script:
  - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client git -y )'
  
  - eval $(ssh-agent -s)
  
  - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
  
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh

  - ssh-keyscan "$ip" >> ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts

```

SSH_PRIVATE_KEY是配置在gitlab项目中的环境变量，通过页面项目中setting -> CI/CD -> Variables中配置，
key是SSH_PRIVATE_KEY，value是192.168.1.202的私钥。

## 发布项目

```
before_script:
  - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client git -y )'
  
  - eval $(ssh-agent -s)
  
  - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add - > /dev/null
  
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh

  - ssh-keyscan "$ip" >> ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts 

stages:
  - build
  - deploy

build:
  image: maven:latest
  stage: build
  script:
    - echo "=============== 开始编译构建任务 ==============="
    - mvn compile
  tags:
    - springdemo

deploy:
  image: maven:latest
  stage: deploy
  tags:
  - springdemo
  script:
  - echo "=============== 开始部署任务 ==============="
  - mvn clean package -Dmaven.test.skip=true -Pdev -B && scp /builds/java/springclouddemo/web/target/web.jar ${username}@${ip}:/home/work/web.jar
  - ssh -T ${username}@${ip} sh /home/command/start.sh
  only:
  - master
```

username和ip为gitlab-ci的变量配置，

start.sh脚本如下

```
#! /bin/sh

WEB_APP="web"

CUSTOM_JVM_OFFLINE=" -Xmx2048m
             -Xms2048m
             -Xmn756m
             -XX:+UseConcMarkSweepGC
             -XX:+UseParNewGC
             -XX:+PrintGCDetails
             -XX:+PrintGCDateStamps
             -XX:+ExplicitGCInvokesConcurrent
             -XX:MetaspaceSize=256m
             -XX:MaxMetaspaceSize=256m
             -Dspring.profiles.active=dev
             -Dorg.jboss.logging.provider=slf4j
             -Dfile.encoding=UTF-8
             -Dsun.jnu.encoding=UTF-8
             -Djava.net.preferIPv6Addresses=false
             -Djava.io.tmpdir=/tmp
             -Dprism.pooldebug=true
             -Xrunjdwp:server=y,transport=dt_socket,address=8088,suspend=n"

CUSTOM_LOG_PATH_OFFLINE="/home/work/logs"


CUSTOM_JVM_ONLINE=" -Xmx4048m
             -Xms4048m
             -Xmn1512m
             -XX:+ExplicitGCInvokesConcurrent
             -XX:MetaspaceSize=256m
             -XX:MaxMetaspaceSize=256m
             -XX:+UseConcMarkSweepGC
             -XX:+UseParNewGC
             -XX:+PrintGCDetails
             -XX:+PrintGCDateStamps
             -Dspring.profiles.active=prod
             -Dorg.jboss.logging.provider=slf4j"

CUSTOM_LOG_PATH_ONLINE="/home/work/logs"

DEFAULT_LOG_PATH="/home/work/logs"


if [ -z ${LOG_PATH} ]; then
    LOG_PATH="${DEFAULT_LOG_PATH}"
fi
echo ${LOG_PATH}


DEFAULT_JVM=" -Xloggc:$LOG_PATH/$WEB_APP.gc.log.`date +%Y%m%d%H%M`
              -XX:ErrorFile=$LOG_PATH/$WEB_APP.vmerr.log.`date +%Y%m%d%H%M`
              -XX:HeapDumpPath=$LOG_PATH/$WEB_APP.heaperr.log.`date +%Y%m%d%H%M`
              -XX:+HeapDumpOnOutOfMemoryError
              -XX:+PrintGCDetails
              -XX:+PrintGCDateStamps"


function is_online() {
    # 得到主机名
    HOST_NAME=`hostname`

    if [[ ${HOST_NAME} =~ ".inf" ]] ; then
        echo false
    elif [[ ${HOST_NAME} =~ ".sankuai.com" ]]; then
        echo true
    else
        echo false
    fi
}

function init() {
    echo ${WEB_APP}
    PID_NUM=$(ps aux |grep "${WEB_APP}\.jar"|grep -v "grep"|awk '{print $2}')
    echo ${PID_NUM}
    if [ ${PID_NUM} ] ; then
        echo kill ${WEB_APP}.jar
        kill  ${PID_NUM}

        echo sleep 10
        sleep 10

        PID_NUM=$(ps aux |grep "${WEB_APP}\.jar"|grep -v "grep"|awk '{print $2}')
        if [ ${PID_NUM} ] ; then
            echo kill -9 ${WEB_APP}.jar
            kill -9 ${PID_NUM}
        fi
    else
            echo ${WEB_APP}.jar not running
    fi
}


function run() {
    echo starting...

    if [ -d "logs" ]; then
        echo rm logs
        rm -rf logs
    fi

    # 根据主机名判断是否是线上环境
    IS_ONLINE=`is_online`
    echo "IS_ONLINE:"${IS_ONLINE}

    # 环境设置
    if [ ${IS_ONLINE} = "true" ]; then
        CUSTOM_JVM="${CUSTOM_JVM_ONLINE} ${DEFAULT_JVM}"
        # 指定LOG_PATH为线上路径
        LOG_PATH="${CUSTOM_LOG_PATH_ONLINE}"
        ln -s ${CUSTOM_LOG_PATH_ONLINE} logs
    else
        CUSTOM_JVM="${CUSTOM_JVM_OFFLINE} ${JACOCO_ARGS} ${DEFAULT_JVM}"
        # 指定LOG_PATH为线下路径
        LOG_PATH="${CUSTOM_LOG_PATH_OFFLINE}"
        ln -s ${CUSTOM_LOG_PATH_OFFLINE} logs
    fi

    exec /home/w/java/default/bin/java ${CUSTOM_JVM} -jar /home/work/${WEB_APP}.jar  > output.log 2>&1 &

    echo run finished
}


init
run
```