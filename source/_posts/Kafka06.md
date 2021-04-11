---
title: 为什么Kafka不允许减少主题的分区数
date: 2021-04-11 10:00:00
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

## 为什么Kafka不允许减少主题的分区数

回答这个问题前，先看看如何增删查改主题。

### 创建主题

可以通过kafka-topics脚本进行创建，可以添加的参数往后加就是了

```bash
kafka-topics.bat --bootstrap-server localhost:9092 --create --topic test1 --partitions 1 --replication-factor 1
```

创建一个名为test1的主题，分区数为1，副本数为1。

```
kafka-topics.bat --bootstrap-server localhost:9093 --create --topic test2 --partitions 2 --replication-factor 2
```

在创建一个名为test2的主题，分区数为2，副本数为2。

### 查看主题

```
kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic test1
```

```
Topic: test1    PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: test1    Partition: 0    Leader: 1       Replicas: 1     Isr: 1
```

```
kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic test2
```

```
Topic: test2    PartitionCount: 2       ReplicationFactor: 2    Configs: segment.bytes=1073741824
        Topic: test2    Partition: 0    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: test2    Partition: 1    Leader: 0       Replicas: 0,2   Isr: 0,2
```

describe命令可以指定主题名称查询指定主题信息，如果不指定主题名称，则可以查询所有当前发起命令用户可见的主题信息。

```
Topic: test1    PartitionCount: 1       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: test1    Partition: 0    Leader: 1       Replicas: 1     Isr: 1
Topic: test2    PartitionCount: 2       ReplicationFactor: 2    Configs: segment.bytes=1073741824
        Topic: test2    Partition: 0    Leader: 1       Replicas: 1,0   Isr: 1,0
        Topic: test2    Partition: 1    Leader: 0       Replicas: 0,2   Isr: 0,2
```

### 修改主题

#### 增加副本

增加副本使用kafka-reassign-partitions脚本，且需要创建一个包含分区信息的json文件。比如下面给test1增加两个副本

```json
{
    "version":1,
    "partitions":[
        {
            "topic":"test1",
            "partition":0,
            "replicas":[
                0,
                1,
                2
            ]
        }
    ]
}
```

执行命令

```
kafka-reassign-partitions.bat --bootstrap-server localhost:9092 --reassignment-json-file test1.json --execute
```

看到响应内容表示成功

```
Current partition replica assignment

{"version":1,"partitions":[{"topic":"test1","partition":0,"replicas":[1],"log_dirs":["any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started partition reassignment for test1-0
```

再次查看test1主题

```
kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic test1

Topic: test1    PartitionCount: 1       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: test1    Partition: 0    Leader: 0       Replicas: 0,1,2 Isr: 1,2,0

```

看到test1已变为3个副本，分别是0,1,2。

再给test2主题增加副本

```json
{
    "version":1,
    "partitions":[
        {
            "topic":"test2",
            "partition":0,
            "replicas":[
                0,
				1,
                2
            ]
        },
		 {
            "topic":"test2",
            "partition":1,
            "replicas":[
                0,
                1,
                2
            ]
        }
    ]
}
```

```
kafka-reassign-partitions.bat --bootstrap-server localhost:9092 --reassignment-json-file test2.json --execute
```

查看test2主题，发现副本数已变为3

```
kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic test2
Topic: test2    PartitionCount: 2       ReplicationFactor: 3    Configs: segment.bytes=1073741824
        Topic: test2    Partition: 0    Leader: 1       Replicas: 0,1,2 Isr: 1,0,2
        Topic: test2    Partition: 1    Leader: 0       Replicas: 0,1,2 Isr: 0,2,1
```

#### 删除副本

编辑分区信息文件

```json
{
    "version":1,
    "partitions":[
        {
            "topic":"test2",
            "partition":0,
            "replicas":[
                0,
				1
            ]
        },
		 {
            "topic":"test2",
            "partition":1,
            "replicas":[
                0,
                1
            ]
        }
    ]
}
```

执行命令进行修改

```
kafka-reassign-partitions.bat --bootstrap-server localhost:9092 --reassignment-json-file test2.json --execute

Current partition replica assignment

{"version":1,"partitions":[{"topic":"test2","partition":0,"replicas":[0,1,2],"log_dirs":["any","any","any"]},{"topic":"test2","partition":1,"replicas":[0,1,2],"log_dirs":["any","any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started partition reassignments for test2-0,test2-1
```

查看test2主题信息，已变为了两个副本

