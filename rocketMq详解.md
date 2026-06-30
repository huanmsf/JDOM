# RocketMQ 分布式消息队列 - 新手完全指南

> 版本参考：RocketMQ 4.9.x / 5.x  
> 适合人群：Java 后端开发新手、有 Spring Boot 基础的开发者

---

## 目录

1. 什么是消息队列，为什么需要 RocketMQ
2. 核心概念
3. 整体架构
4. 底层原理深度解析
5. 环境搭建
6. 快速上手 - 第一条消息
7. 消息类型详解
8. 消费者模式详解
9. Spring Boot 集成
10. 完整业务案例
11. 生产环境最佳实践
12. 常见问题 FAQ

---

## 1. 什么是消息队列，为什么需要 RocketMQ

### 1.1 从一个真实场景说起

假设你开发一个电商系统，用户下单后需要同步执行：扣库存、建订单、发短信、加积分、通知仓库。所有步骤串行执行，任意一步失败都会导致整个下单失败，促销高并发还会直接压垮下游服务。

**引入消息队列后：**

```
用户下单
   |
   v
1. 扣减库存（同步，核心步骤）
2. 创建订单（同步，核心步骤）
   |
   v  发送消息（毫秒级）
+---------------------+
|    RocketMQ 消息队列  |
|  Topic: order-topic  |
+---+----------+-------+
    |          |      |
    v          v      v
3.短信服务  4.积分服务  5.仓库服务
（异步消费） （异步消费） （异步消费）

立即返回"下单成功"（用户只等了约 50ms）
```

**三大核心作用：**

| 作用 | 描述 | 类比 |
|------|------|------|
| **异步解耦** | 下游服务慢不影响上游响应速度 | 快递员只需把包裹放快递柜，不用等收件人 |
| **流量削峰** | 瞬时高并发请求放入队列，下游按自己节奏消费 | 超市排队结账，避免收银台崩溃 |
| **服务解耦** | 生产者不关心谁在消费，消费者可独立扩展 | 电视台只管播节目，不管谁在看 |

### 1.2 为什么选 RocketMQ

| 特性 | RocketMQ | Kafka | RabbitMQ |
|------|----------|-------|----------|
| 开发者 | 阿里巴巴（Apache） | LinkedIn | Pivotal |
| 消息顺序性 | 支持 | 分区内有序 | 不支持 |
| 延迟消息 | 原生支持 | 不支持 | 插件支持 |
| 事务消息 | 原生支持 | 不支持 | 不支持 |
| 消息回溯 | 支持 | 支持 | 不支持 |
| 吞吐量 | 十万级/秒 | 百万级/秒 | 万级/秒 |
| 适用场景 | 电商、金融、订单 | 日志、大数据 | 企业内部集成 |

**结论：** 电商、金融、订单等业务场景，RocketMQ 是最优选择。

---

## 2. 核心概念

在学习架构之前，先理解这些基本术语：

```
+------------------+----------------------------------------+
|  Producer         | 消息生产者，发送消息的一方               |
|  （生产者）        | 例如：订单服务发送"订单创建"消息          |
+------------------+----------------------------------------+
|  Consumer         | 消息消费者，接收并处理消息的一方           |
|  （消费者）        | 例如：短信服务监听"订单创建"消息          |
+------------------+----------------------------------------+
|  Topic            | 消息的分类标签，生产者和消费者通过         |
|  （主题）          | Topic 找到彼此，例如 "order-topic"       |
+------------------+----------------------------------------+
|  Tag              | Topic 下的二级分类，消费者可按 Tag 过滤   |
|  （标签）          | 例如 Topic=order, Tag=VIP/NORMAL        |
+------------------+----------------------------------------+
|  MessageQueue     | Topic 的物理分区，默认 4 个队列           |
|  （消息队列）      | 是实现并行消费和顺序消费的基础             |
+------------------+----------------------------------------+
|  ConsumerGroup    | 消费者组，组内各实例分摊消费，每条消息     |
|  （消费者组）      | 只被组内某一个消费者处理                  |
+------------------+----------------------------------------+
|  NameServer       | 服务注册中心，存储 Broker 的路由信息      |
|  （命名服务器）    | 类似 Zookeeper，但更轻量               |
+------------------+----------------------------------------+
|  Broker           | 消息存储服务器，真正存储和转发消息         |
|  （消息代理）      | 是 RocketMQ 的核心组件                  |
+------------------+----------------------------------------+
```

### 2.1 Topic 与 MessageQueue 的关系

```
Topic: "order-topic"
|
+-- MessageQueue 0  ----> 存储在 Broker-A
+-- MessageQueue 1  ----> 存储在 Broker-A
+-- MessageQueue 2  ----> 存储在 Broker-B
+-- MessageQueue 3  ----> 存储在 Broker-B

一个 Topic 的消息分散在多个 Broker 的多个 Queue 上
这就是 RocketMQ 高吞吐量的关键：并行写入、并行读取
```

### 2.2 ConsumerGroup 机制

```
Topic: "order-topic" 有 4 个 MessageQueue

消费者组 A（短信服务，3 个实例）：
  Consumer-A1 消费 Queue0、Queue1
  Consumer-A2 消费 Queue2
  Consumer-A3 消费 Queue3

消费者组 B（积分服务，2 个实例）：
  Consumer-B1 消费 Queue0、Queue1
  Consumer-B2 消费 Queue2、Queue3

每个 ConsumerGroup 都消费全量消息，组内各实例分摊负载
```

---

## 3. 整体架构

### 3.1 四大核心组件架构图

