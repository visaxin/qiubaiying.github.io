---
layout:     post                       # 使用的布局（不需要改）
title:      Golang http server performance tuning practice                  # 标题 
subtitle:   Golang HTTP服务器性能调优实践 #副标题
date:       2017-02-06                 # 时间
author:     BY                         # 作者
header-img: img/post-bg-2015.jpg     #这篇文章标题背景图片
catalog: true                         # 是否归档
tags:                                #标签
    - Golang
    - Performance tunning
---

## Hey
>一篇关于golang的碎碎念。

Golang 1.8后本身自带了pprof的这个神器，可以帮助我们很方便的对服务做一个比较全面的profile。对于一个Golang的程序，可以从多个方面进行profile，比如memory和CPU两个最基本的指标，也是我们最关注的，同时对特有的goroutine也有运行状况profile。关于golang profiling本身就不做过多的说明，可以从官方博客中了解到详细的过程。

 Profile的环境



Ubuntu 14.04.4 LTS (GNU/Linux 3.19.0-25-generic x86_64)
go version go1.9.2 linux/amd64
 profile的为一个elassticsearch数据导入接口，承担接受上游数据，根据元数据信息写入相应的es索引中。目前的状况是平均每秒是1.3Million的Doc数量。

  在增加了profile后，从CPU层面发现几个问题。

runtime mallocgc 占用了17.96%的CPU。 SVG部分图如下

![SVG CPU Profile](../blog_img/cpu.jpg)

![SVG CPU Profile](../blog_img/cpu-g.jpg)


通过SVG图，可以看到调用链为：

ioutil.ReadAll -> buffer.ReadFrom -> makeSlice -> malloc.go 

然后进入ReadAll的源码。



readAll()方法

```
func readAll(r io.Reader, capacity int64) (b []byte, err error) {

buf := bytes.NewBuffer(make([]byte, 0, capacity))

// If the buffer overflows, we will get bytes.ErrTooLarge. // Return that as an error. Any other panic remains. 

defer func() {

e := recover()

if e == nil {

return }

if panicErr, ok := e.(error); ok && panicErr == bytes.ErrTooLarge {

err = panicErr } else {

panic(e)

}

}()

_, err = buf.ReadFrom(r)

return buf.Bytes(), err}
```


可以看到，每次调用readAll时，每次都会NewBuffer，大小为bytes.MinRead=512.

再进入核心的方法。 buf.ReadFrom()


```
func (b *Buffer) ReadFrom(r io.Reader) (n int64, err error) {

b.lastRead = opInvalid // If buffer is empty, reset to recover space. if b.off >= len(b.buf) {

b.Reset()

}

for {

if free := cap(b.buf) - len(b.buf); free < MinRead {

// not enough space at end newBuf := b.buf if b.off+free < MinRead {

// not enough space using beginning of buffer; // double buffer capacity newBuf = makeSlice(2*cap(b.buf) + MinRead)

}

copy(newBuf, b.buf[b.off:])

b.buf = newBuf[:len(b.buf)-b.off]

b.off = 0 }

m, e := r.Read(b.buf[len(b.buf):cap(b.buf)])

b.buf = b.buf[0 : len(b.buf)+m]

n += int64(m)

if e == io.EOF {

break }

if e != nil {

return n, e }

}

return n, nil // err is EOF, so return nil explicitly

}
```





这个方法时从io.Reader中读取数据，并且copy到我们刚才new出来的buffer中。如果buffer的大小不够，则对buffer进行扩展，大小为之前的2倍，然后把数据copy进去。



同时，分析线上数据发现，平均一个请求体的大小为5MB大小。每次请求都会造成大量的makeSlice操作和随之而来的GC，造成CPU的浪费。

解决问题的思路也很清晰。 

1. 减少make([]byte,0,bytes.MinRead)操作 

2. 避免频繁的makeSlice（对数组的扩展操作）和copy操作。



使用一个 buffer的pool可以解决问题。参考在shadowsocks在leakybuf中实现，并且针对业务来做了如下修改。


```
// Put add the buffer into the free buffer pool for reuse. Panic if the buffer// size is not the same with the leaky buffer's. This is intended to expose// error usage of leaky buffer.func (lb *LeakyBuf) Put(b []byte) {
if len(b) != lb.bufSize {
// 源码中panic，这里为了防止[]byte数组被修改后放进来，是默认重新赋值为默认的大小
b = make([]byte, lb.bufSize)
}
select {
case lb.freeList <- b:
default:
}
return}
const leakyBufSize = 5 * 1024 * 1024 // 5MB 根据统计，解决大部分请求时再申请扩大内存的操作
```

代替ioutil.ReadAll的代码如下

```
cacheBuf := leakyBuf.Get()

defer leakyBuf.Put(cacheBuf) // 用完后放回去

buf := bytes.NewBuffer(cacheBuf)

buf.Reset() // buffer在用之前rest是个好习惯

_, err = buf.ReadFrom(request.Body) // 从http body中读取数据if err != nil {
return
}
body := buf.Bytes()
```