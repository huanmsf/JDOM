# Elasticsearch 详解 - 从零到精通

> 版本：基于 Elasticsearch 8.x
> 更新日期：2026年7月

---

## 目录

- [Part 1: Elasticsearch 整体架构](#part-1)
- [Part 2: 倒排索引原理（核心）](#part-2)
- [Part 3: 分词器（Analyzer）深度解析](#part-3)
- [Part 4: 数据类型与 Mapping](#part-4)
- [Part 5: Query DSL 深度解析](#part-5)
- [Part 6: 聚合（Aggregations）深度解析](#part-6)
- [Part 7: 相关性评分原理](#part-7)
- [Part 8: 索引管理与优化](#part-8)
- [Part 9: 集群架构与高可用](#part-9)
- [Part 10: 写入与读取原理](#part-10)
- [Part 11: Spring Boot 集成实战](#part-11)
- [Part 12: 完整实战案例](#part-12)
- [Part 13: 性能调优](#part-13)
- [Part 14: 常见面试题 FAQ](#part-14)

---

<a name="part-1"></a>
# Part 1: Elasticsearch 整体架构

## 1.1 Elasticsearch 是什么

Elasticsearch（简称 ES）是一个基于 Apache Lucene 构建的**分布式、RESTful 风格的搜索和数据分析引擎**。它由 Shay Banon 于 2010 年创建，目前由 Elastic 公司维护，是 Elastic Stack（ELK Stack）的核心组件。

### 核心定位

```
+--------------------------------------------------+
|           Elasticsearch 核心定位                  |
+--------------------------------------------------+
|                                                  |
|   搜索引擎    +    文档数据库    +    分析平台      |
|                                                  |
|  - 全文搜索   |  - JSON存储    |  - 聚合统计      |
|  - 模糊匹配   |  - CRUD操作    |  - 实时分析      |
|  - 相关性评分 |  - Schema自由  |  - 可视化支持    |
|                                                  |
+--------------------------------------------------+
```

### ES 的核心特性

1. **分布式架构**：天然支持水平扩展，数据自动分片存储在多个节点
2. **近实时搜索**：文档写入后约 1 秒内即可被搜索到（Near Real-Time, NRT）
3. **RESTful API**：所有操作通过标准 HTTP + JSON 接口完成
4. **高可用性**：副本机制保障节点故障时服务不中断
5. **强大的查询 DSL**：支持复杂的组合查询、聚合分析、地理空间查询
6. **无 Schema**：文档结构灵活，支持动态 mapping

---

## 1.2 与传统数据库的区别

### 对比分析

| 特性 | 传统关系型数据库（MySQL） | Elasticsearch |
|------|------------------------|---------------|
| 数据模型 | 结构化表格（行+列） | 非结构化 JSON 文档 |
| 查询语言 | SQL | Query DSL（JSON） |
| 索引方式 | B+ 树索引 | 倒排索引 |
| 全文搜索 | LIKE（低效） | 原生支持（高效） |
| 事务支持 | ACID 完整支持 | 不支持跨文档事务 |
| JOIN 操作 | 完整支持 | 有限支持（nested/has_child） |
| 水平扩展 | 困难（需分库分表） | 原生支持（自动分片） |
| 数据一致性 | 强一致性 | 最终一致性 |
| 适合场景 | 业务数据 CRUD | 搜索、日志分析 |
| 写入实时性 | 毫秒级可查 | 约 1 秒延迟（NRT） |

### 适用场景

**Elasticsearch 适合的场景：**

```
+------------------------------------------+
|         ES 适用场景                        |
+------------------------------------------+
|  1. 全文搜索        - 电商商品搜索          |
|                     - 文章内容检索          |
|                     - 代码搜索（GitHub）    |
|                                           |
|  2. 日志分析        - 应用日志聚合          |
|                     - 安全审计日志          |
|                     - 运维监控（ELK）       |
|                                           |
|  3. 实时分析        - 业务指标仪表板        |
|                     - 用户行为分析          |
|                     - 实时统计报表          |
|                                           |
|  4. 地理空间搜索    - 附近的人/门店         |
|                     - 地图数据检索          |
|                                           |
|  5. 推荐系统        - 相似商品推荐          |
|                     - 个性化内容推荐        |
+------------------------------------------+
```

**不适合 ES 的场景：**
- 需要强事务的金融交易
- 频繁更新的业务数据主库
- 需要复杂 JOIN 的关系查询
- 数据量极小（< 10万条）且无全文搜索需求

---

## 1.3 核心概念详解

### 概念对比表：ES vs 关系型数据库

| Elasticsearch | 关系型数据库 | 说明 |
|--------------|------------|------|
| Index（索引） | Database（数据库）| 文档的逻辑集合 |
| ~~Type（类型）~~ | Table（表）| 7.x 后废弃，8.x 完全移除 |
| Document（文档）| Row（行）| 最小数据存储单元，JSON格式 |
| Field（字段） | Column（列）| 文档的属性 |
| Mapping（映射）| Schema（表结构）| 字段类型定义 |
| Shard（分片） | Partition（分区）| 数据分布单元 |
| Replica（副本）| Slave（从库）| 数据冗余备份 |

#### 1.3.1 Index（索引）

Index 是 ES 中数据的逻辑命名空间，相当于数据库中的一个"数据库"。

```
Index: products
+---------------------------------------------+
|  Document 1:                                |
|  {                                          |
|    "_id": "1",                              |
|    "name": "iPhone 15 Pro",                 |
|    "price": 8999,                           |
|    "category": "手机"                        |
|  }                                          |
|                                             |
|  Document 2:                                |
|  {                                          |
|    "_id": "2",                              |
|    "name": "MacBook Pro M3",                |
|    "price": 14999,                          |
|    "category": "电脑"                        |
|  }                                          |
+---------------------------------------------+
```

#### 1.3.2 Shard（分片）

分片是 ES 实现分布式存储的核心机制。每个 Index 可以被切分为多个 Primary Shard，分散存储在集群的不同节点上。

```
Index: products (3个主分片 + 每个1个副本)

+------------------+  +------------------+  +------------------+
|    Node 1        |  |    Node 2        |  |    Node 3        |
|  +------------+  |  |  +------------+  |  |  +------------+  |
|  | Shard P0   |  |  |  | Shard P1   |  |  |  | Shard P2   |  |
|  | Doc1,Doc4  |  |  |  | Doc2,Doc5  |  |  |  | Doc3,Doc6  |  |
|  +------------+  |  |  +------------+  |  |  +------------+  |
|  +------------+  |  |  +------------+  |  |  +------------+  |
|  | Shard R1   |  |  |  | Shard R2   |  |  |  | Shard R0   |  |
|  | (P1副本)   |  |  |  | (P2副本)   |  |  |  | (P0副本)   |  |
|  +------------+  |  |  +------------+  |  |  +------------+  |
+------------------+  +------------------+  +------------------+
```

**分片路由算法：**

```
shard = hash(document_id) % number_of_primary_shards
```

这也是为什么**主分片数量创建后不能更改**的原因——改变分片数会导致路由计算结果不同，无法找到已有文档。

#### 1.3.3 Replica（副本）

每个主分片（Primary Shard）可以有零个或多个副本（Replica Shard）。副本的作用：

- **高可用**：主分片所在节点宕机时，副本可升级为主分片
- **提高读取吞吐量**：搜索请求可以在主分片和副本上并行执行

#### 1.3.4 Node（节点）

Node 是 ES 集群中的单个服务器实例。节点类型详见 Part 9。

#### 1.3.5 Cluster（集群）

Cluster 是多个节点的集合，共同存储数据并提供搜索能力。集群通过唯一的名称标识（默认 `elasticsearch`）。

---

## 1.4 整体架构图

```
+=====================================================================+
|                    Elasticsearch 集群架构                            |
+=====================================================================+
|                                                                     |
|   Client（客户端）                                                   |
|   +----------+  +----------+  +----------+                         |
|   | REST API |  | Java API |  | 其他SDK  |                         |
|   +----+-----+  +-----+----+  +----+-----+                         |
|        |              |            |                                 |
|        +-------- HTTP/9200 --------+                                |
|                       |                                             |
|   +-------------------v-------------------------------------+       |
|   |              Coordinating Node（协调节点）               |       |
|   |   接收请求 -> 路由分发 -> 汇总结果 -> 返回客户端            |       |
|   +---+-------------------+-------------------+---+         |       |
|       |                   |                   |             |       |
|   +---v---+           +---v---+           +---v---+         |       |
|   | Node1 |           | Node2 |           | Node3 |         |       |
|   |Master |           | Data  |           | Data  |         |       |
|   +-------+           +-------+           +-------+         |       |
|   | P0    |           | P1    |           | P2    |         |       |
|   | R1    |           | R2    |           | R0    |         |       |
|   +-------+           +-------+           +-------+         |       |
|                                                             |       |
|   [Shard P = Primary Shard  R = Replica Shard]             |       |
+-------------------------------------------------------------+-------+
```

---

## 1.5 ES vs Solr 对比

Solr 也是基于 Lucene 的搜索引擎，两者经常被比较：

| 对比维度 | Elasticsearch | Solr |
|---------|--------------|------|
| 创建时间 | 2010年 | 2004年 |
| 分布式 | 原生分布式 | 依赖 ZooKeeper（SolrCloud）|
| 实时搜索 | 更好（近实时1s） | 略差 |
| 配置方式 | 动态配置，API驱动 | XML配置文件为主 |
| 社区活跃度 | 非常活跃 | 相对较低 |
| 文档格式 | JSON | XML/JSON/CSV |
| 聚合分析 | 非常强大 | 功能较弱 |
| 学习曲线 | 相对平缓 | 较陡 |
| 企业支持 | Elastic 公司 | Apache 基金会 |
| 日志分析 | ELK生态，优势明显 | 需要额外配置 |

**结论**：目前业界 ES 的市场占有率远高于 Solr，新项目优先选择 ES。

---

## 1.6 ES 版本演进

### 重要版本变化

```
ES 版本演进时间线（注意：没有3.x和4.x版本）

+------+------+------+------+------+------+
| 1.x  | 2.x  | 5.x  | 6.x  | 7.x  | 8.x  |
| 2014 | 2015 | 2016 | 2017 | 2019 | 2022 |
+------+------+------+------+------+------+
```

### 5.x 重要变化
- 引入 BM25 评分算法替代 TF/IDF
- 引入 `keyword` 类型（替代 `not_analyzed`）
- `string` 类型废弃，改用 `text` + `keyword`
- 引入 Ingest Node
- 默认使用 `_doc` type

### 6.x 重要变化
- 一个 Index 只能有一个 type（为 7.x 移除 type 做准备）
- 引入 Sequence Numbers（用于副本同步）
- 引入 Cross Cluster Replication (CCR)
- 滚动升级改进

### 7.x 重要变化（里程碑版本）
- **完全移除 type**（_doc 作为默认值）
- 引入新的集群协调层（基于 Raft 算法），替代 Zen Discovery
- `minimum_master_nodes` 自动配置
- 默认主分片数从 5 改为 **1**
- 引入 High-Level REST Client
- 引入 Data Streams

### 8.x 重要变化
- **默认开启安全认证**（TLS + 用户名密码）
- 移除旧版 REST Client，推荐使用新版 Java API Client
- 引入 kNN 向量搜索（ANN）
- Natural Language Processing (NLP) 支持
- 更好的 Kubernetes 支持

---
<a name="part-2"></a>
# Part 2: 倒排索引原理（核心）

## 2.1 为什么需要倒排索引

传统数据库使用 B+ 树索引，适合精确查找和范围查找。但对于全文搜索场景，传统索引效率极低。

**问题示例**：
```sql
-- 查找包含"苹果手机"的商品，传统数据库需要全表扫描
SELECT * FROM products WHERE description LIKE '%苹果手机%';
```

这种查询无法利用 B+ 树索引，必须逐行扫描所有数据，性能极差。

---

## 2.2 正排索引 vs 倒排索引

### 正排索引（Forward Index）

正排索引是"文档 -> 词项"的映射，即给定文档，找到其包含的所有词。

```
正排索引（Forward Index）
+--------+----------------------------------+
| 文档ID  | 包含的词项                        |
+--------+----------------------------------+
| Doc1   | 苹果, 手机, iPhone, 5G, 屏幕      |
| Doc2   | 华为, 手机, Mate, 5G, 鸿蒙        |
| Doc3   | 苹果, 笔记本, MacBook, M3         |
| Doc4   | 三星, 手机, Galaxy, 折叠屏        |
+--------+----------------------------------+

问题：搜索包含"苹果"的文档，需要扫描所有行 -> O(n)
```

### 倒排索引（Inverted Index）

倒排索引是"词项 -> 文档"的映射，即给定词项，找到包含该词的所有文档。

```
倒排索引（Inverted Index）
+--------+----------------------------+
| 词项   | 包含该词的文档列表（倒排列表）  |
+--------+----------------------------+
| 苹果   | Doc1, Doc3                 |
| 手机   | Doc1, Doc2, Doc4           |
| iPhone | Doc1                       |
| 5G     | Doc1, Doc2                 |
| 华为   | Doc2                       |
| Mate   | Doc2                       |
| 鸿蒙   | Doc2                       |
| 笔记本 | Doc3                       |
| MacBook| Doc3                       |
| 三星   | Doc4                       |
| 折叠屏 | Doc4                       |
+--------+----------------------------+

搜索"苹果"：直接查词项字典 -> 返回 [Doc1, Doc3] -> O(1)
```

---

## 2.3 倒排索引的组成结构

完整的倒排索引包含以下组成部分：

```
+==============================================================+
|                    完整倒排索引结构                            |
+==============================================================+
|                                                              |
|  Term Dictionary（词项字典）                                  |
|  +--------------------------------------------------+        |
|  | 词项(Term) | 文档频率(DF) | 倒排列表指针(Pointer) |        |
|  +--------------------------------------------------+        |
|  | 苹果       |     2       |  --> Posting List 1   |        |
|  | 手机       |     3       |  --> Posting List 2   |        |
|  | iPhone     |     1       |  --> Posting List 3   |        |
|  +--------------------------------------------------+        |
|              |                   |                            |
|              v                   v                            |
|  Posting List（倒排列表）                                      |
|  +--------------------------------------------------+        |
|  | DocID | TF(词频) | Position(位置) | Offset(偏移)  |        |
|  +--------------------------------------------------+        |
|  |  1    |   2     |   [0, 5]      |  [0-2, 8-10]  |        |
|  |  3    |   1     |   [0]         |  [0-2]        |        |
|  +--------------------------------------------------+        |
|                                                              |
|  Term Index（词项索引，FST结构）                               |
|  用于快速定位 Term Dictionary 中的词项                         |
|  +--------------------------------------------------+        |
|  | FST 压缩前缀树，存储在内存中                        |        |
|  | 苹 -> 果 -> [offset in Term Dict]                 |        |
|  +--------------------------------------------------+        |
+==============================================================+
```

### 倒排列表详细字段说明

| 字段 | 说明 | 用途 |
|------|------|------|
| DocID | 文档 ID | 标识哪个文档 |
| TF (Term Frequency) | 词在该文档中出现的次数 | 计算相关性评分 |
| Position | 词在文档中的位置序号 | 支持短语搜索（match_phrase）|
| Offset | 词在文档中的字节偏移 | 支持高亮显示 |

---

## 2.4 Term Dictionary 的 FST 压缩存储

### 为什么需要 FST

词项字典可能包含数百万个词项（尤其是日志场景），如果全部加载到内存，内存消耗巨大。ES 使用 **FST（Finite State Transducer，有限状态转换器）** 来压缩存储词项字典的索引部分。

### FST 原理

FST 是一种有向无环图，利用词项的公共前缀/后缀进行压缩。

**示例**：存储词项 [mop, moth, pop, star, stop, top]

```
普通存储方式（每个词独立存储）：
mop  -> 3 bytes
moth -> 4 bytes
pop  -> 3 bytes
star -> 4 bytes
stop -> 4 bytes
top  -> 3 bytes
Total: 21 bytes + 指针开销

FST 压缩后（共享前缀/后缀）：

        (start)
           |
     +-----+-----+
     |           |
    'm'          's'
     |            |
    'o'    +-----+-----+
     |     |           |
  +--+---+'t'         't'
  |       |            |
 'p'[3] 'th'[4]   +----+----+
                  |         |
                 'a'        'o'
                  |          |
                 'r'[4]     'p'[4]

内存占用大幅减少，且支持前缀查询
```

### FST 的特性

1. **极高压缩率**：通常比原始数据小 10-100 倍
2. **存储在内存**：FST 足够小，可以常驻内存
3. **快速查找**：O(len) 时间复杂度，len 为词项长度
4. **前缀遍历**：天然支持前缀查询（prefix query）

---

## 2.5 Posting List 的压缩：Frame Of Reference (FOR)

### 问题背景

高频词（如"的"、"手机"）的倒排列表可能包含数百万个文档 ID。这些 DocID 如果直接存储，每个需要 4 字节，100万个 DocID 就需要 4MB。

### FOR 算法原理

**步骤一：Delta 编码（差值编码）**

```
原始 DocID 列表（升序）：
[73, 300, 302, 332, 343, 372]

Delta 编码（只存相邻差值）：
[73, 227, 2, 30, 11, 29]

数字变小了，可以用更少的 bit 存储
```

**步骤二：分块压缩**

```
将 Delta 列表分成固定大小的块（每块256个数）：

块1: [73, 227, 2, 30, 11, 29, ...]
     找到块内最大值: 227
     227 需要 8 bits 存储
     -> 块内所有数都用 8 bits 存储

块2: [1, 2, 1, 3, 2, 1, ...]
     找到块内最大值: 3
     3 需要 2 bits 存储
     -> 块内所有数都用 2 bits 存储

压缩效果：原来每个数字 32 bits，现在只需 2-8 bits
压缩率：可达 75%-94%
```

**FOR 压缩图示：**

```
+------------------------------------------+
|        FOR (Frame Of Reference) 算法      |
+------------------------------------------+
|                                          |
|  原始列表: [1000, 1002, 1003, 1100, 1105]|
|                                          |
|  Step1 - Delta编码:                      |
|  [1000, 2, 1, 97, 5]                    |
|                                          |
|  Step2 - 分块(块大小=4):                 |
|  Block1: [1000, 2, 1, 97]               |
|           最大值=1000, 需要10 bits        |
|  Block2: [5]                            |
|           最大值=5, 需要3 bits           |
|                                          |
|  Step3 - 存储:                           |
|  Block头: 存储每个块的位宽                 |
|  Block数据: 每个数用固定位宽存储           |
|                                          |
|  压缩前: 5 x 32 bits = 160 bits          |
|  压缩后: ~50 bits (压缩率 ~69%)           |
+------------------------------------------+
```

---

## 2.6 Roaring Bitmap 优化

### 什么是 Roaring Bitmap

对于需要进行集合运算（AND、OR、NOT）的场景（如 bool 查询的多条件组合），ES 使用 **Roaring Bitmap** 来高效存储和运算文档 ID 集合。

### 原理

Roaring Bitmap 将 32 位整数空间划分为 **65536（2^16）个桶**，每个桶对应一个 16 位的高位值，桶内存储对应的低 16 位值。

```
Roaring Bitmap 结构：

高16位 -> 容器类型
+--------+------------------+
| 0x0000 | Array Container  |  (元素少时，用数组存储低16位)
| 0x0001 | Bitmap Container |  (元素多时，用65536 bit的位图)
| 0x0002 | Run Container    |  (连续范围时，用(start,length)对存储)
| ...    | ...              |
+--------+------------------+

集合运算效率：
AND 操作：两个 Bitmap Container 做 AND -> O(n/64)，使用 SIMD 加速
OR  操作：两个 Bitmap Container 做 OR  -> O(n/64)
比遍历链表快 1000x 倍
```

---

## 2.7 联合查询加速：跳表 + Bitset

### 跳表（Skip List）

对于多条件查询，需要对多个倒排列表求交集。跳表可以加速这个过程。

```
查询：A AND B AND C

倒排列表 A: [1, 3, 5, 8, 12, 20, 25, ...]
倒排列表 B: [2, 5, 10, 12, 15, 20, ...]
倒排列表 C: [5, 7, 12, 18, 20, ...]

计算交集步骤：
1. 从最短列表 C 开始
2. 取 C[0]=5，在 A 中跳表查找 >=5 -> 找到5，在 B 中 >=5 -> 找到5
3. 5 在 A、B、C 中都存在 -> 加入结果
4. 继续处理 C[1]=7，A中找>=7 -> 8，不等于7，跳过
5. 继续处理 C[2]=12，A中找>=12 -> 12，B中找>=12 -> 12
6. 12 在 A、B、C 中都存在 -> 加入结果
7. 继续处理 C[3]=18，A中找>=18 -> 20，不等于18，跳过
8. 继续处理 C[4]=20，A中找>=20 -> 20，B中找>=20 -> 20
9. 最终结果：[5, 12, 20]

相比逐个比较，跳表使得每次比较可以跳过大量不匹配的元素，时间复杂度大幅降低
```

---
<a name="part-3"></a>
# Part 3: 分词器（Analyzer）深度解析

## 3.1 分析器（Analyzer）概述

在 ES 中，文本数据在写入和搜索时都需要经过**分析（Analysis）**过程，将原始文本转换为可被索引的词项（Token）。

```
分析器处理流程：

原始文本: "Hello World! 快乐编程 2024"
     |
     v
+------------------+
| Character Filter |  字符过滤器（可选，可多个）
| - html_strip     |  去除 HTML 标签
| - mapping        |  字符替换
| - pattern_replace|  正则替换
+------------------+
     |
     v  "Hello World 快乐编程 2024"
+------------------+
|   Tokenizer      |  分词器（必须，只能一个）
| - standard       |  按空格/标点分词
| - ik_smart       |  中文智能分词
| - ngram          |  N-gram 分词
+------------------+
     |
     v  ["Hello", "World", "快乐", "编程", "2024"]
+------------------+
|  Token Filter    |  词项过滤器（可选，可多个）
| - lowercase      |  转小写
| - stop           |  去停用词
| - synonym        |  同义词扩展
| - stemmer        |  词干提取
+------------------+
     |
     v
最终词项: ["hello", "world", "快乐", "编程", "2024"]
（小写转换后写入倒排索引）
```

---

## 3.2 内置分析器详解

### standard 分析器（默认）

ES 默认分析器，适合英文和大多数西方语言。

```json
{
  "analyzer": {
    "my_standard": {
      "type": "standard",
      "max_token_length": 255,
      "stopwords": "_none_"
    }
  }
}
```

**处理示例：**
```
输入: "The Quick Brown-Fox jumps over 2 lazy dogs!"
输出: ["the", "quick", "brown", "fox", "jumps", "over", "2", "lazy", "dogs"]

规则：
- 按空格和标点分词
- 自动转小写
- 去除标点符号
- 中文：按字符逐个分词（不适合中文！）
```

### simple 分析器

```
输入: "Hello World 2024 Test-Case"
输出: ["hello", "world", "test", "case"]

规则：
- 只保留字母字符
- 数字和特殊字符被丢弃
- 转小写
```

### whitespace 分析器

```
输入: "Hello World 2024 Test-Case"
输出: ["Hello", "World", "2024", "Test-Case"]

规则：
- 只按空格分词
- 不做任何转换
- 保留大小写和特殊字符
```

### english 分析器

```
输入: "Running foxes are very quickly"
输出: ["run", "fox", "veri", "quickli"]

规则：
- 词干提取（stemming）：running->run, foxes->fox
- 去停用词：are, very 被删除
- Porter stemmer 算法
```

### keyword 分析器

```
输入: "Hello World 2024"
输出: ["Hello World 2024"]  整体作为一个 token

规则：
- 不做任何处理
- 整个输入作为单一词项
- 适合精确匹配场景
```

---

## 3.3 中文分词器：IK Analyzer

标准分析器对中文的处理方式是逐字分词（每个汉字单独作为词项），无法识别词语边界，搜索效果极差。

**IK Analyzer** 是最常用的中文分词插件，基于词典匹配算法。

### 安装

```bash
# 安装 IK 分词插件（版本需与ES一致）
bin/elasticsearch-plugin install \
  https://release.infinilabs.com/analysis-ik/stable/elasticsearch-analysis-ik-8.13.0.zip

# 安装完成后重启 ES
```

### ik_smart vs ik_max_word

**ik_smart（智能分词/最少切分）**

尽量使用最长的词进行匹配，减少重复词项，适合**搜索时**使用。

```
输入: "中华人民共和国成立了"

ik_smart 输出:
["中华人民共和国", "成立", "了"]

特点：
- 词语之间无重叠
- 精确度高，召回率相对低
- 适合搜索词（query analyzer）
```

**ik_max_word（最大切分/全切分）**

尽可能多地切分词语，产生更多可能的词项，适合**建立索引时**使用。

```
输入: "中华人民共和国成立了"

ik_max_word 输出:
["中华人民共和国", "中华人民", "中华", "华人", "人民共和国",
 "人民", "共和国", "共和", "国成", "成立", "了"]

特点：
- 包含所有可能的词语组合
- 召回率高，但索引体积更大
- 适合索引（index analyzer）
```

### 推荐配置方式

```json
PUT /products
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart"
      }
    }
  }
}
```

---

## 3.4 自定义分词器

```json
PUT /my_index
{
  "settings": {
    "analysis": {
      "char_filter": {
        "my_char_filter": {
          "type": "mapping",
          "mappings": [
            "& => and",
            "| => or",
            "① => 1",
            "② => 2"
          ]
        }
      },
      "tokenizer": {
        "my_tokenizer": {
          "type": "pattern",
          "pattern": "[\\s,，。！？!?;；]"
        }
      },
      "filter": {
        "my_stop_filter": {
          "type": "stop",
          "stopwords": ["的", "了", "是", "在", "和", "与"]
        },
        "my_synonym_filter": {
          "type": "synonym",
          "synonyms": [
            "手机, 移动电话, 手持电话",
            "笔记本, 笔记本电脑, laptop"
          ]
        }
      },
      "analyzer": {
        "my_custom_analyzer": {
          "type": "custom",
          "char_filter": ["my_char_filter", "html_strip"],
          "tokenizer": "ik_max_word",
          "filter": [
            "lowercase",
            "my_stop_filter",
            "my_synonym_filter"
          ]
        }
      }
    }
  }
}
```

---

## 3.5 热词更新方案（IK 远程词典）

IK 分词器支持通过 HTTP 接口实时更新词典，无需重启 ES。

### 配置远程词典

编辑 `config/analysis-ik/IKAnalyzer.cfg.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
    <comment>IK Analyzer 扩展配置</comment>
    <!-- 本地扩展词典 -->
    <entry key="ext_dict">custom/mydict.dic</entry>
    <!-- 本地停用词词典 -->
    <entry key="ext_stopwords">custom/mystopwords.dic</entry>
    <!-- 远程扩展词典（支持热更新） -->
    <entry key="remote_ext_dict">http://your-server/api/hotwords</entry>
    <!-- 远程停用词词典 -->
    <entry key="remote_ext_stopwords">http://your-server/api/stopwords</entry>
</properties>
```

### 服务端实现（Spring Boot）

```java
@RestController
@RequestMapping("/api")
public class HotWordsController {

    @Autowired
    private HotWordsService hotWordsService;

    /**
     * IK 分词器远程词典接口
     * IK 会定期 GET 此接口，通过 Last-Modified 和 ETag 判断是否更新
     */
    @GetMapping("/hotwords")
    public ResponseEntity<String> getHotWords(HttpServletResponse response) {
        List<String> words = hotWordsService.getHotWords();
        String content = String.join("\n", words);

        // 设置 Last-Modified，IK 通过此判断是否需要重新加载
        response.setHeader("Last-Modified", hotWordsService.getLastModifiedTime());
        response.setHeader("ETag", hotWordsService.getWordsHash());
        response.setContentType("text/plain;charset=UTF-8");

        return ResponseEntity.ok(content);
    }
}
```

### 热词更新流程

```
+------------------------------------------+
|           热词更新流程                     |
+------------------------------------------+
|                                          |
|  业务系统                                 |
|  新增热词 -> 更新DB                        |
|       |                                  |
|       v                                  |
|  Spring Boot API Server                  |
|  提供 /api/hotwords 接口                  |
|       |                                  |
|       | IK 每60秒轮询一次                  |
|       v                                  |
|  Elasticsearch                           |
|  IK 比较 Last-Modified/ETag              |
|  如果变化则重新加载词典                    |
|  自动生效，无需重启                        |
+------------------------------------------+
```

---

## 3.6 Analyze API 调试分词

```bash
# 测试内置分析器
GET /_analyze
{
  "analyzer": "ik_smart",
  "text": "中华人民共和国成立了"
}

# 响应示例
{
  "tokens": [
    {"token": "中华人民共和国", "start_offset": 0, "end_offset": 7, "type": "CN_WORD", "position": 0},
    {"token": "成立", "start_offset": 7, "end_offset": 9, "type": "CN_WORD", "position": 1},
    {"token": "了", "start_offset": 9, "end_offset": 10, "type": "CN_CHAR", "position": 2}
  ]
}

# 测试索引中配置的分析器
GET /my_index/_analyze
{
  "field": "title",
  "text": "测试商品名称分词效果"
}

# 测试自定义分析器组合
GET /_analyze
{
  "tokenizer": "ik_smart",
  "filter": ["lowercase", "asciifolding"],
  "text": "Elasticsearch ES搜索引擎"
}
```

---
<a name="part-4"></a>
# Part 4: 数据类型与 Mapping

## 4.1 Mapping 概述

Mapping 类似于关系型数据库中的 Schema，定义了索引中字段的名称、类型和配置。

ES 的 Mapping 有两种方式：
- **动态 Mapping**：ES 自动推断字段类型
- **显式 Mapping**：手动定义字段类型（推荐）

---

## 4.2 文本类型：text vs keyword

### text 类型（全文搜索）

```json
{
  "title": {
    "type": "text",
    "analyzer": "ik_max_word",
    "search_analyzer": "ik_smart",
    "index": true,
    "store": false,
    "term_vector": "with_positions_offsets"
  }
}
```

- **用途**：全文搜索，内容会被分词
- **无法用于**：精确匹配、聚合、排序（除非开启 fielddata，不推荐）
- **原理**：写入时经过分析器处理，存储词项而非原文

### keyword 类型（精确匹配）

```json
{
  "status": {
    "type": "keyword",
    "index": true,
    "doc_values": true,
    "ignore_above": 256
  }
}
```

- **用途**：精确匹配、聚合、排序
- **无法用于**：全文搜索
- **原理**：不经过分析器，存储完整原始值
- **ignore_above**：超过指定长度的值不被索引（防止超长字符串占用内存）

### 双字段映射（multi-fields）

很多场景需要对同一字段既做全文搜索又做精确匹配：

```json
{
  "product_name": {
    "type": "text",
    "analyzer": "ik_max_word",
    "fields": {
      "keyword": {
        "type": "keyword",
        "ignore_above": 256
      },
      "pinyin": {
        "type": "text",
        "analyzer": "pinyin_analyzer"
      }
    }
  }
}

// 使用方式：
// product_name         -> 全文搜索（text）
// product_name.keyword -> 精确匹配/聚合/排序（keyword）
// product_name.pinyin  -> 拼音搜索
```

---

## 4.3 数值类型

| 类型 | 描述 | Java 等价 | 范围 |
|------|------|----------|------|
| `byte` | 8位有符号整数 | byte | -128 ~ 127 |
| `short` | 16位有符号整数 | short | -32768 ~ 32767 |
| `integer` | 32位有符号整数 | int | -2^31 ~ 2^31-1 |
| `long` | 64位有符号整数 | long | -2^63 ~ 2^63-1 |
| `float` | 32位单精度浮点 | float | |
| `double` | 64位双精度浮点 | double | |
| `half_float` | 16位半精度浮点 | - | 精度较低，省空间 |
| `scaled_float` | 缩放浮点 | - | 用整数+比例因子存储 |

**scaled_float 的妙用：**

```json
{
  "price": {
    "type": "scaled_float",
    "scaling_factor": 100
  }
}
// 8999.99 -> 存储为 899999（整数）
// 优点：比 double 更省空间，且避免浮点精度问题
```

---

## 4.4 日期类型

```json
{
  "create_time": {
    "type": "date",
    "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
  }
}
```

ES 支持多种日期格式，并可以用 `||` 分隔多个格式。

**常用格式：**
```
yyyy-MM-dd              -> "2024-01-15"
yyyy-MM-dd HH:mm:ss     -> "2024-01-15 10:30:00"
epoch_millis            -> 1705294200000 (Unix时间戳毫秒)
epoch_second            -> 1705294200 (Unix时间戳秒)
strict_date_optional_time -> ISO 8601 格式
```

---

## 4.5 复杂类型：object vs nested

这是 ES 初学者最容易混淆的部分。

### object 类型

普通对象类型，ES 内部会将其扁平化存储：

```json
// 写入数据
{
  "author": {
    "name": "张三",
    "age": 30
  }
}

// ES 内部存储（扁平化）
{
  "author.name": "张三",
  "author.age": 30
}
```

**object 类型的陷阱（数组对象问题）：**

```json
// 写入含数组的对象
{
  "comments": [
    {"user": "张三", "content": "很好"},
    {"user": "李四", "content": "一般"}
  ]
}

// ES 扁平化后
{
  "comments.user": ["张三", "李四"],
  "comments.content": ["很好", "一般"]
}

// 问题：搜索 user=张三 AND content=一般 会错误地匹配上！
// 因为 user 和 content 的对应关系丢失了
```

### nested 类型（解决数组对象问题）

nested 类型将数组中的每个对象作为**独立的隐藏文档**存储，保留对象内部字段的关联关系。

```json
// Mapping 定义
{
  "mappings": {
    "properties": {
      "comments": {
        "type": "nested",
        "properties": {
          "user": {"type": "keyword"},
          "content": {"type": "text", "analyzer": "ik_smart"},
          "score": {"type": "integer"}
        }
      }
    }
  }
}

// 查询 nested 对象
GET /products/_search
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            {"term": {"comments.user": "张三"}},
            {"match": {"comments.content": "很好"}}
          ]
        }
      }
    }
  }
}
```

**object vs nested 对比：**

```
+----------------------------------+----------------------------------+
|          object 类型              |          nested 类型              |
+----------------------------------+----------------------------------+
| 扁平化存储                         | 独立文档存储                       |
| 字段关联关系丢失                    | 保留字段关联关系                    |
| 查询速度快                         | 查询速度稍慢                       |
| 索引大小小                         | 索引大小大（每个嵌套对象是独立文档）  |
| 适合：无数组/简单对象               | 适合：对象数组且需要精确关联查询      |
+----------------------------------+----------------------------------+
```

---

## 4.6 地理类型

### geo_point（地理坐标点）

```json
// Mapping
{
  "location": {
    "type": "geo_point"
  }
}

// 写入格式（多种方式）
// 方式1：对象
{"location": {"lat": 39.9042, "lon": 116.4074}}

// 方式2：字符串
{"location": "39.9042,116.4074"}

// 方式3：数组 [lon, lat]（注意顺序！）
{"location": [116.4074, 39.9042]}

// 方式4：GeoHash
{"location": "wx4g0"}
```

### geo_shape（地理形状）

```json
// Mapping
{
  "area": {
    "type": "geo_shape"
  }
}

// 写入多边形
{
  "area": {
    "type": "polygon",
    "coordinates": [
      [[116.3, 39.8], [116.5, 39.8], [116.5, 40.0], [116.3, 40.0], [116.3, 39.8]]
    ]
  }
}
```

---

## 4.7 动态 Mapping vs 显式 Mapping

### 动态 Mapping 的类型推断规则

| JSON 类型 | ES 推断类型 |
|----------| -----------|
| `true/false` | boolean |
| 整数 `123` | long |
| 浮点数 `1.5` | float |
| 字符串（含日期格式）| date |
| 字符串（普通）| text + keyword |
| 对象 `{}` | object |
| 数组 `[]` | 取决于第一个元素 |

### 动态 Mapping 的问题

1. **字符串默认同时创建 text 和 keyword 字段**，浪费存储空间
2. **数字可能被推断为错误类型**（如 ID 被推断为 long，但可能是字符串）
3. **无法控制分析器**
4. **字段膨胀**：日志场景可能产生大量不必要的字段

### 关闭动态 Mapping

```json
PUT /my_index
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "title": {"type": "text"},
      "price": {"type": "float"}
    }
  }
}
```

`dynamic` 取值说明：
- `true`：自动创建新字段（默认）
- `false`：忽略未知字段，不报错，但不索引
- `strict`：遇到未定义字段直接报错

---

## 4.8 Mapping 设计最佳实践

### index: false

不需要被搜索的字段，设置 `"index": false`，节省存储空间：

```json
{
  "log_raw": {
    "type": "text",
    "index": false
  }
}
```

### doc_values

`doc_values` 是列式存储，用于聚合和排序。默认对 keyword、数值、date 等类型启用。

```json
{
  "tag": {
    "type": "keyword",
    "doc_values": false
  }
}
```

### store

默认情况下，字段值存储在 `_source` 中，`store: false`。如果关闭 `_source`，可以单独设置 `store: true`。

```json
{
  "title": {
    "type": "text",
    "store": true
  }
}
```

---

## 4.9 完整 Mapping 示例（电商商品）

```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "ik_pinyin_analyzer": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "py_filter"]
        }
      },
      "filter": {
        "py_filter": {
          "type": "pinyin",
          "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true
        }
      }
    }
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "product_id": {
        "type": "keyword",
        "doc_values": true
      },
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          },
          "pinyin": {
            "type": "text",
            "analyzer": "ik_pinyin_analyzer"
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "ik_max_word",
        "index": true
      },
      "brand": {
        "type": "keyword"
      },
      "category_path": {
        "type": "keyword"
      },
      "price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "original_price": {
        "type": "scaled_float",
        "scaling_factor": 100
      },
      "stock": {
        "type": "integer"
      },
      "sales_count": {
        "type": "long"
      },
      "rating": {
        "type": "float"
      },
      "is_on_sale": {
        "type": "boolean"
      },
      "tags": {
        "type": "keyword"
      },
      "images": {
        "type": "keyword",
        "index": false,
        "doc_values": false
      },
      "attributes": {
        "type": "nested",
        "properties": {
          "key": {"type": "keyword"},
          "value": {"type": "keyword"}
        }
      },
      "location": {
        "type": "geo_point"
      },
      "create_time": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||epoch_millis"
      },
      "update_time": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||epoch_millis"
      }
    }
  }
}
```

---
<a name="part-5"></a>
# Part 5: Query DSL 深度解析

## 5.1 Query DSL 概述

ES 使用 JSON 格式的 Query DSL（Domain Specific Language）来定义查询条件。查询分为两大类：

```
查询上下文（Query Context）
  - 计算相关性评分（_score）
  - 用于 "query" 子句

过滤上下文（Filter Context）
  - 不计算评分（只做是/否判断）
  - 用于 "filter" 子句
  - 结果可以被缓存
  - 性能更好
```

---

## 5.2 全文检索查询

### match 查询

最常用的全文检索查询，会对搜索词进行分词处理。

```json
// 基本用法
GET /products/_search
{
  "query": {
    "match": {
      "title": "苹果手机"
    }
  }
}
// 搜索词会被分词为 ["苹果", "手机"]，任一词匹配即可（OR关系）

// 指定 operator 为 AND
GET /products/_search
{
  "query": {
    "match": {
      "title": {
        "query": "苹果手机",
        "operator": "and",
        "minimum_should_match": "75%"
      }
    }
  }
}
```

### match_phrase 短语查询

要求词项按顺序连续出现，适合精确短语搜索。

```json
GET /products/_search
{
  "query": {
    "match_phrase": {
      "title": {
        "query": "苹果手机",
        "slop": 0
      }
    }
  }
}
// 只匹配"苹果手机"连续出现的文档
// slop=1 允许："苹果 xxx 手机" 也能匹配
```

### match_phrase_prefix 前缀短语查询

常用于搜索建议（输入提示）：

```json
GET /products/_search
{
  "query": {
    "match_phrase_prefix": {
      "title": {
        "query": "苹果手",
        "max_expansions": 50
      }
    }
  }
}
// 匹配：苹果手机、苹果手表、苹果手环...
```

### multi_match 多字段查询

在多个字段上同时搜索：

```json
// best_fields（默认）：取最佳匹配字段的评分
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "苹果手机",
      "fields": ["title^3", "description^1", "brand^2"],
      "type": "best_fields",
      "tie_breaker": 0.3
    }
  }
}
// ^3 表示 title 字段的评分权重乘以3

// most_fields：合并所有匹配字段的评分
GET /products/_search
{
  "query": {
    "multi_match": {
      "query": "苹果手机",
      "fields": ["title", "description"],
      "type": "most_fields"
    }
  }
}

// cross_fields：将多个字段视为一个大字段
GET /users/_search
{
  "query": {
    "multi_match": {
      "query": "张 三",
      "fields": ["first_name", "last_name"],
      "type": "cross_fields",
      "operator": "and"
    }
  }
}
```

**multi_match type 对比：**

```
+------------------+----------------------------------------------------+
|      type        |       说明                                         |
+------------------+----------------------------------------------------+
| best_fields      | 最高分字段为主，tie_breaker参与其他字段分数           |
| most_fields      | 累加所有匹配字段的分数                              |
| cross_fields     | 把多字段当做一个字段来匹配，适合拆分字段（姓名）      |
| phrase           | 在每个字段上做 match_phrase                        |
| phrase_prefix    | 在每个字段上做 match_phrase_prefix                 |
+------------------+----------------------------------------------------+
```

### query_string 查询

支持 Lucene 查询语法：

```json
GET /products/_search
{
  "query": {
    "query_string": {
      "query": "(苹果 OR 华为) AND 手机 AND price:[5000 TO 10000]",
      "fields": ["title", "description"]
    }
  }
}
```

### simple_query_string 查询

更安全的查询语法版本，不会因语法错误抛出异常：

```json
GET /products/_search
{
  "query": {
    "simple_query_string": {
      "query": "苹果 + 手机 - 二手",
      "fields": ["title", "description"],
      "default_operator": "or"
    }
  }
}
// +  等同于 must（必须包含）
// -  等同于 must_not（必须不包含）
// |  等同于 should（可能包含）
```

---

## 5.3 精确查询

### term 查询

精确匹配，不分词，适用于 keyword、数值、boolean、date 类型。

```json
// 不要对 text 字段用 term 查询！
// text 字段已被分词，存储的是分词后的词项

// 正确：对 keyword 字段用 term
GET /products/_search
{
  "query": {
    "term": {
      "brand": {
        "value": "Apple",
        "boost": 1.5
      }
    }
  }
}
```

### terms 查询（IN 操作）

```json
GET /products/_search
{
  "query": {
    "terms": {
      "brand": ["Apple", "Huawei", "Samsung"]
    }
  }
}
```

### range 查询

```json
GET /products/_search
{
  "query": {
    "range": {
      "price": {
        "gte": 1000,
        "lte": 5000
      }
    }
  }
}

// 日期范围查询
GET /orders/_search
{
  "query": {
    "range": {
      "create_time": {
        "gte": "2024-01-01",
        "lte": "2024-12-31",
        "format": "yyyy-MM-dd",
        "time_zone": "+08:00"
      }
    }
  }
}

// 相对时间
GET /logs/_search
{
  "query": {
    "range": {
      "timestamp": {
        "gte": "now-7d/d",
        "lte": "now/d"
      }
    }
  }
}
```

**range 操作符说明：**
- `gt`：大于
- `gte`：大于等于
- `lt`：小于
- `lte`：小于等于

### exists 查询

```json
// 字段存在
GET /products/_search
{
  "query": {
    "exists": {"field": "description"}
  }
}

// 字段不存在（使用 must_not）
GET /products/_search
{
  "query": {
    "bool": {
      "must_not": [
        {"exists": {"field": "description"}}
      ]
    }
  }
}
```

### ids 查询

```json
GET /products/_search
{
  "query": {
    "ids": {
      "values": ["1", "2", "3", "100"]
    }
  }
}
```

---

## 5.4 复合查询

### bool 查询

bool 查询是最重要的复合查询，组合多个子查询：

```json
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "手机"}}
      ],
      "should": [
        {"term": {"brand": "Apple"}},
        {"term": {"brand": "Huawei"}}
      ],
      "must_not": [
        {"term": {"is_on_sale": false}},
        {"range": {"stock": {"lt": 1}}}
      ],
      "filter": [
        {"range": {"price": {"gte": 1000, "lte": 8000}}}
      ],
      "minimum_should_match": 1
    }
  }
}
```

**四个子句的评分影响：**

```
+--------------------------------------------------------------+
|              bool 查询各子句的评分规则                         |
+--------------------------------------------------------------+
|  must     | 必须匹配 | 参与评分（影响 _score）                  |
|  should   | 可能匹配 | 参与评分（matched should子句越多分越高）  |
|  must_not | 必须不匹配| 不参与评分（只作为过滤器）               |
|  filter   | 必须匹配 | 不参与评分（只作为过滤器，可被缓存）       |
+--------------------------------------------------------------+

性能建议：
- 只需要过滤，不关心相关性 -> 使用 filter（有缓存，速度快）
- 需要影响相关性评分 -> 使用 must/should
```

### constant_score 查询

将查询转换为过滤器，所有匹配文档获得相同的固定评分：

```json
GET /products/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {"brand": "Apple"}
      },
      "boost": 1.5
    }
  }
}
```

### function_score 查询

自定义相关性评分，常用于个性化推荐和排序：

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {
        "match": {"title": "手机"}
      },
      "functions": [
        {
          "filter": {"term": {"brand": "Apple"}},
          "weight": 2
        },
        {
          "field_value_factor": {
            "field": "sales_count",
            "factor": 0.1,
            "modifier": "log1p",
            "missing": 1
          }
        },
        {
          "gauss": {
            "create_time": {
              "origin": "now",
              "scale": "30d",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "multiply",
      "boost_mode": "multiply",
      "max_boost": 10
    }
  }
}
```

**衰减函数（Decay Functions）：**

```
gauss（高斯衰减）：
  score
    1.0  |      ****
         |    *      *
    0.5  |   *        *
         |  *          *
    0.0  +--+----------+----> 距离
            scale
  最常用，平滑衰减

exp（指数衰减）：
  score
    1.0  |*
         | *
    0.5  |  **
         |    ***
    0.0  +-------*****-----> 距离
  近处衰减快，远处衰减慢

linear（线性衰减）：
  score
    1.0  |***
         |   *
    0.5  |    *
         |     *
    0.0  +------*-----------> 距离
  均匀线性衰减
```

### dis_max 查询

取多个查询中分数最高的那个：

```json
GET /products/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {"match": {"title": "苹果手机"}},
        {"match": {"description": "苹果手机"}}
      ],
      "tie_breaker": 0.3
    }
  }
}
// 最终分数 = max(title分数, description分数) + 0.3 * 其他匹配字段分数
```

---

## 5.5 嵌套与关联查询

### nested query

查询 nested 类型字段：

```json
GET /products/_search
{
  "query": {
    "nested": {
      "path": "attributes",
      "query": {
        "bool": {
          "must": [
            {"term": {"attributes.key": "颜色"}},
            {"term": {"attributes.value": "黑色"}}
          ]
        }
      },
      "score_mode": "avg",
      "inner_hits": {
        "name": "matched_attrs",
        "size": 5
      }
    }
  }
}
```

### has_child / has_parent（父子文档）

父子文档通过 `join` 字段类型实现，父子文档存储在同一分片：

```json
// 定义父子关系
PUT /blog
{
  "mappings": {
    "properties": {
      "join_field": {
        "type": "join",
        "relations": {
          "post": "comment"
        }
      }
    }
  }
}

// 写入父文档
PUT /blog/_doc/1
{
  "title": "ES入门教程",
  "join_field": "post"
}

// 写入子文档（必须与父文档在同一分片，用 routing 保证）
PUT /blog/_doc/2?routing=1
{
  "content": "写得很好",
  "join_field": {
    "name": "comment",
    "parent": "1"
  }
}

// has_child 查询：找有特定评论的文章
GET /blog/_search
{
  "query": {
    "has_child": {
      "type": "comment",
      "query": {"match": {"content": "很好"}},
      "score_mode": "max"
    }
  }
}

// has_parent 查询：找属于特定文章的评论
GET /blog/_search
{
  "query": {
    "has_parent": {
      "parent_type": "post",
      "query": {"match": {"title": "ES入门"}},
      "score": true
    }
  }
}
```

---
<a name="part-6"></a>
# Part 6: 聚合（Aggregations）深度解析

## 6.1 聚合概述

ES 的聚合功能强大，类似 SQL 中的 GROUP BY + 聚合函数，但更为灵活，支持多层嵌套。

```
聚合类型层次：
+------------------------------------------+
|              聚合（Aggregation）            |
+------------------------------------------+
|                                          |
|  桶聚合（Bucket）    指标聚合（Metric）     |
|  - 分组/分桶         - 计算指标            |
|  - 类似GROUP BY      - 类似SUM/AVG/COUNT  |
|                                          |
|        管道聚合（Pipeline）                |
|        - 对其他聚合结果再计算              |
|        - 类似窗口函数                     |
+------------------------------------------+
```

---

## 6.2 桶聚合（Bucket Aggregations）

### terms 聚合

按字段值分组，类似 SQL 的 `GROUP BY`：

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "brand_count": {
      "terms": {
        "field": "brand",
        "size": 10,
        "order": {"_count": "desc"},
        "min_doc_count": 1,
        "missing": "未知品牌"
      }
    }
  }
}
```

**terms 聚合精度问题：** 在分布式场景下，每个分片返回 top N 个桶，协调节点再合并，可能导致结果不精确。

```json
"terms": {
  "field": "brand",
  "size": 10,
  "shard_size": 100
}
```

### range 聚合

按数值范围分桶：

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          {"to": 1000, "key": "1000以下"},
          {"from": 1000, "to": 3000, "key": "1000-3000"},
          {"from": 3000, "to": 5000, "key": "3000-5000"},
          {"from": 5000, "key": "5000以上"}
        ]
      }
    }
  }
}
```

### date_histogram 日期直方图聚合

按时间周期分桶，常用于时序数据分析：

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "sales_by_day": {
      "date_histogram": {
        "field": "create_time",
        "calendar_interval": "day",
        "format": "yyyy-MM-dd",
        "min_doc_count": 0,
        "extended_bounds": {
          "min": "2024-01-01",
          "max": "2024-12-31"
        },
        "time_zone": "+08:00"
      }
    }
  }
}
```

### histogram 数值直方图聚合

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_histogram": {
      "histogram": {
        "field": "price",
        "interval": 500,
        "min_doc_count": 0,
        "extended_bounds": {"min": 0, "max": 10000}
      }
    }
  }
}
```

### filter 聚合

对满足特定条件的文档进行聚合：

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "apple_products": {
      "filter": {"term": {"brand": "Apple"}},
      "aggs": {
        "avg_price": {"avg": {"field": "price"}}
      }
    }
  }
}
```

### nested 聚合

对 nested 类型字段进行聚合：

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "attributes_agg": {
      "nested": {"path": "attributes"},
      "aggs": {
        "attr_keys": {
          "terms": {"field": "attributes.key"},
          "aggs": {
            "attr_values": {
              "terms": {"field": "attributes.value"}
            }
          }
        }
      }
    }
  }
}
```

---

## 6.3 指标聚合（Metric Aggregations）

```json
GET /products/_search
{
  "size": 0,
  "aggs": {
    "price_min":   {"min":         {"field": "price"}},
    "price_max":   {"max":         {"field": "price"}},
    "price_avg":   {"avg":         {"field": "price"}},
    "price_sum":   {"sum":         {"field": "price"}},
    "total_count": {"value_count": {"field": "price"}},

    "price_stats": {"stats": {"field": "price"}},

    "price_extended_stats": {
      "extended_stats": {"field": "price"}
    },

    "brand_count": {
      "cardinality": {
        "field": "brand",
        "precision_threshold": 100
      }
    },

    "price_percentiles": {
      "percentiles": {
        "field": "price",
        "percents": [25, 50, 75, 90, 95, 99]
      }
    },

    "price_ranks": {
      "percentile_ranks": {
        "field": "price",
        "values": [1000, 3000, 5000]
      }
    }
  }
}
```

**各指标聚合含义：**

| 聚合名称 | 说明 | SQL 等价 |
|---------|------|---------|
| `min` | 最小值 | `MIN()` |
| `max` | 最大值 | `MAX()` |
| `avg` | 平均值 | `AVG()` |
| `sum` | 求和 | `SUM()` |
| `value_count` | 计数 | `COUNT()` |
| `stats` | 综合统计（min/max/avg/sum/count）| 多个聚合 |
| `extended_stats` | 扩展统计（含方差、标准差）| - |
| `cardinality` | 去重计数（近似值）| `COUNT(DISTINCT)` |
| `percentiles` | 百分位数 | - |
| `percentile_ranks` | 某值处于哪个百分位 | - |

---

## 6.4 管道聚合（Pipeline Aggregations）

管道聚合以其他聚合的输出作为输入，对聚合结果做二次计算。

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "sales_by_month": {
      "date_histogram": {
        "field": "create_time",
        "calendar_interval": "month"
      },
      "aggs": {
        "monthly_revenue": {"sum": {"field": "amount"}}
      }
    },

    "avg_monthly_revenue": {
      "avg_bucket": {
        "buckets_path": "sales_by_month>monthly_revenue"
      }
    },

    "moving_avg_revenue": {
      "moving_avg": {
        "buckets_path": "sales_by_month>monthly_revenue",
        "window": 3,
        "model": "simple"
      }
    },

    "cumulative_revenue": {
      "cumulative_sum": {
        "buckets_path": "sales_by_month>monthly_revenue"
      }
    },

    "revenue_derivative": {
      "derivative": {
        "buckets_path": "sales_by_month>monthly_revenue"
      }
    }
  }
}
```

---

## 6.5 嵌套聚合（多级聚合）

ES 支持无限层嵌套聚合，可以实现复杂的数据分析：

```json
GET /orders/_search
{
  "size": 0,
  "aggs": {
    "by_category": {
      "terms": {
        "field": "category",
        "size": 10
      },
      "aggs": {
        "by_brand": {
          "terms": {
            "field": "brand",
            "size": 5
          },
          "aggs": {
            "avg_price": {"avg": {"field": "price"}},
            "total_sales": {"sum": {"field": "amount"}},
            "price_percentiles": {
              "percentiles": {
                "field": "price",
                "percents": [50, 95]
              }
            }
          }
        },
        "category_total": {"sum": {"field": "amount"}}
      }
    }
  }
}
```

---

## 6.6 完整案例：电商销售统计仪表板

```json
GET /orders/_search
{
  "size": 0,
  "query": {
    "range": {
      "create_time": {
        "gte": "2024-01-01",
        "lte": "2024-12-31"
      }
    }
  },
  "aggs": {
    "total_revenue": {"sum": {"field": "amount"}},
    "total_orders": {"value_count": {"field": "order_id"}},
    "avg_order_value": {"avg": {"field": "amount"}},

    "daily_trend": {
      "date_histogram": {
        "field": "create_time",
        "calendar_interval": "day",
        "format": "yyyy-MM-dd",
        "time_zone": "+08:00"
      },
      "aggs": {
        "daily_revenue": {"sum": {"field": "amount"}},
        "daily_orders": {"value_count": {"field": "order_id"}},
        "daily_avg": {"avg": {"field": "amount"}},
        "moving_avg": {
          "moving_avg": {
            "buckets_path": "daily_revenue",
            "window": 7
          }
        }
      }
    },

    "top_categories": {
      "terms": {
        "field": "category",
        "size": 10,
        "order": {"category_revenue": "desc"}
      },
      "aggs": {
        "category_revenue": {"sum": {"field": "amount"}},
        "category_orders": {"value_count": {"field": "order_id"}}
      }
    },

    "user_segments": {
      "range": {
        "field": "amount",
        "ranges": [
          {"to": 100, "key": "小额(<100)"},
          {"from": 100, "to": 500, "key": "中额(100-500)"},
          {"from": 500, "to": 1000, "key": "大额(500-1000)"},
          {"from": 1000, "key": "高额(>1000)"}
        ]
      },
      "aggs": {
        "segment_count": {"value_count": {"field": "order_id"}},
        "segment_revenue": {"sum": {"field": "amount"}}
      }
    },

    "province_heatmap": {
      "terms": {
        "field": "province",
        "size": 34
      },
      "aggs": {
        "province_revenue": {"sum": {"field": "amount"}}
      }
    }
  }
}
```

---
<a name="part-7"></a>
# Part 7: 相关性评分原理

## 7.1 BM25 算法详解

从 ES 5.0 开始，默认评分算法从经典的 TF/IDF 换成了 **BM25（Best Match 25）**。

### 为什么用 BM25 替代 TF/IDF

**TF/IDF 的问题：**
- **TF 线性增长**：词出现次数越多，分数线性增加，没有上限
- 实际上，词出现 1000 次不比出现 20 次提供更多信息
- 对长文档不公平（长文档词频天然更高）

**BM25 的改进：**
- **TF 饱和**：词频对评分的贡献有上限，避免词堆砌欺骗
- **文档长度归一化**：通过参数 `b` 控制长度惩罚力度

### BM25 公式

```
BM25 评分公式：

Score(Q, D) = Sum_i [ IDF(qi) x TF_BM25(qi, D) ]

其中：
IDF(qi) = log(1 + (N - n(qi) + 0.5) / (n(qi) + 0.5))

TF_BM25(qi, D) = tf(qi, D) x (k1 + 1)
                 ------------------------------------------
                 tf(qi, D) + k1 x (1 - b + b x |D| / avgdl)

参数说明：
- N        : 总文档数
- n(qi)    : 包含词 qi 的文档数
- tf(qi, D): 词 qi 在文档 D 中的词频
- |D|      : 文档 D 的长度（词数）
- avgdl    : 所有文档的平均长度
- k1       : TF 饱和因子（默认1.2，范围0~3）
- b        : 长度归一化因子（默认0.75，0=不归一化，1=完全归一化）
```

### TF 饱和效果对比

```
词频对评分的贡献：

TF/IDF（线性增长，无上限）：
score
  10 |                    /
   8 |                  /
   6 |                /
   4 |              /
   2 |            /
   0 +----------+-----------> 词频
   0            50

BM25（饱和，有上限）：
score
  10 |          **************
   8 |      ***
   6 |    **
   4 |  **
   2 | *
   0 +----+------+-----------> 词频
   0   5    20

k1=1.2 时，词频从1增到20，评分约增加2.5倍
词频从20增到100，评分只增加约0.1倍
有效防止词堆砌
```

### 三大评分因素

**TF（词频）**：词在文档中出现的次数越多，相关性越高（但有饱和上限）

**IDF（逆文档频率）**：包含该词的文档越少，说明该词越能区分文档，权重越高

```
举例：
- "的"：出现在99%的文档中 -> IDF很低，权重低
- "Elasticsearch"：只出现在1%的文档中 -> IDF很高，权重高
```

**Field Length Norm（字段长度归一化）**：字段越短，词的权重越高

```
举例：
- 文档A title="手机"，文档B title="苹果手机评测推荐精选大全合集"
- 搜索"手机"时，文档A的title更短，说明"手机"更能代表文档A
- 文档A的分数会更高
```

---

## 7.2 Explain API 分析评分

```bash
GET /products/_explain/1
{
  "query": {
    "match": {"title": "苹果手机"}
  }
}

# 响应示例
{
  "_explanation": {
    "value": 4.532,
    "description": "sum of:",
    "details": [
      {
        "value": 2.156,
        "description": "weight(title:苹果 in 0) [PerFieldSimilarity], result of:",
        "details": [
          {
            "value": 2.156,
            "description": "score(freq=1.0), computed as boost * idf * tf from:",
            "details": [
              {"value": 2.2, "description": "boost"},
              {"value": 1.847, "description": "idf, computed as log(1 + (N - n + 0.5) / (n + 0.5))"},
              {"value": 0.533, "description": "tf, computed as freq / (freq + k1 * (1 - b + b * dl / avgdl))"}
            ]
          }
        ]
      }
    ]
  }
}
```

---

## 7.3 自定义相关性评分

### function_score 的高级用法

```json
GET /products/_search
{
  "query": {
    "function_score": {
      "query": {"match": {"title": "手机"}},
      "functions": [
        {
          "field_value_factor": {
            "field": "sales_count",
            "factor": 1.2,
            "modifier": "log1p",
            "missing": 1
          }
        },
        {
          "filter": {"term": {"brand": "Apple"}},
          "weight": 1.5
        },
        {
          "gauss": {
            "create_time": {
              "origin": "now",
              "scale": "30d",
              "offset": "7d",
              "decay": 0.5
            }
          }
        },
        {
          "gauss": {
            "location": {
              "origin": "39.9042,116.4074",
              "scale": "5km",
              "decay": 0.5
            }
          }
        }
      ],
      "score_mode": "sum",
      "boost_mode": "multiply",
      "min_score": 0.1
    }
  }
}
```

### script_score（脚本评分）

```json
GET /products/_search
{
  "query": {
    "script_score": {
      "query": {"match": {"title": "手机"}},
      "script": {
        "source": """
          double base = _score;
          double sales = doc['sales_count'].size() == 0 ? 0 : doc['sales_count'].value;
          double rating = doc['rating'].size() == 0 ? 3.0 : doc['rating'].value;
          // 综合评分 = 基础相关性 x ln(1+销量) x (评分/5)
          return base * Math.log1p(sales) * (rating / 5.0);
        """
      }
    }
  }
}
```

---

## 7.4 搜索结果排序

### 按字段排序

```json
GET /products/_search
{
  "query": {"match_all": {}},
  "sort": [
    {"price": {"order": "asc"}},
    {"sales_count": {"order": "desc"}},
    {"_score": {"order": "desc"}},
    {"create_time": {"order": "desc", "missing": "_last"}}
  ]
}
```

### 按地理距离排序

```json
GET /stores/_search
{
  "query": {"match_all": {}},
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 39.9042,
          "lon": 116.4074
        },
        "order": "asc",
        "unit": "km",
        "mode": "min",
        "distance_type": "arc"
      }
    }
  ]
}
```

---
<a name="part-8"></a>
# Part 8: 索引管理与优化

## 8.1 索引生命周期管理（ILM）

ILM（Index Lifecycle Management）是 ES 提供的自动化索引管理功能，特别适合日志类时序数据。

### ILM 生命周期阶段

```
索引生命周期阶段：

+--------+    +--------+    +--------+    +--------+
|  Hot   | -> |  Warm  | -> |  Cold  | -> | Delete |
+--------+    +--------+    +--------+    +--------+
   |               |               |            |
 写入活跃         只读副本减少     只读无副本    自动删除
 高性能SSD       普通磁盘         低成本冷存储  释放空间
                 force merge      冻结索引
                 段合并优化
```

### 创建 ILM 策略

```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "min_age": "0ms",
        "actions": {
          "rollover": {
            "max_age": "1d",
            "max_size": "10gb",
            "max_docs": 1000000
          },
          "set_priority": {"priority": 100}
        }
      },
      "warm": {
        "min_age": "2d",
        "actions": {
          "allocate": {
            "number_of_replicas": 1,
            "include": {"box_type": "warm"}
          },
          "forcemerge": {
            "max_num_segments": 1
          },
          "set_priority": {"priority": 50}
        }
      },
      "cold": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0,
            "include": {"box_type": "cold"}
          },
          "freeze": {},
          "set_priority": {"priority": 0}
        }
      },
      "delete": {
        "min_age": "30d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

---

## 8.2 索引模板（Index Template）

```json
// 创建组件模板（可复用）
PUT _component_template/logs_settings
{
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy",
      "index.lifecycle.rollover_alias": "logs"
    }
  }
}

