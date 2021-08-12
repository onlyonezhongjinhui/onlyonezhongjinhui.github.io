---
title: Mybatis（二）：Configuration配置详细解析
date: 2021-08-10 15:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Configuration配置详细解析
tags: Mybatis
categories:
- [Mybatis]
---
## Configuration配置详细解析

### Mybatis的一般使用过程

```java
    final String resource = "org/apache/ibatis/builder/MapperConfig.xml";
    final Reader reader = Resources.getResourceAsReader(resource);
    sqlMapper = new SqlSessionFactoryBuilder().build(reader);
    SqlSession session = sqlMapper.openSession()
    Integer count = session.selectOne("org.apache.ibatis.domain.blog.mappers.BlogMapper.selectCountOfPosts");

```

如上代码所示，这是Mybatis的一般使用步骤

1. 读取配置文件流
2. 通过XMLConfigBuilder解析XML文件并生成Configuration对象
3. SqlSessionFactoryBuilder根据Configuration生成一个SqlSessionFactory,默认是DefaultSqlSessionFactory
4. 通过DefaultSqlSessionFactory生成SqlSession
5. SqlSession通过statementId匹配到相应的SQL语句进行增删查改操作

### Configuration的生成

Configuration是Mybatis存储所有配置的类。

Configuration是由XMLConfigBuilder类创建并填充的。XMLConfigBuilder类构造函数有多个，可传递配置文件流、环境变量、额外属性配置。创建XMLConfigBuilder的时候就创建了一个Configuration对象，用来储存解析出来的全部配置

```java
  public XMLConfigBuilder(Reader reader) {
    this(reader, null, null);
  }

  public XMLConfigBuilder(Reader reader, String environment) {
    this(reader, environment, null);
  }

  public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
  }

  public XMLConfigBuilder(InputStream inputStream) {
    this(inputStream, null, null);
  }

  public XMLConfigBuilder(InputStream inputStream, String environment) {
    this(inputStream, environment, null);
  }

  public XMLConfigBuilder(InputStream inputStream, String environment, Properties props) {
    this(new XPathParser(inputStream, true, props, new XMLMapperEntityResolver()), environment, props);
  }

  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
```

解析是通过XMLConfigBuilder的parse方法

