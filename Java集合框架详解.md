# Java 集合框架详解 - 从零到精通

> 本文档涵盖 Java 集合框架的完整知识体系，从底层数据结构到源码实现，从基础使用到高并发场景，适合初学者入门与进阶学习。所有代码示例均可直接运行。

---

## 目录

- [Part 1: 集合框架概览](#part-1)
- [Part 2: ArrayList 深度解析](#part-2)
- [Part 3: LinkedList 深度解析](#part-3)
- [Part 4: HashMap 深度解析](#part-4)
- [Part 5: LinkedHashMap 深度解析](#part-5)
- [Part 6: TreeMap 深度解析](#part-6)
- [Part 7: HashSet / LinkedHashSet / TreeSet](#part-7)
- [Part 8: ConcurrentHashMap 深度解析](#part-8)
- [Part 9: Queue 与 Deque](#part-9)
- [Part 10: 常用算法与数据结构](#part-10)
- [Part 11: 性能对比与选型指南](#part-11)
- [Part 12: 常见面试题 FAQ](#part-12)
---

# Part 1: 集合框架概览

## 1.1 为什么需要集合框架

在 Java 引入集合框架（Java Collections Framework，JCF）之前，程序员通常使用数组来存储多个对象。但数组有明显的局限性：

1. **长度固定**：数组一旦创建，长度不能改变
2. **类型单一**：数组只能存储同一类型的元素
3. **操作繁琐**：插入、删除等操作需要手动移动元素
4. **缺乏算法支持**：没有内置的排序、搜索等功能

Java 集合框架（JDK 1.2 引入）提供了一套统一的接口和实现，彻底解决了上述问题。

## 1.2 集合框架整体架构（ASCII UML 继承图）

```
                    +--------------------------------------------------+
                    |          Java Collections Framework               |
                    +--------------------------------------------------+
                                          |
                    +---------------------+---------------------+
                    |                                           |
          +---------+----------+                   +-----------+-----------+
          |  <<interface>>     |                   |   <<interface>>       |
          |   Iterable<E>      |                   |      Map<K,V>         |
          +---------+----------+                   +-----------+-----------+
                    |                                          |
          +---------+----------+          +-----------+-------+--------+
          |  <<interface>>     |          |           |                |
          |   Collection<E>    |     HashMap<K,V> TreeMap<K,V>  LinkedHashMap<K,V>
          +---------+----------+
                    |
     +--------------+--------------+
     |              |              |
+----+------+  +----+------+  +----+--------+
|<<iface>>  |  |<<iface>>  |  |<<iface>>    |
| List<E>   |  |  Set<E>   |  |  Queue<E>   |
+----+------+  +----+------+  +----+--------+
     |               |              |
  ArrayList      HashSet        ArrayDeque
  LinkedList     LinkedHashSet  PriorityQueue
  Vector         TreeSet        LinkedList
  Stack
```

**Map 接口继承体系：**

```
          +----------------+
          | <<interface>>  |
          |   Map<K,V>     |
          +-------+--------+
                  |
     +------------+-------------+
     |            |             |
 HashMap      Hashtable    +---+--------+
     |        (legacy)     | <<iface>>  |
     |                     | SortedMap  |
     +-> LinkedHashMap     +---+--------+
                               |
                           +---+----------+
                           | <<iface>>    |
                           | NavigableMap |
                           +---+----------+
                               |
                           TreeMap
```

## 1.3 Collection 接口体系详解

`Collection<E>` 是所有单列集合的根接口，定义了集合的基本操作：

```java
public interface Collection<E> extends Iterable<E> {
    // 基本操作
    int size();
    boolean isEmpty();
    boolean contains(Object o);
    Iterator<E> iterator();
    Object[] toArray();
    <T> T[] toArray(T[] a);

    // 修改操作
    boolean add(E e);
    boolean remove(Object o);

    // 批量操作
    boolean containsAll(Collection<?> c);
    boolean addAll(Collection<? extends E> c);
    boolean removeAll(Collection<?> c);
    boolean retainAll(Collection<?> c);  // 保留交集
    void clear();

    // Java 8 新增
    default boolean removeIf(Predicate<? super E> filter) { ... }
    default Stream<E> stream() { ... }
    default Stream<E> parallelStream() { ... }
}
```

### 1.3.1 List 接口

`List<E>` 继承 `Collection<E>`，额外特性：
- **有序**：元素按插入顺序排列
- **允许重复**：同一元素可以出现多次
- **支持索引访问**：通过下标访问元素

主要实现：`ArrayList`、`LinkedList`、`Vector`、`Stack`、`CopyOnWriteArrayList`

```java
public interface List<E> extends Collection<E> {
    E get(int index);
    E set(int index, E element);
    void add(int index, E element);
    E remove(int index);
    int indexOf(Object o);
    int lastIndexOf(Object o);
    List<E> subList(int fromIndex, int toIndex); // 返回视图（非拷贝）
    ListIterator<E> listIterator();              // 双向迭代器
}
```

### 1.3.2 Set 接口

`Set<E>` 继承 `Collection<E>`，额外特性：
- **不允许重复**：每个元素只能出现一次（通过 `equals` 判断）
- **无序**（HashSet）或**有序**（TreeSet/LinkedHashSet）

主要实现：`HashSet`、`LinkedHashSet`、`TreeSet`、`CopyOnWriteArraySet`

### 1.3.3 Queue 接口

`Queue<E>` 继承 `Collection<E>`，表示 FIFO 队列：

| 操作     | 抛异常版本   | 返回特殊值版本 |
|----------|-------------|--------------|
| 入队     | `add(e)`    | `offer(e)`   |
| 出队     | `remove()`  | `poll()`     |
| 查看队头  | `element()` | `peek()`     |

`Deque<E>` 继承 `Queue<E>`，表示双端队列，可作为栈或队列使用。

## 1.4 Map 接口体系

`Map<K,V>` 存储键值对，键不重复：

```java
public interface Map<K, V> {
    int size();
    boolean isEmpty();
    boolean containsKey(Object key);
    boolean containsValue(Object value);
    V get(Object key);
    V put(K key, V value);
    V remove(Object key);
    void putAll(Map<? extends K, ? extends V> m);
    void clear();
    Set<K> keySet();
    Collection<V> values();
    Set<Map.Entry<K, V>> entrySet();

    // Java 8 新增（default 方法）
    default V getOrDefault(Object key, V defaultValue) { ... }
    default V putIfAbsent(K key, V value) { ... }
    default V computeIfAbsent(K key, Function<? super K, ? extends V> fn) { ... }
    default V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> fn) { ... }
    default V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> fn) { ... }
    default void forEach(BiConsumer<? super K, ? super V> action) { ... }
    default void replaceAll(BiFunction<? super K, ? super V, ? extends V> fn) { ... }
}
```

## 1.5 Iterator / Iterable / Comparable / Comparator

### 1.5.1 Iterable 与 Iterator

```java
// Iterable 接口：实现此接口的对象可以使用 for-each 循环
public interface Iterable<T> {
    Iterator<T> iterator();

    default void forEach(Consumer<? super T> action) {
        Objects.requireNonNull(action);
        for (T t : this) {
            action.accept(t);
        }
    }
}

// Iterator 接口：迭代器，用于遍历集合
public interface Iterator<E> {
    boolean hasNext();   // 是否还有下一个元素
    E next();            // 返回下一个元素
    default void remove() {
        throw new UnsupportedOperationException("remove");
    }
    default void forEachRemaining(Consumer<? super E> action) { ... }
}
```

**迭代器使用示例：**

```java
import java.util.*;

public class IteratorDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C", "D"));

        // 方式1：传统 Iterator（遍历时安全删除）
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            String s = it.next();
            if ("B".equals(s)) {
                it.remove(); // 安全删除，不会抛 ConcurrentModificationException
            }
        }
        System.out.println(list); // [A, C, D]

        // 方式2：for-each（底层也是 Iterator）
        for (String s : list) {
            System.out.print(s + " "); // A C D
        }

        // 方式3：ListIterator（双向，可以向前遍历）
        ListIterator<String> lit = list.listIterator(list.size());
        System.out.println();
        while (lit.hasPrevious()) {
            System.out.print(lit.previous() + " "); // D C A
        }
    }
}
```

### 1.5.2 Comparable 与 Comparator

```java
// Comparable：自然排序，实现此接口的对象可以相互比较
// 类本身定义排序规则（"我能和自己的同类比较"）
public interface Comparable<T> {
    int compareTo(T o);
    // 返回负数：this < o
    // 返回0：   this == o（建议与 equals 一致）
    // 返回正数：this > o
}

// Comparator：外部比较器，可以为任意类定义排序规则（不修改原类）
@FunctionalInterface
public interface Comparator<T> {
    int compare(T o1, T o2);

    // Java 8 静态工厂方法（链式调用）
    static <T extends Comparable<? super T>> Comparator<T> naturalOrder() { ... }
    static <T extends Comparable<? super T>> Comparator<T> reverseOrder() { ... }
    static <T, U extends Comparable<? super U>> Comparator<T>
        comparing(Function<? super T, ? extends U> keyExtractor) { ... }
    default Comparator<T> thenComparing(Comparator<? super T> other) { ... }
    default Comparator<T> reversed() { ... }
}
```

**Comparable vs Comparator 示例：**

```java
import java.util.*;

// 实现 Comparable：自然排序按年龄
class Person implements Comparable<Person> {
    String name;
    int age;

    Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    @Override
    public int compareTo(Person other) {
        return Integer.compare(this.age, other.age); // 按年龄升序
    }

    @Override
    public String toString() {
        return name + "(" + age + ")";
    }
}

public class ComparableDemo {
    public static void main(String[] args) {
        List<Person> people = new ArrayList<>(Arrays.asList(
            new Person("Alice", 30),
            new Person("Bob", 25),
            new Person("Charlie", 35),
            new Person("Dave", 25)
        ));

        // 自然排序（按年龄升序）
        Collections.sort(people);
        System.out.println(people); // [Bob(25), Dave(25), Alice(30), Charlie(35)]

        // Comparator：按姓名排序（不修改 Person 类）
        people.sort(Comparator.comparing(p -> p.name));
        System.out.println(people); // [Alice(30), Bob(25), Charlie(35), Dave(25)]

        // 链式比较：先按年龄，再按姓名
        people.sort(Comparator.comparingInt((Person p) -> p.age)
                              .thenComparing(p -> p.name));
        System.out.println(people); // [Bob(25), Dave(25), Alice(30), Charlie(35)]

        // 逆序
        people.sort(Comparator.comparingInt((Person p) -> p.age).reversed());
        System.out.println(people); // [Charlie(35), Alice(30), Bob(25), Dave(25)]
    }
}
```

## 1.6 Collections 工具类

`java.util.Collections` 提供了一系列静态方法用于操作集合：

```java
import java.util.*;

public class CollectionsDemo {
    public static void main(String[] args) {
        List<Integer> list = new ArrayList<>(Arrays.asList(3, 1, 4, 1, 5, 9, 2, 6));

        // ===== 排序相关 =====
        Collections.sort(list);
        System.out.println("排序后: " + list); // [1, 1, 2, 3, 4, 5, 6, 9]

        Collections.sort(list, Comparator.reverseOrder());
        System.out.println("逆序: " + list);   // [9, 6, 5, 4, 3, 2, 1, 1]

        Collections.reverse(list);
        System.out.println("反转: " + list);   // [1, 1, 2, 3, 4, 5, 6, 9]

        Collections.shuffle(list);             // 随机打乱
        Collections.shuffle(list, new Random(42)); // 使用指定随机数种子（可重现）

        // ===== 查找相关 =====
        Collections.sort(list); // binarySearch 前必须排序
        int idx = Collections.binarySearch(list, 5);
        System.out.println("5 的位置: " + idx);

        System.out.println("最大值: " + Collections.max(list)); // 9
        System.out.println("最小值: " + Collections.min(list)); // 1
        System.out.println("1的频率: " + Collections.frequency(list, 1)); // 2

        // ===== 填充相关 =====
        List<String> strList = new ArrayList<>(Arrays.asList("a", "b", "c"));
        Collections.fill(strList, "X");
        System.out.println("填充: " + strList); // [X, X, X]

        List<String> copies = Collections.nCopies(5, "hello"); // 5个"hello"的不可变列表

        // ===== 互换/移动 =====
        List<Integer> nums = new ArrayList<>(Arrays.asList(1, 2, 3, 4, 5));
        Collections.swap(nums, 0, 4);
        System.out.println("交换0和4: " + nums); // [5, 2, 3, 4, 1]

        Collections.rotate(nums, 2); // 右移2位（循环移动）
        System.out.println("右移2: " + nums); // [4, 1, 5, 2, 3]

        // ===== 线程安全包装 =====
        List<String> syncList = Collections.synchronizedList(new ArrayList<>());
        Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
        Set<String> syncSet = Collections.synchronizedSet(new HashSet<>());
        // 注意：遍历时仍需手动同步！
        synchronized (syncList) {
            for (String s : syncList) {
                // 安全遍历
            }
        }

        // ===== 不可变视图 =====
        List<String> immutableList = Collections.unmodifiableList(
            new ArrayList<>(Arrays.asList("a", "b"))
        );
        // immutableList.add("c"); // 抛出 UnsupportedOperationException

        // ===== 空集合（单例，不可变）=====
        List<String> emptyList = Collections.emptyList();
        Set<String> emptySet = Collections.emptySet();
        Map<String, String> emptyMap = Collections.emptyMap();

        // ===== 单元素集合 =====
        List<String> singleton = Collections.singletonList("only");
        Set<String> singletonSet = Collections.singleton("only");
    }
}
```

## 1.7 Arrays 工具类

`java.util.Arrays` 提供操作数组的静态方法：

```java
import java.util.*;

public class ArraysDemo {
    public static void main(String[] args) {
        int[] arr = {3, 1, 4, 1, 5, 9, 2, 6};

        // ===== 排序 =====
        Arrays.sort(arr);
        System.out.println(Arrays.toString(arr)); // [1, 1, 2, 3, 4, 5, 6, 9]

        int[] partial = {5, 3, 8, 1, 7, 2};
        Arrays.sort(partial, 1, 4); // 只排序 [1,4) 区间
        System.out.println(Arrays.toString(partial)); // [5, 1, 3, 8, 7, 2]

        // 对象数组使用 Comparator
        String[] strs = {"banana", "apple", "cherry"};
        Arrays.sort(strs, Comparator.reverseOrder());
        System.out.println(Arrays.toString(strs)); // [cherry, banana, apple]

        // ===== 二分查找（前提：已排序）=====
        Arrays.sort(arr);
        int pos = Arrays.binarySearch(arr, 5);
        System.out.println("5 在位置: " + pos);

        // ===== 复制 =====
        int[] copy = Arrays.copyOf(arr, 5);              // 复制前5个元素
        int[] rangeCopy = Arrays.copyOfRange(arr, 2, 6); // 复制 [2,6) 区间

        // ===== 填充 =====
        int[] filled = new int[5];
        Arrays.fill(filled, 7);
        System.out.println(Arrays.toString(filled)); // [7, 7, 7, 7, 7]

        // ===== 比较 =====
        int[] a = {1, 2, 3};
        int[] b = {1, 2, 3};
        System.out.println(Arrays.equals(a, b)); // true
        System.out.println(a == b);              // false（引用不同）

        int[][] matrix1 = {{1,2},{3,4}};
        int[][] matrix2 = {{1,2},{3,4}};
        System.out.println(Arrays.deepEquals(matrix1, matrix2)); // true

        // ===== 转为 List =====
        String[] strArr = {"a", "b", "c"};
        List<String> fixedList = Arrays.asList(strArr); // 固定大小，不能增删
        // fixedList.add("d"); // UnsupportedOperationException
        fixedList.set(0, "A"); // 可以修改已有元素

        // 转为真正可变的 ArrayList
        List<String> mutableList = new ArrayList<>(Arrays.asList(strArr));
        mutableList.add("d"); // 正常

        // ===== 流操作 =====
        Arrays.stream(arr).filter(x -> x > 3).forEach(System.out::println);

        // ===== toString =====
        System.out.println(Arrays.toString(arr));            // 一维数组
        System.out.println(Arrays.deepToString(matrix1));   // [[1, 2], [3, 4]]
    }
}
```

## 1.8 fail-fast vs fail-safe 机制

### 1.8.1 fail-fast（快速失败）

`java.util` 包下的集合类大多采用 fail-fast 机制。当遍历集合时，如果集合结构发生变化（增加/删除元素），迭代器会立即抛出 `ConcurrentModificationException`。

**实现原理：** 集合内部维护一个 `modCount` 字段，每次结构性修改（add/remove 等）都会递增 `modCount`。迭代器在创建时记录 `expectedModCount = modCount`。每次调用 `next()` 时检查 `modCount == expectedModCount`，不一致则抛异常。

```java
// ArrayList 中 Itr（内部迭代器类）的部分源码
private class Itr implements Iterator<E> {
    int cursor;       // 下一个元素的索引
    int lastRet = -1; // 上一个返回元素的索引
    int expectedModCount = modCount; // 创建时记录 modCount

    public E next() {
        checkForComodification(); // 先检查是否被修改
        // ...返回下一个元素
    }

    final void checkForComodification() {
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }

    // 通过 Iterator 删除是安全的（同步更新了 expectedModCount）
    public void remove() {
        // ...
        ArrayList.this.remove(lastRet);
        expectedModCount = modCount; // 同步 modCount！
    }
}
```

**触发示例与正确写法：**

```java
import java.util.*;

public class FailFastDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));

        // 错误示例1：for-each 中删除（底层是 Iterator，触发 fail-fast）
        try {
            for (String s : list) {
                if ("B".equals(s)) {
                    list.remove(s); // ConcurrentModificationException!
                }
            }
        } catch (ConcurrentModificationException e) {
            System.out.println("捕获 ConcurrentModificationException");
        }

        // 错误示例2：普通 for 循环删除（不抛异常但结果错误）
        List<String> list2 = new ArrayList<>(Arrays.asList("A", "B", "B", "C"));
        for (int i = 0; i < list2.size(); i++) {
            if ("B".equals(list2.get(i))) {
                list2.remove(i); // 删除后索引错位，会漏掉相邻的 B
            }
        }
        System.out.println(list2); // [A, B, C]  -- 第二个B没被删掉！

        // 正确方式1：使用 Iterator.remove()
        List<String> list3 = new ArrayList<>(Arrays.asList("A", "B", "B", "C"));
        Iterator<String> it = list3.iterator();
        while (it.hasNext()) {
            if ("B".equals(it.next())) {
                it.remove(); // 安全删除
            }
        }
        System.out.println(list3); // [A, C]

        // 正确方式2：Java 8 removeIf（推荐）
        List<String> list4 = new ArrayList<>(Arrays.asList("A", "B", "B", "C"));
        list4.removeIf("B"::equals);
        System.out.println(list4); // [A, C]

        // 正确方式3：倒序遍历删除
        List<String> list5 = new ArrayList<>(Arrays.asList("A", "B", "B", "C"));
        for (int i = list5.size() - 1; i >= 0; i--) {
            if ("B".equals(list5.get(i))) {
                list5.remove(i);
            }
        }
        System.out.println(list5); // [A, C]
    }
}
```

### 1.8.2 fail-safe（安全失败）

`java.util.concurrent` 包下的集合类采用 fail-safe 机制。遍历时不会抛出 `ConcurrentModificationException`，因为它们遍历的是集合的**副本**或使用了特殊机制。

```java
import java.util.concurrent.*;
import java.util.*;

public class FailSafeDemo {
    public static void main(String[] args) {
        // CopyOnWriteArrayList：写时复制，遍历的是快照，不会抛异常
        CopyOnWriteArrayList<String> cowList = new CopyOnWriteArrayList<>(
            Arrays.asList("A", "B", "C")
        );
        for (String s : cowList) {
            if ("B".equals(s)) {
                cowList.remove(s); // 不抛异常，但迭代器遍历的是原始快照
            }
        }
        System.out.println(cowList); // [A, C]

        // ConcurrentHashMap：CAS + 分段锁，允许并发修改
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
        map.put("a", 1); map.put("b", 2); map.put("c", 3);
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            if ("b".equals(entry.getKey())) {
                map.remove("b"); // 不抛异常
            }
        }
        System.out.println(map); // {a=1, c=3}
    }
}
```

| 特性          | fail-fast                          | fail-safe                               |
|---------------|------------------------------------|-----------------------------------------|
| 代表类        | ArrayList, HashMap, HashSet        | CopyOnWriteArrayList, ConcurrentHashMap |
| 异常          | ConcurrentModificationException    | 不抛异常                                |
| 遍历对象      | 原集合                             | 副本/快照                               |
| 内存开销      | 低                                 | 高（需要副本）                          |
| 修改可见性    | 立即可见                           | 遍历期间不可见                          |
| 适用场景      | 单线程/不需要并发修改              | 并发读多写少                            |


---

# Part 2: ArrayList 深度解析

## 2.1 底层数据结构

`ArrayList` 底层使用 `Object[]` 数组存储元素。实现了 `RandomAccess` 标记接口，支持 O(1) 随机访问。

```java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable {

    // 默认初始容量（懒初始化，第一次 add 才分配）
    private static final int DEFAULT_CAPACITY = 10;

    // 空数组（用于有参构造器传入0时）
    private static final Object[] EMPTY_ELEMENTDATA = {};

    // 空数组（用于默认构造器，与 EMPTY_ELEMENTDATA 区分以知道何时扩容）
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    // 实际存储元素的数组（transient 表示不直接序列化）
    transient Object[] elementData;

    // 实际元素数量（size <= elementData.length）
    private int size;

    // 最大数组容量（避免某些 JVM 在数组头部存储元数据导致溢出）
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
}
```

## 2.2 初始容量与懒初始化

```java
// 默认构造器：创建空数组，不分配实际容量（懒初始化）
// 内存中只有一个空数组引用，直到第一次 add 才分配10个元素的空间
public ArrayList() {
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}

// 带初始容量的构造器
public ArrayList(int initialCapacity) {
    if (initialCapacity > 0) {
        this.elementData = new Object[initialCapacity]; // 直接分配指定容量
    } else if (initialCapacity == 0) {
        this.elementData = EMPTY_ELEMENTDATA;
    } else {
        throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
    }
}

// 通过集合构造
public ArrayList(Collection<? extends E> c) {
    Object[] a = c.toArray();
    if ((size = a.length) != 0) {
        if (c.getClass() == ArrayList.class) {
            elementData = a; // 直接使用 ArrayList 的底层数组
        } else {
            elementData = Arrays.copyOf(a, size, Object[].class);
        }
    } else {
        elementData = EMPTY_ELEMENTDATA;
    }
}
```

**懒初始化的好处：** 使用默认构造器创建 ArrayList 时，不立即分配10个元素的内存，只有第一次 `add` 时才分配，节省内存。如果创建了大量 ArrayList 但从未使用，节省效果显著。

## 2.3 扩容机制（1.5倍扩容）

```
内存变化过程（使用默认构造器）：

创建时：
  elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA（空数组引用）
  size = 0

第一次 add(e)：
  calculateCapacity → 因为是 DEFAULTCAPACITY_EMPTY_ELEMENTDATA
  → minCapacity = max(DEFAULT_CAPACITY=10, 1) = 10
  → grow(10) → elementData = new Object[10]
  elementData = [e1, null, null, null, null, null, null, null, null, null]
  size = 1

第11次 add：
  size+1=11 > elementData.length=10 → 触发扩容
  oldCapacity = 10
  newCapacity = 10 + (10 >> 1) = 10 + 5 = 15
  Arrays.copyOf(elementData, 15) → 创建容量为15的新数组，复制元素

扩容容量序列（从10开始）：
  10 → 15 → 22 → 33 → 49 → 73 → 109 → 163 → 244 → 366 → ...
  （每次 * 1.5，有小的取整误差）
```

**扩容源码（JDK 8）：**

```java
// 确保容量充足
private void ensureCapacityInternal(int minCapacity) {
    ensureExplicitCapacity(calculateCapacity(elementData, minCapacity));
}

private static int calculateCapacity(Object[] elementData, int minCapacity) {
    // 如果是默认构造器创建的空列表，首次扩容到 DEFAULT_CAPACITY（10）
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
        return Math.max(DEFAULT_CAPACITY, minCapacity);
    }
    return minCapacity;
}

private void ensureExplicitCapacity(int minCapacity) {
    modCount++; // 结构性修改计数（用于 fail-fast）

    // 如果需要的容量大于当前数组长度，则扩容
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}

private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 新容量 = 旧容量 + 旧容量/2（即 1.5 倍，右移1位 = 除以2）
    int newCapacity = oldCapacity + (oldCapacity >> 1);

    // 如果1.5倍还不够（如 addAll 一次添加大量元素），直接用 minCapacity
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;

    // 如果超过最大值，特殊处理
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);

    // Arrays.copyOf 底层调用 System.arraycopy，将旧数组复制到新数组
    elementData = Arrays.copyOf(elementData, newCapacity);
}

private static int hugeCapacity(int minCapacity) {
    if (minCapacity < 0) // overflow
        throw new OutOfMemoryError();
    return (minCapacity > MAX_ARRAY_SIZE) ?
        Integer.MAX_VALUE : MAX_ARRAY_SIZE;
}
```

**手动预分配容量（优化性能）：**

```java
// 场景：已知要插入大量元素，避免多次扩容
// 方式1：构造时指定初始容量（推荐）
List<Integer> list = new ArrayList<>(100000);

// 方式2：使用 ensureCapacity 手动触发扩容
List<Integer> list2 = new ArrayList<>();
list2.ensureCapacity(100000); // 触发一次性扩容到100000

// 反向操作：trimToSize() 去掉多余容量，节省内存
list2.trimToSize(); // elementData.length == size
```

## 2.4 核心方法源码分析

### 2.4.1 add 方法（时间复杂度：O(1) 均摊 / O(n) 中间插入）

```java
// 在末尾添加（均摊 O(1)，偶尔触发扩容 O(n)）
public boolean add(E e) {
    ensureCapacityInternal(size + 1); // 确保有足够容量
    elementData[size++] = e;          // 在末尾放入元素，size 自增
    return true;
}

// 在指定位置插入（O(n)，需要移动元素）
public void add(int index, E element) {
    rangeCheckForAdd(index); // 范围检查：0 <= index <= size
    ensureCapacityInternal(size + 1);
    // 将 index 及之后的元素向后移动一位
    // System.arraycopy 是 native 方法，比手动循环快得多
    System.arraycopy(elementData, index, elementData, index + 1, size - index);
    elementData[index] = element;
    size++;
}

// addAll：一次添加多个元素
public boolean addAll(Collection<? extends E> c) {
    Object[] a = c.toArray();
    int numNew = a.length;
    ensureCapacityInternal(size + numNew); // 一次性扩容
    System.arraycopy(a, 0, elementData, size, numNew);
    size += numNew;
    return numNew != 0;
}
```

### 2.4.2 remove 方法

```java
// 按索引删除（O(n)，需要移动元素）
public E remove(int index) {
    rangeCheck(index); // 检查 index < size（不检查 < 0，会触发数组越界异常）
    modCount++;
    E oldValue = elementData(index);

    int numMoved = size - index - 1;
    if (numMoved > 0)
        // 将 index+1 及之后的元素向前移动一位
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null; // 清除最后位置的引用，防止内存泄漏（帮助 GC）
    return oldValue;
}

// 按对象删除（O(n)，删除第一个匹配的元素）
public boolean remove(Object o) {
    if (o == null) {
        for (int index = 0; index < size; index++)
            if (elementData[index] == null) {
                fastRemove(index);
                return true;
            }
    } else {
        for (int index = 0; index < size; index++)
            if (o.equals(elementData[index])) { // 使用 equals 比较
                fastRemove(index);
                return true;
            }
    }
    return false;
}

// 快速删除（不做范围检查，不返回值，内部使用）
private void fastRemove(int index) {
    modCount++;
    int numMoved = size - index - 1;
    if (numMoved > 0)
        System.arraycopy(elementData, index+1, elementData, index, numMoved);
    elementData[--size] = null;
}
```

### 2.4.3 get 和 set 方法（O(1) 随机访问）

```java
// O(1) 随机访问（直接通过数组下标）
public E get(int index) {
    rangeCheck(index); // 检查 0 <= index < size
    return elementData(index);
}

@SuppressWarnings("unchecked")
E elementData(int index) {
    return (E) elementData[index]; // 直接数组下标访问，极快
}

public E set(int index, E element) {
    rangeCheck(index);
    E oldValue = elementData(index);
    elementData[index] = element;
    return oldValue; // 返回旧值
}
```

### 2.4.4 indexOf 方法（O(n)）

```java
// 查找第一次出现的位置（O(n)）
public int indexOf(Object o) {
    if (o == null) {
        for (int i = 0; i < size; i++)
            if (elementData[i] == null)
                return i;
    } else {
        for (int i = 0; i < size; i++)
            if (o.equals(elementData[i])) // 使用 equals，不是 ==
                return i;
    }
    return -1; // 未找到
}

// 查找最后一次出现的位置（从后往前遍历，O(n)）
public int lastIndexOf(Object o) {
    if (o == null) {
        for (int i = size-1; i >= 0; i--)
            if (elementData[i] == null)
                return i;
    } else {
        for (int i = size-1; i >= 0; i--)
            if (o.equals(elementData[i]))
                return i;
    }
    return -1;
}
```

## 2.5 随机访问 O(1) vs 中间插入 O(n)

```
随机访问（get/set）：
  elementData[index] → 直接数组下标访问，O(1)
  特点：ArrayList 实现了 RandomAccess 接口，标记支持高效随机访问

中间插入（add(index, e)）：
  before:  [A, B, C, D, E]
                 ^ 在 index=2 处插入 X
  步骤1: System.arraycopy 将 C,D,E 向后移动
  after:   [A, B, _, C, D, E]
  步骤2: elementData[2] = X
  result:  [A, B, X, C, D, E]

  最坏情况（index=0，头部插入）：移动所有 n 个元素，O(n)
  平均情况（随机位置）：移动 n/2 个元素，O(n)

中间删除（remove(index)）：
  before:  [A, B, C, D, E]
                 ^ 删除 index=2（C）
  步骤1: System.arraycopy 将 D,E 向前移动
  中间:   [A, B, D, E, E]
  步骤2: elementData[--size] = null（清除末尾引用）
  result:  [A, B, D, E]
  时间复杂度同上：O(n)

性能图示：
  index=0 (头部)：移动 n 个元素，最慢  |████████████████████| O(n)
  index=n/2(中部)：移动 n/2 个元素     |██████████          | O(n/2)
  index=n (尾部)：不需要移动，最快     |                    | O(1)
```

```java
// 性能测试示例
public class ArrayListPerformance {
    public static void main(String[] args) {
        int N = 100000;
        List<Integer> list = new ArrayList<>();
        for (int i = 0; i < N; i++) list.add(i);

        // 随机访问：极快 O(1)
        long start = System.nanoTime();
        for (int i = 0; i < N; i++) list.get(i);
        System.out.printf("随机访问 %d 次: %d ms%n", N,
                          (System.nanoTime()-start)/1_000_000);

        // 头部插入：极慢 O(n)
        start = System.nanoTime();
        for (int i = 0; i < 1000; i++) list.add(0, i);
        System.out.printf("头部插入 1000 次: %d ms%n",
                          (System.nanoTime()-start)/1_000_000);

        // 尾部追加：快 O(1) 均摊
        start = System.nanoTime();
        for (int i = 0; i < N; i++) list.add(i);
        System.out.printf("尾部追加 %d 次: %d ms%n", N,
                          (System.nanoTime()-start)/1_000_000);
        // 典型输出：随机访问<1ms，头部插入100ms+，尾部追加<10ms
    }
}
```

## 2.6 subList 的陷阱（视图，非拷贝）

`subList` 返回的是**视图（View）**，而不是独立的拷贝。底层数据结构是同一个数组。

```java
import java.util.*;

public class SubListDemo {
    public static void main(String[] args) {
        List<String> original = new ArrayList<>(Arrays.asList("A","B","C","D","E"));
        List<String> sub = original.subList(1, 4); // 视图，对应索引 [1,4) → [B,C,D]

        System.out.println("original: " + original); // [A, B, C, D, E]
        System.out.println("sub: " + sub);            // [B, C, D]

        // 陷阱1：修改子列表 → 影响原列表
        sub.set(0, "X");
        System.out.println("original after sub.set: " + original); // [A, X, C, D, E]

        // 陷阱2：对子列表增删 → 影响原列表
        sub.add("Y");
        System.out.println("original after sub.add: " + original); // [A, X, C, D, Y, E]

        // 陷阱3：对原列表进行结构性修改 → 子列表失效（抛异常）
        original.add("Z");
        try {
            sub.get(0); // ConcurrentModificationException!
        } catch (ConcurrentModificationException e) {
            System.out.println("子列表失效了！");
        }

        // 正确做法：需要独立副本时，用 new ArrayList<>(subList)
        List<String> list2 = new ArrayList<>(Arrays.asList("A","B","C","D","E"));
        List<String> independentCopy = new ArrayList<>(list2.subList(1, 4));
        list2.add("Z"); // 修改原列表
        System.out.println("独立副本: " + independentCopy); // [B, C, D]，不受影响

        // 常用技巧：用 subList(x,y).clear() 快速删除一段连续元素 O(n)
        List<String> list3 = new ArrayList<>(Arrays.asList("A","B","C","D","E"));
        list3.subList(1, 4).clear(); // 删除索引 1,2,3 的元素
        System.out.println(list3); // [A, E]
    }
}
```

## 2.7 toArray() 的两种方式

```java
import java.util.*;

public class ToArrayDemo {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));

        // 方式1：toArray() - 返回 Object[]，不能直接强转为 String[]
        Object[] objArr = list.toArray();
        // String[] wrong = (String[]) list.toArray(); // ClassCastException!!!
        // 原因：Object[] 无法直接强转为 String[]（虽然元素都是 String）
        String s = (String) objArr[0]; // 只能逐个强转

        // 方式2：toArray(T[] a) - 推荐方式，返回指定类型数组
        // 传入长度为0的数组（JVM 自动创建正确大小的新数组）
        String[] strArr = list.toArray(new String[0]);
        System.out.println(Arrays.toString(strArr)); // [A, B, C]

        // 传入足够大的数组（旧风格，避免创建额外数组）
        String[] strArr2 = new String[list.size()];
        list.toArray(strArr2); // 直接填充到 strArr2
        System.out.println(Arrays.toString(strArr2)); // [A, B, C]

        // 传入过大的数组（多余位置填null，容易引起 NullPointerException）
        String[] strArr3 = new String[10];
        list.toArray(strArr3);
        System.out.println(Arrays.toString(strArr3));
        // [A, B, C, null, null, null, null, null, null, null]

        // Java 11+ 更简洁的写法（方法引用）
        // String[] strArr4 = list.toArray(String[]::new);
    }
}
```

## 2.8 与数组互转

```java
import java.util.*;
import java.util.stream.*;

public class ArrayListConversionDemo {
    public static void main(String[] args) {
        // ===== 数组 → List =====

        // 方式1：Arrays.asList（固定大小，不能增删，可修改元素）
        String[] arr = {"a", "b", "c"};
        List<String> fixed = Arrays.asList(arr);
        // fixed.add("d"); // UnsupportedOperationException
        fixed.set(0, "A"); // 可以
        System.out.println(fixed); // [A, b, c]

        // 方式2：new ArrayList<>(Arrays.asList(...)) —— 最常用
        List<String> mutable = new ArrayList<>(Arrays.asList("a", "b", "c"));
        mutable.add("d"); // 正常

        // 方式3：Collections.addAll
        List<String> list3 = new ArrayList<>();
        Collections.addAll(list3, arr);

        // 方式4：Java 8 Stream（适合需要过滤/转换的场景）
        List<String> list4 = Arrays.stream(arr).collect(Collectors.toList());

        // 基本类型数组转 List（int[] 不能直接用 Arrays.asList！）
        int[] intArr = {1, 2, 3};
        // List<Integer> wrong = Arrays.asList(intArr); // 错！得到 List<int[]>
        //                                                // 只有一个元素，是 int[] 本身

        // 正确方式：Stream.boxed()
        List<Integer> intList = Arrays.stream(intArr).boxed().collect(Collectors.toList());
        System.out.println(intList); // [1, 2, 3]

        // ===== List → 数组 =====
        List<String> list = new ArrayList<>(Arrays.asList("x", "y", "z"));
        String[] result = list.toArray(new String[0]);
        System.out.println(Arrays.toString(result)); // [x, y, z]

        // Java 11+
        // String[] result2 = list.toArray(String[]::new);

        // List<Integer> → int[]（需要 Stream）
        List<Integer> integerList = Arrays.asList(1, 2, 3);
        int[] intResult = integerList.stream().mapToInt(Integer::intValue).toArray();
        System.out.println(Arrays.toString(intResult)); // [1, 2, 3]
    }
}
```

## 2.9 序列化：transient elementData

`ArrayList` 的 `elementData` 字段被 `transient` 修饰，不会被 Java 默认序列化机制序列化。`ArrayList` 自定义了 `writeObject` 和 `readObject` 方法，只序列化实际有效的元素，而不是整个底层数组（数组末尾可能有大量空槽）。

```java
// ArrayList 自定义序列化（JDK 8 源码节选）
private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
    int expectedModCount = modCount;
    s.defaultWriteObject(); // 序列化非 transient 字段（size、loadFactor等）

    // 写入实际元素数量（用于反序列化时分配合适的数组大小）
    s.writeInt(size);

    // 只序列化有效元素（索引 0 到 size-1），跳过末尾的空槽
    for (int i = 0; i < size; i++) {
        s.writeObject(elementData[i]);
    }

    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}

private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
    elementData = EMPTY_ELEMENTDATA;
    s.defaultReadObject();
    s.readInt(); // 读取 capacity（忽略，只是为了向后兼容）

    if (size > 0) {
        // 分配合适大小的数组（不超过 DEFAULT_CAPACITY）
        int capacity = calculateCapacity(elementData, size);
        ensureCapacityInternal(size);
        Object[] a = elementData;
        for (int i = 0; i < size; i++) {
            a[i] = s.readObject(); // 逐个反序列化元素
        }
    }
}
```

**好处举例：**
- ArrayList 容量 capacity=1000，但只有 size=10 个元素
- 默认序列化：会序列化整个 elementData 数组（1000个槽位，990个是 null）
- 自定义序列化：只序列化10个有效元素，节省约 99% 的序列化大小

## 2.10 ArrayList vs Vector

`Vector` 是 JDK 1.0 就存在的同步 List，`ArrayList` 是 JDK 1.2 引入的非同步替代品。

```java
// Vector 的方法都加了 synchronized
public synchronized boolean add(E e) {
    modCount++;
    ensureCapacityHelper(elementCount + 1);
    elementData[elementCount++] = e;
    return true;
}

// ArrayList 的方法没有 synchronized，性能更好
public boolean add(E e) {
    ensureCapacityInternal(size + 1);
    elementData[size++] = e;
    return true;
}
```

| 特性       | ArrayList      | Vector                    |
|------------|----------------|---------------------------|
| 线程安全   | 否             | 是（方法级 synchronized） |
| 性能       | 好             | 差（同步开销）            |
| 扩容倍数   | 1.5倍          | 2倍（默认）               |
| 推荐使用   | 是             | 否（已过时）              |
| 出现版本   | JDK 1.2        | JDK 1.0                   |

**线程安全替代方案：**
1. `Collections.synchronizedList(new ArrayList<>())`：包装器，锁粒度大（对象级）
2. `CopyOnWriteArrayList`：写时复制，适合读多写少场景
3. 手动使用 `synchronized` 块


---

# Part 3: LinkedList 深度解析

## 3.1 底层数据结构（双向链表）

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable {

    transient int size = 0;
    transient Node<E> first; // 头节点（prev=null）
    transient Node<E> last;  // 尾节点（next=null）

    // 内部节点类（双向链表节点）
    private static class Node<E> {
        E item;       // 存储的元素
        Node<E> next; // 后继节点指针
        Node<E> prev; // 前驱节点指针

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
}
```

**双向链表结构图：**

```
LinkedList（4个元素）：

 first                                          last
   |                                              |
   v                                              v
+------+-----+------+   +------+-----+------+   +------+-----+------+   +------+-----+------+
| null | "A" | next-+-->| prev | "B" | next-+-->| prev | "C" | next-+-->| prev | "D" | null |
+------+-----+------+   +------+-----+------+   +------+-----+------+   +------+-----+------+
                    <---+-prev             <---+-prev             <---+-prev
 Node[0]              Node[1]              Node[2]              Node[3]

特点：
- 每个节点包含：prev指针(8B) + item引用(8B) + next指针(8B) = 24B（不含对象头16B）
- 相比 ArrayList，每个元素额外耗费约 32B 内存开销
- 插入/删除节点只需修改指针，O(1)（但需要先找到位置）
- 不支持 O(1) 随机访问（需要从头/尾遍历）
```

## 3.2 实现了 Deque 接口（可作为栈和队列）

`LinkedList` 同时实现了 `List` 和 `Deque` 接口，用途多样：

| 接口/用途   | 可用方法                                |
|------------|----------------------------------------|
| List       | get/set/add(index)/remove(index)（O(n)）|
| Queue(FIFO)| offer/poll/peek（操作队头/队尾）       |
| Stack(LIFO)| push/pop/peek（操作链表头部）          |
| Deque      | addFirst/addLast/removeFirst/removeLast|

```java
import java.util.*;

public class LinkedListDequeDemo {
    public static void main(String[] args) {
        // === 作为队列（FIFO）===
        Queue<String> queue = new LinkedList<>();
        queue.offer("A"); // 入队（队尾 addLast）
        queue.offer("B");
        queue.offer("C");
        System.out.println("队列: " + queue);   // [A, B, C]
        System.out.println(queue.poll());        // 出队（队头）: A
        System.out.println(queue.peek());        // 查看队头（不删除）: B
        System.out.println("队列: " + queue);   // [B, C]

        // === 作为栈（LIFO）===
        Deque<String> stack = new LinkedList<>();
        stack.push("A"); // 入栈（addFirst，放到链表头部）
        stack.push("B"); // 头部: B → A
        stack.push("C"); // 头部: C → B → A
        System.out.println("栈: " + stack);     // [C, B, A]
        System.out.println(stack.pop());         // 出栈（removeFirst）: C
        System.out.println(stack.peek());        // 查看栈顶（不删除）: B
        System.out.println("栈: " + stack);     // [B, A]

        // === 作为双端队列 ===
        Deque<String> deque = new LinkedList<>();
        deque.addFirst("B");  // [B]
        deque.addFirst("A");  // [A, B]
        deque.addLast("C");   // [A, B, C]
        deque.addLast("D");   // [A, B, C, D]
        System.out.println("双端队列: " + deque); // [A, B, C, D]
        System.out.println(deque.removeFirst()); // A
        System.out.println(deque.removeLast());  // D
        System.out.println("双端队列: " + deque); // [B, C]

        // === offerXxx/pollXxx/peekXxx 系列（不抛异常）===
        deque.offerFirst("Z");
        deque.offerLast("Z");
        System.out.println(deque.peekFirst()); // Z
        System.out.println(deque.peekLast());  // Z
        System.out.println(deque.pollFirst()); // Z
        System.out.println(deque.pollLast());  // Z
    }
}
```

## 3.3 核心方法源码分析

### 3.3.1 addFirst / addLast（O(1)）

```java
// 在头部添加（O(1)，只修改指针）
public void addFirst(E e) {
    linkFirst(e);
}

private void linkFirst(E e) {
    final Node<E> f = first;
    // 创建新节点：prev=null（它是新头节点），item=e，next=旧头节点
    final Node<E> newNode = new Node<>(null, e, f);
    first = newNode;
    if (f == null)
        last = newNode; // 链表原来为空，尾节点也指向新节点
    else
        f.prev = newNode; // 旧头节点的前驱指向新节点
    size++;
    modCount++;
}

// 在尾部添加（O(1)，只修改指针）
public void addLast(E e) {
    linkLast(e);
}

void linkLast(E e) {
    final Node<E> l = last;
    // 创建新节点：prev=旧尾节点，item=e，next=null（它是新尾节点）
    final Node<E> newNode = new Node<>(l, e, null);
    last = newNode;
    if (l == null)
        first = newNode; // 链表原来为空，头节点也指向新节点
    else
        l.next = newNode; // 旧尾节点的后继指向新节点
    size++;
    modCount++;
}
```

### 3.3.2 removeFirst / removeLast（O(1)）

```java
// 删除头节点（O(1)）
public E removeFirst() {
    final Node<E> f = first;
    if (f == null) throw new NoSuchElementException();
    return unlinkFirst(f);
}

private E unlinkFirst(Node<E> f) {
    final E element = f.item;
    final Node<E> next = f.next;
    f.item = null; // 清除引用（帮助 GC）
    f.next = null;
    first = next;
    if (next == null)
        last = null;  // 链表变空
    else
        next.prev = null; // 新头节点的前驱置为 null
    size--;
    modCount++;
    return element;
}

// 删除尾节点（O(1)）
public E removeLast() {
    final Node<E> l = last;
    if (l == null) throw new NoSuchElementException();
    return unlinkLast(l);
}

private E unlinkLast(Node<E> l) {
    final E element = l.item;
    final Node<E> prev = l.prev;
    l.item = null;
    l.prev = null;
    last = prev;
    if (prev == null)
        first = null;
    else
        prev.next = null; // 新尾节点的后继置为 null
    size--;
    modCount++;
    return element;
}
```

### 3.3.3 get 方法（二分优化，O(n)）

```java
// 按索引获取（O(n)，但有二分优化）
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
}

// 核心：根据索引定位节点（二分优化）
Node<E> node(int index) {
    // 如果 index 在前半段，从头节点开始向后遍历
    if (index < (size >> 1)) {
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    }
    // 如果 index 在后半段，从尾节点开始向前遍历
    else {
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}

// 示意图：size=10，访问 index=7
// index(7) >= size/2(5)，从尾部开始
// last → 9 → 8 → 7（只需遍历2步）
// 如果从头开始需要7步
// 最坏情况（index = size/2）：遍历 n/4 步，仍然是 O(n)
```

### 3.3.4 在指定位置插入

```java
// add(index, element)：O(n)（先找到节点，再 O(1) 插入）
public void add(int index, E element) {
    checkPositionIndex(index);

    if (index == size)
        linkLast(element); // 尾部插入，O(1)
    else
        linkBefore(element, node(index)); // 先 O(n) 找到节点，再 O(1) 插入
}

// 在节点 succ 之前插入新节点
void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
    modCount++;
}
```

## 3.4 LinkedList vs ArrayDeque 作为队列/栈

**结论：推荐使用 `ArrayDeque` 代替 `LinkedList` 作为队列和栈。**

**原因分析：**

1. **内存效率**：`ArrayDeque` 底层是循环数组，不需要为每个元素创建 `Node` 对象
   - LinkedList：每个元素 = 对象头(16B) + prev指针(8B) + item引用(8B) + next指针(8B) = 40B+
   - ArrayDeque：每个元素 ≈ 一个引用(8B)

2. **CPU 缓存友好**：数组的连续内存布局对 CPU 缓存行更友好，减少缓存 miss

3. **GC 压力**：LinkedList 每次入队创建 Node 对象，GC 压力更大

4. **性能数据**（仅供参考，实际因 JVM 而异）：

```java
import java.util.*;

public class ArrayDequeVsLinkedList {
    public static void main(String[] args) {
        int N = 1_000_000;

        // ArrayDeque 作为栈
        Deque<Integer> arrayDeque = new ArrayDeque<>();
        long start = System.nanoTime();
        for (int i = 0; i < N; i++) arrayDeque.push(i);
        for (int i = 0; i < N; i++) arrayDeque.pop();
        long time1 = (System.nanoTime() - start) / 1_000_000;

        // LinkedList 作为栈
        Deque<Integer> linkedList = new LinkedList<>();
        start = System.nanoTime();
        for (int i = 0; i < N; i++) linkedList.push(i);
        for (int i = 0; i < N; i++) linkedList.pop();
        long time2 = (System.nanoTime() - start) / 1_000_000;

        System.out.printf("ArrayDeque:  %d ms%n", time1);
        System.out.printf("LinkedList:  %d ms%n", time2);
        // ArrayDeque 通常比 LinkedList 快 20%~50%

        // ArrayDeque 作为队列（FIFO）
        Queue<Integer> dq = new ArrayDeque<>();
        dq.offer(1); dq.offer(2); dq.offer(3);
        System.out.println(dq.poll()); // 1（FIFO）

        // 注意：ArrayDeque 不支持 null 元素
        try {
            dq.offer(null); // NullPointerException!
        } catch (NullPointerException e) {
            System.out.println("ArrayDeque 不支持 null 元素");
        }
    }
}
```

| 特性         | LinkedList                | ArrayDeque                  |
|--------------|---------------------------|-----------------------------|
| 底层结构     | 双向链表                  | 循环数组（head/tail指针）   |
| 内存消耗     | 高（每节点额外开销）      | 低（连续数组）              |
| 随机访问     | O(n)                      | O(1)                        |
| 头尾操作     | O(1)                      | O(1) 均摊                   |
| 中间插入     | O(n)找位置 + O(1)插入     | O(n)                        |
| null 元素    | 支持                      | 不支持                      |
| 推荐用途     | 作为 List 使用/中间插入   | 队列/栈/双端队列            |
| 实现接口     | List + Deque              | Deque                       |


---

# Part 4: HashMap 深度解析（重点）

## 4.1 JDK 7 vs JDK 8 实现对比

| 对比项       | JDK 7                          | JDK 8                              |
|--------------|--------------------------------|------------------------------------|
| 数据结构     | 数组 + 链表                    | 数组 + 链表 + 红黑树               |
| 链表插入     | 头插法（新节点插到链表头部）   | 尾插法（新节点追加到链表尾部）     |
| 扩容时链表   | 重新哈希，头插法（会死循环）   | 高低位拆分（保持顺序，无死循环）   |
| 哈希函数     | 9次右移和异或                  | 1次右移16位异或（更简洁）          |
| 树化         | 无                             | 链表长度>8 且 数组长度>=64 时树化  |
| 并发问题     | 死循环（多线程扩容）           | 数据丢失（多线程 put）             |

## 4.2 JDK 8 数据结构

```
HashMap 内部结构（JDK 8）：

table 数组（初始长度16，2的幂次方）：
+-------+-------+-------+-------+-------+-------+-------+-------+
|  [0]  |  [1]  |  [2]  |  [3]  |  [4]  |  [5]  |  [6]  |  [7]  |
+---+---+-------+-------+---+---+-------+---+---+-------+-------+
    |                       |               |
    | (哈希碰撞，形成链表)   |               | (哈希碰撞，已树化)
    v                       v               v
 [K1,V1]                 [K4,V4]        [K7,V7]    <- 红黑树根节点
    |                       |            /     \
 [K2,V2]                 [K5,V5]    [K8,V8]   [K9,V9]
    |                                  |
 [K3,V3]                           [K10,V10]
    |
   null                  <- 链表长度<=8

链表节点（Node）：
+---------+---------+---------+---------+
|  hash   |   key   |  value  |  next   |
| (int)   | (K ref) | (V ref) | (Node)  |
+---------+---------+---------+---------+

红黑树节点（TreeNode extends Node，增加了树的指针）：
+--------+------+-------+------+------+-------+--------+------+
|  hash  |  key | value | next | left | right | parent |  red |
+--------+------+-------+------+------+-------+--------+------+
```

## 4.3 核心参数详解

```java
public class HashMap<K,V> extends AbstractMap<K,V>
        implements Map<K,V>, Cloneable, Serializable {

    // 默认初始容量（必须是 2 的幂）
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 16

    // 最大容量（2^30 约10亿）
    static final int MAXIMUM_CAPACITY = 1 << 30;

    // 默认负载因子（time-space tradeoff 的最优值，见4.8节）
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    // 链表转红黑树的阈值（链表节点数 >= 8 时考虑树化）
    static final int TREEIFY_THRESHOLD = 8;

    // 红黑树退化为链表的阈值（树节点数 <= 6 时退化）
    // 设为6而非8：留2的缓冲区，防止频繁转换（见4.9节）
    static final int UNTREEIFY_THRESHOLD = 6;

    // 树化时数组的最小长度（数组长度 < 64 时优先扩容而非树化）
    // 原因：数组小时，扩容能更有效地分散元素，避免不必要的树化
    static final int MIN_TREEIFY_CAPACITY = 64;

    transient Node<K,V>[] table; // 哈希桶数组（懒初始化，第一次 put 时分配）
    transient Set<Map.Entry<K,V>> entrySet; // entrySet 缓存
    transient int size;          // 实际键值对数量
    transient int modCount;      // 结构修改计数（fail-fast 用）
    int threshold;               // 下次扩容的阈值 = capacity * loadFactor
    final float loadFactor;      // 负载因子
}
```

## 4.4 put 流程详解（源码级分析）

### 4.4.1 哈希函数设计

```java
// HashMap 的哈希扰动函数
static final int hash(Object key) {
    int h;
    // null 的 hash 为 0，存放在 table[0]
    // 非 null：h = key.hashCode()，然后 h XOR (h 右移16位)
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// === 设计原理 ===
// 问题：直接使用 hashCode 的低位，高位信息被忽略
//
// 例：hashCode = 0xABCD1234，容量 n=16
// (n-1) & hashCode = 0b1111 & 0xABCD1234
//                  = 0b1111 & 0b...0001 0010 0011 0100
//                  = 0b0100 = 4（只用了最低4位！）
// 高位 0xABCD 完全被忽略，高位碰撞严重时性能下降
//
// 解决方案：h ^ (h >>> 16)（高16位与低16位异或）
// hashCode:    1010 1011 1100 1101 | 0001 0010 0011 0100  (32位)
// h >>> 16:    0000 0000 0000 0000 | 1010 1011 1100 1101  (右移16位)
// XOR result:  1010 1011 1100 1101 | 1011 1001 1111 1001  (高位混入低位)
// 现在低16位包含了高16位的信息，减少碰撞

// === 为什么只右移16位而不是更多次？ ===
// JDK 7 的哈希函数做了4次扰动（更复杂），JDK 8 简化为1次
// 理由：统计分析表明1次扰动已足够，额外开销不值得
```

### 4.4.2 确定桶位置

```java
// 桶位置计算：(n-1) & hash
// n 必须是 2 的幂次方

// 原理：n=2^k 时，(n-1) 的二进制表示为 k 个 1
// 例：n=16=0b10000，n-1=15=0b01111
// (n-1) & hash 取 hash 的低 k 位 = hash % n（等价，但位运算更快）

// 验证等价性（n=16时）：
// hash=65=0b01000001
// hash % 16 = 65 % 16 = 1
// (n-1) & hash = 0b01111 & 0b01000001 = 0b00000001 = 1  ✓

// 为什么必须是 2 的幂？
// 如果 n=10（非2的幂），n-1=9=0b01001，不是全1的掩码
// 0b01001 & hash：某些位位置永远是0，对应的桶永远空置（浪费空间）
// 例：& 0b01001 的结果只可能是 0,1,8,9（4种），其他桶永远空！
```

### 4.4.3 完整 put 源码注解

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// onlyIfAbsent: true=只有key不存在时才插入（putIfAbsent用）
// evict: false=创建模式（LinkedHashMap 用来维护链表顺序）
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
    Node<K,V>[] tab;
    Node<K,V> p;
    int n, i;

    // 步骤1：懒初始化，第一次 put 时通过 resize() 初始化 table
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length; // 初始化为容量16

    // 步骤2：计算桶位置，如果该桶为空，直接创建节点放入
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);

    else {
        // 步骤3：桶不为空，需要处理哈希碰撞
        Node<K,V> e;
        K k;

        // 情况3a：桶的头节点就是目标（hash 相同且 key 相等）
        // 先比较 hash（便宜），再比较 key（可能昂贵）
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p; // 找到了已有节点

        // 情况3b：桶的头节点是红黑树节点（链表已树化）
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);

        // 情况3c：桶的头节点是链表节点，遍历链表
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    // 遍历到末尾，没找到相同 key，尾插新节点（JDK 8 尾插法）
                    p.next = newNode(hash, key, value, null);
                    // 链表长度达到阈值（binCount从0开始，binCount=7时已是第8个节点）
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash); // 尝试树化（内部还会检查数组长度>=64）
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break; // 找到了相同 key
                p = e; // 继续遍历
            }
        }

        // 步骤4：e != null 表示找到了已有的 key，更新其 value
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value; // 更新 value
            afterNodeAccess(e); // LinkedHashMap 钩子：将节点移到链表末尾
            return oldValue;    // 返回旧 value（区别于新增返回 null）
        }
    }

    // 步骤5：新增了节点
    ++modCount;
    if (++size > threshold) // 元素数超过阈值，触发扩容
        resize();
    afterNodeInsertion(evict); // LinkedHashMap 钩子：可能删除最老节点（LRU）
    return null; // 新增时返回 null
}
```

**treeifyBin 树化逻辑：**

```java
final void treeifyBin(Node<K,V>[] tab, int hash) {
    int n, index;
    Node<K,V> e;
    // 重要：数组长度 < MIN_TREEIFY_CAPACITY(64) 时，优先扩容而非树化
    // 因为数组小时，扩容能更均匀地分散元素，避免不必要的树化
    if (tab == null || (n = tab.length) < MIN_TREEIFY_CAPACITY)
        resize(); // 扩容（让元素重新分布）
    else if ((e = tab[index = (n - 1) & hash]) != null) {
        // 将链表节点转换为 TreeNode 节点，然后构建红黑树
        TreeNode<K,V> hd = null, tl = null;
        do {
            TreeNode<K,V> p = replacementTreeNode(e, null);
            if (tl == null) hd = p;
            else { p.prev = tl; tl.next = p; }
            tl = p;
        } while ((e = e.next) != null);
        if ((tab[index] = hd) != null)
            hd.treeify(tab); // 调用 TreeNode.treeify() 构建红黑树
    }
}
```

## 4.5 resize 扩容机制（JDK 8 关键优化）

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;

    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            // 已达到最大容量，不再扩容，直接将阈值设为最大值
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 正常扩容：新容量 = 旧容量 * 2，新阈值 = 旧阈值 * 2
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1;
    }
    else if (oldThr > 0)
        newCap = oldThr; // 有初始阈值（指定了初始容量），用作新容量
    else {
        // 首次初始化（默认构造器）
        newCap = DEFAULT_INITIAL_CAPACITY; // 16
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY); // 12
    }

    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;

    // 创建新数组（newCap = oldCap * 2）
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;

    // 迁移旧数组中的元素到新数组
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null; // 清除旧引用（帮助 GC）

                if (e.next == null)
                    // 链表只有1个节点，直接重新哈希放入新数组
                    newTab[e.hash & (newCap - 1)] = e;

                else if (e instanceof TreeNode)
                    // 红黑树节点，调用 split 方法拆分树
                    // 如果拆分后节点数 <= UNTREEIFY_THRESHOLD(6)，退化为链表
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);

                else {
                    // === JDK 8 关键优化：链表高低位拆分 ===
                    // 不需要重新计算 hash！只需判断 (e.hash & oldCap) 是 0 还是 1
                    Node<K,V> loHead = null, loTail = null; // 低位链表（新位置 = 旧位置 j）
                    Node<K,V> hiHead = null, hiTail = null; // 高位链表（新位置 = j + oldCap）
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            // 扩容新增的高位 bit 为 0 → 新位置不变
                            if (loTail == null) loHead = e;
                            else loTail.next = e;
                            loTail = e;
                        } else {
                            // 扩容新增的高位 bit 为 1 → 新位置 = 旧位置 + oldCap
                            if (hiTail == null) hiHead = e;
                            else hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);

                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;           // 低位链表放原位置
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;  // 高位链表放原位置 + oldCap
                    }
                    // 注意：尾插法保持了链表原有顺序（不会产生死循环）
                }
            }
        }
    }
    return newTab;
}
```

**高低位拆分原理图解：**

```
扩容前：oldCap = 8，某个桶有3个节点（位置 j=5）
hash A = 0b...0_101（低3位=5，bit3=0 → 低位）
hash B = 0b...1_101（低3位=5，bit3=1 → 高位）
hash C = 0b...0_101（低3位=5，bit3=0 → 低位）

链表：A → B → C → null

扩容后：newCap = 16

判断 (hash & oldCap)，即 (hash & 8 = 0b1000)：
A: 0b...0_101 & 0b1000 = 0 → 低位，新位置 = 5
B: 0b...1_101 & 0b1000 = 8 → 高位，新位置 = 5 + 8 = 13
C: 0b...0_101 & 0b1000 = 0 → 低位，新位置 = 5

结果：
newTab[5]  → A → C → null（低位链表，顺序保持）
newTab[13] → B → null（高位链表）

与 JDK 7 对比：
JDK 7 头插法 + 重新 hash：
  扩容后 A,C 在同一桶，但顺序变为 C → A（头插法反转）
  多线程下这种反转会导致循环引用（死循环）
JDK 8 尾插法 + 高低位拆分：
  顺序保持（A → C），安全
```

## 4.6 get 流程（O(1) 均摊）

```java
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}

final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab;
    Node<K,V> first, e;
    int n;
    K k;

    // 先检查 table 不为空且对应桶不为空
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {

        // 检查桶头节点（大多数情况直接命中，O(1)）
        if (first.hash == hash &&
            ((k = first.key) == key || (key != null && key.equals(k))))
            return first;

        if ((e = first.next) != null) {
            // 红黑树查找（O(log n)）
            if (first instanceof TreeNode)
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);

            // 链表查找（O(k)，k 为链表长度，通常很短）
            do {
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

## 4.7 为什么容量必须是 2 的幂

```
理由1：位运算取模，比取模运算快
  普通取模：hash % n（需要除法指令）
  位运算：  (n-1) & hash（仅需与运算，快约5~10倍）
  等价条件：n 是 2 的幂次方时，hash % n == (n-1) & hash

理由2：扩容时高低位拆分（不需要重新计算 hash）
  只需判断 (hash & oldCap) 是 0 还是非0（见4.5节）
  这只对 n 是 2 的幂次方时成立

理由3：元素分布更均匀
  n=2^k 时，(n-1) 低k位全为1，& 运算能均匀利用 hash 的低k位
  如果 n 不是 2 的幂，(n-1) 中有 0 的位置，对应的桶永远空置

保证容量是 2 的幂的算法（tableSizeFor）：
```

```java
// 找到 >= cap 的最小 2 的幂
static final int tableSizeFor(int cap) {
    // 为什么先 cap-1？如果 cap 本身就是 2 的幂，直接 cap 即可
    // 不减1的话，tableSizeFor(16) = 32（错误，应该返回16）
    int n = cap - 1;
    // 将最高位1右侧的所有位都变为1
    n |= n >>> 1;  // 处理最高位和次高位
    n |= n >>> 2;  // 处理前4位
    n |= n >>> 4;  // 处理前8位
    n |= n >>> 8;  // 处理前16位
    n |= n >>> 16; // 处理前32位
    // n 现在是全1的低位掩码，n+1 就是下一个 2 的幂
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

// 示例演示：tableSizeFor(13)
// cap = 13，cap-1 = 12 = 0b00001100
// |= >>>1  : 0b00001100 | 0b00000110 = 0b00001110
// |= >>>2  : 0b00001110 | 0b00000011 = 0b00001111
// |= >>>4  : 0b00001111 | 0b00000000 = 0b00001111
// ...（后续没变化）
// n = 0b00001111 = 15，n+1 = 16 = 0b00010000 ✓（16 是大于13的最小2的幂）
```

## 4.8 为什么负载因子是 0.75

```
理论依据：泊松分布分析

假设 hashCode 服从均匀分布，桶中链表长度 k 服从泊松分布：
P(k) = (λ^k / k!) * e^(-λ)
其中 λ = 期望负载因子 = n/capacity

当负载因子为 0.75 时（λ ≈ 0.5 的泊松分布，来自 HashMap 源码注释）：
k=0: 0.60653066 （约60%的桶为空）
k=1: 0.30326533 （约30%的桶有1个元素）
k=2: 0.07581633 （约7.5%的桶有2个元素）
k=3: 0.01263606 （约1.3%）
k=4: 0.00157952 （约0.16%）
k=5: 0.00015795 （约0.016%）
k=6: 0.00001316
k=7: 0.00000094
k=8: 0.00000006 （0.000006%，即1600万次中才遇到1次！）

结论：
- 负载因子0.75时，链表长度达到8的概率极低（约 6e-8），几乎不会触发树化
- 0.75 是时间和空间的最优平衡点：
  * 太小（如0.5）：扩容频繁，空间浪费多，时间开销大
  * 太大（如1.0）：碰撞多，链表长，查找性能下降，接近 O(n)
```

## 4.9 为什么树化阈值是 8，退树化阈值是 6

```
树化阈值 = 8，退树化阈值 = 6（差值为2）：

设计原因一：概率极低（见4.8节）
  正常情况下（hashCode 均匀分布），链表很少能长到8，基本不会树化

设计原因二：性能权衡
  链表查找：O(n)，常数因子小（结构简单，内存连续）
  红黑树查找：O(log n)，常数因子大（结构复杂，涉及旋转/比较）
  n=8 时：链表8步 vs 红黑树3步（log2(8)=3），树的优势开始显现

设计原因三：缓冲区防止频繁转换
  如果树化和退树化阈值相同（都是8）：
    8个元素 → 树化
    删除1个 → 7个元素 → 退树化（链表）
    再插入1个 → 8个元素 → 树化
    ... 来回转换，开销大

  差值为2（8和6）：
    必须先降到6才退树化，有2个元素的缓冲区
    避免在7/8元素时反复转换，节省性能开销

链表 vs 红黑树的实际内存对比：
  链表节点（Node）：24B（hash 4B + key引用 8B + value引用 8B + next引用 8B，不含对象头16B）
  树节点（TreeNode）：约48B（额外的 left/right/parent/red 字段，约24B额外）
  树化会增加约2倍的内存开销，因此默认情况下能不树化就不树化
```

## 4.10 HashMap 线程不安全

### 4.10.1 JDK 7：头插法导致死循环

在 JDK 7 中，多线程扩容时，链表采用头插法，会将链表反转，多线程下可能形成环形链表，导致 `get` 时 CPU 100%。

```
初始状态：桶j → A → B → null

Thread-1 开始扩容，处理节点 A 时被抢占：
  e = A，next = B
  （Thread-1 暂停）

Thread-2 完成整个扩容（头插法导致链表反转）：
  新数组：A.next=null, B.next=A（反转！）
  newTab[j] → B → A → null

Thread-1 继续：
  (e=A)，(next=B)
  头插：newTab[j] → A → B → A → ... （环！）
  next = e.next = B
  (e=B，next=B.next=A)
  头插：newTab[j] → B → A → ... （A.next=B，B.next=A，成环！）

此后调用 get()，遍历链表时陷入死循环，CPU 100%
```

### 4.10.2 JDK 8：数据丢失

JDK 8 改用尾插法解决了死循环，但多线程 put 仍有数据丢失：

```java
// putVal 中的非原子操作（判断为null 到 赋值 之间没有锁保护）
if ((p = tab[i = (n - 1) & hash]) == null)  // 检查桶是否为空
    tab[i] = newNode(hash, key, value, null); // 赋值（非原子）

// Thread-1 执行到判断 tab[i]==null，得到 true，准备赋值
// Thread-2 也执行到判断 tab[i]==null，得到 true，也准备赋值
// Thread-1 赋值 tab[i] = Node1
// Thread-2 赋值 tab[i] = Node2（覆盖了 Node1！）
// Thread-1 的数据丢失！
```

### 4.10.3 线程安全替代方案

```java
import java.util.*;
import java.util.concurrent.*;

public class ThreadSafeMapDemo {
    public static void main(String[] args) {

        // 方案1：ConcurrentHashMap（强烈推荐）
        // JDK 8 使用 CAS + synchronized，细粒度锁，高并发下性能最好
        ConcurrentHashMap<String, Integer> chm = new ConcurrentHashMap<>();
        chm.put("key", 1);
        chm.putIfAbsent("key2", 2);

        // 方案2：Collections.synchronizedMap（包装器，整个 Map 对象级别加锁）
        // 所有操作互斥，并发性能差，但可以快速包装任意 Map
        Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
        // 注意：遍历时仍需手动加锁！
        synchronized (syncMap) {
            for (Map.Entry<String, Integer> e : syncMap.entrySet()) {
                // 安全遍历
            }
        }

        // 方案3：Hashtable（已过时，不推荐）
        // JDK 1.0 时代产物，方法级 synchronized，不支持 null key/value
        Hashtable<String, Integer> hashtable = new Hashtable<>();
        hashtable.put("key", 1);
        // hashtable.put(null, 1); // NullPointerException!
    }
}
```

## 4.11 HashMap key 为 null 的处理

```java
// HashMap 允许 key 为 null（hash=0，存放在 table[0]）
// Hashtable、ConcurrentHashMap 不允许 null key

// hash(null) = 0
// 所以 null key 永远存放在 table[0] 对应的桶中

HashMap<String, Integer> map = new HashMap<>();
map.put(null, 100);        // 合法，存入 table[0]
map.put("a", null);        // 合法，value 可以为 null
System.out.println(map.get(null)); // 100
System.out.println(map.get("b")); // null（不存在）

// 如何区分 "key不存在" 和 "key存在但value是null"？
System.out.println(map.containsKey(null)); // true
System.out.println(map.containsKey("b")); // false

// null value 的陷阱
map.put("c", null);
// get("c") == null 和 key "c" 不存在时结果相同！
// 务必使用 containsKey 来区分
```

## 4.12 完整使用示例

```java
import java.util.*;

public class HashMapCompleteDemo {
    public static void main(String[] args) {
        // === 基本操作 ===
        Map<String, Integer> map = new HashMap<>();
        map.put("Alice", 95); map.put("Bob", 87); map.put("Charlie", 92);
        map.put("Alice", 98); // 更新：返回旧值95，存入新值98

        System.out.println(map.get("Alice"));                    // 98
        System.out.println(map.getOrDefault("Dave", 0));         // 0（不存在时返回默认值）
        System.out.println(map.containsKey("Bob"));              // true
        System.out.println(map.containsValue(100));              // false

        // === Java 8 新方法 ===
        map.putIfAbsent("Dave", 80);       // Dave 不存在，插入80
        map.putIfAbsent("Alice", 70);      // Alice 已存在，不更新

        // computeIfAbsent：key不存在时通过函数计算value并插入
        // 常用于"Map作为缓存"的场景（如多值Map）
        Map<String, List<Integer>> multiMap = new HashMap<>();
        multiMap.computeIfAbsent("scores", k -> new ArrayList<>()).add(95);
        multiMap.computeIfAbsent("scores", k -> new ArrayList<>()).add(87);
        System.out.println(multiMap); // {scores=[95, 87]}

        // computeIfPresent：key存在时更新value
        map.computeIfPresent("Bob", (k, v) -> v + 10); // 87 → 97

        // merge：key不存在则插入，key存在则用函数合并旧值和新值
        map.merge("Alice", 2, Integer::sum);  // 已存在：98 + 2 = 100
        map.merge("Frank", 75, Integer::sum); // 不存在：直接插入75

        // replaceAll：对所有entry应用函数
        map.replaceAll((k, v) -> v + 5); // 所有分数+5

        // === 遍历方式（推荐 entrySet）===
        // 方式1：entrySet（推荐，只遍历一次）
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            System.out.printf("  %s: %d%n", entry.getKey(), entry.getValue());
        }

        // 方式2：keySet + get（不推荐，每次 get 都要再哈希查找一次）
        for (String key : map.keySet()) {
            System.out.println(key + ": " + map.get(key));
        }

        // 方式3：Java 8 forEach Lambda
        map.forEach((k, v) -> System.out.printf("%s -> %d%n", k, v));

        // 方式4：Stream API（需要过滤/排序时）
        map.entrySet().stream()
           .filter(e -> e.getValue() > 90)
           .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
           .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));

        // === 统计词频（经典场景）===
        String[] words = {"apple", "banana", "apple", "cherry", "banana", "apple"};
        Map<String, Integer> freq = new HashMap<>();
        for (String word : words) {
            freq.merge(word, 1, Integer::sum); // 优雅地统计词频
        }
        System.out.println(freq); // {apple=3, banana=2, cherry=1}

        // 等价的传统写法（更冗长）
        Map<String, Integer> freq2 = new HashMap<>();
        for (String word : words) {
            freq2.put(word, freq2.getOrDefault(word, 0) + 1);
        }

        // === 分组（按某个属性分组）===
        List<String> names = Arrays.asList("Alice", "Bob", "Anna", "Charlie", "Ben");
        Map<Character, List<String>> byFirstLetter = new HashMap<>();
        for (String name : names) {
            byFirstLetter.computeIfAbsent(name.charAt(0), k -> new ArrayList<>()).add(name);
        }
        System.out.println(byFirstLetter);
        // {A=[Alice, Anna], B=[Bob, Ben], C=[Charlie]}
    }
}
```


---

# Part 5: LinkedHashMap 深度解析

## 5.1 数据结构：HashMap + 双向链表

`LinkedHashMap` 继承 `HashMap`，在 HashMap 的哈希桶基础上**额外维护一个双向链表**，按照插入顺序（或访问顺序）串联所有节点。

```
LinkedHashMap 内部结构：

HashMap 的哈希桶（数组）：
+------+------+------+------+------+------+------+------+
| [0]  | [1]  | [2]  | [3]  | [4]  | [5]  | [6]  | [7]  |
+--+---+------+--+---+------+--+---+------+------+------+
   |              |              |
   v              v              v
 [A,1]          [C,3]          [E,5]
   |
 [D,4]

额外的双向链表（记录插入顺序 A → C → D → E）：
head <-> [A,1] <-> [C,3] <-> [D,4] <-> [E,5] <-> tail
  (虚拟哨兵节点)                                   (虚拟哨兵节点)

LinkedHashMap.Entry 结构（继承 HashMap.Node，额外增加 before/after）：
+--------+------+-------+------+--------+-------+
|  hash  |  key | value | next | before | after |
+--------+------+-------+------+--------+-------+
                        (HashMap链表)  (双向链表)
```

```java
// LinkedHashMap 核心字段
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V> {

    transient LinkedHashMap.Entry<K,V> head; // 双向链表的头节点（最旧的节点）
    transient LinkedHashMap.Entry<K,V> tail; // 双向链表的尾节点（最新的节点）

    // 顺序模式：
    // false（默认）：维护插入顺序（按照 put 的先后顺序）
    // true：维护访问顺序（每次 get/put 将节点移到尾部，适合 LRU）
    final boolean accessOrder;

    // Entry 继承 HashMap.Node，额外维护双向链表指针
    static class Entry<K,V> extends HashMap.Node<K,V> {
        Entry<K,V> before, after; // 双向链表的前驱和后继
        Entry(int hash, K key, V value, Node<K,V> next) {
            super(hash, key, value, next);
        }
    }
}
```

## 5.2 插入顺序 vs 访问顺序

```java
import java.util.*;

public class LinkedHashMapOrderDemo {
    public static void main(String[] args) {
        // === 插入顺序（默认，accessOrder=false）===
        Map<String, Integer> insertOrder = new LinkedHashMap<>();
        insertOrder.put("C", 3);
        insertOrder.put("A", 1);
        insertOrder.put("B", 2);
        System.out.println("插入顺序: " + insertOrder); // {C=3, A=1, B=2}

        insertOrder.get("C"); // 访问操作不影响顺序
        System.out.println("访问C后: " + insertOrder);  // {C=3, A=1, B=2}

        insertOrder.put("A", 99); // 更新操作也不影响顺序（A 仍在第2位）
        System.out.println("更新A后: " + insertOrder);  // {C=3, A=99, B=2}

        // === 访问顺序（accessOrder=true，LRU 核心机制）===
        // 构造器：(initialCapacity, loadFactor, accessOrder)
        Map<String, Integer> accessOrder = new LinkedHashMap<>(16, 0.75f, true);
        accessOrder.put("C", 3);
        accessOrder.put("A", 1);
        accessOrder.put("B", 2);
        System.out.println("\n初始: " + accessOrder);   // {C=3, A=1, B=2}

        accessOrder.get("C"); // 访问C → C 移到链表尾部（最近使用）
        System.out.println("访问C后: " + accessOrder); // {A=1, B=2, C=3}

        accessOrder.get("A"); // 访问A → A 移到尾部
        System.out.println("访问A后: " + accessOrder); // {B=2, C=3, A=1}

        accessOrder.put("D", 4); // 新插入 → 放到尾部
        System.out.println("插入D后: " + accessOrder); // {B=2, C=3, A=1, D=4}

        accessOrder.put("B", 99); // 更新B → B 移到尾部
        System.out.println("更新B后: " + accessOrder); // {C=3, A=1, D=4, B=99}

        // 链表头部（head.after）是"最久未使用"的节点
        // 链表尾部（tail）是"最近使用"的节点
        // removeEldestEntry 删除的就是 head.after（最旧的节点）
    }
}
```

## 5.3 afterNodeAccess 源码：访问后移到末尾

```java
// 当 accessOrder=true 时，每次 get/put 成功访问一个节点，都会调用此方法
// 将该节点从当前位置移动到双向链表的末尾（表示"最近使用"）
void afterNodeAccess(Node<K,V> e) {
    LinkedHashMap.Entry<K,V> last;
    // 只在 accessOrder=true 且 e 不已经是尾节点时处理
    if (accessOrder && (last = tail) != e) {
        LinkedHashMap.Entry<K,V> p = (LinkedHashMap.Entry<K,V>)e;
        LinkedHashMap.Entry<K,V> b = p.before; // p 的前驱
        LinkedHashMap.Entry<K,V> a = p.after;  // p 的后继

        p.after = null; // p 将成为新尾节点，after 置 null

        // 从当前位置摘除 p
        if (b == null)
            head = a; // p 原来是头节点，a 成为新头节点
        else
            b.after = a;

        if (a != null)
            a.before = b;
        else
            last = b; // a 为 null 说明 p 已经是尾节点（前面条件已排除）

        // 将 p 追加到末尾
        if (last == null)
            head = p; // 链表为空
        else {
            p.before = last;
            last.after = p;
        }
        tail = p;
        ++modCount;
    }
}
```

## 5.4 afterNodeInsertion：删除最老节点（LRU 的关键）

```java
// 每次插入新节点后调用（evict=true 表示处于正常运行模式，非初始化）
void afterNodeInsertion(boolean evict) {
    LinkedHashMap.Entry<K,V> first;
    // removeEldestEntry 返回 true 时，删除链表头部节点（最老/最少使用的节点）
    if (evict && (first = head) != null && removeEldestEntry(first)) {
        K key = first.key;
        removeNode(hash(key), key, null, false, true);
    }
}

// 默认实现：永远返回 false（不自动删除），子类覆盖此方法实现 LRU 缓存
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false;
}
```

## 5.5 LRU 缓存完整实现

```java
import java.util.*;
import java.util.concurrent.locks.*;

/**
 * 基于 LinkedHashMap 的 LRU 缓存
 *
 * LRU（Least Recently Used，最近最少使用）：
 * 当缓存满时，淘汰最近最久没有被访问过的数据
 *
 * 实现原理：
 * - accessOrder=true：每次访问将节点移到链表末尾
 * - head（链表头部）= 最久未访问的节点 = LRU 候选
 * - tail（链表尾部）= 最近访问的节点 = MRU（Most Recently Used）
 * - 重写 removeEldestEntry：容量超限时返回 true → 自动删除头部节点
 *
 * 时间复杂度：get O(1)，put O(1)
 * 空间复杂度：O(capacity)
 */
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        // initialCapacity 设大一点避免扩容
        // accessOrder=true：按访问顺序排列（get/put 都会更新顺序）
        super(capacity + 1, 1.0f, true);
        this.capacity = capacity;
    }

    /**
     * 重写 removeEldestEntry：当 size > capacity 时，自动删除最久未访问的节点
     * eldest 参数是链表头部节点（最久未访问）
     */
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity;
    }

    /**
     * 测试
     */
    public static void main(String[] args) {
        LRUCache<Integer, String> cache = new LRUCache<>(3);

        cache.put(1, "one");   // 缓存: {1=one}
        cache.put(2, "two");   // 缓存: {1=one, 2=two}
        cache.put(3, "three"); // 缓存: {1=one, 2=two, 3=three}
        System.out.println(cache); // {1=one, 2=two, 3=three}

        cache.get(1);          // 访问1，1移到尾部: {2=two, 3=three, 1=one}
        System.out.println(cache); // {2=two, 3=three, 1=one}

        cache.put(4, "four");  // 缓存满，删除最旧的2: {3=three, 1=one, 4=four}
        System.out.println(cache); // {3=three, 1=one, 4=four}

        System.out.println(cache.get(2)); // null（已被淘汰）
        System.out.println(cache.get(3)); // "three"，3移到尾部
        System.out.println(cache); // {1=one, 4=four, 3=three}

        cache.put(5, "five");  // 删除最旧的1
        System.out.println(cache); // {4=four, 3=three, 5=five}
    }
}
```

**LRU 缓存的 LeetCode 146 完整解法：**

```java
// LeetCode 146: LRU Cache
// 要求：get(key) 和 put(key, value) 都是 O(1)

import java.util.*;

class LRUCacheLeetCode {
    private final int capacity;
    private final LinkedHashMap<Integer, Integer> cache;

    public LRUCacheLeetCode(int capacity) {
        this.capacity = capacity;
        // 使用匿名内部类，重写 removeEldestEntry
        this.cache = new LinkedHashMap<Integer, Integer>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
            }
        };
    }

    public int get(int key) {
        return cache.getOrDefault(key, -1);
    }

    public void put(int key, int value) {
        cache.put(key, value);
    }
}
```


---

# Part 6: TreeMap 深度解析

## 6.1 底层红黑树

`TreeMap` 底层使用**红黑树（Red-Black Tree）**，保证键的有序性，并提供 O(log n) 的 get/put/remove。

```java
public class TreeMap<K,V>
    extends AbstractMap<K,V>
    implements NavigableMap<K,V>, Cloneable, java.io.Serializable {

    // 比较器（null 表示使用 key 的自然排序 Comparable）
    private final Comparator<? super K> comparator;

    // 红黑树根节点（null 表示空树）
    private transient Entry<K,V> root;

    // 元素数量
    private transient int size = 0;

    static final boolean RED   = false;
    static final boolean BLACK = true;

    // 红黑树节点
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;    // 左子节点
        Entry<K,V> right;   // 右子节点
        Entry<K,V> parent;  // 父节点
        boolean color = BLACK; // 默认黑色
    }
}
```

## 6.2 红黑树五条性质

```
红黑树（Red-Black Tree）是一种自平衡二叉搜索树，定义如下：

性质1：每个节点是红色或黑色
性质2：根节点是黑色
性质3：每个叶子节点（NIL节点/null节点）是黑色
性质4：如果一个节点是红色，则它的两个子节点都是黑色
        （推论：从根到叶子的路径上，不能有连续的红节点）
性质5：从任意节点到其所有后代叶节点的路径，都包含相同数目的黑色节点
        （即：黑高相同/黑色完美平衡）

这5条性质保证：
  最长路径（红黑交替） <= 2 * 最短路径（全黑）
  因此 O(log n) 时间复杂度有保证

红黑树示例（B=黑，R=红）：

                   (B) 13
                 /         \
           (R) 8            (B) 17
          /     \           /     \
       (B) 1   (B) 11   (B) 15   (R) 25
          \        \              /     \
          (R)6   (R)12         (B)22   (B)27
                               /
                             (R)20

验证性质4：没有连续红节点 ✓
验证性质5：从根到所有 NIL 节点的黑高均为 3 ✓
```

## 6.3 红黑树旋转与变色

```
左旋（Left Rotation）：
  把节点 x 的右子节点 y 提升为 x 的父节点
  x 成为 y 的左子节点，y 的左子树 B 成为 x 的右子树

     x                      y
    / \       左旋(x)       / \
   A   y    ---------->   x   C
      / \                / \
     B   C              A   B

右旋（Right Rotation）：
  把节点 y 的左子节点 x 提升为 y 的父节点
  y 成为 x 的右子节点，x 的右子树 B 成为 y 的左子树

       y                    x
      / \     右旋(y)       / \
     x   C  ---------->  A   y
    / \                      / \
   A   B                    B   C

变色：将节点颜色从红变黑，或从黑变红

插入修复的3种情况（新节点总是红色）：

Case 1：叔叔节点是红色 → 变色后向上递归
         (B)G                  (R)G
         /   \     变色         /   \
      (R)P  (R)U  ------->  (B)P  (B)U
      /                     /
   (R)N                  (R)N
   （G变红，P和U变黑，然后以G为新节点向上递归）

Case 2：叔叔黑色，新节点是"内侧"（LR/RL型） → 先旋转转成 Case 3
       (B)G                    (B)G
       /   \    P左旋           /   \
    (R)P  (B)U  -------->   (R)N  (B)U
       \                    /
       (R)N              (R)P

Case 3：叔叔黑色，新节点是"外侧"（LL/RR型） → 旋转+变色
       (B)G                    (B)P
       /   \    右旋G           /   \
    (R)P  (B)U  -------->   (R)N  (R)G
    /                                \
 (R)N                               (B)U
 （G右旋，P变黑，G变红）
```

## 6.4 put 插入流程源码

```java
public V put(K key, V value) {
    Entry<K,V> t = root;

    // 空树：直接创建根节点
    if (t == null) {
        compare(key, key); // 类型/null检查
        root = new Entry<>(key, value, null);
        size = 1;
        modCount++;
        return null;
    }

    int cmp;
    Entry<K,V> parent;
    Comparator<? super K> cpr = comparator;

    // 阶段1：BST 标准插入，找到合适的叶子位置
    if (cpr != null) {
        // 使用自定义比较器
        do {
            parent = t;
            cmp = cpr.compare(key, t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value); // key 已存在，更新 value
        } while (t != null);
    } else {
        // 使用自然排序（key 必须实现 Comparable）
        if (key == null)
            throw new NullPointerException(); // TreeMap 不允许 null key！
        @SuppressWarnings("unchecked")
        Comparable<? super K> k = (Comparable<? super K>) key;
        do {
            parent = t;
            cmp = k.compareTo(t.key);
            if (cmp < 0) t = t.left;
            else if (cmp > 0) t = t.right;
            else return t.setValue(value);
        } while (t != null);
    }

    // 创建新节点并连接到树上
    Entry<K,V> e = new Entry<>(key, value, parent);
    if (cmp < 0) parent.left = e;
    else parent.right = e;

    // 阶段2：修复红黑树性质（插入后可能违反性质4）
    fixAfterInsertion(e);
    size++;
    modCount++;
    return null;
}

// 插入后修复红黑树性质
private void fixAfterInsertion(Entry<K,V> x) {
    x.color = RED; // 新节点设为红色（尽量减少黑高变化）

    // 父节点是红色时，违反性质4，需要修复
    while (x != null && x != root && x.parent.color == RED) {
        if (parentOf(x) == leftOf(parentOf(parentOf(x)))) {
            // 父节点是祖父节点的左子节点
            Entry<K,V> y = rightOf(parentOf(parentOf(x))); // 叔叔节点
            if (colorOf(y) == RED) {
                // Case 1：叔叔是红色，变色
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x)); // 向上递归
            } else {
                if (x == rightOf(parentOf(x))) {
                    // Case 2：内侧插入，先左旋转成 Case 3
                    x = parentOf(x);
                    rotateLeft(x);
                }
                // Case 3：外侧插入，右旋+变色
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateRight(parentOf(parentOf(x)));
            }
        } else {
            // 镜像情况（父节点是祖父节点的右子节点）
            Entry<K,V> y = leftOf(parentOf(parentOf(x)));
            if (colorOf(y) == RED) {
                setColor(parentOf(x), BLACK);
                setColor(y, BLACK);
                setColor(parentOf(parentOf(x)), RED);
                x = parentOf(parentOf(x));
            } else {
                if (x == leftOf(parentOf(x))) {
                    x = parentOf(x);
                    rotateRight(x);
                }
                setColor(parentOf(x), BLACK);
                setColor(parentOf(parentOf(x)), RED);
                rotateLeft(parentOf(parentOf(x)));
            }
        }
    }
    root.color = BLACK; // 性质2：根节点始终是黑色
}

// 左旋实现
private void rotateLeft(Entry<K,V> p) {
    if (p != null) {
        Entry<K,V> r = p.right; // r = p 的右子节点（将成为新父节点）
        p.right = r.left;       // r 的左子树 成为 p 的右子树
        if (r.left != null)
            r.left.parent = p;
        r.parent = p.parent;    // r 替代 p 的位置
        if (p.parent == null)
            root = r;
        else if (p.parent.left == p)
            p.parent.left = r;
        else
            p.parent.right = r;
        r.left = p;             // p 成为 r 的左子节点
        p.parent = r;
    }
}
```

## 6.5 NavigableMap 接口详解

`TreeMap` 实现了 `NavigableMap`，提供丰富的范围查询和导航方法：

```java
import java.util.*;

public class TreeMapNavigableDemo {
    public static void main(String[] args) {
        TreeMap<Integer, String> map = new TreeMap<>();
        for (int i : new int[]{1, 3, 5, 7, 9, 11, 13, 15}) {
            map.put(i, "v" + i);
        }
        System.out.println("原始: " + map.keySet());
        // [1, 3, 5, 7, 9, 11, 13, 15]

        // ===== 端点查询 =====
        System.out.println("最小键: " + map.firstKey());   // 1
        System.out.println("最大键: " + map.lastKey());    // 15
        System.out.println("最小条目: " + map.firstEntry()); // 1=v1
        System.out.println("最大条目: " + map.lastEntry());  // 15=v15

        // ===== 邻近查询（常用于查找区间）=====
        System.out.println("floorKey(6) = " + map.floorKey(6));     // 5（<= 6 的最大键）
        System.out.println("ceilingKey(6) = " + map.ceilingKey(6)); // 7（>= 6 的最小键）
        System.out.println("lowerKey(7) = " + map.lowerKey(7));     // 5（< 7 的最大键）
        System.out.println("higherKey(7) = " + map.higherKey(7));   // 9（> 7 的最小键）

        // 特殊情况
        System.out.println("floorKey(0) = " + map.floorKey(0));     // null（没有 <= 0 的键）
        System.out.println("ceilingKey(20) = " + map.ceilingKey(20)); // null

        // ===== 子映射视图（都是视图，修改影响原 TreeMap）=====
        // subMap(fromKey, toKey)：[fromKey, toKey)，左闭右开
        System.out.println("subMap(3,9) = " + map.subMap(3, 9).keySet());
        // [3, 5, 7]

        // subMap(fromKey, fromInclusive, toKey, toInclusive)：灵活控制端点
        System.out.println("subMap(3,true,9,true) = " + map.subMap(3, true, 9, true).keySet());
        // [3, 5, 7, 9]（包含两端）

        // headMap(toKey)：[最小键, toKey)
        System.out.println("headMap(7) = " + map.headMap(7).keySet());   // [1, 3, 5]
        // headMap(toKey, inclusive)
        System.out.println("headMap(7, true) = " + map.headMap(7, true).keySet()); // [1, 3, 5, 7]

        // tailMap(fromKey)：[fromKey, 最大键]
        System.out.println("tailMap(9) = " + map.tailMap(9).keySet());   // [9, 11, 13, 15]

        // ===== 逆序视图 =====
        System.out.println("逆序键集: " + map.descendingKeySet());
        // [15, 13, 11, 9, 7, 5, 3, 1]

        NavigableMap<Integer, String> descMap = map.descendingMap();
        System.out.println("逆序第一键: " + descMap.firstKey()); // 15

        // ===== 轮询（删除并返回）=====
        System.out.println("pollFirstEntry: " + map.pollFirstEntry()); // 1=v1（删除最小）
        System.out.println("pollLastEntry: " + map.pollLastEntry());   // 15=v15（删除最大）
        System.out.println("剩余: " + map.keySet()); // [3, 5, 7, 9, 11, 13]

        // ===== 自定义排序 =====
        // 按字符串长度排序，长度相同按字母序
        TreeMap<String, Integer> customMap = new TreeMap<>(
            Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder())
        );
        customMap.put("banana", 6);
        customMap.put("fig", 3);
        customMap.put("apple", 5);
        customMap.put("kiwi", 4);
        customMap.put("pear", 4);
        System.out.println(customMap.keySet());
        // [fig, kiwi, pear, apple, banana]（按长度排序，相同长度按字母）
    }
}
```

## 6.6 TreeMap 实际应用场景

```java
import java.util.*;

public class TreeMapApplicationDemo {
    // 应用1：区间查询（如时间线上的事件）
    static void timelineDemo() {
        TreeMap<Long, String> timeline = new TreeMap<>();
        timeline.put(1000L, "Event A");
        timeline.put(3000L, "Event B");
        timeline.put(5000L, "Event C");
        timeline.put(8000L, "Event D");

        long queryTime = 4000L;
        // 查找时间线上 queryTime 之前的最近事件
        Map.Entry<Long, String> prev = timeline.floorEntry(queryTime);
        // 查找时间线上 queryTime 之后的最近事件
        Map.Entry<Long, String> next = timeline.ceilingEntry(queryTime);
        System.out.println("查询时间: " + queryTime);
        System.out.println("之前事件: " + prev); // 3000=Event B
        System.out.println("之后事件: " + next); // 5000=Event C

        // 查找某个时间范围内的所有事件
        System.out.println("2000-6000之间: " + timeline.subMap(2000L, 6000L));
        // {3000=Event B, 5000=Event C}
    }

    // 应用2：实现计数器（统计分数段人数）
    static void scoreRangeDemo() {
        TreeMap<Integer, Integer> scoreCount = new TreeMap<>();
        int[] scores = {95, 87, 92, 78, 95, 87, 65, 92, 78, 99};
        for (int s : scores) {
            scoreCount.merge(s, 1, Integer::sum);
        }
        System.out.println("分数分布: " + scoreCount);

        // 查询 [80, 95] 分段的人数
        int count = scoreCount.subMap(80, true, 95, true)
                              .values().stream()
                              .mapToInt(Integer::intValue).sum();
        System.out.println("80-95分人数: " + count);
    }

    public static void main(String[] args) {
        timelineDemo();
        scoreRangeDemo();
    }
}
```

## 6.7 TreeMap vs HashMap vs LinkedHashMap 对比

| 特性          | HashMap                  | LinkedHashMap              | TreeMap                    |
|---------------|--------------------------|----------------------------|----------------------------|
| 底层数据结构  | 数组 + 链表 + 红黑树     | 同左 + 双向链表            | 纯红黑树                   |
| 有序性        | 无序                     | 插入/访问顺序              | 键的自然/自定义排序         |
| null key      | 允许（1个）              | 允许（1个）                | **不允许**（NullPointerException）|
| null value    | 允许                     | 允许                       | 允许                        |
| get/put       | O(1) 均摊                | O(1) 均摊                  | O(log n)                   |
| 内存占用      | 较低                     | 较高（额外双向链表）       | 较高（树节点含父/左/右指针）|
| 范围查询      | 不支持                   | 不支持                     | 支持（NavigableMap接口）   |
| 适用场景      | 通用键值存储（最常用）   | 需要顺序/LRU 缓存          | 需要按键排序/范围查询      |
| 线程安全      | 否                       | 否                         | 否                          |


---

# Part 7: HashSet / LinkedHashSet / TreeSet

## 7.1 HashSet 底层是 HashMap

`HashSet` 完全委托给 `HashMap` 实现：元素作为 key，固定的 `PRESENT` 对象作为 value（占位符）。

```java
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    private transient HashMap<E,Object> map;

    // 所有 HashMap 的 value 都是这个共享的占位符对象
    // 这样做节省内存（只需要一个对象），语义上表示"存在"
    private static final Object PRESENT = new Object();

    public HashSet() {
        map = new HashMap<>();
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    // 从 Collection 构造（initialCapacity = max(16, collection.size()*2)）
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int)(c.size()/.75f) + 1, 16));
        addAll(c);
    }

    // 给 LinkedHashSet 用的包内构造器（使用 LinkedHashMap）
    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }

    public boolean add(E e) {
        // map.put 返回 null 表示是新 key（新元素），返回 PRESENT 表示已存在
        return map.put(e, PRESENT) == null;
    }

    public boolean remove(Object o) {
        return map.remove(o) == PRESENT;
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public int size() {
        return map.size();
    }

    public Iterator<E> iterator() {
        return map.keySet().iterator(); // 迭代 HashMap 的 keySet
    }
}
```

## 7.2 LinkedHashSet 底层是 LinkedHashMap

```java
// LinkedHashSet 继承 HashSet，通过包内构造器使用 LinkedHashMap
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    public LinkedHashSet() {
        super(16, .75f, true); // 调用 HashSet(int, float, boolean) 构造器
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }
}

// HashSet 的包内构造器（专门给 LinkedHashSet 用）：
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
    // 注意：LinkedHashMap 默认 accessOrder=false（插入顺序）
    // LinkedHashSet 维护插入顺序
}
```

**使用对比：**

```java
import java.util.*;

public class SetOrderDemo {
    public static void main(String[] args) {
        // HashSet：无序（遍历顺序不定）
        Set<String> hashSet = new HashSet<>(Arrays.asList("C", "A", "B", "D"));
        System.out.println("HashSet: " + hashSet); // 顺序不定，如 [A, B, C, D] 或其他

        // LinkedHashSet：保持插入顺序
        Set<String> linkedHashSet = new LinkedHashSet<>(Arrays.asList("C", "A", "B", "D"));
        System.out.println("LinkedHashSet: " + linkedHashSet); // [C, A, B, D]

        // TreeSet：自然排序（字母序）
        Set<String> treeSet = new TreeSet<>(Arrays.asList("C", "A", "B", "D"));
        System.out.println("TreeSet: " + treeSet); // [A, B, C, D]
    }
}
```

## 7.3 TreeSet 底层是 TreeMap

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable {

    // 底层委托给 TreeMap
    private transient NavigableMap<E,Object> m;
    private static final Object PRESENT = new Object();

    public TreeSet() {
        this(new TreeMap<>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator)); // 自定义比较器
    }

    public boolean add(E e) {
        return m.put(e, PRESENT) == null;
    }

    // NavigableSet 方法都委托给 NavigableMap
    public E first() { return m.firstKey(); }
    public E last()  { return m.lastKey(); }
    public E floor(E e)   { return m.floorKey(e); }
    public E ceiling(E e) { return m.ceilingKey(e); }
    public E lower(E e)   { return m.lowerKey(e); }
    public E higher(E e)  { return m.higherKey(e); }

    public SortedSet<E> subSet(E fromElement, E toElement) {
        return subSet(fromElement, true, toElement, false);
    }
    public NavigableSet<E> subSet(E from, boolean fromInclusive,
                                  E to, boolean toInclusive) {
        return new TreeSet<>(m.subMap(from, fromInclusive, to, toInclusive));
    }
}
```

## 7.4 Set 去重原理（hashCode + equals）

```
HashSet/LinkedHashSet 去重原理（基于 HashMap）：
步骤1: 计算元素的 hash 值（通过 HashMap.hash(e.hashCode())）
步骤2: 定位桶：(n-1) & hash
步骤3: 检查桶中是否有 hash 相同且 equals 相同的 key
  - 有 → map.put 返回旧 value（PRESENT），add 返回 false（重复）
  - 无 → map.put 返回 null，add 返回 true（新增）

TreeSet 去重原理（基于 TreeMap）：
使用 Comparator 或 Comparable 的 compareTo 方法
compareTo 返回 0 → 认为是同一个 key，更新 value，add 返回 false

关键区别：
- HashSet：通过 hashCode + equals 判断重复
- TreeSet：通过 compareTo（或 Comparator.compare）判断重复
  注意：如果 compareTo 不一致（inconsistent with equals），可能产生奇怪结果！

hashCode + equals 契约（必须遵守！）：
a.equals(b) == true → a.hashCode() == b.hashCode()（反过来不必然成立）
```

## 7.5 自定义对象作为 Set 元素（必须重写 hashCode 和 equals）

```java
import java.util.*;

// 错误示例：没有重写 hashCode 和 equals
class BadPoint {
    int x, y;
    BadPoint(int x, int y) { this.x = x; this.y = y; }
    // 使用默认的 hashCode（基于对象内存地址）和 equals（== 比较）
    // 两个内容相同的 BadPoint 对象，hashCode 不同，equals 为 false
}

// 正确示例：正确重写 hashCode 和 equals
class GoodPoint {
    int x, y;
    GoodPoint(int x, int y) { this.x = x; this.y = y; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;               // 同一引用
        if (!(o instanceof GoodPoint)) return false; // 类型检查
        GoodPoint p = (GoodPoint) o;
        return x == p.x && y == p.y;              // 内容比较
    }

    @Override
    public int hashCode() {
        // 推荐使用 Objects.hash，它会正确处理 null 和多字段
        return Objects.hash(x, y);
        // 等价的手动实现：return 31 * x + y;
    }

    @Override
    public String toString() { return "(" + x + "," + y + ")"; }
}

public class CustomSetDemo {
    public static void main(String[] args) {
        // ===== 错误：BadPoint 无法正确去重 =====
        Set<BadPoint> badSet = new HashSet<>();
        badSet.add(new BadPoint(1, 2)); // hashCode = h1
        badSet.add(new BadPoint(1, 2)); // hashCode = h2（不同！因为是不同对象）
        System.out.println("BadPoint Set size: " + badSet.size()); // 2（没有去重！）

        // ===== 正确：GoodPoint 能正确去重 =====
        Set<GoodPoint> goodSet = new HashSet<>();
        goodSet.add(new GoodPoint(1, 2));
        goodSet.add(new GoodPoint(1, 2)); // hashCode 相同，equals 相同 → 去重
        goodSet.add(new GoodPoint(3, 4));
        System.out.println("GoodPoint Set size: " + goodSet.size()); // 2
        System.out.println("GoodPoint Set: " + goodSet); // [(1,2), (3,4)]

        // 用于 HashMap 的 key 同理
        Map<GoodPoint, String> map = new HashMap<>();
        map.put(new GoodPoint(0, 0), "origin");
        System.out.println(map.get(new GoodPoint(0, 0))); // "origin"（正确）

        Map<BadPoint, String> badMap = new HashMap<>();
        badMap.put(new BadPoint(0, 0), "origin");
        System.out.println(badMap.get(new BadPoint(0, 0))); // null（错误！）

        // ===== TreeSet 需要实现 Comparable 或提供 Comparator =====
        // 方式1：实现 Comparable（GoodPoint 已有 equals，让 Comparable 一致）
        Set<GoodPoint> treeSet = new TreeSet<>((p1, p2) -> {
            if (p1.x != p2.x) return Integer.compare(p1.x, p2.x);
            return Integer.compare(p1.y, p2.y);
        });
        treeSet.add(new GoodPoint(3, 1));
        treeSet.add(new GoodPoint(1, 2));
        treeSet.add(new GoodPoint(1, 1));
        treeSet.add(new GoodPoint(1, 2)); // 重复（compareTo=0）
        System.out.println("TreeSet: " + treeSet); // [(1,1), (1,2), (3,1)]（3个，去重了）
    }
}
```

## 7.6 hashCode 和 equals 的最佳实践

```java
import java.util.Objects;

/**
 * 标准的 hashCode + equals 实现模板
 * 适用于：作为 HashMap/HashSet 的 key，或需要通过 contains/indexOf 查找
 */
class Student {
    private final String id;   // 不可变字段（如果可变，放入集合后修改会导致迷失）
    private final String name;
    private int score;         // 可变字段（不要放入 hashCode/equals）

    Student(String id, String name, int score) {
        this.id = id;
        this.name = name;
        this.score = score;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;            // 引用相同直接返回 true
        if (!(o instanceof Student)) return false; // 类型不同返回 false
        Student s = (Student) o;
        // 使用 Objects.equals 安全处理 null
        return Objects.equals(id, s.id) && Objects.equals(name, s.name);
        // 注意：score 不参与 equals（可变字段不应参与）
    }

    @Override
    public int hashCode() {
        // Objects.hash 内部调用 Arrays.hashCode，正确处理 null
        return Objects.hash(id, name);
        // 注意：hashCode 的字段集合必须是 equals 字段集合的子集
        // 规则：equals 返回 true 的两个对象，hashCode 必须相同
    }

    // 可变字段的修改警告：
    // 如果 Student 对象已经放入 HashSet，修改 id/name 会导致：
    // - hashCode 变化 → 对象落入不同的桶
    // - 原来的桶找不到这个对象
    // - set.contains(student) 返回 false！（即使对象在set中）
    // 解决方案：尽量使用不可变对象作为 key
}
```

## 7.7 Set 集合运算

```java
import java.util.*;
import java.util.stream.*;

public class SetOperationsDemo {
    public static void main(String[] args) {
        Set<Integer> a = new HashSet<>(Arrays.asList(1, 2, 3, 4, 5));
        Set<Integer> b = new HashSet<>(Arrays.asList(3, 4, 5, 6, 7));

        // === 并集（a ∪ b）===
        Set<Integer> union = new HashSet<>(a);
        union.addAll(b);
        System.out.println("并集: " + union); // [1, 2, 3, 4, 5, 6, 7]

        // === 交集（a ∩ b）===
        Set<Integer> intersection = new HashSet<>(a);
        intersection.retainAll(b);
        System.out.println("交集: " + intersection); // [3, 4, 5]

        // === 差集（a - b）===
        Set<Integer> difference = new HashSet<>(a);
        difference.removeAll(b);
        System.out.println("差集: " + difference); // [1, 2]

        // === 对称差集（a △ b = (a∪b) - (a∩b)）===
        Set<Integer> symDiff = new HashSet<>(union);
        symDiff.removeAll(intersection);
        System.out.println("对称差集: " + symDiff); // [1, 2, 6, 7]

        // === 子集判断 ===
        System.out.println("intersection 是 a 的子集: " + a.containsAll(intersection)); // true
        System.out.println("b 是 a 的子集: " + a.containsAll(b)); // false

        // === Java 8 Stream 方式（不修改原集合）===
        Set<Integer> unionStream = Stream.concat(a.stream(), b.stream())
                                         .collect(Collectors.toSet());
        Set<Integer> intersectionStream = a.stream()
                                           .filter(b::contains)
                                           .collect(Collectors.toSet());

        // === List 去重（转为 LinkedHashSet 保持顺序）===
        List<String> listWithDups = Arrays.asList("a", "b", "a", "c", "b", "d");
        List<String> deduplicated = new ArrayList<>(new LinkedHashSet<>(listWithDups));
        System.out.println("去重保序: " + deduplicated); // [a, b, c, d]

        // 不保持顺序（用 HashSet，性能稍好）
        List<String> dedup2 = new ArrayList<>(new HashSet<>(listWithDups));
        System.out.println("去重不保序: " + dedup2); // 顺序不定

        // Java 8 Stream 去重
        List<String> dedup3 = listWithDups.stream().distinct().collect(Collectors.toList());
        System.out.println("Stream去重: " + dedup3); // [a, b, c, d]（保持顺序）
    }
}
```


---

# Part 8: ConcurrentHashMap 深度解析（重点）

## 8.1 JDK 7 分段锁 vs JDK 8 CAS + synchronized

### JDK 7 实现：Segment 分段锁

`
ConcurrentHashMap（JDK7）：

  Segment[0]（extends ReentrantLock）
    HashEntry[0..15]（桶数组）

  Segment[1]
    HashEntry[0..15]

  ...

  Segment[15]（默认16个段）

每个 Segment 有自己的桶数组，加锁粒度是 Segment
最大并发度 = 段数量（默认16）
`

### JDK 8 实现：CAS + synchronized

`
ConcurrentHashMap（JDK8）：

volatile Node[] table（一个大的桶数组）
  [0]: null
  [1]: Node(K1,V1) -> Node(K2,V2)  <- 链表
  [2]: TreeBin                      <- 红黑树
  [3]: ForwardingNode               <- 正在扩容

加锁粒度：synchronized(桶头节点)，粒度比 Segment 更细
最大并发度 = 数组长度
`

## 8.2 JDK 8 put 流程

`java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 不允许 null key 或 null value（区别于 HashMap）
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;

    for (Node<K,V>[] tab = table;;) { // 自旋
        Node<K,V> f;
        int n, i, fh;

        // 情况1：table 未初始化 -> CAS initTable()
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();

        // 情况2：目标桶为空 -> CAS 无锁插入
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;

        // 情况3：桶头是 ForwardingNode（正在扩容）-> 帮助扩容
        } else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);

        // 情况4：桶不为空 -> synchronized(桶头) 加锁
        } else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) { // 双重检查
                    if (fh >= 0) { // 链表
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            if (e.hash == hash &&
                                ((ek = e.key) == key || key.equals(ek))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    } else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                if (binCount >= TREEIFY_THRESHOLD) treeifyBin(tab, i);
                if (oldVal != null) return oldVal;
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
`

**put 流程图：**

`
putVal(key, value)
  |
  v
null 检查 --> NullPointerException
  |
  v
自旋 {
  table 未初始化?  --> CAS initTable()
  桶为空?          --> CAS casTabAt()（成功则break）
  桶是 MOVED?      --> helpTransfer()（参与扩容）
  其他             --> synchronized(桶头) { 链表/树操作 }
}
  |
  v
addCount(1) --> 更新 size，可能触发扩容
`

## 8.3 initTable（CAS 保证单线程初始化）

`java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        if ((sc = sizeCtl) < 0)
            Thread.yield(); // 其他线程正在初始化，让出CPU
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            // CAS 将 sizeCtl 改为 -1，只有一个线程能成功
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY; // 16
                    table = tab = new Node[n];
                    sc = n - (n >>> 2); // 0.75 * n
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
`

## 8.4 get 方法（无锁）

`java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());

    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n-1) & h)) != null) { // volatile 读
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val; // 头节点命中
        } else if (eh < 0)
            // TreeBin(hash=-2) 或 ForwardingNode(hash=-1)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