PUT _component_template/logs_mappings
{
  "template": {
    "mappings": {
      "properties": {
        "@timestamp": {"type": "date"},
        "level": {"type": "keyword"},
        "service": {"type": "keyword"},
        "message": {"type": "text", "analyzer": "ik_smart"},
        "trace_id": {"type": "keyword"},
        "span_id": {"type": "keyword"}
      }
    }
  }
}

// 创建索引模板（使用组件模板）
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "composed_of": ["logs_settings", "logs_mappings"],
  "priority": 100,
  "data_stream": {}
}
```

---

## 8.3 别名（Alias）与滚动索引

别名是零停机切换索引的关键技术：

```json
// 创建索引并指定别名
PUT /products_v1
{
  "aliases": {
    "products": {
      "is_write_index": true
    }
  }
}

// 新建 products_v2 后，原子切换别名
POST /_aliases
{
  "actions": [
    {"add": {"index": "products_v2", "alias": "products", "is_write_index": true}},
    {"remove": {"index": "products_v1", "alias": "products"}}
  ]
}
// 整个操作是原子的，客户端无感知

// 过滤别名（只看到部分数据）
PUT /orders/_alias/active_orders
{
  "filter": {"term": {"status": "active"}},
  "routing": "active"
}

// Rollover：自动滚动创建新索引
POST /logs/_rollover
{
  "conditions": {
    "max_age": "7d",
    "max_docs": 1000000,
    "max_size": "5gb"
  }
}
```

---

## 8.4 Reindex 数据迁移

```json
// 基本 reindex
POST /_reindex
{
  "source": {
    "index": "products_v1",
    "query": {
      "range": {"create_time": {"gte": "2024-01-01"}}
    }
  },
  "dest": {
    "index": "products_v2",
    "version_type": "external"
  }
}

