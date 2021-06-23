---
title: 从零搭建spring security oauth2认证授权体系（五）
date: 2021-06-20 10:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: RBAC
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
## RBAC

### 选择RBAC模型

#### 基于角色的访问控制（Role-Based Access Control）

RBAC基于角色的访问控制是按角色进行授权，例如：主体的角色为总经理可以查询企业运营报表，查询员工工资信息等，访问控制流程如下

```java
if(主体.hasRole("总经理角色id")){
    查询工资
}
```

如果查询工资所需要的角色变化为总经理和部门经理，此时就需要修改判断逻辑为“判断用户的角色是否是
总经理或部门经理”，修改代码如下：

```java
if(主体.hasRole("总经理角色id") || 主体.hasRole("部门经理角色id")){
    查询工资
}
```

**可以发现基于角色的访问控制缺点相当明显，当需要修改角色的权限的时候就需要修改代码才可以，这样相当于写死了，故可扩展性很差。**

#### 基于资源的访问控制（Resource-Based Access Control）

基于资源的访问控制是按资源（或权限）进行授权，例如：用户必须具有查询工资权限才可以查询员工工资信息，访问控制流程如下

```java
if(主体.hasPermission("查询员工工资信息权限id")){
    查询工资
}
```

**系统设计时定义好查询工资信息的权限标识，即使查询工资所需要的角色变化为总经理和部门经理也不需要修改
授权代码，系统可扩展性强**

所以这里我们采用基于资源的访问控制

### 创建用户、权限相关表

这里选择把权限和资源表合并的方式，至于权限资源表，根据需要适当增加字段，这里加入的前端资源的控制，比如菜单、路由等信息

```sql

create table `user`
(
    id       varchar(64)  not null primary key,
    username varchar(64)  not null,
    password varchar(256) not null,
    phone    varchar(11)  null,
    locked   tinyint(1)   not null,
    enabled  tinyint(1)   not null,
    open_id  varchar(64)  null,
    unique key username (`username`),
    unique key phone (`phone`),
    unique key open_id (`open_id`)
);

create table `role`
(
    `id`   varchar(64) not null primary key,
    `code` varchar(40) not null,
    `name` varchar(64) not null,
    unique key code (`code`),
    unique key name (`name`)
);

create table `user_role`
(
    `id`      varchar(64) not null primary key,
    `user_id` varchar(64) not null,
    `role_id` varchar(64) not null,
    key user_id (`user_id`),
    key role_id (`role_id`)
);

create table `permission`
(
    `id`            varchar(64) not null primary key,
    `parent_id`     varchar(64) not null,
    `code`          varchar(40) not null,
    `name`          varchar(64) not null,
    `resource_type` tinyint(1)  not null comment 'resource type eg 0:main menu 1:child menu 2:function button',
    `route_name`    varchar(64) not null comment 'route name',
    `route_url`     varchar(64) not null comment 'route url',
    `component`     varchar(32) not null comment 'component',
    `icon`          varchar(32) not null comment 'icon',
    unique key code (`code`),
    unique key name (`name`)
);

create table `role_permission`
(
    `id`            varchar(64) not null primary key,
    `permission_id` varchar(64) not null,
    `role_id`       varchar(64) not null,
    key permission_id (`permission_id`),
    key role_id (`role_id`)
);
```

预置数据

