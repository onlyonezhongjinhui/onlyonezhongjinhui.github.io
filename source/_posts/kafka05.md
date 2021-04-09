---
title: Kafka集群参数
date: 2021-04-09 23:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary:
tags: Kafka
categories:
- [MQ]
---

## Kafka集群参数

2.7版本的kafka的有216个参数，这些参数主要分为以下7大类

1. Broker Configs
2. Topic Configs
3. Producer Configs
4. Consumer Configs
5. Kafka Connect Configs
6. Kafka Streams Configs
7. AdminClient Configs

当然那么多参数并不需要每个都要掌握，只需要掌握关键的参数就可以应付日常的生产了。笔者就列举一些关键的必须掌握的参数，一起讨论讨论，注意各版本Kafka配置会不一样，所以没特殊说明，这里说的都是2.7版本的配置

### Broker Configs

#### broker.id

Broker的唯一标识，这个肯定是要全局唯一。

#### log.dirs

指定Kafka使用的文件目录路径，是一个集合，多个路径用逗号分隔，比如/temp/kafka01,/temp/kafka02。如果有条件的话把这些目录分别挂载到不同的物理磁盘上，这样多块硬盘读写可以提升读写性能，且能实现故障转移。当Broker上的一块磁盘坏掉，Kafka会将坏掉的硬盘的数据转移到正常的磁盘上，这样Broker就实现了故障转移，还能继续工作。

#### zookeeper.connect

指定Zookeeper连接字符串，可以配置多个Zookeeper，多个用逗号分隔即可，比如

```properties
hostname1:port1,hostname2:port2,hostname3:port3
```

当你有多套Kafka集群想共用一个Zookeeper环境的时候，可以给连接字符串加上ZooKeeper chroot路径，作为命名空间进行区分，比如

```properties
hostname1:port1,hostname2:port2/kafka1,hostname1:port1,hostname2:port2/kafka2
```

#### listeners

监听器，指定可以通过什么协议、主机名、端口连接Kafka服务。格式为<协议名称，主机名，端口号>，多个配置用逗号分隔即可。协议名称有两个可选值，PLAINTEXT、SSL。PLAINTEXT表示明文传输，SSL表示用SSL或TLS加密传输。当然也可以自定义协议，但是就需要把listener.security.protocol.map参数也配置上，告诉这个协议使用了哪种安全协议。比如

```properties
listener.security.protocol.map=CLIENT:SSL
```

表示CLIENT这个自定义协议底层使用加密传输数据。

#### advertised.listeners

和 listeners 相比多了个 advertised。Advertised的含义表示宣称的、公布的，就是说这组监听器是Broker用于对外发布的。Broker启动后会向Zookeeper中注册自己，同时从Zookeeper中获取兄弟Broker的地址，以便和兄弟Broker通信。同样，客户端连接Kafka后，也会从Zookeeper中获取所有Broker的访问地址。而这些地址就是advertised.listeners参数提供的。注意，当advertised.listeners没有配置的时候，默认就会使用listeners。这个参数比较难以理解，对比以下两个使用场景，结合起来更加容易理解。

* Kafka集群仅供内网使用

那么只需要配置listeners参数

```properties
listeners=PLAINTEXT://192.168.0.54:9092
```

* Kafka集群供外网访问

当然如果宿主机有外网网卡，直接配置外网就可以了。如果没有外网网卡，（ifconfig看不到外网ip的网卡，基本上就不存在这个外网网卡），很可能是通过NAT映射或者啥办法搞出来的外网ip，此时kafka无法监听这个外网ip（因为不存在，启动就会报错）。这时候就是advertised.listeners真正发挥作用的时候了。配置如下：

```properties
listeners=PLAINTEXT://192.168.0.54:9092
advertised.listeners=PLAINTEXT://10.89.239.1:9092
```

这时候，客户端访问的流程如下：

1. 客户端访问10.89.239.1:9092就被Kafka宿主机映射到了内网的192.168.0.54:9092上，连上了Kafka节点请求获取Kafka服务端的访问地址。
2. Kafka从Zookeeper中把自己和其它节点通过advertised.listeners注册到Zookeeper中的外网地址返回给客户端。
3. 客户端通过这些外网地址就可以访问到Kafka了。

#### auto.create.topics.enable

是否允许自动创建 Topic，建议配置为false，避免不小心错误的创建Topic。比如启动一个生产者程序，本来是要给名为order的topic发送消息，但是由于疏忽拼错了，发成了oder，那么如果这个参数配置的是true，那么就会自动创建一个oder的topic，这不是我们想看到的结果，所以配置为false更合适。很遗憾，auto.create.topics.enable默认值就是true，所以最好把它改为false。