// 异步 reindex（大数据量）
POST /_reindex?wait_for_completion=false
{
  "source": {"index": "old_index"},
  "dest": {"index": "new_index"}
}
// 返回 task_id，通过 GET /_tasks/{task_id} 监控进度

// 跨集群 reindex
POST /_reindex
{
  "source": {
    "remote": {
      "host": "http://remote-cluster:9200",
      "username": "user",
      "password": "pass"
    },
    "index": "products"
  },
  "dest": {
    "index": "products_new"
  }
}
```

---

## 8.5 Force Merge（段合并）

### Segment（段）的概念

```
Lucene Index 结构（每个 ES Shard 对应一个 Lucene Index）：

+------------------------------------------+
|              Lucene Index                 |
+------------------------------------------+
|  Segment 1 (旧，已合并)  -> 不可变          |
|  Segment 2 (旧，已合并)  -> 不可变          |
|  Segment 3 (新写入)      -> 不可变          |
|  In-Memory Buffer        -> 可变，等待refresh|
+------------------------------------------+

每次 refresh（默认1秒）将内存 buffer 刷新为新 segment
随着时间推移，segment 越来越多 -> 查询需要检查更多 segment -> 性能下降
Merge 合并小 segment 为大 segment，提升查询性能
```

### Force Merge API

```bash
# 合并为最多1个段（适合只读的历史索引）
POST /old_index/_forcemerge?max_num_segments=1