```sql

insert into role (id, code, name)
VALUES ('1406151407442780162', 'user', '普通用户'),
       ('1406151407442780163', 'admin', '管理员');

insert into permission (id, parent_id, code, name, resource_type, route_name, route_url, component, icon)
values ('1406153943707496456', '', 'menu:order', '订单管理菜单', '0', '商品管理', '/goods', 'GoodsList', 'goods'),
       ('1406153943707496457', '', 'menu:goods', '商品管理菜单', '0', '商品管理', '/order', 'OrderList', 'order'),
       ('1406144185783369728', '1406153943707496456', 'order:add', '添加订单', '2', '', '', '', ''),
       ('1406144185783369729', '1406153943707496456', 'order:update', '更新订单', '2', '', '', '', ''),
       ('1406144185783369730', '1406153943707496456', 'order:delete', '删除订单', '2', '', '', '', ''),
       ('1406144185783369731', '1406153943707496456', 'order:query', '查询订单', '2', '', '', '', ''),
       ('1406144185783369732', '1406153943707496457', 'goods:add', '添加商品', '2', '', '', '', ''),
       ('1406144185783369733', '1406153943707496457', 'goods:update', '更新商品', '2', '', '', '', ''),
       ('1406144185783369734', '1406153943707496457', 'goods:delete', '删除商品', '2', '', '', '', ''),
       ('1406144185787564032', '1406153943707496457', 'goods:query', '查询商品', '2', '', '', '', '');

insert into role_permission (id, permission_id, role_id)
values ('1406151407442780167', '1406153943707496456', '1406151407442780162'),
       ('1406151407442780168', '1406153943707496457', '1406151407442780162'),
       ('1406151407442780169', '1406144185783369731', '1406151407442780162'),
       ('1406151407442780170', '1406144185787564032', '1406151407442780162'),

       ('1406151407442780171', '1406153943707496456', '1406151407442780163'),
       ('1406151407442780172', '1406153943707496457', '1406151407442780163'),
       ('1406151407442780173', '1406144185783369728', '1406151407442780163'),
       ('1406153943707496448', '1406144185783369729', '1406151407442780163'),
       ('1406153943707496449', '1406144185783369730', '1406151407442780163'),
       ('1406153943707496450', '1406144185783369731', '1406151407442780163'),
       ('1406153943707496451', '1406144185783369732', '1406151407442780163'),
       ('1406153943707496452', '1406144185783369733', '1406151407442780163'),
       ('1406153943707496453', '1406144185783369734', '1406151407442780163'),
       ('1406153943707496454', '1406144185787564032', '1406151407442780163');

insert into user (id, username, password, phone, locked, enabled, open_id)
VALUES ('1406151407442780160', 'user', 'password', null, 0, 1, null),
       ('1406151407442780161', 'admin', 'password', null, 0, 1, null);

insert into user_role (id, user_id, role_id)
VALUES ('1406151407442780164', '1406151407442780160', '1406151407442780162'),
       ('1406151407442780165', '1406151407442780161', '1406151407442780162'),
       ('1406151407442780166', '1406151407442780161', '1406151407442780163');
```

### 增加依赖

引入mybatis进行数据库CRUD

```xml
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.1.1</version>
        </dependency>
```

并配置mybatis的mapper路径

```yaml
mybatis:
  mapper-locations: classpath:/mappers/*.xml
```

### 生成CRUD的代码

这里通过idea插件EasyCodeMybatisCodeHelper生成CRUD的代码

![](/medias/assets/20210621093428.png)

### 自定义UserDetailsService、UserDetails

```java
package io.github.oauth2.spring;

import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.SpringSecurityCoreVersion;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.util.Assert;

import java.io.Serializable;
import java.util.*;

public class SpringUserDetails implements UserDetails {
    private final String password;
    private final String username;
    private final Set<GrantedAuthority> authorities;
    private final boolean accountNonExpired;
    private final boolean accountNonLocked;
    private final boolean credentialsNonExpired;
    private final boolean enabled;

    public SpringUserDetails(String username, String password, boolean enabled,
                             boolean accountNonExpired, boolean credentialsNonExpired,
                             boolean accountNonLocked, Collection<? extends GrantedAuthority> authorities) {
        if (((username == null) || "".equals(username)) || (password == null)) {
            throw new IllegalArgumentException(
                    "Cannot pass null or empty values to constructor");
        }

        this.username = username;
        this.password = password;
        this.enabled = enabled;
        this.accountNonExpired = accountNonExpired;
        this.credentialsNonExpired = credentialsNonExpired;
        this.accountNonLocked = accountNonLocked;
        this.authorities = Collections.unmodifiableSet(sortAuthorities(authorities));
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return authorities;
    }

    @Override
    public String getPassword() {
        return password;
    }

    @Override
    public String getUsername() {
        return username;
    }

    @Override
    public boolean isAccountNonExpired() {
        return accountNonExpired;
    }

    @Override
    public boolean isAccountNonLocked() {
        return accountNonLocked;
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return credentialsNonExpired;
    }

    @Override
    public boolean isEnabled() {
        return enabled;
    }

    private static SortedSet<GrantedAuthority> sortAuthorities(
            Collection<? extends GrantedAuthority> authorities) {
        Assert.notNull(authorities, "Cannot pass a null GrantedAuthority collection");
        // Ensure array iteration order is predictable (as per
        // UserDetails.getAuthorities() contract and SEC-717)
        SortedSet<GrantedAuthority> sortedAuthorities = new TreeSet<>(
                new SpringUserDetails.AuthorityComparator());
        for (GrantedAuthority grantedAuthority : authorities) {
            Assert.notNull(grantedAuthority,
                    "GrantedAuthority list cannot contain any null elements");
            sortedAuthorities.add(grantedAuthority);
        }

        return sortedAuthorities;
    }

    private static class AuthorityComparator implements Comparator<GrantedAuthority>,
            Serializable {
        private static final long serialVersionUID = SpringSecurityCoreVersion.SERIAL_VERSION_UID;

        public int compare(GrantedAuthority g1, GrantedAuthority g2) {
            // Neither should ever be null as each entry is checked before adding it to
            // the set.
            // If the authority is null, it is a custom authority and should precede
            // others.
            if (g2.getAuthority() == null) {
                return -1;
            }

            if (g1.getAuthority() == null) {
                return 1;
            }

            return g1.getAuthority().compareTo(g2.getAuthority());
        }
    }

}

```

