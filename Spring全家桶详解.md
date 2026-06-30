# Spring 全家桶详解

> 面向初学者的 Spring / Spring Boot / Spring Cloud 完整指南
>
> 版本参考：Spring 6.x · Spring Boot 3.x · Spring Cloud 2023.x

---

## 目录

1. [Spring Core](#1-spring-core)
   - 1.1 IoC 容器
   - 1.2 依赖注入 (DI)
   - 1.3 AOP 面向切面编程
   - 1.4 常用注解速查
2. [Spring Boot](#2-spring-boot)
   - 2.1 自动配置原理
   - 2.2 快速入门
   - 2.3 配置文件详解
   - 2.4 Starter 机制
3. [Spring Cloud](#3-spring-cloud)
   - 3.1 微服务概念
   - 3.2 Nacos 服务注册与发现
   - 3.3 Spring Cloud LoadBalancer
   - 3.4 OpenFeign 声明式 HTTP 客户端
   - 3.5 Sentinel 熔断限流
   - 3.6 Nacos Config 配置中心
   - 3.7 Spring Cloud Gateway 网关
4. [完整微服务示例](#4-完整微服务示例)
5. [常见问题 FAQ](#5-常见问题-faq)

---

## 1. Spring Core

### 1.1 IoC 容器

#### 什么是 IoC？

IoC（Inversion of Control，控制反转）是 Spring 框架的核心思想。传统编程中，对象由开发者手动创建和管理；使用 IoC 后，对象的创建和生命周期由 **Spring 容器** 统一管理，开发者只需要声明依赖，容器负责注入。

```
传统方式：
  开发者 --new--> 对象A --new--> 对象B

IoC 方式：
  开发者 --声明--> Spring容器 --创建并注入--> 对象A, 对象B
```

#### IoC 容器的两种实现

| 接口 | 实现类 | 特点 |
|------|--------|------|
| `BeanFactory` | `DefaultListableBeanFactory` | 懒加载，基础容器 |
| `ApplicationContext` | `ClassPathXmlApplicationContext`<br>`AnnotationConfigApplicationContext`<br>`GenericWebApplicationContext` | 即时加载，功能丰富，生产首选 |

#### Bean 的生命周期

```
+------------------------------------------------------------------+
|                        Bean 生命周期                              |
|                                                                   |
|  实例化(Instantiation)                                            |
|       |                                                           |
|  属性填充(Populate Properties)                                    |
|       |                                                           |
|  Aware 接口回调 (setBeanName, setBeanFactory, setApplicationContext)|
|       |                                                           |
|  BeanPostProcessor#postProcessBeforeInitialization               |
|       |                                                           |
|  初始化 (@PostConstruct / InitializingBean#afterPropertiesSet)    |
|       |                                                           |
|  BeanPostProcessor#postProcessAfterInitialization                |
|       |                                                           |
|  Bean 可以使用 (就绪状态)                                         |
|       |                                                           |
|  销毁 (@PreDestroy / DisposableBean#destroy)                      |
+------------------------------------------------------------------+
```

#### Bean 作用域

| 作用域 | 说明 |
|--------|------|
| `singleton` | 默认，容器中只有一个实例 |
| `prototype` | 每次获取创建新实例 |
| `request` | 每个 HTTP 请求一个实例（Web 环境）|
| `session` | 每个 HTTP Session 一个实例（Web 环境）|
| `application` | 整个 ServletContext 一个实例（Web 环境）|

```java
@Component
@Scope("prototype")  // 声明为原型作用域
public class MyService {
    // 每次从容器获取都是新对象
}
```

#### 基于注解的容器配置示例

```java
// 配置类 —— 替代 XML 配置文件
@Configuration
@ComponentScan("com.example")       // 扫描该包下所有带注解的类
@PropertySource("classpath:app.properties") // 加载属性文件
public class AppConfig {

    // 手动声明一个 Bean
    @Bean
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost:3306/demo");
        ds.setUsername("root");
        ds.setPassword("123456");
        return ds;
    }
}
```

```java
// 启动容器并获取 Bean
public class Main {
    public static void main(String[] args) {
        ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
        MyService service = ctx.getBean(MyService.class);
        service.doSomething();
    }
}
```

---

### 1.2 依赖注入 (DI)

依赖注入（Dependency Injection）是 IoC 的具体实现方式，Spring 支持三种注入方式：

#### 方式一：构造器注入（推荐）

```java
@Service
public class OrderService {

    private final UserService userService;
    private final ProductService productService;

    // Spring 4.3+ 单构造器可省略 @Autowired
    @Autowired
    public OrderService(UserService userService, ProductService productService) {
        this.userService = userService;
        this.productService = productService;
    }

    public Order createOrder(Long userId, Long productId) {
        User user = userService.findById(userId);
        Product product = productService.findById(productId);
        return new Order(user, product);
    }
}
```

> **为什么推荐构造器注入？**
> - 依赖不可变（`final` 修饰），线程安全
> - 对象创建时依赖就绪，避免空指针
> - 容易发现循环依赖（启动时报错）

#### 方式二：Setter 注入（可选依赖）

```java
@Service
public class NotificationService {

    private EmailSender emailSender;

    @Autowired(required = false) // required=false 表示该依赖可以不存在
    public void setEmailSender(EmailSender emailSender) {
        this.emailSender = emailSender;
    }
}
```

#### 方式三：字段注入（不推荐用于生产）

```java
@Service
public class UserService {

    @Autowired // 直接注入字段，测试和维护困难
    private UserRepository userRepository;
}
```

#### @Qualifier —— 按名称区分多个实现

```java
public interface PaymentGateway {
    void pay(BigDecimal amount);
}

@Component("alipay")
public class AlipayGateway implements PaymentGateway { ... }

@Component("wechat")
public class WechatGateway implements PaymentGateway { ... }

@Service
public class CheckoutService {

    private final PaymentGateway gateway;

    public CheckoutService(@Qualifier("alipay") PaymentGateway gateway) {
        this.gateway = gateway;
    }
}
```

#### @Value —— 注入配置值

```java
@Component
public class AppProperties {

    @Value("${app.name:默认应用名}")   // 冒号后面是默认值
    private String appName;

    @Value("${server.port}")
    private int serverPort;

    @Value("#{systemProperties['java.version']}") // SpEL 表达式
    private String javaVersion;
}
```

---

### 1.3 AOP 面向切面编程

#### 核心概念

```
+-------------------------------------------------------------+
|                      AOP 核心概念                            |
|                                                              |
|  切面 (Aspect)      = 横切关注点的模块化封装                  |
|  连接点 (JoinPoint) = 程序执行中的某个点（如方法调用）         |
|  切入点 (Pointcut)  = 匹配连接点的表达式                      |
|  通知 (Advice)      = 切面在连接点执行的动作                  |
|  织入 (Weaving)     = 将切面应用到目标对象的过程              |
|                                                              |
|  通知类型：                                                   |
|    @Before          前置通知 -- 方法执行前                    |
|    @After           后置通知 -- 方法执行后（无论成功失败）     |
|    @AfterReturning  返回通知 -- 方法成功返回后                |
|    @AfterThrowing   异常通知 -- 方法抛出异常后                |
|    @Around          环绕通知 -- 完全控制方法执行               |
+-------------------------------------------------------------+
```

#### 实际示例：统一日志切面

```java
@Aspect
@Component
public class LogAspect {

    private static final Logger log = LoggerFactory.getLogger(LogAspect.class);

    // 切入点表达式：匹配 com.example.service 包下所有类的所有方法
    @Pointcut("execution(* com.example.service..*.*(..))")
    public void serviceLayer() {}

    // 前置通知
    @Before("serviceLayer()")
    public void logBefore(JoinPoint jp) {
        log.info("调用方法: {}.{}，参数: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            Arrays.toString(jp.getArgs()));
    }

    // 返回通知
    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logAfterReturn(JoinPoint jp, Object result) {
        log.info("方法 {} 返回: {}", jp.getSignature().getName(), result);
    }

    // 异常通知
    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void logException(JoinPoint jp, Throwable ex) {
        log.error("方法 {} 抛出异常: {}", jp.getSignature().getName(), ex.getMessage());
    }

    // 环绕通知（最强大）
    @Around("serviceLayer()")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        try {
            Object result = pjp.proceed(); // 执行目标方法
            long elapsed = System.currentTimeMillis() - start;
            log.info("方法 {} 执行耗时: {}ms", pjp.getSignature().getName(), elapsed);
            return result;
        } catch (Throwable e) {
            log.error("方法 {} 执行失败", pjp.getSignature().getName(), e);
            throw e;
        }
    }
}
```

#### 实际示例：自定义注解 + AOP 实现权限校验

```java
// 1. 定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequireRole {
    String value(); // 需要的角色
}

// 2. 在业务方法上使用注解
@RestController
public class AdminController {

    @RequireRole("ADMIN")
    @GetMapping("/admin/users")
    public List<User> listUsers() {
        return userService.findAll();
    }
}

// 3. 编写切面逻辑
@Aspect
@Component
public class SecurityAspect {

    @Around("@annotation(requireRole)")
    public Object checkRole(ProceedingJoinPoint pjp, RequireRole requireRole) throws Throwable {
        String requiredRole = requireRole.value();
        // 从 SecurityContext 获取当前用户角色
        String currentRole = SecurityContextHolder.getCurrentRole();
        if (!requiredRole.equals(currentRole)) {
            throw new AccessDeniedException("需要角色: " + requiredRole);
        }
        return pjp.proceed();
    }
}
```

#### 切入点表达式语法速查

```
execution(修饰符? 返回类型 包名.类名.方法名(参数) throws异常?)

常用示例：
execution(* com.example.service.*.*(..))          匹配 service 包下所有方法
execution(* com.example..*.*(..))                 匹配 example 包及子包所有方法
execution(public * *(..))                         匹配所有 public 方法
execution(* save*(..))                            匹配所有 save 开头的方法
@annotation(com.example.annotation.MyAnnotation)  匹配带有指定注解的方法
@within(org.springframework.stereotype.Service)   匹配 @Service 类中的所有方法
bean(userService)                                 匹配名为 userService 的 Bean
```

---

### 1.4 常用注解速查

#### 组件注册注解

| 注解 | 说明 |
|------|------|
| `@Component` | 通用组件，注册为 Bean |
| `@Service` | 业务层组件（语义化的 @Component）|
| `@Repository` | 数据访问层组件，自动转换数据库异常 |
| `@Controller` | MVC 控制器 |
| `@RestController` | `@Controller + @ResponseBody`，返回 JSON |
| `@Configuration` | 配置类，内部方法可用 `@Bean` 声明 Bean |
| `@Bean` | 方法级别，返回值注册为 Bean |

#### 依赖注入注解

| 注解 | 说明 |
|------|------|
| `@Autowired` | 按类型自动注入（Spring 原生）|
| `@Qualifier("name")` | 配合 @Autowired 按名称选择 |
| `@Resource` | 按名称注入（JSR-250 标准）|
| `@Value("${key}")` | 注入配置属性 |
| `@Inject` | JSR-330 标准，等同 @Autowired |

#### Web MVC 注解

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping("/{id}")                   // GET /api/v1/users/1
    public User getById(@PathVariable Long id) { ... }

    @GetMapping                            // GET /api/v1/users?page=0&size=10
    public Page<User> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) { ... }

    @PostMapping                           // POST /api/v1/users
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@RequestBody @Valid CreateUserRequest req) { ... }

    @PutMapping("/{id}")                   // PUT /api/v1/users/1
    public User update(@PathVariable Long id,
                       @RequestBody @Valid UpdateUserRequest req) { ... }

    @DeleteMapping("/{id}")                // DELETE /api/v1/users/1
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { ... }
}
```

#### 事务注解

```java
@Service
@Transactional(readOnly = true)  // 类级别：所有方法默认只读事务
public class UserService {

    // 覆盖类级别配置：读写事务
    @Transactional(
        propagation = Propagation.REQUIRED,   // 默认：有则加入，无则创建
        isolation = Isolation.READ_COMMITTED, // 隔离级别
        timeout = 30,                         // 超时秒数
        rollbackFor = Exception.class         // 哪些异常触发回滚
    )
    public User createUser(CreateUserRequest req) {
        // 数据库操作...
    }
}
```

#### 事务传播行为速查

| 传播行为 | 说明 |
|----------|------|
| `REQUIRED` | 默认。有事务加入，没有则创建新事务 |
| `REQUIRES_NEW` | 始终创建新事务，挂起当前事务 |
| `SUPPORTS` | 有事务加入，没有则非事务执行 |
| `NOT_SUPPORTED` | 以非事务执行，挂起当前事务 |
| `MANDATORY` | 必须在事务中执行，否则抛异常 |
| `NEVER` | 不能在事务中执行，否则抛异常 |
| `NESTED` | 有事务则创建嵌套事务，没有则创建新事务 |

---

## 2. Spring Boot

### 2.1 自动配置原理

Spring Boot 最大的特点是 **"约定优于配置"**，通过自动配置大幅减少样板代码。

#### 启动流程图

```
+------------------------------------------------------------------------+
|                        Spring Boot 启动流程                             |
|                                                                         |
|  @SpringBootApplication                                                 |
|       |                                                                 |
|       +--- @SpringBootConfiguration  (等同 @Configuration)             |
|       +--- @ComponentScan            (扫描当前包及子包)                  |
|       +--- @EnableAutoConfiguration (开启自动配置)                      |
|                    |                                                    |
|                    v                                                    |
|  加载 META-INF/spring/                                                  |
|  org.springframework.boot.autoconfigure.AutoConfiguration.imports       |
|                    |                                                    |
|                    v                                                    |
|  逐个评估 @ConditionalOnClass                                           |
|           @ConditionalOnMissingBean                                     |
|           @ConditionalOnProperty  等条件注解                            |
|                    |                                                    |
|                    v                                                    |
|  满足条件 ---> 注册自动配置 Bean                                          |
|  不满足   ---> 跳过                                                      |
+------------------------------------------------------------------------+
```

#### 自动配置示例：理解 DataSource 自动配置

```java
// Spring Boot 内部的 DataSourceAutoConfiguration（简化版）
@AutoConfiguration
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })  // 需要有 DataSource 类
@ConditionalOnMissingBean(type = "io.r2dbc.spi.ConnectionFactory")
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean  // 如果用户没有自定义 DataSource，则自动创建
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```

> **关键点**：`@ConditionalOnMissingBean` 意味着你自己定义了 `DataSource` Bean，自动配置就不会生效，这就是 "用户配置优先" 原则。

#### 自动配置调试

```bash
# 启动时添加参数，查看哪些自动配置生效/未生效
java -jar app.jar --debug

