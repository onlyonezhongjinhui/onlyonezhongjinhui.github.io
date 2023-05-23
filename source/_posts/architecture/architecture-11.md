---
title: 从零学架构（十一）
date: 2021-07-16 11:30:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 实战（三）：微信红包架构设计
tags: 架构
categories:
- [架构]
---
## 微信红包架构设计

### 面向复杂度架构设计步骤

![](/medias/assets/architecture/架构设计步骤.png)

### 复杂度分析

![](/medias/assets/architecture/20210716104331.png)

**红包业务业务复杂度不高，质量复杂度要求高**

### 分析质量复杂度

![](/medias/assets/architecture/20210716104514.png)

从描述可看出红包业务性能要求高

![](/medias/assets/architecture/20210716104613.png)

高性能指标估算

### 高性能架构设计套路

![](/medias/assets/architecture/20210716104902.png)

### 设计方案

#### 发红包

![](/medias/assets/architecture/20210716105007.png)

![](/medias/assets/architecture/20210716105103.png)

#### 抢红包

![](/medias/assets/architecture/20210716105248.png)

![](/medias/assets/architecture/20210716105319.png)

#### 看红包

![](/medias/assets/architecture/20210716105430.png)

![](/medias/assets/architecture/20210716105453.png)

### 整体架构

![](/medias/assets/architecture/20210716105539.png)

![](/medias/assets/architecture/20210716105646.png)

### 成本因素影响架构方案

去掉Redis Cluster，全部用关系型数据库存储

![](/medias/assets/architecture/20210716105857.png)

### 方案优化

无锁化提升处理性能

![](/medias/assets/architecture/20210716110056.png)
