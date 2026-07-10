# Spring 源码深度解析 - 从零到精通

> 版本覆盖：Spring Framework 5.x / 6.x
> 分析层次：源码级，含关键代码片段与流程图
> 适用场景：架构设计、面试准备、技术深挖

---

## 目录

- [Part 1: Spring 整体架构与模块](#part-1)
- [Part 2: IoC 容器启动源码解析](#part-2)
- [Part 3: Bean 生命周期源码解析](#part-3)
- [Part 4: 循环依赖源码解析](#part-4)
- [Part 5: AOP 源码解析](#part-5)
- [Part 6: Spring MVC 源码解析](#part-6)
- [Part 7: Spring Boot 自动配置源码解析](#part-7)
- [Part 8: Spring 事件机制源码](#part-8)
- [Part 9: 常用扩展点汇总](#part-9)
- [Part 10: 高频面试题 FAQ（源码级）](#part-10)

---

# Part 1: Spring 整体架构与模块 {#part-1}

## 1.1 Spring 模块划分

Spring Framework 是一个分层的、模块化的框架，各模块之间既相互独立又相互协作。下图展示了 Spring 的完整模块体系：

```
+-----------------------------------------------------------------------------------+
|                          Spring Framework 模块全景图                               |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|   +----------------+   +----------------+   +------------------------------+     |
|   |   spring-test  |   | spring-context |   |  spring-webmvc               |     |
|   |                |   |   -support     |   |  spring-webflux              |     |
|   | MockMvc        |   |                |   |  spring-websocket            |     |
|   | SpringRunner   |   | Caching        |   |  spring-web                  |     |
|   +----------------+   | Scheduling     |   +------------------------------+     |
|                        | Mail/Formatting|                                         |
|   +----------------------------------------------------------+                   |
|   |                   spring-context                          |                   |
|   |  ApplicationContext  EventPublisher  MessageSource        |                   |
|   |  ResourceLoader      Environment     LifecycleProcessor   |                   |
|   +----------------------------------------------------------+                   |
|                                                                                   |
|   +--------------------+   +--------------------+   +-------------------+        |
|   |    spring-aop      |   |   spring-aspects   |   |  spring-instrument |        |
|   |  ProxyFactory      |   | AspectJ Integration|   |  Load-time Weaving |        |
|   |  Advisor/Pointcut  |   |                    |   |                   |        |
|   +--------------------+   +--------------------+   +-------------------+        |
|                                                                                   |
|   +-------------------------------------------------------------------+          |
|   |                      spring-beans                                  |          |
|   |  BeanFactory  BeanDefinition  BeanPostProcessor  PropertyEditor   |          |
|   +-------------------------------------------------------------------+          |
|                                                                                   |
|   +----------------------------+   +---------------------------------------+      |
|   |       spring-core          |   |           spring-expression           |      |
|   |  Resource  IO  Codec       |   |  SpEL - Spring Expression Language    |      |
|   |  Converter  Utils          |   |  #{expression} 支持                   |      |
|   +----------------------------+   +---------------------------------------+      |
|                                                                                   |
|   +-----------------+   +-----------------+   +-----------------+                |
|   |   spring-jdbc   |   |   spring-tx     |   |   spring-orm    |                |
|   |  JdbcTemplate   |   |  Transaction    |   |  Hibernate/JPA  |                |
|   |  DataSource     |   |  Manager        |   |  MyBatis        |                |
|   +-----------------+   +-----------------+   +-----------------+                |
+-----------------------------------------------------------------------------------+
```

### 各模块职责详解

**spring-core**（基础核心）
- `Resource` 接口体系：统一资源抽象（ClassPathResource / FileSystemResource / UrlResource）
- `ResolvableType`：泛型类型解析工具
- `ReflectionUtils`：反射工具类
- `Assert`：断言工具
- `StringUtils / CollectionUtils / ObjectUtils`：通用工具

**spring-beans**（Bean 定义与管理）
- `BeanFactory` 接口体系：IoC 容器的最顶层抽象
- `BeanDefinition`：Bean 的元数据描述
- `BeanPostProcessor`：Bean 初始化前后的拦截器
- `PropertyEditor / ConversionService`：类型转换

**spring-context**（应用上下文）
- `ApplicationContext`：功能更丰富的 IoC 容器
- `ApplicationEvent / ApplicationListener`：事件发布/订阅
- `MessageSource`：国际化支持
- `@Component / @Autowired / @Value` 等注解处理

**spring-aop**（面向切面编程）
- `Advisor / Pointcut / Advice`：AOP 核心概念
- `ProxyFactory`：代理工厂
- JDK 动态代理 + CGLIB 两种代理方式

**spring-tx**（事务管理）
- `PlatformTransactionManager`：事务管理器接口
- `TransactionDefinition`：事务定义（传播行为、隔离级别）
- `@Transactional` 注解驱动事务

**spring-web / spring-webmvc**（Web 层）
- `DispatcherServlet`：前端控制器
- `HandlerMapping / HandlerAdapter`：请求路由与适配
- `@RequestMapping / @RestController` 注解处理

---

## 1.2 Spring 版本演进

### 版本演进时间线

```
Spring 版本演进
===============

2003          2009          2013          2017          2022
  |             |             |             |             |
  v             v             v             v             v
+------+    +------+    +------+    +------+    +------+
| 1.x  |    | 3.x  |    | 4.x  |    | 5.x  |    | 6.x  |
| XML  |--->| Anno |--->| Java8|--->|Reactive--->|AOT   |
| 配置  |    | 驱动  |    | 支持  |    |WebFlux|    |GraalVM|
+------+    +------+    +------+    +------+    +------+
               |             |           |           |
            @Component   @Conditional  Reactor    虚拟线程
            @Autowired   泛型注入       Project   (Java21)
            JavaConfig   Lambda支持    支持
```

### Spring 3.x 关键变化（2009-2013）

```java
// Spring 3.0: JavaConfig 正式可用
@Configuration
public class AppConfig {
    @Bean
    public UserService userService() {
        return new UserServiceImpl();
    }
}

// Spring 3.1: @Profile 条件配置
@Configuration
@Profile("production")
public class ProductionConfig { ... }

// Spring 3.2: @ControllerAdvice 全局异常处理
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(Exception.class)
    public ResponseEntity<?> handleException(Exception e) { ... }
}
```

### Spring 4.x 关键变化（2013-2017）

```java
// Spring 4.0: 泛型依赖注入
@Autowired
private Repository<User> userRepository; // 精确匹配泛型类型

// Spring 4.0: @Conditional 条件注解基础
@Conditional(OnProductionCondition.class)
@Bean
public DataSource productionDataSource() { ... }

// Spring 4.1: @EventListener 注解
@EventListener
public void handleContextRefreshed(ContextRefreshedEvent event) { ... }

// Spring 4.3: @GetMapping / @PostMapping 快捷注解
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) { ... }

// Spring 4.x: 构造注入不再需要 @Autowired（单构造）
public class UserService {
    private final UserRepository repository;

    // 单构造函数，Spring 4.3+ 自动注入，无需 @Autowired
    public UserService(UserRepository repository) {
        this.repository = repository;
    }
}
```

### Spring 5.x 关键变化（2017-2022）

```java
// Spring 5.0: 响应式编程支持 (WebFlux)
@RestController
public class ReactiveController {
    @GetMapping("/users")
    public Flux<User> getUsers() {
        return userRepository.findAll(); // 非阻塞流
    }

    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userRepository.findById(id);
    }
}

// Spring 5.0: 函数式 Bean 注册（无注解）
GenericApplicationContext ctx = new GenericApplicationContext();
ctx.registerBean(UserService.class, () -> new UserServiceImpl());
ctx.refresh();

// Spring 5.2: @Lazy 注解在注入点（延迟代理）
@Autowired
@Lazy
private HeavyService heavyService; // 首次调用时才初始化
```

### Spring 6.x 关键变化（2022+）

```java
// Spring 6.0: 基于 Java 17+，雅加达 EE 9（javax -> jakarta）
import jakarta.servlet.http.HttpServletRequest; // 包名变化
import jakarta.persistence.Entity;

// Spring 6.0: AOT（提前编译）支持，为 GraalVM Native Image 准备
@Component
public class MyHintsRegistrar implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(UserService.class,
            MemberCategory.INVOKE_DECLARED_METHODS);
    }
}

// Spring 6.1: 虚拟线程（Java 21 Project Loom）支持
@Configuration
public class ThreadConfig {
    @Bean
    public TomcatProtocolHandlerCustomizer<?> virtualThreadCustomizer() {
        return handler -> handler.setExecutor(
            Executors.newVirtualThreadPerTaskExecutor()
        );
    }
}
```

---

## 1.3 Spring 核心设计思想

### IoC（控制反转）与 DI（依赖注入）

```
传统编程：对象自己创建依赖
+------------------+
|   UserService    |
|  UserRepo repo = new UserRepoImpl()  <-- 主动创建
+------------------+

Spring IoC：容器控制对象的创建和依赖关系
+------------------+        +------------------+
|   UserService    |        |  Spring IoC 容器  |
|                  |<-------|                  |
|  @Autowired      |  注入   | UserRepository   |
|  UserRepository  |        | UserService      |
+------------------+        +------------------+
       对象只声明需要什么，容器负责提供
```

**控制反转原则的本质**：

```java
// 传统方式：高层模块依赖低层模块的具体实现
public class OrderService {
    // 直接 new，强耦合
    private UserRepository userRepository = new MysqlUserRepository();
    private EmailService emailService = new SmtpEmailService();
}

// IoC 方式：依赖抽象，由容器注入
public class OrderService {
    private final UserRepository userRepository;   // 接口
    private final EmailService emailService;        // 接口

    // 构造注入，测试时可轻易替换实现
    public OrderService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

### 面向接口编程

```
具体实现可替换：
BeanFactory (接口)
    +-- DefaultListableBeanFactory  (XML/注解场景)
    +-- StaticListableBeanFactory   (测试场景)
    +-- 自定义实现...

PlatformTransactionManager (接口)
    +-- DataSourceTransactionManager  (JDBC/MyBatis)
    +-- JpaTransactionManager         (JPA/Hibernate)
    +-- JtaTransactionManager         (分布式事务)
    +-- 自定义实现...
```

### AOP（面向切面编程）

```java
// 横切关注点：日志、事务、权限等逻辑与业务逻辑分离

// 业务逻辑（不含横切逻辑）
@Service
public class OrderService {
    public void createOrder(Order order) {
        orderRepository.save(order);
    }
}

// 横切逻辑（独立维护）
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        log.info("方法调用前：{}", joinPoint.getSignature());
        Object result = joinPoint.proceed();
        log.info("方法调用后：{}", result);
        return result;
    }
}
```

---

## 1.4 源码阅读环境搭建

### Gradle 构建 Spring 源码

```bash
# 1. 克隆源码
git clone https://github.com/spring-projects/spring-framework.git
cd spring-framework

# 2. 切换到稳定版本标签
git checkout v5.3.27  # 或 v6.0.x

# 3. 预编译 spring-oxm（必须先执行）
./gradlew :spring-oxm:compileTestJava

# 4. 构建整个项目
./gradlew build -x test

# 5. 生成 IDEA 工程文件
./gradlew idea
```

### IDEA 导入技巧

```
IDEA 导入 Spring 源码步骤：
1. File -> Open -> 选择 spring-framework 目录
2. 等待 Gradle 同步（可能需要 10-30 分钟）
3. 设置编译器：Preferences -> Build -> Compiler ->
   勾选 "Enable annotation processing"
4. 添加 coroutines 依赖（如编译失败）：
   在 build.gradle 中检查 kotlin 版本
5. 创建 spring-test-my 模块，引入需要分析的模块作为依赖
   方便添加断点调试
```

### 推荐调试方式

```java
// 在自己的测试模块中创建如下测试类
public class SpringIoCTest {
    public static void main(String[] args) {
        // 在 refresh() 方法的关键步骤打断点
        AnnotationConfigApplicationContext ctx =
            new AnnotationConfigApplicationContext(AppConfig.class);

        UserService userService = ctx.getBean(UserService.class);
        userService.doSomething();

        ctx.close();
    }
}
```

---

# Part 2: IoC 容器启动源码解析 {#part-2}

## 2.1 BeanFactory 体系

BeanFactory 是 Spring IoC 容器的顶层接口，定义了容器的最基本行为。

```
BeanFactory 接口继承体系（完整）
=================================

                    BeanFactory
                    (顶层接口)
                         |
          +--------------+--------------+
          |                             |
  HierarchicalBeanFactory    ListableBeanFactory
  (父子容器：getParentBF)     (批量查询：getBeanNamesForType)
          |                             |
          +-------------+---------------+
                        |
               ConfigurableBeanFactory
               (配置能力：registerSingleton/addBeanPostProcessor)
                        |
          +-------------+-------------+
          |                           |
  ConfigurableListableBeanFactory  AutowireCapableBeanFactory
  (最完整的配置接口)                (自动装配：createBean/autowireBean)
          |
          v
  DefaultListableBeanFactory
  (Spring 最核心的 BeanFactory 实现)
  实现了上述所有接口
```


---

# Part 2: IoC 容器启动源码解析 {#part-2}

## 2.1 BeanFactory 体系

BeanFactory 是 Spring IoC 容器的根接口，定义了最基本的 Bean 获取协议。

```
BeanFactory（顶级接口）
    │
    ├── HierarchicalBeanFactory（支持父子容器）
    │       └── ConfigurableBeanFactory（可配置）
    │               └── ConfigurableListableBeanFactory（可列举+可配置）
    │                       └── DefaultListableBeanFactory ★ 核心实现
    │
    ├── ListableBeanFactory（可列举所有Bean）
    │
    └── AutowireCapableBeanFactory（支持自动装配）
```

### DefaultListableBeanFactory 核心字段

```java
public class DefaultListableBeanFactory
        extends AbstractAutowireCapableBeanFactory
        implements ConfigurableListableBeanFactory, BeanDefinitionRegistry {

    // BeanDefinition 注册表：beanName -> BeanDefinition
    private final Map<String, BeanDefinition> beanDefinitionMap
            = new ConcurrentHashMap<>(256);

    // 保证注册顺序
    private volatile List<String> beanDefinitionNames = new ArrayList<>(256);

    // 按类型索引：type -> [beanName...]
    private final Map<Class<?>, String[]> allBeanNamesByType
            = new ConcurrentHashMap<>(64);

    // 单例 Bean 类型缓存
    private final Map<Class<?>, String[]> singletonBeanNamesByType
            = new ConcurrentHashMap<>(64);

    // 手动注册的单例（resolvableDependencies）
    private final Map<Class<?>, Object> resolvableDependencies
            = new ConcurrentHashMap<>(16);
}
```

### BeanDefinition 体系

```
BeanDefinition（接口）
    │
    ├── AbstractBeanDefinition（抽象基类，含大部分属性）
    │       ├── GenericBeanDefinition（通用，XML解析默认产物）
    │       ├── RootBeanDefinition    ★ 最终合并结果
    │       └── ChildBeanDefinition（继承父BeanDefinition）
    │
    └── AnnotatedBeanDefinition（注解元数据扩展）
            └── ScannedGenericBeanDefinition（@Component扫描产物）
            └── AnnotatedGenericBeanDefinition（@Configuration产物）
```

关键属性一览：

```java
public abstract class AbstractBeanDefinition implements BeanDefinition {
    private volatile Object beanClass;      // Bean的Class或className
    private String scope = SCOPE_DEFAULT;   // singleton / prototype
    private boolean lazyInit = false;       // 是否懒加载
    private String[] dependsOn;             // depends-on
    private boolean autowireCandidate = true; // 是否参与自动装配
    private boolean primary = false;        // 是否为首选Bean
    private ConstructorArgumentValues constructorArgumentValues; // 构造参数
    private MutablePropertyValues propertyValues;  // 属性值
    private String initMethodName;          // init-method
    private String destroyMethodName;       // destroy-method
    private int role = BeanDefinition.ROLE_APPLICATION;
    // ROLE_APPLICATION=0, ROLE_SUPPORT=1, ROLE_INFRASTRUCTURE=2
}
```

---

## 2.2 ApplicationContext 体系

```
ApplicationContext
    │
    ├── ConfigurableApplicationContext（可刷新、可关闭）
    │       └── AbstractApplicationContext ★ 模板方法核心
    │               ├── AbstractRefreshableApplicationContext
    │               │       └── AbstractXmlApplicationContext
    │               │               └── ClassPathXmlApplicationContext
    │               │               └── FileSystemXmlApplicationContext
    │               │
    │               └── GenericApplicationContext（通用，持有DefaultListableBeanFactory）
    │                       └── AnnotationConfigApplicationContext ★ 注解配置入口
    │                       └── GenericWebApplicationContext
    │                               └── AnnotationConfigServletWebServerApplicationContext（Boot）
    │
    └── WebApplicationContext（Web环境扩展）
```

ApplicationContext 相较于 BeanFactory 增加了以下能力：

| 能力 | 描述 |
|------|------|
| `MessageSource` | 国际化支持 |
| `ApplicationEventPublisher` | 事件发布/监听 |
| `ResourcePatternResolver` | 资源加载（classpath*:） |
| `EnvironmentCapable` | 环境与配置文件（profiles） |
| `LifecycleProcessor` | Lifecycle Bean 管理 |
| `BeanFactoryPostProcessor` | BeanDefinition 级后处理 |
| `BeanPostProcessor` | Bean 实例级后处理 |

---

## 2.3 refresh() —— 容器启动的核心入口

`AbstractApplicationContext.refresh()` 是整个 Spring 容器启动的核心，共 **12 个步骤**。

```java
// AbstractApplicationContext.java
@Override
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {

        // 1. 准备刷新
        prepareRefresh();

        // 2. 获取（创建）BeanFactory，加载BeanDefinition
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 3. 对BeanFactory进行标准化配置
        prepareBeanFactory(beanFactory);

        try {
            // 4. 子类扩展点：BeanFactory准备完毕后的后置处理
            postProcessBeanFactory(beanFactory);

            // 5. 执行所有 BeanFactoryPostProcessor
            invokeBeanFactoryPostProcessors(beanFactory);

            // 6. 注册所有 BeanPostProcessor
            registerBeanPostProcessors(beanFactory);

            // 7. 初始化 MessageSource（国际化）
            initMessageSource();

            // 8. 初始化事件广播器
            initApplicationEventMulticaster();

            // 9. 子类扩展点：特殊Bean初始化（如Web容器创建Servlet）
            onRefresh();

            // 10. 注册监听器
            registerListeners();

            // 11. ★★★ 初始化所有非懒加载单例Bean
            finishBeanFactoryInitialization(beanFactory);

            // 12. 完成刷新，发布ContextRefreshedEvent
            finishRefresh();
        }
        catch (BeansException ex) {
            destroyBeans();
            cancelRefresh(ex);
            throw ex;
        }
        finally {
            resetCommonCaches();
        }
    }
}
```

### Step 1: prepareRefresh()

```java
protected void prepareRefresh() {
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    // 子类可重写，初始化PropertySource（如Web环境加载ServletContext参数）
    initPropertySources();

    // 验证必要属性是否存在（setRequiredProperties配置的）
    getEnvironment().validateRequiredProperties();

    // 保存早期事件（此时multicaster还未创建）
    this.earlyApplicationEvents = new LinkedHashSet<>();
}
```

### Step 2: obtainFreshBeanFactory()

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    return getBeanFactory();
}

// AbstractRefreshableApplicationContext（XML场景）会真正销毁+重建BeanFactory
@Override
protected final void refreshBeanFactory() throws BeansException {
    if (hasBeanFactory()) {
        destroyBeans();
        closeBeanFactory();
    }
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    loadBeanDefinitions(beanFactory);  // 解析XML或扫描注解，注册BeanDefinition
    this.beanFactory = beanFactory;
}
```

### Step 3: prepareBeanFactory()

```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    beanFactory.setBeanClassLoader(getClassLoader());
    beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver());
    beanFactory.addPropertyEditorRegistrar(
        new ResourceEditorRegistrar(this, getEnvironment()));

    // ★ 注册 ApplicationContextAwareProcessor（处理各种 Aware 接口）
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

    // 忽略以下接口的自动装配（由Aware回调注入）
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 注册可解析依赖（直接按类型注入时使用）
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

    // 注册 ApplicationListenerDetector（检测 ApplicationListener Bean）
    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 注册默认环境Bean
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
}
```

### Step 5: invokeBeanFactoryPostProcessors() ★★★

```java
// PostProcessorRegistrationDelegate.java
public static void invokeBeanFactoryPostProcessors(
        ConfigurableListableBeanFactory beanFactory,
        List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    Set<String> processedBeans = new HashSet<>();

    if (beanFactory instanceof BeanDefinitionRegistry registry) {
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor rdpp) {
                rdpp.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(rdpp);
            } else {
                regularPostProcessors.add(postProcessor);
            }
        }

        // 1. 实现 PriorityOrdered 的 BeanDefinitionRegistryPostProcessor
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();
        String[] ppNames = beanFactory.getBeanNamesForType(
                BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : ppNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                currentRegistryProcessors.add(
                        beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                processedBeans.add(ppName);
            }
        }
        sortPostProcessors(currentRegistryProcessors, beanFactory);
        // ★ 此处执行 ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
        invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
    }
}
```

### ConfigurationClassPostProcessor 解析流程

```
parse(configClass)
    └── processConfigurationClass()
            ├── processMemberClasses()        // 处理内部类
            ├── processPropertySources()      // @PropertySource
            ├── processComponentScans()       // @ComponentScan → 扫描包，找@Component
            ├── processImports()              // @Import → ImportSelector/ImportBDRegistrar
            ├── processImportResource()       // @ImportResource
            ├── processBeanMethods()          // @Bean 方法
            └── processInterfaces()           // 接口上的 default @Bean 方法
```

### Step 11: finishBeanFactoryInitialization() ★★★

```java
protected void finishBeanFactoryInitialization(
        ConfigurableListableBeanFactory beanFactory) {
    // 冻结BeanDefinition（不再允许修改）
    beanFactory.freezeConfiguration();

    // ★★★ 初始化所有非懒加载单例Bean
    beanFactory.preInstantiateSingletons();
}
```

### preInstantiateSingletons() 源码

```java
// DefaultListableBeanFactory.java
@Override
public void preInstantiateSingletons() throws BeansException {
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    // 第一轮：实例化所有非懒加载单例
    for (String beanName : beanNames) {
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            if (isFactoryBean(beanName)) {
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName); // "&beanName"
                if (bean instanceof SmartFactoryBean<?> smartFactoryBean
                        && smartFactoryBean.isEagerInit()) {
                    getBean(beanName);
                }
            } else {
                getBean(beanName); // ★ 触发Bean创建
            }
        }
    }

    // 第二轮：触发 SmartInitializingSingleton.afterSingletonsInstantiated()
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton smartSingleton) {
            smartSingleton.afterSingletonsInstantiated();
        }
    }
}
```

---

## 2.4 refresh() 完整流程图

```
AnnotationConfigApplicationContext("com.example")
        │
        ▼
  构造函数
  ┌────────────────────────────────────────────────────────┐
  │ 1. new DefaultListableBeanFactory()                    │
  │ 2. new AnnotatedBeanDefinitionReader(this)             │
  │    └── 注册 ConfigurationClassPostProcessor 等基础BPP  │
  │ 3. new ClassPathBeanDefinitionScanner(this)            │
  │ 4. register(componentClasses)                          │
  │ 5. refresh()  ←─────────────────────── 关键！          │
  └────────────────────────────────────────────────────────┘
        │
        ▼
  refresh() 12步
  ┌─────────────────────────────────────────────────────────┐
  │  Step 1  prepareRefresh()         设置启动时间、验证属性 │
  │  Step 2  obtainFreshBeanFactory() 获取/刷新BeanFactory   │
  │  Step 3  prepareBeanFactory()     配置BeanFactory标准特性│
  │  Step 4  postProcessBeanFactory() 子类扩展              │
  │  Step 5  invokeBFPPs()      ★    ConfigClassPP扫描注解  │
  │  Step 6  registerBPPs()          注册所有BPP            │
  │  Step 7  initMessageSource()     初始化国际化           │
  │  Step 8  initEventMulticaster()  初始化事件广播器       │
  │  Step 9  onRefresh()             子类扩展               │
  │  Step 10 registerListeners()     注册监听器             │
  │  Step 11 finishBFInit()    ★★★  实例化所有非懒加载单例  │
  │  Step 12 finishRefresh()         发布ContextRefreshedEvent│
  └─────────────────────────────────────────────────────────┘
```


---

# Part 3: Bean 生命周期源码解析 {#part-3}

## 3.1 Bean 完整生命周期流程图

```
getBean(beanName)
        │
        ▼
  getSingleton()  ──→ 缓存命中？ ──YES──→ 返回缓存对象
        │NO
        ▼
  createBean()
        │
        ├── resolveBeforeInstantiation()
        │     └── InstantiationAwareBPP.postProcessBeforeInstantiation()
        │           └── 如果返回非null → postProcessAfterInitialization() → 直接返回（短路）
        │
        └── doCreateBean()
                │
                ├─ [1] createBeanInstance()          实例化（构造器/工厂方法）
                │
                ├─ [2] applyMergedBeanDefinitionPostProcessors()
                │       └── MergedBeanDefinitionPostProcessor（收集@Autowired等元数据）
                │
                ├─ [3] addSingletonFactory()          提前曝光（解决循环依赖）
                │       └── singletonFactories.put(beanName, () -> getEarlyBeanReference())
                │
                ├─ [4] populateBean()                 属性填充
                │       ├── InstantiationAwareBPP.postProcessAfterInstantiation()
                │       └── InstantiationAwareBPP.postProcessProperties()
                │             ├── AutowiredAnnotationBPP → @Autowired/@Value
                │             └── CommonAnnotationBPP → @Resource
                │
                └─ [5] initializeBean()               初始化
                        ├── invokeAwareMethods()
                        │     ├── BeanNameAware.setBeanName()
                        │     ├── BeanClassLoaderAware.setBeanClassLoader()
                        │     └── BeanFactoryAware.setBeanFactory()
                        ├── BPP.postProcessBeforeInitialization()
                        │     └── CommonAnnotationBPP → @PostConstruct
                        ├── invokeInitMethods()
                        │     ├── InitializingBean.afterPropertiesSet()
                        │     └── init-method
                        └── BPP.postProcessAfterInitialization()
                              └── AbstractAutoProxyCreator → 创建AOP代理 ★

        │ （单例：存入一级缓存）
        ▼
  Bean 就绪，可被使用
        │
        ▼ （容器关闭）
  destroyBean()
        ├── DestructionAwareBPP.postProcessBeforeDestruction()
        │     └── CommonAnnotationBPP → @PreDestroy
        ├── DisposableBean.destroy()
        └── destroy-method
```

---

## 3.2 doCreateBean() 核心源码

```java
// AbstractAutowireCapableBeanFactory.java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd,
        Object[] args) throws BeanCreationException {

    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }

    // [1] 实例化
    if (instanceWrapper == null) {
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    Object bean = instanceWrapper.getWrappedInstance();

    // [2] 收集注解元数据（@Autowired字段/方法等）
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    // [3] 提前曝光（循环依赖处理）
    boolean earlySingletonExposure = (mbd.isSingleton()
            && this.allowCircularReferences
            && isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        addSingletonFactory(beanName,
                () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    Object exposedObject = bean;
    try {
        // [4] 属性填充
        populateBean(beanName, mbd, instanceWrapper);

        // [5] 初始化
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        throw new BeanCreationException(beanName, ex);
    }

    // 注册 DisposableBean
    registerDisposableBeanIfNecessary(beanName, bean, mbd);

    return exposedObject;
}
```

---

## 3.3 createBeanInstance() —— 实例化策略

```java
protected BeanWrapper createBeanInstance(String beanName,
        RootBeanDefinition mbd, Object[] args) {

    // 有工厂方法则使用工厂方法（@Bean方法场景）
    if (mbd.getFactoryMethodName() != null) {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // 由 SmartInstantiationAwareBPP 决定候选构造函数
    // AutowiredAnnotationBPP 会推选 @Autowired 标注的构造器
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(
            beanClass, beanName);
    if (ctors != null || mbd.hasConstructorArgumentValues()) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // 使用默认无参构造函数
    return instantiateBean(beanName, mbd);
}
```

实例化底层使用 `InstantiationStrategy`：

```
InstantiationStrategy（接口）
    └── SimpleInstantiationStrategy
            └── CglibSubclassingInstantiationStrategy ★
                （当Bean有方法需要被重写时使用CGLIB，如@Lookup）
```

---

## 3.4 populateBean() —— 属性填充

```java
protected void populateBean(String beanName, RootBeanDefinition mbd,
        BeanWrapper bw) {

    // postProcessAfterInstantiation 返回 false 可跳过属性填充
    if (hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp :
                getBeanPostProcessorCache().instantiationAware) {
            if (!bp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                return;
            }
        }
    }

    // XML autowire 属性处理（AUTOWIRE_BY_NAME / AUTOWIRE_BY_TYPE）
    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
        autowireByName(beanName, mbd, bw, newPvs);
    } else if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        autowireByType(beanName, mbd, bw, newPvs);
    }

    // ★ 注解方式注入（@Autowired/@Resource）
    if (hasInstantiationAwareBeanPostProcessors()) {
        for (InstantiationAwareBeanPostProcessor bp :
                getBeanPostProcessorCache().instantiationAware) {
            PropertyValues pvsToUse = bp.postProcessProperties(pvs,
                    bw.getWrappedInstance(), beanName);
            pvs = pvsToUse;
        }
    }

    // 应用属性值到Bean（XML中的<property>配置）
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

---

## 3.5 @Autowired 注入源码解析

```java
public class AutowiredAnnotationBeanPostProcessor
        implements SmartInstantiationAwareBeanPostProcessor,
                   MergedBeanDefinitionPostProcessor {

    // Step 1: 收集注入元数据
    @Override
    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition,
            Class<?> beanType, String beanName) {
        InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
        metadata.checkConfigMembers(beanDefinition);
    }

    // Step 2: 执行注入
    @Override
    public PropertyValues postProcessProperties(PropertyValues pvs,
            Object bean, String beanName) {
        InjectionMetadata metadata = findAutowiringMetadata(
                beanName, bean.getClass(), pvs);
        metadata.inject(bean, beanName, pvs); // ★ 执行注入
        return pvs;
    }
}
```

字段注入调用链：

```
AutowiredFieldElement.inject()
    └── beanFactory.resolveDependency(descriptor, beanName)
            └── doResolveDependency()
                    ├── resolveMultipleBeans()   // 处理 List/Map/数组
                    ├── findAutowireCandidates() // 按类型找候选Bean
                    │     └── 过滤 @Qualifier、@Primary 等
                    └── determineAutowireCandidate() // 最终决策
                          ├── @Primary 优先
                          ├── @Priority 次之
                          └── 按字段名匹配 beanName
```

---

## 3.6 initializeBean() —— 初始化阶段

```java
protected Object initializeBean(String beanName, Object bean,
        RootBeanDefinition mbd) {

    // [1] Aware 接口回调（BeanNameAware/BeanClassLoaderAware/BeanFactoryAware）
    invokeAwareMethods(beanName, bean);

    // [2] BeanPostProcessor.postProcessBeforeInitialization()
    //     → ApplicationContextAwareProcessor 注入 ApplicationContext 级 Aware
    //     → CommonAnnotationBPP 执行 @PostConstruct
    Object wrappedBean = applyBeanPostProcessorsBeforeInitialization(bean, beanName);

    // [3] InitializingBean.afterPropertiesSet() + init-method
    invokeInitMethods(beanName, wrappedBean, mbd);

    // [4] BeanPostProcessor.postProcessAfterInitialization()
    //     → AbstractAutoProxyCreator 创建 AOP 代理 ★★★
    wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

    return wrappedBean;
}
```

初始化方法优先级（从先到后）：

```
@PostConstruct  →  InitializingBean.afterPropertiesSet()  →  init-method
```

销毁方法优先级（从先到后）：

```
@PreDestroy  →  DisposableBean.destroy()  →  destroy-method
```


---

# Part 4: 循环依赖源码解析 {#part-4}

## 4.1 三级缓存机制

Spring 通过三个 Map 来解决单例 Bean 的循环依赖问题：

```java
// DefaultSingletonBeanRegistry.java

// 一级缓存：完全初始化完毕的单例Bean（成品）
private final Map<String, Object> singletonObjects
        = new ConcurrentHashMap<>(256);

// 二级缓存：早期暴露的Bean对象（可能是代理，已从三级缓存晋升）
private final Map<String, Object> earlySingletonObjects
        = new ConcurrentHashMap<>(16);

// 三级缓存：ObjectFactory（Bean工厂，用于生成早期引用）
private final Map<String, ObjectFactory<?>> singletonFactories
        = new HashMap<>(16);

// 记录正在创建中的Bean
private final Set<String> singletonsCurrentlyInCreation
        = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
```

---

## 4.2 getSingleton() 核心源码

```java
// DefaultSingletonBeanRegistry.java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 快速检查一级缓存（无锁）
    Object singletonObject = this.singletonObjects.get(beanName);
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 检查二级缓存
        singletonObject = this.earlySingletonObjects.get(beanName);
        if (singletonObject == null && allowEarlyReference) {
            synchronized (this.singletonObjects) {
                // 双重检查（加锁后再次检查）
                singletonObject = this.singletonObjects.get(beanName);
                if (singletonObject == null) {
                    singletonObject = this.earlySingletonObjects.get(beanName);
                    if (singletonObject == null) {
                        // ★ 从三级缓存获取ObjectFactory，调用getObject()
                        ObjectFactory<?> singletonFactory =
                                this.singletonFactories.get(beanName);
                        if (singletonFactory != null) {
                            singletonObject = singletonFactory.getObject();
                            // 晋升到二级缓存
                            this.earlySingletonObjects.put(beanName, singletonObject);
                            // 移出三级缓存
                            this.singletonFactories.remove(beanName);
                        }
                    }
                }
            }
        }
    }
    return singletonObject;
}
```

三级缓存中存放的 `ObjectFactory` 实际上是：

```java
// doCreateBean() 中注册
addSingletonFactory(beanName,
        () -> getEarlyBeanReference(beanName, mbd, bean));

// getEarlyBeanReference：如果有AOP，返回代理；否则返回原始bean
protected Object getEarlyBeanReference(String beanName,
        RootBeanDefinition mbd, Object bean) {
    Object exposedObject = bean;
    for (SmartInstantiationAwareBeanPostProcessor bp :
            getBeanPostProcessorCache().smartInstantiationAware) {
        exposedObject = bp.getEarlyBeanReference(exposedObject, beanName);
        // AbstractAutoProxyCreator 在此创建代理（如果该Bean需要被代理）
    }
    return exposedObject;
}
```

---

## 4.3 循环依赖解决时序图（A 依赖 B，B 依赖 A）

```
getBean("A")
    │
    ├── getSingleton("A") → 一二三级缓存均无
    ├── 标记 A 正在创建
    ├── createBeanInstance → 实例化 A（原始对象 a）
    ├── ★ addSingletonFactory("A", () -> getEarlyBeanReference(a))
    │     三级缓存：{"A": ObjectFactory<a>}
    │
    └── populateBean("A") → 注入 B → getBean("B")
                │
                ├── getSingleton("B") → 缓存无
                ├── 标记 B 正在创建
                ├── createBeanInstance → 实例化 B（原始对象 b）
                ├── addSingletonFactory("B", () -> getEarlyBeanReference(b))
                │
                └── populateBean("B") → 注入 A → getBean("A")
                            │
                            └── getSingleton("A", true)
                                    ├── 一级缓存：无
                                    ├── A 正在创建中 ✓
                                    ├── 二级缓存：无
                                    └── ★ 三级缓存命中！
                                          调用 ObjectFactory.getObject()
                                          → getEarlyBeanReference(a)
                                          → 返回 a（或a的代理）
                                          晋升到二级缓存
                                          返回给 B 的注入点 ✓

    B 注入完毕 → initializeBean("B") → addSingleton("B") 进入一级缓存
    A 注入完毕 → initializeBean("A") → addSingleton("A") 进入一级缓存
```

---

## 4.4 为什么需要三级缓存而不是二级？

**核心原因**：三级缓存中的 `ObjectFactory` 允许在循环依赖发生时**延迟决定是否创建 AOP 代理**，保证注入到其他 Bean 中的引用与容器最终存储的对象一致。

```
二级缓存方案（有问题）：
  早期引用 = 原始 a  →  B.fieldA = 原始a  →  但容器最终是代理A  → 不一致！

三级缓存方案（正确）：
  ObjectFactory.getObject() 调用 getEarlyBeanReference()
  → AbstractAutoProxyCreator.getEarlyBeanReference() → 返回代理A
  → 晋升到二级缓存（代理A）
  → B.fieldA = 代理A  →  容器最终也是代理A  → 一致！ ✓
```

---

## 4.5 构造注入为何无法解决循环依赖？

属性注入：先实例化（构造完成），再注入依赖，三级缓存在实例化后立即曝光：

```
实例化 A（无参构造）→ 曝光 A 到三级缓存 → 填充 A 的属性（注入 B）→ ...
```

构造注入：实例化 A 本身就需要 B 实例，A 尚未实例化完，不在任何缓存中：

```
实例化 A（需要 B）→ 去创建 B → 实例化 B（需要 A）→ 去创建 A
→ A 不在缓存中 → BeanCurrentlyInCreationException ✗
```

---

## 4.6 @Lazy 解决构造注入循环依赖原理

```java
@Component
public class A {
    private final B b;
    public A(@Lazy B b) {  // @Lazy 告诉Spring注入一个代理
        this.b = b;
    }
}
```

`@Lazy` 让 Spring 注入一个 CGLIB 代理对象（而不是真实的 B）。这个代理在第一次被调用方法时才真正去容器中查找 B。由于注入的只是一个轻量级代理（不需要 B 的实例），循环依赖被打破。

```
实例化 A（注入 B 的 CGLIB 代理）→ A 完成实例化
→ 之后 B 也正常创建完毕
→ 调用 A 中的 b.someMethod() 时，代理懒加载真实 B ✓
```

---

## 4.7 Spring 6.x 对循环依赖的变更

Spring 6.0 起，默认**禁用循环依赖**（`allowCircularReferences = false`）。  
若需要保持原行为，需显式开启：

```java
// Spring Boot 3.x 中
spring.main.allow-circular-references=true

// 编程方式
SpringApplication app = new SpringApplication(MainApp.class);
app.setAllowCircularReferences(true);
app.run(args);
```


---

# Part 5: AOP 源码解析 {#part-5}

## 5.1 AOP 核心概念体系

```
AOP 核心组件关系图

  Pointcut（切点）：定义"在哪里"切
  ┌─────────────────────────────────┐
  │  ClassFilter                    │  匹配类
  │  MethodMatcher                  │  匹配方法
  │  ├── StaticMethodMatcher        │
  │  └── DynamicMethodMatcher       │
  └─────────────────────────────────┘

  Advice（通知）：定义"做什么"
  ┌─────────────────────────────────┐
  │  BeforeAdvice                   │  前置
  │  AfterAdvice                    │  后置
  │    ├── AfterReturningAdvice      │  返回后
  │    └── ThrowsAdvice             │  异常后
  │  MethodInterceptor              │  环绕（最底层接口）
  └─────────────────────────────────┘

  Advisor（切面 = Pointcut + Advice）
  ┌─────────────────────────────────┐
  │  PointcutAdvisor                │  带切点的切面
  │    ├── DefaultPointcutAdvisor   │
  │    └── AspectJExpressionAdvisor ★ │  @Aspect注解对应
  │  IntroductionAdvisor            │  引介切面
  └─────────────────────────────────┘
```

---

## 5.2 @AspectJ 注解对应的 Advice 实现

| 注解 | Advice 实现类 |
|------|--------------| 
| `@Before` | `AspectJMethodBeforeAdvice` → 包装为 `MethodBeforeAdviceInterceptor` |
| `@After` | `AspectJAfterAdvice`（实现 `MethodInterceptor`） |
| `@AfterReturning` | `AspectJAfterReturningAdvice` → 包装为 `AfterReturningAdviceInterceptor` |
| `@AfterThrowing` | `AspectJAfterThrowingAdvice`（实现 `MethodInterceptor`） |
| `@Around` | `AspectJAroundAdvice`（实现 `MethodInterceptor`） |

---

## 5.3 AbstractAutoProxyCreator —— 代理创建入口

```java
public abstract class AbstractAutoProxyCreator
        extends ProxyProcessorSupport
        implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {

    // ★ 在 Bean 初始化完成后调用
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean != null) {
            Object cacheKey = getCacheKey(bean.getClass(), beanName);
            if (this.earlyProxyReferences.remove(cacheKey) != bean) {
                return wrapIfNecessary(bean, beanName, cacheKey);
            }
        }
        return bean;
    }

    // ★ 循环依赖场景下提前创建代理（来自三级缓存）
    @Override
    public Object getEarlyBeanReference(Object bean, String beanName) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        this.earlyProxyReferences.put(cacheKey, bean);
        return wrapIfNecessary(bean, beanName, cacheKey);
    }

    protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
        if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
            return bean;
        }

        // ★ 获取适用于该Bean的所有 Advisor
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(
                bean.getClass(), beanName, null);

        if (specificInterceptors != DO_NOT_PROXY) {
            this.advisedBeans.put(cacheKey, Boolean.TRUE);
            // ★ 创建代理
            Object proxy = createProxy(bean.getClass(), beanName,
                    specificInterceptors, new SingletonTargetSource(bean));
            return proxy;
        }

        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }
}
```

### 代理类型选择

```java
// DefaultAopProxyFactory.java
@Override
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isProxyTargetClass()       // proxyTargetClass=true
            || hasNoUserSuppliedProxyInterfaces(config)) { // 没有实现接口
        Class<?> targetClass = config.getTargetClass();
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config); // ★ CGLIB
    } else {
        return new JdkDynamicAopProxy(config); // ★ JDK动态代理
    }
}
```

---

## 5.4 JdkDynamicAopProxy.invoke() —— 调用链执行

```java
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

    // 获取该方法的拦截链
    List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
            method, targetClass);

    if (chain.isEmpty()) {
        // 无拦截器，直接反射调用
        return AopUtils.invokeJoinpointUsingReflection(target, method, args);
    } else {
        // ★ 创建方法调用对象，执行责任链
        MethodInvocation invocation = new ReflectiveMethodInvocation(
                proxy, target, method, args, targetClass, chain);
        return invocation.proceed();
    }
}
```

---

## 5.5 ReflectiveMethodInvocation.proceed() —— 责任链执行

```java
@Override
public Object proceed() throws Throwable {
    // 所有拦截器都已执行完毕，执行目标方法
    if (this.currentInterceptorIndex ==
            this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint(); // 反射调用真实方法
    }

    // 取下一个拦截器
    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher dm) {
        if (dm.methodMatcher().matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor().invoke(this);
        } else {
            return proceed(); // 不匹配，跳过
        }
    } else {
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

### AOP 拦截器责任链执行顺序图

```
代理方法调用
    │
    ▼
ExposeInvocationInterceptor.invoke(mi)
    │ mi.proceed()
    ▼
AspectJAfterThrowingAdvice.invoke(mi)   // @AfterThrowing（try-catch包裹）
    │ try { mi.proceed() }
    ▼
AfterReturningAdviceInterceptor.invoke(mi)  // @AfterReturning
    │ try { mi.proceed() }
    ▼
AspectJAfterAdvice.invoke(mi)           // @After（finally保证执行）
    │ try { mi.proceed() } finally { 执行@After逻辑 }
    ▼
MethodBeforeAdviceInterceptor.invoke(mi) // @Before
    │ 先执行@Before逻辑
    │ mi.proceed()
    ▼
invokeJoinpoint()  →  目标方法执行
    │
    ▲（返回值沿调用链回传）

同一切面内通知执行顺序（Spring 5.2.7+）：
@Around（前） → @Before → 目标方法 → @Around（后） → @After → @AfterReturning/@AfterThrowing
```

---

## 5.6 @Transactional 事务原理

Spring 事务基于 AOP，通过 `TransactionInterceptor` 实现。

```java
// TransactionAspectSupport.java
protected Object invokeWithinTransaction(Method method, Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {

    TransactionAttributeSource tas = getTransactionAttributeSource();
    final TransactionAttribute txAttr = tas.getTransactionAttribute(method, targetClass);
    final TransactionManager tm = determineTransactionManager(txAttr);
    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);

    TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);

    Object retVal;
    try {
        retVal = invocation.proceedWithInvocation(); // 执行目标方法
    } catch (Throwable ex) {
        // 异常时：回滚或提交（取决于 rollbackFor 配置）
        completeTransactionAfterThrowing(txInfo, ex);
        throw ex;
    } finally {
        cleanupTransactionInfo(txInfo);
    }
    // 正常返回：提交事务
    commitTransactionAfterReturning(txInfo);
    return retVal;
}
```

### 事务传播机制

| 传播行为 | 说明 |
|---------|------|
| `REQUIRED`（默认） | 有事务则加入，没有则新建 |
| `REQUIRES_NEW` | 始终新建事务，挂起当前事务 |
| `SUPPORTS` | 有事务则加入，没有则以非事务执行 |
| `NOT_SUPPORTED` | 以非事务执行，挂起当前事务 |
| `MANDATORY` | 必须在事务中，否则抛异常 |
| `NEVER` | 不能在事务中，否则抛异常 |
| `NESTED` | 在当前事务中嵌套执行（SavePoint） |

### @Transactional 事务失效场景

| 场景 | 原因 |
|------|------|
| 同类内部方法调用 | 绕过代理，直接调用原始对象，拦截器不生效 |
| 方法非 `public` | Spring AOP 只代理 public 方法 |
| 未被 Spring 管理 | new 出来的对象不是代理 |
| 异常被吞掉（catch后不抛） | `completeTransactionAfterThrowing` 得不到异常 |
| 默认只回滚 RuntimeException/Error | 受检异常不触发回滚，需配置 `rollbackFor` |
| 多线程异步 | `TransactionSynchronizationManager`（ThreadLocal）不同线程不共享 |
| 数据库引擎不支持事务 | 如 MyISAM |
| 传播行为设置错误 | 如 `NOT_SUPPORTED` 导致无事务运行 |


---

# Part 6: Spring MVC 源码解析 {#part-6}

## 6.1 DispatcherServlet 处理请求流程

```java
// DispatcherServlet.java
protected void doDispatch(HttpServletRequest request,
        HttpServletResponse response) throws Exception {

    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    ModelAndView mv = null;
    Exception dispatchException = null;

    try {
        // [1] 检查是否为文件上传请求
        processedRequest = checkMultipart(request);

        // [2] ★ 查找 Handler（Controller方法）
        mappedHandler = getHandler(processedRequest);
        if (mappedHandler == null) {
            noHandlerFound(processedRequest, response);
            return;
        }

        // [3] ★ 查找 HandlerAdapter
        HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

        // [4] 执行拦截器 preHandle
        if (!mappedHandler.applyPreHandle(processedRequest, response)) {
            return; // 返回false则终止请求
        }

        // [5] ★★★ 执行 Handler（Controller方法）
        mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

        // [6] 设置默认视图名
        applyDefaultViewName(processedRequest, mv);

        // [7] 执行拦截器 postHandle
        mappedHandler.applyPostHandle(processedRequest, response, mv);
    }
    catch (Exception ex) {
        dispatchException = ex;
    }

    // [8] 视图渲染 / 异常处理 / afterCompletion
    processDispatchResult(processedRequest, response, mappedHandler,
            mv, dispatchException);
}
```

---

## 6.2 HandlerMapping 体系

```
HandlerMapping（接口）
    │
    ├── AbstractHandlerMapping
    │       ├── AbstractUrlHandlerMapping
    │       │       └── SimpleUrlHandlerMapping（XML配置url->controller）
    │       │
    │       └── AbstractHandlerMethodMapping ★
    │               └── RequestMappingHandlerMapping（@RequestMapping解析）
    │                     在 afterPropertiesSet() 中扫描所有 @RequestMapping
    │
    └── BeanNameUrlHandlerMapping（BeanName以"/"开头）
```

`RequestMappingHandlerMapping` 初始化过程：

```java
// AbstractHandlerMethodMapping.java
protected void initHandlerMethods() {
    for (String beanName : getCandidateBeanNames()) {
        processCandidateBean(beanName);
    }
}

protected void processCandidateBean(String beanName) {
    Class<?> beanType = obtainApplicationContext().getType(beanName);
    if (beanType != null && isHandler(beanType)) { // 检查是否有@Controller/@RequestMapping
        detectHandlerMethods(beanName); // 扫描方法上的@RequestMapping，注册路由
    }
}
```

---

## 6.3 HandlerAdapter 体系

```
HandlerAdapter（接口）
    │
    ├── RequestMappingHandlerAdapter ★（处理@RequestMapping方法）
    ├── HttpRequestHandlerAdapter（处理HttpRequestHandler）
    └── SimpleControllerHandlerAdapter（处理Controller接口）
```

`RequestMappingHandlerAdapter.invokeHandlerMethod()` 核心流程：

```java
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) {

    ServletInvocableHandlerMethod invocableMethod =
            createInvocableHandlerMethod(handlerMethod);

    // 设置参数解析器（ArgumentResolver）
    invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
    // 设置返回值处理器（ReturnValueHandler）
    invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);

    ModelAndViewContainer mavContainer = new ModelAndViewContainer();

    // ★ 解析参数 → 调用方法 → 处理返回值
    invocableMethod.invokeAndHandle(webRequest, mavContainer);

    return getModelAndView(mavContainer, modelFactory, webRequest);
}
```

---

## 6.4 参数解析器（HandlerMethodArgumentResolver）

```
参数注解                    解析器
────────────────────────────────────────────────────
@RequestParam           RequestParamMethodArgumentResolver
@PathVariable           PathVariableMethodArgumentResolver
@RequestBody            RequestResponseBodyMethodProcessor ★
@RequestHeader          RequestHeaderMethodArgumentResolver
@CookieValue            ServletCookieValueMethodArgumentResolver
@ModelAttribute         ModelAttributeMethodProcessor
@SessionAttribute       SessionAttributeMethodArgumentResolver
HttpServletRequest      ServletRequestMethodArgumentResolver
HttpSession             ServletRequestMethodArgumentResolver
```

`@RequestBody` 解析核心：

```java
// RequestResponseBodyMethodProcessor.java
@Override
public Object resolveArgument(MethodParameter parameter, ...) throws Exception {
    // ★ 使用 HttpMessageConverter 读取请求体（如 MappingJackson2HttpMessageConverter）
    Object arg = readWithMessageConverters(webRequest, parameter,
            parameter.getNestedGenericParameterType());

    // 数据绑定与校验（@Valid/@Validated）
    if (binderFactory != null) {
        WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
        if (arg != null) {
            validateIfApplicable(binder, parameter);
            if (binder.getBindingResult().hasErrors()) {
                throw new MethodArgumentNotValidException(parameter,
                        binder.getBindingResult());
            }
        }
    }
    return adaptArgumentIfNecessary(arg, parameter);
}
```

---

## 6.5 过滤器与拦截器执行顺序

```
HTTP 请求
    │
    ▼
┌────────────────────────────────────────────────────────────┐
│  Filter 1（doFilter 前置）                    Servlet容器层 │
│    └── Filter 2（doFilter 前置）                           │
│         └── Filter 3（doFilter 前置）                      │
│              └──────────────────────────────────────────── │
│                   DispatcherServlet.service()              │
│              ┌──────────────────────────────────────────── │
│              │  Interceptor 1.preHandle()     Spring 层    │
│              │    └── Interceptor 2.preHandle()            │
│              │         └── Controller 方法执行              │
│              │    ┌─── Interceptor 2.postHandle()          │
│              │  Interceptor 1.postHandle()                 │
│              │  视图渲染（如有）                             │
│              │  Interceptor 2.afterCompletion()            │
│              │  Interceptor 1.afterCompletion()            │
│              └──────────────────────────────────────────── │
│         └── Filter 3（doFilter 后置）                      │
│    └── Filter 2（doFilter 后置）                           │
│  Filter 1（doFilter 后置）                                 │
└────────────────────────────────────────────────────────────┘
    │
    ▼
HTTP 响应

注意：
- Filter 先于 Interceptor 执行
- preHandle 按注册顺序执行（1→2→3）
- postHandle 按注册逆序执行（3→2→1），异常时不执行
- afterCompletion 按注册逆序执行，一定会执行（即使异常）
```

---

## 6.6 HandlerExceptionResolver 异常处理链

```
HandlerExceptionResolver（接口）
    │
    └── HandlerExceptionResolverComposite（组合模式，按顺序尝试）
            ├── ExceptionHandlerExceptionResolver ★（处理@ExceptionHandler）
            ├── ResponseStatusExceptionResolver（处理@ResponseStatus）
            └── DefaultHandlerExceptionResolver（处理Spring MVC标准异常）
```

`@ExceptionHandler` 的处理由 `ExceptionHandlerExceptionResolver` 负责，
原理与 `RequestMappingHandlerAdapter` 类似（参数解析 + 返回值处理）。


---

# Part 7: Spring Boot 自动配置源码解析 {#part-7}

## 7.1 @SpringBootApplication 解析

```java
@SpringBootConfiguration      // 等同于 @Configuration
@EnableAutoConfiguration      // ★ 开启自动配置
@ComponentScan(               // 组件扫描（默认扫描当前类所在包）
    excludeFilters = {
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class)
    }
)
public @interface SpringBootApplication { ... }
```

`@EnableAutoConfiguration` 通过 `@Import` 引入 `AutoConfigurationImportSelector`：

```java
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)  // ★
public @interface EnableAutoConfiguration { ... }
```

---

## 7.2 AutoConfigurationImportSelector 源码解析

```java
public class AutoConfigurationImportSelector
        implements DeferredImportSelector, BeanClassLoaderAware {

    @Override
    public String[] selectImports(AnnotationMetadata annotationMetadata) {
        AutoConfigurationEntry entry = getAutoConfigurationEntry(annotationMetadata);
        return StringUtils.toStringArray(entry.getConfigurations());
    }

    protected AutoConfigurationEntry getAutoConfigurationEntry(
            AnnotationMetadata annotationMetadata) {
        // 获取所有候选自动配置类
        List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);

        // 去重
        configurations = removeDuplicates(configurations);

        // 获取 exclude 排除项
        Set<String> exclusions = getExclusions(annotationMetadata, attributes);
        configurations.removeAll(exclusions);

        // ★ 使用 AutoConfigurationImportFilter 过滤（@ConditionalOnXxx 快速过滤）
        configurations = getConfigurationClassFilter().filter(configurations);

        return new AutoConfigurationEntry(configurations, exclusions);
    }

    protected List<String> getCandidateConfigurations(...) {
        // ★ Spring Boot 2.x：读取 META-INF/spring.factories
        List<String> configurations = new ArrayList<>(
                SpringFactoriesLoader.loadFactoryNames(
                        getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader()));

        // ★ Spring Boot 3.x 还读取：
        // META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
        ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())
                .forEach(configurations::add);

        return configurations;
    }
}
```

---

## 7.3 SpringFactoriesLoader 源码

```java
public final class SpringFactoriesLoader {

    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

    // 缓存：ClassLoader -> (factoryType -> [factoryImpl...])
    static final Map<ClassLoader, Map<String, List<String>>> cache
            = new ConcurrentReferenceHashMap<>();

    private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        Map<String, List<String>> result = cache.get(classLoader);
        if (result != null) return result;

        result = new HashMap<>();
        // ★ 扫描 classpath 下所有 jar 中的 META-INF/spring.factories
        Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
        while (urls.hasMoreElements()) {
            URL url = urls.nextElement();
            Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));
            for (Map.Entry<?, ?> entry : properties.entrySet()) {
                String factoryTypeName = ((String) entry.getKey()).trim();
                String[] impls = StringUtils.commaDelimitedListToStringArray(
                        (String) entry.getValue());
                for (String impl : impls) {
                    result.computeIfAbsent(factoryTypeName, k -> new ArrayList<>())
                            .add(impl.trim());
                }
            }
        }
        cache.put(classLoader, result);
        return result;
    }
}
```

---

## 7.4 @ConditionalOnXxx 条件注解原理

```java
// 所有 @ConditionalOnXxx 都基于 @Conditional 机制
@Conditional(OnClassCondition.class)  // ★ 指定 Condition 实现
public @interface ConditionalOnClass {
    Class<?>[] value() default {};
    String[] name() default {};
}
```

`OnClassCondition` 判断逻辑（检查类是否在 classpath 中存在）：

```java
public class OnClassCondition extends FilteringSpringBootCondition {
    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context,
            AnnotatedTypeMetadata metadata) {
        List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
        if (onClasses != null) {
            List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
            if (!missing.isEmpty()) {
                return ConditionOutcome.noMatch(...);
            }
        }
        return ConditionOutcome.match(matchMessage);
    }
}
```

常用条件注解速查：

| 注解 | 条件 |
|------|------|
| `@ConditionalOnClass` | classpath 中存在指定类 |
| `@ConditionalOnMissingClass` | classpath 中不存在指定类 |
| `@ConditionalOnBean` | 容器中存在指定 Bean |
| `@ConditionalOnMissingBean` | 容器中不存在指定 Bean（★自定义覆盖的关键） |
| `@ConditionalOnProperty` | 配置属性满足条件 |
| `@ConditionalOnResource` | classpath 中存在指定资源文件 |
| `@ConditionalOnWebApplication` | 是 Web 应用 |
| `@ConditionalOnExpression` | SpEL 表达式为 true |
| `@ConditionalOnJava` | JDK 版本满足 |
| `@ConditionalOnSingleCandidate` | 容器中只有一个候选 Bean |

---

## 7.5 SpringApplication.run() 启动流程

```java
public ConfigurableApplicationContext run(String... args) {
    long startTime = System.nanoTime();
    DefaultBootstrapContext bootstrapContext = createBootstrapContext();

    // [1] 获取 SpringApplicationRunListeners（来自spring.factories）
    SpringApplicationRunListeners listeners = getRunListeners(args);
    listeners.starting(bootstrapContext, this.mainApplicationClass);

    try {
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);

        // [2] 准备 Environment（加载配置文件：application.properties/yml）
        ConfigurableEnvironment environment = prepareEnvironment(listeners,
                bootstrapContext, applicationArguments);

        // [3] 打印 Banner
        Banner printedBanner = printBanner(environment);

        // [4] 创建 ApplicationContext（根据类型：Servlet/Reactive/None）
        context = createApplicationContext();

        // [5] 准备 ApplicationContext（加载主类BeanDefinition）
        prepareContext(bootstrapContext, context, environment, listeners,
                applicationArguments, printedBanner);

        // [6] ★★★ 刷新容器（调用 refresh()，包含内嵌Tomcat创建）
        refreshContext(context);

        // [7] 刷新后处理
        afterRefresh(context, applicationArguments);

        // [8] 通知监听器：started
        listeners.started(context, Duration.ofNanos(System.nanoTime() - startTime));

        // [9] 执行 CommandLineRunner / ApplicationRunner
        callRunners(context, applicationArguments);
    }
    catch (Throwable ex) {
        handleRunFailure(context, ex, listeners);
        throw new IllegalStateException(ex);
    }

    // [10] 通知监听器：ready
    listeners.ready(context, Duration.ofNanos(System.nanoTime() - startTime));

    return context;
}
```

### 启动流程时序图

```
main()
  │
  └── SpringApplication.run()
        │
        ├── [1] 触发 ApplicationStartingEvent
        ├── [2] prepareEnvironment()
        │     ├── 创建 StandardServletEnvironment
        │     ├── 配置属性源（命令行参数、系统属性、环境变量）
        │     ├── 触发 ApplicationEnvironmentPreparedEvent
        │     └── ConfigFileApplicationListener 加载 application.yml
        │
        ├── [3] 打印 Banner
        ├── [4] createApplicationContext()
        │     └── AnnotationConfigServletWebServerApplicationContext
        │
        ├── [5] prepareContext()
        │     ├── 执行 ApplicationContextInitializer
        │     ├── 触发 ApplicationContextInitializedEvent
        │     ├── 注册主类 BeanDefinition
        │     └── 触发 ApplicationPreparedEvent
        │
        ├── [6] refreshContext() → AbstractApplicationContext.refresh()
        │     └── onRefresh() ★
        │           └── createWebServer()
        │                 └── 创建内嵌 Tomcat/Jetty/Undertow，启动监听
        │
        ├── [8] 触发 ApplicationStartedEvent
        ├── [9] 执行 CommandLineRunner / ApplicationRunner（按@Order排序）
        └── [10] 触发 ApplicationReadyEvent
```


---

# Part 8: Spring 事件机制源码 {#part-8}

## 8.1 事件体系

```
EventObject（Java标准）
    └── ApplicationEvent（Spring基类）
            ├── ContextRefreshedEvent      容器refresh完成
            ├── ContextStartedEvent        容器start()
            ├── ContextStoppedEvent        容器stop()
            ├── ContextClosedEvent         容器close()
            ├── ApplicationStartingEvent   Boot启动中
            ├── ApplicationReadyEvent      Boot就绪
            └── 自定义事件（继承ApplicationEvent）

ApplicationListener<E extends ApplicationEvent>
    └── SmartApplicationListener（支持事件类型/源类型过滤）

ApplicationEventMulticaster
    └── AbstractApplicationEventMulticaster
            └── SimpleApplicationEventMulticaster ★（默认同步实现）
```

---

## 8.2 事件发布流程

```java
// AbstractApplicationContext.java
protected void publishEvent(Object event, ResolvableType typeHint) {
    ApplicationEvent applicationEvent;
    if (event instanceof ApplicationEvent ae) {
        applicationEvent = ae;
    } else {
        applicationEvent = new PayloadApplicationEvent<>(this, event, typeHint);
    }

    // ★ 通过广播器发布
    if (this.applicationEventMulticaster != null) {
        this.applicationEventMulticaster.multicastEvent(applicationEvent, typeHint);
    }

    // 同时发布到父容器（如果有）
    if (this.parent != null) {
        this.parent.publishEvent(event);
    }
}
```

---

## 8.3 SimpleApplicationEventMulticaster 源码

```java
@Override
public void multicastEvent(ApplicationEvent event, ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType
            : ResolvableType.forInstance(event));

    Executor executor = getTaskExecutor();

    // ★ 找到所有匹配的监听器并调用
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event)); // 异步
        } else {
            invokeListener(listener, event); // 同步（默认）
        }
    }
}

private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
    listener.onApplicationEvent(event); // ★ 回调监听器
}
```

---

## 8.4 @EventListener 注解原理

`@EventListener` 方法由 `EventListenerMethodProcessor`（`SmartInitializingSingleton`）处理：

```java
// EventListenerMethodProcessor.java
@Override
public void afterSingletonsInstantiated() {
    // 在所有单例Bean初始化完毕后执行
    for (String beanName : beanNames) {
        processBean(beanName, targetType);
    }
}

private void processBean(String beanName, Class<?> targetType) {
    // 扫描类中所有 @EventListener 标注的方法
    Map<Method, EventListener> annotatedMethods = MethodIntrospector.selectMethods(
            targetType,
            method -> AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));

    for (Map.Entry<Method, EventListener> entry : annotatedMethods.entrySet()) {
        Method method = entry.getKey();
        EventListenerFactory factory = selectEventListenerFactory(method);
        // ★ 使用 ApplicationListenerMethodAdapter 包装成 ApplicationListener
        ApplicationListener<?> applicationListener =
                factory.createApplicationListener(beanName, targetType, method);
        context.addApplicationListener(applicationListener);
    }
}
```

---

## 8.5 异步事件与事务绑定事件

### 异步事件

```java
// 配置异步广播器
@Bean
public ApplicationEventMulticaster applicationEventMulticaster() {
    SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
    multicaster.setTaskExecutor(Executors.newCachedThreadPool());
    return multicaster;
}

// 或者在监听器上使用 @Async（需要 @EnableAsync）
@EventListener
@Async
public void handleEvent(MyEvent event) {
    // 异步执行
}
```

### 事务绑定事件（@TransactionalEventListener）

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleOrderCreated(OrderCreatedEvent event) {
    // 只在事务提交后执行
    // phase 可选：BEFORE_COMMIT / AFTER_COMMIT / AFTER_ROLLBACK / AFTER_COMPLETION
}
```

原理：`TransactionalApplicationListenerMethodAdapter` 实现了 `TransactionSynchronization`，在事务提交/回滚的回调中才触发事件处理。如果当前无事务，默认不执行（可通过 `fallbackExecution = true` 改变此行为）。


---

# Part 9: 常用扩展点汇总 {#part-9}

## 9.1 所有扩展点执行顺序总览

```
Spring 容器启动扩展点执行顺序
════════════════════════════════════════════════════════════

【BeanDefinition 阶段】
  BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry()
    → 优先级：PriorityOrdered → Ordered → 无排序
    → 典型：ConfigurationClassPostProcessor（处理@Configuration）
    → 可以：新增/修改 BeanDefinition

  BeanFactoryPostProcessor.postProcessBeanFactory()
    → 优先级同上
    → 典型：PropertySourcesPlaceholderConfigurer（处理${}）
    → 可以：修改 BeanDefinition 属性

【Bean 实例化阶段（每个Bean都经历）】

  InstantiationAwareBPP.postProcessBeforeInstantiation()
    → 若返回非null，跳过后续实例化，直接执行 postProcessAfterInitialization()

  ★ Bean 实例化（构造函数/工厂方法）

  MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition()
    → 收集注解元数据（@Autowired字段/方法）

  ★ 三级缓存曝光（单例循环依赖支持）

  InstantiationAwareBPP.postProcessAfterInstantiation()
    → 返回false可跳过属性填充

  InstantiationAwareBPP.postProcessProperties()
    → AutowiredAnnotationBPP → @Autowired/@Value/@Inject 注入
    → CommonAnnotationBPP → @Resource 注入

  ★ applyPropertyValues()（XML <property> 配置）

【Bean 初始化阶段】

  BeanNameAware / BeanClassLoaderAware / BeanFactoryAware 回调

  BeanPostProcessor.postProcessBeforeInitialization()
    → ApplicationContextAwareProcessor（注入各种ApplicationContext级Aware）
    → CommonAnnotationBPP → @PostConstruct

  InitializingBean.afterPropertiesSet()
  init-method / @Bean(initMethod)

  BeanPostProcessor.postProcessAfterInitialization()
    → AbstractAutoProxyCreator ★ → 创建AOP代理

【所有单例初始化完毕】
  SmartInitializingSingleton.afterSingletonsInstantiated()
    → EventListenerMethodProcessor → 处理@EventListener

【容器就绪】
  ApplicationListener<ContextRefreshedEvent>
  CommandLineRunner.run() / ApplicationRunner.run()（Boot）

【容器关闭阶段】
  DestructionAwareBPP.postProcessBeforeDestruction()
    → CommonAnnotationBPP → @PreDestroy
  DisposableBean.destroy()
  destroy-method / @Bean(destroyMethod)
  ApplicationListener<ContextClosedEvent>
════════════════════════════════════════════════════════════
```

---

## 9.2 扩展点详细说明

### BeanFactoryPostProcessor 实战

```java
@Component
public class MyBFPP implements BeanFactoryPostProcessor {
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) {
        BeanDefinition bd = bf.getBeanDefinition("myBean");
        bd.getPropertyValues().add("name", "modified");
        // 注意：不要在此处调用 bf.getBean()（会导致过早实例化）
    }
}
```

### BeanDefinitionRegistryPostProcessor 实战

```java
@Component
public class MyBDRPP implements BeanDefinitionRegistryPostProcessor {
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        GenericBeanDefinition bd = new GenericBeanDefinition();
        bd.setBeanClass(MyDynamicBean.class);
        registry.registerBeanDefinition("dynamicBean", bd);
    }
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory bf) { }
}
```

### BeanPostProcessor 实战

```java
@Component
public class MyBeanPostProcessor implements BeanPostProcessor {

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof MyInterface) {
            System.out.println("Before init: " + beanName);
        }
        return bean; // 必须返回bean（或替换品）
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof MyInterface) {
            // 可以用JDK动态代理包装
            return Proxy.newProxyInstance(
                bean.getClass().getClassLoader(),
                bean.getClass().getInterfaces(),
                (proxy, method, args) -> {
                    System.out.println("Before method: " + method.getName());
                    return method.invoke(bean, args);
                }
            );
        }
        return bean;
    }
}
```

### ImportBeanDefinitionRegistrar（@Import 扩展）

```java
public class MyRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
        // 读取注解属性，动态注册BeanDefinition
        AnnotationAttributes attrs = AnnotationAttributes.fromMap(
            importingClassMetadata.getAnnotationAttributes(EnableMyFeature.class.getName()));
        String basePackage = attrs.getString("basePackage");
        new ClassPathBeanDefinitionScanner(registry).scan(basePackage);
    }
}
// MyBatis 的 @MapperScan 就是此原理
```

### Aware 接口体系

| Aware 接口 | 注入内容 | 注入时机 |
|-----------|---------|---------|
| `BeanNameAware` | Bean 的名称 | `invokeAwareMethods()` |
| `BeanClassLoaderAware` | ClassLoader | `invokeAwareMethods()` |
| `BeanFactoryAware` | BeanFactory | `invokeAwareMethods()` |
| `EnvironmentAware` | Environment | `ApplicationContextAwareProcessor` |
| `EmbeddedValueResolverAware` | StringValueResolver | `ApplicationContextAwareProcessor` |
| `ResourceLoaderAware` | ResourceLoader | `ApplicationContextAwareProcessor` |
| `ApplicationEventPublisherAware` | ApplicationEventPublisher | `ApplicationContextAwareProcessor` |
| `ApplicationContextAware` | ApplicationContext | `ApplicationContextAwareProcessor` |

### FactoryBean 扩展点

```java
@Component
public class MyFactoryBean implements FactoryBean<MyService> {

