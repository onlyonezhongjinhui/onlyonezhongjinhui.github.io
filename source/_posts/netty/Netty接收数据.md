---
title: 从零学Netty：Netty接收数据
date: 2023-05-06 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Netty接收数据既贪心又公平
tags: Netty
categories:
- [Netty]
---

### Netty接收数据

#### 主线

##### Worker Thread

1. NioEventLoop中的Selector多路复用器轮询接收到的OP_READ事件
2. RecvByteBufAllocator创建初始容量大小为2048 byte的ByteBuf用来接收数据
3. 从Channel里接收数据
4. 记录实际接收数据的大小，调整下次分配ByteBuf容量
5. 通过fireChannelRead把接收的数据传递出去
6. 判断接收数据的ByteBuf是否满载而归：如果满载而归则尝试读取直到没有数据或者读取够16次然后结束读取等待下一次OP_READ事件，否则直接结束读取等待下一次的OP_READ事件

#### 本质

​	sun.nio.ch.SocketChannelImpl.read(ByteBuffer dst) 从Channel里读取数据

#### 知识点

1. NioServerSocketChannel使用AbstractNioMessageChannel.read()创建连接,NioSocketChannel使用AbstractNioByteChannel.read()读取数据
2.  pipeline.fireChannelRead(byteBuf)表示完成一次读操作,pipeline.fireChannelReadComplete()表示完成一次OP_READ事件处理，一次OP_READ事件可能有多次读操作
3. 读操作最多尝试16次而不是一直读取，防止某个连接一直霸占NioEventLoop：共同富裕
4. AdaptiveRecvByteBufAllocator对分配ByteBuf容量的猜测：放大果断，缩小谨慎（需要连续两次判断）

#### 总结

​	Netty接收数据既贪心又公平：贪心就是尽可能读取更多的数据，公平就是不会一直霸占线程读取数据，最多读取16次后就把读的机会让给别人实现共同富裕