```java
  public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));
    return configuration;
  }

  private void parseConfiguration(XNode root) {
    try {
      // issue #117 read properties first
      // 属性配置
      propertiesElement(root.evalNode("properties"));
      // setting配置
      Properties settings = settingsAsProperties(root.evalNode("settings"));
      loadCustomVfs(settings);
      loadCustomLogImpl(settings);
      // 别名注册
      typeAliasesElement(root.evalNode("typeAliases"));
      // 插件
      pluginElement(root.evalNode("plugins"));
      // 对象工厂
      objectFactoryElement(root.evalNode("objectFactory"));
      // 对象包装工厂
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      // 反射工厂
      reflectorFactoryElement(root.evalNode("reflectorFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      // 环境
      environmentsElement(root.evalNode("environments"));
      // 数据库提供商
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      // 类型转换器
      typeHandlerElement(root.evalNode("typeHandlers"));
      // mapper
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

这就完全对应了配置文件XML中的configuration标签，一个个标签顺序进行解析。下面是一个包含全部节点的configuration配置文件样例

```xml
<!DOCTYPE configuration
    PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
    "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

  <properties resource="org/apache/ibatis/builder/jdbc.properties">
    <property name="prop1" value="aaaa"/>
    <property name="jdbcTypeForNull" value="NULL" />
  </properties>

  <settings>
    <setting name="autoMappingBehavior" value="NONE"/>
    <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
    <setting name="cacheEnabled" value="false"/>
    <setting name="proxyFactory" value="CGLIB"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="aggressiveLazyLoading" value="true"/>
    <setting name="multipleResultSetsEnabled" value="false"/>
    <setting name="useColumnLabel" value="false"/>
    <setting name="useGeneratedKeys" value="true"/>
    <setting name="defaultExecutorType" value="BATCH"/>
    <setting name="defaultStatementTimeout" value="10"/>
    <setting name="defaultFetchSize" value="100"/>
    <setting name="defaultResultSetType" value="SCROLL_INSENSITIVE"/>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
    <setting name="safeRowBoundsEnabled" value="true"/>
    <setting name="localCacheScope" value="STATEMENT"/>
    <setting name="jdbcTypeForNull" value="${jdbcTypeForNull}"/>
    <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString,xxx"/>
    <setting name="safeResultHandlerEnabled" value="false"/>
    <setting name="defaultScriptingLanguage" value="org.apache.ibatis.scripting.defaults.RawLanguageDriver"/>
    <setting name="callSettersOnNulls" value="true"/>
    <setting name="logPrefix" value="mybatis_"/>
    <setting name="logImpl" value="SLF4J"/>
    <setting name="vfsImpl" value="org.apache.ibatis.io.JBoss6VFS"/>
    <setting name="configurationFactory" value="java.lang.String"/>
    <setting name="defaultEnumTypeHandler" value="org.apache.ibatis.type.EnumOrdinalTypeHandler"/>
    <setting name="shrinkWhitespacesInSql" value="true"/>
    <setting name="defaultSqlProviderType" value="org.apache.ibatis.builder.XmlConfigBuilderTest$MySqlProvider"/>
  </settings>

  <typeAliases>
    <typeAlias alias="BlogAuthor" type="org.apache.ibatis.domain.blog.Author"/>
    <typeAlias type="org.apache.ibatis.domain.blog.Blog"/>
    <typeAlias type="org.apache.ibatis.domain.blog.Post"/>
    <package name="org.apache.ibatis.domain.jpetstore"/>
  </typeAliases>

  <typeHandlers>
    <typeHandler javaType="String" handler="org.apache.ibatis.builder.CustomStringTypeHandler"/>
    <typeHandler javaType="String" jdbcType="VARCHAR" handler="org.apache.ibatis.builder.CustomStringTypeHandler"/>
    <typeHandler handler="org.apache.ibatis.builder.CustomLongTypeHandler"/>
    <package name="org.apache.ibatis.builder.typehandler"/>
  </typeHandlers>

  <objectFactory type="org.apache.ibatis.builder.ExampleObjectFactory">
    <property name="objectFactoryProperty" value="100"/>
  </objectFactory>

  <objectWrapperFactory type="org.apache.ibatis.builder.CustomObjectWrapperFactory" />

  <reflectorFactory type="org.apache.ibatis.builder.CustomReflectorFactory"/>

  <plugins>
    <plugin interceptor="org.apache.ibatis.builder.ExamplePlugin">
      <property name="pluginProperty" value="100"/>
    </plugin>
  </plugins>

  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC">
        <property name="" value=""/>
      </transactionManager>
      <dataSource type="UNPOOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>

  <databaseIdProvider type="DB_VENDOR">
    <property name="Apache Derby" value="derby"/>
  </databaseIdProvider>

  <mappers>
    <mapper resource="org/apache/ibatis/builder/BlogMapper.xml"/>
    <mapper url="file:./src/test/java/org/apache/ibatis/builder/NestedBlogMapper.xml"/>
    <mapper class="org.apache.ibatis.builder.CachedAuthorMapper"/>
    <package name="org.apache.ibatis.builder.mapper"/>
  </mappers>

</configuration>

```

### 节点解析详细说明

#### properties

```xml
  <properties resource="xxxxx"></properties>

  <properties url="xxxxx"></properties>

  <properties resource="org/apache/ibatis/builder/jdbc.properties">
    <property name="prop1" value="aaaa"/>
    <property name="jdbcTypeForNull" value="NULL" />
  </properties>
```

properties支持三种方式配置，称为resource、url、直接配置

```java
  private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      // 子节点配置
      Properties defaults = context.getChildrenAsProperties();
      // resource
      String resource = context.getStringAttribute("resource");
      // url
      String url = context.getStringAttribute("url");
      // 不支持同时使用resource、url方式配置
      if (resource != null && url != null) {
        throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
      }
      if (resource != null) {
        // resource方式读取
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        // url方式读取
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      // 已有属性
      Properties vars = configuration.getVariables();
      if (vars != null) {
        // 如果非空则加入已有的属性中
        defaults.putAll(vars);
      }
      parser.setVariables(defaults);
      // 保存到configuration的variables属性中
      configuration.setVariables(defaults);
    }
  }
