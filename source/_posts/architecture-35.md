---
title: 从零学架构（三十五）
date: 2021-09-15 17:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 异地多活架构的三种模式
tags: 架构
categories:
- [架构]
---
## 异地多活架构的三种架构模式

### 业务定制型异地多活

![](/medias/assets/20210915153742.png)

### 业务通用型异地多活

![](/medias/assets/20210915154859.png)

案例1：淘宝单元化机构

![](/medias/assets/20210915160317.png)

案例2：蚂蚁LDC架构

![](/medias/assets/20210915161736.png)

LDC路由：

![](/medias/assets/20210915162013.png)

LDC容灾：

![](/medias/assets/20210915162126.png)

LDC蓝绿发布：

![](/medias/assets/20210915162703.png)

核心——配套服务

![](/medias/assets/20210915162830.png)

### 存储通用型异地多活

![](/medias/assets/20210915163143.png)

案例：OceanBase

![](/medias/assets/20210915163533.png)

### 异地多活架构模式对比

![](/medias/assets/20210915164121.png)

## 小结

![](/medias/assets/异地多活架构的三种模式.png)