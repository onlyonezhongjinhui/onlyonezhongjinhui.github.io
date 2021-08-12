---
title: Mybatis：Mapper配置解析详解（三）
date: 2021-08-11 15:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Mapper配置解析详解
tags: Mybatis
categories:
- [Mybatis]
{: id="20210810152348-th3zxsi"}
{: id="20210810175110-m2gteis"}
---
# Mapper配置解析详解

mapper解析的目的：

1. 注册mapper接口
2. 解析二级缓存
3. 解析参数映射
4. 解析结果映射
5. 解析sql片段
6. 解析sql语句

解析完后填充到configuration的这些成员变量里

```java
public class Configuration {
  protected final MapperRegistry mapperRegistry = new MapperRegistry(this);
  protected final Map<String, MappedStatement> mappedStatements = new StrictMap<MappedStatement>("Mapped Statements collection")
      .conflictMessageProducer((savedValue, targetValue) ->
          ". please check " + savedValue.getResource() + " and " + targetValue.getResource());
  protected final Map<String, Cache> caches = new StrictMap<>("Caches collection");
  protected final Map<String, ResultMap> resultMaps = new StrictMap<>("Result Maps collection");
  protected final Map<String, ParameterMap> parameterMaps = new StrictMap<>("Parameter Maps collection");
  protected final Map<String, KeyGenerator> keyGenerators = new StrictMap<>("Key Generators collection");
  protected final Map<String, XNode> sqlFragments = new StrictMap<>("XML fragments parsed from previous mappers");
}
```

看看具体的解析过程，为了专注mapper的解析，忽略XML的node节点的读取

mapper的解析分为4种配置情形，分别是下面四种

```xml
  <mappers>
    <mapper resource="org/apache/ibatis/builder/BlogMapper.xml"/>
    <mapper url="file:./src/test/java/org/apache/ibatis/builder/NestedBlogMapper.xml"/>
    <mapper class="org.apache.ibatis.builder.CachedAuthorMapper"/>
    <package name="org.apache.ibatis.builder.mapper"/>
  </mappers>
```

对应代码解析代码片段，这里简称package、resource、url、class方式进行解析

```java
  private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          // 通过包名去添加mapper接口
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            // resource方式
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            // url方式
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            // 通过类名直接添加
            Class<?> mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```

pakcage方式的注册过程

```java
1. XMLConfigBuilder类的mapperElement直接传入包名调用MapperRegistry的addMappers方法 
 public void addMappers(String packageName) {
    mapperRegistry.addMappers(packageName);
  }

2. 指定包名下的Object子类都是Mapper接口
  public void addMappers(String packageName) {
    addMappers(packageName, Object.class);
  }

3. 通过resolverUtil去扫描指定包路径下的类添加到MapperRegistry中
    public void addMappers(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> mapperSet = resolverUtil.getClasses();
    for (Class<?> mapperClass : mapperSet) {
      addMapper(mapperClass);
    }
  }

4. 生成对应的动态代理工厂保存到MapperRegistry的属性knownMappers中完成mapper接口的注册
  public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }

5. 接下来通过MapperAnnotationBuilder类解析对应的mapper配置
 public void parse() {
    String resource = type.toString();
    // 判断mapper接口是否已经加载过了，如果没加载过，则加载后再解析，如果已经加载过了，那么通过抢占互斥锁来完成已加载而未解析完成的操作
    if (!configuration.isResourceLoaded(resource)) {
      // 解析接口命名空间对应下的XML，调用的就是XML解析方式
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      // 解析二级缓存配置
      parseCache();
      // 解析二级缓存引入配置（多个mapper共享二级缓存）
      parseCacheRef();
      // 扫描接口下的所有方法
      for (Method method : type.getMethods()) {
        // 排除桥接、默认方法
        if (!canHaveStatement(method)) {
          continue;
        }
        // 解析包含@Select、@SelectProvider的方法
        if (getAnnotationWrapper(method, false, Select.class, SelectProvider.class).isPresent()
            && method.getAnnotation(ResultMap.class) == null) {
          // 解析select应该对应的resultMap
          parseResultMap(method);
        }
        try {
          // 解析SQL
          parseStatement(method);
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    // 抢占互斥锁解析
    parsePendingMethods();
  }
```

class方式注册过程

```java
1. 直接把类添加到MapperRegistry即可  
  public <T> void addMapper(Class<T> type) {
    mapperRegistry.addMapper(type);
  }

2. 后续步骤和package方式的4、5步骤一致
```

resource、url注册过程

