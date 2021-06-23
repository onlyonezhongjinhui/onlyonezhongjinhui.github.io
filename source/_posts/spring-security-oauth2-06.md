---
title: 从零搭建spring security oauth2认证授权体系（六）
date: 2021-06-21 16:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: 丰富当前登录用户信息
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
## 丰富当前登录用户信息

### 资源服务器获取登录用户信息

#### 获取登录用户信息代码

```java
SecurityContextHolder.getContext().getAuthentication()
```

#### 资源服务器获取登录用户信息流程

![](/medias/assets/1223423423234.png)

从流程可以看出，关键的类是DefaultAccessTokenConverter、UserAuthenticationConverter，只要扩展这两个类，就能够放入更多信息。

### 授权服务自定义DefaultUserAuthenticationConverter

创建SpringUserAuthenticationConverter类继承DefaultUserAuthenticationConverter，重写convertUserAuthentication方法，放入更多用户相关的信息

```java
public class SpringUserAuthenticationConverter extends DefaultUserAuthenticationConverter {

    @Override
    public Map<String, ?> convertUserAuthentication(Authentication authentication) {
        Map<String, Object> response = new LinkedHashMap<String, Object>();
        SpringUserDetails userDetails = (SpringUserDetails) authentication.getPrincipal();
        response.put(USERNAME, authentication.getName());
        if (authentication.getAuthorities() != null && !authentication.getAuthorities().isEmpty()) {
            response.put(AUTHORITIES, AuthorityUtils.authorityListToSet(authentication.getAuthorities()));
        }
        response.put("phone", userDetails.getPhone());
        response.put("openId", userDetails.getOpenId());
        response.put("userId", userDetails.getUserId());
        return response;
    }

}

```

### 授权服务配置自定义SpringUserAuthenticationConverter

```java
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) {
        DefaultAccessTokenConverter defaultAccessTokenConverter = new DefaultAccessTokenConverter();
        defaultAccessTokenConverter.setUserTokenConverter(new SpringUserAuthenticationConverter());

        endpoints
                .authenticationManager(this.authenticationManager)
                .userDetailsService(this.userDetailsService)
                .tokenStore(tokenStore()) // 设置 jdbc TokenStore
                .authorizationCodeServices(authorizationCodeServices()) // 设置授权码服务
                .accessTokenConverter(defaultAccessTokenConverter); // 设置token转换器
    }
```

### 资源服务器自定义DefaultAccessTokenConverter

创建SpringAccessTokenConverter类继承DefaultAccessTokenConverter，重写extractAuthentication方法，读取授权服务中放入的用户相关信息

```java

public class SpringAccessTokenConverter extends DefaultAccessTokenConverter {
    private final UserAuthenticationConverter userTokenConverter = new DefaultUserAuthenticationConverter();

    @Override
    public OAuth2Authentication extractAuthentication(Map<String, ?> map) {
        Map<String, String> parameters = new HashMap<>();
        Set<String> scope = extractScope(map);
        Authentication user = userTokenConverter.extractAuthentication(map);
        String clientId = (String) map.get(CLIENT_ID);
        parameters.put(CLIENT_ID, clientId);
        if (map.containsKey(GRANT_TYPE)) {
            parameters.put(GRANT_TYPE, (String) map.get(GRANT_TYPE));
        }
        Set<String> resourceIds = new LinkedHashSet<>(map.containsKey(AUD) ? getAudience(map)
                : Collections.emptySet());

        Collection<? extends GrantedAuthority> authorities = null;
        if (user == null && map.containsKey(AUTHORITIES)) {
            @SuppressWarnings("unchecked")
            String[] roles = ((Collection<String>) map.get(AUTHORITIES)).toArray(new String[0]);
            authorities = AuthorityUtils.createAuthorityList(roles);
        }

        Map<String, Serializable> extensions = new HashMap<>();
        extensions.put("phone", (String) map.get("phone"));
        extensions.put("openId", (String) map.get("openId"));
        extensions.put("userId", (String) map.get("userId"));

        OAuth2Request request = new OAuth2Request(parameters, clientId, authorities, true, scope, resourceIds, null, null,
                extensions);
        return new OAuth2Authentication(request, user);
    }

    private Set<String> extractScope(Map<String, ?> map) {
        Set<String> scope = Collections.emptySet();
        if (map.containsKey(SCOPE)) {
            Object scopeObj = map.get(SCOPE);
            if (scopeObj instanceof String) {
                scope = new LinkedHashSet<>(Arrays.asList(((String) scopeObj).split(" ")));
            } else if (Collection.class.isAssignableFrom(scopeObj.getClass())) {
                @SuppressWarnings("unchecked")
                Collection<String> scopeColl = (Collection<String>) scopeObj;
                scope = new LinkedHashSet<>(scopeColl);    // Preserve ordering
            }
        }
        return scope;
    }

    private Collection<String> getAudience(Map<String, ?> map) {
        Object auds = map.get(AUD);
        if (auds instanceof Collection) {
            @SuppressWarnings("unchecked")
            Collection<String> result = (Collection<String>) auds;
            return result;
        }
        return Collections.singleton((String) auds);
    }

}
```

### 资源服务配置自定义SpringAccessTokenConverter

```java
    @Bean
    public ResourceServerTokenServices tokenServices() {
        // 使用远程服务请求授权服务器校验token,必须指定校验token 的url、client_id，client_secret
        RemoteTokenServices tokenServices = new RemoteTokenServices();
        tokenServices.setCheckTokenEndpointUrl("http://localhost:8090/oauth/check_token");
        tokenServices.setClientId("order");
        tokenServices.setClientSecret("secret");
        tokenServices.setAccessTokenConverter(new SpringAccessTokenConverter());
        return tokenServices;
    }
```

### 创建资源服务获取用户信息工具类SpringSecurityUtils

```java
public class SpringSecurityUtils {

    public static Optional<String> getCurrentUserPhone() {
        return Optional.ofNullable((String) getAuthenticationExtensions("phone"));
    }

    public static Optional<String> getCurrentUserOpenId() {
        return Optional.ofNullable((String) getAuthenticationExtensions("openId"));
    }

    public static Optional<String> getCurrentUserId() {
        return Optional.ofNullable((String) getAuthenticationExtensions("userId"));
    }

    private static Serializable getAuthenticationExtensions(String key) {
        OAuth2Authentication auth2Authentication = (OAuth2Authentication) SecurityContextHolder.getContext().getAuthentication();
        if (auth2Authentication == null) {
            return null;
        }
        return auth2Authentication.getOAuth2Request().getExtensions().get(key);
    }

}
```

### 受保护资源获取用户信息

```java
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
        String userId = SpringSecurityUtils.getCurrentUserId().orElse("");
        String phone = SpringSecurityUtils.getCurrentUserPhone().orElse("");
        String openid = SpringSecurityUtils.getCurrentUserOpenId().orElse("");
        return "userId:" + userId + ",phone:" + phone + ",openId:" + openid;
    }

}
```

### 测试

获取token：

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
  "access_token": "dd62d9d3-75c2-4d25-a5de-056da7d80a56",
  "token_type": "bearer",
  "refresh_token": "c612fadc-2df9-4aba-8c39-28d18a63b804",
  "expires_in": 2591999,
  "scope": "read write"
}
```

请求order的get接口：

```
GET http://localhost:8092/api/order
Authorization: Bearer dd62d9d3-75c2-4d25-a5de-056da7d80a56
```

响应：

```
userId:1406151407442780160,phone:,openId:

Response code: 200; Time: 35ms; Content length: 41 bytes
```