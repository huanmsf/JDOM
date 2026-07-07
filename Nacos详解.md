# Nacos 详解 - 从零到精通

> 版本：Nacos 2.x | Spring Boot 2.7.x | Spring Cloud 2021.x  
> 日期：2026-07-07

---

## 目录

- [Part 1: Nacos 整体架构](#part-1-nacos-整体架构)
- [Part 2: Nacos 注册中心深度解析](#part-2-nacos-注册中心深度解析)
- [Part 3: Nacos 配置中心深度解析](#part-3-nacos-配置中心深度解析)
- [Part 4: Nacos 集群与高可用](#part-4-nacos-集群与高可用)
- [Part 5: Spring Boot/Cloud 集成实战](#part-5-spring-bootcloud-集成实战)
- [Part 6: Nacos 与 OpenFeign 集成](#part-6-nacos-与-openfeign-集成)
- [Part 7: Nacos 与 LoadBalancer 集成](#part-7-nacos-与-loadbalancer-集成)
- [Part 8: Nacos 运维管理](#part-8-nacos-运维管理)
- [Part 9: 完整实战案例](#part-9-完整实战案例)
- [Part 10: 常见面试题 FAQ](#part-10-常见面试题-faq)

---

# Part 1: Nacos 整体架构

## 1.1 Nacos 是什么

Nacos（**Na**ming and **Co**nfiguration **S**ervice）是阿里巴巴开源的一款**动态服务发现、配置管理和服务管理**平台，于 2018 年正式开源。它是阿里巴巴内部 ConfigServer 和 VIPServer 两大核心中间件经过多年生产环境打磨后对外开放的产物。

Nacos 的核心定位是：**一个更易于构建云原生应用的动态服务发现、配置管理和服务管理平台**。

一句话概括：**Nacos = 注册中心 + 配置中心 + 服务管理**，三合一的微服务基础设施。

### Nacos 发展历程

```text
2008年  阿里巴巴内部 ConfigServer 上线（配置中心）
2009年  阿里巴巴内部 VIPServer 上线（服务注册发现）
2018年  Nacos 0.1.0 正式开源（GitHub）
2019年  Nacos 1.0.0 发布，生产可用
2020年  Nacos 1.3.x 支持集群 Raft 协议
2021年  Nacos 2.0.0 发布，全面升级 gRPC 长连接
2022年  Nacos 2.1.x 稳定版，Kubernetes 生态集成
2023年  Nacos 2.3.x，支持更多云原生特性
```

### Nacos 核心能力

| 能力 | 说明 |
|------|------|
| **服务注册与发现** | 服务实例向 Nacos 注册，消费者从 Nacos 查询可用实例列表 |
| **动态配置管理** | 集中管理各环境配置，支持热更新，无需重启服务 |
| **服务健康检查** | 主动探测（持久实例）和心跳检测（临时实例）两种机制 |
| **动态 DNS** | 基于权重的 DNS 服务，支持流量管控 |
| **服务元数据管理** | 路由策略、灰度策略、安全策略等元数据 |

---

## 1.2 Nacos vs 其他注册中心对比

```text
+------------------+--------------+--------------+--------------+--------------+
|     特性          |    Nacos     |    Eureka    |    Consul    |  ZooKeeper   |
+------------------+--------------+--------------+--------------+--------------+
| CAP 理论          | AP + CP 可切 |     AP       |     CP       |     CP       |
| 一致性协议        | Raft+Distro  | 自定义复制   |    Raft      |    ZAB       |
| 健康检查          | TCP/HTTP/心跳|    心跳      | TCP/HTTP/脚本|  Keep-Alive  |
| 负载均衡          | 权重/集群优先|    Ribbon    |  Fabio/自带  |     无       |
| 雪崩保护          |      是      |     是       |     否       |     否       |
| 自动注销实例      |      是      |     是       |     是       |     是       |
| 访问协议          |  HTTP/gRPC   |    HTTP      |   HTTP/DNS   |    TCP       |
| 配置中心          |     内置     |     否       |     内置     |     可扩展   |
| 多数据中心        |     是       |     是       |     是       |     否       |
| SpringCloud集成   |   原生支持   |   原生支持   |   原生支持   |   需扩展     |
| Dubbo 集成        |   原生支持   |     否       |     否       |   原生支持   |
| K8s 集成          |     是       |     否       |     是       |     否       |
| 界面管理          |   功能完善   |   基础界面   |   功能完善   |  第三方工具  |
| 社区活跃度        |     高       |   停止维护   |     高       |     高       |
| 数据持久化        | Derby/MySQL  |    内存      |   内置K/V    |  文件/内存   |
| 语言实现          |    Java      |    Java      |     Go       |    Java      |
+------------------+--------------+--------------+--------------+--------------+
```

### 选型建议

- **Nacos**：Spring Cloud Alibaba 生态首选，同时满足注册中心 + 配置中心需求，推荐
- **Consul**：Go 语言技术栈或需要 Service Mesh 场景
- **Eureka**：Netflix 老项目维护，2.x 停止维护，不推荐新项目使用
- **ZooKeeper**：Dubbo 遗留项目，或需要强一致性分布式协调场景

---

## 1.3 Nacos 整体架构 ASCII 图

```text
                     +-----------------------------------------------+
                     |            用户 / 运维人员                      |
                     +--------------------+--------------------------+
                                          |  控制台访问 (8848)
                                          v
+-----------------------------------------------------------------------------------+
|                            Nacos Server Cluster                                    |
|                                                                                    |
|  +-----------------+      +-----------------+      +-----------------+            |
|  |  Nacos Node 1   |      |  Nacos Node 2   |      |  Nacos Node 3   |            |
|  |  (Leader)       +<---->+  (Follower)     +<---->+  (Follower)     |            |
|  |                 |      |                 |      |                 |            |
|  | +-------------+ |      | +-------------+ |      | +-------------+ |            |
|  | |NamingService| |      | |NamingService| |      | |NamingService| |            |
|  | |ConfigService| |      | |ConfigService| |      | |ConfigService| |            |
|  | +-------------+ |      | +-------------+ |      | +-------------+ |            |
|  +---------+-------+      +---------+-------+      +---------+-------+            |
|            |                        |                        |                    |
|            +------------------------+------------------------+                    |
|                             Raft / Distro 协议同步                                 |
|                                       |                                            |
|                      +----------------+----------------+                          |
|                      |        持久化存储                 |                          |
|                      |    MySQL / Derby (内嵌)          |                          |
|                      +---------------------------------+                          |
+-----------------------------------------------------------------------------------+
        ^                          ^                          ^
        |  HTTP/gRPC               |  HTTP/gRPC               |  HTTP/gRPC
        |  服务注册/发现            |  配置拉取/订阅            |  健康心跳
        |                          |                          |
+-------+----------+  +------------+----------+  +-----------+-----------+
|   微服务 A        |  |   微服务 B             |  |   微服务 C             |
| (订单服务)        |  |  (商品服务)            |  |  (用户服务)            |
| Nacos Client SDK |  |  Nacos Client SDK    |  | Nacos Client SDK      |
| +------------+   |  |  +------------+      |  | +------------+        |
| |本地服务缓存 |   |  |  |本地服务缓存 |      |  | |本地服务缓存 |        |
| |本地配置缓存 |   |  |  |本地配置缓存 |      |  | |本地配置缓存 |        |
| +------------+   |  |  +------------+      |  | +------------+        |
+------------------+  +----------------------+  +-----------------------+
```

---

## 1.4 Nacos 核心功能详解

### 1.4.1 服务注册发现流程

```text
服务提供者启动
       |
       v
向 Nacos 注册 (ip, port, 服务名, 权重, 元数据...)
       |
       v
Nacos 存储实例信息 (内存 + 可选持久化)
       |
       v
服务消费者查询 (根据服务名查询实例列表)
       |
       v
客户端缓存实例列表 (~/.nacos/naming/...)
       |
       v
负载均衡选择实例
       |
       v
发起 RPC/HTTP 调用
```

### 1.4.2 动态配置管理

配置管理解决了微服务中配置文件分散、修改繁琐、需要重启的痛点：

- **集中管理**：所有服务的配置存储在 Nacos，统一查看和修改
- **动态推送**：配置变更后，订阅了该配置的服务实例会在毫秒级收到通知
- **灰度发布**：可以指定配置只推送给部分实例，实现灰度测试
- **版本历史**：每次修改都有记录，支持一键回滚到任意历史版本

### 1.4.3 服务健康检查

Nacos 支持两种健康检查模式：

**临时实例**（默认模式）：
- 客户端主动向 Nacos 发送心跳，默认心跳间隔：5 秒
- 15 秒未收到心跳：标记为不健康（从服务列表中隐藏）
- 30 秒未收到心跳：自动删除该实例，通知订阅方

**持久实例**：
- Nacos 服务端主动探测客户端（TCP/HTTP/MySQL 探测）
- 实例不健康时不删除，只标记状态，便于运维排查

---

## 1.5 Nacos 数据模型

Nacos 使用层级数据模型来组织服务和配置数据：

```text
Nacos 数据模型层级结构
=====================================================

Namespace (命名空间)
|-- 作用：最高层级隔离，通常用于环境隔离
|-- 默认值：public (ID 为空字符串)
|-- 示例：dev / test / staging / prod
|
+-- Group (分组)
    |-- 作用：同一命名空间内的逻辑分组
    |-- 默认值：DEFAULT_GROUP
    |-- 示例：ORDER_GROUP / PAYMENT_GROUP
    |
    +-- Service (服务) [注册中心使用]
    |   |-- 服务名：com.example.order-service
    |   |-- 保护阈值：0.0 - 1.0
    |   |
    |   +-- Cluster (集群)
    |       |-- 作用：同一服务内的实例分组，通常按机房/可用区划分
    |       |-- 默认值：DEFAULT
    |       |-- 示例：BJ-CLUSTER / SH-CLUSTER
    |       |
    |       +-- Instance (实例)
    |           |-- IP: 192.168.1.100
    |           |-- Port: 8080
    |           |-- 权重: 1.0
    |           |-- 健康状态: true/false
    |           |-- 临时实例: true/false
    |           +-- 元数据: {"version": "v1", "env": "prod"}
    |
    +-- DataId (配置ID) [配置中心使用]
        |-- 作用：配置的唯一标识（通常为文件名）
        |-- 示例：order-service.yml / order-service-dev.yml
        |-- 内容：YAML/Properties/JSON/XML/TEXT
        +-- 版本：每次修改自动记录历史
```

### 数据模型应用实例（电商平台）

```text
Nacos
+-- Namespace: dev (开发环境)
|   +-- Group: ORDER_GROUP
|   |   +-- Service: order-service
|   |   +-- DataId: order-service.yml
|   +-- Group: PAYMENT_GROUP
|       +-- Service: payment-service
|       +-- DataId: payment-service.yml
|
+-- Namespace: test (测试环境)
|   +-- Group: DEFAULT_GROUP
|       +-- Service: order-service (指向测试实例)
|
+-- Namespace: prod (生产环境)
    +-- Group: DEFAULT_GROUP
        +-- Service: order-service (指向生产实例)
```

---

# Part 2: Nacos 注册中心深度解析

## 2.1 服务注册原理

### 2.1.1 注册流程总览

```text
Spring Boot 应用启动
         |
         v
Spring Context 初始化完成
         |
         v  ApplicationStartedEvent / WebServerInitializedEvent
         |
         v
NacosAutoServiceRegistration.start()
         |
         v
NacosRegistration 构建注册信息
(serviceName, ip, port, metadata, weight...)
         |
         v
NacosServiceRegistry.register()
         |
         v
NamingService.registerInstance()
         |
     +---+-------------------------+
     |                             |
  临时实例                       持久实例
(ephemeral=true)             (ephemeral=false)
     |                             |
     v                             v
gRPC 长连接注册               HTTP POST 注册
(Nacos 2.x)               /nacos/v1/ns/instance
     |                             |
     v                             v
服务端存储到内存              服务端写入 MySQL
DistroMap                    Raft 日志复制
     |                             |
     +-------------+---------------+
                   |
                   v
          注册成功，开始心跳
```

### 2.1.2 Nacos 1.x vs 2.x 注册协议对比

**Nacos 1.x（HTTP 短连接模式）**：

```text
客户端                            Nacos Server
  |                                    |
  |-- POST /nacos/v1/ns/instance ----->|  注册（HTTP）
  |                                    |
  |-- PUT  /nacos/v1/ns/instance/beat->|  心跳（每5秒一次独立 HTTP 请求）
  |                                    |
  |<-- UDP 推送服务变更通知 ------------|  服务变更推送（UDP，不可靠）
  |                                    |
  |-- GET  /nacos/v1/ns/instance/list->|  查询服务列表
```

**Nacos 2.x（gRPC 长连接模式）**：

```text
客户端                            Nacos Server
  |                                    |
  |==== gRPC 长连接建立 =============>|  一次连接复用
  |                                    |
  |-- InstanceRequest (注册) --------->|  注册（gRPC）
  |<-- InstanceResponse ---------------| 
  |                                    |
  |-- HealthCheckRequest (心跳) ------>|  心跳复用长连接，无 TCP 握手开销
  |<-- HealthCheckResponse ------------|
  |                                    |
  |<-- NotifySubscriberRequest --------|  服务变更推送（可靠的 gRPC）
  |-- NotifySubscriberResponse ------->|
  |                                    |
  |-- SubscribeServiceRequest -------->|  订阅服务
  |<-- SubscribeServiceResponse -------|
```

**Nacos 2.x 性能提升对比**：

| 指标 | Nacos 1.x | Nacos 2.x | 提升幅度 |
|------|-----------|-----------|----------|
| 单节点最大实例 | ~20万 | ~100万 | 5倍 |
| 心跳CPU消耗 | 高（每次建立TCP连接） | 低（复用gRPC） | -50% |
| 推送可靠性 | UDP（可能丢包） | gRPC（可靠） | 显著提升 |
| 推送延迟 | ~1-5s（重试） | ~100-200ms | 10-25倍 |

---

## 2.2 临时实例 vs 持久实例

```text
+------------------+----------------------+------------------------------+
|     特性          |      临时实例         |        持久实例               |
|                  |  (ephemeral=true)    |    (ephemeral=false)         |
+------------------+----------------------+------------------------------+
| 默认值            |        是             |          否                   |
| 健康检查方式      | 客户端主动上报心跳     | 服务端主动探测                |
| 心跳间隔          |       5秒             |          N/A                 |
| 不健康阈值        |      15秒无心跳       | 探测失败                     |
| 删除条件          |      30秒无心跳       | 手动删除，不自动删除           |
| 存储位置          |   内存（DistroMap）   |   MySQL（持久化）              |
| 一致性协议        | Distro（AP，最终一致） |  Raft（CP，强一致）            |
| 适用场景          | 容器/虚拟机微服务      | 物理机/固定IP服务              |
| 实例下线处理      | 自动清除              | 保留记录，标记不健康            |
| 优点              | 自动感知服务上下线     | 实例信息不丢失，便于运维排查    |
| 缺点              | 数据不持久            | 需要手动管理实例生命周期        |
+------------------+----------------------+------------------------------+
```

### 配置临时/持久实例示例

```yaml
# application.yml
spring:
  cloud:
    nacos:
      discovery:
        # 设置为持久实例（默认为 true 即临时实例）
        ephemeral: false
        server-addr: 127.0.0.1:8848
```

---

## 2.3 健康检查机制深度解析

### 2.3.1 临时实例心跳机制

```text
时间轴（临时实例心跳机制）：

t=0s    服务启动，向 Nacos 注册，状态: HEALTHY
        |
t=5s    发送心跳 ---------------------------------------->  Nacos 收到，重置计时
        |
t=10s   发送心跳 ---------------------------------------->  Nacos 收到，重置计时
        |
t=15s   发送心跳 ---------------------------------------->  Nacos 收到，重置计时
        |
        | (假设此时服务宕机，心跳停止)
        |
t=20s   [无心跳]                                            Nacos 计时: 5s
t=25s   [无心跳]                                            Nacos 计时: 10s
t=30s   [无心跳] ---------------------------------------->  Nacos 计时: 15s
                                                            ★ 标记为 UNHEALTHY
                                                            ★ 从服务列表中隐藏
t=35s   [无心跳]                                            Nacos 计时: 20s
t=40s   [无心跳]                                            Nacos 计时: 25s
t=45s   [无心跳] ---------------------------------------->  Nacos 计时: 30s
                                                            ★★ 删除该实例 ★★
                                                            ★ 通知订阅方服务列表变更
```

**心跳相关配置参数**：

```yaml
spring:
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        # 心跳间隔（毫秒），默认5000
        heart-beat-interval: 5000
        # 心跳超时时间（毫秒），默认15000，超过此时间标记为不健康
        heart-beat-timeout: 15000
        # 实例删除超时（毫秒），默认30000，超过此时间删除实例
        ip-delete-timeout: 30000
```

### 2.3.2 持久实例主动探测机制

**TCP 端口探测**：

```text
Nacos Server                         持久实例
     |                                   |
     |---- TCP Connect(ip:port) -------->|
     |                                   |
     |<---- TCP ACK --------------------|  连接成功 -> 健康
     |                                   |
     |  或者（服务宕机时）                 |
     |                                   |
     |---- TCP Connect(ip:port) -------->|  连接被拒 -> 不健康
     |<---- Connection Refused ----------|
```

**HTTP 接口探测**（在 Nacos 控制台配置）：

```text
探测路径: /actuator/health
预期状态码: 200
超时时间: 3000ms
探测间隔: 5000ms
连续失败次数: 3次后标记不健康
```

---

## 2.4 AP vs CP 模式深度解析

### 2.4.1 CAP 理论回顾

```text
CAP 理论：分布式系统只能同时满足以下三点中的两点

        C (Consistency)
        一致性：所有节点看到的数据一致
             /\
            /  \
           /    \
          /      \
         /  CAP   \
        /____________\
A (Availability)    P (Partition Tolerance)
可用性：服务始终可用  分区容忍性：网络分区时系统继续工作

CP: 保证一致性 + 分区容忍，牺牲可用性 (如 ZooKeeper, Raft)
AP: 保证可用性 + 分区容忍，牺牲一致性 (如 Eureka, Distro)
```

### 2.4.2 Nacos 的 AP + CP 双模式

```text
Nacos 一致性模式
=====================================================

临时实例 (ephemeral=true)
     |
     +-- 使用 Distro 协议 (AP)
         |-- 数据存储在内存
         |-- 各节点平等，没有 Leader
         |-- 数据异步同步到其他节点
         |-- 网络分区时，各节点继续提供服务
         +-- 最终一致性（可能短暂读到旧数据）

持久实例 (ephemeral=false)
     |
     +-- 使用 Raft 协议 (CP)
         |-- 数据持久化到数据库
         |-- 有 Leader 节点，写操作通过 Leader
         |-- 超半数节点确认后才写入成功
         |-- 网络分区时，少数节点停止写服务
         +-- 强一致性（读写一定是最新数据）
```

**为什么临时实例选 AP？**

在微服务场景下，服务注册发现要求**高可用性**比**强一致性**更重要：
- 即使 Nacos 集群发生网络分区，服务仍然能注册和发现
- 短暂的数据不一致（某实例在另一个节点短暂可见）影响不大
- 系统恢复后，数据最终会一致

**为什么持久实例选 CP？**

持久实例通常是重要的基础设施服务，需要：
- 保证实例信息的准确性，不能出现幻影实例
- 配置信息需要强一致性，不能丢失
- 宁可短暂不可用，也要保证数据准确

---

## 2.5 Distro 协议详解

Distro 是 Nacos 自研的 AP 分布式一致性协议，专为临时实例的注册发现场景设计。

### 2.5.1 Distro 核心设计思想

```text
Distro 协议核心思想：数据分片 + 负责人机制

传统 AP 协议问题：所有节点都接受写，然后广播同步，容易冲突

Distro 解决方案：
+-----------------------------------------------------+
|  每个节点负责一部分数据（按客户端 IP hash 分片）       |
|  只有"负责"该数据的节点才能写这部分数据               |
|  其他节点转发写请求给负责节点                         |
+-----------------------------------------------------+

示例（3节点集群，6个客户端）：

客户端A (hash%3=0) --> Node1 负责
客户端B (hash%3=1) --> Node2 负责
客户端C (hash%3=2) --> Node3 负责
客户端D (hash%3=0) --> Node1 负责
客户端E (hash%3=1) --> Node2 负责
客户端F (hash%3=2) --> Node3 负责

Node1                Node2               Node3
  | 负责A,D的数据     | 负责B,E的数据      | 负责C,F的数据
  |                  |                   |
  |<--- 定期同步 ---->|<--- 定期同步 ----->|
        (全量+增量)         (全量+增量)
```

### 2.5.2 Distro 数据同步流程

```text
Distro 数据写入流程：

Client 向 Node2 注册 ServiceA
         |
         v
Node2 是否负责该 Client？
  |
  +-- 是 -> 直接写入本节点内存
  |         异步同步给 Node1, Node3（延迟 < 100ms）
  |
  +-- 否 -> 转发请求给负责的节点
            等待负责节点写入成功
            返回成功给 Client

Distro 数据同步两种方式：
+-------------------------------------------------+
| 全量同步（启动时）：                              |
|   新节点加入集群时，从其他节点拉取全量数据         |
|   确保加入时数据完整                              |
+-------------------------------------------------+
| 增量同步（运行时）：                              |
|   数据变更时，异步推送变更给其他节点              |
|   延迟通常 < 100ms                               |
+-------------------------------------------------+
```

### 2.5.3 Distro vs Raft 对比

```text
+--------------------+--------------------+--------------------+
|      特性           |       Distro       |       Raft         |
+--------------------+--------------------+--------------------+
| CAP 倾向            |       AP           |       CP           |
| 写入节点            | 任意节点（转发机制） | 只有 Leader        |
| 读取节点            | 任意节点（本地读）  | Leader 或 Follower |
| 数据存储            |      内存          |   持久化（磁盘）     |
| 选举机制            |      无            |  有（Leader 选举）  |
| 网络分区            | 各分区继续服务       | 少数分区停写        |
| 一致性              |    最终一致        |     强一致          |
| 写入延迟            |      低            |   较高（需多数确认） |
| 适用场景            | 服务注册发现        | 配置管理/持久实例   |
+--------------------+--------------------+--------------------+
```

---

## 2.6 服务发现机制：客户端缓存 + 推送更新

### 2.6.1 服务发现流程

```text
客户端首次发现服务：

ServiceConsumer                      Nacos Server
      |                                    |
      |-- 订阅 order-service ------------->|
      |<-- 返回当前实例列表 ---------------|
      |                                    |
      |  本地缓存实例列表                   |
      |  ~/.nacos/naming/[namespace]/      |
      |  [group]@@[service].json           |

服务实例变更推送（Nacos 1.x）：

Nacos Server                    ServiceConsumer
      |                                    |
      | 检测到 order-service 实例变更       |
      |-- UDP 推送变更通知 --------------->|  不可靠，可能丢包
      |<-- UDP ACK (可能丢失) -------------|
      |    未收到 ACK 则重发               |

服务实例变更推送（Nacos 2.x）：

Nacos Server                    ServiceConsumer
      |                                    |
      | 检测到 order-service 实例变更       |
      |-- gRPC NotifySubscriberRequest --->|  可靠推送
      |<-- gRPC Response ------------------|
```

### 2.6.2 本地缓存文件结构

```text
~/.nacos/naming/
+-- [namespace-id]/
    +-- [group]@@[service-name].json    # 服务实例缓存
    +-- failover/                       # 故障转移目录
        +-- [group]@@[service-name]     # 故障转移文件
```

缓存文件内容示例（JSON格式）：

```json
{
    "dom": "order-service",
    "cacheMillis": 10000,
    "hosts": [
        {
            "ip": "192.168.1.100",
            "port": 8080,
            "weight": 1.0,
            "healthy": true,
            "enabled": true,
            "ephemeral": true,
            "clusterName": "DEFAULT",
            "serviceName": "DEFAULT_GROUP@@order-service",
            "metadata": {
                "preserved.register.source": "SPRING_CLOUD"
            }
        }
    ],
    "name": "DEFAULT_GROUP@@order-service",
    "checksum": "abc123",
    "lastRefTime": 1688111111111
}
```

### 2.6.3 故障转移（Failover）机制

当 Nacos Server 完全不可用时（如网络故障），Nacos Client 会：

1. **使用内存缓存**：继续使用最后一次从服务端获取的实例列表
2. **使用磁盘缓存**：内存缓存失效后，从本地磁盘缓存文件读取
3. **故障转移文件**：支持手动创建故障转移文件，指定固定的实例列表

```text
故障转移文件路径：
~/.nacos/naming/[namespace-id]/failover/DEFAULT_GROUP@@order-service

文件内容与正常缓存文件格式相同
手动维护该文件后，开启故障转移模式，客户端将从该文件读取实例列表
```

---

## 2.7 服务列表负载均衡

### 2.7.1 权重负载均衡

```text
权重负载均衡示例：

实例列表：
  实例A (ip:8081, 权重=1)
  实例B (ip:8082, 权重=2)
  实例C (ip:8083, 权重=3)

总权重 = 1 + 2 + 3 = 6

请求分配概率：
  实例A: 1/6 ≈ 16.7%
  实例B: 2/6 ≈ 33.3%
  实例C: 3/6 ≈ 50.0%

应用场景：
  - 服务器性能不均等时，给高配置服务器设置更高权重
  - 灰度发布时，新版本实例设置较低权重（如 0.1）
  - 服务器维护前，先将权重设为 0（优雅下线）
```

### 2.7.2 集群优先策略

在多机房/多区域部署时，优先调用同集群的实例，避免跨机房调用延迟：

```text
北京机房                            上海机房
---------------------------         ---------------------------
order-service                       order-service
  实例A (Cluster=BJ)                  实例C (Cluster=SH)
  实例B (Cluster=BJ)                  实例D (Cluster=SH)

product-service (BJ 消费者，Cluster=BJ)
     |
     +-- 优先选择: 实例A, 实例B (同集群 BJ)，延迟: ~1ms
     |
     +-- 兜底选择: 实例C, 实例D (跨集群 SH)，延迟: ~50ms
         触发条件: BJ 集群无健康实例时才使用
```

---

# Part 3: Nacos 配置中心深度解析

## 3.1 配置管理核心概念

### 3.1.1 三元组定位配置

Nacos 配置中心使用 **Namespace + Group + DataId** 三元组来唯一定位一个配置：

```text
+----------------------------------------------------------+
|                    配置三元组                              |
|                                                          |
|  Namespace: dev-env-id (或 public)                       |
|      |                                                   |
|      +-- Group: DEFAULT_GROUP (或 ORDER_GROUP)           |
|              |                                           |
|              +-- DataId: order-service.yml               |
|                  |-- 类型: YAML                          |
|                  +-- 内容: spring.datasource.url=...     |
+----------------------------------------------------------+
```

**DataId 命名规范**：在 Spring Cloud Alibaba 中，DataId 的默认生成规则为：

```text
格式：-.

例如：
  应用名: order-service
  环境:   dev
  格式:   yaml
  DataId: order-service-dev.yaml
```

完整的配置加载优先级（由高到低）：

```text
order-service-dev.yaml          <- 最高优先级（应用名 + 环境 + 格式）
order-service-dev.properties
order-service.yaml
order-service.properties        <- 应用名 + 格式
application-dev.yaml            <- 共享配置（带环境）
application.yaml                <- 共享配置（最低优先级）
```

---

## 3.2 配置推送原理：长轮询机制详解

Nacos 配置中心采用**长轮询（Long Polling）**机制来实现配置变更的近实时推送。

### 3.2.1 三种推送方式对比

```text
普通轮询（Polling）：
客户端每隔固定时间（如1秒）主动查询
优点：简单     缺点：延迟高，大量无效请求（1000个客户端就是1000 QPS）

Client --GET /config------------------------->Server
Client <--200 (无变化)-------------------------Server  [1s后]
Client --GET /config------------------------->Server
...（大量无效请求）

长轮询（Long Polling）—— Nacos 配置中心使用：
Client 发送请求，Server 挂起请求等待变更（最多30s）
有变更立即返回，否则29.5s后返回空响应

Client --POST /config/listener-------------->Server
                                             |
                                             | （挂起，等待变更...）
                                             |
     -- 配置发生变更 --                      |
     <-- 200 (返回变更的 DataId) -----------  | （立即返回）
Client --GET /config (获取新值)------------>Server
Client <-- 200 (新配置内容) ----------------Server

或者（无变更情况）：
     <-- 304 (无变化，空响应) -------------- | （29.5s后）
Client --POST /config/listener-------------->Server  （立即再次发起）
```

### 3.2.2 长轮询详细流程

```text
Nacos 配置长轮询完整流程：

初始化阶段：
+-----------------------------------------------------------+
|  1. 应用启动，读取 bootstrap.yml 中的 nacos 配置信息       |
|  2. 从 Nacos 拉取全量配置（同步请求）                      |
|  3. 计算每个配置的 MD5 值并缓存                            |
|  4. 启动长轮询线程（ClientWorker.LongPollingRunnable）     |
+-----------------------------------------------------------+

长轮询循环：
+-------------------------------------------------------------+
|                                                             |
|  步骤1: 构建请求体（包含所有监听配置的 dataId+group+MD5）   |
|                                                             |
|  步骤2: POST /nacos/v1/cs/configs/listener                  |
|         请求头: Long-Pulling-Timeout: 30000 (30秒)          |
|         请求体: dataId^group^md5^tenant\r\n...              |
|                                                             |
|  服务端处理：                                               |
|  +-----------------------------------------------------+   |
|  | 比较客户端 MD5 vs 服务端当前 MD5                      |   |
|  |                                                     |   |
|  | 有变更？                                             |   |
|  |  是 -> 立即返回变更的 dataId 列表（不返回配置内容）    |   |
|  |  否 -> 挂起请求，注册 hold                            |   |
|  |        等待 29.5s                                    |   |
|  |        若期间有变更，立即响应                          |   |
|  |        若 29.5s 内无变更，返回空响应                   |   |
|  +-----------------------------------------------------+   |
|                                                             |
|  步骤3: 收到变更的 dataId 列表                              |
|                                                             |
|  步骤4: 针对每个变更的 dataId，                             |
|         GET /nacos/v1/cs/configs?dataId=xxx&group=xxx       |
|         获取最新配置内容                                    |
|                                                             |
|  步骤5: 触发 Listener.receiveConfigInfo() 回调             |
|         通知应用配置已更新                                  |
|                                                             |
|  步骤6: 回到步骤1，开始下一轮长轮询                         |
|                                                             |
+-------------------------------------------------------------+
```

### 3.2.3 为什么是 29.5 秒而不是 30 秒？

这是一个细节设计：
- 客户端设置 Long-Pulling-Timeout: 30000（毫秒）
- 服务端实际等待 **29.5 秒**（30000 - 500ms 的预留时间）
- 这 500ms 的差值是为了保证服务端**先于**客户端超时返回响应
- 避免客户端超时（如 HttpClient 的 30s 超时）先触发，导致连接被强制关闭

```text
时间轴：
0s        服务端开始 hold 请求
...
29.5s     服务端主动返回空响应（无变更）
30s       客户端 HttpClient 超时点（但此时已经收到响应，不会超时）

如果服务端也等 30s：
0s        服务端开始 hold 请求
30s       服务端超时 vs 客户端超时 -> 竞争条件，可能出现异常
```

---

## 3.3 配置变更通知：MD5 比对机制

### 3.3.1 MD5 比对原理

```text
配置 MD5 比对流程：

Nacos Server 收到长轮询请求：
  请求体: order-service.yml^DEFAULT_GROUP^abc123^dev-ns\r\n
           (dataId)           (group)        (MD5)  (tenant)

Server 逻辑：
  1. 查询 order-service.yml 当前内容的 MD5 = ?

  情况A: 当前 MD5 = abc123（与客户端相同）
     -> 配置未变更，继续 hold 请求

  情况B: 当前 MD5 = xyz789（与客户端不同）
     -> 配置已变更！立即返回:
       "order-service.yml%02DEFAULT_GROUP%02dev-ns%01"

说明：
  %02 是 GROUP_KEY_DELIMITER，表示字段分隔符
  %01 是 LINE_SEPARATOR，表示多个配置之间的分隔符
  
  客户端解析后得到变更的配置三元组，
  再去服务端拉取最新内容
```

### 3.3.2 配置变更监听代码实战（原生 SDK）

```java
import com.alibaba.nacos.api.NacosFactory;
import com.alibaba.nacos.api.config.ConfigService;
import com.alibaba.nacos.api.config.listener.Listener;
import java.util.Properties;
import java.util.concurrent.Executor;

/**
 * Nacos 配置变更监听示例（原生 SDK，不依赖 Spring Boot）
 */
public class NacosConfigListenerExample {

    public static void main(String[] args) throws Exception {
        // 配置 Nacos 连接属性
        Properties properties = new Properties();
        properties.put("serverAddr", "127.0.0.1:8848");
        properties.put("namespace", "dev-namespace-id");

        // 创建 ConfigService（底层维护长轮询线程）
        ConfigService configService = NacosFactory.createConfigService(properties);

        String dataId = "order-service.yml";
        String group = "DEFAULT_GROUP";

        // 首次获取配置（超时5000ms）
        String config = configService.getConfig(dataId, group, 5000);
        System.out.println("初始配置: " + config);

        // 添加配置变更监听器
        configService.addListener(dataId, group, new Listener() {

            @Override
            public Executor getExecutor() {
                // 返回 null 表示使用默认线程池
                // 也可以返回自定义线程池来异步处理配置更新（避免阻塞长轮询线程）
                return null;
            }

            @Override
            public void receiveConfigInfo(String configInfo) {
                // 配置变更时触发此回调
                System.out.println("=== 配置已更新 ===");
                System.out.println("新配置内容: " + configInfo);
                // 在这里处理配置更新逻辑
                // 例如：更新数据源连接池参数、刷新缓存等
            }
        });

        System.out.println("监听中，等待配置变更...");
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

---

## 3.4 配置快照（本地缓存）：断网容灾

### 3.4.1 快照存储路径

```text
配置快照路径（默认）：
~/.nacos/config/
+-- [server-addr]/
    +-- [namespace-id]/
        +-- [group]/
            +-- [dataId]        <- 配置内容文件

示例：
~/.nacos/config/
+-- 127.0.0.1_8848/
    +-- dev-ns-id/
        +-- DEFAULT_GROUP/
            +-- order-service.yml
            +-- payment-service.yml
            +-- common.properties
```

### 3.4.2 配置加载优先级（含快照）

```text
配置加载优先级（从高到低）：

1. 远程 Nacos Server 配置（在线状态）
   |
2. 本地快照文件（~/.nacos/config/...）
   |  当 Nacos Server 不可用时自动使用
   |
3. 本地配置文件（application.yml/bootstrap.yml）
   |  完全离线时的兜底配置

代码中的体现：
NacosConfigService.getConfigInner()
  |
  +-- try: 从 Server 获取
  |    |
  |    +-- 成功 -> 写入快照文件 -> 返回
  |    |
  |    +-- 失败（网络异常）-> 读取快照文件返回
  |
  +-- 快照也不存在 -> 返回 null（使用 @Value 默认值）
```

---

## 3.5 配置隔离策略

### 3.5.1 Namespace 环境隔离

```text
Namespace 隔离（不同环境完全隔离）：

+-----------------------------------------------------+
| Namespace: public (默认)                            |
|   DataId: common.properties                        |
+-----------------------------------------------------+
| Namespace: dev (开发环境，ID=dev-xxxxx)             |
|   DataId: order-service.yml  (内容: DB指向开发库)   |
|   DataId: payment-service.yml                      |
+-----------------------------------------------------+
| Namespace: test (测试环境，ID=test-xxxxx)           |
|   DataId: order-service.yml  (内容: DB指向测试库)   |
|   DataId: payment-service.yml                      |
+-----------------------------------------------------+
| Namespace: prod (生产环境，ID=prod-xxxxx)           |
|   DataId: order-service.yml  (内容: DB指向生产库)   |
|   DataId: payment-service.yml                      |
+-----------------------------------------------------+

特点：不同 Namespace 之间数据完全隔离，互不可见
使用场景：区分开发/测试/生产等不同环境
```

### 3.5.2 Group 业务隔离

```text
Group 隔离（同一命名空间内，按业务分组）：

Namespace: prod
+-- Group: ORDER_GROUP
|   +-- DataId: order-service.yml
|   +-- DataId: order-common.properties
+-- Group: PAYMENT_GROUP
|   +-- DataId: payment-service.yml
|   +-- DataId: payment-common.properties
+-- Group: DEFAULT_GROUP
    +-- DataId: application.yml  (全局共享配置)

使用场景：同一环境内，不同业务线/团队管理各自的配置
```

---

## 3.6 配置格式支持

Nacos 支持以下配置格式：

```text
+--------------+------------------------------------------------+
|   格式        |              使用场景                          |
+--------------+------------------------------------------------+
| PROPERTIES   | Spring 传统配置格式，key=value                  |
| YAML         | Spring Boot 推荐格式，层级结构清晰              |
| JSON         | 程序化读取配置，如前端配置、规则配置             |
| XML          | 遗留系统，如 MyBatis mapper XML、Log4j 配置     |
| TEXT         | 纯文本，如 SQL 脚本、脚本模板                   |
| HTML         | Web 内容模板                                    |
+--------------+------------------------------------------------+
```

**YAML 格式配置示例（Nacos 中 DataId=order-service.yml 的内容）**：

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/order_db
    username: order_user
    password: 
    driver-class-name: com.mysql.cj.jdbc.Driver
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000

order:
  timeout: 5000
  retry-count: 3
  enable-cache: true
  payment-methods:
    - ALIPAY
    - WECHAT_PAY
    - BANK_CARD
```

---

## 3.7 灰度发布（Beta 发布）

### 3.7.1 什么是配置灰度发布

配置灰度发布允许将新配置只推送给**指定的部分实例**，用于：
- 验证新配置的正确性（先在2个实例上验证）
- 逐步推进高风险配置变更（分批推送）
- A/B 测试不同的配置效果

```text
灰度发布示意图：

配置: order-service.yml
+-----------------------------------------------------+
| 当前线上版本: v1 (10个实例都在用)                    |
|                                                     |
| 灰度版本: v2                                        |
|   目标IP: 192.168.1.10, 192.168.1.11 (2个实例)     |
+-----------------------------------------------------+

实例列表：
  192.168.1.10  <- 收到 v2 配置（灰度实例）
  192.168.1.11  <- 收到 v2 配置（灰度实例）
  192.168.1.12  <- 继续使用 v1 配置
  192.168.1.13  <- 继续使用 v1 配置
  ...（其他8个实例继续用 v1）

观察 2 个实例的表现
  |
  +-- 无问题 -> 逐步扩大灰度范围 -> 全量发布
  |
  +-- 有问题 -> 快速回滚，只影响 2 个实例
```

### 3.7.2 灰度发布 API

```bash
# 使用 Nacos Open API 发布灰度配置
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service.yml' \
  -d 'group=DEFAULT_GROUP' \
  -d 'tenant=prod-ns-id' \
  -d 'type=yaml' \
  -d 'content=order.timeout=8000' \
  -d 'betaIps=192.168.1.10,192.168.1.11'
  # betaIps 指定灰度的 IP 列表（逗号分隔）
```

---

## 3.8 历史版本回滚

Nacos 自动记录每次配置变更的历史版本，默认保留最近 **30天** 的历史记录。

### 3.8.1 历史版本查询 API

```bash
# 查询配置历史版本列表
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/history' \
  -G \
  --data-urlencode 'dataId=order-service.yml' \
  --data-urlencode 'group=DEFAULT_GROUP' \
  --data-urlencode 'tenant=prod-ns-id' \
  --data-urlencode 'pageNo=1' \
  --data-urlencode 'pageSize=10'

# 响应示例
{
  "totalCount": 5,
  "pageNumber": 1,
  "pageItems": [
    {
      "id": "100",
      "dataId": "order-service.yml",
      "group": "DEFAULT_GROUP",
      "content": "... v5 内容 ...",
      "md5": "abc123",
      "createdTime": "2024-01-10 10:30:00"
    }
  ]
}
```

### 3.8.2 控制台回滚操作步骤

```text
Nacos 控制台回滚步骤：

1. 登录 Nacos 控制台: http://localhost:8848/nacos
2. 导航到: 配置管理 -> 配置列表
3. 找到目标配置，点击"更多" -> "历史版本"
4. 查看历史版本列表（显示时间、操作者、MD5）
5. 找到要回滚的版本，点击"回滚"
6. 确认回滚操作
7. Nacos 将该历史版本内容设为当前版本
8. 订阅该配置的所有实例收到变更通知，自动刷新

效果：整个过程不需要重启服务，约1秒内生效
```

---

# Part 4: Nacos 集群与高可用

## 4.1 集群部署架构

### 4.1.1 标准 3 节点集群架构

```text
                    +------------------------------------------+
                    |          SLB / Nginx                      |
                    |    (负载均衡 / 虚拟IP)                     |
                    |    入口: http://nacos-cluster              |
                    +------------------+-----------------------+
                                       |
          +--------------------------+-+------------------------+
          |                          |                          |
          v                          v                          v
+-----------------+       +-----------------+       +-----------------+
| Nacos Server 1  |       | Nacos Server 2  |       | Nacos Server 3  |
| (Leader)        +<----->+ (Follower)      +<----->+ (Follower)      |
| IP:192.168.1.1  |       | IP:192.168.1.2  |       | IP:192.168.1.3  |
| Port:8848/9848  |       | Port:8848/9848  |       | Port:8848/9848  |
|                 |       |                 |       |                 |
| Nacos 进程      |       | Nacos 进程       |       | Nacos 进程       |
| JVM: 4GB        |       | JVM: 4GB        |       | JVM: 4GB        |
+---------+-------+       +---------+-------+       +---------+-------+
          |                         |                         |
          +-------------------------+-------------------------+
                                    | 共用同一 MySQL
                                    v
                      +---------------------------+
                      |       MySQL 5.7+          |
                      |  IP: 192.168.1.10         |
                      |  Port: 3306               |
                      |  DB: nacos_config         |
                      +---------------------------+
```

### 4.1.2 集群配置文件说明

**cluster.conf 文件**（每个节点都需要配置，内容相同）：

```text
# 文件位置: /conf/cluster.conf
# 列出集群中所有节点的 IP:Port
192.168.1.1:8848
192.168.1.2:8848
192.168.1.3:8848
```

**application.properties 关键配置**（每个节点分别配置 nacos.inetutils.ip-address）：

```properties
# Nacos 数据存储模式，集群必须使用 MySQL
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://192.168.1.10:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user=nacos
db.password=nacos_password

# Nacos Server 端口
server.port=8848

# 本节点 IP（多网卡时需要指定，否则可能注册到错误的网卡IP）
nacos.inetutils.ip-address=192.168.1.1
```

### 4.1.3 Docker Compose 集群部署示例

```yaml
version: '3'
services:
  nacos1:
    image: nacos/nacos-server:v2.2.3
    container_name: nacos1
    ports:
      - "8848:8848"
      - "9848:9848"
      - "9849:9849"
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_PORT=3306
      - MYSQL_SERVICE_DB_NAME=nacos_config
      - MYSQL_SERVICE_USER=nacos
      - MYSQL_SERVICE_PASSWORD=nacos_password
      - JVM_XMS=2g
      - JVM_XMX=2g
    networks:
      - nacos-network

  nacos2:
    image: nacos/nacos-server:v2.2.3
    container_name: nacos2
    ports:
      - "8849:8848"
      - "9849:9848"
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_DB_NAME=nacos_config
      - MYSQL_SERVICE_USER=nacos
      - MYSQL_SERVICE_PASSWORD=nacos_password
    networks:
      - nacos-network

  nacos3:
    image: nacos/nacos-server:v2.2.3
    container_name: nacos3
    ports:
      - "8850:8848"
      - "9850:9848"
    environment:
      - MODE=cluster
      - NACOS_SERVERS=nacos1:8848 nacos2:8848 nacos3:8848
      - SPRING_DATASOURCE_PLATFORM=mysql
      - MYSQL_SERVICE_HOST=mysql
      - MYSQL_SERVICE_DB_NAME=nacos_config
      - MYSQL_SERVICE_USER=nacos
      - MYSQL_SERVICE_PASSWORD=nacos_password
    networks:
      - nacos-network

  mysql:
    image: mysql:5.7
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=nacos_config
      - MYSQL_USER=nacos
      - MYSQL_PASSWORD=nacos_password
    volumes:
      - ./nacos-mysql.sql:/docker-entrypoint-initdb.d/nacos-mysql.sql
    networks:
      - nacos-network

networks:
  nacos-network:
    driver: bridge
```

---

## 4.2 数据持久化：Derby vs MySQL

### 4.2.1 Derby vs MySQL 对比

```text
+-------------------+----------------------+---------------------------+
|      特性          |       Derby          |       MySQL               |
+-------------------+----------------------+---------------------------+
| 类型              | 内嵌数据库            | 外部数据库                 |
| 安装              | 无需安装              | 需要单独安装               |
| 集群支持          | 不支持                | 支持                      |
| 数据持久化        | 本地文件              | 独立存储                   |
| 高可用            | 无                   | 可搭建主从/MGR              |
| 适用场景          | 单机开发测试          | 生产环境（必须）            |
+-------------------+----------------------+---------------------------+
```

### 4.2.2 MySQL 核心表结构

```sql
-- 创建数据库
CREATE DATABASE nacos_config CHARACTER SET utf8mb4;
USE nacos_config;

-- =====================================================
-- 配置信息表（核心表）
-- =====================================================
CREATE TABLE config_info (
    id            BIGINT(20)   NOT NULL AUTO_INCREMENT COMMENT '自增主键',
    data_id       VARCHAR(255) NOT NULL COMMENT 'data_id',
    group_id      VARCHAR(128) DEFAULT NULL COMMENT 'group_id',
    content       LONGTEXT     NOT NULL COMMENT '内容',
    md5           VARCHAR(32)  DEFAULT NULL COMMENT 'md5',
    gmt_create    DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    gmt_modified  DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    src_user      TEXT         COMMENT '操作人',
    src_ip        VARCHAR(50)  DEFAULT NULL COMMENT '操作IP',
    app_name      VARCHAR(128) DEFAULT NULL,
    tenant_id     VARCHAR(128) DEFAULT '' COMMENT '租户字段（namespace）',
    c_desc        VARCHAR(256) DEFAULT NULL COMMENT '配置描述',
    type          VARCHAR(64)  DEFAULT NULL COMMENT '配置类型',
    encrypted_data_key TEXT    NOT NULL DEFAULT '' COMMENT '加密Key',
    PRIMARY KEY (id),
    UNIQUE KEY uk_configinfo_datagrouptenant (data_id, group_id, tenant_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='config_info';

-- =====================================================
-- 配置历史表（保留历史版本，用于回滚）
-- =====================================================
CREATE TABLE his_config_info (
    id           BIGINT(64)   UNSIGNED NOT NULL,
    nid          BIGINT(20)   UNSIGNED NOT NULL AUTO_INCREMENT,
    data_id      VARCHAR(255) NOT NULL,
    group_id     VARCHAR(128) NOT NULL,
    content      LONGTEXT     NOT NULL COMMENT '历史内容',
    md5          VARCHAR(32)  DEFAULT NULL,
    gmt_create   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    gmt_modified DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP,
    src_user     TEXT,
    src_ip       VARCHAR(50)  DEFAULT NULL,
    op_type      CHAR(10)     DEFAULT NULL COMMENT '操作类型 I/U/D',
    tenant_id    VARCHAR(128) DEFAULT '',
    PRIMARY KEY (nid),
    KEY idx_gmt_create (gmt_create),
    KEY idx_did (data_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='配置历史记录';

-- =====================================================
-- 租户（Namespace）信息表
-- =====================================================
CREATE TABLE tenant_info (
    id            BIGINT(20)   NOT NULL AUTO_INCREMENT,
    kp            VARCHAR(128) NOT NULL COMMENT '固定值 1',
    tenant_id     VARCHAR(128) DEFAULT '' COMMENT 'namespace ID',
    tenant_name   VARCHAR(128) DEFAULT '' COMMENT 'namespace 名称',
    tenant_desc   VARCHAR(256) DEFAULT NULL,
    create_source VARCHAR(32)  DEFAULT NULL,
    gmt_create    BIGINT(20)   NOT NULL,
    gmt_modified  BIGINT(20)   NOT NULL,
    PRIMARY KEY (id),
    UNIQUE KEY uk_tenant_info_kptenantid (kp, tenant_id),
    KEY idx_tenant_id (tenant_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='tenant_info';

-- =====================================================
-- 灰度配置表（Beta 发布）
-- =====================================================
CREATE TABLE config_info_beta (
    id             BIGINT(20)    NOT NULL AUTO_INCREMENT,
    data_id        VARCHAR(255)  NOT NULL,
    group_id       VARCHAR(128)  NOT NULL,
    content        LONGTEXT      NOT NULL COMMENT '灰度配置内容',
    beta_ips       VARCHAR(1024) DEFAULT NULL COMMENT '灰度IP列表（逗号分隔）',
    md5            VARCHAR(32)   DEFAULT NULL,
    gmt_create     DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    gmt_modified   DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP,
    src_user       TEXT,
    src_ip         VARCHAR(50)   DEFAULT NULL,
    tenant_id      VARCHAR(128)  DEFAULT '',
    PRIMARY KEY (id),
    UNIQUE KEY uk_configinfobeta_datagrouptenant (data_id, group_id, tenant_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='灰度配置表';

-- =====================================================
-- 用户表（鉴权功能）
-- =====================================================
CREATE TABLE users (
    username VARCHAR(50)  NOT NULL PRIMARY KEY,
    password VARCHAR(500) NOT NULL,
    enabled  BOOLEAN      NOT NULL
);

CREATE TABLE roles (
    username VARCHAR(50) NOT NULL,
    role     VARCHAR(50) NOT NULL,
    UNIQUE INDEX idx_user_role (username, role)
);

CREATE TABLE permissions (
    role     VARCHAR(50)  NOT NULL,
    resource VARCHAR(512) NOT NULL,
    action   VARCHAR(8)   NOT NULL,
    UNIQUE INDEX uk_role_permission (role, resource, action)
);

-- 插入默认管理员（密码: nacos，BCrypt 加密）
INSERT INTO users (username, password, enabled)
VALUES ('nacos', '', TRUE);
INSERT INTO roles (username, role) VALUES ('nacos', 'ROLE_ADMIN');
```

---

## 4.3 Raft 协议在 Nacos 中的应用

### 4.3.1 Raft Leader 选举流程

```text
Raft Leader 选举流程：

初始状态：所有节点都是 Follower（等待心跳，随机超时则发起选举）
         Node1(Follower)   Node2(Follower)   Node3(Follower)

Step1: 随机选举超时触发（150-300ms 随机，避免同时触发）
       假设 Node1 最先超时

       Node1 变为 Candidate，递增 currentTerm = 1
       Node1(Candidate)   Node2(Follower)   Node3(Follower)

Step2: Node1 向其他节点发送 RequestVote RPC
       Node1 --RequestVote(term=1, lastLogIndex=5)--> Node2
       Node1 --RequestVote(term=1, lastLogIndex=5)--> Node3

Step3: Node2, Node3 在 term=1 尚未投票，且 Node1 的日志不落后，同意
       Node2 --VoteGranted(term=1)--> Node1
       Node3 --VoteGranted(term=1)--> Node1

Step4: Node1 获得多数票（2/3 >= 半数+1），成为 Leader
       Node1(Leader)   Node2(Follower)   Node3(Follower)

Step5: Node1 立即开始发送心跳，防止其他节点选举超时
       Node1 --AppendEntries(heartbeat, term=1)--> Node2
       Node1 --AppendEntries(heartbeat, term=1)--> Node3

网络分区场景：
  假设 Node1 和 Node2 网络隔离
  Node3 超时，发起选举（term=2），Node2 投票 -> Node3 成为新 Leader
  Node1 仍以为自己是 Leader，但写入操作无法得到多数确认（只有自己）
  -> Node1 的写入失败，保证了集群数据一致性
```

### 4.3.2 Raft 日志复制

```text
配置写入 Raft 日志复制流程：

Client --写入配置--> Node1(Leader)
                         |
                    Step1: Leader 将日志追加到本地（未提交状态）
                    Log: [index=100, term=2, data=config_v2, committed=false]
                         |
                    Step2: 并行发送 AppendEntries 给所有 Follower
                         |--AppendEntries(index=100, data=config_v2)--> Node2
                         |--AppendEntries(index=100, data=config_v2)--> Node3
                         |
                    Step3: Follower 写入本地日志，返回成功
                    Node2: --Success(index=100)-->  Node1
                    Node3: --Success(index=100)-->  Node1
                         |
                    Step4: Leader 收到多数确认（2/3），提交日志
                    Log: [index=100, committed=true]
                    写入 MySQL
                         |
                    Step5: 下次心跳告知 Follower commitIndex=100
                         |
                    Step6: Follower 提交，写入 MySQL
                         |
                    Step7: Leader 返回成功给 Client
Client <--写入成功-- Node1(Leader)
```

---

## 4.4 Nacos 2.x gRPC 长连接改造

### 4.4.1 1.x vs 2.x 架构对比

```text
Nacos 1.x 架构（HTTP 短连接）：
+-----------------------------------------------------+
| 问题：                                               |
|  · 每次心跳都是独立的 HTTP 请求                      |
|  · 大量短连接，TCP 握手/挥手开销大                   |
|  · 推送使用 UDP，不可靠，需要重试机制                |
|  · 服务端需要维护大量连接状态                        |
|                                                     |
| 典型性能（1万实例）：                                |
|  · 心跳 QPS: 约 2000 QPS HTTP 请求                  |
|  · 推送延迟: ~1-5s（UDP可能丢包重试）                |
+-----------------------------------------------------+

Nacos 2.x 架构（gRPC 长连接）：
+-----------------------------------------------------+
| 改进：                                               |
|  · 每个客户端建立一条 gRPC 长连接                    |
|  · 心跳、注册、查询、推送复用同一连接                |
|  · 推送通过 gRPC 双向流，可靠且低延迟                |
|  · 服务端连接管理更轻量                              |
|                                                     |
| 性能提升：                                          |
|  · 支持实例数: 20万 -> 100万                         |
|  · 心跳消耗: 减少约 50%                             |
|  · 推送延迟: ~100-200ms                             |
+-----------------------------------------------------+
```

### 4.4.2 Nacos 2.x 端口分配

```text
Nacos 2.x 端口分配：

8848  --- HTTP 端口（控制台访问、REST API、兼容 1.x 客户端）
9848  --- gRPC 端口（2.x 客户端长连接，= 8848 + 1000）
9849  --- gRPC 集群通信端口（节点间同步，= 8848 + 1001）
7848  --- Raft 端口（集群选举，= 8848 - 1000）

端口计算规则：
  gRPC Port = HTTP Port + 1000
  Raft Port  = HTTP Port - 1000

防火墙规则（集群部署必须开放）：
  外部访问：  8848（控制台和API）, 9848（客户端gRPC）
  节点间通信：8848, 9848, 9849, 7848

注意：如果自定义 HTTP Port（如 8849），
      gRPC Port 自动变为 9849，不能单独修改
```

### 4.4.3 2.x 客户端连接生命周期

```text
gRPC 连接生命周期：

客户端启动
     |
     v
建立 gRPC 长连接到 Nacos Server（选择可用节点）
     |
     v
发送 ConnectionSetupRequest（包含客户端信息）
     |
     v
服务端注册该连接
     |
     +-- 注册实例 -> 通过长连接发送 InstanceRequest
     |
     +-- 心跳     -> 通过长连接发送 HealthCheckRequest（每5秒）
     |
     +-- 推送     <- 服务端通过长连接发送 NotifySubscriberRequest
     |
     +-- 连接断开（网络异常）
          |
          v
         自动重连（指数退避，最长30s）
          |
          v
         重新注册实例（重连后自动重新注册）
```

---

# Part 5: Spring Boot/Cloud 集成实战

## 5.1 Maven 依赖配置

### 5.1.1 父 POM 版本管理

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.18</version>
    <relativePath/>
</parent>

<properties>
    <java.version>11</java.version>
    <spring-cloud.version>2021.0.9</spring-cloud.version>
    <spring-cloud-alibaba.version>2021.0.6.0</spring-cloud-alibaba.version>
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
```

### 5.1.2 服务注册 + 配置中心完整依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Nacos 服务注册发现 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    </dependency>

    <!-- Nacos 配置中心 -->
    <dependency>
        <groupId>com.alibaba.cloud</groupId>
        <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    </dependency>

    <!-- Spring Cloud LoadBalancer -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>

    <!-- OpenFeign -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-openfeign</artifactId>
    </dependency>

    <!-- Spring Boot Actuator -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!--
        spring-cloud-starter-bootstrap
        Spring Cloud 2020.x 以后 bootstrap.yml 默认不加载，需要此依赖开启
    -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-bootstrap</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 5.2 bootstrap.yml 完整配置模板

```yaml
# bootstrap.yml
# 注意：bootstrap.yml 在 application.yml 之前加载
# 必须将 Nacos 配置中心的配置放在 bootstrap.yml 中

spring:
  application:
    name: order-service        # 服务名（也是注册到 Nacos 的服务名）

  profiles:
    active: dev                # 激活的环境

  cloud:
    nacos:
      # ============================================================
      # 服务注册发现配置
      # ============================================================
      discovery:
        server-addr: 127.0.0.1:8848   # Nacos 服务地址（集群用逗号分隔）
        namespace: dev-namespace-id    # 命名空间 ID（不是名字！）
        group: DEFAULT_GROUP           # 分组
        cluster-name: BJ-CLUSTER       # 集群名称（用于同集群优先路由）
        weight: 1.0                    # 服务权重（1.0 为默认权重）
        ephemeral: true                # 是否临时实例（默认 true）
        enabled: true                  # 是否启用服务发现
        heart-beat-interval: 5000      # 心跳间隔（毫秒）
        heart-beat-timeout: 15000      # 实例不健康阈值（毫秒）
        ip-delete-timeout: 30000       # 实例删除阈值（毫秒）
        metadata:
          version: v1.0.0
          env: dev
          region: beijing
        username: nacos
        password: nacos
        register-enabled: true         # 是否注册当前服务

      # ============================================================
      # 配置中心配置
      # ============================================================
      config:
        server-addr: 127.0.0.1:8848
        namespace: dev-namespace-id
        group: DEFAULT_GROUP
        file-extension: yaml           # 配置文件格式
        username: nacos
        password: nacos
        refresh-enabled: true          # 自动刷新（默认 true）
        config-long-poll-timeout: 46000
        config-retry-time: 2200
        max-retry: 3

        # 共享配置（多个服务共用的配置，优先级低于主配置）
        shared-configs:
          - data-id: common-database.yaml
            group: DEFAULT_GROUP
            refresh: true
          - data-id: common-redis.yaml
            group: DEFAULT_GROUP
            refresh: true

        # 扩展配置（优先级低于主配置但高于共享配置）
        extension-configs:
          - data-id: order-service-extra.yaml
            group: ORDER_GROUP
            refresh: true

server:
  port: 8080

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,nacos-discovery,refresh
  endpoint:
    health:
      show-details: always
```

---

## 5.3 服务注册启动类配置

```java
package com.example.orderservice;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.openfeign.EnableFeignClients;

/**
 * 订单服务启动类
 *
 * @EnableDiscoveryClient：开启服务注册发现功能
 *   - 向 Nacos 注册当前服务实例
 *   - 启用从 Nacos 发现其他服务的能力
 *   - Spring Boot 2.x 中可以省略（auto-configure 会自动开启）
 *
 * @EnableFeignClients：开启 OpenFeign 声明式 HTTP 客户端
 */
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients(basePackages = "com.example.orderservice.feign")
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

---

## 5.4 服务发现：DiscoveryClient 使用

```java
package com.example.orderservice.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/discovery")
public class DiscoveryController {

    @Autowired
    private DiscoveryClient discoveryClient;

    /** 获取所有注册的服务名称列表 */
    @GetMapping("/services")
    public List<String> getServices() {
        return discoveryClient.getServices();
    }

    /** 根据服务名获取所有实例 */
    @GetMapping("/instances/{serviceName}")
    public List<ServiceInstance> getInstances(@PathVariable String serviceName) {
        List<ServiceInstance> instances = discoveryClient.getInstances(serviceName);
        instances.forEach(instance -> {
            System.out.println("服务ID: " + instance.getServiceId());
            System.out.println("IP: " + instance.getHost());
            System.out.println("端口: " + instance.getPort());
            System.out.println("URI: " + instance.getUri());
            System.out.println("元数据: " + instance.getMetadata());
        });
        return instances;
    }
}
```

---

## 5.5 配置中心集成：@Value 与 @ConfigurationProperties

### 5.5.1 @Value 方式

```java
package com.example.orderservice.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

/**
 * 使用 @Value 注入配置
 *
 * 关键：@RefreshScope 注解使该 Bean 支持配置动态刷新
 * 当 Nacos 配置发生变更时，Spring 会重新创建该 Bean
 * 新 Bean 中的 @Value 字段会获取最新的配置值
 *
 * 注意事项：
 * 1. @RefreshScope 是懒加载的，第一次使用时才创建
 * 2. 刷新时会销毁旧 Bean，重新创建新 Bean
 * 3. 如果 Bean 中有状态（如连接池），需要处理重建逻辑
 */
@Component
@RefreshScope  // 重要！没有此注解，配置变更后 @Value 不会更新
public class OrderConfig {

    /**
     * 订单超时时间（毫秒）
     * 对应 Nacos 配置: order.timeout=5000
     * 冒号后是默认值，Nacos 中没有配置时使用
     */
    @Value("${order.timeout:5000}")
    private int timeout;

    @Value("${order.retry-count:3}")
    private int retryCount;

    @Value("${order.enable-cache:true}")
    private boolean enableCache;

    @Value("${spring.application.name}")
    private String appName;

    public int getTimeout() { return timeout; }
    public int getRetryCount() { return retryCount; }
    public boolean isEnableCache() { return enableCache; }
    public String getAppName() { return appName; }
}
```

### 5.5.2 @ConfigurationProperties 方式（推荐）

```java
package com.example.orderservice.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;

/**
 * 使用 @ConfigurationProperties 批量绑定配置
 *
 * 优点：
 * 1. 类型安全（编译期检查）
 * 2. IDE 自动补全（需要 spring-boot-configuration-processor）
 * 3. 支持复杂类型（List/Map/嵌套对象）
 * 4. 支持 JSR-303 校验（@NotNull、@Min 等）
 */
@Component
@RefreshScope
@ConfigurationProperties(prefix = "order")
public class OrderProperties {

    private int timeout = 5000;
    private int retryCount = 3;
    private boolean enableCache = true;
    private BigDecimal maxAmount = BigDecimal.valueOf(10000);
    private List<String> paymentMethods = new ArrayList<>();
    private DatabaseConfig database = new DatabaseConfig();

    public static class DatabaseConfig {
        private String url;
        private String username;
        private int maxPoolSize = 10;

        public String getUrl() { return url; }
        public void setUrl(String url) { this.url = url; }
        public String getUsername() { return username; }
        public void setUsername(String username) { this.username = username; }
        public int getMaxPoolSize() { return maxPoolSize; }
        public void setMaxPoolSize(int maxPoolSize) { this.maxPoolSize = maxPoolSize; }
    }

    public int getTimeout() { return timeout; }
    public void setTimeout(int timeout) { this.timeout = timeout; }
    public int getRetryCount() { return retryCount; }
    public void setRetryCount(int retryCount) { this.retryCount = retryCount; }
    public boolean isEnableCache() { return enableCache; }
    public void setEnableCache(boolean enableCache) { this.enableCache = enableCache; }
    public BigDecimal getMaxAmount() { return maxAmount; }
    public void setMaxAmount(BigDecimal maxAmount) { this.maxAmount = maxAmount; }
    public List<String> getPaymentMethods() { return paymentMethods; }
    public void setPaymentMethods(List<String> paymentMethods) { this.paymentMethods = paymentMethods; }
    public DatabaseConfig getDatabase() { return database; }
    public void setDatabase(DatabaseConfig database) { this.database = database; }
}
```

对应的 Nacos 配置内容（YAML格式）：

```yaml
# Nacos DataId: order-service.yml
order:
  timeout: 8000
  retry-count: 5
  enable-cache: true
  max-amount: 50000
  payment-methods:
    - ALIPAY
    - WECHAT_PAY
    - BANK_CARD
  database:
    url: jdbc:mysql://prod-db:3306/order_db
    username: order_user
    max-pool-size: 20
```

---

## 5.6 @RefreshScope 动态刷新深度解析

### 5.6.1 @RefreshScope 工作原理

```text
@RefreshScope 工作原理：

正常 Bean（singleton）：
  Spring Context 启动时创建 Bean，存入 BeanFactory
  配置变更时，Bean 不重新创建，@Value 字段值不变  <- 无法动态刷新

@RefreshScope Bean：
  第一次使用时创建 Bean（懒加载），存入 RefreshScope 作用域缓存

  配置变更流程：
  +----------------------------------------------------------+
  | 1. Nacos 推送配置变更通知（长轮询或 gRPC）               |
  | 2. NacosContextRefresher 接收通知                         |
  | 3. 发布 RefreshEvent 事件                                 |
  | 4. ContextRefresher.refresh() 被触发                     |
  | 5. Environment 重新绑定新的配置属性                       |
  | 6. RefreshScope.refreshAll() 清空所有 @RefreshScope Bean  |
  |    的缓存（销毁旧 Bean 实例）                              |
  | 7. 下次使用这些 Bean 时，重新创建（获取新配置值）          |
  +----------------------------------------------------------+

注意事项：
  · @RefreshScope 内的 @Autowired 字段也会重新注入
  · 如果 Bean 有复杂初始化逻辑（如建立连接），刷新后会重新执行
  · @RefreshScope 和 @Scheduled 配合使用时需注意（每次刷新后定时任务会重启）
```

### 5.6.2 监听配置刷新事件

```java
package com.example.orderservice.event;

import org.springframework.cloud.context.environment.EnvironmentChangeEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

import java.util.Set;

/**
 * 配置刷新事件监听器
 * 可以在这里执行配置刷新后的回调逻辑（如重建连接池、清空缓存等）
 */
@Component
public class ConfigRefreshListener {

    @EventListener
    public void onEnvironmentChange(EnvironmentChangeEvent event) {
        Set<String> changedKeys = event.getKeys();
        System.out.println("=== 配置已刷新，变更的 Key: " + changedKeys + " ===");

        if (changedKeys.contains("order.database.max-pool-size")) {
            System.out.println("数据库连接池大小配置变更，执行连接池重建...");
            // rebuildConnectionPool();
        }

        if (changedKeys.stream().anyMatch(k -> k.startsWith("order.cache"))) {
            System.out.println("缓存配置变更，清空本地缓存...");
            // clearLocalCache();
        }
    }
}
```

### 5.6.3 手动触发配置刷新

```bash
# 手动触发所有 @RefreshScope Bean 刷新
curl -X POST http://localhost:8080/actuator/refresh

# 响应（返回变更的 key 列表）
["order.timeout","order.retry-count"]

# 注意：需要在 bootstrap.yml 中开放 refresh 端点
# management.endpoints.web.exposure.include: refresh
```

---

## 5.7 多环境配置方案

### 5.7.1 方案一：Namespace 隔离（推荐生产使用）

```yaml
# 通过 JVM 参数 -Dspring.profiles.active=prod 区分环境
# 通过环境变量 NACOS_NAMESPACE 注入对应 namespace ID

spring:
  application:
    name: order-service
  cloud:
    nacos:
      config:
        server-addr: ${NACOS_SERVER_ADDR:127.0.0.1:8848}
        namespace: ${NACOS_NAMESPACE:}   # dev/test/prod 各自的 namespace ID
        file-extension: yaml
```

### 5.7.2 方案二：Profile 激活不同 DataId

```yaml
# bootstrap.yml
spring:
  application:
    name: order-service
  profiles:
    active: dev    # 通过 JVM 参数 -Dspring.profiles.active=prod 覆盖
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        file-extension: yaml
        # Spring Cloud 会自动加载：
        # 1. order-service.yaml           (基础配置，所有环境共用)
        # 2. order-service-dev.yaml       (开发环境特定配置，会覆盖基础配置)
```

对应 Nacos 中的配置文件：

```text
Namespace: public（或统一 namespace）
+-- order-service.yaml          <- 基础配置（通用部分）
+-- order-service-dev.yaml      <- 开发环境覆盖配置
+-- order-service-test.yaml     <- 测试环境覆盖配置
+-- order-service-prod.yaml     <- 生产环境覆盖配置
```

---

## 5.8 配置共享：shared-configs 与 extension-configs

```yaml
spring:
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        namespace: prod-ns-id
        file-extension: yaml

        # 共享配置（最低优先级，多个服务共用）
        shared-configs:
          - data-id: common-datasource.yaml
            group: COMMON_GROUP
            refresh: true
          - data-id: common-redis.yaml
            group: COMMON_GROUP
            refresh: true
          - data-id: common-logging.yaml
            group: COMMON_GROUP
            refresh: false

        # 扩展配置（优先级高于 shared-configs，低于主配置）
        extension-configs:
          - data-id: order-service-mq.yaml
            group: ORDER_GROUP
            refresh: true

# 最终配置优先级（由高到低）：
# 1. order-service-{profile}.yaml   <- 主配置（带 profile）最高优先
# 2. order-service.yaml              <- 主配置（不带 profile）
# 3. order-service-mq.yaml           <- extension-configs
# 4. common-logging.yaml             <- shared-configs（后声明的优先级高）
# 5. common-redis.yaml               <- shared-configs
# 6. common-datasource.yaml          <- shared-configs（最低）
```

---

# Part 6: Nacos 与 OpenFeign 集成

## 6.1 OpenFeign 基本原理

OpenFeign 是 Spring Cloud 官方提供的声明式 HTTP 客户端，它通过动态代理将 HTTP 调用封装为接口方法，结合 Nacos 的服务发现和 LoadBalancer 的负载均衡，实现透明的微服务调用。

```text
OpenFeign 调用流程：

OrderService.createOrder()
     |
     v
productFeignClient.getProductById(productId)  <- 接口方法调用
     |
     v
JDK 动态代理拦截 (FeignInvocationHandler)
     |
     v
ReflectiveFeign.dispatch 获取 MethodHandler
     |
     v
LoadBalancerFeignClient.execute()
     |
     v
SpringCloudLoadBalancer 从 Nacos 获取 product-service 实例列表
     |
     v
负载均衡选择一个实例（如 192.168.1.200:8081）
     |
     v
发送 HTTP 请求: GET http://192.168.1.200:8081/api/v1/products/1
     |
     v
返回响应，反序列化为 ProductDTO
```

## 6.2 Feign 客户端定义

```java
package com.example.orderservice.feign;

import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.*;

import java.util.List;

/**
 * 商品服务 Feign 客户端接口
 *
 * @FeignClient 参数说明：
 *   name/value: 目标服务名（必须与 Nacos 中注册的服务名完全一致）
 *   path:       URL 前缀（可选，对应目标服务的 context-path 或公共路径前缀）
 *   fallback:   降级处理类（接口实现，服务不可用时返回默认值）
 *   url:        直连 URL（用于调试时绕过 Nacos，不走服务发现）
 *   contextId:  当多个 @FeignClient 指向同一个服务时，用于区分 Bean 名称
 */
@FeignClient(
    name = "product-service",
    path = "/api/v1",
    fallback = ProductFeignFallback.class
)
public interface ProductFeignClient {

    /** 根据 ID 查询商品（GET http://product-service/api/v1/products/{id}） */
    @GetMapping("/products/{id}")
    ProductDTO getProductById(@PathVariable("id") Long id);

    /** 批量查询商品（GET http://product-service/api/v1/products/batch?ids=1,2,3） */
    @GetMapping("/products/batch")
    List<ProductDTO> getProductsByIds(@RequestParam("ids") List<Long> ids);

    /** 扣减库存（POST http://product-service/api/v1/products/deduct-stock） */
    @PostMapping("/products/deduct-stock")
    Boolean deductStock(@RequestBody StockDeductRequest request);

    /** 更新商品状态（通过 URL 变量 + 请求头） */
    @PutMapping("/products/{id}/status")
    void updateProductStatus(
        @PathVariable("id") Long id,
        @RequestParam("status") String status,
        @RequestHeader("X-Operator") String operator
    );
}
```

## 6.3 Feign 降级实现

```java
package com.example.orderservice.feign;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

import java.util.Collections;
import java.util.List;

/**
 * 商品服务降级处理
 * 当商品服务不可用时，返回默认值或友好的错误信息
 */
@Component
public class ProductFeignFallback implements ProductFeignClient {

    private static final Logger logger = LoggerFactory.getLogger(ProductFeignFallback.class);

    @Override
    public ProductDTO getProductById(Long id) {
        logger.warn("商品服务不可用，返回默认商品信息，productId={}", id);
        ProductDTO fallback = new ProductDTO();
        fallback.setId(id);
        fallback.setName("商品服务暂时不可用，请稍后重试");
        fallback.setAvailable(false);
        return fallback;
    }

    @Override
    public List<ProductDTO> getProductsByIds(List<Long> ids) {
        logger.warn("商品服务不可用，返回空列表，ids={}", ids);
        return Collections.emptyList();
    }

    @Override
    public Boolean deductStock(StockDeductRequest request) {
        logger.error("库存扣减失败：商品服务不可用");
        return false;
    }

    @Override
    public void updateProductStatus(Long id, String status, String operator) {
        logger.error("更新商品状态失败：商品服务不可用，id={}", id);
    }
}
```

## 6.4 Feign 全局配置

```java
package com.example.orderservice.config;

import feign.Logger;
import feign.Request;
import feign.Retryer;
import feign.codec.ErrorDecoder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.util.concurrent.TimeUnit;

@Configuration
public class FeignConfig {

    /**
     * Feign 日志级别配置
     *
     * NONE：不输出任何日志（默认，生产推荐）
     * BASIC：只记录请求方法、URL、响应状态码和时间（测试推荐）
     * HEADERS：在 BASIC 基础上记录请求/响应头
     * FULL：记录请求/响应的 headers、body 和元数据（开发调试使用）
     *
     * 注意：日志级别还需要在 logging.level 中配置 DEBUG
     * logging.level.com.example.orderservice.feign: DEBUG
     */
    @Bean
    public Logger.Level feignLoggerLevel() {
        return Logger.Level.BASIC;
    }

    /**
     * 超时配置
     */
    @Bean
    public Request.Options requestOptions() {
        return new Request.Options(
            3, TimeUnit.SECONDS,   // connectTimeout: 3秒
            10, TimeUnit.SECONDS,  // readTimeout: 10秒
            true                   // followRedirects
        );
    }

    /**
     * 重试机制配置
     * 警告：开启重试需要确保接口是幂等的！
     * 非幂等接口（如扣减库存）不应该开启重试
     *
     * period: 初始重试间隔（ms）
     * maxPeriod: 最大重试间隔（ms）（指数退避）
     * maxAttempts: 最大重试次数（包含第一次）
     */
    @Bean
    public Retryer feignRetryer() {
        return new Retryer.Default(100, 1000, 3);
    }

    /**
     * 自定义错误解码器
     * 将服务端的错误响应（4xx/5xx）转换为业务异常
     */
    @Bean
    public ErrorDecoder errorDecoder() {
        return (methodKey, response) -> {
            int status = response.status();
            String url = response.request().url();
            switch (status) {
                case 400: return new IllegalArgumentException("请求参数错误，URL=" + url);
                case 401: return new SecurityException("未授权，请先登录，URL=" + url);
                case 403: return new SecurityException("权限不足，URL=" + url);
                case 404: return new RuntimeException("资源不存在，URL=" + url);
                case 503: return new RuntimeException("服务暂时不可用，URL=" + url);
                default:  return new RuntimeException("Feign 调用异常，status=" + status + "，URL=" + url);
            }
        };
    }
}
```

## 6.5 Feign 配置 YAML 方式

```yaml
feign:
  okhttp:
    enabled: true   # 使用 OkHttp 替代默认 HttpURLConnection（性能更好）
  client:
    config:
      default:           # 全局默认配置
        connectTimeout: 3000
        readTimeout: 10000
        loggerLevel: BASIC
      product-service:   # 针对 product-service 的配置（覆盖默认）
        connectTimeout: 5000
        readTimeout: 30000
        loggerLevel: FULL
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/xml,application/json
      min-request-size: 2048  # 2KB 以上才压缩
    response:
      enabled: true

logging:
  level:
    com.example.orderservice.feign: DEBUG
```

## 6.6 完整服务调用案例

```java
package com.example.orderservice.service;

import com.example.orderservice.config.OrderProperties;
import com.example.orderservice.feign.ProductFeignClient;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.time.LocalDateTime;

/**
 * 订单服务实现
 * 通过 Feign 调用商品服务完成订单创建流程
 */
@Service
public class OrderService {

    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);

    @Autowired
    private ProductFeignClient productFeignClient;

    @Autowired
    private OrderProperties orderProperties;  // 动态配置，支持热更新

    public OrderDTO createOrder(CreateOrderRequest request) {
        Long productId = request.getProductId();
        Integer quantity = request.getQuantity();

        logger.info("开始创建订单，productId={}, quantity={}", productId, quantity);

        // 检查订单金额限制（来自 Nacos 动态配置，可以热更新）
        BigDecimal maxAmount = orderProperties.getMaxAmount();

        // 1. 通过 Feign 查询商品信息
        //    Feign + Nacos + LoadBalancer 自动完成：
        //    服务名 "product-service" -> 从 Nacos 获取实例列表 -> 负载均衡 -> HTTP 请求
        ProductDTO product = productFeignClient.getProductById(productId);

        if (product == null || !product.isAvailable()) {
            throw new RuntimeException("商品不存在或已下架，productId=" + productId);
        }

        BigDecimal totalAmount = product.getPrice().multiply(BigDecimal.valueOf(quantity));
        if (totalAmount.compareTo(maxAmount) > 0) {
            throw new RuntimeException("订单金额超出限制，maxAmount=" + maxAmount);
        }

        // 2. 通过 Feign 扣减库存
        StockDeductRequest stockRequest = new StockDeductRequest();
        stockRequest.setProductId(productId);
        stockRequest.setQuantity(quantity);
        Boolean deductResult = productFeignClient.deductStock(stockRequest);

        if (!Boolean.TRUE.equals(deductResult)) {
            throw new RuntimeException("库存不足，productId=" + productId);
        }

        // 3. 创建订单记录
        OrderDTO order = new OrderDTO();
        order.setOrderNo("ORD" + System.currentTimeMillis());
        order.setProductId(productId);
        order.setProductName(product.getName());
        order.setQuantity(quantity);
        order.setTotalAmount(totalAmount);
        order.setStatus("CREATED");
        order.setCreateTime(LocalDateTime.now());

        logger.info("订单创建成功，orderNo={}, amount={}", order.getOrderNo(), totalAmount);
        return order;
    }
}
```

---

# Part 7: Nacos 与 LoadBalancer 集成

## 7.1 Spring Cloud LoadBalancer 概述

从 Spring Cloud 2020.x 开始，官方推荐使用 **Spring Cloud LoadBalancer** 替代已停止维护的 **Netflix Ribbon**。

```text
Ribbon vs Spring Cloud LoadBalancer

+--------------------------------------------------+
|  Ribbon（已停止维护）                              |
|  +-- Netflix 开源，已进入维护模式                  |
|  +-- 功能丰富（多种负载策略、熔断集成）            |
|  +-- Spring Cloud 2020.x 已从默认依赖中移除       |
+--------------------------------------------------+
|  Spring Cloud LoadBalancer（推荐）                |
|  +-- Spring 官方实现，代码简洁                    |
|  +-- 内置轮询和随机两种策略                       |
|  +-- 支持自定义策略（实现接口即可）               |
|  +-- 完整支持 WebFlux（响应式编程）               |
|  +-- 支持缓存（减少服务发现调用）                 |
+--------------------------------------------------+
```

## 7.2 RestTemplate + LoadBalancer

```java
package com.example.orderservice.config;

import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;

@Configuration
public class RestTemplateConfig {

    /**
     * 创建支持负载均衡的 RestTemplate
     *
     * @LoadBalanced 注解：
     *   - 给 RestTemplate 添加 LoadBalancerInterceptor
     *   - 拦截请求，将服务名解析为实际 IP:Port
     *   - 支持 http://服务名/path 格式的 URL
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}
```

```java
@Service
public class ProductServiceRestClient {

    @Autowired
    private RestTemplate restTemplate;  // 必须是 @LoadBalanced 的那个

    public ProductDTO getProduct(Long productId) {
        // "product-service" 是 Nacos 中注册的服务名
        // LoadBalancer 自动替换为实际 IP:Port
        String url = "http://product-service/api/v1/products/" + productId;
        return restTemplate.getForObject(url, ProductDTO.class);
    }
}
```

## 7.3 LoadBalancer 全局配置

```yaml
spring:
  cloud:
    loadbalancer:
      cache:
        enabled: true
        ttl: 35s        # 缓存过期时间
        capacity: 256   # 最大缓存数量
      health-check:
        initial-delay: 0
        interval: 25s
        path:
          default: /actuator/health
```

## 7.4 自定义负载均衡策略：同集群优先

在多机房/多区域部署时，同集群优先是最常用的负载均衡策略：

```java
package com.example.orderservice.loadbalancer;

import com.alibaba.cloud.nacos.NacosDiscoveryProperties;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.DefaultResponse;
import org.springframework.cloud.client.loadbalancer.EmptyResponse;
import org.springframework.cloud.client.loadbalancer.Request;
import org.springframework.cloud.client.loadbalancer.Response;
import org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import reactor.core.publisher.Mono;

import java.util.List;
import java.util.Random;
import java.util.stream.Collectors;

/**
 * 自定义负载均衡策略：同集群优先 + 权重
 *
 * 策略说明：
 * 1. 优先选择与当前服务同集群（cluster-name）的实例
 * 2. 如果同集群无健康实例，则从所有实例中选择（跨集群兜底）
 * 3. 在候选实例中，按权重进行随机选择
 */
public class NacosClusterFirstLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    private static final Logger logger = LoggerFactory.getLogger(NacosClusterFirstLoadBalancer.class);

    private final String serviceId;
    private final ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider;
    private final NacosDiscoveryProperties nacosDiscoveryProperties;
    private final Random random = new Random();

    public NacosClusterFirstLoadBalancer(
            String serviceId,
            ObjectProvider<ServiceInstanceListSupplier> serviceInstanceListSupplierProvider,
            NacosDiscoveryProperties nacosDiscoveryProperties) {
        this.serviceId = serviceId;
        this.serviceInstanceListSupplierProvider = serviceInstanceListSupplierProvider;
        this.nacosDiscoveryProperties = nacosDiscoveryProperties;
    }

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        ServiceInstanceListSupplier supplier = serviceInstanceListSupplierProvider
                .getIfAvailable();
        return supplier.get(request).next()
                .map(this::getInstanceResponse);
    }

    private Response<ServiceInstance> getInstanceResponse(List<ServiceInstance> instances) {
        if (instances.isEmpty()) {
            logger.warn("服务 {} 没有可用实例", serviceId);
            return new EmptyResponse();
        }

        // 获取当前服务所在的集群名称
        String clusterName = nacosDiscoveryProperties.getClusterName();

        // Step1: 筛选同集群的实例
        List<ServiceInstance> sameClusterInstances = instances.stream()
                .filter(instance -> {
                    String instanceCluster = instance.getMetadata().get("nacos.cluster");
                    return clusterName != null && clusterName.equals(instanceCluster);
                })
                .collect(Collectors.toList());

        List<ServiceInstance> candidateInstances;
        if (!sameClusterInstances.isEmpty()) {
            logger.debug("使用同集群实例，集群={}, 可用实例数={}", clusterName, sameClusterInstances.size());
            candidateInstances = sameClusterInstances;
        } else {
            logger.warn("同集群无可用实例，使用跨集群实例，集群={}", clusterName);
            candidateInstances = instances;
        }

        // Step2: 按权重选择实例
        ServiceInstance selected = selectByWeight(candidateInstances);
        return new DefaultResponse(selected);
    }

    /** 按权重随机选择实例 */
    private ServiceInstance selectByWeight(List<ServiceInstance> instances) {
        double totalWeight = instances.stream()
                .mapToDouble(instance -> {
                    String weightStr = instance.getMetadata().get("nacos.weight");
                    return weightStr != null ? Double.parseDouble(weightStr) : 1.0;
                })
                .sum();

        double randomValue = random.nextDouble() * totalWeight;
        double cumulativeWeight = 0;

        for (ServiceInstance instance : instances) {
            String weightStr = instance.getMetadata().get("nacos.weight");
            double weight = weightStr != null ? Double.parseDouble(weightStr) : 1.0;
            cumulativeWeight += weight;
            if (randomValue <= cumulativeWeight) {
                return instance;
            }
        }

        return instances.get(0);  // 兜底
    }
}
```

## 7.5 注册自定义 LoadBalancer

```java
package com.example.orderservice.config;

import com.alibaba.cloud.nacos.NacosDiscoveryProperties;
import com.example.orderservice.loadbalancer.NacosClusterFirstLoadBalancer;
import org.springframework.beans.factory.ObjectProvider;
import org.springframework.cloud.loadbalancer.core.ReactorServiceInstanceLoadBalancer;
import org.springframework.cloud.loadbalancer.core.ServiceInstanceListSupplier;
import org.springframework.cloud.loadbalancer.support.LoadBalancerClientFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.core.env.Environment;

/**
 * 自定义 LoadBalancer 配置
 * 注意：此类不能加 @Configuration 注解（否则变成全局配置，影响所有服务）
 */
public class CustomLoadBalancerConfig {

    @Bean
    public ReactorServiceInstanceLoadBalancer customLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory,
            NacosDiscoveryProperties nacosDiscoveryProperties) {
        String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
        return new NacosClusterFirstLoadBalancer(
                name,
                loadBalancerClientFactory.getLazyProvider(name, ServiceInstanceListSupplier.class),
                nacosDiscoveryProperties
        );
    }
}
```

```java
package com.example.orderservice;

import com.example.orderservice.config.CustomLoadBalancerConfig;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClient;
import org.springframework.cloud.loadbalancer.annotation.LoadBalancerClients;
import org.springframework.context.annotation.Configuration;

// 方式1：对指定服务使用自定义策略
@Configuration
@LoadBalancerClient(name = "product-service", configuration = CustomLoadBalancerConfig.class)
public class LoadBalancerConfiguration {
}

// 方式2：对所有服务使用自定义策略
// @LoadBalancerClients(defaultConfiguration = CustomLoadBalancerConfig.class)
```

---

# Part 8: Nacos 运维管理

## 8.1 Nacos 控制台使用详解

### 8.1.1 控制台访问

```text
访问地址: http://localhost:8848/nacos
默认账号: nacos / nacos

主要功能模块：
+-- 服务管理
|   +-- 服务列表：查看所有注册的服务
|   +-- 服务详情：查看服务的集群、实例信息
|   +-- 实例管理：上下线实例、修改权重、元数据
|   +-- 订阅者查询：查看哪些消费者订阅了该服务
|
+-- 配置管理
|   +-- 配置列表：查看所有配置项
|   +-- 创建配置：新建配置
|   +-- 配置详情：查看、编辑、克隆配置
|   +-- 历史版本：查看配置历史，支持回滚
|   +-- 监听查询：查看哪些实例在监听该配置
|
+-- 权限控制（需开启鉴权）
|   +-- 用户管理
|   +-- 角色管理
|   +-- 权限管理
|
+-- 命名空间
    +-- 创建/删除命名空间
    +-- 查看命名空间 ID
```

### 8.1.2 常用操作步骤

**创建命名空间**：

```text
1. 左下角 -> 命名空间
2. 点击"新建命名空间"
3. 填写命名空间ID（建议自定义有意义的ID，如 dev、test、prod）
   注意：命名空间ID一旦创建不能修改，在 bootstrap.yml 中使用该ID
4. 填写命名空间名称和描述
5. 确定
```

**服务实例上下线**：

```text
1. 服务管理 -> 服务列表 -> 找到目标服务
2. 点击服务名进入详情
3. 找到目标集群，展开实例列表
4. 点击实例右侧的"编辑"按钮
5. 修改"上线/下线"状态
6. 点击确认

效果：下线后的实例不会出现在服务发现的结果中，但实例本身不会被删除
      这是优雅下线的推荐方式（比直接停止服务更安全）
```

**监听查询（查看谁在监听某个配置）**：

```text
1. 配置管理 -> 配置列表 -> 找到目标配置
2. 点击"更多" -> "监听查询"
3. 可以看到所有正在监听该配置的客户端 IP 和 MD5

作用：
  - 确认配置推送是否成功（客户端 MD5 是否与服务端一致）
  - 排查配置未生效的问题（客户端是否在监听）
```

---

## 8.2 Nacos Open API

Nacos 提供了完整的 REST API，可以通过 HTTP 请求进行各种操作。

### 8.2.1 服务注册与管理 API

```bash
# =============================================================
# 注册实例
# =============================================================
curl -X POST 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service' \
  -d 'groupName=DEFAULT_GROUP' \
  -d 'namespaceId=dev-ns-id' \
  -d 'ip=192.168.1.100' \
  -d 'port=8080' \
  -d 'weight=1.0' \
  -d 'healthy=true' \
  -d 'ephemeral=true' \
  -d 'metadata={"version":"v1.0"}'

# =============================================================
# 注销实例
# =============================================================
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service' \
  -d 'groupName=DEFAULT_GROUP' \
  -d 'ip=192.168.1.100' \
  -d 'port=8080'

# =============================================================
# 查询服务实例列表
# =============================================================
curl -X GET 'http://127.0.0.1:8848/nacos/v1/ns/instance/list' \
  -G \
  --data-urlencode 'serviceName=order-service' \
  --data-urlencode 'groupName=DEFAULT_GROUP' \
  --data-urlencode 'namespaceId=dev-ns-id'

# =============================================================
# 发送心跳（仅临时实例需要）
# =============================================================
curl -X PUT 'http://127.0.0.1:8848/nacos/v1/ns/instance/beat' \
  -d 'serviceName=order-service' \
  -d 'groupName=DEFAULT_GROUP' \
  -d 'ip=192.168.1.100' \
  -d 'port=8080'

# =============================================================
# 修改实例（如权重、元数据）
# =============================================================
curl -X PUT 'http://127.0.0.1:8848/nacos/v1/ns/instance' \
  -d 'serviceName=order-service' \
  -d 'ip=192.168.1.100' \
  -d 'port=8080' \
  -d 'weight=2.0' \
  -d 'metadata={"version":"v2.0"}'
```

### 8.2.2 配置管理 API

```bash
# =============================================================
# 获取配置
# =============================================================
curl -X GET 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -G \
  --data-urlencode 'dataId=order-service.yml' \
  --data-urlencode 'group=DEFAULT_GROUP' \
  --data-urlencode 'tenant=dev-ns-id'

# =============================================================
# 发布配置
# =============================================================
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service.yml' \
  -d 'group=DEFAULT_GROUP' \
  -d 'tenant=dev-ns-id' \
  -d 'type=yaml' \
  -d 'content=order.timeout=8000'

# =============================================================
# 删除配置
# =============================================================
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service.yml' \
  -d 'group=DEFAULT_GROUP' \
  -d 'tenant=dev-ns-id'

# =============================================================
# 监听配置（长轮询）
# =============================================================
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs/listener' \
  -H 'Long-Pulling-Timeout:30000' \
  -d 'Listening-Configs=order-service.yml%02DEFAULT_GROUP%02abc123%02dev-ns-id%01'
```

---

## 8.3 监控集成（Prometheus + Grafana）

### 8.3.1 Nacos 开启 Prometheus 监控

```properties
# /conf/application.properties
management.endpoints.web.exposure.include=prometheus,health,info,metrics
```

### 8.3.2 Prometheus 配置

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'nacos'
    static_configs:
      - targets:
        - 'nacos-server-1:8848'
        - 'nacos-server-2:8848'
        - 'nacos-server-3:8848'
    metrics_path: /nacos/actuator/prometheus
    scrape_interval: 15s
```

### 8.3.3 关键监控指标

```text
Nacos 核心监控指标：

服务注册相关：
  nacos_monitor_service_count              - 注册的服务数量
  nacos_monitor_ip_count                   - 注册的实例总数
  nacos_monitor_dom_count                  - 命名空间数量

配置相关：
  nacos_monitor_configCount                - 配置总数
  nacos_config_ops_total                   - 配置操作次数（标签区分read/write）

性能相关：
  nacos_http_server_requests_seconds       - HTTP 请求耗时（百分位）
  nacos_grpc_server_requests_seconds       - gRPC 请求耗时

Raft 相关（集群）：
  nacos_raft_leader                        - 当前节点是否是 Leader
  nacos_raft_term                          - 当前任期

JVM 相关：
  jvm_memory_used_bytes                    - JVM 内存使用
  jvm_gc_pause_seconds                     - GC 停顿时间
```

---

## 8.4 Nacos 日志配置

```text
Nacos 日志目录（/logs/）：
+-- nacos.log          主日志（启动信息、配置变更）
+-- nacos-naming.log   服务注册/发现相关日志
+-- nacos-config.log   配置中心相关日志
+-- nacos-raft.log     Raft 协议日志（集群）
+-- remote.log         gRPC 通信日志（2.x）
+-- remote-digest.log  gRPC 摘要日志

日志级别调整（动态调整，不需要重启）：
```

```bash
# 动态调整日志级别（2.x 支持）
curl -X PUT 'http://127.0.0.1:8848/nacos/v1/ns/operator/log' \
  --data-urlencode 'logName=naming-raft' \
  --data-urlencode 'logLevel=DEBUG'
```

---

## 8.5 常见问题排查

### 8.5.1 服务注册失败

```text
问题现象: 服务启动后，在 Nacos 控制台看不到注册的实例

排查步骤：
1. 检查 bootstrap.yml 配置
   - server-addr 是否正确？
   - namespace 是否使用的是 ID 而不是名称？
   - 端口 8848 是否可访问（telnet 测试）？

2. 检查启动日志
   - 搜索 "register to server" 关键字
   - 是否有连接拒绝或超时的异常？

3. 检查 Nacos 服务端日志
   - /logs/nacos-naming.log
   - 搜索对应的服务名

4. 检查网络
   - 客户端到 Nacos 的网络是否通畅？
   - 防火墙是否放开了 8848 和 9848 端口？

5. 检查鉴权配置
   - 如果 Nacos 开启了鉴权，username/password 是否正确？
```

### 8.5.2 配置不更新问题

```text
问题现象: 在 Nacos 控制台修改了配置，但服务端没有生效

排查步骤：
1. 检查是否添加了 @RefreshScope 注解
   - 使用 @Value 的类必须加 @RefreshScope
   - 使用 @ConfigurationProperties 的类也需要加 @RefreshScope

2. 检查配置三元组是否匹配
   - 控制台修改的是哪个 namespace/group/dataId？
   - 客户端 bootstrap.yml 中配置的是哪个 namespace/group？
   - 应用名和 file-extension 是否与 dataId 匹配？

3. 查看监听查询
   - 在控制台的"监听查询"中检查客户端是否在监听该配置
   - 客户端 MD5 与服务端 MD5 是否不一致（不一致说明推送成功但刷新失败）

4. 检查 Actuator 端点
   curl http://localhost:8080/actuator/nacos-config
   查看当前加载的配置

5. 手动触发刷新测试
   curl -X POST http://localhost:8080/actuator/refresh
   如果手动刷新成功，说明问题在于自动推送机制
```

### 8.5.3 集群脑裂问题

```text
问题现象: Nacos 集群节点之间出现数据不一致，或多个节点都认为自己是 Leader

产生原因：
  - 网络分区：节点之间网络不通，但各自与客户端的网络正常
  - JVM GC 停顿：STW 时间过长，导致心跳超时，重新触发选举

排查步骤：
1. 检查 Raft 日志
   /logs/nacos-raft.log
   查看是否有多个 Leader 产生的记录

2. 检查网络
   - 节点之间的 7848 端口（Raft 通信）是否互通？
   - 节点之间的 9849 端口（gRPC 集群同步）是否互通？

3. 检查 JVM 配置
   - GC 是否频繁？STW 时间是否过长？
   - 建议使用 G1GC 或 ZGC，降低 STW 时间

解决方案：
  - 恢复网络连通性
  - 重启出现问题的节点（会触发重新选举和数据同步）
  - 增加 Raft 心跳超时时间（减少误判）

预防措施：
  - 集群节点数量建议奇数（3/5/7），避免投票平局
  - 使用高速网络（万兆），减少延迟
  - JVM 配置足够内存，避免频繁 GC
```

---

# Part 9: 完整实战案例

## 案例1：微服务注册与发现（order-service 调用 product-service）

### 9.1.1 项目架构

```text
微服务架构示意图：

              +----------------+
              |   API Gateway  |
              |   (8080)       |
              +-------+--------+
                      |
          +-----------+-----------+
          |                       |
+---------+--------+  +-----------+------+
|  order-service   |  |  product-service  |
|  (8081)          |  |  (8082/8083)      |
|                  |  |                   |
| @EnableDiscovery |  | @EnableDiscovery  |
| Feign调用产品服务 |  | 提供商品查询接口  |
+------------------+  +-------------------+
          |                       |
          +----------+------------+
                     |
              +------+-------+
              | Nacos Server  |
              | (8848)        |
              +---------------+
```

### 9.1.2 product-service 完整代码

**ProductController.java**：

```java
package com.example.productservice.controller;

import com.example.productservice.dto.ProductDTO;
import com.example.productservice.service.ProductService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    @Autowired
    private ProductService productService;

    @GetMapping("/{id}")
    public ProductDTO getProductById(@PathVariable Long id) {
        return productService.findById(id);
    }

    @GetMapping("/batch")
    public List<ProductDTO> getProductsByIds(@RequestParam List<Long> ids) {
        return productService.findByIds(ids);
    }

    @PostMapping("/deduct-stock")
    public Boolean deductStock(@RequestBody StockDeductRequest request) {
        return productService.deductStock(request.getProductId(), request.getQuantity());
    }
}
```

**ProductService.java**：

```java
package com.example.productservice.service;

import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@Service
public class ProductService {

    private static final Map<Long, ProductDTO> PRODUCT_DB = new ConcurrentHashMap<>();
    private static final Map<Long, Integer> STOCK_DB = new ConcurrentHashMap<>();

    static {
        PRODUCT_DB.put(1L, new ProductDTO(1L, "iPhone 15 Pro", new BigDecimal("7999"), true));
        PRODUCT_DB.put(2L, new ProductDTO(2L, "MacBook Pro M3", new BigDecimal("14999"), true));
        STOCK_DB.put(1L, 100);
        STOCK_DB.put(2L, 50);
    }

    public ProductDTO findById(Long id) {
        ProductDTO product = PRODUCT_DB.get(id);
        if (product == null) {
            throw new RuntimeException("商品不存在，id=" + id);
        }
        return product;
    }

    public List<ProductDTO> findByIds(List<Long> ids) {
        return ids.stream().map(this::findById).collect(Collectors.toList());
    }

    public synchronized Boolean deductStock(Long productId, Integer quantity) {
        Integer stock = STOCK_DB.get(productId);
        if (stock == null || stock < quantity) {
            return false;
        }
        STOCK_DB.put(productId, stock - quantity);
        return true;
    }
}
```

---

## 案例2：多环境配置管理（dev/test/prod 自动切换）

### 9.2.1 Nacos 配置设计

```text
Nacos 多环境配置结构：

Namespace: dev (dev-xxxx-namespace-id)
  +-- DataId: order-service.yml
      内容：DB 地址指向开发库，order.max-amount: 1000

Namespace: test (test-xxxx-namespace-id)
  +-- DataId: order-service.yml
      内容：DB 地址指向测试库，order.max-amount: 5000

Namespace: prod (prod-xxxx-namespace-id)
  +-- DataId: order-service.yml
      内容：DB 地址指向生产库，order.max-amount: 100000
```

### 9.2.2 bootstrap.yml 配置

```yaml
spring:
  application:
    name: order-service
  profiles:
    active: dev    # 通过 -Dspring.profiles.active=prod JVM参数覆盖
  cloud:
    nacos:
      config:
        server-addr: 127.0.0.1:8848
        # 通过环境变量注入对应 namespace ID
        # 启动命令: java -jar app.jar -DNACOS_NAMESPACE=prod-xxxx
        namespace: ${NACOS_NAMESPACE:}
        file-extension: yaml
```

### 9.2.3 Kubernetes 部署配置

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
    spec:
      containers:
      - name: order-service
        image: order-service:v1.0.0
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: NACOS_SERVER_ADDR
          value: "nacos-cluster:8848"
        - name: NACOS_NAMESPACE
          valueFrom:
            secretKeyRef:
              name: nacos-secrets
              key: prod-namespace-id
```

---

## 案例3：灰度发布（配置灰度推送 + 服务权重调整）

### 9.3.1 配置灰度推送步骤

```bash
# 步骤1: 发布灰度配置（只推送给指定的 2 个实例）
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service.yml' \
  -d 'group=DEFAULT_GROUP' \
  -d 'tenant=prod-ns-id' \
  -d 'type=yaml' \
  -d 'betaIps=192.168.1.10,192.168.1.11' \
  -d 'content=order.max-amount=50000'

# 步骤2: 观察 192.168.1.10, 192.168.1.11 运行正常后，全量推送
# 删除灰度配置
curl -X DELETE 'http://127.0.0.1:8848/nacos/v1/cs/configs?beta=true' \
  -d 'dataId=order-service.yml' \
  -d 'group=DEFAULT_GROUP' \
  -d 'tenant=prod-ns-id'

# 发布正式配置（全量推送）
curl -X POST 'http://127.0.0.1:8848/nacos/v1/cs/configs' \
  -d 'dataId=order-service.yml' \
  -d 'group=DEFAULT_GROUP' \
  -d 'tenant=prod-ns-id' \
  -d 'type=yaml' \
  -d 'content=order.max-amount=50000'
```

### 9.3.2 服务权重灰度调整

```java
package com.example.admin.service;

import com.alibaba.nacos.api.naming.NamingService;
import com.alibaba.nacos.api.naming.pojo.Instance;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

/**
 * 服务权重管理（用于灰度发布时逐步调整流量）
 *
 * 灰度策略：
 * 阶段1: 新实例权重 0.1，旧实例权重 1.0（约 10% 流量到新实例）
 * 阶段2: 新实例权重 0.5，旧实例权重 1.0（约 33% 流量到新实例）
 * 阶段3: 新实例权重 1.0，旧实例权重 1.0（约 50% 流量到新实例）
 * 阶段4: 下线旧实例，权重设为 0，最终完全切换到新版本
 */
@Service
public class ServiceWeightManager {

    @Autowired
    private NamingService namingService;

    /**
     * 动态调整实例权重
     */
    public void setInstanceWeight(String serviceName, String ip, int port, double weight)
            throws Exception {
        // 获取实例列表
        java.util.List<Instance> instances = namingService.getAllInstances(serviceName);
        for (Instance instance : instances) {
            if (instance.getIp().equals(ip) && instance.getPort() == port) {
                instance.setWeight(weight);
                namingService.updateInstance(serviceName, instance);
                System.out.println("实例 " + ip + ":" + port + " 权重已更新为 " + weight);
                return;
            }
        }
        throw new RuntimeException("未找到实例: " + ip + ":" + port);
    }

    /**
     * 优雅下线（将权重设为 0，等待存量请求处理完后再停止服务）
     */
    public void gracefulShutdown(String serviceName, String ip, int port) throws Exception {
        setInstanceWeight(serviceName, ip, port, 0);
        System.out.println("实例 " + ip + ":" + port + " 已优雅下线，权重设为 0");
    }
}
```

---

## 案例4：配置中心集中管理数据库连接池参数（动态刷新）

### 9.4.1 需求背景

```text
需求场景：
  生产环境数据库压力激增，需要调整连接池大小
  但服务不能重启（会影响用户请求）

  使用 Nacos 配置中心 + @RefreshScope 实现：
  在 Nacos 控制台修改 maximum-pool-size，服务端自动生效
```

### 9.4.2 动态数据源配置

```java
package com.example.orderservice.config;

import com.zaxxer.hikari.HikariDataSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.cloud.context.environment.EnvironmentChangeEvent;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;
import java.util.Set;

@Component
public class DynamicDataSourceConfig {

    private static final Logger logger = LoggerFactory.getLogger(DynamicDataSourceConfig.class);

    private final HikariDataSource dataSource;
    private final DatasourceProperties properties;

    public DynamicDataSourceConfig(DatasourceProperties properties) {
        this.properties = properties;
        this.dataSource = createDataSource(properties);
    }

    @EventListener
    public void onEnvironmentChange(EnvironmentChangeEvent event) {
        Set<String> changedKeys = event.getKeys();

        if (changedKeys.contains("spring.datasource.hikari.maximum-pool-size")) {
            int newSize = properties.getHikari().getMaximumPoolSize();
            logger.info("动态调整最大连接池大小: {}", newSize);
            dataSource.setMaximumPoolSize(newSize);
        }

        if (changedKeys.contains("spring.datasource.hikari.minimum-idle")) {
            int newMinIdle = properties.getHikari().getMinimumIdle();
            logger.info("动态调整最小空闲连接数: {}", newMinIdle);
            dataSource.setMinimumIdle(newMinIdle);
        }

        if (changedKeys.contains("spring.datasource.hikari.connection-timeout")) {
            long newTimeout = properties.getHikari().getConnectionTimeout();
            logger.info("动态调整连接超时时间: {}ms", newTimeout);
            dataSource.setConnectionTimeout(newTimeout);
        }
    }

    private HikariDataSource createDataSource(DatasourceProperties props) {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl(props.getUrl());
        ds.setUsername(props.getUsername());
        ds.setPassword(props.getPassword());
        ds.setMaximumPoolSize(props.getHikari().getMaximumPoolSize());
        ds.setMinimumIdle(props.getHikari().getMinimumIdle());
        ds.setConnectionTimeout(props.getHikari().getConnectionTimeout());
        return ds;
    }
}
```

```yaml
# Nacos DataId: order-service.yml（可动态修改下列参数）
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/order_db
    username: order_user
    password: secret_password
    hikari:
      maximum-pool-size: 20      # 生产高峰可改为 50，无需重启
      minimum-idle: 5
      connection-timeout: 30000
```

```text
动态调整验证流程：

1. 初始状态: maximum-pool-size=20
   GET /actuator/metrics/hikaricp.connections.max -> 返回 20

2. 在 Nacos 控制台将 maximum-pool-size 改为 50

3. 约1秒后，服务端收到配置更新通知（长轮询返回）

4. DynamicDataSourceConfig.onEnvironmentChange() 被触发

5. dataSource.setMaximumPoolSize(50) 执行（HikariCP 动态扩容）

6. GET /actuator/metrics/hikaricp.connections.max -> 返回 50，无需重启！
```

---

---

# Part 10: 常见面试题 FAQ

> 精选 15+ 道 Nacos 高频面试题，附详细解析与对比图表。

---

## 10.1 Nacos 与 Eureka 的区别是什么？

**考察点**：注册中心选型、CAP 理论、功能对比

| 对比维度 | Nacos | Eureka |
|---------|-------|--------|
| 开发语言 | Java（阿里巴巴） | Java（Netflix） |
| 维护状态 | 持续维护（活跃） | 2.x 已停止维护 |
| 功能 | 注册中心 + 配置中心 | 仅注册中心 |
| 一致性模型 | AP（默认）/ CP 可切换 | AP |
| 健康检查 | 心跳 + 主动探测 | 心跳 |
| 实例类型 | 临时实例 + 持久实例 | 仅临时实例 |
| 负载均衡 | 内置支持权重/集群优先 | Ribbon（已停维护） |
| 配置管理 | 内置配置中心 | 无，需 Spring Config |
| 控制台 | 功能丰富的 Web UI | 基础 UI |
| 通信协议 | HTTP + gRPC（2.x） | HTTP |
| 集群支持 | 原生支持 Raft 集群 | 点对点同步 |
| Spring Cloud 支持 | spring-cloud-starter-alibaba-nacos | spring-cloud-starter-netflix-eureka |

**核心区别总结**：
1. **功能**：Nacos = 注册中心 + 配置中心，Eureka 仅注册中心
2. **一致性**：Nacos 支持 AP/CP 切换，Eureka 只支持 AP
3. **维护**：Eureka 2.x 停止维护，Nacos 持续迭代
4. **性能**：Nacos 2.x 用 gRPC 替代长轮询，性能提升约 10 倍

---

## 10.2 Nacos 临时实例和持久实例的区别？

**考察点**：实例类型、健康检查机制、使用场景

```
临时实例（ephemeral=true，默认）
┌─────────────────────────────────────────┐
│  • 客户端主动发送心跳（5s/次）            │
│  • 服务端 15s 未收到心跳 -> 标记不健康    │
│  • 服务端 30s 未收到心跳 -> 删除实例      │
│  • 适用：微服务、容器等弹性实例            │
│  • 一致性协议：Distro（AP）               │
└─────────────────────────────────────────┘

持久实例（ephemeral=false）
┌─────────────────────────────────────────┐
│  • 服务端主动探测（TCP/HTTP/MySQL 等）    │
│  • 探测失败 -> 标记不健康（不删除）       │
│  • 服务器重启后实例依然存在               │
│  • 适用：数据库、中间件、第三方服务        │
│  • 一致性协议：Raft（CP）                │
└─────────────────────────────────────────┘
```

**配置示例**：
```yaml
# application.yml
spring:
  cloud:
    nacos:
      discovery:
        ephemeral: false  # 注册为持久实例
```

**关键区别**：
- 临时实例：**客户端** 维持心跳，断开即删除
- 持久实例：**服务端** 主动探测，断开仅标记不健康，不删除

---

## 10.3 Nacos 如何实现配置的动态刷新？

**考察点**：@RefreshScope、长轮询、事件机制

**完整刷新链路**：

```
配置变更 -> Nacos Server 推送 -> Nacos Client 接收
    -> 更新本地缓存 -> 发布 RefreshEvent
    -> Spring Cloud Context 刷新 @RefreshScope Bean
    -> 旧 Bean 销毁 -> 新 Bean 重新创建（注入新配置值）
```

**关键注解**：
```java
@RestController
@RefreshScope  // 标记该 Bean 在配置刷新时重建
public class ConfigController {

    @Value("${app.feature.enabled:false}")
    private boolean featureEnabled;

    @GetMapping("/feature")
    public boolean getFeature() {
        return featureEnabled;  // 每次调用返回最新值
    }
}
```

**注意事项**：
1. `@ConfigurationProperties` 类不需要 `@RefreshScope`，天然支持刷新
2. `@Value` 注入的字段**必须**配合 `@RefreshScope` 才能刷新
3. 静态字段、构造函数注入不会被刷新
4. `@RefreshScope` Bean 在刷新时会有短暂的销毁重建过程，需注意并发安全

---

## 10.4 Nacos 配置中心的长轮询机制是如何工作的？

**考察点**：推拉模型、长轮询原理、超时机制

**工作原理**：

```
Client                          Server
  │                               │
  │── GET /listener (携带 MD5) ──>│
  │                               │  等待最多 29.5s
  │                               │  （服务端挂起请求）
  │                               │
  │  情况1：配置变更               │
  │<── 立即返回变更的 dataId ──────│
  │                               │
  │  情况2：29.5s 内无变更         │
  │<── 返回空响应 ─────────────────│
  │                               │
  │── 重新发起长轮询 ─────────────>│
```

**为什么是 29.5 秒而不是 30 秒**？
- 客户端超时时间设置为 30s
- 服务端提前 0.5s 返回，避免客户端超时报错
- 留出网络传输时间缓冲

**服务端实现细节**：
1. 收到请求后，先检查配置是否有变更（对比 MD5）
2. 若有变更，立即返回
3. 若无变更，将请求挂起，加入延迟队列（29.5s）
4. 配置变更时，立即触发挂起请求的响应

**优势**：
- 相比短轮询：减少无效请求，降低服务端压力
- 相比 WebSocket：兼容性好，HTTP 协议穿透性强
- 变更感知延迟：接近实时（毫秒级）

---

## 10.5 Nacos 集群如何保证数据一致性？

**考察点**：Raft 协议、数据分类、CP vs AP

**两种数据类型使用不同协议**：

```
服务注册数据（临时实例）          配置数据 + 持久实例
┌────────────────────┐          ┌────────────────────┐
│   Distro 协议       │          │    Raft 协议        │
│   AP 模型           │          │    CP 模型          │
│   最终一致性         │          │    强一致性          │
│   无中心化           │          │    Leader 写入      │
│   每节点负责一部分    │          │    多数派确认        │
└────────────────────┘          └────────────────────┘
```

**Raft 写入流程**：
```
Client -> Leader
Leader -> 写入本地日志
Leader -> 并行发给 Follower
Follower -> 确认接收
Leader -> 收到多数派(n/2+1)确认
Leader -> Commit，返回成功
Leader -> 通知 Follower Commit
```

**Distro 写入流程**：
```
Client -> 任意节点
节点 -> 写入本地
节点 -> 异步同步给其他节点
（最终所有节点数据一致）
```

---

## 10.6 Distro 协议是什么？与 Raft 有何区别？

**考察点**：分布式协议、一致性级别

**Distro 协议特点**：
- 阿里巴巴自研的 AP 协议
- **无中心化**：每个节点都可以处理写请求
- **数据分片**：每个节点负责一部分服务实例数据
- **异步同步**：写入后异步广播给其他节点
- **健壮性强**：节点故障不影响整体可用性

```
Distro 数据分片示意
┌──────────┐  ┌──────────┐  ┌──────────┐
│  Node A  │  │  Node B  │  │  Node C  │
│ 负责服务  │  │ 负责服务  │  │ 负责服务  │
│ A/B/C    │  │ D/E/F    │  │ G/H/I    │
│          │  │          │  │          │
│ 同步所有  │<─┤ 同步所有  │<─┤ 同步所有  │
└──────────┘  └──────────┘  └──────────┘
```

**Distro vs Raft 对比**：

| 维度 | Distro | Raft |
|------|--------|------|
| 一致性 | 最终一致（AP） | 强一致（CP） |
| 中心节点 | 无 | 有（Leader） |
| 写入节点 | 任意节点 | 只能写 Leader |
| 性能 | 高（无需多数派确认） | 相对较低 |
| 数据安全 | 可能丢失少量数据 | 不丢失已提交数据 |
| 适用场景 | 服务注册（允许短暂不一致） | 配置管理（要求强一致） |

---

## 10.7 为什么临时实例用 AP（Distro），持久实例用 CP（Raft）？

**考察点**：业务场景与 CAP 理论的结合

**临时实例选择 AP 的原因**：
1. **业务特性**：微服务实例频繁上下线，需要高可用的注册/注销
2. **容错性**：宁可有短暂的脏数据，也不能因为网络分区导致整个注册中心不可用
3. **自愈能力**：通过心跳机制，不健康的实例会被自动剔除
4. **高并发**：微服务规模大时，AP 模型写入性能更高

**持久实例选择 CP 的原因**：
1. **业务特性**：数据库、中间件等基础设施很少变更
2. **数据准确性**：这类服务的元数据必须准确，否则业务出错
3. **不频繁写入**：变更少，CP 的性能损耗可接受
4. **持久保存**：服务重启后元数据必须存在，需要强一致保证

**结论**：技术选型要结合业务场景，没有绝对的好坏，只有合适与否。

---

## 10.8 Nacos 2.x 相比 1.x 有哪些改进？

**考察点**：版本演进、gRPC、性能优化

**核心改进**：

| 维度 | Nacos 1.x | Nacos 2.x |
|------|-----------|-----------|
| 通信协议 | HTTP（短轮询/长轮询） | gRPC + HTTP |
| 连接方式 | 无状态 HTTP | 长连接（gRPC） |
| 推送延迟 | ~1s（长轮询等待） | 毫秒级（gRPC 推送） |
| 服务端压力 | 高（大量长轮询连接） | 低（长连接复用） |
| 心跳频率 | 5s/次 HTTP 请求 | gRPC 连接复用，无需额外心跳 |
| 性能提升 | 基准 | 连接数减少约 80%，TPS 提升约 10x |
| 端口 | 8848 | 8848（HTTP）+ 9848（gRPC）+ 9849（gRPC集群） |
| API 兼容性 | - | 兼容 1.x HTTP API |

**gRPC 改造带来的变化**：
```
Nacos 1.x:
Client ──(HTTP 长轮询)──> Server  每次轮询都是新的 HTTP 连接

Nacos 2.x:
Client ══(gRPC 长连接)══> Server  一个连接处理所有请求和推送
         <──推送────────
```

**升级注意事项**：
1. 需要开放额外端口：9848、9849
2. 客户端需升级到 2.x 版本的 SDK
3. Nginx 需配置 gRPC 代理（L4 而非 L7）
4. 数据库 schema 有变更，需执行升级 SQL

---

## 10.9 @RefreshScope 的工作原理是什么？

**考察点**：Spring Cloud 原理、Bean 生命周期

**原理深度解析**：

```
@RefreshScope 原理链路

1. 标注 @RefreshScope 的 Bean 被 Spring 包装为 Proxy（代理对象）
2. 实际 Bean 存储在 RefreshScope 的缓存中
3. 每次调用 Proxy 方法时，从缓存中获取实际 Bean

配置变更时：
4. Nacos 推送变更 -> 触发 RefreshEvent
5. RefreshScope.refreshAll() 清空缓存
6. 下次调用 Proxy 方法时，重新创建 Bean（注入新值）
```

**源码关键类**：
```java
// Spring Cloud Context
public class RefreshScope extends GenericScope {
    
    // 清空所有 @RefreshScope Bean 的缓存
    public void refreshAll() {
        super.destroy();  // 销毁所有缓存的 Bean
        this.context.publishEvent(new ScopeRefreshedEvent(this));
    }
}
```

**使用陷阱**：
```java
// 错误：@RefreshScope 不能与 @Configuration 同时使用
@Configuration
@RefreshScope  // 会导致子 Bean 刷新问题
public class DataSourceConfig { }

// 正确：在具体的 Bean 上使用
@Component
@RefreshScope
public class DynamicConfig {
    @Value("${pool.size:10}")
    private int poolSize;
}
```

---

## 10.10 Nacos 服务保护阈值是什么？

**考察点**：服务降级、雪崩防护

**定义**：服务保护阈值是一个 0~1 之间的浮点数，代表健康实例占总实例的最低比例。

**工作机制**：
```
假设保护阈值 = 0.5（50%）

正常情况：
  总实例: 10, 健康: 8 -> 健康比例 80% > 50%
  只返回健康实例（8个）给调用方

触发保护：
  总实例: 10, 健康: 4 -> 健康比例 40% < 50%
  返回所有实例（10个，包括不健康的）给调用方
  
目的：宁可让部分请求失败，也不让所有流量压垮仅剩的健康实例
```

**控制台配置**：
- 服务管理 -> 服务列表 -> 编辑 -> 保护阈值（0~1）

**什么时候配置**：
- 服务有大量实例时，避免少数健康实例被打垮
- 通常设置为 0.3 ~ 0.5
- 配合熔断器（Sentinel）使用效果更好

---

## 10.11 如何实现 Nacos 服务的同集群优先路由？

**考察点**：集群感知、自定义负载均衡

**背景**：同一服务部署在多个集群（如北京集群、上海集群），希望优先调用同集群的实例，减少跨机房延迟。

**实现步骤**：

**Step 1：配置服务所在集群**
```yaml
spring:
  cloud:
    nacos:
      discovery:
        cluster-name: BJ  # 当前服务所在集群
```

**Step 2：自定义负载均衡策略**
```java
@Bean
public ReactorLoadBalancer<ServiceInstance> clusterAwareLoadBalancer(
        Environment environment,
        LoadBalancerClientFactory factory) {
    String name = environment.getProperty(LoadBalancerClientFactory.PROPERTY_NAME);
    return new NacosClusterAwareLoadBalancer(
        factory.getLazyProvider(name, ServiceInstanceListSupplier.class),
        name,
        nacosDiscoveryProperties
    );
}

public class NacosClusterAwareLoadBalancer implements ReactorServiceInstanceLoadBalancer {

    @Override
    public Mono<Response<ServiceInstance>> choose(Request request) {
        return serviceInstanceListSupplierProvider.getIfAvailable()
            .get()
            .next()
            .map(instances -> {
                String localCluster = nacosDiscoveryProperties.getClusterName();
                
                // 过滤同集群实例
                List<ServiceInstance> sameCluster = instances.stream()
                    .filter(i -> localCluster.equals(
                        ((NacosServiceInstance) i).getMetadata().get("nacos.cluster")))
                    .collect(Collectors.toList());
                
                // 同集群有实例则优先，否则跨集群
                List<ServiceInstance> chosen = sameCluster.isEmpty() ? instances : sameCluster;
                
                // 随机选一个
                int index = ThreadLocalRandom.current().nextInt(chosen.size());
                return new DefaultResponse(chosen.get(index));
            });
    }
}
```

---

## 10.12 Nacos 的配置快照有什么作用？

**考察点**：容灾设计、离线可用性

**快照机制**：

```
正常情况：
Client ──(拉取配置)──> Nacos Server
                        │
                        v
                   写入本地快照
            ~/.nacos/config/snapshot/
            └── {namespace}/
                └── {group}/
                    └── {dataId}

Nacos Server 不可用时：
Client ──(拉取配置)──X  Server（连接失败）
        │
        v（降级）
   读取本地快照文件
   使用上次成功获取的配置继续运行
```

**快照文件位置**：
```bash
# Linux/Mac
~/.nacos/config/snapshot/{namespace}/{group}/{dataId}

# Windows
C:\Users\{user}\.nacos\config\snapshot\...
```

**作用**：
1. **容灾**：Nacos Server 故障时，应用仍可正常启动和运行
2. **快速启动**：应用启动时先用快照，再从 Server 同步最新配置
3. **减少依赖**：降低对注册中心的强依赖

**注意**：快照是上次成功的配置，不是最新配置，Server 恢复后会自动同步更新。

---

## 10.13 Nacos 集群脑裂如何预防和处理？

**考察点**：分布式高可用、网络分区

**脑裂场景**：
```
正常集群（3节点）：
Node A ←──→ Node B ←──→ Node C

网络分区后：
[Node A] ←─X─→ [Node B ←──→ Node C]
  少数派              多数派
```

**Raft 协议的天然防护**：
- 少数派（Node A）无法选出新 Leader（需要多数派 n/2+1 票）
- 少数派停止对外服务（拒绝写入）
- 多数派继续正常工作
- 网络恢复后，少数派自动同步多数派数据

**预防措施**：
1. **奇数节点部署**：3、5、7 节点，避免平票
2. **网络冗余**：多网卡、多网段，降低网络分区概率
3. **节点跨机架/AZ 部署**：避免单点网络故障影响多数节点

**排查脑裂**：
```bash
# 查看各节点 Raft 状态
curl http://nacos-node:8848/nacos/v1/ns/raft/state

# 输出中查看 leader 信息
# 如果不同节点显示不同 leader，可能发生脑裂
```

**处理步骤**：
1. 确认网络分区已恢复
2. 重启少数派节点
3. 验证集群状态恢复正常（只有一个 Leader）
4. 检查数据一致性

---

## 10.14 shared-configs 和 extension-configs 的区别？

**考察点**：多配置文件管理、配置共享

**两者对比**：

| 维度 | shared-configs | extension-configs |
|------|---------------|-------------------|
| 用途 | 跨服务共享配置 | 单服务扩展配置 |
| 命名空间 | 默认 namespace | 当前 namespace |
| Group | 默认 DEFAULT_GROUP | 可自定义 Group |
| 优先级 | 最低 | 中等（高于 shared） |
| 适用场景 | 公共配置（如数据库公共参数） | 分模块配置拆分 |

**优先级顺序（从高到低）**：
```
1. {spring.application.name}-{profile}.yaml  （主配置，最高优先级）
2. {spring.application.name}.yaml
3. extension-configs（按 index 从大到小）
4. shared-configs（按 index 从大到小）
```

**配置示例**：
```yaml
spring:
  cloud:
    nacos:
      config:
        # 主配置：order-service-prod.yaml
        file-extension: yaml
        
        # 扩展配置（同 namespace，高优先级）
        extension-configs:
          - data-id: order-datasource.yaml
            group: ORDER_GROUP
            refresh: true
          - data-id: order-redis.yaml
            group: ORDER_GROUP
            refresh: true
            
        # 共享配置（跨服务，低优先级）
        shared-configs:
          - data-id: common-log.yaml
            group: COMMON_GROUP
            refresh: true
          - data-id: common-sentinel.yaml
            group: COMMON_GROUP
            refresh: false
```

**最佳实践**：
- `shared-configs`：存放所有微服务通用的配置（日志格式、公共 Redis 地址等）
- `extension-configs`：存放当前服务的模块化配置（数据源、MQ 配置等）
- 主配置文件：存放服务核心业务配置和环境差异配置

---

## 10.15 Nacos 与 Spring Cloud Gateway 如何集成？

**考察点**：API 网关、服务发现、路由配置

**集成架构**：
```
外部请求
    │
    v
Spring Cloud Gateway（集成 Nacos）
    │  从 Nacos 获取服务列表
    │  动态路由
    ├──> user-service（Nacos 注册）
    ├──> order-service（Nacos 注册）
    └──> product-service（Nacos 注册）
```

**Maven 依赖**：
```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

**Gateway 配置（基于服务发现自动路由）**：
```yaml
spring:
  application:
    name: api-gateway
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848
        namespace: production
    gateway:
      discovery:
        locator:
          enabled: true            # 开启基于服务发现的自动路由
          lower-case-service-id: true  # 服务名转小写
      routes:
        # 自定义路由规则
        - id: order-service-route
          uri: lb://order-service  # lb:// 表示负载均衡
          predicates:
            - Path=/order/**
          filters:
            - StripPrefix=1        # 去掉路径前缀 /order
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 100
                redis-rate-limiter.burstCapacity: 200
                
        - id: user-service-route
          uri: lb://user-service
          predicates:
            - Path=/user/**
            - Header=X-Request-Version, v2
          filters:
            - StripPrefix=1
            - AddRequestHeader=X-Gateway-Source, api-gateway
```

**启动类**：
```java
@SpringBootApplication
@EnableDiscoveryClient
public class GatewayApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayApplication.class, args);
    }
}
```

**动态路由（从 Nacos 配置中心加载路由）**：
```java
@Component
@RefreshScope
public class DynamicRouteService implements ApplicationEventPublisherAware {

    @Autowired
    private RouteDefinitionWriter routeDefinitionWriter;
    
    private ApplicationEventPublisher publisher;

    // 从 Nacos 配置中心读取路由配置，动态更新
    @NacosConfigListener(dataId = "gateway-routes.json", groupId = "GATEWAY_GROUP")
    public void onRoutesChange(String routeJson) {
        List<RouteDefinition> routes = JSON.parseArray(routeJson, RouteDefinition.class);
        routes.forEach(route -> {
            routeDefinitionWriter.save(Mono.just(route)).subscribe();
        });
        publisher.publishEvent(new RefreshRoutesEvent(this));
    }

    @Override
    public void setApplicationEventPublisher(ApplicationEventPublisher publisher) {
        this.publisher = publisher;
    }
}
```

---

## 10.16 Nacos 注册中心的服务订阅原理？

**考察点**：订阅机制、推送 vs 拉取

**订阅流程**：
```
服务消费者启动
    │
    v
向 Nacos Server 订阅 service-A
    │
    v
Server 返回当前 service-A 的实例列表
    │
    v
Client 本地缓存实例列表
    │
service-A 实例变更（上线/下线）
    │
    v
Server 主动推送变更给订阅者（Nacos 2.x gRPC 推送）
    │
    v
Client 更新本地缓存
    │
    v（兜底机制）
Client 每 10s 定时轮询一次（防止推送丢失）
```

**本地缓存 + 推送双保险**：
- **主推送**：Server 变更时主动推送（实时性好）
- **兜底轮询**：定期拉取全量数据（防止推送遗漏）

这种"推+拉"结合的方式保证了服务发现的实时性和可靠性。

---

## 10.17 如何在不重启应用的情况下切换 Nacos 配置的命名空间？

**结论：不支持运行时切换命名空间。**

**原因**：
- 命名空间（namespace）是客户端启动时连接 Nacos 的基本参数
- 它决定了客户端与哪个命名空间建立连接
- 运行时无法动态切换（类似于切换数据库连接地址）

**正确的多环境方案**：
```
方案1：不同环境部署不同应用实例，每个实例配置对应的 namespace

方案2：使用 Spring Profile + 不同配置文件
  application-dev.yaml  -> namespace: dev-ns-id
  application-prod.yaml -> namespace: prod-ns-id
  启动时指定: --spring.profiles.active=prod

方案3：同一 namespace 下用 Group 区分环境（不推荐，污染命名空间）
```

---

## 小结：Nacos 核心知识图谱

```
                        Nacos 核心知识体系
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
     注册中心              配置中心              集群
          │                   │                   │
  ┌───────┴───────┐   ┌───────┴───────┐   ┌───────┴───────┐
  │ 临时实例(AP)   │   │ 三元组标识     │   │ Raft(CP)      │
  │ 持久实例(CP)   │   │ 长轮询29.5s   │   │ Distro(AP)    │
  │ 心跳/主动探测  │   │ MD5比对       │   │ 3节点奇数部署  │
  │ 服务保护阈值   │   │ 本地快照      │   │ Derby/MySQL   │
  │ 服务订阅推拉   │   │ @RefreshScope │   │ gRPC长连接    │
  └───────────────┘   │ 灰度发布      │   └───────────────┘
                      │ 历史回滚      │
                      └───────────────┘
```

---

*文档完结。如需进一步了解某个主题，建议参考官方文档：https://nacos.io/zh-cn/docs/what-is-nacos.html*
