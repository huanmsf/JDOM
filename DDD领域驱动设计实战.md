# DDD 领域驱动设计实战 - 从零到精通

> 本文档全面覆盖 Domain-Driven Design（领域驱动设计）的战略设计、战术设计、架构模式、代码实现及落地实践，
> 包含完整的 Java / Spring Boot 代码示例，适合希望深入理解 DDD 并在企业项目中落地的开发者。

---

## 目录

- [Part 1: DDD 概述与背景](#part-1-ddd-概述与背景)
- [Part 2: DDD 核心概念体系](#part-2-ddd-核心概念体系)
- [Part 3: DDD 四层架构](#part-3-ddd-四层架构)
- [Part 4: 聚合设计原则](#part-4-聚合设计原则)
- [Part 5: 领域事件](#part-5-领域事件)
- [Part 6: Repository 模式实现](#part-6-repository-模式实现)
- [Part 7: CQRS 实现](#part-7-cqrs-实现)
- [Part 8: EventStorming 事件风暴](#part-8-eventstorming-事件风暴)
- [Part 9: 完整电商 DDD 实战案例](#part-9-完整电商-ddd-实战案例)
- [Part 10: DDD 与微服务](#part-10-ddd-与微服务)
- [Part 11: DDD 落地常见问题](#part-11-ddd-落地常见问题)
- [Part 12: 常见面试题 FAQ](#part-12-常见面试题-faq)

---
# Part 1: DDD 概述与背景

## 1.1 什么是 DDD

领域驱动设计（Domain-Driven Design，简称 DDD）是由 Eric Evans 在其 2003 年出版的经典著作
《Domain-Driven Design: Tackling Complexity in the Heart of Software》（中译《领域驱动设计：
软件核心复杂性应对之道》）中系统阐述的一套软件设计方法论。

DDD 的核心思想是：**将软件系统的设计与业务领域深度绑定，通过与领域专家（Domain Expert）
协作建立统一语言（Ubiquitous Language），并将该语言直接映射到代码模型中**，从而使软件能够
持续适应复杂业务的变化。

```
+----------------------------------------------------------+
|              Eric Evans 领域驱动设计 核心主张              |
+----------------------------------------------------------+
|                                                          |
|  业务复杂性     <---->    软件模型复杂性                     |
|                                                          |
|  领域专家语言   <---->    代码中的类/方法/包名               |
|                                                          |
|  业务规则变化   <---->    领域模型演进                       |
|                                                          |
|  核心：用代码忠实地表达业务领域的知识和规则                    |
+----------------------------------------------------------+
```

### DDD 的两大设计范畴

```
+-----------------------------+-----------------------------+
|         战略设计             |          战术设计            |
|      (Strategic Design)     |      (Tactical Design)      |
+-----------------------------+-----------------------------+
| 领域（Domain）               | 实体（Entity）               |
| 子域（Subdomain）            | 值对象（Value Object）       |
| 限界上下文（Bounded Context) | 聚合（Aggregate）            |
| 上下文映射（Context Map）    | 聚合根（Aggregate Root）     |
| 统一语言（Ubiquitous Lang）  | 领域服务（Domain Service）   |
|                             | 领域事件（Domain Event）     |
|                             | 仓储（Repository）           |
|                             | 工厂（Factory）              |
+-----------------------------+-----------------------------+
       宏观：划分边界、定义关系        微观：具体代码设计实现
```

---

## 1.2 为什么需要 DDD

### 1.2.1 业务复杂度爆炸

现代企业软件系统动辄数十万行代码，涉及数百个业务流程，在传统开发方式下极易产生以下问题：

```
传统开发方式的问题树
├── 代码层面
│   ├── 业务逻辑分散在各个 Service 类中
│   ├── Service 越来越大（上万行的 OrderService）
│   ├── 类之间依赖混乱（A 依赖 B，B 依赖 C，C 又依赖 A）
│   └── 数据库结构驱动设计（表驱动模型）
├── 协作层面
│   ├── 开发与业务对话困难（术语不统一）
│   ├── 需求变更难以快速响应
│   └── 新人上手成本极高
└── 演进层面
    ├── 修改一处影响全局
    ├── 无法安全重构
    └── 技术债务持续累积
```

### 1.2.2 贫血模型 vs 充血模型（详细代码对比）

**贫血模型（Anemic Domain Model）—— 反模式**

```java
// ============ 贫血模型示例 ============

// 贫血实体 - 只是一个数据容器
public class Order {
    private Long orderId;
    private String customerId;
    private List<OrderItem> items;
    private BigDecimal totalAmount;
    private String status; // "PENDING","PAID","SHIPPED","CANCELLED"
    private LocalDateTime createdAt;

    // 只有 getter/setter，零业务逻辑
    public Long getOrderId() { return orderId; }
    public void setOrderId(Long orderId) { this.orderId = orderId; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) {
        this.totalAmount = totalAmount;
    }
    // ... 其他 getter/setter
}

// 所有业务逻辑都堆在 Service 中
@Service
@Transactional
public class OrderService {

    @Autowired
    private OrderRepository orderRepository;
    @Autowired
    private InventoryClient inventoryClient;
    @Autowired
    private NotificationClient notificationClient;

    // 取消订单 - 业务逻辑完全在 Service 里
    public void cancelOrder(Long orderId, String reason) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new RuntimeException("订单不存在"));

        // 状态检查（业务规则散落在 Service）
        if (!"PENDING".equals(order.getStatus())
                && !"PAID".equals(order.getStatus())) {
            throw new RuntimeException("当前状态不允许取消");
        }
        // 金额检查
        if (order.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
            throw new RuntimeException("订单金额异常");
        }
        // 修改状态（直接操作数据字段）
        order.setStatus("CANCELLED");
        order.setCancelReason(reason);
        order.setCancelledAt(LocalDateTime.now());

        orderRepository.save(order);

        // 触发其他操作（也散落在 Service）
        inventoryClient.releaseStock(orderId);
        notificationClient.sendCancelNotification(order.getCustomerId());
    }
}
```

**充血模型（Rich Domain Model）—— DDD 推荐**

```java
// ============ 充血模型示例 ============

// 充血实体 - 有数据、有行为、有规则
public class Order extends AggregateRoot<OrderId> {
    private OrderId orderId;
    private CustomerId customerId;
    private List<OrderItem> items;
    private Money totalAmount;
    private OrderStatus status;
    private LocalDateTime createdAt;
    private String cancelReason;
    private LocalDateTime cancelledAt;

    // 构造器强制验证不变性规则
    public Order(CustomerId customerId, List<OrderItem> items) {
        Assert.notNull(customerId, "客户ID不能为空");
        Assert.notEmpty(items, "订单必须包含至少一个商品");
        this.orderId = OrderId.generate();
        this.customerId = customerId;
        this.items = new ArrayList<>(items);
        this.status = OrderStatus.PENDING;
        this.createdAt = LocalDateTime.now();
        this.totalAmount = calculateTotal();
        // 发布领域事件
        registerEvent(new OrderCreatedEvent(this.orderId, customerId, totalAmount));
    }

    // 业务行为内聚在实体内部
    public void cancel(String reason) {
        if (!status.canCancel()) {
            throw new OrderDomainException(
                "订单[" + orderId + "]状态[" + status + "]不允许取消");
        }
        this.status = OrderStatus.CANCELLED;
        this.cancelReason = reason;
        this.cancelledAt = LocalDateTime.now();
        registerEvent(new OrderCancelledEvent(this.orderId, reason));
    }

    public void markAsPaid(PaymentId paymentId) {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("只有待支付订单才能标记为已支付");
        }
        this.status = OrderStatus.PAID;
        registerEvent(new OrderPaidEvent(this.orderId, paymentId, totalAmount));
    }

    public void ship(String trackingNumber) {
        if (status != OrderStatus.PAID) {
            throw new OrderDomainException("只有已支付订单才能发货");
        }
        this.status = OrderStatus.SHIPPED;
        registerEvent(new OrderShippedEvent(this.orderId, trackingNumber));
    }

    // 私有业务方法
    private Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // 安全的只读访问
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }

    public boolean isPending() { return status == OrderStatus.PENDING; }
    public boolean isCancelled() { return status == OrderStatus.CANCELLED; }
    public OrderId getId() { return orderId; }
    public Money getTotalAmount() { return totalAmount; }
    public OrderStatus getStatus() { return status; }
}

// 应用服务 - 只做编排，不含业务规则
@Service
@Transactional
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    public OrderId cancelOrder(CancelOrderCommand command) {
        // 1. 加载聚合根
        Order order = orderRepository.findById(command.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(command.getOrderId()));

        // 2. 调用领域方法（业务规则在领域层）
        order.cancel(command.getReason());

        // 3. 持久化
        orderRepository.save(order);

        // 4. 发布领域事件
        eventPublisher.publishAll(order.getDomainEvents());

        return order.getId();
    }
}
```

**两种模型对比总结**

```
+------------------+-------------------------+-------------------------+
|     维度          |       贫血模型           |       充血模型           |
+------------------+-------------------------+-------------------------+
| 业务逻辑位置      | Service 层              | 实体/值对象/聚合根内部    |
| 实体职责         | 数据载体                 | 数据 + 行为 + 规则       |
| 不变性保证       | Service 负责             | 实体构造器/方法内部保证   |
| 可测试性        | 需要 Spring 容器         | 纯 POJO 单元测试         |
| 业务规则散落     | 是（多处重复）            | 否（内聚在实体）          |
| OO 程度        | 面向过程（伪OO）          | 真正面向对象              |
| 代码可读性      | 需要读 Service 理解业务   | 读实体即可理解业务         |
| 领域专家沟通    | 难（Service 方法名抽象）  | 易（方法名即业务动作）      |
+------------------+-------------------------+-------------------------+
```

---

## 1.3 DDD 适用与不适用场景

### 适用场景

```
+--------------------------------------------------+
|              DDD 适用的系统特征                    |
+--------------------------------------------------+
|                                                  |
|  1. 业务复杂度高                                   |
|     - 复杂的业务规则（保险核保、金融风控、供应链）   |
|     - 频繁的业务规则变化                           |
|     - 多个子系统交互                               |
|                                                  |
|  2. 团队规模大                                     |
|     - 多个团队并行开发                             |
|     - 需要清晰的职责边界                           |
|     - 长期维护（5年以上）                          |
|                                                  |
|  3. 技术与业务紧密结合                              |
|     - 开发与业务专家能够深度合作                    |
|     - 愿意投入建模时间                             |
|                                                  |
|  典型场景：                                       |
|  + 电商平台的订单/交易域                           |
|  + 保险理赔系统                                   |
|  + 银行核心账务系统                               |
|  + 物流调度系统                                   |
|  + ERP/CRM 核心模块                              |
+--------------------------------------------------+
```

### 不适用场景

```
+--------------------------------------------------+
|              DDD 不适用的系统特征                   |
+--------------------------------------------------+
|                                                  |
|  1. 业务简单（CRUD 为主）                           |
|     - 数据录入和展示系统                           |
|     - 简单的配置管理后台                           |
|     - 纯报表系统                                  |
|                                                  |
|  2. 短生命周期项目                                 |
|     - 活动页面（3个月内下线）                       |
|     - MVP 验证阶段产品                            |
|                                                  |
|  3. 团队不具备条件                                 |
|     - 没有领域专家配合                             |
|     - DDD 学习曲线成本超过收益                      |
|                                                  |
|  - 简单的增删改查管理系统                           |
|  - 数据迁移/ETL 工具                              |
|  - 简单 API 网关代理                              |
+--------------------------------------------------+
```

---

## 1.4 DDD vs 传统三层架构

传统三层架构（Controller → Service → DAO）是当前最主流的 Java Web 开发架构，
但与 DDD 四层架构存在本质区别：

```
传统三层架构
+------------------+
|   Controller     |  接收 HTTP 请求，参数校验
+------------------+
         |
         v
+------------------+
|    Service       |  所有业务逻辑都在这里（上帝Service）
+------------------+
         |
         v
+------------------+
|      DAO         |  数据库操作（Mapper/Repository）
+------------------+
         |
         v
+------------------+
|    Database      |  MySQL/Oracle
+------------------+

问题：
1. Service 越来越臃肿
2. 业务逻辑与技术细节混在一起
3. 领域对象（Entity/DO）只是数据载体
4. 无法自然地表达业务边界
```

```
DDD 四层架构
+------------------------+
|    用户接口层           |  Controller / DTO / Assembler
|  (Interfaces Layer)    |
+------------------------+
           |
           v
+------------------------+
|    应用层              |  ApplicationService / Command / Query Handler
|  (Application Layer)   |  事务边界，不含业务规则
+------------------------+
           |
           v
+------------------------+
|    领域层              |  Entity / ValueObject / AggregateRoot
|   (Domain Layer)       |  DomainService / Repository接口 / DomainEvent
|   （核心，不依赖任何    |
|    框架和基础设施）      |
+------------------------+
           |
           v
+------------------------+
|   基础设施层            |  Repository实现 / MQ / Cache / 外部API
| (Infrastructure Layer) |  JPA / MyBatis / Spring Data
+------------------------+

优势：
1. 领域层不依赖框架，可独立测试
2. 业务规则内聚在领域对象中
3. 清晰的职责边界
4. 自然地表达业务语义
```

**传统三层 vs DDD 四层代码对比**

```java
// =========== 传统三层：新建订单 ===========
@RestController
public class OrderController {
    @PostMapping("/orders")
    public ResponseEntity<Long> createOrder(@RequestBody CreateOrderRequest req) {
        Long orderId = orderService.createOrder(req);
        return ResponseEntity.ok(orderId);
    }
}

@Service
@Transactional
public class OrderService {
    // Service 直接操作 DO（贫血模型）
    public Long createOrder(CreateOrderRequest req) {
        // 参数校验
        if (req.getItems() == null || req.getItems().isEmpty()) {
            throw new IllegalArgumentException("商品不能为空");
        }
        // 查库存（直接调用 DAO）
        for (OrderItemRequest item : req.getItems()) {
            InventoryDO inv = inventoryMapper.selectById(item.getProductId());
            if (inv.getStock() < item.getQuantity()) {
                throw new RuntimeException("库存不足");
            }
        }
        // 计算总价
        BigDecimal total = req.getItems().stream()
            .map(i -> i.getPrice().multiply(new BigDecimal(i.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        // 构建 DO
        OrderDO orderDO = new OrderDO();
        orderDO.setCustomerId(req.getCustomerId());
        orderDO.setTotalAmount(total);
        orderDO.setStatus("PENDING");
        orderDO.setCreatedAt(LocalDateTime.now());
        orderMapper.insert(orderDO);
        // ... 插入订单项
        return orderDO.getId();
    }
}

// =========== DDD 四层：新建订单 ===========
// 用户接口层
@RestController
public class OrderController {
    @PostMapping("/orders")
    public ResponseEntity<String> placeOrder(
            @RequestBody PlaceOrderRequest request) {
        PlaceOrderCommand command = orderAssembler.toCommand(request);
        OrderId orderId = placeOrderHandler.handle(command);
        return ResponseEntity.ok(orderId.getValue());
    }
}

// 应用层 - 只做编排，不含业务规则
@Service
@Transactional
public class PlaceOrderHandler {
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final DomainEventPublisher eventPublisher;

    public OrderId handle(PlaceOrderCommand command) {
        // 1. 加载必要的领域对象
        List<OrderItem> orderItems = command.getItems().stream()
            .map(item -> {
                Product product = productRepository
                    .findById(item.getProductId())
                    .orElseThrow();
                return new OrderItem(product, item.getQuantity());
            })
            .collect(toList());

        // 2. 创建聚合根（业务规则在领域层执行）
        Order order = new Order(command.getCustomerId(), orderItems);

        // 3. 持久化
        orderRepository.save(order);

        // 4. 发布领域事件
        eventPublisher.publishAll(order.getDomainEvents());

        return order.getId();
    }
}
```

---

# Part 2: DDD 核心概念体系

## 2.1 战略设计

### 2.1.1 领域（Domain）

领域是 DDD 中最宏观的概念，代表软件所要解决的**业务问题空间**。

以电商平台为例：

```
电商平台 - 领域全景图
+----------------------------------------------------------+
|                     电商平台领域                          |
+--------------------+-------------------+-----------------+
|      核心域         |      支撑域        |     通用域      |
|   (Core Domain)    |  (Support Domain) | (Generic Domain)|
+--------------------+-------------------+-----------------+
|                    |                   |                 |
| 订单域（★最核心）   | 用户域             | 通知域          |
| - 下单流程         | - 用户注册/登录    | - 短信发送       |
| - 订单状态机       | - 用户画像         | - 邮件发送       |
| - 退换货           |                   | - Push消息       |
|                    | 仓储域             |                 |
| 交易域             | - 库位管理         | 权限域          |
| - 支付流程         | - 拣货分拣         | - RBAC           |
| - 风控             |                   | - 菜单管理       |
|                    | 物流域             |                 |
| 商品域             | - 运单管理         | 文件域          |
| - 商品上架         | - 轨迹查询         | - 图片存储       |
| - 定价             |                   | - 文档管理       |
| - 选品             |                   |                 |
+--------------------+-------------------+-----------------+
    公司最大竞争力        支撑核心运转          通用能力/可购买
    投入最多研发资源       自研或购买            直接购买SaaS
```

**三种域的区别**

| 域类型 | 定义 | 策略 | 举例 |
|--------|------|------|------|
| 核心域 | 企业核心竞争力所在，最重要、最复杂 | 重点投入，最优秀工程师，深度 DDD | 电商的订单/交易 |
| 支撑域 | 支撑核心域运转，不是差异化竞争力 | 自研或外购，适度 DDD | 仓储管理、用户域 |
| 通用域 | 通用技术能力，行业成熟解决方案 | 直接购买 SaaS，不自研 | 短信、邮件、权限 |

### 2.1.2 限界上下文（Bounded Context）

限界上下文是 DDD 最重要的概念之一，是**语义的边界**。

**核心理念：同一个词在不同上下文有不同含义。**

```
"订单（Order）"在不同上下文的含义差异

+-------------------+-------------------+-------------------+
|   电商订单上下文   |   仓储上下文        |   财务上下文       |
+-------------------+-------------------+-------------------+
| Order             | Order             | Order             |
| - orderId         | - pickingOrderId  | - invoiceOrderId  |
| - customerId      | - warehouseId     | - accountId       |
| - productList     | - shelfLocation   | - amount          |
| - shippingAddress | - pickingStatus   | - taxAmount       |
| - paymentStatus   | - operator        | - invoiceStatus   |
| - totalAmount     | - pickingTime     | - vatRate         |
+-------------------+-------------------+-------------------+
| 关注：客户购买体验 | 关注：仓库作业效率  | 关注：财务合规核算  |
+-------------------+-------------------+-------------------+
```

**识别限界上下文的方法：EventStorming（事件风暴）**

具体参见 Part 8，这里给出结果：

```
电商平台限界上下文划分

+----------+   +----------+   +----------+
|  用户上下文 |   | 商品上下文  |   | 订单上下文  |
|          |   |          |   |          |
| User     |   | Product  |   | Order    |
| Profile  |   | SKU      |   | OrderItem|
| Address  |   | Category |   | Promotion|
| Points   |   | Price    |   | Coupon   |
+----------+   +----------+   +----------+
                                   |
              +--------------------+--------------------+
              |                    |                    |
         +----------+        +----------+        +----------+
         | 库存上下文  |        | 支付上下文  |        | 通知上下文  |
         |          |        |          |        |          |
         | Inventory|        | Payment  |        |Notification|
         | Stock    |        | Account  |        | Template |
         | Warehouse|        | Channel  |        | Record   |
         +----------+        +----------+        +----------+
```

### 2.1.3 上下文映射（Context Map）

上下文之间的关系类型：

```
上下文映射关系类型

1. 合作关系（Partnership）
   两个上下文团队紧密合作，共同成功或失败
   [Team A]  <==合作==>  [Team B]

2. 共享内核（Shared Kernel）
   两个上下文共享一部分领域模型（慎用！）
   [Context A] --- 共享 User/Money 模型 --- [Context B]

3. 客户-供应商（Customer-Supplier）
   下游（客户）依赖上游（供应商），上游排期影响下游
   [订单上下文（客户）] <--- 依赖 --- [用户上下文（供应商）]

4. 遵奉者（Conformist）
   下游无力影响上游，直接使用上游模型（可能会污染本地模型）
   [内部系统] --遵奉--> [第三方系统（如支付宝）]

5. 防腐层（Anti-Corruption Layer，ACL）★重要
   下游通过翻译层隔离上游模型，保护本地领域模型纯洁性
   [本地上下文] --> [ACL（翻译/适配）] --> [外部上下文]

6. 开放主机服务（Open Host Service）
   上游提供稳定的协议/API（如 REST/gRPC）供多个下游消费
   [上游服务] --OHS(REST API)--> [多个下游消费者]

7. 发布语言（Published Language）
   上下游通过公开的语言交流（如 JSON Schema/Protobuf）
   [Producer] --PL(Protobuf)--> [Consumer1, Consumer2]
```

### 2.1.4 防腐层（ACL）完整实现

```java
// ============ 防腐层代码示例 ============
// 场景：订单上下文调用外部物流系统，
//       物流系统返回自己的数据结构，ACL 负责翻译

// 外部物流系统的数据结构（第三方，我们无法修改）
public class LogisticsSystemDTO {
    private String trackNo;           // 运单号
    private String delivStatus;       // "001"=待揽件,"002"=运输中,"003"=已签收
    private String estimateDate;      // "2024-01-15" 格式字符串
    private List<TrackNodeDTO> nodes; // 轨迹节点

    public static class TrackNodeDTO {
        private String time;     // "2024-01-13 15:30:00"
        private String location; // "上海浦东转运中心"
        private String desc;     // "包裹已到达"
    }
}

// 订单上下文自己的物流值对象（本地领域模型）
public class ShipmentInfo {
    private TrackingNumber trackingNumber;
    private ShipmentStatus status;
    private LocalDate estimatedArrivalDate;
    private List<TrackingEvent> events;

    public enum ShipmentStatus {
        WAITING_PICKUP, IN_TRANSIT, DELIVERED
    }
}

// ★ 防腐层接口（在领域层定义）
public interface LogisticsService {
    ShipmentInfo getShipmentInfo(TrackingNumber trackingNumber);
}

// ★ 防腐层实现（在基础设施层）
@Component
public class LogisticsServiceAcl implements LogisticsService {

    private final ExternalLogisticsClient externalClient;  // 真实 HTTP 客户端

    @Override
    public ShipmentInfo getShipmentInfo(TrackingNumber trackingNumber) {
        // 1. 调用外部 API
        LogisticsSystemDTO dto =
            externalClient.queryByTrackNo(trackingNumber.getValue());

        // 2. 翻译：外部模型 -> 本地领域模型（ACL 的核心工作）
        return translate(dto);
    }

    private ShipmentInfo translate(LogisticsSystemDTO dto) {
        ShipmentInfo.ShipmentStatus status = mapStatus(dto.getDelivStatus());

        LocalDate estimatedDate = dto.getEstimateDate() != null
            ? LocalDate.parse(dto.getEstimateDate())
            : null;

        List<TrackingEvent> events = dto.getNodes().stream()
            .map(node -> new TrackingEvent(
                LocalDateTime.parse(node.getTime(),
                    DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")),
                node.getLocation(),
                node.getDesc()))
            .collect(Collectors.toList());

        return new ShipmentInfo(
            new TrackingNumber(dto.getTrackNo()),
            status,
            estimatedDate,
            events);
    }

    private ShipmentInfo.ShipmentStatus mapStatus(String code) {
        return switch (code) {
            case "001" -> ShipmentInfo.ShipmentStatus.WAITING_PICKUP;
            case "002" -> ShipmentInfo.ShipmentStatus.IN_TRANSIT;
            case "003" -> ShipmentInfo.ShipmentStatus.DELIVERED;
            default -> throw new IllegalArgumentException(
                "未知物流状态码: " + code);
        };
    }
}
```

---

## 2.2 战术设计

### 2.2.1 实体（Entity）

**定义：** 有唯一标识（Identity），状态可以改变，通过标识判断相等性。

```java
// ============ 实体基类 ============
public abstract class Entity<ID> {
    protected ID id;

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Entity<?> entity = (Entity<?>) obj;
        // 实体相等性通过 ID 判断（不是值相等）
        return Objects.equals(id, entity.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// ============ 订单项实体 ============
public class OrderItem extends Entity<OrderItemId> {
    private OrderItemId id;
    private ProductId productId;
    private String productName;
    private Money unitPrice;
    private int quantity;
    private Money subtotal;

    public OrderItem(ProductId productId, String productName,
                     Money unitPrice, int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("商品数量必须大于0");
        }
        if (unitPrice.isNegativeOrZero()) {
            throw new IllegalArgumentException("商品单价必须大于0");
        }
        this.id = OrderItemId.generate();
        this.productId = productId;
        this.productName = productName;
        this.unitPrice = unitPrice;
        this.quantity = quantity;
        this.subtotal = unitPrice.multiply(quantity);
    }

    // 业务行为：修改数量（只有聚合根能调用）
    void updateQuantity(int newQuantity) {
        if (newQuantity <= 0) {
            throw new IllegalArgumentException("数量必须大于0");
        }
        this.quantity = newQuantity;
        this.subtotal = unitPrice.multiply(newQuantity);  // 自动重算小计
    }

    public Money getSubtotal() { return subtotal; }
    public ProductId getProductId() { return productId; }
    public int getQuantity() { return quantity; }
}
```

### 2.2.2 值对象（Value Object）

**定义：** 无唯一标识，不可变（Immutable），通过属性值判断相等性。

```java
// ============ Money 值对象 ============
public final class Money {
    public static final Money ZERO = new Money(BigDecimal.ZERO, "CNY");

    private final BigDecimal amount;
    private final String currency;

    public Money(BigDecimal amount, String currency) {
        if (amount == null) {
            throw new IllegalArgumentException("金额不能为null");
        }
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("金额不能为负数: " + amount);
        }
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    // 值对象的运算返回新的值对象（不可变！）
    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new InsufficientFundsException("金额不足");
        }
        return new Money(result, this.currency);
    }

    public Money multiply(int multiplier) {
        return new Money(this.amount.multiply(new BigDecimal(multiplier)), currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isNegativeOrZero() {
        return amount.compareTo(BigDecimal.ZERO) <= 0;
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException(
                "不同货币不能运算: " + this.currency + " vs " + other.currency);
        }
    }

    // 值对象：相等性基于所有属性值
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Money)) return false;
        Money money = (Money) obj;
        return Objects.equals(amount, money.amount)
            && Objects.equals(currency, money.currency);
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }

    @Override
    public String toString() {
        return currency + " " + amount.toPlainString();
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
}

// ============ Address 值对象 ============
public final class Address {
    private final String province;
    private final String city;
    private final String district;
    private final String street;
    private final String detail;
    private final String zipCode;
    private final String receiverName;
    private final String receiverPhone;

    public Address(String province, String city, String district,
                   String street, String detail, String zipCode,
                   String receiverName, String receiverPhone) {
        // 完整性验证
        Assert.hasText(province, "省份不能为空");
        Assert.hasText(city, "城市不能为空");
        Assert.hasText(detail, "详细地址不能为空");
        Assert.hasText(receiverName, "收货人姓名不能为空");
        Assert.hasText(receiverPhone, "收货人电话不能为空");
        if (!isValidPhone(receiverPhone)) {
            throw new IllegalArgumentException("手机号格式不正确");
        }
        this.province = province;
        this.city = city;
        this.district = district;
        this.street = street;
        this.detail = detail;
        this.zipCode = zipCode;
        this.receiverName = receiverName;
        this.receiverPhone = receiverPhone;
    }

    public String getFullAddress() {
        return province + city + district + street + detail;
    }

    private boolean isValidPhone(String phone) {
        return phone.matches("^1[3-9]\\d{9}$");
    }

    // 值对象：相等性基于所有字段
    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof Address)) return false;
        Address address = (Address) obj;
        return Objects.equals(province, address.province)
            && Objects.equals(city, address.city)
            && Objects.equals(district, address.district)
            && Objects.equals(street, address.street)
            && Objects.equals(detail, address.detail)
            && Objects.equals(receiverName, address.receiverName)
            && Objects.equals(receiverPhone, address.receiverPhone);
    }

    @Override
    public int hashCode() {
        return Objects.hash(province, city, district,
                            street, detail, receiverName, receiverPhone);
    }
}

// ============ OrderId 值对象（强类型 ID）============
public final class OrderId {
    private final String value;

    private OrderId(String value) {
        if (!value.matches("ORD-[0-9]{18}")) {
            throw new IllegalArgumentException("无效的订单ID格式: " + value);
        }
        this.value = value;
    }

    // 工厂方法
    public static OrderId generate() {
        String timestamp = String.valueOf(System.currentTimeMillis());
        String random = String.format("%03d", ThreadLocalRandom.current().nextInt(1000));
        return new OrderId("ORD-" + timestamp + random);
    }

    public static OrderId of(String value) {
        return new OrderId(value);
    }

    public String getValue() { return value; }

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (!(obj instanceof OrderId)) return false;
        return Objects.equals(value, ((OrderId) obj).value);
    }

    @Override
    public int hashCode() { return Objects.hash(value); }

    @Override
    public String toString() { return value; }
}
```

### 2.2.3 聚合（Aggregate）与聚合根（Aggregate Root）

**聚合是最重要的 DDD 战术概念**，代表一组必须保持业务一致性的对象集合。

```
聚合的一致性边界示意

+------------------------------------------+
|            Order 聚合                     |
|                                          |
|  +----------------------------------+   |
|  |      OrderId（聚合根 ID）         |   |
|  +----------------------------------+   |
|                                          |
|  +----------------------------------+   |
|  |      Order（聚合根）              |   |  <-- 外部只能通过
|  |  - orderId                       |   |      聚合根访问聚合
|  |  - status                        |   |
|  |  - totalAmount                   |   |
|  |  - shippingAddress               |   |
|  +----------------------------------+   |
|          |              |               |
|  +------------+  +------------+         |
|  | OrderItem  |  | OrderItem  |         |
|  | (实体)     |  | (实体)     |         |
|  +------------+  +------------+         |
+------------------------------------------+

聚合外部：
  外部对象只能持有 OrderId（值对象），不能持有 OrderItem 的引用！
  [Payment 聚合] ---- orderId (OrderId) ----> [Order 聚合]
```

```java
// ============ 聚合根基类 ============
public abstract class AggregateRoot<ID> extends Entity<ID> {
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected void registerEvent(DomainEvent event) {
        this.domainEvents.add(event);
    }

    public List<DomainEvent> getDomainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    public void clearDomainEvents() {
        this.domainEvents.clear();
    }
}

// ============ 订单聚合根 ============
public class Order extends AggregateRoot<OrderId> {
    private OrderId orderId;
    private CustomerId customerId;
    private List<OrderItem> items;       // 聚合内的实体
    private Address shippingAddress;     // 值对象
    private Money totalAmount;           // 值对象
    private OrderStatus status;          // 值对象（枚举）
    private LocalDateTime createdAt;
    private int version;                 // 乐观锁版本号

    // ===== 工厂方法：创建新订单 =====
    public static Order create(CustomerId customerId,
                               List<OrderItem> items,
                               Address shippingAddress) {
        Order order = new Order();
        // 不变性规则验证
        if (items == null || items.isEmpty()) {
            throw new OrderDomainException("订单必须包含至少一个商品");
        }
        if (items.size() > 100) {
            throw new OrderDomainException("单笔订单最多100种商品");
        }

        order.orderId = OrderId.generate();
        order.customerId = customerId;
        order.items = new ArrayList<>(items);
        order.shippingAddress = shippingAddress;
        order.status = OrderStatus.PENDING;
        order.createdAt = LocalDateTime.now();
        order.totalAmount = order.calculateTotal();
        order.version = 0;

        // 发布领域事件
        order.registerEvent(new OrderCreatedEvent(
            order.orderId, customerId, order.totalAmount, LocalDateTime.now()));
        return order;
    }

    // ===== 业务行为：支付 =====
    public void pay(PaymentId paymentId, Money paidAmount) {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException(
                "订单[" + orderId + "]状态[" + status + "]不允许支付");
        }
        if (!paidAmount.equals(totalAmount)) {
            throw new OrderDomainException(
                "支付金额[" + paidAmount + "]与订单金额[" + totalAmount + "]不一致");
        }
        this.status = OrderStatus.PAID;
        registerEvent(new OrderPaidEvent(orderId, paymentId, paidAmount));
    }

    // ===== 业务行为：取消 =====
    public void cancel(String reason) {
        if (!status.canCancel()) {
            throw new OrderDomainException(
                "订单[" + orderId + "]当前状态[" + status + "]不允许取消");
        }
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelledEvent(orderId, reason, LocalDateTime.now()));
    }

    // ===== 业务行为：发货 =====
    public void ship(String trackingNumber) {
        if (status != OrderStatus.PAID) {
            throw new OrderDomainException("只有已支付的订单才能发货");
        }
        if (trackingNumber == null || trackingNumber.isBlank()) {
            throw new OrderDomainException("运单号不能为空");
        }
        this.status = OrderStatus.SHIPPED;
        registerEvent(new OrderShippedEvent(orderId, trackingNumber));
    }

    // ===== 业务行为：确认收货 =====
    public void confirm() {
        if (status != OrderStatus.SHIPPED) {
            throw new OrderDomainException("只有已发货的订单才能确认收货");
        }
        this.status = OrderStatus.COMPLETED;
        registerEvent(new OrderCompletedEvent(orderId, LocalDateTime.now()));
    }

    // ===== 内部计算 =====
    private Money calculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // ===== 查询方法 =====
    public OrderId getId() { return orderId; }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
    public int getVersion() { return version; }
}

// ============ 订单状态枚举（含状态机）============
public enum OrderStatus {
    PENDING("待支付") {
        @Override
        public boolean canCancel() { return true; }
        @Override
        public boolean canPay() { return true; }
    },
    PAID("已支付") {
        @Override
        public boolean canCancel() { return true; }
        @Override
        public boolean canShip() { return true; }
    },
    SHIPPED("已发货") {
        @Override
        public boolean canConfirm() { return true; }
    },
    COMPLETED("已完成"),
    CANCELLED("已取消");

    private final String description;

    OrderStatus(String description) {
        this.description = description;
    }

    // 状态机：各状态允许的操作
    public boolean canCancel() { return false; }
    public boolean canPay() { return false; }
    public boolean canShip() { return false; }
    public boolean canConfirm() { return false; }
}
```

### 2.2.4 领域服务（Domain Service）

当一个业务行为**不自然地属于某个实体或值对象**时，使用领域服务。

```java
// ============ 领域服务：订单定价服务 ============
// 定价涉及多个概念（订单、优惠券、会员等级），不适合放在任一实体中

public interface OrderPricingService {
    /**
     * 计算订单最终价格
     * 考虑：商品原价、促销折扣、优惠券、会员折扣
     */
    Money calculateFinalPrice(Order order,
                               List<Promotion> promotions,
                               Coupon coupon,
                               MemberLevel memberLevel);
}

@DomainService  // 自定义注解，标识这是领域服务
public class OrderPricingServiceImpl implements OrderPricingService {

    @Override
    public Money calculateFinalPrice(Order order,
                                      List<Promotion> promotions,
                                      Coupon coupon,
                                      MemberLevel memberLevel) {
        // 1. 原始总价
        Money originalPrice = order.getTotalAmount();

        // 2. 应用促销活动（满减、打折）
        Money afterPromotion = applyPromotions(originalPrice, promotions, order);

        // 3. 应用优惠券
        Money afterCoupon = applyCoupon(afterPromotion, coupon);

        // 4. 应用会员折扣
        Money finalPrice = applyMemberDiscount(afterCoupon, memberLevel);

        // 5. 最低价保证（不得低于0）
        return finalPrice.isNegativeOrZero() ? Money.ZERO : finalPrice;
    }

    private Money applyPromotions(Money price, List<Promotion> promotions, Order order) {
        Money result = price;
        for (Promotion promo : promotions) {
            if (promo.isApplicable(order)) {
                result = promo.apply(result);
            }
        }
        return result;
    }

    private Money applyCoupon(Money price, Coupon coupon) {
        if (coupon == null || !coupon.isValid()) {
            return price;
        }
        return coupon.apply(price);
    }

    private Money applyMemberDiscount(Money price, MemberLevel level) {
        if (level == null) return price;
        BigDecimal discountRate = level.getDiscountRate();
        return new Money(
            price.getAmount().multiply(discountRate).setScale(2, RoundingMode.HALF_UP),
            price.getCurrency());
    }
}
```

### 2.2.5 仓储（Repository）

```java
// ============ Repository 接口（领域层定义）============
// 仓储接口是领域层的一部分，实现在基础设施层

public interface OrderRepository {
    /**
     * 根据ID查找订单（主要入口）
     */
    Optional<Order> findById(OrderId orderId);

    /**
     * 保存订单（新增或更新）
     */
    void save(Order order);

    /**
     * 删除订单
     */
    void delete(OrderId orderId);

    /**
     * 根据客户ID查找订单列表
     */
    List<Order> findByCustomerId(CustomerId customerId);

    /**
     * 查找指定状态的超时订单（用于定时任务）
     */
    List<Order> findTimeoutOrders(OrderStatus status,
                                   LocalDateTime before,
                                   int limit);
}

// 注意：Repository 是聚合的入口，不是表的入口！
// 不应该有 OrderItemRepository，
// OrderItem 通过 Order 聚合根访问
```

### 2.2.6 工厂（Factory）

当创建复杂对象时，使用工厂封装创建逻辑：

```java
// ============ 订单工厂 ============
@Component
public class OrderFactory {

    private final ProductRepository productRepository;
    private final InventoryService inventoryService;

    // 从命令对象创建订单聚合
    public Order createOrder(PlaceOrderCommand command) {
        // 1. 加载商品信息
        List<OrderItem> orderItems = command.getItems().stream()
            .map(itemCmd -> {
                Product product = productRepository
                    .findById(itemCmd.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException(
                        itemCmd.getProductId()));

                // 验证库存
                inventoryService.checkStock(
                    itemCmd.getProductId(), itemCmd.getQuantity());

                return new OrderItem(
                    product.getId(),
                    product.getName(),
                    product.getPrice(),
                    itemCmd.getQuantity());
            })
            .collect(Collectors.toList());

        // 2. 构建收货地址
        Address shippingAddress = new Address(
            command.getProvince(), command.getCity(),
            command.getDistrict(), command.getStreet(),
            command.getDetail(), command.getZipCode(),
            command.getReceiverName(), command.getReceiverPhone());

        // 3. 创建聚合根
        return Order.create(command.getCustomerId(), orderItems, shippingAddress);
    }
}
```

---

# Part 3: DDD 四层架构

## 3.1 架构总览

```
+=========================================================+
|                   DDD 四层架构全景                        |
+=========================================================+

+-----------------------------------------------------------+
|               用户接口层 (Interfaces Layer)                 |
|  Controller  |  DTO  |  Assembler  |  Form Validator       |
|  REST API / GraphQL / gRPC / Message Consumer             |
+-----------------------------------------------------------+
                          | 依赖（向下单向）
                          v
+-----------------------------------------------------------+
|               应用层 (Application Layer)                   |
|  ApplicationService  |  CommandHandler  |  QueryHandler   |
|  EventHandler  |  Saga  |  事务边界                        |
|  【不含业务规则，只做流程编排】                              |
+-----------------------------------------------------------+
                          | 依赖（向下单向）
                          v
+-----------------------------------------------------------+
|               领域层 (Domain Layer)                        |  <- 核心
|  Entity  |  ValueObject  |  AggregateRoot                  |
|  DomainService  |  Repository接口  |  DomainEvent          |
|  Factory  |  Specification  |  领域异常                    |
|  【不依赖任何框架，纯POJO，独立可测试】                      |
+-----------------------------------------------------------+
                          | 依赖（向上提供实现）
                          ^
+-----------------------------------------------------------+
|              基础设施层 (Infrastructure Layer)              |
|  Repository实现(JPA/MyBatis)  |  MQ Adapter                |
|  Cache Adapter  |  外部API Client  |  ACL                  |
|  DB Configuration  |  Spring Bean 配置                    |
+-----------------------------------------------------------+

依赖方向：用户接口层 -> 应用层 -> 领域层 <- 基础设施层
领域层不依赖任何外部！
```

## 3.2 项目包结构

```
com.example.ecommerce
├── interfaces                          # 用户接口层
│   ├── rest
│   │   ├── OrderController.java
│   │   ├── dto
│   │   │   ├── PlaceOrderRequest.java  # 入参 DTO
│   │   │   └── OrderResponse.java      # 出参 DTO
│   │   └── assembler
│   │       └── OrderAssembler.java     # DTO <-> Command/领域对象 转换
│   └── event
│       └── OrderEventConsumer.java     # MQ 消费者（也属于接口层）
│
├── application                         # 应用层
│   ├── command
│   │   ├── PlaceOrderCommand.java
│   │   └── CancelOrderCommand.java
│   ├── handler
│   │   ├── PlaceOrderHandler.java
│   │   └── CancelOrderHandler.java
│   ├── query
│   │   ├── OrderQuery.java
│   │   └── OrderQueryHandler.java
│   └── event
│       └── OrderPaidEventHandler.java
│
├── domain                              # 领域层（核心，零框架依赖）
│   └── order                           # 订单限界上下文
│       ├── model
│       │   ├── Order.java              # 聚合根
│       │   ├── OrderItem.java          # 实体
│       │   ├── OrderId.java            # 值对象
│       │   ├── OrderStatus.java        # 值对象（枚举）
│       │   ├── Money.java              # 值对象
│       │   └── Address.java            # 值对象
│       ├── event
│       │   ├── OrderCreatedEvent.java
│       │   ├── OrderPaidEvent.java
│       │   └── OrderCancelledEvent.java
│       ├── repository
│       │   └── OrderRepository.java    # 仓储接口（接口在领域层）
│       ├── service
│       │   └── OrderPricingService.java # 领域服务接口
│       └── exception
│           └── OrderDomainException.java
│
└── infrastructure                      # 基础设施层
    ├── persistence
    │   ├── OrderRepositoryImpl.java    # 仓储实现
    │   ├── po
    │   │   ├── OrderPO.java            # 持久化对象（与数据库表映射）
    │   │   └── OrderItemPO.java
    │   ├── mapper
    │   │   └── OrderMapper.java        # MyBatis Mapper
    │   └── converter
    │       └── OrderConverter.java     # 领域对象 <-> PO 转换
    ├── messaging
    │   └── OrderEventPublisher.java    # MQ 事件发布
    ├── cache
    │   └── OrderCacheRepository.java
    └── acl
        └── LogisticsServiceAcl.java    # 防腐层
```

## 3.3 各层详细说明与代码

### 3.3.1 用户接口层

```java
// ============ Controller ============
@RestController
@RequestMapping("/api/v1/orders")
@Validated
public class OrderController {

    private final PlaceOrderHandler placeOrderHandler;
    private final CancelOrderHandler cancelOrderHandler;
    private final OrderQueryHandler orderQueryHandler;
    private final OrderAssembler assembler;

    // 下单
    @PostMapping
    public ResponseEntity<ApiResponse<String>> placeOrder(
            @RequestBody @Valid PlaceOrderRequest request,
            @AuthenticationPrincipal UserPrincipal user) {
        PlaceOrderCommand command = assembler.toCommand(request, user.getUserId());
        OrderId orderId = placeOrderHandler.handle(command);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(ApiResponse.success(orderId.getValue()));
    }

    // 取消订单
    @PostMapping("/{orderId}/cancel")
    public ResponseEntity<ApiResponse<Void>> cancelOrder(
            @PathVariable String orderId,
            @RequestBody CancelOrderRequest request) {
        CancelOrderCommand command = new CancelOrderCommand(
            OrderId.of(orderId), request.getReason());
        cancelOrderHandler.handle(command);
        return ResponseEntity.ok(ApiResponse.success());
    }

    // 查询订单详情
    @GetMapping("/{orderId}")
    public ResponseEntity<ApiResponse<OrderDetailResponse>> getOrder(
            @PathVariable String orderId) {
        OrderDetailResponse response = orderQueryHandler.getById(OrderId.of(orderId));
        return ResponseEntity.ok(ApiResponse.success(response));
    }
}

// ============ DTO ============
public class PlaceOrderRequest {
    @NotNull(message = "收货地址不能为空")
    private AddressRequest shippingAddress;

    @NotEmpty(message = "订单商品不能为空")
    @Size(max = 100, message = "单笔订单最多100种商品")
    private List<OrderItemRequest> items;

    public static class OrderItemRequest {
        @NotNull
        private String productId;
        @Min(value = 1, message = "商品数量至少为1")
        @Max(value = 9999, message = "单商品数量不超过9999")
        private int quantity;
    }

    public static class AddressRequest {
        @NotBlank private String province;
        @NotBlank private String city;
        @NotBlank private String detail;
        @NotBlank private String receiverName;
        @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
        private String receiverPhone;
    }
}

// ============ Assembler（DTO <-> Command 转换）============
@Component
public class OrderAssembler {

    public PlaceOrderCommand toCommand(PlaceOrderRequest request, String userId) {
        List<PlaceOrderCommand.OrderItemCommand> items = request.getItems().stream()
            .map(item -> new PlaceOrderCommand.OrderItemCommand(
                ProductId.of(item.getProductId()),
                item.getQuantity()))
            .collect(Collectors.toList());

        PlaceOrderCommand.AddressCommand addr = new PlaceOrderCommand.AddressCommand(
            request.getShippingAddress().getProvince(),
            request.getShippingAddress().getCity(),
            request.getShippingAddress().getDetail(),
            request.getShippingAddress().getReceiverName(),
            request.getShippingAddress().getReceiverPhone());

        return new PlaceOrderCommand(CustomerId.of(userId), items, addr);
    }

    public OrderDetailResponse toResponse(Order order) {
        List<OrderDetailResponse.ItemResponse> items = order.getItems().stream()
            .map(item -> new OrderDetailResponse.ItemResponse(
                item.getProductId().getValue(),
                item.getProductName(),
                item.getUnitPrice().getAmount(),
                item.getQuantity(),
                item.getSubtotal().getAmount()))
            .collect(Collectors.toList());

        return new OrderDetailResponse(
            order.getId().getValue(),
            order.getStatus().name(),
            order.getTotalAmount().getAmount(),
            order.getCreatedAt(),
            items);
    }
}
```

### 3.3.2 应用层

```java
// ============ Command 对象（不可变）============
public class PlaceOrderCommand {
    private final CustomerId customerId;
    private final List<OrderItemCommand> items;
    private final AddressCommand address;

    public PlaceOrderCommand(CustomerId customerId,
                              List<OrderItemCommand> items,
                              AddressCommand address) {
        this.customerId = Objects.requireNonNull(customerId);
        this.items = Collections.unmodifiableList(
            Objects.requireNonNull(items));
        this.address = Objects.requireNonNull(address);
    }

    // getter only，无 setter（Command 不可变）
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderItemCommand> getItems() { return items; }
    public AddressCommand getAddress() { return address; }

    // 内嵌命令对象
    public record OrderItemCommand(ProductId productId, int quantity) {}
    public record AddressCommand(String province, String city,
                                 String detail, String receiverName,
                                 String receiverPhone) {}
}

// ============ Command Handler ============
@Service
@Transactional
@RequiredArgsConstructor
public class PlaceOrderHandler {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final OrderPricingService pricingService;
    private final DomainEventPublisher eventPublisher;
    private final OrderFactory orderFactory;

    public OrderId handle(PlaceOrderCommand command) {
        // 1. 使用工厂创建聚合根（工厂内部处理商品加载、库存验证等）
        Order order = orderFactory.createOrder(command);

        // 2. 应用定价规则（领域服务）
        // 这里简化处理，实际可以查询促销、优惠券等
        // Money finalPrice = pricingService.calculate(order, ...);

        // 3. 持久化聚合
        orderRepository.save(order);

        // 4. 发布领域事件（解耦后续流程）
        eventPublisher.publishAll(order.getDomainEvents());
        order.clearDomainEvents();

        return order.getId();
    }
}

// ============ 领域事件发布器接口（应用层使用）============
public interface DomainEventPublisher {
    void publish(DomainEvent event);
    void publishAll(List<DomainEvent> events);
}
```

### 3.3.3 领域层（核心）

```java
// ============ 领域事件基类 ============
public abstract class DomainEvent {
    private final String eventId;
    private final LocalDateTime occurredOn;
    private final String eventType;

    protected DomainEvent(String eventType) {
        this.eventId = UUID.randomUUID().toString();
        this.occurredOn = LocalDateTime.now();
        this.eventType = eventType;
    }

    public String getEventId() { return eventId; }
    public LocalDateTime getOccurredOn() { return occurredOn; }
    public String getEventType() { return eventType; }
}

// 订单创建事件
public class OrderCreatedEvent extends DomainEvent {
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money totalAmount;

    public OrderCreatedEvent(OrderId orderId, CustomerId customerId,
                              Money totalAmount, LocalDateTime occurredOn) {
        super("ORDER_CREATED");
        this.orderId = orderId;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
    }

    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public Money getTotalAmount() { return totalAmount; }
}
```

### 3.3.4 基础设施层

```java
// ============ Repository 实现（基础设施层）============
@Repository
public class OrderRepositoryImpl implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderConverter converter;

    @Override
    public Optional<Order> findById(OrderId orderId) {
        return jpaRepository.findById(orderId.getValue())
            .map(converter::toDomain);  // PO -> 领域对象
    }

    @Override
    public void save(Order order) {
        OrderPO po = converter.toPo(order);  // 领域对象 -> PO
        jpaRepository.save(po);
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepository
            .findByCustomerIdOrderByCreatedAtDesc(customerId.getValue())
            .stream()
            .map(converter::toDomain)
            .collect(Collectors.toList());
    }
}

// JPA 接口（Spring Data）
public interface OrderJpaRepository extends JpaRepository<OrderPO, String> {
    List<OrderPO> findByCustomerIdOrderByCreatedAtDesc(String customerId);

    @Query("SELECT o FROM OrderPO o WHERE o.status = :status " +
           "AND o.createdAt < :before")
    List<OrderPO> findTimeoutOrders(@Param("status") String status,
                                     @Param("before") LocalDateTime before,
                                     Pageable pageable);
}

// PO（持久化对象，与数据库表对应）
@Entity
@Table(name = "t_order")
@Data
public class OrderPO {
    @Id
    @Column(name = "order_id")
    private String orderId;

    @Column(name = "customer_id")
    private String customerId;

    @Column(name = "status")
    private String status;

    @Column(name = "total_amount")
    private BigDecimal totalAmount;

    @Column(name = "currency")
    private String currency;

    @Column(name = "shipping_province")
    private String shippingProvince;

    @Column(name = "shipping_city")
    private String shippingCity;

    @Column(name = "shipping_detail")
    private String shippingDetail;

    @Column(name = "receiver_name")
    private String receiverName;

    @Column(name = "receiver_phone")
    private String receiverPhone;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Version  // 乐观锁
    private int version;

    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER,
               orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderItemPO> items;
}

// ============ Converter（PO <-> 领域对象 互转）============
@Component
public class OrderConverter {

    // PO -> 领域对象（从数据库重建聚合）
    public Order toDomain(OrderPO po) {
        List<OrderItem> items = po.getItems().stream()
            .map(this::toOrderItem)
            .collect(Collectors.toList());

        Address address = new Address(
            po.getShippingProvince(), po.getShippingCity(),
            null, null, po.getShippingDetail(), null,
            po.getReceiverName(), po.getReceiverPhone());

        // 使用 package-private 构造器重建（或反射/Builder模式）
        return Order.reconstitute(
            OrderId.of(po.getOrderId()),
            CustomerId.of(po.getCustomerId()),
            items,
            address,
            OrderStatus.valueOf(po.getStatus()),
            new Money(po.getTotalAmount(), po.getCurrency()),
            po.getCreatedAt(),
            po.getVersion());
    }

    // 领域对象 -> PO（写入数据库）
    public OrderPO toPo(Order order) {
        OrderPO po = new OrderPO();
        po.setOrderId(order.getId().getValue());
        po.setCustomerId(order.getCustomerId().getValue());
        po.setStatus(order.getStatus().name());
        po.setTotalAmount(order.getTotalAmount().getAmount());
        po.setCurrency(order.getTotalAmount().getCurrency());
        po.setCreatedAt(order.getCreatedAt());
        po.setVersion(order.getVersion());

        Address addr = order.getShippingAddress();
        po.setShippingProvince(addr.getProvince());
        po.setShippingCity(addr.getCity());
        po.setShippingDetail(addr.getDetail());
        po.setReceiverName(addr.getReceiverName());
        po.setReceiverPhone(addr.getReceiverPhone());

        List<OrderItemPO> itemPOs = order.getItems().stream()
            .map(this::toOrderItemPo)
            .collect(Collectors.toList());
        po.setItems(itemPOs);

        return po;
    }

    private OrderItem toOrderItem(OrderItemPO po) {
        return OrderItem.reconstitute(
            OrderItemId.of(po.getItemId()),
            ProductId.of(po.getProductId()),
            po.getProductName(),
            new Money(po.getUnitPrice(), po.getCurrency()),
            po.getQuantity());
    }

    private OrderItemPO toOrderItemPo(OrderItem item) {
        OrderItemPO po = new OrderItemPO();
        po.setItemId(item.getId().getValue());
        po.setProductId(item.getProductId().getValue());
        po.setProductName(item.getProductName());
        po.setUnitPrice(item.getUnitPrice().getAmount());
        po.setCurrency(item.getUnitPrice().getCurrency());
        po.setQuantity(item.getQuantity());
        po.setSubtotal(item.getSubtotal().getAmount());
        return po;
    }
}
```

---

## 3.4 CQRS 在 DDD 中的应用

CQRS（Command Query Responsibility Segregation，命令查询职责分离）是一种与 DDD 高度匹配的架构模式。

```
CQRS 架构示意

              用户请求
                 |
       +---------+---------+
       |                   |
  写操作(Command)       读操作(Query)
       |                   |
       v                   v
+----------+         +----------+
| Command  |         |  Query   |
| Handler  |         | Handler  |
+----------+         +----------+
       |                   |
       v                   v
+----------+         +----------+
| 领域模型  |         | 查询模型  |
| (聚合根) |         |(扁平化VO)|
+----------+         +----------+
       |                   |
       v                   v
+----------+         +----------+
| 写数据库  |         | 读数据库  |
| (规范化)  |         |(可反规范化)|
+----------+         +----------+
                           ^
                           |
                     (同步/异步同步)
```

```java
// ============ CQRS 查询端实现 ============

// 查询条件对象（不可变）
public class OrderListQuery {
    private final CustomerId customerId;
    private final OrderStatus status;     // 可选过滤
    private final LocalDate startDate;   // 可选过滤
    private final LocalDate endDate;     // 可选过滤
    private final int page;
    private final int size;

    // 构造器（可用 Builder 模式）
    public static Builder builder(CustomerId customerId) {
        return new Builder(customerId);
    }

    public static class Builder {
        // ... Builder 实现
    }
}

// 查询结果 VO（专为读场景设计的扁平化结构）
public class OrderSummaryVO {
    private String orderId;
    private String status;
    private String statusText;
    private BigDecimal totalAmount;
    private String currency;
    private int itemCount;
    private String firstProductName;  // 显示第一件商品名称
    private LocalDateTime createdAt;
    // ... getter
}

// 查询 Handler（直接查询，不走聚合根，可跨聚合）
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class OrderQueryHandler {

    private final OrderQueryRepository queryRepository;  // 专用读库接口

    // 查询订单列表（跨多表 JOIN，不走聚合加载）
    public PageResult<OrderSummaryVO> queryList(OrderListQuery query) {
        return queryRepository.findOrderSummaries(query);
    }

    // 查询订单详情
    public OrderDetailVO getById(OrderId orderId) {
        return queryRepository.findOrderDetail(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }
}

// 查询专用 Repository（可直接使用 MyBatis/JDBC）
@Repository
public class OrderQueryRepositoryImpl implements OrderQueryRepository {

    private final OrderMapper orderMapper;

    @Override
    public PageResult<OrderSummaryVO> findOrderSummaries(OrderListQuery query) {
        // MyBatis 直接 SQL，可做各种优化（索引、缓存等）
        PageHelper.startPage(query.getPage(), query.getSize());
        List<OrderSummaryVO> list = orderMapper.selectOrderSummaries(
            query.getCustomerId().getValue(),
            query.getStatus() != null ? query.getStatus().name() : null,
            query.getStartDate(),
            query.getEndDate());
        PageInfo<OrderSummaryVO> pageInfo = new PageInfo<>(list);
        return PageResult.of(pageInfo.getList(), pageInfo.getTotal());
    }
}
```

## 3.5 六边形架构（Hexagonal Architecture）

六边形架构（也称端口-适配器架构）强调领域层与外部的完全隔离：

```
                        +-------------------+
                        |   六边形架构        |
                        +-------------------+

      +--------+             +-------+             +--------+
      |  REST  |             |  Web  |             |  gRPC  |
      |  Client|             |Browser|             | Client |
      +---+----+             +---+---+             +---+----+
          |                      |                     |
          | (Driving Adapters 主动适配器)                |
          v                      v                     v
    +-----------+          +-----------+         +-----------+
    | REST      |          |  Web      |         | gRPC      |
    | Adapter   |          | Adapter   |         | Adapter   |
    +-----------+          +-----------+         +-----------+
          |                      |                     |
          +----------------------+---------------------+
                                 |
                                 v
                    +========================+
                    |  INPUT PORT (接口)      |
                    | OrderApplicationPort   |
                    +========================+
                                 |
                    +========================+
                    |   APPLICATION CORE     |
                    |     (领域层 + 应用层)   |
                    |  Order / OrderService  |
                    +========================+
                                 |
                    +========================+
                    | OUTPUT PORT (接口)      |
                    | OrderRepository        |
                    | DomainEventPublisher   |
                    +========================+
                                 |
          +----------------------+---------------------+
          |                      |                     |
    +-----------+          +-----------+         +-----------+
    | JPA       |          |  Kafka    |         | Redis     |
    | Adapter   |          | Adapter   |         | Adapter   |
    +-----------+          +-----------+         +-----------+
          |                      |                     |
          v                      v                     v
      +-------+            +----------+           +-------+
      | MySQL |            | Kafka MQ |           | Redis |
      +-------+            +----------+           +-------+
               (Driven Adapters 被动适配器)
```

---

# Part 4: 聚合设计原则（重点）

## 4.1 聚合的本质：一致性边界

聚合不是简单的对象组合，而是**业务一致性边界**。所谓一致性，是指：
在任何时刻，聚合内的所有对象都必须满足业务不变性规则（Invariant）。

```
一致性边界的直观理解

+------------------------------------------+
|           Order 聚合（一致性边界）          |
|                                          |
|  Order（聚合根）                          |
|  - totalAmount = sum(items.subtotal)     |  <- 这个不变性必须
|  - status 状态机约束                     |     在聚合内部时刻保证
|                                          |
|  OrderItem[] items                       |
|  - quantity > 0                          |
|  - unitPrice > 0                         |
|  - subtotal = quantity * unitPrice       |
|                                          |
|  一旦修改任何 item 的 quantity，           |
|  totalAmount 必须同步更新。               |
|  这个操作必须在同一个事务内完成。           |
+------------------------------------------+

外部的 Inventory 聚合不需要和 Order 强一致！
可以通过领域事件+最终一致性处理。
```

## 4.2 聚合设计四大原则

### 原则一：聚合内部强一致性

```java
// 正确做法：在聚合根内部维护一致性
public class Order extends AggregateRoot<OrderId> {

    private List<OrderItem> items;
    private Money totalAmount;  // 必须与 items 保持一致

    // 添加商品时，聚合根同步更新总价
    public void addItem(ProductId productId, String productName,
                         Money unitPrice, int quantity) {
        // 检查是否重复商品
        items.stream()
            .filter(i -> i.getProductId().equals(productId))
            .findFirst()
            .ifPresentOrElse(
                existing -> existing.increaseQuantity(quantity),
                () -> items.add(new OrderItem(productId, productName,
                                               unitPrice, quantity))
            );
        // 关键：总价与商品列表同步更新
        this.totalAmount = recalculateTotal();
    }

    // 移除商品时，聚合根同步更新总价
    public void removeItem(ProductId productId) {
        boolean removed = items.removeIf(
            i -> i.getProductId().equals(productId));
        if (!removed) {
            throw new OrderDomainException("商品不存在: " + productId);
        }
        if (items.isEmpty()) {
            throw new OrderDomainException("订单至少要保留一个商品");
        }
        this.totalAmount = recalculateTotal();
    }

    private Money recalculateTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

### 原则二：聚合间最终一致性（通过领域事件）

```java
// 正确做法：跨聚合用领域事件实现最终一致性
// 订单下单 -> 发布事件 -> 库存服务异步消费 -> 扣减库存

// 领域层：Order 聚合发布事件
public class Order extends AggregateRoot<OrderId> {
    public void pay(PaymentId paymentId, Money paidAmount) {
        // 检查...（省略）
        this.status = OrderStatus.PAID;
        // 发布领域事件，通知其他上下文
        registerEvent(new OrderPaidEvent(orderId, paymentId, items));
    }
}

// 库存上下文：消费 OrderPaidEvent
@Component
public class OrderPaidEventHandler {

    private final InventoryRepository inventoryRepository;

    @EventHandler  // 或 @KafkaListener
    @Transactional
    public void handle(OrderPaidEvent event) {
        // 库存服务在自己的事务中扣减库存
        for (OrderPaidEvent.OrderItem item : event.getItems()) {
            Inventory inventory = inventoryRepository
                .findByProductId(item.getProductId())
                .orElseThrow();
            inventory.deduct(item.getQuantity());  // 库存聚合内部的业务方法
            inventoryRepository.save(inventory);
        }
    }
}

// 错误做法：在同一个事务中修改多个聚合（强耦合！）
// @Transactional
// public void placeOrder(...) {
//     Order order = Order.create(...);
//     orderRepository.save(order);
//
//     // 错误！在同一事务里修改另一个聚合
//     Inventory inventory = inventoryRepository.findById(...);
//     inventory.deduct(quantity);
//     inventoryRepository.save(inventory);
// }
```

### 原则三：聚合尽量小

```
错误的大聚合设计（反模式）
+--------------------------------------------------+
|                   Customer 大聚合                  |
| Customer（聚合根）                                 |
|   - personalInfo（姓名/手机/邮箱）                  |
|   - List<Address> addresses（收货地址列表）          |
|   - List<Order> orders  <-- 错误！订单不属于这里    |
|   - List<Review> reviews <-- 错误！评论不属于这里   |
|   - List<Coupon> coupons <-- 错误！优惠券不属于这里  |
|   - MemberInfo memberInfo                         |
|   - List<Points> pointsHistory                   |
+--------------------------------------------------+
问题：
1. 加载 Customer 需要 JOIN 几十张表，性能极差
2. 并发修改同一 Customer 的不同属性会产生大量冲突
3. 难以维护，职责不清

正确的小聚合设计
+------------+   +----------+   +-----------+
| Customer   |   |  Order   |   |  Review   |
| 聚合         |   |  聚合     |   |  聚合     |
| - name     |   | - items  |   | - rating  |
| - phone    |   | - status |   | - content |
| - email    |   | - total  |   | - orderId |
+------------+   +----------+   +-----------+
                 customerId      customerId
                 (引用，不是对象)  (引用，不是对象)
```

### 原则四：通过 ID 引用其他聚合

```java
// 错误做法：直接持有另一个聚合的对象引用
public class Order {
    private Customer customer;  // 错误！直接持有 Customer 对象
    // 问题：
    // 1. 加载 Order 时会联级加载整个 Customer 聚合
    // 2. 两个聚合的生命周期耦合
    // 3. 破坏聚合边界
}

// 正确做法：只持有 ID（值对象）
public class Order {
    private CustomerId customerId;  // 正确！只持有 ID

    // 如果需要客户信息，在应用服务中加载
    // 而不是在聚合内直接访问
}

// 应用服务中需要客户信息时：
@Service
public class PlaceOrderHandler {
    private final OrderRepository orderRepository;
    private final CustomerRepository customerRepository;  // 在应用层加载

    public void handle(PlaceOrderCommand command) {
        // 在应用层分别加载两个聚合
        Customer customer = customerRepository
            .findById(command.getCustomerId()).orElseThrow();

        // 将必要信息传入 Order 的业务方法
        Order order = Order.create(
            customer.getId(),
            command.getItems(),
            customer.getDefaultAddress());  // 传值，不传引用

        orderRepository.save(order);
    }
}
```

---

## 4.3 不变性规则（Invariant）实现

```java
// ============ 聚合根中的不变性规则 ============
public class Order extends AggregateRoot<OrderId> {

    private static final int MAX_ITEMS = 100;
    private static final Money MAX_ORDER_AMOUNT =
        new Money(new BigDecimal("99999.99"), "CNY");
    private static final Money MIN_ORDER_AMOUNT =
        new Money(new BigDecimal("0.01"), "CNY");

    // 创建时验证所有不变性
    public static Order create(CustomerId customerId,
                               List<OrderItem> items,
                               Address shippingAddress) {
        // 不变性规则 1：客户ID不能为空
        Objects.requireNonNull(customerId, "客户ID不能为空");

        // 不变性规则 2：订单必须有商品
        if (items == null || items.isEmpty()) {
            throw new OrderDomainException("订单必须包含至少一个商品");
        }

        // 不变性规则 3：商品数量限制
        if (items.size() > MAX_ITEMS) {
            throw new OrderDomainException(
                "单笔订单最多" + MAX_ITEMS + "种商品，当前：" + items.size());
        }

        // 不变性规则 4：收货地址不能为空
        Objects.requireNonNull(shippingAddress, "收货地址不能为空");

        Order order = new Order();
        order.orderId = OrderId.generate();
        order.customerId = customerId;
        order.items = new ArrayList<>(items);
        order.shippingAddress = shippingAddress;
        order.status = OrderStatus.PENDING;
        order.createdAt = LocalDateTime.now();
        order.totalAmount = order.calculateTotal();

        // 不变性规则 5：订单金额范围
        if (order.totalAmount.isNegativeOrZero()) {
            throw new OrderDomainException("订单金额必须大于0");
        }
        if (order.totalAmount.isGreaterThan(MAX_ORDER_AMOUNT)) {
            throw new OrderDomainException(
                "单笔订单金额不能超过" + MAX_ORDER_AMOUNT);
        }

        order.registerEvent(new OrderCreatedEvent(
            order.orderId, customerId, order.totalAmount,
            order.createdAt));
        return order;
    }

    // 每次状态变更也要验证不变性
    public void addItem(OrderItem newItem) {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("只有待支付订单才能修改商品");
        }
        if (items.size() >= MAX_ITEMS) {
            throw new OrderDomainException("商品数量已达上限");
        }
        items.add(newItem);
        this.totalAmount = calculateTotal();
        // 重新验证金额不变性
        if (totalAmount.isGreaterThan(MAX_ORDER_AMOUNT)) {
            // 回滚
            items.remove(newItem);
            this.totalAmount = calculateTotal();
            throw new OrderDomainException("添加商品后订单金额超限");
        }
    }
}
```

---

## 4.4 乐观锁与聚合并发控制

```java
// ============ 乐观锁实现 ============

// 聚合根包含版本字段
public class Order extends AggregateRoot<OrderId> {
    @Version  // JPA 乐观锁注解（在 PO 上）
    private int version;

    // 聚合根暴露版本（只读）
    public int getVersion() { return version; }
}

// PO 中的 @Version 注解（实际乐观锁在 JPA 层）
@Entity
@Table(name = "t_order")
public class OrderPO {
    @Id
    private String orderId;

    @Version  // JPA 自动处理版本检查
    private int version;

    // ... 其他字段
}

// 测试乐观锁场景
@SpringBootTest
public class OrderConcurrencyTest {

    @Test
    public void testOptimisticLockConflict() throws InterruptedException {
        // 创建订单
        OrderId orderId = createTestOrder();

        CountDownLatch latch = new CountDownLatch(2);
        AtomicInteger successCount = new AtomicInteger(0);
        AtomicInteger failCount = new AtomicInteger(0);

        // 两个线程同时取消同一订单
        for (int i = 0; i < 2; i++) {
            final int idx = i;
            new Thread(() -> {
                try {
                    cancelOrderHandler.handle(
                        new CancelOrderCommand(orderId, "测试取消" + idx));
                    successCount.incrementAndGet();
                } catch (ObjectOptimisticLockingFailureException e) {
                    // 其中一个会抛出乐观锁异常
                    failCount.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            }).start();
        }

        latch.await(5, TimeUnit.SECONDS);
        // 只有一个成功，另一个失败
        assertEquals(1, successCount.get());
        assertEquals(1, failCount.get());
    }
}

// 处理乐观锁冲突：重试
@Service
public class CancelOrderHandler {

    @Retryable(
        value = {ObjectOptimisticLockingFailureException.class},
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2))
    @Transactional
    public void handle(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId())
            .orElseThrow();
        order.cancel(command.getReason());
        orderRepository.save(order);
        eventPublisher.publishAll(order.getDomainEvents());
    }

    @Recover
    public void recover(ObjectOptimisticLockingFailureException ex,
                         CancelOrderCommand command) {
        throw new OrderDomainException("订单取消失败，请重试", ex);
    }
}
```

---

# Part 5: 领域事件（Domain Event）

## 5.1 领域事件定义与设计原则

领域事件代表**业务领域中发生的、有意义的状态变化**。

```
领域事件的特征
+------------------------------------------+
|              领域事件特征                  |
+------------------------------------------+
| 1. 命名：过去式动词（OrderPlaced,          |
|          OrderCancelled, PaymentReceived）|
|                                          |
| 2. 不可变：事件发生后不可更改               |
|                                          |
| 3. 有时间戳：记录事件发生时间               |
|                                          |
| 4. 携带足够信息：消费者不需要回查原聚合       |
|                                          |
| 5. 表达业务意图，不是技术意图               |
|    好：OrderPaid  坏：OrderStatusUpdated  |
+------------------------------------------+
```

## 5.2 领域事件 vs 系统事件

```
+-------------------+------------------------+
|    领域事件         |       系统事件          |
+-------------------+------------------------+
| 表达业务语义        | 表达技术操作             |
| OrderPlaced       | RecordInserted         |
| PaymentReceived   | CacheInvalidated       |
| ItemShipped       | DatabaseUpdated        |
+-------------------+------------------------+
| 有业务价值          | 无直接业务价值           |
| 领域专家能理解       | 领域专家不关心           |
+-------------------+------------------------+
| 在领域层定义        | 在基础设施层定义          |
+-------------------+------------------------+
```

## 5.3 同步发布（Spring ApplicationEvent）

```java
// ============ Spring 同步事件 ============

// 领域事件基类（实现 ApplicationEvent 或不实现，两种方式都可）
public abstract class DomainEvent {
    private final String eventId;
    private final LocalDateTime occurredOn;

    protected DomainEvent() {
        this.eventId = UUID.randomUUID().toString();
        this.occurredOn = LocalDateTime.now();
    }

    public String getEventId() { return eventId; }
    public LocalDateTime getOccurredOn() { return occurredOn; }
}

// Spring 事件发布实现
@Component
public class SpringDomainEventPublisher implements DomainEventPublisher {

    private final ApplicationEventPublisher publisher;

    @Override
    public void publish(DomainEvent event) {
        publisher.publishEvent(event);
    }

    @Override
    public void publishAll(List<DomainEvent> events) {
        events.forEach(this::publish);
    }
}

// 同步事件监听器（同一事务内）
@Component
@Transactional(propagation = Propagation.MANDATORY)
public class OrderCreatedEventHandler {

    private final OutboxRepository outboxRepository;

    // @TransactionalEventListener 在事务提交后触发
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void handle(OrderCreatedEvent event) {
        // 在事务提交后发送通知，避免事务回滚后通知已发
        notificationService.sendOrderCreatedNotice(event.getCustomerId());
    }

    // 保证事件可靠发布：写入发件箱（Outbox Pattern）
    @EventListener
    @Transactional(propagation = Propagation.MANDATORY)
    public void saveToOutbox(OrderCreatedEvent event) {
        // 在同一事务内，将事件写入发件箱表
        OutboxMessage message = new OutboxMessage(
            event.getEventId(),
            "ORDER_CREATED",
            serializeEvent(event),
            LocalDateTime.now());
        outboxRepository.save(message);
        // 定时任务轮询发件箱表，确保事件最终发出
    }
}
```

## 5.4 异步发布（Kafka 可靠消息）

```java
// ============ Outbox Pattern（发件箱模式）实现可靠消息 ============

// 发件箱消息 PO
@Entity
@Table(name = "t_outbox_message")
public class OutboxMessagePO {
    @Id
    private String messageId;
    private String eventType;
    private String payload;          // JSON 序列化的事件
    private String aggregateId;
    private String aggregateType;
    private LocalDateTime createdAt;
    private LocalDateTime processedAt;
    private String status;           // PENDING / PROCESSED / FAILED
    private int retryCount;
}

// 发件箱轮询任务：将待发消息推送到 Kafka
@Component
@RequiredArgsConstructor
public class OutboxPollingPublisher {

    private final OutboxRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Scheduled(fixedDelay = 1000)  // 每1秒轮询一次
    @Transactional
    public void publishPendingMessages() {
        List<OutboxMessagePO> messages =
            outboxRepository.findPendingMessages(100);  // 每次最多100条

        for (OutboxMessagePO msg : messages) {
            try {
                // 发送到 Kafka
                kafkaTemplate.send(
                    getTopicByEventType(msg.getEventType()),
                    msg.getAggregateId(),
                    msg.getPayload())
                .get(5, TimeUnit.SECONDS);  // 等待确认

                // 标记为已处理
                msg.setStatus("PROCESSED");
                msg.setProcessedAt(LocalDateTime.now());
            } catch (Exception e) {
                msg.setStatus("FAILED");
                msg.setRetryCount(msg.getRetryCount() + 1);
                log.error("发件箱消息发送失败: {}", msg.getMessageId(), e);
            }
            outboxRepository.save(msg);
        }
    }

    private String getTopicByEventType(String eventType) {
        return switch (eventType) {
            case "ORDER_CREATED" -> "order.created";
            case "ORDER_PAID" -> "order.paid";
            case "ORDER_CANCELLED" -> "order.cancelled";
            default -> "order.events";
        };
    }
}

// Kafka 消费者（库存服务消费 OrderPaid 事件）
@Component
@Slf4j
public class OrderPaidEventConsumer {

    private final InventoryApplicationService inventoryService;
    private final ObjectMapper objectMapper;

    @KafkaListener(
        topics = "order.paid",
        groupId = "inventory-service",
        containerFactory = "kafkaListenerContainerFactory")
    public void consume(ConsumerRecord<String, String> record) {
        try {
            OrderPaidEventDTO event = objectMapper.readValue(
                record.value(), OrderPaidEventDTO.class);

            log.info("收到订单支付事件: orderId={}", event.getOrderId());

            // 幂等处理（根据 eventId 去重）
            if (inventoryService.isEventProcessed(event.getEventId())) {
                log.info("事件已处理，跳过: {}", event.getEventId());
                return;
            }

            // 扣减库存
            inventoryService.deductStock(
                DeductStockCommand.fromEvent(event));

        } catch (Exception e) {
            log.error("处理订单支付事件失败", e);
            throw new RuntimeException(e);  // 触发重试
        }
    }
}
```

## 5.5 Saga 模式：跨聚合的最终一致性

```java
// ============ Saga 编排模式（Orchestration Saga）============
// 场景：下单 -> 扣库存 -> 扣支付 -> 确认订单
// 任何一步失败都需要补偿（回滚）

@Component
@Slf4j
public class PlaceOrderSaga {

    private final OrderRepository orderRepository;
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final DomainEventPublisher eventPublisher;

    // Saga 步骤1：创建订单
    @EventHandler
    @Transactional
    public void onOrderCreated(OrderCreatedEvent event) {
        log.info("Saga step 1: 订单已创建 {}", event.getOrderId());
        // 发送库存预留命令
        inventoryClient.reserveStock(
            new ReserveStockCommand(event.getOrderId(), event.getItems()));
    }

    // Saga 步骤2：库存预留成功，发起支付
    @EventHandler
    @Transactional
    public void onStockReserved(StockReservedEvent event) {
        log.info("Saga step 2: 库存已预留 orderId={}", event.getOrderId());
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        paymentClient.initiatePayment(
            new InitiatePaymentCommand(event.getOrderId(), order.getTotalAmount()));
    }

    // Saga 步骤3：支付成功，确认订单
    @EventHandler
    @Transactional
    public void onPaymentSuccess(PaymentSuccessEvent event) {
        log.info("Saga step 3: 支付成功 orderId={}", event.getOrderId());
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.pay(event.getPaymentId(), event.getAmount());
        orderRepository.save(order);
        eventPublisher.publishAll(order.getDomainEvents());
    }

    // 补偿事务1：库存预留失败 -> 取消订单
    @EventHandler
    @Transactional
    public void onStockReservationFailed(StockReservationFailedEvent event) {
        log.warn("Saga 补偿: 库存不足，取消订单 {}", event.getOrderId());
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.cancel("库存不足，系统自动取消");
        orderRepository.save(order);
        eventPublisher.publishAll(order.getDomainEvents());
    }

    // 补偿事务2：支付失败 -> 释放库存 + 取消订单
    @EventHandler
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        log.warn("Saga 补偿: 支付失败，释放库存并取消订单 {}", event.getOrderId());

        // 补偿1：释放已预留库存
        inventoryClient.releaseStock(event.getOrderId());

        // 补偿2：取消订单
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.cancel("支付失败，系统自动取消");
        orderRepository.save(order);
        eventPublisher.publishAll(order.getDomainEvents());
    }
}
```

## 5.6 事件溯源（Event Sourcing）简介

```
传统持久化 vs 事件溯源

传统方式：存储当前状态
+----------+----------+-----------+----------+
| order_id | status   | total_amt | updated  |
+----------+----------+-----------+----------+
| ORD-001  | SHIPPED  | 299.00    | 2024-1-3 |
+----------+----------+-----------+----------+
只能看到"现在是什么"，无法知道"怎么到达这个状态"

事件溯源：存储所有事件
+----------+-------------------+----------+---------+
| order_id | event_type        | payload  | occurred|
+----------+-------------------+----------+---------+
| ORD-001  | ORDER_CREATED     | {...}    | 2024-1-1|
| ORD-001  | ITEM_ADDED        | {...}    | 2024-1-1|
| ORD-001  | ORDER_PAID        | {...}    | 2024-1-2|
| ORD-001  | ORDER_SHIPPED     | {...}    | 2024-1-3|
+----------+-------------------+----------+---------+
当前状态 = 重放所有事件的结果
```

```java
// ============ Event Sourcing 聚合根 ============
public class OrderES {  // ES = Event Sourced
    private OrderId orderId;
    private OrderStatus status;
    private Money totalAmount;
    private List<OrderItem> items = new ArrayList<>();

    // 重放事件重建状态
    public static OrderES reconstitute(List<DomainEvent> events) {
        OrderES order = new OrderES();
        events.forEach(order::apply);
        return order;
    }

    // 业务方法：生成事件，不直接修改状态
    public List<DomainEvent> pay(PaymentId paymentId, Money amount) {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("状态不允许支付");
        }
        return List.of(new OrderPaidEvent(orderId, paymentId, amount));
    }

    // apply 方法：消费事件，更新状态
    private void apply(DomainEvent event) {
        if (event instanceof OrderCreatedEvent e) {
            this.orderId = e.getOrderId();
            this.status = OrderStatus.PENDING;
            this.totalAmount = e.getTotalAmount();
            this.items = new ArrayList<>(e.getItems());
        } else if (event instanceof OrderPaidEvent e) {
            this.status = OrderStatus.PAID;
        } else if (event instanceof OrderCancelledEvent e) {
            this.status = OrderStatus.CANCELLED;
        }
        // ... 其他事件
    }
}
```

---

# Part 6: Repository 模式实现

## 6.1 Repository 的设计原则

```
Repository 的职责边界
+------------------------------------------+
|             Repository 是什么              |
+------------------------------------------+
| 对外（领域层视角）：                        |
|   - 聚合的内存集合（假装所有对象都在内存里）   |
|   - 隐藏持久化细节                         |
|   - 以聚合根为边界（不是以数据库表为边界）     |
|                                          |
| 对内（基础设施视角）：                      |
|   - 实现领域层定义的接口                    |
|   - 负责 ORM 映射、SQL、缓存等技术细节       |
+------------------------------------------+

Repository 不是 DAO！
DAO（Data Access Object）是以数据库表为中心的。
Repository 是以聚合根为中心的。

OrderRepository.findById()        // 返回 Order 聚合
OrderItemDAO.findByOrderId()      // 返回数据行，不是聚合
```

## 6.2 Repository 接口（领域层）

```java
// ============ 领域层 Repository 接口 ============
// 注意：这个接口在 domain 包下，不依赖任何框架

public interface OrderRepository {

    /**
     * 根据聚合根ID加载订单聚合
     * Repository 的最核心方法
     */
    Optional<Order> findById(OrderId orderId);

    /**
     * 保存订单（新增/更新，由实现决定）
     */
    void save(Order order);

    /**
     * 根据客户查询订单列表
     * 注意：只放在领域层需要的查询，
     * 复杂查询放到 QueryRepository（CQRS 读端）
     */
    List<Order> findByCustomerId(CustomerId customerId);

    /**
     * 查找需要自动取消的超时订单
     * 供定时任务使用
     */
    List<Order> findPendingOrdersBefore(LocalDateTime time, int limit);

    /**
     * 检查订单是否存在（避免加载整个聚合）
     */
    boolean existsById(OrderId orderId);
}
```

## 6.3 基于 JPA 的 Repository 实现

```java
// ============ JPA 实现（基础设施层）============

@Repository
@RequiredArgsConstructor
public class OrderRepositoryJpa implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderPOConverter converter;

    @Override
    public Optional<Order> findById(OrderId orderId) {
        return jpaRepository.findById(orderId.getValue())
            .map(converter::toDomain);
    }

    @Override
    public void save(Order order) {
        OrderPO po = converter.toPo(order);
        jpaRepository.save(po);
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return jpaRepository
            .findByCustomerIdOrderByCreatedAtDesc(customerId.getValue())
            .stream()
            .map(converter::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public List<Order> findPendingOrdersBefore(LocalDateTime time, int limit) {
        Pageable pageable = PageRequest.of(0, limit);
        return jpaRepository
            .findByStatusAndCreatedAtBefore("PENDING", time, pageable)
            .stream()
            .map(converter::toDomain)
            .collect(Collectors.toList());
    }

    @Override
    public boolean existsById(OrderId orderId) {
        return jpaRepository.existsById(orderId.getValue());
    }
}

// Spring Data JPA 接口
@Repository
public interface OrderJpaRepository extends JpaRepository<OrderPO, String>,
                                             JpaSpecificationExecutor<OrderPO> {
    List<OrderPO> findByCustomerIdOrderByCreatedAtDesc(String customerId);

    @Query("SELECT o FROM OrderPO o WHERE o.status = :status " +
           "AND o.createdAt < :before ORDER BY o.createdAt")
    List<OrderPO> findByStatusAndCreatedAtBefore(
        @Param("status") String status,
        @Param("before") LocalDateTime before,
        Pageable pageable);
}
```

## 6.4 基于 MyBatis 的 Repository 实现

```java
// ============ MyBatis 实现（基础设施层）============

@Repository
@RequiredArgsConstructor
public class OrderRepositoryMyBatis implements OrderRepository {

    private final OrderMapper orderMapper;
    private final OrderItemMapper orderItemMapper;
    private final OrderDomainConverter converter;

    @Override
    public Optional<Order> findById(OrderId orderId) {
        OrderDO orderDO = orderMapper.selectById(orderId.getValue());
        if (orderDO == null) {
            return Optional.empty();
        }
        List<OrderItemDO> itemDOs =
            orderItemMapper.selectByOrderId(orderId.getValue());
        return Optional.of(converter.toDomain(orderDO, itemDOs));
    }

    @Override
    @Transactional
    public void save(Order order) {
        OrderDO orderDO = converter.toOrderDO(order);
        List<OrderItemDO> itemDOs = converter.toOrderItemDOs(order);

        // 判断是新增还是更新
        if (orderMapper.existsById(order.getId().getValue())) {
            // 更新（带乐观锁）
            int affected = orderMapper.updateWithVersion(orderDO);
            if (affected == 0) {
                throw new OptimisticLockException(
                    "订单[" + order.getId() + "]更新失败，版本冲突");
            }
            // 删除旧的订单项，插入新的
            orderItemMapper.deleteByOrderId(order.getId().getValue());
            if (!itemDOs.isEmpty()) {
                orderItemMapper.batchInsert(itemDOs);
            }
        } else {
            // 新增
            orderMapper.insert(orderDO);
            if (!itemDOs.isEmpty()) {
                orderItemMapper.batchInsert(itemDOs);
            }
        }
    }
}

// MyBatis Mapper 接口
@Mapper
public interface OrderMapper {
    OrderDO selectById(String orderId);
    int insert(OrderDO orderDO);
    int updateWithVersion(OrderDO orderDO);  // WHERE version = #{version}
    boolean existsById(String orderId);
}
```

```xml
<!-- OrderMapper.xml -->
<mapper namespace="com.example.infrastructure.mapper.OrderMapper">

    <resultMap id="orderResultMap" type="OrderDO">
        <id property="orderId" column="order_id"/>
        <result property="customerId" column="customer_id"/>
        <result property="status" column="status"/>
        <result property="totalAmount" column="total_amount"/>
        <result property="currency" column="currency"/>
        <result property="version" column="version"/>
        <result property="createdAt" column="created_at"/>
    </resultMap>

    <select id="selectById" resultMap="orderResultMap">
        SELECT order_id, customer_id, status, total_amount, currency,
               version, created_at
        FROM t_order
        WHERE order_id = #{orderId}
    </select>

    <insert id="insert" parameterType="OrderDO">
        INSERT INTO t_order (order_id, customer_id, status, total_amount,
                             currency, version, created_at)
        VALUES (#{orderId}, #{customerId}, #{status}, #{totalAmount},
                #{currency}, 0, #{createdAt})
    </insert>

    <!-- 乐观锁更新 -->
    <update id="updateWithVersion" parameterType="OrderDO">
        UPDATE t_order
        SET status = #{status},
            total_amount = #{totalAmount},
            version = version + 1,
            updated_at = NOW()
        WHERE order_id = #{orderId}
          AND version = #{version}
    </update>

</mapper>
```

## 6.5 领域对象与持久化对象的转换

```
DO / PO / DTO 的区别（常见混淆点）

+------+------------------+---------------------------+
| 缩写  | 全称              | 用途                      |
+------+------------------+---------------------------+
| DO   | Domain Object    | 领域对象（Entity/ValueObj） |
|      |                  | 含业务行为，是DDD的核心      |
+------+------------------+---------------------------+
| PO   | Persistent Object| 持久化对象                 |
|      |                  | 与数据库表字段一一对应       |
|      |                  | 只有数据，无业务行为         |
+------+------------------+---------------------------+
| DTO  | Data Transfer    | 数据传输对象               |
|      | Object           | 用于 API 接口的入参/出参    |
|      |                  | 根据视图需求设计，非领域模型 |
+------+------------------+---------------------------+
| VO   | View Object      | 视图对象                   |
|      |                  | 返回给前端的展示数据         |
+------+------------------+---------------------------+

数据流：
DTO -> (Assembler) -> Command -> (Handler) -> DO -> (Converter) -> PO -> DB
DB -> PO -> (Converter) -> DO -> (Assembler) -> VO -> DTO
```

```java
// ============ 完整的转换器实现 ============
@Component
public class OrderDomainConverter {

    // PO -> 领域对象（从数据库重建）
    public Order toDomain(OrderPO orderPO, List<OrderItemPO> itemPOs) {
        // 重建值对象
        Money totalAmount = new Money(
            orderPO.getTotalAmount(), orderPO.getCurrency());

        Address shippingAddress = new Address(
            orderPO.getShippingProvince(),
            orderPO.getShippingCity(),
            orderPO.getShippingDistrict(),
            orderPO.getShippingStreet(),
            orderPO.getShippingDetail(),
            orderPO.getShippingZipCode(),
            orderPO.getReceiverName(),
            orderPO.getReceiverPhone());

        // 重建实体列表
        List<OrderItem> items = itemPOs.stream()
            .map(this::toOrderItem)
            .collect(Collectors.toList());

        // 使用 reconstitute 工厂方法重建聚合根
        // （不触发构造器中的业务验证和事件发布）
        return Order.reconstitute(
            OrderId.of(orderPO.getOrderId()),
            CustomerId.of(orderPO.getCustomerId()),
            items,
            shippingAddress,
            OrderStatus.valueOf(orderPO.getStatus()),
            totalAmount,
            orderPO.getCreatedAt(),
            orderPO.getVersion());
    }

    // 领域对象 -> PO（写入数据库）
    public OrderPO toOrderPO(Order order) {
        OrderPO po = new OrderPO();
        po.setOrderId(order.getId().getValue());
        po.setCustomerId(order.getCustomerId().getValue());
        po.setStatus(order.getStatus().name());
        po.setTotalAmount(order.getTotalAmount().getAmount());
        po.setCurrency(order.getTotalAmount().getCurrency());
        po.setVersion(order.getVersion());
        po.setCreatedAt(order.getCreatedAt());

        Address addr = order.getShippingAddress();
        po.setShippingProvince(addr.getProvince());
        po.setShippingCity(addr.getCity());
        po.setShippingDistrict(addr.getDistrict());
        po.setShippingDetail(addr.getDetail());
        po.setReceiverName(addr.getReceiverName());
        po.setReceiverPhone(addr.getReceiverPhone());

        return po;
    }

    private OrderItem toOrderItem(OrderItemPO po) {
        return OrderItem.reconstitute(
            OrderItemId.of(po.getItemId()),
            ProductId.of(po.getProductId()),
            po.getProductName(),
            new Money(po.getUnitPrice(), po.getCurrency()),
            po.getQuantity());
    }
}
```

## 6.6 Specification 模式（复杂查询规格）

```java
// ============ Specification 模式 ============

// 规格接口
public interface Specification<T> {
    boolean isSatisfiedBy(T entity);
    Specification<T> and(Specification<T> other);
    Specification<T> or(Specification<T> other);
    Specification<T> not();
}

// 抽象规格基类
public abstract class AbstractSpecification<T> implements Specification<T> {
    @Override
    public Specification<T> and(Specification<T> other) {
        return new AndSpecification<>(this, other);
    }

    @Override
    public Specification<T> or(Specification<T> other) {
        return new OrSpecification<>(this, other);
    }

    @Override
    public Specification<T> not() {
        return new NotSpecification<>(this);
    }
}

// 具体规格：可取消的订单
public class CancellableOrderSpecification extends AbstractSpecification<Order> {
    @Override
    public boolean isSatisfiedBy(Order order) {
        return order.getStatus().canCancel()
            && order.getCreatedAt().isAfter(
                LocalDateTime.now().minusDays(30));  // 30天内的才能取消
    }
}

// JPA Specification（用于 JpaSpecificationExecutor）
public class OrderSpecifications {

    public static org.springframework.data.jpa.domain.Specification<OrderPO>
            byCustomerId(String customerId) {
        return (root, query, cb) ->
            cb.equal(root.get("customerId"), customerId);
    }

    public static org.springframework.data.jpa.domain.Specification<OrderPO>
            byStatus(String status) {
        return (root, query, cb) ->
            cb.equal(root.get("status"), status);
    }

    public static org.springframework.data.jpa.domain.Specification<OrderPO>
            createdAfter(LocalDateTime time) {
        return (root, query, cb) ->
            cb.greaterThan(root.get("createdAt"), time);
    }
}

// 使用 Specification 组合查询
@Repository
public class OrderRepositoryJpa implements OrderRepository {

    public List<Order> findByConditions(OrderQuery query) {
        var spec = Specification.where(
                OrderSpecifications.byCustomerId(query.getCustomerId()))
            .and(query.getStatus() != null
                ? OrderSpecifications.byStatus(query.getStatus().name())
                : null)
            .and(query.getStartDate() != null
                ? OrderSpecifications.createdAfter(
                    query.getStartDate().atStartOfDay())
                : null);

        return jpaRepository.findAll(spec).stream()
            .map(converter::toDomain)
            .collect(Collectors.toList());
    }
}
```

---

# Part 7: CQRS 实现

## 7.1 CQRS 核心理念

```
CQRS 命令查询职责分离

+---------------------------------------------------+
|                  CQRS 核心原则                     |
+---------------------------------------------------+
|                                                   |
|  Command（命令）：                                  |
|  - 改变系统状态                                    |
|  - 不返回数据（或只返回 ID）                        |
|  - 有业务语义（PlaceOrder, CancelOrder）            |
|  - 经过领域模型处理，确保业务规则                    |
|                                                   |
|  Query（查询）：                                   |
|  - 不改变系统状态                                  |
|  - 返回数据                                        |
|  - 可以绕过领域模型，直接查询（性能优先）             |
|  - 可以跨聚合、跨上下文                             |
|                                                   |
+---------------------------------------------------+

为什么分离：
1. 写：一致性优先，走领域模型，有复杂的业务规则
2. 读：性能优先，扁平化 SQL，可以用物化视图/缓存
3. 两者读写比例通常是 1:10~1:100，需求完全不同
```

## 7.2 命令端实现（Command Side）

```java
// ============ 下单命令完整实现 ============

// Command（命令对象，不可变）
public final class PlaceOrderCommand {
    private final String commandId;        // 幂等ID
    private final CustomerId customerId;
    private final List<OrderItemCommand> items;
    private final AddressCommand shippingAddress;
    private final String couponCode;       // 可选

    @Builder
    public PlaceOrderCommand(String commandId,
                              CustomerId customerId,
                              List<OrderItemCommand> items,
                              AddressCommand shippingAddress,
                              String couponCode) {
        this.commandId = commandId != null
            ? commandId : UUID.randomUUID().toString();
        this.customerId = Objects.requireNonNull(customerId, "客户ID不能为空");
        this.items = Collections.unmodifiableList(
            Objects.requireNonNull(items, "订单商品不能为空"));
        this.shippingAddress = Objects.requireNonNull(
            shippingAddress, "收货地址不能为空");
        this.couponCode = couponCode;
    }

    public record OrderItemCommand(ProductId productId, int quantity) {
        public OrderItemCommand {
            Objects.requireNonNull(productId, "商品ID不能为空");
            if (quantity <= 0) throw new IllegalArgumentException("数量必须大于0");
        }
    }

    public record AddressCommand(
            String province, String city, String district,
            String street, String detail, String zipCode,
            String receiverName, String receiverPhone) {}

    // getter
    public String getCommandId() { return commandId; }
    public CustomerId getCustomerId() { return customerId; }
    public List<OrderItemCommand> getItems() { return items; }
    public AddressCommand getShippingAddress() { return shippingAddress; }
    public String getCouponCode() { return couponCode; }
}

// Command Handler（应用服务）
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class PlaceOrderCommandHandler {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final CouponRepository couponRepository;
    private final OrderPricingService pricingService;
    private final DomainEventPublisher eventPublisher;
    private final IdempotencyService idempotencyService;

    public OrderId handle(PlaceOrderCommand command) {
        // 1. 幂等检查（防止重复下单）
        if (idempotencyService.isDuplicate(command.getCommandId())) {
            log.info("重复命令，返回已有结果: {}", command.getCommandId());
            return idempotencyService.getResult(command.getCommandId(), OrderId.class);
        }

        // 2. 加载商品（应用层协调，不在领域层）
        List<OrderItem> items = loadOrderItems(command.getItems());

        // 3. 构建地址值对象
        AddressCommand addrCmd = command.getShippingAddress();
        Address address = new Address(
            addrCmd.province(), addrCmd.city(), addrCmd.district(),
            addrCmd.street(), addrCmd.detail(), addrCmd.zipCode(),
            addrCmd.receiverName(), addrCmd.receiverPhone());

        // 4. 创建聚合根（业务规则在聚合内部执行）
        Order order = Order.create(command.getCustomerId(), items, address);

        // 5. 应用优惠券（如果有）
        if (command.getCouponCode() != null) {
            Coupon coupon = couponRepository
                .findByCode(command.getCouponCode())
                .orElseThrow(() -> new CouponNotFoundException(command.getCouponCode()));
            order.applyCoupon(coupon);
        }

        // 6. 持久化
        orderRepository.save(order);

        // 7. 保存幂等结果
        idempotencyService.saveResult(command.getCommandId(), order.getId());

        // 8. 发布领域事件
        eventPublisher.publishAll(order.getDomainEvents());
        order.clearDomainEvents();

        log.info("订单创建成功: {}", order.getId());
        return order.getId();
    }

    private List<OrderItem> loadOrderItems(
            List<PlaceOrderCommand.OrderItemCommand> itemCmds) {
        return itemCmds.stream()
            .map(cmd -> {
                Product product = productRepository
                    .findById(cmd.productId())
                    .orElseThrow(() -> new ProductNotFoundException(cmd.productId()));

                if (!product.isOnSale()) {
                    throw new ProductNotAvailableException(
                        cmd.productId(), "商品已下架");
                }

                return new OrderItem(
                    product.getId(),
                    product.getName(),
                    product.getPrice(),
                    cmd.quantity());
            })
            .collect(Collectors.toList());
    }
}
```

## 7.3 查询端实现（Query Side）

```java
// ============ 查询端完整实现 ============

// 查询条件（Query Object）
public class OrderListQuery {
    private final CustomerId customerId;
    private final OrderStatus statusFilter;
    private final LocalDate startDate;
    private final LocalDate endDate;
    private final int pageNo;
    private final int pageSize;

    private OrderListQuery(Builder builder) {
        this.customerId = Objects.requireNonNull(builder.customerId);
        this.statusFilter = builder.statusFilter;
        this.startDate = builder.startDate;
        this.endDate = builder.endDate;
        this.pageNo = builder.pageNo > 0 ? builder.pageNo : 1;
        this.pageSize = builder.pageSize > 0
            ? Math.min(builder.pageSize, 100) : 20;
    }

    public static Builder of(CustomerId customerId) {
        return new Builder(customerId);
    }

    public static class Builder {
        private final CustomerId customerId;
        private OrderStatus statusFilter;
        private LocalDate startDate;
        private LocalDate endDate;
        private int pageNo = 1;
        private int pageSize = 20;

        private Builder(CustomerId customerId) {
            this.customerId = customerId;
        }

        public Builder status(OrderStatus status) {
            this.statusFilter = status; return this;
        }
        public Builder dateRange(LocalDate start, LocalDate end) {
            this.startDate = start; this.endDate = end; return this;
        }
        public Builder page(int pageNo, int pageSize) {
            this.pageNo = pageNo; this.pageSize = pageSize; return this;
        }
        public OrderListQuery build() { return new OrderListQuery(this); }
    }

    // getters
    public CustomerId getCustomerId() { return customerId; }
    public OrderStatus getStatusFilter() { return statusFilter; }
    public LocalDate getStartDate() { return startDate; }
    public LocalDate getEndDate() { return endDate; }
    public int getPageNo() { return pageNo; }
    public int getPageSize() { return pageSize; }
}

// 查询 VO（视图对象，扁平化）
public class OrderListItemVO {
    private String orderId;
    private String statusCode;
    private String statusText;
    private String totalAmount;
    private String currency;
    private int itemCount;
    private String previewText;        // 如 "iPhone 15 等3件商品"
    private String thumbnailUrl;       // 第一件商品缩略图
    private String createdAtText;      // 格式化时间
    private boolean canCancel;
    private boolean canConfirm;

    // 全参数构造器
    @Builder
    public OrderListItemVO(String orderId, String statusCode,
                            String statusText, String totalAmount,
                            String currency, int itemCount,
                            String previewText, String thumbnailUrl,
                            String createdAtText, boolean canCancel,
                            boolean canConfirm) {
        this.orderId = orderId;
        this.statusCode = statusCode;
        this.statusText = statusText;
        this.totalAmount = totalAmount;
        this.currency = currency;
        this.itemCount = itemCount;
        this.previewText = previewText;
        this.thumbnailUrl = thumbnailUrl;
        this.createdAtText = createdAtText;
        this.canCancel = canCancel;
        this.canConfirm = canConfirm;
    }
    // ... getters
}

// 查询 Handler
@Service
@Transactional(readOnly = true)
@RequiredArgsConstructor
public class OrderQueryHandler {

    private final OrderQueryMapper queryMapper;
    private final RedisTemplate<String, Object> redisTemplate;

    // 查询订单列表（分页）
    public PageResult<OrderListItemVO> queryOrderList(OrderListQuery query) {
        // 注意：查询端直接用 MyBatis SQL，不走聚合加载
        // 可以跨表 JOIN，性能更好
        OrderListQueryDO queryDO = new OrderListQueryDO(
            query.getCustomerId().getValue(),
            query.getStatusFilter() != null
                ? query.getStatusFilter().name() : null,
            query.getStartDate(),
            query.getEndDate(),
            (query.getPageNo() - 1) * query.getPageSize(),
            query.getPageSize());

        long total = queryMapper.countOrderList(queryDO);
        List<OrderListItemVO> items = queryMapper.selectOrderList(queryDO);

        return PageResult.of(items, total, query.getPageNo(), query.getPageSize());
    }

    // 查询订单详情（有缓存）
    public OrderDetailVO queryOrderDetail(OrderId orderId) {
        String cacheKey = "order:detail:" + orderId.getValue();

        // 先查缓存
        OrderDetailVO cached =
            (OrderDetailVO) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }

        // 缓存未命中，查 DB
        OrderDetailVO detail = queryMapper.selectOrderDetail(orderId.getValue());
        if (detail == null) {
            throw new OrderNotFoundException(orderId);
        }

        // 写入缓存（已完成/已取消的订单，缓存较长时间）
        long ttl = detail.getStatusCode().equals("COMPLETED")
            || detail.getStatusCode().equals("CANCELLED")
            ? 3600 : 60;  // 秒
        redisTemplate.opsForValue().set(cacheKey, detail, ttl, TimeUnit.SECONDS);

        return detail;
    }
}
```

```xml
<!-- OrderQueryMapper.xml：读优化的 SQL -->
<select id="selectOrderList" parameterType="OrderListQueryDO"
        resultType="OrderListItemVO">
    SELECT
        o.order_id          AS orderId,
        o.status            AS statusCode,
        CASE o.status
            WHEN 'PENDING'   THEN '待支付'
            WHEN 'PAID'      THEN '已支付'
            WHEN 'SHIPPED'   THEN '已发货'
            WHEN 'COMPLETED' THEN '已完成'
            WHEN 'CANCELLED' THEN '已取消'
        END                 AS statusText,
        o.total_amount      AS totalAmount,
        o.currency          AS currency,
        COUNT(oi.item_id)   AS itemCount,
        CONCAT(
            (SELECT product_name FROM t_order_item
             WHERE order_id = o.order_id LIMIT 1),
            CASE WHEN COUNT(oi.item_id) > 1
                 THEN CONCAT(' 等', COUNT(oi.item_id), '件商品')
                 ELSE '' END
        )                   AS previewText,
        DATE_FORMAT(o.created_at, '%Y-%m-%d %H:%i') AS createdAtText
    FROM t_order o
    LEFT JOIN t_order_item oi ON o.order_id = oi.order_id
    WHERE o.customer_id = #{customerId}
    <if test="status != null">
        AND o.status = #{status}
    </if>
    <if test="startDate != null">
        AND o.created_at >= #{startDate}
    </if>
    <if test="endDate != null">
        AND o.created_at &lt;= #{endDate}
    </if>
    GROUP BY o.order_id
    ORDER BY o.created_at DESC
    LIMIT #{offset}, #{pageSize}
</select>
```

---

# Part 8: EventStorming 事件风暴

## 8.1 什么是事件风暴

事件风暴（EventStorming）是由 Alberto Brandolini 发明的一种**协作式领域建模工作坊**，
通过将所有参与者（领域专家、开发人员、产品经理、测试人员）聚集在一起，
用便利贴在白板上快速探索复杂的业务领域。

```
事件风暴工作坊元素

便利贴颜色规范：
+------------------+  橙色  = 领域事件（Domain Event）过去时
| OrderPlaced      |         "发生了什么"
+------------------+

+------------------+  蓝色  = 命令（Command）
| Place Order      |         "触发了什么行为"
+------------------+

+------------------+  黄色  = 聚合（Aggregate）
|    Order         |         "谁执行了命令"
+------------------+

+------------------+  粉色  = 外部系统（External System）
|  Payment System  |         "外部触发源"
+------------------+

+------------------+  绿色  = 读模型（Read Model）
|  Order Detail    |         "决策依赖的信息"
+------------------+

+------------------+  紫色  = 策略/规则（Policy）
|  Auto Cancel     |         "当...时，执行..."
+------------------+
```

## 8.2 EventStorming 流程

```
事件风暴四步流程

步骤 1：发散 - 识别领域事件（橙色便利贴）
   所有人随意贴：发生了什么业务事件？
   
步骤 2：聚合 - 识别命令和触发者（蓝色+黄色便利贴）
   每个事件 <- 命令触发 <- 用户/策略执行
   
步骤 3：聚类 - 识别聚合和限界上下文
   相关的命令+事件聚在一起，找出业务边界
   
步骤 4：设计 - 系统间关系和流程
   识别子域、限界上下文、上下文映射
```

## 8.3 电商订单流程完整事件风暴示例

```
电商下单完整流程 - EventStorming 结果图

时间轴 ────────────────────────────────────────────────────────────────>

[用户操作区域]
用户 ──> [提交订单] ──> [确认支付] ──> [确认收货]
                                    
命令（蓝色）:
+------------+   +------------+   +-------------+   +-------------+
|Place Order |   |Pay Order   |   | Confirm     |   | Cancel Order|
|(提交订单)   |   |(支付订单)   |   | Receipt     |   | (取消订单)   |
+------------+   +------------+   | (确认收货)   |   +-------------+
                                  +-------------+

聚合（黄色）:
+-------+    +-------+    +-----------+    +-------+
| Order |    | Order |    |   Order   |    | Order |
+-------+    +-------+    +-----------+    +-------+

领域事件（橙色）:
+------------+   +------------+   +-------------+   +-------------+
|OrderPlaced |   |OrderPaid   |   |OrderCompleted|  |OrderCancelled|
|(订单已创建) |   |(订单已支付) |   |(订单已完成)  |   |(订单已取消)  |
+------------+   +------------+   +-------------+   +-------------+
      |                |                 |                 |
      v                v                 v                 v
  策略（紫色）:
+------------+   +------------+   +-------------+   +-------------+
|当订单创建  |   |当订单支付  |   |当订单完成   |   |当订单取消   |
|预留库存    |   |通知仓库发货|   |增加积分     |   |释放库存     |
+------------+   +------------+   +-------------+   +-------------+
      |                |                 |                 |
      v                v                 v                 v
  外部系统（粉色）:
+----------+    +----------+   +----------+    +----------+
|Inventory |    |Warehouse |   |  Points  |    |Inventory |
|Service   |    |Service   |   |  Service |    |Service   |
+----------+    +----------+   +----------+    +----------+

外部触发的事件:
+----------------+   +----------------+   +----------------+
|StockReserved   |   |OrderShipped    |   |StockReleased   |
|(库存已预留)     |   |(包裹已发货)    |   |(库存已释放)     |
+----------------+   +----------------+   +----------------+

读模型（绿色，决策依赖）:
+------------------+   +------------------+   +------------------+
|商品库存详情       |   |订单支付状态       |   |积分账户信息       |
|(下单时查库存)     |   |(防止重复支付)     |   |(完成时增加积分)   |
+------------------+   +------------------+   +------------------+
```

## 8.4 从事件风暴到代码

```
事件风暴结果映射到 DDD 概念

EventStorming 元素    ->   DDD 战术概念
────────────────────────────────────────
命令（蓝色便利贴）    ->   Command（PlaceOrderCommand）
聚合（黄色便利贴）    ->   AggregateRoot（Order）
领域事件（橙色）      ->   DomainEvent（OrderPlacedEvent）
策略（紫色）         ->   EventHandler / Policy
外部系统（粉色）      ->   ACL / ExternalService
读模型（绿色）        ->   QueryModel / ReadModel
```

```java
// 从上图事件风暴直接映射到代码

// 命令
public class PlaceOrderCommand { ... }  // 对应"Place Order"蓝色便利贴

// 聚合根处理命令，产生事件
public class Order extends AggregateRoot<OrderId> {
    public static Order place(PlaceOrderCommand cmd) {
        // ... 创建聚合，发布 OrderPlacedEvent
    }
}

// 领域事件
public class OrderPlacedEvent extends DomainEvent {
    private final OrderId orderId;
    private final List<OrderItem> items;
    // ...
}

// 策略（订单创建后预留库存）
@Component
public class ReserveStockOnOrderPlacedPolicy {  // 对应"当订单创建，预留库存"

    @EventHandler
    @Transactional
    public void on(OrderPlacedEvent event) {
        inventoryService.reserveStock(
            new ReserveStockCommand(event.getOrderId(), event.getItems()));
    }
}

// 外部系统（库存服务）发出的事件被本域消费
@Component
public class OrderInventoryEventHandler {

    @EventHandler
    @Transactional
    public void onStockReserved(StockReservedEvent event) {
        // 库存预留成功，通知订单可以进入下一步
        Order order = orderRepository.findById(event.getOrderId()).orElseThrow();
        order.confirmInventoryReserved();
        orderRepository.save(order);
    }
}
```

---

# Part 9: 完整电商 DDD 实战案例

## 9.1 领域划分全景

```
电商平台 DDD 完整领域划分

+----------------------------------------------------------------+
|                        电商平台领域                              |
+----------------------------------------------------------------+

+----------------+  +----------------+  +------------------+
|  用户上下文     |  |  商品上下文     |  |   订单上下文★     |
| User Context   |  |Product Context |  |  Order Context   |
|                |  |                |  |  (核心域)         |
| 聚合：          |  | 聚合：          |  | 聚合：            |
| - Customer     |  | - Product      |  | - Order（聚合根）  |
|   - CustomerId |  |   - ProductId  |  |   - OrderItem    |
|   - Profile    |  |   - SKU        |  |                  |
|   - Address[]  |  |   - Category   |  | 值对象：           |
|                |  |   - Price      |  | - Money           |
| 领域服务：      |  |                |  | - Address         |
| - AuthService  |  | 领域服务：      |  | - OrderStatus     |
|                |  | -PricingService|  |                  |
+----------------+  +----------------+  +------------------+
                                               |  |
                        +----------------------+  +--------------------+
                        |                                             |
              +------------------+                        +------------------+
              |  库存上下文       |                        |   支付上下文      |
              | Inventory Context|                        | Payment Context  |
              |                  |                        |                  |
              | 聚合：            |                        | 聚合：            |
              | - Inventory      |                        | - Payment        |
              |   - ProductId    |                        |   - PaymentId    |
              |   - StockQty     |                        |   - Amount       |
              |   - ReservedQty  |                        |   - Channel      |
              |                  |                        |   - Status       |
              +------------------+                        +------------------+
                        |
              +------------------+
              |  通知上下文       |
              |Notification Ctx  |
              |                  |
              | - Notification   |
              |   - Template     |
              |   - Channel      |
              |   - Status       |
              +------------------+
```

## 9.2 订单域完整实现

### 9.2.1 值对象

```java
// ============ Money 值对象（完整版）============
@Value  // Lombok 不可变
public final class Money {
    public static final Money ZERO = new Money(BigDecimal.ZERO, "CNY");

    BigDecimal amount;
    String currency;

    public Money(BigDecimal amount, String currency) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException(
                "金额不合法: " + amount);
        }
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = Objects.requireNonNull(currency, "货币不能为空");
    }

    public static Money of(String amount, String currency) {
        return new Money(new BigDecimal(amount), currency);
    }

    public static Money cny(double amount) {
        return new Money(BigDecimal.valueOf(amount), "CNY");
    }

    public Money add(Money other) {
        assertSameCurrency(other);
        return new Money(this.amount.add(other.amount), currency);
    }

    public Money subtract(Money other) {
        assertSameCurrency(other);
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new InsufficientMoneyException(this, other);
        }
        return new Money(result, currency);
    }

    public Money multiply(int factor) {
        if (factor < 0) throw new IllegalArgumentException("倍数不能为负");
        return new Money(amount.multiply(BigDecimal.valueOf(factor)), currency);
    }

    public Money discountBy(BigDecimal rate) {
        if (rate.compareTo(BigDecimal.ZERO) < 0
                || rate.compareTo(BigDecimal.ONE) > 0) {
            throw new IllegalArgumentException("折扣率必须在0~1之间");
        }
        return new Money(amount.multiply(rate)
            .setScale(2, RoundingMode.HALF_UP), currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return amount.compareTo(other.amount) > 0;
    }

    public boolean isNegativeOrZero() {
        return amount.compareTo(BigDecimal.ZERO) <= 0;
    }

    private void assertSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(currency, other.currency);
        }
    }

    @Override
    public String toString() {
        return currency + " " + amount.toPlainString();
    }
}

// ============ OrderStatus 枚举（状态机）============
public enum OrderStatus {
    PENDING("待支付", 0) {
        @Override
        public Set<OrderStatus> nextAllowed() {
            return Set.of(PAID, CANCELLED);
        }
    },
    PAID("已支付", 1) {
        @Override
        public Set<OrderStatus> nextAllowed() {
            return Set.of(SHIPPED, CANCELLED);
        }
    },
    SHIPPED("已发货", 2) {
        @Override
        public Set<OrderStatus> nextAllowed() {
            return Set.of(COMPLETED);
        }
    },
    COMPLETED("已完成", 3) {
        @Override
        public Set<OrderStatus> nextAllowed() {
            return Collections.emptySet();  // 终态
        }
    },
    CANCELLED("已取消", 4) {
        @Override
        public Set<OrderStatus> nextAllowed() {
            return Collections.emptySet();  // 终态
        }
    };

    private final String description;
    private final int sort;

    OrderStatus(String description, int sort) {
        this.description = description;
        this.sort = sort;
    }

    public abstract Set<OrderStatus> nextAllowed();

    public boolean canTransitionTo(OrderStatus target) {
        return nextAllowed().contains(target);
    }

    public boolean canCancel() {
        return canTransitionTo(CANCELLED);
    }

    public String getDescription() { return description; }
}
```

### 9.2.2 聚合根完整实现

```java
// ============ Order 聚合根完整实现 ============
public class Order extends AggregateRoot<OrderId> {

    // ========== 字段 ==========
    private OrderId orderId;
    private CustomerId customerId;
    private List<OrderItem> items;           // 聚合内实体
    private Address shippingAddress;          // 值对象
    private Money originalAmount;             // 原始金额
    private Money discountAmount;             // 折扣金额
    private Money totalAmount;                // 实付金额
    private OrderStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
    private String cancelReason;
    private LocalDateTime cancelledAt;
    private String couponCode;
    private int version;                      // 乐观锁

    // ========== 不变性常量 ==========
    private static final int MAX_ITEMS = 100;
    private static final Money MAX_AMOUNT = Money.cny(99999.99);

    // ========== 私有构造器 ==========
    private Order() {}

    // ========== 创建方法（Factory Method）==========
    public static Order create(CustomerId customerId,
                               List<OrderItem> items,
                               Address shippingAddress) {
        // 不变性验证
        validateCreation(customerId, items, shippingAddress);

        Order order = new Order();
        order.orderId = OrderId.generate();
        order.customerId = customerId;
        order.items = new ArrayList<>(items);
        order.shippingAddress = shippingAddress;
        order.originalAmount = order.calculateItemsTotal();
        order.discountAmount = Money.ZERO;
        order.totalAmount = order.originalAmount;
        order.status = OrderStatus.PENDING;
        order.createdAt = LocalDateTime.now();
        order.updatedAt = LocalDateTime.now();
        order.version = 0;

        // 验证金额不变性
        if (order.totalAmount.isNegativeOrZero()) {
            throw new OrderDomainException("订单金额必须大于0");
        }
        if (order.totalAmount.isGreaterThan(MAX_AMOUNT)) {
            throw new OrderDomainException("订单金额不能超过" + MAX_AMOUNT);
        }

        // 发布创建事件
        order.registerEvent(new OrderCreatedEvent(
            order.orderId,
            customerId,
            order.totalAmount,
            new ArrayList<>(items),
            order.createdAt));

        return order;
    }

    // ========== 数据库重建方法（不触发业务逻辑）==========
    public static Order reconstitute(OrderId orderId,
                                      CustomerId customerId,
                                      List<OrderItem> items,
                                      Address shippingAddress,
                                      Money originalAmount,
                                      Money discountAmount,
                                      Money totalAmount,
                                      OrderStatus status,
                                      LocalDateTime createdAt,
                                      String couponCode,
                                      int version) {
        Order order = new Order();
        order.orderId = orderId;
        order.customerId = customerId;
        order.items = new ArrayList<>(items);
        order.shippingAddress = shippingAddress;
        order.originalAmount = originalAmount;
        order.discountAmount = discountAmount;
        order.totalAmount = totalAmount;
        order.status = status;
        order.createdAt = createdAt;
        order.couponCode = couponCode;
        order.version = version;
        return order;
    }

    // ========== 业务行为 ==========

    /**
     * 应用优惠券
     */
    public void applyCoupon(Coupon coupon) {
        if (status != OrderStatus.PENDING) {
            throw new OrderDomainException("只有待支付订单才能应用优惠券");
        }
        if (!coupon.isValid()) {
            throw new CouponInvalidException(coupon.getCode(), "优惠券已过期或不可用");
        }
        if (!coupon.isApplicable(this.originalAmount)) {
            throw new CouponNotApplicableException(
                coupon.getCode(), "不满足优惠券使用条件");
        }
        this.discountAmount = coupon.calculateDiscount(this.originalAmount);
        this.totalAmount = originalAmount.subtract(discountAmount);
        this.couponCode = coupon.getCode();
        this.updatedAt = LocalDateTime.now();
    }

    /**
     * 支付确认
     */
    public void pay(PaymentId paymentId, Money paidAmount) {
        if (!status.canTransitionTo(OrderStatus.PAID)) {
            throw new OrderDomainException(
                "订单[" + orderId + "]状态[" + status.getDescription() + "]不允许支付");
        }
        if (!paidAmount.equals(totalAmount)) {
            throw new OrderDomainException(
                "支付金额[" + paidAmount + "]与应付金额[" + totalAmount + "]不符");
        }
        this.status = OrderStatus.PAID;
        this.updatedAt = LocalDateTime.now();
        registerEvent(new OrderPaidEvent(
            orderId, paymentId, paidAmount, LocalDateTime.now()));
    }

    /**
     * 取消订单
     */
    public void cancel(String reason) {
        if (!status.canTransitionTo(OrderStatus.CANCELLED)) {
            throw new OrderDomainException(
                "订单[" + orderId + "]状态[" + status.getDescription() + "]不允许取消");
        }
        if (reason == null || reason.isBlank()) {
            throw new OrderDomainException("取消原因不能为空");
        }
        this.status = OrderStatus.CANCELLED;
        this.cancelReason = reason;
        this.cancelledAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
        registerEvent(new OrderCancelledEvent(orderId, reason, cancelledAt));
    }

    /**
     * 发货（记录运单号）
     */
    public void ship(String trackingNumber, String carrier) {
        if (!status.canTransitionTo(OrderStatus.SHIPPED)) {
            throw new OrderDomainException("订单状态不允许发货");
        }
        if (trackingNumber == null || trackingNumber.isBlank()) {
            throw new OrderDomainException("运单号不能为空");
        }
        this.status = OrderStatus.SHIPPED;
        this.updatedAt = LocalDateTime.now();
        registerEvent(new OrderShippedEvent(
            orderId, trackingNumber, carrier, LocalDateTime.now()));
    }

    /**
     * 确认收货
     */
    public void confirmReceipt() {
        if (!status.canTransitionTo(OrderStatus.COMPLETED)) {
            throw new OrderDomainException("订单状态不允许确认收货");
        }
        this.status = OrderStatus.COMPLETED;
        this.updatedAt = LocalDateTime.now();
        registerEvent(new OrderCompletedEvent(orderId, LocalDateTime.now()));
    }

    // ========== 内部方法 ==========
    private Money calculateItemsTotal() {
        return items.stream()
            .map(OrderItem::getSubtotal)
            .reduce(Money.ZERO, Money::add);
    }

    private static void validateCreation(CustomerId customerId,
                                          List<OrderItem> items,
                                          Address shippingAddress) {
        Objects.requireNonNull(customerId, "客户ID不能为空");
        Objects.requireNonNull(shippingAddress, "收货地址不能为空");
        if (items == null || items.isEmpty()) {
            throw new OrderDomainException("订单必须包含至少一件商品");
        }
        if (items.size() > MAX_ITEMS) {
            throw new OrderDomainException(
                "单笔订单最多" + MAX_ITEMS + "种商品");
        }
    }

    // ========== 查询方法（只读）==========
    public OrderId getId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public OrderStatus getStatus() { return status; }
    public Money getTotalAmount() { return totalAmount; }
    public Money getOriginalAmount() { return originalAmount; }
    public Money getDiscountAmount() { return discountAmount; }
    public Address getShippingAddress() { return shippingAddress; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public String getCouponCode() { return couponCode; }
    public int getVersion() { return version; }
    public List<OrderItem> getItems() {
        return Collections.unmodifiableList(items);
    }
    public boolean isPending() { return status == OrderStatus.PENDING; }
    public boolean isCompleted() { return status == OrderStatus.COMPLETED; }
    public boolean isCancelled() { return status == OrderStatus.CANCELLED; }
}
```

### 9.2.3 领域事件

```java
// ============ 订单领域事件 ============

// 订单创建事件
public class OrderCreatedEvent extends DomainEvent {
    private final OrderId orderId;
    private final CustomerId customerId;
    private final Money totalAmount;
    private final List<EventOrderItem> items;
    private final LocalDateTime createdAt;

    public OrderCreatedEvent(OrderId orderId, CustomerId customerId,
                              Money totalAmount, List<OrderItem> items,
                              LocalDateTime createdAt) {
        super("ORDER_CREATED");
        this.orderId = orderId;
        this.customerId = customerId;
        this.totalAmount = totalAmount;
        // 快照商品信息（防止后续修改影响事件）
        this.items = items.stream()
            .map(i -> new EventOrderItem(
                i.getProductId(), i.getProductName(),
                i.getQuantity(), i.getUnitPrice()))
            .collect(Collectors.toList());
        this.createdAt = createdAt;
    }

    public record EventOrderItem(
            ProductId productId, String productName,
            int quantity, Money unitPrice) {}

    // getters...
    public OrderId getOrderId() { return orderId; }
    public CustomerId getCustomerId() { return customerId; }
    public Money getTotalAmount() { return totalAmount; }
    public List<EventOrderItem> getItems() { return items; }
}

// 订单支付事件
public class OrderPaidEvent extends DomainEvent {
    private final OrderId orderId;
    private final PaymentId paymentId;
    private final Money amount;
    private final LocalDateTime paidAt;

    public OrderPaidEvent(OrderId orderId, PaymentId paymentId,
                           Money amount, LocalDateTime paidAt) {
        super("ORDER_PAID");
        this.orderId = orderId;
        this.paymentId = paymentId;
        this.amount = amount;
        this.paidAt = paidAt;
    }
    // getters...
}

// 订单取消事件
public class OrderCancelledEvent extends DomainEvent {
    private final OrderId orderId;
    private final String cancelReason;
    private final LocalDateTime cancelledAt;

    public OrderCancelledEvent(OrderId orderId, String cancelReason,
                                LocalDateTime cancelledAt) {
        super("ORDER_CANCELLED");
        this.orderId = orderId;
        this.cancelReason = cancelReason;
        this.cancelledAt = cancelledAt;
    }
    // getters...
}
```

### 9.2.4 应用服务完整实现

```java
// ============ 下单应用服务 ============
@Service
@Transactional
@RequiredArgsConstructor
@Slf4j
public class PlaceOrderCommandHandler {

    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final CouponRepository couponRepository;
    private final InventoryCheckService inventoryCheckService;
    private final DomainEventPublisher eventPublisher;

    public PlaceOrderResult handle(PlaceOrderCommand command) {
        log.info("处理下单命令: customerId={}, items={}",
            command.getCustomerId(), command.getItems().size());

        // 1. 库存预检（应用层协调，非领域规则）
        inventoryCheckService.checkAvailable(command.getItems());

        // 2. 加载商品信息，创建订单项
        List<OrderItem> orderItems = buildOrderItems(command.getItems());

        // 3. 构建收货地址
        Address shippingAddress = buildAddress(command.getAddress());

        // 4. 创建订单聚合（领域规则在此执行）
        Order order = Order.create(
            command.getCustomerId(), orderItems, shippingAddress);

        // 5. 应用优惠券（如果有）
        if (command.getCouponCode() != null) {
            Coupon coupon = couponRepository
                .findByCodeForCustomer(
                    command.getCouponCode(), command.getCustomerId())
                .orElseThrow(() -> new CouponNotFoundException(
                    command.getCouponCode()));
            order.applyCoupon(coupon);
        }

        // 6. 持久化
        orderRepository.save(order);

        // 7. 发布领域事件（触发库存预留、通知等）
        List<DomainEvent> events = order.getDomainEvents();
        eventPublisher.publishAll(events);
        order.clearDomainEvents();

        log.info("订单创建成功: orderId={}, amount={}",
            order.getId(), order.getTotalAmount());

        return PlaceOrderResult.success(
            order.getId(), order.getTotalAmount());
    }

    private List<OrderItem> buildOrderItems(
            List<PlaceOrderCommand.OrderItemCommand> cmds) {
        return cmds.stream()
            .map(cmd -> {
                Product product = productRepository
                    .findById(cmd.productId())
                    .orElseThrow(() ->
                        new ProductNotFoundException(cmd.productId()));
                // 商品状态检查（应用层）
                if (!product.isOnSale()) {
                    throw new ProductUnavailableException(
                        cmd.productId(), product.getName() + " 已下架");
                }
                return new OrderItem(
                    product.getId(),
                    product.getName(),
                    product.getCurrentPrice(),
                    cmd.quantity());
            })
            .collect(Collectors.toList());
    }

    private Address buildAddress(PlaceOrderCommand.AddressCommand cmd) {
        return new Address(
            cmd.province(), cmd.city(), cmd.district(),
            cmd.street(), cmd.detail(), cmd.zipCode(),
            cmd.receiverName(), cmd.receiverPhone());
    }
}

// 下单结果
public record PlaceOrderResult(
        OrderId orderId,
        Money totalAmount,
        boolean success,
        String errorMessage) {

    public static PlaceOrderResult success(OrderId orderId, Money amount) {
        return new PlaceOrderResult(orderId, amount, true, null);
    }
}
```

### 9.2.5 单元测试

```java
// ============ 领域对象单元测试（不需要 Spring 容器！）============
class OrderTest {

    private static final CustomerId CUSTOMER_ID = CustomerId.of("C001");
    private static final ProductId PRODUCT_ID = ProductId.of("P001");

    @Test
    void should_create_order_successfully() {
        // Given
        List<OrderItem> items = List.of(
            new OrderItem(PRODUCT_ID, "iPhone 15", Money.cny(8999.00), 1));
        Address address = buildTestAddress();

        // When
        Order order = Order.create(CUSTOMER_ID, items, address);

        // Then
        assertThat(order.getId()).isNotNull();
        assertThat(order.getStatus()).isEqualTo(OrderStatus.PENDING);
        assertThat(order.getTotalAmount()).isEqualTo(Money.cny(8999.00));
        assertThat(order.getDomainEvents()).hasSize(1);
        assertThat(order.getDomainEvents().get(0))
            .isInstanceOf(OrderCreatedEvent.class);
    }

    @Test
    void should_cancel_pending_order() {
        // Given
        Order order = createPendingOrder();

        // When
        order.cancel("用户主动取消");

        // Then
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
        assertThat(order.getDomainEvents())
            .filteredOn(e -> e instanceof OrderCancelledEvent)
            .hasSize(1);
    }

    @Test
    void should_reject_cancel_completed_order() {
        // Given
        Order order = createCompletedOrder();

        // When / Then
        assertThatThrownBy(() -> order.cancel("测试"))
            .isInstanceOf(OrderDomainException.class)
            .hasMessageContaining("不允许取消");
    }

    @Test
    void should_reject_order_with_empty_items() {
        // When / Then
        assertThatThrownBy(() ->
            Order.create(CUSTOMER_ID, Collections.emptyList(), buildTestAddress()))
            .isInstanceOf(OrderDomainException.class)
            .hasMessageContaining("至少一件商品");
    }

    @Test
    void should_apply_coupon_and_update_total() {
        // Given
        Order order = createPendingOrder();
        Coupon coupon = FixedDiscountCoupon.of("COUPON100",
            Money.cny(100.00), Money.cny(500.00));  // 满500减100

        // When
        order.applyCoupon(coupon);

        // Then
        assertThat(order.getDiscountAmount()).isEqualTo(Money.cny(100.00));
        assertThat(order.getTotalAmount())
            .isEqualTo(order.getOriginalAmount().subtract(Money.cny(100.00)));
    }

    // 辅助方法
    private Order createPendingOrder() {
        List<OrderItem> items = List.of(
            new OrderItem(PRODUCT_ID, "iPhone 15", Money.cny(8999.00), 1));
        return Order.create(CUSTOMER_ID, items, buildTestAddress());
    }

    private Address buildTestAddress() {
        return new Address("广东省", "深圳市", "南山区",
            "科技园路1号", "A栋101", "518000",
            "张三", "13800138000");
    }
}
```

---

## 9.3 跨域协作：订单 -> 库存 Saga

```java
// ============ 订单-库存 Saga 完整实现 ============

// 库存上下文的防腐层接口（在订单域定义）
public interface InventoryService {
    void reserveStock(ReserveStockRequest request);
    void releaseStock(ReleaseStockRequest request);
}

// 防腐层实现（在基础设施层）
@Component
public class InventoryServiceAcl implements InventoryService {

    private final InventoryFeignClient inventoryClient;

    @Override
    public void reserveStock(ReserveStockRequest request) {
        // 将订单域的请求对象翻译成库存系统的 DTO
        ReserveStockDTO dto = new ReserveStockDTO();
        dto.setOrderId(request.getOrderId().getValue());
        dto.setItems(request.getItems().stream()
            .map(item -> {
                ReserveStockDTO.Item i = new ReserveStockDTO.Item();
                i.setSkuId(item.getProductId().getValue());
                i.setQuantity(item.getQuantity());
                return i;
            })
            .collect(Collectors.toList()));

        try {
            inventoryClient.reserve(dto);
        } catch (FeignException.BadRequest e) {
            // 将外部异常翻译为领域异常
            throw new InsufficientStockException(
                "库存不足，请减少购买数量", e);
        }
    }
}

// 订单创建后，Saga 触发库存预留
@Component
@RequiredArgsConstructor
public class OrderCreatedSagaStep {

    private final InventoryService inventoryService;
    private final OrderRepository orderRepository;
    private final DomainEventPublisher eventPublisher;

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void onOrderCreated(OrderCreatedEvent event) {
        try {
            // 调用库存服务预留库存
            inventoryService.reserveStock(
                buildReserveRequest(event));
        } catch (InsufficientStockException e) {
            // 库存不足，触发补偿：取消订单
            compensate(event.getOrderId(), "库存不足: " + e.getMessage());
        }
    }

    private void compensate(OrderId orderId, String reason) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.cancel(reason);
        orderRepository.save(order);
        eventPublisher.publishAll(order.getDomainEvents());
        order.clearDomainEvents();
    }

    private ReserveStockRequest buildReserveRequest(OrderCreatedEvent event) {
        List<ReserveStockRequest.Item> items = event.getItems().stream()
            .map(i -> new ReserveStockRequest.Item(
                i.productId(), i.quantity()))
            .collect(Collectors.toList());
        return new ReserveStockRequest(event.getOrderId(), items);
    }
}
```

---

# Part 10: DDD 与微服务

## 10.1 限界上下文与微服务的关系

```
限界上下文与微服务的关系

+--------------------------------------------------+
|              三个常见误区                          |
+--------------------------------------------------+
|                                                  |
|  误区1：一个限界上下文 = 一个微服务                 |
|  实际：早期可以多个BC合在一个服务，随业务增长再拆分  |
|                                                  |
|  误区2：微服务越小越好                             |
|  实际：微服务边界应该对齐限界上下文，不是越小越好    |
|                                                  |
|  误区3：DDD 只适用于微服务                         |
|  实际：DDD 同样适用于单体应用（Modular Monolith）  |
|                                                  |
+--------------------------------------------------+

正确关系：
- 限界上下文是业务边界（what to split）
- 微服务是部署边界（how to deploy）
- 一个BC可以部署为多个微服务（水平扩展）
- 多个BC也可以在一个微服务中（逻辑分层，物理合并）

推荐演进路径：
模块化单体(Modular Monolith)
    → 成熟后，按BC拆分微服务
    → 避免过早优化
```

## 10.2 微服务间通信策略

```java
// ============ 同步通信（REST/gRPC）：查询场景 ============

// OpenFeign 客户端（在基础设施层）
@FeignClient(
    name = "user-service",
    url = "${user.service.url}",
    fallback = UserServiceFallback.class)
public interface UserServiceFeignClient {

    @GetMapping("/api/v1/users/{userId}/address/default")
    UserAddressDTO getDefaultAddress(@PathVariable("userId") String userId);
}

// 熔断降级
@Component
public class UserServiceFallback implements UserServiceFeignClient {
    @Override
    public UserAddressDTO getDefaultAddress(String userId) {
        // 降级：返回 null，让调用方决定如何处理
        return null;
    }
}

// ACL（防腐层）翻译外部 DTO 为领域值对象
@Component
public class UserContextAcl {

    private final UserServiceFeignClient client;

    public Optional<Address> getCustomerDefaultAddress(CustomerId customerId) {
        UserAddressDTO dto = client.getDefaultAddress(customerId.getValue());
        if (dto == null) return Optional.empty();

        return Optional.of(new Address(
            dto.getProvince(), dto.getCity(), dto.getDistrict(),
            dto.getStreet(), dto.getDetail(), dto.getZipCode(),
            dto.getContactName(), dto.getContactPhone()));
    }
}

// ============ 异步通信（消息队列）：状态通知场景 ============

// 事件消息 DTO（跨服务传输，与领域事件分开定义）
public class OrderPaidMessage {
    private String messageId;
    private String orderId;
    private String customerId;
    private BigDecimal amount;
    private String currency;
    private List<OrderItemMessage> items;
    private long timestamp;

    public static class OrderItemMessage {
        private String productId;
        private String productName;
        private int quantity;
        private BigDecimal unitPrice;
    }
}

// 消息发布（将领域事件转换为消息并发布）
@Component
@RequiredArgsConstructor
public class OrderPaidEventMessagePublisher {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @EventListener
    @Async
    public void handleOrderPaid(OrderPaidEvent event) {
        try {
            OrderPaidMessage message = toMessage(event);
            String json = objectMapper.writeValueAsString(message);
            kafkaTemplate.send("order.paid", event.getOrderId().getValue(), json);
        } catch (Exception e) {
            throw new RuntimeException("发送订单支付消息失败", e);
        }
    }

    private OrderPaidMessage toMessage(OrderPaidEvent event) {
        OrderPaidMessage msg = new OrderPaidMessage();
        msg.setMessageId(event.getEventId());
        msg.setOrderId(event.getOrderId().getValue());
        msg.setAmount(event.getAmount().getAmount());
        msg.setCurrency(event.getAmount().getCurrency());
        msg.setTimestamp(event.getOccurredOn().toEpochSecond(ZoneOffset.UTC));
        return msg;
    }
}
```

## 10.3 共享内核的处理

```java
// ============ 共享内核（common 模块）============
// 警告：共享内核应该最小化，只包含真正跨BC共享的内容

// common-domain 模块：所有 BC 共享
package com.example.common.domain;

// 1. 基础值对象（通用）
public final class Money {
    // 所有 BC 都使用相同的 Money 定义
    // 这是共享内核的合理候选
}

// 2. 基础设施抽象
public abstract class AggregateRoot<ID> { ... }
public abstract class Entity<ID> { ... }
public abstract class DomainEvent { ... }

// 3. 通用异常
public class DomainException extends RuntimeException { ... }

// 不应该放在共享内核的内容：
// - 具体业务实体（Order、Product、User 等）
// - 业务枚举（OrderStatus 等）
// - 业务规则（定价逻辑等）
// 这些属于各自的 BC，不应共享

// common-api 模块：对外暴露的 DTO（Open Host Service）
// 供其他服务调用的稳定 API 定义
public class OrderQueryDTO {
    // 这是给外部消费者的数据结构，与内部领域模型分离
}
```

## 10.4 分布式事务与聚合一致性

```
分布式事务处理策略

+--------------------------------------------------+
|           分布式场景一致性方案选择                   |
+--------------------------------------------------+
|                                                  |
|  1. 避免分布式事务（最佳策略）                      |
|     - 重新设计聚合边界                             |
|     - 将必须强一致的操作放在同一聚合/服务中           |
|                                                  |
|  2. Saga 模式（推荐）                              |
|     - 最终一致性                                  |
|     - 编排Saga（Orchestration）：有中心协调者       |
|     - 编舞Saga（Choreography）：事件驱动，去中心化  |
|                                                  |
|  3. TCC（Try-Confirm-Cancel）                    |
|     - 强一致，性能较差                             |
|     - 适合金融场景                                |
|                                                  |
|  4. 本地消息表（Outbox Pattern）                   |
|     - 保证消息可靠投递                             |
|     - 结合 Saga 使用                              |
|                                                  |
+--------------------------------------------------+
```

```java
// ============ 编舞 Saga（Choreography）示例 ============
// 无中心协调者，通过事件驱动

// 订单服务：发布事件
@Service
public class PlaceOrderHandler {
    public OrderId handle(PlaceOrderCommand cmd) {
        Order order = Order.create(...);
        orderRepository.save(order);
        // 发布事件，库存服务会监听
        eventPublisher.publish(new OrderCreatedEvent(order));
        return order.getId();
    }
}

// 库存服务：消费事件，执行操作，发布结果事件
@KafkaListener(topics = "order.created")
public void onOrderCreated(OrderCreatedMessage msg) {
    try {
        inventory.reserve(msg);
        // 成功：发布预留成功事件
        kafkaTemplate.send("inventory.reserved", msg.getOrderId(), ...);
    } catch (InsufficientStockException e) {
        // 失败：发布预留失败事件（触发订单服务补偿）
        kafkaTemplate.send("inventory.reserve.failed", msg.getOrderId(), ...);
    }
}

// 订单服务：监听库存失败事件，执行补偿
@KafkaListener(topics = "inventory.reserve.failed")
public void onInventoryReserveFailed(InventoryReserveFailedMessage msg) {
    // 补偿：取消订单
    Order order = orderRepository.findById(
        OrderId.of(msg.getOrderId())).orElseThrow();
    order.cancel("库存不足");
    orderRepository.save(order);
}
```

---

# Part 11: DDD 落地常见问题

## 11.1 贫血模型转充血模型的步骤

```
贫血模型转充血模型步骤

Step 1：识别业务规则（从 Service 中找）
  - 寻找 if/else 业务判断
  - 寻找状态修改逻辑
  - 寻找计算逻辑

Step 2：将规则移入实体/值对象
  - 状态校验 -> 实体方法的前置条件
  - 状态修改 -> 实体的行为方法
  - 计算逻辑 -> 实体/值对象的方法

Step 3：识别聚合边界
  - 哪些对象必须在同一事务中保持一致？
  - 哪些是真正的实体（有生命周期）？
  - 哪些是值对象（无标识，可替换）？

Step 4：引入领域事件
  - 状态变更 -> 发布领域事件
  - 解耦跨聚合协作

Step 5：应用服务瘦身
  - 移除业务规则，只保留编排逻辑
```

```java
// ============ 贫血 -> 充血 重构示例 ============

// 重构前（贫血）
@Service
public class OrderService {
    public void confirmOrder(Long orderId) {
        Order order = orderMapper.selectById(orderId);
        // 业务规则散落在 Service
        if (!"SHIPPED".equals(order.getStatus())) {
            throw new RuntimeException("只有已发货订单才能确认");
        }
        order.setStatus("COMPLETED");
        order.setConfirmedAt(LocalDateTime.now());
        orderMapper.updateById(order);

        // 增加积分（直接调用积分 Service，强耦合）
        pointsService.addPoints(order.getCustomerId(), calculatePoints(order));
    }
}

// 重构后（充血）
// Step 1: 聚合根包含业务行为和事件
public class Order extends AggregateRoot<OrderId> {
    public void confirmReceipt() {
        if (status != OrderStatus.SHIPPED) {
            throw new OrderDomainException("只有已发货订单才能确认收货");
        }
        this.status = OrderStatus.COMPLETED;
        this.confirmedAt = LocalDateTime.now();
        // 通过事件解耦积分增加
        registerEvent(new OrderCompletedEvent(orderId, totalAmount));
    }
}

// Step 2: 应用服务只做编排
@Service
@Transactional
public class ConfirmReceiptHandler {
    public void handle(ConfirmReceiptCommand command) {
        Order order = orderRepository.findById(command.getOrderId()).orElseThrow();
        order.confirmReceipt();  // 领域行为
        orderRepository.save(order);
        eventPublisher.publishAll(order.getDomainEvents());
    }
}

// Step 3: 积分服务通过事件异步增加积分（解耦）
@EventHandler
public void onOrderCompleted(OrderCompletedEvent event) {
    pointsService.addPoints(event.getCustomerId(),
        calculatePoints(event.getTotalAmount()));
}
```

## 11.2 过度设计的预防

```
DDD 的过度设计症状及对策

+------------------------------------------+
|              常见过度设计症状               |
+------------------------------------------+
|                                          |
|  1. 为所有字符串创建值对象                  |
|     症状：UserId, ProductId 每个都是类      |
|     对策：只在有业务意义时才封装              |
|                                          |
|  2. 每个业务方法都创建命令对象               |
|     症状：AddressUpdateCommand, EmailUpdateCommand |
|     对策：简单操作直接用参数即可             |
|                                          |
|  3. 强行拆微服务                           |
|     症状：5人团队20个微服务                 |
|     对策：先建模块化单体，再按需拆分          |
|                                          |
|  4. 把CRUD页面也做成DDD                    |
|     症状：简单的配置表也有聚合根             |
|     对策：CRUD直接用传统三层，不要过度设计     |
|                                          |
+------------------------------------------+

原则：只在复杂业务场景使用 DDD，
      简单场景用最简单的方式。
```

## 11.3 遗留系统改造策略

### 扼杀者模式（Strangler Fig Pattern）

```
扼杀者模式：渐进式替换遗留系统

Phase 1: 新旧系统共存，新请求走新系统
+----------+    +--------+    +----------+
|  用户     | -> | 路由层  | -> | 新DDD系统 |
|  请求     |    |        |    |          |
+----------+    +--------+    +----------+
                     |
                     v
                +----------+
                | 旧系统    |
                | (遗留)   |
                +----------+

Phase 2: 数据迁移，逐步关闭旧功能
+----------+    +----------+
|  用户     | -> | 新DDD系统 |
|  请求     |    |  (全量)   |
+----------+    +----------+
                     |
                旧系统下线

关键技术：
- API Gateway 做路由（按功能模块路由）
- 数据双写（新老系统数据同步）
- 功能开关（Feature Flag）控制切换比例
```

### 气泡上下文（Bubble Context）

```java
// ============ 气泡上下文：在遗留系统中创建一个干净的 DDD 区域 ============

// 遗留系统的"脏"数据结构（无法修改）
public class LegacyOrder {
    // 各种不规范的字段名、混乱的状态码...
    private String ord_id;
    private Integer stat;  // 1=待支付 2=已支付 3=已完成 4=已取消
    private String cust_no;
    private BigDecimal tot_amt;
    // ...
}

// 气泡上下文：建立一个独立的 DDD 区域
// 通过 ACL 将遗留对象翻译为干净的领域模型
@Component
public class LegacyOrderAcl {

    private final LegacyOrderDAO legacyDAO;

    // 将遗留对象翻译为 DDD 领域对象
    public Optional<Order> findById(OrderId orderId) {
        LegacyOrder legacy = legacyDAO.findByOrdId(orderId.getValue());
        if (legacy == null) return Optional.empty();

        OrderStatus status = mapStatus(legacy.getStat());
        Money amount = new Money(legacy.getTotAmt(), "CNY");

        return Optional.of(Order.reconstitute(
            OrderId.of(legacy.getOrdId()),
            CustomerId.of(legacy.getCustNo()),
            Collections.emptyList(), // 简化处理
            null,
            amount, Money.ZERO, amount,
            status,
            legacy.getCreateTime(),
            null, 0));
    }

    // 将 DDD 领域对象保存回遗留系统
    public void save(Order order) {
        LegacyOrder legacy = legacyDAO.findByOrdId(order.getId().getValue());
        if (legacy == null) {
            legacy = new LegacyOrder();
            legacy.setOrdId(order.getId().getValue());
        }
        legacy.setStat(mapStatusToLegacy(order.getStatus()));
        legacy.setTotAmt(order.getTotalAmount().getAmount());
        legacyDAO.save(legacy);
    }

    private OrderStatus mapStatus(int stat) {
        return switch (stat) {
            case 1 -> OrderStatus.PENDING;
            case 2 -> OrderStatus.PAID;
            case 3 -> OrderStatus.COMPLETED;
            case 4 -> OrderStatus.CANCELLED;
            default -> throw new IllegalArgumentException("未知状态码: " + stat);
        };
    }

    private int mapStatusToLegacy(OrderStatus status) {
        return switch (status) {
            case PENDING -> 1;
            case PAID -> 2;
            case COMPLETED -> 3;
            case CANCELLED -> 4;
            default -> throw new IllegalArgumentException("无法映射状态: " + status);
        };
    }
}
```

## 11.4 DDD 与 Spring Boot 集成最佳实践

```java
// ============ Spring Boot DDD 集成最佳实践 ============

// 1. 领域层不使用 Spring 注解（保持纯洁）
// 错误做法：
@Service  // 不要在领域对象上加 Spring 注解！
public class Order extends AggregateRoot<OrderId> { }

// 正确做法：领域服务用自定义注解或无注解
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface DomainService {
    // 自定义注解，标识语义，但不影响 Spring 扫描
    // 如果需要 Spring 管理，单独加 @Component
}

// 2. 事务边界在应用层
// 应用服务上加 @Transactional
@Service
@Transactional  // 事务边界在应用层
public class PlaceOrderHandler {
    // 领域层的方法不应该有 @Transactional
}

// 3. 领域事件发布时机（避免事务外发布）
@Service
@Transactional
public class CancelOrderHandler {
    public void handle(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.getOrderId()).orElseThrow();
        order.cancel(command.getReason());
        orderRepository.save(order);
        // 在事务提交前发布（Spring 处理事务内的事件监听）
        eventPublisher.publishAll(order.getDomainEvents());
    }
}

// 4. 使用 @TransactionalEventListener 处理异步场景
@Component
public class AsyncOrderEventHandler {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async  // 异步处理，不阻塞主事务
    public void onOrderCreated(OrderCreatedEvent event) {
        // 在事务提交后异步处理，如：发送通知
        notificationService.sendNewOrderNotice(event.getCustomerId());
    }
}

// 5. 配置示例
@Configuration
@EnableTransactionManagement
@EnableAsync
public class DomainEventConfig {

    @Bean
    public DomainEventPublisher domainEventPublisher(
            ApplicationEventPublisher publisher) {
        return new SpringDomainEventPublisher(publisher);
    }

    @Bean
    @ConditionalOnProperty("app.async.enabled")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(4);
        executor.setMaxPoolSize(8);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("domain-event-");
        executor.initialize();
        return executor;
    }
}
```

---

# Part 12: 常见面试题 FAQ

## 问题 1：什么是 DDD？它解决了什么问题？

**答：**

DDD（领域驱动设计）是 Eric Evans 在 2003 年提出的一套软件设计方法论，核心思想是：
**以业务领域为核心，通过与领域专家协作建立统一语言，将业务模型映射到代码中**。

它主要解决以下问题：
1. **业务复杂性管理**：通过限界上下文划分边界，控制复杂度
2. **贫血模型问题**：提倡充血模型，让实体包含业务行为
3. **技术与业务脱节**：统一语言使代码能直接表达业务意图
4. **代码腐化**：通过聚合约束边界，防止依赖混乱

---

## 问题 2：什么是限界上下文？如何划分？

**答：**

限界上下文（Bounded Context）是 DDD 中**语义的边界**，在这个边界内，
同一个术语有唯一、明确的含义。

**划分方法：**
1. **EventStorming（事件风暴）**：协作建模，识别业务边界
2. **语言边界**：当同一个词（如 "User"）在两个地方意义不同时，就是边界
3. **团队边界**：不同团队负责不同的上下文
4. **变化率**：变化频率差异大的业务应该分开
5. **业务能力**：每个上下文代表一个独立的业务能力

**识别示例：**
```
"订单" 的含义：
- 电商上下文：包含商品、价格、配送地址
- 仓储上下文：包含库位、拣货员、拣货状态
- 财务上下文：包含发票信息、税额、对账状态
→ 三个不同的限界上下文
```

---

## 问题 3：实体和值对象的区别是什么？

**答：**

| 维度 | 实体（Entity） | 值对象（Value Object） |
|------|--------------|----------------------|
| 标识 | 有唯一ID，通过ID判断相等 | 无ID，通过属性值判断相等 |
| 可变性 | 状态可以改变 | 不可变（Immutable） |
| 生命周期 | 有独立的生命周期 | 依附于实体存在 |
| 相等性 | `order1.equals(order2)` 基于ID | `money1.equals(money2)` 基于所有属性 |
| 举例 | Order, Customer, Product | Money, Address, Email |

**代码示例：**
```java
// 实体：同一个 Order 即使金额改变，还是同一个 Order（同一个 ID）
Order order = ...; // orderId = "ORD-001"
order.applyCoupon(coupon);  // 金额变了，但还是 ORD-001 这个订单

// 值对象：100元CNY 和 100元CNY 完全相等，不区分"哪一个100元"
Money m1 = new Money(100, "CNY");
Money m2 = new Money(100, "CNY");
m1.equals(m2) == true  // 完全相等
```

---

## 问题 4：聚合和聚合根是什么？有什么设计原则？

**答：**

**聚合（Aggregate）**：一组需要保持业务一致性的对象集合，是事务的边界。
**聚合根（Aggregate Root）**：聚合的入口，外部只能通过聚合根访问聚合内的对象。

**四大设计原则：**
1. **聚合内强一致性**：同一事务内保证所有不变性规则
2. **聚合间最终一致性**：通过领域事件实现
3. **聚合尽量小**：避免大聚合，减少锁竞争
4. **通过 ID 引用其他聚合**：不直接持有对象引用

---

## 问题 5：领域服务什么时候使用？

**答：**

当一个业务行为**无法自然地归属于某个实体或值对象**时，使用领域服务。

**使用场景：**
- 操作涉及多个聚合（如：从账户A转账到账户B）
- 操作需要访问外部资源（如：汇率转换需要查汇率）
- 操作是纯粹的计算（如：定价服务综合多种折扣）

```java
// 转账不属于 Account 任何一方，应该用领域服务
public class MoneyTransferService {
    public void transfer(Account from, Account to, Money amount) {
        from.debit(amount);   // 账户A 的行为
        to.credit(amount);    // 账户B 的行为
    }
}
```

---

## 问题 6：应用服务和领域服务的区别？

**答：**

| 维度 | 应用服务（Application Service） | 领域服务（Domain Service） |
|------|-------------------------------|--------------------------|
| 位置 | 应用层 | 领域层 |
| 职责 | 用例编排，不含业务规则 | 含业务规则，跨聚合操作 |
| 事务 | 是（@Transactional） | 否（不应有事务注解） |
| 依赖 | 可依赖基础设施 | 不能依赖基础设施 |
| 举例 | PlaceOrderHandler | OrderPricingService |

---

## 问题 7：什么是领域事件？与 Spring ApplicationEvent 的关系？

**答：**

**领域事件**：表示业务领域中发生的、有意义的状态变化。以过去时命名（OrderPlaced）。

**与 Spring ApplicationEvent 的关系：**
- 领域事件是**业务概念**，定义在领域层，不依赖 Spring
- Spring ApplicationEvent 是**技术实现机制**，定义在基础设施层
- 实现方式：Spring 的 `ApplicationEventPublisher` 可以作为领域事件发布的技术实现

```java
// 领域层：纯 POJO，不依赖 Spring
public class OrderPaidEvent extends DomainEvent { ... }

// 基础设施层：用 Spring 机制发布
@Component
public class SpringDomainEventPublisher implements DomainEventPublisher {
    @Override
    public void publish(DomainEvent event) {
        applicationEventPublisher.publishEvent(event);
    }
}
```

---

## 问题 8：CQRS 是什么？为什么与 DDD 配合使用？

**答：**

CQRS（Command Query Responsibility Segregation，命令查询职责分离）将系统操作分为：
- **Command（命令）**：改变状态，不返回数据
- **Query（查询）**：不改变状态，只返回数据

**与 DDD 配合的原因：**
1. DDD 的聚合模型非常适合写操作（命令），能保证业务一致性
2. 但聚合加载方式不适合复杂查询（涉及多表 JOIN）
3. CQRS 让查询端可以绕过聚合，直接用 SQL/视图，性能更好
4. 读写分离后，各自可以独立优化和扩展

---

## 问题 9：什么是防腐层（ACL）？何时使用？

**答：**

防腐层（Anti-Corruption Layer）是在**本地领域模型和外部系统之间的翻译层**，
保护本地模型不被外部模型污染。

**使用场景：**
- 集成遗留系统
- 集成第三方服务（支付、物流等）
- 上下文映射中的"下游保护"

```java
// 场景：集成第三方物流，防止物流的概念污染订单域
public interface LogisticsService {                   // 领域层接口
    ShipmentInfo getShipmentInfo(TrackingNumber no);  // 领域模型
}

@Component
public class LogisticsServiceAcl implements LogisticsService {
    @Override
    public ShipmentInfo getShipmentInfo(TrackingNumber no) {
        LogisticsDTO dto = externalClient.query(no.getValue()); // 调外部
        return translate(dto);  // 翻译，隔离外部概念
    }
}
```

---

## 问题 10：贫血模型和充血模型谁更好？

**答：**

在复杂业务系统中，**充血模型更好**，原因：

1. **业务规则内聚**：规则在实体内部，不会散落在多处
2. **可测试性强**：领域对象是纯 POJO，无需 Spring 容器即可测试
3. **表达力强**：代码即文档，实体方法名直接表达业务意图
4. **防止规则重复**：业务规则只在一处定义

贫血模型适用场景：**简单 CRUD 系统**（此时 DDD 本身就是过度设计）。

---

## 问题 11：DDD 中的仓储（Repository）和 DAO 有什么区别？

**答：**

| 维度 | Repository（仓储） | DAO（数据访问对象） |
|------|------------------|--------------------|
| 面向对象 | 面向领域聚合 | 面向数据库表 |
| 操作单位 | 聚合根（含所有子对象） | 单张表的记录 |
| 业务语义 | 有（findByCustomerId） | 无（selectByField） |
| 接口位置 | 领域层 | 无（直接在 Mapper） |
| 实现位置 | 基础设施层 | 基础设施层 |

仓储的入口是聚合根，不应该有 `OrderItemRepository`，
OrderItem 只能通过 `OrderRepository` 访问。

---

## 问题 12：如何在 Spring Boot 项目中落地 DDD？

**答：**

核心步骤：
1. **明确领域划分**：进行 EventStorming，划分限界上下文
2. **建立四层架构**：interfaces / application / domain / infrastructure
3. **从领域层开始**：先定义实体、值对象、聚合，不依赖任何框架
4. **仓储接口在领域层**：基础设施层提供实现
5. **应用层管事务**：`@Transactional` 只在应用服务上
6. **领域事件解耦**：跨聚合用事件而不是直接调用

---

## 问题 13：聚合设计中，如何判断两个对象是否应该在同一个聚合？

**答：**

判断标准：**这两个对象是否需要在同一个事务中保持一致？**

```
判断流程：
修改 A 的时候，B 是否必须同步更新？
  YES → 考虑放在同一聚合
    B 的规模会导致聚合过大吗？
      YES → 可能需要重新设计（最终一致性是否可接受？）
      NO  → 放在同一聚合
  NO  → 用领域事件实现最终一致性
```

经验法则：
- `OrderItem` 属于 `Order` 聚合（删除订单时，订单项必须一起删除）
- `CustomerAddress` 不属于 `Order` 聚合（地址变更不影响已下单订单）

---

## 问题 14：什么是统一语言（Ubiquitous Language）？

**答：**

统一语言是 DDD 最核心的实践之一，指**领域专家和开发人员共同使用的一套语言**，
这套语言在代码（类名、方法名、变量名）、文档、对话中完全一致。

**实践方法：**
- 建立词汇表（Glossary），对每个业务术语达成共识
- 代码中的命名与业务术语一一对应
- 当业务专家说"下单"，代码中就有 `placeOrder()`

**反例（统一语言缺失）：**
```
业务专家说：客户提交购买申请
开发说：用户发送下单请求
数据库：INSERT INTO t_order
代码：createOrderRecord()
→ 四种说法，混乱不堪
```

**正例（统一语言）：**
```
业务专家说：客户下单
开发说：下单
数据库：t_order（order_placed_event）
代码：placeOrder() / OrderPlacedEvent
→ 一种说法，清晰一致
```

---

## 问题 15：如何处理 DDD 中的复杂查询（跨多个聚合的报表查询）？

**答：**

核心原则：**查询不走聚合，走专用的查询模型（CQRS 读端）**。

实现方式：
1. **直接 SQL**：MyBatis 写原生 SQL，允许跨表 JOIN
2. **物化视图**：对复杂报表预计算，定期刷新
3. **搜索引擎**：Elasticsearch 存储非规范化的读模型
4. **读写数据库分离**：写库走聚合，读库用只读副本

```java
// 读模型与写模型分离
@Repository
public class OrderReportRepositoryImpl {
    @Autowired
    private OrderReportMapper mapper;  // 专用读 Mapper

    public List<OrderReportVO> queryMonthlyReport(
            int year, int month, String customerId) {
        // 复杂 JOIN 查询，完全不走聚合
        return mapper.selectMonthlyReport(year, month, customerId);
    }
}
```

---

## 附录 A：DDD 概念速查表

```
DDD 核心概念速查

战略设计
├── 领域（Domain）：软件要解决的业务问题空间
│   ├── 核心域：最重要的竞争力所在
│   ├── 支撑域：支撑核心域运作
│   └── 通用域：通用技术能力
├── 子域（Subdomain）：领域的细分
├── 限界上下文（Bounded Context）：语义的边界
├── 统一语言（Ubiquitous Language）：领域专家+开发的共同语言
└── 上下文映射（Context Map）：上下文间的关系
    ├── 合作（Partnership）
    ├── 共享内核（Shared Kernel）
    ├── 客户-供应商（Customer-Supplier）
    ├── 遵奉者（Conformist）
    ├── 防腐层（Anti-Corruption Layer）★
    ├── 开放主机（Open Host Service）
    └── 发布语言（Published Language）

战术设计
├── 实体（Entity）：有唯一标识，状态可变
├── 值对象（Value Object）：无标识，不可变
├── 聚合（Aggregate）：一致性边界，一组对象集合
├── 聚合根（Aggregate Root）：聚合的入口，外部唯一访问点
├── 领域服务（Domain Service）：跨聚合的业务逻辑
├── 应用服务（Application Service）：用例编排
├── 领域事件（Domain Event）：业务状态变化通知
├── 仓储（Repository）：聚合的持久化抽象
└── 工厂（Factory）：复杂对象的创建
```

## 附录 B：常见反模式（Anti-Patterns）

```
DDD 常见反模式

1. 贫血领域模型（Anemic Domain Model）
   症状：实体只有 getter/setter，业务逻辑全在 Service
   危害：面向过程编程披上 OOP 的外衣
   解决：将业务规则移入实体

2. 大聚合（Fat Aggregate）
   症状：一个聚合包含几十个实体，加载缓慢
   危害：并发冲突多，事务大，性能差
   解决：按一致性边界拆小聚合，用事件解耦

3. 上帝服务（God Service）
   症状：所有业务都在一个 Service 类里（几千行）
   危害：高耦合，难维护，无法并行开发
   解决：按聚合/子域拆分，每个 Service 只负责一个聚合

4. 仓储变 DAO（Repository as DAO）
   症状：OrderItemRepository.findById() 存在
   危害：破坏聚合边界
   解决：Repository 只有聚合根级别的接口

5. 直接引用其他聚合对象
   症状：Order 里有 Customer 属性（而不是 CustomerId）
   危害：聚合间强耦合，加载时联级加载
   解决：只持有 ID（值对象），需要时在应用层加载

6. 跨聚合事务
   症状：一个事务同时修改 Order 和 Inventory 聚合
   危害：高并发下容易死锁，违反聚合自治原则
   解决：用领域事件+最终一致性
```

## 附录 C：推荐学习资源

```
DDD 推荐学习路径

入门（1-2 个月）：
1. Eric Evans《领域驱动设计》- 经典，必读（略偏理论）
2. Vaughn Vernon《实现领域驱动设计》- 更多代码，更实践
3. 陈皓《左耳听风》DDD 系列文章

进阶（2-4 个月）：
4. Alberto Brandolini EventStorming 官方网站
5. Martin Fowler《企业应用架构模式》- 了解背景
6. Udi Dahan CQRS/Event Sourcing 视频课程

实践：
7. 找一个真实的复杂业务场景，亲自做 EventStorming
8. 按 DDD 四层架构重构一个模块
9. 实现一个完整的聚合根（含领域事件）

开源参考项目：
- ddd-by-examples/library（GitHub）
- Event-Driven.io（GitHub）
- COLA（阿里开源 DDD 框架）
```

---

*文档结束 - 本文档包含 DDD 领域驱动设计完整知识体系，涵盖战略设计、战术设计、架构模式、代码实现及落地实践。*

*最后更新：2024 年 | 版本：1.0*