# 合并并删除已删除文档（释放空间）
POST /products/_forcemerge?only_expunge_deletes=true

# 合并后刷新
POST /products/_forcemerge?flush=true
```

**注意**：Force Merge 是 CPU/IO 密集型操作，应在业务低峰期执行，且不要对活跃写入的索引执行。

---

## 8.6 索引设计原则

### 分片数规划

```
分片数量建议：

单个分片推荐大小：20-40GB（最大不超过50GB）
单个分片推荐文档数：数千万

示例：
100GB 数据 -> 建议 3-5 个主分片
1TB 数据  -> 建议 25-50 个主分片

公式（仅供参考）：
主分片数 = max(数据总大小 / 30GB, 节点数)

副本数建议：
- 生产环境至少 1 个副本（高可用）
- 如需提高读取吞吐，可设置 2-3 个副本
- 写入压力大时，可临时设置 0 个副本，写入完成后恢复

警告：主分片数创建后无法修改！
（副本数可以随时修改）
```

---
<a name="part-9"></a>
# Part 9: 集群架构与高可用

## 9.1 节点类型详解

### 节点角色（Node Roles）

```yaml
# 节点角色配置（elasticsearch.yml）
node.roles: [master, data, ingest, ml, remote_cluster_client]
```

| 节点角色 | 说明 | 适用场景 |
|---------|------|---------|
| `master` | 集群元数据管理，不处理数据 | 小集群各节点都配 |
| `master` (voting_only) | 参与选举，但不能成为主节点 | 3节点集群的第3个 |
| `data` | 存储数据，处理搜索/写入 | 大多数节点 |
| `data_hot` | 存储热数据（高性能SSD）| ILM热层 |
| `data_warm` | 存储温数据 | ILM温层 |
| `data_cold` | 存储冷数据 | ILM冷层 |
| `data_frozen` | 存储冻结数据（对象存储）| ILM冻结层 |
| `ingest` | 预处理管道（类似Logstash）| ETL处理 |
| `coordinating` | 只做路由/聚合，不存数据 | 大型集群的网关层 |
| `ml` | 机器学习任务 | ML特性 |
| `remote_cluster_client` | 跨集群搜索/复制 | CCS/CCR |

### 典型集群拓扑

```
生产集群推荐部署（7节点）：