# 或在 application.properties 中
debug=true
```

---

### 2.2 快速入门

#### 项目结构

```
my-spring-boot-app/
+-- src/
|   +-- main/
|   |   +-- java/
|   |   |   +-- com/example/demo/
|   |   |       +-- DemoApplication.java        <- 启动类
|   |   |       +-- controller/
|   |   |       |   +-- UserController.java
|   |   |       +-- service/
|   |   |       |   +-- UserService.java
|   |   |       |   +-- impl/
|   |   |       |       +-- UserServiceImpl.java
|   |   |       +-- repository/
|   |   |       |   +-- UserRepository.java
|   |   |       +-- model/
|   |   |           +-- entity/
|   |   |           |   +-- User.java
|   |   |           +-- dto/
|   |   |               +-- UserDTO.java
|   |   +-- resources/
|   |       +-- application.yml                 <- 主配置文件
|   |       +-- application-dev.yml             <- 开发环境配置
|   |       +-- application-prod.yml            <- 生产环境配置
|   |       +-- static/                         <- 静态资源
|   +-- test/
|       +-- java/
|           +-- com/example/demo/
|               +-- DemoApplicationTests.java
+-- pom.xml
```

#### pom.xml 基础依赖

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <!-- 继承 Spring Boot 父 POM，管理所有依赖版本 -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <name>demo</name>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Web 开发 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- 数据库访问（JPA）-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- MySQL 驱动 -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- 参数校验 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- 热重载（开发时使用）-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-devtools</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Lombok（简化 getter/setter）-->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- 测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

#### 启动类

```java
package com.example.demo;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

#### 完整 CRUD 示例