`

**为什么 get 不需要加锁：**
1. Node.val 和 Node.next 均为 olatile，写操作对读立即可见
2. 	able 通过 Unsafe.getObjectVolatile 读（volatile 语义）
3. 扩容期间 ForwardingNode.find() 可在新数组中查找
4. 不修改任何状态，不会破坏数据结构

## 8.5 size 统计（分段累加，LongAdder 思想）

`java
// 高并发下，避免单一原子计数器的竞争
// 原理类似 LongAdder：
//   - 低竞争时：CAS 修改 baseCount
//   - 高竞争时：将增量写入某个 CounterCell（通过线程哈希选择）
//   - size() = baseCount + sum(所有 CounterCell.value)

private volatile long baseCount;
private volatile CounterCell[] counterCells; // CounterCell 带 @Contended 防止伪共享

final long sumCount() {
    CounterCell[] as = counterCells;
    long sum = baseCount;
    if (as != null)
        for (CounterCell a : as)
            if (a != null) sum += a.value;
    return sum; // 注意：这是近似值（弱一致性）
}

public int size() {
    long n = sumCount();
    return (n < 0L) ? 0 : (n > Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int)n;
}
`

## 8.6 扩容（多线程协同）

`
多线程协同扩容原理：

1. 触发扩容：第一个线程创建新数组（2倍），设置 nextTable
2. 分配任务：按 stride（= max(16, n/8/CPU核数)）划分区段
3. 线程认领：CAS 修改 transferIndex，每个线程领取一段（从右到左迁移）
4. 迁移完成：将已迁移的桶设置为 ForwardingNode（hash=MOVED）
5. 帮助扩容：其他线程 put 时遇到 MOVED 节点，调用 helpTransfer 参与迁移
6. 扩容完成：所有桶迁移完毕，table 替换为 nextTable

