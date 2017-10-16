---
layout:     post
title: ioutil.ReadAll 详解
date: 2017-10-16 21:21:35
summary: 读取超大 buffer 时，会申请多大的内存?
categories: golang, io
---

### 使用场景
**ioutil.ReadAll** 用于从 **io.Reader** 中读取出所有的字节，如果 **io.Reader** 的内容很大，几十上百GB，这些内容可以正常读取吗，如果内存不够，会怎样处理？

### 源码

关键逻辑

1、首先开辟 MinRead=512 字节进行读取，如果不够，再开辟2倍+MinRead的空间

2、如果开辟失败，panic(bytes.ErrTooLarge)，此错误在上层捕获，**ioutil.ReadAll** 最终返回错误为 **bytes.ErrTooLarge**

```go
// ReadFrom reads data from r until EOF and appends it to the buffer, growing
// the buffer as needed. The return value n is the number of bytes read. Any
// error except io.EOF encountered during the read is also returned. If the
// buffer becomes too large, ReadFrom will panic with ErrTooLarge.
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {
	b.lastRead = opInvalid
	// If buffer is empty, reset to recover space.
	if b.off >= len(b.buf) {
		b.Truncate(0)
	}
	for {
		if free := cap(b.buf) - len(b.buf); free < MinRead {
			// not enough space at end
			newBuf := b.buf
			if b.off+free < MinRead {
				// not enough space using beginning of buffer;
				// double buffer capacity
				newBuf = makeSlice(2*cap(b.buf) + MinRead)
			}
			copy(newBuf, b.buf[b.off:])
			b.buf = newBuf[:len(b.buf)-b.off]
			b.off = 0
		}
		m, e := r.Read(b.buf[len(b.buf):cap(b.buf)])
		b.buf = b.buf[0 : len(b.buf)+m]
		n += int64(m)
		if e == io.EOF {
			break
		}
		if e != nil {
			return n, e
		}
	}
	return n, nil // err is EOF, so return nil explicitly
}
```

### 结论
1、**ioutil.ReadAll** 只要内存足够，就会一直读取内容到内存中，如果内存不够，返回报错：bytes.ErrTooLarge

2、如果内容很大，需要自定义读取方式，比如按行读等