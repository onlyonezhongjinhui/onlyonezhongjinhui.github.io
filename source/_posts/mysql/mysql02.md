---
title: MySQL有了redo log为何还要binlog
date: 2021-05-07 22:30:00
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

## MySQL有了redo log为何还要binlog？

其实这个问题应该是反过来说的，因为binlog是比redo log更早。当InnoDB引擎还没有的时候，MySQL默认存储引擎是MyISAM。所以binlog是MySQL通用的日志，存在于Server层，不属于引擎层。binlog只有归档能力，也就是归档日志。有了binlog，MySQL就能实现主备同步，数据恢复等。所以说redo log和binlog并不冲突，反而相辅相成。

binlog是逻辑日志，记录的是SQL语句的原始逻辑。它是追加写入模式，当binlog文件到达一定大小会新建立文件进行写入，并不会覆盖原来的文件。记录的格式有三种，statement、row、mixed。对比一下三种格式的优劣。

#### **statement**

记录的是修改的SQL语句，优点是日志文件小，节约IO，挺能高。缺点是准确性差，对一些系统函数不能准确复制或不能复制，入now()、uuid()等。

#### **row**

记录的是每行实际数据的变更，优点是准确性高。缺点就是日志文件大，需要更多的网络IO和磁盘IO。

#### **mixed**

statemen和row混合，优点是准确性高，文件又没row那么大。缺点是可能发生主从不一致。

如今万兆网络带宽高性能磁盘或者SSD加入，推荐使用row模式。毕竟数据一致性还是首先保障。

## 小结

* binlog是server层通用的归档日志。
* binlog是逻辑日志，追加写入，不会覆盖。
* binlog有三种格式，statemen、row、mixed，推荐使用row格式。