    @Override
    public MyService getObject() throws Exception {
        return new MyServiceImpl(); // 自定义实例化逻辑
    }

    @Override
    public Class<?> getObjectType() { return MyService.class; }

    @Override
    public boolean isSingleton() { return true; }
}

// 使用：
// getBean("myFactoryBean")   → 获取 MyService 实例
// getBean("&myFactoryBean")  → 获取 MyFactoryBean 本身
```

### ApplicationContextInitializer（Boot 扩展）

```java
// 在 ApplicationContext refresh() 之前调用
public class MyInitializer
        implements ApplicationContextInitializer<ConfigurableApplicationContext> {

    @Override
    public void initialize(ConfigurableApplicationContext context) {
        context.getEnvironment().getPropertySources()
               .addFirst(new MapPropertySource("custom",
                       Collections.singletonMap("key", "value")));
    }
}

// 注册（spring.factories）：
// org.springframework.context.ApplicationContextInitializer=com.example.MyInitializer
```


---

# Part 10: 高频面试题 FAQ（源码级） {#part-10}

---

### Q1. Spring Bean 的生命周期是什么？

**答**：完整生命周期（参见 Part 3 流程图）：

1. **实例化**：`createBeanInstance()` 通过构造函数/工厂方法创建原始对象
2. **属性填充**：`populateBean()` 注入依赖（`@Autowired`、`@Resource`、XML property）
3. **初始化前**：`BPP.postProcessBeforeInitialization()` → `@PostConstruct`
4. **初始化**：`InitializingBean.afterPropertiesSet()` → `init-method`
5. **初始化后**：`BPP.postProcessAfterInitialization()` → AOP 代理创建
6. **就绪**：Bean 可被使用
7. **销毁前**：`@PreDestroy`
8. **销毁**：`DisposableBean.destroy()` → `destroy-method`

---

### Q2. BeanFactory 与 ApplicationContext 的区别？

| 维度 | BeanFactory | ApplicationContext |
|------|------------|-------------------|
| 实例化时机 | 懒加载（getBean时） | 提前实例化所有非懒加载单例 |
| 国际化 | 不支持 | 支持 MessageSource |
| 事件 | 不支持 | 支持 ApplicationEventPublisher |
| AOP | 需要手动集成 | 自动集成（BeanPostProcessor） |
| 环境 | 不支持 | 支持 Environment/Profile |
| 资源加载 | 基础 | ResourcePatternResolver（classpath*:） |
| 适用场景 | 资源受限环境 | 生产应用 |

---

### Q3. Spring 如何解决循环依赖？

**答**：通过三级缓存（参见 Part 4）：

- **一级缓存**（`singletonObjects`）：完整 Bean
- **二级缓存**（`earlySingletonObjects`）：早期引用（已从三级缓存晋升）
- **三级缓存**（`singletonFactories`）：`ObjectFactory`，可按需创建代理

**只能解决**：单例 Bean 的属性注入/setter注入循环依赖  
**不能解决**：构造注入循环依赖、原型 Bean 循环依赖  
**为何需要三级**：三级缓存的 `ObjectFactory` 允许提前创建 AOP 代理，保证注入引用与最终对象一致。

---

### Q4. @Autowired 与 @Resource 的区别？

| 维度 | `@Autowired` | `@Resource` |
|------|-------------|------------|
| 来源 | Spring | JSR-250（Jakarta） |
| 处理器 | `AutowiredAnnotationBPP` | `CommonAnnotationBPP` |
| 默认注入方式 | **按类型**（byType） | **按名称**（byName） |
| 多个候选时 | 按 `@Qualifier`/`@Primary`/字段名决策 | 按名称决策 |
| 可用位置 | 字段、setter、构造函数、配置方法 | 字段、setter |

---

### Q5. Spring AOP 的实现原理？JDK 动态代理与 CGLIB 的区别？

**答**：Spring AOP 基于代理模式，`AbstractAutoProxyCreator`（BPP）在 Bean 初始化完成后创建代理。

| 维度 | JDK 动态代理 | CGLIB |
|------|------------|-------|
| 要求 | 目标类必须实现接口 | 无要求（不能是 final 类/方法） |
| 原理 | `java.lang.reflect.Proxy` + `InvocationHandler` | 字节码增强，生成子类 |
| Spring 选择 | 有接口且未设置proxyTargetClass=true | 无接口，或proxyTargetClass=true |

调用时通过 `ReflectiveMethodInvocation.proceed()` 以递归方式执行**拦截器责任链**。

---

### Q6. @Transactional 失效场景有哪些？

1. **同类内部调用**：`this.methodB()` 绕过代理 → 解决：注入自身或用 `AopContext.currentProxy()`
2. **方法非 public**：Spring AOP 只代理 public 方法
3. **异常被吞**：catch 后不重新抛出
4. **受检异常**：默认只回滚 RuntimeException，需配置 `@Transactional(rollbackFor = Exception.class)`
5. **多线程**：`TransactionSynchronizationManager` 使用 ThreadLocal，跨线程失效
6. **数据库引擎不支持事务**：如 MyISAM

---

### Q7. Spring 事务的传播机制是什么？

| 传播行为 | 说明 |
|---------|------|
| `REQUIRED`（默认） | 加入现有事务或新建 |
| `REQUIRES_NEW` | 挂起现有事务，新建独立事务 |
| `NESTED` | 嵌套事务（Savepoint），内部回滚不影响外部 |
| `SUPPORTS` | 有事务则加入，无事务则非事务执行 |
| `NOT_SUPPORTED` | 以非事务执行，挂起当前事务 |
| `MANDATORY` | 必须在事务中，否则抛异常 |
| `NEVER` | 不能在事务中，否则抛异常 |

---

### Q8. Spring Boot 自动配置原理？

1. `@SpringBootApplication` 包含 `@EnableAutoConfiguration`
2. `@Import(AutoConfigurationImportSelector.class)` 引入选择器
3. `selectImports()` 读取 `META-INF/spring.factories`（Boot 2.x）或 `AutoConfiguration.imports`（Boot 3.x）
4. 通过 `@ConditionalOnXxx` 过滤，只有满足条件的配置类才生效
5. 用户可通过 `@ConditionalOnMissingBean` 覆盖默认自动配置

---

### Q9. refresh() 方法中各步骤的作用？

| 步骤 | 方法 | 作用 |
|------|------|------|
| 1 | `prepareRefresh()` | 记录启动时间，验证必要属性 |
| 2 | `obtainFreshBeanFactory()` | 获取/刷新 BeanFactory，加载 BeanDefinition |
| 3 | `prepareBeanFactory()` | 配置标准特性（ClassLoader、Aware处理器等） |
| 4 | `postProcessBeanFactory()` | 子类扩展（Web 环境添加 request/session scope） |
| 5 | `invokeBeanFactoryPostProcessors()` | ★ ConfigurationClassPostProcessor 扫描注解 |
| 6 | `registerBeanPostProcessors()` | 注册所有 BPP |
| 7 | `initMessageSource()` | 初始化国际化 |
| 8 | `initApplicationEventMulticaster()` | 初始化事件广播器 |
| 9 | `onRefresh()` | 子类扩展（Boot 创建内嵌服务器） |
| 10 | `registerListeners()` | 注册事件监听器 |
| 11 | `finishBeanFactoryInitialization()` | ★★★ 实例化所有非懒加载单例 |
| 12 | `finishRefresh()` | 发布 `ContextRefreshedEvent` |

---

### Q10. @Configuration 与 @Component 的区别？

**答**：

- **`@Configuration`（Full 模式）**：类被 CGLIB 增强，`@Bean` 方法之间的调用会走 Spring 容器，保证单例语义
- **`@Component`（Lite 模式）**：类不被增强，`@Bean` 方法之间直接调用，每次调用都产生新实例

```java
@Configuration
public class AppConfig {
    @Bean
    public A a() { return new A(b()); } // b() 走Spring容器，返回单例B
    @Bean
    public B b() { return new B(); }
}

