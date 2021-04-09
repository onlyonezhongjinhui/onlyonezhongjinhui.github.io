---
title: Kafka集群安装演示
date: 2021-04-09 09:00:00
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

搞清楚了Kafka如何安装部署，那么我们就来个具体实操，做个演示环境，方便后续的使用。这里笔者平常使用Windows，为了方便，这里就直接在Windows上搭建演示集群了。
1. 官网上下载最新2.7版本的Kafka，解压缩后复制三份。
2. 这里我选择偷懒，直接使用Kafka自带的Zookeeper。选择其中一个Kafka，修改zookeeper.properties配置文件，当然也可以不修改采用默认配置即可。这里我只修改一下zookeeper的数据路径dataDir=./../../zookeeper
```properties
  # the directory where the snapshot is stored.
  dataDir=./../../zookeeper
  # the port at which the clients will connect
  clientPort=2181
  # disable the per-ip limit on the number of connections since this is a non-production config
  maxClientCnxns=0
  # Disable the adminserver by default to avoid port conflicts.
  # Set the port to something non-conflicting if choosing to enable this
  admin.enableServer=false
```
3. 分别修改三个Kafka的server.properties配置文件。只需修改broker.id、log.dirs、listeners这三个参数就能跑起来了。
* broker.id:broker的唯一标识，不能重复
* log.dirs：指定Broker需要使用的若干个文件目录路径。在线上生产环境中一定要为log.dirs配置多个路径，具体格式是一个 CSV 格式，也就是用逗号分隔的多个路径，比如/home/kafka1,/home/kafka2,/home/kafka3这样。如果有条件的话你最好保证这些目录挂载到不同的物理磁盘上。
* listeners：指定外部连接者要通过什么协议访问指定主机名和端口开放的 Kafka 服务
```properties
  broker.id=0
  log.dirs=./../../kafka-logs
  listeners=PLAINTEXT://:9092

  broker.id=0
  log.dirs=./../../kafka-logs
  listeners=PLAINTEXT://:9092

  broker.id=0
  log.dirs=./../../kafka-logs
  listeners=PLAINTEXT://:9092
```
4. 偷懒，写个脚本启动zookeeper、kafka。注意先启动zookeeper、再启动Kafka即可。

```bat
  @echo off
  cd %cd%\bin\windows
  @echo start zookeeper
  call zookeeper-server-start.bat ./../../config/zookeeper.properties
  @pause

  @echo off
  cd %cd%\bin\windows
  @echo start kafka
  call kafka-server-start.bat ./../../config/server.properties
  @pause
```
5. 使用Kafka tool连接上刚搭建起来的集群
![](/medias/assets/20210409092648.png)
6. 尝试添加一个test主题，包含2个分区、2个副本
![](/medias/assets/20210409092811.png)
![](/medias/assets/20210409092936.png)
![](/medias/assets/20210409093023.png)
6. 尝试发送2条消息，后查看分区信息、消息信息
![](/medias/assets/20210409093512.png)
![](/medias/assets/20210409093921.png)
![](/medias/assets/20210409093942.png)
![](/medias/assets/20210409094010.png)

## 小结
Kafka集群安装特别简单，虽然可配置的参数有几百个，但是只需要配置三个参数就能运行起来了，非常方便。Windows上的安装仅仅用来学习和试验，千万别用在生产环境中哦！
