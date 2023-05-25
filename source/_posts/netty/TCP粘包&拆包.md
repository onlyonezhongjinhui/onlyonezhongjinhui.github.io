---
title: 【从零学Netty】TCP粘包&拆包
date: 2023-05-02 10:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: TCP是流式协议，存在粘包&拆包问题
tags: Netty
categories:
- [Netty]
---

### TCP粘包&拆包

#### 粘包&拆包

![](/medias/assets/netty/package.jpg)

#### 粘包的原因

1. 发送方每次写入的数据小于套接字缓冲区的大小
2. 接收方读取套接字缓冲区数据不够及时

#### 拆包的原因

1. 发送方每次写入的数据大于套接字缓冲区的大小
2. 发送的数据大于协议的最大传输单元（Maximum Transmission Unit），则必须拆包

#### TCP为什么会出现粘包&拆包

​	TCP是流式协议，消息没有边界

#### 粘包&拆包的解决方法

<table>
    <tr>
        <th>解决方法</th>
        <th>寻找消息边界方式</th>
        <th>优点</th>
        <th>缺点</th>
        <th>推荐度</th>
    </tr>
    <tr>
        <td>一个请求一个TCP连接</td>
        <td> 建立连接到释放连接之间的消息即为完整的消息 </td>
        <td> 简单 </td>
        <td> 效率低下 </td>
        <td> 不推荐 </td>
    </tr>
    <tr>
        <td>固定长度</td>
        <td> 满足固定长度即可 </td>
        <td> 简单 </td>
        <td> 浪费空间 </td>
        <td> 不推荐 </td>
    </tr>
    <tr>
        <td>分隔符</td>
        <td> 分隔符之间即为完整消息 </td>
        <td> 简单、不浪费空间 </td>
        <td> 内容本身包含分隔符时需要转义，需要内容扫描 </td>
        <td> 推荐 </td>
    </tr>
    <tr>
        <td>固定长度字段存储内容的长度信息</td>
        <td> 先读取固定长度字段获得内容长度，然后根据内容长度读取后续内容 </td>
        <td> 精确定位用户数据，内容也不用转义 </td>
        <td> 长度理论上有限制，需要提前预知内容的最大长度从而定义长度占用的字节数</td>
        <td> 推荐+ </td>
    </tr>
    <tr>
        <td>其他</td>
        <td> 每种方式都不同，例如JSON可以判断{}是否成对作为判断 </td>
        <td colspan="3"> 需要根据实际场景衡量 </td>
    </tr>
</table>

#### Netty对粘包&拆包的支持

| 方法                           | 解码                         | 编码                 |
| ------------------------------ | ---------------------------- | -------------------- |
| 固定长度                       | FixedLengthFrameDecoder      | 简单                 |
| 分隔符                         | DelimiterBasedFrameDecoder   | 简单                 |
| 固定长度字段存储内容的长度信息 | LengthFieldBasedFrameDecoder | LengthFieldPrepender |

Netty除了这三个基本方式的支持，对于常见协议也都有相应的支持，非常丰富，开箱即用

#### 总结

1. TCP是流式协议，存在粘包&拆包问题
2. Netty对粘包&拆包有丰富易用的支持，开箱即用
