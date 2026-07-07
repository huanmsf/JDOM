# MongoDB 详解 - 从零到精通

> 版本覆盖：MongoDB 4.x / 5.x / 6.x / 7.x
> 适用人群：后端开发工程师、架构师、技术面试备考者
> 代码环境：Java 17 + Spring Boot 3.x + spring-boot-starter-data-mongodb

---

## 目录

- [Part 1: MongoDB 整体架构](#part-1)
- [Part 2: MongoDB CRUD 操作](#part-2)
- [Part 3: 聚合管道深度解析](#part-3)
- [Part 4: 索引深度解析](#part-4)
- [Part 5: 数据模型设计](#part-5)
- [Part 6: 事务](#part-6)
- [Part 7: 副本集 Replica Set](#part-7)
- [Part 8: 分片集群 Sharded Cluster](#part-8)
- [Part 9: Spring Boot 集成实战](#part-9)
- [Part 10: 完整实战案例](#part-10)
- [Part 11: 性能调优](#part-11)
- [Part 12: 常见面试题 FAQ](#part-12)

---

# Part 1: MongoDB 整体架构

## 1.1 MongoDB 是什么

MongoDB 是一个开源的、面向文档的 NoSQL 数据库，由 MongoDB Inc.（原 10gen）开发，2009年首次发布。
它以 **BSON（Binary JSON）** 格式存储数据，每条记录是一个文档（Document），文档被组织在集合
（Collection）中，集合属于数据库（Database）。

### 核心设计哲学

```
+--------------------------------------------------+
|              MongoDB 设计哲学                     |
+--------------------------------------------------+
|                                                  |
|  1. 灵活性  - Schema-free，无需预定义表结构       |
|  2. 可扩展  - 原生支持水平分片（Sharding）        |
|  3. 高性能  - 内存映射、索引优化、WiredTiger引擎  |
|  4. 高可用  - 副本集（Replica Set）自动故障切换   |
|  5. 丰富查询 - 支持嵌套查询/聚合/全文搜索/地理   |
|                                                  |
+--------------------------------------------------+
```

### MongoDB 的典型使用场景

| 场景 | 原因 |
|------|------|
| 内容管理系统（CMS） | 文章结构各异，文档模型天然适配 |
| 产品目录（电商SKU） | 商品属性不统一，灵活 Schema |
| 用户行为日志 | 写入量大，TTL自动清理，时序场景 |
| 实时分析 | 聚合管道功能强大，就地计算 |
| 物联网数据 | 传感器数据结构多变，时间序列 |
| 地理位置应用 | 原生 2dsphere 地理空间索引 |
| 社交网络 | 好友关系/动态 feed，嵌套文档 |
| 游戏数据 | 玩家状态/装备属性，Schema灵活 |

---

## 1.2 MongoDB vs MySQL 全面对比

### 核心概念映射

```
MySQL                          MongoDB
+-------------------+          +-------------------+
| Database          |  <--->   | Database          |
| Table             |  <--->   | Collection        |
| Row               |  <--->   | Document          |
| Column            |  <--->   | Field             |
| Index             |  <--->   | Index             |
| JOIN              |  <--->   | lookup / 嵌入式   |
| Primary Key       |  <--->   | _id               |
| Foreign Key       |  <--->   | DBRef / 手动引用  |
| Schema（固定）    |  <--->   | Schema-free（动态）|
| Transaction       |  <--->   | Transaction 4.0+  |
+-------------------+          +-------------------+
```

### 详细对比表

| 维度 | MySQL | MongoDB |
|------|-------|---------|
| **数据模型** | 关系型（表/行/列） | 文档型（JSON/BSON） |
| **Schema** | 强Schema，DDL预定义 | 动态Schema，字段随时增减 |
| **查询语言** | SQL | MQL（MongoDB Query Language）|
| **JOIN操作** | 原生多表JOIN | $lookup（聚合阶段）或嵌入 |
| **事务** | ACID（成熟，InnoDB） | ACID（4.0+，多文档事务） |
| **扩展方式** | 主要垂直扩展（Scale Up） | 水平扩展（Scale Out，分片）|
| **索引** | B+Tree，覆盖多种类型 | B-Tree，支持地理/文本/TTL等 |
| **存储格式** | 行存储 | BSON文档 |
| **最大记录大小** | 行大小受页大小限制 | 单文档最大 16MB |
| **聚合** | GROUP BY / 窗口函数 | 聚合管道（功能更灵活）|
| **全文搜索** | 有限支持 | 文本索引 + $text查询 |
| **地理空间** | 有限支持 | 原生 2dsphere/2d 索引 |
| **写入性能** | 一般（ACID约束） | 高（可调Write Concern）|
| **读取性能** | JOIN查询可能较慢 | 嵌入文档一次IO读取 |
| **一致性** | 强一致性 | 可调（最终/强一致） |

### 选型建议

```
什么时候选 MySQL：
  1. 数据结构稳定，关系复杂（如ERP、财务系统）
  2. 需要复杂多表JOIN
  3. 严格的事务一致性要求（如银行转账）
  4. 团队熟悉SQL，DBA资源充足

什么时候选 MongoDB：
  1. Schema 经常变化（产品快速迭代）
  2. 数据自然呈层次/嵌套结构
  3. 需要水平扩展，数据量超大
  4. 地理位置/时间序列/内容管理场景
  5. 敏捷开发，需要快速上手
```

---

## 1.3 核心概念详解

### Database（数据库）

MongoDB 中的数据库是集合的逻辑容器。一个 MongoDB 实例可以承载多个数据库，每个数据库有独立的文件集合。

```javascript
// MongoDB Shell 示例
use myDatabase          // 切换/创建数据库
db.getName()           // 获取当前数据库名
show dbs               // 列出所有数据库（有数据才显示）
db.dropDatabase()      // 删除当前数据库
```

**系统保留数据库：**
- `admin`：权限管理，存储用户和角色
- `local`：副本集 oplog，不参与复制
- `config`：分片集群的元数据

### Collection（集合）

集合是 MongoDB 中文档的分组，类似于 MySQL 中的表，但无需预先定义结构。

```javascript
// 集合操作
db.createCollection("users")                    // 显式创建
db.createCollection("logs", {                   // 创建带选项的集合
  capped: true,                                 // 固定大小集合
  size: 10485760,                               // 最大 10MB
  max: 1000                                     // 最多 1000 条文档
})
db.getCollectionNames()                         // 列出所有集合
db.users.drop()                                 // 删除集合
db.users.stats()                                // 集合统计信息
```

**固定集合（Capped Collection）** 是一种特殊集合，大小固定，按插入顺序存储，旧数据自动覆盖。
适合日志、消息队列等场景。

### Document（文档）

文档是 MongoDB 的基本数据单元，以 BSON 格式存储，类似于 JSON 对象。

```javascript
// 一个典型的文档示例
{
  "_id": ObjectId("64a7b3c9d1e2f3a4b5c6d7e8"),  // 主键
  "username": "zhangsan",                          // 字符串
  "age": 28,                                       // 整数
  "score": 98.5,                                   // 浮点数
  "isActive": true,                                // 布尔
  "createdAt": ISODate("2024-01-15T08:30:00Z"),   // 日期
  "tags": ["mongodb", "nosql", "database"],        // 数组
  "address": {                                     // 嵌套文档
    "city": "上海",
    "district": "浦东新区",
    "zipCode": "200120"
  },
  "orders": [                                      // 文档数组
    { "orderId": "ORD-001", "amount": 299.00 },
    { "orderId": "ORD-002", "amount": 599.00 }
  ],
  "metadata": null                                 // null值
}
```

**文档限制：**
- 单文档最大大小：**16MB**
- `_id` 字段必须唯一，可以是任意类型（默认 ObjectId）
- 字段名不能以 `$` 开头，不能包含 `.`（点号）

### Field（字段）

字段是文档中的键值对。MongoDB 支持任意嵌套层级，但建议嵌套不超过 5 层。

```javascript
// 字段访问路径（点表示法）
{ "a": { "b": { "c": 1 } } }
// 访问方式：a.b.c

// 数组元素访问
{ "items": ["apple", "banana", "cherry"] }
// 访问方式：items.0（第一个元素）
```

---

## 1.4 BSON 格式详解

BSON（Binary JSON）是 MongoDB 存储和网络传输数据的格式，是 JSON 的二进制超集。

### 为什么不直接用 JSON？

```
JSON 的局限性：
  1. 纯文本，解析慢
  2. 不支持 Date 类型
  3. 不支持 Binary 数据
  4. 数字不区分整型/浮点
  5. 不记录长度，必须完整解析

BSON 的优势：
  1. 二进制编码，解析快
  2. 记录文档和字段长度，支持快速跳过
  3. 支持更丰富的数据类型
  4. 内嵌长度信息，可以快速定位字段
```

### BSON 数据类型完整列表

| 类型名 | BSON类型码 | 示例 | 说明 |
|--------|-----------|------|------|
| Double | 1 | `3.14` | 64位浮点数 |
| String | 2 | `"hello"` | UTF-8编码字符串 |
| Object | 3 | `{ "k": "v" }` | 嵌套文档 |
| Array | 4 | `[1, 2, 3]` | 数组 |
| Binary | 5 | `BinData(0, "abc")` | 二进制数据 |
| ObjectId | 7 | `ObjectId("...")` | 12字节唯一标识 |
| Boolean | 8 | `true / false` | 布尔值 |
| Date | 9 | `ISODate("2024-01-15")` | UTC毫秒时间戳 |
| Null | 10 | `null` | 空值 |
| Regex | 11 | `/pattern/flags` | 正则表达式 |
| JavaScript | 13 | `function() {}` | JS代码（少用）|
| Int32 | 16 | `NumberInt(42)` | 32位整数 |
| Timestamp | 17 | `Timestamp(...)` | 内部时间戳（副本集用）|
| Int64 | 18 | `NumberLong(123456789)` | 64位整数 |
| Decimal128 | 19 | `NumberDecimal("9.99")` | 128位高精度小数 |
| MinKey | -1 | `MinKey()` | 最小值（比较用）|
| MaxKey | 127 | `MaxKey()` | 最大值（比较用）|

### ObjectId 结构解析

ObjectId 是 MongoDB 默认的主键类型，12字节，保证全局唯一性：

```
ObjectId: 64a7b3c9 d1e2f3 a4b5c6
          |______| |____| |_____|
             |        |      |
          4字节      3字节   5字节
          Unix时间戳  机器ID  进程ID + 随机计数器
          (秒级精度)
```

```javascript
// ObjectId 操作
const id = ObjectId("64a7b3c9d1e2f3a4b5c6d7e8")
id.getTimestamp()   // ISODate("2023-07-07T04:00:09Z") 提取创建时间
id.str              // "64a7b3c9d1e2f3a4b5c6d7e8" 十六进制字符串
ObjectId().equals(ObjectId()) // false，每次生成不同
```

---

## 1.5 MongoDB 存储引擎：WiredTiger

### 存储引擎演进历史

```
MongoDB 2.x/3.0:  MMAPv1（内存映射文件，已废弃）
                    |
MongoDB 3.0+:      WiredTiger 作为默认引擎引入
                    |
MongoDB 4.0+:      WiredTiger 唯一支持的引擎
                   （MMAPv1 彻底移除）
```

### WiredTiger 架构全景

```
+================================================================+
|                    WiredTiger 存储引擎                          |
+================================================================+
|                                                                |
|  应用层请求                                                     |
|      |                                                         |
|      v                                                         |
|  +------------------+   +------------------+                  |
|  | 读操作           |   | 写操作           |                  |
|  +------------------+   +------------------+                  |
|         |                       |                             |
|         v                       v                             |
|  +----------------------------------------------+            |
|  |           WiredTiger Cache（内存缓存）         |            |
|  |  默认 = max(256MB, (RAM-1GB) * 50%)           |            |
|  |  存放热点数据页，使用 LRU 淘汰策略             |            |
|  +----------------------------------------------+            |
|         |                       |                             |
|         v                       v                             |
|  +------------------+   +------------------+                  |
|  | 读取 -> B-Tree   |   | 写入 -> Journal  |                  |
|  | 数据文件         |   | (WAL预写日志)    |                  |
|  +------------------+   +------------------+                  |
|                                 |                             |
|                    +------------------+                        |
|                    | Checkpoint 机制  |                        |
|                    | 每60秒或2GB数据  |                        |
|                    | 将脏页刷盘       |                        |
|                    +------------------+                        |
|                                 |                             |
|                    +------------------+                        |
|                    | 数据文件(.wt)    |                        |
|                    | 块压缩存储       |                        |
|                    | snappy/zlib/zstd |                        |
|                    +------------------+                        |
|                                                                |
+================================================================+
```

### WiredTiger 核心特性详解

#### 1. B-Tree 存储结构

WiredTiger 使用 B-Tree（注意：不是 B+Tree）存储数据，每个集合和索引都有独立的 B-Tree 文件。

```
B-Tree 节点结构：
+------------------------------------------+
|  内部节点（Internal Page）               |
|  [Key1] [Key2] [Key3] ... [KeyN]         |
|    |       |       |           |         |
|    v       v       v           v         |
|  Leaf    Leaf    Leaf        Leaf        |
|  Page    Page    Page        Page        |
+------------------------------------------+

叶子页（Leaf Page）存储实际文档数据
默认页大小：32KB（可通过 blockCompressor 配置）
```

#### 2. 文档级并发控制（Document-Level Locking）

```
MySQL InnoDB:  行级锁（Row-level lock）
WiredTiger:    文档级锁（Document-level lock）

意义：多个写操作可以同时修改同一集合中的不同文档，
     极大提高并发写入性能。

锁粒度层次：
  全局锁（Global Lock）     <- 仅 DDL 操作
      |
  数据库锁（Database Lock） <- 创建/删除集合
      |
  集合锁（Collection Lock） <- 集合级操作
      |
  文档锁（Document Lock）   <- 普通读写（MVCC实现）
```

#### 3. MVCC（多版本并发控制）

WiredTiger 使用快照隔离（Snapshot Isolation）实现 MVCC：

```
时间线：
T1: 读事务开始，获取快照版本 V1
T2: 写事务修改文档 D，生成版本 V2
T3: T1 读取文档 D，仍然看到 V1（快照隔离）
T4: T2 提交
T5: 新的读事务看到 V2
```

#### 4. Journal（预写日志 WAL）

```
写操作流程：
1. 写入 Journal（WAL） <- 顺序写，极快
2. 写入 WiredTiger Cache（内存）
3. 定期 Checkpoint 刷入数据文件（随机写）

Journal 文件位置：/data/journal/
Journal 文件大小：100MB（循环使用）
同步间隔：默认每100ms fsync一次

崩溃恢复：
  启动时检测上次是否正常关闭
  -> 回放上次 Checkpoint 之后的 Journal 日志
  -> 保证数据不丢失（最多丢失100ms内的数据）
```

#### 5. 压缩算法

```javascript
// 查看/设置压缩算法
db.createCollection("myCol", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"  // 或 snappy, zlib
    }
  }
})
```

| 压缩算法 | 压缩率 | 速度 | 适用场景 |
|---------|--------|------|---------|
| snappy（默认）| 中等 | 最快 | 通用场景，CPU优先 |
| zlib | 高 | 中等 | 存储优先，冷数据 |
| zstd（推荐）| 最高 | 快 | MongoDB 4.2+，最佳平衡 |
| none | 无 | 最快 | 临时数据，无需压缩 |

#### 6. 内存缓存（Cache）配置

```yaml
# mongod.conf
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 4        # 固定 4GB 缓存
      # 或者使用默认：max(256MB, (RAM - 1GB) / 2)
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: zstd
    indexConfig:
      prefixCompression: true  # 索引前缀压缩
```

---

# Part 2: MongoDB CRUD 操作

## 2.1 插入操作

### insertOne - 插入单个文档

```javascript
// ==================== insertOne ====================
// 语法：db.collection.insertOne(document, options)

// 基础插入
db.users.insertOne({
  username: "zhangsan",
  email: "zhangsan@example.com",
  age: 28,
  createdAt: new Date()
})

// 返回结果：
// {
//   acknowledged: true,
//   insertedId: ObjectId("64a7b3c9d1e2f3a4b5c6d7e8")
// }

// 指定 _id（自定义主键）
db.users.insertOne({
  _id: "USER-001",   // 使用字符串作为主键
  username: "lisi",
  age: 25
})

// 带 writeConcern 选项
db.users.insertOne(
  { username: "wangwu", age: 30 },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
```

### insertMany - 批量插入

```javascript
// ==================== insertMany ====================
// 语法：db.collection.insertMany(documents, options)

// 有序插入（默认）- 遇到错误立即停止
db.products.insertMany([
  { name: "iPhone 15", price: 7999, category: "手机" },
  { name: "MacBook Pro", price: 14999, category: "电脑" },
  { name: "AirPods Pro", price: 1899, category: "耳机" }
])

// 返回结果：
// {
//   acknowledged: true,
//   insertedIds: {
//     '0': ObjectId("..."),
//     '1': ObjectId("..."),
//     '2': ObjectId("...")
//   }
// }

// 无序插入 - 遇到错误继续执行其他文档
db.products.insertMany(
  [
    { _id: 1, name: "产品A" },
    { _id: 1, name: "重复ID产品" },   // 这个会报错
    { _id: 2, name: "产品B" }          // 有序时不执行，无序时执行
  ],
  { ordered: false }                   // 无序模式
)
// 无序模式下：产品A和产品B插入成功，重复ID的报错，但不影响其他
```

---

## 2.2 查询操作

### find / findOne 基础

```javascript
// ==================== find ====================
// 语法：db.collection.find(query, projection)

// 查询所有文档
db.users.find()
db.users.find({})    // 等价

// findOne - 只返回第一条
db.users.findOne({ username: "zhangsan" })

// 格式化输出
db.users.find().pretty()

// 限制返回字段（投影）
// 1 表示返回，0 表示不返回（_id 默认返回）
db.users.find({}, { username: 1, email: 1, _id: 0 })

// 链式调用
db.users.find({ age: { $gt: 18 } })
  .sort({ age: -1 })    // 按age降序
  .skip(0)              // 跳过0条（分页用）
  .limit(10)            // 只返回10条

// 推荐计数方式
db.users.countDocuments({ age: { $gt: 18 } })
db.users.estimatedDocumentCount()  // 估算总数（快）
```

### 比较操作符

```javascript
// $eq - 等于（默认）
db.users.find({ age: { $eq: 28 } })
db.users.find({ age: 28 })            // 等价简写

// $ne - 不等于
db.users.find({ status: { $ne: "banned" } })

// $gt / $gte - 大于 / 大于等于
db.users.find({ age: { $gt: 18 } })
db.users.find({ age: { $gte: 18 } })
db.users.find({ score: { $gt: 60, $lte: 100 } })  // 范围查询

// $lt / $lte - 小于 / 小于等于
db.users.find({ price: { $lt: 1000 } })
db.users.find({ price: { $lte: 999 } })

// $in - 包含（相当于 SQL 的 IN）
db.users.find({ status: { $in: ["active", "pending"] } })
db.users.find({ age: { $in: [18, 25, 30] } })

// $nin - 不包含（相当于 SQL 的 NOT IN）
db.users.find({ role: { $nin: ["admin", "superadmin"] } })

// $exists - 字段是否存在
db.users.find({ nickname: { $exists: true } })   // 存在 nickname 字段
db.users.find({ nickname: { $exists: false } })  // 不存在 nickname 字段

// $type - 字段类型过滤
db.users.find({ age: { $type: "int" } })
db.users.find({ age: { $type: ["int", "double"] } })  // 多类型

// $regex - 正则表达式
db.users.find({ email: { $regex: /gmail\.com$/i } })
db.users.find({ username: { $regex: "^zhang", $options: "i" } })

// $mod - 取模运算
db.items.find({ quantity: { $mod: [2, 0] } })  // quantity 为偶数
```

### 逻辑操作符

```javascript
// $and - 逻辑与（同一字段的多条件默认就是and）
db.users.find({
  $and: [
    { age: { $gt: 18 } },
    { age: { $lt: 60 } },
    { status: "active" }
  ]
})
// 同字段多条件，可以简写：
db.users.find({ age: { $gt: 18, $lt: 60 }, status: "active" })

// $or - 逻辑或
db.users.find({
  $or: [
    { email: { $regex: "@gmail.com$" } },
    { email: { $regex: "@hotmail.com$" } }
  ]
})

// $not - 逻辑非
db.users.find({ age: { $not: { $gt: 30 } } })  // age <= 30

// $nor - 都不满足
db.users.find({
  $nor: [
    { status: "banned" },
    { age: { $lt: 18 } }
  ]
})

// 复合逻辑示例
db.orders.find({
  $and: [
    { status: { $in: ["pending", "processing"] } },
    {
      $or: [
        { amount: { $gt: 1000 } },
        { isPriority: true }
      ]
    }
  ]
})
```

### 查询嵌套文档

```javascript
// 准备数据
db.users.insertOne({
  username: "zhangsan",
  address: {
    city: "上海",
    district: "浦东新区",
    zipCode: "200120"
  }
})

// ==================== 嵌套文档查询 ====================

// 方式1：点表示法（推荐）
db.users.find({ "address.city": "上海" })
db.users.find({ "address.zipCode": { $regex: "^200" } })

// 方式2：精确匹配整个嵌套文档（字段顺序必须完全一致！）
db.users.find({ address: { city: "上海", district: "浦东新区", zipCode: "200120" } })
// 注意：顺序不对就匹配不到，很容易出错，不推荐

// 多层嵌套
db.orders.find({ "shipping.address.city": "北京" })

// 嵌套文档 + 逻辑操作符
db.users.find({
  "address.city": "上海",
  "address.district": { $in: ["浦东新区", "徐汇区"] }
})
```

### 查询数组

```javascript
// 准备数据
db.articles.insertMany([
  { title: "MongoDB入门", tags: ["mongodb", "nosql", "database"] },
  { title: "Redis详解", tags: ["redis", "nosql", "cache"] },
  { title: "MySQL优化", tags: ["mysql", "sql", "database"] }
])

// ==================== 数组查询 ====================

// 包含指定元素（至少包含其中一个）
db.articles.find({ tags: "nosql" })
// 结果：MongoDB入门 + Redis详解

// 精确匹配整个数组（顺序必须一致）
db.articles.find({ tags: ["mongodb", "nosql", "database"] })

// $all - 必须包含所有指定元素（顺序无关）
db.articles.find({ tags: { $all: ["nosql", "database"] } })
// 结果：MongoDB入门

// $size - 数组长度
db.articles.find({ tags: { $size: 3 } })

// $elemMatch - 数组中至少一个元素满足所有条件
db.orders.find({
  items: {
    $elemMatch: {
      productId: "P001",
      quantity: { $gt: 2 }
    }
  }
})

// 数组索引访问（固定位置元素）
db.scores.find({ "scores.0": { $gt: 90 } })  // 第一个分数大于90

// 嵌套文档数组
db.orders.insertOne({
  orderId: "ORD-001",
  items: [
    { productId: "P001", name: "手机", qty: 1, price: 4999 },
    { productId: "P002", name: "保护壳", qty: 2, price: 49 }
  ]
})

// 查询数组中嵌套文档的字段（点表示法）
db.orders.find({ "items.productId": "P001" })

// $elemMatch 确保同一元素满足多个条件
db.orders.find({
  items: {
    $elemMatch: { productId: "P001", qty: { $gte: 1 } }
  }
})
```

---

## 2.3 更新操作

### updateOne / updateMany

```javascript
// ==================== updateOne ====================
// 语法：db.collection.updateOne(filter, update, options)

// $set - 设置字段值（字段不存在则创建）
db.users.updateOne(
  { username: "zhangsan" },
  { $set: { age: 29, lastLoginAt: new Date() } }
)

// $unset - 删除字段
db.users.updateOne(
  { username: "zhangsan" },
  { $unset: { oldField: "" } }  // 值填空字符串即可
)

// $inc - 数值增减
db.users.updateOne(
  { username: "zhangsan" },
  { $inc: { loginCount: 1, score: -5 } }  // 登录次数+1，积分-5
)

// $mul - 乘以因子
db.products.updateOne(
  { _id: "P001" },
  { $mul: { price: 0.9 } }  // 打九折
)

// $min / $max - 只在新值更小/更大时才更新
db.scores.updateOne(
  { userId: "U001" },
  { $min: { lowestScore: 85 } }   // 如果85比当前值小，则更新
)
db.scores.updateOne(
  { userId: "U001" },
  { $max: { highestScore: 95 } }  // 如果95比当前值大，则更新
)

// $rename - 重命名字段
db.users.updateOne(
  { username: "zhangsan" },
  { $rename: { "oldName": "newName" } }
)

// $currentDate - 设置为当前时间
db.users.updateOne(
  { username: "zhangsan" },
  { $currentDate: { updatedAt: true } }  // true表示Date类型
)

// ==================== upsert ====================
// upsert: true - 找不到则插入
db.users.updateOne(
  { username: "newuser" },
  {
    $set: { email: "newuser@example.com" },
    $setOnInsert: { createdAt: new Date() }  // 只在插入时设置
  },
  { upsert: true }
)

// ==================== updateMany ====================
// 更新所有匹配文档
db.users.updateMany(
  { status: "inactive" },
  { $set: { status: "active", reactivatedAt: new Date() } }
)

// ==================== 数组更新操作符 ====================

// $push - 追加元素到数组
db.articles.updateOne(
  { _id: "A001" },
  { $push: { tags: "javascript" } }
)

// $push + $each + $sort + $slice
db.articles.updateOne(
  { _id: "A001" },
  {
    $push: {
      tags: {
        $each: ["vue", "react"],  // 追加多个元素
        $sort: 1,                  // 排序（1升序，-1降序）
        $slice: 10                 // 保留前N个元素（负数保留后N个）
      }
    }
  }
)

// $addToSet - 追加（集合语义，不重复）
db.articles.updateOne(
  { _id: "A001" },
  { $addToSet: { tags: "mongodb" } }
)
db.articles.updateOne(
  { _id: "A001" },
  { $addToSet: { tags: { $each: ["a", "b", "c"] } } }
)

// $pull - 删除匹配值
db.articles.updateOne(
  { _id: "A001" },
  { $pull: { tags: "nosql" } }
)
db.orders.updateOne(
  { _id: "ORD001" },
  { $pull: { items: { qty: 0 } } }  // 删除数量为0的item
)

// $pop - 删除数组首/尾元素
db.articles.updateOne(
  { _id: "A001" },
  { $pop: { tags: 1 } }   // 删除最后一个（-1删第一个）
)

// 定位操作符 $ - 更新数组中匹配的元素
db.orders.updateOne(
  { "items.productId": "P001" },
  { $set: { "items.$.price": 4599 } }    // $ 代表匹配的数组元素
)

// $[identifier] - 过滤的定位操作符（更新所有匹配的数组元素）
db.orders.updateMany(
  { },
  { $inc: { "items.$[elem].price": 100 } },
  { arrayFilters: [{ "elem.category": "手机" }] }
)
```

### findOneAndUpdate / findOneAndDelete

```javascript
// ==================== findOneAndUpdate ====================
// 查找并更新，返回文档（原子操作）
// returnDocument: "before"（默认，返回更新前）/ "after"（返回更新后）

const result = db.users.findOneAndUpdate(
  { username: "zhangsan" },
  { $inc: { loginCount: 1 }, $set: { lastLoginAt: new Date() } },
  {
    returnDocument: "after",      // 返回更新后的文档
    projection: { password: 0 }, // 不返回密码字段
    upsert: false
  }
)
// result: 更新后的完整文档

// ==================== findOneAndDelete ====================
// 查找并删除，返回被删除的文档
const deleted = db.queue.findOneAndDelete(
  { status: "pending" },
  { sort: { priority: -1, createdAt: 1 } }  // 按优先级最高、时间最早
)
// 经典"消费队列"模式

// ==================== findOneAndReplace ====================
// 查找并替换整个文档
db.users.findOneAndReplace(
  { username: "zhangsan" },
  { username: "zhangsan", email: "new@example.com", updatedAt: new Date() },
  { returnDocument: "after" }
)
// 注意：replace 会替换整个文档（_id保留），不是更新部分字段
```

---

## 2.4 删除操作

```javascript
// ==================== deleteOne ====================
// 删除第一条匹配文档
db.users.deleteOne({ username: "spammer" })
// 返回结果：{ acknowledged: true, deletedCount: 1 }

// ==================== deleteMany ====================
// 删除所有匹配文档
db.logs.deleteMany({ createdAt: { $lt: new Date("2023-01-01") } })

// 删除所有文档（保留集合和索引）
db.logs.deleteMany({})

// 彻底删除集合（包括索引）
db.logs.drop()
```

---

## 2.5 bulkWrite 批量操作

bulkWrite 允许在一次请求中执行多种操作（插入/更新/删除的混合），减少网络往返。

```javascript
// ==================== bulkWrite ====================
const result = db.users.bulkWrite([
  // 插入操作
  {
    insertOne: {
      document: { username: "user1", age: 20 }
    }
  },
  // 更新单个
  {
    updateOne: {
      filter: { username: "zhangsan" },
      update: { $inc: { loginCount: 1 } }
    }
  },
  // 更新多个
  {
    updateMany: {
      filter: { status: "inactive" },
      update: { $set: { status: "active" } }
    }
  },
  // 替换
  {
    replaceOne: {
      filter: { username: "olduser" },
      replacement: { username: "olduser", migrated: true },
      upsert: false
    }
  },
  // 删除单个
  {
    deleteOne: {
      filter: { username: "banned_user" }
    }
  },
  // 删除多个
  {
    deleteMany: {
      filter: { createdAt: { $lt: new Date("2020-01-01") } }
    }
  }
],
{
  ordered: true,     // 有序执行（默认true）
  writeConcern: { w: "majority" }
})

// 返回结果：
// {
//   acknowledged: true,
//   insertedCount: 1,
//   matchedCount: 5,
//   modifiedCount: 5,
//   deletedCount: 3,
//   upsertedCount: 0
// }
```

---

## 2.6 完整 CRUD 示例（MongoDB Shell）

```javascript
// ==================== 完整实战示例：用户管理系统 ====================

// 1. 切换到数据库
use user_management_db

// 2. 批量插入测试数据
db.users.insertMany([
  {
    username: "zhangsan",
    email: "zhangsan@example.com",
    age: 28,
    role: "user",
    status: "active",
    tags: ["java", "mongodb"],
    address: { city: "上海", district: "浦东新区" },
    loginCount: 15,
    createdAt: ISODate("2023-06-01T00:00:00Z"),
    lastLoginAt: ISODate("2024-01-10T08:30:00Z")
  },
  {
    username: "lisi",
    email: "lisi@example.com",
    age: 32,
    role: "admin",
    status: "active",
    tags: ["python", "docker"],
    address: { city: "北京", district: "朝阳区" },
    loginCount: 200,
    createdAt: ISODate("2022-03-15T00:00:00Z"),
    lastLoginAt: ISODate("2024-01-11T09:00:00Z")
  },
  {
    username: "wangwu",
    email: "wangwu@gmail.com",
    age: 24,
    role: "user",
    status: "inactive",
    tags: ["javascript", "vue"],
    address: { city: "深圳", district: "南山区" },
    loginCount: 3,
    createdAt: ISODate("2023-11-20T00:00:00Z"),
    lastLoginAt: ISODate("2023-12-01T10:00:00Z")
  }
])

// 3. 各种查询
// 3.1 查询上海的活跃用户，只返回用户名和邮箱
db.users.find(
  { "address.city": "上海", status: "active" },
  { username: 1, email: 1, _id: 0 }
)

// 3.2 查询年龄在25-35之间，有java标签的用户
db.users.find({
  age: { $gte: 25, $lte: 35 },
  tags: "java"
})

// 3.3 模糊搜索gmail邮箱
db.users.find({ email: /gmail\.com$/ })

// 3.4 查询登录次数前3名的用户
db.users.find().sort({ loginCount: -1 }).limit(3)

// 4. 更新操作
// 4.1 将所有inactive用户激活
db.users.updateMany(
  { status: "inactive" },
  {
    $set: { status: "active", reactivatedAt: new Date() },
    $inc: { reactivationCount: 1 }
  }
)

// 4.2 为张三添加新标签
db.users.updateOne(
  { username: "zhangsan" },
  { $addToSet: { tags: { $each: ["spring", "kubernetes"] } } }
)

// 5. 索引创建
db.users.createIndex({ email: 1 }, { unique: true })
db.users.createIndex({ username: 1 }, { unique: true })
db.users.createIndex({ status: 1, createdAt: -1 })
db.users.createIndex({ "address.city": 1 })
db.users.getIndexes()
```


---

# Part 3: 聚合管道（Aggregation Pipeline）深度解析

## 3.1 聚合管道概念

聚合管道（Aggregation Pipeline）是 MongoDB 强大的数据处理框架，类比 Linux 管道：

```
Linux 管道：
  cat access.log | grep "ERROR" | awk '{print $5}' | sort | uniq -c

MongoDB 聚合管道：
  db.orders.aggregate([
    { $match: { status: "completed" } },
    { $group: { _id: "$userId", total: { $sum: "$amount" } } },
    { $sort: { total: -1 } },
    { $limit: 10 }
  ])
```

```
聚合管道数据流：

  文档集合
      |
      v
  [Stage 1: $match]    <- 过滤，减少数据量
      |
      v
  [Stage 2: $group]    <- 分组聚合
      |
      v
  [Stage 3: $project]  <- 字段转换
      |
      v
  [Stage 4: $sort]     <- 排序
      |
      v
  [Stage 5: $limit]    <- 截取
      |
      v
  输出结果
```

**重要限制（默认）：**
- 单个聚合管道内存使用上限：**100MB**（可通过 `allowDiskUse: true` 突破）
- 返回结果集最大：**16MB**（受 BSON 文档大小限制）

---

## 3.2 核心阶段详解

### $match - 过滤阶段

```javascript
// $match 支持所有查询操作符
db.orders.aggregate([
  {
    $match: {
      status: { $in: ["completed", "shipped"] },
      amount: { $gt: 100 },
      createdAt: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2024-02-01T00:00:00Z")
      }
    }
  }
])

// $match 要尽量放在管道前面，利用索引减少数据量
// 原则：越早过滤，后续阶段处理的数据越少
```

### $group - 分组聚合

```javascript
// $group 语法：
// { $group: { _id: <分组键>, <字段>: { <累加器>: <表达式> } } }

// 基础分组 - 按状态统计订单数和总金额
db.orders.aggregate([
  {
    $group: {
      _id: "$status",                          // 分组键
      count: { $sum: 1 },                      // 计数
      totalAmount: { $sum: "$amount" },        // 求和
      avgAmount: { $avg: "$amount" },          // 平均值
      maxAmount: { $max: "$amount" },          // 最大值
      minAmount: { $min: "$amount" },          // 最小值
      firstOrder: { $first: "$createdAt" },    // 第一个值
      lastOrder: { $last: "$createdAt" }       // 最后一个值
    }
  }
])

// 多字段分组
db.orders.aggregate([
  {
    $group: {
      _id: { year: { $year: "$createdAt" }, month: { $month: "$createdAt" } },
      revenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { "_id.year": 1, "_id.month": 1 } }
])

// $push - 将值收集到数组
db.orders.aggregate([
  {
    $group: {
      _id: "$userId",
      orderIds: { $push: "$_id" },
      productsBought: { $push: "$productId" }
    }
  }
])

// $addToSet - 收集唯一值到数组
db.orders.aggregate([
  {
    $group: {
      _id: "$userId",
      uniqueProducts: { $addToSet: "$productId" }  // 去重
    }
  }
])

// _id 为 null - 对整个集合聚合
db.orders.aggregate([
  {
    $group: {
      _id: null,
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$amount" }
    }
  }
])
```

### $project - 投影/变换

```javascript
// $project 语法：
// 包含字段：1 或 true
// 排除字段：0 或 false
// 计算新字段：表达式

db.orders.aggregate([
  {
    $project: {
      _id: 0,                              // 排除 _id
      orderId: 1,                          // 包含 orderId
      customerName: "$customer.name",      // 重命名（从嵌套字段提取）

      // 字符串操作
      upperName: { $toUpper: "$productName" },
      namePrefix: { $substr: ["$productName", 0, 5] },
      fullLabel: { $concat: ["订单:", "$orderId", "-", "$status"] },

      // 数学运算
      discountedPrice: { $multiply: ["$price", 0.9] },
      profitMargin: { $divide: [{ $subtract: ["$price", "$cost"] }, "$price"] },

      // 条件表达式
      statusLabel: {
        $switch: {
          branches: [
            { case: { $eq: ["$status", "completed"] }, then: "已完成" },
            { case: { $eq: ["$status", "pending"] }, then: "待处理" },
            { case: { $eq: ["$status", "cancelled"] }, then: "已取消" }
          ],
          default: "未知状态"
        }
      },

      // 三元运算
      isHighValue: { $cond: { if: { $gt: ["$amount", 1000] }, then: true, else: false } },
      isVip: { $cond: [{ $gte: ["$orderCount", 10] }, "VIP", "普通"] },

      // $ifNull - 空值处理
      displayName: { $ifNull: ["$nickname", "$username"] },

      // 日期操作
      year: { $year: "$createdAt" },
      month: { $month: "$createdAt" },
      formattedDate: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },

      // 数组操作
      firstTag: { $arrayElemAt: ["$tags", 0] },
      tagCount: { $size: "$tags" },
      hasJava: { $in: ["java", "$tags"] }
    }
  }
])
```

### $sort、$limit、$skip

```javascript
// $sort
db.orders.aggregate([
  {
    $sort: {
      amount: -1,          // 金额降序
      createdAt: 1         // 时间升序
    }
  }
])

// 分页模式
const page = 2
const pageSize = 10

db.products.aggregate([
  { $match: { status: "active" } },
  { $sort: { createdAt: -1 } },
  { $skip: (page - 1) * pageSize },   // 跳过前面的页
  { $limit: pageSize }                 // 取当前页数据
])

// 注意：$skip 在大偏移量时性能较差（扫描后丢弃）
// 对于深度分页，推荐使用"游标分页"（基于上一页最后一条文档的ID）
```

### $lookup - 关联查询（JOIN）

```javascript
// 基础 $lookup（类比 LEFT JOIN）
db.orders.aggregate([
  {
    $lookup: {
      from: "users",           // 关联的集合名
      localField: "userId",    // 当前集合的字段
      foreignField: "_id",     // 关联集合的字段
      as: "userInfo"           // 结果字段名（数组）
    }
  },
  {
    // 将数组展开为单个文档
    $unwind: {
      path: "$userInfo",
      preserveNullAndEmptyArrays: true   // 保留没有关联的文档
    }
  }
])

// 高级 $lookup - 带条件的子管道（更灵活）
db.orders.aggregate([
  {
    $lookup: {
      from: "orderItems",
      let: { orderId: "$_id", minQty: 1 },  // 定义变量
      pipeline: [
        {
          $match: {
            $expr: {
              $and: [
                { $eq: ["$orderId", "$$orderId"] },  // $$引用外部变量
                { $gte: ["$quantity", "$$minQty"] }
              ]
            }
          }
        },
        { $project: { productName: 1, quantity: 1, price: 1 } }
      ],
      as: "items"
    }
  }
])

// 多级关联（联3张表）
db.orders.aggregate([
  // 第一级：关联用户
  {
    $lookup: {
      from: "users",
      localField: "userId",
      foreignField: "_id",
      as: "user"
    }
  },
  { $unwind: "$user" },
  // 第二级：关联商品
  {
    $lookup: {
      from: "products",
      localField: "productId",
      foreignField: "_id",
      as: "product"
    }
  },
  { $unwind: "$product" }
])
```

### $unwind - 展开数组

```javascript
// $unwind 将数组中每个元素单独展开为一条文档
db.articles.aggregate([
  { $unwind: "$tags" }
])
// 输出3条文档：
// { title: "MongoDB指南", tags: "mongodb" }
// { title: "MongoDB指南", tags: "nosql" }
// { title: "MongoDB指南", tags: "database" }

// 带选项的 $unwind
db.articles.aggregate([
  {
    $unwind: {
      path: "$tags",
      includeArrayIndex: "tagIndex",           // 包含数组索引
      preserveNullAndEmptyArrays: true         // 保留空数组/null的文档
    }
  }
])

// 经典用法：$unwind + $group 统计标签频率
db.articles.aggregate([
  { $unwind: "$tags" },
  { $group: { _id: "$tags", count: { $sum: 1 } } },
  { $sort: { count: -1 } },
  { $limit: 10 }
])
```

### $addFields / $set - 添加字段

```javascript
// $addFields 向文档添加新字段（不影响原有字段）
// $set 是 $addFields 的别名（MongoDB 4.2+）
db.orders.aggregate([
  {
    $addFields: {
      totalWithTax: { $multiply: ["$amount", 1.1] },
      isExpensive: { $gt: ["$amount", 1000] },
      processedAt: new Date()
    }
  }
])
```

### $count - 计数

```javascript
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $count: "completedOrderCount" }
])
// 输出：{ completedOrderCount: 1234 }
```

### $facet - 多维度聚合

$facet 允许在单次管道中并行执行多个子管道，适合需要多维度统计的场景：

```javascript
db.products.aggregate([
  { $match: { status: "active" } },
  {
    $facet: {
      // 维度1：按分类统计数量
      "byCategory": [
        { $group: { _id: "$category", count: { $sum: 1 } } },
        { $sort: { count: -1 } }
      ],
      // 维度2：价格区间统计
      "priceRange": [
        {
          $bucket: {
            groupBy: "$price",
            boundaries: [0, 100, 500, 1000, 5000, 100000],
            default: "其他",
            output: { count: { $sum: 1 }, avgPrice: { $avg: "$price" } }
          }
        }
      ],
      // 维度3：总计
      "totalStats": [
        { $group: { _id: null, total: { $sum: 1 }, avgPrice: { $avg: "$price" } } }
      ]
    }
  }
])
```

### $bucket / $bucketAuto - 分桶统计

```javascript
// $bucket - 手动指定边界
db.products.aggregate([
  {
    $bucket: {
      groupBy: "$price",
      boundaries: [0, 100, 500, 1000, 5000],   // 分桶边界
      default: "5000以上",
      output: {
        count: { $sum: 1 },
        products: { $push: "$name" },
        avgPrice: { $avg: "$price" }
      }
    }
  }
])
// 输出：
// { _id: 0, count: 5, ... }      // 0 <= price < 100
// { _id: 100, count: 12, ... }   // 100 <= price < 500
// { _id: 500, count: 8, ... }    // 500 <= price < 1000

// $bucketAuto - 自动均匀分桶
db.products.aggregate([
  {
    $bucketAuto: {
      groupBy: "$price",
      buckets: 5,          // 自动分为5个桶（尽量均匀）
      output: {
        count: { $sum: 1 },
        minPrice: { $min: "$price" },
        maxPrice: { $max: "$price" }
      }
    }
  }
])
```

### $graphLookup - 图遍历（递归查询）

```javascript
// 场景：组织架构树（员工-上级关系）
db.employees.insertMany([
  { _id: 1, name: "CEO", reportsTo: null },
  { _id: 2, name: "CTO", reportsTo: 1 },
  { _id: 3, name: "CFO", reportsTo: 1 },
  { _id: 4, name: "架构师", reportsTo: 2 },
  { _id: 5, name: "高级工程师", reportsTo: 4 },
  { _id: 6, name: "工程师", reportsTo: 5 }
])

// 查找某人的完整上级链（从下往上遍历）
db.employees.aggregate([
  { $match: { name: "工程师" } },
  {
    $graphLookup: {
      from: "employees",
      startWith: "$reportsTo",        // 起始值
      connectFromField: "reportsTo",  // 当前文档中用于连接的字段
      connectToField: "_id",          // 目标文档中匹配的字段
      as: "reportingHierarchy",       // 结果字段名
      maxDepth: 10,                   // 最大递归深度
      depthField: "depth"             // 记录递归深度的字段名
    }
  }
])
// 输出：工程师 -> 高级工程师 -> 架构师 -> CTO -> CEO 的完整路径
```

---

## 3.3 聚合管道优化

```javascript
// ==================== 优化原则 ====================

// 1. $match 尽量前置（利用索引，减少数据量）
// 不好的写法：
db.orders.aggregate([
  { $sort: { createdAt: -1 } },
  { $match: { status: "completed" } }  // match在sort后，无法利用索引
])

// 好的写法：
db.orders.aggregate([
  { $match: { status: "completed" } },  // 先过滤
  { $sort: { createdAt: -1 } }          // 再排序
])

// 2. $project 尽早减少字段（减少内存占用）
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $project: { userId: 1, amount: 1, createdAt: 1 } },  // 早期投影
  { $group: { _id: "$userId", total: { $sum: "$amount" } } }
])

// 3. $sort + $limit 组合优化（Top-K 算法）
// MongoDB 会自动将 $sort + $limit 合并为 Top-K 排序，只保留前K个
db.orders.aggregate([
  { $sort: { amount: -1 } },
  { $limit: 10 }   // 只需要前10，MongoDB不会对全部数据排序
])

// 4. 允许磁盘使用（大数据量时）
db.orders.aggregate(
  [ /* pipeline */ ],
  { allowDiskUse: true }  // 超过100MB时，使用磁盘临时文件
)

// 5. 使用 hint 强制指定索引
db.orders.aggregate(
  [{ $match: { status: "completed" } }],
  { hint: { status: 1, createdAt: -1 } }
)
```

---

## 3.4 完整案例：电商订单统计仪表板

```javascript
// ==================== 仪表板查询 ====================

// 1. 本月概览：总收入、订单数、平均客单价
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: {
        $gte: ISODate("2024-01-01T00:00:00Z"),
        $lt: ISODate("2024-02-01T00:00:00Z")
      }
    }
  },
  {
    $group: {
      _id: null,
      totalRevenue: { $sum: "$amount" },
      orderCount: { $sum: 1 },
      avgOrderValue: { $avg: "$amount" },
      maxOrder: { $max: "$amount" },
      uniqueCustomers: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      _id: 0,
      totalRevenue: { $round: ["$totalRevenue", 2] },
      orderCount: 1,
      avgOrderValue: { $round: ["$avgOrderValue", 2] },
      maxOrder: 1,
      customerCount: { $size: "$uniqueCustomers" }
    }
  }
])

// 2. 每日销售趋势（最近30天）
db.orders.aggregate([
  {
    $match: {
      status: "completed",
      createdAt: { $gte: new Date(Date.now() - 30 * 24 * 3600 * 1000) }
    }
  },
  {
    $group: {
      _id: { $dateToString: { format: "%Y-%m-%d", date: "$createdAt" } },
      revenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])

// 3. 各城市销售排行（Top 10）
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $group: {
      _id: "$userCity",
      revenue: { $sum: "$amount" },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { revenue: -1 } },
  { $limit: 10 },
  {
    $project: {
      city: "$_id",
      revenue: { $round: ["$revenue", 0] },
      orderCount: 1,
      _id: 0
    }
  }
])

// 4. 品类销售分析（展开 items 数组）
db.orders.aggregate([
  { $match: { status: "completed" } },
  { $unwind: "$items" },
  {
    $group: {
      _id: "$items.category",
      totalQty: { $sum: "$items.qty" },
      totalRevenue: { $sum: { $multiply: ["$items.qty", "$items.price"] } },
      orderCount: { $sum: 1 }
    }
  },
  { $sort: { totalRevenue: -1 } }
])

// 5. 支付方式分布 + 价格区间分布（$facet 多维度）
db.orders.aggregate([
  { $match: { status: "completed" } },
  {
    $facet: {
      paymentStats: [
        { $group: { _id: "$paymentMethod", count: { $sum: 1 }, revenue: { $sum: "$amount" } } },
        { $sort: { count: -1 } }
      ],
      amountDistribution: [
        {
          $bucket: {
            groupBy: "$amount",
            boundaries: [0, 100, 300, 500, 1000, 3000, 100000],
            default: "其他",
            output: { count: { $sum: 1 } }
          }
        }
      ],
      userFrequency: [
        { $group: { _id: "$userId", orderCount: { $sum: 1 } } },
        {
          $bucket: {
            groupBy: "$orderCount",
            boundaries: [1, 2, 5, 10, 50],
            default: "50次以上",
            output: { userCount: { $sum: 1 } }
          }
        }
      ]
    }
  }
])
```

---

# Part 4: 索引深度解析

## 4.1 索引基础

索引是提升查询性能的关键，MongoDB 索引基于 **B-Tree** 结构（WiredTiger 底层），存储指定字段的有序副本
及文档位置指针。

```
没有索引（全集合扫描）：
  Query: { age: 28 }

  文档1 -> 读取 -> age=25 不匹配
  文档2 -> 读取 -> age=28 匹配
  文档3 -> 读取 -> age=31 不匹配
  ... 扫描所有文档 (O(n))

有索引（B-Tree查找）：
  Index on "age":
  [22] -> [25] -> [28] -> [31] -> [35]
                  |
             直接定位 (O(log n))
```

**`_id` 索引：** 每个集合自动创建，唯一，不可删除。

---

## 4.2 索引类型详解

### 单字段索引

```javascript
// 创建升序索引（1）
db.users.createIndex({ age: 1 })

// 创建降序索引（-1）
db.users.createIndex({ createdAt: -1 })

// 带选项的索引
db.users.createIndex(
  { email: 1 },
  {
    name: "idx_email_unique",    // 索引名称
    unique: true,                // 唯一约束
    background: true             // 后台构建（不阻塞读写）
  }
)

// 对嵌套字段创建索引
db.users.createIndex({ "address.city": 1 })

// 查看索引
db.users.getIndexes()
db.users.aggregate([{ $indexStats: {} }])  // 索引使用统计

// 删除索引
db.users.dropIndex("idx_email_unique")
db.users.dropIndex({ age: 1 })
db.users.dropIndexes()  // 删除所有索引（_id除外）
```

### 复合索引

```javascript
// 复合索引 - 多字段联合索引
db.orders.createIndex({ userId: 1, status: 1, createdAt: -1 })

// ==================== ESR 规则 ====================
// E - Equality（等值查询字段）放最前
// S - Sort（排序字段）放中间
// R - Range（范围查询字段）放最后

// 查询场景：db.orders.find({userId: "U001", status: "completed"}).sort({createdAt: -1})
// 按 ESR 规则：
// E: userId（等值）-> 放第1位
// E: status（等值）-> 放第2位
// S: createdAt（排序）-> 放第3位
// R: amount（范围，若有）-> 放最后

// ==================== 索引前缀原则 ====================
// 对于复合索引 {a: 1, b: 1, c: 1}，以下查询都能使用：
// 能用：find({a: ...})
// 能用：find({a: ..., b: ...})
// 能用：find({a: ..., b: ..., c: ...})
// 不用：find({b: ...})         // 跳过了a，不能使用索引（全扫描）
// 不用：find({b: ..., c: ...}) // 跳过了a，不能使用索引
// 能用：find({a: ..., c: ...}) // 可以用索引，但只能用到a的部分
```

### 多键索引（Multikey Index）

```javascript
// 当索引字段是数组时，MongoDB 自动创建多键索引
// 数组中每个元素都会在索引中有一条记录

db.articles.createIndex({ tags: 1 })
// 文档 { tags: ["mongodb", "nosql", "database"] }
// 索引中产生3个条目：
//   "database" -> 文档指针
//   "mongodb" -> 文档指针
//   "nosql" -> 文档指针

// 注意：复合索引最多只能有一个数组字段
db.articles.createIndex({ tags: 1, views: 1 })  // 正确：tags是数组，views不是
// db.articles.createIndex({ tags: 1, authors: 1 }) // 报错，两个都是数组
```

### 文本索引（Text Index）

```javascript
// 每个集合只能有一个文本索引
db.articles.createIndex(
  {
    title: "text",
    content: "text",
    summary: "text"
  },
  {
    name: "idx_full_text",
    weights: { title: 10, summary: 5, content: 1 },  // 权重
    default_language: "none",     // 禁用词干处理（中文必须用 none）
    language_override: "language" // 支持按文档指定语言
  }
)

// 全文搜索查询
db.articles.find({ $text: { $search: "mongodb 索引" } })

// 多词搜索（OR关系）
db.articles.find({ $text: { $search: "mongodb redis" } })

// 精确短语（用引号）
db.articles.find({ $text: { $search: "\"mongodb索引\"" } })

// 排除某词
db.articles.find({ $text: { $search: "mongodb -redis" } })

// 按相关性分数排序
db.articles.find(
  { $text: { $search: "mongodb" } },
  { score: { $meta: "textScore" } }  // 投影相关性分数
).sort({ score: { $meta: "textScore" } })

// 注意：MongoDB 文本索引对中文支持有限
// 中文场景推荐使用 Elasticsearch
```

### 地理空间索引（2dsphere）

```javascript
// 2dsphere 索引 - 支持球面几何（适合地球坐标）
db.locations.createIndex({ location: "2dsphere" })

// 数据格式：GeoJSON
db.locations.insertMany([
  {
    name: "上海东方明珠",
    location: { type: "Point", coordinates: [121.4995, 31.2397] }  // [经度, 纬度]
  },
  {
    name: "上海外滩",
    location: { type: "Point", coordinates: [121.4899, 31.2357] }
  },
  {
    name: "北京天安门",
    location: { type: "Point", coordinates: [116.3912, 39.9077] }
  }
])

// 查找附近的地点（$near）
db.locations.find({
  location: {
    $near: {
      $geometry: { type: "Point", coordinates: [121.4995, 31.2397] },
      $maxDistance: 2000,   // 最大距离（米）
      $minDistance: 100     // 最小距离（米）
    }
  }
})

// 查找区域内的地点（$geoWithin）
db.locations.find({
  location: {
    $geoWithin: {
      $geometry: {
        type: "Polygon",
        coordinates: [[
          [121.4, 31.2],
          [121.6, 31.2],
          [121.6, 31.3],
          [121.4, 31.3],
          [121.4, 31.2]    // 闭合多边形
        ]]
      }
    }
  }
})

// $geoIntersects - 查找与指定几何体相交的文档
db.deliveryZones.find({
  area: {
    $geoIntersects: {
      $geometry: { type: "Point", coordinates: [121.4995, 31.2397] }
    }
  }
})
```

### 哈希索引（Hashed Index）

```javascript
// 哈希索引 - 主要用于分片键
// 存储字段值的哈希，支持等值查询，不支持范围查询
db.users.createIndex({ userId: "hashed" })

// 哈希分片（创建分片集合时指定）
sh.shardCollection("mydb.users", { userId: "hashed" })

// 注意：哈希索引不支持范围查询，如 $gt/$lt
```

### TTL 索引（自动过期删除）

```javascript
// TTL 索引 - 字段必须是 Date 类型或包含 Date 的数组

// 方式1：到达 expireAt 时间点立即删除（expireAfterSeconds=0）
db.sessions.createIndex(
  { expireAt: 1 },
  { expireAfterSeconds: 0 }
)

// 方式2：从创建时间起7天后删除
db.logs.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 7 * 24 * 3600 }  // 7天
)

// TTL 删除机制：
// MongoDB 后台线程每60秒扫描一次TTL索引，删除过期文档
// 精度约 +-60秒，不适合精确定时任务

// 修改 TTL 时间（不能先删后建，需用 collMod）
db.runCommand({
  collMod: "sessions",
  index: {
    keyPattern: { expireAt: 1 },
    expireAfterSeconds: 3600   // 改为1小时
  }
})
```

### 稀疏索引（Sparse Index）

```javascript
// 稀疏索引：只索引包含该字段的文档（不索引字段不存在的文档）
// 适合：可选字段，只有部分文档有该字段

db.users.createIndex(
  { githubProfile: 1 },
  { sparse: true }   // 没有 githubProfile 字段的文档不进入索引
)

// 好处：减少索引大小，避免大量 null 值占用空间
// 稀疏索引 + 唯一索引组合，可以允许多个文档没有该字段（不违反唯一约束）
db.users.createIndex(
  { phoneNumber: 1 },
  { unique: true, sparse: true }
  // 多个用户没有 phoneNumber 不会报错
  // 但有 phoneNumber 的用户之间必须唯一
)
```

### 部分索引（Partial Index）

```javascript
// 部分索引：只索引满足条件的文档子集
// 比稀疏索引更灵活，可以指定任意过滤条件

// 只索引活跃用户
db.users.createIndex(
  { username: 1 },
  { partialFilterExpression: { status: "active" } }
)

// 只索引高价值订单（金额 > 1000）
db.orders.createIndex(
  { userId: 1, createdAt: -1 },
  { partialFilterExpression: { amount: { $gt: 1000 } } }
)

// 好处：索引更小，查询更快（只针对子集）
// 注意：查询时必须包含过滤条件，否则无法使用部分索引
// db.users.find({ username: "zhangsan", status: "active" }) 能使用索引
// db.users.find({ username: "zhangsan" }) 不使用部分索引（全扫描）
```

### 唯一索引

```javascript
// 唯一索引确保字段值不重复
db.users.createIndex({ email: 1 }, { unique: true })

// 复合唯一索引（组合唯一）
db.userProfiles.createIndex(
  { userId: 1, profileType: 1 },
  { unique: true }
  // 同一userId下profileType不能重复
)
```

---

## 4.3 索引覆盖查询（Covered Query）

当查询结果所需的所有字段都在索引中，MongoDB 无需访问原始文档，直接从索引返回，
称为"覆盖查询"，性能极高。

```javascript
// 创建复合索引
db.users.createIndex({ status: 1, username: 1, email: 1 })

// 覆盖查询：查询条件和返回字段都在索引中
db.users.find(
  { status: "active" },           // 查询条件用了索引字段
  { username: 1, email: 1, _id: 0 }  // 返回字段都在索引中，且排除_id
)
// MongoDB 只读索引，不读原文档 <- 这就是覆盖查询

// 验证是否覆盖查询：查看 explain 的 totalDocsExamined
db.users.find(
  { status: "active" },
  { username: 1, email: 1, _id: 0 }
).explain("executionStats")
// 覆盖查询：executionStats.totalDocsExamined = 0
```

---

## 4.4 explain() 详解

```javascript
// explain() 三种模式
// "queryPlanner" - 显示查询计划（默认，不执行）
// "executionStats" - 执行并显示统计信息（常用）
// "allPlansExecution" - 执行所有候选计划，显示完整信息

const result = db.orders.find({
  status: "completed",
  amount: { $gt: 100 }
}).explain("executionStats")

/*
重点关注字段解读：

queryPlanner.winningPlan.stage:
  COLLSCAN - 全集合扫描（没有使用索引，性能差，需要优化）
  IXSCAN   - 索引扫描（使用了索引，性能好）
  FETCH    - 通过索引找到文档位置后，访问原文档
  SORT     - 在内存中排序（没有利用索引排序）
  LIMIT    - 限制结果数量
  SKIP     - 跳过

executionStats:
  nReturned         - 返回的文档数量
  totalKeysExamined - 扫描的索引条目数
  totalDocsExamined - 扫描的原始文档数
  executionTimeMillis - 执行时间（毫秒）

分析指标：
  理想情况：totalDocsExamined == nReturned（精确命中）
  覆盖查询：totalDocsExamined == 0（无需访问原文档）
  全扫描：totalDocsExamined >> nReturned（索引失效，需要添加索引）

示例 explain 输出（简化）：
{
  "queryPlanner": {
    "winningPlan": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",
        "indexName": "status_1_amount_1",
        "direction": "forward"
      }
    }
  },
  "executionStats": {
    "nReturned": 100,
    "executionTimeMillis": 2,
    "totalKeysExamined": 100,
    "totalDocsExamined": 100
  }
}
*/
```

---

## 4.5 索引设计原则

```
索引设计黄金原则：

1. 为高频查询字段建立索引
   -> 分析慢查询日志，找出频繁执行的查询

2. 遵循 ESR 规则设计复合索引
   -> E（等值）-> S（排序）-> R（范围）

3. 覆盖查询（把常用返回字段也加入索引）
   -> 避免回表查询，减少IO

4. 避免过多索引
   -> 每个索引都增加写操作的开销（插入/更新需维护索引）
   -> 通常一个集合不超过 5-10 个索引

5. 监控索引使用情况
   -> db.collection.aggregate([{ $indexStats: {} }])
   -> 删除长期未使用的索引（accesses.ops 长期为0）

6. 大集合谨慎建索引
   -> 建索引会锁写（MongoDB 4.2+ 支持在线索引构建）
   -> 评估建索引期间的业务影响

7. 对区分度高的字段建索引
   -> 如果字段只有 2-3 个值（如性别），索引效果有限
   -> 区分度 = 不同值数量 / 总文档数，越接近1越好

8. 利用部分索引节省空间
   -> 只对频繁查询的子集建索引
```


---

# Part 5: 数据模型设计

## 5.1 嵌入式文档 vs 引用

这是 MongoDB 数据建模的核心决策。

```
嵌入式（Embedded）：
  将相关数据放在同一文档中

  {
    _id: ...,
    title: "订单",
    user: { name: "张三", email: "..." },  // 嵌入用户信息
    items: [{ name: "手机", price: 999 }]  // 嵌入商品列表
  }

引用（Reference）：
  只存储 ID，通过 $lookup 或应用层代码关联

  {
    _id: ...,
    title: "订单",
    userId: ObjectId("..."),              // 只存 ID
    itemIds: [ObjectId("..."), ...]       // 商品 ID 列表
  }
```

### 选择标准

| 维度 | 选嵌入 | 选引用 |
|------|-------|-------|
| **关系** | 一对一 / 一对少 | 一对多（多端无界增长）/ 多对多 |
| **访问模式** | 总是一起访问 | 常常单独访问 |
| **数据大小** | 子数据小且有限 | 子数据大或无限增长 |
| **更新频率** | 子数据很少单独更新 | 子数据频繁独立更新 |
| **共享需求** | 不被多个文档共享 | 被多个父文档共享 |
| **文档大小** | 总大小 < 16MB | 嵌入会超过 16MB |

---

## 5.2 一对一建模

```javascript
// 场景：用户 与 用户详细信息（扩展信息）

// 方案1：嵌入（推荐，总是一起访问）
{
  _id: ObjectId("..."),
  username: "zhangsan",
  email: "zhangsan@example.com",
  profile: {               // 嵌入详细信息
    realName: "张三",
    idNumber: "310...",
    birthDate: ISODate("1996-01-01"),
    bio: "热爱编程的工程师"
  }
}

// 方案2：引用（适合信息很大或访问频率差异大）
// users 集合
{ _id: ObjectId("U001"), username: "zhangsan", email: "..." }

// userProfiles 集合
{ _id: ObjectId("P001"), userId: ObjectId("U001"), realName: "张三", ... }
```

---

## 5.3 一对多建模

```javascript
// 场景：博客文章 与 评论

// 方案1：嵌入评论（评论数量有限，如最多100条）
{
  _id: ObjectId("A001"),
  title: "MongoDB最佳实践",
  content: "...",
  comments: [
    { author: "张三", text: "写得很好！", createdAt: ISODate("...") },
    { author: "李四", text: "有所收获", createdAt: ISODate("...") }
  ]
}
// 优点：一次查询获取文章和所有评论
// 缺点：评论过多时文档变大，更新某条评论需要知道位置

// 方案2：引用（评论无限增长）
// articles 集合
{ _id: ObjectId("A001"), title: "MongoDB最佳实践", content: "..." }

// comments 集合（推荐：在"多"端存父ID）
{ _id: ObjectId("..."), articleId: ObjectId("A001"), author: "张三", text: "..." }
{ _id: ObjectId("..."), articleId: ObjectId("A001"), author: "李四", text: "..." }

// 创建索引优化查询
db.comments.createIndex({ articleId: 1, createdAt: -1 })

// 混合方案：嵌入热门评论 + 引用全部评论
{
  _id: ObjectId("A001"),
  title: "...",
  commentCount: 1234,
  topComments: [                        // 只嵌入 Top 3 评论
    { author: "张三", text: "...", likes: 200 }
  ]
  // 更多评论通过 comments 集合查询
}
```

---

## 5.4 多对多建模

```javascript
// 场景：学生 与 课程（学生可以选多门课，课程有多个学生）

// 方案1：关联集合（类似SQL中间表）
// students 集合
{ _id: ObjectId("S001"), name: "张三", grade: "大三" }

// courses 集合
{ _id: ObjectId("C001"), name: "MongoDB高级编程", teacher: "李老师" }

// enrollments 集合（关联）
{
  _id: ObjectId("..."),
  studentId: ObjectId("S001"),
  courseId: ObjectId("C001"),
  enrolledAt: ISODate("2024-09-01"),
  score: 95
}

// 索引
db.enrollments.createIndex({ studentId: 1 })
db.enrollments.createIndex({ courseId: 1 })

// 方案2：两端都存ID数组（数量有限时）
// students 集合
{ _id: ObjectId("S001"), name: "张三", courseIds: [ObjectId("C001"), ObjectId("C002")] }

// courses 集合
{ _id: ObjectId("C001"), name: "MongoDB", studentIds: [ObjectId("S001"), ObjectId("S003")] }
// 缺点：需要维护两端数组的一致性
```

---

## 5.5 大型文档设计模式

### 桶模式（Bucket Pattern）

适合时间序列数据，将多个细粒度文档合并为一个桶：

```javascript
// 反模式：每次测量一条文档（海量小文档）
{ sensorId: "S001", temperature: 25.3, humidity: 60, timestamp: ISODate("2024-01-01T00:00:00Z") }
{ sensorId: "S001", temperature: 25.4, humidity: 61, timestamp: ISODate("2024-01-01T00:01:00Z") }
// 每分钟一条，一年5.25亿条文档

// 桶模式：按小时或天聚合
{
  sensorId: "S001",
  date: ISODate("2024-01-01T00:00:00Z"),  // 桶的时间标识（小时）
  measurements: [
    { ts: ISODate("2024-01-01T00:00:00Z"), temp: 25.3, humi: 60 },
    { ts: ISODate("2024-01-01T00:01:00Z"), temp: 25.4, humi: 61 },
    // ... 60条（一个小时的分钟级数据）
  ],
  count: 60,           // 测量次数（便于统计）
  sumTemp: 1520.0,     // 温度总和（便于计算均值）
  minTemp: 24.8,       // 预计算最小值
  maxTemp: 26.1        // 预计算最大值
}
// 文档数从 5.25亿 降到 876万（每天24桶 * 365天）
```

### 子集模式（Subset Pattern）

将大文档拆分，只在主文档中保留常用数据：

```javascript
// 场景：电商商品，评论数可能数万条

// 子集模式：主文档只存热门评论，完整评论存独立集合
// products 集合（主文档）
{
  _id: ObjectId("P001"),
  name: "iPhone 15",
  price: 7999,
  avgRating: 4.8,
  reviewCount: 12345,
  topReviews: [          // 只存Top 10评论（子集）
    { author: "...", rating: 5, text: "...", helpful: 500 }
  ]
}

// productReviews 集合（完整评论）
{
  productId: ObjectId("P001"),
  author: "...",
  rating: 5,
  text: "...",
  helpful: 500,
  createdAt: ISODate("...")
}
```

### 计算模式（Computed Pattern）

预计算并存储聚合结果，避免重复运算：

```javascript
// 计算模式：写入时更新预计算字段
// 评论写入时同步更新商品的统计字段
db.products.updateOne(
  { _id: productId },
  {
    $inc: { reviewCount: 1, totalRating: newRating },
    $set: {
      avgRating: (currentTotalRating + newRating) / (currentCount + 1),
      lastReviewedAt: new Date()
    }
  }
)

// 商品文档包含预计算值
{
  _id: ObjectId("P001"),
  name: "iPhone 15",
  reviewCount: 12345,   // 预计算
  totalRating: 59256,   // 预计算
  avgRating: 4.8,       // 预计算
  ratingDistribution: { "5": 8000, "4": 3000, "3": 1000, "2": 200, "1": 145 }
}
```

### 异常值模式（Outlier Pattern）

处理极少数超大文档（如百万粉丝的大V）：

```javascript
// 普通用户：关注者存在文档内
{
  _id: ObjectId("U001"),
  username: "普通用户",
  followerCount: 50,
  followers: [ObjectId("..."), ObjectId("..."), /* 50个ID */],
  hasOverflow: false
}

// 大V：超过阈值的关注者存入额外集合
{
  _id: ObjectId("U002"),
  username: "大V",
  followerCount: 1500000,
  followers: [ObjectId("..."), /* 前1000个 */],
  hasOverflow: true  // 标记有溢出
}

// followerOverflow 集合（存大V的额外关注者）
{
  userId: ObjectId("U002"),
  followers: [ObjectId("..."), /* 后面的999000个 */]
}
```

---

# Part 6: 事务

## 6.1 MongoDB 事务概述

```
MongoDB 事务发展历程：
  MongoDB 1.x/2.x/3.x: 只支持单文档原子操作
  MongoDB 4.0:          支持副本集上的多文档事务
  MongoDB 4.2:          支持分片集群上的分布式事务
  MongoDB 5.0+:         事务性能和功能持续优化
```

### 单文档原子操作 vs 多文档事务

```javascript
// 单文档原子操作（无需事务）
// MongoDB 对单个文档的读-改-写天然是原子的
db.accounts.updateOne(
  { _id: "ACC001", balance: { $gte: 100 } },
  { $inc: { balance: -100 } }
)
// 这个操作要么成功要么失败，不存在部分执行

// 嵌套文档/数组更新也是原子的
db.shoppingCart.updateOne(
  { userId: "U001" },
  {
    $push: { items: { productId: "P001", qty: 1 } },
    $inc: { totalItems: 1, totalAmount: 99 }
  }
)
// 整体原子，不会出现只更新了 items 没更新 totalItems 的情况

// 需要多文档事务的场景：
// 银行转账：A账户扣款 + B账户入账，必须同时成功
// 库存扣减 + 订单创建，必须同时成功
// 涉及多个集合/多个文档的原子操作
```

---

## 6.2 多文档事务实战

```javascript
// ==================== MongoDB Shell 事务示例 ====================

// 场景：银行转账

// 1. 启动 Session
const session = db.getMongo().startSession()

// 2. 设置事务选项
const transactionOptions = {
  readPreference: 'primary',
  readConcern: { level: 'local' },
  writeConcern: { w: 'majority' }
}

try {
  // 3. 开始事务
  session.startTransaction(transactionOptions)

  const accounts = session.getDatabase("bank").accounts

  // 4. 在事务中执行操作
  const fromAccount = accounts.findOne({ _id: "ACC001" }, { session })

  if (!fromAccount || fromAccount.balance < 500) {
    throw new Error("余额不足")
  }

  // 扣款
  accounts.updateOne(
    { _id: "ACC001" },
    { $inc: { balance: -500 } },
    { session }   // 关键：传入 session
  )

  // 入账
  accounts.updateOne(
    { _id: "ACC002" },
    { $inc: { balance: 500 } },
    { session }
  )

  // 记录转账流水
  session.getDatabase("bank").transactions.insertOne({
    fromId: "ACC001",
    toId: "ACC002",
    amount: 500,
    createdAt: new Date()
  }, { session })

  // 5. 提交事务
  session.commitTransaction()
  console.log("转账成功")

} catch (error) {
  // 6. 出现错误，回滚事务
  session.abortTransaction()
  console.error("转账失败，已回滚：", error.message)
} finally {
  // 7. 关闭 Session
  session.endSession()
}
```

---

## 6.3 事务使用建议

```
事务性能注意事项：
  1. 事务超时默认 60秒，可通过 transactionLifetimeLimitSeconds 调整
  2. 事务会持有文档级锁，长事务影响并发
  3. 事务中的操作在提交前对其他事务不可见
  4. WiredTiger 快照隔离：事务内读取的是事务开始时的快照

最佳实践：
  1. 事务要尽量短（毫秒级，不要秒级）
  2. 只在真正需要跨文档/集合原子性时使用
  3. 优先考虑单文档原子操作或反范式化设计
  4. 使用重试逻辑处理瞬时错误（如写冲突 112 错误）
  5. 不要在事务中做耗时操作（HTTP请求、复杂计算等）

不适合用事务的场景：
  1. 批量写入（用 bulkWrite 或 ordered:false）
  2. 报表统计（用聚合管道）
  3. 大量文档更新（分批次更新，每批用一个事务）
```

---

# Part 7: 副本集（Replica Set）

## 7.1 副本集架构

副本集是 MongoDB 高可用的基础，包含多个 mongod 节点，数据自动同步。

```
+===========================================================+
|                    副本集架构                              |
+===========================================================+
|                                                           |
|    客户端应用                                             |
|        |                                                  |
|        | 写操作（默认发往Primary）                        |
|        | 读操作（可配置读偏好）                           |
|        v                                                  |
|  +------------+                                           |
|  |  Primary   |  <- 主节点（唯一可接受写操作）            |
|  |  mongod    |                                           |
|  | Port:27017 |                                           |
|  +------------+                                           |
|    |        |                                             |
|    | oplog  | oplog  <- 操作日志复制（异步）              |
|    v        v                                             |
|  +----------+  +----------+                              |
|  | Secondary|  | Secondary|  <- 从节点（只读，自动同步）  |
|  | mongod   |  | mongod   |                              |
|  | Port:27018|  | Port:27019|                            |
|  +----------+  +----------+                              |
|       |              |                                   |
|       +------+-------+                                   |
|              |                                           |
|        +----------+                                      |
|        | Arbiter  |  <- 仲裁节点（不存数据，只参与选举） |
|        | 可选     |                                       |
|        +----------+                                      |
|                                                           |
+===========================================================+
```

### 副本集成员类型

| 类型 | 存储数据 | 可选举为Primary | 投票权 | 说明 |
|------|---------|----------------|--------|------|
| Primary | 是 | 是（当前主） | 是 | 接受所有读写 |
| Secondary | 是 | 是 | 是 | 同步数据，可提供读 |
| Arbiter | 否 | 否 | 是 | 只参与选举，节省存储 |
| Hidden | 是 | 否 | 是（可配置）| 不接受应用读，用于备份 |
| Delayed | 是 | 否 | 是 | 延迟复制，防误操作 |
| Non-voting | 是 | 是 | 否 | 超过50个成员时用 |

---

## 7.2 选举机制

```
选举触发条件：
  1. 副本集初始化
  2. Primary 宕机或无响应（超过 electionTimeoutMillis，默认10秒）
  3. 手动触发（rs.stepDown()）

选举过程（Raft 协议变体）：

  Step 1: Secondary 检测到 Primary 不可达
          -> 等待选举超时（随机化，防止split vote）

  Step 2: 发起选举的节点成为候选者（Candidate）
          -> 增加 term（任期）编号
          -> 向其他节点发送投票请求

  Step 3: 其他节点评估是否投票（条件）：
          1. 没有在当前 term 投过票
          2. 候选者的 oplog 不比自己旧

  Step 4: 获得多数票（> N/2）的候选者成为新 Primary

  Step 5: 新 Primary 通知所有节点

  选举时间：通常 < 12秒（electionTimeoutMillis 10秒 + 网络延迟）

Priority 配置：
  Priority 越高，越优先被选为 Primary
  Priority = 0 的节点永远不会成为 Primary
```

---

## 7.3 oplog（操作日志）

```
oplog 特性：
  - 存储在 local.oplog.rs 集合（固定大小集合 Capped Collection）
  - 默认大小：5% 磁盘空间（64位系统，最小1GB，最大50GB）
  - 幂等性：每个操作可以重复执行而不改变结果
  - 全量同步：Secondary 第一次加入，全量复制（initial sync）
  - 增量同步：之后通过 oplog 增量复制

oplog 条目示例：
{
  "ts": Timestamp(1704067200, 1),  // 操作时间戳
  "t": NumberLong(1),              // 选举 term
  "v": 2,                          // oplog 版本
  "op": "i",                       // 操作类型：i=insert, u=update, d=delete, c=command
  "ns": "mydb.orders",             // 命名空间（数据库.集合）
  "o": { _id: ObjectId("..."), username: "zhangsan" }  // 操作对象
}

操作类型：
  "i" - Insert
  "u" - Update
  "d" - Delete
  "c" - Command（如 createCollection, dropCollection）
  "n" - No-op（心跳）

监控复制延迟：
  rs.printReplicationInfo()            // Primary 的 oplog 信息
  rs.printSecondaryReplicationInfo()   // Secondary 的复制延迟
```

---

## 7.4 写关注（Write Concern）

Write Concern 控制写操作的持久性保证级别。

```javascript
// 写关注选项：
// w: 1  - 只需Primary确认（默认，快）
// w: 0  - 不需要任何确认（最快，可能丢数据）
// w: "majority" - 多数节点确认（推荐，持久）
// w: N  - N个节点确认

// j: false - 不等待 Journal 落盘（默认）
// j: true  - 等待 Journal 落盘（持久但慢）

// wtimeout: 毫秒数 - 等待超时时间

db.orders.insertOne(
  { orderId: "ORD-001", amount: 999 },
  { writeConcern: { w: "majority", j: true, wtimeout: 5000 } }
)
```

| 写关注 | 速度 | 持久性 | 适用场景 |
|--------|------|--------|---------|
| `{w:0}` | 最快 | 最低（可能丢失）| 统计计数、缓存数据 |
| `{w:1}` | 快 | Primary宕机可能丢失 | 一般业务数据 |
| `{w:"majority"}` | 中 | 高（多数节点确认）| 重要业务数据 |
| `{w:"majority",j:true}` | 慢 | 最高 | 金融/支付数据 |

---

## 7.5 读偏好（Read Preference）

```javascript
// 读偏好模式：
// primary（默认）       - 只从 Primary 读（强一致性）
// primaryPreferred      - 优先 Primary，Primary 不可用时读 Secondary
// secondary             - 只从 Secondary 读（可能读到旧数据）
// secondaryPreferred    - 优先 Secondary，没有可用 Secondary 时读 Primary
// nearest               - 网络延迟最低的节点（地理就近读）

// 读偏好 + 标签（Tag Sets）- 将读操作路由到特定数据中心
rs.reconfig({
  members: [
    { _id: 0, host: "mongo-1:27017", tags: { dc: "hz", type: "primary" } },
    { _id: 1, host: "mongo-2:27017", tags: { dc: "hz", type: "secondary" } },
    { _id: 2, host: "mongo-3:27017", tags: { dc: "sh", type: "secondary" } }
  ]
})

// 读取上海数据中心的节点
db.collection.find().readPref("secondary", [{ dc: "sh" }])
```

| 读偏好 | 一致性 | 性能 | 适用场景 |
|--------|--------|------|---------|
| primary | 强一致 | 低（所有请求走Primary） | 金融交易、实时数据 |
| primaryPreferred | 强（Primary可用时）| 中 | 一般业务 |
| secondary | 可能读到旧数据 | 高（分散读压力）| 报表、数据分析 |
| secondaryPreferred | 一般 | 高 | 读多写少 |
| nearest | 取决于拓扑 | 最低延迟 | 地理分布式应用 |

---

## 7.6 副本集配置与运维

```javascript
// ==================== 初始化副本集 ====================
rs.initiate({
  _id: "myReplicaSet",
  members: [
    { _id: 0, host: "mongo-1:27017", priority: 2 },  // 高优先级，优先选为Primary
    { _id: 1, host: "mongo-2:27017", priority: 1 },
    { _id: 2, host: "mongo-3:27017", priority: 1 }
  ]
})

// ==================== 查看副本集状态 ====================
rs.status()      // 各节点状态、健康度、延迟
rs.conf()        // 副本集配置
db.hello()       // 当前节点信息（新版接口）

// ==================== 添加/删除成员 ====================
rs.add("mongo-4:27017")                         // 添加 Secondary
rs.add({ host: "mongo-5:27017", priority: 0 })  // 添加不可选为Primary的节点
rs.addArb("mongo-arb:27017")                    // 添加仲裁节点
rs.remove("mongo-4:27017")                      // 移除成员

// ==================== 手动切换 Primary ====================
rs.stepDown(60)  // 当前Primary让位，60秒内不参与选举

// ==================== 查看复制延迟 ====================
rs.printSecondaryReplicationInfo()
// Member: mongo-2:27017
// behind the primary by 0 secs
// Member: mongo-3:27017
// behind the primary by 2 secs
```

---

# Part 8: 分片集群（Sharded Cluster）

## 8.1 分片集群架构

```
+===================================================================+
|                      MongoDB 分片集群架构                          |
+===================================================================+
|                                                                   |
|   客户端应用                                                      |
|       |                                                           |
|       v                                                           |
|  +----------+  +----------+                                      |
|  | mongos   |  | mongos   |  <- 路由层（可多个，负载均衡）        |
|  | (Router) |  | (Router) |    解析请求，路由到正确Shard          |
|  +----------+  +----------+                                      |
|       |               |                                          |
|       +-------+-------+                                          |
|               |                                                   |
|               v                                                   |
|  +----------------------------------------------------+          |
|  |             Config Server（配置服务器）              |          |
|  |  +----------+  +----------+  +----------+          |          |
|  |  | cfgsvr-1 |  | cfgsvr-2 |  | cfgsvr-3 | <- 副本集|          |
|  |  +----------+  +----------+  +----------+          |          |
|  |  存储：分片元数据、Chunk映射、均衡器状态              |          |
|  +----------------------------------------------------+          |
|               |                                                   |
|       +-------+-------+----------+                               |
|       |               |          |                               |
|       v               v          v                               |
|  +-----------+  +-----------+  +-----------+                     |
|  |  Shard 1  |  |  Shard 2  |  |  Shard 3  |                    |
|  | (副本集)   |  | (副本集)   |  | (副本集)   |                    |
|  | P + S + S |  | P + S + S |  | P + S + S |                    |
|  +-----------+  +-----------+  +-----------+                     |
|                                                                   |
+===================================================================+
```

### 组件说明

| 组件 | 职责 | 推荐数量 |
|------|------|---------|
| mongos | 路由层，接收客户端请求，路由到正确Shard | 2-3个（负载均衡） |
| Config Server | 存储集群元数据（分片键、Chunk映射） | 3个（副本集） |
| Shard | 存储实际数据，每个Shard是一个副本集 | 3个起步，按需扩展 |

---

## 8.2 分片键选择

分片键（Shard Key）是决定数据如何分布到各 Shard 的字段，选择不当会导致数据倾斜。

### 分片键选择原则

```
1. 高基数（Cardinality）
   -> 字段的不同值越多越好
   -> 如 userId（百万用户）比 status（只有几种值）好

2. 均匀分布（Distribution）
   -> 数据能均匀分布到各个Shard
   -> 避免热点（如时间戳作为范围分片键，新数据全写最后一个Shard）

3. 针对性查询（Targeted Queries）
   -> 大部分查询包含分片键，避免广播查询（全Shard扫描）
   -> 如按 userId 查询，分片键应包含 userId

4. 不可变性（Immutability）
   -> 分片键值不应该被更新（更新分片键需要删除重插）
   -> MongoDB 4.4+ 支持更新分片键，但代价高

好的分片键示例：
  userId + createdAt    (用户相关数据)
  orderId（高基数）     (订单数据)

坏的分片键示例：
  status               (区分度太低，3-5种值)
  createdAt（递增）    (范围分片时写入热点)
```

---

## 8.3 分片策略

### 范围分片（Range Sharding）

```javascript
// 按分片键的值范围分片
// Shard 1: userId 0 - 10000
// Shard 2: userId 10001 - 20000
// Shard 3: userId 20001+

// 优点：范围查询高效（连续范围在同一Shard）
// 缺点：递增字段（如ObjectId）会导致写入热点

sh.shardCollection("mydb.orders", { orderId: 1 })  // 范围分片
```

### 哈希分片（Hash Sharding）

```javascript
// 对分片键做哈希，均匀分布到各Shard
// 优点：数据分布均匀，避免热点
// 缺点：范围查询需要广播到所有Shard

sh.shardCollection("mydb.users", { userId: "hashed" })  // 哈希分片
```

### 区域分片（Zone Sharding）

```javascript
// 将特定范围的数据固定到特定Shard
// 适合：多地域部署、合规要求（数据本地化）

// 给Shard打标签
sh.addShardTag("shard0000", "CN_EAST")   // 华东Shard
sh.addShardTag("shard0001", "CN_NORTH")  // 华北Shard

// 将特定分片键范围映射到Zone
sh.addTagRange(
  "mydb.users",
  { region: "shanghai" },     // 范围起始
  { region: "shenzhen" },     // 范围结束（不包含）
  "CN_EAST"                   // Zone标签
)
// 上海用户的数据存到 CN_EAST（华东）Shard
```

---

## 8.4 Chunk 概念与自动均衡

```
Chunk 是分片数据的基本单元：
  - 每个 Chunk 包含一个连续范围的分片键值
  - 默认大小：64MB（可配置，范围 1MB-1024MB）
  - 当 Chunk 超过阈值时，自动分裂（Split）
  - Balancer（均衡器）自动迁移 Chunk，保持各Shard数据均匀

Chunk 分裂过程：
  Chunk A: [min, max] 64MB
           |
           v（达到阈值，自动分裂）
  Chunk A1: [min, mid]  32MB
  Chunk A2: [mid, max]  32MB

Balancer（均衡器）：
  - Config Server 上运行
  - 当Shard间Chunk数量差超过阈值时触发迁移
  - 默认在业务低峰期执行（可配置迁移窗口）
  - 迁移时数据不丢失，对业务透明

查看分片状态：
  sh.status()              // 分片集群整体状态
  sh.getBalancerState()    // 均衡器是否启用
  sh.isBalancerRunning()   // 均衡器是否在运行
```

---

## 8.5 广播操作 vs 目标操作

```
目标操作（Targeted Operation）- 高效：
  查询条件包含分片键，mongos 只发往对应Shard

  db.orders.find({ userId: "U001" })
  -> mongos 计算 hash(U001) -> 确定 Shard 1
  -> 只查询 Shard 1（高效）

广播操作（Scatter-Gather）- 低效：
  查询条件不包含分片键，mongos 广播到所有Shard

  db.orders.find({ status: "completed" })
  -> mongos 无法确定在哪个Shard
  -> 广播到所有3个Shard，汇聚结果（低效）

优化建议：
  1. 业务查询尽量包含分片键
  2. 分片键选择应覆盖最常见查询
  3. 用 explain() 检查是否是广播查询
     （SHARDING_FILTER + SHARD_MERGE = 广播）
```


---

# Part 9: Spring Boot 集成实战

## 9.1 依赖配置

```xml
<!-- pom.xml -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-mongodb</artifactId>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

---

## 9.2 application.yml 配置

```yaml
spring:
  data:
    mongodb:
      # 单机连接
      uri: mongodb://localhost:27017/mydb

      # 带认证
      # uri: mongodb://username:password@localhost:27017/mydb?authSource=admin

      # 副本集连接
      # uri: mongodb://mongo-1:27017,mongo-2:27017,mongo-3:27017/mydb?replicaSet=myReplicaSet

      # 分片集群连接（连 mongos）
      # uri: mongodb://mongos-1:27017,mongos-2:27017/mydb

      # 高级连接参数（连接池、超时等）
      # uri: mongodb://localhost:27017/mydb?connectTimeoutMS=5000&maxPoolSize=20&minPoolSize=5
```

---

## 9.3 实体类注解

```java
import lombok.Data;
import lombok.Builder;
import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.mapping.Field;
import org.springframework.data.mongodb.core.index.Indexed;
import org.springframework.data.mongodb.core.index.CompoundIndex;
import org.springframework.data.mongodb.core.index.CompoundIndexes;
import org.bson.types.ObjectId;
import java.time.LocalDateTime;
import java.util.List;

/**
 * 用户实体类
 * @Document   - 对应 MongoDB 集合
 * @Id         - 主键（对应 _id 字段）
 * @Field      - 指定 MongoDB 中的字段名（默认使用属性名）
 * @Indexed    - 创建索引
 * @CompoundIndex - 复合索引
 */
@Data
@Builder
@Document(collection = "users")
@CompoundIndexes({
    @CompoundIndex(
        name = "idx_status_created",
        def = "{'status': 1, 'createdAt': -1}"
    ),
    @CompoundIndex(
        name = "idx_address_city",
        def = "{'address.city': 1}"
    )
})
public class User {

    @Id
    private ObjectId id;

    @Indexed(unique = true)
    private String username;

    @Indexed(unique = true)
    private String email;

    private String password;
    private Integer age;
    private String role;
    private String status;
    private List<String> tags;
    private Address address;
    private Integer loginCount;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @Data
    @Builder
    public static class Address {
        private String city;
        private String district;
        private String zipCode;
    }
}

// ==================== 订单实体 ====================
@Data
@Builder
@Document(collection = "orders")
public class Order {

    @Id
    private ObjectId id;

    @Indexed(unique = true)
    private String orderId;

    @Indexed
    private ObjectId userId;

    private List<OrderItem> items;
    private Double amount;
    private String status;
    private String paymentMethod;

    @Indexed
    @CreatedDate
    private LocalDateTime createdAt;

    @Data
    @Builder
    public static class OrderItem {
        private String productId;
        private String productName;
        private String category;
        private Integer qty;
        private Double price;
    }
}
```

---

## 9.4 MongoRepository CRUD

```java
import org.springframework.data.mongodb.repository.MongoRepository;
import org.springframework.data.mongodb.repository.Query;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.bson.types.ObjectId;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

public interface UserRepository extends MongoRepository<User, ObjectId> {

    // ==================== 方法名衍生查询 ====================
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    List<User> findByStatus(String status);
    List<User> findByAgeGreaterThan(int age);
    List<User> findByAgeBetween(int min, int max);
    List<User> findByUsernameContaining(String keyword);
    List<User> findByStatusAndRole(String status, String role);
    List<User> findByAddressCity(String city);
    List<User> findByTagsContaining(String tag);
    List<User> findByStatusOrderByCreatedAtDesc(String status);
    Page<User> findByStatus(String status, Pageable pageable);
    long countByStatus(String status);
    boolean existsByEmail(String email);
    void deleteByStatus(String status);

    // ==================== @Query 注解（自定义查询）====================

    @Query("{ 'address.city': ?0, 'status': ?1 }")
    List<User> findByCityAndStatus(String city, String status);

    @Query("{ 'email': { $regex: ?0, $options: 'i' } }")
    List<User> findByEmailRegex(String pattern);

    @Query("{ 'age': { $gte: ?0, $lte: ?1 } }")
    List<User> findByAgeRange(int minAge, int maxAge);

    // 投影（只返回部分字段）
    @Query(value = "{ 'status': ?0 }", fields = "{ 'username': 1, 'email': 1 }")
    List<User> findUsernameAndEmailByStatus(String status);

    @Query(value = "{ 'status': 'active' }", sort = "{ 'loginCount': -1 }")
    List<User> findActiveUsersSortedByLoginCount();

    @Query("{ 'createdAt': { $gte: ?0, $lt: ?1 } }")
    List<User> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);
}
```

---

## 9.5 MongoTemplate 复杂查询

```java
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.data.mongodb.core.query.Update;
import org.springframework.data.mongodb.core.aggregation.*;
import org.springframework.data.domain.Sort;
import org.springframework.stereotype.Service;
import lombok.RequiredArgsConstructor;
import java.util.List;

@Service
@RequiredArgsConstructor
public class UserService {

    private final MongoTemplate mongoTemplate;

    // ==================== Query + Criteria ====================

    public List<User> findActiveUsersInCity(String city, int minAge, int maxAge) {
        Query query = new Query();
        Criteria criteria = new Criteria().andOperator(
            Criteria.where("status").is("active"),
            Criteria.where("address.city").is(city),
            Criteria.where("age").gte(minAge).lte(maxAge)
        );
        query.addCriteria(criteria);
        query.fields().include("username").include("email").include("age").exclude("_id");
        query.with(Sort.by(Sort.Direction.DESC, "loginCount"));
        query.skip(0).limit(20);
        return mongoTemplate.find(query, User.class);
    }

    public List<User> findByKeyword(String keyword) {
        Query query = new Query();
        Criteria criteria = new Criteria().orOperator(
            Criteria.where("username").regex(keyword, "i"),
            Criteria.where("email").regex(keyword, "i")
        );
        query.addCriteria(criteria);
        return mongoTemplate.find(query, User.class);
    }

    // ==================== Update ====================

    public long updateUserStatus(String fromStatus, String toStatus) {
        Query query = new Query(Criteria.where("status").is(fromStatus));
        Update update = new Update()
            .set("status", toStatus)
            .set("updatedAt", java.time.LocalDateTime.now())
            .inc("version", 1);
        return mongoTemplate.updateMulti(query, update, User.class).getModifiedCount();
    }

    public void addTagToUser(ObjectId userId, String tag) {
        Query query = new Query(Criteria.where("_id").is(userId));
        Update update = new Update().addToSet("tags", tag);
        mongoTemplate.updateFirst(query, update, User.class);
    }

    public User findAndIncrementLoginCount(String username) {
        Query query = new Query(Criteria.where("username").is(username));
        Update update = new Update()
            .inc("loginCount", 1)
            .set("lastLoginAt", java.time.LocalDateTime.now());
        return mongoTemplate.findAndModify(query, update,
            org.springframework.data.mongodb.core.FindAndModifyOptions
                .options().returnNew(true),
            User.class);
    }

    // ==================== Aggregation Pipeline Java API ====================

    public List<CityStats> aggregateByCityStats() {
        MatchOperation match = Aggregation.match(
            Criteria.where("status").is("active")
        );
        GroupOperation group = Aggregation.group("address.city")
            .count().as("userCount")
            .avg("age").as("avgAge");
        ProjectionOperation project = Aggregation.project()
            .and("_id").as("city")
            .andInclude("userCount")
            .andInclude("avgAge")
            .andExclude("_id");
        SortOperation sort = Aggregation.sort(Sort.Direction.DESC, "userCount");
        LimitOperation limit = Aggregation.limit(10);

        Aggregation aggregation = Aggregation.newAggregation(
            match, group, project, sort, limit
        );
        return mongoTemplate.aggregate(aggregation, "users", CityStats.class)
            .getMappedResults();
    }

    // $lookup 关联查询
    public List<OrderWithUser> findOrdersWithUserInfo(String status) {
        MatchOperation match = Aggregation.match(Criteria.where("status").is(status));
        LookupOperation lookup = LookupOperation.newLookup()
            .from("users")
            .localField("userId")
            .foreignField("_id")
            .as("user");
        UnwindOperation unwind = Aggregation.unwind("user", true);
        ProjectionOperation project = Aggregation.project(
            "orderId", "amount", "status", "createdAt"
        )
        .and("user.username").as("username")
        .and("user.email").as("userEmail");

        Aggregation aggregation = Aggregation.newAggregation(
            match, lookup, unwind, project
        );
        return mongoTemplate.aggregate(aggregation, "orders", OrderWithUser.class)
            .getMappedResults();
    }

    @Data public static class CityStats { private String city; private int userCount; private double avgAge; }
    @Data public static class OrderWithUser { private String orderId; private double amount; private String status; private String username; private String userEmail; }
}
```

---

## 9.6 分页查询实现

```java
@RestController
@RequestMapping("/api/users")
@RequiredArgsConstructor
public class UserController {

    private final UserRepository userRepository;
    private final MongoTemplate mongoTemplate;

    // MongoRepository 标准分页
    @GetMapping
    public Page<User> listUsers(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "active") String status
    ) {
        Pageable pageable = PageRequest.of(page, size,
            Sort.by(Sort.Direction.DESC, "createdAt"));
        return userRepository.findByStatus(status, pageable);
    }

    // MongoTemplate 手动分页
    @GetMapping("/manual")
    public PageResult<User> listUsersManual(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(required = false) String city
    ) {
        Query query = new Query(Criteria.where("status").is("active"));
        if (city != null) {
            query.addCriteria(Criteria.where("address.city").is(city));
        }
        long total = mongoTemplate.count(query, User.class);
        query.skip((long) page * size).limit(size)
             .with(Sort.by(Sort.Direction.DESC, "createdAt"));
        List<User> users = mongoTemplate.find(query, User.class);
        return new PageResult<>(users, total, page, size);
    }

    // 游标分页（深度分页优化）
    @GetMapping("/cursor")
    public List<User> cursorPage(
        @RequestParam(required = false) String lastId,
        @RequestParam(defaultValue = "10") int size
    ) {
        Query query = new Query(Criteria.where("status").is("active"));
        if (lastId != null) {
            query.addCriteria(Criteria.where("_id").gt(new ObjectId(lastId)));
        }
        query.limit(size).with(Sort.by(Sort.Direction.ASC, "_id"));
        return mongoTemplate.find(query, User.class);
    }

    @Data
    @AllArgsConstructor
    public static class PageResult<T> {
        private List<T> data;
        private long total;
        private int page;
        private int size;
        private long totalPages;
        public PageResult(List<T> data, long total, int page, int size) {
            this.data = data; this.total = total;
            this.page = page; this.size = size;
            this.totalPages = (total + size - 1) / size;
        }
    }
}
```

---

## 9.7 GridFS（大文件存储）

GridFS 是 MongoDB 存储超过 16MB 文件（如图片、视频、文档）的方案，将文件分块存储。

```java
@Service
@RequiredArgsConstructor
public class FileService {

    private final GridFsTemplate gridFsTemplate;
    private final GridFsOperations gridFsOperations;

    public String uploadFile(MultipartFile file) throws Exception {
        org.bson.Document metadata = new org.bson.Document();
        metadata.put("originalName", file.getOriginalFilename());
        metadata.put("contentType", file.getContentType());
        metadata.put("size", file.getSize());

        ObjectId fileId = gridFsTemplate.store(
            file.getInputStream(),
            file.getOriginalFilename(),
            file.getContentType(),
            metadata
        );
        return fileId.toString();
    }

    public void downloadFile(String fileId, OutputStream outputStream) throws Exception {
        GridFSFile gridFSFile = gridFsTemplate.findOne(
            new Query(Criteria.where("_id").is(new ObjectId(fileId)))
        );
        if (gridFSFile == null) throw new RuntimeException("文件不存在: " + fileId);

        try (InputStream is = gridFsOperations.getResource(gridFSFile).getInputStream()) {
            byte[] buffer = new byte[8192];
            int bytesRead;
            while ((bytesRead = is.read(buffer)) != -1) {
                outputStream.write(buffer, 0, bytesRead);
            }
        }
    }

    public void deleteFile(String fileId) {
        gridFsTemplate.delete(new Query(Criteria.where("_id").is(new ObjectId(fileId))));
    }
}
```

---

## 9.8 事务支持（Spring Boot）

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MongoTemplate mongoTemplate;
    private final OrderRepository orderRepository;

    /**
     * 创建订单（事务）
     * 注意：需要 MongoDB 副本集环境才能使用事务
     */
    @Transactional
    public Order createOrder(ObjectId userId, List<Order.OrderItem> items) {
        double totalAmount = 0.0;
        for (Order.OrderItem item : items) {
            // 原子扣减库存（check-and-set）
            Query query = new Query(
                Criteria.where("_id").is(item.getProductId())
                    .and("stock").gte(item.getQty())
            );
            Update update = new Update().inc("stock", -item.getQty());
            var result = mongoTemplate.updateFirst(query, update, "products");
            if (result.getModifiedCount() == 0) {
                throw new RuntimeException("库存不足：" + item.getProductId());
            }
            totalAmount += item.getPrice() * item.getQty();
        }

        Order order = Order.builder()
            .orderId("ORD-" + System.currentTimeMillis())
            .userId(userId)
            .items(items)
            .amount(totalAmount)
            .status("pending")
            .build();

        return orderRepository.save(order);
        // 若此处异常，前面的库存扣减自动回滚
    }
}
```


---

# Part 10: 完整实战案例

## 案例1：内容管理系统（CMS）

### 数据模型设计

```javascript
// 文章文档（嵌套作者快照，引用评论）
db.articles.insertOne({
  _id: ObjectId("..."),
  title: "深入理解 MongoDB 索引",
  slug: "mongodb-index-deep-dive",
  content: "...",
  summary: "本文深入介绍MongoDB索引类型和最佳实践...",
  author: {                                // 嵌入作者快照（反范式化）
    userId: ObjectId("..."),
    username: "张技术",
    avatar: "https://cdn.example.com/avatar.jpg"
  },
  tags: ["mongodb", "database", "nosql"],
  categories: ["技术", "数据库"],
  status: "published",
  viewCount: 12345,
  likeCount: 567,
  commentCount: 89,
  topComments: [                           // 子集模式：只存热门评论
    {
      commentId: ObjectId("..."),
      author: "读者A",
      content: "写得很好！",
      likes: 100,
      createdAt: ISODate("...")
    }
  ],
  seo: {
    metaTitle: "MongoDB索引详解",
    metaDescription: "...",
    keywords: ["mongodb", "索引"]
  },
  publishedAt: ISODate("2024-01-15T10:00:00Z"),
  createdAt: ISODate("2024-01-14T08:00:00Z"),
  updatedAt: ISODate("2024-01-15T10:00:00Z")
})

// 评论文档（支持子评论）
db.comments.insertOne({
  _id: ObjectId("..."),
  articleId: ObjectId("..."),
  parentId: null,                          // null=顶级评论，非null=子评论
  depth: 0,
  author: { userId: ObjectId("..."), username: "读者A" },
  content: "写得很好！",
  likes: 100,
  replyCount: 3,
  status: "visible",
  createdAt: ISODate("...")
})

// 索引策略
db.articles.createIndex({ slug: 1 }, { unique: true })
db.articles.createIndex({ status: 1, publishedAt: -1 })
db.articles.createIndex({ tags: 1, status: 1 })
db.articles.createIndex(
  { title: "text", content: "text", summary: "text" },
  { weights: { title: 10, summary: 5, content: 1 }, default_language: "none" }
)

db.comments.createIndex({ articleId: 1, createdAt: -1 })
db.comments.createIndex({ articleId: 1, parentId: 1 })
```

```java
// 文章搜索 + 浏览量统计
@Service
@RequiredArgsConstructor
public class ArticleService {

    private final MongoTemplate mongoTemplate;

    // 全文搜索
    public List<Article> searchArticles(String keyword, int page, int size) {
        TextCriteria textCriteria = TextCriteria.forDefaultLanguage()
            .matchingAny(keyword.split("\\s+"));
        Query query = TextQuery.queryText(textCriteria)
            .sortByScore()
            .addCriteria(Criteria.where("status").is("published"));
        query.skip((long) page * size).limit(size);
        return mongoTemplate.find(query, Article.class);
    }

    // 原子增加浏览量
    public long incrementViewCount(String slug) {
        Article result = mongoTemplate.findAndModify(
            new Query(Criteria.where("slug").is(slug)),
            new Update().inc("viewCount", 1),
            FindAndModifyOptions.options().returnNew(true),
            Article.class
        );
        return result != null ? result.getViewCount() : 0;
    }
}
```

---

## 案例2：电商商品SKU（灵活属性存储）

```javascript
// 商品主文档（灵活属性 + SKU列表）
db.products.insertOne({
  _id: ObjectId("..."),
  name: "小米14 Pro",
  brand: "Xiaomi",
  category: "smartphone",
  basePrice: 3999,

  // 商品规格（不同品类属性完全不同，MongoDB灵活Schema完美适配）
  attributes: {
    screen: "6.73英寸 OLED",
    processor: "骁龙8 Gen 3",
    camera: "50MP莱卡主摄",
    battery: "4880mAh",
    os: "MIUI 15"
  },

  // SKU列表
  skus: [
    {
      skuId: "SKU-001",
      specs: { color: "钛金黑", storage: "256GB", ram: "12GB" },
      price: 3999,
      stock: 100,
      images: ["https://cdn.example.com/black-front.jpg"]
    },
    {
      skuId: "SKU-002",
      specs: { color: "钛金黑", storage: "512GB", ram: "12GB" },
      price: 4299,
      stock: 50,
      images: ["https://cdn.example.com/black-front.jpg"]
    }
  ],

  // 预计算统计（计算模式）
  totalStock: 150,
  minPrice: 3999,
  maxPrice: 4299,
  salesCount: 5678,
  avgRating: 4.8,
  reviewCount: 1234,
  status: "on_sale"
})

// 多维度商品过滤（电商搜索）
db.products.find({
  category: "smartphone",
  minPrice: { $lte: 4000 },
  maxPrice: { $gte: 3000 },
  status: "on_sale",
  totalStock: { $gt: 0 }
})

// 查找存储512GB的黑色在售SKU
db.products.find({
  "skus": {
    $elemMatch: {
      "specs.storage": "512GB",
      "specs.color": "钛金黑",
      "stock": { $gt: 0 }
    }
  }
})

// 索引
db.products.createIndex({ category: 1, minPrice: 1, maxPrice: 1 })
db.products.createIndex({ brand: 1, category: 1 })
db.products.createIndex({ "skus.skuId": 1 }, { unique: true })
db.products.createIndex(
  { name: "text", brand: "text" },
  { default_language: "none" }
)
```

---

## 案例3：实时日志分析（时间序列 + 桶模式）

```javascript
// 日志桶文档（每小时一个桶，减少文档数量）
db.log_buckets.insertOne({
  service: "payment-service",
  level: "ERROR",
  hour: ISODate("2024-01-15T10:00:00Z"),  // 桶标识
  logs: [
    {
      ts: ISODate("2024-01-15T10:03:25Z"),
      traceId: "T-001",
      message: "支付超时",
      details: { orderId: "ORD-001", amount: 999 }
    },
    {
      ts: ISODate("2024-01-15T10:07:12Z"),
      traceId: "T-002",
      message: "数据库连接失败"
    }
  ],
  count: 2,
  firstTs: ISODate("2024-01-15T10:03:25Z"),
  lastTs: ISODate("2024-01-15T10:07:12Z")
})

// TTL索引：30天后自动删除
db.log_buckets.createIndex(
  { hour: 1 },
  { expireAfterSeconds: 30 * 24 * 3600 }
)

// 实时追加日志到桶（upsert模式）
db.log_buckets.updateOne(
  {
    service: "payment-service",
    level: "ERROR",
    hour: ISODate("2024-01-15T10:00:00Z")
  },
  {
    $push: { logs: { ts: new Date(), traceId: "T-003", message: "新错误" } },
    $inc: { count: 1 },
    $min: { firstTs: new Date() },
    $max: { lastTs: new Date() }
  },
  { upsert: true }
)

// 过去1小时各服务错误统计
db.log_buckets.aggregate([
  {
    $match: {
      level: "ERROR",
      hour: { $gte: new Date(Date.now() - 3600 * 1000) }
    }
  },
  {
    $group: {
      _id: "$service",
      totalErrors: { $sum: "$count" }
    }
  },
  { $sort: { totalErrors: -1 } }
])

// 错误趋势（展开桶内日志，按分钟统计）
db.log_buckets.aggregate([
  { $match: { service: "payment-service", level: "ERROR" } },
  { $unwind: "$logs" },
  {
    $group: {
      _id: {
        $dateToString: { format: "%Y-%m-%dT%H:%M", date: "$logs.ts" }
      },
      count: { $sum: 1 }
    }
  },
  { $sort: { _id: 1 } }
])
```

---

## 案例4：地理位置服务（2dsphere索引）

```java
@Data
@Document(collection = "restaurants")
public class Restaurant {
    @Id private ObjectId id;
    private String name;
    private String category;

    @GeoSpatialIndexed(type = GeoSpatialIndexType.GEO_2DSPHERE)
    private GeoJsonPoint location;  // GeoJSON Point

    private Double avgRating;
    private Integer priceLevel;
    private Boolean isOpen;
    private String address;
}

@Service
@RequiredArgsConstructor
public class RestaurantService {

    private final MongoTemplate mongoTemplate;

    /**
     * 查找附近餐厅（$near），按距离排序
     */
    public List<GeoResult<Restaurant>> findNearby(
            double longitude, double latitude, double maxKm) {

        NearQuery nearQuery = NearQuery
            .near(new Point(longitude, latitude), Metrics.KILOMETERS)
            .maxDistance(maxKm)
            .spherical(true)
            .num(20);

        nearQuery.query(new Query(Criteria.where("isOpen").is(true)));

        GeoResults<Restaurant> results = mongoTemplate.geoNear(
            nearQuery, Restaurant.class
        );
        return results.getContent();
    }

    /**
     * 查找多边形区域内的餐厅（$geoWithin）
     */
    public List<Restaurant> findInArea(List<double[]> polygonPoints) {
        Point[] points = polygonPoints.stream()
            .map(p -> new Point(p[0], p[1]))
            .toArray(Point[]::new);

        return mongoTemplate.find(
            new Query(Criteria.where("location").within(new Polygon(points))),
            Restaurant.class
        );
    }
}

@RestController
@RequestMapping("/api/restaurants")
@RequiredArgsConstructor
public class RestaurantController {

    private final RestaurantService restaurantService;

    @GetMapping("/nearby")
    public List<Map<String, Object>> getNearby(
        @RequestParam double lng,
        @RequestParam double lat,
        @RequestParam(defaultValue = "2.0") double maxKm,
        @RequestParam(required = false) String category
    ) {
        return restaurantService.findNearby(lng, lat, maxKm)
            .stream()
            .map(r -> Map.of(
                "restaurant", r.getContent(),
                "distanceKm", r.getDistance().getValue()
            ))
            .collect(java.util.stream.Collectors.toList());
    }
}
```

---

## 案例5：用户行为数据（TTL索引自动清理）

```javascript
// 用户 Session（精确到期时间）
db.user_sessions.insertOne({
  _id: ObjectId("..."),
  userId: ObjectId("..."),
  token: "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...",
  deviceInfo: { type: "mobile", os: "Android", appVersion: "2.3.1" },
  ipAddress: "192.168.1.1",
  expireAt: new Date(Date.now() + 30 * 24 * 3600 * 1000),  // 30天后过期
  createdAt: new Date()
})

// Session TTL索引（精确到达 expireAt 时删除）
db.user_sessions.createIndex(
  { expireAt: 1 },
  { expireAfterSeconds: 0 }
)

// 用户行为日志（7天自动过期）
db.user_behaviors.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 7 * 24 * 3600 }
)

db.user_behaviors.insertOne({
  userId: ObjectId("..."),
  eventType: "product_view",     // product_view/add_cart/purchase
  productId: "P001",
  sessionId: "S001",
  properties: { source: "search", keyword: "手机" },
  createdAt: new Date()
})

// 热门商品统计（24小时浏览量）
db.user_behaviors.aggregate([
  {
    $match: {
      eventType: "product_view",
      createdAt: { $gte: new Date(Date.now() - 24 * 3600 * 1000) }
    }
  },
  {
    $group: {
      _id: "$productId",
      viewCount: { $sum: 1 },
      uniqueUsers: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      productId: "$_id",
      viewCount: 1,
      uniqueUserCount: { $size: "$uniqueUsers" },
      _id: 0
    }
  },
  { $sort: { viewCount: -1 } },
  { $limit: 20 }
])

// 用户购买漏斗分析
db.user_behaviors.aggregate([
  {
    $match: {
      createdAt: { $gte: new Date(Date.now() - 7 * 24 * 3600 * 1000) }
    }
  },
  {
    $group: {
      _id: "$eventType",
      userCount: { $addToSet: "$userId" }
    }
  },
  {
    $project: {
      eventType: "$_id",
      userCount: { $size: "$userCount" },
      _id: 0
    }
  }
])
```


---

# Part 11: 性能调优

## 11.1 慢查询分析（Profiler）

```javascript
// ==================== 开启慢查询分析 ====================

// 查看当前 profiler 配置
db.getProfilingStatus()

// 设置 profiler level
// level 0: 关闭
// level 1: 只记录慢操作（超过 slowms）
// level 2: 记录所有操作（慎用，影响性能）
db.setProfilingLevel(1, { slowms: 100 })   // 记录超过100ms的操作

// ==================== 分析慢查询 ====================

// 查询最近的慢操作（按时间降序）
db.system.profile.find().sort({ ts: -1 }).limit(10).pretty()

// 重要字段说明：
// op: 操作类型（query/update/insert/remove/command）
// ns: 命名空间（db.collection）
// millis: 执行时间（毫秒）
// nreturned: 返回文档数
// keysExamined: 扫描索引条目数
// docsExamined: 扫描文档数
// planSummary: 查询计划摘要（COLLSCAN=全扫描，IXSCAN=索引扫描）

// 找出最慢的查询
db.system.profile.find(
  { op: "query", millis: { $gt: 500 } }
).sort({ millis: -1 }).limit(5)

// 统计按命名空间分组的慢查询
db.system.profile.aggregate([
  { $match: { millis: { $gt: 100 } } },
  {
    $group: {
      _id: "$ns",
      count: { $sum: 1 },
      avgTime: { $avg: "$millis" },
      maxTime: { $max: "$millis" }
    }
  },
  { $sort: { avgTime: -1 } }
])

// 找出全集合扫描（COLLSCAN）的操作 - 需要添加索引
db.system.profile.find({
  "planSummary": /COLLSCAN/,
  millis: { $gt: 50 }
}).sort({ millis: -1 })
```

---

## 11.2 内存使用优化（WiredTiger缓存）

```javascript
// 查看缓存统计
const cacheStats = db.serverStatus().wiredTiger.cache
printjson({
  "当前缓存使用量(GB)": (cacheStats["bytes currently in the cache"] / 1024 / 1024 / 1024).toFixed(2),
  "最大缓存配置(GB)": (cacheStats["maximum bytes configured"] / 1024 / 1024 / 1024).toFixed(2),
  "脏页占比(%)": ((cacheStats["tracked dirty bytes in the cache"] /
                   cacheStats["maximum bytes configured"]) * 100).toFixed(2),
  "读取页数": cacheStats["pages read into cache"],
  "请求页数": cacheStats["pages requested from the cache"]
})
```

```yaml
# mongod.conf 缓存配置最佳实践
storage:
  wiredTiger:
    engineConfig:
      # 公式：max(256MB, (totalRAM - 1GB) * 0.5)
      # 例如：16GB RAM -> cacheSizeGB = 7.5
      cacheSizeGB: 7.5
      journalCompressor: snappy
    collectionConfig:
      blockCompressor: zstd      # 最佳压缩率+性能
    indexConfig:
      prefixCompression: true    # 索引前缀压缩（默认开启）
```

---

## 11.3 连接池配置

```yaml
# Spring Boot application.yml
spring:
  data:
    mongodb:
      uri: "mongodb://localhost:27017/mydb?maxPoolSize=20&minPoolSize=5&waitQueueTimeoutMS=5000&connectTimeoutMS=3000&socketTimeoutMS=30000"
```

```java
@Configuration
public class MongoConfig extends AbstractMongoClientConfiguration {

    @Override
    protected String getDatabaseName() { return "mydb"; }

    @Override
    public MongoClient mongoClient() {
        ConnectionPoolSettings poolSettings = ConnectionPoolSettings.builder()
            .maxSize(20)                                    // 最大连接数
            .minSize(5)                                     // 最小连接数
            .maxWaitTime(5, TimeUnit.SECONDS)               // 等待连接最大时间
            .maxConnectionIdleTime(60, TimeUnit.SECONDS)    // 连接空闲最大时间
            .maxConnectionLifeTime(300, TimeUnit.SECONDS)   // 连接最大生命周期
            .build();

        SocketSettings socketSettings = SocketSettings.builder()
            .connectTimeout(3, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build();

        MongoClientSettings settings = MongoClientSettings.builder()
            .applyConnectionString(new ConnectionString("mongodb://localhost:27017"))
            .applyToConnectionPoolSettings(b -> b.applySettings(poolSettings))
            .applyToSocketSettings(b -> b.applySettings(socketSettings))
            .writeConcern(WriteConcern.MAJORITY.withWTimeout(5, TimeUnit.SECONDS))
            .readPreference(ReadPreference.secondaryPreferred())
            .build();

        return MongoClients.create(settings);
    }
}
```

---

## 11.4 写入性能优化

```javascript
// 低效：循环单条插入（N次网络往返）
for (const doc of documents) {
  db.orders.insertOne(doc)
}

// 高效：批量插入（1次网络往返）
db.orders.insertMany(documents, { ordered: false })
// ordered: false - 无序插入，出错不影响其他文档，性能更好

// bulkWrite 混合操作（无序模式更快）
db.orders.bulkWrite(
  operations,
  {
    ordered: false,          // 无序：出错继续，性能更好
    writeConcern: { w: 1 }   // 可适当降低写关注级别
  }
)

// 对于不重要的日志数据，可以降低写关注
db.access_logs.insertMany(
  logs,
  { writeConcern: { w: 0 } }  // 不等待确认（最快）
)

// 重要数据仍要保证一致性
db.financial_records.insertOne(
  record,
  { writeConcern: { w: "majority", j: true } }
)
```

---

## 11.5 数据压缩

```javascript
// 查看压缩比
const stats = db.runCommand({ collStats: "orders" })
printjson({
  "数据大小(MB)": (stats.size / 1024 / 1024).toFixed(2),
  "存储大小(MB)": (stats.storageSize / 1024 / 1024).toFixed(2),
  "压缩比": (stats.size / stats.storageSize).toFixed(2)
})

// 创建集合时指定压缩算法
db.createCollection("logs", {
  storageEngine: {
    wiredTiger: {
      configString: "block_compressor=zstd"
    }
  }
})
```

| 压缩算法 | 压缩率 | CPU开销 | 推荐场景 |
|---------|--------|---------|---------|
| none | 无 | 无 | 临时数据，内存充足 |
| snappy（默认）| ~35% | 低 | 热数据，读写频繁 |
| zlib | ~60% | 中等 | 冷数据，存储优先 |
| zstd（推荐）| ~65% | 低-中 | 通用，最佳平衡 |

---

## 11.6 常见性能问题排查

```
性能问题排查清单：

1. 查询慢
   -> explain("executionStats") 查看是否 COLLSCAN
   -> 检查索引是否存在，是否被使用
   -> 检查索引顺序是否符合 ESR 规则
   -> 检查是否内存排序（SORT stage）- 考虑添加排序索引

2. 写入慢
   -> 检查写关注配置是否过高
   -> 检查索引数量是否过多（写入时需维护每个索引）
   -> 检查是否使用了单条插入（改用 insertMany）

3. 内存不足
   -> 检查 WiredTiger 缓存命中率
   -> 增大 cacheSizeGB 配置
   -> 检查是否有大量内存排序（$sort without index）

4. 连接数耗尽
   -> 检查 maxPoolSize 配置
   -> 检查是否有连接泄漏（未关闭的连接）
   -> 检查是否有大量长时间事务占用连接

5. 复制延迟高
   -> 检查 rs.printSecondaryReplicationInfo()
   -> 检查网络带宽和延迟
   -> 检查 Secondary 是否有大量查询影响复制

常用诊断命令：
  db.serverStatus()              // 服务器整体状态
  db.currentOp()                 // 当前正在执行的操作
  db.killOp(opid)                // 终止某个操作
  db.stats()                     // 数据库统计
  db.collection.stats()          // 集合统计
  db.adminCommand({ top: 1 })    // 各集合操作统计
```

---

# Part 12: 常见面试题 FAQ

## Q1: MongoDB 是什么？和关系型数据库有什么区别？

```
MongoDB 是文档型 NoSQL 数据库，核心区别：

数据模型：关系型用表/行/列，MongoDB 用文档（JSON/BSON）
Schema：  关系型强Schema（需DDL定义），MongoDB 动态Schema（灵活）
关联：    关系型用JOIN，MongoDB 优先嵌入文档，必要时用 $lookup
扩展性：  关系型主要垂直扩展，MongoDB 支持水平分片（Scale Out）
事务：    关系型事务成熟，MongoDB 4.0+ 才支持多文档事务

适用场景：
  数据结构多变（CMS、电商SKU）        -> MongoDB
  复杂关联查询（ERP、财务）            -> 关系型
  需要水平扩展（大数据量）             -> MongoDB
  严格 ACID 事务（金融核心）           -> 关系型（或两者结合）
```

## Q2: MongoDB 如何保证高可用？

```
三层保障机制：

1. 副本集（Replica Set）
   - 1个Primary + 多个Secondary
   - 数据自动同步（oplog异步复制）
   - Primary宕机自动选举（通常<12秒）

2. 写关注（Write Concern）
   - w:majority 确保多数节点写入成功
   - j:true 确保 Journal 落盘

3. 读偏好（Read Preference）
   - 可从Secondary读，分散压力
   - 副本集成员宕机不影响读可用性
```

## Q3: MongoDB 索引有哪些类型？分别适用什么场景？

```
单字段索引：    最基础，适合单字段查询/排序
复合索引：      多字段查询，遵循ESR规则（等值->排序->范围）
多键索引：      数组字段自动创建，索引数组中每个元素
文本索引：      全文搜索（中文支持有限，推荐用ES）
地理空间索引：  2dsphere（经纬度）/ 2d（平面坐标）
TTL索引：       日期字段，定时自动删除过期文档
稀疏索引：      只索引有该字段的文档，节省空间
部分索引：      只索引满足条件的文档，更精细控制
哈希索引：      等值查询，主要用作分片键
唯一索引：      保证字段值唯一
```

## Q4: 什么是 ESR 规则？为什么要遵循它？

```
ESR 规则是复合索引的字段排序原则：
  E - Equality（等值查询字段）放最前
  S - Sort（排序字段）放中间
  R - Range（范围查询字段）放最后

原因：
  1. 等值字段放前面：精确过滤，最大程度减少扫描范围
  2. 排序字段放中间：等值确定后，相同前缀内的数据已有序，
     无需内存排序（避免 SORT stage）
  3. 范围字段放最后：范围查询后数据不连续，放在最后可以
     充分利用前面的精确过滤

示例：
  查询：find({status:"active", age:{$gt:18}}).sort({createdAt:-1})
  错误：{age:1, status:1, createdAt:-1}  // R在E前面
  正确：{status:1, createdAt:-1, age:1}  // E->S->R
```

## Q5: 什么是覆盖查询？如何实现？

```
覆盖查询（Covered Query）：
  查询所需的所有字段都在索引中，MongoDB 无需访问原文档，
  直接从索引返回数据。性能极高，不需要随机IO读文档。

实现方法：
  1. 创建包含查询条件字段和返回字段的复合索引
  2. 查询时投影只返回索引中的字段，且必须排除 _id
     （_id 不在自定义索引中，不排除会导致无法覆盖）

示例：
  db.users.createIndex({ status:1, username:1, email:1 })
  db.users.find(
    { status: "active" },
    { username:1, email:1, _id:0 }    // 必须排除_id
  )

验证：explain("executionStats") 中 totalDocsExamined = 0
```

## Q6: 聚合管道的性能优化有哪些？

```
1. $match 前置（最重要）
   利用索引过滤，减少后续阶段的数据量
   $match 一定要在 $group、$lookup 之前

2. $project 早期减少字段
   早点删除不需要的字段，减少内存占用
   在 $unwind 之前投影可以大量减少数据

3. $sort + $limit 组合优化
   MongoDB 会合并这两个阶段，使用 Top-K 算法
   不会对所有数据排序，只维护前K个

4. $lookup 性能注意
   $lookup 相当于 JOIN，性能有代价
   建议在 localField/foreignField 上都有索引
   尽量在 $lookup 前用 $match 减少关联的数据量
   考虑反范式化（嵌入文档）替代 $lookup

5. allowDiskUse: true
   超过100MB内存限制时使用磁盘临时文件
   会降低性能，但避免出错
```

## Q7: 副本集的选举机制是什么？如何保证数据不丢失？

```
选举机制（基于Raft协议变体）：
  1. 触发条件：Primary不可达超过10秒（electionTimeoutMillis）
  2. 候选者：任意Secondary增加term，发起投票请求
  3. 投票规则：每个term只投一票，候选者oplog不比自己旧
  4. 当选条件：获得多数票（>N/2）
  5. 选举时间：通常 < 12秒

数据不丢失保障：
  使用 w:"majority" 写关注 + j:true
  -> 写操作必须在多数节点的Journal写入成功才返回
  -> 即使Primary宕机，数据已在多数节点持久化

最多丢失数据的情况：
  w:1（只Primary确认）-> Primary宕机且未复制到Secondary
  解决：重要数据使用 w:"majority"
```

## Q8: MongoDB 分片键选择的原则和注意事项？

```
分片键选择原则：

1. 高基数（Cardinality）
   好：userId（数百万唯一值）
   差：status（3-5种值，数据倾斜）

2. 写入分布均匀
   好：userId 哈希分片（均匀分布）
   差：时间戳范围分片（单调递增，全写最后一个Shard）

3. 覆盖主要查询（避免广播查询）
   好：应用总是按 userId 查询 -> 分片键包含 userId
   差：分片键是 region，但应用按 userId 查询 -> 广播

4. 不可变性
   分片键值一旦设置不应更改（MongoDB 4.4+ 支持但代价高）

5. 复合分片键
   可以组合多个字段：{ userId: 1, createdAt: 1 }
   好处：既保证均匀分布，又支持按用户的范围查询

常见陷阱：
  ObjectId 范围分片 -> 写入热点（单调递增）
  建议：ObjectId 哈希分片 或 选其他字段
```

## Q9: MongoDB 事务和单文档操作有什么区别？

```
单文档操作：
  MongoDB 对单个文档的操作天然是原子的
  （包括嵌套文档和数组）
  不需要事务，性能更好

多文档事务（4.0+）：
  跨文档/集合/分片的原子操作
  提供 ACID 保证（快照隔离）
  有性能开销（锁、日志、网络往返）

什么时候用事务：
  需要跨多个文档/集合的原子性时：
  账户转账（A账户-B账户+流水记录）
  订单 + 库存同时更新

什么时候不用事务：
  单文档更新（天然原子）
  批量写入（用 bulkWrite）
  可以反范式化的场景

事务注意事项：
  事务时间要短（默认60秒超时）
  不要在事务中做耗时操作
  副本集环境才支持事务
```

## Q10: 嵌入式文档 vs 引用，如何选择？

```
嵌入式文档（Embedded）适合：
  1. 数据"总是在一起"访问（文章+作者信息）
  2. 子数据数量有限（订单items最多100条）
  3. 子数据不被多个父文档共享
  4. 子数据很少单独更新

引用（Reference）适合：
  1. 数据需要单独访问（评论列表独立分页）
  2. 子数据无限增长（用户的所有历史订单）
  3. 数据被多个文档共享（产品被多个订单引用）
  4. 嵌入会超过16MB限制
  5. 需要单独更新的数据

黄金法则：
  "一起读取 -> 嵌入，单独读取 -> 引用"
  "数据有限 -> 嵌入，无限增长 -> 引用"
```

## Q11: MongoDB 的 BSON 类型和 JSON 有什么区别？

```
BSON (Binary JSON) 比 JSON 多支持：
  ObjectId   - 12字节唯一ID（带时间戳）
  Date       - 64位UTC毫秒时间戳（JSON没有Date类型）
  Int32/Int64 - 明确区分整型（JSON只有Number）
  Decimal128 - 高精度小数（金融计算）
  Binary     - 二进制数据
  Timestamp  - 内部时间戳（副本集用）
  Regex      - 正则表达式

BSON 的优势：
  1. 记录文档/字段长度，支持快速跳过
  2. 二进制编码，解析比纯文本JSON快
  3. 支持更丰富的数据类型
```

## Q12: MongoDB 如何处理 16MB 文档大小限制？

```
设计模式解决方案：

1. 子集模式（Subset Pattern）
   主文档只存热数据（如Top 10评论），
   完整数据存独立集合

2. 桶模式（Bucket Pattern）
   将时序细粒度数据按时间聚合（如按小时聚合传感器数据）
   避免海量小文档，也避免单个超大文档

3. 异常值模式（Outlier Pattern）
   普通文档正常存储，
   超大文档（如大V的百万粉丝）溢出到额外集合

4. 引用替代嵌入
   数组无限增长时，改用单独集合 + 引用

5. GridFS
   超过16MB的文件（图片/视频）使用GridFS存储
   自动分块存储，支持流式读取
```

## Q13: WiredTiger 如何实现文档级并发控制？

```
WiredTiger 使用 MVCC（多版本并发控制）：

1. 每个写操作创建文档的新版本
2. 读操作看到事务开始时的快照版本
3. 不同事务可以同时修改不同文档（文档级锁）
4. 同一文档的并发写操作：后写者等待或重试（写写冲突）

比较：
  MMAPv1（旧引擎）: 集合级锁（同时只能一个写操作）
  WiredTiger:       文档级锁（不同文档并发写，性能大幅提升）

注意：DDL操作（如创建集合）仍需要全局锁
     但持续时间极短，基本不影响业务
```

## Q14: MongoDB 的读关注（Read Concern）有哪些级别？

```
local（默认）:
  读取本地节点最新数据，可能读到未多数确认的数据
  最快，适合一般场景

available:
  类似local，分片集群不保证孤儿文档过滤

majority:
  只读取多数节点已确认的数据
  保证读到的数据不会回滚
  需要开启 enableMajorityReadConcern

linearizable:
  读取多数节点已确认的最新数据 + 保证线性一致性
  性能最低，适合强一致性场景

snapshot（4.0+）:
  事务级别，读取事务开始时的一致性快照
  只在多文档事务中使用
```

## Q15: 如何设计 MongoDB 的时间序列数据模型？

```
反模式：每条数据一个文档
  -> 海量小文档，索引开销大，查询聚合慢

推荐：桶模式（Bucket Pattern）
  -> 按时间窗口（小时/天）聚合多条数据到一个文档
  -> 显著减少文档数量
  -> 预计算统计字段（sum/min/max）加速查询

MongoDB 5.0+ 时间序列集合（Time Series Collection）：

db.createCollection("sensor_data", {
  timeseries: {
    timeField: "timestamp",    // 时间字段
    metaField: "sensorId",     // 元数据字段（用于分组）
    granularity: "seconds"     // 时间粒度：seconds/minutes/hours
  },
  expireAfterSeconds: 86400    // TTL：1天后过期
})

特点：
  MongoDB 原生支持，自动优化存储（类列式存储）
  支持 $match $group $sort 等聚合操作
  配合 TTL 自动清理过期数据
  只支持 insert（不支持 update/delete 特定条目）

适用场景：
  IoT传感器数据、监控指标、股票行情、用户行为日志等
```

---

## 附录：MongoDB 常用命令速查

```javascript
// ==================== 数据库操作 ====================
show dbs                               // 列出所有数据库
use dbname                             // 切换/创建数据库
db.dropDatabase()                      // 删除当前数据库
db.stats()                             // 数据库统计信息

// ==================== 集合操作 ====================
show collections                       // 列出所有集合
db.createCollection("name")            // 创建集合
db.collection.drop()                   // 删除集合
db.collection.stats()                  // 集合统计
db.collection.countDocuments({})       // 精确文档总数
db.collection.estimatedDocumentCount() // 估算文档总数（快）

// ==================== 索引操作 ====================
db.collection.getIndexes()                         // 查看所有索引
db.collection.createIndex({field: 1})              // 创建索引
db.collection.dropIndex("indexName")               // 删除指定索引
db.collection.dropIndexes()                        // 删除所有索引（保留_id）
db.collection.aggregate([{ $indexStats: {} }])     // 索引使用统计

// ==================== 性能诊断 ====================
db.serverStatus()                      // 服务器状态
db.currentOp()                         // 当前操作
db.killOp(opid)                        // 终止操作
db.setProfilingLevel(1, {slowms: 100}) // 开启慢查询
db.system.profile.find().sort({ts:-1}).limit(5)    // 查看慢查询
db.collection.find(query).explain("executionStats")  // 查询分析

// ==================== 副本集 ====================
rs.status()                            // 副本集状态
rs.conf()                              // 副本集配置
rs.initiate()                          // 初始化副本集
rs.add("host:port")                    // 添加成员
rs.remove("host:port")                 // 移除成员
rs.stepDown(30)                        // 让Primary让位
rs.printReplicationInfo()              // 查看oplog信息
rs.printSecondaryReplicationInfo()     // 查看复制延迟

// ==================== 分片集群 ====================
sh.status()                            // 分片集群状态
sh.addShard("host:port")               // 添加分片
sh.shardCollection("db.col", {key:1})  // 启用分片
sh.getBalancerState()                  // 均衡器状态
sh.enableBalancing("db.col")           // 开启均衡
sh.disableBalancing("db.col")          // 关闭均衡

// ==================== 用户管理 ====================
use admin
db.createUser({
  user: "admin",
  pwd: "password",
  roles: [{ role: "root", db: "admin" }]
})
db.createUser({
  user: "appuser",
  pwd: "password",
  roles: [{ role: "readWrite", db: "mydb" }]
})
db.dropUser("username")                // 删除用户
db.changeUserPassword("user", "newpwd") // 修改密码

// ==================== 备份恢复（命令行工具）====================
// 备份
// mongodump --uri="mongodb://localhost:27017" --out=/backup/

// 恢复
// mongorestore --uri="mongodb://localhost:27017" /backup/

// 导出 JSON
// mongoexport --collection=users --out=users.json

// 导入 JSON
// mongoimport --collection=users --file=users.json
```

---

> **文档总结**
>
> 本文覆盖了 MongoDB 从基础架构到高级特性的全部核心知识：
>
> - Part 1-2: 架构原理 + CRUD 基础操作
> - Part 3:   聚合管道（最强大的数据处理能力）
> - Part 4:   索引设计（性能优化的核心）
> - Part 5:   数据建模（MongoDB 设计精髓）
> - Part 6:   事务（多文档原子性）
> - Part 7-8: 副本集 + 分片集群（高可用 + 高扩展）
> - Part 9:   Spring Boot 集成（生产实践）
> - Part 10:  完整实战案例（5个典型场景）
> - Part 11:  性能调优（慢查询 + 缓存 + 连接池）
> - Part 12:  15道高频面试题（含详细解答）
>
> 技术版本：MongoDB 7.x | Spring Boot 3.x | Java 17
