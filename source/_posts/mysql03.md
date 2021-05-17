---
title: 如果binlog格式为mixed，语句insert into t values(1,1,now())会记录为row格式还是statement格式？
date: 2021-05-08 21:49:00
author: maybe
top: false
cover: true
toc: false
mathjax: false
summary:
tags: MySQL
categories:
- [MySQL]
---

## 如果binlog格式为mixed，语句insert into t values(1,1,now())会记录为row格式还是statement格式？

MySQL的binlog有三种模式，statement、row、mixed。先做个实验，看看他们的庐山真面目吧。（本次实验是在MySQL8.0版本上进行）。

MySQL默认binlog日志为mixed。

```sql
show variables like '%binlog_format%' \G;
*************************** 1. row ***************************
Variable_name: binlog_format
        Value: ROW
1 row in set, 1 warning (0.25 sec)

```

创建测试表t

```sql
CREATE TABLE `t`
(
    `id`         int(11)   NOT NULL,
    `a`          int(11)            DEFAULT NULL,
    `t_modified` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`),
    KEY `a` (`a`),
    KEY `t_modified` (`t_modified`)
) ENGINE = InnoDB;
```

执行插入语句

```sql
insert into t values(1,1,now());
Query OK, 1 row affected (0.32 sec)
```

查看binlog，笔者实验实例上最新的binlog文件是binlog.000409，通过命令查看

```sql
show binlog events in 'binlog.000409';
```

结果如下

```sql
| binlog.000409 | 1108 | Anonymous_Gtid |         1 |        1183 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                     |
| binlog.000409 | 1183 | Query          |         1 |        1268 | BEGIN                                                                                                                                                                                                                                    |
| binlog.000409 | 1268 | Table_map      |         1 |        1318 | table_id: 71 (test.t)                                                                                                                                                                                                                    |
| binlog.000409 | 1318 | Write_rows     |         1 |        1366 | table_id: 71 flags: STMT_END_F                                                                                                                                                                                                           |
| binlog.000409 | 1366 | Xid            |         1 |        1397 | COMMIT /* xid=17 */  
```

使用mysqlbinlog工具查看详细日志

```bash
mysqlbinlog --no-defaults --database=test F:\Mr.Zhong\software\mysql-8.0.13-winx64\data\binlog.00040
```

结果如下

```sql
BEGIN
/*!*/;
# at 1268
#210510 20:48:40 server id 1  end_log_pos 1318 CRC32 0x9eb42e6e         Table_map: `test`.`t` mapped to number 71
# at 1318
#210510 20:48:40 server id 1  end_log_pos 1366 CRC32 0x9101035b         Write_rows: table id 71 flags: STMT_END_F