```java
1. 通过resource、url获取配置mapper配置文件流，然后通过XMLMapperBuilder类的parse方法进行解析
  if (resource != null && url == null && mapperClass == null) {
    // resource方式
    ErrorContext.instance().resource(resource);
    InputStream inputStream = Resources.getResourceAsStream(resource);
    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
    mapperParser.parse();
  } else if (resource == null && url != null && mapperClass == null) {
    // url方式
    ErrorContext.instance().resource(url);
    InputStream inputStream = Resources.getUrlAsStream(url);
    XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
    mapperParser.parse();
  }

2. parse方法具体解析和package的步骤5意思是一样的，只不过这里是XML方式的
  public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      // 注册mapper接口
      bindMapperForNamespace();
    }

    // 解析resultMap
    parsePendingResultMaps();
    // 解析二级缓存
    parsePendingCacheRefs();
    // 解析SQL节点
    parsePendingStatements();
  }
```

看到了mapper解析的大致流程，发现是有两种方式，XML和注解方式（注解方式包含注解和XML，因为注解毕竟不能全部表达全部的意思）。

重点跟踪XML方式，XML方式是通过XMLMapperBuilder类来解析的，具体就是它的parse方法

```java
  public void parse() {
    // 判断资源是否已经加载
    if (!configuration.isResourceLoaded(resource)) {
      // 初次解析
      configurationElement(parser.evalNode("/mapper"));
      // 把resource添加到已加载资源中
      configuration.addLoadedResource(resource);
      // 注册mapper接口
      bindMapperForNamespace();
    }

    // 重试：解析未完成的resultMap
    parsePendingResultMaps();
    // 重试：解析未完成的二级缓存
    parsePendingCacheRefs();
    // 重试：解析未完成的SQL节点
    parsePendingStatements();
  }
```

初次解析把各个节点都解析了一个遍

```java
  private void configurationElement(XNode context) {
    try {
      // 判断是否配置了namespace，没有配置抛出异常终止解析
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.isEmpty()) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      // 校验并保存命名空间
      builderAssistant.setCurrentNamespace(namespace);
      // 解析二级缓存引用配置（跨mapper，多个mapper共用一个缓存，缓存影响范围更大）
      cacheRefElement(context.evalNode("cache-ref"));
      // 解析二级缓存配置
      cacheElement(context.evalNode("cache"));
      // 解析参数映射配置
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      // 解析结果映射配置
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      // sql片段配置解析
      sqlElement(context.evalNodes("/mapper/sql"));
      // 解析sql语句
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. The XML location is '" + resource + "'. Cause: " + e, e);
    }
  }
```

解析二级缓存引用配置

```java
  private void cacheRefElement(XNode context) {
    if (context != null) {
      // 添加到Configuration成员变量cacheRefMap中
      configuration.addCacheRef(builderAssistant.getCurrentNamespace(), context.getStringAttribute("namespace"));
      // 创建一个CacheRefResolver
      CacheRefResolver cacheRefResolver = new CacheRefResolver(builderAssistant, context.getStringAttribute("namespace"));
      try {
        // 真正解析
        cacheRefResolver.resolveCacheRef();
      } catch (IncompleteElementException e) {
        // 如果抛出IncompleteElementException异常，表示引用的是还未解析的mapper，则把它加入到未解析完成列表中
        configuration.addIncompleteCacheRef(cacheRefResolver);
      }
    }
  }
```

解析二级缓存配置

```java
  private void cacheElement(XNode context) {
    if (context != null) {
      // 读取缓存类型，如果未配置则使用默认PerpetualCache
      String type = context.getStringAttribute("type", "PERPETUAL");
      // 获取缓存类型的Class对象
      Class<? extends Cache> typeClass = typeAliasRegistry.resolveAlias(type);
      // 读取缓存淘汰策略，如果没有配置，则使用LRU（最近最少使用）
      String eviction = context.getStringAttribute("eviction", "LRU");
      // 获取指定淘汰策略缓存类型Class对象
      Class<? extends Cache> evictionClass = typeAliasRegistry.resolveAlias(eviction);
      // 读取缓存刷新间隔
      Long flushInterval = context.getLongAttribute("flushInterval");
      // 缓存容量大小（元素个数）
      Integer size = context.getIntAttribute("size");
      // 读取读写缓存标识，默认是只读缓存
      boolean readWrite = !context.getBooleanAttribute("readOnly", false);
      // 读取阻塞缓存标识，默认是非阻塞缓存
      boolean blocking = context.getBooleanAttribute("blocking", false);
      // 属性配置
      Properties props = context.getChildrenAsProperties();
      // 创建新缓存
      builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
    }
  }
```

解析参数映射配置