ForwardingNode 的作用：
- 标记该桶已迁移完成
- get 操作遇到 ForwardingNode，调用 find() 在新数组中查找
- put 操作遇到 MOVED，调用 helpTransfer 参与扩容，然后继续插入
`

## 8.7 JDK 8 vs JDK 7 改进总结

`
JDK 7 分段锁缺点：
1. 并发级别固定（默认16），无法动态扩展
2. Segment 内存开销较大（继承 ReentrantLock）
3. 跨段操作复杂（如精确 size 需要锁所有段）

JDK 8 改进：
1. 锁粒度降至桶级，最大并发度 = 数组长度（随扩容增大）
2. 空桶 CAS 操作，无需加锁
3. synchronized（经过 JVM 优化：偏向锁→轻量级锁→重量级锁）
4. 多线程协同扩容，加速扩容过程
5. LongAdder 思想统计 size，高并发下性能更好
`

## 8.8 ConcurrentHashMap vs Hashtable vs synchronizedMap 对比

| 特性               | Hashtable          | synchronizedMap     | ConcurrentHashMap       |
|-------------------|--------------------|---------------------|-------------------------|
| 线程安全           | 是                 | 是                  | 是                       |
| 锁粒度             | 整个对象           | 整个对象            | 单个桶（细粒度）         |
| null key/value    | 不允许             | 取决于底层 Map      | 不允许                   |
| 迭代器             | fail-fast          | fail-fast           | fail-safe（弱一致性）    |
| 遍历需要手动同步   | 否                 | 是！                 | 否                       |
| 性能               | 差                 | 差                  | 好                       |
| 复合原子操作       | 无                 | 无                  | putIfAbsent/compute/merge|
| 推荐               | 否（已过时）       | 否（并发性差）      | 是                       |

`java
import java.util.*;
import java.util.concurrent.*;

