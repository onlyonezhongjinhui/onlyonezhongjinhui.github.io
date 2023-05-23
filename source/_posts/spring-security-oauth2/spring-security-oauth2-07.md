---
title: 从零搭建spring security oauth2认证授权体系（七）
date: 2021-06-23 17:45:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: Redis替换Jdbc
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
## Redis替换Jdbc

### 添加redis依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>
```

### 设置RedisTokenStore进行token存储

```java
    private final RedisConnectionFactory factory;

    @Bean
    public TokenStore tokenStore() {
        return new RedisTokenStore(factory);
    }
```

### 使用redis替换JdbcAuthorizationCodeServices

仿照JdbcAuthorizationCodeServices，实现一个自定义SpringRedisAuthorizationCodeServices类来存储授权码

```java
public class SpringRedisAuthorizationCodeServices extends RandomValueAuthorizationCodeServices {
    private final RedisTemplate<String, OAuth2Authentication> redisTemplate;

    public SpringRedisAuthorizationCodeServices(RedisTemplate<String, OAuth2Authentication> redisTemplate) {
        Assert.notNull(redisTemplate, "RedisTemplate required");
        this.redisTemplate = redisTemplate;
    }

    @Override
    protected void store(String code, OAuth2Authentication authentication) {
        redisTemplate.opsForValue().set("oauth2:oauth_code:" + code, authentication, 10, TimeUnit.MINUTES);
    }

    @Override
    protected OAuth2Authentication remove(String code) {
        OAuth2Authentication token = redisTemplate.opsForValue().get("oauth2:oauth_code:" + code);
        this.redisTemplate.delete("oauth2:oauth_code:" + code);
        return token;
    }

}
```

### 设置SpringRedisAuthorizationCodeServices

```java
   private final RedisTemplate<String, Object> redisTemplate;
 
   @Bean
    public AuthorizationCodeServices authorizationCodeServices() {
        return new SpringRedisAuthorizationCodeServices(redisTemplate);
    }
```

### 配置redis

创建RedisConfig配置类,**注意只能采用JdkSerializationRedisSerializer进行字符串的序列化，不然读取授权码的时候会报错**。

```java
@EnableCaching
@Configuration
public class RedisConfig extends CachingConfigurerSupport {
    @Resource
    private RedisConnectionFactory factory;

    @Override
    @Bean
    public KeyGenerator keyGenerator() {
        return (target, method, params) -> {
            StringBuilder sb = new StringBuilder();
            sb.append(target.getClass().getName());
            sb.append(".").append(method.getName());
            if (params.length > 0) {
                sb.append(".");
            }
            for (Object param : params) {
                sb.append(param.toString());
            }
            return sb.toString();
        };
    }

    @Bean
    public RedisTemplate<?, ?> redisTemplate() {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(factory);
        GenericJackson2JsonRedisSerializer genericJackson2JsonRedisSerializer = new GenericJackson2JsonRedisSerializer();
        redisTemplate.setKeySerializer(new StringRedisSerializer());
        redisTemplate.setValueSerializer(new JdkSerializationRedisSerializer());
        redisTemplate.setHashKeySerializer(new StringRedisSerializer());
        redisTemplate.setHashValueSerializer(genericJackson2JsonRedisSerializer);
        return redisTemplate;
    }

    @Bean
    @Override
    public CacheResolver cacheResolver() {
        return new SimpleCacheResolver(Objects.requireNonNull(cacheManager()));
    }

    @Bean
    @Override
    public CacheErrorHandler errorHandler() {
        // 用于捕获从Cache中进行CRUD时的异常的回调处理器。
        return new SimpleCacheErrorHandler();
    }

    @Bean
    @Override
    public CacheManager cacheManager() {
        RedisCacheConfiguration cacheConfiguration =
                defaultCacheConfig()
                        .disableCachingNullValues()
                        // 默认缓存过期时间8天+随机秒数,防止集中过期
                        .entryTtl(Duration.ofDays(8).plusSeconds(new Random().nextInt(100)))
                        .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
        return RedisCacheManager.builder(factory).cacheDefaults(cacheConfiguration).build();
    }

}
```

### 设置redis key前缀

```java
    @Bean
    public TokenStore tokenStore() {
        RedisTokenStore tokenStore = new RedisTokenStore(factory);
        tokenStore.setPrefix("oauth2:");
        return tokenStore;
    }
```

### 测试

#### 密码模式

获取token

```
POST http://localhost:8090/oauth/token
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXI6c2VjcmV0

grant_type=password&username=user&password=password
```

响应：

```json
{
  "access_token": "756a8b37-f14d-4402-9969-f72d261e34b2",
  "token_type": "bearer",
  "refresh_token": "45d12dbb-4c9c-4cef-b711-c0cbbd3d15c7",
  "expires_in": 2591999,
  "scope": "read write"
}
```

请求受保护资源

```
GET http://localhost:8092/api/order
Authorization: Bearer 756a8b37-f14d-4402-9969-f72d261e34b2
```

响应：

```
userId:1406151407442780160,phone:,openId:

Response code: 200; Time: 41ms; Content length: 41 bytes
```

#### 授权码模式

```
GET http://localhost:8090/oauth/authorize?client_id=order&response_type=code&grant_type=authorization_code&redirect_uri=http://www.baidu.com
```

响应：

```
https://www.baidu.com/?code=CXmj01
```

继续获取token

```
POST http://localhost:8090/oauth/token
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXI6c2VjcmV0

grant_type=authorization_code&client_id=order&client_secret=secret&redirect_uri=http://www.baidu.com&code=CXmj01
```

响应：

```json
{
  "access_token": "ea2d2e3b-4dd9-4fa5-bbb7-0ceb56d3fb0a",
  "token_type": "bearer",
  "refresh_token": "36abfc53-2f24-4f2e-8efe-5a32da8e5da2",
  "expires_in": 2591786,
  "scope": "read write"
}
```

查看redis，相关token信息已经存储在里面了

![](/medias/assets/spring-security-oauth2/20210624091958.png)

### 代码地址

[https://github.com/onlyonezhongjinhui/spring-security-ouath2-learning/tree/main/%E4%BB%8E%E9%9B%B6%E6%90%AD%E5%BB%BAspring%20security%20oauth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E4%BD%93%E7%B3%BB%EF%BC%88%E4%B8%83%EF%BC%89](https://)