```java
  private void parameterMapElement(List<XNode> list) {
    // 遍历参数映射节点
    for (XNode parameterMapNode : list) {
      // 读取唯一id
      String id = parameterMapNode.getStringAttribute("id");
      // 读取参数类型
      String type = parameterMapNode.getStringAttribute("type");
      // 获取参数类型Class对象
      Class<?> parameterClass = resolveClass(type);
      // 读取parameter子节点
      List<XNode> parameterNodes = parameterMapNode.evalNodes("parameter");
      List<ParameterMapping> parameterMappings = new ArrayList<>();
      // 遍历parameter子节点
      for (XNode parameterNode : parameterNodes) {
        // 读取属性
        String property = parameterNode.getStringAttribute("property");
        // 读取java类型
        String javaType = parameterNode.getStringAttribute("javaType");
        // 读取jdbc类型
        String jdbcType = parameterNode.getStringAttribute("jdbcType");
        // 读取resultMap
        String resultMap = parameterNode.getStringAttribute("resultMap");
        // 参数方式（入参、出参、出入参）
        String mode = parameterNode.getStringAttribute("mode");
        // java类型jdbc类型转换处理器
        String typeHandler = parameterNode.getStringAttribute("typeHandler");
        // 数字精度
        Integer numericScale = parameterNode.getIntAttribute("numericScale");
        // 获取参数枚举
        ParameterMode modeEnum = resolveParameterMode(mode);
        // 获取java类型class对象
        Class<?> javaTypeClass = resolveClass(javaType);
        // 获取jdbc类型枚举
        JdbcType jdbcTypeEnum = resolveJdbcType(jdbcType);
        // 获取java类型jdbc类型转换处理器class对象
        Class<? extends TypeHandler<?>> typeHandlerClass = resolveClass(typeHandler);
        // 创建一个参数映射
        ParameterMapping parameterMapping = builderAssistant.buildParameterMapping(parameterClass, property, javaTypeClass, jdbcTypeEnum, resultMap, modeEnum, typeHandlerClass, numericScale);
        // 加入参数映射节点集合中
        parameterMappings.add(parameterMapping);
      }
      // 加入到configuration的成员属性parameterMaps中
      builderAssistant.addParameterMap(id, parameterClass, parameterMappings);
    }
  }
```

解析结果映射配置

```java
  private void resultMapElements(List<XNode> list) {
    // 遍历resultMap节点
    for (XNode resultMapNode : list) {
      try {
        // 逐个节点解析
        resultMapElement(resultMapNode);
      } catch (IncompleteElementException e) {
        // ignore, it will be retried
      }
    }
  }

  private ResultMap resultMapElement(XNode resultMapNode) {
    return resultMapElement(resultMapNode, Collections.emptyList(), null);
  }

  private ResultMap resultMapElement(XNode resultMapNode, List<ResultMapping> additionalResultMappings, Class<?> enclosingType) {
    ErrorContext.instance().activity("processing " + resultMapNode.getValueBasedIdentifier());
    // 读取结果类型(会递归调用)
    // 对于resultMap 获取 type
    // 对于collection 获取 ofType 或 javaType
    // 对于association 获取 javaType
    String type = resultMapNode.getStringAttribute("type",
      resultMapNode.getStringAttribute("ofType",
        resultMapNode.getStringAttribute("resultType",
          resultMapNode.getStringAttribute("javaType"))));
    // 获取结果类型class对象
    Class<?> typeClass = resolveClass(type);
    if (typeClass == null) {
      // 这里enclosingType一般都是父节点的type
      // 比如<resultMap type="test">
      //        <collection/>
      // 那么enclosingType便是test
      // 如果有些节点没有配置type，允许的情况下，可以直接使用父节点的type
      // 例如：
      //     <discriminator javaType="int" column="draft">
      //         <case value="1" resultType="DraftPost"/>
      //     </discriminator>
      //  case节点中就没有配置type
      typeClass = inheritEnclosingType(resultMapNode, enclosingType);
    }
    // 鉴别器
    Discriminator discriminator = null;
    // 结果属性映射
    List<ResultMapping> resultMappings = new ArrayList<>(additionalResultMappings);
    List<XNode> resultChildren = resultMapNode.getChildren();
    // 遍历子节点
    for (XNode resultChild : resultChildren) {
      if ("constructor".equals(resultChild.getName())) {
        // 解析构造函数配置
        processConstructorElement(resultChild, typeClass, resultMappings);
      } else if ("discriminator".equals(resultChild.getName())) {
        // 解析鉴别器
        discriminator = processDiscriminatorElement(resultChild, typeClass, resultMappings);
      } else {
        // 普通属性
        List<ResultFlag> flags = new ArrayList<>();
        if ("id".equals(resultChild.getName())) {
          flags.add(ResultFlag.ID);
        }
        // 添加单个结果属性映射
        resultMappings.add(buildResultMappingFromContext(resultChild, typeClass, flags));
      }
    }
    // 唯一id
    String id = resultMapNode.getStringAttribute("id",
      resultMapNode.getValueBasedIdentifier());
    // 继承类
    String extend = resultMapNode.getStringAttribute("extends");
    // 自动映射
    Boolean autoMapping = resultMapNode.getBooleanAttribute("autoMapping");
    // 创建ResultMapResolver
    ResultMapResolver resultMapResolver = new ResultMapResolver(builderAssistant, id, typeClass, extend, discriminator, resultMappings, autoMapping);
    try {
      // 添加resultMap到configuration的resultMaps属性中
      return resultMapResolver.resolve();
    } catch (IncompleteElementException e) {
      // 如果解析失败，则把它加入到未完成列表，后面可进行重试
      configuration.addIncompleteResultMap(resultMapResolver);
      throw e;
    }
  }

  protected Class<?> inheritEnclosingType(XNode resultMapNode, Class<?> enclosingType) {
    if ("association".equals(resultMapNode.getName()) && resultMapNode.getStringAttribute("resultMap") == null) {
      String property = resultMapNode.getStringAttribute("property");
      if (property != null && enclosingType != null) {
        MetaClass metaResultType = MetaClass.forClass(enclosingType, configuration.getReflectorFactory());
        return metaResultType.getSetterType(property);
      }
    } else if ("case".equals(resultMapNode.getName()) && resultMapNode.getStringAttribute("resultMap") == null) {
      return enclosingType;
    }
    return null;
  }
```

