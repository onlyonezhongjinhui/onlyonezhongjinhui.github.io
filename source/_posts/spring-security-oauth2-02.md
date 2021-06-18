---
title: 从零搭建spring security oauth2认证授权体系（二）
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
## 搭建资源服务

### 创建资源服务应用

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

### 资源服务配置

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

### 完整代码在这里:https://github.com/onlyonezhongjinhui/spring-security-ouath2-learning/tree/main/%E4%BB%8E%E9%9B%B6%E6%90%AD%E5%BB%BAspring%20security%20oauth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E4%BD%93%E7%B3%BB%EF%BC%88%E4%BA%8C%EF%BC%89
