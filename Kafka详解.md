# Kafka 详解 - 从零到精通

> 本文档涵盖 Apache Kafka 从基础概念到高级特性的完整技术解析，适合初学者入门和进阶工程师深度学习。
> 文档版本：基于 Kafka 3.x，兼容 2.x

---

# 目录

- [Part 1: Kafka 整体架构](#part-1-kafka-整体架构)
- [Part 2: 生产者（Producer）深度解析](#part-2-生产者producer深度解析)
- [Part 3: Broker 存储架构深度解析](#part-3-broker-存储架构深度解析)
- [Part 4: 副本机制与高可用](#part-4-副本机制与高可用)
- [Part 5: 消费者（Consumer）深度解析](#part-5-消费者consumer深度解析)
- [Part 6: Exactly-Once 语义](#part-6-exactly-once-语义)
- [Part 7: Kafka 集群管理](#part-7-kafka-集群管理)
- [Part 8: Spring Boot 集成实战](#part-8-spring-boot-集成实战)
- [Part 9: 完整实战案例](#part-9-完整实战案例)
- [Part 10: 性能调优](#part-10-性能调优)
- [Part 11: 监控与运维](#part-11-监控与运维)
- [Part 12: 常见面试题 FAQ](#part-12-常见面试题-faq)

---

# Part 1: Kafka 整体架构

## 1.1 Kafka 是什么

Apache Kafka 是一个由 LinkedIn 开发并于 2011 年开源的**分布式流处理平台**。它被设计为高吞吐量的分布式发布-订阅消息系统，后演变为完整的流处理平台。

Kafka 的核心定位：
1. **消息队列（Message Queue）**：支持发布-订阅和点对点两种消息模式
2. **存储系统（Storage System）**：以持久化、可复制、容错方式存储数据流
3. **流处理平台（Stream Processing Platform）**：通过 Kafka Streams API 实时处理数据流

### 消息队列的三大核心应用场景

#### 场景一：异步处理

```
传统同步模式（串行调用，延迟叠加）：
用户注册 --> [写DB 50ms] --> [发邮件 300ms] --> [发短信 200ms] --> 返回响应(总计550ms+)

引入 Kafka 异步模式（并行处理）：
用户注册 --> [写DB 50ms] --> 写入Kafka --> 立即返回响应(约60ms)
                                  |
               [邮件服务消费]  [短信服务消费]（异步并行执行，不阻塞用户）
```

#### 场景二：系统解耦

```
强耦合模式（直接调用）：
                 +-------------+
                 |  订单服务   |
                 +------+------+
                        | 直接 RPC（同步调用，强依赖）
          +-------------+-------------+
          v             v             v
    +----------+  +----------+  +----------+
    | 库存服务 |  | 积分服务 |  | 通知服务 |
    +----------+  +----------+  +----------+
    库存服务宕机 -> 订单服务调用失败 -> 整个下单流程中断！

解耦模式（通过 Kafka 发布-订阅）：
                 +-------------+
                 |  订单服务   | --> 发布 OrderCreated 事件到 Kafka
                 +-------------+    （不关心谁在监听，完全解耦）
                        |
                 +-------+-------+
                 |    Kafka      |
                 +--+--+--+--+--+
                    |  |  |
          +---------+  |  +----------+
          v            v             v
    +----------+  +----------+  +----------+
    | 库存服务 |  | 积分服务 |  | 通知服务 |
    +----------+  +----------+  +----------+
    各服务独立消费，库存服务宕机不影响订单创建，消息会堆积等待恢复后处理
    新增服务（如风控服务）只需订阅 Topic，无需修改订单服务
```

#### 场景三：流量削峰

```
无削峰（瞬间洪峰直接打垮下游）：

请求量   ##################
         ##################  <- 峰值10万QPS,超过DB上限(1万QPS)，崩溃！
         ##################
DB容量   ==================
时间轴   ------------------>

有 Kafka 削峰（削峰填谷）：

请求量   ##################
         ##################  -> 超出的请求堆积在 Kafka 中（不丢弃）
         ##################
DB容量   ==================  -> DB 匀速处理（始终在 1万QPS 以内）
         ==================
         ==================  -> 高峰后继续消费积压（填谷）
时间轴   ------------------>
```

---

## 1.2 Kafka 整体架构图

```
+=============================================================================+
|                         Apache Kafka 整体架构                               |
+=============================================================================+

  生产者集群                      Kafka Broker 集群                  消费者集群
  +----------+                 +=========================+          +-----------------+
  |Producer 1|--------+        |  +------------------+  |     +--> |Consumer Group A |
  +----------+        |        |  |    Broker 1       |  |     |   |  +-----------+  |
                      |        |  | +--------------+  |  |     |   |  |Consumer 1 |  |
  +----------+        +------> |  | |order-events  |  |  | ----+   |  +-----------+  |
  |Producer 2|--------+        |  | | Partition 0  |  |  |     |   |  +-----------+  |
  +----------+        |        |  | | Partition 1  |  |  |     |   |  |Consumer 2 |  |
                      |        |  | +--------------+  |  |     |   |  +-----------+  |
  +----------+        |        |  +------------------+  |     |   +-----------------+
  |Producer 3|--------+        |                        |     |
  +----------+                 |  +------------------+  |     |   +-----------------+
                               |  |    Broker 2       |  |     +-->|Consumer Group B |
  +------------------+         |  | +--------------+  |  |        |  +-----------+  |
  |   ZooKeeper  OR  | <-----> |  | |order-events  |  |  | ------>|  |Consumer 3 |  |
  |   KRaft 集群     |         |  | | Partition 2  |  |  |        |  +-----------+  |
  | (元数据管理)     |         |  | | Partition 3  |  |  |        |  +-----------+  |
  +------------------+         |  | +--------------+  |  |        |  |Consumer 4 |  |
                               |  +------------------+  |        |  +-----------+  |
                               |                        |        +-----------------+
                               |  +------------------+  |
                               |  |    Broker 3       |  |
                               |  | +--------------+  |  |
                               |  | |order-events  |  |  |
                               |  | | Partition 4  |  |  |
                               |  | | Partition 5  |  |  |
                               |  | +--------------+  |  |
                               |  +------------------+  |
                               +=========================+

核心交互说明：
  1. Producer 将消息发送到指定 Topic 某 Partition 的 Leader
  2. Leader 写入本地日志，Follower 副本异步同步（ISR 机制）
  3. 同一 Consumer Group 共同消费：每个 Partition 只被一个 Consumer 消费
  4. 不同 Consumer Group 独立消费，都能收到全量消息（广播效果）
  5. ZooKeeper/KRaft 管理集群元数据（Broker地址、Topic分区信息、Leader等）
```

---

## 1.3 核心概念详解

### 1.3.1 Topic（主题）

Topic 是 Kafka 中消息的逻辑分类单元，类似数据库中的表。

| 特性 | 说明 |
|------|------|
| 逻辑概念 | 物理上由多个 Partition 实现 |
| 唯一性 | Topic 名称在集群内唯一 |
| 多写多读 | 多 Producer 同时写，多 Consumer Group 同时读 |
| 独立配置 | 每个 Topic 可独立配置保留策略、分区数、副本数等 |

### 1.3.2 Partition（分区）

Partition 是 Kafka 并发和扩展的核心机制，每个 Partition 是**有序、不可变**的消息序列。

```
Topic: order-events（3个分区）

Partition 0（只追加写，offset 单调递增）:
  +-----+-----+-----+-----+-----+-----+
  |  0  |  1  |  2  |  3  |  4  |  5  |  --> 持续追加...
  +-----+-----+-----+-----+-----+-----+
  ^                              ^
  最旧消息（可能被清理）           LEO（Log End Offset）= 6

Partition 1:
  +-----+-----+-----+-----+
  |  0  |  1  |  2  |  3  |  --> 持续追加...
  +-----+-----+-----+-----+

Partition 2:
  +-----+-----+-----+-----+-----+
  |  0  |  1  |  2  |  3  |  4  |  --> 持续追加...
  +-----+-----+-----+-----+-----+
```

**Partition 的关键特性**：
- **有序性**：同一 Partition 内，消息按写入顺序严格排列
- **不可变性**：已写入的消息不能修改，只能追加（Append-Only）
- **独立 Offset**：每个 Partition 的 offset 从 0 开始独立计数
- **分布式**：不同 Partition 可分布在不同 Broker，实现负载均衡

### 1.3.3 Replica（副本）、Leader、Follower

```
Topic: order-events，6个Partition，3副本

副本分布图（AR = Assigned Replicas）：
+=========+===========+===========+===========+
|Partition |  Broker 1  |  Broker 2  |  Broker 3  |
+=========+===========+===========+===========+
|   P0    |  [Leader] | [Follower]| [Follower]|
+---------+-----------+-----------+-----------+
|   P1    | [Follower]|  [Leader] | [Follower]|
+---------+-----------+-----------+-----------+
|   P2    | [Follower]| [Follower]|  [Leader] |
+---------+-----------+-----------+-----------+
|   P3    |  [Leader] | [Follower]| [Follower]|
+---------+-----------+-----------+-----------+
|   P4    | [Follower]|  [Leader] | [Follower]|
+---------+-----------+-----------+-----------+
|   P5    | [Follower]| [Follower]|  [Leader] |
+---------+-----------+-----------+-----------+

每个 Broker 各是 2 个 Partition 的 Leader，负载均衡！

Leader 职责：处理 Producer 写入和 Consumer 读取（所有读写请求）
Follower 职责：异步从 Leader 同步数据，不处理客户端请求（备份用）

注意：Kafka 2.4+ 支持 Follower 提供读服务（replica.selector.class），
但默认仍由 Leader 处理所有请求（简化一致性模型）
```

### 1.3.4 ISR、OSR、AR

```
AR (Assigned Replicas) = ISR + OSR

AR：某个 Partition 的所有副本集合（创建时确定，通常不变）

ISR (In-Sync Replicas)：与 Leader 保持同步的副本集合（动态变化！）
  判断标准：副本距上次成功 Fetch 的时间 < replica.lag.time.max.ms（默认30s）
  ISR 中的副本：数据最多落后 Leader 30s 内
  ISR 的重要性：acks=-1 时，Leader 等待 ISR 全部写入才响应 Producer

OSR (Out-of-Sync Replicas)：同步滞后的副本集合
  触发条件：Follower 超过 30s 未向 Leader 发送 Fetch 请求
  从 OSR 重新加入 ISR 条件：Follower 追上 Leader 的 HW（高水位）

动态变化示例：
  正常：ISR={B1(Leader), B2, B3}，OSR={}
  B3 网络故障：ISR={B1, B2}，OSR={B3}
  B3 恢复并追上进度：ISR={B1, B2, B3}，OSR={}
```

---

## 1.4 Broker 与集群元数据管理

### Broker 核心配置

```bash
# server.properties（Kafka Broker 配置文件）

# 唯一 Broker ID
broker.id=1

# 监听地址
listeners=PLAINTEXT://0.0.0.0:9092
advertised.listeners=PLAINTEXT://kafka-1.example.com:9092

# 日志存储（可配置多个磁盘，Kafka 轮询使用，提升 IO）
log.dirs=/data/kafka1,/data/kafka2,/data/kafka3

# 默认 Topic 配置
num.partitions=3
default.replication.factor=3
min.insync.replicas=2

# ZooKeeper 连接（传统模式）
zookeeper.connect=zk1:2181,zk2:2181,zk3:2181/kafka

# 网络线程数（处理网络IO）
num.network.threads=8
# 磁盘IO线程数
num.io.threads=16
```

### ZooKeeper vs KRaft 对比

```
ZooKeeper 模式（Kafka < 2.8）：
  +------------------+     Watch/Notify     +------------------+
  |  Kafka Broker    | <------------------> |   ZooKeeper      |
  |  集群            |                      |   集群           |
  +------------------+                      +------------------+
  问题：
  - 两套系统，运维复杂度加倍
  - ZooKeeper 存储元数据有上限（约200万分区）
  - Controller 选举慢（需要从 ZooKeeper 读取完整状态）
  - ZooKeeper 成为性能瓶颈

KRaft 模式（Kafka >= 2.8，3.x 成熟）：
  +------------------+     Raft 共识        +------------------+
  |  Kafka Broker    | <------------------> | KRaft Controller |
  |  集群            |                      | （内置于 Broker） |
  +------------------+                      +------------------+
  优势：
  - 单一系统，无 ZooKeeper 依赖
  - 支持百万级分区
  - Controller 选举更快（毫秒级）
  - 统一安全配置
  - 可以 Controller 和 Broker 合一（小集群）或分开（大集群）
```

---

## 1.5 Kafka 与其他消息队列对比

| 特性 | Kafka | RocketMQ | RabbitMQ | ActiveMQ |
|------|-------|----------|----------|----------|
| **开发语言** | Scala/Java | Java | Erlang | Java |
| **单机吞吐量** | 百万级/s | 十万级/s | 万级/s | 万级/s |
| **消息延迟** | 毫秒级 | 微秒级 | 微秒级 | 毫秒级 |
| **高可用** | 分布式多副本 | 分布式多副本 | 主从复制 | 主从复制 |
| **消息可靠性** | 可配置（acks=0/1/-1） | 默认不丢 | 基本不丢 | 有丢失概率 |
| **消费模式** | Pull（拉取） | Push+Pull | Push（推送） | Push（推送） |
| **顺序消息** | 分区内有序 | 分区内有序 | 不支持 | 不支持 |
| **消息回溯** | 支持（offset/时间） | 支持 | 不支持 | 不支持 |
| **消息堆积** | 优秀（TB级磁盘存储） | 良好 | 差（影响性能） | 差 |
| **事务消息** | 支持 | 原生支持 | 不支持 | 支持 |
| **延迟消息** | 不原生支持 | 原生支持 | 插件支持 | 支持 |
| **社区活跃度** | 极高（Apache 顶级） | 高（Apache） | 高 | 低 |
| **适用场景** | 大数据/日志/流处理 | 电商/金融 | 企业集成 | 传统SOA |

**选型建议**：
- **日志收集、大数据管道、高吞吐场景** → Kafka（LinkedIn/Netflix/Uber 的选择）
- **电商订单、金融交易等强一致性场景** → RocketMQ（阿里双11经验证）
- **企业应用集成、复杂路由规则** → RabbitMQ（AMQP 标准）

---

# Part 2: 生产者（Producer）深度解析

## 2.1 生产者发送完整流程

理解 Producer 的内部工作机制，是调优和排查问题的基础。

### 发送流程图

```
+==========================================================================+
|                    Producer 完整消息发送流程                               |
+==========================================================================+

【应用线程】调用 producer.send(ProducerRecord, Callback)

Step 1: ProducerInterceptors（拦截器链）
  onSend(record) 依次调用，可修改 record
  常见用途：添加 TraceId Header、统计发送指标、数据脱敏
        |
Step 2: Serializer（序列化）
  key.serializer:   key   Object  -->  byte[]
  value.serializer: value Object  -->  byte[]
        |
Step 3: Partitioner（分区计算）
  指定 partition?  YES --> 直接使用
  有 key?          YES --> murmur2(keyBytes) % numPartitions
  否则（无key）     --> UniformStickyPartitioner（粘性分区）
        |
Step 4: RecordAccumulator（消息累积缓冲区）
  将消息追加到对应 TopicPartition 的 ProducerBatch
  batch.size 满 OR linger.ms 超时 --> 标记为就绪（Ready）
  buffer.memory 不足 --> 阻塞等待（最多 max.block.ms）
        |
【Sender 线程】（后台线程，独立于应用线程）

Step 5: Sender 轮询就绪 Batch
  从 RecordAccumulator 取出就绪的 Batch，按 Broker 分组
  检查元数据是否可用（Topic 分区的 Leader 地址）
        |
Step 6: 发送 ProduceRequest
  通过 NetworkClient（基于 Selector/NIO）建立 TCP 连接
  将 Batch 封装为 ProduceRequest（可合并多个 Partition 的 Batch）
  InFlightRequests 追踪未响应请求（最多 max.in.flight.requests.per.connection）
        |
Step 7: Broker 处理（Leader）
  校验：格式、权限、大小限制
  追加到 Partition Log（顺序写 Page Cache）
  根据 acks 配置等待 ISR 副本确认
  返回 ProduceResponse（成功的 offset 或 ErrorCode）
        |
Step 8: 处理响应
  成功 --> 回调 Callback(RecordMetadata, null)
          回调 Interceptor.onAcknowledgement
  可重试错误 --> 等待 retry.backoff.ms，重新入队重试
  不可重试错误 --> 回调 Callback(null, Exception)
  超过 retries --> 回调 Callback(null, Exception)
```

### 关键参数汇总

```java
// 创建生产者的完整配置示例
Properties props = new Properties();
// 必填
props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka1:9092,kafka2:9092");
props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

// 可靠性配置
props.put(ProducerConfig.ACKS_CONFIG, "all");              // -1/all=最高可靠
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true); // 启用幂等
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE); // 无限重试

// 吞吐量配置
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 32768);         // 32KB batch
props.put(ProducerConfig.LINGER_MS_CONFIG, 5);              // 等 5ms 积累
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864L);  // 64MB 缓冲
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "zstd");  // 压缩

// 拦截器
props.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, "com.example.MyInterceptor");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
```

---

## 2.2 分区策略详解

### 2.2.1 内置分区策略

```
策略 1：指定 Partition（优先级最高）
  new ProducerRecord("topic", 2, key, value)  // 直接发到 Partition 2

策略 2：Key Hash（有 key 时）
  计算公式：Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions
  
  例：4个分区
    key="order-001" hash -> 12345 % 4 = 1 -> P1
    key="order-002" hash -> 67891 % 4 = 3 -> P3
    key="order-001" hash -> 12345 % 4 = 1 -> P1（同key总在同分区，保证顺序）
  
  注意：增加分区数后，同一 key 的路由分区会改变！
  key="order-001" hash -> 12345 % 8 = 5 -> P5（从P1变成P5了！）
  因此，对顺序敏感的 Topic 不能随意增加分区数。

策略 3：粘性分区（无 key，Kafka 2.4+ 默认）
  随机选择一个分区作为"粘性分区"
  在该分区的 batch 满（>= batch.size）或 linger.ms 到期前
  持续发到同一分区（减少小 batch，提升吞吐量）
  之后随机切换到另一个分区

  vs 旧版轮询（DefaultPartitioner，Kafka 2.4 前）：
    每条消息轮询换分区 -> 产生大量 1条消息的小 batch
    网络 RTT 多，压缩效率低
```

### 2.2.2 自定义分区器

```java
/**
 * 示例：按消息优先级分区
 * VIP 用户 -> P0~P1（独享，分配专属高速消费者）
 * 普通用户 -> P2~P5（共享）
 */
public class PriorityPartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        int total = cluster.partitionCountForTopic(topic);
        int vipEnd = total / 4;  // 前 1/4 分区给 VIP

        String keyStr = (key != null) ? key.toString() : "";

        if (keyStr.startsWith("VIP_")) {
            // VIP 消息路由到前 vipEnd 个分区
            if (keyBytes == null) return 0;
            return Utils.toPositive(Utils.murmur2(keyBytes)) % vipEnd;
        } else {
            // 普通消息路由到剩余分区
            if (keyBytes == null) {
                return vipEnd + ThreadLocalRandom.current().nextInt(total - vipEnd);
            }
            return vipEnd + Utils.toPositive(Utils.murmur2(keyBytes)) % (total - vipEnd);
        }
    }

    @Override
    public void close() {}

    @Override
    public void configure(Map<String, ?> configs) {}
}
```

---

## 2.3 RecordAccumulator 批次机制

```
RecordAccumulator 内存结构：

总容量 = buffer.memory（默认 32MB）
使用内存 >= buffer.memory 时，send() 阻塞（最多 max.block.ms=60s）

内部结构：
ConcurrentMap<TopicPartition, Deque<ProducerBatch>>

TP(orders,0) --> Deque: [Batch(16KB,全满)] --> [Batch(8KB,写入中)]
TP(orders,1) --> Deque: [Batch(3KB,写入中)]
TP(users, 0) --> Deque: [Batch(16KB,全满)] --> [Batch(16KB,全满)] --> [Batch(1KB,写入中)]
                                                    ^
                                               这些已满的 batch 等待 Sender 线程发送

ProducerBatch 触发发送的条件：
  1. 当前 batch 字节数 >= batch.size（默认 16384 = 16KB）
  2. 自上次发送经过 linger.ms（默认 0ms，即立即发送）
  3. 调用了 producer.flush()（强制立即发送所有积累的消息）
  4. 调用了 producer.close()（关闭前发送所有消息）
  5. RecordAccumulator 内存不足，强制触发发送

ProducerBatch 内部格式（RecordBatch v2）：
+========================================+
| RecordBatch Header（共61字节）          |
|  baseOffset: 0（由 Broker 分配真实值）  |
|  magic: 2                              |
|  attributes: 压缩类型 | 时间戳类型       |
|  producerId: PID（幂等性）              |
|  producerEpoch: epoch（事务）           |
|  baseSequence: SeqNum（幂等性）         |
|  numRecords: batch 内消息数            |
|  crc: CRC32C 校验码                    |
+========================================+
| Record 1: offsetDelta=0, key, value   |
| Record 2: offsetDelta=1, key, value   |
| ...（如果开启压缩，Records 部分被压缩） |
+========================================+
```

### 重要参数调优表

| 参数 | 默认值 | 高吞吐配置 | 低延迟配置 | 说明 |
|------|--------|-----------|-----------|------|
| `batch.size` | 16384 | 131072 | 4096 | batch 最大字节数 |
| `linger.ms` | 0 | 20 | 0 | 等待时间（积累更多消息） |
| `buffer.memory` | 33554432 | 134217728 | 33554432 | 总缓冲区大小 |
| `compression.type` | none | lz4/zstd | none | 压缩算法 |
| `max.in.flight.requests.per.connection` | 5 | 5 | 1 | 未响应请求数（幂等时<=5） |

---

## 2.4 三种消息发送模式

### 发后即忘（Fire and Forget）

```java
// 最高性能，可能丢消息（acks=0时）
producer.send(new ProducerRecord<>("access-log", "user:123", "login"));
// 不处理 Future，不关心结果
// 适用：日志收集、监控上报（少量丢失可接受，吞吐量优先）
```

### 同步发送（Synchronous）

```java
// 阻塞等待 Broker 确认，吞吐量最低，可靠性最高
try {
    ProducerRecord<String, String> record =
        new ProducerRecord<>("order-events", "order-001", "创建订单");

    Future<RecordMetadata> future = producer.send(record);
    RecordMetadata metadata = future.get(30, TimeUnit.SECONDS); // 阻塞！

    System.out.printf("发送成功: topic=%s, partition=%d, offset=%d%n",
        metadata.topic(), metadata.partition(), metadata.offset());

} catch (TimeoutException e) {
    System.err.println("发送超时，网络或 Broker 问题");
} catch (ExecutionException e) {
    Throwable cause = e.getCause();
    if (cause instanceof RecordTooLargeException) {
        System.err.println("消息过大，超过 max.message.bytes");
    } else {
        System.err.println("发送失败: " + cause.getMessage());
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
// 适用：关键业务（支付、转账），必须确认写入成功才继续
// 缺点：串行等待，单线程吞吐量约 100~1000 msg/s
```

### 异步发送（Asynchronous with Callback）

```java
// 不阻塞，通过 Callback 处理结果，吞吐量最高
// 注意：Callback 在 Sender 线程中调用，不要做耗时操作！

producer.send(record, (metadata, exception) -> {
    if (exception == null) {
        // 发送成功
        log.debug("发送成功: partition={}, offset={}",
            metadata.partition(), metadata.offset());
    } else {
        // 发送最终失败（已经过全部重试）
        log.error("发送失败（已重试{}次）", retries, exception);

        // 常见补偿措施：
        // 1. 写入本地磁盘文件，定时重试
        // 2. 写入 DB dead_letter 表，人工处理
        // 3. 发送告警通知
        failedMessages.offer(record);
    }
});
// 适用：高并发场景（日志流水、事件上报），需要感知结果但不阻塞主流程
```

---

## 2.5 ACK 机制与消息可靠性

```
acks 参数控制 Producer 需要多少副本确认才认为发送成功。

acks=0（不等待确认）：
  Producer --> Broker（发完即认为成功）
  Broker 可能崩溃 -> 消息丢失，Producer 无感知
  场景：传感器数据、可丢的日志（最高吞吐量）

acks=1（Leader 确认，默认）：
  Producer --> Leader（写入成功，立即 ACK）
                 |
           --> Follower 1  （异步复制中...）
           --> Follower 2  （异步复制中...）
  风险：Leader 在 ACK 后，Follower 同步前崩溃 -> 消息丢失

acks=-1 或 acks=all（ISR 全部确认）：
  Producer --> Leader（写入）
                 |
  Leader 等待所有 ISR 副本确认：
           --> Follower 1 确认 +
           --> Follower 2 确认 +
           = 所有 ISR 确认后，Leader 才 ACK Producer

  配合 min.insync.replicas=2（必须至少2个ISR副本写入才返回成功）：
    即使 Leader 立即崩溃，至少还有 1 个 Follower 有完整数据
    新 Leader 选举后，消息不丢失

性能比较（1KB 消息，6 Partition，3副本）：
  acks=0:   ~3,000,000 msg/s
  acks=1:   ~1,200,000 msg/s
  acks=-1:  ~300,000 msg/s（受 ISR 同步延迟影响）
```

---

## 2.6 幂等生产者（Idempotent Producer）

### 问题：重试导致消息重复

```
网络故障导致消息重复的场景（未启用幂等性）：

Producer（seq=5）                    Broker（Leader）
      |                                    |
      |------ 发送 msg（seq=5）  --------->|  写入成功，offset=1000
      |                                    |
      |          ACK 在网络中丢失！         |
      |                                    |
      | （Producer 等待超时，自动重试）      |
      |------ 重试 msg（seq=5）  --------->|  再次写入！offset=1001
      |<---------- ACK（成功） ------------|
      
最终：offset=1000 和 offset=1001 存有相同的消息！
消费者会消费到两条重复消息，导致业务异常。
```

### 解决：PID + Sequence Number 去重

```
启用幂等性后（enable.idempotence=true）：

Broker 维护每个 (PID, Partition) 的期望序列号：
  expectedSeq[(PID=1001, P=0)] = 0  (初始值)

发送流程：
  第一次发送 msg（PID=1001, seq=0）：
    Broker 检查：seq(0) == expected(0)? YES -> 写入，expected -> 1

  ACK 丢失，Producer 重试（PID=1001, seq=0）：
    Broker 检查：seq(0) < expected(1)? YES -> 重复消息，丢弃！返回 ACK（成功）

  第二条消息（PID=1001, seq=1）：
    Broker 检查：seq(1) == expected(1)? YES -> 写入，expected -> 2

  消息乱序（PID=1001, seq=5，跳过了中间序号）：
    Broker 检查：seq(5) > expected(2)+1? YES -> 返回 OutOfOrderSequenceException
    Producer 会触发 Fatal 错误（需要关闭重建 Producer）

幂等性的局限性：
  1. 只保证同一 (PID, Partition) 维度的去重
  2. 只在单个 Producer 会话内有效
     （重启 Producer 会获取新 PID，旧序列号信息失效）
  3. 不能跨分区保证原子性（需要事务实现）
```

```java
// 最简单的幂等性配置（推荐生产环境使用）
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
// 启用幂等性会自动强制以下配置：
// acks = -1（必须）
// retries = Integer.MAX_VALUE（必须）
// max.in.flight.requests.per.connection <= 5（保证有序性）
// 若手动设置了不兼容的值（如 acks=1），会抛出 ConfigException
```

---

## 2.7 事务消息

### 事务使用场景

```
场景：下单操作原子性修改多个 Topic

不使用事务（部分成功，数据不一致）：
  写入 order-events（成功）
  写入 inventory-events（Broker 超时！失败）
  --> 订单创建了，但库存未扣减 -> 超卖！

使用 Kafka 事务：
  beginTransaction()
  写入 order-events
  写入 inventory-events
  commitTransaction()  --> 两个写入对 read_committed 消费者原子可见

  如果 inventory-events 写入失败：
  abortTransaction()   --> 两个写入对消费者都不可见（完全回滚）
```

### 事务 API 完整示例

```java
@Service
public class OrderTransactionService {

    private final KafkaProducer<String, String> txProducer;

    @PostConstruct
    public void init() {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        // 事务 ID 必须：唯一 + 固定（同一实例重启使用相同 ID，用于事务恢复）
        props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-tx-001");

        this.txProducer = new KafkaProducer<>(props);
        // 向 TransactionCoordinator 注册，获取 PID 和 epoch
        // 若有相同 transactional.id 的未完成事务，此处会触发恢复或中止
        this.txProducer.initTransactions();
    }

    public void createOrder(String orderId, String productId, int qty) {
        try {
            txProducer.beginTransaction(); // 本地状态变为 ONGOING

            // 在事务中发送消息（这些消息对 read_committed 消费者暂时不可见）
            txProducer.send(new ProducerRecord<>(
                "order-events", orderId, toJson(orderId, "CREATED")));
            txProducer.send(new ProducerRecord<>(
                "inventory-events", productId, toJson(productId, qty)));
            txProducer.send(new ProducerRecord<>(
                "notification-events", orderId, toJson(orderId, "SMS_NOTIFY")));

            // 提交事务：TransactionCoordinator 写入 PREPARE_COMMIT
            //           再向各 Partition 写入 COMMIT marker
            //           消费者现在可以看到这些消息
            txProducer.commitTransaction();

        } catch (ProducerFencedException | OutOfOrderSequenceException e) {
            // 不可恢复：同 transactional.id 有更新的 Producer 实例（zombie fencing）
            // 或序列号严重乱序，直接关闭
            txProducer.close();
            throw new RuntimeException("Producer 异常，已关闭", e);
        } catch (KafkaException e) {
            // 可恢复（网络超时等）：中止当前事务，可重试整个业务逻辑
            txProducer.abortTransaction();
            // 向各 Partition 写入 ABORT marker，消费者看不到这些消息
            throw new RetryableException("事务失败，请重试", e);
        }
    }

    private String toJson(String id, String action) {
        return String.format("{\"id\":\"%s\",\"action\":\"%s\"}", id, action);
    }

    @PreDestroy
    public void destroy() {
        txProducer.close();
    }
}
```

### 事务内部流程

```
TransactionCoordinator = __transaction_state 对应分区的 Leader Broker
分配公式：abs(hash(transactional.id)) % __transaction_state.num.partitions（默认50）

完整流程：
Producer                TransactionCoordinator      Broker（各Partition）
   |                          |                           |
   |--initTransactions------->|  注册 transactional.id    |
   |<--PID=1001, epoch=5------|  递增 epoch（Fencing旧实例）|
   |                          |                           |
   |--beginTransaction------->|  TC 本地：state=ONGOING   |
   |                          |                           |
   |--send(order-topic,P0)------------------------------------>|写入
   |--addPartitionsToTxn----->| 记录事务涉及的 Partition   |
   |                          |                           |
   |--send(inv-topic,P0)-------------------------------------->|写入
   |--addPartitionsToTxn----->| 更新分区列表               |
   |                          |                           |
   |--commitTransaction------>|                           |
   |                          |--写 PREPARE_COMMIT------->|持久化！
   |                          |  (状态变为 PREPARE_COMMIT) |
   |                          |                           |
   |                          |--WriteTxnMarkers(COMMIT)-->|每个分区写 COMMIT 标记
   |                          |<--OK----------------------|
   |                          |                           |
   |                          |--写 COMPLETE_COMMIT------>|清理事务状态
   |<--OK---------------------|                           |
```

---

## 2.8 消息压缩详解

```
压缩在 RecordBatch 级别进行（一个 Batch 整体压缩，而非逐条）
Batch 中消息越多，数据相关性越高，压缩效果越好
（这也是增大 batch.size 和 linger.ms 的另一个好处）

压缩算法对比（100KB JSON 数据，batch=100条消息）：
  原始大小: 100KB
  gzip:     12KB  (压缩比 8.3x, CPU消耗高, 速度慢)
  snappy:   25KB  (压缩比 4.0x, CPU消耗中, 速度快)
  lz4:      28KB  (压缩比 3.6x, CPU消耗低, 速度最快)
  zstd:     18KB  (压缩比 5.6x, CPU消耗中, 速度快) <- 推荐

Producer 端配置：
  props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "zstd");

Broker 端配置（compression.type）：
  producer    (默认) 保留 Producer 的压缩格式（无需重新压缩，推荐）
  uncompressed        解压存储（浪费 CPU，不推荐）
  snappy/lz4/gzip/zstd  统一使用该压缩格式（可能重新压缩）

Consumer 自动解压（无需配置），透明感知。
```

---
# Part 3：Broker 存储架构

## 3.1 存储目录结构

```
log.dirs=/data/kafka-logs        # Broker 数据目录（可配置多个磁盘）

/data/kafka-logs/
├── orders-0/                    # Topic=orders, Partition=0
│   ├── 00000000000000000000.log     # 数据文件（segment）
│   ├── 00000000000000000000.index   # 稀疏偏移量索引
│   ├── 00000000000000000000.timeindex  # 时间索引
│   └── leader-epoch-checkpoint
├── orders-1/                    # Topic=orders, Partition=1
│   ├── 00000000000000000000.log
│   ├── 00000000000000000000.index
│   ├── 00000000000000001234.log     # 第二个 segment（offset从1234开始）
│   └── 00000000000000001234.index
└── __consumer_offsets-0/        # 内部 Topic，存储 offset
```

每个 **Partition** 对应一个目录，目录内有多个 **Segment**（段文件）。
每个 Segment 由三个文件组成：`.log`（数据）、`.index`（偏移索引）、`.timeindex`（时间索引）。

---

## 3.2 Log Segment 详解

### Segment 文件命名规则

文件名 = 该 Segment **第一条消息的 offset**（20位，不足补0）：

```
00000000000000000000.log   → 第一个 Segment，从 offset=0 开始
00000000000000001234.log   → 第二个 Segment，从 offset=1234 开始
```

### .log 文件（消息数据）

每条消息在 .log 文件中的存储格式（RecordBatch）：

```
┌─────────────────────────────────────────────────────┐
│  RecordBatch                                        │
│  ┌──────────────┬────────────────────────────────┐  │
│  │ baseOffset   │ 8 bytes - Batch 第一条消息offset│  │
│  │ batchLength  │ 4 bytes - Batch 总长度          │  │
│  │ magic        │ 1 byte  - 消息格式版本(0/1/2)   │  │
│  │ crc          │ 4 bytes - CRC32校验            │  │
│  │ attributes   │ 2 bytes - 压缩/事务/控制标志    │  │
│  │ lastOffsetDelta│4 bytes - 最后一条消息相对offset│  │
│  │ firstTimestamp│8 bytes - 第一条消息时间戳      │  │
│  │ maxTimestamp │ 8 bytes - 最大时间戳            │  │
│  │ producerId   │ 8 bytes - 幂等/事务 Producer ID │  │
│  │ producerEpoch│ 2 bytes - Producer Epoch       │  │
│  │ baseSequence │ 4 bytes - 起始序列号            │  │
│  │ records[]    │ N bytes - 实际消息列表          │  │
│  └──────────────┴────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘

每条 Record（消息）格式：
  length         varint  - 消息长度
  attributes     int8    - 消息属性（保留字段）
  timestampDelta varint  - 相对 firstTimestamp 的增量
  offsetDelta    varint  - 相对 baseOffset 的增量
  keyLength      varint  - key 长度（-1 表示 null）
  key            bytes   - 消息 key
  valueLength    varint  - value 长度
  value          bytes   - 消息 value（业务数据）
  headers[]      Headers - 自定义 header
```

### .index 文件（稀疏偏移量索引）

`.index` 不是每条消息都有索引，而是**稀疏索引**（每隔 `log.index.interval.bytes`=4096 字节记录一个）：

```
.index 文件结构（每条索引 8 bytes）：
┌────────────────┬────────────────┐
│ relativeOffset │ position       │
│ 4 bytes        │ 4 bytes        │
│ (相对baseOffset│ (.log 文件中的 │
│  的相对偏移量) │  物理字节位置) │
└────────────────┴────────────────┘

示例（Segment baseOffset=1000）：
  relativeOffset=0,   position=0       → 对应 offset=1000, 在.log第0字节
  relativeOffset=100, position=12345   → 对应 offset=1100, 在.log第12345字节
  relativeOffset=250, position=30210   → 对应 offset=1250, 在.log第30210字节

查找 offset=1150 的消息：
  1. 二分查找 .index，找到 relativeOffset=100（position=12345）
  2. 从 .log 第 12345 字节开始顺序扫描，直到找到 offset=1150
```

---

## 3.3 顺序写与页缓存

Kafka 之所以快，核心原因之一是**顺序写磁盘 + 页缓存**：

```
传统随机写：
  应用 → 随机位置写 → 磁盘寻道（约10ms） → 写入
  IOPS 约 100~200次/秒（机械硬盘）

Kafka 顺序写：
  应用 → append 到文件末尾 → 无需寻道 → 写入
  吞吐量可达 600MB/s+（机械硬盘）

页缓存（Page Cache）机制：
  Producer → write() → Page Cache（内存）→ 异步 pdflush → 磁盘
                           ↑
  Consumer ← read()  ← Page Cache（内存命中，无磁盘IO）

这意味着：高速生产消费场景下，消息可能完全在内存中流转，
磁盘只是持久化保障，实际延迟极低。

建议：给 Kafka Broker 留出大量系统内存给 Page Cache，
JVM Heap 只需 6-8GB，剩余内存留给 OS Page Cache。
```

---

## 3.4 零拷贝（Zero Copy）

Consumer 从 Broker 拉取消息时，Kafka 使用 **sendfile** 系统调用实现零拷贝：

```
传统数据读取流程（4次拷贝，4次上下文切换）：
  磁盘 → [DMA] → 内核Buffer
  内核Buffer → [CPU拷贝] → 用户空间Buffer（read()返回）
  用户空间Buffer → [CPU拷贝] → Socket内核Buffer（write()调用）
  Socket内核Buffer → [DMA] → 网卡

Kafka 零拷贝（sendfile，2次拷贝，2次上下文切换）：
  磁盘 → [DMA] → 内核Buffer
  内核Buffer → [DMA] → 网卡（直接发送，跳过用户空间）

性能对比：
  传统方式：1GB数据需要约 3.5秒
  零拷贝：  1GB数据需要约 1.0秒（提升 3x+）
```

Java 中对应的实现是 `FileChannel.transferTo()`，底层调用 Linux `sendfile(2)`。

---

## 3.5 日志清理策略

Kafka 数据不会永久保存，支持两种清理策略（通过 `log.cleanup.policy` 配置）：

### 策略一：delete（删除，默认）

按时间或大小删除旧 Segment：

```
# 按时间保留（默认7天）
log.retention.hours=168           # 7天
log.retention.minutes=            # 分钟级（优先级高于hours）
log.retention.ms=                 # 毫秒级（优先级最高）

# 按大小保留（默认-1=不限制）
log.retention.bytes=-1            # Partition 级别大小限制

# Segment 大小（超过则创建新Segment）
log.segment.bytes=1073741824      # 1GB（默认）

# 检查频率
log.retention.check.interval.ms=300000  # 5分钟检查一次

清理逻辑：
  LogCleaner 线程定期扫描所有 Partition
  对于非活跃 Segment（不是最新的那个），检查最后修改时间
  若超过 retention 时间/大小，直接删除整个 Segment 文件
```

### 策略二：compact（日志压实）

**保留每个 Key 的最新 Value**，适合 changelog 类场景（如数据库变更日志）：

```
log.cleanup.policy=compact

原始日志（key→value）：
  offset 0: K1→v1
  offset 1: K2→v2
  offset 2: K1→v3   ← K1的新值
  offset 3: K3→v4
  offset 4: K2→v5   ← K2的新值
  offset 5: K1→null ← tombstone（删除标记）

压实后保留：
  K1→null（最新是删除）→ 等下次压实时彻底删除
  K2→v5（保留最新值）
  K3→v4（只有一个值，保留）

适用场景：
  - __consumer_offsets（消费者位移，key=groupId+topic+partition）
  - Kafka Streams 的状态存储
  - CDC（Change Data Capture）场景
```

---

## 3.6 Broker 核心配置参数

```properties
# server.properties 关键配置

# 基础配置
broker.id=0                           # Broker唯一ID（集群内唯一）
listeners=PLAINTEXT://0.0.0.0:9092    # 监听地址
advertised.listeners=PLAINTEXT://192.168.1.100:9092  # 对外宣告地址

# 存储配置
log.dirs=/data/kafka-logs             # 数据目录（多磁盘：/data1/kafka,/data2/kafka）
log.segment.bytes=1073741824          # Segment大小：1GB
log.retention.hours=168               # 数据保留时长：7天
log.retention.bytes=-1                # 不限大小
log.index.interval.bytes=4096         # 索引间隔：4KB

# 网络配置
num.network.threads=3                 # 网络IO线程数（建议 = CPU核心数）
num.io.threads=8                      # 磁盘IO线程数（建议 = CPU核心数 * 2）
socket.send.buffer.bytes=102400       # Socket发送缓冲区
socket.receive.buffer.bytes=102400    # Socket接收缓冲区
socket.request.max.bytes=104857600    # 最大请求大小：100MB

# 副本配置
default.replication.factor=3         # 默认副本数
min.insync.replicas=2                 # 最少同步副本数（配合acks=all使用）
replica.lag.time.max.ms=30000         # Follower落后超过30s则移出ISR

# 日志清理
log.cleanup.policy=delete             # 清理策略
log.cleaner.enable=true               # 启用日志清理线程
```


# Part 4：副本机制与高可用

## 4.1 AR / ISR / OSR 概念回顾

```
AR (Assigned Replicas)  = 分区的全部副本集合
ISR (In-Sync Replicas)  = 与 Leader 保持同步的副本集合
OSR (Out-of-Sync Replicas) = 落后太多、被踢出 ISR 的副本

AR = ISR + OSR

判断 Follower 是否在 ISR 中的条件（二者满足之一则踢出）：
  1. Follower 超过 replica.lag.time.max.ms（默认30s）未向 Leader 发送 Fetch 请求
  2. Follower 的 LEO（Log End Offset）与 Leader 的 LEO 差距超过阈值
     （0.9以前用 replica.lag.max.messages，0.10+已废弃，只用时间判断）
```

---

## 4.2 LEO 与 HW（高水位）

```
LEO (Log End Offset)：每个副本下一条待写入消息的 offset（即已写入最大offset+1）
HW  (High Watermark)：消费者可见的最大 offset（ISR 中最小的 LEO）

示例（3副本，ISR=[Leader, F1, F2]）：

Leader:  LEO=10  [0,1,2,3,4,5,6,7,8,9]
F1:      LEO=8   [0,1,2,3,4,5,6,7]
F2:      LEO=9   [0,1,2,3,4,5,6,7,8]

HW = min(LEO of all ISR) = min(10, 8, 9) = 8

消费者只能读到 offset < 8 的消息（即0-7），
offset 8、9 虽已写入 Leader，但未被 ISR 全员确认，
对消费者不可见（防止 Leader 切换后数据不一致）。

当 F1 同步到 offset=9，F2 同步到 offset=10 后：
HW = min(10, 9, 10) 还是看最慢的... 直到全部同步 HW 才推进。
```

### HW 更新流程

```
1. Producer 发送消息到 Leader（Leader 写入，LEO=10）
2. F1, F2 发 Fetch 请求拉取数据
3. Leader 在 Fetch Response 中携带当前 HW
4. F1, F2 写入本地，更新自己的 LEO
5. F1, F2 下次 Fetch 时携带自己的 LEO
6. Leader 收到 F1/F2 的 LEO，更新 ISR 中各 Follower 的 LEO，
   重新计算 HW = min(all ISR LEO)
7. Leader 在下一个 Fetch Response 中将新 HW 通知 Follower
8. Follower 更新自己的 HW

注意：HW 的更新存在延迟（至少一轮 Fetch RPC 来回），
这是 Kafka 引入 Leader Epoch 的原因之一。
```

---

## 4.3 Leader 选举

当 Leader 宕机时，Controller 负责从 ISR 中选举新 Leader：

```
选举算法（从 ISR 中选第一个存活的副本）：

ISR = [Broker1(Leader, 宕机), Broker2, Broker3]
      ↓
存活的 ISR = [Broker2, Broker3]
选举结果 = Broker2（ISR 中第一个存活的）

关键点：
  - 只从 ISR 中选（保证数据不丢失）
  - ISR 为空时的处理（unclean.leader.election.enable）：
      false（默认）: 等待 ISR 中有副本存活才选举，可能导致分区不可用
      true:          从 OSR 中选举（可能丢数据，但保证可用性）

Controller 感知 Leader 宕机的方式：
  - ZooKeeper 模式：监听 /brokers/ids/[brokerId] 临时节点消失
  - KRaft 模式：通过 Raft 日志感知（无需 ZooKeeper）
```

---

## 4.4 Leader Epoch（解决 HW 截断问题）

Kafka 0.11 引入 **Leader Epoch** 解决 HW 机制的数据不一致问题：

```
问题场景（HW 截断导致数据丢失）：

1. Leader(B1) offset=[0,1,2], HW=2
   Follower(B2) offset=[0,1], HW=1（未收到最新HW）
2. B2 崩溃重启，发现自己 LEO(2)>HW(1)，截断到 HW=1
3. 此时 B1 也崩溃，B2 当选新 Leader
4. B2 的 offset=[0,1]，但 B1 原本已有 [0,1,2]，offset=2 的消息丢失！

Leader Epoch 解决方案：
  每次 Leader 变更，Epoch + 1，并记录 (epoch, startOffset)
  副本重启时不再依赖 HW 截断，而是向新 Leader 发送
  OffsetsForLeaderEpoch 请求，获取上一个 Epoch 的结束 offset，
  以此决定是否需要截断（且截断到正确位置）。

Leader Epoch 日志示例：
  Epoch 0: startOffset=0  （第一任 Leader）
  Epoch 1: startOffset=5  （第二任 Leader 从 offset=5 开始写）
  Epoch 2: startOffset=12 （第三任 Leader 从 offset=12 开始写）
```

---

## 4.5 Controller 的职责

Kafka 集群中有且仅有一个 **Controller**（通过 ZooKeeper/KRaft 选举产生）：

```
Controller 职责：
  1. 监听 Broker 加入/离开（ZooKeeper 临时节点）
  2. 为宕机 Broker 上的 Partition 重新选举 Leader
  3. 监听 Topic/Partition 变更（创建、删除、扩容分区）
  4. 将 Leader/ISR 变更信息同步给所有 Broker（通过 LeaderAndIsr 请求）
  5. 管理分区的 preferred replica（优先副本）选举

Controller 选举（ZooKeeper 模式）：
  所有 Broker 争抢创建 /controller 临时节点（ZK 保证只有一个成功）
  成功创建的 Broker 成为 Controller
  其他 Broker 监听 /controller 节点，节点消失时重新竞选

Controller 选举（KRaft 模式）：
  基于 Raft 协议，从 controller.quorum.voters 配置的节点中选举
  无需 ZooKeeper，选举更快（故障恢复从分钟级→秒级）
```

---

## 4.6 优先副本选举（Preferred Replica Election）

```
创建 Topic 时，Kafka 会均匀分配各 Partition 的 Leader 到不同 Broker：

Topic: orders, 3 Partition, 3 Replica
  Partition 0: AR=[Broker1, Broker2, Broker3], Leader=Broker1（优先副本）
  Partition 1: AR=[Broker2, Broker3, Broker1], Leader=Broker2（优先副本）
  Partition 2: AR=[Broker3, Broker1, Broker2], Leader=Broker3（优先副本）

集群负载均衡，每个 Broker 各承担 1 个 Leader。

问题：某 Broker 重启后，其上的 Partition 已在其他 Broker 上重新选举 Leader。
重启完成后，Leader 不会自动迁移回来，导致负载不均衡。

解决：优先副本选举（Preferred Replica Election）
  将各 Partition AR 列表第一个副本（优先副本）重新选为 Leader。

触发方式：
  # 自动触发（推荐）
  auto.leader.rebalance.enable=true          # 默认 true
  leader.imbalance.check.interval.seconds=300 # 每5分钟检查
  leader.imbalance.per.broker.percentage=10   # Leader 不均衡超过10%则触发

  # 手动触发
  kafka-preferred-replica-election.sh --bootstrap-server localhost:9092
```


# Part 5：消费者（Consumer）深度解析

## 5.1 Consumer Group 模型

```
Consumer Group 是 Kafka 消费的核心概念：
  - 同一 Group 内，每个 Partition 只能被一个 Consumer 消费（互斥）
  - 不同 Group 之间，互相独立，都能消费全量消息（广播）

示例（Topic orders，6 Partitions）：

Group A（3个Consumer）：
  Consumer-1 → Partition 0, 1
  Consumer-2 → Partition 2, 3
  Consumer-3 → Partition 4, 5

Group B（2个Consumer）：
  Consumer-1 → Partition 0, 1, 2
  Consumer-2 → Partition 3, 4, 5

Group C（1个Consumer，消费全部）：
  Consumer-1 → Partition 0, 1, 2, 3, 4, 5

Group D（7个Consumer，有1个空闲）：
  Consumer-1 → Partition 0
  Consumer-2 → Partition 1
  ...
  Consumer-6 → Partition 5
  Consumer-7 → 无分配（Partition数量 < Consumer数量，多余的Consumer空闲）

结论：Consumer 数量 > Partition 数量时，多余的 Consumer 不消费任何分区。
      因此，Partition 数量决定了消费并发上限。
```

---

## 5.2 分区分配策略（Partition Assignment Strategy）

Kafka 提供三种内置分配策略，通过 `partition.assignment.strategy` 配置：

### RangeAssignor（范围分配，默认）

```
按 Topic 维度，将 Partition 范围分配给 Consumer：

Topic A: 6 Partitions [0,1,2,3,4,5]
Topic B: 3 Partitions [0,1,2]
Group:   Consumer C1, C2, C3

Topic A 分配：
  6 / 3 = 2，余 0
  C1 → A[0,1], C2 → A[2,3], C3 → A[4,5]

Topic B 分配：
  3 / 3 = 1，余 0
  C1 → B[0], C2 → B[1], C3 → B[2]

最终：C1=[A0,A1,B0], C2=[A2,A3,B1], C3=[A4,A5,B2] ✓ 均衡

但若 Topic A 有 7 个 Partition：
  7 / 3 = 2，余 1
  C1 → A[0,1,2], C2 → A[3,4], C3 → A[5,6]  ← C1 多一个

多 Topic 场景下，每个 Topic 余数都分给 C1，可能导致 C1 负载过重。
```

### RoundRobinAssignor（轮询分配）

```
将所有 Topic 的所有 Partition 混合排序后，轮询分配给 Consumer：

Topic A: [A0,A1,A2,A3,A4,A5,A6]
Topic B: [B0,B1,B2]

混合排序（按 Topic+Partition 字典序）：
  [A0,A1,A2,A3,A4,A5,A6,B0,B1,B2]

轮询分配给 C1, C2, C3：
  C1 → A0, A3, A6, B1
  C2 → A1, A4, B0, B2
  C3 → A2, A5

优点：整体更均衡，避免 RangeAssignor 的首个Consumer负载过重问题
缺点：当 Consumer 订阅的 Topic 不同时，分配可能不均（要求订阅相同 Topic）
```

### StickyAssignor（粘性分配，推荐）

```
目标：1. 尽量均衡  2. Rebalance 时尽量保持原有分配不变（减少数据迁移）

初始分配（与 RoundRobin 类似，尽量均衡）：
  C1 → [A0, A3, B0]
  C2 → [A1, A4, B1]
  C3 → [A2, A5, B2]

C3 宕机触发 Rebalance（StickyAssignor）：
  保留 C1 原有分配，保留 C2 原有分配，只重新分配 C3 的分区：
  C1 → [A0, A3, B0, A2]  ← 接管 C3 的 A2
  C2 → [A1, A4, B1, A5, B2] ← 接管 C3 的 A5, B2

若用 RoundRobinAssignor，可能会重新打乱所有分配，导致大量分区迁移。
StickyAssignor 最小化了重新分配的分区数量，减少状态迁移开销。
```

---

## 5.3 Rebalance（再平衡）机制

### 触发 Rebalance 的场景

```
1. Consumer 加入 Group（新Consumer启动）
2. Consumer 离开 Group（Consumer停止/崩溃）
3. Consumer 心跳超时（超过 session.timeout.ms=10000ms）
4. Consumer 处理消息时间过长（超过 max.poll.interval.ms=300000ms=5min）
5. Topic 分区数变化（kafka-topics.sh --alter 增加分区）
6. Consumer 订阅的 Topic 变化（正则订阅时新 Topic 出现）
```

### Rebalance 流程

```
涉及角色：Consumer、GroupCoordinator（Broker端，负责管理Group）

阶段一：Join Group
  所有 Consumer 向 GroupCoordinator 发送 JoinGroup 请求
  GroupCoordinator 选出一个 Consumer 作为 Group Leader（通常是第一个加入的）
  GroupCoordinator 将成员信息（含订阅 Topic）发送给 Group Leader

阶段二：Sync Group
  Group Leader 根据分配策略计算分配方案
  Group Leader 通过 SyncGroup 请求将方案发送给 GroupCoordinator
  GroupCoordinator 将各 Consumer 的分配结果下发给对应 Consumer

Rebalance 期间影响：
  所有 Consumer 暂停消费（Stop-The-World）
  分区所有权转移可能丢失内存中未提交的处理状态
  高频 Rebalance 会严重影响消费吞吐量

┌──────────────────────────────────────────────────────────┐
│              Rebalance 时序图                             │
│                                                          │
│  Consumer1    Consumer2    Consumer3    GroupCoordinator │
│     │             │            │              │          │
│     │──JoinGroup──┼────────────┼─────────────>│          │
│     │             │──JoinGroup─┼─────────────>│          │
│     │             │            │──JoinGroup──>│          │
│     │             │            │              │ 选 Leader │
│     │<─JoinResp(Leader,成员列表)│              │          │
│     │             │<──JoinResp─┼──────────────│          │
│     │(计算分配方案)│            │<─JoinResp────│          │
│     │──SyncGroup(分配方案)──────┼─────────────>│          │
│     │             │──SyncGroup─┼─────────────>│          │
│     │             │            │──SyncGroup──>│          │
│     │<─SyncResp(C1的分配)       │              │          │
│     │             │<─SyncResp(C2的分配)        │          │
│     │             │            │<─SyncResp────│          │
│  开始消费         │开始消费    │开始消费        │          │
└──────────────────────────────────────────────────────────┘
```

### 减少 Rebalance 的建议

```java
// 1. 增大 session.timeout.ms（心跳超时时间）
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, "30000"); // 30秒

// 2. 增大 max.poll.interval.ms（处理时间上限）
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, "600000"); // 10分钟

// 3. 减小 max.poll.records（每次拉取消息数，减少处理时间）
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, "100");

// 4. 发送心跳频率（建议 < session.timeout.ms / 3）
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, "3000"); // 3秒

// 5. 使用 StickyAssignor（减少 Rebalance 时的分区迁移）
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    StickyAssignor.class.getName());
```

---

## 5.4 Offset 提交

### 自动提交 vs 手动提交

```java
// 自动提交（默认）
props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, "true");
props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, "5000"); // 5秒提交一次
// 风险：可能重复消费（消息处理完但 offset 未提交时 Consumer 崩溃）
//       或消息丢失（offset 提交了但业务处理失败）

// 手动同步提交
consumer.commitSync(); // 提交当前拉取的所有消息的 offset，阻塞直到成功

// 手动异步提交
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("Commit failed for offsets: {}", offsets, exception);
    }
});

// 推荐：异步提交 + 最后同步提交（兼顾性能和可靠性）
try {
    while (running) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        process(records); // 业务处理
        consumer.commitAsync(); // 正常情况异步提交
    }
} catch (Exception e) {
    log.error("Unexpected error", e);
} finally {
    try {
        consumer.commitSync(); // 退出前同步提交，确保 offset 不丢
    } finally {
        consumer.close();
    }
}
```

### 指定 offset 消费

```java
// 从头消费
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

// 从最新位置消费（默认）
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");

// 指定 offset 消费
consumer.assign(Arrays.asList(new TopicPartition("orders", 0)));
consumer.seek(new TopicPartition("orders", 0), 1000L); // 从 offset=1000 开始

// 指定时间戳消费（找到该时间戳对应的 offset）
Map<TopicPartition, Long> timestampsToSearch = new HashMap<>();
timestampsToSearch.put(new TopicPartition("orders", 0),
    LocalDateTime.of(2024, 1, 1, 0, 0).toInstant(ZoneOffset.UTC).toEpochMilli());
Map<TopicPartition, OffsetAndTimestamp> result = consumer.offsetsForTimes(timestampsToSearch);
result.forEach((tp, offsetAndTimestamp) -> {
    if (offsetAndTimestamp != null) {
        consumer.seek(tp, offsetAndTimestamp.offset());
    }
});
```

---

## 5.5 __consumer_offsets 内部 Topic

```
Kafka 将消费者 offset 存储在内部 Topic __consumer_offsets 中：

__consumer_offsets 特性：
  - 默认 50 个 Partition（offsets.topic.num.partitions=50）
  - 副本数 = offsets.topic.replication.factor（默认3）
  - 使用 log.cleanup.policy=compact（保留每个Key的最新值）

Key 格式：groupId + topic + partition
Value 格式：offset + metadata + timestamp

哪个 Partition 存储某个 Group 的 offset？
  partitionIndex = Math.abs(groupId.hashCode()) % 50

对应的 GroupCoordinator 就是该 Partition 的 Leader 所在 Broker。

查看消费者 offset：
  kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --group my-group --describe

输出示例：
GROUP        TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
my-group     orders   0          1000            1050            50
my-group     orders   1          980             1000            20
```

---

## 5.6 Consumer 核心配置参数

```properties
# 基础配置
bootstrap.servers=localhost:9092
group.id=my-consumer-group
key.deserializer=org.apache.kafka.common.serialization.StringDeserializer
value.deserializer=org.apache.kafka.common.serialization.StringDeserializer

# 拉取配置
fetch.min.bytes=1                    # 最小拉取字节数（服务端等到至少1字节才返回）
fetch.max.bytes=52428800             # 最大拉取字节数：50MB
fetch.max.wait.ms=500                # 服务端等待时间（凑满fetch.min.bytes前的等待上限）
max.poll.records=500                 # 每次 poll() 返回的最大消息数
max.partition.fetch.bytes=1048576    # 每个 Partition 每次拉取最大字节数：1MB

# 会话配置
session.timeout.ms=10000             # 心跳超时时间：10秒
heartbeat.interval.ms=3000           # 心跳间隔：3秒
max.poll.interval.ms=300000          # 两次 poll() 之间最大时间间隔：5分钟

# offset 配置
auto.offset.reset=latest             # 新 Group 或 offset 过期时从哪里开始消费
enable.auto.commit=true              # 是否自动提交 offset
auto.commit.interval.ms=5000         # 自动提交间隔：5秒
```


# Part 6：Exactly-Once 语义

## 6.1 三种消息语义对比

```
┌─────────────────┬──────────────────────────────────────────────────────┐
│ 语义            │ 说明                                                  │
├─────────────────┼──────────────────────────────────────────────────────┤
│ At Most Once    │ 最多一次。消息可能丢失，但不会重复。                    │
│ （最多一次）     │ Producer: acks=0；Consumer: 先提交offset再处理消息。   │
├─────────────────┼──────────────────────────────────────────────────────┤
│ At Least Once   │ 至少一次。消息不会丢失，但可能重复。                    │
│ （至少一次）     │ Producer: acks=all + retries>0；                      │
│                 │ Consumer: 先处理消息再提交offset。                     │
│                 │ 这是 Kafka 默认语义。                                  │
├─────────────────┼──────────────────────────────────────────────────────┤
│ Exactly Once    │ 恰好一次。消息不丢失也不重复。                          │
│ （精确一次）     │ 需要 幂等性（Idempotent）+ 事务（Transaction）实现。   │
└─────────────────┴──────────────────────────────────────────────────────┘
```

---

## 6.2 幂等性（Idempotent Producer）

### 问题：Producer 重试导致消息重复

```
正常流程：
  Producer → 发送消息(seq=1) → Broker 写入 → 返回 ACK ✓

异常流程（重试导致重复）：
  Producer → 发送消息(seq=1) → Broker 写入 → 网络异常，ACK 丢失
  Producer → 超时重试(seq=1) → Broker 再次写入 → 消息重复！
```

### 幂等性解决方案

```
启用幂等性：
  props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
  // 同时要求：acks=all, retries>0, max.in.flight.requests.per.connection<=5

幂等性原理：
  每个 Producer 分配唯一的 ProducerId（PID）
  每条消息携带递增的 SequenceNumber
  Broker 为每个 (PID, Partition) 维护已写入的最大 SequenceNumber
  若收到重复的 SequenceNumber，直接丢弃（不写入），返回成功

  Producer → {PID=101, Seq=5, msg} → Broker
  Broker 已记录 (PID=101, Partition=0) 的最大Seq=5
  → 重复，忽略，返回成功

局限性：
  只保证单个 Producer 会话内（单 Partition）的幂等性
  Producer 重启后 PID 变化，幂等性失效
  不支持跨 Partition 的幂等性
  → 需要事务来解决跨 Partition、跨会话的精确一次
```

---

## 6.3 事务（Transactional Producer）

### 启用事务

```java
// Producer 配置
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, "true");
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-1"); // 全局唯一

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions(); // 初始化事务（向 TransactionCoordinator 注册）

try {
    producer.beginTransaction(); // 开启事务

    // 发送到多个 Partition（原子性）
    producer.send(new ProducerRecord<>("orders", "order-1", "..."));
    producer.send(new ProducerRecord<>("order-events", "event-1", "..."));

    // 提交消费者 offset（消费-处理-生产 原子性，用于 Kafka Streams）
    producer.sendOffsetsToTransaction(
        Collections.singletonMap(
            new TopicPartition("input-topic", 0),
            new OffsetAndMetadata(100L)
        ),
        new ConsumerGroupMetadata("my-consumer-group")
    );

    producer.commitTransaction(); // 提交事务
} catch (ProducerFencedException e) {
    // 同一 transactional.id 的旧 Producer 实例（被 fence 了）
    producer.close();
} catch (KafkaException e) {
    producer.abortTransaction(); // 回滚事务
    throw e;
}
```

### 事务实现原理

```
核心组件：
  TransactionCoordinator：Broker 端事务协调者（与 GroupCoordinator 类似）
  Transaction Log：内部 Topic __transaction_state（50个Partition）
  事务 ID → 对应哪个 TransactionCoordinator 的公式：
    abs(transactionalId.hashCode()) % 50

事务状态机：
  Empty → Ongoing → PrepareCommit → CompleteCommit
                   → PrepareAbort  → CompleteAbort

事务提交流程：
  1. Producer 向 TransactionCoordinator 发送 AddPartitionsToTxn
     （告知本次事务涉及哪些 Partition）
  2. Producer 向各 Partition Leader 发送消息（带 PID + Epoch）
  3. Producer 向 TransactionCoordinator 发送 EndTransaction(commit)
  4. TransactionCoordinator 向 Transaction Log 写入 PrepareCommit
  5. TransactionCoordinator 向各 Partition Leader 发送 WriteTxnMarkers
     （写入事务结束标记 COMMIT 控制消息）
  6. TransactionCoordinator 向 Transaction Log 写入 CompleteCommit

Consumer 端的事务隔离级别：
  isolation.level=read_committed（只读已提交的事务消息，默认 read_uncommitted）
  props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

---

## 6.4 消费-处理-生产的 Exactly-Once（Kafka Streams 场景）

```
场景：从 Topic A 消费，处理后写入 Topic B，要求精确一次。

实现方式：
  使用事务型 Producer，将 "提交消费offset" 和 "发送结果消息" 放在同一事务中：

  producer.beginTransaction();
  // 1. 处理消息
  ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
  for (ConsumerRecord<String, String> record : records) {
      String result = process(record); // 业务处理
      producer.send(new ProducerRecord<>("output-topic", result));
  }
  // 2. 将 offset 提交到事务（而非直接 commitSync）
  Map<TopicPartition, OffsetAndMetadata> offsets = currentOffsets(records);
  producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());
  // 3. 提交事务（输出消息 + offset 更新 原子提交）
  producer.commitTransaction();

这样即使中途崩溃：
  - 若事务未提交：重启后重新消费，重新处理，重新提交（At Least Once → 精确一次）
  - 输出 Topic 的 Consumer 设置 read_committed，只读已提交的消息
  - 最终效果：消息恰好被处理并输出一次（Exactly Once）
```


# Part 7：Kafka 集群管理

## 7.1 ZooKeeper 在 Kafka 中的作用

```
Kafka 2.8 之前强依赖 ZooKeeper，它负责：

1. Broker 注册与发现
   /brokers/ids/[brokerId]   （临时节点，Broker 宕机自动删除）
   /brokers/topics/[topic]/partitions/[partition]/state

2. Controller 选举
   /controller               （临时节点，抢占式选举）

3. Topic 元数据存储
   /brokers/topics/[topic]   （Topic 配置）

4. ACL 和认证信息
   /kafka-acl/

5. 消费者 Group 信息（旧版本，新版本已迁移到 __consumer_offsets）

ZooKeeper 的缺点：
  - 引入外部依赖，运维复杂度增加
  - ZooKeeper 有节点数量限制（影响 Partition 上限）
  - Broker 重启后元数据恢复慢（需从 ZK 重新加载）
  - Controller 故障转移慢（需重新从 ZK 读取全量数据）
```

---

## 7.2 KRaft 模式（无需 ZooKeeper，Kafka 2.8+ 预览，3.3+ 正式）

```
KRaft = Kafka + Raft（将 Raft 共识协议内置到 Kafka 自身）

架构变化：
  - 废弃 ZooKeeper，元数据存储在 Kafka 内部的 __cluster_metadata Topic
  - 引入 Controller Quorum（控制器仲裁组），通过 Raft 选举 Active Controller
  - Broker 直接从 Active Controller 同步元数据（而非从 ZK）

KRaft 优势：
  - 消除 ZooKeeper 依赖，简化部署
  - 控制器故障转移从分钟级 → 秒级
  - 支持更多分区（百万级 vs ZK 的数万级）
  - 元数据变更延迟更低

KRaft 配置示例（server.properties）：
  # 节点角色（controller 或 broker 或两者）
  process.roles=broker,controller    # 小集群可以 broker 和 controller 合并

  # Controller 仲裁组成员（controller nodes）
  controller.quorum.voters=1@kafka1:9093,2@kafka2:9093,3@kafka3:9093

  # KRaft 集群唯一ID（通过 kafka-storage.sh random-uuid 生成）
  # node.id=1
  # log.dirs=/data/kafka-logs

初始化 KRaft 集群：
  # 生成集群 ID
  CLUSTER_ID=$(kafka-storage.sh random-uuid)

  # 格式化存储目录（每个节点执行）
  kafka-storage.sh format -t $CLUSTER_ID -c server.properties

  # 启动
  kafka-server-start.sh server.properties
```

---

## 7.3 常用运维命令大全

### Topic 管理

```bash
# 创建 Topic（6个Partition，3个副本）
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic orders \
  --partitions 6 \
  --replication-factor 3

# 查看所有 Topic
kafka-topics.sh --bootstrap-server localhost:9092 --list

# 查看 Topic 详情（包括 Leader/ISR 信息）
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic orders

# 修改 Topic 分区数（只能增加，不能减少）
kafka-topics.sh --bootstrap-server localhost:9092 \
  --alter --topic orders --partitions 12

# 删除 Topic
kafka-topics.sh --bootstrap-server localhost:9092 \
  --delete --topic orders

# 修改 Topic 配置（如保留时间）
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders \
  --alter --add-config retention.ms=86400000   # 1天

# 查看 Topic 配置
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics --entity-name orders --describe
```

### Consumer Group 管理

```bash
# 查看所有 Consumer Group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# 查看 Consumer Group 详情（offset、lag）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --describe

# 重置 Consumer Group offset（先停止 Consumer）
# 重置到最早
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic orders \
  --reset-offsets --to-earliest --execute

# 重置到最新
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic orders \
  --reset-offsets --to-latest --execute

# 重置到指定 offset
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic orders:0 \
  --reset-offsets --to-offset 1000 --execute

# 重置到指定时间
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --topic orders \
  --reset-offsets --to-datetime 2024-01-01T00:00:00.000 --execute

# 删除 Consumer Group（需先停止所有 Consumer）
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --delete
```

### 消息生产与消费（测试）

```bash
# 控制台生产者
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --property key.separator=: \
  --property parse.key=true

# 控制台消费者
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic orders \
  --from-beginning \
  --property print.key=true \
  --property key.separator=:

# 性能测试（生产）
kafka-producer-perf-test.sh \
  --topic perf-test \
  --num-records 1000000 \
  --record-size 1024 \
  --throughput -1 \
  --producer-props bootstrap.servers=localhost:9092

# 性能测试（消费）
kafka-consumer-perf-test.sh \
  --bootstrap-server localhost:9092 \
  --topic perf-test \
  --messages 1000000
```

### 分区副本迁移

```bash
# 生成迁移计划（将 orders topic 迁移到 broker 1,2,3）
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --topics-to-move-json-file topics.json \
  --broker-list "1,2,3" \
  --generate

# topics.json 内容：
# {"topics":[{"topic":"orders"}],"version":1}

# 执行迁移计划
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassign.json \
  --execute

# 查看迁移进度
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassign.json \
  --verify
```

### 日志工具

```bash
# 查看 .log 文件内容（人类可读格式）
kafka-dump-log.sh \
  --files /data/kafka-logs/orders-0/00000000000000000000.log \
  --print-data-log

# 查看 .index 文件
kafka-dump-log.sh \
  --files /data/kafka-logs/orders-0/00000000000000000000.index

# 查看 __consumer_offsets 中某 Group 的 offset
kafka-simple-consumer-shell.sh \
  --bootstrap-server localhost:9092 \
  --topic __consumer_offsets \
  --partition 15 \
  --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter"
```


# Part 8：Spring Boot 集成实战

## 8.1 依赖配置

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <!-- Spring Kafka（包含 spring-kafka 和 kafka-clients） -->
    <dependency>
        <groupId>org.springframework.kafka</groupId>
        <artifactId>spring-kafka</artifactId>
    </dependency>

    <!-- 可选：Avro 序列化 + Schema Registry -->
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>7.5.0</version>
    </dependency>

    <!-- 可选：JSON 序列化 -->
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

---

## 8.2 application.yml 配置

```yaml
spring:
  kafka:
    # 集群地址
    bootstrap-servers: localhost:9092,localhost:9093,localhost:9094

    # 生产者配置
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all                          # 最强确认
      retries: 3                         # 重试次数
      batch-size: 16384                  # 16KB
      linger-ms: 5                       # 等待5ms凑批
      buffer-memory: 33554432            # 缓冲区32MB
      compression-type: zstd             # 压缩
      properties:
        enable.idempotence: true         # 幂等性
        max.in.flight.requests.per.connection: 5

    # 消费者配置
    consumer:
      group-id: my-consumer-group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest
      enable-auto-commit: false          # 禁用自动提交（手动提交）
      max-poll-records: 100
      properties:
        spring.json.trusted.packages: "com.example.dto"  # JsonDeserializer 信任包
        isolation.level: read_committed

    # 监听器配置
    listener:
      ack-mode: MANUAL_IMMEDIATE         # 手动提交模式
      concurrency: 3                     # 并发消费线程数
      poll-timeout: 3000                 # poll 超时

    # 模板配置（用于事务）
    # template:
    #   transaction-id-prefix: tx-
```

---

## 8.3 KafkaTemplate（生产者）

```java
@Service
@Slf4j
public class OrderProducerService {

    @Autowired
    private KafkaTemplate<String, Object> kafkaTemplate;

    // 简单发送（fire-and-forget）
    public void sendOrder(Order order) {
        kafkaTemplate.send("orders", order.getId(), order);
    }

    // 异步发送，监听结果
    public void sendOrderAsync(Order order) {
        CompletableFuture<SendResult<String, Object>> future =
            kafkaTemplate.send("orders", order.getId(), order);

        future.whenComplete((result, ex) -> {
            if (ex == null) {
                RecordMetadata metadata = result.getRecordMetadata();
                log.info("发送成功: topic={}, partition={}, offset={}, timestamp={}",
                    metadata.topic(), metadata.partition(),
                    metadata.offset(), metadata.timestamp());
            } else {
                log.error("发送失败: order={}", order.getId(), ex);
                // 可以在这里做失败补偿（如存入数据库重试）
            }
        });
    }

    // 同步发送（阻塞等待结果）
    public SendResult<String, Object> sendOrderSync(Order order) throws Exception {
        return kafkaTemplate.send("orders", order.getId(), order).get(10, TimeUnit.SECONDS);
    }

    // 发送到指定 Partition
    public void sendToPartition(Order order, int partition) {
        kafkaTemplate.send("orders", partition, order.getId(), order);
    }

    // 批量发送
    public void sendBatch(List<Order> orders) {
        orders.forEach(order ->
            kafkaTemplate.send("orders", order.getId(), order));
        // KafkaTemplate 会自动合并到同一个 Batch 中（受 batch-size 和 linger-ms 控制）
    }
}
```

---

## 8.4 @KafkaListener（消费者）

```java
@Component
@Slf4j
public class OrderConsumer {

    // 基础用法
    @KafkaListener(topics = "orders", groupId = "order-processor")
    public void consume(Order order, Acknowledgment ack) {
        log.info("收到订单: {}", order.getId());
        processOrder(order);
        ack.acknowledge(); // 手动提交 offset
    }

    // 获取消息元数据
    @KafkaListener(topics = "orders", groupId = "order-processor-2")
    public void consumeWithMetadata(
            @Payload Order order,
            @Header(KafkaHeaders.RECEIVED_TOPIC) String topic,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            @Header(KafkaHeaders.RECEIVED_TIMESTAMP) long timestamp,
            Acknowledgment ack) {
        log.info("topic={}, partition={}, offset={}, ts={}, order={}",
            topic, partition, offset, timestamp, order.getId());
        ack.acknowledge();
    }

    // 批量消费（需配置 BatchListener）
    @KafkaListener(topics = "orders", groupId = "batch-processor",
                   containerFactory = "batchKafkaListenerContainerFactory")
    public void consumeBatch(List<Order> orders, Acknowledgment ack) {
        log.info("批量消费 {} 条订单", orders.size());
        processBatch(orders);
        ack.acknowledge();
    }

    // 监听多个 Topic
    @KafkaListener(topics = {"orders", "payments"}, groupId = "multi-topic")
    public void consumeMultiTopic(ConsumerRecord<String, String> record, Acknowledgment ack) {
        log.info("收到消息: topic={}, key={}, value={}",
            record.topic(), record.key(), record.value());
        ack.acknowledge();
    }

    // 指定分区消费
    @KafkaListener(
        topicPartitions = @TopicPartition(
            topic = "orders",
            partitionOffsets = {
                @PartitionOffset(partition = "0", initialOffset = "0"),
                @PartitionOffset(partition = "1", initialOffset = "0")
            }
        )
    )
    public void consumeSpecificPartitions(ConsumerRecord<String, String> record) {
        log.info("分区消费: partition={}, offset={}", record.partition(), record.offset());
    }

    // 错误处理（异常时不提交 offset，触发重试）
    @KafkaListener(topics = "orders", groupId = "retry-processor")
    public void consumeWithRetry(Order order, Acknowledgment ack) {
        try {
            processOrder(order);
            ack.acknowledge();
        } catch (RetryableException e) {
            log.warn("可重试异常，不提交offset: {}", e.getMessage());
            // 不调用 ack.acknowledge()，消息会在下次 poll 时重新消费
        } catch (Exception e) {
            log.error("不可重试异常，发送到死信队列: {}", e.getMessage());
            // 发送到死信队列后再提交
            sendToDeadLetterQueue(order);
            ack.acknowledge();
        }
    }

    private void processOrder(Order order) { /* 业务逻辑 */ }
    private void processBatch(List<Order> orders) { /* 批量处理 */ }
    private void sendToDeadLetterQueue(Order order) { /* 死信处理 */ }
}
```

---

## 8.5 ConcurrentKafkaListenerContainerFactory 配置

```java
@Configuration
@EnableKafka
public class KafkaConfig {

    @Autowired
    private ConsumerFactory<String, Object> consumerFactory;

    // 标准单条消费工厂
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object>
    kafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setConcurrency(3);  // 3个并发线程，每个线程消费一个分区
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
        factory.getContainerProperties().setPollTimeout(3000);

        // 设置错误处理器（消息处理失败时）
        factory.setCommonErrorHandler(new DefaultErrorHandler(
            new DeadLetterPublishingRecoverer(kafkaTemplate), // 发送到死信 Topic
            new FixedBackOff(1000L, 3)  // 重试3次，间隔1秒
        ));

        return factory;
    }

    // 批量消费工厂
    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object>
    batchKafkaListenerContainerFactory() {
        ConcurrentKafkaListenerContainerFactory<String, Object> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory);
        factory.setBatchListener(true);  // 开启批量模式
        factory.setConcurrency(3);
        factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
        return factory;
    }

    // 事务型 KafkaTemplate（需配合 transactional-id-prefix 配置）
    @Bean
    public KafkaTemplate<String, Object> kafkaTemplate(
            ProducerFactory<String, Object> producerFactory) {
        return new KafkaTemplate<>(producerFactory);
    }
}
```

---

## 8.6 死信队列（DLT）处理

```java
// 自动死信 Topic：原 Topic 名 + ".DLT"（如 orders.DLT）
@Component
@Slf4j
public class DeadLetterConsumer {

    // 消费死信 Topic
    @KafkaListener(topics = "orders.DLT", groupId = "dlt-processor")
    public void consumeDeadLetter(
            ConsumerRecord<String, Order> record,
            @Header(KafkaHeaders.DLT_EXCEPTION_MESSAGE) String exceptionMessage,
            @Header(KafkaHeaders.DLT_ORIGINAL_TOPIC) String originalTopic,
            @Header(KafkaHeaders.DLT_ORIGINAL_PARTITION) int originalPartition,
            @Header(KafkaHeaders.DLT_ORIGINAL_OFFSET) long originalOffset,
            Acknowledgment ack) {
        log.error("死信消息 from topic={}, partition={}, offset={}: exception={}",
            originalTopic, originalPartition, originalOffset, exceptionMessage);

        // 记录到数据库，人工处理
        saveDeadLetterRecord(record.value(), exceptionMessage,
            originalTopic, originalPartition, originalOffset);

        ack.acknowledge();
    }

    private void saveDeadLetterRecord(Order order, String exMsg,
            String topic, int partition, long offset) {
        // 持久化到 DB
    }
}
```

---

## 8.7 自定义序列化（JSON + TypeMapping）

```java
// 配置 JsonDeserializer 支持多类型
@Configuration
public class KafkaDeserializerConfig {

    @Bean
    public ConsumerFactory<String, Object> consumerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "my-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
        // 信任的包（防止反序列化攻击）
        props.put(JsonDeserializer.TRUSTED_PACKAGES, "com.example.dto");
        // 类型映射（处理不同服务间的类名不一致）
        props.put(JsonDeserializer.TYPE_MAPPINGS,
            "order:com.example.dto.Order,payment:com.example.dto.Payment");
        return new DefaultKafkaConsumerFactory<>(props);
    }
}

// Producer 端配置类型头
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, Object> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        // 发送类型信息（在 Header 中携带类名）
        props.put(JsonSerializer.ADD_TYPE_INFO_HEADERS, true);
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```


# Part 9：完整实战案例

## 9.1 案例一：订单系统异步解耦

### 业务场景

```
用户下单 → 订单服务创建订单 → 发送 Kafka 消息 → 多个下游服务异步处理：
  ├── 库存服务：扣减库存
  ├── 支付服务：初始化支付
  ├── 通知服务：发送短信/邮件
  └── 积分服务：增加积分
```

### 完整代码实现

```java
// 订单 DTO
@Data
@AllArgsConstructor
@NoArgsConstructor
public class OrderCreatedEvent {
    private String orderId;
    private String userId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;
}

// 订单服务（生产者）
@Service
@Slf4j
@Transactional
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private KafkaTemplate<String, OrderCreatedEvent> kafkaTemplate;

    public Order createOrder(CreateOrderRequest request) {
        // 1. 创建订单（DB 操作）
        Order order = Order.builder()
            .id(UUID.randomUUID().toString())
            .userId(request.getUserId())
            .items(request.getItems())
            .totalAmount(request.getTotalAmount())
            .status(OrderStatus.CREATED)
            .build();
        orderRepository.save(order);

        // 2. 发送事件（Kafka 消息）
        OrderCreatedEvent event = new OrderCreatedEvent(
            order.getId(), order.getUserId(),
            order.getItems(), order.getTotalAmount(), LocalDateTime.now());

        kafkaTemplate.send("order-events", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("订单事件发送失败: orderId={}", order.getId(), ex);
                    // 实际项目中应有补偿机制（如事务消息表）
                } else {
                    log.info("订单事件发送成功: orderId={}, offset={}",
                        order.getId(), result.getRecordMetadata().offset());
                }
            });

        return order;
    }
}

// 库存服务（消费者）
@Component
@Slf4j
public class InventoryConsumer {

    @Autowired
    private InventoryService inventoryService;

    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderCreated(
            OrderCreatedEvent event,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {
        log.info("库存服务收到订单: orderId={}, offset={}", event.getOrderId(), offset);
        try {
            // 幂等性检查（防止重复消费）
            if (inventoryService.isProcessed(event.getOrderId())) {
                log.warn("订单已处理过，跳过: {}", event.getOrderId());
                ack.acknowledge();
                return;
            }
            // 扣减库存
            inventoryService.deductInventory(event.getItems());
            inventoryService.markProcessed(event.getOrderId());
            ack.acknowledge();
        } catch (InsufficientInventoryException e) {
            log.error("库存不足: orderId={}", event.getOrderId(), e);
            // 发布库存不足事件，让订单服务回滚
            // ...
            ack.acknowledge(); // 即使业务失败也提交，避免无限重试
        }
    }
}

// 通知服务（消费者）
@Component
@Slf4j
public class NotificationConsumer {

    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderCreated(OrderCreatedEvent event, Acknowledgment ack) {
        log.info("通知服务收到订单: {}", event.getOrderId());
        // 发送短信
        smsService.sendOrderConfirmation(event.getUserId(), event.getOrderId());
        ack.acknowledge();
    }
}
```

---

## 9.2 案例二：日志收集（高吞吐场景）

```java
// 日志 DTO
@Data
public class AccessLog {
    private String requestId;
    private String userId;
    private String url;
    private int statusCode;
    private long responseTime;
    private String userAgent;
    private LocalDateTime timestamp;
}

// 高吞吐 Producer 配置
@Configuration
public class HighThroughputKafkaConfig {

    @Bean("highThroughputProducerFactory")
    public ProducerFactory<String, AccessLog> highThroughputProducerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, JsonSerializer.class);
        // 高吞吐配置
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);     // 64KB
        props.put(ProducerConfig.LINGER_MS_CONFIG, 20);          // 等20ms凑批
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4"); // 快速压缩
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864); // 64MB缓冲
        props.put(ProducerConfig.ACKS_CONFIG, "1");               // 只需Leader确认
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean("logKafkaTemplate")
    public KafkaTemplate<String, AccessLog> logKafkaTemplate() {
        return new KafkaTemplate<>(highThroughputProducerFactory());
    }
}

// 日志收集 AOP 切面
@Aspect
@Component
@Slf4j
public class AccessLogAspect {

    @Autowired
    @Qualifier("logKafkaTemplate")
    private KafkaTemplate<String, AccessLog> logKafkaTemplate;

    @Around("@annotation(org.springframework.web.bind.annotation.RequestMapping) || " +
            "@annotation(org.springframework.web.bind.annotation.GetMapping)")
    public Object logAccess(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = null;
        int statusCode = 200;
        try {
            result = pjp.proceed();
        } catch (Exception e) {
            statusCode = 500;
            throw e;
        } finally {
            AccessLog log = new AccessLog();
            log.setRequestId(UUID.randomUUID().toString());
            log.setResponseTime(System.currentTimeMillis() - start);
            log.setStatusCode(statusCode);
            log.setTimestamp(LocalDateTime.now());
            // 异步发送（不影响主流程）
            logKafkaTemplate.send("access-logs", log.getRequestId(), log);
        }
        return result;
    }
}

// 日志消费（写入 Elasticsearch）
@Component
@Slf4j
public class LogConsumer {

    @Autowired
    private ElasticsearchClient esClient;

    @KafkaListener(
        topics = "access-logs",
        groupId = "log-indexer",
        containerFactory = "batchKafkaListenerContainerFactory"
    )
    public void consumeLogs(List<AccessLog> logs, Acknowledgment ack) {
        log.info("批量索引日志: {} 条", logs.size());
        // 批量写入 ES
        BulkRequest.Builder br = new BulkRequest.Builder();
        for (AccessLog accessLog : logs) {
            br.operations(op -> op.index(idx -> idx
                .index("access-logs-" + LocalDate.now())
                .document(accessLog)));
        }
        try {
            esClient.bulk(br.build());
            ack.acknowledge();
        } catch (Exception e) {
            log.error("ES 批量写入失败", e);
            // 不提交 offset，下次重试
        }
    }
}
```

---

## 9.3 案例三：Kafka Streams 实时统计

```java
// 实时统计每分钟各商品的销售数量
@Configuration
public class SalesStreamConfig {

    @Bean
    public KafkaStreams salesStream() {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "sales-counter");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        // 输入：订单事件流
        KStream<String, String> ordersStream = builder.stream("order-events");

        // 按商品 ID 统计销量（1分钟滚动窗口）
        ordersStream
            .flatMapValues(value -> {
                // 解析订单，提取商品列表
                OrderCreatedEvent event = parseOrder(value);
                return event.getItems().stream()
                    .map(item -> item.getProductId() + ":" + item.getQuantity())
                    .collect(Collectors.toList());
            })
            .map((key, value) -> {
                String[] parts = value.split(":");
                return new KeyValue<>(parts[0], Integer.parseInt(parts[1]));
            })
            .groupByKey(Grouped.with(Serdes.String(), Serdes.Integer()))
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .reduce(Integer::sum)
            .toStream()
            .map((windowedKey, count) -> {
                String key = windowedKey.key() + "@" + windowedKey.window().startTime();
                return new KeyValue<>(key, count.toString());
            })
            .to("sales-stats-per-minute"); // 输出到结果 Topic

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
        return streams;
    }

    private OrderCreatedEvent parseOrder(String json) {
        // Jackson 解析
        return new ObjectMapper().readValue(json, OrderCreatedEvent.class);
    }
}
```


# Part 10：性能调优

## 10.1 生产者调优

```
目标：最大化吞吐量 vs 最小化延迟（二者往往是权衡）

┌─────────────────────────────────────────────────────────────────────┐
│ 参数                         │ 低延迟优先   │ 高吞吐优先            │
├─────────────────────────────────────────────────────────────────────┤
│ linger.ms                    │ 0（默认）    │ 20~100ms              │
│ batch.size                   │ 16384(16KB) │ 65536~131072(64~128KB)│
│ compression.type             │ none        │ lz4 / zstd            │
│ buffer.memory                │ 33554432    │ 67108864~134217728     │
│ max.in.flight.requests       │ 1           │ 5（启用幂等时≤5）      │
│ acks                         │ 1           │ 1（可靠性要求高用all） │
└─────────────────────────────────────────────────────────────────────┘

关键公式：
  吞吐量 ≈ (batch.size × max.in.flight) / 网络RTT
  实际批填充率 = min(linger.ms内积累的消息, batch.size)

生产者线程数建议：
  单 Producer 多 Topic 场景：1~3个Producer实例即可（内部是线程安全的）
  超高并发场景：多个 Producer 实例 + 各自的线程池
  不建议每个线程一个 Producer（连接开销大）
```

### 生产者监控指标（JMX）

```
kafka.producer:type=producer-metrics,client-id=*
  record-send-rate          # 每秒发送消息数
  record-error-rate         # 每秒错误数
  request-latency-avg       # 平均请求延迟（ms）
  outgoing-byte-rate        # 出站字节速率
  batch-size-avg            # 平均 batch 大小
  record-queue-time-avg     # 消息在缓冲队列中的平均等待时间
  compression-rate-avg      # 平均压缩率

监控建议：
  record-error-rate > 0  → 有发送失败，检查网络/Broker状态
  request-latency-avg 高 → 网络延迟或 Broker 负载高
  record-queue-time-avg 高 → buffer.memory 不够或发送速率超过 Sender 处理能力
```

---

## 10.2 消费者调优

```
目标：最大化消费吞吐量，同时避免 Rebalance

关键参数优化：

1. 增大 fetch.min.bytes（减少空轮询）
   fetch.min.bytes=65536    # 等到有64KB数据才返回（减少小包请求）
   fetch.max.wait.ms=500    # 最多等500ms

2. 增大 max.poll.records（批量处理）
   max.poll.records=500     # 每次 poll 拉取500条（需确保能在max.poll.interval.ms内处理完）

3. 增大 fetch.max.bytes
   fetch.max.bytes=10485760 # 单次 fetch 最大10MB

4. 并发消费（多线程）
   # 方案一：增加 Consumer 实例数（最多等于 Partition 数）
   concurrency=6            # 在 @KafkaListener 容器中设置

   # 方案二：Consumer 内部多线程处理（适合IO密集型场景）
   @KafkaListener(topics = "orders", concurrency = "6")
   public void consume(Order order, Acknowledgment ack) {
       CompletableFuture.runAsync(() -> processAsync(order), executor)
           .thenRun(ack::acknowledge);
   }

5. 避免 Rebalance
   session.timeout.ms=30000       # 提高超时容忍度
   heartbeat.interval.ms=10000    # 心跳间隔 < timeout/3
   max.poll.interval.ms=600000    # 确保处理时间上限够用
```

---

## 10.3 Broker 调优

### JVM 参数

```bash
# KAFKA_HEAP_OPTS（建议 6-8GB，不要太大，避免 GC Stop-the-World）
export KAFKA_HEAP_OPTS="-Xms6g -Xmx6g"

# GC 选择（Kafka 建议使用 G1GC）
export KAFKA_JVM_PERFORMANCE_OPTS="-server \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=20 \
  -XX:InitiatingHeapOccupancyPercent=35 \
  -XX:+ExplicitGCInvokesConcurrent \
  -Djava.awt.headless=true"
```

### OS 参数调优

```bash
# 1. 文件描述符限制（Kafka 需要大量文件句柄）
echo "* soft nofile 100000" >> /etc/security/limits.conf
echo "* hard nofile 100000" >> /etc/security/limits.conf

# 2. 网络参数
echo "net.core.somaxconn=65535" >> /etc/sysctl.conf
echo "net.ipv4.tcp_max_syn_backlog=65535" >> /etc/sysctl.conf
echo "net.core.netdev_max_backlog=250000" >> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem=4096 87380 16777216" >> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem=4096 65536 16777216" >> /etc/sysctl.conf

# 3. 虚拟内存（减少 swap，避免 Page Cache 被置换）
echo "vm.swappiness=1" >> /etc/sysctl.conf      # 几乎不使用 swap
echo "vm.dirty_ratio=60" >> /etc/sysctl.conf    # 脏页比例（触发强制刷盘）
echo "vm.dirty_background_ratio=10" >> /etc/sysctl.conf  # 后台刷盘阈值

sysctl -p  # 生效

# 4. 磁盘
# 推荐使用 XFS 文件系统（比 EXT4 在 Kafka 场景下性能更好）
# 挂载参数加 noatime（不更新访问时间，减少IO）
# mount -o noatime /dev/sdb1 /data/kafka-logs
```

### Broker 核心参数调优

```properties
# 网络线程（处理客户端请求）
num.network.threads=8          # 建议 = CPU 核心数

# IO 线程（处理磁盘读写）
num.io.threads=16              # 建议 = CPU 核心数 * 2（IO密集型）

# 队列大小（等待处理的请求队列）
queued.max.requests=500

# 副本 Fetch 线程数
num.replica.fetchers=4         # 增加 Follower 同步速度（默认1）

# Socket 缓冲区（与 OS 的 tcp_rmem/tcp_wmem 匹配）
socket.send.buffer.bytes=1048576    # 1MB
socket.receive.buffer.bytes=1048576

# 日志刷盘（通常依赖 OS Page Cache，不建议强制同步刷盘）
log.flush.interval.messages=Long.MAX_VALUE  # 不强制刷（默认）
log.flush.interval.ms=Long.MAX_VALUE        # 不强制按时刷（默认）
# 强制刷盘会严重影响性能，Kafka 依赖多副本而非磁盘 fsync 保证持久性
```

---

## 10.4 分区数与吞吐量关系

```
分区数越多，并行度越高，但也有上限和代价：

吞吐量 ≈ 分区数 × 单分区吞吐量

单分区吞吐量受限于：
  - 磁盘顺序写速度（约 200~600MB/s）
  - 网络带宽
  - 单个消费者的处理速度

分区数过多的代价：
  1. 文件句柄增多（每个 Partition = 3个文件）
  2. Controller 和 ZooKeeper 压力增大
  3. 分区选举时间增长
  4. 端到端延迟略微增加（Leader 需要等更多分区的 ACK）

建议分区数计算公式：
  目标吞吐量 = max(P吞吐量, C吞吐量) / 单Partition吞吐量

例如：
  目标Producer吞吐量: 600MB/s
  单 Partition Producer 吞吐量: 100MB/s
  → 推荐 Partition 数 = 600/100 = 6

  目标Consumer吞吐量: 1200MB/s（消费者处理较慢，每个只有50MB/s）
  → 推荐 Partition 数 = 1200/50 = 24

  取最大值: 24 个 Partition

实践建议：
  - 不超过单 Broker 1000 个 Partition（否则故障恢复慢）
  - 整个集群 Partition 总数控制在 Broker数量 × 2000 以内
  - 分区数可以后续增加，但 key 分区的消息顺序性会变化
```


# Part 11：监控与运维

## 11.1 JMX 监控指标

Kafka 暴露大量 JMX 指标，开启方式：

```bash
# 启动时开启 JMX（kafka-server-start.sh 之前）
export JMX_PORT=9999
kafka-server-start.sh config/server.properties
```

### Broker 核心指标

```
# 消息吞吐
kafka.server:type=BrokerTopicMetrics,name=MessagesInPerSec       # 入站消息数/秒
kafka.server:type=BrokerTopicMetrics,name=BytesInPerSec          # 入站字节数/秒
kafka.server:type=BrokerTopicMetrics,name=BytesOutPerSec         # 出站字节数/秒
kafka.server:type=BrokerTopicMetrics,name=FailedProduceRequestsPerSec  # 生产失败数

# 请求性能
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=Produce    # 生产请求总耗时
kafka.network:type=RequestMetrics,name=TotalTimeMs,request=FetchConsumer  # 消费请求总耗时

# 副本
kafka.server:type=ReplicaManager,name=IsrShrinksPerSec    # ISR 收缩速率（>0说明副本落后）
kafka.server:type=ReplicaManager,name=IsrExpandsPerSec    # ISR 扩展速率
kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions  # 副本不足的分区数（应=0）
kafka.server:type=ReplicaManager,name=OfflinePartitionsCount      # 离线分区数（应=0）
kafka.controller:type=KafkaController,name=ActiveControllerCount  # Controller数量（应=1）

# 日志
kafka.log:type=LogFlushStats,name=LogFlushRateAndTimeMs    # 日志刷盘速率和耗时

关键告警指标：
  UnderReplicatedPartitions > 0  → 副本同步异常，检查 Follower 节点
  OfflinePartitionsCount > 0     → 有分区无法提供服务，需紧急处理
  ActiveControllerCount != 1     → Controller 异常
  IsrShrinksPerSec 持续>0        → Follower 持续落后，检查网络/磁盘
```

---

## 11.2 消费者 Lag 监控

**Lag（消费延迟）** = Log End Offset - Current Offset，是消费者最重要的监控指标。

```bash
# 命令行查看 Lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-group --describe

GROUP        TOPIC    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG     CONSUMER-ID
my-group     orders   0          5000            5100            100     consumer-1
my-group     orders   1          4900            5000            100     consumer-2
my-group     orders   2          4800            4900            100     consumer-3
TOTAL LAG: 300

Lag 监控建议：
  - Lag 持续增长 → 消费者处理速度跟不上生产速度（扩容消费者或优化处理逻辑）
  - Lag 突然变0 → 可能消费者重置了 offset（检查是否有误操作）
  - Lag 波动正常 → 生产高峰期 Lag 增大，低峰期追上，属正常现象
```

### Java 代码监控 Lag

```java
@Component
@Slf4j
public class KafkaLagMonitor {

    @Autowired
    private KafkaAdmin kafkaAdmin;

    @Scheduled(fixedRate = 60000) // 每分钟检查一次
    public void checkLag() {
        try (AdminClient adminClient = AdminClient.create(kafkaAdmin.getConfigurationProperties())) {
            // 获取 Consumer Group 详情
            DescribeConsumerGroupsResult result =
                adminClient.describeConsumerGroups(Collections.singletonList("my-group"));
            ConsumerGroupDescription description =
                result.describedGroups().get("my-group").get();

            // 获取各分区 offset
            Map<TopicPartition, OffsetSpec> partitionOffsets = description.members().stream()
                .flatMap(m -> m.assignment().topicPartitions().stream())
                .collect(Collectors.toMap(tp -> tp, tp -> OffsetSpec.latest()));

            Map<TopicPartition, ListOffsetsResult.ListOffsetsResultInfo> endOffsets =
                adminClient.listOffsets(partitionOffsets).all().get();

            // 获取已提交 offset
            Map<TopicPartition, OffsetAndMetadata> committedOffsets =
                adminClient.listConsumerGroupOffsets("my-group")
                    .partitionsToOffsetAndMetadata().get();

            long totalLag = 0;
            for (Map.Entry<TopicPartition, OffsetAndMetadata> entry : committedOffsets.entrySet()) {
                TopicPartition tp = entry.getKey();
                long committed = entry.getValue().offset();
                long end = endOffsets.getOrDefault(tp,
                    new ListOffsetsResult.ListOffsetsResultInfo(0, 0, Optional.empty())).offset();
                long lag = end - committed;
                totalLag += lag;
                log.info("Lag: topic={}, partition={}, lag={}", tp.topic(), tp.partition(), lag);
            }

            log.info("Total Lag: {}", totalLag);
            if (totalLag > 10000) {
                // 发送告警
                alertService.sendAlert("Kafka Lag 过高: " + totalLag);
            }
        } catch (Exception e) {
            log.error("监控 Lag 失败", e);
        }
    }
}
```

---

## 11.3 Kafka Eagle / CMAK（可视化监控）

### Kafka Eagle（EFAK）

```
Kafka Eagle（现更名为 EFAK - Eagle For Apache Kafka）
  下载：https://www.kafka-eagle.org/
  功能：
    - Topic 列表、消息查看、生产/消费速率图表
    - Consumer Group Lag 趋势图
    - Broker 集群状态、JMX 指标可视化
    - SQL 查询 Kafka 消息（KQL）
    - 告警配置（Lag 超阈值发邮件/钉钉）

配置（system-config.properties）：
  kafka.eagle.zk.cluster.alias=cluster1
  cluster1.zk.list=localhost:2181
  # 或 KRaft 模式
  cluster1.efak.broker.size=3
  cluster1.efak.jmx.uri=service:jmx:rmi:///jndi/rmi://%s/jmxrmi
```

### CMAK（Cluster Manager for Apache Kafka）

```
原名 Kafka Manager（Yahoo 开源），现为 CMAK
  GitHub: https://github.com/yahoo/CMAK
  功能：
    - Broker/Topic/Partition 管理
    - 副本迁移
    - 优先副本选举
    - Consumer 监控（需手动配置）

Docker 快速启动：
  docker run -d \
    -p 9000:9000 \
    -e ZK_HOSTS="localhost:2181" \
    hlebalbau/kafka-manager:latest
```

---

## 11.4 常见故障排查

### 故障一：Consumer Lag 持续增长

```
排查步骤：
1. 检查消费者是否存活
   kafka-consumer-groups.sh --describe --group my-group
   → 看 CONSUMER-ID 列是否有值

2. 检查消费者处理速度
   监控 max.poll.interval.ms（是否频繁 Rebalance）
   查看应用日志（是否有慢处理、DB 慢查询等）

3. 检查是否有 Rebalance 风暴
   应用日志搜索 "Rebalancing" 字样
   → 增大 session.timeout.ms 和 max.poll.interval.ms

4. 扩容消费者（增加实例数，不超过 Partition 数）

5. 优化消费逻辑（批量处理、异步处理）
```

### 故障二：Producer 发送超时

```
排查步骤：
1. 检查 Broker 是否正常
   kafka-broker-api-versions.sh --bootstrap-server localhost:9092

2. 检查网络连接
   telnet localhost 9092

3. 检查 Broker 日志（/var/log/kafka/server.log）
   搜索 "ERROR" 或 "WARN"

4. 检查 JVM GC（长时间 GC Stop-the-World 会导致超时）
   jstat -gcutil [pid] 1000

5. 检查磁盘 IO（磁盘满或 IO 过高）
   df -h && iostat -x 1 5

6. 增大 request.timeout.ms 和 delivery.timeout.ms
```

### 故障三：UnderReplicatedPartitions > 0

```
排查步骤：
1. 确认哪些 Partition 副本不足
   kafka-topics.sh --describe --under-replicated-partitions

2. 检查 Follower 所在 Broker 是否存活
   kafka-broker-api-versions.sh --bootstrap-server localhost:9092

3. 检查 Follower 的网络（带宽是否打满）
4. 检查 Follower 的磁盘（是否写入缓慢）
5. 检查 replica.lag.time.max.ms（是否设置过小）

6. 若 Broker 已恢复，ISR 会自动重新扩展，无需手动干预
   观察 IsrExpandsPerSec 指标是否>0
```

### 故障四：消息重复消费

```
常见原因：
1. Consumer 处理消息后崩溃，offset 未提交
   → 实现幂等消费（检查消息是否已处理）

2. max.poll.interval.ms 太小，处理中触发 Rebalance
   → 增大 max.poll.interval.ms 或减少每批处理数量

3. 手动提交 offset 的代码有 bug
   → 确保 ack.acknowledge() 在业务成功后调用

幂等消费示例：
  if (redisTemplate.setIfAbsent("processed:" + messageId, "1", 24, HOURS)) {
      processMessage(message);  // 只处理一次
  } else {
      log.warn("重复消息，跳过: {}", messageId);
  }
```


# Part 12：常见面试题 FAQ

## Q1：Kafka 为什么这么快？

```
Kafka 高性能的核心原因（4大设计）：

1. 顺序写磁盘（Sequential I/O）
   所有消息 append 到 Segment 文件末尾，无随机写。
   机械硬盘顺序写可达 600MB/s，与内存读写差距不大。

2. 零拷贝（Zero Copy）
   Consumer 消费时使用 sendfile 系统调用，
   数据从磁盘→内核缓冲区→网卡，跳过用户空间，节省2次CPU拷贝。

3. 页缓存（Page Cache）
   Producer 写入先到 OS Page Cache（内存），异步刷盘。
   Consumer 大概率直接从 Page Cache 读取，不触发磁盘 IO。

4. 批量处理 + 压缩
   Producer 端攒批（batch.size + linger.ms）后整批发送，
   Broker 存储整个 RecordBatch，Consumer 整批拉取。
   + 压缩（lz4/zstd）进一步减少网络和磁盘占用。
```

---

## Q2：Kafka 如何保证消息不丢失？

```
三个层面的保障：

Producer 端：
  acks=all            所有 ISR 副本写入后才确认
  retries > 0         发送失败自动重试
  enable.idempotence=true  幂等性，防止重试导致重复

Broker 端：
  replication.factor >= 3   至少3副本
  min.insync.replicas=2     至少2个ISR才允许写入
  unclean.leader.election.enable=false  不允许从 OSR 选举 Leader（防止数据丢失）

Consumer 端：
  enable.auto.commit=false  禁用自动提交
  先处理消息，再手动提交 offset
  处理失败时不提交，触发重试或进入死信队列
```

---

## Q3：Kafka 消息有序性如何保证？

```
Kafka 的有序性是分区级别的（Partition 内有序，不同 Partition 之间无序）。

保证同一业务实体消息有序的方法：
  发送时指定相同的 key，相同 key 的消息会路由到同一 Partition（默认分区器使用 key 的 hash）：
  producer.send(new ProducerRecord<>("orders", orderId, orderEvent));

注意事项：
  max.in.flight.requests.per.connection=1  → 保证单分区严格有序（关闭幂等性时）
  enable.idempotence=true → 开启幂等性后，max.in.flight 可设为5，Broker会自动排序

特殊情况：
  增加 Partition 数量后，原有相同 key 可能路由到不同 Partition，
  历史消息和新消息的顺序性会打乱（扩分区需谨慎）。
```

---

## Q4：Kafka Rebalance 是什么？如何避免频繁 Rebalance？

```
Rebalance（再平衡）是 Consumer Group 中分区所有权重新分配的过程。

触发场景：
  - Consumer 加入/离开 Group
  - Consumer 心跳超时（session.timeout.ms）
  - Consumer 处理超时（max.poll.interval.ms）
  - Topic 分区数变化

Rebalance 的危害：
  - Rebalance 期间所有 Consumer 停止消费（Stop-The-World）
  - 可能导致消息重复消费（未提交的 offset 重新消费）
  - 频繁 Rebalance 严重影响吞吐量

避免措施：
  session.timeout.ms=30000        增大心跳超时
  heartbeat.interval.ms=10000     心跳要 < timeout/3
  max.poll.interval.ms=600000     处理时间上限够用
  max.poll.records=100            减少每批处理量（确保在时间内处理完）
  partition.assignment.strategy=StickyAssignor  减少 Rebalance 时分区迁移
```

---

## Q5：ISR 是什么？ISR 收缩和扩展的条件？

```
ISR (In-Sync Replicas) = 与 Leader 保持同步的副本集合。

Follower 被踢出 ISR 的条件（满足之一）：
  超过 replica.lag.time.max.ms（默认30秒）未向 Leader 发送 Fetch 请求
  （Kafka 0.10+ 只用时间判断，废弃了 replica.lag.max.messages 参数）

Follower 重新加入 ISR 的条件：
  Follower 的 LEO 追上了 Leader 的 LEO
  且发送了 Fetch 请求证明自己活着

ISR 的重要性：
  acks=all 要求写入所有 ISR 副本才算成功（而非所有 AR 副本）
  Leader 选举只从 ISR 中选（保证不丢数据）
  HW = min(ISR中所有副本的LEO)（决定消费者可见的最大offset）
```

---

## Q6：HW 和 LEO 是什么？它们如何影响消费？

```
LEO (Log End Offset)：
  每个副本（Leader/Follower）下一条待写入消息的 offset（已写入最大offset + 1）

HW (High Watermark，高水位)：
  ISR 中所有副本 LEO 的最小值（= 消费者可见的最大 offset）

消费者只能消费 offset < HW 的消息，HW 以上的消息对消费者不可见。
这样即使 Leader 切换，新 Leader 的数据也至少有 HW 以下的消息，
防止消费者消费到后来被截断的消息（数据一致性）。

HW 更新延迟：
  HW 的更新需要至少一个 Fetch RTT（Follower Fetch → Leader更新HW → 下次Fetch通知Follower）
  这是 Kafka 端到端延迟的来源之一。
```

---

## Q7：Kafka 的事务消息是如何实现的？

```
核心组件：
  TransactionCoordinator：Broker端，管理事务状态
  __transaction_state：内部Topic，持久化事务状态（50个Partition）
  transactional.id：全局唯一，跨会话标识同一Producer

事务流程（5步）：
  1. initTransactions() → 向 TC 注册，获取 PID + Epoch
     TC 会 fence 掉旧的同一 transactional.id 的 Producer（ProducerFencedException）
  2. beginTransaction() → 本地标记，TC 无感知
  3. send() → 发送消息（携带 PID+Epoch，TC 记录涉及的 Partition）
  4. commitTransaction() → TC 写 PrepareCommit → 各Partition写Marker → TC写CompleteCommit
     abortTransaction() → TC 写 PrepareAbort  → 各Partition写Marker → TC写CompleteAbort
  5. Consumer 端 isolation.level=read_committed 只读 COMMIT 标记之前的消息

原子性保证：
  TransactionLog（__transaction_state）的 PrepareCommit 相当于分布式事务的"Prepared"状态
  即使 TC 崩溃，下一个 TC 也能从 Log 中恢复并继续完成或回滚事务
```

---

## Q8：Kafka 和 RocketMQ 的主要区别？

```
┌──────────────────┬─────────────────────────┬──────────────────────────┐
│ 特性             │ Kafka                   │ RocketMQ                 │
├──────────────────┼─────────────────────────┼──────────────────────────┤
│ 出身             │ LinkedIn 开源，Apache   │ 阿里开源，Apache         │
│ 语言             │ Scala + Java            │ Java                     │
│ 吞吐量           │ 极高（百万级TPS）        │ 高（十万级TPS）           │
│ 延迟             │ 毫秒级（批处理优化）     │ 微秒级（适合低延迟场景）  │
│ 消息顺序         │ 分区内有序              │ 队列内有序（支持全局顺序）│
│ 消息查询         │ 不支持按业务key查询     │ 支持 MessageId/Key 查询  │
│ 延迟消息         │ 不原生支持              │ 支持（18个延迟级别）      │
│ 死信队列         │ 需手动配置              │ 原生支持                  │
│ 事务消息         │ 支持（两阶段）          │ 支持（半消息+回查机制）   │
│ 适用场景         │ 日志收集、大数据管道    │ 电商交易、金融业务        │
│ 消费者模型       │ Pull                    │ Pull（长轮询）           │
│ 主从切换         │ ISR自动选举（秒级）     │ 需手动或Dledger（秒级）  │
└──────────────────┴─────────────────────────┴──────────────────────────┘
```

---

## Q9：如何实现 Kafka 的幂等消费？

```
幂等消费（Idempotent Consumer）= 同一消息处理多次，结果与处理一次相同。

常见实现方案：

方案一：业务主键去重（推荐）
  消费到消息后，用消息中的唯一ID（订单号、事件ID等）查 DB：
  if (orderRepo.existsById(event.getOrderId())) {
      return; // 已处理，跳过
  }
  // 处理 + 入库（用 DB 唯一约束兜底）

方案二：Redis 去重
  String key = "kafka:processed:" + messageId;
  Boolean isFirst = redisTemplate.opsForValue()
      .setIfAbsent(key, "1", Duration.ofDays(1));
  if (Boolean.TRUE.equals(isFirst)) {
      processMessage();
  }

方案三：消费 offset 幂等
  将已消费的 offset 存入 DB，消费前检查：
  if (offsetRepo.isProcessed(topic, partition, offset)) return;
  process(); // + 在同一 DB 事务中保存 offset

方案四：数据库唯一索引兜底
  业务表加唯一索引（如 message_id），INSERT 失败（重复键）时忽略错误。
```

---

## Q10：Kafka 分区数如何设计？

```
原则：
  1. 分区数 >= Consumer 实例数（否则有 Consumer 闲置）
  2. 分区数根据目标吞吐量设计（见 Part 10）
  3. 单 Broker 分区数不超过 1000~2000
  4. 分区数只能增加，不能减少（减少需重建 Topic）

计算公式：
  分区数 = max(生产者目标TPS / 单Partition生产TPS,
               消费者目标TPS / 单Partition消费TPS)

举例：
  生产目标：1,000,000 条/秒
  单 Partition 生产能力：100,000 条/秒
  消费者处理能力：50,000 条/秒（较慢）
  → 分区数 = max(1000000/100000, 1000000/50000) = max(10, 20) = 20 个

容量规划：
  总 Partition 数 = 预计峰值吞吐 / 单Partition吞吐 × 1.5（预留扩容空间）
  避免后期频繁扩 Partition（会影响 key 路由的顺序性）
```

---

## Q11：Kafka Controller 的作用是什么？Controller 宕机怎么办？

```
Controller 职责：
  1. 监控 Broker 存活（ZK 临时节点 / KRaft Raft日志）
  2. 分区 Leader 选举（Broker 宕机时）
  3. 分区/副本状态管理
  4. 下发元数据变更到所有 Broker（LeaderAndIsr、UpdateMetadata 请求）
  5. 处理 Topic 创建/删除/分区变更

Controller 宕机处理（ZooKeeper 模式）：
  ZooKeeper 检测到 /controller 临时节点消失
  所有 Broker 竞争重新创建 /controller 节点
  第一个成功的 Broker 成为新 Controller
  新 Controller 从 ZooKeeper 读取全量元数据，开始工作
  （这个过程可能需要几秒到几十秒，取决于集群规模）

Controller 宕机处理（KRaft 模式）：
  通过 Raft 协议选举新的 Active Controller
  其他 Controller 节点（Quorum成员）已有完整元数据日志
  故障转移时间更短（秒级）
```

---

## Q12：生产者的 acks 参数如何选择？

```
acks=0：
  Producer 发送后不等待任何确认，最高吞吐，可能丢消息。
  适用：日志采集等允许少量丢失的场景。

acks=1（默认）：
  Leader 写入 Page Cache 后即返回成功。
  若 Leader 未刷盘就宕机，消息丢失。
  适用：一般业务消息，平衡吞吐和可靠性。

acks=all（-1）：
  所有 ISR 副本写入后才确认。
  配合 min.insync.replicas=2 使用：至少2个副本写入成功。
  最强可靠性，延迟略高。
  适用：金融交易、订单等核心业务。

推荐组合（不丢消息）：
  acks=all
  min.insync.replicas=2
  replication.factor=3
  enable.idempotence=true
  retries=Integer.MAX_VALUE
  delivery.timeout.ms=120000
```

---

## Q13：Kafka 消费者组 Rebalance 期间，消息会丢失吗？

```
不会丢失，但可能重复消费。

Rebalance 过程分析：
  1. Consumer 停止消费，提交最新 offset（或等待自动提交间隔）
  2. 分区重新分配
  3. 新 Consumer 从上次提交的 offset 开始消费

可能重复消费的场景：
  Consumer A 消费了 offset=100~200 的消息，处理中，尚未提交
  此时触发 Rebalance，Consumer A 停止消费
  自动提交时，只提交到了 offset=150（上次自动提交点）
  Consumer B 接管该分区，从 offset=151 开始消费
  → offset 151~200 被重复消费

解决方案：
  使用手动提交（enable.auto.commit=false）
  Rebalance 前回调：ConsumerRebalanceListener.onPartitionsRevoked()
  在回调中立即 commitSync()，确保 Rebalance 前提交最新 offset：

  consumer.subscribe(topics, new ConsumerRebalanceListener() {
      @Override
      public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
          consumer.commitSync(currentOffsets); // 立即提交
      }
      @Override
      public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
          // 可以在这里重置状态
      }
  });
