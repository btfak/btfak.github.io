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
// Main malloc heap.
// The heap itself is the "free[]" and "large" arrays,
// but all the other global data is here too.
//
// mheap must not be heap-allocated because it contains mSpanLists,
// which must not be heap-allocated.
//
//go:notinheap
type mheap struct {
	lock      mutex
	free      [_MaxMHeapList]mSpanList // free lists of given length
	freelarge mSpanList                // free lists length >= _MaxMHeapList
	busy      [_MaxMHeapList]mSpanList // busy lists of large objects of given length
	busylarge mSpanList                // busy lists of large objects length >= _MaxMHeapList
	sweepgen  uint32                   // sweep generation, see comment in mspan
	sweepdone uint32                   // all spans are swept

	// allspans is a slice of all mspans ever created. Each mspan
	// appears exactly once.
	//
	// The memory for allspans is manually managed and can be
	// reallocated and move as the heap grows.
	//
	// In general, allspans is protected by mheap_.lock, which
	// prevents concurrent access as well as freeing the backing
	// store. Accesses during STW might not hold the lock, but
	// must ensure that allocation cannot happen around the
	// access (since that may free the backing store).
	allspans []*mspan // all spans out there

	// spans is a lookup table to map virtual address page IDs to *mspan.
	// For allocated spans, their pages map to the span itself.
	// For free spans, only the lowest and highest pages map to the span itself.
	// Internal pages map to an arbitrary span.
	// For pages that have never been allocated, spans entries are nil.
	//
	// This is backed by a reserved region of the address space so
	// it can grow without moving. The memory up to len(spans) is
	// mapped. cap(spans) indicates the total reserved memory.
	spans []*mspan

	// sweepSpans contains two mspan stacks: one of swept in-use
	// spans, and one of unswept in-use spans. These two trade
	// roles on each GC cycle. Since the sweepgen increases by 2
	// on each cycle, this means the swept spans are in
	// sweepSpans[sweepgen/2%2] and the unswept spans are in
	// sweepSpans[1-sweepgen/2%2]. Sweeping pops spans from the
	// unswept stack and pushes spans that are still in-use on the
	// swept stack. Likewise, allocating an in-use span pushes it
	// on the swept stack.
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

