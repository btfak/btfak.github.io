---
layout:     post
title: 基于消息队列NATS构建微服务
date: 2016-04-11 15:21:35 
summary: NATS是一个开源的消息系统，或者说消息队列。NATS的作者是Derek Collison,Apcera的作者。它起源于VMWare，最开始是一个ruby的系统。后来使用golang进行重写，逐步的成为了一个高扩展性的高性能消息系统
categories: 微服务
---

这是一系列介绍[Micro](http://github.com/micro)框架的文章的第四篇，我将会把作者的博客翻译成中文，推广Micro这个微服务框架。



这篇文章我们会讨论基于[NATS](http://nats.io/)使用[Micro](https://github.com/micro/micro)。讨论包括了服务发现，同步通信和异步通信。如果需要了解Micro，请翻看以前的文章。



## NATS是什么?

[NATS](http://nats.io/)是一个开源的消息系统，或者说消息队列。NATS的作者是Derek Collison， [Apcera](https://www.apcera.com/)的作者。它起源于VMWare，最开始是一个ruby的系统。后来使用golang进行重写，逐步的成为了一个高扩展性的高性能消息系统。



## 为什么是NATS?

为什么不是呢?过去我使用过很多消息队列，很明显NATS是鹤立鸡群的，在过去消息系统似乎成了企业的灵丹妙药了，结果是每个人参与的每个系统都离不开消息系统。导致了很多无效的承诺、高昂的研发投入，制造出比它解决的更多的麻烦。



NATS，相比之下，就十分专注，在难以置信的简洁之下，解决了性能问题，解决了高可用行问题。它的口号是『always on and available』，使用了一种『fire and forget』的消息模式。它简单、专注、轻量的特性使它在微服务的候选中脱颖而出。我们相信，在服务间的消息传递领域，它很快会变成主要的候选者。



NATS能提供什么?

- 高性能、高扩展

- 高可用

- 极致的轻量级

- 一次部署

  ​

NATS不支持什么？

- 持久化
- 事务
- Enhanced delivery modes
- Enterprise queueing

NATS是否适合Micro呢？我们接着讨论



## Micro on NATS

[Micro](https://github.com/micro/micro) 采用插件化的架构设计，用户可以替换底层的实现，而不更改任何底层的代码。每个[Go-Micro](https://github.com/micro/go-micro)框架的底层模块定义了相应的接口， [registry](https://godoc.org/github.com/micro/go-micro/registry#Registry) 是作为服务发现，[transport](https://godoc.org/github.com/micro/go-micro/transport#Transport)作为同步通信， [broker](https://godoc.org/github.com/micro/go-micro/broker#Broker)作为异步通信。



为每个组件创建一个插件很简单，只需要实现这些组件的接口即可。我们会在未来的文章里详细讨论，怎样写插件，如果你想详细了解NATS的插件或者其他的类似etcd作为服务发现，kafka作异步通信，rabbitmq作同步通信，在这里可以看到： [github.com/micro/go-plugins](https://github.com/micro/go-plugins)



Micro on NATS本质上是实现一系列的插件，将NATS消息系统整合到Micro的框架中来。通过为Go Micro提供不同的插件，我们可以创造出很多灵活的组合。



在实践过程中，不能一套系统打天下，Micro on NATS的可以灵活的定义各种模型。



下面我们会讨论一下NATS插件怎样在transport，broker和registry下工作。



## Transport

![](https://blog.micro.mu/assets/images/request-response.png)

transport 是 go-micro定义的同步通信接口，它使用了泛型的语法描述，类似其他golang代码，比如`Liesten,Dial,Accept`。这些理念和模式在类似TCP和HTTP这样的同步通信中很容易理解，但是在消息系统中要怎样兼容呢？通信过程中，连接是与消息队列建立的，而不是服务本身。为了达到目的，我们使用消息系统中的topics和channels来创造虚连接。



它是怎样工作起来的？



一个服务使用`transport.Listen`来监听消息，这会与NATS之间建立连接。当`transport.Accept`被调用时，一个唯一的topic会被创建并订阅。这个唯一的topic地址会作为服务的地址，注册到go-micro的注册器中。每个收到的消息会被虚连接处理，如果一个连接已经存在，而且回复的地址是一样的，我们会简单的把消息积压在连接上。



客户端通过`transport.Dial`来连接服务端，其实它是建立了与NATS的连接，然后创建了唯一的topic并且订阅这个topic。这个topic用于接收服务端的返回。任何时候客户端发送消息到服务端时，它都会把回复地址设置成这个topic。(译注：服务端的地址是固定的，而客户端的每个请求，都会生成一个客户端地址，唯一的请求与唯一的地址实现一一关联)



当任何一边想要关闭连接时，简单的调用`transport.Close`就会断开与NATS的连接。

![](https://blog.micro.mu/assets/images/nats-transport.png)



使用transport插件

引用插件

```go
import _ "github.com/micro/go-plugins/transport/nats"
```

启动时传入参数

```go
go run main.go --transport=nats --transport_address=127.0.0.1:4222
```

或者直接在代码中创建

```go
transport := nats.NewTransport()
```

go-micro的transport接口：

```go
type Transport interface {
    Dial(addr string, opts ...DialOption) (Client, error)
    Listen(addr string, opts ...ListenOption) (Listener, error)
    String() string
}
```



## Broker

![](https://blog.micro.mu/assets/images/pub-sub.png)

broke是go-micro的异步通信接口，它定义了泛型的、高等级的通用接口。NATS本身作为消息系统，是可以作为消息broker的。这里只有一个警告：nats并不持久化消息。虽然在某些场景有风险，但我们相信NATS可以也应该用于go-micro的broker。持久化并不是必须的，它提供了高扩展的发布和订阅架构。



NATS通过Topics和Channels的理念，非常直接的实现了发布和订阅机制。这里并没有其他工作需要做。消息可以被发布到任何异步的fire and forget manner。通过NATS的Queue Group，订阅方可以通过channel的名字就行订阅。实现消息自动的分发的多个订阅者。

![](https://blog.micro.mu/assets/images/nats-broker.png)

使用broker插件

引入包

```go
import _ "github.com/micro/go-plugins/broker/nats"
```

(译注：注意此时引用的是broker中的nats包，上面引用的是transport中的nats包)

通过参数启动

```go
go run main.go --broker=nats --broker_address=127.0.0.1:4222
```

也可以直接启动

```go
broker := nats.NewBroker()
```

go-micro的broker需要实现的接口：

```go
type Broker interface {
    Options() Options
    Address() string
    Connect() error
    Disconnect() error
    Init(...Option) error
    Publish(string, *Message, ...PublishOption) error
    Subscribe(string, Handler, ...SubscribeOption) (Subscriber, error)
    String() string
}
```



## Register

![](https://blog.micro.mu/assets/images/service-discovery.png)

注册器是go-micro定义的服务发现接口，你也许想过。用消息系统做服务发现？这能工作吗？事实上它不仅能工作，还工作的很好。许多人在服务发现中使用消息系统，避免了不一致的发现机制。这是因为消息队列通过topics和channels实现了路由，Topics定义为服务的名字可以作为路由的key，订阅topics的服务自动实现了负载均衡。



Go-micro把服务发现和传输机制看做两个不同的领域，任何时候，客户端向服务发起请求，都会首先在注册器中根据服务的名字查询到正在运行的服务节点，然后客户端通过transport与节点进行通信。



一般来说，最常见的存储服务发现信息的方式是通过分布式key-value store，比如zookeeper、etcd等等。正如你了解到的，NATS闭市一个分布式的KV store，我们将做一下改动。。。



广播查询！



广播查询正如你想象的那样，服务都会监听一个特点的topic，任何需要服务发现信息的都需要订阅这个topic，然后对这个topic做一个反馈。



因为我们并不知道有多少服务正在运行，我们对响应时间设置一个上限。这是一个粗糙的服务发现机制，但因为NATS本身的高扩展性和高性能，它运行的反而非常好。它也间接的提高了过滤服务节点的响应时间。未来我们会试着提高底层的实现。



我们总结一下它是怎么工作的：

1. 创建一个reply topic并订阅
2. 向广播topic发起查询，附带上reply topic
3. 监听回复，超过限制时间就注销订阅
4. 聚合响应并返回结果



![](https://blog.micro.mu/assets/images/nats-registry.png)



使用register插件

引入包

```go
import _ "github.com/micro/go-plugins/registry/nats"
```

通过参数启动

```go
go run main.go --registry=nats --registry_address=127.0.0.1:4222
```

或者直接在代码中设置

```
registry := nats.NewRegistry()

```

go-micro中的register接口

```go
type Registry interface {
    Register(*Service, ...RegisterOption) error
    Deregister(*Service) error
    GetService(string) ([]*Service, error)
    ListServices() ([]*Service, error)
    Watch() (Watcher, error)
    String() string
}
```



## 大规模在Micro中使用NATS

在上面的例子中，我们只是用了单个的NATS服务，但是在实际情况下，我们一般都是使用NATS集群来实现高可用和容错。你可以在[这里](http://nats.io/documentation/server/gnatsd-cluster/)看看更多NATS集群的知识。



Micro 在启动时可以接收一系列的参数，如环境变量等。如果直接使用client的库，也可以在创建时指定参数。



在目前云计算的架构下，我们过去的经验是，每个AZ都应该有集群。大部分的云计算公司，不同的AZ之间的延迟大概是3到5毫秒，集群的通信是没有任何问题的。当我们运行一个高可用的配置，你的系统必须要能在AZ宕机、甚至整个region崩溃时还能提供服务。我们不建议跨region部署集群。理想的高等级工具应该是用于管理多集群和多region系统。



Micro是一个极其灵活的微服务系统，它本身就被设计成运行在任何配置的任何地方。整个Micro世界是通过服务发现进行指导，服务的集群可以运行在机器集群中，AZ或者regions。再考虑到NATS集群，它可以让你构建高可用的服务。

![](https://blog.micro.mu/assets/images/region.png)



## 总结

NATS是一个高可用和高性能的消息系统，在我们的微服务生态系统中运行的很好。它与Micro配合的非常的好，我们可以把它用在register，transport，broker中，我们也已经全部实现了这些插件。



NATS在Micro中的使用只是一个示例，它展示了Micro高度的插件化架构，每个go-micro的包都可以被替换，底层不需要改动代码，上层也只需要改动非常少的代码。在未来我们还会看到更多的Micro on X，下一个可能是Micro on kubernetes。



希望这可以鼓励你来使用Micro on NATS,或者编写自己的插件并回馈给社区。



在这里看更多的NATS插件[github.com/micro/go-plugins](https://github.com/micro/go-plugins)



如果你想了解更多，请看这个[blog](https://blog.micro.mu/)，或者这个[repo](https://github.com/micro/micro)，Twitter可以关注[@MicroHQ](https://twitter.com/microhq)，Slack社区在[这里](http://slack.micro.mu/)