```

---

## Q14：什么是 Kafka 的 Log Compaction（日志压实）？

```
Log Compaction（日志压实）保留每个 Key 的最新值，删除旧的相同 Key 的消息。

适用场景：
  - 数据库 CDC（变更数据捕获）：保留每行的最新状态
  - Kafka Streams 状态存储：保留每个 key 的最新聚合结果
  - 配置中心：保留最新配置值

配置方式：
  log.cleanup.policy=compact          # 对整个 Broker 所有 Topic 生效
  # 或针对特定 Topic
  kafka-configs.sh --alter --entity-type topics --entity-name my-topic \
    --add-config cleanup.policy=compact

压实原理：
  后台 LogCleaner 线程定期扫描 Partition 中的旧 Segment（非活跃Segment）
  读取所有消息，只保留每个 key 最新的消息
  重写成新的 Segment 文件，删除旧文件

删除标记（Tombstone）：
  发送 value=null 的消息（key存在，value为null）表示删除该key
  压实后 tombstone 消息会在 delete.retention.ms 后被彻底删除

混合模式：
  log.cleanup.policy=delete,compact   # 同时启用删除和压实
  先压实（保留最新值），超过保留时间的 key 再删除
```

---

## Q15：Kafka 如何处理大消息？

```
默认限制：
  max.message.bytes=1048576 (1MB)  # Broker 和 Topic 级别
  fetch.max.bytes=52428800 (50MB)  # Consumer 单次拉取上限
  max.request.size=1048576 (1MB)   # Producer 单条消息上限