```

从源码可以看出，resource和url不能同时配置，配置优先级由高到低为

Configuration -> resource/url -> 直接配置

#### settings

```xml
  <settings>
    <setting name="autoMappingBehavior" value="NONE"/>
    <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
    <setting name="cacheEnabled" value="false"/>
    <setting name="proxyFactory" value="CGLIB"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="aggressiveLazyLoading" value="true"/>
    <setting name="multipleResultSetsEnabled" value="false"/>
    <setting name="useColumnLabel" value="false"/>
    <setting name="useGeneratedKeys" value="true"/>
    <setting name="defaultExecutorType" value="BATCH"/>
    <setting name="defaultStatementTimeout" value="10"/>
    <setting name="defaultFetchSize" value="100"/>
    <setting name="defaultResultSetType" value="SCROLL_INSENSITIVE"/>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
    <setting name="safeRowBoundsEnabled" value="true"/>
    <setting name="localCacheScope" value="STATEMENT"/>
    <setting name="jdbcTypeForNull" value="${jdbcTypeForNull}"/>
    <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString,xxx"/>
    <setting name="safeResultHandlerEnabled" value="false"/>
    <setting name="defaultScriptingLanguage" value="org.apache.ibatis.scripting.defaults.RawLanguageDriver"/>
    <setting name="callSettersOnNulls" value="true"/>
    <setting name="logPrefix" value="mybatis_"/>
    <setting name="logImpl" value="SLF4J"/>
    <setting name="vfsImpl" value="org.apache.ibatis.io.JBoss6VFS"/>
    <setting name="configurationFactory" value="java.lang.String"/>
    <setting name="defaultEnumTypeHandler" value="org.apache.ibatis.type.EnumOrdinalTypeHandler"/>
    <setting name="shrinkWhitespacesInSql" value="true"/>
    <setting name="defaultSqlProviderType" value="org.apache.ibatis.builder.XmlConfigBuilderTest$MySqlProvider"/>
  </settings>
```

setting节点直接对应Configuration里面的成员属性，读取后直接赋值到成员

```java
  private void settingsElement(Properties props) {
    configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
    configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
    configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
    configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
    configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
    configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), false));
    configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
    configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
    configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
    configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
    configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
    configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
    configuration.setDefaultResultSetType(resolveResultSetType(props.getProperty("defaultResultSetType")));
    configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
    configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
    configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
    configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
    configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
    configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
    configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
    configuration.setDefaultEnumTypeHandler(resolveClass(props.getProperty("defaultEnumTypeHandler")));
    configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
    configuration.setUseActualParamName(booleanValueOf(props.getProperty("useActualParamName"), true));
    configuration.setReturnInstanceForEmptyRow(booleanValueOf(props.getProperty("returnInstanceForEmptyRow"), false));
    configuration.setLogPrefix(props.getProperty("logPrefix"));
    configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
    configuration.setShrinkWhitespacesInSql(booleanValueOf(props.getProperty("shrinkWhitespacesInSql"), false));
    configuration.setDefaultSqlProviderType(resolveClass(props.getProperty("defaultSqlProviderType")));
  }
```

所有配置以及默认值

![](/medias/assets/20210812095200.png)

#### typeAliases

typeAliases用来设置别名，有了别名就不用指定完整的全限定名。例如org.apache.ibatis.domain.blog.Author可以用简短的BlogAuthor表示，非常方便。别名的注册，最终是注册到configuration的成员属性上

```xml
<typeAliases>
    <typeAlias alias="BlogAuthor" type="org.apache.ibatis.domain.blog.Author"/>
    <typeAlias type="org.apache.ibatis.domain.blog.Blog"/>
    <typeAlias type="org.apache.ibatis.domain.blog.Post"/>
    <package name="org.apache.ibatis.domain.jpetstore"/>
  </typeAliases>
```

```
  protected final TypeAliasRegistry typeAliasRegistry = new TypeAliasRegistry();
```

TypeAliasRegistry就是用一个HashMap保存别名对应的类对象而已

```java
public class TypeAliasRegistry {

  private final Map<String, Class<?>> typeAliases = new HashMap<>();

