---
title: 从零搭建spring security oauth2认证授权体系（三）
date: 2021-06-18 16:00:00
author: maybe
top: true
cover: true
toc: false
mathjax: false
summary: 使用JWT令牌
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
## 使用JWT令牌

### 什么是JWT

JSON Web Token（JWT）是一个开放的行业标准（RFC 7519），它定义了一种简介的、自包含的协议格式，用于在通信双方传递JSON对象，传递的信息经过数字签名可以被验证和信任。JWT可以使用HMAC算法或使用RSA的公钥/私钥对来签名，防止被篡改。

### JWT优缺点

* JWT基于JSON,非常方便解析
* 可以在令牌中自定义丰富内容，易扩展
* 通过非对称加密算法以及数字签名技术，防止JWT令牌被篡改，安全性高
* 资源服务使用JWT可不依赖认证服务就能完成授权，减少远程调用校验令牌的网络开销，提升性能
* JWT的无状态，有助于增强系统的可用性和可伸缩性
* 一般不存在cookie中，有效避免了CSRF攻击
* 适合移动端（有些不支持cookie的）
* 对单点登录友好

### JWT缺点

* JWT令牌很长，占用存储空间占用大
* 无法在服务器注销，有被劫持的风险
* JWT验证签名耗CPU

### 如何使用JWT

#### 配置JWT令牌服务

授权服务配置类中**AuthorizationServerConfig**配置JWT令牌服务

```java
    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints
                .authenticationManager(this.authenticationManager)
                .userDetailsService(this.userDetailsService)
                .tokenStore(tokenStore()) // 配置令牌管理为JWT
                .accessTokenConverter(accessTokenConverter()); // 配置令牌转换器（授权信息<->令牌双向转换）
    }

    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        // 对称秘钥，资源服务器使用该秘钥来验证
        converter.setSigningKey("test123456");
        return converter;
    }
```

#### 配置资源服务器令牌认证

资源服务器配置类**ResourceServerConfig**配置JWT令牌认证

```java
    @Bean
    public TokenStore tokenStore() {
        return new JwtTokenStore(accessTokenConverter());
    }

    @Bean
    public JwtAccessTokenConverter accessTokenConverter() {
        JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
        // 对称秘钥，资源服务器使用该秘钥来验证
        converter.setSigningKey("test123456");
        return converter;
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
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzY29wZSI6WyJyZWFkIiwid3JpdGUiXSwiZXhwIjoxNjI0MDMwMTUxLCJqdGkiOiJmMDcyNjZmZi1kMTJiLTRlODMtOGE4MS0yYmUyNjdhOTEwNWIiLCJjbGllbnRfaWQiOiJvcmRlciJ9.kX4YcgrKH_XEDmeXUs_2XufJ-NbjNeUX2FVyi7CPfgI",
  "token_type": "bearer",
  "expires_in": 43199,
  "scope": "read write",
  "jti": "f07266ff-d12b-4e83-8a81-2be267a9105b"
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
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MjQwMzAyNDYsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiIsIlJPTEVfVVNFUiJdLCJqdGkiOiJhNzdiODM2YS03ZTBhLTRjNzctYmJjNC01ZGM5Y2QyZjViNTQiLCJjbGllbnRfaWQiOiJvcmRlciIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdfQ.rF1b4K8S8R276GXDfjocC4zGvTP5bNjVLkFb_8vmZrg",
  "token_type": "bearer",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdLCJhdGkiOiJhNzdiODM2YS03ZTBhLTRjNzctYmJjNC01ZGM5Y2QyZjViNTQiLCJleHAiOjE2MjY1NzkwNDYsImF1dGhvcml0aWVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1VTRVIiXSwianRpIjoiNTQxZTJiYTAtMTkxOC00ZTg2LTljNDItYzA2NWI1ZDZiMTk4IiwiY2xpZW50X2lkIjoib3JkZXIifQ.n6KTMcXShbcVdgllGFGZxHSALrrx-xVnOgFSGpcrI-I",
  "expires_in": 43199,
  "scope": "read write",
  "jti": "a77b836a-7e0a-4c77-bbc4-5dc9cd2f5b54"
}
```

#### 授权码模式

请求：

```
GET http://localhost:8090/oauth/authorize?client_id=order&response_type=code&grant_type=authorization_code&redirect_uri=http://www.baidu.com
```

响应：

```
https://www.baidu.com/?code=1lZbxO
```

使用响应连接中的code获取token

```
POST http://localhost:8090/oauth/token
Accept: application/json
Content-Type: application/x-www-form-urlencoded
Authorization: Basic b3JkZXI6c2VjcmV0

grant_type=authorization_code&client_id=order&client_secret=secret&redirect_uri=http://www.baidu.com&code=1lZbxO
```

响应：

