---
title: docker搭建gitlab ci环境
date: 2018-10-13 15:28:55
comments: false
categories: devops
tags: [devops]
description: 如何用docker+gitlab搭建CI？
---

# 背景介绍
随着devops的越来越火，CI(Continuous Integration)、CD(continuous Deployment)越来越流行，有规模的公司都会有一套持续集成的环境。比较主流的开源的组件如jenkins。提供github开源项目的CI组件travis。笔者这里介绍的是gitlab ci，gitlab8.0之后提供了自身的CI功能，这使得配置变得简单。

# 概念介绍
- gitlab server：gitlab的基础环境，比如你的代码需要提交到这里，它可以集成ci，运行runner
- gitlab ci：ci默认是随gitlab-server一起安装的。
- gitlab runner：每一个项目要指定自己的任务执行，这个任务相当于runner，当然也可以使用公共的runner。

# 为什么使用docker
docker的概念这里不赘述，之所以使用，当然是应为它简单，举个例子，如果你自己安装一个gitlab，你要安装gitlab依赖的redis、postgresql等环境，docker会打包安装好。想想这是一件多么简单的事情。

# 安装步骤
## linuxbrew
作为mac的用户，homebrew是必不可少的工具，那么像centos的系统有没有类似的工具，这里推荐linuxbrew，使用方式和homebrew一样，官网：[http://linuxbrew.sh/](http://linuxbrew.sh/)  

1. 安装

```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/Linuxbrew/install/master/install.sh)"
```

2. 环境设置

```
test -d ~/.linuxbrew && PATH="$HOME/.linuxbrew/bin:$HOME/.linuxbrew/sbin:$PATH"
test -d /home/linuxbrew/.linuxbrew && PATH="/home/linuxbrew/.linuxbrew/bin:/home/linuxbrew/.linuxbrew/sbin:$PATH"
test -r ~/.bash_profile && echo "export PATH='$(brew --prefix)/bin:$(brew --prefix)/sbin'":'"$PATH"' >>~/.bash_profile
echo "export PATH='$(brew --prefix)/bin:$(brew --prefix)/sbin'":'"$PATH"' >>~/.profile
```

## 安装docker

就是这么简单！！！等待安装完成就好。 

```
brew install docker
```

检查是否成功

```
docker version
```
## 安装gitlab&gitlab runner
安装之前查看image

```
docker search gitlab
docker search gitlab-runner
```
安装

```
docker pull gitlab
docker pull gitlab-runner
```
## 启动gitlab

```
sudo docker run --detach \
    --hostname 192.168.1.201 \
    --publish 44301:443 --publish 8001:80 --publish 2201:22 \
    --name gitlab \
    --restart always \
    --volume /home/w/gitlab/config:/etc/gitlab \
    --volume /home/w/gitlab/logs:/var/log/gitlab \
    --volume /home/w/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:latest
```
192.168.1.201是你本机的ip，注意不要写localhost或者127.0.0.1
/home/w是你本地映射docker环境的文件，要先建立出这个文件夹。

## 配置端口映射

```
vim /home/w/gitlab/config/gitlab.rb
```

```
external_url 'http://192.168.1.201:8001'
nginx['listen_port'] = 80
gitlab_rails['gitlab_ssh_host'] = '192.168.1.201'
gitlab_rails['gitlab_shell_ssh_port'] = 2201
```

```
# (imageId通过docker ps -a查看)
docker restart imageid
```

## 注册gitlab-runner
官网：[https://docs.gitlab.com/runner/install/docker.html](https://docs.gitlab.com/runner/install/docker.html)

```
docker run --rm -t -i -v /home/w/gitlab-runner/config:/etc/gitlab-runner --name gitlab-runner gitlab/gitlab-runner register
```
这里会一步一步提示需要的信息，详情见官网，
如果是specific runner 则token在项目下的ci页面内

如果是共享的runner，则token在管理员的runner配置内Admin-Area-> Overview -> Runners
也就是可以注册多个runner

```
docker run --rm -t -i -v /home/w/gitlab-runner2/config:/etc/gitlab-runner --name gitlab-runner2 gitlab/gitlab-runner register
```

## 启动gitlab-runner

```
docker run -d --name gitlab-runner --restart always \
  -v /home/w/gitlab-runner/config:/etc/gitlab-runner \
  -v /var/run/docker.sock:/var/run/docker.sock \
  gitlab/gitlab-runner:latest
```
现在可以部署你自己的项目到gitlab了！