  public TypeAliasRegistry() {
    registerAlias("string", String.class);

    registerAlias("byte", Byte.class);
    registerAlias("long", Long.class);
    registerAlias("short", Short.class);
    registerAlias("int", Integer.class);
    registerAlias("integer", Integer.class);
    registerAlias("double", Double.class);
    registerAlias("float", Float.class);
    registerAlias("boolean", Boolean.class);

    registerAlias("byte[]", Byte[].class);
    registerAlias("long[]", Long[].class);
    registerAlias("short[]", Short[].class);
    registerAlias("int[]", Integer[].class);
    registerAlias("integer[]", Integer[].class);
    registerAlias("double[]", Double[].class);
    registerAlias("float[]", Float[].class);
    registerAlias("boolean[]", Boolean[].class);

    registerAlias("_byte", byte.class);
    registerAlias("_long", long.class);
    registerAlias("_short", short.class);
    registerAlias("_int", int.class);
    registerAlias("_integer", int.class);
    registerAlias("_double", double.class);
    registerAlias("_float", float.class);
    registerAlias("_boolean", boolean.class);

    registerAlias("_byte[]", byte[].class);
    registerAlias("_long[]", long[].class);
    registerAlias("_short[]", short[].class);
    registerAlias("_int[]", int[].class);
    registerAlias("_integer[]", int[].class);
    registerAlias("_double[]", double[].class);
    registerAlias("_float[]", float[].class);
    registerAlias("_boolean[]", boolean[].class);

    registerAlias("date", Date.class);
    registerAlias("decimal", BigDecimal.class);
    registerAlias("bigdecimal", BigDecimal.class);
    registerAlias("biginteger", BigInteger.class);
    registerAlias("object", Object.class);

    registerAlias("date[]", Date[].class);
    registerAlias("decimal[]", BigDecimal[].class);
    registerAlias("bigdecimal[]", BigDecimal[].class);
    registerAlias("biginteger[]", BigInteger[].class);
    registerAlias("object[]", Object[].class);

    registerAlias("map", Map.class);
    registerAlias("hashmap", HashMap.class);
    registerAlias("list", List.class);
    registerAlias("arraylist", ArrayList.class);
    registerAlias("collection", Collection.class);
    registerAlias("iterator", Iterator.class);

    registerAlias("ResultSet", ResultSet.class);
  }

}
```

注册过程

```java
  private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          // 如果是包名，则把包中的所有类都注册
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {
          // 如果是类，则单独注册一个
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class<?> clazz = Resources.classForName(type);
            if (alias == null) {
              // 如果没有配置别名，则自动生成别名
              typeAliasRegistry.registerAlias(clazz);
            } else {
              // 直接按照别名进行注册
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
```

具体操作就委托给了TypeAliasRegistry的各个registryAlias方法

```java
  public void registerAliases(String packageName) {
    registerAliases(packageName, Object.class);
  }

  public void registerAliases(String packageName, Class<?> superType) {
    ResolverUtil<Class<?>> resolverUtil = new ResolverUtil<>();
    resolverUtil.find(new ResolverUtil.IsA(superType), packageName);
    Set<Class<? extends Class<?>>> typeSet = resolverUtil.getClasses();
    for (Class<?> type : typeSet) {
      // Ignore inner classes and interfaces (including package-info.java)
      // Skip also inner classes. See issue #6
      if (!type.isAnonymousClass() && !type.isInterface() && !type.isMemberClass()) {
        registerAlias(type);
      }
    }
  }

  public void registerAlias(Class<?> type) {
    // 加单类名当作初始别名
    String alias = type.getSimpleName();
    // 检查Alias注解
    Alias aliasAnnotation = type.getAnnotation(Alias.class);
    if (aliasAnnotation != null) {
      // 如果有配置Alias注解，则取value值覆盖初始别名
      alias = aliasAnnotation.value();
    }
    // 注册
    registerAlias(alias, type);
  }

  public void registerAlias(String alias, Class<?> value) {
    if (alias == null) {
      throw new TypeException("The parameter alias cannot be null");
    }
    // issue #748
    String key = alias.toLowerCase(Locale.ENGLISH);
    if (typeAliases.containsKey(key) && typeAliases.get(key) != null && !typeAliases.get(key).equals(value)) {
      throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + typeAliases.get(key).getName() + "'.");
    }
    typeAliases.put(key, value);
  }

  public void registerAlias(String alias, String value) {
    try {
      registerAlias(alias, Resources.classForName(value));
    } catch (ClassNotFoundException e) {
      throw new TypeException("Error registering type alias " + alias + " for " + value + ". Cause: " + e, e);
    }
  }
```

#### typeHandlers

类型转换器用来处理数据库字段和java属性之间的转换，默认已经注册了常用的类型转换器。最终保存在configuration中

```xml
  <typeHandlers>
    <typeHandler javaType="String" handler="org.apache.ibatis.builder.CustomStringTypeHandler"/>
    <typeHandler javaType="String" jdbcType="VARCHAR" handler="org.apache.ibatis.builder.CustomStringTypeHandler"/>
    <typeHandler handler="org.apache.ibatis.builder.CustomLongTypeHandler"/>
    <package name="org.apache.ibatis.builder.typehandler"/>
  </typeHandlers>
```

注册过程

```java
  protected final TypeHandlerRegistry typeHandlerRegistry = new TypeHandlerRegistry(this);
```

```java
public final class TypeHandlerRegistry {
  // jdbc type 对应的TypeHandler
  private final Map<JdbcType, TypeHandler<?>>  jdbcTypeHandlerMap = new EnumMap<>(JdbcType.class);
  // java type对应的jdbc TypeHandler
  private final Map<Type, Map<JdbcType, TypeHandler<?>>> typeHandlerMap = new ConcurrentHashMap<>();
  // 未知类型
  private final TypeHandler<Object> unknownTypeHandler;
  // 全部类型转换器
  private final Map<Class<?>, TypeHandler<?>> allTypeHandlersMap = new HashMap<>();
  // null对应应的转换器
  private static final Map<JdbcType, TypeHandler<?>> NULL_TYPE_HANDLER_MAP = Collections.emptyMap();
  // 默认枚举类型转换器
  private Class<? extends TypeHandler> defaultEnumTypeHandler = EnumTypeHandler.class;

  /**
   * The default constructor.
   */
  public TypeHandlerRegistry() {
    this(new Configuration());
  }

  /**
   * The constructor that pass the MyBatis configuration.
   *
   * @param configuration a MyBatis configuration
   * @since 3.5.4
   */
  public TypeHandlerRegistry(Configuration configuration) {
    this.unknownTypeHandler = new UnknownTypeHandler(configuration);

    register(Boolean.class, new BooleanTypeHandler());
    register(boolean.class, new BooleanTypeHandler());
    register(JdbcType.BOOLEAN, new BooleanTypeHandler());
    register(JdbcType.BIT, new BooleanTypeHandler());

    register(Byte.class, new ByteTypeHandler());
    register(byte.class, new ByteTypeHandler());
    register(JdbcType.TINYINT, new ByteTypeHandler());

    register(Short.class, new ShortTypeHandler());
    register(short.class, new ShortTypeHandler());
    register(JdbcType.SMALLINT, new ShortTypeHandler());

    register(Integer.class, new IntegerTypeHandler());
    register(int.class, new IntegerTypeHandler());
    register(JdbcType.INTEGER, new IntegerTypeHandler());

    register(Long.class, new LongTypeHandler());
    register(long.class, new LongTypeHandler());

    register(Float.class, new FloatTypeHandler());
    register(float.class, new FloatTypeHandler());
    register(JdbcType.FLOAT, new FloatTypeHandler());

    register(Double.class, new DoubleTypeHandler());
    register(double.class, new DoubleTypeHandler());
    register(JdbcType.DOUBLE, new DoubleTypeHandler());

    register(Reader.class, new ClobReaderTypeHandler());
    register(String.class, new StringTypeHandler());
    register(String.class, JdbcType.CHAR, new StringTypeHandler());
    register(String.class, JdbcType.CLOB, new ClobTypeHandler());
    register(String.class, JdbcType.VARCHAR, new StringTypeHandler());
    register(String.class, JdbcType.LONGVARCHAR, new StringTypeHandler());
    register(String.class, JdbcType.NVARCHAR, new NStringTypeHandler());
    register(String.class, JdbcType.NCHAR, new NStringTypeHandler());
    register(String.class, JdbcType.NCLOB, new NClobTypeHandler());
    register(JdbcType.CHAR, new StringTypeHandler());
    register(JdbcType.VARCHAR, new StringTypeHandler());
    register(JdbcType.CLOB, new ClobTypeHandler());
    register(JdbcType.LONGVARCHAR, new StringTypeHandler());
    register(JdbcType.NVARCHAR, new NStringTypeHandler());
    register(JdbcType.NCHAR, new NStringTypeHandler());
    register(JdbcType.NCLOB, new NClobTypeHandler());

    register(Object.class, JdbcType.ARRAY, new ArrayTypeHandler());
    register(JdbcType.ARRAY, new ArrayTypeHandler());

    register(BigInteger.class, new BigIntegerTypeHandler());
    register(JdbcType.BIGINT, new LongTypeHandler());

    register(BigDecimal.class, new BigDecimalTypeHandler());
    register(JdbcType.REAL, new BigDecimalTypeHandler());
    register(JdbcType.DECIMAL, new BigDecimalTypeHandler());
    register(JdbcType.NUMERIC, new BigDecimalTypeHandler());

    register(InputStream.class, new BlobInputStreamTypeHandler());
    register(Byte[].class, new ByteObjectArrayTypeHandler());
    register(Byte[].class, JdbcType.BLOB, new BlobByteObjectArrayTypeHandler());
    register(Byte[].class, JdbcType.LONGVARBINARY, new BlobByteObjectArrayTypeHandler());
    register(byte[].class, new ByteArrayTypeHandler());
    register(byte[].class, JdbcType.BLOB, new BlobTypeHandler());
    register(byte[].class, JdbcType.LONGVARBINARY, new BlobTypeHandler());
    register(JdbcType.LONGVARBINARY, new BlobTypeHandler());
    register(JdbcType.BLOB, new BlobTypeHandler());

    register(Object.class, unknownTypeHandler);
    register(Object.class, JdbcType.OTHER, unknownTypeHandler);
    register(JdbcType.OTHER, unknownTypeHandler);

    register(Date.class, new DateTypeHandler());
    register(Date.class, JdbcType.DATE, new DateOnlyTypeHandler());
    register(Date.class, JdbcType.TIME, new TimeOnlyTypeHandler());
    register(JdbcType.TIMESTAMP, new DateTypeHandler());
    register(JdbcType.DATE, new DateOnlyTypeHandler());
    register(JdbcType.TIME, new TimeOnlyTypeHandler());

    register(java.sql.Date.class, new SqlDateTypeHandler());
    register(java.sql.Time.class, new SqlTimeTypeHandler());
    register(java.sql.Timestamp.class, new SqlTimestampTypeHandler());

    register(String.class, JdbcType.SQLXML, new SqlxmlTypeHandler());

    register(Instant.class, new InstantTypeHandler());
    register(LocalDateTime.class, new LocalDateTimeTypeHandler());
    register(LocalDate.class, new LocalDateTypeHandler());
    register(LocalTime.class, new LocalTimeTypeHandler());
    register(OffsetDateTime.class, new OffsetDateTimeTypeHandler());
    register(OffsetTime.class, new OffsetTimeTypeHandler());
    register(ZonedDateTime.class, new ZonedDateTimeTypeHandler());
    register(Month.class, new MonthTypeHandler());
    register(Year.class, new YearTypeHandler());
    register(YearMonth.class, new YearMonthTypeHandler());
    register(JapaneseDate.class, new JapaneseDateTypeHandler());

    // issue #273
    register(Character.class, new CharacterTypeHandler());
    register(char.class, new CharacterTypeHandler());
  }
}
```

注册过程

```java
  private void typeHandlerElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          // 如果配置是包的话，把包下的全部类作为为typeHandler
          String typeHandlerPackage = child.getStringAttribute("name");
          typeHandlerRegistry.register(typeHandlerPackage);
        } else {
          // 如果配置的是单个，那就单个注册
          String javaTypeName = child.getStringAttribute("javaType");
          String jdbcTypeName = child.getStringAttribute("jdbcType");
          String handlerTypeName = child.getStringAttribute("handler");
          Class<?> javaTypeClass = resolveClass(javaTypeName);
          JdbcType jdbcType = resolveJdbcType(jdbcTypeName);
          Class<?> typeHandlerClass = resolveClass(handlerTypeName);
          if (javaTypeClass != null) {
            if (jdbcType == null) {
              typeHandlerRegistry.register(javaTypeClass, typeHandlerClass);
            } else {
              typeHandlerRegistry.register(javaTypeClass, jdbcType, typeHandlerClass);
            }
          } else {
            typeHandlerRegistry.register(typeHandlerClass);
          }
        }
      }
    }
  }
