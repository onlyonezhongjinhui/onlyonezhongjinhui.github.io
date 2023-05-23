---
title: 从零学架构（二十九）
date: 2021-08-18 22:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 微服务架构基础设施选型
tags: 架构
categories:
- [架构]
---

## 微服务架基础设施选型

### 微服务基础设施

![](/medias/assets/architecture/20210115205733.png)

**核心设施：服务注册、服务发现、服务路由**

### 微服务框架模式

#### 嵌入SDK式

![](/medias/assets/architecture/20210818211028.png)

##### 举例

![](/medias/assets/architecture/20210818211533.png)

![](/medias/assets/architecture/20210818211703.png)

#### 反向代理式

![](/medias/assets/architecture/20210818211150.png)

##### 举例

![](/medias/assets/architecture/20210818211825.png)

#### 网络代理式（Service Mesh）

![](/medias/assets/architecture/20210818211300.png)

##### 举例

![](/medias/assets/architecture/20210818211910.png)

#### 对比

![](/medias/assets/architecture/20210818211332.png)

### 常见微服务框架

#### Dubbo

![](/medias/assets/architecture/20210818211533.png)

#### Spring Cloud

![](/medias/assets/architecture/20210818211703.png)

### 如何选择开源微服务框架

![](/medias/assets/architecture/20210818212057.png)

**遇事不决Spring，选择太多Apache**

## 小结

![](/medias/assets/architecture/%E5%BE%AE%E6%9C%8D%E5%8A%A1%E5%9F%BA%E7%A1%80%E8%AE%BE%E6%96%BD.png)
