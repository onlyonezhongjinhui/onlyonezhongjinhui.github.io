---
title: Mybatis（九）：分页机制
date: 2021-08-28 12:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 分页机制
tags: Mybatis
categories:
- [Mybatis]
---
## 分页机制

### 介绍

Mybatis分页查询属于逻辑分页，它是执行完SQL拿到全部的数据后在内存中再进行分页拿指定分页数据，很显然当数据量大的时候，这样做性能很差，很容易发生OOM

### 源码分析

#### 关键类

##### RowBounds

```java
public class RowBounds {

  public static final int NO_ROW_OFFSET = 0;
  public static final int NO_ROW_LIMIT = Integer.MAX_VALUE;
  public static final RowBounds DEFAULT = new RowBounds();

  private final int offset;
  private final int limit;

  public RowBounds() {
    this.offset = NO_ROW_OFFSET;
    this.limit = NO_ROW_LIMIT;
  }

  public RowBounds(int offset, int limit) {
    this.offset = offset;
    this.limit = limit;
  }

  public int getOffset() {
    return offset;
  }

  public int getLimit() {
    return limit;
  }

}
```

RowBounds定义了offset、limit两个属性，这就是分页查询关心的两个属性，开始的偏移量和没页数量限制。

#### 过程分析

1. StatementHandler执行查询SQL
2. ResultSetHandler处理查询结果

以SimpleStatementHandler为例，代码执行流程如下：

SimpleStatementHandler执行query方法，执行完SQL后交给ResultSetHandler处理执行结果

```java
  @Override
  public <E> List<E> query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    statement.execute(sql);
    return resultSetHandler.handleResultSets(statement);
  }
```

默认的DefaultResultSetHandler执行handleResultSets方法，结果的处理又交给了handleResultSet方法

```java
 @Override
  public List<Object> handleResultSets(Statement stmt) throws SQLException {
    ErrorContext.instance().activity("handling results").object(mappedStatement.getId());

    final List<Object> multipleResults = new ArrayList<>();

    int resultSetCount = 0;
    ResultSetWrapper rsw = getFirstResultSet(stmt);

    List<ResultMap> resultMaps = mappedStatement.getResultMaps();
    int resultMapCount = resultMaps.size();
    validateResultMapsCount(rsw, resultMapCount);
    while (rsw != null && resultMapCount > resultSetCount) {
      ResultMap resultMap = resultMaps.get(resultSetCount);
      handleResultSet(rsw, resultMap, multipleResults, null);
      rsw = getNextResultSet(stmt);
      cleanUpAfterHandlingResultSet();
      resultSetCount++;
    }

    String[] resultSets = mappedStatement.getResultSets();
    if (resultSets != null) {
      while (rsw != null && resultSetCount < resultSets.length) {
        ResultMapping parentMapping = nextResultMaps.get(resultSets[resultSetCount]);
        if (parentMapping != null) {
          String nestedResultMapId = parentMapping.getNestedResultMapId();
          ResultMap resultMap = configuration.getResultMap(nestedResultMapId);
          handleResultSet(rsw, resultMap, null, parentMapping);
        }
        rsw = getNextResultSet(stmt);
        cleanUpAfterHandlingResultSet();
        resultSetCount++;
      }
    }

    return collapseSingleResultList(multipleResults);
  }
```

handleResultSet方法中进一步委托给handleRowValues处理

```java
  private void handleResultSet(ResultSetWrapper rsw, ResultMap resultMap, List<Object> multipleResults, ResultMapping parentMapping) throws SQLException {
    try {
      if (parentMapping != null) {
        handleRowValues(rsw, resultMap, null, RowBounds.DEFAULT, parentMapping);
      } else {
        if (resultHandler == null) {
          DefaultResultHandler defaultResultHandler = new DefaultResultHandler(objectFactory);
          handleRowValues(rsw, resultMap, defaultResultHandler, rowBounds, null);
          multipleResults.add(defaultResultHandler.getResultList());
        } else {
          handleRowValues(rsw, resultMap, resultHandler, rowBounds, null);
        }
      }
    } finally {
      // issue #228 (close resultsets)
      closeResultSet(rsw.getResultSet());
    }
  }
```

handleRowValues进一步调用了handleRowValuesForNestedResultMap或handleRowValuesForSimpleResultMap方法

```java
  public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }
```

以handleRowValuesForSimpleResultMap为例，通过层层调用，分页的地方终于出现了

```java
  private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }
```

skipRows就是分页的具体实现

```java
  private void skipRows(ResultSet rs, RowBounds rowBounds) throws SQLException {
  
    if (rs.getType() != ResultSet.TYPE_FORWARD_ONLY) {
      // 如果结果集类型不是只能往前滚动，offset不是0，则结果集直接定位到offset
      if (rowBounds.getOffset() != RowBounds.NO_ROW_OFFSET) {
        rs.absolute(rowBounds.getOffset());
      }
    } else {
      // 如果结果集类型是只能往前滚动，则一条条往前翻滚，直到offset
      for (int i = 0; i < rowBounds.getOffset(); i++) {
        if (!rs.next()) {
          break;
        }
      }
    }
  }
```

通过skipRows方法，就能定位到要获取结果的位置了。接下来就是RowBounds的limit属性发挥作用的地方了

```java
  private boolean shouldProcessMoreRows(ResultContext<?> context, RowBounds rowBounds) {
    return !context.isStopped() && context.getResultCount() < rowBounds.getLimit();
  }
```

shouldProcessMoreRows方法中断取的结果条数是不是到了limit限制的条数，如果没到就继续取，否则退出while循环

## 小结

1. Mybatis分页属于逻辑分页，存在性能问题，不建议使用
2. 分页信息通过RowBounds定义