@Component
public class AppConfig2 {
    @Bean
    public A a() { return new A(b()); } // b() 直接调用，产生新的B实例！
    @Bean
    public B b() { return new B(); }
}
```

---

### Q11. Spring 中的 Scope 有哪些？

| Scope | 说明 | 场景 |
|-------|------|------|
| `singleton`（默认） | 容器中只有一个实例 | 无状态服务 |
| `prototype` | 每次 `getBean()` 都创建新实例 | 有状态对象 |
| `request` | 每个 HTTP 请求一个实例 | 请求级数据 |
| `session` | 每个 HTTP Session 一个实例 | 会话级数据 |
| `application` | 每个 ServletContext 一个实例 | 应用级共享 |
| `websocket` | 每个 WebSocket 会话一个实例 | WebSocket |

**注意**：单例 Bean 注入 prototype Bean 时，每次都会得到同一个 prototype 实例（只注入一次）。
解决方案：`@Lookup` 方法注入，或每次通过 `ApplicationContext.getBean()` 获取。

---

### Q12. Spring 中如何实现动态代理的？

```
代理创建链路：

@EnableAspectJAutoProxy
    └── @Import(AspectJAutoProxyRegistrar)
            └── 注册 AnnotationAwareAspectJAutoProxyCreator（BPP）
                    └── 继承 AbstractAutoProxyCreator
                            └── postProcessAfterInitialization() 调用 wrapIfNecessary()
                                    ├── 查找匹配的 Advisor
                                    └── ProxyFactory.getProxy()
                                            └── DefaultAopProxyFactory.createAopProxy()
                                                    ├── JdkDynamicAopProxy（有接口）
                                                    └── ObjenesisCglibAopProxy（无接口）
