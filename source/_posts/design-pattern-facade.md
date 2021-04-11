---
title: SPI和门面模式有什么区别
date: 2021-04-11 18:00:00
author: maybe
top: true
cover: true
toc: true
mathjax: true
summary:
tags: Design Pattern
categories:
- [Design Pattern]
---

## SPI和门面模式有什么区别

要回答这个问题，就先得明白什么是SPI和门面模式。

### SPI

SPI，全称Service Provider Interface，是Java提供的一种服务加载方式。它是一种将服务接口和服务实现分离达到解耦，大大提升了程序的可扩展性的机制。引入服务提供者就是引入了SPI接口的实现者，通过本地的注册发现获取具体的实现类，轻松可插拔。

要实现SPI服务注册发现加载，需要做以下几件事情

1. 在 META-INF/services/ 目录中创建以接口全限定名命名的文件，该文件内容为API具体实现类的全限定名
2. 使用 ServiceLoader 类动态加载 META-INF 中的实现类
3. 如 SPI 的实现类为 Jar 则需要放在主程序 ClassPath 中
4. API 具体实现类必须有一个不带参数的构造方法

#### 举个栗子

要实现一个文件上传的功能，文件可以是普通文本文件、视频、图片等。Java都是面向接口编程的，那么就先定义一个上传接口

```java
public interface Uploader {

    /**
     * upload.
     */
    void upload();

}

```

现在我们要实现普通文本文件、视频、图片的上传。假如它们上传的时候需要做不同的操作，所以就分别定义了三个类，都实现Uploader接口

```java
public class FileUploader implements Uploader {

    @Override
    public void upload() {
        System.out.println("this is file uploader");
    }

}


public class ImageUploader implements Uploader {
    @Override
    public void upload() {
        System.out.println("this image uploader");
    }
}


public class VideoUploader implements Uploader {

    @Override
    public void upload() {
        System.out.println("this is video uploader");
    }

}
```

然后在META-INF/services目录下创建SPI的描述文件,文件名为Uploader全限定名io.zjh.Uploader，内容为实现类的全限定名

```
io.zjh.FileUploader
io.zjh.ImageUploader
io.zjh.VideoUploader
```

好了，现在就可以通过SPI机制去加载实现了Uploader接口的服务了

```java
public class Main {
    public static void main(String[] args) {
        ServiceLoader<Uploader> uploaders = ServiceLoader.load(Uploader.class);
        uploaders.forEach(Uploader::upload);
    }
}
```

运行这个类后能看到输出如下

```
this is file uploader
this image uploader
this is video uploader
```

假如现在要对视频增加更强的功能，这个时候只需要在自己的工程里面增加一个实现类VideoUploader进行增强，然后在自己的工程META-INF/services/ 目录下创建一个名为io.zjh.Uploader内容为io.zjh.VideoUploader的SPI描述文件即可。

再次加载输出如下

```
this is file uploader
this image uploader
this is a strong Video uploader
```

这就是SPI的威力。

**顺便提一下API和SPI的区别。API，Application Programming Interface，是实现方制定接口并完成对接口的实现，调用方无权选择不同的实现。而SPI是调用方来制定接口，提供给外部来实现，调用方在调用时选择自己需要的外部实现。简单的说就是API是给开发人员使用的，而SPI是给框架使用的。**

### 门面模式

门面模式，也叫外观模式，英文全称是 Facade Design Pattern。在 GoF 的《设计模式》一书中，门面模式是这样定义的：Provide a unified interface to a set of interfaces in a subsystem. Facade Pattern defines a higher-level interface that makes the subsystem easier to use.翻译成中文就是：门面模式为子系统提供一组统一的接口，定义一组高层接口让子系统更易用。

举个例子：

假设有一个系统 A，提供了 a、b、c、d 四个接口。系统 B 完成某个业务功能，需要调用 A 系统的 a、b、d 接口。利用门面模式，提供一个包裹 a、b、d 接口调用的门面接口 x，给系统 B 直接使用，易用性就上去了。

#### 使用场景

**解决易用性问题**

门面模式可以用来封装系统的底层实现，隐藏系统的复杂性，提供一组更加简单易用、更高层的接口。比如上面的例子就是典型的一个例子。再比如Kafka提供的脚本工具就可以看作是门面模式的经典用法，通过这些门面，隐藏了Kafka服务调用的复杂性，可以通过这些简单的命令去操作Kafka。

**解决性能问题**

假设有个业务，需要调用a、b、c、d、e这5个网络接口完成，如果这个时候做个f门面接口，来在后端调用这几个接口的业务完成处理，就可以减少网络请求，提升性能。

**解决分布式事务问题**

假设有这样一个业务场景：在用户注册的时候，我们不仅会创建用户（在数据库 User 表中），还会给用户创建一个钱包（在数据库的 Wallet 表中）。对于这样一个简单的业务需求，我们可以通过依次调用用户的创建接口和钱包的创建接口来完成。但是，用户注册需要支持事务，也就是说，创建用户和钱包的两个操作，要么都成功，要么都失败，不能一个成功、一个失败。要支持两个接口调用在一个事务中执行，是比较难实现的，这涉及分布式事务问题。虽然我们可以通过引入分布式事务框架或者事后补偿的机制来解决，但代码实现都比较复杂。而最简单的解决方案是，利用数据库事务或者 Spring 框架提供的事务（如果是 Java 语言的话），在一个事务中，执行创建用户和创建钱包这两个 SQL 操作。这就要求两个 SQL 操作要在一个接口中完成，所以，我们可以借鉴门面模式的思想，再设计一个包裹这两个操作的新接口，让新接口在一个事务中执行两个 SQL 操作。

好了，明白了SPI和门面模式，来回答开篇的问题，“SPI和门面模式有什么区别”。

**门面模式是接口设计的指导思想，旨在为子系统提供一组统一的接口，定义一组高层接口让子系统更易用。典型使用场景是提升接口易用性，解决性能问题，解决分布式事务问题。SPI则是一种将服务接口和服务实现分离达到解耦，大大提升了程序的可扩展性的机制。**

## 小结

1. SPI则是一种将服务接口和服务实现分离达到解耦，大大提升了程序的可扩展性的机制
2. 门面模式是接口设计的指导思想，旨在为子系统提供一组统一的接口，定义一组高层接口让子系统更易用。