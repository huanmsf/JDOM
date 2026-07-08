# Java 面试八股文汇总 - 高频考点速查手册

> 版本：2026年版 | 覆盖 JDK 17 / Spring Boot 3.x / MySQL 8.x / Redis 7.x
>
> 使用说明：每个问题包含核心答案、要点列表，以及必要的代码示例或 ASCII 图示。

---

## 目录

- [第一章：Java 基础](#第一章java-基础)
- [第二章：Java 集合](#第二章java-集合)
- [第三章：Java 并发（JUC）](#第三章java-并发juc)
- [第四章：JVM](#第四章jvm)
- [第五章：MySQL](#第五章mysql)
- [第六章：Redis](#第六章redis)
- [第七章：Spring 全家桶](#第七章spring-全家桶)
- [第八章：MyBatis](#第八章mybatis)
- [第九章：消息队列](#第九章消息队列)
- [第十章：分布式](#第十章分布式)
- [第十一章：网络协议](#第十一章网络协议)
- [第十二章：系统设计高频题](#第十二章系统设计高频题)

---

# 第一章：Java 基础

## 1.1 Java 基本类型与包装类（装箱拆箱/缓存池）

**核心答案：**
Java 有 8 种基本类型（byte/short/int/long/float/double/char/boolean），每种都有对应包装类。装箱（Boxing）是基本类型转包装类，拆箱（Unboxing）是包装类转基本类型，编译器自动完成。Integer 等缓存池默认缓存 -128~127 的对象实例。

**要点：**
- `Integer.valueOf(100) == Integer.valueOf(100)` 返回 `true`（命中缓存池）
- `Integer.valueOf(200) == Integer.valueOf(200)` 返回 `false`（超出缓存范围，堆上新建）
- `new Integer(100) == new Integer(100)` 返回 `false`（强制新建对象）
- 缓存池上界可通过 `-XX:AutoBoxCacheMax=<size>` 调整（仅 Integer）
- 拆箱时如包装类为 `null`，会抛出 `NullPointerException`
- 频繁装箱拆箱（如循环内）会产生大量临时对象，影响 GC

**缓存池范围：**

| 类型 | 缓存范围 |
|------|---------|
| Byte | -128 ~ 127（全部） |
| Short | -128 ~ 127 |
| Integer | -128 ~ 127（可配置上限） |
| Long | -128 ~ 127 |
| Character | 0 ~ 127 |
| Boolean | true / false（两个常量） |
| Float/Double | 无缓存 |

**代码示例：**
```java
// 装箱底层调用 Integer.valueOf()
Integer a = 100;  // 等价于 Integer.valueOf(100)
int b = a;        // 等价于 a.intValue()

// 陷阱：混合比较触发拆箱，比较数值
Integer x = 200;
Integer y = 200;
System.out.println(x == y);       // false（引用不同）
System.out.println(x.equals(y));  // true（值相同）
System.out.println(x == 200);     // true（拆箱后比较 int）
```

---
## 1.2 String / StringBuilder / StringBuffer 区别

**核心答案：**
`String` 不可变（`final char[]`），线程安全，适合少量字符串操作。`StringBuilder` 可变、非线程安全，适合单线程大量拼接。`StringBuffer` 可变、线程安全（方法加 `synchronized`），性能略低于 `StringBuilder`。

**要点：**
- `String` 不可变性依赖：`private final char[] value`（JDK9+ 改为 `byte[]`+编码标识）
- `String` 常量池（String Pool）存储字面量，`intern()` 可手动入池
- `"a" + "b"` 编译期优化为 `"ab"`，多变量拼接编译为 `StringBuilder`
- `StringBuffer` 所有 public 方法均为 `synchronized`，开销大
- 循环内字符串拼接必须使用 `StringBuilder`，否则每次 `+` 都创建新对象

**代码示例：**
```java
// 错误写法（循环内用 +）
String s = "";
for (int i = 0; i < 100000; i++) {
    s += i;  // 每次创建新 String 对象！
}

// 正确写法
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 100000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

---

## 1.3 == 与 equals() 区别

**核心答案：**
`==` 比较的是**引用地址**（基本类型比较值），而 `equals()` 默认比较引用地址（继承自 Object），但大多数类（String/Integer等）重写了 `equals()` 比较**对象内容**。

**要点：**
- 基本类型无 `equals()` 方法，只能用 `==`
- Object 默认 `equals()` 实现即为 `==`
- `String.equals()` 逐字符比较内容
- 重写 `equals()` 必须同时重写 `hashCode()`（契约要求）
- `Objects.equals(a, b)` 可安全处理 null（推荐使用）

**源码对比：**
```java
// Object 默认实现
public boolean equals(Object obj) {
    return (this == obj);  // 就是比较引用
}
// String 重写实现（简化版）
public boolean equals(Object anObject) {
    if (this == anObject) return true;
    if (anObject instanceof String aString) {
        return StringLatin1.equals(value, aString.value);
    }
    return false;
}
```

---

## 1.4 hashCode() 与 equals() 契约

**核心答案：**
Java 规范要求：若 `a.equals(b)` 为 `true`，则 `a.hashCode() == b.hashCode()` 必须成立。反之不要求（哈希碰撞允许）。违反契约会导致 HashMap/HashSet 行为异常。

**要点：**
- 契约一：相等对象必须有相同 hashCode
- 契约二：不相等对象的 hashCode 尽量不同（减少碰撞）
- 只重写 `equals()` 不重写 `hashCode()`：同值对象存入 `HashMap` 会出现两个 key
- JDK 提供 `Objects.hash(field1, field2, ...)` 辅助生成
- IDEA/Lombok 可自动生成正确的 `equals()` + `hashCode()`

**反例说明：**
```java
class Point {
    int x, y;
    @Override
    public boolean equals(Object o) {
        Point p = (Point) o;
        return x == p.x && y == p.y;
    }
    // 继承 Object.hashCode()，基于内存地址（违反契约！）
}

Set<Point> set = new HashSet<>();
set.add(new Point(1, 2));
set.contains(new Point(1, 2));  // 可能返回 false！因为 hashCode 不同
```

---

## 1.5 final / finally / finalize 区别

**核心答案：**
三个关键字完全不同：`final` 是修饰符（不可变/不可继承/不可重写），`finally` 是异常处理块（必定执行），`finalize()` 是 Object 方法（GC 前回调，已废弃）。

**要点：**

| 关键字 | 用途 | 注意事项 |
|--------|------|---------|
| `final` | 修饰变量（不可重新赋值）、方法（不可重写）、类（不可继承） | final 引用不可变但指向对象可变 |
| `finally` | try-catch-finally 中必定执行的代码块 | `System.exit()` 或 JVM 崩溃时不执行 |
| `finalize()` | GC 前调用，用于资源清理 | JDK9 deprecated，不可预期，不推荐使用 |

**finally 执行顺序陷阱：**
```java
public int test() {
    try {
        return 1;
    } finally {
        return 2;  // finally 中 return 会覆盖 try 中的 return，返回 2！
    }
}
```

---

## 1.6 接口 vs 抽象类（Java 8 default 方法后的区别）

**核心答案：**
Java 8 后接口可有 `default` 和 `static` 方法，Java 9 后可有 `private` 方法，差距缩小。核心区别仍在于：接口表示**能力（has-a）**，抽象类表示**类型（is-a）**。类只能单继承抽象类，但可多实现接口。

**要点对比：**

| 特性 | 接口（Interface） | 抽象类（Abstract Class） |
|------|-----------------|----------------------|
| 继承 | 可多实现 | 只能单继承 |
| 构造器 | 无 | 有 |
| 成员变量 | 只能 public static final | 无限制 |
| 方法 | 默认 public abstract；Java8 可 default/static | 无限制 |
| 设计语义 | 能力/契约（Comparable/Runnable） | 类型/模板（AbstractList） |
| 状态 | 无实例状态 | 可有实例状态 |

**Java 8 default 方法冲突解决：**
```java
interface A { default void hello() { System.out.println("A"); } }
interface B { default void hello() { System.out.println("B"); } }
class C implements A, B {
    @Override
    public void hello() { A.super.hello(); }  // 必须手动解决冲突
}
```

---

## 1.7 重载（Overload）vs 重写（Override）

**核心答案：**
**重载**是同类中方法名相同、参数列表不同（编译时多态，静态分派）。**重写**是子类覆盖父类方法（运行时多态，动态分派），方法签名相同。

| 特性 | 重载 | 重写 |
|------|------|------|
| 发生位置 | 同一个类 | 子类与父类 |
| 方法名 | 相同 | 相同 |
| 参数列表 | 不同 | 相同 |
| 返回类型 | 可不同 | 必须相同或协变（子类型） |
| 访问权限 | 无限制 | 不能比父类更严格 |
| 异常 | 无限制 | 不能抛出更宽泛的 checked 异常 |
| 多态类型 | 编译时（静态）多态 | 运行时（动态）多态 |

**重写规则（OAET）：**
- **O**verride：方法名+参数完全一致
- **A**ccess：访问权限只能放宽（public > protected > default > private）
- **E**xception：只能抛出更窄或相同的 checked 异常
- **T**ype：返回类型相同或为子类型（协变返回类型）

---

## 1.8 深拷贝 vs 浅拷贝

**核心答案：**
**浅拷贝**只复制对象本身及基本类型字段，引用类型字段仍指向同一对象。**深拷贝**递归复制所有引用对象，完全独立。

**图示：**
```
浅拷贝：
原对象   --> [name, address_ref] --> Address对象（共享！）
拷贝对象 --> [name, address_ref] --> 同一个Address对象

深拷贝：
原对象   --> [name, address_ref1] --> Address对象A
拷贝对象 --> [name, address_ref2] --> Address对象B（新建副本！）
```

**实现方式：**
```java
// 1. Cloneable（浅拷贝）
class Person implements Cloneable {
    String name; Address address;
    @Override protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}
// 2. 深拷贝：手动递归
Person p = (Person) super.clone();
p.address = (Address) address.clone();  // 递归复制引用类型

// 3. 序列化深拷贝（简单但性能差）
// ByteArrayOutputStream bos; ObjectOutputStream oos ...
// 4. 第三方库：SerializationUtils.clone(obj)
```

---

## 1.9 Java 异常体系（Checked/Unchecked/Error）

**核心答案：**
Java 异常层次：`Throwable` 分为 `Error`（系统级错误，不应捕获）和 `Exception`（`RuntimeException` 为非受检，其余为受检异常）。

**异常层次结构：**
```
Throwable
|-- Error（不应捕获）
|   |-- OutOfMemoryError
|   |-- StackOverflowError
|   `-- VirtualMachineError
`-- Exception
    |-- RuntimeException（Unchecked，不强制处理）
    |   |-- NullPointerException
    |   |-- ArrayIndexOutOfBoundsException
    |   |-- ClassCastException
    |   `-- IllegalArgumentException
    `-- 受检异常 Checked（必须 try-catch 或 throws）
        |-- IOException
        |-- SQLException
        `-- ClassNotFoundException
```

**要点：**
- **Checked Exception**：编译器强制处理，代表可预期的外部失败（IO/网络/SQL）
- **Unchecked Exception（RuntimeException）**：代表程序逻辑错误，不强制捕获
- **Error**：JVM 级别错误，通常无法恢复（OOM/StackOverflow）
- `finally` 中不要写 `return`，会吞掉异常
- 自定义业务异常继承 `RuntimeException`（框架主流做法）

---

## 1.10 泛型擦除原理

**核心答案：**
Java 泛型是**编译期特性**，运行时类型信息被擦除（Type Erasure）。编译后泛型类型参数替换为 `Object` 或边界类型，编译器在读取时自动插入类型转换。

**要点：**
- `List<String>` 和 `List<Integer>` 编译后均为 `List`（原始类型）
- 无法创建泛型数组：`new T[]` 编译错误
- 无法做 `instanceof T` 检查
- 泛型方法的桥接方法（bridge method）由编译器自动生成，保证多态正确
- 通过**超类型令牌**（`new TypeReference<List<String>>(){}`）可保留泛型信息

**擦除过程示例：**
```java
// 编译前
public <T extends Comparable<T>> T max(T a, T b) {
    return a.compareTo(b) > 0 ? a : b;
}
// 编译后（擦除 T 为 Comparable）
public Comparable max(Comparable a, Comparable b) {
    return a.compareTo(b) > 0 ? a : b;
}
```

---

## 1.11 反射原理与性能损耗

**核心答案：**
反射通过 JVM 在运行时动态获取类信息、调用方法、访问字段。底层依赖 `Class` 对象（类加载时创建）。反射调用性能损耗主要来自：安全检查、方法查找、参数装箱拆箱，约比直接调用慢 10~100 倍。

**要点：**
- `Class.forName()` 触发类加载，`getClass()` 获取已加载类
- `setAccessible(true)` 跳过访问权限检查（显著提升性能）
- JDK7+ 引入 `MethodHandles`（`java.lang.invoke`），性能接近直接调用
- 反射方法调用次数超过阈值（默认15次）后 JVM 会生成专用字节码（膨胀机制）
- Spring、Hibernate 等框架大量使用反射，通常配合缓存减少开销

**性能优化：**
```java
// 快：缓存 Method 对象 + setAccessible
Method m = clazz.getMethod("getName");
m.setAccessible(true);  // 跳过安全检查
for (int i = 0; i < 10000; i++) { m.invoke(obj); }
```

---

## 1.12 注解处理器（APT）原理

**核心答案：**
APT（Annotation Processing Tool）在**编译期**扫描注解并生成新代码/资源文件，不修改原有源码。通过实现 `AbstractProcessor`，注册到 `META-INF/services/javax.annotation.processing.Processor` 触发。

**要点：**
- 典型应用：Lombok（生成 getter/setter）、MapStruct（生成映射代码）、Dagger2（依赖注入）
- 处理阶段：在 `javac` 编译的 `process` 阶段执行，可多轮处理
- `RoundEnvironment` 提供当前轮次的注解元素集合
- 生成文件用 `Filer` API，写到 `generated-sources` 目录
- **运行时注解**（`@Retention(RUNTIME)`）配合反射使用；**编译时注解**（`SOURCE/CLASS`）配合 APT

---

## 1.13 序列化（Serializable/Externalizable/serialVersionUID）

**核心答案：**
序列化将对象转为字节流（持久化/网络传输），反序列化还原对象。`Serializable` 是标记接口，JVM 自动处理。`Externalizable` 需手动实现 `writeExternal/readExternal`，性能更好。`serialVersionUID` 用于版本兼容校验。

**要点：**
- 不序列化字段用 `transient` 修饰（如密码字段）
- `static` 字段不参与序列化（属于类，不属于对象）
- 未定义 `serialVersionUID` 时 JVM 自动计算，类结构变化后值改变，导致反序列化失败
- Java 序列化存在安全漏洞（反序列化攻击），生产推荐使用 JSON/Protobuf/Kryo

**最佳实践：**
```java
public class User implements Serializable {
    private static final long serialVersionUID = 1L;  // 显式声明
    private String name;
    private transient String password;  // 不序列化
    private static int count;           // static 不序列化
}
```

---

## 1.14 SPI 机制原理

**核心答案：**
SPI（Service Provider Interface）是 Java 内置的服务发现机制。服务定义接口，服务提供者在 `META-INF/services/<接口全限名>` 文件中声明实现类，通过 `ServiceLoader` 加载。实现**接口与实现解耦**。

**要点：**
- JDK 内置应用：JDBC 驱动（`java.sql.Driver`）、日志框架（`LoggerFinder`）
- Spring Boot 自动配置也基于类似 SPI（`spring.factories` / `AutoConfiguration.imports`）
- `ServiceLoader` 使用线程上下文类加载器（Thread Context ClassLoader），可突破双亲委派
- 缺点：无法按需加载特定实现，每次需遍历所有实现

**SPI 使用流程：**
```
1. 定义接口：public interface DataBaseDriver { void connect(); }
2. 创建实现类：public class MySQLDriver implements DataBaseDriver { ... }
3. 注册（文件名为接口全限定名）：
   META-INF/services/com.example.DataBaseDriver
   文件内容：com.example.MySQLDriver
4. 加载使用：
   ServiceLoader<DataBaseDriver> loaders = ServiceLoader.load(DataBaseDriver.class);
   for (DataBaseDriver driver : loaders) { driver.connect(); }
```

---

# 第二章：Java 集合

## 2.1 ArrayList vs LinkedList vs Vector

**核心答案：**
`ArrayList` 基于动态数组，随机访问 O(1)，增删尾部 O(1)，中间增删 O(n)。`LinkedList` 基于双向链表，随机访问 O(n)，头尾增删 O(1)。`Vector` 是线程安全的 ArrayList（方法加 `synchronized`），已不推荐使用。

| 特性 | ArrayList | LinkedList | Vector |
|------|-----------|------------|--------|
| 底层结构 | 动态数组 | 双向链表 | 动态数组 |
| 随机访问 | O(1) | O(n) | O(1) |
| 头部插入/删除 | O(n) | O(1) | O(n) |
| 尾部插入 | 均摊 O(1) | O(1) | 均摊 O(1) |
| 内存占用 | 紧凑 | 每节点额外 prev/next | 紧凑 |
| 线程安全 | 否 | 否 | 是（synchronized） |
| 扩容 | 1.5 倍 | 无需扩容 | 2 倍 |

**要点：**
- `ArrayList` 初始容量 10，扩容为 `oldCapacity + (oldCapacity >> 1)`（即 1.5 倍）
- `LinkedList` 同时实现了 `Deque` 接口，可作双端队列使用
- 遍历性能：ArrayList 远快于 LinkedList（缓存友好性/CPU 预取）
- 线程安全替代：`Collections.synchronizedList()` 或 `CopyOnWriteArrayList`

---

## 2.2 HashMap 底层结构（JDK8 数组+链表+红黑树）

**核心答案：**
JDK8 HashMap 采用**数组 + 链表 + 红黑树**结构。数组（`Node[] table`）按 hash 定位桶，同桶元素以链表连接。当链表长度 >= 8 且数组长度 >= 64 时，链表转为红黑树；红黑树节点 <= 6 时退化为链表。

**结构图示：**
```
table[] 数组
 [0] --> null
 [1] --> Node(key1,val1) --> Node(key5,val5) --> null  (链表)
 [2] --> null
 [3] --> TreeNode(红黑树根) --> 子节点...
 ...
[15] --> Node(key16,val16)
```

**要点：**
- 默认初始容量 16，负载因子 0.75
- 哈希计算：`(h = key.hashCode()) ^ (h >>> 16)`（高位参与运算，减少碰撞）
- 桶下标：`index = (n - 1) & hash`（等价于取模，要求容量为 2 的幂）
- 链表转红黑树阈值：`TREEIFY_THRESHOLD = 8`（泊松分布，桶内8个元素概率极低）
- 红黑树转链表阈值：`UNTREEIFY_THRESHOLD = 6`（留有缓冲，避免频繁转换）
- JDK7 采用头插法，JDK8 改为尾插法（解决多线程死循环问题）

---

## 2.3 HashMap 扩容机制（resize，2 倍扩容，rehash）

**核心答案：**
当元素数量超过 `capacity × loadFactor`（默认 `16 × 0.75 = 12`）时触发扩容，容量翻倍（2 倍），重新计算所有元素位置（rehash）。JDK8 利用 2 倍扩容的特性，新位置只有两种情况：**原位置** 或 **原位置 + 旧容量**。

**JDK8 优化 rehash：**
```
旧容量 16，新容量 32
key 的 hash 在第 4 位（16 = 2^4）：
  - 第 4 位为 0：新位置 = 旧位置
  - 第 4 位为 1：新位置 = 旧位置 + 16
判断方式：hash & oldCap（而非重新取模）
链表被分为两组（lo链表 和 hi链表），无需重新计算 hash
```

**代码核心逻辑：**
```java
// JDK8 resize 核心（链表分拆）
Node<K,V> loHead = null, loTail = null;
Node<K,V> hiHead = null, hiTail = null;
do {
    if ((e.hash & oldCap) == 0) { /* lo链 - 原位置 */ }
    else { /* hi链 - 原位置 + 旧容量 */ }
} while ((e = e.next) != null);
newTab[j] = loHead;           // 原位置
newTab[j + oldCap] = hiHead;  // 原位置 + 旧容量
```

---

## 2.4 HashMap 线程不安全（死循环/数据丢失）

**核心答案：**
JDK7 多线程扩容时头插法导致**链表环形死循环**。JDK8 改为尾插法，消除死循环，但仍有**数据覆盖丢失**问题（两线程同时写入同一桶位置）。

**JDK7 死循环原因：**
```
扩容前链表：A --> B --> null
线程1执行到一半，线程2已完成 rehash（B.next=A，形成 A-->B-->A 的环）
线程1继续处理：遍历链表时无限循环！
```

**JDK8 数据丢失场景：**
```java
// 线程1 和 线程2 同时执行 put，hash 值相同（桶位置相同）
// 线程1 读取 tab[i] = null，线程2 读取 tab[i] = null
// 线程2 先写入：tab[i] = nodeB
// 线程1 后写入：tab[i] = nodeA  <-- 覆盖了线程2的数据！
```

**解决方案：** 多线程使用 `ConcurrentHashMap`（推荐），或 `Collections.synchronizedMap()`

---

## 2.5 ConcurrentHashMap JDK8 实现原理

**核心答案：**
JDK8 `ConcurrentHashMap` 放弃了分段锁（Segment），采用**数组 + 链表 + 红黑树** + **CAS + synchronized** 实现细粒度并发。只锁定链表/红黑树的**头节点**，并发度等于数组长度（最大 65536）。

**要点：**
- 初始化：通过 CAS 设置 `sizeCtl` 标志，防止多线程重复初始化
- put 流程：桶为空时 CAS 直接写入；桶不为空时 `synchronized(头节点)` 后写入
- `size()` 使用 `LongAdder` 思想（`CounterCell[]` 数组分散竞争）
- `get()` 全程无锁（Node 的 `val` 和 `next` 均为 `volatile`）
- 扩容协助：多线程可以协助 transfer（`ForwardingNode` 标记已迁移桶）

**结构对比：**
```
JDK7：Segment（继承ReentrantLock） --> HashEntry[]
      并发度 = Segment 数量（默认16）

JDK8：Node[] table
      CAS（空桶） + synchronized（头节点）
      并发度 = table.length（最大65536）
```

---

## 2.6 LinkedHashMap（插入顺序/访问顺序，LRU 实现）

**核心答案：**
`LinkedHashMap` 继承 `HashMap`，内部维护一个**双向链表**连接所有节点，记录插入/访问顺序。`accessOrder=true` 时每次 `get` 将节点移到链表尾部，天然实现 **LRU 缓存**。

**LRU 缓存实现：**
```java
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    public LRUCache(int maxSize) {
        super(maxSize, 0.75f, true);  // accessOrder=true
        this.maxSize = maxSize;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize;  // 超出容量时删除最久未访问的头节点
    }
}

// 使用示例
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "a");  // [1]
cache.put(2, "b");  // [1,2]
cache.put(3, "c");  // [1,2,3]
cache.get(1);       // [2,3,1]（1 移到尾部）
cache.put(4, "d");  // [3,1,4]（2 被淘汰，最久未访问）
```

---

## 2.7 TreeMap（红黑树，compareTo 排序）

**核心答案：**
`TreeMap` 基于**红黑树**实现，保证 key 有序（自然顺序或自定义 Comparator）。增删查均为 O(logn)。不允许 null key（需 compareTo，null 会抛 NPE）。

**要点：**
- 红黑树是近似平衡的 BST，保证最长路径 <= 2 × 最短路径
- 5 个性质：节点红/黑、根黑、叶（NIL）黑、红节点子黑、黑高相同
- `firstKey()`/`lastKey()`：O(logn) 获取最小/最大 key
- `subMap(from, to)`：获取范围子 map
- 线程不安全，多线程用 `Collections.synchronizedSortedMap()`

---

## 2.8 HashSet / LinkedHashSet / TreeSet 底层

**核心答案：**
三者均是对应 Map 的包装（值统一为 `PRESENT` 常量对象）：`HashSet -> HashMap`，`LinkedHashSet -> LinkedHashMap`，`TreeSet -> TreeMap`。所有特性与对应 Map 完全一致。

**源码核心：**
```java
public class HashSet<E> {
    private transient HashMap<E, Object> map;
    private static final Object PRESENT = new Object();
    public boolean add(E e) { return map.put(e, PRESENT) == null; }
    public boolean contains(Object o) { return map.containsKey(o); }
}
```

| 需求 | 选择 |
|------|------|
| 无序不重复，最快 | HashSet |
| 保留插入顺序 | LinkedHashSet |
| 自动排序 | TreeSet |
| 线程安全不重复 | CopyOnWriteArraySet |

---

## 2.9 PriorityQueue（小顶堆，堆化原理）

**核心答案：**
`PriorityQueue` 基于**完全二叉树（数组实现）**的小顶堆。堆顶元素最小（自然顺序），`poll()` 取出堆顶后从最后一个元素开始下沉（sift down），`offer()` 元素追加到末尾后上浮（sift up）。

**堆的数组表示：**
```
       1
      /      3   2
    / \ /
   7  4 5

数组：[1, 3, 2, 7, 4, 5]
父节点 i：子节点为 2i+1（左）、2i+2（右）
子节点 i：父节点为 (i-1)/2
```

**Top-K 问题：**
```java
// 求数组中前K大的数，使用大小为K的小顶堆
PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);
for (int num : nums) {
    minHeap.offer(num);
    if (minHeap.size() > k) {
        minHeap.poll();  // 弹出最小值，保留最大的K个
    }
}
// 堆中剩余即为前K大，时间复杂度 O(nlogk)
```

---

## 2.10 ArrayDeque vs LinkedList 作为栈/队列

**核心答案：**
`ArrayDeque` 基于循环数组，性能优于 `LinkedList`（无节点对象分配，缓存友好）。作为栈和队列均推荐使用 `ArrayDeque`，`LinkedList` 因为还实现了 `List` 接口，功能混杂，不纯粹。

| 特性 | ArrayDeque | LinkedList |
|------|------------|------------|
| 底层 | 循环数组 | 双向链表 |
| 内存 | 连续，缓存友好 | 分散，每节点额外开销 |
| null 元素 | 不允许 | 允许 |
| 推荐场景 | 栈/队列首选 | 需要 List 特性时 |

**使用示例：**
```java
// 作为栈
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1); stack.pop(); stack.peek();

// 作为队列
Deque<Integer> queue = new ArrayDeque<>();
queue.offer(1); queue.poll(); queue.peek();
```

---

# 第三章：Java 并发（JUC）

## 3.1 synchronized 锁升级（偏向锁/轻量级锁/重量级锁）

**核心答案：**
JDK6 对 `synchronized` 进行优化，引入锁升级机制，避免直接用 OS 互斥锁（重量级）。锁只能升级，不能降级。

**锁状态转换：**
```
无锁 --> 偏向锁 --> 轻量级锁 --> 重量级锁
         只有一个线程   多线程交替访问   多线程竞争
```

| 锁状态 | Mark Word 内容 | 适用场景 |
|--------|---------------|---------|
| 无锁 | hashCode + 分代年龄 | 未同步 |
| 偏向锁 | 线程ID + epoch + 分代年龄 | 单线程反复加锁 |
| 轻量级锁 | 栈中 Lock Record 指针 | 多线程交替执行（无竞争） |
| 重量级锁 | monitor 对象指针 | 多线程激烈竞争 |

**要点：**
- **偏向锁**：Mark Word 记录线程 ID，同一线程加锁仅比较 ID，无 CAS
- **轻量级锁**：CAS 将 Mark Word 复制到线程栈（Lock Record），失败则自旋
- **重量级锁**：操作系统 Mutex，线程挂起/唤醒，开销大
- JDK15 废弃偏向锁，JDK21 默认关闭

---

## 3.2 volatile 可见性与禁止重排序原理

**核心答案：**
`volatile` 保证两件事：①**可见性**（写操作立即刷新到主内存，读操作从主内存读取），②**禁止指令重排序**（通过内存屏障）。但不保证原子性（i++ 仍线程不安全）。

**内存屏障（Memory Barrier）：**
```
volatile 写：
  StoreStore 屏障
  volatile 写操作
  StoreLoad 屏障  <-- 开销最大，防止写操作与后续读操作重排

volatile 读：
  volatile 读操作
  LoadLoad 屏障
  LoadStore 屏障
```

**DCL（双重检查锁）单例必须用 volatile：**
```java
public class Singleton {
    private static volatile Singleton instance;  // 必须加 volatile！

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                    // 若无 volatile，new 的 3 步（分配内存/初始化/赋引用）
                    // 可能被重排，导致另一线程拿到未初始化的对象
                }
            }
        }
        return instance;
    }
}
```

---

## 3.3 JMM 内存模型与 happens-before

**核心答案：**
JMM（Java Memory Model）规定线程私有工作内存与主内存的交互规则。**happens-before** 是 JMM 中可见性与有序性的核心抽象：若操作 A happens-before 操作 B，则 A 的结果对 B 可见。

**happens-before 8 条规则：**

| 规则 | 说明 |
|------|------|
| 程序顺序规则 | 单线程内，前面操作 hb 后面操作 |
| 监视器锁规则 | unlock hb 后续对同一锁的 lock |
| volatile 变量规则 | volatile 写 hb 后续对同一变量的读 |
| 线程启动规则 | `Thread.start()` hb 线程内所有操作 |
| 线程终止规则 | 线程所有操作 hb `Thread.join()` 返回 |
| 线程中断规则 | `interrupt()` hb 检测到中断 |
| 对象初始化规则 | 对象构造完成 hb `finalize()` |
| 传递性 | A hb B，B hb C -> A hb C |

**JMM 抽象结构：**
```
线程A                        线程B
[工作内存A: 变量副本]         [工作内存B: 变量副本]
         |  read/write                |
              主内存 [变量]
```

---

## 3.4 ReentrantLock vs synchronized

**核心答案：**
`ReentrantLock` 基于 AQS 实现，功能更丰富（可中断、可超时、公平锁、多条件变量）；`synchronized` 是 JVM 内置关键字，更简洁，JDK6 优化后性能接近。一般优先用 `synchronized`，需要高级功能时用 `ReentrantLock`。

| 特性 | synchronized | ReentrantLock |
|------|--------------|---------------|
| 实现层 | JVM 内置 | JDK（AQS） |
| 是否可中断 | 否 | 是（lockInterruptibly） |
| 超时获取 | 否 | 是（tryLock(time, unit)） |
| 公平锁 | 否 | 是（new ReentrantLock(true)） |
| 多条件变量 | 一个（wait/notify） | 多个（Condition） |
| 是否可重入 | 是 | 是 |
| 自动释放 | 是（退出代码块） | 否（必须 finally unlock） |

**ReentrantLock 使用模板：**
```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    // 临界区
} finally {
    lock.unlock();  // 必须在 finally 中释放！
}

// 超时锁
if (lock.tryLock(1, TimeUnit.SECONDS)) {
    try { /* ... */ } finally { lock.unlock(); }
}
```

---

## 3.5 AQS 核心原理（state + CLH 队列）

**核心答案：**
AQS（AbstractQueuedSynchronizer）是 JUC 中锁和同步器的核心框架。维护一个 `int state` 状态变量和一个 **CLH 变体双向等待队列**。子类通过重写 `tryAcquire/tryRelease` 定义加锁/释放语义。

**AQS 结构：**
```
AQS
|-- volatile int state          // 同步状态（ReentrantLock: 0=未锁, >0=重入次数）
|-- volatile Node head          // 队列头（虚节点）
`-- volatile Node tail          // 队列尾

CLH 等待队列（双向链表）：
HEAD(虚) <--> Node(T1,SIGNAL) <--> Node(T2,SIGNAL) <--> Node(T3) = TAIL
```

**独占锁 acquire 流程：**
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&          // 1. 尝试获取锁（子类实现）
        acquireQueued(                // 3. 自旋等待
            addWaiter(Node.EXCLUSIVE) // 2. 入队
        , arg))
        selfInterrupt();
}
```

**基于 AQS 的工具类：**
- `ReentrantLock`：state 表示重入次数
- `Semaphore`：state 表示许可数量
- `CountDownLatch`：state 表示倒计数值
- `ReentrantReadWriteLock`：state 高16位=读锁数，低16位=写锁重入数

---

## 3.6 ThreadPoolExecutor 七大参数与执行流程

**核心答案：**
线程池通过复用线程降低线程创建/销毁开销。七大参数控制线程池行为，核心是：先用核心线程，满了进队列，队列满了创建非核心线程，都满了执行拒绝策略。

**七大参数：**
```java
new ThreadPoolExecutor(
    int corePoolSize,          // 核心线程数（一直保留）
    int maximumPoolSize,       // 最大线程数
    long keepAliveTime,        // 非核心线程空闲存活时间
    TimeUnit unit,             // 时间单位
    BlockingQueue<Runnable> workQueue,  // 任务队列
    ThreadFactory threadFactory,        // 线程工厂
    RejectedExecutionHandler handler    // 拒绝策略
);
```

**执行流程：**
```
提交任务
    |
当前线程数 < corePoolSize？ 是 --> 创建核心线程执行
    |
任务队列未满？ 是 --> 入队等待
    |
当前线程数 < maximumPoolSize？ 是 --> 创建非核心线程执行
    |
执行拒绝策略
```

| 队列 | 特点 | 适用场景 |
|------|------|---------|
| ArrayBlockingQueue | 有界 | 限制最大任务数 |
| LinkedBlockingQueue | 无界（Integer.MAX_VALUE） | 慎用（OOM风险） |
| SynchronousQueue | 不存储（直接移交） | newCachedThreadPool |
| PriorityBlockingQueue | 优先级排序 | 有优先级的任务 |

---

## 3.7 线程池拒绝策略

**核心答案：**
当任务队列已满且线程数达到最大值时，触发拒绝策略。JDK 内置 4 种，可自定义实现 `RejectedExecutionHandler` 接口。

| 策略 | 行为 | 适用场景 |
|------|------|---------|
| `AbortPolicy`（默认） | 抛出 `RejectedExecutionException` | 调用方需感知失败 |
| `CallerRunsPolicy` | 由提交任务的线程执行 | 降速机制，不丢任务 |
| `DiscardPolicy` | 静默丢弃新任务 | 允许丢任务（日志等） |
| `DiscardOldestPolicy` | 丢弃队列最旧任务，重试提交 | 新任务优先级更高 |

**自定义拒绝策略（持久化到 DB）：**
```java
public class PersistRejectedHandler implements RejectedExecutionHandler {
    @Override
    public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
        log.warn("任务被拒绝，持久化: {}", r);
        database.save(r);  // 持久化，后续补偿执行
    }
}
```

---

## 3.8 CountDownLatch vs CyclicBarrier vs Semaphore

**核心答案：**
三者用途不同：`CountDownLatch` 一次性倒计数等待（主线程等多个子线程），`CyclicBarrier` 可重用的屏障（多线程互相等待到达），`Semaphore` 信号量（控制并发数量）。

| 特性 | CountDownLatch | CyclicBarrier | Semaphore |
|------|---------------|---------------|-----------|
| 计数方向 | 递减 | 递增到屏障数 | 许可池 |
| 是否可重用 | 否 | 是（reset） | 是 |
| 等待方式 | await() 等 count=0 | await() 等所有线程到达 | acquire() 获取许可 |
| 典型场景 | 主线程等子线程完成 | 多线程分阶段协调 | 限流、连接池 |

**代码示例：**
```java
// CountDownLatch：主线程等待3个工人完成
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    executor.submit(() -> { doWork(); latch.countDown(); });
}
latch.await();  // 阻塞直到 count=0

// Semaphore：限制最多5个线程同时访问
Semaphore semaphore = new Semaphore(5);
semaphore.acquire();
try { accessResource(); } finally { semaphore.release(); }
```

---

## 3.9 CAS 原理与 ABA 问题

**核心答案：**
CAS（Compare And Swap）是 CPU 级别的原子指令（`cmpxchg`）。包含三个操作数：内存地址 V、预期值 A、新值 B。仅当 V 的值等于 A 时，才将 V 更新为 B，否则重试。ABA 问题：值从 A->B->A，CAS 无法感知中间变化。

**ABA 问题及解决：**
```
时间线：
T1 读到 V=A
T2 修改 V: A --> B --> A
T1 CAS(A->C) 成功，但实际 V 已被修改过！

解决：AtomicStampedReference（带版本号的引用）
V = A，stamp = 1
T2: A-->B(stamp=2) --> B-->A(stamp=3)
T1: CAS(A,C, stamp=1->2) 失败！因为当前 stamp=3
```

**代码示例：**
```java
// 普通 CAS（有 ABA 问题）
AtomicInteger count = new AtomicInteger(0);
count.compareAndSet(0, 1);

// 带版本号（解决 ABA）
AtomicStampedReference<Integer> stampedRef =
    new AtomicStampedReference<>(100, 1);
int stamp = stampedRef.getStamp();
stampedRef.compareAndSet(100, 200, stamp, stamp + 1);
```

---

## 3.10 LongAdder vs AtomicLong

**核心答案：**
`AtomicLong` 单个变量 CAS，高并发下竞争激烈（大量 CAS 失败重试，性能下降）。`LongAdder` 采用**分散竞争**策略：维护 `base` + `Cell[]` 数组，不同线程更新不同 Cell，最终 `sum()` 汇总，高并发下吞吐量大幅提升。

**LongAdder 结构：**
```
LongAdder
|-- volatile long base    // 无竞争时直接 CAS base
`-- Cell[] cells          // 有竞争时每个线程映射一个 Cell
    |-- Cell[0]: 100
    |-- Cell[1]: 200
    `-- Cell[2]: 300

sum() = base + Cell[0] + Cell[1] + Cell[2] = 600
```

**要点：**
- `LongAdder.sum()` 不是强一致（并发时读到中间状态），适合统计计数
- `AtomicLong` 适合需要精确值的原子操作（如 ID 生成）
- Cell 数组大小为 2 的幂，最大等于 CPU 核心数

---

## 3.11 ThreadLocal 内存泄漏

**核心答案：**
`ThreadLocal` 存储在 `Thread.threadLocals`（`ThreadLocalMap`）中，Key 为 `ThreadLocal` 弱引用，Value 为强引用。当 `ThreadLocal` 变量置 null 后，Key 被 GC 回收，但 Value 仍被强引用，Entry 无法回收，造成内存泄漏，线程池场景下尤其严重。

**内存结构：**
```
Thread
`-- ThreadLocalMap
    |-- Entry(WeakRef(ThreadLocal1), Value1)  <-- key GC后变null
    |-- Entry(null, Value2)                   <-- 内存泄漏！Value2无法被GC
    `-- Entry(WeakRef(ThreadLocal3), Value3)
```

**解决方案：**
```java
ThreadLocal<Connection> connHolder = new ThreadLocal<>();
try {
    connHolder.set(getConnection());
    // 使用连接...
} finally {
    connHolder.remove();  // 必须！尤其在线程池中
}
```

---

## 3.12 CompletableFuture 异步编排

**核心答案：**
`CompletableFuture` 是 JDK8 引入的异步编程框架，支持链式调用、组合多个异步任务、异常处理。基于 `ForkJoinPool.commonPool()`（默认）或自定义线程池执行。

**核心 API：**
```java
// 1. 创建异步任务
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello", executor);

// 2. 串行执行
CompletableFuture<String> f2 = f1.thenApply(s -> s + " World");

// 3. 并行执行，等待全部完成
CompletableFuture.allOf(f1, f2, f3).join();

// 4. 并行执行，任一完成即返回
CompletableFuture.anyOf(f1, f2, f3).thenAccept(result -> {});

// 5. 异常处理
f1.exceptionally(e -> "default value")
  .whenComplete((result, ex) -> log.info("done: {}", result));

// 6. 两任务合并
f1.thenCombine(f2, (r1, r2) -> r1 + " " + r2);
```

---

## 3.13 死锁四个必要条件与检测

**核心答案：**
死锁四个必要条件（Coffman 条件）：互斥、持有并等待、不可抢占、循环等待。四个条件**缺一不可**，破坏任意一个即可预防死锁。

| 条件 | 说明 | 破坏方式 |
|------|------|---------|
| 互斥 | 资源同时只能被一个线程使用 | 使用可共享资源（读锁） |
| 持有并等待 | 持有资源时等待其他资源 | 一次性申请所有资源 |
| 不可抢占 | 已获得资源不可被强制剥夺 | 超时放弃（tryLock） |
| 循环等待 | 形成资源等待环 | 统一资源申请顺序 |

**死锁检测（jstack）：**
```bash
jps -l                          # 找到 Java 进程 PID
jstack <pid> > thread_dump.txt  # 导出线程快照
# 输出示例：
# Found 1 deadlock.
# "Thread-1": waiting to lock <0x000...> (held by "Thread-0")
# "Thread-0": waiting to lock <0x000...> (held by "Thread-1")
```

---

# 第四章：JVM

## 4.1 JVM 内存区域

**核心答案：**
JVM 内存分为：**线程共享区**（堆、方法区/元空间）和**线程私有区**（程序计数器、Java 虚拟机栈、本地方法栈）。

**内存区域图示：**
```
+-----------------------------------------------------+
|                   JVM 内存区域                       |
+----------------------------+------------------------+
|       线程共享              |       线程私有          |
+----------+------------------+--------+-------+------+
|   堆     |  方法区/元空间    | PC寄存器|虚拟机栈|本地栈|
| (对象实例)|  (类信息/常量/   |(字节码  |(栈帧: |(native|
| Eden/S0  |  静态变量/运行   | 行号,   |局变量/ | 方法) |
| S1/Old   |  时常量池)       | 无OOM)  |操作数栈|      |
+----------+------------------+--------+-------+------+
```

| 区域 | OOM 类型 | 常见原因 |
|------|---------|---------| 
| 堆 | `Java heap space` | 内存泄漏/对象过多 |
| 方法区/元空间 | `Metaspace` | 动态生成大量类（CGLib） |
| 虚拟机栈 | `StackOverflowError` | 递归过深 |
| 直接内存 | `Direct buffer memory` | NIO 频繁分配 |

---

## 4.2 对象创建流程

**核心答案：**
JVM 创建对象经历 5 步：类加载检查 -> 分配内存 -> 零值初始化 -> 设置对象头 -> 执行 `<init>` 方法。

**详细流程：**
```
1. 类加载检查：检查该类是否已被加载、解析、初始化，若未加载则先执行类加载

2. 分配内存：
   - 内存规整（Serial/ParNew）-> 指针碰撞（Bump the Pointer）
   - 内存不规整（CMS）-> 空闲列表（Free List）
   - 并发安全：CAS + 失败重试 / TLAB（Thread Local Allocation Buffer）

3. 初始化零值：int=0, boolean=false, reference=null（不执行用户代码）

4. 设置对象头（Object Header）：
   - Mark Word：hashCode、GC年龄、锁状态
   - 类型指针：指向类元数据

5. 执行 <init>（构造函数）：按用户代码初始化字段
```

**TLAB 优化：**
```
线程本地分配缓冲（Thread Local Allocation Buffer）
每个线程预先申请一小块堆内存，线程内分配无需同步
默认占 Eden 区 1%，可通过 -XX:TLABSize 调整
```

---

## 4.3 对象内存布局（对象头/实例数据/对齐填充）

**核心答案：**
Java 对象由三部分组成：**对象头**（Mark Word + 类型指针）、**实例数据**（字段值）、**对齐填充**（凑整到 8 字节倍数）。64位JVM（默认开启压缩指针）对象头 12 字节。

**布局示例：**
```
64位JVM（开启压缩指针 -XX:+UseCompressedOops）

+-------------------------------------------+
|  Mark Word（8字节）                         |
|  HashCode(25bit)|Age(4bit)|Bias(1)|Lock(2) |
+-------------------------------------------+
|  Klass Pointer（4字节，压缩后）              |
+-------------------------------------------+
|  实例数据（字段）                            |
|  long(8字节) > int(4字节) > ...             |
+-------------------------------------------+
|  对齐填充（0~7字节，使总大小为8的倍数）       |
+-------------------------------------------+

一个 Object 对象大小：16字节（头12 + 填充4）
一个含 int 字段的对象：16字节（头12 + int4）
```

---

## 4.4 垃圾回收算法（标记清除/标记整理/复制算法）

**核心答案：**
三种基础 GC 算法各有取舍，现代 GC 通常结合使用。

| 算法 | 步骤 | 优点 | 缺点 | 适用 |
|------|------|------|------|------|
| 标记-清除 | 标记存活对象->清除未标记 | 无需移动对象 | 内存碎片，效率低 | CMS 老年代 |
| 标记-整理 | 标记->将存活对象移到一端->清除边界外 | 无碎片 | 移动对象开销大 | Serial Old/Parallel Old |
| 复制 | 将存活对象复制到另半区->清空当前半区 | 无碎片，分配快 | 浪费一半空间 | 新生代（8:1:1）|

**新生代 8:1:1 复制算法：**
```
Eden(80%) + Survivor0(10%) + Survivor1(10%)

Minor GC：
  Eden + S0（From）中存活对象 --> 复制到 S1（To）
  年龄+1，达到阈值（默认15）--> 晋升老年代
  若 S1 空间不足 --> 直接晋升老年代（空间担保）
  S0 和 S1 角色互换
```

---

## 4.5 分代收集（Eden/S0/S1/老年代）

**核心答案：**
分代假说：大多数对象朝生夕灭（弱代假说），少量对象长时间存活（强代假说）。基于此将堆分为**新生代**（频繁回收）和**老年代**（偶尔回收）。

**堆内存结构：**
```
+-----------------------------------------------------+
|                    堆内存                            |
+-----------------------------+-----------------------+
|         新生代（1/3）        |      老年代（2/3）     |
+-----------+--------+--------+-----------------------+
|  Eden(80%)| S0(10%)|S1(10%) |  存放长期存活对象       |
|  新对象分配| Survivor|       |  大对象直接进老年代      |
+-----------+--------+--------+-----------------------+
```

**对象晋升老年代条件：**
1. 年龄超过阈值（默认 `-XX:MaxTenuringThreshold=15`）
2. Survivor 区相同年龄对象超过 Survivor 50%（动态年龄判断）
3. Survivor 区空间不足（空间担保）
4. 大对象（`-XX:PretenureSizeThreshold` 超过阈值直接进老年代）

---

## 4.6 GC 收集器对比（Serial/Parallel/CMS/G1/ZGC）

**核心答案：**
各 GC 收集器适用不同场景，从 Serial（单线程）-> Parallel（吞吐量优先）-> CMS（低延迟，已废弃）-> G1（均衡）-> ZGC/Shenandoah（超低延迟）演进。

| 收集器 | 代 | 线程 | 算法 | 目标 | 适用场景 |
|--------|-----|------|------|------|---------|
| Serial | 新生代 | 单线程 | 复制 | 简单低开销 | 客户端，内存<100MB |
| ParNew | 新生代 | 多线程 | 复制 | 配合CMS | 与CMS组合 |
| Parallel Scavenge | 新生代 | 多线程 | 复制 | 吞吐量 | 批处理，后台计算 |
| Parallel Old | 老年代 | 多线程 | 标记整理 | 吞吐量 | 同上 |
| CMS | 老年代 | 多线程 | 标记清除 | 低停顿 | JDK9废弃 |
| G1 | 全堆 | 多线程 | 混合 | 可预期停顿 | JDK9+默认 |
| ZGC | 全堆 | 多线程 | 染色指针 | 超低延迟(<10ms) | JDK15+稳定，大内存 |

---

## 4.7 G1 收集器原理（Region/混合回收/SATB）

**核心答案：**
G1（Garbage First）将堆划分为大小相等的 **Region**（默认2048个，1~32MB），通过维护每个 Region 的**回收收益估算**，优先回收价值最高的 Region，实现可预期停顿时间（`-XX:MaxGCPauseMillis`）。

**G1 回收阶段：**
```
1. 初始标记（STW，<1ms）：标记 GC Roots 直接可达对象
2. 并发标记（并发，可达性分析）：与应用线程并发执行
3. 最终标记（STW，SATB处理）：处理并发标记期间的变动
4. 筛选回收（STW）：选取回收收益高的 Region，复制/清理

SATB（Snapshot At The Beginning）：
  并发标记开始时创建存活对象快照
  新增的引用关系通过写屏障记录到 SATB 队列，防止漏标
```

**Region 布局：**
```
G1 堆 = N×Region
+---+---+---+---+---+---+---+---+
| E | E | S | O | O | H | E | O |
+---+---+---+---+---+---+---+---+
E=Eden  S=Survivor  O=Old  H=Humongous(大对象,占连续Region)
```

---

## 4.8 类加载机制（加载/验证/准备/解析/初始化）

**核心答案：**
类的生命周期：加载->验证->准备->解析->初始化->使用->卸载。其中**验证、准备、解析**合称**链接**。

| 阶段 | 工作内容 |
|------|---------|
| 加载 | 读取字节码到内存，创建 `java.lang.Class` 对象 |
| 验证 | 字节码格式/语义/字节码/符号引用验证，确保安全 |
| 准备 | 为静态变量分配内存，赋**零值**（int=0，非初始值） |
| 解析 | 符号引用->直接引用（内存地址），可延迟（动态绑定） |
| 初始化 | 执行 `<clinit>`（静态变量赋值+静态代码块），JVM保证线程安全 |

**类初始化触发时机（主动引用）：**
- `new` 一个类的实例
- 调用类的静态方法/读写静态字段（常量除外）
- 反射调用
- 初始化子类时先初始化父类
- JVM 启动时初始化 Main 类

---

## 4.9 双亲委派模型及破坏场景

**核心答案：**
双亲委派：类加载器收到加载请求，先委托父加载器处理，父加载器无法处理才自己加载。保证核心类库（如 `java.lang.String`）只能由 BootstrapClassLoader 加载，防止恶意类替换。

**加载器层次：**
```
Bootstrap ClassLoader（C++实现，加载 JDK/lib）
        ^
        | 父
Extension/Platform ClassLoader（加载 JDK/lib/ext）
        ^
        | 父
Application ClassLoader（加载 classpath）
        ^
        | 父
自定义 ClassLoader
```

| 场景 | 说明 |
|------|------|
| JNDI/JDBC/JCE（SPI） | 核心类（BootstrapCL）调用第三方实现（AppCL），需线程上下文CL |
| OSGi 模块化 | 每个 Bundle 有独立 CL，类加载为网状而非树状 |
| Tomcat 多应用隔离 | 每个 WebApp 有独立 CL，相同类名加载不同版本 |
| 热部署/热更新 | 替换 CL 实现动态更新（Arthas、Spring DevTools） |

---

## 4.10 OOM 类型及排查

**核心答案：**
常见 OOM 类型：堆内存溢出（最常见）、元空间溢出（动态代理/CGLib）、直接内存溢出（NIO）、栈溢出（StackOverflowError）。

**排查步骤：**
```bash
# 1. 导出堆转储文件（事前配置）
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/path/to/dump.hprof

# 2. 或运行时手动导出
jmap -dump:format=b,file=dump.hprof <pid>

# 3. 分析工具
# MAT（Eclipse Memory Analyzer）：找 Leak Suspects
# VisualVM：堆 Dump 分析
# Arthas：在线分析

# 4. 元空间OOM排查
jstat -gcmetacapacity <pid> 1000
```

| OOM 类型 | 常见原因 | 解决方案 |
|---------|---------|---------|
| heap space | 内存泄漏/缓存过大/大对象 | MAT分析 + 增大堆 |
| Metaspace | CGLib代理/JSP重加载/ClassLoader泄漏 | 增大元空间/检查CL |
| Direct buffer | NIO直接内存/Netty | 增大MaxDirectMemorySize |

---

## 4.11 CPU 100% 排查步骤

**核心答案：**
CPU 飙高通常由：死循环、大量 GC、正则回溯、JSON序列化等引起。排查步骤：top -> 定位进程 -> 定位线程（`top -Hp`）-> 转换线程ID -> `jstack` 找堆栈。

**标准排查流程：**
```bash
# Step 1: 找到 CPU 高的 Java 进程
top  # 按 P 按CPU排序，找到 PID

# Step 2: 找到该进程内 CPU 高的线程
top -Hp <pid>
# 记录 CPU 最高的线程 TID（如 12345）

# Step 3: 将 TID 转换为十六进制
printf '%x\n' 12345  # 输出：3039

# Step 4: 用 jstack 查找该线程堆栈
jstack <pid> | grep -A 30 "nid=0x3039"

# Step 5: 分析堆栈，定位问题代码
```

---

## 4.12 JIT 编译与逃逸分析

**核心答案：**
JIT（Just-In-Time）编译器将热点字节码编译为本地机器码，极大提升执行速度。逃逸分析（Escape Analysis）分析对象是否逃逸出方法/线程，基于此做栈上分配、标量替换等优化。

| 概念 | 说明 |
|------|------|
| C1 编译器 | 快速编译，简单优化，适合启动速度 |
| C2 编译器 | 慢速编译，深度优化，适合长期运行 |
| 分层编译（JDK7+）| 先C1快速编译，热点代码再C2深度优化 |
| OSR（栈上替换）| 循环执行时将方法替换为编译版本 |

**逃逸分析优化：**
```java
// 逃逸分析：对象不逃逸，可在栈上分配
public void method() {
    Point p = new Point(1, 2);  // p 不逃逸出方法
    int result = p.x + p.y;     // 标量替换：直接用 int x=1, y=2
    // 经过优化后，Point 对象不需要在堆上分配！
}

// 同步消除：对象不逃逸，锁无竞争
public void sync() {
    Object lock = new Object();  // 局部变量，不逃逸
    synchronized (lock) {        // 锁消除！不需要加锁
        doSomething();
    }
}
```

---

# 第五章：MySQL

## 5.1 InnoDB vs MyISAM

**核心答案：**
InnoDB 支持事务、行锁、外键、崩溃恢复，是 MySQL 5.5+ 的默认存储引擎。MyISAM 不支持事务/行锁，适合读多写少的归档场景（如今极少使用）。

| 特性 | InnoDB | MyISAM |
|------|--------|--------|
| 事务 | 支持（ACID） | 不支持 |
| 锁粒度 | 行锁 | 表锁 |
| 外键 | 支持 | 不支持 |
| 崩溃恢复 | 支持（redo log） | 不支持（易损坏） |
| COUNT(*) | 需扫描（无缓存） | 直接读取元数据（快） |
| 聚簇索引 | 是（主键=数据） | 否（索引与数据分离） |
| 存储文件 | .ibd（数据+索引） | .myd + .myi |
| 缓冲池 | Buffer Pool | Key Cache |

---

## 5.2 B+ 树索引原理（为什么不用 B 树/红黑树/Hash）

**核心答案：**
MySQL InnoDB 使用 **B+ 树**作为索引结构：叶节点存完整数据（行记录或主键），内节点仅存 key，叶节点双向链表连接（支持范围查询）。B+ 树高度低（千万数据仅3~4层），IO 次数少。

| 结构 | 不适合原因 |
|------|----------|
| 哈希 | 不支持范围/排序/模糊查询，哈希碰撞 |
| 红黑树 | 百万数据树高30+，IO 次数多 |
| B 树 | 内节点也存数据，每页存的 key 更少，树更高；不支持顺序访问 |
| B+ 树 | 内节点只存 key（页内 key 数多，树矮）；叶节点链表支持范围查询 |

**B+ 树结构：**
```
                [10 | 30]            <- 非叶节点，只存 key
               /    |             [5|7]    [15|20]  [35|50]   <- 非叶节点
        /   \      ...      ...
 [1,2,3,4]-[5,6,7]--...--[35...]    <- 叶节点存完整数据，双向链表

高度估算：
InnoDB 页大小：16KB
内节点：每个 key+指针 约14字节 -> 16×1024/14 约 1170 个指针
叶节点：每行1KB -> 16行/页
3 层 B+ 树：1170 × 1170 × 16 约 2100万行
```

---

## 5.3 聚簇索引 vs 非聚簇索引（回表/覆盖索引）

**核心答案：**
**聚簇索引**：叶节点存储完整行记录，InnoDB 主键索引即聚簇索引（一个表只有一个）。**非聚簇索引**：叶节点存储主键值，查询时需**回表**（再查一次聚簇索引）。**覆盖索引**：查询字段全在索引中，无需回表。

**回表示意：**
```
查询：SELECT name FROM user WHERE age = 25

非聚簇索引（age）：
age=25 --> 叶节点存 [age=25, pk=100]
                          | 回表
                          v
聚簇索引（pk）：pk=100 --> 完整行 [id=100, age=25, name="张三", ...]

覆盖索引（age, name）：
age=25 --> 叶节点存 [age=25, name="张三", pk=100]  直接返回，无需回表！
```

**要点：**
- 无主键时，InnoDB 使用第一个 NOT NULL UNIQUE 列，否则生成隐式6字节 rowid
- 二级索引都存储主键值，主键越短二级索引越小（建议用 INT 主键而非 UUID）
- `EXPLAIN` 中 `Extra: Using index` 表示覆盖索引

---

## 5.4 联合索引最左前缀原则

**核心答案：**
联合索引（a, b, c）底层按 a->b->c 的顺序排序。查询时必须从最左列开始且不能跳过中间列，否则后续列无法利用索引。

```sql
INDEX(a, b, c)

-- 能用索引
WHERE a = 1                          -- 用 a
WHERE a = 1 AND b = 2                -- 用 a, b
WHERE a = 1 AND b = 2 AND c = 3      -- 用 a, b, c
WHERE a = 1 AND c = 3                -- 只用 a（跳过 b，c 不走索引）
WHERE a > 1 AND b = 2                -- 只用 a（范围查询后的列不走索引）

-- 不能用索引
WHERE b = 2                          -- 缺少最左列 a
WHERE b = 2 AND c = 3                -- 同上
WHERE c = 3                          -- 同上
```

---

## 5.5 EXPLAIN 关键字段

**核心答案：**
`EXPLAIN` 分析查询执行计划，关键字段：`type`（访问类型）、`key`（实际使用的索引）、`rows`（预估扫描行数）、`Extra`（附加信息）。

**type 字段值（从好到差）：**
```
system > const > eq_ref > ref > range > index > ALL

const ：通过主键/唯一索引 = 条件，最多1行（WHERE id=1）
eq_ref：join时每行精确匹配一行（主键关联）
ref   ：使用非唯一索引 = 查询
range ：索引范围扫描（BETWEEN/IN/>/<）
index ：全索引扫描（比 ALL 好，不用回表）
ALL   ：全表扫描（必须优化！）
```

| Extra 值 | 含义 | 是否需优化 |
|---------|------|----------|
| Using index | 覆盖索引，无需回表 | 优 |
| Using where | WHERE 过滤 | 正常 |
| Using filesort | 额外排序 | 需优化 |
| Using temporary | 使用临时表 | 需优化 |

---

## 5.6 索引失效场景（10种）

**核心答案：**
以下情况导致索引失效，触发全表扫描：

```sql
-- 1. 对索引列做函数/计算操作
WHERE YEAR(create_time) = 2024  -- 改为范围查询
WHERE id + 1 = 10               -- 改为 id = 9

-- 2. 隐式类型转换（字符串字段用数字查询）
WHERE phone = 13800138000        -- phone是VARCHAR，转换后失效

-- 3. 前导模糊查询
WHERE name LIKE '%张%'           -- 失效
WHERE name LIKE '张%'            -- 正确（后缀模糊可用索引）

-- 4. OR 连接的字段未全部建索引
WHERE a = 1 OR b = 2             -- 若 b 无索引，全表扫描

-- 5. NOT IN / NOT EXISTS
WHERE id NOT IN (1, 2, 3)        -- 可改为左连接

-- 6. IS NULL / IS NOT NULL（受null值比例影响优化器决策）

-- 7. 联合索引违反最左前缀
-- INDEX(a,b,c), WHERE b=2 -> 失效

-- 8. 索引列区分度太低（优化器放弃索引）
WHERE gender = '男'              -- 只有两个值，全表扫描更快

-- 9. 范围查询后的列不走索引
-- INDEX(a,b), WHERE a>1 AND b=2 -> b 不走索引

-- 10. 使用 <> 或 !=
WHERE status != 0                -- 一般不走索引
```

---

## 5.7 事务 ACID 与隔离级别

**核心答案：**
事务四大特性 ACID：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）。

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|---------|------|-----------|------|------|
| READ UNCOMMITTED | 存在 | 存在 | 存在 | 最高 |
| READ COMMITTED | 解决 | 存在 | 存在 | 高 |
| REPEATABLE READ（MySQL默认）| 解决 | 解决 | 基本解决 | 中 |
| SERIALIZABLE | 解决 | 解决 | 解决 | 最低 |

**三大并发问题：**
- **脏读**：读到其他事务未提交的数据
- **不可重复读**：同一事务内两次读同一数据，值不同（其他事务已提交修改）
- **幻读**：同一事务内两次查询，行数不同（其他事务插入新行）

**ACID 实现机制：**
- 原子性：undo log（回滚）
- 持久性：redo log（崩溃恢复）
- 隔离性：MVCC + 锁
- 一致性：以上三个共同保证

---

## 5.8 MVCC 实现原理（ReadView/undo log 版本链）

**核心答案：**
MVCC（Multi-Version Concurrency Control）通过维护数据的多个历史版本实现读写不阻塞。每行记录包含隐藏字段 `trx_id`（事务ID）和 `roll_pointer`（指向 undo log 的版本链）。读操作通过 **ReadView** 判断哪个版本可见。

**版本链示意：**
```
当前行: [name="王五", trx_id=100, roll_ptr]
                                    |
                                    v
undo log版本链: [name="李四", trx_id=50, roll_ptr]
                                    |
                                    v
              [name="张三", trx_id=10, roll_ptr --> null]
```

**ReadView 可见性判断规则：**
```
对于版本链中某版本的 trx_id：
1. trx_id == creator_trx_id -> 自己修改的，可见
2. trx_id < min_trx_id -> 已提交的旧事务，可见
3. trx_id >= max_trx_id -> 将来的事务，不可见
4. min_trx_id <= trx_id < max_trx_id：
   - trx_id 在活跃事务列表 m_ids 中 -> 未提交，不可见
   - trx_id 不在 m_ids 中 -> 已提交，可见
```

**RC vs RR 的 ReadView 时机：**
- **RC（读已提交）**：每次 SELECT 都创建新的 ReadView -> 能看到最新提交
- **RR（可重复读）**：事务第一次 SELECT 时创建 ReadView，后续复用 -> 快照读一致

---

## 5.9 行锁/间隙锁/临键锁

**核心答案：**
InnoDB 在 REPEATABLE READ 下通过**临键锁（Next-Key Lock）**防止幻读。行锁锁定具体记录，间隙锁锁定记录间的间隙（防插入），临键锁 = 行锁 + 其左边的间隙锁。

**锁类型图示：**
```
数据行：1, 5, 10, 20

行锁：    只锁 id=5 这一行
间隙锁：  锁 (5,10) 这个区间，防止插入 6,7,8,9
临键锁：  锁 (5,10] 区间 = 间隙锁(5,10) + 行锁(10)

加锁规则（简化）：
- 等值查询命中记录 --> 行锁
- 等值查询未命中 --> 间隙锁
- 范围查询 --> 临键锁（每个扫描到的行+左侧间隙）
- 唯一索引等值查询 --> 退化为行锁（无需间隙锁）
```

---

## 5.10 redo log / undo log / binlog 三者区别

**核心答案：**
三种日志职责完全不同：redo log 保证**持久性**（崩溃恢复），undo log 保证**原子性**（事务回滚/MVCC），binlog 用于**主从复制和时间点恢复**。

| 特性 | redo log | undo log | binlog |
|------|---------|---------|--------|
| 所属层 | InnoDB 引擎 | InnoDB 引擎 | MySQL Server |
| 内容 | 物理日志（页的修改） | 逻辑日志（回滚信息） | 逻辑日志（SQL/行变更）|
| 大小 | 固定大小（循环写） | 存在 undo 表空间 | 可无限增长 |
| 用途 | 崩溃恢复 | 事务回滚 + MVCC | 主从复制 + 时间点恢复 |
| 写入时机 | 事务执行中（WAL先写日志）| 事务执行中 | 事务提交时 |

---

## 5.11 两阶段提交（崩溃恢复）

**核心答案：**
为保证 redo log 和 binlog 的一致性，InnoDB 采用**两阶段提交（2PC）**：先写 redo log（prepare 状态）-> 写 binlog -> redo log 标记 commit。崩溃恢复时依据两者状态判断是否提交。

**两阶段提交流程：**
```
事务提交过程：
1. 写 redo log（prepare） <- 持久化
2. 写 binlog               <- 持久化
3. 写 redo log（commit）   <- 持久化

崩溃恢复规则：
- 崩溃在步骤1（redo=prepare，无binlog）-> 回滚
- 崩溃在步骤2（redo=prepare，有binlog）-> 提交（binlog已写，从库已同步）
- 崩溃在步骤3（redo=commit）-> 正常提交
```

---

## 5.12 主从复制原理（binlog 同步）

**核心答案：**
MySQL 主从复制基于 binlog：主库写 binlog -> 从库 IO 线程请求 binlog -> 写入 relay log -> SQL 线程回放 relay log。

**复制流程：**
```
主库(Master)                          从库(Slave)
[业务操作] -> [binlog 写入]
[Binlog Dump 线程] ---binlog---> [IO 线程] --> relay log
                                 [SQL 线程] <-- relay log
                                       |
                                 [执行SQL，更新数据]
```

| 类型 | 说明 | 优缺点 |
|------|------|------|
| 异步复制（默认）| 主库不等从库确认 | 性能高，可能丢数据 |
| 半同步复制 | 至少一个从库确认 | 性能略低，不丢数据 |
| 全同步复制 | 所有从库确认 | 性能最低，强一致 |
| GTID 复制 | 全局事务ID，自动定位 | 运维更简单，推荐 |

---

## 5.13 慢查询优化思路

**核心答案：**
慢查询优化五步法：定位慢SQL -> EXPLAIN 分析 -> 索引优化 -> SQL 改写 -> 表结构优化。

| 问题 | 优化方案 |
|------|---------|
| 全表扫描（type=ALL）| 添加合适索引 |
| 回表多（大量随机IO）| 使用覆盖索引 |
| filesort | 利用索引排序（ORDER BY 字段加索引）|
| 大 OFFSET 分页 | 延迟关联：WHERE id > last_id LIMIT N |
| JOIN 无索引 | 给 JOIN 字段加索引，小表驱动大表 |
| SELECT * | 只查必要字段 |

```bash
# 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;
# 分析慢查询日志
mysqldumpslow -s t -t 10 /path/to/slow.log  # Top10 慢查询
```

---


---

## 第六章 Redis

### 6.1 Redis 跳表（Skip List）原理

**核心答案：**
跳表是 Redis 有序集合（ZSet）底层数据结构之一。它在有序链表基础上增加多层索引，使查找、插入、删除的平均时间复杂度达到 O(log N)，空间复杂度 O(N)。与红黑树相比，实现简单、范围查询更高效。

**关键考点：**
- 节点层数随机决定，期望层数 O(log N)
- 查找时从最高层向下逐层跳跃，缩小范围
- ZSet 数据量少时用 ziplist，超过阈值转跳表
- Redis 跳表节点包含：score、member、后退指针、各层前进指针

```
Level 3: head ---------> [30] ---------> tail
Level 2: head --> [10] -> [30] -> [50] -> tail
Level 1: head -> [5]->[10]->[20]->[30]->[40]->[50] -> tail
```

---

### 6.2 Redis 持久化：RDB 与 AOF

**核心答案：**
RDB 是快照持久化，将某时刻内存数据以二进制文件保存；AOF 以追加日志方式记录每条写命令。两者可混合使用（Redis 4.0+ 混合持久化）。

**关键考点：**
- RDB：fork 子进程做快照，文件小、恢复快；但数据可能丢失（两次快照间隔内）
- AOF：默认每秒 fsync，数据更安全；文件较大，恢复慢
- AOF 重写：BGREWRITEAOF 命令，压缩 AOF 文件
- 混合持久化：AOF 文件前半段是 RDB 快照 + 后半段增量 AOF
- 恢复优先级：AOF 存在则优先用 AOF 恢复

| 特性 | RDB | AOF |
|------|-----|-----|
| 数据安全性 | 低（可能丢失分钟级） | 高（最多丢失1秒） |
| 文件大小 | 小（二进制压缩） | 大（文本命令） |
| 恢复速度 | 快 | 慢 |
| 性能影响 | fork 时有短暂停顿 | 每秒fsync影响较小 |

---

### 6.3 Redis 缓存穿透、击穿、雪崩

**核心答案：**
- **穿透**：查询不存在的 key，每次都打到数据库。解决：布隆过滤器 / 缓存空值
- **击穿**：热点 key 过期瞬间，大量请求直击数据库。解决：互斥锁重建 / 逻辑过期
- **雪崩**：大量 key 同时过期或 Redis 宕机。解决：过期时间加随机值 / 集群 / 熔断限流

**关键考点：**
- 布隆过滤器：空间高效，存在误判率，不支持删除
- 互斥锁重建：只有一个线程重建缓存，其余等待（双重检查）
- 逻辑过期：不设 TTL，数据中存过期时间字段，过期后异步更新
- 雪崩时熔断降级：返回默认值或提示"服务繁忙"

```
穿透：请求 -> Cache Miss -> DB Miss -> 返回空（需拦截）
击穿：请求 -> key刚过期 -> 大量并发 -> DB 压力激增
雪崩：大批key同时过期 -> 全部打DB -> DB 崩溃
```

---

### 6.4 Redis 过期策略与内存淘汰

**核心答案：**
Redis 过期采用**惰性删除 + 定期删除**组合策略；内存不足时根据 maxmemory-policy 配置进行淘汰。

**关键考点：**
- 惰性删除：访问时检查是否过期，过期则删除（节省 CPU，但内存浪费）
- 定期删除：每 100ms 随机抽取一批 key，删除其中过期的
- 8 种淘汰策略：
  - noeviction：不淘汰，写入报错
  - allkeys-lru：所有 key 中 LRU 淘汰（最常用）
  - volatile-lru：只在设了过期时间的 key 中 LRU 淘汰
  - allkeys-lfu：LFU 算法（Redis 4.0+）
  - allkeys-random / volatile-random：随机淘汰
  - volatile-ttl：淘汰剩余 TTL 最短的 key

---

### 6.5 Redis 集群方案

**核心答案：**
Redis 提供三种集群方案：主从复制、哨兵模式、Redis Cluster（分片集群）。

**关键考点：**
- **主从复制**：master 负责写，slave 负责读；全量同步用 RDB，增量同步用 repl_backlog
- **哨兵模式**：哨兵监控 master/slave，master 故障时自动选举新 master（raft 投票）
- **Redis Cluster**：数据分布到 16384 个 slot，每个节点负责部分 slot；客户端根据 CRC16(key)%16384 路由
- Cluster 无中心化，节点间通过 Gossip 协议通信
- Cluster 要求至少 3 主 3 从，保证高可用

```
Redis Cluster 路由：
Client -> CRC16(key) % 16384 -> 对应 slot -> 对应节点
节点A: slot 0-5460
节点B: slot 5461-10922
节点C: slot 10923-16383
```

---

### 6.6 Redis 事务与 Lua 脚本

**核心答案：**
Redis 事务通过 MULTI/EXEC 实现命令批量执行，但不支持回滚。Lua 脚本可保证原子性，是实现复杂原子操作的推荐方式。

**关键考点：**
- MULTI 开启事务，EXEC 执行，DISCARD 取消
- 编译错误（语法错误）：全部不执行；运行时错误：其他命令仍执行（无回滚）
- WATCH 实现乐观锁：监视 key，若 EXEC 前 key 被修改则事务失败
- Lua 脚本：EVALSHA 复用，保证原子性，适合限流、扣库存等场景

```lua
-- Lua 原子扣减库存示例
local stock = tonumber(redis.call('get', KEYS[1]))
if stock > 0 then
    redis.call('decr', KEYS[1])
    return 1
end
return 0
```

---

### 6.7 Redis 分布式锁

**核心答案：**
使用 SET key value NX PX timeout 命令实现分布式锁，value 使用唯一标识防止误删。Redisson 客户端提供了看门狗机制自动续期。

**关键考点：**
- SET NX PX 原子操作，避免死锁
- value 使用 UUID，释放时先判断是否是自己的锁（Lua 脚本保证原子性）
- 看门狗（WatchDog）：持有锁时每隔 1/3 过期时间自动续期
- RedLock：在 N 个独立 Redis 实例上加锁，超过半数成功才算加锁成功
- RedLock 争议：Martin Kleppmann 指出时钟漂移问题

```java
// Redisson 分布式锁示例
RLock lock = redisson.getLock("myLock");
try {
    lock.lock(); // 自动续期
    // 业务逻辑
} finally {
    lock.unlock();
}
```

---

### 6.8 Redis 数据结构及使用场景

**核心答案：**
Redis 提供 String、List、Hash、Set、ZSet 五种基础结构，以及 HyperLogLog、Bitmap、Stream 等高级结构，每种结构有特定使用场景。

**关键考点：**

| 数据结构 | 底层实现 | 典型场景 |
|---------|---------|---------|
| String | SDS | 缓存、计数器、分布式锁 |
| List | ziplist/quicklist | 消息队列、最新列表 |
| Hash | ziplist/hashtable | 对象存储、购物车 |
| Set | intset/hashtable | 标签、共同好友、抽奖 |
| ZSet | ziplist/skiplist | 排行榜、延迟队列 |
| HyperLogLog | - | UV 统计（有误差） |
| Bitmap | - | 签到、在线状态 |
| Stream | - | 消息队列（支持消费组） |

---

### 6.9 Redis 与 MySQL 数据一致性

**核心答案：**
缓存与数据库一致性核心策略：**Cache Aside Pattern（旁路缓存）**，即读时先查缓存再查库，写时先更新数据库再删除缓存（而非更新缓存）。

**关键考点：**
- 为什么删除缓存而不是更新缓存？更新缓存可能造成并发写的脏数据
- 为什么先更新 DB 再删缓存？先删缓存可能导致读请求拿到旧数据并重新写入缓存
- 延迟双删：先删缓存 -> 更新 DB -> 延迟一段时间再删一次缓存（防止并发读写不一致）
- Canal 方案：监听 MySQL binlog，异步更新缓存，最终一致性
- 强一致场景：分布式锁 + 串行化，但性能差

---

### 6.10 Redis 热点 Key 与 BigKey 问题

**核心答案：**
热点 Key 指访问频率极高的 key，可能导致单节点压力过大；BigKey 指 value 过大的 key，影响性能和内存。

**关键考点：**
- **热点 Key 解决**：
  - 本地缓存（Caffeine/GuavaCache）+ Redis 二级缓存
  - 热点 key 复制到多个节点（key加后缀分散）
  - 限流保护
- **BigKey 危害**：序列化/反序列化慢、阻塞其他命令、网络传输慢、主从同步延迟
- BigKey 发现：redis-cli --bigkeys / SCAN + OBJECT ENCODING
- BigKey 删除：不能直接 DEL，应分批删除（HSCAN/SSCAN/ZSCAN）或 UNLINK 异步删除
- 规范：String < 10KB，集合元素 < 5000


---

## 第七章 Spring 全家桶

### 7.1 Spring IoC 容器原理

**核心答案：**
IoC（控制反转）是将对象的创建和依赖关系的管理交给 Spring 容器，而不是由对象自己管理。Spring 通过 BeanFactory/ApplicationContext 容器，基于配置（XML/注解/Java Config）实例化和装配 Bean。

**关键考点：**
- BeanFactory：基础容器，懒加载 Bean
- ApplicationContext：BeanFactory 子接口，预加载单例 Bean，支持事件发布、国际化、AOP 等
- 依赖注入方式：构造器注入（推荐）、Setter 注入、字段注入（@Autowired）
- @Autowired 按类型注入，@Resource 按名称注入
- 循环依赖：Spring 三级缓存解决（见 7.3）

```
BeanDefinition -> BeanFactory -> getBean() -> Bean 实例
         (配置解析)    (注册)      (创建/注入)
```

---

### 7.2 Spring Bean 生命周期

**核心答案：**
Bean 生命周期从实例化到销毁共经历多个阶段，各阶段均有扩展点供自定义逻辑。

**关键考点：**
完整生命周期流程：

```
1. 实例化 (Instantiation) - 调用构造方法
2. 属性填充 (Populate) - 依赖注入
3. Aware 接口回调 - setBeanName/setBeanFactory/setApplicationContext
4. BeanPostProcessor#postProcessBeforeInitialization
5. InitializingBean#afterPropertiesSet / @PostConstruct
6. 自定义 init-method
7. BeanPostProcessor#postProcessAfterInitialization
8. Bean 就绪，放入容器
9. 使用 Bean
10. DisposableBean#destroy / @PreDestroy / 自定义 destroy-method
```

- @PostConstruct 和 @PreDestroy 是 JSR-250 注解，Spring 通过 CommonAnnotationBeanPostProcessor 处理
- BeanPostProcessor 是 Spring AOP 的实现基础

---

### 7.3 Spring 循环依赖与三级缓存

**核心答案：**
Spring 通过三级缓存解决单例 Bean 的 setter 循环依赖。构造器循环依赖和原型（prototype）Bean 的循环依赖无法解决。

**关键考点：**
三级缓存：
- **一级缓存** singletonObjects：完整的 Bean（初始化完毕）
- **二级缓存** earlySingletonObjects：早期暴露的 Bean（已实例化，未完成属性填充）
- **三级缓存** singletonFactories：ObjectFactory，用于生成早期 Bean 引用（支持 AOP）

```
A 依赖 B，B 依赖 A：
1. 创建 A -> 放入三级缓存
2. A 注入 B -> 创建 B -> 放入三级缓存
3. B 注入 A -> 从三级缓存取出 A 的 ObjectFactory -> 生成早期 A -> 放二级缓存
4. B 完成初始化 -> 放一级缓存
5. A 完成初始化 -> 放一级缓存
```

为什么需要三级缓存？若 A 需要被 AOP 代理，三级缓存中的 ObjectFactory 在此时生成代理对象，保证注入的是代理对象。

---

### 7.4 Spring AOP 原理

**核心答案：**
AOP（面向切面编程）通过动态代理在不修改源代码的情况下增强方法行为。Spring AOP 默认使用 JDK 动态代理（接口）或 CGLIB 代理（无接口/类）。

**关键考点：**
- JDK 动态代理：实现 InvocationHandler，要求目标类实现接口
- CGLIB 代理：字节码增强，继承目标类，不需要接口（final 类不可用）
- Spring Boot 2.x 默认优先使用 CGLIB
- AOP 核心概念：Aspect（切面）、JoinPoint（连接点）、Pointcut（切入点）、Advice（通知）
- 通知类型：@Before、@After、@Around、@AfterReturning、@AfterThrowing
- AOP 失效场景：同类内部方法调用（不走代理）、final 方法、private 方法

```java
@Around("execution(* com.example.service.*.*(..))")
public Object around(ProceedingJoinPoint pjp) throws Throwable {
    // 前置逻辑
    Object result = pjp.proceed();
    // 后置逻辑
    return result;
}
```

---

### 7.5 Spring 事务管理

**核心答案：**
Spring 事务通过 AOP 代理实现，底层使用 PlatformTransactionManager 接口统一管理事务，支持声明式（@Transactional）和编程式两种方式。

**关键考点：**
- 7 种传播行为（Propagation）：
  - REQUIRED（默认）：有事务则加入，没有则新建
  - REQUIRES_NEW：总是新建事务，挂起外层事务
  - NESTED：嵌套事务（保存点 savepoint）
  - SUPPORTS / NOT_SUPPORTED / NEVER / MANDATORY
- 4 种隔离级别：同 MySQL（READ_UNCOMMITTED/READ_COMMITTED/REPEATABLE_READ/SERIALIZABLE）
- **事务失效场景**：
  1. 同类内部方法调用（不走代理）
  2. 方法非 public
  3. 异常被 catch 未抛出
  4. 异常类型不匹配（默认只回滚 RuntimeException）
  5. 数据库不支持事务（MyISAM）
  6. Bean 未被 Spring 管理

---

### 7.6 Spring MVC 请求处理流程

**核心答案：**
Spring MVC 基于前端控制器模式（DispatcherServlet），请求经过一系列组件处理后返回响应。

**关键考点：**
```
1. 请求到达 DispatcherServlet
2. HandlerMapping 查找对应 Handler（Controller方法）
3. HandlerAdapter 适配执行 Handler
4. Handler 返回 ModelAndView
5. ViewResolver 解析视图名为具体视图
6. View 渲染，返回响应
```

- HandlerInterceptor：前置/后置/完成后拦截
- @RequestBody/@ResponseBody：JSON 序列化由 HttpMessageConverter 处理
- 常用注解：@Controller、@RestController、@RequestMapping、@PathVariable、@RequestParam

---

### 7.7 Spring Boot 自动配置原理

**核心答案：**
Spring Boot 通过 @SpringBootApplication 中的 @EnableAutoConfiguration，利用 SPI 机制加载 META-INF/spring.factories（Spring Boot 3.x 改为 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports）中的自动配置类，配合条件注解（@ConditionalOnClass 等）按需生效。

**关键考点：**
- @SpringBootApplication = @SpringBootConfiguration + @EnableAutoConfiguration + @ComponentScan
- @EnableAutoConfiguration 通过 AutoConfigurationImportSelector 加载配置类
- 条件注解：@ConditionalOnClass、@ConditionalOnMissingBean、@ConditionalOnProperty
- spring.factories 中 key 为 EnableAutoConfiguration，value 为自动配置类全限定名
- 自定义 Starter：创建自动配置类 + spring.factories 注册

---

### 7.8 Spring Cloud 核心组件

**核心答案：**
Spring Cloud 提供微服务开发的一站式解决方案，核心组件包括服务注册发现、负载均衡、熔断降级、网关、配置中心等。

**关键考点：**

| 组件功能 | Netflix 方案 | Alibaba 方案 |
|---------|------------|------------|
| 服务注册发现 | Eureka | Nacos |
| 负载均衡 | Ribbon | LoadBalancer |
| 熔断降级 | Hystrix | Sentinel |
| 配置中心 | Config | Nacos Config |
| 网关 | Zuul | Gateway |
| RPC调用 | Feign | Dubbo |

- Nacos 同时支持 AP（服务发现）和 CP（配置管理）模式
- Sentinel 基于滑动窗口统计 QPS，支持流控、熔断、热点、系统规则
- Spring Cloud Gateway 基于 WebFlux 响应式编程，比 Zuul 性能更好


---

## 第八章 MyBatis

### 8.1 MyBatis 执行流程

**核心答案：**
MyBatis 通过 SqlSessionFactory 创建 SqlSession，SqlSession 执行 SQL 语句，底层通过 Executor 和 StatementHandler 与数据库交互。

**关键考点：**
```
1. 读取配置文件 (mybatis-config.xml + Mapper XML)
2. SqlSessionFactoryBuilder -> SqlSessionFactory
3. SqlSessionFactory.openSession() -> SqlSession
4. SqlSession.getMapper(XxxMapper.class) -> JDK 动态代理 MapperProxy
5. 调用 Mapper 方法 -> MapperProxy.invoke()
6. 找到对应 MappedStatement
7. Executor 执行（检查二级缓存 -> 一级缓存 -> 数据库）
8. StatementHandler 预处理 SQL
9. ParameterHandler 设置参数
10. ResultSetHandler 处理结果集
```

- SqlSession 不是线程安全的，应在方法内创建和关闭
- SqlSessionFactory 是线程安全的，全局单例

---

### 8.2 MyBatis 一级缓存与二级缓存

**核心答案：**
- **一级缓存**：SqlSession 级别，默认开启，同一 SqlSession 中相同查询复用结果
- **二级缓存**：Mapper（namespace）级别，需显式开启，跨 SqlSession 共享

**关键考点：**
- 一级缓存失效：执行 update/insert/delete、手动清空（clearCache）、SqlSession 关闭
- 二级缓存开启：mybatis-config.xml 中 cacheEnabled=true + Mapper 中添加 <cache/>
- 二级缓存数据先存入一级缓存，SqlSession 关闭后才刷入二级缓存
- 二级缓存要求对象可序列化
- 多表查询避免使用二级缓存（脏读风险）
- 生产环境推荐使用 Redis 替代 MyBatis 二级缓存

---

### 8.3 MyBatis 插件（Plugin）机制

**核心答案：**
MyBatis 插件基于责任链模式和 JDK 动态代理，拦截 Executor、StatementHandler、ParameterHandler、ResultSetHandler 四大对象的指定方法。

**关键考点：**
- 实现 Interceptor 接口，添加 @Intercepts/@Signature 注解
- 插件执行顺序：配置顺序的逆序（后配置先执行）
- 常见插件应用：分页（PageHelper）、SQL 性能监控、数据权限过滤
- @Signature 指定要拦截的类、方法、参数类型

```java
@Intercepts({@Signature(
    type = Executor.class,
    method = "query",
    args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}
)})
public class MyPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        // 前置处理
        Object result = invocation.proceed();
        // 后置处理
        return result;
    }
}
```

---

### 8.4 MyBatis 动态 SQL

**核心答案：**
MyBatis 提供 <if>、<choose>、<where>、<set>、<foreach>、<trim> 等动态 SQL 标签，支持根据条件动态拼接 SQL。

**关键考点：**
- <if>：条件判断
- <where>：自动处理 AND/OR 前缀，忽略多余的 AND
- <set>：用于 UPDATE，自动处理末尾逗号
- <foreach>：遍历集合，用于 IN 查询或批量插入
- <choose>/<when>/<otherwise>：类似 switch-case
- #{} 和 ${} 区别：#{} 预编译防 SQL 注入；${} 直接字符串替换，有注入风险

```xml
<select id="findUsers" resultType="User">
  SELECT * FROM user
  <where>
    <if test="name != null">AND name = #{name}</if>
    <if test="age != null">AND age = #{age}</if>
  </where>
</select>
```

---

### 8.5 MyBatis 与 Spring 整合原理

**核心答案：**
MyBatis-Spring 通过 SqlSessionFactoryBean 创建 SqlSessionFactory，通过 MapperScannerConfigurer 扫描 Mapper 接口并注册为 Spring Bean，实际类型为 MapperFactoryBean。

**关键考点：**
- MapperFactoryBean 实现了 FactoryBean，getObject() 返回 Mapper 代理对象
- SqlSession 使用 SqlSessionTemplate，线程安全（每次操作使用 Spring 事务中的 SqlSession）
- @MapperScan 等价于配置 MapperScannerConfigurer
- Spring 事务与 MyBatis 整合：MyBatis 使用 SpringManagedTransaction，事务由 Spring 管理

---

## 第九章 消息队列

### 9.1 消息队列的作用与选型

**核心答案：**
消息队列主要用于**解耦、异步、削峰**三大场景。常见 MQ 选型对比：

| 特性 | RabbitMQ | RocketMQ | Kafka |
|-----|---------|---------|-------|
| 语言 | Erlang | Java | Scala/Java |
| 吞吐量 | 万级 | 十万级 | 百万级 |
| 延迟 | 微秒级 | 毫秒级 | 毫秒级 |
| 可靠性 | 高 | 高 | 高 |
| 事务消息 | 不支持 | 支持 | 支持 |
| 延迟消息 | 插件支持 | 原生支持 | 不支持 |
| 适用场景 | 金融、复杂路由 | 电商、金融 | 日志、大数据 |

---

### 9.2 Kafka 核心架构

**核心答案：**
Kafka 是分布式流处理平台，核心概念：Producer、Consumer、Broker、Topic、Partition、Consumer Group。

**关键考点：**
- Topic 分为多个 Partition，每个 Partition 是有序的 append-only 日志
- 每个 Partition 有一个 Leader 和多个 Follower（ISR 集合）
- Consumer Group：同一组内每个 Partition 只被一个 Consumer 消费（实现负载均衡）
- Offset 由 Consumer 维护（存在 __consumer_offsets topic）
- Kafka 高吞吐原因：顺序写磁盘、零拷贝（sendfile）、批量发送、PageCache

```
Producer -> [Topic-Partition0 (Leader)] <- Follower1
                                        <- Follower2
         -> [Topic-Partition1 (Leader)] <- Follower1
Consumer Group A: Consumer1 消费 P0，Consumer2 消费 P1
```

---

### 9.3 Kafka 消息可靠性保证

**核心答案：**
Kafka 通过 Producer acks 配置、ISR 机制、Consumer 手动提交 offset 三方面保证消息可靠性。

**关键考点：**
- Producer acks：
  - acks=0：不等待确认（最快，可能丢失）
  - acks=1：等待 Leader 写入（可能丢失，Leader 未同步就宕机）
  - acks=-1/all：等待 ISR 所有副本写入（最安全）
- ISR（In-Sync Replicas）：与 Leader 保持同步的副本集合
- min.insync.replicas：最小同步副本数，与 acks=all 配合防止数据丢失
- Consumer 手动提交 offset：处理完业务逻辑后再提交，防止丢消息
- 幂等性 Producer（enable.idempotence=true）：防止重试导致重复消息

---

### 9.4 RocketMQ 事务消息

**核心答案：**
RocketMQ 事务消息通过两阶段提交实现分布式事务，确保消息发送与本地事务的原子性。

**关键考点：**
```
1. Producer 发送半消息（Half Message）到 RocketMQ
2. RocketMQ 回传发送成功 ACK
3. Producer 执行本地事务
4. 本地事务成功 -> 发送 Commit -> 消息对 Consumer 可见
   本地事务失败 -> 发送 Rollback -> 删除半消息
5. 若 Producer 宕机，RocketMQ 定时回查本地事务状态
```

- 半消息对消费者不可见，存储在特殊 Topic（RMQ_SYS_TRANS_HALF_TOPIC）
- 回查接口：实现 TransactionListener 的 checkLocalTransaction 方法
- 适用场景：下单后扣减库存、支付后更新订单状态

---

### 9.5 消息幂等性处理

**核心答案：**
消息队列至少一次（At Least Once）语义可能导致重复消费，需要在消费端实现幂等处理。

**关键考点：**
- 重复消费原因：Consumer 处理完未提交 offset 就宕机；网络超时重试
- 幂等实现方案：
  1. **数据库唯一约束**：消息 ID 作为唯一键，重复插入报错捕获
  2. **Redis 去重**：消费前 SETNX 消息 ID，成功才处理
  3. **状态机**：业务状态只能单向流转，重复处理同一状态无效果
  4. **乐观锁**：UPDATE table SET status=1 WHERE id=? AND status=0
- 消息 ID 设计：全局唯一，可用雪花算法或 UUID
- 幂等与去重的区别：幂等是多次执行结果一致；去重是物理过滤重复消息

---

### 9.6 消息顺序性保证

**核心答案：**
要保证消息全局有序极难且性能差；通常只需保证局部有序（同一业务的消息有序）。

**关键考点：**
- Kafka 同一 Partition 内消息有序；将同一业务 key 路由到同一 Partition
- RocketMQ 同一 MessageQueue 内有序；发送时使用 MessageQueueSelector 选择固定队列
- 消费端单线程消费对应队列，或使用局部顺序消费器
- 顺序消费代价：并发度降低，单个队列积压影响整个业务
- 全局有序：只用一个 Partition/Queue（吞吐量极低，生产基本不用）

---

### 9.7 消息积压处理

**核心答案：**
消息积压通常由消费能力不足或生产速度突增导致。处理方案：扩容消费者、提升单消费者处理速度、临时扩容。

**关键考点：**
- 紧急处理：
  1. 增加 Consumer 实例数量（不超过 Partition 数）
  2. 消费端开多线程并发处理（注意幂等）
  3. 临时新建 Topic（分区数 10x），迁移积压消息，用更多 Consumer 消费
- 根本原因排查：消费端性能瓶颈（DB 慢查、外部接口慢）、消费端异常频繁重试
- 预防：合理设置 Consumer 数量；监控消费延迟（lag）；设置告警阈值

---

### 9.8 死信队列（DLQ）

**核心答案：**
死信队列（Dead Letter Queue）存储消费失败达到最大重试次数的消息，避免"毒消息"阻塞正常消息处理。

**关键考点：**
- RocketMQ DLQ：重试 16 次后进入死信队列（%DLQ%ConsumerGroup）
- RabbitMQ DLQ：消息 TTL 到期、被 nack 且 requeue=false、队列满时进入死信交换机
- 死信处理：监控死信队列，人工排查原因后重新投递或丢弃
- 消息重试策略：指数退避（1s, 5s, 10s, 30s...）避免立即重试加重下游压力

---

### 9.9 Kafka Rebalance 机制

**核心答案：**
Rebalance 是 Consumer Group 内的 Partition 重新分配过程，由 Group Coordinator 协调，在 Consumer 加入/离开或 Topic Partition 变化时触发。

**关键考点：**
- 触发时机：Consumer 加入/退出 Group、心跳超时、Topic Partition 数变化
- Rebalance 过程：
  1. 所有 Consumer 向 Group Coordinator 发送 JoinGroup 请求
  2. Coordinator 选出 Group Leader（第一个发送 JoinGroup 的）
  3. Leader 执行分区分配策略（RangeAssignor/RoundRobinAssignor/StickyAssignor）
  4. Leader 将分配结果通过 SyncGroup 告知 Coordinator
  5. Coordinator 将分配结果下发给所有 Consumer
- Rebalance 期间消费暂停（Stop the World）
- 减少 Rebalance：合理设置 session.timeout.ms 和 heartbeat.interval.ms；使用 StickyAssignor 减少迁移


---

## 第十章 分布式

### 10.1 CAP 理论与 BASE 理论

**核心答案：**
- **CAP**：分布式系统不能同时满足一致性（C）、可用性（A）、分区容错性（P），分区容错性必须保证，因此只能在 CP 和 AP 之间选择
- **BASE**：是 CAP 的工程实践，强调基本可用（BA）、软状态（S）、最终一致性（E）

**关键考点：**
- C（Consistency）：所有节点同一时刻看到相同数据
- A（Availability）：每个请求都能收到响应（不保证最新数据）
- P（Partition Tolerance）：网络分区时系统仍可运行
- CP 系统：ZooKeeper（强一致，分区时可能不可用）
- AP 系统：Eureka、Cassandra（可用但可能读到旧数据）
- Nacos 默认 AP，配置中心用 CP 模式
- BASE 的最终一致性实现：消息队列、定时对账、补偿机制

---

### 10.2 分布式事务

**核心答案：**
分布式事务保证跨服务/数据库操作的原子性。主要方案：2PC、3PC、TCC、Saga、本地消息表、MQ 事务消息。

**关键考点：**

| 方案 | 原理 | 优点 | 缺点 |
|-----|-----|-----|-----|
| 2PC | 协调者两阶段提交 | 强一致 | 同步阻塞、协调者单点 |
| 3PC | 三阶段提交+超时 | 减少阻塞 | 实现复杂、仍有数据不一致 |
| TCC | Try-Confirm-Cancel | 性能好 | 业务侵入大 |
| Saga | 长事务拆分+补偿 | 无锁、高可用 | 中间态可见 |
| 本地消息表 | 消息与业务在同一 DB 事务 | 简单 | 依赖消息表 |
| MQ 事务 | RocketMQ 半消息 | 高可用 | 消费端需幂等 |

- Seata 框架：支持 AT（自动）、TCC、Saga、XA 模式
- AT 模式：自动生成 undo_log，通过全局锁保证隔离性

---

### 10.3 Raft 共识算法

**核心答案：**
Raft 是一种易于理解的分布式共识算法，用于保证多节点日志一致性。核心分为 Leader 选举、日志复制、安全性三部分。

**关键考点：**
- 节点角色：Leader（领导者）、Follower（跟随者）、Candidate（候选者）
- **Leader 选举**：Follower 超时未收到心跳 -> 转为 Candidate -> 发起投票 -> 获得多数票成为 Leader
- 任期（Term）：单调递增，每次选举新任期
- **日志复制**：Leader 收到请求 -> 写入本地日志 -> 广播给 Follower -> 多数确认后 Commit -> 返回客户端
- 安全性：只有拥有最新日志的节点才能成为 Leader
- 应用：etcd、TiKV、CockroachDB、Consul

```
正常状态：Follower <-- 心跳 -- Leader
Leader 宕机：Follower 超时 -> Candidate -> 发起选举
         Candidate 获得 > N/2 票 -> 成为新 Leader
```

---

### 10.4 分布式 ID 生成

**核心答案：**
分布式 ID 需满足全局唯一、趋势递增、高性能、可用性高。常见方案：UUID、数据库自增（分段）、Redis 自增、雪花算法（Snowflake）、号段模式。

**关键考点：**
- **UUID**：简单，无序，不适合做主键（B+ 树插入性能差）
- **数据库自增**：单点瓶颈，可用分库（多 DB 设置不同起始值和步长）
- **Redis incr**：高性能，需持久化防止重置，依赖 Redis 可用性
- **雪花算法（64位）**：
  ```
  1位符号 + 41位时间戳 + 10位机器ID + 12位序列号
  每毫秒最多生成 4096 个 ID
  ```
  问题：时钟回拨导致重复 ID（解决：抛异常/等待/备用位）
- **美团 Leaf**：支持号段模式和雪花模式，解决了时钟回拨问题
- **百度 UidGenerator**：环形队列预生成

---

### 10.5 分布式限流

**核心答案：**
限流算法：计数器、滑动窗口、漏桶、令牌桶。分布式环境下需用 Redis 等共享存储实现。

**关键考点：**

| 算法 | 原理 | 优点 | 缺点 |
|-----|-----|-----|-----|
| 固定窗口计数 | 窗口内计数，超限拒绝 | 简单 | 临界点突刺 |
| 滑动窗口 | 维护时间窗口内请求列表 | 平滑 | 内存占用高 |
| 漏桶 | 固定速率流出，队列缓冲 | 输出平稳 | 不能处理突发 |
| 令牌桶 | 固定速率放令牌，有令牌才处理 | 允许突发 | 实现稍复杂 |

- Redis + Lua 实现令牌桶/滑动窗口限流（原子性）
- Sentinel 基于滑动窗口统计 QPS/线程数
- Nginx limit_req_zone：漏桶算法

---

### 10.6 服务熔断与降级

**核心答案：**
- **熔断（Circuit Breaker）**：当下游服务失败率超阈值时，自动断开调用，避免级联故障
- **降级（Fallback）**：熔断或超时时返回默认值/缓存值，保证基本可用

**关键考点：**
熔断器三态：
```
关闭(Closed) -> [失败率超阈值] -> 打开(Open) -> [超冷却时间] -> 半开(Half-Open)
半开 -> [探测请求成功] -> 关闭
半开 -> [探测请求失败] -> 打开
```
- Hystrix 基于命令模式和线程池/信号量隔离
- Sentinel 基于滑动窗口，实时统计异常比例和响应时间
- 服务降级策略：返回默认值、读缓存、限制功能、直接拒绝

---

### 10.7 分布式配置中心

**核心答案：**
配置中心将配置从代码中分离，支持动态刷新、多环境管理、版本控制。常见方案：Nacos、Apollo、Spring Cloud Config。

**关键考点：**
- **Nacos 配置中心**：
  - 客户端长轮询（Long Polling）监听配置变更，默认 30s 超时
  - 服务端配置变更时主动推送（Hold 住请求后立即返回）
  - 支持配置分组（Group）和命名空间（Namespace）隔离
  - @RefreshScope + @Value 实现 Bean 属性动态刷新
- **Apollo**：携程开源，支持灰度发布、权限管理、操作审计
- 配置加密：Jasypt/ACMS 对敏感配置加密存储

---

### 10.8 分布式链路追踪

**核心答案：**
链路追踪记录请求在分布式系统中的完整调用链路，用于性能分析和故障排查。核心概念：Trace、Span、Context Propagation。

**关键考点：**
- Trace：一次完整请求的调用链，由多个 Span 组成
- Span：一次服务调用，包含操作名、开始/结束时间、标签、日志
- Context 传播：通过 HTTP Header（B3/W3C TraceContext）或 MQ Header 传递 traceId
- 常见方案：Zipkin、Jaeger、SkyWalking（字节码增强，无侵入）
- 采样率：全量采样影响性能，通常采用概率采样（1%~10%）或自适应采样

---

### 10.9 微服务拆分原则

**核心答案：**
微服务拆分没有唯一标准，遵循高内聚低耦合原则，参考 DDD 领域驱动设计的限界上下文划分。

**关键考点：**
- 拆分原则：单一职责、按业务领域、按数据独立性、按团队组织（Conway 定律）
- 拆分粒度：不要过细（运维复杂度指数增加），也不要过粗（失去微服务优势）
- 数据库拆分：每个服务独立 DB，禁止跨服务直接查询 DB
- 跨服务数据查询：API 聚合、CQRS（命令查询责任分离）、数据冗余
- 服务间通信：同步（Feign/gRPC）vs 异步（MQ）

---

### 10.10 幂等设计

**核心答案：**
幂等是指同一操作多次执行与执行一次结果相同。在分布式系统中，网络重试、消息重复消费等场景需要幂等保证。

**关键考点：**
- HTTP 方法幂等性：GET/PUT/DELETE 天然幂等；POST 不幂等
- 幂等实现方案：
  1. **唯一索引**：数据库唯一约束防止重复插入
  2. **Token 机制**：请求前获取 token，提交时校验并删除（防重复提交）
  3. **Redis SETNX**：分布式去重，key 为业务唯一标识
  4. **状态机**：只允许合法的状态转换
  5. **乐观锁版本号**：UPDATE ... WHERE version = #{version}
- 接口幂等与业务幂等的区别


---

## 第十一章 网络协议

### 11.1 TCP 三次握手与四次挥手

**核心答案：**
TCP 通过三次握手建立连接，四次挥手断开连接，保证连接建立时双方收发能力均已确认，断开时双方数据均已传输完毕。

**关键考点：**
三次握手：
```
Client          Server
  | ---SYN(seq=x)---> |   第一次：客户端发 SYN
  | <--SYN+ACK(seq=y,ack=x+1)-- |   第二次：服务端发 SYN+ACK
  | ---ACK(ack=y+1)--> |   第三次：客户端发 ACK
```
四次挥手：
```
Client          Server
  | ---FIN(seq=u)---> |   第一次：客户端发 FIN（我发完了）
  | <---ACK(ack=u+1)-- |   第二次：服务端确认（我知道了）
  | <---FIN(seq=v)---- |   第三次：服务端发 FIN（我也发完了）
  | ---ACK(ack=v+1)--> |   第四次：客户端确认，等待 2MSL
```
- 为什么是三次握手？两次无法确认客户端收发正常；防止历史连接请求干扰
- 为什么四次挥手？TCP 全双工，两个方向分别关闭
- TIME_WAIT（2MSL）：保证最后一个 ACK 能到达对方；让旧连接的报文在网络中消失
- SYN Flood 攻击：大量 SYN 半连接耗尽服务器资源；防御：SYN Cookie

---

### 11.2 TCP 可靠性机制

**核心答案：**
TCP 通过序号/确认应答、超时重传、流量控制（滑动窗口）、拥塞控制四大机制保证可靠传输。

**关键考点：**
- **序号 + 确认应答**：接收方返回 ACK 确认收到，发送方超时未收到 ACK 则重传
- **滑动窗口**：发送方可以连续发送多个报文段，窗口大小由接收方 rwnd 控制
- **拥塞控制**四个阶段：
  - 慢开始：cwnd 从 1 指数增长
  - 拥塞避免：cwnd 超过 ssthresh 后线性增长
  - 快重传：收到 3 个重复 ACK 立即重传（不等超时）
  - 快恢复：快重传后 ssthresh=cwnd/2，cwnd=ssthresh（不归 1）
- **Nagle 算法**：合并小包，减少网络报文数量（可禁用 TCP_NODELAY）
- **粘包/拆包**：TCP 是字节流协议，需应用层自定义消息边界（长度字段/分隔符）

---

### 11.3 HTTP/1.0、HTTP/1.1、HTTP/2、HTTP/3 对比

**核心答案：**
HTTP 协议持续演进，主要解决性能问题：从短连接到长连接、从串行到多路复用、从文本到二进制、从 TCP 到 QUIC（UDP）。

**关键考点：**

| 版本 | 核心特性 |
|-----|---------|
| HTTP/1.0 | 短连接，每次请求建立新 TCP 连接 |
| HTTP/1.1 | 长连接（Keep-Alive），管道化（但有队头阻塞） |
| HTTP/2 | 二进制分帧，多路复用，头部压缩（HPACK），服务器推送 |
| HTTP/3 | 基于 QUIC（UDP），解决 TCP 队头阻塞，0-RTT 连接 |

- HTTP/1.1 队头阻塞：同一连接上的请求必须串行等待前一个响应
- HTTP/2 多路复用：同一 TCP 连接上并发多个 Stream，Stream 互不阻塞
- HTTP/2 仍有 TCP 层队头阻塞；HTTP/3 用 QUIC 彻底解决
- HPACK 头部压缩：静态表 + 动态表 + Huffman 编码

---

### 11.4 HTTPS 握手过程

**核心答案：**
HTTPS = HTTP + TLS，TLS 握手在 TCP 握手后进行，协商加密参数、验证身份、交换密钥，之后用对称密钥加密通信。

**关键考点：**
TLS 1.2 握手（4次往返）：
```
1. Client Hello：支持的 TLS 版本、加密套件、随机数(Random1)
2. Server Hello：选定 TLS 版本和加密套件、随机数(Random2)、证书
3. Client：验证证书 -> 生成预主密钥(Pre-Master) -> 用服务器公钥加密发送
4. 双方用 Random1+Random2+Pre-Master 生成会话密钥
5. Client/Server 发送 Finished（用会话密钥加密）
```
TLS 1.3 优化：
- 握手减少为 1-RTT（甚至 0-RTT 会话恢复）
- 废弃不安全算法，只支持 ECDHE 等前向安全算法
- 证书验证链：叶证书 -> 中间 CA -> 根 CA

---

### 11.5 HTTP 常见状态码

**核心答案：**
HTTP 状态码分 5 类，每类有明确语义，掌握常见状态码及含义。

**关键考点：**

| 状态码 | 含义 |
|-------|-----|
| 200 OK | 请求成功 |
| 201 Created | 资源创建成功 |
| 204 No Content | 成功但无响应体 |
| 301 Moved Permanently | 永久重定向 |
| 302 Found | 临时重定向 |
| 304 Not Modified | 资源未修改，使用缓存 |
| 400 Bad Request | 请求参数错误 |
| 401 Unauthorized | 未认证 |
| 403 Forbidden | 无权限 |
| 404 Not Found | 资源不存在 |
| 405 Method Not Allowed | 方法不允许 |
| 429 Too Many Requests | 限流 |
| 500 Internal Server Error | 服务器内部错误 |
| 502 Bad Gateway | 网关错误（上游服务异常） |
| 503 Service Unavailable | 服务不可用 |
| 504 Gateway Timeout | 网关超时 |

---

### 11.6 HTTP 缓存机制

**核心答案：**
HTTP 缓存分强缓存和协商缓存。强缓存期间直接使用本地缓存；强缓存失效后通过协商缓存验证资源是否更新。

**关键考点：**
- **强缓存**：
  - Expires（HTTP/1.0）：绝对时间，依赖客户端时钟
  - Cache-Control（HTTP/1.1）：max-age=秒数，优先级高于 Expires
  - 常见指令：no-cache（需协商）、no-store（不缓存）、public/private
- **协商缓存**：
  - Last-Modified / If-Modified-Since：基于文件修改时间（精度秒级）
  - ETag / If-None-Match：基于内容 hash（精度更高，优先级更高）
  - 服务器返回 304 表示资源未变，使用缓存；200 表示已更新，返回新内容
- 缓存策略：HTML 不缓存或 no-cache；JS/CSS 加内容 hash 后缀 + 长期强缓存

---

### 11.7 从输入 URL 到页面展示的全过程

**核心答案：**
这是一道综合考题，涵盖 DNS、TCP、HTTP、渲染等全栈流程。

**关键考点：**
```
1. URL 解析（协议、域名、路径、参数）
2. DNS 解析（浏览器缓存 -> OS缓存 -> 本地DNS服务器 -> 根DNS -> 顶级DNS -> 权威DNS）
3. TCP 三次握手建立连接（HTTPS 还需 TLS 握手）
4. 发送 HTTP 请求（请求行 + 请求头 + 请求体）
5. 服务器处理请求，返回 HTTP 响应
6. 浏览器接收响应
7. 解析 HTML，构建 DOM 树
8. 解析 CSS，构建 CSSOM 树
9. 合并生成渲染树（Render Tree）
10. 布局（Layout/Reflow）：计算元素位置和大小
11. 绘制（Paint）：像素填充
12. 合成（Composite）：GPU 合成图层
13. TCP 四次挥手（或保持长连接）
```

---

### 11.8 WebSocket 与 HTTP 长轮询对比

**核心答案：**
WebSocket 是全双工通信协议，基于 HTTP 升级握手，建立后双向实时通信；长轮询是 HTTP 的模拟服务器推送，需反复建立连接。

**关键考点：**
- WebSocket 握手：HTTP 请求携带 Upgrade: websocket，服务器返回 101 Switching Protocols
- WebSocket 特点：持久连接、全双工、低延迟、支持二进制
- 长轮询：客户端发请求，服务器有数据时才返回，客户端收到后立即再发请求
- SSE（Server-Sent Events）：服务器向客户端单向推送，基于 HTTP，自动重连
- 选型：实时聊天/游戏用 WebSocket；简单通知推送用 SSE 或长轮询

---

### 11.9 DNS 解析过程与优化

**核心答案：**
DNS 将域名解析为 IP 地址，采用分层递归解析架构。DNS 解析性能直接影响首屏加载时间。

**关键考点：**
DNS 解析层级：
```
浏览器缓存 -> OS hosts 文件 -> 本地 DNS 服务器
-> 根 DNS 服务器（.）-> 顶级域 DNS（.com）-> 权威 DNS（example.com）
```
- 递归查询：客户端向本地 DNS 查询，本地 DNS 负责完整解析
- 迭代查询：本地 DNS 分别向根、顶级域、权威 DNS 查询
- TTL：DNS 记录的缓存时间，降低 TTL 有利于快速切换 IP
- DNS 预解析：`<link rel="dns-prefetch" href="//example.com">` 提前解析第三方域名
- HTTPDNS：App 直接向 HTTP 接口查询 DNS，绕过运营商劫持


---

# 第十章：分布式

## 10.1 CAP 定理与 BASE 理论

**核心答案：**
CAP 定理指出分布式系统无法同时满足一致性（Consistency）、可用性（Availability）和分区容错性（Partition Tolerance）三者，只能三选二。BASE 理论是对 CAP 的妥协，强调基本可用（Basically Available）、软状态（Soft State）、最终一致性（Eventually Consistent）。

**关键考点：**
- **CP 系统**：保证一致性和分区容错，牺牲可用性。例：ZooKeeper、HBase、etcd
- **AP 系统**：保证可用性和分区容错，牺牲强一致性。例：Eureka、Cassandra、DynamoDB
- **CA 系统**：保证一致性和可用性，不支持分区容错（单机数据库，不是真正的分布式）
- 实践中 P 必须保证（网络不可靠），所以是 CP vs AP 的选择
- BASE 允许分布式系统在分区故障期间暂时不一致，但最终收敛到一致

| 特性 | CP | AP |
|------|----|----|
| 一致性 | 强一致 | 最终一致 |
| 可用性 | 分区时不可用 | 始终可用 |
| 典型场景 | 金融事务、配置中心 | 购物车、DNS |
| 代表产品 | ZooKeeper | Eureka、Cassandra |

---

## 10.2 分布式事务方案

**核心答案：**
分布式事务是跨服务/跨库操作的事务一致性问题。常见方案有 2PC、3PC、TCC、Saga、本地消息表、Seata。

**关键考点：**

**2PC（两阶段提交）：**
```
阶段1 Prepare：协调者询问各参与者是否可以提交
阶段2 Commit/Rollback：所有参与者 OK 则提交，否则回滚
缺点：同步阻塞、单点故障、数据不一致（部分提交）
```

**TCC（Try-Confirm-Cancel）：**
- Try：预留资源（冻结库存、预扣余额）
- Confirm：确认提交（真正扣减）
- Cancel：释放预留资源
- 优点：无锁、高性能；缺点：业务侵入强，需实现三个接口

**Saga 模式：**
- 将长事务拆分为多个本地事务序列
- 补偿事务（逆向操作）回滚已完成步骤
- 适合微服务场景，但不保证隔离性

**本地消息表：**
```
1. 业务操作 + 插入消息表（同一本地事务）
2. 定时任务扫描消息表，发送 MQ 消息
3. 消费者处理后删除/标记消息
```

**Seata AT 模式（推荐）：**
- 自动解析 SQL，生成 undoLog
- 一阶段直接提交，全局锁保证隔离
- 二阶段提交删除 undoLog，回滚则执行 undoLog

---

## 10.3 分布式锁实现

**核心答案：**
分布式锁用于控制分布式系统中多节点对共享资源的互斥访问。常见实现：Redis（SETNX）、ZooKeeper（临时顺序节点）、数据库（唯一索引）。

**关键考点：**

**Redis 分布式锁（SET NX PX）：**
```java
// 加锁：原子操作 SET key value NX PX timeout
String lockValue = UUID.randomUUID().toString();
Boolean acquired = redis.set(lockKey, lockValue, 
    SetArgs.ifAbsent().px(30000));

// 解锁：Lua 脚本保证原子性（先判断再删除）
String luaScript = 
    "if redis.call('get',KEYS[1]) == ARGV[1] then " +
    "return redis.call('del',KEYS[1]) else return 0 end";
redis.eval(luaScript, lockKey, lockValue);
```

**Redisson 看门狗机制：**
- 默认锁过期时间 30s，每 10s 自动续期
- 避免业务未完成锁已过期的问题
- `RLock lock = redisson.getLock("myLock");`

**ZooKeeper 分布式锁：**
- 创建临时顺序节点，序号最小的获取锁
- 监听前一个节点删除事件
- 天然公平锁，客户端宕机自动释放临时节点

**对比：**
| | Redis | ZooKeeper |
|--|-------|-----------|
| 性能 | 高 | 中 |
| 可靠性 | 较高（集群） | 高（强一致） |
| 死锁 | TTL 自动释放 | 临时节点自动释放 |
| 公平性 | 不公平 | 公平 |

---

## 10.4 分布式 ID 生成方案

**核心答案：**
分布式 ID 需要满足全局唯一、有序（趋势递增）、高性能、高可用的特点。常见方案：UUID、数据库自增、Redis incr、雪花算法（Snowflake）、号段模式。

**关键考点：**

**雪花算法（Snowflake）结构（64位）：**
```
1位(符号位) | 41位(毫秒时间戳) | 10位(机器ID) | 12位(序列号)
  固定为0   |  可用约69年      |  最多1024节点 | 每毫秒4096个ID
```

```java
public class SnowflakeIdWorker {
    private final long workerId;
    private final long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;
    
    // 时间戳左移22位，数据中心左移17位，机器ID左移12位
    public synchronized long nextId() {
        long timestamp = System.currentTimeMillis();
        if (timestamp == lastTimestamp) {
            sequence = (sequence + 1) & 4095;
            if (sequence == 0) timestamp = waitNextMillis(lastTimestamp);
        } else {
            sequence = 0;
        }
        lastTimestamp = timestamp;
        return ((timestamp - EPOCH) << 22) | (datacenterId << 17) 
             | (workerId << 12) | sequence;
    }
}
```

**时钟回拨问题：**
- 百毫秒内：等待时钟追上
- 较长回拨：抛出异常，人工介入
- 美团 Leaf-Snowflake：ZooKeeper 存储各节点时间，启动校验

**号段模式（Leaf-Segment）：**
- DB 分配号段，如每次取 1000 个 ID
- 内存中消费，号段用完再取
- 双 buffer 预加载，避免取号卡顿

---

## 10.5 限流算法

**核心答案：**
限流是保护系统不被过量请求压垮的手段。常见算法：计数器、滑动窗口、漏桶、令牌桶。

**关键考点：**

**固定窗口计数器：**
- 简单，但有临界问题（窗口边界可能通过 2N 请求）

**滑动窗口：**
- 将窗口细分为多个小槽，解决临界问题
- Sentinel、Redis + ZSet 实现

**漏桶算法：**
```
请求 -> 漏桶（固定容量） -> 固定速率流出
优点：平滑输出，绝对限速
缺点：无法应对突发流量（始终固定速率）
```

**令牌桶算法（推荐）：**
```
固定速率向桶中放令牌，桶满则丢弃
请求到来消耗令牌，令牌不足则拒绝/等待
优点：允许一定突发（桶中积累的令牌）
代表：Guava RateLimiter、Spring Cloud Gateway
```

```java
// Guava RateLimiter 示例
RateLimiter limiter = RateLimiter.create(100.0); // 每秒100个令牌
if (limiter.tryAcquire()) {
    // 处理请求
} else {
    // 拒绝请求
}
```

**Sentinel 限流规则：**
- QPS 限流：基于滑动窗口统计
- 并发线程数限流：统计当前处理请求的线程数
- 熔断：异常比例/慢调用比例超阈值自动熔断

---

## 10.6 服务注册与发现原理

**核心答案：**
服务注册发现是微服务架构的基础，服务启动时向注册中心注册自己的地址，消费者通过注册中心查询并调用目标服务。

**关键考点：**

**Nacos vs Eureka：**
| 特性 | Nacos | Eureka |
|------|-------|--------|
| CAP | CP（永久实例）+ AP（临时实例） | AP |
| 健康检查 | 客户端心跳 + 主动健康检查 | 客户端心跳 |
| 推/拉模式 | 推送（UDP）+ 拉取 | 拉取 |
| 配置中心 | 支持 | 不支持 |
| 维护状态 | 活跃维护 | 停更 |

**Nacos 临时/持久实例：**
- 临时实例（默认）：心跳续约，超时自动下线（AP）
- 持久实例：主动探活，不心跳也不删除（CP）

**服务发现流程：**
```
1. 服务启动 -> 注册到 Nacos（HTTP/gRPC）
2. 消费者启动 -> 订阅服务列表
3. Nacos 推送变更（UDP 通知）
4. 消费者本地缓存服务列表
5. 负载均衡选择实例 -> 发起调用
```

---

# 第十一章：网络协议

## 11.1 TCP 三次握手与四次挥手

**核心答案：**
TCP 三次握手建立连接（SYN→SYN+ACK→ACK），四次挥手断开连接（FIN→ACK→FIN→ACK）。三次握手确保双向通信能力；四次挥手因 TCP 半关闭特性需要各方单独关闭。

**关键考点：**

**三次握手：**
```
Client                    Server
  | ---SYN(seq=x)--------> |   第一次：客户端发 SYN，进入 SYN_SENT
  | <--SYN+ACK(ack=x+1)--- |   第二次：服务端发 SYN+ACK，进入 SYN_RCVD
  | ---ACK(ack=y+1)-------> |   第三次：客户端发 ACK，双方进入 ESTABLISHED
```

**为什么需要三次而不是两次？**
- 两次握手服务端无法确认客户端已收到自己的 SYN+ACK
- 防止历史连接（网络中残留的旧SYN包）被误用

**四次挥手：**
```
Client                    Server
  | ---FIN--------------> |   客户端发 FIN，进入 FIN_WAIT_1
  | <--ACK--------------- |   服务端发 ACK，进入 CLOSE_WAIT；客户端进入 FIN_WAIT_2
  | <--FIN--------------- |   服务端数据发完后发 FIN，进入 LAST_ACK
  | ---ACK--------------> |   客户端发 ACK，进入 TIME_WAIT（等待 2MSL）
```

**TIME_WAIT 为什么等待 2MSL？**
- 确保最后一个 ACK 能到达对端（若丢失，服务端会重传 FIN，客户端收到后重发 ACK）
- 让本次连接的报文在网络中消散，避免影响下一个连接

---

## 11.2 HTTP/1.1 vs HTTP/2 vs HTTP/3

**核心答案：**
HTTP 协议经历了从 1.0 的短连接，到 1.1 的持久连接/管道，到 2.0 的多路复用，到 3.0 基于 QUIC 的无队头阻塞的演进。

**关键考点：**

| 特性 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 底层协议 | TCP | TCP | UDP（QUIC） |
| 多路复用 | 不支持（管道化有队头阻塞） | 支持（帧/流） | 支持 |
| 头部压缩 | 无 | HPACK | QPACK |
| 服务端推送 | 不支持 | 支持 | 支持 |
| TLS | 可选 | 强烈推荐 | 内置（QUIC） |
| 队头阻塞 | 应用层+传输层 | 仍有传输层 | 解决 |

**HTTP/2 多路复用原理：**
- 将 HTTP 消息分解为独立的帧（Frame），同一连接上的多个流（Stream）并行传输
- Stream 间互不阻塞，但 TCP 层的丢包还是会影响所有流

**HTTPS 握手流程（TLS 1.3）：**
```
1. Client Hello（支持的密码套件、随机数）
2. Server Hello（选择密码套件、证书、随机数）
3. 客户端验证证书 -> 生成 Pre-Master Secret
4. 双方计算 Session Key
5. 加密通信开始
TLS 1.3 支持 0-RTT 复用会话
```

---

## 11.3 DNS 解析过程

**核心答案：**
DNS 将域名解析为 IP 地址，采用分层递归查询。解析顺序：浏览器缓存 → OS 缓存 → hosts 文件 → 本地 DNS 服务器 → 递归查询根/顶级/权威 DNS。

**关键考点：**
```
1. 浏览器缓存（几秒~几分钟）
2. OS 缓存 + /etc/hosts
3. 本地 DNS 服务器（ISP 提供，也称递归解析器）
4. 根域名服务器（返回顶级域 .com 的地址）
5. 顶级域名服务器（返回 example.com 权威 DNS 地址）
6. 权威域名服务器（返回 www.example.com 的 IP）
7. 本地 DNS 服务器缓存结果，返回给客户端
```

- A 记录：域名 -> IPv4
- AAAA 记录：域名 -> IPv6
- CNAME 记录：域名 -> 另一个域名（别名）
- MX 记录：邮件服务器
- TTL：缓存存活时间

---

## 11.4 TCP 粘包/拆包问题

**核心答案：**
TCP 是流式协议，不保留消息边界。发送方多次 write 可能被合并（粘包），或一次 write 被拆分成多个包（拆包）。需在应用层定义消息边界。

**关键考点：**

**原因：**
- 粘包：发送方数据小，Nagle 算法合并；接收方读取不及时
- 拆包：数据大于 MSS（最大报文段），被拆分传输

**解决方案：**
1. **固定长度**：每次读固定字节数（简单，浪费）
2. **分隔符**：用特殊字符（如 `
`）分隔消息（需转义）
3. **长度前缀**：消息头包含消息体长度（推荐，Netty LengthFieldBasedFrameDecoder）
4. **自定义协议**：如 Dubbo、gRPC 的帧头设计

```
Netty 解决方案：
- LineBasedFrameDecoder：以换行符为分隔
- DelimiterBasedFrameDecoder：自定义分隔符
- FixedLengthFrameDecoder：固定长度
- LengthFieldBasedFrameDecoder：基于长度字段（最常用）
```

---

# 第十二章：系统设计高频题

## 12.1 如何设计一个秒杀系统

**核心答案：**
秒杀系统面临的核心挑战是瞬时高并发（10w+ QPS）、超卖问题、数据一致性。需要从前端到后端多层次限流、缓存、异步化处理。

**关键考点：**

**整体架构：**
```
用户 -> CDN（静态页面）-> Nginx（限流）-> 网关（鉴权/限流）
  -> 秒杀服务（Redis 库存扣减）-> MQ 异步下单
  -> 订单服务（DB 落库）-> 支付服务
```

**防超卖方案：**
```java
// Redis Lua 脚本原子扣减库存
String lua = 
    "local stock = redis.call('get', KEYS[1]) " +
    "if tonumber(stock) <= 0 then return 0 end " +
    "redis.call('decr', KEYS[1]) return 1";
Long result = redis.eval(lua, stockKey);
```

**关键设计点：**
1. **库存预热**：活动开始前将库存加载到 Redis
2. **多级限流**：Nginx 限制同一 IP QPS，网关限制接口 QPS，Redis 限制令牌
3. **接口防刷**：Token 机制（只有点击活动页才能获取 token，凭 token 参与秒杀）
4. **MQ 异步**：秒杀成功后发 MQ，订单服务异步处理，削峰填谷
5. **幂等设计**：订单号唯一索引防重复提交
6. **降级兜底**：Redis 故障时切换为 DB 扣减（加分布式锁）

---

## 12.2 如何设计一个短链接服务

**核心答案：**
短链接服务将长 URL 转换为短 URL（如 t.cn/abc123），访问短链接时 302/301 重定向到原始 URL。核心是短码生成算法和高性能查询。

**关键考点：**

**短码生成方案：**
1. **Hash 截取**：取 MD5 或 MurmurHash 的前 6 位（有冲突需处理）
2. **自增 ID + 62 进制**：用分布式 ID（Snowflake）转 62 进制（0-9a-zA-Z）
   - 6 位 62 进制 = 62^6 ≈ 568 亿个短链（足够用）

```java
// 10进制转62进制
private static final String CHARS = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";
public String toBase62(long id) {
    StringBuilder sb = new StringBuilder();
    while (id > 0) {
        sb.append(CHARS.charAt((int)(id % 62)));
        id /= 62;
    }
    return sb.reverse().toString();
}
```

**数据存储：**
- MySQL：短码(PK) -> 原始URL + 创建时间 + 过期时间
- Redis：短码 -> 原始URL（缓存热点数据，TTL 同过期时间）

**301 vs 302：**
- 301 永久重定向：浏览器缓存，减少服务器压力，但无法统计点击
- 302 临时重定向：每次都走服务器，可统计点击、过期失效（推荐）

---

## 12.3 如何设计一个消息推送系统

**核心答案：**
消息推送系统用于向 App/Web 用户推送实时通知。核心技术：长连接（WebSocket/SSE）、推拉结合、设备在线状态管理。

**关键考点：**

**推送方式对比：**
| 方式 | 原理 | 优缺点 |
|------|------|--------|
| 短轮询 | 客户端定时 HTTP 请求 | 实现简单，延迟高，浪费资源 |
| 长轮询 | 服务端持有请求直到有消息 | 降低空请求，仍有延迟 |
| WebSocket | 全双工长连接 | 实时性好，服务端复杂 |
| SSE | 服务端单向推送（HTTP） | 简单，仅服务端推 |
| APNs/FCM | 系统级推送（App离线推送） | 省电，依赖第三方 |

**架构设计：**
```
消息来源 -> 消息队列（Kafka）-> 推送服务（维护连接池）
         -> 设备在线 -> WebSocket 推送
         -> 设备离线 -> APNs/FCM 厂商推送
         -> 用户离线消息存储（Redis/DB）-> 用户上线后拉取
```

**百万连接优化：**
- Netty 每个连接占用约 1MB 内存，百万连接需 1TB（不现实）
- 实际：每连接约 10KB（使用堆外内存、优化缓冲区）
- 多机横向扩展，用 Redis 存储 `userId -> serverId` 路由关系

---

## 12.4 如何设计一个高可用架构

**核心答案：**
高可用（HA）通过冗余、故障转移、降级、限流等手段使系统在部分故障时仍能提供服务。目标是 SLA（服务等级协议）：99.9%（年停机 8.76h）、99.99%（年停机 52min）、99.999%（年停机 5min）。

**关键考点：**

**高可用核心手段：**
1. **冗余**：主从/多副本，消除单点故障
2. **健康检查**：定期探活，自动剔除故障节点
3. **故障转移**：自动切换到备用节点（Keepalived VIP、数据库 MHA）
4. **熔断降级**：依赖服务故障时快速失败，返回兜底数据
5. **限流**：保护系统不被打垮
6. **超时与重试**：设置合理超时时间，幂等接口可重试

**数据库高可用方案：**
```
读写分离：Master 写 -> Binlog -> Slave 读（延迟几ms~几s）
主从切换：MHA / MySQL Router / Orchestrator 自动 Failover
多主：MGR（MySQL Group Replication）/ Galera Cluster
分库分表：ShardingSphere / MyCat
```

**服务高可用设计：**
- 同城双活：两个数据中心互为主备，数据库同步
- 异地多活：多城市部署，用户就近接入，数据最终一致
- 单元化架构：按业务域或用户 ID 分片，每个单元独立自治

---

## 12.5 如何设计一个分布式缓存系统

**核心答案：**
分布式缓存系统用于缓解数据库压力，提升读取性能。核心问题：缓存一致性、缓存穿透/击穿/雪崩、数据分片策略。

**关键考点：**

**缓存三大问题：**

| 问题 | 原因 | 解决方案 |
|------|------|--------|
| 缓存穿透 | 查询不存在的数据，每次打到DB | 布隆过滤器拦截；缓存空值（TTL 短） |
| 缓存击穿 | 热点key过期，大量并发打到DB | 互斥锁重建缓存；热点key不过期 |
| 缓存雪崩 | 大量key同时过期 | 过期时间加随机抖动；多级缓存；熔断 |

**数据一致性方案：**
1. **Cache Aside（旁路缓存）**：读：查缓存→不命中查DB→写缓存；写：先更新DB→再删缓存（推荐）
2. **Write Through**：写时同时更新DB和缓存
3. **Write Behind**：先写缓存，异步批量写DB（高性能，有丢失风险）

**为什么先更新DB再删缓存（而非先删缓存）？**
```
先删缓存的问题（并发场景）：
Thread1: 删缓存 -> (被抢占)
Thread2: 查缓存未命中 -> 查DB(旧数据) -> 写缓存(旧数据)
Thread1: 更新DB(新数据)
结果：缓存中是旧数据！
```

**Redis 集群方案：**
- 主从复制：读写分离，手动故障转移
- Sentinel：自动故障转移，适合中小规模
- Cluster：数据分片（16384个槽），横向扩展，适合大规模

---

> 本文档涵盖 Java 核心技术面试高频考点，建议结合实际项目经验深度理解每个知识点。
> 持续更新，欢迎补充和纠错。


---

## 第十二章 系统设计高频题

### 12.1 高并发系统设计方案

**核心答案：**
高并发系统设计的核心思路：**纵向扩展（Scale Up）+ 横向扩展（Scale Out）+ 缓存 + 异步 + 限流熔断**。

**关键考点：**
分层优化策略：
```
流量入口层：CDN 静态资源加速 + Nginx 负载均衡 + 限流
应用层：    无状态服务水平扩展 + 本地缓存 + 连接池
缓存层：    Redis 集群缓存热点数据 + 缓存预热
消息层：    MQ 削峰填谷 + 异步处理
数据库层：  读写分离 + 分库分表 + 索引优化
```
- 计算并发量：QPS = 日活用户 × 请求频率 / 86400，峰值 = 平均 × 3~10
- 单机 Tomcat 约 1000 QPS；引入缓存可提升 10x；引入 MQ 可平滑峰值
- 无状态化：Session 外移到 Redis，方便水平扩展

---

### 12.2 接口幂等性设计

**核心答案：**
幂等接口设计要求同一请求多次执行结果一致，是支付、订单等核心业务的必备能力。

**关键考点：**
Token 防重复提交方案：
```
1. 客户端请求接口前，先请求 /token 获取唯一 token（存入 Redis，设 TTL）
2. 正式请求携带 token
3. 服务端用 Lua 脚本原子检查并删除 token
4. token 存在则处理业务；不存在则返回"重复请求"
```
订单幂等设计：
- 订单号作为幂等 key，数据库唯一索引
- 状态机校验：已支付订单不可重复支付
- 乐观锁：UPDATE orders SET status=2 WHERE id=? AND status=1

---

### 12.3 秒杀系统设计

**核心答案：**
秒杀系统特点：瞬间高并发、库存有限。核心思路：分层过滤请求，尽量在上层拦截，减少对数据库的压力。

**关键考点：**
```
前端层：  按钮防重点击 + 页面静态化 + CDN 缓存
网关层：  用户鉴权 + IP 限流（令牌桶）+ 恶意请求过滤
应用层：  随机拒绝（只放 1% 请求进入） + Redis 预减库存
消息层：  MQ 异步下单，削峰填谷
数据库层：库存扣减（乐观锁/数据库行锁）
```
Redis 预减库存：
```java
Long stock = redisTemplate.opsForValue().decrement("stock:itemId");
if (stock < 0) {
    // 库存不足，返回失败
    redisTemplate.opsForValue().increment("stock:itemId"); // 回补
    return "售罄";
}
// 发送 MQ，异步创建订单
```
注意事项：
- 库存超卖防护：Redis lua 脚本原子操作 + DB 乐观锁双重防护
- 热点 key：库存 key 分片（stock:itemId:0 ~ stock:itemId:N）
- 防刷：用户维度限速 + 验证码

---

### 12.4 短链接系统设计

**核心答案：**
短链系统将长 URL 映射为短码（6~8位），核心是短码生成算法、存储、重定向。

**关键考点：**
短码生成方案：
1. **自增 ID + Base62 编码**：ID 转 62 进制（0-9 A-Z a-z），简单，有规律可被猜测
2. **Hash 截断**：MD5 取前 8 位，有碰撞风险
3. **随机 + 布隆过滤器**：随机生成 + 布隆过滤器去重

系统设计：
```
写入：长URL -> 生成短码 -> 存 DB(短码, 长URL, 有效期) + Redis 缓存
读取：短码 -> 查 Redis -> 查 DB -> 301/302 重定向到长URL
```
- 301（永久）vs 302（临时）：301 浏览器缓存，减少请求但无法统计点击；302 每次都到服务器，可统计
- 自定义短链：用户指定短码，检查唯一性后直接使用
- 分库分表：以短码 hash 分片

---

### 12.5 设计一个消息推送系统

**核心答案：**
消息推送系统需支持实时性、可靠性、多端适配（APP、Web、邮件、短信）。

**关键考点：**
架构分层：
```
业务系统 -> 推送服务 -> 渠道路由 -> APNs/FCM/WebSocket/短信/邮件
                    -> 消息存储  -> 离线消息持久化
```
核心功能：
- 消息优先级：紧急（实时推送）/ 普通（批量推送）
- 离线消息：用户上线后拉取未读消息（消息存 DB，用 offset 标记已读）
- 消息去重：推送 ID 作为唯一标识，客户端去重
- 推送失败重试：指数退避 + 死信队列
- 大规模推送（全量）：分批推送，避免瞬间打满下游

---

### 12.6 数据库分库分表方案

**核心答案：**
当单表数据量超过 1000 万或单库 QPS 超过 2000 时，考虑分库分表。分库解决并发问题，分表解决存储问题。

**关键考点：**
分片策略：
- **Hash 分片**：key % N，数据均匀，但扩容需迁移（一致性 hash 减少迁移量）
- **Range 分片**：按时间/ID 范围，方便范围查询，但有热点问题

分库分表问题：
| 问题 | 解决方案 |
|-----|---------|
| 跨分片查询 | 冗余存储，或通过 ES 解决 |
| 跨分片排序/分页 | 二次查询合并排序（性能差）|
| 跨分片事务 | 分布式事务（Seata）|
| 全局唯一ID | 雪花算法/号段模式 |
| 扩容迁移 | 双写 + 数据迁移 + 灰度切换 |

中间件：ShardingSphere（ShardingJDBC/ShardingProxy）、MyCat

---

### 12.7 OOM 问题排查

**核心答案：**
OOM（OutOfMemoryError）排查分三步：获取 Heap Dump -> 分析内存泄漏对象 -> 定位代码。

**关键考点：**
OOM 类型：
- `java.lang.OutOfMemoryError: Java heap space`：堆内存溢出
- `java.lang.OutOfMemoryError: GC overhead limit exceeded`：GC 占用时间过长
- `java.lang.OutOfMemoryError: Metaspace`：元空间溢出（类加载过多）
- `java.lang.StackOverflowError`：栈溢出（递归过深）

排查步骤：
```
1. JVM 参数加 -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/dump.hprof
2. 使用 MAT (Memory Analyzer Tool) 或 VisualVM 分析 dump 文件
3. 查看 Dominator Tree，找到占用内存最大的对象
4. 找到 GC Roots 到该对象的引用链
5. 定位到代码中持有该对象引用的地方
```
常见原因：
- 数据库查询未加分页，一次查询百万数据
- 缓存对象无限增长（Map 未清理）
- 内存泄漏：ThreadLocal 未 remove、监听器未注销
- 字符串拼接产生大量临时对象

---

### 12.8 CPU 飙高问题排查

**核心答案：**
Java 进程 CPU 飙高通常由死循环、大量 GC、线程竞争激烈导致。排查流程：找高 CPU 线程 -> 找对应 Java 线程 -> 分析堆栈。

**关键考点：**
排查步骤（Linux）：
```bash
1. top -c 找到高 CPU 的 Java 进程 PID
2. top -H -p {PID} 找到高 CPU 的线程 TID
3. printf "%x
" {TID} 将 TID 转为 16 进制
4. jstack {PID} > stack.txt 导出线程堆栈
5. 在 stack.txt 中搜索 16 进制 TID 对应的线程堆栈
6. 分析堆栈定位代码
```
常见原因：
- **死循环**：堆栈显示某方法持续循环执行
- **频繁 GC**：jstat -gcutil {PID} 1000 观察 GC 频率和耗时
- **正则回溯**：复杂正则表达式在某些输入下指数级回溯
- **序列化/反序列化**：大对象频繁序列化

---

## 附录：面试高频手写代码

### A.1 手写单例模式（双重检查锁）

```java
public class Singleton {
    private static volatile Singleton instance;
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

### A.2 手写 LRU 缓存

```java
public class LRUCache extends LinkedHashMap<Integer, Integer> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder=true
        this.capacity = capacity;
    }
    
    public int get(int key) {
        return super.getOrDefault(key, -1);
    }
    
    public void put(int key, int value) {
        super.put(key, value);
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
}
```

### A.3 手写线程安全的生产者消费者

```java
public class ProducerConsumer {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int MAX_SIZE = 10;
    private final Object lock = new Object();
    
    public void produce(int item) throws InterruptedException {
        synchronized (lock) {
            while (queue.size() == MAX_SIZE) {
                lock.wait();
            }
            queue.offer(item);
            lock.notifyAll();
        }
    }
    
    public int consume() throws InterruptedException {
        synchronized (lock) {
            while (queue.isEmpty()) {
                lock.wait();
            }
            int item = queue.poll();
            lock.notifyAll();
            return item;
        }
    }
}
```

### A.4 手写快速排序

```java
public void quickSort(int[] arr, int left, int right) {
    if (left >= right) return;
    int pivot = partition(arr, left, right);
    quickSort(arr, left, pivot - 1);
    quickSort(arr, pivot + 1, right);
}

private int partition(int[] arr, int left, int right) {
    int pivot = arr[right];
    int i = left - 1;
    for (int j = left; j < right; j++) {
        if (arr[j] <= pivot) {
            i++;
            int temp = arr[i]; arr[i] = arr[j]; arr[j] = temp;
        }
    }
    int temp = arr[i+1]; arr[i+1] = arr[right]; arr[right] = temp;
    return i + 1;
}
```

### A.5 手写 ThreadLocal 实现原理摘要

```
每个 Thread 对象内部有一个 ThreadLocalMap（以 ThreadLocal 为 key，值为 value）
ThreadLocal.set(value)  -> Thread.currentThread().threadLocalMap.set(this, value)
ThreadLocal.get()       -> Thread.currentThread().threadLocalMap.get(this)

内存泄漏：ThreadLocalMap 的 key 是 ThreadLocal 弱引用，value 是强引用
当 ThreadLocal 被 GC 后，key 为 null，value 无法被回收 -> 内存泄漏
解决：使用完后调用 ThreadLocal.remove()
```

---

*本手册涵盖 Java 后端面试核心知识点，建议结合项目经验深入理解，祝面试顺利！*

---

# 附录：高频算法题速查

## A.1 常见排序算法复杂度

| 算法 | 平均时间 | 最坏时间 | 空间 | 稳定性 |
|------|---------|---------|------|--------|
| 冒泡 | O(n²) | O(n²) | O(1) | 稳定 |
| 选择 | O(n²) | O(n²) | O(1) | 不稳定 |
| 插入 | O(n²) | O(n²) | O(1) | 稳定 |
| 希尔 | O(n log n) | O(n²) | O(1) | 不稳定 |
| 归并 | O(n log n) | O(n log n) | O(n) | 稳定 |
| 快排 | O(n log n) | O(n²) | O(log n) | 不稳定 |
| 堆排 | O(n log n) | O(n log n) | O(1) | 不稳定 |
| 计数 | O(n+k) | O(n+k) | O(k) | 稳定 |
| 桶排 | O(n+k) | O(n²) | O(n+k) | 稳定 |
| 基数 | O(n*k) | O(n*k) | O(n+k) | 稳定 |

---

## A.2 数据结构时间复杂度速查

| 数据结构 | 查找 | 插入 | 删除 | 备注 |
|---------|------|------|------|------|
| 数组（有序） | O(log n) | O(n) | O(n) | 二分查找 |
| 链表 | O(n) | O(1) | O(1) | 需先找到节点 |
| 哈希表 | O(1) | O(1) | O(1) | 哈希冲突时退化 |
| 二叉搜索树 | O(log n) | O(log n) | O(log n) | 平衡时 |
| AVL 树 | O(log n) | O(log n) | O(log n) | 严格平衡 |
| 红黑树 | O(log n) | O(log n) | O(log n) | Java TreeMap 底层 |
| B+ 树 | O(log n) | O(log n) | O(log n) | MySQL 索引 |
| 跳表 | O(log n) | O(log n) | O(log n) | Redis ZSet |
| 堆（最大/最小） | O(1)取顶 | O(log n) | O(log n) | 优先队列 |

---

## A.3 Java 面试黄金法则

**回答问题的 STAR 结构：**
- **S**ituation（背景）：描述遇到的场景
- **T**ask（任务）：你需要解决什么问题
- **A**ction（行动）：你采取了哪些措施
- **R**esult（结果）：最终效果如何

**技术深度回答框架（以 HashMap 为例）：**
1. 是什么（定义、用途）
2. 怎么用（基本 API）
3. 底层原理（数据结构、核心算法）
4. 源码关键点（put/get/resize 流程）
5. 常见问题（线程安全、性能调优）
6. 与同类对比（HashTable、ConcurrentHashMap）
7. 实际应用（项目中的使用经验）

**面试常见陷阱总结：**
- `i++` 不是原子操作，需要 `AtomicInteger`
- `==` 比较引用，`.equals()` 比较值（重写 equals 必须重写 hashCode）
- `ArrayList` 扩容 1.5 倍（`int newCapacity = oldCapacity + (oldCapacity >> 1)`）
- `HashMap` 扩容 2 倍，负载因子默认 0.75，树化阈值 8，退树化阈值 6
- `String` 是不可变的，`intern()` 方法从字符串池获取引用
- `try-catch-finally` 中 finally 必然执行（JVM 退出除外）
- `static` 方法不能被重写（Override），可以被隐藏（Hide）
- 泛型在编译后类型擦除，运行时只有原始类型

---

## A.4 Spring Boot 项目性能优化清单

**启动优化：**
- 延迟初始化：`spring.main.lazy-initialization=true`
- 去除不用的自动配置：`@SpringBootApplication(exclude=...)`
- 减少 classpath 扫描范围

**接口性能优化：**
1. 数据库：索引优化、减少 N+1 查询、分页优化（避免深分页）
2. 缓存：Redis 缓存热点数据，本地缓存（Caffeine）二级缓存
3. 并发：CompletableFuture 并行调用多个服务
4. 序列化：使用 Protobuf 代替 JSON（体积减少 3-10x）
5. 连接池：HikariCP 连接池调优（maximumPoolSize、connectionTimeout）

**JVM 调优：**
```
# 推荐 JVM 参数（Java 17+，G1 GC）
-Xms4g -Xmx4g                    # 堆大小固定，避免扩容停顿
-XX:+UseG1GC                      # 使用 G1 GC
-XX:MaxGCPauseMillis=200          # 目标最大停顿 200ms
-XX:+HeapDumpOnOutOfMemoryError   # OOM 时生成 Heap Dump
-XX:HeapDumpPath=/tmp/heapdump    # Dump 路径
-Xlog:gc*:file=/tmp/gc.log        # GC 日志
```

---

*文档版本：2026-07 | 适用范围：Java 后端开发工程师（1-5年）面试备战*