#### unclean.leader.election.enable

是否开启unclean选举。默认值是false。unclean选举意思是，当topic的ISR集合为空，也就是和挂掉的领导者副本同步比较相近的副本和领导者都挂掉的时候， 是否允许落后较多的副本成为领导者。如果该参数配置的是false，那么就是坚决不允许，如果是true则就会允许，那么这个时候让一个落后较多的副本成为领导者，那么丢数据那就是必然的事情了。所以设置为false是最可靠的。

#### auto.leader.rebalance.enable

是否允许Kafka定期的对一些分区进行Leader重选举，当然这是有特定条件的，要满足才会冲选举。该参数默认值是true，建议设置为false。理由是，领导者副本a也许一直表现得都挺好，但是有可能一段时间后就被强行卸任领导者换成了b，这就导致了无缘无故得性能开销。你要知道换一次 Leader 代价很高的，原本向 a 发送请求的所有客户端都要切换成向 b 发送请求，而且这种换 Leader 本质上没有任何性能收益，建议在生产环境中把这个参数设置成 false。

#### log.retention.hours、log.retention.minutes、log.retention.ms

用来指定消息保留得时长。默认配置是168个小时，也就是7天。

```properties
log.retention.hours = 168
log.retention.minutes = null
log.retention.ms = null
```

这三个参数优先级从高到低分别是log.retention.hours、log.retention.minutes、log.retention.ms。

#### log.retention.bytes

指定 Broker 为消息保存的总磁盘容量大小。默认值是-1，表示没有限制。当需要限制磁盘容量大小的时候可以使用。

#### message.max.bytes

指定一条消息最大字节数。默认值是1048588byte约1M，这个可根据具体消息大小来设置即可。

### Topic Configs

Topic级别的参数可以覆盖全局Broker的参数，当需要根据Topic设置不同的参数时候，可以使用Topic级别参数覆盖Broker的全局参数。

#### retention.ms

消息的保留时长，设置后会覆盖Broker全局的参数。默认值是604800000，也就是7天。

#### retention.bytes

指定Topic可以使用的最大磁盘空间。默认值是-1，表示无限制。

### JVM Configs

Kafka是跑在JVM上的，配置最重要的参数莫过设置合适的堆空间大小，以及合适的垃圾收集器。现在一般最低也是使用JDK8以上了。Kafka运行过程中会创建大量的ByteBuffer实例，配置一个较大的堆空间还是比较有必要的，建议不要低于6G吧。然后就是垃圾收集器，无需什么调优的情况下就无脑使用G1就好了。相对较大的堆空间使用G1还是比较有优势的。

#### 操作系统参数

* 文件描述符限制：尽量设置一个较大的数，比如ulimit -n 1000000。
* 文件系统类型：ext3->ext4->XFS->ZFS,性能由低到高，有条件就选择更好的。
* Swappiness：建议不要设置为0而是设置一个较小的值。因为完全禁用swap空间，一旦物理内存耗尽，操作系统就会触发OOM killer这个组件，它会随机挑选一个进程然后kill掉。但如果设置成一个比较小的值，当开始使用 swap 空间时，你至少能够观测到 Broker 性能开始出现急剧下降，从而给你进一步调优和诊断问题的时间。基于这个考虑，建议将 swappniess 配置成一个接近 0 但不为 0 的值，比如 1。
* 提交时间：提交时间或者说是 Flush 落盘时间。向 Kafka 发送数据并不是真要等数据被写入磁盘才会认为成功，而是只要数据被写入到操作系统的页缓存（Page Cache）上就可以了，随后操作系统根据 LRU 算法会定期将页缓存上的“脏”数据落盘到物理磁盘上。这个定期就是由提交时间来确定的，默认是 5 秒。一般情况下我们会认为这个时间太频繁了，可以适当地增加提交间隔来降低物理磁盘的写操作。当然你可能会有这样的疑问：如果在页缓存中的数据在写入到磁盘前机器宕机了，那岂不是数据就丢失了。的确，这种情况数据确实就丢失了，但鉴于 Kafka 在软件层面已经提供了多副本的冗余机制，因此这里稍微拉大提交间隔去换取性能还是一个合理的做法

## 小结

上面略列了Kafka常用的且比较关键的参数，包括了Broker Configs、Topic Configs 、JVM Configs、操作系统方面的，并且提出了以下最佳实践的配置，不过切记要根据实际情况进行配置哦。

##### 本文参考了胡夕老师的Kafka核心技术与实践专栏。