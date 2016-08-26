---
layout: post
title: 使用Micro构建有弹性的、高容错的应用
date: 2016-05-15 15:21:35
summary: 构建分布式系统是很有挑战性的，这毫无疑问。虽然我们已经解决了很多工程上的问题，我们仍然重复的在构建许多模块。目前，由于我们开始了更高级别的抽象，虚拟机到容器技术，适应新的语言，作用于云计算，都对微服务提出了要求。总有一些事情需要我们不断的去学习，怎样构建高性能的、高容错的系统仍然是下一波的技术浪潮
categories: 微服务
---

这是一系列介绍[Micro](http://github.com/micro)框架的文章的第七篇，我将会把作者的博客翻译成中文，推广Micro这个微服务框架。



自发表上篇文章以来，已经有一段时间了，但我们仍然努力在为Micro添砖加瓦，现在确实需要还债了，让我们一次解决掉。



如果你想先了解一下 [Micro](https://github.com/micro/micro)工具箱，阅读以前的文章即可。



构建分布式系统是很有挑战性的，这毫无疑问。虽然我们已经解决了很多工程上的问题，我们仍然重复的在构建许多模块。目前，由于我们开始了更高级别的抽象，虚拟机到容器技术，适应新的语言，作用于云计算，都对微服务提出了要求。总有一些事情需要我们不断的去学习，怎样构建高性能的、高容错的系统仍然是下一波的技术浪潮。



重复与创新之间的战争从未停止，但我们需要做一些事情，通过云计算、容器技术、微服务来缓解我们的痛苦。



## 动机

我们为什么这样做？为什么我们持续的重新构建同样的模块，为什么我们持续的尝试解决大规模、容错性和分布式系统的问题？



我脑海中出现的是『*bigger, stronger, faster*』，或者是『*speed, scale, agility*』，你可能经常从C级别的管理人员口中听到这个说法。但关键的是，确实存在这样的需求，需要我们构建更加高性能和要弹性的系统。



在互联网的早期，只有数千或者数万用户在线，随着时间的推移，我们看到开始加速，现在我们面对的是数十亿用户和数十亿的设备。我们需要学习怎样为目前的情况构建系统。



上一代的人也许记得[C10K problem](http://www.kegel.com/c10k.html)，我不确定我们现在处在什么阶段，但我想我们现在谈论的是百万级的并发。世界上最大的技术公司，在10年前就已经解决了这个问题，也有了模式来构建这样大规模的系统。但剩下的其他人仍然在学习。



像Amazon，Google，Microsoft现在提供给我们的云计算平台，对大规模部署是有益的。但我们仍然努力在搞清楚，怎样编写应用程序，可以高效的利用这些大规模的资源。你也许这些天听说了容器的编排，微服务、云计算很多了。工作在很多层面上推进着，当我们完全确定了我们的模式，确定了需要解决的问题，我们的Micro就能作为工业级的产品发布了，这还需要一段时间。



许多公司现在求助的问题是『我该怎样构建可扩展的、高容错的系统？』但目前对这些重要的问题，有帮助的回答是很少的。



我该怎样编写可扩展的、高容错的系统？



Micro看起来通过专注于微服务的必要的软件开发工具，定位了问题。我们会详细的谈谈怎样帮助你构建有弹性的、高容错的系统，我们从client端开始。



## 客户端

客户端在go-micro中是用于发起请求的模块，如果你已经构建过微服务挥着SOA架构，你会知道重要的一部分时间和执行过程是花在调用其他服务获取相关信息上面。



然而在巨大的应用中，关注点主要在接受请求返回内容，在微服务世界中，更像是取回或者发布内容。



这里是精简过的go-micro中client的接口，有最重要的三个方法Call, Publish  和 Stream。

```go
type Client interface {
	Call(ctx context.Context, req Request, rsp interface{}, opts ...CallOption) error
	Publish(ctx context.Context, p Publication, opts ...PublishOption) error
	Stream(ctx context.Context, req Request, opts ...CallOption) (Streamer, error)
}
```

```go
type Request interface {
	Service() string
	Method() string
	ContentType() string
	Request() interface{}
	Stream() bool
}
```

Call和Stream是用来做同步通信请求，Call返回一个单一的结果，而Stream是一个双向的流式连接，与另一个服务维持着，其中任何消息都可以发进来也可以发出去。Publish用于发布异步的消息，通过broker，但我们今天不讨论它。



客户端是怎样工作的，前面的文章已经讨论过了。翻看以前的文章即可。



我们只是特别的讨论一些重要的内部细节。



客户端使用RPC层，结合broker，codec，register，selector和transport来提供丰富的组合。分层的架构非常重要，所以我们可以把单个的组件进行分离，减少了整体的复杂性，也提供了插件化的能力。



## 为什么客户端重要?

客户端本质上抽象了与服务端之间的有弹性的、高容错的通信过程。像另一个服务发起请求看起来是非常直接的，但有几种情况是可能潜在的发生失败的。



下面我们开始了解这些功能和他们是怎样运作的。



## 服务发现

在分布式系统中，服务因为各种原因会频繁的加入和脱离集群。网络隔离、机器故障、调度等等。我们并不真正想关心它们。



当我们像另一个服务发起请求，我们通过名字识别服务，并允许客户端通过服务发现获取到服务的一系列实例，得到各个实例的地址和端口。服务在启动时在服务发现中心进行注册，在退出时进行注销。

![](https://blog.micro.mu/assets/images/discovery.png)

正如我们提到的，任何类型的问题都会出现在分布式系统，服务发现也不例外。所以我们依赖于经过严格测试的分布式服务发现系统，例如consul、etcd和zookeeper，使用它们存储服务的信息。



它们都使用基于Paxos的Raft算法来进行网络选举，这解决了我们的一致性问题。通过运行一个3到5个节点的集群，我们可以容忍大部分的系统故障，为客户端提供稳定可靠的服务。



## 节点选择

现在我们可靠的把服务名字解析到了一堆地址列表。我们怎样选择其中的一个进行调用呢?这就是go-micro中的selector发挥作用的地方。它基于register模块构建，提供负载均衡策略，比如轮询或者随机，也提供过滤、缓存和黑名单的功能。



这里是定义的接口：

```go
type Selector interface {
	Select(service string, opts ...SelectOption) (Next, error)
	Mark(service string, node *registry.Node, err error)
	Reset(service string)
}

type Next func() (*registry.Node, error)
type Filter func([]*registry.Service) []*registry.Service
type Strategy func([]*registry.Service) Next
```



## 负载策略

当前的策略是非常简单直接的，当Selector被调用时，它从register获取到服务，然后创建一个Next函数，从节点池中选择出符合要求的节点。



客户端会调用这个Next函数，根据负载均衡策略，获取到下一个符合要求的节点，并发出请求。如果这个请求失败了，而且重试次数大于1，它会使用相同的程序，获取下一个节点，再次调用。



这里是有很多不同的策略的，比如轮询、随机、最少连接、权重等等。负载均衡策略对于分布式系统是必不可少的。



## 缓存

虽然有一个可靠的服务发现系统是很好的，但每次请求都去查询一次并不高效。如果你想象一个大规模的系统，每个服务都这样做，很容易就会使服务发现系统过载。这会让整个系统不可用。



为了避免这种情况，我们可以使用缓存。大部分的服务发现系统提供了一个监听更新的机制，一般来说叫做Watcher。不是去轮询服务发现系统，而是等待事件发送给我们。go-micro 的Registry提供了Watch的概念。



我们已经编写了一个带缓存的selector，它把服务缓存在内存中。如果缓存中不存在时，它会去服务发现系统查找，缓存，并用于之后的请求。如果watch事件收到了，缓存模块会与register进行更新。



首先，通过移除服务查找，大大的提高了性能。这也提供了一定的容错，万一服务发现系统宕机了呢？我们仍然有一点偏执，害怕缓存由于节点发生故障而被污染了，所以节点都维持着合适的TTL。



## 黑名单

下面介绍一下黑名单，注意一下Selector的接口有Mark和Reset方法。我们不同真正的保证，注册进来的节点都是健康的，所以我们需要做黑名单。



任何一个请求发送之后，我们都会跟踪它的结果。如果这个服务的实例出现了多次失败，我们就可以大体上把这个节点加入黑名单，并过滤掉它。



节点在回到节点池之前会在黑名单中会存在一段时间，这是很严格的，如果这个节点失败了我们就需要移除掉它。这样我们可以持续的返回成功的请求，不会有任何延迟。



## 超时与重试

Adrian Cockroft最近开始讨论在微服务架构中消失的组件，其中一个有意思的是，传统的超时和重试策略导致了雪崩效应。我建议你看看这个[演示](http://www.slideshare.net/adriancockcroft/microservices-whats-missing-oreilly-software-architecture-new-york#24)。这个演示把问题总结的特别好。

![](https://blog.micro.mu/assets/images/timeouts.png)

Adrian 在上面描述的是一种常见的情况，一个缓慢的响应会导致超时，然后客户端会触发重试。这事实上是一个请求链路，这创造了一系列的新请求，而旧有的请求仍然在处理中。这样的配置失误会导致大量服务的过载，造成的调用失败是很难回滚的。



在微服务世界，我们需要重新想想，处理重试和超时的策略。Adrian继续讨论了潜在的解决方案。其中一种方式是超时之后，在新的节点上发起请求。

![](https://blog.micro.mu/assets/images/good-timeouts.png)

在重试的这方面，我们已经在Micro中使用了。重试的次数可以进行配置，如果你调用了一个失败的节点，客户端会在新的节点发起重试。



超时经常被深思熟虑，但事实上经常从传统的静态超时设置开始。直到Adrian演示了他的想法，超时策略变得很清晰了。



预算型超时策略现在也内置在Micro中，让我们看看它是怎样工作的。



第一个调用设置了超时，每个调用链上的请求都会消耗整体的超时时间。如果时间为0了，我们就会停止请求或重试，并返回调用链。



按照Adrian提到的，提供动态的预算型超时是非常好的，避免了不必要的雪崩。



更远一点来说，下一步应该是移除任何类型静态的超时。服务的响应时间根据环境的不同，请求的不同是不同的。这应该是动态的SLA，根据当前的状态进行调整，但这些事会留在未来解决。



## 连接池

连接池是构建可扩展系统的很重要部分，我们很快就看到了没有连接池的局限性，经常导致文件描述符数量达到限制，导致端口用尽。



目前有个进行中的[PR](https://github.com/micro/go-micro/pull/86)为go-micro增加了连接池，由于Micro插件化的特性，把连接池放在transport的上层很重要，这样HTTP，NATS，RabbitMQ等等，都会受益。



你也许会想，这是特定实现的，一些transport也许已经支持了。这是对的，不能总是保证在不同的transport下工作效果是一样的。通过把这个放置于上层，我们减少了transport模块的复杂性。



## 其他?

确实有很多好用的东西是go-micro内置的，那么还有什么呢？我很高兴你这么问...


## 服务版本

我们有这个功能，这个功能在前面的文章也讨论过了。服务包括名字和版本，注册在服务发现系统。当一个服务从注册器中查询出来时，它的节点是按照版本分组的。这样一样，selector就可以根据版本，进行流量负载。

![](https://blog.micro.mu/assets/images/selector.png)

### 为什么版本很重要

当我们发布新版本时，这非常重要，它可以确保所有事情运作正常，这样才能把所有服务进行升级。新版本可以被部署到一个小型的节点上，客户端会自动的分发一定比例的请求到这个新的节点。通过结合一些编排系统比如Kubernetes，你可以非常有信心的部署，一旦有任何问题也可以回滚。



## 过滤

我们也有，selector是非常强大的，它有能力把过滤条件传递进去，对节点进行过滤。这在client端调用时可以传递参数。一些意见存在的过滤可以在[这里](https://github.com/micro/go-micro/blob/master/selector/filter.go)看到，比如metadata，endpoint和版本过滤。

### 为什么过滤重要

你也许有一些功能只在某些特定版本的服务上存在。需要将这些请求分发到这些特定版本的服务上。这是非常好的功能，特别是多个不同版本的服务在同时运行时。



另外一个有用的地方是，你想要根据地区对服务进行路由。通过设置数据中心的标签在服务上，你可以过滤出本地的节点。根据metadata进行过滤是非常强大的，希望有更多的应该能够把这个功能使用起来。



## 插件化的架构

Micro原生的插件化架构是你一次又一次听到的。这从设计的第一天就已经确定了。这是非常重要的，Micro提供模块来构建整个系统。有时候的运行会超出控制，但这些都可以改善。



### 为什么插件化很重要?

每个人对怎样构建分布式系统都有自己的想法，我们实际上是提供了一个方式，让人们能设计他们想要的解决方案。不仅如此，现在也有很多经过严格测试的工具，我们可以直接使用，而不是自己重写任何东西。



技术始终在进化，全新的、更好的工具每天都在出现。我们怎样避免止步不前，插件化的架构意味着我们可以使用目前的组件，未来也可以使用更好的组件进行替代。



### 插件

每个go-micro的特性都被设计成golang中的接口，通过这样做，我们可以实际上替换底层的实现，这几乎不需要进行代码改动。在大部分情况下，只需要简单的引用这个包，然后在启动时加入参数就可以了。



在[go-plugins](https://github.com/micro/go-plugins)有很多现成的插件可以使用。



go-micro目前提供了默认的consul作为服务发现系统，http作为transport，你也许会想要使用一些别的东西，或者实现自己的插件。我们已经有社区的贡献者分享了[Kubernetes](https://github.com/micro/go-plugins/tree/master/registry/kubernetes) 的注册插件和[Zookeeper](https://github.com/micro/go-plugins/pull/24)的注册插件。



### 我怎样使用插件

大部分时候，插件的使用类似这样：

```go
# Import the plugin
import _ "github.com/micro/go-plugins/registry/etcd"
```

```go
go run main.go --registry=etcd --registry_address=10.0.0.1:2379
```

如果你想要看更多的细节，参考之前讨论 Micro on NATS的文章。



## wrappers

客户端和服务端都支持中间件的概念，称为wrapper。通过支持中间件，我们可以增加在请求和返回的业务逻辑前面或者后面，添加自定义的逻辑。



中间件是很容易理解的概念，数以千计的库在使用它。在处理崩溃、限制并发、认证、日志、记录等场景下，很容易发现它的妙处。

```go
# Client Wrappers
type Wrapper func(Client) Client
type StreamWrapper func(Streamer) Streamer

# Server Wrappers
type HandlerWrapper func(HandlerFunc) HandlerFunc
type SubscriberWrapper func(SubscriberFunc) SubscriberFunc
type StreamerWrapper func(Streamer) Streamer
```

## 我怎样使用Wrapper

这里是一个很直接的插件

```go
import (
	"github.com/micro/go-micro"
	"github.com/micro/go-plugins/wrapper/breaker/hystrix"
)

func main() {
	service := micro.NewService(
		micro.Name("myservice"),
		micro.WrapClient(hystrix.NewClientWrapper()),
	)
}
```

很容易对不对，我们发现很多公司在Micro上层，创建了自己的层级，用于初始化大部分默认的wrapper，所以所有的wrapper可以在同一个地方进行添加。

现在我们看看wrapper怎样让应用更有弹性，更能容错。



## circuit breaker

在SOA或者微服务世界，一个单独的请求可能会调用多个服务。大部分情况下，聚合许多信息返回给调用者。在成功的情况下，它运行的很好，但一旦发生错误，很容易触发雪崩式的错误，除了重启整个系统，很难恢复。



我们部分的解决了这个问题，通过在客户端使用重试机制和黑名单。但在一些情况下，我们需要组织客户端发起这个请求。



这里是circuit breaker怎样起作用的

![](https://blog.micro.mu/assets/images/circuit.png)

circuit breakers的理念非常直接，方法的执行是根据对失败的情况进行监控而进行封装的。当失败的情况达到一个阈值时，breaker开始起作用，任何未来的调用尝试都会返回错误，而不会调用实际的业务函数。在超时时间过了以后，进入一个半开状态。如果某个请求失败了，breaker会再次生效，如果成功了就会恢复到正常。



虽然内部的Micro客户端有一些容错特性，但我们不应该依赖它来解决所有问题。在wrapper中使用circuit breakers让我们受益很多。



## Rate Limiting

如果我们非常轻松的能响应世界上所有的请求，那就太好了，不过是在梦里。真实的世界不是这样工作的，执行一个查询需要消耗时间，资源的限制让我们只能响应一定数量的请求。



我们需要考虑限制发起请求的数量，或者限制并发响应的数量。这就是rate limiting发挥作用的地方。如果没有rate limiting，很容易会把资源耗尽，或者完全的让系统崩溃，让系统不能响应未来的任何请求。这经常是DDOS攻击的常见做法。



每个人都听说过，使用过或者实现过一些类型的rate limiting。这里有很多不同的算法，其中一种是[Leaky Bucket](https://en.wikipedia.org/wiki/Leaky_bucket) 算法，我们不会在这里展开，但值得一读。



我们可以使用Micro Wrapper和已经存在的库来使用这个函数，一个已经存在的库在[这里](https://github.com/micro/go-plugins/blob/master/wrapper/ratelimiter/ratelimit/ratelimit.go)。



我们实际上对YouTube实现的[Doorman](https://github.com/youtube/doorman)算法很感兴趣，一个全局的客户端rate limiter，我们也在寻求社区的其他实现。



## 服务端

前面介绍了很多客户端的很多特性和使用方式，那么服务端呢，第一件事需要注意的是Micro在go-micro的API、CLI、Sidecar等等都使用了客户端，客户端的特性让整个架构都收益，但我们仍然需要在服务端解决一些问题。



在客户端，register用于发现服务，服务端进行注册。当一个服务的实例运行起来时，它在服务发现系统进行注册，在退出时进行注销，关键词是『gracefully』。

![](https://blog.micro.mu/assets/images/register.png)

## 处理错误

在分布式环境中，我们都需要处理错误，我们需要容忍错误。register支持通过ttl来进行过期检查，一旦过期节点就是不健康的，底层的服务发现机制类型consul都支持这些功能。同时服务端也支持重新注册。这两者的结合意味着，节点可以在间隔时间内会重新注册，如果节点因为运行失败等等没有重新注册，register就会因为超时而认为节点不健康，将节点从register删除。



这种容错设计最先没有出现在go-micro中，但我们很快发现，在真实的世界中，因为服务的崩溃或其他原因程序退出时，并没有注销自己，所以需要这种ttl健康监测。



带来的影响就是，客户端需要处理一系列污染的请求。客户端也需要容错性，我们认为这样的功能设计排除了许多明显的问题。



## 增加更多功能设计

另一件需要注意的事情是，服务端也提供了能力来使用Wrapper和中间件，这意味着我们也可以做circuit breaking， rate limiting等其他一些特性。



服务端的这个功能故意的设计的简单，但插件化的特性可以让你自由扩展。



## 客户端与Sidecar

大部分我们讨论的都是存在于go-micro库中，这对所有的golang使用者是很好的，但其他人在想，我怎样从这里收益呢。



在最开始，Micro就包含了[Sidecar](https://github.com/micro/micro/tree/master/car)的设计理念，这是一个HTTP的代理，所有的go-micro的功能都内置其中，所以不管你用哪种语言构建你的应用，你都可以收益于我们在上面的讨论。

![](https://blog.micro.mu/assets/images/sidecar-rpc.png)

sidecar的设计模式并不是新东西，NetflixOSS有一个叫做[Prana](https://github.com/Netflix/Prana)的项目。Buoyant有一个叫[Linkerd](https://linkerd.io/)的项目。



Micro Sidecar使用了默认的go-micro客户端，如果你想使用其他功能，你可以添加参数，很容易的重新编译。我们会想办法在未来简化这个程序。



## 还有更多

这里讨论了许多[go-micro](https://github.com/micro/go-micro)的包和相关的工具，这些工具是很好的开始，但他们还不够。当你想要运行一个可扩展的、数以百计的微服务，处理数百万请求，仍然有许多问题需要解决。



### Platform

这是[go-platform](https://github.com/micro/go-platform)和[platform](https://github.com/micro/platform)发挥作用的地方了，micro解决了基础的组件，Platform则更进一步，解决运行可扩展的服务的更多问题。比如认证、分布式trace、同步锁、健康检查等等。



分布式系统需要一系列的工具用于提高容错性，Platform看起来会有帮助。通过提供一个分层的架构，我们可以在原始的核心工具上，构建任何自己需要的功能。



Platform仍然在早期，但Platform会解决大部分公司构建分布式平台时会遇到的问题。



## 总结

科技在快速的进化，云计算给了我们不受限制的扩展能力。设法与变化保持同步很难，构建一个可扩展的，高容错的系统在今天仍然具有很大的挑战。



但不应该用以前的方式解决问题，作为一个社区，我们可以互相帮助，适应这个新的环境，构建随着不断增长的需求而不断扩张的系统。



Micro在这个过程中看起来提供了一些帮助，通过提供工具，简化了构建和管理分布式系统。希望这个文章能示范我们处理这些问题的方式。



如果你想了解更多，请看这个[blog](https://blog.micro.mu/)，或者这个[repo](https://github.com/micro/micro)，Twitter可以关注[@MicroHQ](https://twitter.com/microhq)，Slack社区在[这里