+----------------+  +----------------+  +----------------+
| Master-eligible|  | Master-eligible|  | Master-eligible|
|     Node 1     |  |     Node 2     |  |     Node 3     |
| master+data    |  | master+data    |  | master(只参选) |
+----------------+  +----------------+  +----------------+

+------------------+  +------------------+  +------------------+
|    Data Node 4   |  |    Data Node 5   |  |    Data Node 6   |
|    data_hot      |  |    data_hot      |  |    data_warm     |
+------------------+  +------------------+  +------------------+

+---------------------------+
|   Coordinating Node 7     |
|   只做请求路由和结果汇总     |
|   面向应用层               |
+---------------------------+
```

### Master 节点职责

Master 节点负责管理集群级别的操作：
1. **索引管理**：创建/删除索引、调整 mapping 和 settings
2. **节点管理**：监控节点加入/离开
3. **分片分配**：决定分片在哪个节点上
4. **集群状态**：维护集群的状态（节点、分片分配信息等）

Master 节点不参与数据的存储和搜索，职责轻量，因此可以使用配置较低的机器。

---

## 9.2 主节点选举机制

### 7.x 之前：Zen Discovery

```
Zen Discovery 选举规则：

条件：参与选举的节点数 >= minimum_master_nodes（推荐设置为 N/2+1）

选举流程：
1. 节点发现：所有节点互相 Ping
2. 主节点失效：超时无响应的主节点被判定为宕机
3. 新一轮选举：所有 master-eligible 节点参与
4. 投票：选择 ID 最小的节点（或有更新集群状态的节点）
5. 确认：获得多数票的节点成为新主节点

问题：minimum_master_nodes 配置错误会导致"脑裂"
```

### 7.x 之后：基于 Raft 的选举

```
新集群协调层（Cluster Coordination）特点：

1. 自动计算 quorum（法定人数），无需手动配置
2. 使用 Raft 算法确保一致性
3. 引入 voting configuration（投票配置）
4. 支持 voting_only 角色（参与投票但不成为主节点）

# elasticsearch.yml（只在首次启动时使用，集群稳定后删除）
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
```

**Raft 选举简化流程：**

```
+------------------------------------------+
|          Raft 选举流程（简化）              |
+------------------------------------------+
|                                          |
|  1. 所有节点初始状态：Follower             |
|                                          |
|  2. 超时未收到 Leader 心跳                 |
|     Follower -> Candidate                |
|                                          |
|  3. Candidate 向所有节点发送投票请求        |
|     vote_request(term=1, lastLogIndex=N)  |
|                                          |
|  4. 获得多数票(N/2+1) -> 成为 Leader       |
|                                          |
|  5. Leader 周期性发送心跳维持任期           |
|                                          |
|  6. 旧 Leader 重连后发现更高 term          |
|     自动降级为 Follower                   |
+------------------------------------------+
```

---

## 9.3 脑裂（Split Brain）问题

### 什么是脑裂

```
脑裂场景：

正常状态：
+--------+     +--------+     +--------+
| Node 1 |<--->| Node 2 |<--->| Node 3 |
| Master |     | Data   |     | Data   |
+--------+     +--------+     +--------+

网络分区后：
+--------+  X分区X  +--------+     +--------+
| Node 1 |          | Node 2 |<--->| Node 3 |
| Master |          | 选举新  |     | Master |
|        |          | Master |     |        |
+--------+          +--------+     +--------+

结果：
- 两个 Master 同时存在
- 两个分区各自接受写入
- 数据不一致！这是非常严重的问题
```

### 解决方案

**旧方案（6.x及以前）：**
```yaml
# elasticsearch.yml
discovery.zen.minimum_master_nodes: 2  # N/2+1，N=3个master节点
```

**新方案（7.x+，自动处理）：**
```yaml
# 首次启动时配置（之后删除）
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]

# 使用 voting_only 角色的节点作为奇数"调解人"
# 确保 master-eligible 节点（包含voting_only）总数为奇数
```

---

## 9.4 分片分配策略

ES 提供了丰富的分片分配控制选项：

```json
// 集群级别分配控制
PUT /_cluster/settings
{
  "transient": {
    // 并发恢复分片数量
    "cluster.routing.allocation.node_concurrent_recoveries": 2,
    // 排除特定节点（缩容时使用）
    "cluster.routing.allocation.exclude._name": "node-1",
    // 磁盘阈值控制
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}

// 索引级别分配控制
PUT /my_index/_settings
{
  "index.routing.allocation.require.box_type": "hot",
  "index.routing.allocation.total_shards_per_node": 2
}
```

---

## 9.5 集群状态：Green/Yellow/Red

```
集群状态说明：

+-------+--------------------------------------------------+
| Green | 所有主分片和副本分片均已分配，集群完全正常            |
+-------+--------------------------------------------------+
|Yellow | 所有主分片已分配，但部分副本未分配（单节点时常见）     |
|       | 功能正常，但高可用性降低                           |
+-------+--------------------------------------------------+
|  Red  | 部分主分片未分配，相关索引数据不可用                 |
|       | 需要立即排查！                                    |
+-------+--------------------------------------------------+

查看集群状态：
GET /_cluster/health
GET /_cluster/health?level=indices
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason
```

**常见 Yellow/Red 原因及排查：**

```bash
# 查看未分配分片原因
GET /_cluster/allocation/explain
{
  "index": "products",
  "shard": 0,
  "primary": false
}

# 常见原因：
# 1. 单节点集群，副本无处分配（Yellow）-> 正常，设置副本数为0
# 2. 磁盘空间不足 -> 清理磁盘或扩容
# 3. 节点宕机 -> 恢复节点或重新分配
# 4. 分片损坏 -> 从备份恢复
```

---

## 9.6 集群扩缩容操作

### 扩容（增加节点）

```bash
# 新节点加入集群只需在 elasticsearch.yml 中配置相同的集群名称
cluster.name: my-cluster
discovery.seed_hosts: ["master-node-1", "master-node-2"]

# ES 会自动发现新节点并重新均衡分片
# 可以手动触发重新均衡
POST /_cluster/reroute?retry_failed=true
```

### 缩容（下线节点）

```bash
# 下线前先迁移分片
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.exclude._name": "node-to-remove"
  }
}

# 等待分片迁移完成后再关闭节点
GET /_cat/shards?v  # 确认该节点上没有分片了

# 关闭节点后清理设置
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.exclude._name": null
  }
}
```

---

## 9.7 跨集群搜索（CCS）

```json
// 在 elasticsearch.yml 配置远程集群
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_b": {
          "seeds": ["node-b1:9300", "node-b2:9300"]
        }
      }
    }
  }
}

// 跨集群搜索
GET /local_index,cluster_b:remote_index/_search
{
  "query": {"match_all": {}}
}

// 同时搜索多个远程集群
GET /cluster_a:products,cluster_b:products,products/_search
{
  "query": {"match": {"title": "手机"}}
}
```

---
<a name="part-10"></a>
# Part 10: 写入与读取原理

## 10.1 文档写入全流程

```
文档写入流程详解：

步骤1: 客户端发送写入请求到任意节点（协调节点）

         Client
           |
           | PUT /products/_doc/1
           v
    +-------------+
    | Coordinating|
    |    Node     |
    +-------------+

步骤2: 协调节点通过路由算法确定主分片
    shard = hash("1") % 3 = 0
    -> 主分片 P0 在 Node 1

步骤3: 转发请求到 Node 1 的 Primary Shard P0
           |
           v
    +-------------+
    |   Node 1    |
    | Primary P0  |
    +-------------+
    |1.写入内存Buffer|
    |2.写入translog |
    +-------------+

步骤4: Primary P0 并行转发给所有副本分片 R0
    +-------------+   +-------------+
    |   Node 3    |   |   Node 2    |
    | Replica R0  |   | Replica R1  |
    +-------------+   +-------------+

步骤5: 等待副本确认（根据wait_for_active_shards配置）

步骤6: 协调节点返回成功给客户端
```

---

## 10.2 内存 Buffer -> Translog -> Segment 流程

```
+================================================================+
|               ES 写入持久化流程详解                              |
+================================================================+
|                                                                |
|  写入请求                                                       |
|      |                                                         |
|      v                                                         |
| +----------+                                                   |
| |  Memory  |  <- 所有新写入的文档先进内存Buffer                  |
| |  Buffer  |  <- 不可搜索（索引尚未建立）                        |
| +----------+                                                   |
|      |                                                         |
|      | 同时写入（每次写入都写）                                    |
|      v                                                         |
| +----------+                                                   |
| | translog |  <- 事务日志，持久化到磁盘（fsync）                  |
| |          |  <- 崩溃恢复的依据                                 |
| +----------+                                                   |
|                                                                |
|  [每隔1秒执行 REFRESH]                                          |
|      |                                                         |
|      | Buffer -> Segment（OS Cache，未fsync）                   |
|      v                                                         |
| +----------+                                                   |
| | Segment  |  <- 可被搜索！（但尚未持久化到磁盘）                  |
| |  (新)    |  <- 如果节点崩溃，可从translog恢复                  |
| +----------+                                                   |
|                                                                |
|  [每隔30分钟或translog超过512MB 执行 FLUSH]                     |
|      |                                                         |
|      | 将所有内存中的segment fsync到磁盘                         |
|      | 清空translog（已持久化，不再需要）                          |
|      v                                                         |
| +----------+                                                   |
| | Segment  |  <- 完全持久化到磁盘                               |
| | (持久化)  |                                                   |
| +----------+                                                   |
|                                                                |
|  [后台 MERGE 进程]                                              |
|      |                                                         |
|      | 将多个小segment合并为大segment                            |
|      | 真正删除被标记删除的文档                                    |
|      v                                                         |
| +------------------+                                          |
| | Merged Segment   |  <- 合并后的大段，更少的段 = 更快的搜索      |
| +------------------+                                          |
+================================================================+

关键点：
- 文档更新/删除 -> 旧段标记删除位，写入新段（段不可变！）
- Merge 时才真正删除标记的文档
- refresh_interval 默认 1s，这就是"近实时"的由来
```

### refresh、flush 和 merge 的区别

```
+---------------+----------------------------------------+
|    操作        |                说明                    |
+---------------+----------------------------------------+
| refresh       | 将内存buffer写入新segment（OS Cache）    |
|               | 文档变为可搜索                          |
|               | 默认每1秒自动触发                       |
+---------------+----------------------------------------+
| flush         | 将OS Cache中的segment fsync到磁盘       |
|               | 清空translog                           |
|               | 默认每30分钟或translog>=512MB时触发      |
+---------------+----------------------------------------+
| merge/segment | 后台自动合并小segment为大segment         |
|               | 真正删除被标记为删除的文档               |
|               | 不可主动停止（但可force merge）          |
+---------------+----------------------------------------+
```

### 相关 API 操作

```bash
# 手动触发 refresh（立即可搜索）
POST /products/_refresh

# 临时关闭 refresh（批量写入时提高性能）
PUT /products/_settings
{"index.refresh_interval": "-1"}

# 恢复 refresh
PUT /products/_settings
{"index.refresh_interval": "1s"}

# 手动触发 flush（持久化）
POST /products/_flush

# 查看 translog 状态
GET /products/_stats/translog

# 查看 segment 信息
GET /products/_segments
GET /products/_stats/segments
```

---

## 10.3 搜索两阶段：Query Phase + Fetch Phase

```
两阶段搜索流程：

+================================================================+
|                    搜索两阶段详解                                |
+================================================================+
|                                                                |
|  Query Phase（查询阶段）：                                       |
|                                                                |
|  Client -> Coordinating Node                                   |
|                    |                                           |
|        广播给所有分片（主or副）                                   |
|        /              \              \                         |
|       v               v              v                         |
|   Shard P0         Shard P1        Shard P2                   |
|  各分片执行查询      各分片执行查询   各分片执行查询                |
|  返回: [docId, _score]  返回: [docId, _score] ...              |
|        \              /              /                         |
|         +------------+              /                          |
|          \           +--------------+                          |
|           v                                                    |
|  Coordinating Node 合并所有分片的结果                            |
|  全局排序，取 from+size 个最优结果                               |
|  （此时只有 docId 和 _score，没有文档内容）                       |
|                                                                |
|  Fetch Phase（获取阶段）：                                       |
|                                                                |
|  根据第一阶段得到的 docId 列表                                    |
|  向对应分片发送 MGET 请求                                        |
|  获取完整的文档内容（_source）                                   |
|  返回给客户端                                                   |
+================================================================+

为什么要两阶段？
- Query Phase 只传 docId+score（轻量）
- 如果每个分片直接返回完整文档，大量无用数据传输
- 两阶段极大减少了网络传输量
```

---

## 10.4 Deep Pagination 问题

### from + size 的性能陷阱

```
问题场景：from=9990, size=10（第1000页）

每个分片需要返回前 9990+10=10000 条记录
协调节点需要合并：3分片 x 10000 = 30000 条记录
然后全局排序，取最后10条

随着页数增加，内存消耗和性能急剧下降！

ES 默认限制：
index.max_result_window = 10000
超过此限制会报错
```

### scroll API（适合历史数据导出）

```json
// 第一次请求：创建 scroll 上下文
GET /products/_search?scroll=5m
{
  "query": {"match_all": {}},
  "sort": ["_doc"],
  "size": 1000
}
// 返回 scroll_id

// 后续请求：使用 scroll_id 获取下一批
GET /_search/scroll
{
  "scroll": "5m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2g..."
}

// 使用完毕后清除 scroll 上下文
DELETE /_search/scroll
{
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2g..."
}
```

### search_after（深度分页最佳实践）

```json
// 第一页
GET /products/_search
{
  "query": {"match_all": {}},
  "sort": [
    {"create_time": "desc"},
    {"product_id": "asc"}
  ],
  "size": 10
}
// 记录最后一条的 sort 值：["2024-01-15T10:30:00", "prod_123"]

// 第二页（使用上一页最后一条的 sort 值）
GET /products/_search
{
  "query": {"match_all": {}},
  "sort": [
    {"create_time": "desc"},
    {"product_id": "asc"}
  ],
  "size": 10,
  "search_after": ["2024-01-15T10:30:00", "prod_123"]
}
```

**search_after vs scroll 对比：**

```
+-------------------+--------------------+----------------------+
|      特性          |      scroll        |    search_after      |
+-------------------+--------------------+----------------------+
| 实时性            | 快照视图，不实时      | 实时数据              |
| 跳页              | 只能顺序            | 只能顺序              |
| 内存消耗          | 维护scroll上下文     | 无状态，低内存        |
| 适用场景          | 全量数据导出         | 实时深度分页          |
| 7.x 推荐          | 不推荐新场景        | 推荐使用              |
+-------------------+--------------------+----------------------+
```

---
<a name="part-11"></a>
# Part 11: Spring Boot 集成实战

## 11.1 依赖配置

### pom.xml

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
    </dependency>
</dependencies>
```

---

## 11.2 application.yml 配置

```yaml
spring:
  elasticsearch:
    uris: http://localhost:9200
    username: elastic
    password: your_password
    connection-timeout: 10s
    socket-timeout: 30s

elasticsearch:
  index:
    products: products
    orders: orders
  search:
    default-size: 10
    max-size: 100
```

---

## 11.3 实体类注解