```

---

### Q13. Spring 事件监听有哪几种方式？

```java
// 方式1：实现 ApplicationListener 接口
@Component
public class MyListener implements ApplicationListener<MyEvent> {
    @Override
    public void onApplicationEvent(MyEvent event) { ... }
}

// 方式2：@EventListener 注解（推荐）
@Component
public class MyListener {
    @EventListener
    public void handleEvent(MyEvent event) { ... }

    // 条件过滤
    @EventListener(condition = "#event.success")
    public void handleSuccess(MyEvent event) { ... }

    // 返回值自动发布为新事件
    @EventListener
    public AnotherEvent handleAndPublish(MyEvent event) {
        return new AnotherEvent(this);
    }
}

// 方式3：@TransactionalEventListener（事务绑定）
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleAfterCommit(MyEvent event) { ... }
```

---

### Q14. @Value 注入原理是什么？

`@Value` 由 `AutowiredAnnotationBeanPostProcessor` 处理（与 `@Autowired` 同一处理器）。

解析流程：
1. `AutowiredAnnotationBPP` 在 `postProcessProperties()` 中发现 `@Value` 注解
2. 调用 `beanFactory.resolveEmbeddedValue(value)` → `StringValueResolver`
3. `PropertySourcesPlaceholderConfigurer` 解析 `${}` 占位符
4. `StandardBeanExpressionResolver` 解析 `#{}` SpEL 表达式
5. `TypeConverter` 将字符串转换为目标类型

