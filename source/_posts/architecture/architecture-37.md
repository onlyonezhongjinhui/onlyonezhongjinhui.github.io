---
title: 从零学架构（三十七）
date: 2021-09-17 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 网络模型
tags: 架构
categories:
- [架构]
---
## 网络模型

### 传统网络模型

#### PPC、prefork

![](/medias/assets/architecture/20210917091938.png)

##### 优点

实现简单

##### 缺点

1. PPC：fork代价高，性能低
2. 父子进程通信要用IPC，监控统计等实现复杂
3. OS的上下文切换会限制并发连接数，一般几百

#### TPC、prethread

![](/medias/assets/architecture/20210917092206.png)

##### 优点

1. 实现简单
2. 无需IPC，线程中通信即可
3. 无需fork，线程创建代价相对创建进程低

##### 缺点

1. 线程互斥和共享比PPC、prefork复杂
2. 某个线程故障可能导致整个进程退出
3. OS的上下文切换会限制并发连接数，一般几百，比PPC、prefork高

### Reactor网络模型

Reactor：Reactor基于多路复用的事件响应网络编程模型

多路复用：多个连接复用同一个阻塞对象，例如Java的Selector、epoll的epoll_fd（epoll_create函数创建）

事件响应：不同事件分发给不同对象处理，Java的事件有OP_ACCEPT、OP_CONNECT、OP_READ、OP_WRITE

缺点：实现比传统的网络模型复杂

优点：支持海量连接

#### 模式一：单Reactor单进程/单线程

![](/medias/assets/architecture/20210917093132.png)

#### 模式二：单Reactor多线程

![](/medias/assets/architecture/20210917093258.png)

#### 模式三：多Reactor多进程/线程

![](/medias/assets/architecture/20210917093428.png)

### Proactor网络模型

![](/medias/assets/architecture/20210917095102.png)

### 网络模型对比

![](/medias/assets/architecture/20210917095251.png)

### 三类网络模型实战技巧

![](/medias/assets/architecture/20210917095403.png)

## 小结

![](/medias/assets/architecture/网络模型.png)