```java
// ===== 实体类 =====
@Entity
@Table(name = "users")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 50)
    @NotBlank(message = "用户名不能为空")
    private String username;

    @Column(nullable = false, unique = true)
    @Email(message = "邮箱格式不正确")
    private String email;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @PrePersist
    void prePersist() {
        this.createdAt = LocalDateTime.now();
    }
}

// ===== Repository =====
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Spring Data JPA 根据方法名自动生成 SQL
    Optional<User> findByEmail(String email);

    List<User> findByUsernameContainingIgnoreCase(String keyword);

    @Query("SELECT u FROM User u WHERE u.createdAt > :date")
    List<User> findRecentUsers(@Param("date") LocalDateTime date);
}

// ===== Service 接口 =====
public interface UserService {
    User createUser(CreateUserRequest req);
    User getUserById(Long id);
    Page<User> listUsers(int page, int size);
    User updateUser(Long id, UpdateUserRequest req);
    void deleteUser(Long id);
}

// ===== Service 实现 =====
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class UserServiceImpl implements UserService {

    private final UserRepository userRepository;

    @Override
    @Transactional
    public User createUser(CreateUserRequest req) {
        if (userRepository.findByEmail(req.getEmail()).isPresent()) {
            throw new BusinessException("邮箱已被注册: " + req.getEmail());
        }
        User user = User.builder()
            .username(req.getUsername())
            .email(req.getEmail())
            .build();
        return userRepository.save(user);
    }

    @Override
    public User getUserById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("用户不存在: " + id));
    }

    @Override
    public Page<User> listUsers(int page, int size) {
        return userRepository.findAll(
            PageRequest.of(page, size, Sort.by("createdAt").descending()));
    }

    @Override
    @Transactional
    public User updateUser(Long id, UpdateUserRequest req) {
        User user = getUserById(id);
        user.setUsername(req.getUsername());
        return userRepository.save(user);
    }

    @Override
    @Transactional
    public void deleteUser(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("用户不存在: " + id);
        }
        userRepository.deleteById(id);
    }
}

// ===== Controller =====
@RestController
@RequestMapping("/api/v1/users")
@RequiredArgsConstructor
@Validated
public class UserController {

    private final UserService userService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ApiResponse<User> createUser(@RequestBody @Valid CreateUserRequest req) {
        return ApiResponse.success(userService.createUser(req));
    }

    @GetMapping("/{id}")
    public ApiResponse<User> getUser(@PathVariable Long id) {
        return ApiResponse.success(userService.getUserById(id));
    }

    @GetMapping
    public ApiResponse<Page<User>> listUsers(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return ApiResponse.success(userService.listUsers(page, size));
    }

    @PutMapping("/{id}")
    public ApiResponse<User> updateUser(@PathVariable Long id,
                                         @RequestBody @Valid UpdateUserRequest req) {
        return ApiResponse.success(userService.updateUser(id, req));
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}

// ===== 统一响应体 =====
@Data
@Builder
public class ApiResponse<T> {
    private int code;
    private String message;
    private T data;

    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder().code(200).message("success").data(data).build();
    }

    public static <T> ApiResponse<T> error(int code, String message) {
        return ApiResponse.<T>builder().code(code).message(message).build();
    }
}

// ===== 全局异常处理 =====
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ApiResponse<Void> handleNotFound(ResourceNotFoundException e) {
        log.warn("资源未找到: {}", e.getMessage());
        return ApiResponse.error(404, e.getMessage());
    }

    @ExceptionHandler(BusinessException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Void> handleBusiness(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return ApiResponse.error(400, e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiResponse<Map<String, String>> handleValidation(MethodArgumentNotValidException e) {
        Map<String, String> errors = new HashMap<>();
        e.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        return ApiResponse.error(400, "参数校验失败");
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ApiResponse<Void> handleGeneral(Exception e) {
        log.error("系统异常", e);
        return ApiResponse.error(500, "系统内部错误");
    }
}
```

---

### 2.3 配置文件详解

#### application.yml 完整示例

```yaml
# 激活的环境配置
spring:
  profiles:
    active: dev   # 激活 application-dev.yml

  # 数据源配置
  datasource:
    url: jdbc:mysql://localhost:3306/demo?useSSL=false&serverTimezone=Asia/Shanghai
    username: root
    password: 123456
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:                    # HikariCP 连接池配置
      minimum-idle: 5          # 最小空闲连接数
      maximum-pool-size: 20    # 最大连接池大小
      idle-timeout: 600000     # 空闲连接超时（毫秒）
      max-lifetime: 1800000    # 连接最大存活时间
      connection-timeout: 30000

  # JPA 配置
  jpa:
    hibernate:
      ddl-auto: update         # create/create-drop/update/validate/none
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.MySQL8Dialect

  # Redis 配置
  data:
    redis:
      host: localhost
      port: 6379
      password: ""
      database: 0
      timeout: 3000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 8
          min-idle: 0

  # Jackson 配置
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
    default-property-inclusion: non_null   # 不序列化 null 值

# 服务器配置
server:
  port: 8080
  servlet:
    context-path: /
  tomcat:
    threads:
      max: 200
      min-spare: 10
    accept-count: 100

# 日志配置
logging:
  level:
    root: INFO
    com.example: DEBUG
    org.hibernate.SQL: DEBUG
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{50} - %msg%n"
  file:
    name: logs/app.log

# 自定义配置
app:
  name: 我的应用
  version: 1.0.0
  jwt:
    secret: your-secret-key-at-least-256-bits
    expiration: 86400000
  upload:
    path: /data/uploads
    max-size: 10MB
    allowed-types:
      - image/jpeg
      - image/png
      - application/pdf
```

#### 多环境配置

```yaml
# application-dev.yml（开发环境）
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/demo_dev
  jpa:
    show-sql: true

logging:
  level:
    com.example: DEBUG

server:
  port: 8080
```

```yaml
# application-prod.yml（生产环境）
spring:
  datasource:
    url: jdbc:mysql://prod-db-host:3306/demo_prod
    username: ${DB_USERNAME}    # 从环境变量读取敏感信息
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate        # 生产环境不允许自动建表

logging:
  level:
    root: WARN
    com.example: INFO

server:
  port: 80
```

#### @ConfigurationProperties 类型安全的配置绑定

```java
@ConfigurationProperties(prefix = "app")
@Component
@Data
@Validated
public class AppProperties {

    @NotBlank
    private String name;

    private String version = "1.0.0";

    private JwtProperties jwt = new JwtProperties();
    private UploadProperties upload = new UploadProperties();

    @Data
    public static class JwtProperties {
        private String secret;
        private long expiration = 86400000L;
    }

    @Data
    public static class UploadProperties {
        private String path = "/tmp/uploads";
        private List<String> allowedTypes = List.of("image/jpeg", "image/png");
    }
}
```

---

### 2.4 Starter 机制

#### 常用 Starter 列表

| Starter | 提供的功能 |
|---------|-----------|
| `spring-boot-starter-web` | Spring MVC, Tomcat, Jackson |
| `spring-boot-starter-data-jpa` | Hibernate, Spring Data JPA |
| `spring-boot-starter-data-redis` | Lettuce/Jedis Redis 客户端 |
| `spring-boot-starter-security` | Spring Security 认证授权 |
| `spring-boot-starter-actuator` | 监控端点（健康检查、指标等）|
| `spring-boot-starter-cache` | 缓存抽象（@Cacheable 等）|
| `spring-boot-starter-amqp` | RabbitMQ 消息队列 |
| `spring-boot-starter-mail` | 邮件发送 |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ |

#### Actuator 健康监控

```yaml
# 开启 Actuator 端点
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,env,beans
  endpoint:
    health:
      show-details: always
  info:
    env:
      enabled: true

info:
  app:
    name: ${spring.application.name}
    version: ${app.version}
```

```
常用 Actuator 端点：
GET /actuator/health          健康状态
GET /actuator/info            应用信息
GET /actuator/metrics         指标列表
GET /actuator/metrics/{name}  具体指标
GET /actuator/env             环境变量
GET /actuator/beans           所有 Bean
GET /actuator/mappings        URL 映射
POST /actuator/loggers/{name} 动态修改日志级别
```

---

## 3. Spring Cloud

### 3.1 微服务概念

