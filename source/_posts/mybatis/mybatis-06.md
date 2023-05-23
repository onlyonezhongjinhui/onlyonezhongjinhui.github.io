---
title: Mybatis（六）：二级缓存
date: 2021-08-15 12:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 二级缓存
tags: Mybatis
categories:
- [Mybatis]
---

## 二级缓存

#### 介绍

二级缓存是比一级缓存更大作用范围的缓存方案，作用范围是整个namespace，通常就是一个mapper。当然也可以通过引用其它mapper扩大作用范围。但是更大的作用范围也意味着一旦出现更新，缓存就会大面积失效，缓存的效果就会大大折扣。

#### 配置

二级缓存默认是开启的，但是要使用则需要进行显示配置

```java
protected boolean cacheEnabled = true;
```

```xml
  <cache readOnly="true"/>
```

##### 二级缓存支持的配置

##### type

缓存实现类型，Mybatis提供了Cache接口，而唯一实现了Cache接口的子类是PerpetualCache，也就是一级缓存用的缓存类。为了丰富缓存的类型，Mybatis通过装饰器模式，增加了很多缓存类型。默认使用PerpetualCache缓存。

1. BlockingCache：阻塞缓存，当在缓存中找不到元素时，它会在缓存键上设置锁，这样，其他线程将等待该元素被填充，而不是命中数据库。从本质上讲，此实现在不正确使用时可能导致死锁。
2. FifoCache：先进先出缓存，当缓存队列满了以后先删除最先放入的缓存数据。
3. LoggingCache：日志缓存，用于记录请求缓存Key的次数、命中Key的次数，计算命中率。如果开启debug日志，则会使用该日志缓存。
4. LruCache：最近最少使用缓存，通过继承LinkedHashMap并重写removeEldestEntry方法实现，当缓存大小超过了指定大小，removeEldestEntry返回true并保存最近最少使用的key，然后通过key删除缓存。
5. ScheduledCache：定时清除缓存，根据指定时间间隔进行清除。
6. SerializedCache：序列化缓存，放入缓存前先进行缓存对象序列化，取出后再反序列化为对象，缓存值必须实现序列化接口。
7. SoftCache：软引用缓存
8. SynchronizedCache：同步缓存，所有操作都加锁
9. TransactionCache：事务缓存，事务提交后才生效
10. WeakCache：弱引用缓存。

##### eviction

淘汰策略，默认LRU，也就是会使用LruCache。也可以配置FIFO使用FifoCache。

##### flushInterval

刷新缓存时间间隔，配置后使用ScheduledCache。

##### size

缓存容量大小（元素个数）

##### readOnly

读写缓存标识，默认是只读缓存。如果配置为true，则使用SerializedCache。

##### blocking

阻塞标识，默认是非阻塞，也就是会使用BlockingCache。

#### 源码解析

二级缓存是CachingExecutor加持，通过TransactionalCacheManager进行管理

```java
public class CachingExecutor implements Executor {
  private final TransactionalCacheManager tcm = new TransactionalCacheManager();
}
```

TransactionalCacheManager类比较简单，维护了一个HashMap，key是namespace对应的缓存，value是事务缓存

```java
public class TransactionalCacheManager {

  private final Map<Cache, TransactionalCache> transactionalCaches = new HashMap<>();

  public void clear(Cache cache) {
    getTransactionalCache(cache).clear();
  }

  public Object getObject(Cache cache, CacheKey key) {
    return getTransactionalCache(cache).getObject(key);
  }

  public void putObject(Cache cache, CacheKey key, Object value) {
    getTransactionalCache(cache).putObject(key, value);
  }

  public void commit() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.commit();
    }
  }

  public void rollback() {
    for (TransactionalCache txCache : transactionalCaches.values()) {
      txCache.rollback();
    }
  }

  private TransactionalCache getTransactionalCache(Cache cache) {
    return transactionalCaches.computeIfAbsent(cache, TransactionalCache::new);
  }

}
```

TransactionalCache事务缓存的特点是查询后会把数据记录在entriesToAddOnCommit属性中，只有事务提交了，才会把数据从entriesToAddOnCommit真正放入缓存队列delegate中

