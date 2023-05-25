---
title: 【从零学Netty】Netty调优：参数调优
date: 2023-05-23 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Netty虽然可调整的参数很多，但是一般应用场景需要调整的很少
tags: Netty
categories:
- [Netty]
---

### Netty参数调优

#### Linux系统参数

| 参数   | 描述                                     | 默认值 | 推荐调整 |
| ------ | ---------------------------------------- | ------ | -------- |
| ulimit | 规定的单个进程能够打开的最大文件句柄数量 | 1024   | **是**   |

#### Netty系统参数

##### ServerSocketChannel参数，通过option来设置

| 参数         | 描述                                                         | 默认值                | 推荐调整         |
| ------------ | ------------------------------------------------------------ | --------------------- | ---------------- |
| SO_RCVBUF    | 为Accept创建的Socket Channel设置SO_RCVBUF,让SO_RCVBUF这个参数更快的生效，因为有可能连接一创建马上就开始接收数据了 | 4KB                   | 否               |
| SO_REUSEADDR | 端口重用                                                     | 默认关闭              | 根据实际场景判断 |
| SO_BACKLOG   | 最大的等待连接数                                             | 默认128               | **是**           |
| IP_TOS       | 设置IP头部的Type-of-service字段，用于描述IP包的优先级和Qos选项 | 0000 - normal service | 否               |

##### SocketChannel参数，通过childOption来设置

| 参数         | 描述                                                         | 默认值                | 推荐调整                         |
| ------------ | ------------------------------------------------------------ | --------------------- | -------------------------------- |
| SO_SNDBUF    | TCP数据发送缓冲区大小                                        | 4KB                   | 否                               |
| SO_RCVBUF    | TCP数据接受缓冲区大小                                        | 4KB                   | 否                               |
| SO_KEEPALIVE | TCP层KEEPALIVE                                               | 默认关闭              | 否                               |
| SO_REUSEADDR | 端口重用，解决"Address already in use"。让关闭连接释放的端口更早可使用。常用于多网卡（IP）绑定相同端口场景。并不是让TCP绑定完全相同的IP+端口来重复启动 | 默认不开启            | 否                               |
| SO_LINGER    | 关闭Socket的延迟时间，默认禁止该功能，Socket.close方法立即返回 | 默认不开启            | 否                               |
| IP_TOS       | 设置IP头部的Type-of-service字段，用于描述IP包的优先级和Qos选项。 | 0000 - normal service | 否                               |
| TCP_NODELAY  | 设置是否启用Nagle算法：将小的碎片数据连接成更大的报文来提高发送速率。如果需要发送一些较小的报文关闭该算法 | 默认关闭              | 根据实际场景判断，一般开启没问题 |

#### Netty核心参数

##### ChannelOption参数，通过option来设置

| 参数                           | 描述                                                         | 默认值                                                       | 推荐调整 |
| ------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | -------- |
| WRITE_BUFFER_WATER_MARK        | 高低水位线、间接防止写数据OOM                                | 底水位32KB，高水位64KB                                       | 否       |
| CONNECT_TIMEOUT_MILLIS         | 客户端连接服务器最大允许时间                                 | 30秒                                                         | **是**   |
| MAX_MESSAGES_PER_READ          | 最大允许连续读次数                                           | 16次                                                         | 否       |
| WRITE_SPIN_COUNT               | 最大允许连续写次数                                           | 16次                                                         | 否       |
| ALLOCATOR                      | ByteBuf分配器，设置Buf分配是池化还是非池化，是堆内存还是堆外内存 | ByteBufAllocator.DEFULT：池化+堆外                           | 否       |
| RCVBUF_ALLOCATOR               | 数据接收ByteBuf分配大小计算器+读次数控制器                   | AdaptiveRecvByteBufAllocator                                 | 否       |
| AUTO_READ                      | 是否监听读事件                                               | 默认打开                                                     | 否       |
| AUTO_CLOSE                     | 写数据失败是否关闭连接                                       | 默认打开。如果失败不关闭的话下次还会写，可能还是失败         | 否       |
| MESSAGE_SIZE_ESTIMATOR         | 数据(ByteBuf、FileRegion等)大小计算器                        | DefaultMessageSizeEstimator.DEFULT                           | 否       |
| SINGLE_EVENTEXECUTOR_PER_GROUP | 当增加一个handler且指定EventExecutorGroup时决定这个handler是否只用EventExecutorGroup中的一个固定的EventExecutor | 默认True。一个handler不管是否共享，都绑定唯一一个EventExecutor。所以小名叫pinEventExecutor。没有指定EventExecutor就复用channel的NioEventLoop | 否       |
| ALLOW_HALF_CLOSURE             | 关闭连接时，允许半关                                         | 默认不允许半关                                               | 否       |

#### Netty系统属性

##### System Property属性，通过-Dio.netty.{xxx}={ooo}来设置

| 参数                         | 描述                    | 默认值                                     | 推荐调整               |
| ---------------------------- | ----------------------- | ------------------------------------------ | ---------------------- |
| io.netty.eventLoopThreads    | IO Thread数量           | 默认:availableProcessors * 2               | 否                     |
| io.netty.availableProcessors | 指定availableProcessors | 读取系统参数，注意Docker获取不准确需要指定 | **Docker部署需要调整** |
| io.netty.allocator.type      | 内存池化还是非池化      | 池化                                       | 否                     |
| io.netty.noPreferDirect      | 内存堆内还是堆外        | 堆外                                       | 否                     |
| io.netty.leakDetection.level | 内存泄露检测级别        | 默认SIMPLE                                 | 否                     |
| ......                       |                         |                                            |                        |

#### Netty参数调优注意点

1. 不懂不要乱动，避免过早优化（所有调参的场景都适用）
2. childOption/option设置错误不会报错,但是不会生效

#### 总结

​	Netty一般应用场景需要调整的参数很少，省心