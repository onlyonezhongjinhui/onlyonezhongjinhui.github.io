---
title: 从零学架构（八）
date: 2021-07-12 09:30:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 如何设计高性能架构
tags: 架构
categories:
- [架构]
---
## 如何设计高性能架构

### 高性能架构复杂度模型

![](/medias/assets/architecture/20210712085012.png)

### 高性能架构复杂度模型分析

![](/medias/assets/architecture/20210712085101.png)

### 集群高性能架构设计

#### 指导理论

鸡蛋篮子理论第二法则——叠加法则

![](/medias/assets/architecture/20210712085546.png)

#### 任务分配

##### 定义

将任务分配给多个服务器执行

##### 复杂度分析

1. 增加“任务分配器”节点，可以是独立的服务器也可以是SDK
2. 任务分配器需要管理所有服务器，可以通过配置文件，也可以通过配置服务器，例如Zookeeper

3. 任务分配器需要根据不同的需求采用不同的算法分配

![](/medias/assets/architecture/20210712091927.png)

![](/medias/assets/architecture/20210712092039.png)

##### 关键点

![](/medias/assets/architecture/20210712092223.png)

##### 案例

![](/medias/assets/architecture/20210712092705.png)

![](/medias/assets/architecture/20210712092812.png)

![](/medias/assets/architecture/20210712092906.png)

#### 任务分解

##### 定义

将服务器拆分为不同的角色，不同服务器处理不同业务

##### 复杂度分析

1. 增加“任务分解器”节点，可以是独立的服务器也可以是SDK
2. 任务分解器需要管理所有服务器，可以通过配置文件，也可以通过配置服务器，例如Zookeeper
3. 需要设计拆分任务的方式，任务分解器需要记录“任务”和“服务器”的映射关系
4. 任务分解器需要根据不同需求采用不同的算法分配

![](/medias/assets/architecture/20210712093533.png)

##### 关键点

![](/medias/assets/architecture/20210712093637.png)

##### 案例

![](/medias/assets/architecture/20210712094216.png)

![](/medias/assets/architecture/20210712094256.png)

![](/medias/assets/architecture/20210712094359.png)

## 小结

![](/medias/assets/architecture/高性能架构设计.png)