BINLOG '
qCuZYBMBAAAAMgAAACYFAAAAAEcAAAAAAAEABHRlc3QAAXQAAwMDEQEAAgEBAG4utJ4=
qCuZYB4BAAAAMAAAAFYFAAAAAEcAAAAAAAEAAgAD/wABAAAAAQAAAGCZK6hbAwGR
'/*!*/;
# at 1366
#210510 20:48:40 server id 1  end_log_pos 1397 CRC32 0x7b4fa6d4         Xid = 17
COMMIT/*!*/;
SET @@SESSION.GTID_NEXT= 'AUTOMATIC' /* added by mysqlbinlog */ /*!*/;
DELIMITER ;
# End of log file
/*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
/*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
```

**可以看到日志记录的是row格式。实践出真知，在MySQL8.0上，insert into t values(1,1,now());语句记录为row格式。**

接下来解读一下日志里面包含的重要内容：

* 第一行SET @@SESSION.GTID_NEXT= 'ANONYMOUS'，是用来主备切换时确定binlog位点
* 第二行的BEGIN和第五行的COMMIT表示中间的是一个事务，xid表示成功提交事务，事务id为17
* 第三行是Table_map事件，用来说明接下来操作的是test库的表t
* 第四行是Write_rows事件，是插入数据

从mysqlbinlog工具解析出来还看到更加完整的信息，里面包含了

* server id 1，说明这个事务是在server id 为1的实例上执行的
* 每个事件event都有CRC32值，这是因为设置binlog-checksum默认是CRC32

接下来看看Statement格式的日志会记录成什么样子呢？

修改binlog格式为Statement

```sql
SET SESSION binlog_format = 'statement';

show variables like '%binlog_format%';
+---------------+-----------+
| Variable_name | Value     |
+---------------+-----------+
| binlog_format | STATEMENT |
```

执行insert into t values(1,1,now());语句后查看binlog

```sql
                                                             |
| binlog.000409 | 2026 | Anonymous_Gtid |         1 |        2101 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                     |
| binlog.000409 | 2101 | Query          |         1 |        2193 | BEGIN                                                                                                                                                                                                                                    |
| binlog.000409 | 2193 | Query          |         1 |        2311 | use `test`; insert into t values(1,1,now())                                                                                                                                                                                              |
| binlog.000409 | 2311 | Xid            |         1 |        2342 | COMMIT /* xid=38 */                                                                                                                   
```

```sql
/*!80001 SET @@session.original_commit_timestamp=1620654173918469*//*!*/;
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 2101
#210510 21:42:53 server id 1  end_log_pos 2193 CRC32 0x215227b1         Query   thread_id=10    exec_time=0     error_code=0
SET TIMESTAMP=1620654173/*!*/;
/*!80013 SET @@session.sql_require_primary_key=0*//*!*/;
BEGIN
/*!*/;
# at 2193
#210510 21:42:53 server id 1  end_log_pos 2311 CRC32 0xdf2dd0df         Query   thread_id=10    exec_time=0     error_code=0
SET TIMESTAMP=1620654173/*!*/;
/*!80013 SET @@session.sql_require_primary_key=0*//*!*/;
insert into t values(1,1,now())
/*!*/;
# at 2311
#210510 21:42:53 server id 1  end_log_pos 2342 CRC32 0xbc576b08         Xid = 38
COMMIT/*!*/;
```

可以看到statement格式的binlog是直接记录了原始的SQL语句，值得注意的是，在记录语句之前还记录了一个重要的一行**SET TIMESTAMP=1620654173/*!*/;**，这行的作用就是表明，后续的now()函数的返回时间就是这个时间，这样就能保证从库使用该binlog进行同步的时候不会出现主备不一致。

再来看看这条SQL语句，**delete from t  where a >= 4 and t_modified <= '2018-11-10' limit 1**在statement格式下的表现。

```sql
| binlog.000409 | 3498 | Anonymous_Gtid |         1 |        3573 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                     |
| binlog.000409 | 3573 | Query          |         1 |        3665 | BEGIN                                                                                                                                                                                                                                    |
| binlog.000409 | 3665 | Query          |         1 |        3818 | use `test`; delete from t  where a >= 4 and t_modified <= '2018-11-10' limit 1                                                                                                                                                           |
| binlog.000409 | 3818 | Xid            |         1 |        3849 | COMMIT /* xid=54 */  
```

```sql
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 3573
#210511 20:09:25 server id 1  end_log_pos 3665 CRC32 0x3275901b         Query   thread_id=12    exec_time=0     error_code=0
SET TIMESTAMP=1620734965/*!*/;
/*!80013 SET @@session.sql_require_primary_key=0*//*!*/;
BEGIN
/*!*/;
# at 3665
#210511 20:09:25 server id 1  end_log_pos 3818 CRC32 0x73fa16d2         Query   thread_id=12    exec_time=0     error_code=0
SET TIMESTAMP=1620734965/*!*/;
/*!80013 SET @@session.sql_require_primary_key=0*//*!*/;
delete from t  where a >= 4 and t_modified <= '2018-11-10' limit 1
/*!*/;
# at 3818
#210511 20:09:25 server id 1  end_log_pos 3849 CRC32 0x4ded88af         Xid = 54
COMMIT/*!*/;
```

貌似没啥问题的样子，但是从语句我可以分析到，如果删除的时候主备走的索引不一致，删除的数据行就是不确定的，这会产生主备不一致。那么就来验证一下是不是如猜测的那样。

运行命令show warnings

```sql
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Level | Code | Message                                                                                                                                                                                                                         |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Note  | 1592 | Unsafe statement written to the binary log using statement format since BINLOG_FORMAT = STATEMENT. The statement is unsafe because it uses a LIMIT clause. This is unsafe because the set of rows included cannot be predicted. |
+-------+------+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

可以看到产生了一个warning，原因是在statement格式下，上述的语句带了limit，无法精确确定删除的记录，是不安全的。

那我们改为Row格式再尝试后再看看是不是就ok了呢？

查看warning信息、binlog，发现warning没有了，binlog也正常

```sql
mysql> show warnings;
Empty set (0.00 sec)
```

```sql
                                                             |
| binlog.000409 | 7770 | Anonymous_Gtid |         1 |        7845 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'                                                                                                                                                                                                     |
| binlog.000409 | 7845 | Query          |         1 |        7930 | BEGIN                                                                                                                                                                                                                                    |
| binlog.000409 | 7930 | Table_map      |         1 |        7980 | table_id: 76 (test.t)                                                                                                                                                                                                                    |
| binlog.000409 | 7980 | Delete_rows    |         1 |        8028 | table_id: 76 flags: STMT_END_F                                                                                                                                                                                                           |
| binlog.000409 | 8028 | Xid            |         1 |        8059 | COMMIT /* xid=76 */                                                                                                                       
```

```bash
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;
# at 7845
#210511 20:32:08 server id 1  end_log_pos 7930 CRC32 0xadbda736         Query   thread_id=12    exec_time=0     error_code=0
SET TIMESTAMP=1620736328/*!*/;
/*!80013 SET @@session.sql_require_primary_key=0*//*!*/;
BEGIN
/*!*/;
# at 7930
#210511 20:32:08 server id 1  end_log_pos 7980 CRC32 0x9415caa5         Table_map: `test`.`t` mapped to number 76
# at 7980
#210511 20:32:08 server id 1  end_log_pos 8028 CRC32 0xce4cf023         Delete_rows: table id 76 flags: STMT_END_F

BINLOG '
SHmaYBMBAAAAMgAAACwfAAAAAEwAAAAAAAEABHRlc3QAAXQAAwMDEQEAAgEBAKXKFZQ=
SHmaYCABAAAAMAAAAFwfAAAAAEwAAAAAAAEAAgAD/wAEAAAABAAAAFvlrwAj8EzO
'/*!*/;
# at 8028
#210511 20:32:08 server id 1  end_log_pos 8059 CRC32 0x3efcb90c         Xid = 76
COMMIT/*!*/;
```

## 小结

* MySQL8 binlog默认格式是Mixed。
* 包含日期这种不一致的因素的语句，MySQL8也做了相应的优化，通过SET TIMESTAMP命令保持主备一致
* 对于statement格式下的binlog会有主备不一致的隐患，建议binlog至少设置为Mixed，当然推荐还是设置为row格式