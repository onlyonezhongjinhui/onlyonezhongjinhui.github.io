---
title: 代理模式
date: 2021-08-05 10:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 代理模式
tags: Design Pattern
categories:
- [Design Pattern]
---
## 代理模式

### 定义

在不改变原始类代码情况下，通过引入代理类来给原始类附加功能

### 角色

![](/medias/assets/design-pattern/20210805093311.png)

* Subject：定义 RealSubject 和 Proxy 角色都应该实现的接口
* Real Subject：原始类，真正完成业务服务功能
* Proxy：代理类，将自身的请求用Real Subject对应的功能来实现，代理类对象并不真正的去实现其业务功能

### 应用场景

* 业务系统的非功能性需求开发，比如监控、统计、鉴权、限流、事务、幂等、日志......
* RPC
* 缓存

### 静态代理

#### 缺点

* 必须实现接口
* 每个原始类都要创建一个对应的代理类，类数量爆炸。且每个代理类中的代码有点像模板式代码，增加了维护成本和开发成本
* 接口一旦增加新方法，原始类和代理类都得修改，不符合开闭原则

### 动态代理

#### 原理

动态代理是在运行时动态生成类字节码，并加载到 JVM中，生成Class对象

#### 优点

* 动态代理更加灵活，不需要必须实现接口。既可代理接口也可直接代理原始类
* 不用每个原始类都创建一个对应的代理类，减少了开发和维护成本
* 接口新增方法，代理类不用修改，符合开闭原则

#### 字节码生成框架

* ASM
* CGLIB
* Javassist

#### JDK动态代理

#### 角色

![](/medias/assets/design-pattern/20210805094841.png)

* Subject
* Real Subject
* InvocationHandler：创建一个处理类并实现 InvocationHandler 接口，将原始类注入处理类，重写其 invoke 方法，在 invoke 方法中利用反射机制调用原始类的方法，并自定义一些处理逻辑
* Proxy

#### 缺点

只能代理接口的实现类，且只能代理接口中的方法，原始类的其它方法不能代理

### CGLIB动态代理

#### 角色

![](/medias/assets/design-pattern/20210805100845.png)

* Subject
* Real Subject
* MethodInterceptor：创建一个方法拦截器实现MethodInterceptor接口，并重写 intercept 方法，intercept用于拦截并增强原始类的方法
* Proxy

### 栗子

#### 场景

一个UserMapper接口是提供对用户数据的增删查改功能，现在要统计增删查改的执行次数，典型的代理模式使用场景

#### 共用代码

```java
package io.zjh.entity;

public class User {
    private String id;
    private String name;
    private int age;

    public User(String id, String name, int age) {
        this.id = id;
        this.name = name;
        this.age = age;
    }

    public String getId() {
        return id;
    }

    public void setId(String id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "User{" +
                "name='" + name + '\'' +
                ", age=" + age +
                '}';
    }
}

```

```java
package io.zjh.metric;

import java.util.concurrent.atomic.AtomicInteger;

public class CrudMetric {

    private final static AtomicInteger insertCount = new AtomicInteger(0);
    private final static AtomicInteger updateCount = new AtomicInteger(0);
    private final static AtomicInteger deleteCount = new AtomicInteger(0);
    private final static AtomicInteger selectCount = new AtomicInteger(0);

    public static void increaseInsert() {
        insertCount.incrementAndGet();
    }

    public static void increaseUpdate() {
        updateCount.incrementAndGet();
    }

    public static void increaseDelete() {
        deleteCount.incrementAndGet();
    }

    public static void increaseSelect() {
        selectCount.incrementAndGet();
    }

    public static void metric() {
        System.out.println("----------------metric-------------------");
        System.out.println("insert count : " + insertCount.get());
        System.out.println("update count : " + updateCount.get());
        System.out.println("delete count : " + deleteCount.get());
        System.out.println("select count : " + selectCount.get());
        System.out.println("----------------metric-------------------");
    }

    public static void clear() {
        insertCount.set(0);
        updateCount.set(0);
        deleteCount.set(0);
        selectCount.set(0);
    }

}

```

