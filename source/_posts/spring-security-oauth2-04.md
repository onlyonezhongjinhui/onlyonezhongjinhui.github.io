---
title: 从零搭建spring security oauth2认证授权体系（四）
date: 2021-06-19 16:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: 客户端信息和授权码持久化
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
## 客户端信息和授权码持久化

### 添加jdbc、mysql驱动依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jdbc</artifactId>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
        </dependency>
```

### 创建数据表

sql脚本从github源码中拷贝的[https://github.com/spring-projects/spring-security-oauth/blob/main/tests/annotation/jdbc/src/main/resources/schema.sql](https://)

稍微修改一下符合mysql的语法

```sql
drop database if exists user_center;
create database user_center character set utf8mb4 collate utf8mb4_unicode_ci;
use user_center;

create table oauth_client_details
(
    client_id               varchar(256) not null primary key,
    resource_ids            varchar(256) null,
    client_secret           varchar(256) not null,
    scope                   varchar(256) not null,
    authorized_grant_types  varchar(256) not null,
    web_server_redirect_uri varchar(256) null,
    authorities             varchar(256) null,
    access_token_validity   int          not null,
    refresh_token_validity  int          not null,
    additional_information  varchar(4096),
    autoapprove             varchar(256)
);

create table oauth_client_token
(
    authentication_id varchar(256) primary key,
    token_id          varchar(256),
    token             blob,
    user_name         varchar(256),
    client_id         varchar(256)
);

create table oauth_access_token
(
    authentication_id varchar(256) primary key,
    token_id          varchar(256),
    token             blob,
    user_name         varchar(256),
    client_id         varchar(256),
    authentication    blob,
    refresh_token     varchar(256),
    key token_id (token_id),
    key user_name (user_name),
    key client_id (client_id),
    key refresh_token (refresh_token)
);

create table oauth_refresh_token
(
    token_id       varchar(256) primary key,
    token          blob,
    authentication blob
);

create table oauth_code
(
    code           varchar(256) primary key,
    authentication blob
);

insert into oauth_client_details (client_id, resource_ids, client_secret, scope, authorized_grant_types,
                                  web_server_redirect_uri, authorities, access_token_validity, refresh_token_validity,
                                  additional_information, autoapprove)
values ('order', 'order', '{noop}secret', 'read,write',
        'authorization_code,client_credentials,password,implicit,refresh_token', 'http://www.baidu.com', 'ADMIN,USER', 2592000,
        15552000, null, true);
```

### 配置数据源

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/user_center?useUnicode=true&autoReconnect=true&useSSL=false&zeroDateTimeBehavior=convertToNull&serverTimezone=Asia/Shanghai
    username: root
    password:
    platform: mysql
    driver-class-name: com.mysql.cj.jdbc.Driver
    type: com.zaxxer.hikari.HikariDataSource
```

### 配置授权服务

```java
@Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.jdbc(dataSource);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        endpoints
                .authenticationManager(this.authenticationManager)
                .userDetailsService(this.userDetailsService)
                .tokenStore(tokenStore()) // 设置 jdbc TokenStore
                .authorizationCodeServices(authorizationCodeServices()); // 设置授权码服务
    }

    @Bean
    public TokenStore tokenStore() {
        return new JdbcTokenStore(dataSource);
    }

    @Bean
    public AuthorizationCodeServices authorizationCodeServices() {
        return new JdbcAuthorizationCodeServices(dataSource);
    }
```

### 置资源服务

```java
    @Bean
    public ResourceServerTokenServices tokenServices() {
        // 使用远程服务请求授权服务器校验token,必须指定校验token 的url、client_id，client_secret
        RemoteTokenServices tokenServices = new RemoteTokenServices();
        tokenServices.setCheckTokenEndpointUrl("http://localhost:8090/oauth/check_token");
        tokenServices.setClientId("order");
        tokenServices.setClientSecret("secret");
        return tokenServices;
    }
```

### 测试授权服务

#### 客户端模式

*注意：POST请求需要在请求头带上Authorization参数，参数格式固定为Base64({client_id}:{client_secret})。POST请求带参的Content-Type是application/x-www-form-urlencoded。这里使用的是IDEA的http client工具测试*

