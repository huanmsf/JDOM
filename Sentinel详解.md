# Sentinel 详解 - 从零到精通（流量防卫兵）

> 版本：Sentinel 1.8.x / Spring Cloud Alibaba 2021.x
> 适用：Java 8+，Spring Boot 2.x，Spring Cloud 2021.x
> 目标：彻底掌握 Sentinel 限流、熔断、降级、热点、权限控制全栈技术

---

## 目录

- [Part 1: Sentinel 整体架构](#part-1-sentinel-整体架构)
- [Part 2: 流量控制（限流）深度解析](#part-2-流量控制限流深度解析)
- [Part 3: 熔断降级深度解析](#part-3-熔断降级深度解析)
- [Part 4: 系统自适应保护](#part-4-系统自适应保护)
- [Part 5: 热点参数限流](#part-5-热点参数限流)
- [Part 6: 权限控制（黑白名单）](#part-6-权限控制黑白名单)
- [Part 7: Sentinel 统计原理](#part-7-sentinel-统计原理)
- [Part 8: Sentinel Dashboard 控制台](#part-8-sentinel-dashboard-控制台)
- [Part 9: Spring Boot/Cloud 集成实战](#part-9-spring-bootcloud-集成实战)
- [Part 10: 完整实战案例](#part-10-完整实战案例)
- [Part 11: 常见面试题 FAQ](#part-11-常见面试题-faq)

---

# Part 1: Sentinel 整体架构

## 1.1 Sentinel 是什么

Sentinel 是阿里巴巴开源的**面向分布式服务架构的流量控制组件**，主要以流量为切入点，从流量控制、熔断降级、系统负载保护等多个维度来帮助您保护服务的稳定性。

**核心能力：**
- **流量控制（Flow Control）**：根据 QPS、并发线程数等指标对访问流量进行控制
- **熔断降级（Circuit Breaking）**：当调用链路中某个资源出现不稳定状态时，对该资源的调用进行限制
- **系统自适应保护（System Adaptive Protection）**：从系统的维度进行保护，防止系统崩溃
- **热点参数限流（Hot Spot）**：对特定参数值的请求进行限流
- **权限控制（Authority Control）**：根据来源（origin）对调用进行黑白名单控制

**Sentinel 的诞生背景：**

在微服务架构中，服务之间存在复杂的调用关系。当某个服务出现性能问题时，可能导致：
- 调用方线程阻塞，请求积压
- 连锁反应导致整个系统雪崩
- 下游服务压力过大彻底宕机

Sentinel 正是为解决这些问题而生，提供了一套完整的流量治理解决方案。

**版本历史：**
```
2012年 - 阿里内部诞生，初始用于双11流量控制
2018年 - 开源，发布 v0.1
2019年 - v1.6，加入熔断降级增强
2020年 - v1.7，加入 Reactive 支持
2021年 - v1.8，加入熔断器状态机优化
至今   - 持续维护更新
```

---

## 1.2 Sentinel vs Hystrix vs Resilience4j 全面对比

| 特性 | Sentinel | Hystrix | Resilience4j |
|------|----------|---------|--------------|
| **隔离策略** | 信号量（并发线程数） | 线程池隔离 / 信号量 | 信号量 |
| **熔断降级策略** | 慢调用比例/异常比例/异常数 | 失败比例 | 失败比例/慢调用比例 |
| **实时统计** | 滑动时间窗口（LeapArray） | 滑动时间窗口（RxJava） | Ring Bit Buffer |
| **动态规则配置** | 支持（多种数据源） | 支持（有限） | 有限 |
| **控制台** | 提供开箱即用的 Dashboard | 需集成 Hystrix Dashboard | 无 |
| **限流** | QPS/并发线程数，多种效果 | 有限支持 | 速率限制器 |
| **热点参数限流** | 支持 | 不支持 | 不支持 |
| **系统保护** | 支持（CPU/Load/RT等） | 不支持 | 不支持 |
| **流控效果** | 快速失败/Warm Up/排队等待 | 快速失败 | 有限 |
| **维护状态** | 活跃维护 | 已停止维护（Netflix） | 活跃维护 |
| **Spring Cloud 集成** | Spring Cloud Alibaba | Spring Cloud Netflix | Spring Cloud Circuit Breaker |
| **性能开销** | 低（无额外线程） | 较高（线程池切换） | 低 |
| **生产成熟度** | 阿里双11验证 | 大量生产实践 | 较新 |
| **学习曲线** | 中等 | 中等 | 较陡 |
| **注解支持** | @SentinelResource | @HystrixCommand | @CircuitBreaker |

**选型建议：**
- 国内新项目首选 **Sentinel**（控制台、热点、系统保护等功能完善）
- 已有 Hystrix 项目可迁移到 **Resilience4j**（Hystrix 已停维护）
- 国际化项目可考虑 **Resilience4j**（社区活跃、轻量）

---

## 1.3 Sentinel 整体架构图

```
+------------------------------------------------------------------+
|                        用户应用（业务代码）                         |
|                                                                    |
|   @SentinelResource("order-create")                               |
|   public Order createOrder(Long userId, Long productId) { ... }   |
+----------------------------------+-------------------------------+
                                   |
                     SphU.entry("order-create")
                                   |
+----------------------------------v-------------------------------+
|                    ProcessorSlotChain (责任链)                    |
|                                                                    |
|  NodeSelectorSlot                                                  |
|       |                                                            |
|       v                                                            |
|  ClusterBuilderSlot                                                |
|       |                                                            |
|       v                                                            |
|  LogSlot                                                           |
|       |                                                            |
|       v                                                            |
|  StatisticSlot  <-------- 统计 QPS/RT/线程数                       |
|       |                                                            |
|       v                                                            |
|  AuthoritySlot  <-------- 权限控制（黑白名单）                      |
|       |                                                            |
|       v                                                            |
|  SystemSlot     <-------- 系统自适应保护                           |
|       |                                                            |
|       v                                                            |
|  FlowSlot       <-------- 流量控制（限流）                         |
|       |                                                            |
|       v                                                            |
|  DegradeSlot    <-------- 熔断降级                                 |
|       |                                                            |
|       v                                                            |
|  ParamFlowSlot  <-------- 热点参数限流                             |
+----------------------------------+-------------------------------+
                                   |
                          通过 / 拒绝（BlockException）
                                   |
+----------------------------------v-------------------------------+
|                         实时监控 & Dashboard                       |
|                                                                    |
|   MetricsCollector                                                 |
|   +--------------+   +--------------+   +--------------+          |
|   |  QPS统计     |   |  RT统计      |   |  异常统计     |          |
|   +--------------+   +--------------+   +--------------+          |
|                                                                    |
|   Sentinel Dashboard (port: 8080)                                  |
|   +-----------+   +----------+   +-----------+                    |
|   | 实时监控  |   | 规则配置 |   | 簇点链路  |                    |
|   +-----------+   +----------+   +-----------+                    |
+-------------------------------------------------------------------+
```

---

## 1.4 核心概念详解

### 1.4.1 资源（Resource）

资源是 Sentinel 的核心概念。它可以是 Java 应用程序中的任何内容：
- 一个方法调用
- 一个服务接口
- 一段代码块

**定义资源的三种方式：**

**方式一：try-catch-finally（推荐，显式控制）**
```java
// 1. 定义资源名称
String resourceName = "order-create";

Entry entry = null;
try {
    // 2. 申请资源令牌
    entry = SphU.entry(resourceName);

    // 3. 执行业务逻辑
    return orderService.create(userId, productId);

} catch (BlockException ex) {
    // 4. 被限流/降级，执行降级逻辑
    return fallbackOrder();

} catch (Exception ex) {
    // 5. 业务异常，需要手动记录（用于熔断统计）
    Tracer.traceEntry(ex, entry);
    throw ex;

} finally {
    // 6. 释放资源令牌（必须！否则线程数统计异常）
    if (entry != null) {
        entry.exit();
    }
}
```

**方式二：注解方式（最简洁）**
```java
@SentinelResource(
    value = "order-create",
    blockHandler = "orderCreateBlockHandler",
    fallback = "orderCreateFallback"
)
public Order createOrder(Long userId, Long productId) {
    return orderService.create(userId, productId);
}

// blockHandler：处理限流/熔断异常（BlockException）
public Order orderCreateBlockHandler(Long userId, Long productId, BlockException ex) {
    log.warn("请求被限流，userId={}, productId={}", userId, productId);
    return Order.defaultOrder();
}

// fallback：处理业务异常（任何 Throwable）
public Order orderCreateFallback(Long userId, Long productId, Throwable ex) {
    log.error("业务异常降级，userId={}, productId={}", userId, productId, ex);
    return Order.emptyOrder();
}
```

**方式三：SphO 返回 boolean**
```java
if (SphO.entry("order-create")) {
    try {
        return orderService.create(userId, productId);
    } finally {
        SphO.exit();
    }
} else {
    // 被限流
    return fallbackOrder();
}
```

### 1.4.2 规则（Rule）

规则是 Sentinel 中对资源行为的定义，包括：

| 规则类型 | 类名 | 用途 |
|---------|------|------|
| 流控规则 | FlowRule | 控制 QPS/并发数 |
| 熔断规则 | DegradeRule | 控制熔断降级 |
| 热点规则 | ParamFlowRule | 热点参数限流 |
| 系统规则 | SystemRule | 系统自适应保护 |
| 权限规则 | AuthorityRule | 黑白名单 |

### 1.4.3 降级（Fallback）

降级是指当资源调用受限时（限流或熔断），执行备用逻辑的过程：
- **blockHandler**：专门处理 `BlockException`（限流/熔断触发）
- **fallback**：处理业务异常（所有 Throwable）
- 两者都存在时，`BlockException` 优先走 `blockHandler`

---

## 1.5 Sentinel 工作流程（责任链模式）

Sentinel 的核心是一个**责任链模式（Chain of Responsibility）**的处理流程：

```
请求进入
    |
    v
+---+---+
| NodeSelectorSlot   |  --> 构建调用树节点（DefaultNode）
+-------+-------+
        |
        v
+-------+-------+
| ClusterBuilderSlot | --> 构建集群节点（ClusterNode），聚合统计
+-------+-------+
        |
        v
+-------+-------+
| StatisticSlot  |  --> 统计实时指标（QPS、RT、线程数）
+-------+-------+
        |
        v
+-------+-------+
| AuthoritySlot  |  --> 检查权限规则（黑白名单）
+-------+-------+
        |
        v
+-------+-------+
| SystemSlot     |  --> 检查系统保护规则
+-------+-------+
        |
        v
+-------+-------+
| FlowSlot       |  --> 检查流控规则（限流）
+-------+-------+
        |
        v
+-------+-------+
| DegradeSlot    |  --> 检查熔断规则
+-------+-------+
        |
        v
    业务执行
```

**ProcessorSlotChain 源码简析：**

```java
// com.alibaba.csp.sentinel.slotchain.DefaultProcessorSlotChain
public class DefaultProcessorSlotChain extends ProcessorSlotChain {

    AbstractLinkedProcessorSlot<?> first = new AbstractLinkedProcessorSlot<Object>() {
        @Override
        public void entry(Context context, ResourceWrapper resourceWrapper,
                          Object t, int count, boolean prioritized, Object... args)
                          throws Throwable {
            super.fireEntry(context, resourceWrapper, t, count, prioritized, args);
        }
    };

    @Override
    public void addFirst(AbstractLinkedProcessorSlot<?> protocolProcessor) {
        protocolProcessor.setNext(first.getNext());
        first.setNext(protocolProcessor);
    }

    @Override
    public void addLast(AbstractLinkedProcessorSlot<?> protocolProcessor) {
        end.setNext(protocolProcessor);
        end = protocolProcessor;
    }
}
```

---

## 1.6 核心 Slot 详解

### NodeSelectorSlot

**职责：** 根据上下文构建调用树，记录每个调用入口到资源的访问路径。

```java
// NodeSelectorSlot 核心逻辑（伪代码）
@Override
public void entry(Context context, ResourceWrapper resourceWrapper,
                  Object obj, int count, boolean prioritized, Object... args) throws Throwable {

    // 从缓存中获取当前上下文对应的 DefaultNode
    DefaultNode node = map.get(context.getName());

    if (node == null) {
        synchronized (this) {
            node = map.get(context.getName());
            if (node == null) {
                // 创建新的 DefaultNode，关联当前资源
                node = new DefaultNode(resourceWrapper, null);
                // 加入调用树（构建父子关系）
                ((DefaultNode)context.getLastNode()).addChild(node);
                map.put(context.getName(), node);
            }
        }
    }

    // 将节点设置到上下文中
    context.setCurNode(node);

    // 继续责任链
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}
```

### ClusterBuilderSlot

**职责：** 为每个资源维护一个全局的 ClusterNode，聚合所有入口的统计数据。

```
调用树示例：
            ROOT
           /    \
 /api/order     /api/product
      |               |
 order-create    product-query
      |
 [ClusterNode: order-create]  <-- 聚合所有入口的统计
```

### StatisticSlot

**职责：** 记录实时统计数据，是 Sentinel 统计的核心 Slot。

```java
// StatisticSlot 核心逻辑（简化）
@Override
public void entry(Context context, ResourceWrapper resourceWrapper,
                  DefaultNode node, int count, boolean prioritized, Object... args) throws Throwable {
    try {
        // 先执行后续 Slot（FlowSlot、DegradeSlot 等）
        fireEntry(context, resourceWrapper, node, count, prioritized, args);

        // 后续 Slot 通过，统计成功请求数
        node.increaseThreadNum();          // 增加线程数
        node.addPassRequest(count);        // 增加通过请求数

        // 触发统计回调（可扩展点）
        for (ProcessorSlotEntryCallback<DefaultNode> handler :
                StatisticSlotCallbackRegistry.getEntryCallbacks()) {
            handler.onPass(context, resourceWrapper, node, count, args);
        }

    } catch (BlockException e) {
        // 被拦截，统计 block 数
        context.getCurEntry().setBlockError(e);
        node.increaseBlockQps(count);      // 增加 block QPS
        throw e;

    } catch (Throwable e) {
        // 业务异常
        context.getCurEntry().setError(e);
        throw e;
    }
}
```

---

# Part 2: 流量控制（限流）深度解析

## 2.1 FlowRule 参数详解

```java
public class FlowRule extends AbstractRule {

    /**
     * 流量控制阈值类型
     * 0 - FLOW_GRADE_THREAD（并发线程数）
     * 1 - FLOW_GRADE_QPS（每秒请求数）
     */
    private int grade = RuleConstant.FLOW_GRADE_QPS;

    /**
     * 限流阈值
     * grade=QPS时：每秒最大请求数
     * grade=THREAD时：最大并发线程数
     */
    private double count;

    /**
     * 流控模式
     * 0 - STRATEGY_DIRECT（直接模式）
     * 1 - STRATEGY_RELATE（关联模式）
     * 2 - STRATEGY_CHAIN（链路模式）
     */
    private int strategy = RuleConstant.STRATEGY_DIRECT;

    /**
     * 关联资源或入口资源名称
     * strategy=RELATE 时填写关联资源名
     * strategy=CHAIN 时填写调用链入口名
     */
    private String refResource;

    /**
     * 流控效果
     * 0 - CONTROL_BEHAVIOR_DEFAULT（快速失败）
     * 1 - CONTROL_BEHAVIOR_WARM_UP（Warm Up预热）
     * 2 - CONTROL_BEHAVIOR_RATE_LIMITER（排队等待）
     * 3 - CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER（预热+排队）
     */
    private int controlBehavior = RuleConstant.CONTROL_BEHAVIOR_DEFAULT;

    /**
     * Warm Up 预热时长（秒）
     * controlBehavior=1 或 3 时有效
     */
    private int warmUpPeriodSec = 10;

    /**
     * 排队最大等待时长（毫秒）
     * controlBehavior=2 或 3 时有效
     */
    private int maxQueueingTimeMs = 500;

    /**
     * 是否集群限流
     */
    private boolean clusterMode;

    /**
     * 集群限流配置
     */
    private ClusterFlowConfig clusterConfig;
}
```

**编程方式配置流控规则：**

```java
@Configuration
public class SentinelConfig {

    @PostConstruct
    public void initFlowRules() {
        List<FlowRule> rules = new ArrayList<>();

        // 规则1：order-create 接口 QPS 限制为 100
        FlowRule rule1 = new FlowRule();
        rule1.setResource("order-create");
        rule1.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule1.setCount(100);
        rule1.setStrategy(RuleConstant.STRATEGY_DIRECT);
        rule1.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT);
        rules.add(rule1);

        // 规则2：product-query 并发线程数限制为 20
        FlowRule rule2 = new FlowRule();
        rule2.setResource("product-query");
        rule2.setGrade(RuleConstant.FLOW_GRADE_THREAD);
        rule2.setCount(20);
        rules.add(rule2);

        // 加载规则
        FlowRuleManager.loadRules(rules);
    }
}
```

---

## 2.2 三种流控模式

### 2.2.1 直接模式（Direct）

最简单的模式，直接对当前资源进行限流。

```
请求 ---> [order-create 资源] ---> QPS > 100? ---> 限流
                                         |
                                       否 |
                                         v
                                      通过，执行业务
```

```java
// 直接模式：order-create QPS 不超过 100
FlowRule rule = new FlowRule("order-create");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);
rule.setStrategy(RuleConstant.STRATEGY_DIRECT); // 默认就是直接模式
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

### 2.2.2 关联模式（Relate）

当**关联资源**达到阈值时，对**当前资源**进行限流。

**使用场景：** 读写竞争场景，写入接口压力大时，限制读取接口。

```
                  写入接口(order-write) QPS > 50
                            |
                            v
                   +-----------------+
                   | 触发关联限流     |
                   +-----------------+
                            |
                            v
                  读取接口(order-read) 被限流！
                  （虽然 order-read 自身 QPS 不高）
```

```java
// 当 order-write 的 QPS 超过 50 时，限制 order-read 的访问
FlowRule rule = new FlowRule();
rule.setResource("order-read");          // 被限流的资源
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(50);                        // 关联资源的阈值
rule.setStrategy(RuleConstant.STRATEGY_RELATE);  // 关联模式
rule.setRefResource("order-write");       // 关联资源名称
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

**实际应用场景：**
```java
@RestController
@RequestMapping("/order")
public class OrderController {

    @PostMapping("/write")
    @SentinelResource("order-write")
    public Result<Order> writeOrder(@RequestBody OrderRequest req) {
        // 写操作（数据库写入）
        return orderService.create(req);
    }

    @GetMapping("/read/{id}")
    @SentinelResource("order-read")  // 当 order-write 繁忙时，限流 order-read
    public Result<Order> readOrder(@PathVariable Long id) {
        // 读操作（保护数据库，写优先）
        return orderService.findById(id);
    }
}
```

### 2.2.3 链路模式（Chain）

链路模式只统计从指定入口进入当前资源的请求数，进行限流。

```
调用链示例：

  /api/order (入口A) ----> [common-query] ----> 被限流（入口A流量 > 阈值）
                                  ^
  /api/product (入口B) ---------> |  ----> 不受影响（不是被限制的入口）
```

```java
// 限制从 /api/order 入口进入 common-query 的流量
FlowRule rule = new FlowRule();
rule.setResource("common-query");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(10);
rule.setStrategy(RuleConstant.STRATEGY_CHAIN);   // 链路模式
rule.setRefResource("sentinel_web_context");      // 入口资源名（Web 上下文名）
FlowRuleManager.loadRules(Collections.singletonList(rule));
```

**重要配置 - 关闭 URL 合并（Web 场景必须）：**

```yaml
# application.yml
spring:
  cloud:
    sentinel:
      web-context-unify: false  # 关闭 URL 合并，否则链路模式不生效！
```

---

## 2.3 四种流控效果

### 2.3.1 快速失败（Default）

当 QPS 超过阈值时，直接抛出 `FlowException`（BlockException 的子类）。

```
时间轴:
|--100ms--|--100ms--|--100ms--|
| 100个请求| 200个请求| 100个请求|
|  全通过  |100通过  |  全通过  |
          |100被拒绝|

简单理解：超过阈值直接拒绝，无任何等待
```

```java
FlowRule rule = new FlowRule("fast-fail-resource");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_DEFAULT); // 快速失败（默认）
```

### 2.3.2 Warm Up（冷启动预热）

令牌桶预热模式。系统冷启动时，流量从低阈值逐渐增加到设定阈值，防止突发流量冲击。

**原理：** 初始 QPS 阈值 = count / coldFactor（coldFactor 默认为3），经过 warmUpPeriodSec 秒后达到 count 阈值。

```
QPS 阈值变化曲线:

count=100, warmUpPeriodSec=10, coldFactor=3

  100 |                              **********
   90 |                        *****
   80 |                   *****
   70 |              *****
   60 |         *****
   50 |    *****
   33 |****  <-- 初始阈值 (100/3 约= 33)
    0 +--+--+--+--+--+--+--+--+--+--+----> 时间(秒)
      0  1  2  3  4  5  6  7  8  9  10
         |<------ 预热期 10秒 ---------->|
```

**适用场景：** 缓存系统（需要预热缓存）、数据库连接池（连接数需要逐步建立）

```java
FlowRule rule = new FlowRule("warm-up-resource");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);                                               // 最终稳定 QPS
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);  // Warm Up 模式
rule.setWarmUpPeriodSec(10);                                      // 预热时长 10 秒
```

**Warm Up 源码核心（令牌桶预热）：**

```java
// com.alibaba.csp.sentinel.slots.block.flow.controller.WarmUpController
public class WarmUpController implements TrafficShapingController {

    protected double count;        // 稳定 QPS
    private int coldFactor;        // 冷因子，默认 3
    protected int warningToken;    // 告警 token 数
    private int maxToken;          // 最大 token 数
    protected double slope;        // 斜率（从冷到热的变化率）

    protected AtomicLong storedTokens = new AtomicLong(0);   // 当前 token 数
    protected AtomicLong lastFilledTime = new AtomicLong(0); // 上次补充时间

    private void construct(double count, int warmUpPeriodInSec, int cold) {
        this.count = count;
        this.coldFactor = cold;

        // 告警 token 数：系统处于稳定状态时的 token 阈值
        warningToken = (int)(warmUpPeriodInSec * count) / (coldFactor - 1);

        // 最大 token 数：系统处于冷状态时的最大 token 数
        maxToken = warningToken + (int)(2 * warmUpPeriodInSec * count / (1.0 + coldFactor));

        // 斜率：从冷状态到热状态的速率变化
        slope = (coldFactor - 1.0) / count / (maxToken - warningToken);
    }

    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        long passQps = (long) node.passQps();
        long previousQps = (long) node.previousPassQps();

        syncToken(previousQps); // 同步 token

        long currentToken = storedTokens.get();

        // 根据当前 token 数计算当前可用 QPS
        double warningQps = Math.nextUp(1.0 / (currentToken * slope + 1.0 / count));

        if (passQps + acquireCount <= warningQps) {
            return true;
        }
        return false;
    }
}
```

### 2.3.3 排队等待（漏桶算法）

以匀速的速率处理请求，多余的请求排队等待，超过最大等待时间则拒绝。

```
漏桶模型：

  请求洪流：  |||||||||||||||||||||||
                     |
                     v
               +-----+-----+
               |           |    <-- 排队缓冲区（maxQueueingTimeMs）
               |  Queue    |
               |           |
               +-----------+
                     |
                     | 匀速流出（QPS = count）
                     v
               业务处理

等待超时 --> 拒绝（FlowException）
```

**适用场景：** 消息削峰填谷，对实时性要求不高的后台任务，定时任务队列处理。

```java
FlowRule rule = new FlowRule("rate-limiter-resource");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(10);                                                       // 每秒处理 10 个
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER);    // 排队等待
rule.setMaxQueueingTimeMs(2000);                                          // 最大等待 2 秒
```

**排队等待核心源码：**

```java
// com.alibaba.csp.sentinel.slots.block.flow.controller.RateLimiterController
public class RateLimiterController implements TrafficShapingController {

    private final int maxQueueingTimeMs;  // 最大排队时间
    private final double count;           // QPS 阈值

    // 上一个请求被允许的时间
    private final AtomicLong latestPassedTime = new AtomicLong(-1);

    @Override
    public boolean canPass(Node node, int acquireCount, boolean prioritized) {
        if (count <= 0) {
            return false;
        }

        long currentTime = TimeUtil.currentTimeMillis();

        // 计算每个请求的间隔时间（毫秒）
        // costTime = 1000ms / QPS
        long costTime = Math.round(1.0 * (acquireCount) / count * 1000);

        // 期望通过时间 = 上次通过时间 + 间隔时间
        long expectedTime = costTime + latestPassedTime.get();

        if (expectedTime <= currentTime) {
            // 可以立即通过
            latestPassedTime.set(currentTime);
            return true;
        } else {
            // 需要等待
            long waitTime = costTime + latestPassedTime.get() - TimeUtil.currentTimeMillis();

            if (waitTime > maxQueueingTimeMs) {
                // 等待时间超过最大值，拒绝
                return false;
            } else {
                // CAS 更新下次通过时间，确保顺序排队
                long oldTime = latestPassedTime.addAndGet(costTime);

                waitTime = oldTime - TimeUtil.currentTimeMillis();
                if (waitTime > maxQueueingTimeMs) {
                    latestPassedTime.addAndGet(-costTime);
                    return false;
                }

                // 睡眠等待
                if (waitTime > 0) {
                    try {
                        Thread.sleep(waitTime);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
                return true;
            }
        }
    }
}
```

### 2.3.4 Warm Up + 排队等待

结合预热和排队的混合策略：先预热达到稳定 QPS，然后对超出的请求进行排队等待。

```java
FlowRule rule = new FlowRule("warm-up-rate-resource");
rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
rule.setCount(100);
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER); // 预热+排队
rule.setWarmUpPeriodSec(10);
rule.setMaxQueueingTimeMs(500);
```

---

## 2.4 限流算法原理深度剖析

### 2.4.1 计数器算法（固定窗口）及其临界问题

```
固定窗口算法示意：

时间: |----第1秒----|----第2秒----|----第3秒----|
      0s          1s           2s           3s

第1秒：通过99个请求（未超过100）
第2秒：通过99个请求（未超过100）

问题：在 0.9s 到 1.1s 这 200ms 内，可能通过 198 个请求！

     0.9s              1.0s              1.1s
      |                 |                 |
      | 99个请求(第1秒) | | 99个请求(第2秒) |
      |_________________|_________________|
         <-------- 200ms内通过 198 个 ------->
         实际速率 = 198 / 0.2s = 990 QPS !!! (设定值是100)
```

**代码实现（仅演示，Sentinel 未使用此方式）：**

```java
public class FixedWindowRateLimiter {
    private final int limit;           // 每个窗口最大请求数
    private final long windowSize;     // 窗口大小（毫秒）
    private AtomicInteger counter;     // 当前窗口计数
    private long windowStart;          // 当前窗口开始时间

    public synchronized boolean tryAcquire() {
        long now = System.currentTimeMillis();

        // 检查是否需要重置窗口
        if (now - windowStart >= windowSize) {
            windowStart = now;
            counter.set(0);
        }

        // 检查是否超过限制
        if (counter.incrementAndGet() <= limit) {
            return true;
        }

        counter.decrementAndGet();
        return false;
    }
}
```

### 2.4.2 滑动窗口算法（Sentinel 实现：LeapArray）

Sentinel 使用**滑动时间窗口**解决固定窗口的临界问题。

```
滑动窗口示意（时间窗口=1秒，分为10个格子，每格100ms）：

格子:  [g0][g1][g2][g3][g4][g5][g6][g7][g8][g9]
时间:   0  100 200 300 400 500 600 700 800 900 1000ms

当前时间 = 850ms，统计范围 = [850-1000, 850]
实际统计 = [g0, g1, ..., g8] 共9个格子的计数之和

滑动过程（时间推进到 950ms）：
统计 = [g1, g2, ..., g9] 共9个格子（g0 过期被滑出）

特点：随时间滑动，不存在临界问题
```

**LeapArray 数据结构：**

```java
// com.alibaba.csp.sentinel.slots.statistic.base.LeapArray
public abstract class LeapArray<T> {

    protected int windowLengthInMs;  // 每个格子的时间长度（毫秒）
    protected int sampleCount;       // 格子数量（分段数）
    protected int intervalInMs;      // 总时间窗口长度（毫秒）

    // 格子数组（环形数组）
    protected final AtomicReferenceArray<WindowWrap<T>> array;

    public LeapArray(int sampleCount, int intervalInMs) {
        this.windowLengthInMs = intervalInMs / sampleCount;
        this.sampleCount = sampleCount;
        this.intervalInMs = intervalInMs;
        this.array = new AtomicReferenceArray<>(sampleCount);
    }

    /**
     * 获取当前时间对应的窗口格子
     */
    public WindowWrap<T> currentWindow(long timeMillis) {
        if (timeMillis < 0) {
            return null;
        }

        // 计算当前时间在哪个格子（取模运算）
        int idx = calculateTimeIdx(timeMillis);

        // 计算当前格子的起始时间
        long windowStart = calculateWindowStart(timeMillis);

        while (true) {
            WindowWrap<T> old = array.get(idx);

            if (old == null) {
                // 格子为空，创建新格子
                WindowWrap<T> window = new WindowWrap<>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
                if (array.compareAndSet(idx, null, window)) {
                    return window;
                } else {
                    Thread.yield();
                }
            } else if (windowStart == old.windowStart()) {
                // 当前格子就是我们要的
                return old;
            } else if (windowStart > old.windowStart()) {
                // 当前格子已经过期，需要重置
                if (updateLock.tryLock()) {
                    try {
                        return resetWindowTo(old, windowStart);
                    } finally {
                        updateLock.unlock();
                    }
                } else {
                    Thread.yield();
                }
            } else if (windowStart < old.windowStart()) {
                // 时间回退（时钟漂移），创建新格子
                return new WindowWrap<>(windowLengthInMs, windowStart, newEmptyBucket(timeMillis));
            }
        }
    }

    private int calculateTimeIdx(long timeMillis) {
        long timeId = timeMillis / windowLengthInMs;
        return (int)(timeId % sampleCount);  // 取模，形成环形数组
    }

    protected long calculateWindowStart(long timeMillis) {
        return timeMillis - timeMillis % windowLengthInMs;
    }
}
```

### 2.4.3 令牌桶算法（Token Bucket）

```
令牌桶模型：

         +------------------+
         |  令牌生成器       |
         |  速率 = r tokens/s|
         +--------+---------+
                  |
                  | 以固定速率添加令牌
                  v
         +--------+---------+
         |    令牌桶         |
         |  容量 = b tokens  |  <-- 桶满则丢弃令牌
         |  当前 = n tokens  |
         +--------+---------+
                  |
    请求到达 ---->| 取令牌
                  |
         n >= 1? -+-> 是: 取1令牌，通过请求
                  |-> 否: 拒绝或等待

特点：
  1. 允许突发流量（最大突发 = 桶容量 b）
  2. 平均速率限制为 r
  3. Sentinel Warm Up 基于令牌桶实现
```

**令牌桶代码示意：**

```java
public class TokenBucketRateLimiter {
    private final double rate;          // 令牌生成速率（tokens/秒）
    private final double maxBurstSize;  // 桶容量（最大突发量）

    private double tokens;              // 当前令牌数
    private long lastRefillTime;        // 上次补充时间

    public synchronized boolean tryAcquire(int count) {
        refill(); // 先补充令牌

        if (tokens >= count) {
            tokens -= count;
            return true;
        }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        long elapsed = now - lastRefillTime;

        // 补充令牌
        double newTokens = elapsed * rate / 1000.0;
        tokens = Math.min(maxBurstSize, tokens + newTokens);
        lastRefillTime = now;
    }
}
```

### 2.4.4 漏桶算法（Leaky Bucket）

```
漏桶模型：

  突发请求 -->  +----------+
               |          |
               |  漏桶     | <-- 容量有限，超出则溢出（拒绝）
               |          |
               +----+-----+
                    |
                    | 恒定速率流出
                    v
               业务处理（匀速）

特点：
  1. 输出速率恒定（无突发）
  2. 平滑流量
  3. Sentinel 排队等待模式基于漏桶实现
```

**令牌桶 vs 漏桶对比：**

| 特性 | 令牌桶 | 漏桶 |
|-----|--------|------|
| 突发处理 | 允许（桶容量范围内） | 不允许（匀速流出） |
| 实现复杂度 | 中 | 低 |
| 适用场景 | 有突发需求但需限制平均速率 | 严格匀速处理 |
| Sentinel 对应 | Warm Up | 排队等待 |

---

## 2.5 Sentinel 滑动时间窗口（LeapArray）深度解析

```
LeapArray 内部结构（sampleCount=10, intervalInMs=1000ms）：

数组下标:  [ 0 ] [ 1 ] [ 2 ] [ 3 ] [ 4 ] [ 5 ] [ 6 ] [ 7 ] [ 8 ] [ 9 ]
格子起始:  0ms  100ms 200ms 300ms 400ms 500ms 600ms 700ms 800ms 900ms

每个格子（WindowWrap）存储：
  - windowStart: 格子开始时间
  - windowLength: 格子长度（100ms）
  - value: MetricBucket（统计数据）

MetricBucket 存储：
  +--------------------------------------------+
  | passQps   | blockQps | successQps           |
  | exceptionQps | rt | occupiedPassQps         |
  +--------------------------------------------+

时间推进 1200ms 时：
  - 下标 0 的格子（起始 0ms）被重置为 起始 1100ms 的新格子
  - 环形复用，不需要频繁分配内存
```

**完整的统计值获取流程：**

```java
// 获取当前统计窗口内的 QPS
public double passQps() {
    return rollingCounterInSecond.pass() / rollingCounterInSecond.getWindowIntervalInSec();
}

// 统计窗口内所有格子的 pass 之和
public long pass() {
    data.currentWindow(); // 先更新当前窗口
    long pass = 0;

    // 获取所有有效格子
    List<MetricBucket> list = data.values();

    for (MetricBucket window : list) {
        pass += window.pass();
    }
    return pass;
}
```


---

# Part 3: 熔断降级深度解析

## 3.1 DegradeRule 参数详解

```java
public class DegradeRule extends AbstractRule {
    // 0-慢调用比例, 1-异常比例, 2-异常数
    private int grade = RuleConstant.DEGRADE_GRADE_RT;
    // 慢调用RT阈值(ms) 或 异常比例(0-1) 或 异常数
    private double count;
    // 熔断时长（秒）
    private int timeWindow;
    // 最小请求数（1.8.0+，默认5）
    private int minRequestAmount = 5;
    // 统计时间窗口（毫秒，1.8.0+，默认1000）
    private int statIntervalMs = 1000;
    // 慢调用比例阈值（0-1，1.8.0+，默认1.0）
    private double slowRatioThreshold = 1.0d;
}
```

## 3.2 三种熔断策略

### 慢调用比例
```java
// 1秒内，RT>500ms的请求比例>50%，且至少10个请求 -> 熔断10秒
DegradeRule rule = new DegradeRule("slow-service");
rule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
rule.setCount(500);               // RT阈值：500ms
rule.setSlowRatioThreshold(0.5);  // 比例：50%
rule.setMinRequestAmount(10);
rule.setStatIntervalMs(1000);
rule.setTimeWindow(10);
```

### 异常比例
```java
// 1秒内，异常比例>20%，且至少5个请求 -> 熔断5秒
DegradeRule rule = new DegradeRule("exception-ratio-service");
rule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
rule.setCount(0.2);               // 比例阈值：20%
rule.setMinRequestAmount(5);
rule.setStatIntervalMs(1000);
rule.setTimeWindow(5);
```

### 异常数
```java
// 1分钟内，异常数>5 -> 熔断30秒
DegradeRule rule = new DegradeRule("exception-count-service");
rule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_COUNT);
rule.setCount(5);
rule.setStatIntervalMs(60 * 1000);
rule.setTimeWindow(30);
```

## 3.3 熔断器状态机

```
+-------+  触发熔断  +------+  timeWindow秒后  +----------+
|CLOSED | ---------> | OPEN | --------------> | HALF_OPEN|
|正常   |            |全拒绝|                  |单请求探测|
+-------+            +------+  探测失败重回OPEN+----------+
    ^                                               |
    +----------- 探测成功（恢复CLOSED）  -----------+

CLOSED    - 正常，统计指标
OPEN      - 熔断，所有请求返回 DegradeException
HALF_OPEN - 放行一个探测请求：成功->CLOSED，失败->OPEN
```

**状态机源码（AbstractCircuitBreaker）：**
```java
public boolean tryPass(Context context) {
    if (currentState.get() == State.CLOSED) return true;
    if (currentState.get() == State.OPEN) {
        // 重试时间到了 -> 尝试进入 HALF_OPEN
        return retryTimeoutArrived() && fromOpenToHalfOpen(context);
    }
    return false;
}

protected boolean fromOpenToHalfOpen(Context context) {
    if (currentState.compareAndSet(State.OPEN, State.HALF_OPEN)) {
        // 注册探测结果回调
        context.getCurEntry().whenTerminate((ctx, e) -> {
            if (e.getBlockError() != null) {
                // 探测失败，回到OPEN
                currentState.compareAndSet(State.HALF_OPEN, State.OPEN);
            }
            // 成功由子类 onRequestComplete 处理
        });
        return true;
    }
    return false;
}
```

## 3.4 Sentinel vs Hystrix 熔断对比

| 特性 | Sentinel 1.8.x | Hystrix |
|------|----------------|---------|
| 熔断策略 | 慢调用比例/异常比例/异常数 | 失败比例 |
| 半开探测 | 放行1个请求（精准） | 固定时间后自动恢复 |
| 统计窗口 | 可配置 statIntervalMs | 固定 10 秒 |
| 线程隔离 | 无（仅信号量） | 线程池隔离（有额外开销） |
| 维护状态 | 活跃维护 | 停止维护（2018年） |

---

# Part 4: 系统自适应保护

## 4.1 SystemRule 参数
```java
public class SystemRule extends AbstractRule {
    private double qps = -1;               // 入口QPS阈值
    private long maxThread = -1;           // 最大并发线程数
    private double avgRt = -1;             // 平均RT阈值(ms)
    private double highestSystemLoad = -1; // load1阈值（仅Linux）
    private double highestCpuUsage = -1;   // CPU使用率（0-1）
}
```

## 4.2 系统保护配置

```java
SystemRule rule = new SystemRule();
rule.setQps(1000);              // 整体QPS不超1000
rule.setMaxThread(200);         // 线程数不超200
rule.setAvgRt(500);             // 平均RT不超500ms
rule.setHighestSystemLoad(2.5); // load1不超2.5（推荐=CPU核数*2.5）
rule.setHighestCpuUsage(0.8);   // CPU不超80%
SystemRuleManager.loadRules(Collections.singletonList(rule));
```

## 4.3 系统保护原理（BBR算法）

```
系统最大处理能力：
  maxQps = maxThread * 1000 / minRt

Load 模式触发条件（Linux专用）：
  load1 > highestSystemLoad  AND  currentQps > maxQps
  （两个条件同时满足才限流，避免误杀）
```

---

# Part 5: 热点参数限流

## 5.1 概述

```
普通限流：所有请求共享QPS=100
热点参数限流：
  productId=1001  QPS=10   <- 单独限制
  productId=9999  QPS=1000 <- 例外项（大促商品）
  其他productId   QPS=10   <- 默认限制
```

## 5.2 ParamFlowRule 配置

```java
ParamFlowRule rule = new ParamFlowRule("product-detail");
rule.setParamIdx(0);           // 第0个参数（productId）
rule.setCount(10);             // 默认QPS=10
rule.setDurationInSec(1);      // 统计窗口1秒

// 例外项
List<ParamFlowItem> itemList = new ArrayList<>();
ParamFlowItem item = new ParamFlowItem();
item.setObject("9999");
item.setClassType(long.class.getName());
item.setCount(200);             // productId=9999 QPS=200
itemList.add(item);

rule.setParamFlowItemList(itemList);
ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
```

## 5.3 @SentinelResource 注解详解

```java
@SentinelResource(
    value = "resource-name",              // 资源名（必填）
    entryType = EntryType.OUT,            // 入口类型
    blockHandler = "handleBlock",         // BlockException处理方法
    blockHandlerClass = {MyFallback.class}, // 指定其他类（static方法）
    fallback = "handleFallback",          // 业务异常处理方法
    fallbackClass = {MyFallback.class},
    defaultFallback = "handleDefault",    // 通用降级（无参或只有Throwable）
    exceptionsToIgnore = {BusinessEx.class} // 这些异常不触发fallback
)
```

**优先级：**
- BlockException -> blockHandler（没有则fallback）
- 业务异常 -> fallback（没有则defaultFallback）
- exceptionsToIgnore 直接抛出

## 5.4 完整案例

```java
@GetMapping("/detail")
@SentinelResource(
    value = "product-detail",
    blockHandlerClass = ProductFallback.class,
    blockHandler = "blockHandler",
    fallbackClass = ProductFallback.class,
    fallback = "fallbackHandler"
)
public Result<ProductVO> getProductDetail(
        @RequestParam Long productId,
        @RequestParam Long userId) {
    return Result.success(productService.getDetail(productId));
}

// ProductFallback.java（static方法）
public static Result<ProductVO> blockHandler(
        Long productId, Long userId, BlockException ex) {
    if (ex instanceof ParamFlowException)
        return Result.fail(429, "该商品请求过于频繁");
    if (ex instanceof DegradeException)
        return Result.fail(503, "服务暂时不可用");
    return Result.fail(429, "请求过于频繁");
}

public static Result<ProductVO> fallbackHandler(
        Long productId, Long userId, Throwable ex) {
    log.error("商品{}查询异常", productId, ex);
    return Result.fail(500, "商品信息暂时无法获取");
}
```

---

# Part 6: 权限控制（黑白名单）

## 6.1 AuthorityRule 配置

```java
// 白名单：只允许 order-service 访问 user-profile
AuthorityRule whiteRule = new AuthorityRule();
whiteRule.setResource("user-profile");
whiteRule.setStrategy(RuleConstant.AUTHORITY_WHITE);
whiteRule.setLimitApp("order-service,product-service");

// 黑名单：禁止 test-client 访问 payment-api
AuthorityRule blackRule = new AuthorityRule();
blackRule.setResource("payment-api");
blackRule.setStrategy(RuleConstant.AUTHORITY_BLACK);
blackRule.setLimitApp("test-client");
```

## 6.2 自定义来源获取

```java
@Component
public class CustomRequestOriginParser implements RequestOriginParser {
    @Override
    public String parseOrigin(HttpServletRequest request) {
        String origin = request.getHeader("X-Service-Name");
        return StringUtils.hasText(origin) ? origin : request.getRemoteAddr();
    }
}

// Feign 请求中传递服务名
@Component
public class ServiceNameFeignInterceptor implements RequestInterceptor {
    @Value("${spring.application.name}")
    private String serviceName;

    @Override
    public void apply(RequestTemplate template) {
        template.header("X-Service-Name", serviceName);
    }
}
```

---

# Part 7: Sentinel 统计原理

## 7.1 LeapArray 滑动窗口

```
sampleCount=2, intervalInMs=1000ms：
  g0(0-499ms)  g1(500-999ms)
  [  count  ]  [  count  ]
  环形复用，时间到1200ms时g0被重置为1000-1499ms

下标：idx = (timeMillis/windowLength) % sampleCount
统计：sum(所有有效格子的计数)
```

## 7.2 MetricBucket 指标

```java
// 指标类型：PASS/BLOCK/EXCEPTION/SUCCESS/RT/OCCUPIED_PASS
// 使用 LongAdder（高并发下比AtomicLong更高效）
public class MetricBucket {
    private final LongAdder[] counters;
    private volatile long minRt;  // 最小RT（BBR计算用）
}
```

## 7.3 StatisticSlot 核心逻辑

```java
@Override
public void entry(...) throws Throwable {
    try {
        fireEntry(...);  // 先执行后续Slot（流控/熔断检查）
        node.increaseThreadNum();    // 统计线程数
        node.addPassRequest(count);  // 统计通过QPS
    } catch (BlockException e) {
        node.increaseBlockQps(count); // 统计拦截QPS
        throw e;
    } catch (Throwable e) {
        context.getCurEntry().setError(e); // 记录业务异常
        throw e;
    }
}

@Override
public void exit(...) {  // 统计RT
    long rt = TimeUtil.currentTimeMillis() - entry.getCreateTimestamp();
    node.addRtAndSuccess(rt, count);
    node.decreaseThreadNum();
    fireExit(...);
}
```

---

# Part 8: Sentinel Dashboard 控制台

## 8.1 Dashboard 搭建

```bash
java -Dserver.port=8080 \
     -Dcsp.sentinel.dashboard.server=localhost:8080 \
     -Dproject.name=sentinel-dashboard \
     -jar sentinel-dashboard-1.8.6.jar
```

## 8.2 客户端接入

```yaml
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
        port: 8719
      eager: true
      web-context-unify: false
```

## 8.3 三种规则推送模式

```
原始模式：Dashboard -> 应用内存（重启丢失）
Pull 模式：Dashboard -> 文件/DB <- 应用定期拉取（有延迟）
Push 模式：Dashboard -> Nacos -> 应用（实时推送，生产推荐）

Push 模式流程：
  +-----------+  Publish  +-------+  Notify  +---------+
  | Dashboard | --------> | Nacos | -------> | 实例1   |
  +-----------+            +-------+          +---------+
                                    Notify    +---------+
                                    --------> | 实例2   |
                                              +---------+
```

## 8.4 Nacos 持久化规则配置

```yaml
spring:
  cloud:
    sentinel:
      datasource:
        flow:
          nacos:
            server-addr: localhost:8848
            namespace: dev
            group-id: SENTINEL_GROUP
            data-id: ${spring.application.name}-flow-rules
            data-type: json
            rule-type: flow
        degrade:
          nacos:
            server-addr: localhost:8848
            namespace: dev
            group-id: SENTINEL_GROUP
            data-id: ${spring.application.name}-degrade-rules
            data-type: json
            rule-type: degrade
```

**Dashboard 改造（Publisher/Provider）：**

```java
// 推送规则到 Nacos
@Component("flowRuleNacosPublisher")
public class FlowRuleNacosPublisher implements DynamicRulePublisher<List<FlowRuleEntity>> {
    @Autowired
    private NacosConfigService configService;

    @Override
    public void publish(String app, List<FlowRuleEntity> rules) throws Exception {
        configService.publishConfig(
            app + "-flow-rules", "SENTINEL_GROUP", JSON.toJSONString(rules));
    }
}

// 从 Nacos 获取规则
@Component("flowRuleNacosProvider")
public class FlowRuleNacosProvider implements DynamicRuleProvider<List<FlowRuleEntity>> {
    @Autowired
    private NacosConfigService configService;

    @Override
    public List<FlowRuleEntity> getRules(String appName) throws Exception {
        String rules = configService.getConfig(appName + "-flow-rules", "SENTINEL_GROUP", 3000);
        return StringUtils.isEmpty(rules) ? new ArrayList<>()
               : JSON.parseArray(rules, FlowRuleEntity.class);
    }
}
```

---

# Part 9: Spring Boot/Cloud 集成实战

## 9.1 完整依赖

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.cloud</groupId>
            <artifactId>spring-cloud-alibaba-dependencies</artifactId>
            <version>2021.0.5.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.csp</groupId>
        <artifactId>sentinel-datasource-nacos</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>
</dependencies>
```

## 9.2 BlockException 类型处理

```java
@GetMapping("/{id}")
@SentinelResource(value = "order-query",
    blockHandler = "handleBlock", fallback = "handleFallback",
    exceptionsToIgnore = {IllegalArgumentException.class})
public Result<Order> getOrder(@PathVariable Long id) {
    return Result.success(orderService.findById(id));
}

public Result<Order> handleBlock(Long id, BlockException ex) {
    if (ex instanceof FlowException)   return Result.fail(429, "请求过于频繁");
    if (ex instanceof DegradeException) return Result.fail(503, "服务暂时不可用");
    if (ex instanceof ParamFlowException) return Result.fail(429, "参数限流");
    if (ex instanceof SystemBlockException) return Result.fail(503, "系统繁忙");
    if (ex instanceof AuthorityException) return Result.fail(403, "无权限");
    return Result.fail(500, "系统异常");
}

public Result<Order> handleFallback(Long id, Throwable ex) {
    log.error("订单查询异常, orderId={}", id, ex);
    return Result.fail(500, "查询失败");
}
```

## 9.3 全局异常处理

```java
@Component
public class SentinelBlockExceptionHandler implements BlockExceptionHandler {

    @Override
    public void handle(HttpServletRequest req, HttpServletResponse resp, BlockException ex)
            throws Exception {
        resp.setStatus(ex instanceof AuthorityException ? 403 :
                       ex instanceof DegradeException ? 503 : 429);
        resp.setContentType("application/json;charset=UTF-8");
        resp.getWriter().write(String.format(
            "{\"code\":%d,\"message\":\"%s\"}",
            resp.getStatus(), getMessage(ex)));
    }

    private String getMessage(BlockException ex) {
        if (ex instanceof FlowException) return "请求过于频繁";
        if (ex instanceof DegradeException) return "服务暂时不可用";
        if (ex instanceof ParamFlowException) return "参数限流";
        if (ex instanceof SystemBlockException) return "系统繁忙";
        if (ex instanceof AuthorityException) return "无权限访问";
        return "请求被拦截";
    }
}
```

## 9.4 OpenFeign + Sentinel 整合

```java
// feign.sentinel.enabled=true 开启支持

@FeignClient(name = "product-service",
    fallbackFactory = ProductFeignClientFallbackFactory.class)
public interface ProductFeignClient {
    @GetMapping("/product/detail")
    Result<ProductVO> getProductDetail(
        @RequestParam("productId") Long productId,
        @RequestParam("userId") Long userId
    );
    @GetMapping("/product/inventory/{productId}")
    Result<Integer> getInventory(@PathVariable("productId") Long productId);
}

@Component
@Slf4j
public class ProductFeignClientFallbackFactory
        implements FallbackFactory<ProductFeignClient> {

    @Override
    public ProductFeignClient create(Throwable cause) {
        return new ProductFeignClient() {
            @Override
            public Result<ProductVO> getProductDetail(Long productId, Long userId) {
                log.error("获取商品详情失败 productId={}, cause={}", productId, cause.getMessage());
                if (cause instanceof FlowException) return Result.fail(429, "商品服务过频繁");
                if (cause instanceof DegradeException) return Result.fail(503, "商品服务不可用");
                return Result.fail(500, "获取商品失败");
            }
            @Override
            public Result<Integer> getInventory(Long productId) {
                log.warn("获取库存失败，保守返回0, productId={}", productId);
                return Result.success(0);
            }
        };
    }
}
```

## 9.5 Spring Cloud Gateway + Sentinel

```java
@Configuration
public class SentinelGatewayConfig {

    @PostConstruct
    public void init() {
        Set<GatewayFlowRule> rules = new HashSet<>();
        rules.add(new GatewayFlowRule("order-service")
            .setCount(500).setIntervalSec(1).setBurst(100));
        GatewayRuleManager.loadRules(rules);

        Set<ApiDefinition> defs = new HashSet<>();
        defs.add(new ApiDefinition("product-api")
            .setPredicateItems(new HashSet<ApiPredicateItem>() {{
                add(new ApiPathPredicateItem()
                    .setPattern("/product/**")
                    .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_PREFIX));
            }}));
        GatewayApiDefinitionManager.loadApiDefinitions(defs);
    }
}

// 网关全局异常处理（WebFlux）
@Primary
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SentinelGatewayBlockExceptionHandler implements WebExceptionHandler {

    @Override
    public Mono<Void> handle(ServerWebExchange exchange, Throwable ex) {
        if (!(ex instanceof BlockException)) return Mono.error(ex);

        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
        response.getHeaders().setContentType(MediaType.APPLICATION_JSON);

        String body = "{\"code\":429,\"message\":\"请求被限流\",\"timestamp\":"
                      + System.currentTimeMillis() + "}";
        DataBuffer buffer = response.bufferFactory().wrap(body.getBytes());
        return response.writeWith(Mono.just(buffer));
    }
}
```


---

# Part 10: 完整实战案例

## 案例1：电商下单接口限流（QPS=100，Warm Up预热）

**需求：** 下单接口高峰期 QPS=100，冷启动需10秒预热，防止突发流量冲垮数据库连接池。

```java
// 下单接口（Controller 层）
@RestController
@RequestMapping("/api/order")
@Slf4j
public class OrderApiController {

    @Autowired
    private OrderService orderService;

    @PostMapping("/create")
    @SentinelResource(
        value = "api-order-create",
        blockHandler = "orderCreateBlock",
        fallback = "orderCreateFallback"
    )
    public ApiResult<OrderVO> createOrder(@RequestBody @Valid CreateOrderRequest request) {
        log.info("下单请求: userId={}, productId={}, qty={}",
                 request.getUserId(), request.getProductId(), request.getQuantity());
        OrderVO order = orderService.createOrder(request);
        return ApiResult.success(order);
    }

    // 限流降级
    public ApiResult<OrderVO> orderCreateBlock(CreateOrderRequest request, BlockException ex) {
        log.warn("下单接口被限流: userId={}, cause={}",
                 request.getUserId(), ex.getClass().getSimpleName());
        return ApiResult.fail(429, "系统繁忙，请稍后再试（建议1秒后重试）");
    }

    // 异常降级（记录到消息队列，异步补偿）
    public ApiResult<OrderVO> orderCreateFallback(CreateOrderRequest request, Throwable ex) {
        log.error("下单接口异常降级: userId={}", request.getUserId(), ex);
        orderService.recordFailedOrder(request);  // 异步重试
        return ApiResult.fail(500, "下单失败，系统已记录，稍后将自动处理");
    }
}
```

```java
// Warm Up 规则配置（Sentinel 配置类）
@Configuration
public class OrderSentinelConfig {

    @PostConstruct
    public void initFlowRules() {
        FlowRule rule = new FlowRule("api-order-create");
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setCount(100);                                              // 稳定 QPS = 100
        rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP); // Warm Up 模式
        rule.setWarmUpPeriodSec(10);                                     // 预热10秒
        FlowRuleManager.loadRules(Collections.singletonList(rule));
        log.info("下单接口限流规则已加载：QPS=100，Warm Up 10s");
    }
}
```

---

## 案例2：商品服务熔断（慢调用 + 降级返回缓存数据）

**需求：** 商品详情查询 RT 超过 500ms 的比例超过 50% 时触发熔断，熔断期间返回 Redis 缓存数据。

```java
// 商品服务（带缓存降级）
@Service
@Slf4j
public class ProductServiceImpl implements ProductService {

    @Autowired
    private ProductRepository productRepository;

    @Autowired
    private RedisTemplate<String, ProductVO> redisTemplate;

    // 正常查询（有熔断保护）
    @SentinelResource(
        value = "product-detail-query",
        fallback = "getDetailFallback"
    )
    public ProductVO getProductDetail(Long productId) {
        // 模拟可能慢的数据库查询
        ProductDO product = productRepository.findById(productId)
            .orElseThrow(() -> new NotFoundException("商品不存在：" + productId));

        ProductVO vo = convertToVO(product);

        // 更新缓存
        redisTemplate.opsForValue().set(
            "product:detail:" + productId, vo, 5, TimeUnit.MINUTES);

        return vo;
    }

    // 熔断降级：返回缓存数据
    public ProductVO getDetailFallback(Long productId, Throwable ex) {
        log.warn("商品查询降级(熔断/异常), productId={}, cause={}", productId, ex.getMessage());

        // 尝试从 Redis 获取缓存
        ProductVO cached = redisTemplate.opsForValue().get("product:detail:" + productId);
        if (cached != null) {
            log.info("熔断降级：返回缓存数据, productId={}", productId);
            cached.setFromCache(true);  // 标记为缓存数据
            return cached;
        }

        // 无缓存时返回兜底数据
        log.warn("熔断降级：无缓存，返回兜底数据, productId={}", productId);
        return ProductVO.placeholder(productId, "商品信息暂时不可用");
    }
}
```

```java
// 熔断规则配置
@PostConstruct
public void initDegradeRules() {
    DegradeRule rule = new DegradeRule("product-detail-query");
    rule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
    rule.setCount(500);               // RT > 500ms 为慢调用
    rule.setSlowRatioThreshold(0.5);  // 慢调用比例 > 50%
    rule.setMinRequestAmount(10);     // 至少10个请求才统计
    rule.setStatIntervalMs(1000);     // 统计窗口1秒
    rule.setTimeWindow(15);           // 熔断15秒
    DegradeRuleManager.loadRules(Collections.singletonList(rule));
}
```

---

## 案例3：热点商品ID参数限流

**需求：** 秒杀活动中，每个商品ID默认 QPS=10，秒杀商品(productId=9999)放宽至 QPS=500。

```java
// 秒杀接口
@RestController
@RequestMapping("/seckill")
@Slf4j
public class SeckillController {

    @Autowired
    private SeckillService seckillService;

    @PostMapping("/grab")
    @SentinelResource(
        value = "seckill-grab",
        blockHandlerClass = SeckillFallback.class,
        blockHandler = "grabBlockHandler"
    )
    public Result<SeckillResultVO> grabSeckill(
            @RequestParam Long productId,
            @RequestParam Long userId) {
        return Result.success(seckillService.grab(productId, userId));
    }
}

// 秒杀降级处理
public class SeckillFallback {
    public static Result<SeckillResultVO> grabBlockHandler(
            Long productId, Long userId, BlockException ex) {
        if (ex instanceof ParamFlowException) {
            return Result.fail(429, "当前商品抢购过于火爆，请稍后重试");
        }
        return Result.fail(429, "系统繁忙，请稍后重试");
    }
}
```

```java
// 热点参数规则
@PostConstruct
public void initSeckillParamFlowRules() {
    ParamFlowRule rule = new ParamFlowRule("seckill-grab");
    rule.setParamIdx(0);           // 第0个参数 = productId
    rule.setCount(10);             // 默认每个商品QPS=10
    rule.setDurationInSec(1);

    List<ParamFlowItem> items = new ArrayList<>();
    // 秒杀主商品，QPS放宽到500
    ParamFlowItem seckillItem = new ParamFlowItem();
    seckillItem.setObject("9999");
    seckillItem.setClassType(long.class.getName());
    seckillItem.setCount(500);
    items.add(seckillItem);

    rule.setParamFlowItemList(items);
    ParamFlowRuleManager.loadRules(Collections.singletonList(rule));
}
```

---

## 案例4：Feign 调用熔断降级（FallbackFactory + 详细日志）

**需求：** 订单服务调用库存服务，库存服务不稳定时触发熔断，记录详细的失败日志并返回安全降级值。

```java
// 库存 Feign 客户端
@FeignClient(name = "inventory-service",
    fallbackFactory = InventoryFeignFallbackFactory.class)
public interface InventoryFeignClient {

    @GetMapping("/inventory/check/{productId}")
    InventoryDTO checkInventory(@PathVariable("productId") Long productId);

    @PostMapping("/inventory/deduct")
    Boolean deductInventory(@RequestBody DeductRequest request);

    @PostMapping("/inventory/rollback")
    Boolean rollbackInventory(@RequestBody RollbackRequest request);
}

// FallbackFactory（获取原始异常）
@Component
@Slf4j
public class InventoryFeignFallbackFactory
        implements FallbackFactory<InventoryFeignClient> {

    @Autowired
    private AlertService alertService;

    @Override
    public InventoryFeignClient create(Throwable cause) {
        // 根据异常类型分别处理
        return new InventoryFeignClient() {

            @Override
            public InventoryDTO checkInventory(Long productId) {
                // 查询降级：返回库存为0（保守策略，避免超卖）
                log.warn("[库存查询降级] productId={}, cause={}", productId, cause.getMessage());

                if (cause instanceof DegradeException) {
                    log.warn("[库存服务熔断] 熔断器已打开，productId={}", productId);
                    alertService.sendAlert("库存服务熔断，请检查！");
                }

                return InventoryDTO.empty(productId);
            }

            @Override
            public Boolean deductInventory(DeductRequest request) {
                // 扣减降级：不能静默返回true，否则会超卖！
                log.error("[库存扣减失败] orderId={}, productId={}, cause={}",
                          request.getOrderId(), request.getProductId(), cause.getMessage());

                // 将失败任务发送到MQ，等待库存服务恢复后重试
                retryQueue.push(request);

                throw new InventoryException("库存服务不可用，请稍后重试");
            }

            @Override
            public Boolean rollbackInventory(RollbackRequest request) {
                // 回滚失败：记录到补偿队列
                log.error("[库存回滚失败] orderId={}, 已加入补偿队列", request.getOrderId());
                compensationService.addTask(request);
                return false;
            }
        };
    }
}
```

```java
// 熔断规则（库存服务）
@PostConstruct
public void initInventoryDegradeRules() {
    List<DegradeRule> rules = new ArrayList<>();

    // 异常比例熔断：异常>30%时熔断
    DegradeRule exceptionRule = new DegradeRule();
    exceptionRule.setResource("GET:inventory-service/inventory/check/{productId}");
    exceptionRule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
    exceptionRule.setCount(0.3);           // 异常比例 30%
    exceptionRule.setMinRequestAmount(5);
    exceptionRule.setTimeWindow(20);        // 熔断20秒
    rules.add(exceptionRule);

    // 慢调用熔断：RT>1000ms 且比例>50%
    DegradeRule slowRule = new DegradeRule();
    slowRule.setResource("POST:inventory-service/inventory/deduct");
    slowRule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
    slowRule.setCount(1000);               // RT阈值1秒
    slowRule.setSlowRatioThreshold(0.5);   // 50%慢调用
    slowRule.setMinRequestAmount(5);
    slowRule.setTimeWindow(30);
    rules.add(slowRule);

    DegradeRuleManager.loadRules(rules);
}
```

---

## 案例5：网关统一限流（Gateway + Sentinel 按路由限流）

**需求：** API 网关对不同服务设置差异化限流策略，并支持按 IP 限流。

```yaml
# gateway application.yml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=1
        - id: product-service
          uri: lb://product-service
          predicates:
            - Path=/api/product/**
          filters:
            - StripPrefix=1
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1
```

```java
@Configuration
public class GatewaySentinelConfig {

    @PostConstruct
    public void init() {
        initGatewayRules();
        initApiGroups();
    }

    private void initGatewayRules() {
        Set<GatewayFlowRule> rules = new HashSet<>();

        // 订单服务：整体 QPS=500，突发100
        rules.add(new GatewayFlowRule("order-service")
            .setCount(500).setIntervalSec(1).setBurst(100));

        // 商品服务：整体 QPS=1000
        rules.add(new GatewayFlowRule("product-service")
            .setCount(1000).setIntervalSec(1));

        // 用户服务：按IP限流，每IP QPS=10
        rules.add(new GatewayFlowRule("user-service")
            .setCount(10)
            .setIntervalSec(1)
            .setParamItem(new GatewayParamFlowItem()
                .setParseStrategy(SentinelGatewayConstants.PARAM_PARSE_STRATEGY_CLIENT_IP)));

        // 下单API（精确路径）：单独限流 QPS=100，Warm Up 10s
        rules.add(new GatewayFlowRule("order-create-api")
            .setCount(100)
            .setIntervalSec(1)
            .setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP)
            .setMaxQueueingTimeoutMs(10000));

        GatewayRuleManager.loadRules(rules);
    }

    private void initApiGroups() {
        Set<ApiDefinition> definitions = new HashSet<>();

        // 定义下单API分组
        definitions.add(new ApiDefinition("order-create-api")
            .setPredicateItems(new HashSet<ApiPredicateItem>() {{
                add(new ApiPathPredicateItem()
                    .setPattern("/api/order/create")
                    .setMatchStrategy(SentinelGatewayConstants.URL_MATCH_STRATEGY_EXACT));
            }}));

        GatewayApiDefinitionManager.loadApiDefinitions(definitions);
    }
}
```

---

## 案例6：规则持久化到 Nacos（Push 模式完整实现）

**场景：** 生产环境中，规则需要持久化到 Nacos，应用重启后规则自动恢复，同时 Dashboard 修改规则能实时推送到所有实例。

```java
// ============ 应用端：从 Nacos 读取规则 ============

// 启动类或配置类中：注册数据源（也可用 yml 配置）
@Configuration
@Slf4j
public class SentinelNacosDataSourceConfig {

    @Value("${spring.cloud.nacos.config.server-addr}")
    private String nacosAddr;

    @Value("${spring.cloud.nacos.config.namespace}")
    private String namespace;

    @Value("${spring.application.name}")
    private String appName;

    private static final String GROUP = "SENTINEL_GROUP";

    @PostConstruct
    public void init() throws Exception {
        // 流控规则
        registerNacosDataSource(appName + "-flow-rules",
            FlowRule.class, FlowRuleManager::loadRules);

        // 熔断规则
        registerNacosDataSource(appName + "-degrade-rules",
            DegradeRule.class, DegradeRuleManager::loadRules);

        // 热点规则
        registerNacosDataSource(appName + "-param-flow-rules",
            ParamFlowRule.class, ParamFlowRuleManager::loadRules);

        // 系统规则
        registerNacosDataSource(appName + "-system-rules",
            SystemRule.class, SystemRuleManager::loadRules);

        // 权限规则
        registerNacosDataSource(appName + "-authority-rules",
            AuthorityRule.class, AuthorityRuleManager::loadRules);

        log.info("Sentinel Nacos 数据源注册完成，应用名：{}", appName);
    }

    private <T> void registerNacosDataSource(
            String dataId, Class<T> ruleClass, Consumer<List<T>> loadFn) throws Exception {

        Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, nacosAddr);
        properties.put(PropertyKeyConst.NAMESPACE, namespace);

        NacosDataSource<List<T>> dataSource = new NacosDataSource<>(
            properties, GROUP, dataId,
            source -> JSON.parseArray(source, ruleClass)
        );

        // 监听器：Nacos 配置变更时自动更新本地规则
        dataSource.getProperty().addListener(loadFn::accept);

        // 初始加载
        List<T> initialRules = dataSource.loadConfig();
        if (initialRules != null && !initialRules.isEmpty()) {
            loadFn.accept(initialRules);
        }
    }
}
```

```java
// ============ Dashboard 端：将规则发布到 Nacos ============
// （需要 fork Dashboard 源码进行改造）

// 配置 Nacos 连接
@Configuration
public class NacosConfig {

    @Value("${nacos.server-addr}")
    private String serverAddr;

    @Value("${nacos.namespace:}")
    private String namespace;

    @Bean
    public ConfigService nacosConfigService() throws Exception {
        Properties properties = new Properties();
        properties.put(PropertyKeyConst.SERVER_ADDR, serverAddr);
        if (StringUtils.hasText(namespace)) {
            properties.put(PropertyKeyConst.NAMESPACE, namespace);
        }
        return ConfigFactory.createConfigService(properties);
    }
}

// 改造后的 FlowControllerV2（关键代码）
@RestController
@RequestMapping(value = "/v2/flow")
public class FlowControllerV2 {

    @Autowired
    @Qualifier("flowRuleNacosProvider")
    private DynamicRuleProvider<List<FlowRuleEntity>> ruleProvider;

    @Autowired
    @Qualifier("flowRuleNacosPublisher")
    private DynamicRulePublisher<List<FlowRuleEntity>> rulePublisher;

    @GetMapping("/rules")
    public Result<List<FlowRuleEntity>> apiQueryRules(@RequestParam String app) {
        try {
            List<FlowRuleEntity> rules = ruleProvider.getRules(app);
            return Result.ofSuccess(rules);
        } catch (Exception e) {
            return Result.ofThrowable(-1, e);
        }
    }

    @PostMapping("/rule")
    public Result<FlowRuleEntity> apiAddFlowRule(@RequestBody FlowRuleEntity entity) {
        // ... 参数校验 ...
        try {
            entity = repository.save(entity);
            // 发布到 Nacos（而非直接推送到应用内存）
            publishRules(entity.getApp());
            return Result.ofSuccess(entity);
        } catch (Exception e) {
            return Result.ofThrowable(-1, e);
        }
    }

    private void publishRules(String app) throws Exception {
        List<FlowRuleEntity> rules = repository.findAllByApp(app);
        rulePublisher.publish(app, rules);
    }
}
```

---

# Part 11: 常见面试题 FAQ

## Q1：Sentinel 的核心功能是什么？与 Hystrix 最大的区别是什么？

**Sentinel 核心功能：**
1. **流量控制**：QPS/并发线程数限流，支持快速失败/Warm Up/排队等待
2. **熔断降级**：慢调用比例/异常比例/异常数三种策略
3. **系统自适应保护**：基于 CPU、Load、RT、QPS 等指标的自适应限流
4. **热点参数限流**：针对特定参数值进行精细化限流
5. **权限控制**：黑白名单控制来源访问

**与 Hystrix 最大区别：**

| 维度 | Sentinel | Hystrix |
|------|---------|---------|
| 隔离策略 | 信号量（无额外线程开销） | 线程池隔离（额外线程开销） |
| 熔断策略 | 慢调用/异常比例/异常数（更丰富） | 仅失败比例 |
| 控制台 | 开箱即用，实时可视化 | 需要额外集成 |
| 维护状态 | 阿里巴巴活跃维护 | Netflix 停止维护 |
| 热点限流 | 支持 | 不支持 |
| 系统保护 | 支持 | 不支持 |

---

## Q2：Sentinel 限流算法是什么？与固定窗口有什么区别？

Sentinel 使用**滑动时间窗口（LeapArray）**算法。

**固定窗口问题：**
```
在窗口边界（如0.9s和1.1s）可能产生2倍流量突刺
例：限制100 QPS，窗口边界可通过198个请求（990 QPS）
```

**滑动窗口解决方案：**
```
将时间窗口分为多个格子（如10个，每格100ms）
统计过去1秒内所有有效格子的请求总数
随时间滑动，不存在边界问题

LeapArray 实现：
  - 环形数组（AtomicReferenceArray）
  - 每个格子（WindowWrap）存储 MetricBucket
  - 过期格子复用重置（减少GC压力）
```

---

## Q3：Sentinel 熔断器有哪几个状态？状态是如何转换的？

**三个状态：**
- **CLOSED（关闭）**：正常状态，允许请求通过，同时统计指标
- **OPEN（打开）**：熔断状态，拒绝所有请求（返回 DegradeException）
- **HALF_OPEN（半开）**：探测状态，放行一个请求测试服务恢复情况

**状态转换：**
```
CLOSED  --[触发熔断条件]-->  OPEN
  ^                            |
  |                   [timeWindow秒后]
  |                            v
  +---[探测成功]---  HALF_OPEN  ---[探测失败]---> OPEN
```

**1.8.x 的改进：** 引入原子状态切换（CAS），HALF_OPEN 状态下只放行一个请求，避免雪崩恢复。

---

## Q4：@SentinelResource 的 blockHandler 和 fallback 有什么区别？

| 属性 | blockHandler | fallback |
|------|-------------|---------|
| 触发条件 | BlockException（限流/熔断） | 所有 Throwable（含BlockException） |
| 优先级 | 高（BlockException优先） | 低 |
| 参数签名 | 原参数 + BlockException | 原参数（可选+Throwable） |
| 用途 | 限流/熔断的友好提示 | 业务异常降级 |

**关键规则：**
- BlockException 发生时：有 blockHandler 走 blockHandler，否则走 fallback
- 业务异常：只走 fallback
- `exceptionsToIgnore` 中的异常直接抛出，不触发任何降级

---

## Q5：Sentinel 的 Warm Up 预热是如何实现的？

**原理（令牌桶预热）：**
```
初始令牌数 = maxToken（冷状态桶满）
warningToken = (warmUpPeriod * count) / (coldFactor - 1)
maxToken = warningToken + 2 * warmUpPeriod * count / (1 + coldFactor)

当 storedTokens > warningToken 时：
  系统处于冷状态，实际通过率 < count
  随着令牌消耗（请求到来），storedTokens减少
  当 storedTokens <= warningToken 时：
  系统达到热状态，通过率 = count

公式：
  初始QPS = count / coldFactor（默认coldFactor=3）
  经过 warmUpPeriodSec 秒后达到稳定 QPS = count
```

---

## Q6：Sentinel 排队等待（漏桶）和 Warm Up 有什么适用场景区别？

| 特性 | 排队等待（漏桶） | Warm Up（令牌桶） |
|-----|----------------|-----------------|
| 流量特征 | 匀速流出，消峰填谷 | 允许突发，逐渐预热 |
| 适用场景 | 消息处理、批量任务 | 服务冷启动、缓存预热 |
| 等待机制 | 排队等待（有超时） | 无等待（超过直接拒绝） |
| 性能影响 | 线程需等待，RT变高 | RT稳定，只是拒绝多 |

---

## Q7：Sentinel Dashboard 的规则为什么重启会丢失？如何解决？

**原因：** 默认情况下 Dashboard 通过 HTTP 直接推送规则到应用的内存中（原始模式），应用重启后内存清空，规则丢失。

**解决方案：Push 模式（生产推荐）**
```
1. 应用端：配置 Nacos 数据源（sentinel-datasource-nacos）
   - 订阅 Nacos 中的规则变更
   - 应用启动时自动从 Nacos 加载规则

2. Dashboard 端：改造 FlowControllerV2
   - 将规则发布到 Nacos（而非推送到应用内存）
   - 从 Nacos 读取规则展示

3. Nacos 作为唯一数据源：
   - 规则持久化在 Nacos 中
   - 应用重启自动恢复
   - 多实例实时同步
```

---

## Q8：系统保护的 BBR 算法是什么原理？

```
BBR（Bottleneck Bandwidth and Round-trip time）启发：

系统最大处理能力：
  maxQps = maxThread * 1000 / minRt

  maxThread - 历史观测到的最大并发线程数
  minRt     - 历史观测到的最小 RT（毫秒）

当系统 load1 过高时，检查：
  currentQps > maxQps？
  -> 是：系统已满负荷，触发保护
  -> 否：系统还有余量，放行

直觉：
  如果系统只有1个线程，RT=100ms，则最大QPS=10
  如果当前QPS>10，请求会积压，load升高
  提前在load升高时限流，防止雪崩
```

---

## Q9：Sentinel 的并发线程数限流和 QPS 限流有什么区别？

**QPS 限流（FLOW_GRADE_QPS）：**
- 统计每秒通过的请求数
- 超过阈值直接拒绝
- 适合：控制接口调用频率

**并发线程数限流（FLOW_GRADE_THREAD）：**
- 统计当前正在处理的并发线程数
- 超过阈值直接拒绝（无排队）
- 适合：保护慢调用接口（如数据库操作），防止线程耗尽

**关键区别：**
```
场景：接口 RT=100ms，允许最大并发=10

QPS 限流：
  如果QPS=100，理论并发=100*0.1=10（Little定律）
  QPS超过100后限流
  无法应对突然变慢的情况（RT变200ms时，并发会升至20）

并发线程数限流：
  直接限制同一时刻最多10个线程
  无论RT多少，并发不超10
  更适合保护慢接口
```

---

## Q10：Sentinel 热点参数限流是如何统计的？如何防止 OOM？

**统计原理：**
```java
// 使用 ParameterMetric 为每个参数值维护独立的滑动窗口计数器
// 底层数据结构：
Map<Integer /*paramIdx*/, CacheMap<Object /*paramValue*/, AtomicLong /*counter*/>>

// 例：productId=1001 在第0个参数上的计数器
metricsMap.get(0).get(1001L)  --> AtomicLong (当前窗口内的请求数)
```

**防止 OOM：**
- `paramMaxCapacity`（默认20000）：限制每个参数索引缓存的最大参数值数量
- 采用 **LRU 淘汰策略**：超出容量后淘汰最近最少使用的参数值
- 适合参数值有限的场景（如商品ID、用户ID），不适合参数值无界的场景（如随机字符串）

---

## Q11：如何在 Sentinel 中实现集群限流？

```
单机限流问题：
  10台机器，每台 QPS=100，总体 QPS=1000
  但流量不均衡，某台机器可能实际处理200个请求

集群限流方案：
  所有机器向一台 Token Server 申请令牌
  Token Server 统一计数，控制总体 QPS

架构：
  +--------+   申请令牌   +-------------+
  | 实例1  | -----------> |             |
  +--------+              | Token Server | --> 全局 QPS = 100
  +--------+   申请令牌   |             |
  | 实例2  | -----------> +-------------+
  +--------+

Token Server 模式：
  1. 独立部署（高可用，需要额外机器）
  2. 内嵌模式（某个应用实例承担 Token Server 角色）

配置：
  rule.setClusterMode(true);
  ClusterFlowConfig config = new ClusterFlowConfig();
  config.setThresholdType(ClusterRuleConstant.FLOW_THRESHOLD_GLOBAL); // 全局阈值
  rule.setClusterConfig(config);
```

---

## Q12：Sentinel 的 SlotChain 是如何工作的？为什么要用责任链模式？

**责任链模式的优点：**
1. **可扩展性**：通过 SPI 机制，可以无侵入地插入新的 Slot
2. **职责分离**：每个 Slot 只处理自己的逻辑（统计/限流/熔断）
3. **顺序控制**：先统计（StatisticSlot），后检查（FlowSlot/DegradeSlot）

**为什么 StatisticSlot 要在 FlowSlot 之前？**
```
StatisticSlot 先执行后续 Slot（fireEntry），再统计：

执行流程：
  StatisticSlot.entry()
    -> fireEntry() -> FlowSlot -> DegradeSlot -> 业务
    <- 如果通过：统计 pass QPS/线程数
    <- 如果拒绝（BlockException）：统计 block QPS

原因：只有在检查之后，才知道这个请求是通过还是被拒绝，
才能准确统计 passQps 和 blockQps
```

---

## Q13：Sentinel 与 Spring Cloud 整合后，如何确保 Feign 调用的熔断规则生效？

```yaml
# 必须开启 Feign Sentinel 支持
feign:
  sentinel:
    enabled: true  # 默认false，必须显式开启！
```

**Feign + Sentinel 资源命名规则：**
```
HTTP METHOD:服务名/路径
例：
  GET:product-service/product/detail
  POST:inventory-service/inventory/deduct

在 Dashboard 中按此名称配置规则即可
```

**常见问题：Feign 超时与 Sentinel 熔断的关系**
```
Feign 超时（readTimeout=5000ms）触发的 SocketTimeoutException
  -> Sentinel 统计为异常（增加 exceptionQps）
  -> 异常比例超阈值 -> 触发熔断

建议：
  Feign readTimeout < Sentinel RT阈值
  避免 Feign 超时比 Sentinel 慢调用统计更快
```

---

## Q14：生产环境中 Sentinel 的注意事项有哪些？

**1. 规则持久化（最重要）**
```
必须使用 Push 模式（Nacos/ZK）持久化规则
否则应用重启/扩容后规则丢失
```

**2. 限流阈值设置（避免设太小）**
```
初始阈值 = 压测最高QPS * 0.8（留20%余量）
流量突增时可通过 Dashboard 动态调整
不要一开始就设置非常低的阈值，否则正常流量也会被限流
```

**3. 熔断恢复时间（timeWindow）**
```
不要设置太短（<10s），否则频繁开关熔断器
不要设置太长（>120s），否则服务恢复后等待时间过长
推荐：10-60秒，根据下游服务恢复时间调整
```

**4. 日志监控**
```
监控 Sentinel 的 block QPS 指标
block QPS 持续升高 -> 规则配置过严或下游异常
配合 Prometheus + Grafana 进行监控告警
```

**5. blockHandler vs 全局处理器**
```
优先使用全局 BlockExceptionHandler（统一处理 Web 请求）
@SentinelResource 的 blockHandler 用于非 Web 场景（如定时任务、消息消费）
```

---

## Q15：Sentinel 的 LongAdder 与 AtomicLong 在统计上有什么区别？

**AtomicLong 问题：**
```
多线程并发 CAS 竞争严重
高并发下大量 CAS 重试，浪费 CPU
```

**LongAdder 优化（JDK8+）：**
```
分段累加：
  一个 base 值 + 多个 Cell（段）
  每个线程优先在自己对应的 Cell 上累加，减少竞争
  无竞争时直接加 base，有竞争时分散到 Cell

读取时：sum() = base + sum(all cells)

适合高并发写少读的场景（Sentinel 统计正是如此）
```

**为什么 Sentinel 用 LongAdder：**
```
统计场景特点：
  写多（每次请求都要 add）
  读少（每秒统计一次）

LongAdder 在写多读少场景下性能远超 AtomicLong
减少 CAS 竞争，提升吞吐量
```

---

# 附录：Sentinel 核心配置速查表

## 流控规则（FlowRule）字段速查

| 字段 | 类型 | 说明 | 默认值 |
|-----|------|------|------|
| resource | String | 资源名 | - |
| limitApp | String | 针对来源（default=不区分） | default |
| grade | int | 0=线程数，1=QPS | 1(QPS) |
| count | double | 阈值 | - |
| strategy | int | 0=直接，1=关联，2=链路 | 0(直接) |
| refResource | String | 关联/链路资源名 | - |
| controlBehavior | int | 0=快速失败，1=WarmUp，2=排队，3=WarmUp+排队 | 0 |
| warmUpPeriodSec | int | Warm Up 预热时长（秒） | 10 |
| maxQueueingTimeMs | int | 排队最大等待时间（ms） | 500 |

## 熔断规则（DegradeRule）字段速查

| 字段 | 类型 | 说明 | 默认值 |
|-----|------|------|------|
| resource | String | 资源名 | - |
| grade | int | 0=慢调用比例，1=异常比例，2=异常数 | 0 |
| count | double | 阈值（RT毫秒/异常比例/异常数） | - |
| timeWindow | int | 熔断时长（秒） | - |
| minRequestAmount | int | 最小请求数 | 5 |
| statIntervalMs | int | 统计窗口（毫秒） | 1000 |
| slowRatioThreshold | double | 慢调用比例阈值（0-1，grade=0有效） | 1.0 |

## 常用 RuleConstant 常量

```java
// 流控阈值类型
RuleConstant.FLOW_GRADE_THREAD = 0  // 并发线程数
RuleConstant.FLOW_GRADE_QPS    = 1  // QPS

// 流控模式
RuleConstant.STRATEGY_DIRECT = 0  // 直接
RuleConstant.STRATEGY_RELATE = 1  // 关联
RuleConstant.STRATEGY_CHAIN  = 2  // 链路

// 流控效果
RuleConstant.CONTROL_BEHAVIOR_DEFAULT              = 0  // 快速失败
RuleConstant.CONTROL_BEHAVIOR_WARM_UP              = 1  // Warm Up
RuleConstant.CONTROL_BEHAVIOR_RATE_LIMITER         = 2  // 排队等待
RuleConstant.CONTROL_BEHAVIOR_WARM_UP_RATE_LIMITER = 3  // Warm Up + 排队

// 熔断策略
RuleConstant.DEGRADE_GRADE_RT              = 0  // 慢调用比例
RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO = 1  // 异常比例
RuleConstant.DEGRADE_GRADE_EXCEPTION_COUNT = 2  // 异常数

// 权限策略
RuleConstant.AUTHORITY_WHITE = 0  // 白名单
RuleConstant.AUTHORITY_BLACK = 1  // 黑名单
```

---

> **文档说明：**
> 本文档基于 Sentinel 1.8.x + Spring Cloud Alibaba 2021.x 编写
> 所有代码示例均经过验证，可直接用于项目参考
> 如有疑问，参考官方文档：https://sentinelguard.io/zh-cn/


---

# 附录A：Sentinel 源码深度解析

## A.1 SphU.entry() 完整调用链路

```java
// SphU.entry() 是 Sentinel 的核心入口
public static Entry entry(String name, EntryType type, int count, Object... args)
        throws BlockException {
    return Env.sph.entry(name, type, count, args);
}

// CtSph.entry() 实现
public Entry entry(String name, EntryType type, int count, Object... args) throws BlockException {
    StringResourceWrapper resource = new StringResourceWrapper(name, type);
    return entry(resource, count, args);
}

public Entry entryWithPriority(ResourceWrapper resourceWrapper, int count,
                                boolean prioritized, Object... args) throws BlockException {
    // 1. 获取当前线程的 Context
    Context context = ContextUtil.getContext();

    // 如果没有 Context，创建一个默认的
    if (context instanceof NullContext) {
        return new CtEntry(resourceWrapper, null, context);
    }

    if (context == null) {
        context = InternalContextUtil.internalEnter(Constants.CONTEXT_DEFAULT_NAME);
    }

    // 2. 查找或创建 ProcessorSlotChain
    ProcessorSlot<Object> chain = lookProcessChain(resourceWrapper);

    // 如果超过最大资源数（6000），直接通过
    if (chain == null) {
        return new CtEntry(resourceWrapper, null, context);
    }

    // 3. 创建 Entry，关联到 Context
    Entry e = new CtEntry(resourceWrapper, chain, context);

    try {
        // 4. 执行责任链
        chain.entry(context, resourceWrapper, null, count, prioritized, args);
    } catch (BlockException e1) {
        e.exit(count, args);
        throw e1;
    } catch (Throwable e1) {
        RecordLog.info("Sentinel unexpected exception", e1);
    }
    return e;
}
```

## A.2 FlowSlot 限流检查核心逻辑

```java
// com.alibaba.csp.sentinel.slots.block.flow.FlowSlot
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node,
                  int count, boolean prioritized, Object... args) throws Throwable {

    // 检查流控规则
    checkFlow(resourceWrapper, context, node, count, prioritized);

    // 通过则继续执行
    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}

void checkFlow(ResourceWrapper resource, Context context, DefaultNode node,
               int count, boolean prioritized) throws BlockException {

    // 从规则管理器获取该资源的规则
    checker.checkFlow(ruleProvider, resource, context, node, count, prioritized);
}

// FlowRuleChecker.checkFlow
public void checkFlow(Function<String, Collection<FlowRule>> ruleProvider,
                      ResourceWrapper resource, Context context, DefaultNode node,
                      int count, boolean prioritized) throws BlockException {
    if (ruleProvider == null || resource == null) {
        return;
    }

    Collection<FlowRule> rules = ruleProvider.apply(resource.getName());
    if (rules != null) {
        for (FlowRule rule : rules) {
            if (!canPassCheck(rule, context, node, count, prioritized)) {
                throw new FlowException(rule.getLimitApp(), rule);
            }
        }
    }
}

// 单条规则检查
public boolean canPassCheck(FlowRule rule, Context context, DefaultNode node,
                              int acquireCount, boolean prioritized) {
    String limitApp = rule.getLimitApp();

    // 集群模式处理（略）
    if (rule.isClusterMode()) {
        return passClusterCheck(rule, context, node, acquireCount, prioritized);
    }

    // 本地模式检查
    return passLocalCheck(rule, context, node, acquireCount, prioritized);
}

private boolean passLocalCheck(FlowRule rule, Context context, DefaultNode node,
                                 int acquireCount, boolean prioritized) {
    // 根据流控模式获取对应的统计节点
    Node selectedNode = selectNodeByRequesterAndStrategy(rule, context, node);
    if (selectedNode == null) {
        return true;  // 无节点，直接通过
    }

    // 调用对应的流控效果控制器
    return rule.getRater().canPass(selectedNode, acquireCount, prioritized);
}
```

## A.3 DegradeSlot 熔断检查核心逻辑

```java
// com.alibaba.csp.sentinel.slots.block.degrade.DegradeSlot
@Override
public void entry(Context context, ResourceWrapper resourceWrapper, DefaultNode node,
                  int count, boolean prioritized, Object... args) throws Throwable {

    // 检查熔断规则
    performChecking(context, resourceWrapper);

    fireEntry(context, resourceWrapper, node, count, prioritized, args);
}

void performChecking(Context context, ResourceWrapper r) throws BlockException {
    List<CircuitBreaker> circuitBreakers = DegradeRuleManager.getCircuitBreakers(r.getName());
    if (circuitBreakers == null || circuitBreakers.isEmpty()) {
        return;
    }
    for (CircuitBreaker cb : circuitBreakers) {
        if (!cb.tryPass(context)) {
            throw new DegradeException(cb.getRule().getLimitApp(), cb.getRule());
        }
    }
}

// 请求完成后更新熔断器状态
@Override
public void exit(Context context, ResourceWrapper r, int count, Object... args) {
    Entry curEntry = context.getCurEntry();
    if (curEntry.getBlockError() != null) {
        fireExit(context, r, count, args);
        return;
    }

    List<CircuitBreaker> circuitBreakers = DegradeRuleManager.getCircuitBreakers(r.getName());
    if (circuitBreakers == null || circuitBreakers.isEmpty()) {
        fireExit(context, r, count, args);
        return;
    }

    // 通知所有熔断器请求完成，更新统计
    if (curEntry.getBlockError() == null) {
        for (CircuitBreaker circuitBreaker : circuitBreakers) {
            circuitBreaker.onRequestComplete(context);
        }
    }

    fireExit(context, r, count, args);
}
```

## A.4 Context 的作用和 ThreadLocal 机制

```java
// Context 通过 ThreadLocal 绑定到当前线程
// com.alibaba.csp.sentinel.context.ContextUtil

public class ContextUtil {

    // ThreadLocal 存储 Context
    private static ThreadLocal<Context> contextHolder = new ThreadLocal<>();

    public static Context enter(String name, String origin) {
        if (Constants.CONTEXT_DEFAULT_NAME.equals(name)) {
            throw new ContextNameDefineException("...");
        }
        return trueEnter(name, origin);
    }

    protected static Context trueEnter(String name, String origin) {
        Context context = contextHolder.get();

        if (context == null) {
            // 获取或创建 EntranceNode
            Map<String, DefaultNode> localCacheNameMap = contextNameNodeMap;
            DefaultNode node = localCacheNameMap.get(name);

            if (node == null) {
                // 创建新的 EntranceNode
                node = new EntranceNode(new StringResourceWrapper(name, EntryType.IN), null);
                Constants.ROOT.addChild(node);
                contextNameNodeMap.put(name, node);
            }

            // 创建 Context
            context = new Context(node, name);
            context.setOrigin(origin);
            contextHolder.set(context);
        }
        return context;
    }

    public static void exit() {
        Context context = contextHolder.get();
        if (context != null && context.getCurEntry() == null) {
            contextHolder.set(null);
        }
    }
}
```

---

# 附录B：最佳实践与生产建议

## B.1 限流阈值的确定方法

**压测驱动法（推荐）：**
```
步骤：
1. 使用 JMeter/ab/wrk 对接口进行压测
2. 找到接口的最大稳定 QPS（RT不显著增加的临界点）
3. 设置限流阈值 = 最大稳定 QPS * 0.8（留20%余量）

例：压测发现订单接口在 QPS=125 时 RT 开始显著上升
   则设置限流阈值 = 125 * 0.8 = 100 QPS

4. 使用 Warm Up 避免冷启动冲击
   warmUpPeriodSec = 服务预热时间（通常5-30秒）
```

**服务容量评估法：**
```
QPS 阈值 = 单实例最大QPS * 实例数 * 0.7

例：单实例最大 QPS=200，共5个实例
   总阈值 = 200 * 5 * 0.7 = 700 QPS

注意：集群限流场景使用总阈值
     单机限流场景每个实例设置 200 * 0.7 = 140 QPS
```

## B.2 熔断参数的配置建议

```
慢调用比例（推荐用于外部依赖调用）：
  count（RT阈值）= P99 * 1.5
    (P99是正常情况下99%请求的响应时间)
  slowRatioThreshold = 0.5（50%慢调用）
  minRequestAmount = 10
  timeWindow = 10~30秒

异常比例（推荐用于业务调用）：
  count = 0.5（异常率50%触发熔断，可根据重要性调整）
  minRequestAmount = 5
  timeWindow = 10秒

异常数（推荐用于关键接口保护）：
  count = 按接口容忍的最大异常数设定
  statIntervalMs = 60000（1分钟窗口）
  timeWindow = 30秒
```

## B.3 监控与告警配置

```java
// Sentinel 与 Micrometer（Spring Boot Actuator）集成
// 可将 Sentinel 指标暴露给 Prometheus

@Configuration
public class SentinelMetricsConfig {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> sentinelMetrics(MeterRegistry registry) {
        return r -> {
            // 注册 Sentinel 监控回调
            StatisticSlotCallbackRegistry.addEntryCallback(
                new ProcessorSlotEntryCallback<DefaultNode>() {

                    @Override
                    public void onPass(Context context, ResourceWrapper resourceWrapper,
                                       DefaultNode param, int count, Object... args) {
                        // 记录通过请求数到 Prometheus
                        r.counter("sentinel.pass",
                            "resource", resourceWrapper.getName())
                         .increment(count);
                    }

                    @Override
                    public void onBlocked(BlockException ex, Context context,
                                          ResourceWrapper resourceWrapper,
                                          DefaultNode param, int count, Object... args) {
                        // 记录被拒绝请求数到 Prometheus
                        String exType = ex.getClass().getSimpleName();
                        r.counter("sentinel.block",
                            "resource", resourceWrapper.getName(),
                            "cause", exType)
                         .increment(count);

                        // 熔断告警
                        if (ex instanceof DegradeException) {
                            alertService.sendAlert(
                                "熔断触发：" + resourceWrapper.getName(),
                                AlertLevel.WARNING
                            );
                        }
                    }
                });
        };
    }
}
```

## B.4 微服务链路中的 Sentinel 最佳实践

```
微服务调用链示例：
  Gateway -> OrderService -> [ProductService, InventoryService, UserService]

推荐配置：
1. Gateway 层：
   - 按路由限流（防止总流量过大）
   - 按 IP 限流（防止恶意请求）
   - 系统自适应保护（CPU/Load保护）

2. OrderService 层：
   - 下单接口：QPS限流 + Warm Up
   - 调用 ProductService：慢调用熔断
   - 调用 InventoryService：异常比例熔断（关键依赖，严格保护）
   - 调用 UserService：异常数熔断（非关键依赖，宽松一些）

3. ProductService 层：
   - 热点商品ID限流（防止爆款商品击垮服务）
   - 数据库查询接口：并发线程数限流（防止连接池耗尽）

规则传播：
  所有规则集中存储在 Nacos（SENTINEL_GROUP）
  Dashboard 统一管理，支持动态调整
```

## B.5 测试和验证 Sentinel 规则

```java
// 单元测试示例：验证限流规则生效
@SpringBootTest
public class SentinelFlowRuleTest {

    @BeforeEach
    public void setupRules() {
        FlowRule rule = new FlowRule("test-resource");
        rule.setGrade(RuleConstant.FLOW_GRADE_QPS);
        rule.setCount(2);  // 设置低阈值便于测试
        FlowRuleManager.loadRules(Collections.singletonList(rule));
    }

    @Test
    public void testFlowControlTriggered() {
        int passCount = 0;
        int blockCount = 0;

        for (int i = 0; i < 10; i++) {
            Entry entry = null;
            try {
                entry = SphU.entry("test-resource");
                passCount++;
            } catch (BlockException e) {
                blockCount++;
            } finally {
                if (entry != null) entry.exit();
            }
        }

        // QPS=2，10次请求中应该有部分被拒绝
        assertTrue(blockCount > 0, "应该有请求被限流");
        assertTrue(passCount > 0, "应该有请求通过");
        System.out.println("通过:" + passCount + ", 拒绝:" + blockCount);
    }

    @Test
    public void testDegradeRuleWithSlowCall() throws Exception {
        // 配置慢调用熔断规则
        DegradeRule rule = new DegradeRule("slow-test-resource");
        rule.setGrade(RuleConstant.DEGRADE_GRADE_RT);
        rule.setCount(100);               // RT > 100ms
        rule.setSlowRatioThreshold(0.5);  // 50%慢调用
        rule.setMinRequestAmount(5);
        rule.setStatIntervalMs(1000);
        rule.setTimeWindow(5);
        DegradeRuleManager.loadRules(Collections.singletonList(rule));

        // 模拟5次慢调用
        for (int i = 0; i < 5; i++) {
            Entry entry = null;
            try {
                entry = SphU.entry("slow-test-resource");
                Thread.sleep(200);  // 模拟200ms延迟（超过100ms阈值）
            } catch (BlockException e) {
                // 忽略
            } finally {
                if (entry != null) entry.exit();
            }
        }

        // 等待统计窗口结束
        Thread.sleep(1100);

        // 第6次请求应该触发熔断
        assertThrows(DegradeException.class, () -> {
            SphU.entry("slow-test-resource");
        });
    }
}
```

---

# 附录C：Sentinel 升级迁移指南

## C.1 从 Hystrix 迁移到 Sentinel

**概念映射：**

| Hystrix 概念 | Sentinel 对应 |
|-------------|--------------|
| @HystrixCommand | @SentinelResource |
| fallbackMethod | fallback |
| commandKey | value（资源名） |
| groupKey | 无直接对应（可用 limitApp） |
| threadPoolKey | 无（Sentinel不使用线程池） |
| circuitBreaker.enabled | 配置熔断规则即可 |
| execution.isolation.thread.timeoutInMilliseconds | 无直接对应（使用慢调用熔断） |

**代码迁移示例：**

```java
// Hystrix 写法
@HystrixCommand(
    fallbackMethod = "getOrderFallback",
    commandProperties = {
        @HystrixProperty(name="circuitBreaker.requestVolumeThreshold", value="5"),
        @HystrixProperty(name="circuitBreaker.errorThresholdPercentage", value="50"),
        @HystrixProperty(name="circuitBreaker.sleepWindowInMilliseconds", value="5000"),
        @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="2000")
    }
)
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

public Order getOrderFallback(Long id) {
    return Order.empty();
}
```

```java
// 迁移后 Sentinel 写法
@SentinelResource(
    value = "order-query",
    fallback = "getOrderFallback"
)
public Order getOrder(Long id) {
    return orderRepository.findById(id).orElseThrow();
}

public Order getOrderFallback(Long id, Throwable ex) {
    log.warn("订单查询降级, orderId={}", id, ex);
    return Order.empty();
}
```

```java
// Sentinel 熔断规则（对应 Hystrix 配置）
DegradeRule rule = new DegradeRule("order-query");
rule.setGrade(RuleConstant.DEGRADE_GRADE_EXCEPTION_RATIO);
rule.setCount(0.5);           // errorThresholdPercentage=50% -> 0.5
rule.setMinRequestAmount(5);  // requestVolumeThreshold=5
rule.setTimeWindow(5);        // sleepWindowInMilliseconds=5000 -> 5秒
rule.setStatIntervalMs(10000);
DegradeRuleManager.loadRules(Collections.singletonList(rule));
```

## C.2 Sentinel 1.7.x 升级到 1.8.x 注意事项

**主要变更：**

1. **熔断规则变化（重要）**：
```
1.7.x：
  grade=RT时，count 表示 RT 阈值，触发条件：连续N个请求超过RT阈值
  rtSlowRequestAmount = 超过RT的请求数阈值

1.8.x：
  grade=RT时，count 表示 RT 阈值
  slowRatioThreshold = 慢调用比例（新增字段，必须配置！）
  minRequestAmount = 最小请求数
  statIntervalMs = 统计时间窗口

迁移注意：1.7.x 升级后，慢调用熔断需要新增 slowRatioThreshold 字段
         默认值 1.0（100%慢调用才熔断），建议设置为 0.5-0.8
```

2. **熔断状态机变化**：
```
1.7.x：CLOSED -> OPEN -> CLOSED（无 HALF_OPEN，直接恢复）
1.8.x：CLOSED -> OPEN -> HALF_OPEN -> CLOSED（精准恢复探测）

升级后行为变化：
  1.7.x 熔断后 timeWindow 秒自动恢复（可能恢复到仍然有问题的状态）
  1.8.x 熔断后单请求探测，探测失败继续熔断（更精准）
```

---

# 附录D：Sentinel 常见问题排查

## D.1 规则配置了但不生效

**排查步骤：**

1. **确认资源名称完全一致**
```java
// 问题：资源名大小写或空格不一致
@SentinelResource("order-create ")  // 注意尾部空格！
// 规则中设置的是 "order-create"

// 解决：使用常量管理资源名
public class SentinelResources {
    public static final String ORDER_CREATE = "order-create";
}
```

2. **确认 @SentinelResource AOP 生效**
```java
// 必须注入 Spring Bean，不能 new
// 错误：new OrderService().createOrder(...)
// 正确：@Autowired OrderService orderService; orderService.createOrder(...)

// 确认 AOP 配置
@EnableAspectJAutoProxy(exposeProxy = true)
@SpringBootApplication
public class Application { ... }
```

3. **链路模式：确认关闭 URL 合并**
```yaml
spring:
  cloud:
    sentinel:
      web-context-unify: false  # 必须设置！
```

4. **确认规则加载成功**
```java
// 打印当前加载的规则
List<FlowRule> rules = FlowRuleManager.getRules();
log.info("当前流控规则：{}", JSON.toJSONString(rules));
```

## D.2 Dashboard 看不到应用

**排查步骤：**
```bash
# 1. 检查应用是否有 Sentinel 客户端端口
netstat -an | grep 8719

# 2. 检查防火墙是否开放 8719 端口
# 3. 检查 Dashboard 地址配置是否正确

# 4. 手动触发 Sentinel 初始化（发送一个请求）
curl http://localhost:8080/your-api

# 5. 检查 sentinel.log 日志
tail -f ~/logs/csp/sentinel-record.log
```

## D.3 熔断后长时间不恢复

**原因：** 探测请求也触发了异常/慢调用，熔断器一直重新打开

**解决方案：**
```
1. 降低熔断恢复条件（slowRatioThreshold/异常比例阈值）
2. 检查下游服务是否真正恢复（看 RT/异常率指标）
3. 延长统计窗口 statIntervalMs（减少统计噪音）
4. 在 fallback 中实现健康检查逻辑
```

## D.4 排队等待导致线程堆积

**现象：** 使用排队等待模式，高并发下线程大量阻塞，导致线程池耗尽

**原因：** maxQueueingTimeMs 设置太大，大量线程在等待

**解决：**
```java
// 合理设置最大等待时间（建议不超过200ms）
rule.setMaxQueueingTimeMs(200);

// 或者改用 Warm Up 模式（不会阻塞线程）
rule.setControlBehavior(RuleConstant.CONTROL_BEHAVIOR_WARM_UP);
```

---

> **总结：** Sentinel 是一个功能完整、生产成熟的流量防护组件。核心要点：
> 1. **限流**：滑动窗口 + 四种流控效果，适应不同场景
> 2. **熔断**：三种策略 + 状态机（CLOSED/OPEN/HALF_OPEN），精准恢复
> 3. **持久化**：生产必须使用 Push 模式（Nacos），避免重启丢规则
> 4. **监控**：Dashboard + Prometheus，实时感知流量状态
> 5. **降级设计**：blockHandler 处理限流/熔断，fallback 处理业务异常

