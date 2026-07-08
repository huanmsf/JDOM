# JUnit5 + Mockito 测试详解

> 从单元测试入门到 Spring Boot 集成测试实战，全面掌握 Java 测试技术栈

---

## 目录

- [Part 1: 测试基础与理念](#part-1)
- [Part 2: JUnit 5 架构与配置](#part-2)
- [Part 3: JUnit 5 核心注解](#part-3)
- [Part 4: 断言 Assertions](#part-4)
- [Part 5: Mockito 核心用法](#part-5)
- [Part 6: Mockito 高级特性](#part-6)
- [Part 7: Spring Boot 测试](#part-7)
- [Part 8: TestContainers](#part-8)
- [Part 9: 测试覆盖率 JaCoCo](#part-9)
- [Part 10: 完整实战案例](#part-10)
- [Part 11: 测试最佳实践](#part-11)
- [Part 12: 高频面试题](#part-12)

---

## Part 1: 测试基础与理念 {#part-1}

### 1.1 为什么要写测试？

软件测试是保障代码质量的核心手段。良好的测试体系能够：

1. **快速发现回归 Bug**：修改代码后立即发现是否破坏了已有功能
2. **充当活文档**：测试用例即规格说明，清晰描述业务逻辑
3. **支撑重构**：有测试保护才敢大胆重构，减少对已有代码的恐惧
4. **提升设计质量**：可测试的代码通常具有低耦合、高内聚的特点
5. **缩短反馈周期**：在本地秒级运行，而非等待部署后人工验证

```
没有测试的代码库示意图：

  需求变更
     │
     ▼
  修改代码 ─────────────────────────────►  上线
     │                                        │
     │  (不知道改坏了什么)                    │
     │                                        ▼
     └──────────────────────────────►  生产 Bug 告警
```

```
有测试的代码库示意图：

  需求变更
     │
     ▼
  修改代码
     │
     ▼
  运行测试套件 ──► 失败 ──► 定位并修复问题
     │
     ▼ 全部通过
  上线（高置信度）
```

### 1.2 测试金字塔

```
           /\
          /  \
         / UI \         少量 E2E 测试（慢、成本高）
        /------\
       /        \
      / 集成测试  \       适量集成测试（中等速度）
     /------------\
    /              \
   /   单元测试      \    大量单元测试（快、成本低）
  /------------------\
```

| 层级 | 数量 | 速度 | 成本 | 说明 |
|------|------|------|------|------|
| 单元测试 | 70% | 毫秒级 | 低 | 测试单个类/方法 |
| 集成测试 | 20% | 秒级 | 中 | 测试多个组件协作 |
| E2E 测试 | 10% | 分钟级 | 高 | 模拟真实用户操作 |

### 1.3 TDD（测试驱动开发）

TDD 的核心循环：**红 → 绿 → 重构**

```
  ┌─────────────────────────────────────────┐
  │                                         │
  │  1. RED    写一个失败的测试              │
  │     ↓                                   │
  │  2. GREEN  写最少代码让测试通过          │
  │     ↓                                   │
  │  3. REFACTOR 重构代码，保持测试绿色     │
  │     ↓                                   │
  │  循环往复                               │
  └─────────────────────────────────────────┘
```

**TDD 示例**：

```java
// 第一步：写失败的测试（RED）
@Test
void should_return_fizz_when_divisible_by_3() {
    FizzBuzz fb = new FizzBuzz();
    assertEquals("Fizz", fb.convert(3));
}

// 第二步：写最少代码通过测试（GREEN）
public class FizzBuzz {
    public String convert(int n) {
        if (n % 3 == 0) return "Fizz";
        return String.valueOf(n);
    }
}

// 第三步：继续添加测试，驱动完整实现
@Test
void should_return_buzz_when_divisible_by_5() {
    assertEquals("Buzz", fb.convert(5));
}

@Test
void should_return_fizzbuzz_when_divisible_by_15() {
    assertEquals("FizzBuzz", fb.convert(15));
}
```

### 1.4 BDD（行为驱动开发）

BDD 强调用"业务语言"描述测试，采用 **Given-When-Then** 格式：

```java
@Test
@DisplayName("Given 账户余额100元，When 取款50元，Then 余额应为50元")
void should_deduct_balance_after_withdrawal() {
    // Given
    BankAccount account = new BankAccount(100.0);

    // When
    account.withdraw(50.0);

    // Then
    assertEquals(50.0, account.getBalance());
}
```

### 1.5 FIRST 原则

优秀的单元测试应遵循 FIRST 原则：

| 字母 | 含义 | 说明 |
|------|------|------|
| F | Fast（快速） | 测试要运行得很快，不能有数据库/网络等慢操作 |
| I | Independent（独立） | 测试之间不能相互依赖，可以任意顺序执行 |
| R | Repeatable（可重复） | 任何环境下多次运行结果一致 |
| S | Self-Validating（自验证） | 测试本身判断成功/失败，不依赖人工查看日志 |
| T | Timely（及时） | 与生产代码同步编写，而非事后补充 |

---

## Part 2: JUnit 5 架构与配置 {#part-2}

### 2.1 JUnit 5 三大模块

```
┌─────────────────────────────────────────────────────────┐
│                    JUnit 5                              │
├─────────────────┬───────────────────┬───────────────────┤
│  JUnit Platform │  JUnit Jupiter    │  JUnit Vintage    │
│                 │                   │                   │
│  测试引擎的     │  JUnit 5 的新     │  运行 JUnit 3/4   │
│  启动框架       │  编程模型和扩展   │  的兼容层         │
│                 │  模型（@Test等）  │                   │
└─────────────────┴───────────────────┴───────────────────┘
```

- **JUnit Platform**：在 JVM 上启动测试框架的基础，定义了 TestEngine API
- **JUnit Jupiter**：包含 JUnit 5 新的注解和编程模型，是我们主要使用的模块
- **JUnit Vintage**：提供向后兼容，让旧的 JUnit 4/3 测试也能在 JUnit 5 平台上运行

### 2.2 JUnit 5 vs JUnit 4 对比

| 特性 | JUnit 4 | JUnit 5 |
|------|---------|---------|
| 包名 | `org.junit` | `org.junit.jupiter.api` |
| 测试注解 | `@Test` | `@Test` |
| 测试前置 | `@Before` | `@BeforeEach` |
| 测试后置 | `@After` | `@AfterEach` |
| 类前置 | `@BeforeClass` | `@BeforeAll` |
| 类后置 | `@AfterClass` | `@AfterAll` |
| 忽略测试 | `@Ignore` | `@Disabled` |
| 异常测试 | `@Test(expected=...)` | `assertThrows()` |
| 超时测试 | `@Test(timeout=...)` | `@Timeout` |
| 参数化 | `@RunWith(Parameterized.class)` | `@ParameterizedTest` |
| 扩展机制 | `@RunWith` + `@Rule` | `@ExtendWith` |
| Lambda 支持 | 不支持 | 支持 |
| 嵌套测试 | 不支持 | `@Nested` |
| Java 版本 | Java 5+ | Java 8+ |

### 2.3 Maven 配置

```xml
<!-- pom.xml -->
<properties>
    <java.version>17</java.version>
    <junit.version>5.10.1</junit.version>
    <mockito.version>5.8.0</mockito.version>
    <assertj.version>3.24.2</assertj.version>
</properties>

<dependencies>
    <!-- JUnit 5 BOM（统一版本管理） -->
    <dependency>
        <groupId>org.junit</groupId>
        <artifactId>junit-bom</artifactId>
        <version>${junit.version}</version>
        <type>pom</type>
        <scope>import</scope>
    </dependency>

    <!-- JUnit Jupiter API（编写测试用） -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- JUnit Jupiter Engine（运行测试用） -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-engine</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- JUnit Jupiter Params（参数化测试用） -->
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-params</artifactId>
        <scope>test</scope>
    </dependency>

    <!-- Mockito -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-core</artifactId>
        <version>${mockito.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- Mockito JUnit 5 扩展 -->
    <dependency>
        <groupId>org.mockito</groupId>
        <artifactId>mockito-junit-jupiter</artifactId>
        <version>${mockito.version}</version>
        <scope>test</scope>
    </dependency>

    <!-- AssertJ 断言库 -->
    <dependency>
        <groupId>org.assertj</groupId>
        <artifactId>assertj-core</artifactId>
        <version>${assertj.version}</version>
        <scope>test</scope>
    </dependency>
</dependencies>

<build>
    <plugins>
        <!-- Maven Surefire Plugin（需要 3.x 版本才支持 JUnit 5） -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
        </plugin>
    </plugins>
</build>
```

### 2.4 Spring Boot 项目配置

Spring Boot 2.2+ 已内置 JUnit 5 支持，只需引入 `spring-boot-starter-test`：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
    <!-- 自动包含：JUnit 5, Mockito, AssertJ, Hamcrest, JsonPath 等 -->
</dependency>
```

### 2.5 Gradle 配置

```groovy
// build.gradle
dependencies {
    testImplementation platform('org.junit:junit-bom:5.10.1')
    testImplementation 'org.junit.jupiter:junit-jupiter'
    testImplementation 'org.mockito:mockito-core:5.8.0'
    testImplementation 'org.mockito:mockito-junit-jupiter:5.8.0'
    testImplementation 'org.assertj:assertj-core:3.24.2'
}

test {
    useJUnitPlatform()
}
```

---

## Part 3: JUnit 5 核心注解 {#part-3}

### 3.1 @Test

最基础的测试注解，标记一个方法为测试方法：

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class CalculatorTest {

    @Test
    void addition_should_return_correct_sum() {
        Calculator calc = new Calculator();
        int result = calc.add(2, 3);
        assertEquals(5, result);
    }

    @Test
    void division_by_zero_should_throw_exception() {
        Calculator calc = new Calculator();
        assertThrows(ArithmeticException.class, () -> calc.divide(10, 0));
    }
}
```

**注意**：JUnit 5 中测试方法不需要是 `public`，可以是包级别（无修饰符）。

### 3.2 @DisplayName

为测试类或测试方法提供自定义的显示名称，支持中文和特殊字符：

```java
@DisplayName("计算器测试套件")
class CalculatorTest {

    @Test
    @DisplayName("两个正整数相加应返回正确结果")
    void test_add_positive_numbers() {
        assertEquals(5, new Calculator().add(2, 3));
    }

    @Test
    @DisplayName("除数为零时应抛出 ArithmeticException")
    void test_divide_by_zero() {
        assertThrows(ArithmeticException.class,
            () -> new Calculator().divide(10, 0));
    }
}
```

### 3.3 生命周期注解

```java
@DisplayName("生命周期注解演示")
class LifecycleTest {

    @BeforeAll
    static void initAll() {
        // 整个测试类运行前执行一次（必须是 static）
        System.out.println("=== 测试类开始 ===");
    }

    @AfterAll
    static void tearDownAll() {
        // 整个测试类运行后执行一次（必须是 static）
        System.out.println("=== 测试类结束 ===");
    }

    @BeforeEach
    void init() {
        // 每个测试方法执行前都会调用
        System.out.println("--- 测试方法开始 ---");
    }

    @AfterEach
    void tearDown() {
        // 每个测试方法执行后都会调用
        System.out.println("--- 测试方法结束 ---");
    }

    @Test
    void test1() {
        System.out.println("执行 test1");
    }

    @Test
    void test2() {
        System.out.println("执行 test2");
    }
}
```

执行顺序：
```
=== 测试类开始 ===
--- 测试方法开始 ---
执行 test1
--- 测试方法结束 ---
--- 测试方法开始 ---
执行 test2
--- 测试方法结束 ---
=== 测试类结束 ===
```

### 3.4 @Disabled

跳过某个测试（类似 JUnit 4 的 `@Ignore`）：

```java
@Disabled("该功能尚未实现，待 Issue#123 完成后启用")
@Test
void future_feature_test() {
    // 该测试会被跳过
}

@Disabled("已知 Bug，待修复")
class BrokenFeatureTest {
    // 整个测试类都会被跳过
}
```

### 3.5 @Tag

给测试打标签，用于过滤执行：

```java
@Tag("fast")
@Tag("unit")
@Test
void fast_unit_test() { }

@Tag("slow")
@Tag("integration")
@Test
void slow_integration_test() { }
```

Maven 中按 Tag 过滤：
```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <configuration>
        <!-- 只运行 fast 标签的测试 -->
        <groups>fast</groups>
        <!-- 排除 slow 标签的测试 -->
        <excludedGroups>slow</excludedGroups>
    </configuration>
</plugin>
```

### 3.6 @Nested（嵌套测试）

用内部类组织相关测试，提高可读性：

```java
@DisplayName("订单服务测试")
class OrderServiceTest {

    OrderService orderService;

    @BeforeEach
    void setUp() {
        orderService = new OrderService();
    }

    @Nested
    @DisplayName("创建订单")
    class CreateOrder {

        @Test
        @DisplayName("正常商品应创建成功")
        void should_create_order_successfully() {
            Order order = orderService.create("iPhone 15", 1, 6999.0);
            assertNotNull(order.getId());
            assertEquals(OrderStatus.PENDING, order.getStatus());
        }

        @Test
        @DisplayName("库存不足时应抛出异常")
        void should_throw_when_out_of_stock() {
            assertThrows(OutOfStockException.class,
                () -> orderService.create("iPhone 15", 9999, 6999.0));
        }
    }

    @Nested
    @DisplayName("取消订单")
    class CancelOrder {

        @Test
        @DisplayName("待付款订单可以取消")
        void should_cancel_pending_order() {
            Order order = orderService.create("iPhone 15", 1, 6999.0);
            orderService.cancel(order.getId());
            assertEquals(OrderStatus.CANCELLED, order.getStatus());
        }

        @Test
        @DisplayName("已发货订单不可取消")
        void should_fail_to_cancel_shipped_order() {
            assertThrows(IllegalStateException.class,
                () -> orderService.cancel("shipped-order-id"));
        }
    }
}
```

### 3.7 @RepeatedTest（重复测试）

重复执行某个测试多次：

```java
@RepeatedTest(5)
void repeated_test() {
    assertTrue(new Random().nextInt(100) >= 0);
}

// 带自定义名称的重复测试
@RepeatedTest(value = 3, name = "第 {currentRepetition}/{totalRepetitions} 次执行")
void repeated_with_name(RepetitionInfo info) {
    System.out.println("当前第 " + info.getCurrentRepetition() + " 次");
}
```

### 3.8 @ParameterizedTest（参数化测试）

用不同参数运行同一测试，这是 JUnit 5 最强大的特性之一。

#### @ValueSource - 简单值数组

```java
@ParameterizedTest
@ValueSource(ints = {1, 2, 3, 4, 5})
void test_with_int_values(int number) {
    assertTrue(number > 0);
}

@ParameterizedTest
@ValueSource(strings = {"apple", "banana", "cherry"})
void test_with_string_values(String fruit) {
    assertFalse(fruit.isEmpty());
}

@ParameterizedTest
@ValueSource(classes = {String.class, Integer.class, Double.class})
void test_with_class_values(Class<?> type) {
    assertNotNull(type);
}
```

#### @NullSource / @EmptySource / @NullAndEmptySource

```java
@ParameterizedTest
@NullSource
void test_with_null(String input) {
    // input == null
    assertNull(input);
}

@ParameterizedTest
@EmptySource
void test_with_empty(String input) {
    // input == ""
    assertTrue(input.isEmpty());
}

@ParameterizedTest
@NullAndEmptySource
void test_with_null_and_empty(String input) {
    // 先用 null，再用 ""
    assertTrue(input == null || input.isEmpty());
}

// 组合使用
@ParameterizedTest
@NullAndEmptySource
@ValueSource(strings = {"  ", "\t", "\n"})
void test_blank_strings(String input) {
    // 测试 null、""、空白字符串
    assertTrue(input == null || input.isBlank());
}
```

#### @EnumSource - 枚举值

```java
enum Status { ACTIVE, INACTIVE, PENDING, DELETED }

@ParameterizedTest
@EnumSource(Status.class)
void test_all_statuses(Status status) {
    assertNotNull(status);
}

// 只测试部分枚举值
@ParameterizedTest
@EnumSource(value = Status.class, names = {"ACTIVE", "INACTIVE"})
void test_active_statuses(Status status) {
    assertTrue(status == Status.ACTIVE || status == Status.INACTIVE);
}

// 排除部分枚举值
@ParameterizedTest
@EnumSource(value = Status.class, mode = EXCLUDE, names = {"DELETED"})
void test_non_deleted(Status status) {
    assertNotEquals(Status.DELETED, status);
}
```

#### @CsvSource - CSV 内联数据

```java
@ParameterizedTest
@CsvSource({
    "apple,     1",
    "banana,    2",
    "'lemon, lime', 3",   // 含逗号的值用单引号包裹
    "strawberry, 4"
})
void test_with_csv(String fruit, int rank) {
    assertNotNull(fruit);
    assertTrue(rank > 0);
}

// 计算器测试
@ParameterizedTest(name = "{0} + {1} = {2}")
@CsvSource({
    "1, 2, 3",
    "5, 3, 8",
    "-1, 1, 0",
    "100, 200, 300"
})
void test_add(int a, int b, int expected) {
    assertEquals(expected, new Calculator().add(a, b));
}
```

#### @CsvFileSource - CSV 文件数据

```java
@ParameterizedTest
@CsvFileSource(resources = "/test-data/calculator.csv", numLinesToSkip = 1)
void test_from_csv_file(int a, int b, int expected) {
    assertEquals(expected, new Calculator().add(a, b));
}
```

`src/test/resources/test-data/calculator.csv`:
```csv
a,b,expected
1,2,3
5,3,8
-1,1,0
```

#### @MethodSource - 方法提供参数

```java
@ParameterizedTest
@MethodSource("provideStringsForIsBlank")
void test_isBlank(String input, boolean expected) {
    assertEquals(expected, input == null || input.isBlank());
}

// 参数提供方法（必须是 static）
static Stream<Arguments> provideStringsForIsBlank() {
    return Stream.of(
        Arguments.of(null, true),
        Arguments.of("", true),
        Arguments.of("  ", true),
        Arguments.of("not blank", false),
        Arguments.of("  also not  ", false)
    );
}

// 外部类的方法
@ParameterizedTest
@MethodSource("com.example.TestDataProvider#provideData")
void test_with_external_method(String data) { }
```

#### @ArgumentsSource - 自定义参数源

```java
@ParameterizedTest
@ArgumentsSource(CustomArgumentsProvider.class)
void test_with_custom_source(int number, String name) { }

class CustomArgumentsProvider implements ArgumentsProvider {
    @Override
    public Stream<? extends Arguments> provideArguments(ExtensionContext ctx) {
        return Stream.of(
            Arguments.of(1, "one"),
            Arguments.of(2, "two"),
            Arguments.of(3, "three")
        );
    }
}
```

### 3.9 @TestMethodOrder（测试顺序）

```java
@TestMethodOrder(MethodOrderer.OrderAnnotation.class)
class OrderedTest {

    @Test
    @Order(1)
    void first() { System.out.println("第一个"); }

    @Test
    @Order(2)
    void second() { System.out.println("第二个"); }

    @Test
    @Order(3)
    void third() { System.out.println("第三个"); }
}

// 按字母顺序
@TestMethodOrder(MethodOrderer.MethodName.class)
class AlphabeticalTest { }

// 随机顺序（用于发现测试间的隐式依赖）
@TestMethodOrder(MethodOrderer.Random.class)
class RandomOrderTest { }
```

### 3.10 @Timeout（超时控制）

```java
@Test
@Timeout(5)  // 5秒超时
void test_with_timeout() throws InterruptedException {
    // 如果超过 5 秒，测试失败
    Thread.sleep(4000);
}

@Test
@Timeout(value = 500, unit = TimeUnit.MILLISECONDS)
void test_fast_response() {
    // 必须在 500ms 内完成
}
```

---

## Part 4: 断言 Assertions {#part-4}

### 4.1 JUnit 5 内置断言

JUnit 5 的 `Assertions` 类提供了丰富的断言方法：

```java
import static org.junit.jupiter.api.Assertions.*;

class AssertionTest {

    // 基础断言
    @Test
    void basic_assertions() {
        // 相等断言
        assertEquals(5, 2 + 3);
        assertEquals(5, 2 + 3, "加法结果不正确");  // 自定义消息

        // 不等断言
        assertNotEquals(6, 2 + 3);

        // 真/假断言
        assertTrue(5 > 3);
        assertFalse(5 < 3);

        // null 断言
        assertNull(null);
        assertNotNull("not null");

        // 相同对象断言（==）
        String str = "hello";
        assertSame(str, str);
        assertNotSame("hello", new String("hello"));
    }

    // 数组断言
    @Test
    void array_assertions() {
        int[] expected = {1, 2, 3};
        int[] actual = {1, 2, 3};
        assertArrayEquals(expected, actual);

        double[] doubles = {1.0, 2.0, 3.0};
        assertArrayEquals(new double[]{1.0, 2.0, 3.0}, doubles, 0.001);
    }

    // 异常断言
    @Test
    void exception_assertions() {
        // 断言抛出指定异常
        ArithmeticException ex = assertThrows(
            ArithmeticException.class,
            () -> { int r = 10 / 0; }
        );
        assertEquals("/ by zero", ex.getMessage());

        // 断言不抛出异常
        assertDoesNotThrow(() -> {
            int r = 10 / 2;
        });
    }

    // 超时断言
    @Test
    void timeout_assertions() {
        // 断言在指定时间内完成
        assertTimeout(Duration.ofSeconds(1), () -> {
            Thread.sleep(500);
        });

        // 超时则直接中断（抢占式）
        assertTimeoutPreemptively(Duration.ofMillis(100), () -> {
            // 必须在100ms内完成
        });
    }

    // 组合断言（所有断言都执行，汇总失败信息）
    @Test
    void grouped_assertions() {
        User user = new User("Alice", 30, "alice@example.com");

        assertAll("user properties",
            () -> assertEquals("Alice", user.getName()),
            () -> assertEquals(30, user.getAge()),
            () -> assertEquals("alice@example.com", user.getEmail())
        );
        // 即使第一个失败，后两个也会执行
    }
}
```

### 4.2 AssertJ 流式断言

AssertJ 提供了更流畅、可读性更高的断言风格：

```java
import static org.assertj.core.api.Assertions.*;

class AssertJTest {

    // 字符串断言
    @Test
    void string_assertions() {
        String str = "Hello World";

        assertThat(str)
            .isNotNull()
            .isNotEmpty()
            .startsWith("Hello")
            .endsWith("World")
            .contains("lo W")
            .hasSize(11)
            .isEqualToIgnoringCase("hello world");
    }

    // 数字断言
    @Test
    void number_assertions() {
        int age = 25;

        assertThat(age)
            .isPositive()
            .isGreaterThan(18)
            .isLessThan(100)
            .isBetween(20, 30);

        double price = 9.99;
        assertThat(price).isCloseTo(10.0, within(0.1));
    }

    // 集合断言
    @Test
    void collection_assertions() {
        List<String> fruits = Arrays.asList("apple", "banana", "cherry");

        assertThat(fruits)
            .isNotNull()
            .isNotEmpty()
            .hasSize(3)
            .contains("apple", "banana")
            .containsExactly("apple", "banana", "cherry")  // 严格顺序
            .doesNotContain("durian")
            .allMatch(f -> f.length() > 3);
    }

    // 对象断言
    @Test
    void object_assertions() {
        User user = new User("Alice", 30);

        assertThat(user)
            .isNotNull()
            .extracting(User::getName, User::getAge)
            .containsExactly("Alice", 30);
    }

    // 异常断言（AssertJ 风格）
    @Test
    void exception_assertions() {
        assertThatThrownBy(() -> new Calculator().divide(10, 0))
            .isInstanceOf(ArithmeticException.class)
            .hasMessage("/ by zero");

        assertThatExceptionOfType(IllegalArgumentException.class)
            .isThrownBy(() -> new User(null, -1))
            .withMessageContaining("name");
    }

    // Map 断言
    @Test
    void map_assertions() {
        Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87);

        assertThat(scores)
            .hasSize(2)
            .containsKey("Alice")
            .containsEntry("Alice", 95)
            .doesNotContainKey("Charlie");
    }
}
```

### 4.3 软断言（Soft Assertions）

软断言会执行所有断言后，一次性报告所有失败：

```java
// JUnit 5 软断言
import org.junit.jupiter.api.extension.ExtendWith;
import org.assertj.core.api.SoftAssertions;
import org.assertj.core.api.junit.jupiter.SoftAssertionsExtension;

@ExtendWith(SoftAssertionsExtension.class)
class SoftAssertionTest {

    @Test
    void soft_assertions_example(SoftAssertions softly) {
        User user = getUserFromApi();

        // 所有断言都会执行
        softly.assertThat(user.getName()).isEqualTo("Alice");
        softly.assertThat(user.getAge()).isEqualTo(30);
        softly.assertThat(user.getEmail()).contains("@");

        // 不需要手动调用 softly.assertAll()，扩展会自动处理
    }

    // 手动使用软断言
    @Test
    void manual_soft_assertions() {
        SoftAssertions softly = new SoftAssertions();

        softly.assertThat(1 + 1).isEqualTo(3);  // 失败但继续
        softly.assertThat("hello").isEqualTo("world");  // 失败但继续
        softly.assertThat(true).isFalse();  // 失败但继续

        softly.assertAll();  // 这里汇总报告所有失败
    }
}
```

---

## Part 5: Mockito 核心用法 {#part-5}

### 5.1 Mock、Stub、Spy 的区别

```
概念区分：

Mock  - 完全虚假的对象，所有方法默认返回空值/0/false
        主要用于验证交互（方法是否被调用）

Stub  - 为特定调用预设返回值的对象
        主要用于控制被测对象的依赖行为

Spy   - 真实对象的部分替换，默认调用真实方法
        可以选择性地替换某些方法
```

### 5.2 Mockito 注解

```java
// 方式1：使用 @ExtendWith（推荐）
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    UserRepository userRepository;    // 完全Mock的对象

    @Mock
    EmailService emailService;

    @Spy
    PasswordEncoder encoder = new BCryptPasswordEncoder();  // Spy对象

    @InjectMocks
    UserService userService;          // 被测对象，自动注入上面的Mock

    @Captor
    ArgumentCaptor<User> userCaptor;  // 参数捕获器

    @Test
    void should_create_user() {
        // 直接使用 userService，它的依赖已被注入
    }
}

// 方式2：手动创建（不推荐，但了解）
class ManualMockTest {
    @Test
    void manual_mock() {
        UserRepository repo = Mockito.mock(UserRepository.class);
        UserService service = new UserService(repo);
        // ...
    }
}
```

### 5.3 打桩（Stubbing）

打桩是指预设 Mock 对象方法的返回值：

```java
@ExtendWith(MockitoExtension.class)
class StubbingTest {

    @Mock
    UserRepository userRepository;

    // --- when(...).thenReturn(...) ---
    @Test
    void stub_return_value() {
        User mockUser = new User(1L, "Alice", "alice@example.com");

        // 当调用 findById(1L) 时，返回 mockUser
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));

        Optional<User> result = userRepository.findById(1L);
        assertTrue(result.isPresent());
        assertEquals("Alice", result.get().getName());
    }

    // --- 链式返回不同值 ---
    @Test
    void stub_multiple_calls() {
        when(userRepository.count())
            .thenReturn(0L)       // 第一次调用返回 0
            .thenReturn(1L)       // 第二次调用返回 1
            .thenReturn(2L);      // 第三次及以后返回 2

        assertEquals(0L, userRepository.count());
        assertEquals(1L, userRepository.count());
        assertEquals(2L, userRepository.count());
        assertEquals(2L, userRepository.count()); // 仍是 2
    }

    // --- thenThrow ---
    @Test
    void stub_throw_exception() {
        when(userRepository.findById(999L))
            .thenThrow(new EntityNotFoundException("User not found: 999"));

        assertThrows(EntityNotFoundException.class,
            () -> userRepository.findById(999L));
    }

    // --- thenAnswer（动态返回值） ---
    @Test
    void stub_with_answer() {
        when(userRepository.save(any(User.class)))
            .thenAnswer(invocation -> {
                User user = invocation.getArgument(0);
                user.setId(100L);  // 模拟数据库生成ID
                return user;
            });

        User user = new User(null, "Bob", "bob@example.com");
        User saved = userRepository.save(user);
        assertEquals(100L, saved.getId());
    }

    // --- doReturn（void方法或Spy的打桩） ---
    @Test
    void stub_void_method() {
        // void 方法不能用 when().thenReturn()，要用 doNothing/doThrow
        doNothing().when(userRepository).delete(any(User.class));
        doThrow(new RuntimeException("DB Error"))
            .when(userRepository).deleteById(999L);

        // 不抛异常
        userRepository.delete(new User());

        // 抛异常
        assertThrows(RuntimeException.class,
            () -> userRepository.deleteById(999L));
    }
}
```

### 5.4 参数匹配器（Argument Matchers）

```java
@ExtendWith(MockitoExtension.class)
class ArgumentMatcherTest {

    @Mock
    UserRepository repo;

    @Test
    void argument_matchers() {
        // any() - 匹配任何对象（包括null）
        when(repo.findById(any())).thenReturn(Optional.empty());

        // anyLong(), anyString(), anyInt() 等
        when(repo.findById(anyLong())).thenReturn(Optional.empty());

        // eq() - 精确匹配
        when(repo.findByName(eq("Alice"))).thenReturn(List.of(new User()));

        // 字符串匹配
        when(repo.findByNameContaining(startsWith("Al")))
            .thenReturn(List.of());

        // 自定义匹配器（argThat）
        when(repo.save(argThat(user ->
            user.getAge() != null && user.getAge() > 0)))
            .thenAnswer(inv -> inv.getArgument(0));
    }

    // 重要规则：混用精确值和匹配器时，所有参数都必须用匹配器
    @Test
    void matcher_rule() {
        // 错误！不能混用
        // when(repo.findByNameAndAge("Alice", anyInt())).thenReturn(...);

        // 正确：所有参数都用匹配器
        when(repo.findByNameAndAge(eq("Alice"), anyInt()))
            .thenReturn(List.of());
    }
}
```

### 5.5 验证（Verification）

```java
@ExtendWith(MockitoExtension.class)
class VerificationTest {

    @Mock
    EmailService emailService;

    @Mock
    UserRepository userRepository;

    @InjectMocks
    UserService userService;

    // 验证方法被调用
    @Test
    void verify_method_called() {
        when(userRepository.save(any())).thenReturn(new User(1L, "Alice", "a@b.c"));

        userService.register("Alice", "alice@example.com");

        // 验证 emailService.sendWelcomeEmail 被调用了1次
        verify(emailService).sendWelcomeEmail("alice@example.com");

        // 等同于 times(1)
        verify(emailService, times(1)).sendWelcomeEmail(anyString());
    }

    // 验证调用次数
    @Test
    void verify_call_times() {
        userService.sendReminder(1L);

        verify(emailService, times(2)).sendEmail(anyString()); // 精确2次
        verify(emailService, atLeast(1)).sendEmail(anyString()); // 至少1次
        verify(emailService, atMost(3)).sendEmail(anyString());  // 至多3次
        verify(emailService, never()).sendWelcomeEmail(anyString()); // 从不调用
    }

    // 验证调用顺序
    @Test
    void verify_order() {
        userService.processAndNotify(1L);

        InOrder inOrder = inOrder(userRepository, emailService);
        inOrder.verify(userRepository).findById(1L);
        inOrder.verify(emailService).sendEmail(anyString());
    }

    // 验证没有更多交互
    @Test
    void verify_no_more_interactions() {
        when(userRepository.findById(1L)).thenReturn(Optional.of(new User()));

        userService.getUser(1L);

        verify(userRepository).findById(1L);
        verifyNoMoreInteractions(userRepository); // 没有其他未验证的调用
        verifyNoInteractions(emailService);        // emailService 完全没被调用
    }
}
```

### 5.6 参数捕获器（ArgumentCaptor）

捕获传递给 Mock 方法的参数，用于详细验证：

```java
@ExtendWith(MockitoExtension.class)
class ArgumentCaptorTest {

    @Mock
    UserRepository userRepository;

    @Captor
    ArgumentCaptor<User> userCaptor;

    @InjectMocks
    UserService userService;

    @Test
    void capture_saved_user() {
        when(userRepository.save(any(User.class)))
            .thenAnswer(inv -> inv.getArgument(0));

        userService.register("Alice", "alice@example.com");

        // 捕获传递给 save() 的 User 对象
        verify(userRepository).save(userCaptor.capture());

        User savedUser = userCaptor.getValue();
        assertEquals("Alice", savedUser.getName());
        assertEquals("alice@example.com", savedUser.getEmail());
        assertNotNull(savedUser.getCreatedAt());
    }

    @Test
    void capture_multiple_calls() {
        userService.registerBatch(List.of("Alice", "Bob", "Charlie"));

        verify(userRepository, times(3)).save(userCaptor.capture());

        List<User> allSaved = userCaptor.getAllValues();
        assertEquals(3, allSaved.size());
        assertEquals("Alice", allSaved.get(0).getName());
        assertEquals("Bob", allSaved.get(1).getName());
        assertEquals("Charlie", allSaved.get(2).getName());
    }
}
```

---

## Part 6: Mockito 高级特性 {#part-6}

### 6.1 静态方法 Mock（Mockito 3.4+）

```java
// 需要额外依赖：mockito-inline
// <dependency>
//     <groupId>org.mockito</groupId>
//     <artifactId>mockito-inline</artifactId>
//     <version>5.8.0</version>
//     <scope>test</scope>
// </dependency>

@Test
void mock_static_method() {
    try (MockedStatic<UUID> mockedUUID = mockStatic(UUID.class)) {
        UUID fixedUUID = UUID.fromString("123e4567-e89b-12d3-a456-426614174000");
        mockedUUID.when(UUID::randomUUID).thenReturn(fixedUUID);

        // 在此 try 块内，UUID.randomUUID() 返回固定值
        String id = orderService.createOrder();
        assertEquals("123e4567-e89b-12d3-a456-426614174000", id);
    }
    // try 块外，UUID.randomUUID() 恢复正常
}

@Test
void mock_static_with_args() {
    try (MockedStatic<LocalDate> mockedDate = mockStatic(LocalDate.class)) {
        LocalDate fixedDate = LocalDate.of(2024, 1, 15);
        mockedDate.when(LocalDate::now).thenReturn(fixedDate);

        String result = service.getTodayFormatted();
        assertEquals("2024-01-15", result);
    }
}
```

### 6.2 构造函数 Mock

```java
@Test
void mock_constructor() {
    try (MockedConstruction<HttpClient> mockedConstruction =
            mockConstruction(HttpClient.class, (mock, ctx) -> {
                when(mock.get(anyString())).thenReturn("mocked response");
            })) {

        // 在此 try 块内，new HttpClient() 返回 Mock 对象
        String result = apiService.fetchData("https://api.example.com/data");
        assertEquals("mocked response", result);
    }
}
```

### 6.3 Spy（间谍对象）

Spy 包装真实对象，默认调用真实方法，可以选择性替换：

```java
@ExtendWith(MockitoExtension.class)
class SpyTest {

    @Spy
    List<String> spyList = new ArrayList<>();

    @Test
    void spy_real_object() {
        // 真实方法被调用
        spyList.add("apple");
        spyList.add("banana");

        assertEquals(2, spyList.size());  // 真实size()

        // 替换 size() 方法
        doReturn(100).when(spyList).size();
        assertEquals(100, spyList.size());  // 返回伪造值
    }

    @Test
    void spy_partial_mock() {
        EmailService realService = new EmailService();
        EmailService spy = spy(realService);

        // 替换发送逻辑（避免真实发送邮件）
        doNothing().when(spy).actualSend(anyString(), anyString());

        spy.sendWelcomeEmail("user@example.com");

        // 验证内部流程（模板生成等）被调用
        verify(spy).buildEmailTemplate("user@example.com");
        verify(spy).actualSend(eq("user@example.com"), anyString());
    }
}
```

**Spy 打桩注意事项**：
```java
// 对 Spy 打桩时，必须用 doReturn，不能用 when().thenReturn()
// 错误（会调用真实方法）：
when(spyList.get(0)).thenReturn("first");  // 可能抛出 IndexOutOfBoundsException

// 正确：
doReturn("first").when(spyList).get(0);
```

### 6.4 Answer 接口（动态响应）

```java
@Test
void custom_answer() {
    when(userRepository.save(any(User.class)))
        .thenAnswer(new Answer<User>() {
            private long idCounter = 1;

            @Override
            public User answer(InvocationOnMock invocation) {
                User user = invocation.getArgument(0);
                user.setId(idCounter++);
                user.setCreatedAt(LocalDateTime.now());
                return user;
            }
        });

    User u1 = userRepository.save(new User("Alice"));
    User u2 = userRepository.save(new User("Bob"));

    assertEquals(1L, u1.getId());
    assertEquals(2L, u2.getId());
}

// 使用 Lambda 简化
@Test
void lambda_answer() {
    when(cache.get(anyString()))
        .thenAnswer(inv -> {
            String key = inv.getArgument(0);
            return "cached:" + key;
        });

    assertEquals("cached:user:1", cache.get("user:1"));
}
```

### 6.5 BDD 风格的 Mockito

```java
import static org.mockito.BDDMockito.*;

@ExtendWith(MockitoExtension.class)
class BDDStyleTest {

    @Mock
    UserRepository userRepository;

    @InjectMocks
    UserService userService;

    @Test
    void bdd_style_test() {
        // Given（使用 given 替代 when）
        User mockUser = new User(1L, "Alice", "alice@example.com");
        given(userRepository.findById(1L)).willReturn(Optional.of(mockUser));

        // When
        User result = userService.getUser(1L);

        // Then（使用 then 替代 verify）
        then(userRepository).should().findById(1L);
        then(userRepository).shouldHaveNoMoreInteractions();

        assertThat(result.getName()).isEqualTo("Alice");
    }

    @Test
    void bdd_exception_test() {
        // Given
        given(userRepository.findById(999L))
            .willThrow(new EntityNotFoundException("Not found"));

        // When / Then
        assertThatThrownBy(() -> userService.getUser(999L))
            .isInstanceOf(UserNotFoundException.class);
    }
}
```

### 6.6 重置 Mock

```java
@Test
void reset_mock() {
    // 打桩
    when(userRepository.count()).thenReturn(10L);
    assertEquals(10L, userRepository.count());

    // 重置（清除所有打桩和验证记录）
    reset(userRepository);

    // 重置后默认返回 0
    assertEquals(0L, userRepository.count());
}
```

### 6.7 忽略打桩（lenient）

```java
@ExtendWith(MockitoExtension.class)
class LenientTest {

    @Mock
    UserRepository userRepository;

    @Test
    void lenient_stubbing() {
        // 默认情况下，未使用的打桩会触发 UnnecessaryStubbingException
        // 使用 lenient() 可以忽略这个检查
        lenient().when(userRepository.findById(999L))
            .thenReturn(Optional.empty());

        // 即使这个打桩没被用到，测试也不会失败
        assertEquals(0L, userRepository.count());
    }
}
```

---

## Part 7: Spring Boot 测试 {#part-7}

### 7.1 @SpringBootTest（全上下文测试）

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerIntegrationTest {

    @Autowired
    TestRestTemplate restTemplate;

    @Autowired
    UserRepository userRepository;

    @BeforeEach
    void setUp() {
        userRepository.deleteAll();
    }

    @Test
    void should_create_user_and_return_201() {
        CreateUserRequest request = new CreateUserRequest("Alice", "alice@example.com");

        ResponseEntity<UserResponse> response = restTemplate.postForEntity(
            "/api/users",
            request,
            UserResponse.class
        );

        assertEquals(HttpStatus.CREATED, response.getStatusCode());
        assertNotNull(response.getBody());
        assertEquals("Alice", response.getBody().getName());
    }

    @Test
    void should_return_404_when_user_not_found() {
        ResponseEntity<String> response = restTemplate.getForEntity(
            "/api/users/9999",
            String.class
        );

        assertEquals(HttpStatus.NOT_FOUND, response.getStatusCode());
    }
}
```

**WebEnvironment 选项**：

| 值 | 说明 |
|----|------|
| `MOCK`（默认） | Mock Servlet 环境，不启动真实 HTTP 服务器 |
| `RANDOM_PORT` | 启动真实服务器，随机端口 |
| `DEFINED_PORT` | 启动真实服务器，使用配置的端口 |
| `NONE` | 不加载 Servlet 环境 |

### 7.2 @WebMvcTest（Controller 层测试）

只加载 Web 层相关的 Bean，速度快：

```java
@WebMvcTest(UserController.class)
class UserControllerTest {

    @Autowired
    MockMvc mockMvc;

    @MockBean  // 替换 Spring Context 中的 Bean
    UserService userService;

    @Autowired
    ObjectMapper objectMapper;

    @Test
    void should_return_user_when_found() throws Exception {
        // Given
        UserResponse user = new UserResponse(1L, "Alice", "alice@example.com");
        when(userService.getUser(1L)).thenReturn(user);

        // When & Then
        mockMvc.perform(get("/api/users/1")
                .contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("Alice"))
            .andExpect(jsonPath("$.email").value("alice@example.com"))
            .andDo(print());  // 打印请求/响应详情
    }

    @Test
    void should_return_400_when_invalid_request() throws Exception {
        CreateUserRequest invalidRequest = new CreateUserRequest("", "not-an-email");

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidRequest)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray());
    }

    @Test
    void should_return_201_when_user_created() throws Exception {
        CreateUserRequest request = new CreateUserRequest("Bob", "bob@example.com");
        UserResponse created = new UserResponse(2L, "Bob", "bob@example.com");
        when(userService.createUser(any())).thenReturn(created);

        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().string("Location", containsString("/api/users/2")));
    }
}
```

### 7.3 @DataJpaTest（JPA 层测试）

只加载 JPA 相关 Bean，使用内存数据库：

```java
@DataJpaTest
class UserRepositoryTest {

    @Autowired
    TestEntityManager entityManager;

    @Autowired
    UserRepository userRepository;

    @Test
    void should_find_user_by_email() {
        // Given
        User user = new User("Alice", "alice@example.com");
        entityManager.persistAndFlush(user);

        // When
        Optional<User> found = userRepository.findByEmail("alice@example.com");

        // Then
        assertTrue(found.isPresent());
        assertEquals("Alice", found.get().getName());
    }

    @Test
    void should_return_empty_when_email_not_found() {
        Optional<User> found = userRepository.findByEmail("notexist@example.com");
        assertFalse(found.isPresent());
    }

    @Test
    void should_find_active_users() {
        // Given
        entityManager.persist(new User("Alice", "a@b.c", UserStatus.ACTIVE));
        entityManager.persist(new User("Bob", "b@b.c", UserStatus.INACTIVE));
        entityManager.persist(new User("Charlie", "c@b.c", UserStatus.ACTIVE));
        entityManager.flush();

        // When
        List<User> activeUsers = userRepository.findByStatus(UserStatus.ACTIVE);

        // Then
        assertThat(activeUsers).hasSize(2)
            .extracting(User::getName)
            .containsExactlyInAnyOrder("Alice", "Charlie");
    }
}
```

使用真实数据库（而非 H2）：

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryRealDbTest {
    // 使用 application-test.properties 中配置的真实数据库
}
```

### 7.4 @MockBean 和 @SpyBean

```java
@SpringBootTest
class NotificationServiceTest {

    @Autowired
    NotificationService notificationService;

    @MockBean  // 用 Mock 替换 Spring 容器中的 EmailService
    EmailService emailService;

    @SpyBean  // 用 Spy 包装 Spring 容器中的 SmsService
    SmsService smsService;

    @Test
    void should_send_notification() {
        notificationService.notify(1L, "Your order has been shipped!");

        verify(emailService).send(anyString(), contains("shipped"));
        verify(smsService).send(anyString(), contains("shipped"));
    }
}
```

### 7.5 MockMvc 高级用法

```java
@WebMvcTest
class AdvancedMockMvcTest {

    @Autowired
    MockMvc mockMvc;

    // 文件上传测试
    @Test
    void should_upload_file() throws Exception {
        MockMultipartFile file = new MockMultipartFile(
            "file", "test.txt", "text/plain", "Hello World".getBytes()
        );

        mockMvc.perform(multipart("/api/files/upload")
                .file(file))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.filename").value("test.txt"));
    }

    // 认证测试
    @Test
    @WithMockUser(username = "admin", roles = {"ADMIN"})
    void should_allow_admin_access() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(username = "user", roles = {"USER"})
    void should_deny_user_access_to_admin() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isForbidden());
    }

    // 请求参数和 Header 测试
    @Test
    void should_filter_users_by_params() throws Exception {
        mockMvc.perform(get("/api/users")
                .param("status", "ACTIVE")
                .param("page", "0")
                .param("size", "10")
                .header("X-Request-ID", "req-123")
                .accept(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk())
            .andExpect(header().string("X-Request-ID", "req-123"));
    }
}
```

### 7.6 @TestConfiguration

```java
@TestConfiguration
class TestConfig {

    @Bean
    @Primary  // 覆盖同类型的 Bean
    public Clock fixedClock() {
        return Clock.fixed(
            Instant.parse("2024-01-15T10:00:00Z"),
            ZoneId.of("UTC")
        );
    }
}

// 在测试类中使用
@SpringBootTest
@Import(TestConfig.class)
class TimeAwareServiceTest {

    @Autowired
    TimeAwareService service;

    @Test
    void should_use_fixed_time() {
        String result = service.getCurrentTimeFormatted();
        assertEquals("2024-01-15 10:00:00", result);
    }
}
```

### 7.7 测试属性配置

```java
// 使用测试专用配置文件
@SpringBootTest
@TestPropertySource(locations = "classpath:application-test.properties")
class WithTestPropertiesTest { }

// 直接在注解中指定属性
@SpringBootTest(properties = {
    "app.feature.enabled=true",
    "spring.datasource.url=jdbc:h2:mem:testdb"
})
class WithInlinePropertiesTest { }

// 使用 @ActiveProfiles
@SpringBootTest
@ActiveProfiles("test")
class WithProfileTest { }
```

---

## Part 8: TestContainers {#part-8}

TestContainers 允许在测试中使用真实的 Docker 容器，避免使用内存数据库带来的差异。

### 8.1 依赖配置

```xml
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>mysql</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>redis</artifactId>
    <version>1.19.3</version>
    <scope>test</scope>
</dependency>
```

### 8.2 MySQL 容器测试

```java
@Testcontainers
@SpringBootTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryContainerTest {

    @Container
    static MySQLContainer<?> mysql = new MySQLContainer<>("mysql:8.0")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }

    @Autowired
    UserRepository userRepository;

    @Test
    void should_persist_and_retrieve_user() {
        User user = new User("Alice", "alice@example.com");
        userRepository.save(user);

        Optional<User> found = userRepository.findByEmail("alice@example.com");
        assertTrue(found.isPresent());
    }
}
```

### 8.3 Redis 容器测试

```java
@Testcontainers
@SpringBootTest
class CacheServiceContainerTest {

    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:7-alpine")
        .withExposedPorts(6379);

    @DynamicPropertySource
    static void configureRedis(DynamicPropertyRegistry registry) {
        registry.add("spring.redis.host", redis::getHost);
        registry.add("spring.redis.port", () -> redis.getMappedPort(6379));
    }

    @Autowired
    CacheService cacheService;

    @Test
    void should_cache_and_retrieve_value() {
        cacheService.set("key1", "value1", Duration.ofMinutes(5));

        Optional<String> result = cacheService.get("key1");

        assertTrue(result.isPresent());
        assertEquals("value1", result.get());
    }
}
```

### 8.4 Kafka 容器测试

```java
@Testcontainers
@SpringBootTest
class OrderEventConsumerTest {

    @Container
    static KafkaContainer kafka = new KafkaContainer(
        DockerImageName.parse("confluentinc/cp-kafka:7.5.0"));

    @DynamicPropertySource
    static void configureKafka(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", kafka::getBootstrapServers);
    }

    @Autowired
    KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Autowired
    OrderEventConsumer consumer;

    @Test
    void should_process_order_event() throws InterruptedException {
        OrderEvent event = new OrderEvent("order-123", "CREATED");

        kafkaTemplate.send("order-events", event);

        // 等待消费者处理
        Thread.sleep(2000);

        verify(consumer, times(1)).processOrderCreated(
            argThat(e -> e.getOrderId().equals("order-123"))
        );
    }
}
```

### 8.5 共享容器（提高测试性能）

```java
// 通用基类，容器在所有子类测试中共享
abstract class BaseIntegrationTest {

    @Container
    protected static final MySQLContainer<?> mysql =
        new MySQLContainer<>("mysql:8.0")
            .withDatabaseName("testdb")
            .withUsername("test")
            .withPassword("test");

    static {
        mysql.start();
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", mysql::getJdbcUrl);
        registry.add("spring.datasource.username", mysql::getUsername);
        registry.add("spring.datasource.password", mysql::getPassword);
    }
}

// 子类复用容器
@SpringBootTest
class UserServiceTest extends BaseIntegrationTest {
    // 使用共享的 mysql 容器
}

@SpringBootTest
class OrderServiceTest extends BaseIntegrationTest {
    // 使用同一个 mysql 容器，不会重新启动
}
```

---

## Part 9: 测试覆盖率 JaCoCo {#part-9}

### 9.1 JaCoCo 简介

JaCoCo（Java Code Coverage）是 Java 主流的测试覆盖率工具，支持：
- **行覆盖率（Line Coverage）**：哪些代码行被执行
- **分支覆盖率（Branch Coverage）**：if/else/switch 的分支是否都被测试
- **方法覆盖率（Method Coverage）**：哪些方法被调用
- **类覆盖率（Class Coverage）**：哪些类被测试到

### 9.2 Maven 配置

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.11</version>
            <executions>
                <!-- 准备 JaCoCo Agent -->
                <execution>
                    <id>prepare-agent</id>
                    <goals><goal>prepare-agent</goal></goals>
                </execution>
                <!-- 生成报告 -->
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals><goal>report</goal></goals>
                </execution>
                <!-- 覆盖率检查（低于阈值则构建失败） -->
                <execution>
                    <id>check</id>
                    <goals><goal>check</goal></goals>
                    <configuration>
                        <rules>
                            <rule>
                                <element>BUNDLE</element>
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum>  <!-- 80% 行覆盖率 -->
                                    </limit>
                                    <limit>
                                        <counter>BRANCH</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.70</minimum>  <!-- 70% 分支覆盖率 -->
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
            <configuration>
                <!-- 排除不需要统计覆盖率的类 -->
                <excludes>
                    <exclude>**/config/**</exclude>
                    <exclude>**/dto/**</exclude>
                    <exclude>**/*Application.class</exclude>
                    <exclude>**/entity/**</exclude>
                </excludes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

运行命令：
```bash
# 运行测试并生成覆盖率报告
mvn clean test

# 报告路径：target/site/jacoco/index.html
```

### 9.3 Gradle 配置

```groovy
plugins {
    id 'jacoco'
}

jacoco {
    toolVersion = "0.8.11"
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/config/**',
                '**/dto/**',
                '**/*Application.class'
            ])
        }))
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            limit {
                counter = 'LINE'
                value = 'COVEREDRATIO'
                minimum = 0.8
            }
        }
    }
}

test.finalizedBy jacocoTestReport
check.dependsOn jacocoTestCoverageVerification
```

### 9.4 排除特定代码

```java
// 方法级别排除
@ExcludeFromJacocoGeneratedReport
public String generatedMethod() { }

// 或使用 Lombok 的 @Generated
@Generated
public String toString() { }
```

---

## Part 10: 完整实战案例 {#part-10}

### 案例 1：用户注册服务测试

**业务代码**：

```java
@Service
@RequiredArgsConstructor
public class UserRegistrationService {

    private final UserRepository userRepository;
    private final EmailService emailService;
    private final PasswordEncoder passwordEncoder;

    public UserResponse register(RegisterRequest request) {
        // 1. 检查邮箱是否已注册
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new EmailAlreadyExistsException(request.getEmail());
        }

        // 2. 加密密码
        String encodedPassword = passwordEncoder.encode(request.getPassword());

        // 3. 创建用户
        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(encodedPassword)
            .status(UserStatus.PENDING_VERIFICATION)
            .createdAt(LocalDateTime.now())
            .build();

        User saved = userRepository.save(user);

        // 4. 发送验证邮件
        emailService.sendVerificationEmail(saved.getEmail(), saved.getId());

        return UserResponse.from(saved);
    }
}
```

**完整测试**：

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("用户注册服务测试")
class UserRegistrationServiceTest {

    @Mock
    UserRepository userRepository;

    @Mock
    EmailService emailService;

    @Mock
    PasswordEncoder passwordEncoder;

    @InjectMocks
    UserRegistrationService service;

    @Captor
    ArgumentCaptor<User> userCaptor;

    @Nested
    @DisplayName("注册成功场景")
    class RegisterSuccess {

        @Test
        @DisplayName("正常注册应返回用户信息并发送验证邮件")
        void should_register_successfully() {
            // Given
            RegisterRequest request = new RegisterRequest(
                "alice", "alice@example.com", "Password123!"
            );

            when(userRepository.existsByEmail("alice@example.com"))
                .thenReturn(false);
            when(passwordEncoder.encode("Password123!"))
                .thenReturn("$2a$10$encoded");
            when(userRepository.save(any(User.class)))
                .thenAnswer(inv -> {
                    User u = inv.getArgument(0);
                    u.setId(1L);
                    return u;
                });

            // When
            UserResponse response = service.register(request);

            // Then
            assertAll("注册结果验证",
                () -> assertNotNull(response),
                () -> assertEquals("alice", response.getUsername()),
                () -> assertEquals("alice@example.com", response.getEmail())
            );

            // 验证用户被正确保存
            verify(userRepository).save(userCaptor.capture());
            User savedUser = userCaptor.getValue();
            assertEquals("$2a$10$encoded", savedUser.getPassword());
            assertEquals(UserStatus.PENDING_VERIFICATION, savedUser.getStatus());

            // 验证验证邮件被发送
            verify(emailService).sendVerificationEmail("alice@example.com", 1L);
        }
    }

    @Nested
    @DisplayName("注册失败场景")
    class RegisterFailure {

        @Test
        @DisplayName("邮箱已存在时应抛出异常且不保存用户")
        void should_throw_when_email_exists() {
            // Given
            RegisterRequest request = new RegisterRequest(
                "alice2", "alice@example.com", "Password123!"
            );
            when(userRepository.existsByEmail("alice@example.com")).thenReturn(true);

            // When & Then
            assertThrows(EmailAlreadyExistsException.class,
                () -> service.register(request));

            // 验证不会保存用户，也不会发邮件
            verify(userRepository, never()).save(any());
            verify(emailService, never()).sendVerificationEmail(any(), any());
        }

        @ParameterizedTest(name = "无效邮箱: {0}")
        @ValueSource(strings = {"", "not-an-email", "@", "a@"})
        @DisplayName("无效邮箱格式应抛出异常")
        void should_throw_on_invalid_email(String invalidEmail) {
            RegisterRequest request = new RegisterRequest(
                "alice", invalidEmail, "Password123!"
            );

            assertThrows(InvalidEmailException.class,
                () -> service.register(request));
        }
    }
}
```