```java
package io.github.oauth2.spring;

import io.github.oauth2.entity.RolePermissionDO;
import io.github.oauth2.entity.UserDO;
import io.github.oauth2.entity.UserRoleDO;
import io.github.oauth2.mapper.PermissionMapper;
import io.github.oauth2.mapper.RolePermissionMapper;
import io.github.oauth2.mapper.UserMapper;
import io.github.oauth2.mapper.UserRoleMapper;
import io.github.oauth2.query.RolePermissionQuery;
import io.github.oauth2.query.UserRoleQuery;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.core.userdetails.UsernameNotFoundException;
import org.springframework.stereotype.Service;

import java.util.HashSet;
import java.util.Objects;
import java.util.Set;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class SpringUserDetailsService implements UserDetailsService {
    private final UserMapper userMapper;
    private final UserRoleMapper userRoleMapper;
    private final RolePermissionMapper rolePermissionMapper;
    private final PermissionMapper permissionMapper;

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        UserDO userDO = userMapper.selectByUsername(username);
        if (Objects.isNull(userDO)) {
            throw new UsernameNotFoundException("用户不存在");
        }

        Set<SimpleGrantedAuthority> authorities = new HashSet<>();

        UserRoleQuery userRoleQuery = new UserRoleQuery();
        userRoleQuery.setUserId(userDO.getId());
        Set<String> roleIds = userRoleMapper.selectByQuery(userRoleQuery).stream().map(UserRoleDO::getRoleId).collect(Collectors.toSet());

        if (!roleIds.isEmpty()) {
            RolePermissionQuery rolePermissionQuery = new RolePermissionQuery();
            rolePermissionQuery.setRoleIds(roleIds);
            Set<String> permissionIds = rolePermissionMapper.selectByQuery(rolePermissionQuery).stream().map(RolePermissionDO::getPermissionId).collect(Collectors.toSet());

            authorities = permissionMapper.selectBatchIds(permissionIds).stream().map(e -> new SimpleGrantedAuthority(e.getCode())).collect(Collectors.toSet());
        }

        return new SpringUserDetails(userDO.getUsername(), userDO.getPassword(), userDO.isEnabled(),
                true, true, !userDO.isLocked(), authorities);
    }

}

```

最重要的是实现**loadUserByUsername**方法加载用户、权限信息。

### 增加PasswordEncoder

这里存储的是明文密码，所以使用NoOpPasswordEncoder

```java
    @Bean
    public PasswordEncoder passwordEncoder() {
        return NoOpPasswordEncoder.getInstance();
    }
```

### 配置资源服务器

```java
    @Override
    public void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers(HttpMethod.GET, "/api/order").hasAuthority("order:query")
                .antMatchers(HttpMethod.POST, "/api/order").hasAuthority("order:add")
                .antMatchers(HttpMethod.PUT, "/api/order").hasAuthority("order:update")
                .antMatchers(HttpMethod.DELETE, "/api/order").hasAuthority("order:delete")
                .antMatchers(HttpMethod.GET, "/api/goods").hasAuthority("goods:query")
                .antMatchers(HttpMethod.POST, "/api/goods").hasAuthority("goods:add")
                .antMatchers(HttpMethod.PUT, "/api/goods").hasAuthority("goods:update")
                .antMatchers(HttpMethod.DELETE, "/api/goods").hasAuthority("goods:delete")
                .anyRequest().authenticated()
                .and().csrf().disable().sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
```