```java
package com.example.es.entity;

import lombok.Data;
import lombok.Builder;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

/**
 * 商品 ES 文档实体
 */
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
@Document(indexName = "products", createIndex = true)
@Setting(settingPath = "es-settings/products-settings.json")
public class ProductDocument {

    @Id
    private String productId;

    /**
     * 商品标题：全文搜索 + 关键词 + 拼音
     */
    @MultiField(
        mainField = @Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart"),
        otherFields = {
            @InnerField(suffix = "keyword", type = FieldType.Keyword, ignoreAbove = 256),
            @InnerField(suffix = "pinyin", type = FieldType.Text, analyzer = "ik_pinyin_analyzer")
        }
    )
    private String title;

    @Field(type = FieldType.Text, analyzer = "ik_max_word", searchAnalyzer = "ik_smart")
    private String description;

    @Field(type = FieldType.Keyword)
    private String brand;

    @Field(type = FieldType.Keyword)
    private String categoryPath;

    @Field(type = FieldType.Scaled_Float, scalingFactor = 100)
    private BigDecimal price;

    @Field(type = FieldType.Integer)
    private Integer stock;

    @Field(type = FieldType.Long)
    private Long salesCount;

    @Field(type = FieldType.Float)
    private Float rating;

    @Field(type = FieldType.Boolean)
    private Boolean isOnSale;

    @Field(type = FieldType.Keyword)
    private List<String> tags;

    @Field(type = FieldType.Nested)
    private List<ProductAttribute> attributes;

    @GeoPointField
    private GeoPoint location;

    @Field(type = FieldType.Date, format = DateFormat.custom,
           pattern = "yyyy-MM-dd HH:mm:ss||epoch_millis")
    private LocalDateTime createTime;

    @Field(type = FieldType.Date, format = DateFormat.custom,
           pattern = "yyyy-MM-dd HH:mm:ss||epoch_millis")
    private LocalDateTime updateTime;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ProductAttribute {
        @Field(type = FieldType.Keyword)
        private String key;

        @Field(type = FieldType.Keyword)
        private String value;
    }
}
```

---

## 11.4 Repository 接口

```java
package com.example.es.repository;

import com.example.es.entity.ProductDocument;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;

import java.math.BigDecimal;
import java.util.List;

public interface ProductRepository extends ElasticsearchRepository<ProductDocument, String> {

    // 方法名推导查询
    List<ProductDocument> findByBrand(String brand);

    Page<ProductDocument> findByBrandAndIsOnSaleTrue(String brand, Pageable pageable);

    List<ProductDocument> findByPriceBetween(BigDecimal minPrice, BigDecimal maxPrice);

    List<ProductDocument> findByTitleContaining(String keyword);

    long countByBrand(String brand);

    void deleteByBrand(String brand);
}
```

---

## 11.5 ElasticsearchOperations 复杂查询

```java
package com.example.es.service;

import com.example.es.dto.ProductSearchRequest;
import com.example.es.dto.ProductSearchResponse;
import com.example.es.entity.ProductDocument;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.elasticsearch.client.elc.NativeQuery;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.SearchHit;
import org.springframework.data.elasticsearch.core.SearchHits;
import org.springframework.data.elasticsearch.core.query.HighlightQuery;
import org.springframework.data.elasticsearch.core.query.highlight.Highlight;
import org.springframework.data.elasticsearch.core.query.highlight.HighlightField;
import org.springframework.stereotype.Service;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProductSearchService {

    private final ElasticsearchOperations elasticsearchOperations;
    private final ProductRepository productRepository;

    /**
     * 商品全文搜索（支持分词+过滤+高亮+聚合）
     */
    public ProductSearchResponse search(ProductSearchRequest request) {
        // 构建查询
        NativeQuery nativeQuery = NativeQuery.builder()
            .withQuery(q -> q.bool(b -> {
                // must: 关键词全文搜索
                if (hasText(request.getKeyword())) {
                    b.must(m -> m.multiMatch(mm -> mm
                        .query(request.getKeyword())
                        .fields("title^3", "description^1", "brand^2",
                                "title.pinyin^1")
                        .minimumShouldMatch("75%")
                    ));
                }

                // filter: 不影响评分的过滤条件
                if (hasText(request.getBrand())) {
                    b.filter(f -> f.term(t -> t
                        .field("brand")
                        .value(request.getBrand())
                    ));
                }

                if (request.getMinPrice() != null || request.getMaxPrice() != null) {
                    b.filter(f -> f.range(r -> {
                        r.field("price");
                        if (request.getMinPrice() != null) {
                            r.gte(co.elastic.clients.json.JsonData.of(request.getMinPrice()));
                        }
                        if (request.getMaxPrice() != null) {
                            r.lte(co.elastic.clients.json.JsonData.of(request.getMaxPrice()));
                        }
                        return r;
                    }));
                }

                // must_not: 下架商品
                b.mustNot(mn -> mn.term(t -> t
                    .field("isOnSale")
                    .value(false)
                ));

                return b;
            }))
            // 高亮配置
            .withHighlightQuery(new HighlightQuery(
                new Highlight(List.of(
                    new HighlightField("title"),
                    new HighlightField("description")
                )),
                null
            ))
            // 分页
            .withPageable(PageRequest.of(
                request.getPage() != null ? request.getPage() : 0,
                request.getSize() != null ? request.getSize() : 10
            ))
            .build();

        SearchHits<ProductDocument> searchHits =
            elasticsearchOperations.search(nativeQuery, ProductDocument.class);

        // 处理结果
        List<ProductSearchResponse.ProductItem> items = searchHits.stream()
            .map(hit -> buildProductItem(hit))
            .collect(Collectors.toList());

        return ProductSearchResponse.builder()
            .total(searchHits.getTotalHits())
            .items(items)
            .build();
    }

    /**
     * 构建结果项（含高亮处理）
     */
    private ProductSearchResponse.ProductItem buildProductItem(
            SearchHit<ProductDocument> hit) {
        ProductDocument doc = hit.getContent();
        Map<String, List<String>> highlights = hit.getHighlightFields();

        // 高亮结果替换原文
        String title = highlights.containsKey("title")
            ? String.join("", highlights.get("title"))
            : doc.getTitle();

        String description = highlights.containsKey("description")
            ? String.join("...", highlights.get("description"))
            : null;

        return ProductSearchResponse.ProductItem.builder()
            .productId(doc.getProductId())
            .title(title)
            .description(description)
            .brand(doc.getBrand())
            .price(doc.getPrice())
            .rating(doc.getRating())
            .salesCount(doc.getSalesCount())
            .score(hit.getScore())
            .build();
    }

    private boolean hasText(String str) {
        return str != null && !str.trim().isEmpty();
    }

    /**
     * 批量写入（高性能）
     */
    public void bulkSave(List<ProductDocument> products) {
        productRepository.saveAll(products);
    }

    /**
     * 根据品牌聚合统计
     */
    public Map<String, Long> aggregateByBrand() {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q.matchAll(m -> m))
            .withAggregation("brand_agg", a -> a
                .terms(t -> t.field("brand").size(50))
            )
            .withMaxResults(0)
            .build();

        SearchHits<ProductDocument> hits =
            elasticsearchOperations.search(query, ProductDocument.class);

        // 处理聚合结果
        return hits.getAggregations() == null ? Map.of() :
            hits.getAggregations().get("brand_agg");
    }
}
```

---

## 11.6 高亮搜索实现

```java
/**
 * 高亮搜索 Controller
 */
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final ProductSearchService searchService;

    @GetMapping("/search")
    public ResponseEntity<ProductSearchResponse> search(
            @RequestParam(required = false) String keyword,
            @RequestParam(required = false) String brand,
            @RequestParam(required = false) BigDecimal minPrice,
            @RequestParam(required = false) BigDecimal maxPrice,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {

        ProductSearchRequest request = ProductSearchRequest.builder()
            .keyword(keyword)
            .brand(brand)
            .minPrice(minPrice)
            .maxPrice(maxPrice)
            .page(page)
            .size(size)
            .build();

        return ResponseEntity.ok(searchService.search(request));
    }
}
```

---

## 11.7 DTO 类定义

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductSearchRequest {
    private String keyword;
    private String brand;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private Integer page;
    private Integer size;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductSearchResponse {
    private long total;
    private List<ProductItem> items;

    @Data
    @Builder
    @NoArgsConstructor
    @AllArgsConstructor
    public static class ProductItem {
        private String productId;
        private String title;
        private String description;
        private String brand;
        private BigDecimal price;
        private Float rating;
        private Long salesCount;
        private Float score;
    }
}
```

---
<a name="part-12"></a>
# Part 12: 完整实战案例

## 案例1：商品全文搜索（IK分词+拼音搜索+高亮+聚合统计）

### 场景描述

实现一个电商商品搜索功能，支持：
- 中文分词搜索
- 拼音搜索（输入"pingguo"可搜索到"苹果"）
- 搜索结果高亮
- 品牌、价格区间聚合筛选

### Step 1：创建索引

```json
PUT /products
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "analysis": {
      "analyzer": {
        "ik_pinyin": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["py"]
        },
        "pinyin_search": {
          "type": "custom",
          "tokenizer": "keyword",
          "filter": ["py"]
        }
      },
      "filter": {
        "py": {
          "type": "pinyin",
          "keep_full_pinyin": false,
          "keep_joined_full_pinyin": true,
          "keep_original": true,
          "limit_first_letter_length": 16,
          "remove_duplicated_term": true,
          "none_chinese_pinyin_tokenize": false
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "id": {"type": "keyword"},
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "copy_to": "all",
        "fields": {
          "pinyin": {
            "type": "text",
            "analyzer": "ik_pinyin",
            "search_analyzer": "pinyin_search"
          }
        }
      },
      "brand": {"type": "keyword", "copy_to": "all"},
      "price": {"type": "scaled_float", "scaling_factor": 100},
      "sales": {"type": "long"},
      "rating": {"type": "float"},
      "all": {"type": "text", "analyzer": "ik_max_word"}
    }
  }
}
```

### Step 2：写入测试数据

```json
POST /products/_bulk
{"index": {"_id": "1"}}
{"id": "1", "title": "苹果 iPhone 15 Pro 手机", "brand": "Apple", "price": 8999, "sales": 10000, "rating": 4.8}
{"index": {"_id": "2"}}
{"id": "2", "title": "华为 Mate 60 Pro 手机", "brand": "Huawei", "price": 6999, "sales": 8000, "rating": 4.7}
{"index": {"_id": "3"}}
{"id": "3", "title": "小米 14 Pro 手机", "brand": "Xiaomi", "price": 4999, "sales": 12000, "rating": 4.6}
```

### Step 3：完整搜索查询

```json
GET /products/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "multi_match": {
            "query": "苹果手机",
            "fields": ["title^10", "title.pinyin^7", "brand^8", "all^1"],
            "type": "best_fields",
            "minimum_should_match": "1"
          }
        },
        {
          "term": {
            "brand": {
              "value": "苹果手机",
              "boost": 5
            }
          }
        }
      ],
      "filter": [
        {"range": {"price": {"gte": 0, "lte": 10000}}},
        {"range": {"rating": {"gte": 4.0}}}
      ]
    }
  },
  "highlight": {
    "pre_tags": ["<em class='highlight'>"],
    "post_tags": ["</em>"],
    "fields": {
      "title": {
        "fragment_size": 50,
        "number_of_fragments": 3
      }
    }
  },
  "aggs": {
    "brand_filter": {
      "terms": {"field": "brand", "size": 20}
    },
    "price_filter": {
      "range": {
        "field": "price",
        "ranges": [
          {"to": 2000, "key": "2000以下"},
          {"from": 2000, "to": 5000, "key": "2000-5000"},
          {"from": 5000, "to": 8000, "key": "5000-8000"},
          {"from": 8000, "key": "8000以上"}
        ]
      }
    }
  },
  "sort": [
    {"_score": {"order": "desc"}},
    {"sales": {"order": "desc"}}
  ],
  "from": 0,
  "size": 10
}
```

---

## 案例2：日志中心（多字段搜索+时间范围+聚合分析）

### 场景描述

构建应用日志分析平台，类似 ELK Stack，支持：
- 多字段关键词搜索（message、service、host）
- 时间范围过滤
- 日志级别统计
- 错误趋势分析

### 日志 Mapping

```json
PUT /app-logs-2024
{
  "mappings": {
    "properties": {
      "@timestamp": {"type": "date"},
      "level": {"type": "keyword"},
      "service": {"type": "keyword"},
      "host": {"type": "keyword"},
      "message": {
        "type": "text",
        "analyzer": "ik_smart"
      },
      "trace_id": {"type": "keyword"},
      "span_id": {"type": "keyword"},
      "duration_ms": {"type": "long"},
      "status_code": {"type": "integer"},
      "user_id": {"type": "keyword"},
      "extra": {"type": "object", "dynamic": true}
    }
  }
}
```

### 日志搜索查询

```json
GET /app-logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "数据库连接超时",
            "fields": ["message^3", "service^2"],
            "type": "best_fields"
          }
        }
      ],
      "filter": [
        {"term": {"level": "ERROR"}},
        {
          "range": {
            "@timestamp": {
              "gte": "now-1h",
              "lte": "now"
            }
          }
        }
      ]
    }
  },
  "aggs": {
    "error_timeline": {
      "date_histogram": {
        "field": "@timestamp",
        "fixed_interval": "5m",
        "format": "yyyy-MM-dd HH:mm"
      },
      "aggs": {
        "level_breakdown": {
          "terms": {"field": "level"}
        }
      }
    },
    "top_services": {
      "terms": {
        "field": "service",
        "size": 10,
        "order": {"_count": "desc"}
      }
    },
    "avg_duration": {
      "avg": {"field": "duration_ms"}
    },
    "slow_requests": {
      "filter": {
        "range": {"duration_ms": {"gt": 1000}}
      },
      "aggs": {
        "count": {"value_count": {"field": "trace_id"}}
      }
    }
  },
  "sort": [{"@timestamp": {"order": "desc"}}],
  "size": 50
}
```

---

## 案例3：附近的人/门店（geo_distance查询）

### 场景描述

实现"附近5公里内的门店"功能，支持：
- 按距离过滤
- 按距离排序
- 返回距离信息

### 门店 Mapping

```json
PUT /stores
{
  "mappings": {
    "properties": {
      "store_id": {"type": "keyword"},
      "name": {
        "type": "text",
        "analyzer": "ik_smart"
      },
      "category": {"type": "keyword"},
      "location": {"type": "geo_point"},
      "address": {"type": "text"},
      "rating": {"type": "float"},
      "open_time": {"type": "keyword"},
      "tags": {"type": "keyword"},
      "is_open": {"type": "boolean"}
    }
  }
}
```

### 写入门店数据

```json
POST /stores/_doc/1
{
  "store_id": "S001",
  "name": "星巴克 国贸店",
  "category": "咖啡",
  "location": {"lat": 39.9087, "lon": 116.4615},
  "address": "北京市朝阳区国贸商城B1",
  "rating": 4.5,
  "open_time": "08:00-22:00",
  "tags": ["咖啡", "甜品", "WiFi"],
  "is_open": true
}
```

### 附近门店查询

```json
GET /stores/_search
{
  "query": {
    "bool": {
      "must": [
        {"term": {"is_open": true}}
      ],
      "filter": [
        {
          "geo_distance": {
            "distance": "5km",
            "location": {
              "lat": 39.9042,
              "lon": 116.4074
            }
          }
        }
      ],
      "should": [
        {"term": {"category": "咖啡"}}
      ]
    }
  },
  "sort": [
    {
      "_geo_distance": {
        "location": {
          "lat": 39.9042,
          "lon": 116.4074
        },
        "order": "asc",
        "unit": "km",
        "distance_type": "arc"
      }
    },
    {"rating": {"order": "desc"}}
  ],
  "script_fields": {
    "distance_km": {
      "script": {
        "source": "doc['location'].arcDistance(params.lat, params.lon) / 1000",
        "params": {"lat": 39.9042, "lon": 116.4074}
      }
    }
  },
  "size": 20
}
```

### Java 代码实现

```java
@Service
@RequiredArgsConstructor
public class NearbyStoreService {

    private final ElasticsearchOperations elasticsearchOperations;

    public List<StoreVO> findNearbyStores(double lat, double lon,
                                          double distanceKm, String category) {
        NativeQuery query = NativeQuery.builder()
            .withQuery(q -> q.bool(b -> {
                // 地理距离过滤
                b.filter(f -> f.geoDistance(g -> g
                    .field("location")
                    .distance(distanceKm + "km")
                    .location(l -> l.latlon(ll -> ll.lat(lat).lon(lon)))
                ));

                // 营业状态
                b.filter(f -> f.term(t -> t.field("is_open").value(true)));

                // 分类过滤
                if (category != null && !category.isEmpty()) {
                    b.filter(f -> f.term(t -> t.field("category").value(category)));
                }

                return b;
            }))
            .withSort(s -> s.geoDistance(g -> g
                .field("location")
                .location(l -> l.latlon(ll -> ll.lat(lat).lon(lon)))
                .order(co.elastic.clients.elasticsearch._types.SortOrder.Asc)
                .unit(co.elastic.clients.elasticsearch._types.DistanceUnit.Kilometers)
            ))
            .withMaxResults(20)
            .build();

        SearchHits<StoreDocument> hits =
            elasticsearchOperations.search(query, StoreDocument.class);

        return hits.stream()
            .map(hit -> buildStoreVO(hit, lat, lon))
            .collect(Collectors.toList());
    }