```

委托给TypeHandlerRegistry的各个register方法完成

```java
  public void register(JdbcType jdbcType, TypeHandler<?> handler) {
    jdbcTypeHandlerMap.put(jdbcType, handler);
  }

  //
  // REGISTER INSTANCE
  //

  // Only handler

  @SuppressWarnings("unchecked")
  public <T> void register(TypeHandler<T> typeHandler) {
    boolean mappedTypeFound = false;
    MappedTypes mappedTypes = typeHandler.getClass().getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
      for (Class<?> handledType : mappedTypes.value()) {
        register(handledType, typeHandler);
        mappedTypeFound = true;
      }
    }
    // @since 3.1.0 - try to auto-discover the mapped type
    if (!mappedTypeFound && typeHandler instanceof TypeReference) {
      try {
        TypeReference<T> typeReference = (TypeReference<T>) typeHandler;
        register(typeReference.getRawType(), typeHandler);
        mappedTypeFound = true;
      } catch (Throwable t) {
        // maybe users define the TypeReference with a different type and are not assignable, so just ignore it
      }
    }
    if (!mappedTypeFound) {
      register((Class<T>) null, typeHandler);
    }
  }

  // java type + handler

  public <T> void register(Class<T> javaType, TypeHandler<? extends T> typeHandler) {
    register((Type) javaType, typeHandler);
  }

  private <T> void register(Type javaType, TypeHandler<? extends T> typeHandler) {
    MappedJdbcTypes mappedJdbcTypes = typeHandler.getClass().getAnnotation(MappedJdbcTypes.class);
    if (mappedJdbcTypes != null) {
      for (JdbcType handledJdbcType : mappedJdbcTypes.value()) {
        register(javaType, handledJdbcType, typeHandler);
      }
      if (mappedJdbcTypes.includeNullJdbcType()) {
        register(javaType, null, typeHandler);
      }
    } else {
      register(javaType, null, typeHandler);
    }
  }

  public <T> void register(TypeReference<T> javaTypeReference, TypeHandler<? extends T> handler) {
    register(javaTypeReference.getRawType(), handler);
  }

  // java type + jdbc type + handler

  // Cast is required here
  @SuppressWarnings("cast")
  public <T> void register(Class<T> type, JdbcType jdbcType, TypeHandler<? extends T> handler) {
    register((Type) type, jdbcType, handler);
  }

  private void register(Type javaType, JdbcType jdbcType, TypeHandler<?> handler) {
    if (javaType != null) {
      Map<JdbcType, TypeHandler<?>> map = typeHandlerMap.get(javaType);
      if (map == null || map == NULL_TYPE_HANDLER_MAP) {
        map = new HashMap<>();
      }
      map.put(jdbcType, handler);
      typeHandlerMap.put(javaType, map);
    }
    allTypeHandlersMap.put(handler.getClass(), handler);
  }

  //
  // REGISTER CLASS
  //

  // Only handler type

  public void register(Class<?> typeHandlerClass) {
    boolean mappedTypeFound = false;
    MappedTypes mappedTypes = typeHandlerClass.getAnnotation(MappedTypes.class);
    if (mappedTypes != null) {
      for (Class<?> javaTypeClass : mappedTypes.value()) {
        register(javaTypeClass, typeHandlerClass);
        mappedTypeFound = true;
      }
    }
    if (!mappedTypeFound) {
      register(getInstance(null, typeHandlerClass));
    }
  }

  // java type + handler type

  public void register(String javaTypeClassName, String typeHandlerClassName) throws ClassNotFoundException {
    register(Resources.classForName(javaTypeClassName), Resources.classForName(typeHandlerClassName));
  }

  public void register(Class<?> javaTypeClass, Class<?> typeHandlerClass) {
    register(javaTypeClass, getInstance(javaTypeClass, typeHandlerClass));
  }

  // java type + jdbc type + handler type

  public void register(Class<?> javaTypeClass, JdbcType jdbcType, Class<?> typeHandlerClass) {
    register(javaTypeClass, jdbcType, getInstance(javaTypeClass, typeHandlerClass));
  }
