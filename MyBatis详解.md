# MyBatis 详解 - 从零到精通

> 本文档覆盖 MyBatis 核心原理、配置、动态SQL、缓存、插件、Spring整合、MyBatis-Plus、实战案例与面试题，适合初学者入门到高级工程师深度掌握。

---

## 目录

- [Part 1: MyBatis 整体架构](#part-1)
- [Part 2: MyBatis 核心配置深度解析](#part-2)
- [Part 3: Mapper XML 深度解析](#part-3)
- [Part 4: 动态 SQL 深度解析](#part-4)
- [Part 5: MyBatis 缓存机制](#part-5)
- [Part 6: MyBatis 插件（拦截器）机制](#part-6)
- [Part 7: MyBatis 执行器（Executor）原理](#part-7)
- [Part 8: MyBatis 与 Spring 整合原理](#part-8)
- [Part 9: MyBatis-Plus 详解](#part-9)
- [Part 10: 高级特性](#part-10)
- [Part 11: 完整实战案例](#part-11)
- [Part 12: 性能优化](#part-12)
- [Part 13: 常见面试题 FAQ](#part-13)

---

# Part 1: MyBatis 整体架构 <a id="part-1"></a>

## 1.1 MyBatis 是什么

MyBatis 是一款优秀的**持久层框架**，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO 为数据库中的记录。

### MyBatis 的历史

MyBatis 最初是 Apache 的一个开源项目 iBatis，2010 年这个项目由 Apache Software Foundation 迁移到了 Google Code，并且改名为 MyBatis。2013 年 11 月迁移到 GitHub。

当前 MyBatis 的最新稳定版本为 3.5.x 系列，与 Spring Boot 3.x 完美兼容。

### 与 JDBC 的对比

**原生 JDBC 的痛点：**

```java
// 原生 JDBC 代码 - 繁琐且易出错
Connection conn = null;
PreparedStatement pstmt = null;
ResultSet rs = null;
try {
    // 1. 加载驱动
    Class.forName("com.mysql.cj.jdbc.Driver");
    // 2. 获取连接
    conn = DriverManager.getConnection(
        "jdbc:mysql://localhost:3306/test", "root", "password");
    // 3. 创建 PreparedStatement
    pstmt = conn.prepareStatement(
        "SELECT id, name, email FROM user WHERE id = ?");
    // 4. 设置参数
    pstmt.setLong(1, userId);
    // 5. 执行查询
    rs = pstmt.executeQuery();
    // 6. 手动映射结果集
    User user = null;
    if (rs.next()) {
        user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        user.setEmail(rs.getString("email"));
    }
    return user;
} catch (Exception e) {
    throw new RuntimeException(e);
} finally {
    // 7. 手动关闭资源（容易遗忘）
    if (rs != null) try { rs.close(); } catch (SQLException e) {}
    if (pstmt != null) try { pstmt.close(); } catch (SQLException e) {}
    if (conn != null) try { conn.close(); } catch (SQLException e) {}
}
```

**MyBatis 方式：**

```java
// MyBatis 方式 - 简洁优雅
@Mapper
public interface UserMapper {
    User selectById(Long id);
}
```

```xml
<!-- UserMapper.xml -->
<select id="selectById" resultType="User">
    SELECT id, name, email FROM user WHERE id = #{id}
</select>
```

```java
// 调用
User user = userMapper.selectById(1L);
```

| 对比维度 | 原生 JDBC | MyBatis | Hibernate/JPA |
|---------|-----------|---------|---------------|
| SQL 控制 | 完全手写 | 手写SQL，XML管理 | 自动生成，HQL |
| 学习成本 | 低（SQL本身）| 低-中 | 高 |
| 灵活性 | 最高 | 高 | 低-中 |
| 开发效率 | 低 | 中-高 | 高 |
| 性能 | 最高（手动优化）| 高 | 中（N+1问题）|
| 数据库移植性 | 低 | 低-中 | 高 |
| 复杂SQL支持 | 完全支持 | 完全支持 | 有限 |
| 自动建表 | 不支持 | 不支持 | 支持 |
| 适用场景 | 极致性能 | 复杂SQL系统 | 快速开发简单系统 |

### 与 Hibernate 的对比

**Hibernate（全自动ORM）的特点：**
- 通过实体类与数据库表的映射关系，自动生成 SQL
- 开发者几乎不需要写 SQL，提高开发效率
- 对于复杂查询，生成的 SQL 不够优化
- N+1 查询问题是老生常谈的痛点

**MyBatis（半自动ORM）的特点：**
- 需要开发者自己写 SQL，但提供了参数映射和结果映射
- SQL 完全可控，适合对性能要求高的系统
- 在中国企业应用中使用率极高
- 与 Spring 生态集成非常好

**选型建议：**
- 互联网公司、金融系统、复杂业务逻辑 -> **MyBatis**（SQL可控、性能优）
- 快速原型开发、简单 CRUD、DDD 领域驱动设计 -> **JPA/Hibernate**
- 极致性能、特殊数据库操作 -> **原生 JDBC**

---

## 1.2 MyBatis 整体架构图

```
+-------------------------------------------------------------------------+
|                        MyBatis 整体架构                                  |
+-------------------------------------------------------------------------+
|                                                                           |
|  +-----------------------------------------------------------------------+|
|  |                        【接口层】                                      ||
|  |                                                                        ||
|  |   SqlSession API  <->  Mapper 接口代理                                ||
|  |   +------------+      +-------------------------------------+         ||
|  |   | SqlSession |      |  UserMapper / OrderMapper / ...     |         ||
|  |   | .select()  |      |  (JDK 动态代理生成实现类)             |         ||
|  |   | .insert()  |      +-------------------------------------+         ||
|  |   | .update()  |                                                       ||
|  |   | .delete()  |                                                       ||
|  |   +------------+                                                       ||
|  +-----------------------------------------------------------------------+|
|                              |                                            |
|                              v                                            |
|  +-----------------------------------------------------------------------+|
|  |                       【数据处理层】                                   ||
|  |                                                                        ||
|  |  +--------------+  +---------------+  +------------------------+      ||
|  |  | 参数映射      |  |  SQL解析       |  |  结果映射               |     ||
|  |  | ParameterHan |  |  SqlSource    |  |  ResultSetHandler      |     ||
|  |  |    dler      |  |  动态SQL解析   |  |  TypeHandler转换        |     ||
|  |  +--------------+  +---------------+  +------------------------+      ||
|  |                                                                        ||
|  |  +-------------------------------------------------------------+      ||
|  |  |                    Executor（执行器）                         |     ||
|  |  |  SimpleExecutor | ReuseExecutor | BatchExecutor              |     ||
|  |  |         ^ CachingExecutor（装饰者模式）                       |     ||
|  |  +-------------------------------------------------------------+      ||
|  +-----------------------------------------------------------------------+|
|                              |                                            |
|                              v                                            |
|  +-----------------------------------------------------------------------+|
|  |                       【框架支撑层】                                   ||
|  |                                                                        ||
|  |  +----------+  +----------+  +----------+  +------------------+      ||
|  |  | 事务管理  |  |连接池管理 |  | 缓存管理  |  | SQL语句配置      |     ||
|  |  |Transactn |  |DataSource|  | Cache    |  | MappedStatement |     ||
|  |  +----------+  +----------+  +----------+  +------------------+      ||
|  |                                                                        ||
|  |  +----------+  +--------------------------------------------+        ||
|  |  | 插件机制  |  |        类型处理器注册表                      |        ||
|  |  | Plugins  |  |      TypeHandlerRegistry                   |        ||
|  |  +----------+  +--------------------------------------------+        ||
|  +-----------------------------------------------------------------------+|
|                              |                                            |
|                              v                                            |
|  +-----------------------------------------------------------------------+|
|  |                      【引导层（初始化层）】                             ||
|  |                                                                        ||
|  |  +-------------------------+   +-------------------------------+      ||
|  |  |  基于 XML 配置           |   |  基于 Java API 配置           |      ||
|  |  |  mybatis-config.xml     |   |  Configuration 对象           |      ||
|  |  |  Mapper XML 文件         |   |  @Configuration + @Bean       |      ||
|  |  +-------------------------+   +-------------------------------+      ||
|  |                  |                                                     ||
|  |          SqlSessionFactoryBuilder                                      ||
|  |                  |                                                     ||
|  |          SqlSessionFactory（单例，线程安全）                            ||
|  +-----------------------------------------------------------------------+|
|                                                                           |
+-------------------------------------------------------------------------+
```

---

## 1.3 核心组件详解

### 1.3.1 SqlSessionFactory

`SqlSessionFactory` 是 MyBatis 的核心工厂类，用于创建 `SqlSession` 实例。它是**线程安全的**，应该在应用程序级别作为单例使用。

```java
// SqlSessionFactory 的创建过程
String resource = "mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory =
    new SqlSessionFactoryBuilder().build(inputStream);

// 从 SqlSessionFactory 获取 SqlSession
try (SqlSession session = sqlSessionFactory.openSession()) {
    UserMapper mapper = session.getMapper(UserMapper.class);
    User user = mapper.selectById(1L);
}
```

**SqlSessionFactory 的方法：**

```
SqlSessionFactory
  openSession()                        // 开启一个事务（autoCommit=false）
  openSession(boolean autoCommit)      // 指定是否自动提交
  openSession(Connection conn)         // 使用指定连接
  openSession(TransactionIsolationLevel level)  // 指定事务隔离级别
  openSession(ExecutorType execType)   // 指定执行器类型
  getConfiguration()                   // 获取全局配置对象
```

### 1.3.2 SqlSession

`SqlSession` 是 MyBatis 最核心的接口，它包含了执行 SQL 语句所有必要的方法。**SqlSession 是非线程安全的**，每次使用都应该从 SqlSessionFactory 中获取新的实例，并在使用完毕后关闭。

```java
public interface SqlSession extends Closeable {
    // 查询单个结果
    <T> T selectOne(String statement);
    <T> T selectOne(String statement, Object parameter);

    // 查询多个结果
    <E> List<E> selectList(String statement);
    <E> List<E> selectList(String statement, Object parameter);
    <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds);

    // 查询 Map
    <K, V> Map<K, V> selectMap(String statement, String mapKey);

    // 游标查询（大数据量）
    <T> Cursor<T> selectCursor(String statement);

    // 插入
    int insert(String statement);
    int insert(String statement, Object parameter);

    // 更新
    int update(String statement);
    int update(String statement, Object parameter);

    // 删除
    int delete(String statement);
    int delete(String statement, Object parameter);

    // 事务控制
    void commit();
    void commit(boolean force);
    void rollback();
    void rollback(boolean force);

    // 清除缓存
    void clearCache();

    // 获取 Mapper 代理
    <T> T getMapper(Class<T> type);

    // 获取数据库连接
    Connection getConnection();

    // 关闭
    void close();
}
```

### 1.3.3 Executor（执行器）

Executor 是 MyBatis 的 SQL 执行引擎，负责与数据库的交互。

```
Executor（接口）
  BaseExecutor（抽象类，实现一级缓存逻辑）
    SimpleExecutor     // 简单执行器，每次新建 Statement
    ReuseExecutor      // 复用执行器，缓存 PreparedStatement
    BatchExecutor      // 批量执行器，addBatch/executeBatch
  CachingExecutor      // 缓存执行器（装饰者模式，包装上面三种）
```

### 1.3.4 MappedStatement

`MappedStatement` 封装了一个 XML 中的 `<select|update|insert|delete>` 标签的所有信息：

```
MappedStatement
  id                  // 语句的全限定名（namespace + "." + id）
  sqlSource           // SQL 源，包含动态SQL逻辑
  sqlCommandType      // 语句类型（SELECT/INSERT/UPDATE/DELETE）
  resultMaps          // 结果映射列表
  parameterMap        // 参数映射
  statementType       // STATEMENT/PREPARED/CALLABLE
  fetchSize           // 抓取大小
  timeout             // 超时时间
  useCache            // 是否使用二级缓存
  flushCacheRequired  // 是否刷新缓存
  resultSetType       // 结果集类型
  keyGenerator        // 主键生成器
```

### 1.3.5 ParameterHandler（参数处理器）

负责将 Java 对象中的参数值设置到 `PreparedStatement` 中。

```java
public interface ParameterHandler {
    // 获取参数对象
    Object getParameterObject();

    // 将参数设置到 PreparedStatement
    void setParameters(PreparedStatement ps) throws SQLException;
}
```

默认实现是 `DefaultParameterHandler`，它通过 `TypeHandler` 将 Java 类型转换为 JDBC 类型。

### 1.3.6 ResultSetHandler（结果集处理器）

负责将 `ResultSet` 转换为 Java 对象。

```java
public interface ResultSetHandler {
    // 处理结果集，转换为 List
    <E> List<E> handleResultSets(Statement stmt) throws SQLException;

    // 处理游标结果集
    <E> Cursor<E> handleCursorResultSets(Statement stmt) throws SQLException;

    // 处理存储过程输出参数
    void handleOutputParameters(CallableStatement cs) throws SQLException;
}
```

### 1.3.7 StatementHandler（语句处理器）

负责创建和执行 SQL 语句，是对 JDBC Statement 的封装。

```java
public interface StatementHandler {
    Statement prepare(Connection connection, Integer transactionTimeout)
        throws SQLException;

    void parameterize(Statement statement) throws SQLException;

    void batch(Statement statement) throws SQLException;

    int update(Statement statement) throws SQLException;

    <E> List<E> query(Statement statement, ResultHandler resultHandler)
        throws SQLException;

    BoundSql getBoundSql();

    ParameterHandler getParameterHandler();
}
```

---

## 1.4 MyBatis 完整执行流程

```
                    MyBatis 执行流程详解

用户代码调用
    |
    v
+----------------------------------+
|  UserMapper.selectById(1L)       |  <- Mapper 接口方法调用
+----------------------------------+
    |  JDK动态代理拦截
    v
+----------------------------------+
|  MapperProxy.invoke()            |
|  - 根据方法名找到 MapperMethod   |
|  - 调用 sqlSession.selectOne()  |
+----------------------------------+
    |
    v
+----------------------------------+
|  DefaultSqlSession.selectOne()   |
|  - 找到 MappedStatement         |
|  - 调用 executor.query()        |
+----------------------------------+
    |
    v
+----------------------------------+
|  CachingExecutor.query()         |  <- 二级缓存检查
|  - 构建 CacheKey                |
|  - 查二级缓存                   |
|  - 未命中，转 BaseExecutor      |
+----------------------------------+
    |
    v
+----------------------------------+
|  BaseExecutor.query()            |  <- 一级缓存检查
|  - 构建 CacheKey                |
|  - 查一级缓存 (localCache)      |
|  - 未命中，查数据库              |
+----------------------------------+
    |
    v
+----------------------------------+
|  SimpleExecutor.doQuery()        |
|  - 创建 StatementHandler        |
|  - StatementHandler.prepare()   |
|    - 从连接池获取 Connection    |
|    - conn.prepareStatement(sql) |
+----------------------------------+
    |
    v
+----------------------------------+
|  StatementHandler.parameterize() |  <- 参数设置
|  - ParameterHandler.setParams() |
|  - TypeHandler.setParameter()   |
|    - ps.setLong(1, 1L)         |
+----------------------------------+
    |
    v
+----------------------------------+
|  PreparedStatement.execute()     |  <- 执行 SQL
|  SELECT id,name FROM user        |
|  WHERE id = 1                   |
+----------------------------------+
    |  返回 ResultSet
    v
+----------------------------------+
|  ResultSetHandler.handle()       |  <- 结果集映射
|  - 遍历 ResultSet 每一行        |
|  - 根据 ResultMap 映射对象      |
|  - TypeHandler 类型转换         |
|    - rs.getLong("id") -> id    |
|    - rs.getString("name")      |
+----------------------------------+
    |
    v
+----------------------------------+
|  写入一级缓存                    |
|  写入二级缓存（如果开启）         |
+----------------------------------+
    |
    v
   返回 User 对象给调用方
```

---

# Part 2: MyBatis 核心配置深度解析 <a id="part-2"></a>

## 2.1 mybatis-config.xml 完整结构

`mybatis-config.xml` 是 MyBatis 的全局配置文件，其内部元素的**顺序必须严格按照规定**，否则会报 DTD 验证错误。

配置文件元素的合法顺序（必须按此顺序）：
1. properties
2. settings
3. typeAliases
4. typeHandlers
5. objectFactory
6. objectWrapperFactory
7. reflectorFactory
8. plugins
9. environments
10. databaseIdProvider
11. mappers

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

  <!-- 1. properties：引入外部属性文件或定义属性 -->
  <properties resource="jdbc.properties">
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
  </properties>

  <!-- 2. settings：MyBatis 全局行为设置 -->
  <settings>
    <setting name="cacheEnabled" value="true"/>
    <setting name="lazyLoadingEnabled" value="true"/>
    <setting name="mapUnderscoreToCamelCase" value="true"/>
    <setting name="logImpl" value="SLF4J"/>
  </settings>

  <!-- 3. typeAliases：类型别名 -->
  <typeAliases>
    <package name="com.example.entity"/>
  </typeAliases>

  <!-- 4. typeHandlers：类型处理器 -->
  <typeHandlers>
    <package name="com.example.handler"/>
  </typeHandlers>

  <!-- 6. plugins：插件（拦截器）-->
  <plugins>
    <plugin interceptor="com.github.pagehelper.PageInterceptor">
      <property name="helperDialect" value="mysql"/>
    </plugin>
  </plugins>

  <!-- 7. environments：数据库环境配置 -->
  <environments default="development">
    <environment id="development">
      <transactionManager type="JDBC"/>
      <dataSource type="POOLED">
        <property name="driver" value="${driver}"/>
        <property name="url" value="${url}"/>
        <property name="username" value="${username}"/>
        <property name="password" value="${password}"/>
      </dataSource>
    </environment>
    <environment id="production">
      <transactionManager type="MANAGED"/>
      <dataSource type="JNDI">
        <property name="data_source" value="java:comp/env/jdbc/MyDB"/>
      </dataSource>
    </environment>
  </environments>

  <!-- 8. databaseIdProvider：数据库厂商标识 -->
  <databaseIdProvider type="DB_VENDOR">
    <property name="MySQL" value="mysql"/>
    <property name="Oracle" value="oracle"/>
    <property name="SQL Server" value="sqlserver"/>
  </databaseIdProvider>

  <!-- 9. mappers：Mapper 注册 -->
  <mappers>
    <!-- 方式1：XML文件路径 -->
    <mapper resource="mapper/UserMapper.xml"/>
    <!-- 方式2：Java 全限定类名 -->
    <mapper class="com.example.mapper.UserMapper"/>
    <!-- 方式3：URL（少用）-->
    <mapper url="file:///mapper/UserMapper.xml"/>
    <!-- 方式4：扫描包（推荐）-->
    <package name="com.example.mapper"/>
  </mappers>

</configuration>
```

---

## 2.2 environments（数据库环境配置）

`environments` 用于配置多个数据库环境，通过 `default` 属性指定当前使用的环境。

### transactionManager 事务管理器

| 类型 | 说明 | 使用场景 |
|------|------|----------|
| `JDBC` | 使用 JDBC 的事务管理（commit/rollback）| 独立应用，不使用Spring |
| `MANAGED` | 由容器管理事务（如 JEE 容器）| Spring 环境（Spring 接管事务）|

> **重要提示：** 在 Spring + MyBatis 整合时，不需要配置 MyBatis 的 `transactionManager`，因为 Spring 的 `PlatformTransactionManager` 会完全接管事务管理。

### dataSource 数据源

| 类型 | 说明 | 适用场景 |
|------|------|----------|
| `UNPOOLED` | 每次请求时打开/关闭连接，无连接池 | 简单应用、测试 |
| `POOLED` | MyBatis 内置连接池 | 小型项目 |
| `JNDI` | 使用 JNDI 数据源 | 传统 JavaEE 项目 |
| 自定义 | 实现 `DataSourceFactory` 接口 | 集成 Druid/HikariCP |

**POOLED 数据源配置参数：**

```xml
<dataSource type="POOLED">
  <property name="driver" value="com.mysql.cj.jdbc.Driver"/>
  <property name="url" value="jdbc:mysql://localhost:3306/mydb"/>
  <property name="username" value="root"/>
  <property name="password" value="123456"/>

  <!-- 连接池最大连接数，默认10 -->
  <property name="poolMaximumActiveConnections" value="10"/>
  <!-- 最大空闲连接数，默认5 -->
  <property name="poolMaximumIdleConnections" value="5"/>
  <!-- 等待获取连接的最大时间（毫秒），默认20000 -->
  <property name="poolMaximumCheckoutTime" value="20000"/>
  <!-- 连接检测超时时间（毫秒），默认20000 -->
  <property name="poolTimeToWait" value="20000"/>
  <!-- 发送到数据库的测试 SQL -->
  <property name="poolPingQuery" value="SELECT 1"/>
  <!-- 是否开启连接测试，默认false -->
  <property name="poolPingEnabled" value="true"/>
  <!-- 连接超过该时间必须测试，默认0（不测试）-->
  <property name="poolPingConnectionsNotUsedFor" value="3600000"/>
</dataSource>
```

---

## 2.3 typeAliases（类型别名）

类型别名可以为 Java 类型设置短名字，减少全类名书写。

### MyBatis 内置别名

MyBatis 为 Java 基础类型提供了大量内置别名：

| 别名 | 映射的类型 | 别名 | 映射的类型 |
|------|-----------|------|-----------|
| `_byte` | byte | `string` | String |
| `_short` | short | `byte` | Byte |
| `_int` | int | `short` | Short |
| `_integer` | int | `int` | Integer |
| `_long` | long | `integer` | Integer |
| `_double` | double | `long` | Long |
| `_float` | float | `float` | Float |
| `_boolean` | boolean | `double` | Double |
| `date` | Date | `boolean` | Boolean |
| `decimal` | BigDecimal | `map` | Map |
| `bigdecimal` | BigDecimal | `hashmap` | HashMap |
| `object` | Object | `list` | List |
| `date[]` | Date[] | `arraylist` | ArrayList |
| `map` | Map | `collection` | Collection |

### 自定义别名

```xml
<typeAliases>
  <!-- 方式1：为单个类设置别名 -->
  <typeAlias alias="User" type="com.example.entity.User"/>
  <typeAlias alias="Order" type="com.example.entity.Order"/>

  <!-- 方式2：扫描整个包（推荐，别名为类名首字母小写）-->
  <package name="com.example.entity"/>
</typeAliases>
```

**使用注解设置别名：**

```java
// 在类上使用 @Alias 注解
@Alias("user")
public class User {
    // ...
}
```

**在 XML 中使用别名：**

```xml
<!-- 不使用别名 -->
<select id="selectById" resultType="com.example.entity.User">
    SELECT * FROM user WHERE id = #{id}
</select>

<!-- 使用别名 -->
<select id="selectById" resultType="User">
    SELECT * FROM user WHERE id = #{id}
</select>
```

---

## 2.4 typeHandlers（类型处理器）

TypeHandler 负责 Java 类型与 JDBC 类型之间的相互转换，这是 MyBatis 类型系统的核心。

### 内置 TypeHandler 体系

```
TypeHandler 体系
  BaseTypeHandler<T>（抽象基类）
    StringTypeHandler         // String  <->  VARCHAR
    IntegerTypeHandler        // Integer <->  INTEGER
    LongTypeHandler           // Long    <->  BIGINT
    DoubleTypeHandler         // Double  <->  DOUBLE
    BigDecimalTypeHandler     // BigDecimal <-> DECIMAL
    BooleanTypeHandler        // Boolean <->  BOOLEAN
    DateTypeHandler           // Date    <->  TIMESTAMP
    LocalDateTimeTypeHandler  // LocalDateTime <-> TIMESTAMP
    LocalDateTypeHandler      // LocalDate <-> DATE
    LocalTimeTypeHandler      // LocalTime <-> TIME
    EnumTypeHandler           // Enum    <->  VARCHAR（存枚举名）
    EnumOrdinalTypeHandler    // Enum    <->  INTEGER（存枚举序号）
    JsonTypeHandler           // Object  <->  JSON（3.4.3+）
```

### 自定义 TypeHandler 案例1：List<String> 转逗号分隔字符串

**场景：将 Java 的 `List<String>` 与数据库的 `VARCHAR`（逗号分隔字符串）互相转换**

```java
package com.example.handler;

import org.apache.ibatis.type.BaseTypeHandler;
import org.apache.ibatis.type.JdbcType;
import org.apache.ibatis.type.MappedJdbcTypes;
import org.apache.ibatis.type.MappedTypes;

import java.sql.*;
import java.util.Arrays;
import java.util.Collections;
import java.util.List;

/**
 * 将 List<String> 与数据库 VARCHAR（逗号分隔）互转的 TypeHandler
 * 例如：["Java", "Python", "Go"] <-> "Java,Python,Go"
 */
@MappedTypes(List.class)             // 指定处理的 Java 类型
@MappedJdbcTypes(JdbcType.VARCHAR)   // 指定处理的 JDBC 类型
public class StringListTypeHandler extends BaseTypeHandler<List<String>> {

    private static final String DELIMITER = ",";

    /**
     * 将 Java 对象设置到 PreparedStatement（写入数据库）
     */
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                     List<String> parameter, JdbcType jdbcType)
            throws SQLException {
        String value = String.join(DELIMITER, parameter);
        ps.setString(i, value);
    }

    /**
     * 通过列名从 ResultSet 获取结果（读取数据库）
     */
    @Override
    public List<String> getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        return convertToList(rs.getString(columnName));
    }

    /**
     * 通过列索引从 ResultSet 获取结果
     */
    @Override
    public List<String> getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        return convertToList(rs.getString(columnIndex));
    }

    /**
     * 从 CallableStatement 获取结果（存储过程）
     */
    @Override
    public List<String> getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        return convertToList(cs.getString(columnIndex));
    }

    private List<String> convertToList(String value) {
        if (value == null || value.isEmpty()) {
            return Collections.emptyList();
        }
        return Arrays.asList(value.split(DELIMITER));
    }
}
```

**注册自定义 TypeHandler：**

```xml
<!-- mybatis-config.xml -->
<typeHandlers>
  <!-- 方式1：指定具体类 -->
  <typeHandler handler="com.example.handler.StringListTypeHandler"/>

  <!-- 方式2：扫描包（推荐）-->
  <package name="com.example.handler"/>
</typeHandlers>
```

**在 Mapper XML 中使用：**

```xml
<!-- 写入时指定 typeHandler -->
<insert id="insertUser">
    INSERT INTO user (id, tags)
    VALUES (#{id}, #{tags, typeHandler=com.example.handler.StringListTypeHandler})
</insert>

<!-- 在 resultMap 中指定 typeHandler -->
<resultMap id="userResultMap" type="User">
    <id property="id" column="id"/>
    <result property="tags" column="tags"
            typeHandler="com.example.handler.StringListTypeHandler"/>
</resultMap>
```

### 自定义 TypeHandler 案例2：枚举类型处理

```java
/**
 * 订单状态枚举
 */
public enum OrderStatus {
    PENDING(0, "待支付"),
    PAID(1, "已支付"),
    SHIPPED(2, "已发货"),
    COMPLETED(3, "已完成"),
    CANCELLED(4, "已取消");

    private final int code;
    private final String desc;

    OrderStatus(int code, String desc) {
        this.code = code;
        this.desc = desc;
    }

    public int getCode() { return code; }
    public String getDesc() { return desc; }

    public static OrderStatus fromCode(int code) {
        for (OrderStatus status : values()) {
            if (status.code == code) return status;
        }
        throw new IllegalArgumentException("Unknown order status code: " + code);
    }
}

/**
 * OrderStatus 枚举的 TypeHandler（存/取整数code）
 */
@MappedTypes(OrderStatus.class)
@MappedJdbcTypes(JdbcType.INTEGER)
public class OrderStatusTypeHandler extends BaseTypeHandler<OrderStatus> {

    @Override
    public void setNonNullParameter(PreparedStatement ps, int i,
                                     OrderStatus parameter, JdbcType jdbcType)
            throws SQLException {
        ps.setInt(i, parameter.getCode());
    }

    @Override
    public OrderStatus getNullableResult(ResultSet rs, String columnName)
            throws SQLException {
        int code = rs.getInt(columnName);
        return rs.wasNull() ? null : OrderStatus.fromCode(code);
    }

    @Override
    public OrderStatus getNullableResult(ResultSet rs, int columnIndex)
            throws SQLException {
        int code = rs.getInt(columnIndex);
        return rs.wasNull() ? null : OrderStatus.fromCode(code);
    }

    @Override
    public OrderStatus getNullableResult(CallableStatement cs, int columnIndex)
            throws SQLException {
        int code = cs.getInt(columnIndex);
        return cs.wasNull() ? null : OrderStatus.fromCode(code);
    }
}
```

---

## 2.5 settings（全局设置，30+ 参数详解）

`settings` 是 MyBatis 中最重要的调整设置，它们会改变 MyBatis 的运行时行为。

```xml
<settings>
  <!-- ==================== 缓存设置 ==================== -->

  <!-- 全局开关：是否开启二级缓存，默认 true -->
  <setting name="cacheEnabled" value="true"/>

  <!-- ==================== 懒加载设置 ==================== -->

  <!-- 是否开启延迟加载，默认 false -->
  <setting name="lazyLoadingEnabled" value="true"/>

  <!-- 是否开启激进懒加载
       true: 任何方法调用都会触发所有属性的加载
       false: 按需加载（默认，3.4.1+）-->
  <setting name="aggressiveLazyLoading" value="false"/>

  <!-- 触发懒加载的方法名（默认 equals,clone,hashCode,toString）-->
  <setting name="lazyLoadTriggerMethods" value="equals,clone,hashCode,toString"/>

  <!-- ==================== 映射设置 ==================== -->

  <!-- 是否允许单个语句返回多结果集，默认 true -->
  <setting name="multipleResultSetsEnabled" value="true"/>

  <!-- 是否使用列标签代替列名，默认 true -->
  <setting name="useColumnLabel" value="true"/>

  <!-- 是否允许 JDBC 支持自动生成主键，默认 false -->
  <setting name="useGeneratedKeys" value="false"/>

  <!--
    自动映射行为：
    NONE    - 取消自动映射
    PARTIAL - 只会自动映射没有定义嵌套结果映射的字段（默认）
    FULL    - 自动映射任何复杂的结果集
  -->
  <setting name="autoMappingBehavior" value="PARTIAL"/>

  <!--
    自动映射时遇到未知列的行为：
    NONE    - 什么都不做（默认）
    WARNING - 输出警告日志
    FAILING - 抛出异常
  -->
  <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>

  <!--
    默认执行器类型：
    SIMPLE - 普通执行器（默认）
    REUSE  - 复用预处理语句的执行器
    BATCH  - 批量执行器
  -->
  <setting name="defaultExecutorType" value="SIMPLE"/>

  <!-- 超时时间（秒），默认无超时 -->
  <setting name="defaultStatementTimeout" value="30"/>

  <!-- 结果集抓取大小，默认无设置 -->
  <setting name="defaultFetchSize" value="100"/>

  <!-- 是否允许在嵌套语句中使用分页 RowBounds，默认 false -->
  <setting name="safeRowBoundsEnabled" value="false"/>

  <!-- 是否允许在嵌套语句中使用 ResultHandler，默认 true -->
  <setting name="safeResultHandlerEnabled" value="true"/>

  <!-- ==================== 命名转换 ==================== -->

  <!--
    开启驼峰命名自动映射（默认 false）
    开启后：数据库列 user_name -> Java 属性 userName
    这是最常用的配置项之一！
  -->
  <setting name="mapUnderscoreToCamelCase" value="true"/>

  <!-- ==================== 一级缓存设置 ==================== -->

  <!--
    本地缓存范围：
    SESSION   - 缓存整个会话期间的所有查询（默认）
    STATEMENT - 仅在当前语句中缓存，语句执行后立即清除
  -->
  <setting name="localCacheScope" value="SESSION"/>

  <!-- ==================== JDBC 类型设置 ==================== -->

  <!-- null 值的默认 JDBC 类型，默认 OTHER
       注意：某些数据库（如 Oracle）不支持 OTHER，需要改为 NULL -->
  <setting name="jdbcTypeForNull" value="NULL"/>

  <!-- ==================== 日志设置 ==================== -->

  <!--
    指定 MyBatis 使用的日志框架：
    SLF4J | LOG4J | LOG4J2 | JDK_LOGGING |
    COMMONS_LOGGING | STDOUT_LOGGING | NO_LOGGING
  -->
  <setting name="logImpl" value="SLF4J"/>

  <!-- 日志前缀，用于过滤日志 -->
  <!-- <setting name="logPrefix" value="dao."/> -->

  <!-- ==================== 代理设置 ==================== -->

  <!--
    懒加载代理工厂：
    CGLIB     - 使用 CGLIB 代理（可以代理没有接口的类）
    JAVASSIST - 使用 Javassist 代理（3.3+默认）
  -->
  <setting name="proxyFactory" value="JAVASSIST"/>

  <!-- ==================== 其他设置 ==================== -->

  <!-- 结果集类型：FORWARD_ONLY|SCROLL_SENSITIVE|SCROLL_INSENSITIVE|DEFAULT -->
  <setting name="defaultResultSetType" value="DEFAULT"/>

  <!-- 当结果为 null 时是否调用 setter，默认 false -->
  <setting name="callSettersOnNulls" value="false"/>

  <!-- 是否为空行返回空对象实例（而不是null），默认 false -->
  <setting name="returnInstanceForEmptyRow" value="false"/>

  <!-- 是否收缩 SQL 中的空白，默认 false（3.5.5+）-->
  <setting name="shrinkWhitespacesInSql" value="false"/>

  <!-- 枚举默认 TypeHandler（默认 EnumTypeHandler）-->
  <setting name="defaultEnumTypeHandler"
           value="org.apache.ibatis.type.EnumTypeHandler"/>

  <!-- 指定一个提供 Configuration 实例的类（3.2.3+）-->
  <!-- <setting name="configurationFactory" value="com.example.MyConfigFactory"/> -->

</settings>
```

**最常用的 10 个 settings 配置：**

```
排名  配置项                          说明
----  -------------------------------- ---------------------------
 1    mapUnderscoreToCamelCase=true    下划线自动转驼峰（必备）
 2    cacheEnabled=true                开启二级缓存
 3    lazyLoadingEnabled=true          开启懒加载
 4    aggressiveLazyLoading=false      按需懒加载（非激进）
 5    useGeneratedKeys=true            使用数据库自增主键
 6    defaultExecutorType=SIMPLE       默认执行器
 7    logImpl=SLF4J                    日志实现
 8    defaultStatementTimeout=30       语句超时时间（秒）
 9    localCacheScope=SESSION          一级缓存范围
10    autoMappingBehavior=PARTIAL      自动映射行为
```

---

## 2.6 mappers（Mapper 注册方式详解）

```xml
<mappers>
  <!--
    方式1：通过 XML 文件路径注册（classpath相对路径）
    优点：精确控制，路径清晰
    缺点：每个 Mapper 都要单独配置
  -->
  <mapper resource="com/example/mapper/UserMapper.xml"/>

  <!--
    方式2：通过 Java 接口全限定名注册
    要求：接口必须在同一包下有同名的 XML 文件，或使用注解 SQL
    优点：无需 XML
    缺点：XML 文件放置有限制
  -->
  <mapper class="com.example.mapper.UserMapper"/>

  <!--
    方式3：通过包扫描注册（最常用，推荐）
    要求：XML 与接口同名同路径，或使用注解 SQL
    优点：一次配置，批量注册
  -->
  <package name="com.example.mapper"/>
</mappers>
```

---

## 2.7 Spring Boot 集成完整配置

在 Spring Boot 项目中，通过 `application.yml` 来配置 MyBatis：

```yaml
# application.yml 完整 MyBatis 配置

spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai&useSSL=false&allowPublicKeyRetrieval=true
    username: root
    password: 123456
    # 使用 Druid 连接池
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      initial-size: 5
      min-idle: 5
      max-active: 20
      max-wait: 60000
      time-between-eviction-runs-millis: 60000
      min-evictable-idle-time-millis: 300000
      validation-query: SELECT 1
      test-while-idle: true
      test-on-borrow: false
      test-on-return: false
      # 开启 Druid 监控 SQL
      filters: stat,wall,slf4j
      # Web 监控（Druid 控制台）
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        login-username: admin
        login-password: admin123
      web-stat-filter:
        enabled: true
        url-pattern: /*
        exclusions: "*.js,*.gif,*.jpg,*.css,/druid/*"

# MyBatis 配置
mybatis:
  # Mapper XML 文件位置（支持通配符）
  mapper-locations: classpath*:mapper/**/*.xml
  # 实体类别名包扫描
  type-aliases-package: com.example.entity
  # 类型处理器包扫描
  type-handlers-package: com.example.handler
  # 如需引入完整配置文件（与下面 configuration 二选一）
  # config-location: classpath:mybatis-config.xml
  configuration:
    # 最常用：驼峰命名映射（强烈推荐）
    map-underscore-to-camel-case: true
    # 开启二级缓存（使用时要注意序列化问题）
    cache-enabled: true
    # 开启懒加载
    lazy-loading-enabled: true
    # 非激进懒加载（按需加载）
    aggressive-lazy-loading: false
    # 日志实现（开发环境推荐 STDOUT，生产推荐 SLF4J）
    log-impl: org.apache.ibatis.logging.slf4j.Slf4jImpl
    # 全局超时时间（秒）
    default-statement-timeout: 30
    # NULL 参数的 JDBC 类型（Oracle 需要设置为 NULL）
    jdbc-type-for-null: NULL
    # 默认执行器类型
    default-executor-type: simple
    # 自动映射行为
    auto-mapping-behavior: partial
    # 未知列处理
    auto-mapping-unknown-column-behavior: warning
    # 结果为空时返回空对象
    return-instance-for-empty-row: false
```

**Spring Boot 启动类配置：**

```java
package com.example;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
@MapperScan("com.example.mapper")  // 扫描 Mapper 接口
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**pom.xml 依赖（完整）：**

```xml
<dependencies>
    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- MyBatis Spring Boot Starter（含 mybatis-spring 和 mybatis）-->
    <dependency>
        <groupId>org.mybatis.spring.boot</groupId>
        <artifactId>mybatis-spring-boot-starter</artifactId>
        <version>3.0.3</version>
    </dependency>

    <!-- MySQL 驱动 -->
    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Druid 连接池 -->
    <dependency>
        <groupId>com.alibaba</groupId>
        <artifactId>druid-spring-boot-starter</artifactId>
        <version>1.2.20</version>
    </dependency>

    <!-- Lombok（简化实体类）-->
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

    <!-- PageHelper 分页（可选）-->
    <dependency>
        <groupId>com.github.pagehelper</groupId>
        <artifactId>pagehelper-spring-boot-starter</artifactId>
        <version>1.4.7</version>
    </dependency>
</dependencies>
```

---

# Part 3: Mapper XML 深度解析 <a id="part-3"></a>

## 3.1 Mapper XML 文件结构

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!--
  namespace 必须与 Mapper 接口的全限定名一致
  这是 MyBatis 将 XML 与接口绑定的关键
-->
<mapper namespace="com.example.mapper.UserMapper">

  <!-- SQL 片段复用 -->
  <sql id="baseColumns">
    id, username, email, phone, create_time, update_time
  </sql>

  <!-- 结果映射 -->
  <resultMap id="userResultMap" type="com.example.entity.User">
    <id property="id" column="id" jdbcType="BIGINT"/>
    <result property="username" column="username" jdbcType="VARCHAR"/>
    <result property="email" column="email" jdbcType="VARCHAR"/>
  </resultMap>

  <!-- 查询 -->
  <select id="selectById" resultMap="userResultMap">
    SELECT <include refid="baseColumns"/>
    FROM user WHERE id = #{id}
  </select>

  <!-- 插入 -->
  <insert id="insert" parameterType="User" useGeneratedKeys="true" keyProperty="id">
    INSERT INTO user (username, email) VALUES (#{username}, #{email})
  </insert>

  <!-- 更新 -->
  <update id="updateById" parameterType="User">
    UPDATE user SET username = #{username}, email = #{email}
    WHERE id = #{id}
  </update>

  <!-- 删除 -->
  <delete id="deleteById">
    DELETE FROM user WHERE id = #{id}
  </delete>

</mapper>
```

---

## 3.2 select 标签详解

```xml
<select
  id="selectUsers"             <!-- 与 Mapper 接口方法名对应（必须匹配）-->
  parameterType="User"         <!-- 参数类型（通常可省略，MyBatis 自动推断）-->
  resultType="User"            <!-- 结果类型（简单对象时使用）-->
  resultMap="userResultMap"    <!-- 结果映射（复杂映射时使用，与 resultType 互斥）-->
  flushCache="false"           <!-- 是否每次调用都清空缓存，默认 false -->
  useCache="true"              <!-- 是否使用二级缓存，默认 true（select才有此属性）-->
  timeout="30"                 <!-- 超时时间（秒），覆盖全局设置 -->
  fetchSize="100"              <!-- 每次批量取回的行数，建议设置以提高性能 -->
  statementType="PREPARED"     <!-- STATEMENT|PREPARED|CALLABLE，默认PREPARED -->
  resultSetType="FORWARD_ONLY" <!-- FORWARD_ONLY|SCROLL_SENSITIVE|SCROLL_INSENSITIVE -->
  databaseId="mysql"           <!-- 数据库厂商标识（多数据库支持）-->
  resultOrdered="false"        <!-- 嵌套结果集是否有序，默认false -->
  resultSets=""                <!-- 多结果集的名称列表（存储过程）-->
>
  SELECT * FROM user WHERE id = #{id}
</select>
```

**select 常见用法示例：**

```xml
<!-- 查询单个对象 -->
<select id="selectById" resultType="User">
    SELECT id, username, email FROM user WHERE id = #{id}
</select>

<!-- 查询列表 -->
<select id="selectAll" resultType="User">
    SELECT id, username, email FROM user ORDER BY create_time DESC
</select>

<!-- 查询总数 -->
<select id="countAll" resultType="long">
    SELECT COUNT(*) FROM user WHERE deleted = 0
</select>

<!-- 查询返回 Map -->
<select id="selectAsMap" resultType="java.util.Map">
    SELECT id, username, email FROM user WHERE id = #{id}
</select>

<!-- 使用 RowBounds 进行逻辑分页（不推荐，会全量查询然后截取）-->
<select id="selectWithRowBounds" resultType="User">
    SELECT id, username, email FROM user
</select>
```

---

## 3.3 insert 标签详解

```xml
<!-- 基本插入 -->
<insert id="insert" parameterType="User">
    INSERT INTO user (username, email, create_time)
    VALUES (#{username}, #{email}, NOW())
</insert>

<!-- 获取自增主键（MySQL / H2 / SQL Server）-->
<insert id="insertWithKey"
        parameterType="User"
        useGeneratedKeys="true"    <!-- 告诉 MyBatis 使用 JDBC 的 getGeneratedKeys() -->
        keyProperty="id"           <!-- 将生成的主键值回填到 User 对象的 id 属性 -->
        keyColumn="id">            <!-- 数据库表的主键列名（多主键时用逗号分隔）-->
    INSERT INTO user (username, email) VALUES (#{username}, #{email})
</insert>

<!-- 批量插入（使用 foreach）-->
<insert id="insertBatch" parameterType="list">
    INSERT INTO user (username, email, create_time) VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.username}, #{user.email}, NOW())
    </foreach>
</insert>

<!-- Oracle 序列方式获取主键（BEFORE 代表在 INSERT 之前执行）-->
<insert id="insertOracle" parameterType="User">
    <selectKey keyProperty="id" resultType="Long" order="BEFORE">
        SELECT user_seq.NEXTVAL FROM DUAL
    </selectKey>
    INSERT INTO user (id, username, email) VALUES (#{id}, #{username}, #{email})
</insert>

<!-- MySQL 使用 LAST_INSERT_ID()（AFTER 代表在 INSERT 之后执行）-->
<insert id="insertMysqlSelectKey" parameterType="User">
    INSERT INTO user (username, email) VALUES (#{username}, #{email})
    <selectKey keyProperty="id" resultType="Long" order="AFTER">
        SELECT LAST_INSERT_ID()
    </selectKey>
</insert>
```

**useGeneratedKeys 完整示例（推荐）：**

```java
// Mapper 接口
@Mapper
public interface UserMapper {
    int insert(User user);
}

// 调用
User user = new User();
user.setUsername("张三");
user.setEmail("zhangsan@example.com");
System.out.println("插入前 id: " + user.getId());  // null

userMapper.insert(user);

System.out.println("插入后 id: " + user.getId());  // 1001（数据库自增回填）
```

---

## 3.4 update 和 delete 标签

```xml
<!-- update 标签 -->
<update id="updateById" parameterType="User">
    UPDATE user
    SET username    = #{username},
        email       = #{email},
        update_time = NOW()
    WHERE id = #{id}
</update>

<!-- 按条件更新（动态 SQL）-->
<update id="updateSelective" parameterType="User">
    UPDATE user
    <set>
        <if test="username != null">username = #{username},</if>
        <if test="email != null">email = #{email},</if>
        <if test="phone != null">phone = #{phone},</if>
        update_time = NOW(),
    </set>
    WHERE id = #{id}
</update>

<!-- delete 标签 -->
<delete id="deleteById">
    DELETE FROM user WHERE id = #{id}
</delete>

<!-- 批量删除（使用 foreach）-->
<delete id="deleteBatchByIds">
    DELETE FROM user WHERE id IN
    <foreach collection="list" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</delete>

<!-- 逻辑删除（不真正删除，设置 deleted=1）-->
<update id="logicalDeleteById">
    UPDATE user SET deleted = 1, update_time = NOW() WHERE id = #{id}
</update>
```

---

## 3.5 多参数传递方式

### 方式1：@Param 注解（推荐，最清晰）

```java
// Mapper 接口
@Mapper
public interface UserMapper {

    // 使用 @Param 命名参数
    List<User> selectByNameAndStatus(
        @Param("name") String name,
        @Param("status") Integer status
    );

    // 分页查询：多参数场景
    List<User> selectPage(
        @Param("offset") int offset,
        @Param("limit") int limit,
        @Param("keyword") String keyword
    );
}
```

```xml
<select id="selectByNameAndStatus" resultType="User">
    SELECT * FROM user
    WHERE username LIKE CONCAT('%', #{name}, '%')
    AND status = #{status}
</select>

<select id="selectPage" resultType="User">
    SELECT * FROM user
    <where>
        <if test="keyword != null and keyword != ''">
            AND username LIKE CONCAT('%', #{keyword}, '%')
        </if>
    </where>
    LIMIT #{limit} OFFSET #{offset}
</select>
```

### 方式2：使用 Map（灵活但失去类型安全）

```java
// Mapper 接口
List<User> selectByCondition(Map<String, Object> params);
```

```java
// 调用
Map<String, Object> params = new HashMap<>();
params.put("name", "张三");
params.put("status", 1);
List<User> users = userMapper.selectByCondition(params);
```

```xml
<select id="selectByCondition" resultType="User">
    SELECT * FROM user
    WHERE username LIKE CONCAT('%', #{name}, '%')
    AND status = #{status}
</select>
```

### 方式3：使用 JavaBean（推荐用于复杂查询条件）

```java
// 查询条件 DTO
@Data
public class UserQueryDTO {
    private String name;
    private String email;
    private Integer status;
    private Integer minAge;
    private Integer maxAge;
    private LocalDateTime createTimeStart;
    private LocalDateTime createTimeEnd;
    // getter/setter...
}

// Mapper 接口
List<User> selectByCondition(UserQueryDTO queryDTO);
```

```xml
<select id="selectByCondition" parameterType="UserQueryDTO" resultType="User">
    SELECT * FROM user
    <where>
        <if test="name != null and name != ''">
            AND username LIKE CONCAT('%', #{name}, '%')
        </if>
        <if test="email != null">
            AND email = #{email}
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
        <if test="minAge != null">
            AND age >= #{minAge}
        </if>
        <if test="maxAge != null">
            AND age &lt;= #{maxAge}
        </if>
    </where>
</select>
```

---

## 3.6 resultType vs resultMap

### resultType（简单映射，自动映射）

```xml
<!-- 返回单个 POJO 对象 -->
<select id="selectById" resultType="com.example.entity.User">
    SELECT id, username, email FROM user WHERE id = #{id}
</select>

<!-- 返回基本类型 -->
<select id="countUsers" resultType="int">
    SELECT COUNT(*) FROM user
</select>

<!-- 返回 String -->
<select id="selectUsernameById" resultType="String">
    SELECT username FROM user WHERE id = #{id}
</select>

<!-- 返回 Map（列名 -> 值）-->
<select id="selectAsMap" resultType="java.util.Map">
    SELECT id, username, email FROM user WHERE id = #{id}
</select>

<!-- 返回 List<Map> -->
<select id="selectAllAsMaps" resultType="java.util.Map">
    SELECT id, username, email FROM user
</select>
```

**使用 resultType 的前提：**
- 数据库列名与 Java 属性名一致
- 或者开启了 `mapUnderscoreToCamelCase=true`（下划线转驼峰）

### resultMap（高级映射，完全控制）

```xml
<!-- 基础 resultMap -->
<resultMap id="userBaseMap" type="com.example.entity.User">
    <!--
      id 标签：主键列映射
      - 有性能优化作用（MyBatis 用它来区分结果对象，避免重复实例化）
      - 对于一对多查询，id 标签非常重要
    -->
    <id property="id" column="id" jdbcType="BIGINT"/>

    <!-- result 标签：普通列映射 -->
    <result property="username" column="user_name" jdbcType="VARCHAR"/>
    <result property="email" column="email_address" jdbcType="VARCHAR"/>
    <result property="createTime" column="create_time" jdbcType="TIMESTAMP"/>

    <!-- 指定特定的 TypeHandler -->
    <result property="status" column="status"
            typeHandler="com.example.handler.OrderStatusTypeHandler"/>

    <!-- 指定列名和属性名都相同时可以省略，通过 autoMapping 自动处理 -->
    <!-- <result property="phone" column="phone"/> -->
</resultMap>

<!-- resultMap 继承（extends 属性复用已有映射）-->
<resultMap id="userDetailMap" type="com.example.entity.UserDetail"
           extends="userBaseMap">
    <result property="address" column="address"/>
    <result property="bio" column="bio"/>
    <result property="birthday" column="birthday" jdbcType="DATE"/>
</resultMap>
```

**resultMap 的所有子标签：**

```
<resultMap>
  <constructor>         -- 构造函数参数注入
    <idArg/>            -- 主键构造参数
    <arg/>              -- 普通构造参数
  </constructor>
  <id/>                 -- 主键列映射
  <result/>             -- 普通列映射
  <association/>        -- 一对一关联（has one）
  <collection/>         -- 一对多关联（has many）
  <discriminator/>      -- 鉴别器（根据列值选择不同映射）
    <case/>             -- 鉴别器的分支
</resultMap>
```

---

## 3.7 association（一对一关联映射）

### 场景说明

每个用户（User）有一个用户详情（UserProfile），是一对一关系。

```sql
-- 数据库表
CREATE TABLE user (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    username   VARCHAR(50) NOT NULL,
    email      VARCHAR(100),
    created_at DATETIME
);

CREATE TABLE user_profile (
    id         BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id    BIGINT NOT NULL UNIQUE,  -- 唯一外键
    real_name  VARCHAR(50),
    avatar     VARCHAR(200),
    bio        VARCHAR(500),
    FOREIGN KEY (user_id) REFERENCES user(id)
);
```

```java
// 实体类
@Data
public class User {
    private Long id;
    private String username;
    private String email;
    private UserProfile profile;  // 一对一关联
}

@Data
public class UserProfile {
    private Long id;
    private Long userId;
    private String realName;
    private String avatar;
    private String bio;
}
```

### 方式1：嵌套结果（JOIN 查询，一条 SQL，推荐）

```xml
<resultMap id="userWithProfileMap" type="User">
    <id property="id" column="u_id"/>
    <result property="username" column="u_username"/>
    <result property="email" column="u_email"/>

    <!--
      association 属性：
      property   - User 中关联对象的属性名
      javaType   - 关联对象的 Java 类型
      columnPrefix - 为关联表的列名添加前缀，避免列名冲突
    -->
    <association property="profile"
                 javaType="UserProfile"
                 columnPrefix="p_">
        <id property="id" column="id"/>
        <result property="userId" column="user_id"/>
        <result property="realName" column="real_name"/>
        <result property="avatar" column="avatar"/>
        <result property="bio" column="bio"/>
    </association>
</resultMap>

<select id="selectUserWithProfile" resultMap="userWithProfileMap">
    SELECT
        u.id        AS u_id,
        u.username  AS u_username,
        u.email     AS u_email,
        p.id        AS p_id,
        p.user_id   AS p_user_id,
        p.real_name AS p_real_name,
        p.avatar    AS p_avatar,
        p.bio       AS p_bio
    FROM user u
    LEFT JOIN user_profile p ON u.id = p.user_id
    WHERE u.id = #{id}
</select>
```

### 方式2：嵌套查询（子查询，支持懒加载）

```xml
<resultMap id="userWithProfileLazyMap" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <result property="email" column="email"/>

    <!--
      select    - 引用另一个查询的 id（用于执行子查询）
      column    - 作为子查询参数的列名
      fetchType - eager（立即加载）| lazy（懒加载），覆盖全局配置
    -->
    <association property="profile"
                 column="id"
                 select="selectProfileByUserId"
                 fetchType="lazy"/>
</resultMap>

<select id="selectUserById" resultMap="userWithProfileLazyMap">
    SELECT id, username, email FROM user WHERE id = #{id}
</select>

<select id="selectProfileByUserId" resultType="UserProfile">
    SELECT id, user_id, real_name, avatar, bio
    FROM user_profile WHERE user_id = #{userId}
</select>
```

---

## 3.8 collection（一对多关联映射）

### 场景说明

每个用户（User）有多个订单（Order），是一对多关系。

```java
// 实体类
@Data
public class User {
    private Long id;
    private String username;
    private List<Order> orders;  // 一对多
}

@Data
public class Order {
    private Long id;
    private Long userId;
    private String orderNo;
    private BigDecimal amount;
    private Integer status;
    private LocalDateTime createTime;
}
```

### 方式1：嵌套结果（联表查询）

```xml
<resultMap id="userWithOrdersMap" type="User">
    <id property="id" column="u_id"/>
    <result property="username" column="u_username"/>

    <!--
      collection 属性：
      property    - User 中集合属性的名称
      ofType      - 集合中元素的 Java 类型（注意：是 ofType，不是 javaType）
      columnPrefix - 列前缀（避免列名冲突）
    -->
    <collection property="orders"
                ofType="Order"
                columnPrefix="o_">
        <id property="id" column="id"/>
        <result property="userId" column="user_id"/>
        <result property="orderNo" column="order_no"/>
        <result property="amount" column="amount"/>
        <result property="status" column="status"/>
        <result property="createTime" column="create_time"/>
    </collection>
</resultMap>

<select id="selectUserWithOrders" resultMap="userWithOrdersMap">
    SELECT
        u.id        AS u_id,
        u.username  AS u_username,
        o.id        AS o_id,
        o.user_id   AS o_user_id,
        o.order_no  AS o_order_no,
        o.amount    AS o_amount,
        o.status    AS o_status,
        o.create_time AS o_create_time
    FROM user u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.id = #{id}
    ORDER BY o.create_time DESC
</select>
```

### 方式2：嵌套查询（支持懒加载）

```xml
<resultMap id="userWithOrdersLazyMap" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>

    <collection property="orders"
                column="id"
                ofType="Order"
                select="selectOrdersByUserId"
                fetchType="lazy"/>
</resultMap>

<select id="selectUserById" resultMap="userWithOrdersLazyMap">
    SELECT id, username, email FROM user WHERE id = #{id}
</select>

<select id="selectOrdersByUserId" resultType="Order">
    SELECT id, user_id, order_no, amount, status, create_time
    FROM orders WHERE user_id = #{userId}
    ORDER BY create_time DESC
</select>
```

---

## 3.9 discriminator（鉴别器）

鉴别器类似 Java 中的 switch 语句，根据某一列的值决定使用哪个 resultMap。

```java
// 基类
@Data
public class Vehicle {
    private Long id;
    private String vin;
    private String type;   // "car" | "bus" | "truck"
}

// 子类
@Data
public class Car extends Vehicle {
    private int doorCount;
}

@Data
public class Bus extends Vehicle {
    private int riderCapacity;
}

@Data
public class Truck extends Vehicle {
    private BigDecimal boxSize;
}
```

```xml
<resultMap id="vehicleResult" type="Vehicle">
    <id property="id" column="id"/>
    <result property="vin" column="vin"/>
    <result property="type" column="type"/>

    <!--
      discriminator 属性：
      javaType - 鉴别列的 Java 类型
      column   - 鉴别列的列名
    -->
    <discriminator javaType="String" column="type">
        <case value="car"   resultMap="carResultMap"/>
        <case value="bus"   resultMap="busResultMap"/>
        <case value="truck" resultMap="truckResultMap"/>
    </discriminator>
</resultMap>

<resultMap id="carResultMap" type="Car" extends="vehicleResult">
    <result property="doorCount" column="door_count"/>
</resultMap>

<resultMap id="busResultMap" type="Bus" extends="vehicleResult">
    <result property="riderCapacity" column="rider_capacity"/>
</resultMap>

<resultMap id="truckResultMap" type="Truck" extends="vehicleResult">
    <result property="boxSize" column="box_size"/>
</resultMap>

<select id="selectVehicle" resultMap="vehicleResult">
    SELECT v.id, v.vin, v.type,
           c.door_count,
           b.rider_capacity,
           t.box_size
    FROM vehicle v
    LEFT JOIN car c ON v.id = c.vehicle_id AND v.type = 'car'
    LEFT JOIN bus b ON v.id = b.vehicle_id AND v.type = 'bus'
    LEFT JOIN truck t ON v.id = t.vehicle_id AND v.type = 'truck'
    WHERE v.id = #{id}
</select>
```

---

## 3.10 #{} vs ${} 深度解析

这是 MyBatis 面试中出现频率极高的问题，也是实际开发中安全性的关键。

### #{} 预编译占位符

```
#{} 工作原理：

1. MyBatis 解析 XML，将 #{xxx} 替换为 JDBC 的 ? 占位符
2. 交给数据库驱动进行预编译
3. 通过 TypeHandler 将参数值安全地设置到 PreparedStatement
4. 数据库驱动负责处理特殊字符转义，彻底防止 SQL 注入

示例（使用 #{} 的流程）：
  Mapper XML: SELECT * FROM user WHERE name = #{name}
       |
       v（MyBatis处理）
  预编译SQL: SELECT * FROM user WHERE name = ?
       |
       v（设置参数）
  ps.setString(1, "张三")  <- 即使输入含特殊字符也安全

SQL 注入攻击无效：
  输入: name = "' OR '1'='1"
  实际: SELECT * FROM user WHERE name = ?
        参数值: ' OR '1'='1  <- 被当作普通字符串，不破坏SQL结构
```

### ${} 字符串替换（字符串插值）

```
${} 工作原理：

1. MyBatis 解析 XML，将 ${xxx} 直接替换为变量的字符串值
2. 拼接成最终的 SQL 字符串
3. 直接交给数据库执行（不经过预编译）
4. 存在 SQL 注入风险！必须谨慎使用！

示例（使用 ${} 的流程）：
  Mapper XML: SELECT * FROM ${tableName}
       |
       v（MyBatis直接替换）
  最终SQL: SELECT * FROM user_2024

SQL 注入攻击有效：
  输入: tableName = "user; DROP TABLE user; --"
  最终SQL: SELECT * FROM user; DROP TABLE user; --  <- 危险！
```

```xml
<!-- ${} 安全使用场景：动态表名 -->
<select id="selectFromTable" resultType="User">
    SELECT * FROM ${tableName}
</select>

<!-- ${} 安全使用场景：动态列名（ORDER BY）-->
<select id="selectOrderBy" resultType="User">
    SELECT * FROM user ORDER BY ${orderColumn} ${orderDir}
</select>

<!-- ${} 安全使用场景：动态 Schema -->
<select id="selectFromSchema" resultType="User">
    SELECT * FROM ${schema}.user WHERE id = #{id}
</select>
```

**${} 使用时的安全措施（白名单验证）：**

```java
@Service
public class UserService {

    private static final Set<String> ALLOWED_COLUMNS =
        Set.of("id", "username", "email", "create_time", "update_time");

    private static final Set<String> ALLOWED_DIRS = Set.of("ASC", "DESC");

    public List<User> getUsersSorted(String column, String dir) {
        // 白名单验证，防止 SQL 注入
        if (!ALLOWED_COLUMNS.contains(column)) {
            throw new IllegalArgumentException("非法排序字段: " + column);
        }
        if (!ALLOWED_DIRS.contains(dir.toUpperCase())) {
            throw new IllegalArgumentException("非法排序方向: " + dir);
        }

        return userMapper.selectOrderBy(column, dir.toUpperCase());
    }
}
```

**#{} 与 ${} 完整对比表：**

| 特性 | #{} | ${} |
|------|-----|-----|
| 类型 | 预编译占位符（? 替换）| 字符串直接替换 |
| SQL注入防护 | 完全防护（安全）| 无防护（危险）|
| 预编译 | 是（PreparedStatement）| 否（Statement）|
| 性能 | 相同SQL可复用预编译缓存 | 每次重新编译 |
| 类型处理 | TypeHandler 自动转换 | 直接字符串拼接 |
| null 处理 | 安全处理 null | null 字面量拼接为 "null" |
| 适用场景 | 参数值（数字、字符串等）| 列名、表名、ORDER BY 子句 |
| 引号 | 字符串值自动加引号 | 不加引号（原样插入）|
| 推荐 | 大多数场景（强烈推荐）| 仅在必要时（配合白名单）|

---

# Part 4: 动态 SQL 深度解析 <a id="part-4"></a>

动态 SQL 是 MyBatis 的强大特性之一，可以根据不同的条件动态生成 SQL 语句，避免了手动拼接 SQL 字符串的繁琐和错误。MyBatis 的动态 SQL 基于 OGNL 表达式。

## 4.1 if 标签

`if` 是最基础的动态 SQL 元素，用于条件判断。

```xml
<select id="selectByCondition" resultType="User">
    SELECT * FROM user
    WHERE 1=1
    <if test="username != null and username != ''">
        AND username LIKE CONCAT('%', #{username}, '%')
    </if>
    <if test="email != null and email != ''">
        AND email = #{email}
    </if>
    <if test="status != null">
        AND status = #{status}
    </if>
    <if test="minAge != null">
        AND age >= #{minAge}
    </if>
    <if test="maxAge != null">
        AND age &lt;= #{maxAge}
    </if>
    <if test="createTimeStart != null">
        AND create_time >= #{createTimeStart}
    </if>
    <if test="createTimeEnd != null">
        AND create_time &lt;= #{createTimeEnd}
    </if>
    AND deleted = 0
    ORDER BY create_time DESC
</select>
```

**if 标签 test 属性的常用写法大全：**

```xml
<!-- 判断 null -->
<if test="name != null">

<!-- 判断 null 且不为空字符串 -->
<if test="name != null and name != ''">

<!-- 判断集合不为空 -->
<if test="list != null and list.size() > 0">

<!-- 等价写法 -->
<if test="list != null and !list.isEmpty()">

<!-- 判断枚举值（使用 @ 符号引用枚举常量）-->
<if test="status == @com.example.enums.UserStatus@ACTIVE">

<!-- 判断布尔值 -->
<if test="deleted == true">
<if test="deleted">
<if test="!deleted">

<!-- 数值比较（注意：< 需要转义为 &lt;，> 也建议转义为 &gt;）-->
<if test="age &gt;= 18">
<if test="age &lt;= 60">
<if test="age &gt; 0 and age &lt; 100">

<!-- 字符串比较（OGNL 中字符串比较用 == 或 .equals()）-->
<if test="type == 'ADMIN'">
<if test='type == "ADMIN"'>
<if test="type.equals('ADMIN')">

<!-- 字符串不等 -->
<if test="type != 'GUEST'">

<!-- 使用 or 和 and -->
<if test="(age &gt;= 18 and age &lt;= 60) or type == 'VIP'">

<!-- 判断数组不为空 -->
<if test="ids != null and ids.length > 0">
```

---

## 4.2 choose/when/otherwise（多分支选择）

`choose` 类似 Java 的 switch 语句，只会执行第一个满足条件的 `when`，都不满足则执行 `otherwise`。

```xml
<!-- 场景：根据传入的不同查询条件，优先级选择查询方式 -->
<select id="selectByPriority" resultType="User">
    SELECT * FROM user
    WHERE
    <choose>
        <!-- 优先使用 id 精确查询 -->
        <when test="id != null">
            id = #{id}
        </when>
        <!-- 其次使用 username 精确查询 -->
        <when test="username != null and username != ''">
            username = #{username}
        </when>
        <!-- 再次使用 email 查询 -->
        <when test="email != null and email != ''">
            email = #{email}
        </when>
        <!-- 都没有传入时，查询所有启用的用户 -->
        <otherwise>
            status = 1
        </otherwise>
    </choose>
</select>
```

```java
// 对应的 Mapper 接口
@Mapper
public interface UserMapper {
    List<User> selectByPriority(
        @Param("id") Long id,
        @Param("username") String username,
        @Param("email") String email
    );
}

// 调用示例
// 情况1：只传 id，生成 WHERE id = 1
List<User> result1 = userMapper.selectByPriority(1L, null, null);

// 情况2：只传 username，生成 WHERE username = 'admin'
List<User> result2 = userMapper.selectByPriority(null, "admin", null);

// 情况3：都不传，生成 WHERE status = 1
List<User> result3 = userMapper.selectByPriority(null, null, null);
```

---

## 4.3 where/set/trim 标签

### where 标签（处理多余 AND/OR）

`where` 标签会在至少有一个子元素返回内容时，自动插入 `WHERE`；并且会自动去除内容**开头**多余的 `AND` 或 `OR`。

```xml
<!-- 不使用 where 标签（有 BUG 风险）-->
<select id="selectBad" resultType="User">
    SELECT * FROM user
    WHERE
    <if test="username != null">
        username = #{username}
    </if>
    <if test="email != null">
        AND email = #{email}
    </if>
    <!--
    BUG场景：当 username=null，email="a@b.com" 时
    生成: WHERE AND email = ?  <- SQL 语法错误！
    -->
</select>

<!-- 使用 where 标签（正确方式）-->
<select id="selectGood" resultType="User">
    SELECT * FROM user
    <where>
        <if test="username != null and username != ''">
            AND username = #{username}
        </if>
        <if test="email != null and email != ''">
            AND email = #{email}
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
        <if test="phone != null">
            AND phone = #{phone}
        </if>
    </where>
    <!--
    场景1: username="张三", email=null
    生成: WHERE username = '张三'  <- where 标签自动去除了开头的 AND

    场景2: username=null, email="a@b.com"
    生成: WHERE email = 'a@b.com'  <- 正确！

    场景3: 所有条件都为 null
    生成: SELECT * FROM user  <- where 标签不插入 WHERE 关键字
    -->
</select>
```

**where 标签的底层原理：**
```
where 标签等价于：
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>
```

### set 标签（处理多余逗号）

`set` 标签用于动态更新，会自动插入 `SET` 关键字，并去除**末尾**多余的逗号。

```xml
<!-- 不使用 set 标签（有 BUG 风险）-->
<update id="updateBad" parameterType="User">
    UPDATE user SET
    <if test="username != null">username = #{username},</if>
    <if test="email != null">email = #{email},</if>
    WHERE id = #{id}
    <!--
    BUG场景：只更新 username 时
    生成: UPDATE user SET username = '张三', WHERE id = 1  <- 多余逗号！
    -->
</update>

<!-- 使用 set 标签（正确方式）-->
<update id="updateSelective" parameterType="User">
    UPDATE user
    <set>
        <if test="username != null and username != ''">
            username = #{username},
        </if>
        <if test="email != null and email != ''">
            email = #{email},
        </if>
        <if test="phone != null">
            phone = #{phone},
        </if>
        <if test="status != null">
            status = #{status},
        </if>
        update_time = NOW(),
    </set>
    WHERE id = #{id}
    <!--
    场景: 只传入 username
    生成: UPDATE user SET username = '张三', update_time = NOW() WHERE id = 1
    set 标签自动去除了最后的逗号，并加入 SET 关键字
    -->
</update>
```

**set 标签的底层原理：**
```
set 标签等价于：
<trim prefix="SET" suffixOverrides=",">
  ...
</trim>
```

### trim 标签（万能标签，where 和 set 的底层实现）

`trim` 可以自定义前缀/后缀的处理，是最灵活的动态 SQL 标签。

```xml
<!--
  trim 属性详解：
  prefix         - 当 trim 内容非空时，在最前面插入的内容
  prefixOverrides - 去除内容开头匹配的字符串（| 分隔多个候选项，不区分大小写）
  suffix         - 当 trim 内容非空时，在最后面插入的内容
  suffixOverrides - 去除内容末尾匹配的字符串
-->

<!-- 等价于 <where> 标签 -->
<trim prefix="WHERE" prefixOverrides="AND |OR ">
    <if test="username != null">AND username = #{username}</if>
    <if test="email != null">AND email = #{email}</if>
</trim>

<!-- 等价于 <set> 标签 -->
<trim prefix="SET" suffixOverrides=",">
    <if test="username != null">username = #{username},</if>
    <if test="email != null">email = #{email},</if>
</trim>

<!-- 自定义：生成 INSERT 语句的字段列表和值列表 -->
<insert id="insertTrim" parameterType="User">
    INSERT INTO user
    <trim prefix="(" suffix=")" suffixOverrides=",">
        <if test="username != null">username,</if>
        <if test="email != null">email,</if>
        <if test="phone != null">phone,</if>
        <if test="status != null">status,</if>
        create_time,
    </trim>
    VALUES
    <trim prefix="(" suffix=")" suffixOverrides=",">
        <if test="username != null">#{username},</if>
        <if test="email != null">#{email},</if>
        <if test="phone != null">#{phone},</if>
        <if test="status != null">#{status},</if>
        NOW(),
    </trim>
</insert>
```

---

## 4.4 foreach（遍历集合）

`foreach` 用于遍历集合，常用于 IN 查询和批量操作。

```xml
<!--
  foreach 属性详解：
  collection  - 集合的变量名（list/array/map 或 @Param 指定的名称）
  item        - 集合中每个元素的迭代变量名
  index       - 元素的索引（List: 下标 0,1,2...; Map: key）
  open        - 整个 foreach 内容的开始字符串
  close       - 整个 foreach 内容的结束字符串
  separator   - 每两个元素之间的分隔符
-->

<!-- 基础 IN 查询（参数直接用 list）-->
<select id="selectByIds" resultType="User">
    SELECT * FROM user
    WHERE id IN
    <foreach collection="list" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>

<!-- IN 查询（@Param 命名集合，更推荐）-->
<select id="selectByIdList" resultType="User">
    SELECT * FROM user
    WHERE id IN
    <foreach collection="ids" item="id" open="(" separator="," close=")">
        #{id}
    </foreach>
</select>
```

```java
// Mapper 接口
@Mapper
public interface UserMapper {
    List<User> selectByIds(List<Long> list);  // 参数名直接用 "list"

    List<User> selectByIdList(@Param("ids") List<Long> ids);  // @Param 命名
}
```

**批量插入（MySQL 多 values 语法，一条 SQL 高效插入）：**

```xml
<insert id="insertBatch" parameterType="list">
    INSERT INTO user (username, email, status, create_time) VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.username}, #{user.email}, #{user.status}, NOW())
    </foreach>
</insert>
```

```java
// 使用示例
@Test
public void testInsertBatch() {
    List<User> users = new ArrayList<>();
    for (int i = 1; i <= 1000; i++) {
        User user = new User();
        user.setUsername("user" + i);
        user.setEmail("user" + i + "@example.com");
        user.setStatus(1);
        users.add(user);
    }
    // 生成一条 INSERT 语句，包含 1000 个 VALUES
    // INSERT INTO user (...) VALUES (...), (...), ...
    int count = userMapper.insertBatch(users);
    System.out.println("插入数量: " + count);  // 1000
}
```

**批量更新（MySQL，需要 allowMultiQueries=true）：**

```xml
<update id="updateBatch">
    <foreach collection="list" item="user" separator=";">
        UPDATE user
        <set>
            <if test="user.username != null">username = #{user.username},</if>
            <if test="user.email != null">email = #{user.email},</if>
            update_time = NOW(),
        </set>
        WHERE id = #{user.id}
    </foreach>
</update>
```

**遍历 Map：**

```xml
<!-- 遍历 Map，index 是 key，item 是 value -->
<select id="selectByMapCondition" resultType="User">
    SELECT * FROM user
    <where>
        <foreach collection="conditionMap" index="column" item="value" separator="AND">
            ${column} = #{value}
        </foreach>
    </where>
</select>
```

**遍历对象列表（index 可用）：**

```xml
<insert id="insertWithIndex">
    INSERT INTO user_batch_log (batch_no, seq_no, user_id, username)
    VALUES
    <foreach collection="list" item="user" index="i" separator=",">
        (#{batchNo}, #{i}, #{user.id}, #{user.username})
    </foreach>
</insert>
```

**数组参数：**

```xml
<!-- 参数为数组时，collection 写 "array" -->
<select id="selectByStatusArray" resultType="User">
    SELECT * FROM user
    WHERE status IN
    <foreach collection="array" item="status" open="(" separator="," close=")">
        #{status}
    </foreach>
</select>
```

```java
List<User> selectByStatusArray(int[] statusArray);
```

---

## 4.5 bind（变量绑定）

`bind` 元素可以从 OGNL 表达式创建一个变量，并绑定到上下文中。

**常用场景1：模糊查询（避免数据库方言差异）**

```xml
<!-- 方式1：MySQL CONCAT（数据库相关）-->
<select id="selectByNameV1" resultType="User">
    SELECT * FROM user WHERE username LIKE CONCAT('%', #{name}, '%')
</select>

<!-- 方式2：使用 bind（数据库无关）-->
<select id="selectByNameV2" resultType="User">
    <bind name="nameLike" value="'%' + name + '%'"/>
    SELECT * FROM user WHERE username LIKE #{nameLike}
</select>

<!-- 复合 bind 使用 -->
<select id="searchUsers" resultType="User">
    <bind name="usernameLike" value="'%' + (username != null ? username : '') + '%'"/>
    <bind name="emailPrefix" value="(email != null ? email : '') + '%'"/>
    SELECT * FROM user
    <where>
        <if test="username != null and username != ''">
            AND username LIKE #{usernameLike}
        </if>
        <if test="email != null and email != ''">
            AND email LIKE #{emailPrefix}
        </if>
    </where>
</select>
```

**常用场景2：计算偏移量**

```xml
<select id="selectPage" resultType="User">
    <bind name="offset" value="(pageNum - 1) * pageSize"/>
    SELECT * FROM user
    WHERE status = 1
    ORDER BY create_time DESC
    LIMIT #{pageSize} OFFSET #{offset}
</select>
```

---

## 4.6 sql/include（SQL 片段复用）

`sql` 和 `include` 用于提取公共的 SQL 片段，避免重复代码，提高可维护性。

```xml
<!-- 定义 SQL 片段 -->
<sql id="userColumns">
    u.id, u.username, u.email, u.phone, u.status,
    u.create_time, u.update_time
</sql>

<!-- 定义带表别名的 SQL 片段 -->
<sql id="userJoinProfile">
    user u
    LEFT JOIN user_profile p ON u.id = p.user_id
</sql>

<!-- 定义带参数的 SQL 片段（用 ${property} 接收参数）-->
<sql id="orderByClause">
    <if test="_parameter != null and _parameter.orderBy != null">
        ORDER BY ${orderBy}
        <if test="_parameter.orderDir != null">
            ${orderDir}
        </if>
    </if>
</sql>

<!-- 带属性传入的动态片段 -->
<sql id="limitClause">
    LIMIT ${pageSize} OFFSET ${offset}
</sql>

<!-- 引用 SQL 片段 -->
<select id="selectUserList" resultType="User">
    SELECT
    <include refid="userColumns"/>
    FROM user u
    WHERE u.status = 1
</select>

<!-- 引用并传属性 -->
<select id="selectUserPage" resultType="User">
    SELECT
    <include refid="userColumns"/>
    FROM
    <include refid="userJoinProfile"/>
    WHERE u.status = 1
    <include refid="orderByClause">
        <property name="orderBy" value="u.create_time"/>
        <property name="orderDir" value="DESC"/>
    </include>
</select>

<!-- 跨 namespace 引用（使用全限定 id）-->
<include refid="com.example.mapper.BaseMapper.commonColumns"/>
```

---

## 4.7 完整案例：通用查询条件构建器

这是实际项目中最常用的动态 SQL 场景，实现一个通用的多条件查询+分页功能。

```java
// 通用分页查询请求 DTO
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserQueryRequest {

    // 基础筛选条件
    private Long id;                          // 精确 ID
    private List<Long> ids;                   // 批量 ID
    private String username;                  // 模糊匹配
    private String email;                     // 精确匹配
    private String phone;                     // 精确匹配
    private Integer status;                   // 状态

    // 范围查询
    private Integer minAge;
    private Integer maxAge;
    private LocalDateTime createTimeStart;
    private LocalDateTime createTimeEnd;

    // 排序
    private String orderBy;   // 排序字段（前端传入，需白名单验证）
    private String orderDir;  // ASC 或 DESC

    // 分页
    private Integer pageNum  = 1;    // 当前页码，默认第1页
    private Integer pageSize = 10;   // 每页大小，默认10条
}
```

```xml
<!-- UserMapper.xml -->

<!-- 通用查询条件片段（复用于多个查询）-->
<sql id="userQueryConditions">
    <where>
        <if test="id != null">
            AND u.id = #{id}
        </if>
        <if test="ids != null and ids.size() > 0">
            AND u.id IN
            <foreach collection="ids" item="id" open="(" separator="," close=")">
                #{id}
            </foreach>
        </if>
        <if test="username != null and username != ''">
            <bind name="usernameLike" value="'%' + username + '%'"/>
            AND u.username LIKE #{usernameLike}
        </if>
        <if test="email != null and email != ''">
            AND u.email = #{email}
        </if>
        <if test="phone != null and phone != ''">
            AND u.phone = #{phone}
        </if>
        <if test="status != null">
            AND u.status = #{status}
        </if>
        <if test="minAge != null">
            AND u.age >= #{minAge}
        </if>
        <if test="maxAge != null">
            AND u.age &lt;= #{maxAge}
        </if>
        <if test="createTimeStart != null">
            AND u.create_time >= #{createTimeStart}
        </if>
        <if test="createTimeEnd != null">
            AND u.create_time &lt;= #{createTimeEnd}
        </if>
        AND u.deleted = 0
    </where>
</sql>

<!-- 查询总数（用于分页）-->
<select id="countByCondition" resultType="long">
    SELECT COUNT(*)
    FROM user u
    <include refid="userQueryConditions"/>
</select>

<!-- 分页查询 -->
<select id="pageByCondition" resultType="User">
    SELECT
        u.id, u.username, u.email, u.phone,
        u.status, u.age, u.create_time, u.update_time
    FROM user u
    <include refid="userQueryConditions"/>
    <if test="orderBy != null and orderBy != ''">
        ORDER BY ${orderBy}
        <choose>
            <when test="orderDir != null and orderDir.toUpperCase() == 'ASC'">
                ASC
            </when>
            <otherwise>
                DESC
            </otherwise>
        </choose>
    </if>
    <if test="orderBy == null or orderBy == ''">
        ORDER BY u.create_time DESC
    </if>
    LIMIT #{pageSize}
    <bind name="offset" value="(pageNum - 1) * pageSize"/>
    OFFSET #{offset}
</select>

<!-- 导出查询（不分页）-->
<select id="exportByCondition" resultType="User">
    SELECT
        u.id, u.username, u.email, u.phone,
        u.status, u.age, u.create_time
    FROM user u
    <include refid="userQueryConditions"/>
    ORDER BY u.create_time DESC
</select>
```

```java
// Mapper 接口
@Mapper
public interface UserMapper {
    long countByCondition(UserQueryRequest query);
    List<User> pageByCondition(UserQueryRequest query);
    List<User> exportByCondition(UserQueryRequest query);
}

// Service 层
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public PageResult<User> queryUsers(UserQueryRequest request) {
        // 查询总数
        long total = userMapper.countByCondition(request);
        if (total == 0) {
            return PageResult.empty();
        }
        // 查询数据
        List<User> list = userMapper.pageByCondition(request);
        return new PageResult<>(total, list, request.getPageNum(), request.getPageSize());
    }
}

// 调用示例
UserQueryRequest request = UserQueryRequest.builder()
    .username("张")          // 模糊搜索
    .status(1)               // 状态为启用
    .minAge(18)              // 年龄 >= 18
    .maxAge(60)              // 年龄 <= 60
    .createTimeStart(LocalDateTime.of(2024, 1, 1, 0, 0))
    .ids(Arrays.asList(1L, 2L, 3L))  // 在这几个 ID 范围内
    .orderBy("create_time")
    .orderDir("DESC")
    .pageNum(1)
    .pageSize(20)
    .build();

PageResult<User> result = userService.queryUsers(request);
System.out.println("总数: " + result.getTotal());
System.out.println("数据: " + result.getList().size() + " 条");
```

---

# Part 5: MyBatis 缓存机制 <a id="part-5"></a>

## 5.1 缓存整体架构

```
+------------------------------------------------------------------------+
|                        MyBatis 缓存体系                                  |
|                                                                          |
|  查询请求                                                                 |
|      |                                                                   |
|      v                                                                   |
|  +------------------------------------------------------------------+   |
|  |              二级缓存（Mapper 级别 / namespace 级别）              |   |
|  |                                                                  |   |
|  |  特点：跨 SqlSession 共享，同一 Mapper namespace 内的所有会话共享   |   |
|  |  默认实现：PerpetualCache（HashMap）                              |   |
|  |  扩展实现：Ehcache、Redis 等第三方缓存                             |   |
|  |                                                                  |   |
|  |  [命中] -> 返回缓存对象（反序列化副本）                             |   |
|  |  [未命中] -> 向下查询一级缓存                                      |   |
|  +------------------------------------------------------------------+   |
|      | 未命中                                                            |
|      v                                                                   |
|  +------------------------------------------------------------------+   |
|  |              一级缓存（SqlSession 级别 / 本地缓存）                |   |
|  |                                                                  |   |
|  |  特点：SqlSession 内私有，非线程安全，生命周期随 SqlSession         |   |
|  |  实现：BaseExecutor.localCache（PerpetualCache）                  |   |
|  |  key: statementId + RowBounds + SQL + 参数值                     |   |
|  |                                                                  |   |
|  |  [命中] -> 返回缓存对象（同一引用！）                               |   |
|  |  [未命中] -> 查询数据库                                            |   |
|  +------------------------------------------------------------------+   |
|      | 未命中                                                            |
|      v                                                                   |
|  +------------------------------------------------------------------+   |
|  |                           数据库                                  |   |
|  |  执行 SQL -> 获取 ResultSet -> 映射为 Java 对象                   |   |
|  |  -> 写入一级缓存 -> 写入二级缓存（SqlSession.close() 时）          |   |
|  +------------------------------------------------------------------+   |
|                                                                          |
+------------------------------------------------------------------------+

缓存查询顺序：二级缓存 -> 一级缓存 -> 数据库
```

---

## 5.2 一级缓存（SqlSession 级别）

### 一级缓存的实现原理

一级缓存是 MyBatis 的**默认缓存**，存在于 `BaseExecutor` 中，本质是一个 `HashMap`（通过 `PerpetualCache` 封装）。**一级缓存无法关闭**，但可以通过设置 `localCacheScope=STATEMENT` 来使其失效（每次查询后立即清除）。

```java
// PerpetualCache 的核心实现
public class PerpetualCache implements Cache {

    private final String id;

    // 底层就是一个 HashMap！
    private final Map<Object, Object> cache = new HashMap<>();

    @Override
    public void putObject(Object key, Object value) {
        cache.put(key, value);
    }

    @Override
    public Object getObject(Object key) {
        return cache.get(key);
    }

    @Override
    public Object removeObject(Object key) {
        return cache.remove(key);
    }

    @Override
    public void clear() {
        cache.clear();
    }

    @Override
    public int getSize() {
        return cache.size();
    }
}
```

### CacheKey 的构成（决定是否命中缓存）

```
CacheKey 由以下 5 个要素组成（全部相同才能命中）：

1. MappedStatement.id         -> "com.example.mapper.UserMapper.selectById"
2. RowBounds.offset           -> 0（默认）
3. RowBounds.limit            -> Integer.MAX_VALUE（默认）
4. BoundSql.getSql()          -> "SELECT * FROM user WHERE id = ?"
5. 参数值（依次 append）       -> [1L]（id=1）
6. Environment.id             -> "development"（多数据源时区分）

任何一个要素不同，都会产生不同的 CacheKey，无法命中缓存。
```

### 一级缓存失效的 5 种场景

```java
@Test
public void testLevel1Cache() throws Exception {
    try (SqlSession session = sqlSessionFactory.openSession()) {
        UserMapper mapper = session.getMapper(UserMapper.class);

        // 第一次查询：查数据库，结果存入一级缓存
        User user1 = mapper.selectById(1L);
        System.out.println("第一次查询: " + user1.getUsername());

        // 第二次查询相同 key：命中一级缓存，不查数据库！
        User user2 = mapper.selectById(1L);
        System.out.println("是否同一对象: " + (user1 == user2));  // true
    }
}

// ================== 失效场景1：执行了增删改操作 ==================
@Test
public void testInvalidate_DML() {
    try (SqlSession session = sqlSessionFactory.openSession()) {
        UserMapper mapper = session.getMapper(UserMapper.class);

        User user1 = mapper.selectById(1L);  // 查DB，存入缓存

        // 执行任意增删改操作（即使操作的是其他记录！）
        // 都会导致整个 localCache 被清空
        mapper.updateById(User.builder().id(99L).username("other").build());

        User user2 = mapper.selectById(1L);  // 缓存已清，重新查DB

        System.out.println(user1 == user2);  // false
    }
}

// ================== 失效场景2：使用了不同的 SqlSession ==================
@Test
public void testInvalidate_DifferentSession() {
    // SqlSession1 查询后存入缓存
    try (SqlSession session1 = sqlSessionFactory.openSession()) {
        User user1 = session1.getMapper(UserMapper.class).selectById(1L);
        // session1 的缓存中有 id=1 的用户

        // SqlSession2 完全独立，不共享 session1 的缓存
        try (SqlSession session2 = sqlSessionFactory.openSession()) {
            User user2 = session2.getMapper(UserMapper.class).selectById(1L);
            System.out.println(user1 == user2);  // false，两个 SqlSession 互不共享
        }
    }
}

// ================== 失效场景3：手动清除缓存 ==================
@Test
public void testInvalidate_ManualClear() {
    try (SqlSession session = sqlSessionFactory.openSession()) {
        UserMapper mapper = session.getMapper(UserMapper.class);

        User user1 = mapper.selectById(1L);  // 查DB，存入缓存

        // 手动清除一级缓存
        session.clearCache();

        User user2 = mapper.selectById(1L);  // 缓存已清，重新查DB

        System.out.println(user1 == user2);  // false
    }
}

// ================== 失效场景4：XML 配置了 flushCache="true" ==================
// <select id="selectById" flushCache="true" resultType="User">
//     SELECT * FROM user WHERE id = #{id}
// </select>
// 每次调用该方法前都会清除一级缓存

// ================== 失效场景5：设置了 localCacheScope=STATEMENT ==================
// 每次查询执行完毕后立即清除一级缓存，相当于禁用一级缓存
```

### 一级缓存在 Spring 整合中失效的原因（高频面试题）

```
问题：为什么在 Spring 中，对同一 Mapper 的相同查询会访问两次数据库？
       即：一级缓存在 Spring 中为什么失效了？

根本原因：
  Spring 通过 SqlSessionTemplate 代理 SqlSession
  SqlSessionTemplate 在每次方法调用时：
  1. 从 TransactionSynchronizationManager（事务同步管理器）查找
     当前线程绑定的 SqlSession
  2. 如果当前方法没有 @Transactional 注解（或没有事务传播）：
     -> 每次创建一个新的 SqlSession
     -> 执行 SQL
     -> 立即关闭这个 SqlSession（close/commit）
     -> 下次调用时，又是新的 SqlSession
     -> 一级缓存形同虚设！

图示：
  无事务情况：
  Mapper.selectById(1L) -> 新建 SqlSession1 -> 查DB -> 关闭Session1
  Mapper.selectById(1L) -> 新建 SqlSession2 -> 查DB -> 关闭Session2
  （两次都查了数据库！）

  有 @Transactional 情况：
  @Transactional 开始 -> 创建 SqlSession，绑定到当前线程
  Mapper.selectById(1L) -> 复用线程绑定的 SqlSession -> 查DB -> 存入缓存
  Mapper.selectById(1L) -> 复用同一 SqlSession -> 命中缓存！
  @Transactional 结束 -> 提交事务，关闭 SqlSession
```

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    // 无事务：一级缓存失效，两次查询都访问数据库
    public void noTransaction() {
        User user1 = userMapper.selectById(1L);  // 查数据库
        User user2 = userMapper.selectById(1L);  // 还是查数据库！
        System.out.println(user1 == user2);      // false（不同对象！）
    }

    // 有事务：一级缓存生效，第二次命中缓存
    @Transactional(readOnly = true)
    public void withTransaction() {
        User user1 = userMapper.selectById(1L);  // 查数据库，存入缓存
        User user2 = userMapper.selectById(1L);  // 命中一级缓存！
        System.out.println(user1 == user2);      // true（同一对象！）
    }
}
```

---

## 5.3 二级缓存（Mapper 级别）

### 开启二级缓存

**步骤1：** 全局开关（`mybatis-config.xml` 或 `application.yml`，默认已开启）：

```xml
<settings>
    <setting name="cacheEnabled" value="true"/>  <!-- 默认就是 true -->
</settings>
```

**步骤2：** 在 Mapper XML 中添加 `<cache/>` 标签：

```xml
<mapper namespace="com.example.mapper.UserMapper">

    <!-- 最简单的开启方式，使用默认配置 -->
    <cache/>

    <!-- 详细配置版本 -->
    <cache
        eviction="LRU"          <!-- 回收策略: LRU(默认)|FIFO|SOFT|WEAK -->
        flushInterval="60000"   <!-- 60秒刷新一次（毫秒），默认不刷新 -->
        size="1024"             <!-- 最多缓存 1024 个对象，默认1024 -->
        readOnly="false"        <!-- false时通过序列化复制（安全，默认）; true直接引用（不安全但快）-->
        type=""                 <!-- 自定义 Cache 实现类（如 Redis 缓存）-->
        blocking="false"        <!-- 防止缓存穿透（key未命中时阻塞其他线程）-->
    />

    <!--
      eviction（回收策略）详解：
      LRU  - 最近最少使用（默认，推荐）：移除最长时间不被访问的对象
      FIFO - 先进先出：按进入缓存的顺序移除
      SOFT - 软引用：移除基于垃圾回收器状态和软引用规则的对象
      WEAK - 弱引用：更积极地移除基于垃圾回收器状态和弱引用规则的对象
    -->

    <!-- 单个语句级别控制：关闭二级缓存 -->
    <select id="selectRealtime" useCache="false" resultType="User">
        SELECT * FROM user WHERE id = #{id}
    </select>

    <!-- 插入/更新/删除默认都会清空二级缓存（flushCache=true）-->
    <insert id="insert" parameterType="User">
        INSERT INTO user (username) VALUES (#{username})
    </insert>

    <!-- 如果不想让查询操作清空缓存 -->
    <select id="selectNonFlush" flushCache="false" resultType="User">
        SELECT * FROM user
    </select>

</mapper>
```

**步骤3：** 实体类实现 `Serializable`（因为二级缓存默认通过序列化存储）：

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User implements Serializable {

    // 强烈建议指定 serialVersionUID，避免序列化版本不一致问题
    private static final long serialVersionUID = 1L;

    private Long id;
    private String username;
    private String email;
    private Integer status;
    private LocalDateTime createTime;
    private LocalDateTime updateTime;
}
```

### 二级缓存工作原理

```
二级缓存数据流：

第1个 SqlSession 查询：
  SqlSession1 -> 查询 -> DB（未命中一级缓存）
  -> 结果暂存在 Session 的 TransactionalCacheManager 中
  -> SqlSession1.close() 时，提交到二级缓存

第2个 SqlSession 查询（相同 key）：
  SqlSession2 -> 查询 -> 命中二级缓存！
  -> 通过反序列化返回缓存对象的副本
  -> （注意：返回的是副本，不是原始对象）

关键点：
- 二级缓存在 SqlSession.close() 或 commit() 时写入
- 这是为了防止未提交的事务污染缓存
- 读取时通过反序列化返回副本（readOnly=false时）
```

```java
@Test
public void testLevel2Cache() {
    // 第一个 SqlSession
    SqlSession session1 = sqlSessionFactory.openSession(true);
    UserMapper mapper1 = session1.getMapper(UserMapper.class);

    User user1 = mapper1.selectById(1L);  // 查数据库
    System.out.println("Cache Hit Ratio: " + getCacheHitRatio());  // 0.0（未命中）

    session1.close();  // 重要：close 时才写入二级缓存！

    // 第二个 SqlSession
    SqlSession session2 = sqlSessionFactory.openSession(true);
    UserMapper mapper2 = session2.getMapper(UserMapper.class);

    User user2 = mapper2.selectById(1L);  // 命中二级缓存！
    System.out.println("Cache Hit Ratio: " + getCacheHitRatio());  // 0.5（命中一次）

    session2.close();

    // 二级缓存返回的是反序列化后的副本，不是原始对象
    System.out.println(user1 == user2);              // false（不同对象）
    System.out.println(user1.getId().equals(user2.getId()));  // true（内容相同）
}
```

### 二级缓存的脏读问题（重要！）

```
WARNING：二级缓存在多表关联查询中容易产生脏读！

场景复现：
  1. UserMapper 中有一个查询：SELECT u.*, d.dept_name FROM user u
     JOIN department d ON u.dept_id = d.id WHERE u.id = 1
     查询结果被缓存到 UserMapper 的二级缓存中

  2. 另一个操作更新了 department 表（通过 DepartmentMapper）：
     UPDATE department SET dept_name = '新部门名' WHERE id = 1

  3. DepartmentMapper 的更新会清空 DepartmentMapper 的二级缓存
     但不会清空 UserMapper 的二级缓存！

  4. 再次查询 User（通过 UserMapper），命中了二级缓存
     返回的 dept_name 仍然是旧值 -> 脏读！

解决方案：

方案1：对于关联多表的查询，关闭二级缓存
<select id="selectWithDept" useCache="false" resultType="UserVO">
    SELECT u.*, d.dept_name FROM user u
    JOIN department d ON u.dept_id = d.id
</select>

方案2：使用 cache-ref 让多个 Mapper 共享一个缓存
<!-- 在 UserMapper.xml 中 -->
<cache-ref namespace="com.example.mapper.DepartmentMapper"/>
这样 department 更新时，User 的缓存也会被清空

方案3（最推荐）：关闭 MyBatis 二级缓存，使用 Spring Cache + Redis
# application.yml
mybatis:
  configuration:
    cache-enabled: false  # 关闭 MyBatis 二级缓存

# 在 Service 层使用 Spring Cache
@Service
public class UserService {
    @Cacheable(value = "user", key = "#id")
    public User selectById(Long id) {
        return userMapper.selectById(id);
    }

    @CacheEvict(value = "user", key = "#user.id")
    public void update(User user) {
        userMapper.updateById(user);
    }
}
```

---

## 5.4 自定义缓存（集成 Redis）

实现 `org.apache.ibatis.cache.Cache` 接口，将缓存数据存储到 Redis 中：

```java
package com.example.cache;

import org.apache.ibatis.cache.Cache;
import org.springframework.data.redis.core.RedisTemplate;

import java.util.Set;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReadWriteLock;
import java.util.concurrent.locks.ReentrantReadWriteLock;

/**
 * MyBatis 自定义 Redis 缓存实现
 * 将 MyBatis 的二级缓存存储到 Redis 中，支持集群共享
 */
public class MybatisRedisCache implements Cache {

    private final String id;
    private final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();
    private static final long EXPIRE_SECONDS = 3600L;
    private static final String KEY_PREFIX = "mybatis:l2cache:";

    public MybatisRedisCache(String id) {
        if (id == null) {
            throw new IllegalArgumentException("Cache id cannot be null");
        }
        this.id = id;
    }

    @Override
    public String getId() {
        return id;
    }

    @Override
    public void putObject(Object key, Object value) {
        try {
            RedisTemplate<String, Object> template = getRedisTemplate();
            template.opsForValue().set(buildKey(key), value, EXPIRE_SECONDS, TimeUnit.SECONDS);
        } catch (Exception e) {
            // Redis 异常不影响主业务
            e.printStackTrace();
        }
    }

    @Override
    public Object getObject(Object key) {
        try {
            RedisTemplate<String, Object> template = getRedisTemplate();
            return template.opsForValue().get(buildKey(key));
        } catch (Exception e) {
            e.printStackTrace();
            return null;
        }
    }

    @Override
    public Object removeObject(Object key) {
        try {
            RedisTemplate<String, Object> template = getRedisTemplate();
            template.delete(buildKey(key));
        } catch (Exception e) {
            e.printStackTrace();
        }
        return null;
    }

    @Override
    public void clear() {
        try {
            RedisTemplate<String, Object> template = getRedisTemplate();
            // 使用 SCAN 遍历删除该 namespace 下的所有 key
            String pattern = KEY_PREFIX + id + ":*";
            Set<String> keys = template.keys(pattern);
            if (keys != null && !keys.isEmpty()) {
                template.delete(keys);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    @Override
    public int getSize() {
        try {
            RedisTemplate<String, Object> template = getRedisTemplate();
            Set<String> keys = template.keys(KEY_PREFIX + id + ":*");
            return keys == null ? 0 : keys.size();
        } catch (Exception e) {
            return 0;
        }
    }

    @Override
    public ReadWriteLock getReadWriteLock() {
        return readWriteLock;
    }

    private String buildKey(Object key) {
        return KEY_PREFIX + id + ":" + key.toString();
    }

    @SuppressWarnings("unchecked")
    private RedisTemplate<String, Object> getRedisTemplate() {
        // 通过 Spring 工具类获取 Bean（因为 Cache 对象不是 Spring 管理的）
        return (RedisTemplate<String, Object>)
            SpringContextHolder.getBean("redisTemplate");
    }
}
```

**Spring 上下文持有者工具类：**

```java
package com.example.util;

import org.springframework.beans.BeansException;
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.stereotype.Component;

@Component
public class SpringContextHolder implements ApplicationContextAware {

    private static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        SpringContextHolder.applicationContext = context;
    }

    public static <T> T getBean(String beanName) {
        return (T) applicationContext.getBean(beanName);
    }

    public static <T> T getBean(Class<T> clazz) {
        return applicationContext.getBean(clazz);
    }
}
```

**在 Mapper XML 中配置使用 Redis 缓存：**

```xml
<mapper namespace="com.example.mapper.UserMapper">
    <!-- 使用自定义 Redis 缓存 -->
    <cache type="com.example.cache.MybatisRedisCache"/>
</mapper>
```

---

# Part 6: MyBatis 插件（拦截器）机制 <a id="part-6"></a>

## 6.1 插件原理：JDK 动态代理 + 责任链模式

MyBatis 插件通过 **JDK 动态代理** 对四大核心对象进行拦截，多个插件形成**责任链（Chain of Responsibility）**。

```
+--------------------------------------------------------------------------+
|                    MyBatis 插件拦截链（责任链模式）                         |
|                                                                            |
|  业务代码调用 executor.query()                                             |
|         |                                                                  |
|         v                                                                  |
|  +-----------------------+                                                 |
|  | Plugin3 代理对象       |  <- 最后注册的插件，最外层，最先执行            |
|  | (JDK Proxy)          |                                                  |
|  |    +-----------------+|                                                 |
|  |    | Plugin2 代理对象  ||  <- 第二层                                     |
|  |    | (JDK Proxy)     ||                                                 |
|  |    |  +-------------+||                                                 |
|  |    |  | Plugin1 代理 |||  <- 最先注册的插件，最内层，最后执行            |
|  |    |  | (JDK Proxy) |||                                                 |
|  |    |  | +---------+ |||                                                 |
|  |    |  | | 真实对象  | |||  <- SimpleExecutor / StatementHandler 等      |
|  |    |  | +---------+ |||                                                 |
|  |    |  +-------------+||                                                 |
|  |    +-----------------+|                                                 |
|  +-----------------------+                                                 |
|                                                                            |
|  注册顺序：Plugin1 -> Plugin2 -> Plugin3                                  |
|  执行顺序：Plugin3.before -> Plugin2.before -> Plugin1.before             |
|            -> 真实方法执行 ->                                              |
|            Plugin1.after -> Plugin2.after -> Plugin3.after               |
+--------------------------------------------------------------------------+
```

### 插件代理的创建原理

```java
// MyBatis 在创建四大对象时，会通过 InterceptorChain 包装代理
public class InterceptorChain {

    private final List<Interceptor> interceptors = new ArrayList<>();

    // 对目标对象 target 应用所有拦截器，返回最终的代理对象
    public Object pluginAll(Object target) {
        for (Interceptor interceptor : interceptors) {
            target = interceptor.plugin(target);
            // 每个插件的 plugin() 方法返回代理对象或原始对象
            // Plugin.wrap() 会判断 target 是否在 @Signature 中声明
        }
        return target;
    }

    public void addInterceptor(Interceptor interceptor) {
        interceptors.add(interceptor);
    }
}

// Plugin.wrap() 内部实现
public class Plugin implements InvocationHandler {

    private final Object target;
    private final Interceptor interceptor;
    private final Map<Class<?>, Set<Method>> signatureMap;

    // 创建代理对象
    public static Object wrap(Object target, Interceptor interceptor) {
        Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
        Class<?> type = target.getClass();
        Class<?>[] interfaces = getAllInterfaces(type, signatureMap);

        if (interfaces.length > 0) {
            // 目标对象实现了被拦截的接口，创建 JDK 动态代理
            return Proxy.newProxyInstance(
                type.getClassLoader(),
                interfaces,
                new Plugin(target, interceptor, signatureMap)
            );
        }
        // 否则直接返回原始对象（不需要代理）
        return target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        try {
            Set<Method> methods = signatureMap.get(method.getDeclaringClass());
            if (methods != null && methods.contains(method)) {
                // 该方法需要被拦截，调用拦截器的 intercept 方法
                return interceptor.intercept(new Invocation(target, method, args));
            }
            // 不需要拦截，直接调用原始方法
            return method.invoke(target, args);
        } catch (Exception e) {
            throw ExceptionUtil.unwrapThrowable(e);
        }
    }
}
```

---

## 6.2 四大可拦截对象

```
可拦截的四大对象和方法：

1. Executor（执行器）- 最常用的拦截点
   update(MappedStatement ms, Object parameter)
   query(MappedStatement ms, Object parameter, RowBounds rowBounds,
         ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql)
   query(MappedStatement ms, Object parameter, RowBounds rowBounds,
         ResultHandler resultHandler)
   flushStatements()
   commit(boolean required)
   rollback(boolean required)
   getTransaction()
   close(boolean forceRollback)
   isClosed()

2. ParameterHandler（参数处理器）
   getParameterObject()
   setParameters(PreparedStatement ps)

3. ResultSetHandler（结果集处理器）
   handleResultSets(Statement stmt)
   handleCursorResultSets(Statement stmt)
   handleOutputParameters(CallableStatement cs)

4. StatementHandler（语句处理器）
   prepare(Connection connection, Integer transactionTimeout)
   parameterize(Statement statement)
   batch(Statement statement)
   update(Statement statement)
   query(Statement statement, ResultHandler resultHandler)
```

---

## 6.3 @Intercepts / @Signature 注解详解

```java
@Intercepts({  // 可以声明多个拦截点
    @Signature(
        type = StatementHandler.class,     // 拦截的类型（四大对象之一）
        method = "prepare",                // 拦截的方法名
        args = {Connection.class, Integer.class}  // 方法的参数类型（精确匹配）
    ),
    @Signature(
        type = Executor.class,
        method = "update",
        args = {MappedStatement.class, Object.class}
    ),
    @Signature(
        type = Executor.class,
        method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
    )
})
public class MyPlugin implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // invocation.getTarget()   - 获取被代理的真实对象
        // invocation.getMethod()   - 获取被拦截的方法
        // invocation.getArgs()     - 获取方法参数数组
        // invocation.proceed()     - 调用原始方法

        Object target = invocation.getTarget();
        Method method = invocation.getMethod();
        Object[] args = invocation.getArgs();

        // 前置处理
        System.out.println("拦截前: " + method.getName());

        // 调用原始方法
        Object result = invocation.proceed();

        // 后置处理
        System.out.println("拦截后: " + method.getName());

        return result;
    }

    @Override
    public Object plugin(Object target) {
        // 判断是否需要代理（Plugin.wrap 内部会检查 @Signature）
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        // 接收 mybatis-config.xml 或 Spring Boot 配置中的参数
        String param = properties.getProperty("myParam", "defaultValue");
    }
}
```

---

## 6.4 实战案例1：SQL 性能监控插件

记录 SQL 执行时间，超过阈值输出慢 SQL 警告。

```java
package com.example.plugin;

import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.plugin.*;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.sql.Statement;
import java.util.Properties;

/**
 * SQL 性能监控插件
 * 拦截 StatementHandler 的 query 和 update 方法
 * 记录每条 SQL 的执行时间，超过阈值输出慢 SQL 警告
 */
@Intercepts({
    @Signature(type = StatementHandler.class, method = "query",
               args = {Statement.class, ResultHandler.class}),
    @Signature(type = StatementHandler.class, method = "update",
               args = {Statement.class})
})
public class SqlPerformancePlugin implements Interceptor {

    private static final Logger log = LoggerFactory.getLogger(SqlPerformancePlugin.class);

    private long slowThreshold = 1000L;  // 慢 SQL 阈值（毫秒）

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 获取当前的 StatementHandler
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        BoundSql boundSql = statementHandler.getBoundSql();

        // 格式化 SQL（去除多余空白）
        String sql = boundSql.getSql()
            .replaceAll("[\\s]+", " ")
            .trim();

        // 记录开始时间
        long startTime = System.currentTimeMillis();

        try {
            // 执行原始方法
            return invocation.proceed();
        } finally {
            long elapsed = System.currentTimeMillis() - startTime;

            if (elapsed > slowThreshold) {
                // 慢 SQL 警告
                log.warn("[慢SQL告警] 执行耗时: {}ms | SQL: {}", elapsed, sql);
            } else {
                log.debug("[SQL执行] 耗时: {}ms | SQL: {}", elapsed, sql);
            }
        }
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {
        String threshold = properties.getProperty("slowThreshold");
        if (threshold != null) {
            this.slowThreshold = Long.parseLong(threshold);
        }
    }
}
```

---

## 6.5 实战案例2：分页插件原理（PageHelper 原理解析）

```java
/**
 * PageHelper 分页插件原理演示（简化实现）
 *
 * 原理：
 * 1. 用户调用 PageHelper.startPage(1, 10) 设置分页参数到 ThreadLocal
 * 2. 插件拦截 Executor.query() 方法
 * 3. 从 ThreadLocal 获取分页参数，修改 SQL 为 LIMIT 查询
 * 4. 同时执行 COUNT 查询获取总数
 * 5. 返回 Page 对象（包含数据和总数）
 * 6. 查询结束后清除 ThreadLocal
 */
@Intercepts({
    @Signature(type = Executor.class, method = "query",
               args = {MappedStatement.class, Object.class,
                       RowBounds.class, ResultHandler.class})
})
public class PaginationPlugin implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 从 ThreadLocal 获取分页参数
        PageInfo pageInfo = PageContext.getPage();
        if (pageInfo == null) {
            // 无分页请求，正常执行
            return invocation.proceed();
        }

        try {
            Object[] args = invocation.getArgs();
            MappedStatement ms = (MappedStatement) args[0];
            Object parameter = args[1];

            // 获取原始 SQL
            BoundSql originalBoundSql = ms.getBoundSql(parameter);
            String originalSql = originalBoundSql.getSql().trim();

            // 执行 COUNT 查询
            String countSql = "SELECT COUNT(*) FROM (" + originalSql + ") AS count_tmp";
            long total = executeCount(invocation, ms, parameter, countSql);
            pageInfo.setTotal(total);

            if (total == 0) {
                return new ArrayList<>();
            }

            // 构建分页 SQL
            int offset = (pageInfo.getPageNum() - 1) * pageInfo.getPageSize();
            String pageSql = originalSql + " LIMIT " + pageInfo.getPageSize()
                             + " OFFSET " + offset;

            // 通过反射修改 BoundSql 的 SQL
            Field sqlField = BoundSql.class.getDeclaredField("sql");
            sqlField.setAccessible(true);
            sqlField.set(originalBoundSql, pageSql);

            // 执行分页查询
            return invocation.proceed();

        } finally {
            PageContext.clear();  // 必须清除，避免内存泄漏
        }
    }

    // 省略 executeCount 方法实现...

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {}
}

// ThreadLocal 分页上下文
public class PageContext {
    private static final ThreadLocal<PageInfo> PAGE_LOCAL = new ThreadLocal<>();

    public static void setPage(int pageNum, int pageSize) {
        PAGE_LOCAL.set(new PageInfo(pageNum, pageSize));
    }

    public static PageInfo getPage() {
        return PAGE_LOCAL.get();
    }

    public static void clear() {
        PAGE_LOCAL.remove();
    }
}

// 使用方式（与 PageHelper 完全一样）
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public PageResult<User> getUsers(int pageNum, int pageSize) {
        PageContext.setPage(pageNum, pageSize);  // 设置分页参数
        List<User> users = userMapper.selectAll();  // 插件自动处理分页
        // users 中已经包含分页数据，PageContext 中有 total
        return new PageResult<>(PageContext.getPage().getTotal(), users);
    }
}
```

---

## 6.6 实战案例3：多租户插件（自动追加 tenant_id）

```java
package com.example.plugin;

import net.sf.jsqlparser.parser.CCJSqlParserUtil;
import net.sf.jsqlparser.statement.Statement;
import net.sf.jsqlparser.statement.delete.Delete;
import net.sf.jsqlparser.statement.select.PlainSelect;
import net.sf.jsqlparser.statement.select.Select;
import net.sf.jsqlparser.statement.update.Update;
import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.mapping.BoundSql;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.session.ResultHandler;
import org.apache.ibatis.session.RowBounds;

import java.lang.reflect.Field;
import java.util.Properties;

/**
 * 多租户插件
 * 自动在 SQL 中追加 tenant_id 过滤条件
 * 实现数据隔离，每个租户只能访问自己的数据
 */
@Intercepts({
    @Signature(type = Executor.class, method = "query",
               args = {MappedStatement.class, Object.class,
                       RowBounds.class, ResultHandler.class}),
    @Signature(type = Executor.class, method = "update",
               args = {MappedStatement.class, Object.class})
})
public class MultiTenantPlugin implements Interceptor {

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];
        Object parameter = args[1];

        // 获取当前租户 ID（从请求上下文/Token 中获取）
        Long tenantId = TenantContext.getTenantId();
        if (tenantId == null) {
            // 无租户信息（超级管理员），不限制
            return invocation.proceed();
        }

        // 检查该 SQL 是否跳过多租户
        if (isIgnoreTenant(ms)) {
            return invocation.proceed();
        }

        // 获取原始 SQL
        BoundSql boundSql = ms.getBoundSql(parameter);
        String originalSql = boundSql.getSql().trim();

        // 解析并修改 SQL
        String modifiedSql = appendTenantCondition(originalSql, tenantId);

        // 通过反射修改 BoundSql 中的 SQL
        Field sqlField = BoundSql.class.getDeclaredField("sql");
        sqlField.setAccessible(true);
        sqlField.set(boundSql, modifiedSql);

        return invocation.proceed();
    }

    private String appendTenantCondition(String sql, Long tenantId) {
        try {
            Statement statement = CCJSqlParserUtil.parse(sql);

            if (statement instanceof Select) {
                // SELECT 语句
                PlainSelect plainSelect =
                    (PlainSelect) ((Select) statement).getSelectBody();
                String tenantExpr = "tenant_id = " + tenantId;
                if (plainSelect.getWhere() == null) {
                    plainSelect.setWhere(
                        CCJSqlParserUtil.parseCondExpression(tenantExpr));
                } else {
                    String newWhere = "(" + plainSelect.getWhere() + ")"
                                    + " AND " + tenantExpr;
                    plainSelect.setWhere(
                        CCJSqlParserUtil.parseCondExpression(newWhere));
                }
                return statement.toString();

            } else if (statement instanceof Update) {
                // UPDATE 语句
                Update update = (Update) statement;
                String tenantExpr = "tenant_id = " + tenantId;
                if (update.getWhere() == null) {
                    update.setWhere(CCJSqlParserUtil.parseCondExpression(tenantExpr));
                } else {
                    String newWhere = "(" + update.getWhere() + ")"
                                    + " AND " + tenantExpr;
                    update.setWhere(CCJSqlParserUtil.parseCondExpression(newWhere));
                }
                return statement.toString();

            } else if (statement instanceof Delete) {
                // DELETE 语句
                Delete delete = (Delete) statement;
                String tenantExpr = "tenant_id = " + tenantId;
                if (delete.getWhere() == null) {
                    delete.setWhere(CCJSqlParserUtil.parseCondExpression(tenantExpr));
                } else {
                    String newWhere = "(" + delete.getWhere() + ")"
                                    + " AND " + tenantExpr;
                    delete.setWhere(CCJSqlParserUtil.parseCondExpression(newWhere));
                }
                return statement.toString();
            }
        } catch (Exception e) {
            // SQL 解析失败时，使用简单字符串追加（不推荐）
            // 或直接返回原 SQL 并记录错误
        }
        return sql;
    }

    private boolean isIgnoreTenant(MappedStatement ms) {
        // 检查 Mapper 接口方法是否有 @IgnoreTenant 注解
        String statementId = ms.getId();
        // 可以通过 statementId 找到 Mapper 接口方法，检查注解
        // 简化实现：通过黑名单配置
        return false;
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {}
}

// 租户上下文
public class TenantContext {
    private static final ThreadLocal<Long> TENANT_LOCAL = new ThreadLocal<>();

    public static void setTenantId(Long tenantId) { TENANT_LOCAL.set(tenantId); }
    public static Long getTenantId() { return TENANT_LOCAL.get(); }
    public static void clear() { TENANT_LOCAL.remove(); }
}
```

---

## 6.7 实战案例4：数据权限插件（行级权限）

```java
/**
 * 数据权限插件（行级权限控制）
 * 根据当前用户的权限范围，自动追加 WHERE 条件
 *
 * 例如：
 * - 普通员工只能查看自己的数据：WHERE creator_id = ?
 * - 部门经理只能查看本部门数据：WHERE dept_id IN (?, ?, ?)
 * - 超级管理员可以查看所有数据：不追加条件
 */
@Intercepts({
    @Signature(type = Executor.class, method = "query",
               args = {MappedStatement.class, Object.class,
                       RowBounds.class, ResultHandler.class})
})
@Component  // Spring 管理的插件
public class DataPermissionPlugin implements Interceptor {

    @Autowired
    private DeptService deptService;  // 用于查询用户有权限的部门列表

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object[] args = invocation.getArgs();
        MappedStatement ms = (MappedStatement) args[0];

        // 只处理 SELECT 语句
        if (ms.getSqlCommandType() != SqlCommandType.SELECT) {
            return invocation.proceed();
        }

        // 获取当前登录用户
        LoginUser loginUser = SecurityUtils.getCurrentUser();
        if (loginUser == null || loginUser.isSuperAdmin()) {
            return invocation.proceed();
        }

        // 获取 Mapper 方法上的 @DataScope 注解
        DataScope dataScope = getDataScopeAnnotation(ms);
        if (dataScope == null) {
            return invocation.proceed();
        }

        // 根据数据权限类型构建 SQL 条件
        String permissionSql = buildPermissionSql(loginUser, dataScope);
        if (permissionSql == null || permissionSql.isEmpty()) {
            return invocation.proceed();
        }

        // 修改 SQL 追加权限条件
        Object parameter = args[1];
        BoundSql boundSql = ms.getBoundSql(parameter);
        String originalSql = boundSql.getSql().trim();
        String wrappedSql = "SELECT * FROM (" + originalSql + ") data_scope_tmp "
                           + "WHERE " + permissionSql;

        Field sqlField = BoundSql.class.getDeclaredField("sql");
        sqlField.setAccessible(true);
        sqlField.set(boundSql, wrappedSql);

        return invocation.proceed();
    }

    private String buildPermissionSql(LoginUser user, DataScope dataScope) {
        String scopeType = user.getDataScopeType();

        switch (scopeType) {
            case "SELF":
                // 只能查看自己的数据
                return dataScope.userAlias() + ".creator_id = " + user.getId();
            case "DEPT":
                // 只能查看本部门的数据
                return dataScope.deptAlias() + ".dept_id = " + user.getDeptId();
            case "DEPT_AND_BELOW":
                // 可以查看本部门及下级部门的数据
                List<Long> deptIds = deptService.getDeptAndBelowIds(user.getDeptId());
                if (deptIds.isEmpty()) return null;
                return dataScope.deptAlias() + ".dept_id IN ("
                    + deptIds.stream().map(String::valueOf)
                              .collect(Collectors.joining(",")) + ")";
            case "ALL":
            default:
                return null;  // 无限制
        }
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {}
}

// 数据权限注解
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DataScope {
    String deptAlias() default "d";   // 部门表的别名
    String userAlias() default "u";   // 用户表的别名
}

// 在 Mapper 中使用
@Mapper
public interface OrderMapper {
    @DataScope(userAlias = "o", deptAlias = "o")
    List<Order> selectList(OrderQueryDTO query);
}
```

---

# Part 7: MyBatis 执行器（Executor）原理 <a id="part-7"></a>

## 7.1 执行器体系

```
Executor 接口
  |
  +-- BaseExecutor（抽象类）
  |     提供：一级缓存、事务管理、懒加载等基础功能
  |
  |     +-- SimpleExecutor（简单执行器）
  |     |     每次执行都会新建并关闭 Statement
  |     |     适合：普通 CRUD 操作（默认选项）
  |     |
  |     +-- ReuseExecutor（复用执行器）
  |     |     在同一 SqlSession 内，复用相同 SQL 的 PreparedStatement
  |     |     减少 PreparedStatement 的重复创建开销
  |     |     适合：同一 SqlSession 内多次执行相同 SQL
  |     |
  |     +-- BatchExecutor（批量执行器）
  |           批量执行 SQL，通过 JDBC 的 addBatch/executeBatch 实现
  |           适合：大批量的 INSERT/UPDATE/DELETE 操作
  |
  +-- CachingExecutor（缓存执行器，装饰者模式）
        包装上面三种执行器，在其外层添加二级缓存功能
        当 cacheEnabled=true 时，自动包装为 CachingExecutor
```

---

## 7.2 SimpleExecutor（简单执行器）

SimpleExecutor 是最简单的执行器，也是默认的执行器。**每次执行 SQL 都会创建一个新的 Statement（PreparedStatement），执行完毕后立即关闭**。

```java
// SimpleExecutor 核心实现
public class SimpleExecutor extends BaseExecutor {

    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            // 每次都创建新的 StatementHandler
            StatementHandler handler = configuration.newStatementHandler(
                this, ms, parameter, RowBounds.DEFAULT, null, null);
            // 准备 Statement（每次都新建）
            stmt = prepareStatement(handler, ms.getStatementLog());
            return handler.update(stmt);
        } finally {
            // 使用完毕，立即关闭
            closeStatement(stmt);
        }
    }

    @Override
    public <E> List<E> doQuery(MappedStatement ms, Object parameter,
                                RowBounds rowBounds, ResultHandler resultHandler,
                                BoundSql boundSql) throws SQLException {
        Statement stmt = null;
        try {
            Configuration configuration = ms.getConfiguration();
            StatementHandler handler = configuration.newStatementHandler(
                wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
            // 每次都新建 PreparedStatement
            stmt = prepareStatement(handler, ms.getStatementLog());
            return handler.query(stmt, resultHandler);
        } finally {
            closeStatement(stmt);
        }
    }

    private Statement prepareStatement(StatementHandler handler, Log statementLog)
            throws SQLException {
        Statement stmt;
        Connection connection = getConnection(statementLog);
        // conn.prepareStatement(sql)
        stmt = handler.prepare(connection, transaction.getTimeout());
        // ps.setXxx(i, value) 设置参数
        handler.parameterize(stmt);
        return stmt;
    }
}
```

```
SimpleExecutor 执行流程：
  query(sql, param)
    -> 从连接池获取 Connection
    -> conn.prepareStatement(sql)     <- 创建新 PreparedStatement
    -> ps.setXxx(参数)
    -> ps.executeQuery()
    -> handleResultSet(rs)
    -> ps.close()                     <- 立即关闭

优点：实现简单，无状态，线程安全性好
缺点：相同 SQL 执行多次时，每次都重新创建 PreparedStatement（有开销）
适用：默认推荐，大多数场景使用
```

---

## 7.3 ReuseExecutor（复用执行器）

ReuseExecutor 通过维护一个 `Map<String, Statement>` 来**缓存 PreparedStatement**，当同一 SQL 再次执行时，直接复用已创建的 PreparedStatement。

```java
// ReuseExecutor 核心实现
public class ReuseExecutor extends BaseExecutor {

    // 缓存：SQL -> PreparedStatement（在同一 SqlSession 内复用）
    private final Map<String, Statement> statementMap = new HashMap<>();

    @Override
    public int doUpdate(MappedStatement ms, Object parameter) throws SQLException {
        Configuration configuration = ms.getConfiguration();
        StatementHandler handler = configuration.newStatementHandler(
            this, ms, parameter, RowBounds.DEFAULT, null, null);
        // 复用已有的 Statement
        Statement stmt = prepareStatement(handler, ms.getStatementLog());
        return handler.update(stmt);
        // 注意：这里不关闭 stmt！Statement 被缓存复用
    }

    private Statement prepareStatement(StatementHandler handler, Log statementLog)
            throws SQLException {
        Statement stmt;
        BoundSql boundSql = handler.getBoundSql();
        String sql = boundSql.getSql();

        if (hasStatementFor(sql)) {
            // 命中缓存，复用已有的 PreparedStatement
            stmt = getStatement(sql);
            applyTransactionTimeout(stmt);
        } else {
            // 未命中缓存，新建并放入缓存
            Connection connection = getConnection(statementLog);
            stmt = handler.prepare(connection, transaction.getTimeout());
            putStatement(sql, stmt);  // 存入 statementMap
        }
        handler.parameterize(stmt);
        return stmt;
    }

    @Override
    public void doFlushStatements(boolean isRollback) {
        // SqlSession 关闭时，关闭所有缓存的 Statement
        for (Statement stmt : statementMap.values()) {
            closeStatement(stmt);
        }
        statementMap.clear();
    }
}
```

```
ReuseExecutor 使用场景：

场景：同一 SqlSession 内，多次执行相同的 SQL 但参数不同

// 开启 ReuseExecutor
try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.REUSE)) {
    UserMapper mapper = session.getMapper(UserMapper.class);

    // 第一次：创建 PreparedStatement，存入 statementMap
    User user1 = mapper.selectById(1L);

    // 第二次：复用 statementMap 中的 PreparedStatement（只设置参数，不重新创建）
    User user2 = mapper.selectById(2L);

    // 第三次：继续复用
    User user3 = mapper.selectById(3L);
}
// 关闭时，一次性关闭所有缓存的 PreparedStatement
```

---

## 7.4 BatchExecutor（批量执行器）

BatchExecutor 专用于批量操作，通过 JDBC 的 `addBatch()` / `executeBatch()` 机制，**将多条 SQL 攒在一起批量发送给数据库**，大幅减少网络往返次数。

```java
// BatchExecutor 核心实现
public class BatchExecutor extends BaseExecutor {

    public static final int BATCH_UPDATE_RETURN_VALUE = Integer.MIN_VALUE + 1002;

    private final List<Statement> statementList = new ArrayList<>();
    private final List<BatchResult> batchResultList = new ArrayList<>();
    private String currentSql;

    @Override
    public int doUpdate(MappedStatement ms, Object parameterObject) throws SQLException {
        // ...
        Statement stmt;
        if (sql.equals(currentSql) && ms.equals(currentStatement)) {
            // 相同 SQL，复用 Statement，仅 addBatch 新参数
            int last = statementList.size() - 1;
            stmt = statementList.get(last);
            applyTransactionTimeout(stmt);
            handler.parameterize(stmt);
            BatchResult batchResult = batchResultList.get(last);
            batchResult.addParameterObject(parameterObject);
        } else {
            // 新 SQL，创建新 Statement
            Connection connection = getConnection(ms.getStatementLog());
            stmt = handler.prepare(connection, transaction.getTimeout());
            handler.parameterize(stmt);
            currentSql = sql;
            currentStatement = ms;
            statementList.add(stmt);
            batchResultList.add(new BatchResult(ms, sql, parameterObject));
        }
        // 添加到批处理队列（不立即执行！）
        handler.batch(stmt);
        return BATCH_UPDATE_RETURN_VALUE;
    }

    @Override
    public List<BatchResult> doFlushStatements(boolean isRollback) throws SQLException {
        // flushStatements() 被调用时，才真正执行所有批处理
        for (int i = 0, n = statementList.size(); i < n; i++) {
            Statement stmt = statementList.get(i);
            // 执行批处理，获取每条 SQL 影响的行数
            int[] results = stmt.executeBatch();
            batchResultList.get(i).setUpdateCounts(results);
            closeStatement(stmt);
        }
        statementList.clear();
        batchResultList.clear();
        return batchResultList;
    }
}
```

**BatchExecutor 使用示例：**

```java
@Service
public class UserBatchService {

    @Autowired
    private SqlSessionFactory sqlSessionFactory;

    /**
     * 使用 BatchExecutor 批量插入 10000 条数据
     * 性能对比：
     * 单条插入: ~10 秒
     * foreach 批量: ~1 秒
     * BatchExecutor: ~0.5 秒（网络往返最少）
     */
    @Transactional
    public void batchInsert(List<User> userList) {
        // 手动创建使用 BatchExecutor 的 SqlSession
        try (SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH)) {
            UserMapper mapper = sqlSession.getMapper(UserMapper.class);

            for (int i = 0; i < userList.size(); i++) {
                mapper.insert(userList.get(i));

                // 每 500 条提交一次，避免内存溢出
                if (i > 0 && i % 500 == 0) {
                    sqlSession.flushStatements();  // 将攒的 SQL 发送给数据库
                }
            }

            // 最后再 flush 一次，确保剩余的 SQL 都被执行
            sqlSession.flushStatements();
            sqlSession.commit();

        } catch (Exception e) {
            throw new RuntimeException("批量插入失败", e);
        }
    }
}
```

```
BatchExecutor 性能原理：

普通方式（N 条 SQL）：
  网络: 客户端 -> 发送SQL1 -> 数据库 -> 返回结果1 -> 客户端
  网络: 客户端 -> 发送SQL2 -> 数据库 -> 返回结果2 -> 客户端
  ...（N 次网络往返）
  每次往返约 1-5ms，N=1000 时总计 1-5 秒

BatchExecutor 方式：
  addBatch(SQL1, param1)  <- 攒在本地，不发送
  addBatch(SQL2, param2)  <- 攒在本地，不发送
  addBatch(SQL3, param3)  <- 攒在本地，不发送
  ...
  executeBatch()          <- 一次性发送所有 SQL
  网络往返仅 1 次！
  1000 条 SQL 约 0.1-0.5 秒
```

---

## 7.5 CachingExecutor（二级缓存装饰者）

`CachingExecutor` 使用**装饰者模式**，包装其他执行器，在其外层添加二级缓存功能。

```java
public class CachingExecutor implements Executor {

    // 被装饰的真实执行器（SimpleExecutor/ReuseExecutor/BatchExecutor）
    private final Executor delegate;
    // 事务缓存管理器（管理二级缓存的事务性写入）
    private final TransactionalCacheManager tcm = new TransactionalCacheManager();

    @Override
    public <E> List<E> query(MappedStatement ms, Object parameterObject,
                              RowBounds rowBounds, ResultHandler resultHandler,
                              CacheKey key, BoundSql boundSql) throws SQLException {
        // 获取该 Mapper 的二级缓存
        Cache cache = ms.getCache();

        if (cache != null) {
            // 检查是否需要清空缓存（flushCache=true）
            flushCacheIfRequired(ms);

            if (ms.isUseCache() && resultHandler == null) {
                ensureNoOutParams(ms, boundSql);

                // 查询二级缓存
                List<E> list = (List<E>) tcm.getObject(cache, key);

                if (list == null) {
                    // 二级缓存未命中，委托给真实执行器查询
                    list = delegate.query(ms, parameterObject, rowBounds,
                                         resultHandler, key, boundSql);
                    // 暂存结果（等 commit/close 时写入二级缓存）
                    tcm.putObject(cache, key, list);
                }
                return list;
            }
        }

        // 没有二级缓存，直接委托
        return delegate.query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
    }

    @Override
    public void commit(boolean required) throws SQLException {
        // 提交时将暂存的数据写入二级缓存
        delegate.commit(required);
        tcm.commit();  // 将 TransactionalCache 中的数据提交到真实缓存
    }

    @Override
    public void rollback(boolean required) throws SQLException {
        delegate.rollback(required);
        tcm.rollback();  // 回滚，丢弃暂存的缓存数据
    }
}
```

---

## 7.6 执行器选择与性能对比

```
执行器选择指南：

+-------------------+------------------+-------------------+------------------+
| 特性              | SimpleExecutor   | ReuseExecutor     | BatchExecutor    |
+-------------------+------------------+-------------------+------------------+
| Statement 管理    | 每次新建/关闭    | 复用相同SQL的stmt | addBatch批量执行  |
| 适用场景          | 通用（默认）     | 同Session重复SQL  | 大批量DML操作    |
| 内存占用          | 低               | 中等              | 高（攒SQL）       |
| 执行单条SQL速度   | 中               | 稍快              | 慢（需flush）     |
| 执行大批量SQL     | 慢               | 中                | 最快             |
| 获取即时结果      | 支持             | 支持              | 不支持（异步）    |
| 使用复杂度        | 简单             | 简单              | 需手动flush      |
+-------------------+------------------+-------------------+------------------+

推荐场景：
  SimpleExecutor  - 90% 的普通业务场景（增删改查）
  ReuseExecutor   - 批量查询，但每次 SQL 相同、参数不同（如批量查单个用户）
  BatchExecutor   - 大批量数据导入（INSERT/UPDATE 几千到几十万条）
```

**Spring Boot 中指定执行器类型：**

```java
// 方式1：全局默认执行器类型（在 application.yml 中）
# mybatis.configuration.default-executor-type: simple

// 方式2：特定场景使用 BatchExecutor
@Service
public class DataImportService {

    @Autowired
    private SqlSessionFactory sqlSessionFactory;

    @Autowired
    private UserMapper userMapper;  // 这个是普通的 SimpleExecutor Mapper

    @Transactional
    public void importUsers(List<User> users) {
        // 临时使用 BatchExecutor
        try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH, false)) {
            UserMapper batchMapper = session.getMapper(UserMapper.class);

            for (User user : users) {
                batchMapper.insert(user);
            }

            // 执行批量操作
            session.flushStatements();
            session.commit();
        }
    }
}
```

---

# Part 8: MyBatis 与 Spring 整合原理 <a id="part-8"></a>

## 8.1 整合原理概述

```
+------------------------------------------------------------------------+
|                    MyBatis + Spring 整合架构                            |
|                                                                          |
|  Spring 容器                                                             |
|    |                                                                     |
|    +-- DataSource（数据源 Bean）                                          |
|    |                                                                     |
|    +-- SqlSessionFactory（单例 Bean）                                    |
|    |     由 SqlSessionFactoryBean 创建和管理                              |
|    |                                                                     |
|    +-- SqlSessionTemplate（线程安全的 SqlSession 代理）                   |
|    |     替代原生 SqlSession，处理 Spring 事务集成                         |
|    |                                                                     |
|    +-- MapperScannerConfigurer（Mapper 扫描配置）                         |
|    |     扫描 @Mapper 注解的接口，为每个接口创建 MapperFactoryBean          |
|    |                                                                     |
|    +-- UserMapper（JDK 动态代理实现）                                    |
|    +-- OrderMapper（JDK 动态代理实现）                                   |
|    +-- ...（所有 Mapper 接口都是 Spring Bean）                            |
|                                                                          |
|  请求流程：                                                               |
|  @Autowired UserMapper -> MapperProxy -> SqlSessionTemplate              |
|                        -> 事务同步管理器获取 SqlSession                   |
|                        -> Executor -> JDBC -> 数据库                     |
+------------------------------------------------------------------------+
```

---

## 8.2 SqlSessionFactoryBean

`SqlSessionFactoryBean` 是 Spring 与 MyBatis 整合的核心类，实现了 `FactoryBean<SqlSessionFactory>` 和 `InitializingBean` 接口。它负责创建 `SqlSessionFactory` Bean。

```java
// mybatis-spring 源码简化版
public class SqlSessionFactoryBean
    implements FactoryBean<SqlSessionFactory>, InitializingBean, ApplicationListener<ApplicationEvent> {

    private Resource configLocation;           // mybatis-config.xml 位置
    private Resource[] mapperLocations;        // Mapper XML 文件位置
    private DataSource dataSource;             // 数据源
    private TransactionFactory transactionFactory;  // 事务工厂
    private String typeAliasesPackage;         // 别名包扫描
    private String typeHandlersPackage;        // 类型处理器包扫描
    private Interceptor[] plugins;             // 插件
    private Configuration configuration;       // 直接传入 Configuration

    // Spring 容器初始化后，自动调用此方法
    @Override
    public void afterPropertiesSet() throws Exception {
        // 构建 SqlSessionFactory
        this.sqlSessionFactory = buildSqlSessionFactory();
    }

    protected SqlSessionFactory buildSqlSessionFactory() throws Exception {
        // 1. 创建 Configuration
        Configuration targetConfiguration;
        if (this.configuration != null) {
            targetConfiguration = this.configuration;
        } else if (this.configLocation != null) {
            // 解析 mybatis-config.xml
            XMLConfigBuilder xmlConfigBuilder = new XMLConfigBuilder(
                this.configLocation.getInputStream(), null, this.configurationProperties);
            targetConfiguration = xmlConfigBuilder.parse();
        } else {
            targetConfiguration = new Configuration();
        }

        // 2. 配置数据源和事务（Spring 事务管理器）
        targetConfiguration.setEnvironment(new Environment(
            "SqlSessionFactoryBean",
            // 关键：使用 SpringManagedTransactionFactory，让 Spring 接管事务
            this.transactionFactory == null
                ? new SpringManagedTransactionFactory()
                : this.transactionFactory,
            this.dataSource
        ));

        // 3. 扫描 Mapper XML
        if (!isEmpty(this.mapperLocations)) {
            for (Resource mapperLocation : this.mapperLocations) {
                XMLMapperBuilder xmlMapperBuilder = new XMLMapperBuilder(
                    mapperLocation.getInputStream(),
                    targetConfiguration,
                    mapperLocation.toString(),
                    targetConfiguration.getSqlFragments()
                );
                xmlMapperBuilder.parse();
            }
        }

        // 4. 注册类型别名、TypeHandler、插件等...

        // 5. 构建并返回 SqlSessionFactory
        return new DefaultSqlSessionFactory(targetConfiguration);
    }

    @Override
    public SqlSessionFactory getObject() {
        return this.sqlSessionFactory;
    }

    @Override
    public Class<? extends SqlSessionFactory> getObjectType() {
        return SqlSessionFactory.class;
    }

    @Override
    public boolean isSingleton() {
        return true;  // SqlSessionFactory 是单例
    }
}
```

**Spring Boot 中 SqlSessionFactory 的自动配置：**

```java
// mybatis-spring-boot-autoconfigure 源码简化
@Configuration(proxyBeanMethods = false)
@ConditionalOnClass({SqlSessionFactory.class, SqlSessionFactoryBean.class})
@ConditionalOnSingleCandidate(DataSource.class)
@EnableConfigurationProperties(MybatisProperties.class)
@AutoConfigureAfter({DataSourceAutoConfiguration.class, ...})
public class MybatisAutoConfiguration implements InitializingBean {

    private final MybatisProperties properties;
    private final DataSource dataSource;

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
        SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
        factory.setDataSource(dataSource);
        // 应用 application.yml 中的 mybatis.* 配置
        factory.setMapperLocations(this.properties.resolveMapperLocations());
        factory.setTypeAliasesPackage(this.properties.getTypeAliasesPackage());
        // ...
        return factory.getObject();
    }

    @Bean
    @ConditionalOnMissingBean
    public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
        return new SqlSessionTemplate(sqlSessionFactory);
    }
}
```

---

## 8.3 SqlSessionTemplate（线程安全的 SqlSession 代理）

`SqlSessionTemplate` 是 Spring 与 MyBatis 整合中最核心的类。它实现了 `SqlSession` 接口，内部通过 **JDK 动态代理**创建了一个 `SqlSession` 代理对象，将所有 SQL 操作委托给代理处理，从而实现线程安全和 Spring 事务整合。

```java
public class SqlSessionTemplate implements SqlSession, DisposableBean {

    private final SqlSessionFactory sqlSessionFactory;
    private final ExecutorType executorType;
    private final SqlSession sqlSessionProxy;  // JDK 动态代理创建的 SqlSession

    public SqlSessionTemplate(SqlSessionFactory sqlSessionFactory,
                               ExecutorType executorType,
                               PersistenceExceptionTranslator exceptionTranslator) {
        this.sqlSessionFactory = sqlSessionFactory;
        this.executorType = executorType;

        // 创建 SqlSession 的 JDK 动态代理
        this.sqlSessionProxy = (SqlSession) Proxy.newProxyInstance(
            SqlSessionFactory.class.getClassLoader(),
            new Class[] { SqlSession.class },
            new SqlSessionInterceptor()  // 拦截所有 SqlSession 方法
        );
    }

    // SqlSessionTemplate 的所有方法都委托给代理
    @Override
    public <T> T selectOne(String statement) {
        return this.sqlSessionProxy.selectOne(statement);
    }

    // ... 其他方法同理

    // SqlSession 代理的 InvocationHandler
    private class SqlSessionInterceptor implements InvocationHandler {
        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // 1. 从 Spring 事务管理器获取 SqlSession（线程安全！）
            SqlSession sqlSession = getSqlSession(
                SqlSessionTemplate.this.sqlSessionFactory,
                SqlSessionTemplate.this.executorType,
                SqlSessionTemplate.this.exceptionTranslator
            );

            try {
                // 2. 调用真实 SqlSession 的方法
                Object result = method.invoke(sqlSession, args);

                // 3. 如果当前没有 Spring 事务，则立即提交
                if (!isSqlSessionTransactional(sqlSession, SqlSessionTemplate.this.sqlSessionFactory)) {
                    sqlSession.commit(true);
                }

                return result;
            } catch (Throwable t) {
                throw exceptionTranslator.translateExceptionIfPossible((PersistenceException) t);
            } finally {
                if (sqlSession != null) {
                    // 4. 关闭 SqlSession（如果没有事务则真正关闭，有事务则不关闭）
                    closeSqlSession(sqlSession, SqlSessionTemplate.this.sqlSessionFactory);
                }
            }
        }
    }
}
```

**`getSqlSession` 的核心逻辑：**

```java
// SqlSessionUtils.getSqlSession 核心逻辑
public static SqlSession getSqlSession(SqlSessionFactory sessionFactory, ...) {
    // 1. 从 Spring 事务同步管理器中，查找当前线程绑定的 SqlSession
    SqlSessionHolder holder =
        (SqlSessionHolder) TransactionSynchronizationManager.getResource(sessionFactory);

    SqlSession session = sessionHolder(executorType, holder);

    if (session != null) {
        // 2. 找到了！当前线程有活跃的事务（@Transactional），复用同一个 SqlSession
        return session;
    }

    // 3. 没有找到（无事务）：创建一个新的 SqlSession
    session = sessionFactory.openSession(executorType);

    // 4. 如果 Spring 事务正在进行（比如通过 TransactionTemplate），注册到事务同步
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        // 注册到 Spring 事务，事务结束时自动关闭/提交
        TransactionSynchronizationManager.registerSynchronization(
            new SqlSessionSynchronization(holder, sessionFactory)
        );
    }

    return session;
}
```

---

## 8.4 MapperFactoryBean 和 MapperScannerConfigurer

每个 Mapper 接口都会被包装成一个 `MapperFactoryBean`，后者通过 `getObject()` 方法返回 Mapper 的 JDK 动态代理实例。

```java
// MapperFactoryBean 简化实现
public class MapperFactoryBean<T> extends SqlSessionDaoSupport
    implements FactoryBean<T> {

    private Class<T> mapperInterface;  // 被代理的 Mapper 接口

    @Override
    public T getObject() throws Exception {
        // 通过 SqlSessionTemplate 获取 Mapper 代理对象
        return getSqlSession().getMapper(this.mapperInterface);
    }

    @Override
    public Class<T> getObjectType() {
        return this.mapperInterface;
    }

    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

**`@MapperScan` 扫描原理：**

```java
// @MapperScan 注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(MapperScannerRegistrar.class)  // 导入 MapperScannerRegistrar
public @interface MapperScan {
    String[] value() default {};
    String[] basePackages() default {};
    Class<?>[] basePackageClasses() default {};
    Class<? extends Annotation> annotationClass() default Annotation.class;
    Class<?> markerInterface() default Class.class;
    // ...
}

// MapperScannerRegistrar 的工作
public class MapperScannerRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
                                         BeanDefinitionRegistry registry) {
        AnnotationAttributes mapperScanAttrs = AnnotationAttributes.fromMap(
            importingClassMetadata.getAnnotationAttributes(MapperScan.class.getName())
        );

        // 创建 ClassPathMapperScanner（继承自 Spring 的 ClassPathBeanDefinitionScanner）
        ClassPathMapperScanner scanner = new ClassPathMapperScanner(registry);

        // 配置扫描注解
        Class<? extends Annotation> annotationClass =
            mapperScanAttrs.getClass("annotationClass");
        scanner.setAnnotationClass(annotationClass);

        // 执行扫描
        scanner.scan(StringUtils.toStringArray(basePackages));
    }
}

// ClassPathMapperScanner 扫描到 Mapper 接口后：
// 将每个接口的 BeanDefinition 的 beanClass 改为 MapperFactoryBean
// 这样 Spring 容器在初始化时，会调用 MapperFactoryBean.getObject()
// 返回 Mapper 的代理实例作为 Bean
```

---

## 8.5 Spring 事务与 MyBatis 的集成

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 演示 Spring 事务与 MyBatis 的集成
     *
     * @Transactional 的工作原理：
     * 1. Spring AOP 拦截方法调用
     * 2. 开启事务：调用 DataSourceTransactionManager.getTransaction()
     *    -> 从 DataSource 获取 Connection
     *    -> connection.setAutoCommit(false)
     *    -> 将 Connection 绑定到当前线程（TransactionSynchronizationManager）
     * 3. 执行业务逻辑
     *    -> userMapper.insert() 调用 SqlSessionTemplate
     *    -> SqlSessionTemplate 从 TransactionSynchronizationManager 获取 SqlSession
     *    -> SqlSession 内部复用与事务绑定的 Connection
     * 4. 方法结束：
     *    -> 无异常：connection.commit()
     *    -> 有异常：connection.rollback()
     *    -> connection.setAutoCommit(true)
     *    -> 关闭 SqlSession
     *    -> 归还 Connection 到连接池
     */
    @Transactional(rollbackFor = Exception.class)
    public void createUserWithOrder(User user, Order order) {
        // 两个 Mapper 操作在同一个 Spring 事务中
        // 底层使用同一个 Connection，同一个 SqlSession
        userMapper.insert(user);            // SQL1
        order.setUserId(user.getId());
        orderMapper.insert(order);          // SQL2

        // 如果这里抛出异常，SQL1 和 SQL2 都会回滚
        if (order.getAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("订单金额必须大于0");
        }
    }

    /**
     * 事务传播级别示例
     */
    @Transactional(propagation = Propagation.REQUIRED)        // 默认，加入或新建事务
    public void methodA() { ... }

    @Transactional(propagation = Propagation.REQUIRES_NEW)    // 总是新建事务
    public void methodB() { ... }

    @Transactional(propagation = Propagation.NESTED)          // 嵌套事务（savepoint）
    public void methodC() { ... }

    @Transactional(propagation = Propagation.NOT_SUPPORTED)   // 非事务方式运行
    public void methodD() { ... }
}
```

**Spring 事务与 MyBatis 的数据流向：**

```
@Transactional 方法开始
    |
    v
DataSourceTransactionManager.doBegin()
    -> conn = dataSource.getConnection()
    -> conn.setAutoCommit(false)
    -> 绑定 conn 到 ThreadLocal

    |
    v
userMapper.insert(user)
    -> SqlSessionTemplate.insert()
    -> SqlSessionInterceptor.invoke()
    -> getSqlSession(factory) -> 从 ThreadLocal 获取 SqlSession
    -> SqlSession 内部 -> Executor
    -> SpringManagedTransaction.getConnection()
        -> 从 TransactionSynchronizationManager 获取当前线程的 conn
    -> conn.prepareStatement(sql)
    -> ps.executeUpdate()

    |
    v
orderMapper.insert(order)
    -> 同上，复用同一个 conn（事务保证）

    |
    v
DataSourceTransactionManager.doCommit()
    -> conn.commit()
    -> conn.setAutoCommit(true)
    -> 从 ThreadLocal 解绑 conn
    -> dataSource.releaseConnection(conn)
```

---

# Part 9: MyBatis-Plus 详解 <a id="part-9"></a>

## 9.1 MyBatis-Plus 是什么

MyBatis-Plus（简称 MP）是 MyBatis 的增强工具，在 MyBatis 的基础上**只做增强，不做改变**，为简化开发、提高效率而生。

**核心特性：**
- **无侵入**：只做增强不做改变，引入不影响现有工程
- **损耗小**：启动即会自动注入基本 CURD，性能基本无损耗
- **强大的 CRUD 操作**：内置通用 Mapper、通用 Service，少量配置即可实现单表大部分 CRUD 操作
- **支持 Lambda 形式调用**：通过 Lambda 表达式，方便编写各类查询条件，无需担心字段写错
- **支持主键自动生成**：支持多达 4 种主键策略（雪花算法、UUID、自增 ID 等）
- **支持 ActiveRecord 模式**
- **支持自定义全局通用操作**：支持全局通用方法注入
- **内置代码生成器**：采用代码或者 Maven 插件可快速生成 Mapper、Model、Service、Controller 层代码
- **内置分页插件**
- **内置性能分析插件**
- **内置全局拦截插件**

---

## 9.2 pom.xml 依赖

```xml
<!-- MyBatis-Plus Spring Boot Starter -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-spring-boot3-starter</artifactId>
    <version>3.5.7</version>
</dependency>

<!-- MySQL 驱动 -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>

<!-- Lombok -->
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

---

## 9.3 实体类注解

```java
package com.example.entity;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data
@TableName("sys_user")       // 指定数据库表名（默认是类名的下划线形式）
public class User {

    @TableId(
        value = "id",        // 对应数据库列名
        type = IdType.ASSIGN_ID  // 主键类型
    )
    // IdType 选项：
    // AUTO         - 数据库自增
    // NONE         - 无状态（手动设置）
    // INPUT        - 手动输入
    // ASSIGN_ID    - 雪花算法（默认，Long/String 类型）
    // ASSIGN_UUID  - UUID（不含横线的 UUID，String 类型）
    private Long id;

    @TableField("username")     // 指定列名（默认是属性名的下划线形式）
    private String username;

    @TableField("email")
    private String email;

    @TableField(exist = false)  // 不是数据库字段，排除
    private String confirmPassword;

    @TableField(
        fill = FieldFill.INSERT    // 自动填充：仅在 INSERT 时填充
    )
    private LocalDateTime createTime;

    @TableField(
        fill = FieldFill.INSERT_UPDATE  // 自动填充：INSERT 和 UPDATE 时都填充
    )
    private LocalDateTime updateTime;

    @Version  // 乐观锁版本字段
    private Integer version;

    @TableLogic  // 逻辑删除字段
    // 逻辑删除：查询时自动追加 WHERE deleted = 0
    // 删除时：UPDATE SET deleted = 1（而不是真正 DELETE）
    private Integer deleted;
}
```

---

## 9.4 BaseMapper 内置方法大全

```java
package com.example.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import org.apache.ibatis.annotations.Mapper;

@Mapper
public interface UserMapper extends BaseMapper<User> {
    // 继承 BaseMapper<User> 后，自动获得以下所有方法，无需写 XML！
}
```

**BaseMapper 内置方法完整列表：**

```java
// ==================== 插入 ====================

// 插入一条记录
int insert(T entity);

// ==================== 删除 ====================

// 根据 entity 条件删除记录
int delete(@Param(Constants.WRAPPER) Wrapper<T> wrapper);

// 删除（根据ID批量删除）
int deleteBatchIds(@Param(Constants.COLLECTION) Collection<?> idList);

// 根据 ID 删除
int deleteById(Serializable id);

// 根据 columnMap 条件删除记录
int deleteByMap(@Param(Constants.COLUMN_MAP) Map<String, Object> columnMap);

// ==================== 更新 ====================

// 根据 whereWrapper 条件，更新记录
int update(@Param(Constants.ENTITY) T updateEntity,
           @Param(Constants.WRAPPER) Wrapper<T> whereWrapper);

// 根据 ID 修改（null 字段不更新）
int updateById(@Param(Constants.ENTITY) T entity);

// ==================== 查询 ====================

// 根据 ID 查询
T selectById(Serializable id);

// 根据 entity 条件，查询一条记录
T selectOne(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 查询（根据ID批量查询）
List<T> selectBatchIds(@Param(Constants.COLLECTION) Collection<? extends Serializable> idList);

// 根据 entity 条件，查询全部记录
List<T> selectList(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 查询（根据 columnMap 条件）
List<T> selectByMap(@Param(Constants.COLUMN_MAP) Map<String, Object> columnMap);

// 根据 Wrapper 条件，查询全部记录（返回 Map）
List<Map<String, Object>> selectMaps(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 根据 Wrapper 条件，查询全部记录（返回第一个字段的值）
List<Object> selectObjs(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 根据 entity 条件，查询全部记录（分页）
IPage<T> selectPage(IPage<T> page, @Param(Constants.WRAPPER) Wrapper<T> queryWrapper);

// 根据 Wrapper 条件，查询总记录数
Long selectCount(@Param(Constants.WRAPPER) Wrapper<T> queryWrapper);
```

**使用示例：**

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public void demonstrate() {
        // 1. 插入
        User user = new User();
        user.setUsername("张三");
        user.setEmail("zs@example.com");
        userMapper.insert(user);
        System.out.println("自动生成的 ID: " + user.getId());  // 雪花算法ID

        // 2. 查询（根据ID）
        User found = userMapper.selectById(1L);

        // 3. 批量查询
        List<User> users = userMapper.selectBatchIds(Arrays.asList(1L, 2L, 3L));

        // 4. 更新（null 字段不更新）
        User updateUser = new User();
        updateUser.setId(1L);
        updateUser.setEmail("new@email.com");
        userMapper.updateById(updateUser);  // 只更新 email，其他字段不变

        // 5. 逻辑删除（因为有 @TableLogic，实际执行 UPDATE deleted=1）
        userMapper.deleteById(1L);

        // 6. 查询所有（因为有 @TableLogic，自动追加 WHERE deleted=0）
        List<User> allUsers = userMapper.selectList(null);
    }
}
```

---

## 9.5 Wrapper 条件构造器

条件构造器是 MyBatis-Plus 的核心特性，提供了链式 API 构建 SQL WHERE 条件。

### QueryWrapper（字符串字段名）

```java
@Service
public class UserQueryService {

    @Autowired
    private UserMapper userMapper;

    public void queryWrapper() {
        // 1. 基础条件查询
        QueryWrapper<User> wrapper = new QueryWrapper<>();
        wrapper
            .eq("username", "张三")           // WHERE username = '张三'
            .ne("status", 0)                  // AND status != 0
            .gt("age", 18)                    // AND age > 18
            .ge("age", 18)                    // AND age >= 18
            .lt("age", 60)                    // AND age < 60
            .le("age", 60)                    // AND age <= 60
            .between("age", 18, 60)           // AND age BETWEEN 18 AND 60
            .notBetween("age", 0, 17)         // AND age NOT BETWEEN 0 AND 17
            .like("username", "张")            // AND username LIKE '%张%'
            .notLike("username", "test")      // AND username NOT LIKE '%test%'
            .likeLeft("username", "三")        // AND username LIKE '%三'
            .likeRight("username", "张")       // AND username LIKE '张%'
            .isNull("email")                  // AND email IS NULL
            .isNotNull("email")               // AND email IS NOT NULL
            .in("status", 1, 2, 3)            // AND status IN (1, 2, 3)
            .in("id", Arrays.asList(1L, 2L))  // AND id IN (1, 2)
            .notIn("status", 0)               // AND status NOT IN (0)
            .inSql("id", "SELECT id FROM vip_user")  // AND id IN (SELECT id FROM vip_user)
            .exists("SELECT 1 FROM orders WHERE orders.user_id = user.id")  // AND EXISTS(...)
            .orderByDesc("create_time")        // ORDER BY create_time DESC
            .orderByAsc("id")                  // , id ASC
            .groupBy("dept_id")               // GROUP BY dept_id
            .having("count(*) > 5")           // HAVING count(*) > 5
            .last("LIMIT 10");                // 在末尾追加 SQL（谨慎使用）

        List<User> users = userMapper.selectList(wrapper);

        // 2. 指定查询的列（select 方法）
        QueryWrapper<User> selectWrapper = new QueryWrapper<>();
        selectWrapper
            .select("id", "username", "email")  // 只查指定列
            .eq("status", 1);
        List<User> partUsers = userMapper.selectList(selectWrapper);

        // 3. OR 条件
        QueryWrapper<User> orWrapper = new QueryWrapper<>();
        orWrapper
            .eq("username", "admin")
            .or()
            .eq("email", "admin@example.com");
        // WHERE username = 'admin' OR email = 'admin@example.com'

        // 4. 嵌套 OR
        QueryWrapper<User> nestedWrapper = new QueryWrapper<>();
        nestedWrapper
            .eq("status", 1)
            .and(w -> w.eq("username", "admin").or().eq("username", "root"));
        // WHERE status = 1 AND (username = 'admin' OR username = 'root')

        // 5. 条件判断（按需拼接，避免 null 条件）
        String keyword = null;
        Integer statusFilter = 1;
        QueryWrapper<User> condWrapper = new QueryWrapper<>();
        condWrapper
            // condition 为 true 时才加入该条件
            .like(StringUtils.hasText(keyword), "username", keyword)
            .eq(statusFilter != null, "status", statusFilter);
    }
}
```

### LambdaQueryWrapper（Lambda，推荐）

```java
public void lambdaQueryWrapper() {
    // LambdaQueryWrapper 使用方法引用代替字段名字符串
    // 优点：编译期检查，重构安全，不会写错字段名
    LambdaQueryWrapper<User> wrapper = new LambdaQueryWrapper<>();
    wrapper
        .eq(User::getUsername, "张三")      // 使用 getter 引用，而不是 "username" 字符串
        .gt(User::getAge, 18)
        .isNotNull(User::getEmail)
        .orderByDesc(User::getCreateTime)
        .select(User::getId, User::getUsername, User::getEmail);

    List<User> users = userMapper.selectList(wrapper);

    // 简洁写法（链式构建器）
    List<User> result = userMapper.selectList(
        Wrappers.<User>lambdaQuery()
            .eq(User::getStatus, 1)
            .like(User::getUsername, "张")
            .ge(User::getAge, 18)
            .orderByDesc(User::getCreateTime)
    );

    // 条件按需拼接（Lambda 版本）
    String name = "张三";
    Integer age = null;
    List<User> condResult = userMapper.selectList(
        Wrappers.<User>lambdaQuery()
            .like(name != null, User::getUsername, name)
            .gt(age != null, User::getAge, age)
    );
}
```

### UpdateWrapper 和 LambdaUpdateWrapper

```java
public void updateWrapper() {
    // UpdateWrapper：可以指定 SET 子句中的字段
    UpdateWrapper<User> wrapper = new UpdateWrapper<>();
    wrapper
        .set("email", "new@email.com")         // SET email = 'new@email.com'
        .set("status", 1)                      // , status = 1
        .set("update_time", LocalDateTime.now()) // , update_time = ?
        .eq("id", 1L);                         // WHERE id = 1

    userMapper.update(null, wrapper);  // 第一个参数为 null（字段通过 set 指定）

    // LambdaUpdateWrapper
    LambdaUpdateWrapper<User> lambdaWrapper = new LambdaUpdateWrapper<>();
    lambdaWrapper
        .set(User::getEmail, "new@email.com")
        .set(User::getStatus, 1)
        .eq(User::getId, 1L)
        .gt(User::getAge, 18);  // 额外的条件

    userMapper.update(null, lambdaWrapper);

    // 更简洁的 Wrappers 工厂方法
    userMapper.update(null, Wrappers.<User>lambdaUpdate()
        .set(User::getEmail, "updated@email.com")
        .set(User::getUpdateTime, LocalDateTime.now())
        .eq(User::getId, 1L)
    );
}
```

---

## 9.6 IService 和 ServiceImpl

MyBatis-Plus 提供了通用的 Service 层，内置了大量常用的业务方法。

```java
// Service 接口（继承 IService）
public interface UserService extends IService<User> {
    // 自定义业务方法
    User findByEmail(String email);
    PageResult<User> queryPage(UserQueryRequest request);
}

// Service 实现类（继承 ServiceImpl）
@Service
public class UserServiceImpl extends ServiceImpl<UserMapper, User>
    implements UserService {

    // ServiceImpl 提供了 this.baseMapper（即 UserMapper）
    // 和 this.getBaseMapper() 方法

    @Override
    public User findByEmail(String email) {
        return this.lambdaQuery()  // ServiceImpl 内置 lambdaQuery()
            .eq(User::getEmail, email)
            .one();  // 查询一条，多条则抛异常
    }

    @Override
    public PageResult<User> queryPage(UserQueryRequest request) {
        Page<User> page = new Page<>(request.getPageNum(), request.getPageSize());
        LambdaQueryWrapper<User> wrapper = Wrappers.<User>lambdaQuery()
            .like(request.getUsername() != null, User::getUsername, request.getUsername())
            .eq(request.getStatus() != null, User::getStatus, request.getStatus())
            .orderByDesc(User::getCreateTime);

        Page<User> result = this.page(page, wrapper);
        return new PageResult<>(result.getTotal(), result.getRecords());
    }
}
```

**IService 内置方法大全：**

```java
// ========== 保存/插入 ==========
boolean save(T entity);                        // 插入一条（null字段也插入）
boolean saveBatch(Collection<T> entityList);   // 批量插入（默认每批1000条）
boolean saveBatch(Collection<T> entityList, int batchSize);  // 批量插入（指定批大小）
boolean saveOrUpdate(T entity);                // 根据ID判断是插入还是更新
boolean saveOrUpdateBatch(Collection<T> entityList);

// ========== 删除 ==========
boolean removeById(Serializable id);           // 根据ID删除
boolean removeByIds(Collection<?> idList);     // 批量ID删除
boolean remove(Wrapper<T> queryWrapper);       // 条件删除
boolean removeByMap(Map<String, Object> columnMap);

// ========== 更新 ==========
boolean updateById(T entity);                  // 根据ID更新（null字段不更新）
boolean update(T entity, Wrapper<T> updateWrapper);  // 条件更新
boolean update(Wrapper<T> updateWrapper);      // 条件更新（不需要实体）
boolean updateBatchById(Collection<T> entityList);   // 批量ID更新

// ========== 查询 ==========
T getById(Serializable id);                    // 根据ID查询
T getOne(Wrapper<T> queryWrapper);             // 查询一条（多条报错）
T getOne(Wrapper<T> queryWrapper, boolean throwEx);  // 查询一条（false时不报错，返回null）
Map<String, Object> getMap(Wrapper<T> queryWrapper);
List<T> listByIds(Collection<? extends Serializable> idList);
List<T> list();                                // 查询所有
List<T> list(Wrapper<T> queryWrapper);         // 条件查询所有
List<Map<String, Object>> listMaps(Wrapper<T> queryWrapper);
long count();                                  // 统计总数
long count(Wrapper<T> queryWrapper);           // 条件统计

// ========== 分页 ==========
IPage<T> page(IPage<T> page);
IPage<T> page(IPage<T> page, Wrapper<T> queryWrapper);

// ========== 链式操作 ==========
LambdaQueryChainWrapper<T> lambdaQuery();
LambdaUpdateChainWrapper<T> lambdaUpdate();
QueryChainWrapper<T> query();
UpdateChainWrapper<T> update();
```

---

## 9.7 分页插件（PaginationInnerInterceptor）

```java
// Spring Boot 配置分页插件
@Configuration
@MapperScan("com.example.mapper")
public class MybatisPlusConfig {

    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        // 添加分页插件（必须指定数据库类型）
        PaginationInnerInterceptor paginationInterceptor =
            new PaginationInnerInterceptor(DbType.MYSQL);
        paginationInterceptor.setMaxLimit(500L);  // 每页最大条数限制
        paginationInterceptor.setOverflow(false); // 溢出总页数后返回第一页(false:不处理)

        interceptor.addInnerInterceptor(paginationInterceptor);

        // 也可以添加乐观锁插件
        interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());

        return interceptor;
    }
}
```

**使用分页：**

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public IPage<User> getPage(int pageNum, int pageSize, String keyword) {
        // 创建分页对象（当前页, 每页大小）
        Page<User> page = new Page<>(pageNum, pageSize);

        // 构建查询条件
        LambdaQueryWrapper<User> wrapper = Wrappers.<User>lambdaQuery()
            .like(StringUtils.hasText(keyword), User::getUsername, keyword)
            .eq(User::getStatus, 1)
            .orderByDesc(User::getCreateTime);

        // 执行分页查询
        IPage<User> result = userMapper.selectPage(page, wrapper);

        System.out.println("总记录数: " + result.getTotal());
        System.out.println("总页数: " + result.getPages());
        System.out.println("当前页: " + result.getCurrent());
        System.out.println("每页大小: " + result.getSize());
        System.out.println("数据列表: " + result.getRecords().size() + " 条");

        return result;
    }
}
```

---

## 9.8 乐观锁插件（@Version）

```java
@Data
@TableName("product")
public class Product {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stock;

    @Version  // 乐观锁版本字段
    private Integer version;  // 每次更新时版本号+1
}

// 配置乐观锁插件（在 MybatisPlusConfig 中已配置 OptimisticLockerInnerInterceptor）

// 使用乐观锁更新
@Service
public class ProductService {

    @Autowired
    private ProductMapper productMapper;

    /**
     * 扣减库存（乐观锁防止超卖）
     */
    public boolean deductStock(Long productId, int quantity) {
        // 1. 查询商品（包含 version）
        Product product = productMapper.selectById(productId);

        if (product.getStock() < quantity) {
            return false;  // 库存不足
        }

        // 2. 扣减库存
        product.setStock(product.getStock() - quantity);

        // 3. 更新时，MyBatis-Plus 自动追加 version 检查
        // 生成 SQL: UPDATE product SET stock=?, version=version+1
        //           WHERE id=? AND version=?  <- 乐观锁条件
        int result = productMapper.updateById(product);

        if (result == 0) {
            // 更新失败（version 不匹配，说明有并发修改）
            throw new OptimisticLockException("库存更新冲突，请重试");
        }

        return true;
    }
}
```

---

## 9.9 逻辑删除（@TableLogic）

```java
// 全局配置（application.yml）
// mybatis-plus:
//   global-config:
//     db-config:
//       logic-delete-field: deleted  # 逻辑删除字段名
//       logic-delete-value: 1        # 删除值
//       logic-not-delete-value: 0    # 未删除值

// 实体类
@Data
@TableName("user")
public class User {
    @TableId(type = IdType.AUTO)
    private Long id;
    private String username;

    @TableLogic  // 也可以不加此注解，通过全局配置指定字段名
    private Integer deleted;  // 0: 未删除, 1: 已删除
}

// 使用效果
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public void test() {
        // 查询：自动追加 WHERE deleted = 0
        // SQL: SELECT * FROM user WHERE deleted = 0
        List<User> users = userMapper.selectList(null);

        // 删除：执行 UPDATE 而不是 DELETE
        // SQL: UPDATE user SET deleted = 1 WHERE id = 1 AND deleted = 0
        userMapper.deleteById(1L);

        // 如果要查询已删除的数据（需要手写 SQL）：
        // SELECT * FROM user WHERE deleted = 1
    }
}
```

---

## 9.10 自动填充（@TableField fill）

```java
// 自动填充处理器
@Component
public class MyMetaObjectHandler implements MetaObjectHandler {

    @Override
    public void insertFill(MetaObject metaObject) {
        // 插入时自动填充
        this.strictInsertFill(metaObject, "createTime", LocalDateTime.class, LocalDateTime.now());
        this.strictInsertFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());

        // 从 SecurityContext 获取当前操作人
        Long userId = SecurityUtils.getCurrentUserId();
        this.strictInsertFill(metaObject, "createBy", Long.class, userId);
        this.strictInsertFill(metaObject, "updateBy", Long.class, userId);
    }

    @Override
    public void updateFill(MetaObject metaObject) {
        // 更新时自动填充
        this.strictUpdateFill(metaObject, "updateTime", LocalDateTime.class, LocalDateTime.now());

        Long userId = SecurityUtils.getCurrentUserId();
        this.strictUpdateFill(metaObject, "updateBy", Long.class, userId);
    }
}

// 实体类中的注解
@Data
public class BaseEntity {
    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableField(fill = FieldFill.INSERT)
    private Long createBy;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private Long updateBy;
}
```

---

## 9.11 代码生成器（AutoGenerator）

```java
// MyBatis-Plus 代码生成器（3.5.1+ 新版本 API）
public class CodeGenerator {

    public static void main(String[] args) {
        FastAutoGenerator.create(
            "jdbc:mysql://localhost:3306/mydb",
            "root",
            "password"
        )
        .globalConfig(builder -> builder
            .author("YourName")
            .outputDir(System.getProperty("user.dir") + "/src/main/java")
            .commentDate("yyyy-MM-dd")
            .enableSwagger()             // 开启 Swagger 注解
            .disableOpenDir()            // 生成后不打开目录
        )
        .packageConfig(builder -> builder
            .parent("com.example")       // 父包名
            .moduleName("user")          // 模块名
            .entity("entity")            // Entity 包名
            .mapper("mapper")            // Mapper 包名
            .service("service")          // Service 包名
            .serviceImpl("service.impl") // ServiceImpl 包名
            .controller("controller")    // Controller 包名
            .xml("mapper")               // XML 文件位置（resources下）
        )
        .strategyConfig(builder -> builder
            .addInclude("sys_user", "sys_role", "sys_menu")  // 需要生成的表
            .addTablePrefix("sys_", "t_")  // 表名前缀（生成实体时去掉）
            .entityBuilder()
                .enableLombok()                  // 开启 Lombok
                .enableTableFieldAnnotation()    // 开启字段注解
                .logicDeleteColumnName("deleted") // 逻辑删除字段
                .versionColumnName("version")     // 乐观锁字段
                .addTableFills(                   // 自动填充
                    new Column("create_time", FieldFill.INSERT),
                    new Column("update_time", FieldFill.INSERT_UPDATE)
                )
                .build()
            .controllerBuilder()
                .enableRestStyle()               // 生成 @RestController
                .enableHyphenStyle()             // URL 驼峰转连字符
                .build()
            .mapperBuilder()
                .enableBaseResultMap()           // 生成 BaseResultMap
                .enableBaseColumnList()          // 生成 Base_Column_List
                .build()
        )
        .templateEngine(new FreemarkerTemplateEngine())  // 使用 Freemarker 模板
        .execute();
    }
}
```

---

## 9.12 多数据源（dynamic-datasource）

```xml
<!-- 多数据源依赖 -->
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>dynamic-datasource-spring-boot3-starter</artifactId>
    <version>4.3.0</version>
</dependency>
```

```yaml
# application.yml 多数据源配置
spring:
  datasource:
    dynamic:
      primary: master          # 默认数据源
      strict: false            # 严格模式（找不到数据源是否报错）
      datasource:
        master:                # 主库（写）
          url: jdbc:mysql://localhost:3306/master_db
          username: root
          password: 123456
          driver-class-name: com.mysql.cj.jdbc.Driver
        slave_1:               # 从库1（读）
          url: jdbc:mysql://localhost:3307/slave_db
          username: root
          password: 123456
        slave_2:               # 从库2（读）
          url: jdbc:mysql://localhost:3308/slave_db
          username: root
          password: 123456
```

```java
// 在 Service 方法上使用 @DS 注解切换数据源
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    // 写操作走主库
    @DS("master")
    @Transactional
    public void createUser(User user) {
        userMapper.insert(user);
    }

    // 读操作走从库（读写分离）
    @DS("slave_1")
    public User findById(Long id) {
        return userMapper.selectById(id);
    }

    // 不加 @DS 注解，默认使用 primary（master）数据源
    public List<User> listUsers() {
        return userMapper.selectList(null);
    }
}
```

---

# Part 10: 高级特性 <a id="part-10"></a>

## 10.1 懒加载原理（CGLIB 代理）

MyBatis 的懒加载（延迟加载）通过 **CGLIB** 或 **Javassist** 动态代理实现。

```
懒加载原理：

1. 查询用户时，只执行主查询（SELECT * FROM user WHERE id = 1）
2. 返回的 User 对象实际上是 User 的 CGLIB 代理类实例
3. User 的 orders 属性初始值为 null（或特殊的占位对象）
4. 当程序第一次访问 user.getOrders() 时
   -> CGLIB 代理拦截 getOrders() 方法
   -> 发现 orders 还未加载
   -> 执行子查询：SELECT * FROM orders WHERE user_id = 1
   -> 将结果填充到 orders 属性
   -> 返回 orders 集合
5. 后续再访问 user.getOrders() 时，直接返回已加载的数据
```

**配置懒加载：**

```xml
<!-- mybatis-config.xml -->
<settings>
    <!-- 开启懒加载 -->
    <setting name="lazyLoadingEnabled" value="true"/>
    <!-- 非激进懒加载（按需加载，推荐）-->
    <setting name="aggressiveLazyLoading" value="false"/>
    <!-- 使用 CGLIB 代理（可代理没有接口的类）-->
    <setting name="proxyFactory" value="CGLIB"/>
</settings>
```

```xml
<!-- Mapper XML 中的懒加载配置 -->
<resultMap id="userWithOrdersLazy" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>

    <!-- fetchType="lazy" 覆盖全局配置，指定此关联使用懒加载 -->
    <!-- fetchType="eager" 立即加载（覆盖全局懒加载配置）-->
    <collection property="orders"
                column="id"
                select="selectOrdersByUserId"
                fetchType="lazy"/>
</resultMap>
```

**懒加载的坑（序列化问题）：**

```java
// 坑1：序列化时触发懒加载
// Jackson 序列化 User 对象时，会调用所有 getter，触发懒加载
// 如果此时 SqlSession 已关闭，会报错：
// org.apache.ibatis.executor.ExecutorException: Executor was closed.

// 解决方案：
// 1. 在 Controller 层的 @RestController 返回前，确保在 @Transactional 中完成所有数据加载
// 2. 使用 DTO 转换，在 Service 层（事务内）完成数据提取
// 3. 避免直接返回有懒加载属性的实体类给前端

// 坑2：aggressiveLazyLoading=true 时，调用 toString/hashCode 等也会触发全部懒加载
// 解决：设置 aggressiveLazyLoading=false（默认值）

// 正确使用懒加载的方式
@Service
public class UserService {

    @Transactional(readOnly = true)  // 事务保证 SqlSession 在整个方法内有效
    public UserVO getUserWithOrders(Long userId) {
        User user = userMapper.selectById(userId);  // 只查用户

        // 在事务内访问懒加载属性
        UserVO vo = new UserVO();
        vo.setId(user.getId());
        vo.setUsername(user.getUsername());

        // 这里触发懒加载，查询订单
        List<Order> orders = user.getOrders();  // 懒加载执行子查询
        vo.setOrderCount(orders.size());

        return vo;  // 返回 DTO，不直接返回 User 实体
    }
}
```

---

## 10.2 存储过程调用（statementType="CALLABLE"）

```sql
-- 创建存储过程（MySQL）
DELIMITER //
CREATE PROCEDURE get_user_stats(
    IN p_dept_id BIGINT,
    OUT p_user_count INT,
    OUT p_avg_age DECIMAL(5,2)
)
BEGIN
    SELECT COUNT(*), AVG(age)
    INTO p_user_count, p_avg_age
    FROM user
    WHERE dept_id = p_dept_id AND deleted = 0;
END //
DELIMITER ;
```

```xml
<!-- Mapper XML 调用存储过程 -->
<select id="getUserStats"
        statementType="CALLABLE"
        parameterMap="getUserStatsParamMap">
    {CALL get_user_stats(
        #{deptId, mode=IN, jdbcType=BIGINT},
        #{userCount, mode=OUT, jdbcType=INTEGER},
        #{avgAge, mode=OUT, jdbcType=DECIMAL}
    )}
</select>
```

```java
// DTO 对象
@Data
public class UserStatsDTO {
    private Long deptId;     // IN 参数
    private Integer userCount;  // OUT 参数（由存储过程输出）
    private BigDecimal avgAge;  // OUT 参数
}

// Mapper 接口
@Mapper
public interface UserMapper {
    void getUserStats(UserStatsDTO params);
}

// 使用
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    public UserStatsDTO getDeptStats(Long deptId) {
        UserStatsDTO dto = new UserStatsDTO();
        dto.setDeptId(deptId);

        userMapper.getUserStats(dto);  // 调用存储过程

        // dto.getUserCount() 和 dto.getAvgAge() 已被存储过程填充
        System.out.println("部门人数: " + dto.getUserCount());
        System.out.println("平均年龄: " + dto.getAvgAge());

        return dto;
    }
}
```

---

## 10.3 批量操作最佳实践

### 批量操作性能对比

```
场景：插入 10000 条用户数据

方式1：单条 INSERT（循环调用）
  for (User user : users) {
      userMapper.insert(user);  // 每次一条 SQL，10000 次网络往返
  }
  耗时：约 10-30 秒

方式2：foreach 批量 INSERT（一条 SQL）
  userMapper.insertBatch(users);  // 一条 SQL：INSERT INTO ... VALUES (...),(...),...
  耗时：约 0.5-2 秒
  注意：SQL 过长（超过 max_allowed_packet）时报错，建议每批 1000 条

方式3：BatchExecutor（JDBC batch）
  使用 addBatch/executeBatch，减少网络往返
  耗时：约 0.3-1 秒

性能排序：BatchExecutor ≈ foreach 批量 > 单条 >> 单条（无事务）

推荐：数据量 < 1000 条用 foreach 批量；> 1000 条用 BatchExecutor 分批处理
```

**通用批量操作工具方法：**

```java
@Service
public class BatchOperationService {

    @Autowired
    private SqlSessionFactory sqlSessionFactory;

    private static final int DEFAULT_BATCH_SIZE = 500;

    /**
     * 通用批量插入方法
     */
    @Transactional
    public <T> void batchInsert(Class<?> mapperClass, String methodName,
                                 List<T> dataList) {
        batchInsert(mapperClass, methodName, dataList, DEFAULT_BATCH_SIZE);
    }

    @Transactional
    public <T> void batchInsert(Class<?> mapperClass, String methodName,
                                 List<T> dataList, int batchSize) {
        if (dataList == null || dataList.isEmpty()) return;

        try (SqlSession sqlSession = sqlSessionFactory.openSession(
                ExecutorType.BATCH, false)) {

            Object mapper = sqlSession.getMapper(mapperClass);
            int total = dataList.size();
            int batchCount = (total + batchSize - 1) / batchSize;

            for (int i = 0; i < batchCount; i++) {
                int fromIndex = i * batchSize;
                int toIndex = Math.min(fromIndex + batchSize, total);
                List<T> batchList = dataList.subList(fromIndex, toIndex);

                // 通过反射调用 mapper 方法
                Method method = mapperClass.getMethod(methodName, Object.class);
                for (T item : batchList) {
                    method.invoke(mapper, item);
                }

                // 每批提交一次
                sqlSession.flushStatements();
            }

            sqlSession.commit();
            System.out.println("批量插入完成，共 " + total + " 条");

        } catch (Exception e) {
            throw new RuntimeException("批量插入失败: " + e.getMessage(), e);
        }
    }
}
```

---

## 10.4 MyBatis 与连接池整合

### Druid 连接池完整配置

```yaml
# application.yml - Druid 完整配置
spring:
  datasource:
    type: com.alibaba.druid.pool.DruidDataSource
    druid:
      # 基础配置
      driver-class-name: com.mysql.cj.jdbc.Driver
      url: jdbc:mysql://localhost:3306/mydb?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
      username: root
      password: 123456

      # 连接池大小
      initial-size: 5        # 初始化连接数
      min-idle: 5            # 最小空闲连接
      max-active: 20         # 最大连接数

      # 获取连接等待超时
      max-wait: 60000        # 60秒

      # 检测空闲连接的间隔时间
      time-between-eviction-runs-millis: 60000

      # 连接最小空闲时间（超过则被回收）
      min-evictable-idle-time-millis: 300000

      # 验证连接的 SQL
      validation-query: SELECT 1
      validation-query-timeout: 5

      # 空闲时检测连接（推荐开启）
      test-while-idle: true
      # 获取连接时检测（影响性能，不推荐）
      test-on-borrow: false
      # 归还连接时检测（影响性能，不推荐）
      test-on-return: false

      # 开启 PSCache（提高 PreparedStatement 缓存效率）
      pool-prepared-statements: true
      max-pool-prepared-statement-per-connection-size: 20

      # 过滤器配置
      filters: stat,wall,slf4j

      # 监控统计
      filter:
        stat:
          enabled: true
          slow-sql-millis: 1000    # 慢 SQL 阈值（1秒）
          log-slow-sql: true       # 打印慢 SQL

      # Druid 监控控制台
      stat-view-servlet:
        enabled: true
        url-pattern: /druid/*
        reset-enable: false
        login-username: admin
        login-password: admin123
```

### HikariCP 连接池配置

```yaml
# application.yml - HikariCP 配置（Spring Boot 默认连接池）
spring:
  datasource:
    type: com.zaxxer.hikari.HikariDataSource
    hikari:
      # 最大连接池大小（核心 CPU 数 * 2 + 有效磁盘数）
      maximum-pool-size: 20
      # 最小空闲连接
      minimum-idle: 5
      # 空闲连接超时（毫秒），默认 600000（10分钟）
      idle-timeout: 600000
      # 连接最大存活时间（毫秒），默认 1800000（30分钟）
      max-lifetime: 1800000
      # 获取连接超时（毫秒）
      connection-timeout: 30000
      # 连接测试 SQL
      connection-test-query: SELECT 1
      # 连接池名称
      pool-name: MyHikariPool
      # 是否自动提交（默认 true）
      auto-commit: true
```

---

# Part 11: 完整实战案例 <a id="part-11"></a>

## 案例1：电商商品管理（CRUD + 动态SQL + 分页）

### 数据库表

```sql
CREATE TABLE product (
    id           BIGINT PRIMARY KEY AUTO_INCREMENT COMMENT '商品ID',
    name         VARCHAR(200) NOT NULL COMMENT '商品名称',
    category_id  BIGINT COMMENT '分类ID',
    brand        VARCHAR(100) COMMENT '品牌',
    price        DECIMAL(10,2) NOT NULL COMMENT '价格',
    stock        INT DEFAULT 0 COMMENT '库存',
    status       TINYINT DEFAULT 1 COMMENT '状态 0:下架 1:上架',
    description  TEXT COMMENT '商品描述',
    images       VARCHAR(1000) COMMENT '图片URL（逗号分隔）',
    create_time  DATETIME DEFAULT CURRENT_TIMESTAMP,
    update_time  DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted      TINYINT DEFAULT 0 COMMENT '逻辑删除',
    INDEX idx_category (category_id),
    INDEX idx_status (status),
    INDEX idx_create_time (create_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='商品表';
```

### 实体类

```java
@Data
@TableName("product")
public class Product implements Serializable {
    private static final long serialVersionUID = 1L;

    @TableId(type = IdType.AUTO)
    private Long id;
    private String name;
    private Long categoryId;
    private String brand;
    private BigDecimal price;
    private Integer stock;
    private Integer status;
    private String description;

    @TableField(typeHandler = JacksonTypeHandler.class)
    private List<String> images;  // 存储为 JSON

    @TableField(fill = FieldFill.INSERT)
    private LocalDateTime createTime;

    @TableField(fill = FieldFill.INSERT_UPDATE)
    private LocalDateTime updateTime;

    @TableLogic
    private Integer deleted;
}
```

### Mapper 接口和 XML

```java
@Mapper
public interface ProductMapper extends BaseMapper<Product> {

    // 复杂查询：联表获取分类名
    List<ProductVO> selectPageWithCategory(
        @Param("query") ProductQueryRequest query,
        @Param("offset") long offset,
        @Param("limit") int limit
    );

    long countWithCategory(@Param("query") ProductQueryRequest query);
}
```

```xml
<!-- ProductMapper.xml -->
<mapper namespace="com.example.mapper.ProductMapper">

    <sql id="queryConditions">
        <where>
            <if test="query.name != null and query.name != ''">
                <bind name="nameLike" value="'%' + query.name + '%'"/>
                AND p.name LIKE #{nameLike}
            </if>
            <if test="query.categoryId != null">
                AND p.category_id = #{query.categoryId}
            </if>
            <if test="query.brand != null and query.brand != ''">
                AND p.brand = #{query.brand}
            </if>
            <if test="query.status != null">
                AND p.status = #{query.status}
            </if>
            <if test="query.minPrice != null">
                AND p.price >= #{query.minPrice}
            </if>
            <if test="query.maxPrice != null">
                AND p.price &lt;= #{query.maxPrice}
            </if>
            <if test="query.minStock != null">
                AND p.stock >= #{query.minStock}
            </if>
            AND p.deleted = 0
        </where>
    </sql>

    <select id="selectPageWithCategory" resultType="com.example.vo.ProductVO">
        SELECT
            p.id, p.name, p.brand, p.price, p.stock, p.status,
            p.create_time,
            c.name AS category_name
        FROM product p
        LEFT JOIN category c ON p.category_id = c.id
        <include refid="queryConditions"/>
        <choose>
            <when test="query.orderBy != null and query.orderBy != ''">
                ORDER BY p.${query.orderBy}
                <choose>
                    <when test="query.orderDir != null and query.orderDir.toUpperCase() == 'ASC'">ASC</when>
                    <otherwise>DESC</otherwise>
                </choose>
            </when>
            <otherwise>ORDER BY p.create_time DESC</otherwise>
        </choose>
        LIMIT #{limit} OFFSET #{offset}
    </select>

    <select id="countWithCategory" resultType="long">
        SELECT COUNT(*)
        FROM product p
        LEFT JOIN category c ON p.category_id = c.id
        <include refid="queryConditions"/>
    </select>

</mapper>
```

### Service 层

```java
@Service
public class ProductService {

    @Autowired
    private ProductMapper productMapper;

    /**
     * 分页查询商品列表
     */
    public PageResult<ProductVO> queryPage(ProductQueryRequest request) {
        long offset = (long)(request.getPageNum() - 1) * request.getPageSize();
        long total = productMapper.countWithCategory(request);

        if (total == 0) return PageResult.empty();

        List<ProductVO> list = productMapper.selectPageWithCategory(
            request, offset, request.getPageSize());

        return new PageResult<>(total, list);
    }

    /**
     * 创建商品
     */
    @Transactional(rollbackFor = Exception.class)
    public Long createProduct(ProductCreateRequest request) {
        Product product = new Product();
        BeanUtils.copyProperties(request, product);
        product.setStatus(1);  // 默认上架

        productMapper.insert(product);
        return product.getId();
    }

    /**
     * 更新商品（只更新非null字段）
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateProduct(Long id, ProductUpdateRequest request) {
        Product product = productMapper.selectById(id);
        if (product == null) {
            throw new BusinessException("商品不存在");
        }

        Product updateProduct = new Product();
        updateProduct.setId(id);
        BeanUtils.copyProperties(request, updateProduct);

        productMapper.updateById(updateProduct);
    }

    /**
     * 批量更新商品状态
     */
    @Transactional(rollbackFor = Exception.class)
    public void batchUpdateStatus(List<Long> ids, Integer status) {
        LambdaUpdateWrapper<Product> wrapper = Wrappers.<Product>lambdaUpdate()
            .set(Product::getStatus, status)
            .in(Product::getId, ids);

        productMapper.update(null, wrapper);
    }
}
```

---

## 案例2：订单-订单项 一对多关联查询

```sql
-- 订单表
CREATE TABLE orders (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_no    VARCHAR(32) UNIQUE NOT NULL COMMENT '订单号',
    user_id     BIGINT NOT NULL COMMENT '用户ID',
    total_amount DECIMAL(10,2) NOT NULL COMMENT '总金额',
    status      TINYINT DEFAULT 1 COMMENT '状态 1:待支付 2:已支付 3:已发货 4:已完成',
    create_time DATETIME DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB;

-- 订单项表
CREATE TABLE order_item (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id    BIGINT NOT NULL COMMENT '订单ID',
    product_id  BIGINT NOT NULL COMMENT '商品ID',
    product_name VARCHAR(200) NOT NULL COMMENT '商品名称快照',
    unit_price  DECIMAL(10,2) NOT NULL COMMENT '单价快照',
    quantity    INT NOT NULL COMMENT '数量',
    subtotal    DECIMAL(10,2) NOT NULL COMMENT '小计',
    INDEX idx_order_id (order_id)
) ENGINE=InnoDB;
```

```java
// Order 实体
@Data
public class Order {
    private Long id;
    private String orderNo;
    private Long userId;
    private BigDecimal totalAmount;
    private Integer status;
    private LocalDateTime createTime;
    private List<OrderItem> items;  // 订单项列表（一对多）
}

// OrderItem 实体
@Data
public class OrderItem {
    private Long id;
    private Long orderId;
    private Long productId;
    private String productName;
    private BigDecimal unitPrice;
    private Integer quantity;
    private BigDecimal subtotal;
}
```

```xml
<!-- OrderMapper.xml -->
<mapper namespace="com.example.mapper.OrderMapper">

    <resultMap id="orderWithItemsMap" type="Order">
        <id property="id" column="o_id"/>
        <result property="orderNo" column="o_order_no"/>
        <result property="userId" column="o_user_id"/>
        <result property="totalAmount" column="o_total_amount"/>
        <result property="status" column="o_status"/>
        <result property="createTime" column="o_create_time"/>

        <collection property="items" ofType="OrderItem" columnPrefix="i_">
            <id property="id" column="id"/>
            <result property="orderId" column="order_id"/>
            <result property="productId" column="product_id"/>
            <result property="productName" column="product_name"/>
            <result property="unitPrice" column="unit_price"/>
            <result property="quantity" column="quantity"/>
            <result property="subtotal" column="subtotal"/>
        </collection>
    </resultMap>

    <!-- 查询订单及所有订单项（一条 SQL）-->
    <select id="selectOrderWithItems" resultMap="orderWithItemsMap">
        SELECT
            o.id         AS o_id,
            o.order_no   AS o_order_no,
            o.user_id    AS o_user_id,
            o.total_amount AS o_total_amount,
            o.status     AS o_status,
            o.create_time AS o_create_time,
            i.id         AS i_id,
            i.order_id   AS i_order_id,
            i.product_id AS i_product_id,
            i.product_name AS i_product_name,
            i.unit_price AS i_unit_price,
            i.quantity   AS i_quantity,
            i.subtotal   AS i_subtotal
        FROM orders o
        LEFT JOIN order_item i ON o.id = i.order_id
        WHERE o.id = #{orderId}
        ORDER BY i.id ASC
    </select>

    <!-- 批量查询用户的所有订单（包含订单项）-->
    <select id="selectOrdersByUserId" resultMap="orderWithItemsMap">
        SELECT
            o.id         AS o_id,
            o.order_no   AS o_order_no,
            o.user_id    AS o_user_id,
            o.total_amount AS o_total_amount,
            o.status     AS o_status,
            o.create_time AS o_create_time,
            i.id         AS i_id,
            i.order_id   AS i_order_id,
            i.product_name AS i_product_name,
            i.unit_price AS i_unit_price,
            i.quantity   AS i_quantity,
            i.subtotal   AS i_subtotal
        FROM orders o
        LEFT JOIN order_item i ON o.id = i.order_id
        WHERE o.user_id = #{userId}
        ORDER BY o.create_time DESC, i.id ASC
    </select>

    <!-- 创建订单（包含订单项）-->
    <insert id="insertOrder" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO orders (order_no, user_id, total_amount, status)
        VALUES (#{orderNo}, #{userId}, #{totalAmount}, #{status})
    </insert>

    <insert id="insertOrderItems">
        INSERT INTO order_item (order_id, product_id, product_name, unit_price, quantity, subtotal)
        VALUES
        <foreach collection="items" item="item" separator=",">
            (#{item.orderId}, #{item.productId}, #{item.productName},
             #{item.unitPrice}, #{item.quantity}, #{item.subtotal})
        </foreach>
    </insert>

</mapper>
```

---

## 案例3：用户-角色 多对多关联查询

```sql
-- 用户表
CREATE TABLE sys_user (
    id       BIGINT PRIMARY KEY AUTO_INCREMENT,
    username VARCHAR(50) NOT NULL UNIQUE,
    email    VARCHAR(100)
);

-- 角色表
CREATE TABLE sys_role (
    id          BIGINT PRIMARY KEY AUTO_INCREMENT,
    role_name   VARCHAR(50) NOT NULL,
    role_key    VARCHAR(50) NOT NULL UNIQUE COMMENT '角色标识符',
    description VARCHAR(200)
);

-- 用户-角色中间表
CREATE TABLE sys_user_role (
    user_id BIGINT NOT NULL,
    role_id BIGINT NOT NULL,
    PRIMARY KEY (user_id, role_id),
    INDEX idx_role_id (role_id)
);
```

```java
// User 实体
@Data
public class SysUser {
    private Long id;
    private String username;
    private String email;
    private List<SysRole> roles;  // 多对多：用户的所有角色
}

// Role 实体
@Data
public class SysRole {
    private Long id;
    private String roleName;
    private String roleKey;
    private String description;
    private List<SysUser> users;  // 多对多：角色下的所有用户
}
```

```xml
<!-- SysUserMapper.xml -->
<mapper namespace="com.example.mapper.SysUserMapper">

    <!-- 用户包含角色的 resultMap -->
    <resultMap id="userWithRolesMap" type="SysUser">
        <id property="id" column="u_id"/>
        <result property="username" column="u_username"/>
        <result property="email" column="u_email"/>

        <collection property="roles" ofType="SysRole" columnPrefix="r_">
            <id property="id" column="id"/>
            <result property="roleName" column="role_name"/>
            <result property="roleKey" column="role_key"/>
            <result property="description" column="description"/>
        </collection>
    </resultMap>

    <!-- 查询用户及其所有角色 -->
    <select id="selectUserWithRoles" resultMap="userWithRolesMap">
        SELECT
            u.id        AS u_id,
            u.username  AS u_username,
            u.email     AS u_email,
            r.id        AS r_id,
            r.role_name AS r_role_name,
            r.role_key  AS r_role_key,
            r.description AS r_description
        FROM sys_user u
        LEFT JOIN sys_user_role ur ON u.id = ur.user_id
        LEFT JOIN sys_role r ON ur.role_id = r.id
        WHERE u.id = #{userId}
    </select>

    <!-- 为用户分配角色（批量插入中间表）-->
    <insert id="assignRoles">
        INSERT INTO sys_user_role (user_id, role_id) VALUES
        <foreach collection="roleIds" item="roleId" separator=",">
            (#{userId}, #{roleId})
        </foreach>
    </insert>

    <!-- 删除用户的所有角色 -->
    <delete id="deleteUserRoles">
        DELETE FROM sys_user_role WHERE user_id = #{userId}
    </delete>

    <!-- 查询拥有某角色的所有用户 -->
    <select id="selectUsersByRoleKey" resultType="SysUser">
        SELECT u.id, u.username, u.email
        FROM sys_user u
        INNER JOIN sys_user_role ur ON u.id = ur.user_id
        INNER JOIN sys_role r ON ur.role_id = r.id
        WHERE r.role_key = #{roleKey}
        ORDER BY u.id
    </select>

</mapper>
```

```java
// Service 层
@Service
public class SysUserService {

    @Autowired
    private SysUserMapper userMapper;

    /**
     * 更新用户角色（先删后插）
     */
    @Transactional(rollbackFor = Exception.class)
    public void updateUserRoles(Long userId, List<Long> roleIds) {
        // 1. 删除原有角色关系
        userMapper.deleteUserRoles(userId);

        // 2. 如果有新角色，批量插入
        if (roleIds != null && !roleIds.isEmpty()) {
            userMapper.assignRoles(userId, roleIds);
        }
    }
}
```

---

## 案例4：通用 Repository 封装（BaseMapper + ServiceImpl 模式）

```java
// 通用响应结果
@Data
@AllArgsConstructor
@NoArgsConstructor
public class PageResult<T> {
    private long total;          // 总记录数
    private List<T> list;        // 数据列表
    private int pageNum;         // 当前页
    private int pageSize;        // 每页大小
    private int totalPages;      // 总页数

    public PageResult(long total, List<T> list) {
        this.total = total;
        this.list = list;
    }

    public static <T> PageResult<T> empty() {
        return new PageResult<>(0L, Collections.emptyList());
    }
}

// 通用查询请求基类
@Data
public abstract class BasePageRequest {
    private Integer pageNum = 1;
    private Integer pageSize = 10;
    private String orderBy;
    private String orderDir = "DESC";

    public long getOffset() {
        return (long)(pageNum - 1) * pageSize;
    }
}

// 通用 Mapper 基类（集成 MyBatis-Plus BaseMapper）
public interface CrudMapper<T> extends BaseMapper<T> {
    // 可以添加通用的自定义方法
}

// 通用 Service 基类
public abstract class CrudService<M extends CrudMapper<T>, T> extends ServiceImpl<M, T> {

    /**
     * 通用分页查询
     */
    public PageResult<T> pageQuery(BasePageRequest request, LambdaQueryWrapper<T> wrapper) {
        Page<T> page = new Page<>(request.getPageNum(), request.getPageSize());
        IPage<T> result = this.page(page, wrapper);
        return new PageResult<>(result.getTotal(), result.getRecords(),
                                result.getCurrent(), (int)result.getSize(),
                                (int)result.getPages());
    }

    /**
     * 批量保存（超过阈值使用批量插入）
     */
    public boolean saveBatchOptimized(List<T> list) {
        if (list == null || list.isEmpty()) return true;
        if (list.size() <= 100) {
            return this.saveBatch(list, list.size());
        }
        return this.saveBatch(list, 500);
    }
}
```

---

## 案例5：数据权限（部门树查询 + 行级权限）

```sql
-- 部门表（树形结构）
CREATE TABLE dept (
    id        BIGINT PRIMARY KEY AUTO_INCREMENT,
    parent_id BIGINT DEFAULT 0 COMMENT '父部门ID，0为根节点',
    name      VARCHAR(100) NOT NULL COMMENT '部门名称',
    ancestors VARCHAR(500) COMMENT '祖级列表（逗号分隔的部门ID）',
    sort      INT DEFAULT 0 COMMENT '排序',
    deleted   TINYINT DEFAULT 0
);
```

```java
@Mapper
public interface DeptMapper extends BaseMapper<Dept> {

    /**
     * 递归查询部门及所有子部门 ID（MySQL 递归 CTE）
     */
    @Select("""
        WITH RECURSIVE dept_tree AS (
            SELECT id, parent_id, name
            FROM dept WHERE id = #{deptId} AND deleted = 0
            UNION ALL
            SELECT d.id, d.parent_id, d.name
            FROM dept d
            INNER JOIN dept_tree dt ON d.parent_id = dt.id
            WHERE d.deleted = 0
        )
        SELECT id FROM dept_tree
        """)
    List<Long> selectDeptAndChildIds(Long deptId);

    /**
     * 查询完整部门树
     */
    List<DeptTreeVO> selectDeptTree();
}
```

```xml
<!-- DeptMapper.xml -->
<select id="selectDeptTree" resultType="com.example.vo.DeptTreeVO">
    SELECT id, parent_id, name, sort
    FROM dept
    WHERE deleted = 0
    ORDER BY parent_id ASC, sort ASC, id ASC
</select>
```

```java
// Service 层：构建部门树并进行数据权限过滤
@Service
public class DeptService {

    @Autowired
    private DeptMapper deptMapper;

    /**
     * 获取当前用户有权限的所有部门ID
     */
    public List<Long> getPermittedDeptIds(Long currentUserDeptId,
                                           String dataScopeType) {
        switch (dataScopeType) {
            case "SELF":
                return Collections.emptyList();  // 只看自己，不用部门限制

            case "DEPT":
                return Collections.singletonList(currentUserDeptId);

            case "DEPT_AND_BELOW":
                // 本部门及所有子部门
                return deptMapper.selectDeptAndChildIds(currentUserDeptId);

            case "ALL":
            default:
                return null;  // null 表示无限制
        }
    }

    /**
     * 将扁平列表构建为树形结构
     */
    public List<DeptTreeVO> buildDeptTree(List<DeptTreeVO> flatList) {
        Map<Long, DeptTreeVO> nodeMap = flatList.stream()
            .collect(Collectors.toMap(DeptTreeVO::getId, v -> v));

        List<DeptTreeVO> roots = new ArrayList<>();

        for (DeptTreeVO node : flatList) {
            if (node.getParentId() == 0L) {
                roots.add(node);
            } else {
                DeptTreeVO parent = nodeMap.get(node.getParentId());
                if (parent != null) {
                    if (parent.getChildren() == null) {
                        parent.setChildren(new ArrayList<>());
                    }
                    parent.getChildren().add(node);
                }
            }
        }

        return roots;
    }
}
```

---

# Part 12: 性能优化 <a id="part-12"></a>

## 12.1 N+1 查询问题及解决方案

N+1 问题是 ORM 框架中最常见的性能陷阱之一。

```
N+1 问题示例：
  查询 N 个用户 -> 执行 1 条 SQL
  对每个用户查询其订单 -> 执行 N 条 SQL
  总共执行 N+1 条 SQL，当 N=100 时，就是 101 次数据库查询！

场景复现：
  使用嵌套查询（子查询方式）的 collection 映射，且未开启懒加载：
  
  第1条 SQL: SELECT id, username FROM user LIMIT 100
  第2条 SQL: SELECT * FROM orders WHERE user_id = 1
  第3条 SQL: SELECT * FROM orders WHERE user_id = 2
  ...
  第101条 SQL: SELECT * FROM orders WHERE user_id = 100
```

**解决方案1：使用 JOIN 联表查询（最推荐）**

```xml
<!-- 一条 SQL 完成所有数据查询 -->
<resultMap id="userWithOrdersMap" type="User">
    <id property="id" column="u_id"/>
    <result property="username" column="username"/>
    <collection property="orders" ofType="Order" columnPrefix="o_">
        <id property="id" column="id"/>
        <result property="orderNo" column="order_no"/>
        <result property="amount" column="amount"/>
    </collection>
</resultMap>

<select id="selectUsersWithOrders" resultMap="userWithOrdersMap">
    SELECT u.id AS u_id, u.username,
           o.id AS o_id, o.order_no AS o_order_no, o.amount AS o_amount
    FROM user u
    LEFT JOIN orders o ON u.id = o.user_id
    WHERE u.status = 1
    ORDER BY u.id, o.create_time DESC
</select>
```

**解决方案2：懒加载（适合大多数情况不需要关联数据的场景）**

```xml
<!-- 懒加载：只在真正需要 orders 时才查询 -->
<resultMap id="userLazyMap" type="User">
    <id property="id" column="id"/>
    <result property="username" column="username"/>
    <collection property="orders"
                column="id"
                select="selectOrdersByUserId"
                fetchType="lazy"/>  <!-- 懒加载 -->
</resultMap>
```

**解决方案3：批量查询 + 内存聚合（适合需要分页的场景）**

```java
@Service
public class UserService {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 批量查询 + 内存聚合，解决分页场景下的 N+1 问题
     */
    @Transactional(readOnly = true)
    public List<UserWithOrdersVO> getUsersWithOrders(int page, int size) {
        // 1. 分页查询用户（1条SQL）
        Page<User> pageResult = userMapper.selectPage(
            new Page<>(page, size),
            Wrappers.<User>lambdaQuery().eq(User::getStatus, 1)
        );
        List<User> users = pageResult.getRecords();

        if (users.isEmpty()) return Collections.emptyList();

        // 2. 收集所有用户ID
        List<Long> userIds = users.stream().map(User::getId).collect(Collectors.toList());

        // 3. 一次性批量查询所有用户的订单（1条SQL）
        List<Order> allOrders = orderMapper.selectList(
            Wrappers.<Order>lambdaQuery()
                .in(Order::getUserId, userIds)
                .orderByDesc(Order::getCreateTime)
        );

        // 4. 在内存中按 userId 分组
        Map<Long, List<Order>> ordersByUserId = allOrders.stream()
            .collect(Collectors.groupingBy(Order::getUserId));

        // 5. 组装结果
        return users.stream().map(user -> {
            UserWithOrdersVO vo = new UserWithOrdersVO();
            BeanUtils.copyProperties(user, vo);
            vo.setOrders(ordersByUserId.getOrDefault(user.getId(), Collections.emptyList()));
            return vo;
        }).collect(Collectors.toList());
    }
}
```

---

## 12.2 批量插入性能对比

```java
@SpringBootTest
public class BatchInsertPerformanceTest {

    @Autowired
    private UserMapper userMapper;

    @Autowired
    private SqlSessionFactory sqlSessionFactory;

    private List<User> buildTestData(int size) {
        List<User> users = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            users.add(User.builder()
                .username("user" + i)
                .email("user" + i + "@test.com")
                .status(1)
                .build());
        }
        return users;
    }

    /**
     * 方式1：单条循环插入
     * 结果：10000条 ≈ 15-30秒（取决于网络延迟）
     */
    @Test
    @Transactional
    public void testSingleInsert() {
        List<User> users = buildTestData(10000);
        long start = System.currentTimeMillis();

        for (User user : users) {
            userMapper.insert(user);
        }

        System.out.println("单条插入耗时: " + (System.currentTimeMillis() - start) + "ms");
    }

    /**
     * 方式2：foreach 批量 INSERT（一条 SQL）
     * 结果：10000条 ≈ 0.5-2秒
     * 注意：MySQL 默认 max_allowed_packet=4MB，数据量太大需要分批
     */
    @Test
    @Transactional
    public void testForeachBatchInsert() {
        List<User> users = buildTestData(10000);
        long start = System.currentTimeMillis();

        // 分批插入，每批500条
        int batchSize = 500;
        for (int i = 0; i < users.size(); i += batchSize) {
            List<User> batch = users.subList(i, Math.min(i + batchSize, users.size()));
            userMapper.insertBatch(batch);
        }

        System.out.println("foreach批量插入耗时: " + (System.currentTimeMillis() - start) + "ms");
    }

    /**
     * 方式3：BatchExecutor（JDBC batch）
     * 结果：10000条 ≈ 0.3-1秒
     */
    @Test
    public void testBatchExecutorInsert() {
        List<User> users = buildTestData(10000);
        long start = System.currentTimeMillis();

        try (SqlSession session = sqlSessionFactory.openSession(ExecutorType.BATCH, false)) {
            UserMapper mapper = session.getMapper(UserMapper.class);

            for (int i = 0; i < users.size(); i++) {
                mapper.insert(users.get(i));
                if (i > 0 && i % 500 == 0) {
                    session.flushStatements();
                }
            }

            session.flushStatements();
            session.commit();
        }

        System.out.println("BatchExecutor插入耗时: " + (System.currentTimeMillis() - start) + "ms");
    }

    /**
     * 方式4：MyBatis-Plus 的 saveBatch（底层也是 BatchExecutor）
     * 结果：10000条 ≈ 0.5-1.5秒
     */
    @Test
    @Transactional
    public void testMPSaveBatch() {
        List<User> users = buildTestData(10000);
        long start = System.currentTimeMillis();

        // MyBatis-Plus 自动分批（默认1000条/批）
        userService.saveBatch(users, 500);

        System.out.println("MP saveBatch耗时: " + (System.currentTimeMillis() - start) + "ms");
    }
}
```

**性能对比汇总：**

```
插入10000条数据性能对比：

方式               | 网络往返  | 耗时估计 | 适用场景
------------------+-----------+----------+------------------
单条插入（无事务） | 10000次   | 30-60秒  | 不推荐
单条插入（有事务） | 10000次   | 5-15秒   | 数据量极少
foreach批量INSERT | 分批次    | 0.5-2秒  | 数据量中等（推荐）
BatchExecutor     | 分批次    | 0.3-1秒  | 大批量数据（最快）
MP saveBatch      | 分批次    | 0.5-1.5秒| 与 BatchExecutor 相似

推荐策略：
  < 100条   : 普通单条插入或 MP 的 save/saveBatch
  100-5000条 : foreach 批量 INSERT（分批，每批500条）
  > 5000条   : BatchExecutor + 分批（每批500条）
```

---

## 12.3 连接池参数调优

```
连接池调优原则：

最大连接数（max-active/maximum-pool-size）：
  - 公式参考：核心 CPU 数量 × 2 + 有效磁盘数量
  - 对于 IO 密集型（数据库查询）：CPU核数 * 2
  - 对于 CPU 密集型：CPU核数 + 1
  - 生产环境通常设置 20-50
  - 注意：连接数不是越多越好，过多反而增加切换开销！

最小空闲连接（min-idle/minimum-idle）：
  - 设置为 max-active 的 10%-25%
  - 保持一定数量的空闲连接，减少创建连接的延迟

获取连接超时（max-wait/connection-timeout）：
  - HikariCP: 30000ms（30秒）
  - Druid: 60000ms（60秒）
  - 不要设置过大，否则请求会积压

连接有效性检测（validation-query）：
  - MySQL: SELECT 1
  - Oracle: SELECT 1 FROM DUAL
  - 建议开启 test-while-idle，不建议 test-on-borrow（影响性能）

连接生命周期（max-lifetime）：
  - HikariCP 默认 1800000ms（30分钟）
  - 必须小于数据库的 wait_timeout（MySQL默认8小时）
  - 推荐设置为数据库 wait_timeout 的 50%-75%
```

```yaml
# 生产环境推荐的 HikariCP 配置
spring:
  datasource:
    hikari:
      maximum-pool-size: 20         # 根据服务器 CPU 核数调整
      minimum-idle: 5
      connection-timeout: 30000     # 30秒获取不到连接则报错
      idle-timeout: 600000          # 10分钟空闲超时
      max-lifetime: 1800000         # 30分钟连接最大生命周期
      keepalive-time: 60000         # 60秒保活检测
      connection-test-query: SELECT 1
      pool-name: MainPool
      # 重要：leak-detection-threshold 连接泄露检测
      leak-detection-threshold: 60000  # 60秒内未归还则打印警告日志
```

---

## 12.4 MyBatis 二级缓存使用建议

```
二级缓存使用建议（总结）：

适合使用二级缓存的场景：
  1. 数据极少变化的参数表（如省市区、字典表）
  2. 单表查询，不涉及关联
  3. 对数据实时性要求不高的场景
  4. 读多写少的场景

不适合使用二级缓存的场景：
  1. 多表关联查询（容易产生脏读）
  2. 对数据实时性要求高的场景（如订单、库存）
  3. 写操作频繁的场景（每次写都会清空整个 namespace 的缓存）
  4. 分布式环境（默认 JVM 内存缓存，不同节点不共享）

生产环境推荐：
  1. 关闭 MyBatis 二级缓存
  2. 使用 Spring Cache + Redis 实现分布式缓存
  3. 使用 @Cacheable / @CacheEvict 注解管理缓存

配置示例：
  # 关闭 MyBatis 二级缓存
  mybatis:
    configuration:
      cache-enabled: false  # 关闭
```

---

# Part 13: 常见面试题 FAQ <a id="part-13"></a>

## Q1: MyBatis 中 #{} 和 ${} 的区别？

**答：**

**`#{}`（预编译参数）：**
- 将参数替换为 JDBC 的 `?` 占位符，使用 `PreparedStatement` 预编译
- 通过 `TypeHandler` 安全地设置参数值
- **防止 SQL 注入**（最重要的区别）
- 适合所有参数值的传递（字符串、数字、日期等）

**`${}`（字符串替换）：**
- 直接将变量值拼接到 SQL 字符串中
- **存在 SQL 注入风险**
- 不经过预编译
- 适合：动态表名、列名、ORDER BY 子句等无法用 `?` 表示的场景
- 使用时必须配合白名单验证！

```sql
-- #{} 生成的 SQL
SELECT * FROM user WHERE name = ?   -- 安全
-- ${} 生成的 SQL
SELECT * FROM user WHERE name = '张三'  -- 直接拼接
```

---

## Q2: MyBatis 一级缓存和二级缓存的区别？

**答：**

| 特性 | 一级缓存 | 二级缓存 |
|------|---------|---------|
| 作用域 | SqlSession 级别 | Mapper namespace 级别 |
| 共享范围 | 同一个 SqlSession | 跨 SqlSession 共享 |
| 默认状态 | 默认开启，无法关闭 | 默认关闭，需手动开启 |
| 存储位置 | BaseExecutor.localCache | 堆内存或外部存储 |
| 返回对象 | 同一引用（==成立）| 反序列化副本（==不成立）|
| 失效条件 | DML/手动清除/不同Session | DML/close前未提交 |
| 实体要求 | 无 | 必须实现 Serializable |
| Spring中 | 无事务时自动失效 | 需手动配置 |

**查询顺序：** 二级缓存 -> 一级缓存 -> 数据库

---

## Q3: MyBatis 为什么在 Spring 中一级缓存失效？

**答：**

Spring 使用 `SqlSessionTemplate` 代理 `SqlSession`。当方法没有 `@Transactional` 注解时，每次 Mapper 方法调用都会：
1. 创建新的 `SqlSession`
2. 执行 SQL
3. 立即关闭 `SqlSession`

因此无法复用一级缓存。

**解决方案：**
- 在需要利用一级缓存的方法上添加 `@Transactional`
- 或者开启二级缓存
- 或者使用 Redis 外部缓存

---

## Q4: MyBatis 四大核心对象（拦截器可以拦截哪些对象）？

**答：**

MyBatis 插件可以拦截的四大对象：

1. **Executor**（执行器）：执行增删改查的核心对象
   - `update()` - 处理 INSERT/UPDATE/DELETE
   - `query()` - 处理 SELECT

2. **ParameterHandler**（参数处理器）：将 Java 参数设置到 PreparedStatement
   - `setParameters()` - 参数绑定

3. **ResultSetHandler**（结果集处理器）：将 ResultSet 映射为 Java 对象
   - `handleResultSets()` - 结果集映射

4. **StatementHandler**（语句处理器）：创建和执行 SQL Statement
   - `prepare()` - 准备 Statement
   - `parameterize()` - 设置参数
   - `query()/update()` - 执行 SQL

---

## Q5: MyBatis 的动态 SQL 标签有哪些？分别用于什么场景？

**答：**

| 标签 | 用途 | 示例场景 |
|------|------|---------|
| `<if>` | 条件判断 | 可选的查询条件 |
| `<choose>/<when>/<otherwise>` | 多分支选择（如switch）| 按优先级选择查询方式 |
| `<where>` | 自动处理 AND/OR 前缀 | 多条件查询 |
| `<set>` | 自动处理末尾逗号 | 选择性更新 |
| `<trim>` | 自定义前后缀处理 | 通用场景（where和set的底层）|
| `<foreach>` | 遍历集合 | IN查询、批量插入 |
| `<bind>` | 创建变量 | 模糊查询的 `%` 拼接 |
| `<sql>` | 定义SQL片段 | 提取公共列名、条件等 |
| `<include>` | 引用SQL片段 | 复用 `<sql>` 片段 |

---

## Q6: resultMap 和 resultType 的区别是什么？什么时候用哪个？

**答：**

**resultType：**
- 适合列名与 Java 属性名一致的简单映射
- 自动映射（配合 `mapUnderscoreToCamelCase=true` 更好用）
- 返回 POJO、基本类型、Map 时使用

**resultMap：**
- 适合复杂映射场景：
  - 列名与属性名不一致
  - 一对一关联（`<association>`）
  - 一对多关联（`<collection>`）
  - 鉴别器（`<discriminator>`）
  - 自定义 TypeHandler

**原则：** 能用 resultType 就用 resultType，需要复杂映射时才用 resultMap。

---

## Q7: MyBatis 插件的实现原理是什么？

**答：**

MyBatis 插件基于 **JDK 动态代理 + 责任链模式** 实现：

1. 每个插件实现 `Interceptor` 接口，使用 `@Intercepts/@Signature` 声明拦截点

2. MyBatis 创建四大核心对象时，通过 `InterceptorChain.pluginAll()` 将对象逐一传入每个插件的 `plugin()` 方法

3. `Plugin.wrap()` 方法判断目标对象是否需要被拦截，如果是，使用 `Proxy.newProxyInstance()` 创建 JDK 动态代理

4. 多个插件形成责任链，最后注册的插件最先执行

5. 当调用被拦截的方法时，触发 `InvocationHandler.invoke()`，调用插件的 `intercept()` 方法

---

## Q8: MyBatis 如何实现 Mapper 接口的？（不需要写实现类的原因）

**答：**

MyBatis 使用 **JDK 动态代理** 为每个 Mapper 接口创建代理实例：

1. `MapperRegistry` 管理 Mapper 接口与 `MapperProxyFactory` 的映射

2. 调用 `sqlSession.getMapper(UserMapper.class)` 时，`MapperProxyFactory` 通过 `Proxy.newProxyInstance()` 创建代理对象

3. 代理对象的 `InvocationHandler` 是 `MapperProxy`

4. 当调用 `userMapper.selectById(1L)` 时：
   - `MapperProxy.invoke()` 被调用
   - 根据方法名找到对应的 `MappedStatement`
   - 确定 SQL 命令类型（SELECT/INSERT/UPDATE/DELETE）
   - 调用 `sqlSession.selectOne/insert/update/delete()`

---

## Q9: MyBatis 的 Executor 有哪几种？分别有什么特点？

**答：**

| Executor | 特点 | 适用场景 |
|---------|------|---------|
| SimpleExecutor | 每次新建 Statement，执行完关闭 | 默认，普通 CRUD |
| ReuseExecutor | 缓存相同 SQL 的 PreparedStatement，复用 | 同 Session 内多次相同 SQL |
| BatchExecutor | addBatch/executeBatch，批量执行 | 大批量 DML 操作 |
| CachingExecutor | 装饰者，为上三种添加二级缓存 | cacheEnabled=true 时自动使用 |

---

## Q10: MyBatis 与 Hibernate 的区别？什么场景选哪个？

**答：**

| 维度 | MyBatis | Hibernate |
|------|---------|-----------|
| SQL 控制 | 手写 SQL（完全控制）| 自动生成 SQL |
| 学习曲线 | 较平 | 较陡（需要 HQL、映射关系等）|
| 灵活性 | 高（复杂 SQL 友好）| 低（复杂 SQL 困难）|
| 性能 | 高（SQL 可优化）| 中（生成 SQL 不可控）|
| 数据库移植性 | 低 | 高 |
| N+1 问题 | 需注意（嵌套查询）| 更容易遇到 |
| 国内使用 | 极广 | 较少 |

**选型：**
- 复杂 SQL、互联网/金融系统 -> **MyBatis**
- 快速开发、CRUD 简单系统 -> **Hibernate/JPA**
- Spring Boot 新项目 -> **MyBatis-Plus**（推荐）

---

## Q11: MyBatis-Plus 的乐观锁实现原理？

**答：**

1. 在实体类的版本字段上加 `@Version` 注解

2. 配置 `OptimisticLockerInnerInterceptor` 插件

3. 执行 UPDATE 时，插件自动在 WHERE 条件追加版本号检查：
   ```sql
   UPDATE product
   SET stock = ?, version = version + 1
   WHERE id = ? AND version = ?  -- 乐观锁条件
   ```

4. 如果 `UPDATE` 影响行数为 0，说明版本号不匹配（有并发更新），需要重试

---

## Q12: MyBatis-Plus 的逻辑删除是如何工作的？

**答：**

配置了 `@TableLogic` 注解后：
- **查询**：自动追加 `WHERE deleted = 0`
- **删除**：执行 `UPDATE SET deleted = 1` 而非真正的 `DELETE`
- **插入**：插入时自动设置 `deleted = 0`
- **注意**：使用 `@TableLogic` 后，无法通过 BaseMapper 查询已删除的数据，需要手写 SQL（绕过逻辑删除）

---

## Q13: 如何防止 MyBatis 的 SQL 注入？

**答：**

1. **优先使用 `#{}`**：参数通过 PreparedStatement 预编译，无法注入

2. **使用 `${}` 时必须白名单验证**：
   ```java
   Set<String> allowedColumns = Set.of("id", "name", "create_time");
   if (!allowedColumns.contains(column)) throw new IllegalArgumentException("...");
   ```

3. **避免直接拼接用户输入**：绝不要将前端传入的字符串直接用 `${}` 拼接

4. **使用 `<if>` 条件过滤**：对传入的参数进行合法性校验

---

## Q14: MyBatis 中 association 和 collection 的区别？

**答：**

| 特性 | association | collection |
|------|------------|------------|
| 关系类型 | 一对一（has one）| 一对多（has many）|
| 属性类型 | 单个对象 | 集合（List 等）|
| 配置项 | `javaType` 指定关联类型 | `ofType` 指定集合元素类型 |
| 场景 | 用户-详情、订单-地址 | 用户-订单、部门-员工 |

两者都支持：
- 嵌套结果（JOIN 查询，一条 SQL）
- 嵌套查询（子查询，支持懒加载）
- `columnPrefix` 避免列名冲突
- `fetchType` 覆盖全局懒加载配置

---

## Q15: MyBatis 中如何获取数据库自增主键？

**答：**

```xml
<!-- 方式1：useGeneratedKeys（推荐，MySQL/H2/SQL Server）-->
<insert id="insert"
        useGeneratedKeys="true"   <!-- 开启自动获取主键 -->
        keyProperty="id"          <!-- 回填到实体的 id 属性 -->
        keyColumn="id">           <!-- 数据库列名（可省略）-->
    INSERT INTO user (username) VALUES (#{username})
</insert>

<!-- 方式2：selectKey（Oracle 序列或 MySQL 的 LAST_INSERT_ID）-->
<insert id="insert">
    <selectKey keyProperty="id" resultType="Long" order="AFTER">
        SELECT LAST_INSERT_ID()
    </selectKey>
    INSERT INTO user (username) VALUES (#{username})
</insert>
```

调用后，`user.getId()` 会自动返回数据库生成的主键值。

---

## Q16: MyBatis 分页查询的实现方式有哪些？

**答：**

1. **RowBounds（物理分页的"逻辑分页"，不推荐）**：
   - 全量查询后在内存中截取
   - 性能差，不适合大数据量

2. **手动 LIMIT 分页（推荐）**：
   ```xml
   <select id="selectPage">
       SELECT * FROM user LIMIT #{pageSize} OFFSET #{offset}
   </select>
   ```

3. **PageHelper 插件（推荐）**：
   ```java
   PageHelper.startPage(1, 10);
   List<User> users = userMapper.selectAll();
   PageInfo<User> pageInfo = new PageInfo<>(users);
   ```

4. **MyBatis-Plus 分页（推荐）**：
   ```java
   Page<User> page = new Page<>(1, 10);
   IPage<User> result = userMapper.selectPage(page, wrapper);
   ```

---

## Q17: @Transactional 注解在 MyBatis 中的作用？

**答：**

`@Transactional` 在 MyBatis + Spring 整合中至关重要：

1. **保证一级缓存有效**：事务内的多次相同查询共享同一 SqlSession，一级缓存生效

2. **保证多个 Mapper 操作的原子性**：多个 Mapper 操作使用同一个数据库连接和事务

3. **工作原理**：
   - Spring AOP 开启事务
   - 创建数据库连接，`connection.setAutoCommit(false)`
   - 将连接绑定到当前线程
   - 所有 Mapper 操作复用这个连接
   - 方法结束：commit 或 rollback

---

## Q18: MyBatis 的 TypeHandler 有什么用？如何自定义？

**答：**

**TypeHandler 的作用：** 负责 Java 类型与 JDBC 类型之间的相互转换。

**内置 TypeHandler 举例：**
- `StringTypeHandler`：String <-> VARCHAR
- `DateTypeHandler`：Date <-> TIMESTAMP
- `EnumTypeHandler`：Enum <-> VARCHAR（存枚举名）

**自定义 TypeHandler 步骤：**
1. 继承 `BaseTypeHandler<T>`
2. 实现 4 个方法：
   - `setNonNullParameter()` - 写入数据库
   - `getNullableResult(ResultSet, String)` - 按列名读取
   - `getNullableResult(ResultSet, int)` - 按列序号读取
   - `getNullableResult(CallableStatement, int)` - 存储过程读取
3. 使用 `@MappedTypes` 和 `@MappedJdbcTypes` 注解声明处理的类型
4. 在配置文件中注册

**常见自定义场景：**
- `List<String>` <-> 逗号分隔的 VARCHAR
- 枚举 <-> 整数（存 code 而非枚举名）
- JSON 对象 <-> VARCHAR（存 JSON 字符串）

---

## Q19: MyBatis 二级缓存为什么会导致脏读？如何解决？

**答：**

**脏读原因：**
- 二级缓存的作用域是 Mapper 的 namespace
- 如果 Mapper A 查询了关联表 B 的数据（JOIN 查询），结果被缓存在 Mapper A 的缓存中
- 当 Mapper B 的数据被更新时，只会清空 Mapper B 的缓存
- Mapper A 的缓存不会被清空，导致返回旧数据（脏读）

**解决方案：**
1. 关联表查询不使用二级缓存（`useCache="false"`）
2. 使用 `<cache-ref>` 共享缓存 namespace
3. 关闭 MyBatis 二级缓存，使用 Spring Cache + Redis（最推荐）

---

## Q20: MyBatis 如何处理批量插入？forEach 和 BatchExecutor 有什么区别？

**答：**

**foreach 批量 INSERT：**
- 生成一条包含多个 VALUES 的 INSERT 语句
- `INSERT INTO user VALUES (...), (...), ...`
- 简单易用，一条 SQL 完成
- 数据量过大时 SQL 超过 `max_allowed_packet` 限制需分批

**BatchExecutor：**
- 使用 JDBC 的 `addBatch()` + `executeBatch()` 机制
- 在内存中攒多条 SQL，一次性提交到数据库
- 网络往返次数最少，性能最优
- 需要手动 `flushStatements()` 才能真正执行

**选择建议：**
- 数据量 < 1000 条：foreach 批量 INSERT（简单）
- 数据量 > 1000 条：BatchExecutor（性能最优）
- 需要一条 SQL 完成：foreach 批量 INSERT
- 需要最高性能：BatchExecutor

---

## Q21: MyBatis-Plus 的 Wrapper 有哪几种？各有什么特点？

**答：**

| Wrapper | 特点 | 使用场景 |
|---------|------|---------|
| QueryWrapper | 字符串字段名，灵活 | 普通查询，字段名确定不变时 |
| LambdaQueryWrapper | 方法引用，编译期检查 | 推荐，重构安全 |
| UpdateWrapper | 字符串字段名，支持 SET 子句 | 指定字段更新 |
| LambdaUpdateWrapper | 方法引用 + SET 子句 | 推荐，重构安全 |

**LambdaQueryWrapper 的优势：**
- 使用方法引用（`User::getUsername`）代替字段名字符串（`"username"`）
- 编译器会检查属性是否存在
- 重命名字段时，编译器会给出错误提示，而不是运行时才发现

---

## Q22: 如何在 MyBatis 中实现多租户隔离？

**答：**

常见实现方案：

1. **插件方式（推荐）**：
   - 实现 `Interceptor` 接口
   - 拦截 `Executor.query()` 和 `Executor.update()`
   - 自动在 SQL 中追加 `tenant_id = ?` 条件
   - 优点：业务代码无感知

2. **MyBatis-Plus 多租户插件**：
   - 使用 `TenantLineInnerInterceptor`
   - 实现 `TenantLineHandler` 接口
   - 返回当前租户 ID 即可

3. **手动方式**：
   - 在每个 SQL 中手动添加 `tenant_id` 条件
   - 缺点：容易遗漏，维护困难

---

## 总结：MyBatis 核心知识点思维导图

```
MyBatis 核心知识体系
├── 架构
│   ├── 四大核心对象：Executor/StatementHandler/ParameterHandler/ResultSetHandler
│   ├── 三大执行器：Simple/Reuse/Batch（+Caching装饰者）
│   └── 完整执行流程：接口调用->代理->缓存->SQL->结果映射
├── 配置
│   ├── mybatis-config.xml：properties/settings/typeAliases/plugins/environments/mappers
│   ├── 关键 settings：mapUnderscoreToCamelCase/lazyLoadingEnabled/cacheEnabled
│   └── Spring Boot：application.yml 中的 mybatis.* 配置
├── Mapper XML
│   ├── 标签：select/insert/update/delete
│   ├── 参数：#{} 预编译 vs ${} 字符串替换（SQL注入风险）
│   ├── 结果映射：resultType（简单）vs resultMap（复杂）
│   └── 关联映射：association（一对一）/collection（一对多）/discriminator
├── 动态 SQL
│   ├── 条件：if/choose/when/otherwise
│   ├── 处理多余字符：where/set/trim
│   ├── 集合遍历：foreach（IN查询/批量插入）
│   └── 变量绑定：bind（模糊查询）
├── 缓存
│   ├── 一级缓存（SqlSession级）：默认开启，Spring无事务时失效
│   ├── 二级缓存（Mapper级）：手动开启，注意脏读问题
│   └── 查询顺序：二级->一级->数据库
├── 插件
│   ├── 原理：JDK动态代理 + 责任链
│   └── 常见：分页/慢SQL监控/数据权限/多租户
├── Spring 整合
│   ├── SqlSessionTemplate：线程安全的 SqlSession 代理
│   ├── MapperFactoryBean：Mapper 接口 Bean 工厂
│   └── 事务：@Transactional 保证 SqlSession 复用
└── MyBatis-Plus
    ├── BaseMapper：内置 CRUD
    ├── LambdaQueryWrapper：Lambda 条件构造（推荐）
    ├── 功能增强：逻辑删除/乐观锁/自动填充/分页
    └── 代码生成器：AutoGenerator
```

---

> **文档信息**
> - 作者：MyBatis 技术文档
> - 版本：MyBatis 3.5.x / MyBatis-Plus 3.5.x
> - Spring Boot 版本：3.x
> - 最后更新：2026年
> - 本文档涵盖从入门到精通的全部 MyBatis 知识点，适合反复翻阅参考

---