---

### Q15. Spring 中如何获取 ApplicationContext？

```java
// 方式1：实现 ApplicationContextAware（最常用工具类写法）
@Component
public class SpringContextHolder implements ApplicationContextAware {
    private static ApplicationContext context;
    @Override
    public void setApplicationContext(ApplicationContext ctx) {
        SpringContextHolder.context = ctx;
    }
    public static <T> T getBean(Class<T> clazz) {
        return context.getBean(clazz);
    }
}

// 方式2：直接 @Autowired 注入
@Component
public class MyService {
    @Autowired
    private ApplicationContext applicationContext;
}
```

---

### Q16. Spring 中 BeanDefinition 的加载流程是什么？

```
AnnotationConfigApplicationContext(AppConfig.class)
    │
    └── register(AppConfig.class)
          └── AnnotatedBeanDefinitionReader.register()
                └── doRegisterBean()
                      └── AnnotatedGenericBeanDefinition 注册到 BeanDefinitionRegistry
    │
    └── refresh()
          └── invokeBeanFactoryPostProcessors()
                └── ConfigurationClassPostProcessor.postProcessBeanDefinitionRegistry()
                      └── ConfigurationClassParser.parse(AppConfig)
                            ├── @ComponentScan → ClassPathBeanDefinitionScanner.scan()
                            │     └── 扫描包，对每个候选类生成 ScannedGenericBeanDefinition
                            ├── @Import → 处理 ImportSelector / ImportBDRegistrar
                            └── @Bean 方法 → ConfigurationClassBeanDefinitionReader
                                  └── 注册 ConfigurationClassBeanDefinition
```