```
+-------------------------------------------------------------------+
|                      RocketMQ 架构图                              |
|                                                                   |
|  +------------+  +------------+  +------------+                  |
|  | NameServer |  | NameServer |  | NameServer |  (无状态集群)     |
|  +------+-----+  +------+-----+  +------+-----+                  |
|         |               |               |                        |
|   --------- 路由注册（Broker 每 30 秒心跳上报） --------           |
|                         |                                        |
|  +----------------------------------------------------------+    |
|  |                  Broker 集群                              |    |
|  |  +---------------+     +---------------+                 |    |
|  |  |  Broker-A     |     |  Broker-B     |                 |    |
|  |  |  (Master)     |     |  (Master)     |                 |    |
|  |  |  CommitLog    |     |  CommitLog    |                 |    |
|  |  |  ConsumeQueue |     |  ConsumeQueue |                 |    |
|  |  +------+--------+     +------+--------+                 |    |
|  |         | 同步复制              | 同步复制                 |    |
|  |  +------+--------+     +------+--------+                 |    |
|  |  |  Broker-A     |     |  Broker-B     |                 |    |
|  |  |  (Slave)      |     |  (Slave)      |                 |    |
|  |  +---------------+     +---------------+                 |    |
|  +----------------------------------------------------------+    |
|                        ^             ^                           |
|                发送消息  |             |  消费消息                 |
|  +-------------+       |             |      +-------------+      |
|  |  Producer   |-------+             +------| Consumer    |      |
|  +-------------+                            +-------------+      |
+-------------------------------------------------------------------+
```

### 3.2 各组件职责

**NameServer（命名服务器）**
- 轻量级服务注册与路由发现中心
- Broker 启动时向所有 NameServer 注册（定时心跳）
- Producer/Consumer 从 NameServer 获取路由信息
- 节点之间**互不通信**（无状态），任意节点挂掉不影响整体

**Broker（消息代理）**
- 核心组件，负责消息的接收、存储、转发
- Master 负责读写，Slave 负责备份（也可读）
- 每 30 秒向所有 NameServer 发送心跳

**Producer（生产者）**
- 从 NameServer 获取 Topic 的路由信息
- 按路由策略选择 MessageQueue 发送消息
- 本地缓存路由信息，每 30 秒更新一次

**Consumer（消费者）**
- 从 NameServer 获取路由信息
- 向 Broker 长轮询拉取消息
- 消费后向 Broker 提交消费进度（Offset）

---

## 4. 底层原理深度解析

### 4.1 消息存储 - CommitLog（性能之源）

```
传统方式（每个 Queue 一个文件，随机写）：
  Queue0 写 file0、Queue1 写 file1 -> 磁盘频繁寻址，性能差

RocketMQ 的方式（所有消息顺序写入同一文件）：

所有消息 ----------------> CommitLog（顺序追加写）
（无论哪个 Topic、哪个 Queue）

CommitLog 文件内容示意：
+-----------------------------------------------------------+
| [消息1][消息2][消息3][消息4][消息5]...（全部顺序写入）     |
|  Q0      Q2     Q1     Q0     Q3                          |
+-----------------------------------------------------------+
        ^ 顺序写磁盘，速度接近内存（600MB/s+）

文件规则：
  每个文件大小 = 1GB，写满后创建新文件
  文件名 = 起始偏移量（20 位数字）
    00000000000000000000  (第 0 字节)
    00000000001073741824  (第 1GB 字节)
```

**顺序写为什么这么快？**

```
随机写磁盘：磁头不停寻址，约 100 次/秒（慢）
顺序写磁盘：磁头几乎不移动，约 600MB/s（快）

CommitLog 顺序写 + PageCache 内存映射 = RocketMQ 高吞吐量的基础
```

### 4.2 ConsumeQueue - 消费索引

CommitLog 把所有消息混在一起，消费者如何找到自己 Topic 的消息？

```
CommitLog（混合存储）：
  [offset=0,  Topic=order, Queue=0, 消息体...]
  [offset=20, Topic=user,  Queue=0, 消息体...]
  [offset=55, Topic=order, Queue=1, 消息体...]
  [offset=90, Topic=order, Queue=0, 消息体...]

ConsumeQueue（消费索引，每个 Topic 每个 Queue 一个文件）：

Topic=order, Queue=0 的索引文件：
+--------------------------------------------+
| [CommitLog偏移量=0,  消息大小=20, tagHash]  |  <- 每条记录固定 20 字节
| [CommitLog偏移量=90, 消息大小=35, tagHash]  |
+--------------------------------------------+

消费流程：
  1. Consumer 记录消费到第 N 条（Offset = N）
  2. 读 ConsumeQueue[N] 得到 CommitLog 的物理偏移量
  3. 根据物理偏移量从 CommitLog 读取完整消息
```

### 4.3 IndexFile - 消息检索索引

用于根据消息 Key 快速查找消息（类似数据库索引）：

```
IndexFile 结构（哈希表 + 链表）：

+------------------------------------------------+
|  Header（40字节）：时间范围、条目数量等            |
+------------------------------------------------+
|  HashSlot（500 万个槽）: slot[hash % 500万]    |
+------------------------------------------------+
|  Index 条目（每条 20 字节）：                    |
|  [keyHash | CommitLog偏移量 | 时间差 | 前驱]    |
+------------------------------------------------+

查询 Key="orderId_12345" 的消息：
  1. hash("orderId_12345") % 500万 -> 找到 HashSlot
  2. 沿链表找匹配的 Index 条目
  3. 获取 CommitLog 偏移量，读取消息
```

### 4.4 消息发送完整流程（Producer 视角）

