---
title: MySQL如何保证crash-safe
date: 2021-05-07 21:49:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary:
tags: MySQL
categories:
- [MySQL]
---

## MySQL如何保证crash-safe？

### crash-safe

什么是crash-safe，简单来说就是MySQL异常重启，之前提交的记录也不会丢失。

### 如何保证crash-safe

那么MySQL是如何提供crash-safe这个能力的呢？具体来说，应该是InnoDB提供的这个能力，这就得益于InnoDB的redo log。

redo log是固定大小的一组文件，可通过配置修改数量和文件大小。redo log是物理日志，记录的是数据页上修改了什么内容。这组redo log文件就像一个环形的磁盘，由两个名为write pos和checkpoint的磁头进行控制，write pos表示日志写入的地方，checkpoint表示刷盘的地方，write pos和checkpoint之间的距离就是redo log剩余的空间。如果write pos赶上了checkpoint，那么MySQL就得停下写入，先把日志上的修改刷入磁盘后往前挪动一下checkpoint。当然MySQL不会一直等到write pos追上checkpoint才刷盘，平常如果系统空闲，后台线程就会默默刷盘，如果系统实在繁忙，实在没有空间了，也只能刷盘，这就导致了吞吐量下降。

### redo log 来源

为什么会设计出redo log？也就是鼎鼎大名的WAL(Write-Ahead Logging)技术。其实是在机械硬盘时代，磁盘写入是非常耗时的操作，如果数据库每次修改都修改磁盘，代价实在是太大，这无疑是不能接收的。在这种情景下，首先想到的肯定是在内存中修改即可，但是内存并不能保证数据不丢失，所以单单在内存中修改是肯定不行的。这个时候就想到了，磁盘虽然随机写入性能差，但是顺序写入性能还是可以的，这个时候WAL应运而生了。时至今日WAL设计思路应用已经非常广泛，大名鼎鼎的Kafka的消息存储也利用了这种机制，实现了高效消息存储。

## 小结

* redo log是MySQL的InnoDB存储引擎提供的一种重做日志，能保障MySQL异常重启不丢数据
* redo log是物理日志，记录的是数据页上的修改内容。
* redo log由一组日志文件组成，空间固定，循环写入，由write pos和checkpoint进行控制。
* redo log是一种WAL技术，保证MySQL的高吞吐量。
