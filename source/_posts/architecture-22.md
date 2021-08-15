---
title: 从零学架构（二十二）
date: 2021-08-15 16:30:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 计算架构模式之多级缓存架构
tags: 架构
categories:
- [架构]
---

## 计算架构模式之多级缓存架构

### 概念介绍

#### 缓存

##### 定义

Cache。指位于速度相差较大的两种硬件之间，用于协调两者数据传输速度差异的结构，均可称为缓存。

##### 本质

空间换时间

##### 举例

1. CPU L1、L2、L3缓存
2. Linux文件系统page cache
3. innodb buffer pool
4. redis、memcached

#### 缓冲

##### 定义

Buffer。指某个临时存储区域，保存将要从一个设备（或者系统）传输到另一个设备（或者系统）的数据。

##### 举例

1. Java IO BufferdInputStream等
2. 磁盘控制器写缓存（Write cache）
3. MySQL log buffer
4. 消息队列缓冲写请求
5. innodb buffer pool

### 缓存设计

#### 3W1H

![](/medias/assets/20210815181903.png)

#### 更新机制

![](/medias/assets/20210815182207.png)

### 多级缓存架构

#### 5级缓存架构

![](/medias/assets/20210815182458.png)

#### 4级缓存架构

![](/medias/assets/20210815182651.png)

#### 3级缓存架构

![](/medias/assets/20210815182738.png)

### 缓存技术

#### 本地缓存

##### APP缓存

###### 定义

App将数据缓存在本地

###### 应用场景

所有能缓存的都可以缓存

###### 常见技术

1. SQLite缓存
2. 本地文件缓存
3. 图片缓存Picasso(Square)、Fresco(Facebook)、Glide(Google)

##### HTTP缓存

###### 定义

HTTP标准协议缓存

###### 应用场景

HTTP资源

###### 常见技术

1. 参考HTTP协议、Cache-Control、Etag/If-None-Match等指令

#### CDN缓存

![](/medias/assets/20210815190434.png)

##### 定义

Content Delivery NetWork ，即内容分发网络，依靠部署在各地的边缘服务器，通过中心平台的负载均衡、内容分发、调度等功能模块，使用户就近获取所需的内容，降低网络拥塞，提高用户访问响应速度和命中率，关键技术是内容存储和分发技术

##### 优缺点

1. 功能强大，能够支撑超高流量
2. 贵

##### 应用场景

1. 直播（RTMP、HLS）
2. 视频
3. 资讯

##### 国内供应商

阿里云、网宿、腾讯云、金山云、七牛云......

#### Web容器缓存

Web容器一般缓存静态资源，例如图片、Javascript、CSS等，配合HTTP协议实现缓存

#### 应用缓存

##### 定义

应用在本地缓存数据

##### 应用场景

所有能缓存的都可以缓存

##### 常见技术

1. 进程内缓存、HashMap、OSCache、Ehcache等
2. 进程外缓存，堆外内存
3. 本地磁盘SSD缓存

#### 分布式缓存

##### 定义

由分布式系统提供缓存功能

##### 应用场景

所有能缓存的都可以缓存

##### 具体实现

1. Redis
2. Memcached

![](/medias/assets/20210815193759.png)

## 小结

![](/medias/assets/%E8%AE%A1%E7%AE%97%E6%9E%B6%E6%9E%84%E6%A8%A1%E5%BC%8F%E4%B9%8B%E5%A4%9A%E7%BA%A7%E7%BC%93%E5%AD%98%E6%9E%B6%E6%9E%84.png)