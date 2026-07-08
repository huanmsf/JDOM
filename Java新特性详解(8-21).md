# Java 新特性详解（Java 8 到 Java 21）

> 本文档覆盖 Java 8 到 Java 21 所有重要新特性，每个特性配有完整可运行代码示例。
> 适用于：深度学习、面试备考、项目升级参考。

---

## 目录

- [Part 1: Java 8 核心特性](#part-1-java-8-核心特性)
- [Part 2: Java 9 特性](#part-2-java-9-特性)
- [Part 3: Java 10 特性](#part-3-java-10-特性)
- [Part 4: Java 11 特性（LTS）](#part-4-java-11-特性lts)
- [Part 5: Java 14-16 语法糖](#part-5-java-14-16-语法糖重点)
- [Part 6: Java 17 特性（LTS）](#part-6-java-17-特性lts-重点)
- [Part 7: Java 21 特性（LTS）](#part-7-java-21-特性lts-重点)
- [Part 8: Java 18-20 其他特性](#part-8-java-18-20-其他特性)
- [Part 9: 各版本新特性对比速查表](#part-9-各版本新特性对比速查表)
- [Part 10: 实战整合案例](#part-10-实战整合案例)
- [Part 11: 面试题 FAQ](#part-11-面试题-faq)

---

## Part 1: Java 8 核心特性

### 1.1 Lambda 表达式

#### 1.1.1 函数式接口 @FunctionalInterface

函数式接口是 Lambda 表达式的基础，有且仅有一个抽象方法（可以有 default 方法和 static 方法）。

```java
@FunctionalInterface
public interface Calculator {
    int calculate(int a, int b);

    default Calculator andThen(Calculator after) {
        return (a, b) -> after.calculate(this.calculate(a, b), 0);
    }

    static Calculator add() {
        return (a, b) -> a + b;
    }
}
```

#### 1.1.2 四大内置函数式接口

```java
import java.util.function.*;

public class FunctionalInterfaceDemo {

    public static void main(String[] args) {

        // ========== Function<T, R>：接收T，返回R ==========
        Function<String, Integer> strToLen = s -> s.length();
        Function<Integer, Integer> doubleIt = n -> n * 2;

        // andThen：先执行当前函数，再执行参数函数
        // compose：先执行参数函数，再执行当前函数
        Function<String, Integer> lenThenDouble = strToLen.andThen(doubleIt);
        System.out.println(lenThenDouble.apply("Hello"));  // 10

        Function<Integer, Integer> doubleThenAdd = doubleIt.compose(n -> n + 5);
        System.out.println(doubleThenAdd.apply(3));  // (3+5)*2 = 16

        // ========== Consumer<T>：接收T，无返回 ==========
        Consumer<String> print = System.out::println;
        Consumer<String> printUpper = s -> System.out.println(s.toUpperCase());
        Consumer<String> printBoth = print.andThen(printUpper);
        printBoth.accept("hello");  // 打印 hello 和 HELLO

        BiConsumer<String, Integer> repeatPrint = (s, n) -> {
            for (int i = 0; i < n; i++) System.out.print(s + " ");
            System.out.println();
        };
        repeatPrint.accept("Java", 3);  // Java Java Java

        // ========== Supplier<T>：无参数，返回T ==========
        Supplier<String> greet = () -> "Hello, World!";
        Supplier<Double> randomNum = Math::random;
        Supplier<java.util.List<String>> listFactory = java.util.ArrayList::new;

        System.out.println(greet.get());
        System.out.println(randomNum.get());

        // ========== Predicate<T>：接收T，返回boolean ==========
        Predicate<String> isEmpty = String::isEmpty;
        Predicate<String> isLong = s -> s.length() > 5;
        Predicate<Integer> isPositive = n -> n > 0;
        Predicate<Integer> isEven = n -> n % 2 == 0;

        Predicate<String> isNotEmpty = isEmpty.negate();
        Predicate<String> isEmptyOrLong = isEmpty.or(isLong);
        Predicate<Integer> isPositiveEven = isPositive.and(isEven);

        System.out.println(isNotEmpty.test(""));               // false
        System.out.println(isEmptyOrLong.test("Hello World")); // true
        System.out.println(isPositiveEven.test(4));            // true
        System.out.println(isPositiveEven.test(-2));           // false

        // BiFunction<T, U, R>：两个参数
        BiFunction<String, String, String> concat = (a, b) -> a + " " + b;
        System.out.println(concat.apply("Hello", "Java")); // Hello Java

        // UnaryOperator<T>：T -> T
        UnaryOperator<String> trim = String::trim;
        UnaryOperator<Integer> square = n -> n * n;
        System.out.println(trim.apply("  hello  "));  // hello
        System.out.println(square.apply(5));           // 25

        // BinaryOperator<T>：(T, T) -> T
        BinaryOperator<Integer> max = (a, b) -> a > b ? a : b;
        System.out.println(max.apply(3, 7));  // 7

        // 基本类型特化（避免装箱拆箱）
        IntFunction<String> intToStr = i -> "Number: " + i;
        ToIntFunction<String> strLen = String::length;
        IntUnaryOperator triple = i -> i * 3;
        IntBinaryOperator add = Integer::sum;

        System.out.println(intToStr.apply(42));      // Number: 42
        System.out.println(strLen.applyAsInt("Hi")); // 2
        System.out.println(triple.applyAsInt(4));    // 12
        System.out.println(add.applyAsInt(3, 5));    // 8
    }
}
```

#### 1.1.3 Lambda 语法全形式

```java
import java.util.*;
import java.util.function.*;

public class LambdaSyntaxDemo {

    public static void main(String[] args) {

        // ===== 无参数 =====
        Runnable r1 = () -> System.out.println("No args");
        Supplier<String> s1 = () -> "hello";

        // ===== 单参数（括号可省略）=====
        Consumer<String> c1 = s -> System.out.println(s);
        Consumer<String> c2 = (s) -> System.out.println(s);
        Function<Integer, Integer> f1 = x -> x * 2;

        // ===== 多参数 =====
        BiFunction<Integer, Integer, Integer> b1 = (x, y) -> x + y;
        Comparator<String> comp = (a, b) -> a.compareTo(b);

        // ===== 显式类型声明 =====
        BiFunction<String, Integer, String> b2 = (String str, Integer n) -> str.repeat(n);

        // ===== 代码块 Lambda（多语句）=====
        Function<Integer, String> f2 = n -> {
            if (n < 0) return "负数";
            else if (n == 0) return "零";
            else return "正数";
        };

        r1.run();
        System.out.println(s1.get());
        c1.accept("Hello");
        System.out.println(f1.apply(5));         // 10
        System.out.println(b1.apply(3, 4));      // 7
        System.out.println(b2.apply("Java", 3)); // JavaJavaJava
        System.out.println(f2.apply(-5));        // 负数
        System.out.println(f2.apply(0));         // 零
        System.out.println(f2.apply(3));         // 正数

        // ===== 变量捕获（effectively final）=====
        String prefix = "Hello, ";  // effectively final
        Function<String, String> greet = name -> prefix + name;
        System.out.println(greet.apply("Java")); // Hello, Java

        // prefix = "Hi, ";  // 编译错误：Lambda 已捕获 prefix

        final int[] counter = {0};  // 数组绕过限制（不推荐）
        Runnable increment = () -> counter[0]++;
        increment.run();
        System.out.println(counter[0]);  // 1
    }
}
```

#### 1.1.4 方法引用四种形式

```java
import java.util.*;
import java.util.function.*;
import java.util.stream.*;

public class MethodReferenceDemo {

    static String staticMethod(String s) {
        return s.toUpperCase();
    }

    String instanceMethod(String s) {
        return s + " from instance";
    }

    public static void main(String[] args) {
        MethodReferenceDemo demo = new MethodReferenceDemo();

        // ===== 1. 静态方法引用：ClassName::staticMethod =====
        Function<String, String> f1 = MethodReferenceDemo::staticMethod;
        // 等价于：s -> MethodReferenceDemo.staticMethod(s)
        System.out.println(f1.apply("hello"));  // HELLO

        Function<Double, Double> abs = Math::abs;
        Function<String, Integer> parseInt = Integer::parseInt;
        System.out.println(abs.apply(-3.14));     // 3.14
        System.out.println(parseInt.apply("42")); // 42

        // ===== 2. 特定实例的方法引用：instance::instanceMethod =====
        Function<String, String> f2 = demo::instanceMethod;
        // 等价于：s -> demo.instanceMethod(s)
        System.out.println(f2.apply("Java"));  // Java from instance

        String str = "Hello World";
        Supplier<String> toUpper = str::toUpperCase;
        System.out.println(toUpper.get());  // HELLO WORLD

        Consumer<String> printer = System.out::println;
        printer.accept("Method Reference");

        // ===== 3. 任意实例的方法引用：ClassName::instanceMethod =====
        // 第一个参数成为调用方法的对象
        Function<String, String> f3 = String::toUpperCase;
        // 等价于：s -> s.toUpperCase()
        System.out.println(f3.apply("hello"));  // HELLO

        BiFunction<String, String, Boolean> startsWith = String::startsWith;
        System.out.println(startsWith.apply("Hello", "He")); // true

        Comparator<String> cmp = String::compareTo;
        List<String> names = Arrays.asList("Charlie", "Alice", "Bob");
        names.sort(cmp);
        System.out.println(names); // [Alice, Bob, Charlie]

        // ===== 4. 构造器引用：ClassName::new =====
        Supplier<ArrayList<String>> listFactory = ArrayList::new;
        Function<Integer, ArrayList<String>> listWithCapacity = ArrayList::new;
        Function<String, StringBuilder> sbFactory = StringBuilder::new;
        System.out.println(sbFactory.apply("Hello"));  // Hello

        // 数组构造器引用
        Function<Integer, String[]> arrayFactory = String[]::new;
        String[] arr = arrayFactory.apply(5);
        System.out.println(arr.length);  // 5

        // 综合示例
        List<String> words = Arrays.asList("java", "stream", "lambda", "method", "reference");
        List<String> result = words.stream()
            .filter(s -> s.length() > 4)
            .map(String::toUpperCase)               // 任意实例方法引用
            .sorted(String::compareTo)              // 任意实例方法引用
            .collect(Collectors.toList());
        System.out.println(result);
    }
}
```

#### 1.1.5 Lambda vs 匿名内部类

```java
public class LambdaVsAnonymous {

    String name = "Outer";

    @FunctionalInterface
    interface Greeter { void greet(); }

    void demonstrate() {
        String localName = "Local";

        // ===== 匿名内部类 =====
        Greeter anon = new Greeter() {
            String name = "Anonymous"; // 有自己的字段

            @Override
            public void greet() {
                // this 指向匿名内部类实例
                System.out.println("Anonymous this.name: " + this.name);
                System.out.println("Outer name: " + LambdaVsAnonymous.this.name);
                System.out.println("Local name: " + localName);
            }
        };

        // ===== Lambda =====
        Greeter lambda = () -> {
            // this 指向外部类实例
            System.out.println("Lambda this.name: " + this.name); // Outer
            System.out.println("Local name: " + localName);
        };

        anon.greet();
        lambda.greet();

        /*
         * 关键区别：
         * 1. this 指向：匿名内部类 this=自身；Lambda this=外部类
         * 2. 字节码：匿名内部类->独立 .class；Lambda->invokedynamic（运行时生成）
         * 3. 变量捕获：两者都要求 effectively final
         * 4. Lambda 只适用于函数式接口；匿名内部类可实现任意接口/继承类
         */
    }

    public static void main(String[] args) {
        new LambdaVsAnonymous().demonstrate();
    }
}
```
---

### 1.2 Stream API（重中之重）

#### 1.2.1 Stream 创建

```java
import java.util.*;
import java.util.stream.*;
import java.io.*;
import java.nio.file.*;

public class StreamCreationDemo {

    public static void main(String[] args) throws IOException {

        // 1. 集合创建
        List<String> list = Arrays.asList("a", "b", "c", "d");
        Stream<String> stream1 = list.stream();           // 顺序流
        Stream<String> stream2 = list.parallelStream();   // 并行流

        // 2. 数组创建
        String[] arr = {"x", "y", "z"};
        Stream<String> stream3 = Arrays.stream(arr);
        Stream<String> stream4 = Arrays.stream(arr, 0, 2); // 指定范围 [0, 2)
        int[] intArr = {1, 2, 3, 4, 5};
        IntStream intStream = Arrays.stream(intArr);

        // 3. Stream.of()
        Stream<String> stream5 = Stream.of("Hello", "World", "Java");
        Stream<Integer> stream6 = Stream.of(1, 2, 3, 4, 5);
        Stream<String> emptyStream = Stream.empty();

        // 4. Stream.generate()（无限流，需要 limit 截断）
        Stream<Double> randoms = Stream.generate(Math::random).limit(5);
        Stream<String> constants = Stream.generate(() -> "Java").limit(3);
        randoms.forEach(d -> System.out.printf("%.4f ", d));
        System.out.println();
        constants.forEach(System.out::print);
        System.out.println();

        // 5. Stream.iterate()
        // Java 8：iterate(seed, UnaryOperator) 无限流
        Stream<Integer> naturals = Stream.iterate(0, n -> n + 1).limit(10);
        naturals.forEach(n -> System.out.print(n + " "));
        System.out.println();

        // Java 9：iterate(seed, hasNext, next) 有限流
        Stream<Integer> range = Stream.iterate(0, n -> n < 10, n -> n + 2);
        range.forEach(n -> System.out.print(n + " ")); // 0 2 4 6 8
        System.out.println();

        // 斐波那契数列
        Stream.iterate(new long[]{0, 1}, f -> new long[]{f[1], f[0] + f[1]})
              .limit(10).map(f -> f[0])
              .forEach(n -> System.out.print(n + " ")); // 0 1 1 2 3 5 8 13 21 34
        System.out.println();

        // 6. Files.lines()（读取文件为 Stream）
        Path tempFile = Files.createTempFile("test", ".txt");
        Files.writeString(tempFile, "line1\nline2\nline3\n");
        try (Stream<String> lines = Files.lines(tempFile)) {
            lines.filter(l -> !l.isEmpty()).map(String::trim).forEach(System.out::println);
        }
        Files.delete(tempFile);

        // 7. 数值流
        IntStream.range(1, 6).forEach(n -> System.out.print(n + " "));      // 1 2 3 4 5
        System.out.println();
        IntStream.rangeClosed(1, 5).forEach(n -> System.out.print(n + " ")); // 1 2 3 4 5
        System.out.println();

        // 8. Pattern 分割
        java.util.regex.Pattern.compile(",").splitAsStream("a,b,c,d")
            .forEach(System.out::println);

        // 9. Random 生成随机数流
        new Random().ints(5, 1, 100).forEach(n -> System.out.print(n + " "));
        System.out.println();
    }
}
```

#### 1.2.2 中间操作（惰性求值）

```java
import java.util.*;
import java.util.stream.*;

public class StreamIntermediateOpsDemo {

    record Person(String name, int age, String city, double salary) {}

    public static void main(String[] args) {
        List<Person> people = List.of(
            new Person("Alice", 30, "Beijing", 15000.0),
            new Person("Bob", 25, "Shanghai", 12000.0),
            new Person("Charlie", 35, "Beijing", 20000.0),
            new Person("Diana", 28, "Guangzhou", 13500.0),
            new Person("Eve", 30, "Shanghai", 16000.0),
            new Person("Frank", 22, "Beijing", 9000.0)
        );

        // filter：过滤
        System.out.println("=== filter ===");
        people.stream()
            .filter(p -> p.age() >= 28)
            .filter(p -> p.salary() > 12000)
            .map(Person::name)
            .forEach(System.out::println);

        // map：转换
        System.out.println("=== map ===");
        List<String> names = people.stream()
            .map(Person::name)
            .map(String::toUpperCase)
            .collect(Collectors.toList());
        System.out.println(names);

        // flatMap：扁平化
        System.out.println("=== flatMap ===");
        List<List<Integer>> nested = List.of(
            List.of(1, 2, 3), List.of(4, 5), List.of(6, 7, 8, 9)
        );
        List<Integer> flat = nested.stream()
            .flatMap(Collection::stream)
            .collect(Collectors.toList());
        System.out.println(flat); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

        // flatMap 应用：分词
        List<String> sentences = List.of("Hello World", "Java Stream API", "Lambda Expression");
        List<String> words = sentences.stream()
            .flatMap(s -> Arrays.stream(s.split(" ")))
            .distinct().sorted()
            .collect(Collectors.toList());
        System.out.println(words);

        // distinct：去重
        System.out.println("=== distinct ===");
        List<Integer> nums = List.of(1, 2, 2, 3, 3, 3, 4);
        nums.stream().distinct().forEach(n -> System.out.print(n + " ")); // 1 2 3 4
        System.out.println();

        // sorted：排序（自然/自定义）
        System.out.println("=== sorted ===");
        Stream.of(3, 1, 4, 1, 5, 9, 2, 6).sorted().forEach(n -> System.out.print(n + " "));
        System.out.println();

        // 多级排序
        people.stream()
            .sorted(Comparator.comparing(Person::city)
                .thenComparingDouble(Person::salary).reversed())
            .map(p -> p.city() + " - " + p.name())
            .forEach(System.out::println);

        // peek：调试用中间操作（不改变元素）
        System.out.println("=== peek ===");
        long count = people.stream()
            .filter(p -> p.age() >= 28)
            .peek(p -> System.out.println("After filter: " + p.name()))
            .map(Person::salary)
            .peek(s -> System.out.println("After map: " + s))
            .filter(s -> s > 13000)
            .count();
        System.out.println("Count: " + count);

        // limit 和 skip（分页）
        System.out.println("=== limit & skip ===");
        // 取薪资前3名
        people.stream()
            .sorted(Comparator.comparingDouble(Person::salary).reversed())
            .limit(3).map(Person::name)
            .forEach(System.out::println);

        // 分页：第1页，每页2条
        int pageSize = 2, pageNum = 1;
        people.stream().skip((long) pageNum * pageSize).limit(pageSize)
            .map(Person::name).forEach(System.out::println);

        // mapToInt/mapToLong/mapToDouble：转换为数值流
        System.out.println("=== mapToXxx ===");
        IntSummaryStatistics ageStat = people.stream().mapToInt(Person::age).summaryStatistics();
        System.out.printf("年龄统计 - 总数:%d, 最小:%d, 最大:%d, 总和:%d, 平均:%.2f%n",
            ageStat.getCount(), ageStat.getMin(), ageStat.getMax(),
            ageStat.getSum(), ageStat.getAverage());

        DoubleSummaryStatistics salaryStat = people.stream()
            .mapToDouble(Person::salary).summaryStatistics();
        System.out.printf("薪资统计 - 总和:%.0f, 平均:%.2f%n",
            salaryStat.getSum(), salaryStat.getAverage());
    }
}
```

#### 1.2.3 终端操作（完整 Collectors 解析）

```java
import java.util.*;
import java.util.stream.*;
import java.util.function.*;

public class StreamTerminalOpsDemo {

    record Order(String id, String customer, String status, double amount, List<String> items) {}

    public static void main(String[] args) {

        List<Order> orders = List.of(
            new Order("O001", "Alice", "PAID", 199.0, List.of("Java Book", "Pen")),
            new Order("O002", "Bob", "PENDING", 299.0, List.of("Laptop Stand")),
            new Order("O003", "Alice", "PAID", 99.0, List.of("Notebook")),
            new Order("O004", "Charlie", "CANCELLED", 149.0, List.of("Keyboard")),
            new Order("O005", "Bob", "PAID", 499.0, List.of("Monitor", "Cable")),
            new Order("O006", "Diana", "PENDING", 79.0, List.of("Mouse")),
            new Order("O007", "Alice", "PAID", 349.0, List.of("Course", "Book"))
        );

        // toList / toSet
        List<String> paidIds = orders.stream().filter(o -> "PAID".equals(o.status()))
            .map(Order::id).collect(Collectors.toList());
        System.out.println("PAID订单ID: " + paidIds);

        Set<String> customers = orders.stream().map(Order::customer).collect(Collectors.toSet());
        System.out.println("客户集合: " + customers);

        // toMap（处理重复key）
        Map<String, Double> orderAmounts = orders.stream().collect(Collectors.toMap(
            Order::id, Order::amount, (a, b) -> a));
        System.out.println("订单金额: " + orderAmounts);

        // groupingBy：按状态分组
        Map<String, List<Order>> ordersByStatus = orders.stream()
            .collect(Collectors.groupingBy(Order::status));
        ordersByStatus.forEach((status, list) ->
            System.out.printf("状态[%s]: %d 个订单%n", status, list.size()));

        // groupingBy + downstream：按状态统计金额
        Map<String, Double> totalByStatus = orders.stream().collect(
            Collectors.groupingBy(Order::status, Collectors.summingDouble(Order::amount)));
        System.out.println("各状态金额: " + totalByStatus);

        // 多级 groupingBy
        Map<String, Map<String, Long>> grouped = orders.stream().collect(
            Collectors.groupingBy(Order::customer,
                Collectors.groupingBy(Order::status, Collectors.counting())));
        System.out.println("客户-状态统计: " + grouped);

        // partitioningBy：按布尔条件分组
        Map<Boolean, List<Order>> partitioned = orders.stream()
            .collect(Collectors.partitioningBy(o -> o.amount() > 200));
        System.out.println("高额(>200): " + partitioned.get(true).stream()
            .map(Order::id).collect(Collectors.toList()));

        // joining：字符串拼接（分隔符, 前缀, 后缀）
        String orderList = orders.stream().filter(o -> "PAID".equals(o.status()))
            .map(Order::id).collect(Collectors.joining(", ", "[", "]"));
        System.out.println("已支付: " + orderList);

        // counting
        Map<String, Long> countByStatus = orders.stream()
            .collect(Collectors.groupingBy(Order::status, Collectors.counting()));
        System.out.println("各状态数量: " + countByStatus);

        // summingDouble / averagingDouble
        double avgAmount = orders.stream().filter(o -> "PAID".equals(o.status()))
            .collect(Collectors.averagingDouble(Order::amount));
        System.out.printf("已支付平均金额: 元%.2f%n", avgAmount);

        // summarizingDouble
        DoubleSummaryStatistics stats = orders.stream()
            .collect(Collectors.summarizingDouble(Order::amount));
        System.out.printf("金额统计 - 数量:%d 总额:%.0f 最小:%.0f 最大:%.0f 平均:%.2f%n",
            stats.getCount(), stats.getSum(), stats.getMin(), stats.getMax(), stats.getAverage());

        // mapping：先 map 再 collect
        Map<String, List<String>> idsByCustomer = orders.stream().collect(
            Collectors.groupingBy(Order::customer,
                Collectors.mapping(Order::id, Collectors.toList())));
        System.out.println("客户订单ID: " + idsByCustomer);

        // collectingAndThen：对 collect 结果再处理
        List<String> top3Customers = orders.stream().collect(
            Collectors.collectingAndThen(
                Collectors.groupingBy(Order::customer, Collectors.summingDouble(Order::amount)),
                map -> map.entrySet().stream()
                    .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
                    .limit(3).map(Map.Entry::getKey).collect(Collectors.toList())
            ));
        System.out.println("消费前3客户: " + top3Customers);

        // toCollection：指定集合类型
        TreeSet<String> sortedIds = orders.stream().map(Order::id)
            .collect(Collectors.toCollection(TreeSet::new));
        System.out.println("排序ID: " + sortedIds);

        // reduce（三种形式）
        System.out.println("\n=== reduce ===");
        // 1. 有初始值
        double totalAmount = orders.stream().mapToDouble(Order::amount).reduce(0.0, Double::sum);
        System.out.printf("总金额: 元%.2f%n", totalAmount);

        // 2. 无初始值（返回 Optional）
        Optional<Double> maxAmount = orders.stream().map(Order::amount).reduce(Double::max);
        maxAmount.ifPresent(a -> System.out.printf("最大金额: 元%.2f%n", a));

        // 3. 三参数（并行化用）
        List<String> allItems = orders.stream().reduce(new ArrayList<>(),
            (list, order) -> { List<String> n = new ArrayList<>(list); n.addAll(order.items()); return n; },
            (l1, l2) -> { List<String> c = new ArrayList<>(l1); c.addAll(l2); return c; });
        System.out.println("所有商品: " + allItems);

        // findFirst / findAny
        System.out.println("\n=== findFirst & findAny ===");
        Optional<Order> firstPaid = orders.stream()
            .filter(o -> "PAID".equals(o.status())).findFirst();
        firstPaid.ifPresent(o -> System.out.println("第一个已支付: " + o.id()));

        // anyMatch / allMatch / noneMatch（短路求值）
        System.out.println("\n=== match operations ===");
        boolean hasCancelled = orders.stream().anyMatch(o -> "CANCELLED".equals(o.status()));
        boolean allHaveItems = orders.stream().allMatch(o -> !o.items().isEmpty());
        boolean noneNegative = orders.stream().noneMatch(o -> o.amount() < 0);
        System.out.println("有取消订单: " + hasCancelled);    // true
        System.out.println("所有订单有商品: " + allHaveItems); // true
        System.out.println("没有负金额: " + noneNegative);     // true

        // count / min / max
        long total = orders.stream().count();
        Optional<Order> minOrder = orders.stream().min(Comparator.comparingDouble(Order::amount));
        Optional<Order> maxOrder = orders.stream().max(Comparator.comparingDouble(Order::amount));
        System.out.println("总订单: " + total);
        minOrder.ifPresent(o -> System.out.printf("最小: %s 元%.2f%n", o.id(), o.amount()));
        maxOrder.ifPresent(o -> System.out.printf("最大: %s 元%.2f%n", o.id(), o.amount()));

        // toArray
        String[] arr = orders.stream().map(Order::id).toArray(String[]::new);
        System.out.println("arr: " + Arrays.toString(arr));
    }
}
```

## 1.2.4 数值流与并行流

### 数值流（IntStream / LongStream / DoubleStream）

```java
import java.util.stream.*;

public class NumericStreamDemo {
    public static void main(String[] args) {
        // IntStream 创建
        IntStream.range(1, 6).forEach(i -> System.out.print(i + " ")); // 1 2 3 4 5
        System.out.println();
        IntStream.rangeClosed(1, 5).forEach(i -> System.out.print(i + " ")); // 1 2 3 4 5
        System.out.println();

        // 统计方法
        IntSummaryStatistics stats = IntStream.rangeClosed(1, 100).summaryStatistics();
        System.out.println("sum=" + stats.getSum() + " avg=" + stats.getAverage()
            + " min=" + stats.getMin() + " max=" + stats.getMax());

        // 对象流 -> 数值流
        int totalLen = Stream.of("hello", "world", "java")
            .mapToInt(String::length).sum();
        System.out.println("总字符数: " + totalLen);

        // 数值流 -> 对象流
        Stream<String> strStream = IntStream.rangeClosed(1, 3)
            .mapToObj(i -> "item-" + i);
        strStream.forEach(System.out::println);

        // 装箱
        List<Integer> list = IntStream.rangeClosed(1, 5).boxed()
            .collect(Collectors.toList());
        System.out.println(list);
    }
}
```

### 并行流（Parallel Stream）

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.stream.*;

public class ParallelStreamDemo {
    public static void main(String[] args) throws Exception {
        List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
            .boxed().collect(Collectors.toList());

        // 串行 vs 并行 性能对比
        long start = System.currentTimeMillis();
        long sum1 = numbers.stream().mapToLong(Integer::longValue).sum();
        System.out.println("串行耗时: " + (System.currentTimeMillis() - start) + "ms, sum=" + sum1);

        start = System.currentTimeMillis();
        long sum2 = numbers.parallelStream().mapToLong(Integer::longValue).sum();
        System.out.println("并行耗时: " + (System.currentTimeMillis() - start) + "ms, sum=" + sum2);

        // 注意：并行流共享变量会有线程安全问题
        List<Integer> unsafe = new ArrayList<>();
        // 错误示例（不要这样做）:
        // numbers.parallelStream().forEach(unsafe::add); // ConcurrentModificationException 风险

        // 正确做法：使用 collect
        List<Integer> safe = numbers.parallelStream()
            .filter(n -> n % 2 == 0)
            .collect(Collectors.toList());
        System.out.println("偶数个数: " + safe.size());

        // 控制并行度（使用自定义 ForkJoinPool）
        ForkJoinPool pool = new ForkJoinPool(4);
        long result = pool.submit(() ->
            numbers.parallelStream().mapToLong(Integer::longValue).sum()
        ).get();
        pool.shutdown();
        System.out.println("自定义线程池结果: " + result);
    }
}
```

---

## 1.3 Optional

`Optional<T>` 是一个容器，用来表示值可能存在也可能不存在，彻底消灭 NPE。

```java
import java.util.*;

public class OptionalDemo {
    record User(String name, Address address) {}
    record Address(String city, String street) {}

    public static void main(String[] args) {
        // 创建
        Optional<String> empty   = Optional.empty();
        Optional<String> present = Optional.of("Java");
        Optional<String> nullable = Optional.ofNullable(null); // 等同 empty

        // 判断 / 获取
        System.out.println(present.isPresent());    // true
        System.out.println(present.isEmpty());      // false (Java 11+)
        System.out.println(present.get());          // Java

        // orElse / orElseGet / orElseThrow
        String v1 = empty.orElse("default");                            // default
        String v2 = empty.orElseGet(() -> "computed-default");          // computed-default
        // empty.orElseThrow(() -> new RuntimeException("no value"));   // 抛异常

        // map / flatMap / filter
        Optional<Integer> len = present.map(String::length);   // Optional[4]
        System.out.println(len.orElse(0));

        Optional<String> upper = present
            .filter(s -> s.length() > 2)
            .map(String::toUpperCase);
        System.out.println(upper.orElse(""));  // JAVA

        // 链式避免 NPE（替代多层 null 判断）
        User user = new User("Alice", new Address("Beijing", "Chaoyang"));
        String city = Optional.ofNullable(user)
            .map(User::address)
            .map(Address::city)
            .orElse("Unknown");
        System.out.println(city);  // Beijing

        // ifPresent / ifPresentOrElse (Java 9+)
        present.ifPresent(s -> System.out.println("Value: " + s));
        empty.ifPresentOrElse(
            s -> System.out.println("present: " + s),
            () -> System.out.println("empty!")
        );

        // or (Java 9+) - 返回另一个 Optional
        Optional<String> result = empty.or(() -> Optional.of("fallback"));
        System.out.println(result.get());  // fallback

        // stream (Java 9+) - 转为 Stream
        long count = present.stream().count();  // 1
        System.out.println(count);
    }
}
```

---

## 1.4 接口增强：default 与 static 方法

Java 8 允许在接口中定义 `default` 和 `static` 方法，解决了接口演化问题。

```java
public class InterfaceEnhancementDemo {

    interface Vehicle {
        String getName();

        // default 方法：有默认实现，子类可选择覆盖
        default String describe() {
            return "Vehicle: " + getName();
        }

        default void service() {
            System.out.println(getName() + " is being serviced.");
        }

        // static 方法：只能通过接口名调用，不能被继承
        static Vehicle create(String name) {
            return () -> name;
        }

        // Java 9+: private 方法（供 default/static 方法复用）
        // private void log(String msg) { System.out.println("[LOG] " + msg); }
    }

    interface Electric {
        default String describe() {
            return "Electric vehicle";
        }
    }

    // 多继承冲突：必须显式覆盖
    static class Tesla implements Vehicle, Electric {
        @Override
        public String getName() { return "Tesla"; }

        @Override
        public String describe() {
            // 显式选择其中一个
            return Vehicle.super.describe() + " (Electric)";
        }
    }

    public static void main(String[] args) {
        Tesla tesla = new Tesla();
        System.out.println(tesla.describe());   // Vehicle: Tesla (Electric)
        tesla.service();

        // 通过接口 static 方法创建实例
        Vehicle bike = Vehicle.create("Bike");
        System.out.println(bike.describe());    // Vehicle: Bike
    }
}
```

---

## 1.5 新日期时间 API（java.time）

Java 8 引入了全新的 `java.time` 包，彻底替代旧的 `Date`/`Calendar`。

```java
import java.time.*;
import java.time.format.*;
import java.time.temporal.*;
import java.util.*;

public class DateTimeDemo {
    public static void main(String[] args) {
        // ===== 基本类型 =====
        LocalDate date = LocalDate.now();
        LocalTime time = LocalTime.now();
        LocalDateTime dateTime = LocalDateTime.now();
        ZonedDateTime zonedDateTime = ZonedDateTime.now(ZoneId.of("Asia/Shanghai"));

        System.out.println("date: " + date);
        System.out.println("time: " + time);
        System.out.println("dateTime: " + dateTime);
        System.out.println("zonedDateTime: " + zonedDateTime);

        // ===== 创建指定日期时间 =====
        LocalDate birthday = LocalDate.of(1990, Month.JUNE, 15);
        LocalTime meetingTime = LocalTime.of(14, 30, 0);
        LocalDateTime meeting = LocalDateTime.of(birthday, meetingTime);
        System.out.println("生日: " + birthday);
        System.out.println("会议时间: " + meeting);

        // ===== 日期计算 =====
        LocalDate nextWeek = date.plusWeeks(1);
        LocalDate lastMonth = date.minusMonths(1);
        LocalDate nextYear = date.plus(1, ChronoUnit.YEARS);
        System.out.println("下周: " + nextWeek);
        System.out.println("上月: " + lastMonth);

        // ===== 比较 =====
        LocalDate d1 = LocalDate.of(2024, 1, 1);
        LocalDate d2 = LocalDate.of(2024, 12, 31);
        System.out.println("d1 在 d2 之前: " + d1.isBefore(d2));     // true
        System.out.println("相差天数: " + ChronoUnit.DAYS.between(d1, d2));  // 365

        // ===== Period 和 Duration =====
        Period period = Period.between(birthday, date);
        System.out.printf("年龄: %d年%d月%d天%n", period.getYears(), period.getMonths(), period.getDays());

        Duration duration = Duration.between(LocalTime.of(9, 0), LocalTime.of(17, 30));
        System.out.println("工作时长: " + duration.toHours() + "小时" + (duration.toMinutesPart()) + "分");

        // ===== 格式化与解析 =====
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy年MM月dd日 HH:mm:ss");
        String formatted = dateTime.format(formatter);
        System.out.println("格式化: " + formatted);

        LocalDateTime parsed = LocalDateTime.parse("2024年06月15日 14:30:00", formatter);
        System.out.println("解析: " + parsed);

        // 预定义格式
        String iso = dateTime.format(DateTimeFormatter.ISO_LOCAL_DATE_TIME);
        System.out.println("ISO格式: " + iso);

        // ===== 时区转换 =====
        ZoneId shanghai = ZoneId.of("Asia/Shanghai");
        ZoneId newYork  = ZoneId.of("America/New_York");
        ZonedDateTime shanghaiTime = ZonedDateTime.now(shanghai);
        ZonedDateTime newYorkTime  = shanghaiTime.withZoneSameInstant(newYork);
        System.out.println("上海: " + shanghaiTime.format(DateTimeFormatter.ofPattern("HH:mm z")));
        System.out.println("纽约: " + newYorkTime.format(DateTimeFormatter.ofPattern("HH:mm z")));

        // ===== Instant（时间戳） =====
        Instant now = Instant.now();
        System.out.println("时间戳(ms): " + now.toEpochMilli());

        // 与 Date 互转（兼容旧代码）
        Date legacyDate = Date.from(now);
        Instant fromLegacy = legacyDate.toInstant();
        System.out.println("Date->Instant: " + fromLegacy);

        // ===== TemporalAdjuster（日期调整器） =====
        LocalDate firstDayOfMonth = date.with(TemporalAdjusters.firstDayOfMonth());
        LocalDate lastDayOfMonth  = date.with(TemporalAdjusters.lastDayOfMonth());
        LocalDate nextMonday = date.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
        System.out.println("本月第一天: " + firstDayOfMonth);
        System.out.println("本月最后一天: " + lastDayOfMonth);
        System.out.println("下一个周一: " + nextMonday);
    }
}
```

---

## 1.6 其他 Java 8 特性

### CompletableFuture（异步编程）

```java
import java.util.concurrent.*;

public class CompletableFutureDemo {
    static String fetchUser(int id) {
        try { Thread.sleep(100); } catch (Exception e) {}
        return "User-" + id;
    }
    static String fetchOrder(String user) {
        try { Thread.sleep(100); } catch (Exception e) {}
        return "Order of " + user;
    }

    public static void main(String[] args) throws Exception {
        // 基本用法
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> fetchUser(1));
        System.out.println(future.get());  // User-1

        // 链式处理
        CompletableFuture<String> chain = CompletableFuture
            .supplyAsync(() -> fetchUser(2))
            .thenApply(user -> user.toUpperCase())
            .thenApply(user -> "Hello, " + user);
        System.out.println(chain.get());   // Hello, USER-2

        // thenCompose（扁平化）
        CompletableFuture<String> composed = CompletableFuture
            .supplyAsync(() -> fetchUser(3))
            .thenCompose(user -> CompletableFuture.supplyAsync(() -> fetchOrder(user)));
        System.out.println(composed.get()); // Order of User-3

        // thenCombine（合并两个独立 Future）
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");
        CompletableFuture<String> combined = f1.thenCombine(f2, (a, b) -> a + " " + b);
        System.out.println(combined.get());  // Hello World

        // allOf（等待全部完成）
        CompletableFuture<Void> all = CompletableFuture.allOf(
            CompletableFuture.runAsync(() -> System.out.println("Task1")),
            CompletableFuture.runAsync(() -> System.out.println("Task2")),
            CompletableFuture.runAsync(() -> System.out.println("Task3"))
        );
        all.join();

        // anyOf（任一完成即返回）
        CompletableFuture<Object> any = CompletableFuture.anyOf(
            CompletableFuture.supplyAsync(() -> { try{Thread.sleep(200);}catch(Exception e){} return "slow"; }),
            CompletableFuture.supplyAsync(() -> "fast")
        );
        System.out.println("最快的: " + any.get());  // fast

        // 异常处理
        CompletableFuture<String> withError = CompletableFuture
            .supplyAsync(() -> { throw new RuntimeException("oops"); })
            .exceptionally(ex -> "Error handled: " + ex.getMessage())
            .thenApply(r -> "Result: " + r);
        System.out.println(withError.get());
    }
}
```

---
# Part 2: Java 9 新特性

## 2.1 模块系统（JPMS - Java Platform Module System）

模块系统是 Java 9 最重大的变革，将 JDK 本身和应用程序拆分为模块。

```
// 项目结构示例
my-app/
├── src/
│   ├── com.myapp.api/
│   │   ├── module-info.java
│   │   └── com/myapp/api/UserService.java
│   └── com.myapp.impl/
│       ├── module-info.java
│       └── com/myapp/impl/UserServiceImpl.java
```

```java
// com.myapp.api/module-info.java
module com.myapp.api {
    exports com.myapp.api;           // 导出给所有模块
    exports com.myapp.api.internal to com.myapp.impl;  // 只导出给特定模块
}

// com.myapp.impl/module-info.java
module com.myapp.impl {
    requires com.myapp.api;          // 声明依赖
    requires java.logging;           // 依赖 JDK 模块
    provides com.myapp.api.UserService with com.myapp.impl.UserServiceImpl;
}
```

```java
// UserService.java
package com.myapp.api;
public interface UserService {
    String getUser(int id);
}

// UserServiceImpl.java
package com.myapp.impl;
import com.myapp.api.UserService;
public class UserServiceImpl implements UserService {
    @Override
    public String getUser(int id) { return "User-" + id; }
}
```

## 2.2 集合工厂方法（List.of / Set.of / Map.of）

```java
import java.util.*;

public class CollectionFactoryDemo {
    public static void main(String[] args) {
        // 不可变 List
        List<String> list = List.of("a", "b", "c");
        System.out.println(list);
        // list.add("d");  // UnsupportedOperationException

        // 不可变 Set（不允许重复元素）
        Set<Integer> set = Set.of(1, 2, 3, 4, 5);
        System.out.println(set.contains(3));  // true
        // Set.of(1, 1); // IllegalArgumentException 重复元素

        // 不可变 Map（最多 10 对用 Map.of，更多用 Map.ofEntries）
        Map<String, Integer> map = Map.of(
            "one", 1, "two", 2, "three", 3
        );
        System.out.println(map.get("two"));  // 2

        // 超过 10 对键值使用 Map.ofEntries
        Map<String, Integer> bigMap = Map.ofEntries(
            Map.entry("a", 1),
            Map.entry("b", 2),
            Map.entry("c", 3),
            Map.entry("d", 4),
            Map.entry("e", 5),
            Map.entry("f", 6),
            Map.entry("g", 7),
            Map.entry("h", 8),
            Map.entry("i", 9),
            Map.entry("j", 10),
            Map.entry("k", 11)
        );
        System.out.println("size: " + bigMap.size());

        // 注意：不允许 null 值
        // List.of(null);  // NullPointerException

        // 与 Collections.unmodifiableXxx 的区别：
        // - List.of 不允许 null，性能更好，内存占用更少
        // - unmodifiableList 只是包装，底层集合修改会反映出来
        List<String> mutable = new ArrayList<>(Arrays.asList("x", "y"));
        List<String> view = Collections.unmodifiableList(mutable);
        mutable.add("z");
        System.out.println("view has z: " + view.contains("z")); // true（坑！）

        List<String> immutable = List.of("x", "y");
        System.out.println("immutable size: " + immutable.size()); // 2（稳定）
    }
}
```

## 2.3 JShell（交互式 REPL）

JShell 是 Java 9 引入的交互式编程工具，可以即时执行 Java 代码片段。

```
# 启动 JShell
$ jshell

# 基本运算
jshell> 1 + 2
$1 ==> 3

# 定义变量
jshell> String name = "Java"
name ==> "Java"

jshell> int version = 21
version ==> 21

jshell> name + " " + version
$4 ==> "Java 21"

# 定义方法
jshell> String greet(String s) { return "Hello, " + s + "!"; }
|  已创建 方法 greet(String)

jshell> greet("World")
$6 ==> "Hello, World!"

# 使用 Tab 补全
jshell> "hello".toUp<TAB>  -> 补全为 toUpperCase

# /list 查看历史
jshell> /list

# /exit 退出
jshell> /exit
```

## 2.4 Stream API 增强（Java 9）

```java
import java.util.*;
import java.util.stream.*;

public class StreamEnhancementJava9 {
    public static void main(String[] args) {
        // takeWhile：取元素直到条件为 false（有序流）
        List<Integer> result1 = Stream.of(1, 2, 3, 4, 5, 1, 2)
            .takeWhile(n -> n < 4)
            .collect(Collectors.toList());
        System.out.println("takeWhile < 4: " + result1);  // [1, 2, 3]

        // dropWhile：丢弃元素直到条件为 false
        List<Integer> result2 = Stream.of(1, 2, 3, 4, 5, 1, 2)
            .dropWhile(n -> n < 4)
            .collect(Collectors.toList());
        System.out.println("dropWhile < 4: " + result2);  // [4, 5, 1, 2]

        // Stream.ofNullable：避免 NPE
        Stream<String> s1 = Stream.ofNullable("hello");   // 1个元素
        Stream<String> s2 = Stream.ofNullable(null);      // 空流
        System.out.println("ofNullable hello: " + s1.count()); // 1
        System.out.println("ofNullable null: " + s2.count());  // 0

        // Stream.iterate 增强版（带终止条件）
        // Java 8: Stream.iterate(0, n -> n + 2) 无限流，需 limit
        // Java 9: 新增带 Predicate 参数的重载
        List<Integer> evens = Stream.iterate(0, n -> n < 20, n -> n + 2)
            .collect(Collectors.toList());
        System.out.println("偶数: " + evens);  // [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]

        // Optional.stream()（前面已介绍）
        List<String> names = List.of(
            Optional.of("Alice"),
            Optional.empty(),
            Optional.of("Bob")
        ).stream()
         .flatMap(Optional::stream)
         .collect(Collectors.toList());
        System.out.println("非空名字: " + names);  // [Alice, Bob]
    }
}
```

## 2.5 Flow API（响应式编程基础）

```java
import java.util.concurrent.*;

public class FlowDemo {
    // 自定义 Publisher
    static class StringPublisher implements Flow.Publisher<String> {
        private final List<String> items;
        StringPublisher(List<String> items) { this.items = items; }

        @Override
        public void subscribe(Flow.Subscriber<? super String> subscriber) {
            subscriber.onSubscribe(new Flow.Subscription() {
                private int index = 0;
                private boolean cancelled = false;

                @Override
                public void request(long n) {
                    for (long i = 0; i < n && index < items.size() && !cancelled; i++) {
                        subscriber.onNext(items.get(index++));
                    }
                    if (index >= items.size()) subscriber.onComplete();
                }

                @Override
                public void cancel() { cancelled = true; }
            });
        }
    }

    // 自定义 Subscriber
    static class PrintSubscriber implements Flow.Subscriber<String> {
        private Flow.Subscription subscription;

        @Override public void onSubscribe(Flow.Subscription s) {
            this.subscription = s;
            s.request(Long.MAX_VALUE);  // 请求所有数据
        }
        @Override public void onNext(String item) {
            System.out.println("收到: " + item);
        }
        @Override public void onError(Throwable t) {
            System.err.println("错误: " + t.getMessage());
        }
        @Override public void onComplete() {
            System.out.println("完成！");
        }
    }

    public static void main(String[] args) throws Exception {
        StringPublisher publisher = new StringPublisher(
            java.util.List.of("Hello", "World", "Java", "9")
        );
        PrintSubscriber subscriber = new PrintSubscriber();
        publisher.subscribe(subscriber);

        // SubmissionPublisher（JDK 内置实现）
        try (SubmissionPublisher<String> sp = new SubmissionPublisher<>()) {
            sp.subscribe(new PrintSubscriber());
            sp.submit("async-1");
            sp.submit("async-2");
            Thread.sleep(100);  // 等待异步处理
        }
    }
}
```

---

# Part 3: Java 10 新特性

## 3.1 局部变量类型推断（var）

```java
import java.util.*;
import java.util.stream.*;

public class VarDemo {
    public static void main(String[] args) {
        // 基本用法 - 编译器自动推断类型
        var number = 42;                    // int
        var pi = 3.14;                      // double
        var name = "Java 10";               // String
        var list = new ArrayList<String>(); // ArrayList<String>
        var map = new HashMap<String, Integer>(); // HashMap<String, Integer>

        list.add("hello");
        map.put("a", 1);

        // 在 for 循环中使用
        var items = List.of("apple", "banana", "cherry");
        for (var item : items) {
            System.out.println(item.toUpperCase());  // 仍有类型提示
        }

        // 在 try-with-resources 中
        // try (var reader = new java.io.BufferedReader(...)) { ... }

        // 复杂泛型场景下减少冗余
        var result = items.stream()
            .filter(s -> s.length() > 5)
            .collect(Collectors.toList());
        System.out.println(result);

        // var 在 lambda 参数中（Java 11+）
        // var 用于 lambda 参数以便添加注解
        // (@NotNull var s) -> s.toUpperCase()

        // === var 的限制 ===
        // var x;              // 错误：必须初始化
        // var y = null;       // 错误：无法推断 null 类型
        // var z = {1, 2, 3};  // 错误：数组初始化器不支持
        // 不能用于字段、方法参数、返回类型
    }

    // var 不能用于方法签名（参数和返回类型）
    // public var compute() { ... }  // 错误
    // public void process(var x) {} // 错误
}
```

## 3.2 集合 copyOf 方法

```java
import java.util.*;

public class CopyOfDemo {
    public static void main(String[] args) {
        // List.copyOf / Set.copyOf / Map.copyOf 创建不可变副本
        List<String> original = new ArrayList<>(Arrays.asList("a", "b", "c"));
        List<String> copy = List.copyOf(original);

        original.add("d");
        System.out.println("original: " + original);  // [a, b, c, d]
        System.out.println("copy:     " + copy);      // [a, b, c] 不受影响

        // copy.add("e");  // UnsupportedOperationException

        // Set.copyOf
        Set<Integer> nums = Set.copyOf(Arrays.asList(1, 2, 3, 2, 1));
        System.out.println("set copy: " + nums);  // [1, 2, 3]（去重）

        // Map.copyOf
        Map<String, Integer> mutableMap = new HashMap<>();
        mutableMap.put("x", 10);
        mutableMap.put("y", 20);
        Map<String, Integer> immutableMap = Map.copyOf(mutableMap);
        System.out.println("map copy: " + immutableMap);
    }
}
```

---

# Part 4: Java 11 新特性

## 4.1 String 新方法

```java
public class StringNewMethodsDemo {
    public static void main(String[] args) {
        // isBlank()：判断字符串是否为空或只含空白字符（比 isEmpty() 更强）
        System.out.println("".isBlank());          // true
        System.out.println("  \t\n".isBlank());    // true
        System.out.println(" hello ".isBlank());   // false

        // strip() / stripLeading() / stripTrailing()
        // 与 trim() 的区别：strip() 支持 Unicode 空白字符
        String padded = "  Hello World  ";
        System.out.println("|" + padded.strip()          + "|");  // |Hello World|
        System.out.println("|" + padded.stripLeading()   + "|");  // |Hello World  |
        System.out.println("|" + padded.stripTrailing()  + "|");  // |  Hello World|

        // lines()：按行分割，返回 Stream<String>
        String multiline = "line1\nline2\nline3\nline4";
        multiline.lines().forEach(System.out::println);
        long lineCount = multiline.lines().count();
        System.out.println("行数: " + lineCount);  // 4

        // repeat(n)：重复字符串 n 次
        String sep = "-".repeat(20);
        System.out.println(sep);  // --------------------
        String stars = "* ".repeat(5);
        System.out.println(stars);  // * * * * *

        // 综合示例：解析简单 CSV
        String csv = "  Alice , 30 , Beijing  \n  Bob , 25 , Shanghai  ";
        csv.lines()
           .map(line -> line.strip())
           .filter(line -> !line.isBlank())
           .map(line -> {
               String[] parts = line.split(",");
               return String.format("名字:%s 年龄:%s 城市:%s",
                   parts[0].strip(), parts[1].strip(), parts[2].strip());
           })
           .forEach(System.out::println);
    }
}
```

## 4.2 Files 新方法

```java
import java.nio.file.*;

public class FilesNewMethodsDemo {
    public static void main(String[] args) throws Exception {
        // writeString / readString
        Path tempFile = Files.createTempFile("test", ".txt");

        Files.writeString(tempFile, "Hello Java 11!\n第二行内容");
        String content = Files.readString(tempFile);
        System.out.println(content);

        // 指定编码
        Files.writeString(tempFile, "中文内容", java.nio.charset.StandardCharsets.UTF_8);
        String utf8Content = Files.readString(tempFile, java.nio.charset.StandardCharsets.UTF_8);
        System.out.println(utf8Content);

        // isSameFile 检查两个 Path 是否指向同一文件
        Path p1 = Path.of("file.txt");
        // Path p2 = Path.of("./file.txt");
        // System.out.println(Files.isSameFile(p1, p2));  // true

        Files.deleteIfExists(tempFile);

        // Path.of 替代 Paths.get
        Path path = Path.of("/usr", "local", "lib");
        System.out.println(path);
    }
}
```

## 4.3 HTTP Client（正式版）

```java
import java.net.http.*;
import java.net.*;
import java.time.Duration;
import java.util.concurrent.*;

public class HttpClientDemo {
    public static void main(String[] args) throws Exception {
        // 创建 HttpClient
        HttpClient client = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_2)
            .connectTimeout(Duration.ofSeconds(10))
            .followRedirects(HttpClient.Redirect.NORMAL)
            .build();

        // 同步 GET 请求
        HttpRequest request = HttpRequest.newBuilder()
            .uri(URI.create("https://httpbin.org/get"))
            .timeout(Duration.ofSeconds(30))
            .header("Accept", "application/json")
            .GET()
            .build();

        HttpResponse<String> response = client.send(request,
            HttpResponse.BodyHandlers.ofString());

        System.out.println("状态码: " + response.statusCode());
        System.out.println("响应头: " + response.headers().map());
        System.out.println("响应体: " + response.body().substring(0, 100) + "...");

        // 异步 GET 请求
        CompletableFuture<HttpResponse<String>> asyncResponse =
            client.sendAsync(request, HttpResponse.BodyHandlers.ofString());

        asyncResponse
            .thenApply(HttpResponse::body)
            .thenAccept(body -> System.out.println("异步响应长度: " + body.length()))
            .join();

        // POST 请求
        HttpRequest postRequest = HttpRequest.newBuilder()
            .uri(URI.create("https://httpbin.org/post"))
            .header("Content-Type", "application/json")
            .POST(HttpRequest.BodyPublishers.ofString("{\"name\":\"Java\",\"version\":11}"))
            .build();

        HttpResponse<String> postResponse = client.send(postRequest,
            HttpResponse.BodyHandlers.ofString());
        System.out.println("POST 状态: " + postResponse.statusCode());
    }
}
```

## 4.4 其他 Java 11 特性

```java
// var 在 Lambda 参数中（Java 11）
import java.util.function.*;
import java.util.List;

public class Java11OtherFeatures {
    public static void main(String[] args) {
        // Lambda 参数使用 var（主要用于加注解）
        Function<String, String> f = (@SuppressWarnings("unused") var s) -> s.toUpperCase();
        System.out.println(f.apply("hello"));

        // 运行单文件 Java 程序（无需 javac）
        // java HelloWorld.java  直接运行

        // String 新增：
        // toArray() 已有，11 新增了通过 Collectors.joining() 等

        // 增强 NullPointerException 消息（Java 14 正式，11 可启用）
        // -XX:+ShowCodeDetailsInExceptionMessages

        // 嵌套访问控制
        class Outer {
            private int x = 10;
            class Inner {
                void access() {
                    // Java 11 之前：通过生成 access$xxx 方法
                    // Java 11+：直接访问，isNestmateOf()
                    System.out.println(x);
                }
            }
        }
    }
}
```

---
# Part 5: Java 14-16 语法糖

## 5.1 Switch 表达式（Java 14 正式）

```java
public class SwitchExpressionDemo {
    enum Day { MON, TUE, WED, THU, FRI, SAT, SUN }

    public static void main(String[] args) {
        Day day = Day.WED;

        // === 旧式 switch 语句（容易 fall-through 出错）===
        int oldResult;
        switch (day) {
            case MON: case TUE: case WED: case THU: case FRI:
                oldResult = 1; break;
            case SAT: case SUN:
                oldResult = 2; break;
            default:
                oldResult = -1;
        }

        // === 新式 switch 表达式（Java 14+）===
        // 箭头语法：不会 fall-through
        int workOrRest = switch (day) {
            case MON, TUE, WED, THU, FRI -> 1;
            case SAT, SUN -> 2;
        };
        System.out.println("工作日=1/休息日=2: " + workOrRest);  // 1

        // 多行 case + yield 返回值
        String description = switch (day) {
            case MON -> "周一，新的开始";
            case FRI -> "周五，快乐！";
            case SAT, SUN -> {
                String base = "周末";
                yield base + "，好好休息";  // yield 从块中返回值
            }
            default -> {
                yield "普通工作日: " + day.name();
            }
        };
        System.out.println(description);

        // switch 用于返回值
        String type = getType(42);
        System.out.println(type);
    }

    static String getType(Object obj) {
        return switch (obj) {
            case Integer i when i > 0 -> "正整数: " + i;  // Java 21 模式匹配
            case Integer i            -> "非正整数: " + i;
            case String s             -> "字符串: " + s;
            case null                 -> "null";
            default                   -> "其他: " + obj.getClass().getSimpleName();
        };
    }
}
```

## 5.2 文本块（Text Block，Java 15 正式）

```java
public class TextBlockDemo {
    public static void main(String[] args) {
        // 旧式多行字符串（丑陋）
        String oldJson = "{\n" +
            "  \"name\": \"Alice\",\n" +
            "  \"age\": 30,\n" +
            "  \"city\": \"Beijing\"\n" +
            "}";

        // 文本块（优雅）
        String json = """
                {
                  "name": "Alice",
                  "age": 30,
                  "city": "Beijing"
                }
                """;
        System.out.println(json);

        // HTML 文本块
        String html = """
                <html>
                    <body>
                        <h1>Hello, Java 15!</h1>
                        <p>Text Blocks are awesome.</p>
                    </body>
                </html>
                """;
        System.out.println(html);

        // SQL 文本块
        String sql = """
                SELECT u.id, u.name, o.amount
                FROM users u
                JOIN orders o ON u.id = o.user_id
                WHERE u.status = 'ACTIVE'
                  AND o.amount > 100
                ORDER BY o.amount DESC
                LIMIT 10
                """;
        System.out.println(sql);

        // 转义序列
        String noTrailingNewline = """
                no newline at end\
                """;  // \  续行符，末尾无换行
        System.out.println("|" + noTrailingNewline + "|");  // |no newline at end|

        // \s 强制保留尾部空格
        String withSpaces = """
                line1   \s
                line2   \s
                """;
        System.out.print(withSpaces);

        // 文本块与 formatted()
        String template = """
                Dear %s,
                Your order #%d has been shipped.
                Total: ¥%.2f
                """.formatted("Alice", 12345, 299.99);
        System.out.println(template);
    }
}
```

## 5.3 instanceof 模式匹配（Java 16 正式）

```java
import java.util.*;

public class InstanceofPatternDemo {
    sealed interface Shape permits Circle, Rectangle, Triangle {}
    record Circle(double radius) implements Shape {}
    record Rectangle(double w, double h) implements Shape {}
    record Triangle(double base, double height) implements Shape {}

    public static void main(String[] args) {
        Object obj = "Hello, Java 16!";

        // 旧式写法（繁琐）
        if (obj instanceof String) {
            String s = (String) obj;
            System.out.println("旧式: 长度=" + s.length());
        }

        // 新式模式匹配（简洁）
        if (obj instanceof String s) {
            System.out.println("新式: 长度=" + s.length());
        }

        // 带条件的模式匹配
        if (obj instanceof String s && s.length() > 5) {
            System.out.println("长字符串: " + s);
        }

        // 在复杂逻辑中使用
        List<Object> items = List.of(1, "hello", 3.14, true, List.of(1, 2), null);
        for (Object item : items) {
            String desc;
            if (item instanceof Integer i)         desc = "整数: " + i;
            else if (item instanceof String s)     desc = "字符串: " + s.toUpperCase();
            else if (item instanceof Double d)     desc = "小数: " + d;
            else if (item instanceof Boolean b)    desc = "布尔: " + b;
            else if (item instanceof List<?> l)    desc = "列表，大小: " + l.size();
            else                                   desc = "其他: " + item;
            System.out.println(desc);
        }

        // 计算形状面积
        List<Shape> shapes = List.of(
            new Circle(5), new Rectangle(4, 6), new Triangle(3, 8)
        );
        for (Shape shape : shapes) {
            double area;
            if (shape instanceof Circle c)         area = Math.PI * c.radius() * c.radius();
            else if (shape instanceof Rectangle r) area = r.w() * r.h();
            else if (shape instanceof Triangle t)  area = 0.5 * t.base() * t.height();
            else                                   area = 0;
            System.out.printf("%s 面积: %.2f%n", shape.getClass().getSimpleName(), area);
        }
    }
}
```

## 5.4 Records（Java 16 正式）

```java
import java.util.*;

public class RecordsDemo {
    // 基本 record：自动生成构造器、getter、equals、hashCode、toString
    record Point(double x, double y) {}

    // 带验证的 record（compact constructor）
    record Range(int min, int max) {
        Range {  // compact constructor
            if (min > max) throw new IllegalArgumentException("min > max");
        }
    }

    // 带自定义方法的 record
    record Money(double amount, String currency) {
        // 静态工厂方法
        static Money of(double amount) { return new Money(amount, "CNY"); }

        // 实例方法
        Money add(Money other) {
            if (!this.currency.equals(other.currency))
                throw new IllegalArgumentException("货币不匹配");
            return new Money(this.amount + other.amount, this.currency);
        }

        // 覆盖 toString
        @Override
        public String toString() {
            return String.format("%s %.2f", currency, amount);
        }
    }

    // 实现接口的 record
    interface Describable { String describe(); }
    record Person(String name, int age) implements Describable {
        @Override
        public String describe() {
            return String.format("%s(%d岁)", name, age);
        }
    }

    // 泛型 record
    record Pair<A, B>(A first, B second) {
        static <A, B> Pair<A, B> of(A a, B b) { return new Pair<>(a, b); }
    }

    public static void main(String[] args) {
        // Point
        Point p1 = new Point(3.0, 4.0);
        Point p2 = new Point(3.0, 4.0);
        System.out.println(p1);             // Point[x=3.0, y=4.0]
        System.out.println(p1.x());         // 3.0
        System.out.println(p1.equals(p2));  // true
        System.out.println(p1 == p2);       // false

        // Range 验证
        Range r = new Range(1, 10);
        System.out.println(r);
        try { new Range(10, 1); }
        catch (IllegalArgumentException e) { System.out.println("捕获: " + e.getMessage()); }

        // Money
        Money m1 = Money.of(100);
        Money m2 = new Money(50, "CNY");
        System.out.println(m1.add(m2));  // CNY 150.00

        // Person
        Person alice = new Person("Alice", 30);
        System.out.println(alice.describe());  // Alice(30岁)

        // Pair
        Pair<String, Integer> pair = Pair.of("age", 25);
        System.out.println(pair);  // Pair[first=age, second=25]

        // record 在 Stream 中的应用
        List<Person> people = List.of(
            new Person("Bob", 25),
            new Person("Carol", 35),
            new Person("Dave", 28)
        );
        people.stream()
            .filter(p -> p.age() > 26)
            .sorted(Comparator.comparing(Person::age))
            .forEach(p -> System.out.println(p.describe()));
    }
}
```

---

# Part 6: Java 17 新特性

## 6.1 密封类（Sealed Classes，Java 17 正式）

```java
public class SealedClassesDemo {

    // 密封接口：限制哪些类可以实现它
    sealed interface Expr
        permits Num, Add, Mul, Neg {}

    record Num(double value) implements Expr {}
    record Add(Expr left, Expr right) implements Expr {}
    record Mul(Expr left, Expr right) implements Expr {}
    record Neg(Expr expr) implements Expr {}

    // 密封类：限制哪些类可以继承它
    sealed abstract class Shape
        permits Circle, Rectangle, Triangle {}

    final class Circle extends Shape {
        final double radius;
        Circle(double radius) { this.radius = radius; }
    }

    // non-sealed：允许进一步扩展
    non-sealed class Rectangle extends Shape {
        final double w, h;
        Rectangle(double w, double h) { this.w = w; this.h = h; }
    }

    final class Triangle extends Shape {
        final double base, height;
        Triangle(double base, double height) { this.base = base; this.height = height; }
    }

    // 与 switch 模式匹配完美配合（Java 21）
    static double eval(Expr expr) {
        return switch (expr) {
            case Num(var v)         -> v;
            case Add(var l, var r)  -> eval(l) + eval(r);
            case Mul(var l, var r)  -> eval(l) * eval(r);
            case Neg(var e)         -> -eval(e);
        };
    }

    public static void main(String[] args) {
        // (3 + 4) * -2 = -14
        Expr expr = new Mul(
            new Add(new Num(3), new Num(4)),
            new Neg(new Num(2))
        );
        System.out.println("表达式结果: " + eval(expr));  // -14.0

        // 密封类的优势：编译器知道所有子类，switch 无需 default
        Shape shape = new Circle(5);
        double area = switch (shape) {
            case Circle c    -> Math.PI * c.radius * c.radius;
            case Rectangle r -> r.w * r.h;
            case Triangle t  -> 0.5 * t.base * t.height;
        };
        System.out.printf("面积: %.2f%n", area);
    }
}
```

## 6.2 增强型随机数 API（Java 17）

```java
import java.util.random.*;
import java.util.*;
import java.util.stream.*;

public class RandomApiDemo {
    public static void main(String[] args) {
        // 新接口层次：RandomGenerator -> SplittableGenerator -> JumpableGenerator...
        // 所有随机数生成器统一接口

        // 使用具体算法
        RandomGenerator rng = RandomGenerator.of("Xoshiro256PlusPlus");
        System.out.println("随机 int: " + rng.nextInt(1, 101));   // [1, 100]
        System.out.println("随机 double: " + rng.nextDouble(0, 1));

        // 列出所有可用算法
        RandomGeneratorFactory.all()
            .map(RandomGeneratorFactory::name)
            .sorted()
            .forEach(System.out::println);

        // 默认随机数（L64X128MixRandom）
        RandomGenerator def = RandomGeneratorFactory.getDefault().create();
        IntStream random5 = def.ints(5, 1, 100);
        System.out.println("随机5个: " + random5.boxed().collect(Collectors.toList()));

        // SplittableGenerator：适合并行场景
        SplittableGenerator splittable = RandomGeneratorFactory
            .<SplittableGenerator>of("L64X128MixRandom")
            .create();
        // 可分裂出新的独立生成器用于并行流
        long sum = splittable.splits(8)
            .flatMapToLong(gen -> gen.longs(100, 0, 1000))
            .sum();
        System.out.println("并行随机总和: " + sum);
    }
}
```

---

# Part 7: Java 21 新特性

## 7.1 虚拟线程（Virtual Threads，正式版）

```java
import java.util.concurrent.*;
import java.util.*;
import java.time.*;

public class VirtualThreadsDemo {
    public static void main(String[] args) throws Exception {
        // === 创建虚拟线程 ===

        // 方式1：Thread.ofVirtual()
        Thread vt = Thread.ofVirtual()
            .name("my-virtual-thread")
            .start(() -> System.out.println("运行在: " + Thread.currentThread()));
        vt.join();

        // 方式2：Thread.startVirtualThread()（快捷方式）
        Thread.startVirtualThread(() -> {
            System.out.println("虚拟线程: " + Thread.currentThread().isVirtual());
        }).join();

        // 方式3：ExecutorService（推荐，用于批量场景）
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<String>> futures = new ArrayList<>();
            for (int i = 0; i < 10; i++) {
                final int id = i;
                futures.add(executor.submit(() -> {
                    Thread.sleep(50);  // 模拟 IO 操作
                    return "Task-" + id + " done by " + Thread.currentThread().getName();
                }));
            }
            for (Future<String> f : futures) {
                System.out.println(f.get());
            }
        }

        // === 性能对比：虚拟线程 vs 平台线程 ===
        int taskCount = 10_000;

        // 平台线程池（有限容量）
        Instant start1 = Instant.now();
        try (ExecutorService pool = Executors.newFixedThreadPool(200)) {
            List<Future<?>> futures = new ArrayList<>();
            for (int i = 0; i < taskCount; i++) {
                futures.add(pool.submit(() -> { Thread.sleep(10); return null; }));
            }
            for (Future<?> f : futures) f.get();
        }
        System.out.println("平台线程池(200): " + Duration.between(start1, Instant.now()).toMillis() + "ms");

        // 虚拟线程（百万级别）
        Instant start2 = Instant.now();
        try (ExecutorService vExecutor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<?>> futures = new ArrayList<>();
            for (int i = 0; i < taskCount; i++) {
                futures.add(vExecutor.submit(() -> { Thread.sleep(10); return null; }));
            }
            for (Future<?> f : futures) f.get();
        }
        System.out.println("虚拟线程: " + Duration.between(start2, Instant.now()).toMillis() + "ms");

        // === 结构化并发（Java 21 预览）===
        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
            StructuredTaskScope.Subtask<String> user  = scope.fork(() -> fetchUser(1));
            StructuredTaskScope.Subtask<String> order = scope.fork(() -> fetchOrder(1));
            scope.join();           // 等待所有任务
            scope.throwIfFailed();  // 有失败则抛异常
            System.out.println(user.get() + " -> " + order.get());
        }
    }

    static String fetchUser(int id) throws InterruptedException {
        Thread.sleep(100); return "User-" + id;
    }
    static String fetchOrder(int id) throws InterruptedException {
        Thread.sleep(150); return "Order-" + id;
    }
}
```

## 7.2 Switch 模式匹配（Java 21 正式）

```java
public class SwitchPatternDemo {

    sealed interface Animal permits Dog, Cat, Bird {}
    record Dog(String name) implements Animal {}
    record Cat(String name, boolean indoor) implements Animal {}
    record Bird(String name, String species) implements Animal {}

    static String describe(Object obj) {
        return switch (obj) {
            // 类型模式
            case Integer i when i < 0 -> "负整数: " + i;
            case Integer i            -> "正整数: " + i;
            case String s when s.isEmpty() -> "空字符串";
            case String s             -> "字符串(%d): %s".formatted(s.length(), s);
            case int[] arr            -> "int数组，长度: " + arr.length;
            // null 处理
            case null                 -> "null值";
            default                   -> "其他: " + obj.getClass().getSimpleName();
        };
    }

    static String describeAnimal(Animal animal) {
        return switch (animal) {
            case Dog d        -> d.name() + " 是一只狗";
            case Cat(var n, true)  -> n + " 是一只室内猫";  // 解构 + 条件
            case Cat(var n, false) -> n + " 是一只室外猫";
            case Bird(var n, var s) when s.equals("Eagle") -> n + " 是老鹰！";
            case Bird b       -> b.name() + " 是一只" + b.species();
        };
    }

    public static void main(String[] args) {
        System.out.println(describe(-5));
        System.out.println(describe(42));
        System.out.println(describe("hello"));
        System.out.println(describe(""));
        System.out.println(describe(null));
        System.out.println(describe(new int[]{1,2,3}));

        System.out.println(describeAnimal(new Dog("Buddy")));
        System.out.println(describeAnimal(new Cat("Whiskers", true)));
        System.out.println(describeAnimal(new Bird("Sam", "Eagle")));
        System.out.println(describeAnimal(new Bird("Tweety", "Canary")));
    }
}
```

## 7.3 SequencedCollection（Java 21）

```java
import java.util.*;

public class SequencedCollectionDemo {
    public static void main(String[] args) {
        // SequencedCollection 接口：有顺序的集合
        // 新增方法：getFirst() getFirst() getLast() addFirst() addLast() reversed()

        // List
        List<String> list = new ArrayList<>(List.of("a", "b", "c", "d"));
        System.out.println("first: " + list.getFirst());  // a
        System.out.println("last:  " + list.getLast());   // d
        list.addFirst("Z");
        list.addLast("Z");
        System.out.println("after add: " + list);         // [Z, a, b, c, d, Z]
        list.removeFirst();
        list.removeLast();
        System.out.println("after remove: " + list);      // [a, b, c, d]

        // reversed() 返回逆序视图（不是副本！）
        List<String> reversed = list.reversed();
        System.out.println("reversed: " + reversed);      // [d, c, b, a]
        list.add("e");
        System.out.println("reversed after add: " + reversed);  // [e, d, c, b, a]

        // Deque 也实现了 SequencedCollection
        Deque<Integer> deque = new ArrayDeque<>(List.of(1, 2, 3));
        System.out.println("deque first: " + deque.getFirst());  // 1
        System.out.println("deque reversed: " + deque.reversed());

        // SequencedMap
        LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
        map.put("x", 1); map.put("y", 2); map.put("z", 3);
        System.out.println("first entry: " + map.firstEntry());  // x=1
        System.out.println("last entry:  " + map.lastEntry());   // z=3
        System.out.println("reversed map: " + map.reversed());
    }
}
```

## 7.4 Record Patterns（Java 21 正式）

```java
public class RecordPatternsDemo {

    record Point(int x, int y) {}
    record Line(Point start, Point end) {}
    record ColoredPoint(Point point, String color) {}

    static void printPoint(Object obj) {
        // 解构 record
        if (obj instanceof Point(int x, int y)) {
            System.out.printf("点: (%d, %d)%n", x, y);
        }
    }

    static void printLine(Object obj) {
        // 嵌套解构
        if (obj instanceof Line(Point(int x1, int y1), Point(int x2, int y2))) {
            System.out.printf("线: (%d,%d) -> (%d,%d)%n", x1, y1, x2, y2);
        }
    }

    static String describeShape(Object obj) {
        return switch (obj) {
            case Point(int x, int y) when x == 0 && y == 0 -> "原点";
            case Point(int x, int y) when x == 0           -> "Y轴上的点: " + y;
            case Point(int x, int y) when y == 0           -> "X轴上的点: " + x;
            case Point(int x, int y)                       -> "普通点(%d,%d)".formatted(x, y);
            case Line(Point s, Point e)                    -> "线段: %s -> %s".formatted(s, e);
            case ColoredPoint(Point p, String c)           -> "%s 颜色: %s".formatted(p, c);
            default                                        -> "未知";
        };
    }

    public static void main(String[] args) {
        printPoint(new Point(3, 4));
        printLine(new Line(new Point(0, 0), new Point(5, 5)));

        System.out.println(describeShape(new Point(0, 0)));
        System.out.println(describeShape(new Point(0, 5)));
        System.out.println(describeShape(new Point(3, 0)));
        System.out.println(describeShape(new Point(3, 4)));
        System.out.println(describeShape(new Line(new Point(1,1), new Point(2,2))));
        System.out.println(describeShape(new ColoredPoint(new Point(1,2), "red")));
    }
}
```

---
# Part 8: Java 18-20 其他特性

## 8.1 Simple Web Server（Java 18）

```java
// 命令行启动（无需代码）
// jwebserver -p 8080 -d /path/to/static/files

// 通过 API 启动
import com.sun.net.httpserver.*;
import java.net.*;
import java.nio.file.*;

public class SimpleWebServerDemo {
    public static void main(String[] args) throws Exception {
        // 创建简单文件服务器
        InetSocketAddress addr = new InetSocketAddress(8080);
        Path root = Path.of(".");

        HttpServer server = SimpleFileServer.createFileServer(addr, root,
            SimpleFileServer.OutputLevel.VERBOSE);
        server.start();
        System.out.println("服务器启动: http://localhost:8080");

        // 自定义处理器
        server.createContext("/api/hello", exchange -> {
            String response = "{\"message\": \"Hello from Java 18!\"}";
            exchange.getResponseHeaders().set("Content-Type", "application/json");
            exchange.sendResponseHeaders(200, response.length());
            exchange.getResponseBody().write(response.getBytes());
            exchange.getResponseBody().close();
        });

        // 5秒后停止（演示用）
        Thread.sleep(5000);
        server.stop(0);
    }
}
```

## 8.2 代码片段（@snippet，Java 18 Javadoc）

```java
/**
 * 工具类示例，展示 Java 18 的 @snippet 标签。
 *
 * {@snippet lang="java" :
 *   // 使用示例
 *   var result = Utils.format("hello %s", "world");
 *   System.out.println(result); // hello world
 * }
 */
public class Utils {
    public static String format(String template, Object... args) {
        return String.format(template, args);
    }
}
```

## 8.3 Pattern Matching for switch（Java 19-20 预览，Java 21 正式）

（已在 Part 7.2 中详细介绍）

## 8.4 Record Patterns（Java 19-20 预览，Java 21 正式）

（已在 Part 7.4 中详细介绍）

## 8.5 Foreign Function & Memory API（Java 19-21 孵化/预览）

```java
// FFM API 允许 Java 直接调用本地库和操作堆外内存，无需 JNI
import java.lang.foreign.*;
import java.lang.invoke.*;

public class FfmApiDemo {
    public static void main(String[] args) throws Throwable {
        // 调用 C 标准库的 strlen 函数
        Linker linker = Linker.nativeLinker();
        SymbolLookup stdlib = linker.defaultLookup();

        MethodHandle strlen = linker.downcallHandle(
            stdlib.find("strlen").orElseThrow(),
            FunctionDescriptor.of(ValueLayout.JAVA_LONG, ValueLayout.ADDRESS)
        );

        try (Arena arena = Arena.ofConfined()) {
            MemorySegment str = arena.allocateUtf8String("Hello FFM API!");
            long len = (long) strlen.invoke(str);
            System.out.println("strlen = " + len);  // 14
        }

        // 操作堆外内存
        try (Arena arena = Arena.ofConfined()) {
            // 分配 1KB 堆外内存
            MemorySegment segment = arena.allocate(1024);
            // 写入数据
            segment.setAtIndex(ValueLayout.JAVA_INT, 0, 42);
            segment.setAtIndex(ValueLayout.JAVA_INT, 1, 100);
            // 读取数据
            int v1 = segment.getAtIndex(ValueLayout.JAVA_INT, 0);
            int v2 = segment.getAtIndex(ValueLayout.JAVA_INT, 1);
            System.out.println(v1 + " " + v2);  // 42 100
        }
    }
}
```

## 8.6 虚拟线程（Java 19-20 预览，Java 21 正式）

（已在 Part 7.1 中详细介绍）

## 8.7 String Templates（Java 21 预览，Java 23 废弃后重新设计中）

```java
// 注意：String Templates 在 Java 21 作为预览特性引入
// 但在 Java 23 被撤回，正在重新设计中
// 以下为 Java 21 预览版本的语法

// import static java.lang.StringTemplate.STR;

// String name = "World";
// int year = 2024;
// String greeting = STR."Hello, \{name}! Year: \{year}";
// System.out.println(greeting);  // Hello, World! Year: 2024

// 多行模板
// String json = STR."""
//     {
//       "name": "\{name}",
//       "year": \{year}
//     }
//     """;

// 目前建议使用 String.formatted() 或 MessageFormat 替代
String name = "World";
int year = 2024;
String greeting = "Hello, %s! Year: %d".formatted(name, year);
System.out.println(greeting);
```

---

# Part 9: 各版本新特性速查表

## Java 8-21 主要特性对比

| 特性 | 引入版本 | 状态 | 简介 |
|------|---------|------|------|
| Lambda 表达式 | Java 8 | 正式 | 函数式编程基础 |
| Stream API | Java 8 | 正式 | 声明式数据处理 |
| Optional | Java 8 | 正式 | 空值安全容器 |
| 接口 default/static 方法 | Java 8 | 正式 | 接口演化支持 |
| java.time API | Java 8 | 正式 | 现代日期时间处理 |
| CompletableFuture | Java 8 | 正式 | 异步编程 |
| JPMS 模块系统 | Java 9 | 正式 | 代码封装与依赖管理 |
| 集合工厂方法 | Java 9 | 正式 | List/Set/Map.of() |
| JShell | Java 9 | 正式 | 交互式 REPL |
| Stream 增强 | Java 9 | 正式 | takeWhile/dropWhile/iterate |
| Flow API | Java 9 | 正式 | 响应式流标准接口 |
| var 类型推断 | Java 10 | 正式 | 局部变量类型推断 |
| copyOf | Java 10 | 正式 | 不可变集合副本 |
| String 新方法 | Java 11 | 正式 | isBlank/strip/lines/repeat |
| Files 新方法 | Java 11 | 正式 | readString/writeString |
| HTTP Client | Java 11 | 正式 | 内置 HTTP/2 客户端 |
| Switch 表达式 | Java 14 | 正式 | 表达式形式的 switch |
| 有帮助的 NPE | Java 14 | 正式 | 精准定位 NPE 位置 |
| Text Blocks | Java 15 | 正式 | 多行字符串字面量 |
| Records | Java 16 | 正式 | 不可变数据载体 |
| instanceof 模式匹配 | Java 16 | 正式 | 类型检查+自动转型 |
| 密封类 | Java 17 | 正式 | 限制继承层次 |
| 随机数 API 增强 | Java 17 | 正式 | RandomGenerator 统一接口 |
| Simple Web Server | Java 18 | 正式 | 轻量静态文件服务器 |
| 虚拟线程 | Java 21 | 正式 | 轻量级用户态线程 |
| Switch 模式匹配 | Java 21 | 正式 | switch 支持类型/解构模式 |
| Record Patterns | Java 21 | 正式 | record 解构匹配 |
| SequencedCollection | Java 21 | 正式 | 统一有序集合接口 |
| 结构化并发 | Java 21 | 预览 | 任务树管理并发 |

## 核心 API 演化对比

```
日期时间处理演化：
Date/Calendar → Joda-Time（第三方）→ java.time (Java 8)

字符串多行：
"line1\nline2\n" → 字符串连接 → Text Block (Java 15)

集合创建：
Arrays.asList() → Collections.unmodifiableList() → List.of() (Java 9)

类型检查：
if (x instanceof Foo) { Foo f = (Foo)x; } → if (x instanceof Foo f) {} (Java 16)

HTTP 请求：
URLConnection → Apache HttpClient（第三方）→ HttpClient (Java 11)

并发线程：
Thread → ExecutorService → ForkJoinPool → Virtual Threads (Java 21)

空值处理：
null 检查 → @Nullable 注解 → Optional (Java 8)
```

---

# Part 10: 实战整合案例

## 案例1：电商订单分析系统（综合使用 Java 8-16 特性）

```java
import java.time.*;
import java.util.*;
import java.util.stream.*;

// Records (Java 16)
record Product(String id, String name, String category, double price) {}
record OrderItem(Product product, int quantity) {
    double subtotal() { return product.price() * quantity; }
}
record Order(
    String id,
    String customerId,
    List<OrderItem> items,
    LocalDateTime createdAt,
    String status
) {
    double total() {
        return items.stream().mapToDouble(OrderItem::subtotal).sum();
    }
}

public class EcommerceAnalytics {

    static List<Order> generateOrders() {
        var p1 = new Product("P001", "MacBook Pro", "Electronics", 12999.0);
        var p2 = new Product("P002", "iPhone 15",  "Electronics",  6999.0);
        var p3 = new Product("P003", "AirPods Pro", "Accessories",  1999.0);
        var p4 = new Product("P004", "Java编程思想", "Books",          89.0);
        var p5 = new Product("P005", "机械键盘",    "Accessories",    599.0);

        return List.of(
            new Order("O001", "C001",
                List.of(new OrderItem(p1, 1), new OrderItem(p3, 2)),
                LocalDateTime.of(2024, 1, 15, 10, 30), "PAID"),
            new Order("O002", "C002",
                List.of(new OrderItem(p2, 2), new OrderItem(p4, 3)),
                LocalDateTime.of(2024, 1, 20, 14, 0), "PAID"),
            new Order("O003", "C001",
                List.of(new OrderItem(p5, 1)),
                LocalDateTime.of(2024, 2, 1, 9, 15), "PAID"),
            new Order("O004", "C003",
                List.of(new OrderItem(p1, 1), new OrderItem(p2, 1)),
                LocalDateTime.of(2024, 2, 10, 16, 45), "PAID"),
            new Order("O005", "C002",
                List.of(new OrderItem(p3, 1)),
                LocalDateTime.of(2024, 2, 15, 11, 0), "CANCELLED"),
            new Order("O006", "C004",
                List.of(new OrderItem(p4, 5), new OrderItem(p5, 2)),
                LocalDateTime.of(2024, 3, 1, 8, 30), "PAID")
        );
    }

    public static void main(String[] args) {
        var orders = generateOrders();
        var paidOrders = orders.stream()
            .filter(o -> "PAID".equals(o.status()))
            .collect(Collectors.toList());

        // 1. 总销售额
        double totalRevenue = paidOrders.stream()
            .mapToDouble(Order::total).sum();
        System.out.printf("总销售额: ¥%.2f%n", totalRevenue);

        // 2. 各类别销售额（使用 flatMap 展开订单项）
        System.out.println("\n各类别销售额:");
        paidOrders.stream()
            .flatMap(o -> o.items().stream())
            .collect(Collectors.groupingBy(
                item -> item.product().category(),
                Collectors.summingDouble(OrderItem::subtotal)
            ))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .forEach(e -> System.out.printf("  %-15s ¥%.2f%n", e.getKey(), e.getValue()));

        // 3. Top 3 商品（按销售金额）
        System.out.println("\nTop 3 热销商品:");
        paidOrders.stream()
            .flatMap(o -> o.items().stream())
            .collect(Collectors.groupingBy(
                item -> item.product().name(),
                Collectors.summingDouble(OrderItem::subtotal)
            ))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Double>comparingByValue().reversed())
            .limit(3)
            .forEach(e -> System.out.printf("  %s: ¥%.2f%n", e.getKey(), e.getValue()));

        // 4. 客户消费分析
        System.out.println("\n客户消费分析:");
        paidOrders.stream()
            .collect(Collectors.groupingBy(
                Order::customerId,
                Collectors.summarizingDouble(Order::total)
            ))
            .forEach((cid, stats) ->
                System.out.printf("  %s: 订单数=%d 总消费=¥%.2f 平均=¥%.2f%n",
                    cid, stats.getCount(), stats.getSum(), stats.getAverage()));

        // 5. 月度销售趋势（java.time）
        System.out.println("\n月度销售趋势:");
        paidOrders.stream()
            .collect(Collectors.groupingBy(
                o -> YearMonth.from(o.createdAt()),
                TreeMap::new,  // 保证顺序
                Collectors.summingDouble(Order::total)
            ))
            .forEach((month, amount) ->
                System.out.printf("  %s: ¥%.2f%n", month, amount));

        // 6. 统计使用 Optional
        OptionalDouble maxOrder = paidOrders.stream()
            .mapToDouble(Order::total).max();
        maxOrder.ifPresent(max -> System.out.printf("%n最大订单: ¥%.2f%n", max));
    }
}
```

## 案例2：并发任务处理器（虚拟线程 + CompletableFuture）

```java
import java.util.*;
import java.util.concurrent.*;
import java.util.function.*;
import java.time.*;

public class ConcurrentTaskProcessor {

    record Task(String id, String type, String payload) {}
    record TaskResult(String taskId, String result, Duration duration, boolean success) {}

    // 模拟不同类型的处理器
    static final Map<String, Function<String, String>> PROCESSORS = Map.of(
        "EMAIL",  payload -> { sleep(50);  return "Email sent: " + payload; },
        "SMS",    payload -> { sleep(30);  return "SMS sent: " + payload; },
        "NOTIFY", payload -> { sleep(20);  return "Notification: " + payload; },
        "REPORT", payload -> { sleep(200); return "Report generated: " + payload; }
    );

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    static TaskResult process(Task task) {
        Instant start = Instant.now();
        try {
            Function<String, String> processor = PROCESSORS.getOrDefault(
                task.type(), p -> "Unknown type: " + p);
            String result = processor.apply(task.payload());
            return new TaskResult(task.id(), result, Duration.between(start, Instant.now()), true);
        } catch (Exception e) {
            return new TaskResult(task.id(), "Error: " + e.getMessage(),
                Duration.between(start, Instant.now()), false);
        }
    }

    public static void main(String[] args) throws Exception {
        // 生成100个混合任务
        List<Task> tasks = new ArrayList<>();
        String[] types = {"EMAIL", "SMS", "NOTIFY", "REPORT"};
        for (int i = 0; i < 100; i++) {
            tasks.add(new Task("T" + String.format("%03d", i),
                types[i % types.length], "payload-" + i));
        }

        // 使用虚拟线程并发处理
        Instant start = Instant.now();
        List<TaskResult> results;
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<CompletableFuture<TaskResult>> futures = tasks.stream()
                .map(task -> CompletableFuture.supplyAsync(() -> process(task), executor))
                .collect(Collectors.toList());

            results = futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
        }

        Duration total = Duration.between(start, Instant.now());

        // 分析结果
        long success = results.stream().filter(TaskResult::success).count();
        long failed  = results.stream().filter(r -> !r.success()).count();
        OptionalDouble avgMs = results.stream()
            .mapToDouble(r -> r.duration().toMillis()).average();

        System.out.println("===== 任务处理报告 =====");
        System.out.printf("总任务: %d | 成功: %d | 失败: %d%n", tasks.size(), success, failed);
        System.out.printf("总耗时: %dms | 平均: %.1fms%n", total.toMillis(), avgMs.orElse(0));

        // 按类型统计
        results.stream()
            .collect(Collectors.groupingBy(
                r -> tasks.stream().filter(t -> t.id().equals(r.taskId()))
                    .findFirst().map(Task::type).orElse("?"),
                Collectors.averagingDouble(r -> r.duration().toMillis())
            ))
            .forEach((type, avg) ->
                System.out.printf("  %s 平均耗时: %.1fms%n", type, avg));
    }
}
```

## 案例3：领域模型设计（密封类 + Records + Pattern Matching）

```java
import java.util.*;

public class DomainModelDemo {

    // 支付结果：密封类表示有限状态
    sealed interface PaymentResult
        permits PaymentResult.Success, PaymentResult.Failure, PaymentResult.Pending {}

    record Success(String transactionId, double amount) implements PaymentResult {}
    record Failure(String errorCode, String message)   implements PaymentResult {}
    record Pending(String requestId, String checkUrl)  implements PaymentResult {}

    // 用 switch 模式匹配处理所有情况
    static String handlePayment(PaymentResult result) {
        return switch (result) {
            case Success(var txId, var amount) ->
                "支付成功！交易号: %s, 金额: ¥%.2f".formatted(txId, amount);
            case Failure(var code, var msg) ->
                "支付失败！错误码: %s, 原因: %s".formatted(code, msg);
            case Pending(var reqId, var url) ->
                "处理中，请访问 %s 查询 %s 的结果".formatted(url, reqId);
        };
    }

    // 表达式树（密封类 + record 递归）
    sealed interface Expression
        permits Literal, Variable, BinaryOp, UnaryOp {}

    record Literal(double value)                             implements Expression {}
    record Variable(String name)                             implements Expression {}
    record BinaryOp(String op, Expression l, Expression r)  implements Expression {}
    record UnaryOp(String op, Expression e)                  implements Expression {}

    static double evaluate(Expression expr, Map<String, Double> env) {
        return switch (expr) {
            case Literal(var v)             -> v;
            case Variable(var name)         -> env.getOrDefault(name, 0.0);
            case BinaryOp("+", var l, var r) -> evaluate(l, env) + evaluate(r, env);
            case BinaryOp("-", var l, var r) -> evaluate(l, env) - evaluate(r, env);
            case BinaryOp("*", var l, var r) -> evaluate(l, env) * evaluate(r, env);
            case BinaryOp("/", var l, var r) -> evaluate(l, env) / evaluate(r, env);
            case UnaryOp("-", var e)         -> -evaluate(e, env);
            case UnaryOp("abs", var e)       -> Math.abs(evaluate(e, env));
            default                          -> throw new IllegalArgumentException("未知操作: " + expr);
        };
    }

    public static void main(String[] args) {
        // 支付结果处理
        List<PaymentResult> results = List.of(
            new Success("TXN-001", 299.99),
            new Failure("INSUFFICIENT_FUNDS", "余额不足"),
            new Pending("REQ-003", "https://pay.example.com/check/REQ-003")
        );
        results.forEach(r -> System.out.println(handlePayment(r)));

        // 表达式求值: (x + 2) * (y - 1)
        Expression expr = new BinaryOp("*",
            new BinaryOp("+", new Variable("x"), new Literal(2)),
            new BinaryOp("-", new Variable("y"), new Literal(1))
        );
        Map<String, Double> env = Map.of("x", 3.0, "y", 5.0);
        System.out.printf("%n(x+2)*(y-1) = %.1f%n", evaluate(expr, env));  // (3+2)*(5-1) = 20.0
    }
}
```

---

# Part 11: 面试题 FAQ

## Q1: Lambda 表达式和匿名内部类有什么区别？

```
答：主要区别如下：

1. this 关键字
   - 匿名内部类：this 指向匿名内部类自身实例
   - Lambda：this 指向外部类实例（无自己的 this）

2. 变量捕获
   - 匿名内部类：可以捕获 effectively final 局部变量，也可以访问自身字段
   - Lambda：只能捕获 effectively final 的局部变量

3. 编译方式
   - 匿名内部类：编译生成 Outer$1.class 文件
   - Lambda：使用 invokedynamic 指令（更高效，延迟绑定）

4. 适用范围
   - 匿名内部类：可实现有多个方法的接口/抽象类
   - Lambda：只能用于函数式接口（只有一个抽象方法）

5. 性能
   - Lambda 在大多数情况下比匿名内部类性能更好
```

## Q2: Stream 的 map 和 flatMap 有什么区别？

```java
// map：一对一映射，每个元素产生一个元素
Stream.of("hello", "world")
      .map(s -> s.toUpperCase())  // Stream<String> -> Stream<String>
      .forEach(System.out::println);  // HELLO WORLD

// flatMap：一对多映射，每个元素产生零个或多个元素，然后"拍平"
Stream.of("hello world", "java 21")
      .flatMap(s -> Arrays.stream(s.split(" ")))  // Stream<String> -> Stream<String>
      .forEach(System.out::println);  // hello world java 21

// 经典场景：展开嵌套集合
List<List<Integer>> nested = List.of(List.of(1,2), List.of(3,4), List.of(5));
List<Integer> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());  // [1, 2, 3, 4, 5]

// Optional.flatMap：避免 Optional<Optional<T>>
Optional<String> email = Optional.of("user@example.com");
// map 返回 Optional<Optional<String>>
Optional<Optional<String>> bad = email.map(s -> Optional.of(s.toLowerCase()));
// flatMap 返回 Optional<String>
Optional<String> good = email.flatMap(s -> Optional.of(s.toLowerCase()));
```

## Q3: Optional 应该如何正确使用？

```java
// ❌ 错误用法1：用 get() 不检查
Optional<String> opt = findUser();
String name = opt.get();  // 可能 NoSuchElementException

// ❌ 错误用法2：直接判断 isPresent() 再 get()（等同 null 检查，没有意义）
if (opt.isPresent()) {
    String name2 = opt.get();  // 没有充分利用 Optional
}

// ❌ 错误用法3：方法参数中使用 Optional
void process(Optional<String> name) {}  // 不推荐

// ❌ 错误用法4：集合中放 Optional
List<Optional<String>> list = new ArrayList<>();  // 不推荐

// ✅ 正确用法1：orElse / orElseGet
String name = opt.orElse("匿名用户");
String computed = opt.orElseGet(() -> generateDefaultName());  // 懒加载

// ✅ 正确用法2：链式处理
String result = opt.map(String::toUpperCase)
                   .filter(s -> s.length() > 3)
                   .orElse("DEFAULT");

// ✅ 正确用法3：ifPresent / ifPresentOrElse
opt.ifPresent(s -> System.out.println("Found: " + s));
opt.ifPresentOrElse(
    s -> System.out.println("Found: " + s),
    () -> System.out.println("Not found")
);

// ✅ 正确用法4：返回值中使用 Optional（推荐）
Optional<User> findUser(int id) {
    return userRepository.findById(id);
}
```

## Q4: Java 8 的 Stream 是懒执行的吗？

```java
// Stream 的中间操作是懒执行的，终端操作触发执行

List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "Dave", "Eve");

// 演示懒执行
Optional<String> result = names.stream()
    .filter(s -> {
        System.out.println("filter: " + s);
        return s.length() > 3;
    })
    .map(s -> {
        System.out.println("map: " + s);
        return s.toUpperCase();
    })
    .findFirst();  // 终端操作，找到第一个就停止

// 输出：
// filter: Alice  （length=5，通过）
// map: Alice     （映射）
// 结果：ALICE
// Bob、Charlie 等不会被处理！

// 这就是"短路"特性，极大提高效率
System.out.println("结果: " + result.orElse("none"));
```

## Q5: var 关键字（局部变量类型推断）有哪些限制？

```
1. 只能用于局部变量（方法内部、for 循环变量、try-with-resources）
2. 声明时必须初始化：var x; // 错误
3. 不能初始化为 null：var x = null; // 错误（无法推断类型）
4. 不能用于方法参数、返回类型、字段
5. 不能用于数组初始化器：var arr = {1,2,3}; // 错误
6. Lambda 表达式本身不能赋给 var：var f = x -> x+1; // 错误（需显式指定函数接口）
7. 可以用于 Lambda 参数（Java 11+）：(var x, var y) -> x + y
```

## Q6: Record 有哪些限制？

```
1. 隐式 final：不能被继承，也不能继承其他类（可实现接口）
2. 所有字段都是 private final（不可变）
3. 不能显式声明实例字段（组件外的字段）
4. 不能使用 native 方法
5. compact 构造器中不能访问字段（只能通过参数访问）
6. 可以定义静态字段、静态方法、实例方法
7. 可以实现接口
8. 可以有泛型参数
9. 不适合作为 JPA/Hibernate 实体（需要无参构造器）
```

## Q7: 虚拟线程和平台线程的区别？

```
平台线程（OS Thread）：
- 与操作系统线程 1:1 映射
- 创建成本高（约 1MB 栈内存）
- 阻塞时占用 OS 线程
- JVM 中通常最多数千个

虚拟线程（Virtual Thread）：
- M:N 映射（多个虚拟线程复用少量平台线程）
- 创建成本极低（约 几KB）
- 阻塞时（IO、sleep 等）自动卸载，不占用平台线程
- JVM 中可创建数百万个
- 适合 IO 密集型任务

注意事项：
- CPU 密集型任务用虚拟线程无优势
- 虚拟线程中避免使用 synchronized（会 pin 住平台线程），用 ReentrantLock 替代
- ThreadLocal 依然有效，但大量虚拟线程时内存占用要注意
- 不要池化虚拟线程，随用随创建
```

## Q8: 密封类（Sealed Class）的作用是什么？

```java
// 密封类解决的问题：在某些场景下，需要严格控制继承层次
// 例如：编译器辅助的穷举检查

sealed interface Result<T> permits Success, Failure {}
record Success<T>(T value) implements Result<T> {}
record Failure<T>(String error) implements Result<T> {}

// 使用 switch 时，编译器知道所有子类，无需 default
static <T> String handle(Result<T> result) {
    return switch (result) {
        case Success<T> s -> "成功: " + s.value();
        case Failure<T> f -> "失败: " + f.error();
        // 无需 default，编译器确保穷举
    };
}

// 好处：
// 1. 穷举性：switch 不写 default 也不会警告/错误
// 2. 领域建模：精确表达"有限的子类型"
// 3. 安全性：防止第三方随意继承破坏协议
```

## Q9: Java 21 新增的 SequencedCollection 解决什么问题？

```java
// Java 21 之前，不同集合获取首尾元素方式不统一：
list.get(0);          list.get(list.size()-1);  // List
deque.peekFirst();    deque.peekLast();          // Deque
sortedSet.first();    sortedSet.last();          // SortedSet

// Java 21 之后，统一接口：
list.getFirst();  list.getLast();    // 同一方式
deque.getFirst(); deque.getLast();
// 都来自 SequencedCollection 接口

// 新增 reversed() 获取逆序视图（非副本）
// addFirst() / addLast() / removeFirst() / removeLast()
// SequencedMap 对应 firstEntry() / lastEntry() / reversed()
```

## Q10: Stream 的并行流需要注意哪些问题？

```java
// 1. 线程安全：避免修改共享可变状态
List<Integer> unsafeResult = new ArrayList<>();
// 错误：
numbers.parallelStream().forEach(unsafeResult::add);  // 竞态条件！
// 正确：
List<Integer> safeResult = numbers.parallelStream().collect(Collectors.toList());

// 2. 性能：并行不一定更快
// - 数据量小：线程开销 > 收益
// - 简单操作：串行反而更快
// - 数据结构：ArrayList 拆分效率高，LinkedList 差

// 3. 顺序：并行流不保证顺序（除非用 forEachOrdered）
numbers.parallelStream().forEachOrdered(System.out::println);  // 保证顺序但降低并行效率

// 4. 合适场景
// - CPU 密集型运算（非 IO）
// - 数据量大（通常 > 10万）
// - 无共享状态
// - 操作独立、可组合

// 5. 自定义线程池（避免影响 common pool）
ForkJoinPool pool = new ForkJoinPool(4);
long result = pool.submit(() ->
    numbers.parallelStream().mapToLong(n -> heavyCompute(n)).sum()
).get();
pool.shutdown();
```

## Q11: CompletableFuture thenApply vs thenCompose 的区别？

```java
// thenApply：同步转换，f(T) -> U，不产生新的 Future
CompletableFuture<String> f1 = CompletableFuture
    .supplyAsync(() -> "hello")
    .thenApply(s -> s.toUpperCase());  // String -> String
// 结果类型：CompletableFuture<String>

// thenCompose：异步转换，f(T) -> CompletableFuture<U>，拍平嵌套
CompletableFuture<String> f2 = CompletableFuture
    .supplyAsync(() -> 1)
    .thenCompose(id -> CompletableFuture.supplyAsync(() -> "User-" + id)); // 避免嵌套
// 结果类型：CompletableFuture<String>
// 如果用 thenApply：CompletableFuture<CompletableFuture<String>>（嵌套，不好用）

// 类比：
// thenApply  <-> Stream.map
// thenCompose <-> Stream.flatMap
```

## Q12: Java 的 var 与 JavaScript 的 var 有何不同？

```
Java var（Java 10+）：
- 静态类型推断，编译时确定类型，运行时类型完全确定
- 类型一旦推断不可改变（仍然是强类型）
- 只是语法糖，不影响字节码
- 只能用于局部变量

JavaScript var：
- 动态类型，变量本身无类型限制
- 可以赋予不同类型的值
- 有函数作用域（而非块作用域）
- 存在变量提升问题
- 现代 JS 推荐用 let/const 替代

// Java:
var x = "hello";
x = 42;  // 编译错误！x 已推断为 String

// JavaScript:
var x = "hello";
x = 42;  // 完全合法
```

## Q13: Records 与 Lombok @Data 注解有什么区别？

```
Records（Java 16+，语言内置）：
+ 无需第三方依赖
+ 编译器原生支持，IDE/工具链兼容性好
+ 隐式 final，不可变
+ 语法简洁
- 不能继承其他类
- 字段必须是 final
- 不适合需要可变状态的场景

Lombok @Data（第三方）：
+ 支持可变字段
+ 可以继承其他类
+ 更灵活（@Builder, @Setter 等）
- 需要额外依赖
- IDE 插件支持
- 编译时字节码生成，调试时 stacktrace 可能混乱
- 有时与其他注解处理器冲突

选择建议：
- 新项目 Java 16+ 且数据不可变 -> Records
- 需要可变/Builder/继承     -> Lombok
- 已有 Lombok 的项目        -> 保持一致性
```

## Q14: 什么是有效最终变量（effectively final）？

```java
// Lambda 和匿名内部类只能捕获 effectively final 变量
// "有效最终" = 声明后没有被重新赋值（不需要显式 final）

int x = 10;  // effectively final（从未被重新赋值）
Runnable r = () -> System.out.println(x);  // OK

int y = 10;
y = 20;  // y 不再是 effectively final
// Runnable r2 = () -> System.out.println(y);  // 编译错误

// 解决方法：
// 1. 用 final 修饰（显式）
final int z = 10;

// 2. 借助数组（引用是 final 的，但内容可变 - 不推荐）
int[] arr = {10};
arr[0] = 20;  // 合法但不好的实践
Runnable r3 = () -> System.out.println(arr[0]);

// 3. 借助 AtomicInteger
AtomicInteger count = new AtomicInteger(0);
Runnable r4 = () -> count.incrementAndGet();
```

## Q15: Java 9 的模块系统（JPMS）解决了什么问题？

```
JPMS（Java Platform Module System）解决的问题：

1. JAR Hell：类路径混乱，同一类多个版本冲突
   - JPMS 提供严格的模块依赖声明，版本冲突在启动时即发现

2. 封装不足：反射可以访问任何 private 成员
   - JPMS 强制 exports 声明，未导出包对外不可见
   - 限制了深度反射访问

3. JDK 本身也被模块化（约 98 个模块）
   - 减少应用程序大小（jlink 工具制作精简 JRE）
   - 例如：java --list-modules 可见所有模块

4. 安全性提升
   - 内部 API（如 sun.misc.Unsafe）默认不可访问

实际场景：
- 微服务：每个服务只包含需要的 JDK 模块，镜像更小
- 安全：防止框架通过反射访问业务代码的私有实现
- 性能：模块系统可以进行更好的 AOT 编译优化

注意：大多数应用仍在 classpath 模式运行（兼容性模式），
完全迁移到 JPMS 需要所有依赖库也提供 module-info.java
```

---

# 附录：学习资源与建议

## 官方文档

- JEP（Java Enhancement Proposals）：每个新特性都对应一个 JEP 文档
- OpenJDK 官网：https://openjdk.org
- Java SE API 文档：https://docs.oracle.com/en/java/javase/

## 学习路径建议

```
初学者：
  Java 8 核心三件套
  └─ Lambda → Stream API → Optional
  └─ 新日期时间 API
  └─ 接口 default 方法

进阶：
  Java 9-11 实用特性
  └─ 集合工厂方法（List.of 等）
  └─ var 类型推断
  └─ String 新方法
  └─ HTTP Client

现代 Java：
  Java 14-16 语法糖
  └─ Switch 表达式
  └─ Text Block
  └─ Records
  └─ instanceof 模式匹配

高级特性：
  Java 17-21
  └─ 密封类 + 模式匹配 switch
  └─ 虚拟线程（IO 密集型场景）
  └─ Record Patterns
  └─ SequencedCollection
```

## 版本选择建议

| 场景 | 推荐版本 | 原因 |
|------|---------|------|
| 新项目 | Java 21 (LTS) | 最新 LTS，虚拟线程、模式匹配等全部特性 |
| 企业老项目 | Java 17 (LTS) | 较新 LTS，密封类、Records 等 |
| 保守升级 | Java 11 (LTS) | 广泛支持，HTTP Client 等实用特性 |
| 最低要求 | Java 8 (LTS) | Stream/Lambda，仍被广泛使用 |

---
*文档版本：2024年，覆盖 Java 8 至 Java 21 主要新特性*
*代码示例均可在对应 Java 版本及以上环境中运行*