```
① 获取路由信息
   本地缓存有 Topic 路由？
     有 -> 直接使用（每 30 秒后台刷新）
     无 -> 向 NameServer 查询并缓存

② 选择 MessageQueue（默认轮询策略）
   第1条 -> Queue0 (Broker-A)
   第2条 -> Queue1 (Broker-A)
   第3条 -> Queue2 (Broker-B)
   ... 循环

③ 建立 TCP 连接，发送消息到 Broker

④ Broker 写入 PageCache（内存映射文件）
   根据刷盘策略决定何时落盘

⑤ 返回结果
   同步发送：等待 Broker 确认 ACK 后返回
   异步发送：注册回调，立即返回
   单向发送：不等任何确认（用于日志等场景）
```

### 4.5 刷盘机制

```
+------------------+----------------------------------------+
|  同步刷盘         | 写入 PageCache 后立即刷盘，再返回 ACK   |
|  (SYNC_FLUSH)    | 优点：消息绝对不会丢失                  |
|                  | 缺点：性能较低（磁盘 IO 阻塞）           |
|                  | 场景：金融、支付等关键业务               |
+------------------+----------------------------------------+
|  异步刷盘（默认）  | 写入 PageCache 后立即返回 ACK，         |
|  (ASYNC_FLUSH)   | 后台线程异步刷盘                        |
|                  | 优点：吞吐量高，性能好                   |
|                  | 缺点：宕机可能丢失少量消息               |
|                  | 场景：日志、通知等非关键业务              |
+------------------+----------------------------------------+
```

### 4.6 消息消费流程（Consumer 视角）

```
Push 模式底层实现（实际上是 Pull + 长轮询）：

Consumer 启动
    |
    v
向 Broker 发送 PULL 请求
    +-- 有消息 -> 立即返回消息数据
    +-- 无消息 -> 挂起请求（长轮询，默认 15 秒）
                  有新消息到来时立即唤醒并返回

收到消息 -> 放入 ProcessQueue（本地缓冲区）
    |
    v
消费线程池执行用户注册的 MessageListener
    |
    v
成功 -> 提交 Offset（消费进度持久化）
失败 -> 消息进入重试队列（%RETRY%ConsumerGroup）
```

### 4.7 消费者负载均衡（Rebalance）

```
场景：Topic 有 4 个 Queue，消费者组有 3 个实例

初始状态：
  Consumer-1 -> Queue0、Queue1
  Consumer-2 -> Queue2
  Consumer-3 -> Queue3

Consumer-4 加入（触发 Rebalance）：
  Consumer-1 -> Queue0
  Consumer-2 -> Queue1
  Consumer-3 -> Queue2
  Consumer-4 -> Queue3

Consumer-1 宕机（触发 Rebalance）：
  Consumer-2 -> Queue0、Queue1
  Consumer-3 -> Queue2
  Consumer-4 -> Queue3

重要：Queue 数量是并行消费度的上限
      消费者实例数 > Queue 数量时，多余实例空闲
```

### 4.8 消息重试与死信队列

```
消费者返回 RECONSUME_LATER
         |
         v
Broker 放入重试队列：%RETRY%{ConsumerGroup}
按延迟等级重新投递（共 16 个等级）：
  第  1 次重试：10  秒后
  第  2 次重试：30  秒后
  第  3 次重试：1   分钟后
  第  4 次重试：2   分钟后
  ...
  第 16 次重试：2   小时后
         |
重试 16 次仍失败
         |
         v
进入死信队列：%DLQ%{ConsumerGroup}
（需要人工介入，分析并处理失败原因）
```

---

## 5. 环境搭建

### 5.1 前置要求

- JDK 1.8+（RocketMQ 5.x 需要 JDK 11+）
- Maven 3.x
- 内存至少 4GB（NameServer + Broker）

### 5.2 单机快速部署（Linux/Mac）

**Step 1：下载并解压**

```bash
wget https://dist.apache.org/repos/dist/release/rocketmq/4.9.7/rocketmq-all-4.9.7-bin-release.zip
unzip rocketmq-all-4.9.7-bin-release.zip
cd rocketmq-all-4.9.7-bin-release
```

**Step 2：调整内存（开发环境必须做，否则 OOM）**

```bash
# 修改 NameServer 内存（找到 -Xms4g -Xmx4g，改小）
vim bin/runserver.sh
# 改为：-Xms512m -Xmx512m -Xmn256m

# 修改 Broker 内存
vim bin/runbroker.sh
# 改为：-Xms1g -Xmx1g -Xmn512m
```

**Step 3：启动 NameServer**

```bash
nohup sh bin/mqnamesrv &
tail -f ~/logs/rocketmqlogs/namesrv.log
# 出现 "The Name Server boot success" 表示成功
```

**Step 4：启动 Broker**

```bash
nohup sh bin/mqbroker -n localhost:9876 autoCreateTopicEnable=true &
tail -f ~/logs/rocketmqlogs/broker.log
# 出现 "boot success" 表示成功
```

**Step 5：验证**

```bash
export NAMESRV_ADDR=localhost:9876
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Producer
sh bin/tools.sh org.apache.rocketmq.example.quickstart.Consumer
```

### 5.3 Windows 部署

```bat
:: 启动 NameServer
start bin\mqnamesrv.cmd

:: 新开窗口启动 Broker
start bin\mqbroker.cmd -n 127.0.0.1:9876 autoCreateTopicEnable=true
```

### 5.4 可视化管理工具（RocketMQ Dashboard）

```bash
git clone https://github.com/apache/rocketmq-dashboard.git
cd rocketmq-dashboard
# 修改 src/main/resources/application.properties：
# rocketmq.config.namesrvAddr=127.0.0.1:9876
mvn spring-boot:run
# 访问 http://localhost:8080
```