#### 单体应用 vs 微服务

```
单体应用（Monolith）：
+----------------------------------------------+
|              单体应用                         |
|  +------------+  +------------+              |
|  |  用户模块   |  |  订单模块   |              |
|  +------------+  +------------+              |
|  +------------+  +------------+              |
|  |  商品模块   |  |  支付模块   |              |
|  +------------+  +------------+              |
|       统一部署、统一扩展                       |
+----------------------------------------------+
  优点：开发简单、部署方便
  缺点：扩展困难、技术栈固定、故障影响全局

微服务架构（Microservices）：
+------------+   +------------+   +------------+
| 用户服务    |   | 订单服务    |   | 商品服务    |
|  :8081     |   |  :8082     |   |  :8083     |
+-----+------+   +-----+------+   +-----+------+
      |                |                |
      +----------------+----------------+
                       |
               +-------+-------+
               |   服务注册中心   |
               |    (Nacos)    |
               +---------------+
  优点：独立部署、按需扩展、技术栈灵活
  缺点：分布式复杂性高、运维成本高
```

#### Spring Cloud 组件全景图

```
+----------------------------------------------------------------+
|                   Spring Cloud 生态                             |
|                                                                 |
|  客户端请求                                                      |
|       |                                                         |
|       v                                                         |
|  +---------+     路由/限流/认证                                  |
|  | Gateway |  <-- Spring Cloud Gateway                         |
|  +----+----+                                                    |
|       |  转发                                                   |
|       v                                                         |
|  +-----------------------------------------------+             |
|  |              微服务集群                         |             |
|  |                                               |             |
|  | +----------+ +----------+ +----------+        |             |
|  | | 用户服务  | | 订单服务  | | 商品服务  |        |             |
|  | +----------+ +----------+ +----------+        |             |
|  |     服务间调用：OpenFeign + LoadBalancer        |             |
|  +---------------------+-------------------------+             |
|                        |                                        |
|  +---------------------+----------------------------------+     |
|  |  Nacos                                                 |     |
|  |  +-----------------------+ +-----------------------+  |     |
|  |  |   服务注册与发现        | |       配置中心          |  |     |
|  |  +-----------------------+ +-----------------------+  |     |
|  +--------------------------------------------------------+     |
|                                                                 |
|  +-------------------+                                         |
|  |  Sentinel          |  熔断、限流、降级                        |
|  +-------------------+                                         |
+----------------------------------------------------------------+
```

---

### 3.2 Nacos 服务注册与发现

#### 安装 Nacos

```bash
# 下载 Nacos（以 2.3.0 为例）
wget https://github.com/alibaba/nacos/releases/download/2.3.0/nacos-server-2.3.0.tar.gz
tar -xzf nacos-server-2.3.0.tar.gz
cd nacos/bin

# 单机模式启动
./startup.sh -m standalone        # Linux/Mac
startup.cmd -m standalone         # Windows

# 访问控制台：http://localhost:8848/nacos
# 默认账号密码：nacos/nacos
```

#### 服务注册到 Nacos

**pom.xml 添加依赖**

```xml
<!-- Spring Cloud Alibaba 父依赖管理 -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2023.0.1.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2023.0.1</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Nacos 服务注册发现 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>
</dependencies>
```

**application.yml 配置**

```yaml
spring:
  application:
    name: user-service        # 服务名，注册到 Nacos 的标识

  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848    # Nacos 地址
        namespace: dev                 # 命名空间（可选）
        group: DEFAULT_GROUP           # 分组（可选）

server:
  port: 8081
```

**启动类**

```java
@SpringBootApplication
@EnableDiscoveryClient  // 开启服务注册发现（新版本可省略）
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}
```

---

### 3.3 Spring Cloud LoadBalancer

**添加依赖**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**使用 RestTemplate + @LoadBalanced**

```java
@Configuration
public class RestTemplateConfig {

    @Bean
    @LoadBalanced  // 开启负载均衡
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
@RequiredArgsConstructor
public class OrderService {

    private final RestTemplate restTemplate;

    public User getUserById(Long userId) {
        // 使用服务名代替 IP:Port，LoadBalancer 自动解析并负载均衡
        return restTemplate.getForObject(
            "http://user-service/api/v1/users/" + userId,
            User.class
        );
    }
}
```

**自定义负载均衡策略（随机）**

```java
@Configuration
@LoadBalancerClient(name = "user-service", configuration = RandomLoadBalancerConfig.class)
public class LoadBalancerConfig {}

public class RandomLoadBalancerConfig {
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment env,
            LoadBalancerClientFactory loadBalancerClientFactory) {
        String name = env.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
            name);
    }
}
```

---

### 3.4 OpenFeign 声明式 HTTP 客户端

**添加依赖**

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

**开启 Feign**

```java
@SpringBootApplication
@EnableFeignClients  // 开启 Feign 客户端扫描
public class OrderServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

**定义 Feign 客户端**

```java
@FeignClient(
    name = "user-service",          // 服务名（与 Nacos 注册名一致）
    path = "/api/v1/users",         // 统一路径前缀
    fallback = UserClientFallback.class  // 降级类
)
public interface UserClient {

    @GetMapping("/{id}")
    ApiResponse<User> getUserById(@PathVariable("id") Long id);

    @GetMapping
    ApiResponse<Page<User>> listUsers(
        @RequestParam("page") int page,
        @RequestParam("size") int size);

    @PostMapping
    ApiResponse<User> createUser(@RequestBody CreateUserRequest req);
}

// 降级实现
@Component
public class UserClientFallback implements UserClient {

    @Override
    public ApiResponse<User> getUserById(Long id) {
        return ApiResponse.error(503, "用户服务不可用，请稍后重试");
    }

    @Override
    public ApiResponse<Page<User>> listUsers(int page, int size) {
        return ApiResponse.error(503, "用户服务不可用");
    }

    @Override
    public ApiResponse<User> createUser(CreateUserRequest req) {
        return ApiResponse.error(503, "用户服务不可用");
    }
}
```

**Feign 超时与日志配置**

```yaml
spring:
  cloud:
    openfeign:
      client:
        config:
          default:               # 全局配置
            connect-timeout: 5000
            read-timeout: 10000
            logger-level: FULL   # NONE/BASIC/HEADERS/FULL
          user-service:          # 针对特定服务的配置
            connect-timeout: 2000
            read-timeout: 5000
```

**Feign 请求拦截器（传递 Token）**

```java
@Configuration
public class FeignConfig {

    @Bean
    public RequestInterceptor authInterceptor() {
        return requestTemplate -> {
            ServletRequestAttributes attrs =
                (ServletRequestAttributes) RequestContextHolder.getRequestAttributes();
            if (attrs != null) {
                String token = attrs.getRequest().getHeader("Authorization");
                if (token != null) {
                    requestTemplate.header("Authorization", token);
                }
            }
        };
    }
}
```

---

### 3.5 Sentinel 熔断限流

#### 安装 Sentinel Dashboard

```bash
java -Dserver.port=8080 \
     -Dcsp.sentinel.dashboard.server=localhost:8080 \
     -Dproject.name=sentinel-dashboard \
     -jar sentinel-dashboard-1.8.6.jar
# 访问：http://localhost:8080  账号密码：sentinel/sentinel
```

#### 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>
```

#### 配置文件

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080   # Sentinel 控制台地址
        port: 8719
      eager: true
```

#### @SentinelResource 注解

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    @GetMapping("/{id}")
    @SentinelResource(
        value = "getOrder",
        blockHandler = "getOrderBlock",    // 限流/熔断处理
        fallback = "getOrderFallback"      // 业务异常处理
    )
    public ApiResponse<Order> getOrder(@PathVariable Long id) {
        return ApiResponse.success(orderService.getById(id));
    }

    // 限流处理方法（必须加 BlockException 参数）
    public ApiResponse<Order> getOrderBlock(Long id, BlockException ex) {
        log.warn("订单查询被限流: id={}", id);
        return ApiResponse.error(429, "系统繁忙，请稍后重试");
    }

    // 降级处理方法（可加 Throwable 参数）
    public ApiResponse<Order> getOrderFallback(Long id, Throwable t) {
        log.error("订单查询降级: id={}", id, t);
        return ApiResponse.error(503, "服务暂时不可用");
    }
}
```

