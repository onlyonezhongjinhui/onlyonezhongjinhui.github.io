---
title: 【从零学Netty】Netty自带注解
date: 2023-05-25 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Sharable注解比较常用
tags: Netty
categories:
- [Netty]
---

### Netty自带注解

#### @Sharable

​		标识Handler可共享。没有标识可共享的Handler不能重复加入pipeline，否则报错。

#### @Skip

​		标识Handler上的方法，表示pipeline将不执行该方法。当需要某个Handler上的某个方法不执行的时候，可加上该注解。但是因为@Skip

所在的类ChannelHandlerMask被标记为了final，包外不能使用该注解。

#### @UnstableApi

​		标识API是不稳定的，随时有可能修改。这样的API要慎重使用。不过该注解不太建议使用，如果有问题，可以删掉无用的类，避免引起不必要的麻烦。

#### @SuppressJava6Requirement

​		去除“Java6需求”的报警 。比如项目想在jdk6上运行。不过现在Java版本已经更新到20了，用不到了。

#### @SuppressForbidden

​		去除 “禁用” 报警

#### 总结

​		@Sharable注解比较常用