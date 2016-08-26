---
layout: post
title: BT搜索引擎爬虫实践
date: 2016-06-15 09:21:35
summary: 最近完成了一个BT搜索引擎，基于磁力链接，目前这个技术已经很成熟了，从三年前的论文发表开始，以及去年的一系列开源，技术上几乎已经没有难题，更多难点在工程上。后端采用Golang+Mysql+Elasticsearch，前端采用bootstrap
categories: BT搜索引擎
---



最近完成了一个BT搜索引擎，基于[磁力链接](https://zh.wikipedia.org/zh/%E7%A3%81%E5%8A%9B%E9%93%BE%E6%8E%A5)，目前这个技术已经很成熟了，从三年前的[这篇文章](http://xiaoxia.org/2013/05/11/magnet-search-engine/)开始，和去年的[开源](http://xiaoxia.org/2015/05/15/shousibaocai-opensource/)，技术上几乎已经没有难题，更多难点在工程上。后端采用Golang+Mysql+Elasticsearch，前端采用bootstrap。后端由三个单独的模块组成，spider组件是从DHT网络获取活跃资源id，storage组件是根据id查询到种子信息（metadata），存储到Mysql，并索引到Elasticsearch。API组件是对外提供查询、搜索服务，API组件采用nginx进行负载均衡。前端采用bootstrap，使用nginx做为web服务器，前端直接与后端的API组件进行交互。



本文介绍爬虫的实现技巧。



磁力搜索的原理看[这个文章](http://codemacro.com/2013/05/19/crawl-dht/)，近两年已经有多个开源版本，我参考了[nodejs版本](https://github.com/alanyang/dhtspider)和[golang版本](https://github.com/xiaojiong/DhtCrawler)，前者在算法上更为巧妙，后者在实现上已经有现成的逻辑。结合两者，实现了自有版本，会在未来开源出来。



在单核，内存768M的VPS上，完全运行时，每秒UDP请求约12K，每日入口流量约220G，出口流量约300G，CPU会跑到100%，内存占用约100MB。



爬虫的关键在于认识尽可能多的节点，一旦进入某节点的哈希表中，此节点想要下载资源，都会询问你，这样你就得到了一个活跃的资源id。



第一个技巧：动态控制find_node请求的频率

启动时应该全力调用find_node，尽快的结识尽可能多的节点，随着认识你的节点数量迅速递增，再降低速率，前期结识的节点已经足够跑满你的CPU，否则会无端占用带宽、CPU资源。一个可以参考的算法是每分钟收到的get_peers请求每增加一万，find_node的sleep时间增加50毫秒，算下来当你每分钟收到40万个get_peers请求时，每两秒钟才会调用一次find_node，根据你的机器，这个50毫秒是可调整的。



第二个技巧：均匀分配节点id

节点会根据自己的id认识周围的节点，整个网络节点数量最大是16的40次方个，每个节点的id是随机的。加入我们运行100个节点，随机分配显然是不行的，应该是把整个网络均分为100份，每一份中运行我们的一个节点，才能尽可能接触到不同的资源，否则多个节点收到的资源可能会是同一个。参考代码：

```go
type ID []byte

func GenerateIDList(count int64) (ids []ID) {
	if count <= 0 {
		return
	}
	max := []byte{0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff,
		0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff, 0xff}
	num := big.NewInt(0).SetBytes(max)
	step := big.NewInt(0).Div(num, big.NewInt(count+2))
	for i := 1; i < int(count+1); i++ {
		item := big.NewInt(0).Mul(step, big.NewInt(int64(i)))
		item.Add(item, big.NewInt(int64(rand.Intn(99))))
		ids = append(ids, item.Bytes())
	}
	return
}
```



第三个技巧：内存复用

在如此高的流量压力下怎样控制内存占用是关键的问题，理论上每个UDP请求都需要使用新的内存空间，但分析协议你会发现有趣的事情，协议定义看[这里](http://www.bittorrent.org/beps/bep_0005.html)。爬虫只会使用5条协议，发出的只有find_node，收到的是ping，get_peers，announce_peer，find_node。收到的四条协议在格式上是很接近的，而且长度是可控的。通过使用pprof进行分析，最主要的内存消耗发生在读取请求内容和解析请求内容上。如果我们运行的每个节点复用这些空间呢？那么内存总量就是可控的，只会随着节点数量线性增加，因此我们可以并发的运行一批节点，而节点接收UDP请求是单线程的。参考以下代码：

```go
func (nw *Network) Listening() {
	val := make(map[string]interface{})  //每个节点解析请求需要占用的空间
	buf := make([]byte, 1024)            //每个节点读取请求内容需要占用的空间
	for {
		n, raddr, err := nw.Conn.ReadFromUDP(buf)
		if err != nil {
			continue
		}
		nw.Dht.krpc.Decode(buf[:n], val, raddr)
	}
}
```



第四个技巧：高并发下优化nf_conntrack

终于运行起了爬虫，但运行没几分钟，各种linux问题出现了，最开始应该是ulimit问题，这个问题很好解决，参考[这个文章](http://www.stutostu.com/?p=1322)。然后会出现开始大量报出：`nf_conntrack: table full, dropping packet`。这个问题参考[这个文章](http://jaseywang.me/2012/08/16/%E8%A7%A3%E5%86%B3-nf_conntrack-table-full-dropping-packet-%E7%9A%84%E5%87%A0%E7%A7%8D%E6%80%9D%E8%B7%AF/)。原因就是，

```
nf_conntrack/ip_conntrack 跟 nat 有关，用来跟踪连接条目，它会使用一个哈希表来记录 established 的记录。nf_conntrack 在 2.6.15 被引入，而 ip_conntrack 在 2.6.22 被移除，如果该哈希表满了，就会出现：nf_conntrack: table full, dropping packet。
```

解决办法很简单，我们让某些端口的流量不要被记录即可。假如我们运行100个节点，而节点监听的端口是20000到20099，我们只需要执行以下命令即可。

```
iptables -A INPUT -m state --state UNTRACKED -j ACCEPT
iptables -t raw -A PREROUTING -p udp -m udp --dport 20000 -j NOTRACK
...... //从端口20000一直到20099，每个端口一行
iptables -t raw -A PREROUTING -p udp -m udp --dport 20099 -j NOTRACK
```
