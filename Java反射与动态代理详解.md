# Java 反射与动态代理详解 - 从零到精通

> 版本：Java 8 / Java 11 / Java 17 全覆盖
> 难度：★★★★☆
> 适合：中高级 Java 开发工程师、架构师、框架源码学习者

---

```
+==============================================================================+
|           Java 反射与动态代理 - 知识体系全景图                               |
|==============================================================================|
|                                                                              |
|  +-------------+    +-------------+    +---------------------------+        |
|  |  Class 对象  |    |  反射 API   |    |       动态代理            |        |
|  |             |    |             |    |                           |        |
|  | .class 文件 |--->| Constructor |    |  +---------------------+  |        |
|  | 类加载器    |    | Field       |    |  |  JDK 动态代理        |  |        |
|  | JVM 元数据  |    | Method      |    |  |  （基于接口）        |  |        |
|  +-------------+    | Annotation  |    |  +---------------------+  |        |
|                     +-------------+    |  +---------------------+  |        |
|  +-------------+    +-------------+    |  |  CGLIB 代理          |  |        |
|  | 泛型与反射   |    |  性能优化   |    |  |  （基于子类继承）    |  |        |
|  |             |    |             |    |  +---------------------+  |        |
|  | 类型擦除    |    | MethodHandle|    +---------------------------+        |
|  | TypeToken   |    | LambdaMeta  |                                         |
|  | 泛型解析    |    | 缓存策略    |    +---------------------------+        |
|  +-------------+    +-------------+    |       实战应用            |        |
|                                        |  Spring IoC / AOP        |        |
|  +-------------+    +-------------+    |  MyBatis Mapper 代理     |        |
|  | 字节码操作  |    | 注解处理器  |    |  序列化框架              |        |
|  |             |    |             |    |  简易 IoC 容器           |        |
|  | Javassist   |    | APT 编译时  |    +---------------------------+        |
|  | ASM         |    | JavaPoet    |                                         |
|  +-------------+    +-------------+                                         |
+==============================================================================+
```

---

## 目录

