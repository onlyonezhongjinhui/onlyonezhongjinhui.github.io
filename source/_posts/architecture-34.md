---
title: 从零学架构（三十四）
date: 2021-09-14 22:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 业务级灾备架构设计
tags: 架构
categories:
- [架构]
---

## 业务级灾备架构设计

### 同城多中心

#### 同城双中心

![](/medias/assets/20210914211351.png)

![](/medias/assets/20210914211508.png)

![](/medias/assets/20210914211732.png)

#### 同城三中心

![](/medias/assets/20210914211827.png)

**事实上很少采用这种架构，原因是成本很高，可用性没增加**

### 跨城多中心

#### 跨城双中心

![](/medias/assets/20210914212052.png)

#### 跨城双中心——近邻城市

![](/medias/assets/20210914212228.png)

#### 跨城双中心——远端城市

![](/medias/assets/20210914212524.png)

#### 跨城多中心

![](/medias/assets/20210914212611.png)

### 跨国数据中心

![](/medias/assets/20210914212745.png)

### 业务灾备架构对比

![](/medias/assets/20210914213055.png)

## 小结

![](/medias/assets/%E4%B8%9A%E5%8A%A1%E7%BA%A7%E7%81%BE%E5%A4%87%E6%9E%B6%E6%9E%84%E8%AE%BE%E8%AE%A1.png)