#### Sentinel 规则说明

```
流量控制规则（FlowRule）：
  grade=QPS     按每秒请求数限流
  grade=Thread  按并发线程数限流
  count         阈值（QPS 或线程数）
  strategy      DIRECT/RELATE/CHAIN

熔断降级规则（DegradeRule）：
  策略1: 慢调用比例（SLOW_REQUEST_RATIO）
    count         慢调用时间阈值（ms）
    slowRatioThreshold  慢调用比例阈值（0.0-1.0）
  策略2: 异常比例（ERROR_RATIO）
    count         异常比例阈值（0.0-1.0）
  策略3: 异常数（ERROR_COUNT）
    count         异常数阈值
  minRequestAmount   最小请求数
  timeWindow         熔断持续时间（秒）
```

---

### 3.6 Nacos Config 配置中心

#### 添加依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>
```

#### 配置文件（bootstrap.yml）

```yaml
# 注意：必须是 bootstrap.yml，在 application.yml 之前加载
spring:
  application:
    name: user-service

  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        namespace: dev
        group: DEFAULT_GROUP
        file-extension: yaml    # 配置文件格式
        # 额外共享配置
        shared-configs:
          - data-id: common.yaml
            group: SHARED_GROUP
            refresh: true
```

#### 在 Nacos 控制台创建配置

```
Data ID: user-service.yaml
Group:   DEFAULT_GROUP
Type:    YAML

内容示例：
app:
  name: 用户服务
  version: 2.0.0
  feature-flag:
    new-ui: true
    beta-api: false
database:
  pool-size: 20
```

#### 动态刷新配置

```java
// 方式一：@RefreshScope + @Value
@RestController
@RefreshScope  // 重要！配置变更时重新创建 Bean
public class FeatureFlagController {

    @Value("${app.feature-flag.new-ui:false}")
    private boolean newUiEnabled;

    @GetMapping("/feature/new-ui")
    public boolean isNewUiEnabled() {
        return newUiEnabled;
    }
}

// 方式二：@ConfigurationProperties（自动刷新，无需 @RefreshScope）
@ConfigurationProperties(prefix = "app")
@Component
@Data
public class AppProperties {
    private String name;
    private String version;
    private FeatureFlag featureFlag = new FeatureFlag();

    @Data
    public static class FeatureFlag {
        private boolean newUi = false;
        private boolean betaApi = false;
    }
}
```

---

### 3.7 Spring Cloud Gateway 网关

#### 添加依赖

```xml
<!-- Gateway 不能与 spring-boot-starter-web 同时使用！ -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

#### 路由配置

```yaml
spring:
  application:
    name: gateway-service

  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848

    gateway:
      # 开启根据服务名自动路由（lb://服务名）
      discovery:
        locator:
          enabled: true
          lower-case-service-id: true  # 服务名小写

      routes:
        # 路由到用户服务
        - id: user-service-route
          uri: lb://user-service          # lb:// 表示负载均衡
          predicates:
            - Path=/api/users/**          # 路径匹配
          filters:
            - StripPrefix=1              # 去掉路径前缀 /api
            - AddRequestHeader=X-Source, gateway  # 添加请求头
            - name: RequestRateLimiter   # 限流过滤器
              args:
                redis-rate-limiter.replenishRate: 100    # 每秒补充令牌数
                redis-rate-limiter.burstCapacity: 200    # 令牌桶容量

        # 路由到订单服务
        - id: order-service-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
            - Method=GET,POST             # 仅 GET 和 POST 请求
          filters:
            - StripPrefix=1

      # 全局过滤器（默认 CORS 配置）
      default-filters:
        - DedupeResponseHeader=Access-Control-Allow-Credentials Access-Control-Allow-Origin
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns: "*"
            allowedMethods: "*"
            allowedHeaders: "*"
            allowCredentials: true

server:
  port: 9000
```

#### 自定义全局过滤器（鉴权）

```java
@Component
@Slf4j
public class AuthGlobalFilter implements GlobalFilter, Ordered {

    private static final List<String> WHITE_LIST = List.of(
        "/api/users/login",
        "/api/users/register",
        "/actuator/health"
    );

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String path = exchange.getRequest().getPath().toString();

        // 白名单直接放行
        if (WHITE_LIST.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }

        // 获取 Token
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        if (token == null || token.isBlank()) {
            return unauthorized(exchange, "缺少认证 Token");
        }

        // 验证 Token（此处简化，实际应解析 JWT）
        if (!isValidToken(token)) {
            return unauthorized(exchange, "Token 无效或已过期");
        }

        // 将用户信息传递给下游服务
        String userId = extractUserId(token);
        ServerWebExchange mutatedExchange = exchange.mutate()
            .request(r -> r.header("X-User-Id", userId))
            .build();

        return chain.filter(mutatedExchange);
    }

    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
        exchange.getResponse().getHeaders().add("Content-Type", "application/json");
        String body = "{\"code\":401,\"message\":\"" + message + "\"}";
        DataBuffer buffer = exchange.getResponse().bufferFactory()
            .wrap(body.getBytes(StandardCharsets.UTF_8));
        return exchange.getResponse().writeWith(Mono.just(buffer));
    }

    @Override
    public int getOrder() {
        return -100;  // 数字越小，优先级越高
    }

    private boolean isValidToken(String token) {
        // 实际项目中解析并验证 JWT
        return token.startsWith("Bearer ");
    }

    private String extractUserId(String token) {
        // 从 JWT 中提取用户 ID
        return "user-123";
    }
}
```

#### Gateway 常用 Predicate（断言）

```yaml
predicates:
  - Path=/api/**                          # 路径匹配
  - Method=GET,POST                       # HTTP 方法
  - Header=X-Request-Id, \d+             # 请求头匹配（正则）
  - Query=color, red                      # 查询参数匹配
  - Host=**.example.com                   # 主机名匹配
  - After=2024-01-01T00:00:00+08:00[Asia/Shanghai]  # 时间之后
  - Before=2025-12-31T23:59:59+08:00[Asia/Shanghai] # 时间之前
  - RemoteAddr=192.168.1.0/24             # IP 段匹配
```

#### Gateway 常用 Filter（过滤器）

```yaml
filters:
  - StripPrefix=1                         # 去除路径前 N 段
  - AddRequestHeader=X-Source, gateway    # 添加请求头
  - AddResponseHeader=X-Powered-By, Spring-Gateway  # 添加响应头
  - RewritePath=/api/(?<segment>.*), /$\{segment}  # 路径重写
  - Retry=3                               # 重试次数
  - CircuitBreaker=myCircuitBreaker       # 断路器
  - name: RequestRateLimiter              # 限流（需要 Redis）
    args:
      redis-rate-limiter.replenishRate: 10
      redis-rate-limiter.burstCapacity: 20
```

---

## 4. 完整微服务示例

本章通过一个电商系统演示如何将 Spring Boot + Spring Cloud 组件组合使用。

### 4.1 系统架构

```
+-----------------------------------------------------------------------+
|                         电商微服务系统架构                               |
|                                                                         |
|  浏览器/APP                                                              |
|       |                                                                  |
|       v  HTTP 请求                                                       |
|  +----------+          端口: 9000                                        |
|  | Gateway  |  <-- 统一入口：路由、鉴权、限流、CORS                        |
|  +----+-----+                                                            |
|       |                                                                  |
|       +------------------+------------------+                            |
|       |                  |                  |                            |
|       v 8081             v 8082             v 8083                       |
|  +---------+       +-----------+       +-----------+                     |
|  | user-   |       |  order-   |       | product-  |                     |
|  | service |       |  service  |       |  service  |                     |
|  +---------+       +-----------+       +-----------+                     |
|       |              |      |               |                            |
|       v              v      v               v                            |
|  +--------+    +--------+  +----------+ +--------+                      |
|  |  用户DB |   | 订单DB  |  | 调用产品 | | 产品DB  |                      |
|  | MySQL  |    | MySQL  |  | OpenFeign| | MySQL  |                      |
|  +--------+    +--------+  +----------+ +--------+                      |
|                                                                          |
|  +-----------------------------------------------+                      |
|  |                  Nacos                         |                      |
|  |   服务注册/发现           配置中心                |                      |
|  +-----------------------------------------------+                      |
|                                                                          |
|  +-------------------+      +-------------------+                       |
|  |  Sentinel          |      |    Redis           |                      |
|  |  熔断/限流          |      |   缓存/限流存储      |                      |
|  +-------------------+      +-------------------+                       |
+-----------------------------------------------------------------------+
```