Dashboard 功能：查看 Broker 状态、Topic 列表、消息查询、消费进度、死信队列等。

---

## 6. 快速上手 - 第一条消息

### 6.1 Maven 依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-client</artifactId>
    <version>4.9.7</version>
</dependency>
```

### 6.2 第一个生产者

```java
import org.apache.rocketmq.client.producer.DefaultMQProducer;
import org.apache.rocketmq.client.producer.SendResult;
import org.apache.rocketmq.common.message.Message;
import java.nio.charset.StandardCharsets;

public class HelloProducer {
    public static void main(String[] args) throws Exception {

        // 1. 创建生产者，指定生产者组名
        DefaultMQProducer producer = new DefaultMQProducer("hello-producer-group");

        // 2. 设置 NameServer 地址
        producer.setNamesrvAddr("localhost:9876");

        // 3. 启动生产者
        producer.start();

        // 4. 创建消息
        //    参数一：Topic 名称
        //    参数二：Tag 标签（可选，用于过滤）
        //    参数三：消息体（字节数组）
        Message message = new Message(
            "hello-topic",
            "TagA",
            "Hello RocketMQ!".getBytes(StandardCharsets.UTF_8)
        );

        // 5. 同步发送，等待 Broker 确认
        SendResult result = producer.send(message);
        System.out.println("发送结果：" + result);
        // 输出：SendResult [sendStatus=SEND_OK, msgId=..., ...]

        // 6. 关闭生产者
        producer.shutdown();
    }
}
```

### 6.3 第一个消费者

```java
import org.apache.rocketmq.client.consumer.DefaultMQPushConsumer;
import org.apache.rocketmq.client.consumer.listener.*;
import org.apache.rocketmq.common.message.MessageExt;
import java.util.List;

public class HelloConsumer {
    public static void main(String[] args) throws Exception {

        // 1. 创建消费者，指定消费者组名
        DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("hello-consumer-group");

        // 2. 设置 NameServer 地址
        consumer.setNamesrvAddr("localhost:9876");

        // 3. 订阅 Topic，"*" 表示接收该 Topic 下所有 Tag 的消息
        consumer.subscribe("hello-topic", "*");

        // 4. 注册消息监听器（并发消费）
        consumer.registerMessageListener(new MessageListenerConcurrently() {
            @Override
            public ConsumeConcurrentlyStatus consumeMessage(
                    List<MessageExt> messages,
                    ConsumeConcurrentlyContext context) {

                for (MessageExt msg : messages) {
                    System.out.println("收到消息：" + new String(msg.getBody()));
                    System.out.println("  Topic: " + msg.getTopic());
                    System.out.println("  Tag:   " + msg.getTags());
                    System.out.println("  msgId: " + msg.getMsgId());
                }

                // 返回消费成功
                return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;

                // 消费失败，触发重试：
                // return ConsumeConcurrentlyStatus.RECONSUME_LATER;
            }
        });

        // 5. 启动消费者（持续运行，监听消息）
        consumer.start();
        System.out.println("消费者已启动，等待消息...");
    }
}
```

---

## 7. 消息类型详解

### 7.1 普通消息的三种发送方式

```java
// 方式一：同步发送（最常用）—— 等待 Broker 确认后返回
SendResult result = producer.send(message);
System.out.println("状态：" + result.getSendStatus()); // SEND_OK

// 方式二：异步发送 —— 立即返回，通过回调获取结果
producer.send(message, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        System.out.println("发送成功：" + sendResult.getMsgId());
    }
    @Override
    public void onException(Throwable e) {
        System.err.println("发送失败：" + e.getMessage());
    }
});

// 方式三：单向发送 —— 只管发，不关心结果（适合日志、监控场景）
producer.sendOneway(message);
```

### 7.2 延迟消息（定时消息）

**使用场景：** 下单后 30 分钟未支付，自动取消订单。

RocketMQ 4.x 支持 18 个固定延迟等级（不支持任意时间延迟）：

```
等级:  1    2    3    4    5   6   7   8   9  10  11  12  13  14  15   16   17  18
时间: 1s   5s  10s  30s  1m  2m  3m  4m  5m  6m  7m  8m  9m 10m 20m  30m  1h  2h
```

```java
Message message = new Message("order-topic", "TagA",
    orderId.getBytes(StandardCharsets.UTF_8));

// 设置延迟等级 16 = 30 分钟后投递
message.setDelayTimeLevel(16);

