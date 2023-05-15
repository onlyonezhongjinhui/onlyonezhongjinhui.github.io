---
title: 从零学Netty：Keepalive与Idle
date: 2023-05-01 22:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: keepalive+idle监测可以减少keepalive的探测消息发送
tags: Netty
categories:
- [Netty]
---

### keepalive与idle监测

#### keepalive

​	keepalive并不是TCP协议规范的一部分，但在几乎所有的TCP/IP协议栈中，都实现了KeepAlive功能。

​	keepalive通过定时发送探测包来探测连接的对端是否存活。可以用来：

​		1、及时检测出挂掉的连接。连接挂掉的原因可能有很多，如服务停止、网络波动、宕机、应用重启等。

​		2、防止因为连接长时间无数据交互而断连。相当于TCP层面的心跳包

#### TCP keepalive核心参数

​	net.ipv4.tcp_keepalive_time=7200 # 7200秒后发送keepalive消息

​	net.ipv4.tcp_keepalive_intvl=75 #keepalive消息重试间隔为75秒

​	net.ipv4.tcp_keepalive_probes=9 # 无确认则一直重试9次

​	当启用keepalive的时候（默认是关闭的），TCP在连接在7200秒内没有收发数据后，会发送keepalive消息，如果探测消息没有得到回应，那么每隔75秒就发送一次，如果连续9次都没有得到回应，则认为连接是失效了。期间总计耗时2小时11分钟，相当长。

#### keepalive生活场景类比

​	还是A餐馆，为了进一步扩大经营，老板开通了电话订餐配送服务，专门招聘了几个小姐姐负责电话订餐工作。这个服务一开通，餐馆的生意又好了不少。某天顾客来电订餐，小姐姐接通了电话后，顾客点了一些菜品后突然就没有了声音（临时走开、处理其它事情、线路故障等原因）。这个时候小姐姐该如何处理呢？小姐姐会问一句，“您还在吗？”，如果顾客没有应答，那么她就会挂机，如果应答了，那么小姐姐就会继续等待顾客说话。小姐姐这个操作就是keepalive机制。

​	类比对象：

​		电话线路——TCP连接

​		语言沟通——数据

​		顾客——数据发送方

​		小姐姐——数据接收方

#### 需要keepalive场景

1. 对端异常崩溃（对应电话订餐场景中顾客临时走开）
2. 对端在但是处理不过来了（对应电话订餐场景中顾客处理其它事情）
3. 对端在但是不可达（对应电话订餐场景中顾客端线路故障）

#### 为什么需要keepalive

​	如果没有keepalive，连接已经坏掉但是还浪费资源维持，下次使用会直接报错。对应A餐馆电话订餐场景，后果就是占用线路，导致其它人不能订餐，影响餐馆收益

#### 为什么TCP有了keepalive应用层还要做keepalive

1. TCP层默认关闭keepalive，且经过路由等中转设备keepalive探测包可能被丢弃
2. TCP层keepalive默认时间过长，虽然可配置，但是该配置是系统参数，会影响所有应用程序
3. 协议分层，各层关注点不一样。传输层关注的是线路通否，而应用层关注的是能否服务

#### TCP的keepalive和HTTP的keepalive有什么不同

​	HTTP1.0的协议都是建立在TCP协议基础上，其特点就是传输完数据后，立马就释放掉该TCP链接，也就是短连接。随着技术的发展，一个网页需要建立很多次短连接，这大大影响了消息的处理，所以HTTP就提出了持续连接的概念，也就是让连接保存一段时间，后续的HTTP消息可以复用这个连接继续传输消息，也就是keepalive模式。当使用keepalive模式时，keepalive功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，避免了建立或者重新建立连接，从而大大提升了性能。

​	keepalive在HTTP1.0中默认是关闭的，需要在请求头加入"Connection: Keep-Alive“；HTTP 1.1中默认启用的，如果加入"Connection: close "才是关闭。目前大部分浏览器都是用HTTP1.1协议，默认都是长连接。

#### idle监测

​	idle监测是一种空闲监测，负责诊断。诊断后做出不同行为，决定了idle监测的最终用途：

 1. 发送keepalive：一般配合keepalive，减少keepalive消息发送

    keepalive设计演进：

    ​	v1：keepalive消息和服务器正常消息交换无关，定时发送。

    ​	v2：有其它消息传输的时候，不发送keepalive消息，无其它消息传输且超过一定时间的时候，判断为idle再发送keepalive消息。

​		v1版本的keepalive设计，如果业务数据交换不多，keepalive消息很有可能超过正常数据交换的频率，明显不合理。而v2版本就有效减少了大量的keepalive消息的发送，更为合理。

	2. 直接关闭连接：快速释放坏掉的、恶意的、很久不用的连接，让系统时刻保持最好的状态。但这比较简单粗暴，客户端可能需要重连。所以实际场景需和keepalive结合使用，保证不会空闲，如果空闲则关闭连接。	

#### Netty开启keepalive+idle监测

```java
ServerBootstrap b = new ServerBootstrap();     
// 两种设置方式
b.option(NioChannelOption.SO_KEEPALIVE, true); 
b.option(ChannelOption.SO_KEEPALIVE, true);

// pipeline添加IdleStateHandler
long readerIdleTime = 0;
long writerIdleTime = 60;
long allIdleTime = 0;
p.addLast(new IdleStateHandler(readerIdleTime, writerIdleTime, allIdleTime, TimeUnit.SECONDS));
```

#### 总结

​	keepalive+idle监测可以减少keepalive的探测消息发送