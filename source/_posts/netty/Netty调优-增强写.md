---
title: 【从零学Netty】Netty调优：增强写
date: 2023-05-26 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 牺牲延迟提升吞吐量，推荐直接使用FlushConsolidationHandler增强写
tags: Netty
categories:
- [Netty]
---

### Netty增强写

​		当服务端把业务处理完后需要发送数据给客户端，调用writeAndFlush方法，每写一次就调用一次flush，延迟虽然低但是吞吐量降低了。其实大多数场景不需要那么急着把数据发送出去，可以等一等，多写一些数据再flush，这样稍微增加一点点延迟就能提升吞吐量。

#### 减少flush次数方法

##### channelReadComplete中调用flush

​		处理业务后调用write方法，然后在channelReadComplete方法中调用flush方法。

```java
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.write(msg);
    }

    @Override
    public void channelReadComplete(ChannelHandlerContext ctx) {
        ctx.flush();
    }
```

​		虽然这是一种方式，但是它有缺点：

1. 不适合异步线程处理（业务处理设置独立线程池，不复用NIOEventLoop），因为channelRead中的业务处理的结果write很可能发生在channelReadComplete的后面，这个时候flush就无效了。

​		复用IO线程时(同步)：read->writeAndFlush->readComplete

​		不复用IO线程时(异步)：read->readComplete->writeAndFlush（writeAndFlush很可能发生在readComplete之后

2. 不适合更精细的控制：如果连续读，达到了最大允许连续读次数（默认16次），这个时候就是要flush了。也就是说读16次后就要flush一次。想单独控制连续读的次数和flush次数做不到，因为只有MAX_MESSAGES_PER_READ这个参数去控制。

##### 使用FlushConsolidationHandler

​		在pipeline中增加FlushConsolidationHandler，业务处理发送数据依然使用writeAndFlush方法

```java
    // explicitFlushAfterFlushes=3：调用3次flush后真的调用一次flush
	// consolidateWhenNoReadInProgress=true：开启异步增强写
	p.addLast(new FlushConsolidationHandler(3, true));
    p.addLast(serverHandler);
```

#### 总结

​	对于复用IO线程处理业务的场景，使用在channelReadComplete中flush的方式是可行的，异步业务处理的场景就必须使用FlushConsolidationHandler了。推荐直接使用FlushConsolidationHandler。

