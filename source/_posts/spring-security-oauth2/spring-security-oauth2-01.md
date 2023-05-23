---
title: 从零搭建spring security oauth2认证授权体系（一）
date: 2021-06-16 16:00:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 搭建授权服务
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
## 搭建授权服务

### 创建授权服务应用

依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
    <version>2.2.4.RELEASE</version>
</dependency>
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

项目结构

![](/medias/assets/spring-security-oauth2/20210617174356.png)

### 授权服务配置

创建授权服务器配置类AuthorizationServerConfig进行Oauth2授权服务器配置，该类继承自AuthorizationServerConfigurerAdapter，并使用@EnableAuthorizationServer注解启用授权服务器。

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
  // 省略......
}
```

```java
public class AuthorizationServerConfigurerAdapter implements AuthorizationServerConfigurer {
  public AuthorizationServerConfigurerAdapter() {}
  public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {}
  public void configure(ClientDetailsServiceConfigurer clients) throws Exception {}
  public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {}
}
```

**AuthorizationServerConfigurerAdapter**需要配置上面的几个类，这几个类是由Spring创建的独立配置对象，它们会被Spring传入AuthorizationServerConfigurerAdapter中。

**AuthorizationServerSecurityConfigurer**：用来配置token的访问端点和token服务。

**ClientDetailsServiceConfigurer**：配置客户端详情服务ClientDetailsService，客户端详情信息在这里初始化，可以写死或者通过数据库获取。ClientDetailsService负责查找ClientDetails。ClientDetailsService是个接口，默认提供有两个实现类InMemoryClientDetailsService和JdbcClientDetailsService。
ClientDetails包含属性有

* clientId：（必须的）用来标识客户的Id
* secret：（需要值得信任的客户端）客户端安全码
* scope：用来限制客户端的访问范围，如果为空（默认）的话，那么客户端拥有全部的访问范围
* authorizedGrantTypes：此客户端可以使用的授权类型，默认为空
* authorities：此客户端可以使用的权限（基于Spring Security authorities）

**AuthorizationServerEndpointsConfigurer**：用来配置token端点安全约束。

### 配置客户端详情

```java
    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("order")
                .authorizedGrantTypes("authorization_code", "client_credentials", "password", "implicit", "refresh_token")
                .scopes("read", "write")
                .secret("{noop}secret")
                .autoApprove(true)
                .redirectUris("http://www.baidu.com");
    }
```

这里使用内存存储的方式配置客户端详情

### 配置令牌管理

AuthorizationServerTokenServices接口定义了一些操作使得你可以对令牌进行一些必要的管理，令牌可以被用来加载身份信息，里面包含了这个令牌的相关权限。自己可以创建 AuthorizationServerTokenServices 这个接口的实现，则需要继承 DefaultTokenServices 这个类，里面包含了一些有用实现，你可以使用它来修改令牌的格式和令牌的存储。默认的，当它尝试创建一个令牌的时候，是使用随机值来进行填充的，除了持久化令牌是委托一个 TokenStore 接口来实现以外，这个类几乎帮你做了所有的事情。并且 TokenStore 这个接口有一个默认的实现，它就是 InMemoryTokenStore ，如其命名，所有的令牌是被保存在了内存中。除了使用这个类以外，你还可以使用一些其他的预定义实现，下面有几个版本，它们都实现了TokenStore接口：

* *InMemoryTokenStore*：这个版本的实现是被默认采用的，它可以完美的工作在单服务器上（即访问并发量压力不大的情况下，并且它在失败的时候不会进行备份），大多数的项目都可以使用这个版本的实现来进行尝试，你可以在开发的时候使用它来进行管理，因为不会被保存到磁盘中，所以更易于调试
* *JdbcTokenStore*：这是一个基于JDBC的实现版本，令牌会被保存进关系型数据库。使用这个版本的实现时，你可以在不同的服务器之间共享令牌信息，使用这个版本的时候请注意添加"spring-jdbc"依赖
* *JwtTokenStore*：这个版本的全称是 JSON Web Token（JWT），它可以把令牌相关的数据进行编码（因此对于后端服务来说，它不需要进行存储，这将是一个重大优势），但是它有一个缺点，那就是撤销一个已经授权令牌将会非常困难，所以它通常用来处理一个生命周期较短的令牌以及撤销刷新令牌（refresh_token）。另外一个缺点就是这个令牌占用的空间会比较大，如果你加入了比较多用户凭证信息。JwtTokenStore 不会保存任何数据，但是它在转换令牌值以及授权信息方面与 DefaultTokenServices 所扮演的角色是一样的

```java
@Bean
public TokenStore tokenStore() {
    return new InMemoryTokenStore();
}
```

这里使用InMemoryTokenStore生成普通的token

### 配置令牌访问端点

框架默认端点有以下几个

* /oauth/authorize：授权端点。
* /oauth/token：令牌端点。
* /oauth/confirm_access：用户确认授权提交端点。
* /oauth/error：授权服务错误信息端点。
* /oauth/check_token：用于资源服务访问的令牌解析端点。
* /oauth/token_key：提供公有密匙的端点，如果你使用JWT令牌的话。

可以对默认端点URL进行修改，比如通过pathMapping("/oauth/token", "/api/oauth/token")就可以把/oauth/token修改为/api/oauth/token。这里使用不进行修改默认即可

### 配置令牌端点安全约束

```java
@Override
public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
    security
            .tokenKeyAccess("permitAll()")
            .checkTokenAccess("permitAll()")
            .allowFormAuthenticationForClients();
}
```

* tokenkey这个endpoint当使用JwtToken且使用非对称加密时，资源服务用于获取公钥，所以这个endpoint也开放
* checkToken这个endpoint开放
* allowFormAuthenticationForClients表示允许表单认证

这里开放/oauth/check_token、/oauth/token_key两个端点，并允许表单认证

### WEB安全配置

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.csrf().disable()
            .authorizeRequests()
            .anyRequest().authenticated()
            .and().formLogin();
}
```