### 案例 2：订单处理服务测试

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("订单处理服务测试")
class OrderServiceTest {

    @Mock OrderRepository orderRepository;
    @Mock ProductService productService;
    @Mock PaymentService paymentService;
    @Mock NotificationService notificationService;
    @InjectMocks OrderService orderService;

    @Test
    @DisplayName("正常下单流程")
    void should_process_order_successfully() {
        // Given
        Long userId = 1L;
        CreateOrderRequest request = new CreateOrderRequest(
            userId,
            List.of(new OrderItem("PROD-001", 2))
        );

        Product product = new Product("PROD-001", "iPhone 15", 6999.0, 10);
        given(productService.getProduct("PROD-001")).willReturn(product);
        given(productService.reserveStock("PROD-001", 2)).willReturn(true);
        given(paymentService.createPayment(eq(userId), anyDouble()))
            .willReturn(new Payment("PAY-123", PaymentStatus.PENDING));
        given(orderRepository.save(any())).thenAnswer(inv -> {
            Order o = inv.getArgument(0);
            o.setId("ORD-" + System.currentTimeMillis());
            return o;
        });

        // When
        OrderResponse response = orderService.createOrder(request);

        // Then
        assertThat(response.getStatus()).isEqualTo(OrderStatus.PENDING_PAYMENT);
        assertThat(response.getTotalAmount()).isEqualTo(13998.0); // 6999 * 2

        InOrder inOrder = inOrder(productService, paymentService, orderRepository, notificationService);
        inOrder.verify(productService).getProduct("PROD-001");
        inOrder.verify(productService).reserveStock("PROD-001", 2);
        inOrder.verify(paymentService).createPayment(userId, 13998.0);
        inOrder.verify(orderRepository).save(any());
        inOrder.verify(notificationService).notifyOrderCreated(anyString());
    }

