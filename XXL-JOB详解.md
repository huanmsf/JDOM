# XXL-JOB 分布式任务调度平台 — 新手完全指南

> 版本参考：XXL-JOB v2.4.x  
> 适合人群：Java 后端开发新手、有 Spring Boot 基础的开发者

---

## 目录

1. [什么是 XXL-JOB](#1-什么是-xxl-job)
2. [核心概念](#2-核心概念)
3. [整体架构](#3-整体架构)
4. [底层原理深度解析](#4-底层原理深度解析)
5. [环境搭建](#5-环境搭建)
6. [快速上手 — 第一个定时任务](#6-快速上手--第一个定时任务)
7. [任务类型详解](#7-任务类型详解)
8. [路由策略与执行器集群](#8-路由策略与执行器集群)
9. [失败处理机制](#9-失败处理机制)
10. [日志与监控](#10-日志与监控)
11. [完整业务案例](#11-完整业务案例)
12. [常见问题 FAQ](#12-常见问题-faq)

---

## 1. 什么是 XXL-JOB

### 1.1 背景

在企业开发中，经常需要处理定时任务，例如：
- 每天凌晨 2 点统计昨日报表
- 每小时同步第三方数据
- 每 5 分钟检查订单超时

Java 原生的 `@Scheduled` 注解或 `Quartz` 框架可以处理简单场景，但在**分布式、集群**环境下会遇到以下问题：

| 问题 | 描述 |
|------|------|
| 重复执行 | 多个实例同时触发同一个任务 |
| 无法监控 | 任务执行结果不可见 |
| 故障无感知 | 任务失败没有报警 |
| 无法动态调整 | 修改 Cron 表达式需要重启服务 |

**XXL-JOB** 就是为解决这些问题而生的分布式任务调度平台，由大众点评许雪里（XuXueli）开源，目前已被众多大厂采用。

### 1.2 核心特性

- **简单**：提供可视化 Web 管理界面，操作简便
- **动态**：支持动态修改 Cron 表达式、启停任务，无需重启
- **分布式**：天然支持集群，解决多实例重复执行问题
- **高可用**：调度中心和执行器都支持集群部署
- **失败重试**：任务失败可自动重试，支持告警通知
- **分片广播**：支持将一个大任务拆分到多个执行器并行执行
- **可追溯**：完整的执行日志和报表

---

## 2. 核心概念

在学习架构之前，先理解几个核心术语：

```
┌─────────────────────────────────────────────┐
│               核心概念速查表                  │
├──────────────┬──────────────────────────────┤
│  调度中心     │ 负责触发任务、管理任务配置       │
│  (Admin)     │ 相当于"指挥官"                  │
├──────────────┼──────────────────────────────┤
│  执行器       │ 实际执行任务的业务应用            │
│  (Executor)  │ 相当于"士兵"                    │
├──────────────┼──────────────────────────────┤
│  任务(Job)   │ 具体的业务逻辑代码               │
├──────────────┼──────────────────────────────┤
│  Cron 表达式  │ 定义任务执行的时间规律            │
├──────────────┼──────────────────────────────┤
│  JobHandler  │ 任务处理器，即 @XxlJob 注解的方法 │
└──────────────┴──────────────────────────────┘
```

### 2.1 调度中心 (xxl-job-admin)

- 独立部署的 Spring Boot 应用
- 提供 Web 管理界面（任务管理、日志查看、报表统计）
- 负责在指定时间触发任务，通过 HTTP 调用执行器

### 2.2 执行器 (Executor)

- 集成在你的业务 Spring Boot 项目中
- 启动时向调度中心注册自己的地址
- 接收调度中心的 HTTP 请求，执行具体的 JobHandler

### 2.3 JobHandler

- 业务代码中用 `@XxlJob("taskName")` 注解标记的方法
- 一个执行器中可以有多个 JobHandler

---

## 3. 整体架构

### 3.1 架构图

```
                    ┌─────────────────────────────────┐
                    │         调度中心 (Admin)           │
                    │  ┌─────────┐  ┌───────────────┐  │
                    │  │  Web UI │  │  任务调度引擎   │  │
                    │  │  管理界面 │  │ (触发器/调度器) │  │
                    │  └─────────┘  └───────┬───────┘  │
                    │                       │           │
                    │  ┌────────────────────┴────────┐  │
                    │  │         MySQL 数据库          │  │
                    │  │  (任务配置/执行日志/注册信息)  │  │
                    │  └─────────────────────────────┘  │
                    └──────────────┬──────────────────┘
                                   │  HTTP(s) 调用
              ┌────────────────────┼────────────────────┐
              │                    │                    │
    ┌─────────▼───────┐  ┌─────────▼───────┐  ┌─────────▼───────┐
    │   执行器节点 1    │  │   执行器节点 2    │  │   执行器节点 3    │
    │                 │  │                 │  │                 │
    │ @XxlJob("job1") │  │ @XxlJob("job1") │  │ @XxlJob("job1") │
    │ @XxlJob("job2") │  │ @XxlJob("job2") │  │ @XxlJob("job2") │
    │                 │  │                 │  │                 │
    │   业务服务 A     │  │   业务服务 A     │  │   业务服务 A     │
    └─────────────────┘  └─────────────────┘  └─────────────────┘
              │                    │                    │
              └────────────────────┼────────────────────┘
                                   │
                            注册到调度中心
```

### 3.2 执行流程（一次任务触发的完整过程）

```
第一步：执行器启动，向调度中心注册
执行器 ──── 注册请求 (appName + IP:Port) ────► 调度中心
调度中心  ──  将执行器地址存入数据库  ──

第二步：到达 Cron 触发时间
调度中心 ── 扫描数据库中即将触发的任务 ──
调度中心 ── 根据路由策略选择一个执行器节点 ──
调度中心 ──── HTTP POST /run ────────────► 执行器

第三步：执行器执行任务
执行器 ── 根据 jobHandler 名称找到对应方法 ──
执行器 ── 在独立线程池中执行该方法 ──
执行器 ──── 执行结果回调 ─────────────────► 调度中心

第四步：调度中心记录结果
调度中心 ── 将执行结果写入日志表 ──
调度中心 ── 如失败且配置了重试，则重新触发 ──
```

---

## 4. 底层原理深度解析

### 4.1 注册与心跳机制

**执行器如何让调度中心知道自己的存在？**

```
执行器启动
    │
    ▼
读取配置 (adminAddresses, appName, port)
    │
    ▼
启动内嵌 HTTP Server (Netty 实现，默认端口 9999)
    │
    ▼
向调度中心发送注册请求
POST http://admin/api/registry
Body: { registryGroup: "EXECUTOR", registryKey: "myApp", registryValue: "192.168.1.100:9999" }
    │
    ▼
开启心跳线程 (每 30 秒发送一次心跳)
    │
    ▼
调度中心收到注册/心跳 → 更新数据库中的 update_time
    │
    ▼
调度中心有定时清理线程：超过 90 秒未心跳的执行器标记为下线
```

**关键数据库表：**

```sql
-- 执行器注册表
CREATE TABLE xxl_job_registry (
  id          INT PRIMARY KEY AUTO_INCREMENT,
  registry_group VARCHAR(50),   -- "EXECUTOR"
  registry_key   VARCHAR(255),  -- appName，如 "my-job-executor"
  registry_value VARCHAR(255),  -- IP:Port，如 "192.168.1.100:9999"
  update_time    DATETIME       -- 最后心跳时间
);
```

### 4.2 任务调度引擎原理

XXL-JOB 采用**时间轮（Time Wheel）**算法进行任务调度，而非简单的轮询扫描。

```
传统轮询方式（低效）：
每秒扫描全部任务 → 数据库压力大，精度低

XXL-JOB 的方式（高效）：

第一步：预读线程（每 5 秒执行一次）
    读取未来 5 秒内需要触发的所有任务
    按秒放入时间轮（内存中的 HashMap<Integer, List<Integer>>）
    key = 秒数(0-59)，value = 该秒需要触发的 jobId 列表

第二步：时间轮线程（每 1 秒执行一次）
    取出当前秒对应的任务列表
    批量触发这些任务

┌────────────────────────────────────────┐
│              时间轮示意                  │
│  秒  0: [jobId:1, jobId:5]             │
│  秒  1: []                             │
│  秒  2: [jobId:3]                      │
│  ...                                   │
│  秒 58: [jobId:2, jobId:7]             │
│  秒 59: []                             │
└────────────────────────────────────────┘
```

### 4.3 通信协议

调度中心与执行器之间通过 **HTTP** 通信（底层使用 Netty 实现的嵌入式服务器）：

```
调度中心 → 执行器 的请求（触发任务）：
POST http://executor-ip:9999/run
{
  "jobId": 1,
  "executorHandler": "myJobHandler",
  "executorParams": "参数字符串",
  "executorBlockStrategy": "SERIAL_EXECUTION",
  "executorTimeout": 0,
  "logId": 12345,
  "logDateTime": 1719000000000,
  "glueType": "BEAN",
  "glueSource": "",
  "glueUpdatetime": 0,
  "broadcastIndex": 0,
  "broadcastTotal": 1
}

执行器 → 调度中心 的回调（任务结果）：
POST http://admin/api/callback
[{
  "logId": 12345,
  "logDateTim": 1719000000000,
  "handleCode": 200,   // 200=成功, 500=失败
  "handleMsg": "执行成功"
}]
```

### 4.4 阻塞处理策略

当任务执行时间过长，上一次还没结束，下一次触发时间到了，该怎么办？

```
┌──────────────────────────────────────────────────────┐
│                  三种阻塞处理策略                       │
├────────────────┬─────────────────────────────────────┤
│ 串行执行        │ 新触发的任务进入队列等待上一次执行完毕   │
│ (默认)         │ 适用：任务必须顺序执行的场景            │
├────────────────┼─────────────────────────────────────┤
│ 丢弃后续调度    │ 如果执行器忙，直接丢弃本次触发          │
│                │ 适用：允许跳过某次执行的场景            │
├────────────────┼─────────────────────────────────────┤
│ 覆盖之前调度    │ 终止上一次正在执行的任务，立即执行新任务  │
│                │ 适用：需要以最新状态执行的场景           │
└────────────────┴─────────────────────────────────────┘
```

### 4.5 分片广播原理

适用于大数据量任务的并行处理：

```
场景：需要处理 100 万条用户数据，部署了 4 个执行器节点

分片广播触发时：
调度中心 同时 向 4 个执行器发送请求
  执行器1: broadcastIndex=0, broadcastTotal=4  → 处理 id % 4 == 0 的数据
  执行器2: broadcastIndex=1, broadcastTotal=4  → 处理 id % 4 == 1 的数据
  执行器3: broadcastIndex=2, broadcastTotal=4  → 处理 id % 4 == 2 的数据
  执行器4: broadcastIndex=3, broadcastTotal=4  → 处理 id % 4 == 3 的数据

代码获取分片参数：
ShardingUtil.ShardingVO shardingVO = ShardingUtil.getShardingVo();
int index = shardingVO.getIndex();   // 当前分片索引
int total = shardingVO.getTotal();   // 总分片数
```

---

## 5. 环境搭建

### 5.1 前置要求

- JDK 1.8+
- Maven 3.x
- MySQL 5.7+
- Spring Boot 2.x（执行器集成用）

### 5.2 部署调度中心

**Step 1：下载源码**

```bash
git clone https://github.com/xuxueli/xxl-job.git
cd xxl-job
```

或者直接下载 Release 包：https://github.com/xuxueli/xxl-job/releases

**Step 2：初始化数据库**

执行 `/xxl-job/doc/db/tables_xxl_job.sql`，该脚本会创建以下核心表：

```sql
-- 主要表结构说明
xxl_job_group      -- 执行器分组表
xxl_job_info       -- 任务配置表（Cron、Handler名称、路由策略等）
xxl_job_log        -- 任务执行日志表
xxl_job_log_report -- 日志统计报表
xxl_job_registry   -- 执行器注册表（IP:Port + 心跳时间）
xxl_job_user       -- 管理界面用户表
xxl_job_lock       -- 分布式锁表（防止集群重复调度）
```

**Step 3：修改调度中心配置**

编辑 `xxl-job-admin/src/main/resources/application.properties`：

```properties
# 服务端口
server.port=8080

# 数据源配置
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/xxl_job?useUnicode=true&characterEncoding=UTF-8&autoReconnect=true&serverTimezone=Asia/Shanghai
spring.datasource.username=root
spring.datasource.password=your_password

# 调度中心访问 token（执行器注册时需要匹配）
xxl.job.accessToken=your-secret-token
```

**Step 4：启动调度中心**

```bash
cd xxl-job-admin
mvn spring-boot:run
```

访问 `http://localhost:8080/xxl-job-admin`  
默认账号：`admin`，默认密码：`123456`

### 5.3 调度中心 Web 界面概览

```
导航菜单：
├── 运行报表    → 任务执行统计图表
├── 任务管理    → 创建/编辑/启停任务
├── 调度日志    → 每次执行的详细日志
├── 执行器管理  → 查看已注册的执行器
└── 用户管理    → 管理 Web 登录账号
```

---

## 6. 快速上手 — 第一个定时任务

### 6.1 在业务项目中集成执行器

**Step 1：添加 Maven 依赖**

```xml
<dependency>
    <groupId>com.xuxueli</groupId>
    <artifactId>xxl-job-core</artifactId>
    <version>2.4.0</version>
</dependency>
```

**Step 2：添加配置**

在 `application.yml` 中添加：

```yaml
xxl:
  job:
    admin:
      # 调度中心地址（集群用逗号分隔）
      addresses: http://127.0.0.1:8080/xxl-job-admin
    # 访问 token（与调度中心保持一致）
    accessToken: your-secret-token
    executor:
      # 执行器 AppName（在调度中心配置执行器时需要对应）
      appname: my-job-executor
      # 执行器注册地址（留空则自动获取本机 IP）
      address:
      ip:
      # 执行器端口（调度中心通过此端口调用执行器）
      port: 9999
      # 执行器日志文件存储路径
      logpath: /data/applogs/xxl-job/jobhandler
      # 日志保留天数（-1 表示永久保留）
      logretentiondays: 30
```

**Step 3：创建执行器配置类**

```java
package com.example.config;

import com.xxl.job.core.executor.impl.XxlJobSpringExecutor;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class XxlJobConfig {

    private Logger logger = LoggerFactory.getLogger(XxlJobConfig.class);

    @Value("${xxl.job.admin.addresses}")
    private String adminAddresses;

    @Value("${xxl.job.accessToken}")
    private String accessToken;

    @Value("${xxl.job.executor.appname}")
    private String appname;

    @Value("${xxl.job.executor.address}")
    private String address;

    @Value("${xxl.job.executor.ip}")
    private String ip;

    @Value("${xxl.job.executor.port}")
    private int port;

    @Value("${xxl.job.executor.logpath}")
    private String logPath;

    @Value("${xxl.job.executor.logretentiondays}")
    private int logRetentionDays;

    @Bean
    public XxlJobSpringExecutor xxlJobExecutor() {
        logger.info(">>>>>>>>>>> xxl-job config init.");
        XxlJobSpringExecutor xxlJobSpringExecutor = new XxlJobSpringExecutor();
        xxlJobSpringExecutor.setAdminAddresses(adminAddresses);
        xxlJobSpringExecutor.setAppname(appname);
        xxlJobSpringExecutor.setAddress(address);
        xxlJobSpringExecutor.setIp(ip);
        xxlJobSpringExecutor.setPort(port);
        xxlJobSpringExecutor.setAccessToken(accessToken);
        xxlJobSpringExecutor.setLogPath(logPath);
        xxlJobSpringExecutor.setLogRetentionDays(logRetentionDays);
        return xxlJobSpringExecutor;
    }
}
```

**Step 4：编写第一个 JobHandler**

```java
package com.example.job;

import com.xxl.job.core.handler.annotation.XxlJob;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Component;

@Component  // 必须是 Spring 管理的 Bean
public class HelloJobHandler {

    private static final Logger logger = LoggerFactory.getLogger(HelloJobHandler.class);

    /**
     * 最简单的任务：每次执行打印一行日志
     * 注解中的 "helloJob" 是任务标识，在调度中心配置时要填写这个名字
     */
    @XxlJob("helloJob")
    public void helloJob() {
        logger.info("Hello XXL-JOB! 当前时间：{}", System.currentTimeMillis());
        // 在这里写你的业务逻辑
        System.out.println("第一个 XXL-JOB 任务执行成功！");
    }
}
```

### 6.2 在调度中心创建任务

1. 登录调度中心 Web 界面
2. 进入「执行器管理」→ 点击「新增」
   - AppName：`my-job-executor`（与配置文件一致）
   - 名称：`我的业务执行器`
   - 注册方式：选「自动注册」
3. 进入「任务管理」→ 点击「新增」
   - 执行器：选择刚才创建的执行器
   - 任务描述：`Hello 示例任务`
   - Cron：`0/5 * * * * ?`（每 5 秒执行一次）
   - 运行模式：`BEAN`
   - JobHandler：`helloJob`（与 `@XxlJob` 注解中的名字一致）
   - 阻塞处理策略：`串行执行`
4. 启动业务服务，等待执行器自动注册
5. 在任务列表中点击「启动」

### 6.3 Cron 表达式快速参考

```
格式：秒 分 时 日 月 星期

常用示例：
0/5 * * * * ?       每 5 秒执行一次
0 0/1 * * * ?       每 1 分钟执行一次
0 0 2 * * ?         每天凌晨 2 点执行
0 0 9,18 * * ?      每天 9 点和 18 点执行
0 0 0 1 * ?         每月 1 号凌晨执行
0 0 0 ? * MON       每周一凌晨执行

在线生成工具：https://cron.qqe2.com/
```

---

## 7. 任务类型详解

XXL-JOB 支持多种运行模式，适合不同场景：

### 7.1 BEAN 模式（最常用）

使用 `@XxlJob` 注解，代码与项目融合，适合大多数场景。

```java
@XxlJob("demoJob")
public void demoJob() {
    // 获取任务参数（在调度中心配置任务时填写的参数）
    String param = XxlJobHelper.getJobParam();
    logger.info("收到参数: {}", param);

    // 主动记录日志（会显示在调度中心的日志详情中）
    XxlJobHelper.log("开始执行任务，参数={}", param);

    // 业务逻辑...

    // 标记任务成功（可选，不调用默认也是成功）
    XxlJobHelper.handleSuccess("任务执行完毕");

    // 标记任务失败（需要时调用）
    // XxlJobHelper.handleFail("执行出错：xxx");
}
```

### 7.2 GLUE 模式（动态脚本）

直接在调度中心 Web 界面编写代码，**无需部署，即改即生效**。

支持：Java、Shell、Python、NodeJS、PHP、PowerShell

```java
// GLUE Java 模式示例（直接在 Web 界面的代码编辑器中写）
package com.xxl.job.service.handler;

import com.xxl.job.core.context.XxlJobHelper;
import com.xxl.job.core.handler.IJobHandler;

public class DemoGlueJobHandler extends IJobHandler {
    @Override
    public void execute() throws Exception {
        XxlJobHelper.log("GLUE 模式运行成功！");
    }
}
```

```shell
# GLUE Shell 模式示例（直接在 Web 界面写 Shell 脚本）
#!/bin/bash
echo "Shell 脚本任务执行"
date
echo "执行完毕"
```

### 7.3 HTTP 任务模式

在调度中心配置时选择「HTTP」类型，直接填写 URL，无需部署执行器：

```
URL: http://your-service/api/task/execute
请求方式: POST
请求体: {"taskType": "daily_report"}
超时时间: 60 秒
```

---

## 8. 路由策略与执行器集群

当有多个执行器节点时，调度中心如何选择由哪个节点执行？

```
┌────────────────────────────────────────────────────────┐
│                    路由策略说明                          │
├──────────────────┬─────────────────────────────────────┤
│ FIRST            │ 固定选第一个注册的节点                 │
│ LAST             │ 固定选最后一个注册的节点               │
│ ROUND            │ 轮询，依次选择每个节点                  │
│ RANDOM           │ 随机选择一个节点                       │
│ CONSISTENT_HASH  │ 一致性哈希，同一 Job 固定打到同一节点   │
│ LEAST_FREQUENTLY │ 最不经常使用的节点（LFU）              │
│ LEAST_RECENTLY   │ 最久未使用的节点（LRU）                │
│ FAILOVER         │ 故障转移：按顺序探活，选第一个存活的节点  │
│ BUSYOVER         │ 忙碌转移：检测节点是否忙，选第一个空闲的  │
│ SHARDING_BROADCAST│ 分片广播：全部节点同时执行，传递分片参数 │
└──────────────────┴─────────────────────────────────────┘
```

**新手推荐：**
- 普通任务 → `ROUND`（轮询，负载均衡）
- 大数据量任务 → `SHARDING_BROADCAST`（分片广播）
- 高可用要求 → `FAILOVER`（故障自动转移）

---

## 9. 失败处理机制

### 9.1 失败重试

在调度中心创建任务时，设置「失败重试次数」：
- 0：不重试（默认）
- n：失败后最多重试 n 次

```java
@XxlJob("retryableJob")
public void retryableJob() {
    try {
        // 可能失败的业务逻辑
        callExternalAPI();
        XxlJobHelper.handleSuccess();
    } catch (Exception e) {
        logger.error("任务执行失败", e);
        // 标记失败，如果配置了重试次数，调度中心会自动重新触发
        XxlJobHelper.handleFail("调用外部 API 失败：" + e.getMessage());
    }
}
```

### 9.2 失败告警

调度中心支持邮件告警，在 `application.properties` 中配置：

```properties
# 邮件配置
spring.mail.host=smtp.qq.com
spring.mail.port=25
spring.mail.username=your@qq.com
spring.mail.password=your_auth_code
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true
xxl.job.alarm.email=alert@yourcompany.com
```

在任务配置中填写「报警邮件」字段，任务失败时自动发送邮件通知。

### 9.3 超时控制

在任务配置中设置「任务超时时间」（单位：秒，0 表示不超时）：

```java
@XxlJob("timeoutJob")
public void timeoutJob() throws InterruptedException {
    // 如果任务超过配置的超时时间，调度中心会标记为超时失败
    // 但注意：仅标记失败，不会强制中断 Java 线程
    doLongRunningTask();
}
```

---

## 10. 日志与监控

### 10.1 在任务中写日志

```java
@XxlJob("loggingJob")
public void loggingJob() {
    // 使用 XxlJobHelper.log 写的日志会显示在调度中心「日志详情」页面
    XxlJobHelper.log("任务开始执行");
    XxlJobHelper.log("处理数据量：{}", 1000);

    for (int i = 0; i < 10; i++) {
        // 可以在循环中持续写日志，方便追踪进度
        XxlJobHelper.log("正在处理第 {} 条", i + 1);
    }

    XxlJobHelper.log("任务执行完毕");
}
```

### 10.2 查看日志

1. 登录调度中心 → 「调度日志」菜单
2. 可按执行器、任务、时间范围过滤
3. 点击「执行日志」按钮查看完整输出
4. 支持实时滚动查看正在执行任务的日志

### 10.3 运行报表

调度中心首页「运行报表」展示：
- 任务数量统计
- 调度次数折线图
- 执行器在线状态
- 近期失败任务

---

## 11. 完整业务案例

### 案例：每日订单数据统计任务

**需求：** 每天凌晨 1 点统计昨日订单总量、总金额，写入报表数据库。

#### 项目结构

```
src/main/java/com/example/
├── config/
│   └── XxlJobConfig.java          # 执行器配置
├── job/
│   └── OrderStatisticsJob.java    # 任务 Handler
├── service/
│   └── OrderStatisticsService.java # 业务逻辑
└── mapper/
    ├── OrderMapper.java
    └── ReportMapper.java
```

#### OrderStatisticsJob.java

```java
package com.example.job;

import com.example.service.OrderStatisticsService;
import com.xxl.job.core.context.XxlJobHelper;
import com.xxl.job.core.handler.annotation.XxlJob;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

@Component
public class OrderStatisticsJob {

    private static final Logger log = LoggerFactory.getLogger(OrderStatisticsJob.class);

    @Autowired
    private OrderStatisticsService statisticsService;

    /**
     * 每日订单统计任务
     * Cron: 0 0 1 * * ?  （每天凌晨 1 点）
     */
    @XxlJob("dailyOrderStatisticsJob")
    public void dailyOrderStatisticsJob() {

        // 1. 确定统计日期（昨天）
        LocalDate yesterday = LocalDate.now().minusDays(1);
        String dateStr = yesterday.format(DateTimeFormatter.ofPattern("yyyy-MM-dd"));

        XxlJobHelper.log("===== 开始执行每日订单统计，统计日期：{} =====", dateStr);

        try {
            // 2. 检查是否已经统计过（幂等性保障）
            if (statisticsService.isAlreadyStatistic(yesterday)) {
                XxlJobHelper.log("日期 {} 已统计，跳过", dateStr);
                XxlJobHelper.handleSuccess("已统计，跳过");
                return;
            }

            // 3. 执行统计逻辑
            XxlJobHelper.log("开始查询订单数据...");
            long orderCount = statisticsService.countOrders(yesterday);
            XxlJobHelper.log("订单总量：{}", orderCount);

            double totalAmount = statisticsService.sumOrderAmount(yesterday);
            XxlJobHelper.log("订单总金额：{}", totalAmount);

            // 4. 写入报表
            statisticsService.saveReport(yesterday, orderCount, totalAmount);
            XxlJobHelper.log("报表写入成功");

            // 5. 标记成功
            XxlJobHelper.handleSuccess(
                String.format("统计完成，订单量=%d，总金额=%.2f", orderCount, totalAmount)
            );

        } catch (Exception e) {
            log.error("订单统计任务执行失败，日期={}", dateStr, e);
            // 标记失败（配置了重试次数后会自动重试）
            XxlJobHelper.handleFail("统计失败：" + e.getMessage());
        }
    }
}
```

#### OrderStatisticsService.java

```java
package com.example.service;

import com.example.mapper.OrderMapper;
import com.example.mapper.ReportMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;

@Service
public class OrderStatisticsService {

    @Autowired
    private OrderMapper orderMapper;

    @Autowired
    private ReportMapper reportMapper;

    public boolean isAlreadyStatistic(LocalDate date) {
        return reportMapper.existsByDate(date) > 0;
    }

    public long countOrders(LocalDate date) {
        return orderMapper.countByDate(date);
    }

    public double sumOrderAmount(LocalDate date) {
        Double amount = orderMapper.sumAmountByDate(date);
        return amount != null ? amount : 0.0;
    }

    @Transactional
    public void saveReport(LocalDate date, long count, double amount) {
        reportMapper.insertDailyReport(date, count, amount);
    }
}
```

#### 调度中心配置

| 配置项 | 值 |
|--------|-----|
| 任务描述 | 每日订单统计 |
| Cron | `0 0 1 * * ?` |
| 运行模式 | BEAN |
| JobHandler | `dailyOrderStatisticsJob` |
| 路由策略 | FIRST（只需一个节点执行） |
| 失败重试次数 | 3 |
| 任务超时时间 | 300（5分钟） |
| 报警邮件 | ops@yourcompany.com |

### 案例二：分片广播处理大量用户推送

**需求：** 每天下午 3 点向 100 万用户推送活动通知，部署 4 个执行器节点并行处理。

```java
@XxlJob("broadcastPushJob")
public void broadcastPushJob() {
    // 获取分片参数
    int shardIndex = XxlJobHelper.getShardIndex();  // 当前分片：0, 1, 2, 3
    int shardTotal = XxlJobHelper.getShardTotal();  // 总分片数：4

    XxlJobHelper.log("分片执行：{}/{}", shardIndex, shardTotal);

    // 按分片查询该节点需要处理的用户
    // 例如：SELECT * FROM user WHERE id % 4 = shardIndex
    List<Long> userIds = userService.getUserIdsBySharding(shardIndex, shardTotal);
    XxlJobHelper.log("本分片需要处理 {} 个用户", userIds.size());

    int successCount = 0;
    int failCount = 0;

    for (Long userId : userIds) {
        try {
            pushService.sendActivityNotification(userId);
            successCount++;
        } catch (Exception e) {
            failCount++;
            XxlJobHelper.log("推送失败，userId={}, error={}", userId, e.getMessage());
        }
    }

    XxlJobHelper.log("分片 {} 执行完毕，成功={}，失败={}", shardIndex, successCount, failCount);

    if (failCount > 0) {
        XxlJobHelper.handleFail(String.format("有 %d 条推送失败", failCount));
    } else {
        XxlJobHelper.handleSuccess();
    }
}
```

---

## 12. 常见问题 FAQ

### Q1：执行器启动后，调度中心显示「无执行器」？

**原因排查：**
1. 检查 `appname` 配置是否与调度中心「执行器管理」中的 AppName 完全一致（区分大小写）
2. 检查执行器端口（默认 9999）是否被防火墙屏蔽
3. 检查 `adminAddresses` 是否能从执行器所在机器访问
4. 检查 `accessToken` 是否与调度中心一致

```bash
# 验证调度中心可访问
curl http://127.0.0.1:8080/xxl-job-admin/api/registry

# 验证执行器端口开放
telnet executor-ip 9999
```

### Q2：任务重复执行了怎么办？

**原因：** 多个执行器节点都注册了相同 AppName，路由策略选了「广播」。

**解决方案：**
- 路由策略改为 `ROUND` 或 `FIRST`，确保每次只有一个节点执行
- 在业务代码中加分布式锁或幂等性检查

```java
@XxlJob("idempotentJob")
public void idempotentJob() {
    String lockKey = "job:idempotent:" + LocalDate.now();
    // 使用 Redis 分布式锁，确保同一天只执行一次
    if (!redisLock.tryLock(lockKey, 3600)) {
        XxlJobHelper.log("任务已在其他节点执行，跳过");
        return;
    }
    // 业务逻辑...
}
```

### Q3：如何传递参数给任务？

在调度中心创建任务时，「任务参数」字段填写参数值，在代码中获取：

```java
@XxlJob("paramJob")
public void paramJob() {
    String param = XxlJobHelper.getJobParam();  // 获取任务参数
    // param 可以是 JSON 字符串，然后自行解析
    // 例如：{"type": "VIP", "batchSize": 1000}
    if (param != null && param.contains("VIP")) {
        handleVIPUsers();
    }
}
```

### Q4：任务执行很慢，如何查看进度？

在任务中间歇性调用 `XxlJobHelper.log()`，在调度中心「调度日志」→「执行日志」中可实时查看：

```java
for (int i = 0; i < total; i++) {
    processItem(items.get(i));
    if (i % 100 == 0) {
        // 每 100 条记录一次进度
        XxlJobHelper.log("进度：{}/{}", i, total);
    }
}
```

### Q5：XXL-JOB 与 Spring @Scheduled 如何选择？

| 维度 | @Scheduled | XXL-JOB |
|------|-----------|---------|
| 适用场景 | 单机、简单定时 | 分布式、企业级 |
| 可视化管理 | 无 | 有 Web 界面 |
| 集群支持 | 需要自己实现 | 原生支持 |
| 动态调整 | 需重启 | 实时生效 |
| 执行日志 | 无 | 完整记录 |
| 失败重试 | 无 | 支持 |
| 部署复杂度 | 低 | 需要额外部署调度中心 |

**建议：** 单机应用用 `@Scheduled` 即可；生产环境分布式部署，选 XXL-JOB。

---

## 总结

```
XXL-JOB 核心流程回顾：

1. 部署调度中心 (xxl-job-admin)
        ↓
2. 业务项目集成 xxl-job-core 依赖
        ↓
3. 配置执行器 (XxlJobConfig)
        ↓
4. 编写任务方法 (@XxlJob 注解)
        ↓
5. 调度中心新建任务（填写 Cron + Handler名称）
        ↓
6. 启动任务，查看日志
```

XXL-JOB 的设计思想是**调度与执行分离**，调度中心只负责"什么时候触发"，执行器负责"怎么执行"。这种设计使得两者可以独立扩展、独立部署，是生产环境构建可靠定时任务系统的最佳实践之一。

---

*文档版本：2026-06-30 | 参考 XXL-JOB v2.4.x 官方文档*