#### 最终配置如下

```java
@Configuration
@RequiredArgsConstructor
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {
    private final AuthenticationManager authenticationManager;
    private final UserDetailsService userDetailsService;

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
        security
                .tokenKeyAccess("permitAll()")
                .checkTokenAccess("permitAll()")
                .allowFormAuthenticationForClients();
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.inMemory()
                .withClient("client")
                .authorizedGrantTypes("authorization_code", "client_credentials", "password", "implicit", "refresh_token")
                .scopes("all")
                .secret("{noop}secret")
                .autoApprove(true)
                .redirectUris("http://www.baidu.com");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
                .authenticationManager(this.authenticationManager)
                .userDetailsService(this.userDetailsService) // 使用刷新token必须配置
                .tokenStore(tokenStore());
    }

    @Bean
    public TokenStore tokenStore() {
        return new InMemoryTokenStore();
    }
  }
```

```java
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .authorizeRequests()
                .anyRequest().authenticated()
                .and().formLogin();
    }

    @Bean
    public UserDetailsService users() throws Exception {
        User.UserBuilder users = User.withDefaultPasswordEncoder();
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        manager.createUser(users.username("user1").password("password").roles("USER").build());
        manager.createUser(users.username("admin").password("password").roles("USER", "ADMIN").build());
        return manager;
    }

    @Bean
    @Override
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

}
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
  "access_token": "1f7c97f2-c048-4425-8463-244be95b2ee1",
  "token_type": "bearer",
  "expires_in": 42496,
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
  "access_token": "500ac853-aa32-4415-b7e8-ee91c410652c",
  "token_type": "bearer",
  "refresh_token": "c36079e9-72ee-46a0-a629-82da565062d7",
  "expires_in": 42099,
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
https://www.baidu.com/?code=kY8L5A
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
  "access_token": "500ac853-aa32-4415-b7e8-ee91c410652c",
  "token_type": "bearer",
  "refresh_token": "c36079e9-72ee-46a0-a629-82da565062d7",
  "expires_in": 41631,
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
https://www.baidu.com/#access_token=500ac853-aa32-4415-b7e8-ee91c410652c&token_type=bearer&expires_in=41575&scope=read%20write
```

### 验证token

请求：

```
GET http://localhost:8090/oauth/check_token?token=7f779d38-0629-4df1-84e0-6fa6a3da64cd
```

响应：

```
{
  "active": true,
  "exp": 1623938919,
  "user_name": "admin",
  "authorities": [
    "ROLE_ADMIN",
    "ROLE_USER"
  ],
  "client_id": "order",
  "scope": [
    "read",
    "write"
  ]
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
  "access_token": "cea4788b-a2f5-4d39-b411-05dd36ed6257",
  "token_type": "bearer",
  "refresh_token": "f32fe1fa-a5c4-4bdd-b325-8130cb39430e",
  "expires_in": 43200,
  "scope": "read write"
}
```

### 完整代码

[https://github.com/onlyonezhongjinhui/spring-security-ouath2-learning/tree/main/%E4%BB%8E%E9%9B%B6%E6%90%AD%E5%BB%BAspring%20security%20oauth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E4%BD%93%E7%B3%BB%EF%BC%88%E4%B8%80%EF%BC%89/authorization-server](https://)