---

### Q17. Spring 如何处理 @Async 异步方法？

`@Async` 由 `AsyncAnnotationBeanPostProcessor`（或 `AsyncAnnotationAdvisor`）处理：

1. `@EnableAsync` 导入 `AsyncConfigurationSelector`
2. 注册 `AsyncAnnotationBeanPostProcessor`（或通过 AOP advisor）
3. 在 `postProcessAfterInitialization()` 中为有 `@Async` 方法的 Bean 创建代理
4. 代理拦截器 `AnnotationAsyncExecutionInterceptor` 在方法调用时：
   - 获取 `TaskExecutor`（`@Async("executorName")` 或默认）
   - 将方法调用提交到线程池异步执行
   - 返回 `Future`/`CompletableFuture`（如果方法返回值为此类型）

**注意**：`@Async` 与 `@Transactional` 一样，同类内部调用无效（绕过代理）。

---

### Q18. FactoryBean 与 BeanFactory 的区别？

| 维度 | BeanFactory | FactoryBean |
|------|------------|-------------|
| 性质 | Spring IoC 容器的根接口 | 一个特殊的 Bean（生产其他Bean的工厂） |
| 作用 | 管理和获取 Bean | 通过 `getObject()` 产生 Bean 实例 |
| 使用场景 | 框架核心 | 需要复杂初始化逻辑的 Bean（如 MyBatis Mapper） |
| 获取 | 直接是容器 | `getBean("name")` 返回产物，`getBean("&name")` 返回 FactoryBean 本身 |

