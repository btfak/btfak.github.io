---
layout: post
title: Micro Bot - 微服务中的ChatOps
date: 2016-04-25 13:21:35
summary: 现在我知道你在想什么，现在有许多关于机器人的夸张说法。如果你对聊天机器人熟悉的话，你会知道这些都不是什么新说法，事实上最早的历史开始于Eliza。大众对机器人重新开始着迷，是因为我们发现了机器人有更多的功能，而不仅是简单的好玩。同时他们也提醒了我们下一代的人机交互接口会演变成什么样
categories: 微服务
---

这是一系列介绍[Micro](http://github.com/micro)框架的文章的第六篇，我将会把作者的博客翻译成中文，推广Micro这个微服务框架。



今天我想聊一下机器人。



## 机器人?真的吗...

现在我知道你在想什么，现在有许多关于机器人的夸张说法。如果你对聊天机器人熟悉的话，你会知道这些都不是什么新说法，事实上最早的历史开始于[Eliza](https://en.wikipedia.org/wiki/ELIZA)。大众对机器人重新开始着迷，是因为我们发现了机器人有更多的功能，而不仅是简单的好玩。同时他们也提醒了我们下一代的人机交互接口会演变成什么样。



从工程师的思维来看，机器人不仅是为了交谈的目的，他们在执行任务的时候，超出想象的好用。大部分的我们已经对ChatOps很熟悉了。Github创造了这个概念，推出了他们的 [Hubot](https://hubot.github.com/),这是一个聊天机器人，可以管理技术上和业务上的操作任务。



看看这篇Jesse Newland的演讲了解更多：[ChatOpts at GitHub](https://www.youtube.com/watch?v=NST3u-GjjFw)



Hubot和机器人看起来已经证明了，在技术公司他们是非常有用的，他们在运维和自动化方面成为了好用的助手。现在通过HipChat或者Slack操控机器人来执行任务，而以前我们是手动的执行一些脚本，这明显要强大的多。这对整个团队带来的价值是显而易见的，每个人都能看到你在做的事情，已经事情的结果。



## Micro服务怎样与ChatOps结合起来?

[**Micro**](https://github.com/micro/micro)，这个微服务工具箱，包括了一系列的服务，提供了接入点连接你正在运行的系统。API，Web控制台，CLI等等。他们都能与你的服务进行交互，观察你的服务的运行环境。在过去的几个月，这已经变得很清楚了，机器人是另外一种接入点，用于与你的服务进行交互与观察你的服务，这也是Micro世界的第一等公民。

这样一来

![](https://blog.micro.mu/assets/images/micro_bot.png)



首先我们要明确，Micro 机器人是处于非常早期的阶段，目前主要是通过CLI提供功能。我们现在不能说实现了ChatOps，但或许有一天可以呢...



Micro机器人包括了类似hubot的语法命令，已经一种实现的输入，比如Slack或者HipChat。这是粗糙的第一个版本，但我相信随着工作的投入，不久以后就能大大提供机器人的能力。希望社区也能加入进来。



Bot 包括了所有的CLI命令，也提供了Slack和HipChat的入口。我们的机器人目前运行在一个demo环境中，通过Micro Slack提供，在[这里](http://slack.micro.mu/)加入我们。



在最近的开发周期中，我们会看看增加一些入口，比如IRC,XMPP，让我们可以通过命令简单的管理运行中的微服务。如果你有新的入口或者命令需要添加，请提交PR，贡献者是非常欢迎的。目前的插件可以在这里看到：[github.com/micro/go-plugins/bot](https://github.com/micro/go-plugins/tree/master/bot)



这确实是一个基础的框架，用于对Micro生态系统做可编程的机器人。整个工具箱拥有插件化的特性。让我们看看Inputs和Commands是怎样工作的。



## Inputs

Inputs是micro机器人怎样连接hipchat,slack,irc,xmpp等等。我们目前已经实现了HipChat和Slack，应该覆盖了大部分的用户。



这里是Input的接口定义

```go
type Input interface {
	// Provide cli flags
	Flags() []cli.Flag
	// Initialise input using cli context
	Init(*cli.Context) error
	// Stream events from the input
	Stream() (Conn, error)
	// Start the input
	Start() error
	// Stop the input
	Stop() error
	// name of the input
	String() string
}
```

Input提供了方便的功能，用于添加你自己的命令行参数。`Flag()`这个方法会在初始化之前调用，任何自定义的参数会增加到全局参数列表里面。



在参数被解析之后，`Init()`会被调用，这样一来，这个入口的任何中间数据都会被初始化，一旦所有事情执行完成，机器人就会调用`Start（）`然后是`Stream（）`方法，用于与Input建立连接。



这是Stream方法返回的Conn接口

```go
type Conn interface {
	Close() error
	Recv(*Event) error
	Send(*Event) error
}
```



机器人会持续的调用`Recv()`来监听事件。`Recv()`应该是一个阻塞的调用，否则我们会陷入死循环，耗尽CPU。一旦机器人处理完了事件，它会通过`Send()`返回一些结果。



Event是一个基础的类型，用户在机器人和入口之间通信。他可以让我们把不同的消息类型，封装成统一的格式。目前只有一个`TextEvent`类型，在未来我们会有更多。



机器人是不知道命令是来自于Slack,HipChat还是其他地方。它只知道收到了一个事件，然后需要执行它。这是一种很好的方式，用于把机器人和Input拆分开。

这里是Event类型

```go
type Event struct {
	Type EventType
	From string
	To   string
	Data []byte
	Meta map[string]interface{}
}
```



## Commands

commands是可以被机器人执行的函数。这很简单，它们存储在map中，key经过正则，它们会匹配上input接收到的事件。如果正则匹配上了某个事件，关联的函数就会被执行。命令的执行结果就会被发送回input。如果事件的From字段不为空，返回会被发送到To字段。你可以看到这是怎样让机器人直接的进行交流，不管任何地方，任何时候。



当前的Command的接口非常直接，但未来可能会更改，一旦我们遇到更复杂的情况。

command的接口：

```go
type Command interface {
	// Executes the command with args passed in
	Exec(args ...string) ([]byte, error)
	// Usage of the command
	Usage() string
	// Description of the command
	Description() string
	// Name of the command
	String() string
}
```

这里是一个Echo Command的示例

```go
// Echo returns the same message
func Echo(ctx *cli.Context) Command {
	usage := "echo [text]"
	desc := "Returns the [text]"

	return NewCommand("echo", usage, desc, func(args ...string) ([]byte, error) {
		if len(args) < 2 {
			return []byte("echo what?"), nil
		}
		return []byte(strings.Join(args[1:], " ")), nil
	})
}
```



## 其他?

只有Inputs和Commands是不够的。如果我们以后想要做些其他的操作呢？我们怎样持久化机器人的状态?双向的交流怎么样？而不是仅仅返回内容。



这必须要编译！



我们仍然处于构建这个机器人框架的早期，这是一个机会，讨论基础的接口应该是什么样的。



下一步是提供各种类型的接口。更严肃一点，我们需要一个`Stream`接口或者类似的。还需要`Input.Conn`,这样我们可以处理任何插件的事件流。



这潜在的让我们有能力实现同一时间接收多个input的事件流，因此我们可以从事件流中获取事件，处理后返回。



一个例子是，从Slack中接受到消息，查询micro的服务，最后发送一个总结性的邮件。



## 怎样运行起来?

micro机器人在你的环境中单独运行起来，就像其他某个服务一样。也会通过服务发现进行注册。![](https://blog.micro.mu/assets/images/bot.png)



## 我怎样运行?

因为机器人就像运行一个其他服务一下，你首先需要启动服务发现机制，默认是consul

使用支持Slack的机器人

```go
micro bot --inputs=slack --slack_token=SLACK_TOKEN
```

以及HipChat

```
micro bot --inputs=hipchat --hipchat_username=XMPP_USERNAME --hipchat_password=XMPP_PASSWORD
```



## 运行中的机器人

这里有一些运行起来的机器人的截图，就像你看到的，它是一个CLI命令的复制。我们有一些额外的命令比如动画和地图。在这里可以看到[github.com/micro/go-plugins](https://github.com/micro/go-plugins) 

![](https://blog.micro.mu/assets/images/hipchat.png)

## 增加新的Commands

Commands是一个可以被机器人执行的函数，通过字符进行匹配，类似其他的机器人比如Hubot

这里是怎样写一个简单的ping命令



### 编写命令

首先通过NewCommand创建一个命令，这个一个快速的方式，你也可以实现这个接口。

```go
import "github.com/micro/micro/bot/command"

func Ping() command.Command {
	usage := "ping"
	description := "Returns pong"

	return command.NewCommand("ping", usage, desc, func(args ...string) ([]byte, error) {
		return []byte("pong"), nil
	})
}
```



### 注册命令

把命令添加到Commands map中，匹配的key需要被[golang/regexp.Match](https://golang.org/pkg/regexp/#Match)匹配。

这里我们只对ping命令作出响应

```go
import "github.com/micro/micro/bot/command"

func init() {
	command.Commands["^ping$"] = Ping()
}
```



### 连接命令

在这里引入你的命令

```go
import _ "path/to/import"
```

接下来进行编译

```go
cd github.com/micro/micro
go build -o micro main.go link_input.go
```



## 下一步?

我们要意识到微服务世界并不容易，它需要一系列的工具，还要进行观测。比如监控服务、分布式tracing、结构化日志，这都是重要的组成部分。



想象一个世界，机器人有能力感知分布式系统。当我们需要的时候，提供反馈给我们，而不是需要盯着控制台，处理一个个错误提示。你也许听说过NoOps?那么什么是BotOps?你不会被电话催促怎么样？常见的错误，都通过事先预定的程序处理怎么样？



## 总结

机器人的革命只是刚刚开始，基础设施和自动化的世界正在改变，我们相信机器人会扮演一个重要的角色，最初是传统的ChatOps ，未来会走的更远。



机器人需要被看做第一等公民，跟配置管理、命令行、和API一样。我们只是把机器人加入到Micro的生态系统中来。



这仍然是处于早期，但不就的将来将会让我们满意。



如果你想了解更多，请看这个[blog](https://blog.micro.mu/)，或者这个[repo](https://github.com/micro/micro)，Twitter可以关注[@MicroHQ](https://twitter.com/microhq)，Slack社区在[这里](http://slack.micro.mu/)
