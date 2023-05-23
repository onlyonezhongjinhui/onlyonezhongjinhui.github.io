---
title: 从零学架构（十七）
date: 2021-08-05 12:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 数据库存储架构
tags: 架构
categories:
- [架构]
---
## 数据库存储架构

### 数据库读写分离

![](/medias/assets/architecture/20210805103555.png)

#### 复杂度

![](/medias/assets/architecture/20210805103751.png)

#### 复制延迟应对之道

![](/medias/assets/architecture/20210805104014.png)

#### 任务分解应对之道

![](/medias/assets/architecture/20210805104220.png)

#### 缺点

* 单机写入性能有限
* 单机存储容量有限

### 数据库分库分表

#### 分库

![](/medias/assets/architecture/20210805105916.png)

#### 分表

![](/medias/assets/architecture/20210805110243.png)

#### 复杂度和应对之道

![](/medias/assets/architecture/20210805110538.png)

#### 伸缩瓶颈

![](/medias/assets/architecture/20210805110714.png)

#### 分布式事务

##### 2PC

![](/medias/assets/architecture/20210805110952.png)

样例

![](/medias/assets/architecture/20210805111504.png)

##### 3PC

![](/medias/assets/architecture/20210805111314.png)

## 小结

![](/medias/assets/architecture/数据库存储架构.png)