public class ConcurrentHashMapDemo {
    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

        // 基本操作（与 HashMap 相同）
        map.put("Alice", 90);
        map.put("Bob", 85);
        System.out.println(map.get("Alice")); // 90

        // 原子复合操作（线程安全）
        map.putIfAbsent("Charlie", 95); // 不存在才插入
        map.computeIfAbsent("scores", k -> 0); // 不存在则计算

        // 原子计数（线程安全的计数器实现）
        String[] words = {"apple", "banana", "apple", "cherry"};
        ConcurrentHashMap<String, Integer> wordCount = new ConcurrentHashMap<>();
        for (String w : words) {
            wordCount.merge(w, 1, Integer::sum); // 原子：不存在则插入1，存在则+1
        }
        System.out.println(wordCount); // {apple=2, banana=1, cherry=1}

        // 并发测试
        ConcurrentHashMap<Integer, Integer> concurrent = new ConcurrentHashMap<>();
        Thread[] threads = new Thread[10];
        for (int t = 0; t < 10; t++) {
            final int tid = t;
            threads[t] = new Thread(() -> {
                for (int i = 0; i < 1000; i++) {
                    concurrent.merge(i % 100, 1, Integer::sum);
                }
            });
            threads[t].start();
        }
        for (Thread t : threads) t.join();