    private StoreVO buildStoreVO(SearchHit<StoreDocument> hit,
                                  double lat, double lon) {
        StoreDocument doc = hit.getContent();
        // 计算距离
        double distance = calculateDistance(
            lat, lon,
            doc.getLocation().getLat(),
            doc.getLocation().getLon()
        );

        return StoreVO.builder()
            .storeId(doc.getStoreId())
            .name(doc.getName())
            .category(doc.getCategory())
            .distance(Math.round(distance * 100) / 100.0)
            .rating(doc.getRating())
            .address(doc.getAddress())
            .build();
    }
}
```

---

## 案例4：自动补全（Completion Suggester）

### 场景描述

实现搜索框输入时的实时补全功能，如输入"苹果"自动提示"苹果手机、苹果平板、苹果电脑"。

### 配置 completion 字段

```json
PUT /suggest_index
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word"
      },
      "suggest": {
        "type": "completion",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "preserve_separators": false,
        "preserve_position_increments": true,
        "max_input_length": 50
      }
    }
  }
}

// 写入文档时同时写入 suggest 字段
POST /suggest_index/_doc
{
  "title": "苹果 iPhone 15 Pro 手机",
  "suggest": {
    "input": ["苹果手机", "iPhone手机", "苹果iPhone", "apple phone"],
    "weight": 100
  }
}
```

### 自动补全查询

```json
GET /suggest_index/_search
{
  "suggest": {
    "product_suggest": {
      "prefix": "苹果",
      "completion": {
        "field": "suggest",
        "skip_duplicates": true,
        "size": 10
      }
    }
  }
}
```

### Java 实现

```java
@Service
public class AutoCompleteService {

    @Autowired
    private ElasticsearchOperations operations;

    public List<String> suggest(String prefix) {
        SuggestBuilder suggestBuilder = new SuggestBuilder()
            .addSuggestion("product_suggest",
                SuggestBuilders.completionSuggestion("suggest")
                    .prefix(prefix)
                    .skipDuplicates(true)
                    .size(10)
            );

        // 使用原生查询执行 suggest
        NativeQuery query = NativeQuery.builder()
            .withSuggester(s -> s
                .suggesters("product_suggest", ss -> ss
                    .prefix(prefix)
                    .completion(c -> c
                        .field("suggest")
                        .skipDuplicates(true)
                        .size(10)
                    )
                )
            )
            .build();

        SearchHits<SuggestDocument> hits =
            operations.search(query, SuggestDocument.class);

        // 提取建议结果
        return hits.getSuggest() == null ? List.of() :
            hits.getSuggest().getSuggestion("product_suggest")
                .getEntries().stream()
                .flatMap(e -> e.getOptions().stream())
                .map(o -> o.getText())
                .collect(Collectors.toList());
    }
}
```

---

## 案例5：同义词搜索（synonym token filter）

### 场景描述

搜索"手机"时，同时能搜到"移动电话"、"mobile phone"的相关文档。

### 配置同义词分析器

```json
PUT /products_synonym
{
  "settings": {
    "analysis": {
      "filter": {
        "my_synonym": {
          "type": "synonym",
          "synonyms": [
            "手机, 移动电话, 大哥大, mobile phone, smartphone",
            "笔记本, 笔记本电脑, laptop, notebook",
            "耳机, 耳塞, headphones, earphone",
            "平板, 平板电脑, tablet, ipad"
          ]
        },
        "my_synonym_path": {
          "type": "synonym",
          "synonyms_path": "analysis/synonyms.txt",
          "updateable": true
        }
      },
      "analyzer": {
        "ik_synonym": {
          "type": "custom",
          "tokenizer": "ik_max_word",
          "filter": ["lowercase", "my_synonym"]
        },
        "ik_synonym_search": {
          "type": "custom",
          "tokenizer": "ik_smart",
          "filter": ["lowercase", "my_synonym"]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_synonym",
        "search_analyzer": "ik_synonym_search"
      }
    }
  }
}
```

**注意**：同义词建议只在搜索时（search_analyzer）使用，索引时不扩展，否则索引体积会大幅增加。

---
<a name="part-13"></a>
# Part 13: 性能调优

## 13.1 写入性能调优

### 使用 Bulk API

单条写入和批量写入的性能差距巨大：

```java
// 错误做法：循环单条写入（性能极差）
for (Product product : products) {
    productRepository.save(product);  // 每次都是 HTTP 请求+确认
}

// 正确做法：批量写入
productRepository.saveAll(products);  // 一次 HTTP 请求，多条文档

// 或使用 BulkProcessor
BulkProcessor bulkProcessor = BulkProcessor.builder(
    (request, bulkListener) -> client.bulk(request, bulkListener),
    new BulkProcessor.Listener() {
        @Override
        public void beforeBulk(long executionId, BulkRequest request) {
            log.info("Executing bulk request: {} actions", request.numberOfActions());
        }

        @Override
        public void afterBulk(long executionId, BulkRequest request, BulkResponse response) {
            if (response.hasFailures()) {
                log.error("Bulk execution had failures: {}", response.buildFailureMessage());
            }
        }

        @Override
        public void afterBulk(long executionId, BulkRequest request, Throwable failure) {
            log.error("Bulk execution failed", failure);
        }
    })
    .setBulkActions(1000)          // 达到1000条时提交
    .setBulkSize(new ByteSizeValue(5, ByteSizeUnit.MB))  // 达到5MB时提交
    .setFlushInterval(TimeValue.timeValueSeconds(10))    // 最多10秒提交一次
    .setConcurrentRequests(2)      // 并发请求数
    .build();
```

### refresh_interval 优化

```json
// 批量写入前，临时关闭 refresh（不需要近实时可搜索）
PUT /products/_settings
{
  "index.refresh_interval": "-1"
}

// 批量写入完成后，恢复 refresh
PUT /products/_settings
{
  "index.refresh_interval": "1s"
}

// 首次建库时，批量写入推荐设置
PUT /products/_settings
{
  "index.refresh_interval": "30s",
  "index.number_of_replicas": 0
}
// 写入完成后恢复
PUT /products/_settings
{
  "index.refresh_interval": "1s",
  "index.number_of_replicas": 1
}
```

### 写入调优参数汇总

```yaml
# elasticsearch.yml 调优
# 增加索引缓冲区
indices.memory.index_buffer_size: 20%

# 增加 translog 大小（降低 flush 频率）
index.translog.flush_threshold_size: 1024mb

# 异步 translog（牺牲部分可靠性换取性能）
index.translog.durability: async
index.translog.sync_interval: 30s
```

---

## 13.2 查询性能调优

### Filter 缓存

过滤器查询（filter context）的结果会被缓存：

```json
// 好的写法：过滤条件放 filter 子句（有缓存）
{
  "query": {
    "bool": {
      "must": {"match": {"title": "手机"}},
      "filter": [
        {"term": {"brand": "Apple"}},
        {"range": {"price": {"gte": 1000}}}
      ]
    }
  }
}

// 差的写法：过滤条件放 must 子句（无缓存，每次重新计算）
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "手机"}},
        {"term": {"brand": "Apple"}},
        {"range": {"price": {"gte": 1000}}}
      ]
    }
  }
}
```

### Doc Values

`doc_values` 是列式存储，聚合和排序操作使用 doc_values 而非 fielddata，性能更好。

```json
// 确保需要聚合/排序的字段开启 doc_values（默认开启）
{
  "price": {
    "type": "float",
    "doc_values": true
  }
}

// 不需要聚合/排序的文本字段，关闭 fielddata
{
  "title": {
    "type": "text",
    "fielddata": false  // 默认就是 false
  }
}
```

### Routing 优化

通过 routing，将相关文档写入同一分片，查询时只需访问一个分片：

```json
// 写入时指定 routing
PUT /orders/_doc/1?routing=user_001
{
  "user_id": "user_001",
  "amount": 100
}

// 查询时也指定 routing
GET /orders/_search?routing=user_001
{
  "query": {"term": {"user_id": "user_001"}}
}
// 只搜索 routing=user_001 对应的分片，而不是所有分片
```

### 分页优化

```java
// 不要使用大 from 值
// from=10000 会导致所有分片各返回 10010 条记录

// 推荐：使用 search_after
SearchRequest request = SearchRequest.of(s -> s
    .index("products")
    .query(q -> q.matchAll(m -> m))
    .sort(so -> so.field(f -> f.field("create_time").order(SortOrder.Desc)))
    .sort(so -> so.field(f -> f.field("product_id").order(SortOrder.Asc)))
    .searchAfter(lastSortValues)  // 上一页最后一条的排序值
    .size(10)
);
```

---

## 13.3 内存调优

### JVM 堆内存设置

```bash
# jvm.options 配置
# 堆内存设置为物理内存的 50%，最大不超过 32GB
-Xms16g
-Xmx16g
# 注意：-Xms 和 -Xmx 必须相同，避免堆调整带来的性能波动
```

**为什么不超过 32GB？**

```
JVM 在堆内存 < 32GB 时使用 Compressed OOPs（压缩对象指针）
  -> 指针从 64 bit 压缩为 32 bit
  -> 同样的内存可以存储更多对象

堆内存 > 32GB：
  -> 必须使用 64 bit 指针
  -> 内存效率急剧下降
  -> 实际上还不如使用 30GB

建议：单个 ES 节点堆内存设置为 26-30GB 最合适
```

### Field Data Cache

text 字段的聚合需要加载 fielddata 到内存（非常耗内存）：

```yaml
# elasticsearch.yml
# 限制 fielddata cache 大小
indices.fielddata.cache.size: 20%
# 或者
indices.fielddata.cache.size: 8gb

# 监控 fielddata 使用
GET /_cat/fielddata?v
GET /_nodes/stats/indices/fielddata?fields=*
```

### Circuit Breaker（熔断器）

防止内存溢出的保护机制：

```yaml
# elasticsearch.yml
# 父熔断器（总内存限制）
indices.breaker.total.use_real_memory: true
indices.breaker.total.limit: 95%

# Fielddata 熔断器
indices.breaker.fielddata.limit: 40%

# Request 熔断器（单次请求内存限制）
indices.breaker.request.limit: 60%
```

---

## 13.4 JVM 参数配置

```bash
# jvm.options 完整推荐配置

# 堆内存（物理内存50%，不超过32GB）
-Xms16g
-Xmx16g

# 使用 G1 GC（ES 7.x+ 默认）
-XX:+UseG1GC
-XX:G1ReservePercent=25
-XX:InitiatingHeapOccupancyPercent=30

# GC 日志
-Xlog:gc*:logs/gc.log:time,uptime:filecount=32,filesize=64m

# 堆转储
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=logs/

# 关闭 swap（必须！否则严重影响性能）
# 在 OS 级别：swapoff -a
# ES 配置：
# bootstrap.memory_lock: true

# 禁用 JVM DNS 缓存（云环境IP可能变化）
-Des.networkaddress.cache.ttl=60
-Des.networkaddress.cache.negative.ttl=10
```

---

## 13.5 慢查询日志配置

```json
PUT /my_index/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.query.debug": "2s",
  "index.search.slowlog.threshold.query.trace": "500ms",

  "index.search.slowlog.threshold.fetch.warn": "1s",
  "index.search.slowlog.threshold.fetch.info": "800ms",

  "index.indexing.slowlog.threshold.index.warn": "10s",
  "index.indexing.slowlog.threshold.index.info": "5s",
  "index.indexing.slowlog.threshold.index.debug": "2s",
  "index.indexing.slowlog.threshold.index.trace": "500ms",

  "index.indexing.slowlog.source": "1000"
}
```

---

## 13.6 常见性能问题排查

### 查询慢

```bash
# 1. 查看慢查询日志
tail -f /var/log/elasticsearch/slow-query.log

# 2. 使用 Profile API 分析查询耗时
GET /products/_search
{
  "profile": true,
  "query": {
    "match": {"title": "手机"}
  }
}

# 3. 检查分片状态
GET /_cat/shards?v&h=index,shard,state,docs,store,node
GET /_nodes/hot_threads

# 4. 检查是否有大量 fielddata
GET /_cat/fielddata?v&s=size:desc
```

### 写入慢

```bash
# 1. 检查 JVM GC
GET /_nodes/stats/jvm
# 关注 gc.collectors.young.collection_time_in_millis

# 2. 检查 segment 数量（太多 segment 影响性能）
GET /products/_segments
GET /_cat/segments?v&h=index,shard,segment,docs.count,size

# 3. 检查 translog 大小
GET /products/_stats/translog

# 4. 检查磁盘 I/O
GET /_nodes/stats/indices/store,translog
```

### 内存不足

```bash
# 1. 检查 heap 使用率
GET /_nodes/stats/jvm
# heap.percent > 80% 需要关注

# 2. 检查 fielddata
GET /_nodes/stats/indices/fielddata
# field_data.memory_size_in_bytes 过大时，清除缓存
POST /products/_cache/clear?fielddata=true

# 3. 检查熔断器状态
GET /_nodes/stats/breaker

# 4. 检查 segment memory
GET /_nodes/stats/indices/segments
# segments.memory_in_bytes 过大时，执行 force merge
```

### 集群状态异常

```bash
# 1. 检查集群健康
GET /_cluster/health?pretty

# 2. 检查未分配分片
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason

# 3. 获取分片分配说明
GET /_cluster/allocation/explain

# 4. 强制重新分配
POST /_cluster/reroute?retry_failed=true