发送大消息的方案：

方案一：调大消息大小限制（简单但不推荐）
  Broker: message.max.bytes=10485760 (10MB)
  Producer: max.request.size=10485760
  Consumer: max.partition.fetch.bytes=10485760
  风险：大消息占用网络带宽，可能影响其他消息的延迟

方案二：消息分片（推荐）
  在应用层将大消息拆分成多个小消息，Consumer 重组：
  // Producer 端分片
  byte[] payload = largeData.getBytes();
  int chunkSize = 900 * 1024; // 900KB
  for (int i = 0; i * chunkSize < payload.length; i++) {
      byte[] chunk = Arrays.copyOfRange(payload, i*chunkSize,
          Math.min((i+1)*chunkSize, payload.length));
      producer.send(new ProducerRecord<>("large-msg", msgId + "-" + i, chunk));
  }

方案三：引用传递（最推荐）
  大文件存储在对象存储（S3/MinIO/HDFS）
  Kafka 只传递引用（URL 或对象路径）
  Consumer 拉取到 URL 后，从对象存储下载实际内容
  这是处理超大消息的最佳实践。
```

---

## Q16：说说 Kafka 的零拷贝原理

```
传统数据读取（4次拷贝）：
  1. DMA 将磁盘数据拷贝到内核缓冲区（Kernel Buffer）
  2. CPU 将内核缓冲区数据拷贝到用户缓冲区（User Buffer）—— read() 系统调用
  3. CPU 将用户缓冲区数据拷贝到 Socket 缓冲区（Socket Buffer）—— write() 系统调用
  4. DMA 将 Socket 缓冲区数据发送到网卡

