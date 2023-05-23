---
title: 从零搭建spring security oauth2认证授权体系（八）
date: 2021-06-24 17:30:00
author: maybe
top: false
cover: false
toc: false
mathjax: false
summary: 性能优化
tags: Spring Security Oauth2
categories:
- [Spring Security Oauth2]
---
## 性能优化

### 优化点1

授权服务每次登录认证的时候，都去数据库中加载一次用户信息，可以修改为从redis中加载

```java
@Service
@RequiredArgsConstructor
public class SpringUserDetailsService implements UserDetailsService {
    private final UserMapper userMapper;
    private final UserRoleMapper userRoleMapper;
    private final RolePermissionMapper rolePermissionMapper;
    private final PermissionMapper permissionMapper;
    private final RedisTemplate<String, SpringUserDetails> redisTemplate;
    private static final ReentrantLock reentrantLock = new ReentrantLock();

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        SpringUserDetails userDetails = loadUserByUsernameFromRedis(username);
        if (Objects.nonNull(userDetails)) {
            return userDetails;
        }

        reentrantLock.lock();
        try {
            userDetails = loadUserByUsernameFromRedis(username);
            if (Objects.nonNull(userDetails)) {
                return userDetails;
            }

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

            userDetails = new SpringUserDetails(userDO.getUsername(), userDO.getPassword(), userDO.isEnabled(),
                    true, true, !userDO.isLocked(), authorities, userDO.getPhone(), userDO.getOpenId(), userDO.getId());

            redisTemplate.opsForValue().set(redisKey(username), userDetails);

            return userDetails;
        } finally {
            reentrantLock.unlock();
        }
    }

    private SpringUserDetails loadUserByUsernameFromRedis(final String username) {
        return redisTemplate.opsForValue().get(redisKey(username));
    }

    private String redisKey(final String username) {
        return "oauth2:user_details:" + username;
    }

}
```

### 优化点2

授权服务每次登录认证、每次授权校验token的时候都使用JdbcClientDetailsService加载数次客户端信息，可以实现自定义ClientDetailsService从redis中加载

