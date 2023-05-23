---
title: 从零学架构（七）
date: 2021-07-10 15:30:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 如何设计可扩展架构
tags: 架构
categories:
- [架构]
---
## 如何设计可扩展架构

### 复杂度模型

![](/medias/assets/architecture/20210710150101.png)

#### 业务复杂度

业务固有的复杂度，主要体现为难以理解，难以扩展，例如业务数量多（微信），业务流程长（支付宝），业务之间关系复杂（例如ERP）

#### 质量复杂度

高性能、高可用、成本、安全等质量属性要求

### 架构复杂度应对之道

![](/medias/assets/architecture/20210710150858.png)

![](/medias/assets/architecture/架构设计步骤.png)

### 可扩展（extensibility）

系统适应变化的能力，包含可理解和可复用两个部分

### 可伸缩（scalability）

系统通过添加更多的资源来提升性能的能力

### 可扩展复杂度模型

![](/medias/assets/architecture/20210710152734.png)

### 可扩展架构设计

#### 拆分

##### 拆分复杂度模型

![](/medias/assets/architecture/20210710153155.png)

##### 拆分粒度

###### 内部复杂度

又称为“单体复杂度”，指单个对象内部的复杂度，例如传统的单体系统，所以业务都在一个系统里面。可以用参与的开发人数来衡量单个拆分对象的复杂度。例如：3个人负责一个子系统/子模块是比较合理的，20个人来在同一个子系统上开发，则内部复杂度过高

###### 外部复杂度

又称为“群体复杂度”，指拆分后的多个对象之间的关系复杂度。可以用业务流程涉及对象数量来衡量外部复杂度。例如：一次用户请求需要5个子系统参与比较合理的，如果需要20个子系统参与，则外部复杂度过高。

##### 拆分原则

1. 内外平衡原则
2. 先粗后细原则

#### 封装

##### 封装复杂度模型

![](/medias/assets/architecture/20210710154359.png)

##### 预测原则

1. 2年原则

只预测2年内的可能变化，不要试图预测10年后的变化。例如你准备接入微信支付，那么预测接入支付宝是很自然的，但数字钱包就没那么必要了

1. 3次法则

预测没有把握就不要封装，等到需要的时候重构即可。例如要不要支持数据库为 Oracle、MySQL、PostgreSQL。(Martin Fowler)Rule of three: Three strikes and you refactor （1写2抄3封装）。

##### 封装技巧

![](/medias/assets/architecture/20210710155106.png)

## 小结

![](/medias/assets/architecture/可扩展架构设计.png)