```
kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic test2
[2021-04-11 13:55:23,465] WARN [AdminClient clientId=adminclient-1] Connection to node 2 (DESKTOP-3LGVDML/192.168.0.100:9094) could not be established. Broker may not be available. (org.apache.kafka.clients.NetworkClient)
Topic: test2    PartitionCount: 2       ReplicationFactor: 2    Configs: segment.bytes=1073741824
        Topic: test2    Partition: 0    Leader: 0       Replicas: 0,1   Isr: 1,0
        Topic: test2    Partition: 1    Leader: 0       Replicas: 0,1   Isr: 0,1
```

再尝试改为1个副本

```json
{
    "version":1,
    "partitions":[
        {
            "topic":"test2",
            "partition":0,
            "replicas":[
                0
            ]
        },
		 {
            "topic":"test2",
            "partition":1,
            "replicas":[
                0
            ]
        }
    ]
}
```

```

kafka-reassign-partitions.bat --bootstrap-server localhost:9092 --reassignment-json-file test2.json --execute
Current partition replica assignment

{"version":1,"partitions":[{"topic":"test2","partition":0,"replicas":[0,1],"log_dirs":["any","any"]},{"topic":"test2","partition":1,"replicas":[0,1],"log_dirs":["any","any"]}]}

Save this to use as the --reassignment-json-file option during rollback
Successfully started partition reassignments for test2-0,test2-1
```

```
kafka-topics.bat --bootstrap-server localhost:9092 --describe --topic test2
Topic: test2    PartitionCount: 2       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: test2    Partition: 0    Leader: 0       Replicas: 0     Isr: 0
        Topic: test2    Partition: 1    Leader: 0       Replicas: 0     Isr: 0
```

#### 修改分区

通过kafka-topics脚本配合--alter参数进行进行修改。

给主题test1增加一个分区

```
kafka-topics.bat --bootstrap-server localhost:9092  --alter --topic test1 --partitions 2

Topic: test1    PartitionCount: 2       ReplicationFactor: 1    Configs: segment.bytes=1073741824
        Topic: test1    Partition: 0    Leader: 0       Replicas: 0     Isr: 0
        Topic: test1    Partition: 1    Leader: 0       Replicas: 0     Isr: 0

```

给主题test1删除一个分区

```
kafka-topics.bat --bootstrap-server localhost:9092  --alter --topic test1 --partitions 1
Error while executing topic command : Topic currently has 2 partitions, which is higher than the requested 1.
[2021-04-11 14:18:57,294] ERROR org.apache.kafka.common.errors.InvalidPartitionsException: Topic currently has 2 partitions, which is higher than the requested 1.
 (kafka.admin.TopicCommand$)

```

可以看到，kafka并不允许删除分区，执行后报错了，提示需要超过2。

### 删除主题

使用kafka-topic脚本配合--delete参数进行删除

```
kafka-topics.bat --bootstrap-server localhost:9092 --delete --topic test1
```

注意，删除主题是异步的，命令执行后kafka会在后台默默进行删除。

由于上面修改主题的操作可能设置出现了问题，当执行上面删除操作的时候发现主题删除不了且Broker宕机了。遇到这种情况，只能采取非常措施了。

1. 手动删除Zookeeper节点/admin/delete_topics 下待删除主题为名的znode
2. 然后再手动删除该主题的磁盘上的分区目录
3. 最后在Zookeeper中执行rmr/controller，触发Controller重选举，刷新Controller缓存。在执行最后一步时，要慎重，因为他可能造成大面积的分区Leader重选举。事实上，仅仅执行前两步也是可以的，只是Controller缓存中没有清空删除主题，不影响使用。

**好了，回答开篇的问题，“为什么Kafka不允许减少主题的分区数”**

**照Kafka现有的代码逻辑而言，此功能完全可以实现，不过也会使得代码的复杂度急剧增大。实现此功能需要考虑的因素很多，比如删除掉的分区中的消息该作何处理？如果随着分区一起消失则消息的可靠性得不到保障；如果需要保留则又需要考虑如何保留。直接存储到现有分区的尾部，消息的时间戳就不会递增，如此对于Spark、Flink这类需要消息时间戳（事件时间）的组件将会受到影响；如果分散插入到现有的分区中，那么在消息量很大的时候，内部的数据复制会占用很大的资源，而且在复制期间，此主题的可用性又如何得到保障？与此同时，顺序性问题、事务性问题、以及分区和副本的状态机切换问题都是不得不面对的。反观这个功能的收益点却是很低，如果真的需要实现此类的功能，完全可以重新创建一个分区数较小的主题，然后将现有主题中的消息按照既定的逻辑复制过去即可**

## 小结

1. Kafka可以通过kafka-topic脚本配合参数--create、--describe、--alter、--delete增删查改主题
2. Kafka可以通过kafka-reassign-partitions脚本配参数--reassignment-json-file指定带有分区信息的json文件进行副本数的变更
3. Kafka只允许增加分区不允许减少分区