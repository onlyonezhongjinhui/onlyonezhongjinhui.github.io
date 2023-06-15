---
title: Kafka监控工具
date: 2021-04-08 09:29:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 大公司定制开发，小公司一般采用现成的工具
tags: Kafka
categories:
- [MQ]
---

说监控工具之前，先看看Kafka都有哪些“发行版本”(和Linux发行版本一样的概念)。

## "Kafka发行版本"
#### Apache Kafka
Apache Kafka 是Apache 基金会顶级开源项目。社区版 Kafka优势在于迭代速度快，社区响应度高，使用它可以让你有更高的把控度；缺陷在于仅提供基础核心组件，缺失一些高级的特性。
#### Confluent Kafka
Confluent公司发布的Confluent Kafka。Confluent Kafka 提供了一些 Apache Kafka 没有的高级特性，比如跨数据中心备份、Schema 注册中心以及集群监控工具等。优势在于集成了很多高级特性且由 Kafka 原班人马打造，质量上有保证；缺陷在于相关文档资料不全，普及率较低，没有太多可供参考的范例。
#### Cloudera/Hortonworks Kafka
Cloudera 提供的 CDH 和 Hortonworks 提供的 HDP 是非常著名的大数据平台，里面集成了目前主流的大数据框架，能够帮助用户实现从分布式存储、集群调度、流处理到机器学习、实时数据库等全方位的数据处理。优势在于操作简单，节省运维成本；缺陷在于把控度低，演进速度较慢。

由于社区版本并未提供监控工具，所以监控就是一个比较麻烦的问题，大公司一般自己定制开发，创业公司或者小公司一般采用现成的监控工具，我就罗列几种监控工具，具体使用还得根据自己的实际情况进行选择。
## Kafka监控工具
* Logi-KafkaManager
* JMXTrans + InfluxDB + Grafana
* kafka-eagle
* Kafka tool
* Kafka Manager
* Kafka Monitor