```

Mybatis3.4.5默认支持了JSR-310（日期和时间API），全部类型处理器如下

![](/medias/assets/20210812101836.png)

#### objectFactory

每次创建结果对象的时候，Mybatis都会使用对象工厂来完成实例化工作。默认的对象工厂仅仅实例化目标类，通过无参构造方法或者通过指定参数的构造方法进行实例化。

```xml
  <objectFactory type="org.apache.ibatis.builder.ExampleObjectFactory">
    <property name="objectFactoryProperty" value="100"/>
  </objectFactory>
```

解析过程

```java
  private void objectFactoryElement(XNode context) throws Exception {
    if (context != null) {
      // 对象工厂类型
      String type = context.getStringAttribute("type");
      // 对象工厂属性
      Properties properties = context.getChildrenAsProperties();
      // 实例化对象工厂
      ObjectFactory factory = (ObjectFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      // 设置对象工厂属性
      factory.setProperties(properties);
      // 保存到configuration中
      configuration.setObjectFactory(factory);
    }
  }
```

#### objectWrapperFactory

objectWrapperFactory是创建包装类工厂

```xml
<objectWrapperFactory type="org.apache.ibatis.builder.CustomObjectWrapperFactory" />
```

解析过程

```java
  private void objectWrapperFactoryElement(XNode context) throws Exception {
    if (context != null) {
      // 类型
      String type = context.getStringAttribute("type");
      // 实例化
      ObjectWrapperFactory factory = (ObjectWrapperFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      // 保存到configuration中
      configuration.setObjectWrapperFactory(factory);
    }
  }
```

#### reflectorFactory

ReflectorFactory接口主要是为了实现对Reflector对象的创建和缓存，Mybatis为ReflectorFactory接口提供了一个默认实现类DefaultReflectorFactory，其中findForClass()方法实现会为指定的Class创建Reflector对象，并将Reflector 对象缓存到reflectorMap中

```xml
<reflectorFactory type="org.apache.ibatis.builder.CustomReflectorFactory"/>
```

解析过程

```java
  private void reflectorFactoryElement(XNode context) throws Exception {
    if (context != null) {
      // 类型
      String type = context.getStringAttribute("type");
      // 实例化
      ReflectorFactory factory = (ReflectorFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      // 保存到configuration中
      configuration.setReflectorFactory(factory);
    }
  }
```

#### plugins

MyBatis 允许你在映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

```java
Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
ParameterHandler (getParameterObject, setParameters)
ResultSetHandler (handleResultSets, handleOutputParameters)
StatementHandler (prepare, parameterize, batch, update, query)
```

插件保存在configuration的interceptorChain中

```java
protected final InterceptorChain interceptorChain = new InterceptorChain();
```

```java
public class InterceptorChain {

  private final List<Interceptor> interceptors = new ArrayList<>();

  public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }

  public void addInterceptor(Interceptor interceptor) {
    interceptors.add(interceptor);
  }

  public List<Interceptor> getInterceptors() {
    return Collections.unmodifiableList(interceptors);
  }

}
```

```xml
  <plugins>
    <plugin interceptor="org.apache.ibatis.builder.ExamplePlugin">
      <property name="pluginProperty" value="100"/>
    </plugin>
  </plugins>
```

注册过程

```java
  private void pluginElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        // 拦截器
        String interceptor = child.getStringAttribute("interceptor");
        // 拦截器属性
        Properties properties = child.getChildrenAsProperties();
        // 初始化拦截器
        Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).getDeclaredConstructor().newInstance();
        // 设置拦截器属性
        interceptorInstance.setProperties(properties);
        // 保存到configuration中
        configuration.addInterceptor(interceptorInstance);
      }
    }
  }
