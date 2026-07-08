# SkyWalking 详解：从原理到实战的完整指南

> 作者：技术文档中心  
> 版本：v1.0  
> 更新日期：2026-07-08  
> 适用版本：Apache SkyWalking 9.x / OAP 9.x

---

## 目录

- [Part 1: 可观测性概述](#part-1)
- [Part 2: SkyWalking 整体架构](#part-2)
- [Part 3: Java Agent 探针原理](#part-3)
- [Part 4: 部署与配置](#part-4)
- [Part 5: Trace 链路追踪](#part-5)
- [Part 6: Metrics 指标监控与 OAL](#part-6)
- [Part 7: 日志关联](#part-7)
- [Part 8: 告警配置](#part-8)
- [Part 9: Service Mesh 集成](#part-9)
- [Part 10: 完整实战案例](#part-10)
- [Part 11: 性能调优与运维](#part-11)
- [Part 12: 常见面试题 FAQ](#part-12)

---

## Part 1: 可观测性概述 {#part-1}

### 1.1 什么是可观测性（Observability）

可观测性（Observability）是指通过系统对外暴露的输出数据，推断系统内部状态的能力。这一概念源自控制理论，由 Rudolf Kálmán 于 1960 年提出，后被引入分布式系统领域。

在现代微服务架构下，一个业务请求可能经过数十个服务，每个服务又可能依赖数据库、缓存、消息队列等中间件。当系统出现异常时，如何快速定位根因、如何评估系统健康度，成为工程师面临的核心挑战。

可观测性与传统监控的本质区别在于：
- **传统监控**：预先定义需要检测的异常（已知的未知）
- **可观测性**：能够回答任意关于系统状态的问题，包括未曾预料的异常（未知的未知）

### 1.2 可观测性的三大支柱

```
+--------------------------------------------------+
|              可观测性三大支柱                      |
|                                                  |
|   +----------+  +----------+  +----------+      |
|   |  Metrics  |  | Tracing  |  | Logging  |      |
|   |  指标监控  |  | 链路追踪  |  | 日志记录  |      |
|   +----------+  +----------+  +----------+      |
|        |              |              |           |
|    聚合型数据       分布式追踪       结构化日志     |
|    (What/How)    (Where/Why)    (Detail/Context) |
+--------------------------------------------------+
```

#### 1.2.1 Metrics（指标）

Metrics 是对系统状态的**定量测量**，以时间序列数据的形式存储。其特点是：
- **高效聚合**：可以按维度（label）聚合，如 `http_requests_total{service="order", status="200"}`
- **存储高效**：相比日志，指标数据非常紧凑
- **实时告警**：适合设置阈值告警，如 CPU > 80%、错误率 > 1%

常见的 Metrics 类型：

| 类型 | 说明 | 示例 |
|------|------|------|
| Counter | 单调递增计数器 | 请求总数、错误总数 |
| Gauge | 可增可减的当前值 | 当前在线用户数、内存使用量 |
| Histogram | 值的分布统计 | 请求延迟分布（P50/P90/P99） |
| Summary | 滑动窗口分位数 | 过去 5 分钟的 P99 延迟 |

Metrics 的典型应用场景：
1. **SLI/SLO 监控**：服务可用性 99.9%、P99 延迟 < 500ms
2. **容量规划**：CPU/Memory 趋势分析
3. **业务监控**：每秒订单量、支付成功率

#### 1.2.2 Tracing（链路追踪）

分布式追踪用于记录一次请求在多个服务间的**完整调用路径**，帮助工程师理解请求流转过程和性能瓶颈。

核心概念：

```
TraceID: abc-123
|
+-- Span(A): 前端服务 [0ms -> 120ms]
    |
    +-- Span(B): 订单服务 [10ms -> 100ms]
        |
        +-- Span(C): 数据库查询 [20ms -> 50ms]
        |
        +-- Span(D): 库存服务 [55ms -> 90ms]
            |
            +-- Span(E): Redis查询 [60ms -> 70ms]
```

- **Trace**：一次完整的请求调用，由全局唯一的 TraceID 标识
- **Span**：Trace 中的单个操作单元，包含操作名、开始时间、持续时长、状态码
- **Context Propagation**：TraceID/SpanID 在服务间通过 HTTP Header 或消息队列 Header 传递

分布式追踪解决的问题：
1. **性能分析**：定位请求链路中的慢节点
2. **依赖分析**：自动生成服务拓扑图
3. **错误追踪**：快速定位报错的根因服务
4. **容量评估**：统计各服务的调用频率

#### 1.2.3 Logging（日志）

日志是系统运行时产生的**事件记录**，包含最丰富的上下文信息。

日志级别（从低到高）：
```
TRACE -> DEBUG -> INFO -> WARN -> ERROR -> FATAL
```

结构化日志 vs 非结构化日志：

```
# 非结构化（传统）
2026-07-08 10:00:01 ERROR OrderService: Order 12345 failed: timeout

# 结构化（推荐）
{
  "timestamp": "2026-07-08T10:00:01.000Z",
  "level": "ERROR",
  "service": "order-service",
  "traceId": "abc-123",
  "spanId": "def-456",
  "message": "Order processing failed",
  "orderId": "12345",
  "error": "ReadTimeoutException",
  "duration_ms": 5001
}
```

结构化日志的优势：
- 机器可读，方便自动化分析
- 可与 TraceID 关联，实现日志-追踪联动
- 支持按字段过滤和聚合

### 1.3 三大支柱的关联与协同

```
+-----------------------------------------------------------+
|                    一次异常排查流程                          |
+-----------------------------------------------------------+
|                                                           |
|  1. Metrics 触发告警                                       |
|     "order-service 错误率 > 5%"                            |
|              |                                            |
|              v                                            |
|  2. Tracing 定位根因                                       |
|     打开慢 Trace，发现 inventory-service 耗时 > 3s          |
|              |                                            |
|              v                                            |
|  3. Logging 确认细节                                       |
|     通过 TraceID 找到对应日志                               |
|     "Connection pool exhausted: max=10, active=10"        |
|                                                           |
+-----------------------------------------------------------+
```

这三者互为补充：
- Metrics 提供**鸟瞰视角**，回答"系统是否健康"
- Tracing 提供**链路视角**，回答"哪里出了问题"
- Logging 提供**细节视角**，回答"为什么出了问题"

### 1.4 主流可观测性工具对比

| 工具 | 类型 | 优势 | 劣势 |
|------|------|------|------|
| Prometheus + Grafana | Metrics | 生态成熟、查询语言强大 | 不含 Tracing/Logging |
| Jaeger | Tracing | CNCF 项目、部署简单 | Metrics 能力弱 |
| Zipkin | Tracing | 历史悠久、社区活跃 | 功能相对基础 |
| ELK Stack | Logging | 日志分析能力强 | 资源消耗大 |
| **Apache SkyWalking** | **一体化** | **Metrics+Tracing+Logging 三合一** | 学习曲线稍陡 |
| Datadog | 一体化（商业） | 功能完整、易用 | 价格昂贵 |
| New Relic | 一体化（商业） | APM 功能强大 | 价格昂贵 |

SkyWalking 的核心竞争优势：
1. **无侵入式 Java Agent**：业务代码零修改
2. **三合一**：同一平台覆盖 Metrics、Tracing、Logging
3. **云原生**：原生支持 Kubernetes、Istio Service Mesh
4. **高性能**：OAP 服务端支持高并发数据摄入
5. **Apache 顶级项目**：社区活跃，持续迭代

---

## Part 2: SkyWalking 整体架构 {#part-2}

### 2.1 架构全景图

```
+--------------------------------------------------------------------+
|                    Apache SkyWalking 整体架构                       |
+--------------------------------------------------------------------+
|                                                                    |
|  +------------------+    +------------------+                     |
|  |   Java 微服务 A    |    |   Java 微服务 B    |                     |
|  |  +-----------+   |    |  +-----------+   |                     |
|  |  | SW Agent  |   |    |  | SW Agent  |   |                     |
|  |  +-----------+   |    |  +-----------+   |                     |
|  +--------+---------+    +--------+---------+                     |
|           |                       |                               |
|           +----------+------------+                               |
|                      | gRPC/HTTP                                  |
|                      v                                            |
|           +----------+----------+                                 |
|           |    OAP Server        |                                |
|           | (Observability       |                                |
|           |  Analysis Platform)  |                                |
|           +----+--------+--------+                                |
|                |        |                                         |
|         +------+        +------+                                  |
|         |                      |                                  |
|         v                      v                                  |
|  +------+------+    +----------+--------+                        |
|  |   存储层      |    |   告警/通知        |                        |
|  | Elasticsearch|    | (Webhook/钉钉等)  |                        |
|  | MySQL        |    +-------------------+                        |
|  | TiDB 等      |                                                  |
|  +------+------+                                                  |
|         |                                                         |
|         v                                                         |
|  +------+------+                                                  |
|  |   UI 展示    |                                                  |
|  | (SkyWalking  |                                                  |
|  |   RocketBot) |                                                  |
|  +-------------+                                                  |
|                                                                    |
+--------------------------------------------------------------------+
```

### 2.2 四大核心组件详解

#### 2.2.1 Agent（探针）

Agent 是运行在被监控应用进程内的字节码增强工具，支持多种语言：

| 语言 | Agent 类型 | 实现方式 |
|------|-----------|---------|
| Java | Java Agent (-javaagent) | ByteBuddy 字节码增强 |
| .NET | CLR Profiler | CLR Profiling API |
| Node.js | NPM 包 | 代码织入 |
| Go | SDK | 手动埋点 |
| Python | PyPI 包 | 自动/手动 |
| PHP | PHP 扩展 | C 扩展 |

Java Agent 核心功能：
1. **自动拦截**：HTTP 入口/出口、数据库调用、RPC 调用、消息队列等
2. **上下文传播**：自动在跨进程调用时注入/提取 TraceID
3. **数据上报**：将 Span 数据通过 gRPC 发送到 OAP Server
4. **插件机制**：超过 100 个官方插件，覆盖主流框架

#### 2.2.2 OAP Server（观测分析平台）

OAP（Observability Analysis Platform）是 SkyWalking 的核心服务端，负责：

```
+---------------------------------------+
|           OAP Server 内部架构          |
+---------------------------------------+
|                                       |
|  +----------+    +----------------+  |
|  |  Receiver |    |   Core Module  |  |
|  |  (数据接收) +--> | (数据处理/聚合)|  |
|  +----------+    +-------+--------+  |
|                          |           |
|           +--------------+-----+     |
|           |                    |     |
|           v                    v     |
|  +--------+------+   +---------+---+ |
|  | Storage Module|   |  Query Module| |
|  | (写入存储层)   |   | (GraphQL API)| |
|  +---------------+   +-------------+ |
|                                       |
+---------------------------------------+
```

OAP 的核心模块：
1. **Receiver**：接收来自 Agent 的 Tracing/Metrics/Logging 数据
2. **OAL (Observability Analysis Language)**：实时流式计算引擎，将原始数据聚合为 Metrics
3. **Alarm**：基于规则的告警引擎
4. **Query**：提供 GraphQL API，供 UI 查询数据

#### 2.2.3 Storage（存储层）

SkyWalking 支持多种存储后端：

| 存储 | 适用场景 | 特点 |
|------|---------|------|
| H2（内存） | 本地开发/测试 | 无需额外部署，重启数据丢失 |
| Elasticsearch | 生产推荐 | 分布式、搜索能力强 |
| OpenSearch | 生产推荐 | ES 开源替代方案 |
| MySQL | 中小规模 | 熟悉度高，但性能有限 |
| TiDB | 大规模 | 兼容 MySQL，水平扩展 |
| BanyanDB | 官方新存储 | 专为 SkyWalking 设计 |

#### 2.2.4 UI（RocketBot）

SkyWalking UI 提供以下视图：
1. **Dashboard**：服务/实例/端点的 Metrics 仪表板
2. **Topology**：服务依赖拓扑图（自动发现）
3. **Trace**：分布式追踪查询与瀑布图展示
4. **Log**：日志查询与 TraceID 关联
5. **Alarm**：告警记录
6. **Profiling**：持续性能剖析（On-demand / Continuous）

### 2.3 数据流转全链路

```
+--------------------------------------------------------------+
|                     数据流转详解                               |
+--------------------------------------------------------------+
|                                                              |
|  业务请求 --> [Java 应用 + SW Agent]                           |
|                      |                                       |
|          +-----------+-----------+                           |
|          |           |           |                           |
|    [Trace Span]  [Metrics]  [Log with TraceID]               |
|          |           |           |                           |
|          +-----------+-----------+                           |
|                      | gRPC(11800) / HTTP(12800)             |
|                      v                                       |
|              [OAP Server]                                    |
|                      |                                       |
|          +-----------+-----------+                           |
|          |           |           |                           |
|    [Trace存储]   [OAL计算]   [Log存储]                         |
|          |           |           |                           |
|          v           v           v                           |
|      [ES Index]  [ES Index]  [ES Index]                      |
|    trace-yyyyMM  metrics-xxx  log-yyyyMM                     |
|                      |                                       |
|                      v                                       |
|              [UI / GraphQL API]                              |
|                   端口 8080                                   |
|                                                              |
+--------------------------------------------------------------+
```

### 2.4 集群部署模式

在生产环境中，OAP Server 通常以集群模式部署：

```
+-------------------------------------------------------+
|                  OAP 集群部署                          |
+-------------------------------------------------------+
|                                                       |
|   Agent                                               |
|    |                                                  |
|    v                                                  |
|  [负载均衡器]                                          |
|    |          |          |                            |
|    v          v          v                            |
| [OAP-1]    [OAP-2]    [OAP-3]                        |
|    |          |          |                            |
|    +----------+----------+                            |
|               |                                       |
|         [Zookeeper / Nacos]  <-- 集群协调               |
|               |                                       |
|    +----------+----------+                            |
|    |                     |                            |
|    v                     v                            |
| [ES 集群]           [MySQL / TiDB]                    |
|                                                       |
+-------------------------------------------------------+
```

---

## Part 3: Java Agent 探针原理 {#part-3}

### 3.1 Java Agent 基础原理

Java Agent 是 JDK 1.5 引入的机制，允许在 JVM 启动时（或运行时）修改类的字节码。其核心是 `java.lang.instrument` 包。

启动方式：
```bash
java -javaagent:/path/to/skywalking-agent.jar \
     -Dskywalking.agent.service_name=my-service \
     -jar my-app.jar
```

Java Agent 入口类必须包含以下方法之一：
```java
// 静态加载（JVM 启动时）
public static void premain(String agentArgs, Instrumentation inst)

// 动态加载（运行时 attach）
public static void agentmain(String agentArgs, Instrumentation inst)
```

### 3.2 字节码增强框架：ByteBuddy

SkyWalking 使用 **ByteBuddy** 框架进行字节码增强（早期版本使用 Javassist）。

ByteBuddy 的核心 API：
```java
new AgentBuilder.Default()
    .type(ElementMatchers.nameStartsWith("com.example"))
    .transform((builder, typeDescription, classLoader, module) ->
        builder.method(ElementMatchers.named("doGet"))
               .intercept(MethodDelegation.to(MyInterceptor.class))
    )
    .installOn(instrumentation);
```

### 3.3 SkyWalking 插件加载机制

SkyWalking Agent 采用**插件化架构**，每个框架对应一个插件 JAR：

```
skywalking-agent/
+-- skywalking-agent.jar          # Agent 核心
+-- config/
|   +-- agent.config              # 配置文件
+-- plugins/                      # 激活的插件
|   +-- apm-springmvc-annotation-5.x-plugin.jar
|   +-- apm-httpclient-4.x-plugin.jar
|   +-- apm-mysql-8.x-plugin.jar
|   +-- apm-kafka-plugin.jar
|   +-- ...（100+ 插件）
+-- optional-plugins/             # 可选插件（需手动移入 plugins/）
+-- bootstrap-plugins/            # Bootstrap ClassLoader 级别插件
+-- logs/
```

插件定义规范（`skywalking-plugin.def`）：
```
# 格式：插件名=定义类全路径
springmvc-annotation-5.x=org.apache.skywalking.apm.plugin.spring.mvc.v5.define.ControllerInstrumentation
httpclient-4.x=org.apache.skywalking.apm.plugin.httpclient.v4.define.HttpClientInstrumentation
```

### 3.4 编写自定义插件

以拦截一个自定义 RPC 框架为例：

#### Step 1: 定义 Instrumentation

```java
package com.example.skywalking.plugin;

import net.bytebuddy.description.method.MethodDescription;
import net.bytebuddy.matcher.ElementMatcher;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.ConstructorInterceptPoint;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.InstanceMethodsInterceptPoint;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.ClassInstanceMethodsEnhancePluginDefine;
import org.apache.skywalking.apm.agent.core.plugin.match.ClassMatch;
import org.apache.skywalking.apm.agent.core.plugin.match.NameMatch;

import static net.bytebuddy.matcher.ElementMatchers.named;

/**
 * 拦截 MyRpcClient.invoke() 方法的插件定义
 */
public class MyRpcClientInstrumentation extends ClassInstanceMethodsEnhancePluginDefine {

    private static final String ENHANCE_CLASS = "com.example.rpc.MyRpcClient";
    private static final String INTERCEPT_CLASS = "com.example.skywalking.plugin.MyRpcClientInterceptor";

    @Override
    protected ClassMatch enhanceClass() {
        // 指定要增强的类
        return NameMatch.byName(ENHANCE_CLASS);
    }

    @Override
    public ConstructorInterceptPoint[] getConstructorsInterceptPoints() {
        return new ConstructorInterceptPoint[0]; // 不拦截构造器
    }

    @Override
    public InstanceMethodsInterceptPoint[] getInstanceMethodsInterceptPoints() {
        return new InstanceMethodsInterceptPoint[]{
            new InstanceMethodsInterceptPoint() {
                @Override
                public ElementMatcher<MethodDescription> getMethodsMatcher() {
                    // 拦截名为 invoke 的方法
                    return named("invoke");
                }

                @Override
                public String getMethodsInterceptor() {
                    return INTERCEPT_CLASS;
                }

                @Override
                public boolean isOverrideArgs() {
                    return false;
                }
            }
        };
    }
}
```

#### Step 2: 编写 Interceptor

```java
package com.example.skywalking.plugin;

import org.apache.skywalking.apm.agent.core.context.CarrierItem;
import org.apache.skywalking.apm.agent.core.context.ContextCarrier;
import org.apache.skywalking.apm.agent.core.context.ContextManager;
import org.apache.skywalking.apm.agent.core.context.tag.Tags;
import org.apache.skywalking.apm.agent.core.context.trace.AbstractSpan;
import org.apache.skywalking.apm.agent.core.context.trace.SpanLayer;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.EnhancedInstance;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.InstanceMethodsAroundInterceptor;
import org.apache.skywalking.apm.agent.core.plugin.interceptor.enhance.MethodInterceptResult;

import java.lang.reflect.Method;

/**
 * MyRpcClient.invoke() 方法拦截器
 * 负责创建 Exit Span 并传播 TraceContext
 */
public class MyRpcClientInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst,
                             Method method,
                             Object[] allArguments,
                             Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {

        // 从方法参数中获取 RPC Request 对象
        MyRpcRequest request = (MyRpcRequest) allArguments[0];
        String remotePeer = request.getHost() + ":" + request.getPort();
        String operationName = request.getServiceName() + "/" + request.getMethodName();

        // 创建 ContextCarrier 用于跨进程传播
        ContextCarrier contextCarrier = new ContextCarrier();

        // 创建 Exit Span（调用下游服务的出口）
        AbstractSpan span = ContextManager.createExitSpan(operationName, contextCarrier, remotePeer);

        // 设置 Span 属性
        span.setComponent(ComponentsDefine.MY_RPC_FRAMEWORK); // 自定义组件标识
        Tags.RPC_PARAMETERS.set(span, request.getParams().toString());
        SpanLayer.asRPCFramework(span); // 标记为 RPC 类型

        // 将 TraceContext 注入到 RPC Header
        CarrierItem item = contextCarrier.items();
        while (item.hasNext()) {
            item = item.next();
            request.addHeader(item.getHeadKey(), item.getHeadValue());
        }
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst,
                              Method method,
                              Object[] allArguments,
                              Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
        // 请求完成后停止 Span
        ContextManager.stopSpan();
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst,
                                      Method method,
                                      Object[] allArguments,
                                      Class<?>[] argumentsTypes,
                                      Throwable t) {
        // 发生异常时标记 Span 为错误
        AbstractSpan span = ContextManager.activeSpan();
        span.errorOccurred().log(t);
    }
}
```

#### Step 3: 服务端 Interceptor（提取 TraceContext）

```java
/**
 * MyRpcServer 方法拦截器（Entry Span）
 * 从 RPC Header 中提取 TraceContext，实现跨进程追踪
 */
public class MyRpcServerInterceptor implements InstanceMethodsAroundInterceptor {

    @Override
    public void beforeMethod(EnhancedInstance objInst,
                             Method method,
                             Object[] allArguments,
                             Class<?>[] argumentsTypes,
                             MethodInterceptResult result) throws Throwable {

        MyRpcContext rpcContext = (MyRpcContext) allArguments[0];
        String operationName = rpcContext.getServiceName() + "/" + rpcContext.getMethodName();

        // 创建 ContextCarrier 用于提取上游传来的 TraceContext
        ContextCarrier contextCarrier = new ContextCarrier();
        CarrierItem item = contextCarrier.items();
        while (item.hasNext()) {
            item = item.next();
            // 从 RPC Header 中读取 SkyWalking 注入的追踪信息
            item.setHeadValue(rpcContext.getHeader(item.getHeadKey()));
        }

        // 创建 Entry Span，关联上游 TraceContext
        AbstractSpan span = ContextManager.createEntrySpan(operationName, contextCarrier);
        span.setComponent(ComponentsDefine.MY_RPC_FRAMEWORK);
        SpanLayer.asRPCFramework(span);
    }

    @Override
    public Object afterMethod(EnhancedInstance objInst, Method method,
                              Object[] allArguments, Class<?>[] argumentsTypes,
                              Object ret) throws Throwable {
        ContextManager.stopSpan();
        return ret;
    }

    @Override
    public void handleMethodException(EnhancedInstance objInst, Method method,
                                      Object[] allArguments, Class<?>[] argumentsTypes,
                                      Throwable t) {
        ContextManager.activeSpan().errorOccurred().log(t);
    }
}
```

### 3.5 TraceContext 传播机制

```
+------------------------------------------------------------------+
|                   跨进程 TraceContext 传播                         |
+------------------------------------------------------------------+
|                                                                  |
|  Service A                          Service B                    |
|  +-----------------+                +-----------------+          |
|  | [Entry Span]    |                | [Entry Span]    |          |
|  |   traceId: T1   |  HTTP Request  |   traceId: T1   |          |
|  |   spanId: 0     | +-----------> |   parentSpanId:0|          |
|  |                 |  Header:       |   spanId: 1     |          |
|  | [Exit Span]     |  sw8: T1-...-B |                 |          |
|  |   traceId: T1   |                |                 |          |
|  |   spanId: 0.1   |                |                 |          |
|  +-----------------+                +-----------------+          |
|                                                                  |
|  sw8 Header 格式：                                               |
|  {sample}-{traceId}-{segmentId}-{spanId}-{service}-{instance}-   |
|  {endpoint}-{clientType}                                         |
|                                                                  |
+------------------------------------------------------------------+
```

HTTP Header 示例：
```
sw8: 1-MS4wLjA=-MS4wLjA=.1-0-order-service-192.168.1.1-11800-/api/order-1
```

各字段含义：
1. `1` - 采样标志（1=采样，0=不采样）
2. `MS4wLjA=` - TraceID（Base64 编码）
3. `MS4wLjA=.1` - 父 SegmentID
4. `0` - 父 SpanID
5. `order-service` - 父服务名
6. `192.168.1.1-11800` - 父实例
7. `/api/order` - 父端点
8. `1` - 是否跨线程（0=跨进程，1=跨线程）

---

## Part 4: 部署与配置 {#part-4}

### 4.1 单机快速启动（Docker Compose）

#### 完整的 docker-compose.yml

```yaml
version: '3.8'

services:
  # Elasticsearch 存储
  elasticsearch:
    image: elasticsearch:8.10.0
    container_name: sw-elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
      - bootstrap.memory_lock=true
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - es-data:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
    networks:
      - sw-net
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9200/_cluster/health"]
      interval: 30s
      timeout: 10s
      retries: 5

  # SkyWalking OAP Server
  oap:
    image: apache/skywalking-oap-server:9.6.0
    container_name: sw-oap
    depends_on:
      elasticsearch:
        condition: service_healthy
    environment:
      SW_STORAGE: elasticsearch
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
      SW_HEALTH_CHECKER: default
      SW_TELEMETRY: prometheus
      JAVA_OPTS: -Xms2g -Xmx2g
    ports:
      - "11800:11800"   # gRPC（Agent 上报）
      - "12800:12800"   # HTTP REST（UI 查询）
      - "1234:1234"     # Prometheus metrics
    networks:
      - sw-net
    healthcheck:
      test: ["CMD", "bash", "-c", "cat < /dev/null > /dev/tcp/localhost/11800"]
      interval: 30s
      timeout: 10s
      retries: 5

  # SkyWalking UI
  ui:
    image: apache/skywalking-ui:9.6.0
    container_name: sw-ui
    depends_on:
      oap:
        condition: service_healthy
    environment:
      SW_OAP_ADDRESS: http://oap:12800
    ports:
      - "8080:8080"
    networks:
      - sw-net

volumes:
  es-data:

networks:
  sw-net:
    driver: bridge
```

启动命令：
```bash
docker-compose up -d

# 查看状态
docker-compose ps

# 查看 OAP 日志
docker-compose logs -f oap

# 访问 UI
# http://localhost:8080
```

### 4.2 Spring Boot 集成

#### 4.2.1 方式一：Java Agent（推荐，零侵入）

下载 Agent：
```bash
wget https://archive.apache.org/dist/skywalking/java-agent/9.1.0/\
apache-skywalking-java-agent-9.1.0.tgz
tar -xzf apache-skywalking-java-agent-9.1.0.tgz
```

启动 Spring Boot 应用：
```bash
java -javaagent:/opt/skywalking-agent/skywalking-agent.jar \
     -Dskywalking.agent.service_name=order-service \
     -Dskywalking.collector.backend_service=oap-host:11800 \
     -Dskywalking.logging.level=INFO \
     -jar order-service.jar
```

Docker 镜像中使用 Agent：
```dockerfile
FROM openjdk:17-slim

# 复制 SkyWalking Agent
COPY --from=apache/skywalking-java-agent:9.1.0-java17 \
     /skywalking/agent /skywalking/agent

COPY target/order-service.jar /app/order-service.jar

ENV SW_AGENT_NAME=order-service
ENV SW_AGENT_COLLECTOR_BACKEND_SERVICES=oap:11800

ENTRYPOINT ["java", \
  "-javaagent:/skywalking/agent/skywalking-agent.jar", \
  "-jar", "/app/order-service.jar"]
```

#### 4.2.2 方式二：Spring Boot Starter（SDK 方式）

pom.xml 依赖：
```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-trace</artifactId>
    <version>9.1.0</version>
</dependency>

<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>9.1.0</version>
</dependency>
```

### 4.3 Agent 配置详解

`agent.config` 核心配置项：
```properties
# ========== Agent 基础配置 ==========
# 服务名（重要！这是在 SkyWalking UI 中显示的服务名称）
agent.service_name=${SW_AGENT_NAME:your-service-name}

# 实例名（默认为主机名@PID，可自定义）
agent.instance_name=${SW_INSTANCE_NAME:}

# OAP 服务器地址（gRPC 端口 11800）
collector.backend_service=${SW_AGENT_COLLECTOR_BACKEND_SERVICES:127.0.0.1:11800}

# ========== 采样配置 ==========
# 采样率：-1=全采样，0-9999=千分比（3000=30%）
agent.sample_n_per_3_secs=${SW_AGENT_SAMPLE:-1}

# ========== 日志配置 ==========
logging.level=${SW_LOGGING_LEVEL:INFO}
logging.file_name=${SW_LOGGING_FILE:skywalking-api.log}
logging.dir=${SW_LOGGING_DIR:logs}
logging.max_file_size=${SW_LOGGING_MAX_FILE_SIZE:314572800}

# ========== 性能配置 ==========
# 单个 Segment 最大 Span 数量（超出截断）
agent.span_limit_per_segment=${SW_AGENT_SPAN_LIMIT:300}

# 忽略的端点（正则表达式，多个用逗号分隔）
agent.ignore_suffix=${SW_AGENT_IGNORE_SUFFIX:.jpg,.jpeg,.js,.css,.png,.bmp,.gif,.ico,.mp3,.mp4,.html,.svg}

# ========== 插件配置 ==========
# Spring MVC：是否收集 HTTP 参数
plugin.springmvc.collect_http_params=${SW_SPRINGMVC_COLLECT_HTTP_PARAMS:false}
# HTTP 参数最大长度
plugin.http.http_params_length_threshold=${SW_HTTP_PARAMS_LENGTH_THRESHOLD:1024}

# Kafka 插件：是否追踪消息内容
plugin.kafka.consumer_topics=order-topic,payment-topic

# MySQL 插件：是否记录 SQL 参数（生产慎用）
plugin.mysql.trace_sql_parameters=${SW_MYSQL_TRACE_SQL_PARAMETERS:false}
# SQL 最大长度
plugin.mysql.sql_body_max_length=${SW_MYSQL_SQL_BODY_MAX_LENGTH:2048}

# ========== 指标上报配置 ==========
# JVM 指标采集间隔（秒）
jvm.buffer_size=${SW_JVM_BUFFER_SIZE:600}
```

### 4.4 Kubernetes 部署

#### 4.4.1 使用 Init Container 注入 Agent

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
      annotations:
        # 用于 SkyWalking 自动关联 Pod 信息（可选）
        skywalking.apache.org/service.name: "order-service"
    spec:
      initContainers:
        # Init Container：将 Agent 复制到共享卷
        - name: skywalking-agent-init
          image: apache/skywalking-java-agent:9.1.0-java17
          command: ['cp', '-r', '/skywalking/agent', '/agent-dir']
          volumeMounts:
            - name: skywalking-agent
              mountPath: /agent-dir

      containers:
        - name: order-service
          image: my-registry/order-service:v1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: SW_AGENT_NAME
              value: "order-service"
            - name: SW_AGENT_COLLECTOR_BACKEND_SERVICES
              value: "skywalking-oap.monitoring.svc.cluster.local:11800"
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: SW_INSTANCE_NAME
              value: "$(POD_NAME)"
            - name: JAVA_TOOL_OPTIONS
              value: "-javaagent:/skywalking/agent/skywalking-agent.jar"
          volumeMounts:
            - name: skywalking-agent
              mountPath: /skywalking/agent
              subPath: agent
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi

      volumes:
        - name: skywalking-agent
          emptyDir: {}
```

#### 4.4.2 使用 SkyWalking Operator（推荐）

安装 Operator：
```bash
kubectl apply -f https://github.com/apache/skywalking-swck/releases/latest/download/operator.yaml
```

创建 JavaAgent 自定义资源：
```yaml
apiVersion: operator.skywalking.apache.org/v1alpha1
kind: JavaAgent
metadata:
  name: order-service-agent
spec:
  serviceName: order-service
  backendService: oap.monitoring.svc:11800
  agentConfiguration:
    agent.sample_n_per_3_secs: "3000"
    plugin.springmvc.collect_http_params: "true"
```

然后只需在 Deployment 加一个 annotation：
```yaml
metadata:
  annotations:
    strategy.skywalking.apache.org/inject.Container: "order-service"
```

---

## Part 5: Trace 链路追踪 {#part-5}

### 5.1 Span 类型与生命周期

SkyWalking 定义了三种 Span 类型：

| 类型 | 说明 | 适用场景 |
|------|------|---------|
| EntrySpan | 接收请求的入口 Span | HTTP Server、MQ Consumer、RPC Server |
| ExitSpan | 发起请求的出口 Span | HTTP Client、MQ Producer、RPC Client |
| LocalSpan | 本地方法调用 Span | 本地方法、异步操作 |

Span 的核心属性：
```
Span {
  traceId: string          // 全局唯一追踪 ID
  segmentId: string        // 当前进程内的 Segment ID
  spanId: int              // Segment 内递增 ID
  parentSpanId: int        // 父 Span ID（-1 表示无父）
  startTime: long          // 开始时间戳（毫秒）
  endTime: long            // 结束时间戳（毫秒）
  operationName: string    // 操作名（如 /api/order, mysql/select）
  peer: string             // 对端地址（仅 ExitSpan）
  isError: boolean         // 是否发生错误
  layer: SpanLayer         // 层级（HTTP/DB/RPC/CACHE/MQ）
  component: int           // 组件编号（Spring MVC=14, MySQL=5）
  tags: Map<String, String> // 自定义标签
  logs: List<LogData>      // 事件日志（含异常堆栈）
}
```

### 5.2 手动埋点 API

#### 5.2.1 使用 @Trace 注解

```java
import org.apache.skywalking.apm.toolkit.trace.Trace;
import org.apache.skywalking.apm.toolkit.trace.Tag;
import org.apache.skywalking.apm.toolkit.trace.Tags;

@Service
public class OrderService {

    /**
     * @Trace 注解：自动为此方法创建 LocalSpan
     * operationName: Span 的操作名（默认为方法名）
     */
    @Trace(operationName = "createOrder")
    @Tags({
        @Tag(key = "userId", value = "arg[0]"),     // 取第一个参数
        @Tag(key = "productId", value = "arg[1]"),  // 取第二个参数
        @Tag(key = "result", value = "returnedObj") // 取返回值
    })
    public Order createOrder(Long userId, Long productId, int quantity) {
        // 业务逻辑
        Order order = new Order();
        order.setUserId(userId);
        order.setProductId(productId);
        order.setQuantity(quantity);
        return orderRepository.save(order);
    }
}
```

#### 5.2.2 使用 TraceContext API

```java
import org.apache.skywalking.apm.toolkit.trace.TraceContext;
import org.apache.skywalking.apm.toolkit.trace.ActiveSpan;

@RestController
@RequestMapping("/api/order")
public class OrderController {

    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest req) {

        // 获取当前 TraceID（可记录到业务日志或返回给前端）
        String traceId = TraceContext.traceId();
        String segmentId = TraceContext.segmentId();
        int spanId = TraceContext.spanId();

        log.info("Processing order, traceId={}", traceId);

        // 在 Span 上添加自定义 Tag
        ActiveSpan.tag("order.userId", String.valueOf(req.getUserId()));
        ActiveSpan.tag("order.amount", String.valueOf(req.getAmount()));

        try {
            Order order = orderService.createOrder(req);

            // 添加成功标签
            ActiveSpan.tag("order.id", String.valueOf(order.getId()));

            // 在响应头中返回 TraceID（方便前端排查问题）
            return ResponseEntity.ok()
                .header("X-Trace-Id", traceId)
                .body(order);

        } catch (Exception e) {
            // 标记 Span 为错误状态并记录异常
            ActiveSpan.error(e);
            throw e;
        }
    }
}
```

#### 5.2.3 手动创建 Span（高级用法）

```java
import org.apache.skywalking.apm.agent.core.context.ContextManager;
import org.apache.skywalking.apm.agent.core.context.trace.AbstractSpan;
import org.apache.skywalking.apm.agent.core.context.trace.SpanLayer;

/**
 * 手动创建 Span，适用于自定义中间件或框架集成
 */
public class ManualTracingExample {

    public void callExternalService(String url, String payload) {
        // 创建 Exit Span
        ContextCarrier carrier = new ContextCarrier();
        AbstractSpan span = ContextManager.createExitSpan(
            "HTTP/POST " + url,  // 操作名
            carrier,              // 用于上下文传播
            "external-service:8080" // 对端地址
        );

        try {
            // 设置 Span 属性
            SpanLayer.asHttp(span);
            span.setComponent(ComponentsDefine.HTTPCLIENT);
            Tags.URL.set(span, url);
            Tags.HTTP.METHOD.set(span, "POST");

            // 将 TraceContext 注入到 HTTP Header
            CarrierItem item = carrier.items();
            while (item.hasNext()) {
                item = item.next();
                httpRequest.addHeader(item.getHeadKey(), item.getHeadValue());
            }

            // 发起实际 HTTP 请求
            HttpResponse response = httpClient.post(url, payload);

            Tags.STATUS_CODE.set(span, String.valueOf(response.getStatusCode()));

            if (response.getStatusCode() >= 400) {
                span.errorOccurred();
            }

        } catch (Exception e) {
            span.errorOccurred().log(e);
            throw e;
        } finally {
            // 必须停止 Span，否则会内存泄漏
            ContextManager.stopSpan();
        }
    }

    /**
     * 异步场景：跨线程传递 TraceContext
     */
    public void asyncTask() {
        // 在父线程中捕获 ContextSnapshot
        ContextSnapshot snapshot = ContextManager.capture();

        CompletableFuture.runAsync(() -> {
            // 在子线程中创建 LocalSpan
            AbstractSpan asyncSpan = ContextManager.createLocalSpan("async-task");

            // 关联父线程的 TraceContext
            ContextManager.continued(snapshot);

            try {
                // 执行异步业务逻辑
                doAsyncWork();
            } finally {
                ContextManager.stopSpan();
            }
        });
    }
}
```

### 5.3 采样策略

SkyWalking 提供三种采样策略：

#### 5.3.1 全采样（默认）

```properties
agent.sample_n_per_3_secs=-1
```

#### 5.3.2 速率采样

每 3 秒最多采集 N 条 Trace：
```properties
# 每 3 秒最多采集 3000 条（高流量服务推荐）
agent.sample_n_per_3_secs=3000
```

#### 5.3.3 基于百分比的采样

```java
// 自定义采样器（实现 TraceSampler 接口）
@Component
public class PercentageSampler implements TraceSampler {

    @Value("${skywalking.sample.rate:0.1}")
    private double sampleRate;

    @Override
    public boolean decideSampling() {
        return Math.random() < sampleRate;
    }
}
```

#### 5.3.4 动态采样（推荐生产使用）

通过 SkyWalking UI 的 Dynamic Configuration 动态调整采样率，无需重启应用：

```yaml
# OAP 配置：启用动态配置
configuration:
  selector: ${SW_CONFIGURATION:grpc}  # 支持 etcd/nacos/apollo/zookeeper
  grpc:
    host: ${SW_DCS_SERVER_HOST:localhost}
    port: ${SW_DCS_SERVER_PORT:11800}
```

### 5.4 查询 Trace 实战

在 SkyWalking UI 中查询 Trace 的步骤：

1. 打开 UI：`http://localhost:8080`
2. 进入 "Trace" 菜单
3. 设置过滤条件：
   - 服务名：`order-service`
   - 时间范围：过去 15 分钟
   - 最小耗时：> 500ms（查找慢请求）
   - 状态：Error（查找错误请求）
4. 点击某条 Trace 查看瀑布图

通过 GraphQL API 查询：
```graphql
query queryTrace($traceId: ID!) {
  trace: queryTrace(traceId: $traceId) {
    spans {
      traceId
      segmentId
      spanId
      parentSpanId
      serviceCode
      serviceInstanceName
      startTime
      endTime
      endpointName
      type
      peer
      component
      isError
      layer
      tags {
        key
        value
      }
      logs {
        time
        data {
          key
          value
        }
      }
    }
  }
}
```

---

## Part 6: Metrics 指标监控与 OAL {#part-6}

### 6.1 OAL（Observability Analysis Language）

OAL 是 SkyWalking 自定义的流式计算脚本语言，用于在 Trace/Log 数据的基础上实时计算 Metrics。

OAL 脚本位置：`config/oal/*.oal`

#### 6.1.1 内置 OAL 脚本示例

```
// config/oal/core.oal

// ===== Service 级别指标 =====

// 服务端点调用成功率（百分比，0-10000 表示 0%-100%）
service_sla = from(Service.*).percent(status == true);

// 服务端点平均响应时间（毫秒）
service_resp_time = from(Service.*).longAvg();

// 服务端点 CPM（每分钟调用次数）
service_cpm = from(Service.*).cpm();

// 服务端点 P99 响应时间
service_percentile = from(Service.*).percentile(10);

// ===== Endpoint 级别指标 =====

endpoint_cpm = from(Endpoint.*).cpm();
endpoint_avg = from(Endpoint.*).longAvg();
endpoint_sla = from(Endpoint.*).percent(status == true);
endpoint_percentile = from(Endpoint.*).percentile(10);

// ===== Database 级别指标 =====

// 数据库操作平均响应时间
database_access_resp_time = from(DatabaseAccess.*).longAvg();

// 数据库操作失败率
database_access_sla = from(DatabaseAccess.*).percent(status == true);
```

#### 6.1.2 自定义 OAL 指标

```
// 自定义：统计登录接口的错误率
// 文件：config/oal/custom.oal

// 按服务统计登录错误次数
login_error_count = from(Endpoint.*).filter(endpointName == "/api/auth/login").filter(status == false).count();

// 按服务统计登录 P95 响应时间
login_p95 = from(Endpoint.*).filter(endpointName == "/api/auth/login").percentile(10);

// 按 HTTP 状态码统计（需要 Tag）
http_5xx_count = from(Service.*).filter(httpResponseStatusCode >= 500).count();
```

### 6.2 内置 Metrics 一览

#### 6.2.1 Service 级别 Metrics

| Metric 名称 | 说明 | 单位 |
|------------|------|------|
| service_sla | 服务成功率 | 万分比（10000=100%） |
| service_cpm | 每分钟调用次数 | 次/分钟 |
| service_resp_time | 平均响应时间 | 毫秒 |
| service_percentile | P50/P75/P90/P95/P99 响应时间 | 毫秒 |
| service_apdex | Apdex 评分 | 0-10000 |

#### 6.2.2 Endpoint 级别 Metrics

| Metric 名称 | 说明 |
|------------|------|
| endpoint_cpm | 端点每分钟调用次数 |
| endpoint_avg | 端点平均响应时间 |
| endpoint_sla | 端点成功率 |
| endpoint_percentile | 端点响应时间分位数 |

#### 6.2.3 JVM 指标（由 Agent 自动采集）

| Metric 名称 | 说明 |
|------------|------|
| jvm_memory_heap_used | 堆内存使用量 |
| jvm_memory_heap_max | 堆内存最大值 |
| jvm_gc_count | GC 次数（Young/Old） |
| jvm_gc_time | GC 时间 |
| jvm_cpu_usage | CPU 使用率 |
| jvm_thread_live_count | 活跃线程数 |
| jvm_thread_daemon_count | 守护线程数 |

### 6.3 通过 GraphQL 查询 Metrics

```graphql
# 查询服务的响应时间趋势
query queryServiceMetrics {
  getValues: readMetricsValues(
    condition: {
      name: "service_resp_time"
      entity: {
        scope: Service
        serviceName: "order-service"
      }
    }
    duration: {
      start: "2026-07-08 0900"
      end: "2026-07-08 1000"
      step: MINUTE
    }
  ) {
    label
    values {
      values {
        id
        value
      }
    }
  }
}

# 查询 P99 响应时间
query queryP99 {
  readLabeledMetricsValues(
    condition: {
      name: "service_percentile"
      entity: {
        scope: Service
        serviceName: "order-service"
      }
    }
    labels: ["0", "1", "2", "3", "4"]  # P50, P75, P90, P95, P99
    duration: {
      start: "2026-07-08 0000"
      end: "2026-07-08 2359"
      step: HOUR
    }
  ) {
    label
    values {
      values {
        id
        value
      }
    }
  }
}
```

### 6.4 自定义 Dashboard

SkyWalking UI 支持自定义 Dashboard，通过 JSON 配置导入：

```json
{
  "name": "Order Service Dashboard",
  "type": "service",
  "configuration": [
    {
      "rowName": "Traffic",
      "columns": [
        {
          "name": "CPM",
          "type": "line",
          "metricName": "service_cpm",
          "unit": "req/min",
          "width": 4
        },
        {
          "name": "Success Rate",
          "type": "line",
          "metricName": "service_sla",
          "unit": "%",
          "width": 4,
          "tips": "value / 100"
        },
        {
          "name": "Avg Response Time",
          "type": "line",
          "metricName": "service_resp_time",
          "unit": "ms",
          "width": 4
        }
      ]
    },
    {
      "rowName": "Percentile",
      "columns": [
        {
          "name": "Response Time Percentile",
          "type": "line",
          "metricName": "service_percentile",
          "labels": "P50,P75,P90,P95,P99",
          "unit": "ms",
          "width": 12
        }
      ]
    }
  ]
}
```

---

## Part 7: 日志关联 {#part-7}

### 7.1 日志与 TraceID 关联

日志关联是指在日志中自动注入 TraceID 和 SpanID，使得日志与分布式追踪可以相互关联。

#### 7.1.1 Logback 集成

添加依赖：
```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-logback-1.x</artifactId>
    <version>9.1.0</version>
</dependency>
```

`logback-spring.xml` 配置：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <!-- SkyWalking TraceID 转换器 -->
    <conversionRule
        conversionWord="tid"
        converterClass="org.apache.skywalking.apm.toolkit.log.logback.v1.x.LogbackPatternConverter"/>

    <!-- 控制台输出（含 TraceID） -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>
                %d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50}
                [SW_TRACE_ID:%tid] - %msg%n
            </pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <!-- 文件输出（JSON 结构化格式） -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/application.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxHistory>30</maxHistory>
            <timeBasedFileNamingAndTriggeringPolicy
                class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
            <providers>
                <timestamp>
                    <timeZone>Asia/Shanghai</timeZone>
                </timestamp>
                <logLevel/>
                <loggerName/>
                <threadName/>
                <message/>
                <mdc/>
                <!-- 注入 SkyWalking TraceID -->
                <provider
                    class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.logstash.TraceIdJsonProvider"/>
                <stackTrace/>
            </providers>
        </encoder>
    </appender>

    <!-- 通过 gRPC 将日志上报到 OAP Server -->
    <appender name="GRPC-LOG"
        class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.log.GRPCLogClientAppender">
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
                <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%X{tid}] %-5level %logger{36} - %msg%n</Pattern>
            </layout>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="GRPC-LOG"/>
    </root>

</configuration>
```

输出效果：
```
2026-07-08 10:00:01.234 [http-nio-8080-exec-1] INFO  c.e.OrderService
[SW_TRACE_ID:abc123def456.1.1] - Creating order for user 12345
```

#### 7.1.2 Log4j2 集成

```xml
<dependency>
    <groupId>org.apache.skywalking</groupId>
    <artifactId>apm-toolkit-log4j-2.x</artifactId>
    <version>9.1.0</version>
</dependency>
```

`log4j2-spring.xml`：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="CONSOLE" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36}
                [TraceID: %traceId] - %msg%n" />
        </Console>

        <!-- SkyWalking gRPC Log Appender -->
        <GRPCLogClientAppender name="GRPC-LOG">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36}
                - %msg%n"/>
        </GRPCLogClientAppender>
    </Appenders>

    <Loggers>
        <Root level="info">
            <AppenderRef ref="CONSOLE"/>
            <AppenderRef ref="GRPC-LOG"/>
        </Root>
    </Loggers>
</Configuration>
```

### 7.2 通过 MDC 传递 TraceID

当使用 SkyWalking SDK 时，TraceID 会自动注入到 SLF4J MDC 中：

```java
// 在日志 Pattern 中使用 %X{SW_CTX} 或 %X{tid}
// SkyWalking 会自动将以下信息写入 MDC：
// tid = traceId (如果没有活跃Trace则为 N/A)

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Order processOrder(Long orderId) {
        // 手动将 TraceID 写入 MDC（兜底方案）
        String traceId = TraceContext.traceId();
        if (!"N/A".equals(traceId)) {
            MDC.put("traceId", traceId);
        }

        log.info("Processing order: {}", orderId);
        // 日志输出：2026-07-08 10:00:01 INFO ... [traceId=abc-123] Processing order: 456

        try {
            return doProcess(orderId);
        } finally {
            MDC.remove("traceId");
        }
    }
}
```

### 7.3 日志收集架构

```
+------------------------------------------------------------+
|                    日志收集全链路                             |
+------------------------------------------------------------+
|                                                            |
|  应用层                                                     |
|  +------------------+                                      |
|  | Spring Boot App  |                                      |
|  | + SW Agent       |                                      |
|  |                  |                                      |
|  |  Logback         +---> logs/app.log (含TraceID)          |
|  |  GRPC Appender   +---> OAP:11800 (实时上报)              |
|  +------------------+                                      |
|           |                                                |
|           v (filebeat/fluentd)                             |
|  +------------------+                                      |
|  | Log Collector    |                                      |
|  | (Filebeat 等)    |                                       |
|  +------------------+                                      |
|           |                                                |
|           v                                                |
|  +------------------+                                      |
|  | OAP Server       |                                      |
|  | (日志分析/存储)    |                                       |
|  +------------------+                                      |
|           |                                                |
|           v                                                |
|  +------------------+                                      |
|  | Elasticsearch    |                                      |
|  | Index: sw-log-*  |                                      |
|  +------------------+                                      |
|           |                                                |
|           v                                                |
|  +------------------+                                      |
|  | SkyWalking UI    |                                      |
|  | (Log 查询 + 与    |                                       |
|  |  Trace 联动)      |                                      |
|  +------------------+                                      |
|                                                            |
+------------------------------------------------------------+
```

### 7.4 通过 Log 查询 Trace（UI 操作）

1. 在 SkyWalking UI 中进入 "Log" 菜单
2. 按关键词搜索：`error timeout orderId:12345`
3. 找到目标日志条目
4. 点击日志旁边的 "Trace" 图标
5. 直接跳转到对应的 Trace 详情页

反向操作（从 Trace 找 Log）：
1. 在 Trace 详情页中选中某个 Span
2. 点击 "Logs" Tab
3. 查看该 Span 时间范围内的相关日志

---

## Part 8: 告警配置 {#part-8}

### 8.1 告警规则配置

告警配置文件：`config/alarm-settings.yml`

```yaml
rules:
  # ===== 服务响应时间告警 =====
  service_resp_time_rule:
    metrics-name: service_resp_time           # 使用的 Metrics 名称
    op: ">"                                   # 操作符：> < >= <= =
    threshold: 1000                           # 阈值：1000ms
    period: 10                                # 检测周期：10 分钟
    count: 3                                  # 触发次数：10 分钟内超过 3 次
    silence-period: 5                         # 静默期：告警后 5 分钟内不重复告警
    message: "服务【{name}】响应时间超过 1000ms，当前值：{value}ms"
    tags:
      level: WARNING

  # ===== 服务成功率告警 =====
  service_sla_rule:
    metrics-name: service_sla
    op: "<"
    threshold: 8000                           # 8000 = 80%（单位是万分比）
    period: 10
    count: 2
    silence-period: 3
    message: "服务【{name}】成功率低于 80%，当前值：{value}"
    tags:
      level: CRITICAL

  # ===== Endpoint P99 响应时间告警 =====
  endpoint_relation_resp_time_rule:
    metrics-name: endpoint_p99
    threshold: 500
    op: ">"
    period: 5
    count: 1
    message: "端点【{name}】P99 响应时间超过 500ms，当前值：{value}ms"
    include-endpoints:
      - /api/order
      - /api/payment

  # ===== 实例 JVM 内存告警 =====
  instance_jvm_memory_heap_rule:
    metrics-name: instance_jvm_memory_heap_used
    threshold: 1073741824                     # 1GB（字节）
    op: ">"
    period: 5
    count: 1
    message: "实例【{name}】堆内存使用超过 1GB，当前值：{value} bytes"
    tags:
      level: WARNING

  # ===== 数据库调用响应时间告警 =====
  database_access_resp_time_rule:
    metrics-name: database_access_resp_time
    threshold: 200
    op: ">"
    period: 10
    count: 5
    message: "数据库【{name}】平均响应时间超过 200ms，当前值：{value}ms"

# 告警钩子配置
hooks:
  webhook:
    default:
      is-default: true
      urls:
        - http://your-alert-receiver:8080/skywalking/alerts

  # 钉钉告警
  dingtalk:
    default:
      is-default: true
      url: https://oapi.dingtalk.com/robot/send?access_token=YOUR_TOKEN
      secret: YOUR_SECRET_KEY
      atMobiles:
        - "13800138000"
      isAtAll: false

  # 企业微信告警
  wechat:
    default:
      is-default: true
      url: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=YOUR_KEY

  # Slack 告警
  slack:
    default:
      is-default: true
      textTemplate: |-
        {
          "type": "section",
          "text": {
            "type": "mrkdwn",
            "text": "*SkyWalking Alert*: {{ .RuleName }}\n{{ .Message }}"
          }
        }
      channels:
        - alerts-channel
```

### 8.2 自定义 Webhook 接收器（Spring Boot）

```java
@RestController
@RequestMapping("/skywalking")
@Slf4j
public class SkyWalkingAlertController {

    @Autowired
    private AlertNotificationService notificationService;

    /**
     * SkyWalking 告警 Webhook 接收端点
     * SkyWalking 发送 POST 请求，Body 为 JSON 数组
     */
    @PostMapping("/alerts")
    public ResponseEntity<Void> receiveAlerts(
            @RequestBody List<AlertMessage> alerts) {

        log.info("Received {} alerts from SkyWalking", alerts.size());

        for (AlertMessage alert : alerts) {
            log.warn("SkyWalking Alert: ruleName={}, message={}, startTime={}",
                alert.getRuleName(), alert.getAlarmMessage(),
                Instant.ofEpochMilli(alert.getStartTime()));

            // 根据告警级别路由通知
            String level = alert.getTags().getOrDefault("level", "WARNING");

            switch (level) {
                case "CRITICAL":
                    notificationService.sendPagerDuty(alert);
                    notificationService.sendSMS(alert);
                    break;
                case "WARNING":
                    notificationService.sendDingTalk(alert);
                    break;
                default:
                    notificationService.sendEmail(alert);
            }
        }

        return ResponseEntity.ok().build();
    }
}

/**
 * SkyWalking 告警消息结构
 */
@Data
public class AlertMessage {
    private String scopeId;
    private String scope;
    private String name;          // 触发告警的实体名（服务名/端点名）
    private String id0;
    private String id1;
    private String ruleName;      // 告警规则名
    private String alarmMessage;  // 告警消息内容
    private long startTime;       // 告警开始时间（毫秒时间戳）
    private Map<String, String> tags; // 自定义标签
}
```

### 8.3 告警静默与抑制

SkyWalking 内置的静默机制：
- `silence-period`：触发告警后，在静默期内相同规则不再触发
- `count`：在 `period` 时间窗口内，需要满足 `count` 次才触发（避免抖动）

进阶抑制（外部实现）：
```java
@Service
public class AlertSuppressService {

    @Autowired
    private RedisTemplate<String, String> redisTemplate;

    /**
     * 基于 Redis 实现告警去重（1 小时内同一服务的同一规则只通知一次）
     */
    public boolean shouldNotify(AlertMessage alert) {
        String key = String.format("sw:alert:%s:%s",
            alert.getName(), alert.getRuleName());

        // setIfAbsent = SETNX（不存在才设置），有效期 1 小时
        Boolean isNew = redisTemplate.opsForValue()
            .setIfAbsent(key, "1", Duration.ofHours(1));

        return Boolean.TRUE.equals(isNew);
    }
}
```

---

## Part 9: Service Mesh 集成 {#part-9}

### 9.1 SkyWalking 与 Istio 集成

在 Service Mesh 架构中，Envoy Sidecar 代理了所有的网络流量。SkyWalking 可以接收 Envoy 的 Access Log 数据，自动生成服务拓扑和 Metrics，**无需部署 Java Agent**。

```
+---------------------------------------------------------------+
|                    Service Mesh 可观测性架构                    |
+---------------------------------------------------------------+
|                                                               |
|  Pod A                          Pod B                         |
|  +--------------------+         +--------------------+        |
|  | [App Container]    |         | [App Container]    |        |
|  |   (无 Agent)        |         |   (无 Agent)        |        |
|  +---------+----------+         +---------+----------+        |
|            | 127.0.0.1                    |                   |
|  +---------+----------+         +---------+----------+        |
|  | [Envoy Sidecar]    | ------> | [Envoy Sidecar]    |        |
|  |  Access Log &      |         |  Access Log &      |        |
|  |  Metrics Export    |         |  Metrics Export    |        |
|  +--------------------+         +--------------------+        |
|            |                             |                    |
|            +------------+----------------+                    |
|                         | ALS (Access Log Service)            |
|                         v                                     |
|                 +-------+-------+                             |
|                 |  OAP Server   |                             |
|                 | (处理 ALS数据) |                             |
|                 +-------+-------+                             |
|                         |                                     |
|                 +-------+-------+                             |
|                 | Elasticsearch |                             |
|                 +---------------+                             |
|                                                               |
+---------------------------------------------------------------+
```

### 9.2 配置 Envoy ALS 接收器

在 OAP `application.yml` 中启用 ALS 接收器：
```yaml
receiver-envoy:
  selector: ${SW_RECEIVER_ENVOY:default}
  default:
    grpcPort: ${SW_RECEIVER_GRPC_PORT:11800}

envoy-metric:
  selector: ${SW_ENVOY_METRIC:default}
  default:
    # ALS 分析策略：k8s-mesh（使用 K8s 元数据）
    analysisK8sInfo: ${SW_ENVOY_METRIC_ANALYSIS_K8S_INFO:true}
    # 是否开启 Zipkin 格式的 Trace 接收
    acceptProxyRequests: ${SW_ENVOY_METRIC_ACCEPT_PROXY_REQUESTS:false}
```

### 9.3 Istio 配置（开启 Envoy ALS）

```yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
metadata:
  namespace: istio-system
spec:
  meshConfig:
    # 开启 Envoy Access Log Service
    enableEnvoyAccessLogService: true
    defaultConfig:
      envoyAccessLogService:
        address: skywalking-oap.monitoring:11800
      tracing:
        # 使用 Zipkin 协议发送 Trace 到 SkyWalking OAP
        zipkin:
          address: skywalking-oap.monitoring:9411
    # 追踪采样率（1.0 = 100%）
    traceSampling: 100.0
```

### 9.4 混合架构：Agent + Service Mesh

在过渡期，可以同时使用 Java Agent 和 Service Mesh，OAP 会自动合并来自两侧的数据：

```
+---------------------------------------------------+
|             混合部署模式                            |
+---------------------------------------------------+
|                                                   |
|  Java 服务（保留 Agent）                           |
|  +----------------+                               |
|  | Java App       |                               |
|  | + SW Agent     | ---gRPC--> OAP :11800          |
|  +-----+----------+                               |
|        | HTTP                                     |
|        v                                          |
|  Go/Node.js 服务（无 Agent，依赖 Envoy）            |
|  +----------------+                               |
|  | Go App         |                               |
|  | [Envoy Sidecar]| ---ALS---> OAP :11800          |
|  +----------------+                               |
|                                                   |
|  OAP 会将两条链路自动关联成完整 Trace               |
|  （通过 x-b3-traceid 或 sw8 Header）               |
|                                                   |
+---------------------------------------------------+
```

---

## Part 10: 完整实战案例 {#part-10}

### 10.1 项目背景

以一个电商平台为例，包含以下微服务：
- **api-gateway**：API 网关（Spring Cloud Gateway）
- **order-service**：订单服务
- **inventory-service**：库存服务
- **payment-service**：支付服务
- **notification-service**：通知服务（Kafka 消费者）

### 10.2 完整项目结构

```
ecommerce-platform/
+-- docker-compose.yml
+-- api-gateway/
|   +-- src/main/resources/
|   |   +-- application.yml
|   |   +-- logback-spring.xml
|   +-- Dockerfile
+-- order-service/
|   +-- src/main/java/com/example/order/
|   |   +-- OrderServiceApplication.java
|   |   +-- controller/OrderController.java
|   |   +-- service/OrderService.java
|   |   +-- config/TracingConfig.java
|   +-- src/main/resources/
|   |   +-- application.yml
|   |   +-- logback-spring.xml
|   +-- Dockerfile
+-- skywalking/
    +-- agent.config
    +-- alarm-settings.yml
```

### 10.3 核心代码实现

#### 10.3.1 api-gateway 集成 SkyWalking

```yaml
# api-gateway/application.yml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**
          filters:
            - StripPrefix=0

server:
  port: 8080
```

```dockerfile
# api-gateway/Dockerfile
FROM openjdk:17-slim
COPY --from=apache/skywalking-java-agent:9.1.0-java17 \
     /skywalking/agent /skywalking/agent
COPY target/api-gateway.jar /app/api-gateway.jar
ENV SW_AGENT_NAME=api-gateway
ENV SW_AGENT_COLLECTOR_BACKEND_SERVICES=oap:11800
ENTRYPOINT ["java", \
  "-javaagent:/skywalking/agent/skywalking-agent.jar", \
  "-jar", "/app/api-gateway.jar"]
```

#### 10.3.2 order-service 核心实现

```java
// OrderController.java
@RestController
@RequestMapping("/api/order")
@Slf4j
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @RequestBody @Valid CreateOrderRequest request,
            HttpServletRequest httpRequest) {

        String traceId = TraceContext.traceId();
        log.info("Received create order request, userId={}, traceId={}",
            request.getUserId(), traceId);

        // 在 Span 上记录业务信息
        ActiveSpan.tag("order.userId", String.valueOf(request.getUserId()));
        ActiveSpan.tag("order.productId", String.valueOf(request.getProductId()));
        ActiveSpan.tag("order.quantity", String.valueOf(request.getQuantity()));

        try {
            OrderResponse response = orderService.createOrder(request);

            log.info("Order created successfully, orderId={}, traceId={}",
                response.getOrderId(), traceId);

            return ResponseEntity.ok()
                .header("X-Trace-Id", traceId)
                .body(response);

        } catch (InsufficientInventoryException e) {
            log.warn("Insufficient inventory for product {}, traceId={}",
                request.getProductId(), traceId);
            ActiveSpan.error(e);
            return ResponseEntity.badRequest()
                .header("X-Trace-Id", traceId)
                .body(OrderResponse.failed("库存不足"));

        } catch (Exception e) {
            log.error("Failed to create order, traceId={}", traceId, e);
            ActiveSpan.error(e);
            return ResponseEntity.internalServerError()
                .header("X-Trace-Id", traceId)
                .body(OrderResponse.failed("系统错误"));
        }
    }

    @GetMapping("/{orderId}")
    @Trace(operationName = "getOrderById")
    @Tag(key = "orderId", value = "arg[0]")
    public ResponseEntity<Order> getOrder(@PathVariable Long orderId) {
        return orderService.findById(orderId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

```java
// OrderService.java
@Service
@Slf4j
@Transactional
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private InventoryClient inventoryClient;    // Feign Client

    @Autowired
    private PaymentClient paymentClient;        // Feign Client

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Trace(operationName = "OrderService.createOrder")
    public OrderResponse createOrder(CreateOrderRequest request) {

        // Step 1: 检查库存（跨服务调用，SW Agent 自动追踪）
        InventoryResponse inventory = inventoryClient.checkInventory(
            request.getProductId(), request.getQuantity());

        if (!inventory.isAvailable()) {
            throw new InsufficientInventoryException(request.getProductId());
        }

        // Step 2: 创建订单记录
        Order order = Order.builder()
            .userId(request.getUserId())
            .productId(request.getProductId())
            .quantity(request.getQuantity())
            .amount(inventory.getUnitPrice().multiply(
                BigDecimal.valueOf(request.getQuantity())))
            .status(OrderStatus.PENDING)
            .traceId(TraceContext.traceId()) // 存储 TraceID 到订单记录
            .build();

        order = orderRepository.save(order);

        // Step 3: 扣减库存
        inventoryClient.deductInventory(request.getProductId(), request.getQuantity());

        // Step 4: 发起支付（异步，通过 Kafka）
        OrderEvent event = new OrderEvent(order.getId(), order.getAmount(),
            TraceContext.traceId()); // 在事件中传递 TraceID
        kafkaTemplate.send("order-created", event);

        // Step 5: 更新订单状态
        order.setStatus(OrderStatus.PROCESSING);
        orderRepository.save(order);

        log.info("Order {} created, sent to payment queue", order.getId());

        return OrderResponse.success(order.getId(), TraceContext.traceId());
    }
}
```

#### 10.3.3 Kafka 消息追踪

SkyWalking Kafka 插件会自动传递 TraceContext，但在消费端需要正确配置：

```java
// notification-service/KafkaConsumer.java
@Component
@Slf4j
public class OrderEventConsumer {

    @KafkaListener(topics = "order-created", groupId = "notification-group")
    public void handleOrderCreated(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic) {

        // SkyWalking Kafka 插件会自动：
        // 1. 从 Kafka Header 中提取 sw8 (TraceContext)
        // 2. 创建 Entry Span 关联上游 Trace
        // 所以这里直接获取的 TraceID 会与上游 Order Service 的 Trace 关联

        String traceId = TraceContext.traceId();
        log.info("Received order event, orderId={}, traceId={}",
            event.getOrderId(), traceId);

        // 业务处理：发送订单确认通知
        sendOrderConfirmation(event.getOrderId());
    }

    @Trace(operationName = "sendOrderConfirmation")
    private void sendOrderConfirmation(Long orderId) {
        // 发送短信/邮件/推送通知
        log.info("Sending order confirmation for orderId={}", orderId);
    }
}
```

### 10.4 完整 docker-compose.yml

```yaml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.10.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - ES_JAVA_OPTS=-Xms2g -Xmx2g
    volumes:
      - es-data:/usr/share/elasticsearch/data
    networks:
      - ecommerce

  oap:
    image: apache/skywalking-oap-server:9.6.0
    depends_on:
      - elasticsearch
    environment:
      SW_STORAGE: elasticsearch
      SW_STORAGE_ES_CLUSTER_NODES: elasticsearch:9200
    ports:
      - "11800:11800"
      - "12800:12800"
    networks:
      - ecommerce

  skywalking-ui:
    image: apache/skywalking-ui:9.6.0
    depends_on:
      - oap
    environment:
      SW_OAP_ADDRESS: http://oap:12800
    ports:
      - "8080:8080"
    networks:
      - ecommerce

  zookeeper:
    image: bitnami/zookeeper:3.8
    environment:
      ALLOW_ANONYMOUS_LOGIN: "yes"
    networks:
      - ecommerce

  kafka:
    image: bitnami/kafka:3.4
    depends_on:
      - zookeeper
    environment:
      KAFKA_CFG_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      ALLOW_PLAINTEXT_LISTENER: "yes"
    networks:
      - ecommerce

  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ecommerce
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - ecommerce

  redis:
    image: redis:7-alpine
    networks:
      - ecommerce

  api-gateway:
    build: ./api-gateway
    depends_on:
      - oap
      - order-service
    environment:
      SW_AGENT_NAME: api-gateway
      SW_AGENT_COLLECTOR_BACKEND_SERVICES: oap:11800
    ports:
      - "9090:8080"
    networks:
      - ecommerce

  order-service:
    build: ./order-service
    depends_on:
      - oap
      - mysql
      - kafka
      - inventory-service
    environment:
      SW_AGENT_NAME: order-service
      SW_AGENT_COLLECTOR_BACKEND_SERVICES: oap:11800
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/ecommerce
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    networks:
      - ecommerce

  inventory-service:
    build: ./inventory-service
    depends_on:
      - oap
      - mysql
      - redis
    environment:
      SW_AGENT_NAME: inventory-service
      SW_AGENT_COLLECTOR_BACKEND_SERVICES: oap:11800
    networks:
      - ecommerce

  notification-service:
    build: ./notification-service
    depends_on:
      - oap
      - kafka
    environment:
      SW_AGENT_NAME: notification-service
      SW_AGENT_COLLECTOR_BACKEND_SERVICES: oap:11800
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:9092
    networks:
      - ecommerce

volumes:
  es-data:
  mysql-data:

networks:
  ecommerce:
    driver: bridge
```

---

## Part 11: 性能调优与运维 {#part-11}

### 11.1 OAP Server 性能调优

#### 11.1.1 JVM 参数优化

```bash
# 生产环境推荐的 JVM 参数（16GB 内存机器）
export JAVA_OPTS="
  -Xms8g
  -Xmx8g
  -XX:+UseG1GC
  -XX:G1HeapRegionSize=32m
  -XX:MaxGCPauseMillis=200
  -XX:+HeapDumpOnOutOfMemoryError
  -XX:HeapDumpPath=/logs/oap-heapdump.hprof
  -verbose:gc
  -Xloggc:/logs/oap-gc.log
  -XX:+PrintGCDetails
  -XX:+PrintGCDateStamps
"
```

#### 11.1.2 OAP 配置调优

`application.yml` 关键配置：

```yaml
core:
  selector: ${SW_CORE:default}
  default:
    # 数据过期时间（天）
    recordDataTTL: ${SW_CORE_RECORD_DATA_TTL:3}      # Trace/Log 保留 3 天
    metricsDataTTL: ${SW_CORE_METRICS_DATA_TTL:7}   # Metrics 保留 7 天

    # 降采样（超过阈值的 Trace 进行降采样存储）
    enableDatabaseSession: ${SW_CORE_ENABLE_DB_SESSION:true}

storage:
  elasticsearch:
    # 每个 ES 索引的分片数（建议 = ES 数据节点数）
    indexShardsNumber: ${SW_STORAGE_ES_INDEX_SHARDS_NUMBER:1}
    # 副本数（生产至少为 1）
    indexReplicasNumber: ${SW_STORAGE_ES_INDEX_REPLICAS_NUMBER:1}
    # 批量写入配置
    bulkActions: ${SW_STORAGE_ES_BULK_ACTIONS:5000}      # 每批 5000 条
    bulkSize: ${SW_STORAGE_ES_BULK_SIZE:40}               # 每批最大 40MB
    flushInterval: ${SW_STORAGE_ES_FLUSH_INTERVAL:10}    # 10 秒刷新一次
    concurrentRequests: ${SW_STORAGE_ES_CONCURRENT_REQUESTS:2}

receiver-trace:
  default:
    # Agent 上报队列大小（增大可提升并发，但消耗内存）
    bufferPath: ${SW_RECEIVER_BUFFER_PATH:../trace-buffer/}
    bufferOffsetMaxFileSize: ${SW_RECEIVER_BUFFER_OFFSET_MAX_FILE_SIZE:100}
    bufferDataMaxFileSize: ${SW_RECEIVER_BUFFER_DATA_MAX_FILE_SIZE:500}

# Zipkin 兼容接收器（如果需要接收 Zipkin 格式的 Trace）
receiver-zipkin:
  selector: ${SW_RECEIVER_ZIPKIN:-}  # 默认关闭，需要时设置为 default
```

### 11.2 Agent 性能调优

#### 11.2.1 减少 Agent 性能开销

```properties
# 启用异步上报（减少业务线程阻塞）
agent.collector.grpc_channel_check_interval=30
agent.collector.heartbeat_period=30

# 调整采样率（高流量服务降低采样率）
agent.sample_n_per_3_secs=1000  # 每 3 秒最多 1000 条

# 忽略不需要追踪的路径（减少数据量）
agent.ignore_suffix=.jpg,.jpeg,.js,.css,.png,.ico,.html,.svg,.woff,.ttf
agent.ignored_pathes=/health,/actuator/**,/metrics

# 降低 JVM Metrics 采集频率
jvm.buffer_size=600

# 关闭不需要的插件
plugin.exclude_plugins=shenyu,motan  # 排除不使用的插件
```

#### 11.2.2 插件按需启用

将不需要的插件从 `plugins/` 目录移到 `optional-plugins/`：

```bash
# 示例：禁用 dubbo 插件（如果不使用 dubbo）
mv plugins/apm-dubbo-3.x-plugin.jar optional-plugins/

# 查看已加载的插件
ls plugins/*.jar | wc -l
```

### 11.3 Elasticsearch 调优

#### 11.3.1 索引生命周期管理（ILM）

SkyWalking 默认按天/月创建 ES 索引，可以配合 ILM 自动管理：

```bash
# 创建 ILM Policy
PUT _ilm/policy/skywalking-policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "3d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          },
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "delete": {
        "min_age": "7d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

#### 11.3.2 ES 查询优化

为常见查询字段创建索引优化：
```bash
# SkyWalking 会自动创建以下核心索引
# sw-segment-YYYYMM: Trace Segment 数据
# sw-metrics-*: Metrics 数据
# sw-log-YYYYMM: 日志数据
# sw-records-*: 慢 SQL、采样端点数据

# 监控 ES 健康状态
GET _cluster/health

# 查看 SkyWalking 相关索引
GET _cat/indices/sw-*?v&h=index,docs.count,store.size&s=store.size:desc
```

### 11.4 常见问题排查

#### 11.4.1 Agent 无法连接 OAP

排查步骤：
```bash
# 1. 检查 Agent 日志
tail -f /opt/skywalking-agent/logs/skywalking-api.log

# 常见错误：
# "Can not connect to [oap:11800]" -> OAP 地址不可达
# "Connect to OAP server timeout" -> 防火墙/安全组问题

# 2. 验证网络连通性
telnet oap-host 11800
nc -zv oap-host 11800

# 3. 检查 OAP 是否正常启动
curl http://oap-host:12800/graphql -X POST \
  -H "Content-Type: application/json" \
  -d '{"query": "{ version }"}'
```

#### 11.4.2 Trace 数据不显示

```bash
# 1. 确认服务名配置正确
# agent.service_name 不能含特殊字符，不能以数字开头

# 2. 检查采样率
grep "sample_n_per_3_secs" /opt/skywalking-agent/config/agent.config

# 3. 查看 OAP 是否收到数据
# 在 OAP 日志中搜索
grep -i "trace" /opt/skywalking-oap/logs/skywalking-oap.log | tail -50

# 4. 直接测试 gRPC 连接
grpc_cli ls oap-host:11800
```

#### 11.4.3 Elasticsearch 存储故障

```bash
# 1. 检查 ES 磁盘使用率（超过 85% 会触发只读保护）
GET _cat/allocation?v

# 2. 清理旧索引
DELETE sw-segment-202501   # 删除 2025 年 1 月的 Trace 数据

# 3. 修改 OAP TTL 配置（不重启，动态生效）
# 在 SkyWalking UI -> Settings -> TTL 中修改
```

### 11.5 监控 OAP 自身

SkyWalking OAP 支持将自身的指标暴露给 Prometheus：

```yaml
# application.yml
telemetry:
  selector: ${SW_TELEMETRY:prometheus}
  prometheus:
    host: ${SW_TELEMETRY_PROMETHEUS_HOST:0.0.0.0}
    port: ${SW_TELEMETRY_PROMETHEUS_PORT:1234}
```

关键指标：
- `oap_uptime` - OAP 运行时间
- `grpc_server_receive_queue_size` - gRPC 接收队列长度（告警阈值：> 1000）
- `persistence_timer_bulk_error_count` - ES 写入错误数（告警阈值：> 0）
- `remote_in_latency` - OAP 集群节点间延迟

---

## Part 12: 常见面试题 FAQ {#part-12}

### Q1: SkyWalking 的 Java Agent 是如何做到无侵入监控的？

**答：**

SkyWalking Java Agent 基于 Java 的 **Instrumentation API** 和 **ByteBuddy** 字节码增强框架实现无侵入监控。

具体原理：
1. 通过 JVM 启动参数 `-javaagent:skywalking-agent.jar` 加载 Agent
2. Agent 的 `premain()` 方法在 `main()` 之前执行
3. 注册 `ClassFileTransformer`，在类加载时拦截目标类（如 Spring MVC Controller）
4. 使用 ByteBuddy 在方法前后注入代码（`before/after/exception` 拦截点）
5. 注入的代码负责创建 Span、填充属性、传播 Context、上报数据

业务代码完全无感知，不需要修改任何代码或添加任何注解。

---

### Q2: SkyWalking 和 Zipkin、Jaeger 的区别是什么？

**答：**

| 维度 | SkyWalking | Zipkin | Jaeger |
|------|-----------|--------|--------|
| 功能范围 | Metrics + Tracing + Logging | 仅 Tracing | 仅 Tracing |
| Agent 方式 | 自动注入（无侵入） | 需手动集成库 | 需手动集成库 |
| 语言支持 | Java/Go/.NET/Node/PHP/Python | Java/Go/Ruby 等 | Java/Go/Python 等 |
| UI 丰富度 | 丰富（拓扑/分析/剖析） | 基础 | 基础 |
| 服务拓扑 | 自动生成 | 不支持 | 基础支持 |
| 持续剖析 | 支持 | 不支持 | 不支持 |
| 存储后端 | ES/MySQL/TiDB/BanyanDB | ES/MySQL/Cassandra | ES/Cassandra/Kafka |
| 协议兼容 | 自有协议 + Zipkin 兼容 | Zipkin | Jaeger + Zipkin |

**总结**：SkyWalking 是功能更完整的 APM 平台，Zipkin/Jaeger 是专注于分布式追踪的轻量级工具。

---

### Q3: 分布式追踪中 TraceID 是如何在多个服务间传递的？

**答：**

以 HTTP 调用为例：

1. **发起调用时（Exit Span）**：SkyWalking Agent 在 HTTP Request Header 中注入 `sw8` 字段，内容包含 TraceID、SegmentID、SpanID、服务名等信息（Base64 编码）

2. **接收请求时（Entry Span）**：目标服务的 Agent 从 HTTP Header 中提取 `sw8`，解析出父 TraceID 和 SpanID，创建关联的 Entry Span

3. **消息队列**：通过 Kafka/RocketMQ 的自定义 Header 传递 TraceContext

4. **跨线程**：通过 `ContextManager.capture()` 捕获 ContextSnapshot，在子线程中通过 `ContextManager.continued(snapshot)` 恢复

`sw8` Header 格式：
```
sw8: {sample}-{traceId}-{segmentId}-{spanId}-{service}-{instance}-{endpoint}-{clientType}
```

---

### Q4: OAL 是什么？如何自定义 Metrics？

**答：**

OAL（Observability Analysis Language）是 SkyWalking 设计的**流式计算 DSL**，用于从 Trace/Log 原始数据实时计算 Metrics。

**语法结构：**
```
metrics_name = from(Scope.filter).function(condition)
```

**示例：**
```
# 计算订单服务的成功率
order_sla = from(Service.*).filter(serviceName == "order-service").percent(status == true);

# 计算支付接口的 P99 响应时间（按端点维度）
payment_p99 = from(Endpoint.*).filter(endpointName == "/api/payment").percentile(10);
```

**支持的聚合函数：**
- `longAvg()` - 平均值
- `cpm()` - 每分钟次数
- `percent(condition)` - 满足条件的百分比
- `percentile(precision)` - 分位数（P50/P75/P90/P95/P99）
- `count()` - 计数
- `sum()` - 求和
- `histogram()` - 直方图

---

### Q5: SkyWalking 如何实现告警？支持哪些通知方式？

**答：**

SkyWalking 告警基于 **滑动窗口** 机制：

1. 每隔 `period` 分钟检查一次 Metrics 是否满足告警条件
2. 如果在 `period` 时间窗口内，触发次数 >= `count`，则发送告警
3. 发送告警后，在 `silence-period` 分钟内不重复告警

**支持的通知方式：**
- Webhook（自定义 HTTP 接口）
- 钉钉机器人
- 企业微信机器人
- Slack
- PagerDuty
- Discord
- 飞书（Lark）
- 邮件（需要自定义 Webhook 实现）

---

### Q6: 如何排查某个接口响应慢的问题？

**答：**

标准排查流程：

1. **Metrics 定位**：在 SkyWalking UI 的 Dashboard 中查看各服务的 P99 响应时间趋势，确认哪个服务/端点有异常

2. **Trace 分析**：
   - 进入 Trace 查询，按服务名 + 慢响应时间（> Xms）过滤
   - 选择一条典型的慢 Trace
   - 查看瀑布图，找出耗时最长的 Span

3. **Span 详情**：
   - 查看 Span 的 Tags（如 `db.statement` 可以看到慢 SQL）
   - 查看 Span 的 Logs（可能有异常堆栈）

4. **关联日志**：
   - 在 Log 视图中按 TraceID 查询，获取完整的上下文日志
   - 确认是否有 timeout / retry / connection refused 等关键词

5. **持续剖析**：
   - 如果无法通过 Trace 定位，使用 SkyWalking 的 Continuous Profiling
   - 开启对目标实例的 CPU/Memory 剖析，获取方法级别的火焰图

---

### Q7: SkyWalking 的采样策略有哪些？生产环境如何配置？

**答：**

SkyWalking 提供以下采样策略：

1. **全采样**（`sample_n_per_3_secs=-1`）：所有 Trace 都记录，适合低流量或开发环境

2. **速率采样**（`sample_n_per_3_secs=N`）：每 3 秒最多采集 N 条 Trace，适合中高流量

3. **动态采样**：通过 Dynamic Configuration（Nacos/Etcd/Apollo）动态调整，无需重启

**生产环境推荐：**
- 普通接口：`sample_n_per_3_secs=1000`（约 33% 采样率，高流量下）
- 核心链路（支付/订单）：`sample_n_per_3_secs=-1`（全采样）
- 错误 Trace 强制采样：通过自定义采样器，对 status=error 的 Trace 强制采样

**注意**：采样仅影响 Trace 存储，Metrics 始终是全量计算的（100% 准确）。

---

### Q8: 如何在异步/多线程场景中保持 TraceContext？

**答：**

SkyWalking 提供了跨线程 TraceContext 传递机制：

```java
// 方案 1：使用 ContextSnapshot 手动传递
ContextSnapshot snapshot = ContextManager.capture();

CompletableFuture.runAsync(() -> {
    AbstractSpan span = ContextManager.createLocalSpan("async-task");
    ContextManager.continued(snapshot);  // 关联父 TraceContext
    try {
        // 业务逻辑
    } finally {
        ContextManager.stopSpan();
    }
});

// 方案 2：使用 @TraceCrossThread 注解（需要 apm-toolkit-trace）
@TraceCrossThread
public Callable<String> createAsyncTask() {
    return () -> {
        // 自动关联父 TraceContext
        return "result";
    };
}

// 方案 3：使用 RunnableWrapper/CallableWrapper
ExecutorService executor = Executors.newFixedThreadPool(10);
executor.submit(RunnableWrapper.of(() -> {
    // 自动关联父 TraceContext
    doWork();
}));
```

---

### Q9: SkyWalking 的存储有哪些选择？如何选型？

**答：**

| 存储 | 适用规模 | 优点 | 缺点 |
|------|---------|------|------|
| H2 | 开发/测试 | 零配置 | 重启丢数据，不支持集群 |
| MySQL | 小规模（< 100 QPM） | 运维简单 | 扩展性差，超大数据量性能下降 |
| TiDB | 中等规模 | MySQL 兼容，水平扩展 | 部署复杂度高 |
| **Elasticsearch** | **中大规模（推荐）** | 搜索性能强，生态成熟 | 资源消耗较大 |
| OpenSearch | 中大规模 | ES 开源替代，AWS 友好 | 社区相对小 |
| BanyanDB | 大规模（官方推荐） | 专为 SkyWalking 设计，资源效率高 | 较新，生态不如 ES |

**选型建议：**
- 单机/小团队：MySQL + 短 TTL（Metrics 7天，Trace 3天）
- 中等团队：Elasticsearch 单节点或 3 节点集群
- 大规模：Elasticsearch 集群（5+ 节点）或 BanyanDB

---

### Q10: SkyWalking 与 Prometheus + Grafana 的关系是什么？

**答：**

SkyWalking 和 Prometheus + Grafana 可以互补使用：

1. **SkyWalking 导出 Metrics 给 Prometheus**：
   OAP 可以将 Metrics 以 Prometheus 格式暴露（端口 1234），供 Prometheus 抓取
   ```yaml
   telemetry:
     selector: prometheus
     prometheus:
       port: 1234
   ```

2. **Prometheus 数据在 Grafana 中展示**：
   Grafana 有官方的 SkyWalking 数据源插件，可以直接查询 OAP 的 GraphQL API

3. **典型架构**：
   - 基础设施 Metrics（CPU/内存/网络）：Prometheus + Node Exporter + Grafana
   - 应用 APM（Trace/业务 Metrics/Log）：SkyWalking
   - 统一 Dashboard：Grafana（同时对接 Prometheus 和 SkyWalking 数据源）

---

### Q11: 如何实现 SkyWalking OAP 的高可用部署？

**答：**

OAP 高可用需要以下几个层面：

1. **多实例部署**：部署 3+ 个 OAP 实例，前置负载均衡器（Nginx/K8s Service）

2. **集群协调**：配置集群注册中心
   ```yaml
   cluster:
     selector: nacos  # 或 zookeeper/etcd/consul
     nacos:
       serviceName: SkyWalking_OAP_Cluster
       hostPort: nacos-host:8848
   ```

3. **存储高可用**：使用 Elasticsearch 集群（3+ 节点，副本数 >= 1）

4. **健康检查**：
   ```bash
   # K8s 中配置健康检查
   livenessProbe:
     tcpSocket:
       port: 11800
     initialDelaySeconds: 60
   readinessProbe:
     httpGet:
       path: /
       port: 12800
     initialDelaySeconds: 30
   ```

5. **Agent 侧重试**：Agent 内置重试机制，OAP 短暂不可用时会缓冲数据

---

### Q12: 如何保护 SkyWalking 中的敏感数据？

**答：**

1. **SQL 参数脱敏**：
   ```properties
   # 禁止记录 SQL 参数（默认）
   plugin.mysql.trace_sql_parameters=false
   ```

2. **HTTP 参数过滤**：
   ```properties
   plugin.http.include_http_headers=X-Request-Id,Authorization
   # Authorization Header 不会被自动排除，需手动配置过滤
   ```

3. **OAP 访问控制**：
   - 在 Nginx 前置认证，UI 只对内网开放
   - 使用 OAP Token 认证（9.x 支持）

4. **ES 数据加密**：
   - 启用 Elasticsearch 的 TLS 传输加密
   - 配置 ES 角色权限，OAP 只有最小必要权限

5. **自定义数据脱敏**：
   ```java
   // 在自定义插件 Interceptor 中手动脱敏
   Tags.DB.DB_STATEMENT.set(span, maskSensitiveData(sql));

   private String maskSensitiveData(String sql) {
       // 替换手机号、身份证号等敏感字段
       return sql.replaceAll("'\\d{11}'", "'***'");
   }
   ```

---

### Q13: SkyWalking 的 Continuous Profiling 是什么？

**答：**

Continuous Profiling（持续性能剖析）是 SkyWalking 9.x 引入的功能，支持在**不影响业务的情况下**对 JVM 进行持续采样分析。

功能类型：
1. **CPU Profiling**：采集 CPU 热点方法，生成火焰图，定位计算密集型瓶颈
2. **Memory Profiling**：追踪内存分配，定位内存泄漏
3. **Off-CPU Profiling**：追踪线程等待/阻塞时间，定位 I/O 瓶颈

使用场景：
- Trace 显示某个方法耗时异常，但原因不明
- CPU 使用率高但无法确定具体方法
- 内存使用持续增长（内存泄漏排查）

触发方式：
1. **On-Demand**：在 UI 中手动选择服务实例，开启 15 分钟剖析
2. **Continuous**：基于 Metrics 阈值自动触发（如 CPU > 80% 时自动开启剖析）

---

### Q14: 如何处理 SkyWalking 数据量过大导致的性能问题？

**答：**

数据量控制策略：

1. **降低采样率**：将 `sample_n_per_3_secs` 从 -1 改为合理值（如 1000/3s）

2. **缩短 TTL**：
   - Trace/Log：从 7 天降为 3 天
   - Metrics：从 30 天降为 7 天

3. **忽略低价值请求**：
   ```properties
   # 忽略健康检查和静态资源
   agent.ignore_suffix=.jpg,.css,.js,.html
   agent.ignored_pathes=/health,/actuator/**,/favicon.ico
   ```

4. **减少 Metrics 指标**：注释掉不需要的 OAL 指标

5. **ES 冷热分离**：将旧数据迁移到冷节点（HDD），热数据留在 SSD

6. **增加 OAP 节点**：水平扩展 OAP 集群处理能力

7. **调整 ES 刷新间隔**：
   ```yaml
   bulkActions: 10000    # 增大批量写入数量
   flushInterval: 30     # 增大刷新间隔（秒）
   ```

---

### Q15: 如何编写 SkyWalking 自定义插件来监控私有框架？

**答：**

编写自定义插件的完整步骤：

1. **添加依赖**：
   ```xml
   <dependency>
       <groupId>org.apache.skywalking</groupId>
       <artifactId>apm-agent-core</artifactId>
       <version>9.1.0</version>
       <scope>provided</scope>
   </dependency>
   ```

2. **编写 Instrumentation 类**：继承 `ClassInstanceMethodsEnhancePluginDefine`，指定要增强的类和方法

3. **编写 Interceptor 类**：实现 `InstanceMethodsAroundInterceptor`，在 `beforeMethod`/`afterMethod`/`handleMethodException` 中操作 Span

4. **创建 SPI 描述文件**：`resources/skywalking-plugin.def`
   ```
   my-framework=com.example.plugin.MyFrameworkInstrumentation
   ```

5. **打包部署**：
   ```bash
   mvn clean package
   cp target/my-framework-plugin.jar /opt/skywalking-agent/plugins/
   # 重启应用即可生效
   ```

关键注意事项：
- 插件类不能依赖业务代码中的类（ClassLoader 隔离）
- `beforeMethod` 中如果抛出异常，会影响业务逻辑，必须 try-catch
- `stopSpan()` 必须在 `finally` 块中调用，否则 Span 泄漏会导致内存溢出
- Exit Span 的 `peer` 参数会影响服务拓扑图的展示

---

### Q16: SkyWalking 如何与 Spring Cloud 集成？有哪些注意事项？

**答：**

Spring Cloud 集成方案：

1. **Spring Cloud Gateway**：Agent 自动支持，会为每个路由请求创建 Entry Span，并在转发请求时创建 Exit Span，实现全链路追踪

2. **OpenFeign**：Agent 内置 Feign 插件，自动拦截 Feign 调用，在请求 Header 中注入 `sw8`

3. **Spring Cloud LoadBalancer / Ribbon**：Agent 自动处理负载均衡层，不影响追踪

4. **注意事项**：
   - 服务名（`agent.service_name`）要与 Spring Cloud 的 `spring.application.name` 保持一致
   - 在 K8s 环境中，实例名建议设为 Pod 名（便于在拓扑图中定位具体实例）
   - 使用 `Spring Cloud Sleuth` 时，需要关闭 Sleuth 的 Brave 集成，避免 `sw8` 和 Zipkin Header 冲突

5. **与 Sentinel 集成**：
   ```xml
   <!-- 引入 Sentinel + SkyWalking 桥接 -->
   <dependency>
       <groupId>com.alibaba.cloud</groupId>
       <artifactId>spring-cloud-alibaba-sentinel</artifactId>
   </dependency>
   <!-- 移动 sentinel 插件到 plugins 目录即可 -->
   ```

---

## 附录：SkyWalking 快速参考

### A.1 常用端口

| 端口 | 说明 |
|------|------|
| 11800 | OAP gRPC 端口（Agent 上报） |
| 12800 | OAP HTTP REST 端口（UI/API 查询） |
| 9411 | Zipkin 兼容 HTTP 端口 |
| 8080 | SkyWalking UI 端口 |
| 1234 | OAP Prometheus Metrics 端口 |

### A.2 重要文件

| 文件 | 说明 |
|------|------|
| `agent.config` | Agent 核心配置 |
| `config/alarm-settings.yml` | 告警规则配置 |
| `config/oal/*.oal` | OAL 指标计算脚本 |
| `config/application.yml` | OAP Server 配置 |
| `config/log4j2.xml` | OAP 自身日志配置 |

### A.3 常用环境变量

```bash
# Agent 端
SW_AGENT_NAME=my-service
SW_AGENT_COLLECTOR_BACKEND_SERVICES=oap:11800
SW_AGENT_SAMPLE=-1

# OAP 端
SW_STORAGE=elasticsearch
SW_STORAGE_ES_CLUSTER_NODES=es:9200
SW_CORE_RECORD_DATA_TTL=3
SW_CORE_METRICS_DATA_TTL=7
```

### A.4 SkyWalking GraphQL API 常用查询

```graphql
# 查询所有服务
{ getAllServices(duration: {start: "2026-07-08 0000", end: "2026-07-08 2359", step: DAY}) { id name } }

# 查询服务的端点列表
{ searchEndpoint(keyword: "/api/order", serviceId: "xxx", limit: 10) { id name } }

# 查询 Trace 列表
{ queryBasicTraces(condition: { serviceId: "xxx", queryDuration: {...}, minTraceDuration: 500, traceState: ERROR }) { traces { traceIds } } }
```

---

> **文档结束**  
> 本文档涵盖了 Apache SkyWalking 从基础原理到生产实践的完整内容。  
> 如需了解最新特性，请参考官方文档：https://skywalking.apache.org/docs/

