---
title: 从零学架构（二十四）
date: 2021-08-16 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 计算架构模式之负载均衡架构
tags: 架构
categories:
- [架构]
---
## 计算架构模式之负载均衡架构

### 多级负载均衡架构

#### 4级负载均衡架构

![](/medias/assets/architecture/20210816084015.png)

#### 3级负载均衡架构

![](/medias/assets/architecture/20210816084148.png)

#### 2级负载均衡架构

![](/medias/assets/architecture/20210816084241.png)

### 负载均衡技术

#### DNS

![](/medias/assets/architecture/20210816084632.png)

##### 应用

地理位置和机房级别的负载均衡

##### 优点

标准协议

##### 缺点

1. 能力有限，不够灵活
2. DNS劫持
3. DNS缓存

#### HTTP-DNS

![](/medias/assets/architecture/20210816084807.png)

##### 应用

App、客户端

##### 优缺点

1. 可以根据业务和团队技术灵活控制
2. 非标准协议，不通用，不太适合Web业务

##### 架构设计关键点

1. 智能调度模块可以独立也可以嵌入到HTTP-DNS，一般独立成运维系统，因为智能调度系统有很多作用
2. 正常走DNS，异常的时候才走HTTP-DNS
3. SDK会缓存HTTP-DNS解析结果

#### GSLB

##### 定义

GSLB（Global Server Load Balancing）全局负载均衡，主要用于在多地区拥有自己的服务器的节点，为了使全球用户只以一个IP地址或域名就能访问到离自己最近的服务器，从而获得最快的访问速度

##### 适用场景

适合超大规模业务，多地甚至全球部署的业务，例如Google、Facebook等

##### 优缺点

1. 功能强大，可以就近访问、容灾切换、流量调节
2. 实现复杂

#### 基于DNS的GSLB

![](/medias/assets/architecture/20210816092949.png)

##### 优缺点

1. 实现简单、实施容易、成本低
2. 可能判断不准确，例如用户手工指定了DNS服务器

#### 基于HTTP redirect的GSLB

![](/medias/assets/architecture/20210816100759.png)

##### 优缺点

1. 能够拿到用户真实ip地址，判断准确
2. 只适用HTTP业务

#### 基于IP欺骗的GSLB

![](/medias/assets/architecture/20210816100950.png)

##### 优缺点

1. 适用所有业务
2. 每次都经过GSLB设备，性能低（IP包欺骗需要GSLB接收所有请求，重定向跳转可以缓存，不需要每次都经过GSLB）
3. 一般配合HTTP redirect GSLB一起应用

#### F5

![](/medias/assets/architecture/20210816104108.png)

#### LVS

![](/medias/assets/architecture/20210816104250.png)

#### LVS-NAT

![](/medias/assets/architecture/20210816104451.png)

#### LVS-DR

![](/medias/assets/architecture/20210816104549.png)

#### LVS-TUN

![](/medias/assets/architecture/20210816104726.png)

#### F5/LVS/Nginx对比

![](/medias/assets/architecture/20210816104916.png)

## 小结

![](/medias/assets/architecture/计算架构模式之负载均衡架构.png)
