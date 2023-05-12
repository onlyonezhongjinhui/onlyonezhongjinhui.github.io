---
title: 从零学Netty：Netty业务处理
date: 2023-05-07 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Netty处理业务很方便很灵活，只需要实现业务handler添加的pipeline中即可
tags: Netty
categories:
- [Netty]
---

#### Netty业务处理

#### 主线

##### Worker Thread

1. NioEventLoop处理OP_READ事件，把读取到的数据传递出去：pipeline.fireChannelRead(byteBuf)
2. pipeline触发handler包含业务处理的channelRead方法完成业务的处理

![](/medias/assets/netty/pipeline.png)

#### 本质

​	数据在pipeline中所有handler的channelRead方法的执行

#### 知识点

1. pipeline上的handler不是都会执行，必须实现ChannelInboundHandler且channelRead方法不能有@Skip注解
1. pipeline从Head Handler开始到Tail Handler结束，但是不保证都会执行到Tail Handler，中途是可以退出的
1. 默认Handler处理的线程是Channel绑定的NioEventLoop线程，也可以设置其它：

​		p.addLast(new UnorderedThreadPoolEventExecutor(128), xxHandler)

#### 总结

​	Netty处理业务很方便很灵活，只需要实现业务handler添加的pipeline中即可



​		

