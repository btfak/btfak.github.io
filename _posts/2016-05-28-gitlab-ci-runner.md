---
layout: post
title: 怎样使用gitlab-ci-runner
date: 2016-05-28 15:21:35
summary: gitlab-ci-runner是gitlab官方出品的持续集成工具，简单来说就是当你的代码触发了某个持续集成任务，运行在主机上的gitlab-ci-runner就会执行预先设计好的脚本
categories: CI
---



这篇文件简单介绍，怎样安装使用gitlab-ci-runner，执行持续集成任务。



## 介绍

gitlab-ci-runner是gitlab官方出品的持续集成工具，简单来说就是当你的代码触发了某个持续集成任务，运行在主机上的gitlab-ci-runner就会执行预先设计好的脚本。比如我们设计好，某个项目的develop分支有更新时，发送一段脚本到runner，这段事先写好的脚本，主要工作是进入这个项目的目录，git pull，编译，重启。这样就你只是推送了代码，但已经实现了简单的自动化部署。



## 安装

官方的安装文档在这里，非常简单，因为runner是采用golang编写的，所以你本质上只是下载了一个可执行文件，没有任何依赖项。按照你的平台选择即可：

https://gitlab.com/gitlab-org/gitlab-ci-multi-runner

不必多说。

注：gitlab-ci-multi-runner和gitlab-ci-runner就是一个东西，两个名字而已。



## 连接

安装好以后，运行起来的效果应该类似这样

![](/images/gitlab-ci-multi-runner-01.jpg)

注意，接下来的命令不要使用sudo，在linux环境下，如果使用sudo，在执行任务时会带来权限上的问题。



### 注册runner

接下来执行`gitlab-ci-multi-runner register`，进入交互式的页面，依次输入各个参数即可

![](/images/gitlab-ci-multi-runner-02.jpg)



### 激活runner

执行`gitlab-ci-multi-runner verify`

![](/images/gitlab-ci-multi-runner-03.jpg)



### 运行runner

执行`gitlab-ci-multi-runner run &`

此时runner就已经运行起来，等待着gitlab发送任务。



此时在gitlab后台的runner页面中应该可以看到绑定成功的runner

 ![](/images/gitlab-ci-multi-runner-04.jpg)



## 为项目绑定runner

在gitlab进入某个需要进行持续集成的项目目录，`setting > runners`

为这个项目绑定runner，`ENABLE FOR THIS PROJECT`

![](/images/gitlab-ci-multi-runner-05.jpg)



## 设置runner变量

在某个项目的`setting > variables`中，设置全局变量，注意这里设置的变量，所有项目都可以读取到。

![](/images/gitlab-ci-multi-runner-06.jpg)