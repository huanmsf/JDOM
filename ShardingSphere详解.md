# ShardingSphere 详解 - 从零到精通

> 版本：Apache ShardingSphere 5.x  
> 适合人群：有 Java/Spring Boot 基础，想深入掌握分库分表的开发者  
> 目标：从理论到实战，彻底搞懂 ShardingSphere

---

## 目录

- [Part 1: 分库分表背景与理论](#part-1)
- [Part 2: ShardingSphere 整体架构](#part-2)
- [Part 3: 分片策略深度解析](#part-3)
- [Part 4: 数据节点配置](#part-4)
- [Part 5: 读写分离](#part-5)
- [Part 6: 分布式主键生成](#part-6)
- [Part 7: 分布式事务](#part-7)
- [Part 8: 数据加密](#part-8)
- [Part 9: 影子库与全链路压测](#part-9)
- [Part 10: Spring Boot 集成实战](#part-10)
- [Part 11: 常见问题与解决方案](#part-11)
- [Part 12: 运维与监控](#part-12)
- [Part 13: 常见面试题 FAQ](#part-13)

---

---

# Part 1: 分库分表背景与理论 {#part-1}

## 1.1 为什么需要分库分表

### 单库单表的瓶颈

随着业务增长，传统单体数据库架构面临严峻挑战。以一个典型电商平台为例：

```
早期阶段（用户 < 100万）：
  单台 MySQL 服务器
  订单表：1000万条记录
  响应时间：< 10ms
  完全够用 ✓

成长阶段（用户 > 1000万）：
  订单表：5亿条记录
  单表 B+ 树索引深度增加
  全表扫描越来越慢
  开始出现性能问题 ✗

爆发阶段（用户 > 5000万）：
  订单表：50亿条记录
  磁盘 I/O 成为瓶颈
  连接池被耗尽
  系统频繁超时 ✗✗✗
```

### 数据量瓶颈分析

**MySQL 单表性能拐点：**

```
记录数              索引层高      查询性能
-----------        ----------    --------
< 100万             2层           极快 (< 1ms)
100万 ~ 500万       3层           快   (1~5ms)
500万 ~ 2000万      3层           一般 (5~20ms)
2000万 ~ 1亿        4层           慢   (20~100ms)
> 1亿               4层+          很慢 (> 100ms, 甚至秒级)
```

**B+ 树层高计算公式：**

```
假设：
  每页大小 = 16KB = 16384 字节
  主键 bigint = 8 字节
  页指针 = 6 字节
  每条记录平均 1KB

非叶子节点每页可存：16384 / (8+6) ≈ 1170 个指针
叶子节点每页可存：  16384 / 1024  ≈ 16   条记录

树高为 3 时：1170 × 1170 × 16 ≈ 2190 万条记录
树高为 4 时：1170^3 × 16   ≈ 256 亿条记录

实际上，超过 2000 万后性能下降原因：
  ① 页分裂导致磁盘碎片化，随机 I/O 增多
  ② 锁竞争加剧（行锁、间隙锁）
  ③ undo log / redo log 膨胀
  ④ buffer pool 无法缓存所有热数据
```

### 并发量瓶颈

```
单台 MySQL 并发天花板：

  最大连接数：默认 151，建议不超过 1000
  QPS 上限：读约 2~5 万/s，写约 1~2 万/s

  高并发场景（如秒杀活动）：
    峰值请求：50000 QPS
    单库处理：20000 QPS
    差距：    30000 QPS → 大量超时/拒绝

资源类型    单库上限           瓶颈现象
--------    --------           --------
CPU         8~32核             复杂查询CPU飙升100%
内存        64~256GB           buffer pool不够，频繁磁盘IO
磁盘IOPS    SSD约10万IOPS      写入延迟上升
网络带宽    1~10Gbps           大数据量查询网络阻塞
连接数      < 1000             Too many connections 报错
```

---

## 1.2 垂直拆分 vs 水平拆分

### 垂直分库（Vertical Database Sharding）

按业务模块将不同的表拆分到不同的数据库：

```
拆分前（单体数据库）：
+------------------------------------------+
|            monolith_db                   |
|  +----------+  +----------+  +--------+  |
|  |  user    |  |  order   |  | goods  |  |
|  | id       |  | id       |  | id     |  |
|  | name     |  | user_id  |  | name   |  |
|  | email    |  | amount   |  | price  |  |
|  | address  |  | status   |  | stock  |  |
|  +----------+  +----------+  +--------+  |
+------------------------------------------+

拆分后（垂直分库）：
+----------------+  +----------------+  +----------------+
|   user_db      |  |   order_db     |  |   goods_db     |
|  +----------+  |  |  +----------+  |  |  +----------+  |
|  |  user    |  |  |  |  order   |  |  |  |  goods   |  |
|  | id       |  |  |  | id       |  |  |  | id       |  |
|  | name     |  |  |  | user_id  |  |  |  | name     |  |
|  | email    |  |  |  | amount   |  |  |  | price    |  |
|  | address  |  |  |  | status   |  |  |  | stock    |  |
|  +----------+  |  |  +----------+  |  |  +----------+  |
+----------------+  +----------------+  +----------------+
```

**垂直分库优缺点：**
```
优点：
  ✓ 业务解耦，微服务化的基础
  ✓ 每个数据库压力降低
  ✓ 不同业务可以用不同数据库配置

缺点：
  ✗ 跨库 JOIN 变得困难
  ✗ 分布式事务复杂度上升
  ✗ 跨库聚合查询困难
```

### 垂直分表（Vertical Table Sharding）

将宽表按列的冷热程度拆分：

```
拆分前（宽表，所有列在一张表）：
+---------------------------------------------------------------+
|                         user                                  |
| id | name | email | avatar(大字段,200KB) | intro(大字段,10KB) |
+---------------------------------------------------------------+
  频繁访问的热字段 ↑           很少访问的冷字段 ↑

问题：
  加载任何用户信息都要读取整行
  avatar 字段很大，导致每页存储的行数极少
  buffer pool 命中率低

拆分后（垂直分表）：
+------------------------+    +------------------------------+
|      user_base         |    |        user_extra            |
| id | name | email      |    | user_id | avatar  | intro   |
+------------------------+    +------------------------------+
  （小行，高频访问）               （大字段，按需JOIN）

优势：
  user_base 每页可存更多行 → buffer pool 利用率翻倍
  查看用户列表只读 user_base，不加载大字段
  需要头像时才 JOIN user_extra
```

### 水平分库（Horizontal Database Sharding）

```
同一张表的数据按规则分散到多个数据库：

拆分前：
+---------------------------+
|         order_db          |
|  +---------------------+  |
|  |      t_order        |  |
|  | id=1  user_id=1001  |  |
|  | id=2  user_id=1002  |  |
|  | id=3  user_id=1003  |  |
|  | id=4  user_id=1004  |  |
|  | ...   ...           |  |
|  +---------------------+  |
+---------------------------+

拆分后（按 user_id % 2 水平分库）：
+---------------------------+    +---------------------------+
|       order_db_0          |    |       order_db_1          |
|  +---------------------+  |    |  +---------------------+  |
|  |      t_order        |  |    |  |      t_order        |  |
|  | id=2  user_id=1002  |  |    |  | id=1  user_id=1001  |  |
|  | id=4  user_id=1004  |  |    |  | id=3  user_id=1003  |  |
|  | ...   user_id偶数   |  |    |  | ...   user_id奇数   |  |
|  +---------------------+  |    |  +---------------------+  |
+---------------------------+    +---------------------------+
```

### 水平分表（Horizontal Table Sharding）

```
同库内，同一张表按规则分散到多个物理表：

+----------------------------------------------+
|                  order_db                    |
|  +------------+  +------------+              |
|  | t_order_0  |  | t_order_1  |              |
|  | id%4==0    |  | id%4==1    |              |
|  | id=4,8,12  |  | id=1,5,9   |              |
|  +------------+  +------------+              |
|  +------------+  +------------+              |
|  | t_order_2  |  | t_order_3  |              |
|  | id%4==2    |  | id%4==3    |              |
|  | id=2,6,10  |  | id=3,7,11  |              |
|  +------------+  +------------+              |
+----------------------------------------------+
```

### 四种拆分方式对比

```
+----------+-------------+-------------+-----------------------------+
| 拆分方式  | 解决问题     | 带来问题     | 适用场景                     |
+----------+-------------+-------------+-----------------------------+
| 垂直分库  | 业务解耦     | 跨库JOIN     | 业务明确解耦，微服务化        |
|          | 连接数瓶颈   | 分布式事务   |                              |
+----------+-------------+-------------+-----------------------------+
| 垂直分表  | 大行IO性能   | JOIN增加     | 有大字段的宽表，冷热明显      |
|          | 热点IO       |              |                              |
+----------+-------------+-------------+-----------------------------+
| 水平分库  | 并发写入     | 跨库查询     | 并发极高，需多实例            |
|          | 磁盘容量     | 全局ID生成   |                              |
+----------+-------------+-------------+-----------------------------+
| 水平分表  | 单表行数多   | 分页排序复杂 | 单库容量够，但单表行数过多    |
|          | 索引过大     | 聚合查询     |                              |
+----------+-------------+-------------+-----------------------------+
```

---

## 1.3 分库分表带来的核心挑战

### 挑战一：跨库 JOIN

```
问题场景：查询用户的订单详情（含商品名称）

SELECT o.id, o.amount, g.name AS goods_name
FROM order_db_0.t_order o
JOIN goods_db.t_goods g      -- 商品在不同数据库实例！
ON o.goods_id = g.id
WHERE o.user_id = 1001;

-- MySQL 不支持跨数据库实例 JOIN！

解决方案：
  方案1：应用层关联（推荐）
    Step1: SELECT * FROM t_order WHERE user_id = 1001
    Step2: 提取 goods_id 列表 → [101, 102, 103]
    Step3: SELECT * FROM t_goods WHERE id IN (101, 102, 103)
    Step4: 应用层拼装结果

  方案2：广播表（适合字典/配置类小表）
    将 t_goods_category 等小表复制到每个分片库

  方案3：冗余字段（反范式设计）
    在 t_order 中冗余 goods_name 字段

  方案4：使用 Elasticsearch
    将需要复杂查询的数据同步到 ES，用 ES 做联合查询
```

### 挑战二：分布式事务

```
问题场景：下单需要同时：
  1. order_db_0：INSERT 订单记录
  2. goods_db：UPDATE 库存（stock - 1）
  3. user_db：UPDATE 用户余额（balance - amount）

三步操作分布在不同数据库实例！

传统 @Transactional 失效：
  Spring 事务只能管理单个数据源
  跨数据源时无法保证原子性

解决方案：
  XA 强一致（Atomikos/Narayana）：性能较低
  Seata AT 最终一致：性能好，适合大多数场景
  消息最终一致：异步，复杂，最高性能
```

### 挑战三：分布式分页排序

```
问题：
  SELECT * FROM t_order ORDER BY create_time DESC LIMIT 100000, 10;
  数据分布在 t_order_0 ~ t_order_3 四张表

ShardingSphere 处理方式：
  改写为对每个分片执行：
    SELECT * FROM t_order_x ORDER BY create_time DESC LIMIT 0, 100010

  每个分片返回 100010 条！
  归并节点对 4 × 100010 = 400040 条记录排序
  取第 100001~100010 条

  结论：
    页码越大，内存消耗越大，性能越差！
    这是分库分表最大的痛点之一。

推荐替代方案：
  ① 游标分页（记录上次最后一条ID）
     SELECT * FROM t_order WHERE user_id=1001
     AND id > last_id ORDER BY id LIMIT 10;

  ② 禁止深分页，限制最大页码 < 100页

  ③ 历史数据归档到 ES，MySQL 只保留近期数据
```

### 挑战四：全局唯一 ID

```
问题：数据库自增 ID 在分库后不全局唯一

  order_db_0.t_order 中有 id=1
  order_db_1.t_order 中也有 id=1
  → 合并查询时数据混乱！

改良（自增步长）的缺陷：
  ds_0 起始值1，步长2 → 1, 3, 5, 7...
  ds_1 起始值2，步长2 → 2, 4, 6, 8...
  扩容时步长和起始值都要改，代价极高

推荐方案：雪花算法（Snowflake）
  基于时间+机器ID+序列号生成全局唯一 64位 ID
  单机每秒可生成 400万+ 个 ID
  生成的 ID 单调递增（趋势有序）
```

---

## 1.4 ShardingSphere 产品线全景

```
+------------------------------------------------------------------+
|                    Apache ShardingSphere                         |
|                                                                  |
|  +-------------------+  +------------------+  +--------------+  |
|  | ShardingSphere-   |  | ShardingSphere-  |  |ShardingSphere|  |
|  |      JDBC         |  |     Proxy        |  |  -Sidecar    |  |
|  |                   |  |                  |  |  (规划中)    |  |
|  | Java客户端内嵌     |  | 独立部署代理服务  |  | K8s Sidecar  |  |
|  | 零额外部署         |  | 语言无关         |  | 模式         |  |
|  | 低延迟             |  | 透明接入         |  |              |  |
|  | Spring Boot友好    |  | 统一运维入口     |  |              |  |
|  +-------------------+  +------------------+  +--------------+  |
|                                                                  |
|           核心功能：                                              |
|  分库分表  读写分离  分布式事务  数据加密  影子库  高可用          |
+------------------------------------------------------------------+
```

### ShardingSphere-JDBC 架构

```
应用程序（Java）
      |
      | new ShardingSphereDataSource(config)
      ↓
+-------------------+
| ShardingSphere-   |
|     JDBC          |   ← 实现了 javax.sql.DataSource 接口
| （增强DataSource） |   ← 对应用完全透明
+-------------------+
      |
      |----→ MySQL 实例1 (ds_0)  3306
      |----→ MySQL 实例2 (ds_1)  3307
      |----→ MySQL 实例3 (ds_2)  3308
      +----→ MySQL 实例4 (ds_3)  3309

特点：
  ✓ 纯 Java，无额外组件
  ✓ 无网络开销（进程内）
  ✓ Spring Boot Starter 集成简单
  ✓ 适合 Java 微服务
  ✗ 每个服务都要配置
  ✗ 不支持非 Java 应用
```

### ShardingSphere-Proxy 架构

```
Python应用  Java应用   Go应用   任意语言
    |           |         |         |
    +-----------+---------+---------+
                    |
                    | MySQL 协议（5.7/8.0）
                    | 应用无感知，就像连接普通MySQL
                    ↓
          +------------------+
          | ShardingSphere-  |
          |     Proxy        |  监听 3307 端口
          | （模拟MySQL服务器）|
          +------------------+
                    |
          +---------+---------+
          |                   |
          ↓                   ↓
    MySQL实例1           MySQL实例2
    (3306)               (3306)

特点：
  ✓ 多语言支持（Python/Go/PHP...）
  ✓ 应用零改造（改连接字符串即可）
  ✓ 统一运维，可用 MySQL 客户端工具操作
  ✓ 支持 DistSQL 动态改规则
  ✗ 多一跳网络，略增加延迟（通常 < 1ms）
  ✗ 需要独立部署运维
```

---

---

# Part 2: ShardingSphere 整体架构 {#part-2}

## 2.1 核心架构图

```
+------------------------------------------------------------------+
|                    ShardingSphere-JDBC 架构                       |
+------------------------------------------------------------------+
|                                                                  |
|   应用程序 (Application)                                          |
|       |                                                          |
|       | JDBC API (标准接口，对应用透明)                            |
|       ↓                                                          |
|   +------------------------------------------------------------+ |
|   |              ShardingSphere JDBC Core                      | |
|   |                                                            | |
|   |  +-----------+  +----------+  +------------------------+  | |
|   |  |  配置管理  |  | 规则引擎  |  |     元数据管理器        |  | |
|   |  | ConfigMgr |  |RuleEngine|  |   MetaDataManager      |  | |
|   |  +-----------+  +----------+  +------------------------+  | |
|   |                                                            | |
|   |  +------------------------------------------------------+  | |
|   |  |                  SQL 执行引擎                         |  | |
|   |  |                                                      |  | |
|   |  |  +---------+  +--------+  +--------+  +---------+   |  | |
|   |  |  | 解析引擎 |  | 路由引擎 |  | 改写引擎 |  | 执行引擎 |  |  | |
|   |  |  | Parser  |  | Router |  |Rewriter|  |Executor |   |  | |
|   |  |  +---------+  +--------+  +--------+  +---------+   |  | |
|   |  |                                                      |  | |
|   |  |  +---------+                                         |  | |
|   |  |  | 归并引擎 |                                         |  | |
|   |  |  | Merger  |                                         |  | |
|   |  |  +---------+                                         |  | |
|   |  +------------------------------------------------------+  | |
|   +------------------------------------------------------------+ |
|       |              |              |              |            |
|       ↓              ↓              ↓              ↓            |
|   ds_0(MySQL)   ds_1(MySQL)   ds_2(MySQL)   ds_3(MySQL)        |
+------------------------------------------------------------------+
```

---

## 2.2 五大核心引擎详解

### 引擎1：SQL 解析引擎（Parse Engine）

将 SQL 字符串解析为抽象语法树（AST）。

```
输入 SQL：
  SELECT * FROM t_order WHERE user_id = 1001 AND order_id > 2000

词法分析（Lexer）：
  [SELECT] [*] [FROM] [t_order] [WHERE]
  [user_id] [=] [1001] [AND] [order_id] [>] [2000]

语法分析（Parser）→ AST：
  SelectStatement
  ├── ProjectionsSegment: [*]
  ├── FromSegment
  │   └── SimpleTableSegment: t_order
  └── WhereSegment
      └── AndExpression
          ├── BinaryOperationExpression
          │   ├── ColumnSegment: user_id
          │   ├── Operator: =
          │   └── LiteralExpressionSegment: 1001
          └── BinaryOperationExpression
              ├── ColumnSegment: order_id
              ├── Operator: >
              └── LiteralExpressionSegment: 2000

支持数据库方言：
  MySQL / PostgreSQL / Oracle / SQL Server / openGauss
```

**ShardingSphere SQL 解析器特点：**
```
解析方案演进：
  v1-v3：使用 Druid SQL Parser（功能强，但与业务耦合）
  v4+  ：自研 SQL Parser，基于 ANTLR4 语法定义
         → 更标准，更好维护，支持更多方言

解析缓存（Parse Cache）：
  相同 SQL 模板的解析结果会被缓存
  参数化 SQL（PreparedStatement）可以复用缓存
  性能大幅提升

示例（缓存命中）：
  第一次：SELECT * FROM t_order WHERE user_id = ?  → 解析 + 缓存
  第二次：SELECT * FROM t_order WHERE user_id = ?  → 直接从缓存取 AST
```

---

### 引擎2：路由引擎（Route Engine）

根据分片规则和 SQL 中的条件，决定 SQL 应该发往哪些分片。

```
路由决策树：

SQL 进入路由引擎
       |
       ├──[包含 Hint 强制路由]──→ Hint 路由
       |
       ├──[广播表]──────────────→ 广播路由（所有分片）
       |
       ├──[单播（unicast）]──────→ 单播路由（任意一个分片）
       |     用于不需要路由的 DDL/特殊查询
       |
       └──[分片表]
             |
             ├──[含分片键，精确值]──→ 标准路由（精确1~N个分片）
             |    = 或 IN 操作
             |
             ├──[含分片键，范围值]──→ 范围路由（连续多个分片）
             |    BETWEEN、>、<
             |
             ├──[多表关联]
             |    ├──[绑定表]──────→ 绑定表路由（避免笛卡尔积）
             |    └──[非绑定表]────→ 笛卡尔积路由（慎用！）
             |
             └──[无分片键]──────────→ 全路由（所有分片）
                   性能最差！监控告警

路由结果示例：
  user_id = 1001 → ds_1（1001 % 4 = 1）
  order_id = 5678 → t_order_2（5678 % 4 = 2）
  
  最终路由目标：ds_1.t_order_2
```

---

### 引擎3：改写引擎（Rewrite Engine）

将面向逻辑表的 SQL 改写为面向物理表的 SQL。

```
改写类型1：表名改写
  原始：SELECT * FROM t_order WHERE user_id = 1001
  改写：SELECT * FROM t_order_2 WHERE user_id = 1001
        （路由到 ds_1.t_order_2）

改写类型2：分页改写（LIMIT 改写）
  原始：SELECT * FROM t_order ORDER BY id LIMIT 100, 10
  问题：第100条数据在哪个分片不确定

  改写后（对每个分片）：
    SELECT * FROM t_order_0 ORDER BY id LIMIT 0, 110
    SELECT * FROM t_order_1 ORDER BY id LIMIT 0, 110
    SELECT * FROM t_order_2 ORDER BY id LIMIT 0, 110
    SELECT * FROM t_order_3 ORDER BY id LIMIT 0, 110

  解释：每个分片取前 110 条（offset + count = 100 + 10）
       归并引擎从 4×110 条中取第 101~110 条
       
  代价：页码越大，每个分片取的数据越多！

改写类型3：分组改写（AVG 改写）
  原始：SELECT AVG(amount) FROM t_order
  
  直接 AVG 是错的！（不同分片数量不同）
  
  改写：SELECT COUNT(amount) AS cnt, SUM(amount) AS sum FROM t_order_x
  归并：SUM(all_sum) / SUM(all_cnt)

改写类型4：加密改写
  原始 INSERT：
    INSERT INTO t_user (id, phone) VALUES (1, '13800138000')
  
  改写后：
    INSERT INTO t_user (id, phone_cipher, phone_plain)
    VALUES (1, 'AES_ENCRYPTED', '138****0000')

改写类型5：审计 SQL（添加注释）
  改写：SELECT * FROM t_order_2 WHERE user_id = 1001
        /* sharding-datasource:ds_1, logic-table:t_order */
```

---

### 引擎4：执行引擎（Execute Engine）

管理 SQL 在多个数据源上的并行/串行执行。

```
执行流程：

RouteResult（路由结果）
       |
       ↓
+--------------------+
| StatementExecuteUnit 列表：                |
|   Unit1: ds_0.t_order_0 → SQL_A          |
|   Unit2: ds_1.t_order_1 → SQL_B          |
|   Unit3: ds_2.t_order_2 → SQL_C          |
+--------------------+
       |
       ↓（并行执行）
+--------+ +--------+ +--------+
| 获取    | | 获取    | | 获取    |
| ds_0    | | ds_1    | | ds_2    |
| 连接    | | 连接    | | 连接    |
+--------+ +--------+ +--------+
     |           |           |
     ↓           ↓           ↓
  执行SQL_A  执行SQL_B  执行SQL_C
  (并行)     (并行)     (并行)
     |           |           |
     ↓           ↓           ↓
  ResultSet0  ResultSet1  ResultSet2
       |
       ↓（收集）
  [RS0, RS1, RS2] → 传给归并引擎

执行策略：
  单分片：直接同步执行（避免线程创建开销）
  多分片：使用线程池并行执行
  
  线程池配置：
    executor-size: 20  （默认为 CPU 核数）
  
  连接模式：
    MEMORY_STRICTLY：每个分片一个连接（内存友好，适合结果集大）
    CONNECTION_STRICTLY：尽量少连接（连接池友好，适合连接数有限）
```

---

### 引擎5：归并引擎（Merge Engine）

将多个分片的 ResultSet 合并为逻辑上的单一结果集。

```
归并类型：

1. 遍历归并（Traversal Merge）
   最简单，不需要排序
   适合：单分片结果 或 不需要排序的多分片结果

2. 排序归并（Order By Merge）
   每个分片已经内部排好序
   使用优先队列（最小堆）归并
   
   示例（4个分片，按 id ASC 归并）：
   t_order_0: [2, 6, 10, 14]
   t_order_1: [1, 5, 9, 13]
   t_order_2: [4, 8, 12, 16]
   t_order_3: [3, 7, 11, 15]
   
   优先队列（最小堆）：
   初始化：push(2,1,4,3) → 堆:[1,2,3,4]
   取最小 1（来自t_order_1），push t_order_1 下一个 5
   取最小 2（来自t_order_0），push t_order_0 下一个 6
   ...
   结果：[1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16]
   时间复杂度：O(N log K)，N=总记录数，K=分片数

3. 聚合归并（Aggregation Merge）
   COUNT：各分片 COUNT 值相加
   SUM：  各分片 SUM 值相加
   MAX：  取各分片 MAX 中的最大值
   MIN：  取各分片 MIN 中的最小值
   AVG：  改写为 COUNT+SUM，最后 SUM/COUNT
   
   示例：SELECT COUNT(*), SUM(amount), AVG(amount) FROM t_order
   分片0：COUNT=100, SUM=10000
   分片1：COUNT=150, SUM=16500
   归并：COUNT = 100+150 = 250
         SUM   = 10000+16500 = 26500
         AVG   = 26500/250 = 106.0

4. 分组归并（Group By Merge）
   各分片分别 GROUP BY
   归并时相同 group key 的记录再次聚合
   
   注意：如果 GROUP BY 的列不是分片键
         每个分片的结果可能有相同的 key
         需要归并引擎再次聚合

5. 分页归并（Pagination Merge）
   结合排序归并，取第 offset+1 ~ offset+count 条
   前面 offset 条数据读出来后丢弃（内存浪费）
   深分页性能差的根本原因
```

---

## 2.3 SQL 执行全流程（端到端）

```
+------------------------------------------------------------------+
|                  ShardingSphere SQL 完整执行流程                   |
+------------------------------------------------------------------+

用户 SQL：
  "SELECT * FROM t_order WHERE user_id = 1001 ORDER BY create_time DESC LIMIT 10"

STEP 1: SQL 解析
  ┌─────────────────────────────────────────────────────┐
  │ ParseEngine                                         │
  │                                                     │
  │ 词法分析 → 语法分析 → 生成 AST                         │
  │                                                     │
  │ SelectStatement {                                   │
  │   table: "t_order",                                 │
  │   where: { user_id = 1001 },                        │
  │   orderBy: { create_time DESC },                    │
  │   limit: { offset:0, rowCount:10 }                  │
  │ }                                                   │
  └─────────────────────────────────────────────────────┘
                          ↓
STEP 2: 路由计算
  ┌─────────────────────────────────────────────────────┐
  │ RouteEngine                                         │
  │                                                     │
  │ 分片键: user_id = 1001                              │
  │ 分库算法: 1001 % 4 = 1 → ds_1                       │
  │ 分表算法: 无 order_id 条件 → t_order_0~3 全扫       │
  │                                                     │
  │ RouteResult: [ds_1.t_order_0, ds_1.t_order_1,      │
  │               ds_1.t_order_2, ds_1.t_order_3]      │
  └─────────────────────────────────────────────────────┘
                          ↓
STEP 3: SQL 改写
  ┌─────────────────────────────────────────────────────┐
  │ RewriteEngine                                       │
  │                                                     │
  │ 对每个路由目标改写 SQL：                               │
  │                                                     │
  │ ds_1.t_order_0:                                     │
  │   SELECT * FROM t_order_0 WHERE user_id = 1001      │
  │   ORDER BY create_time DESC LIMIT 0, 10             │
  │                                                     │
  │ ds_1.t_order_1:                                     │
  │   SELECT * FROM t_order_1 WHERE user_id = 1001      │
  │   ORDER BY create_time DESC LIMIT 0, 10             │
  │                                                     │
  │ （分页改写：LIMIT 10 → LIMIT 0, 10，因offset=0不变） │
  └─────────────────────────────────────────────────────┘
                          ↓
STEP 4: 并行执行
  ┌─────────────────────────────────────────────────────┐
  │ ExecuteEngine                                       │
  │                                                     │
  │ 线程1: 执行 t_order_0 → ResultSet_0                 │
  │ 线程2: 执行 t_order_1 → ResultSet_1                 │
  │ 线程3: 执行 t_order_2 → ResultSet_2                 │
  │ 线程4: 执行 t_order_3 → ResultSet_3                 │
  │                                                     │
  │ 等待所有线程完成...                                   │
  └─────────────────────────────────────────────────────┘
                          ↓
STEP 5: 结果归并
  ┌─────────────────────────────────────────────────────┐
  │ MergeEngine                                         │
  │                                                     │
  │ 各 ResultSet 已按 create_time DESC 排序              │
  │ 使用优先队列（最大堆）做 ORDER BY 归并                 │
  │ 取前 10 条记录                                       │
  │                                                     │
  │ 最终结果：10 条按 create_time 排好序的订单记录         │
  └─────────────────────────────────────────────────────┘
                          ↓
返回给用户：10 条订单数据（完全透明，应用无感知分片）

+------------------------------------------------------------------+
```

---

---

# Part 3: 分片策略深度解析 {#part-3}

## 3.1 分片键选择原则

分片键（Sharding Key）是决定数据路由的关键字段，选择直接影响系统性能和可维护性。

```
选择原则1：高频查询必带该字段
  ✓ 用户ID（90% 查询按用户维度）
  ✓ 订单ID（订单详情查询）
  ✗ 创建时间（范围查询多，但数据倾斜严重）
  ✗ 商品分类（查询条件不固定）

选择原则2：数据分布均匀，无热点
  ✓ 用户ID（雪花ID，分布均匀）
  ✓ 订单ID（雪花ID自带分散性）
  ✗ 地区码（南方用户多 → 部分分片热）
  ✗ 性别（男女比例可能不均）

选择原则3：业务关联性强，减少跨分片
  ✓ 订单表和订单详情表都用 order_id 分片
     同一订单的所有数据在同一分片，可以 JOIN
  ✗ 订单表用 user_id，订单详情用 order_id
     同一用户的订单和明细不在同一分片，必须全路由

选择原则4：确定后不能轻易修改
  修改分片键 = 全量数据迁移（迁移 50亿条数据需数天）
  建议：先规划好，再上线

选择原则5：基数要大（Cardinality）
  ✓ 用户ID（1000万用户，基数大）
  ✗ 订单状态（只有5种状态，基数太小）
     用状态分片 → 每个分片数据量严重不均
```

---

## 3.2 四种标准分片算法

### 3.2.1 精确分片算法（PreciseShardingAlgorithm）

适用于 `=` 和 `IN` 操作符，每次路由到**一个**目标分片。

```java
/**
 * 自定义精确分片算法：按 user_id 取模分片
 *
 * 使用场景：
 *   WHERE user_id = 1001          → 精确路由到1个分片
 *   WHERE user_id IN (1001, 1002) → 路由到多个分片（每个值调用一次）
 */
@SPI("USER_PRECISE_MOD")
public class UserIdPreciseShardingAlgorithm
        implements PreciseShardingAlgorithm<Long> {

    private int shardCount;

    @Override
    public String doSharding(
            Collection<String> availableTargetNames,
            PreciseShardingValue<Long> shardingValue) {

        Long userId = shardingValue.getValue();
        // 取模计算目标分片索引
        int index = (int) (userId % shardCount);
        // 构造目标分片名（表名或库名）
        String target = shardingValue.getLogicTableName() + "_" + index;

        // 从可用目标列表中匹配
        for (String name : availableTargetNames) {
            if (name.endsWith("_" + index)) {
                return name;
            }
        }
        throw new IllegalArgumentException(
            "找不到匹配的分片，目标: " + target +
            "，可用分片: " + availableTargetNames
        );
    }

    @Override
    public void init(Properties props) {
        this.shardCount = Integer.parseInt(
            props.getProperty("shard-count", "4")
        );
    }

    @Override
    public String getType() {
        return "USER_PRECISE_MOD";
    }

    @Override
    public Properties getProps() {
        Properties props = new Properties();
        props.setProperty("shard-count", String.valueOf(shardCount));
        return props;
    }
}
```

**配置文件引用：**
```yaml
sharding-algorithms:
  user_precise_mod:
    type: USER_PRECISE_MOD
    props:
      shard-count: 4
```

**SPI 注册（META-INF/services/）：**
```
# 文件：META-INF/services/org.apache.shardingsphere.sharding.spi.ShardingAlgorithm
com.example.sharding.UserIdPreciseShardingAlgorithm
```

---

### 3.2.2 范围分片算法（RangeShardingAlgorithm）

适用于 `BETWEEN AND`、`>`、`<`、`>=`、`<=` 等范围条件，可能路由到**多个**分片。

```java
/**
 * 按时间范围分片（按月分表）
 *
 * 表命名规则：t_order_202301, t_order_202302, ...
 *
 * 使用场景：
 *   WHERE create_time BETWEEN '2023-01-01' AND '2023-03-31'
 *   → 路由到 t_order_202301, t_order_202302, t_order_202303
 */
@SPI("ORDER_MONTH_RANGE")
public class OrderMonthRangeShardingAlgorithm
        implements RangeShardingAlgorithm<Date> {

    private static final SimpleDateFormat SDF =
        new SimpleDateFormat("yyyyMM");

    @Override
    public Collection<String> doSharding(
            Collection<String> availableTargetNames,
            RangeShardingValue<Date> shardingValue) {

        Range<Date> range = shardingValue.getValueRange();

        Date lower = range.hasLowerBound()
                     ? range.lowerEndpoint() : null;
        Date upper = range.hasUpperBound()
                     ? range.upperEndpoint() : null;

        List<String> result = new ArrayList<>();

        for (String tableName : availableTargetNames) {
            if (tableInRange(tableName, lower, upper)) {
                result.add(tableName);
            }
        }

        if (result.isEmpty()) {
            // 降级：返回所有分片（全路由）
            return new ArrayList<>(availableTargetNames);
        }
        return result;
    }

    /**
     * 判断表是否在查询范围内
     * 表名格式：t_order_202301 → 提取 202301 → 2023年1月
     */
    private boolean tableInRange(String tableName, Date lower, Date upper) {
        // 提取表名后缀（如 202301）
        String suffix = tableName.substring(tableName.lastIndexOf('_') + 1);
        int yearMonth;
        try {
            yearMonth = Integer.parseInt(suffix);
        } catch (NumberFormatException e) {
            return false;
        }

        int year  = yearMonth / 100;
        int month = yearMonth % 100;

        // 计算该分片表覆盖的时间范围
        Calendar tableStart = Calendar.getInstance();
        tableStart.set(year, month - 1, 1, 0, 0, 0);
        tableStart.set(Calendar.MILLISECOND, 0);

        Calendar tableEnd = Calendar.getInstance();
        tableEnd.set(year, month - 1, 1, 0, 0, 0);
        tableEnd.set(Calendar.MILLISECOND, 0);
        tableEnd.add(Calendar.MONTH, 1);

        // 判断范围是否与表的时间区间有交集
        if (lower != null &&
            lower.compareTo(tableEnd.getTime()) >= 0) {
            return false;   // 查询下界已超过该表的上界
        }
        if (upper != null &&
            upper.compareTo(tableStart.getTime()) < 0) {
            return false;   // 查询上界低于该表的下界
        }
        return true;
    }

    @Override
    public String getType() { return "ORDER_MONTH_RANGE"; }

    @Override
    public void init(Properties props) {}

    @Override
    public Properties getProps() { return new Properties(); }
}
```

---

### 3.2.3 复合分片算法（ComplexKeysShardingAlgorithm）

同时使用**多个列**作为分片键，实现多维度路由。

```java
/**
 * 复合分片算法：
 *   - user_id 决定分库（4个库）
 *   - order_type 决定分表（4张表）
 *
 * SQL 示例：
 *   WHERE user_id = 1001 AND order_type = 2
 *   → ds_1.t_order_2（精确路由）
 *
 *   WHERE user_id = 1001
 *   → ds_1.t_order_0, ds_1.t_order_1, ds_1.t_order_2, ds_1.t_order_3
 *   （只有库分片键，表全扫）
 */
@SPI("ORDER_COMPLEX")
public class OrderComplexShardingAlgorithm
        implements ComplexKeysShardingAlgorithm<Long> {

    private int dbCount;
    private int tableCount;

    @Override
    public Collection<String> doSharding(
            Collection<String> availableTargetNames,
            ComplexKeysShardingValue<Long> shardingValue) {

        // 获取各分片键的精确值集合
        Map<String, Collection<Long>> columnValues =
            shardingValue.getColumnNameAndShardingValuesMap();

        // 获取各分片键的范围值
        Map<String, Range<Long>> columnRanges =
            shardingValue.getColumnNameAndRangeValuesMap();

        Collection<Long> userIds  = columnValues.get("user_id");
        Collection<Long> orderTypes = columnValues.get("order_type");

        Set<String> result = new LinkedHashSet<>();

        if (userIds != null && !userIds.isEmpty()) {
            // 有 user_id 条件，可以精确分库
            for (Long userId : userIds) {
                int dbIdx = (int) (userId % dbCount);
                String dbSuffix = "_" + dbIdx + ".";

                if (orderTypes != null && !orderTypes.isEmpty()) {
                    // 同时有 order_type，精确路由到单个分片
                    for (Long orderType : orderTypes) {
                        int tableIdx = (int) (orderType % tableCount);
                        for (String target : availableTargetNames) {
                            if (target.contains(dbSuffix) &&
                                target.endsWith("_" + tableIdx)) {
                                result.add(target);
                            }
                        }
                    }
                } else {
                    // 只有 user_id，分库内全表扫描
                    for (String target : availableTargetNames) {
                        if (target.contains(dbSuffix)) {
                            result.add(target);
                        }
                    }
                }
            }
        } else {
            // 无分片键，全路由
            result.addAll(availableTargetNames);
        }

        return result;
    }

    @Override
    public void init(Properties props) {
        this.dbCount    = Integer.parseInt(props.getProperty("db-count", "4"));
        this.tableCount = Integer.parseInt(props.getProperty("table-count", "4"));
    }

    @Override
    public String getType() { return "ORDER_COMPLEX"; }

    @Override
    public Properties getProps() { return new Properties(); }
}
```

**复合分片配置：**
```yaml
tables:
  t_order:
    actual-data-nodes: ds_$->{0..3}.t_order_$->{0..3}
    database-strategy:
      complex:
        sharding-columns: user_id,order_type       # 多个分片键
        sharding-algorithm-name: order_complex_alg
    table-strategy:
      complex:
        sharding-columns: user_id,order_type
        sharding-algorithm-name: order_complex_alg

sharding-algorithms:
  order_complex_alg:
    type: ORDER_COMPLEX
    props:
      db-count: 4
      table-count: 4
```

---

### 3.2.4 Hint 强制路由算法（HintShardingAlgorithm）

通过代码手动指定路由目标，绕过 SQL 解析，适合特殊场景。

```java
/**
 * Hint 分片算法：根据代码中手动设置的分片值路由
 *
 * 适用场景：
 *   1. SQL 中没有分片键列（如全表统计）但知道要查哪个分片
 *   2. 需要强制读主库（读写分离 + Hint）
 *   3. 分片键在 HTTP Header 或 Token 中
 */
@SPI("ORDER_HINT_MOD")
public class OrderHintShardingAlgorithm
        implements HintShardingAlgorithm<Long> {

    private int shardCount;

    @Override
    public Collection<String> doSharding(
            Collection<String> availableTargetNames,
            HintShardingValue<Long> shardingValue) {

        Collection<Long> values = shardingValue.getValues();
        List<String> result = new ArrayList<>();

        for (Long value : values) {
            int index = (int) (value % shardCount);
            for (String target : availableTargetNames) {
                if (target.endsWith("_" + index)) {
                    result.add(target);
                }
            }
        }

        return result.isEmpty() ? new ArrayList<>(availableTargetNames) : result;
    }

    @Override
    public String getType() { return "ORDER_HINT_MOD"; }

    @Override
    public void init(Properties props) {
        this.shardCount = Integer.parseInt(
            props.getProperty("shard-count", "4")
        );
    }

    @Override
    public Properties getProps() { return new Properties(); }
}
```

**Hint 使用示例：**
```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 使用 Hint 强制路由到指定分片
     * 场景：查询统计数据，SQL 无分片键，但已知分片
     */
    public List<OrderStatDTO> getStatByHint(Long userId) {
        // HintManager 实现了 AutoCloseable，try-with-resources 自动清理
        try (HintManager hintManager = HintManager.getInstance()) {
            // 为 t_order 表设置数据库分片值
            hintManager.addDatabaseShardingValue("t_order", userId);
            // 为 t_order 表设置表分片值
            hintManager.addTableShardingValue("t_order", userId);

            // SQL 中无分片键也能精确路由
            return orderMapper.selectStatByUserId(userId);
        }
    }

    /**
     * 强制路由到主库（读写分离场景，确保读到最新数据）
     */
    public Order getForUpdate(Long orderId) {
        try (HintManager hintManager = HintManager.getInstance()) {
            hintManager.setWriteRouteOnly(); // 强制走主库
            return orderMapper.selectById(orderId);
        }
    }

    /**
     * Web 层拦截器：从请求头提取分片信息设置 Hint
     */
    // 在 Filter/Interceptor 中：
    // String userId = request.getHeader("X-User-Id");
    // if (userId != null) {
    //     HintManager.getInstance()
    //         .addDatabaseShardingValue("t_order", Long.parseLong(userId));
    // }
}
```

---

## 3.3 内置分片算法大全

ShardingSphere 5.x 内置多种开箱即用的分片算法：

### MOD（取模算法）

```yaml
# 最简单的分片算法，最常用
sharding-algorithms:
  order_mod:
    type: MOD
    props:
      sharding-count: 4    # 分片总数

# 路由规则：分片值 % sharding-count → 分片索引
# user_id = 1001 → 1001 % 4 = 1 → ds_1 / t_order_1
# user_id = 1004 → 1004 % 4 = 0 → ds_0 / t_order_0

# 适用：分片键为整数，分布均匀
# 注意：扩容时（4→8分片）需要全量数据迁移！
```

### HASH_MOD（哈希取模）

```yaml
# 先哈希再取模，适合字符串等非整数分片键
sharding-algorithms:
  order_hash_mod:
    type: HASH_MOD
    props:
      sharding-count: 4

# 路由规则：hash(分片值) % sharding-count → 分片索引
# phone = "13800138000" → hash("13800138000") % 4 → 某个分片

# 适用：
#   分片键为字符串（UUID、手机号）
#   整数分布不均匀（如地区编码集中在某个范围）
```

### BOUNDARY_RANGE（范围边界算法）

```yaml
# 按固定值范围划分分片
sharding-algorithms:
  order_id_boundary:
    type: BOUNDARY_RANGE
    props:
      sharding-ranges: 1-10000000,10000001-20000000,20000001-30000000,30000001-MAX

# 分片映射：
#   [1, 10000000]         → 分片0
#   [10000001, 20000000]  → 分片1
#   [20000001, 30000000]  → 分片2
#   [30000001, MAX]       → 分片3

# 适用：有明确数值范围，新老数据迁移
# 缺点：范围设置不当会导致数据倾斜
```

### INTERVAL（时间间隔分片）

```yaml
# 按时间间隔自动路由，最适合日志/流水类按月分表
sharding-algorithms:
  create_time_interval:
    type: INTERVAL
    props:
      # 时间戳格式（分片键的格式）
      datetime-pattern: "yyyy-MM-dd HH:mm:ss"
      # 分片起始时间（早于此时间的数据都路由到第一个分片）
      datetime-lower: "2020-01-01 00:00:00"
      # 分片结束时间（超出此时间的数据路由到最后一个分片）
      datetime-upper: "2030-12-31 23:59:59"
      # 分片表名后缀格式
      sharding-suffix-pattern: "yyyyMM"
      # 每个分片覆盖的时间跨度
      datetime-interval-amount: 1
      datetime-interval-unit: MONTHS   # 可选: DAYS/MONTHS/YEARS

# 效果示例：
#   create_time = '2023-01-15' → 后缀 202301 → t_order_202301
#   create_time = '2023-06-20' → 后缀 202306 → t_order_202306
#   create_time = '2024-12-01' → 后缀 202412 → t_order_202412

# 注意：需要提前创建好对应的物理分片表！
# 生产建议：写定时任务，每月自动创建下个月的分片表
```

**自动建表脚本示例：**
```java
@Component
@Slf4j
public class MonthlyTableCreator {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 每月1号凌晨1点，提前创建下个月的分片表
     */
    @Scheduled(cron = "0 0 1 1 * ?")
    public void createNextMonthTables() {
        // 计算下个月的年月
        LocalDate nextMonth = LocalDate.now().plusMonths(1);
        String suffix = nextMonth.format(DateTimeFormatter.ofPattern("yyyyMM"));

        // 4个库，每个库创建一张对应月份的订单表
        for (int dbIndex = 0; dbIndex < 4; dbIndex++) {
            String createSQL = String.format(
                "CREATE TABLE IF NOT EXISTS `order_db_%d`.`t_order_%s` (" +
                "  `order_id`    BIGINT          NOT NULL COMMENT '订单ID'," +
                "  `user_id`     BIGINT          NOT NULL COMMENT '用户ID'," +
                "  `amount`      DECIMAL(12,2)   NOT NULL COMMENT '金额'," +
                "  `create_time` DATETIME        NOT NULL COMMENT '创建时间'," +
                "  PRIMARY KEY (`order_id`)," +
                "  KEY `idx_user_id` (`user_id`)," +
                "  KEY `idx_create_time` (`create_time`)" +
                ") ENGINE=InnoDB DEFAULT CHARSET=utf8mb4",
                dbIndex, suffix
            );

            try {
                jdbcTemplate.execute(createSQL);
                log.info("创建分片表成功: order_db_{}.t_order_{}", dbIndex, suffix);
            } catch (Exception e) {
                log.error("创建分片表失败: order_db_{}.t_order_{}", dbIndex, suffix, e);
                // 告警通知
            }
        }
    }
}
```

### COSID_MOD（高性能雪花ID取模）

```yaml
# 使用 CosID 雪花算法生成的 ID，支持更高并发
sharding-algorithms:
  cosid_mod:
    type: COSID_MOD
    props:
      mod: 4
      logic-name-prefix: "t_order_"

# CosID vs Snowflake 性能对比：
#   Snowflake：有时钟回拨风险，单机 QPS < 100万
#   CosID：    解决时钟回拨，单机 QPS > 500万
```

### AUTO_INTERVAL（自动时间分片）

```yaml
# 根据起止时间和总分片数自动计算分片间隔
sharding-algorithms:
  auto_interval:
    type: AUTO_INTERVAL
    props:
      datetime-lower: "2020-01-01 00:00:00"
      datetime-upper: "2025-01-01 00:00:00"
      sharding-seconds: 2592000   # 每30天（秒数）一个分片

# 特点：无需手动指定后缀格式，自动按索引命名
# 适合：预估总分片数，不关心具体时间段的场景
```

---

## 3.4 分片路由类型详解

```
路由全景图：

+------------------------------------------------------------------+
|                        路由类型决策                               |
+------------------------------------------------------------------+
|                                                                  |
|  进入路由引擎                                                     |
|        |                                                         |
|        ├── [Hint 已设置]─────────────────────→ Hint 路由         |
|        |                                        强制指定分片      |
|        |                                                         |
|        ├── [广播表]──────────────────────────→ 广播路由           |
|        |                                        所有分片执行       |
|        |                                        DDL/修改广播表    |
|        |                                                         |
|        ├── [不含任何分片表的 SQL]────────────→ 单播路由            |
|        |                                        发往任意一个分片   |
|        |                                                         |
|        └── [含分片表]                                            |
|                  |                                               |
|                  ├── [含分片键，= / IN]────→ 标准路由（精确）      |
|                  |     user_id=1001           路由到1个分片       |
|                  |                                               |
|                  ├── [含分片键，范围]──────→ 范围路由              |
|                  |     user_id>1000            路由到连续多个分片  |
|                  |                                               |
|                  ├── [多分片表 JOIN]                              |
|                  |     ├── [绑定表]──────→ 绑定表路由             |
|                  |     │                   同分片内 JOIN，1条SQL  |
|                  |     └── [非绑定表]────→ 笛卡尔积路由            |
|                  |                         多条 SQL，结果合并    |
|                  |                                               |
|                  └── [无分片键]──────────→ 全路由（Full Route）   |
|                                              所有分片执行         |
|                                              性能最差！           |
+------------------------------------------------------------------+
```

### 各路由类型性能对比

```
路由类型        目标分片数    性能    适用场景
-----------    ----------   ------  ---------------
Hint路由        1~N          最高    强制指定，精确
标准路由(=)     1            高      等值查询
标准路由(IN)    N（N<总数）   中      多值查询
绑定表路由      1            高      JOIN绑定表
范围路由        部分          中      范围查询
广播路由        全部          低      DDL，广播表写
全路由          全部          最低    无分片键查询

4库16表 全路由 = 执行 16 条 SQL！
务必通过 SQL 解析日志监控全路由告警
```

---

---

# Part 4: 数据节点配置 {#part-4}

## 4.1 行内表达式（Inline Expression）

ShardingSphere 支持 Groovy 行内表达式，使配置更简洁灵活。

```yaml
# 语法：$->{Groovy表达式} 或 ${Groovy表达式}
# 在 YAML 中推荐使用 $->{}（避免与 YAML 占位符冲突）

# 示例1：连续范围（最常用）
actual-data-nodes: ds_$->{0..3}.t_order_$->{0..3}
# 展开：ds_0.t_order_0, ds_0.t_order_1, ds_0.t_order_2, ds_0.t_order_3
#       ds_1.t_order_0, ds_1.t_order_1, ds_1.t_order_2, ds_1.t_order_3
#       ds_2.t_order_0, ...
#       ds_3.t_order_0, ... （共 4×4=16 个节点）

# 示例2：单库多表
actual-data-nodes: ds_0.t_order_$->{0..7}
# 展开：ds_0.t_order_0 ~ ds_0.t_order_7（8张表）

# 示例3：固定列表（枚举）
actual-data-nodes: ds_$->{['primary','replica']}.t_user
# 展开：ds_primary.t_user, ds_replica.t_user

# 示例4：步长
actual-data-nodes: ds_$->{(0..7).step(2)}.t_order
# 展开：ds_0.t_order, ds_2.t_order, ds_4.t_order, ds_6.t_order

# 示例5：作为分片算法（INLINE类型）
sharding-algorithms:
  db_inline:
    type: INLINE
    props:
      algorithm-expression: ds_$->{user_id % 4}
      # user_id = 1001 → ds_1
      # user_id = 1004 → ds_0

  table_inline:
    type: INLINE
    props:
      algorithm-expression: t_order_$->{order_id % 4}
      # order_id = 5678 → t_order_2
```

**行内表达式注意事项：**
```
注意1：INLINE 类型仅支持精确分片（= 和 IN）
  不支持 BETWEEN AND 等范围查询
  范围查询需要自定义 RangeShardingAlgorithm

注意2：INLINE 不支持复杂 Groovy
  正确：t_order_$->{user_id % 4}
  错误：t_order_$->{user_id > 100 ? user_id % 4 : 0}  # 不支持条件

注意3：Groovy 闭包中的类型
  分片键为 Long 时，% 操作返回 Long，需要注意
  t_order_$->{user_id.toLong() % 4}
```

---

## 4.2 数据节点配置策略

### 均匀分布（推荐）

```yaml
# 最推荐的配置，数据均匀分布
tables:
  t_order:
    actual-data-nodes: ds_$->{0..3}.t_order_$->{0..3}

# 物理表结构：
# +--------+-----------+-----------+-----------+-----------+
# |        | t_order_0 | t_order_1 | t_order_2 | t_order_3 |
# +--------+-----------+-----------+-----------+-----------+
# | ds_0   |    ✓      |    ✓      |    ✓      |    ✓      |
# | ds_1   |    ✓      |    ✓      |    ✓      |    ✓      |
# | ds_2   |    ✓      |    ✓      |    ✓      |    ✓      |
# | ds_3   |    ✓      |    ✓      |    ✓      |    ✓      |
# +--------+-----------+-----------+-----------+-----------+
# 共 16 张物理表，理论上数据均匀分布

# 分片键路由（双分片键场景）：
#   库键：user_id % 4    → 决定用哪个 ds
#   表键：order_id % 4   → 决定用哪张 t_order_x
```

### 非对称分布

```yaml
# 不同库拥有不同数量的分表（遗留系统迁移场景）
tables:
  t_order:
    actual-data-nodes: >
      ds_0.t_order_0,
      ds_0.t_order_1,
      ds_0.t_order_2,
      ds_0.t_order_3,
      ds_1.t_order_0,
      ds_1.t_order_1,
      ds_1.t_order_2,
      ds_1.t_order_3,
      ds_2.t_order_history    # 历史归档库，只有一张汇总表
```

### 按时间分片的动态节点

```yaml
# 按月分表，表名含年月后缀
tables:
  t_order_log:
    # 配置已存在的月份表
    actual-data-nodes: >
      ds_0.t_order_log_202301,
      ds_0.t_order_log_202302,
      ds_0.t_order_log_202303,
      ds_0.t_order_log_202304,
      ds_0.t_order_log_202305,
      ds_0.t_order_log_202306,
      ds_0.t_order_log_202307,
      ds_0.t_order_log_202308,
      ds_0.t_order_log_202309,
      ds_0.t_order_log_202310,
      ds_0.t_order_log_202311,
      ds_0.t_order_log_202312
    table-strategy:
      standard:
        sharding-column: create_time
        sharding-algorithm-name: log_month_interval

# 注意：actual-data-nodes 需要包含所有已存在的物理表
# 新建月份表后，需要同步更新配置（或使用治理中心动态推送）
```

---

## 4.3 绑定表（Binding Tables）

绑定表是指使用相同分片规则的一组关联表。ShardingSphere 保证绑定表中的关联数据落在同一个分片，从而避免 JOIN 时的笛卡尔积。

```
绑定表示例：

  t_order       ←→  t_order_item
  分片键: user_id     分片键: user_id
  都用 user_id % 4 分片

  保证：
    user_id = 1001 的订单 → ds_1.t_order_1
    user_id = 1001 的明细 → ds_1.t_order_item_1

  同一用户的订单和明细 永远在同一个分片！
  → JOIN 只需要一条 SQL，不需要跨分片

未配置绑定表的问题（笛卡尔积）：
  t_order 路由到 ds_1.t_order_1
  t_order_item 独立路由到 ds_1.t_order_item_0/1/2/3

  JOIN 产生笛卡尔积：1 × 4 = 4 条 SQL
    SQL1: t_order_1 JOIN t_order_item_0
    SQL2: t_order_1 JOIN t_order_item_1
    SQL3: t_order_1 JOIN t_order_item_2
    SQL4: t_order_1 JOIN t_order_item_3
  → 结果需要应用层合并，可能有重复数据，性能极差
```

**绑定表配置：**
```yaml
rules:
  sharding:
    tables:
      t_order:
        actual-data-nodes: ds_$->{0..3}.t_order_$->{0..3}
        database-strategy:
          standard:
            sharding-column: user_id
            sharding-algorithm-name: db_mod
        table-strategy:
          standard:
            sharding-column: order_id
            sharding-algorithm-name: table_mod
        key-generate-strategy:
          column: order_id
          key-generator-name: snowflake

      t_order_item:
        actual-data-nodes: ds_$->{0..3}.t_order_item_$->{0..3}
        # 分片规则必须与 t_order 完全相同！
        database-strategy:
          standard:
            sharding-column: user_id
            sharding-algorithm-name: db_mod    # 同一个算法
        table-strategy:
          standard:
            sharding-column: order_id
            sharding-algorithm-name: table_mod # 同一个算法
        key-generate-strategy:
          column: item_id
          key-generator-name: snowflake

    # 声明绑定关系
    binding-tables:
      - t_order,t_order_item          # 第一组绑定表
      - t_user,t_user_account         # 第二组绑定表（不同组）
```

---

## 4.4 广播表（Broadcast Tables）

广播表是指在每个分片数据库中都有完整数据的表，适合字典表、配置表等数据量小、变更少的表。

```
广播表原理：

写操作（INSERT/UPDATE/DELETE）：
  广播到所有分片库执行
  保证各分片数据一致

  UPDATE t_dict_category SET name='服装' WHERE id=1;
  → 在 ds_0, ds_1, ds_2, ds_3 分别执行
  → 各库数据同步更新

读操作（SELECT）：
  只从任意一个分片读（单播路由）
  因为每个分片数据相同，读哪个都一样

JOIN 效果：
  t_order JOIN t_dict_category
  t_order 在 ds_1.t_order_2
  t_dict_category 在 ds_1 也有完整数据！
  → 可以在 ds_1 内部 JOIN，无需跨库
```

**广播表配置：**
```yaml
rules:
  sharding:
    tables:
      t_order:
        actual-data-nodes: ds_$->{0..3}.t_order_$->{0..3}
        # ...分片配置...

    # 广播表（每个分片库都有完整数据）
    broadcast-tables:
      - t_dict_category     # 商品分类字典
      - t_dict_status       # 状态字典
      - t_sys_config        # 系统配置
      - t_region            # 行政区划
      - t_goods_brand       # 品牌表（数据量不超过10万）
```

**广播表使用注意事项：**
```
注意1：数据量限制
  广播表适合数据量 < 10万行的小表
  大表（> 100万行）用广播表会导致：
    写操作耗时增加（N个库同时写）
    每个库都占用大量存储

注意2：广播表写失败处理
  如果某个分片库写失败而其他库写成功：
  → 数据不一致！
  ShardingSphere 用 XA 事务保证广播表写的原子性

注意3：广播表查询不走路由引擎
  SELECT * FROM t_dict_category
  → 在任意一个分片库查询（通常是 ds_0）
  → 不需要归并

注意4：不能与分片表混用路由
  不要对广播表设置分片策略！
  广播表和分片表是互斥的配置
```

---

---

# Part 5: 读写分离 {#part-5}

## 5.1 读写分离原理

```
读写分离架构：

  应用程序
       |
       ↓
+-----------------------------+
|  ShardingSphere 读写分离    |
|                             |
|  写路由：INSERT/UPDATE/DELETE|
|  读路由：SELECT（默认从库）  |
|  强制主库：Hint / 事务内     |
+-----------------------------+
       |              |
       | 写           | 读（负载均衡）
       ↓              ↓
  +--------+    +--------+  +--------+
  | 主库    |    | 从库1  |  | 从库2  |
  | Master |    | Slave1 |  | Slave2 |
  | 写入   |    | 只读   |  | 只读   |
  +--------+    +--------+  +--------+
       |              ↑          ↑
       |   binlog同步  |          |
       +--------------+----------+
            MySQL主从复制（异步）

路由规则：
  SQL类型                      路由目标
  --------                    --------
  INSERT / UPDATE / DELETE     主库（强制）
  SELECT（非事务）              从库（负载均衡）
  SELECT（事务内）              主库（事务一致性）
  SELECT（Hint强制主库）        主库
  DDL（CREATE/ALTER/DROP）      主库
```

---

## 5.2 完整读写分离配置

```yaml
spring:
  shardingsphere:
    datasource:
      # 数据源名称列表
      names: primary_ds,replica_ds_0,replica_ds_1

      # 主库（写库）
      primary_ds:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://db-master:3306/shop_db?useSSL=false&serverTimezone=Asia/Shanghai
        username: root
        password: Master@123
        hikari:
          minimum-idle: 10
          maximum-pool-size: 50
          connection-test-query: SELECT 1

      # 从库1（读库）
      replica_ds_0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://db-slave-1:3306/shop_db?useSSL=false&serverTimezone=Asia/Shanghai
        username: readonly
        password: Slave@123
        hikari:
          minimum-idle: 5
          maximum-pool-size: 30

      # 从库2（读库）
      replica_ds_1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://db-slave-2:3306/shop_db?useSSL=false&serverTimezone=Asia/Shanghai
        username: readonly
        password: Slave@123
        hikari:
          minimum-idle: 5
          maximum-pool-size: 30

    rules:
      readwrite-splitting:
        data-sources:
          # 逻辑数据源名称（应用使用这个名称）
          readwrite_ds:
            # 写库
            write-data-source-name: primary_ds
            # 读库列表（多个从库）
            read-data-source-names:
              - replica_ds_0
              - replica_ds_1
            # 事务内读操作路由策略
            # PRIMARY：事务内 SELECT 也走主库（推荐）
            # FIXED：固定走同一个从库
            # DYNAMIC：根据负载均衡策略走从库
            transactional-read-query-strategy: PRIMARY
            # 负载均衡算法
            load-balancer-name: round_robin_lb

        # 负载均衡器定义
        load-balancers:
          # 轮询（默认）：依次轮流访问从库
          round_robin_lb:
            type: ROUND_ROBIN

          # 随机：随机选择从库
          random_lb:
            type: RANDOM

          # 权重：按权重比例分配读请求
          weight_lb:
            type: WEIGHT
            props:
              replica_ds_0: 2   # replica_ds_0 分配 2/3 的读请求
              replica_ds_1: 1   # replica_ds_1 分配 1/3 的读请求

          # 最少连接数（ShardingSphere 暂不内置，可自定义）

    props:
      sql-show: true
```

---

## 5.3 负载均衡算法详解

### 轮询（ROUND_ROBIN）

```
请求顺序：     1   2   3   4   5   6   7   8
路由目标：  slave1 slave2 slave1 slave2 slave1 slave2 slave1 slave2

特点：
  简单均匀，每个从库分配相同请求数
  适合从库配置相同的场景
  不考虑当前负载
```

### 随机（RANDOM）

```
请求：每次随机选择 slave1 或 slave2

特点：
  简单，无状态
  长期来看请求分布均匀（概率论）
  短期可能不均匀
```

### 权重（WEIGHT）

```
配置：slave1 权重=2, slave2 权重=1

请求分配：
  slave1 处理 2/(2+1) ≈ 66.7% 的请求
  slave2 处理 1/(2+1) ≈ 33.3% 的请求

适用场景：
  从库配置不同（slave1 配置高，slave2 配置低）
  冷热从库分离（slave1 是主要读库，slave2 是备用）
```

### 自定义负载均衡器

```java
/**
 * 最少活跃连接数负载均衡算法（自定义实现）
 */
@SPI("LEAST_ACTIVE")
public class LeastActiveLoadBalancer implements ReadQueryLoadBalanceAlgorithm {

    // 每个从库当前活跃连接数
    private final ConcurrentHashMap<String, AtomicInteger> activeConnections
        = new ConcurrentHashMap<>();

    @Override
    public String getDataSource(String name,
                                 String writeDataSourceName,
                                 List<String> readDataSourceNames) {
        if (readDataSourceNames.isEmpty()) {
            return writeDataSourceName;
        }

        String selected = readDataSourceNames.get(0);
        int minActive = Integer.MAX_VALUE;

        for (String dsName : readDataSourceNames) {
            int active = activeConnections
                .computeIfAbsent(dsName, k -> new AtomicInteger(0))
                .get();
            if (active < minActive) {
                minActive = active;
                selected  = dsName;
            }
        }

        // 增加选中库的活跃连接计数
        activeConnections.get(selected).incrementAndGet();
        return selected;
    }

    /**
     * 连接释放时调用（需要和连接池集成）
     */
    public void releaseConnection(String dsName) {
        AtomicInteger counter = activeConnections.get(dsName);
        if (counter != null && counter.get() > 0) {
            counter.decrementAndGet();
        }
    }

    @Override
    public String getType() { return "LEAST_ACTIVE"; }

    @Override
    public void init(Properties props) {}

    @Override
    public Properties getProps() { return new Properties(); }
}
```

---

## 5.4 事务内强制路由主库

```java
@Service
@Slf4j
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private StockMapper stockMapper;

    /**
     * 场景1：事务内查询强制走主库
     *
     * 配置了 transactional-read-query-strategy: PRIMARY 后，
     * 事务内的所有 SELECT 自动走主库，无需任何代码修改。
     */
    @Transactional
    public OrderDTO createOrderAndQuery(CreateOrderReq req) {
        // 写操作 → 主库
        Order order = buildOrder(req);
        orderMapper.insert(order);

        // 事务内的读操作 → 自动走主库（防止主从延迟）
        // 如果走从库，可能读不到刚插入的订单（延迟1~100ms）
        Order saved = orderMapper.selectById(order.getOrderId());

        // 更新库存 → 主库
        stockMapper.deductStock(req.getGoodsId(), req.getQuantity());

        // 事务内查询库存 → 主库（避免延迟）
        Stock stock = stockMapper.selectByGoodsId(req.getGoodsId());

        log.info("订单创建成功, orderId={}, 剩余库存={}",
                 order.getOrderId(), stock.getStock());

        return convertToDTO(saved);
    }

    /**
     * 场景2：非事务场景，强制读主库（用 Hint）
     *
     * 适用：刚写完数据，立即需要读的场景
     *       但不想开启事务（减少事务开销）
     */
    public Order getLatestOrder(Long orderId) {
        try (HintManager hintManager = HintManager.getInstance()) {
            // 强制本次查询走主库
            hintManager.setWriteRouteOnly();
            return orderMapper.selectById(orderId);
        }
        // try-with-resources 自动调用 HintManager.close() 清理
    }

    /**
     * 场景3：普通查询走从库（默认行为）
     */
    public List<Order> queryOrders(Long userId, int page, int size) {
        // 默认走从库（轮询负载均衡）
        // 主从延迟在 100ms 以内，列表查询可以接受轻微延迟
        return orderMapper.selectByUserId(userId,
               (page - 1) * size, size);
    }

    private Order buildOrder(CreateOrderReq req) {
        Order order = new Order();
        // ShardingSphere 雪花算法自动生成 ID
        order.setUserId(req.getUserId());
        order.setGoodsId(req.getGoodsId());
        order.setAmount(req.getAmount());
        order.setStatus(0);
        order.setCreateTime(LocalDateTime.now());
        return order;
    }
}
```

---

## 5.5 读写分离 + 分库分表混合使用

**这是生产环境最常见的组合，配置稍复杂，分两步理解：**

```
第一步：配置读写分离（组合物理数据源 → 逻辑数据源）

  物理数据源：
    primary_ds0（主库0）
    replica_ds0_0（从库0-1）
    replica_ds0_1（从库0-2）
    primary_ds1（主库1）
    replica_ds1_0（从库1-1）
    replica_ds1_1（从库1-2）

  逻辑数据源（读写分离抽象）：
    ds_0 = primary_ds0 + replica_ds0_0 + replica_ds0_1
    ds_1 = primary_ds1 + replica_ds1_0 + replica_ds1_1

第二步：配置分库分表（使用逻辑数据源）

  t_order 按 user_id 分到 ds_0 或 ds_1
  t_order 按 order_id 分到 t_order_0~3

  ds_0/ds_1 内部自动处理读写分离
  写 → primary_ds0/primary_ds1
  读 → replica_ds0_x/replica_ds1_x（轮询）
```

**完整混合配置：**
```yaml
spring:
  shardingsphere:
    datasource:
      names: primary_ds0,replica_ds0_0,replica_ds0_1,primary_ds1,replica_ds1_0,replica_ds1_1

      primary_ds0:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://master-0:3306/order_db?useSSL=false
        username: root
        password: pwd123
        hikari:
          maximum-pool-size: 50

      replica_ds0_0:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave-0-1:3306/order_db?useSSL=false
        username: readonly
        password: pwd123
        hikari:
          maximum-pool-size: 30

      replica_ds0_1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave-0-2:3306/order_db?useSSL=false
        username: readonly
        password: pwd123
        hikari:
          maximum-pool-size: 30

      primary_ds1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://master-1:3306/order_db?useSSL=false
        username: root
        password: pwd123
        hikari:
          maximum-pool-size: 50

      replica_ds1_0:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave-1-1:3306/order_db?useSSL=false
        username: readonly
        password: pwd123
        hikari:
          maximum-pool-size: 30

      replica_ds1_1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave-1-2:3306/order_db?useSSL=false
        username: readonly
        password: pwd123
        hikari:
          maximum-pool-size: 30

    rules:
      # 规则1：读写分离（定义逻辑数据源）
      readwrite-splitting:
        data-sources:
          ds_0:                           # 逻辑数据源名
            write-data-source-name: primary_ds0
            read-data-source-names:
              - replica_ds0_0
              - replica_ds0_1
            transactional-read-query-strategy: PRIMARY
            load-balancer-name: rr
          ds_1:
            write-data-source-name: primary_ds1
            read-data-source-names:
              - replica_ds1_0
              - replica_ds1_1
            transactional-read-query-strategy: PRIMARY
            load-balancer-name: rr
        load-balancers:
          rr:
            type: ROUND_ROBIN

      # 规则2：分库分表（使用上面定义的逻辑数据源 ds_0/ds_1）
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds_$->{0..1}.t_order_$->{0..3}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db_mod
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table_mod
            key-generate-strategy:
              column: order_id
              key-generator-name: snowflake

        sharding-algorithms:
          db_mod:
            type: MOD
            props:
              sharding-count: 2
          table_mod:
            type: MOD
            props:
              sharding-count: 4

        key-generators:
          snowflake:
            type: SNOWFLAKE
            props:
              worker-id: 1

    props:
      sql-show: true
```

**整体数据流向图：**
```
写请求（INSERT ORDER）：
  user_id=1001 → ds_1（1001%2=1）→ t_order_x → primary_ds1（主库）
  
读请求（SELECT ORDER）：
  user_id=1001 → ds_1 → 轮询 → replica_ds1_0 或 replica_ds1_1

事务内读：
  user_id=1001 → ds_1 → PRIMARY策略 → primary_ds1（主库）
```

---

---

# Part 6: 分布式主键生成 {#part-6}

## 6.1 为什么不能用数据库自增 ID

```
问题演示（4库4表）：

ds_0.t_order_0: id从1开始自增 → 1, 2, 3, 4, 5...
ds_1.t_order_0: id从1开始自增 → 1, 2, 3, 4, 5...
ds_2.t_order_0: id从1开始自增 → 1, 2, 3, 4, 5...
ds_3.t_order_0: id从1开始自增 → 1, 2, 3, 4, 5...

结果：每个分片都有 id=1 的记录！

SELECT * FROM t_order WHERE id = 1;
→ 路由到哪个分片？不知道！只能全路由
→ 4个库 × 4张表 = 16条SQL
→ 查到16条 id=1 的记录，数据混乱！

改良方案1：步长模式
  ds_0：起始1，步长4 → 1, 5,  9,  13...
  ds_1：起始2，步长4 → 2, 6,  10, 14...
  ds_2：起始3，步长4 → 3, 7,  11, 15...
  ds_3：起始4，步长4 → 4, 8,  12, 16...

  缺陷：
  ① 扩容时步长和起始值需全量修改（代价极高）
  ② ID泄露分片信息（安全隐患）
  ③ 分表后每张分片表也要设置步长，配置复杂

结论：分布式系统需要全局唯一ID生成方案
```

---

## 6.2 雪花算法（Snowflake）深度解析

雪花算法是 Twitter 2010 年开源的分布式 ID 方案，生成 64 位有序整数。

```
64位 ID 结构：

 63    62~22          21~17       16~12        11~0
 ┌──┬──────────────────┬─────────┬───────────┬──────────┐
 │0 │  41位毫秒时间戳   │5位DataID│ 5位WorkID │ 12位序列号│
 └──┴──────────────────┴─────────┴───────────┴──────────┘
  ↑       ↑                ↑           ↑           ↑
符号位  距自定义起始时间   数据中心ID  工作机器ID  同毫秒内序列

各段说明：
  符号位（1bit）  ：始终为0，保证ID为正数
  时间戳（41bit） ：毫秒级时间戳 - 自定义起始时间
                    可用 2^41/1000/3600/24/365 ≈ 69年
  数据中心（5bit）：支持 2^5 = 32个数据中心
  工作节点（5bit）：每个数据中心支持 2^5 = 32台机器
  序列号（12bit） ：同一毫秒内 2^12 = 4096个ID

理论最大吞吐量：
  32 × 32 × 4096 个/ms = 4,194,304 个/ms = 约42亿/s
  单机每秒：4096 × 1000 = 4,096,000 = 约400万/s

ID 示例解析：
  ID = 1700000000000000001
  二进制：0 00110000111011100110101100101000000 00001 00001 000000000001
  时间戳部分：约 2023年某时刻
  数据中心：1
  工作节点：1
  序列号：  1
```

### 完整雪花算法 Java 实现

```java
package com.example.sharding.id;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

/**
 * 雪花算法 ID 生成器
 * 线程安全，包含时钟回拨处理
 */
public class SnowflakeIdGenerator {

    private static final Logger log = LoggerFactory.getLogger(SnowflakeIdGenerator.class);

    // ===== 各段位数 =====
    private static final long WORKER_ID_BITS     = 5L;  // 工作节点ID位数
    private static final long DATACENTER_ID_BITS = 5L;  // 数据中心ID位数
    private static final long SEQUENCE_BITS      = 12L; // 序列号位数

    // ===== 最大值（通过位运算计算）=====
    // -1L 的二进制：64个1；左移N位后取反得到 N个1
    private static final long MAX_WORKER_ID     = ~(-1L << WORKER_ID_BITS);     // 31
    private static final long MAX_DATACENTER_ID = ~(-1L << DATACENTER_ID_BITS); // 31
    private static final long MAX_SEQUENCE      = ~(-1L << SEQUENCE_BITS);      // 4095

    // ===== 位移量 =====
    private static final long WORKER_ID_SHIFT     = SEQUENCE_BITS;                              // 12
    private static final long DATACENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS;             // 17
    private static final long TIMESTAMP_SHIFT     = SEQUENCE_BITS + WORKER_ID_BITS
                                                    + DATACENTER_ID_BITS;                       // 22

    // 自定义起始时间（2020-01-01 00:00:00 UTC）
    private static final long EPOCH = 1577836800000L;

    // ===== 实例变量 =====
    private final long workerId;
    private final long datacenterId;

    private long lastTimestamp = -1L;
    private long sequence      = 0L;

    // 时钟回拨最大容忍毫秒数（超过则抛异常）
    private static final long MAX_CLOCK_BACKWARD_MS = 10L;

    public SnowflakeIdGenerator(long workerId, long datacenterId) {
        if (workerId > MAX_WORKER_ID || workerId < 0) {
            throw new IllegalArgumentException(
                String.format("workerId 超出范围 [0, %d]，实际值: %d",
                              MAX_WORKER_ID, workerId)
            );
        }
        if (datacenterId > MAX_DATACENTER_ID || datacenterId < 0) {
            throw new IllegalArgumentException(
                String.format("datacenterId 超出范围 [0, %d]，实际值: %d",
                              MAX_DATACENTER_ID, datacenterId)
            );
        }
        this.workerId     = workerId;
        this.datacenterId = datacenterId;
        log.info("雪花算法初始化，datacenterId={}, workerId={}", datacenterId, workerId);
    }

    /**
     * 生成下一个全局唯一 ID（线程安全）
     */
    public synchronized long nextId() {
        long currentMs = System.currentTimeMillis();

        // ===== 处理时钟回拨 =====
        if (currentMs < lastTimestamp) {
            long backwardMs = lastTimestamp - currentMs;
            if (backwardMs > MAX_CLOCK_BACKWARD_MS) {
                // 回拨太大（超过容忍范围）→ 直接报错，触发告警
                throw new RuntimeException(
                    String.format("时钟回拨 %dms 超过容忍上限 %dms，" +
                                  "请检查服务器时间同步！",
                                  backwardMs, MAX_CLOCK_BACKWARD_MS)
                );
            }
            // 小幅回拨：等待时钟追上来
            log.warn("检测到时钟回拨 {}ms，等待时钟同步...", backwardMs);
            try {
                Thread.sleep(backwardMs + 1);
                currentMs = System.currentTimeMillis();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new RuntimeException("等待时钟同步时被中断", e);
            }
        }

        // ===== 同一毫秒内，序列号递增 =====
        if (currentMs == lastTimestamp) {
            sequence = (sequence + 1) & MAX_SEQUENCE;
            if (sequence == 0) {
                // 当前毫秒内序列号已用完（达到4096），等待下一毫秒
                currentMs = waitNextMillis(lastTimestamp);
            }
        } else {
            // 新的毫秒，序列号重置为0
            sequence = 0L;
        }

        lastTimestamp = currentMs;

        // ===== 组装 64 位 ID =====
        return ((currentMs - EPOCH) << TIMESTAMP_SHIFT)
             | (datacenterId       << DATACENTER_ID_SHIFT)
             | (workerId           << WORKER_ID_SHIFT)
             | sequence;
    }

    /**
     * 忙等待到下一毫秒
     */
    private long waitNextMillis(long lastTimestamp) {
        long ts = System.currentTimeMillis();
        while (ts <= lastTimestamp) {
            ts = System.currentTimeMillis();
        }
        return ts;
    }

    /**
     * 解析 ID 各分段（调试用）
     */
    public static IdInfo parseId(long id) {
        long timestamp    = (id >> TIMESTAMP_SHIFT) + EPOCH;
        long datacenterId = (id >> DATACENTER_ID_SHIFT) & MAX_DATACENTER_ID;
        long workerId     = (id >> WORKER_ID_SHIFT) & MAX_WORKER_ID;
        long sequence     = id & MAX_SEQUENCE;

        return new IdInfo(id, timestamp, datacenterId, workerId, sequence);
    }

    public static class IdInfo {
        public final long id;
        public final long timestamp;
        public final long datacenterId;
        public final long workerId;
        public final long sequence;
        public final java.util.Date generateTime;

        public IdInfo(long id, long timestamp, long datacenterId,
                      long workerId, long sequence) {
            this.id           = id;
            this.timestamp    = timestamp;
            this.datacenterId = datacenterId;
            this.workerId     = workerId;
            this.sequence     = sequence;
            this.generateTime = new java.util.Date(timestamp);
        }

        @Override
        public String toString() {
            return String.format(
                "ID=%d [生成时间=%s, 数据中心=%d, 工作节点=%d, 序列号=%d]",
                id, generateTime, datacenterId, workerId, sequence
            );
        }
    }
}
```

### ShardingSphere 雪花算法配置

```yaml
spring:
  shardingsphere:
    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds_$->{0..3}.t_order_$->{0..3}
            key-generate-strategy:
              column: order_id          # 主键列
              key-generator-name: snowflake_keygen
          t_order_item:
            actual-data-nodes: ds_$->{0..3}.t_order_item_$->{0..3}
            key-generate-strategy:
              column: item_id
              key-generator-name: snowflake_keygen

        key-generators:
          snowflake_keygen:
            type: SNOWFLAKE
            props:
              # WorkerID：每个服务节点必须唯一！
              # 生产环境从环境变量/配置中心读取
              worker-id: ${SNOWFLAKE_WORKER_ID:1}
              # 最大振动偏移（时钟回拨容忍范围）
              max-vibration-offset: 1
```

**WorkerID 注册方案（生产环境）：**

```java
/**
 * 基于 Redis 的 WorkerID 自动注册
 * 每个实例启动时自动获取唯一 WorkerID（0~31）
 * 实例下线时自动释放
 */
@Component
@Slf4j
public class RedisWorkerIdAllocator implements InitializingBean, DisposableBean {

    private static final String REDIS_PREFIX = "snowflake:worker:";
    private static final int    MAX_WORKER_ID = 31;
    private static final int    EXPIRE_SECONDS = 30; // 租约时间

    @Autowired
    private StringRedisTemplate redisTemplate;

    @Value("${spring.application.name}")
    private String appName;

    private long workerId = -1;
    private ScheduledExecutorService renewExecutor;

    @Override
    public void afterPropertiesSet() throws Exception {
        String hostName = getHostIdentifier();
        workerId = registerWorkerId(hostName);

        // 启动租约续期线程（防止 WorkerID 被别的实例占用）
        renewExecutor = Executors.newSingleThreadScheduledExecutor(
            r -> new Thread(r, "worker-id-renew")
        );
        renewExecutor.scheduleAtFixedRate(
            this::renewLease, 10, 10, TimeUnit.SECONDS
        );

        log.info("WorkerID 注册成功：{}", workerId);
    }

    private long registerWorkerId(String hostName) {
        // 先检查是否已注册
        String existingKey = REDIS_PREFIX + appName + ":" + hostName;
        String existingId = redisTemplate.opsForValue().get(existingKey);
        if (existingId != null) {
            log.info("复用已注册的 WorkerID: {}", existingId);
            return Long.parseLong(existingId);
        }

        // 寻找可用的 WorkerID
        for (int id = 0; id <= MAX_WORKER_ID; id++) {
            String lockKey = REDIS_PREFIX + appName + ":lock:" + id;
            // SETNX 原子占位
            Boolean success = redisTemplate.opsForValue()
                .setIfAbsent(lockKey, hostName, EXPIRE_SECONDS, TimeUnit.SECONDS);
            if (Boolean.TRUE.equals(success)) {
                // 记录主机 → ID 映射
                redisTemplate.opsForValue().set(existingKey,
                    String.valueOf(id), EXPIRE_SECONDS, TimeUnit.SECONDS);
                return id;
            }
        }
        throw new RuntimeException("所有 WorkerID（0~" + MAX_WORKER_ID + "）已被占用！");
    }

    private void renewLease() {
        try {
            String hostName  = getHostIdentifier();
            String lockKey   = REDIS_PREFIX + appName + ":lock:" + workerId;
            String hostKey   = REDIS_PREFIX + appName + ":" + hostName;
            redisTemplate.expire(lockKey, EXPIRE_SECONDS, TimeUnit.SECONDS);
            redisTemplate.expire(hostKey, EXPIRE_SECONDS, TimeUnit.SECONDS);
        } catch (Exception e) {
            log.error("WorkerID 租约续期失败", e);
        }
    }

    @Override
    public void destroy() {
        // 应用下线时主动释放 WorkerID
        if (workerId >= 0) {
            String hostName = getHostIdentifier();
            redisTemplate.delete(REDIS_PREFIX + appName + ":lock:" + workerId);
            redisTemplate.delete(REDIS_PREFIX + appName + ":" + hostName);
            log.info("WorkerID {} 已释放", workerId);
        }
        if (renewExecutor != null) {
            renewExecutor.shutdown();
        }
    }

    private String getHostIdentifier() {
        try {
            return java.net.InetAddress.getLocalHost().getHostAddress()
                + ":" + System.getProperty("server.port", "8080");
        } catch (Exception e) {
            return "unknown-" + System.currentTimeMillis();
        }
    }

    public long getWorkerId() { return workerId; }
}
```

---

## 6.3 UUID（不推荐原因）

```
UUID 示例：550e8400-e29b-41d4-a716-446655440000

优点：
  实现极简，无需注册中心
  全局唯一（碰撞概率极低）

致命缺点：
  1. 字符串类型，36字节，比 bigint(8字节) 大4.5倍
     → 索引更大，占用更多 buffer pool

  2. 无序（随机）
     → 每次 INSERT 都可能导致 B+ 树页分裂
     → 大量随机 I/O，性能极差！
     → 在高并发写入时，比雪花算法慢 10~100 倍

  3. 可读性差（调试困难）

结论：分库分表场景禁止使用 UUID 作为主键！
      只有在极少写入且无排序需求的场景才考虑使用
```

---

## 6.4 NanoID

ShardingSphere 5.3+ 内置 NanoID 支持：

```yaml
key-generators:
  nanoid_gen:
    type: NANOID
    # 生成21字符的随机字符串
    # 示例：V1StGXR8_Z5jdHi6B-myT

# NanoID vs UUID：
#   NanoID：21字符，URL友好，可自定义字母表
#   UUID：  36字符，包含连字符

# 缺点：仍然是字符串，无序，不推荐作为MySQL主键
# 适合：文件名、短链接等场景
```

---

## 6.5 自定义主键生成器（号段模式）

```java
package com.example.sharding.id;

import org.apache.shardingsphere.sharding.spi.KeyGenerateAlgorithm;

import java.util.Properties;
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

/**
 * 号段模式主键生成器
 *
 * 原理：
 *   批量从数据库/Redis申请号段（如每次申请1000个）
 *   本地消费完再申请下一段
 *   双号段预加载（当前段消耗到80%时，异步预加载下一段）
 *
 * 性能：
 *   99% 的 ID 生成在本地内存完成（无网络IO）
 *   极低延迟，极高吞吐
 */
public class SegmentKeyGenerateAlgorithm
        implements KeyGenerateAlgorithm {

    private static final int SEGMENT_SIZE    = 1000;
    private static final double PRELOAD_RATIO = 0.8; // 消耗到80%时预加载

    // 双 Buffer（当前段 + 预备段）
    private volatile Segment current;
    private volatile Segment next;
    private volatile boolean nextLoading = false;

    private SegmentLoader loader;
    private final ExecutorService asyncLoader = Executors.newSingleThreadExecutor();

    @Override
    public synchronized Comparable<?> generateKey() {
        if (current == null || current.isExhausted()) {
            // 当前段耗尽，切换到预备段
            if (next != null) {
                current = next;
                next = null;
                nextLoading = false;
            } else {
                // 同步加载（首次或预加载失败）
                current = loader.loadSegment();
            }
        }

        long id = current.next();

        // 检查是否需要异步预加载
        if (!nextLoading && current.usageRatio() >= PRELOAD_RATIO) {
            nextLoading = true;
            asyncLoader.submit(() -> {
                try {
                    next = loader.loadSegment();
                } catch (Exception e) {
                    nextLoading = false;
                }
            });
        }

        return id;
    }

    @Override
    public void init(Properties props) {
        // 从 props 获取数据库连接信息，初始化 loader
        String jdbcUrl  = props.getProperty("jdbc-url");
        String username = props.getProperty("username");
        String password = props.getProperty("password");
        String keyName  = props.getProperty("key-name", "default");
        this.loader = new DatabaseSegmentLoader(jdbcUrl, username, password, keyName);
    }

    @Override
    public String getType() { return "SEGMENT"; }

    @Override
    public Properties getProps() { return new Properties(); }

    // 号段
    private static class Segment {
        private final long min;
        private final long max;
        private final AtomicLong current;

        Segment(long min, long max) {
            this.min     = min;
            this.max     = max;
            this.current = new AtomicLong(min);
        }

        long next() {
            long id = current.getAndIncrement();
            if (id > max) throw new RuntimeException("号段已耗尽");
            return id;
        }

        boolean isExhausted() { return current.get() > max; }

        double usageRatio() {
            return (double)(current.get() - min) / (max - min + 1);
        }
    }

    // 号段加载器（从数据库加载）
    private static class DatabaseSegmentLoader implements SegmentLoader {
        // 数据库表：
        // CREATE TABLE id_generator (
        //   key_name   VARCHAR(50) PRIMARY KEY,
        //   max_id     BIGINT NOT NULL DEFAULT 0,
        //   step       INT    NOT NULL DEFAULT 1000,
        //   version    INT    NOT NULL DEFAULT 0    -- 乐观锁
        // );
        private final String jdbcUrl, username, password, keyName;

        DatabaseSegmentLoader(String jdbcUrl, String username,
                               String password, String keyName) {
            this.jdbcUrl  = jdbcUrl;
            this.username = username;
            this.password = password;
            this.keyName  = keyName;
        }

        @Override
        public Segment loadSegment() {
            // 使用 CAS 乐观锁更新号段
            // UPDATE id_generator
            // SET max_id = max_id + step, version = version + 1
            // WHERE key_name = ? AND version = ?
            // 重试直到成功
            try (java.sql.Connection conn = java.sql.DriverManager
                    .getConnection(jdbcUrl, username, password)) {

                int maxRetry = 5;
                for (int i = 0; i < maxRetry; i++) {
                    // 查询当前号段状态
                    try (java.sql.PreparedStatement query = conn.prepareStatement(
                            "SELECT max_id, step, version FROM id_generator WHERE key_name = ?")) {
                        query.setString(1, keyName);
                        try (java.sql.ResultSet rs = query.executeQuery()) {
                            if (!rs.next()) {
                                throw new RuntimeException("key_name 不存在: " + keyName);
                            }
                            long   maxId   = rs.getLong("max_id");
                            int    step    = rs.getInt("step");
                            int    version = rs.getInt("version");

                            // CAS 更新
                            try (java.sql.PreparedStatement update = conn.prepareStatement(
                                    "UPDATE id_generator SET max_id=max_id+?,version=version+1 " +
                                    "WHERE key_name=? AND version=?")) {
                                update.setInt(1, step);
                                update.setString(2, keyName);
                                update.setInt(3, version);
                                if (update.executeUpdate() > 0) {
                                    // 更新成功，返回号段
                                    return new Segment(maxId + 1, maxId + step);
                                }
                            }
                        }
                    }
                    // CAS 失败，短暂等待后重试
                    Thread.sleep(10 * (i + 1));
                }
                throw new RuntimeException("获取号段失败，超过最大重试次数");
            } catch (Exception e) {
                throw new RuntimeException("加载号段异常", e);
            }
        }
    }

    @FunctionalInterface
    interface SegmentLoader {
        Segment loadSegment();
    }
}
```

---

---

# Part 7: 分布式事务 {#part-7}

## 7.1 分布式事务问题背景

```
CAP 定理：分布式系统不可能同时满足以下三点
  C（一致性）：所有节点在同一时刻看到相同数据
  A（可用性）：每个请求都能得到响应（不保证最新数据）
  P（分区容错）：网络分区时系统仍能运行

分库分表后，必须要分区容错（P），所以：
  CA → CP（放弃部分可用性，保证强一致）→ XA 事务
  CA → AP（放弃强一致，保证最终一致）  → Seata AT / 消息队列

三种分布式事务模型：
  ACID（强一致）：XA 协议，两阶段提交
  BASE（最终一致）：
    B（Basically Available）：基本可用
    S（Soft state）：软状态（允许中间态）
    E（Eventually Consistent）：最终一致

ShardingSphere 支持的事务类型：
  LOCAL → 单库本地事务（不跨数据源）
  XA    → 两阶段提交（强一致）
  BASE  → Seata AT（最终一致）
```

---

## 7.2 LOCAL 事务

```java
/**
 * LOCAL 事务：依赖各数据源本地事务
 * 适用：SQL 只路由到单个数据源
 */
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private OrderItemMapper orderItemMapper;

    /**
     * 场景：下单 + 明细 都路由到同一分片
     * user_id = 1001 → ds_1
     * order_id = xxx → t_order_1（具体分表无所谓，都在 ds_1）
     * order_item_id → t_order_item_1（绑定表，也在 ds_1）
     *
     * 结果：所有操作在 ds_1 单数据源内，LOCAL 事务有效
     */
    @Transactional
    // 等价于 @ShardingTransactionType(TransactionType.LOCAL)
    public Order createOrder(OrderDTO dto) {
        // 插入订单（路由到 ds_1.t_order_1）
        Order order = new Order();
        order.setUserId(dto.getUserId());
        order.setAmount(dto.getAmount());
        orderMapper.insert(order); // ShardingSphere 自动生成 order_id

        // 插入订单明细（绑定表，路由到 ds_1.t_order_item_1）
        for (OrderItemDTO item : dto.getItems()) {
            OrderItem orderItem = new OrderItem();
            orderItem.setOrderId(order.getOrderId());
            orderItem.setUserId(dto.getUserId()); // 分片键
            orderItem.setGoodsId(item.getGoodsId());
            orderItem.setQuantity(item.getQuantity());
            orderItemMapper.insert(orderItem);
        }

        // 模拟异常：事务回滚
        // if (dto.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
        //     throw new IllegalArgumentException("金额不合法");
        //     // orderMapper 和 orderItemMapper 的操作都会回滚
        // }

        return order;
    }

    /**
     * LOCAL 事务的限制演示
     * 如果 user_id=1001 路由到 ds_1，
     *    user_id=1002 路由到 ds_0，
     * 则两个插入分属不同数据源，LOCAL 事务无法保证原子性！
     */
    @Transactional
    public void batchInsertToDifferentDS(Long userId1, Long userId2) {
        // ds_1.t_order_1
        Order order1 = new Order();
        order1.setUserId(userId1);
        orderMapper.insert(order1);

        // ds_0.t_order_0（如果 userId2 % 4 = 0）
        Order order2 = new Order();
        order2.setUserId(userId2);
        orderMapper.insert(order2);

        // 如果第二条插入后抛异常：
        // ds_1 的操作可以回滚（同一连接）
        // ds_0 的操作是另一个连接，无法保证也回滚！
        // 这是 LOCAL 事务的不足，需要 XA 或 BASE
    }
}
```

---

## 7.3 XA 两阶段提交

```
XA 协议流程图：

                协调者（ShardingSphere）
                        |
          +-------------+-------------+
          |                           |
          ↓                           ↓
    参与者 ds_0                参与者 ds_1
  （MySQL XA 实现）          （MySQL XA 实现）

=== 第一阶段（Prepare）===

协调者 → ds_0: "XA START 'tx-1-0'; SQL...; XA END; XA PREPARE;"
协调者 → ds_1: "XA START 'tx-1-1'; SQL...; XA END; XA PREPARE;"

ds_0 → 协调者: "OK，redo log 已写入，资源已锁定"
ds_1 → 协调者: "OK，redo log 已写入，资源已锁定"

=== 第二阶段（Commit）===

如果所有参与者都 Prepare 成功：
  协调者 → ds_0: "XA COMMIT 'tx-1-0';"
  协调者 → ds_1: "XA COMMIT 'tx-1-1';"

如果任意参与者 Prepare 失败：
  协调者 → ds_0: "XA ROLLBACK 'tx-1-0';"
  协调者 → ds_1: "XA ROLLBACK 'tx-1-1';"

XA 核心保证：
  Prepare 后，参与者必须能提交（除非 Rollback）
  协调者崩溃后，参与者会一直持锁等待恢复
  → 实际上存在"悬挂事务"问题（需要 TM 持久化+恢复）
```

**XA 事务集成：**

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-xa-core</artifactId>
    <version>5.4.1</version>
</dependency>

<!-- 选择一个 XA 事务管理器 -->
<!-- 方案1：Atomikos（开源，轻量） -->
<dependency>
    <groupId>com.atomikos</groupId>
    <artifactId>transactions-jdbc</artifactId>
    <version>5.0.9</version>
</dependency>
<dependency>
    <groupId>com.atomikos</groupId>
    <artifactId>transactions-jta</artifactId>
    <version>5.0.9</version>
</dependency>

<!-- 方案2：Narayana（JBoss，功能最强） -->
<!-- <dependency>
    <groupId>org.jboss.narayana.jta</groupId>
    <artifactId>narayana-jta</artifactId>
    <version>5.13.0.Final</version>
</dependency> -->
```

```yaml
# application.yml
spring:
  shardingsphere:
    rules:
      transaction:
        default-type: XA
        provider-type: Atomikos   # 或 Narayana

# Atomikos 配置
com:
  atomikos:
    icatch:
      log-base-dir: ./logs/atomikos   # 事务日志目录（必须持久化！）
      max-timeout: 300000             # 事务超时 5分钟
      default-jta-timeout: 10000      # 默认 JTA 超时 10秒
```

```java
@Service
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private StockMapper stockMapper;

    /**
     * XA 强一致性事务
     * 即使 orderMapper 和 stockMapper 路由到不同数据源
     * 两步操作也能保证原子性（两阶段提交）
     */
    @ShardingTransactionType(TransactionType.XA)
    @Transactional(rollbackFor = Exception.class)
    public void createOrderXA(OrderDTO dto) {
        // 步骤1：创建订单（可能路由到 ds_1）
        Order order = buildOrder(dto);
        orderMapper.insert(order);

        // 步骤2：扣减库存（库存可能在 ds_0 的 t_stock 表）
        int affected = stockMapper.deductStock(
            dto.getGoodsId(), dto.getQuantity()
        );
        if (affected == 0) {
            throw new RuntimeException("库存不足！");
            // 此异常导致：
            // ds_1 的 order 插入回滚
            // ds_0 的 stock 扣减回滚
            // 两步操作同时回滚！
        }

        // 步骤3：记录支付流水（可能路由到第3个数据源）
        // payMapper.insert(buildPay(order));
        // 同样在 XA 事务保护下
    }
}
```

---

## 7.4 BASE 事务（Seata AT 模式）

Seata AT（Automatic Transaction）是阿里开源的分布式事务中间件，基于 undo log 补偿。

```
Seata 架构：

  TC  (Transaction Coordinator) - 事务协调器
      独立部署的 Seata Server
      负责全局事务的开始/提交/回滚

  TM  (Transaction Manager) - 事务管理器
      集成在业务代码中（@GlobalTransactional 注解）
      向 TC 开启/提交/回滚全局事务

  RM  (Resource Manager) - 资源管理器
      集成在 JDBC 驱动层（Seata 代理 DataSource）
      向 TC 注册分支事务，执行 undo log 回滚

AT 模式第一阶段（提交本地事务）：
  RM 拦截 SQL 执行
  → 解析 SQL，查询执行前镜像（before image）
  → 执行业务 SQL
  → 查询执行后镜像（after image）
  → 生成 undo log（before → after 的逆向操作）
  → 提交本地事务（包含业务数据 + undo log）
  → 向 TC 报告：分支事务已完成

AT 模式第二阶段（全局提交）：
  TC 收到所有分支事务提交报告
  → 通知各 RM 删除 undo log（清理）
  → 全局事务完成

AT 模式第二阶段（全局回滚）：
  某分支事务失败，TM 通知 TC 回滚
  → TC 通知各 RM 根据 undo log 执行补偿
  → RM 执行逆向 SQL（将 after image 恢复为 before image）
  → 全局事务回滚

核心优势 vs XA：
  XA：第一阶段持有锁 → 高并发下锁竞争严重
  AT：第一阶段提交本地事务，锁立即释放 → 并发性好
  
  代价：AT 属于乐观锁模型
        在全局事务提交期间，数据可能被其他事务读到（脏读）
        通过"全局锁"机制解决（但全局锁也会竞争）
```

**Seata 集成步骤：**

**第一步：部署 Seata Server**
```bash
# Docker 部署 Seata Server
docker run -d \
  --name seata-server \
  -p 8091:8091 \
  -e SEATA_IP=your_server_ip \
  -e SEATA_PORT=8091 \
  -e STORE_MODE=db \
  -e SEATA_DB_URL=jdbc:mysql://mysql:3306/seata \
  -e SEATA_DB_USER=root \
  -e SEATA_DB_PASSWORD=root123 \
  seataio/seata-server:1.7.0
```

**第二步：建 undo_log 表（每个业务数据库）**
```sql
-- 在每个业务数据库中执行
CREATE TABLE `undo_log` (
  `branch_id`     BIGINT(20) NOT NULL COMMENT '分支事务ID',
  `xid`           VARCHAR(100) NOT NULL COMMENT '全局事务ID',
  `context`       VARCHAR(128) NOT NULL COMMENT '上下文',
  `rollback_info` LONGBLOB NOT NULL COMMENT '回滚信息',
  `log_status`    INT(11) NOT NULL COMMENT '0:正常，1:防御',
  `log_created`   DATETIME(6) NOT NULL COMMENT '创建时间',
  `log_modified`  DATETIME(6) NOT NULL COMMENT '修改时间',
  UNIQUE KEY `ux_undo_log` (`xid`, `branch_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

**第三步：引入依赖**
```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-transaction-base-seata-at</artifactId>
    <version>5.4.1</version>
</dependency>
<dependency>
    <groupId>io.seata</groupId>
    <artifactId>seata-spring-boot-starter</artifactId>
    <version>1.7.0</version>
</dependency>
```

**第四步：配置**
```yaml
# application.yml
seata:
  enabled: true
  application-id: order-service
  tx-service-group: order_tx_group   # 事务分组名

  # TC 地址（Seata Server）
  registry:
    type: nacos
    nacos:
      application: seata-server
      server-addr: nacos:8848
      namespace: ""
      group: SEATA_GROUP

  config:
    type: nacos
    nacos:
      server-addr: nacos:8848
      namespace: ""
      group: SEATA_GROUP
      data-id: seataServer.properties

spring:
  shardingsphere:
    rules:
      transaction:
        default-type: BASE
        provider-type: Seata
```

**第五步：业务代码**
```java
@Service
@Slf4j
public class OrderFacadeService {

    @Autowired
    private OrderMapper orderMapper;     // order_db（分片）

    @Autowired
    private StockService stockService;   // goods_db（远程调用）

    @Autowired
    private AccountService accountService; // user_db（远程调用）

    /**
     * 下单（跨服务、跨数据库的分布式事务）
     *
     * @GlobalTransactional 由 Seata TM 开启全局事务
     * XID 通过 RPC 上下文传播（Feign/Dubbo 自动传递）
     */
    @GlobalTransactional(
        name          = "order-create-tx",
        timeoutMills  = 60000,
        rollbackFor   = Exception.class
    )
    @ShardingTransactionType(TransactionType.BASE)
    public OrderVO createOrder(CreateOrderCmd cmd) {
        log.info("[Seata] 全局事务开始，XID={}", RootContext.getXID());

        try {
            // 1. 创建订单（本地 DB 操作，Seata RM 拦截生成 undo log）
            Order order = Order.create(cmd);
            orderMapper.insert(order);
            log.info("[Seata] 分支事务1：订单写入 order_id={}", order.getOrderId());

            // 2. 扣减库存（调用库存服务，XID 自动通过 Feign Header 传递）
            stockService.deductStock(
                new DeductStockReq(cmd.getGoodsId(), cmd.getQuantity())
            );
            log.info("[Seata] 分支事务2：库存扣减 goodsId={}", cmd.getGoodsId());

            // 3. 扣减余额（调用账户服务）
            accountService.deductBalance(
                new DeductBalanceReq(cmd.getUserId(), cmd.getPayAmount())
            );
            log.info("[Seata] 分支事务3：余额扣减 userId={}", cmd.getUserId());

            // 全部成功 → TM 通知 TC 全局提交
            // TC 通知各 RM 删除 undo log
            log.info("[Seata] 全局事务提交成功");
            return OrderVO.from(order);

        } catch (Exception e) {
            // 任何异常 → TM 通知 TC 全局回滚
            // TC 通知各 RM 使用 undo log 补偿
            log.error("[Seata] 全局事务回滚，原因: {}", e.getMessage());
            throw e; // 必须重新抛出，让 @GlobalTransactional 感知
        }
    }
}
```

---

## 7.5 三种事务方案对比

```
+----------+-----------+-----------+-----------+----------------------+
| 模式      | 一致性    | 性能      | 适用场景   | 主要限制              |
+----------+-----------+-----------+-----------+----------------------+
| LOCAL    | 强一致    | 最高      | 单分片操作 | 不支持跨数据源        |
|          | ACID      |           |            |                      |
+----------+-----------+-----------+-----------+----------------------+
| XA       | 强一致    | 中等      | 金融类     | 持锁时间长            |
|          | ACID      | (持锁)    | 强一致场景 | 高并发性能差          |
|          |           |           |            | 悬挂事务问题          |
+----------+-----------+-----------+-----------+----------------------+
| BASE     | 最终一致  | 高        | 大多数     | 短暂不一致            |
| (Seata)  |           |           | 互联网业务 | 需部署 Seata Server   |
|          |           |           |            | undo log 存储成本     |
+----------+-----------+-----------+-----------+----------------------+

选型建议：
  ① 单分片操作（用户ID分片，同用户的所有操作）
     → LOCAL 事务（性能最好）

  ② 跨分片操作，金融强一致要求（支付、转账）
     → XA + Atomikos，并控制事务粒度

  ③ 跨服务分布式事务，高并发业务（下单、退款）
     → Seata AT，最优平衡点

  ④ 极致性能，允许最终一致（日志写入、通知发送）
     → 本地消息表 + 消息队列（不依赖 ShardingSphere 事务）
```

---

---

# Part 8: 数据加密 {#part-8}

## 8.1 加密功能概述

ShardingSphere 数据加密实现了**对应用透明**的自动加解密：应用层使用明文，底层自动加密存储，查询时自动解密返回。

```
数据加密流程：

写入流程（INSERT）：
  应用 INSERT INTO t_user (phone) VALUES ('13800138000')
       ↓ ShardingSphere 拦截
       ↓ 加密处理
  实际 INSERT INTO t_user
         (phone_cipher,   phone_plain, phone_assisted)
       VALUES
         ('AES("13800138000")', '138****0000', 'MD5("13800138000")')
       ↓ 存储到数据库

物理表实际数据：
+----+---------------------+-------------+------------------+
| id | phone_cipher        | phone_plain | phone_assisted   |
+----+---------------------+-------------+------------------+
|  1 | XBz9kLq3mP...==     | 138****0000 | a1b2c3d4e5f6...  |
+----+---------------------+-------------+------------------+

查询流程（SELECT）：
  应用 SELECT phone FROM t_user WHERE id = 1
       ↓ ShardingSphere 改写
  实际 SELECT phone_cipher FROM t_user WHERE id = 1
       ↓ 解密 phone_cipher
  返回 '13800138000'（明文）

条件查询（使用辅助查询列）：
  应用 SELECT * FROM t_user WHERE phone = '13800138000'
       ↓ ShardingSphere 改写
  实际 SELECT * FROM t_user WHERE phone_assisted = MD5('13800138000')
       ↓ 用 phone_assisted 精确匹配
       ↓ 结果集中 phone_cipher 解密后返回
```

---

## 8.2 数据库物理表设计

```sql
-- 逻辑表（应用层看到的，不实际存在）
-- 字段：id, username, phone, id_card

-- 物理表（实际建的表）
CREATE TABLE t_user (
    id                 BIGINT          NOT NULL COMMENT '用户ID',
    username           VARCHAR(50)     NOT NULL COMMENT '用户名（不加密）',
    phone_plain        VARCHAR(20)     DEFAULT NULL COMMENT '手机号脱敏明文（138****0000）',
    phone_cipher       VARCHAR(200)    NOT NULL COMMENT '手机号密文（AES加密）',
    phone_assisted     VARCHAR(64)     DEFAULT NULL COMMENT '手机号辅助查询（HMAC，用于= 查询）',
    id_card_plain      VARCHAR(20)     DEFAULT NULL COMMENT '身份证脱敏明文（1101***08）',
    id_card_cipher     VARCHAR(200)    NOT NULL COMMENT '身份证密文（AES加密）',
    create_time        DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id),
    KEY idx_phone_assisted (phone_assisted)   -- 辅助查询列建索引
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

-- 注意：
-- phone        是逻辑列（只存在于 ShardingSphere 配置中）
-- phone_cipher 是密文列（AES 加密，可解密）
-- phone_plain  是脱敏明文列（可选，方便人工查看）
-- phone_assisted 是辅助查询列（确定性加密，用于 = 精确查询）
```

---

## 8.3 完整加密配置

```yaml
spring:
  shardingsphere:
    rules:
      encrypt:
        tables:
          t_user:
            columns:
              # ===== 手机号加密 =====
              phone:
                # 密文列（AES 加密存储，可解密）
                cipher-column: phone_cipher
                # 明文脱敏列（可选，138****0000，方便运维排查）
                plain-column: phone_plain
                # 辅助查询列（用 HMAC-SHA256 生成，固定密钥，用于 = 查询）
                assisted-query-column: phone_assisted
                # 密文加密算法
                encrypt-algorithm-name: phone_aes
                # 辅助查询加密算法（与密文算法不同）
                assisted-query-encrypt-algorithm-name: phone_hmac
                # 是否用密文列查询（true = 走 cipher，false = 走 plain）
                query-with-cipher-column: true

              # ===== 身份证加密 =====
              id_card:
                cipher-column: id_card_cipher
                plain-column: id_card_plain
                assisted-query-column: id_card_assisted
                encrypt-algorithm-name: id_card_aes
                assisted-query-encrypt-algorithm-name: id_card_hmac
                query-with-cipher-column: true

        # ===== 加密算法定义 =====
        encryptors:
          # AES 对称加密（可解密，适合需要看明文的场景）
          phone_aes:
            type: AES
            props:
              # 生产环境：从 Vault/配置中心读取，不写死
              aes-key-value: "PhoneKey@SecureXXXXXX16B"

          # AES 身份证加密
          id_card_aes:
            type: AES
            props:
              aes-key-value: "IdCardKey@Secure16BB"

          # HMAC 辅助查询（确定性：相同明文→相同密文，用于 = 查询）
          phone_hmac:
            type: MD5
            # 注意：MD5 是不可逆的，只能用于精确匹配，不能解密

          id_card_hmac:
            type: MD5
    props:
      sql-show: true
```

---

## 8.4 Java 代码完整示例

```java
// ===== Entity（使用逻辑列名）=====
@Data
@TableName("t_user")
public class User {

    @TableId(type = IdType.INPUT)
    private Long id;

    private String username;

    /**
     * 使用逻辑列名 "phone"，不是物理列名 "phone_cipher"
     * ShardingSphere 自动映射到 phone_cipher
     */
    private String phone;

    /**
     * 使用逻辑列名 "id_card"
     */
    private String idCard;

    private LocalDateTime createTime;
}

// ===== Mapper =====
@Mapper
public interface UserMapper extends BaseMapper<User> {

    /**
     * 精确查询：WHERE phone = '13800138000'
     * ShardingSphere 自动改写为：
     *   WHERE phone_assisted = MD5('13800138000')
     */
    @Select("SELECT * FROM t_user WHERE phone = #{phone}")
    User findByPhone(@Param("phone") String phone);

    /**
     * 模糊查询：WHERE phone LIKE '138%'
     * 注意：模糊查询无法用辅助查询列！
     * ShardingSphere 无法改写，只能：
     *   方案1：在 phone_plain（脱敏明文）上模糊查询
     *   方案2：用 ES 做全文检索
     */
    @Select("SELECT * FROM t_user WHERE phone_plain LIKE #{phoneLike}")
    List<User> findByPhoneLike(@Param("phoneLike") String phoneLike);
}

// ===== Service =====
@Service
@Slf4j
public class UserService {

    @Autowired
    private UserMapper userMapper;

    /**
     * 注册用户（手机号和身份证自动加密）
     */
    public User register(UserRegisterDTO dto) {
        // 检查手机号是否已注册
        User existing = userMapper.findByPhone(dto.getPhone());
        if (existing != null) {
            throw new BizException("手机号已注册");
        }

        User user = new User();
        user.setId(generateId()); // 雪花算法
        user.setUsername(dto.getUsername());

        // 写入明文，ShardingSphere 自动加密
        // 实际存储：
        //   phone_cipher    = AES('13800138000')
        //   phone_plain     = '138****0000'（脱敏）
        //   phone_assisted  = MD5('13800138000')
        user.setPhone(dto.getPhone());

        // 同样自动加密
        user.setIdCard(dto.getIdCard());

        userMapper.insert(user);
        log.info("用户注册成功，userId={}", user.getId());

        return user;
    }

    /**
     * 查询用户（返回解密后的明文）
     */
    public User findById(Long id) {
        User user = userMapper.selectById(id);
        if (user != null) {
            // user.getPhone() 返回解密后的明文：13800138000
            // ShardingSphere 自动从 phone_cipher 解密
            log.info("查询用户，phone={}", maskPhone(user.getPhone()));
        }
        return user;
    }

    /**
     * 按手机号精确查找（走辅助查询列）
     */
    public User findByPhone(String phone) {
        // SELECT * FROM t_user WHERE phone = '13800138000'
        // → 改写为 WHERE phone_assisted = MD5('13800138000')
        // → 结果集中 phone_cipher 解密后填充到 phone 字段
        return userMapper.findByPhone(phone);
    }

    /**
     * 手机号脱敏（仅显示前3后4）
     */
    private String maskPhone(String phone) {
        if (phone == null || phone.length() < 7) return "***";
        return phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
    }
}
```

---

## 8.5 加密算法说明

```
ShardingSphere 内置加密算法：

1. AES（Advanced Encryption Standard）
   类型：对称加密，可逆
   密钥长度：128/192/256 bit
   适用：手机号、身份证、银行卡号（需要解密看明文）

   配置：
     type: AES
     props:
       aes-key-value: "16字节/24字节/32字节密钥"

2. MD5（Message Digest 5）
   类型：哈希，不可逆
   适用：密码存储（不需要解密）、辅助查询列

   配置：
     type: MD5

3. RC4（已不推荐）
   类型：流加密，已有安全漏洞
   不建议生产使用

4. SM4（国密算法，ShardingSphere 5.3+）
   类型：中国商用密码算法，类似AES
   适用：金融合规场景（国密要求）

   配置：
     type: SM4
     props:
       sm4-key-value: "16字节密钥"
       sm4-mode: ECB  # 或 CBC
       sm4-padding: PKCS5Padding

自定义加密算法：
  实现 EncryptAlgorithm 接口
  注册 SPI：META-INF/services/org.apache.shardingsphere.encrypt.spi.EncryptAlgorithm
```

---

# Part 9: 影子库与全链路压测 {#part-9}

## 9.1 影子库（Shadow Database）原理

```
传统压测的问题：
  直接对生产环境压测
  → 产生大量脏数据（测试订单、测试用户）
  → 污染真实业务数据
  → 压测后清理困难，容易误删生产数据

影子库解决方案：
  生产流量   ──→ 生产数据库（真实数据）
  压测流量   ──→ 影子数据库（测试数据，完全隔离）

识别压测流量的方式：
  1. 请求头标记：X-Stress-Test: true
  2. SQL 注释标记：/* shadow:true */
  3. 特殊用户ID（如 ID > 9000000000）

隔离粒度：
  ShardingSphere 在 SQL 执行时判断
  → 生产请求 → order_db     → t_order
  → 压测请求 → order_db_shadow → t_order（影子库中的同名表）
```

## 9.2 影子库配置详解

```yaml
spring:
  shardingsphere:
    datasource:
      names: ds_0,ds_0_shadow
      ds_0:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://prod-db:3306/order_db?useSSL=false
        username: root
        password: prod@123
      # 影子库数据源（结构与生产库相同）
      ds_0_shadow:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://shadow-db:3306/order_db_shadow?useSSL=false
        username: root
        password: shadow@123

    rules:
      shadow:
        # 数据源映射：生产 → 影子
        data-sources:
          shadow-data-source:
            source-data-source-name: ds_0         # 生产库
            shadow-data-source-name: ds_0_shadow   # 影子库

        # 哪些表需要影子路由
        tables:
          t_order:
            data-source-names:
              - shadow-data-source
            shadow-algorithm-names:
              - user-id-shadow-algo       # 按用户ID识别
              - sql-hint-shadow-algo      # 按SQL注释识别
          t_order_item:
            data-source-names:
              - shadow-data-source
            shadow-algorithm-names:
              - user-id-shadow-algo

        # 影子算法
        shadow-algorithms:
          # 算法1：用户ID匹配（压测用户ID特征）
          user-id-shadow-algo:
            type: VALUE_MATCH
            props:
              operation: insert          # 作用于 INSERT
              column: user_id            # 判断字段
              value: "9999999999"        # 压测用户ID（特殊值）

          # 算法2：SQL 注释标记
          sql-hint-shadow-algo:
            type: SQL_HINT
            props:
              shadow: true               # SQL 中包含 /*shadow:true*/ 走影子库
```

## 9.3 全链路压测请求标记传播

```java
/**
 * Spring MVC 拦截器：识别压测请求，设置影子标记
 */
@Component
public class ShadowTrafficInterceptor implements HandlerInterceptor {

    // 压测标记的 HTTP Header
    private static final String STRESS_TEST_HEADER = "X-Stress-Test";

    // 线程本地变量，标识当前线程是否为压测请求
    public static final ThreadLocal<Boolean> IS_STRESS_TEST =
        ThreadLocal.withInitial(() -> false);

    @Override
    public boolean preHandle(HttpServletRequest request,
                              HttpServletResponse response,
                              Object handler) {
        String stressTest = request.getHeader(STRESS_TEST_HEADER);
        boolean isStress  = "true".equalsIgnoreCase(stressTest);

        IS_STRESS_TEST.set(isStress);

        if (isStress) {
            // 设置 ShardingSphere 影子 Hint
            HintManager.getInstance().setShadow();
        }
        return true;
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                 HttpServletResponse response,
                                 Object handler, Exception ex) {
        // 必须清理，防止内存泄漏 和 线程复用时的污染
        IS_STRESS_TEST.remove();
        HintManager.clear();
    }
}

/**
 * Feign 拦截器：微服务间传播压测标记
 */
@Component
public class ShadowFeignInterceptor implements RequestInterceptor {

    @Override
    public void apply(RequestTemplate template) {
        if (Boolean.TRUE.equals(ShadowTrafficInterceptor.IS_STRESS_TEST.get())) {
            template.header("X-Stress-Test", "true");
        }
    }
}
```

## 9.4 全链路压测整体方案

```
全链路压测架构：

压测平台（JMeter/PTS/Gatling）
  |
  | 携带压测标记：HTTP Header: X-Stress-Test: true
  ↓
Nginx / API Gateway
  |
  | 透传压测标记（HEADER COPY）
  ↓
订单服务（Order Service）
  ├── ShardingSphere 识别影子请求
  ├── SQL → 影子库 order_db_shadow
  ├── Redis → 影子 Redis（前缀区分）
  └── Feign → 库存服务（携带压测 Header）
       ↓
     库存服务（Stock Service）
       ├── SQL → 影子库 goods_db_shadow
       └── Feign → 账户服务（携带压测 Header）
            ↓
          账户服务（Account Service）
            └── SQL → 影子库 user_db_shadow

全链路隔离要点：
  1. MySQL    → ShardingSphere 影子库
  2. Redis    → 压测 key 加前缀（shadow:xxx）
  3. MQ       → 影子 Topic（order-topic-shadow）
  4. 三方服务  → Mock Server（避免产生真实费用）
  5. ES       → 影子索引（order-shadow）

压测结束后：
  生产库：数据完全干净，无压测痕迹
  影子库：可以随时清空，不影响生产
```

---

---

# Part 10: Spring Boot 集成实战（完整案例） {#part-10}

## 10.1 案例说明

**场景：4库16表的订单系统**

```
业务规模：
  用户数：1000万+
  日订单量：500万+
  历史订单总量：50亿+

分片设计：
  分库键：user_id（4个库）
  分表键：order_id（每库4张表）
  共 4 × 4 = 16 个物理分片

分片规则：
  ds_x  = user_id  % 4 → ds_0, ds_1, ds_2, ds_3
  t_order_y = order_id % 4 → t_order_0, t_order_1, t_order_2, t_order_3

示例：
  user_id=1001, order_id=88888888
    库: 1001 % 4 = 1 → ds_1
    表: 88888888 % 4 = 0 → t_order_0
    → ds_1.t_order_0
```

## 10.2 完整 pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>sharding-demo</artifactId>
    <version>1.0.0</version>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.7.14</version>
    </parent>

    <properties>
        <java.version>11</java.version>
        <shardingsphere.version>5.4.1</shardingsphere.version>
        <mybatis-plus.version>3.5.3.1</mybatis-plus.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Web -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- MyBatis Plus -->
        <dependency>
            <groupId>com.baomidou</groupId>
            <artifactId>mybatis-plus-boot-starter</artifactId>
            <version>${mybatis-plus.version}</version>
        </dependency>

        <!-- ShardingSphere JDBC -->
        <dependency>
            <groupId>org.apache.shardingsphere</groupId>
            <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
            <version>${shardingsphere.version}</version>
        </dependency>

        <!-- MySQL Connector -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>

        <!-- HikariCP 连接池 -->
        <dependency>
            <groupId>com.zaxxer</groupId>
            <artifactId>HikariCP</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- 测试 -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

## 10.3 application.yml（完整配置）

```yaml
server:
  port: 8080

spring:
  main:
    allow-bean-definition-overriding: true

  shardingsphere:
    # ========== 数据源 ==========
    datasource:
      names: ds_0,ds_1,ds_2,ds_3

      ds_0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3306/order_db_0?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
        username: root
        password: root123
        hikari:
          minimum-idle: 5
          maximum-pool-size: 20
          connection-timeout: 30000
          idle-timeout: 600000
          max-lifetime: 1800000
          connection-test-query: SELECT 1

      ds_1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3307/order_db_1?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
        username: root
        password: root123
        hikari:
          minimum-idle: 5
          maximum-pool-size: 20
          connection-timeout: 30000

      ds_2:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3308/order_db_2?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
        username: root
        password: root123
        hikari:
          minimum-idle: 5
          maximum-pool-size: 20

      ds_3:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://127.0.0.1:3309/order_db_3?useSSL=false&serverTimezone=Asia/Shanghai&characterEncoding=utf8
        username: root
        password: root123
        hikari:
          minimum-idle: 5
          maximum-pool-size: 20

    # ========== 分片规则 ==========
    rules:
      sharding:
        tables:
          # 订单主表
          t_order:
            actual-data-nodes: ds_$->{0..3}.t_order_$->{0..3}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db_user_mod
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table_order_mod
            key-generate-strategy:
              column: order_id
              key-generator-name: snowflake_gen

          # 订单明细表（绑定表）
          t_order_item:
            actual-data-nodes: ds_$->{0..3}.t_order_item_$->{0..3}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db_user_mod
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table_order_mod
            key-generate-strategy:
              column: item_id
              key-generator-name: snowflake_gen

          # 用户表（只分库，不分表）
          t_user:
            actual-data-nodes: ds_$->{0..3}.t_user
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db_user_mod
            key-generate-strategy:
              column: user_id
              key-generator-name: snowflake_gen

        # 绑定表（t_order 和 t_order_item 使用相同分片键）
        binding-tables:
          - t_order,t_order_item

        # 广播表（每个分片库都有完整数据）
        broadcast-tables:
          - t_goods_category
          - t_sys_config

        # 分片算法
        sharding-algorithms:
          db_user_mod:
            type: MOD
            props:
              sharding-count: 4

          table_order_mod:
            type: MOD
            props:
              sharding-count: 4

        # 主键生成器
        key-generators:
          snowflake_gen:
            type: SNOWFLAKE
            props:
              worker-id: ${WORKER_ID:1}

    props:
      sql-show: true            # 开发环境打印SQL
      sql-simple: false
      executor-size: 20

mybatis-plus:
  mapper-locations: classpath:/mapper/**/*.xml
  type-aliases-package: com.example.sharding.entity
  configuration:
    map-underscore-to-camel-case: true
    default-fetch-size: 100
    default-statement-timeout: 30
  global-config:
    db-config:
      id-type: INPUT            # 主键由外部（ShardingSphere）生成

logging:
  level:
    com.example.sharding: DEBUG
    org.apache.shardingsphere: INFO
```

## 10.4 建表 SQL

```sql
-- ===== 脚本：创建4个数据库和所有分片表 =====

-- 创建数据库
CREATE DATABASE IF NOT EXISTS order_db_0 CHARACTER SET utf8mb4;
CREATE DATABASE IF NOT EXISTS order_db_1 CHARACTER SET utf8mb4;
CREATE DATABASE IF NOT EXISTS order_db_2 CHARACTER SET utf8mb4;
CREATE DATABASE IF NOT EXISTS order_db_3 CHARACTER SET utf8mb4;

-- ===== 在每个数据库（order_db_0 ~ order_db_3）执行以下建表 =====
-- 以 order_db_0 为例：

USE order_db_0;

-- 订单表（每库4张，t_order_0 ~ t_order_3）
CREATE TABLE IF NOT EXISTS t_order_0 (
    order_id    BIGINT          NOT NULL            COMMENT '订单ID（雪花算法）',
    user_id     BIGINT          NOT NULL            COMMENT '用户ID（分库键）',
    goods_id    BIGINT          NOT NULL            COMMENT '商品ID',
    quantity    INT             NOT NULL DEFAULT 1  COMMENT '购买数量',
    amount      DECIMAL(12,2)   NOT NULL            COMMENT '订单金额',
    status      TINYINT         NOT NULL DEFAULT 0
                                COMMENT '状态:0创建中,1待支付,2已支付,3已发货,4已完成,5已取消',
    address_id  BIGINT          DEFAULT NULL        COMMENT '收货地址ID',
    remark      VARCHAR(200)    DEFAULT NULL        COMMENT '备注',
    create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP  COMMENT '创建时间',
    update_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP
                                ON UPDATE CURRENT_TIMESTAMP         COMMENT '更新时间',
    PRIMARY KEY (order_id),
    KEY idx_user_id     (user_id),
    KEY idx_create_time (create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单表-分片0';

-- 复制3张同结构的表
CREATE TABLE IF NOT EXISTS t_order_1 LIKE t_order_0;
CREATE TABLE IF NOT EXISTS t_order_2 LIKE t_order_0;
CREATE TABLE IF NOT EXISTS t_order_3 LIKE t_order_0;

-- 修改每张表的注释
ALTER TABLE t_order_1 COMMENT='订单表-分片1';
ALTER TABLE t_order_2 COMMENT='订单表-分片2';
ALTER TABLE t_order_3 COMMENT='订单表-分片3';

-- 订单明细表（绑定表，每库4张）
CREATE TABLE IF NOT EXISTS t_order_item_0 (
    item_id     BIGINT          NOT NULL            COMMENT '明细ID（雪花算法）',
    order_id    BIGINT          NOT NULL            COMMENT '订单ID（分表键）',
    user_id     BIGINT          NOT NULL            COMMENT '用户ID（分库键）',
    goods_id    BIGINT          NOT NULL            COMMENT '商品ID',
    goods_name  VARCHAR(200)    NOT NULL            COMMENT '商品名（冗余）',
    price       DECIMAL(10,2)   NOT NULL            COMMENT '单价',
    quantity    INT             NOT NULL DEFAULT 1  COMMENT '数量',
    create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (item_id),
    KEY idx_order_id (order_id),
    KEY idx_user_id  (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='订单明细-分片0';

CREATE TABLE IF NOT EXISTS t_order_item_1 LIKE t_order_item_0;
CREATE TABLE IF NOT EXISTS t_order_item_2 LIKE t_order_item_0;
CREATE TABLE IF NOT EXISTS t_order_item_3 LIKE t_order_item_0;

-- 用户表（分库，不分表）
CREATE TABLE IF NOT EXISTS t_user (
    user_id     BIGINT          NOT NULL            COMMENT '用户ID',
    username    VARCHAR(50)     NOT NULL            COMMENT '用户名',
    phone       VARCHAR(20)     NOT NULL            COMMENT '手机号',
    status      TINYINT         NOT NULL DEFAULT 1  COMMENT '状态:1正常,2禁用',
    create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    update_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (user_id),
    UNIQUE KEY uk_phone (phone)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

-- 广播表（每个分片库都有完整数据）
CREATE TABLE IF NOT EXISTS t_goods_category (
    id          BIGINT          NOT NULL AUTO_INCREMENT,
    name        VARCHAR(50)     NOT NULL COMMENT '分类名称',
    parent_id   BIGINT          NOT NULL DEFAULT 0 COMMENT '父分类ID',
    sort        INT             NOT NULL DEFAULT 0 COMMENT '排序',
    create_time DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品分类（广播表）';

-- ===== 在 order_db_1, order_db_2, order_db_3 执行相同的建表语句 =====
-- （省略重复，实际脚本中需要分别执行）
```

## 10.5 Entity / Mapper / Service

```java
// ===== Order.java =====
package com.example.sharding.entity;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Data
@TableName("t_order")  // 逻辑表名
public class Order {

    @TableId(type = IdType.INPUT) // 主键由 ShardingSphere 雪花算法生成
    private Long orderId;

    private Long userId;
    private Long goodsId;
    private Integer quantity;
    private BigDecimal amount;
    private Integer status;   // 0创建中 1待支付 2已支付 3已发货 4已完成 5取消
    private Long addressId;
    private String remark;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;
}

// ===== OrderItem.java =====
@Data
@TableName("t_order_item")
public class OrderItem {

    @TableId(type = IdType.INPUT)
    private Long itemId;

    private Long orderId;
    private Long userId;
    private Long goodsId;
    private String goodsName;
    private BigDecimal price;
    private Integer quantity;

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;
}

// ===== OrderMapper.java =====
package com.example.sharding.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import com.example.sharding.entity.Order;
import org.apache.ibatis.annotations.*;

import java.util.List;

@Mapper
public interface OrderMapper extends BaseMapper<Order> {

    /**
     * 按用户ID和状态查询（带分页）
     * 分片键：user_id → 精确路由到1个库
     * 无 order_id 条件 → 该库内所有4张表都查（4条SQL）
     */
    @Select("SELECT * FROM t_order " +
            "WHERE user_id = #{userId} " +
            "AND status = #{status} " +
            "ORDER BY create_time DESC " +
            "LIMIT #{offset}, #{limit}")
    List<Order> findByUserIdAndStatus(
            @Param("userId") Long userId,
            @Param("status") Integer status,
            @Param("offset") int offset,
            @Param("limit")  int limit);

    /**
     * 按订单ID查询（最精确路由）
     * 分片键：user_id + order_id → 精确路由到1个分片
     */
    @Select("SELECT * FROM t_order WHERE user_id = #{userId} AND order_id = #{orderId}")
    Order findByUserIdAndOrderId(
            @Param("userId")  Long userId,
            @Param("orderId") Long orderId);

    /**
     * 统计用户订单总数
     * 触发4条SQL（该库的4张分片表），归并引擎 SUM
     */
    @Select("SELECT COUNT(*) FROM t_order WHERE user_id = #{userId}")
    Long countByUserId(@Param("userId") Long userId);

    /**
     * 更新订单状态（CAS 乐观更新）
     */
    @Update("UPDATE t_order SET status = #{newStatus}, update_time = NOW() " +
            "WHERE user_id = #{userId} AND order_id = #{orderId} AND status = #{oldStatus}")
    int updateStatus(@Param("userId")    Long userId,
                     @Param("orderId")   Long orderId,
                     @Param("oldStatus") Integer oldStatus,
                     @Param("newStatus") Integer newStatus);
}

// ===== OrderService.java =====
package com.example.sharding.service;

import com.example.sharding.dto.*;
import com.example.sharding.entity.*;
import com.example.sharding.mapper.*;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@Slf4j
public class OrderService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private OrderItemMapper orderItemMapper;

    /**
     * 创建订单（含明细）
     * order 和 order_item 是绑定表，路由到同一分片
     */
    @Transactional(rollbackFor = Exception.class)
    public Order createOrder(CreateOrderDTO dto) {
        // 构建订单
        Order order = new Order();
        order.setUserId(dto.getUserId());
        order.setGoodsId(dto.getGoodsId());
        order.setQuantity(dto.getQuantity());
        order.setAmount(dto.getAmount());
        order.setStatus(0); // 创建中
        order.setAddressId(dto.getAddressId());

        // 插入订单（ShardingSphere 自动生成 orderId 并路由）
        orderMapper.insert(order);
        log.info("订单创建成功: orderId={}, 路由: user_id={} → ds_{}",
                 order.getOrderId(),
                 order.getUserId(),
                 order.getUserId() % 4);

        // 插入订单明细（绑定表，同一分片）
        for (CreateOrderItemDTO itemDTO : dto.getItems()) {
            OrderItem item = new OrderItem();
            item.setOrderId(order.getOrderId());
            item.setUserId(dto.getUserId()); // 必须设置分库键
            item.setGoodsId(itemDTO.getGoodsId());
            item.setGoodsName(itemDTO.getGoodsName());
            item.setPrice(itemDTO.getPrice());
            item.setQuantity(itemDTO.getQuantity());
            orderItemMapper.insert(item);
        }

        return order;
    }

    /**
     * 按用户查询订单列表（分页）
     */
    public List<Order> getUserOrders(Long userId, int page, int pageSize) {
        int offset = (page - 1) * pageSize;
        // user_id 分片键存在 → 精确路由到1个库，但4张分片表都查
        return orderMapper.findByUserIdAndStatus(userId, null, offset, pageSize);
    }

    /**
     * 查询订单详情（最精确路由）
     */
    public Order getOrderDetail(Long userId, Long orderId) {
        // user_id + order_id 双分片键 → 精确路由到1个分片（1条SQL）
        return orderMapper.findByUserIdAndOrderId(userId, orderId);
    }

    /**
     * 支付订单
     */
    @Transactional(rollbackFor = Exception.class)
    public boolean payOrder(Long userId, Long orderId) {
        // CAS 更新：0（创建中）→ 2（已支付）
        int affected = orderMapper.updateStatus(userId, orderId, 0, 2);
        if (affected == 0) {
            log.warn("订单状态更新失败，可能已支付或不存在: orderId={}", orderId);
            return false;
        }
        log.info("订单支付成功: orderId={}", orderId);
        return true;
    }
}
```

## 10.6 Controller 和测试

```java
// ===== OrderController.java =====
package com.example.sharding.controller;

import com.example.sharding.dto.*;
import com.example.sharding.entity.Order;
import com.example.sharding.service.OrderService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @PostMapping
    public Order createOrder(@RequestBody CreateOrderDTO dto) {
        return orderService.createOrder(dto);
    }

    @GetMapping("/user/{userId}")
    public List<Order> getUserOrders(
            @PathVariable Long userId,
            @RequestParam(defaultValue = "1") int page,
            @RequestParam(defaultValue = "10") int pageSize) {
        return orderService.getUserOrders(userId, page, pageSize);
    }

    @GetMapping("/{orderId}")
    public Order getDetail(
            @RequestParam Long userId,
            @PathVariable Long orderId) {
        return orderService.getOrderDetail(userId, orderId);
    }

    @PostMapping("/{orderId}/pay")
    public boolean pay(
            @RequestParam Long userId,
            @PathVariable Long orderId) {
        return orderService.payOrder(userId, orderId);
    }
}

// ===== ShardingDemoApplicationTests.java =====
package com.example.sharding;

import com.example.sharding.dto.*;
import com.example.sharding.entity.Order;
import com.example.sharding.service.OrderService;
import lombok.extern.slf4j.Slf4j;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.math.BigDecimal;
import java.util.*;

@SpringBootTest
@Slf4j
class ShardingDemoApplicationTests {

    @Autowired
    private OrderService orderService;

    /**
     * 测试1：插入不同用户的订单，验证路由分布
     */
    @Test
    void testInsertAndRouting() {
        for (long userId = 1001L; userId <= 1012L; userId++) {
            CreateOrderDTO dto = CreateOrderDTO.builder()
                .userId(userId)
                .goodsId(100L)
                .quantity(1)
                .amount(new BigDecimal("99.99"))
                .items(Collections.singletonList(
                    CreateOrderItemDTO.builder()
                        .goodsId(100L)
                        .goodsName("测试商品")
                        .price(new BigDecimal("99.99"))
                        .quantity(1)
                        .build()
                ))
                .build();

            Order order = orderService.createOrder(dto);

            log.info("userId={} → ds_{}.t_order_{} orderId={}",
                     userId,
                     userId % 4,
                     order.getOrderId() % 4,
                     order.getOrderId());
        }
    }

    /**
     * 测试2：验证分片键查询精确路由
     */
    @Test
    void testPreciseRouting() {
        Long userId  = 1001L;
        Long orderId = createTestOrder(userId);

        // 精确路由（双分片键）：只执行1条SQL
        Order order = orderService.getOrderDetail(userId, orderId);
        log.info("精确查询结果: {}", order);
        assert order != null : "订单应存在";
        assert order.getUserId().equals(userId) : "用户ID应匹配";
    }

    /**
     * 测试3：跨分片统计（触发全路由，所有分片）
     */
    @Test
    void testCrossShardAggregation() {
        Long userId = 1001L;
        // 先插入多个订单
        for (int i = 0; i < 5; i++) {
            createTestOrder(userId);
        }

        // COUNT(*) 触发该库4张分片表的查询，归并引擎求和
        Long total = orderMapper.countByUserId(userId);
        log.info("用户 {} 的订单总数（跨分片归并）: {}", userId, total);
    }

    private Long createTestOrder(Long userId) {
        CreateOrderDTO dto = CreateOrderDTO.builder()
            .userId(userId)
            .goodsId(100L)
            .quantity(1)
            .amount(new BigDecimal("99.99"))
            .items(Collections.emptyList())
            .build();
        return orderService.createOrder(dto).getOrderId();
    }

    @Autowired
    private com.example.sharding.mapper.OrderMapper orderMapper;
}
```

---

---

# Part 11: 常见问题与解决方案 {#part-11}

## 11.1 深分页性能问题

```
问题描述：
  SELECT * FROM t_order ORDER BY create_time DESC LIMIT 1000000, 10;
  （翻到第 100000 页，每页10条）

ShardingSphere 处理过程：
  改写后对每个分片执行：
    SELECT * FROM t_order_0 ORDER BY create_time DESC LIMIT 0, 1000010
    SELECT * FROM t_order_1 ORDER BY create_time DESC LIMIT 0, 1000010
    SELECT * FROM t_order_2 ORDER BY create_time DESC LIMIT 0, 1000010
    SELECT * FROM t_order_3 ORDER BY create_time DESC LIMIT 0, 1000010

  每个分片返回 1000010 条！
  归并节点排序 4 × 1000010 = 4000040 条
  取第 1000001 ~ 1000010 条

  内存占用：约 4M × 记录大小（可能 GB 级别）
  查询时间：可能超过 30 秒！
```

**解决方案1：游标分页（推荐）**

```java
/**
 * 游标分页（无限深度，性能稳定）
 * 关键：用上次最后一条记录的分片键+主键作为游标
 */
@Service
public class OrderQueryService {

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 游标分页接口
     * 首页：lastOrderId = null
     * 下页：传入上次最后一条的 orderId
     */
    public List<Order> getNextPage(Long userId, Long lastOrderId, int pageSize) {
        if (lastOrderId == null) {
            // 首页（最新N条）
            return orderMapper.findLatest(userId, pageSize);
        } else {
            // 下一页（order_id < lastOrderId，降序翻页）
            return orderMapper.findBefore(userId, lastOrderId, pageSize);
        }
    }
}
```

```sql
-- Mapper SQL
-- 首页
SELECT * FROM t_order
WHERE user_id = #{userId}
ORDER BY order_id DESC
LIMIT #{pageSize};

-- 下一页（游标 = 上次最后一条 order_id）
SELECT * FROM t_order
WHERE user_id = #{userId}
AND order_id < #{lastOrderId}    -- 关键：游标条件，分片键精确路由
ORDER BY order_id DESC
LIMIT #{pageSize};

-- 为什么这样高效：
-- user_id → 精确路由到1个库
-- order_id < xxx → 范围条件，利用B+树索引快速定位
-- 无论翻到多少页，查询时间恒定
```

**解决方案2：数据归档到 ES**

```java
/**
 * MySQL 保留近90天热数据，历史数据归档到 Elasticsearch
 * ES 支持高性能深分页（scroll API）
 */
@Service
public class OrderSearchService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private ElasticsearchClient esClient;

    /**
     * 近期数据：MySQL 分页（限制前10页）
     */
    public List<Order> searchRecent(OrderSearchDTO dto) {
        if (dto.getPage() > 10) {
            throw new BizException("最多查询前10页，请缩小时间范围");
        }
        return orderMapper.search(dto);
    }

    /**
     * 历史数据：ES 分页（支持无限深度）
     */
    public SearchResult<Order> searchHistory(OrderSearchDTO dto) {
        // 使用 ES search_after 游标分页
        SearchRequest request = SearchRequest.of(s -> s
            .index("order_history")
            .query(q -> q.bool(b -> b
                .must(m -> m.term(t -> t.field("user_id").value(dto.getUserId())))
            ))
            .sort(so -> so.field(f -> f.field("create_time").order(SortOrder.Desc)))
            .searchAfter(dto.getSearchAfter())  // 游标
            .size(dto.getPageSize())
        );
        // 执行ES查询...
        return esClient.search(request, Order.class);
    }
}
```

---

## 11.2 跨分片聚合查询

```
问题：聚合函数在分库分表后的计算

SELECT COUNT(*), SUM(amount), AVG(amount), MAX(amount), MIN(amount)
FROM t_order WHERE status = 2;

ShardingSphere 处理：
  路由到所有16个分片（无分片键）

  改写 AVG（关键！）：
  原始：SELECT AVG(amount) FROM t_order WHERE status = 2
  改写：SELECT COUNT(amount) AS ct, SUM(amount) AS sm
        FROM t_order_x WHERE status = 2

  各分片结果：
    ds_0.t_order_0: ct=100, sm=10000, max=500, min=10
    ds_0.t_order_1: ct=150, sm=18000, max=600, min=15
    ds_1.t_order_0: ct=200, sm=22000, max=800, min=5
    ...

  归并：
    COUNT = SUM(各分片ct) = 1600
    SUM   = SUM(各分片sm) = 180000
    AVG   = 总SUM / 总COUNT = 180000 / 1600 = 112.5  ✓（正确）
    MAX   = MAX(各分片max) = 800  ✓
    MIN   = MIN(各分片min) = 5    ✓
```

**注意：直接计算 AVG 会出错！**
```sql
-- 错误做法（应用层直接平均各分片的 AVG）：
-- 分片0 AVG = 100.0
-- 分片1 AVG = 120.0
-- 分片2 AVG = 110.0
-- 错误结果：(100+120+110)/3 = 110.0   ← 不正确！各分片记录数不同
-- 正确结果：SUM(所有sm) / SUM(所有ct) = 112.5

-- ShardingSphere 已自动处理这个问题
-- 会将 AVG 改写为 SUM + COUNT，归并时正确计算
```

---

## 11.3 跨分片排序与分组

```
GROUP BY + ORDER BY 的归并逻辑：

SELECT user_id, COUNT(*) AS order_cnt, SUM(amount) AS total_amount
FROM t_order
GROUP BY user_id
ORDER BY total_amount DESC
LIMIT 10;

注意：user_id 是分片键，所以同一 user_id 的数据都在同一个库
但是不同库有不同的 user_id

处理流程：
  1. 每个分片独立 GROUP BY → 本分片内 user_id 的统计
  2. 归并引擎收集所有分片结果
  3. 相同 user_id 的统计需要再次合并（跨分片同一 user_id？）
     → 实际上分片键是 user_id，所以同一 user_id 只在1个库
     → 这种情况下，每个分片的 GROUP BY 结果不重叠
     → 直接 ORDER BY 归并即可
  
  4. 最终排序：ORDER BY total_amount DESC
     → 归并引擎对所有分片的结果做 ORDER BY 归并
     → 取前 10 条

潜在问题（当 GROUP BY 的列不是分片键时）：
  SELECT goods_id, COUNT(*) FROM t_order GROUP BY goods_id
  -- goods_id 不是分片键！同一 goods_id 的数据分布在多个分片
  
  分片0 GROUP BY goods_id：{goods_1:100, goods_2:200, goods_3:50}
  分片1 GROUP BY goods_id：{goods_1:80,  goods_2:150, goods_3:70}
  
  归并时需要合并相同 goods_id：
    goods_1: 100+80 = 180
    goods_2: 200+150 = 350
    goods_3: 50+70 = 120
  
  ShardingSphere 归并引擎自动处理！
  内部实现：Map<GroupKey, AggregatedValue>，遍历所有分片结果
```

---

## 11.4 广播表使用

```java
/**
 * 广播表的增删改查示例
 */
@Service
public class CategoryService {

    @Autowired
    private CategoryMapper categoryMapper;

    /**
     * 新增分类（广播写：所有分片库都执行）
     * ShardingSphere 自动在4个分片库各执行一次 INSERT
     * 保证所有分片库数据一致
     */
    @Transactional
    public GoodsCategory addCategory(GoodsCategory category) {
        categoryMapper.insert(category);
        // 实际执行：
        // INSERT INTO order_db_0.t_goods_category ...
        // INSERT INTO order_db_1.t_goods_category ...
        // INSERT INTO order_db_2.t_goods_category ...
        // INSERT INTO order_db_3.t_goods_category ...
        return category;
    }

    /**
     * 查询分类列表（单播读：任意一个分片库执行）
     * 因为每个库数据相同，读哪个都一样
     */
    public List<GoodsCategory> listCategories() {
        // 实际只执行一条 SQL（ShardingSphere 选择 ds_0）
        return categoryMapper.selectList(null);
    }

    /**
     * 广播表与分片表的 JOIN（高效！）
     *
     * 因为每个分片库都有 t_goods_category 的完整数据，
     * JOIN 可以在单个分片库内完成，无需跨库
     */
    public List<OrderWithCategory> getOrdersWithCategory(Long userId) {
        // t_order 路由到 ds_1（user_id=1001）
        // ds_1 也有完整的 t_goods_category
        // 可以在 ds_1 内部直接 JOIN！
        return orderMapper.findWithCategory(userId);
    }
}
```

---

## 11.5 不支持的 SQL

```
ShardingSphere 不支持或限制的 SQL：

1. 跨分片 UNION（部分支持）
   支持：UNION ALL（结果合并，不去重）
   不支持：UNION（需要全局去重，代价高）

2. 子查询中使用分片键
   SELECT * FROM t_order WHERE order_id IN (
     SELECT order_id FROM t_order WHERE status = 2   -- 子查询中 status 非分片键
   )
   建议改为 JOIN 或应用层处理

3. 存储过程和函数
   ShardingSphere 无法解析存储过程内部的 SQL
   不支持路由到特定分片
   建议：将存储过程逻辑迁移到应用层

4. 多表 UPDATE（部分支持）
   不支持：
   UPDATE t_order o, t_order_item i
   SET o.status = 2
   WHERE o.order_id = i.order_id
   建议：改为单表更新，应用层控制顺序

5. LAST_INSERT_ID()
   在分片环境中无意义（每个连接有自己的值）
   主键已由 ShardingSphere 生成，直接使用即可

6. 部分聚合函数
   支持：COUNT / SUM / MAX / MIN / AVG
   不支持：GROUP_CONCAT（跨分片合并困难）
   不支持：DISTINCT + ORDER BY（复杂归并）

7. SELECT LAST_INSERT_ID() 等连接状态函数
   这些函数返回的是当前连接的状态
   在连接池模式下，连接可能来自任意分片

8. DDL 对分片表无效（需通过 DistSQL 或手动执行）
   ALTER TABLE t_order ADD COLUMN new_col INT;
   → ShardingSphere 会把这条 DDL 路由到所有16个物理表
   → MySQL 支持，但 ShardingSphere 处理方式不同版本有差异
   建议：分片表的 DDL 手动在所有物理表上执行

已知限制列表（ShardingSphere 文档）：
   https://shardingsphere.apache.org/document/current/cn/features/sharding/limitation/
```

---

## 11.6 分片扩容方案

```
场景：从 4库16表 扩容为 8库32表

方案1：一致性哈希扩容（迁移数据最少）
  使用一致性哈希而非取模分片
  扩容时只有部分数据需要迁移（理论上 1/N 的数据）
  
方案2：成倍扩容（推荐，最简单）
  4库 → 8库（翻倍）
  
  迁移策略：
  Step1: 新建8个数据库（order_db_0 ~ order_db_7）
  Step2: 将原 4库 数据复制到对应的 8库
         原 ds_0 → 新 ds_0 + ds_4（根据新规则再次分配）
  Step3: 数据校验（新旧库数据一致性验证）
  Step4: 切换 ShardingSphere 配置到8库规则
  Step5: 验证服务正常
  Step6: 删除旧库中已迁移的数据（可延后）

方案3：停写迁移（简单但有停机窗口）
  Step1: 停写（维护窗口）
  Step2: 数据迁移
  Step3: 修改配置
  Step4: 恢复服务

工具支持：
  ShardingSphere Scaling（内置数据迁移）
    支持在线不停服扩容
    通过 CDC（Change Data Capture）实时同步增量数据
  
  第三方工具：
    DTS（阿里云数据传输服务）
    Canal + 自定义消费者
```

---

---

# Part 12: 运维与监控 {#part-12}

## 12.1 ShardingSphere-Proxy 部署

ShardingSphere-Proxy 是独立部署的数据库代理，对外暴露 MySQL 协议。

```
部署结构：

  Java应用       Python脚本     DBA工具（Navicat）
      |              |                |
      +--------------+----------------+
                     |
                     | MySQL协议 3307
                     ↓
          +---------------------+
          | ShardingSphere-Proxy|
          |    （3307端口）      |
          | 模拟 MySQL 8.0 服务器|
          +---------------------+
                     |
          +----------+----------+
          |          |          |
        ds_0       ds_1       ds_2    （真实 MySQL 实例）
       3306       3307       3308
```

**Docker 部署：**

```bash
# 1. 创建配置目录
mkdir -p /opt/shardingsphere-proxy/conf
mkdir -p /opt/shardingsphere-proxy/ext-lib

# 2. 编写配置文件

# server.yaml（全局配置）
cat > /opt/shardingsphere-proxy/conf/server.yaml << 'EOF'
authority:
  users:
    - user: root
      password: root123
      hostname: "%"    # 允许所有IP连接
  privilege:
    type: ALL_PERMITTED

props:
  sql-show: true
  proxy-backend-query-fetch-size: 100
  proxy-default-port: 3307
EOF

# config-sharding.yaml（分片规则）
cat > /opt/shardingsphere-proxy/conf/config-sharding.yaml << 'EOF'
databaseName: order_db_proxy    # Proxy 暴露的逻辑数据库名

dataSources:
  ds_0:
    url: jdbc:mysql://mysql-0:3306/order_db_0?useSSL=false
    username: root
    password: root123
    connectionTimeoutMilliseconds: 30000
    idleTimeoutMilliseconds: 60000
    maxLifetimeMilliseconds: 1800000
    maxPoolSize: 50
    minPoolSize: 1

  ds_1:
    url: jdbc:mysql://mysql-1:3306/order_db_1?useSSL=false
    username: root
    password: root123
    connectionTimeoutMilliseconds: 30000
    maxPoolSize: 50
    minPoolSize: 1

rules:
  - !SHARDING
    tables:
      t_order:
        actualDataNodes: ds_$->{0..1}.t_order_$->{0..3}
        databaseStrategy:
          standard:
            shardingColumn: user_id
            shardingAlgorithmName: db_mod
        tableStrategy:
          standard:
            shardingColumn: order_id
            shardingAlgorithmName: table_mod
        keyGenerateStrategy:
          column: order_id
          keyGeneratorName: snowflake

    shardingAlgorithms:
      db_mod:
        type: MOD
        props:
          sharding-count: 2
      table_mod:
        type: MOD
        props:
          sharding-count: 4

    keyGenerators:
      snowflake:
        type: SNOWFLAKE
        props:
          worker-id: 1
EOF

# 3. 启动 Proxy
docker run -d \
  --name shardingsphere-proxy \
  -p 3307:3307 \
  -v /opt/shardingsphere-proxy/conf:/opt/shardingsphere-proxy/conf \
  -v /opt/shardingsphere-proxy/ext-lib:/opt/shardingsphere-proxy/ext-lib \
  apache/shardingsphere-proxy:5.4.1

# 4. 验证连接
mysql -h 127.0.0.1 -P 3307 -u root -proot123 -e "SHOW DATABASES;"
# 应该看到 order_db_proxy
```

**应用连接 Proxy（完全透明）：**
```yaml
# 应用只需要修改数据库连接地址！
spring:
  datasource:
    url: jdbc:mysql://shardingsphere-proxy:3307/order_db_proxy?useSSL=false
    username: root
    password: root123
# 完全不需要引入 ShardingSphere 任何依赖！
# 分片对应用完全透明
```

---

## 12.2 DistSQL（分布式 SQL）

DistSQL 是 ShardingSphere 独有的 SQL 扩展，用于通过 SQL 命令动态管理分片规则，无需重启服务。

```sql
-- ===== 连接 Proxy =====
mysql -h 127.0.0.1 -P 3307 -u root -proot123

-- ===== 查看已有规则 =====
SHOW DATABASES;                              -- 查看逻辑数据库
USE order_db_proxy;
SHOW SHARDING TABLE RULES;                   -- 查看分片规则
SHOW READWRITE_SPLITTING RULES;              -- 查看读写分离规则
SHOW ENCRYPT RULES;                          -- 查看加密规则

-- ===== 动态添加数据源 =====
REGISTER STORAGE UNIT ds_4 (
    URL="jdbc:mysql://mysql-4:3306/order_db_4?useSSL=false",
    USER="root",
    PASSWORD="root123",
    PROPERTIES("maximumPoolSize"="50")
);

-- ===== 添加分片规则 =====
CREATE SHARDING TABLE RULE t_order (
    DATANODES("ds_$->{0..4}.t_order_$->{0..3}"),
    DATABASE_STRATEGY(TYPE=MOD, SHARDING_COLUMN=user_id, SHARDING_COUNT=5),
    TABLE_STRATEGY(TYPE=MOD, SHARDING_COLUMN=order_id, SHARDING_COUNT=4),
    KEY_GENERATE_STRATEGY(COLUMN=order_id, TYPE=SNOWFLAKE)
);

-- ===== 修改分片规则 =====
ALTER SHARDING TABLE RULE t_order (
    DATANODES("ds_$->{0..7}.t_order_$->{0..3}"),
    DATABASE_STRATEGY(TYPE=MOD, SHARDING_COLUMN=user_id, SHARDING_COUNT=8),
    TABLE_STRATEGY(TYPE=MOD, SHARDING_COLUMN=order_id, SHARDING_COUNT=4),
    KEY_GENERATE_STRATEGY(COLUMN=order_id, TYPE=SNOWFLAKE)
);

-- ===== 删除分片规则 =====
DROP SHARDING TABLE RULE t_order;

-- ===== 读写分离规则 =====
CREATE READWRITE_SPLITTING RULE ds_readwrite (
    WRITE_STORAGE_UNIT=primary_ds,
    READ_STORAGE_UNITS(replica_ds_0, replica_ds_1),
    TRANSACTIONAL_READ_QUERY_STRATEGY=PRIMARY,
    TYPE(NAME=ROUND_ROBIN)
);

-- ===== 加密规则 =====
CREATE ENCRYPT RULE t_user (
    COLUMNS(
        (NAME=phone,
         CIPHER=phone_cipher,
         PLAIN=phone_plain,
         ENCRYPT_ALGORITHM(TYPE(NAME=AES,PROPERTIES('aes-key-value'='SecretKey@123'))))
    )
);

-- ===== 查看 SQL 路由（调试工具）=====
PREVIEW SELECT * FROM t_order WHERE user_id = 1001 AND order_id = 88888;
-- 输出：
-- +----------+---------------------------------------------+
-- | DataSource | Actual SQL                                |
-- +----------+---------------------------------------------+
-- | ds_1      | SELECT * FROM t_order_0 WHERE ...          |
-- +----------+---------------------------------------------+

-- ===== 查看数据节点 =====
SHOW SHARDING TABLE NODES;
```

---

## 12.3 治理中心（动态配置推送）

```
治理中心架构：

  ShardingSphere-JDBC/Proxy
            |
            | 监听配置变更
            ↓
  +-------------------+
  |   ZooKeeper/Etcd  |
  |    治理中心        |
  +-------------------+
            ↑
            | 推送配置变更
  +-------------------+
  |   运维控制台        |
  |   (ShardingSphere  |
  |    Dashboard)      |
  +-------------------+

好处：
  - 多实例配置同步（修改一处，所有实例实时生效）
  - 无需重启服务
  - 配置版本管理（支持回滚）
```

**ZooKeeper 治理中心配置：**
```yaml
# application.yml
spring:
  shardingsphere:
    mode:
      type: Cluster     # 集群模式（使用注册中心）
      repository:
        type: ZooKeeper  # 使用 ZooKeeper 作为治理中心
        props:
          namespace: shardingsphere-demo     # ZK 命名空间
          server-lists: zoo1:2181,zoo2:2181,zoo3:2181
          retryIntervalMilliseconds: 500
          timeToLiveSeconds: 60
          maxRetries: 3
          operationTimeoutMilliseconds: 500
      overwrite: false   # false：ZK 中的配置优先；true：本地配置覆盖 ZK
    # 其他配置...
```

**Etcd 治理中心：**
```yaml
spring:
  shardingsphere:
    mode:
      type: Cluster
      repository:
        type: Etcd
        props:
          server-lists: etcd1:2379,etcd2:2379,etcd3:2379
          timeToLiveSeconds: 30
          connectionTimeoutSeconds: 5
```

---

## 12.4 监控集成

### OpenTelemetry（推荐）

```yaml
# application.yml
spring:
  shardingsphere:
    props:
      sql-show: false  # 生产关闭
    
    # 可观测性配置
    # 通过 Agent 接入 OpenTelemetry
```

```bash
# 使用 ShardingSphere Agent 启动
# 下载 shardingsphere-agent-5.4.1.jar

java -javaagent:/path/to/shardingsphere-agent.jar \
     -Dplugin.tracing.OpenTelemetry.otel-service-name=order-service \
     -Dplugin.tracing.OpenTelemetry.otel-target-url=http://jaeger:4317 \
     -jar your-app.jar
```

### Prometheus + Grafana

```yaml
# Agent 配置文件 agent.yaml
plugins:
  # 启用 Prometheus 指标暴露
  metrics:
    Prometheus:
      host: "localhost"
      port: 9090
  # 启用链路追踪
  tracing:
    OpenTelemetry:
      otel-service-name: "order-service"
      otel-target-url: "http://jaeger-collector:4317"
      otel-sampler-type: "ratio"
      otel-sampler-ratio: 0.1    # 采样10%
```

**关键监控指标：**

```
SQL 执行指标：
  shardingsphere.execute.latency    - SQL 执行延迟（P99）
  shardingsphere.execute.count      - SQL 执行次数
  shardingsphere.execute.error      - SQL 执行错误次数

路由指标：
  shardingsphere.route.datasource.count - 每次路由的分片数
  全路由告警：当路由分片数 = 总分片数时告警！

连接池指标：
  shardingsphere.connection.active  - 当前活跃连接数
  shardingsphere.connection.idle    - 空闲连接数
  shardingsphere.connection.pending - 等待连接的请求数

告警规则建议：
  1. 全路由比例 > 5%       → 告警（可能缺少分片键条件）
  2. 连接池使用率 > 80%    → 告警
  3. SQL P99 延迟 > 500ms  → 告警
  4. SQL 错误率 > 0.1%     → 告警
```

---

---

# Part 13: 常见面试题 FAQ {#part-13}

## Q1. ShardingSphere-JDBC 和 ShardingSphere-Proxy 如何选择？

```
答：根据技术栈和场景选择。

ShardingSphere-JDBC（推荐 Java 项目）：
  ✓ 纯 Java 客户端，无额外部署
  ✓ 无网络开销（进程内执行），延迟最低
  ✓ Spring Boot Starter 集成简单（5分钟上手）
  ✗ 仅支持 Java 语言
  ✗ 多服务每个都要配置，配置分散

ShardingSphere-Proxy（推荐多语言/统一运维）：
  ✓ 支持任意语言（Python/Go/PHP 直接连接）
  ✓ 对应用完全透明（只改连接字符串）
  ✓ 统一运维入口（DBA 可用 MySQL 工具直接操作）
  ✓ 支持 DistSQL 动态修改规则
  ✗ 多一次网络跳转（通常 < 1ms 影响）
  ✗ 需要独立部署 Proxy 集群

最佳实践：
  Java 微服务 → ShardingSphere-JDBC
  多语言团队或强调 DBA 运维 → ShardingSphere-Proxy
  大型系统 → JDBC（Java服务）+ Proxy（DBA运维）同时使用
```

---

## Q2. 分片键选择不当会有什么问题？

```
答：分片键选错会导致严重的性能和数据分布问题。

问题1：热点分片
  错误：按省份分片（order_province）
  结果：广东省订单占40%，远超其他省份
  后果：ds_guangdong 持续热，其他 ds 资源浪费

  正确：按用户ID（随机分布，均匀）

问题2：全路由频繁
  错误：按商品状态（status）分片
  结果：大多数查询都带 user_id 而非 status
  后果：95% 的查询触发全路由，性能极差

  正确：选择高频查询条件作为分片键

问题3：跨分片 JOIN 爆炸
  错误：t_order 按 user_id 分片，t_order_item 按 goods_id 分片
  结果：查询订单明细必须跨分片 JOIN
  后果：笛卡尔积路由，N×M 条 SQL

  正确：绑定表，使用相同分片键

问题4：分片键改变导致全量迁移
  分片键是 immutable（不可变）的
  一旦上线，修改分片键 = 全量数据重迁移（数亿条）
  代价极高！
```

---

## Q3. 什么是绑定表？为什么可以避免笛卡尔积？

```
答：

绑定表是指使用完全相同分片策略的一组关联表。
ShardingSphere 保证绑定表中分片键相同的数据落在同一物理分片。

为什么避免笛卡尔积：

未配置绑定表：
  t_order    路由：user_id=1001 → ds_1.t_order_1（1个目标）
  t_order_item 路由：独立判断 → ds_1的4张表（4个目标）
  
  JOIN 时：ShardingSphere 不知道两表数据是否在同一分片
  → 生成笛卡尔积：1 × 4 = 4 条 JOIN SQL
  → 结果合并可能有重复

配置绑定表后：
  两表使用相同分片规则
  user_id=1001 → t_order_1 AND t_order_item_1（同一分片号）
  
  ShardingSphere 知道两表对应分片一致
  → 生成 1 条 JOIN SQL（ds_1.t_order_1 JOIN ds_1.t_order_item_1）
  → 高效，结果正确

配置方式：
  binding-tables:
    - t_order,t_order_item  # 同一组
    - t_user,t_user_account # 另一组

注意事项：
  绑定表的分片键（sharding-column）和分片算法必须完全相同
  不同组的绑定表之间不影响
```

---

## Q4. 雪花算法时钟回拨是什么问题？如何解决？

```
答：

时钟回拨原因：
  服务器时间通过 NTP 同步，有时会往回调整
  例如：当前时间 10:00:01.500，NTP 同步后变回 10:00:01.200
  时钟回拨了 300ms

危害：
  ID 中的时间戳部分减小
  可能生成与之前相同的 ID → ID 重复！

ShardingSphere 默认处理：
  max-vibration-offset: 5  # 容忍最大5ms的回拨
  
  回拨 ≤ max-vibration-offset：
    稍微调大序列号位，避免重复（抖动容忍）
  
  回拨 > max-vibration-offset：
    抛异常，阻止生成 ID

工程解决方案：

方案1：等待时钟追上（小幅回拨）
  // 检测到 3ms 回拨
  Thread.sleep(3 + 1); // 等待时钟同步

方案2：使用 CosID（推荐，高性能）
  CosID 通过原子序列号解决时钟回拨问题
  不依赖系统时钟单调性，性能是普通雪花算法的5倍以上

方案3：时钟回拨告警 + 自动重试
  捕获时钟回拨异常
  等待 N 毫秒后重试
  超过阈值则告警（服务器时钟问题需要运维处理）

方案4：NTP 配置优化
  使用 chrony 替代 ntpd，支持平滑调整（slewing）
  避免大幅跳变，每次调整幅度 < 1ms
```

---

## Q5. ShardingSphere 支持哪些分布式事务？各有什么优缺点？

```
答：见 Part 7 详细说明，面试答要点：

三种事务类型：
1. LOCAL（本地事务）
   - 各数据源独立的本地事务
   - 只能保证单数据源的原子性
   - 跨数据源时无效（不保证原子性）
   - 适合：所有 SQL 路由到同一数据源的操作

2. XA（强一致，两阶段提交）
   - 强一致性（ACID）
   - 第一阶段 Prepare 时持有锁
   - 性能较低（高并发下锁竞争）
   - 适合：金融场景、强一致要求

3. BASE（最终一致，Seata AT）
   - 最终一致性
   - 第一阶段提交本地事务，锁释放快
   - 性能好
   - 需要额外部署 Seata Server
   - 适合：大多数互联网业务（下单、退款）

选型原则：
  单分片 → LOCAL
  金融强一致 → XA
  高并发跨分片 → BASE（Seata）
```

---

## Q6. 如何解决分库分表后的分页问题？

```
答：

问题本质：
  分页的 LIMIT offset, count 需要跳过前 offset 条
  在分片环境下，"前 offset 条"分布在多个分片
  每个分片必须多返回数据，归并后才能确定正确结果

ShardingSphere 的处理（归并排序）：
  对每个分片执行 LIMIT 0, (offset + count)
  归并后取第 (offset+1) ~ (offset+count) 条
  
  深分页时性能极差（每个分片返回大量数据）

推荐方案（按优先级）：

1. 游标分页（最推荐）
   原理：记录上次最后一条记录的分片键
   优势：无论翻多少页，性能恒定 O(1)
   限制：只能顺序翻页，不能跳转

2. 限制最大页码
   if (page > 100) throw BizException("超出查询范围");
   简单粗暴，实际可满足大多数场景

3. ES 历史数据
   MySQL 保留最近N天热数据
   历史数据归档到 ES，ES 支持高性能深分页

4. 用户维度分片 + 统一库查询
   如果按 user_id 分片，查某用户的所有数据 → 只在1个分片库
   只有4张表，LIMIT 1000, 10 代价可接受
```

---

## Q7. 广播表和绑定表的区别？

```
答：

广播表（Broadcast Table）：
  定义：在每个分片库中都有完整数据的表
  特点：
    写操作广播到所有分片库（保持一致）
    读操作只查任意一个分片（单播）
  适用：小数据量、不常变的参照数据（字典、配置）
  示例：t_goods_category（商品分类），数据量 < 10万行

绑定表（Binding Table）：
  定义：使用相同分片策略的一组关联表
  特点：
    相同分片键的数据在同一分片
    JOIN 时无需跨分片，避免笛卡尔积
  适用：有关联 JOIN 的分片表组
  示例：t_order + t_order_item（都按 user_id 和 order_id 分片）

核心区别：
  广播表：数据完全复制到所有分片（小表）
  绑定表：数据按规则分布（大表），但保证关联数据同分片

联合使用：
  SELECT o.*, c.name FROM t_order o        -- 分片表
  JOIN t_goods_category c ON o.cat_id = c.id  -- 广播表
  WHERE o.user_id = 1001;
  
  t_order → ds_1（精确路由）
  t_goods_category → ds_1 也有完整数据（广播表）
  → 在 ds_1 内部直接 JOIN！完全不跨库！
```

---

## Q8. 什么情况下会触发全路由？如何避免？

```
答：

触发全路由的场景：
  1. SQL 中没有分片键条件
     SELECT * FROM t_order WHERE status = 2  -- status 非分片键

  2. 分片键使用了函数
     WHERE YEAR(create_time) = 2023  -- 无法从函数推导分片值

  3. 分片键在 OR 条件中（复杂情况）
     WHERE user_id = 1001 OR status = 2  -- OR 导致无法精确路由

  4. 分片键使用了 NOT IN 或 !=
     WHERE user_id != 1001  -- 无法确定路由哪些分片

全路由的代价（4库16表）：
  16 条 SQL 并行执行
  16 个 ResultSet 归并
  适量小数据还好，大数据量时极慢

如何避免：
  1. 业务上确保高频查询都带分片键
  2. 联合索引覆盖：(user_id, status) 索引
     WHERE user_id = 1001 AND status = 2
     → user_id 是分片键 → 精确路由1个库
     → status 在库内走联合索引

  3. 对全路由 SQL 设置告警阈值
     当单次查询命中分片数 > 4 时告警

  4. 不支持全路由的业务场景改用 ES
     如：按商品名搜索订单 → 用 ES 搜索 orderId，再按 orderId 查 MySQL
```

---

## Q9. 为什么 AVG 需要特殊处理？ShardingSphere 如何处理？

```
答：

错误做法（直接平均各分片 AVG）：
  分片0：100条记录，AVG = 100.0
  分片1：200条记录，AVG = 200.0
  
  错误归并：(100.0 + 200.0) / 2 = 150.0  ← 错！
  
  正确值：(100 × 100.0 + 200 × 200.0) / (100 + 200)
         = (10000 + 40000) / 300
         = 166.67  ← 才是正确的 AVG

ShardingSphere 的处理：
  自动将 AVG(amount) 改写为：
    COUNT(amount) AS ct_amount, SUM(amount) AS sum_amount
  
  归并时：
    总AVG = SUM(所有分片的sum_amount) / SUM(所有分片的ct_amount)

使用者注意事项：
  ShardingSphere 已自动正确处理，无需特别关注
  但如果你用 MyBatis 手写聚合 SQL，需要自己正确处理
```

---

## Q10. 分库分表后如何做数据库迁移？

```
答：在线不停服迁移流程（推荐）：

背景：从单库迁移到 4库16表

Step1: 新旧双写（保证数据完整性）
  修改代码：同时写入旧单库和新分片库
  持续时间：1~2周
  验证：比较新旧库数据是否一致

Step2: 历史数据迁移
  将旧库历史数据按分片规则导入新分片库
  工具：ShardingSphere Scaling / 自研迁移脚本
  注意：迁移期间继续双写，增量数据也迁移

Step3: 数据核验
  行数对比：旧库 COUNT = 新库各分片 COUNT 之和
  抽样比对：随机取 1% 记录逐字段比较
  业务验证：切换部分流量到新库，观察业务正确性

Step4: 灰度切换读请求
  先将10%读流量切到新分片库
  观察报错和慢 SQL
  逐步增加到100%读

Step5: 停写旧库
  将写请求切到新分片库
  旧库只保留只读备份
  保持7~30天后下线旧库

使用 ShardingSphere Scaling（内置迁移工具）：
  支持在线不停服迁移
  通过 DistSQL 触发迁移任务：
  
  START MIGRATION 'jobId';
  SHOW MIGRATION STATUS 'jobId';
  COMMIT MIGRATION 'jobId';
```

---

## Q11. 分片键是如何传递到分片算法的？

```
答：SQL 解析 → 路由引擎 → 分片算法

完整流程：
  SQL: SELECT * FROM t_order WHERE user_id = 1001 AND order_id = 88888

  1. SQL 解析引擎（Parse Engine）
     解析出条件：{user_id: 1001, order_id: 88888}
     识别这是 = 操作（精确分片）

  2. 路由引擎（Route Engine）
     查找 t_order 的分片规则：
     - database-strategy.sharding-column = user_id
     - table-strategy.sharding-column = order_id

     构造 PreciseShardingValue：
     - PreciseShardingValue(table="t_order", column="user_id", value=1001L)
     - PreciseShardingValue(table="t_order", column="order_id", value=88888L)

  3. 调用分片算法
     dbAlgorithm.doSharding(["ds_0","ds_1","ds_2","ds_3"], ShardingValue(user_id=1001))
     → 1001 % 4 = 1 → "ds_1"

     tableAlgorithm.doSharding(["t_order_0"..3], ShardingValue(order_id=88888))
     → 88888 % 4 = 0 → "t_order_0"

  4. 组合路由结果
     目标：ds_1.t_order_0（单个分片，最精确）

  5. 改写 SQL
     SELECT * FROM t_order_0 WHERE user_id = 1001 AND order_id = 88888
```

---

## Q12. 如何处理分库分表后的跨分片查询性能？

```
答：

跨分片查询的本质代价：
  路由到N个分片 → 执行N条SQL → 归并N个结果集
  N越大，代价越高

优化策略：

1. 确保分片键在查询条件中（最重要）
   业务开发规范：所有接口必须传入 user_id
   API设计上强制携带分片键

2. 使用复合索引减少跨表扫描
   在分片表上创建 (user_id, status, create_time) 联合索引
   WHERE user_id=1001 AND status=2 ORDER BY create_time
   → user_id 精确路由 → status+create_time 利用联合索引

3. 限制跨分片查询的分片数
   设置 max-connections-size-per-query 参数
   监控并告警路由分片数 > N 的查询

4. 将聚合类查询异步化
   实时聚合：读 MySQL
   历史聚合：定时任务写入汇总表 / 写入 ClickHouse

5. 引入 OLAP 层
   同步数据到 ClickHouse / TiDB / Doris
   复杂分析查询走 OLAP
   简单业务查询走 MySQL 分片

6. 冗余设计（空间换时间）
   在 t_order 中冗余 goods_name（避免 JOIN t_goods）
   用 Redis 缓存常用统计数据
```

---

## Q13. Seata AT 模式和 XA 事务有什么本质区别？

```
答：

锁机制不同（核心区别）：

XA（悲观锁）：
  第一阶段（Prepare）：执行SQL + 写redo log + 锁定行
  锁在第二阶段（Commit/Rollback）后才释放
  
  时间线：
  ←────── 全局事务持续时间 ──────→
  Prepare(锁) ... 等待其他分支 ... Commit(解锁)
  
  问题：锁持有时间 = 全局事务时间（可能秒级）
        高并发下锁等待严重

Seata AT（乐观锁/补偿）：
  第一阶段：执行SQL + 写undo log + 提交本地事务 + 释放锁
  第二阶段（全局提交）：删除undo log（清理）
  第二阶段（全局回滚）：执行undo log（补偿）
  
  时间线：
  LocalCommit(解锁) ... 等待其他分支 ... GlobalCommit(删undo log)
  
  本地事务很快提交，锁持有时间极短
  → 并发性好

代价（Seata 的缺点）：
  "已提交但可能回滚"状态
  全局事务确认期间，其他事务可能读到已修改的数据（脏读）
  → 通过"全局锁"保护（但全局锁本身也有性能开销）
  → 全局锁仅保护 Seata 事务间的隔离，无法保护非 Seata 事务读

适用场景对比：
  XA：    金融转账、库存扣减（强一致）
  Seata： 下单、退款（高并发，允许极短暂不一致）
```

---

## Q14. ShardingSphere 的 SQL 解析是如何实现的？

```
答：

解析实现（v5.x）：
  使用 ANTLR4 框架，基于形式语法定义 SQL 语法规则
  支持多种数据库方言（MySQL/PG/Oracle/SQLServer/openGauss）

解析流程：
  SQL 字符串
     ↓ 词法分析（Lexer）
  Token 流：[SELECT] [*] [FROM] [t_order] [WHERE] ...
     ↓ 语法分析（Parser）
  解析树（Parse Tree）
     ↓ 语义分析（Visitor）
  SQL Statement（语义模型）：
     SelectStatement {
       projections: [*],
       from: t_order,
       where: {user_id=1001}
     }

解析缓存优化：
  SQL 模板（去掉参数值）作为缓存 key
  SELECT * FROM t_order WHERE user_id = ? → 缓存 AST
  相同模板的 SQL 直接复用缓存，无需重新解析
  性能提升约 10~20 倍

支持 SQL 类型：
  DML（SELECT/INSERT/UPDATE/DELETE）
  DDL（CREATE/ALTER/DROP TABLE）→ 广播到所有分片
  DCL（GRANT/REVOKE）
  TCL（BEGIN/COMMIT/ROLLBACK）
  DAL（SHOW/EXPLAIN）

不支持的 SQL 特性：
  部分方言特有语法（如 MySQL 的 INSERT ... ON DUPLICATE KEY UPDATE 有限支持）
  复杂子查询中的分片（分片键在子查询中无法路由）
```

---

## Q15. 如何保证分库分表后的全局唯一 ID 且趋势递增？

```
答：

为什么需要趋势递增：
  MySQL InnoDB B+ 树特性：
    主键单调递增 → 每次插入都在最右叶子节点 → 顺序写
    主键随机（UUID）→ 随机插入 → 页分裂 → 碎片 → 性能下降

方案比较：

1. 雪花算法（最推荐）
   64bit = 时间戳(41) + 机器ID(10) + 序列号(12)
   全局唯一：机器ID + 时间戳 + 序列号组合保证唯一
   趋势递增：高位是时间戳，时间越大 ID 越大
   单机性能：约400万/s
   缺点：时钟回拨、WorkerID 管理成本

2. 号段模式（美团 Leaf 方案）
   批量从 DB 申请号段（如每次1000个）
   本地消费，极低延迟
   全局唯一：DB 保证号段不重叠
   趋势递增：是
   缺点：每次重启可能浪费一个号段

3. Redis INCR
   使用 Redis 原子自增
   性能高（10万/s）
   全局唯一：Redis 单线程保证
   趋势递增：是
   缺点：Redis 宕机风险（需持久化），数据迁移复杂

4. CosID（高性能雪花改进版）
   解决时钟回拨（原子序列号，不依赖时钟单调性）
   性能 > 500万/s（是普通雪花的5倍以上）
   适合极高并发场景

5. UUID（不推荐作为主键）
   全局唯一：是
   趋势递增：否！完全随机
   性能极差（大量页分裂）

面试标准答案：
  推荐雪花算法，时间戳保证趋势有序，机器ID保证分布式唯一
  生产环境通过 ZooKeeper/Redis 自动分配 WorkerID
  通过 max-vibration-offset 处理时钟回拨
  高并发场景考虑 CosID
```

---

## Q16. ShardingSphere 查询缓存（SQL 解析缓存）的原理？

```
答：

缓存对象：SQL 模板的解析结果（AST + 分片上下文）

缓存 key 生成：
  将 SQL 中的字面量替换为 ?
  "SELECT * FROM t_order WHERE user_id = 1001" 
  → key = "SELECT * FROM t_order WHERE user_id = ?"

缓存内容（SQLStatementContext）：
  - 解析好的 AST
  - 分片键的位置信息（哪个参数是 user_id）
  - 路由规则信息

执行时：
  用缓存的 AST + 当前参数值 → 直接计算路由
  跳过词法分析和语法分析（最耗时的步骤）
  
  性能提升：
    无缓存：解析耗时约 1~5ms
    有缓存：解析耗时约 0.1~0.5ms（10x提升）

缓存大小配置：
  默认缓存 65535 个不同的 SQL 模板
  # props:
  #   sql-parser-cache-size: 65535

注意事项：
  SQL 拼接（非参数化）会导致缓存失效
  // 错误：缓存 miss，每次都解析
  String sql = "WHERE user_id = " + userId;
  
  // 正确：缓存 hit，复用解析结果
  String sql = "WHERE user_id = ?";
  // 使用 PreparedStatement 绑定参数
```

---

## 附录：ShardingSphere 版本演进

```
版本演进历史：

v1.x（2015年）：
  Dangdang（当当）开源
  最初名为 Sharding-JDBC
  仅支持分库分表，无其他功能

v2.x（2016年）：
  增加读写分离
  重构路由引擎

v3.x（2018年）：
  增加 Sharding-Proxy（服务端代理）
  增加 Sharding-Sidecar（规划中）
  合并为 ShardingSphere

v4.x（2020年）：
  加入 Apache 孵化器
  大规模重构（内核独立化）
  增加分布式事务（XA/BASE）
  增加数据加密

v5.x（2021年，当前主流）：
  毕业成为 Apache 顶级项目
  内核完全重构（可插拔架构）
  增加影子库（全链路压测）
  增加 DistSQL
  增加 ShardingSphere-UI（控制台）
  支持 openGauss、PostgreSQL 等更多数据库

v5.4.x（最新稳定版，2023年）：
  性能大幅优化
  支持 CosID
  增强 ShardingSphere Scaling（在线迁移）
  增强监控（OpenTelemetry 集成）
```

---

## 附录：常用 ShardingSphere 配置速查表

```yaml
# ===== 最简分片配置 =====
spring:
  shardingsphere:
    datasource:
      names: ds_0,ds_1
      ds_0:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://localhost:3306/db_0?useSSL=false
        username: root
        password: root

      ds_1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://localhost:3307/db_1?useSSL=false
        username: root
        password: root

    rules:
      sharding:
        tables:
          t_order:
            actual-data-nodes: ds_$->{0..1}.t_order_$->{0..1}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db_mod
            table-strategy:
              standard:
                sharding-column: order_id
                sharding-algorithm-name: table_mod
            key-generate-strategy:
              column: order_id
              key-generator-name: snowflake

        sharding-algorithms:
          db_mod:
            type: MOD
            props:
              sharding-count: 2
          table_mod:
            type: MOD
            props:
              sharding-count: 2

        key-generators:
          snowflake:
            type: SNOWFLAKE
            props:
              worker-id: 1

    props:
      sql-show: true
```

---

*文档版本：Apache ShardingSphere 5.4.x*  
*最后更新：2024年*  
*作者：技术文档工程*

---