```

#### environments

MyBatis 可以配置成适应多种环境，这种机制有助于将 SQL 映射应用于多种数据库之中， 现实情况下有多种理由需要这么做。例如，开发、测试和生产环境需要有不同的配置；或者想在具有相同 Schema 的多个生产数据库中使用相同的 SQL 映射。还有许多类似的使用场景。尽管可以配置多个环境，但每个 SqlSessionFactory 实例只能选择一种环境。

```xml
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC">
        <property name="" value=""/>
      </transactionManager>
      <dataSource type="UNPOOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
  </environments>
```

解析过程

```java
  private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      if (environment == null) {
        environment = context.getStringAttribute("default");
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        if (isSpecifiedEnvironment(id)) {
          // 事务工厂
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));
          // 数据源工厂
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource"));
          // 创建数据源
          DataSource dataSource = dsFactory.getDataSource();
          // 创建环境对象
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          // 保存到configuration中
          configuration.setEnvironment(environmentBuilder.build());
        }
      }
    }
  }
```

解析事务工厂

```java
  private TransactionFactory transactionManagerElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      Properties props = context.getChildrenAsProperties();
      TransactionFactory factory = (TransactionFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a TransactionFactory.");
  }
```

解析数据源工厂

```java
  private DataSourceFactory dataSourceElement(XNode context) throws Exception {
    if (context != null) {
      String type = context.getStringAttribute("type");
      Properties props = context.getChildrenAsProperties();
      DataSourceFactory factory = (DataSourceFactory) resolveClass(type).getDeclaredConstructor().newInstance();
      factory.setProperties(props);
      return factory;
    }
    throw new BuilderException("Environment declaration requires a DataSourceFactory.");
  }
