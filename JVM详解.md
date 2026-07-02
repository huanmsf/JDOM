# JVM 深度详解 - 从入门到精通

> 面向 Java 初学者和中级开发者的完整 JVM 学习指南
> 包含架构图、代码示例、实战案例、面试题解析

---

## 目录

- [Part 1 - JVM 整体架构](#part-1---jvm-整体架构)
- [Part 2 - 类加载机制](#part-2---类加载机制)
- [Part 3 - 运行时数据区（内存模型）](#part-3---运行时数据区内存模型)
- [Part 4 - 垃圾回收（GC）机制](#part-4---垃圾回收gc机制)
- [Part 5 - 垃圾收集器详解](#part-5---垃圾收集器详解)
- [Part 6 - JVM 调优实战](#part-6---jvm-调优实战)
- [Part 7 - JVM 监控与诊断工具](#part-7---jvm-监控与诊断工具)
- [Part 8 - 执行引擎与 JIT 编译](#part-8---执行引擎与-jit-编译)
- [Part 9 - 完整实战案例](#part-9---完整实战案例)
- [Part 10 - 常见面试题 & FAQ](#part-10---常见面试题--faq)

---

# Part 1 - JVM 整体架构

## 1.1 JVM 是什么？为什么需要 JVM？

### 通俗比喻

想象一下，你写了一封信（Java 源代码），但是世界上有各种不同语言的人（Windows、Linux、macOS 等不同操作系统）。如果没有翻译官，每个人都看不懂这封信。JVM 就是那个「万能翻译官」——它能把同一封信翻译给所有人听。

**JVM（Java Virtual Machine，Java 虚拟机）** 是一个虚拟的计算机，它有自己的指令集、内存管理和执行引擎。JVM 屏蔽了底层操作系统的差异，让 Java 程序可以在任何安装了 JVM 的机器上运行。

### Write Once, Run Anywhere（一次编写，到处运行）

不同语言的人喻示不同操作系统。没有翻译（JVM）就无法通用。

### JVM 的主要职责

1. **加载字节码**：将 .class 文件加载到内存中
2. **内存管理**：自动分配和回收内存（垃圾回收 GC）
3. **执行字节码**：通过解释执行或 JIT 编译执行字节码
4. **安全沙箱**：提供安全隔离，防止恶意代码破坏系统
5. **线程管理**：管理 Java 线程的调度

---

## 1.2 JVM 整体架构图

```
+==================================================================+
|                         JVM 整体架构                              |
+==================================================================+
|                                                                  |
|   +----------------------------------------------------------+   |
|   |                   类加载子系统                             |   |
|   |  加载(Loading) -> 验证(Verify) -> 准备(Prepare) ->        |   |
|   |  解析(Resolve) -> 初始化(Initialize)                      |   |
|   +-------------------------+--------------------------------+   |
|                             |                                    |
|   +-------------------------v--------------------------------+   |
|   |                   运行时数据区                             |   |
|   |  +----------+ +----------+ +------------------------+   |   |
|   |  | 程序计数器 | | 虚拟机栈  | |      本地方法栈          |   |   |
|   |  |(PC Reg)   | |(VM Stack)| |(Native Method Stack)  |   |   |
|   |  |  线程私有  | |  线程私有 | |       线程私有           |   |   |
|   |  +----------+ +----------+ +------------------------+   |   |
|   |  +-----------------------------------------------------+ |   |
|   |  |                    堆 (Heap)                        | |   |
|   |  |  +---------------------+  +-----------------+      | |   |
|   |  |  |      新生代           |  |     老年代        |      | |   |
|   |  |  | +------+----+----+  |  |                 |      | |   |
|   |  |  | |Eden  | S0 | S1 |  |  |    Old Gen      |      | |   |
|   |  |  | +------+----+----+  |  |                 |      | |   |
|   |  |  +---------------------+  +-----------------+      | |   |
|   |  |                    线程共享                          | |   |
|   |  +-----------------------------------------------------+ |   |
|   |  +-----------------------------------------------------+ |   |
|   |  |              方法区 (Method Area)                   | |   |
|   |  |     类信息 / 常量池 / 静态变量 / 即时编译代码           | |   |
|   |  |                    线程共享                          | |   |
|   |  +-----------------------------------------------------+ |   |
|   +----------------------------------------------------------+   |
|                                                                  |
|   +----------------------------------------------------------+   |
|   |                      执行引擎                             |   |
|   |   解释器(Interpreter) | JIT编译器 | 垃圾收集器(GC)          |   |
|   +----------------------------------------------------------+   |
|                                                                  |
|   +----------------------------------------------------------+   |
|   |              本地库接口 (Native Interface)                 |   |
|   |                本地方法库 (Native Libraries)               |   |
|   +----------------------------------------------------------+   |
+==================================================================+
```

---

## 1.3 Java 程序从源码到运行的完整流程

```
第一步：编写源代码
+---------------------------------+
|  HelloWorld.java                |
|  public class HelloWorld {      |
|    public static void main(...) |
|      System.out.println("Hi");  |
|    }                            |
|  }                              |
+-------------+-------------------+
              | javac 编译器编译
              v
第二步：编译成字节码
+---------------------------------+
|  HelloWorld.class               |
|  cafe babe 0000 0034 001d ...   |
+---------------------------------+
              | java 命令启动JVM
              v
第三步：类加载器加载
+---------------------------------+
|  ClassLoader                    |
|  1. 读取 .class 文件              |
|  2. 验证字节码合法性               |
|  3. 在方法区创建 Class 对象         |
+---------------------------------+
              v
第四步：执行引擎执行
+---------------------------------+
|  Execution Engine               |
|  1. 解释执行（逐条翻译字节码）       |
|  2. JIT编译（热点代码编译成机器码）   |
|  3. 调用本地方法（如 println）       |
+---------------------------------+
              v
第五步：输出结果
+---------------------------------+
|  控制台：Hello World              |
+---------------------------------+
```

字节码示例（javap -c HelloWorld.class）：

```java
// 源代码
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

```
// 对应的字节码
public static void main(java.lang.String[]);
  Code:
     0: getstatic     #2  // 获取 System.out 字段
     3: ldc           #3  // 加载字符串常量 "Hello, World!"
     5: invokevirtual #4  // 调用 println 方法
     8: return             // 方法返回
```

---

## 1.4 HotSpot VM 简介

**HotSpot VM** 是目前最主流的 JVM 实现，由 Sun Microsystems（现 Oracle）开发。

### 为什么叫 HotSpot？

HotSpot 的核心理念是「**热点探测**」——找出程序中被频繁执行的「热点代码」，对其进行 JIT（即时编译），将字节码编译成本地机器码以提升性能。就像高速路上的车流热点——对高频使用的路段进行优化改造。

### HotSpot 的主要特性

| 特性 | 说明 |
|------|------|
| 自适应优化 | 运行时分析热点代码并动态优化 |
| 分层编译 | C1（客户端编译器）+ C2（服务端编译器）分层优化 |
| 逃逸分析 | 分析对象是否逃逸，进行栈上分配等优化 |
| 垃圾回收 | 支持多种 GC 算法（Serial/Parallel/CMS/G1/ZGC） |

### 其他 JVM 实现

- **OpenJ9**（IBM，现 Eclipse）：内存占用低，容器场景友好
- **GraalVM**：支持多语言，可以编译成本地镜像（Native Image）
- **Azul Zing**：商业 JVM，低延迟特性

---

## 1.5 JVM、JRE、JDK 三者关系

```
+=====================================================+
|                    JDK                              |
|  +-----------------------------------------------+ |
|  |                 JRE                           | |
|  |  +-----------------------------------------+ | |
|  |  |              JVM                        | | |
|  |  |   类加载 + 运行时数据区 + 执行引擎             | | |
|  |  +-----------------------------------------+ | |
|  |  Java 核心类库（java.lang, java.util...）      | |
|  +-----------------------------------------------+ |
|  开发工具：javac/javap/jdb/jmap/jstat/...           |
+=====================================================+

JVM：最小运行单元，负责执行字节码
JRE：JVM + Java标准类库，能运行Java程序（用户需要）
JDK：JRE + 开发工具，能开发Java程序（开发者需要）
```

**类比**：
- JVM = 汽车发动机（核心动力）
- JRE = 完整的汽车（能跑路）
- JDK = 汽车 + 修车工具箱（能开车 + 能修车）
---

# Part 2 - 类加载机制

## 2.1 类加载的五个阶段

类的生命周期分为七个阶段，其中前五个阶段称为「类加载过程」：

```
加载       验证       准备       解析       初始化      使用       卸载
(Loading)->(Verify)->(Prepare)->(Resolve)->(Initialize)->(Using)->(Unloading)
```

### 阶段一：加载（Loading）

**做什么**：把 .class 文件的二进制数据读入内存。

具体步骤：
1. 通过类的全限定名（如 com.example.HelloWorld）找到 .class 文件
2. 将字节流转换为方法区的运行时数据结构
3. 在堆中生成一个代表此类的 java.lang.Class 对象

**类比**：就像图书馆员工把书从仓库搬到书架上，并在目录系统中登记。

```
磁盘上的 HelloWorld.class
        | ClassLoader.loadClass()
        v
方法区：类的元数据（字段、方法、常量池...）
        |
        v
堆区：java.lang.Class 对象（Class的「镜像」）
```

### 阶段二：验证（Verification）

**做什么**：确保字节码文件内容合法，不会危害 JVM 安全。

验证内容：
1. **文件格式验证**：是否以 0xCAFEBABE 开头，版本号是否支持
2. **元数据验证**：类是否有父类，是否实现了接口要求的所有方法
3. **字节码验证**：字节码语义是否合法，如跳转指令是否指向合法位置
4. **符号引用验证**：确保符号引用能解析到实际引用

**类比**：海关检查护照——确保入境者的证件合法、真实、没有被篡改。

### 阶段三：准备（Preparation）

**做什么**：为类的**静态变量**分配内存，并设置**默认初始值**（注意：不是代码中写的初始值！）。

```java
public class Example {
    public static int value = 100;  // 准备阶段：value = 0（默认值）
    public static final int MAX = 100; // 准备阶段：MAX = 100（final常量直接赋值）
}
// 初始化阶段才会：value = 100
```

各类型默认值：

| 类型 | 默认值 |
|------|--------|
| int/short/byte/char | 0 |
| long | 0L |
| float | 0.0f |
| double | 0.0 |
| boolean | false |
| reference | null |

**类比**：装修新房，先把所有房间都刷成白色（默认值），之后再按需求涂上不同颜色（初始化）。

### 阶段四：解析（Resolution）

**做什么**：将常量池中的**符号引用**替换为**直接引用**。

- **符号引用**：用字符串描述的目标（如 com/example/HelloWorld）
- **直接引用**：直接指向目标的指针、偏移量或句柄

**类比**：把通讯录中的「张三」（符号引用）换成具体的手机号码（直接引用）。

### 阶段五：初始化（Initialization）

**做什么**：执行类的 &lt;clinit&gt; 方法，真正执行静态变量赋值和静态代码块。

```java
public class InitExample {
    static int a = 10;        // 静态变量赋值
    static int b;
    
    static {                  // 静态代码块
        b = a + 5;
        System.out.println("类初始化执行！");
    }
    
    // JVM 合成 <clinit> 方法，按代码顺序执行以上操作
}
```

**重要特性**：
- &lt;clinit&gt; 方法是线程安全的（JVM 保证只执行一次）
- 父类的 &lt;clinit&gt; 先于子类执行

---

## 2.2 类加载器的种类

```
类加载器层次结构：

      +-------------------------------+
      |   Bootstrap ClassLoader        |  启动类加载器
      |   加载：JAVA_HOME/lib/*.jar    |  (C++ 实现，Java中为null)
      |   如：rt.jar, resources.jar     |
      +--------------+----------------+
                     | 父类加载器
      +--------------v----------------+
      |  Extension ClassLoader         |  扩展类加载器
      |  加载：JAVA_HOME/lib/ext/*.jar |  (Java实现)
      |  JDK9后改为Platform ClassLoader |
      +--------------+----------------+
                     | 父类加载器
      +--------------v----------------+
      |  Application ClassLoader       |  应用类加载器
      |  加载：classpath 下的类          |  (也叫系统类加载器)
      |  即你写的所有业务代码             |
      +--------------+----------------+
                     | 父类加载器
      +--------------v----------------+
      |  Custom ClassLoader            |  自定义类加载器
      |  由开发者自定义加载逻辑           |
      |  如：从网络、数据库加载类          |
      +-------------------------------+
```

---

## 2.3 双亲委派模型

### 原理

当一个类加载器收到加载请求时，**不会自己先去加载**，而是把这个请求委托给父类加载器。只有当父类加载器无法完成加载时，子加载器才会尝试自己加载。

```
类加载请求流程（以 java.lang.String 为例）：

应用类加载器 收到请求："加载 java.lang.String"
          | 委托给父类
          v
扩展类加载器 收到请求
          | 委托给父类
          v
启动类加载器 收到请求
          | 在 rt.jar 中找到！
         <-- 加载成功，返回 Class 对象

如果是自定义类 com.example.MyClass：
启动类加载器：rt.jar中没有 -> 返回失败
          | 子类尝试
          v
扩展类加载器：ext目录没有 -> 返回失败
          | 子类尝试
          v
应用类加载器：classpath中找到！-> 加载成功
```

### 为什么需要双亲委派？

1. **避免重复加载**：同一个类只会被加载一次
2. **安全性**：防止核心类库被篡改

```java
// 如果没有双亲委派，你可以这样写：
package java.lang;
public class String {
    static {
        System.out.println("我替换了 java.lang.String！");
        // 发送密码到服务器... (恶意代码)
    }
}
// 有了双亲委派，java.lang.String 永远由启动类加载器加载
// 你自己写的这个 String 类永远不会生效
```

### 代码示例：查看类加载器

```java
public class ClassLoaderDemo {
    public static void main(String[] args) {
        // 查看 String 的类加载器（Bootstrap，Java层表示为null）
        System.out.println(String.class.getClassLoader());
        // 输出：null（Bootstrap ClassLoader）
        
        // 查看自己写的类的加载器
        System.out.println(ClassLoaderDemo.class.getClassLoader());
        // 输出：sun.misc.Launcher\@...
        
        // 类加载器的层级关系
        ClassLoader cl = ClassLoaderDemo.class.getClassLoader();
        System.out.println("AppClassLoader: " + cl);
        System.out.println("ExtClassLoader: " + cl.getParent());
        System.out.println("BootstrapClassLoader: " + cl.getParent().getParent());
        // 最后输出 null，因为Bootstrap是C++实现的
    }
}
```

---

## 2.4 打破双亲委派的场景

### 场景一：JDBC 驱动加载

**问题**：java.sql.Driver 接口在 rt.jar 中（由Bootstrap加载），但具体实现（如 com.mysql.jdbc.Driver）在用户的classpath中。Bootstrap加载器无法加载用户的实现类。

**解决方案**：线程上下文类加载器（Thread Context ClassLoader）

```java
// JDBC 使用线程上下文类加载器加载驱动
ServiceLoader<Driver> loadedDrivers = 
    ServiceLoader.load(Driver.class, 
        Thread.currentThread().getContextClassLoader());
// 线程上下文类加载器默认是 AppClassLoader
// 这样 Bootstrap 加载的类就能「反向」使用 AppClassLoader
```

### 场景二：Tomcat 类加载

Tomcat 需要支持在同一个 JVM 中运行多个 Web 应用，每个应用可能使用不同版本的同一个类库。

```
Tomcat 类加载器结构：

Bootstrap ClassLoader（JDK核心库）
        |
        v
Extension ClassLoader
        |
        v
AppClassLoader（Tomcat自身类库）
        |
    +---+------------------+
    v                      v
WebApp1 ClassLoader   WebApp2 ClassLoader
（应用1独立类空间）      （应用2独立类空间）
```

每个 WebApp 有独立的类加载器，彼此隔离，可以加载不同版本的 Spring 等框架。

### 场景三：OSGi 模块化框架

OSGi（如 Eclipse 插件系统）实现了更复杂的类加载网络，允许模块间相互依赖并动态更新。

---

## 2.5 类初始化的触发时机（6种主动引用）

**主动引用**（会触发初始化）：

```java
// 1. 创建类的实例（new）
MyClass obj = new MyClass();

// 2. 访问或赋值类的静态字段
int v = MyClass.staticField;
MyClass.staticField = 100;

// 3. 调用类的静态方法
MyClass.staticMethod();

// 4. 反射调用
Class.forName("com.example.MyClass");

// 5. 初始化子类（先触发父类初始化）
class Parent {}
class Child extends Parent {}
new Child(); // 先初始化Parent，再初始化Child

// 6. JVM启动时的主类（含main方法的类）
// java HelloWorld -> HelloWorld 类被初始化
```

**被动引用**（不会触发初始化）：

```java
// 通过子类引用父类的静态字段（只触发父类初始化）
System.out.println(Child.parentStaticField); // 只初始化Parent

// 通过数组定义引用类（不触发初始化）
MyClass[] array = new MyClass[10]; // 不触发MyClass的初始化

// 访问编译时常量（final static）
System.out.println(MyClass.FINAL_CONSTANT); // 不触发MyClass初始化
// 常量在编译期就已经内联到调用方的字节码中了
```

---

## 2.6 完整自定义类加载器代码示例

```java
import java.io.*;
import java.nio.file.*;

/**
 * 自定义类加载器示例：从指定目录加载加密的 .class 文件
 * 加密方式：每个字节异或 0xFF（简单示例）
 */
public class EncryptedClassLoader extends ClassLoader {
    
    private final String classDir;
    
    public EncryptedClassLoader(String classDir, ClassLoader parent) {
        super(parent); // 设置父类加载器（遵循双亲委派）
        this.classDir = classDir;
    }
    
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // 将类名转换为文件路径
        String fileName = name.replace('.', '/') + ".class";
        String fullPath = classDir + File.separator + fileName;
        
        try {
            byte[] encryptedBytes = Files.readAllBytes(Paths.get(fullPath));
            byte[] classBytes = decrypt(encryptedBytes);
            return defineClass(name, classBytes, 0, classBytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException("无法找到类文件：" + fullPath, e);
        }
    }
    
    private byte[] decrypt(byte[] data) {
        byte[] result = new byte[data.length];
        for (int i = 0; i < data.length; i++) {
            result[i] = (byte) (data[i] ^ 0xFF);
        }
        return result;
    }
    
    public static void main(String[] args) throws Exception {
        EncryptedClassLoader loader = new EncryptedClassLoader(
            "/encrypted", 
            EncryptedClassLoader.class.getClassLoader()
        );
        
        Class<?> helloClass = loader.loadClass("Hello");
        Object instance = helloClass.getDeclaredConstructor().newInstance();
        helloClass.getMethod("sayHello").invoke(instance);
        
        System.out.println("类加载器：" + helloClass.getClassLoader());
    }
}
```
---

# Part 3 - 运行时数据区（内存模型）

## 3.1 整体内存区域图

```
+============================================================+
|                    JVM 运行时数据区                          |
+============================================================+
|  +------------------------------------------------------+  |
|  |               线程共享区域                            |  |
|  |  +------------------------------------------------+  |  |
|  |  |                  堆 (Heap)                    |  |  |
|  |  |  +--------------------+  +--------------+    |  |  |
|  |  |  |       新生代         |  |    老年代      |    |  |  |
|  |  |  |  Eden: S0:S1 = 8:1:1|  |              |    |  |  |
|  |  |  +--------------------+  +--------------+    |  |  |
|  |  |         占堆总大小的 1/3          占 2/3        |  |  |
|  |  +------------------------------------------------+  |  |
|  |  +------------------------------------------------+  |  |
|  |  |              方法区 (Method Area)              |  |  |
|  |  |  JDK7: 永久代(PermGen) JDK8+: 元空间(Metaspace)|  |  |
|  |  +------------------------------------------------+  |  |
|  +------------------------------------------------------+  |
|                                                            |
|  +------------------------------------------------------+  |
|  |               线程私有区域（每个线程独有）              |  |
|  |  线程1：[程序计数器][虚拟机栈][本地方法栈]              |  |
|  |  线程2：[程序计数器][虚拟机栈][本地方法栈]              |  |
|  |  线程3：[程序计数器][虚拟机栈][本地方法栈]              |  |
|  +------------------------------------------------------+  |
|                                                            |
|  +------------------------------------------------------+  |
|  |              直接内存 (Direct Memory)                |  |
|  |  不属于JVM规范，但Java NIO会使用                      |  |
|  +------------------------------------------------------+  |
+============================================================+
```

---

## 3.2 程序计数器（PC Register）

### 是什么？

程序计数器是一块很小的内存，**记录当前线程正在执行的字节码指令的地址**（行号）。

**类比**：就像书签——当你读书读到一半放下，书签帮你记住读到第几页了。每个线程都有自己独立的书签。

### 特点

- **线程私有**：每个线程有独立的程序计数器，互不干扰
- **生命周期**：随线程创建而创建，随线程销毁而销毁
- **执行 Java 方法**：记录虚拟机字节码指令地址
- **执行 Native 方法**：值为 undefined（不需要记录）

### 为什么不会 OOM？

程序计数器只存储一个地址值（指令地址），**大小固定**，不会随程序运行增长，因此 Java 规范规定程序计数器是**唯一一个不会发生 OutOfMemoryError 的内存区域**。

---

## 3.3 虚拟机栈（VM Stack）

### 栈帧结构

每次调用一个方法，JVM 就会在虚拟机栈中压入一个**栈帧（Stack Frame）**。方法返回时，栈帧弹出。

**类比**：虚拟机栈就像一叠盘子，每次调用方法放上一个盘子（压栈），方法返回取下一个盘子（出栈）。

```
虚拟机栈结构（以调用链 main->methodA->methodB 为例）：

栈顶 +------------------------------------------+
     |          methodB 的栈帧                   | <- 当前执行的方法
     |  +----------------+ +------------------+ |
     |  |   局部变量表    | |    操作数栈        | |
     |  |  slot[0]=this  | |  [操作数][操作数]  | |
     |  |  slot[1]=x     | |                  | |
     |  |  slot[2]=y     | +------------------+ |
     |  +----------------+ +------------------+ |
     |                     |    动态链接        | |
     |                     | (指向运行时常量池)  | |
     |                     +------------------+ |
     |                     +------------------+ |
     |                     |   方法返回地址     | |
     |                     | (返回给调用者位置)  | |
     |                     +------------------+ |
     +------------------------------------------+
     |          methodA 的栈帧                   |
     +------------------------------------------+
栈底 |           main 的栈帧                    |
     +------------------------------------------+
```

### 栈帧的四个组成部分

**① 局部变量表（Local Variable Table）**

存储方法的参数和局部变量，以**槽（Slot）**为单位：
- 大多数类型占 1 个 Slot（int、引用类型等）
- long 和 double 占 2 个 Slot

```java
public void example(int a, String b, long c) {
    // slot[0] = this（实例方法第0个位置是this）
    // slot[1] = a（int，占1个slot）
    // slot[2] = b（String引用，占1个slot）
    // slot[3-4] = c（long，占2个slot）
    int local = 10;
    // slot[5] = local
}
```

**② 操作数栈（Operand Stack）**

JVM 的字节码是基于栈的架构，操作数栈是计算的工作区域：

```
计算 1 + 2 的字节码执行过程：

初始状态：操作数栈为空 []

执行 iconst_1：将整数1压栈  [1]
执行 iconst_2：将整数2压栈  [1, 2]
执行 iadd：    弹出两个值相加，结果压栈  [3]
执行 istore_1：将3存入局部变量表slot[1]  []
```

**③ 动态链接（Dynamic Linking）**

每个栈帧都保存一个指向运行时常量池中该方法的符号引用的指针，方便在运行时将符号引用解析为直接引用。

**④ 方法返回地址（Return Address）**

方法执行完毕后，需要返回到调用者继续执行，返回地址记录了调用者的下一条指令位置。

### 栈相关异常

```java
// StackOverflowError：栈深度超过限制
// 典型场景：无限递归
public void infiniteRecursion() {
    infiniteRecursion(); // 每次调用创建新栈帧，最终栈满溢出
}
// 抛出：java.lang.StackOverflowError
```

`ash
-Xss256k    # 设置每个线程的虚拟机栈大小（默认512k~1M）
`

---

## 3.4 本地方法栈（Native Method Stack）

与虚拟机栈类似，但专门服务于 **Native 方法**（用 C/C++ 编写的方法）。

**类比**：虚拟机栈是专门服务 Java 员工的休息室，本地方法栈是专门服务「外包员工」（Native方法）的休息室。

HotSpot VM 将虚拟机栈和本地方法栈合并为同一块区域。

---

## 3.5 堆（Heap）

### 堆的地位

**堆是 JVM 中最大的内存区域**，几乎所有对象实例都在堆中分配内存。堆是 GC 管理的主要区域，也叫「GC堆」。

**类比**：堆就像一个大仓库，所有新创建的「货物」（对象）都存放在这里。仓库管理员（GC）定期清理不用的货物。

### 堆的分区详解

```
堆内存分区：

+-----------------------------------------------------------+
|                        Java 堆                            |
|  +-------------------------------+  +------------------+ |
|  |           新生代（Young Gen）  |  |   老年代（Old Gen） | |
|  |  +-------------+----+------+  |  |                  | |
|  |  |     Eden    | S0 |  S1  |  |  |                  | |
|  |  |    (8份)    |(1) | (1)  |  |  |                  | |
|  |  +-------------+----+------+  |  |                  | |
|  |       Eden:S0:S1 = 8:1:1      |  |                  | |
|  |   新生代默认占堆的 1/3           |  | 老年代默认占堆的 2/3  | |
|  +-------------------------------+  +------------------+ |
+-----------------------------------------------------------+
```

### 各分区详细说明

**Eden 区（伊甸园区）**
- 新创建的小对象首先分配在 Eden 区
- Eden 区满时触发 Minor GC

**Survivor 区（幸存者区）**
- 有 S0 和 S1 两个区（也叫 From 和 To）
- Minor GC 后，存活对象从 Eden 移到 Survivor
- 每次 Minor GC，存活对象在 S0 和 S1 之间来回复制
- 对象年龄（Age）每熬过一次 Minor GC + 1

**老年代（Old Generation）**
- 存放长期存活的对象（年龄超过阈值，默认15）
- 大对象直接进入老年代（避免 Eden 区复制开销）
- 老年代满时触发 Full GC

### 堆参数配置

`ash
-Xms512m    # 堆初始大小
-Xmx2g      # 堆最大大小（建议 Xms=Xmx，避免频繁扩缩容）
-Xmn256m    # 新生代大小
-XX:NewRatio=2  # 老年代:新生代 = 2:1（新生代占堆的1/3）
-XX:SurvivorRatio=8  # Eden:S0:S1 = 8:1:1
`

---

## 3.6 方法区（Method Area）

### 存储内容

方法区是**线程共享**的内存区域，存储：
- **已加载类的元数据**（类名、父类、接口、字段、方法等）
- **运行时常量池**（字面量、符号引用）
- **静态变量**
- **即时编译器编译后的代码缓存**

**类比**：方法区是图书馆的「馆藏目录系统」，记录所有书籍（类）的信息、索引和摘要，而不是书籍本身。

### JDK7 永久代 vs JDK8 元空间

```
JDK7：永久代（PermGen）
+--------------------------------------------------+
|                      JVM 堆                      |
|  +-----------+  +--------+  +----------------+  |
|  |    新生代   |  |  老年代  |  | 永久代(PermGen) |  |
|  |            |  |        |  | 大小: -XX:MaxPermSize|  |
|  |            |  |        |  | 默认: 64M~256M  |  |
|  +-----------+  +--------+  +----------------+  |
+--------------------------------------------------+

JDK8：元空间（Metaspace）
+--------------------------------------------------+
|                      JVM 堆                      |
|  +----------------------------+  +-----------+  |
|  |          新生代              |  |     老年代   |  |
|  +----------------------------+  +-----------+  |
+--------------------------------------------------+
+--------------------------------------------------+
|         元空间（Metaspace）— 本地内存（Native）       |
|         大小：-XX:MaxMetaspaceSize（默认不限制）       |
+--------------------------------------------------+
```

### 为什么 JDK8 要用元空间替换永久代？

1. **永久代大小固定**：MaxPermSize 默认值较小，加载大量类时容易出现 PermGen space OOM
2. **永久代 GC 困难**：永久代的 GC 效率低，Full GC 才能回收，而 Full GC 代价很高
3. **内存融合计划**：Oracle 计划将 HotSpot 与 JRockit（没有永久代）合并，统一使用本地内存存储元数据
4. **元空间动态扩展**：使用本地内存，不受 JVM 堆大小限制，理论上不会再出现 PermGen OOM

---

## 3.7 运行时常量池

运行时常量池是方法区的一部分，存储编译期生成的**字面量和符号引用**。

```java
// 示例：字符串字面量
String s1 = "hello";      // 存在常量池中
String s2 = "hello";      // 复用常量池中的同一个
String s3 = new String("hello"); // 在堆中新建对象

System.out.println(s1 == s2);  // true（都指向常量池同一对象）
System.out.println(s1 == s3);  // false（s3在堆中，s1在常量池）
System.out.println(s1 == s3.intern()); // true（intern()返回常量池引用）
```

**String.intern() 的原理**：
1. 查找常量池中是否存在等值字符串
2. 如果存在，返回常量池中的引用
3. 如果不存在，将该字符串加入常量池（JDK7+），返回引用

---

## 3.8 直接内存（Direct Memory）

直接内存不是 JVM 运行时数据区的一部分，但 Java NIO 会频繁使用。

```java
// Java NIO 使用直接内存（堆外内存）
ByteBuffer buffer = ByteBuffer.allocateDirect(1024); // 分配 1KB 直接内存
// 直接内存由 OS 管理，不受 GC 管理
// 避免了 JVM 堆内存和本地内存之间的数据复制
// 适合大数据量的 I/O 操作
```

**优点**：减少数据从 JVM 堆到操作系统的拷贝次数，提升 I/O 性能
**缺点**：不受 GC 管理，需要手动释放；分配和释放代价较高

`ash
-XX:MaxDirectMemorySize=256m  # 设置直接内存最大值
`

---

## 3.9 各内存区域可能抛出的异常

| 内存区域 | 可能异常 | 触发场景 |
|---------|---------|---------|
| 程序计数器 | 无 | 唯一不会OOM的区域 |
| 虚拟机栈 | StackOverflowError | 栈深度超过限制（如无限递归） |
| 虚拟机栈 | OutOfMemoryError | 动态扩展时内存不足（创建大量线程） |
| 本地方法栈 | StackOverflowError / OOM | 同虚拟机栈 |
| 堆 | OOM: Java heap space | 对象过多，堆放不下 |
| 方法区 | OOM: PermGen space | JDK7，加载类过多 |
| 方法区 | OOM: Metaspace | JDK8，元空间耗尽 |
| 直接内存 | OOM: Direct buffer memory | 直接内存分配过多 |

---

## 3.10 对象在内存中的布局

```
对象内存布局：

+-----------------------------------------------------------+
|                     对象头（Header）                        |
|  +-----------------------------------------------------+  |
|  |                 Mark Word（8字节）                   |  |
|  |  存储：hashCode / GC年龄 / 锁状态 / 线程ID 等          |  |
|  |  +----------+--------+---------------------------+  |  |
|  |  |  hashCode | GC分代 |       锁状态              |  |  |
|  |  | (25 bit)  | (4bit) |  (2bit=无锁/偏向/轻量/重量)|  |  |
|  |  +----------+--------+---------------------------+  |  |
|  +-----------------------------------------------------+  |
|  +-----------------------------------------------------+  |
|  |          类型指针（4字节，开启压缩指针）                 |  |
|  |          指向该对象的 Class 对象                       |  |
|  +-----------------------------------------------------+  |
|  （如果是数组，还有数组长度字段，4字节）                       |
+-----------------------------------------------------------+
|                   实例数据（Instance Data）                 |
|  存储对象实际的字段值                                       |
|  如：int age = 18; String name = "Java"; 等               |
+-----------------------------------------------------------+
|                   对齐填充（Padding）                      |
|  JVM 要求对象大小必须是 8 字节的整数倍                        |
|  不够的用 0 填充                                          |
+-----------------------------------------------------------+
```

### Mark Word 的状态变化

```
Mark Word 在不同锁状态下的存储内容（64位JVM）：

无锁状态：
  [  hashCode(25bit)  |  GC年龄(4bit)  |  偏向锁位(1=0)  |  锁标志(2=01)  ]

偏向锁：
  [  线程ID(54bit)  |  Epoch(2bit)  |  偏向锁位(1=1)  |  锁标志(2=01)  ]

轻量级锁：
  [         栈中锁记录的指针(62bit)         |  锁标志(2=00)  ]

重量级锁：
  [      指向互斥量的指针(62bit)             |  锁标志(2=10)  ]

GC标志：
  [                 空(62bit)              |  锁标志(2=11)  ]
```

---

## 3.11 对象的创建过程

```
JVM 创建对象的完整步骤（Object obj = new Object()）：

步骤1：类检查
  - 检查该类是否已被加载、解析、初始化
  - 如果未加载，先执行类加载流程

步骤2：分配内存
  - 确定对象需要的内存大小（类加载完成后就确定了）
  - 分配方式：
    指针碰撞（Bump the Pointer）：堆内存连续整齐，移动分界指针
    空闲列表（Free List）：堆内存不连续，维护可用内存列表
  - 线程安全：CAS + 失败重试 / TLAB（线程本地分配缓冲区）

步骤3：初始化零值
  - 将分配的内存空间初始化为零值（int=0, ref=null, boolean=false...）

步骤4：设置对象头
  - 填充 Mark Word：对象的hashCode、GC年龄、锁状态
  - 填充类型指针：指向该类的 Class 对象

步骤5：执行 init 方法（<init>）
  - 执行构造函数，按程序员的意愿初始化对象字段
```
---

# Part 4 - 垃圾回收（GC）机制

## 4.1 什么是垃圾？

**垃圾**：不再被任何活跃引用指向的对象，即「死对象」。

**类比**：家里不再使用的东西就是垃圾，如果所有人都不再需要某件东西（没有任何引用指向它），就可以当垃圾处理了。

---

## 4.2 如何判断对象是垃圾

### 方法一：引用计数法（Reference Counting）

**原理**：给每个对象添加一个计数器，有引用+1，引用失效-1，计数器为0则是垃圾。

```java
Object a = new Object(); // a 的引用计数 = 1
Object b = a;            // a 的引用计数 = 2
a = null;                // a 的引用计数 = 1
b = null;                // a 的引用计数 = 0 -> 可以回收
```

**缺点**：无法解决**循环引用**问题

```java
// 循环引用导致内存泄漏
class Node { Node next; }

Node n1 = new Node();  // n1引用计数=1
Node n2 = new Node();  // n2引用计数=1

n1.next = n2;  // n2引用计数=2
n2.next = n1;  // n1引用计数=2

n1 = null;     // n1引用计数=1（还有n2.next指向它）
n2 = null;     // n2引用计数=1（还有n1.next指向它）

// 两个对象引用计数都是1，但实际上无法访问
// 引用计数法无法识别为垃圾！
```

**现状**：Java 没有使用引用计数法（Python 使用）

### 方法二：可达性分析算法（Reachability Analysis）

**原理**：从一组「GC Roots」出发，通过引用链遍历，能到达的对象是存活的，不能到达的是垃圾。

**类比**：就像警察寻人——从已知的人（GC Roots）出发，找他们认识的人（引用），再找认识的认识的人，最终所有能联系到的人都是活着的，联系不到的就是「失踪」的。

```
GC Roots 出发的引用链：

  GC Roots（虚拟机栈/静态变量/常量/本地方法栈引用等）
             |
    +--------v--------+
    |    Object A     |   <- 可达（存活）
    |  +----------+   |
    +--| Object B |---+   <- 可达（存活）
       +----+-----+
            |
       +----v-----+
       | Object C |       <- 可达（存活）
       +----------+

       +----------+
       | Object D |       <- 不可达（垃圾！）
       +----+-----+
            |
       +----v-----+
       | Object E |       <- 不可达（垃圾！）
       +----------+
       （D和E即使互相引用，也是垃圾）
```

**GC Roots 包括**：
1. 虚拟机栈（栈帧中的局部变量表）中引用的对象
2. 方法区中类静态属性引用的对象（如 static Object obj）
3. 方法区中常量引用的对象（如 final static Object CONST）
4. 本地方法栈中 JNI 引用的对象
5. Java 虚拟机内部引用（基本数据类型的 Class 对象、常驻异常对象）
6. 所有被同步锁（synchronized）持有的对象

---

## 4.3 四种引用类型

Java 提供了四种引用强度，从强到弱：

### 强引用（Strong Reference）

```java
// 最常见的引用
Object obj = new Object(); // 强引用
// 只要强引用存在，GC 永远不会回收该对象
// 即使 OOM 也不会回收
```

### 软引用（Soft Reference）

```java
// 内存不足时才会被 GC 回收
SoftReference<byte[]> softRef = new SoftReference<>(new byte[1024 * 1024]);

// 典型场景：内存敏感的缓存（如图片缓存）
Map<String, SoftReference<Image>> imageCache = new HashMap<>();
```

### 弱引用（Weak Reference）

```java
// 不管内存是否足够，下次 GC 时一定会被回收
WeakReference<Object> weakRef = new WeakReference<>(new Object());
Object obj = weakRef.get(); // 可能返回 null（已被GC）

// 典型场景：ThreadLocal 的 Entry 继承 WeakReference
WeakHashMap<Object, String> map = new WeakHashMap<>();
```

### 虚引用（Phantom Reference）

```java
// 不影响对象的生命周期，完全无法通过虚引用获取对象
// 唯一用途：对象被回收时收到通知
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantomRef = new PhantomReference<>(new Object(), queue);

System.out.println(phantomRef.get()); // 永远返回 null

// 典型场景：DirectByteBuffer 的堆外内存回收通知（Cleaner 机制）
```

### 四种引用对比表

| 引用类型 | GC 回收时机 | 用途 | 获取对象 |
|---------|-----------|------|---------|
| 强引用 | 永不（OOM也不） | 普通对象 | 直接使用 |
| 软引用 | 内存不足时 | 内存敏感缓存 | get() 可能为 null |
| 弱引用 | 每次 GC | WeakHashMap/ThreadLocal | get() 很快变 null |
| 虚引用 | 任何时候 | 对象回收通知/堆外内存管理 | get() 永远 null |

---

## 4.4 三大垃圾回收算法

### 算法一：标记-清除算法（Mark-Sweep）

```
执行过程：

Step 1 - 标记阶段：  [存] = 存活对象, [垃] = 垃圾对象
  +---------------------------------------------------+
  | [存] [垃] [存] [垃] [垃] [存] [垃] [存] [垃]       |
  +---------------------------------------------------+

Step 2 - 清除阶段：回收未标记的垃圾对象
  +---------------------------------------------------+
  | [存]       [存]           [存]       [存]          |
  +---------------------------------------------------+
  （注意：清除后留下大量碎片！）
```

**缺点**：
- 执行效率低（需要遍历整个堆两次）
- **内存碎片化**：清除后产生大量不连续碎片，无法分配大对象

### 算法二：标记-整理算法（Mark-Compact）

```
Step 1 - 标记（同上）

Step 2 - 整理（Compact）：将存活对象向一端移动，清除边界以外的内存

  Before:
  +---------------------------------------------------+
  | [存]       [存]           [存]       [存]          |
  +---------------------------------------------------+

  After:
  +---------------------------------------------------+
  | [存] [存] [存] [存]                                |
  +---------------------------------------------------+
              ^
              边界指针，新对象从这里开始分配（指针碰撞）
```

**优点**：不产生碎片，内存分配使用指针碰撞效率高
**缺点**：移动对象时需要更新所有引用，代价较高，适合老年代

### 算法三：复制算法（Copying）

```
执行过程（新生代 Eden + Survivor 实现）：

初始状态：
  +---------------------------+  +----------+  +----------+
  |         Eden              |  |    S0     |  |    S1     |
  |  [存] [垃] [存] [垃] [存]  |  |  （空）    |  |  （空）    |
  +---------------------------+  +----------+  +----------+

Minor GC 执行：存活对象全部复制到 S1，Eden 和 S0 完全清空
  +---------------------------+  +----------+  +----------+
  |         Eden（清空）        |  | S0（清空）  |  | S1[存][存]|
  +---------------------------+  +----------+  +----------+

  下次 Minor GC：
  Eden + S1 的存活对象复制到 S0，然后清空 Eden 和 S1
  （S0和S1来回交替使用）
```

**优点**：
- 只需扫描存活对象（而非整个堆），效率高
- 复制后内存整齐，没有内存碎片

**为什么新生代用复制算法**：
新生代的特点是「**朝生暮死**」——绝大多数（约 98%）对象创建后很快变为垃圾，每次 Minor GC 后存活的对象极少，复制成本很低。
JVM 利用 Eden:S0:S1 = 8:1:1 的比例，有效内存利用率达到 90%。

---

## 4.5 分代收集理论

**分代假说**（Generational Hypothesis）：
- **弱分代假说**：绝大多数对象都是「朝生暮死」的
- **强分代假说**：熬过越多次垃圾收集的对象，就越难以消亡
- **跨代引用假说**：跨代引用相对于同代引用来说仅占少数

```
分代收集策略：

新生代（Young Generation）：
  特点：对象死亡率高（>98%），朝生暮死
  算法：复制算法（高效）
  GC类型：Minor GC / Young GC
  频率：高（几分钟甚至几秒一次）

老年代（Old Generation）：
  特点：对象存活率高，难以消亡
  算法：标记-清除 或 标记-整理
  GC类型：Major GC / Full GC
  频率：低（几小时甚至几天一次）
```

---

## 4.6 Minor GC / Major GC / Full GC 区别

| GC 类型 | 回收区域 | 触发条件 | STW 时间 |
|---------|---------|---------|---------|
| Minor GC | 新生代（Eden + Survivor） | Eden 区满 | 短（毫秒级） |
| Major GC | 老年代 | 老年代空间不足 | 长（秒级） |
| Full GC | 整个堆 + 方法区 | 多种条件（见下） | 最长（秒~分钟级） |

**Full GC 触发条件**：
1. 调用 System.gc()（建议回收，JVM可忽略）
2. 老年代空间不足（存放大对象或晋升对象时）
3. 方法区（元空间）空间不足
4. 空间分配担保失败（Minor GC 前检查老年代剩余空间）
5. JDK7 中的 CMS GC 出现 Concurrent Mode Failure

---

## 4.7 对象晋升老年代的过程

```
对象从出生到晋升老年代的完整过程：

1. 对象诞生：在 Eden 区分配
2. 经过多次 Minor GC，年龄累积
   - 每次 Minor GC，存活对象年龄 +1
   - 年龄阈值默认 15（-XX:MaxTenuringThreshold）
3. 年龄超过阈值，晋升到老年代

特殊情况 - 直接进老年代：
  - 大对象（超过 -XX:PretenureSizeThreshold 字节）直接进老年代
  - 动态年龄判断：
    若 Survivor 区中相同年龄所有对象大小的总和 > Survivor 空间的一半
    则年龄 >= 该年龄的对象直接晋升老年代

对象晋升流程图：

Eden 区满
    | Minor GC
    v
存活？-> 否 -> 直接回收
  | 是
  v
大对象？-> 是 -> 直接进老年代
  | 否
  v
年龄 >= 15？-> 是 -> 进老年代
  | 否
  v
动态年龄判断通过？-> 是 -> 进老年代
  | 否
  v
Survivor 区能放下？-> 是 -> 进 Survivor（年龄+1）
  | 否
  v
进老年代（空间分配担保）
```

---

## 4.8 STW（Stop The World）

### 什么是 STW？

STW（Stop The World）是指 GC 执行期间，**JVM 必须暂停所有应用线程**，只留 GC 线程工作。

**类比**：你在整理仓库（GC），但仓库里的工人（应用线程）不停地搬东西，你根本数不清有多少东西。所以必须让所有工人先停下来（STW），你整理完了大家再继续工作。

### 为什么必须 STW？

```
如果不 STW，GC 标记过程中可能出现：

  时刻1：GC 扫描，对象 C 不可达，标记为垃圾
  时刻2：应用线程建立了 A -> B -> C 的引用链
  时刻3：GC 回收了 C（但 C 现在是存活的！）
  -> 这是灾难性的错误，会导致程序崩溃！

  三色标记法（黑白灰）用来解决并发标记问题：
  白色：未被标记（可能是垃圾）
  灰色：已标记，但其引用的对象还没全部标记
  黑色：已标记，且引用的所有对象也已标记
```

### 减少 STW 的方法

现代 GC（CMS、G1、ZGC）的设计目标之一就是**缩短 STW 时间**：
- **CMS**：并发标记，STW只在初始标记和重新标记阶段
- **G1**：可预测停顿，STW时间可配置
- **ZGC**：几乎所有阶段并发，STW通常不超过10ms
---

# Part 5 - 垃圾收集器详解

## 5.1 垃圾收集器总览

```
JVM 垃圾收集器发展史：

Serial(1998) -> Parallel(2002) -> CMS(2008) -> G1(2017正式) -> ZGC(2018)

收集器对应关系（新生代 + 老年代可组合使用）：

新生代收集器：                   老年代收集器：
+--------------------+          +--------------------+
|  Serial             |----------| Serial Old          |
|  ParNew             |----------| CMS                 |
|  Parallel Scavenge  |----------| Parallel Old        |
+--------------------+          +--------------------+

              G1（整堆收集器，无需配对）
              ZGC（整堆收集器）
              Shenandoah（整堆收集器）
```

---

## 5.2 Serial GC（串行收集器）

使用**单线程**进行垃圾收集，收集时必须 STW。

```
Serial GC 执行时序：

应用线程1: ==========|               |=============
应用线程2: ==========|   STW 停止    |=============
应用线程3: ==========|               |=============
GC线程:              -------GC-------
                     ^               ^
                     暂停             恢复
```

- 新生代：Serial（复制算法）
- 老年代：Serial Old（标记-整理算法）
- **最简单**的 GC 实现，单线程串行执行
- 适用：单核 CPU 或内存较小的嵌入式/桌面应用；小堆（< 100MB）

`ash
-XX:+UseSerialGC  # 开启 Serial GC
`

---

## 5.3 Parallel GC（并行收集器）

使用**多线程**并行进行垃圾收集，但仍需 STW。

```
Parallel GC 执行时序：

应用线程1: ==========|               |=============
应用线程2: ==========|   STW 停止    |=============
GC线程1:             ------GC-----
GC线程2:             ------GC-----
GC线程3:             ------GC-----
GC线程4:             ------GC-----
                     ^             ^
                     暂停           恢复（多线程更快）
```

- 新生代：Parallel Scavenge（复制算法）
- 老年代：Parallel Old（标记-整理算法）
- **JDK8 的默认 GC**
- 适用：批处理、科学计算、数据挖掘等吞吐量优先的后台应用

`ash
-XX:+UseParallelGC         # 开启 Parallel GC（JDK8默认）
-XX:ParallelGCThreads=4    # 设置并行GC线程数
-XX:GCTimeRatio=99         # GC时间比例（1/(1+99) = 1%的GC时间）
-XX:MaxGCPauseMillis=200   # 最大停顿时间（ms）
`

---

## 5.4 CMS（Concurrent Mark Sweep）

CMS 是第一款真正意义上的**并发收集器**，以获取最短停顿时间为目标。

### CMS 四个阶段详解

```
CMS 执行时序：

应用线程: ====|     |=============================|     |====
              | STW |  并发（与应用线程同时运行）    | STW |
GC线程:       |-----|                             |-----|
         初始标记  并发标记                    重新标记  并发清除
         短暂STW   长（无STW）               短暂STW   长（无STW）
```

**阶段一：初始标记（Initial Mark）** —— 短暂STW
- 仅标记 GC Roots **直接关联**的对象，速度很快

**阶段二：并发标记（Concurrent Mark）** —— 并发，无STW
- 从初始标记的对象出发，遍历整个引用链
- 与应用线程**并发**执行，时间最长但不影响应用

**阶段三：重新标记（Remark）** —— 短暂STW
- 修正并发标记期间，由于用户线程继续运行而导致标记变动的那部分对象

**阶段四：并发清除（Concurrent Sweep）** —— 并发，无STW
- 并发清除已死对象，释放内存

### CMS 优缺点

**优点**：并发收集，低停顿（STW时间短），适合互联网应用

**缺点**：
1. **CPU 资源敏感**：并发阶段占用 CPU，降低吞吐量
2. **浮动垃圾（Floating Garbage）**：并发清除阶段产生的新垃圾无法在本次 GC 中处理
   因此需要预留空间：-XX:CMSInitiatingOccupancyFraction=68
3. **内存碎片**：标记-清除算法不整理内存

`ash
-XX:+UseConcMarkSweepGC              # 开启 CMS（JDK9已标记弃用）
-XX:CMSInitiatingOccupancyFraction=70  # 老年代使用70%时触发CMS
-XX:+UseCMSCompactAtFullCollection     # Full GC 时开启内存整理
-XX:CMSFullGCsBeforeCompaction=5       # 5次Full GC后进行一次整理
`

---

## 5.5 G1 GC（Garbage First）

### G1 的设计革命：Region 分区

G1 打破了传统的「新生代/老年代物理隔离」，将堆内存划分为**若干等大的 Region（区域）**。

```
G1 堆内存模型：

+--+--+--+--+--+--+--+--+
|E |E |S |O |O |E |H |H |   E = Eden
+--+--+--+--+--+--+--+--+   S = Survivor
|O |O |E |E |O |S |E |O |   O = Old
+--+--+--+--+--+--+--+--+   H = Humongous（巨大对象）
|E |H |O |O |E |O |E |O |   （空白 = 未分配）
+--+--+--+--+--+--+--+--+
|O |E |E |  |O |  |S |E |
+--+--+--+--+--+--+--+--+

每个 Region 大小相同（1MB~32MB，2的幂次），
角色可以动态变化，不再固定分新生代/老年代区域
```

**Region 分区的好处**：
- GC 时可以选择**垃圾最多的 Region** 优先回收（Garbage First 的由来！）
- 可以设置停顿时间目标，GC 在时间允许的范围内尽量多回收
- 避免整堆扫描，提高效率

### G1 的四个阶段

```
G1 工作流程：

Phase 1: 年轻代 GC（Young GC）
  - STW
  - 收集所有 Eden 和 Survivor Region
  - 存活对象复制到新的 Survivor Region 或晋升到 Old Region

Phase 2: 并发标记（Concurrent Marking）
  - 初始标记（STW，短暂，随 Young GC 顺带完成）
  - 并发标记（无STW，与应用线程并发）
  - 最终标记（STW，修正并发标记期间的变化）
  - 清理（STW，统计每个Region的存活率，按回收价值排序）

Phase 3: 混合回收（Mixed GC）
  - STW
  - 同时收集年轻代所有Region + 部分高价值老年代Region
  - 「混合」指的是新生代+老年代一起回收

Phase 4: Full GC（兜底方案）
  - 当并发回收跟不上对象分配速度时触发
  - STW，单线程执行（G1的Full GC是弱点）
  - 尽量避免：通过调参减少Full GC触发概率
```

### Humongous 对象

当对象大小**超过 Region 大小的 50%** 时，被认为是 Humongous（巨大）对象：
- 直接分配在老年代的连续 Region（可能跨多个 Region）
- Humongous 对象的回收代价很高
- 如果 Humongous 对象过多，需要适当增大 -XX:G1HeapRegionSize

### 可预测停顿时间模型

G1 的核心优势：通过 -XX:MaxGCPauseMillis 设置期望最大停顿时间（默认200ms），G1 会动态调整每次收集的 Region 数量。

`ash
-XX:+UseG1GC                    # 开启 G1（JDK9+ 默认）
-XX:MaxGCPauseMillis=200        # 期望最大停顿时间200ms
-XX:G1HeapRegionSize=16m        # Region大小（1m-32m，2的幂次）
-XX:G1NewSizePercent=5          # 新生代最小占堆比例（默认5%）
-XX:G1MaxNewSizePercent=60      # 新生代最大占堆比例（默认60%）
-XX:InitiatingHeapOccupancyPercent=45  # 堆使用45%时开始并发标记
`

---

## 5.6 ZGC（Java 11+）

ZGC（Z Garbage Collector）是 JDK11 引入的低延迟垃圾收集器，目标是将 STW 时间控制在**10ms 以内**（不论堆大小）。

### ZGC 的三大核心技术

**① 着色指针（Colored Pointers）**

```
64位指针布局：
[63-44位：空闲][43bit：Finalizable][42bit：Remapped][41bit：Marked1][40bit：Marked0][39-0位：实际地址(44位)]

通过读取指针的颜色位，GC可以知道对象的标记状态，不需要额外的标记表
```

**② 读屏障（Load Barrier）**

```java
Object obj = someRef.field; // ZGC 在这里插入读屏障
// 读屏障检查指针颜色，如果需要则触发修正操作
// 使得 ZGC 可以在不 STW 的情况下移动对象
```

**③ 并发整理**

ZGC 可以在**不停止应用线程**的情况下移动对象（通过读屏障保证一致性）。

### ZGC 适用场景

- 超大堆（TB 级别）+ 低延迟要求
- 金融交易系统、实时游戏服务器等对延迟极敏感的场景

`ash
-XX:+UseZGC  # 开启 ZGC（JDK11+）
`

---

## 5.7 各 GC 收集器对比表

| 收集器 | 算法 | 是否并发 | STW时长 | 适用场景 | JDK版本 |
|--------|------|---------|---------|---------|--------|
| Serial | 复制+标记整理 | 否 | 长 | 单核/客户端/小堆 | JDK1.3+ |
| ParNew | 复制 | 并行 | 较长 | 配合CMS使用 | JDK1.4+ |
| Parallel Scavenge | 复制+标记整理 | 并行 | 较长 | 吞吐量优先/批处理 | JDK1.4+ |
| CMS | 标记清除 | 大部分并发 | 短 | 低延迟/互联网服务 | JDK1.5+ |
| G1 | 复制+标记整理 | 部分并发 | 可控 | 大堆/均衡延迟与吞吐 | JDK9默认 |
| ZGC | 复制 | 几乎全并发 | <10ms | 超大堆/极低延迟 | JDK11+ |
| Shenandoah | 复制 | 几乎全并发 | <10ms | 类似ZGC | JDK12+ |

---

## 5.8 如何选择垃圾收集器

```
GC 选择决策树：

堆 < 100MB？
  -> 是 -> Serial GC (-XX:+UseSerialGC)

单线程或客户端应用？
  -> 是 -> Serial GC

吞吐量优先（批处理等）？
  -> 是 -> Parallel GC (-XX:+UseParallelGC) [JDK8默认]

低延迟要求（<200ms停顿可接受）？
  -> 是, JDK8 -> CMS (-XX:+UseConcMarkSweepGC)
  -> 是, JDK9+ -> G1 (-XX:+UseG1GC) [JDK9+默认，推荐]

超低延迟（<10ms）或超大堆（>32GB）？
  -> 是 -> ZGC (-XX:+UseZGC) 或 Shenandoah
```
---

# Part 6 - JVM 调优实战

## 6.1 JVM 调优目标

JVM 调优是一个需要在三个目标之间找到平衡点的过程：

- **吞吐量（Throughput）**：用户代码运行时间占总时间的比例
- **延迟（Latency）**：单次 GC 停顿时间（STW 时间）
- **内存占用（Memory Footprint）**：JVM 进程占用的总内存

三者不可兼得：提升吞吐量就会优先低频大GC；降低延迟就需要频繁小规模GC；减少内存删则会导致GC更频繁。

---

## 6.2 常用 JVM 参数分类详解

### 堆大小参数

```bash
-Xms512m          # 堆最小值（初始大小）
-Xmx2g            # 堆最大值（建议：Xms=Xmx）
-Xmn256m          # 新生代大小
-XX:MetaspaceSize=256m     # 元空间初始大小
-XX:MaxMetaspaceSize=512m  # 元空间最大大小
-XX:MaxDirectMemorySize=256m  # 直接内存最大值
```

### 新生代/老年代比例参数

```bash
-XX:NewRatio=2       # 老年代:新生代 = 2:1（默认）
-XX:SurvivorRatio=8  # Eden:S0:S1 = 8:1:1（默认）
```

### GC 选择参数

```bash
-XX:+UseSerialGC          # Serial + Serial Old
-XX:+UseParallelGC        # Parallel（JDK8默认）
-XX:+UseConcMarkSweepGC   # ParNew + CMS
-XX:+UseG1GC              # G1（JDK9+默认）
-XX:+UseZGC               # ZGC（JDK11+）
```

### GC 日志参数（完整配置）

```bash
# JDK8 及之前的 GC 日志参数：
-XX:+PrintGCDetails              # 打印详细GC日志
-XX:+PrintGCDateStamps           # 打印GC时间戳（日期格式）
-XX:+PrintGCTimeStamps           # 打印GC时间戳（相对启动时间）
-XX:+PrintHeapAtGC               # GC前后打印堆信息
-XX:+PrintTenuringDistribution   # 打印对象年龄分布
-Xloggc:/var/log/app/gc.log      # GC日志输出到文件
-XX:+UseGCLogFileRotation        # GC日志滚动
-XX:NumberOfGCLogFiles=10        # 保留10个日志文件
-XX:GCLogFileSize=100m           # 每个日志文件最大100MB

# JDK9+ 统一日志参数（-Xlog）：
-Xlog:gc*:file=/var/log/app/gc.log:time,level,tags:filecount=10,filesize=100m
```

### OOM 时自动 dump

```bash
-XX:+HeapDumpOnOutOfMemoryError          # OOM时自动生成堆快照
-XX:HeapDumpPath=/var/log/app/heap.hprof # 指定dump文件路径
-XX:OnOutOfMemoryError="sh /restart.sh"  # OOM时执行脚本（如自动重启）
```

---

## 6.3 GC 日志分析

### JDK8 GC 日志示例解读

```
# Minor GC 日志：
2024-01-15T10:30:00.000+0800: 123.456: [GC (Allocation Failure) 
  [PSYoungGen: 204800K->12800K(230400K)] 
  307200K->115200K(614400K), 
  0.0563085 secs] 
  [Times: user=0.19 sys=0.01, real=0.06 secs]

解读：
  Allocation Failure:       触发原因（分配失败，即Eden满了）
  PSYoungGen:               新生代收集器
  204800K->12800K(230400K): GC前204800K -> GC后12800K
  0.0563085 secs:           本次GC耗时56ms
  real=0.06:                实际停顿时间（STW时间）
```

```
# Full GC 日志：
2.35 secs: Full GC停顿2.35秒！（需要优化的信号）
```

---

## 6.4 JVM 调优步骤

```
JVM 调优四步法：

步骤1：监控（Monitor）
  - 使用 jstat/VisualVM/Arthas 观宾GC频率和停顿时间
  - Full GC 频率（正常：小时级别，异常：分钟级别）
  - Minor GC 频率（正常：几十秒一次，异常：每秒多次）

步骤2：分析（Analyze）
  - 分析 GC 日志（GCViewer/GCEasy在线工具）
  - 使用 jmap 生成堆快照，MAT 分析内存泄漏

步骤3：优化（Optimize）
  - 代码层面：减少对象创建、避免大对象、及时释放引用
  - 参数层面：合理设置堆大小、调整新老年代比例、选合适GC
  - 架构层面：增加机器内存、优化业务逻辑

步骤4：验证（Verify）
  - 压测验证优化效果
  - 对比优化前后的 GC 日志
```

---

## 6.5 常见 OOM 场景及解决方案

### OOM 1：Java heap space

**现象**：java.lang.OutOfMemoryError: Java heap space

**原因**：堆大小不够 / 内存泄漏 / 大量大对象

```bash
-XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof
# 用 MAT 分析 heap.hprof 找出大对象和内存泄漏
-Xmx4g  # 临时增加堆大小
```

### OOM 2：PermGen space / Metaspace

**现象**：java.lang.OutOfMemoryError: PermGen space（JDK7）或 Metaspace（JDK8+）

```bash
-XX:MaxPermSize=512m       # JDK7
-XX:MaxMetaspaceSize=512m  # JDK8+
```

### OOM 3：Unable to create new native thread

**现象**：java.lang.OutOfMemoryError: unable to create new native thread

```bash
ulimit -u  # Linux 查看线程数限制
echo "* soft nproc 65535" >> /etc/security/limits.conf
-Xss128k   # 减小线程栈大小
# 代码层面：使用线程池
ExecutorService pool = Executors.newFixedThreadPool(100);
```

### OOM 4：Direct buffer memory

```bash
-XX:MaxDirectMemorySize=1g
# ((DirectBuffer) byteBuffer).cleaner().clean();
```

### OOM 5：GC overhead limit exceeded

**触发条件**：JVM 花费超过 98% 的时间做GC，但回收内存不足 2%，连续发生5次

```bash
-Xmx4g  # 增大堆大小
-XX:-UseGCOverheadLimit  # 关闭该检查（不推荐）
```

---

# Part 7 - JVM 监控与诊断工具

## 7.1 jps（查看 Java 进程）

```bash
# 查看当前机器上所有 Java 进程
$ jps
12345 Main          # PID 进程名
23456 Jps           # jps 本身
34567 Bootstrap     # Tomcat

$ jps -l   # 显示完整类名
$ jps -v   # 显示JVM参数
$ jps -m   # 显示main方法参数

# 示例：
$ jps -lv
12345 com.example.Main -Xmx2g -Xms512m -XX:+UseG1GC
```

---

## 7.2 jstat（GC 统计监控）

jstat 是监控 JVM GC 状态的利器，可以实时查看各内存区域的使用情况和 GC 统计。

```bash
# 最常用：每1000ms输出一次GC统计，共输出10次
$ jstat -gcutil 12345 1000 10

# 输出示例（-gcutil 显示使用率百分比）：
S0     S1     E      O      M     CCS    YGC   YGCT   FGC  FGCT    GCT
0.00  76.20  65.43  30.21  95.12  92.34  123   3.456    2   1.234   4.690

# 字段含义：
# S0:   Survivor0 使用率（76.20%）
# S1:   Survivor1 使用率（0%，S0和S1交替使用）
# E:    Eden 使用率（65.43%）
# O:    Old 老年代使用率（30.21%）
# M:    Metaspace 元空间使用率（95.12%）—— 过高！需要关注
# YGC:  Young GC 次数（123次）
# YGCT: Young GC 总耗时（3.456秒）
# FGC:  Full GC 次数（2次）
# FGCT: Full GC 总耗时（1.234秒）
# GCT:  所有 GC 总耗时（4.690秒）

$ jstat -gc 12345 1000     # 显示各区大小（KB）和GC统计
$ jstat -gccapacity 12345  # 显示各区容量
$ jstat -gccause 12345     # 显示GC原因
$ jstat -class 12345       # 类加载统计
$ jstat -compiler 12345    # JIT编译统计
```

---

## 7.3 jmap（堆内存快照）

```bash
# 查看堆内存使用摘要
$ jmap -heap 12345

# 查看存活对象的数量和大小（触发一次Full GC）
$ jmap -histo:live 12345 | head -20

# 输出示例（按内存占用排序）：
# num     #instances         #bytes  class name
# ----------------------------------------------
#   1:       1234567      123456789  [B（byte数组）
#   2:        234567       56789012  java.lang.String
#   3:        123456       12345678  java.util.HashMap\$Node

# 生成堆快照（heap dump）文件，用于 MAT 分析
$ jmap -dump:live,format=b,file=/tmp/heap.hprof 12345
# live:     只 dump 存活对象（触发Full GC）
# format=b: 二进制格式

# 注意：jmap -dump 会触发 STW，生产环境谨慎使用
# 推荐：配置 -XX:+HeapDumpOnOutOfMemoryError 在OOM时自动dump
```

---

## 7.4 jstack（线程堆栈）

```bash
# 打印所有线程堆栈信息
$ jstack 12345

# 输出示例：
# "http-nio-8080-exec-1" #25 daemon prio=5 os_prio=0 tid=0x... nid=0x1234 waiting
#    java.lang.Thread.State: WAITING (parking)
#         at sun.misc.Unsafe.park(Native Method)
#         at java.util.concurrent.LinkedBlockingQueue.take(...)

# 死锁检测
$ jstack 12345 | grep -A 30 "deadlock"

# jstack 输出中的死锁示例：
# Found one Java-level deadlock:
# =============================
# "Thread-1":
#   waiting to lock monitor 0x... (a java.lang.Object),
#   which is held by "Thread-0"
# "Thread-0":
#   waiting to lock monitor 0x... (a java.lang.Object),
#   which is held by "Thread-1"
```

### 线程状态说明

| 状态 | 含义 |
|------|------|
| RUNNABLE | 运行中或等待操作系统调度 |
| BLOCKED | 等待获取 synchronized 锁 |
| WAITING | 无限等待（Object.wait()/Thread.join()/LockSupport.park()） |
| TIMED_WAITING | 限时等待（sleep/wait(timeout)/join(timeout)） |
| TERMINATED | 已终止 |

---

## 7.5 jinfo（JVM 参数查看/动态修改）

```bash
# 查看 JVM 启动参数
$ jinfo -flags 12345

# 查看单个参数的值
$ jinfo -flag MaxHeapSize 12345
# -XX:MaxHeapSize=2147483648

# 动态修改可写参数
$ jinfo -flag +PrintGCDetails 12345   # 动态开启GC详细日志
$ jinfo -flag -PrintGCDetails 12345   # 动态关闭
$ jinfo -flag MaxHeapFreeRatio=80 12345  # 修改参数值

# 查看系统属性
$ jinfo -sysprops 12345
```

---

## 7.6 jcmd（多功能诊断命令）

jcmd 是 JDK7+ 引入的多功能诊断命令，可以替代很多工具。

```bash
$ jcmd                               # 列出所有Java进程
$ jcmd 12345 help                    # 查看进程支持的命令
$ jcmd 12345 GC.heap_dump /tmp/heap.hprof  # 生成堆快照
$ jcmd 12345 GC.heap_info            # 查看GC统计
$ jcmd 12345 Thread.print            # 查看线程堆栈
$ jcmd 12345 GC.run                  # 触发一次GC
$ jcmd 12345 VM.flags                # 查看JVM启动参数
$ jcmd 12345 VM.system_properties    # 查看系统属性
$ jcmd 12345 VM.classloader_stats    # 查看类加载器统计
$ jcmd 12345 GC.class_histogram      # 查看类直方图（不触发GC）
```

---

## 7.7 VisualVM 使用指南

VisualVM 是一个图形化的 JVM 监控工具，JDK 自带（$JAVA_HOME/bin/jvisualvm）。

```
VisualVM 功能概览：

1. 概述（Overview） - JVM版本、启动参数、系统属性

2. 监控（Monitor）
   - CPU 使用率实时图表
   - 堆/元空间内存使用实时图表
   - 线程数实时图表
   - 类加载数量统计

3. 线程（Threads）
   - 所有线程状态一览
   - 线程快照（相当于jstack）

4. 抽样（Sampler）
   - CPU 抽样：找出热点方法
   - 内存抽样：找出内存大户

5. 堆 Dump 分析（Heap Dump）
   - 对象占用排名
   - 对象实例列表
   - 引用链分析
```

连接远程 JVM 需要开启 JMX：

```bash
-Dcom.sun.management.jmxremote
-Dcom.sun.management.jmxremote.port=9999
-Dcom.sun.management.jmxremote.authenticate=false
-Dcom.sun.management.jmxremote.ssl=false
-Djava.rmi.server.hostname=192.168.1.100
# 在 VisualVM 中：文件 -> 添加JMX连接 -> 192.168.1.100:9999
```

---

## 7.8 Arthas（阿里开源诊断工具）

Arthas 是阿里巴巴开源的 Java 诊断工具，可以在不重启应用的情况下动态诊断问题。

```bash
# 下载并启动 Arthas
curl -O https://arthas.aliyun.com/arthas-boot.jar
java -jar arthas-boot.jar
java -jar arthas-boot.jar 12345  # 直接attach到进程
```

### 常用命令

**dashboard（实时面板）**
```bash
arthas> dashboard
# 输出实时面板：线程列表、内存使用、GC 统计、系统信息
```

**thread（线程分析）**
```bash
arthas> thread        # 查看所有线程
arthas> thread 1      # 查看线程1的堆栈
arthas> thread -b     # 查找死锁（block）
arthas> thread -n 3   # 查看占用CPU最多的3个线程
```

**jad（反编译）**
```bash
# 在线反编译字节码，无需源码
arthas> jad com.example.UserService
arthas> jad com.example.UserService getUserById  # 仅反编译指定方法
```

**watch（方法监控）**
```bash
# 监控方法的入参和返回值
arthas> watch com.example.UserService getUserById "{params,returnObj}" -x 2
# params:     方法入参
# returnObj:  方法返回值
# -x 2:       展开深度

# 条件过滤：只在方法耗时>100ms时打印
arthas> watch com.example.UserService getUserById "{params,returnObj}" '#cost>100'

# 监控异常
arthas> watch com.example.UserService getUserById "{params,throwExp}" -e
```

**trace（链路追踪）**
```bash
# 追踪方法内部的调用链路和每步耗时
arthas> trace com.example.UserService getUserById
# 输出：
# `---[12.5ms] com.example.UserService:getUserById()
#     `---[8.2ms] com.example.dao.UserDao:findById()
#         `---[7.9ms] org.hibernate.Session:get()
```

**ognl（执行表达式）**
```bash
# 在运行中的JVM执行OGNL表达式
arthas> ognl "@java.lang.System@getenv('HOME')"
arthas> ognl "@com.example.CacheManager@getInstance().getCacheSize()"
```

---

## 7.9 MAT（Memory Analyzer Tool）分析堆 dump 文件

MAT 是 Eclipse 基金会开源的堆内存分析工具。

```
使用步骤：

第一步：获取堆快照
  # 方法1：JVM OOM时自动生成
  -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/tmp/heap.hprof
  
  # 方法2：手动生成
  jmap -dump:live,format=b,file=/tmp/heap.hprof <pid>

第二步：用 MAT 打开 .hprof 文件

第三步：分析
  - Leak Suspects Report：MAT 自动分析可能的内存泄漏点
  - Dominator Tree：按内存占用排序，找出最大的内存大户
  - Retained Heap：若某对象被GC，会释放多少内存
  - OQL：SELECT * FROM java.lang.String s WHERE s.value.length > 1000
  - References：找到对象被谁引用（Incoming References）
```

```java
// 常见分析场景：静态Map导致的内存泄漏
// MAT 中，Dominator Tree 排名第一是 HashMap
// 展开后发现是 UserCache.cache（静态字段）持有了大量 User 对象

// 修复方案：
// 方案1：换用 SoftReference 或 WeakReference
private static Map<Long, SoftReference<User>> cache = new ConcurrentHashMap<>();

// 方案2：加过期策略（如 Guava Cache 或 Caffeine）
LoadingCache<Long, User> cache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(key -> userService.findById(key));
```

---

# Part 8 - 执行引擎与 JIT 编译

---

## 8.1 解释执行 vs 编译执行

Java 程序的执行方式是一个从"慢但通用"到"快但专用"的演进过程。

```
                     Java 源代码 (.java)
                            |
                     javac 编译
                            |
                     字节码文件 (.class)
                            |
                    JVM 类加载系统
                            |
                    ┌───────┴───────┐
                    │               │
              解释器执行         JIT 编译执行
           (Interpreter)       (JIT Compiler)
                    │               │
             逐条翻译字节码     编译为本地机器码
             慢，但启动快        快，但编译有开销
                    │               │
                    └───────┬───────┘
                            │
                      最终执行本地机器码
```

### 解释器（Interpreter）

**工作原理**：逐条读取字节码指令，翻译成机器码并立即执行。

**优点**：
- 启动速度快（不需要预先编译）
- 内存占用少

**缺点**：
- 每次执行同一段代码都要重新翻译，效率低

**类比**：就像有个翻译官在会议上逐句翻译，虽然能立刻开始，但每次说同一句话都要重新翻译。

### JIT 编译器（Just-In-Time Compiler）

**工作原理**：在程序运行时，将热点代码（频繁执行的代码）编译为本地机器码缓存，之后直接执行机器码。

**优点**：
- 热点代码执行速度接近 C/C++ 编译的原生程序

**缺点**：
- 编译本身有时间和内存开销
- 程序刚启动时，JIT 还未生效，性能不如稳定运行时

**类比**：就像同声传译员，第一次听到某段话需要思考，但多次听到相同内容后，已经准备好了直接说出来，速度极快。

---

## 8.2 热点代码检测

JIT 不会编译所有代码，只编译"热点代码"（Hot Code）—— 即频繁被调用或循环执行的代码。

### 热点探测方式

HotSpot VM 使用**基于计数器的热点探测**，有两种计数器：

#### 1. 方法调用计数器（Invocation Counter）

```
每次方法被调用时，计数器 +1
当计数器 ≥ 阈值（默认 10000 次）时，触发 JIT 编译
```

参数控制：
```bash
-XX:CompileThreshold=10000    # 方法调用触发JIT编译的阈值（默认10000）
```

#### 2. 回边计数器（Back Edge Counter）

专门用于监测循环代码。当循环体执行次数达到阈值，触发 OSR（On-Stack Replacement，栈上替换）编译。

```java
// 示例：这个循环会触发 OSR 编译
for (int i = 0; i < 1000000; i++) {
    // 循环体执行次数 ≥ 阈值 → 触发 JIT 编译此方法
    doSomething(i);
}
```

### 计数器衰减

为了避免一个很久以前调用频繁、但现在不常用的方法被编译，HotSpot 会定期对计数器进行**半衰期衰减**：

```
计数器值 = 计数器值 / 2   （每隔一段时间）
```

---

## 8.3 C1 与 C2 编译器

HotSpot VM 内置两个 JIT 编译器，各有侧重：

```
┌─────────────────────────────────────────────────────────────────┐
│                        JIT 编译器对比                            │
├──────────────┬──────────────────────┬───────────────────────────┤
│   特性        │    C1（Client）       │    C2（Server）            │
├──────────────┼──────────────────────┼───────────────────────────┤
│ 编译速度      │   快                  │   慢                       │
│ 编译质量      │   一般               │   高（更激进的优化）         │
│ 适用场景      │   桌面应用/短生命周期  │   服务端/长期运行应用        │
│ 启动参数      │   -client            │   -server                  │
│ 优化技术      │   简单方法内联等       │   全套激进优化               │
└──────────────┴──────────────────────┴───────────────────────────┘
```

---

## 8.4 分层编译（Tiered Compilation）

Java 7 引入、Java 8 默认开启。融合 C1 和 C2 的优点。

```
代码执行阶段
─────────────────────────────────────────────────────────────────▶
第0层：解释执行
  ↓ (调用次数少)
第1层：C1编译（无性能监控数据）
  ↓ (收集计数器数据)
第2层：C1编译（有少量性能监控数据）
  ↓ (收集完整监控数据)
第3层：C1编译（有完整性能监控数据）
  ↓ (方法足够热，C2接手)
第4层：C2编译（最终高度优化的机器码）
─────────────────────────────────────────────────────────────────▶
代码执行效率
```

**5个层次详解**：

| 层次 | 编译器 | 说明 |
|-----|--------|-----|
| 0   | 解释器  | 纯解释执行，收集性能信息 |
| 1   | C1     | 简单可靠的优化，无 profiling 信息 |
| 2   | C1     | 有限度的 profiling（方法/回边计数器）|
| 3   | C1     | 完整 profiling（含分支预测数据）|
| 4   | C2     | 激进优化，使用3层收集的全部 profiling 数据 |

相关 JVM 参数：
```bash
-XX:+TieredCompilation     # 开启分层编译（JDK8+ 默认开启）
-XX:-TieredCompilation     # 关闭分层编译
-XX:+PrintCompilation      # 打印 JIT 编译日志
```

---

## 8.5 JIT 优化技术详解

### 8.5.1 方法内联（Method Inlining）

**最重要**的 JIT 优化，将被调用方法的代码直接插入到调用处，消除方法调用开销。

```java
// 原始代码
int add(int a, int b) { return a + b; }
int result = add(3, 4);

// JIT 内联后（等效）
int result = 3 + 4;  // 直接计算，无方法调用开销
```

**触发条件**：
- 方法体字节码 ≤ 35 字节（`-XX:MaxInlineSize=35`）
- 方法调用足够频繁

**重要性**：很多其他优化（逃逸分析、常量折叠）都依赖方法内联先把代码展开。

---

### 8.5.2 逃逸分析（Escape Analysis）

JVM 分析对象的**作用域是否"逃逸"出当前方法或线程**，如果没有逃逸，则可以做更激进的优化。

```java
// 分析逃逸情况的代码示例
public int compute() {
    // Point 对象只在 compute() 内部使用，不返回、不传给其他线程
    // ↓ 这个对象"没有逃逸"！
    Point p = new Point(3, 4);
    return p.x + p.y;
}

class Point {
    int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
}
```

JVM 参数：
```bash
-XX:+DoEscapeAnalysis      # 开启逃逸分析（JDK6u23+ 默认开启）
-XX:-DoEscapeAnalysis      # 关闭（用于问题排查）
```

逃逸分析的三个关键优化：

---

### 8.5.3 栈上分配（Stack Allocation）

基于逃逸分析：若对象不逃逸，可以直接在**虚拟机栈**上分配，而不是堆上。

```
无逃逸分析：
  Point p = new Point(3,4);  →  在堆上分配  →  需要GC回收

有逃逸分析（栈上分配）：
  Point p = new Point(3,4);  →  在栈帧上分配  →  方法返回自动销毁，无GC压力
```

**好处**：
- 减少堆内存占用
- 减少 GC 压力
- 访问栈比访问堆更快

**验证示例**：
```java
// 开启逃逸分析时，下面代码几乎不触发 GC
public class StackAllocationDemo {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 10_000_000; i++) {
            alloc();  // 每次创建 Point，逃逸分析后在栈上分配
        }
        long end = System.currentTimeMillis();
        System.out.println("Time: " + (end - start) + "ms");
    }

    private static void alloc() {
        // Point 不逃逸出此方法（没有返回值被使用）
        Point p = new Point(1, 2);
        int sum = p.x + p.y;  // 只在内部使用
    }
}
// 加 -XX:-DoEscapeAnalysis 关闭后，GC 会频繁触发，耗时明显增加
```

---

### 8.5.4 标量替换（Scalar Replacement）

比栈上分配更进一步：JIT 直接把对象**拆解为多个基本类型字段**，放入寄存器或栈，对象本身不再实际存在。

```java
// 原始代码
Point p = new Point(3, 4);
int sum = p.x + p.y;

// 标量替换后（等效）
int p_x = 3;  // 标量（基本类型）
int p_y = 4;  // 标量（基本类型）
int sum = p_x + p_y;
// Point 对象完全消失，无堆分配，无 GC
```

JVM 参数：
```bash
-XX:+EliminateAllocations   # 开启标量替换（默认开启）
```

---

### 8.5.5 锁消除（Lock Elimination）

JIT 检测到某个锁对象**不可能被多个线程争用**（不逃逸），直接消除锁操作。

```java
// StringBuffer 是线程安全的，内部有 synchronized
public String buildString() {
    StringBuffer sb = new StringBuffer();  // sb 不逃逸出此方法
    sb.append("Hello");
    sb.append(" World");
    return sb.toString();
    // JIT 发现 sb 只在本方法用，没有线程竞争
    // 所有 synchronized 锁操作都被消除！
}
// 等效于用了 StringBuilder（无锁）
```

---

### 8.5.6 锁粗化（Lock Coarsening）

将多次对**同一个锁对象**的连续加锁/解锁合并为一次加锁范围更大的操作，减少加解锁次数。

```java
// 原始代码（频繁加解锁）
for (int i = 0; i < 1000; i++) {
    synchronized(lock) {   // ← 每次循环都要加锁
        list.add(i);
    }                      // ← 每次循环都要解锁
}

// JIT 锁粗化后（等效）
synchronized(lock) {       // ← 一次加锁
    for (int i = 0; i < 1000; i++) {
        list.add(i);
    }
}                          // ← 一次解锁
// 减少了 999 次加解锁操作
```

---

## 8.6 AOT 编译（Ahead-Of-Time）

Java 9 引入 `jaotc` 工具（实验性），Java 17 后由 **GraalVM Native Image** 主导。

```
JIT（Just-In-Time）：运行时编译，边跑边优化
AOT（Ahead-Of-Time）：运行前静态编译为原生二进制

AOT 优点：
  - 启动速度极快（无预热期）
  - 内存占用更小
  - 适合 Serverless / 云原生场景

AOT 缺点：
  - 编译时间长
  - 无法使用部分反射/动态代理特性（需要配置）
  - 运行时优化机会少于 JIT
```

GraalVM Native Image 示例：
```bash
# 将 Spring Boot 应用编译为原生二进制
mvn -Pnative native:compile

# 生成的 target/myapp 是原生可执行文件，启动时间 < 100ms
./target/myapp
```

---

# Part 9 - 完整实战案例

---

## 9.1 案例一：线上服务频繁 Full GC 排查

### 问题描述

某电商服务突然出现接口响应变慢，监控告警显示 Full GC 频率从每天 1 次升高到每分钟 3 次，每次 Full GC 耗时 3-8 秒（STW），导致大量请求超时。

### 排查步骤

**第一步：确认问题 —— 用 jstat 观察 GC 状态**

```bash
# 每秒输出一次 GC 统计，观察 30 次
jstat -gcutil <pid> 1000 30

# 输出示例（发现问题）：
#  S0     S1     E      O      M     CCS    YGC     YGCT    FGC    FGCT     GCT
#   0.00  85.12  93.45  98.76  85.23  78.12    523   12.345     47   98.234  110.579
#   0.00  85.12  99.99  98.76  85.23  78.12    524   12.567     47   98.234  110.801
#   0.00   0.00   3.45  99.12  85.23  78.12    525   12.789     48  101.456  114.245
#
# 发现：
#   O（老年代）使用率持续 98%+ ← 核心问题
#   FGC 每秒触发 1 次 ← 严重
#   每次 FGCT 增长约 3 秒 ← STW 严重
```

**第二步：定位大对象 —— 用 jmap 查看堆对象分布**

```bash
# 查看存活对象统计（会触发一次 Full GC，谨慎）
jmap -histo:live <pid> | head -30

# 输出示例：
#  num     #instances         #bytes  class name
# --------------------------------------------------
#    1:      2847293      682340432  [B   ← byte数组！682MB
#    2:       845729       27063328  java.lang.String
#    3:       234591       18767280  java.util.HashMap$Node
#    4:        89234       14277440  com.example.SessionData  ← 可疑
#
# 发现：[B（byte数组）占 682MB，SessionData 对象数量异常多
```

**第三步：生成 heap dump 进行深度分析**

```bash
# 生成 dump 文件（注意：会有短暂 STW）
jmap -dump:live,format=b,file=/tmp/heap_$(date +%Y%m%d_%H%M%S).hprof <pid>

# 下载到本机分析
scp user@server:/tmp/heap_20240101_120000.hprof ./
```

**第四步：用 MAT 分析 dump 文件**

```
打开 MAT → File → Open Heap Dump → 选择 .hprof 文件

分析步骤：
1. 查看 Leak Suspects Report：
   → 发现 "SessionManager 持有大量 SessionData 对象"
   → 单个 SessionData 平均 7KB，共 89234 个 = ~600MB

2. 查看 Dominator Tree：
   → com.example.SessionManager.sessionCache (HashMap)
   → 占用总堆内存的 82%

3. 查看 GC Roots 引用链：
   → SessionManager 是单例（静态字段）
   → sessionCache 没有过期机制，只有 put 没有 remove
```

**第五步：定位代码问题**

```java
// 问题代码（SessionManager.java）
@Component
public class SessionManager {
    // ← 这里！静态 HashMap，无过期，无上限
    private static final Map<String, SessionData> sessionCache = new HashMap<>();

    public void createSession(String token, SessionData data) {
        sessionCache.put(token, data);  // 只有 put
        // 没有过期清理逻辑！
    }
    // getSession 方法存在，但没有 removeSession 在 logout 时调用
}
```

**第六步：修复方案**

```java
// 修复后：使用 Caffeine 带过期时间的缓存
@Component
public class SessionManager {
    private final Cache<String, SessionData> sessionCache = Caffeine.newBuilder()
        .maximumSize(50000)                    // 最多 50000 个会话
        .expireAfterAccess(30, TimeUnit.MINUTES)  // 30 分钟不访问则过期
        .recordStats()                         // 统计命中率
        .build();

    public void createSession(String token, SessionData data) {
        sessionCache.put(token, data);
    }

    public SessionData getSession(String token) {
        return sessionCache.getIfPresent(token);
    }

    public void removeSession(String token) {
        sessionCache.invalidate(token);
    }
}
```

**修复结果**：
- Full GC 恢复到每天 0-1 次
- 老年代使用率从 98% 降到 40%
- 接口 P99 从 5s 降到 50ms

---

## 9.2 案例二：内存泄漏排查

### 常见内存泄漏场景

#### 场景一：静态集合无限增长（最常见）

```java
// 问题代码
public class BlackListService {
    // 静态 Set，只加不删
    private static final Set<String> blackList = new HashSet<>();

    public static void addToBlackList(String ip) {
        blackList.add(ip);  // 每次拦截都加入，从不清理
    }
    // 随着时间推移，blackList 无限增长 → 内存泄漏
}

// 修复：定期清理 + 使用有界集合
private static final int MAX_SIZE = 100000;
public static void addToBlackList(String ip) {
    if (blackList.size() >= MAX_SIZE) {
        // 超过上限，清理最早加入的
        blackList.clear();  // 简单粗暴，或用 LinkedHashMap 实现 LRU
    }
    blackList.add(ip);
}
```

#### 场景二：ThreadLocal 未 remove（生产中高发）

```java
// 问题代码（线程池场景下严重）
public class UserContextFilter implements Filter {
    private static final ThreadLocal<UserContext> USER_CONTEXT = new ThreadLocal<>();

    @Override
    public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
            throws IOException, ServletException {
        USER_CONTEXT.set(new UserContext(getCurrentUser()));
        // 处理请求...
        chain.doFilter(req, resp);
        // ← 忘记 remove！线程池中线程会被复用
        // 每次请求都 set 一个新对象，旧对象无法被 GC
    }
}

// 修复：try-finally 确保 remove
@Override
public void doFilter(ServletRequest req, ServletResponse resp, FilterChain chain)
        throws IOException, ServletException {
    try {
        USER_CONTEXT.set(new UserContext(getCurrentUser()));
        chain.doFilter(req, resp);
    } finally {
        USER_CONTEXT.remove();  // ← 必须在 finally 中 remove
    }
}
```

#### 场景三：监听器/回调未注销

```java
// 问题代码
public class DataChangeListener {
    public DataChangeListener(EventBus bus) {
        bus.register(this);  // 注册监听器
        // 从未 unregister！即使这个对象"不用了"
        // EventBus 持有引用 → 对象无法被 GC
    }
}

// 修复：实现生命周期方法
@PreDestroy
public void destroy() {
    bus.unregister(this);  // 销毁时注销
}
```

#### 场景四：连接/流未关闭

```java
// 问题代码
public void processFile(String path) throws IOException {
    InputStream is = new FileInputStream(path);  // 未用 try-with-resources
    // 如果中途抛异常，is 永远不会被关闭
    // 文件句柄/连接泄漏
}

// 修复：try-with-resources（Java 7+）
public void processFile(String path) throws IOException {
    try (InputStream is = new FileInputStream(path)) {
        // 无论是否异常，is 都会自动关闭
        processData(is);
    }
}
```

---

## 9.3 案例三：JVM 参数配置参考方案

根据服务器内存规格，提供三套典型配置：

### 方案一：小内存服务（512MB，如测试环境/微服务）

```bash
# 适用：内存有限的测试环境或轻量微服务
java \
  -Xms256m \                          # 初始堆大小
  -Xmx512m \                          # 最大堆大小（等于物理内存的50-70%）
  -Xss256k \                          # 每线程栈大小（减小以支持更多线程）
  -XX:MetaspaceSize=64m \             # 元空间初始大小
  -XX:MaxMetaspaceSize=128m \         # 元空间上限
  -XX:+UseG1GC \                      # 使用 G1 收集器
  -XX:MaxGCPauseMillis=200 \          # 期望最大 GC 暂停 200ms
  -XX:+HeapDumpOnOutOfMemoryError \   # OOM 时自动 dump
  -XX:HeapDumpPath=/logs/heap.hprof \ # dump 路径
  -jar app.jar
```

### 方案二：中等内存服务（4GB，如普通 Java 后端服务）

```bash
# 适用：常见业务服务，中等流量
java \
  -Xms2g \                            # 初始堆 = 最大堆（避免扩容开销）
  -Xmx2g \
  -Xss512k \
  -XX:MetaspaceSize=256m \
  -XX:MaxMetaspaceSize=512m \
  -XX:+UseG1GC \
  -XX:MaxGCPauseMillis=100 \          # 目标暂停时间 100ms
  -XX:G1HeapRegionSize=4m \           # Region 大小（堆大小/2048，4GB堆用8m）
  -XX:InitiatingHeapOccupancyPercent=45 \  # 老年代占比达45%触发并发标记
  -XX:G1ReservePercent=15 \           # 预留 15% 堆防止疏散失败
  -XX:+PrintGCDetails \               # 打印 GC 详情（JDK8）
  -XX:+PrintGCDateStamps \            # GC 时间戳
  -Xloggc:/logs/gc.log \              # GC 日志路径
  -XX:+UseGCLogFileRotation \         # GC 日志轮转
  -XX:NumberOfGCLogFiles=5 \
  -XX:GCLogFileSize=50m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/logs/heap.hprof \
  -jar app.jar
```

### 方案三：大内存服务（16GB，如大数据处理/高并发网关）

```bash
# 适用：大内存高并发场景，优先低延迟
java \
  -Xms12g \                           # 留 4GB 给 OS 和非堆内存
  -Xmx12g \
  -Xss256k \                          # 大量线程时减小栈
  -XX:MetaspaceSize=512m \
  -XX:MaxMetaspaceSize=1g \
  -XX:+UseZGC \                       # ZGC：适合超低延迟需求（JDK15+ 生产可用）
  # 或使用 G1:
  # -XX:+UseG1GC
  # -XX:MaxGCPauseMillis=50
  # -XX:G1HeapRegionSize=16m
  -XX:+AlwaysPreTouch \               # JVM启动时立即分配所有内存页（避免运行时缺页）
  -XX:+DisableExplicitGC \            # 禁止代码中 System.gc() 调用
  -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags:filecount=5,filesize=50m \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/logs/ \
  -XX:ErrorFile=/logs/hs_err_pid%p.log \  # JVM 崩溃日志
  -jar app.jar
```

---

## 9.4 案例四：CPU 100% 排查

### 问题描述

线上某服务 CPU 使用率突然飙升到 100%，响应极慢甚至不响应。

### 排查步骤

**第一步：找到 CPU 最高的 Java 进程**

```bash
top
# 按 P 按 CPU 使用率排序
# 找到 CPU 最高的 Java 进程 PID，如 PID = 12345
```

**第二步：找到该进程中 CPU 最高的线程**

```bash
# Linux 下查看进程内各线程的 CPU 使用情况
top -Hp 12345
# 或
ps -mp 12345 -o THREAD,tid,time

# 找到 CPU 最高的线程 TID（十进制），如 TID = 12400
```

**第三步：将线程 TID 转为十六进制**

```bash
printf "%x\n" 12400
# 输出：3070   ← 这就是 jstack 中的 nid
```

**第四步：用 jstack 获取线程堆栈**

```bash
jstack 12345 > /tmp/thread_dump.txt

# 在 thread_dump.txt 中搜索 nid=0x3070
grep -A 30 "nid=0x3070" /tmp/thread_dump.txt
```

**第五步：分析堆栈，找到根因**

```
# 输出示例 1：死循环
"pool-1-thread-1" #25 prio=5 os_prio=0 tid=0x... nid=0x3070 runnable
   java.lang.Thread.State: RUNNABLE
    at com.example.DataProcessor.process(DataProcessor.java:87)  ← 死循环位置
    at com.example.TaskRunner.run(TaskRunner.java:45)
    ...

# 去代码 DataProcessor.java:87 检查，发现：
while (iterator.hasNext()) {
    Object obj = iterator.next();
    if (shouldSkip(obj)) {
        continue;       // ← 在某个条件下，iterator 状态未更新
        // 导致 hasNext() 永远返回 true → 死循环
    }
    process(obj);
}

# 输出示例 2：频繁 Full GC
"GC task thread#0 (ParallelGC)" os_prio=0 tid=0x... nid=0x1234 runnable
"GC task thread#1 (ParallelGC)" os_prio=0 tid=0x... nid=0x1235 runnable
# 大量 GC 线程都在 runnable → 说明 GC 在疯狂运行
# 这时需要回到 9.1 的 Full GC 排查流程
```

**Arthas 一键排查 CPU 热点**（更简便）：

```bash
java -jar arthas-boot.jar
arthas> thread -n 5          # 查看 CPU 占用最高的 5 个线程
# 直接显示每个线程的堆栈，无需手动转换 TID

arthas> thread -b            # 查找阻塞其他线程的线程（找死锁）
```

---

# Part 10 - 常见面试题 & FAQ

---

## 10.1 面试题精选（含详细解答）

---

### 面试题 1：JVM 内存区域有哪些？各自的作用是什么？

**标准回答思路**：

JVM 内存区域分为**线程私有**和**线程共享**两类：

**线程私有（每个线程独立持有）**：
1. **程序计数器（PC Register）**：记录当前线程执行的字节码指令地址。唯一不会 OOM 的区域。
2. **虚拟机栈（VM Stack）**：每个方法调用对应一个栈帧，存储局部变量表、操作数栈、动态链接、方法出口信息。栈过深抛 `StackOverflowError`，内存不足抛 `OutOfMemoryError`。
3. **本地方法栈（Native Method Stack）**：为 Native 方法服务，功能与虚拟机栈类似。

**线程共享（所有线程共享）**：
4. **堆（Heap）**：最大的内存区域，几乎所有对象实例都在这里分配。分为新生代（Eden + Survivor×2）和老年代。OOM 最常见的地方。
5. **方法区（Method Area）**：存储类的结构信息（类名、字段、方法字节码）、运行时常量池、静态变量。JDK8 之前叫永久代（PermGen），JDK8 起改为元空间（Metaspace，使用直接内存）。

**直接内存**（不属于 JVM 规范但实际重要）：通过 NIO 的 `ByteBuffer.allocateDirect()` 分配，不受 `-Xmx` 限制，受 `-XX:MaxDirectMemorySize` 控制。

---

### 面试题 2：什么是双亲委派模型？为什么需要它？

**双亲委派模型**：

当一个类加载器收到类加载请求时，**不会自己先去加载**，而是把请求委派给父类加载器。只有当父类加载器无法完成加载（在其搜索范围内找不到该类），子类加载器才会尝试自己加载。

```
Bootstrap ClassLoader（最顶层，加载 rt.jar，C++ 实现）
       ↑ 委派
Extension ClassLoader（加载 ext/*.jar）
       ↑ 委派
App ClassLoader（加载 classpath）
       ↑ 委派
自定义 ClassLoader
```

**为什么需要**：保证**核心类库的唯一性与安全性**。
- 如果没有双亲委派，任何人都可以写一个 `java.lang.String` 类并加载，会造成混乱甚至安全漏洞。
- 有了双亲委派，`java.lang.String` 始终只由 Bootstrap ClassLoader 加载，保证了全局唯一。

**打破双亲委派的场景**：
1. **JNDI/SPI 机制**（如 JDBC）：Bootstrap 加载了 JDBC 接口，但实现类在 classpath（由 App ClassLoader 加载），需要**线程上下文类加载器**反向委派。
2. **OSGi / Tomcat 类隔离**：每个 WebApp 有自己的类加载器，优先加载自己的类，实现不同应用间的类隔离。
3. **热替换/热部署**：需要重新加载已加载的类，必须使用新的类加载器实例。

---

### 面试题 3：Minor GC、Major GC、Full GC 有什么区别？

| 类型 | 回收区域 | 触发条件 | STW |
|-----|---------|---------|-----|
| Minor GC | 新生代（Eden + Survivor）| Eden 区满 | 短暂 |
| Major GC | 老年代 | 老年代空间不足（通常伴随 Minor GC）| 较长 |
| Full GC | 整个堆 + 方法区 | 多种原因（见下）| 最长 |

**Full GC 触发条件**（高频考点）：
1. 调用 `System.gc()`（建议触发，不保证立即执行）
2. 老年代空间不足（大对象直接进老年代后放不下）
3. 空间分配担保失败（Minor GC 前，老年代剩余空间 < 新生代历次晋升平均大小）
4. 方法区/元空间不足
5. 使用 CMS 时并发模式失败（Concurrent Mode Failure）

**如何减少 Full GC**：
- 合理设置堆大小，避免老年代过小
- 避免大量大对象（超过 `-XX:PretenureSizeThreshold`）
- 排查内存泄漏，避免老年代持续增长
- 使用 G1/ZGC 替代 CMS，减少 Full GC 发生

---

### 面试题 4：G1 和 CMS 有什么区别？如何选择？

**CMS（Concurrent Mark Sweep）**：
- 以最短 GC 停顿时间为目标
- 老年代使用标记-清除算法，会产生内存碎片
- 有并发模式失败（CMF）风险，退化为 Serial Old（极长 STW）
- JDK 14 已废弃

**G1（Garbage First）**：
- 将堆划分为等大小的 Region（默认 ~2MB）
- 可以对任意 Region 进行回收，优先回收垃圾最多的 Region
- 使用标记-整理算法，无内存碎片
- 可设置停顿目标（`-XX:MaxGCPauseMillis`）
- JDK 9+ 默认收集器

**选择建议**：
```
堆 < 4GB，对停顿不敏感  →  Parallel GC（吞吐量优先）
堆 4-8GB，需要较低停顿  →  G1 GC
堆 > 8GB，需要极低停顿  →  G1 GC 或 ZGC（JDK15+）
延迟要求 < 10ms          →  ZGC 或 Shenandoah
```

---

### 面试题 5：什么是 OOM？常见的 OOM 有哪些？如何排查？

**OOM（OutOfMemoryError）**：JVM 无法再分配所需内存时抛出。

**常见类型**：

1. **`Java heap space`**（最常见）
   - 堆内存不够用
   - 排查：`jmap -histo:live <pid>` 看哪些对象最多；MAT 分析 dump 找内存泄漏

2. **`GC overhead limit exceeded`**
   - GC 花费时间超过 98%，但回收内存不足 2%（GC 卡死）
   - 说明内存严重不足，通常是内存泄漏的前兆

3. **`Metaspace`**（JDK8+）
   - 元空间不足，通常是类加载过多（如动态代理、反射生成大量类）
   - 排查：`jstat -gcmetacapacity <pid>`；检查是否有类加载框架无限生成代理类

4. **`Direct buffer memory`**
   - NIO 直接内存不足
   - 排查：检查 `ByteBuffer.allocateDirect()` 的使用，检查 `-XX:MaxDirectMemorySize` 设置

5. **`Unable to create new native thread`**
   - 无法创建新线程（操作系统级别限制）
   - 排查：`ps -efL | grep java | wc -l` 查线程数；检查线程池是否泄漏；调整 `/proc/sys/kernel/threads-max`

---

### 面试题 6：什么是 STW？如何减少 STW 的影响？

**STW（Stop The World）**：JVM 在执行 GC 时，暂停所有用户线程的现象。GC 线程独占运行，用户线程完全静止。

**为什么需要 STW**：
- GC 在遍历对象引用链时，如果用户线程同时在修改引用关系，会导致标记结果不一致（"漏标"问题）
- 部分 GC 阶段必须保证引用关系不变化

**不同 GC 的 STW 对比**：
```
Serial GC：Major GC 完全 STW，可能数秒
CMS：初始标记+重新标记 STW，其余并发
G1：初始标记+最终标记 STW，混合回收可控
ZGC：只在初始标记/最终标记 STW（通常 < 1ms）
```

**减少 STW 影响的手段**：
1. 选择低延迟 GC（G1/ZGC）
2. 设置合理的 `-XX:MaxGCPauseMillis` 目标
3. 增大堆内存，减少 GC 频率
4. 降低对象分配速率（减少短命大对象）
5. 错峰 Full GC（在流量低峰触发）

---

### 面试题 7：对象一定分配在堆上吗？

**不一定**，有以下例外：

1. **栈上分配**：经过逃逸分析，若对象不会逃逸出当前方法，JIT 可以将其分配在虚拟机栈上。方法返回时自动释放，无 GC 压力。

2. **标量替换**：逃逸分析后，JIT 直接把对象拆解为基本类型标量，放入寄存器或栈，对象本身不再存在（连栈上分配都省了）。

3. **TLAB（Thread Local Allocation Buffer）**：虽然 TLAB 仍在堆内，但每个线程在 Eden 区有专属的小块内存，分配时不需要加锁，是线程局部的快速分配。

**完整对象分配流程**：
```
new 对象
  → 是否触发逃逸分析 & 标量替换？是 → 寄存器/栈（无堆分配）
  → 是否超过 TLAB 大小？否 → 在 TLAB 中快速分配（Eden 区内）
  → TLAB 装不下 → 在 Eden 区加锁分配
  → 对象太大（> PretenureSizeThreshold）→ 直接在老年代分配
```

---

### 面试题 8：什么是类加载的双亲委派，如何自定义类加载器打破它？

**打破双亲委派的核心**：重写 `ClassLoader.loadClass()` 方法（而非 `findClass()`）。

```java
public class HotSwapClassLoader extends ClassLoader {
    private final String classPath;

    public HotSwapClassLoader(String classPath) {
        // 注意：父加载器设为 null（Bootstrap），或者指定特定的父加载器
        super(null);
        this.classPath = classPath;
    }

    @Override
    public Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        // 打破双亲委派：先自己尝试加载，找不到再委派给父类
        // （正常双亲委派是先委派父类）
        synchronized (getClassLoadingLock(name)) {
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                try {
                    c = findClass(name);  // ← 先自己找
                } catch (ClassNotFoundException e) {
                    // 自己找不到，再委派父类
                    c = super.loadClass(name, false);
                }
            }
            if (resolve) resolveClass(c);
            return c;
        }
    }

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        // 从自定义路径加载 .class 文件字节码
        String filePath = classPath + name.replace('.', '/') + ".class";
        try {
            byte[] bytes = Files.readAllBytes(Paths.get(filePath));
            return defineClass(name, bytes, 0, bytes.length);
        } catch (IOException e) {
            throw new ClassNotFoundException(name, e);
        }
    }
}
```

**使用场景**：
- Tomcat 的 WebAppClassLoader 先加载 `/WEB-INF/classes` 和 `/WEB-INF/lib` 中的类，实现 WebApp 隔离
- 热部署：每次热替换时创建新的类加载器实例，重新加载修改过的类

---

### 面试题 9：强引用、软引用、弱引用、虚引用有什么区别？

```
引用类型     GC 行为           典型使用场景
─────────────────────────────────────────────────────────────
强引用       不回收             Object obj = new Object()（默认）
软引用       内存不足时回收      缓存（图片缓存、对象池）
弱引用       下次 GC 必定回收    WeakHashMap，ThreadLocal 的 key
虚引用       随时可回收         追踪对象被回收的时机，管理堆外内存
```

**软引用缓存示例**：
```java
// 内存充足时保留缓存，内存不足时让 GC 自动清理
Map<String, SoftReference<byte[]>> imageCache = new HashMap<>();

// 存入缓存
imageCache.put("logo.png", new SoftReference<>(loadImage("logo.png")));

// 读取缓存（注意软引用可能已被回收）
SoftReference<byte[]> ref = imageCache.get("logo.png");
byte[] image = (ref != null) ? ref.get() : null;
if (image == null) {
    // 缓存已被 GC 回收，重新加载
    image = loadImage("logo.png");
    imageCache.put("logo.png", new SoftReference<>(image));
}
```

**弱引用的 ThreadLocal 内存泄漏**：
```
ThreadLocalMap 的 Entry 继承 WeakReference<ThreadLocal<?>>
└── key（ThreadLocal 对象）= 弱引用，GC 后变为 null
└── value（业务对象）= 强引用，key 为 null 后 value 无法被 GC

解决：每次使用完 ThreadLocal 后调用 threadLocal.remove()
```

---

### 面试题 10：JVM 调优的一般思路是什么？

**调优方法论（三步法）**：

**第一步：明确目标**
- 高吞吐量（如批处理）：用 Parallel GC，接受长 GC 停顿
- 低延迟（如在线服务）：用 G1/ZGC，控制 `-XX:MaxGCPauseMillis`
- 快速启动（如 Serverless）：考虑 GraalVM AOT

**第二步：监控和定位**
```bash
# 1. 观察 GC 频率和停顿时间
jstat -gcutil <pid> 1000 60

# 2. 分析 GC 日志（开启后）
# 重点关注：Full GC 频率、单次 GC 耗时、老年代趋势

# 3. 内存使用分析
jmap -histo:live <pid>   # 看对象分布
```

**第三步：调整参数并验证**
```
堆太小 → 增加 -Xmx，调整 NewRatio
Full GC 太多 → 检查内存泄漏，或增大老年代
Young GC 太频繁 → 增大 -Xmn（新生代大小）
GC 停顿太长 → 换 G1，设置 MaxGCPauseMillis
Metaspace OOM → 增大 MaxMetaspaceSize
线程 OOM → 减少 -Xss，或减少线程数
```

**调优黄金法则**：
1. **先测量，再调优**：不要凭感觉调参数
2. **一次只改一个参数**：否则无法判断哪个参数起了作用
3. **在生产类似环境中测试**：单机测试结果不能代表生产
4. **关注业务指标**：最终目标是接口响应时间和吞吐量，而不是 GC 统计数据看起来好看

---

## 10.2 总结：JVM 核心知识点速查表

```
┌─────────────────────────────────────────────────────────────────────┐
│                     JVM 核心知识点速查表                              │
├───────────────────┬─────────────────────────────────────────────────┤
│ 内存区域           │ 程序计数器/虚拟机栈/本地方法栈（线程私有）           │
│                   │ 堆/方法区（线程共享）                              │
├───────────────────┼─────────────────────────────────────────────────┤
│ 类加载             │ 加载→验证→准备→解析→初始化→使用→卸载               │
│                   │ Bootstrap > Extension > App（双亲委派）            │
├───────────────────┼─────────────────────────────────────────────────┤
│ 对象布局           │ 对象头（Mark Word + 类型指针）+ 实例数据 + 对齐填充  │
├───────────────────┼─────────────────────────────────────────────────┤
│ GC 算法            │ 标记-清除（碎片）/ 标记-复制（浪费50%空间）/         │
│                   │ 标记-整理（无碎片、慢）                             │
├───────────────────┼─────────────────────────────────────────────────┤
│ GC 收集器          │ Serial < Parallel < CMS < G1 < ZGC               │
│                   │ JDK8默认Parallel，JDK9+默认G1                     │
├───────────────────┼─────────────────────────────────────────────────┤
│ 常用 JVM 参数       │ -Xms/-Xmx（堆大小）                              │
│                   │ -Xss（栈大小）                                     │
│                   │ -XX:MetaspaceSize（元空间）                        │
│                   │ -XX:+UseG1GC / UseZGC（收集器选择）                │
│                   │ -XX:MaxGCPauseMillis（G1目标停顿）                 │
│                   │ -XX:+HeapDumpOnOutOfMemoryError（OOM自动dump）     │
├───────────────────┼─────────────────────────────────────────────────┤
│ 诊断工具           │ jps/jstat/jmap/jstack/jinfo/jcmd（命令行）         │
│                   │ VisualVM（图形化）/ Arthas（在线诊断）/ MAT（dump分析）│
├───────────────────┼─────────────────────────────────────────────────┤
│ JIT 优化           │ 方法内联 > 逃逸分析 > 栈上分配 > 标量替换           │
│                   │ 锁消除 / 锁粗化                                    │
├───────────────────┼─────────────────────────────────────────────────┤
│ 高频 OOM          │ Java heap space（堆满）                            │
│                   │ Metaspace（类太多）                                │
│                   │ Direct buffer memory（NIO直接内存）                │
│                   │ Unable to create new native thread（线程太多）     │
└───────────────────┴─────────────────────────────────────────────────┘
```

---

> **学习建议**：
> 1. 先理解概念（Why），再记参数（What），最后动手实验（How）
> 2. 搭建本地测试环境，用 `-verbose:gc` 观察 GC 日志
> 3. 使用 VisualVM 连接本地 Java 程序，实时观察内存变化
> 4. 遇到真实 OOM 或 GC 问题时，按本文案例步骤逐步排查
> 5. 持续关注 JVM 新版本特性（ZGC、GraalVM、Virtual Threads 等）

---

*文档完结 - JVM 深度详解（从入门到精通）*


---