        // 验证：每个 key 被更新了 10 * (1000/100) = 100 次
        System.out.println("key 0 count: " + concurrent.get(0)); // 应该是 100
    }
}
`


---

# Part 9: Queue 与 Deque

## 9.1 Queue 接口（FIFO 队列）

`java
public interface Queue<E> extends Collection<E> {
    // 入队：add 满时抛异常，offer 满时返回 false
    boolean add(E e);
    boolean offer(E e);

    // 出队（删除并返回）：remove 空时抛异常，poll 空时返回 null
    E remove();
    E poll();

    // 查看队头（不删除）：element 空时抛异常，peek 空时返回 null
    E element();
    E peek();
}
`

| 操作 | 抛异常版本（队满/空时） | 返回特殊值版本 |
|------|----------------------|---------------|
| 入队 | add(e)（抛 IllegalStateException） | offer(e)（返回 false）|
| 出队 | remove()（抛 NoSuchElementException）| poll()（返回 null）|
| 查头 | element()（抛 NoSuchElementException）| peek()（返回 null）|

## 9.2 Deque 接口（双端队列）

`java
public interface Deque<E> extends Queue<E> {
    // 头部操作
    void addFirst(E e);    boolean offerFirst(E e);
    E removeFirst();       E pollFirst();
    E getFirst();          E peekFirst();

