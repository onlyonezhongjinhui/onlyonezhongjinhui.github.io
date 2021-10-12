---
title: Mybatis（十）：枚举转换器
date: 2021-10-12 12:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 枚举转换器
tags: Mybatis
categories:
- [Mybatis]
---
## 枚举转换器

Mybatis在做数据库表和实体转换的时候，需要使用类型转换器TypeHandler。当实体类中使用了枚举的时候，需要枚举类型转换器，Mybatis自带了两个枚举的转换器，分别是EnumOrdinalTypeHandler、EnumTypeHandler。

```java
public class EnumOrdinalTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

  private final Class<E> type;
  private final E[] enums;

  public EnumOrdinalTypeHandler(Class<E> type) {
    if (type == null) {
      throw new IllegalArgumentException("Type argument cannot be null");
    }
    this.type = type;
    this.enums = type.getEnumConstants();
    if (this.enums == null) {
      throw new IllegalArgumentException(type.getSimpleName() + " does not represent an enum type.");
    }
  }

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    ps.setInt(i, parameter.ordinal());
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    int ordinal = rs.getInt(columnName);
    if (ordinal == 0 && rs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  @Override
  public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    int ordinal = rs.getInt(columnIndex);
    if (ordinal == 0 && rs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  @Override
  public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    int ordinal = cs.getInt(columnIndex);
    if (ordinal == 0 && cs.wasNull()) {
      return null;
    }
    return toOrdinalEnum(ordinal);
  }

  private E toOrdinalEnum(int ordinal) {
    try {
      return enums[ordinal];
    } catch (Exception ex) {
      throw new IllegalArgumentException("Cannot convert " + ordinal + " to " + type.getSimpleName() + " by ordinal value.", ex);
    }
  }
}
```

```java
public class EnumTypeHandler<E extends Enum<E>> extends BaseTypeHandler<E> {

  private final Class<E> type;

  public EnumTypeHandler(Class<E> type) {
    if (type == null) {
      throw new IllegalArgumentException("Type argument cannot be null");
    }
    this.type = type;
  }

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, E parameter, JdbcType jdbcType) throws SQLException {
    if (jdbcType == null) {
      ps.setString(i, parameter.name());
    } else {
      ps.setObject(i, parameter.name(), jdbcType.TYPE_CODE); // see r3589
    }
  }

  @Override
  public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
    String s = rs.getString(columnName);
    return s == null ? null : Enum.valueOf(type, s);
  }

  @Override
  public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    String s = rs.getString(columnIndex);
    return s == null ? null : Enum.valueOf(type, s);
  }

  @Override
  public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    String s = cs.getString(columnIndex);
    return s == null ? null : Enum.valueOf(type, s);
  }
}
```

从代码可以看出，EnumOrdinalTypeHandler是使用枚举的ordinal属性进行转换，EnumTypeHandler是使用枚举的name属性进行转换。然而平常我们的枚举，值有多种类型，也会附带一些描述属性。因此这两个转化器都不能够满足我们的使用要求，这个时候就需要自定义枚举转换器了。

## 自定义枚举转换器

自定义转换器继承BaseTypeHandler，实现必须重写的4个方法

```java
@Override
public void setNonNullParameter(PreparedStatement ps, int i, IEnum<?> parameter, JdbcType jdbcType) throws SQLException {
    // javaType转换为jdbcType
}

@Override
public IEnum<?> getNullableResult(ResultSet rs, String columnName) throws SQLException {
    // 根据列名获取值时候调用
    // jdbcType转换为javaType
    return null;
}

@Override
public IEnum<?> getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
    // 根据下标获取值时候调用
    // jdbcType转换为javaType
    return null;
}

@Override
public IEnum<?> getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
    // 存储过程通过下标获取值时候调用
    // jdbcType转换为javaType
    return null;
}
```

项目中可定义统一枚举接口方便统一处理

```java
public interface IEnum<T> {

    T getValue();

    String getDescription();

}
```

创建IEnumTypeHandler枚举转换器统一转换实现了IEnum接口的枚举

```java
public class IEnumTypeHandler<E extends IEnum<?>> extends BaseTypeHandler<IEnum<?>> {
    private static final ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
    private final Class<E> type;
    private final Invoker invoker;

    public IEnumTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
        MetaClass metaClass = MetaClass.forClass(type, reflectorFactory);
        // TODO 这个地方写死了，可以写活，比如通过注解标记属性
        String name = "value";
        this.invoker = metaClass.getGetInvoker(name);
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, IEnum parameter, JdbcType jdbcType) throws SQLException {
        if (jdbcType == null) {
            ps.setObject(i, parameter.getValue());
        } else {
            ps.setObject(i, parameter.getValue(), jdbcType.TYPE_CODE);
        }
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        if (null == rs.getObject(columnName) && rs.wasNull()) {
            return null;
        }
        return this.valueOf(this.type, rs.getObject(columnName));
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        if (null == rs.getObject(columnIndex) && rs.wasNull()) {
            return null;
        }
        return this.valueOf(this.type, rs.getObject(columnIndex));
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        if (null == cs.getObject(columnIndex) && cs.wasNull()) {
            return null;
        }
        return this.valueOf(this.type, cs.getObject(columnIndex));
    }

    private E valueOf(Class<E> enumClass, Object value) {
        E[] es = enumClass.getEnumConstants();
        return Arrays.stream(es).filter((e) -> value.equals(getValue(e))).findAny().orElse(null);
    }

    private Object getValue(Object object) {
        try {
            return invoker.invoke(object, new Object[0]);
        } catch (ReflectiveOperationException e) {
            e.printStackTrace();
        }
        return null;
    }

}
```