```java

@Slf4j
public class SpringRedisClientDetailsService implements ClientDetailsService {
    private static final String CLIENT_FIELDS_FOR_UPDATE = "resource_ids, scope, "
            + "authorized_grant_types, web_server_redirect_uri, authorities, access_token_validity, "
            + "refresh_token_validity, additional_information, autoapprove";
    private static final String CLIENT_FIELDS = "client_secret, " + CLIENT_FIELDS_FOR_UPDATE;
    private static final String BASE_FIND_STATEMENT = "select client_id, " + CLIENT_FIELDS
            + " from oauth_client_details";
    private static final String DEFAULT_SELECT_STATEMENT = BASE_FIND_STATEMENT + " where client_id = ?";
    private String selectClientDetailsSql = DEFAULT_SELECT_STATEMENT;
    private final RedisTemplate<String, ClientDetails> redisTemplate;
    private final JdbcTemplate jdbcTemplate;
    private static final ReentrantLock LOCK = new ReentrantLock();

    public SpringRedisClientDetailsService(RedisTemplate<String, ClientDetails> redisTemplate, DataSource dataSource) {
        Assert.notNull(redisTemplate, "RedisTemplate required");
        Assert.notNull(dataSource, "DataSource required");
        this.redisTemplate = redisTemplate;
        this.jdbcTemplate = new JdbcTemplate(dataSource);
    }

    @Override
    public ClientDetails loadClientByClientId(String clientId) throws ClientRegistrationException {
        ClientDetails details = redisTemplate.opsForValue().get("oauth2:client:" + clientId);
        if (Objects.nonNull(details)) {
            return details;
        }

        LOCK.lock();
        try {
            try {
                details = jdbcTemplate.queryForObject(selectClientDetailsSql, new SpringRedisClientDetailsService.ClientDetailsRowMapper(), clientId);
            } catch (EmptyResultDataAccessException e) {
                throw new NoSuchClientException("No client with requested id: " + clientId);
            }
            assert details != null;
            redisTemplate.opsForValue().set("oauth2:client:" + clientId, details);
            return details;
        } finally {
            LOCK.unlock();
        }
    }

    private static class ClientDetailsRowMapper implements RowMapper<ClientDetails> {
        private final SpringRedisClientDetailsService.JsonMapper mapper = createJsonMapper();

        public ClientDetails mapRow(ResultSet rs, int rowNum) throws SQLException {
            BaseClientDetails details = new BaseClientDetails(rs.getString(1), rs.getString(3), rs.getString(4),
                    rs.getString(5), rs.getString(7), rs.getString(6));
            details.setClientSecret(rs.getString(2));
            if (rs.getObject(8) != null) {
                details.setAccessTokenValiditySeconds(rs.getInt(8));
            }
            if (rs.getObject(9) != null) {
                details.setRefreshTokenValiditySeconds(rs.getInt(9));
            }
            String json = rs.getString(10);
            if (json != null) {
                try {
                    @SuppressWarnings("unchecked")
                    Map<String, Object> additionalInformation = mapper.read(json, Map.class);
                    details.setAdditionalInformation(additionalInformation);
                } catch (Exception e) {
                    log.warn("Could not decode JSON for additional information: " + details, e);
                }
            }
            String scopes = rs.getString(11);
            if (scopes != null) {
                details.setAutoApproveScopes(StringUtils.commaDelimitedListToSet(scopes));
            }
            return details;
        }
    }

    interface JsonMapper {
        String write(Object input) throws Exception;

        <T> T read(String input, Class<T> type) throws Exception;
    }

    private static SpringRedisClientDetailsService.JsonMapper createJsonMapper() {
        if (ClassUtils.isPresent("org.codehaus.jackson.map.ObjectMapper", null)) {
            return new SpringRedisClientDetailsService.JacksonMapper();
        } else if (ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", null)) {
            return new SpringRedisClientDetailsService.Jackson2Mapper();
        }
        return new SpringRedisClientDetailsService.NotSupportedJsonMapper();
    }

    private static class JacksonMapper implements SpringRedisClientDetailsService.JsonMapper {
        private final org.codehaus.jackson.map.ObjectMapper mapper = new org.codehaus.jackson.map.ObjectMapper();

        @Override
        public String write(Object input) throws Exception {
            return mapper.writeValueAsString(input);
        }

        @Override
        public <T> T read(String input, Class<T> type) throws Exception {
            return mapper.readValue(input, type);
        }
    }

    private static class Jackson2Mapper implements SpringRedisClientDetailsService.JsonMapper {
        private final com.fasterxml.jackson.databind.ObjectMapper mapper = new com.fasterxml.jackson.databind.ObjectMapper();

        @Override
        public String write(Object input) throws Exception {
            return mapper.writeValueAsString(input);
        }

        @Override
        public <T> T read(String input, Class<T> type) throws Exception {
            return mapper.readValue(input, type);
        }
    }

    private static class NotSupportedJsonMapper implements SpringRedisClientDetailsService.JsonMapper {
        @Override
        public String write(Object input) {
            throw new UnsupportedOperationException(
                    "Neither Jackson 1 nor 2 is available so JSON conversion cannot be done");
        }

        @Override
        public <T> T read(String input, Class<T> type) {
            throw new UnsupportedOperationException(
                    "Neither Jackson 1 nor 2 is available so JSON conversion cannot be done");
        }
    }
}
```

### 优化点3

RemoteTokenServices配置信息从代码中转移到配置文件中

```java
    @Bean
    public ResourceServerTokenServices tokenServices() {
        // 使用远程服务请求授权服务器校验token,必须指定校验token 的url、client_id，client_secret
        RemoteTokenServices tokenServices = new RemoteTokenServices();
        tokenServices.setCheckTokenEndpointUrl(securityClientConfig.getCheckTokenUri());
        tokenServices.setClientId(securityClientConfig.getClientId());
        tokenServices.setClientSecret(securityClientConfig.getClientSecret());
        tokenServices.setAccessTokenConverter(new SpringAccessTokenConverter());
        return tokenServices;
    }

```

```java
@Data
@Configuration
@ConfigurationProperties(
        prefix = "oauth2"
)
public class SecurityClientConfig {
    private String clientId;
    private String clientSecret;
    private String checkTokenUri;
}
```

```yaml
oauth2:
  client-id: order
  client-secret: secret
  check-token-uri: http://localhost:8090/oauth/check_token
```

### 代码地址

[https://github.com/onlyonezhongjinhui/spring-security-ouath2-learning/tree/main/%E4%BB%8E%E9%9B%B6%E6%90%AD%E5%BB%BAspring%20security%20oauth2%E8%AE%A4%E8%AF%81%E6%8E%88%E6%9D%83%E4%BD%93%E7%B3%BB%EF%BC%88%E5%85%AB%EF%BC%89](https://)