- [Part 1: Class 对象与类加载](#part-1-class-对象与类加载)
- [Part 2: 反射 API 完整详解](#part-2-反射-api-完整详解)
- [Part 3: 反射性能优化](#part-3-反射性能优化)
- [Part 4: 泛型与反射](#part-4-泛型与反射)
- [Part 5: JDK 动态代理深度解析](#part-5-jdk-动态代理深度解析)
- [Part 6: CGLIB 动态代理深度解析](#part-6-cglib-动态代理深度解析)
- [Part 7: JDK 代理 vs CGLIB 代理](#part-7-jdk-代理-vs-cglib-代理)
- [Part 8: Javassist 字节码操作](#part-8-javassist-字节码操作)
- [Part 9: MethodHandle 与 VarHandle](#part-9-methodhandle-与-varhandle)
- [Part 10: 注解处理器（APT）](#part-10-注解处理器apt)
- [Part 11: 反射与框架底层应用](#part-11-反射与框架底层应用)
- [Part 12: 完整实战案例](#part-12-完整实战案例)
- [Part 13: 常见面试题 FAQ](#part-13-常见面试题-faq)

---

# Part 1: Class 对象与类加载

## 1.1 Class 对象是什么

在 JVM 中，每当一个 `.class` 文件被类加载器加载到内存后，JVM 就会在堆区创建一个对应的
`java.lang.Class` 对象。这个 Class 对象是该类在运行时的**元数据描述**，是反射的入口。

```
+----------------------------------------------------------------------+
|                         JVM 内存结构                                  |
+----------------------------------------------------------------------+
|  +--------------------------+   +-----------------------------+      |
|  |   方法区 / 元空间         |   |          堆区               |      |
|  |  (Metaspace)             |   |         (Heap)              |      |
|  |                          |   |                             |      |
|  |  +-------------------+   |   |  +---------------------+   |      |
|  |  | 类的元数据         |   |   |  |  Class 对象          |   |      |
|  |  | - 类名             |<--+---+--+  java.lang.Class     |   |      |
|  |  | - 父类             |   |   |  |  - 反射 API 入口     |   |      |
|  |  | - 接口列表         |   |   |  +---------------------+   |      |
|  |  | - 字段/方法描述符  |   |   |                             |      |
|  |  | - 常量池           |   |   |  +---------------------+   |      |
|  |  | - 静态变量         |   |   |  |  普通实例对象         |   |      |
|  |  +-------------------+   |   |  |  new Person() ...    |   |      |
|  +--------------------------+   |  +---------------------+   |      |
|                                 +-----------------------------+      |
|  关键：每个类只有一个 Class 对象（同类加载器，同类名唯一）             |
+----------------------------------------------------------------------+
```

**Class 对象的核心作用：**

| 功能 | 说明 |
|------|------|
| 类型信息 | 存储类名、修饰符、父类、接口等 |
| 构造方法 | 通过 `Constructor` 对象创建实例 |
| 字段访问 | 通过 `Field` 对象读写实例字段 |
| 方法调用 | 通过 `Method` 对象动态调用方法 |
| 注解读取 | 获取类/方法/字段上的注解信息 |
| 类型判断 | `instanceof` 的运行时实现基础 |

## 1.2 获取 Class 对象的四种方式

```java
package com.example.reflection;

/**
 * 演示获取 Class 对象的四种方式
 */
public class ObtainClassDemo {

    public static void main(String[] args) throws Exception {

        // ====================================================
        // 方式一：类字面量（Class Literal）
        // 特点：编译时确定；不触发类初始化（不执行静态代码块）
        // 适用：已知具体类型，性能最好
        // ====================================================
        Class<String>  c1 = String.class;
        Class<Integer> c2 = Integer.class;
        Class<int[]>   c3 = int[].class;
        System.out.println("类字面量: " + c1.getName());  // java.lang.String
        System.out.println("int[]:   " + c3.getName());  // [I

        // ====================================================
        // 方式二：对象的 getClass() 方法
        // 特点：运行时确定，返回对象实际类型（多态时很重要）
        // 适用：已有对象实例，需获取实际运行时类型
        // ====================================================
        String str = "Hello";
        Class<?> c4 = str.getClass();
        System.out.println("getClass: " + c4.getName());  // java.lang.String

        // 多态场景：返回实际类型
        Object obj = new java.util.ArrayList<>();
        System.out.println("多态: " + obj.getClass().getName());  // java.util.ArrayList

        // ====================================================
        // 方式三：Class.forName()
        // 特点：通过全限定名加载；【会触发类初始化】（执行静态块）
        // 适用：类名运行时才知道（JDBC 驱动加载）
        // ====================================================
        Class<?> c6 = Class.forName("java.util.HashMap");
        System.out.println("forName: " + c6.getName());  // java.util.HashMap

        // ====================================================
        // 方式四：ClassLoader.loadClass()
        // 特点：可指定类加载器；【不触发类初始化】
        // 适用：OSGi、热部署、隔离类加载场景
        // ====================================================
        ClassLoader cl  = Thread.currentThread().getContextClassLoader();
        Class<?> c8 = cl.loadClass("java.util.LinkedList");
        System.out.println("ClassLoader: " + c8.getName());  // java.util.LinkedList

        // ====================================================
        // 验证：同一类加载器加载同一类，Class 对象唯一
        // ====================================================
        Class<?> a = String.class;
        Class<?> b = "test".getClass();
        Class<?> cc = Class.forName("java.lang.String");
        System.out.println("a == b: " + (a == b));    // true
        System.out.println("b == cc: " + (b == cc));  // true
        System.out.println("c2=" + c2.getSimpleName()); // 消除未使用警告
    }
}
```

**四种方式对比：**

```
+--------------------+------------+-----------+-------------------------+
|      获取方式       | 触发初始化 | 编译检查  |         场景            |
+--------------------+------------+-----------+-------------------------+
| ClassName.class    |    否      |    是     | 类型已知，性能最优       |
| obj.getClass()     | 已初始化   |    是     | 获取对象实际运行时类型   |
| Class.forName()    |    是      |    否     | 运行时动态加载（JDBC）   |
| ClassLoader.load() |    否      |    否     | 自定义类加载器场景       |
+--------------------+------------+-----------+-------------------------+
```

## 1.3 类加载时机（主动引用 vs 被动引用）

类的生命周期：**加载 → 验证 → 准备 → 解析 → 初始化 → 使用 → 卸载**

初始化（执行 `<clinit>`）仅在 6 种**主动引用**时触发：

```
+------------------------------------------------------------+
|              6 种主动引用（触发类初始化）                    |
+------------------------------------------------------------+
|  1. new MyClass()              实例化对象                  |
|  2. MyClass.STATIC_FIELD       访问非 final 静态字段        |
|  3. MyClass.staticMethod()     调用静态方法                 |
|  4. Class.forName("MyClass")   forName 默认触发初始化       |
|  5. new ChildClass()           先触发父类初始化              |
|  6. main 所在类                JVM 启动时自动初始化          |
+------------------------------------------------------------+
|  5 种被动引用（不触发初始化）                               |
+------------------------------------------------------------+
|  1. MyClass.class              类字面量                     |
|  2. SubClass.parentField       通过子类引用父类静态字段      |
|  3. MyClass[] arr = new MyClass[5]  数组声明               |
|  4. final static String X = "x"    编译时常量              |
|  5. ClassLoader.loadClass()         ClassLoader 直接加载    |
+------------------------------------------------------------+
```

```java
package com.example.reflection;

/**
 * 演示主动引用 vs 被动引用对类初始化的影响
 */
public class ClassLoadingTimingDemo {

    public static void main(String[] args) throws Exception {

        System.out.println("=== 被动引用（不触发初始化）===");

        // 类字面量不触发
        Class<?> c = TargetClass.class;
        System.out.println("类字面量：不触发初始化，c=" + c.getSimpleName());

        // 数组声明不触发
        TargetClass[] arr = new TargetClass[3];
        System.out.println("数组声明：不触发初始化，length=" + arr.length);

        // 编译期常量（String 字面量 / 整型字面量）不触发
        System.out.println("编译期常量: " + TargetClass.COMPILE_CONST);

        System.out.println("\n=== 主动引用（触发初始化）===");

        // 访问运行时常量（非编译期常量）触发
        System.out.println("运行时常量: " + TargetClass.RUNTIME_CONST);
        // 上面一行会打印 [TargetClass] 静态块执行！
    }
}

class TargetClass {
    // 编译期常量（final + 编译时可确定）：被动引用，不触发初始化
    public static final String COMPILE_CONST = "编译期常量，不触发初始化";

    // 运行时常量（虽然 final，但值在运行时才确定）：触发初始化
    public static final String RUNTIME_CONST
        = System.getProperty("user.name", "unknown-user");

    static {
        System.out.println("[TargetClass] *** 静态初始化块执行！类被初始化了 ***");
    }
}
```

## 1.4 Class 对象常用方法速览

```java
package com.example.reflection;

import java.lang.reflect.Modifier;

/**
 * Class 对象常用方法全览
 */
public class ClassMethodsOverview {

    public static void main(String[] args) throws Exception {
        Class<?> clazz = java.util.ArrayList.class;

        System.out.println("=== 类名 ===");
        System.out.println("getName():          " + clazz.getName());         // java.util.ArrayList
        System.out.println("getSimpleName():    " + clazz.getSimpleName());   // ArrayList
        System.out.println("getCanonicalName(): " + clazz.getCanonicalName()); // java.util.ArrayList
        System.out.println("getPackageName():   " + clazz.getPackageName());  // java.util (Java 9+)

        System.out.println("\n=== 继承关系 ===");
        System.out.println("getSuperclass(): " + clazz.getSuperclass().getName());
        System.out.println("getGenericSuperclass(): " + clazz.getGenericSuperclass());
        System.out.println("getInterfaces():");
        for (Class<?> i : clazz.getInterfaces())
            System.out.println("  - " + i.getName());

        System.out.println("\n=== 修饰符 ===");
        int mod = clazz.getModifiers();
        System.out.println("Modifier.toString(): " + Modifier.toString(mod));
        System.out.println("isPublic:    " + Modifier.isPublic(mod));
        System.out.println("isAbstract:  " + Modifier.isAbstract(mod));
        System.out.println("isFinal:     " + Modifier.isFinal(mod));

        System.out.println("\n=== 类型判断 ===");
        Class<?>[] samples = { String.class, int.class, int[].class,
            Runnable.class, Thread.State.class, Override.class, void.class };
        System.out.printf("%-30s %-11s %-9s %-8s %-11s %-12s%n",
            "类名","isInterface","isArray","isEnum","isPrimitive","isAnnotation");
        System.out.println("-".repeat(88));
        for (Class<?> c : samples) {
            System.out.printf("%-30s %-11s %-9s %-8s %-11s %-12s%n",
                c.getName(), c.isInterface(), c.isArray(), c.isEnum(),
                c.isPrimitive(), c.isAnnotation());
        }

        System.out.println("\n=== 实例化与兼容性 ===");
        // 通过 Constructor 创建实例（推荐，比 newInstance() 更明确）
        Object list = clazz.getDeclaredConstructor(int.class).newInstance(10);
        System.out.println("创建 ArrayList(10): " + list);

        // isAssignableFrom：检查类型兼容性
        System.out.println("Object.isAssignableFrom(String): "
            + Object.class.isAssignableFrom(String.class));  // true
        System.out.println("String.isAssignableFrom(Object): "
            + String.class.isAssignableFrom(Object.class));  // false
        System.out.println("Number.isInstance(42): " + Number.class.isInstance(42)); // true

        System.out.println("\n=== 成员统计 ===");
        System.out.println("getDeclaredFields():       " + clazz.getDeclaredFields().length);
        System.out.println("getDeclaredMethods():      " + clazz.getDeclaredMethods().length);
        System.out.println("getDeclaredConstructors(): " + clazz.getDeclaredConstructors().length);
        System.out.println("getFields()（public含继承）: " + clazz.getFields().length);
        System.out.println("getMethods()（public含继承）: " + clazz.getMethods().length);
    }
}
```

## 1.5 数组类型的 Class 对象

```java
package com.example.reflection;

import java.lang.reflect.Array;

/**
 * 数组类型 Class 的特殊规则
 * JVM 用特殊格式表示数组类名（以 [ 开头）
 */
public class ArrayClassDemo {

    public static void main(String[] args) {
        System.out.println("=== 数组 getName() 格式 ===");
        System.out.println("byte[]:    " + byte[].class.getName());    // [B
        System.out.println("char[]:    " + char[].class.getName());    // [C
        System.out.println("double[]:  " + double[].class.getName());  // [D
        System.out.println("float[]:   " + float[].class.getName());   // [F
        System.out.println("int[]:     " + int[].class.getName());     // [I
        System.out.println("long[]:    " + long[].class.getName());    // [J
        System.out.println("short[]:   " + short[].class.getName());   // [S
        System.out.println("boolean[]: " + boolean[].class.getName()); // [Z
        System.out.println("String[]:  " + String[].class.getName());  // [Ljava.lang.String;
        System.out.println("int[][]:   " + int[][].class.getName());   // [[I

        System.out.println("\n=== JVM 类型描述符 ===");
        System.out.println("+--------+---------+");
        System.out.println("| 描述符  |  类型   |");
        System.out.println("+--------+---------+");
        System.out.println("|   B    |  byte   |");
        System.out.println("|   C    |  char   |");
        System.out.println("|   D    |  double |");
        System.out.println("|   F    |  float  |");
        System.out.println("|   I    |  int    |");
        System.out.println("|   J    |  long   |");
        System.out.println("|   S    |  short  |");
        System.out.println("|   Z    |  boolean|");
        System.out.println("| L..;   |  引用   |");
        System.out.println("|   [    |  数组前缀|");
        System.out.println("+--------+---------+");

        System.out.println("\n=== 数组相关 API ===");
        System.out.println("int[].isArray():            " + int[].class.isArray());   // true
        System.out.println("int[].getComponentType():   " + int[].class.getComponentType()); // int
        System.out.println("String[].getComponentType():" + String[].class.getComponentType());
        System.out.println("int[][].getComponentType(): " + int[][].class.getComponentType()); // int[]
        System.out.println("int[].getSimpleName():      " + int[].class.getSimpleName()); // int[]
        System.out.println("int[][].getSimpleName():    " + int[][].class.getSimpleName()); // int[][]

        System.out.println("\n=== Array.newInstance 动态创建 ===");
        int[] arr1 = (int[]) Array.newInstance(int.class, 5);
        arr1[0] = 10; arr1[4] = 40;
        System.out.println("int[5]: [0]=" + arr1[0] + ", [4]=" + arr1[4]);

        String[][] arr2 = (String[][]) Array.newInstance(String.class, 3, 4);
        arr2[0][0] = "hello"; arr2[2][3] = "world";
        System.out.println("String[3][4]: " + arr2[0][0] + ", " + arr2[2][3]);

        // 通过反射读写数组元素
        Object intArr = Array.newInstance(int.class, 3);
        Array.setInt(intArr, 0, 111);
        Array.setInt(intArr, 2, 333);
        System.out.println("反射读 intArr[2]: " + Array.getInt(intArr, 2));
        System.out.println("Array.getLength: " + Array.getLength(intArr));
    }
}
```

## 1.6 基本类型与包装类的 Class 对象

```java
package com.example.reflection;

/**
 * int.class != Integer.class
 * 这是反射中最常见的错误原因之一！
 */
public class PrimitiveVsWrapperDemo {

    public static void main(String[] args) throws Exception {

        System.out.println("=== 基本类型 Class ===");
        System.out.println("int.class:     " + int.class);     // int
        System.out.println("long.class:    " + long.class);    // long
        System.out.println("double.class:  " + double.class);  // double
        System.out.println("boolean.class: " + boolean.class); // boolean
        System.out.println("void.class:    " + void.class);    // void

        System.out.println("\n=== 包装类 Class ===");
        System.out.println("Integer.class: " + Integer.class);  // class java.lang.Integer
        System.out.println("Long.class:    " + Long.class);     // class java.lang.Long
        System.out.println("Double.class:  " + Double.class);   // class java.lang.Double

        System.out.println("\n=== 核心区别 ===");
        System.out.println("int.class == Integer.class: " + (int.class == Integer.class)); // false
        System.out.println("int.class.isPrimitive():    " + int.class.isPrimitive()); // true
        System.out.println("Integer.class.isPrimitive():" + Integer.class.isPrimitive()); // false

        System.out.println("\n=== TYPE 字段（包装类 -> 基本类型 Class）===");
        System.out.println("Integer.TYPE == int.class:    " + (Integer.TYPE == int.class)); // true
        System.out.println("Long.TYPE    == long.class:   " + (Long.TYPE == long.class));
        System.out.println("Double.TYPE  == double.class: " + (Double.TYPE == double.class));

        System.out.println("\n=== 对反射方法查找的影响（常见错误）===");
        try {
            // 错误：String.valueOf(int) 参数是 int，不是 Integer
            String.class.getMethod("valueOf", Integer.class);
        } catch (NoSuchMethodException e) {
            System.out.println("Integer.class 找不到: NoSuchMethodException !");
        }
        // 正确：使用 int.class
        java.lang.reflect.Method m = String.class.getMethod("valueOf", int.class);
        System.out.println("int.class 找到: " + m.toGenericString());
        System.out.println("调用: " + m.invoke(null, 2025)); // "2025"

        System.out.println("\n=== 基本类型完整对照 ===");
        Class<?>[] prims    = { boolean.class, byte.class, char.class, short.class,
                                 int.class, long.class, float.class, double.class };
        Class<?>[] wrappers = { Boolean.class, Byte.class, Character.class, Short.class,
                                 Integer.class, Long.class, Float.class, Double.class };
        System.out.printf("%-12s %-22s %-6s%n", "基本类型", "包装类", "相等?");
        System.out.println("-".repeat(42));
        for (int i = 0; i < prims.length; i++) {
            System.out.printf("%-12s %-22s %-6s%n",
                prims[i].getName(), wrappers[i].getName(), prims[i] == wrappers[i]);
        }
    }
}
```

---


---

# Part 2: 反射 API 完整详解

## 2.1 获取类信息

```java
import java.lang.reflect.*;

public class ClassInfoDemo {
    public static void main(String[] args) {
        Class<?> clazz = ArrayList.class;

        // 类名
        System.out.println("简单名: " + clazz.getSimpleName());       // ArrayList
        System.out.println("全限定名: " + clazz.getName());           // java.util.ArrayList
        System.out.println("规范名: " + clazz.getCanonicalName());    // java.util.ArrayList

        // 修饰符
        int mod = clazz.getModifiers();
        System.out.println("public: " + Modifier.isPublic(mod));
        System.out.println("abstract: " + Modifier.isAbstract(mod));

        // 父类与接口
        System.out.println("父类: " + clazz.getSuperclass().getName());
        for (Class<?> iface : clazz.getInterfaces()) {
            System.out.println("接口: " + iface.getName());
        }

        // 包信息
        Package pkg = clazz.getPackage();
        System.out.println("包名: " + pkg.getName());

        // 是否为数组/接口/枚举/注解/基本类型
        System.out.println("isArray: " + clazz.isArray());
        System.out.println("isInterface: " + clazz.isInterface());
        System.out.println("isEnum: " + clazz.isEnum());
        System.out.println("isAnnotation: " + clazz.isAnnotation());
        System.out.println("isPrimitive: " + clazz.isPrimitive());
    }
}
```

## 2.2 Constructor 反射

```java
import java.lang.reflect.*;

public class ConstructorDemo {

    public static class Person {
        private String name;
        private int age;

        public Person() { this.name = "unknown"; this.age = 0; }

        public Person(String name) { this.name = name; this.age = 0; }

        private Person(String name, int age) { this.name = name; this.age = age; }

        @Override
        public String toString() { return "Person{name=" + name + ", age=" + age + "}"; }
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = Person.class;

        // 获取所有 public 构造器
        System.out.println("=== getConstructors (public only) ===");
        for (Constructor<?> c : clazz.getConstructors()) {
            System.out.println(c);
        }

        // 获取所有构造器（含 private）
        System.out.println("=== getDeclaredConstructors (all) ===");
        for (Constructor<?> c : clazz.getDeclaredConstructors()) {
            System.out.println(c);
        }

        // 调用无参构造器
        Constructor<?> noArg = clazz.getConstructor();
        Person p1 = (Person) noArg.newInstance();
        System.out.println("无参构造: " + p1);

        // 调用 public 单参构造器
        Constructor<?> oneArg = clazz.getConstructor(String.class);
        Person p2 = (Person) oneArg.newInstance("Alice");
        System.out.println("单参构造: " + p2);

        // 调用 private 双参构造器（需要 setAccessible）
        Constructor<?> twoArg = clazz.getDeclaredConstructor(String.class, int.class);
        twoArg.setAccessible(true);
        Person p3 = (Person) twoArg.newInstance("Bob", 30);
        System.out.println("私有构造: " + p3);

        // 构造器元数据
        System.out.println("参数类型: ");
        for (Parameter param : twoArg.getParameters()) {
            System.out.println("  " + param.getType().getSimpleName() + " " + param.getName());
        }
    }
}
```

## 2.3 Field 反射

```java
import java.lang.reflect.*;

public class FieldDemo {

    public static class Employee {
        public static final String COMPANY = "MegaCorp";
        public String name;
        protected int age;
        private double salary;

        public Employee(String name, int age, double salary) {
            this.name = name;
            this.age = age;
            this.salary = salary;
        }
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = Employee.class;

        // getFields() - public 字段（含继承）
        System.out.println("=== getFields ===");
        for (Field f : clazz.getFields()) {
            System.out.printf("%-20s %-10s %s%n",
                f.getName(), f.getType().getSimpleName(),
                Modifier.toString(f.getModifiers()));
        }

        // getDeclaredFields() - 本类所有字段（不含继承）
        System.out.println("=== getDeclaredFields ===");
        for (Field f : clazz.getDeclaredFields()) {
            System.out.printf("%-20s %-10s %s%n",
                f.getName(), f.getType().getSimpleName(),
                Modifier.toString(f.getModifiers()));
        }

        Employee emp = new Employee("Alice", 25, 8000.0);

        // 读取 public 字段
        Field nameField = clazz.getField("name");
        System.out.println("name = " + nameField.get(emp));

        // 读取 private 字段
        Field salaryField = clazz.getDeclaredField("salary");
        salaryField.setAccessible(true);
        System.out.println("salary = " + salaryField.get(emp));

        // 修改 private 字段
        salaryField.set(emp, 10000.0);
        System.out.println("修改后 salary = " + salaryField.get(emp));

        // 读取静态 final 字段
        Field companyField = clazz.getField("COMPANY");
        System.out.println("COMPANY = " + companyField.get(null)); // static 字段传 null
    }
}
```

## 2.4 Method 反射

```java
import java.lang.reflect.*;
import java.util.Arrays;

public class MethodDemo {

    public static class Calculator {
        public int add(int a, int b) { return a + b; }
        public double multiply(double a, double b) { return a * b; }
        private String secret(String msg) { return "[SECRET] " + msg; }
        public static int staticAdd(int a, int b) { return a + b; }

        public void varargMethod(String... args) {
            System.out.println("varargs: " + Arrays.toString(args));
        }
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = Calculator.class;

        // getMethods() - public 方法（含继承自 Object）
        System.out.println("=== getMethods ===");
        for (Method m : clazz.getMethods()) {
            System.out.println(m.getName() + " -> " + m.getReturnType().getSimpleName());
        }

        // getDeclaredMethods() - 本类所有方法
        System.out.println("=== getDeclaredMethods ===");
        for (Method m : clazz.getDeclaredMethods()) {
            System.out.printf("%-20s 返回值:%-12s 参数:%s%n",
                m.getName(),
                m.getReturnType().getSimpleName(),
                Arrays.toString(m.getParameterTypes()));
        }

        Calculator calc = new Calculator();

        // 调用 public 方法
        Method addMethod = clazz.getMethod("add", int.class, int.class);
        int result = (int) addMethod.invoke(calc, 3, 5);
        System.out.println("add(3,5) = " + result);

        // 调用 private 方法
        Method secretMethod = clazz.getDeclaredMethod("secret", String.class);
        secretMethod.setAccessible(true);
        String s = (String) secretMethod.invoke(calc, "hello");
        System.out.println("secret: " + s);

        // 调用静态方法（传 null 作为实例）
        Method staticMethod = clazz.getMethod("staticAdd", int.class, int.class);
        int sr = (int) staticMethod.invoke(null, 10, 20);
        System.out.println("staticAdd(10,20) = " + sr);

        // 调用可变参数方法
        Method varargMethod = clazz.getMethod("varargMethod", String[].class);
        varargMethod.invoke(calc, (Object) new String[]{"a", "b", "c"});

        // 方法元数据
        System.out.println("add 方法异常类型: " + Arrays.toString(addMethod.getExceptionTypes()));
        System.out.println("add 方法是否 bridge: " + addMethod.isBridge());
        System.out.println("add 方法是否 synthetic: " + addMethod.isSynthetic());
    }
}
```

## 2.5 Annotation 反射

```java
import java.lang.annotation.*;
import java.lang.reflect.*;

// 自定义注解
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD})
@interface MyAnnotation {
    String value() default "default";
    int priority() default 1;
}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.PARAMETER)
@interface Param {
    String name();
}

public class AnnotationDemo {

    @MyAnnotation(value = "classLevel", priority = 10)
    public static class AnnotatedService {

        @MyAnnotation("fieldLevel")
        public String data = "test";

        @MyAnnotation(value = "methodLevel", priority = 5)
        public String process(@Param(name = "input") String input,
                               @Param(name = "flag") boolean flag) {
            return input + flag;
        }
    }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = AnnotatedService.class;

        // 类上的注解
        MyAnnotation classAnno = clazz.getAnnotation(MyAnnotation.class);
        if (classAnno != null) {
            System.out.println("类注解 value=" + classAnno.value()
                + " priority=" + classAnno.priority());
        }

        // 字段上的注解
        Field dataField = clazz.getField("data");
        MyAnnotation fieldAnno = dataField.getAnnotation(MyAnnotation.class);
        System.out.println("字段注解 value=" + fieldAnno.value());

        // 方法上的注解
        Method processMethod = clazz.getMethod("process", String.class, boolean.class);
        MyAnnotation methodAnno = processMethod.getAnnotation(MyAnnotation.class);
        System.out.println("方法注解 value=" + methodAnno.value()
            + " priority=" + methodAnno.priority());

        // 方法参数上的注解
        Annotation[][] paramAnnotations = processMethod.getParameterAnnotations();
        for (int i = 0; i < paramAnnotations.length; i++) {
            for (Annotation anno : paramAnnotations[i]) {
                if (anno instanceof Param) {
                    System.out.println("参数[" + i + "] name=" + ((Param) anno).name());
                }
            }
        }

        // 判断是否有某注解
        System.out.println("类是否有 MyAnnotation: "
            + clazz.isAnnotationPresent(MyAnnotation.class));

        // 获取所有注解
        System.out.println("类的所有注解:");
        for (Annotation anno : clazz.getAnnotations()) {
            System.out.println("  " + anno.annotationType().getSimpleName());
        }
    }
}
```

## 2.6 反射与数组

```java
import java.lang.reflect.Array;

public class ArrayReflectionDemo {
    public static void main(String[] args) {
        // 动态创建数组
        int[] intArray = (int[]) Array.newInstance(int.class, 5);
        for (int i = 0; i < intArray.length; i++) {
            Array.setInt(intArray, i, i * i);
        }
        System.out.println("int[]: " + java.util.Arrays.toString(intArray));

        // 创建多维数组
        int[][] matrix = (int[][]) Array.newInstance(int.class, 3, 4);
        System.out.println("matrix[2][3] = " + matrix[2][3]);

        // 获取数组 Class
        Class<?> intArrClass = int[].class;
        System.out.println("数组 isArray: " + intArrClass.isArray());
        System.out.println("组件类型: " + intArrClass.getComponentType()); // int

        // 获取数组长度
        String[] strArr = {"a", "b", "c"};
        System.out.println("数组长度: " + Array.getLength(strArr));

        // 通用数组复制
        String[] copy = (String[]) Array.newInstance(String.class, strArr.length);
        System.arraycopy(strArr, 0, copy, 0, strArr.length);
        System.out.println("copy: " + java.util.Arrays.toString(copy));
    }
}
```


---

# Part 3: 反射性能优化

## 3.1 反射慢的原因分析

```
反射调用链路（未优化）:
Method.invoke()
  └─ Reflection.checkAccess()       <- 权限检查（每次）
       └─ NativeMethodAccessorImpl  <- JNI 调用（最慢）
            └─ Java 方法体

优化后调用链路（setAccessible + inflation）:
Method.invoke()
  └─ GeneratedMethodAccessor1       <- JVM 生成的字节码（接近直接调用）
       └─ Java 方法体
```

性能层次（从慢到快）：

| 方式 | 相对性能 | 说明 |
|------|---------|------|
| Method.invoke (原始) | 1x | 每次权限检查 + JNI |
| Method.invoke (setAccessible) | ~3x | 跳过权限检查 |
| MethodHandle.invoke | ~8x | invokedynamic |
| LambdaMetafactory | ~30x | 编译为函数接口 |
| 直接调用 | 100x | 基准 |

## 3.2 setAccessible 优化

```java
import java.lang.reflect.*;

public class SetAccessibleDemo {
    static class Target {
        private int value = 42;
        private int compute(int x) { return x * x; }
    }

    public static void main(String[] args) throws Exception {
        Target t = new Target();
        Method method = Target.class.getDeclaredMethod("compute", int.class);

        // 不调用 setAccessible：每次检查权限
        long start = System.nanoTime();
        for (int i = 0; i < 100_000; i++) {
            try { method.invoke(t, i); } catch (Exception e) { /*ignore*/ }
        }
        System.out.println("未设置 setAccessible: " + (System.nanoTime() - start) / 1_000_000 + "ms");

        // 调用 setAccessible(true)：跳过权限检查
        method.setAccessible(true);
        start = System.nanoTime();
        for (int i = 0; i < 100_000; i++) {
            method.invoke(t, i);
        }
        System.out.println("已设置 setAccessible: " + (System.nanoTime() - start) / 1_000_000 + "ms");
    }
}
```

## 3.3 缓存反射对象

```java
import java.lang.reflect.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.Map;

// 反射缓存工具类
public class ReflectionCache {
    // Method 缓存：类名 + 方法签名 -> Method
    private static final Map<String, Method> METHOD_CACHE = new ConcurrentHashMap<>();
    // Field 缓存：类名 + 字段名 -> Field
    private static final Map<String, Field> FIELD_CACHE = new ConcurrentHashMap<>();

    public static Method getMethod(Class<?> clazz, String name, Class<?>... paramTypes) {
        String key = clazz.getName() + "#" + name + buildParamKey(paramTypes);
        return METHOD_CACHE.computeIfAbsent(key, k -> {
            try {
                Method m = clazz.getDeclaredMethod(name, paramTypes);
                m.setAccessible(true);
                return m;
            } catch (NoSuchMethodException e) {
                throw new RuntimeException(e);
            }
        });
    }

    public static Field getField(Class<?> clazz, String fieldName) {
        String key = clazz.getName() + "#" + fieldName;
        return FIELD_CACHE.computeIfAbsent(key, k -> {
            try {
                Field f = clazz.getDeclaredField(fieldName);
                f.setAccessible(true);
                return f;
            } catch (NoSuchFieldException e) {
                // 向上查找父类
                Class<?> superClass = clazz.getSuperclass();
                if (superClass != null) {
                    return getField(superClass, fieldName);
                }
                throw new RuntimeException(e);
            }
        });
    }

    private static String buildParamKey(Class<?>[] paramTypes) {
        if (paramTypes == null || paramTypes.length == 0) return "()";
        StringBuilder sb = new StringBuilder("(");
        for (Class<?> t : paramTypes) sb.append(t.getName()).append(",");
        sb.append(")");
        return sb.toString();
    }

    // 使用示例
    public static void main(String[] args) throws Exception {
        class Service { private String greet(String name) { return "Hello, " + name; } }
        Service svc = new Service();

        // 第一次：从反射获取并缓存
        Method m = getMethod(Service.class, "greet", String.class);
        System.out.println(m.invoke(svc, "World"));

        // 第二次：直接从缓存获取（无反射开销）
        System.out.println(m.invoke(svc, "Java"));
    }
}
```

## 3.4 MethodHandle 优化

```java
import java.lang.invoke.*;

public class MethodHandleDemo {

    static class StringUtils {
        public static String toUpper(String s) { return s.toUpperCase(); }
        private int len(String s) { return s.length(); }
    }

    public static void main(String[] args) throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();

        // 1. 获取静态方法 MethodHandle
        MethodHandle toUpper = lookup.findStatic(
            StringUtils.class,
            "toUpper",
            MethodType.methodType(String.class, String.class)
        );
        System.out.println((String) toUpper.invoke("hello")); // HELLO

        // 2. 获取实例方法 MethodHandle（需要 privateLookupIn 访问 private）
        MethodHandles.Lookup privateLookup = MethodHandles.privateLookupIn(
            StringUtils.class, MethodHandles.lookup());
        MethodHandle len = privateLookup.findVirtual(
            StringUtils.class,
            "len",
            MethodType.methodType(int.class, String.class)
        );
        StringUtils utils = new StringUtils();
        System.out.println((int) len.invoke(utils, "hello")); // 5

        // 3. invokeExact（最快，类型必须精确匹配）
        String result = (String) toUpper.invokeExact("world");
        System.out.println(result); // WORLD

        // 4. MethodHandle 绑定参数（Currying）
        MethodHandle bound = toUpper.bindTo(null); // static 无需绑定
        // 对实例方法绑定
        MethodHandle boundLen = len.bindTo(utils);
        System.out.println((int) boundLen.invoke("test")); // 4

        // 5. 字段 getter/setter
        MethodHandles.Lookup pubLookup = MethodHandles.publicLookup();
        MethodHandle getter = pubLookup.findGetter(
            StringBuilder.class, "value", char[].class); // 演示用

        // 6. 性能对比 - MethodHandle vs 反射
        long start = System.nanoTime();
        for (int i = 0; i < 1_000_000; i++) {
            toUpper.invoke("test");
        }
        System.out.println("MethodHandle: " + (System.nanoTime() - start) / 1_000_000 + "ms");
    }
}
```

## 3.5 LambdaMetafactory 极致优化

```java
import java.lang.invoke.*;
import java.util.function.*;

public class LambdaMetafactoryDemo {

    static class Person {
        private String name;
        private int age;
        public Person(String name, int age) { this.name = name; this.age = age; }
        public String getName() { return name; }
        public int getAge() { return age; }
        public void setName(String name) { this.name = name; }
    }

    @FunctionalInterface
    interface StringGetter<T> { String get(T obj); }

    @FunctionalInterface
    interface IntGetter<T> { int get(T obj); }

    @SuppressWarnings("unchecked")
    public static <T> StringGetter<T> createStringGetter(Class<T> clazz, String methodName)
            throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.privateLookupIn(clazz, MethodHandles.lookup());
        MethodHandle handle = lookup.findVirtual(clazz, methodName,
            MethodType.methodType(String.class));
        CallSite callSite = LambdaMetafactory.metafactory(
            lookup,
            "get",                                          // 函数接口方法名
            MethodType.methodType(StringGetter.class),      // 工厂签名
            MethodType.methodType(String.class, Object.class), // 泛型签名
            handle,                                         // 目标方法
            MethodType.methodType(String.class, clazz)      // 具体签名
        );
        return (StringGetter<T>) callSite.getTarget().invoke();
    }

    public static void main(String[] args) throws Throwable {
        // 创建一次，重复使用（接近直接调用的性能）
        StringGetter<Person> nameGetter = createStringGetter(Person.class, "getName");

        Person p = new Person("Alice", 30);
        System.out.println(nameGetter.get(p)); // Alice

        // 性能基准
        long start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            nameGetter.get(p);
        }
        System.out.println("LambdaMetafactory: " + (System.nanoTime() - start) / 1_000_000 + "ms");

        // 对比反射
        java.lang.reflect.Method m = Person.class.getDeclaredMethod("getName");
        m.setAccessible(true);
        start = System.nanoTime();
        for (int i = 0; i < 10_000_000; i++) {
            m.invoke(p);
        }
        System.out.println("Reflection: " + (System.nanoTime() - start) / 1_000_000 + "ms");
    }
}
```

## 3.6 性能优化总结

```
+--------------------------------------------------+
|          反射性能优化层次图                        |
+--------------------------------------------------+
|                                                  |
|  Level 1: 直接调用（基准 100%）                  |
|           obj.method()                           |
|                                                  |
|  Level 2: LambdaMetafactory（~95%）              |
|           一次生成，像 Lambda 一样调用            |
|                                                  |
|  Level 3: MethodHandle.invokeExact（~85%）       |
|           类型精确匹配，JIT 可内联               |
|                                                  |
|  Level 4: MethodHandle.invoke（~70%）            |
|           有类型转换开销                         |
|                                                  |
|  Level 5: Method.invoke（缓存+accessible，~30%） |
|           JVM inflation 后生成字节码访问器       |
|                                                  |
|  Level 6: Method.invoke（未优化，~10%）          |
|           每次权限检查 + JNI 调用               |
+--------------------------------------------------+
```


---

# Part 4: 泛型与反射

## 4.1 类型擦除原理

```
Java 泛型编译期行为：

源码：List<String> list = new ArrayList<>();
字节码：List list = new ArrayList();   <- 擦除泛型

类型擦除规则：
  T            -> Object
  T extends X  -> X
  ? super Y    -> Object

保留在字节码中的泛型信息（Signature 属性）：
  - 类声明上的泛型
  - 方法的返回类型和参数类型
  - 字段的泛型类型
  但 局部变量 的泛型信息不保留！
```

```java
import java.lang.reflect.*;
import java.util.*;

public class TypeErasureDemo {
    // 这些泛型信息在字节码中被保留（Signature 属性）
    private List<String> stringList;
    private Map<String, List<Integer>> complexMap;

    public <T extends Comparable<T>> T findMax(List<T> list) { return null; }

    public static void main(String[] args) throws Exception {
        Class<?> clazz = TypeErasureDemo.class;

        // 运行时无法区分 List<String> 和 List<Integer>
        List<String> ls = new ArrayList<>();
        List<Integer> li = new ArrayList<>();
        System.out.println(ls.getClass() == li.getClass()); // true

        // 但字段的泛型类型信息保留在 Field 的 GenericType 中
        Field f1 = clazz.getDeclaredField("stringList");
        System.out.println("原始类型: " + f1.getType());           // java.util.List
        System.out.println("泛型类型: " + f1.getGenericType());     // java.util.List<java.lang.String>

        Field f2 = clazz.getDeclaredField("complexMap");
        Type generic = f2.getGenericType();
        System.out.println("复杂泛型: " + generic);
        // java.util.Map<java.lang.String, java.util.List<java.lang.Integer>>
    }
}
```

## 4.2 四种泛型类型接口

```java
import java.lang.reflect.*;
import java.util.*;

public class GenericTypeDemo {
    // ParameterizedType: 参数化类型，如 List<String>
    private List<String> paramField;

    // GenericArrayType: 泛型数组，如 T[]
    // TypeVariable: 类型变量，如 T
    // WildcardType: 通配符，如 ? extends Number

    public <T> T[] genericArray(T[] arr) { return arr; }
    public void wildcardMethod(List<? extends Number> list) {}

    public static void main(String[] args) throws Exception {
        Class<?> clazz = GenericTypeDemo.class;

        // 1. ParameterizedType
        Field pf = clazz.getDeclaredField("paramField");
        Type pt = pf.getGenericType();
        if (pt instanceof ParameterizedType) {
            ParameterizedType pType = (ParameterizedType) pt;
            System.out.println("rawType: " + pType.getRawType());        // class java.util.List
            System.out.println("ownerType: " + pType.getOwnerType());   // null
            Type[] args2 = pType.getActualTypeArguments();
            System.out.println("actualTypes: " + Arrays.toString(args2)); // [class java.lang.String]
        }

        // 2. GenericArrayType
        Method gam = clazz.getDeclaredMethod("genericArray", Object[].class);
        Type returnType = gam.getGenericReturnType();
        System.out.println("返回类型 class: " + returnType.getClass().getSimpleName());
        // GenericArrayType
        if (returnType instanceof GenericArrayType) {
            GenericArrayType gat = (GenericArrayType) returnType;
            System.out.println("组件类型: " + gat.getGenericComponentType()); // T
        }

        // 3. TypeVariable
        TypeVariable<?>[] tvars = gam.getTypeParameters();
        for (TypeVariable<?> tv : tvars) {
            System.out.println("TypeVariable 名称: " + tv.getName()); // T
            System.out.println("TypeVariable 边界: " + Arrays.toString(tv.getBounds())); // [class java.lang.Object]
        }

        // 4. WildcardType
        Method wm = clazz.getDeclaredMethod("wildcardMethod", List.class);
        Type[] paramTypes = wm.getGenericParameterTypes();
        ParameterizedType listType = (ParameterizedType) paramTypes[0];
        Type wildcard = listType.getActualTypeArguments()[0];
        if (wildcard instanceof WildcardType) {
            WildcardType wt = (WildcardType) wildcard;
            System.out.println("上界: " + Arrays.toString(wt.getUpperBounds())); // [class java.lang.Number]
            System.out.println("下界: " + Arrays.toString(wt.getLowerBounds())); // []
        }
    }
}
```

## 4.3 TypeToken 模式

```java
import java.lang.reflect.*;

// TypeToken 用于捕获泛型类型信息
public abstract class TypeToken<T> {
    private final Type type;

    protected TypeToken() {
        // 通过匿名子类的泛型超类获取实际类型参数
        Type superClass = getClass().getGenericSuperclass();
        if (superClass instanceof ParameterizedType) {
            this.type = ((ParameterizedType) superClass).getActualTypeArguments()[0];
        } else {
            throw new IllegalArgumentException("TypeToken 必须带泛型参数");
        }
    }

    public Type getType() { return type; }

    @Override
    public String toString() { return type.toString(); }

    // 工厂方法（Java 8+ lambda 不能用，需匿名类）
    public static <T> TypeToken<T> of(Class<T> clazz) {
        return new TypeToken<T>(clazz) {};
    }

    // 私有构造（工厂方法专用）
    private TypeToken(Type type) {
        this.type = type;
    }

    // 使用示例
    public static void main(String[] args) {
        // 捕获 List<String> 的完整类型信息
        TypeToken<java.util.List<String>> token =
            new TypeToken<java.util.List<String>>() {};
        System.out.println(token.getType());
        // java.util.List<java.lang.String>

        // 捕获 Map<String, List<Integer>>
        TypeToken<java.util.Map<String, java.util.List<Integer>>> mapToken =
            new TypeToken<java.util.Map<String, java.util.List<Integer>>>() {};
        System.out.println(mapToken.getType());
        // java.util.Map<java.lang.String, java.util.List<java.lang.Integer>>

        // 实际应用：JSON 反序列化时传递类型
        // ObjectMapper mapper = new ObjectMapper();
        // List<String> list = mapper.readValue(json, new TypeReference<List<String>>(){});
    }
}
```

## 4.4 泛型与反射的实战应用

```java
import java.lang.reflect.*;
import java.util.*;

// 通用泛型工具
public class GenericUtils {

    /**
     * 获取类实现的某接口的泛型参数
     * 例如：class MyDao implements Dao<User> -> 返回 User.class
     */
    @SuppressWarnings("unchecked")
    public static <T> Class<T> getInterfaceGenericType(Class<?> clazz, Class<?> iface, int index) {
        for (Type type : clazz.getGenericInterfaces()) {
            if (type instanceof ParameterizedType) {
                ParameterizedType pt = (ParameterizedType) type;
                if (pt.getRawType() == iface) {
                    Type[] typeArgs = pt.getActualTypeArguments();
                    if (index < typeArgs.length && typeArgs[index] instanceof Class) {
                        return (Class<T>) typeArgs[index];
                    }
                }
            }
        }
        return null;
    }

    /**
     * 获取父类的泛型参数
     * 例如：class UserService extends BaseService<User> -> 返回 User.class
     */
    @SuppressWarnings("unchecked")
    public static <T> Class<T> getSuperclassGenericType(Class<?> clazz, int index) {
        Type superclass = clazz.getGenericSuperclass();
        if (superclass instanceof ParameterizedType) {
            ParameterizedType pt = (ParameterizedType) superclass;
            Type[] typeArgs = pt.getActualTypeArguments();
            if (index < typeArgs.length && typeArgs[index] instanceof Class) {
                return (Class<T>) typeArgs[index];
            }
        }
        return null;
    }

    // 示例
    interface Repository<T, ID> { T findById(ID id); }
    static class User { String name; }
    static class UserRepository implements Repository<User, Long> {
        public User findById(Long id) { return new User(); }
    }

    static abstract class BaseService<T> { protected T entity; }
    static class UserService extends BaseService<User> {}

    public static void main(String[] args) {
        Class<User> userClass1 = getInterfaceGenericType(UserRepository.class, Repository.class, 0);
        System.out.println("接口第0个泛型: " + userClass1); // class ...User

        Class<Long> idClass = getInterfaceGenericType(UserRepository.class, Repository.class, 1);
        System.out.println("接口第1个泛型: " + idClass); // class java.lang.Long

        Class<User> userClass2 = getSuperclassGenericType(UserService.class, 0);
        System.out.println("父类泛型: " + userClass2); // class ...User
    }
}
```


---

# Part 5: JDK 动态代理深度解析

## 5.1 JDK 动态代理架构

```
+------------------+         +--------------------+
|   Client Code    |         |   Proxy$0 (生成)   |
|                  |  调用   |                    |
|  service.method()|-------->| method() {         |
+------------------+         |   h.invoke(        |
                             |     this, method,  |
                             |     args)          |
                             | }                  |
                             +--------------------+
                                      |
                                      | invoke()
                                      v
                             +--------------------+
                             |  InvocationHandler |
                             |  (自定义逻辑)      |
                             |  - 前置处理        |
                             |  - method.invoke() |
                             |  - 后置处理        |
                             +--------------------+
                                      |
                                      v
                             +--------------------+
                             |  Target (真实对象) |
                             +--------------------+
```

## 5.2 基础实现

```java
import java.lang.reflect.*;

public class JdkProxyBasic {

    // 目标接口
    interface UserService {
        String findById(long id);
        boolean save(String name);
        void delete(long id);
    }

    // 真实实现
    static class UserServiceImpl implements UserService {
        public String findById(long id) {
            System.out.println("  [DB] SELECT * FROM user WHERE id=" + id);
            return "User#" + id;
        }
        public boolean save(String name) {
            System.out.println("  [DB] INSERT INTO user(name) VALUES('" + name + "')");
            return true;
        }
        public void delete(long id) {
            System.out.println("  [DB] DELETE FROM user WHERE id=" + id);
        }
    }

    // 日志代理处理器
    static class LoggingHandler implements InvocationHandler {
        private final Object target;

        LoggingHandler(Object target) { this.target = target; }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("[LOG] 调用方法: " + method.getName()
                + "  参数: " + java.util.Arrays.toString(args));
            long start = System.currentTimeMillis();
            try {
                Object result = method.invoke(target, args);
                System.out.println("[LOG] 返回值: " + result
                    + "  耗时: " + (System.currentTimeMillis() - start) + "ms");
                return result;
            } catch (InvocationTargetException e) {
                System.out.println("[LOG] 异常: " + e.getCause().getMessage());
                throw e.getCause();
            }
        }
    }

    @SuppressWarnings("unchecked")
    public static <T> T createProxy(T target) {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new LoggingHandler(target)
        );
    }

    public static void main(String[] args) {
        UserService real = new UserServiceImpl();
        UserService proxy = createProxy(real);

        System.out.println("=== findById ===");
        String user = proxy.findById(1L);

        System.out.println("=== save ===");
        proxy.save("Alice");

        System.out.println("=== 代理类信息 ===");
        System.out.println("代理类名: " + proxy.getClass().getName());
        System.out.println("是否是代理: " + Proxy.isProxyClass(proxy.getClass()));
        System.out.println("Handler: " + Proxy.getInvocationHandler(proxy).getClass().getSimpleName());
    }
}
```

## 5.3 代理链（多重代理）

```java
import java.lang.reflect.*;

public class ProxyChainDemo {

    interface OrderService {
        String createOrder(String product, double price);
    }

    static class OrderServiceImpl implements OrderService {
        public String createOrder(String product, double price) {
            System.out.println("  创建订单: " + product + " ¥" + price);
            return "ORDER-" + System.currentTimeMillis();
        }
    }

    // 通用代理工厂
    static class ChainableProxy implements InvocationHandler {
        private final Object target;
        private final String name;

        ChainableProxy(Object target, String name) {
            this.target = target;
            this.name = name;
        }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            System.out.println("[" + name + "] before: " + method.getName());
            Object result = method.invoke(target, args);
            System.out.println("[" + name + "] after: " + method.getName());
            return result;
        }
    }

    @SuppressWarnings("unchecked")
    static <T> T wrap(T target, String name) {
        return (T) Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            new ChainableProxy(target, name)
        );
    }

    public static void main(String[] args) {
        OrderService service = new OrderServiceImpl();

        // 构建代理链：事务 -> 缓存 -> 日志 -> 真实对象
        OrderService withLog = wrap(service, "LOG");
        OrderService withCache = wrap(withLog, "CACHE");
        OrderService withTx = wrap(withCache, "TX");

        System.out.println("=== 代理链调用 ===");
        String orderId = withTx.createOrder("iPhone", 5999.0);
        System.out.println("订单ID: " + orderId);
    }
}
```

## 5.4 JDK 代理字节码分析

```java
import java.lang.reflect.*;
import sun.misc.ProxyGenerator;
import java.io.*;
import java.nio.file.*;

public class ProxyBytecodeAnalysis {

    interface Demo { String hello(String name); }

    public static void main(String[] args) throws Exception {
        // 生成代理类字节码（JDK 内部方法，仅演示）
        // 通过系统属性可以保存代理类字节码
        System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        // 或 JDK 8+: -Djdk.proxy.ProxyGenerator.saveGeneratedFiles=true

        Demo proxy = (Demo) Proxy.newProxyInstance(
            Demo.class.getClassLoader(),
            new Class[]{Demo.class},
            (p, method, a) -> "Hello, " + a[0]
        );

        // 代理类的结构（反编译后大致如下）：
        /*
        public final class $Proxy0 extends Proxy implements Demo {
            private static Method m1; // hashCode
            private static Method m2; // equals
            private static Method m3; // toString
            private static Method m4; // hello

            static {
                m4 = Class.forName("Demo").getMethod("hello", String.class);
                // ...
            }

            public $Proxy0(InvocationHandler h) { super(h); }

            public final String hello(String name) {
                return (String) h.invoke(this, m4, new Object[]{name});
            }
        }
        */

        System.out.println(proxy.hello("World")); // Hello, World

        // 代理类继承链
        Class<?> proxyClass = proxy.getClass();
        System.out.println("父类: " + proxyClass.getSuperclass()); // class java.lang.reflect.Proxy
        for (Class<?> iface : proxyClass.getInterfaces()) {
            System.out.println("接口: " + iface.getName());
        }
    }
}
```

## 5.5 JDK 代理的限制与解决方案

```
JDK 动态代理限制：
1. 只能代理接口（目标类必须实现接口）
2. 无法代理 final 类和方法
3. 代理类继承自 java.lang.reflect.Proxy，无法再继承其他类

解决方案：
1. 目标类没有接口 -> 使用 CGLIB 代理
2. 需要代理 final 方法 -> 使用 ByteBuddy/Javassist
3. 性能敏感场景 -> 使用 LambdaMetafactory 或 MethodHandle

适合 JDK 代理的场景：
- Spring AOP（目标类实现了接口）
- RPC 框架（Dubbo、Feign 的接口代理）
- 测试框架（Mockito 的接口 Mock）
```


---

# Part 6: CGLIB 动态代理深度解析

## 6.1 CGLIB 代理原理

```
CGLIB 使用 ASM 字节码框架，在运行时生成目标类的子类来实现代理：

                    +------------------+
                    |   Target Class   |
                    |  (普通类，无需接口)|
                    +------------------+
                             |
                    extends  |
                             v
                    +------------------+
                    | Target$$EnhancedBySpringCGLIB |
                    |  (运行时生成子类)  |
                    |                  |
                    |  method() {      |
                    |    interceptor   |
                    |    .intercept()  |
                    |  }               |
                    +------------------+

限制：
  - 无法代理 final 类（不能继承）
  - 无法代理 final/static 方法（不能覆盖）
  - 需要有无参构造器（或使用 Objenesis 跳过）
```

## 6.2 基础 CGLIB 代理

```java
import net.sf.cglib.proxy.*;
import java.lang.reflect.Method;

public class CglibBasicDemo {

    // 目标类（无需实现接口）
    static class ProductService {
        public String findProduct(long id) {
            System.out.println("  查询产品: " + id);
            return "Product#" + id;
        }

        public boolean addProduct(String name, double price) {
            System.out.println("  新增产品: " + name + " ¥" + price);
            return true;
        }

        public final void internalMethod() {
            System.out.println("  final 方法（不会被代理）");
        }
    }

    // 拦截器
    static class LogInterceptor implements MethodInterceptor {
        @Override
        public Object intercept(Object obj, Method method, Object[] args,
                                MethodProxy proxy) throws Throwable {
            System.out.println("[CGLIB] 拦截方法: " + method.getName());
            long start = System.currentTimeMillis();
            // 调用父类方法（invokeSuper 比 invoke 性能更好）
            Object result = proxy.invokeSuper(obj, args);
            System.out.println("[CGLIB] 耗时: " + (System.currentTimeMillis() - start) + "ms");
            return result;
        }
    }

    public static void main(String[] args) {
        // 创建 Enhancer
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(ProductService.class);
        enhancer.setCallback(new LogInterceptor());

        // 生成代理对象（调用无参构造）
        ProductService proxy = (ProductService) enhancer.create();

        System.out.println("=== findProduct ===");
        proxy.findProduct(100L);

        System.out.println("=== addProduct ===");
        proxy.addProduct("MacBook", 12999.0);

        System.out.println("=== final 方法 ===");
        proxy.internalMethod(); // 直接调用，不经过拦截器

        System.out.println("\n代理类名: " + proxy.getClass().getName());
        System.out.println("父类: " + proxy.getClass().getSuperclass().getName());
    }
}
```

## 6.3 CallbackFilter 选择性代理

```java
import net.sf.cglib.proxy.*;
import java.lang.reflect.Method;

public class CallbackFilterDemo {

    static class OrderService {
        public String queryOrder(long id) { return "ORDER#" + id; }
        public void createOrder(String product) { System.out.println("创建: " + product); }
        public void deleteOrder(long id) { System.out.println("删除: " + id); }
    }

    public static void main(String[] args) {
        // 不同的回调
        MethodInterceptor logInterceptor = (obj, method, a, proxy) -> {
            System.out.println("[LOG] " + method.getName());
            return proxy.invokeSuper(obj, a);
        };

        MethodInterceptor txInterceptor = (obj, method, a, proxy) -> {
            System.out.println("[TX] begin " + method.getName());
            Object result = proxy.invokeSuper(obj, a);
            System.out.println("[TX] commit");
            return result;
        };

        // NoOp.INSTANCE：直接调用，不拦截
        Callback[] callbacks = {logInterceptor, txInterceptor, NoOp.INSTANCE};

        // CallbackFilter 根据方法决定使用哪个回调
        CallbackFilter filter = method -> {
            String name = method.getName();
            if (name.startsWith("query")) return 0;  // logInterceptor
            if (name.startsWith("create") || name.startsWith("delete")) return 1; // txInterceptor
            return 2; // NoOp
        };

        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(OrderService.class);
        enhancer.setCallbacks(callbacks);
        enhancer.setCallbackFilter(filter);

        OrderService proxy = (OrderService) enhancer.create();

        System.out.println("=== query（仅日志）===");
        proxy.queryOrder(1L);

        System.out.println("=== create（事务）===");
        proxy.createOrder("iPad");

        System.out.println("=== delete（事务）===");
        proxy.deleteOrder(1L);
    }
}
```

## 6.4 MethodProxy 深度解析

```java
import net.sf.cglib.proxy.*;
import java.lang.reflect.Method;

public class MethodProxyDemo {

    static class MathService {
        public int add(int a, int b) { return a + b; }
        public int subtract(int a, int b) { return a - b; }
    }

    public static void main(String[] args) throws Exception {
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(MathService.class);
        enhancer.setCallback((MethodInterceptor) (obj, method, methodArgs, proxy) -> {
            System.out.println("方法: " + method.getName());
            System.out.println("MethodProxy.getSuperName: " + proxy.getSuperName());

            // proxy.invokeSuper(obj, args) - 调用父类方法（推荐）
            // proxy.invoke(target, args)   - 调用目标对象方法（可能无限递归！）
            return proxy.invokeSuper(obj, methodArgs);
        });

        MathService proxy = (MathService) enhancer.create();
        System.out.println("add(3,5) = " + proxy.add(3, 5));

        // 直接通过 MethodProxy 调用（不需要代理对象）
        MethodProxy mp = MethodProxy.find(proxy.getClass(),
            proxy.getClass().getSuperclass().getDeclaredMethod("add", int.class, int.class));
        // 性能注意：invokeSuper 使用 FastClass，比反射快约 3-5 倍
    }
}
```

## 6.5 CGLIB 代理注意事项

```
常见问题与解决方案：

1. 无无参构造器
   问题：enhancer.create() 失败
   解决：enhancer.create(new Class[]{String.class}, new Object[]{"arg"})
         或引入 Objenesis 跳过构造器

2. final 类
   问题：Cannot subclass final class
   解决：改用 JDK 代理（如果有接口）或 ByteBuddy/Javassist

3. Spring 中的 CGLIB
   @Configuration 类必须用 CGLIB 代理（保证 @Bean 单例）
   proxyTargetClass=true 强制使用 CGLIB

4. 性能优化
   - 缓存 Enhancer 实例（创建 Enhancer 开销大）
   - 使用 MethodProxy.invokeSuper（基于 FastClass 跳过反射）
   - 设置 enhancer.setUseFactory(false)（减少内存开销）

5. 序列化
   CGLIB 代理类可序列化，但需要 target 也可序列化
   建议：代理类不要跨网络传输
```


---

# Part 7: JDK 代理 vs CGLIB 代理全面对比

## 7.1 对比总览

```
+----------------------+------------------+------------------+
|       维度           |   JDK 动态代理   |   CGLIB 代理     |
+----------------------+------------------+------------------+
| 代理方式             | 实现接口         | 继承目标类       |
| 是否需要接口         | 必须             | 不需要           |
| 代理 final 类        | 不支持           | 不支持           |
| 代理 final 方法      | 不支持           | 不支持           |
| 代理 static 方法     | 不支持           | 不支持           |
| 创建速度             | 较快             | 较慢（ASM生成）  |
| 调用速度（JDK8+）    | 相当             | 略快（FastClass）|
| 依赖                 | JDK 内置         | 需要 cglib 包    |
| Spring 默认策略      | 有接口时使用     | 无接口或强制时   |
| 内存占用             | 较小             | 较大             |
| 序列化支持           | 需接口可序列化   | 需类可序列化     |
+----------------------+------------------+------------------+
```

## 7.2 选择策略

```java
// Spring AOP 代理选择逻辑（简化版）
public class ProxyStrategy {
    public static Object createProxy(Object target, boolean forceTargetClass) {
        Class<?> targetClass = target.getClass();

        if (!forceTargetClass && targetClass.getInterfaces().length > 0) {
            // 目标类有接口，且未强制使用 CGLIB -> JDK 代理
            return java.lang.reflect.Proxy.newProxyInstance(
                targetClass.getClassLoader(),
                targetClass.getInterfaces(),
                (proxy, method, args) -> method.invoke(target, args)
            );
        } else {
            // 无接口或强制 CGLIB -> CGLIB 代理
            net.sf.cglib.proxy.Enhancer enhancer = new net.sf.cglib.proxy.Enhancer();
            enhancer.setSuperclass(targetClass);
            enhancer.setCallback((net.sf.cglib.proxy.MethodInterceptor)
                (obj, method, args, proxy) -> proxy.invokeSuper(obj, args));
            return enhancer.create();
        }
    }
}
```

## 7.3 性能基准测试

```java
import java.lang.reflect.*;
import net.sf.cglib.proxy.*;

public class ProxyBenchmark {

    interface Service { String call(); }

    static class ServiceImpl implements Service {
        public String call() { return "result"; }
    }

    public static void main(String[] args) throws Exception {
        ServiceImpl target = new ServiceImpl();
        int WARMUP = 100_000;
        int BENCHMARK = 1_000_000;

        // 直接调用
        for (int i = 0; i < WARMUP; i++) target.call();
        long s = System.nanoTime();
        for (int i = 0; i < BENCHMARK; i++) target.call();
        System.out.printf("直接调用:    %,d ns%n", System.nanoTime() - s);

        // JDK 代理
        Service jdkProxy = (Service) Proxy.newProxyInstance(
            ServiceImpl.class.getClassLoader(),
            new Class[]{Service.class},
            (p, m, a) -> m.invoke(target, a));
        for (int i = 0; i < WARMUP; i++) jdkProxy.call();
        s = System.nanoTime();
        for (int i = 0; i < BENCHMARK; i++) jdkProxy.call();
        System.out.printf("JDK 代理:    %,d ns%n", System.nanoTime() - s);

        // CGLIB 代理
        Enhancer e = new Enhancer();
        e.setSuperclass(ServiceImpl.class);
        e.setInterfaces(new Class[]{Service.class});
        e.setCallback((MethodInterceptor) (obj, m, a, proxy) -> proxy.invokeSuper(obj, a));
        Service cglibProxy = (Service) e.create();
        for (int i = 0; i < WARMUP; i++) cglibProxy.call();
        s = System.nanoTime();
        for (int i = 0; i < BENCHMARK; i++) cglibProxy.call();
        System.out.printf("CGLIB 代理:  %,d ns%n", System.nanoTime() - s);
    }
}
```

---

# Part 8: Javassist 字节码操作

## 8.1 Javassist 基础

```java
import javassist.*;

public class JavassistBasic {
    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();

        // 1. 创建新类
        CtClass cc = pool.makeClass("com.example.Hello");
        cc.setSuperclass(pool.get("java.lang.Object"));
        cc.setInterfaces(new CtClass[]{pool.get("java.io.Serializable")});

        // 添加字段
        CtField nameField = new CtField(pool.get("java.lang.String"), "name", cc);
        nameField.setModifiers(Modifier.PRIVATE);
        cc.addField(nameField);

        // 添加无参构造器
        CtConstructor noArgCtor = new CtConstructor(new CtClass[]{}, cc);
        noArgCtor.setBody("{ this.name = \"default\"; }");
        cc.addConstructor(noArgCtor);

        // 添加带参构造器
        CtConstructor argCtor = new CtConstructor(
            new CtClass[]{pool.get("java.lang.String")}, cc);
        argCtor.setBody("{ this.name = $1; }");
        cc.addConstructor(argCtor);

        // 添加方法
        CtMethod greet = new CtMethod(
            pool.get("java.lang.String"), "greet",
            new CtClass[]{}, cc);
        greet.setBody("{ return \"Hello, \" + name + \"!\"; }");
        cc.addMethod(greet);

        // 加载类并使用
        Class<?> clazz = cc.toClass();
        Object obj = clazz.getDeclaredConstructor(String.class).newInstance("World");
        System.out.println(
            clazz.getMethod("greet").invoke(obj) // Hello, World!
        );
    }
}
```

## 8.2 Javassist 修改已有类

```java
import javassist.*;

public class JavassistModify {
    public static class TargetClass {
        public String process(String input) {
            return input.toUpperCase();
        }
    }

    public static void main(String[] args) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        CtClass cc = pool.get("JavassistModify$TargetClass");

        // 修改已有方法（在方法前后插入代码）
        CtMethod method = cc.getDeclaredMethod("process");

        // 在方法开头插入
        method.insertBefore(
            "{ System.out.println(\"[Before] input=\" + $1); }"
        );

        // 在方法末尾插入
        method.insertAfter(
            "{ System.out.println(\"[After] result=\" + $_); }"
        );

        // 将方法体完全替换
        // method.setBody("{ return \"MODIFIED: \" + $1; }");

        // 写出修改后的字节码
        cc.writeFile("./output");
        System.out.println("字节码已写出到 ./output");

        // 也可直接加载
        // cc.toClass();
    }
}
```

## 8.3 Javassist 动态代理

```java
import javassist.*;
import java.lang.reflect.*;

public class JavassistProxy {

    interface Calculator { int add(int a, int b); }

    static class CalculatorImpl implements Calculator {
        public int add(int a, int b) { return a + b; }
    }

    // 使用 Javassist 创建代理类
    static Class<?> createProxyClass(Class<?> targetClass) throws Exception {
        ClassPool pool = ClassPool.getDefault();
        String proxyName = targetClass.getName() + "$$JavassistProxy";

        CtClass proxyCtClass = pool.makeClass(proxyName);
        proxyCtClass.setSuperclass(pool.get(targetClass.getName()));

        // 添加 InvocationHandler 字段
        CtField handlerField = new CtField(
            pool.get("java.lang.reflect.InvocationHandler"), "handler", proxyCtClass);
        handlerField.setModifiers(Modifier.PRIVATE);
        proxyCtClass.addField(handlerField);

        // 添加构造器
        CtConstructor ctor = new CtConstructor(
            new CtClass[]{pool.get("java.lang.reflect.InvocationHandler")}, proxyCtClass);
        ctor.setBody("{ this.handler = $1; }");
        proxyCtClass.addConstructor(ctor);

        // 覆盖方法
        for (Method m : targetClass.getDeclaredMethods()) {
            StringBuilder body = new StringBuilder("{ ");
            body.append("java.lang.reflect.Method m = ")
                .append(targetClass.getName())
                .append(".class.getDeclaredMethod(\"").append(m.getName()).append("\"");
            if (m.getParameterCount() > 0) {
                body.append(", new Class[]{");
                for (Class<?> pt : m.getParameterTypes()) {
                    body.append(pt.getName()).append(".class,");
                }
                body.append("}");
            } else {
                body.append(", new Class[0]");
            }
            body.append("); ");
            body.append("return ($r) handler.invoke(this, m, $args); }");

            CtMethod cm = CtNewMethod.copy(
                pool.get(targetClass.getName()).getDeclaredMethod(m.getName()),
                proxyCtClass, null);
            cm.setBody(body.toString());
            proxyCtClass.addMethod(cm);
        }
        return proxyCtClass.toClass();
    }

    public static void main(String[] args) throws Exception {
        Class<?> proxyClass = createProxyClass(CalculatorImpl.class);
        Calculator proxy = (Calculator) proxyClass.getConstructor(InvocationHandler.class)
            .newInstance((InvocationHandler) (p, m, a) -> {
                System.out.println("[Javassist Proxy] " + m.getName());
                return m.invoke(new CalculatorImpl(), a);
            });
        System.out.println("add(3,4) = " + proxy.add(3, 4));
    }
}
```


---

# Part 9: MethodHandle 与 VarHandle（Java 9+）

## 9.1 MethodHandle 完整指南

```java
import java.lang.invoke.*;

public class MethodHandleGuide {

    static class StringHelper {
        public static String repeat(String s, int n) {
            return s.repeat(n);
        }
        private String capitalize() {
            return Character.toUpperCase(charAt(0)) + substring(1);
        }
    }

    public static void main(String[] args) throws Throwable {
        MethodHandles.Lookup lookup = MethodHandles.lookup();

        // 1. findStatic - 静态方法
        MethodHandle mhStatic = lookup.findStatic(
            String.class, "valueOf",
            MethodType.methodType(String.class, int.class));
        System.out.println((String) mhStatic.invoke(42)); // "42"

        // 2. findVirtual - 实例方法
        MethodHandle mhVirtual = lookup.findVirtual(
            String.class, "substring",
            MethodType.methodType(String.class, int.class));
        System.out.println((String) mhVirtual.invoke("Hello World", 6)); // "World"

        // 3. findSpecial - super 方法调用
        // （需要在子类内部 lookup 中使用）

        // 4. findConstructor - 构造器
        MethodHandle mhCtor = lookup.findConstructor(
            StringBuilder.class,
            MethodType.methodType(void.class, String.class));
        StringBuilder sb = (StringBuilder) mhCtor.invoke("Hello");
        System.out.println(sb); // Hello

        // 5. findGetter / findSetter - 公有字段
        // （字段必须是 public 且 accessible）

        // 6. asType - 类型转换适配器
        MethodHandle mhConvert = mhVirtual.asType(
            MethodType.methodType(CharSequence.class, CharSequence.class, int.class));

        // 7. bindTo - 绑定参数（部分应用）
        MethodHandle mhBound = mhVirtual.bindTo("Hello World");
        System.out.println((String) mhBound.invoke(6)); // "World"

        // 8. asSpreader - 数组参数展开
        MethodHandle mhSpread = mhVirtual.asSpreader(Object[].class, 1);
        System.out.println((String) mhSpread.invoke("Hello World", new Object[]{6}));

        // 9. asCollector - 参数收集为数组
        // 10. filterArguments - 参数过滤
        MethodHandle toInt = lookup.findStatic(
            Integer.class, "parseInt",
            MethodType.methodType(int.class, String.class));
        MethodHandle add = lookup.findStatic(
            Integer.class, "sum",
            MethodType.methodType(int.class, int.class, int.class));
        MethodHandle addStrings = MethodHandles.filterArguments(add, 0, toInt, toInt);
        System.out.println((int) addStrings.invoke("10", "20")); // 30
    }
}
```

## 9.2 VarHandle（Java 9+）

```java
import java.lang.invoke.*;
import java.util.concurrent.atomic.*;

public class VarHandleDemo {

    static class Counter {
        volatile int count = 0;
        int[] data = new int[10];
    }

    public static void main(String[] args) throws Exception {
        // 1. 获取字段 VarHandle
        VarHandle countVH = MethodHandles.lookup().findVarHandle(
            Counter.class, "count", int.class);

        Counter counter = new Counter();

        // 普通读写
        countVH.set(counter, 5);
        System.out.println("get: " + countVH.get(counter)); // 5

        // CAS 操作（原子比较并交换）
        boolean success = countVH.compareAndSet(counter, 5, 10);
        System.out.println("CAS 5->10: " + success); // true
        System.out.println("after CAS: " + counter.count); // 10

        // 原子加法
        int old = (int) countVH.getAndAdd(counter, 3);
        System.out.println("getAndAdd(3): old=" + old + " new=" + counter.count); // 10, 13

        // 原子获取并设置
        int prev = (int) countVH.getAndSet(counter, 100);
        System.out.println("getAndSet(100): prev=" + prev); // 13

        // 内存序（Memory Order）
        countVH.setVolatile(counter, 50);          // volatile write
        int v1 = (int) countVH.getVolatile(counter); // volatile read

        countVH.setOpaque(counter, 60);             // opaque write（弱于 volatile）
        int v2 = (int) countVH.getOpaque(counter);  // opaque read

        countVH.setRelease(counter, 70);            // release write
        int v3 = (int) countVH.getAcquire(counter); // acquire read

        System.out.println("volatile=" + v1 + " opaque=" + v2 + " acquire=" + v3);

        // 2. 数组 VarHandle
        VarHandle arrayVH = MethodHandles.arrayElementVarHandle(int[].class);
        int[] arr = new int[10];
        arrayVH.set(arr, 3, 42);
        System.out.println("arr[3] = " + arrayVH.get(arr, 3)); // 42
        arrayVH.compareAndSet(arr, 3, 42, 99);
        System.out.println("arr[3] after CAS = " + arr[3]); // 99

        // VarHandle vs AtomicInteger 对比
        AtomicInteger ai = new AtomicInteger(0);
        // AtomicInteger 内部就是用 VarHandle 实现的（Java 9+）
        System.out.println("AtomicInteger.getAndIncrement: " + ai.getAndIncrement());
    }
}
```

## 9.3 invokedynamic 与 CallSite

```java
import java.lang.invoke.*;

public class InvokeDynamicDemo {
    // invokedynamic 是 JVM 指令集中最复杂的指令
    // Lambda 表达式、字符串拼接（Java 9+）都是通过 invokedynamic 实现的

    // Bootstrap 方法示例
    public static CallSite myBootstrap(
            MethodHandles.Lookup caller,
            String name,
            MethodType type) throws NoSuchMethodException, IllegalAccessException {

        System.out.println("Bootstrap called: name=" + name + " type=" + type);

        // 寻找并返回 CallSite
        MethodHandle target = caller.findStatic(
            InvokeDynamicDemo.class, name, type);
        return new ConstantCallSite(target);
    }

    public static String greet(String name) { return "Hello, " + name; }

    // Lambda 的字节码大致等价于：
    // invokedynamic #1 <run, BootstrapMethods #0>
    // 其中 BootstrapMethods 指向 LambdaMetafactory.metafactory

    public static void main(String[] args) {
        // 演示 Lambda 的 invokedynamic 本质
        Runnable r1 = () -> System.out.println("Lambda 1");
        Runnable r2 = () -> System.out.println("Lambda 2");

        // 每个 Lambda 在第一次调用时通过 invokedynamic 生成类
        // 后续调用复用同一 CallSite（ConstantCallSite）
        System.out.println("r1.class: " + r1.getClass().getName());
        System.out.println("r2.class: " + r2.getClass().getName());
        // 它们是不同的匿名类
        r1.run();
        r2.run();
    }
}
```

---

# Part 10: 注解处理器（APT）

## 10.1 自定义注解处理器

```java
// 编译时注解（Retention.SOURCE）
import java.lang.annotation.*;
import javax.annotation.processing.*;
import javax.lang.model.*;
import javax.lang.model.element.*;
import javax.tools.*;
import java.util.*;
import java.io.*;

@Retention(RetentionPolicy.SOURCE)
@Target(ElementType.TYPE)
@interface Builder {}

// 注解处理器
@SupportedAnnotationTypes("Builder")
@SupportedSourceVersion(SourceVersion.RELEASE_11)
public class BuilderProcessor extends AbstractProcessor {

    @Override
    public boolean process(Set<? extends TypeElement> annotations,
                           RoundEnvironment roundEnv) {
        for (Element element : roundEnv.getElementsAnnotatedWith(Builder.class)) {
            if (element.getKind() == ElementKind.CLASS) {
                TypeElement typeElement = (TypeElement) element;
                generateBuilder(typeElement);
            }
        }
        return true;
    }

    private void generateBuilder(TypeElement typeElement) {
        String className = typeElement.getSimpleName().toString();
        String packageName = processingEnv.getElementUtils()
            .getPackageOf(typeElement).getQualifiedName().toString();

        // 收集字段
        List<VariableElement> fields = new ArrayList<>();
        for (Element enclosed : typeElement.getEnclosedElements()) {
            if (enclosed.getKind() == ElementKind.FIELD) {
                fields.add((VariableElement) enclosed);
            }
        }

        // 生成 Builder 类源代码
        StringBuilder sb = new StringBuilder();
        sb.append("package ").append(packageName).append(";\n\n");
        sb.append("public class ").append(className).append("Builder {\n");

        // 字段
        for (VariableElement f : fields) {
            sb.append("    private ").append(f.asType()).append(" ")
              .append(f.getSimpleName()).append(";\n");
        }

        // setter 方法
        for (VariableElement f : fields) {
            String fname = f.getSimpleName().toString();
            String ftype = f.asType().toString();
            sb.append("\n    public ").append(className).append("Builder ")
              .append(fname).append("(").append(ftype).append(" ").append(fname).append(") {\n")
              .append("        this.").append(fname).append(" = ").append(fname).append(";\n")
              .append("        return this;\n")
              .append("    }\n");
        }

        // build 方法
        sb.append("\n    public ").append(className).append(" build() {\n");
        sb.append("        ").append(className).append(" obj = new ").append(className).append("();\n");
        for (VariableElement f : fields) {
            String fname = f.getSimpleName().toString();
            sb.append("        obj.").append(fname).append(" = this.").append(fname).append(";\n");
        }
        sb.append("        return obj;\n");
        sb.append("    }\n}\n");

        // 写入文件
        try {
            JavaFileObject file = processingEnv.getFiler()
                .createSourceFile(packageName + "." + className + "Builder");
            try (Writer writer = file.openWriter()) {
                writer.write(sb.toString());
            }
        } catch (IOException e) {
            processingEnv.getMessager().printMessage(
                Diagnostic.Kind.ERROR, "无法生成 Builder: " + e.getMessage());
        }
    }
}
```

## 10.2 APT 使用示例

```java
// 使用 @Builder 注解的类
@Builder
public class UserDTO {
    private String name;
    private int age;
    private String email;
    // getter/setter...
}

// 编译后自动生成 UserDTOBuilder.java
// 使用方式：
// UserDTO user = new UserDTOBuilder()
//     .name("Alice")
//     .age(25)
//     .email("alice@example.com")
//     .build();
```

## 10.3 运行时注解处理（vs APT）

```java
// 运行时注解处理（使用反射）
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
@interface Validate {
    boolean notNull() default false;
    int minLength() default 0;
    int maxLength() default Integer.MAX_VALUE;
    String pattern() default "";
}

public class ValidationProcessor {

    public static List<String> validate(Object obj) throws Exception {
        List<String> errors = new ArrayList<>();
        Class<?> clazz = obj.getClass();

        for (Field field : clazz.getDeclaredFields()) {
            Validate validate = field.getAnnotation(Validate.class);
            if (validate == null) continue;

            field.setAccessible(true);
            Object value = field.get(obj);
            String fieldName = field.getName();

            if (validate.notNull() && value == null) {
                errors.add(fieldName + ": 不能为 null");
                continue;
            }

            if (value instanceof String) {
                String str = (String) value;
                if (str.length() < validate.minLength()) {
                    errors.add(fieldName + ": 长度不能小于 " + validate.minLength());
                }
                if (str.length() > validate.maxLength()) {
                    errors.add(fieldName + ": 长度不能大于 " + validate.maxLength());
                }
                if (!validate.pattern().isEmpty() && !str.matches(validate.pattern())) {
                    errors.add(fieldName + ": 格式不符合规则 " + validate.pattern());
                }
            }
        }
        return errors;
    }

    static class UserForm {
        @Validate(notNull = true, minLength = 2, maxLength = 20)
        private String username;

        @Validate(notNull = true, pattern = "^[\\w.-]+@[\\w.-]+\\.\\w{2,}$")
        private String email;

        @Validate(minLength = 6)
        private String password;

        public UserForm(String username, String email, String password) {
            this.username = username;
            this.email = email;
            this.password = password;
        }
    }

    public static void main(String[] args) throws Exception {
        UserForm form1 = new UserForm("Alice", "alice@example.com", "secret123");
        System.out.println("valid form: " + validate(form1)); // []

        UserForm form2 = new UserForm("A", "not-an-email", "123");
        System.out.println("invalid form: " + validate(form2));
        // [username: 长度不能小于 2, email: 格式不符合规则..., password: 长度不能小于 6]
    }
}
```


---

# Part 11: 反射与框架底层应用

## 11.1 Spring IoC 反射原理

```
Spring Bean 创建流程（反射核心）:

BeanDefinition
    |
    v
BeanFactory.getBean(name)
    |
    v
Class.forName(className)    <- 反射加载类
    |
    v
Constructor.newInstance()   <- 反射实例化
    |
    v
Field/Method 注入依赖        <- 反射注入 @Autowired
    |
    v
BeanPostProcessor.postProcessAfterInitialization()
    |                        <- CGLIB/JDK 代理生成（@Transactional/@Async 等）
    v
完整 Bean 对象
```

```java
// 简化版 Spring @Autowired 注入实现
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;

@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.CONSTRUCTOR})
@interface Autowired {}

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@interface Component { String value() default ""; }

public class SimpleBeanFactory {
    private final Map<Class<?>, Object> beanMap = new HashMap<>();

    // 注册 Bean
    public void register(Class<?>... classes) throws Exception {
        for (Class<?> clazz : classes) {
            Object bean = clazz.getDeclaredConstructor().newInstance();
            beanMap.put(clazz, bean);
        }
        // 注入依赖
        for (Object bean : beanMap.values()) {
            injectFields(bean);
        }
    }

    private void injectFields(Object bean) throws Exception {
        for (Field field : bean.getClass().getDeclaredFields()) {
            if (field.isAnnotationPresent(Autowired.class)) {
                Object dep = beanMap.get(field.getType());
                if (dep == null) {
                    // 查找子类或实现类
                    for (Map.Entry<Class<?>, Object> e : beanMap.entrySet()) {
                        if (field.getType().isAssignableFrom(e.getKey())) {
                            dep = e.getValue();
                            break;
                        }
                    }
                }
                if (dep != null) {
                    field.setAccessible(true);
                    field.set(bean, dep);
                }
            }
        }
    }

    @SuppressWarnings("unchecked")
    public <T> T getBean(Class<T> clazz) {
        return (T) beanMap.get(clazz);
    }

    // 示例
    @Component static class Repository {
        public String find(long id) { return "User#" + id; }
    }

    @Component static class Service {
        @Autowired Repository repository;
        public String getUser(long id) { return repository.find(id); }
    }

    public static void main(String[] args) throws Exception {
        SimpleBeanFactory factory = new SimpleBeanFactory();
        factory.register(Repository.class, Service.class);

        Service service = factory.getBean(Service.class);
        System.out.println(service.getUser(1L)); // User#1
    }
}
```

## 11.2 Spring AOP 代理原理

```java
import java.lang.reflect.*;
import java.lang.annotation.*;

// 模拟 @Transactional 的 AOP 切面
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Transactional {}

public class SpringAopSimulator {

    // 通知接口
    @FunctionalInterface
    interface Around {
        Object proceed(Method method, Object target, Object[] args) throws Throwable;
    }

    // AOP 代理工厂
    @SuppressWarnings("unchecked")
    static <T> T createAopProxy(T target, Around advice) {
        Class<?> clazz = target.getClass();
        Class<?>[] interfaces = clazz.getInterfaces();

        if (interfaces.length > 0) {
            // JDK 代理
            return (T) Proxy.newProxyInstance(
                clazz.getClassLoader(), interfaces,
                (proxy, method, args) -> {
                    if (method.isAnnotationPresent(Transactional.class)) {
                        return advice.proceed(method, target, args);
                    }
                    return method.invoke(target, args);
                });
        } else {
            // 实际 Spring 这里会用 CGLIB
            throw new IllegalArgumentException("需要接口（示例简化）");
        }
    }

    interface AccountService {
        @Transactional
        void transfer(String from, String to, double amount);
        double getBalance(String account);
    }

    static class AccountServiceImpl implements AccountService {
        Map<String, Double> accounts = new HashMap<>(Map.of("Alice", 1000.0, "Bob", 500.0));

        public void transfer(String from, String to, double amount) {
            accounts.merge(from, -amount, Double::sum);
            accounts.merge(to, amount, Double::sum);
            System.out.println("转账完成: " + from + " -> " + to + " ¥" + amount);
        }
        public double getBalance(String account) { return accounts.getOrDefault(account, 0.0); }
    }

    public static void main(String[] args) {
        AccountService real = new AccountServiceImpl();

        // 事务通知
        AccountService proxy = createAopProxy(real, (method, target, methodArgs) -> {
            System.out.println("[TX] begin transaction");
            try {
                Object result = method.invoke(target, methodArgs);
                System.out.println("[TX] commit");
                return result;
            } catch (Exception e) {
                System.out.println("[TX] rollback: " + e.getMessage());
                throw e;
            }
        });

        proxy.transfer("Alice", "Bob", 200.0);
        System.out.println("Alice: " + proxy.getBalance("Alice")); // 800
        System.out.println("Bob: " + proxy.getBalance("Bob")); // 700
    }
}
```

## 11.3 MyBatis Mapper 代理原理

```java
import java.lang.reflect.*;
import java.lang.annotation.*;

// 模拟 MyBatis 的 Mapper 代理机制
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Select { String value(); }

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Insert { String value(); }

public class MyBatisMapperProxy {

    // Mapper 接口
    interface UserMapper {
        @Select("SELECT * FROM user WHERE id = #{id}")
        String findById(long id);

        @Insert("INSERT INTO user(name) VALUES(#{name})")
        int insert(String name);
    }

    // 模拟 SqlSession
    static class MockSqlSession {
        String selectOne(String sql, Object param) {
            return "User[sql=" + sql + ", param=" + param + "]";
        }
        int insert(String sql, Object param) {
            System.out.println("执行: " + sql + " param=" + param);
            return 1;
        }
    }

    // Mapper 代理处理器
    static class MapperProxyHandler implements InvocationHandler {
        private final MockSqlSession sqlSession;

        MapperProxyHandler(MockSqlSession session) { this.sqlSession = session; }

        @Override
        public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
            // 忽略 Object 的方法
            if (method.getDeclaringClass() == Object.class) {
                return method.invoke(this, args);
            }

            Select select = method.getAnnotation(Select.class);
            Insert insert = method.getAnnotation(Insert.class);

            if (select != null) {
                Object param = args != null && args.length > 0 ? args[0] : null;
                return sqlSession.selectOne(select.value(), param);
            }
            if (insert != null) {
                Object param = args != null && args.length > 0 ? args[0] : null;
                return sqlSession.insert(insert.value(), param);
            }
            throw new UnsupportedOperationException("方法 " + method.getName() + " 未映射 SQL");
        }
    }

    @SuppressWarnings("unchecked")
    static <T> T getMapper(Class<T> mapperClass, MockSqlSession session) {
        return (T) Proxy.newProxyInstance(
            mapperClass.getClassLoader(),
            new Class[]{mapperClass},
            new MapperProxyHandler(session)
        );
    }

    public static void main(String[] args) {
        MockSqlSession session = new MockSqlSession();
        UserMapper mapper = getMapper(UserMapper.class, session);

        String user = mapper.findById(1L);
        System.out.println("findById: " + user);

        int rows = mapper.insert("Alice");
        System.out.println("insert rows: " + rows);
    }
}
```

## 11.4 Jackson 反射序列化原理

```java
import java.lang.reflect.*;
import java.util.*;

// 简化版 JSON 序列化（类似 Jackson 原理）
public class SimpleJsonSerializer {

    public static String toJson(Object obj) throws Exception {
        if (obj == null) return "null";
        if (obj instanceof String) return "\"" + obj + "\"";
        if (obj instanceof Number || obj instanceof Boolean) return obj.toString();
        if (obj instanceof Collection) {
            StringBuilder sb = new StringBuilder("[");
            for (Object item : (Collection<?>) obj) {
                sb.append(toJson(item)).append(",");
            }
            if (sb.length() > 1) sb.deleteCharAt(sb.length() - 1);
            return sb.append("]").toString();
        }
        if (obj instanceof Map) {
            StringBuilder sb = new StringBuilder("{");
            for (Map.Entry<?, ?> entry : ((Map<?, ?>) obj).entrySet()) {
                sb.append("\"").append(entry.getKey()).append("\":")
                  .append(toJson(entry.getValue())).append(",");
            }
            if (sb.length() > 1) sb.deleteCharAt(sb.length() - 1);
            return sb.append("}").toString();
        }

        // POJO 对象：通过反射获取所有字段
        Class<?> clazz = obj.getClass();
        StringBuilder sb = new StringBuilder("{");
        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            sb.append("\"").append(field.getName()).append("\":")
              .append(toJson(field.get(obj))).append(",");
        }
        if (sb.length() > 1) sb.deleteCharAt(sb.length() - 1);
        return sb.append("}").toString();
    }

    static class Address {
        String city;
        String zip;
        Address(String city, String zip) { this.city = city; this.zip = zip; }
    }

    static class Person {
        String name;
        int age;
        Address address;
        List<String> hobbies;

        Person(String name, int age, Address address, List<String> hobbies) {
            this.name = name; this.age = age;
            this.address = address; this.hobbies = hobbies;
        }
    }

    public static void main(String[] args) throws Exception {
        Person p = new Person("Alice", 25,
            new Address("Beijing", "100000"),
            Arrays.asList("coding", "reading"));
        System.out.println(toJson(p));
    }
}
```


---

# Part 12: 完整实战案例

## 12.1 迷你 BeanUtils（属性复制）

```java
import java.lang.reflect.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 高性能属性复制工具（类似 BeanUtils.copyProperties）
 * 支持：同名字段复制、类型转换、忽略字段
 */
public class MiniBeanUtils {

    // 缓存：源类 -> (字段名 -> Field)
    private static final Map<Class<?>, Map<String, Field>> FIELD_CACHE = new ConcurrentHashMap<>();

    private static Map<String, Field> getFieldMap(Class<?> clazz) {
        return FIELD_CACHE.computeIfAbsent(clazz, c -> {
            Map<String, Field> map = new LinkedHashMap<>();
            Class<?> current = c;
            while (current != null && current != Object.class) {
                for (Field f : current.getDeclaredFields()) {
                    if (!Modifier.isStatic(f.getModifiers())) {
                        f.setAccessible(true);
                        map.putIfAbsent(f.getName(), f);
                    }
                }
                current = current.getSuperclass();
            }
            return Collections.unmodifiableMap(map);
        });
    }

    /**
     * 复制属性（同名、兼容类型）
     */
    public static void copyProperties(Object source, Object target, String... ignoreFields) {
        Set<String> ignoreSet = new HashSet<>(Arrays.asList(ignoreFields));
        Map<String, Field> sourceFields = getFieldMap(source.getClass());
        Map<String, Field> targetFields = getFieldMap(target.getClass());

        for (Map.Entry<String, Field> entry : sourceFields.entrySet()) {
            String name = entry.getKey();
            if (ignoreSet.contains(name)) continue;

            Field targetField = targetFields.get(name);
            if (targetField == null) continue;

            try {
                Object value = entry.getValue().get(source);
                if (value == null) continue;

                // 简单类型兼容检查
                Class<?> targetType = targetField.getType();
                Class<?> valueType = value.getClass();

                if (targetType.isAssignableFrom(valueType)) {
                    targetField.set(target, value);
                } else {
                    // 基本类型转换
                    Object converted = convertBasic(value, targetType);
                    if (converted != null) targetField.set(target, converted);
                }
            } catch (IllegalAccessException e) {
                // skip
            }
        }
    }

    private static Object convertBasic(Object value, Class<?> targetType) {
        String str = value.toString();
        if (targetType == String.class) return str;
        if (targetType == int.class || targetType == Integer.class) return Integer.parseInt(str);
        if (targetType == long.class || targetType == Long.class) return Long.parseLong(str);
        if (targetType == double.class || targetType == Double.class) return Double.parseDouble(str);
        if (targetType == boolean.class || targetType == Boolean.class) return Boolean.parseBoolean(str);
        return null;
    }

    // 深拷贝（简化版）
    @SuppressWarnings("unchecked")
    public static <T> T deepCopy(T source) throws Exception {
        Class<?> clazz = source.getClass();
        T target = (T) clazz.getDeclaredConstructor().newInstance();
        copyProperties(source, target);
        return target;
    }

    // 示例
    static class UserDO {
        Long id;
        String name;
        int age;
        String password; // 不应复制
        UserDO() {}
        UserDO(Long id, String name, int age, String password) {
            this.id = id; this.name = name; this.age = age; this.password = password;
        }
        @Override public String toString() {
            return "UserDO{id=" + id + ",name=" + name + ",age=" + age + "}";
        }
    }

    static class UserDTO {
        Long id;
        String name;
        String age; // 类型不同（int -> String）
        UserDTO() {}
        @Override public String toString() {
            return "UserDTO{id=" + id + ",name=" + name + ",age=" + age + "}";
        }
    }

    public static void main(String[] args) throws Exception {
        UserDO userDO = new UserDO(1L, "Alice", 25, "secret123");
        UserDTO userDTO = new UserDTO();

        copyProperties(userDO, userDTO, "password");
        System.out.println(userDTO); // UserDTO{id=1,name=Alice,age=25}
    }
}
```

## 12.2 迷你 ORM 框架

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.sql.*;
import java.util.*;

// 注解定义
@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.TYPE)
@interface Table { String name(); }

@Retention(RetentionPolicy.RUNTIME) @Target(ElementType.FIELD)
@interface Column { String name() default ""; boolean primaryKey() default false; }

// 迷你 ORM
public class MiniOrm {

    private Connection conn;

    public MiniOrm(Connection conn) { this.conn = conn; }

    // 查询单条记录
    @SuppressWarnings("unchecked")
    public <T> T findById(Class<T> clazz, Object id) throws Exception {
        Table table = clazz.getAnnotation(Table.class);
        if (table == null) throw new IllegalArgumentException("缺少 @Table 注解");

        Field pkField = findPrimaryKeyField(clazz);
        if (pkField == null) throw new IllegalArgumentException("缺少 @Column(primaryKey=true)");

        Column pkCol = pkField.getAnnotation(Column.class);
        String colName = pkCol.name().isEmpty() ? pkField.getName() : pkCol.name();

        String sql = "SELECT * FROM " + table.name() + " WHERE " + colName + " = ?";
        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setObject(1, id);
            ResultSet rs = ps.executeQuery();
            if (rs.next()) return mapRow(clazz, rs);
        }
        return null;
    }

    // 插入记录
    public <T> int insert(T obj) throws Exception {
        Class<?> clazz = obj.getClass();
        Table table = clazz.getAnnotation(Table.class);

        List<Field> fields = new ArrayList<>();
        List<String> colNames = new ArrayList<>();
        for (Field f : clazz.getDeclaredFields()) {
            Column col = f.getAnnotation(Column.class);
            if (col != null && !col.primaryKey()) {
                f.setAccessible(true);
                fields.add(f);
                colNames.add(col.name().isEmpty() ? f.getName() : col.name());
            }
        }

        String sql = "INSERT INTO " + table.name()
            + "(" + String.join(",", colNames) + ") VALUES("
            + String.join(",", Collections.nCopies(colNames.size(), "?")) + ")";

        try (PreparedStatement ps = conn.prepareStatement(sql)) {
            for (int i = 0; i < fields.size(); i++) {
                ps.setObject(i + 1, fields.get(i).get(obj));
            }
            return ps.executeUpdate();
        }
    }

    @SuppressWarnings("unchecked")
    private <T> T mapRow(Class<T> clazz, ResultSet rs) throws Exception {
        T obj = clazz.getDeclaredConstructor().newInstance();
        ResultSetMetaData meta = rs.getMetaData();

        // 建立列名 -> Field 映射
        Map<String, Field> colToField = new HashMap<>();
        for (Field f : clazz.getDeclaredFields()) {
            Column col = f.getAnnotation(Column.class);
            if (col != null) {
                String colName = col.name().isEmpty() ? f.getName() : col.name();
                f.setAccessible(true);
                colToField.put(colName.toLowerCase(), f);
            }
        }

        for (int i = 1; i <= meta.getColumnCount(); i++) {
            String colName = meta.getColumnName(i).toLowerCase();
            Field field = colToField.get(colName);
            if (field != null) {
                field.set(obj, rs.getObject(i));
            }
        }
        return obj;
    }

    private Field findPrimaryKeyField(Class<?> clazz) {
        for (Field f : clazz.getDeclaredFields()) {
            Column col = f.getAnnotation(Column.class);
            if (col != null && col.primaryKey()) return f;
        }
        return null;
    }

    // 示例 Entity
    @Table(name = "users")
    static class User {
        @Column(name = "id", primaryKey = true)
        private Long id;
        @Column(name = "name")
        private String name;
        @Column(name = "age")
        private Integer age;
        User() {}
        @Override public String toString() {
            return "User{id=" + id + ",name=" + name + ",age=" + age + "}";
        }
    }
}
```

## 12.3 迷你事件总线（EventBus）

```java
import java.lang.annotation.*;
import java.lang.reflect.*;
import java.util.*;
import java.util.concurrent.*;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
@interface Subscribe {
    boolean async() default false;
}

public class MiniEventBus {
    // 事件类型 -> 订阅者列表
    private final Map<Class<?>, List<SubscriberMethod>> subscribers = new ConcurrentHashMap<>();
    private final ExecutorService executor = Executors.newCachedThreadPool();

    // 注册订阅者
    public void register(Object subscriber) {
        Class<?> clazz = subscriber.getClass();
        for (Method method : clazz.getDeclaredMethods()) {
            if (!method.isAnnotationPresent(Subscribe.class)) continue;
            if (method.getParameterCount() != 1) continue;

            Class<?> eventType = method.getParameterTypes()[0];
            method.setAccessible(true);
            subscribers.computeIfAbsent(eventType, k -> new CopyOnWriteArrayList<>())
                       .add(new SubscriberMethod(subscriber, method,
                            method.getAnnotation(Subscribe.class).async()));
        }
    }

    // 发布事件
    public void post(Object event) {
        List<SubscriberMethod> methods = subscribers.get(event.getClass());
        if (methods == null) return;

        for (SubscriberMethod sm : methods) {
            if (sm.async) {
                executor.submit(() -> invokeSubscriber(sm, event));
            } else {
                invokeSubscriber(sm, event);
            }
        }
    }

    private void invokeSubscriber(SubscriberMethod sm, Object event) {
        try {
            sm.method.invoke(sm.subscriber, event);
        } catch (Exception e) {
            System.err.println("事件处理异常: " + e.getMessage());
        }
    }

    static class SubscriberMethod {
        Object subscriber;
        Method method;
        boolean async;
        SubscriberMethod(Object s, Method m, boolean a) {
            subscriber = s; method = m; async = a;
        }
    }

    // 示例
    static class OrderCreatedEvent {
        String orderId;
        OrderCreatedEvent(String id) { this.orderId = id; }
    }

    static class EmailService {
        @Subscribe
        public void onOrderCreated(OrderCreatedEvent e) {
            System.out.println("[Email] 发送订单确认邮件: " + e.orderId);
        }
    }

    static class SmsService {
        @Subscribe(async = true)
        public void onOrderCreated(OrderCreatedEvent e) {
            System.out.println("[SMS] 异步发送短信通知: " + e.orderId);
        }
    }

    public static void main(String[] args) throws Exception {
        MiniEventBus bus = new MiniEventBus();
        bus.register(new EmailService());
        bus.register(new SmsService());

        bus.post(new OrderCreatedEvent("ORDER-001"));
        Thread.sleep(500); // 等待异步完成
        bus.executor.shutdown();
    }
}
```


---

# Part 13: 常见面试题 FAQ（20+）

## Q1：反射是什么？有哪些使用场景？

```
反射（Reflection）是 Java 在运行时检查、修改自身行为的能力。

核心 API：
  Class<T>      - 类的运行时表示
  Constructor<T> - 构造器
  Field         - 字段
  Method        - 方法
  Annotation    - 注解

使用场景：
1. 框架核心（Spring IoC/AOP、MyBatis、Hibernate）
2. 序列化/反序列化（Jackson、Gson、Fastjson）
3. 测试框架（JUnit、TestNG、Mockito）
4. 代码生成工具（Lombok、MapStruct）
5. 远程调用（Dubbo 的动态代理）
6. ORM 框架（Hibernate 实体映射）
7. 插件体系（Eclipse、IDEA 插件）
```

## Q2：Class.forName 与 ClassLoader.loadClass 的区别？

```java
// Class.forName：加载类并执行静态初始化块（static {}）
Class<?> c1 = Class.forName("com.example.MyClass");
// 等价于：
// Class.forName("com.example.MyClass", true, Thread.currentThread().getContextClassLoader())

// ClassLoader.loadClass：只加载字节码，不执行静态初始化
ClassLoader cl = Thread.currentThread().getContextClassLoader();
Class<?> c2 = cl.loadClass("com.example.MyClass");
// 静态块不会被执行！

// 区别总结：
// ┌─────────────────────────┬───────────────┬──────────────────┐
// │ 方法                    │ 链接（Link）  │ 初始化（Init）   │
// ├─────────────────────────┼───────────────┼──────────────────┤
// │ Class.forName           │ 是            │ 是（默认）        │
// │ ClassLoader.loadClass   │ 否            │ 否               │
// └─────────────────────────┴───────────────┴──────────────────┘

// 实际应用：JDBC 驱动加载必须用 Class.forName，因为需要触发静态注册
// Class.forName("com.mysql.cj.jdbc.Driver"); // 触发 DriverManager.registerDriver()
```

## Q3：反射为什么慢？如何优化？

```
原因：
1. 权限检查（SecurityManager + 访问控制）每次调用都检查
2. 参数装箱/拆箱（int -> Integer -> int）
3. 类型检查（无法 JIT 内联）
4. JNI 调用（inflation 前走 native 方法）
5. 无法享受 JIT 内联优化

优化策略（从易到难）：
1. setAccessible(true) - 跳过权限检查（最简单，效果明显）
2. 缓存 Method/Field 对象 - 避免重复 getDeclaredMethod
3. MethodHandle - 比反射快，可被 JIT 内联
4. LambdaMetafactory - 将方法调用编译为函数接口，接近直接调用
5. 代码生成（ByteBuddy/Javassist/APT） - 编译期生成代理代码

性能参考（百万次调用）：
  直接调用：1ms
  LambdaMetafactory：3ms
  MethodHandle.invokeExact：5ms
  Method.invoke（缓存+accessible）：15ms
  Method.invoke（无优化）：80ms
```

## Q4：JDK 动态代理与 CGLIB 代理的区别？

```
JDK 动态代理：
  - 只能代理有接口的类
  - 代理类继承 java.lang.reflect.Proxy
  - 通过 InvocationHandler 拦截
  - JDK 内置，无需额外依赖
  - Spring 目标类有接口时默认使用

CGLIB 代理：
  - 可代理没有接口的类
  - 通过字节码生成目标类的子类
  - 通过 MethodInterceptor 拦截
  - 无法代理 final 类和方法
  - Spring 从 5.x 开始默认使用 CGLIB（@SpringBootApplication 类默认用 CGLIB）

代码区别：
  JDK:   Proxy.newProxyInstance(loader, interfaces, handler)
  CGLIB: new Enhancer().setSuperclass(clazz).setCallback(interceptor).create()
```

## Q5：什么是类型擦除？如何获取泛型实际类型？

```java
// 类型擦除：编译后泛型信息从字节码中移除
List<String> ls = new ArrayList<>();
List<Integer> li = new ArrayList<>();
System.out.println(ls.getClass() == li.getClass()); // true（都是 ArrayList.class）

// 如何获取泛型类型（保留在 Signature 字节码属性中）：

// 1. 字段的泛型类型
class Container { List<String> items; }
Field f = Container.class.getDeclaredField("items");
ParameterizedType pt = (ParameterizedType) f.getGenericType();
System.out.println(pt.getActualTypeArguments()[0]); // class java.lang.String

// 2. 方法返回值/参数的泛型类型
// method.getGenericReturnType(), method.getGenericParameterTypes()

// 3. 父类的泛型类型（TypeToken 模式）
class StringList extends ArrayList<String> {}
Type superType = StringList.class.getGenericSuperclass();
// java.util.ArrayList<java.lang.String>

// 4. TypeToken 模式（保留匿名子类的类型参数）
new TypeToken<Map<String, List<Integer>>>() {}.getType();
```

## Q6：InvocationHandler 的 invoke 方法中，proxy 参数有什么用？

```java
// proxy 是代理对象本身（$Proxy0 实例）
// 常见用途：

// 1. 在代理链中将 proxy 传递给其他方法
// 2. 实现接口中的默认方法
// 3. 处理 equals/hashCode/toString

public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    // 处理 Object 基础方法
    if (method.getName().equals("equals")) {
        return proxy == args[0]; // 代理对象的 equals
    }
    if (method.getName().equals("hashCode")) {
        return System.identityHashCode(proxy);
    }
    if (method.getName().equals("toString")) {
        return "Proxy[" + target.getClass().getSimpleName() + "]";
    }

    // 注意：不要用 method.invoke(proxy, args)！
    // 这会无限递归：proxy.method() -> invoke() -> proxy.method() -> ...
    // 应该用 method.invoke(target, args) 调用真实对象
    return method.invoke(target, args);
}
```

## Q7：如何防止反射破坏单例？

```java
public class SafeSingleton {
    private static final SafeSingleton INSTANCE = new SafeSingleton();
    private static boolean instantiated = false;

    private SafeSingleton() {
        // 防止反射多次实例化
        synchronized (SafeSingleton.class) {
            if (instantiated) {
                throw new RuntimeException("禁止通过反射创建实例！");
            }
            instantiated = true;
        }
    }

    public static SafeSingleton getInstance() { return INSTANCE; }

    // 最佳实践：枚举单例（天然防反射）
    enum EnumSingleton {
        INSTANCE;
        public void doSomething() { System.out.println("枚举单例"); }
    }
    // EnumSingleton.INSTANCE.doSomething()
    // 反射无法 newInstance 枚举类，会抛 IllegalArgumentException
}
```

## Q8：getDeclaredMethods 与 getMethods 的区别？

```
getMethods()：
  - 返回所有 public 方法（包括继承自父类和接口的）
  - 不含 private/protected/package-private 方法
  - 包含 Object 的 public 方法（equals/hashCode/toString 等）

getDeclaredMethods()：
  - 返回本类声明的所有方法（不含继承）
  - 包含 private/protected/package-private 方法
  - 不含父类方法
  - 不含接口 default 方法（除非本类 override）

getMethod(name, paramTypes)：
  - 只获取一个 public 方法（含继承）
  - 找不到抛 NoSuchMethodException

getDeclaredMethod(name, paramTypes)：
  - 只获取本类声明的一个方法（含 private）
  - 找不到抛 NoSuchMethodException
```

## Q9：MethodProxy.invokeSuper 与 MethodProxy.invoke 的区别？

```java
// CGLIB MethodInterceptor 中：

public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)
        throws Throwable {
    // invokeSuper：调用父类（被代理类）的原始方法
    // obj 是代理类实例，调用的是父类中的方法实现
    // 推荐使用：避免无限递归，性能更好（基于 FastClass）
    Object result = proxy.invokeSuper(obj, args);

    // invoke：调用目标对象的方法
    // 如果 target == obj（即代理类实例），会无限递归！
    // 正确用法：传入真实目标对象（非代理）
    Object target = getRealTarget(); // 需要保存真实对象引用
    Object result2 = proxy.invoke(target, args);

    return result;
}
```

## Q10：Spring 中什么时候用 JDK 代理，什么时候用 CGLIB？

```
Spring AOP 代理选择策略：

1. @EnableAspectJAutoProxy(proxyTargetClass = false)（默认）
   - 目标类实现了接口 -> JDK 代理
   - 目标类没有接口 -> CGLIB 代理

2. @EnableAspectJAutoProxy(proxyTargetClass = true)
   - 强制所有情况都用 CGLIB

3. Spring Boot 2.x+ 默认
   - spring.aop.proxy-target-class=true（默认 CGLIB）
   - 避免了接口代理的注入问题

4. @Configuration 类
   - 总是使用 CGLIB 代理（保证 @Bean 方法返回同一个实例）

特殊情况：
  - 如果目标类是 final -> 都无法代理，抛异常
  - 如果目标方法是 final -> CGLIB 无法拦截该方法（JDK 也不会拦截 final 方法）
```

## Q11：什么是 FastClass？为什么 CGLIB 比反射快？

```
FastClass 是 CGLIB 在运行时生成的辅助类，它为目标类的每个方法分配一个索引。

传统反射调用链：
  Method.invoke(obj, args) -> JNI -> native 方法 -> Java 方法

FastClass 调用链：
  MethodProxy.invokeSuper(obj, args)
    -> FastClass.invoke(methodIndex, obj, args)
    -> switch(methodIndex) { case 0: return obj.method0(args); ... }

FastClass 消除了 JNI 调用和参数包装开销，
相当于把"反射查找+调用"转化为"switch case 直接调用"。
性能接近直接方法调用（比反射快 3-5 倍）。
```

## Q12：如何实现一个简单的 AOP 框架？

```java
// 核心三要素：切点（Pointcut）、通知（Advice）、织入（Weaving）
import java.lang.reflect.*;

public class MiniAop {

    // 通知接口
    interface BeforeAdvice { void before(Method m, Object[] args); }
    interface AfterAdvice { void after(Method m, Object[] args, Object result); }

    // 代理工厂
    static Object createProxy(Object target,
                               BeforeAdvice before, AfterAdvice after) {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(),
            (proxy, method, args) -> {
                if (before != null) before.before(method, args);
                Object result = method.invoke(target, args);
                if (after != null) after.after(method, args, result);
                return result;
            });
    }

    interface Greeter { String greet(String name); }
    static class GreeterImpl implements Greeter {
        public String greet(String name) { return "Hello, " + name; }
    }

    public static void main(String[] args) {
        Greeter proxy = (Greeter) createProxy(
            new GreeterImpl(),
            (m, a) -> System.out.println("[Before] " + m.getName()),
            (m, a, r) -> System.out.println("[After] result=" + r)
        );
        proxy.greet("World");
        // [Before] greet
        // [After] result=Hello, World
    }
}
```

## Q13-Q20：高频面试知识点速记

```
Q13: 反射可以访问 private 字段吗？
A: 可以。通过 field.setAccessible(true) 绕过访问控制。
   Java 9+ 模块系统对此有限制（需要模块开放 opens 声明）。

Q14: 反射创建对象有哪些方式？
A: 1. clazz.newInstance()（已废弃，不能调用 private 构造器，会包装异常）
   2. clazz.getDeclaredConstructor().newInstance()（推荐）
   3. Constructor.newInstance(args)
   4. Array.newInstance(componentType, length)（数组）
   5. Unsafe.allocateInstance(clazz)（跳过构造器，慎用）

Q15: 什么是 bridge 方法？
A: 为保证泛型擦除后的类型兼容性，编译器自动生成的合成方法。
   例如：class StringList extends ArrayList<String> 中，
   编译器会生成 add(Object) -> add(String) 的桥接方法。
   method.isBridge() 可检测。

Q16: 动态代理可以代理 Object 的方法吗？
A: equals/hashCode/toString 会被代理（InvocationHandler 接收到）。
   其他 Object 方法（如 wait/notify）不会走代理。
   实践中应在 invoke 中特殊处理这三个方法。

Q17: 什么是 Proxy$0 里的 m0/m1/m2/m3？
A: JDK 生成的代理类中，每个方法对应一个静态 Method 字段：
   m0 = hashCode, m1 = equals, m2 = toString, m3+ = 接口方法
   这些字段在 static {} 块中初始化（一次反射，后续复用）。

Q18: CGLIB 的 Enhancer 可以缓存吗？
A: 可以且应该缓存。Enhancer 生成字节码开销大，
   同一配置的 Enhancer 应只创建一次。
   Spring 内部缓存了所有已生成的代理类（ProxyFactory 维护缓存）。

Q19: Java 9+ 模块系统对反射的影响？
A: 模块默认不 opens，反射访问未开放的模块内字段/方法会抛 InaccessibleObjectException。
   解决方案：
   1. --add-opens java.base/java.lang=ALL-UNNAMED（命令行）
   2. module-info.java 中添加 opens 声明
   3. 框架使用 MethodHandles.privateLookupIn（需要目标模块配合）

Q20: 反射与代码生成的对比？
A: 反射（运行时）：
     - 灵活，可处理任意类
     - 有性能开销
     - 易于实现
   代码生成（编译时/运行时）：
     - APT（编译时）：零运行时开销，但需要注解处理器
     - ByteBuddy/Javassist（运行时字节码生成）：性能接近直接调用
     - LambdaMetafactory（运行时）：性能极好，适合固定签名方法
```

---

# 总结

```
+-----------------------------------------------------------+
|              Java 反射与动态代理 知识体系总结              |
+-----------------------------------------------------------+
|                                                           |
|  反射基础                    动态代理                     |
|  ├── Class 对象（4种获取）   ├── JDK 代理（接口）         |
|  ├── Constructor 反射        ├── CGLIB 代理（子类）        |
|  ├── Field 反射              ├── 代理链与拦截器            |
|  ├── Method 反射             └── 字节码操作（Javassist）   |
|  └── Annotation 反射                                      |
|                                                           |
|  性能优化                    高级特性                     |
|  ├── setAccessible           ├── VarHandle（Java 9+）     |
|  ├── 缓存 Method/Field       ├── invokedynamic            |
|  ├── MethodHandle            ├── APT 编译时处理           |
|  └── LambdaMetafactory       └── 泛型与 TypeToken         |
|                                                           |
|  框架应用                    实战案例                     |
|  ├── Spring IoC/AOP          ├── 迷你 BeanUtils           |
|  ├── MyBatis Mapper 代理     ├── 迷你 ORM                 |
|  ├── Jackson 序列化          └── EventBus                 |
|  └── JUnit5 测试框架                                      |
+-----------------------------------------------------------+
```

> 本文档涵盖了 Java 反射与动态代理的核心知识点，结合大量可运行的示例代码，
> 适合面试备考与框架开发参考。建议结合 JDK 源码（java.lang.reflect 包）深入学习。

---

# 补充：反射常用工具方法速查

## A. 判断类型关系

```java
// 判断类是否可赋值（isAssignableFrom）
System.out.println(Number.class.isAssignableFrom(Integer.class)); // true
System.out.println(Integer.class.isAssignableFrom(Number.class)); // false

// 判断对象是否为某类实例（instanceof 的反射版本）
Object obj = Integer.valueOf(42);
System.out.println(Number.class.isInstance(obj));  // true
System.out.println(String.class.isInstance(obj));  // false

// 获取类的全部层次
Class<?> clazz = ArrayList.class;
List<Class<?>> hierarchy = new ArrayList<>();
Class<?> current = clazz;
while (current != null) {
    hierarchy.add(current);
    current = current.getSuperclass();
}
// hierarchy: [ArrayList, AbstractList, AbstractCollection, Object]

// 获取类实现的所有接口（包括父类的接口）
Set<Class<?>> allInterfaces = new HashSet<>();
Queue<Class<?>> queue = new LinkedList<>();
queue.add(clazz);
while (!queue.isEmpty()) {
    Class<?> c = queue.poll();
    for (Class<?> iface : c.getInterfaces()) {
        if (allInterfaces.add(iface)) queue.add(iface);
    }
    if (c.getSuperclass() != null) queue.add(c.getSuperclass());
}
```

## B. 方法查找（含父类）

```java
// 在类及其父类中查找方法（getDeclaredMethod 不搜索父类）
public static Method findMethod(Class<?> clazz, String name, Class<?>... paramTypes) {
    Class<?> searchType = clazz;
    while (searchType != null) {
        try {
            Method m = searchType.getDeclaredMethod(name, paramTypes);
            m.setAccessible(true);
            return m;
        } catch (NoSuchMethodException e) {
            searchType = searchType.getSuperclass();
        }
    }
    throw new RuntimeException("方法未找到: " + name);
}

// 在类及其父类中查找字段
public static Field findField(Class<?> clazz, String fieldName) {
    Class<?> searchType = clazz;
    while (searchType != null && searchType != Object.class) {
        try {
            Field f = searchType.getDeclaredField(fieldName);
            f.setAccessible(true);
            return f;
        } catch (NoSuchFieldException e) {
            searchType = searchType.getSuperclass();
        }
    }
    throw new RuntimeException("字段未找到: " + fieldName);
}
```

## C. 枚举反射

```java
enum Direction { NORTH, SOUTH, EAST, WEST }

// 通过反射获取枚举值
Class<Direction> enumClass = Direction.class;
System.out.println(enumClass.isEnum()); // true

// 获取所有枚举常量
Direction[] values = enumClass.getEnumConstants();
for (Direction d : values) {
    System.out.println(d.name() + " ordinal=" + d.ordinal());
}

// 通过名称获取枚举值（反射）
Direction north = Enum.valueOf(Direction.class, "NORTH");
System.out.println(north); // NORTH

// 枚举私有构造器（不能被反射实例化）
// Constructor<?> ctor = enumClass.getDeclaredConstructors()[0];
// ctor.setAccessible(true);
// ctor.newInstance("NEW", 4); // 抛 IllegalArgumentException
```

## D. 反射与内部类

```java
public class OuterClass {
    private int x = 10;

    class InnerClass {
        private int y = 20;
        public int sum() { return x + y; } // 访问外部类字段
    }

    static class StaticNested {
        public void hello() { System.out.println("StaticNested"); }
    }

    public static void main(String[] args) throws Exception {
        // 获取内部类（非静态）
        Class<?> innerClass = Class.forName("OuterClass$InnerClass");
        // 非静态内部类需要外部类实例
        OuterClass outer = new OuterClass();
        Constructor<?> ctor = innerClass.getDeclaredConstructor(OuterClass.class);
        ctor.setAccessible(true);
        Object inner = ctor.newInstance(outer);

        Method sumMethod = innerClass.getMethod("sum");
        System.out.println("sum = " + sumMethod.invoke(inner)); // 30

        // 获取静态嵌套类
        Class<?> nestedClass = Class.forName("OuterClass$StaticNested");
        Object nested = nestedClass.getDeclaredConstructor().newInstance();
        nestedClass.getMethod("hello").invoke(nested); // StaticNested

        // 获取外部类（从内部类）
        System.out.println("声明类: " + innerClass.getDeclaringClass()); // OuterClass
        System.out.println("是否内部类: " + !java.lang.reflect.Modifier.isStatic(innerClass.getModifiers()));
    }
}
```

## E. 反射异常处理

```java
// 反射异常层次：
// ReflectiveOperationException（基类）
//   ├── ClassNotFoundException    - Class.forName 找不到类
//   ├── NoSuchMethodException     - 找不到方法
//   ├── NoSuchFieldException      - 找不到字段
//   ├── InstantiationException    - 无法实例化（抽象类/接口）
//   ├── IllegalAccessException    - 访问权限不足
//   └── InvocationTargetException - 方法内部抛异常（需 getCause()）

public static void invokeWithProperHandling(Method method, Object target, Object... args) {
    try {
        method.invoke(target, args);
    } catch (InvocationTargetException e) {
        // 被调用方法抛出的异常
        Throwable cause = e.getCause();
        if (cause instanceof RuntimeException) throw (RuntimeException) cause;
        if (cause instanceof Error) throw (Error) cause;
        throw new RuntimeException("反射调用目标方法异常", cause);
    } catch (IllegalAccessException e) {
        throw new RuntimeException("无访问权限，请检查 setAccessible", e);
    } catch (Exception e) {
        throw new RuntimeException("反射调用失败", e);
    }
}
```

---

# 附录：反射 API 速查表

| 方法 | 说明 | 是否包含继承 | 是否含 private |
|------|------|------------|----------------|
| `getFields()` | 获取所有 public 字段 | 是 | 否 |
| `getDeclaredFields()` | 获取本类所有字段 | 否 | 是 |
| `getMethods()` | 获取所有 public 方法 | 是 | 否 |
| `getDeclaredMethods()` | 获取本类所有方法 | 否 | 是 |
| `getConstructors()` | 获取所有 public 构造器 | N/A | 否 |
| `getDeclaredConstructors()` | 获取所有构造器 | N/A | 是 |
| `getAnnotations()` | 获取所有注解（含继承） | 是 | N/A |
| `getDeclaredAnnotations()` | 获取本类所有注解 | 否 | N/A |
| `getSuperclass()` | 获取直接父类 | N/A | N/A |
| `getInterfaces()` | 获取直接实现的接口 | 否 | N/A |
| `getGenericSuperclass()` | 获取带泛型的父类 | N/A | N/A |
| `getGenericInterfaces()` | 获取带泛型的接口 | 否 | N/A |
| `getTypeParameters()` | 获取类的类型参数 | N/A | N/A |
| `getModifiers()` | 获取修饰符（用 Modifier 解析）| N/A | N/A |