零拷贝（sendfile，2次拷贝）：
  1. DMA 将磁盘数据拷贝到内核缓冲区
  2. DMA 将内核缓冲区数据直接发送到网卡（跳过用户空间和Socket缓冲区）
  （实际上只有2次DMA拷贝，完全没有CPU参与拷贝）

  Linux 内核支持的 sendfile + gather DMA：
  甚至不需要将数据拷贝到 Socket 缓冲区，
  只需要将内核缓冲区的文件描述符和偏移量信息发给网卡，
  网卡直接读取内核缓冲区数据发送。

Java 中实现：FileChannel.transferTo()
Kafka 中用于：Consumer 拉取消息时（数据从磁盘→网络）

注意：零拷贝对已压缩的消息才生效（若 Broker 需要重新压缩，则无法使用零拷贝）
建议：Broker compression.type=producer（保持 Producer 的压缩格式不变）
```

---

## Q17：Kafka 集群如何做数据迁移（跨集群）？

```
场景：将数据从旧 Kafka 集群迁移到新集群（版本升级/扩容）

方案一：MirrorMaker 2（官方，推荐）
  Kafka Connect 框架下的镜像工具，支持双向复制、过滤、转换

  配置示例（mm2.properties）：
  clusters=source,target
  source.bootstrap.servers=old-kafka:9092
  target.bootstrap.servers=new-kafka:9092
  source->target.enabled=true
  source->target.topics=.*   # 复制所有 Topic

  启动：
  connect-mirror-maker.sh mm2.properties

  特性：
  - 自动复制 offset（消费者组 offset 映射到新集群）
  - 支持 Topic 过滤和转换
  - 数据仅复制一次（精确一次语义）