解析sql片段

```java
  private void sqlElement(List<XNode> list) {
    if (configuration.getDatabaseId() != null) {
      // 配置了全局databaseId
      sqlElement(list, configuration.getDatabaseId());
    }
    // 没有全局databaseId
    sqlElement(list, null);
  }

  private void sqlElement(List<XNode> list, String requiredDatabaseId) {
    // 遍历sql节点
    for (XNode context : list) {
      // 数据库id
      String databaseId = context.getStringAttribute("databaseId");
      // sql的id
      String id = context.getStringAttribute("id");
      // 全限定id
      id = builderAssistant.applyCurrentNamespace(id, false);
      // 检查数据库id是不是匹配，如果匹配就放入sqlFragments，否则忽略
      if (databaseIdMatchesCurrent(id, databaseId, requiredDatabaseId)) {
        sqlFragments.put(id, context);
      }
    }
  }
```

解析sql语句

```java
  private void buildStatementFromContext(List<XNode> list) {
    // 区别有无数据库id
    if (configuration.getDatabaseId() != null) {
      buildStatementFromContext(list, configuration.getDatabaseId());
    }
    buildStatementFromContext(list, null);
  }

  private void buildStatementFromContext(List<XNode> list, String requiredDatabaseId) {
    // 遍历sql节点
    for (XNode context : list) {
      // 创建XMLStatementBuilder用来解析sql
      final XMLStatementBuilder statementParser = new XMLStatementBuilder(configuration, builderAssistant, context, requiredDatabaseId);
      try {
        // 解析
        statementParser.parseStatementNode();
      } catch (IncompleteElementException e) {
        // 解析异常则放入未完成解析列表中，后续重试
        configuration.addIncompleteStatement(statementParser);
      }
    }
  }
```

解析过程中出现异常或者解析过程中发现依赖未解析的mapper，那么会在初次解析完成后重试把未解析的继续解析完成

重试未完成的resultMap

```java
  private void parsePendingResultMaps() {
    Collection<ResultMapResolver> incompleteResultMaps = configuration.getIncompleteResultMaps();
    synchronized (incompleteResultMaps) {
      Iterator<ResultMapResolver> iter = incompleteResultMaps.iterator();
      while (iter.hasNext()) {
        try {
          iter.next().resolve();
          iter.remove();
        } catch (IncompleteElementException e) {
          // ResultMap is still missing a resource...
        }
      }
    }
  }
```

重试未完成的二级缓存引用

```java
  private void parsePendingCacheRefs() {
    Collection<CacheRefResolver> incompleteCacheRefs = configuration.getIncompleteCacheRefs();
    synchronized (incompleteCacheRefs) {
      Iterator<CacheRefResolver> iter = incompleteCacheRefs.iterator();
      while (iter.hasNext()) {
        try {
          iter.next().resolveCacheRef();
          iter.remove();
        } catch (IncompleteElementException e) {
          // Cache ref is still missing a resource...
        }
      }
    }
  }
```

重试未完成的sql语句

```java
  private void parsePendingStatements() {
    Collection<XMLStatementBuilder> incompleteStatements = configuration.getIncompleteStatements();
    synchronized (incompleteStatements) {
      Iterator<XMLStatementBuilder> iter = incompleteStatements.iterator();
      while (iter.hasNext()) {
        try {
          iter.next().parseStatementNode();
          iter.remove();
        } catch (IncompleteElementException e) {
          // Statement is still missing a resource...
        }
      }
    }
  }
```

至此，mapper的解析就完成了。