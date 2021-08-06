---
title: 从零学架构（十八）
date: 2021-08-05 15:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: 存储架构模式之复制架构
tags: 架构
categories:
- [架构]
---
## 存储架构模式之复制架构

### 存储类问题处理框架图

![](/medias/assets/20210805145233.png)

### 高可用存储核心指标

![](/medias/assets/20210805145704.png)

**高可用计算不涉及RPO**

### 主备&主从架构

#### 主备&主从复制

![](/medias/assets/20210805150124.png)

#### 主备级联复制

![](/medias/assets/20210805150251.png)

#### 主备架构的灾备部署

![](/medias/assets/20210805150444.png)

#### 主从架构的灾备部署

![](/medias/assets/20210805150534.png)

#### 双机切换架构——主备切换

![](/medias/assets/20210805151552.png)

#### 双机切换架构——主从切换

![](/medias/assets/20210805151832.png)

#### 集群选举

![](/medias/assets/20210805152239.png)

案例

![](/medias/assets/20210805153058.png)

![](/medias/assets/20210805160507.png)

## 小结

![](/medias/assets/存储架构模式之复制架构.png)