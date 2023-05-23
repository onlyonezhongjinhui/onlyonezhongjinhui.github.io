---
title: 从零学架构（二十六）
date: 2021-08-17 15:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 计算架构模式之接口高可用
tags: 架构
categories:
- [架构]
---
## 计算架构模式之接口高可用

### 接口高可用整体框架

![](/medias/assets/architecture/20210817115229.png)

### 限流

![](/medias/assets/architecture/20210817115358.png)

#### 实现方式

![](/medias/assets/architecture/20210817115548.png)

#### 限流算法

##### 固定时间窗口&滑动时间窗口

![](/medias/assets/architecture/20210817115827.png)

##### 漏桶算法

![](/medias/assets/architecture/20210817115944.png)

举例

![](/medias/assets/architecture/20210817120043.png)

##### 漏桶算法变种——写缓冲（Buffer）

![](/medias/assets/architecture/20210817120351.png)

##### 令牌桶算法

![](/medias/assets/architecture/20210817120536.png)

### 排队

![](/medias/assets/architecture/20210817120654.png)

![](/medias/assets/architecture/20210817120756.png)

![](/medias/assets/architecture/20210817143701.png)

![](/medias/assets/architecture/20210817143959.png)

### 降级

![](/medias/assets/architecture/20210817144133.png)

![](/medias/assets/architecture/20210817144238.png)

### 熔断

![](/medias/assets/architecture/20210817144332.png)

![](/medias/assets/architecture/20210817144407.png)

## 小结

![](/medias/assets/architecture/计算架构模式之接口高可用.png)
