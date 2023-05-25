---
title: 【从零学Netty】Netty主线流程：关闭连接
date: 2023-05-09 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Netty关闭连接是对OP_READ事件的处理
tags: Netty
categories:
- [Netty]
---

### Netty关闭连接

#### 主线

##### Worker Thread

1. NioEventLoop处理OP_READ事件，读取的数据字节数小于等于0则执行关闭连接操作
2. 关闭channel，取消绑定channel上注册的所有事件，不再接收新数据
3. 清空所有队列中的数据（flushedEntry、unflushed）
4. 触发pipeline.fireChannelInactive()、pipeline.fireChannelUnregistered()事件

#### 本质

​	sun.nio.ch.SocketChannelImpl.close()关闭channel，sun.nio.ch.SelectionKeyImpl.cancel()取消所有事件不再接收新数据

#### 知识点

1. 关闭连接是对接收OP_READ事件的处理,已读取的数据字节数小于等于0作为判断的依据
2. 数据进行读取时，强行关闭连接会触发IOException，进而执行关闭流程 

#### 总结

​	Netty关闭连接是对OP_READ事件的处理，包含了正常关闭和异常关闭。正常关闭是通过读取的字节数小于等于0来做未判断条件。异常关闭则是通过捕获IOException而触发

