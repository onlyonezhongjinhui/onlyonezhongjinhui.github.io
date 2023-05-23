---
title: 从零学架构（三十二）
date: 2021-09-09 21:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 高可用架构三大核心原理
tags: 架构
categories:
- [架构]
---

## 高可用架构三大核心原理

### FLP不可能原理

FLP Impossibility（FLP 不可能性）是分布式领域中一个非常著名的定理，定理的论文是由 Fischer, Lynch
and Patterson 三位作者于1985年发表。It is impossible to have a deterministic protocol that solves consensus in a message-passing asynchronous system in which at most one process may fail by crashing

#### FLP三大限定条件

##### 确定性协议

Deterministic protocol，给定一个输入，一定会产生相同的输出

##### 异步网络通信

没有统一的时钟、不能时间同步、不能使用超时、不能探测失败、消息可任意延迟、消息可乱序

##### 所有存活节点

所有存活节点必须最终达到一致性

**FLP只是说没有“确定性协议”，而不是说“没有任何协议”！**

#### FLP的不可能三角

![](/medias/assets/architecture/20210909201937.png)

### CAP定理

CAP 定理（CAP theorem）又被称作布鲁尔定理（Brewer‘s theorem），是加州大学伯克利分校的计算机科学家埃里克·布鲁尔（Eric Brewer）在2000年的 ACM PODC 上提出的一个猜想。

2002年，麻省理工学院的赛斯·吉尔伯特（Seth Gilbert）和南希·林奇（Nancy Lynch）发表了布鲁尔猜想的证明，使之成为分布式计算领域公认的一个定理。对于设计分布式系统的架构师来说，CAP 是必须掌握的理
论。

It is impossible for a distributed data store to simultaneously provide more than two out of the
following three guarantees: Consistency, Availability, Partition tolerance.
分布式数据存储系统不可能同时满足一致性、可用性和分区容忍性。

#### CAP的三大限定条件

##### 分布式

分布式，可能发生网络分区

##### 数据存储

通过复制来实现数据共享的存储系统

##### 同时满足

同时满足CAP

#### CAP的不可能三角

![](/medias/assets/architecture/20210909202941.png)

![](/medias/assets/architecture/20210909203215.png)

#### CAP细节

##### 复制延迟

![](/medias/assets/architecture/20210909203823.png)

##### 描述粒度

![](/medias/assets/architecture/20210909203941.png)

##### 更多

![](/medias/assets/architecture/20210909204237.png)

### BASE理论

BASE 是指基本可用（Basically Available）、软状态（ Soft State）、最终一致性（ Eventual Consistency），核心思想是即使无法做到强一致性（CAP 的一致性就是强一致性），但应用可以采用适合的方式达到最终一致性。

Basically Available：读写操作尽可能的可用，但写操作在冲突的时候可能丢失结果，读操作可能读取到旧的值。

Soft State：没有一致性的保证，允许系统存在中间状态，而该中间状态不会影响系统整体可用性，这里的中间状态就是CAP 理论中的数据不一致。

Eventually Consistency：如果系统运行正常且等待足够长的时间，系统最终将达成一致性的状态。

#### BASE与CAP

![](/medias/assets/architecture/20210909204748.png)

## BASE、CAP、FLP落地

![](/medias/assets/architecture/20210909205024.png)

## 小结

![](/medias/assets/architecture/%E9%AB%98%E5%8F%AF%E7%94%A8%E6%9E%B6%E6%9E%84%E4%B8%89%E5%A4%A7%E6%A0%B8%E5%BF%83%E5%8E%9F%E7%90%86.png)