### 4.2 版本依赖矩阵

| Spring Boot | Spring Cloud | Spring Cloud Alibaba |
|-------------|--------------|----------------------|
| 3.2.x | 2023.0.x | 2023.0.1.x |
| 3.1.x | 2022.0.x | 2022.0.0.x |
| 2.7.x | 2021.0.x | 2021.0.5.x |

> **重要**：版本必须严格对应，否则会出现各种兼容性问题！

### 4.3 公共父 POM

```xml
<!-- parent-pom/pom.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.example.shop</groupId>
    <artifactId>shop-parent</artifactId>
    <version>1.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>shop-gateway</module>
        <module>shop-user-service</module>
        <module>shop-order-service</module>
        <module>shop-product-service</module>
        <module>shop-common</module>
    </modules>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <properties>
        <java.version>17</java.version>
        <spring-cloud.version>2023.0.1</spring-cloud.version>
        <spring-cloud-alibaba.version>2023.0.1.0</spring-cloud-alibaba.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.cloud</groupId>
                <artifactId>spring-cloud-dependencies</artifactId>
                <version>${spring-cloud.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
            <dependency>
                <groupId>com.alibaba.cloud</groupId>
                <artifactId>spring-cloud-alibaba-dependencies</artifactId>
                <version>${spring-cloud-alibaba.version}</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

### 4.4 公共模块 (shop-common)

```java
// 统一响应结构
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Result<T> implements Serializable {
    private int code;
    private String message;
    private T data;

    public static <T> Result<T> ok(T data) {
        return Result.<T>builder().code(200).message("OK").data(data).build();
    }

    public static <T> Result<T> fail(int code, String message) {
        return Result.<T>builder().code(code).message(message).build();
    }

    public boolean isSuccess() {
        return this.code == 200;
    }
}

// 业务异常
public class BizException extends RuntimeException {
    private final int code;

    public BizException(int code, String message) {
        super(message);
        this.code = code;
    }

    public BizException(String message) {
        this(400, message);
    }

    public int getCode() { return code; }
}

// 全局异常处理（在每个服务中复用）
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(BizException.class)
    public Result<Void> handleBiz(BizException e) {
        log.warn("业务异常: [{}] {}", e.getCode(), e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Result<Map<String, String>> handleValidation(MethodArgumentNotValidException e) {
        Map<String, String> errors = new LinkedHashMap<>();
        e.getBindingResult().getFieldErrors().forEach(err ->
            errors.put(err.getField(), err.getDefaultMessage()));
        return Result.fail(400, "参数校验失败: " + errors);
    }

    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public Result<Void> handleGeneral(Exception e) {
        log.error("系统异常", e);
        return Result.fail(500, "系统内部错误");
    }
}
```

### 4.5 用户服务 (shop-user-service)

```yaml
# application.yml
spring:
  application:
    name: user-service
  datasource:
    url: jdbc:mysql://localhost:3306/shop_user?serverTimezone=Asia/Shanghai
    username: root
    password: 123456
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848

server:
  port: 8081
```

```java
// 用户实体
@Entity
@Table(name = "t_user")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password;

    @Column(unique = true)
    private String email;

    @Column(name = "phone")
    private String phone;

    @Enumerated(EnumType.STRING)
    private UserStatus status = UserStatus.ACTIVE;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @PrePersist
    void onCreate() { createdAt = LocalDateTime.now(); }
}

public enum UserStatus { ACTIVE, DISABLED, DELETED }

// 用户 Repository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    boolean existsByUsername(String username);
}

// 用户服务
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    @Transactional
    public User register(RegisterRequest req) {
        if (userRepository.existsByUsername(req.getUsername())) {
            throw new BizException("用户名已存在: " + req.getUsername());
        }
        User user = User.builder()
            .username(req.getUsername())
            .password(passwordEncoder.encode(req.getPassword()))
            .email(req.getEmail())
            .phone(req.getPhone())
            .build();
        user = userRepository.save(user);
        log.info("新用户注册成功: id={}, username={}", user.getId(), user.getUsername());
        return user;
    }

    public User getById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new BizException(404, "用户不存在: " + id));
    }

    public User login(LoginRequest req) {
        User user = userRepository.findByUsername(req.getUsername())
            .orElseThrow(() -> new BizException("用户名或密码错误"));
        if (!passwordEncoder.matches(req.getPassword(), user.getPassword())) {
            throw new BizException("用户名或密码错误");
        }
        if (user.getStatus() == UserStatus.DISABLED) {
            throw new BizException("账号已被禁用");
        }
        return user;
    }
}

// 用户 Controller
@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping("/register")
    @ResponseStatus(HttpStatus.CREATED)
    public Result<UserVO> register(@RequestBody @Valid RegisterRequest req) {
        User user = userService.register(req);
        return Result.ok(UserVO.from(user));
    }

    @GetMapping("/{id}")
    public Result<UserVO> getUser(@PathVariable Long id) {
        return Result.ok(UserVO.from(userService.getById(id)));
    }

    @PostMapping("/login")
    public Result<LoginResponse> login(@RequestBody @Valid LoginRequest req) {
        User user = userService.login(req);
        // 生成 JWT Token（简化示例）
        String token = JwtUtils.generateToken(user.getId(), user.getUsername());
        return Result.ok(new LoginResponse(token, UserVO.from(user)));
    }
}
```

### 4.6 商品服务 (shop-product-service)

```yaml
# application.yml
spring:
  application:
    name: product-service
  datasource:
    url: jdbc:mysql://localhost:3306/shop_product?serverTimezone=Asia/Shanghai
    username: root
    password: 123456
  data:
    redis:
      host: localhost
      port: 6379
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848

server:
  port: 8083
```

```java
// 商品实体
@Entity
@Table(name = "t_product")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock;  // 库存

    @Column(nullable = false)
    private Integer sales = 0;  // 销量

    @Enumerated(EnumType.STRING)
    private ProductStatus status = ProductStatus.ON_SALE;
}

// 商品服务（含 Redis 缓存）
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class ProductService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, Object> redisTemplate;

    private static final String PRODUCT_CACHE_KEY = "product:";
    private static final Duration CACHE_TTL = Duration.ofMinutes(30);

    public Product getById(Long id) {
        // 先查缓存
        String cacheKey = PRODUCT_CACHE_KEY + id;
        Product cached = (Product) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }

        // 缓存未命中，查数据库
        Product product = productRepository.findById(id)
            .orElseThrow(() -> new BizException(404, "商品不存在: " + id));

        // 写入缓存
        redisTemplate.opsForValue().set(cacheKey, product, CACHE_TTL);
        return product;
    }

    @Transactional
    public boolean deductStock(Long productId, int quantity) {
        // 使用乐观锁扣减库存
        int updated = productRepository.deductStock(productId, quantity);
        if (updated == 0) {
            throw new BizException("库存不足，商品ID: " + productId);
        }
        // 删除缓存
        redisTemplate.delete(PRODUCT_CACHE_KEY + productId);
        return true;
    }
}

// Repository 中的库存扣减（防超卖）
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Modifying
    @Query("UPDATE Product p SET p.stock = p.stock - :qty, p.sales = p.sales + :qty " +
           "WHERE p.id = :id AND p.stock >= :qty")
    int deductStock(@Param("id") Long id, @Param("qty") int qty);
}
```

### 4.7 订单服务 (shop-order-service)

```yaml
# application.yml
spring:
  application:
    name: order-service
  datasource:
    url: jdbc:mysql://localhost:3306/shop_order?serverTimezone=Asia/Shanghai
    username: root
    password: 123456
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
    openfeign:
      client:
        config:
          default:
            connect-timeout: 3000
            read-timeout: 5000

server:
  port: 8082
```

```java
// Feign 客户端：调用用户服务
@FeignClient(name = "user-service", path = "/users",
             fallbackFactory = UserClientFallbackFactory.class)