请求：

```
POST http://localhost:8090/oauth/token?grant_type=client_credentials
Accept: application/json
Authorization: Basic b3JkZXI6c2VjcmV0
```

响应：

```JSON
{
  "access_token": "2ad17022-7d2e-44b6-9fd4-57d17a2a9568",
  "token_type": "bearer",
  "expires_in": 2589707,
  "scope": "read write"
}
```

数据库中生成了相关记录

![](/medias/assets/20210618170013.png)

![](/medias/assets/20210618170205.png)

#### 密码模式

请求：

```
POST http://localhost:8090/oauth/token
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXI6c2VjcmV0

grant_type=password&username=admin&password=password
```

响应：

```JSON
{
  "access_token": "4e4285e5-2673-42eb-858d-9c299aaae9ce",
  "token_type": "bearer",
  "refresh_token": "eba78cf6-2ff4-49a6-a653-b72f3c95d6ae",
  "expires_in": 2589453,
  "scope": "read write"
}
```

#### 授权码模式

请求：

```
GET http://localhost:8090/oauth/authorize?client_id=order&response_type=code&grant_type=authorization_code&redirect_uri=http://www.baidu.com
```

响应：

```
https://www.baidu.com/?code=9gqy0y
```

使用响应连接中的code获取token

```
POST http://localhost:8090/oauth/token
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXI6c2VjcmV0

grant_type=authorization_code&client_id=order&client_secret=secret&redirect_uri=http://www.baidu.com&code=kY8L5A
```

响应：

```JSON
{
  "access_token": "4e4285e5-2673-42eb-858d-9c299aaae9ce",
  "token_type": "bearer",
  "refresh_token": "eba78cf6-2ff4-49a6-a653-b72f3c95d6ae",
  "expires_in": 2589355,
  "scope": "read write"
}
```

使用后授权码被删除

![](/medias/assets/20210618165609.png)

#### 简易模式

请求：

```
GET http://localhost:8090/oauth/authorize?client_id=order&response_type=token&grant_type=authorization_code&redirect_uri=http://www.baidu.com
```

响应：

```
https://www.baidu.com/#access_token=54c7973e-8ae0-4dee-85bb-e2f44c649fe8&token_type=bearer&expires_in=2591760&scope=read%20write
```

### 验证token

请求：

```
GET http://localhost:8090/oauth/check_token?token=7f779d38-0629-4df1-84e0-6fa6a3da64cd
```

响应：

```json
{
  "aud": [
    "order"
  ],
  "user_name": "admin",
  "scope": [
    "read",
    "write"
  ],
  "active": true,
  "exp": 1626598954,
  "authorities": [
    "ROLE_ADMIN",
    "ROLE_USER"
  ],
  "client_id": "order"
}
```

### 刷新token

请求：

```
POST http://localhost:8090/oauth/token
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXI6c2VjcmV0

grant_type=refresh_token&refresh_token=fab9886a-ef2d-4506-9501-d379103090ca
```

响应：

```JSON
{
  "access_token": "c8e0cf74-b5f2-46f4-9c8d-21fd4d74e36e",
  "token_type": "bearer",
  "refresh_token": "fab9886a-ef2d-4506-9501-d379103090ca",
  "expires_in": 2591999,
  "scope": "read write"
}
```

测试资源服务

不带token请求：

```
GET http://localhost:8092/api/order
```

响应：

```
{
  "error": "unauthorized",
  "error_description": "Full authentication is required to access this resource"
}

Response code: 401; Time: 40ms; Content length: 102 bytes
```

带token请求：

```
GET http://localhost:8092/api/order
Authorization: Bearer b8a2a416-4e67-4aeb-b471-00256c8a5779
```

响应：

```
get order

Response code: 200; Time: 139ms; Content length: 9 bytes
```

### 代码地址

[https://github.com/onlyonezhongjinhui/spring-security-ouath2-learning/tree/main/%E4%BB%8E%E9%9B%B6%E6%90%AD%E5%BB%BAspring%20security%20oauth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E4%BD%93%E7%B3%BB%EF%BC%88%E5%9B%9B%EF%BC%89](https://)