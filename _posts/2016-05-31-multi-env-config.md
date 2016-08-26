---
layout: post
title: 多环境下的配置管理方案
date: 2016-05-31 13:21:35
summary: 在开发中，我们需要面对各种各样的环境，开发环境、测试环境、生产环境……并且，各个环境的参数和配置各不相同，比如数据库连接，服务器配置等。我们怎样在不同环境中调用正确的配置？
categories: go

---



在开发中，我们需要面对各种各样的环境，开发环境、测试环境、生产环境……



并且，各个环境的参数和配置各不相同，比如数据库连接，服务器配置等。我们怎样在不同环境中调用正确的配置？



## 通过配置文件

这是一种常见的思路，通过创建多个配置文件，但根据命名区分，比如开发环境为`develop-app.conf`，测试环境为`testing-app.conf`，生产环境为`production-app.conf`



我们通过在系统中设置环境变量`export ENV_MODE=develop`等等。在读取配置文件时，根据环境变量读取响应的配置文件。



这个方式易于使用，深得大家喜爱。但这个方案在集群扩大的一定程度时，会遇到一下几个主要问题：

- 假如有30~40个微服务需要连接数据库运行，这个量级在中小型团队中很常见了，如果我们需要更改数据库密码，我们不得不将数十个project逐个进行更新，非常不灵活。
- 代码与配置掺杂在一起，代码是许多开发人员都可以看到的，也很容易泄露，而生产环境的各种秘钥应该只有少数人有权限能看到。这对系统的安全有重大影响。


- 对于大量相同的配置（比如数据库配置），逻辑上我们应该存放在同一个地方，保证只有唯一可靠的数据来源。



对于这些问题，我们认为配置应该集中化管理。



集中化管理带来以下好处：

- 各个服务间相同的配置只需要维护一分数据，保证唯一性
- 各个环境的配置环境实现权限隔离，少数人拥有查看生产环境配置的权限
- 更改配置将变得简单，不影响服务本身



最简单的方案就是存储在`redis`中。KV的存储方式天然适合关联配置文件。但要完整的使用整个方案，需要做一些准备。



## 集中式配置管理

我们的基本思路是：将配置文件的值替换为占位符，在系统启动时，相应的工具将根据占位符到`redis`中查询到实际的值，替换回配置文件。

最初的配置文件是这样：

```
{
  "database_host":"127.0.0.1",
  "database_port":3306
}
```



现在我们的配置文件变成了这样：

```
{
  "database_host": "redis_hget global.mysql host",
  "database_port": "redis_hget global.mysql port"
}
```



## 读取配置

在启动时，我们通过这个工具：[https://github.com/gogap/env_json](https://github.com/gogap/env_json)

这样读取配置文件

```go
func main() {
    data, _ := ioutil.ReadFile("./db.conf")
    dbConf := DBConfig{}
    if err := env_json.Unmarshal(data, &dbConf); err != nil {
        fmt.Print(err)
        return
    }
    fmt.Println(dbConf)
}
```

这个工具，默认从`/etv/env_string.conf`读取`redis`的配置信息，当然你可以更改，更多细节参看说明文档。



在这个过程中，`env_json`首先会从`/etv/env_string.conf`读取到`redis`的配置信息。



典型的`/etv/env_string.conf`内容如下

```go
{
"storages": [{
        "engine": "redis",
        "options": {
            "db": 0,
            "password": "",
            "pool_size": 10,
            "address": "127.0.0.1:6379"
        }
    }]
}
```





连接上`redis`后，以上面的例子来说，将执行`hget global.mysql host `以及`hget global.mysql port `，将取到的值通过模板替换，更新到配置文件中，得到一个正常的`json`文本，剩下的就是通过`json`库把`json`内容解码到结构体中。



到目前为止，我们实现了从`redis`中读取并替换配置，那么我们写入配置的时候呢？



假如我们有数十个服务，我们难道需要逐个去`redis`中设置吗？我们怎样把这个流程自动化？



## 写入配置

我们需要另一个工具：[env_sync](https://github.com/gogap/env_strings/tree/master/env_sync)



我们存储配置文件其实是一个具体的git工程，比如开发环境是`develop_env`,生产环境是`production_env`，开发人员都可以编辑`develop_env`这个工程，少数人可以编辑`production_env`。



工程里的内容什么呢？

我们约定了这样的目录结构

```
├── env_develop
│   ├── components.accounts
│   │   └── data
│   ├── components.api
│   │   └── data
```

在工程中，有一系列的文件夹，文件夹中有一个叫data的文件。这样的目录结构会被[env_sync](https://github.com/gogap/env_strings/tree/master/env_sync)识别到，并转化成一系列的`redis`命令。

假如`global.mysql`文件夹下的`data`文件内容是

```
{
  "host":"127.0.0.1",
  "port":3306
}
```

转化出来的命令是：

```
hset global.mysql host 127.0.0.1
hset global.mysql port 3306
```

此过程与读取过程正好相反，同样的，[env_sync](https://github.com/gogap/env_strings/tree/master/env_sync)也是从`/etc/env_strings.conf`读取配置信息。与读取工具保持了统一。





## 总结

整体来看我们需要做几个工作

- 为各个环境维护一个配置文件project

- 安装env_sync，便于同步配置文件到redis

- 设置`/etc/env_strings.conf`

- 更改读取配置文件的代码，兼容env_json

  ​

再结合[自动化部署工具](http://lubia.cn/2016/05/30/deploy-micro/)，每次配置文件有更新时，我们就在线上环境自动同步到redis。



## 更多

还有一种需求时，配置文件会动态变化，而我们不想重启服务就读取到配置文件，那你需要[https://github.com/gogap/redconf](https://github.com/gogap/redconf)

这个工具可以实现对redis中数据的检测，如果数据发生变化，会触发回调，应用可以得到变化前后的值。 