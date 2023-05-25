---
title: 【从零学Netty】Reactor模式
date: 2023-05-01 10:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: Reactor
tags: Netty
categories:
- [Netty]
---

### Reactor模式

#### 生活场景类比

​	由于A饭店的饭菜好吃又实惠，客人越来越多，老板迅速扩大了饭店的规模，现在正在考虑如何更好的服务顾客：

1. 一个伙计包揽全部活：迎宾、点菜、做菜、上菜、送客等
2. 多招聘几个伙计，大家一起干上面的活
3. 进一步分工，招聘一个或多个漂亮的小姐姐专门负责迎宾

​	类比对象：

​		伙计——工作线程

​		迎宾——接入连接

​		点菜——请求

​		做菜——业务处理

​		上菜——响应

#### Reactor单线程模式

![reactor-single-thread](/medias/assets/netty/reactor-single-thread.PNG)

​	单线程包揽所有的活（对应一个伙计包揽全部活的模式）

#### Reactor多线程模式

![reactor-worker-thread-pool](/medias/assets/netty/reactor-worker-thread-pool.PNG)

​	reactor线程负责接收连接，然后在线程池中处理（对应多招聘几个伙计一起干模式）

#### Reactor主从模式

![reactor-main-sub](/medias/assets/netty/reactor-main-sub.PNG)

​	主reactor线程专门负责服务器应用最关心的事情——接收连接，从reactor线程负责后续处理（对应进一步分工迎宾小姐姐模式）

#### Netty如何使用Reactor三种模式

| 模式              | 示例代码                                                     |
| ----------------- | ------------------------------------------------------------ |
| Reactor单线程模式 | EventLoopGroup group = new NioEventLoopGroup(1);<br/>ServerBootstrap b = new ServerBootstrap();<br/>b.group(group); |
| Reactor多线程模式 | EventLoopGroup group = new NioEventLoopGroup();<br/>ServerBootstrap b = new ServerBootstrap();<br/>b.group(group); |
| Reactor主从模式   | EventLoopGroup bossGroup = new NioEventLoopGroup(1);<br/>EventLoopGroup workerGroup = new NioEventLoopGroup();<br/>ServerBootstrap b = new ServerBootstrap();<br/>b.group(bossGroup, workerGroup); |

#### 总结

​	接收连接是服务器最重要的事情。Reactor三种模式从无分工到有分工，分工后有专门的Reactor线程接收连接，极大的提升了服务器接收连接的能力，所以Netty才能轻松实现海量连接的接入

