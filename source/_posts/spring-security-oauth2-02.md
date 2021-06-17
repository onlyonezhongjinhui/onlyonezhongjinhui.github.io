---
title: 使用Spring Security搭建一个简易的Oauth资源服务
date: 2021-06-17 16:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary:
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
### 创建资源服务应用，添加如下依赖

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

![](/medias/assets/20210617164410.png)

### 资源服务器配置

**@EnableResourceServer **注解到一个 @Configuration 配置类上，并且必须使用 ResourceServerConfigurer 这个配置对象来进行配置（可以选择继承自 ResourceServerConfigurerAdapter 然后覆写其中的方法，参数就是这个对象的实例），下面是一些可以配置的属性：

**ResourceServerSecurityConfigurer**中主要包括：

* tokenServices：ResourceServerTokenServices 类的实例，用来实现令牌服务
* tokenStore：TokenStore类的实例，指定令牌如何访问，与tokenServices配置可选
* resourceId：这个资源服务的ID，这个属性是可选的，但是推荐设置并在授权服务中进行验证。
* 其他的拓展属性例如 tokenExtractor 令牌提取器用来提取请求中的令牌

**HttpSecurity**配置这个与Spring Security类似：

* 请求匹配器，用来设置需要进行保护的资源路径，默认的情况下是保护资源服务的全部路径
* 通过http.authorizeRequests()来设置受保护资源的访问规则
* 其他的自定义权限保护规则通过 HttpSecurity 来进行配置

**@EnableResourceServer** 注解自动增加了一个类型为 OAuth2AuthenticationProcessingFilter 的过滤器链

### 配置令牌验证

**ResourceServerTokenServices** 是组成授权服务的另一半，如果授权服务和资源服务在同一个应用程序上的
话，你可以使用 **DefaultTokenServices** ，这样的话，你就不用考虑关于实现所有必要的接口的一致性问题。如果资源服务器是分离开的，那么你就必须要确保能够有匹配授权服务提供的 ResourceServerTokenServices，它知道如何对令牌进行解码。

* ** DefaultTokenServices** 资源服务器本地配置令牌存储、解码、解析方式
* **RemoteTokenServices **资源服务器通过 HTTP 请求来解码令牌，每次都请求授权服务器端点 /oauth/check_token进行验证。所以授权服务要把 /oauth/check_token 端点暴露出去

这里资源服务是独立的应用，则具体配置代码如下所示：

```java
@Configuration
@RequiredArgsConstructor
@EnableResourceServer
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {
    private static final String RESOURCE_ID = "order";
    private final TokenStore tokenStore;

    @Override
    public void configure(ResourceServerSecurityConfigurer security) throws Exception {
        security
                .resourceId(RESOURCE_ID)
                .tokenStore(this.tokenStore)
                .tokenServices(tokenServices());
    }

    @Override
    public void configure(HttpSecurity http) throws Exception {
        http
                .authorizeRequests()
                .antMatchers(HttpMethod.GET, "/api/order/**").hasRole("USER")
                .antMatchers("/api/order/**").hasRole("ADMIN")
                .antMatchers("/api/goods/**").permitAll()
                .anyRequest().authenticated()
                .and().csrf().disable().sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
    }

    @Bean
    public ResourceServerTokenServices tokenServices() {
        // 使用远程服务请求授权服务器校验token,必须指定校验token 的url、client_id，client_secret
        RemoteTokenServices tokenServices = new RemoteTokenServices();
        tokenServices.setCheckTokenEndpointUrl("http://localhost:8090/oauth/check_token");
        tokenServices.setClientId("order");
        tokenServices.setClientSecret("secret");
        return tokenServices;
    }

}
```

对应的授权服务应用的主要配置如下：

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
                .withClient("order")
                .authorizedGrantTypes("authorization_code", "client_credentials", "password", "implicit", "refresh_token")
                .scopes("read", "write")
                .secret("{noop}secret")
                .autoApprove(true)
                .redirectUris("http://www.baidu.com");
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
                .authenticationManager(this.authenticationManager)
                .userDetailsService(this.userDetailsService)
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
        manager.createUser(users.username("user").password("password").roles("USER").build());
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

### 创建受保护资源

```java
@Slf4j
@RestController
@RequestMapping("api/order")
public class OrderController {

    @PostMapping
    public String post() {
        return "post order";
    }

    @PutMapping
    public String put() {
        return "put order";
    }

    @DeleteMapping
    public String delete() {
        return "delete order";
    }

    @GetMapping
    public String get() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return "get order";
    }

}
```

```java
@Slf4j
@RestController
@RequestMapping("api/goods")
public class GoodsController {

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public String post() {
        return "post goods";
    }

    @PutMapping
    @PreAuthorize("hasRole('ADMIN')")
    public String put() {
        return "put goods";
    }

    @DeleteMapping
    @PreAuthorize("hasRole('ADMIN')")
    public String delete() {
        return "delete goods";
    }

    @GetMapping
    @PreAuthorize("hasRole('USER')")
    public String get() {
        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
        return "get goods";
    }

}
```

### 测试

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

Response code: 200; Time: 39ms; Content length: 9 bytes
```
