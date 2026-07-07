# ZooKeeper 详解 - 从零到精通

> 版本：ZooKeeper 3.8.x | Curator 5.x | Spring Boot 3.x  
> 日期：2026-07-07

---

## 目录

1. [ZooKeeper 整体架构](#1-zookeeper-整体架构)
2. [ZNode 深度解析](#2-znode-深度解析)
3. [Watch 机制深度解析](#3-watch-机制深度解析)
4. [ZAB 协议深度解析](#4-zab-协议深度解析)
5. [会话机制](#5-会话机制)
6. [ZooKeeper 客户端](#6-zookeeper-客户端)
7. [分布式应用场景实现](#7-分布式应用场景实现)
8. [集群部署与运维](#8-集群部署与运维)
9. [Spring Boot 集成](#9-spring-boot-集成)
10. [性能与调优](#10-性能与调优)
11. [常见面试题 FAQ](#11-常见面试题-faq)

---

## 1. ZooKeeper 整体架构

### 1.1 ZooKeeper 是什么

ZooKeeper 是 Apache 基金会下的开源分布式协调服务框架，最初由 Yahoo 研究院开发，于 2010 年成为 Apache 顶级项目。它为分布式应用提供一致性服务，包括：配置管理、命名服务、分布式同步、组服务等。

```
+----------------------------------------------------------+
|                  ZooKeeper 定位示意                        |
|                                                          |
|  分布式应用集群                                            |
|  +--------+  +--------+  +--------+  +--------+         |
|  | App-1  |  | App-2  |  | App-3  |  | App-4  |         |
|  +---+----+  +---+----+  +---+----+  +---+----+         |
|      |            |            |            |            |
|      +------------+------------+------------+            |
|                          |                               |
|              +-----------+-----------+                   |
|              |   ZooKeeper Cluster   |                   |
|              |  +------+ +------+    |                   |
|              |  |Leader| |Follow|    |                   |
|              |  +------+ +------+    |                   |
|              |       +------+        |                   |
|              |       |Follow|        |                   |
|              |       +------+        |                   |
|              +-----------------------+                   |
|                                                          |
|  提供：配置管理 | 命名服务 | 分布式锁 | Leader选举         |
+----------------------------------------------------------+
```

**核心特性：**

- **顺序一致性**：来自同一客户端的更新按发送顺序应用
- **原子性**：更新要么成功，要么失败，没有部分更新
- **单一系统镜像**：客户端无论连接到哪个服务器，都看到相同的视图
- **可靠性**：一旦更新被应用，将持久化到下一次更新
- **及时性**：客户端在特定时间范围内保证看到最新数据

### 1.2 核心应用场景

| 场景 | 描述 | 典型实现 |
|------|------|---------|
| 配置管理 | 集中存储和分发配置 | 监听配置节点变化 |
| 命名服务 | 统一的资源名称解析 | 创建持久节点注册 |
| 分布式锁 | 协调多节点互斥访问 | 临时有序节点 |
| Leader 选举 | 选出集群主节点 | 临时有序节点竞争 |
| 服务注册与发现 | 动态感知服务上下线 | 临时节点 + Watch |
| 分布式队列 | 有序任务分发 | 持久有序节点 |
| 分布式屏障 | 多节点同步点 | 临时节点计数 |

### 1.3 ZooKeeper vs etcd vs Consul

```
+-------------+----------------+----------------+----------------+
| 特性         | ZooKeeper      | etcd           | Consul         |
+-------------+----------------+----------------+----------------+
| 一致性协议   | ZAB            | Raft           | Raft           |
| 数据模型     | 层级树(ZNode)  | KV平铺          | KV + 服务目录  |
| Watch机制    | 一次性Watch    | 持续Watch       | 长轮询/Watch   |
| 语言         | Java           | Go             | Go             |
| 性能(读)     | 高(本地读)     | 高              | 高             |
| 性能(写)     | 中             | 高              | 中             |
| 节点大小限制 | 1MB            | 1.5MB           | 无硬限制       |
| 健康检查     | 会话心跳       | Lease机制       | 内置多种检查   |
| ACL          | 精细           | 基于RBAC        | Token/Policy   |
| 生态成熟度   | 非常成熟       | 较成熟(k8s依赖) | 成熟           |
| 推荐场景     | 大数据生态     | 云原生/k8s      | 微服务治理     |
+-------------+----------------+----------------+----------------+
```

**选型建议：**

- Hadoop、HBase、Kafka 生态 → ZooKeeper
- Kubernetes、云原生 → etcd
- 微服务注册发现 + 健康检查 → Consul

### 1.4 集群架构

```
                    +------------------+
                    |   ZooKeeper      |
                    |   Cluster        |
                    +------------------+
                           |
          +----------------+----------------+
          |                |                |
   +------+------+  +------+------+  +------+------+
   |   Leader    |  |  Follower   |  |  Follower   |
   |  (读+写)    |  |  (读+选举)  |  |  (读+选举)  |
   +------+------+  +------+------+  +------+------+
          |                |                |
          +----Observer(可选，只读不投票)-----+

   写请求：Client -> 任意节点 -> 转发给Leader -> Leader广播
           -> 多数Follower确认 -> 提交 -> 响应客户端
   读请求：Client -> 任意节点 -> 直接返回（本地读）
```

**节点角色说明：**

| 角色 | 职责 | 参与投票 | 处理写 |
|------|------|---------|-------|
| Leader | 事务请求处理、广播提案 | 是 | 是(直接) |
| Follower | 处理非事务请求、参与选举 | 是 | 否(转发) |
| Observer | 只提供读服务、扩展读能力 | 否 | 否(转发) |

### 1.5 数据模型 - ZNode 树

```
/ (根节点)
├── /zookeeper                    (系统内置)
│   ├── /zookeeper/quota
│   └── /zookeeper/config
├── /services                     (用户创建)
│   ├── /services/user-service
│   │   ├── /services/user-service/192.168.1.1:8080
│   │   └── /services/user-service/192.168.1.2:8080
│   └── /services/order-service
│       └── /services/order-service/192.168.1.3:8080
├── /config
│   ├── /config/database
│   └── /config/redis
└── /locks
    └── /locks/order-lock
        ├── /locks/order-lock/lock-0000000001  (临时有序)
        └── /locks/order-lock/lock-0000000002  (临时有序)
```

每个 ZNode 可以存储数据（默认上限 1MB），并可以有子节点，形成类似文件系统的树形结构。

---

## 2. ZNode 深度解析

### 2.1 ZNode 类型

ZooKeeper 共有 7 种 ZNode 类型（3.6+ 新增持久递归 Watch 类型）：

```
ZNode 类型
├── 持久节点 (PERSISTENT)
│   └── 创建后永久存在，直到显式删除
├── 持久有序节点 (PERSISTENT_SEQUENTIAL)
│   └── 节点名称自动追加单调递增序号（10位，如 _0000000001）
├── 临时节点 (EPHEMERAL)
│   └── 会话结束后自动删除，不能有子节点
├── 临时有序节点 (EPHEMERAL_SEQUENTIAL)
│   └── 结合临时 + 有序特性
├── Container 节点 (CONTAINER) [3.5.3+]
│   └── 当最后一个子节点被删除后，服务器会自动删除该容器节点
├── TTL 节点 (PERSISTENT_WITH_TTL) [3.6+]
│   └── 超过 TTL 且无子节点时自动删除
└── TTL 有序节点 (PERSISTENT_SEQUENTIAL_WITH_TTL) [3.6+]
    └── 结合 TTL + 有序特性
```

**各类型使用场景对比：**

| 类型 | 持久性 | 有序性 | 典型用途 |
|------|--------|--------|---------|
| PERSISTENT | 持久 | 无 | 配置存储、命名空间 |
| PERSISTENT_SEQUENTIAL | 持久 | 有 | 分布式队列 |
| EPHEMERAL | 临时 | 无 | 服务注册、分布式锁 |
| EPHEMERAL_SEQUENTIAL | 临时 | 有 | 公平锁、Leader选举 |
| CONTAINER | 持久(自清理) | 无 | 父节点容器 |
| TTL | 持久(超时) | 无 | 缓存类数据 |

### 2.2 ZNode 数据结构 - Stat

每个 ZNode 包含两部分：数据内容（byte[]）和元数据（Stat）。

```java
// ZooKeeper Stat 结构完整字段
public class Stat {
    long czxid;      // 创建该节点的事务ID
    long mzxid;      // 最后一次修改该节点数据的事务ID
    long ctime;      // 节点创建时间（毫秒时间戳）
    long mtime;      // 节点最后修改时间（毫秒时间戳）
    int  version;    // 数据版本号（每次setData +1）
    int  cversion;   // 子节点版本号（每次子节点变化 +1）
    int  aversion;   // ACL版本号（每次setACL +1）
    long ephemeralOwner; // 临时节点的会话ID（持久节点为0）
    int  dataLength; // 数据长度（字节）
    int  numChildren;// 直接子节点数量
    long pzxid;      // 最后一次修改子节点列表的事务ID
}
```

**通过 zkCli 查看 Stat：**

```bash
[zk: localhost:2181(CONNECTED) 0] stat /services/user-service
cZxid = 0x100000003
ctime = Mon Jul 07 10:00:00 CST 2026
mZxid = 0x100000005
mtime = Mon Jul 07 10:05:00 CST 2026
pZxid = 0x100000008
cversion = 2
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 32
numChildren = 2
```

### 2.3 ZNode 大小与路径限制

```
+-------------------------------------------+
|          ZNode 限制说明                     |
+-------------------------------------------+
| 数据大小   | 默认 1MB（jute.maxbuffer）     |
| 路径长度   | 不超过 1MB（同上）              |
| 路径规则   | 必须以 / 开头，绝对路径          |
|           | 不能以 / 结尾（根节点除外）      |
|           | 不能包含 null 字符               |
|           | 不能包含控制字符 (0x00-0x1F)     |
|           | 不能使用 . 或 .. 作为路径组件    |
|           | 不能包含 /zookeeper（保留）      |
+-------------------------------------------+
```

**版本控制与乐观锁：**

```java
// 乐观锁写入：version=-1 表示不校验版本，直接覆盖
zk.setData("/config/db", newData, -1);

// 版本校验写入：若服务端version != 期望version，抛出BadVersionException
zk.setData("/config/db", newData, expectedVersion);

// 条件删除（同样支持版本检查）
zk.delete("/config/db", expectedVersion);
```

### 2.4 ACL 权限控制

```
ACL = scheme:id:permissions

schemes（权限模式）：
  world   : 任何人         world:anyone
  auth    : 已认证的用户   auth::rw
  digest  : 用户名+密码    digest:user:Base64(SHA1(user:pass))
  ip      : IP地址         ip:192.168.1.0/24
  x509    : 证书           x509:...

permissions（权限位）：
  READ    (r) : 读取数据和子节点列表
  WRITE   (w) : 写入数据
  CREATE  (c) : 创建子节点
  DELETE  (d) : 删除子节点
  ADMIN   (a) : 设置ACL
```

```java
// Java 代码示例：创建带 ACL 的节点
List<ACL> acls = new ArrayList<>();
acls.add(new ACL(ZooDefs.Perms.READ | ZooDefs.Perms.WRITE,
                  new Id("digest", DigestAuthenticationProvider.generateDigest("user:password"))));
zk.create("/secure-node", "data".getBytes(), acls, CreateMode.PERSISTENT);

// 使用 Curator 设置 ACL
client.create()
      .withACL(Arrays.asList(new ACL(ZooDefs.Perms.ALL, ZooDefs.Ids.ANYONE_ID_UNSAFE)))
      .forPath("/node", "data".getBytes());
```

---

## 3. Watch 机制深度解析

### 3.1 Watch 工作原理

Watch 是 ZooKeeper 提供的事件通知机制，允许客户端在 ZNode 发生变化时收到异步通知。

```
Watch 生命周期（默认一次性触发）：

  客户端                          ZooKeeper 服务端
    |                                    |
    |  getData("/node", watcher=true)    |
    |  --------------------------------> |  注册Watch
    |                                    |  [存储: session -> watch set]
    |  <-- 返回数据 --                   |
    |                                    |
    |  (其他客户端) setData("/node", ..) |
    |                                    |  触发Watch
    |  <-- WatchedEvent 通知 --          |  [异步推送]
    |                                    |
    |  (Watch自动移除，需重新注册)        |
    |                                    |
    |  getData("/node", watcher=true)    |
    |  --------------------------------> |  重新注册Watch
```

**Watch 保证：**

1. Watch 通知在数据变更通知的顺序上是有保证的
2. 客户端在看到 Watch 事件之前，不会看到已设置 Watch 后的数据变更
3. Watch 事件的顺序与 ZooKeeper 服务端的更新顺序一致

### 3.2 Watch 事件类型

```java
// WatchedEvent 包含三个字段
public class WatchedEvent {
    KeeperState state;   // 连接状态
    EventType   type;    // 事件类型
    String      path;    // 涉及的节点路径
}

// KeeperState 连接状态
enum KeeperState {
    Disconnected,        // 客户端与服务端断开
    SyncConnected,       // 客户端已连接到服务端
    AuthFailed,          // 认证失败
    ConnectedReadOnly,   // 以只读模式连接
    Expired              // 会话过期
}

// EventType 事件类型
enum EventType {
    None,                // 连接状态变化（无具体节点）
    NodeCreated,         // 节点被创建
    NodeDeleted,         // 节点被删除
    NodeDataChanged,     // 节点数据被修改
    NodeChildrenChanged, // 子节点列表变化（增/删子节点）
    DataWatchRemoved,    // 持久Watch被移除（3.6+）
    ChildWatchRemoved,   // 持久子Watch被移除（3.6+）
    PersistentWatchRemoved // 持久递归Watch被移除（3.6+）
}
```

**Watch 注册 API 对应的事件：**

| 注册方式 | NodeCreated | NodeDeleted | NodeDataChanged | NodeChildrenChanged |
|---------|-------------|-------------|-----------------|---------------------|
| exists(path, w) | ✓ | ✓ | ✗ | ✗ |
| getData(path, w) | ✗ | ✓ | ✓ | ✗ |
| getChildren(path, w) | ✗ | ✓ | ✗ | ✓ |

### 3.3 Watch 事件处理代码示例

```java
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

public class WatchExample implements Watcher {

    private ZooKeeper zk;
    private static final String WATCH_PATH = "/config/app";

    public WatchExample() throws Exception {
        this.zk = new ZooKeeper("localhost:2181", 5000, this);
    }

    @Override
    public void process(WatchedEvent event) {
        System.out.println("收到事件: " + event);
        if (event.getType() == Event.EventType.None) {
            handleConnectionChange(event.getState());
        } else {
            handleNodeEvent(event);
        }
    }

    private void handleConnectionChange(Event.KeeperState state) {
        switch (state) {
            case SyncConnected:
                System.out.println("已连接到ZooKeeper"); break;
            case Disconnected:
                System.out.println("与ZooKeeper断开，等待重连..."); break;
            case Expired:
                System.out.println("会话过期，需要重新创建ZooKeeper实例");
                reconnect(); break;
        }
    }

    private void handleNodeEvent(WatchedEvent event) {
        String path = event.getPath();
        try {
            switch (event.getType()) {
                case NodeCreated:
                    System.out.println("节点创建: " + path);
                    zk.exists(path, this); // 重新注册Watch
                    break;
                case NodeDataChanged:
                    Stat stat = new Stat();
                    byte[] data = zk.getData(path, this, stat);
                    System.out.println("节点数据变化: " + new String(data));
                    break;
                case NodeDeleted:
                    System.out.println("节点删除: " + path);
                    zk.exists(path, this);
                    break;
                case NodeChildrenChanged:
                    System.out.println("子节点变化: " + path);
                    zk.getChildren(path, this);
                    break;
            }
        } catch (Exception e) { e.printStackTrace(); }
    }

    private void reconnect() {
        try {
            if (zk != null) zk.close();
            this.zk = new ZooKeeper("localhost:2181", 5000, this);
        } catch (Exception e) { e.printStackTrace(); }
    }
}
```

### 3.4 持久化 Watch（3.6+ 新特性）

ZooKeeper 3.6 引入了持久化 Watch，解决了一次性 Watch 需要反复注册的问题：

```java
// 持久Watch：触发后不自动移除，持续监听
zk.addWatch("/config", myWatcher, AddWatchMode.PERSISTENT);

// 持久递归Watch：监听节点及其所有子节点的变化
zk.addWatch("/config", myWatcher, AddWatchMode.PERSISTENT_RECURSIVE);

// Curator 使用 CuratorCache 实现持久监听（推荐）
CuratorCache cache = CuratorCache.build(client, "/config");
cache.listenable().addListener((type, oldData, newData) -> {
    System.out.println("事件类型: " + type);
    System.out.println("新数据: " + (newData != null ? new String(newData.getData()) : "null"));
});
cache.start();
```

### 3.5 Watch 重连与会话过期行为

```
客户端网络断开场景：

时间轴:
  t1: 客户端连接，注册Watch
  t2: 网络断开 (Disconnected 事件)
  t3: 其他客户端修改了被监听节点
  t4: 网络恢复，客户端重连
       如果 t4-t1 < sessionTimeout：
           会话未过期，Watch仍然有效
           注意：t3 时刻的变更事件可能丢失！
           需要在重连后主动读取数据进行对比
       如果 t4-t1 > sessionTimeout：
           Expired 事件，会话失效
           所有Watch全部失效
           所有临时节点被删除
           必须重新创建ZooKeeper实例并重新注册Watch
```

---

## 4. ZAB 协议深度解析

### 4.1 ZAB 协议介绍

ZAB（ZooKeeper Atomic Broadcast，ZooKeeper 原子广播协议）是 ZooKeeper 专门设计的崩溃恢复原子消息广播算法，是保证 ZooKeeper 数据一致性的核心。

```
ZAB 协议两种模式：

+------------------+        崩溃/启动         +------------------+
|                  | <----------------------- |                  |
|  消息广播模式     |                          |  崩溃恢复模式     |
| (Message Bcast)  | -----------------------> | (Crash Recovery) |
|                  |  选出Leader且             |                  |
|  正常工作状态     |  多数节点同步完成         |  Leader不可用     |
+------------------+                          +------------------+
```

### 4.2 ZAB vs Paxos vs Raft

```
+----------------+------------------+------------------+------------------+
| 特性           | ZAB              | Paxos            | Raft             |
+----------------+------------------+------------------+------------------+
| 设计目标       | ZooKeeper定制     | 通用一致性        | 易理解的一致性    |
| 角色           | Leader/Follower/ | Proposer/Acceptor| Leader/Follower  |
|               | Observer         | /Learner         | /Candidate       |
| Leader产生     | 基于epoch+zxid   | 无固定Leader      | 随机超时选举      |
| 日志连续性     | 保证顺序连续      | 不保证            | 保证顺序连续      |
| 读一致性       | 本地读（非强一致）| 线性一致          | 线性一致          |
| 写吞吐         | 中（串行广播）    | 低               | 中高             |
| 使用场景       | ZooKeeper        | Google Chubby    | etcd/TiKV等      |
+----------------+------------------+------------------+------------------+
```

### 4.3 ZXID 结构

ZXID（ZooKeeper Transaction ID）是一个 64 位整数，由两部分组成：

```
ZXID（64位）：
+----------------------------------+------------------------------+
|    高32位：epoch（纪元）           |   低32位：counter（计数器）   |
+----------------------------------+------------------------------+
  每次Leader选举后递增                 每次事务在当前epoch内递增
  标识Leader代际                       从0开始

例如：
  0x100000003
  = epoch: 0x00000001 (1)
  + counter: 0x00000003 (3)
  表示第1届Leader的第3条事务
```

**ZXID 的作用：**

1. 全局唯一事务排序
2. 选举时判断哪个节点数据最新（zxid 最大的优先成为 Leader）
3. 数据同步时判断差异

### 4.4 Leader 选举流程（Fast Leader Election）

```
初始状态（集群启动或Leader崩溃）：
每个节点投票给自己：Vote(myid, zxid, epoch)

选举轮次（以3节点集群为例）：
+----------+         +----------+         +----------+
| Server 1 |         | Server 2 |         | Server 3 |
| (LOOKING)|         | (LOOKING)|         | (LOOKING)|
+----------+         +----------+         +----------+
     |                    |                    |
     | 广播投票(1,z1,e1)  |                    |
     |------------------->|------------------->|
     |                    | 广播投票(2,z2,e1)  |
     |<-------------------|------------------->|
     |                    |  广播投票(3,z3,e1) |
     |<-------------------|--------------------|  

比较规则（优先级从高到低）：
  1. epoch 最大者优先
  2. epoch 相同时，zxid 最大者优先
  3. zxid 相同时，myid 最大者优先

假设 Server3 的 zxid 最大：
  Server1、Server2 更新投票给 Server3
  Server3 获得 2/3 多数票 → 成为 Leader
  其他节点状态变为 FOLLOWING
```

```java
// 节点状态枚举
public enum ServerState {
    LOOKING,    // 正在选举
    FOLLOWING,  // 已确定Leader，作为Follower
    LEADING,    // 自己是Leader
    OBSERVING   // Observer模式
}
```

### 4.5 消息广播流程（两阶段提交）

```
写请求广播流程：

客户端           Leader            Follower1      Follower2
  |                |                   |               |
  | 写请求         |                   |               |
  |--------------> |                   |               |
  |                | 分配ZXID          |               |
  |                | 生成Proposal       |               |
  |                |                   |               |
  |                | 发送PROPOSAL       |               |
  |                |------------------>|               |
  |                |------------------------------>|   |
  |                |   写入本地事务日志              |   |
  |                |        ACK        |               |
  |                |<------------------|               |
  |                |                            ACK    |
  |                |<------------------------------|   |
  |                |                   |               |
  |                | 收到多数ACK → COMMIT              |
  |                |------------------>|               |
  |                |------------------------------>|   |
  |  返回成功       |                   |               |
  |<-------------- |                   |               |

关键点：
  - Leader 只需收到 n/2+1 个 ACK 即可提交
  - Follower 可能在 COMMIT 到达前短暂有数据不一致（最终一致）
```

### 4.6 数据同步（崩溃恢复后）

新 Leader 选出后，需要将自己的数据同步给 Followers：

```
数据同步场景：

场景1：DIFF 同步（Follower只缺少少量事务）
  Leader：[zxid1, zxid2, zxid3, zxid4, zxid5]
  Follower：[zxid1, zxid2, zxid3]
  → Leader 发送 DIFF(zxid4, zxid5) 给 Follower

场景2：SNAP 同步（Follower落后太多或数据差异过大）
  → Leader 发送完整快照给 Follower

场景3：TRUNC 截断（Follower有多余的未提交事务）
  Follower 曾是旧 Leader，有未广播的 Proposal
  → 新 Leader 发送 TRUNC 让 Follower 回滚

同步完成后，过半 Follower 同步完毕，Leader 进入广播模式
```

### 4.7 脑裂预防

```
脑裂场景：网络分区导致两个区域都认为自己有 Leader

+------------------+     网络分区     +------------------+
| 分区A（3节点）    |  X X X X X X X  | 分区B（2节点）    |
| Leader(旧)       |                  | Leader(新选出)   |
| Follower1        |                  | Follower3        |
| Follower2        |                  |                  |
+------------------+                  +------------------+

ZAB 的预防机制（Quorum 过半机制）：
  - 5节点集群，过半 = 3节点
  - 分区A有3节点：可以形成Quorum，继续服务
  - 分区B有2节点：无法形成Quorum，无法提交事务

Epoch 机制防止旧 Leader 误操作：
  - 新 Leader 的 epoch 一定比旧 Leader 大
  - Follower 只接受当前 epoch 或更高 epoch 的消息
  - 旧 Leader 发出的 PROPOSAL 会被 Follower 拒绝
```

---

## 5. 会话机制

### 5.1 Session 创建过程

```
客户端                           ZooKeeper 服务端
  |                                    |
  | ConnectRequest                     |
  | (lastZxidSeen, timeout, sessionId) |
  |----------------------------------->|
  |                                    | 校验 sessionId
  |                                    | 分配会话票据
  |                                    | 设置 sessionTimeout
  | ConnectResponse                    |
  | (timeout, sessionId, password)     |
  |<-----------------------------------|
  |                                    |
  | [会话建立，可以正常通信]            |
```

**会话参数：**

| 参数 | 说明 | 默认值 |
|------|------|--------|
| sessionTimeout | 客户端请求的超时时间 | 客户端配置 |
| negotiatedTimeout | 服务端协商后的实际超时 | min(max, max(min, requested)) |
| minSessionTimeout | 服务端允许的最小超时 | 2 * tickTime |
| maxSessionTimeout | 服务端允许的最大超时 | 20 * tickTime |

### 5.2 心跳机制

```
心跳交互时序：

客户端                        服务端
  |                             |
  | PING (每 sessionTimeout/3)  |
  |---------------------------->|
  |                             | 更新会话最后活跃时间
  | PONG                        |
  |<----------------------------|
  |                             |
  | ... (持续心跳) ...           |
  |                             |
  | [网络异常，心跳失败]          |
  |                             | 等待 sessionTimeout
  |                             | 超时 → 会话过期
  |                             | → 删除所有临时节点
  |                             | → 触发相关Watch
```

### 5.3 临时节点与会话的关系

```java
public class EphemeralNodeDemo {

    private static final String ZK_ADDRESS = "localhost:2181";
    private static final int SESSION_TIMEOUT = 10000;

    public static void main(String[] args) throws Exception {
        ZooKeeper zk = new ZooKeeper(ZK_ADDRESS, SESSION_TIMEOUT, event -> {
            System.out.println("连接事件: " + event.getState());
        });
        Thread.sleep(1000);

        // 创建临时有序节点（服务注册）
        String servicePath = "/services/user-service";
        String instanceData = "{\"host\":\"192.168.1.1\",\"port\":8080}";
        String createdPath = zk.create(
            servicePath + "/instance-",
            instanceData.getBytes(),
            ZooDefs.Ids.OPEN_ACL_UNSAFE,
            CreateMode.EPHEMERAL_SEQUENTIAL
        );
        System.out.println("创建临时节点: " + createdPath);

        Thread.sleep(5000);

        // 会话关闭后，临时节点自动删除
        zk.close();
        System.out.println("会话关闭，临时节点将被自动删除");
    }
}
```

### 5.4 会话重连 vs 会话过期

```
情况1：重连成功（断开时间 < sessionTimeout）
  客户端断开 -> [DISCONNECTED 事件]
  -> 网络恢复 -> 自动重连到任意ZK节点
  -> [CONNECTED 事件]
  -> Watch 仍然有效，临时节点仍然存在

情况2：会话过期（断开时间 > sessionTimeout）
  客户端断开 -> [DISCONNECTED 事件]
  -> 服务端等待超时 -> 会话过期
  -> 删除临时节点，触发相关Watch
  -> 网络恢复 -> [SESSION_EXPIRED 事件]
  -> 必须重新 new ZooKeeper()
  -> 重新注册Watch，重新创建临时节点
```

**Curator 的自动重连策略：**

```java
// Curator 提供多种重连策略

// 1. 指数退避重试（推荐）
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
// baseSleepTime=1000ms，最多重试3次，等待时间：1s, 2s, 4s（带随机抖动）

// 2. 固定间隔重试
retryPolicy = new RetryNTimes(3, 1000); // 重试3次，每次间隔1秒

// 3. 有界指数退避（生产推荐）
retryPolicy = new BoundedExponentialBackoffRetry(1000, 30000, 10);
// 最小1s，最大30s，最多10次

CuratorFramework client = CuratorFrameworkFactory.builder()
    .connectString("localhost:2181")
    .sessionTimeoutMs(10000)
    .connectionTimeoutMs(5000)
    .retryPolicy(retryPolicy)
    .namespace("myapp")  // 命名空间（自动添加前缀）
    .build();
client.start();
```

---

## 6. ZooKeeper 客户端

### 6.1 原生 Java API

```xml
<!-- Maven 依赖 -->
<dependency>
    <groupId>org.apache.zookeeper</groupId>
    <artifactId>zookeeper</artifactId>
    <version>3.8.4</version>
</dependency>
```

```java
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class ZooKeeperNativeDemo {

    private ZooKeeper zk;
    private static final String CONNECT_STRING = "localhost:2181";
    private static final int SESSION_TIMEOUT = 10000;

    public void connect() throws Exception {
        CountDownLatch latch = new CountDownLatch(1);
        zk = new ZooKeeper(CONNECT_STRING, SESSION_TIMEOUT, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });
        latch.await(); // 等待连接建立
        System.out.println("连接成功, session: " + Long.toHexString(zk.getSessionId()));
    }

    // 创建节点
    public String createNode(String path, String data, CreateMode mode) throws Exception {
        return zk.create(path, data.getBytes("UTF-8"),
                ZooDefs.Ids.OPEN_ACL_UNSAFE, mode);
    }

    // 读取节点数据
    public String getData(String path) throws Exception {
        Stat stat = new Stat();
        byte[] data = zk.getData(path, false, stat);
        System.out.println("版本: " + stat.getVersion() + ", 数据长度: " + stat.getDataLength());
        return new String(data, "UTF-8");
    }

    // 更新节点数据（乐观锁）
    public Stat setData(String path, String data, int version) throws Exception {
        return zk.setData(path, data.getBytes("UTF-8"), version);
    }

    // 获取子节点列表
    public List<String> getChildren(String path) throws Exception {
        return zk.getChildren(path, false);
    }

    // 检查节点是否存在
    public boolean exists(String path) throws Exception {
        return zk.exists(path, false) != null;
    }

    // 递归删除节点
    public void deleteRecursively(String path) throws Exception {
        List<String> children = zk.getChildren(path, false);
        for (String child : children) {
            deleteRecursively(path + "/" + child);
        }
        zk.delete(path, -1); // -1表示不校验版本
    }

    public void close() throws Exception {
        if (zk != null) zk.close();
    }

    public static void main(String[] args) throws Exception {
        ZooKeeperNativeDemo demo = new ZooKeeperNativeDemo();
        demo.connect();
        demo.createNode("/test", "hello", CreateMode.PERSISTENT);
        System.out.println("读取: " + demo.getData("/test"));
        demo.setData("/test", "world", 0);
        System.out.println("更新后: " + demo.getData("/test"));
        demo.deleteRecursively("/test");
        System.out.println("删除后存在: " + demo.exists("/test"));
        demo.close();
    }
}
```

### 6.2 Curator 框架

Curator 是 Netflix 开源的 ZooKeeper 客户端封装库，提供更高级别的 API 和分布式原语实现。

```xml
<!-- Curator 依赖 -->
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.5.0</version>
</dependency>
```

```java
import org.apache.curator.framework.*;
import org.apache.curator.framework.recipes.cache.*;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.data.Stat;

public class CuratorDemo {

    public static CuratorFramework buildClient() {
        CuratorFramework client = CuratorFrameworkFactory.builder()
            .connectString("localhost:2181,localhost:2182,localhost:2183")
            .sessionTimeoutMs(10000)
            .connectionTimeoutMs(5000)
            .retryPolicy(new ExponentialBackoffRetry(1000, 3))
            .namespace("myapp")  // 所有路径自动添加 /myapp 前缀
            .build();
        client.start();
        return client;
    }

    // 流式API：创建节点（自动递归创建父节点）
    public static void createWithParents(CuratorFramework client) throws Exception {
        client.create()
            .creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT)
            .forPath("/a/b/c", "data".getBytes());
    }

    // 流式API：读取数据（同时获取Stat）
    public static byte[] getDataWithStat(CuratorFramework client) throws Exception {
        Stat stat = new Stat();
        byte[] data = client.getData()
            .storingStatIn(stat)
            .forPath("/a/b/c");
        System.out.println("数据版本: " + stat.getVersion());
        return data;
    }

    // 流式API：更新数据（乐观锁）
    public static void setDataWithVersion(CuratorFramework client, int version) throws Exception {
        client.setData()
            .withVersion(version)
            .forPath("/a/b/c", "newData".getBytes());
    }

    // 事务操作
    public static void transactionDemo(CuratorFramework client) throws Exception {
        client.transaction().forOperations(
            client.transactionOp().create()
                .withMode(CreateMode.PERSISTENT)
                .forPath("/tx/node1", "v1".getBytes()),
            client.transactionOp().setData()
                .forPath("/tx/node2", "v2".getBytes()),
            client.transactionOp().delete()
                .forPath("/tx/node3")
        );
    }

    // CuratorCache（持久监听，推荐替代旧版 TreeCache/PathChildrenCache）
    public static void watchWithCuratorCache(CuratorFramework client) {
        CuratorCache cache = CuratorCache.build(client, "/config");
        CuratorCacheListener listener = CuratorCacheListener.builder()
            .forCreates(node -> {
                System.out.println("节点创建: " + node.getPath()
                    + " = " + new String(node.getData()));
            })
            .forChanges((oldNode, newNode) -> {
                System.out.println("节点变化: " + newNode.getPath()
                    + " [" + new String(oldNode.getData()) + "]"
                    + " -> [" + new String(newNode.getData()) + "]");
            })
            .forDeletes(node -> {
                System.out.println("节点删除: " + node.getPath());
            })
            .forInitialized(() -> System.out.println("Cache 初始化完成"))
            .build();
        cache.listenable().addListener(listener);
        cache.start();
    }
}
```

### 6.3 Curator 异步 API

```java
import org.apache.curator.framework.api.BackgroundCallback;
import org.apache.curator.framework.api.CuratorEvent;
import org.apache.curator.framework.async.AsyncCuratorFramework;

public class CuratorAsyncDemo {

    // 回调式异步
    public static void asyncGet(CuratorFramework client) throws Exception {
        BackgroundCallback callback = (curatorClient, event) -> {
            System.out.println("事件类型: " + event.getType());
            System.out.println("返回码: " + event.getResultCode());
            if (event.getData() != null) {
                System.out.println("数据: " + new String(event.getData()));
            }
        };
        client.getData()
            .inBackground(callback)
            .forPath("/config/app");
    }

    // Curator 5.x 推荐：CompletableFuture 风格
    public static void asyncWithFuture(CuratorFramework client) {
        AsyncCuratorFramework asyncClient = AsyncCuratorFramework.wrap(client);
        asyncClient.getData().forPath("/config/app")
            .thenAccept(data -> System.out.println("数据: " + new String(data)))
            .exceptionally(ex -> { System.err.println("错误: " + ex.getMessage()); return null; });
    }
}
```

---

## 7. 分布式应用场景实现

### 7.1 分布式锁

#### 7.1.1 Curator 内置互斥锁（推荐）

```java
import org.apache.curator.framework.recipes.locks.InterProcessMutex;

public class DistributedLockDemo {

    private final InterProcessMutex lock;

    public DistributedLockDemo(CuratorFramework client) {
        this.lock = new InterProcessMutex(client, "/locks/order-lock");
    }

    public void doWithLock(Runnable task) throws Exception {
        lock.acquire(); // 阻塞等待获取锁
        try {
            task.run();
        } finally {
            lock.release(); // 确保释放
        }
    }

    public boolean tryDoWithLock(Runnable task, long timeoutMs) throws Exception {
        boolean acquired = lock.acquire(timeoutMs, TimeUnit.MILLISECONDS);
        if (!acquired) return false;
        try {
            task.run();
            return true;
        } finally {
            lock.release();
        }
    }
}
```

#### 7.1.2 公平锁（基于临时有序节点手动实现）

```
公平锁原理（避免羊群效应）：

所有客户端在 /locks/xxx/ 下创建临时有序节点：
  /locks/order-lock/lock-0000000001  <- 客户端A（持锁）
  /locks/order-lock/lock-0000000002  <- 客户端B（监听0001）
  /locks/order-lock/lock-0000000003  <- 客户端C（监听0002）

A释放锁（删除0001）→ 只有B收到通知（而非B和C同时被唤醒）
→ B获得锁，C继续监听0002
```

```java
public class FairDistributedLock {

    private final ZooKeeper zk;
    private final String lockBasePath;
    private String lockNodePath;

    public FairDistributedLock(ZooKeeper zk, String lockName) {
        this.zk = zk;
        this.lockBasePath = "/locks/" + lockName;
    }

    public void lock() throws Exception {
        ensureBasePath();
        // 1. 创建临时有序节点
        lockNodePath = zk.create(lockBasePath + "/lock-", new byte[0],
            ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);
        String myNode = lockNodePath.substring(lockBasePath.length() + 1);

        while (true) {
            // 2. 获取所有锁节点并排序
            List<String> children = zk.getChildren(lockBasePath, false);
            Collections.sort(children);
            int myIndex = children.indexOf(myNode);

            if (myIndex == 0) {
                return; // 3. 是最小节点，获得锁
            }

            // 4. 监听前驱节点（避免羊群效应）
            String prevNode = children.get(myIndex - 1);
            CountDownLatch latch = new CountDownLatch(1);
            Stat stat = zk.exists(lockBasePath + "/" + prevNode, event -> {
                if (event.getType() == Watcher.Event.EventType.NodeDeleted) {
                    latch.countDown();
                }
            });
            if (stat != null) latch.await(); // 等待前驱释放
        }
    }

    public void unlock() throws Exception {
        if (lockNodePath != null) {
            zk.delete(lockNodePath, -1);
            lockNodePath = null;
        }
    }

    private void ensureBasePath() throws Exception {
        // 递归创建父路径（忽略 NodeExistsException）
        String[] parts = lockBasePath.split("/");
        StringBuilder path = new StringBuilder();
        for (String part : parts) {
            if (part.isEmpty()) continue;
            path.append("/").append(part);
            try {
                zk.create(path.toString(), new byte[0],
                    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            } catch (KeeperException.NodeExistsException ignored) {}
        }
    }
}
```

#### 7.1.3 读写锁

```java
import org.apache.curator.framework.recipes.locks.InterProcessReadWriteLock;

public class ReadWriteLockDemo {

    private final InterProcessReadWriteLock rwLock;

    public ReadWriteLockDemo(CuratorFramework client) {
        this.rwLock = new InterProcessReadWriteLock(client, "/locks/rw-lock");
    }

    public String read() throws Exception {
        InterProcessMutex readLock = rwLock.readLock();
        readLock.acquire();
        try {
            return "读取结果（多个读锁可共存）";
        } finally {
            readLock.release();
        }
    }

    public void write(String data) throws Exception {
        InterProcessMutex writeLock = rwLock.writeLock();
        writeLock.acquire();
        try {
            System.out.println("写入: " + data + "（独占，排除所有读写锁）");
        } finally {
            writeLock.release();
        }
    }
}
```

### 7.2 Leader 选举

```java
import org.apache.curator.framework.recipes.leader.*;

public class LeaderElectionDemo {

    private final LeaderSelector leaderSelector;

    public LeaderElectionDemo(CuratorFramework client, String nodeId) {
        leaderSelector = new LeaderSelector(client, "/election/leader",
            new LeaderSelectorListenerAdapter() {
                @Override
                public void takeLeadership(CuratorFramework client) throws Exception {
                    System.out.println("[" + nodeId + "] 我成为了 Leader！");
                    try {
                        doLeaderWork(nodeId);
                    } finally {
                        System.out.println("[" + nodeId + "] 放弃 Leader 职位");
                    }
                }
            });
        leaderSelector.autoRequeue(); // 释放后自动重新参与选举
    }

    public void start() { leaderSelector.start(); }
    public boolean isLeader() { return leaderSelector.hasLeadership(); }

    private void doLeaderWork(String nodeId) throws InterruptedException {
        for (int i = 0; i < 5; i++) {
            System.out.println("[" + nodeId + "] Leader工作中... 轮次 " + i);
            Thread.sleep(2000);
        }
    }

    // 另一种方式：LeaderLatch（锁存式选举）
    public static void leaderLatchDemo(CuratorFramework client) throws Exception {
        LeaderLatch latch = new LeaderLatch(client, "/election/latch", "node-1");
        latch.addListener(new LeaderLatchListener() {
            public void isLeader() { System.out.println("我成为了 Leader"); }
            public void notLeader() { System.out.println("我失去了 Leader 地位"); }
        });
        latch.start();
        latch.await(); // 阻塞等待成为Leader
        System.out.println("执行 Leader 任务...");
        latch.close(); // 主动放弃
    }
}
```

### 7.3 服务注册与发现

```java
import org.apache.curator.x.discovery.*;
import org.apache.curator.x.discovery.details.JsonInstanceSerializer;

public class ServiceDiscoveryDemo {

    // 自定义服务元数据
    public static class ServiceMeta {
        private String version;
        private int weight;
        // getters/setters...
    }

    private ServiceDiscovery<ServiceMeta> serviceDiscovery;

    public void init(CuratorFramework client) throws Exception {
        serviceDiscovery = ServiceDiscoveryBuilder.builder(ServiceMeta.class)
            .client(client)
            .basePath("/services")
            .serializer(new JsonInstanceSerializer<>(ServiceMeta.class))
            .build();
        serviceDiscovery.start();
    }

    // 服务注册
    public void register(String name, String host, int port, ServiceMeta meta) throws Exception {
        ServiceInstance<ServiceMeta> instance = ServiceInstance.<ServiceMeta>builder()
            .name(name).address(host).port(port).payload(meta).build();
        serviceDiscovery.registerService(instance);
    }

    // 服务发现（轮询负载均衡）
    public ServiceInstance<ServiceMeta> discover(String serviceName) throws Exception {
        ServiceProvider<ServiceMeta> provider = serviceDiscovery.serviceProviderBuilder()
            .serviceName(serviceName)
            .providerStrategy(new RoundRobinStrategy<>())
            .build();
        provider.start();
        return provider.getInstance();
    }
}
```

### 7.4 配置中心

```java
public class ConfigCenter {

    private final CuratorFramework client;
    private final String configBasePath;
    private final ConcurrentHashMap<String, String> localCache = new ConcurrentHashMap<>();
    private CuratorCache curatorCache;

    public ConfigCenter(CuratorFramework client, String namespace) {
        this.client = client;
        this.configBasePath = "/" + namespace + "/config";
    }

    public void start() throws Exception {
        loadAllConfigs(); // 初始加载
        curatorCache = CuratorCache.build(client, configBasePath);
        curatorCache.listenable().addListener(
            CuratorCacheListener.builder()
                .forCreates(node -> {
                    String key = extractKey(node.getPath());
                    String value = new String(node.getData());
                    localCache.put(key, value);
                    System.out.println("新增配置: " + key + " = " + value);
                })
                .forChanges((oldNode, newNode) -> {
                    String key = extractKey(newNode.getPath());
                    String newValue = new String(newNode.getData());
                    localCache.put(key, newValue);
                    System.out.println("配置变化: " + key + " -> " + newValue);
                })
                .forDeletes(node -> localCache.remove(extractKey(node.getPath())))
                .build()
        );
        curatorCache.start();
    }

    public String getConfig(String key, String defaultValue) {
        return localCache.getOrDefault(key, defaultValue);
    }

    public void setConfig(String key, String value) throws Exception {
        String path = configBasePath + "/" + key;
        if (client.checkExists().forPath(path) == null) {
            client.create().creatingParentsIfNeeded().forPath(path, value.getBytes());
        } else {
            client.setData().forPath(path, value.getBytes());
        }
    }

    private void loadAllConfigs() throws Exception {
        if (client.checkExists().forPath(configBasePath) == null) {
            client.create().creatingParentsIfNeeded().forPath(configBasePath);
            return;
        }
        List<String> keys = client.getChildren().forPath(configBasePath);
        for (String key : keys) {
            byte[] data = client.getData().forPath(configBasePath + "/" + key);
            localCache.put(key, new String(data));
        }
    }

    private String extractKey(String path) {
        return path.substring(configBasePath.length() + 1);
    }
}
```

### 7.5 分布式队列

```java
import org.apache.curator.framework.recipes.queue.*;

public class DistributedQueueDemo {

    public static DistributedQueue<String> createQueue(
            CuratorFramework client, QueueConsumer<String> consumer) throws Exception {

        QueueSerializer<String> serializer = new QueueSerializer<String>() {
            public byte[] serialize(String item) { return item.getBytes(); }
            public String deserialize(byte[] bytes) { return new String(bytes); }
        };

        DistributedQueue<String> queue = QueueBuilder
            .builder(client, consumer, serializer, "/queue/tasks")
            .buildQueue();
        queue.start();
        return queue;
    }

    // 生产消息
    public static void produce(DistributedQueue<String> queue) throws Exception {
        queue.put("task-" + System.currentTimeMillis());
    }

    // 消费者示例
    public static QueueConsumer<String> buildConsumer() {
        return new QueueConsumer<String>() {
            public void consumeMessage(String message) throws Exception {
                System.out.println("消费消息: " + message);
            }
            public void stateChanged(CuratorFramework client, ConnectionState newState) {
                System.out.println("连接状态变化: " + newState);
            }
        };
    }
}
```

---

## 8. 集群部署与运维

### 8.1 部署模式对比

```
+------------------+------------------+------------------+
| 模式             | 单机             | 伪集群           | 真集群          |
+------------------+------------------+------------------+-----------------+
| 节点数           | 1                | 3（同一主机）    | 3+（多主机）    |
| 生产可用         | 否               | 否               | 是              |
| 用途             | 开发测试         | 功能测试         | 生产环境        |
| 端口配置         | 单组端口         | 多组端口         | 多主机单组端口  |
+------------------+------------------+------------------+-----------------+
```

### 8.2 单机模式配置

```properties
# conf/zoo.cfg（单机）
tickTime=2000           # 基本时间单元（毫秒），心跳间隔
dataDir=/data/zookeeper/data    # 数据快照目录
dataLogDir=/data/zookeeper/logs # 事务日志目录（建议与数据目录分离）
clientPort=2181         # 客户端连接端口
minSessionTimeout=4000  # 最小会话超时（默认2*tickTime）
maxSessionTimeout=40000 # 最大会话超时（默认20*tickTime）
```

### 8.3 伪集群配置（同一主机3节点）

```properties
# node1: conf/zoo1.cfg
tickTime=2000
initLimit=10            # Follower初始同步Leader的最大心跳数（10*2s=20s）
syncLimit=5             # Leader与Follower同步的最大心跳数（5*2s=10s）
dataDir=/tmp/zk/1
clientPort=2181
server.1=localhost:2888:3888   # server.myid=host:数据端口:选举端口
server.2=localhost:2889:3889
server.3=localhost:2890:3890

# node2: conf/zoo2.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zk/2
clientPort=2182
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890

# node3: conf/zoo3.cfg
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zk/3
clientPort=2183
server.1=localhost:2888:3888
server.2=localhost:2889:3889
server.3=localhost:2890:3890
```

```bash
# 创建myid文件（必须与server.N的N一致）
mkdir -p /tmp/zk/1 /tmp/zk/2 /tmp/zk/3
echo "1" > /tmp/zk/1/myid
echo "2" > /tmp/zk/2/myid
echo "3" > /tmp/zk/3/myid

# 启动3个节点
bin/zkServer.sh start conf/zoo1.cfg
bin/zkServer.sh start conf/zoo2.cfg
bin/zkServer.sh start conf/zoo3.cfg

# 查看状态
bin/zkServer.sh status conf/zoo1.cfg  # Mode: follower 或 leader
```

### 8.4 生产集群配置（zoo.cfg 完整参数详解）

```properties
# ===== 基础配置 =====
tickTime=2000
initLimit=10                # Follower初始同步上限：10 * 2000ms = 20秒
syncLimit=5                 # Leader与Follower同步上限：5 * 2000ms = 10秒

# ===== 数据目录 =====
dataDir=/data/zookeeper/data
dataLogDir=/data/zookeeper/txlog  # 事务日志独立目录（提升IO性能）

# ===== 网络配置 =====
clientPort=2181
clientPortAddress=0.0.0.0   # 监听所有网卡
maxClientCnxns=300          # 单IP最大客户端连接数（0=无限制）

# ===== 快照与日志 =====
autopurge.snapRetainCount=3     # 保留的快照数量
autopurge.purgeInterval=1       # 自动清理间隔（小时）
preAllocSize=65536          # 事务日志预分配大小（KB），默认64MB
snapCount=100000            # 触发快照的事务数

# ===== 集群配置 =====
server.1=zk1.example.com:2888:3888
server.2=zk2.example.com:2888:3888
server.3=zk3.example.com:2888:3888
# Observer节点：server.4=zk4.example.com:2888:3888:observer

# ===== 3.5+ AdminServer =====
admin.enableServer=true
admin.serverPort=8080
4lw.commands.whitelist=ruok,stat,mntr,conf,srvr
```

### 8.5 四字命令（Four Letter Words）

```bash
# 需在 4lw.commands.whitelist 中启用
# 使用方式：echo "命令" | nc localhost 2181

echo ruok | nc localhost 2181   # 健康检查 -> 返回 imok
echo stat | nc localhost 2181   # 服务器统计（连接数、延迟、zxid等）
echo srvr | nc localhost 2181   # 服务器详细信息
echo conf | nc localhost 2181   # 显示服务配置
echo cons | nc localhost 2181   # 所有连接的客户端列表
echo dump | nc localhost 2181   # 会话和临时节点信息（仅Leader）
echo envi | nc localhost 2181   # 服务器环境（Java版本、主机名等）
echo wchs | nc localhost 2181   # Watch统计摘要
echo wchc | nc localhost 2181   # 每个客户端的Watch列表
echo wchp | nc localhost 2181   # 每个路径的Watch列表

# mntr 输出示例（适合Prometheus采集）：
echo mntr | nc localhost 2181
# zk_version        3.8.4-...
# zk_avg_latency    0
# zk_max_latency    10
# zk_min_latency    0
# zk_num_alive_connections  3
# zk_outstanding_requests   0
# zk_server_state   leader
# zk_znode_count    245
# zk_watch_count    32
# zk_ephemerals_count  10
# zk_approximate_data_size  12345
# zk_open_file_descriptor_count  50
# zk_max_file_descriptor_count   65536
```

### 8.6 AdminServer HTTP API（3.5+）

```bash
# AdminServer 默认端口 8080
curl http://localhost:8080/commands/configuration  # 查看配置
curl http://localhost:8080/commands/stats          # 统计信息
curl http://localhost:8080/commands/monitor        # 监控数据
curl http://localhost:8080/commands/get_children?path=/  # 查看子节点
```

### 8.7 JMX 监控配置

```bash
# zkServer.sh 启动时添加 JVM 参数
JVMFLAGS="-Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9999 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false"
```

```
JMX MBean 树：
org.apache.ZooKeeperService
  StandaloneServer_port2181    (单机模式)
  Leader_port2181              (集群Leader)
  Follower_port2181            (集群Follower)
      InMemoryDataTree         (数据树统计)
      Connections              (连接管理)
```

---

## 9. Spring Boot 集成

### 9.1 Spring Cloud ZooKeeper 服务发现

```xml
<!-- pom.xml -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>2022.0.4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- 服务发现 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    </dependency>
    <!-- 配置中心 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-config</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
spring:
  application:
    name: user-service
  cloud:
    zookeeper:
      connect-string: localhost:2181
      discovery:
        enabled: true
        root: /services
        register: true
        metadata:
          version: 1.0.0
          zone: cn-beijing
      config:
        enabled: true
        root: /config

server:
  port: 8080
```

```java
// 主应用类
@SpringBootApplication
@EnableDiscoveryClient
public class UserServiceApplication {
    public static void main(String[] args) {
        SpringApplication.run(UserServiceApplication.class, args);
    }
}

// 使用 @LoadBalanced RestTemplate 调用其他服务
@Configuration
public class RestTemplateConfig {
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
}

@Service
public class UserClient {
    @Autowired
    private RestTemplate restTemplate;

    public User getUser(Long userId) {
        // 直接使用服务名，自动解析和负载均衡
        return restTemplate.getForObject(
            "http://user-service/users/" + userId, User.class);
    }
}
```

### 9.2 Spring Cloud ZooKeeper 配置中心

ZooKeeper 中配置的路径规则（按优先级从高到低）：

```
/config/{appName},{profile}  <- 最高优先级
/config/{appName}
/config/application,{profile}
/config/application          <- 最低优先级（全局默认）
```

```bash
# 在 ZooKeeper 中写入配置
bin/zkCli.sh -server localhost:2181
create /config ""
create /config/user-service ""
create /config/user-service/app.feature.enabled "true"
create /config/user-service/app.max-connections "200"
create /config/user-service/app.database-url "jdbc:mysql://localhost:3306/users"

# 更新配置（应用通过 @RefreshScope 自动感知）
set /config/user-service/app.max-connections "500"
```

```java
// 动态配置刷新
@RestController
@RefreshScope  // 支持配置热刷新
public class ConfigController {

    @Value("${app.feature.enabled:false}")
    private boolean featureEnabled;

    @Value("${app.max-connections:100}")
    private int maxConnections;

    @GetMapping("/config")
    public Map<String, Object> getConfig() {
        return Map.of("featureEnabled", featureEnabled, "maxConnections", maxConnections);
    }
}
```

### 9.3 Curator Spring Boot 集成

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.5.0</version>
</dependency>
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.5.0</version>
</dependency>
```

```yaml
# application.yml
zookeeper:
  connect-string: localhost:2181
  session-timeout: 10000
  connection-timeout: 5000
  retry-base-sleep: 1000
  retry-max-retries: 3
  namespace: myapp
```

```java
@Configuration
public class ZooKeeperConfig {

    @Value("${zookeeper.connect-string}")
    private String connectString;
    @Value("${zookeeper.session-timeout:10000}")
    private int sessionTimeout;
    @Value("${zookeeper.connection-timeout:5000}")
    private int connectionTimeout;
    @Value("${zookeeper.retry-base-sleep:1000}")
    private int retryBaseSleep;
    @Value("${zookeeper.retry-max-retries:3}")
    private int retryMaxRetries;
    @Value("${zookeeper.namespace:}")
    private String namespace;

    @Bean(destroyMethod = "close")
    public CuratorFramework curatorFramework() {
        CuratorFrameworkFactory.Builder builder = CuratorFrameworkFactory.builder()
            .connectString(connectString)
            .sessionTimeoutMs(sessionTimeout)
            .connectionTimeoutMs(connectionTimeout)
            .retryPolicy(new ExponentialBackoffRetry(retryBaseSleep, retryMaxRetries));
        if (!namespace.isEmpty()) builder.namespace(namespace);
        CuratorFramework client = builder.build();
        client.start();
        return client;
    }
}

// 分布式锁 AOP 切面
@Component
@Aspect
public class DistributedLockAspect {

    @Autowired
    private CuratorFramework curatorFramework;

    @Around("@annotation(distributedLock)")
    public Object around(ProceedingJoinPoint pjp,
                         DistributedLock distributedLock) throws Throwable {
        String lockPath = "/locks/" + distributedLock.key();
        InterProcessMutex mutex = new InterProcessMutex(curatorFramework, lockPath);
        boolean acquired = mutex.acquire(distributedLock.timeoutMs(), TimeUnit.MILLISECONDS);
        if (!acquired) throw new RuntimeException("获取分布式锁超时: " + lockPath);
        try {
            return pjp.proceed();
        } finally {
            mutex.release();
        }
    }
}

// 自定义注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DistributedLock {
    String key();
    long timeoutMs() default 5000;
}

// 业务使用
@Service
public class OrderService {
    @DistributedLock(key = "order-create", timeoutMs = 3000)
    public Order createOrder(Long userId, OrderRequest request) {
        // 受分布式锁保护的业务逻辑
        return new Order();
    }
}
```

### 9.4 健康检查与 Actuator 集成

```java
@Component
public class ZooKeeperHealthIndicator implements HealthIndicator {

    @Autowired
    private CuratorFramework curatorFramework;

    @Override
    public Health health() {
        try {
            CuratorFramework.State state = curatorFramework.getState();
            if (state == CuratorFramework.State.STARTED) {
                curatorFramework.checkExists().forPath("/");
                return Health.up()
                    .withDetail("state", state)
                    .withDetail("connectString",
                        curatorFramework.getZookeeperClient().getCurrentConnectionString())
                    .build();
            }
            return Health.down().withDetail("state", state).build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}
```

---

## 10. 性能与调优

### 10.1 ZooKeeper 读写特性

```
读性能特点：
  - 任意节点都可以直接处理读请求（本地读）
  - 读操作不涉及网络同步，延迟极低（通常 < 1ms）
  - 可以通过增加 Observer 节点线性扩展读吞吐
  - 缺点：读到的数据可能不是最新的（最终一致性）
  - 如需强一致读：调用 sync() 后再读

写性能特点：
  - 所有写请求都需要 Leader 广播并获得多数 ACK
  - 写吞吐受限于 Leader 处理能力和网络延迟
  - 典型吞吐：5000-10000 写/秒（视硬件和网络）
  - 增加节点数不会提升写性能（反而略有下降）
```

### 10.2 Observer 的作用与配置

```
Observer 特点：
  - 不参与 Leader 选举和写提交投票
  - 可以无限水平扩展（不影响写一致性协议）
  - 接收 Leader 广播的 INFORM 消息（不是 COMMIT）
  - 适合：读密集型场景、跨数据中心部署
```

```properties
# zoo.cfg（所有节点都要有完整的server列表）
server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
server.4=obs1:2888:3888:observer  # 标记为Observer

# obs1 节点的 zoo.cfg 还需要添加：
peerType=observer
```

### 10.3 快照与事务日志

```
数据持久化机制：

事务日志（WAL）：
  - 每次事务提交前先写事务日志（Write-Ahead Log）
  - 顺序写，性能高，推荐独立磁盘
  - 文件格式：log.ZXID（如 log.1）

内存数据树（DataTree）：
  - 所有数据存储在内存中
  - 读操作直接从内存读取（高性能）
  - 重启后从快照 + 事务日志恢复

快照（Snapshot）：
  - 周期性将内存数据树序列化到磁盘
  - 文件格式：snapshot.ZXID
  - 触发条件：每处理 snapCount 个事务（含随机扰动防止多节点同时快照）

启动恢复过程：
  1. 找到最新的快照文件
  2. 加载快照到内存数据树
  3. 重放快照ZXID之后的所有事务日志
  4. 恢复完成，开始提供服务
```

```bash
# 查看快照内容
java -cp zookeeper-3.8.4.jar:lib/* \
  org.apache.zookeeper.server.SnapshotFormatter \
  /data/zookeeper/data/version-2/snapshot.1a

# 查看事务日志
java -cp zookeeper-3.8.4.jar:lib/* \
  org.apache.zookeeper.server.LogFormatter \
  /data/zookeeper/txlog/version-2/log.1
```

### 10.4 JVM 调优参数

```bash
# zkServer.sh 中配置（根据实际内存调整）

# 堆内存（通常 2-8GB）
export JVMFLAGS="-Xmx4g -Xms4g"

# GC 配置（推荐 G1GC）
export JVMFLAGS="$JVMFLAGS -XX:+UseG1GC"
export JVMFLAGS="$JVMFLAGS -XX:MaxGCPauseMillis=20"
export JVMFLAGS="$JVMFLAGS -XX:G1HeapRegionSize=16m"

# GC 日志
export JVMFLAGS="$JVMFLAGS -Xlog:gc*:file=/var/log/zookeeper/gc.log:time:filecount=5,filesize=20m"

# 数据包大小限制（默认 1MB，按需调整）
export JVMFLAGS="$JVMFLAGS -Djute.maxbuffer=10485760"  # 10MB
```

### 10.5 关键性能调优参数

```properties
# zoo.cfg 性能相关配置

# 事务日志预分配大小（减少磁盘分配次数，默认64MB）
preAllocSize=131072  # 128MB

# 快照触发频率（减少可降低磁盘IO，但增加恢复时间）
snapCount=200000  # 默认100000

# 最大客户端连接数（高并发场景增大）
maxClientCnxns=300  # 默认60

# 全局会话超时配置
minSessionTimeout=2000   # 最小2秒
maxSessionTimeout=120000 # 最大2分钟

# 内存中提交日志缓存（提升读性能，占用更多内存）
commitLogCount=1000  # 默认500

# 注意：forceSync=no 可提升写性能，但可能丢数据，仅测试使用
# forceSync=yes  # 生产环境保持默认yes
```

### 10.6 容量规划

```
节点数量建议：
  3节点：容忍1个节点故障（quorum=2），推荐最小生产配置
  5节点：容忍2个节点故障（quorum=3），推荐重要业务
  7节点：容忍3个节点故障（quorum=4），大型集群
  > 7节点：通常不推荐，写吞吐下降，选举复杂

  奇数原则：偶数节点与少一个奇数节点容错能力相同，浪费资源
    4节点 = 3节点容错能力（都只能容忍1个故障）
    → 推荐5节点而非4节点，推荐3节点而非2节点

内存估算：
  每个 ZNode 约消耗 200-500 字节内存（含元数据）
  100万节点 ≈ 200MB-500MB 内存
  JVM堆建议：znode内存 * 3（给GC留空间）

存储估算：
  事务日志：写QPS * 平均事务大小 * 保留天数
  快照：约等于当前内存中的数据量
  建议：事务日志和快照分别使用独立磁盘
        事务日志：高IOPS SSD（顺序写）
        快照：普通SSD即可（批量写）
```

### 10.7 监控告警指标

```
关键监控指标（来自 mntr 命令或 Prometheus exporter）：

性能指标：
  zk_avg_latency          平均请求延迟   （告警阈值：> 50ms）
  zk_max_latency          最大请求延迟   （告警阈值：> 200ms）
  zk_outstanding_requests 待处理请求队列 （告警阈值：> 10）

容量指标：
  zk_znode_count          ZNode总数      （告警阈值：> 1000000）
  zk_approximate_data_size 数据总大小（字节）
  zk_watch_count          Watch总数      （告警阈值：> 100000）
  zk_ephemerals_count     临时节点数量

连接指标：
  zk_num_alive_connections 活跃连接数
  zk_open_file_descriptor_count  已用文件描述符
  zk_max_file_descriptor_count   最大文件描述符（使用率>80%需告警）

集群健康：
  zk_server_state          leader/follower/observer
  zk_followers             Follower总数
  zk_synced_followers      已同步的Follower数（应=总follower数）
  zk_pending_syncs         待同步的Follower数（告警阈值：> 0持续）
```

```yaml
# Docker Compose：ZooKeeper + Prometheus 监控示例
services:
  zookeeper:
    image: zookeeper:3.8
    ports:
      - "2181:2181"
      - "8080:8080"
    environment:
      ZOO_4LW_COMMANDS_WHITELIST: "ruok,stat,mntr,conf,srvr"

  zk-exporter:
    image: dabealu/zookeeper-exporter:latest
    command: ["--zk-hosts=zookeeper:2181"]
    ports:
      - "9141:9141"

  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

---

## 11. 常见面试题 FAQ

### Q1: ZooKeeper 能保证强一致性吗？

**答：不能保证强一致性（线性一致性），只能保证顺序一致性。**

- **写操作**：经过 ZAB 广播协议，多数节点确认后提交，所有节点最终看到相同的写入顺序。
- **读操作**：客户端可以从任意节点读取，可能读到旧数据（Follower 可能尚未收到最新 COMMIT）。

如果需要读到最新数据，需在读取前调用 `sync()`：

```java
// Curator 强制同步后读取
client.sync().forPath("/config/key");
byte[] data = client.getData().forPath("/config/key");
```

### Q2: ZAB 协议和 Raft 有什么区别？

| 方面 | ZAB | Raft |
|------|-----|------|
| 日志提交顺序 | 严格按 ZXID 顺序 | 严格按 term+index 顺序 |
| 读一致性 | 本地读（非强一致） | 强一致（默认线性读） |
| Leader 切换 | 新 Leader 先同步后提供服务 | 新 Leader 选出后立即服务 |
| 设计目的 | ZooKeeper 专用 | 通用共识算法 |
| 成员变更 | 需要重启 | 联合共识，动态变更 |

两者核心思路类似：都基于 quorum 多数派，都有 Leader 机制，都保证日志顺序提交。

### Q3: 如何解决 Watch 一次性触发的问题？

**方案：**

1. **手动重新注册**：在 Watch 回调中每次处理完事件后立即重新注册 Watch
2. **持久 Watch（3.6+）**：`addWatch(path, watcher, AddWatchMode.PERSISTENT)` 触发后不自动移除
3. **Curator CuratorCache**（推荐）：内部实现持续监听，无需手动重新注册，重连后自动重建

**注意重新注册可能丢失事件的时间窗口：**

```
t1: 读数据 + 注册Watch
t2: Watch触发（数据变更）
t3: Watch回调中重新注册Watch
t2~t3 窗口内的变更可能丢失！
解决：重新注册Watch时同时读取最新数据并对比
```

### Q4: 临时节点可以有子节点吗？

**答：不可以。** 临时节点（EPHEMERAL）不允许创建子节点。这是 ZooKeeper 的设计约束。

如果需要临时节点 + 子节点结构，可以用 **容器节点（CONTAINER）** 代替：容器节点是持久的，但当最后一个子节点被删除后，服务器会自动清理容器节点。

### Q5: ZooKeeper 集群节点数为什么推荐奇数？

**答：** ZAB 协议要求超过半数（quorum）节点存活才能正常工作。

```
节点数 vs 容忍故障数：
  3节点：容忍1个故障（quorum=2）
  4节点：容忍1个故障（quorum=3）   <- 与3节点相同，浪费1个节点
  5节点：容忍2个故障（quorum=3）
  6节点：容忍2个故障（quorum=4）   <- 与5节点相同，浪费1个节点

结论：偶数节点增加了资源但没有增加容错能力，推荐奇数节点。
```

### Q6: 会话过期后，临时节点什么时候被删除？

**答：** 在服务端检测到会话超时后，**立即**删除该会话创建的所有临时节点，并触发相关 Watch 通知。

注意：不是客户端断开连接时，而是服务端确认会话超时（超过 sessionTimeout 未收到心跳）后才删除。短暂断线后快速重连，临时节点不会被删除。

### Q7: 如何实现分布式锁？最佳实践是什么？

**临时有序节点实现公平锁：**

```
步骤：
1. 在 /locks/resource/ 下创建临时有序节点
   得到：/locks/resource/lock-0000000003
2. 获取所有子节点并排序
3. 判断自己是否序号最小：是则获得锁，否则监听前驱节点
4. 前驱节点删除时收到Watch通知，回到步骤2
5. 业务执行完毕，删除自己的节点（释放锁）

为何监听前驱而非最小节点？
  避免羊群效应：监听最小节点时，锁释放会唤醒所有等待者
  监听前驱：每次只唤醒一个等待者，减少无效唤醒
```

**生产建议：直接使用 Curator 的 `InterProcessMutex`**，已实现完整的公平锁逻辑（可重入、超时获取等）。

### Q8: ZooKeeper 的 CAP 定理分析？

**答：ZooKeeper 是 CP 系统（一致性 + 分区容忍性），牺牲了可用性。**

```
C（Consistency）：ZAB 协议保证顺序一致性
A（Availability）：集群少数派节点无法处理写请求，牺牲可用性
P（Partition Tolerance）：网络分区时少数派停止服务，保证一致性

体现：
  - 超过半数节点故障 → 整个集群停止写服务
  - 正在进行 Leader 选举时 → 无法处理任何读写请求
  - Follower 与 Leader 失联 → Follower 停止提供服务（默认）

注意：ZooKeeper 读操作默认本地读，可能读到旧数据，
      严格说是"顺序一致性"而非"线性一致性"。
```

### Q9: ZooKeeper 脑裂问题如何解决？

**答：通过 Quorum（多数派）+ Epoch 机制防止脑裂。**

1. **多数派要求**：Leader 提交事务必须获得超过半数 ACK，网络分区后少数派无法提交新事务
2. **Epoch 机制**：每次 Leader 选举 epoch 递增，Follower 拒绝接受低 epoch 消息，旧 Leader 发出的 PROPOSAL 会被拒绝
3. **ZXID 保护**：新 Leader 必须拥有最高 ZXID，确保数据最新节点成为 Leader

### Q10: 如何处理 ZooKeeper 会话过期（SESSION_EXPIRED）？

**答：** 会话过期是**不可恢复**的状态，必须重新创建 `ZooKeeper` 实例：

```java
// 错误做法：忽略会话过期
if (event.getState() == KeeperState.Expired) {
    // do nothing  <- 导致客户端永久无法工作
}

// 正确做法：重建实例并恢复状态
if (event.getState() == KeeperState.Expired) {
    zk.close();
    zk = new ZooKeeper(connectString, timeout, this);
    // 重新注册所有Watch
    // 重新创建临时节点（如服务注册）
    // 重新检查并恢复业务状态
}
```

**使用 Curator 优势：** ConnectionStateListener 会处理 `LOST` 状态，自动重连后通知 `RECONNECTED` 事件，业务层只需在重连后重新执行初始化。

### Q11: ZooKeeper 的 Watcher 是在客户端还是服务端触发？

**答：** Watch 在**服务端**检测到节点变化后触发，通过**异步通知**推送给**客户端**处理。

```
流程：
1. 客户端注册Watch → Watch存储在服务端（按session分组）
2. 节点变化 → 服务端检测到变更
3. 服务端向持有该Watch的客户端发送WatchedEvent通知
   （只含事件类型和路径，不含新数据！）
4. 客户端在EventThread线程中调用Watcher.process()
5. 如需获取新数据，需客户端主动再次调用getData()等方法

Watch通知不含数据的原因：
  减少网络传输量；客户端收到通知时数据可能又变了
```

### Q12: ZooKeeper 节点存储数据大小限制及如何修改？

**答：** 默认限制为 **1MB**（`jute.maxbuffer` 参数控制）。

```bash
# 修改方式（客户端和服务端必须同时修改！）
export JVMFLAGS="-Djute.maxbuffer=10485760"  # 10MB
```

**实践建议：** ZooKeeper 设计用于存储**少量协调数据**，不适合存储大量业务数据。如需存储大数据，只在 ZooKeeper 存储引用（路径/主键），实际数据存储在 MySQL/HDFS/S3 等系统中。

### Q13: Curator 重试策略如何选择？

```java
// 推荐：有界指数退避（生产首选）
new BoundedExponentialBackoffRetry(1000, 30000, 10)
// 最小1s，最大30s，最多10次

// 指数退避（简单常用）
new ExponentialBackoffRetry(1000, 3)
// 1s, 2s, 4s 带随机抖动

// 固定次数固定间隔（短任务/用户请求）
new RetryNTimes(3, 500)

// 有时限重试（批处理任务）
new RetryUntilElapsed(60000, 2000)  // 最多尝试1分钟
```

### Q14: ZooKeeper 如何保证消息顺序性？

**答：** ZooKeeper 通过多层机制保证顺序：

1. **全局 ZXID 单调递增**：所有事务都有唯一且递增的 ZXID，保证全局顺序
2. **Leader 串行处理**：Leader 单线程处理事务 Proposal，不会乱序
3. **FIFO 队列广播**：Leader 通过 FIFO 队列向每个 Follower 发送 Proposal，保证每对 Leader-Follower 间的顺序
4. **客户端 FIFO**：同一客户端的多个请求保证按发送顺序处理

### Q15: tickTime、initLimit、syncLimit 参数如何计算？

```
tickTime = 2000（毫秒）  基本时间单元

initLimit = 10
  Follower 初始连接 Leader 并同步数据的超时
  = initLimit * tickTime = 10 * 2000 = 20秒
  数据量大时需增大（如 initLimit=20 → 40秒）

syncLimit = 5
  Leader 和 Follower 之间心跳和数据同步的超时
  = syncLimit * tickTime = 5 * 2000 = 10秒
  超时后 Leader 将该 Follower 从集群中移除

minSessionTimeout = 2 * tickTime = 4000ms（默认）
maxSessionTimeout = 20 * tickTime = 40000ms（默认）

客户端请求的 sessionTimeout 会被服务端强制约束到 [min, max] 范围内：
  请求 1000ms → 强制为 4000ms
  请求 5000ms → 保持 5000ms
  请求 100000ms → 强制为 40000ms
```

### Q16: ZooKeeper 和 Redis 分布式锁如何选择？

```
+------------------+----------------------------+----------------------------+
| 对比项           | ZooKeeper 分布式锁          | Redis 分布式锁              |
+------------------+----------------------------+----------------------------+
| 实现原理         | 临时有序节点 + Watch        | SET NX PX + Lua脚本         |
| 锁的公平性       | 公平锁（有序节点保证）       | 非公平（竞争SET）            |
| 崩溃自动释放     | 是（临时节点+会话）          | 是（TTL过期）               |
| 锁续期           | 不需要（会话心跳维持）       | 需要（看门狗机制）           |
| 时钟敏感性       | 不敏感                      | 敏感（TTL依赖时钟）          |
| 可重入性         | 支持（Curator内置）          | 需要手动实现（记录线程ID）   |
| 读写锁           | 支持（InterProcessRWLock）  | 需要自行实现                 |
| 性能             | 相对低（写入需ZAB共识）      | 极高（单机毫秒级）           |
+------------------+----------------------------+----------------------------+

选择建议：
  优先 Redis：高并发、对性能要求极高的场景
  优先 ZooKeeper：需要公平锁、强一致性、
                  已有ZooKeeper基础设施的场景
```

### Q17: ZooKeeper 如何避免惊群效应？

**惊群效应**：当锁释放时，所有等待者同时被唤醒并竞争，产生大量无效网络请求。

```
错误做法（惊群）：        正确做法（链式唤醒）：
  lock-1                   lock-1
  ↑  ↑  ↑                  ↓
  c2 c3 c4                 lock-2  <- c2监听lock-1
  (都监听lock-1)             ↓
  锁释放→所有人被唤醒       lock-3  <- c3监听lock-2
                           c4监听lock-3
                           每次只唤醒一个等待者
```

这也是 Curator `InterProcessMutex` 的内部实现原理：每个等待者只监听自己的前驱节点。

---

## 附录：常用命令速查

```bash
# zkCli.sh 常用命令
./bin/zkCli.sh -server localhost:2181

# 基础操作
ls /                          # 列出根节点下的子节点
ls -R /                       # 递归列出所有节点
ls -s /node                   # 列出子节点及Stat
create /node "data"           # 创建持久节点
create -e /node "data"        # 创建临时节点
create -s /node- "data"       # 创建持久有序节点
create -e -s /node- "data"    # 创建临时有序节点
get /node                     # 获取节点数据
get -s /node                  # 获取数据+Stat
set /node "newdata"           # 更新数据（不校验版本）
set /node "newdata" 2         # 更新数据（校验版本=2）
delete /node                  # 删除节点（必须无子节点）
deleteall /node               # 递归删除节点及所有子节点
stat /node                    # 查看节点Stat
addWatch /node                # 添加持久Watch（3.6+）
removewatches /node           # 移除Watch

# ACL操作
getAcl /node                  # 查看ACL
setAcl /node world:anyone:cdrwa  # 设置ACL（所有人所有权限）
addauth digest user:password  # 添加认证

# 四字命令
echo ruok | nc localhost 2181  # 健康检查
echo stat | nc localhost 2181  # 服务统计
echo mntr | nc localhost 2181  # 监控数据
echo conf | nc localhost 2181  # 配置信息
```

---

*文档结束 | ZooKeeper 3.8.x | Curator 5.x | Spring Boot 3.x*

## 12. 补充：深度原理与实践

### 12.1 ZooKeeper 内部请求处理流程

```
客户端请求在服务端的处理流水线（RequestProcessor Chain）：

Leader 节点的 Request Processor Chain：

  Request
     |
     v
  LeaderRequestProcessor        [仅处理来自Follower的本地请求]
     |
     v
  PrepRequestProcessor           [准备事务：创建事务头和数据]
     |
     v
  ProposalRequestProcessor       [将请求封装为Proposal广播给Followers]
     |           \
     |            v
     |         SyncRequestProcessor  [将事务写入事务日志]
     |            |
     |            v
     |         AckRequestProcessor   [向自己发送ACK]
     v
  CommitProcessor                [等待多数ACK后，提交到内存数据树]
     |
     v
  ToBeAppliedRequestProcessor    [应用到DataTree]
     |
     v
  FinalRequestProcessor          [生成响应返回给客户端]

Follower 节点的 Request Processor Chain：

  Request
     |
     v
  FollowerRequestProcessor
     |  (写请求转发给Leader)
     |  (读请求本地处理)
     v
  CommitProcessor
     |
     v
  FinalRequestProcessor
```

### 12.2 ZooKeeper 数据一致性保证详解

ZooKeeper 提供以下几种一致性保证：

```
1. 顺序一致性（Sequential Consistency）
   - 来自同一客户端的所有更新将按照其发送的顺序被应用
   - 不同客户端的操作可以以任意顺序交叉

2. 原子性（Atomicity）
   - 更新操作要么成功，要么失败
   - 不会出现部分更新的情况

3. 单一系统视图（Single System Image）
   - 客户端无论连接到哪个服务器，都将看到相同的服务视图
   - 客户端连接到新服务器时，不会看到比旧服务器更旧的状态

4. 可靠性（Reliability）
   - 一旦更新被成功应用，它将一直持久存在，直到客户端覆盖它

5. 及时性（Timeliness）
   - 客户端视图在某个时间范围内保证是最新的（有界失时性）
   - 不保证实时最新，但保证有上界延迟
```

### 12.3 ZooKeeper 选举算法演进

```
ZooKeeper 历史上有三种选举算法：

electionAlg=0: UDP-based Leader Election（已废弃）
  - 基于 UDP 通信
  - 早期版本使用
  - 3.4+ 版本已移除

electionAlg=1: UDP-based Fast Paxos（已废弃）
  - 基于 UDP 的快速 Paxos 算法
  - 3.4+ 版本已移除

electionAlg=2: Paxos-based Leader Election（已废弃）
  - 基于 Paxos 的 Leader 选举
  - 3.4+ 版本已移除

electionAlg=3: Fast Leader Election（当前唯一算法）
  - 基于 TCP 通信
  - 选举端口：server.x 配置中的第三个端口
  - 基于 epoch + zxid + myid 三元组比较
  - 选举收敛速度快（通常 < 200ms）
  - 3.4+ 版本为唯一可用算法
```

### 12.4 ZooKeeper 网络层架构

```
ZooKeeper 服务端网络组件：

ClientCnxn（客户端连接管理）
  ├── ServerCnxnFactory（NIO/Netty）
  │   ├── NIOServerCnxnFactory（默认）
  │   └── NettyServerCnxnFactory（通过配置开启）
  └── ServerCnxn（单个客户端连接）
      ├── 处理客户端请求
      ├── 发送Watch通知
      └── 会话管理

集群内部通信组件（Quorum）
  ├── QuorumCnxManager（集群节点间连接管理）
  │   └── 每对节点之间只有一条TCP连接（由myid较大者发起）
  ├── FastLeaderElection（选举算法）
  └── Leader/Follower/Observer（角色实现）
      ├── 2888端口：数据同步（Leader广播）
      └── 3888端口：选举通信

开启 Netty 方式：
  zoo.cfg: serverCnxnFactory=org.apache.zookeeper.server.NettyServerCnxnFactory
  优点：支持TLS、更好的并发性能
```

### 12.5 ZooKeeper 数据存储格式

```
ZooKeeper 使用 Jute 序列化框架（自研）：

ZNode 在内存中的存储结构（DataNode）：
  DataNode {
    byte[] data;           数据内容
    Long acl;              ACL引用（指向ACL缓存）
    StatPersisted stat;    持久化Stat
    Set<String> children;  子节点路径集合
  }

DataTree（内存数据树）：
  {
    ConcurrentHashMap<String, DataNode> nodes; // 路径→节点
    WatchManager dataWatches;   // 数据Watch
    WatchManager childWatches;  // 子节点Watch
    Map<Long, HashSet<String>> ephemerals; // 会话→临时节点集合
    long lastProcessedZxid;     // 最后处理的ZXID
  }

快照文件格式（version-2/snapshot.ZXID）：
  magic(8bytes) + version(4bytes) + dbid(8bytes)
  + sessions列表(Jute序列化)
  + ACL map(Jute序列化)
  + DataTree各节点(Jute序列化)
  + checksum(8bytes)

事务日志格式（version-2/log.ZXID）：
  file header(magic + version + dbid)
  + 多个 TxnLog记录：
    { checksum(8bytes) + TxnHeader(Jute) + TxnRecord(Jute) }
```

### 12.6 ZooKeeper 客户端连接原理

```java
// ZooKeeper 客户端内部组件
class ZooKeeper {
    // ClientCnxn 管理所有网络通信
    // 内部有两个线程：
    //   SendThread: 负责网络发送和接收
    //   EventThread: 负责事件分发（调用Watcher.process）

    // 连接到哪个服务器？
    // StaticHostProvider: 随机打散服务器列表，轮询连接
    // 断连后连接下一个服务器（不一定是原来的那个）

    // 请求队列
    // outgoingQueue: 待发送的请求
    // pendingQueue: 已发送等待响应的请求
}
```

```
客户端重连行为：

  1. 断连后，SendThread 尝试连接 HostProvider 中的下一个服务器
  2. 发送 ConnectRequest，携带旧的 sessionId 和 password
  3. 服务端验证 sessionId：
     - 有效：恢复会话，发送 ConnectResponse（含新timeout）
     - 过期：返回 SESSION_EXPIRED，触发 Expired 事件
  4. 重连成功后，客户端会收到 SyncConnected 事件
     注意：此时 Watch 仍然在服务端注册，不需要重新注册
     但断连期间发生的变更事件可能已经丢失！
```

### 12.7 ZooKeeper Sasl 认证

```properties
# zoo.cfg 开启 SASL 认证
authProvider.sasl=org.apache.zookeeper.server.auth.SASLAuthenticationProvider
requireClientAuthScheme=sasl

# Kerberos 配置
# jaasLoginRenewThread=true
```

```bash
# 启动时指定 JAAS 配置
export JVMFLAGS="-Djava.security.auth.login.config=/etc/zookeeper/jaas.conf"
```

```
# jaas.conf 示例（Digest-MD5）
Server {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    user_admin="admin-password"
    user_readonly="readonly-password";
};

Client {
    org.apache.zookeeper.server.auth.DigestLoginModule required
    username="admin"
    password="admin-password";
};
```

### 12.8 ZooKeeper 配额管理

ZooKeeper 支持对节点设置配额（quota），限制子树的节点数和数据大小：

```bash
# zkCli.sh 设置配额

# 设置 /app 路径下最多1000个节点，总数据不超过1MB
setquota -n 1000 -b 1048576 /app

# 查看配额
listquota /app
# Output:
#   /app quota:
#     count=1000
#     bytes=1048576
# count=current_count  bytes=current_bytes

# 删除配额
delquota /app
```

```
注意：ZooKeeper 配额是软限制（soft quota）！
  - 超出配额时，ZooKeeper 会记录警告日志，但不会拒绝写操作
  - 配额状态存储在 /zookeeper/quota/{path}/zookeeper_limits
  - 当前使用量存储在 /zookeeper/quota/{path}/zookeeper_stats
```

### 12.9 ZooKeeper 的 Multi-Request（批量操作）

ZooKeeper 支持将多个操作原子性地批量执行（Multi-Request）：

```java
// 原生 API 的 multi 操作
import org.apache.zookeeper.Op;
import org.apache.zookeeper.OpResult;

List<Op> ops = new ArrayList<>();
ops.add(Op.create("/test/node1", "data1".getBytes(),
    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT));
ops.add(Op.setData("/test/node2", "data2".getBytes(), -1));
ops.add(Op.delete("/test/node3", -1));

List<OpResult> results = zk.multi(ops);
// 所有操作原子执行：要么全部成功，要么全部失败
// 如果任意一个操作失败，所有操作都会回滚

// Curator transaction API（更简洁）
client.transaction().forOperations(
    client.transactionOp().create()
        .withMode(CreateMode.PERSISTENT)
        .forPath("/test/node1", "data1".getBytes()),
    client.transactionOp().setData()
        .forPath("/test/node2", "data2".getBytes()),
    client.transactionOp().delete()
        .forPath("/test/node3")
);
```

### 12.10 ZooKeeper 滚动升级

```
ZooKeeper 集群滚动升级步骤（以3节点集群为例）：

1. 升级 Follower 节点（逐个）
   - 停止 Follower-1
   - 更新 ZooKeeper 版本
   - 启动 Follower-1（新版本）
   - 验证 Follower-1 正常同步（检查 zk_synced_followers）
   - 重复对 Follower-2

2. 升级 Leader 节点
   - 停止 Leader（会触发重新选举）
   - 等待新 Leader 选出（通常 < 1秒）
   - 更新旧 Leader 的版本
   - 启动旧 Leader（变为 Follower）

注意事项：
  - 跨大版本升级（如 3.4 -> 3.8）需要检查兼容性和迁移指南
  - 升级前备份数据目录
  - 生产升级前在测试环境验证
  - 注意 zoo.cfg 新参数的含义和默认值变化
```

---

## 13. ZooKeeper 常见问题排查

### 13.1 连接失败排查

```bash
# 1. 检查服务是否启动
bin/zkServer.sh status

# 2. 检查端口是否监听
netstat -tlnp | grep 2181
ss -tlnp | grep 2181

# 3. 检查防火墙
iptables -L -n | grep 2181
firewall-cmd --list-ports

# 4. 测试连通性
echo ruok | nc localhost 2181  # 应返回 imok
telnet localhost 2181

# 5. 检查日志
tail -100 logs/zookeeper.out
grep -i "error\|exception\|warn" logs/zookeeper.out
```

### 13.2 集群选举异常排查

```bash
# 检查集群状态
echo stat | nc localhost 2181 | grep -E "Mode|Zookeeper|Connections"

# 检查各节点是否都在同一个epoch
echo mntr | nc localhost 2181 | grep zk_server_state
echo mntr | nc localhost 2182 | grep zk_server_state
echo mntr | nc localhost 2183 | grep zk_server_state

# 集群一直处于 LOOKING 状态的常见原因：
# 1. 节点间选举端口（3888）不通 -> 检查防火墙
# 2. myid 文件不存在或内容错误 -> 检查 dataDir/myid
# 3. zoo.cfg 中 server.x 配置不一致 -> 对比各节点配置
# 4. 节点数量不足过半 -> 确认至少有 floor(n/2)+1 个节点正常
```

### 13.3 性能问题排查

```bash
# 查看请求延迟
echo mntr | nc localhost 2181 | grep latency
# zk_avg_latency   延迟高(>50ms)时需要排查

# 查看队列积压
echo mntr | nc localhost 2181 | grep outstanding
# zk_outstanding_requests > 10 时表示处理积压

# 常见性能问题及解决方案：
#
# 问题1: 事务日志磁盘IO高
#   -> 事务日志和快照分到不同磁盘
#   -> 使用 SSD 替代 HDD
#
# 问题2: GC 停顿导致延迟抖动
#   -> 调整堆大小，使用 G1GC
#   -> 减少 ZNode 数据总量
#
# 问题3: 写QPS过高
#   -> 增加批量写入（multi/transaction）
#   -> 应用层合并写操作
#   -> 考虑读写分离（Observer 分担读压力）
#
# 问题4: Watch 数量爆炸
#   -> 检查是否有 Watch 泄漏（未正常取消）
#   -> 使用持久 Watch 替代反复注册
```

### 13.4 数据恢复

```bash
# 场景：误操作删除了重要节点，需要从快照恢复

# 1. 停止 ZooKeeper 服务
bin/zkServer.sh stop

# 2. 找到最近的快照文件
ls -lt /data/zookeeper/data/version-2/snapshot.*

# 3. 使用 SnapshotFormatter 查看快照内容
java -cp "zookeeper-3.8.4.jar:lib/*" \
  org.apache.zookeeper.server.SnapshotFormatter \
  /data/zookeeper/data/version-2/snapshot.100000abc

# 4. 如果需要回滚到某个时间点：
#    - 找到该时间点之前的快照
#    - 删除该快照之后的所有事务日志
#    - 启动 ZooKeeper（会从该快照开始恢复）

# 5. 重启 ZooKeeper
bin/zkServer.sh start
```

### 13.5 内存溢出（OOM）排查

```bash
# 常见 OOM 场景及解决：

# 1. ZNode 数量过多
echo mntr | nc localhost 2181 | grep znode_count
# 如果 zk_znode_count > 500000，考虑清理无用节点

# 2. 单个 ZNode 数据过大
echo mntr | nc localhost 2181 | grep data_size
# 检查 zk_approximate_data_size

# 3. Watch 数量过多（内存泄漏）
echo mntr | nc localhost 2181 | grep watch_count

# 4. 增加 JVM 堆内存
export JVMFLAGS="-Xmx8g -Xms8g"

# 5. 生成堆转储用于分析
jmap -dump:format=b,file=/tmp/zk-heap.hprof $(jps | grep QuorumPeer | cut -d" " -f1)
# 使用 MAT 或 VisualVM 分析 hprof 文件
```

---

## 14. ZooKeeper 生产最佳实践

### 14.1 部署建议

```
1. 节点数量
   - 生产环境最少3节点，重要业务建议5节点
   - 使用奇数节点，避免偶数节点的资源浪费
   - 不建议超过7节点（写性能下降，选举复杂）

2. 跨机架/可用区部署
   - 3节点：每个节点在不同机架/可用区
   - 5节点：建议 2-2-1 分布（容忍整个机架故障）

3. 硬件建议
   - CPU：4核以上（ZooKeeper 是单线程处理模型，主要是单核性能）
   - 内存：8GB以上（视ZNode数量调整，多留GC空间）
   - 磁盘：SSD（事务日志顺序写对延迟敏感）
   - 网络：万兆网络（集群内部通信）

4. 磁盘分离
   - dataDir 和 dataLogDir 配置到不同磁盘
   - 事务日志（dataLogDir）建议高IOPS SSD
   - 快照数据（dataDir）普通SSD即可
```

### 14.2 配置最佳实践

```properties
# 生产推荐配置

# 1. 事务日志和快照分离（最重要！）
dataDir=/data/zk-snap
dataLogDir=/data/zk-txlog

# 2. 配置自动清理（避免磁盘爆满）
autopurge.snapRetainCount=10  # 保留10个快照
autopurge.purgeInterval=1     # 每小时清理一次

# 3. 适当调大超时（视网络质量调整）
tickTime=2000
initLimit=20    # 网络较差时增大
syncLimit=10    # 网络较差时增大

# 4. 适当增大客户端连接数
maxClientCnxns=300

# 5. 开启AdminServer（便于运维）
admin.enableServer=true
admin.serverPort=8080

# 6. 限制四字命令（安全）
4lw.commands.whitelist=ruok,stat,mntr,conf,srvr
```

### 14.3 客户端使用最佳实践

```
1. 使用 Curator 而非原生 API
   - 自动重连（处理 Disconnected/Expired）
   - 内置重试机制
   - 丰富的高级特性（锁、选举、缓存等）

2. 合理设置 sessionTimeout
   - 太短：网络抖动导致临时节点频繁删除
   - 太长：节点故障后临时节点释放慢，影响服务发现准确性
   - 推荐：10-30 秒

3. 处理好会话过期
   - 监听 CONNECTION_SUSPENDED 和 CONNECTION_LOST 事件
   - LOST 状态下重新注册服务、创建临时节点

4. 不要在 ZooKeeper 存储大数据
   - 单个节点数据保持 < 10KB
   - ZooKeeper 不是数据库，是协调服务

5. 避免频繁创建/删除 ZNode
   - 每次写入都需要 ZAB 共识，性能有限
   - 批量操作使用 multi/transaction

6. Watch 使用注意
   - 优先使用 Curator CuratorCache（避免手动管理 Watch）
   - 处理好 Watch 丢失的情况（断连期间变更可能丢失）
   - 避免在 Watch 回调中执行耗时操作（阻塞 EventThread）
```

### 14.4 ZooKeeper 在 Kafka 中的应用（历史）

```
Kafka 2.8 之前依赖 ZooKeeper 的功能：

1. Broker 注册与发现
   路径：/brokers/ids/{brokerId}
   内容：Broker的host、port、版本等信息
   类型：临时节点（Broker挂掉后自动删除）

2. Topic 元数据
   路径：/brokers/topics/{topicName}
   内容：分区数、副本分配方案

3. Controller 选举
   路径：/controller
   类型：临时节点（第一个创建成功的Broker成为Controller）

4. ISR（In-Sync Replica）状态
   路径：/brokers/topics/{topic}/partitions/{id}/state
   内容：leader、isr列表

5. Consumer Group 元数据（旧版）
   路径：/consumers/{groupId}/offsets/{topic}/{partition}
   注：新版Kafka Offset已迁移到内部Topic __consumer_offsets

Kafka KRaft 模式（2.8+ 可用，3.0+ 推荐）：
   完全移除 ZooKeeper 依赖
   使用内置的 Raft 协议管理元数据
   优势：降低运维复杂度，提升启动速度，支持更多分区
```

### 14.5 ZooKeeper 在 Hadoop 生态中的应用

```
Hadoop/HBase 中的 ZooKeeper 使用：

HDFS NameNode HA：
  /hadoop-ha/{nameservice}/ActiveBreadCrumb   当前Active NameNode
  /hadoop-ha/{nameservice}/ActiveStandbyElectorLock   选举锁
  ZKFC（ZooKeeper Failover Controller）监控NameNode健康

YARN ResourceManager HA：
  /yarn-leader-election/{rmId}/ActiveBreadCrumb
  自动故障切换，基于 ZooKeeper 选举

HBase：
  /hbase/master              HBase Master地址
  /hbase/rs/{regionServer}   RegionServer注册（临时节点）
  /hbase/root-region-server  hbase:meta表位置
  /hbase/table/{tableName}   表状态信息

这些框架都内置了 ZooKeeper 客户端，无需单独配置 Curator。
```

---

## 总结

ZooKeeper 作为成熟的分布式协调服务，在大数据和微服务生态中占有重要地位。核心要点回顾：

| 主题 | 核心要点 |
|------|----------|
| 数据模型 | 树形 ZNode，7种类型，1MB上限，Stat版本控制 |
| Watch 机制 | 一次性触发，3.6+ 持久 Watch，推荐 CuratorCache |
| ZAB 协议 | epoch+ZXID+quorum，两阶段广播，崩溃恢复 |
| 一致性 | 顺序一致性（非强一致），sync() 获得最新数据 |
| 会话 | sessionTimeout，心跳，临时节点绑定会话 |
| 客户端 | 推荐 Curator，ExponentialBackoff 重试 |
| 分布式锁 | 临时有序节点+监听前驱，推荐 InterProcessMutex |
| 集群 | 推荐奇数节点（3/5/7），dataDir 与 dataLogDir 分盘 |
| Spring 集成 | Spring Cloud ZooKeeper 或 手动注入 CuratorFramework |
| 性能 | 读本地读高效，写受 ZAB 制约，Observer 扩展读 |
| CAP | CP 系统，分区时少数派停止写服务 |

---


## 附录B：完整 Maven 依赖清单

```xml
<!-- pom.xml 完整ZooKeeper相关依赖 -->
<properties>
    <zookeeper.version>3.8.4</zookeeper.version>
    <curator.version>5.5.0</curator.version>
    <spring-cloud.version>2022.0.4</spring-cloud.version>
</properties>

<dependencies>
    <!-- ZooKeeper 原生客户端 -->
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>${zookeeper.version}</version>
        <!-- 排除日志框架冲突 -->
        <exclusions>
            <exclusion>
                <groupId>org.slf4j</groupId>
                <artifactId>slf4j-log4j12</artifactId>
            </exclusion>
            <exclusion>
                <groupId>log4j</groupId>
                <artifactId>log4j</artifactId>
            </exclusion>
        </exclusions>
    </dependency>

    <!-- Curator 框架（推荐使用） -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>${curator.version}</version>
    </dependency>

    <!-- Curator 分布式原语（锁、选举、队列等） -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>${curator.version}</version>
    </dependency>

    <!-- Curator 服务发现（可选） -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-x-discovery</artifactId>
        <version>${curator.version}</version>
    </dependency>

    <!-- Spring Cloud ZooKeeper 服务发现（可选） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-discovery</artifactId>
    </dependency>

    <!-- Spring Cloud ZooKeeper 配置中心（可选） -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-zookeeper-config</artifactId>
    </dependency>

    <!-- 测试 -->
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-test</artifactId>
        <version>${curator.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>
```

## 附录C：ZooKeeper 测试工具

### C.1 使用 Curator TestServer 进行单元测试

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.RetryOneTime;
import org.apache.curator.test.TestingServer;
import org.junit.jupiter.api.*;

public class ZooKeeperServiceTest {

    private static TestingServer testingServer;
    private static CuratorFramework client;

    @BeforeAll
    static void setUp() throws Exception {
        // 启动内嵌ZooKeeper服务器（测试用）
        testingServer = new TestingServer(2181, true);
        client = CuratorFrameworkFactory.builder()
            .connectString(testingServer.getConnectString())
            .sessionTimeoutMs(5000)
            .connectionTimeoutMs(5000)
            .retryPolicy(new RetryOneTime(100))
            .build();
        client.start();
        client.blockUntilConnected();
    }

    @AfterAll
    static void tearDown() throws Exception {
        client.close();
        testingServer.close();
    }

    @Test
    void testCreateAndGet() throws Exception {
        // 测试节点创建和读取
        client.create().creatingParentsIfNeeded()
              .forPath("/test/node", "hello".getBytes());
        byte[] data = client.getData().forPath("/test/node");
        Assertions.assertEquals("hello", new String(data));
    }

    @Test
    void testDistributedLock() throws Exception {
        InterProcessMutex lock = new InterProcessMutex(client, "/locks/test");
        boolean acquired = lock.acquire(5000, TimeUnit.MILLISECONDS);
        Assertions.assertTrue(acquired);
        try {
            // 测试锁内业务
            byte[] data = client.getData().forPath("/test/node");
            Assertions.assertNotNull(data);
        } finally {
            lock.release();
        }
    }

    @Test
    void testWatch() throws Exception {
        CountDownLatch latch = new CountDownLatch(1);
        // 注册Watch
        client.getData().usingWatcher((Watcher) event -> {
            if (event.getType() == Watcher.Event.EventType.NodeDataChanged) {
                latch.countDown();
            }
        }).forPath("/test/node");

        // 修改数据触发Watch
        client.setData().forPath("/test/node", "world".getBytes());

        // 等待Watch回调
        boolean triggered = latch.await(3, TimeUnit.SECONDS);
        Assertions.assertTrue(triggered, "Watch应该被触发");
    }
}
```

### C.2 使用 TestingCluster 测试集群行为

```java
import org.apache.curator.test.TestingCluster;

public class ZooKeeperClusterTest {

    private TestingCluster cluster;
    private CuratorFramework client;

    @BeforeEach
    void setUp() throws Exception {
        // 启动内嵌3节点集群
        cluster = new TestingCluster(3);
        cluster.start();

        client = CuratorFrameworkFactory.builder()
            .connectString(cluster.getConnectString())
            .sessionTimeoutMs(5000)
            .retryPolicy(new ExponentialBackoffRetry(100, 3))
            .build();
        client.start();
        client.blockUntilConnected();
    }

    @AfterEach
    void tearDown() throws Exception {
        client.close();
        cluster.close();
    }

    @Test
    void testLeaderElection() throws Exception {
        // 测试Leader选举
        List<LeaderLatch> latches = new ArrayList<>();
        List<String> leaders = new CopyOnWriteArrayList<>();

        // 创建3个竞争Leader的客户端
        for (int i = 0; i < 3; i++) {
            final String nodeId = "node-" + i;
            CuratorFramework c = CuratorFrameworkFactory.builder()
                .connectString(cluster.getConnectString())
                .retryPolicy(new RetryOneTime(100))
                .build();
            c.start();

            LeaderLatch latch = new LeaderLatch(c, "/election/test", nodeId);
            latch.addListener(new LeaderLatchListener() {
                public void isLeader() { leaders.add(nodeId); }
                public void notLeader() {}
            });
            latch.start();
            latches.add(latch);
        }

        Thread.sleep(2000); // 等待选举完成
        Assertions.assertEquals(1, leaders.size(), "应该只有一个Leader");

        for (LeaderLatch latch : latches) latch.close();
    }

    @Test
    void testNodeFailover() throws Exception {
        // 测试节点故障时的行为
        client.create().creatingParentsIfNeeded()
              .forPath("/test/data", "initial".getBytes());

        // 停止其中一个实例（模拟故障）
        TestingCluster.InstanceSpec instance = cluster.getInstances().iterator().next();
        cluster.killServer(instance);

        Thread.sleep(1000);

        // 集群仍应可用（还剩2/3节点）
        byte[] data = client.getData().forPath("/test/data");
        Assertions.assertEquals("initial", new String(data));
    }
}
```

## 附录D：ZooKeeper 命令行工具速查

```
zkCli.sh 内置帮助命令：

ZooKeeper -server host:port -client-configuration properties-file cmd args
        addWatch [-m mode] path # optional mode is one of [PERSISTENT,
            PERSISTENT_RECURSIVE] - default is PERSISTENT_RECURSIVE
        close
        config [-c] [-w] [-s]
        connect host:port
        create [-s] [-e] [-c] [-t ttl] path [data] [acl]
        delete [-v version] path
        deleteall path [-b batch size]
        delquota [-n|-b|-N|-B] path
        get [-s] [-w] path
        getAcl [-s] path
        getAllChildrenNumber path
        getEphemerals path
        history
        listquota path
        ls [-s] [-w] [-R] path
        printwatches on|off
        quit
        reconfig [-s] [-v version] [[-file path] | [-members serverID=host:port1:port2;
            port3[,...]*]] | [-add serverId=host:port1:port2;port3[,...]]* [-remove serverId[,...]*]
        redo cmdno
        removewatches path [-c|-d|-a] [-l]
        set [-s] [-v version] path data
        setAcl [-s] [-v version] [-R] path acl
        setquota -n|-b|-N|-B val path
        stat [-w] path
        sync path
        version
        whoami
```

---

*ZooKeeper 详解文档 完整版 | 版本 3.8.x | 最后更新：2026-07-07*

