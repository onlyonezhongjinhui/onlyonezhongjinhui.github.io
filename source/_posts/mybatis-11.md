---
title: Mybatis（十一）：通用CRUD
date: 2021-11-01 11:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 通用CRUD
tags: Mybatis
categories:
- [Mybatis]
---
## 通用CRUD

对于简单的应用来说，大多数数据库操作都是对单表CRUD。使用Mybatis的时候一般通过声明Mapper接口来操作数据库，大量的Mapper声明带来了大量重复工作量，当然可以通过Mybatis插件生成Mapper接口减少工作量，但是对于这种重复的东西，通过封装可能效果更好。这就是通用CRUD的需求来源。

### 实现方式一

*通过Mybatis的Provider特性加上反射构建通用Mapper接口。*

```java
@Mapper
public interface BaseMapper<T> {

    @InsertProvider(type = BaseSqlProvider.class, method = "buildInsertSql")
    void insert(T entity);

    @UpdateProvider(type = BaseSqlProvider.class, method = "buildUpdateSql")
    void updateById(T entity);

    @DeleteProvider(type = BaseSqlProvider.class, method = "buildDeleteSql")
    void deleteById(T entity);

    @SelectProvider(type = BaseSqlProvider.class, method = "buildSelectSql")
    T selectById(T entity);

}
```

Provider只需指定type、method两个属性即可。type是Provider的类，method是返回Sql语句的方法，注意参数和被注解的方法保持一致，方法的返回值是String类型。

通过反射可根据传入的参数类型生成不同的SQL语句。