producer.send(message);
System.out.println("延迟消息已发送，30分钟后消费者才会收到");
```

```java
// 消费端正常处理即可，延迟对消费端透明
consumer.registerMessageListener((messages, context) -> {
    for (MessageExt msg : messages) {
        String orderId = new String(msg.getBody());
        orderService.checkAndCancelIfTimeout(orderId);
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

### 7.3 顺序消息

**使用场景：** 同一个订单的「创建->支付->发货」消息必须按顺序消费。

```java
// 生产端：相同 orderId 的消息发到同一个 Queue
producer.send(message, new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> queues,
                               Message msg, Object arg) {
        Long orderId = (Long) arg;
        // 取模保证相同 orderId 始终选同一个 Queue
        int index = (int) (orderId % queues.size());
        return queues.get(index);
    }
}, orderId); // orderId 作为选择器参数
```

```java
// 消费端：使用 MessageListenerOrderly（同一 Queue 内串行消费）
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(
            List<MessageExt> messages, ConsumeOrderlyContext context) {

        for (MessageExt msg : messages) {
            System.out.println("顺序消费：" + new String(msg.getBody()));
        }
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

**注意事项：**

```
顺序消息的代价：
  - 同一 Queue 内消息串行消费，吞吐量较普通消息低
  - 消费失败会阻塞该 Queue 后续消息（保证顺序的代价）

适用场景：
  - 订单状态流转（创建->支付->发货->完成）
  - 数据库 binlog 同步
  - 有严格前后依赖关系的业务流程
```

### 7.4 事务消息

**使用场景：** 保证「本地数据库操作」和「消息发送」的原子性。

```
两阶段提交流程：

第一阶段：发送"半消息"（Broker 存储但对消费者不可见）
  Producer -----------------------------------------> Broker
  Broker 存储消息，状态为 "prepared"（消费者看不到）

第二阶段：执行本地事务
  Producer 执行本地数据库操作（如扣库存）

第三阶段：根据本地事务结果提交/回滚
  本地事务成功 -> 发送 COMMIT -> 消息对消费者可见
  本地事务失败 -> 发送 ROLLBACK -> Broker 删除消息

补偿机制（Producer 宕机时）：
  Broker 定时回查 Producer 本地事务状态
  根据回查结果自动提交或回滚
```

```java
// 事务消息生产者
TransactionMQProducer producer = new TransactionMQProducer("tx-producer-group");
producer.setNamesrvAddr("localhost:9876");

producer.setTransactionListener(new TransactionListener() {

    // 第二步：执行本地事务
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        try {
            inventoryService.deduct(new String(msg.getBody()));
            return LocalTransactionState.COMMIT_MESSAGE;   // 本地事务成功，提交
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE; // 本地事务失败，回滚
        }
    }

    // 补偿：Broker 未收到 COMMIT/ROLLBACK 时回查
    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        boolean isDone = inventoryService.isDeducted(new String(msg.getBody()));
        return isDone
            ? LocalTransactionState.COMMIT_MESSAGE
            : LocalTransactionState.ROLLBACK_MESSAGE;
    }
});

producer.start();

// 第一步：发送事务消息（半消息）
Message message = new Message("inventory-topic", "deduct",
    orderId.getBytes(StandardCharsets.UTF_8));
producer.sendMessageInTransaction(message, null);
```

---

## 8. 消费者模式详解

### 8.1 集群消费 vs 广播消费

```java
// 集群消费（默认）：同一组内每条消息只被消费一次
// 适合：订单处理、库存扣减等不能重复执行的业务
consumer.setMessageModel(MessageModel.CLUSTERING);

// 广播消费：同一组内每个实例都消费全量消息
// 适合：本地缓存刷新、配置更新推送等需要所有节点都执行的操作
consumer.setMessageModel(MessageModel.BROADCASTING);
```

```
集群消费（4 Queue，3 Consumer实例）：
  Consumer-1 消费 Queue0、Queue1
  Consumer-2 消费 Queue2
  Consumer-3 消费 Queue3
  效果：每条消息只处理一次

广播消费（4 Queue，3 Consumer实例）：
  Consumer-1、Consumer-2、Consumer-3 各自消费全部 4 个 Queue
  效果：每条消息被每个实例各处理一次
```

### 8.2 消息过滤

```java
// Tag 过滤（最常用，在 Broker 端过滤，效率高）
consumer.subscribe("order-topic", "VIP || NORMAL"); // 接收 VIP 或 NORMAL
consumer.subscribe("order-topic", "VIP");           // 只接收 VIP
consumer.subscribe("order-topic", "*");             // 接收所有 Tag

// SQL92 过滤（需 Broker 开启 enablePropertyFilter=true）
consumer.subscribe("order-topic",
    MessageSelector.bySql("amount > 100 AND userType = 'VIP'"));
```

### 8.3 消息幂等性（非常重要！）

RocketMQ 承诺「至少一次投递」，网络异常等情况可能导致重复消费，**消费端必须做幂等处理**：

```java
consumer.registerMessageListener((messages, context) -> {
    for (MessageExt msg : messages) {
        String msgId = msg.getMsgId();        // RocketMQ 的消息 ID
        String bizKey = msg.getKeys();        // 业务唯一 Key（发送时设置）

        // 方案一：Redis SET NX（推荐，原子操作）
        Boolean first = redis.opsForValue()
            .setIfAbsent("consumed:" + msgId, "1", Duration.ofHours(24));
        if (Boolean.FALSE.equals(first)) {
            continue; // 已消费，跳过
        }

        // 方案二：数据库唯一索引
        // INSERT IGNORE INTO message_consumed(msg_id) VALUES(?)
        // 插入成功才处理，插入失败说明已处理

        // 方案三：业务状态检查（直接查数据库状态判断是否已处理）

        doProcess(msg); // 执行业务逻辑
    }
    return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
});
```

```java
// 发送时设置业务 Key，便于追踪和幂等判断
Message message = new Message(
    "order-topic",
    "TagA",
    "ORDER_001",   // Keys：业务唯一标识（可按此 Key 在 Dashboard 查消息）
    orderJson.getBytes(StandardCharsets.UTF_8)
);
```

### 8.4 消费起始位点

```java
// 仅第一次启动该消费者组时生效

// 从最新消息开始（默认，不消费历史消息）
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_LAST_OFFSET);

// 从最早的消息开始消费（回溯历史消息）
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_FIRST_OFFSET);

// 从指定时间点开始消费
consumer.setConsumeFromWhere(ConsumeFromWhere.CONSUME_FROM_TIMESTAMP);
consumer.setConsumeTimestamp("20240101120000"); // yyyyMMddHHmmss
```

---

## 9. Spring Boot 集成

### 9.1 添加依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.2.3</version>
</dependency>
```

### 9.2 配置文件