    @Test
    @DisplayName("库存不足时应回滚并抛出异常")
    void should_rollback_when_insufficient_stock() {
        // Given
        CreateOrderRequest request = new CreateOrderRequest(
            1L, List.of(new OrderItem("PROD-001", 100))
        );

        given(productService.getProduct("PROD-001"))
            .willReturn(new Product("PROD-001", "iPhone 15", 6999.0, 5)); // 只有5个
        given(productService.reserveStock("PROD-001", 100)).willReturn(false);

        // When & Then
        assertThatThrownBy(() -> orderService.createOrder(request))
            .isInstanceOf(InsufficientStockException.class)
            .hasMessageContaining("PROD-001");

        verify(paymentService, never()).createPayment(any(), anyDouble());
        verify(orderRepository, never()).save(any());
    }
}
```

### 案例 3：Controller 层测试

```java
@WebMvcTest(ProductController.class)
@DisplayName("商品 Controller 测试")
class ProductControllerTest {

    @Autowired MockMvc mockMvc;
    @MockBean ProductService productService;
    @Autowired ObjectMapper objectMapper;

    @Test
    @DisplayName("GET /api/products/{id} 返回商品详情")
    void get_product_by_id() throws Exception {
        ProductDTO product = new ProductDTO("PROD-001", "iPhone 15", 6999.0, 10);
        when(productService.findById("PROD-001")).thenReturn(product);

        mockMvc.perform(get("/api/products/PROD-001")
                .accept(MediaType.APPLICATION_JSON))
            .andDo(print())
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.id").value("PROD-001"))
            .andExpect(jsonPath("$.name").value("iPhone 15"))
            .andExpect(jsonPath("$.price").value(6999.0))
            .andExpect(jsonPath("$.stock").value(10));
    }