```

最终保存再Environment对象中

```java
public final class Environment {
  private final String id;
  private final TransactionFactory transactionFactory;
  private final DataSource dataSource;
}
```

#### databaseIdProvider

MyBatis 可以根据不同的数据库厂商执行不同的语句，这种多厂商的支持是基于映射语句中的 databaseId 属性。 MyBatis 会加载带有匹配当前数据库 databaseId 属性和所有不带 databaseId 属性的语句。 如果同时找到带有 databaseId 和不带 databaseId 的相同语句，则后者会被舍弃

```xml
  <databaseIdProvider type="DB_VENDOR">
    <property name="Apache Derby" value="derby"/>
  </databaseIdProvider>
```

解析过程

```java
  private void databaseIdProviderElement(XNode context) throws Exception {
    DatabaseIdProvider databaseIdProvider = null;
    if (context != null) {
      // 类型
      String type = context.getStringAttribute("type");
      // 保持向下兼容VENDOR
      // awful patch to keep backward compatibility
      if ("VENDOR".equals(type)) {
        type = "DB_VENDOR";
      }
      // 属性
      Properties properties = context.getChildrenAsProperties();
      // 实例化DatabaseIdProvider
      databaseIdProvider = (DatabaseIdProvider) resolveClass(type).getDeclaredConstructor().newInstance();
      // 设置属性
      databaseIdProvider.setProperties(properties);
    }
    Environment environment = configuration.getEnvironment();
    if (environment != null && databaseIdProvider != null) {
      String databaseId = databaseIdProvider.getDatabaseId(environment.getDataSource());
      // 保存到configuration中
      configuration.setDatabaseId(databaseId);
    }
  }
```

#### mappers

告诉Mybatis去哪里找mapper

```xml
  <mappers>
    <mapper resource="org/apache/ibatis/builder/BlogMapper.xml"/>
    <mapper url="file:./src/test/java/org/apache/ibatis/builder/NestedBlogMapper.xml"/>
    <mapper class="org.apache.ibatis.builder.CachedAuthorMapper"/>
    <package name="org.apache.ibatis.builder.mapper"/>
  </mappers>
```