方案二：双写（平滑迁移）
  Producer 同时写入新旧两个集群
  Consumer 逐步从旧集群切换到新集群
  稳定后停止向旧集群写入

方案三：Kafka Streams 复制（实时转换后迁移）
  适合需要对消息进行转换/过滤的迁移场景
```

---

## Q18：Kafka 的 __consumer_offsets 内部 Topic 是什么？

```
__consumer_offsets 是 Kafka 内部 Topic，用于存储消费者 Group 的 offset 信息。

特性：
  默认 50 个 Partition（offsets.topic.num.partitions=50）
  副本数由 offsets.topic.replication.factor 决定（默认1，生产应改为3）
  使用 log.cleanup.policy=compact（保留每个 key 的最新值）

存储格式：
  Key: [Group ID] + [Topic] + [Partition]
  Value: [Offset] + [Metadata] + [Timestamp]

路由规则（找到负责某 Group 的 GroupCoordinator）：
  partitionIndex = abs(groupId.hashCode()) % 50
  该 Partition 的 Leader 所在 Broker 即为该 Group 的 GroupCoordinator

查询方式：
  # 方法1：使用 kafka-consumer-groups.sh
  kafka-consumer-groups.sh --describe --group my-group

  # 方法2：直接消费 __consumer_offsets（调试用）
  kafka-console-consumer.sh \
    --bootstrap-server localhost:9092 \
    --topic __consumer_offsets \
    --formatter "kafka.coordinator.group.GroupMetadataManager\$OffsetsMessageFormatter" \
    --from-beginning