```yaml
rocketmq:
  name-server: 127.0.0.1:9876
  producer:
    group: my-producer-group
    send-message-timeout: 3000        # 发送超时（毫秒）
    retry-times-when-send-failed: 2   # 同步发送失败重试次数
    max-message-size: 4194304         # 最大消息体 4MB
```

### 9.3 生产者（Spring Boot 方式）

```java
import org.apache.rocketmq.spring.core.RocketMQTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.stereotype.Service;

@Service
public class OrderMessageProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    // destination 格式："topic" 或 "topic:tag"

    /** 发送简单字符串 */
    public void sendSimple(String orderId) {
        rocketMQTemplate.convertAndSend("order-topic:CREATE", orderId);
    }

    /** 发送对象（自动序列化为 JSON） */
    public void sendObject(OrderDTO order) {
        rocketMQTemplate.convertAndSend("order-topic:CREATE", order);
    }

    /** 同步发送，获取发送结果 */
    public void sendSync(OrderDTO order) {
        SendResult result = rocketMQTemplate.syncSend("order-topic:CREATE", order);
        System.out.println("发送状态：" + result.getSendStatus());
    }

    /** 异步发送 */
    public void sendAsync(OrderDTO order) {
        rocketMQTemplate.asyncSend("order-topic", order, new SendCallback() {
            @Override
            public void onSuccess(SendResult result) {
                System.out.println("发送成功：" + result.getMsgId());
            }
            @Override
            public void onException(Throwable e) {
                System.err.println("发送失败：" + e.getMessage());
            }
        });
    }

    /** 延迟消息（等级 16 = 30 分钟后投递） */
    public void sendDelay(String orderId) {
        rocketMQTemplate.syncSend(
            "order-topic",
            MessageBuilder.withPayload(orderId).build(),
            3000,  // 发送超时（毫秒）
            16     // 延迟等级
        );
    }

    /** 顺序消息（相同 orderId 进同一 Queue） */
    public void sendOrderly(OrderDTO order) {
        rocketMQTemplate.syncSendOrderly(
            "order-topic",
            order,
            order.getOrderId()  // hashKey：相同 key 进入同一 Queue
        );
    }
}
```

### 9.4 消费者（Spring Boot 方式）

```java
import org.apache.rocketmq.spring.annotation.RocketMQMessageListener;
import org.apache.rocketmq.spring.core.RocketMQListener;
import org.springframework.stereotype.Service;

/**
 * 使用注解声明消费者，极简配置
 * - topic：订阅的主题
 * - consumerGroup：消费者组（每个消费者类使用唯一组名）
 * - selectorExpression：Tag 过滤（默认 * 接收全部）
 */
@Service
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-consumer-group",
    selectorExpression = "CREATE || UPDATE"
)
public class OrderConsumer implements RocketMQListener<OrderDTO> {

    @Override
    public void onMessage(OrderDTO order) {
        // 框架自动将 JSON 反序列化为 OrderDTO
        System.out.println("收到订单：" + order.getOrderId());

        processOrder(order);

        // 方法正常返回 = CONSUME_SUCCESS
        // 抛出异常    = RECONSUME_LATER（触发自动重试）
    }

    private void processOrder(OrderDTO order) {
        // 业务逻辑...
    }
}
```

```java
/** 顺序消息消费者 */
@Service
@RocketMQMessageListener(
    topic = "order-topic",
    consumerGroup = "order-orderly-consumer-group",
    consumeMode = ConsumeMode.ORDERLY  // 顺序消费模式
)
public class OrderOrderlyConsumer implements RocketMQListener<String> {

    @Override
    public void onMessage(String message) {
        System.out.println("顺序消费：" + message);
    }
}
```

---

## 10. 完整业务案例

### 案例一：电商下单异步处理

**需求：** 用户下单后，核心逻辑同步执行（扣库存、建订单），非核心逻辑异步执行（发短信、加积分、通知仓库），各服务互不影响。

#### 项目结构

```
src/main/java/com/example/
├── controller/
│   └── OrderController.java          # 接收下单 HTTP 请求
├── service/
│   └── OrderService.java             # 核心下单逻辑
├── mq/
│   ├── OrderMessageProducer.java     # 消息生产者
│   ├── SmsConsumer.java              # 短信通知消费者
│   ├── PointsConsumer.java           # 积分服务消费者
│   └── WarehouseConsumer.java        # 仓库通知消费者
└── dto/
    └── OrderCreatedEvent.java        # 消息体 DTO
```

#### OrderCreatedEvent.java（消息体）

```java
package com.example.dto;

import java.math.BigDecimal;
import java.time.LocalDateTime;

public class OrderCreatedEvent {
    private String orderId;
    private String userId;
    private String userPhone;
    private BigDecimal amount;
    private LocalDateTime createTime;

    public OrderCreatedEvent(String orderId, String userId,
                              String userPhone, BigDecimal amount) {
        this.orderId = orderId;
        this.userId = userId;
        this.userPhone = userPhone;
        this.amount = amount;
        this.createTime = LocalDateTime.now();
    }

    public String getOrderId() { return orderId; }
    public String getUserId() { return userId; }
    public String getUserPhone() { return userPhone; }
    public BigDecimal getAmount() { return amount; }
}
```

#### OrderService.java（核心下单逻辑）

