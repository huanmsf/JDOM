# Prometheus + Grafana 监控详解 - 从零到精通

> 版本：Prometheus 2.x | Grafana 10.x | 更新日期：2026年
> 覆盖：架构原理 / PromQL / Exporter / Alertmanager / Grafana / 实战案例 / 面试题

---

## 目录

- [Part 1: 监控体系概述](#part-1)
- [Part 2: Prometheus 架构与原理](#part-2)
- [Part 3: 数据模型与指标类型](#part-3)
- [Part 4: PromQL 查询语言深度解析](#part-4)
- [Part 5: Exporter 详解](#part-5)
- [Part 6: Spring Boot 集成 Prometheus](#part-6)
- [Part 7: Alertmanager 告警](#part-7)
- [Part 8: Grafana 深度使用](#part-8)
- [Part 9: 完整监控体系搭建](#part-9)
- [Part 10: Loki 日志系统](#part-10)
- [Part 11: 完整实战案例](#part-11)
- [Part 12: 常见面试题 FAQ](#part-12)

---

# Part 1: 监控体系概述 {#part-1}

## 1.1 为什么需要监控

在现代分布式系统中，监控是保障系统稳定运行的核心基础设施。没有监控，就如同在黑暗中驾车——
你不知道前方是否有危险，也无法在问题发生时快速定位和修复。

### 1.1.1 故障发现与快速响应

```
故障发现的三个层次：

+------------------+------------------+-----------------------+
|   被动感知        |   主动监控        |    预测性监控          |
|  （用户投诉）     |  （告警触发）     |  （趋势预测）          |
|                  |                  |                       |
|  发现延迟：分钟级  |  发现延迟：秒级   |  提前预警：小时/天级   |
|  影响：已有损失   |  影响：最小化     |  影响：零损失          |
+------------------+------------------+-----------------------+
```

**场景举例：**

- **故障发现**：系统接口P99延迟突然从50ms升至5000ms，监控系统在10秒内触发告警，工程师30秒内收到通知，5分钟内定位到数据库连接池耗尽问题并修复
- **无监控场景**：直到用户大量投诉才知道服务异常，此时已影响数万用户，MTTR（平均恢复时间）从分钟级变成小时级

### 1.1.2 容量规划

```
容量规划决策模型：

当前状态              历史趋势               预测
+----------+        +----------+          +------------------+
|CPU: 65%  |+分析-> |增长率:   |  +预测-> |30天后CPU: 89%    |
|内存: 72% |        |CPU +3%/周|          |需在20天内扩容    |
|磁盘: 58% |        |内存+2%/周|          |建议增加2台服务器  |
+----------+        +----------+          +------------------+
```

监控数据的积累可以帮助团队回答：
- 当前系统在多少并发下会出现性能瓶颈？
- 未来3个月需要扩容多少服务器？
- 哪些服务是性能热点？
- 用户增长10倍时系统还能支撑吗？

### 1.1.3 性能优化

性能优化需要数据驱动：

```
性能优化循环：

     +---------------------------------------------------+
     |                                                   |
     v                                                   |
  测量基线                                                |
  (P99, QPS, 错误率)                                      |
     |                                                   |
     v                                                   |
  定位瓶颈                                                |
  (火焰图, 慢查询, GC日志)                                 |
     |                                                   |
     v                                                   |
  实施优化                                                |
  (代码优化/配置调整/架构改进)                             |
     |                                                   |
     v                                                   |
  验证效果 --------------------------------------------------+
  (对比优化前后指标)
```

## 1.2 监控分层体系

现代云原生监控体系通常分为四个层次：

```
+-------------------------------------------------------------------+
|                        监控分层体系                               |
|                                                                   |
|  +---------------------------------------------------------+      |
|  |                    业务层监控                            |      |
|  |   订单量 | 支付成功率 | 用户注册量 | GMV | 转化率         |      |
|  +---------------------------------------------------------+      |
|                          ^                                        |
|  +---------------------------------------------------------+      |
|  |                   应用层监控                             |      |
|  |   接口QPS | 响应时间 | 错误率 | JVM堆 | 线程池 | 连接池  |      |
|  +---------------------------------------------------------+      |
|                          ^                                        |
|  +---------------------------------------------------------+      |
|  |                   中间件监控                             |      |
|  |   MySQL慢查询 | Redis命中率 | Kafka延迟 | ES索引性能     |      |
|  +---------------------------------------------------------+      |
|                          ^                                        |
|  +---------------------------------------------------------+      |
|  |                 基础设施监控                             |      |
|  |   CPU | 内存 | 磁盘IO | 网络带宽 | 容器资源 | K8s节点   |      |
|  +---------------------------------------------------------+      |
+-------------------------------------------------------------------+
```

### 1.2.1 基础设施监控

监控对象：物理机/虚拟机/容器/Kubernetes节点

| 指标类别 | 关键指标 | 工具 |
|---------|---------|------|
| CPU | 使用率、负载、steal时间 | node_exporter |
| 内存 | 使用量、swap、Page Cache | node_exporter |
| 磁盘 | IO等待、读写速率、使用率 | node_exporter |
| 网络 | 带宽、包速率、错误包 | node_exporter |
| 容器 | CPU/内存限制使用比 | cAdvisor |

### 1.2.2 中间件监控

| 中间件 | 核心指标 | Exporter |
|-------|---------|---------| 
| MySQL | QPS、慢查询、连接数、锁等待 | mysqld_exporter |
| Redis | 命中率、内存使用、连接数、延迟 | redis_exporter |
| Kafka | 消费延迟、分区Offset、Producer/Consumer TPS | kafka_exporter |
| Elasticsearch | 索引速率、查询延迟、集群健康 | elasticsearch_exporter |
| Nginx | 请求数、4xx/5xx率、连接数 | nginx_exporter |

### 1.2.3 应用层监控（RED方法）

**RED方法**是微服务监控的黄金标准：

```
RED方法三要素：

R - Rate（请求速率）
    每秒处理多少请求？
    PromQL: rate(http_requests_total[5m])

E - Errors（错误率）
    多少请求失败了？
    PromQL: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])

D - Duration（延迟分布）
    请求处理需要多长时间？
    PromQL: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

### 1.2.4 业务层监控

业务指标需要开发人员在代码中埋点实现：

```java
// 订单监控示例
meterRegistry.counter("order.created.total",
    "channel", "app", "status", "success").increment();

meterRegistry.gauge("order.amount.total",
    orderService, OrderService::getTodayAmount);
```

## 1.3 白盒监控 vs 黑盒监控

```
+------------------------+----------------------------------------+
|       白盒监控          |           黑盒监控                     |
|    (White-Box)         |         (Black-Box)                    |
+------------------------+----------------------------------------+
| 从内部暴露指标          | 从外部探测服务可用性                    |
|                        |                                        |
| * Prometheus metrics   | * HTTP 探测 (200状态码)                |
| * JVM metrics          | * TCP 端口探测                         |
| * 业务埋点指标          | * ICMP Ping                           |
|                        | * DNS 解析探测                         |
+------------------------+----------------------------------------+
| 优点：                  | 优点：                                 |
| * 指标丰富细粒度        | * 模拟用户真实体验                     |
| * 能发现内部问题        | * 不依赖被监控系统                     |
| * 适合根因分析          | * 跨数据中心可用性检测                 |
|                        |                                        |
| 缺点：                  | 缺点：                                 |
| * 需修改被监控系统      | * 信息粒度粗                           |
| * 内部问题用户无感知时  | * 发现问题滞后                         |
|   可能已触发            | * 无法发现内部性能问题                 |
+------------------------+----------------------------------------+
```

**实践建议**：白盒 + 黑盒监控互补使用

- 白盒监控发现**性能降级**（如P99从50ms变为500ms）
- 黑盒监控发现**完全不可用**（如服务宕机、网络故障）

## 1.4 Prometheus 生态全景图

```
+===========================================================================+
|                    Prometheus 监控生态全景图                               |
+===========================================================================+
|                                                                           |
|   数据来源层                                                               |
|   +---------+  +---------+  +---------+  +---------+  +---------+       |
|   |  Node   |  |  JMX    |  | Spring  |  | MySQL   |  |  K8s    |       |
|   |Exporter |  |Exporter |  |Actuator |  |Exporter |  |Exporter |       |
|   +----+----+  +----+----+  +----+----+  +----+----+  +----+----+       |
|        |            |            |            |            |              |
|        +------------+------------+------------+------------+              |
|                                 |                                         |
|                      Pull (HTTP Scrape)                                   |
|                                 |                                         |
|   +=================================+                                     |
|   |       Prometheus Server         |                                     |
|   |                                 |                                     |
|   |  +----------+  +----------+     |                                     |
|   |  | Retrieval|  |   TSDB   |     |                                     |
|   |  | (采集)   |->| (时序存储)|     |                                     |
|   |  +----------+  +----------+     |                                     |
|   |       |              |          |                                     |
|   |  +----v----+   +-----v-----+    |                                     |
|   |  |Rules    |   | HTTP API  |    |                                     |
|   |  |Evaluator|   |(查询接口) |    |                                     |
|   |  +---------+   +-----------+    |                                     |
|   +=====|=============|=============+                                     |
|         |             |                                                   |
|         v             v                                                   |
|   +-----+----+   +----+-------+                                          |
|   |Alertmanager|  |  Grafana  |                                           |
|   |路由/分组/抑制|  | Dashboard |                                           |
|   +-----+----+   +-----------+                                            |
|         |                                                                 |
|         v                                                                 |
|   +----------------------------------------+                            |
|   |  Email | 企业微信 | 钉钉 | Slack | PD  |                            |
|   +----------------------------------------+                            |
|                                                                           |
|   额外组件：                                                              |
|   +--------------+  +--------------+  +--------------------+            |
|   | Pushgateway  |  |  Thanos /    |  |   Loki (日志)      |            |
|   | (短期任务推送)|  |  Cortex      |  |   Tempo (链路追踪) |            |
|   +--------------+  |  (长期存储)  |  |   Pyroscope(剖析)  |            |
|                      +--------------+  +--------------------+            |
+===========================================================================+
```

## 1.5 Prometheus vs 其他监控方案对比

```
+----------------+--------------+--------------+--------------+--------------+
|   对比维度      |  Prometheus  |  InfluxDB    |   Zabbix     |    CAT       |
+----------------+--------------+--------------+--------------+--------------+
| 定位           | 指标监控+告警 | 时序数据库    | 传统IT监控   | 应用性能监控  |
| 数据模型       | 标签维度(多)  | 标签维度      | 模板/主机    | 事务/心跳    |
| 采集方式       | Pull为主      | Push/Pull     | Push为主     | SDK埋点      |
| 存储           | 本地TSDB      | 本地/集群     | MySQL/RRD    | MySQL/HBase  |
| 查询语言       | PromQL        | InfluxQL/Flux | 内置函数      | 内置         |
| 告警           | Alertmanager  | Kapacitor     | 内置完善      | 有限         |
| 可视化         | Grafana       | Chronograf    | 内置          | 内置         |
| 云原生支持     | 极强(K8s原生) | 较好          | 较弱          | 一般         |
| 水平扩展       | Thanos/Cortex | InfluxDB集群  | Proxy集群    | 分布式部署   |
| 学习曲线       | 中等          | 中等          | 低            | 低           |
| 社区活跃度     | 极高(CNCF)    | 高            | 高            | 中           |
| 适合场景       | 云原生/微服务  | IoT/时序分析  | 传统运维监控  | Java应用监控 |
| 开源/商业      | 完全开源       | 开源+商业版   | 开源          | 开源(美团)   |
+----------------+--------------+--------------+--------------+--------------+
```

**选型建议：**

| 场景 | 推荐方案 |
|------|---------|
| 云原生/K8s 微服务 | **Prometheus + Grafana**（首选）|
| 传统物理机/VM 运维 | Zabbix 或 Prometheus + Node Exporter |
| IoT 时序数据分析 | InfluxDB |
| Java 单体应用APM | CAT 或 SkyWalking |
| 全栈可观测性 | Prometheus + Grafana + Loki + Tempo |

---

# Part 2: Prometheus 架构与原理 {#part-2}

## 2.1 核心组件详解

### Prometheus 完整架构图

```
+==========================================================================+
|                     Prometheus 核心架构图                                |
|                                                                          |
|  +--------------------------------------------------------------------+ |
|  |                    被监控对象 (Targets)                            | |
|  |  +-----------+  +-----------+  +----------+  +------------+       | |
|  |  |Spring Boot|  |MySQL/Redis|  |Node Exprt|  | Batch Job  |       | |
|  |  | :8080/    |  | Exporter  |  | :9100/   |  | (批处理)   |       | |
|  |  | actuator/ |  |           |  | metrics  |  |            |       | |
|  |  +-----------+  +-----------+  +----------+  +-----+------+       | |
|  +-------+------------------+----------------+--------|---------------+ |
|          | HTTP Pull        |                |      Push               |
|          | /metrics         |                v                         |
|          +------------------+    +---------------------+               |
|                   |              |    Pushgateway      |               |
|                   |              | (短期任务指标缓冲)   |               |
|                   |              +----------+----------+               |
|                   |                         |                          |
|                   +-------------------------+                          |
|                                | Pull                                  |
|  +-----------------------------v----------------------------------+    |
|  |                    Prometheus Server                           |    |
|  |                                                                |    |
|  |  Retrieval: Service Discovery --> Scrape Manager              |    |
|  |             每15s拉取 /metrics                                  |    |
|  |                     |                                          |    |
|  |  TSDB: Head Block(内存+WAL) + Persistent Blocks(磁盘)         |    |
|  |                     |                                          |    |
|  |  HTTP API: /api/v1/query  /api/v1/query_range                 |    |
|  |                     |                                          |    |
|  |  Rules Evaluator: Recording Rules + Alerting Rules            |    |
|  +------+---------------------------------------------+----------+    |
|         |                                             |                |
|         v 推送告警                                    v 查询数据        |
|  +--------------+                          +--------------------+      |
|  | Alertmanager |                          |      Grafana       |      |
|  | 路由/分组/抑制 |                          | Dashboard/Alerting |      |
|  +------+-------+                          +--------------------+      |
|         |                                                               |
|         v                                                               |
|  +---------------------------------------------+                       |
|  | Email | 企业微信 | 钉钉 | Slack | PagerDuty |                       |
|  +---------------------------------------------+                       |
+==========================================================================+
```

### 2.1.1 Prometheus Server 三大核心模块

**1. Retrieval（数据检索/采集）**

```
采集流程：

prometheus.yml 配置
        |
        v
  scrape_configs 解析
        |
        v
  服务发现 (static/consul/k8s)
  获取 Target 列表
        |
        v
  Scrape Manager
  定时 HTTP GET /metrics
        |
        v
  文本格式解析 (OpenMetrics格式)
        |
        v
  写入 Head Block (内存)
```

**2. TSDB（时序数据库）** - 负责数据的持久化存储，详见 2.3 节

**3. HTTP API** - 提供 PromQL 查询接口

```bash
GET /api/v1/query?query=up&time=1609459200
GET /api/v1/query_range?query=rate(http_requests_total[5m])&start=...&end=...&step=60
GET /api/v1/series?match[]=http_requests_total
```

### 2.1.2 Exporter 工作原理

```
被监控系统          Exporter             Prometheus
     |                 |                     |
     | 本地API/文件     |                     |
     |--------------->|                     |
     |                 | Prometheus格式      |
     |                 | /metrics endpoint   |
     |                 |<----- HTTP GET -----|
     |                 |                     |
     |                 |--- 指标文本 ------->|
```

Exporter 暴露的指标格式示例：

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",handler="/api/users",status="200"} 1234
http_requests_total{method="POST",handler="/api/orders",status="201"} 456

# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0",mode="user"} 234.56
```

### 2.1.3 Pushgateway 使用场景

适用场景：**短生命周期任务**（批处理任务、定时任务）

```
+------------------+  Push   +------------------+  Pull   +-------------+
|   Batch Job      |-------->|   Pushgateway    |<--------|  Prometheus |
|  job执行后即退出  |         | 存储最后一次推送  |         |  Server     |
+------------------+         +------------------+         +-------------+
```

```bash
# Shell 脚本推送指标
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/batch_job/instance/server01
# TYPE batch_job_duration_seconds gauge
batch_job_duration_seconds 38.5
# TYPE batch_job_records_processed counter
batch_job_records_processed 9876
EOF
```

**注意事项：**
- 指标不会自动过期（需手动清理）
- 成为单点故障
- 尽量使用 Pull 模式，只在无法使用 Pull 时才用 Pushgateway

## 2.2 Pull vs Push 模型对比

```
+------------------------+--------------------------+--------------------------+
|       对比维度          |       Pull 模型           |       Push 模型           |
+------------------------+--------------------------+--------------------------+
| 控制权                 | 监控系统主动拉取           | 被监控系统主动推送         |
| 服务发现               | 需要知道所有Target地址     | 被监控系统知道监控中心即可 |
| 配置复杂度             | 集中管理（prometheus.yml） | 分散到每个被监控系统       |
| 健康检测               | Pull失败即代表Target不健康 | 需要额外心跳机制           |
| 适合短期任务           | 不适合（任务结束即消失）   | 适合                       |
| 采集频率控制           | 由监控系统统一控制         | 由被监控系统自行决定       |
+------------------------+--------------------------+--------------------------+
| 代表系统               | Prometheus               | StatsD/InfluxDB/OpenTSDB |
| 适合场景               | 服务监控/K8s环境           | IoT/边缘计算/短期任务      |
+------------------------+--------------------------+--------------------------+
```

**Prometheus 选择 Pull 的核心原因：**
1. 易于调试：可以直接 curl 访问 `/metrics` 验证指标
2. 集中管控：所有采集配置集中在 prometheus.yml
3. 健康检测：Pull 失败就是 down，无需额外实现

## 2.3 TSDB 时序数据库存储原理

### 2.3.1 磁盘目录结构

```
/prometheus/data/
+-- 01BKGV7JBM69T2G1BGBGM6KB12/    <- 持久化 Block（2小时数据）
|   +-- chunks/
|   |   +-- 000001                  <- 原始数据文件(128MB)
|   |   +-- 000002
|   +-- index                       <- 倒排索引文件
|   +-- tombstones                  <- 逻辑删除标记
|   +-- meta.json                   <- Block元数据
+-- chunks_head/                    <- Head Block 内存映射
|   +-- 000001
+-- wal/                            <- Write-Ahead Log
|   +-- 00000001                    <- WAL segment
|   +-- checkpoint.000001/
+-- lock                            <- 进程锁文件
```

### 2.3.2 Head Block 与 WAL 机制

```
Head Block 工作机制：

新采集数据
     |
     v
 WAL (Write-Ahead Log)  <- 先写WAL，防止崩溃数据丢失
     |
     v
 Head Block (内存中)
 +-------------------------------------------+
 |  内存中保存最近2小时的数据                 |
 |  chunks_head/ (mmap 内存映射到磁盘)       |
 +-------------------------------------------+
     |
     | 每2小时触发一次 compaction
     v
 写入持久化 Block（磁盘）
```

### 2.3.3 Gorilla 压缩算法

```
时间戳压缩（Delta-of-Delta编码）：

原始时间戳：  T1=1609459200000, T2=1609459215000, T3=1609459230000
Delta:        delta1=15000, delta2=15000, delta3=15000
Delta-Delta:  dd1=0, dd2=0, dd3=0   <- 全是0！只需1bit存储

值压缩（XOR编码）：
  V1 = 12.34 (存储完整值)
  V2 = 12.35 (与V1做XOR，只存储变化的bit位)
```

**压缩效果**：平均每个样本只需 **1.37字节**（原始16字节，压缩率 ~11.6倍）

### 2.3.4 Block 压缩层级

```
Level 1: 原始 Block (2小时)
  [00:00-02:00] [02:00-04:00] [04:00-06:00] ... [22:00-24:00]

Level 2: 合并压缩 (6小时)
  [00:00-06:00]  [06:00-12:00]  [12:00-18:00]  [18:00-24:00]

Level 3: 继续合并 (约2天)
  [day1]  [day2]  ...

Level 4: 更大的合并块（周级别）
  [week1]  [week2]  ...
```

### 2.3.5 数据保留策略

```yaml
# Prometheus 启动参数
--storage.tsdb.retention.time=15d      # 时间保留（默认15天）
--storage.tsdb.retention.size=50GB     # 磁盘大小保留
--storage.tsdb.path=/prometheus/data   # 数据目录
--storage.tsdb.wal-compression         # 开启WAL压缩（推荐）
```

## 2.4 服务发现（六种方式）

### 2.4.1 静态配置（static_configs）

```yaml
scrape_configs:
  - job_name: 'spring-boot-app'
    static_configs:
      - targets:
          - '192.168.1.100:8080'
          - '192.168.1.101:8080'
        labels:
          env: 'production'
          region: 'cn-north'
```

### 2.4.2 文件服务发现（file_sd_configs）

```yaml
scrape_configs:
  - job_name: 'services'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/*.json'
        refresh_interval: 30s
```

```json
[
  {
    "targets": ["192.168.1.100:8080", "192.168.1.101:8080"],
    "labels": { "job": "web-api", "env": "production" }
  }
]
```

### 2.4.3 DNS 服务发现（dns_sd_configs）

```yaml
scrape_configs:
  - job_name: 'dns-services'
    dns_sd_configs:
      - names:
          - '_prometheus._tcp.example.com'
        type: SRV
        refresh_interval: 30s
```

### 2.4.4 Consul 服务发现

```yaml
scrape_configs:
  - job_name: 'consul-services'
    consul_sd_configs:
      - server: 'consul:8500'
        services: ['spring-boot-app', 'payment-service']
    relabel_configs:
      - source_labels: [__meta_consul_service]
        target_label: service
      - source_labels: [__meta_consul_service_address, __meta_consul_service_port]
        separator: ':'
        target_label: __address__
```

### 2.4.5 Kubernetes 服务发现

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)
      - source_labels: [__meta_kubernetes_namespace]
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: kubernetes_pod_name
```

### 2.4.6 EC2 服务发现

```yaml
scrape_configs:
  - job_name: 'ec2-instances'
    ec2_sd_configs:
      - region: us-east-1
        access_key: YOUR_ACCESS_KEY
        secret_key: YOUR_SECRET_KEY
        port: 9100
    relabel_configs:
      - source_labels: [__meta_ec2_tag_Name]
        target_label: instance_name
      - source_labels: [__meta_ec2_tag_Environment]
        target_label: env
```

### 2.4.7 Relabeling 完整操作参考

```yaml
relabel_configs:
  # keep - 只保留符合正则的 Target
  - source_labels: [__meta_kubernetes_pod_ready]
    action: keep
    regex: 'true'

  # drop - 丢弃符合正则的 Target
  - source_labels: [__meta_kubernetes_namespace]
    action: drop
    regex: 'kube-system'

  # replace - 替换标签值（最常用）
  - source_labels: [__meta_kubernetes_pod_name]
    action: replace
    target_label: pod
    regex: '(.*)'
    replacement: '${1}'

  # labelmap - 批量映射标签
  - action: labelmap
    regex: __meta_kubernetes_pod_label_(.+)

  # labeldrop - 删除匹配的标签
  - action: labeldrop
    regex: __meta_kubernetes_pod_label_pod_template_hash

  # labelkeep - 只保留匹配的标签
  - action: labelkeep
    regex: 'job|instance|env|service'
```


---

# Part 3: 数据模型与指标类型 {#part-3}

## 3.1 时间序列数据模型

```
时间序列表示格式：

<metric_name>{<label_name>=<label_value>, ...}

示例：
  http_requests_total{method="GET", handler="/api/users", status="200",
                      job="web-api", instance="192.168.1.100:8080"}

时间序列随时间的样本点：
  http_requests_total{method="GET"}  @  1609459200000  = 1234
  http_requests_total{method="GET"}  @  1609459215000  = 1267
  http_requests_total{method="GET"}  @  1609459230000  = 1298
```

### 3.1.1 指标命名规范

```
格式：<namespace>_<subsystem>_<name>_<unit>

好的命名：
  http_requests_total            (HTTP请求总数)
  http_request_duration_seconds  (HTTP请求耗时，用秒)
  jvm_memory_used_bytes          (JVM内存，用字节)
  node_disk_read_bytes_total     (磁盘读取字节总量)
  mysql_connections_active       (MySQL活跃连接数)

不好的命名：
  requests         (太通用，无命名空间)
  http_req_ms      (时间用秒，不用ms)
  getOrders        (驼峰，应用下划线)
  my-app-requests  (不能含中划线)
```

### 3.1.2 标签的作用

```
标签使单个指标名称可以代表多个维度：

无标签：http_requests_total  =  总请求数（1条时间序列）

有标签（多条时间序列）：
  http_requests_total{method="GET"}          = GET请求数
  http_requests_total{method="POST"}         = POST请求数
  http_requests_total{status="200"}          = 成功请求数
  http_requests_total{handler="/api/users"}  = /api/users的请求数
  http_requests_total{method="GET", status="200", handler="/api/users"}

内置标签（Prometheus自动添加）：
  job      = scrape_configs 中配置的 job_name
  instance = 采集的目标地址（host:port）
```

## 3.2 四种指标类型

### 3.2.1 Counter（计数器）

**特点**：只增不减，服务重启归零。

**命名**：通常以 `_total` 结尾

**适用**：请求总数、错误总数、字节总量

```
Counter 示意图：

值 |                                      /
   |                         /------------
   |             /------------
   | /------------
   | |
   +-+------------------------------------> 时间
     ^ 服务重启（归零）
```

```go
// Go 语言
requestsTotal := prometheus.NewCounterVec(
    prometheus.CounterOpts{Name: "http_requests_total", Help: "Total HTTP requests"},
    []string{"method", "handler", "status"},
)
requestsTotal.WithLabelValues("GET", "/api/users", "200").Inc()
```

```java
// Java Micrometer
Counter.builder("http.requests.total")
    .tags("method", "GET", "status", "200")
    .register(meterRegistry)
    .increment();
```

### 3.2.2 Gauge（仪表盘）

**特点**：可任意增减的瞬时值。

**适用**：CPU使用率、内存使用量、当前连接数、队列长度

```
Gauge 示意图：

值 |    /---\
   | /--+   +--\    /--\
   |/             \--+  \-+
   +-------------------------> 时间
```

```go
// Go 语言
memoryGauge := prometheus.NewGaugeVec(
    prometheus.GaugeOpts{Name: "jvm_memory_used_bytes", Help: "JVM memory"},
    []string{"area"},
)
memoryGauge.WithLabelValues("heap").Set(52428800)
memoryGauge.WithLabelValues("heap").Add(1024)
memoryGauge.WithLabelValues("heap").Sub(512)
```

### 3.2.3 Histogram（直方图）

**特点**：将观测值分布到预定义 bucket，服务端可聚合计算分位数。

**适用**：响应时间分布、请求体大小分布

```
Histogram 三个指标系列：

http_request_duration_seconds_bucket{le="0.005"}  = 1234
http_request_duration_seconds_bucket{le="0.01"}   = 2345
http_request_duration_seconds_bucket{le="0.025"}  = 3456
http_request_duration_seconds_bucket{le="0.05"}   = 4567
http_request_duration_seconds_bucket{le="0.1"}    = 5678
http_request_duration_seconds_bucket{le="0.25"}   = 6789
http_request_duration_seconds_bucket{le="0.5"}    = 7890
http_request_duration_seconds_bucket{le="1"}      = 8901
http_request_duration_seconds_bucket{le="+Inf"}   = 9200
http_request_duration_seconds_sum   = 461.32  (耗时总和/秒)
http_request_duration_seconds_count = 9200    (总请求数)

计算分位数：
  P99 = histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
  P95 = histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))
  P50 = histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))
```

```go
// Go 语言
requestDuration := prometheus.NewHistogramVec(
    prometheus.HistogramOpts{
        Name:    "http_request_duration_seconds",
        Help:    "HTTP request duration",
        Buckets: []float64{0.001, 0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10},
    },
    []string{"method", "handler"},
)
requestDuration.WithLabelValues("GET", "/api/users").Observe(0.054)
```

### 3.2.4 Summary（摘要）

**特点**：客户端直接计算分位数。

```
Summary vs Histogram 对比：

+-----------------+----------------------+---------------------+
|    对比维度      |      Summary         |      Histogram      |
+-----------------+----------------------+---------------------+
| 分位数计算       | 客户端               | 服务端（PromQL）    |
| 精度             | 精确                 | 受bucket边界影响    |
| 跨实例聚合       | 不能聚合             | 可以聚合（最大优势）|
| 存储开销         | 较大                 | 较小               |
| 推荐程度         | 一般                 | ★★★★★ 强烈推荐    |
+-----------------+----------------------+---------------------+

结论：多实例场景一律使用 Histogram！
```

```
Summary 暴露格式：
rpc_duration_seconds{quantile="0.5"}  0.012
rpc_duration_seconds{quantile="0.9"}  0.056
rpc_duration_seconds{quantile="0.99"} 0.234
rpc_duration_seconds_sum   45.67
rpc_duration_seconds_count 1000
```

## 3.3 标签设计原则

### 3.3.1 基数（Cardinality）控制

```
高基数 = Prometheus 性能杀手！

低基数（安全）：                    高基数（危险）：
  status: 5种取值                     user_id: 百万种取值
  method: 4种取值                     order_id: 无限种取值
  env:    3种取值                     request_id: UUID无限种

后果：
  100万用户 x 10接口 = 1000万个时间序列
  -> Prometheus OOM / 性能严重下降

规则：
  * 每个指标不超过 5-7 个标签
  * 单实例建议不超过 1000万 个活跃时间序列
  * 高基数信息（用户ID/请求ID）放日志/链路追踪
```

---

# Part 4: PromQL 查询语言深度解析 {#part-4}

## 4.1 数据类型

```
1. 瞬时向量（Instant Vector）：某时刻所有匹配时间序列的最新值集合
   http_requests_total

2. 区间向量（Range Vector）：某时间范围内的所有样本点
   http_requests_total[5m]

3. 标量（Scalar）：单个浮点数
   scalar(sum(http_requests_total))

4. 字符串（String）：字符串值（较少使用）
```

## 4.2 选择器（标签过滤）

```promql
http_requests_total{method="GET"}               # 精确匹配
http_requests_total{status!="200"}              # 不等于
http_requests_total{status=~"2.."}              # 正则匹配（2xx成功）
http_requests_total{status=~"4..|5.."}          # 4xx或5xx错误
http_requests_total{handler=~"/api/.*"}          # /api/开头
http_requests_total{handler!~"/health|/ping"}    # 排除健康检查
http_requests_total{method="GET", status="200"}  # 组合条件（AND）
```

## 4.3 时间范围与位移

```promql
# 时间单位：ms s m h d w y

http_requests_total[5m]      # 过去5分钟的区间向量
http_requests_total[1h]      # 过去1小时
http_requests_total offset 1h  # 1小时前的瞬时值

# 环比（当前与1小时前对比）
rate(http_requests_total[5m]) / rate(http_requests_total[5m] offset 1h)
```

## 4.4 聚合运算符

```promql
sum(http_requests_total) by (job)              # 按job分组求和
sum(http_requests_total) without (instance)    # 除instance外分组
avg(node_cpu_usage_percent) by (instance)      # 每实例平均CPU
max(jvm_memory_used_bytes) by (instance)       # 每实例最大内存
count(up == 1)                                  # 在线实例数
topk(5, sum(rate(http_requests_total[5m])) by (handler))  # QPS最高5接口
bottomk(3, sum(rate(http_requests_total[5m])) by (instance))
quantile(0.99, http_request_duration_seconds_sum)
```

## 4.5 核心函数

### rate / irate（Counter增长率）

```promql
rate(http_requests_total[5m])   # 5分钟平均每秒QPS（推荐告警用）
irate(http_requests_total[5m])  # 瞬时QPS（推荐实时图表用）
```

### increase（区间增量）

```promql
increase(http_requests_total[1h])   # 过去1小时总请求量
increase(http_requests_total[1m])   # 每分钟请求量
```

### predict_linear（线性预测）

```promql
# 预测4小时后磁盘是否耗尽（<0表示会耗尽）
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600) < 0
```

### histogram_quantile（分位数计算）

```promql
# P99响应时间
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# 按服务分组P99（必须保留le标签！）
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))
```

### absent / absent_over_time（缺失检测）

```promql
absent(up{job="web-api"})                         # 指标消失时返回1
absent_over_time(http_requests_total{job="web-api"}[5m])  # 5分钟无数据
```

### 标签操作

```promql
# 从 instance 提取主机名
label_replace(http_requests_total, "hostname", "$1", "instance", "([^:]+):.*")

# 拼接标签
label_join(up, "target", ":", "job", "instance")
```

### 其他常用函数

```promql
changes(kube_deployment_spec_replicas[1h])  # 变化次数
resets(http_requests_total[1h])             # Counter重置次数
floor(v) / ceil(v) / round(v, 0.1)         # 取整
clamp(v, 0, 100)                            # 限制范围
sort(v) / sort_desc(v)                      # 排序
time()                                       # 当前Unix时间戳

# 时间段告警（只在工作时间触发）
ALERTS{alertname="HighCPU"} and on() (hour() >= 9 and hour() < 18)
```

## 4.6 二元运算符

```promql
# 算术运算
node_memory_MemUsed_bytes / 1024 / 1024 / 1024   # 字节转GB
(MemTotal - MemFree) / MemTotal * 100              # 内存使用率

# 比较运算（过滤）
node_cpu_usage_percent > 80                        # 过滤CPU>80%的实例
node_cpu_usage_percent > bool 80                   # 返回bool值(0/1)

# 集合运算
node_cpu_usage > 80 and node_memory_usage > 90     # 同时满足
node_cpu_usage > 80 or node_memory_usage > 90      # 满足其一
node_cpu_usage > 80 unless node_memory_usage > 90  # 差集

# 向量匹配（标签集不同时）
rate(http_errors_total[5m])
  / on(job, instance) group_left(env, region)
  rate(http_requests_total[5m])
```

## 4.7 实用 PromQL 查询大全（30+）

### HTTP 服务监控

```promql
# QPS（整体/按服务/Top10接口）
sum(rate(http_requests_total[5m]))
sum(rate(http_requests_total[5m])) by (job)
topk(10, sum(rate(http_requests_total[5m])) by (handler))

# 5xx错误率（百分比）
sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
  / sum(rate(http_requests_total[5m])) by (job) * 100

# 4xx+5xx总错误率
sum(rate(http_requests_total{status=~"[45].."}[5m])) by (job)
  / sum(rate(http_requests_total[5m])) by (job)

# P99延迟（全局/按服务）
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job))

# 平均响应时间
sum(rate(http_request_duration_seconds_sum[5m])) by (job)
  / sum(rate(http_request_duration_seconds_count[5m])) by (job)
```

### JVM 监控

```promql
# 堆内存使用率
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100

# GC次数速率
rate(jvm_gc_pause_seconds_count[5m])

# GC耗时占比
rate(jvm_gc_pause_seconds_sum[5m]) * 100

# GC暂停P99
histogram_quantile(0.99, rate(jvm_gc_pause_seconds_bucket[5m]))

# 线程状态分布
jvm_threads_states_threads{state="blocked"}
jvm_threads_states_threads{state="waiting"}
```

### 基础设施监控

```promql
# CPU使用率
(1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100

# 内存使用率
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 磁盘使用率
(node_filesystem_size_bytes - node_filesystem_avail_bytes)
  / node_filesystem_size_bytes{fstype!~"tmpfs|devtmpfs"} * 100

# 磁盘预测（4小时后）
predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600) < 0

# 网络接收速率（Mbps）
rate(node_network_receive_bytes_total{device!~"lo|veth.*"}[5m]) * 8 / 1024 / 1024

# 系统负载/核数比值
node_load1 / count(count(node_cpu_seconds_total) by (cpu))
```

### 业务指标

```promql
# 每分钟下单量
increase(order_created_total[1m])

# 支付成功率
sum(rate(payment_total{status="success"}[5m]))
  / sum(rate(payment_total[5m])) * 100

# 在线实例数 / SLA
count(up{job="web-api"} == 1)
avg_over_time(up{job="web-api"}[1h]) * 100

# 连接池使用率
hikaricp_connections_active / hikaricp_connections_max * 100
```

---

# Part 5: Exporter 详解 {#part-5}

## 5.1 Node Exporter（主机指标）

```bash
# Docker 安装
docker run -d --name node_exporter --pid host --network host \
  -v /proc:/host/proc:ro -v /sys:/host/sys:ro -v /:/rootfs:ro \
  prom/node-exporter:latest \
  --path.procfs=/host/proc --path.sysfs=/host/sys --path.rootfs=/rootfs

# systemd 安装
cat > /etc/systemd/system/node_exporter.service << 'EOF'
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=node_exporter
ExecStart=/usr/local/bin/node_exporter \
  --collector.systemd --collector.processes --web.listen-address=:9100
Restart=always
[Install]
WantedBy=multi-user.target
EOF
systemctl enable --now node_exporter
```

**核心指标：**

```
CPU:     node_cpu_seconds_total{mode}  (idle/user/system/iowait/steal)
内存:    node_memory_MemTotal_bytes / MemFree / MemAvailable / Buffers / Cached
磁盘IO:  node_disk_read_bytes_total / written_bytes_total / io_time_seconds_total
文件系统: node_filesystem_size_bytes / avail_bytes / free_bytes
网络:    node_network_receive_bytes_total / transmit_bytes_total / errs
系统:    node_load1/5/15  node_boot_time_seconds  node_procs_running
```

## 5.2 JMX Exporter（JVM 指标）

```yaml
# jmx_config.yaml
startDelaySeconds: 0
lowercaseOutputName: true
rules:
  - pattern: 'java.lang<type=Memory><>(\w+)MemoryUsage: (\w+): (\d+)'
    name: jvm_memory_$1_$2_bytes
  - pattern: 'java.lang<type=GarbageCollector, name=(.+)><>CollectionCount'
    name: jvm_gc_collection_count_total
    labels:
      gc: "$1"
  - pattern: 'java.lang<type=Threading><>ThreadCount'
    name: jvm_threads_count
```

```bash
# 以 Java Agent 方式启动
java -javaagent:/path/to/jmx_prometheus_javaagent-0.20.0.jar=9090:/path/to/jmx_config.yaml \
     -jar your-application.jar
```

## 5.3 Spring Boot Actuator + Micrometer

### 依赖配置（pom.xml）

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<!-- AOP 支持 @Timed 注解 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

### application.yml 完整配置

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "prometheus,health,info,metrics"
  endpoint:
    prometheus:
      enabled: true
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      # 全局公共标签（所有指标都会带这些标签）
      application: ${spring.application.name}
      env: ${spring.profiles.active:default}
      region: cn-north-1
    distribution:
      percentiles-histogram:
        http.server.requests: true  # 开启Histogram，支持histogram_quantile
      percentiles:
        http.server.requests: 0.5, 0.75, 0.95, 0.99
      slo:
        http.server.requests: 50ms, 100ms, 200ms, 500ms, 1s, 2s
```

### Spring Boot 内置指标

```
HTTP 请求：
  http_server_requests_seconds{method, status, uri, exception}

JVM 内存：
  jvm_memory_used_bytes{area, id}
  jvm_memory_max_bytes{area, id}
  jvm_memory_committed_bytes{area, id}

JVM GC：
  jvm_gc_pause_seconds{action, cause}  (Histogram)
  jvm_gc_memory_allocated_bytes_total
  jvm_gc_memory_promoted_bytes_total

JVM 线程：
  jvm_threads_live_threads
  jvm_threads_daemon_threads
  jvm_threads_peak_threads
  jvm_threads_states_threads{state}

JVM 类加载：
  jvm_classes_loaded_classes
  jvm_classes_unloaded_classes_total

HikariCP 连接池：
  hikaricp_connections_active{pool}
  hikaricp_connections_idle{pool}
  hikaricp_connections_pending{pool}
  hikaricp_connections_timeout_total{pool}
  hikaricp_connections_acquire_seconds{pool}  (Histogram)

Tomcat：
  tomcat_sessions_active_current_sessions
  tomcat_connections_current_connections
  tomcat_threads_busy_threads
```

## 5.4 MySQL Exporter

```bash
# Docker 部署
docker run -d --name mysqld_exporter -p 9104:9104 \
  -e DATA_SOURCE_NAME="exporter:password@(mysql:3306)/" \
  prom/mysqld-exporter:latest \
  --collect.info_schema.innodb_metrics \
  --collect.perf_schema.eventsstatements \
  --collect.slave_status

# MySQL 权限配置
CREATE USER 'exporter'@'%' IDENTIFIED BY 'StrongPassword123!';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
FLUSH PRIVILEGES;
```

```promql
# MySQL 核心监控 PromQL
rate(mysql_global_status_queries[5m])                                          # QPS
rate(mysql_global_status_slow_queries[5m])                                     # 慢查询速率
mysql_global_status_threads_connected / mysql_global_variables_max_connections # 连接使用率
(1 - rate(mysql_global_status_innodb_buffer_pool_reads[5m]) /
    rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])) * 100      # 缓冲池命中率
mysql_slave_status_seconds_behind_master                                        # 复制延迟
rate(mysql_global_status_table_locks_waited[5m])                               # 锁等待
```

## 5.5 Redis Exporter

```bash
docker run -d --name redis_exporter -p 9121:9121 \
  oliver006/redis_exporter \
  --redis.addr=redis://redis:6379 --redis.password=yourpassword
```

```promql
# Redis 核心监控 PromQL
rate(redis_commands_processed_total[5m])                                        # QPS
redis_memory_used_bytes / 1024 / 1024                                          # 内存(MB)
redis_memory_used_bytes / redis_memory_max_bytes * 100                         # 内存使用率
redis_connected_clients                                                         # 连接数
rate(redis_keyspace_hits_total[5m]) /
  (rate(redis_keyspace_hits_total[5m]) + rate(redis_keyspace_misses_total[5m])) * 100  # 命中率
rate(redis_evicted_keys_total[5m])                                             # 驱逐键速率
```

## 5.6 Kafka Exporter

```bash
docker run -d --name kafka_exporter -p 9308:9308 \
  danielqsj/kafka-exporter:latest \
  --kafka.server=kafka1:9092 --kafka.server=kafka2:9092 --kafka.server=kafka3:9092
```

```promql
# Kafka 核心监控 PromQL
kafka_consumergroup_lag{consumergroup="my-group", topic="orders"}              # 消费延迟(Lag)
sum(kafka_consumergroup_lag{consumergroup="my-group"}) by (topic)              # 总Lag
rate(kafka_topic_partition_current_offset[5m])                                 # 生产速率
count(kafka_broker_info)                                                        # Broker数量
```

## 5.7 Blackbox Exporter（黑盒探测）

```yaml
# blackbox.yml - 配置文件
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []  # 空=允许所有2xx
      follow_redirects: true

  https_check_cert:
    prober: http
    timeout: 5s
    http:
      fail_if_not_ssl: true

  tcp_connect:
    prober: tcp
    timeout: 5s

  icmp_ping:
    prober: icmp
    timeout: 5s
```

```yaml
# prometheus.yml - Blackbox 采集配置
scrape_configs:
  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - 'https://www.example.com'
          - 'http://internal-service:8080/health'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

```promql
# Blackbox 核心指标
probe_success{job="blackbox_http"}                          # 探测成功(1)/失败(0)
probe_http_status_code                                       # HTTP状态码
probe_duration_seconds                                       # 响应时间
(probe_ssl_earliest_cert_expiry - time()) / 86400           # 证书剩余天数
(probe_ssl_earliest_cert_expiry - time()) / 86400 < 30      # 证书30天内过期告警
```

## 5.8 自定义 Exporter

### Python 实现

```python
#!/usr/bin/env python3
"""自定义业务指标 Exporter"""
import time
import mysql.connector
from prometheus_client import start_http_server, Gauge, Counter, Histogram, CollectorRegistry
import logging

logger = logging.getLogger(__name__)
registry = CollectorRegistry()

# 定义指标
PENDING_ORDERS = Gauge('business_pending_orders', '待处理订单数', ['order_type'], registry=registry)
TODAY_AMOUNT = Gauge('business_today_amount_yuan', '今日订单总额', registry=registry)
DB_DURATION = Histogram('business_db_query_seconds', '数据库查询耗时',
    ['query_name'], buckets=[0.01, 0.05, 0.1, 0.5, 1.0], registry=registry)
DB_ERRORS = Counter('business_db_errors_total', 'DB错误次数', ['query_name'], registry=registry)

def collect_metrics(db_config):
    """采集所有业务指标"""
    try:
        conn = mysql.connector.connect(**db_config)
        cursor = conn.cursor()

        start = time.time()
        try:
            cursor.execute("SELECT order_type, COUNT(*) FROM orders WHERE status='PENDING' GROUP BY order_type")
            for order_type, count in cursor.fetchall():
                PENDING_ORDERS.labels(order_type=order_type).set(count)
        except Exception as e:
            DB_ERRORS.labels(query_name="pending_orders").inc()
        finally:
            DB_DURATION.labels(query_name="pending_orders").observe(time.time() - start)

        start = time.time()
        try:
            cursor.execute("SELECT COALESCE(SUM(amount), 0) FROM orders WHERE DATE(created_at)=CURDATE()")
            result = cursor.fetchone()
            TODAY_AMOUNT.set(float(result[0]) if result else 0)
        except Exception as e:
            DB_ERRORS.labels(query_name="today_amount").inc()
        finally:
            DB_DURATION.labels(query_name="today_amount").observe(time.time() - start)

        cursor.close()
        conn.close()
    except Exception as e:
        logger.error(f"DB connection failed: {e}")

if __name__ == '__main__':
    db_config = {'host': 'localhost', 'port': 3306,
                 'user': 'metrics_reader', 'password': 'password', 'database': 'business_db'}
    start_http_server(9200, registry=registry)
    logger.info("Exporter started on :9200")
    while True:
        collect_metrics(db_config)
        time.sleep(30)
```

### Java Micrometer 完整自定义指标代码

```java
package com.example.metrics;

import io.micrometer.core.instrument.*;
import io.micrometer.core.instrument.binder.MeterBinder;
import org.springframework.stereotype.Component;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.function.Supplier;

/**
 * 完整的 Micrometer 自定义指标示例
 * Counter / Gauge / Timer / DistributionSummary
 */
@Component
public class BusinessMetrics implements MeterBinder {

    // Counter（计数器）- 单调递增
    private Counter orderCreatedCounter;
    private Counter paymentSuccessCounter;
    private Counter paymentFailedCounter;

    // Gauge（仪表盘）- 实时状态值
    private final AtomicInteger pendingOrderCount = new AtomicInteger(0);

    // Timer（计时器）- 带Histogram的耗时统计
    private Timer orderProcessingTimer;
    private Timer paymentTimer;

    // DistributionSummary（分布摘要）
    private DistributionSummary orderAmountSummary;

    @Override
    public void bindTo(MeterRegistry registry) {
        // 注册 Counter
        orderCreatedCounter = Counter.builder("order.created.total")
            .description("订单创建总数")
            .tag("channel", "web")
            .register(registry);

        paymentSuccessCounter = Counter.builder("payment.total")
            .tag("status", "success").register(registry);

        paymentFailedCounter = Counter.builder("payment.total")
            .tag("status", "failed").register(registry);

        // 注册 Gauge（通过 AtomicInteger）
        Gauge.builder("order.pending.count", pendingOrderCount, AtomicInteger::get)
            .description("待处理订单数").register(registry);

        // 注册 Gauge（通过自定义 Supplier）
        Gauge.builder("jvm.custom.memory", Runtime.getRuntime(),
                r -> (double)(r.totalMemory() - r.freeMemory()))
            .description("自定义JVM内存统计").baseUnit("bytes").register(registry);

        // 注册 Timer（开启Histogram，支持P99计算）
        orderProcessingTimer = Timer.builder("order.processing.time")
            .description("订单处理耗时")
            .publishPercentileHistogram(true)
            .serviceLevelObjectives(
                java.time.Duration.ofMillis(100),
                java.time.Duration.ofMillis(500),
                java.time.Duration.ofSeconds(1),
                java.time.Duration.ofSeconds(2))
            .publishPercentiles(0.5, 0.75, 0.95, 0.99)
            .register(registry);

        paymentTimer = Timer.builder("payment.processing.time")
            .description("支付处理耗时")
            .publishPercentileHistogram(true)
            .register(registry);

        // 注册 DistributionSummary（金额分布）
        orderAmountSummary = DistributionSummary.builder("order.amount.distribution")
            .description("订单金额分布（元）")
            .baseUnit("yuan")
            .serviceLevelObjectives(10, 50, 100, 500, 1000, 5000)
            .publishPercentileHistogram(true)
            .register(registry);
    }

    // 业务埋点方法

    public void recordOrderCreated(double amount) {
        orderCreatedCounter.increment();
        orderAmountSummary.record(amount);
        pendingOrderCount.incrementAndGet();
    }

    public void recordOrderComplete() {
        pendingOrderCount.decrementAndGet();
    }

    public <T> T recordOrderProcessing(Supplier<T> orderLogic) {
        return orderProcessingTimer.record(orderLogic);
    }

    public void recordPayment(boolean success, long durationMs) {
        (success ? paymentSuccessCounter : paymentFailedCounter).increment();
        paymentTimer.record(durationMs, java.util.concurrent.TimeUnit.MILLISECONDS);
    }

    public void updatePendingOrders(int count) {
        pendingOrderCount.set(count);
    }
}
```

### @Timed 注解使用

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Timed(
        value = "order.create.time",
        description = "创建订单耗时",
        extraTags = {"type", "standard"},
        histogram = true,
        percentiles = {0.5, 0.95, 0.99}
    )
    @PostMapping
    public OrderResponse createOrder(@RequestBody CreateOrderRequest request) {
        // 自动计时
        return orderService.createOrder(request);
    }
}

// 必须注册 TimedAspect
@Configuration
public class MetricsConfig {

    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags(
            @Value("${spring.application.name}") String appName,
            @Value("${spring.profiles.active:default}") String env) {
        return registry -> registry.config()
            .commonTags("application", appName, "env", env);
    }
}
```


---

# Part 6: Spring Boot 集成 Prometheus 完整实践 {#part-6}

## 6.1 完整项目结构

```
spring-boot-prometheus-demo/
+-- pom.xml
+-- src/main/java/com/example/
|   +-- DemoApplication.java
|   +-- config/
|   |   +-- MetricsConfig.java
|   +-- controller/
|   |   +-- OrderController.java
|   +-- service/
|   |   +-- OrderService.java
|   +-- metrics/
|       +-- BusinessMetrics.java
+-- src/main/resources/
    +-- application.yml
```

## 6.2 完整 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project>
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>prometheus-demo</artifactId>
    <version>1.0.0</version>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <!-- Actuator：提供 /actuator 端点 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- Micrometer Prometheus：暴露 /actuator/prometheus -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>
        <!-- AOP：支持 @Timed 注解 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-aop</artifactId>
        </dependency>
        <!-- 数据库连接池监控 -->
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </dependency>
    </dependencies>
</project>
```

## 6.3 完整 application.yml

```yaml
spring:
  application:
    name: order-service
  profiles:
    active: prod

server:
  port: 8080

management:
  server:
    port: 9090         # 建议独立端口（安全考虑）
  endpoints:
    web:
      exposure:
        include: "prometheus,health,info,metrics,env"
  endpoint:
    prometheus:
      enabled: true
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
        descriptions: true
    tags:
      # 全局公共标签，所有指标都会带这些标签
      application: ${spring.application.name}
      env: ${spring.profiles.active:default}
      region: cn-north-1
    distribution:
      percentiles-histogram:
        http.server.requests: true  # 开启Histogram支持histogram_quantile
        order.processing.time: true
      percentiles:
        http.server.requests: 0.5, 0.75, 0.95, 0.99
      slo:
        # 自定义SLO边界（生成更精确的bucket）
        http.server.requests: 10ms, 50ms, 100ms, 200ms, 500ms, 1s, 2s, 5s
```

## 6.4 Spring Boot 内置指标完整列表

```
HTTP 请求（自动计时）：
  http_server_requests_seconds{method, status, uri, exception}
  -> count, sum, bucket（Histogram模式）, max

JVM 内存：
  jvm_memory_used_bytes{area, id}
  jvm_memory_max_bytes{area, id}
  jvm_memory_committed_bytes{area, id}

JVM GC：
  jvm_gc_pause_seconds{action, cause}（Histogram）
  jvm_gc_memory_allocated_bytes_total
  jvm_gc_memory_promoted_bytes_total

JVM 线程：
  jvm_threads_live_threads
  jvm_threads_daemon_threads
  jvm_threads_peak_threads
  jvm_threads_states_threads{state}

HikariCP 连接池（重要）：
  hikaricp_connections_active{pool}
  hikaricp_connections_idle{pool}
  hikaricp_connections_max{pool}
  hikaricp_connections_pending{pool}
  hikaricp_connections_timeout_total{pool}
  hikaricp_connections_acquire_seconds{pool}（Histogram）

Tomcat：
  tomcat_sessions_active_current_sessions
  tomcat_connections_current_connections
  tomcat_threads_busy_threads
  tomcat_threads_current_threads

系统：
  system_cpu_usage
  process_cpu_usage
  system_load_average_1m
  jvm_classes_loaded_classes
  process_uptime_seconds
```

## 6.5 Service 层业务埋点

```java
@Service
public class OrderService {

    @Autowired
    private BusinessMetrics metrics;

    @Transactional
    public OrderVO createOrder(CreateOrderRequest request) {
        long startTime = System.currentTimeMillis();
        String status = "success";

        try {
            // 检查库存
            checkInventory(request.getProductId(), request.getQuantity());

            // 创建订单
            Order order = orderRepository.save(Order.builder()
                .userId(request.getUserId())
                .amount(request.getAmount())
                .status(OrderStatus.PENDING)
                .build());

            // 业务指标埋点
            metrics.recordOrderCreated(request.getAmount().doubleValue());

            return OrderVO.from(order);

        } catch (Exception e) {
            status = "failed";
            metrics.recordOrderFailed(e.getClass().getSimpleName());
            throw e;
        } finally {
            metrics.recordOrderProcessingTime(
                System.currentTimeMillis() - startTime, status);
        }
    }

    // 通过 @Timed 注解自动计时
    @Timed(
        value = "payment.processing.time",
        description = "支付处理耗时",
        histogram = true,
        percentiles = {0.5, 0.95, 0.99}
    )
    @Transactional
    public PaymentVO processPayment(Long orderId, PaymentRequest request) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        try {
            PaymentResult result = paymentService.pay(order, request);
            metrics.recordPayment(result.isSuccess(), order.getAmount().doubleValue());
            return PaymentVO.from(result);
        } catch (Exception e) {
            metrics.recordPayment(false, order.getAmount().doubleValue());
            throw e;
        }
    }
}
```

## 6.6 Prometheus 采集配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-services'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'order-service-1:9090'
          - 'order-service-2:9090'
          - 'order-service-3:9090'
        labels:
          service: 'order-service'
          env: 'production'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: instance
        replacement: '${1}'
```


---

# Part 7: Alertmanager 告警 {#part-7}

## 7.1 告警规则（PrometheusRule）格式

```yaml
# alert_rules.yml
groups:
  - name: infrastructure_alerts
    interval: 30s
    rules:

      - alert: HighCPUUsage
        expr: |
          (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100 > 85
        for: 5m
        labels:
          severity: warning
          team: ops
        annotations:
          summary: "CPU 使用率过高: {{ $labels.instance }}"
          description: "CPU使用率 {{ printf `%.1f` $value }}%，已持续5分钟"
          runbook_url: "https://wiki.example.com/runbooks/high-cpu"

      # Recording Rule（预计算，加速查询）
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)
```

## 7.2 完整生产级告警规则集

```yaml
groups:
  # 基础设施
  - name: infrastructure
    rules:

      - alert: HighCPUUsage
        expr: (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "CPU使用率告警: {{ $labels.instance }}"
          description: "CPU使用率 {{ printf `%.1f` $value }}%"

      - alert: CriticalCPUUsage
        expr: (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) by (instance)) * 100 > 95
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "CPU使用率严重告警: {{ $labels.instance }}"
          description: "CPU使用率 {{ printf `%.1f` $value }}%，立即处理！"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "内存使用率告警: {{ $labels.instance }}"
          description: "内存使用率 {{ printf `%.1f` $value }}%"

      - alert: DiskUsageHigh
        expr: |
          (node_filesystem_size_bytes - node_filesystem_avail_bytes)
          / node_filesystem_size_bytes{fstype!~"tmpfs|devtmpfs"} * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "磁盘使用率告警: {{ $labels.instance }}"
          description: "挂载点 {{ $labels.mountpoint }} 使用率 {{ printf `%.1f` $value }}%"

      - alert: DiskWillFillIn4Hours
        expr: predict_linear(node_filesystem_avail_bytes{mountpoint="/"}[1h], 4*3600) < 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "磁盘预计4小时内耗尽: {{ $labels.instance }}"
          description: "根据当前增长趋势，磁盘将在4小时内写满，请立即清理或扩容"

  # 服务可用性
  - name: service_availability
    rules:

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "服务宕机: {{ $labels.job }}/{{ $labels.instance }}"
          description: "服务已宕机超过1分钟"

      - alert: AllInstancesDown
        expr: count(up{job="web-api"} == 1) == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "所有实例宕机: {{ $labels.job }}"

  # 应用层
  - name: application
    rules:

      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)
          / sum(rate(http_requests_total[5m])) by (job) * 100 > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HTTP 5xx错误率过高: {{ $labels.job }}"
          description: "5xx错误率 {{ printf `%.2f` $value }}%"

      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_request_duration_seconds_bucket[5m])) by (le, job)
          ) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99延迟过高: {{ $labels.job }}"
          description: "P99响应时间 {{ printf `%.3f` $value }}s"

      - alert: JVMHeapMemoryHigh
        expr: |
          jvm_memory_used_bytes{area="heap"}
          / jvm_memory_max_bytes{area="heap"} * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM堆内存使用率过高: {{ $labels.instance }}"
          description: "JVM堆内存使用率 {{ printf `%.1f` $value }}%"

      - alert: ConnectionPoolExhausted
        expr: hikaricp_connections_pending > 5
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "数据库连接池等待过长: {{ $labels.instance }}"
          description: "等待连接线程数: {{ $value }}"

  # 中间件
  - name: middleware
    rules:

      - alert: MySQLReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 60
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL主从延迟过高: {{ $labels.instance }}"
          description: "复制延迟 {{ $value }}秒"

      - alert: RedisMemoryHigh
        expr: redis_memory_used_bytes / redis_memory_max_bytes * 100 > 90
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Redis内存使用率过高: {{ $labels.instance }}"
          description: "使用率 {{ printf `%.1f` $value }}%"

      - alert: KafkaConsumerLagHigh
        expr: sum(kafka_consumergroup_lag) by (consumergroup, topic) > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Kafka消费延迟过高: {{ $labels.consumergroup }}"
          description: "Topic {{ $labels.topic }} Lag={{ $value }}"

      - alert: SSLCertificateExpiringSoon
        expr: (probe_ssl_earliest_cert_expiry - time()) / 86400 < 30
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "SSL证书即将过期: {{ $labels.instance }}"
          description: "证书将在 {{ printf `%.0f` $value }} 天后过期"
```

## 7.3 Alertmanager 完整配置

```yaml
# alertmanager.yml

global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.example.com:587'
  smtp_from: 'alertmanager@example.com'
  smtp_auth_username: 'alertmanager@example.com'
  smtp_auth_password: 'smtp-password'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'default-receiver'
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h

  routes:
    - match:
        severity: critical
      receiver: 'critical-receiver'
      group_wait: 10s
      repeat_interval: 1h

    - match:
        team: ops
      receiver: 'ops-team'
      group_by: ['alertname', 'instance']

    - match_re:
        alertname: 'High.*|.*Error.*|.*Latency.*'
      receiver: 'dev-team'

receivers:
  - name: 'default-receiver'
    email_configs:
      - to: 'ops@example.com'
        send_resolved: true

  - name: 'critical-receiver'
    email_configs:
      - to: 'oncall@example.com'
        send_resolved: true
    webhook_configs:
      - url: 'http://dingtalk-webhook:8060/dingtalk/webhook1/send'
        send_resolved: true

  - name: 'ops-team'
    webhook_configs:
      - url: 'http://dingtalk-webhook:8060/dingtalk/ops-webhook/send'
        send_resolved: true
    wechat_configs:
      - corp_id: 'YOUR_CORP_ID'
        to_party: '2'
        agent_id: '1000002'
        api_secret: 'YOUR_SECRET'
        send_resolved: true

  - name: 'dev-team'
    webhook_configs:
      - url: 'http://dingtalk-webhook:8060/dingtalk/dev-webhook/send'
        send_resolved: true

inhibit_rules:
  # critical 告警抑制同实例的 warning 告警
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['instance', 'alertname']

  # 服务宕机时抑制性能相关告警
  - source_match:
      alertname: 'ServiceDown'
    target_match_re:
      alertname: 'High.*|.*Latency.*|.*Error.*'
    equal: ['job']
```

## 7.4 钉钉机器人 Webhook 配置

```yaml
# docker-compose.yml 中的 dingtalk webhook
dingtalk-webhook:
  image: timonwong/prometheus-webhook-dingtalk:latest
  ports:
    - "8060:8060"
  volumes:
    - ./dingtalk/config.yml:/etc/prometheus-webhook-dingtalk/config.yml
    - ./dingtalk/templates:/etc/prometheus-webhook-dingtalk/templates
```

```yaml
# dingtalk/config.yml
timeout: 5s
templates:
  - /etc/prometheus-webhook-dingtalk/templates/default.tmpl

targets:
  ops_webhook:
    url: https://oapi.dingtalk.com/robot/send?access_token=YOUR_OPS_TOKEN
    secret: YOUR_OPS_SECRET
  dev_webhook:
    url: https://oapi.dingtalk.com/robot/send?access_token=YOUR_DEV_TOKEN
    secret: YOUR_DEV_SECRET
```

钉钉消息模板（templates/default.tmpl）：

```
{{ define "ding.link.title" }}
{{ if eq .Status "firing" }}[告警] {{ .Alerts.Firing | len }}条告警触发
{{- else }}[恢复] {{ .Alerts.Resolved | len }}条告警已恢复{{ end }}
{{ end }}

{{ define "ding.link.content" }}
**状态**: {{ if eq .Status "firing" }}告警中{{ else }}已恢复{{ end }}

{{ range .Alerts }}---
**告警名称**: {{ .Labels.alertname }}
**严重程度**: {{ .Labels.severity }}
**告警实例**: {{ .Labels.instance }}
**触发时间**: {{ (.StartsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}
{{ if eq .Status "resolved" }}**恢复时间**: {{ (.EndsAt.Add 28800e9).Format "2006-01-02 15:04:05" }}{{ end }}
**摘要**: {{ .Annotations.summary }}
**详情**: {{ .Annotations.description }}
{{ end }}
{{ end }}
```

## 7.5 告警分组、抑制、静默总结

```
告警三大机制：

1. 分组（Grouping）
   目的：减少告警风暴
   场景：DB宕机导致50个服务同时报错 -> 合并为1条通知
   配置：group_by / group_wait / group_interval / repeat_interval

2. 抑制（Inhibit）
   目的：避免级联告警噪音
   场景：网络分区时抑制服务宕机告警（根因已知）
   配置：inhibit_rules（source_match + target_match + equal）

3. 静默（Silence）
   目的：计划维护时临时屏蔽告警
   操作：Alertmanager UI -> Silences -> New Silence
   命令行：
   amtool silence add \
     --alertmanager.url=http://localhost:9093 \
     --author="ops-team" \
     --comment="系统维护" \
     --duration=2h \
     alertname="HighCPUUsage" instance="192.168.1.100:9100"
```


---

# Part 8: Grafana 深度使用 {#part-8}

## 8.1 核心概念层次图

```
Organization（组织）
  +-- Team（团队）
  +-- Folder（文件夹）
  |     +-- Dashboard（仪表板）
  |           +-- Panel（面板）
  |           |     +-- Query（查询 PromQL）
  |           |     +-- Visualization（可视化类型）
  |           |     +-- Alert（面板告警）
  |           +-- Variable（变量，让Dashboard可交互）
  |           +-- Annotation（注解，在图表上标记事件）
  +-- DataSource（数据源）
        +-- Prometheus
        +-- Loki
        +-- Elasticsearch
        +-- MySQL
```

## 8.2 数据源配置

### Prometheus 数据源
```
Configuration -> Data Sources -> Add data source -> Prometheus

配置项：
  URL: http://prometheus:9090
  Access: Server（推荐，由 Grafana Server 转发）
  Scrape interval: 15s（与 Prometheus 采集间隔一致）
  Query timeout: 60s
  HTTP Method: GET
```

## 8.3 面板类型速查

```
+------------------+----------------------------------------------+
| 面板类型          | 最适合的场景                                  |
+------------------+----------------------------------------------+
| Time series      | 时间序列折线图（最常用，展示趋势变化）        |
| Bar chart        | 分组对比柱状图                               |
| Pie chart        | 占比分布（各接口流量占比）                   |
| Heatmap          | 热力图（响应时间分布热图）                   |
| Table            | 表格（多维数据，支持排序/过滤）              |
| Stat             | 单值展示（当前QPS、在线实例数）              |
| Gauge            | 仪表盘（当前值在最大值中的占比）             |
| Bar gauge        | 横向进度条（多实例对比）                     |
| Geomap           | 地理分布图（流量来源地图）                   |
| Logs             | 日志展示（与 Loki 配合）                     |
| Traces           | 链路追踪（与 Tempo 配合）                    |
| State timeline   | 状态时间线（展示状态变化历史）               |
| Canvas           | 自定义图形（网络拓扑图等）                   |
+------------------+----------------------------------------------+
```

## 8.4 变量（Variables）设计

变量让 Dashboard 变得动态可交互，是 Grafana 最强大的功能之一。

### 8.4.1 Query 变量（动态从数据源查询可选值）

```
Dashboard Settings -> Variables -> Add variable

示例1：动态选择 job
  Name: job
  Type: Query
  Data source: Prometheus
  Query: label_values(up, job)
  Refresh: On Dashboard Load

示例2：依赖 job 动态选择 instance
  Name: instance
  Type: Query
  Query: label_values(up{job="$job"}, instance)
  Multi-value: true（允许多选）
  Include All option: true（允许选"全部"）
  All value: .+（正则表示全选）

示例3：时间区间变量
  Name: interval
  Type: Interval
  Values: 1m,5m,10m,30m,1h,6h,1d
  Auto option: true
```

### 8.4.2 在 PromQL 中使用变量

```promql
# $job 变量替换
sum(rate(http_requests_total{job="$job"}[5m])) by (handler)

# $instance 多选变量（用 =~）
sum(rate(http_requests_total{instance=~"$instance"}[$__interval])) by (handler)

# $__rate_interval（专为 rate/irate 设计，强烈推荐）
rate(http_requests_total{job="$job"}[$__rate_interval])

# $__range（当前仪表板时间范围）
increase(http_requests_total{job="$job"}[$__range])

# 带变量的内存使用率查询
jvm_memory_used_bytes{application="$application", instance=~"$instance", area="heap"}
  / jvm_memory_max_bytes{application="$application", instance=~"$instance", area="heap"} * 100
```

## 8.5 常用 Dashboard ID 速查

```
基础设施：
  1860  - Node Exporter Full（最完整的主机监控，强烈推荐）
  7362  - Node Exporter for Prometheus Dashboard
  13978 - Node Exporter Quickstart and Dashboard

JVM / Java / Spring Boot：
  4701  - JVM Micrometer（Spring Boot 2.x）
  11378 - Micrometer Spring Boot 2.1 Statistics
  12900 - SpringBoot APM Dashboard
  10280 - Spring Boot Actuator

Kubernetes：
  15661 - Kubernetes Cluster Monitoring (via Prometheus)
  13770 - Kubernetes All-in-one Cluster Monitoring
  6417  - Kubernetes cluster prometheus

中间件：
  14057 - MySQL Dashboard
  11835 - Redis Dashboard for Prometheus Redis Exporter
  7589  - Kafka Overview

黑盒探测：
  7587  - Blackbox Exporter 0.14+

容器：
  193   - Docker and system monitoring (cAdvisor)
  14282 - Cadvisor exporter

导入方式：
  Dashboards -> New -> Import -> 输入 Dashboard ID -> Load
```

## 8.6 Grafana Unified Alerting

```
配置流程：

1. 创建 Contact points（联系点）
   Alerting -> Contact points -> Add contact point
   支持：Email / DingTalk / WeChat / Webhook / Slack / PagerDuty

2. 创建 Notification policies（通知策略）
   Alerting -> Notification policies
   路由规则（类似 Alertmanager route）

3. 创建 Alert rules
   方式1：在 Panel 编辑界面直接创建
   方式2：Alerting -> Alert rules -> New alert rule

告警状态机：
  Normal -> Pending（触发但未超过for时间）-> Firing（确认告警）
  Firing -> Normal（条件不满足，恢复）
```

## 8.7 Grafana Provisioning（配置即代码）

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      scrapeInterval: '15s'
      queryTimeout: '60s'
      httpMethod: 'GET'

# /etc/grafana/provisioning/dashboards/default.yml
apiVersion: 1
providers:
  - name: 'Default'
    orgId: 1
    type: file
    disableDeletion: false
    editable: true
    updateIntervalSeconds: 30
    options:
      path: /etc/grafana/dashboards
      foldersFromFilesStructure: true
```

## 8.8 微服务 RED 指标 Dashboard 布局设计

```
+-----------------------------------------------------------------------+
|  服务: $service  实例: $instance  时间: [时间选择器]                   |
+-----------------------------------------------------------------------+
|  关键指标概览（Stat Panel）                                            |
|  +------------------+  +------------------+  +-------------------+   |
|  |     QPS           |  |   5xx 错误率      |  |   P99 延迟         |   |
|  |   1234 req/s     |  |   0.12%           |  |   45ms             |   |
|  +------------------+  +------------------+  +-------------------+   |
+-----------------------------------------------------------------------+
|  趋势图（Time series Panel）                                           |
|  +----------------------------------+  +--------------------------+   |
|  |  请求速率 QPS 趋势               |  |  HTTP 错误率趋势          |   |
|  +----------------------------------+  +--------------------------+   |
|  +----------------------------------+  +--------------------------+   |
|  |  P50/P95/P99 延迟分布           |  |  响应时间热力图（Heatmap） |   |
|  +----------------------------------+  +--------------------------+   |
+-----------------------------------------------------------------------+
|  JVM 监控（Time series + Gauge）                                       |
|  +------------------+  +------------------+  +-------------------+   |
|  |  JVM堆内存使用率  |  |  GC暂停时间趋势   |  |  线程状态分布      |   |
|  +------------------+  +------------------+  +-------------------+   |
+-----------------------------------------------------------------------+
|  基础设施 & 中间件                                                     |
|  +------------------+  +------------------+  +-------------------+   |
|  |  CPU 使用率       |  |  内存使用率       |  |  连接池使用率       |   |
|  +------------------+  +------------------+  +-------------------+   |
+-----------------------------------------------------------------------+
```

---

# Part 9: 完整监控体系搭建 {#part-9}

## 9.1 Docker Compose 一键部署

```yaml
# docker-compose.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:

services:

  prometheus:
    image: prom/prometheus:v2.49.1
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=50GB'
      - '--storage.tsdb.wal-compression'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    networks:
      - monitoring
    restart: always

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - ./alertmanager/templates:/etc/alertmanager/templates
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring
    restart: always

  grafana:
    image: grafana/grafana:10.3.3
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=your_secure_password
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_INSTALL_PLUGINS=grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/etc/grafana/dashboards
    networks:
      - monitoring
    restart: always
    depends_on:
      - prometheus

  node-exporter:
    image: prom/node-exporter:v1.7.0
    container_name: node_exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
    networks:
      - monitoring
    pid: host
    restart: always

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.47.2
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker:/var/lib/docker:ro
    privileged: true
    networks:
      - monitoring
    restart: always

  pushgateway:
    image: prom/pushgateway:v1.7.0
    container_name: pushgateway
    ports:
      - "9091:9091"
    networks:
      - monitoring
    restart: always

  blackbox-exporter:
    image: prom/blackbox-exporter:v0.24.0
    container_name: blackbox_exporter
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox/blackbox.yml:/etc/blackbox_exporter/config.yml
    networks:
      - monitoring
    restart: always

  dingtalk-webhook:
    image: timonwong/prometheus-webhook-dingtalk:latest
    container_name: dingtalk_webhook
    ports:
      - "8060:8060"
    volumes:
      - ./dingtalk/config.yml:/etc/prometheus-webhook-dingtalk/config.yml
    networks:
      - monitoring
    restart: always
```

## 9.2 prometheus.yml 完整配置

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s
  external_labels:
    cluster: 'cn-prod'
    region: 'cn-north-1'

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'alertmanager:9093'

scrape_configs:

  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets:
          - 'node-exporter:9100'
          - '192.168.1.100:9100'
    relabel_configs:
      - source_labels: [__address__]
        regex: '([^:]+):.*'
        target_label: hostname
        replacement: '${1}'

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'spring-boot-order-service'
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'order-service-1:9090'
          - 'order-service-2:9090'
        labels:
          service: 'order-service'
          env: 'production'

  - job_name: 'spring-boot-dynamic'
    metrics_path: '/actuator/prometheus'
    file_sd_configs:
      - files:
          - '/etc/prometheus/targets/spring-boot/*.json'
        refresh_interval: 30s

  - job_name: 'mysql'
    static_configs:
      - targets: ['mysqld-exporter:9104']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']

  - job_name: 'blackbox_http'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - 'https://www.example.com'
          - 'http://api-gateway:8080/health'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'blackbox_tcp'
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
          - 'mysql:3306'
          - 'redis:6379'
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115

  - job_name: 'pushgateway'
    honor_labels: true
    static_configs:
      - targets: ['pushgateway:9091']
```

## 9.3 K8s 部署（kube-prometheus-stack）

```bash
# 添加 Helm 仓库
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# 安装 kube-prometheus-stack
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=50Gi \
  --set grafana.adminPassword=your_secure_password

# 查看安装结果
kubectl get pods -n monitoring
kubectl get svc -n monitoring

# 访问 Grafana
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring
```

**kube-prometheus-stack 包含的组件：**
```
Prometheus Operator     - K8s CRD 管理 Prometheus 实例
Prometheus              - 指标采集和存储
Grafana                 - 可视化
Alertmanager            - 告警管理
kube-state-metrics      - K8s 对象状态指标（Deployment/Pod/Node等）
node-exporter           - 主机指标（DaemonSet）
prometheus-adapter      - K8s HPA 自定义指标适配器
```

## 9.4 Kubernetes 核心监控指标

```promql
# 集群 CPU 使用率
sum(rate(container_cpu_usage_seconds_total{image!=""}[5m])) by (namespace)
  / sum(kube_node_status_allocatable{resource="cpu"})

# 集群内存使用率
sum(container_memory_working_set_bytes{image!=""}) by (namespace)
  / sum(kube_node_status_allocatable{resource="memory"})

# Pod CPU 使用率（相对 request）
sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod, namespace)
  / sum(kube_pod_container_resource_requests{resource="cpu"}) by (pod, namespace)

# Deployment 期望副本 vs 可用副本
kube_deployment_spec_replicas{namespace="production"}
kube_deployment_status_replicas_available{namespace="production"}

# Pod 重启次数（过去1小时）
sum(increase(kube_pod_container_status_restarts_total[1h])) by (pod, namespace)

# 节点是否 Ready
kube_node_status_condition{condition="Ready", status="true"}

# PVC 使用率
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes * 100
```

## 9.5 K8s Pod 注解实现自动服务发现

```yaml
# deployment.yaml - 添加以下注解使 Prometheus 自动发现并采集
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"        # 启用采集
        prometheus.io/port: "9090"          # 采集端口
        prometheus.io/path: "/actuator/prometheus"  # 采集路径
    spec:
      containers:
        - name: order-service
          image: order-service:1.0.0
          ports:
            - containerPort: 8080
              name: http
            - containerPort: 9090
              name: metrics
```

---

# Part 10: Loki 日志系统（Prometheus 生态）

## 10.1 可观测性三支柱

```
┌─────────────────────────────────────────────────────────────────┐
│                    可观测性三支柱 (Three Pillars)                  │
│                                                                  │
│   Metrics (指标)        Logs (日志)          Traces (链路追踪)    │
│   ┌──────────┐          ┌──────────┐          ┌──────────┐       │
│   │Prometheus│          │  Loki    │          │  Tempo   │       │
│   │  Grafana │◄────────►│  Grafana │◄────────►│  Grafana │       │
│   └──────────┘          └──────────┘          └──────────┘       │
│                                                                  │
│   "什么出问题了"         "为什么出问题"        "哪里出问题了"       │
│   (What happened)       (Why it happened)     (Where it happened)│
└─────────────────────────────────────────────────────────────────┘
```

三支柱联动工作流：
1. **Metrics** 触发告警（CPU > 80%）
2. 跳转 **Logs** 查看错误详情（`{app="order-service"} |= "ERROR"`）
3. 从日志中获取 TraceID，跳转 **Traces** 查看完整调用链

## 10.2 Loki 架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         Loki 架构                                │
│                                                                  │
│  日志来源                                                         │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐                     │
│  │Promtail  │   │Fluentd   │   │Logstash  │  日志采集器           │
│  │(K8s Pod) │   │(容器日志)│   │(应用日志)│                      │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘                     │
│       └──────────────┴──────────────┘                            │
│                             │                                    │
│                    ┌────────▼────────┐                           │
│                    │  Loki Gateway   │  负载均衡                  │
│                    └────────┬────────┘                           │
│                             │                                    │
│              ┌──────────────┼──────────────┐                     │
│              ▼              ▼              ▼                     │
│        ┌──────────┐  ┌──────────┐  ┌──────────┐                 │
│        │Distributor│ │Distributor│ │Distributor│ 分发器           │
│        └─────┬────┘  └─────┬────┘  └─────┬────┘                 │
│              └─────────────┼─────────────┘                      │
│                            │  一致性哈希                          │
│              ┌─────────────┼─────────────┐                      │
│              ▼             ▼             ▼                       │
│        ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│        │ Ingester │ │ Ingester │ │ Ingester │  写入器(内存)        │
│        └─────┬────┘ └─────┬────┘ └─────┬────┘                   │
│              └────────────┼────────────┘                        │
│                           │ 定期刷盘                              │
│                    ┌──────▼──────┐                               │
│                    │Object Store │  对象存储                      │
│                    │(S3/GCS/OSS) │                               │
│                    └─────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

### Loki 核心特点

| 特性 | 说明 |
|------|------|
| 仅索引标签 | 不索引日志内容，只索引标签（极低存储开销） |
| 压缩存储 | 日志内容 gzip 压缩，节省 90%+ 空间 |
| 兼容 Prometheus | 使用相同标签体系，天然与 Prometheus 联动 |
| LogQL | 类 PromQL 的日志查询语言 |
| 多租户 | 支持多租户隔离（X-Scope-OrgID） |

## 10.3 Loki 部署配置

### Docker Compose 部署

```yaml
version: '3'
services:
  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki-config.yaml:/etc/loki/local-config.yaml
      - loki-data:/loki
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail-config.yaml:/etc/promtail/config.yaml
    command: -config.file=/etc/promtail/config.yaml
    depends_on:
      - loki
    networks:
      - monitoring

volumes:
  loki-data:
networks:
  monitoring:
    external: true
```

### loki-config.yaml（单机版）

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  instance_addr: 127.0.0.1
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

query_range:
  results_cache:
    cache:
      embedded_cache:
        enabled: true
        max_size_mb: 100

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://alertmanager:9093

limits_config:
  retention_period: 744h   # 31天
  ingestion_rate_mb: 16
  ingestion_burst_size_mb: 32

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  shared_store: filesystem
  retention_enabled: true
  retention_delete_delay: 2h
```

## 10.4 Promtail 配置

### 采集 Docker 容器日志

```yaml
# promtail-config.yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  # 采集 Docker 容器日志
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
        filters:
          - name: status
            values: ["running"]
    relabel_configs:
      # 从容器标签提取 app 标签
      - source_labels: ['__meta_docker_container_label_com_docker_compose_service']
        target_label: app
      # 提取容器名
      - source_labels: ['__meta_docker_container_name']
        regex: '/(.*)'
        target_label: container
      # 提取镜像名
      - source_labels: ['__meta_docker_container_image']
        target_label: image
    pipeline_stages:
      # 解析 Spring Boot JSON 日志
      - json:
          expressions:
            level: level
            logger: logger_name
            message: message
            timestamp: '@timestamp'
            traceId: traceId
            spanId: spanId
      - labels:
          level:
          logger:
          traceId:
      - timestamp:
          source: timestamp
          format: RFC3339Nano
      - output:
          source: message

  # 采集系统日志
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          host: ${HOSTNAME}
          __path__: /var/log/*.log

  # 采集 Nginx 访问日志
  - job_name: nginx
    static_configs:
      - targets:
          - localhost
        labels:
          job: nginx
          __path__: /var/log/nginx/access.log
    pipeline_stages:
      - regex:
          expression: '^(?P<ip>\S+) \S+ \S+ \[(?P<timestamp>[^\]]+)\] "(?P<method>\S+) (?P<path>\S+) \S+" (?P<status>\d+) (?P<size>\d+)'
      - labels:
          method:
          status:
      - metrics:
          nginx_request_count:
            type: Counter
            description: "Total Nginx requests"
            source: status
            config:
              action: inc
```

### 采集 K8s Pod 日志（Promtail DaemonSet）

```yaml
# promtail-k8s-config.yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    pipeline_stages:
      - cri: {}   # 解析 containerd CRI 格式
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'   # 多行日志合并
          max_wait_time: 3s
      - json:
          expressions:
            level: level
            traceId: traceId
      - labels:
          level:
          traceId:
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_node_name]
        target_label: node
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      # 只采集有 log-enabled 注解的 Pod
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_log]
        action: keep
        regex: "true"
```

## 10.5 LogQL 查询语言

LogQL 是 Loki 的查询语言，语法类似 PromQL。

### 基本语法

```
{label_selector} | pipeline_expression
```

### 日志选择器

```logql
# 精确匹配
{app="order-service"}

# 正则匹配
{app=~"order-.*"}

# 不等于
{app!="gateway"}

# 多条件（AND）
{app="order-service", namespace="prod"}

# 多条件（OR）
{app="order-service"} or {app="payment-service"}
```

### 日志过滤

```logql
# 包含字符串
{app="order-service"} |= "ERROR"

# 不包含字符串
{app="order-service"} != "health"

# 正则过滤
{app="order-service"} |~ "timeout|connection refused"

# 不匹配正则
{app="order-service"} !~ "GET /actuator.*"
```

### 解析器（Parser）

```logql
# JSON 解析
{app="order-service"} | json

# 提取特定字段
{app="order-service"} | json level, traceId, message

# logfmt 格式解析
{app="order-service"} | logfmt

# 正则提取
{app="nginx"} | regexp `(?P<method>\w+) (?P<path>\S+) HTTP/\d\.\d" (?P<status>\d+)`

# pattern 解析（更简洁）
{app="nginx"} | pattern `<ip> - - [<_>] "<method> <path> <_>" <status> <size>`
```

### 指标查询（Log Metrics）

```logql
# 统计错误日志数量（过去5分钟速率）
rate({app="order-service"} |= "ERROR" [5m])

# 按 level 统计日志量
sum by (level) (
  rate({app="order-service"} | json | __error__="" [5m])
)

# 错误率
sum(rate({app="order-service"} |= "ERROR" [5m]))
/
sum(rate({app="order-service"} [5m]))

# 解析 JSON 后按字段过滤和统计
sum by (status) (
  rate({app="nginx"} | json | status >= 500 [5m])
)

# P99 响应时间（从日志中提取 duration 字段）
quantile_over_time(0.99,
  {app="order-service"}
  | json
  | duration > 0 [5m]
) by (endpoint)
```

### 常用 LogQL 查询示例

```logql
# 查看最近 1000 条错误日志
{app="order-service", namespace="prod"} |= "ERROR" | limit 1000

# 查看特定 TraceID 的完整链路日志
{namespace="prod"} |= "abc123traceId"

# 统计各服务错误数 Top 10
topk(10,
  sum by (app) (
    count_over_time({namespace="prod"} |= "ERROR" [1h])
  )
)

# 查找 OOM 错误
{namespace="prod"} |~ "OutOfMemoryError|java.lang.OutOfMemory"

# 统计 HTTP 5xx 错误
sum by (app) (
  rate(
    {namespace="prod"} | json | status_code >= 500 [5m]
  )
)

# 查看慢请求（> 1s）
{app="order-service"} | json | duration > 1000

# 按分钟统计日志量
sum by (app) (
  count_over_time({namespace="prod"} [1m])
)
```

## 10.6 Grafana 中展示 Loki 数据

### 添加 Loki 数据源

1. Configuration → Data Sources → Add data source
2. 选择 **Loki**
3. URL 填写 `http://loki:3100`
4. 点击 **Save & Test**

### 探索面板（Explore）

在 Grafana Explore 中：
- 选择 Loki 数据源
- 输入 LogQL 查询
- 可在 **Logs** 和 **Metrics** 视图之间切换

### 关联 Metrics 和 Logs

在 Prometheus 数据源中配置 Derived Fields：

```yaml
# 在 Grafana 数据源配置中
derivedFields:
  - name: TraceID
    matcherRegex: "traceId=([\\w-]+)"
    url: "http://tempo:3000/explore?orgId=1&left=[\"now-1h\",\"now\",\"Tempo\",{\"query\":\"${__value.raw}\"}]"
    urlDisplayLabel: "View in Tempo"
```

实现效果：点击日志中的 TraceID 直接跳转到 Tempo 链路追踪。

## 10.7 Loki 告警规则

```yaml
# loki-rules.yaml
groups:
  - name: loki-alerts
    rules:
      # 错误日志突增
      - alert: HighErrorLogRate
        expr: |
          sum by (app) (
            rate({namespace="prod"} |= "ERROR" [5m])
          ) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "{{ $labels.app }} 错误日志突增"
          description: "过去5分钟错误日志速率 {{ $value | humanize }}/s"

      # OOM 告警
      - alert: OutOfMemoryError
        expr: |
          count_over_time(
            {namespace="prod"} |~ "OutOfMemoryError" [5m]
          ) > 0
        labels:
          severity: critical
        annotations:
          summary: "检测到 OOM 错误"

      # 服务无日志输出（可能宕机）
      - alert: NoLogOutput
        expr: |
          absent_over_time(
            {app="order-service", namespace="prod"} [10m]
          )
        labels:
          severity: critical
        annotations:
          summary: "order-service 10分钟无日志输出"
```

---

# Part 11: 完整实战案例

## 案例一：Spring Boot 微服务完整监控方案

### 目标

为一个电商订单服务配置完整监控：
- JVM 内存、GC、线程监控
- HTTP 接口 QPS、延迟、错误率
- 数据库连接池监控
- 业务指标（订单量、支付成功率）
- 告警规则

### Step 1: 依赖配置

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Micrometer Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Micrometer Tracing (Spring Boot 3.x) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-tracing-bridge-brave</artifactId>
    </dependency>
    <dependency>
        <groupId>io.zipkin.reporter2</groupId>
        <artifactId>zipkin-reporter-brave</artifactId>
    </dependency>

    <!-- HikariCP (已内置，无需额外引入) -->
</dependencies>
```

### Step 2: application.yml 配置

```yaml
spring:
  application:
    name: order-service
  datasource:
    hikari:
      pool-name: OrderHikariPool
      maximum-pool-size: 20
      minimum-idle: 5

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
    prometheus:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      # 全局标签：所有指标都带上这些标签
      application: ${spring.application.name}
      env: ${APP_ENV:dev}
      version: ${APP_VERSION:unknown}
    distribution:
      # HTTP 请求延迟直方图分桶
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5,0.75,0.95,0.99
      slo:
        http.server.requests: 50ms,100ms,200ms,500ms,1s

  tracing:
    sampling:
      probability: 0.1  # 10% 采样率
    zipkin:
      tracing:
        endpoint: http://zipkin:9411/api/v2/spans

logging:
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} traceId=%X{traceId} spanId=%X{spanId} - %msg%n"
```

### Step 3: 业务指标埋点

```java
package com.example.order.metrics;

import io.micrometer.core.instrument.*;
import org.springframework.stereotype.Component;

import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicInteger;

@Component
public class OrderMetrics {

    // ===== 计数器 =====
    private final Counter orderCreatedTotal;
    private final Counter orderPaidTotal;
    private final Counter orderFailedTotal;
    private final Counter paymentTimeoutTotal;

    // ===== 直方图（延迟） =====
    private final Timer orderProcessingTimer;
    private final Timer paymentCallTimer;

    // ===== 仪表盘（实时数量） =====
    private final AtomicInteger pendingOrders = new AtomicInteger(0);
    private final AtomicInteger processingOrders = new AtomicInteger(0);

    // ===== 摘要（金额分布） =====
    private final DistributionSummary orderAmountSummary;

    public OrderMetrics(MeterRegistry registry) {
        // 初始化计数器
        this.orderCreatedTotal = Counter.builder("order.created.total")
                .description("创建订单总数")
                .tag("service", "order")
                .register(registry);

        this.orderPaidTotal = Counter.builder("order.paid.total")
                .description("支付成功订单总数")
                .register(registry);

        this.orderFailedTotal = Counter.builder("order.failed.total")
                .description("失败订单总数")
                .register(registry);

        this.paymentTimeoutTotal = Counter.builder("payment.timeout.total")
                .description("支付超时次数")
                .register(registry);

        // 初始化定时器
        this.orderProcessingTimer = Timer.builder("order.processing.duration")
                .description("订单处理耗时")
                .publishPercentiles(0.5, 0.95, 0.99)
                .publishPercentileHistogram()
                .sla(
                    java.time.Duration.ofMillis(100),
                    java.time.Duration.ofMillis(500),
                    java.time.Duration.ofSeconds(1)
                )
                .register(registry);

        this.paymentCallTimer = Timer.builder("payment.rpc.duration")
                .description("支付服务调用耗时")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);

        // 注册 Gauge（使用 AtomicInteger）
        Gauge.builder("order.pending.count", pendingOrders, AtomicInteger::get)
                .description("待处理订单数")
                .register(registry);

        Gauge.builder("order.processing.count", processingOrders, AtomicInteger::get)
                .description("处理中订单数")
                .register(registry);

        // 初始化 DistributionSummary
        this.orderAmountSummary = DistributionSummary.builder("order.amount")
                .description("订单金额分布（分）")
                .baseUnit("fen")
                .publishPercentiles(0.5, 0.75, 0.9, 0.99)
                .scale(0.01)   // 转换为元
                .register(registry);
    }

    // ===== 业务方法 =====

    public void recordOrderCreated(String channel) {
        orderCreatedTotal.increment();
    }

    public void recordOrderPaid(String paymentMethod, long amountFen) {
        orderPaidTotal.increment();
        orderAmountSummary.record(amountFen);
    }

    public void recordOrderFailed(String reason) {
        registry.counter("order.failed.total", "reason", reason).increment();
    }

    public Timer.Sample startOrderProcessing() {
        pendingOrders.incrementAndGet();
        return Timer.start();
    }

    public void finishOrderProcessing(Timer.Sample sample, String status) {
        pendingOrders.decrementAndGet();
        sample.stop(Timer.builder("order.processing.duration")
                .tag("status", status)
                .register(registry));
    }

    // 带标签的支付计时
    public void recordPaymentCall(long durationMs, String provider, boolean success) {
        paymentCallTimer = Timer.builder("payment.rpc.duration")
                .tag("provider", provider)
                .tag("success", String.valueOf(success))
                .register(registry);
        paymentCallTimer.record(durationMs, TimeUnit.MILLISECONDS);
    }

    // 将 registry 存为成员变量
    private final MeterRegistry registry;
}
```

### Step 4: Service 层使用

```java
@Service
@Slf4j
public class OrderService {

    @Autowired
    private OrderMetrics orderMetrics;

    @Autowired
    private PaymentClient paymentClient;

    public Order createOrder(CreateOrderRequest request) {
        // 记录订单创建
        orderMetrics.recordOrderCreated(request.getChannel());

        // 计时处理过程
        Timer.Sample sample = orderMetrics.startOrderProcessing();

        try {
            // 业务逻辑
            Order order = doCreateOrder(request);

            // 调用支付
            long start = System.currentTimeMillis();
            boolean paySuccess = paymentClient.pay(order);
            long elapsed = System.currentTimeMillis() - start;
            orderMetrics.recordPaymentCall(elapsed, "alipay", paySuccess);

            if (paySuccess) {
                orderMetrics.recordOrderPaid("alipay", order.getAmount());
                orderMetrics.finishOrderProcessing(sample, "success");
            } else {
                orderMetrics.recordOrderFailed("payment_failed");
                orderMetrics.finishOrderProcessing(sample, "payment_failed");
            }
            return order;
        } catch (Exception e) {
            orderMetrics.recordOrderFailed("exception");
            orderMetrics.finishOrderProcessing(sample, "exception");
            throw e;
        }
    }
}
```

### Step 5: Prometheus 告警规则

```yaml
# order-service-rules.yaml
groups:
  - name: order-service
    rules:
      # 订单处理 P99 延迟
      - alert: OrderHighLatency
        expr: |
          histogram_quantile(0.99,
            rate(order_processing_duration_seconds_bucket{application="order-service"}[5m])
          ) > 2
        for: 3m
        labels:
          severity: warning
          team: order
        annotations:
          summary: "订单处理 P99 延迟过高: {{ $value | humanizeDuration }}"

      # 订单失败率
      - alert: HighOrderFailureRate
        expr: |
          rate(order_failed_total[5m])
          /
          rate(order_created_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
          team: order
        annotations:
          summary: "订单失败率 {{ $value | humanizePercentage }} 超过 5%"

      # 支付超时突增
      - alert: PaymentTimeoutSpike
        expr: rate(payment_timeout_total[5m]) > 1
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "支付超时突增: {{ $value }}/s"

      # JVM 内存不足
      - alert: JvmMemoryHigh
        expr: |
          jvm_memory_used_bytes{application="order-service", area="heap"}
          /
          jvm_memory_max_bytes{application="order-service", area="heap"} > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "JVM 堆内存使用率 {{ $value | humanizePercentage }}"

      # 数据库连接池耗尽
      - alert: HikariPoolExhausted
        expr: |
          hikaricp_connections_pending{application="order-service"} > 5
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "HikariCP 等待连接数: {{ $value }}"
```

### Step 6: Grafana Dashboard 面板

```
订单服务监控 Dashboard 布局：

行1（关键指标概览）：
┌────────────┬────────────┬────────────┬────────────┐
│  订单QPS   │  成功率    │  P99延迟   │  待处理数  │
│  Stat面板  │  Stat面板  │  Stat面板  │  Stat面板  │
└────────────┴────────────┴────────────┴────────────┘

行2（趋势图）：
┌──────────────────────────┬──────────────────────────┐
│    订单创建/支付趋势      │      错误率趋势           │
│    (Time series)          │      (Time series)        │
└──────────────────────────┴──────────────────────────┘

行3（延迟分析）：
┌──────────────────────────┬──────────────────────────┐
│  订单处理延迟热图         │  支付调用延迟分布         │
│  (Heatmap)                │  (Histogram)              │
└──────────────────────────┴──────────────────────────┘

行4（JVM监控）：
┌────────────┬────────────┬────────────┬────────────┐
│  堆内存    │  GC次数    │  线程数    │ 连接池状态 │
│  Gauge     │  Time series│  Time series│ Stat      │
└────────────┴────────────┴────────────┴────────────┘
```

---

## 案例二：JVM 全面监控大盘

### 核心 PromQL 查询

```promql
# ===== 内存 =====

# 堆内存使用率
sum(jvm_memory_used_bytes{area="heap"}) by (application)
/
sum(jvm_memory_max_bytes{area="heap"}) by (application)

# 各内存池使用情况
jvm_memory_used_bytes{application="$app"}

# 元空间使用量
jvm_memory_used_bytes{area="nonheap", id="Metaspace"}

# ===== GC =====

# GC 停顿时间（秒/分钟）
rate(jvm_gc_pause_seconds_sum[1m])

# GC 次数/秒
rate(jvm_gc_pause_seconds_count[1m])

# GC 原因分布
sum by (cause) (rate(jvm_gc_pause_seconds_count[5m]))

# ===== 线程 =====

# 活跃线程数
jvm_threads_live_threads{application="$app"}

# 线程状态分布
jvm_threads_states_threads{application="$app"}

# 守护线程数
jvm_threads_daemon_threads{application="$app"}

# ===== 类加载 =====

# 已加载类数量
jvm_classes_loaded_classes{application="$app"}

# 类加载/卸载速率
rate(jvm_classes_loaded_classes[1m])

# ===== CPU =====

# JVM 进程 CPU 使用率
process_cpu_usage{application="$app"}

# 系统 CPU 使用率
system_cpu_usage{application="$app"}

# ===== 文件描述符 =====
process_open_fds{application="$app"}
process_max_fds{application="$app"}
```

---

## 案例三：MySQL 慢查询与性能监控

### Prometheus 查询示例

```promql
# ===== 连接数 =====

# 当前连接数
mysql_global_status_threads_connected

# 最大历史连接数
mysql_global_status_max_used_connections

# 连接使用率
mysql_global_status_threads_connected
/
mysql_global_variables_max_connections

# ===== 查询性能 =====

# QPS（每秒查询数）
rate(mysql_global_status_queries[1m])

# 慢查询速率
rate(mysql_global_status_slow_queries[1m])

# 慢查询占比
rate(mysql_global_status_slow_queries[5m])
/
rate(mysql_global_status_queries[5m])

# 各语句类型 QPS
rate(mysql_global_status_commands_total{command=~"select|insert|update|delete"}[1m])

# ===== 缓冲池 =====

# InnoDB Buffer Pool 命中率
1 - (
  rate(mysql_global_status_innodb_buffer_pool_reads[5m])
  /
  rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])
)

# Buffer Pool 使用率
mysql_global_status_innodb_buffer_pool_pages_data
/
mysql_global_status_innodb_buffer_pool_pages_total

# ===== 锁 =====

# 表锁等待
rate(mysql_global_status_table_locks_waited[1m])

# InnoDB 行锁等待
rate(mysql_global_status_innodb_row_lock_waits[1m])

# 平均行锁等待时间
rate(mysql_global_status_innodb_row_lock_time[1m])
/
rate(mysql_global_status_innodb_row_lock_waits[1m])

# ===== 复制延迟 =====
mysql_slave_status_seconds_behind_master
```

### 告警规则

```yaml
groups:
  - name: mysql
    rules:
      - alert: MySQLHighSlowQueryRate
        expr: rate(mysql_global_status_slow_queries[5m]) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "MySQL 慢查询速率: {{ $value }}/s"

      - alert: MySQLBufferPoolLowHitRate
        expr: |
          1 - (
            rate(mysql_global_status_innodb_buffer_pool_reads[5m])
            /
            rate(mysql_global_status_innodb_buffer_pool_read_requests[5m])
          ) < 0.95
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "InnoDB Buffer Pool 命中率低: {{ $value | humanizePercentage }}"

      - alert: MySQLReplicationLag
        expr: mysql_slave_status_seconds_behind_master > 30
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL 主从复制延迟 {{ $value }}s"

      - alert: MySQLConnectionUsageHigh
        expr: |
          mysql_global_status_threads_connected
          /
          mysql_global_variables_max_connections > 0.8
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "MySQL 连接数使用率 {{ $value | humanizePercentage }}"
```

---

## 案例四：Kafka 消费延迟监控

### 核心指标

```promql
# ===== 消费延迟（Lag）=====

# 各消费者组总 Lag
sum by (consumergroup, topic) (
  kafka_consumergroup_lag
)

# Lag 超过阈值的消费者组
kafka_consumergroup_lag{topic="order-events"} > 10000

# Lag 增长速率（正值表示消费跟不上生产）
rate(kafka_consumergroup_lag[5m])

# ===== Topic 吞吐 =====

# 生产速率 (msg/s)
rate(kafka_topic_partition_current_offset[1m])

# 消费速率 (msg/s)
rate(kafka_consumergroup_current_offset[1m])

# ===== Broker 健康 =====

# 在线 Broker 数量
count(kafka_server_replicamanager_leadercount)

# 未复制分区数（> 0 表示数据风险）
sum(kafka_server_replicamanager_underreplicatedpartitions)

# 离线分区数（> 0 表示数据不可用）
sum(kafka_controller_kafkacontroller_offlinepartitionscount)

# ===== 请求性能 =====

# Produce 请求延迟 P99
kafka_network_requestmetrics_totaltimems{request="Produce",quantile="0.99"}

# Fetch 请求延迟 P99
kafka_network_requestmetrics_totaltimems{request="FetchConsumer",quantile="0.99"}
```

### 告警规则

```yaml
groups:
  - name: kafka
    rules:
      - alert: KafkaConsumerGroupHighLag
        expr: |
          sum by (consumergroup, topic) (kafka_consumergroup_lag) > 50000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "消费组 {{ $labels.consumergroup }} 主题 {{ $labels.topic }} Lag: {{ $value }}"

      - alert: KafkaUnderReplicatedPartitions
        expr: sum(kafka_server_replicamanager_underreplicatedpartitions) > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka 存在未充分复制分区: {{ $value }} 个"

      - alert: KafkaOfflinePartitions
        expr: sum(kafka_controller_kafkacontroller_offlinepartitionscount) > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Kafka 存在离线分区: {{ $value }} 个，数据不可用！"

      - alert: KafkaBrokerDown
        expr: count(kafka_server_replicamanager_leadercount) < 3
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Kafka Broker 数量不足，当前: {{ $value }}"
```

---

## 案例五：业务大盘 - 电商核心指标

### Dashboard 设计

```
电商业务监控大盘

┌──────────────────────────────────────────────────────────────────┐
│  时间范围：Last 1h   刷新：30s   环境：$env   服务：$service      │
├────────────┬────────────┬────────────┬────────────┬─────────────┤
│  实时GMV   │  订单量/分 │  支付成功率│  用户在线数│  系统错误率 │
│  ¥128,450  │    342     │   98.7%    │   15,234   │   0.03%     │
│  ▲ 12%     │  ▲ 8%     │  ▼ 0.2%   │  ▲ 5%     │  ▲ 0.01%   │
├────────────┴────────────┴────────────┴────────────┴─────────────┤
│                         订单漏斗（1h）                            │
│  浏览商品 → 加入购物车 → 提交订单 → 支付成功                      │
│  100,000      45,000       12,000       11,800                   │
│  (100%)       (45%)        (12%)        (11.8%)                  │
├──────────────────────────────────────────────────────────────────┤
│  分钟级订单趋势                    分品类销售占比                  │
│  ┌────────────────────────┐       ┌────────────────────────┐     │
│  │ ~~~~~~~~~~~            │       │   [Pie Chart]           │     │
│  │       ~~~~~~~~~~~      │       │   数码: 35%             │     │
│  │                ~~~~~~~~│       │   服装: 25%             │     │
│  └────────────────────────┘       └────────────────────────┘     │
├──────────────────────────────────────────────────────────────────┤
│  接口延迟热图（按小时）           Top 10 慢接口                   │
│  ┌────────────────────────┐       ┌────────────────────────┐     │
│  │ [Heatmap]              │       │ [Table]                 │     │
│  │ 颜色深=延迟高          │       │ 接口 | P99 | 调用量     │     │
│  └────────────────────────┘       └────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 核心 PromQL

```promql
# 实时 GMV（分钟级）
sum(increase(order_amount_sum{env="prod"}[1m])) * 0.01

# 支付成功率
sum(rate(order_paid_total[5m]))
/
sum(rate(order_created_total[5m]))

# 订单转化漏斗（需要埋点多个指标）
vector(sum(increase(page_view_total{page="product"}[1h])))

# 各接口 P99 延迟
histogram_quantile(0.99,
  sum by (uri, le) (
    rate(http_server_requests_seconds_bucket{application="$app"}[5m])
  )
)

# 按 URI 分组的错误率
sum by (uri) (
  rate(http_server_requests_seconds_count{status=~"5.."}[5m])
)
/
sum by (uri) (
  rate(http_server_requests_seconds_count[5m])
)
```

---
# Part 12: 常见面试题 FAQ

## 12.1 Prometheus 基础

### Q1: Prometheus 的数据采集模式是 Pull 还是 Push？各有什么优缺点？

**答：Prometheus 默认使用 Pull（拉取）模式。**

| 维度 | Pull 模式 | Push 模式 |
|------|-----------|-----------|
| 控制权 | 监控端主动控制 | 被监控端主动推送 |
| 服务发现 | 天然支持，配置集中 | 需要知道监控端地址 |
| 防火墙 | 监控端需能访问目标 | 目标需能访问监控端 |
| 短任务 | 不适合（可能采集不到） | 适合 |
| 状态检测 | 采集失败即知道目标挂了 | 无法区分挂了还是没数据 |
| 代表产品 | Prometheus | InfluxDB, Graphite |

Prometheus 通过 **Pushgateway** 支持短任务推送，但不推荐用于长期运行服务。

---

### Q2: Prometheus 的四种指标类型及使用场景？

| 类型 | 特点 | 适用场景 |
|------|------|----------|
| **Counter** | 单调递增，不能减少，重启归零 | 请求总数、错误次数、字节数 |
| **Gauge** | 可增可减，表示当前值 | 内存使用量、温度、并发连接数 |
| **Histogram** | 预定义桶，可计算分位数（近似） | 请求延迟、响应大小（需要聚合）|
| **Summary** | 客户端计算分位数（精确但不可聚合） | 延迟分布（不需要跨实例聚合）|

**Histogram vs Summary 核心区别：**
- Histogram 在服务端计算（`histogram_quantile`），可以跨实例聚合
- Summary 在客户端计算，精确但**不能跨实例聚合**
- 生产推荐用 **Histogram**

---

### Q3: PromQL 中 rate() 和 irate() 的区别？

```promql
# rate(): 使用区间内所有数据点计算平均速率（平滑）
rate(http_requests_total[5m])

# irate(): 只使用最后两个数据点计算瞬时速率（敏感）
irate(http_requests_total[5m])
```

| 函数 | 计算方式 | 特点 | 适用场景 |
|------|----------|------|----------|
| `rate()` | 区间首尾差 / 时间 | 平滑，抗抖动 | 趋势图、告警（推荐）|
| `irate()` | 最后两点差 / 间隔 | 敏感，反映瞬时 | 实时监控、调试 |

**注意：** `irate` 时间窗口只影响数据点查找范围，不影响计算逻辑。

---

### Q4: Histogram 的 histogram_quantile() 是如何工作的？

`histogram_quantile(φ, metric_bucket)` 基于桶的计数进行线性插值：

```
原理：
1. 找到包含目标分位数（φ * total_count）的桶
2. 在该桶的上下界之间进行线性插值
3. 精度取决于桶的划分密度

例：φ=0.99，total_count=1000
→ 找第 990 个请求落在哪个桶
→ 在桶 [0.5, 1.0] 之间插值
→ 结果：0.5 + (990 - bucket_count_before) / bucket_count * 0.5
```

**影响精度的因素：**
- 桶划分越密集，精度越高
- 请求延迟超过最大桶（`+Inf`）时，分位数会被低估

---

### Q5: Prometheus 如何处理高基数（High Cardinality）问题？

**高基数问题：** 标签值数量过多，导致时间序列数量爆炸，内存/存储耗尽。

**典型错误用法：**
```python
# 错误：user_id 有百万级别的值
Counter('requests_total', labels=['user_id', 'url'])
```

**解决方案：**
1. **不要用高基数值作标签**（user_id、order_id、IP）
2. 对 URL 进行参数化（`/order/123` → `/order/{id}`）
3. 使用 `relabel_configs` 在采集时丢弃或替换标签
4. 设置 `sample_limit` 限制每个 Target 的时间序列数
5. 使用 Recording Rules 预聚合，减少查询时的序列扫描

```yaml
# prometheus.yml 中限制序列数
scrape_configs:
  - job_name: my-app
    sample_limit: 5000   # 超过则整个 scrape 失败
```

---

### Q6: Prometheus 的存储原理是什么？

```
TSDB（时序数据库）存储层次：

内存层（Head Block）
├── WAL（Write-Ahead Log）
│   └── 崩溃恢复，每个 checkpoint 128MB
└── 内存块（约2小时数据）

持久化层（磁盘 Block）
├── block_0/（0~2h）
│   ├── chunks/（实际数据，Gorilla压缩）
│   ├── index（倒排索引）
│   └── meta.json
├── block_1/（2~4h）
└── ...（每2小时一个Block）

压缩（Compaction）
└── 小 Block 合并为大 Block（最大保留时长/10）
```

**Gorilla 压缩算法：**
- 时间戳：delta-of-delta 编码（相邻时间差的差），通常只需 1-2 字节
- 值：XOR 压缩（相邻值的异或），利用时序数据变化小的特点
- 平均压缩比约 **1.37 字节/样本**（原始 16 字节节省 90%）

---

### Q7: 如何防止告警风暴（Alert Storm）？

**Alertmanager 三大机制：**

```yaml
route:
  # 1. 分组（Grouping）：同类告警合并为一条通知
  group_by: ['alertname', 'cluster']
  group_wait: 30s       # 等待同组其他告警
  group_interval: 5m    # 同组已通知后，等多久发增量

  # 2. 抑制（Inhibition）：高优先级告警抑制低优先级
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'instance']

  # 3. 静默（Silence）：维护窗口手动静默
  # 通过 Alertmanager UI 或 amtool 创建
```

**其他防风暴手段：**
- `for` 字段：指标持续 N 分钟才触发（避免毛刺）
- `repeat_interval`：告警重复通知间隔（默认4h）
- 告警路由树：不同团队只收到自己关心的告警

---

### Q8: Prometheus 联邦（Federation）是什么？适用场景？

**联邦（Federation）：** 一个全局 Prometheus 从多个局部 Prometheus 采集聚合后的数据。

```
多数据中心场景：

  DC-A                    DC-B
  ┌──────────┐            ┌──────────┐
  │Prometheus│            │Prometheus│
  │  (local) │            │  (local) │
  └────┬─────┘            └────┬─────┘
       └──────────┬────────────┘
                  ▼
         ┌────────────────┐
         │Global Prometheus│  只采集聚合指标
         │  (federation)   │  /federate?match[]=job:*
         └────────────────┘
                  │
         ┌────────▼────────┐
         │    Grafana      │  全局视图
         └─────────────────┘
```

**联邦配置：**
```yaml
scrape_configs:
  - job_name: 'federate'
    honor_labels: true
    metrics_path: '/federate'
    params:
      match[]:
        - '{job="prometheus"}'
        - '{__name__=~"job:.*"}'   # 只采集 recording rules 预聚合的指标
    static_configs:
      - targets:
        - 'prometheus-dc-a:9090'
        - 'prometheus-dc-b:9090'
```

**现代替代方案：** Thanos / Cortex / VictoriaMetrics（更适合大规模场景）

---

### Q9: 什么是 Recording Rules？为什么要用它？

**Recording Rules：** 将复杂 PromQL 预计算并存储为新的时间序列。

**使用场景：**
1. 复杂查询（多次聚合、高基数）响应慢
2. Dashboard 加载缓慢（每次请求都执行昂贵查询）
3. 跨 Prometheus 联邦时需要预聚合数据

```yaml
groups:
  - name: recording_rules
    interval: 30s    # 计算频率（默认 global.evaluation_interval）
    rules:
      # 命名规范：level:metric:operations
      - record: job:http_requests:rate5m
        expr: |
          sum by (job) (
            rate(http_requests_total[5m])
          )

      - record: job_instance:http_error_rate:ratio5m
        expr: |
          sum by (job, instance) (
            rate(http_requests_total{status=~"5.."}[5m])
          )
          /
          sum by (job, instance) (
            rate(http_requests_total[5m])
          )
```

**命名规范（官方建议）：** `level:metric_name:operations`
- `level`: 聚合级别（如 `job`, `instance`, `cluster`）
- `metric_name`: 原始指标名
- `operations`: 应用的操作（如 `rate5m`, `sum`, `ratio`）

---

### Q10: Prometheus 和 InfluxDB 的区别？

| 维度 | Prometheus | InfluxDB |
|------|------------|----------|
| 数据模型 | Metric + Labels | Measurement + Tags + Fields |
| 采集方式 | Pull（主动拉取） | Push（推送） |
| 查询语言 | PromQL | InfluxQL / Flux |
| 存储 | 本地 TSDB | 本地 / 集群版 |
| 告警 | Alertmanager | 内置 Kapacitor |
| 生态 | Grafana, Kubernetes 原生 | 企业生态 |
| 开源协议 | Apache 2.0 | MIT（v1.x），BSL（v2.x）|
| 适用场景 | 云原生、K8s 监控 | IoT、工业监控 |

---

### Q11: 如何对 Prometheus 做高可用（HA）？

**方案1：双写（简单 HA）**
```
┌─────────┐     ┌─────────┐
│Prometheus│    │Prometheus│  两个实例采集同样的 Target
│  (A)     │    │  (B)     │
└────┬─────┘    └────┬─────┘
     └────────┬───────┘
              ▼
        ┌──────────┐
        │ Grafana  │  配置两个数据源，任一可用即可
        └──────────┘
```
缺点：数据重复，无法合并查询，无长期存储。

**方案2：Thanos（推荐生产）**
```
Prometheus (A) ──┐
                 ├──► Thanos Sidecar ──► Object Storage (S3/OSS)
Prometheus (B) ──┘                           │
                                             ▼
                                       Thanos Store ──► Thanos Query
                                                              │
                                                         ┌────▼────┐
                                                         │ Grafana │
                                                         └─────────┘
```
- **Thanos Sidecar**: 与 Prometheus 部署在一起，上传数据到对象存储
- **Thanos Query**: 全局查询，自动去重（dedup）
- **Thanos Store**: 从对象存储查询历史数据
- **Thanos Compactor**: 压缩、降采样历史数据

---

### Q12: Labels 设计原则有哪些？

**好的标签设计：**
```
✅ 低基数（值的种类有限）
✅ 有意义（帮助区分/聚合）
✅ 稳定（不随时间变化）

示例：
http_requests_total{method="GET", status="200", endpoint="/api/orders"}
```

**错误的标签设计：**
```
❌ 高基数（无限增长）
❌ 使用 user_id, order_id, request_id 等
❌ 使用 IP 地址（动态变化）
❌ 标签值中包含密码、Token 等敏感信息

错误示例：
http_requests_total{user_id="12345678"}  # 百万级基数！
```

**标签命名规范：**
- 使用 `snake_case`（小写+下划线）
- 私有标签用 `__` 前缀（内部使用，不会暴露）
- 区分 `job`（服务类型）和 `instance`（具体实例）

---

### Q13: Grafana 变量（Variables）有哪些类型？

| 类型 | 说明 | 示例 |
|------|------|------|
| Query | 从数据源查询得到 | `label_values(up, job)` |
| Custom | 手动填写固定值 | `dev,test,prod` |
| Textbox | 用户自由输入 | 搜索框 |
| Constant | 固定常量 | 数据源地址 |
| Data source | 数据源选择 | 多集群切换 |
| Interval | 时间间隔 | `1m,5m,1h` |
| Ad hoc | 动态添加标签过滤 | 临时过滤器 |

**常用 Query 变量查询：**
```
# 获取所有 job 值
label_values(up, job)

# 获取特定 job 下的 instance
label_values(up{job="$job"}, instance)

# 获取所有 namespace
label_values(kube_pod_info, namespace)
```

---

### Q14: 如何调试 PromQL？

```bash
# 1. 使用 Prometheus Web UI
# 访问 http://prometheus:9090/graph
# 在 Table 模式下查看原始数据

# 2. 检查 Target 状态
# http://prometheus:9090/targets

# 3. 检查规则评估
# http://prometheus:9090/rules

# 4. 查看原始时间序列
up{job="my-service"}

# 5. 分步骤调试复杂查询
# 先查最内层
rate(http_requests_total[5m])
# 再加聚合
sum by (job) (rate(http_requests_total[5m]))
# 最后加比较
sum by (job) (rate(http_requests_total[5m])) > 100

# 6. 使用 absent() 检查指标是否存在
absent(up{job="my-service"})

# 7. 查看采集到的原始指标
curl http://my-service:8080/actuator/prometheus | grep http_requests
```

---

### Q15: Prometheus 的 `up` 指标有什么用？

```promql
# up 是 Prometheus 自动生成的健康检查指标
# 1 = 采集成功，0 = 采集失败

# 查看所有 down 的 target
up == 0

# 统计各 job 的可用率
avg by (job) (up)

# 告警：服务 down
- alert: ServiceDown
  expr: up == 0
  for: 1m
  annotations:
    summary: "{{ $labels.job }}/{{ $labels.instance }} 已下线"
```

---

## 12.2 面试场景题

### 场景题1：P99 延迟突然升高，如何排查？

```
排查步骤：

1. 确认范围
   histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))
   - 是所有接口还是特定接口？
   - 是所有实例还是单个实例？

2. 检查系统资源（Node Exporter）
   - CPU：node_cpu_seconds_total
   - 内存：node_memory_MemAvailable_bytes
   - 磁盘IO：node_disk_io_time_seconds_total
   - 网络：node_network_receive_bytes_total

3. 检查 JVM（Micrometer）
   - GC停顿：jvm_gc_pause_seconds
   - 堆内存：jvm_memory_used_bytes
   - 线程：jvm_threads_live_threads

4. 检查数据库
   - 连接池等待：hikaricp_connections_pending
   - 慢查询：mysql_global_status_slow_queries

5. 查看日志（Loki）
   {app="$service"} |= "timeout" | json | duration > 1000

6. 检查下游依赖
   - HTTP 客户端指标：http.client.requests
   - 是否某个依赖服务也开始慢了？
```

### 场景题2：设计一个高可用的生产级监控方案

```
方案架构：

指标采集层：
- Node Exporter（每台机器）
- Spring Boot Actuator（每个服务）
- MySQL/Redis/Kafka Exporter（中间件）
- Blackbox Exporter（外部探测）

存储层（高可用）：
- Prometheus × 2（双活，采集相同 target）
- Thanos Sidecar（与每个 Prometheus 同部署）
- 对象存储（OSS/S3，长期存储1年+）
- Thanos Query（全局查询，自动去重）

告警层：
- Alertmanager × 3（集群模式，gossip协议）
- 告警收敛：分组/抑制/静默
- 通知渠道：钉钉/邮件/PagerDuty

可视化层：
- Grafana × 2（负载均衡）
- Dashboard 版本化（Git 管理）
- Provisioning 自动化

日志层：
- Loki（日志聚合）
- Promtail（日志采集）
- 与 Grafana 集成，支持 Metrics-to-Logs 跳转

链路追踪：
- Tempo（链路存储）
- Micrometer Tracing（Spring Boot 集成）
- Grafana 三支柱联动
```

---
---

# 附录：速查手册

## A1: 常用端口速查

| 组件 | 默认端口 | 说明 |
|------|----------|------|
| Prometheus | 9090 | Web UI & API |
| Alertmanager | 9093 | 告警管理 |
| Pushgateway | 9091 | 推送网关 |
| Grafana | 3000 | 可视化 |
| Node Exporter | 9100 | 主机指标 |
| JMX Exporter | 5556 | Java JMX 指标 |
| Blackbox Exporter | 9115 | 探测指标 |
| MySQL Exporter | 9104 | MySQL 指标 |
| Redis Exporter | 9121 | Redis 指标 |
| Kafka Exporter | 9308 | Kafka 指标 |
| Elasticsearch Exporter | 9114 | ES 指标 |
| Spring Boot Actuator | 8080/actuator/prometheus | 应用指标 |
| Loki | 3100 | 日志存储 |
| Promtail | 9080 | 日志采集 |
| Thanos Sidecar | 10901 (gRPC), 10902 (HTTP) | Thanos 组件 |
| Thanos Query | 10904 (HTTP) | 全局查询 |

---

## A2: Prometheus 启动参数速查

```bash
prometheus \
  # 配置文件路径
  --config.file=/etc/prometheus/prometheus.yml \

  # 数据存储目录
  --storage.tsdb.path=/data/prometheus \

  # 数据保留时长（默认15天）
  --storage.tsdb.retention.time=30d \

  # 数据保留大小（超过则删除旧数据）
  --storage.tsdb.retention.size=50GB \

  # Web UI 访问路径前缀
  --web.external-url=http://prometheus.example.com \

  # 启用管理 API（删除数据等）
  --web.enable-admin-api \

  # 启用生命周期 API（热重载配置）
  --web.enable-lifecycle \

  # 监听地址
  --web.listen-address=0.0.0.0:9090 \

  # 最大并发查询数
  --query.max-concurrency=20 \

  # 查询超时
  --query.timeout=2m \

  # 远程存储（Thanos Sidecar 时使用）
  --storage.tsdb.min-block-duration=2h \
  --storage.tsdb.max-block-duration=2h
```

---

## A3: 热重载配置（无需重启）

```bash
# 方法1：发送 SIGHUP 信号
kill -HUP $(pgrep prometheus)

# 方法2：调用 /-/reload API（需要 --web.enable-lifecycle）
curl -X POST http://localhost:9090/-/reload

# Alertmanager 热重载
curl -X POST http://localhost:9093/-/reload

# 验证配置文件语法
promtool check config /etc/prometheus/prometheus.yml
promtool check rules /etc/prometheus/rules/*.yml
```

---

## A4: amtool 命令速查（Alertmanager CLI）

```bash
# 查看当前告警
amtool --alertmanager.url=http://localhost:9093 alert

# 查看已分组的告警
amtool --alertmanager.url=http://localhost:9093 alert groups

# 创建静默（维护窗口）
amtool --alertmanager.url=http://localhost:9093 silence add \
  --comment "scheduled maintenance" \
  --duration 2h \
  alertname="NodeDown" \
  instance="server-01:9100"

# 查看所有静默
amtool --alertmanager.url=http://localhost:9093 silence

# 删除静默
amtool --alertmanager.url=http://localhost:9093 silence expire <silence-id>

# 验证配置
amtool check-config /etc/alertmanager/alertmanager.yml

# 测试路由（看告警会路由到哪个 receiver）
amtool --alertmanager.url=http://localhost:9093 config routes test \
  --verify.receivers=ops-team \
  alertname="HighCPU" severity="critical"
```

---

## A5: Recording Rules 命名规范

```
格式：level:metric_name:operations

level（聚合级别）：
  instance  → 实例级别（不聚合）
  job       → 按 job 聚合
  cluster   → 按集群聚合

metric_name：
  原始指标名（去掉 _total/_bytes 等后缀）

operations（操作列表）：
  rate5m    → rate()[5m]
  sum       → sum()
  ratio     → 比率
  p99       → 99分位数

示例：
  job:http_requests:rate5m
  job_instance:http_error_rate:ratio5m
  cluster:node_cpu:avg_utilization
  job:jvm_heap_used:ratio
```

---

## A6: 告警规则单元测试

```yaml
# test/order-service-test.yml
rule_files:
  - ../rules/order-service-rules.yml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      # 模拟订单失败率超过5%
      - series: 'order_failed_total{app="order-service"}'
        values: '0 10 20 35 55'
      - series: 'order_created_total{app="order-service"}'
        values: '0 100 200 300 400'

    alert_rule_test:
      - eval_time: 5m
        alertname: HighOrderFailureRate
        exp_alerts:
          - exp_labels:
              severity: critical
            exp_annotations:
              summary: "订单失败率超过 5%"
```

```bash
# 运行测试
promtool test rules test/order-service-test.yml
```

---

## A7: 常用 Grafana Dashboard ID（Grafana.com）

| Dashboard | ID | 说明 |
|-----------|-----|------|
| Node Exporter Full | 1860 | 主机全面监控 |
| Spring Boot Statistics | 12900 | Spring Boot 监控 |
| JVM Micrometer | 4701 | JVM 详细监控 |
| MySQL Overview | 7362 | MySQL 监控 |
| Redis Dashboard | 11835 | Redis 监控 |
| Kafka Overview | 7589 | Kafka 监控 |
| Kubernetes Cluster | 6417 | K8s 集群监控 |
| Kubernetes Pods | 6336 | K8s Pod 监控 |
| Blackbox Exporter | 7587 | 探针监控 |
| Elasticsearch | 2322 | ES 监控 |
| Loki Dashboard | 13639 | 日志监控 |
| Alertmanager | 9578 | 告警管理器 |

导入方法：Dashboards → Import → 输入 ID → Load

---

## A8: PromQL 速查卡片

```promql
# ===== 聚合函数 =====
sum()          # 求和
avg()          # 平均值
min()          # 最小值
max()          # 最大值
count()        # 计数
stddev()       # 标准差
topk(n, expr)  # 最大的 n 个
bottomk(n, expr) # 最小的 n 个

# ===== 常用函数 =====
rate(counter[5m])       # 平均速率
irate(counter[5m])      # 瞬时速率
increase(counter[5m])   # 增量
delta(gauge[5m])        # 变化量
deriv(gauge[5m])        # 导数（线性回归）
predict_linear(gauge[1h], 3600)  # 线性预测
histogram_quantile(0.99, rate(hist_bucket[5m]))  # 分位数
absent(metric)          # 指标不存在时返回1
changes(gauge[1h])      # 值变化次数
resets(counter[1h])     # 计数器重置次数
label_replace(metric, "dst", "$1", "src", "(.*)")  # 标签替换
round(expr, 0.01)       # 四舍五入

# ===== 时间函数 =====
time()         # 当前 Unix 时间戳
timestamp(metric)  # 指标样本的时间戳
minute()       # 当前分钟 (0-59)
hour()         # 当前小时 (0-23)
day_of_week()  # 星期几 (0=周日)
days_in_month() # 当月天数

# ===== 二元运算符 =====
and     # 集合交集
or      # 集合并集
unless  # 集合差集
on()    # 指定 join label
ignoring()  # 忽略某些 label
group_left()   # 多对一 join（左边多）
group_right()  # 一对多 join（右边多）

# ===== 常见模式 =====
# 错误率
rate(errors[5m]) / rate(total[5m])

# 使用率
used / total

# 增长率（同比）
(current - offset_value) / offset_value
(metric - metric offset 1d) / metric offset 1d

# 预测磁盘满的时间（小时）
predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0

# 分位数
histogram_quantile(0.99,
  sum by (le, job) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

---

## A9: Docker Compose 一键启动完整监控栈

```yaml
# docker-compose.yml（生产就绪版）
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus-data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager:/etc/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-piechart-panel
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  loki:
    image: grafana/loki:2.9.0
    container_name: loki
    restart: unless-stopped
    ports:
      - "3100:3100"
    volumes:
      - ./loki:/etc/loki
      - loki-data:/loki
    command: -config.file=/etc/loki/config.yaml
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:2.9.0
    container_name: promtail
    restart: unless-stopped
    volumes:
      - /var/log:/var/log:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - ./promtail:/etc/promtail
    command: -config.file=/etc/promtail/config.yaml
    depends_on:
      - loki
    networks:
      - monitoring

  pushgateway:
    image: prom/pushgateway:v1.6.2
    container_name: pushgateway
    restart: unless-stopped
    ports:
      - "9091:9091"
    networks:
      - monitoring

volumes:
  prometheus-data:
  grafana-data:
  loki-data:

networks:
  monitoring:
    driver: bridge
```

```bash
# 启动命令
docker-compose up -d

# 查看状态
docker-compose ps

# 查看日志
docker-compose logs -f prometheus

# 停止
docker-compose down
```

---

## A10: 关键概念总结

```
Prometheus 核心概念速记：

┌─────────────────────────────────────────────────────────────────┐
│  指标命名：namespace_subsystem_name_unit                         │
│  示例：http_server_requests_seconds_total                        │
│                                                                  │
│  四种类型：                                                      │
│  Counter(单调增) → rate/increase                                 │
│  Gauge(上下浮动) → 直接使用                                      │
│  Histogram(桶分布) → histogram_quantile                          │
│  Summary(客户端分位数) → 直接使用，不可聚合                       │
│                                                                  │
│  时间序列标识：metric_name{label="value",...} @ timestamp        │
│                                                                  │
│  核心函数：                                                      │
│  rate() > irate()（平滑 vs 瞬时）                               │
│  histogram_quantile(0.99, rate(hist_bucket[5m]))                 │
│  predict_linear() / absent()                                     │
│                                                                  │
│  告警三要素：expr + for + labels/annotations                     │
│  Alertmanager：分组 + 抑制 + 静默 = 防告警风暴                   │
│                                                                  │
│  生产 HA：Prometheus × 2 + Thanos + 对象存储                    │
│  可观测性：Metrics(Prometheus) + Logs(Loki) + Traces(Tempo)      │
└─────────────────────────────────────────────────────────────────┘
```

---

*文档版本：v1.0 | 最后更新：2024年*
*覆盖内容：Prometheus 2.x, Grafana 10.x, Alertmanager 0.26.x, Loki 2.9.x*