    // 尾部操作
    void addLast(E e);     boolean offerLast(E e);
    E removeLast();        E pollLast();
    E getLast();           E peekLast();

    // 作为栈使用（LIFO）
    void push(E e);        // 等同于 addFirst
    E pop();               // 等同于 removeFirst
    E peek();              // 等同于 peekFirst
}
`

## 9.3 ArrayDeque（循环数组，推荐使用）

ArrayDeque 底层使用**循环数组**实现，是 JDK 推荐的栈和队列实现。

`
循环数组原理（capacity=8）：

head=0, tail=4：[A, B, C, D, _, _, _, _]
              head↑              tail↑

addLast(E)：arr[tail] = E; tail = (tail+1) & (capacity-1)
pollFirst()：E = arr[head]; head = (head+1) & (capacity-1)
addFirst(E)：head = (head-1) & (capacity-1); arr[head] = E

绕回示例（head=6, tail=2）：[C, D, _, _, _, _, A, B]
                         tail↑           head↑
`

`java
import java.util.*;

public class ArrayDequeDemo {
    public static void main(String[] args) {
        // === 作为队列（FIFO）===
        Deque<String> queue = new ArrayDeque<>();
        queue.offer("A");   // 入队（队尾）
        queue.offer("B");
        queue.offer("C");
        System.out.println("队列: " + queue);   // [A, B, C]
        System.out.println(queue.poll());        // A（队头出队）
        System.out.println(queue.peek());        // B（不删除）

        // === 作为栈（LIFO）===
        Deque<Integer> stack = new ArrayDeque<>();
        stack.push(1);  // 入栈 = addFirst
        stack.push(2);
        stack.push(3);
        System.out.println("栈: " + stack);      // [3, 2, 1]
        System.out.println(stack.pop());          // 3（栈顶出栈）
        System.out.println(stack.peek());         // 2

        // === 作为双端队列 ===
        ArrayDeque<Integer> deque = new ArrayDeque<>();
        deque.addFirst(2); // [2]
        deque.addFirst(1); // [1, 2]
        deque.addLast(3);  // [1, 2, 3]
        deque.addLast(4);  // [1, 2, 3, 4]

        System.out.println("双端队列: " + deque);
        System.out.println(deque.peekFirst());   // 1
        System.out.println(deque.peekLast());    // 4
        System.out.println(deque.pollFirst());   // 1
        System.out.println(deque.pollLast());    // 4

        // === 注意：ArrayDeque 不允许 null 元素 ===
        try {
            deque.add(null); // NullPointerException!
        } catch (NullPointerException e) {
            System.out.println("不支持 null");
        }

        // === 实用：用 ArrayDeque 模拟函数调用栈/DFS ===
        // DFS 示例（非递归）
        Deque<Integer> dfsStack = new ArrayDeque<>();
        dfsStack.push(0); // 起始节点
        while (!dfsStack.isEmpty()) {
            int node = dfsStack.pop();
            System.out.print(node + " ");
            // 将邻居节点入栈...
        }
    }
}
`

## 9.4 PriorityQueue（小顶堆）

PriorityQueue 底层使用**二叉堆**（Binary Heap）实现，默认是小顶堆（最小元素优先出队）。

`
小顶堆结构（数组表示）：
                 1
              /     \
             3       5
            / \     / \
           7   4   8   6

数组存储：[1, 3, 5, 7, 4, 8, 6]（按层次遍历顺序）
索引规律：
  父节点：parent(i) = (i-1) / 2
  左孩子：left(i)   = 2*i + 1
  右孩子：right(i)  = 2*i + 2

堆的性质：每个节点 <= 其所有后代节点（小顶堆）
`

`java
import java.util.*;

public class PriorityQueueDemo {
    public static void main(String[] args) {
        // === 默认小顶堆（自然排序）===
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        minHeap.offer(5); minHeap.offer(1); minHeap.offer(3);
        minHeap.offer(8); minHeap.offer(2);
        System.out.println("堆顶(最小): " + minHeap.peek()); // 1
        // 按顺序出队（每次出队最小值）
        while (!minHeap.isEmpty()) {
            System.out.print(minHeap.poll() + " "); // 1 2 3 5 8
        }

        // === 大顶堆（逆序）===
        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
        maxHeap.offer(5); maxHeap.offer(1); maxHeap.offer(3);
        System.out.println("\n大顶堆堆顶(最大): " + maxHeap.peek()); // 5

        // === 自定义对象排序 ===
        PriorityQueue<int[]> taskQueue = new PriorityQueue<>(
            (a, b) -> a[1] - b[1] // 按优先级（第2个元素）升序
        );
        taskQueue.offer(new int[]{1, 5}); // [taskId, priority]
        taskQueue.offer(new int[]{2, 1});
        taskQueue.offer(new int[]{3, 3});
        System.out.println("\n最高优先级任务: " + Arrays.toString(taskQueue.poll())); // [2, 1]

        // === 经典应用：求 Top-K 最大元素（用小顶堆保持K个最大值）===
        int k = 3;
        int[] nums = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5};
        PriorityQueue<Integer> topK = new PriorityQueue<>(k); // 小顶堆，容量k
        for (int num : nums) {
            topK.offer(num);
            if (topK.size() > k) {
                topK.poll(); // 删除堆顶（最小值），保持k个最大值
            }
        }
        System.out.println("\nTop-" + k + ": " + topK); // 包含最大的3个元素
    }
}
`

**堆化操作（siftUp/siftDown）：**

`java
// offer(e)：将元素添加到末尾，然后向上调整（siftUp）
// 时间复杂度：O(log n)
private void siftUp(int k, E x) {
    while (k > 0) {
        int parent = (k - 1) >>> 1; // parent = (k-1)/2
        Object e = queue[parent];
        if (comparator.compare(x, (E) e) >= 0)
            break; // x >= parent，符合堆性质，停止
        queue[k] = e;   // 父节点下移
        k = parent;     // x 继续向上移动
    }
    queue[k] = x;
}

// poll()：取出堆顶，将末尾元素移到堆顶，然后向下调整（siftDown）
// 时间复杂度：O(log n)
private void siftDown(int k, E x) {
    int half = size >>> 1;
    while (k < half) {
        int child = (k << 1) + 1; // 左孩子
        Object c = queue[child];
        int right = child + 1;
        if (right < size && comparator.compare((E) c, (E) queue[right]) > 0)
            c = queue[child = right]; // 取较小的孩子
        if (comparator.compare(x, (E) c) <= 0)
            break; // x <= 较小孩子，符合堆性质，停止
        queue[k] = c;  // 较小孩子上移
        k = child;     // 继续向下调整
    }
    queue[k] = x;
}
`

## 9.5 BlockingQueue 简介（JUC 系列）

BlockingQueue 是线程安全的队列，支持阻塞的入队/出队操作：

| 实现类              | 特点                                      |
|--------------------|------------------------------------------|
| ArrayBlockingQueue  | 有界数组队列，FIFO，单锁（入队出队共用一把锁）|
| LinkedBlockingQueue | 有界/无界链表队列，FIFO，双锁（入队出队分离锁）|
| PriorityBlockingQueue | 无界优先级队列，线程安全版 PriorityQueue   |
| DelayQueue          | 延迟队列，元素到期才能出队                  |
| SynchronousQueue    | 容量为0，每次 put 必须等待 take（握手）     |
| LinkedTransferQueue | 更强大的 SynchronousQueue，Java 7 新增     |

`java
import java.util.concurrent.*;

public class BlockingQueueDemo {
    public static void main(String[] args) throws InterruptedException {
        // 生产者-消费者模式
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);

        // 生产者线程
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 20; i++) {
                    queue.put(i); // 队满时阻塞等待
                    System.out.println("生产: " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        // 消费者线程
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 20; i++) {
                    int val = queue.take(); // 队空时阻塞等待
                    System.out.println("消费: " + val);
                    Thread.sleep(50); // 模拟处理时间
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
        producer.join();
        consumer.join();
    }
}
`


---

# Part 10: 常用算法与数据结构

## 10.1 Arrays.sort 排序算法

`java
// Arrays.sort 针对不同类型使用不同算法
//
// 基本类型（int/long/double等）：
//   小数组（<47个元素）：插入排序（简单高效）
//   中等数组（<286个元素）：单轴快速排序
//   大数组：双轴快速排序（Dual-Pivot Quicksort，JDK 7+，Vladimir Yaroslavskiy 提出）
//     平均时间复杂度：O(n log n)
//     不稳定排序（相同元素的顺序可能改变）
//
// 对象类型（Object[]）：
//   TimSort（归并排序的优化版，JDK 7+）
//   稳定排序（相同元素的顺序保持不变）
//   时间复杂度：O(n log n) 最坏，O(n) 最好（已排序数据）
//   利用已有的"run"（有序子序列）减少排序工作量

import java.util.*;