### 测试四种授权模式

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
  "access_token": "d118fe4e-359a-44f0-aea7-12068b9c352d",
  "token_type": "bearer",
  "expires_in": 2591999,
  "scope": "read write"
}
```

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
  "access_token": "08b2b288-d0a7-48cf-a94a-66e588bb818e",
  "token_type": "bearer",
  "refresh_token": "bcb26967-736e-4b74-af74-3da027921f87",
  "expires_in": 2445744,
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
https://www.baidu.com/?code=7C9ton
```

使用响应连接中的code获取token

```
POST http://localhost:8090/oauth/token
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXI6c2VjcmV0

grant_type=authorization_code&client_id=order&client_secret=secret&redirect_uri=http://www.baidu.com&code=7C9ton
```

响应：

```JSON
{
  "access_token": "08b2b288-d0a7-48cf-a94a-66e588bb818e",
  "token_type": "bearer",
  "refresh_token": "bcb26967-736e-4b74-af74-3da027921f87",
  "expires_in": 2445675,
  "scope": "read write"
}
```

#### 简易模式

请求：

```
GET http://localhost:8090/oauth/authorize?client_id=order&response_type=token&grant_type=authorization_code&redirect_uri=http://www.baidu.com
```

响应：

```
https://www.baidu.com/#access_token=08b2b288-d0a7-48cf-a94a-66e588bb818e&token_type=bearer&expires_in=2445656&scope=read%20write
```

### 验证token

请求：

```
GET http://localhost:8092/oauth/check_token?token=7f779d38-0629-4df1-84e0-6fa6a3da64cd
```

响应：

```
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
  "exp": 1626686213,
  "authorities": [
    "goods:delete",
    "order:query",
    "order:delete",
    "goods:query",
    "order:add",
    "order:update",
    "goods:update",
    "menu:order",
    "menu:goods",
    "goods:add"
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

grant_type=refresh_token&refresh_token=f32fe1fa-a5c4-4bdd-b325-8130cb39430e
```

响应：

```JSON
{
  "access_token": "0a64f791-af0a-481e-8f43-935336bc3c62",
  "token_type": "bearer",
  "refresh_token": "bcb26967-736e-4b74-af74-3da027921f87",
  "expires_in": 2591999,
  "scope": "read write"
}
```

### 测试受保护资源

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

带admin用户的登录后的token请求：

```
GET http://localhost:8092/api/order
Authorization: Bearer 0a64f791-af0a-481e-8f43-935336bc3c62
```

响应：

```
get order

Response code: 200; Time: 39ms; Content length: 9 bytes
```

带user用户的登录后的token请求：

```
GET http://localhost:8092/api/order
Authorization: Bearer ae7897bd-c7d8-42aa-85a3-cb7f8c4ee5ac
```

响应：

```
get order

Response code: 200; Time: 44ms; Content length: 9 bytes
```

不带token请求：

```
POST http://localhost:8092/api/order
```

响应：

```
{
  "error": "unauthorized",
  "error_description": "Full authentication is required to access this resource"
}

Response code: 401; Time: 40ms; Content length: 102 bytes
```

带admin用户的登录后的token请求：

```
POST http://localhost:8092/api/order
Authorization: Bearer 0a64f791-af0a-481e-8f43-935336bc3c62
```

响应：

```
post order

Response code: 200; Time: 42ms; Content length: 10 bytes
```

带user用户的登录后的token请求：

```
POST http://localhost:8092/api/order
Authorization: Bearer ae7897bd-c7d8-42aa-85a3-cb7f8c4ee5ac
```

```
{
  "error": "access_denied",
  "error_description": "Access is denied"
}

Response code: 403; Time: 101ms; Content length: 64 bytes
```

### 代码地址

[https://github.com/onlyonezhongjinhui/spring-security-ouath2-learning/tree/main/%E4%BB%8E%E9%9B%B6%E6%90%AD%E5%BB%BAspring%20security%20oauth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E4%BD%93%E7%B3%BB%EF%BC%88%E4%BA%94%EF%BC%89](https://)