```

---

## Q19：Kafka 如何保证分区内消息的顺序性，以及在什么情况下会乱序？

```
Kafka 保证单个 Partition 内消息的追加顺序（FIFO）。

正常情况的顺序保证：
  Producer 使用相同的 key → 路由到同一 Partition → 顺序写入 → 顺序消费

可能导致乱序的情况：

1. max.in.flight.requests.per.connection > 1 且未开启幂等性
   Batch1 发送失败，Batch2 成功 → Batch1 重试后顺序在 Batch2 之后
   解决：enable.idempotence=true（Broker 会按 sequence 排序）
         或 max.in.flight=1（性能损失）

2. 增加 Partition 数量
   原来 key=A 路由到 Partition1，扩分区后可能路由到 Partition3
   历史消息在 Partition1，新消息在 Partition3，Consumer 无法保证顺序

3. 消费者并发处理
   拉取到100条消息，用线程池并发处理，完成顺序不一定是消费顺序
   解决：顺序场景不使用并发处理，或自己实现排序

4. 多个 Consumer 消费同一 Partition（不可能，Kafka 保证一对一）
```

---

## Q20：生产环境 Kafka 集群应该如何规划？

```
集群规模规划：

节点数量：
  最少3个 Broker（满足3副本 + Controller 高可用）
  推荐5个 Broker（更好的负载均衡和故障容忍）