```java
@Service
public class OrderService {

    @Autowired
    private InventoryService inventoryService;

    @Autowired
    private OrderRepository orderRepository;

    @Autowired
    private ApplicationEventPublisher eventPublisher;

    @Transactional
    public String createOrder(CreateOrderRequest request) {
        String orderId = generateOrderId();

        // 扣减库存（同步，强一致性要求）
        inventoryService.deduct(request.getSkuId(), request.getQuantity());

        // 创建订单记录
        Order order = new Order(orderId, request);
        orderRepository.save(order);

        // 发布事件：在事务提交后再发送 MQ 消息（防止事务回滚但消息已发）
        OrderCreatedEvent event = new OrderCreatedEvent(
            orderId, request.getUserId(),
            request.getUserPhone(), order.getTotalAmount()
        );
        eventPublisher.publishEvent(event);

        return orderId;
    }
}
```

#### OrderMessageProducer.java

```java
@Component
public class OrderMessageProducer {

    @Autowired
    private RocketMQTemplate rocketMQTemplate;

    private static final String TOPIC = "order-created-topic";

    /**
     * 监听 Spring 事务提交后的事件，此时再发送 MQ 消息
     * 确保事务成功提交后，消息才发出，避免事务回滚导致的数据不一致
     */
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendOrderCreatedEvent(OrderCreatedEvent event) {
        rocketMQTemplate.convertAndSend(TOPIC + ":CREATED", event);
        System.out.println("订单创建消息已发送，orderId=" + event.getOrderId());
    }
}
```

#### SmsConsumer.java（短信通知）

```java
@Service
@RocketMQMessageListener(
    topic = "order-created-topic",
    consumerGroup = "sms-consumer-group",
    selectorExpression = "CREATED"
)
public class SmsConsumer implements RocketMQListener<OrderCreatedEvent> {

    @Autowired
    private SmsService smsService;

    @Override
    public void onMessage(OrderCreatedEvent event) {
        System.out.println("【短信服务】处理订单：" + event.getOrderId());

        String content = String.format(
            "您的订单 %s 已创建成功，金额 %.2f 元，请及时付款。",
            event.getOrderId(), event.getAmount()
        );
        smsService.send(event.getUserPhone(), content);
    }
}
```

#### PointsConsumer.java（积分服务）

```java
@Service
@RocketMQMessageListener(
    topic = "order-created-topic",
    consumerGroup = "points-consumer-group",  // 不同服务用不同 ConsumerGroup！
    selectorExpression = "CREATED"
)
public class PointsConsumer implements RocketMQListener<OrderCreatedEvent> {

    @Autowired
    private PointsService pointsService;

    @Override
    public void onMessage(OrderCreatedEvent event) {
        // 每消费 1 元积 10 分
        long points = event.getAmount().longValue() * 10;
        pointsService.addPoints(event.getUserId(), points, event.getOrderId());
        System.out.println("【积分服务】用户 " + event.getUserId() + " 增加积分：" + points);
    }
}
```

#### WarehouseConsumer.java（仓库备货）

```java
@Service
@RocketMQMessageListener(
    topic = "order-created-topic",
    consumerGroup = "warehouse-consumer-group",
    selectorExpression = "CREATED"
)
public class WarehouseConsumer implements RocketMQListener<OrderCreatedEvent> {

    @Override
    public void onMessage(OrderCreatedEvent event) {
        System.out.println("【仓库服务】收到备货通知，订单：" + event.getOrderId());
        // 调用仓库系统 API 开始备货...
    }
}
```

#### 消息流向总览

```
POST /order/create
        |
        v
OrderService（核心同步逻辑，约 50ms）
  +-- 扣减库存（同步）
  +-- 保存订单（同步）
  +-- 事务提交后发消息（约 5ms）
        |
        v
立即返回"下单成功"（用户只等了约 55ms）

        | 消息异步投递
        v
+-----------------------------+
|  order-created-topic:CREATED|
+----+----------+-------------+
     |          |         |
     v          v         v
  短信服务    积分服务   仓库服务
  (sms-group)(points-group)(warehouse-group)
  各自独立消费，任一服务故障不影响其他服务
```

### 案例二：订单超时自动取消（延迟消息）

```java
// 下单时发送延迟消息，30 分钟后触发超时检查
@Transactional
public String createOrder(CreateOrderRequest request) {
    String orderId = doCreateOrder(request);

    // 发送延迟消息（30分钟后投递，延迟等级 16）
    rocketMQTemplate.syncSend(
        "order-timeout-topic",
        MessageBuilder.withPayload(orderId).build(),
        3000,  // 发送超时
        16     // 延迟等级 16 = 30 分钟
    );

    return orderId;
}
```

```java
// 消费超时检查消息
@Service
@RocketMQMessageListener(
    topic = "order-timeout-topic",
    consumerGroup = "order-timeout-consumer-group"
)
public class OrderTimeoutConsumer implements RocketMQListener<String> {

    @Autowired
    private OrderService orderService;

    @Override
    public void onMessage(String orderId) {
        System.out.println("检查订单是否超时支付：" + orderId);
        // 查询订单状态，若仍未支付则关闭订单
        orderService.cancelIfNotPaid(orderId);
    }
}
```

---

## 11. 生产环境最佳实践

### 11.1 消息不丢失三重保障

```
生产者端：
  [√] 使用同步发送（syncSend），不用 sendOneway
  [√] 开启失败重试：retryTimesWhenSendFailed = 3
  [√] 消息发送失败记录到 DB，由定时任务补偿重发

Broker 端：
  [√] 关键业务开启同步刷盘：flushDiskType=SYNC_FLUSH
  [√] 主从同步复制：brokerRole=SYNC_MASTER
  [√] 至少 1 主 1 从部署，保证高可用

消费者端：
  [√] 消费成功后再提交 Offset（不要提前 ACK）
  [√] 消费失败返回 RECONSUME_LATER 触发重试
  [√] 监控死信队列 %DLQ%{ConsumerGroup}，及时人工处理
```

### 11.2 消息积压处理