# 5. 检查磁盘空间
GET /_cat/allocation?v&s=disk.percent:desc
```

---
<a name="part-14"></a>
# Part 14: 常见面试题 FAQ

## Q1: Elasticsearch 是如何实现近实时搜索的？

**答：**

ES 的近实时（NRT）搜索通过 **refresh** 机制实现：

1. 文档写入时先进入**内存 Buffer**（此时不可搜索）
2. 每隔 1 秒（`refresh_interval`），内存 Buffer 被写入新的 **Segment**（存在 OS Cache 中）
3. Segment 一旦创建，文档即可被搜索到
4. 此过程称为 refresh，比 fsync 到磁盘快得多，因此延迟约 1 秒

**要点：**
- refresh 只是写入 OS Cache，并未持久化
- 持久化靠 **flush**（fsync 到磁盘），崩溃恢复靠 **translog**
- 可通过 `POST /index/_refresh` 手动触发，立即可搜索

---

## Q2: Elasticsearch 倒排索引的原理是什么？

**答：**

倒排索引是"词项 -> 文档列表"的映射，与正排索引（文档 -> 词项）相反。

**组成部分：**
1. **Term Dictionary（词项字典）**：存储所有词项，使用 FST 压缩
2. **Posting List（倒排列表）**：每个词项对应的文档 ID 列表，使用 FOR 压缩
3. **Term Index（词项索引）**：FST 结构，常驻内存，快速定位 Term Dictionary

**查询流程：**
```
搜索"苹果" -> 在 Term Index(内存) 中定位 -> 找到 Term Dictionary 中"苹果"的位置
-> 读取对应 Posting List -> 返回 [Doc1, Doc3]
```

**优势：** O(1) 查询词项，避免全表扫描

---

## Q3: ES 中 text 和 keyword 的区别是什么？

**答：**

| 特性 | text | keyword |
|------|------|---------|
| 索引方式 | 分词后索引 | 整体索引，不分词 |
| 用途 | 全文搜索 | 精确匹配、聚合、排序 |
| 能否聚合 | 不能（除非开fielddata）| 能 |
| 能否排序 | 不能 | 能 |
| 适用场景 | 文章内容、商品描述 | 状态码、分类、ID |

**常见最佳实践：**
```json
{
  "title": {
    "type": "text",
    "analyzer": "ik_max_word",
    "fields": {
      "keyword": {"type": "keyword", "ignore_above": 256}
    }
  }
}
```
- `title` 用于全文搜索
- `title.keyword` 用于精确匹配和聚合

---

## Q4: ES 中 filter 和 query 的区别？

**答：**

| 特性 | query（查询上下文）| filter（过滤上下文）|
|------|-----------------|-----------------|
| 相关性评分 | 计算 _score | 不计算 _score |
| 缓存 | 不缓存 | 可以缓存 |
| 性能 | 较慢 | 较快 |
| 适用场景 | 全文搜索，需要排序 | 精确条件过滤 |

**建议：**
- 不需要相关性评分的条件（状态、品牌、价格范围）放 `filter`
- 需要相关性评分的全文搜索放 `must`/`should`

---

## Q5: 什么是脑裂？如何避免？

**答：**

脑裂（Split Brain）是指在网络分区时，集群被分成两个子集，每个子集各自选举出一个 Master 节点，导致两个 Master 同时存在，数据不一致。

**旧方案（7.x 以前）：**
```yaml
discovery.zen.minimum_master_nodes: 2  # 设为 N/2+1
```

**新方案（7.x+）：**
- ES 7.x 引入基于 Raft 算法的新选举机制
- 自动计算 quorum，无需手动配置 `minimum_master_nodes`
- 首次启动时配置 `cluster.initial_master_nodes`
- 保持 master-eligible 节点为奇数（3或5个）

---

## Q6: ES 主分片数量能修改吗？为什么？

**答：**

**不能修改**。主分片数量在索引创建时确定后，**无法更改**。

**原因：** 分片路由算法：
```
shard = hash(document_id) % number_of_primary_shards
```

如果修改了主分片数，同一文档的路由结果会变化，导致找不到已有文档。

**解决方案：** 使用 `reindex` API 将数据迁移到具有不同分片数的新索引，然后通过 alias 切换。

**副本数可以随时修改：**
```json
PUT /products/_settings
{"index.number_of_replicas": 2}
```

---

## Q7: ES 如何保证文档写入不丢失？

**答：**

通过 **translog（事务日志）** 保证：

1. 文档写入内存 Buffer 的**同时**，写入 translog（fsync 到磁盘）
2. 即使节点崩溃，重启时从 translog 恢复尚未 flush 到磁盘的数据
3. flush 完成后，translog 被清空

**默认配置：** `index.translog.durability: request` —— 每次写操作都 fsync translog

**高性能配置（牺牲部分可靠性）：**
```json
{
  "index.translog.durability": "async",
  "index.translog.sync_interval": "30s"
}
```
最坏情况下丢失 30 秒的数据。

---

## Q8: 如何提高 ES 的写入性能？

**答：**

1. **使用 Bulk API** —— 批量写入，减少网络 RTT
2. **临时关闭 refresh** —— `refresh_interval: -1`，批量写完后恢复
3. **临时减少副本数** —— 写入时设为 0，完成后恢复
4. **增大 indexing buffer** —— `indices.memory.index_buffer_size: 20%`
5. **使用 SSD** —— 磁盘 I/O 是写入瓶颈
6. **增加 translog 大小** —— 减少 flush 频率
7. **禁用 swapping** —— `bootstrap.memory_lock: true`

---

## Q9: ES 如何实现分布式搜索？

**答：**

ES 搜索分为**两阶段**：

**Query Phase（查询阶段）：**
1. 协调节点接收请求
2. 广播查询给所有相关分片（主分片或副本）
3. 每个分片独立执行查询，返回 `[docId, score]` 列表
4. 协调节点合并所有分片结果，全局排序，取 `from+size` 条

**Fetch Phase（获取阶段）：**
1. 根据 Query Phase 筛选出的 docId
2. 向对应分片发送 MGET 请求
3. 获取完整 _source 内容
4. 返回客户端

---

## Q10: 什么是 Doc Values？与 fielddata 有什么区别？

**答：**

两者都是为了支持聚合和排序，但存储方式不同：

| 特性 | Doc Values | Fielddata |
|------|-----------|---------|
| 存储位置 | 磁盘（列式存储）| 内存 |
| 适用类型 | keyword、数值、date | text |
| 内存消耗 | 低（OS Page Cache）| 高（JVM Heap）|
| 是否默认 | 默认开启 | 默认关闭 |
| 构建时机 | 索引时构建 | 查询时构建 |

**建议：** 使用 keyword 类型代替 text 的 fielddata，或对 text 字段增加 keyword sub-field。

---

## Q11: ES 集群状态 Yellow 是什么意思？如何排查？

**答：**

Yellow 状态表示：所有**主分片**已分配，但部分**副本分片**未分配。

**常见原因：**
1. **单节点集群** —— 副本无法分配在同一节点（ES 默认不允许主副同节点）
   - 解决：`PUT /index/_settings {"index.number_of_replicas": 0}`
2. **节点数不足** —— 需要至少 2 个节点才能分配副本
3. **磁盘空间不足** —— 节点磁盘使用超过 85%（watermark）
4. **节点宕机** —— 副本还在恢复中

**排查命令：**
```bash
GET /_cluster/health
GET /_cluster/allocation/explain
GET /_cat/shards?v&h=index,shard,prirep,state,unassigned.reason
```

---

## Q12: 如何进行 ES 数据迁移？

**答：**

**方案一：reindex API（同集群或跨集群）**
```json
POST /_reindex?wait_for_completion=false
{
  "source": {"index": "old_index"},
  "dest": {"index": "new_index"}
}
```

**方案二：使用别名（零停机迁移）**
1. 创建新索引（新 mapping 和分片数）
2. 执行 reindex 将数据迁移到新索引
3. 原子切换别名指向新索引

**方案三：Logstash/Elasticsearch-Dump**

**方案四：快照恢复（Snapshot & Restore）**
```bash
PUT /_snapshot/my_backup/snapshot_1
# 注册仓库 -> 创建快照 -> 恢复快照
```

---

## Q13: ES 的 nested 类型和普通 object 类型有什么区别？

**答：**

**object 类型：** 扁平化存储，数组对象的字段关联关系会丢失
```
{"comments": [{"user":"A","score":5}, {"user":"B","score":1}]}
-> 扁平化后：{"comments.user":["A","B"], "comments.score":[5,1]}
-> 查 user=A AND score=1 会误匹配（关联关系丢失）
```

**nested 类型：** 每个数组元素作为独立隐藏文档存储，保留字段关联关系

**代价：**
- 存储空间更大（每个 nested 对象是独立文档）
- 查询更慢（需要 nested query）
- 更新父文档时需要重索引所有 nested 文档

**适用场景：** 需要对对象数组中的多字段联合查询时使用 nested。

---

## Q14: BM25 相比 TF-IDF 有什么优势？

**答：**

**TF-IDF 的问题：**
- TF（词频）线性增长，没有上限，容易被词堆砌欺骗
- 对长文档不公平（长文档词频天然更高）

**BM25 的改进：**

1. **TF 饱和（Saturation）：**
   - 词频贡献有上限，k1 参数控制饱和速度（默认1.2）
   - 词出现100次不比出现20次贡献高太多

2. **文档长度归一化：**
   - b 参数控制长度惩罚力度（默认0.75）
   - 短文档中出现的词权重更高

**公式：**
```
TF_BM25 = tf * (k1+1) / (tf + k1*(1 - b + b*|D|/avgdl))
```

---

## Q15: 如何优化 ES 的深度分页查询？

**答：**

**from + size 的问题：**
- `from=10000, size=10` 时，每个分片需要返回 10010 条记录
- 协调节点需要处理 shard_count * 10010 条记录
- 随分页深度增加，性能急剧下降

**解决方案：**

1. **search_after（推荐）：**
   - 使用上一页最后一条的排序值作为起点
   - 无状态，性能不随页数下降
   - 适合实时翻页

2. **scroll（适合数据导出）：**
   - 创建快照，不反映写入后的变化
   - 维护 scroll_id，有内存开销
   - ES 8.x 推荐用 PIT（Point In Time）替代 scroll

3. **限制最大页数：**
   - 产品层面限制（超过第100页提示重新搜索）

---

## Q16: ES 如何实现高亮显示？

**答：**

ES 支持三种高亮方式：

1. **Plain Highlighter（默认）：**
   - 简单但耗内存
   - 需要重新分析文档

2. **Postings Highlighter：**
   - 需要 `term_vector: "with_positions_offsets"`
   - 性能好，适合大文档

3. **Fast Vector Highlighter（FVH）：**
   - 需要 `term_vector: "with_positions_offsets_payloads"`
   - 最快，支持多字段高亮

```json
GET /products/_search
{
  "query": {"match": {"title": "苹果手机"}},
  "highlight": {
    "pre_tags": ["<em>"],
    "post_tags": ["</em>"],
    "fields": {
      "title": {
        "type": "unified",
        "fragment_size": 100,
        "number_of_fragments": 3
      }
    }
  }
}
```

---

## Q17: 什么情况下应该使用 ES，什么情况下不应该？

**答：**

**适合使用 ES：**
- 全文搜索（电商、内容检索）
- 日志分析（ELK Stack）
- 实时数据分析（聚合统计）
- 地理空间查询（附近门店）
- 数据量大（亿级）、查询模式多样

**不适合使用 ES：**
- 强事务要求（金融交易）
- 频繁更新的主数据（每次更新都要重索引）
- 需要复杂 JOIN（OLTP 业务）
- 数据量小（< 10万条，MySQL 够用）
- 强一致性要求（ES 是最终一致性）

**最佳实践：** MySQL 做主存储（CRUD），ES 做搜索引擎（查询），通过 Canal/Logstash/Flink 同步数据。

---

## Q18: 如何监控 ES 集群的健康状况？

**答：**

**常用监控 API：**

```bash
# 集群健康状态
GET /_cluster/health?pretty

# 节点状态
GET /_nodes/stats?pretty

# 各索引统计
GET /_cat/indices?v&h=index,docs.count,store.size,pri,rep&s=store.size:desc

# 分片分布
GET /_cat/shards?v

# 热点线程（排查性能问题）
GET /_nodes/hot_threads

# 任务列表
GET /_tasks?pretty

# 内存/JVM 状态
GET /_nodes/stats/jvm?pretty
```

**监控工具：**
- **Kibana Stack Monitoring** —— 官方监控面板
- **Elastic APM** —— 应用性能监控
- **Prometheus + Grafana** —— 使用 elasticsearch_exporter
- **自定义报警** —— 监控 heap.percent > 80%、disk.percent > 85%

---

## Q19: ES 的 ILM（索引生命周期管理）主要解决什么问题？

**答：**

ILM 主要解决**时序数据（如日志）的存储成本和性能问题**：

**问题：**
- 日志数据量大，无限增长
- 近期数据访问频繁，需要高性能存储
- 历史数据访问少，但存储成本高

**ILM 解决方案：**
```
Hot（热）  -> Warm（温）  -> Cold（冷）  -> Delete（删除）
  |              |               |              |
高性能SSD      普通磁盘        低成本存储     自动删除
全分片副本    减少副本数       冻结索引      释放空间
活跃写入      force merge      只读
```

**配置关键点：**
- rollover：控制单个索引的大小（防止单索引过大）
- 转移时机：按 min_age 触发
- 配合 index template 自动应用策略

---

## Q20: 如何实现 ES 数据与 MySQL 的同步？

**答：**

常见同步方案：

**方案一：应用双写（最简单，不推荐用于生产）**
```java
// 写 MySQL
userRepository.save(user);
// 同时写 ES
elasticsearchOperations.save(userDoc);
// 问题：两个操作无原子性，可能数据不一致
```

**方案二：Canal 监听 MySQL binlog（推荐）**
```
MySQL -> Canal -> Kafka/RabbitMQ -> Consumer -> ES
优点：解耦、可靠、支持回放
缺点：有延迟（通常 < 1s）
```

**方案三：Logstash JDBC 插件（定时同步）**
```yaml
input {
  jdbc {
    jdbc_driver_library => "/path/to/mysql-connector.jar"
    jdbc_driver_class => "com.mysql.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/mydb"
    jdbc_user => "root"
    jdbc_password => "password"
    schedule => "*/1 * * * *"
    statement => "SELECT * FROM products WHERE update_time > :sql_last_value"
    use_column_value => true
    tracking_column => "update_time"
  }
}
output {
  elasticsearch {
    hosts => ["localhost:9200"]
    index => "products"
    document_id => "%{id}"
  }
}
```

**方案四：Flink CDC（实时流处理）**
```
MySQL CDC Source -> Flink 转换处理 -> ES Sink
优点：实时、支持复杂 ETL、高可靠
缺点：需要 Flink 集群
```

---

## Q21: Elasticsearch 的 doc_values 是什么？为什么聚合比 fielddata 快？

**答：**

**doc_values** 是一种**列式存储**结构，在索引时构建，存储在磁盘上（被 OS Page Cache 缓存）。

**为什么聚合需要列式存储？**

行式存储（正排索引）：
```
Doc1: {name="张三", age=25, city="北京"}
Doc2: {name="李四", age=30, city="上海"}
Doc3: {name="王五", age=25, city="北京"}
```
聚合 age 字段时，需要读取每行所有字段，效率低。

列式存储（doc_values）：
```
age: [25, 30, 25, ...]
     Doc1  Doc2  Doc3
```
聚合 age 字段时，只读 age 列，IO 减少，可 SIMD 优化。

**fielddata vs doc_values：**
- fielddata 在 JVM heap 中动态构建，会造成 GC 压力
- doc_values 在 OS Page Cache 中，通过 mmap 访问，不占 JVM heap

---

## Q22: 说说 ES 的写入和查询的一致性保证

**答：**

**写入一致性：**

ES 默认 `wait_for_active_shards=1`，即只需要主分片确认写入成功就返回。

```json
PUT /products/_doc/1?wait_for_active_shards=all
{
  ...
}
// all 表示等待所有副本确认，更强一致性但延迟更高
```

**读取一致性：**

ES 是**最终一致性**：
- 主分片写入成功后，副本异步同步
- 在同步完成前，从副本读取的数据可能是旧的
- 默认搜索时，主副分片都可能被选中，存在不一致窗口

**如何获得更强的一致性：**
```json
// 写入后立即 refresh，确保可搜索
PUT /products/_doc/1?refresh=wait_for
{
  ...
}
// wait_for: 等待下次 refresh 完成（默认1秒内）
// true: 立即强制 refresh（影响性能，慎用）
```

---

## Q23: Elasticsearch 是如何处理文档更新和删除的？

**答：**

**关键点：Lucene Segment 不可变！**

**更新流程：**
1. 查找旧版本文档所在的 segment
2. 在该 segment 的**删除位图**中标记旧文档为已删除
3. 在内存 Buffer 中写入新版本文档（新 segment）
4. 搜索时自动忽略被标记删除的文档

**删除流程：**
1. 在 segment 的删除位图中标记文档为已删除
2. 文档不会立即从磁盘上删除
3. 直到 **segment merge** 时，才真正删除被标记的文档

**影响：**
- 大量更新/删除后，磁盘占用不会立即减少
- 可以通过 `force merge` 强制合并 segment，释放磁盘空间
- `_deleted_docs` 计数可以通过 `GET /index/_stats` 查看

---

## Q24: 如何设计 ES 的分片策略？

**答：**

**经验法则：**

```
单分片大小：20-50GB（不超过50GB）
单分片文档数：几千万

分片数 = max(预估数据量 / 30GB, 节点数)
副本数 = 1（高可用）或 2（高读吞吐）
```

**过少分片的问题：**
- 单分片数据量过大，查询慢
- 无法充分利用多节点并行

**过多分片的问题：**
- 每个分片是独立的 Lucene 索引，有元数据开销
- 大量小分片导致集群状态过大，master 压力增加
- ES 建议每个节点最多 20-25 个分片/GB 堆内存

**重要提醒：** 主分片数创建后不可修改，规划时要考虑未来增长，预留 20-30% 余量。

---

## Q25: 说说你用 ES 踩过的坑

**答（常见坑）：**

1. **主分片数规划不足** —— 后期数据量大了却无法增加分片，只能 reindex
2. **text 字段乱用 term 查询** —— text 已分词，term 精确匹配永远查不到
3. **object 数组陷阱** —— 用了 object 而非 nested，查询结果出现误匹配
4. **动态 mapping 字段爆炸** —— 日志场景没有限制 dynamic，产生数万个字段
5. **deep pagination** —— 用 from+size 做深度分页，查询第100页时崩溃
6. **忘记关闭 fielddata** —— text 字段聚合开了 fielddata，OOM
7. **reindex 忘记更新别名** —— 迁移后应用还在读写旧索引
8. **副本数设为 0 忘了改回来** —— 批量导入后忘记设回副本，集群 Yellow
9. **堆内存超过 32GB** —— 失去 Compressed OOPs，内存效率反而下降
10. **没有监控 translog**  —— translog 过大导致节点启动慢

---

# 附录：常用 API 速查

## 集群操作

```bash
# 集群健康
GET /_cluster/health?pretty

# 节点列表
GET /_cat/nodes?v

# 分片分布
GET /_cat/shards?v&s=index,shard

# 索引列表
GET /_cat/indices?v&s=store.size:desc

# 任务管理
GET /_tasks
POST /_tasks/{taskId}/_cancel

# 集群设置
GET /_cluster/settings
PUT /_cluster/settings
{
  "transient": {"cluster.routing.allocation.enable": "all"}
}
```

## 索引操作

```bash
# 创建索引
PUT /my_index
{
  "settings": {"number_of_shards": 3, "number_of_replicas": 1},
  "mappings": {...}
}

# 查看 mapping
GET /my_index/_mapping

# 修改 settings
PUT /my_index/_settings
{"index.refresh_interval": "5s"}

# 关闭/开启索引
POST /my_index/_close
POST /my_index/_open

# 刷新
POST /my_index/_refresh

# 强制合并
POST /my_index/_forcemerge?max_num_segments=1

# 删除索引
DELETE /my_index
```

## 文档操作

```bash
# 索引/更新文档
PUT /my_index/_doc/1
{"field": "value"}

# 获取文档
GET /my_index/_doc/1

# 部分更新
POST /my_index/_update/1
{"doc": {"field": "new_value"}}

# 脚本更新
POST /my_index/_update/1
{
  "script": {
    "source": "ctx._source.count += params.count",
    "params": {"count": 4}
  }
}

# 删除文档
DELETE /my_index/_doc/1

# 按查询更新
POST /my_index/_update_by_query
{
  "query": {"term": {"status": "draft"}},
  "script": {"source": "ctx._source.status = 'published'"}
}

# 按查询删除
POST /my_index/_delete_by_query
{
  "query": {"range": {"create_time": {"lt": "2020-01-01"}}}
}

# 批量操作
POST /_bulk
{"index": {"_index": "my_index", "_id": "1"}}
{"field": "value1"}
{"delete": {"_index": "my_index", "_id": "2"}}
```

---

> **文档结束**
>
> 本文档覆盖了 Elasticsearch 的核心原理、实践操作、Spring Boot 集成和性能调优。
> 建议结合官方文档 https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html 深入学习。
>
> 版本：基于 Elasticsearch 8.x，更新日期：2026年7月