#### 静态代理方式实现

Subject

```java
package io.zjh;

import io.zjh.entity.User;

import java.util.List;

public interface IUserMapper {

    void insert(User user);

    void update(User user);

    void deleteById(String id);

    User selectById(String id);

    List<User> selectAll();

}

```

Real Subject

```java
package io.zjh;

import io.zjh.entity.User;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class UserMapper implements IUserMapper {

    private final Map<String, User> maps = new ConcurrentHashMap<>();

    @Override
    public void insert(User user) {
        maps.put(user.getId(), user);
        System.out.println("insert : " + user);
    }

    @Override
    public void update(User user) {
        maps.put(user.getId(), user);
        System.out.println("update : " + user);
    }

    @Override
    public void deleteById(String id) {
        maps.remove(id);
        System.out.println("delete id : " + id);
    }

    @Override
    public User selectById(String id) {
        System.out.println("select id : " + id);
        return maps.get(id);
    }

    @Override
    public List<User> selectAll() {
        System.out.println("select all");
        return new ArrayList<>(maps.values());
    }
}

```

Proxy

```java
package io.zjh.staticproxy;

import io.zjh.IUserMapper;
import io.zjh.UserMapper;
import io.zjh.entity.User;
import io.zjh.metric.CrudMetric;

import java.util.List;

public class UserMapperProxy implements IUserMapper {

    private final UserMapper userMapper;

    public UserMapperProxy(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    @Override
    public void insert(User user) {
        CrudMetric.increaseInsert();
        userMapper.insert(user);
    }

    @Override
    public void update(User user) {
        CrudMetric.increaseUpdate();
        userMapper.update(user);
    }

    @Override
    public void deleteById(String id) {
        CrudMetric.increaseDelete();
        userMapper.deleteById(id);
    }

    @Override
    public User selectById(String id) {
        CrudMetric.increaseSelect();
        return userMapper.selectById(id);
    }

    @Override
    public List<User> selectAll() {
        CrudMetric.increaseSelect();
        return userMapper.selectAll();
    }

}

```

test

```java
package io.zjh.staticproxy;

import io.zjh.UserMapper;
import io.zjh.entity.User;
import io.zjh.metric.CrudMetric;

public class Main {
    public static void main(String[] args) {
        UserMapper userMapper = new UserMapper();
        UserMapperProxy userMapperProxy = new UserMapperProxy(userMapper);
        User user = new User("1", "sky", 18);
        userMapperProxy.insert(user);
        user.setName("人皇sky");
        userMapperProxy.update(user);
        userMapperProxy.selectById(user.getId());
        userMapperProxy.selectAll();
        userMapperProxy.deleteById(user.getId());
        CrudMetric.metric();
    }
}

```

执行结果

```text
insert : User{name='sky', age=18}
update : User{name='人皇sky', age=18}
select id : 1
select all
delete id : 1
----------------metric-------------------
insert count : 1
update count : 1
delete count : 1
select count : 2
----------------metric-------------------
```

#### JDK动态代理实现方式

InvocationHandler

```java
package io.zjh.dynamicproxy.jdk;

import io.zjh.metric.CrudMetric;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class MapperInvocationHandler<T> implements InvocationHandler {
    private final T target;

    public MapperInvocationHandler(T target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String name = method.getName();
        if (name.startsWith("insert")) {
            CrudMetric.increaseInsert();
        } else if (name.startsWith("update")) {
            CrudMetric.increaseUpdate();
        } else if (name.startsWith("delete")) {
            CrudMetric.increaseDelete();
        } else if (name.startsWith("select")) {
            CrudMetric.increaseSelect();
        }
        return method.invoke(target, args);
    }

}

```

创建工厂

```java
package io.zjh.dynamicproxy.jdk;

import java.lang.reflect.Proxy;

@SuppressWarnings("all")
public class JdkMapperProxyFactory {

    public static <T> T newInstance(T target) {
        return (T) Proxy.newProxyInstance(target.getClass().getClassLoader(), target.getClass().getInterfaces(), new MapperInvocationHandler(target));
    }

}

```

