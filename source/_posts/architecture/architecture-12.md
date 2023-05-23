---
title: 从零学架构（十二）
date: 2021-07-16 12:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 实战（四）：香港支付宝钱包高可用架构设计
tags: 架构
categories:
- [架构]
---
## 香港支付宝钱包高可用架构设计

### 容忍度

#### 定义

用户能够接受业务不可用程度，包括时长和影响

#### 特点

不同文化、不同法律、不同用户、不同业务，容忍度差异很大

#### 案例

1. 香港的银行对接 AlipayHK，晚上出故障，要第二天9点上班才开始处理
2. 内部运营系统能够接受不可用时长可以达到2小时
3. 支付业务能够容忍的时间是分钟级
4. 游戏业务可以停服更新

### 容忍度排序

生命->安全->金钱->付费->免费->内部

### 业务背景

![](/medias/assets/architecture/20210716111445.png)

### 面向复杂度架构设计步骤

![](/medias/assets/architecture/架构设计步骤.png)

### 复杂度分析

![](/medias/assets/architecture/20210716104331.png)

**钱包业务业务复杂度不高，质量复杂度要求高**

### 分析质量复杂度

![](/medias/assets/architecture/20210716115006.png)

### 高可用架构设计套路

![](/medias/assets/architecture/20210716115353.png)

#### 余额转账

![](/medias/assets/architecture/20210716115509.png)

![](/medias/assets/architecture/20210716115713.png)

#### 银行卡支付

![](/medias/assets/architecture/20210716115847.png)

![](/medias/assets/architecture/20210716115933.png)

虽然一致性要求不高，但是都上了OceanBase，那就共用好了，保持架构一致

#### 运营后台

![](/medias/assets/architecture/20210716120103.png)

![](/medias/assets/architecture/20210716120128.png)

### 整体架构

![](/medias/assets/architecture/20210716120209.png)

![](/medias/assets/architecture/20210716120243.png)
