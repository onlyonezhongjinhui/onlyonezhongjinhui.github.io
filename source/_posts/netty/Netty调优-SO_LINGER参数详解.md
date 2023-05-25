---
title: 【从零学Netty】Netty调优：SO_LINGER参数详解
date: 2023-05-24 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 一般我们不需要开启SO_LINGER参数
tags: Netty
categories:
- [Netty]
---

### SO_LINGER参数详解
#### ![](/medias/assets/netty/so_linger.png)

TCP协议中，当应用层调用close方法关闭连接的时候，会发送FIN包给对方。默认情况下close会立即返回，并由TCP模块负责将发送缓冲区中的残留数据发送出去，应用层是无法知道缓冲区中的数据是否成功发送完成。SO_LINGER参数可以控制调用close后的行为。SO_LINGER选项结构如下

```c#
struct linger {

     int l_onoff;

     int l_linger;

};
```

有如下三种设置：

1. 设置 l_onoff为0，l_linger的值被忽略，这是内核默认设置。close调用会立即返回给调用者，TCP模块负责尝试发送残留的缓冲区数据

2. 设置 l_onoff为1，l_linger为0。表示立即终止连接，TCP将丢弃残留在发送缓冲区中的数据并发送一个RST分组给对方，而不是正常的的FIN|ACK|FIN|ACK四个分组来关闭连接。这避免了TIME_WAIT状态，直接进入CLOSED状态。

3. 设置 l_onoff 为1，l_linger > 0：

   （a）、阻塞IO：close将阻塞等待l_linger秒的时间，如果在l_linger秒时间内，TCP模块成功发送完残留的缓冲区数据，则close返回0，表示成功。如果l_linger时间内，TCP模块没有成功发送完残留的缓冲区数据，则close返回-1，表示失败，并将errno设置为EWOULDBLOCK。 

   （b）、非阻塞IO：close立即返回，此时需要根据close返回值以及errno来判断TCP模块是否成功发送残留的缓冲区数据。

#### 总结

​	我们为了更好的性能而选择异步IO，如果开启SO_LINGER参数后close就变成阻塞的了，close就变慢了从而影响性能。那是否意味着我们的可靠性降低了呢？其实不然，大多数情况下我们close掉channel后并不会把程序结束掉，后面的TCP模块依然会继续发送数据，网络正常的情况下一般都能把数据发送完。所以一般我们不需要开启SO_LINGER参数。