test

```java
package io.zjh.dynamicproxy.jdk;

import io.zjh.IUserMapper;
import io.zjh.entity.User;
import io.zjh.metric.CrudMetric;
import io.zjh.UserMapper;

public class JDKMain {
    public static void main(String[] args) {
        IUserMapper userMapper = JdkMapperProxyFactory.newInstance(new UserMapper());
        User user = new User("1", "sky", 18);
        userMapper.insert(user);
        user.setName("人皇sky");
        userMapper.update(user);
        userMapper.selectById(user.getId());
        userMapper.selectAll();
        userMapper.deleteById(user.getId());
        CrudMetric.metric();
    }
}

```

执行结果

```text
insert : User{name='sky', age=18}
update : User{name='人皇sky', age=18}
select id : 1
select all
delete id : 1
----------------metric-------------------
insert count : 1
update count : 1
delete count : 1
select count : 2
----------------metric-------------------
```

#### CGLIB动态代理实现方式

MethodInterceptor

```java
package io.zjh.dynamicproxy.cglib;

import io.zjh.metric.CrudMetric;
import net.sf.cglib.proxy.MethodInterceptor;
import net.sf.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class MapperInterceptor implements MethodInterceptor {

    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        String name = method.getName();
        if (name.startsWith("insert")) {
            CrudMetric.increaseInsert();
        } else if (name.startsWith("update")) {
            CrudMetric.increaseUpdate();
        } else if (name.startsWith("delete")) {
            CrudMetric.increaseDelete();
        } else if (name.startsWith("select")) {
            CrudMetric.increaseSelect();
        }
        return methodProxy.invokeSuper(o, objects);
    }

}

```

创建工厂

```java
package io.zjh.dynamicproxy.cglib;

import net.sf.cglib.proxy.Enhancer;

@SuppressWarnings("all")
public class CGLIBMapperProxyFactory {

    public static <T> T newInstance(Class<T> clazz) {
        Enhancer enhancer = new Enhancer();
        enhancer.setClassLoader(clazz.getClassLoader());
        enhancer.setSuperclass(clazz);
        enhancer.setCallback(new MapperInterceptor());
        return (T) enhancer.create();
    }

}

```

test

```java
package io.zjh.dynamicproxy.cglib;

import io.zjh.InnerUserMapper;
import io.zjh.UserMapper;
import io.zjh.entity.User;
import io.zjh.metric.CrudMetric;

public class CGLIBMain {
    public static void main(String[] args) {
        InnerUserMapper innerUserMapper = CGLIBMapperProxyFactory.newInstance(InnerUserMapper.class);
        User user = new User("1", "sky", 18);
        innerUserMapper.insert(user);
        user.setName("人皇sky");
        innerUserMapper.update(user);
        innerUserMapper.selectById(user.getId());
        innerUserMapper.selectAll();
        innerUserMapper.deleteById(user.getId());
        CrudMetric.metric();

        CrudMetric.clear();
        System.out.println("*****************************************");

        UserMapper userMapper = CGLIBMapperProxyFactory.newInstance(UserMapper.class);
        userMapper.insert(user);
        user.setName("sky");
        userMapper.update(user);
        userMapper.selectById(user.getId());
        userMapper.selectAll();
        userMapper.deleteById(user.getId());
        CrudMetric.metric();
    }
}

```

执行结果

```text
insert : User{name='sky', age=18}
update : User{name='人皇sky', age=18}
select id : 1
select all
delete id : 1
----------------metric-------------------
insert count : 1
update count : 1
delete count : 1
select count : 2
----------------metric-------------------
*****************************************
insert : User{name='人皇sky', age=18}
update : User{name='sky', age=18}
select id : 1
select all
delete id : 1
----------------metric-------------------
insert count : 1
update count : 1
delete count : 1
select count : 2
----------------metric-------------------
```

## 小结

![](/medias/assets/design-pattern/代理模式.png)
