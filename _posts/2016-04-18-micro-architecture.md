---
layout:     post
title: Micro的架构与微服务的设计模式
date: 2016-04-18 15:21:35
summary: 在过去的几个月，我们已经有了很多有关micro架构的疑问和微服务的设计模式的问题，今天我们讨论一下这两个话题
categories: 微服务
---

这是一系列介绍[Micro](http://github.com/micro)框架的文章的第五篇，我将会把作者的博客翻译成中文，推广Micro这个微服务框架。



在过去的几个月，我们已经有了很多有关micro架构的疑问和微服务的设计模式的问题，今天我们讨论一下这两个话题。



## 关于Micro

[Micro](https://github.com/micro/micro)是一个微服务工具箱，它有自己固有的设计模式，但插件化的架构可以让底层的实现很轻易的被替换。



micro专注于定位微服务构建过程中的最基本的需求，并通过精密的设计来满足这些需求。



查看过往的文章可以了解微服务的理念和Micro的特性。



## 关于工具箱

[Go Micro](https://github.com/micro/go-micro) 是一个用golang编写的，插件化的RPC框架。它提供了基础的库，比如服务发现、客户端负载均衡、编解码、同步异步通信等。



[Micro API](https://github.com/micro/micro/tree/master/api)是一个API网关，用于将外部的HTTP请求路由到内部的micro服务上。它有单一的接入点，可以通过反向代理或者http转换成RPC来访问。



[Micro Web](https://github.com/micro/micro/tree/master/web)是一个web仪表盘，也是作为micro web应用的反向代理。我们相信web应用也应该是一个微服务，在微服务世界里也应该是第一等公民。它表现的很像Micro API但也有单独的特性比如websocket



[Micro Sidecar](https://github.com/micro/micro/tree/master/car)使用http服务，提供了go-micro的所有特性。虽然我们喜欢golang来构建微服务，但你也许想使用其他语言。所以Sidecar提供了一种其他语言的应用接入Micro世界的方式。



[Micro CLI](https://github.com/micro/micro/tree/master/cli)是一个简单直接的命令行接口，用于与你的服务交互。他也可以使用你的Sidecar作为代理来连接服务。



上面是很简单的介绍，下面我们更加深入一些。



## RPC,REST,Proto...

第一件你想到的事情是，为什么是RPC，而不是REST? 在内部服务间的通信上，我们相信RPC是更合适的。或者更明确一点，RPC使用protobuf做编码，通过protobuf定义API。这种方式把两个需求结合起来了：一个需求是需要明确定义的API接口，另一个需求是高性能的消息编解码。RPC是非常直接的通信方式。



我们在这个选择上并不孤独。



Google是protobuf的创造者，在内部通过gRPC这个框架，大量的使用RPC调用。Hailo也从RPC/protobuf的组合中收益很多，不仅是系统性能，开发速度也提高很多。Uber选择开发自己的RPC框架，名字叫[TChannel](http://uber.github.io/tchannel/)



个人而言我认为未来的API将会使用RPC进行构建，因为它们结构化的格式、高效的编解码提供了定义良好的API和高性能的通信。



## HTTP to RPC,API...

事实上，我们在web上RPC还有很长的路要走。在内部RPC的表现是完美的，但在面对外部请求比如网站、手机app的接口等等，就是另外一回事了。我们需要面对这个，这就是为什么Micro需要一个API网关，用来接受并转换http请求。



API网关在微服务架构中是一个常见的模式。它作为一个单一的接入点，外部世界的请求，通过它进行路由分发。它让HTTP API可以由背后的很多服务所组成。



micro的API网关使用路径到服务的解决方案，因此不同的请求路径，对应了不同的服务。比如/user => user api，/order => order api。



这里有一个例子。一个请求的路径是/comstomer/orders，这个请求会被转发到go.micro.api.customer这个服务，会使用 Customer.Orders.这个方法进行处理。



![](https://blog.micro.mu/assets/images/api.png)



你也许会问，API服务到底是怎样的？我们下面来讨论一下不同类型的服务。



## 服务的类型

微服务的关键理念就是业务的拆解，这是从unix的设计哲学中得到的启示：『doing one thing and doing it well』，因为这个原因，我们认为不同的服务需要有逻辑上和架构上的区别，以实现自己不同的任务。



我们知道这些理念并没有什么太大的新意，但在一些非常大而且成功的公司，它们的实践取得了成功。我们的目标是传播这些开发理念，并通过工具来进行指导。



目前我们定义了下面的几种服务。



### API

通过micro api运行，API 服务在你的架构中处于关键位置，大部分作用是接受外部世界的请求并分发到内部的服务上。你可以通过micro api提供的反向代理REST模式进行访问，也可以通过RPC接口进行访问。



### WEB

通过micro web运行，web服务专注于服务html请求，构建仪表盘。micro web反向代理http和websocket，目前只有这两种协议支持，未来也许会增加。



### SRV

这是后台的RPC服务，他们的目标是为你的系统提供核心的功能，大部分并不是公开的接口。你仍然可以通过micro api和micro web，使用/rpc接入点进行访问。这种接入方式直接使用go-micro的client进行调用。

![](https://blog.micro.mu/assets/images/arch.png)

按照过去的经验，我们发现这样的架构设计非常强大。可以被扩展到数以百计的服务。通过把它整合到Micro架构中，我们发现它为微服务的开发提供了非常好的基础。



## Namespacing

你也许会想，怎样区分micro api或者micro web以及服务呢。我们通过命名空间进行拆分。通过命名的前缀，我们可以很清晰的看到，某个服务是哪种类型的。这很简单但很高效。



micro api也会把/customer这样的请求路径定位到go.micro.api.customer服务。



默认的命名空间是：

- API - go.micro.api
- WEB - go.micro.web
- SRV - go.micro.srv

你应该把它设置成你的域名，比如com.example.api。这些都可以进行配置。



## 同步和异步

你经常听说微服务是很灵活的模式。大多数来说，微服务是关于创造事件驱动的架构，以及设计通过异步通信的方式响应的服务。



Micro把异步通信作为微服务构建中的第一等公民。事件通过异步消息，可以被任何人消费并作出反应，搭建一个新服务不需要对系统的其他部分作出任何更改。这是一种强大的设计模式，因为这个原因，我们在go-micro中定义了[Broker](https://godoc.org/github.com/micro/go-micro/broker#Broker)接口。

![](https://blog.micro.mu/assets/images/pub-sub.png)



异步和同步通信在Micro中是分离开的。 [Transport](https://godoc.org/github.com/micro/go-micro/transport#Transport) 接口用于构建服务之间的点对点的通信。go-micro中的client和server基于transport来进行请求和返回RPC调用，提供了双向的通信流。

![](https://blog.micro.mu/assets/images/request-response.png)



在构建系统时，两种通信方式都应该使用，但关键是理解在什么场景下应该用什么类型的通信方式。在大部分情况下并没有好坏之分，我们需要权衡处理。



一个broker和异步通信的典型使用方式是这样：监听系统通过broker对服务的事件历史进行记录。

![](https://blog.micro.mu/assets/images/audit.png)

在这个例子中，每个服务的每个API在被调用时，都会把事件上报到监听topic，监听系统会订阅这个topic，并把他们存储到时间序列的数据库中。在admin管理平台可以看到任何用户的操作历史。



如果我们通过同步通信做，监听系统直接面对巨大的请求数。如果监听系统宕机了，我们就丢失了这些数据。通过把这些事件发布到broker，我们可以异步的持久化这些数据。这是一种微服务中常见的事件驱动设计模型。



## 我们怎样定义微服务?

我们已经讨论了很多Micro能为微服务提供的工具箱，也定义了服务的类型。但还没有真正讨论，到底什么是微服务。



微服务与其他应用的区别到底在哪里，微服务为什么叫微服务。



现在有很多不同的定义，但有两条适合大部分微服务系统。

```
Loosely coupled service oriented architecture with a bounded context
```

```
An approach to developing a single application as a suite of small services, each running in its own process and communicating with lightweight mechanisms 
```

微服务的哲学与unix也类似

```
Do one thing and do it well
```

我们认为微服务是这样一种应用程序：专注于单一的业务，并通过明确定义的API对外提供服务。



看看我们在社交网络中怎样使用微服务：

![](https://blog.micro.mu/assets/images/facebook.png)



其中一种是流行的MVC架构，在MVC世界中，每个实体代表了一个模型，模型又是作为数据库的抽象。模型之间也许有一对多或者多对多的关系。controller模块负责处理请求，接受model模块返回的数据，并把数据传输到view层，进行渲染，最后输出给用户。



在微服务架构中，面对同样的例子。每个模型实际上是一个服务，通过API进行服务间通信。用户请求，数据的集合以及渲染是通过一系列不同的web服务进行处理的。每个服务有自身的专注点，当我们需要增加一个新特性时，我们只需要把关联的服务进行修改，或者直接写一个新的服务。分离的理念提供了大规模开发的模式。



## 版本

![](https://blog.micro.mu/assets/images/versioning.png)

开发真实世界的软件时，版本是非常重要的。在微服务世界里，严格的把API和业务逻辑分离到许多不同的服务上，因为这个原因，服务的版本控制是核心的工具的很重要的一部分。可以让我们在流量很大时也能进行升级。



在go-micro中，服务定义了名字和版本，[Registry](https://godoc.org/github.com/micro/go-micro/registry#Registry)模块返回服务的列表，根据版本把节点进行了区分。这里是service的接口定义。

```go
type Service struct {
	Name      string
	Version   string
	Metadata  map[string]string
	Endpoints []*Endpoint
	Nodes     []*Node
}
```

板块控制需要与[Selector](https://godoc.org/github.com/micro/go-micro/selector#Selector)结合起来，selector是客户端的负载均衡机制，通过selector的策略实现请求根据版本进行分发。



selector是非常强大的接口，我们根据不同的路由算法，比如随机、轮询、根据标签、响应时间等等。



通过使用默认的随机负载算法，再加上版本控制算法，我们就可以进行灰度发布。

![](https://blog.micro.mu/assets/images/selector.png)



在未来，我们会尝试实现一个全局的负载策略，根据历史的趋势进行选择，可以根据版本，设置不同的百分比，并动态的为服务增加标签。



## 大规模扩展

上面的介绍的版本系统，是大规模扩展服务时的基本模式。register存储了服务的注册信息，我们通过selector实现了路由和负载均衡。



按照` doing one thing well`的理念，扩展架构也应该是简单、明确定义的API、分层次的架构。通过创造这些工具，我们可以构建更加可靠的系统，专注于更高级别的业务需求。



这是Micro编写的基础理念，也是我们希望微服务开发者遵循的理念。



当我们在生产环境部署应用时，我们就需要构建可扩展、高容错、高性能的应用。云计算让我们可以进行不受限制的扩展，但是没有任何东西会一直正常运行。事实上，在构建分布式系统中，怎样对待运行失败的服务是非常重要的一方面，你在构建你的系统时，需要好好考虑。



在云计算的世界，我们想要在数据中运行错误，甚至多个数据中心运行错误的情况下，也能正常提供服务。在过去我们讨论的是冷热备份，或者是灾难恢复计划。在今天，最先进的技术公司，在全世界不停歇的运作，每个程序都会有多个备份，运行在世界上不同的数据中心。



我们需要向google，facebook，netflix和Twitter学习，即使在数据中心运行失败时，也要对用户提供服务，在多个数据中心运行失败时，也需要尽快恢复。



Micro可以让你构建这样的应用，通过插件化的架构，我们可以为不同的分布式系统，实现不同的工具箱。



服务发现和注册器是Micro的关键模块，它们可以用于发现在数据中心中的一系列服务，Micro API可以用于路由和负载一系列的服务。

![](https://blog.micro.mu/assets/images/regions.png)



## 总结

希望这篇文章清晰的讲解了Micro的架构，以及怎样实现可扩展的微服务设计模式。微服务首先是一种软件设计模式，我们可以通过工具实现基础、核心的功能，同时也能灵活组合其他设计模式。



因为Micro是一个插件化的架构，它强大的能力，可以实现不同的设计模式，在不同的场景中都能使用。比如你构建一个视频流的服务，你也许需要基于http的点对点服务。如果你对性能不敏感，你也许需要使用消息队列比如NATS或RabbitMQ。



使用Micro这样的工具进行开发是非常让人兴奋的。



如果你想了解更多，请看这个[blog](https://blog.micro.mu/)，或者这个[repo](https://github.com/micro/micro)，Twitter可以关注[@MicroHQ](https://twitter.com/microhq)，Slack社区在[这里](http://slack.micro.mu/)