---

### Q19. Spring Boot 如何实现零 XML 配置？

核心在于注解驱动：
1. `@SpringBootApplication` 触发组件扫描，自动发现 `@Component` / `@Service` 等
2. `@EnableAutoConfiguration` 通过 SPI 机制自动引入第三方配置
3. `application.yml/properties` 替代 XML 属性配置
4. `@Bean` 方法替代 XML 中的 `<bean>` 标签
5. `@ImportResource` 在需要时仍可引入 XML（兼容旧系统）

---

### Q20. Spring 中如何自定义 Starter？

```
1. 创建 autoconfigure 模块（含自动配置类）
   ├── XxxAutoConfiguration.java（@Configuration + @ConditionalOnXxx）
   ├── XxxProperties.java（@ConfigurationProperties(prefix="xxx")）
   └── META-INF/spring.factories（或 AutoConfiguration.imports）
         └── EnableAutoConfiguration=com.example.XxxAutoConfiguration

2. 创建 starter 模块（只含 pom 依赖，引入 autoconfigure 模块）
   └── pom.xml 声明对 autoconfigure 模块的依赖

3. 用户只需引入 starter 的 Maven 依赖
   └── Spring Boot 自动发现并应用 XxxAutoConfiguration

命名约定：
  官方 starter:      spring-boot-starter-xxx
  第三方 starter:    xxx-spring-boot-starter
```

---

### Q21. 说说你对 Spring 设计模式的理解？

Spring 中使用的主要设计模式：

| 设计模式 | 体现 |
|---------|------|
| 工厂模式 | `BeanFactory`、`FactoryBean` |
| 单例模式 | Spring Bean 默认单例，`DefaultSingletonBeanRegistry` |
| 代理模式 | AOP（JDK动态代理/CGLIB） |
| 模板方法模式 | `AbstractApplicationContext.refresh()`、`JdbcTemplate` |
| 观察者模式 | `ApplicationEvent`/`ApplicationListener` 事件机制 |
| 装饰器模式 | `BeanWrapper`、`HttpServletRequestWrapper` |
| 责任链模式 | `ReflectiveMethodInvocation.proceed()`、Filter/Interceptor链 |
| 策略模式 | `InstantiationStrategy`、`ResourceLoader` |
| 适配器模式 | `HandlerAdapter`（适配不同类型Handler） |
| 组合模式 | `CompositePropertySource`、`HandlerExceptionResolverComposite` |

---

## 后记

本文档覆盖了 Spring Framework 5.x/6.x 的核心源码，包括：

- **IoC 容器**：`refresh()` 12步、`BeanFactory`体系、`BeanDefinition`注册与合并
- **Bean 生命周期**：从 `doCreateBean()` 到销毁的完整链路
- **循环依赖**：三级缓存的设计哲学与实现细节
- **AOP**：代理创建、拦截器责任链、事务实现
- **Spring MVC**：`DispatcherServlet` 请求处理链路
- **Spring Boot**：自动配置 SPI 机制、启动流程
- **事件机制**：`SimpleApplicationEventMulticaster`、`@EventListener`
- **扩展点**：完整执行顺序与最佳实践

建议结合 Spring 源码（GitHub: spring-projects/spring-framework）配合本文档阅读，效果最佳。


---

# 附录 A: Spring 核心源码类索引 {#appendix-a}

## A.1 IoC 容器核心类

| 类名 | 包路径 | 核心职责 |
|------|--------|---------|
| `DefaultListableBeanFactory` | `o.s.beans.factory.support` | BeanFactory 核心实现，BeanDefinition 注册表 |
| `AbstractAutowireCapableBeanFactory` | `o.s.beans.factory.support` | Bean 实例化、属性填充、初始化 |
| `AbstractApplicationContext` | `o.s.context.support` | `refresh()` 模板方法实现 |
| `AnnotationConfigApplicationContext` | `o.s.context.annotation` | 注解驱动容器入口 |
| `ConfigurationClassPostProcessor` | `o.s.context.annotation` | 处理 @Configuration/@ComponentScan/@Import 等 |
| `ConfigurationClassParser` | `o.s.context.annotation` | 递归解析配置类 |
| `ClassPathBeanDefinitionScanner` | `o.s.context.annotation` | classpath 组件扫描 |
| `DefaultSingletonBeanRegistry` | `o.s.beans.factory.support` | 三级缓存、单例注册表 |
| `PostProcessorRegistrationDelegate` | `o.s.context.support` | BFPP/BPP 排序执行逻辑 |

## A.2 AOP 核心类

| 类名 | 包路径 | 核心职责 |
|------|--------|---------|
| `AbstractAutoProxyCreator` | `o.s.aop.framework.autoproxy` | 代理创建 BPP 基类 |
| `AnnotationAwareAspectJAutoProxyCreator` | `o.s.aop.aspectj.annotation` | 处理 @AspectJ 注解的代理创建器 |
| `AspectJExpressionPointcut` | `o.s.aop.aspectj` | AspectJ 切点表达式匹配 |
| `ReflectiveAspectJAdvisorFactory` | `o.s.aop.aspectj.annotation` | 将 @Aspect 类解析为 Advisor |
| `DefaultAopProxyFactory` | `o.s.aop.framework` | 选择 JDK/CGLIB 代理 |
| `JdkDynamicAopProxy` | `o.s.aop.framework` | JDK 动态代理实现 |
| `ObjenesisCglibAopProxy` | `o.s.aop.framework` | CGLIB 代理实现 |
| `ReflectiveMethodInvocation` | `o.s.aop.framework` | 拦截器责任链执行 |
| `TransactionInterceptor` | `o.s.transaction.interceptor` | 事务拦截器 |
| `DataSourceTransactionManager` | `o.s.jdbc.datasource` | JDBC 事务管理器 |

## A.3 Spring MVC 核心类

| 类名 | 包路径 | 核心职责 |
|------|--------|---------|
| `DispatcherServlet` | `o.s.web.servlet` | 前端控制器，请求分发 |
| `RequestMappingHandlerMapping` | `o.s.web.servlet.mvc.method.annotation` | @RequestMapping 路由注册与查找 |
| `RequestMappingHandlerAdapter` | `o.s.web.servlet.mvc.method.annotation` | 执行 Controller 方法 |
| `RequestResponseBodyMethodProcessor` | `o.s.web.servlet.mvc.method.annotation` | @RequestBody/@ResponseBody 处理 |
| `ExceptionHandlerExceptionResolver` | `o.s.web.servlet.mvc.method.annotation` | @ExceptionHandler 异常处理 |
| `InvocableHandlerMethod` | `o.s.web.method.support` | 可调用的 Handler 方法包装 |

## A.4 Spring Boot 核心类

| 类名 | 包路径 | 核心职责 |
|------|--------|---------|
| `SpringApplication` | `o.s.boot` | Boot 启动入口 |
| `AutoConfigurationImportSelector` | `o.s.boot.autoconfigure` | 自动配置选择器 |
| `SpringFactoriesLoader` | `o.s.core.io.support` | SPI 加载机制 |
| `OnClassCondition` | `o.s.boot.autoconfigure.condition` | @ConditionalOnClass 条件判断 |
| `OnBeanCondition` | `o.s.boot.autoconfigure.condition` | @ConditionalOnBean 条件判断 |
| `ConfigurationPropertiesBindingPostProcessor` | `o.s.boot.context.properties` | @ConfigurationProperties 绑定 |
| `ServletWebServerApplicationContext` | `o.s.boot.web.servlet.context` | 嵌入式 Web 容器管理 |
| `TomcatServletWebServerFactory` | `o.s.boot.web.embedded.tomcat` | 内嵌 Tomcat 工厂 |

---

# 附录 B: 常见问题排查指南 {#appendix-b}

## B.1 Bean 创建失败排查树

```
BeanCreationException
    │
    ├── NoSuchBeanDefinitionException
    │     原因：找不到指定 Bean
    │     排查：检查 @ComponentScan 范围、@Component 注解是否存在
    │
    ├── NoUniqueBeanDefinitionException
    │     原因：按类型找到多个候选 Bean
    │     排查：添加 @Qualifier 或 @Primary
    │
    ├── BeanCurrentlyInCreationException
    │     原因：构造注入循环依赖
    │     排查：改为属性注入，或在参数上添加 @Lazy
    │
    ├── UnsatisfiedDependencyException
    │     原因：依赖无法注入（通常包装了其他异常）
    │     排查：查看 caused by 链，找根因
    │
    └── BeanDefinitionStoreException
          原因：BeanDefinition 加载失败（XML解析错误等）
          排查：检查配置文件语法
```

## B.2 事务不生效排查 CheckList

```
□ 1. 是否添加了 @EnableTransactionManagement（或引入了 spring-tx 自动配置）？
□ 2. 方法是否为 public？
□ 3. 是否在同类内部调用（绕过代理）？
□ 4. 是否抛出了异常（异常未被 catch 吞掉）？
□ 5. 异常类型是否匹配 rollbackFor 配置？
□ 6. DataSource 是否配置了 PlatformTransactionManager？
□ 7. 数据库引擎是否支持事务（InnoDB vs MyISAM）？
□ 8. 是否在新线程中执行（ThreadLocal 不传播）？
□ 9. 传播行为是否正确（NOT_SUPPORTED 会挂起事务）？
□ 10. Bean 是否被 Spring 管理（不是 new 出来的）？
```

## B.3 AOP 不生效排查 CheckList

```
□ 1. 是否添加了 @EnableAspectJAutoProxy？
□ 2. @Aspect 类是否被 @Component 标注（是否被 Spring 管理）？
□ 3. 切点表达式是否正确？
□ 4. 目标方法是否为 public？
□ 5. 是否同类内部调用？
□ 6. 目标类或方法是否为 final（CGLIB 无法代理）？
□ 7. proxyTargetClass 设置是否正确？
```

---

# 附录 C: Spring 版本演进重点变化 {#appendix-c}

## C.1 Spring 5.x 重要特性

| 版本 | 关键特性 |
|------|---------|
| 5.0 | WebFlux 响应式框架；Kotlin 支持；JDK 8+ 基线 |
| 5.1 | Bean 定义覆盖控制；Context 索引（spring-context-indexer） |
| 5.2 | `@Configuration(proxyBeanMethods=false)` lite 模式；@EventListener 顺序支持 |
| 5.3 | AOT 基础设施；对 Spring Boot 2.4+ 的全面支持 |

