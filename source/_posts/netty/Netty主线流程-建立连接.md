---
title: 【从零学Netty】Netty主线流程：创建连接
date: 2023-05-05 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Netty创建连接的核心操作就是对OP_ACCEPT、OP_READ事件的处理
tags: Netty
categories:
- [Netty]
---

### Netty创建连接

#### 主线过程

##### Boss Thread

1. NioEventLoop中的Selector轮询创建连接事件(OP_ACCEPT)
2. 创建NioSocketChannel
3. 初始化NioSocketChannel并从Worker Group中选择一个NioEventLoop

##### Worker Thread

1. 将NioSocketChannel注册到选中的NioEventLoop的Selector上
2. 注册读事件(OP_READ)到Selector上

#### 本质

1. Selector.select()/Selector.selectNow()/Selector.select(timeoutMillis) 发现OP_ACCEPT事件
2. NioSocketChannel socketChannel = serverSocketChannel.accept() 创建NioSocketChannel
3. SelectionKey selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this) 把NioSocketChannel注册到NioEventLoop的Selector上
4. selectionKey.interestOps(SelectionKey.OP_READ) 注册监听OP_READ事件

#### 知识点

1. 创建连接的初始化和注册是通过pipeline.fireChannelRead在ServerBootstrapAcceptor中完成的
2. 第一次Register监听的是0而不是OP_READ,SelectionKey selectionKey = javaChannel().register(eventLoop().unwrappedSelector(), 0, this) 
3. 监听OP_READ是通过Register完成后的fireChannelActive触发
4. Worker's NioEventLoop是通过Register操作执行过程启动的
5. 接受连接的读操作，不会尝试读取更多次（16次）

#### 总结

​	Netty创建连接的核心操作就是Boss Group处理OP_ACCEPT事件然后从Worker Group中选择一个NioEventLoop处理OP_READ事件

