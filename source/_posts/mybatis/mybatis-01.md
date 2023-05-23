---
title: Mybatis（一）：总体架构
date: 2021-08-07 15:30:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 总体架构
tags: Mybatis
categories:
- [Mybatis]
---
## 总体架构

![](/medias/assets/mybatis/20210807150324.png)

![](/medias/assets/mybatis/20210807151528.png)

## 核心组件说明

### SqlSession

顶层API，表示和数据库的会话，提供增删查改的接口

```java
public interface SqlSession extends Closeable {
  <T> T selectOne(String statement);
  <T> T selectOne(String statement, Object parameter);
  <E> List<E> selectList(String statement);
  <E> List<E> selectList(String statement, Object parameter);
  <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);
  <K, V> Map<K, V> selectMap(String statement, String mapKey);
  <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey);
  <K, V> Map<K, V> selectMap(String statement, Object parameter, String mapKey, RowBounds rowBounds);
  <T> Cursor<T> selectCursor(String statement);
  <T> Cursor<T> selectCursor(String statement, Object parameter);
  <T> Cursor<T> selectCursor(String statement, Object parameter, RowBounds rowBounds);
  void select(String statement, Object parameter, ResultHandler handler);
  void select(String statement, ResultHandler handler);
  void select(String statement, Object parameter, RowBounds rowBounds, ResultHandler handler);
  int insert(String statement);
  int insert(String statement, Object parameter);
  int update(String statement);
  int update(String statement, Object parameter);
  int delete(String statement);
  int delete(String statement, Object parameter);
  void commit();
  void commit(boolean force);
  void rollback();
  void rollback(boolean force);
  List<BatchResult> flushStatements();
  void close();
  void clearCache();
  Configuration getConfiguration();
  <T> T getMapper(Class<T> type);
  Connection getConnection();
}
```

### Executor

执行器，接受SqlSession的委托，负责调度，包揽SQL语句的生成和维护查询缓存等操作。总共有四个执行器，分别是SimpleExecutor、ReuseExecutor、CachingExecutor、BatchExecutor。

```java
public interface Executor {

  ResultHandler NO_RESULT_HANDLER = null;

  int update(MappedStatement ms, Object parameter) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;

  <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;

  <E> Cursor<E> queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;

  List<BatchResult> flushStatements() throws SQLException;

  void commit(boolean required) throws SQLException;

  void rollback(boolean required) throws SQLException;

  CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);

  boolean isCached(MappedStatement ms, CacheKey key);

  void clearLocalCache();

  void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class<?> targetType);

  Transaction getTransaction();

  void close(boolean forceRollback);

  boolean isClosed();

  void setExecutorWrapper(Executor executor);

}
```

### StatementHandler

封装JDBC的Statement操作，接受Executor的委托，负责对Statement进行操作。

```java
public interface StatementHandler {

  Statement prepare(Connection connection, Integer transactionTimeout)
      throws SQLException;

  void parameterize(Statement statement)
      throws SQLException;

  void batch(Statement statement)
      throws SQLException;

  int update(Statement statement)
      throws SQLException;

  <E> List<E> query(Statement statement, ResultHandler resultHandler)
      throws SQLException;

  <E> Cursor<E> queryCursor(Statement statement)
      throws SQLException;

  BoundSql getBoundSql();

  ParameterHandler getParameterHandler();

}
```

### ResultHandler

负责将JDBC返回的结果ResultSet转换为List集合

```java
public interface ResultHandler<T> {

  void handleResult(ResultContext<? extends T> resultContext);

}
```

### TypeHandler

负责Java类型和JDBC类型之间的转换

```java
public interface TypeHandler<T> {

  void setParameter(PreparedStatement ps, int i, T parameter, JdbcType jdbcType) throws SQLException;

  T getResult(ResultSet rs, String columnName) throws SQLException;

  T getResult(ResultSet rs, int columnIndex) throws SQLException;

  T getResult(CallableStatement cs, int columnIndex) throws SQLException;

}

```

### MapedStatement

封装了一条SQL语句，对应XML中的insert、delete、update、select节点或者注解方式的配置