让枚举转换器生效，可通过以下方式

1、直接在mybatis配置文件中进行全局配置（推荐使用）

```xml
<typeHandler handler="com.demo.handler.IEnumTypeHandler" javaType="com.demo.enums.Nation"/>
<typeHandler handler="com.demo.handler.IEnumTypeHandler" javaType="com.demo.enums.Education"/>
```

2、在mapper配置文件中进行局部配置（相对麻烦）

```xml
<resultMap id="BaseResultMap" type="com.demo.entity.User">
    <id column="id" property="id"/>
    <result column="name" property="name"/>
    <result column="age" property="age"/>
    <result column="gender" property="gender"/>
    <result column="nation" property="nation" typeHandler="com.demo.handler.IEnumTypeHandler"/>
    <result column="education" property="education" typeHandler="com.demo.handler.IEnumTypeHandler"/>
</resultMap>
```

```xml
<insert id="insert">
    insert into t_user (id, `name`, age, gender, nation, education)
    values (#{id}, #{name}, #{age}, #{gender},
    #{nation,typeHandler=com.demo.handler.IEnumTypeHandler,javaType=com.demo.enums.Nation},
    #{education,typeHandler=com.demo.handler.IEnumTypeHandler,javaType=com.demo.enums.Education})
</insert>
```

```xml
<update id="updateById">
    update t_user
    <set>
        <if test="name != null">
            `name` = #{name},
        </if>
        <if test="age != null">
            age = #{age},
        </if>
        <if test="gender != null">
            gender = #{gender},
        </if>
        <if test="nation != null">
            nation = #{nation,typeHandler=com.demo.handler.IEnumTypeHandler,javaType=com.demo.enums.Nation},
        </if>
        <if test="education != null">
            education = #{education,typeHandler=com.demo.handler.IEnumTypeHandler,javaType=com.demo.enums.Education}
        </if>
    </set>
    where id = #{id}
</update>
```

## 优化点

上面自定义枚举转换器只能处理实现IEnum接口的枚举，这里可以进行优化。增加通用性处理。

增加标识枚举转换属性注解

```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.ANNOTATION_TYPE})
public @interface EnumValue {

}
```

增加通用枚举转换器

```java
public class CommonEnumTypeHandler<E extends Enum<?>> extends BaseTypeHandler<Enum<?>> {
    private static final ReflectorFactory reflectorFactory = new DefaultReflectorFactory();
    private final Class<E> type;
    private final Invoker invoker;

    public CommonEnumTypeHandler(Class<E> type) {
        if (type == null) {
            throw new IllegalArgumentException("Type argument cannot be null");
        }
        this.type = type;
        MetaClass metaClass = MetaClass.forClass(type, reflectorFactory);
        String name = "value";
        if (!IEnum.class.isAssignableFrom(type)) {
            name = findEnumValueFieldName(this.type).orElseThrow(() -> new IllegalArgumentException(String.format("Could not find @EnumValue in Class: %s.", this.type.getName())));
        }
        this.invoker = metaClass.getGetInvoker(name);
    }

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Enum<?> parameter, JdbcType jdbcType)
            throws SQLException {
        if (jdbcType == null) {
            ps.setObject(i, this.getValue(parameter));
        } else {
            // see r3589
            ps.setObject(i, this.getValue(parameter), jdbcType.TYPE_CODE);
        }
    }

    @Override
    public E getNullableResult(ResultSet rs, String columnName) throws SQLException {
        if (null == rs.getObject(columnName) && rs.wasNull()) {
            return null;
        }
        return this.valueOf(this.type, rs.getObject(columnName));
    }

    @Override
    public E getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        if (null == rs.getObject(columnIndex) && rs.wasNull()) {
            return null;
        }
        return this.valueOf(this.type, rs.getObject(columnIndex));
    }

    @Override
    public E getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        if (null == cs.getObject(columnIndex) && cs.wasNull()) {
            return null;
        }
        return this.valueOf(this.type, cs.getObject(columnIndex));
    }

    private E valueOf(Class<E> enumClass, Object value) {
        E[] es = enumClass.getEnumConstants();
        return Arrays.stream(es).filter((e) -> value.equals(getValue(e))).findAny().orElse(null);
    }

    private Object getValue(Object object) {
        try {
            return invoker.invoke(object, new Object[0]);
        } catch (ReflectiveOperationException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * 查找标记标记EnumValue字段
     *
     * @param clazz class
     */
    public static Optional<String> findEnumValueFieldName(Class<?> clazz) {
        if (clazz != null && clazz.isEnum()) {
            Optional<Field> optional = Arrays.stream(clazz.getDeclaredFields())
                    .filter(field -> field.isAnnotationPresent(EnumValue.class))
                    .findFirst();
            return Optional.of(optional.map(Field::getName).orElseThrow(() -> new IllegalArgumentException("cannot find enum value")));
        }
        return Optional.empty();
    }

}
```

重点代码就是这里的判断

```java
String name = "value";
if (!IEnum.class.isAssignableFrom(type)) {
    name = findEnumValueFieldName(this.type).orElseThrow(() -> new IllegalArgumentException(String.format("Could not find @EnumValue in Class: %s.", this.type.getName())));
}
```

当不是IEnum的子类的时候，通过@EnumValue查找枚举的转换属性，然后进行转换

最后就配置让其生效即可

```xml
<typeHandler handler="com.demo.handler.CommonEnumTypeHandler" javaType="com.demo.enums.Nation"/>
```

## 代码路径

https://github.com/onlyonezhongjinhui/mybatis-learning/tree/main/customer-enum-type-handler