    @Test
    @DisplayName("GET /api/products/{id} 商品不存在返回 404")
    void get_product_returns_404_when_not_found() throws Exception {
        when(productService.findById("NOTEXIST"))
            .thenThrow(new ProductNotFoundException("NOTEXIST"));

        mockMvc.perform(get("/api/products/NOTEXIST"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.error").value("PRODUCT_NOT_FOUND"));
    }

    @Test
    @DisplayName("POST /api/products 创建商品")
    void create_product() throws Exception {
        CreateProductRequest req = new CreateProductRequest("iPhone 15", 6999.0, 10);
        ProductDTO created = new ProductDTO("PROD-NEW", "iPhone 15", 6999.0, 10);
        when(productService.create(any())).thenReturn(created);

        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(req)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").value("PROD-NEW"));
    }

    @Test
    @DisplayName("POST /api/products 请求体验证失败")
    void create_product_validation_fail() throws Exception {
        CreateProductRequest invalidReq = new CreateProductRequest("", -1.0, -5);

        mockMvc.perform(post("/api/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalidReq)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.violations").isArray())
            .andExpect(jsonPath("$.violations.length()").value(3));
    }
}
```

### 案例 4：Repository 层测试

```java
@DataJpaTest
@DisplayName("用户 Repository 测试")
class UserRepositoryTest {

    @Autowired TestEntityManager em;
    @Autowired UserRepository userRepository;

    @BeforeEach
    void setUp() {
        // 准备测试数据
        em.persist(new User("alice", "alice@example.com", UserStatus.ACTIVE));
        em.persist(new User("bob", "bob@example.com", UserStatus.INACTIVE));
        em.persist(new User("charlie", "charlie@example.com", UserStatus.ACTIVE));
        em.flush();
        em.clear();
    }

    @Test
    void findByEmail_should_return_user() {
        Optional<User> result = userRepository.findByEmail("alice@example.com");
        assertThat(result).isPresent().hasValueSatisfying(u ->
            assertThat(u.getUsername()).isEqualTo("alice")
        );
    }

    @Test
    void findByStatus_should_return_active_users() {
        List<User> activeUsers = userRepository.findByStatus(UserStatus.ACTIVE);
        assertThat(activeUsers).hasSize(2)
            .extracting(User::getUsername)
            .containsExactlyInAnyOrder("alice", "charlie");
    }

    @Test
    void countByStatus_should_return_correct_count() {
        assertEquals(2, userRepository.countByStatus(UserStatus.ACTIVE));
        assertEquals(1, userRepository.countByStatus(UserStatus.INACTIVE));
    }

    @Test
    void existsByEmail_should_return_true_when_exists() {
        assertTrue(userRepository.existsByEmail("alice@example.com"));
        assertFalse(userRepository.existsByEmail("notexist@example.com"));
    }
}
```

### 案例 5：缓存服务测试

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("缓存服务测试")
class CacheableUserServiceTest {

    @Mock UserRepository userRepository;
    @InjectMocks CacheableUserService service;

    @Test
    @DisplayName("第一次查询走数据库，第二次走缓存")
    void should_cache_user_on_second_call() {
        User mockUser = new User(1L, "Alice");
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));

        // 第一次调用
        User first = service.findUser(1L);
        // 第二次调用（应从缓存返回）
        User second = service.findUser(1L);

        assertEquals(first, second);
        // 数据库只被查询了一次
        verify(userRepository, times(1)).findById(1L);
    }

    @Test
    @DisplayName("更新用户后缓存应被清除")
    void should_evict_cache_on_update() {
        User mockUser = new User(1L, "Alice");
        when(userRepository.findById(1L)).thenReturn(Optional.of(mockUser));
        when(userRepository.save(any())).thenReturn(mockUser);

        service.findUser(1L);            // 写入缓存
        service.updateUser(1L, "Alice2"); // 清除缓存
        service.findUser(1L);            // 再次从数据库查询

        // 数据库被查询了两次（更新前1次，更新后1次）
        verify(userRepository, times(2)).findById(1L);
    }
}
```

---

## Part 11: 测试最佳实践 {#part-11}

### 11.1 测试命名规范

好的测试名称应该描述：**场景（what）+ 条件（when）+ 期望（then）**

```java
// 差的命名（不知道测什么）
@Test
void testUser() { }

@Test
void test1() { }

// 好的命名（清晰描述场景）
@Test
void should_return_empty_when_user_not_found() { }

@Test
void given_valid_email_when_register_then_send_verification_email() { }

// 中文命名也很好
@Test
@DisplayName("给定有效邮箱，注册时应发送验证邮件")
void register_with_valid_email() { }
```

### 11.2 测试结构：AAA 模式

每个测试应清晰分为三段：

```java
@Test
void should_register_user_successfully() {
    // Arrange（准备/Given）
    RegisterRequest request = new RegisterRequest("alice", "alice@example.com");
    when(userRepository.existsByEmail(anyString())).thenReturn(false);
    when(userRepository.save(any())).thenAnswer(inv -> {
        User u = inv.getArgument(0);
        u.setId(1L);
        return u;
    });

    // Act（执行/When）
    UserResponse response = service.register(request);

    // Assert（断言/Then）
    assertThat(response.getId()).isEqualTo(1L);
    assertThat(response.getEmail()).isEqualTo("alice@example.com");
    verify(emailService).sendVerificationEmail(anyString(), eq(1L));
}
```

### 11.3 测试隔离

```java
// 反例：测试间共享状态（危险！）
class BadTest {
    static List<User> users = new ArrayList<>();

    @Test
    void test1() {
        users.add(new User("Alice"));
        assertEquals(1, users.size()); // 可能因为执行顺序不同而失败
    }

    @Test
    void test2() {
        assertEquals(0, users.size()); // 可能失败：test1 先运行导致有1个用户
    }
}

// 正例：每次测试前重置状态
class GoodTest {
    List<User> users;

    @BeforeEach
    void setUp() {
        users = new ArrayList<>();  // 每次都重新创建
    }

    @Test
    void test1() {
        users.add(new User("Alice"));
        assertEquals(1, users.size()); // 总是通过
    }

    @Test
    void test2() {
        assertEquals(0, users.size()); // 总是通过
    }
}
```

### 11.4 避免过度 Mock

```java
// 反例：Mock 了太多，测试失去意义
@Test
void bad_excessive_mock() {
    when(validator.validate(any())).thenReturn(true);
    when(converter.convert(any())).thenReturn(new UserDTO());
    when(repository.save(any())).thenReturn(new User());
    when(mapper.toResponse(any())).thenReturn(new UserResponse());

    UserResponse result = service.register(request);

    // 这里只是测试了各个 Mock 的组合，没有测试真正的业务逻辑
    assertNotNull(result);
}

// 正例：只 Mock 外部依赖（数据库、邮件等），保留业务逻辑
@Test
void good_minimal_mock() {
    // 只 Mock 真正需要的外部依赖
    when(userRepository.existsByEmail(anyString())).thenReturn(false);
    when(userRepository.save(any())).thenAnswer(inv -> {
        User u = inv.getArgument(0);
        u.setId(1L);
        return u;
    });

    UserResponse result = service.register(request);

    // 真正在测试业务逻辑
    assertEquals("alice@example.com", result.getEmail());
    assertEquals(UserStatus.PENDING_VERIFICATION, result.getStatus());
}
```

### 11.5 测试数据构建器模式

```java
// 构建器类，简化测试数据创建
class UserTestBuilder {
    private Long id = 1L;
    private String username = "testuser";
    private String email = "test@example.com";
    private String password = "encoded_password";
    private UserStatus status = UserStatus.ACTIVE;

    public static UserTestBuilder aUser() {
        return new UserTestBuilder();
    }

    public UserTestBuilder withId(Long id) { this.id = id; return this; }
    public UserTestBuilder withUsername(String username) { this.username = username; return this; }
    public UserTestBuilder withEmail(String email) { this.email = email; return this; }
    public UserTestBuilder withStatus(UserStatus status) { this.status = status; return this; }

    public User build() {
        return User.builder()
            .id(id).username(username)
            .email(email).password(password)
            .status(status).build();
    }

    public UserResponse buildResponse() {
        return new UserResponse(id, username, email, status);
    }
}

// 使用方式
@Test
void test_with_builder() {
    User alice = UserTestBuilder.aUser()
        .withId(1L)
        .withEmail("alice@example.com")
        .withStatus(UserStatus.ACTIVE)
        .build();

    // 测试代码...
}
```

### 11.6 测试金字塔实践

```
项目测试策略建议：

单元测试 (70%)：
  - Service 层业务逻辑
  - 工具类方法
  - 领域对象行为
  - 运行速度：< 10ms

集成测试 (20%)：
  - Controller + Service（@WebMvcTest）
  - Service + Repository（@DataJpaTest）
  - 外部服务调用（TestContainers）
  - 运行速度：< 1s

E2E 测试 (10%)：
  - 核心业务流程（注册→登录→下单）
  - 运行速度：< 10s
```

---

## Part 12: 高频面试题 {#part-12}

### Q1: JUnit 5 和 JUnit 4 的主要区别？

**答**：
1. **模块化**：JUnit 5 分为 Platform/Jupiter/Vintage 三个模块
2. **注解变化**：`@Before`→`@BeforeEach`，`@After`→`@AfterEach`，`@BeforeClass`→`@BeforeAll`
3. **扩展机制**：JUnit 4 用 `@RunWith` + `@Rule`，JUnit 5 统一为 `@ExtendWith`
4. **Lambda 支持**：JUnit 5 断言支持 Lambda，如 `assertThrows`
5. **嵌套测试**：JUnit 5 新增 `@Nested`
6. **参数化测试**：JUnit 5 的 `@ParameterizedTest` 更强大
7. **Java 版本**：JUnit 4 支持 Java 5+，JUnit 5 需要 Java 8+

### Q2: Mock 和 Stub 的区别？

**答**：
- **Stub**（存根）：为测试预设方法的返回值，目的是"控制输入"，关注状态验证（assert 结果值）
- **Mock**（模拟）：关注"验证交互"，验证某个方法是否被调用、调用了几次、以什么参数调用

```java
// Stub 示例：控制 findById 返回值
when(userRepository.findById(1L)).thenReturn(Optional.of(user));

// Mock 验证：确认 save 被调用了
verify(userRepository, times(1)).save(any(User.class));
```

### Q3: @Mock 和 @Spy 的区别？

**答**：
- `@Mock`：创建完全虚假对象，所有方法默认返回空值/0/false，不调用真实方法
- `@Spy`：包装真实对象，默认调用真实方法，可以选择性地覆写某些方法

```java
@Mock List<String> mockList;      // mockList.size() 默认返回 0
@Spy  List<String> spyList = new ArrayList<>();  // spyList.size() 调用真实方法

@Test
void mock_vs_spy() {
    mockList.add("a"); // add() 什么都不做
    assertEquals(0, mockList.size()); // 默认返回 0

    spyList.add("a"); // 真实调用
    assertEquals(1, spyList.size()); // 真实的 1
}
```

### Q4: @MockBean 和 @Mock 的区别？

**答**：
- `@Mock`（Mockito）：创建独立的 Mock 对象，不放入 Spring 容器
- `@MockBean`（Spring Boot Test）：创建 Mock 并将其注册到 Spring ApplicationContext，替换同类型的 Bean

使用场景：
- 写 Service 单元测试时用 `@Mock`
- 写 `@WebMvcTest` / `@SpringBootTest` 集成测试时用 `@MockBean`

### Q5: 如何测试抛出异常的方法？

```java
// JUnit 5 assertThrows
@Test
void test_exception() {
    ArithmeticException ex = assertThrows(
        ArithmeticException.class,
        () -> calculator.divide(10, 0)
    );
    assertEquals("/ by zero", ex.getMessage());
}

// AssertJ 风格
@Test
void test_exception_assertj() {
    assertThatThrownBy(() -> calculator.divide(10, 0))
        .isInstanceOf(ArithmeticException.class)
        .hasMessage("/ by zero");
}
```

### Q6: 如何测试 void 方法？

```java
// void 方法无返回值，通过验证副作用来测试
@Test
void test_void_method() {
    // 方法1：验证是否调用了其他方法（副作用）
    service.sendWelcomeEmail("user@example.com");
    verify(emailClient).send(eq("user@example.com"), anyString());

    // 方法2：验证状态变化
    user.activate();
    assertEquals(UserStatus.ACTIVE, user.getStatus());

    // 方法3：验证是否抛出异常
    doThrow(new RuntimeException("DB Error")).when(repo).delete(any());
    assertThrows(RuntimeException.class, () -> service.deleteUser(1L));
}
```

### Q7: 参数化测试有哪些数据源？

| 注解 | 用途 |
|------|------|
| `@ValueSource` | 简单基本类型值数组 |
| `@NullSource` | 提供 null 值 |
| `@EmptySource` | 提供空字符串/集合 |
| `@EnumSource` | 枚举值 |
| `@CsvSource` | 内联 CSV 数据 |
| `@CsvFileSource` | 外部 CSV 文件 |
| `@MethodSource` | 静态方法提供 Stream<Arguments> |
| `@ArgumentsSource` | 自定义 ArgumentsProvider |

### Q8: 什么是软断言（Soft Assertions）？何时使用？

**答**：普通断言一旦失败就停止后续断言。软断言会收集所有失败，最后一起报告。

使用场景：验证一个对象的多个属性时，希望一次看到所有问题，而不是逐个修复逐个运行。

```java
@Test
void soft_assertion_example(SoftAssertions softly) {
    User user = getUser();
    softly.assertThat(user.getName()).isEqualTo("Alice");  // 可能失败
    softly.assertThat(user.getAge()).isEqualTo(30);        // 继续执行
    softly.assertThat(user.getEmail()).contains("@");      // 继续执行
    // 三个断言的结果都会被报告
}
```

### Q9: @BeforeAll 为什么必须是 static？

**答**：JUnit 5 默认为每个测试方法创建一个新的测试类实例（以保证测试隔离）。`@BeforeAll` 在测试类实例化之前执行，此时没有实例，所以必须是 static。

如果想让 `@BeforeAll` 非 static，可以修改测试类实例化策略：

```java
@TestInstance(TestInstance.Lifecycle.PER_CLASS)
class PerClassLifecycleTest {

    @BeforeAll
    void init() {  // 不需要 static 了
        // 整个测试类只初始化一次
    }
}
```

### Q10: 如何测试私有方法？

**答**：通常不直接测试私有方法，而是通过公共方法间接测试。私有方法的测试由调用它的公共方法覆盖。

如果确实需要：
1. 将私有方法改为包级别（去掉 private）
2. 使用反射（不推荐）
3. 考虑是否应该将该逻辑提取到另一个类

### Q11: TestContainers 相比 H2 内存数据库的优势？

| 对比点 | H2 内存数据库 | TestContainers |
|--------|--------------|----------------|
| 数据库方言 | H2 的 SQL 方言 | 真实数据库方言 |
| 特有特性 | 不支持 MySQL 特有函数 | 完全支持 |
| 性能 | 极快 | 较慢（需启动 Docker）|
| 生产一致性 | 差 | 完全一致 |
| 依赖 | 只需 JAR | 需要 Docker |

### Q12: 如何处理测试中的时间依赖？

```java
// 反例：直接使用 LocalDate.now()（不可重复）
public boolean isExpired() {
    return expiryDate.isBefore(LocalDate.now()); // 难以测试！
}

// 正例：注入 Clock
@Service
public class SubscriptionService {
    private final Clock clock;

    public boolean isExpired(Subscription sub) {
        return sub.getExpiryDate().isBefore(LocalDate.now(clock));
    }
}

// 测试时注入固定时钟
@Test
void should_detect_expired_subscription() {
    Clock fixedClock = Clock.fixed(
        Instant.parse("2024-01-15T00:00:00Z"), ZoneId.of("UTC")
    );
    SubscriptionService service = new SubscriptionService(fixedClock);

    Subscription expired = new Subscription(LocalDate.of(2024, 1, 1));
    assertTrue(service.isExpired(expired));
}
```

### Q13: 测试覆盖率达到多少才算合格？

**答**：没有绝对标准，通常的行业参考：
- **行覆盖率**：≥ 80% 是常见目标
- **分支覆盖率**：≥ 70%
- **核心业务逻辑**：应尽量 100%

注意：高覆盖率≠高质量测试。覆盖率只是衡量代码被执行过，不代表断言有意义。

### Q14: 如何提高测试运行速度？

1. **单元测试不要启动 Spring 容器**，用 `@ExtendWith(MockitoExtension.class)` 而非 `@SpringBootTest`
2. **TestContainers 使用共享容器**，`@Container` + `static` 关键字
3. **@DataJpaTest / @WebMvcTest** 只加载需要的层，比 `@SpringBootTest` 快
4. **并行执行测试**：

```properties
# junit-platform.properties
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
```

### Q15: @InjectMocks 的注入原理？

**答**：Mockito 的 `@InjectMocks` 按以下顺序尝试注入：
1. **构造函数注入**：找到参数类型最多的构造函数，注入对应的 Mock
2. **Setter 方法注入**：调用 setter 方法注入 Mock
3. **字段注入**：直接设置字段值

推荐使用构造函数注入（符合 Spring 最佳实践，也便于测试）：

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    // 构造函数注入，@InjectMocks 会自动识别
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

---

*文档完*


---

## 附录 A：常用断言速查表

### A.1 JUnit 5 Assertions 速查

```java
// 相等
assertEquals(expected, actual)
assertEquals(expected, actual, "失败消息")
assertEquals(expected, actual, () -> "Lazy 失败消息")  // 延迟计算消息

// 不等
assertNotEquals(unexpected, actual)

// Null
assertNull(object)
assertNotNull(object)

// 布尔
assertTrue(condition)
assertFalse(condition)

// 同一对象
assertSame(expected, actual)
assertNotSame(unexpected, actual)

// 数组
assertArrayEquals(expected, actual)
assertArrayEquals(expected, actual, delta)  // 浮点数容差

// 异常
assertThrows(ExceptionType.class, executable)
assertDoesNotThrow(executable)

// 超时
assertTimeout(duration, executable)
assertTimeoutPreemptively(duration, executable)

// 组合
assertAll(executables...)
assertAll(heading, executables...)
```

### A.2 AssertJ 字符串断言速查

```java
assertThat(str)
    .isEqualTo("expected")
    .isEqualToIgnoringCase("EXPECTED")
    .isEqualToIgnoringWhitespace("  expected  ")
    .startsWith("exp")
    .endsWith("ted")
    .contains("xpect")
    .containsIgnoringCase("XPECT")
    .doesNotContain("xxx")
    .matches("exp.*")
    .matchesPattern(Pattern.compile("exp.*"))
    .hasSize(8)
    .hasSizeBetween(5, 10)
    .isEmpty()
    .isNotEmpty()
    .isBlank()
    .isNotBlank()
    .isNull()
    .isNotNull()
```

### A.3 AssertJ 集合断言速查

```java
assertThat(list)
    .isEmpty()
    .isNotEmpty()
    .hasSize(3)
    .hasSizeGreaterThan(2)
    .hasSizeLessThan(10)
    .hasSizeBetween(1, 5)
    .contains("a", "b")                    // 包含（顺序无关）
    .containsOnly("a", "b", "c")           // 只包含这些（顺序无关）
    .containsExactly("a", "b", "c")        // 严格顺序
    .containsExactlyInAnyOrder("c", "a", "b")
    .containsAnyOf("a", "x", "y")
    .doesNotContain("x", "y")
    .startsWith("a")
    .endsWith("c")
    .allMatch(s -> s.length() > 0)
    .anyMatch(s -> s.startsWith("a"))
    .noneMatch(s -> s.isEmpty())
    .extracting(User::getName).contains("Alice", "Bob")
    .filteredOn(u -> u.getAge() > 18).hasSize(2)
```

---

## 附录 B：Mockito 常用方法速查

### B.1 打桩方法

```java
// 返回值
when(mock.method(args)).thenReturn(value)
when(mock.method(args)).thenReturn(v1, v2, v3)  // 多次调用依次返回
when(mock.method(args)).thenAnswer(invocation -> { ... })
when(mock.method(args)).thenCallRealMethod()     // 调用真实方法

// 抛异常
when(mock.method(args)).thenThrow(new Exception())
when(mock.method(args)).thenThrow(Exception.class)

// void 方法
doNothing().when(mock).voidMethod(args)
doThrow(ex).when(mock).voidMethod(args)
doReturn(value).when(mock).method(args)  // 等同于 when().thenReturn()，但对 Spy 安全
doAnswer(inv -> { ... }).when(mock).method(args)
doCallRealMethod().when(mock).method(args)
```

### B.2 参数匹配器

```java
// 通配符
any()              // 任何对象（含 null）
any(Type.class)    // 指定类型
anyString()
anyInt() / anyLong() / anyDouble() / anyBoolean()
anyList() / anyMap() / anySet() / anyCollection()

// 精确匹配
eq(value)

// 字符串
startsWith("prefix")
endsWith("suffix")
contains("substring")
matches("regex.*")

// 自定义
argThat(predicate)
intThat(predicate)
```

### B.3 验证方法

```java
verify(mock).method(args)                // 默认验证调用1次
verify(mock, times(n)).method(args)      // 精确n次
verify(mock, atLeast(n)).method(args)    // 至少n次
verify(mock, atMost(n)).method(args)     // 至多n次
verify(mock, never()).method(args)        // 从不调用
verify(mock, only()).method(args)         // 只调用这一个方法

verifyNoMoreInteractions(mock1, mock2)    // 没有未验证的调用
verifyNoInteractions(mock1, mock2)        // 完全没有交互

InOrder inOrder = inOrder(mock1, mock2);
inOrder.verify(mock1).method1();
inOrder.verify(mock2).method2();
```

---

## 附录 C：测试配置文件参考

### C.1 junit-platform.properties

```properties
# 启用并行测试
junit.jupiter.execution.parallel.enabled=true
junit.jupiter.execution.parallel.mode.default=concurrent
junit.jupiter.execution.parallel.mode.classes.default=concurrent

# 并发线程数
junit.jupiter.execution.parallel.config.strategy=fixed
junit.jupiter.execution.parallel.config.fixed.parallelism=4

# 测试方法排序
junit.jupiter.testmethod.order.default=org.junit.jupiter.api.MethodOrderer$Random
```

### C.2 application-test.yml（测试配置）

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  mail:
    host: localhost
    port: 25

logging:
  level:
    org.springframework.test: DEBUG
    com.example: DEBUG
```

---

*JUnit5 + Mockito 测试详解 — 完*