```java
public final class MappedStatement {
  // mapper配置文件名，如：UserMapper.xml
  private String resource;
  // 全局配置
  private Configuration configuration;
  // 节点的id属性加命名空间,如：com.lucky.mybatis.dao.UserMapper.selectByExample
  private String id;
  // 获取数据大小
  private Integer fetchSize;
  // 执行超时时间
  private Integer timeout;
  // 操作SQL的对象的类型
  private StatementType statementType;
  // 结果类型
  private ResultSetType resultSetType;
  // sql语句
  private SqlSource sqlSource;
  // 缓存
  private Cache cache;
  // 参数映射
  private ParameterMap parameterMap;
  // 结果集映射
  private List<ResultMap> resultMaps;
  // 是否清除缓存
  private boolean flushCacheRequired;
  // 是否使用缓存
  private boolean useCache;
  // 结果集是否排序
  private boolean resultOrdered;
  // sql语句类型
  private SqlCommandType sqlCommandType;
  // 主键生成器
  private KeyGenerator keyGenerator;
  // 主键属性
  private String[] keyProperties;
  // 主键列
  private String[] keyColumns;
  // 是否有嵌套结果映射
  private boolean hasNestedResultMaps;
  // 数据库id
  private String databaseId;
  // 日志对象
  private Log statementLog;
  // sql语句解析驱动
  private LanguageDriver lang;
  // 结果集
  private String[] resultSets;
}
```

### SqlSource

根据传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中

```java
public interface SqlSource {

  BoundSql getBoundSql(Object parameterObject);

}

```

### BoundSql

表示动态生成的SQL语句以及相应的参数信息

```java
public class BoundSql {

  private final String sql;
  private final List<ParameterMapping> parameterMappings;
  private final Object parameterObject;
  private final Map<String, Object> additionalParameters;
  private final MetaObject metaParameters;

  public BoundSql(Configuration configuration, String sql, List<ParameterMapping> parameterMappings, Object parameterObject) {
    this.sql = sql;
    this.parameterMappings = parameterMappings;
    this.parameterObject = parameterObject;
    this.additionalParameters = new HashMap<>();
    this.metaParameters = configuration.newMetaObject(additionalParameters);
  }

  public String getSql() {
    return sql;
  }

  public List<ParameterMapping> getParameterMappings() {
    return parameterMappings;
  }

  public Object getParameterObject() {
    return parameterObject;
  }

  public boolean hasAdditionalParameter(String name) {
    String paramName = new PropertyTokenizer(name).getName();
    return additionalParameters.containsKey(paramName);
  }

  public void setAdditionalParameter(String name, Object value) {
    metaParameters.setValue(name, value);
  }

  public Object getAdditionalParameter(String name) {
    return metaParameters.getValue(name);
  }
}
```

### Configration

全局配置，所有的配置信息都在这个对象中

