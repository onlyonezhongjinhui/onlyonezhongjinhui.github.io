---
title: 从零学架构（三十）
date: 2021-08-19 22:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 微服务拆分技巧
tags: 架构
categories:
- [架构]
---

## 微服务拆分技巧

### 微服务架构整体思路

![](/medias/assets/architecture/20210819195711.png)

### 常见场景实施建议

![](/medias/assets/architecture/20210819195911.png)

### DDD

#### 介绍

![](/medias/assets/architecture/20210819202247.png)

#### DDD难以落地核心问题

![](/medias/assets/architecture/20210819202356.png)

#### 业务边界划分

![](/medias/assets/architecture/20210819202439.png)

![](/medias/assets/architecture/20210819202514.png)

### 按照业务拆分微服务

![](/medias/assets/architecture/20210819202605.png)

#### 拆分技巧

![](/medias/assets/architecture/20210819202659.png)

##### 三个火枪手原则

![](/medias/assets/architecture/20210819203155.png)

举例

![](/medias/assets/architecture/20210819203351.png)

##### 一对一服务映射

![](/medias/assets/architecture/20210819203502.png)

##### 多对一服务映射

![](/medias/assets/architecture/20210819203631.png)

##### 一对多服务映射

![](/medias/assets/architecture/20210819203753.png)

### 按照质量拆分微服务

#### 按性能拆分

![](/medias/assets/architecture/20210819203915.png)

#### 按业务重要程度拆分

![](/medias/assets/architecture/20210819204017.png)

#### 按可用性拆分

![](/medias/assets/architecture/20210819204104.png)

#### 按稳定性拆分

![](/medias/assets/architecture/20210819204152.png)

## 小结

![](/medias/assets/architecture/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%8B%86%E5%88%86%E6%8A%80%E5%B7%A7.png)
