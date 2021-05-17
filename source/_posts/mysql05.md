---
title: MySQL更新语句执行流程是怎样？
date: 2021-05-17 22:20:00
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

为了方便描述这个过程，先建个只包含一个主键id和age的表t,并插入一条数据：

```sql
create table t(id int primary key, age int);
insert into t values(1,18);
```

假如现在要更新这条记录，更新语句如下：

```sql
update t set age = 28 where id = 1;
```

执行流程如下：

1. 分析器对SQL语句进行解析，通过解析可以判断SQL语句的正确性，如果不正确则会返回语法错误的提示。
2. 优化器选择合适的执行计划。优化器的作用是在表里有多个索引的时候，决定使用哪个索引。或者语句要做join的时候决定join的顺序。这里优化器会选择使用主键索引检索数据。
3. 执行器执行SQL。在执行前会先判断下当前用户时候有操作该表的权限，如果没有权限则会返回权限错误，否则打开表，根据表定义的引擎去调用引擎的接口。因为id是主键索引，引擎直接使用树就能搜索到这条记录。当然如果这条记录所在的页已经在内存中，就直接返回给执行器。否则就先从磁盘中加载这条记录所在的页到内存中，然后再返回给执行器。
4. 执行器拿到这提条记录后，先调用引擎接口把age=18记录到undo log中并写入磁盘。
5. 执行器调用引擎接口把age值改为28然后再调用引擎接口写入这条记录。
6. 引擎将这条记录更新到内存中，同时将这个更新操作记录到redo log中，并把redo log中的这个事务置于prepare状态。
7. 执行器生成这个操作的binlog并把binlog写入磁盘。
8. 执行器调用引擎提交事务接口，引擎把redo log中的这个事务改为提交状态，更新就完成了。

1. 调用存储引擎获取更新的记录
2. 如果记录不在内存中则加载记录所在的页数据到内存中
3. 写undo log
4. 更新缓存
5. 写redo log处于prepare状态
6. 写binlog
7. 提交事务，commit redo log

上面的过程中有一个耳熟能详的概念，**两阶段提交**，就是上面的redo log先是处于prepare状态后再写binlog，最后再commit。两阶段提交保证了redo log 和bin log逻辑的一致性。

## 小结

* 更新语句执行流程过程中涉及三种日志的写入，分别是undo log、redo log、 binlog。
* 两阶段提交保证了redo log和binlog的逻辑一致。