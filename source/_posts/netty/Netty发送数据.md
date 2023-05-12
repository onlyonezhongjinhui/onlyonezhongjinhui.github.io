---
title: 从零学Netty：Netty发送数据
date: 2023-05-08 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Netty发送数据的过程和接收数据的过程一样，既贪心又公平
tags: Netty
categories:
- [Netty]
---

#### Netty发送数据

#### 主线

##### Worker Thread

1. write把数据添加到ChannelOutboundBuffe：

   outboundBuffer.addMessage(msg, size, promise)把数据添加到ChannelOutboundBuffe链表unflushedEntry

2. flash把ChannelOutboundBuffer里面的数据发送出去：

   a) outboundBuffer.addFlush()把unflushedEntry数据搬到ChannelOutboundBuffe链表flushedEntry，然后置空unflushedEntry

   b)  NioSocketChannel.doWrite()把flushedEntry链表的数据发送出去，然后删除已发送的数据

   ![](/medias/assets/netty/write-flush.png)

#### 本质

​	sun.nio.ch.SocketChannelImpl.write()

#### 知识点

1. 数据写不进去的时候，会停止写入然后注册一个OP_WRITE事件等待下次可写了再开始写入
2. OP_WRITE事件发生表示的是数据可以写入而不是说有数据写入。所以正常情况下不能注册，否则会一直触发
3. 批量写数据的时候，如果尝试写的都写进去了，接下来会尝试写更多，体现在maxBytesPerGatheringWrite这个参数的调整
4. 只要有数据写且是可写的状态，会一直尝试写，直到尝试16次,通过writeSpinCount控制。如果超过16次还未写完，那么就schedule一个task来继续写，而不是用注册事件来触发，更加简洁有力
5. 待写入的数据太多，超过最高水位线，则会把channel设置为不可写状态，让应用端程序自己决定是否要继续写，体现在WriteBufferWaterMark.high参数的调整。待写入的数据太少，低于了最低水位线，则会把channel设置为可写状态，通过WriteBufferWaterMark.low参数控制
6. ctx.channel().write()从TailContext开始执行。ctx.write()从当前Context开始执行

#### 总结

​	Netty发送数据的过程和接收数据的过程一样，既贪心又公平

