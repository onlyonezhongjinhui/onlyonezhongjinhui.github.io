---
title: 从零学Netty：Netty关闭服务
date: 2023-05-10 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: NettyNetty优雅关闭服务
tags: Netty
categories:
- [Netty]
---

#### Netty关闭服务

#### 主线

##### Main Thread

1. bossGroup.shutdownGracefully()、workerGroup.shutdownGracefully()
2. 设置NioEventLoop线程的状态为ST_SHUTTING_DOWN

##### Worker Thread

1. NioEventLoop线程检测到状态变为ST_SHUTTING_DOWN后开始执行关闭逻辑
2. 关闭所有channel
3. 取消定时任务调度，防止加入新的定时任务  
4. 执行全部的定时任务、关闭钩子
5. 关闭selector

#### 本质

1. sun.nio.ch.SocketChannelImpl.close()关闭所有channel
2. 关闭所有NioEventLoop线程：退出run方法循环体
3. sun.nio.ch.SelectionKeyImpl.close()关闭selector

#### 知识点

1. 优雅关闭时间：DEFAULT_SHUTDOWN_QUIET_PERIOD（2秒），在此期间如果没有定时任务或关闭钩子执行过则直接退出NioEventLoop线程
2. 可控关闭时间：DEFAULT_SHUTDOWN_TIMEOUT（15秒），超过此时间期限则直接退出NioEventLoop线程，所以会有消息丢失

#### 总结

​	Netty关闭服务的时候尽量做更多的活，但是不能保证所有的活都能干完，否则关闭服务的时间将不可控