硬件建议：
  CPU: 16~32 核（Kafka 是 IO 密集型，CPU 需求不高）
  内存: 64~128GB（给 OS Page Cache 留足空间，JVM Heap 只需 6-8GB）
  磁盘: SAS/SATA HDD（顺序写，RAID10或JBOD，JBOD更灵活）
       NVMe SSD（低延迟场景）
       避免 RAID5/RAID6（重建时间长，影响性能）
  网络: 万兆网卡（10Gbps+）

Topic 设计：
  replication.factor=3        生产环境标配
  min.insync.replicas=2       配合 acks=all
  分区数根据目标吞吐量计算

ZooKeeper（使用ZK模式时）：
  独立部署，3~5个节点（不与 Kafka 混部）
  SSD 磁盘（ZK 对延迟敏感）
  内存 4~8GB 即可

运维建议：
  部署监控（Prometheus + Grafana + 自定义 Kafka Exporter）
  配置告警（UnderReplicatedPartitions、OfflinePartitions、Lag 阈值）
  定期执行优先副本选举（auto.leader.rebalance.enable=true）
  多 AZ 部署副本（跨机房容灾）
  定期测试故障恢复（模拟 Broker 宕机，验证 Leader 切换时间）
```

---

> **文档完结**
>
> 本文档涵盖了 Kafka 从入门到精通所需的核心知识，包括架构设计、生产消费原理、
> 存储机制、高可用、Exactly-Once 语义、Spring Boot 集成、性能调优、监控运维
> 以及常见面试题，适合系统性学习和面试备考。
>
> 最后更新：2024年