public class SortDemo {
    public static void main(String[] args) {
        // 基本类型排序（双轴快排，不稳定）
        int[] arr = {5, 3, 8, 1, 9, 2, 7, 4, 6};
        Arrays.sort(arr);
        System.out.println(Arrays.toString(arr)); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

        // 部分排序
        int[] partial = {5, 3, 8, 1, 9, 2};
        Arrays.sort(partial, 1, 5); // 排序索引 [1, 5) 的元素
        System.out.println(Arrays.toString(partial)); // [5, 1, 3, 8, 9, 2]

        // 对象类型排序（TimSort，稳定）
        String[] words = {"banana", "apple", "cherry", "date"};
        Arrays.sort(words);
        System.out.println(Arrays.toString(words)); // [apple, banana, cherry, date]

        // 自定义排序
        String[] byLength = {"banana", "apple", "cherry", "fig"};
        Arrays.sort(byLength, Comparator.comparingInt(String::length));
        System.out.println(Arrays.toString(byLength)); // [fig, apple, banana, cherry]

        // Collections.sort（底层调用 Arrays.sort，TimSort）
        List<Integer> list = new ArrayList<>(Arrays.asList(5, 3, 8, 1, 2));
        Collections.sort(list);
        System.out.println(list); // [1, 2, 3, 5, 8]

        // List.sort（Java 8，等同于 Collections.sort）
        list.sort(Comparator.reverseOrder());
        System.out.println(list); // [8, 5, 3, 2, 1]
    }
}
`

## 10.2 二分查找（binarySearch）

`java
import java.util.*;

public class BinarySearchDemo {
    public static void main(String[] args) {
        // 前提：数组/List 必须已排序！

        // Arrays.binarySearch（基本类型）
        int[] arr = {1, 2, 3, 5, 8, 13, 21};
        System.out.println(Arrays.binarySearch(arr, 8));  // 4（找到，返回索引）
        System.out.println(Arrays.binarySearch(arr, 6));  // -5（未找到，返回 -(插入点)-1）
        // 插入点 = 如果要插入6，它应该在索引4处（保持有序），所以返回 -(4)-1 = -5

        // Arrays.binarySearch（对象数组）
        String[] words = {"apple", "banana", "cherry"};
        System.out.println(Arrays.binarySearch(words, "banana")); // 1

        // 带 Comparator 的 binarySearch
        String[] nums = {"1", "10", "2", "20", "3"};
        Arrays.sort(nums, Comparator.comparingInt(Integer::parseInt));
        System.out.println(Arrays.binarySearch(nums, "10",
            Comparator.comparingInt(Integer::parseInt))); // 1

        // Collections.binarySearch（List）
        List<Integer> list = Arrays.asList(1, 3, 5, 7, 9, 11);
        System.out.println(Collections.binarySearch(list, 7));  // 3
        System.out.println(Collections.binarySearch(list, 6));  // 负数（未找到）

        // 判断元素是否存在（利用 binarySearch 结果）
        int idx = Collections.binarySearch(list, 5);
        System.out.println(idx >= 0 ? "存在" : "不存在"); // 存在

        // 找到插入位置（维护有序列表）
        int insertIdx = Collections.binarySearch(list, 6);
        if (insertIdx < 0) {
            int insertPos = -(insertIdx) - 1;
            System.out.println("6 应该插入到索引: " + insertPos); // 3
        }
    }
}
`

## 10.3 Comparable vs Comparator（链式比较）

`java
import java.util.*;
import java.util.stream.*;

public class ComparatorChainDemo {
    static class Employee implements Comparable<Employee> {
        String name;
        String dept;
        int salary;
        int age;

        Employee(String name, String dept, int salary, int age) {
            this.name = name;
            this.dept = dept;
            this.salary = salary;
            this.age = age;
        }

        // 自然排序：按姓名
        @Override
        public int compareTo(Employee o) {
            return this.name.compareTo(o.name);
        }

        @Override
        public String toString() {
            return name + "(" + dept + "," + salary + "," + age + ")";
        }
    }

    public static void main(String[] args) {
        List<Employee> employees = new ArrayList<>(Arrays.asList(
            new Employee("Charlie", "Engineering", 90000, 35),
            new Employee("Alice", "Marketing", 75000, 28),
            new Employee("Bob", "Engineering", 85000, 32),
            new Employee("Diana", "Engineering", 90000, 30),
            new Employee("Eve", "Marketing", 75000, 25)
        ));

        // 自然排序（按姓名）
        Collections.sort(employees);
        System.out.println("按姓名: " + employees);

        // 链式 Comparator（Java 8）
        // 多字段排序：先按部门，再按薪水降序，再按年龄升序
        employees.sort(
            Comparator.comparing((Employee e) -> e.dept)  // 先按部门（升序）
                      .thenComparingInt((Employee e) -> e.salary)  // 再按薪水
                      .reversed()  // 反转（薪水降序，部门也变降序了）
                      .thenComparingInt((Employee e) -> e.age)  // 再按年龄升序
        );
        System.out.println("多字段排序: " + employees);

        // 更清晰的写法（不用 reversed）
        Comparator<Employee> comp =
            Comparator.comparing((Employee e) -> e.dept)
                      .thenComparing(Comparator.comparingInt((Employee e) -> e.salary).reversed())
                      .thenComparingInt(e -> e.age);
        employees.sort(comp);
        System.out.println("清晰多字段: " + employees);

        // nullsFirst / nullsLast（处理 null 值）
        List<String> withNulls = Arrays.asList("banana", null, "apple", null, "cherry");
        withNulls.sort(Comparator.nullsFirst(Comparator.naturalOrder()));
        System.out.println("nulls first: " + withNulls);
        // [null, null, apple, banana, cherry]

        withNulls.sort(Comparator.nullsLast(Comparator.naturalOrder()));
        System.out.println("nulls last: " + withNulls);
        // [apple, banana, cherry, null, null]
    }
}
`

## 10.4 Java 8 Stream 与集合结合使用

`java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class StreamCollectionDemo {
    record Person(String name, int age, String city) {}

    public static void main(String[] args) {
        List<Person> people = Arrays.asList(
            new Person("Alice", 30, "Beijing"),
            new Person("Bob", 25, "Shanghai"),
            new Person("Charlie", 35, "Beijing"),
            new Person("Diana", 28, "Guangzhou"),
            new Person("Eve", 22, "Shanghai"),
            new Person("Frank", 40, "Beijing")
        );

        // === 过滤 + 转换 ===
        List<String> names = people.stream()
            .filter(p -> p.age() > 25)
            .map(Person::name)
            .sorted()
            .collect(Collectors.toList());
        System.out.println("年龄>25的姓名: " + names);

        // === 分组 ===
        Map<String, List<Person>> byCity = people.stream()
            .collect(Collectors.groupingBy(Person::city));
        byCity.forEach((city, ps) ->
            System.out.println(city + ": " + ps.stream().map(Person::name).collect(Collectors.toList())));

        // === 统计 ===
        Map<String, Long> countByCity = people.stream()
            .collect(Collectors.groupingBy(Person::city, Collectors.counting()));
        System.out.println("各城市人数: " + countByCity);

        Map<String, Double> avgAgeByCity = people.stream()
            .collect(Collectors.groupingBy(Person::city, Collectors.averagingInt(Person::age)));
        System.out.println("各城市平均年龄: " + avgAgeByCity);

        // === 收集到不同集合 ===
        Set<String> nameSet = people.stream()
            .map(Person::name)
            .collect(Collectors.toSet()); // HashSet

        // 收集到 LinkedHashMap（保持顺序）
        Map<String, Integer> nameAgeMap = people.stream()
            .collect(Collectors.toMap(
                Person::name,
                Person::age,
                (e1, e2) -> e1,  // 重复key时的合并策略
                LinkedHashMap::new  // 使用 LinkedHashMap
            ));

        // === joining（字符串拼接）===
        String joined = people.stream()
            .map(Person::name)
            .collect(Collectors.joining(", ", "[", "]"));
        System.out.println(joined); // [Alice, Bob, Charlie, Diana, Eve, Frank]

        // === 统计汇总 ===
        IntSummaryStatistics stats = people.stream()
            .collect(Collectors.summarizingInt(Person::age));
        System.out.println("年龄统计: count=" + stats.getCount() +
            ", sum=" + stats.getSum() + ", min=" + stats.getMin() +
            ", max=" + stats.getMax() + ", avg=" + stats.getAverage());

        // === partition（按布尔条件分区）===
        Map<Boolean, List<Person>> partition = people.stream()
            .collect(Collectors.partitioningBy(p -> p.age() >= 30));
        System.out.println(">=30: " + partition.get(true).stream().map(Person::name).toList());
        System.out.println("<30:  " + partition.get(false).stream().map(Person::name).toList());

        // === flatMap（展平嵌套集合）===
        List<List<Integer>> nested = Arrays.asList(
            Arrays.asList(1, 2, 3),
            Arrays.asList(4, 5),
            Arrays.asList(6, 7, 8, 9)
        );
        List<Integer> flat = nested.stream()
            .flatMap(Collection::stream)
            .collect(Collectors.toList());
        System.out.println("展平: " + flat); // [1, 2, 3, 4, 5, 6, 7, 8, 9]
    }
}
`

## 10.5 集合转换

`java
import java.util.*;
import java.util.stream.*;

public class CollectionConversionDemo {
    public static void main(String[] args) {
        // === List → Set（去重）===
        List<Integer> list = Arrays.asList(1, 2, 3, 2, 1, 4);
        Set<Integer> set = new HashSet<>(list);          // 无序去重
        Set<Integer> orderedSet = new LinkedHashSet<>(list); // 保持插入顺序去重
        System.out.println("去重(无序): " + set);     // [1, 2, 3, 4]
        System.out.println("去重(有序): " + orderedSet); // [1, 2, 3, 4]

        // === Set → List ===
        List<Integer> fromSet = new ArrayList<>(set);
        Collections.sort(fromSet); // 需要手动排序

        // === Map → List ===
        Map<String, Integer> map = Map.of("Alice", 30, "Bob", 25, "Charlie", 35);
        List<String> keys = new ArrayList<>(map.keySet());
        List<Integer> values = new ArrayList<>(map.values());
        List<Map.Entry<String, Integer>> entries = new ArrayList<>(map.entrySet());

        // === List → Map ===
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");
        Map<String, Integer> nameLength = names.stream()
            .collect(Collectors.toMap(Function.identity(), String::length));
        System.out.println("name->length: " + nameLength);

        // === 数组 ↔ List ===
        // 数组 → ArrayList（可变）
        String[] arr = {"a", "b", "c"};
        List<String> fromArr = new ArrayList<>(Arrays.asList(arr));

        // ArrayList → 数组
        String[] backToArr = fromArr.toArray(new String[0]);

        // === 不可变集合（Java 9+）===
        List<String> immList = List.of("a", "b", "c");         // 不可变列表
        Set<String> immSet = Set.of("x", "y", "z");            // 不可变集合
        Map<String, Integer> immMap = Map.of("k1", 1, "k2", 2); // 不可变Map（<=10个）
        Map<String, Integer> immMap2 = Map.ofEntries(           // 不可变Map（任意个）
            Map.entry("k1", 1),
            Map.entry("k2", 2),
            Map.entry("k3", 3)
        );
        // immList.add("d"); // UnsupportedOperationException

        // Java 8 方式（不可变视图）
        List<String> immList8 = Collections.unmodifiableList(Arrays.asList("a", "b", "c"));
        // 与 List.of 区别：unmodifiable 是视图（原集合改变会影响它），List.of 是真正不可变

        System.out.println("不可变List: " + immList);
        System.out.println("不可变Set: " + immSet);
        System.out.println("不可变Map: " + immMap);
    }
}
`

## 10.6 TimSort 简介

`
TimSort（Java 7+ 用于对象数组排序）：

特点：
1. 稳定排序（相等元素相对顺序不变）
2. 自适应（利用数据中已有的有序片段）
3. 最好 O(n)（数据已有序），最坏 O(n log n)，空间 O(n)

基本思想：
1. 在数组中找"run"（自然有序的子数组，升序或降序）
2. 对过短的 run，用二分插入排序扩展到最小长度（minRun，通常32-64）
3. 将 run 压入栈，满足条件时合并相邻 run（归并排序）
4. 合并策略满足：run(i) > run(i+1) + run(i+2)（避免不平衡合并）

优势：
- 对现实中常见的"部分有序"数据非常高效
- 例如：日志数据通常按时间部分有序
`


---

# Part 11: 性能对比与选型指南

## 11.1 各集合时间复杂度汇总

### List 时间复杂度

| 操作         | ArrayList  | LinkedList  | 说明                          |
|-------------|------------|-------------|-------------------------------|
| get(i)      | O(1)       | O(n)        | ArrayList 直接数组下标         |
| add(末尾)   | O(1) 均摊  | O(1)        | ArrayList 偶尔扩容 O(n)        |
| add(头部)   | O(n)       | O(1)        | ArrayList 需要移动所有元素      |
| add(中间)   | O(n)       | O(n)        | LinkedList 需要先找到位置       |
| remove(末尾)| O(1)       | O(1)        |                               |
| remove(头部)| O(n)       | O(1)        | ArrayList 需要移动所有元素      |
| remove(中间)| O(n)       | O(n)        | LinkedList 需要先找到位置       |
| contains    | O(n)       | O(n)        | 都需要遍历                     |
| indexOf     | O(n)       | O(n)        |                               |
| size        | O(1)       | O(1)        |                               |

### Map 时间复杂度

| 操作       | HashMap    | LinkedHashMap | TreeMap      | ConcurrentHashMap |
|-----------|------------|---------------|--------------|-------------------|
| get       | O(1) 均摊  | O(1) 均摊     | O(log n)     | O(1) 均摊         |
| put       | O(1) 均摊  | O(1) 均摊     | O(log n)     | O(1) 均摊         |
| remove    | O(1) 均摊  | O(1) 均摊     | O(log n)     | O(1) 均摊         |
| containsKey| O(1) 均摊 | O(1) 均摊     | O(log n)     | O(1) 均摊         |
| 遍历所有   | O(n)       | O(n)          | O(n)         | O(n)              |
| firstKey  | -          | O(1)          | O(log n)     | -                 |
| floorKey  | -          | -             | O(log n)     | -                 |
| subMap    | -          | -             | O(log n)     | -                 |

### Set 时间复杂度

| 操作      | HashSet    | LinkedHashSet | TreeSet      |
|----------|------------|---------------|--------------|
| add       | O(1) 均摊  | O(1) 均摊     | O(log n)     |
| remove    | O(1) 均摊  | O(1) 均摊     | O(log n)     |
| contains  | O(1) 均摊  | O(1) 均摊     | O(log n)     |
| 遍历      | O(n)       | O(n)          | O(n)         |
| first/last| -          | -             | O(log n)     |

### Queue/Deque 时间复杂度

| 操作       | ArrayDeque | LinkedList | PriorityQueue |
|-----------|------------|------------|---------------|
| offer/add  | O(1) 均摊  | O(1)       | O(log n)      |
| poll/remove| O(1) 均摊  | O(1)       | O(log n)      |
| peek       | O(1)       | O(1)       | O(1)          |
| contains   | O(n)       | O(n)       | O(n)          |

## 11.2 内存消耗对比

`
各集合每个元素的内存开销估算（64位 JVM，不含元素本身）：

ArrayList：
  每个元素：8B（Object 引用）
  数组头部：16B 对象头 + 4B length
  总列表开销：~24B + 8B * capacity

LinkedList：
  每个元素：Node 对象
  Node = 16B 对象头 + 8B prev + 8B item + 8B next = 40B
  相比 ArrayList，每个元素额外 32B！

HashMap（每个 Entry）：
  Node = 16B 对象头 + 4B hash + 8B key引用 + 8B value引用 + 8B next = 44B（~48B，对齐）
  平均需要 table.length * 8B 的桶数组空间
  负载因子0.75时，约 1.33 * size 个桶

LinkedHashMap（每个 Entry）：
  = HashMap.Node + 8B before + 8B after = ~64B
  额外 16B/元素 相比 HashMap

TreeMap（每个 Entry）：
  = 16B头 + 8B key + 8B value + 8B left + 8B right + 8B parent + 1B color = ~64B

HashSet：
  = HashMap（key=元素，value=PRESENT对象）
  额外一个共享 PRESENT 对象（16B，只有一个）

内存效率排名（从低到高）：
ArrayList < HashMap < LinkedList < LinkedHashMap ≈ TreeMap
`

## 11.3 选型决策树

`
需要存储一组元素？
  |
  +-- 需要键值对（Map）？
  |     |
  |     +-- 需要线程安全？
  |     |     YES --> ConcurrentHashMap（推荐）
  |     |             或 Collections.synchronizedMap（低并发）
  |     |
  |     +-- 需要按键排序或范围查询？
  |     |     YES --> TreeMap
  |     |
  |     +-- 需要维护插入顺序？
  |     |     YES --> LinkedHashMap
  |     |
  |     +-- 其他（通用）
  |           --> HashMap（默认选择）
  |
  +-- 只需要键（Set）？
  |     |
  |     +-- 需要按元素排序？
  |     |     YES --> TreeSet
  |     |
  |     +-- 需要维护插入顺序？
  |     |     YES --> LinkedHashSet
  |     |
  |     +-- 其他（通用去重）
  |           --> HashSet（默认选择）
  |
  +-- 需要有序列表（List）？
  |     |
  |     +-- 主要操作是随机访问（get/set）？
  |     |     YES --> ArrayList
  |     |
  |     +-- 主要操作是头部/中间频繁插入删除？
  |     |     YES --> LinkedList（但大多数情况 ArrayList 仍更快）
  |     |
  |     +-- 其他（通用列表）
  |           --> ArrayList（默认选择，几乎所有情况）
  |
  +-- 需要队列/栈？
        |
        +-- 优先级队列（按优先级出队）？
        |     YES --> PriorityQueue
        |
        +-- 线程安全的队列？
        |     YES --> ArrayBlockingQueue / LinkedBlockingQueue
        |
        +-- 作为栈使用？
        |     YES --> ArrayDeque（不要用 Stack！）
        |
        +-- 一般队列？
              YES --> ArrayDeque（比 LinkedList 快）
`

## 11.4 实际项目中的最佳实践

`java
import java.util.*;
import java.util.concurrent.*;

public class BestPracticesDemo {

    // 实践1：声明时使用接口类型（而非具体实现）
    // 好：灵活，可以轻松替换实现
    List<String> list = new ArrayList<>();
    Map<String, Integer> map = new HashMap<>();
    Set<String> set = new HashSet<>();

    // 坏：绑定具体实现，不灵活
    // ArrayList<String> list = new ArrayList<>(); // 避免

    // 实践2：预估大小，避免扩容（性能敏感场景）
    void loadData(int expectedSize) {
        // 构造时指定初始容量
        List<String> data = new ArrayList<>(expectedSize);
        // HashMap 的初始容量 = 预期元素数 / 负载因子 + 1
        Map<String, Integer> index = new HashMap<>((int)(expectedSize / 0.75f) + 1);
    }

    // 实践3：优先使用不可变集合（防止意外修改）
    void returnImmutable() {
        List<String> result = new ArrayList<>(Arrays.asList("a", "b", "c"));
        // Java 9+
        return; // return List.copyOf(result); 或 List.of("a","b","c")
        // Java 8
        // return Collections.unmodifiableList(result);
    }

    // 实践4：遍历 Map 时使用 entrySet（而非 keySet + get）
    void iterateMap(Map<String, Integer> map) {
        // 好：一次遍历
        for (Map.Entry<String, Integer> entry : map.entrySet()) {
            System.out.println(entry.getKey() + ": " + entry.getValue());
        }

        // 坏：两次哈希查找
        for (String key : map.keySet()) {
            System.out.println(key + ": " + map.get(key)); // 多了一次 get 查找
        }
    }

    // 实践5：多线程场景使用线程安全集合
    ConcurrentHashMap<String, Integer> sharedMap = new ConcurrentHashMap<>();
    CopyOnWriteArrayList<String> sharedList = new CopyOnWriteArrayList<>();

    // 实践6：不要在 for-each 中修改集合（用 removeIf 或 Iterator.remove）
    void removeElements(List<String> list) {
        // 好：
        list.removeIf(s -> s.startsWith("X"));
        // 或
        Iterator<String> it = list.iterator();
        while (it.hasNext()) {
            if (it.next().startsWith("X")) it.remove();
        }
        // 坏：
        // for (String s : list) {
        //     if (s.startsWith("X")) list.remove(s); // ConcurrentModificationException!
        // }
    }

    // 实践7：Map 计数/分组推荐使用 merge/computeIfAbsent
    Map<String, Integer> countWords(String[] words) {
        Map<String, Integer> freq = new HashMap<>();
        // 好：
        for (String w : words) freq.merge(w, 1, Integer::sum);
        // 可读但有冗余：
        // for (String w : words) freq.put(w, freq.getOrDefault(w, 0) + 1);
        return freq;
    }

