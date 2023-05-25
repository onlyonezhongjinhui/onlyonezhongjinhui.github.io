---
title: 【从零学Netty】Netty主线流程：启动过程
date: 2023-05-04 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Netty启动过程最核心的操作就是做好接受连接的准备
tags: Netty
categories:
- [Netty]
---

### Netty启动过程

#### 主线过程

##### Main Thread

1. 创建Selector	
2. 创建NioServerSocketChannel
3. 初始化NioServerSocketChannel
4. 从Boss Group中选择一个NioEventLoop给NioServerSocketChannel使用

##### Boss Thread

1. 将NioServerSocketChannel注册到选择的NioEventLoop的Selector上
2. 绑定地址启动
3. 注册接受连接事件OP_ACCEPT到Selector上

#### 本质

1. Selector selector = sun.nio.ch.SelectorProviderImpl.openSelector() 创建Selector多路复用器
2. ServerSocketChannel channel = provider.openServerSocketChannel() 创建NioServerSocketChannel
3. SelectionKey selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this) NioServerSocketChannel注册到NioEventLoop的Selector
4. javaChannel().bind(localAddress, config.getBacklog()) 绑定地址启动
5. selectionKey.interestOps(SelectionKey.OP_ACCEPT) 注册接受连接事件(OP_ACCEPT)

#### 知识点

1. Selector是在创建NioEventLoopGroup(创建一批NioEventLoop)时创建的
2. 第一次Register监听的是0而不是OP_ACCEPT,SelectionKey selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this);
3. 监听OP_ACCEPT是通过bind完成后的fireChannelActive()异步触发
4. NioEventLoop是通过Register操作的执行来完成启动的
5. ChanneInitializer是一次性的，用完后就会移除。一些handler也可以设计成一次性的，例如授权

#### 总结

​	Netty启动过程最核心的操作就是做好接受连接的准备