## C.2 Spring 6.x 重要变化

| 版本 | 关键变化 |
|------|---------|
| 6.0 | JDK 17 基线；`jakarta.*` 替代 `javax.*`；默认禁用循环依赖；GraalVM 原生镜像支持 |
| 6.1 | 虚拟线程（Project Loom）支持；`RestClient`（替代 RestTemplate）；`@ConditionalOnThreading` |

## C.3 Spring Boot 3.x 迁移注意点

```
javax.* → jakarta.*
  javax.servlet.* → jakarta.servlet.*
  javax.persistence.* → jakarta.persistence.*
  javax.validation.* → jakarta.validation.*

spring.factories → AutoConfiguration.imports
  旧：META-INF/spring.factories
      org.springframework.boot.autoconfigure.EnableAutoConfiguration=XxxAutoConfiguration
  新：META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
      com.example.XxxAutoConfiguration

其他变化：
  循环依赖默认禁用，需显式开启 spring.main.allow-circular-references=true
  Actuator 默认暴露端点变化（health 默认暴露，其他默认关闭）
```

---

# 附录 D: 调试 Spring 的实用技巧 {#appendix-d}

## D.1 打开 Spring 调试日志

```yaml
# application.yml
logging:
  level:
    org.springframework: DEBUG          # 全部 Spring 日志
    org.springframework.beans: DEBUG    # Bean 创建过程
    org.springframework.context: DEBUG  # 容器启动过程
    org.springframework.aop: DEBUG      # AOP 代理创建
    org.springframework.tx: DEBUG       # 事务日志
    org.springframework.web: DEBUG      # MVC 请求处理
```

## D.2 查看自动配置报告（Spring Boot）

```bash
# 方式1：启动时添加 --debug 参数
java -jar app.jar --debug

# 方式2：Actuator 端点
GET /actuator/conditions

# 输出示例：
# Positive matches（已生效的自动配置）：
#   DataSourceAutoConfiguration matched:
#     - @ConditionalOnClass found required classes ...
# Negative matches（未生效的自动配置）：
#   MongoAutoConfiguration:
#     Did not match: @ConditionalOnClass did not find required class 'com.mongodb.MongoClient'
```

## D.3 查看 Bean 定义信息

```java
@Component
public class BeanLister implements ApplicationListener<ContextRefreshedEvent> {
    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        ConfigurableListableBeanFactory bf =
            (ConfigurableListableBeanFactory)
                event.getApplicationContext().getAutowireCapableBeanFactory();

        String[] names = bf.getBeanDefinitionNames();
        System.out.println("Total beans: " + names.length);
        for (String name : names) {
            BeanDefinition bd = bf.getBeanDefinition(name);
            System.out.println("  Bean: " + name
                + " | Scope: " + bd.getScope()
                + " | Class: " + bd.getBeanClassName());
        }
    }
}
```

## D.4 验证 AOP 代理是否生效

```java
@Autowired
private MyService myService;

// 检查是否是代理
boolean isProxy = AopUtils.isAopProxy(myService);        // true/false
boolean isCglib = AopUtils.isCglibProxy(myService);      // 是否CGLIB代理
boolean isJdk   = AopUtils.isJdkDynamicProxy(myService); // 是否JDK代理

// 获取代理的目标对象
Object target = ((Advised) myService).getTargetSource().getTarget();
System.out.println("Target class: " + target.getClass().getName());
```

## D.5 手动触发 BeanDefinition 注册

```java
ConfigurableApplicationContext ctx = SpringApplication.run(App.class);
ConfigurableListableBeanFactory beanFactory = ctx.getBeanFactory();
BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;

// 注册新的 BeanDefinition
GenericBeanDefinition bd = new GenericBeanDefinition();
bd.setBeanClass(MyDynamicService.class);
bd.setScope(BeanDefinition.SCOPE_SINGLETON);
registry.registerBeanDefinition("myDynamicService", bd);

// 手动实例化
beanFactory.getBean("myDynamicService");
```

---

# 附录 E: 高频面试题补充（Q22-Q30） {#appendix-e}

### Q22. @Bean 与 @Component 的区别？

| 维度 | `@Bean` | `@Component` |
|------|---------|-------------|
| 使用位置 | 方法上（在 @Configuration 类中） | 类上 |
| 控制粒度 | 高（可精确控制实例化逻辑） | 低（Spring 自动实例化） |
| 适用场景 | 第三方库的 Bean（无法修改源码） | 自己编写的类 |
| 命名 | 默认方法名，可通过 `name` 属性自定义 | 默认类名首字母小写 |

---

### Q23. @Primary 和 @Qualifier 的区别？

- **`@Primary`**：标记在 Bean 定义处，表示多个同类型候选 Bean 时优先选此。全局生效。
- **`@Qualifier`**：标记在注入点，指定要注入的 Bean 名称。精确控制。

```java
@Bean
@Primary
public DataSource primaryDataSource() { ... }

@Bean
public DataSource secondaryDataSource() { ... }

@Autowired  // 自动注入 primaryDataSource
private DataSource dataSource;

@Autowired
@Qualifier("secondaryDataSource")  // 显式指定次数据源
private DataSource secondary;
```

---

### Q24. @PostConstruct 与 InitializingBean 哪个先执行？

**答**：`@PostConstruct` 先执行，`InitializingBean.afterPropertiesSet()` 后执行。

原因：
- `@PostConstruct` 由 `CommonAnnotationBPP.postProcessBeforeInitialization()` 处理（第2步）
- `InitializingBean.afterPropertiesSet()` 在第3步 `invokeInitMethods()` 中调用

完整顺序：`@PostConstruct` → `afterPropertiesSet()` → `init-method`

---

### Q25. Spring Boot 配置文件加载优先级？

```
优先级从高到低：
1. 命令行参数（--server.port=8081）
2. JNDI 属性（java:comp/env）
3. Java 系统属性（System.getProperties()）
4. 操作系统环境变量
5. application-{profile}.yml（外部，jar同级config目录）
6. application.yml（外部，jar同级config目录）
7. application-{profile}.yml（classpath:/config/）
8. application.yml（classpath:/config/）
9. application-{profile}.yml（classpath根）
10. application.yml（classpath根）

高优先级属性会覆盖低优先级，而非替换整个文件。
```

---

### Q26. Spring 中如何实现方法级别的缓存？

```java
@EnableCaching  // 导入 CachingConfigurationSelector，注册 CacheInterceptor（AOP）
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id",
               condition = "#id > 0", unless = "#result == null")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElse(null);
    }

    @CacheEvict(value = "products", key = "#product.id")
    public void updateProduct(Product product) {
        productRepository.save(product);
    }

    @CachePut(value = "products", key = "#result.id")
    public Product createProduct(Product product) {
        return productRepository.save(product);
    }
}
```

`@Cacheable` 执行逻辑（`CacheInterceptor` 实现）：
1. 生成缓存 key（默认参数，可用 SpEL 自定义）
2. 查询 `CacheManager` 是否命中
3. 命中则返回缓存值；未命中则执行方法并将结果存入缓存

---

### Q27. Prototype Bean 注入到 Singleton Bean 的问题与解决方案？

当 Singleton 注入 Prototype 时，只注入一次，导致每次使用同一个 Prototype 实例：

```java
@Component
public class SingletonBean {

    // 问题：每次调用 prototypeBean 都是同一个实例
    @Autowired
    private PrototypeBean prototypeBean;

    // 解决方案1：@Lookup 方法注入（CGLIB重写此方法，每次从容器获取）
    @Lookup
    public PrototypeBean getPrototype() { return null; }

    // 解决方案2：ObjectProvider（懒加载 + 按需获取）
    @Autowired
    private ObjectProvider<PrototypeBean> provider;

    public void doWork() {
        PrototypeBean bean = provider.getObject(); // 每次获取新实例
        bean.doSomething();
    }

    // 解决方案3：实现 ApplicationContextAware，手动 getBean()
}
```

---

### Q28. Spring 中如何优雅关闭容器？

```java
// 方式1：注册 JVM shutdown hook（Spring Boot 默认已注册）
context.registerShutdownHook();

// 关闭流程：
// ContextClosedEvent 发布
//   → SmartLifecycle.stop()（按 phase 逆序）
//   → DisposableBean.destroy() + @PreDestroy + destroy-method
//   → BeanFactory 销毁

// 方式2：Spring Boot Actuator 端点（需开启）
// POST /actuator/shutdown

// 方式3：容器化环境发送 SIGTERM（推荐）
// Spring Boot 内置优雅关闭支持：
// server.shutdown=graceful
// spring.lifecycle.timeout-per-shutdown-phase=30s
```

---

### Q29. 什么是 Spring 中的 BeanWrapper？

`BeanWrapper` 是 Spring 对 Bean 实例的包装，提供：
- 属性读写（支持嵌套属性 `address.city`、数组/列表 `items[0]`）
- 类型转换（通过 `ConversionService` 或 `PropertyEditor`）
- 属性描述符获取

```java
BeanWrapper wrapper = new BeanWrapperImpl(new User());
wrapper.setPropertyValue("name", "Alice");
wrapper.setPropertyValue("address.city", "Beijing"); // 嵌套属性
String name = (String) wrapper.getPropertyValue("name");
```

Spring 在 `applyPropertyValues()` 阶段使用 `BeanWrapper` 将 XML/注解中的属性值应用到 Bean 实例。

---

### Q30. 如何自定义 Spring Boot Starter？

```
步骤：

1. 创建 xxx-spring-boot-autoconfigure 模块
   ├── XxxAutoConfiguration.java
   │     @Configuration
   │     @ConditionalOnClass(XxxClient.class)  // classpath存在核心类才生效
   │     @EnableConfigurationProperties(XxxProperties.class)
   │     public class XxxAutoConfiguration {
   │         @Bean
   │         @ConditionalOnMissingBean  // 用户未定义时才创建默认Bean
   │         public XxxClient xxxClient(XxxProperties props) {
   │             return new XxxClient(props.getUrl());
   │         }
   │     }
   │
   ├── XxxProperties.java
   │     @ConfigurationProperties(prefix = "xxx")
   │     public class XxxProperties {
   │         private String url;
   │         // getter/setter
   │     }
   │
   └── META-INF/spring/
         org.springframework.boot.autoconfigure.AutoConfiguration.imports
           (内容：com.example.XxxAutoConfiguration)

2. 创建 xxx-spring-boot-starter 模块（仅 pom）
   └── pom.xml 依赖 autoconfigure 模块 + 核心依赖

命名约定：
  官方：spring-boot-starter-xxx
  第三方：xxx-spring-boot-starter（推荐）
```

---

# 附录 F: Spring 关键源码文件路径速查 {#appendix-f}

```
spring-context 模块（org.springframework:spring-context）
  src/main/java/org/springframework/context/
    support/AbstractApplicationContext.java     ← refresh() 12步
    support/PostProcessorRegistrationDelegate.java  ← BFPP/BPP 执行
    annotation/ConfigurationClassPostProcessor.java ← @Configuration处理
    annotation/ConfigurationClassParser.java    ← 配置类递归解析
    event/SimpleApplicationEventMulticaster.java ← 事件广播
    annotation/AnnotationConfigApplicationContext.java ← 注解容器入口

spring-beans 模块（org.springframework:spring-beans）
  src/main/java/org/springframework/beans/factory/
    support/DefaultListableBeanFactory.java     ← BeanFactory核心
    support/AbstractAutowireCapableBeanFactory.java ← doCreateBean
    support/DefaultSingletonBeanRegistry.java   ← 三级缓存
    annotation/AutowiredAnnotationBeanPostProcessor.java ← @Autowired处理

spring-aop 模块（org.springframework:spring-aop）
  src/main/java/org/springframework/aop/
    framework/autoproxy/AbstractAutoProxyCreator.java  ← 代理创建BPP
    framework/JdkDynamicAopProxy.java                  ← JDK代理invoke
    framework/ReflectiveMethodInvocation.java           ← 责任链proceed
    framework/DefaultAopProxyFactory.java               ← 代理类型选择

spring-tx 模块（org.springframework:spring-tx）
  src/main/java/org/springframework/transaction/
    interceptor/TransactionInterceptor.java        ← 事务拦截器
    interceptor/TransactionAspectSupport.java      ← 事务核心逻辑
    annotation/EnableTransactionManagement.java    ← 事务开关

spring-webmvc 模块（org.springframework:spring-webmvc）
  src/main/java/org/springframework/web/servlet/
    DispatcherServlet.java                         ← 前端控制器
    mvc/method/annotation/RequestMappingHandlerMapping.java
    mvc/method/annotation/RequestMappingHandlerAdapter.java

spring-boot-autoconfigure 模块
  src/main/java/org/springframework/boot/autoconfigure/
    AutoConfigurationImportSelector.java           ← 自动配置选择器
    condition/OnClassCondition.java                ← @ConditionalOnClass
    condition/OnBeanCondition.java                 ← @ConditionalOnBean
```

---

## 总结

本文档系统梳理了 Spring Framework 从底层 IoC 到上层 MVC、Boot 的完整技术栈，
涵盖源码级分析、流程图、面试题与实战技巧，适合：

- **面试备战**：所有高频问题均有源码级答案
- **日常开发**：扩展点汇总可指导自定义组件开发
- **问题排查**：附录提供了系统性的排查 CheckList
- **深入学习**：关键源码片段配合 GitHub 源码食用效果最佳

---

*Spring Framework 是站在巨人肩膀上的框架，深入理解其源码是每位 Java 开发者成长的必经之路。*