```JSON
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MjQwMzA1NzAsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiIsIlJPTEVfVVNFUiJdLCJqdGkiOiI4ZjU0NTBhMy1mZGU5LTQwZmEtYjQ4Zi0wNmRkNDQ0NzRjN2EiLCJjbGllbnRfaWQiOiJvcmRlciIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdfQ.39vnXeQwIekYdDXUXUKKIHKcJ71Regm3jtXxfYvIVKQ",
  "token_type": "bearer",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdLCJhdGkiOiI4ZjU0NTBhMy1mZGU5LTQwZmEtYjQ4Zi0wNmRkNDQ0NzRjN2EiLCJleHAiOjE2MjY1NzkzNzAsImF1dGhvcml0aWVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1VTRVIiXSwianRpIjoiZDdkYjdhNzEtYWJkOS00ZjM0LWE4ZmMtNWJmMmM4MzI3MTgxIiwiY2xpZW50X2lkIjoib3JkZXIifQ.d2LCYsCLBjDAzXCHyWPmwu9xaLjXURT-Tj6gVSuKEIE",
  "expires_in": 43199,
  "scope": "read write",
  "jti": "8f5450a3-fde9-40fa-b48f-06dd44474c7a"
}
```

#### 简易模式

请求：

```
GET http://localhost:8090/oauth/authorize?client_id=order&response_type=token&grant_type=authorization_code&redirect_uri=http://www.baidu.com
```

响应：

```
https://www.baidu.com/#access_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MjQwMzA2MzEsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiIsIlJPTEVfVVNFUiJdLCJqdGkiOiJiZDVhODRiNi0wMzNiLTQxYjUtYjQxMC03MmMwYzcyYzgzNDMiLCJjbGllbnRfaWQiOiJvcmRlciIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdfQ.nnNiGaxCq9bJHiJY-sSWAMpGz4xR9XT6WDol0dHxbD8&token_type=bearer&expires_in=43200&scope=read%20write&jti=bd5a84b6-033b-41b5-b410-72c0c72c8343
```

### 验证token

请求：

```
GET http://localhost:8090/oauth/check_token?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MjQwMjcxMjksInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiIsIlJPTEVfVVNFUiJdLCJqdGkiOiJiZGExZTdlOC1iNGE1LTRhY2MtODQwNS0yYzkzMmEwNDkzZjYiLCJjbGllbnRfaWQiOiJvcmRlciIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdfQ.LCnyavxv7hhsTd1pGG0_fVYEV5oeYZg0mKKuiNemSfo
```

响应：

```json
{
  "user_name": "admin",
  "scope": [
    "read",
    "write"
  ],
  "active": true,
  "exp": 1624027129,
  "authorities": [
    "ROLE_ADMIN",
    "ROLE_USER"
  ],
  "jti": "bda1e7e8-b4a5-4acc-8405-2c932a0493f6",
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

grant_type=refresh_token&refresh_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdLCJhdGkiOiI4ZjU0NTBhMy1mZGU5LTQwZmEtYjQ4Zi0wNmRkNDQ0NzRjN2EiLCJleHAiOjE2MjY1NzkzNzAsImF1dGhvcml0aWVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1VTRVIiXSwianRpIjoiZDdkYjdhNzEtYWJkOS00ZjM0LWE4ZmMtNWJmMmM4MzI3MTgxIiwiY2xpZW50X2lkIjoib3JkZXIifQ.d2LCYsCLBjDAzXCHyWPmwu9xaLjXURT-Tj6gVSuKEIE

```

响应：

```JSON
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE2MjQwMzA4MTIsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiIsIlJPTEVfVVNFUiJdLCJqdGkiOiJmOWU5MzdmZC1lNDhiLTQ1ZDEtOWIwMy1iOTkzNWM5ZjEyMzUiLCJjbGllbnRfaWQiOiJvcmRlciIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdfQ.YRWaDwSo3SMrjANYJ3vLpqv8oaDuhiEsHnvn6sW2FEM",
  "token_type": "bearer",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJhZG1pbiIsInNjb3BlIjpbInJlYWQiLCJ3cml0ZSJdLCJhdGkiOiJmOWU5MzdmZC1lNDhiLTQ1ZDEtOWIwMy1iOTkzNWM5ZjEyMzUiLCJleHAiOjE2MjY1NzkzNzAsImF1dGhvcml0aWVzIjpbIlJPTEVfQURNSU4iLCJST0xFX1VTRVIiXSwianRpIjoiZDdkYjdhNzEtYWJkOS00ZjM0LWE4ZmMtNWJmMmM4MzI3MTgxIiwiY2xpZW50X2lkIjoib3JkZXIifQ.x6032Z5h72c5cqxN53too6bA96i1OuaVdtg5by-_ek4",
  "expires_in": 43200,
  "scope": "read write",
  "jti": "f9e937fd-e48b-45d1-9b03-b9935c9f1235"
}
```

### 测试资源服务

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

[https://github.com/onlyonezhongjinhui/spring-security-ouath2-learning/tree/main/%E4%BB%8E%E9%9B%B6%E6%90%AD%E5%BB%BAspring%20security%20oauth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E4%BD%93%E7%B3%BB%EF%BC%88%E4%B8%89%EF%BC%89](https://)