```java
public class TransactionalCache implements Cache {

  private static final Log log = LogFactory.getLog(TransactionalCache.class);

  private final Cache delegate;
  private boolean clearOnCommit;
  private final Map<Object, Object> entriesToAddOnCommit;
  private final Set<Object> entriesMissedInCache;

  public TransactionalCache(Cache delegate) {
    this.delegate = delegate;
    this.clearOnCommit = false;
    this.entriesToAddOnCommit = new HashMap<>();
    this.entriesMissedInCache = new HashSet<>();
  }

  @Override
  public String getId() {
    return delegate.getId();
  }

  @Override
  public int getSize() {
    return delegate.getSize();
  }

  @Override
  public Object getObject(Object key) {
    // issue #116
    Object object = delegate.getObject(key);
    if (object == null) {
      entriesMissedInCache.add(key);
    }
    // issue #146
    if (clearOnCommit) {
      return null;
    } else {
      return object;
    }
  }

  @Override
  public void putObject(Object key, Object object) {
    entriesToAddOnCommit.put(key, object);
  }

  @Override
  public Object removeObject(Object key) {
    return null;
  }

  @Override
  public void clear() {
    clearOnCommit = true;
    entriesToAddOnCommit.clear();
  }

  public void commit() {
    if (clearOnCommit) {
      delegate.clear();
    }
    flushPendingEntries();
    reset();
  }

  public void rollback() {
    unlockMissedEntries();
    reset();
  }

  private void reset() {
    clearOnCommit = false;
    entriesToAddOnCommit.clear();
    entriesMissedInCache.clear();
  }

  private void flushPendingEntries() {
    for (Map.Entry<Object, Object> entry : entriesToAddOnCommit.entrySet()) {
      delegate.putObject(entry.getKey(), entry.getValue());
    }
    for (Object entry : entriesMissedInCache) {
      if (!entriesToAddOnCommit.containsKey(entry)) {
        delegate.putObject(entry, null);
      }
    }
  }

  private void unlockMissedEntries() {
    for (Object entry : entriesMissedInCache) {
      try {
        delegate.removeObject(entry);
      } catch (Exception e) {
        log.warn("Unexpected exception while notifying a rollback to the cache adapter. "
            + "Consider upgrading your cache adapter to the latest version. Cause: " + e);
      }
    }
  }

}
```

查询流程的流程

```java
@Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
    BoundSql boundSql = ms.getBoundSql(parameterObject);
    // 创建缓存键
    CacheKey key = createCacheKey(ms, parameterObject, rowBounds, boundSql);
    return query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }

  @Override
  public <E> List<E> query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
    throws SQLException {
    Cache cache = ms.getCache();
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, boundSql);
        @SuppressWarnings("unchecked")
        // 从二级缓存管理器中获取查询结果
        List<E> list = (List<E>) tcm.getObject(cache, key);
        if (list == null) {
          // 走一级缓存流程
          list = delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          // 二级缓存中放入查询结果
          tcm.putObject(cache, key, list); // issue #578 and #116
        }
        return list;
      }
    }
    return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```

```java
  @Override
  public void commit(boolean required) throws SQLException {
    delegate.commit(required);
    // 只有事务提交了，二级缓存才会生效
    tcm.commit();
  }

  @Override
  public void rollback(boolean required) throws SQLException {
    try {
      delegate.rollback(required);
    } finally {
      if (required) {
        tcm.rollback();
      }
    }
  }
```

1. CachingExecutor执行query方法
2. 调用BaseExecutor的createCacheKey方法生成缓存键，缓存键和一级缓存是一样的
3. 先从二级缓存管理器中查询是否已经有查询结果，如果有则直接返回数据
4. 如果没有则就会进入一级缓存的执行流程
5. 一级缓存流程执行完成后回来，把查询结果也缓存到保存到二级缓存中，注意这个时候缓存并没有生效，还得等待事务提交

更新流程

```java
  @Override
  public int update(MappedStatement ms, Object parameterObject) throws SQLException {
    flushCacheIfRequired(ms);
    return delegate.update(ms, parameterObject);
  }
```

每次更新前都会执行flushCacheIfRequired方法，也就是清除缓存

```java
  private void flushCacheIfRequired(MappedStatement ms) {
    Cache cache = ms.getCache();
    if (cache != null && ms.isFlushCacheRequired()) {
      tcm.clear(cache);
    }
  }
```

## 小结

1. 二级缓存是namespace级别的共享，作用范围更大。
2. 一旦有更新，所有缓存都将失效。
3. 二级缓存需要事务提交后才会生效。
4. 二级缓存可搭配使用的缓存类型很多，可控性挺好。
5. 二级缓存和一级缓存一样，都很容易产生数据脏读，使用条件非常苛刻。
6. 二级缓存作为本地缓存，分布式环境下必定出现脏读。如果需要使用集中式缓存，需要自定义缓存实现Cache接口。一般情况下用Redis、Memcached更香。