```java
public class Configuration {

  // 环境配置,比如prod、stage、dev，可配置不同事务管理器工厂和数据源
  protected Environment environment;

  // 允许在嵌套语句中使用分页（RowBounds）。如果允许使用则设置为false
  protected boolean safeRowBoundsEnabled;
  protected boolean safeResultHandlerEnabled = true;
  // 是否开启自动驼峰命名规则（camel case）映射，即从经典数据库列名 A_COLUMN 到经典 Java 属性名 aColumn 的类似映射
  protected boolean mapUnderscoreToCamelCase;
  protected boolean aggressiveLazyLoading;
  // 是否允许单一语句返回多结果集（需要兼容驱动）
  protected boolean multipleResultSetsEnabled = true;
  // 允许JDBC支持自动生成主键，需要驱动兼容。 如果设置为 true 则这个设置强制使用自动生成主键，尽管一些驱动不能兼容但仍可正常工作（比如 Derby）
  protected boolean useGeneratedKeys;
  // 使用列标签代替列名。不同的驱动在这方面会有不同的表现， 具体可参考相关驱动文档或通过测试这两种不同的模式来观察所用驱动的结果
  protected boolean useColumnLabel = true;
  // 是否开启二级缓存
  protected boolean cacheEnabled = true;
  protected boolean callSettersOnNulls;
  protected boolean useActualParamName = true;
  protected boolean returnInstanceForEmptyRow;
  protected boolean shrinkWhitespacesInSql;

  protected String logPrefix;
  protected Class<? extends Log> logImpl;
  protected Class<? extends VFS> vfsImpl;
  protected Class<?> defaultSqlProviderType;
  // MyBatis 利用本地缓存机制（Local Cache）防止循环引用（circular references）和加速重复嵌套查询。 默认值为 SESSION，这种情况下会缓存一个会话中执行的所有查询。 若设置值为 STATEMENT，本地会话仅用在语句执行上，对相同 SqlSession 的不同调用将不会共享数据
  protected LocalCacheScope localCacheScope = LocalCacheScope.SESSION;
  // 当没有为参数提供特定的 JDBC 类型时，为空值指定 JDBC 类型。 某些驱动需要指定列的 JDBC 类型，多数情况直接用一般类型即可，比如 NULL、VARCHAR 或 OTHER。
  protected JdbcType jdbcTypeForNull = JdbcType.OTHER;
  // 指定哪个对象的方法触发一次延迟加载
  protected Set<String> lazyLoadTriggerMethods = new HashSet<>(Arrays.asList("equals", "clone", "hashCode", "toString"));
  // 设置超时时间，它决定驱动等待数据库响应的秒数
  protected Integer defaultStatementTimeout;
  // 为驱动的结果集获取数量（fetchSize）设置一个提示值。此参数只可以在查询设置中被覆盖
  protected Integer defaultFetchSize;
  protected ResultSetType defaultResultSetType;
  // 配置默认的执行器。SIMPLE 就是普通的执行器；REUSE 执行器会重用预处理语句（prepared statements）； BATCH 执行器将重用语句并执行批量更新
  protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
  // 指定 MyBatis 应如何自动映射列到字段或属性。 NONE 表示取消自动映射；PARTIAL 只会自动映射没有定义嵌套结果集映射的结果集。 FULL 会自动映射任意复杂的结果集（无论是否嵌套）。
  protected AutoMappingBehavior autoMappingBehavior = AutoMappingBehavior.PARTIAL;
  // 指定发现自动映射目标未知列（或者未知属性类型）的行为。NONE: 不做任何反应、WARNING: 输出提醒日志 ('org.apache.ibatis.session.AutoMappingUnknownColumnBehavior'的日志等级必须设置为 WARN)、FAILING: 映射失败 (抛出 SqlSessionException)
  protected AutoMappingUnknownColumnBehavior autoMappingUnknownColumnBehavior = AutoMappingUnknownColumnBehavior.NONE;

  protected Properties variables = new Properties();
  protected ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
  protected ObjectFactory objectFactory = new DefaultObjectFactory();
  protected ObjectWrapperFactory objectWrapperFactory = new DefaultObjectWrapperFactory();

  // 延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。 特定关联关系中可通过设置fetchType属性来覆盖该项的开关状态
  protected boolean lazyLoadingEnabled = false;
  protected ProxyFactory proxyFactory = new JavassistProxyFactory(); // #224 Using internal Javassist instead of OGNL

  protected String databaseId;
  /**
   * Configuration factory class.
   * Used to create Configuration for loading deserialized unread properties.
   *
   * @see <a href='https://github.com/mybatis/old-google-code-issues/issues/300'>Issue 300 (google code)</a>
   */
  protected Class<?> configurationFactory;

  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  protected final InterceptorChain interceptorChain = new InterceptorChain();
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry(this);
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
  protected final LanguageDriverRegistry languageRegistry = new LanguageDriverRegistry();

  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
  protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");

  protected final Set<String> loadedResources = new HashSet<>();
  protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");

  protected final Collection<XMLStatementBuilder> incompleteStatements = new LinkedList<>();
  protected final Collection<CacheRefResolver> incompleteCacheRefs = new LinkedList<>();
  protected final Collection<ResultMapResolver> incompleteResultMaps = new LinkedList<>();
  protected final Collection<MethodResolver> incompleteMethods = new LinkedList<>();

  /*
   * A map holds cache-ref relationship. The key is the namespace that
   * references a cache bound to another namespace and the value is the
   * namespace which the actual cache is bound to.
   */
  protected final Map<String, String> cacheRefMap = new HashMap<>();
```

## 小结

1. Mybatis通过暴露SqlSession接口给用户完成数据增删查改操作。
2. Executor执行器包含4个，分别是SimpleExecutor、CachingExecutor、ReuseExecutor、BatchExecutor。
3. StatementHandler封装JDBC Statement，负责处理Statement相关处理，比如参数设置。
4. ResultHandler负责把JDBC的执行结果ResultSet转换为List集合。
5. MapedStatement封装了XML中的insert、delete、update、select节点。
6. SqlSource负责根据用户传递的parameterObject，动态地生成SQL语句，将信息封装到BoundSql对象中。
7. BoundSql表示动态生成的SQL语句以及相应的参数信息
8. Configuration全局配置对象，包含所有配置信息
