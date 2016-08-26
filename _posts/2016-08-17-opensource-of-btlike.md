---
layout:     post
title: 开源一个完整的BT搜索引擎
date: 2016-08-17 13:21:35
summary: 单核,768MB的VPS上，每秒处理UDP请求超过12K，内存占用不超过100MB
categories: BT搜索引擎
---

### 开源

- 地址  [github.com/btlike](http://github.com/btlike)


- 在线演示 [www.btlike.com](http://www.btlike.com)

### 特性
- 高性能  
  单核,768MB的[VPS](https://www.vultr.com/pricing/)上，每秒处理UDP请求超过12K，内存占用不超过100MB`
- 大容量  
  采用Mysql分表存储,设计容量6千万~8千万数据
- 全文搜索  
  采用[Elasticsearch](https://github.com/elastic/elasticsearch)实现全文索引和热度排序

### 组件
- [spider](https://github.com/btlike/spider)   底层DHT网络爬虫库
- [crawl](https://github.com/btlike/crawl)     从DHT网络中抓取活跃的infohash与metadata并存储
- [storage](https://github.com/btlike/storage) 根据infohash从网络中抓取torrent并存储metadata
- [api](https://github.com/btlike/api)    对外提供接口，响应搜索等请求


### 直接下载可执行文件运行(推荐方式)
- 下载地址: [release](https://github.com/btlike/release)

### 安装准备(从源码安装，较为复杂)
- 安装 golang
- 安装 mysql
- 安装 [Elasticsearch](https://github.com/elastic/elasticsearch)
- 安装组件[crawl](https://github.com/btlike/crawl)  
  go get github.com/btlike/crawl
- 安装组件[api](https://github.com/btlike/api)  
  go get github.com/btlike/api
- 安装组件[storage](https://github.com/btlike/storage)  
  go get github.com/btlike/storage

### 流量图
![flow](http://77g42f.com1.z0.glb.clouddn.com/flow.jpg)

### 设计思路
- 从全网尽可能多的获取infohash
- 从DHT网络，通过[BEP0009](http://www.bittorrent.org/beps/bep_0009.html)获取资源
- 通过资源库查询资源，采用流量+带宽暴力遍历
- 极力优化内存、CPU与带宽资源

### 帮助与联系
在项目下提issue或联系 yanyuan2046 at 126.com