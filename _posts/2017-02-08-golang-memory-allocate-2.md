---
layout:     post
title: golang内存分配 (二)
date: 2017-02-08 18:21:35
summary: 剖析源码,研究golang内存分配细节。本文拆解mheap
categories: golang
---

源码基于go1.8rc3。

首先看看mheap的数据结构

```go
// mheap本身只包含"free[]" and "large"数组
// 但其他的全局数据也在这里
// mheap 禁止从堆上创建，因包含的mSpanLists不能从堆上创建
type mheap struct {
	lock      mutex
	free      [_MaxMHeapList]mSpanList // free lists of given length
	freelarge mSpanList                // free lists length >= _MaxMHeapList
	busy      [_MaxMHeapList]mSpanList // busy lists of large objects of given length
	busylarge mSpanList                // busy lists of large objects length >= _MaxMHeapList
	sweepgen  uint32                   // sweep generation, see comment in mspan
	sweepdone uint32                   // all spans are swept

    // allspans是所有曾经创建过的mspans的数组，每个mspan只存储一次
    // allspans的内存是手动管理的，随着堆的增长，可以重新创建或移动
    // 一般来说，allspans是被mheap.lock所保护，当释放backing store时，锁用于防止并发的访问
    // 在STW期间访问可能不会持有锁，但必须确保分配不会发生在访问的时候(因为这可能释放backing store)
	allspans []*mspan // all spans out there

	
    // spans是一张查询表，用于映射虚拟地址页的id到*mspan
    // 对于已经分配spans，它们的页映射到span本身
    // 对于空闲的spans，只有最低位和最高位的页映射到span本身
    // 内部的页映射到一个固定的span
    // 对于从未分配过的页，span是空的
    // 空间分配在一个预先储备好大块的地址上，因此它可以不断增长而不需要移动。len(spans)是分配出去的内存，
    // cap(spans)是所有储备的内存空间
	spans []*mspan

    // sweepSpans包含两个mspan栈，一个是已经清理的使用中的spans，一个是未清理的使用中的spans。
    // 这两个角色在每个GC周期相互转换，因此sweepgen这个字段在每个GC周期中增加2,这意味着清理完的spans
    // 存放在sweepSpans[sweepgen/2%2]，而未清理的spans存放在sweepSpans[1-sweepgen/2%2]。
    // 清理过程会把spans从未清理的栈中取出来，并把使用中的spans放到清理完成的栈中。同样的，分配一个使用中      
    // 的span会把它放到已清理的栈中
	sweepSpans [2]gcSweepBuf

	_ uint32 // align uint64 fields on 32-bit for atomics

	// Proportional sweep
	pagesInUse        uint64  // pages of spans in stats _MSpanInUse; R/W with mheap.lock
	spanBytesAlloc    uint64  // bytes of spans allocated this cycle; updated atomically
	pagesSwept        uint64  // pages swept this cycle; updated atomically
	sweepPagesPerByte float64 // proportional sweep ratio; written with lock, read without
	// TODO(austin): pagesInUse should be a uintptr, but the 386
	// compiler can't 8-byte align fields.

	// Malloc stats.
	largefree  uint64                  // bytes freed for large objects (>maxsmallsize)
	nlargefree uint64                  // number of frees for large objects (>maxsmallsize)
	nsmallfree [_NumSizeClasses]uint64 // number of frees for small objects (<=maxsmallsize)

	// range of addresses we might see in the heap
	bitmap         uintptr // Points to one byte past the end of the bitmap
	bitmap_mapped  uintptr
	arena_start    uintptr
	arena_used     uintptr // always mHeap_Map{Bits,Spans} before updating
	arena_end      uintptr
	arena_reserved bool

	// central free lists for small size classes.
	// the padding makes sure that the MCentrals are
	// spaced CacheLineSize bytes apart, so that each MCentral.lock
	// gets its own cache line.
	central [_NumSizeClasses]struct {
		mcentral mcentral
		pad      [sys.CacheLineSize]byte
	}

	spanalloc             fixalloc // allocator for span*
	cachealloc            fixalloc // allocator for mcache*
	specialfinalizeralloc fixalloc // allocator for specialfinalizer*
	specialprofilealloc   fixalloc // allocator for specialprofile*
	speciallock           mutex    // lock for special record allocators.
}
```

