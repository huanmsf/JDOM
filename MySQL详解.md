# MySQL 详解 —— 从原理到实战

> 本文面向有一定编程基础的开发者，系统讲解 MySQL 核心原理、性能优化与高可用方案。
> 所有示例基于 MySQL 8.0，InnoDB 存储引擎。

---

## 目录

- [Part 1: MySQL 整体架构](#part-1)
- [Part 2: InnoDB 存储引擎底层原理](#part-2)
- [Part 3: 事务与锁](#part-3)
- [Part 4: 日志系统](#part-4)
- [Part 5: 索引详解与优化](#part-5)
- [Part 6: SQL 优化实战](#part-6)
- [Part 7: MySQL 调优](#part-7)
- [Part 8: 高可用与扩展](#part-8)
- [Part 9: 完整实战案例](#part-9)
- [Part 10: 常见问题 FAQ](#part-10)

---

<a name="part-1"></a>
# Part 1: MySQL 整体架构

## 1.1 分层架构总览

MySQL 采用经典的分层架构，从上到下分为：连接层、服务层、引擎层、存储层。

```
+--------------------------------------------------+
|                   客户端应用                      |
|  (JDBC / MySQL Connector / CLI / ORM框架 等)     |
+--------------------------------------------------+
                        |
                        v
+--------------------------------------------------+
|               连接层 (Connection Layer)           |
|  - 连接管理（TCP握手、认证、SSL）                  |
|  - 线程池 / 连接池                                |
|  - 权限校验                                       |
+--------------------------------------------------+
                        |
                        v
+--------------------------------------------------+
|               服务层 (Service Layer)              |
|  +------------------------------------------+   |
|  |  SQL 接口  |  解析器  |  优化器  |  缓存  |   |
|  +------------------------------------------+   |
|  - SQL Interface: DML/DDL/存储过程/视图/触发器    |
|  - Parser: 词法分析 + 语法分析，生成解析树         |
|  - Optimizer: 基于代价的查询优化，生成执行计划      |
|  - Cache: Query Cache（8.0已废弃）               |
+--------------------------------------------------+
                        |
                        v
+--------------------------------------------------+
|           存储引擎层 (Storage Engine Layer)       |
|  +----------+  +----------+  +----------+       |
|  | InnoDB   |  | MyISAM   |  | Memory   |  ...  |
|  +----------+  +----------+  +----------+       |
|  - 插件式架构，可替换                              |
|  - 负责数据的存储和读取                            |
+--------------------------------------------------+
                        |
                        v
+--------------------------------------------------+
|               存储层 (Storage Layer)              |
|  - 文件系统：.ibd 文件、.frm 文件、日志文件        |
|  - 操作系统 I/O 接口                              |
+--------------------------------------------------+
```

## 1.2 连接层详解

### 连接建立过程

```
客户端                          MySQL Server
   |                                 |
   |-------- TCP 三次握手 ---------->|
   |<------- 握手包(版本/能力/盐值) --|
   |-------- 认证包(用户名/密码) ---->|
   |<------- OK/ERR 包 -------------|
   |                                 |
   |  (连接建立，分配线程)            |
```

### 连接池的作用

每次建立 TCP 连接都有较大开销（三次握手 + 认证 + 线程分配）。连接池通过**复用已有连接**来避免这种开销。

```
应用层连接池 (HikariCP / Druid)
  ┌─────────────────────────────────┐
  │  空闲连接队列: [conn1][conn2]... │
  │  活跃连接:     [conn3]→执行中   │
  └─────────────────────────────────┘
        ↕ 复用，不重新握手
  MySQL Server 线程池
  ┌─────────────────────────────────┐
  │  线程1: 处理 conn1              │
  │  线程2: 处理 conn2              │
  └─────────────────────────────────┘
```

关键参数：
```ini
max_connections = 1000        # 最大连接数
wait_timeout = 28800          # 空闲连接超时（秒）
interactive_timeout = 28800   # 交互连接超时（秒）
```

## 1.3 服务层详解

### SQL 解析过程

以 `SELECT name FROM users WHERE id = 1` 为例：

**词法分析**：将 SQL 字符串切分为 Token
```
[SELECT] [name] [FROM] [users] [WHERE] [id] [=] [1]
```

**语法分析**：根据语法规则构建解析树（AST）
```
         SELECT
        /      \
    字段列表    FROM子句
      |           |
    [name]      users
                  |
               WHERE子句
               /    \
             id      =
                      \
                       1
```

**语义分析**：检查表名、字段名是否存在，权限是否足够。

### 查询优化器

优化器负责将解析树转换为**执行计划**，核心工作：

1. **逻辑优化**：等价变换（谓词下推、子查询展开等）
2. **物理优化**：选择访问路径（全表扫描 vs 索引扫描）、Join 顺序、Join 算法

```sql
-- 原始 SQL
SELECT * FROM orders o JOIN users u ON o.user_id = u.id
WHERE u.city = 'Beijing' AND o.status = 1;

-- 优化器可能的决策：
-- 1. 先过滤 users 表中 city='Beijing' 的行（谓词下推）
-- 2. 以过滤后的 users 作为驱动表
-- 3. 用 orders.user_id 上的索引做 Nested Loop Join
```

## 1.4 InnoDB vs MyISAM 对比

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务支持 | 支持（ACID） | 不支持 |
| 行级锁 | 支持 | 不支持（表锁） |
| 外键 | 支持 | 不支持 |
| 崩溃恢复 | 支持（redo log） | 不支持 |
| 全文索引 | 支持（5.6+） | 支持 |
| 聚簇索引 | 有（主键） | 无（堆表） |
| count(*) | 慢（需扫描） | 快（维护了计数） |
| 适用场景 | OLTP、高并发读写 | 只读/统计场景 |

> **结论**：现代应用几乎全部使用 InnoDB，MyISAM 仅用于极少数特殊场景。

## 1.5 一条 SQL 的完整执行流程

```
用户执行: SELECT * FROM users WHERE id = 100;

Step 1: 连接管理
  └─ 从连接池获取连接，验证权限

Step 2: 查询缓存（MySQL 8.0 已废弃）
  └─ 计算 SQL hash，查找缓存（未命中则继续）

Step 3: 解析器
  └─ 词法分析 → 语法分析 → 生成解析树

Step 4: 预处理器
  └─ 检查表/列是否存在，展开 * 为具体列名

Step 5: 优化器
  └─ 生成多个执行计划 → 估算代价 → 选最优计划
  └─ 决策：使用主键索引，等值查找

Step 6: 执行器
  └─ 调用 InnoDB 存储引擎接口
  └─ 引擎通过 B+ 树定位到 id=100 的记录
  └─ 返回记录给执行器

Step 7: 返回结果
  └─ 执行器将结果集返回给客户端
```

---

<a name="part-2"></a>
# Part 2: InnoDB 存储引擎底层原理

## 2.1 InnoDB 内存架构

```
+=====================================================+
|              InnoDB 内存结构                         |
+=====================================================+
|                                                     |
|  +-----------------------------------------------+ |
|  |           Buffer Pool (缓冲池)                 | |
|  |  +----------+  +----------+  +----------+    | |
|  |  | 数据页   |  | 索引页   |  | 锁信息   |    | |
|  |  +----------+  +----------+  +----------+    | |
|  |  +----------+  +----------+                  | |
|  |  |插入缓冲  |  | 自适应   |                  | |
|  |  |(Change   |  | 哈希索引 |                  | |
|  |  | Buffer)  |  |          |                  | |
|  |  +----------+  +----------+                  | |
|  +-----------------------------------------------+ |
|                                                     |
|  +------------------+  +----------------------+    |
|  |  Log Buffer      |  |  Additional Memory   |    |
|  |  (redo log缓冲)  |  |  Pool                |    |
|  +------------------+  +----------------------+    |
|                                                     |
+=====================================================+
```

## 2.2 Buffer Pool（缓冲池）

Buffer Pool 是 InnoDB **最重要的内存区域**，用于缓存磁盘上的数据页和索引页，减少磁盘 I/O。

### 工作原理

```
读操作：
  请求数据页
      ↓
  检查 Buffer Pool
      ↓
  命中？──是──→ 直接返回（内存速度）
      ↓否
  从磁盘读取 16KB 页
      ↓
  放入 Buffer Pool
      ↓
  返回数据

写操作：
  修改数据
      ↓
  在 Buffer Pool 中修改（脏页）
      ↓
  写入 redo log（WAL）
      ↓
  后台线程异步刷盘（checkpoint）
```

### LRU 链表管理

InnoDB 使用改进的 LRU（Least Recently Used）算法管理 Buffer Pool，防止大查询（全表扫描）污染缓存。

```
传统 LRU:                    InnoDB 改进 LRU:
                             
[最新]                       [最新] ← New Sublist (5/8)
  ↑                            ↑
  |                          midpoint (3/8处)
  ↓                            ↓
[最旧] → 淘汰                [最旧] ← Old Sublist (3/8)
                               ↓ → 淘汰

全表扫描的页只进入 Old Sublist，
不会驱逐 New Sublist 中的热数据
```

关键参数：
```ini
innodb_buffer_pool_size = 8G          # 建议设为物理内存的 50%~75%
innodb_buffer_pool_instances = 8      # 并行实例数，减少锁竞争
innodb_old_blocks_time = 1000         # 页在 old 区停留多久才能移入 new 区(ms)
```

### 如何查看 Buffer Pool 使用情况

```sql
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- 关键指标：
-- Innodb_buffer_pool_read_requests: 逻辑读次数
-- Innodb_buffer_pool_reads:         物理读次数（真正读磁盘）
-- 缓存命中率 = 1 - (物理读/逻辑读) ，理想值 > 99%

SELECT 
  (1 - (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
        WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
       (SELECT VARIABLE_VALUE FROM performance_schema.global_status 
        WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
  ) * 100 AS buffer_pool_hit_rate;
```

## 2.3 Change Buffer（插入缓冲）

Change Buffer 用于**缓存对非唯一二级索引的写操作**（INSERT/UPDATE/DELETE），当对应数据页不在 Buffer Pool 中时，先缓存变更，等页被读入时再合并。

### 为什么需要 Change Buffer

```
场景：INSERT 一条记录，需要更新多个二级索引

没有 Change Buffer:
  更新索引A → 读磁盘(二级索引A的页) → 修改 → 写回
  更新索引B → 读磁盘(二级索引B的页) → 修改 → 写回
  (多次随机 I/O)

有 Change Buffer:
  更新索引A → 写入 Change Buffer (顺序 I/O)
  更新索引B → 写入 Change Buffer (顺序 I/O)
  等索引页被读入时批量合并 (延迟随机 I/O)
```

> 注意：唯一索引不能使用 Change Buffer，因为需要读取旧值来判断唯一性。

```ini
innodb_change_buffer_max_size = 25   # Change Buffer 占 Buffer Pool 的最大比例（%）
```

## 2.4 页（Page）结构

InnoDB 以**页（Page）**为基本 I/O 单位，默认 16KB。

```
+------------------------------------------+
|        InnoDB 数据页 (16KB)               |
+------------------------------------------+
| File Header (38字节)                      |
|  - 页号、上/下页指针、校验和、LSN 等       |
+------------------------------------------+
| Page Header (56字节)                      |
|  - 槽数量、堆顶指针、记录数、页目录等      |
+------------------------------------------+
| Infimum + Supremum Records               |
|  - 虚拟最小/最大记录，形成链表边界         |
+------------------------------------------+
| User Records (用户记录区)                  |
|  - 按主键顺序存储的行记录                  |
|  - 单向链表连接                            |
+------------------------------------------+
| Free Space (空闲空间)                      |
+------------------------------------------+
| Page Directory (页目录)                   |
|  - 稀疏索引，用于页内二分查找              |
|  - 每个槽代表4~8条记录                    |
+------------------------------------------+
| File Trailer (8字节)                      |
|  - 校验和，用于检测页损坏                  |
+------------------------------------------+
```

## 2.5 B+ 树索引结构

### B 树 vs B+ 树

```
B 树（B-Tree）：
  - 内部节点和叶子节点都存储数据
  - 搜索可能在中间节点就结束
  - 不支持范围查询的顺序遍历

         [30]
        /    \
    [10,20]  [40,50]
    /  |  \   |   \
  [5] [15] [25] [35] [45]
   ↑数据存在各层节点

B+ 树（B+Tree）：
  - 内部节点只存储键值，作为路由
  - 所有数据只存在叶子节点
  - 叶子节点通过双向链表连接（支持范围查询）

         [30]
        /    \
    [10,20]  [40,50]
    /  |  \    |   \
  [5][15][25][35][45]  ← 叶子节点（存数据）
   ↔  ↔   ↔   ↔   ↔   ← 双向链表
```

### InnoDB B+ 树特点

- 每个节点对应一个**数据页（16KB）**
- 非叶节点（索引页）存储：键值 + 子页指针
- 叶子节点（数据页）存储：完整行记录（聚簇索引）或主键值（二级索引）
- 树高通常为 **3~4 层**，可存储千万级数据

```
层高估算：
- 每个索引页可存 ~1000 个指针
- 每个叶子页可存 ~100 行记录（假设每行160字节）

2层B+树: 1000 * 100 = 10万行
3层B+树: 1000 * 1000 * 100 = 1亿行
4层B+树: 1000 * 1000 * 1000 * 100 = 1000亿行

→ 3层基本满足大多数业务需求
```

## 2.6 聚簇索引与二级索引

### 聚簇索引（Clustered Index）

```
聚簇索引 = 主键索引 = 完整的行数据

         [主键B+树]
              |
      ________|________
     |                 |
   [PK=100]          [PK=200]
   完整行数据          完整行数据
   name='Alice'       name='Bob'
   age=25             age=30
   ...                ...
```

- 每张表只有**一个**聚簇索引
- 如果没有显式主键，InnoDB 使用 UNIQUE NOT NULL 列，或内部生成 6 字节的 row_id
- **推荐使用自增主键**：顺序插入，减少页分裂

### 二级索引（Secondary Index）

```
二级索引 = 非主键索引，叶子节点存储主键值

         [name索引B+树]
              |
      ________|_________
     |                  |
  ['Alice']           ['Bob']
   PK=100              PK=200
     ↓ 回表              ↓ 回表
  聚簇索引             聚簇索引
  (取完整数据)         (取完整数据)
```

### 回表查询

```sql
-- 表结构
CREATE TABLE users (
  id INT PRIMARY KEY,
  name VARCHAR(50),
  age INT,
  INDEX idx_name (name)
);

-- 查询
SELECT id, name, age FROM users WHERE name = 'Alice';

-- 执行过程：
-- 1. 走 idx_name 索引，找到 name='Alice' → 得到 PK=100
-- 2. 用 PK=100 再查聚簇索引（回表）→ 得到 age=25
-- （共两次B+树查询）
```

### 覆盖索引（避免回表）

```sql
-- 创建联合索引，包含查询所需的所有列
ALTER TABLE users ADD INDEX idx_name_age (name, age);

-- 查询
SELECT id, name, age FROM users WHERE name = 'Alice';

-- 执行过程：
-- 1. 走 idx_name_age 索引，找到 name='Alice' → 叶子节点已有 (name, age, PK)
-- 2. 不需要回表！（索引覆盖了所有需要的列）
-- EXPLAIN 中 Extra 显示: Using index
```

## 2.7 行格式（Row Format）

InnoDB 支持 4 种行格式：

| 格式 | 特点 | 适用场景 |
|------|------|---------|
| COMPACT | 变长字段长度列表，NULL标志位 | MySQL 5.0 默认 |
| REDUNDANT | 完整的字段长度列表，兼容性好 | 旧版本 |
| DYNAMIC | 处理长字段更好（存溢出页） | MySQL 5.7+ 默认 |
| COMPRESSED | 在 DYNAMIC 基础上加压缩 | 存储空间敏感场景 |

```sql
-- 查看表的行格式
SHOW TABLE STATUS LIKE 'users'\G

-- 创建时指定
CREATE TABLE t (id INT) ROW_FORMAT=DYNAMIC;
```

---

<a name="part-3"></a>
# Part 3: 事务与锁

## 3.1 ACID 特性

| 特性 | 含义 | InnoDB 实现 |
|------|------|-------------|
| Atomicity（原子性） | 事务要么全成功，要么全失败 | undo log 回滚 |
| Consistency（一致性） | 事务前后数据满足业务约束 | 原子性+隔离性+持久性共同保障 |
| Isolation（隔离性） | 并发事务互不干扰 | MVCC + 锁 |
| Durability（持久性） | 提交后数据永久保存 | redo log + 刷盘 |

## 3.2 事务隔离级别

### 四种隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 |
|---------|------|-----------|------|
| READ UNCOMMITTED | 可能 | 可能 | 可能 |
| READ COMMITTED | 不可能 | 可能 | 可能 |
| REPEATABLE READ | 不可能 | 不可能 | InnoDB解决 |
| SERIALIZABLE | 不可能 | 不可能 | 不可能 |

> InnoDB 默认隔离级别是 **REPEATABLE READ**，且通过间隙锁解决了幻读问题。

### 脏读、不可重复读、幻读的区别

```
脏读：读到了其他事务未提交的数据
  事务A                    事务B
  BEGIN;                   BEGIN;
                           UPDATE users SET age=30 WHERE id=1;
  SELECT age FROM users    -- 读到 age=30（脏读！B还未提交）
  WHERE id=1; → 30
                           ROLLBACK; -- B回滚了

不可重复读：同一事务内两次读取同一数据，结果不同
  事务A                    事务B
  BEGIN;
  SELECT age → 25;         BEGIN;
                           UPDATE users SET age=30 WHERE id=1;
                           COMMIT;
  SELECT age → 30;  -- 两次结果不同（不可重复读）
  COMMIT;

幻读：同一事务内两次范围查询，第二次多了（或少了）行
  事务A                    事务B
  BEGIN;
  SELECT * WHERE age>20    BEGIN;
  → 2行                    INSERT INTO users(age) VALUES(25);
                           COMMIT;
  SELECT * WHERE age>20
  → 3行  -- 多了一行（幻读）
  COMMIT;
```

### 设置隔离级别

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置会话级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 设置全局级别（需重新连接生效）
SET GLOBAL TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

## 3.3 MVCC（多版本并发控制）

MVCC 是 InnoDB 实现**快照读（非锁定读）**的核心机制，让读操作不阻塞写操作。

### 核心组件

**1. 隐藏列**

每行记录有 3 个隐藏列：
```
+--------+--------+-----+----------+-----------+
| 用户数据 | 用户数据 | ... | trx_id   | roll_ptr  |
|        |        |     | 最近修改  | 指向undo  |
|        |        |     | 的事务ID  | log的指针 |
+--------+--------+-----+----------+-----------+
```
- `trx_id`：最近一次修改该行的事务 ID
- `roll_ptr`：指向 undo log 中上一版本的指针

**2. undo log 版本链**

```
当前版本 (trx_id=100, age=30)
     ↓ roll_ptr
旧版本1 (trx_id=50,  age=25)
     ↓ roll_ptr
旧版本2 (trx_id=20,  age=20)
     ↓ roll_ptr
    NULL
```

**3. Read View（读视图）**

Read View 决定当前事务能看到哪个版本的数据，包含：
```
Read View {
  m_ids:      [当前活跃（未提交）的事务ID列表]
  min_trx_id: m_ids 中最小的事务ID
  max_trx_id: 下一个要分配的事务ID
  creator_trx_id: 创建该 Read View 的事务ID
}
```

**可见性判断规则**：
```
对于版本链上的某一版本，其 trx_id 为 V：

if V == creator_trx_id:
    可见（自己修改的，当然能看到）
elif V < min_trx_id:
    可见（在 Read View 创建之前已提交）
elif V >= max_trx_id:
    不可见（在 Read View 创建之后才开始的事务）
elif V in m_ids:
    不可见（创建 Read View 时还未提交）
else:
    可见（已提交的事务）
```

### READ COMMITTED vs REPEATABLE READ 的区别

```
READ COMMITTED：
  每次执行 SELECT，都生成新的 Read View
  → 能读到其他事务已提交的最新数据（不可重复读）

REPEATABLE READ：
  事务中第一次执行 SELECT 时生成 Read View，之后复用
  → 整个事务看到的是"快照"，不受其他事务影响（可重复读）
```

## 3.4 锁机制

### 锁的分类

```
按粒度：
  表级锁 ── 锁整张表，并发低
  行级锁 ── 锁某行，并发高（InnoDB 主要使用）
  页级锁 ── 介于两者之间（BerkeleyDB）

按模式：
  共享锁（S锁，读锁）── 多个事务可同时持有
  排他锁（X锁，写锁）── 独占，与其他锁互斥

按行锁类型（InnoDB）：
  Record Lock  ── 锁定单条记录
  Gap Lock     ── 锁定记录间的间隙（防止插入，解决幻读）
  Next-Key Lock── Record Lock + Gap Lock（锁记录及其左侧间隙）
  Insert Intention Lock ── 插入意向锁（特殊间隙锁）
```

### 意向锁（表级）

```
意向共享锁（IS）：事务要在行上加 S 锁，先加 IS 表锁
意向排他锁（IX）：事务要在行上加 X 锁，先加 IX 表锁

作用：使表级锁和行级锁能高效共存
  （判断表上是否有行锁，不用逐行检查）

兼容性矩阵：
       IS   IX   S    X
IS    兼容  兼容  兼容  冲突
IX    兼容  兼容  冲突  冲突
S     兼容  冲突  兼容  冲突
X     冲突  冲突  冲突  冲突
```

### Next-Key Lock 解决幻读

```sql
-- 假设 users 表有 id: 1, 5, 10, 15

SELECT * FROM users WHERE id BETWEEN 5 AND 10 FOR UPDATE;

-- InnoDB 会加 Next-Key Locks：
-- (-∞, 1]  Record Lock on 1（实际是根据查询条件）
-- (1,  5]  Next-Key Lock
-- (5, 10]  Next-Key Lock
-- (10, 15) Gap Lock

-- 其他事务无法在 (5,10] 范围内插入任何记录
-- → 防止幻读
```

### 加锁示例

```sql
-- 共享锁（S锁）
SELECT * FROM users WHERE id = 1 LOCK IN SHARE MODE;
-- 或（MySQL 8.0）
SELECT * FROM users WHERE id = 1 FOR SHARE;

-- 排他锁（X锁）
SELECT * FROM users WHERE id = 1 FOR UPDATE;

-- DML 操作自动加 X 锁
UPDATE users SET age = 25 WHERE id = 1;
DELETE FROM users WHERE id = 1;
```

## 3.5 死锁

### 死锁产生原因

```
事务A                           事务B
BEGIN;                          BEGIN;
UPDATE users SET age=25         UPDATE orders SET status=1
WHERE id=1;  -- 锁定 users.1    WHERE id=100; -- 锁定 orders.100

UPDATE orders SET status=1      UPDATE users SET age=30
WHERE id=100; -- 等待B释放       WHERE id=1; -- 等待A释放

  ← 死锁！互相等待 →
```

### 死锁检测与处理

InnoDB 有**自动死锁检测**（innodb_deadlock_detect=ON），检测到死锁后：
1. 选择代价较小的事务作为"牺牲者"
2. 回滚该事务，释放锁
3. 另一个事务得以继续执行

```sql
-- 查看最近的死锁信息
SHOW ENGINE INNODB STATUS\G
-- 输出中找 LATEST DETECTED DEADLOCK 部分

-- 查看锁等待情况
SELECT * FROM performance_schema.data_lock_waits\G
SELECT * FROM performance_schema.data_locks\G
```

### 避免死锁的最佳实践

```
1. 固定加锁顺序：所有事务按相同顺序访问表/行
   (总是先锁 users，再锁 orders)

2. 减小事务范围：尽量缩短事务持锁时间

3. 使用低级别隔离：READ COMMITTED 不使用间隙锁

4. 避免大事务：拆分为多个小事务

5. 合理使用索引：走索引的行锁比全表扫描锁少得多

6. 设置锁等待超时：
   SET innodb_lock_wait_timeout = 5; -- 5秒超时
```

---

<a name="part-4"></a>
# Part 4: 日志系统

## 4.1 三大日志概览

```
+----------------------------------------------------------+
|                    MySQL 日志体系                         |
+----------------------------------------------------------+
|                                                          |
|  redo log (重做日志)                                      |
|  ├─ 位置: InnoDB 存储引擎层                               |
|  ├─ 作用: 保证事务持久性（Durability）                    |
|  ├─ 格式: 物理日志（记录页的修改）                         |
|  └─ 文件: ib_logfile0, ib_logfile1（循环写入）            |
|                                                          |
|  undo log (回滚日志)                                      |
|  ├─ 位置: InnoDB 存储引擎层                               |
|  ├─ 作用: 事务回滚 + MVCC 版本链                          |
|  ├─ 格式: 逻辑日志（记录反向操作）                         |
|  └─ 文件: 系统表空间 / undo 表空间                         |
|                                                          |
|  binlog (二进制日志)                                      |
|  ├─ 位置: MySQL Server 层                                 |
|  ├─ 作用: 数据备份、主从复制、数据恢复                     |
|  ├─ 格式: 逻辑日志（STATEMENT/ROW/MIXED）                 |
|  └─ 文件: mysql-bin.000001, mysql-bin.000002, ...        |
|                                                          |
+----------------------------------------------------------+
```

## 4.2 redo log（重做日志）

### WAL（Write-Ahead Logging）

WAL 是 InnoDB 实现持久性的核心思想：**先写日志，再刷磁盘**。

```
没有 WAL：
  修改数据 → 立即刷盘（随机写，慢）
  如果刷盘一半崩溃 → 数据损坏

有 WAL（redo log）：
  修改数据 → 写 redo log（顺序写，快）→ 修改内存
  崩溃后恢复 → 重放 redo log → 数据一致

顺序写 vs 随机写的性能差异：
  机械硬盘：顺序写 ~100MB/s，随机写 ~1MB/s（相差100倍）
  SSD：     顺序写 ~500MB/s，随机写 ~100MB/s（相差5倍）
```

### redo log 的循环写入

```
redo log 文件组成（例：2个文件，各512MB）：

ib_logfile0        ib_logfile1
+----------------+ +----------------+
|                | |                |
|  write pos→    | |                |
|                | |                |
+----------------+ +----------------+
        ↑ 循环写入，write pos 追着 checkpoint 跑

write pos: 当前写入位置
checkpoint: 已经刷盘的位置（可以被覆盖）

如果 write pos == checkpoint：
  → redo log 满了，必须等刷盘（会影响性能）
  → 解决：增大 innodb_log_file_size
```

### redo log 配置

```ini
innodb_log_file_size = 1G        # 每个日志文件大小
innodb_log_files_in_group = 2    # 日志文件数量
innodb_flush_log_at_trx_commit = 1  # 刷盘策略（关键参数）
```

### innodb_flush_log_at_trx_commit 详解

```
值=0: 每秒刷盘一次（性能最好，最多丢1秒数据）
  提交事务 → 写 log buffer → 每秒刷到 OS cache + 磁盘

值=1: 每次提交都刷盘（默认，最安全）
  提交事务 → 写 log buffer → fsync 到磁盘
  (即使系统崩溃，数据也不丢)

值=2: 提交时写到 OS cache，每秒 fsync（折中）
  提交事务 → 写 log buffer → 写到 OS cache
  → 每秒 fsync 到磁盘
  (MySQL 崩溃不丢数据，OS 崩溃最多丢1秒)
```

## 4.3 undo log（回滚日志）

### 作用

```
1. 事务回滚
   ROLLBACK 时，根据 undo log 执行反向操作：
   - INSERT → DELETE
   - DELETE → INSERT
   - UPDATE → UPDATE（恢复旧值）

2. MVCC 版本链（见 Part 3.3）
   undo log 构成行的历史版本链
```

### undo log 类型

```sql
-- INSERT undo log（事务提交后可立即删除）
INSERT INTO users(id, name) VALUES(1, 'Alice');
-- undo: DELETE FROM users WHERE id=1

-- UPDATE undo log（需要保留用于 MVCC，purge 线程异步删除）
UPDATE users SET name='Bob' WHERE id=1;
-- undo: UPDATE users SET name='Alice' WHERE id=1
```

### undo log 清理

```
purge 线程：后台线程，定期清理不再被任何事务需要的 undo log

如果有长事务（长时间不提交的事务）：
  → undo log 无法清理（其他事务可能需要读历史版本）
  → undo 表空间膨胀

查看长事务：
SELECT * FROM information_schema.INNODB_TRX
WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 60
ORDER BY trx_started;
```

## 4.4 binlog（二进制日志）

### binlog 的三种格式

```
STATEMENT（语句模式）：
  记录原始 SQL 语句
  优点：日志量小
  缺点：某些函数（NOW(), UUID()等）可能导致主从不一致

ROW（行模式，推荐）：
  记录每行数据的变更（before + after）
  优点：精确，主从一致
  缺点：日志量大

MIXED（混合模式）：
  默认用 STATEMENT，遇到不安全语句自动切换为 ROW
```

```ini
binlog_format = ROW                 # 推荐
binlog_row_image = FULL             # 记录完整行（默认）
sync_binlog = 1                     # 每次提交都同步到磁盘（推荐）
expire_logs_days = 7                # binlog 保留天数（MySQL 8.0 用 binlog_expire_logs_seconds）
```

### binlog 操作

```sql
-- 查看 binlog 文件列表
SHOW BINARY LOGS;

-- 查看当前正在写入的 binlog 位置
SHOW MASTER STATUS;

-- 查看 binlog 内容
SHOW BINLOG EVENTS IN 'mysql-bin.000001' LIMIT 20;

-- 命令行工具查看
-- mysqlbinlog mysql-bin.000001

-- 基于 binlog 恢复数据（指定时间范围）
-- mysqlbinlog --start-datetime="2024-01-01 10:00:00" \
--             --stop-datetime="2024-01-01 11:00:00" \
--             mysql-bin.000001 | mysql -u root -p
```

## 4.5 两阶段提交（2PC）

两阶段提交保证 redo log 和 binlog 的一致性。

### 为什么需要两阶段提交

```
场景：执行 UPDATE users SET age=30 WHERE id=1;

如果先写 redo log，再写 binlog：
  写完 redo log → 崩溃 → binlog 没写
  用 binlog 恢复从库 → 从库没有这次更新
  → 主从不一致！

如果先写 binlog，再写 redo log：
  写完 binlog → 崩溃 → redo log 没写
  主库重启后回滚这次更新 → 主库没有这次更新
  但从库通过 binlog 已经有了 → 主从不一致！
```

### 两阶段提交流程

```
事务提交过程：

Step 1: Prepare 阶段
  ├─ 将 redo log 状态设为 prepare
  └─ 将 trx_id 写入 redo log

Step 2: 写 binlog

Step 3: Commit 阶段
  └─ 将 redo log 状态设为 commit
      写入 commit 标记

崩溃恢复逻辑：
  - redo log 是 prepare，binlog 完整 → 提交（补写 commit）
  - redo log 是 prepare，binlog 不完整 → 回滚
  - redo log 是 commit → 直接提交
```

```
时序图：

InnoDB                         Binlog
  |                               |
  |-- 写 redo log (prepare) ---→  |
  |                               |-- 写 binlog ----→ 磁盘
  |-- 写 redo log (commit) ----→  |
  |-- 刷新 Buffer Pool (后台) -→  |
```

### MySQL 8.0 组提交优化

```
传统提交：
  事务1提交 → fsync(redo) → 写binlog → fsync(binlog) → 结束
  事务2提交 → fsync(redo) → 写binlog → fsync(binlog) → 结束
  (串行，每个事务都需要2次fsync)

组提交（Group Commit）：
  事务1  事务2  事务3
   ↓      ↓      ↓
   └──────┴──────┘
          ↓
   批量 fsync(redo) ← 一次I/O
          ↓
   批量写 binlog
          ↓
   批量 fsync(binlog) ← 一次I/O
  (并行，N个事务只需2次fsync，吞吐量大幅提升)
```

---

<a name="part-5"></a>
# Part 5: 索引详解与优化

## 5.1 索引类型

### 按数据结构分

| 类型 | 引擎支持 | 特点 |
|------|---------|------|
| B+ 树索引 | InnoDB/MyISAM | 范围查询，排序，默认类型 |
| 哈希索引 | Memory/InnoDB自适应 | 等值查询极快，不支持范围 |
| 全文索引 | InnoDB/MyISAM | FULLTEXT，中文需分词插件 |
| 空间索引 | MyISAM/InnoDB | R-Tree，GIS数据 |

### 按逻辑功能分

```sql
-- 普通索引
CREATE INDEX idx_name ON users(name);

-- 唯一索引（允许NULL）
CREATE UNIQUE INDEX idx_email ON users(email);

-- 主键索引（唯一+非NULL）
ALTER TABLE users ADD PRIMARY KEY (id);

-- 联合索引（复合索引）
CREATE INDEX idx_name_age ON users(name, age);

-- 前缀索引（节省空间）
CREATE INDEX idx_title ON articles(title(20));

-- 覆盖索引（不是单独创建，而是指索引包含查询需要的所有列）
-- 例：idx_name_age 对于 SELECT name,age WHERE name=? 就是覆盖索引
```

## 5.2 最左前缀原则

联合索引的核心规则：**查询条件必须从索引的最左列开始，不能跳过**。

```sql
-- 联合索引: idx_a_b_c (a, b, c)

-- 能走索引的情况：
WHERE a = 1                    -- 走 a
WHERE a = 1 AND b = 2          -- 走 a, b
WHERE a = 1 AND b = 2 AND c=3  -- 走 a, b, c
WHERE a = 1 AND c = 3          -- 只走 a（跳过了b，c无法用到）
WHERE a = 1 AND b > 2          -- 走 a, b（范围后的列不走索引）
WHERE a = 1 AND b > 2 AND c=3  -- 走 a, b（c不走，因为b是范围）
WHERE a LIKE 'abc%'            -- 走 a（前缀匹配）

-- 不能走索引的情况：
WHERE b = 2                    -- 没有从a开始
WHERE c = 3                    -- 没有从a开始
WHERE b = 2 AND c = 3          -- 没有从a开始
WHERE a LIKE '%abc'            -- 前缀不确定，无法走索引

-- 优化器会调整顺序（等值条件）：
WHERE b = 2 AND a = 1          -- 优化器重排为 a=1 AND b=2，走 a,b
WHERE c=3 AND a=1 AND b=2      -- 优化器重排，走 a,b,c
```

### 索引选择性

选择性越高，索引越有效（扫描的行越少）。

```sql
-- 计算列的选择性
SELECT 
  COUNT(DISTINCT name) / COUNT(*) AS name_selectivity,
  COUNT(DISTINCT age) / COUNT(*) AS age_selectivity
FROM users;

-- 联合索引列的顺序：选择性高的列放前面
-- 例如选择性: name(0.95) > age(0.03)
-- 建议: INDEX(name, age) 而非 INDEX(age, name)
```

## 5.3 索引失效的 10 种场景

```sql
-- 准备：CREATE INDEX idx_name_age ON users(name, age);
--       CREATE INDEX idx_age ON users(age);

-- 1. 对索引列使用函数
SELECT * FROM users WHERE LEFT(name, 3) = 'Ali';  -- 失效
SELECT * FROM users WHERE name LIKE 'Ali%';        -- 有效（改写）

-- 2. 对索引列进行运算
SELECT * FROM users WHERE age + 1 = 26;    -- 失效
SELECT * FROM users WHERE age = 25;         -- 有效

-- 3. 隐式类型转换
-- name 是 VARCHAR，传入整数
SELECT * FROM users WHERE name = 123;       -- 失效（字符串转数字）
SELECT * FROM users WHERE name = '123';     -- 有效

-- 4. 使用 OR 连接不同索引列（其中一个没索引）
SELECT * FROM users WHERE name='Alice' OR phone='13800';  
-- 若 phone 无索引，整个查询走全表扫描
-- 解决：给 phone 加索引，或用 UNION 代替 OR

-- 5. 不等于（!=, <>）
SELECT * FROM users WHERE age != 25;    -- 可能失效（取决于数据分布）
-- 若不等于的行占大多数，优化器选择全表扫描

-- 6. IS NOT NULL
SELECT * FROM users WHERE name IS NOT NULL;  -- 可能失效

-- 7. LIKE 以通配符开头
SELECT * FROM users WHERE name LIKE '%Ali%';  -- 失效
SELECT * FROM users WHERE name LIKE 'Ali%';   -- 有效

-- 8. 全表扫描代价更低（小表）
-- 表只有100行，即使有索引，优化器也可能选全表扫描

-- 9. 范围查询后的列（联合索引）
SELECT * FROM users WHERE name='Alice' AND age>20 AND score=90;
-- name走索引，age走索引（范围），score不走索引

-- 10. 违反最左前缀
SELECT * FROM users WHERE age=25;  -- 不走 idx_name_age
```

## 5.4 EXPLAIN 详解

EXPLAIN 是分析 SQL 执行计划的核心工具。

```sql
EXPLAIN SELECT u.name, COUNT(o.id) 
FROM users u 
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.city = 'Beijing'
GROUP BY u.id;
```

### EXPLAIN 输出字段详解

```
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key      | key_len | ref   | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | u     | NULL       | ref  | idx_city      | idx_city | 83      | const |  150 |   100.00 | Using index |
|  1 | SIMPLE      | o     | NULL       | ref  | idx_user_id   | idx_user_id | 4    | u.id  | 3    |   100.00 | NULL        |
+----+-------------+-------+------------+------+---------------+----------+---------+-------+------+----------+-------------+
```

**id**：查询序号，相同 id 表示同一查询层级，id 越大越先执行

**select_type**：
```
SIMPLE          简单查询（无子查询/UNION）
PRIMARY         最外层查询
SUBQUERY        子查询
DERIVED         FROM子句的子查询（派生表）
UNION           UNION中第二个及以后的查询
UNION RESULT    UNION的结果集
```

**type（访问类型，从优到劣）**：
```
system  只有一行（系统表）
const   通过主键/唯一索引等值查询，最多一行  ← 理想
eq_ref  联接时使用主键/唯一索引             ← 很好
ref     使用非唯一索引等值查询              ← 好
range   索引范围查询（>, <, BETWEEN, IN）  ← 还行
index   全索引扫描（比全表扫描好一点）       ← 不好
ALL     全表扫描                           ← 最差，需优化！
```

**key**：实际使用的索引，NULL 表示没用索引

**key_len**：使用的索引长度（字节数），越长说明用到的索引部分越多
```
-- 计算 key_len：
-- INT: 4字节
-- BIGINT: 8字节
-- VARCHAR(n) utf8mb4: n*4 + 2字节（变长）
-- 允许NULL的列额外 +1字节
```

**rows**：优化器估算需要扫描的行数（不是精确值）

**Extra（重要）**：
```
Using index         覆盖索引，不需要回表（好！）
Using where         WHERE过滤（在Server层，非引擎层）
Using index condition  索引条件下推（ICP），在引擎层过滤
Using filesort      需要额外排序（考虑加索引优化）
Using temporary     使用临时表（GROUP BY/DISTINCT 时）
Using join buffer   Join时缓冲区（可能需要优化Join）
```

### EXPLAIN ANALYZE（MySQL 8.0+）

```sql
-- 实际执行并显示真实耗时
EXPLAIN ANALYZE SELECT * FROM users WHERE name LIKE 'Ali%';

-- 输出示例：
-- -> Index range scan on users using idx_name (cost=2.51 rows=7) 
--    (actual time=0.065..0.087 rows=3 loops=1)
```

## 5.5 慢查询日志

```ini
# my.cnf 配置
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1        # 超过1秒记录
log_queries_not_using_indexes = ON  # 不走索引也记录
```

```sql
-- 运行时开启
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;

-- 查看慢查询统计
SHOW GLOBAL STATUS LIKE 'Slow_queries';
```

### 使用 pt-query-digest 分析慢查询日志

```bash
# 安装 Percona Toolkit
# 分析慢查询日志（Top 10 最慢的查询）
pt-query-digest /var/log/mysql/slow.log | head -200

# 输出示例：
# # Query 1: 0.50 QPS, 2.50x concurrency, ID 0xABC123 at byte 12345
# # This item is included in the report because it matches --limit.
# # Scores: V/M = 0.02
# # Time range: 2024-01-01 10:00:00 to 2024-01-01 11:00:00
# # Attribute    pct   total     min     max     avg     95%  stddev  median
# # ============ === ======= ======= ======= ======= ======= ======= =======
# # Count         15    1800
# # Exec time     50   3600s     1s      5s      2s      4s    0.5s    2s
# # Lock time      2    144s   0.1s    0.3s    0.1s    0.2s   0.05s   0.1s
# # Rows sent     25   4500k    100    5000   2500   4800    800   2400
# # Rows examine  70 126000k   1000  100000  70000  95000  15000  68000
```

---

<a name="part-6"></a>
# Part 6: SQL 优化实战

## 6.1 SELECT 优化

### 只查需要的列

```sql
-- 差：SELECT *（传输无用数据，无法使用覆盖索引）
SELECT * FROM users WHERE name = 'Alice';

-- 好：明确列名
SELECT id, name, email FROM users WHERE name = 'Alice';
```

### 避免在 WHERE 中对索引列做运算

```sql
-- 差（索引失效）
SELECT * FROM orders WHERE YEAR(created_at) = 2024;
SELECT * FROM orders WHERE id + 1 = 100;

-- 好（索引有效）
SELECT * FROM orders 
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01';
SELECT * FROM orders WHERE id = 99;
```

### IN vs EXISTS

```sql
-- 当子查询结果集大时，用 EXISTS 更快（短路求值）
-- 当子查询结果集小时，用 IN 更快

-- IN（子查询结果集小）
SELECT * FROM orders 
WHERE user_id IN (SELECT id FROM users WHERE city = 'Beijing');

-- EXISTS（orders 表大，users 表小）
SELECT * FROM orders o
WHERE EXISTS (
  SELECT 1 FROM users u 
  WHERE u.id = o.user_id AND u.city = 'Beijing'
);

-- 更优：直接 JOIN
SELECT o.* FROM orders o
INNER JOIN users u ON o.user_id = u.id
WHERE u.city = 'Beijing';
```

## 6.2 JOIN 优化

### Nested Loop Join（嵌套循环连接）

InnoDB 最常用的 Join 算法：

```
驱动表（外层）     被驱动表（内层）
for each row in 驱动表:
    for each matching row in 被驱动表:
        output row

关键：
- 驱动表数据量尽量小（过滤后）
- 被驱动表的 Join 列必须有索引！
```

### Join 优化原则

```sql
-- 原则1：小表驱动大表
-- MySQL 优化器通常会自动选择，但可以用 STRAIGHT_JOIN 强制
SELECT * FROM users u               -- users是小表，驱动
STRAIGHT_JOIN orders o              -- orders是大表，被驱动
ON u.id = o.user_id
WHERE u.city = 'Beijing';

-- 原则2：被驱动表 Join 列加索引
-- 给 orders.user_id 加索引
CREATE INDEX idx_user_id ON orders(user_id);

-- 原则3：Join 列的数据类型必须一致
-- 否则类型转换导致被驱动表索引失效
-- users.id INT vs orders.user_id BIGINT → 可能有问题
-- 建议：保持类型完全一致

-- 原则4：减少每个表的扫描行数（先过滤再 Join）
SELECT o.* FROM (
  SELECT id FROM users WHERE city = 'Beijing'
) u
JOIN orders o ON u.id = o.user_id;
```

### Hash Join（MySQL 8.0.18+）

```
当被驱动表没有合适索引时，MySQL 8.0 会自动使用 Hash Join：

构建阶段：将驱动表的 Join 列构建哈希表
探测阶段：遍历被驱动表，探测哈希表

适合：大表和大表的 Join，无索引情况
不适合：需要排序的 Join

-- EXPLAIN Extra 中显示：Hash Join
```

## 6.3 分页优化

### 传统分页的问题

```sql
-- 问题：当 OFFSET 很大时，需要扫描大量无用行
SELECT * FROM orders ORDER BY id DESC LIMIT 10 OFFSET 1000000;
-- 实际扫描 1,000,010 行，返回最后 10 行，效率极低！
```

### 方案一：游标分页（最优）

```sql
-- 记录上一页的最后一个 id，下一页从该 id 开始
-- 初始查询
SELECT * FROM orders ORDER BY id DESC LIMIT 10;
-- 假设最后一条 id = 9999990

-- 下一页（传入上次最后的 id）
SELECT * FROM orders WHERE id < 9999990 ORDER BY id DESC LIMIT 10;
-- 直接走主键索引，无需扫描前面的行

-- 优点：O(1) 复杂度，任意深度翻页不降速
-- 缺点：无法跳页（只能上一页/下一页）
```

### 方案二：子查询优化

```sql
-- 先用子查询找到目标页的起始 id（只查主键，覆盖索引）
-- 再用主键查完整数据

-- 差
SELECT * FROM orders ORDER BY id DESC LIMIT 10 OFFSET 1000000;

-- 好
SELECT o.* FROM orders o
INNER JOIN (
  SELECT id FROM orders ORDER BY id DESC LIMIT 10 OFFSET 1000000
) t ON o.id = t.id;
-- 子查询只查 id（覆盖索引），速度快
-- 再用 id 关联取完整行（10次主键查询）
```

### 方案三：业务限制

```
限制最大翻页深度（如最多100页）
引导用户使用搜索而非翻页
```

## 6.4 ORDER BY 优化

### filesort 的问题

```sql
-- 当 ORDER BY 列没有索引时，MySQL 需要 filesort（内存或磁盘排序）
-- EXPLAIN Extra: Using filesort

-- 差（无索引排序）
SELECT * FROM users ORDER BY age DESC LIMIT 10;

-- 好（给排序列加索引）
CREATE INDEX idx_age ON users(age);
SELECT * FROM users ORDER BY age DESC LIMIT 10;
-- EXPLAIN Extra: Using index（覆盖索引+有序遍历）
```

### 联合索引与 ORDER BY

```sql
-- 联合索引 idx_city_age (city, age)

-- 走索引排序
SELECT * FROM users WHERE city='Beijing' ORDER BY age;
-- city 等值过滤，age 有序 → Using index condition

-- 不走索引排序（filesort）
SELECT * FROM users WHERE city='Beijing' ORDER BY age DESC;
-- 与索引排序方向不一致？
-- 注：MySQL 8.0 支持降序索引，可以创建: INDEX idx(city ASC, age DESC)

-- 不走索引排序
SELECT * FROM users ORDER BY age;
-- 没有 city 的等值过滤，无法使用联合索引排序
```

### 排序 Buffer 优化

```ini
sort_buffer_size = 4M   # 排序缓冲区，太小会用磁盘临时文件
```

## 6.5 GROUP BY 优化

```sql
-- 原则：GROUP BY 列如果有索引，可以避免临时表

-- 差（需要临时表）
SELECT age, COUNT(*) FROM users GROUP BY age;
-- EXPLAIN Extra: Using temporary; Using filesort

-- 好（利用索引有序性）
-- 有索引 idx_age(age)
SELECT age, COUNT(*) FROM users GROUP BY age;
-- MySQL 可以按索引顺序扫描，直接分组，不需要临时表

-- 不需要排序时加 ORDER BY NULL
SELECT age, COUNT(*) FROM users GROUP BY age ORDER BY NULL;
-- 避免额外排序开销

-- 使用 HAVING 要注意
SELECT city, COUNT(*) cnt FROM users 
GROUP BY city HAVING cnt > 100;
-- HAVING 是对分组结果过滤，WHERE 是对原始行过滤
-- 尽量用 WHERE 提前过滤，减少参与分组的行数
```

## 6.6 批量操作优化

### 批量插入

```sql
-- 差：逐条插入（每次一个事务，网络往返）
INSERT INTO orders(user_id, amount) VALUES(1, 100);
INSERT INTO orders(user_id, amount) VALUES(2, 200);
-- 1000次循环 = 1000个事务

-- 好：批量插入（一个事务，一次网络往返）
INSERT INTO orders(user_id, amount) VALUES
  (1, 100),
  (2, 200),
  (3, 300),
  -- ... 建议每批 500-1000 行
  (1000, 99900);

-- 性能对比（10000行）：
-- 逐条插入：~30秒
-- 批量插入（每批1000行）：~0.3秒（100倍提升）
```

### 批量更新

```sql
-- 差：逐条 UPDATE
UPDATE products SET price = 10.0 WHERE id = 1;
UPDATE products SET price = 20.0 WHERE id = 2;
-- N次查询

-- 好：CASE WHEN 批量更新
UPDATE products 
SET price = CASE id
  WHEN 1 THEN 10.0
  WHEN 2 THEN 20.0
  WHEN 3 THEN 30.0
END
WHERE id IN (1, 2, 3);

-- 或：使用临时表
INSERT INTO temp_prices(id, price) VALUES(1,10.0),(2,20.0),(3,30.0);
UPDATE products p 
JOIN temp_prices t ON p.id = t.id
SET p.price = t.price;
```

### 大批量删除

```sql
-- 差：一次删除大量数据（长事务，大锁）
DELETE FROM logs WHERE created_at < '2023-01-01';
-- 可能锁表几分钟，影响业务

-- 好：分批删除
DELIMITER //
CREATE PROCEDURE batch_delete_logs()
BEGIN
  DECLARE affected INT DEFAULT 1;
  WHILE affected > 0 DO
    DELETE FROM logs 
    WHERE created_at < '2023-01-01' 
    LIMIT 1000;
    SET affected = ROW_COUNT();
    DO SLEEP(0.01);  -- 避免过度占用资源
  END WHILE;
END//
DELIMITER ;

CALL batch_delete_logs();
```

---

<a name="part-7"></a>
# Part 7: MySQL 调优

## 7.1 my.cnf 核心参数详解

```ini
[mysqld]
# ============ 基础配置 ============
user = mysql
port = 3306
datadir = /var/lib/mysql
socket = /var/run/mysqld/mysqld.sock
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci
default-time-zone = '+08:00'

# ============ 连接配置 ============
max_connections = 1000          # 最大连接数
max_connect_errors = 1000000    # 连接错误次数上限
wait_timeout = 28800            # 非交互连接超时（秒）
interactive_timeout = 28800     # 交互连接超时（秒）
back_log = 500                  # 连接请求队列深度

# ============ Buffer Pool ============
innodb_buffer_pool_size = 8G    # 物理内存的 50%~75%
innodb_buffer_pool_instances = 8  # CPU核数，减少锁竞争
innodb_buffer_pool_load_at_startup = ON   # 启动时预热
innodb_buffer_pool_dump_at_shutdown = ON  # 关闭时保存

# ============ InnoDB I/O ============
innodb_io_capacity = 2000       # IOPS上限（SSD可设4000+）
innodb_io_capacity_max = 4000   # 最大IOPS
innodb_flush_method = O_DIRECT  # 直接I/O，绕过OS缓存
innodb_file_per_table = ON      # 每张表独立 .ibd 文件

# ============ redo log ============
innodb_log_file_size = 1G       # 单个日志文件大小
innodb_log_files_in_group = 2   # 日志文件数量
innodb_flush_log_at_trx_commit = 1  # 1=最安全, 2=性能好

# ============ binlog ============
log_bin = /var/log/mysql/mysql-bin
binlog_format = ROW
sync_binlog = 1
binlog_expire_logs_seconds = 604800  # 保留7天

# ============ 查询缓存（8.0已废弃） ============
# query_cache_type = 0
# query_cache_size = 0

# ============ 排序/Join缓冲 ============
sort_buffer_size = 4M
join_buffer_size = 4M
read_buffer_size = 2M
read_rnd_buffer_size = 4M

# ============ 临时表 ============
tmp_table_size = 256M
max_heap_table_size = 256M

# ============ 慢查询日志 ============
slow_query_log = ON
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 1
log_queries_not_using_indexes = ON

# ============ 错误日志 ============
log_error = /var/log/mysql/error.log
log_error_verbosity = 2
```

## 7.2 Buffer Pool 调优

### 合理设置 Buffer Pool 大小

```
经验公式：
  专用 MySQL 服务器：innodb_buffer_pool_size = 物理内存 × 75%
  共享服务器：      innodb_buffer_pool_size = 物理内存 × 50%

示例（32GB内存服务器）：
  专用：innodb_buffer_pool_size = 24G
  共享：innodb_buffer_pool_size = 16G
```

### 监控 Buffer Pool

```sql
-- 查看Buffer Pool状态
SHOW ENGINE INNODB STATUS\G

-- 关键指标
SELECT 
  VARIABLE_NAME, 
  ROUND(VARIABLE_VALUE / 1024 / 1024, 2) AS MB
FROM performance_schema.global_status
WHERE VARIABLE_NAME IN (
  'Innodb_buffer_pool_bytes_data',     -- 数据页占用
  'Innodb_buffer_pool_bytes_dirty',    -- 脏页占用
  'Innodb_buffer_pool_pages_free',     -- 空闲页
  'Innodb_buffer_pool_pages_total'     -- 总页数
);

-- 计算命中率
SELECT
  ROUND(
    (1 - (
      SELECT VARIABLE_VALUE FROM performance_schema.global_status 
      WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads'
    ) / (
      SELECT VARIABLE_VALUE FROM performance_schema.global_status 
      WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests'
    )) * 100, 2
  ) AS hit_rate_percent;
-- 理想值 > 99%，若低于95%，需要增大 buffer_pool_size
```

## 7.3 连接池配置（HikariCP）

```yaml
# Spring Boot application.yml
spring:
  datasource:
    hikari:
      # 最大连接数（推荐 = CPU核数 * 2 + 1，或根据压测结果）
      maximum-pool-size: 20
      # 最小空闲连接数
      minimum-idle: 5
      # 连接超时（等待获取连接的最大时间）
      connection-timeout: 30000   # 30秒
      # 空闲连接超时（超过这个时间的空闲连接会被关闭）
      idle-timeout: 600000        # 10分钟
      # 连接最大生存时间（定期强制关闭重建，避免连接泄漏）
      max-lifetime: 1800000       # 30分钟
      # 连接存活检测
      keepalive-time: 30000       # 30秒心跳
      connection-test-query: SELECT 1
```

### 连接数怎么设置

```
错误观点：连接数越多越好

正确理解：
  数据库工作线程 = CPU核数
  每个核同时只能执行一个任务
  如果连接数远超CPU核数，大量连接在等待调度
  
推荐公式（PostgreSQL作者提出，MySQL也适用）：
  connections = (核心数 * 2) + 有效磁盘数
  
  8核CPU + 1个SSD：connections = 8*2 + 1 = 17
  
  实际中可以结合压测，找到吞吐量最高点时的连接数
```

## 7.4 表设计规范

### 选择合适的数据类型

```sql
-- 整数类型（按需选择，不要过度分配）
TINYINT   1字节  -128~127          适合：状态码、类型
SMALLINT  2字节  -32768~32767      适合：年份、小整数
INT       4字节  -21亿~21亿        适合：普通ID、数量
BIGINT    8字节  -922亿亿~922亿亿  适合：雪花ID、超大整数

-- 字符串类型
CHAR(n)     定长，最多255字符，适合：固定长度（MD5、电话）
VARCHAR(n)  变长，额外1-2字节长度头，适合：大多数文本
TEXT        最多65535字节，不建议加索引
LONGTEXT    最多4GB，存大文本（博客内容等）

-- 时间类型
DATETIME    8字节，范围：1000-9999年，不受时区影响
TIMESTAMP   4字节，范围：1970-2038年，自动转换时区
DATE        3字节，只存日期
-- 推荐：DATETIME（避免2038年问题）

-- 小数类型
DECIMAL(M,D) 精确小数，适合：金额（不要用FLOAT/DOUBLE）
FLOAT        4字节，不精确
DOUBLE       8字节，不精确
-- 金额必须用 DECIMAL 或存为整数（分）
```

### 表设计最佳实践

```sql
-- 1. 必须有主键，推荐自增 INT/BIGINT
CREATE TABLE orders (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  -- ...
);

-- 2. 字段不允许为 NULL（除非业务真的需要）
-- NULL 值在索引、统计、比较上有很多特殊处理
name VARCHAR(50) NOT NULL DEFAULT '',
age TINYINT UNSIGNED NOT NULL DEFAULT 0,

-- 3. 时间字段标配
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

-- 4. 软删除（逻辑删除）
is_deleted TINYINT NOT NULL DEFAULT 0,  -- 0:正常 1:已删除
deleted_at DATETIME,

-- 5. 添加适当注释
name VARCHAR(50) NOT NULL COMMENT '用户姓名',
status TINYINT NOT NULL DEFAULT 0 COMMENT '状态：0-待支付 1-已支付 2-已取消',
```

### 反规范化（适当冗余）

```
规范化的问题：Join 越多，性能越差

适当冗余的场景：
  orders 表存储 user_name（冗余，但避免 Join users 表）
  orders 表存储 product_name（下单时快照，也避免 Join）
  article 表存储 comment_count（冗余计数，避免COUNT查询）

冗余的代价：
  写操作需要同步更新多处
  可能存在数据不一致（需要补偿机制）

原则：读多写少的场景，冗余以提高读性能
```

## 7.5 索引设计规范

```
1. 主键使用自增整数（不用UUID，UUID写入会导致大量页分裂）

2. 联合索引：区分度高的列放前面

3. 索引数量控制在5个以内（每个索引都会降低写速度）

4. 前缀索引节省空间：
   -- 对于 VARCHAR(255) 的列，通常前20个字符就能区分大多数值
   CREATE INDEX idx_email_prefix ON users(email(20));

5. 不给低基数列建索引：
   -- 性别（male/female）只有2个值，走索引不如全表扫描
   -- 状态（0/1）通常也不适合单独建索引
   
6. 逻辑删除字段加入联合索引：
   -- 几乎所有查询都要加 is_deleted=0 条件
   CREATE INDEX idx_city_deleted ON users(city, is_deleted);
```

---

<a name="part-8"></a>
# Part 8: 高可用与扩展

## 8.1 主从复制

### 复制原理

```
主库 (Master)                        从库 (Slave)
+------------------+                 +------------------+
|                  |                 |                  |
| 1. 执行写操作     |                 |                  |
| 2. 写入 binlog   |---→ binlog --→  | 3. IO线程读binlog |
|                  |     (TCP)       |    写入relay log  |
+------------------+                 |                  |
                                     | 4. SQL线程重放    |
                                     |    relay log     |
                                     | 5. 应用数据变更   |
                                     +------------------+
```

### 配置主从复制

```sql
-- 主库配置 (my.cnf)
[mysqld]
server-id = 1         # 主库ID，必须唯一
log_bin = mysql-bin   # 开启binlog
binlog_format = ROW

-- 从库配置 (my.cnf)
[mysqld]
server-id = 2         # 从库ID，与主库不同
relay_log = relay-bin
read_only = ON        # 从库只读

-- 主库：创建复制用户
CREATE USER 'repl'@'%' IDENTIFIED BY 'password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;

-- 主库：查看当前binlog位置
SHOW MASTER STATUS;
-- +------------------+----------+
-- | File             | Position |
-- +------------------+----------+
-- | mysql-bin.000001 | 154      |
-- +------------------+----------+

-- 从库：配置并启动复制
CHANGE MASTER TO
  MASTER_HOST = '192.168.1.100',
  MASTER_PORT = 3306,
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'password',
  MASTER_LOG_FILE = 'mysql-bin.000001',
  MASTER_LOG_POS = 154;
START SLAVE;

-- 从库：检查复制状态
SHOW SLAVE STATUS\G
-- Slave_IO_Running: Yes  （IO线程正常）
-- Slave_SQL_Running: Yes （SQL线程正常）
-- Seconds_Behind_Master: 0  （延迟秒数，0=无延迟）
```

### 主从延迟

```
延迟原因：
  - 主库写入是并发的（多个线程同时写）
  - 从库 SQL 线程默认是单线程重放（串行）

解决方案：并行复制
  MySQL 5.7+: slave_parallel_type = LOGICAL_CLOCK
  MySQL 8.0:  slave_parallel_type = WRITESET（更好）

配置并行复制：
  [mysqld] # 从库
  slave_parallel_type = LOGICAL_CLOCK
  slave_parallel_workers = 8  # 并行线程数

监控延迟：
  SHOW SLAVE STATUS\G → Seconds_Behind_Master
  或使用 pt-heartbeat 工具
```

### GTID 复制（推荐）

GTID（Global Transaction ID）基于事务ID而非文件+偏移量，更简单且不易出错。

```ini
# 主库和从库都配置
gtid_mode = ON
enforce_gtid_consistency = ON

# 从库配置复制（无需指定文件和位置）
CHANGE MASTER TO
  MASTER_HOST = '192.168.1.100',
  MASTER_USER = 'repl',
  MASTER_PASSWORD = 'password',
  MASTER_AUTO_POSITION = 1;   -- 自动根据GTID同步
START SLAVE;
```

## 8.2 读写分离

### 架构图

```
应用层
  ↓
中间件（ShardingSphere / MyCAT / ProxySQL）
  ↓              ↓
写请求(INSERT   读请求(SELECT)
UPDATE/DELETE)     ↓
  ↓          负载均衡
主库(Master)  ↙  ↓  ↘
  ↓         从库1 从库2 从库3
  同步复制 →→→→→→→→→→→→→
```

### Spring Boot + ShardingSphere 读写分离

```yaml
# application.yml
spring:
  shardingsphere:
    datasource:
      names: master,slave1,slave2
      master:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://master-host:3306/mydb
        username: root
        password: password
      slave1:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave1-host:3306/mydb
        username: root
        password: password
      slave2:
        type: com.zaxxer.hikari.HikariDataSource
        jdbc-url: jdbc:mysql://slave2-host:3306/mydb
        username: root
        password: password
    rules:
      readwrite-splitting:
        data-sources:
          myds:
            write-data-source-name: master
            read-data-source-names:
              - slave1
              - slave2
            load-balancer-name: round_robin
        load-balancers:
          round_robin:
            type: ROUND_ROBIN
    props:
      sql-show: true
```

### 读写分离的注意事项

```
1. 主从延迟导致读到旧数据
   解决：写后立即读，强制路由到主库
   // 注解方式（ShardingSphere）
   @ShardingTransactionType
   或在代码中强制走主库：
   HintManager.getInstance().setMasterRouteOnly();

2. 事务内的读操作
   事务中的所有读写都应走主库（保证一致性）
   ShardingSphere 自动处理：事务内强制走主库

3. 从库数量与延迟
   从库越多，延迟可能越大（复制线程资源竞争）
   建议：从库数 <= 5，超过5个考虑级联复制
```

## 8.3 分库分表

### 什么时候需要分库分表

```
分库分表的触发条件：
  单表行数 > 2000万（InnoDB B+树超过4层，性能下降）
  单库磁盘 > 1TB
  QPS > 单机上限（通常2~3万写QPS，10万读QPS）
  
分库分表的代价：
  - 无法使用 JOIN（跨库）
  - 分布式事务复杂
  - 聚合查询（COUNT/SUM）需要多库汇总
  - 运维复杂度大幅增加
  
原则：能不拆就不拆，先尝试：
  1. 加索引、SQL优化
  2. 加 Buffer Pool
  3. 读写分离
  4. 升级硬件（SSD、更大内存）
  最后才考虑分库分表
```

### 垂直拆分

```
垂直分库：按业务模块拆分数据库
  原来：单一 mydb 数据库，包含所有表
  拆后：
    user_db:   users, user_profiles, user_addresses
    order_db:  orders, order_items, payments
    product_db: products, categories, inventory

垂直分表：将一张表的列拆分为多张表
  将访问频繁的列和不常访问的大字段分开
  
  原来：articles (id, title, author, content, views, likes)
  拆后：
    articles      (id, title, author, views, likes)  -- 热数据
    articles_content (id, article_id, content)        -- 冷数据（大字段）
```

### 水平分表

```
水平分表：将同一张表的行按规则拆分到多张表
  
  原来：orders 表 (10亿行)
  拆后：
    orders_0 (2.5亿行)
    orders_1 (2.5亿行)
    orders_2 (2.5亿行)
    orders_3 (2.5亿行)

分片键选择：
  - 应选择查询最频繁的列（否则每次查询都需要扫描所有分片）
  - 应选择分布均匀的列（避免数据倾斜）
  - 通常选择用户ID（保证同一用户的数据在同一分片，避免跨片）

分片算法：
  1. 取模：user_id % 4 → 均匀，但扩容难
  2. 范围：id < 1亿→分片0，1亿~2亿→分片1
            优点：扩容简单；缺点：可能数据倾斜
  3. 一致性哈希：扩容时只迁移少量数据（推荐）
```

### ShardingSphere 分片配置示例

```yaml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        jdbc-url: jdbc:mysql://db0-host:3306/orderdb
        username: root
        password: password
      ds1:
        jdbc-url: jdbc:mysql://db1-host:3306/orderdb
        username: root
        password: password
    rules:
      sharding:
        tables:
          orders:
            actual-data-nodes: ds${0..1}.orders_${0..3}
            # 分库策略：user_id % 2
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db_mod
            # 分表策略：user_id % 4
            table-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: table_mod
            # 分布式主键
            key-generate-strategy:
              column: id
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
```

## 8.4 MGR（MySQL Group Replication）

MGR 是 MySQL 官方的高可用方案，提供多主或单主模式。

```
MGR 单主模式：
  Primary (可读写)
      ↑↓ Paxos 协议同步
  Secondary1 (只读)
      ↑↓
  Secondary2 (只读)
  
  Primary 挂了 → 自动选举新 Primary（无需人工切换）
  需要多数节点存活（3节点需要2个存活）

优点：
  - 自动故障切换
  - 强一致性（同步复制）
  - 官方支持

缺点：
  - 写性能有损耗（需要多数节点确认）
  - 部署复杂
  - 网络延迟敏感
```

---

<a name="part-9"></a>
# Part 9: 完整实战案例 —— 电商订单系统

## 9.1 数据库设计

```sql
-- 创建数据库
CREATE DATABASE ecommerce 
  DEFAULT CHARACTER SET utf8mb4 
  DEFAULT COLLATE utf8mb4_unicode_ci;

USE ecommerce;

-- ============================
-- 用户表
-- ============================
CREATE TABLE users (
  id          BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '用户ID',
  username    VARCHAR(50)     NOT NULL                COMMENT '用户名',
  email       VARCHAR(100)    NOT NULL                COMMENT '邮箱',
  phone       VARCHAR(20)     NOT NULL DEFAULT ''     COMMENT '手机号',
  password    VARCHAR(255)    NOT NULL                COMMENT '密码（加密）',
  city        VARCHAR(50)     NOT NULL DEFAULT ''     COMMENT '城市',
  status      TINYINT         NOT NULL DEFAULT 1      COMMENT '状态：1-正常 0-禁用',
  is_deleted  TINYINT         NOT NULL DEFAULT 0      COMMENT '逻辑删除：0-正常 1-已删',
  created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_email (email),
  UNIQUE KEY uk_phone (phone),
  KEY idx_city_status (city, status),
  KEY idx_created_at (created_at)
) ENGINE=InnoDB COMMENT='用户表';

-- ============================
-- 商品分类表
-- ============================
CREATE TABLE categories (
  id          INT UNSIGNED    NOT NULL AUTO_INCREMENT,
  name        VARCHAR(100)    NOT NULL                COMMENT '分类名称',
  parent_id   INT UNSIGNED    NOT NULL DEFAULT 0      COMMENT '父分类ID，0=顶级',
  sort_order  INT             NOT NULL DEFAULT 0      COMMENT '排序',
  is_deleted  TINYINT         NOT NULL DEFAULT 0,
  created_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_parent_id (parent_id)
) ENGINE=InnoDB COMMENT='商品分类';

-- ============================
-- 商品表
-- ============================
CREATE TABLE products (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '商品ID',
  category_id  INT UNSIGNED    NOT NULL                COMMENT '分类ID',
  name         VARCHAR(200)    NOT NULL                COMMENT '商品名称',
  description  TEXT                                    COMMENT '商品描述',
  price        DECIMAL(12,2)   NOT NULL DEFAULT 0.00  COMMENT '价格',
  stock        INT             NOT NULL DEFAULT 0      COMMENT '库存',
  sales        INT             NOT NULL DEFAULT 0      COMMENT '销量',
  status       TINYINT         NOT NULL DEFAULT 1      COMMENT '状态：1-上架 0-下架',
  is_deleted   TINYINT         NOT NULL DEFAULT 0,
  created_at   DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at   DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_category_status (category_id, status),
  KEY idx_price (price),
  KEY idx_sales (sales),
  FULLTEXT KEY ft_name (name)   -- 全文索引，用于搜索
) ENGINE=InnoDB COMMENT='商品表';

-- ============================
-- 订单主表
-- ============================
CREATE TABLE orders (
  id            BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '订单ID',
  order_no      VARCHAR(32)     NOT NULL                COMMENT '订单号（业务唯一标识）',
  user_id       BIGINT UNSIGNED NOT NULL                COMMENT '用户ID',
  total_amount  DECIMAL(12,2)   NOT NULL DEFAULT 0.00  COMMENT '订单总金额',
  pay_amount    DECIMAL(12,2)   NOT NULL DEFAULT 0.00  COMMENT '实付金额',
  discount      DECIMAL(12,2)   NOT NULL DEFAULT 0.00  COMMENT '优惠金额',
  status        TINYINT         NOT NULL DEFAULT 0      
    COMMENT '状态：0-待支付 1-已支付 2-待发货 3-已发货 4-已完成 5-已取消',
  pay_time      DATETIME                                COMMENT '支付时间',
  deliver_time  DATETIME                                COMMENT '发货时间',
  finish_time   DATETIME                                COMMENT '完成时间',
  cancel_time   DATETIME                                COMMENT '取消时间',
  remark        VARCHAR(500)    NOT NULL DEFAULT ''     COMMENT '订单备注',
  -- 冗余用户信息（下单时快照）
  user_name     VARCHAR(50)     NOT NULL DEFAULT ''     COMMENT '下单用户名',
  user_phone    VARCHAR(20)     NOT NULL DEFAULT ''     COMMENT '收货手机',
  address       VARCHAR(500)    NOT NULL DEFAULT ''     COMMENT '收货地址',
  is_deleted    TINYINT         NOT NULL DEFAULT 0,
  created_at    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_order_no (order_no),
  KEY idx_user_id_status (user_id, status),
  KEY idx_status_created (status, created_at),
  KEY idx_pay_time (pay_time)
) ENGINE=InnoDB COMMENT='订单主表';

-- ============================
-- 订单明细表
-- ============================
CREATE TABLE order_items (
  id           BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  order_id     BIGINT UNSIGNED NOT NULL                COMMENT '订单ID',
  order_no     VARCHAR(32)     NOT NULL                COMMENT '订单号',
  product_id   BIGINT UNSIGNED NOT NULL                COMMENT '商品ID',
  -- 冗余商品信息（下单时快照）
  product_name VARCHAR(200)    NOT NULL                COMMENT '商品名称',
  product_price DECIMAL(12,2)  NOT NULL DEFAULT 0.00  COMMENT '下单时单价',
  quantity     INT             NOT NULL DEFAULT 1      COMMENT '购买数量',
  subtotal     DECIMAL(12,2)   NOT NULL DEFAULT 0.00  COMMENT '小计',
  created_at   DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  KEY idx_order_id (order_id),
  KEY idx_product_id (product_id)
) ENGINE=InnoDB COMMENT='订单明细表';

-- ============================
-- 支付记录表
-- ============================
CREATE TABLE payments (
  id             BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
  order_id       BIGINT UNSIGNED NOT NULL                COMMENT '订单ID',
  order_no       VARCHAR(32)     NOT NULL                COMMENT '订单号',
  pay_no         VARCHAR(64)     NOT NULL                COMMENT '第三方支付流水号',
  amount         DECIMAL(12,2)   NOT NULL DEFAULT 0.00  COMMENT '支付金额',
  pay_method     TINYINT         NOT NULL DEFAULT 1      COMMENT '支付方式：1-微信 2-支付宝',
  status         TINYINT         NOT NULL DEFAULT 0      COMMENT '状态：0-待支付 1-支付成功 2-支付失败',
  pay_time       DATETIME                                COMMENT '支付完成时间',
  created_at     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at     DATETIME        NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY uk_pay_no (pay_no),
  KEY idx_order_id (order_id),
  KEY idx_order_no (order_no)
) ENGINE=InnoDB COMMENT='支付记录';
```

## 9.2 常用业务 SQL

### 查询用户订单列表（带分页）

```sql
-- 普通分页（适合前几页）
SELECT 
  o.id,
  o.order_no,
  o.total_amount,
  o.status,
  o.created_at,
  COUNT(oi.id) as item_count
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = 10001
  AND o.is_deleted = 0
GROUP BY o.id, o.order_no, o.total_amount, o.status, o.created_at
ORDER BY o.created_at DESC
LIMIT 20 OFFSET 0;

-- 游标分页（适合深翻页，传入上一页最后的 id 和 created_at）
SELECT 
  o.id,
  o.order_no,
  o.total_amount,
  o.status,
  o.created_at
FROM orders o
WHERE o.user_id = 10001
  AND o.is_deleted = 0
  AND (o.created_at < '2024-01-15 10:00:00' 
       OR (o.created_at = '2024-01-15 10:00:00' AND o.id < 9876))
ORDER BY o.created_at DESC, o.id DESC
LIMIT 20;
```

### 下订单（事务）

```sql
-- 完整的下单事务
START TRANSACTION;

-- 1. 检查库存并扣减（悲观锁，FOR UPDATE）
SELECT stock FROM products WHERE id = 100 FOR UPDATE;
-- 应用层检查 stock >= quantity
UPDATE products 
SET stock = stock - 2, sales = sales + 2
WHERE id = 100 AND stock >= 2;
-- 检查 ROW_COUNT() = 1，否则库存不足，ROLLBACK

-- 2. 创建订单
INSERT INTO orders (
  order_no, user_id, total_amount, pay_amount,
  status, user_name, user_phone, address
) VALUES (
  '20240115100001001', 10001, 199.00, 189.00,
  0, 'Alice', '13800138000', '北京市朝阳区xxx'
);
SET @order_id = LAST_INSERT_ID();

-- 3. 创建订单明细
INSERT INTO order_items (
  order_id, order_no, product_id, 
  product_name, product_price, quantity, subtotal
) VALUES (
  @order_id, '20240115100001001', 100,
  'iPhone 15', 99.50, 2, 199.00
);

COMMIT;
```

### 统计报表查询

```sql
-- 每日订单数量和金额统计（近30天）
SELECT 
  DATE(created_at) AS order_date,
  COUNT(*) AS order_count,
  SUM(pay_amount) AS total_revenue,
  AVG(pay_amount) AS avg_amount
FROM orders
WHERE created_at >= DATE_SUB(NOW(), INTERVAL 30 DAY)
  AND status IN (1, 2, 3, 4)  -- 已支付及以上状态
  AND is_deleted = 0
GROUP BY DATE(created_at)
ORDER BY order_date DESC;

-- 销售额最高的商品 TOP10（上月）
SELECT 
  oi.product_id,
  oi.product_name,
  SUM(oi.quantity) AS total_quantity,
  SUM(oi.subtotal) AS total_amount,
  COUNT(DISTINCT oi.order_id) AS order_count
FROM order_items oi
INNER JOIN orders o ON oi.order_id = o.id
WHERE o.created_at >= DATE_FORMAT(DATE_SUB(NOW(), INTERVAL 1 MONTH), '%Y-%m-01')
  AND o.created_at < DATE_FORMAT(NOW(), '%Y-%m-01')
  AND o.status IN (1, 2, 3, 4)
  AND o.is_deleted = 0
GROUP BY oi.product_id, oi.product_name
ORDER BY total_amount DESC
LIMIT 10;

-- 各城市用户数量统计
SELECT 
  city,
  COUNT(*) AS user_count,
  COUNT(CASE WHEN status = 1 THEN 1 END) AS active_count
FROM users
WHERE is_deleted = 0
GROUP BY city
ORDER BY user_count DESC;
```

## 9.3 慢查询定位与优化案例

### 案例一：缺少索引

```sql
-- 问题 SQL（慢查询日志发现，扫描全表）
SELECT * FROM orders 
WHERE user_id = 10001 AND status = 1
ORDER BY created_at DESC
LIMIT 10;

-- EXPLAIN 分析
EXPLAIN SELECT * FROM orders 
WHERE user_id = 10001 AND status = 1
ORDER BY created_at DESC
LIMIT 10;
-- type: ALL (全表扫描！)
-- rows: 5000000
-- Extra: Using where; Using filesort

-- 优化：创建联合索引
CREATE INDEX idx_user_status_time ON orders(user_id, status, created_at);

-- 再次 EXPLAIN
-- type: range
-- key: idx_user_status_time
-- rows: 25
-- Extra: Using index condition; Using filesort (排序方向问题)

-- 进一步优化：倒序索引（MySQL 8.0+）
CREATE INDEX idx_user_status_time ON orders(user_id, status, created_at DESC);
-- Extra: Using index condition (无filesort!)
```

### 案例二：深分页优化

```sql
-- 问题 SQL
SELECT * FROM orders 
WHERE status = 4  -- 已完成
ORDER BY id DESC
LIMIT 20 OFFSET 500000;
-- 扫描 500020 行，取最后20行，极慢！

-- 方案：子查询优化
SELECT o.* FROM orders o
INNER JOIN (
  SELECT id FROM orders
  WHERE status = 4
  ORDER BY id DESC
  LIMIT 20 OFFSET 500000
) t ON o.id = t.id;
-- 子查询走覆盖索引(id, status)，速度大幅提升
-- 再关联取完整数据（20次主键查询）
```

### 案例三：N+1 查询问题

```java
// 问题代码（Java伪代码）
List<Order> orders = orderDao.findByUserId(userId);  // 1次查询
for (Order order : orders) {
    // N次查询（每个订单查一次明细）
    List<OrderItem> items = itemDao.findByOrderId(order.getId());
    order.setItems(items);
}
// 总共：1 + N 次查询

// 优化：一次查询所有明细，在应用层组装
List<Order> orders = orderDao.findByUserId(userId);  // 1次查询
List<Long> orderIds = orders.stream()
    .map(Order::getId).collect(Collectors.toList());
List<OrderItem> allItems = itemDao.findByOrderIds(orderIds);  // 1次查询

// 按 order_id 分组
Map<Long, List<OrderItem>> itemMap = allItems.stream()
    .collect(Collectors.groupingBy(OrderItem::getOrderId));

// 组装
orders.forEach(o -> o.setItems(itemMap.getOrDefault(o.getId(), emptyList())));
// 总共：2次查询
```

对应的 SQL：
```sql
-- 一次查询所有相关明细
SELECT * FROM order_items
WHERE order_id IN (1001, 1002, 1003, 1004, 1005);
```

### 案例四：库存超卖防护

```sql
-- 方案一：悲观锁（SELECT FOR UPDATE）
-- 适合并发不高的场景
START TRANSACTION;
SELECT stock FROM products WHERE id = 100 FOR UPDATE;
-- 检查库存
UPDATE products SET stock = stock - 1 WHERE id = 100 AND stock >= 1;
COMMIT;

-- 方案二：乐观锁（CAS，无锁化）
-- 适合并发高的场景
-- 更新时检查版本号（或直接用 stock >= quantity 作为条件）
UPDATE products 
SET stock = stock - 1
WHERE id = 100 AND stock >= 1;
-- 检查 ROW_COUNT() == 1，否则库存不足

-- 方案三：Redis 预扣库存（最高性能）
-- 1. 活动开始时，将库存同步到 Redis
-- 2. 下单时，Redis DECRBY 预扣减（原子操作）
-- 3. 扣减成功则创建订单，写入 MQ 异步扣减 DB 库存
-- 4. 扣减失败（库存不足）则直接返回失败
```

---

<a name="part-10"></a>
# Part 10: 常见问题 FAQ

## Q1: 为什么推荐使用自增主键而不是 UUID？

```
自增主键（INT/BIGINT AUTO_INCREMENT）：
  优点：
  ✓ 顺序插入 → B+树叶子节点按顺序追加，极少页分裂
  ✓ 占用空间小（4或8字节）
  ✓ 索引紧凑，查询快
  
  缺点：
  ✗ 分布式环境下ID重复（需要分布式ID方案）
  ✗ ID连续，可能被枚举爬取

UUID（VARCHAR(36)）：
  优点：
  ✓ 全局唯一，适合分布式
  
  缺点：
  ✗ 随机插入 → 大量页分裂，写性能差
  ✗ 占用36字节，索引膨胀
  ✗ 不可读，不便于排序

推荐方案（分布式）：
  使用雪花算法（Snowflake ID）：
  - 64位整数，全局唯一
  - 包含时间戳，单调递增
  - 高性能（每秒千万级）
  
  雪花 ID 结构：
  [1位符号][41位时间戳][10位机器ID][12位序列号]
```

## Q2: CHAR 和 VARCHAR 如何选择？

```
CHAR(n)：
  - 固定长度，不足用空格填充，读取时自动去掉尾部空格
  - 存储固定长度数据时，比 VARCHAR 稍快（无需解析长度）
  - 适合：MD5（32字符）、手机号（11位）、固定格式编码
  
  CHAR(10) 存 'abc' → 实际存储 'abc       '（10字节）

VARCHAR(n)：
  - 变长，额外1~2字节存储实际长度
  - 节省空间，适合长度变化大的数据
  - 适合：姓名、地址、备注等
  
  VARCHAR(255) 存 'abc' → 实际存储 1字节长度 + 3字节数据

注意：
  - VARCHAR(255) 和 VARCHAR(1000) 只影响最大存储，对实际数据量相同的行存储没影响
  - 但在使用内存临时表时，MySQL 按最大长度分配空间（所以不要随意设很大的 VARCHAR）
```

## Q3: 索引越多越好吗？

```
不是！索引有双重代价：

读性能提升（好处）：
  - 加速 SELECT、JOIN、ORDER BY、GROUP BY

写性能下降（代价）：
  - 每次 INSERT：需要更新所有索引（N个索引 = N次B+树插入）
  - 每次 UPDATE/DELETE：需要同步更新相关索引
  - 占用磁盘空间
  - 需要在 Buffer Pool 中占用内存

原则：
  1. 主键索引：必须有
  2. 频繁查询的 WHERE 条件列：建索引
  3. 频繁 JOIN 的连接列：建索引
  4. 频繁排序的列：考虑建索引
  5. 低基数列（性别、状态）：谨慎建索引
  6. 超长字符串列：用前缀索引
  7. 单表索引数量建议不超过5个
```

## Q4: 大事务有什么危害？如何避免？

```
大事务的危害：
  1. 持锁时间长 → 其他事务长时间等待 → 连接堆积 → 系统崩溃
  2. undo log 无法清理 → undo 表空间膨胀
  3. binlog 写入量大 → 主从延迟增大
  4. 回滚代价大（操作越多，回滚越慢）
  5. 死锁概率增加

如何发现大事务：
  SELECT * FROM information_schema.INNODB_TRX
  WHERE TIME_TO_SEC(TIMEDIFF(NOW(), trx_started)) > 10;

如何避免：
  1. 将大事务拆分为多个小事务
  2. 事务中不要做 RPC 调用（如HTTP请求、MQ消息）
  3. 不要在循环中使用事务（批量操作替代）
  4. 减少事务内的查询次数
  5. 避免在事务中做复杂运算
  
  最佳实践：
  @Transactional 注解的方法只包含核心的数据库操作，
  前置的查询、验证、准备工作在事务外完成
```

## Q5: COUNT(*), COUNT(1), COUNT(列名) 有什么区别？

```
COUNT(*) ：统计所有行数（包括 NULL 行），推荐使用
COUNT(1) ：与 COUNT(*) 等价，MySQL 优化器会优化为同等效率
COUNT(列名)：统计该列非 NULL 的行数，注意会忽略 NULL 值

性能：
  在 InnoDB 中，三者效率基本一致（优化器会自动选最小的索引扫描）
  如果存在二级索引，会优先走最小的索引（节省 I/O）
  如果没有索引，走全表扫描

MyISAM 特殊情况：
  MyISAM 维护了行数计数器，COUNT(*) 不加 WHERE 直接返回（O(1)）
  InnoDB 因为 MVCC，每次都需要扫描（O(n)）

推荐：
  使用 COUNT(*) 即可，语义最清晰
```

## Q6: 为什么 SELECT * 不推荐使用？

```
1. 无法使用覆盖索引
   SELECT * 需要回表，即使有联合索引也无法避免
   
2. 传输额外数据
   很多列可能不需要，但都从服务器传输到应用层（网络带宽浪费）
   
3. 无法缓存字段信息
   ORM框架映射所有列，包括不需要的大字段（TEXT、BLOB）
   
4. 表结构变更后可能出错
   应用代码按位置读取列时，新增列可能导致映射错位
   
5. 影响优化器决策
   优化器无法提前判断是否可以走覆盖索引

推荐做法：
  - 只查需要的列：SELECT id, name, email FROM users WHERE ...
  - 对于 ORM，使用 DTO/VO 投影，只映射需要的字段
```

## Q7: 什么情况下应该使用 FORCE INDEX？

```
FORCE INDEX 强制指定索引，绕过优化器的选择。

使用场景（极少数情况）：
  1. 优化器选错了索引（统计信息不准确）
  2. 你通过 EXPLAIN 和测试确认手动指定更快

示例：
  -- 正常查询（优化器选了错误的索引）
  SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10;
  -- EXPLAIN 发现走了 idx_created_at 而非 idx_user_id
  
  -- 强制走正确索引
  SELECT * FROM orders FORCE INDEX (idx_user_id_status)
  WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10;

注意事项：
  - FORCE INDEX 是最后手段，先尝试：分析统计信息、重写 SQL
  - ANALYZE TABLE orders; -- 更新统计信息，可能解决优化器选错问题
  - FORCE INDEX 与表结构绑定，索引更名/删除时忘了改 SQL 会报错
  - 优先通过优化 SQL 和索引来解决，而非 FORCE INDEX
```

## Q8: 如何定位和处理主从延迟？

```
查看延迟：
  -- 在从库执行
  SHOW SLAVE STATUS\G
  -- Seconds_Behind_Master: 延迟秒数
  -- 0 = 正常，> 10 = 需要关注，> 60 = 严重

延迟原因排查：
  1. 大事务：主库有耗时很长的大事务
     → SHOW PROCESSLIST 查看主库当前事务
  2. DDL：ALTER TABLE 等 DDL 操作在从库串行执行
  3. 单线程复制：从库 SQL 线程是单线程，主库高并发时跟不上
  4. 从库性能不足：磁盘 I/O 差，CPU 不够

解决方案：
  1. 开启并行复制（见 Part 8.1）
  2. 大事务拆小事务
  3. DDL 操作在业务低峰期执行（或使用 pt-online-schema-change）
  4. 升级从库硬件（SSD、更多 CPU）
  5. 减少从库数量（每增加一个从库都会带来额外负载）

应用层处理延迟：
  1. 写后读走主库（强一致性场景）
  2. 允许短暂延迟的场景走从库
  3. 设置合理的从库超时，延迟严重时降级到主库读
```

## Q9: 如何安全地对大表做 DDL（ALTER TABLE）？

```
问题：
  ALTER TABLE 在 MySQL 中默认会锁表
  对于上亿行的大表，ALTER 可能持续数小时，期间无法写入！

MySQL 8.0 Online DDL（部分操作支持）：
  -- Instant 操作（瞬间完成，不阻塞）
  ALTER TABLE orders ADD COLUMN remark2 VARCHAR(100), ALGORITHM=INSTANT;
  
  -- In-place 操作（不复制表，但可能短暂锁）
  ALTER TABLE orders ADD INDEX idx_status, ALGORITHM=INPLACE, LOCK=NONE;
  
  -- 查看操作是否支持 INSTANT
  -- https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html

使用 pt-online-schema-change（推荐工具）：
  原理：创建新表 → 触发器同步增量数据 → 逐步拷贝存量数据 → 原子替换
  不锁原表，业务不中断
  
  # 示例：给 orders 表加一列
  pt-online-schema-change \
    --alter "ADD COLUMN note VARCHAR(200) NOT NULL DEFAULT ''" \
    --execute \
    D=mydb,t=orders \
    --host=127.0.0.1 \
    --user=root \
    --password=xxx

使用 gh-ost（GitHub开源，更现代）：
  原理：通过 binlog 而非触发器同步数据，侵入性更低
  支持暂停、限速、动态调整
  
  gh-ost \
    --user="root" --password="xxx" \
    --host="127.0.0.1" \
    --database="mydb" \
    --table="orders" \
    --alter="ADD COLUMN note VARCHAR(200) NOT NULL DEFAULT ''" \
    --execute
```

## Q10: 遇到 "Lock wait timeout exceeded" 怎么处理？

```
错误含义：
  等待行锁超时（默认50秒）
  innodb_lock_wait_timeout = 50  # 可调整

排查步骤：

1. 查看当前锁等待
SELECT 
  r.trx_id waiting_trx_id,
  r.trx_mysql_thread_id waiting_thread,
  r.trx_query waiting_query,
  b.trx_id blocking_trx_id,
  b.trx_mysql_thread_id blocking_thread,
  b.trx_query blocking_query
FROM information_schema.INNODB_TRX b
JOIN information_schema.INNODB_TRX r 
  ON r.trx_wait_started IS NOT NULL
  AND b.trx_id = r.trx_wait_started;

-- MySQL 8.0 更直观：
SELECT * FROM performance_schema.data_lock_waits\G

2. 找到阻塞的 thread id 后，杀死它
KILL 阻塞的线程ID;

3. 分析阻塞原因
  - 查看 SHOW ENGINE INNODB STATUS\G 中的锁信息
  - 确认是否有未提交的事务
  - 检查是否是死锁

4. 根本解决
  - 优化 SQL，减少锁持有时间
  - 拆分大事务
  - 检查代码中是否忘记提交/回滚事务
  - 适当调整 innodb_lock_wait_timeout（不是根本解决方案）
  
预防措施：
  -- 开启死锁检测日志
  SET GLOBAL innodb_print_all_deadlocks = ON;
  
  -- 锁等待超时适当调小（快速失败，避免请求堆积）
  SET innodb_lock_wait_timeout = 10;
```

---

# 总结

| 知识点 | 核心要点 |
|--------|---------|
| 架构 | 连接层→服务层→引擎层→存储层 |
| Buffer Pool | 越大越好，建议内存的75%；命中率>99% |
| 事务 | ACID；RR隔离级别+MVCC；注意大事务危害 |
| 日志 | redo保证持久性，undo保证原子性，binlog用于复制 |
| 索引 | B+树；最左前缀；覆盖索引；EXPLAIN分析 |
| SQL优化 | 明确列名；避免函数；游标分页；批量操作 |
| 调优 | Buffer Pool大小；并行复制；连接池配置 |
| 高可用 | 主从复制+读写分离；MGR；分库分表慎用 |

> **最终原则：没有银弹，所有的优化都需要基于真实的测试数据和业务场景做出决策。**

---
*文档版本：MySQL 8.0 | 最后更新：2025年*
