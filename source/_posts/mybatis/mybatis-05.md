---
title: Mybatis（五）：一级缓存
date: 2021-08-14 20:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 一级缓存
tags: Mybatis
categories:
- [Mybatis]
---

## 一级缓存

#### 介绍

在一次会话中，执行多次查询条件完全相同的SQL，Mybatis提供了一级缓存的方案进行优化。如果相同的查询语句，会优先命中一级缓存，避免直接对数据库进行查询，提高性能。

#### 配置

一级缓存可配置类型有两个

1. SESSION：在一个会话中执行的所有SQL语句都会共享同一个缓存
2. STATEMENT：缓存仅对当前执行的Statement有效

一级缓存默认是开启的，且类型是SESSION

```java
 protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
```

如果需要修改类型，可以在XML种进行配置

```java
<setting name="localCacheScope" value="STATEMENT"/>
```

#### 源码解析

每个Session持有一个Executor

```java
public class DefaultSqlSession implements SqlSession {
  private final Executor executor;
}
```

一级缓存是BaseExecutor的一个属性

```java
public abstract class BaseExecutor implements Executor {

  protected PerpetualCache localCache;

}
```

PerpetualCache实现了Cache接口，内部就是一个HashMap而已

```java
public class PerpetualCache implements Cache {

  private final String id;

  private final Map<Object, Object> cache = new HashMap<>();

  public PerpetualCache(String id) {
    this.id = id;
  }

  @Override
  public String getId() {
    return id;
  }

  @Override
  public int getSize() {
    return cache.size();
  }

  @Override
  public void putObject(Object key, Object value) {
    cache.put(key, value);
  }

  @Override
  public Object getObject(Object key) {
    return cache.get(key);
  }

  @Override
  public Object removeObject(Object key) {
    return cache.remove(key);
  }

  @Override
  public void clear() {
    cache.clear();
  }

  @Override
  public boolean equals(Object o) {
    if (getId() == null) {
      throw new CacheException("Cache instances require an ID.");
    }
    if (this == o) {
      return true;
    }
    if (!(o instanceof Cache)) {
      return false;
    }

    Cache otherCache = (Cache) o;
    return getId().equals(otherCache.getId());
  }

  @Override
  public int hashCode() {
    if (getId() == null) {
      throw new CacheException("Cache instances require an ID.");
    }
    return getId().hashCode();
  }

}
```

执行查询的时候，SqlSession委托Executor执行查询,首先会进入BaseExecutor的query方法

```java
 @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameter);
    CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
    return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
  }

```

生成一级缓存键

```java
  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(rowBounds.getOffset());
    cacheKey.update(rowBounds.getLimit());
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (ParameterMapping parameterMapping : parameterMappings) {
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

可以看出，如果一条SQL的**StatementId + Offset + Limmit + Sql + Params + environment**一样，则认为是相同的SQL

继续执行BaseExecutor的query方法

```java
  @SuppressWarnings("unchecked")
  @Override
  public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      // 从一级缓存中获取查询结果
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        // 存储过程相关
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        // 从数据库中查询
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      // 如果一级缓存设置的是有效范围是STATEMENT，则清除一级缓存
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```

真正去数据库中查询前先在一级缓存中检查是否已存在查询结果，只有不存在的时候才会调用queryFromDatabase方法去数据库中查询，当然最终的查询还得委派给BaseExecutor的子类去执行。

```java
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    // 一级缓存：放入占位缓存值
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      // 一级缓存：保证清除占位缓存值
      localCache.removeObject(key);
    }
    // 一级缓存：放入查询结果
    localCache.putObject(key, list);
    // 存储过程输出参数缓存
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }

```

代码中看到，一级缓存是在queryFromDatabase方法中存入的。

缓存只对查询有效，一旦执行了增删改的操作，缓存就应该失效了。增删改对应的都是BaseExecutor的update方法。

```java
  @Override
  public int update(MappedStatement ms, Object parameter) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing an update").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    clearLocalCache();
    return doUpdate(ms, parameter);
  }
```

代码中看到每次执行更新操作前，都会去清除一级缓存。

#### 存在的问题

一级缓存如果设置是SESSION级别，则它的有效范围是整个会话，如果同时有多个会话去修改数据，那么一级缓存可能会引起脏读

#### 例子

开启两个session，session1查询后让一级缓存生效，然后session2对数据进行更新，session1、session2再进行查询

```java
  @Test
  void test1() {
    try (SqlSession session1 = sqlMapper.openSession(true);
         SqlSession session2 = sqlMapper.openSession(true);) {
      AuthorMapper mapper1 = session1.getMapper(AuthorMapper.class);
      AuthorMapper mapper2 = session2.getMapper(AuthorMapper.class);

      Author expected = mapper1.selectAuthor(101);
      System.out.println("session1-->" + expected);

      Author updateAuthor = new Author();
      updateAuthor.setId(expected.getId());
      updateAuthor.setUsername("NewUsername");
      updateAuthor.setEmail("newEmail");
      updateAuthor.setPassword("newPassword");
      updateAuthor.setBio("newBio");
      int count = mapper2.updateAuthor(updateAuthor);
      assertEquals(1, count);
      System.out.println("session2 update-->");

      System.out.println("session1-->" + mapper1.selectAuthor(101));
      System.out.println("session2-->" + mapper2.selectAuthor(101));
    }
  }
```

执行结果

```text
session1-->Author : 101 : jim : jim@ibatis.apache.org
session2 update-->
session1-->Author : 101 : jim : jim@ibatis.apache.org
session2-->Author : 101 : NewUsername : newEmail
```

从结果可以看到，session2进行了更新二session1还是查到了老数据，这就产生了脏读

## 小结

1. 一级缓存生命周期和SqlSession的一致
2. 一级缓存有两种配置，SESSION和STATEMENT，SESSION是会话中所有的SQL共享，STATEMENT只是当前Statement有效
3. 一级缓存在多会话中会引起脏读，建议把它设置为STATEMENT
4. 一级缓存内部设计简单，只是一个没有容量限定的HashMap，在缓存的功能性上有所欠缺。