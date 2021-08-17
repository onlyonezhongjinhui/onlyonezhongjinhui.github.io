---
title: 从零学架构（二十八）
date: 2021-08-17 22:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 微服务架构陷阱与挑战
tags: 架构
categories:
- [架构]
---

## 微服务架构陷阱与挑战

### 六大陷阱

![](/medias/assets/20210817211724.png)

#### 粒度太细，关系复杂

![](/medias/assets/20210817211907.png)

#### 粒度太细，团队效率下降

![](/medias/assets/20210817212026.png)

#### 粒度太细，问题定位困难

![](/medias/assets/20210817212118.png)

#### 粒度太细，系统性能下降

![](/medias/assets/20210817212234.png)

#### 缺乏基础设施，无法快速交付

![](/medias/assets/20210817212327.png)

#### 缺乏基础设施，服务管理混乱

![](/medias/assets/20210817212555.png)

### 四大挑战

![](/medias/assets/20210817212721.png)

#### 分布式事务

BASE理论之最终一致性

![](/medias/assets/20210817213011.png)

##### 本地事务消息

![](/medias/assets/20210817213105.png)

#### 消息队列事务消息

![](/medias/assets/20210817213205.png)

##### TCC

![](/medias/assets/20210817213249.png)

#### 全局幂等

![](/medias/assets/20210817213420.png)

##### 样例1

![](/medias/assets/20210817213534.png)

##### 样例2

正常处理

![](/medias/assets/20210817213619.png)

异常处理

![](/medias/assets/20210817213837.png)

#### 接口兼容

![](/medias/assets/20210817214041.png)

#### 接口循环调用

![](/medias/assets/20210817214107.png)

## 小结

![](/medias/assets/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84%E9%99%B7%E9%98%B1%E5%92%8C%E6%8C%91%E6%88%98.png)