    // 实践8：避免自动装箱的性能问题
    // 如果需要存储大量 int/long，考虑使用第三方库（如 Eclipse Collections）
    // 或手动管理原始类型数组
    long sumList(List<Integer> list) {
        // 有装箱拆箱开销
        return list.stream().mapToLong(Integer::longValue).sum();
        // 或
        long sum = 0;
        for (int v : list) sum += v; // 自动拆箱
        return sum;
    }
}
`

## 11.5 面试常问：各集合区别对比

`
ArrayList vs LinkedList（频繁考点）：
  ArrayList 底层数组，随机访问 O(1)，中间插入 O(n)，内存紧凑
  LinkedList 底层双向链表，随机访问 O(n)，头尾操作 O(1)，内存分散
  结论：绝大多数场景用 ArrayList，LinkedList 仅在需要频繁头部插入时考虑

HashMap vs HashTable（常问）：
  HashMap 非线程安全，允许 null key/value，性能好，JDK 1.2
  Hashtable 线程安全（同步方法），不允许 null，性能差，JDK 1.0（已过时）
  结论：多线程用 ConcurrentHashMap，单线程用 HashMap

HashMap vs LinkedHashMap vs TreeMap（常问）：
  HashMap 无序，O(1)操作，最常用
  LinkedHashMap 插入/访问顺序，O(1)操作，LRU缓存
  TreeMap 按键排序，O(log n)操作，范围查询

HashSet vs LinkedHashSet vs TreeSet：
  与上面 Map 对应（底层分别用 HashMap/LinkedHashMap/TreeMap）

ArrayList vs Vector（常问）：
  ArrayList 非线程安全，1.5倍扩容，JDK 1.2，推荐
  Vector 线程安全（同步方法），2倍扩容，JDK 1.0，已过时

ConcurrentHashMap vs Hashtable vs synchronizedMap（常问）：
  ConcurrentHashMap 桶级锁+CAS，高并发首选
  Hashtable 整体锁，已过时
  synchronizedMap 整体锁包装器，低并发或快速迁移
`


---

# Part 12: 常见面试题 FAQ

## Q1：ArrayList 和 LinkedList 的区别？什么时候用哪个？

**答：**

| 方面         | ArrayList                    | LinkedList               |
|------------|------------------------------|--------------------------|
| 底层结构     | Object[] 动态数组              | 双向链表（Node 节点）      |
| 随机访问     | O(1)（数组下标访问）            | O(n)（从头/尾遍历）        |
| 头尾插入     | O(n)（头部）/ O(1)均摊（尾部）  | O(1)                     |
| 中间插入删除  | O(n)                         | O(n)（找到位置）           |
| 内存占用     | 低（只有元素引用）              | 高（每节点额外 prev/next）  |
| 标记接口     | RandomAccess（支持快速随机访问）| 未实现 RandomAccess       |
| 额外功能     | 无                            | 实现了 Deque，可作栈/队列  |

**使用原则：**
- 绝大多数情况用 ArrayList（读多写少，随机访问多）
- 频繁在**头部**或**已知位置**插入/删除时，LinkedList 可能更好
- 作为队列/栈时，用 ArrayDeque（比 LinkedList 更快）

---

## Q2：HashMap 的 put 流程是什么？

**答：**

`
1. 计算 key 的 hash：hash = (h = key.hashCode()) ^ (h >>> 16)
   （高位与低位异或，减少碰撞）

2. 如果 table 为空，先 resize() 初始化（默认容量16）

3. 计算桶位置：index = (table.length - 1) & hash

4. 如果 table[index] == null，直接创建新节点放入

5. 如果 table[index] 不为空：
   a. 头节点的 key 相同 → 更新 value
   b. 头节点是 TreeNode（红黑树）→ 调用 putTreeVal()
   c. 头节点是链表节点 → 遍历链表：
      - 找到相同 key → 更新 value
      - 遍历到末尾 → 尾插新节点
        若链表长度 >= 8 → treeifyBin()（先检查数组长度>=64，否则 resize）

6. 若是新增：size++ 后如果 size > threshold（capacity * loadFactor），触发 resize()
`

---

## Q3：HashMap 的扩容机制？

**答：**

- **触发条件：** size > threshold（threshold = capacity * loadFactor = 16 * 0.75 = 12）
- **新容量：** oldCapacity << 1（翻倍，必须是 2 的幂）
- **JDK 8 优化（高低位拆分）：**
  - 不需要重新计算 hash
  - 通过 (hash & oldCapacity) 判断新位置
  - 结果为 0 → 新位置 = 原位置（低位链表）
  - 结果非0 → 新位置 = 原位置 + oldCapacity（高位链表）
  - 使用尾插法，保持链表顺序（避免 JDK 7 头插法的死循环）

---

## Q4：HashMap 为什么线程不安全？

**答：**

**JDK 7 问题：** 多线程扩容时，链表头插法可能导致链表成环，get 操作陷入死循环（CPU 100%）。

**JDK 8 问题：** 虽然改用尾插法避免了死循环，但仍有数据丢失风险：

`java
// 两个线程同时 put 到空桶
if ((p = tab[i = (n-1) & hash]) == null)
    tab[i] = newNode(...); // 非原子操作！线程1和2都判断为null，后者覆盖前者
`

另外，size++ 也不是原子操作，可能导致 size 计数不准确。

**解决方案：** 使用 ConcurrentHashMap。

---

## Q5：ConcurrentHashMap 和 Hashtable 的区别？

**答：**

| 特性          | Hashtable         | ConcurrentHashMap（JDK8） |
|-------------|-------------------|--------------------------|
| 锁粒度        | 整个对象（方法级synchronized）| 单个桶（synchronized桶头节点） |
| 并发度        | 1（串行）           | 数组长度（可达数千）        |
| null key/value | 不允许            | 不允许                    |
| 性能          | 差                 | 好（细粒度并发）            |
| 空桶插入      | 加锁               | CAS 无锁                  |
| size 统计     | 整个对象加锁        | LongAdder 分段累加（近似）  |
| 推荐使用      | 否（已过时）        | 是                        |

---

## Q6：HashMap 的初始容量设置多少合适？

**答：**

如果已知要存储 n 个元素，建议初始容量设为 
 / 0.75 + 1，即 (int)(n / 0.75f) + 1。

`java
// 存储1000个元素，避免扩容
int n = 1000;
Map<String, Integer> map = new HashMap<>((int)(n / 0.75f) + 1);
// 约 1334，HashMap 会取大于此值的最小 2 的幂 = 2048

// 不设置时：
// 默认容量16，阈值12
// 1000个元素需要扩容：16→32→64→128→256→512→1024→2048
// 共扩容7次，每次都要重新哈希，性能损失明显
`

---

## Q7：HashMap 的 key 要求是什么？

**答：**

1. **必须正确重写 hashCode() 和 equals()**（符合契约）
   - .equals(b) 为 true 时，.hashCode() == b.hashCode() 必须成立
   
2. **建议使用不可变对象作为 key**
   - 可变对象放入 HashMap 后，如果修改了参与 hashCode/equals 的字段
   - 导致 hash 值改变，再也找不到这个 key（"幽灵键"）

3. **最佳 key：** String、Integer 等不可变类（JDK 已正确实现）

`java
Map<List<String>, Integer> map = new HashMap<>();
List<String> key = new ArrayList<>(Arrays.asList("a", "b"));
map.put(key, 1);
key.add("c"); // 修改了 key！hashCode 变化
System.out.println(map.get(key)); // null！找不到了！
`

---

## Q8：红黑树为什么要在链表长度8时树化？

**答：**

1. **概率极低：** 使用负载因子0.75时，链表长度达到8的概率约 6×10⁻⁸，正常情况下几乎不会树化

2. **性能权衡：**
   - 链表：O(n)，常数小，结构简单
   - 红黑树：O(log n)，常数大（旋转、染色等操作）
   - n=8 时：链表8步 vs 红黑树3步（log₂8=3），树的优势开始明显

3. **退树化阈值=6（缓冲区设计）：** 防止在7/8元素附近频繁树化和退化，减少不必要的转换开销

---

## Q9：HashMap 和 TreeMap 如何选择？

**答：**

- **HashMap：** 通用场景，O(1) 操作，无序，99%的情况首选
- **TreeMap：** 以下情况使用：
  1. 需要按 key 自然/自定义顺序遍历
  2. 需要范围查询（subMap/headMap/tailMap）
  3. 需要找最近的 key（floorKey/ceilingKey）
  4. 需要按序获取最大/最小 key

---

## Q10：如何实现 LRU 缓存？

**答：**

**方案1（简单）：** 继承 LinkedHashMap，重写 emoveEldestEntry

`java
class LRUCache<K,V> extends LinkedHashMap<K,V> {
    private final int capacity;
    LRUCache(int cap) {
        super(cap, 0.75f, true); // accessOrder=true
        this.capacity = cap;
    }
    @Override
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return size() > capacity;
    }
}
`

**方案2（手动，LeetCode 标准解法）：** HashMap + 双向链表

`java
class LRUCache {
    private Map<Integer, Node> map = new HashMap<>();
    private Node head = new Node(-1, -1); // 虚拟头节点（最旧）
    private Node tail = new Node(-1, -1); // 虚拟尾节点（最新）
    private int capacity;

    LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        remove(node);        // 从当前位置移除
        addToTail(node);     // 移到尾部（最近使用）
        return node.val;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) {
            Node node = map.get(key);
            node.val = value;
            remove(node);
            addToTail(node);
        } else {
            if (map.size() == capacity) {
                Node oldest = head.next; // 最久未使用（虚拟头的下一个）
                remove(oldest);
                map.remove(oldest.key);
            }
            Node node = new Node(key, value);
            addToTail(node);
            map.put(key, node);
        }
    }

    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void addToTail(Node node) {
        tail.prev.next = node;
        node.prev = tail.prev;
        node.next = tail;
        tail.prev = node;
    }

    class Node {
        int key, val;
        Node prev, next;
        Node(int k, int v) { key = k; val = v; }
    }
}
`

---

## Q11：HashSet 如何保证元素不重复？

**答：**

HashSet 底层是 HashMap，元素作为 key，PRESENT 对象作为 value。

当调用 dd(e) 时：
1. 计算 e.hashCode() 并扰动
2. 定位桶
3. 桶为空 → 直接插入（新元素）
4. 桶不空 → 检查链表/树中是否有 hash 相同 && equals 相同 的元素
   - 有 → 更新 value（仍是 PRESENT），map.put 返回旧 PRESENT，dd 返回 alse
   - 无 → 插入新节点，map.put 返回 null，dd 返回 	rue

**关键：必须同时正确重写 hashCode 和 equals！**

---

## Q12：fail-fast 和 fail-safe 的区别？

**答：**

- **fail-fast（快速失败）：** java.util 包的集合，遍历时结构被修改（增删）立即抛 ConcurrentModificationException。实现：modCount != expectedModCount 时抛异常。代表：ArrayList、HashMap

- **fail-safe（安全失败）：** java.util.concurrent 包的集合，遍历时不抛异常，遍历的是**快照**或使用弱一致性机制。代表：CopyOnWriteArrayList、ConcurrentHashMap

---

## Q13：ArrayList 的扩容机制？

**答：**

1. 默认构造：空数组（懒初始化，第一次 add 才分配）
2. 第一次 add：扩容到 DEFAULT_CAPACITY = 10
3. 之后每次容量不够：
ewCapacity = oldCapacity + (oldCapacity >> 1) = 1.5倍
4. 底层：Arrays.copyOf（调用 System.arraycopy）复制到新数组
5. 均摊 O(1) 的分析：每次扩容分摊到之前所有 add 操作

---

## Q14：PriorityQueue 的底层原理？

**答：**

PriorityQueue 底层是**二叉堆（Binary Heap）**，默认小顶堆。

- **数组存储：** queue[0] 是堆顶（最小元素），父子关系：parent(i) = (i-1)/2，left(i) = 2i+1
- **offer(e)：** 插入末尾，然后 siftUp（向上比较交换），O(log n)
- **poll()：** 取出堆顶，将末尾元素移到堆顶，然后 siftDown（向下比较交换），O(log n)
- **peek()：** 直接返回 queue[0]，O(1)
- **不保证遍历顺序：** 只保证堆顶是最小/最大元素，不是全局有序

---

## Q15：TreeMap 和 TreeSet 的底层原理？

**答：**

- TreeMap 底层是**红黑树**，按 key 的 Comparable/Comparator 排序
- TreeSet 底层是 TreeMap（元素作为 key，PRESENT 作为 value）
- 所有操作（put/get/remove/contains）时间复杂度 O(log n)
- TreeMap 不允许 null key（会触发 compareTo(null) 抛 NPE）
- 提供 NavigableMap/NavigableSet 接口的范围查询方法

---

## Q16：Collections.sort 和 Arrays.sort 用的什么排序算法？

**答：**

- Arrays.sort(int[])：**双轴快速排序（Dual-Pivot Quicksort）**，不稳定，平均 O(n log n)
- Arrays.sort(Object[])：**TimSort**（改良归并排序），稳定，最好 O(n)，最坏 O(n log n)
- Collections.sort(List<T>)：内部调用 list.sort(null)，底层是 TimSort，稳定
- List.sort(Comparator)：同上

---

## Q17：如何删除 ArrayList 中的所有偶数元素？

**答：**

`java
List<Integer> list = new ArrayList<>(Arrays.asList(1,2,3,4,5,6));

// 方式1：Iterator.remove()（经典）
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    if (it.next() % 2 == 0) it.remove();
}

// 方式2：removeIf（Java 8，推荐）
list.removeIf(n -> n % 2 == 0);

// 方式3：倒序遍历
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i) % 2 == 0) list.remove(i);
}

// 错误方式：for-each 直接 remove → ConcurrentModificationException
`

---

## Q18：HashMap 中 key 为 null 时如何处理？

**答：**

`java
// hash(null) = 0，null key 存储在 table[0] 对应的桶中
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// HashMap 允许一个 null key，也允许 null value
// Hashtable、ConcurrentHashMap、TreeMap 不允许 null key（会抛 NPE）
`

---

## Q19：如何将 Map 按 value 排序？

**答：**

`java
import java.util.*;
import java.util.stream.*;

Map<String, Integer> scores = Map.of("Alice", 95, "Bob", 87, "Charlie", 92);

// 方式1：Java 8 Stream（推荐）
List<Map.Entry<String, Integer>> sorted = scores.entrySet().stream()
    .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
    .collect(Collectors.toList());
sorted.forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));

// 方式2：toList 再转 LinkedHashMap（保持有序）
Map<String, Integer> sortedMap = scores.entrySet().stream()
    .sorted(Map.Entry.comparingByValue(Comparator.reverseOrder()))
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        Map.Entry::getValue,
        (e1, e2) -> e1,
        LinkedHashMap::new
    ));
System.out.println(sortedMap); // {Alice=95, Charlie=92, Bob=87}
`

---

## Q20：ConcurrentHashMap 在 JDK 8 中如何保证线程安全？

**答：**

JDK 8 的 ConcurrentHashMap 使用以下机制保证线程安全：

1. **CAS 无锁操作：**
   - 初始化 table：CAS(SIZECTL, sc, -1) 确保只有一个线程初始化
   - 空桶插入：casTabAt(tab, i, null, newNode) 原子插入

2. **synchronized 细粒度锁：**
   - 桶不为空时：synchronized(f) 锁定桶头节点
   - 锁粒度是单个桶，不同桶的操作可以并行

3. **volatile 保证可见性：**
   - Node.val 和 Node.next 均为 olatile
   - get 操作通过 volatile 读保证可见性，无需加锁

4. **ForwardingNode 机制：**
   - 扩容期间，已迁移的桶放置 ForwardingNode（hash=MOVED）
   - get 遇到 MOVED 时，调用 ind() 在新数组中查找
   - put 遇到 MOVED 时，调用 helpTransfer() 参与扩容

5. **LongAdder 分段计数：**
   - aseCount（无竞争时 CAS 修改）+ CounterCell[]（竞争时分散计数）
   - size() = baseCount + sum(所有 CounterCell)

---

## Q21：LinkedHashMap 的 accessOrder=true 是什么意思？

**答：**

当 ccessOrder=true 时，每次 get 或 put 操作访问到某个节点，该节点会被移到双向链表的尾部（最近使用）。链表头部始终是最久未访问的节点。

这是实现 **LRU 缓存** 的核心机制：
- 链表尾部 = 最近使用（MRU）
- 链表头部 = 最久未使用（LRU 淘汰候选）
- 重写 emoveEldestEntry 在容量超限时自动淘汰链表头部节点

默认 ccessOrder=false（插入顺序），普通遍历场景使用。

---

## Q22：为什么 HashMap 的容量必须是 2 的幂次方？

**答：**

三个原因：

1. **高效取模：** hash % n 等价于 (n-1) & hash（n 是 2 的幂时成立），位运算比取模运算快约 5-10 倍

2. **高低位拆分优化扩容：** 扩容时无需重新计算 hash，只需检查 (hash & oldCapacity) 是否为 0，确定元素的新位置（见 resize 源码）

3. **元素均匀分布：** n 是 2 的幂时，(n-1) 是全1掩码，& hash 能充分利用 hash 的低位信息，使元素分布均匀

---

## Q23：解释 HashMap 中的 modCount？

**答：**

modCount 是集合的**结构性修改次数计数器**，用于实现 fail-fast 机制：

`java
transient int modCount; // 每次 put/remove/clear/rehash 时递增

// 迭代器创建时记录 modCount
int expectedModCount = modCount;

// 每次 next() 时检查
void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}

// Iterator.remove() 会同步 expectedModCount
public void remove() {
    // ...删除操作...
    expectedModCount = modCount; // 同步，不会抛异常
}
`

注意：fail-fast 是**尽力检测**，不保证一定能检测到所有并发修改（JVM 不提供强保证）。多线程环境必须使用 ConcurrentHashMap。

---

## Q24：subList 有什么陷阱？

**答：**

1. subList 返回的是**视图**（View），不是副本，修改视图会影响原列表

2. 对**原列表**进行结构性修改（add/remove）后，再访问视图会抛 ConcurrentModificationException

3. 不能直接序列化视图（ArrayList.SubList 没有独立序列化）

**安全使用方式：**
`java
// 如果需要独立副本
List<String> copy = new ArrayList<>(original.subList(1, 4));

// 如果需要删除一段元素（利用视图特性）
original.subList(1, 4).clear();
`

---

## Q25：TreeMap 为什么不允许 null key？

**答：**

TreeMap 通过 Comparable.compareTo() 或 Comparator.compare() 来比较 key，以维护红黑树的有序性。

当 key 为 null 时：
- 使用自然排序：
ull.compareTo(other) → NullPointerException
- 使用 Comparator：取决于 Comparator 的实现，大多数不处理 null

因此 TreeMap 不允许 null key。如果需要允许 null，可以提供处理 null 的 Comparator：

`java
TreeMap<String, Integer> map = new TreeMap<>(
    Comparator.nullsFirst(Comparator.naturalOrder())
);
map.put(null, 0); // 合法
map.put("a", 1);
System.out.println(map.firstKey()); // null
`

---

# 附录：Java 集合框架核心知识图谱

`
Java Collections Framework 核心知识总结

一、底层数据结构
  ArrayList   -> Object[] 动态数组
  LinkedList  -> 双向链表 (Node: prev + item + next)
  HashMap     -> 数组 + 链表 + 红黑树 (JDK8)
  LinkedHashMap -> HashMap + 双向链表
  TreeMap     -> 红黑树
  HashSet     -> HashMap (PRESENT 占位)
  TreeSet     -> TreeMap (PRESENT 占位)
  ArrayDeque  -> 循环数组 (head/tail 指针)
  PriorityQueue -> 二叉堆 (小顶堆)
  ConcurrentHashMap -> 数组 + 链表 + 红黑树 + CAS + synchronized

二、核心设计原理
  fail-fast   -> modCount 机制，遍历时检测并发修改
  fail-safe   -> 遍历快照，允许并发修改（弱一致性）
  LRU 缓存    -> LinkedHashMap(accessOrder=true) + removeEldestEntry
  哈希设计    -> hash(key) = hashCode ^ (hashCode >>> 16)（减少低位碰撞）
  容量2的幂   -> 位运算取模 + 扩容高低位拆分
  负载因子0.75 -> 泊松分布下时间/空间最优平衡

三、线程安全选择
  读多写少    -> CopyOnWriteArrayList / CopyOnWriteArraySet
  高并发Map   -> ConcurrentHashMap（首选）
  并发队列    -> ArrayBlockingQueue / LinkedBlockingQueue
  避免使用    -> Hashtable, Vector, Stack（已过时）

四、时间复杂度速记
  O(1):  ArrayList.get/set, HashMap/HashSet 操作, ArrayDeque 头尾
  O(log n): TreeMap/TreeSet 操作, PriorityQueue 入队/出队
  O(n):  ArrayList 中间插入, LinkedList.get, 所有 contains（Hash系除外）
`

---

> **本文档完整覆盖 Java 集合框架的核心知识点，建议结合 JDK 源码对照学习。**
> **作者：Java 技术文档 | 版本：JDK 8/11/17 通用**