public interface UserClient {
    @GetMapping("/{id}")
    Result<UserVO> getUser(@PathVariable Long id);
}

@Component
public class UserClientFallbackFactory implements FallbackFactory<UserClient> {
    @Override
    public UserClient create(Throwable cause) {
        return id -> {
            log.error("调用用户服务失败, userId={}, error={}", id, cause.getMessage());
            return Result.fail(503, "用户服务暂时不可用");
        };
    }
}

// Feign 客户端：调用商品服务
@FeignClient(name = "product-service", path = "/products",
             fallbackFactory = ProductClientFallbackFactory.class)
public interface ProductClient {
    @GetMapping("/{id}")
    Result<ProductVO> getProduct(@PathVariable Long id);

    @PostMapping("/{id}/deduct-stock")
    Result<Boolean> deductStock(@PathVariable Long id, @RequestParam int quantity);
}

// 订单实体
@Entity
@Table(name = "t_order")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "order_no", unique = true)
    private String orderNo;

    @Column(name = "user_id", nullable = false)
    private Long userId;

    @Column(name = "product_id", nullable = false)
    private Long productId;

    @Column(name = "product_name")
    private String productName;

    @Column(nullable = false)
    private Integer quantity;

    @Column(name = "unit_price", nullable = false)
    private BigDecimal unitPrice;

    @Column(name = "total_amount", nullable = false)
    private BigDecimal totalAmount;

    @Enumerated(EnumType.STRING)
    private OrderStatus status = OrderStatus.PENDING;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @PrePersist
    void onCreate() {
        createdAt = LocalDateTime.now();
        if (orderNo == null) {
            orderNo = "ORD" + System.currentTimeMillis();
        }
    }
}

public enum OrderStatus {
    PENDING,    // 待支付
    PAID,       // 已支付
    SHIPPED,    // 已发货
    COMPLETED,  // 已完成
    CANCELLED   // 已取消
}

// 订单服务
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
@Slf4j
public class OrderService {

    private final OrderRepository orderRepository;
    private final UserClient userClient;
    private final ProductClient productClient;

    @Transactional
    public Order createOrder(Long userId, Long productId, int quantity) {
        // 1. 验证用户
        Result<UserVO> userResult = userClient.getUser(userId);
        if (!userResult.isSuccess()) {
            throw new BizException("用户验证失败: " + userResult.getMessage());
        }

        // 2. 获取商品信息
        Result<ProductVO> productResult = productClient.getProduct(productId);
        if (!productResult.isSuccess()) {
            throw new BizException("商品不存在: " + productId);
        }
        ProductVO product = productResult.getData();

        // 3. 扣减库存
        Result<Boolean> deductResult = productClient.deductStock(productId, quantity);
        if (!deductResult.isSuccess() || !Boolean.TRUE.equals(deductResult.getData())) {
            throw new BizException("库存不足");
        }

        // 4. 创建订单
        BigDecimal totalAmount = product.getPrice().multiply(BigDecimal.valueOf(quantity));
        Order order = Order.builder()
            .userId(userId)
            .productId(productId)
            .productName(product.getName())
            .quantity(quantity)
            .unitPrice(product.getPrice())
            .totalAmount(totalAmount)
            .build();

        order = orderRepository.save(order);
        log.info("订单创建成功: orderNo={}, userId={}, productId={}",
                 order.getOrderNo(), userId, productId);
        return order;
    }

    public Order getByOrderNo(String orderNo) {
        return orderRepository.findByOrderNo(orderNo)
            .orElseThrow(() -> new BizException(404, "订单不存在: " + orderNo));
    }

    public Page<Order> getUserOrders(Long userId, int page, int size) {
        return orderRepository.findByUserIdOrderByCreatedAtDesc(
            userId, PageRequest.of(page, size));
    }
}

// 订单 Controller
@RestController
@RequestMapping("/orders")
@RequiredArgsConstructor
public class OrderController {

    private final OrderService orderService;

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Result<OrderVO> createOrder(@RequestBody @Valid CreateOrderRequest req,
                                        @RequestHeader("X-User-Id") Long userId) {
        Order order = orderService.createOrder(userId, req.getProductId(), req.getQuantity());
        return Result.ok(OrderVO.from(order));
    }

    @GetMapping("/{orderNo}")
    public Result<OrderVO> getOrder(@PathVariable String orderNo) {
        return Result.ok(OrderVO.from(orderService.getByOrderNo(orderNo)));
    }