```
发现消息积压时的应对方案：

方案一：临时扩容消费者实例（最快）
  将消费者实例数扩容至与 Queue 数量相同
  Queue 数量是并行消费的上限

方案二：增加 Queue 数量
  修改 Topic 的 Queue 数量（Dashboard 操作）
  同时增加消费者实例数

方案三：跳过积压（仅非关键业务）
  将 Offset 直接设置到最新位置（丢弃积压消息）
  仅适用于日志、统计等允许丢失的业务
```

### 11.3 命名规范

```
Topic：  业务名-场景-topic
         例：order-created-topic、payment-success-topic

Tag：    简短描述消息类型
         例：VIP、NORMAL、CREATE、CANCEL

ConsumerGroup：  服务名-consumer-group
                 例：sms-consumer-group、points-consumer-group

ProducerGroup：  服务名-producer-group
                 例：order-producer-group
```

### 11.4 关键监控指标

```
消息积压量（Consumer Diff Total）
  持续增长 -> 消费速度跟不上生产速度，需要扩容消费者

消费延迟（Consumer Lag）
  最新消息时间 - 最后消费消息时间，越小越好

死信队列消息数（%DLQ%{ConsumerGroup}）
  大于 0 -> 需要人工介入分析失败原因

Broker TPS
  监控生产和消费的 TPS 趋势，用于容量规划
```

---

## 12. 常见问题 FAQ

### Q1：消费者一直收不到消息怎么排查？

```
步骤一：确认 NameServer 地址配置正确
        producer/consumer 的 namesrvAddr 是否完全一致？

步骤二：确认 Topic 和 ConsumerGroup 名称拼写（区分大小写）

步骤三：确认 Tag 过滤表达式
        生产者发送的 Tag 是否被消费者的 selectorExpression 匹配？

步骤四：确认消费起始位点
        CONSUME_FROM_LAST_OFFSET 时，消费者启动前的历史消息不会收到
        改为 CONSUME_FROM_FIRST_OFFSET 可消费全部历史消息

步骤五：打开 Dashboard 检查
        Consumer 页面：该 ConsumerGroup 是否在线，Offset 是否在增长？
        Topic 页面：消息是否已写入 Broker？
```

### Q2：如何防止消息重复消费？

```java
// 关键：消费端实现幂等性

@Override
public void onMessage(OrderDTO order) {
    String key = "order:consumed:" + order.getOrderId();

    // Redis SET NX（原子操作）
    Boolean isFirst = redis.opsForValue()
        .setIfAbsent(key, "1", Duration.ofHours(24));

    if (Boolean.FALSE.equals(isFirst)) {
        // 已处理过，幂等跳过
        return;
    }

    // 首次处理业务逻辑
    doProcess(order);
}
```

### Q3：RocketMQ 与 Spring 事务如何安全配合？

```java
// 错误做法：在事务方法内直接发消息
// 风险：消息已发，但事务回滚了，消费者处理了不存在的订单
@Transactional
public void createOrder(Order order) {
    orderRepo.save(order);
    rocketMQTemplate.convertAndSend("topic", order); // 危险！
}

// 正确做法：使用 @TransactionalEventListener，确保事务提交后才发消息
@Transactional
public void createOrder(Order order) {
    orderRepo.save(order);
    applicationEventPublisher.publishEvent(new OrderCreatedEvent(order));
}

@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void onOrderCreated(OrderCreatedEvent event) {
    rocketMQTemplate.convertAndSend("topic", event.getOrder()); // 安全
}
```

### Q4：消费者抛了异常，消息会重试吗？

```java
// Spring Boot 方式（RocketMQListener）：抛出异常自动触发重试
@Override
public void onMessage(OrderDTO order) {
    if (someErrorCondition) {
        throw new RuntimeException("处理失败，触发重试");
        // 等价于返回 RECONSUME_LATER，消息按延迟等级重试（最多 16 次）
    }
    doProcess(order);
}

// 原生 API 方式：明确返回重试状态
public ConsumeConcurrentlyStatus consumeMessage(...) {
    try {
        doProcess(messages);
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    } catch (Exception e) {
        log.error("消费失败", e);
        return ConsumeConcurrentlyStatus.RECONSUME_LATER; // 明确触发重试
    }
}
```

### Q5：Topic 的 Queue 数量应该设置多少？

```
Queue 数量 = 最大并行消费实例数

建议：
  开发环境：1~4 个 Queue 即可
  生产环境：根据预期消费实例数设置（Queue 数 >= 消费者实例数）
  性能敏感：Queue 数 = CPU 核数 x 消费者实例数

注意：
  Queue 数量设置后可以增加，但不建议减少
  消费者实例数 > Queue 数时，多余实例空闲不消费
```

---

## 总结

```
RocketMQ 核心流程回顾：

1. 部署 NameServer 集群（路由注册中心）
        |
2. 部署 Broker 集群（消息存储服务器）
        |
3. 业务项目引入 rocketmq-spring-boot-starter
        |
4. 配置 name-server 和 producer.group
        |
5. 注入 RocketMQTemplate 发送消息
        |
6. 用 @RocketMQMessageListener 注解声明消费者
        |
7. 消费端做好幂等性处理，防止重复消费
```

**RocketMQ 的设计精髓：**

```
CommitLog 顺序写   -> 保障高吞吐量
ConsumeQueue 索引 -> 保障快速检索
NameServer 无状态 -> 保障高可用
ConsumerGroup 机制 -> 保障灵活消费
```

掌握这四个核心设计，就理解了 RocketMQ 的本质。

---

*文档版本：2026-06-30 | 参考 RocketMQ 4.9.x 官方文档*
