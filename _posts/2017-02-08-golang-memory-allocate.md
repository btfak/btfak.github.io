---
layout:     post
title: golang内存分配 (一)
date: 2017-02-08 15:21:35
summary: 剖析源码,研究golang内存分配细节。本文介绍基于tcmalloc的内存分配器设计原理。
categories: golang
---

源码基于go1.8rc3。

内存分配器是基于tcmalloc的思路，不过实现上有些差异。http://goog-perftools.sourceforge.net/doc/tcmalloc.html

小对象(<=32kb)的分配空间会对齐到67种规格的其中之一，比如大小为900-1024byte的对象都会分配到1024byte的空间。每种规格都有一系列相应规格的空闲空间。空闲的页可以被切分给某种规格使用，使用空闲的bitmap进行管理。

#### 整体架构

- fixalloc：为固定大小的堆外（off-heap）对象分配空闲空间的分配器，并管理分配器使用的存储空间
- mheap：分配内存的堆，按页(8kb)的粒度进行管理
- mspan：由mheap管理的使用中的页
- mcentral：收集了各种指定大小的span
- mcache：线程级别的空闲的mspan缓存
- mstats：内存分配的统计信息

#### 为小对象(<=32kb)分配内存

- 对齐对象的大小到相应的规格，在线程的缓存中查询相应的mspan，遍历mspan的bitmap寻找一个空闲的块，如果存在空闲的块，就分配使用，这个过程是无锁的
- 如果mspan没有空闲的块，且mcentral相应规格的mspan有空闲空间，就从mcentral获取一个mspan，获取mspan的过程会对mcentral加锁
- 如果mcentral的mspan列表是空的，从mheap从获取页并切分为mspan供使用
- 如果mheap是空的，或者没有足够大的页，从操作系统申请一组页供使用（至少1MB）

#### 清理mspan

- 当一个mspan被清理而归还分配的空间，它会被归还到mcache供后续使用
- 如果mspan中仍然有已分配的对象，它会被归还到mcentral的相应规格中
- 如果mspan没有分配的对象，这个mspan就是空闲的（idle），它会被归还到mheap，不再有这个规格存在了
- 如果这个mspan保持的idle状态足够长的时间，就会把相应的页归还到操作系统



*分配和清理大对象直接使用mheap，绕过mcache和mcentral*