    @GetMapping("/my")
    public Result<Page<OrderVO>> myOrders(
            @RequestHeader("X-User-Id") Long userId,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        Page<Order> orders = orderService.getUserOrders(userId, page, size);
        return Result.ok(orders.map(OrderVO::from));
    }
}
```

### 4.8 网关服务 (shop-gateway)

```yaml
# application.yml
spring:
  application:
    name: gateway-service
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
    gateway:
      routes:
        - id: user-route
          uri: lb://user-service
          predicates:
            - Path=/api/users/**
          filters:
            - RewritePath=/api/users/(?<path>.*), /users/$\{path}

        - id: product-route
          uri: lb://product-service
          predicates:
            - Path=/api/products/**
          filters:
            - RewritePath=/api/products/(?<path>.*), /products/$\{path}

        - id: order-route
          uri: lb://order-service
          predicates:
            - Path=/api/orders/**
          filters:
            - RewritePath=/api/orders/(?<path>.*), /orders/$\{path}

server:
  port: 9000

# 白名单路径（不需要鉴权）
security:
  white-list:
    - /api/users/login
    - /api/users/register
    - /actuator/**
```

### 4.9 服务调用流程追踪

```
用户下单完整调用链：

1. 客户端 POST /api/orders  (携带 JWT Token)
        |
        v
2. Gateway 鉴权过滤器
   - 验证 JWT Token
   - 提取 userId
   - 添加 X-User-Id 请求头
        |
        v
3. order-service 接收请求
   - 从请求头获取 userId
        |
        +---> 4. Feign 调用 user-service GET /users/{userId}
        |          - Nacos 获取 user-service 实例列表
        |          - LoadBalancer 选择实例（轮询）
        |          - 发送 HTTP 请求
        |          - 返回用户信息
        |
        +---> 5. Feign 调用 product-service GET /products/{productId}
        |          - 返回商品信息
        |
        +---> 6. Feign 调用 product-service POST /products/{id}/deduct-stock
        |          - 扣减库存（乐观锁）
        |
        v
7. 创建订单并保存到数据库
        |
        v
8. 返回订单信息 --> Gateway --> 客户端
```

---

## 5. 常见问题 FAQ

### Q1：Spring Boot 启动报 "No qualifying bean of type"

**原因**：Spring 找不到需要注入的 Bean。

**常见原因及解决方法**：

```
原因1：类没有加 @Component/@Service 等注解
解决：检查类是否正确加了组件注解

原因2：包路径不在 @ComponentScan 扫描范围
解决：
  - 将类移到启动类的同级或子包下
  - 或在 @ComponentScan 中手动指定包路径
  @SpringBootApplication(scanBasePackages = {"com.example.main", "com.other.pkg"})

原因3：接口有多个实现，未用 @Qualifier 指定
解决：
  @Autowired
  @Qualifier("specificImplBeanName")
  private MyInterface myInterface;

原因4：@ConditionalOnXxx 条件不满足，Bean 未创建
解决：添加 debug=true 查看自动配置报告
```

---

### Q2：Spring 循环依赖报错

Spring Boot 3.x 默认不允许循环依赖。

```
错误信息：
The dependencies of some of the beans in the application context
form a cycle: beanA -> beanB -> beanA
```

**解决方案（按优先级）**：

```java
// 方案1（最佳）：重新设计，消除循环依赖
// 通常是设计问题，提取公共依赖到第三个类

// 方案2：使用 @Lazy 延迟加载
@Service
public class BeanA {
    @Autowired
    @Lazy  // 延迟注入，打破循环
    private BeanB beanB;
}

// 方案3（不推荐）：允许循环依赖
// application.yml
// spring.main.allow-circular-references: true
```

---

### Q3：Feign 调用超时如何排查？

```
排查步骤：

1. 确认服务已注册到 Nacos
   GET http://localhost:8848/nacos/v1/ns/instance/list?serviceName=user-service

2. 检查 Feign 超时配置
   spring.cloud.openfeign.client.config.default.read-timeout=10000

3. 开启 Feign 详细日志
   logging.level.com.example.feign: DEBUG
   spring.cloud.openfeign.client.config.default.logger-level: FULL

4. 检查服务端响应时间
   - 查看目标服务日志
   - 添加慢查询日志

5. 检查网络连通性
   curl http://user-service-host:8081/actuator/health
```

---

### Q4：Nacos 注册服务后，其他服务发现不到

```
排查清单：

1. 确认 Nacos 地址配置正确
   spring.cloud.nacos.discovery.server-addr=localhost:8848

2. 确认所有服务使用相同的 namespace 和 group
   spring.cloud.nacos.discovery.namespace=dev
   spring.cloud.nacos.discovery.group=DEFAULT_GROUP

3. 检查防火墙，确保 8848、9848、9849 端口开放

4. 检查服务是否成功注册
   Nacos 控制台 -> 服务管理 -> 服务列表

5. 确认 spring.application.name 正确，不含特殊字符
```

---

### Q5：Gateway 跨域（CORS）问题

```yaml
# Gateway 统一处理跨域（不要在微服务中重复设置）
spring:
  cloud:
    gateway:
      globalcors:
        cors-configurations:
          '[/**]':
            allowedOriginPatterns:
              - "https://your-frontend.com"
              - "http://localhost:3000"
            allowedMethods:
              - GET
              - POST
              - PUT
              - DELETE
              - OPTIONS
            allowedHeaders: "*"
            allowCredentials: true
            maxAge: 3600
```

> **注意**：不要同时在 Gateway 和下游服务都配置 CORS，会导致重复 header 报错。

---

### Q6：@Transactional 事务不生效

```java
// 常见原因：

// 1. 在同一个类中调用带 @Transactional 的方法（自调用问题）
@Service
public class UserService {
    public void methodA() {
        this.methodB();  // 错误：this 调用不走代理，事务不生效
    }

    @Transactional
    public void methodB() { ... }
}

// 解决方案：注入自身代理
@Service
public class UserService {
    @Autowired
    private UserService self;  // 注入自身代理

    public void methodA() {
        self.methodB();  // 正确：通过代理调用
    }

    @Transactional
    public void methodB() { ... }
}

// 2. 事务方法不是 public（Spring AOP 只代理 public 方法）
// 解决：改为 public 方法

// 3. 异常被 catch 吞掉，没有抛出
@Transactional
public void createUser() {
    try {
        userRepository.save(user);
    } catch (Exception e) {
        log.error("error", e);
        // 错误：异常被吞掉，事务不会回滚
    }
}
// 解决：捕获后重新抛出，或使用 TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();

// 4. rollbackFor 配置不当（默认只回滚 RuntimeException）
@Transactional(rollbackFor = Exception.class)  // 所有异常都回滚
public void createUser() throws Exception { ... }
```

---

### Q7：Sentinel 规则在重启后丢失

Sentinel 默认将规则保存在内存中，服务重启后丢失。生产环境需持久化到 Nacos：

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        # 流控规则
        flow:
          nacos:
            server-addr: localhost:8848
            namespace: prod
            data-id: ${spring.application.name}-flow-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: flow
        # 熔断规则
        degrade:
          nacos:
            server-addr: localhost:8848
            namespace: prod
            data-id: ${spring.application.name}-degrade-rules
            group-id: SENTINEL_GROUP
            data-type: json
            rule-type: degrade
```

在 Nacos 中创建对应 Data ID，内容为 JSON 格式规则数组：

```json
[
  {
    "resource": "getOrder",
    "grade": 1,
    "count": 100,
    "strategy": 0,
    "controlBehavior": 0,
    "clusterMode": false
  }
]
```

---

### Q8：如何优雅停机？

```yaml
# Spring Boot 配置优雅停机
server:
  shutdown: graceful    # 开启优雅停机

spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s  # 最长等待 30 秒
```

```bash
# 发送停机信号
kill -15 <pid>          # SIGTERM，触发优雅停机
# 或通过 Actuator
curl -X POST http://localhost:8080/actuator/shutdown
```

---

### Q9：如何实现接口幂等性？

```java
// 方案：基于 Redis + 自定义注解实现幂等

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    long expireSeconds() default 300;  // 幂等有效期
}

@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class IdempotentAspect {

    private final RedisTemplate<String, String> redisTemplate;

    @Around("@annotation(idempotent)")
    public Object checkIdempotent(ProceedingJoinPoint pjp, Idempotent idempotent) throws Throwable {
        // 从请求头获取幂等 Token
        HttpServletRequest request = ((ServletRequestAttributes)
            RequestContextHolder.getRequestAttributes()).getRequest();
        String idempotentKey = request.getHeader("X-Idempotent-Key");

        if (idempotentKey == null || idempotentKey.isBlank()) {
            throw new BizException(400, "缺少幂等 Key（X-Idempotent-Key 请求头）");
        }

        String redisKey = "idempotent:" + idempotentKey;
        // setIfAbsent：如果不存在才设置（原子操作）
        Boolean isFirstRequest = redisTemplate.opsForValue()
            .setIfAbsent(redisKey, "processing", Duration.ofSeconds(idempotent.expireSeconds()));

        if (!Boolean.TRUE.equals(isFirstRequest)) {
            throw new BizException(409, "重复请求，请勿重复提交");
        }

        try {
            return pjp.proceed();
        } catch (Throwable e) {
            // 执行失败，删除 key 允许重试
            redisTemplate.delete(redisKey);
            throw e;
        }
    }
}

// 使用
@PostMapping("/orders")
@Idempotent(expireSeconds = 600)
public Result<OrderVO> createOrder(@RequestBody CreateOrderRequest req) {
    // ...
}
```

---

### Q10：微服务链路追踪如何配置？

**使用 Micrometer Tracing + Zipkin**

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-brave</artifactId>
</dependency>
<dependency>
    <groupId>io.zipkin.reporter2</groupId>
    <artifactId>zipkin-reporter-brave</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0   # 采样率 100%（生产环境建议 0.1）
  zipkin:
    tracing:
      endpoint: http://zipkin-server:9411/api/v2/spans

logging:
  pattern:
    # 在日志中输出 traceId 和 spanId
    console: "%d{HH:mm:ss} [%thread] %-5level [%X{traceId},%X{spanId}] %logger{36} - %msg%n"
```

---

## 附录：版本兼容性速查

### Spring Boot 3.x 重要变化

| 变化 | Spring Boot 2.x | Spring Boot 3.x |
|------|-----------------|-----------------|
| Java 最低版本 | Java 8 | Java 17 |
| Jakarta EE | javax.* | jakarta.* |
| 自动配置文件 | spring.factories | AutoConfiguration.imports |
| Actuator 默认 | 部分开放 | 仅 health 开放 |
| Spring Security | WebSecurityConfigurerAdapter | SecurityFilterChain Bean |

### 包名迁移（javax -> jakarta）

```java
// Spring Boot 2.x
import javax.persistence.Entity;
import javax.validation.Valid;
import javax.servlet.http.HttpServletRequest;

// Spring Boot 3.x
import jakarta.persistence.Entity;
import jakarta.validation.Valid;
import jakarta.servlet.http.HttpServletRequest;
```

---

## 总结

```
Spring 全家桶学习路径建议：

入门阶段：
  1. 理解 IoC/DI 核心概念
  2. 掌握 Spring Boot 快速开发
  3. 熟悉常用注解和配置

进阶阶段：
  4. 深入理解 AOP 和事务
  5. 学习 Spring Data JPA/MyBatis
  6. 掌握 Spring Security

微服务阶段：
  7. 搭建 Nacos 注册/配置中心
  8. 使用 OpenFeign 服务间调用
  9. Gateway 网关统一路由
  10. Sentinel 保障系统稳定性

生产实践：
  11. 分布式事务（Seata）
  12. 消息队列（RabbitMQ/RocketMQ）
  13. 链路追踪（Zipkin/SkyWalking）
  14. 容器化部署（Docker + K8s）
```

---

*文档最后更新：2024年*
*参考版本：Spring Boot 3.2 · Spring Cloud 2023.0.1 · Spring Cloud Alibaba 2023.0.1.0*