```java
public class BaseSqlProvider {
    @SneakyThrows
    public static <T> String buildInsertSql(T entity) {
        Class<?> clazz = entity.getClass();
        StringBuilder sql = new StringBuilder("insert into ");
        TableName tableNameAnnotation = clazz.getAnnotation(TableName.class);
        if (tableNameAnnotation != null) {
            sql.append(tableNameAnnotation.value());
        } else {
            sql.append(clazz.getSimpleName());
        }
        sql.append(" (");
        Field[] fields = clazz.getDeclaredFields();
        if (fields.length == 0) {
            throw new IllegalStateException("实体类不存在属性");
        }
        String columnNames = Arrays.stream(fields).map(e -> humpToLine(e.getName())).collect(Collectors.joining(","));
        sql.append(columnNames);
        sql.append(") values (");
        for (Field field : fields) {
            if (field.getType().isAssignableFrom(String.class)) {
                sql.append("'");
                field.setAccessible(true);
                sql.append(field.get(entity));
                sql.append("'");
            } else {
                field.setAccessible(true);
                sql.append(field.get(entity));
            }
            sql.append(",");
        }
        sql.deleteCharAt(sql.length() - 1);
        sql.append(")");
        return sql.toString();
    }

    @SneakyThrows
    public static <T> String buildUpdateSql(T entity) {
        Class<?> clazz = entity.getClass();
        StringBuilder sql = new StringBuilder("update ");
        TableName tableNameAnnotation = clazz.getAnnotation(TableName.class);
        if (tableNameAnnotation != null) {
            sql.append(tableNameAnnotation.value());
        } else {
            sql.append(clazz.getSimpleName());
        }
        sql.append(" set ");
        Field[] fields = clazz.getDeclaredFields();
        if (fields.length == 0) {
            throw new IllegalStateException("实体类不存在属性");
        }

        Field idField = null;
        TableId tableId = null;
        for (Field field : fields) {
            TableId tableIdAnnotation = field.getAnnotation(TableId.class);
            if (tableIdAnnotation != null) {
                tableId = tableIdAnnotation;
                idField = field;
            }
            TableField tableFieldAnnotation = field.getAnnotation(TableField.class);
            if (tableFieldAnnotation != null) {
                sql.append(tableFieldAnnotation.value());
            } else {
                sql.append(humpToLine(field.getName()));
            }

            sql.append(" = ");
            if (field.getType().isAssignableFrom(String.class)) {
                sql.append("'");
                field.setAccessible(true);
                sql.append(field.get(entity));
                sql.append("'");
            } else {
                field.setAccessible(true);
                sql.append(field.get(entity));
            }
            sql.append(",");
        }
        sql.deleteCharAt(sql.length() - 1);
        sql.append(" where ");

        if (tableId == null) {
            throw new IllegalStateException("缺少主键");
        }
        if (tableId.value().equals("")) {
            sql.append(humpToLine(idField.getName()));
        } else {
            sql.append(tableId.value());
        }
        sql.append(" = ");
        if (idField.getType().isAssignableFrom(String.class)) {
            sql.append("'");
            idField.setAccessible(true);
            sql.append(idField.get(entity));
            sql.append("'");
        } else {
            idField.setAccessible(true);
            sql.append(idField.get(entity));
        }

        return sql.toString();
    }

    @SneakyThrows
    public static <T> String buildDeleteSql(T entity) {
        Class<?> clazz = entity.getClass();
        StringBuilder sql = new StringBuilder("delete from ");
        TableName tableNameAnnotation = clazz.getAnnotation(TableName.class);
        if (tableNameAnnotation != null) {
            sql.append(tableNameAnnotation.value());
        } else {
            sql.append(clazz.getSimpleName());
        }
        Field[] fields = clazz.getDeclaredFields();
        if (fields.length == 0) {
            throw new IllegalStateException("实体类不存在属性");
        }
        sql.append(" where ");
        for (Field field : fields) {
            TableId tableIdAnnotation = field.getAnnotation(TableId.class);
            if (tableIdAnnotation != null) {
                if (tableIdAnnotation.value().equals("")) {
                    sql.append(humpToLine(field.getName()));
                } else {
                    sql.append(tableIdAnnotation.value());
                }
                sql.append(" = ");
                if (field.getType().isAssignableFrom(String.class)) {
                    sql.append("'");
                    field.setAccessible(true);
                    sql.append(field.get(entity));
                    sql.append("'");
                } else {
                    field.setAccessible(true);
                    sql.append(field.get(entity));
                }
                break;
            }
        }

        return sql.toString();
    }

    @SneakyThrows
    public static <T> String buildSelectSql(T entity) {
        Class<?> clazz = entity.getClass();
        StringBuilder sql = new StringBuilder("select * from ");
        TableName tableNameAnnotation = clazz.getAnnotation(TableName.class);
        if (tableNameAnnotation != null) {
            sql.append(tableNameAnnotation.value());
        } else {
            sql.append(clazz.getSimpleName());
        }
        Field[] fields = clazz.getDeclaredFields();
        sql.append(" where ");
        for (Field field : fields) {
            TableId tableIdAnnotation = field.getAnnotation(TableId.class);
            if (tableIdAnnotation != null) {
                if (tableIdAnnotation.value().equals("")) {
                    sql.append(humpToLine(field.getName()));
                } else {
                    sql.append(tableIdAnnotation.value());
                }
                sql.append(" = ");
                if (field.getType().isAssignableFrom(String.class)) {
                    sql.append("'");
                    field.setAccessible(true);
                    sql.append(field.get(entity));
                    sql.append("'");
                } else {
                    field.setAccessible(true);
                    sql.append(field.get(entity));
                }
                break;
            }
        }
        return sql.toString();
    }

    /**
     * 驼峰转下划线
     */
    public static String humpToLine(String str) {
        return str.replaceAll("[A-Z]", "_$0").toLowerCase();
    }

    private static final Pattern humpPattern = Pattern.compile("[A-Z]");
}
```

业务的Mapper接口只要继承它，就可以获得单表的CRUD方法，相当方便。

```java
@Mapper
public interface UserMapper extends BaseMapper<User> {

}
```

*虽然这种方式可以实现动态CRUD，但是缺点很明显，每次操作数据库都重新生成SQL语句，性能差！*

代码地址：https://github.com/onlyonezhongjinhui/mybatis-learning/tree/main/dynamic_crud

### 实现方式二

*通过扩展Mybatis，在Mybatis启动的过程中注入动态CRUD，也就是给每个Mapper接口添加CRUD的MapperStatement。这种方式很好的解决了方式一的性能问题*

代码地址：https://github.com/onlyonezhongjinhui/mybatis-learning/tree/main/dynamic_crud_x



