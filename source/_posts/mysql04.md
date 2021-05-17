---
title: undo log有什么用？
date: 2021-05-17 21:30:00
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

## undo log

undo log用来存放数据被修改前的值。为了便于说明，建立一个表t，并插入一条数据

```sql
CREATE TABLE `t`
(
    `id`         int(11)   NOT NULL,
    `a`          int(11)            DEFAULT NULL,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB;
insert into t values (1,1);
```

假如现在要更新这条记录

```sql
update t set a = 2 where id = 1;
```

那么这个时候undo log就会记录下事务T修改前a的值为1，修改后的值为2。如果这个事务出现异常需要回滚，就可以使用undo log来实现回滚，这样就能保证事务的一致性。

insert、update、delete可以变更数据，undo log分为两种类型，一种是insert_undo,一种是update_undo。

## undo log相关参数

```sql
mysql> show variables like '%undo%';
+--------------------------+------------+
| Variable_name            | Value      |
+--------------------------+------------+
| innodb_max_undo_log_size | 1073741824 |
| innodb_undo_directory    | .\         |
| innodb_undo_log_encrypt  | OFF        |
| innodb_undo_log_truncate | ON         |
| innodb_undo_tablespaces  | 2          |
+--------------------------+------------+

mysql> show global variables like '%truncate%';
+--------------------------------------+-------+
| Variable_name                        | Value |
+--------------------------------------+-------+
| innodb_purge_rseg_truncate_frequency | 128   |
| innodb_undo_log_truncate             | ON    |
+--------------------------------------+-------+
```

* **innodb_max_undo_log_size**

最大undo tablespace文件的大小，当启动了innodb_undo_tablespaces时，undo tablespace超过innodb_max_undo_log_size的阈值时才会尝试truncate。默认值是1G。truncate后的大小默认为10M。

* innodb_undo_directory

undo log存放目录

* innodb_undo_log_encrypt

日志加密开关，默认关闭

* innodb_undo_log_truncate

InnoDB的purge线程，根据innodb_undo_log_truncate设置开启或关闭、innodb_max_undo_log_size的参数值，以及truncate的频率来进行空间回收和 undo file 的重新初始化。注意该参数生效的前提是，已设置独立表空间且独立表空间个数大于等于2个。

purge线程在truncate undo log file的过程中，需要检查该文件上是否还有活动事务，如果没有，需要把该undo log file标记为不可分配，这个时候，undo log 都会记录到其他文件上，所以至少需要2个独立表空间文件，才能进行truncate 操作，标注不可分配后，会创建一个独立的文件undo_<space_id>_trunc.log，记录现在正在truncate 某个undo log文件，然后开始初始化undo log file到10M，操作结束后，删除表示truncate动作的 undo_<space_id>_trunc.log 文件，这个文件保证了即使在truncate过程中发生了故障重启数据库服务，重启后，服务发现这个文件，也会继续完成truncate操作，删除文件结束后，标识该undo log file可分配。

* innodb_undo_tablespaces

设置undo独立表空间个数，范围为0-128，该参数只能在最开始初始化MySQL实例的时候指定，如果实例已创建，这个参数是不能变动的，如果在数据库配置文 件中指定innodb_undo_tablespaces 的个数大于实例创建时的指定个数，则会启动失败，提示该参数设置有误。默认值是2，所以数据目录下有两个文件，undo_001、undo_002，每个文件大小为12M。

* **innodb_purge_rseg_truncate_frequency**

控制purge回滚段的频度，默认为128。假设设置为n，则说明，当Innodb Purge操作的协调线程 purge事务128次时，就会触发一次History purge，检查当前的undo log 表空间状态是否会触发truncate。

## undo log 空间管理

如果需要设置独立表空间，需要在初始化数据库实例的时候，指定独立表空间的数量。

undo log内部由多个回滚段组成，即 Rollback segment，一共有128个，保存在ibdata系统表空间中，分别从resg slot0 - resg slot127，每一个resg slot，也就是每一个回滚段，内部由1024个undo segment 组成。

回滚段（rollback segment）分配如下：

* slot 0 ，预留给系统表空间；
* slot 1- 32，预留给临时表空间，每次数据库重启的时候，都会重建临时表空间；
* slot33-127，如果有独立表空间，则预留给UNDO独立表空间；如果没有，则预留给系统表空间；

回滚段中除去32个提供给临时表事务使用，剩下的 128-32=96个回滚段，可执行 96*1024 个并发事务操作，每个事务占用一个 undo segment slot，注意，如果事务中有临时表事务，还会在临时表空间中的 undo segment slot 再占用一个 undo segment slot，即占用2个undo segment slot。如果错误日志中有：`Cannot find a free slot for an undo log。`则说明并发的事务太多了，需要考虑下是否要分流业务。

回滚段（rollback segment ）采用 轮询调度的方式来分配使用，如果设置了独立表空间，那么就不会使用系统表空间回滚段中undo segment，而是使用独立表空间的，同时，如果回滚段正在 Truncate操作，则不分配。

## 小结

* undo log记录事务修改的相反面，用来做事务回滚。
* 通过回滚段可以构建版本视图（MVCC），是MVCC的基础。
* undo log日志在事务不需要的时候才能删除。长事务导致回滚日志不能删除，占用大量内存。所以尽量不要使用长事务。