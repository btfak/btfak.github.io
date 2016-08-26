---
layout: post
title: 怎样自动化部署微服务
date: 2016-05-30 13:21:35
summart: 这篇文章将讨论我们团队在实际开发和运维过程中，怎样基于gitlab的CI系统和supervisor，进行微服务的自动化部署
categories: CI
---



这篇文章将讨论我们团队在实际开发和运维过程中，怎样基于gitlab的CI系统和supervisor，进行微服务的自动化部署。



## CI

持续集成的重要性不用多说，能够显著提高开发、部署和运维效率，但非侵入式的持续集成架构是很难的，此处分享我们在小型的开发团队中采用的持续集成方案。



### Gitlab

我们采用自建gitlab进行代码版本管理，通过docker进行搭建极其容易。目前的gitlab CI系统已经非常完善，可以针对特定的代码分支执行持续集成任务。

怎样安装gitlab-ci-runner参考[这篇文章](/2016/05/28/gitlab-ci-runner/)



### Supervisor

测试环境采用supervisor进行进程监控，保证应用挂掉之后能重启，且能正常的杀掉老的进程。



## Example

以目前实现的一个监控组件monitor作为示例，分享怎样实现持续集成。

monitor是一个标准的micro服务，假设代码存放目录为

```bash
$GOPATH/src/git.domain.com/micro/monitor
```

monitor的代码目录如下

 ![micro-monitor-code](/images/micro-monitor-code.jpg)

其中与部署相关的是两个文件

`.gitlab-ci.yml`是一个名字固定的文件，gitlab会根据这个文件名，来找到这个文件，将其中的内容根据分支设定，发送给runner执行，比如我们的文件内容是:

```bash
develop:
    script:
    - cd ${GOPATH}/src/git.xxx.com/micro/monitor  //进入代码目录
    && git pull && git checkout develop && go build  //更新代码、切换到develop分支、编译
    && mkdir -p logs  //创建logs文件夹
    && supervisorctl -c ../../supervisord.conf update  //更新supervisor配置文件
    && supervisorctl -c ../../supervisord.conf restart monitor //重启服务
    tags:
      - micro   //将任务分发给有micro这个tag的runner执行
    only:
      - develop   //只监听develop分支
```

注意，文件里的GOPATH是一个变量，这个变量在gitlab后台的monitor工程中设置，它是全局的，不用每个工程都设置，在某个工程设置一次即可。

具体参考[这篇文章](/2016/05/28/gitlab-ci-runner/)



``supervisord.conf`是supervisor的配置文件，supervisor的安装等等内容请参考[官网](http://supervisord.org/)

内容如下：

```bash
[program:monitor]
directory=%(here)s/micro/%(program_name)s
environment=RUN_MODE=%(ENV_RUN_MODE)s
command=%(here)s/micro/%(program_name)s/%(program_name)s
process_name=%(program_name)s
autorestart = true
startretries = 3
redirect_stderr=true
stdout_logfile_maxbytes = 20MB
stdout_logfile_backups = 20
stdout_logfile=%(here)s/micro/%(program_name)s/logs/%(program_name)s.log
```

内容很简单，就是进去某个设定好的目录，执行某个命令。



实际效果：

![micro-monitor-code](/images/ci-01.jpg)

![micro-monitor-code](/images/ci-02.jpg)



## 总结

本质上就是gitlab+supervisor的组合，需要一些细节的设计，开发的项目需要增加两个文件。部署的服务器需要设计好路径结构，否则可能会找不到文件。实际操作过程中有疑问欢迎给我发邮件。自动化部署如果想要运行起来，还需要另一个方面的配合 — 不同环境的配置文件问题，本地环境、开发环境、测试环境、生产环境的配置大部分情况下都不一样，怎样智能的读取响应环境的配置文件？这个问题我们也有自己的解决方案，在接下来的文章中我会进一步说明。