# Java 并发编程（JUC）详解 - 从零到精通

> 版本：JDK 8/11/17 | 难度：★★★★★ | 适合：有一定 Java 基础，想深入理解并发的开发者

---

## 目录

- [Part 1: 并发基础](#part-1-并发基础)
- [Part 2: Java 内存模型（JMM）](#part-2-java内存模型jmm)
- [Part 3: synchronized 深度解析](#part-3-synchronized深度解析)
- [Part 4: AQS 深度解析](#part-4-aqs深度解析)
- [Part 5: ReentrantLock 深度解析](#part-5-reentrantlock深度解析)
- [Part 6: 读写锁](#part-6-读写锁)
- [Part 7: 并发工具类](#part-7-并发工具类)
- [Part 8: 原子类（Atomic）](#part-8-原子类atomic)
- [Part 9: 并发集合](#part-9-并发集合)
- [Part 10: 线程池深度解析](#part-10-线程池threadpoolexecutor深度解析)
- [Part 11: Future 与异步编程](#part-11-future与异步编程)
- [Part 12: ThreadLocal 深度解析](#part-12-threadlocal深度解析)
- [Part 13: 死锁](#part-13-死锁)
- [Part 14: 完整实战案例](#part-14-完整实战案例)
- [Part 15: 常见面试题 FAQ](#part-15-常见面试题-faq)

---
# Part 1: 并发基础

## 1.1 进程 vs 线程 vs 协程

### 进程（Process）

进程是操作系统进行**资源分配和调度**的基本单位。每个进程拥有独立的内存空间、文件描述符、信号处理器等系统资源。

```
进程内存布局（简化）：
┌─────────────────────────────────────┐
│              内核空间                │  <- 操作系统
├─────────────────────────────────────┤
│               栈 (Stack)            │  <- 向下增长，局部变量
│                  ↓                  │
│               空 闲                  │
│                  ↑                  │
│               堆 (Heap)             │  <- 向上增长，动态分配
├─────────────────────────────────────┤
│          BSS段（未初始化全局变量）    │
├─────────────────────────────────────┤
│          Data段（已初始化全局变量）   │
├─────────────────────────────────────┤
│          Text段（代码段）            │
└─────────────────────────────────────┘
```

**进程特点：**
- 独立的地址空间，进程间通信（IPC）需要特殊机制（管道、消息队列、共享内存、Socket）
- 创建开销大（fork 系统调用）
- 一个进程崩溃不影响其他进程（隔离性强）

### 线程（Thread）

线程是 CPU **调度和执行**的基本单位，是进程内的执行流。同一进程内的线程**共享**堆内存、方法区（元空间）、文件描述符等，但每个线程拥有独立的：
- **程序计数器（PC）**：记录当前执行到哪条指令
- **虚拟机栈（JVM Stack）**：每个方法调用对应一个栈帧
- **本地方法栈（Native Method Stack）**

```
同一进程内多线程内存示意：
┌──────────────────────────────────────────────────────┐
│                      进程空间                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐           │
│  │ 线程1栈  │  │ 线程2栈  │  │ 线程3栈  │           │
│  │  (独立)  │  │  (独立)  │  │  (独立)  │           │
│  └──────────┘  └──────────┘  └──────────┘           │
│  ┌────────────────────────────────────────┐          │
│  │          堆内存（共享）                 │          │
│  │    对象实例、数组 ...                   │          │
│  └────────────────────────────────────────┘          │
│  ┌────────────────────────────────────────┐          │
│  │         方法区/元空间（共享）           │          │
│  │    类信息、静态变量、常量池 ...         │          │
│  └────────────────────────────────────────┘          │
└──────────────────────────────────────────────────────┘
```

**线程特点：**
- 创建/切换开销比进程小（上下文切换仍有代价）
- 共享内存带来并发问题（可见性、原子性、有序性）
- 一个线程崩溃可能导致整个进程崩溃

### 协程（Coroutine）

协程是**用户态**的轻量级线程，由程序自行调度（非操作系统），也叫"绿色线程"。

| 对比维度 | 进程 | 线程（Java） | 协程（Kotlin/Project Loom） |
|---------|------|------------|--------------------------|
| 创建开销 | 极大 | 约 1MB 栈内存 | 极小（KB级） |
| 切换开销 | 大（内核态） | 中（内核态） | 极小（用户态） |
| 隔离性 | 强 | 弱（共享堆） | 弱（共享堆） |
| 数量上限 | 数百 | 数千 | 数百万 |
| Java 支持 | ProcessBuilder | Thread/JUC | Java 21 Virtual Thread |

**Java 21 虚拟线程（Virtual Thread）示例：**
```java
// 传统线程
Thread t = new Thread(() -> System.out.println("传统线程"));
t.start();

// Java 21 虚拟线程
Thread vt = Thread.ofVirtual().start(() -> System.out.println("虚拟线程"));

// 使用 Executor 创建百万虚拟线程
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 1_000_000; i++) {
        final int id = i;
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1));
            return "task-" + id;
        });
    }
}
```

---

## 1.2 并发 vs 并行

**并发（Concurrency）**：多个任务在**同一时间段内**交替执行（单核CPU通过时间片轮转）。

**并行（Parallelism）**：多个任务在**同一时刻**真正同时执行（多核CPU）。

```
并发（单核，时间片轮转）：
时间轴：  ─────────────────────────────>
任务 A：  ████░░░░████░░░░████░░░░
任务 B：  ░░░░████░░░░████░░░░████
         (任意时刻只有一个在运行，但都在推进)

并行（多核）：
时间轴：  ─────────────────────────────>
CPU核1：  ████████████████████████  <- 任务A
CPU核2：  ████████████████████████  <- 任务B
任务 A 和 B 真正同时运行
```

**一句话总结：**
- 并发 = 一个人同时处理多件事（交替）
- 并行 = 多个人同时各处理一件事

```java
// Java 并行计算示例：ForkJoinPool
List<Integer> list = IntStream.rangeClosed(1, 1_000_000)
    .boxed().collect(Collectors.toList());

// 并行流（底层使用 ForkJoinPool.commonPool()）
long sum = list.parallelStream()
               .mapToLong(Integer::longValue)
               .sum();
System.out.println("并行计算结果: " + sum);

// 自定义并行度
ForkJoinPool customPool = new ForkJoinPool(4); // 4核并行
long result = customPool.submit(
    () -> list.parallelStream().mapToLong(Integer::longValue).sum()
).get();
```

---

## 1.3 线程的六种状态

Java 线程状态由 `Thread.State` 枚举定义，共六种：

```java
public enum State {
    NEW,           // 新建
    RUNNABLE,      // 可运行（包含运行中）
    BLOCKED,       // 阻塞（等待 synchronized 锁）
    WAITING,       // 无限等待
    TIMED_WAITING, // 限时等待
    TERMINATED     // 终止
}
```

### 状态转换图（ASCII）

```
线程状态机完整转换图：

  ┌────────┐
  │  NEW   │  <- new Thread()，线程对象已创建，尚未启动
  └────┬───┘
       │ t.start()
       v
  ┌──────────┐       ┌────────────┐
  │ RUNNABLE │ ----> │ TERMINATED │  <- run()正常结束 或 未捕获异常
  └──────────┘       └────────────┘
       │    ^
       │    │
  [等待sync锁]  [获得sync锁]
       │    │
       v    │
  ┌─────────┐
  │ BLOCKED │  <- 等待进入 synchronized 代码块/方法
  └─────────┘

  RUNNABLE -> WAITING（无限等待，需要被明确唤醒）：
    * Object.wait()
    * Thread.join()
    * LockSupport.park()

  RUNNABLE -> TIMED_WAITING（有时间限制的等待）：
    * Thread.sleep(long millis)
    * Object.wait(long timeout)
    * Thread.join(long millis)
    * LockSupport.parkNanos(long nanos)
    * LockSupport.parkUntil(long deadline)

  WAITING/TIMED_WAITING -> RUNNABLE：
    * notify() / notifyAll()
    * interrupt()
    * 超时时间到达（仅TIMED_WAITING）
    * LockSupport.unpark(thread)

完整状态流转示意：
           start()
  NEW ────────────────────────────────────────────────────> RUNNABLE
                                                               │
                                        获得synchronized锁      │ 等待synchronized锁
                                              │                 │
                                    BLOCKED <─────────────────┘
                                    
  RUNNABLE ──wait()──────────> WAITING ──notify()/unpark()──> RUNNABLE
  RUNNABLE ──sleep(t)────> TIMED_WAITING ──timeout──────────> RUNNABLE
  RUNNABLE ──run()结束──────────────────────────────────────> TERMINATED
```

### 代码验证各种状态

```java
public class ThreadStateDemo {

    public static void main(String[] args) throws InterruptedException {

        // 1. NEW 状态
        Thread t1 = new Thread(() -> {}, "t1");
        System.out.println("t1 状态（start前）: " + t1.getState()); // NEW

        // 2. RUNNABLE 状态
        Thread t2 = new Thread(() -> {
            while (true) { /* 忙等 */ }
        }, "t2");
        t2.setDaemon(true);
        t2.start();
        Thread.sleep(100);
        System.out.println("t2 状态: " + t2.getState()); // RUNNABLE

        // 3. TIMED_WAITING 状态
        Thread t3 = new Thread(() -> {
            try { Thread.sleep(10000); } catch (InterruptedException e) {}
        }, "t3");
        t3.start();
        Thread.sleep(100);
        System.out.println("t3 状态（sleep中）: " + t3.getState()); // TIMED_WAITING

        // 4. WAITING 状态
        Object lock = new Object();
        Thread t4 = new Thread(() -> {
            synchronized (lock) {
                try { lock.wait(); } catch (InterruptedException e) {}
            }
        }, "t4");
        t4.start();
        Thread.sleep(100);
        System.out.println("t4 状态（wait中）: " + t4.getState()); // WAITING

        // 5. BLOCKED 状态
        Object lock2 = new Object();
        Thread holder = new Thread(() -> {
            synchronized (lock2) {
                try { Thread.sleep(5000); } catch (InterruptedException e) {}
            }
        });
        holder.setDaemon(true);
        holder.start();
        Thread.sleep(100);

        Thread t5 = new Thread(() -> {
            synchronized (lock2) { /* 等待 lock2 */ }
        }, "t5");
        t5.start();
        Thread.sleep(100);
        System.out.println("t5 状态（等待锁）: " + t5.getState()); // BLOCKED

        // 6. TERMINATED 状态
        Thread t6 = new Thread(() -> {}, "t6");
        t6.start();
        t6.join();
        System.out.println("t6 状态（执行完）: " + t6.getState()); // TERMINATED

        // 清理
        t3.interrupt();
        t4.interrupt();
    }
}
```

---

## 1.4 线程创建方式

### 方式一：继承 Thread 类

```java
public class MyThread extends Thread {
    private String taskName;

    public MyThread(String taskName) {
        super("MyThread-" + taskName);
        this.taskName = taskName;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " 执行任务: " + taskName);
    }

    public static void main(String[] args) {
        MyThread t = new MyThread("数据处理");
        t.start(); // 注意：调用 start() 而不是 run()！
        // t.run() 只是普通方法调用，不会新建线程
    }
}
```

**缺点**：Java 是单继承，继承了 Thread 就不能继承其他类了。

### 方式二：实现 Runnable 接口

```java
public class RunnableDemo {

    static class PrintTask implements Runnable {
        private final String message;

        public PrintTask(String message) {
            this.message = message;
        }

        @Override
        public void run() {
            for (int i = 0; i < 3; i++) {
                System.out.println(Thread.currentThread().getName()
                    + ": " + message + " - " + i);
            }
        }
    }

    public static void main(String[] args) {
        // 传统写法
        Thread t1 = new Thread(new PrintTask("Hello"), "Thread-1");
        t1.start();

        // Lambda 表达式（Java 8+）
        Thread t2 = new Thread(() ->
            System.out.println(Thread.currentThread().getName() + " Lambda线程"),
            "Lambda-Thread");
        t2.start();
    }
}
```

**优点**：可以继续继承其他类；任务与线程解耦，同一 Runnable 可被多个线程执行。

### 方式三：Callable + Future（有返回值）

```java
import java.util.concurrent.*;

public class CallableFutureDemo {

    // Callable 可以有返回值，可以抛出受检异常
    static class ComputeTask implements Callable<Integer> {
        private final int n;

        public ComputeTask(int n) { this.n = n; }

        @Override
        public Integer call() throws Exception {
            System.out.println(Thread.currentThread().getName()
                + " 计算斐波那契(" + n + ")");
            return fibonacci(n);
        }

        private int fibonacci(int n) {
            if (n <= 1) return n;
            return fibonacci(n - 1) + fibonacci(n - 2);
        }
    }

    public static void main(String[] args) throws Exception {
        ExecutorService executor = Executors.newFixedThreadPool(3);

        // 提交任务，获得 Future
        Future<Integer> future1 = executor.submit(new ComputeTask(10));
        Future<Integer> future2 = executor.submit(new ComputeTask(20));

        System.out.println("任务已提交，继续做其他事...");

        // get() 阻塞直到计算完成
        Integer result1 = future1.get();
        Integer result2 = future2.get(5, TimeUnit.SECONDS); // 超时等待

        System.out.println("fib(10) = " + result1);
        System.out.println("fib(20) = " + result2);
        System.out.println("任务是否完成: " + future1.isDone());

        executor.shutdown();
    }
}
```

### 方式四：线程池（推荐）

```java
import java.util.concurrent.*;

public class ThreadPoolDemo {

    public static void main(String[] args) {
        // 推荐自定义线程池，而非 Executors 工厂方法（有 OOM 风险）
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            2,                                  // corePoolSize：核心线程数
            5,                                  // maximumPoolSize：最大线程数
            60, TimeUnit.SECONDS,               // keepAliveTime：空闲线程存活时间
            new ArrayBlockingQueue<>(100),      // workQueue：有界工作队列
            r -> {                              // threadFactory：线程工厂
                Thread t = new Thread(r);
                t.setName("MyPool-" + t.getId());
                t.setDaemon(false);
                return t;
            },
            new ThreadPoolExecutor.CallerRunsPolicy() // 拒绝策略
        );

        for (int i = 0; i < 10; i++) {
            final int taskId = i;
            executor.submit(() -> {
                System.out.println(Thread.currentThread().getName()
                    + " 执行任务-" + taskId);
                try { Thread.sleep(100); } catch (InterruptedException e) {}
            });
        }

        executor.shutdown();
    }
}
```

---

## 1.5 线程常用方法

### sleep() vs wait() 深度对比

| 对比维度 | Thread.sleep() | Object.wait() |
|---------|----------------|---------------|
| 所属类 | Thread（静态方法） | Object |
| 锁的释放 | **不释放**锁 | **释放**锁 |
| 唤醒方式 | 时间到期自动唤醒 | notify()/notifyAll() |
| 使用场所 | 任何地方 | 必须在 synchronized 块内 |
| 异常 | InterruptedException | InterruptedException |

```java
public class SleepVsWaitDemo {

    private static final Object lock = new Object();

    public static void main(String[] args) throws InterruptedException {

        // sleep 示例：持有锁睡眠，不释放锁
        Thread sleepThread = new Thread(() -> {
            synchronized (lock) {
                System.out.println("sleepThread 获得锁，开始 sleep（不释放锁）");
                try {
                    Thread.sleep(2000); // 持有锁睡眠 2 秒
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println("sleepThread sleep 结束");
            }
        });

        // wait 示例：释放锁等待
        Thread waitThread = new Thread(() -> {
            synchronized (lock) {
                System.out.println("waitThread 获得锁，开始 wait（释放锁）");
                try {
                    lock.wait(2000); // 释放锁，等待 2 秒
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
                System.out.println("waitThread wait 结束");
            }
        });

        waitThread.start();
        Thread.sleep(100);
        sleepThread.start(); // waitThread 释放了锁，sleepThread 可以进入
        sleepThread.join();
        waitThread.join();
    }
}
```

### join() - 等待线程结束

```java
public class JoinDemo {

    public static void main(String[] args) throws InterruptedException {
        Thread dataLoader = new Thread(() -> {
            System.out.println("开始加载数据...");
            try { Thread.sleep(2000); } catch (InterruptedException e) {}
            System.out.println("数据加载完成！");
        }, "DataLoader");

        Thread processor = new Thread(() -> {
            System.out.println("开始处理数据...");
            try { Thread.sleep(1000); } catch (InterruptedException e) {}
            System.out.println("数据处理完成！");
        }, "Processor");

        dataLoader.start();
        dataLoader.join(); // 等待 dataLoader 完成才继续

        processor.start(); // dataLoader 完成后才启动 processor
        processor.join();

        System.out.println("所有任务完成！");
    }
}
```

### interrupt() - 中断线程

`interrupt()` 不会强制停止线程，而是设置线程的**中断标志位**，线程需要自行检测并响应中断。

```java
public class InterruptDemo {

    public static void main(String[] args) throws InterruptedException {

        // 示例 1：通过 isInterrupted() 检测中断（无阻塞操作时）
        Thread t1 = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                // 正常工作...
            }
            System.out.println("t1 收到中断信号，退出");
        });
        t1.start();
        Thread.sleep(100);
        t1.interrupt(); // 设置中断标志

        // 示例 2：处理 InterruptedException（有阻塞操作时）
        Thread t2 = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    System.out.println("t2 工作中...");
                    Thread.sleep(500); // 阻塞期间收到中断 -> 抛出 InterruptedException
                } catch (InterruptedException e) {
                    // 重要！捕获 InterruptedException 后中断标志会被自动清除
                    // 必须重新设置，否则 while 条件检测不到中断
                    Thread.currentThread().interrupt();
                    System.out.println("t2 收到中断，退出循环");
                    break;
                }
            }
        });
        t2.start();
        Thread.sleep(1200);
        t2.interrupt();

        t1.join();
        t2.join();
    }
}
```

### yield() - 让出 CPU

```java
public class YieldDemo {
    public static void main(String[] args) {
        Runnable task = () -> {
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName() + " - " + i);
                Thread.yield(); // 提示调度器让出 CPU，但不保证一定让
            }
        };

        new Thread(task, "A").start();
        new Thread(task, "B").start();
    }
}
```

**注意**：`yield()` 只是一个**提示**，JVM 不保证一定会让出 CPU，且不释放锁。

---

## 1.6 守护线程（Daemon Thread）

守护线程是为其他线程提供服务的后台线程。当所有**用户线程（非守护线程）**结束时，JVM 会退出，守护线程也随之强制结束。

典型守护线程：GC 线程、JVM 内部线程。

```java
public class DaemonThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        Thread daemon = new Thread(() -> {
            int count = 0;
            while (true) {
                System.out.println("守护线程运行中... isDaemon="
                    + Thread.currentThread().isDaemon() + " count=" + count++);
                try { Thread.sleep(500); } catch (InterruptedException e) { break; }
            }
            // 这行可能不会被打印，因为 JVM 退出时守护线程被强制终止
            System.out.println("守护线程正常退出");
        }, "DaemonThread");

        // 必须在 start() 之前设置！
        daemon.setDaemon(true);
        daemon.start();

        // 主线程（用户线程）执行 2 秒后结束
        Thread.sleep(2000);
        System.out.println("主线程结束，守护线程也会随之结束");
        // JVM 退出，守护线程被强制终止
    }
}
```

**注意事项：**
1. `setDaemon(true)` 必须在 `start()` 之前调用，否则抛出 `IllegalThreadStateException`
2. 守护线程不应持有需要清理的资源（数据库连接、文件句柄等），因为可能被强制终止
3. 线程池中配置守护线程需通过 `ThreadFactory`

---
# Part 2: Java 内存模型（JMM）

## 2.1 主内存 vs 工作内存

JMM（Java Memory Model）是 Java 定义的**内存访问规范**，屏蔽不同 CPU 和操作系统的内存访问差异。

```
JMM 抽象内存模型：

  ┌──────────────────────────────────────────────────────────────────────┐
  │                           主内存（Main Memory）                       │
  │                                                                      │
  │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
  │   │  共享变量x   │  │  共享变量y   │  │  共享变量z   │               │
  │   │    x = 1    │  │    y = 2    │  │    z = 3    │               │
  │   └─────────────┘  └─────────────┘  └─────────────┘               │
  └───────────────────────────────┬──────────────────────────────────────┘
           ^v read/write           ^v read/write
  ┌─────────────────────┐    ┌─────────────────────┐
  │  线程1 工作内存       │    │  线程2 工作内存       │
  │                     │    │                     │
  │  ┌──────┐ ┌──────┐  │    │  ┌──────┐ ┌──────┐  │
  │  │变量x  │ │变量y  │  │    │  │变量x  │ │变量z  │  │
  │  │副本1  │ │副本2  │  │    │  │副本1' │ │副本3  │  │
  │  └──────┘ └──────┘  │    │  └──────┘ └──────┘  │
  │      CPU1           │    │      CPU2           │
  └─────────────────────┘    └─────────────────────┘

JMM 8个内存交互操作：
  lock（锁定）    -> 把主内存变量标记为一条线程独占
  unlock（解锁）  -> 释放锁定的变量
  read（读取）    -> 从主内存读取变量值到工作内存
  load（载入）    -> 把 read 得到的值放到工作内存变量副本中
  use（使用）     -> 把工作内存变量值传递给执行引擎
  assign（赋值）  -> 把执行引擎的值赋给工作内存变量
  store（存储）   -> 把工作内存变量值传递到主内存
  write（写入）   -> 把 store 得到的值写入主内存变量
```

**对应硬件层面（多级缓存结构）：**
```
  CPU 核心1               CPU 核心2
  ┌──────────────┐        ┌──────────────┐
  │  寄存器       │        │  寄存器       │
  │  L1 Cache    │        │  L1 Cache    │  <- 每核独享，约 32KB，~4周期
  │  L2 Cache    │        │  L2 Cache    │  <- 每核独享，约 256KB，~12周期
  └──────┬───────┘        └──────┬───────┘
         │                       │
  ┌──────┴───────────────────────┴───────┐
  │          L3 Cache（共享）             │  <- 多核共享，约 8-32MB，~40周期
  └──────────────────┬───────────────────┘
                     │
  ┌──────────────────┴───────────────────┐
  │           主内存（RAM）               │  <- GB 级，~200周期
  └──────────────────────────────────────┘
```

---

## 2.2 三大并发问题

### 可见性（Visibility）

一个线程修改了共享变量，另一个线程不能立即看到最新值（值被缓存在 CPU 缓存/寄存器中）。

```java
public class VisibilityProblem {
    // 没有 volatile，可能出现可见性问题
    private static boolean flag = false;

    public static void main(String[] args) throws InterruptedException {
        Thread t = new Thread(() -> {
            // 可能无限循环！flag 被 JIT 缓存到寄存器，看不到主线程的修改
            while (!flag) { }
            System.out.println("线程看到 flag = true，退出");
        });
        t.start();

        Thread.sleep(1000);
        flag = true; // 主线程修改，但子线程可能看不到！
        System.out.println("主线程设置 flag = true");
    }
}
```

### 原子性（Atomicity）

操作不可分割，不会被其他线程中途打断。

```java
public class AtomicityProblem {
    private static int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[100];
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    count++; // 非原子操作！字节码层面拆分为4步：
                    // getstatic  (读取 count)
                    // iconst_1   (压入常量1)
                    // iadd       (相加)
                    // putstatic  (写回 count)
                    // 这4步之间可能被其他线程打断！
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("期望: 100000, 实际: " + count); // 通常小于 100000
    }
}
```

### 有序性（Ordering）

程序执行顺序可能被编译器或 CPU 重排序，单线程下不影响结果，多线程下可能导致问题。

```java
public class OrderingProblem {
    private static int x = 0, y = 0;
    private static int a = 0, b = 0;

    public static void main(String[] args) throws InterruptedException {
        int count = 0;
        while (true) {
            x = 0; y = 0; a = 0; b = 0;
            count++;

            Thread t1 = new Thread(() -> {
                a = 1; // 写 a
                x = b; // 读 b
            });
            Thread t2 = new Thread(() -> {
                b = 1; // 写 b
                y = a; // 读 a
            });

            t1.start(); t2.start();
            t1.join(); t2.join();

            // 正常执行：不可能出现 x==0 && y==0
            // 但指令重排序后（t1: x=b先于a=1，t2: y=a先于b=1），可能出现
            if (x == 0 && y == 0) {
                System.out.println("第 " + count + " 次出现重排序！x=" + x + ", y=" + y);
                break;
            }
        }
    }
}
```

---

## 2.3 happens-before 原则（8条规则）

`happens-before` 是 JMM 定义的**可见性保证**规则：如果操作 A happens-before 操作 B，则 A 的结果对 B 可见，且 A 的执行顺序在 B 之前。

**重要**：happens-before 不等于时间上先发生，它是对内存可见性的保证。

### 规则1：程序顺序规则（Program Order Rule）

同一线程内，按照代码顺序，前面的操作 happens-before 后面的操作。

```java
// 在同一线程内
int a = 1;           // 操作1
int b = a + 1;       // 操作2：操作1 happens-before 操作2
// b 能看到 a 的最新值（b == 2 保证）
```

### 规则2：监视器锁规则（Monitor Lock Rule）

一个 `unlock` 操作 happens-before 后续对同一锁的 `lock` 操作。

```java
synchronized (lock) {
    x = 10; // 解锁前的写操作
} // unlock

// 另一个线程
synchronized (lock) { // lock（在上面 unlock 之后）
    System.out.println(x); // 能看到 x = 10（happens-before 保证）
}
```

### 规则3：volatile 变量规则（Volatile Variable Rule）

对 volatile 变量的写操作 happens-before 后续对该变量的读操作。

```java
volatile int v = 0;

// 线程1
v = 1; // 写 volatile

// 线程2（在线程1写完之后）
int r = v; // 读到 v = 1（happens-before 保证）
```

### 规则4：线程启动规则（Thread Start Rule）

`Thread.start()` happens-before 该线程的每一个动作。

```java
int x = 0;
Thread t = new Thread(() -> {
    // start() 之前的所有写对线程内可见
    System.out.println(x); // 能看到 x = 10
});
x = 10;
t.start(); // start() happens-before 线程内所有操作
```

### 规则5：线程终止规则（Thread Termination Rule）

线程中的所有操作 happens-before 对该线程的 `join()` 返回。

```java
int[] result = {0};
Thread t = new Thread(() -> {
    result[0] = 42; // 线程内写操作
});
t.start();
t.join(); // join() 返回后，result[0] 的最新值对主线程可见
System.out.println(result[0]); // 能看到 42
```

### 规则6：线程中断规则（Thread Interruption Rule）

`interrupt()` 调用 happens-before 被中断线程检测到中断（`isInterrupted()` 返回 true 或抛出 `InterruptedException`）。

### 规则7：对象终结规则（Finalizer Rule）

对象的构造函数执行结束 happens-before `finalize()` 方法的开始。

### 规则8：传递性规则（Transitivity Rule）

如果 A happens-before B，B happens-before C，则 A happens-before C。

```
规则传递性示例（volatile + 传递性）：
volatile int v;
int x;

线程1:  x = 1 --(程序顺序规则)--> v = 2
线程2:  r1 = v --(volatile规则，读到v=2，所以v的写hb读)--> r2 = x

通过传递性：x = 1 happens-before r2 = x
因此 r2 能读到 x = 1（即使 x 不是 volatile！）
```

---

## 2.4 指令重排序

### 三种重排序类型

```
源代码 -> [编译器重排序] -> [指令级并行重排序] -> [内存系统重排序] -> 最终执行
           JIT 编译器       CPU流水线优化          缓存/写缓冲区
           
As-if-serial 语义：
  不管怎么重排序，单线程程序的执行结果不能改变。
  编译器和 CPU 只会对没有数据依赖关系的操作进行重排序。

数据依赖关系（不会被重排序）：
  写后读：a = 1; b = a;    // 不能重排
  写后写：a = 1; a = 2;    // 不能重排
  读后写：a = b; b = 1;    // 不能重排

可以被重排序的情况：
  int a = 1;  // 操作1
  int b = 2;  // 操作2（与操作1无数据依赖）
  // 1和2可以互换顺序，单线程结果不变
```

### 内存屏障（Memory Barrier/Fence）

JMM 通过内存屏障来禁止特定类型的重排序：

```
四种内存屏障：
┌─────────────────┬──────────────────────────────────────────────────────┐
│   屏障类型       │                   作用                                │
├─────────────────┼──────────────────────────────────────────────────────┤
│ LoadLoad        │ Load1; [LoadLoad]; Load2                             │
│                 │ 确保 Load1 数据在 Load2 载入之前完成（读-读屏障）       │
├─────────────────┼──────────────────────────────────────────────────────┤
│ StoreStore      │ Store1; [StoreStore]; Store2                         │
│                 │ 确保 Store1 刷新到主内存，Store2 才能开始（写-写屏障）  │
├─────────────────┼──────────────────────────────────────────────────────┤
│ LoadStore       │ Load1; [LoadStore]; Store2                           │
│                 │ 确保 Load1 在 Store2 刷新到主内存之前完成（读-写屏障）  │
├─────────────────┼──────────────────────────────────────────────────────┤
│ StoreLoad       │ Store1; [StoreLoad]; Load2                           │
│                 │ 确保 Store1 对所有处理器可见后 Load2 才开始（最强屏障） │
│                 │ 开销最大，几乎等同于全内存屏障                          │
└─────────────────┴──────────────────────────────────────────────────────┘

x86 处理器特点：
  x86 是 TSO（Total Store Ordering）内存模型，天然保证 StoreLoad 以外的顺序，
  所以 Java volatile 在 x86 上只需要 StoreLoad 屏障（mfence 指令）。
```

---

## 2.5 volatile 关键字深度解析

### volatile 的两大作用

**1. 保证可见性**：通过 **MESI 缓存一致性协议**实现。

```
MESI 协议状态：
M (Modified)  - 缓存行已修改，与主内存不一致，只有本核有效
E (Exclusive) - 缓存行仅本核有，与主内存一致（未被其他核缓存）
S (Shared)    - 缓存行多核共享，与主内存一致（可被多核读取）
I (Invalid)   - 缓存行无效（被其他核修改，或已失效）

volatile 写操作时序：
┌────────────────────────────────────────────────────────────────────────┐
│  CPU1 修改 volatile 变量 v = 1                                         │
│    -> CPU1 发出 "BusLock" 或 "MESI Invalidate" 消息到总线               │
│    -> 其他 CPU（CPU2, CPU3...）收到消息                                  │
│    -> 其他 CPU 将自己缓存中对应缓存行标记为 I（Invalid）                   │
│    -> CPU1 将 v=1 写入缓存（标记为 M），再刷新到主内存                     │
│                                                                        │
│  CPU2 读取 volatile 变量 v                                             │
│    -> CPU2 检查自己的缓存，发现是 I（Invalid）状态                        │
│    -> 从主内存（或 CPU1 的缓存）加载最新值 v=1                            │
│    -> 缓存行变为 S（Shared）状态                                         │
└────────────────────────────────────────────────────────────────────────┘
```

**2. 保证有序性**：通过**内存屏障**禁止重排序。

volatile 变量的内存屏障插入规则（JSR-133 规范）：

```
volatile 写操作（以 v = x 为例）：
  [StoreStore 屏障]   <- 禁止上面的普通写与本 volatile 写重排序
  volatile write: v = x
  [StoreLoad 屏障]    <- 禁止本 volatile 写与下面的 volatile 读重排序

volatile 读操作（以 x = v 为例）：
  volatile read: x = v
  [LoadLoad 屏障]     <- 禁止本 volatile 读与下面的普通读重排序
  [LoadStore 屏障]    <- 禁止本 volatile 读与下面的普通写重排序
```

**3. volatile 不保证原子性（重要！）**

```java
public class VolatileAtomicityProblem {
    private static volatile int count = 0;

    public static void main(String[] args) throws InterruptedException {
        Thread[] threads = new Thread[20];
        for (int i = 0; i < 20; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    count++; // volatile 不保证 ++ 的原子性！
                    // count++ 等价于：
                    //   1. 从主内存读取 count（volatile read）
                    //   2. count + 1（计算）
                    //   3. 写回主内存（volatile write）
                    // 步骤1和3之间可能有其他线程同时操作！
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("期望: 20000, 实际: " + count); // 可能小于 20000
    }
}
```

**volatile 适用场景：**
- 状态标志（boolean flag）
- 单次写多次读（double-checked locking 中的引用）
- 独立观察量

**volatile 不适用场景：**
- 需要原子性的复合操作（使用 AtomicXxx 或 synchronized）

### DCL 双重检查锁单例模式

```java
public class Singleton {
    // 必须用 volatile！
    private static volatile Singleton instance;

    private Singleton() {
        // 初始化...
    }

    public static Singleton getInstance() {
        if (instance == null) {              // 第一次检查（无锁，性能优化）
            synchronized (Singleton.class) {  // 加锁
                if (instance == null) {       // 第二次检查（防止重复创建）
                    instance = new Singleton();
                    // 上面这行等价于：
                    // 1. memory = allocate()    分配内存
                    // 2. init(memory)           初始化对象
                    // 3. instance = memory      引用赋值
                    // 步骤2和3可能被重排序！-> 必须用 volatile 禁止
                }
            }
        }
        return instance;
    }
}
```

**为什么必须用 volatile？**

```
被重排序后的问题（步骤3先于步骤2）：

时间线：
  线程A 进入 synchronized，执行 instance = new Singleton()
    步骤1: 分配内存
    步骤3: instance = memory  <- instance 不为 null，但对象未初始化！
    （切换到线程B）
  
  线程B 执行第一次检查 if(instance == null)
    -> instance != null（步骤3已完成）
    -> 直接返回 instance
    -> 线程B 使用未初始化的对象 -> 报错！

  （切换回线程A）
    步骤2: 初始化对象（已经晚了）

volatile 通过内存屏障禁止步骤2和步骤3的重排序，解决此问题。
```

**更好的单例实现（静态内部类，无需 volatile）：**

```java
public class SingletonByHolder {
    private SingletonByHolder() {}

    // 利用类加载机制保证线程安全和延迟初始化
    // Holder 类在被引用时才会被加载，JVM 保证类初始化的原子性
    private static class Holder {
        static final SingletonByHolder INSTANCE = new SingletonByHolder();
    }

    public static SingletonByHolder getInstance() {
        return Holder.INSTANCE;
    }
}
```

---
# Part 3: synchronized 深度解析

## 3.1 synchronized 三种用法

```java
public class SynchronizedUsageDemo {

    private int count = 0;

    // 用法1：修饰实例方法（锁 = this 实例对象）
    public synchronized void incrementInstance() {
        count++;
        // 等价于：
        // synchronized(this) { count++; }
    }

    // 用法2：修饰静态方法（锁 = Xxx.class 类对象，全局唯一）
    public static synchronized void staticMethod() {
        System.out.println("静态方法同步，锁是 SynchronizedUsageDemo.class");
        // 所有实例共用同一把锁（类对象），粒度比实例方法更大
    }

    // 用法3：同步代码块（锁 = 指定对象，最灵活）
    private final Object lock = new Object();
    public void incrementBlock() {
        // 只同步必要的代码段，减少锁的持有时间
        doBeforeLock(); // 不需要同步的代码在锁外执行
        synchronized (lock) {
            count++; // 只有这里需要同步
        }
        doAfterLock(); // 不需要同步的代码在锁外执行
    }

    private void doBeforeLock() { /* 耗时预处理 */ }
    private void doAfterLock() { /* 后续处理 */ }

    public static void main(String[] args) throws InterruptedException {
        SynchronizedUsageDemo demo = new SynchronizedUsageDemo();

        // 10个线程并发自增
        Thread[] threads = new Thread[10];
        for (int i = 0; i < 10; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    demo.incrementInstance();
                }
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("最终计数: " + demo.count); // 应该是 10000
    }
}
```

---

## 3.2 对象头（Object Header）结构

Java 对象在堆中的内存布局（64位JVM，默认开启压缩指针 -XX:+UseCompressedOops）：

```
Java 对象内存布局（64位 JVM，开启压缩指针）：

  ┌──────────────────────────────────────────────────────────────────┐
  │                        对象头（Object Header）                    │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │                  Mark Word（8 字节，64位）                 │   │
  │  │  存储锁状态、GC年龄、HashCode（根据状态动态变化）           │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │    Klass Pointer（类指针，压缩后4字节，未压缩8字节）        │   │
  │  │  指向方法区中的 Class 元数据                               │   │
  │  └──────────────────────────────────────────────────────────┘   │
  │  ┌──────────────────────────────────────────────────────────┐   │
  │  │  数组长度（仅数组对象有，4字节）                            │   │
  │  └──────────────────────────────────────────────────────────┘   │
  ├──────────────────────────────────────────────────────────────────┤
  │                     实例数据（Instance Data）                     │
  │  按字段声明顺序（有对齐优化），boolean/byte=1B,int=4B,long=8B...  │
  ├──────────────────────────────────────────────────────────────────┤
  │                       对齐填充（Padding）                         │
  │  使对象总大小为 8 字节的整数倍（Hotspot VM 要求）                  │
  └──────────────────────────────────────────────────────────────────┘

典型对象大小示例：
  new Object()   = 8字节(MarkWord) + 4字节(Klass) + 4字节(Padding) = 16字节
  new int[0]     = 8+4+4(length)+4(Padding) = 16字节
  new String()   = 对象头12B + 引用char[]4B + int hash4B + Padding = 24字节
```

**Mark Word 详细结构（64位）：**

```
Mark Word（64 bit）在不同状态下的布局：

无锁状态（unlocked，biasable=0）：
 63                              33 32       8 7   5 4    2 1 0
┌──────────────────────────────────┬──────────┬──────┬────┬──┐
│         hashCode（31bit）         │ 未使用25b │  GC  │age │01│
└──────────────────────────────────┴──────────┴──────┴────┴──┘

偏向锁（biased lock，bit[2]=1）：
 63                          10 9  8 7    5 4    2 1 0
┌────────────────────────────┬─────┬──────┬────┬─┬──┐
│       Thread ID（54bit）    │Epoch│未使用1│age │1│01│
└────────────────────────────┴─────┴──────┴────┴─┴──┘
注：JDK 15 废弃，JDK 21 移除偏向锁

轻量级锁（thin lock）：
 63                                                    2 1 0
┌──────────────────────────────────────────────────────┬──┐
│            指向栈帧中 Lock Record 的指针（62bit）       │00│
└──────────────────────────────────────────────────────┴──┘

重量级锁（fat lock，膨胀锁）：
 63                                                    2 1 0
┌──────────────────────────────────────────────────────┬──┐
│          指向 ObjectMonitor 对象的指针（62bit）         │10│
└──────────────────────────────────────────────────────┴──┘

GC 标记（正在被 GC 标记或转移）：
 63                                                    2 1 0
┌──────────────────────────────────────────────────────┬──┐
│                      空（62bit）                       │11│
└──────────────────────────────────────────────────────┴──┘

锁标志位（bits[1:0]）含义：
  00 = 轻量级锁
  01 = 无锁或偏向锁（bit[2]区分：0无锁，1偏向）
  10 = 重量级锁
  11 = GC 标记
```

---

## 3.3 锁升级过程

### 偏向锁（Biased Locking）

**适用场景**：锁大多数情况只被**同一个线程**反复获取（如单线程操作 HashMap）。

**原理**：将线程ID存储在对象头的 Mark Word 中，下次同一线程加锁时，只需检查线程ID是否匹配，无需任何 CAS 操作，效率极高（接近无锁）。

```
偏向锁获取/撤销流程：

  线程T1 第一次获取锁：
  ┌─────────────────────────────────────────────────────────┐
  │  1. 检查 Mark Word 的偏向模式位（bit[2]）                 │
  │  2. 若为 0（未偏向）：                                    │
  │     CAS(Mark Word, [hashcode|0|01], [ThreadID|Epoch|1|01])│
  │     成功：偏向 T1，T1 持有偏向锁                           │
  │  3. 若已偏向 T1：直接获得锁（零 CAS！最高性能）             │
  └─────────────────────────────────────────────────────────┘

  线程T2 竞争偏向锁（已偏向 T1）：
  ┌─────────────────────────────────────────────────────────┐
  │  1. T2 发现锁已偏向 T1                                    │
  │  2. 等待 JVM 全局安全点（Stop the World，所有线程暂停）     │
  │  3. 检查 T1 是否还在使用此锁：                            │
  │     - T1 不再持有：撤销偏向，Mark Word 恢复无锁状态        │
  │     - T1 仍持有：升级为轻量级锁                           │
  └─────────────────────────────────────────────────────────┘

偏向锁的代价：
  - 撤销偏向锁需要等待安全点（Stop-the-World），有一定开销
  - 如果竞争频繁，偏向锁反而会降低性能
  - JVM 参数：-XX:+UseBiasedLocking（默认开启，JDK 15+ 废弃）
  - 可以通过 -XX:BiasedLockingStartupDelay=0 关闭启动延迟（默认4秒）
  - JDK 21 已彻底移除偏向锁
```

### 轻量级锁（Lightweight Lock）

**适用场景**：多线程**交替**访问（竞争窗口短，不存在真正的并发），无真正的竞争等待。

**原理**：通过 CAS 将 Mark Word 复制到线程栈帧中的 Lock Record，避免操作系统级别的互斥量（mutex），减少上下文切换。

```
轻量级锁获取流程：

  ┌─────────────────────────────────────────────────────────────────┐
  │  1. 在当前线程的栈帧中创建 Lock Record 区域                      │
  │     Lock Record = { Displaced Mark Word (复制对象头) | owner指针 }│
  │                                                                 │
  │  2. 尝试 CAS 操作：                                              │
  │     compareAndSwap(object.markWord,                             │
  │                    当前Mark Word（无锁状态），                    │
  │                    指向 Lock Record 的指针)                      │
  │                                                                 │
  │  3. CAS 成功：获得轻量级锁                                       │
  │     Mark Word = [Lock Record 指针 | 00]                         │
  │                                                                 │
  │  4. CAS 失败：                                                   │
  │     - 检查 Mark Word 是否指向本线程的栈帧（重入）                  │
  │       -> 是：在 Lock Record 中压入 null（重入计数）               │
  │       -> 否：有竞争，升级为重量级锁                               │
  └─────────────────────────────────────────────────────────────────┘

  自旋策略（JDK 6+ 自适应自旋）：
    升级重量级锁前，先进行有限次自旋（默认约10次）
    JVM 根据历史数据动态调整自旋次数：
    - 上次自旋成功：增加自旋次数
    - 上次自旋失败：减少自旋次数或直接放弃
```

### 重量级锁（Heavyweight Lock）

**适用场景**：多线程真正并发竞争，有线程需要等待较长时间。需要操作系统 mutex/futex 支持，涉及内核态切换。

**核心数据结构：ObjectMonitor**（C++ 实现，位于 JVM 源码 objectMonitor.cpp）：

```
ObjectMonitor 结构详解：

┌─────────────────────────────────────────────────────────────────┐
│                       ObjectMonitor                             │
│                                                                 │
│  _header      -> 原始 Mark Word（解锁后恢复）                    │
│  _owner       -> 当前持有锁的线程指针（null表示未被持有）          │
│  _recursions  -> 锁的重入次数（支持可重入）                       │
│  _count       -> 线程等待 monitor 的总数                         │
│  _waiters     -> 调用 wait() 的线程数量                          │
│                                                                 │
│  _WaitSet（条件等待集合）：                                        │
│  ┌──────┐   ┌──────┐   ┌──────┐                               │
│  │ T3   │ <-> T4   │ <-> T5   │   <- 调用 wait() 后进入        │
│  └──────┘   └──────┘   └──────┘      等待 notify()/超时         │
│                                                                 │
│  _EntryList（入口等待队列，有序）：                                │
│  ┌──────┐   ┌──────┐   ┌──────┐                               │
│  │ T6   │ <-> T7   │ <-> T8   │   <- 等待获取锁的线程           │
│  └──────┘   └──────┘   └──────┘      被唤醒后从这里竞争          │
│                                                                 │
│  _cxq（竞争队列，无锁 LIFO 队列）：                               │
│  ┌──────┐                                                       │
│  │ T9   │   <- 最近竞争失败的线程，CAS 入队                      │
│  └──────┘                                                       │
└─────────────────────────────────────────────────────────────────┘

wait/notify 详细流程：

1. 线程T1 持有锁，调用 obj.wait()：
   -> 将 T1 的 ObjectWaiter 节点加入 _WaitSet
   -> _owner = null（释放锁）
   -> T1 进入 WAITING 状态（park）
   -> 其他线程竞争锁

2. 线程T2 持有锁，调用 obj.notify()：
   -> 从 _WaitSet 取出一个节点（策略：FIFO 或 LIFO，取决于实现）
   -> 将该节点移到 _EntryList（或直接让其竞争）
   -> T1 被唤醒，重新竞争锁（注意：notify不立即交出锁！）

3. 线程T2 持有锁，调用 obj.notifyAll()：
   -> 将 _WaitSet 中所有节点移到 _EntryList
   -> 所有等待线程唤醒，重新竞争锁
```

### 完整锁升级流程图

```
                         new Object()
                              v
                    ┌──────────────────┐
                    │    无锁状态       │  MarkWord: [hashcode | 0 | 01]
                    │  （可偏向状态）    │
                    └────────┬─────────┘
                             │
                    第一个线程首次获取锁
                             │
                             v
                    ┌──────────────────┐
                    │    偏向锁         │  MarkWord: [ThreadID|Epoch|1|01]
                    │  （JDK 21已移除） │  同一线程重入：零CAS，极高性能
                    └────────┬─────────┘
                             │
                    其他线程竞争（等待安全点，撤销偏向）
                             │
                             v
                    ┌──────────────────┐
                    │   轻量级锁        │  MarkWord: [Lock Record指针|00]
                    │  （CAS自旋）      │  线程交替获取：CAS操作，无内核切换
                    └────────┬─────────┘
                             │
                    CAS 失败次数超过自旋阈值
                    或有第三个线程加入竞争
                             │
                             v
                    ┌──────────────────┐
                    │   重量级锁        │  MarkWord: [ObjectMonitor指针|10]
                    │  （操作系统互斥）  │  多线程并发：线程阻塞，内核切换
                    └──────────────────┘

锁只能升级，不能降级（特殊情况：GC 时可以撤销偏向锁）
JDK 版本说明：
  JDK 6：引入偏向锁和轻量级锁，大幅优化 synchronized
  JDK 8：自适应自旋，锁消除，锁粗化
  JDK 15：偏向锁标记废弃（JEP 374）
  JDK 21：彻底移除偏向锁（JEP 374 最终实现）
```

---

## 3.4 锁消除与锁粗化

### 锁消除（Lock Elimination）

JIT 编译器通过**逃逸分析**，对不可能存在共享竞争的锁进行消除。

```java
public class LockEliminationDemo {

    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 1_000_000; i++) {
            createString();
        }
        System.out.println("耗时: " + (System.currentTimeMillis() - start) + "ms");
    }

    public static String createString() {
        // StringBuffer 内部方法是 synchronized 的
        // 但 sb 是方法局部变量，逃逸分析确认它不会逃逸到其他线程
        // JIT 分析后：这个 synchronized 锁永远不会被竞争，直接消除！
        StringBuffer sb = new StringBuffer();
        sb.append("Hello");  // synchronized - 锁消除
        sb.append(" ");      // synchronized - 锁消除
        sb.append("World"); // synchronized - 锁消除
        return sb.toString();
        // 等价于无锁的 StringBuilder
    }

    // 逃逸分析参数：
    // -XX:+DoEscapeAnalysis（默认开启）
    // -XX:+EliminateLocks（默认开启）
    // -XX:+PrintEscapeAnalysis（打印逃逸分析结果）
}
```

**逃逸分析识别规则：**
- 方法逃逸：对象作为方法返回值，或传递给其他方法
- 线程逃逸：对象被赋值给成员变量，可以被其他线程访问

### 锁粗化（Lock Coarsening）

将多个连续的加锁/解锁操作合并为一次，减少加解锁的开销。

```java
public class LockCoarseningDemo {

    private final Object lock = new Object();
    private int count;

    // 优化前：多次加锁解锁（JIT 会自动粗化）
    public void addBefore() {
        synchronized (lock) { count++; }  // lock
        synchronized (lock) { count++; }  // lock
        synchronized (lock) { count++; }  // lock
    }

    // JIT 优化后（锁粗化）：等价于
    public void addAfterOptimized() {
        synchronized (lock) {
            count++;
            count++;
            count++;
        }
    }

    // 注意：for 循环内的锁通常不会被粗化（循环体过大时粗化代价更高）
    public void loopLock() {
        for (int i = 0; i < 100; i++) {
            synchronized (lock) {
                count++; // 每次循环加解锁，设计时应优化为锁外层
            }
        }
    }

    // 正确写法：将锁移到循环外
    public void loopLockOptimized() {
        synchronized (lock) {
            for (int i = 0; i < 100; i++) {
                count++;
            }
        }
    }
}
```

---

## 3.5 synchronized vs volatile 对比

| 对比维度 | synchronized | volatile |
|---------|-------------|----------|
| 保证原子性 | 是（整个同步块） | 否（单个变量读/写，不保证复合操作） |
| 保证可见性 | 是 | 是 |
| 保证有序性 | 是（禁止临界区内重排序） | 是（禁止volatile变量相关重排序） |
| 锁的粒度 | 对象/类 | 变量 |
| 阻塞特性 | 会阻塞（等待锁） | 不会阻塞 |
| 适用场景 | 复合操作/临界区/wait-notify | 状态标志/DCL单例/简单赋值 |
| 性能开销 | 较重（锁竞争时有上下文切换） | 轻量（仅内存屏障） |
| 使用复杂度 | 需注意死锁/异常处理 | 简单 |
| 编译器优化 | 会禁止临界区内某些优化 | 禁止相关重排序 |

**选择建议：**
- 需要原子性（如 count++）-> `synchronized` 或 `AtomicInteger`
- 仅需可见性（如状态标志）-> `volatile`
- 高并发，需要细粒度控制 -> `ReentrantLock`（Part 5）

---
# Part 4: AQS（AbstractQueuedSynchronizer）深度解析

## 4.1 AQS 是什么

AQS（`AbstractQueuedSynchronizer`）是 JUC 的**基石框架**，位于 `java.util.concurrent.locks` 包。Doug Lea 设计，是 JUC 中最核心的类。

JUC 中基于 AQS 实现的组件：

```
AQS 生态全景图：

                    ┌────────────────────────────────────────────────────┐
                    │        AbstractQueuedSynchronizer (AQS)            │
                    │   - volatile int state（同步状态）                  │
                    │   - CLH变体双向队列（等待线程队列）                   │
                    │   - 模板方法：acquire/release/acquireShared 等       │
                    └──────────────────┬─────────────────────────────────┘
                                       │ 继承
        ┌──────────────────────────────┼──────────────────────────────────┐
        │                              │                                  │
  ┌─────┴──────┐               ┌───────┴──────┐                  ┌───────┴──────┐
  │ReentrantLock│               │  Semaphore   │                  │CountDownLatch│
  │ Sync/Fair  │               │    Sync      │                  │    Sync      │
  │ NonfairSync│               │ (共享锁实现)  │                  │ (共享锁实现)  │
  │ (独占锁)   │               └──────────────┘                  └──────────────┘
  └─────┬──────┘
        │
  ┌─────┴──────────────────────┐
  │  ReentrantReadWriteLock    │
  │  ReadLock / WriteLock      │
  │  (读:共享锁 写:独占锁)      │
  └────────────────────────────┘
  
  其他基于 AQS 的工具：
  CyclicBarrier (内部使用 ReentrantLock + Condition)
  ThreadPoolExecutor.Worker (实现 AbstractQueuedSynchronizer)
```

---

## 4.2 AQS 核心数据结构

```
AQS 核心结构详解：

  ┌─────────────────────────────────────────────────────────────────┐
  │                           AQS                                   │
  │                                                                 │
  │  private volatile int state;  // 核心同步状态                    │
  │  ┌──────────────────────────────────────────────────────────┐  │
  │  │  独占模式（ReentrantLock）：                                │  │
  │  │    state=0: 锁空闲，可以获取                               │  │
  │  │    state=1: 锁被占用（第一次获取）                          │  │
  │  │    state=N: 同一线程重入了 N 次（可重入锁）                  │  │
  │  │  共享模式（Semaphore）：                                    │  │
  │  │    state=N: 还有 N 个许可可以获取                           │  │
  │  │    state=0: 没有许可，需等待                               │  │
  │  │  CountDownLatch：                                          │  │
  │  │    state=N: 还需要 N 次 countDown                          │  │
  │  │    state=0: 所有 await 线程可以继续                         │  │
  │  └──────────────────────────────────────────────────────────┘  │
  │                                                                 │
  │  CLH 变体双向队列（FIFO）：                                       │
  │                                                                 │
  │  head ──> ┌──────────────┐    ┌──────────────┐    ┌──────────┐ │
  │           │ 虚拟头节点    │<──>│  Node(T1)    │<──>│ Node(T2) │ <── tail
  │           │ waitStatus=0 │    │ SIGNAL(-1)   │    │ SIGNAL   │ │
  │           │ thread=null  │    │ thread=T1    │    │ thread=T2│ │
  │           └──────────────┘    └──────────────┘    └──────────┘ │
  │                                                                 │
  │  exclusiveOwnerThread: 当前持有独占锁的线程（继承自AOS）           │
  └─────────────────────────────────────────────────────────────────┘

Node 节点结构（AQS 内部类）：
  ┌────────────────────────────────────────────────────────────────┐
  │                         Node                                   │
  │                                                                │
  │  volatile int waitStatus           // 等待状态                  │
  │    CANCELLED =  1  -> 线程已取消等待（超时/中断），不再参与竞争   │
  │    SIGNAL    = -1  -> 后继节点需要被 unpark 唤醒               │
  │    CONDITION = -2  -> 节点在条件队列（ConditionObject）中        │
  │    PROPAGATE = -3  -> 共享模式下：唤醒需要向后传播               │
  │    0               -> 初始状态                                 │
  │                                                                │
  │  volatile Node prev    // 前驱节点（插入时设置，不会为null）      │
  │  volatile Node next    // 后继节点（可能为null，需从tail向前找）  │
  │  volatile Thread thread // 等待的线程（获取锁后设为null）        │
  │  Node nextWaiter        // 条件队列中的下一个节点（或共享标记）   │
  └────────────────────────────────────────────────────────────────┘
```

---

## 4.3 独占锁获取（acquire）流程详解

```java
// AQS acquire 源码分析（JDK 8，精简注释版）
public abstract class AbstractQueuedSynchronizer {

    // ===== 核心：独占锁获取入口 =====
    public final void acquire(int arg) {
        if (!tryAcquire(arg) &&  // 1. 首先尝试直接获取锁（子类实现）
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))  // 2. 失败则入队等待
            selfInterrupt();  // 3. 如果等待期间被中断，重新设置中断标志
    }

    // ===== 步骤2a：将当前线程封装为 Node 加入队列尾部 =====
    private Node addWaiter(Node mode) {
        Node node = new Node(Thread.currentThread(), mode);
        Node pred = tail;
        if (pred != null) {
            node.prev = pred;
            // 快速尝试 CAS 入队（大多数情况下成功）
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
        enq(node);  // CAS 失败或队列为空，自旋入队（初始化队列）
        return node;
    }

    // ===== 自旋入队（初始化虚拟头节点） =====
    private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
            if (t == null) { // 队列为空，初始化虚拟头节点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
    }

    // ===== 步骤2b：在队列中等待获取锁（核心循环） =====
    final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {  // 自旋循环
                final Node p = node.predecessor();
                // 关键：只有前驱是头节点时，才尝试获取锁
                // 原因：保证 FIFO 公平性，头节点的后继才有权竞争
                if (p == head && tryAcquire(arg)) {
                    setHead(node);   // 获取成功，成为新头节点
                    p.next = null;   // 帮助 GC（断开引用）
                    failed = false;
                    return interrupted;
                }
                // 判断是否应该阻塞当前线程
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())  // LockSupport.park 阻塞
                    interrupted = true;       // 记录中断状态
            }
        } finally {
            if (failed)
                cancelAcquire(node);  // 发生异常时，取消该节点
        }
    }

    // ===== 判断是否应该 park（阻塞）当前线程 =====
    private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
        int ws = pred.waitStatus;
        if (ws == Node.SIGNAL)
            // 前驱节点的 waitStatus 已是 SIGNAL，意味着它会在释放锁时 unpark 我
            // 现在可以安全地 park（睡眠）
            return true;
        if (ws > 0) {
            // 前驱节点已取消（CANCELLED=1），跳过所有已取消的节点
            do {
                node.prev = pred = pred.prev;
            } while (pred.waitStatus > 0);
            pred.next = node;
        } else {
            // 将前驱节点的 waitStatus 设为 SIGNAL，表示"下次释放时记得唤醒我"
            compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
        }
        return false;  // 先不 park，再循环一次尝试获取锁（避免遗漏唤醒）
    }

    // ===== 阻塞当前线程，并返回是否被中断 =====
    private final boolean parkAndCheckInterrupt() {
        LockSupport.park(this);  // 阻塞，直到被 unpark 或中断唤醒
        return Thread.interrupted();  // 检查并清除中断标志
    }
}
```

**acquire 完整流程图：**

```
acquire(1) 完整执行流程：

  开始
    │
    v
  tryAcquire(1)  ──成功──> 直接获得锁，返回
    │ 失败
    v
  addWaiter(EXCLUSIVE)
  把当前线程包装成 EXCLUSIVE Node，CAS 加入队尾
    │
    v
  acquireQueued(node, 1)  开始自旋：
  ┌─────────────────────────────────────────────────────┐
  │ loop:                                               │
  │   pred = node.prev                                  │
  │   if pred == head:                                  │
  │     tryAcquire(1) 成功? ──是──> setHead(node)       │
  │                                   返回 interrupted  │
  │                    └─否──> 继续往下                  │
  │   shouldPark?(pred, node):                          │
  │     pred.waitStatus == SIGNAL? ──是──> return true  │
  │                                 └─否──> 设置 SIGNAL  │
  │                                         return false │
  │   if shouldPark == true:                            │
  │     LockSupport.park(this)  <──── 线程在此阻塞       │
  │     ← 被 LockSupport.unpark() 唤醒（release时）      │
  │   重新进入循环                                       │
  └─────────────────────────────────────────────────────┘
```

---

## 4.4 独占锁释放（release）流程

```java
// AQS release 源码
public final boolean release(int arg) {
    if (tryRelease(arg)) {  // 子类实现（ReentrantLock：state 递减到0）
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);  // 唤醒后继节点
        return true;
    }
    return false;
}

private void unparkSuccessor(Node node) {
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);  // 清除头节点的 SIGNAL 标志

    Node s = node.next;
    // 如果后继节点为 null 或已取消（CANCELLED=1）
    // 从尾节点向前遍历找到最前面的有效节点（非取消状态）
    // 注意：必须从 tail 向前找，不能从 head 向后找（因为 addWaiter 中 next 的设置不是原子的）
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;  // 找到距离 head 最近的有效节点
    }
    if (s != null)
        LockSupport.unpark(s.thread);  // 唤醒该线程
}
```

---

## 4.5 共享锁获取（acquireShared）

```java
// 共享锁（如 Semaphore.acquire(), CountDownLatch.await()）
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)  // < 0 表示获取失败
        doAcquireShared(arg);
}

// tryAcquireShared 返回值语义：
//   < 0: 获取失败
//   = 0: 获取成功，但后续节点不能再获取（资源耗尽）
//   > 0: 获取成功，且剩余资源足够，后续节点也可尝试获取（传播唤醒）

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);  // 加入共享模式队列
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    // 获取成功，设置头节点，并传播唤醒（关键！共享模式特有）
                    setHeadAndPropagate(node, r);
                    p.next = null;
                    if (interrupted) selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed) cancelAcquire(node);
    }
}

// 共享模式的关键：setHeadAndPropagate（传播唤醒）
private void setHeadAndPropagate(Node node, int propagate) {
    Node h = head;
    setHead(node);
    // 如果还有剩余资源，或者头节点状态需要传播
    // 唤醒下一个等待节点（如果它也是共享模式的话继续传播）
    if (propagate > 0 || h == null || h.waitStatus < 0 ||
        (h = head) == null || h.waitStatus < 0) {
        Node s = node.next;
        if (s == null || s.isShared())
            doReleaseShared();  // 唤醒后续节点
    }
}
```

---

## 4.6 条件队列（ConditionObject）

```java
// ConditionObject 源码分析
public class ConditionObject implements Condition {

    // 条件队列（单向链表，与 AQS 同步队列分离）
    private transient Node firstWaiter;
    private transient Node lastWaiter;

    // await() 实现：释放锁 -> 加入条件队列 -> 阻塞等待
    public final void await() throws InterruptedException {
        if (Thread.interrupted())
            throw new InterruptedException();
        Node node = addConditionWaiter();  // 加入条件队列
        int savedState = fullyRelease(node);  // 完全释放锁（包括重入）
        int interruptMode = 0;
        while (!isOnSyncQueue(node)) {  // 还在条件队列中？
            LockSupport.park(this);     // 阻塞
            if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
                break;
        }
        // 被 signal 后，节点已从条件队列移到同步队列
        // 重新竞争锁
        if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
            interruptMode = REINTERRUPT;
        // 清理条件队列中已取消的节点
        if (node.nextWaiter != null)
            unlinkCancelledWaiters();
        if (interruptMode != 0)
            reportInterruptAfterWait(interruptMode);
    }

    // signal() 实现：将条件队列中第一个节点移到同步队列
    public final void signal() {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        Node first = firstWaiter;
        if (first != null)
            doSignal(first);
    }

    private void doSignal(Node first) {
        do {
            if ((firstWaiter = first.nextWaiter) == null)
                lastWaiter = null;
            first.nextWaiter = null;
        } while (!transferForSignal(first) &&  // 将节点移到同步队列
                 (first = firstWaiter) != null);
    }
}
```

```
Condition 队列与同步队列的关系图：

AQS 同步队列（CLH 变体，双向）：
  head <-> [虚节点] <-> [Node T1, SIGNAL] <-> [Node T2, SIGNAL] -> tail

条件队列A（单向链表）：
  ConditionA.firstWaiter -> [Node T3, CONDITION] -> [Node T4, CONDITION] -> null

条件队列B（单向链表）：
  ConditionB.firstWaiter -> [Node T5, CONDITION] -> null

流程说明：
  1. T3 调用 condA.await()
     -> T3 释放锁（state=0）
     -> T3 从同步队列移到条件队列A
     -> T3 阻塞

  2. T1 持有锁，调用 condA.signal()
     -> T3 从条件队列A 移到同步队列尾部
     -> T3 的 waitStatus 变为 SIGNAL
     -> T3 等待重新竞争锁（T1 释放锁后）

  3. T1 释放锁
     -> unparkSuccessor(head)
     -> T3（或T2，按队列顺序）被唤醒，获得锁
```

### Condition 使用示例（有界阻塞队列）

```java
import java.util.concurrent.locks.*;

public class BoundedQueue<T> {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();  // 队列不满条件
    private final Condition notEmpty = lock.newCondition();  // 队列不空条件
    private final Object[] items;
    private int count, putIndex, takeIndex;

    public BoundedQueue(int capacity) {
        items = new Object[capacity];
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length)
                notFull.await();  // 等待"不满"条件
            items[putIndex] = item;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            notEmpty.signal();  // 通知"不空"条件（精确唤醒，比 notifyAll 高效）
        } finally {
            lock.unlock();
        }
    }

    @SuppressWarnings("unchecked")
    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0)
                notEmpty.await();  // 等待"不空"条件
            T item = (T) items[takeIndex];
            items[takeIndex] = null;
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            notFull.signal();  // 通知"不满"条件
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try { return count; }
        finally { lock.unlock(); }
    }
}
```

---

## 4.7 AQS 模板方法设计模式

```
AQS 使用经典的模板方法设计模式：

父类（AQS）定义通用流程（final，不可重写）：
  acquire(int)              -> 独占获取（含排队/挂起）
  release(int)              -> 独占释放（含唤醒）
  acquireShared(int)        -> 共享获取
  releaseShared(int)        -> 共享释放
  acquireInterruptibly(int) -> 可中断的独占获取
  tryAcquireNanos(int,long) -> 带超时的独占获取

子类只需实现"钩子方法"（具体策略）：
  tryAcquire(int)       -> 独占：尝试获取，成功返回true
  tryRelease(int)       -> 独占：尝试释放，完全释放返回true
  tryAcquireShared(int) -> 共享：尝试获取，>=0表示成功
  tryReleaseShared(int) -> 共享：尝试释放，需要唤醒后续节点返回true
  isHeldExclusively()   -> 当前线程是否独占持有锁（Condition 需要）

设计优势：
  1. 代码复用：排队/唤醒/中断处理等通用代码由 AQS 统一实现
  2. 可扩展性：子类只关注"能否获取/释放资源"的逻辑
  3. 一致性：所有基于 AQS 的锁/同步器行为一致（中断、超时处理）
```

**自定义二元信号量（基于 AQS）：**

```java
import java.util.concurrent.locks.AbstractQueuedSynchronizer;
import java.util.concurrent.TimeUnit;

/**
 * 二元信号量：只有0和1两个状态，类似互斥锁但不可重入
 */
public class BinarySemaphore {

    private static final class Sync extends AbstractQueuedSynchronizer {

        // 初始 state=1（可获取），state=0（已被获取）
        Sync() { setState(1); }

        @Override
        protected int tryAcquireShared(int acquires) {
            // 共享获取：CAS 将 state 从 1 改为 0
            return (compareAndSetState(1, 0)) ? 1 : -1;
        }

        @Override
        protected boolean tryReleaseShared(int releases) {
            // 共享释放：将 state 从 0 改为 1
            return compareAndSetState(0, 1);
        }
    }

    private final Sync sync = new Sync();

    public void acquire() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    public void release() {
        sync.releaseShared(1);
    }

    public boolean isAvailable() {
        return sync.getState() == 1;
    }

    public static void main(String[] args) throws InterruptedException {
        BinarySemaphore sem = new BinarySemaphore();

        // 线程1获取
        new Thread(() -> {
            try {
                sem.acquire();
                System.out.println("T1 获取信号量");
                Thread.sleep(2000);
                sem.release();
                System.out.println("T1 释放信号量");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();

        Thread.sleep(100);

        // 线程2等待获取
        new Thread(() -> {
            try {
                System.out.println("T2 等待信号量...");
                sem.acquire();
                System.out.println("T2 获取信号量");
                sem.release();
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }
}
```

---
# Part 5: ReentrantLock 深度解析

## 5.1 ReentrantLock vs synchronized 全面对比

| 对比维度 | ReentrantLock | synchronized |
|---------|---------------|-------------|
| 实现层次 | Java API（JUC，纯 Java） | JVM 内置（字节码 monitorenter/exit） |
| 锁的获取 | 显式 lock()/unlock()，必须 finally 释放 | 自动加锁/解锁 |
| 可重入 | 支持，state 计数 | 支持，Monitor 计数 |
| 公平锁 | 支持（new ReentrantLock(true)） | 不支持 |
| 可中断 | lockInterruptibly() | 不可中断等待 |
| 超时等待 | tryLock(time, unit) | 不支持 |
| 多条件变量 | newCondition()（多个） | 只有 wait/notify |
| 锁状态查询 | isLocked(), getQueueLength() | 无 API |

## 5.2 公平锁 vs 非公平锁源码对比

```java
public class ReentrantLock implements Lock {

    abstract static class Sync extends AbstractQueuedSynchronizer {
        final boolean nonfairTryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 非公平锁：直接 CAS 抢锁，不管队列中是否有等待线程
                if (compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                // 可重入：同一线程，state 累加
                int nextc = c + acquires;
                if (nextc < 0) throw new Error("Maximum lock count exceeded");
                setState(nextc);
                return true;
            }
            return false;
        }
    }

    // 非公平锁
    static final class NonfairSync extends Sync {
        final void lock() {
            if (compareAndSetState(0, 1))  // 先直接抢锁（不排队）
                setExclusiveOwnerThread(Thread.currentThread());
            else
                acquire(1);
        }
        @Override
        protected final boolean tryAcquire(int acquires) {
            return nonfairTryAcquire(acquires);
        }
    }

    // 公平锁
    static final class FairSync extends Sync {
        final void lock() {
            acquire(1);  // 直接走 AQS 排队流程
        }
        @Override
        protected final boolean tryAcquire(int acquires) {
            final Thread current = Thread.currentThread();
            int c = getState();
            if (c == 0) {
                // 关键区别：先检查队列中是否有等待线程！
                if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                    setExclusiveOwnerThread(current);
                    return true;
                }
            } else if (current == getExclusiveOwnerThread()) {
                setState(c + acquires);
                return true;
            }
            return false;
        }
    }
}
```

## 5.3 可重入原理（state 计数）

```java
public class ReentrantDemo {
    private final ReentrantLock lock = new ReentrantLock();

    public void outer() {
        lock.lock();  // state: 0 -> 1
        System.out.println("outer: state=" + lock.getHoldCount());  // 1
        try {
            inner();
        } finally {
            lock.unlock();  // state: 2 -> 1
        }
    } // lock.unlock() 后 state: 1 -> 0，真正释放

    public void inner() {
        lock.lock();  // state: 1 -> 2（同一线程，直接累加，不阻塞）
        System.out.println("inner: state=" + lock.getHoldCount());  // 2
        try { /* 临界区 */ }
        finally { lock.unlock(); }
    }

    public static void main(String[] args) {
        new ReentrantDemo().outer();
    }
}
```

## 5.4 可中断锁和超时锁

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class AdvancedLockDemo {
    private final ReentrantLock lock = new ReentrantLock();

    // 可中断锁：等待期间可响应中断
    public void interruptibleTask() throws InterruptedException {
        lock.lockInterruptibly();  // 等待时收到中断，立即抛 InterruptedException
        try {
            System.out.println(Thread.currentThread().getName() + " 获得锁");
            Thread.sleep(3000);
        } finally {
            lock.unlock();
        }
    }

    // 超时锁：等待指定时间后放弃
    public boolean timedTask() throws InterruptedException {
        if (lock.tryLock(2, TimeUnit.SECONDS)) {
            try {
                System.out.println("获得锁，执行业务");
                return true;
            } finally {
                lock.unlock();
            }
        }
        System.out.println("等待2秒未获得锁，放弃");
        return false;
    }

    public static void main(String[] args) throws InterruptedException {
        AdvancedLockDemo demo = new AdvancedLockDemo();

        Thread t1 = new Thread(() -> {
            try { demo.interruptibleTask(); }
            catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName() + " 被中断放弃等待");
            }
        }, "T1");

        Thread t2 = new Thread(() -> {
            try { demo.interruptibleTask(); }
            catch (InterruptedException e) {
                System.out.println(Thread.currentThread().getName() + " 被中断放弃等待");
            }
        }, "T2");

        t1.start();
        Thread.sleep(200);
        t2.start();
        Thread.sleep(500);
        t2.interrupt();  // 中断 T2
        t1.join();
    }
}
```

## 5.5 完整案例：生产者消费者

```java
import java.util.LinkedList;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class ProducerConsumerWithLock {
    private final Queue<Integer> queue = new LinkedList<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull  = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public ProducerConsumerWithLock(int capacity) {
        this.capacity = capacity;
    }

    public void produce(int item) throws InterruptedException {
        lock.lock();
        try {
            // 使用 while 防止虚假唤醒（spurious wakeup）
            while (queue.size() == capacity)
                notFull.await();
            queue.offer(item);
            System.out.printf("[%s] 生产: %d，队列大小: %d%n",
                Thread.currentThread().getName(), item, queue.size());
            notEmpty.signalAll();
        } finally { lock.unlock(); }
    }

    public int consume() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty())
                notEmpty.await();
            int item = queue.poll();
            System.out.printf("[%s] 消费: %d，队列大小: %d%n",
                Thread.currentThread().getName(), item, queue.size());
            notFull.signalAll();
            return item;
        } finally { lock.unlock(); }
    }

    public static void main(String[] args) {
        ProducerConsumerWithLock buf = new ProducerConsumerWithLock(5);

        for (int i = 0; i < 3; i++) {
            final int pid = i;
            new Thread(() -> {
                for (int j = 0; j < 5; j++) {
                    try {
                        buf.produce(pid * 10 + j);
                        Thread.sleep(100);
                    } catch (InterruptedException e) { return; }
                }
            }, "Producer-" + i).start();
        }

        for (int i = 0; i < 2; i++) {
            new Thread(() -> {
                for (int j = 0; j < 8; j++) {
                    try {
                        buf.consume();
                        Thread.sleep(150);
                    } catch (InterruptedException e) { return; }
                }
            }, "Consumer-" + i).start();
        }
    }
}
```

---
# Part 6: 读写锁

## 6.1 ReentrantReadWriteLock

读写锁核心：**读读不互斥，读写互斥，写写互斥**。读多写少场景大幅提升并发性能。

```java
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.locks.ReentrantReadWriteLock;

public class ReadWriteCache<K, V> {
    private final Map<K, V> cache = new HashMap<>();
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final ReentrantReadWriteLock.ReadLock  readLock  = rwLock.readLock();
    private final ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();

    // 读操作：多线程可并发（读锁共享）
    public V get(K key) {
        readLock.lock();
        try {
            return cache.get(key);
        } finally { readLock.unlock(); }
    }

    // 写操作：独占（写锁排他）
    public void put(K key, V value) {
        writeLock.lock();
        try {
            cache.put(key, value);
        } finally { writeLock.unlock(); }
    }

    public V remove(K key) {
        writeLock.lock();
        try { return cache.remove(key); }
        finally { writeLock.unlock(); }
    }
}
```

## 6.2 state 高低16位拆分原理

```
ReentrantReadWriteLock 用单个 int state 同时记录读锁和写锁：

  ┌──────────────────────────────────────────────────────────────────────┐
  │                     state（32位 int）                                 │
  │    高16位（bits 31~16）         低16位（bits 15~0）                    │
  │  ┌──────────────────────┬──────────────────────┐                    │
  │  │  读锁共享持有数        │  写锁重入次数          │                    │
  │  └──────────────────────┴──────────────────────┘                    │
  │                                                                      │
  │  读锁计数 = state >>> 16                                             │
  │  写锁计数 = state & 0xFFFF     (低16位掩码)                           │
  │                                                                      │
  │  示例：state = 0x00030002                                            │
  │    -> 读锁：3个线程持有，写锁：重入2次                                  │
  │  state = 0x00050000 -> 5个读线程，无写操作                             │
  │  state = 0x00000001 -> 无读，写锁被持有1次                            │
  └──────────────────────────────────────────────────────────────────────┘
```

## 6.3 锁降级（写锁 -> 读锁）

```java
public class LockDegradationDemo {
    private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final ReentrantReadWriteLock.ReadLock  readLock  = rwLock.readLock();
    private final ReentrantReadWriteLock.WriteLock writeLock = rwLock.writeLock();
    private volatile boolean cacheValid = false;
    private String cachedData;

    public String getOrLoad() {
        readLock.lock();
        if (cacheValid) {
            try { return cachedData; }
            finally { readLock.unlock(); }
        }
        readLock.unlock();

        writeLock.lock();
        try {
            if (!cacheValid) {
                cachedData = "DB_DATA";  // 从数据库加载
                cacheValid = true;
            }
            // 锁降级：持有写锁时获取读锁
            readLock.lock();  // <-- 此时同时持有写锁和读锁
        } finally {
            writeLock.unlock();  // 释放写锁（降级完成，只剩读锁）
        }

        try {
            return cachedData;  // 持有读锁，安全读取
        } finally {
            readLock.unlock();
        }
    }
}
// 注意：不支持锁升级（读 -> 写），可能导致死锁！
```

## 6.4 StampedLock（Java 8，支持乐观读）

```java
import java.util.concurrent.locks.StampedLock;

public class StampedLockDemo {
    private final StampedLock sl = new StampedLock();
    private double x, y;

    // 写锁
    public void move(double dx, double dy) {
        long stamp = sl.writeLock();
        try { x += dx; y += dy; }
        finally { sl.unlockWrite(stamp); }
    }

    // 乐观读（无锁读取，验证后安全返回）
    public double distanceFromOrigin() {
        long stamp = sl.tryOptimisticRead();  // 获取版本戳，无锁
        double curX = x, curY = y;           // 读取数据（可能不一致）

        if (!sl.validate(stamp)) {  // 验证：期间有写操作？
            // 升级为悲观读锁
            stamp = sl.readLock();
            try { curX = x; curY = y; }
            finally { sl.unlockRead(stamp); }
        }
        return Math.sqrt(curX * curX + curY * curY);
    }

    // 读锁升级为写锁（StampedLock 支持，ReadWriteLock 不支持！）
    public void conditionalUpdate() {
        long stamp = sl.readLock();
        try {
            while (x == 0) {
                long writeStamp = sl.tryConvertToWriteLock(stamp);
                if (writeStamp != 0) {
                    stamp = writeStamp;
                    x = 1.0;
                    break;
                } else {
                    sl.unlockRead(stamp);
                    stamp = sl.writeLock();
                }
            }
        } finally { sl.unlock(stamp); }
    }
}
```

**StampedLock vs ReentrantReadWriteLock：**

| 特性 | StampedLock | ReentrantReadWriteLock |
|------|------------|----------------------|
| 乐观读 | 支持（最高性能） | 不支持 |
| 可重入 | 不支持 | 支持 |
| 条件变量 | 不支持 | 支持 |
| 读锁升写锁 | 支持 | 不支持 |
| 中断处理 | 自旋等待时中断可能 CPU 飙升 | 安全 |

---

# Part 7: 并发工具类

## 7.1 CountDownLatch（倒计时门闩）

```
原理：AQS state = N，每次 countDown() state-1，state=0 时唤醒所有等待线程
特点：一次性，不可重置
```

```java
import java.util.concurrent.*;

public class CountDownLatchDemo {

    // 案例：并发请求聚合
    public static void main(String[] args) throws InterruptedException {
        int serviceCount = 5;
        CountDownLatch latch = new CountDownLatch(serviceCount);
        ExecutorService executor = Executors.newFixedThreadPool(serviceCount);
        String[] results = new String[serviceCount];

        for (int i = 0; i < serviceCount; i++) {
            final int id = i;
            executor.submit(() -> {
                try {
                    Thread.sleep((long)(Math.random() * 2000));
                    results[id] = "Service-" + id + " OK";
                    System.out.println("服务" + id + " 完成");
                } catch (InterruptedException e) {
                    results[id] = "Service-" + id + " ERROR";
                } finally {
                    latch.countDown();  // 无论成功失败都必须 countDown
                }
            });
        }

        boolean done = latch.await(5, TimeUnit.SECONDS);
        System.out.println("全部完成: " + done);
        for (String r : results) System.out.println("  " + r);
        executor.shutdown();
    }
}
```

## 7.2 CyclicBarrier（循环屏障）

```java
import java.util.concurrent.*;

public class CyclicBarrierDemo {

    // 案例：分阶段并行计算（可循环使用）
    public static void main(String[] args) throws InterruptedException {
        int N = 4;
        CyclicBarrier barrier = new CyclicBarrier(N, () ->
            System.out.println("=== 所有线程到达屏障，执行阶段汇总 ==="));

        for (int i = 0; i < N; i++) {
            final int tid = i;
            new Thread(() -> {
                for (int phase = 0; phase < 3; phase++) {  // 3个阶段
                    try {
                        Thread.sleep((long)(Math.random() * 500));
                        System.out.printf("线程%d 第%d阶段完成%n", tid, phase + 1);
                        barrier.await();  // 等待所有线程完成本阶段
                        // barrier 自动重置，可以继续下一阶段
                    } catch (Exception e) {
                        Thread.currentThread().interrupt();
                    }
                }
            }).start();
        }
    }
}
```

**CountDownLatch vs CyclicBarrier：**

| 维度 | CountDownLatch | CyclicBarrier |
|------|----------------|---------------|
| 可重用 | 否 | 是（自动重置） |
| 等待逻辑 | 1组等N个完成 | N个互相等待 |
| 到达后 | 无额外动作 | 可执行 Runnable |
| 基于 | AQS 共享锁 | ReentrantLock + Condition |

## 7.3 Semaphore（信号量）

```java
import java.util.concurrent.*;

public class SemaphoreDemo {

    // 案例：连接池限流
    static class ConnectionPool {
        private final Semaphore semaphore;
        private final String[] conns;
        private final boolean[] used;

        ConnectionPool(int size) {
            semaphore = new Semaphore(size, true);  // 公平
            conns = new String[size];
            used  = new boolean[size];
            for (int i = 0; i < size; i++) conns[i] = "Conn-" + i;
        }

        public String acquire() throws InterruptedException {
            semaphore.acquire();  // 无许可则阻塞
            return checkout();
        }

        public void release(String conn) {
            checkin(conn);
            semaphore.release();
        }

        private synchronized String checkout() {
            for (int i = 0; i < conns.length; i++) {
                if (!used[i]) { used[i] = true; return conns[i]; }
            }
            throw new IllegalStateException();
        }
        private synchronized void checkin(String c) {
            for (int i = 0; i < conns.length; i++) {
                if (c.equals(conns[i])) { used[i] = false; return; }
            }
        }
    }

    public static void main(String[] args) {
        ConnectionPool pool = new ConnectionPool(3);  // 3个连接
        ExecutorService exec = Executors.newFixedThreadPool(10);

        // 10个线程竞争3个连接
        for (int i = 0; i < 10; i++) {
            final int id = i;
            exec.submit(() -> {
                String conn = null;
                try {
                    conn = pool.acquire();
                    System.out.printf("线程%d 获得 %s%n", id, conn);
                    Thread.sleep(300);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    if (conn != null) {
                        pool.release(conn);
                        System.out.printf("线程%d 释放 %s%n", id, conn);
                    }
                }
            });
        }
        exec.shutdown();
    }
}
```

## 7.4 Exchanger（线程间交换数据）

```java
import java.util.concurrent.Exchanger;

public class ExchangerDemo {
    public static void main(String[] args) {
        Exchanger<String> exchanger = new Exchanger<>();

        // 线程A
        new Thread(() -> {
            try {
                String data = "线程A的数据";
                System.out.println("A 发送: " + data);
                String received = exchanger.exchange(data);  // 阻塞，等对方到来
                System.out.println("A 收到: " + received);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "ThreadA").start();

        // 线程B
        new Thread(() -> {
            try {
                Thread.sleep(1000);
                String data = "线程B的数据";
                System.out.println("B 发送: " + data);
                String received = exchanger.exchange(data);  // 双方同时交换
                System.out.println("B 收到: " + received);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }, "ThreadB").start();
    }
}
```

## 7.5 Phaser（多阶段同步，Java 7）

```java
import java.util.concurrent.Phaser;

public class PhaserDemo {
    public static void main(String[] args) {
        Phaser phaser = new Phaser(3) {
            @Override
            protected boolean onAdvance(int phase, int parties) {
                System.out.printf("=== 阶段%d完成，注册者:%d ===%n", phase, parties);
                return parties == 0;
            }
        };

        // 3个线程，3个阶段
        for (int i = 0; i < 3; i++) {
            final int id = i;
            new Thread(() -> {
                for (int phase = 0; phase < 3; phase++) {
                    System.out.printf("线程%d 阶段%d 执行%n", id, phase);
                    try { Thread.sleep((long)(Math.random() * 500)); }
                    catch (InterruptedException e) {}

                    if (phase < 2) {
                        phaser.arriveAndAwaitAdvance();  // 等待所有线程
                    } else {
                        phaser.arriveAndDeregister();    // 最后阶段注销
                    }
                }
                System.out.println("线程" + id + " 完成");
            }).start();
        }
    }
}
```

---
# Part 8: 原子类（Atomic）

## 8.1 CAS 原理

CAS（Compare-And-Swap）是**无锁**原子操作，由 CPU 硬件指令（x86: `cmpxchg`）保证原子性。

```
CAS 语义：
  compareAndSwap(address, expected, newValue):
    if (memory[address] == expected):
        memory[address] = newValue
        return true   // 成功
    else:
        return false  // 失败，需重试

Java CAS（通过 Unsafe 类）：
  unsafe.compareAndSwapInt(Object o, long offset, int expected, int x)

AtomicInteger.getAndAddInt 源码（JDK 8）：
  public final int getAndAddInt(Object o, long offset, int delta) {
      int v;
      do {
          v = getIntVolatile(o, offset);  // volatile 读
      } while (!compareAndSwapInt(o, offset, v, v + delta));  // CAS 循环
      return v;
  }

CAS 的三大问题：
  1. ABA 问题：值从 A->B->A，CAS 检测不到（用 AtomicStampedReference 解决）
  2. 循环时间长开销大：高并发下自旋长时间，CPU 消耗大（用 LongAdder 解决）
  3. 只能保证单个变量的原子操作（多变量用 synchronized 或 AtomicReference）
```

```java
import java.util.concurrent.atomic.*;

public class AtomicDemo {
    public static void main(String[] args) throws InterruptedException {
        AtomicInteger ai = new AtomicInteger(0);

        // 基本操作
        System.out.println(ai.get());              // 0
        System.out.println(ai.getAndSet(10));      // 0（返回旧值，设置新值）
        System.out.println(ai.getAndIncrement());  // 10（返回旧值，自增）
        System.out.println(ai.incrementAndGet());  // 12（自增，返回新值）
        System.out.println(ai.addAndGet(5));       // 17（加5，返回新值）

        // CAS
        System.out.println(ai.compareAndSet(17, 100));  // true
        System.out.println(ai.get());                   // 100

        // Java 9+ 新增方法
        System.out.println(ai.getAndUpdate(x -> x * 2));   // 100，然后变200
        System.out.println(ai.updateAndGet(x -> x + 50));  // 250

        // 多线程验证原子性
        AtomicInteger counter = new AtomicInteger(0);
        Thread[] threads = new Thread[100];
        for (int i = 0; i < 100; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < 1000; j++) counter.incrementAndGet();
            });
            threads[i].start();
        }
        for (Thread t : threads) t.join();
        System.out.println("期望100000，实际: " + counter.get());  // 100000
    }
}
```

## 8.2 ABA 问题与 AtomicStampedReference

```
ABA 问题示意：
  T1 读取值 A（stamp=1）
  T2 将 A->B（stamp=2），再将 B->A（stamp=3）
  T1 执行 CAS(expected=A, new=C)：值是 A，成功！
     但 T1 没有意识到值已经被修改过两次

危害（无锁链表场景）：
  初始栈：A -> B -> C
  T1 读取 head=A
  T2: pop A, pop B, push A（A被重用，A.next指向null或其他）
  T1: CAS(head, A, B) 成功！但 B 已经是垃圾节点
```

```java
import java.util.concurrent.atomic.*;

public class ABADemo {
    public static void main(String[] args) throws InterruptedException {

        // 有 ABA 问题的 AtomicReference
        AtomicReference<String> ref = new AtomicReference<>("A");

        // 解决方案：AtomicStampedReference（版本戳）
        AtomicStampedReference<String> stampedRef =
            new AtomicStampedReference<>("A", 1);  // 初始值+版本号

        Thread t1 = new Thread(() -> {
            int[] stampHolder = new int[1];
            String value = stampedRef.get(stampHolder);
            int stamp = stampHolder[0];
            System.out.println("T1 读取: value=" + value + ", stamp=" + stamp);

            try { Thread.sleep(1000); } catch (InterruptedException e) {}

            // T2 已执行了 ABA，版本号变为 3，T1 的 CAS 会失败
            boolean success = stampedRef.compareAndSet(
                value, "C",      // 期望值和新值
                stamp, stamp + 1 // 期望版本号和新版本号
            );
            System.out.println("T1 CAS: " + success
                + "，当前版本: " + stampedRef.getStamp());
        }, "T1");

        Thread t2 = new Thread(() -> {
            try { Thread.sleep(200); } catch (InterruptedException e) {}
            // A -> B
            int s = stampedRef.getStamp();
            stampedRef.compareAndSet("A", "B", s, s + 1);
            System.out.println("T2 A->B, stamp=" + stampedRef.getStamp());
            // B -> A（ABA）
            s = stampedRef.getStamp();
            stampedRef.compareAndSet("B", "A", s, s + 1);
            System.out.println("T2 B->A, stamp=" + stampedRef.getStamp());
        }, "T2");

        t1.start(); t2.start();
        t1.join(); t2.join();
        // T1 的 CAS 失败，因为 stamp 从1变成了3
    }
}
```

**AtomicMarkableReference**（仅用一位标记，不关心修改次数）：

```java
// 仅标记是否被修改过（不需要精确次数时使用）
AtomicMarkableReference<String> markable =
    new AtomicMarkableReference<>("value", false);

// 标记为 true（已删除标记）
markable.compareAndSet("value", "value", false, true);

// 获取值和标记
boolean[] markHolder = new boolean[1];
String val = markable.get(markHolder);
System.out.println("val=" + val + ", marked=" + markHolder[0]);
```

## 8.3 AtomicReference 与字段更新器

```java
import java.util.concurrent.atomic.*;

public class AtomicReferenceDemo {

    // 原子引用：对象整体 CAS 替换
    static class User {
        final String name;
        final int age;
        User(String name, int age) { this.name = name; this.age = age; }
        @Override public String toString() { return name + "(" + age + ")"; }
    }

    public static void main(String[] args) throws InterruptedException {
        AtomicReference<User> userRef = new AtomicReference<>(new User("Alice", 25));

        // CAS 替换整个对象
        User old = userRef.get();
        User newUser = new User("Bob", 30);
        boolean success = userRef.compareAndSet(old, newUser);
        System.out.println("替换成功: " + success + "，当前: " + userRef.get());

        // AtomicIntegerFieldUpdater：原子更新对象的 volatile 字段
        // 无需将字段包装成 AtomicInteger，节省内存
        AtomicIntegerFieldUpdater<User> ageUpdater =
            AtomicIntegerFieldUpdater.newUpdater(User.class, "age");
        // 注意：User.age 必须声明为 volatile int（不能是 final）

        // AtomicLongFieldUpdater, AtomicReferenceFieldUpdater 同理
    }
}
```

## 8.4 LongAdder vs AtomicLong 性能对比

```
LongAdder 原理（Striped64 设计）：

  低并发：直接 CAS 更新 base
  ┌─────────────────────────────┐
  │   base = 100                │
  └─────────────────────────────┘

  高并发（CAS 失败，自动创建 Cell 数组）：
  ┌────────┬────────┬────────┬────────┐
  │ Cell[0]│ Cell[1]│ Cell[2]│ Cell[3]│
  │  +300  │  +250  │  +200  │  +150  │
  └────────┴────────┴────────┴────────┘
  
  sum() = base + Cell[0] + Cell[1] + Cell[2] + Cell[3]
        = 0   + 300     + 250     + 200     + 150  = 900
  
  线程通过 ThreadLocalRandom.probe（哈希值）映射到 Cell
  Cell 数组大小 <= CPU 核心数（2的幂次）
  每个 Cell 有 @Contended 注解（填充缓存行，避免伪共享）
  
  优势：高并发下减少 CAS 竞争
  劣势：sum() 不是实时精确值（计算时各 Cell 可能仍在被修改）
```

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class LongAdderVsAtomicLong {
    public static void main(String[] args) throws InterruptedException {
        int threads = 50, loopCount = 100_000;

        // AtomicLong 测试
        AtomicLong atomicLong = new AtomicLong();
        long t1 = benchmark(threads, loopCount,
            () -> atomicLong.incrementAndGet());
        System.out.printf("AtomicLong:  %dms, result=%d%n",
            t1, atomicLong.get());

        // LongAdder 测试
        LongAdder longAdder = new LongAdder();
        long t2 = benchmark(threads, loopCount,
            () -> longAdder.increment());
        System.out.printf("LongAdder:   %dms, result=%d%n",
            t2, longAdder.sum());

        System.out.printf("LongAdder 快 %.1fx%n", (double)t1 / t2);
        // 高并发下 LongAdder 通常快 3~10 倍
    }

    static long benchmark(int threads, int loops, Runnable op)
            throws InterruptedException {
        Thread[] ts = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            ts[i] = new Thread(() -> {
                for (int j = 0; j < loops; j++) op.run();
            });
        }
        long start = System.currentTimeMillis();
        for (Thread t : ts) t.start();
        for (Thread t : ts) t.join();
        return System.currentTimeMillis() - start;
    }
}
```

**选型建议：**

| 场景 | 推荐 |
|------|------|
| 低并发，需实时精确值 | AtomicLong |
| 高并发计数/统计 | LongAdder |
| 需要 CAS（compareAndSet） | AtomicLong（LongAdder 不支持） |
| 请求量统计、性能指标采集 | LongAdder |
| 序列号生成（需精确递增） | AtomicLong |

---
# Part 9: 并发集合

## 9.1 ConcurrentHashMap（JDK 8）

### 数据结构

```
ConcurrentHashMap（JDK 8）内部结构：

  table（Node[] 数组，默认16，扩容时翻倍）：
  ┌────┬────┬────┬────┬────┬────┬────┬────┐
  │ 0  │ 1  │ 2  │ 3  │ 4  │ 5  │ 6  │ 7  │
  └─┬──┴────┴─┬──┴────┴─┬──┴────┴────┴────┘
    │          │          │
  链表(Node)  红黑树(TreeBin)  链表
  hash<8     链表>=8转树     hash<8

  Node 结构（普通链表节点）：
    final int hash      <- 键的哈希值（spread后）
    final K key
    volatile V val      <- volatile 保证可见性
    volatile Node next  <- volatile 保证可见性

  特殊节点：
    ForwardingNode（hash=-1）：扩容时占位，指向新 table
    TreeBin（hash=-2）：红黑树容器，封装 TreeNode
    ReservationNode（hash=-3）：计算时占位

  关键常量：
    TREEIFY_THRESHOLD = 8   链表长度 >= 8，考虑转红黑树
    UNTREEIFY_THRESHOLD = 6 红黑树节点 <= 6，退化为链表
    MIN_TREEIFY_CAPACITY = 64 table 长度 < 64 时优先扩容而非树化
```

### put 流程（源码分析）

```java
// ConcurrentHashMap put 核心流程（JDK 8 精简版）
final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    // 注：ConcurrentHashMap 不允许 null key/value（与 HashMap 不同）

    int hash = spread(key.hashCode());  // (h ^ (h >>> 16)) & HASH_BITS，高低位混合

    for (Node<K,V>[] tab = table;;) {  // 外层自旋
        Node<K,V> f; int n, i, fh;

        if (tab == null || (n = tab.length) == 0)
            // 1. table 未初始化：CAS 设置 sizeCtl，懒初始化
            tab = initTable();

        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 2. 对应桶为空：直接 CAS 插入（无需加锁）
            if (casTabAt(tab, i, null, new Node<>(hash, key, value, null)))
                break;  // 成功
            // CAS 失败（并发冲突）：重新循环
        }

        else if ((fh = f.hash) == MOVED)
            // 3. 正在扩容（ForwardingNode）：帮助迁移数据
            tab = helpTransfer(tab, f);

        else {
            V oldVal = null;
            // 4. 对桶的头节点加锁（分段锁，粒度=单个桶）
            synchronized (f) {
                if (tabAt(tab, i) == f) {  // 二次确认（防止桶头被替换）
                    if (fh >= 0) {  // 普通链表节点（hash >= 0）
                        int binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            if (e.hash == hash && key.equals(e.key)) {
                                oldVal = e.val;
                                if (!onlyIfAbsent) e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<>(hash, key, value, null);
                                break;
                            }
                        }
                        if (binCount >= TREEIFY_THRESHOLD)
                            treeifyBin(tab, i);  // 链表转红黑树
                    } else if (f instanceof TreeBin) {
                        Node<K,V> p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value);
                        if (p != null) { oldVal = p.val; if (!onlyIfAbsent) p.val = value; }
                    }
                }
            }
            if (oldVal != null) return oldVal;
            break;
        }
    }
    addCount(1L, -1);  // 计数 +1（类似 LongAdder）
    return null;
}
```

### put 流程图

```
putVal 执行流程：

  key/value 非 null 检查
         |
  hash = spread(key.hashCode())
         |
  外层 for(;;) 自旋：
         |
  table 未初始化? -> initTable()（CAS 设置 sizeCtl 防止并发初始化）
         |
  对应桶为空? -> casTabAt() 直接 CAS 插入，无需加锁
         |
  ForwardingNode? -> helpTransfer() 协助扩容
         |
  synchronized(桶头节点) {   <- 分段锁，只锁该桶
      链表遍历更新或尾插
      或红黑树插入
  }
         |
  addCount(1) -> 计数+1（baseCount + CounterCell[]）
         |
  检查是否需要扩容（loadFactor=0.75）
```

### 扩容（多线程协同 transfer）

```
ConcurrentHashMap 扩容（多线程协作）：

  触发条件：
    - 元素数量 > threshold（capacity * 0.75）
    - 某桶链表长度 >= 8 但 table.length < 64

  扩容流程：
  1. 计算新 table 大小（旧容量 * 2）
  2. 每个线程认领一段桶（最小步长 MIN_TRANSFER_STRIDE=16）
  3. 迁移自己负责的桶：
     - 链表拆分（高位0/高位1，类似 HashMap 1.8 的做法）
     - 红黑树拆分或退化为链表
  4. 迁移完成后，在旧位置放 ForwardingNode（hash=-1）
     其他线程 put 遇到 ForwardingNode，自动帮助迁移

  并发控制：
    sizeCtl 的多重含义：
      -1:        正在初始化
      -(1+N):    N 个线程正在扩容
      正数:      下次扩容的阈值（capacity * 0.75）
      0:         未初始化

  线程协作扩容示意：
  线程A: [0~15]  迁移桶 0,1,2,...,15    放 ForwardingNode
  线程B: [16~31] 迁移桶 16,17,...,31   放 ForwardingNode
  线程C: put 遇到 ForwardingNode -> 帮助迁移
  所有线程迁移完成 -> table = nextTable，迁移结束
```

### size 计数原理

```
ConcurrentHashMap.size() 计数：

  类似 LongAdder 的 Striped64 思想：

  ┌─────────────────────────────────────────────────────────────┐
  │  long baseCount     <- 低并发时直接 CAS 更新                  │
  │  CounterCell[] counterCells  <- 高并发时各线程更新自己的Cell   │
  │                                                             │
  │  size() = baseCount + sum(counterCells[i].value)            │
  └─────────────────────────────────────────────────────────────┘

  addCount(1, N) 逻辑：
  1. 尝试 CAS 更新 baseCount
  2. 失败（竞争激烈）：通过线程 probe 映射到 counterCells[i]，CAS 更新 Cell
  3. 仍失败（Cell 竞争）：rehash probe 换一个 Cell
```

### 完整代码示例

```java
import java.util.concurrent.*;
import java.util.*;

public class ConcurrentHashMapDemo {

    public static void main(String[] args) throws InterruptedException {
        ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

        // 基本操作
        map.put("a", 1);
        map.put("b", 2);
        System.out.println(map.get("a"));             // 1
        System.out.println(map.getOrDefault("c", 0)); // 0

        // putIfAbsent：只有 key 不存在才插入
        map.putIfAbsent("a", 100);  // 不会覆盖，a 仍为 1
        System.out.println(map.get("a"));  // 1

        // computeIfAbsent：原子地初始化
        map.computeIfAbsent("count", k -> 0);
        // 原子累加（ConcurrentHashMap 内置方法，线程安全）
        map.merge("count", 1, Integer::sum);
        map.merge("count", 1, Integer::sum);
        System.out.println("count=" + map.get("count"));  // 2

        // compute：原子更新
        map.compute("a", (k, v) -> v == null ? 1 : v + 10);
        System.out.println("a=" + map.get("a"));  // 11

        // 多线程并发写入
        ExecutorService exec = Executors.newFixedThreadPool(10);
        ConcurrentHashMap<Integer, Integer> concMap = new ConcurrentHashMap<>();

        for (int i = 0; i < 10; i++) {
            exec.submit(() -> {
                for (int j = 0; j < 1000; j++) {
                    concMap.merge(j % 100, 1, Integer::sum);
                }
            });
        }
        exec.shutdown();
        exec.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("期望每个key=100，key[0]=" + concMap.get(0));
    }
}
```

### 与其他 Map 对比

| 特性 | HashMap | Hashtable | synchronizedMap | ConcurrentHashMap |
|-----|---------|-----------|-----------------|-------------------|
| 线程安全 | 否 | 是（全表锁） | 是（全表锁） | 是（桶级锁+CAS） |
| null key/value | 允许 | 不允许 | 允许 | 不允许 |
| 并发性能 | 最高（单线程） | 最低 | 低 | 高 |
| 迭代器 | fail-fast | fail-fast | fail-fast | 弱一致性 |
| 推荐 | 单线程 | 不推荐 | 不推荐 | 多线程 |

---

## 9.2 CopyOnWriteArrayList（写时复制）

```
CopyOnWriteArrayList 写时复制原理：

  读操作（无锁）：
    直接读取 array 引用（volatile），不加锁

  写操作（加锁）：
    1. ReentrantLock 加锁
    2. 复制整个数组（创建 newArray）
    3. 在 newArray 上修改
    4. array = newArray（volatile 写，对所有读线程可见）
    5. 解锁

  代价：
    - 写操作复制整个数组，内存占用翻倍（写时）
    - 大数组写操作慢，不适合写频繁场景

  优势：
    - 读操作完全无锁，并发读性能极高
    - 迭代器是快照语义（不会 ConcurrentModificationException）
    - 适合读多写少：白名单、黑名单、监听器列表
```

```java
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.*;

public class CopyOnWriteDemo {

    public static void main(String[] args) throws InterruptedException {
        CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();
        list.add("a"); list.add("b"); list.add("c");

        // 演示：迭代时修改不会抛出 ConcurrentModificationException
        Thread reader = new Thread(() -> {
            for (String s : list) {
                System.out.println("读取: " + s);
                try { Thread.sleep(100); } catch (InterruptedException e) {}
            }
        });

        Thread writer = new Thread(() -> {
            try {
                Thread.sleep(150);
                list.add("d");  // 写时复制，不影响读线程的迭代器（读线程持有旧数组快照）
                System.out.println("写入 d 完成，当前列表: " + list);
            } catch (InterruptedException e) {}
        });

        reader.start();
        writer.start();
        reader.join();
        writer.join();

        // 适用案例：监听器列表（读多写极少）
        CopyOnWriteArrayList<Runnable> listeners = new CopyOnWriteArrayList<>();
        listeners.add(() -> System.out.println("Listener 1"));
        listeners.add(() -> System.out.println("Listener 2"));

        // 并发安全地遍历并回调
        listeners.forEach(Runnable::run);
    }
}
```

---

## 9.3 ConcurrentLinkedQueue（无锁队列）

```java
import java.util.concurrent.ConcurrentLinkedQueue;

public class ConcurrentLinkedQueueDemo {
    // 基于 Michael-Scott 无锁队列算法（CAS 实现）
    // 特点：无界、非阻塞、FIFO
    // 适用：高并发无界队列（不需要阻塞等待）

    public static void main(String[] args) throws InterruptedException {
        ConcurrentLinkedQueue<Integer> queue = new ConcurrentLinkedQueue<>();

        // 生产者
        Thread producer = new Thread(() -> {
            for (int i = 0; i < 100; i++) {
                queue.offer(i);  // 非阻塞，永不失败（无界）
            }
        });

        // 消费者
        Thread consumer = new Thread(() -> {
            int count = 0;
            while (count < 100) {
                Integer val = queue.poll();  // 非阻塞，队列空返回 null
                if (val != null) {
                    count++;
                } else {
                    Thread.yield();  // 队列暂时为空，让出 CPU
                }
            }
        });

        producer.start(); consumer.start();
        producer.join(); consumer.join();
        System.out.println("队列最终大小: " + queue.size());  // 0
    }
}
```

---

## 9.4 BlockingQueue 系列

```
BlockingQueue 接口：
  put(e)     <- 队列满则阻塞（生产者常用）
  take()     <- 队列空则阻塞（消费者常用）
  offer(e)   <- 队列满返回 false（非阻塞）
  poll()     <- 队列空返回 null（非阻塞）
  offer(e, timeout, unit) <- 超时等待
  poll(timeout, unit)     <- 超时等待

BlockingQueue 家族：
┌────────────────────┬────────┬──────────────────────────────────────────┐
│ 类名               │ 有界?  │ 特点                                       │
├────────────────────┼────────┼──────────────────────────────────────────┤
│ArrayBlockingQueue  │ 有界   │ 数组实现，1把锁（put和take同一锁）           │
│LinkedBlockingQueue │ 可选   │ 链表，2把锁（put/take各一把，吞吐更高）      │
│PriorityBlockingQueue│ 无界  │ 优先级堆，元素需实现Comparable             │
│DelayQueue          │ 无界   │ 延迟队列，元素需实现Delayed接口             │
│SynchronousQueue    │ 无缓冲 │ 不存储元素，直接传递（put必须有take对应）    │
│LinkedTransferQueue │ 无界   │ transfer方法：有等待消费者时直接传，否则入队 │
└────────────────────┴────────┴──────────────────────────────────────────┘
```

```java
import java.util.concurrent.*;

public class BlockingQueueDemo {

    // ArrayBlockingQueue：有界阻塞队列
    static void arrayBlockingQueueDemo() throws InterruptedException {
        ArrayBlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

        // put 阻塞（队列满时）
        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    queue.put(i);
                    System.out.println("生产: " + i + "，队列大小: " + queue.size());
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        // take 阻塞（队列空时）
        Thread consumer = new Thread(() -> {
            try {
                for (int i = 0; i < 10; i++) {
                    int val = queue.take();
                    System.out.println("消费: " + val);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        producer.start(); consumer.start();
        producer.join(); consumer.join();
    }

    // PriorityBlockingQueue：按优先级消费
    static void priorityQueueDemo() throws InterruptedException {
        PriorityBlockingQueue<Task> pq = new PriorityBlockingQueue<>();
        pq.offer(new Task("低优先级", 3));
        pq.offer(new Task("高优先级", 1));
        pq.offer(new Task("中优先级", 2));

        // 按优先级顺序取出（优先级数字小的先出）
        while (!pq.isEmpty()) {
            System.out.println("处理: " + pq.take());
        }
    }

    static class Task implements Comparable<Task> {
        String name; int priority;
        Task(String n, int p) { name = n; priority = p; }
        @Override public int compareTo(Task o) { return this.priority - o.priority; }
        @Override public String toString() { return name + "(priority=" + priority + ")"; }
    }

    // DelayQueue：延迟执行
    static void delayQueueDemo() throws InterruptedException {
        DelayQueue<DelayedTask> dq = new DelayQueue<>();

        long now = System.currentTimeMillis();
        dq.offer(new DelayedTask("Task-3秒", now + 3000));
        dq.offer(new DelayedTask("Task-1秒", now + 1000));
        dq.offer(new DelayedTask("Task-2秒", now + 2000));

        System.out.println("开始时间: " + now);
        for (int i = 0; i < 3; i++) {
            DelayedTask task = dq.take();  // 阻塞直到有到期任务
            System.out.println("执行: " + task.name
                + " 实际延迟: " + (System.currentTimeMillis() - now) + "ms");
        }
    }

    static class DelayedTask implements Delayed {
        String name; long executeTime;
        DelayedTask(String n, long t) { name = n; executeTime = t; }

        @Override
        public long getDelay(TimeUnit unit) {
            return unit.convert(executeTime - System.currentTimeMillis(),
                TimeUnit.MILLISECONDS);
        }

        @Override
        public int compareTo(Delayed o) {
            return Long.compare(this.executeTime, ((DelayedTask)o).executeTime);
        }
    }

    // SynchronousQueue：无缓冲，直接传递
    static void synchronousQueueDemo() throws InterruptedException {
        SynchronousQueue<String> sq = new SynchronousQueue<>();

        Thread producer = new Thread(() -> {
            try {
                System.out.println("生产者等待消费者...");
                sq.put("Hello");  // 阻塞直到有消费者来取
                System.out.println("生产者：消费者已取走数据");
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        Thread consumer = new Thread(() -> {
            try {
                Thread.sleep(1000);  // 1秒后消费者来
                String data = sq.take();  // 阻塞直到有生产者放入
                System.out.println("消费者：收到 " + data);
            } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        });

        producer.start(); consumer.start();
        producer.join(); consumer.join();
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== ArrayBlockingQueue ===");
        arrayBlockingQueueDemo();

        System.out.println("\n=== PriorityBlockingQueue ===");
        priorityQueueDemo();

        System.out.println("\n=== DelayQueue ===");
        delayQueueDemo();

        System.out.println("\n=== SynchronousQueue ===");
        synchronousQueueDemo();
    }
}
```

---
# Part 10: 线程池（ThreadPoolExecutor）深度解析

## 10.1 七大参数详解

```java
public ThreadPoolExecutor(
    int corePoolSize,         // 1. 核心线程数：长期保持的线程数量
    int maximumPoolSize,      // 2. 最大线程数：允许创建的最大线程数
    long keepAliveTime,       // 3. 空闲存活时间：非核心线程空闲超过此时间则回收
    TimeUnit unit,            // 4. 时间单位
    BlockingQueue<Runnable> workQueue,  // 5. 工作队列：存放待执行任务
    ThreadFactory threadFactory,        // 6. 线程工厂：自定义线程创建
    RejectedExecutionHandler handler    // 7. 拒绝策略：队列满且线程数达最大时
)
```

**各参数详解：**

```
参数1：corePoolSize（核心线程数）
  - 线程池启动后，初始没有线程（除非 prestartAllCoreThreads）
  - 有任务时才创建核心线程
  - 核心线程即使空闲也不会被回收（除非 allowCoreThreadTimeOut=true）
  - 推荐值：CPU密集型 = CPU核心数+1；IO密集型 = 2*CPU核心数

参数2：maximumPoolSize（最大线程数）
  - 当队列满时，才会创建额外线程（直到达到此上限）
  - 必须 >= corePoolSize
  - 对无界队列（LinkedBlockingQueue无参）：此参数无意义（队列永不满）

参数5：workQueue（工作队列）
  - ArrayBlockingQueue(N)  有界队列，推荐使用（防止OOM）
  - LinkedBlockingQueue()  默认无界（Integer.MAX_VALUE），OOM风险！
  - SynchronousQueue       不存储，直接交给线程执行
  - PriorityBlockingQueue  按优先级执行
  
  Executors.newCachedThreadPool  使用 SynchronousQueue
  Executors.newFixedThreadPool   使用 LinkedBlockingQueue（无界！）
  Executors.newSingleThreadPool  使用 LinkedBlockingQueue（无界！）

参数6：threadFactory（线程工厂）
  推荐自定义，设置：线程名称（便于监控）、是否守护线程、优先级、UncaughtExceptionHandler

参数7：handler（拒绝策略）
  触发条件：workQueue 已满 AND 线程数达到 maximumPoolSize
  AbortPolicy（默认）    抛出 RejectedExecutionException
  CallerRunsPolicy       调用者线程直接执行（降速保护）
  DiscardPolicy          静默丢弃新任务
  DiscardOldestPolicy    丢弃队列中最旧的任务，重新尝试提交
```

---

## 10.2 线程池执行流程

```
任务提交（execute/submit）流程图：

  提交任务
      │
      v
  当前线程数 < corePoolSize? ──是──> 创建核心线程执行任务（即使有空闲线程）
      │ 否
      v
  workQueue.offer(task)
  队列是否入队成功? ──是──> 任务在队列等待（核心线程空闲时取走执行）
      │ 否（队列已满）
      v
  当前线程数 < maximumPoolSize? ──是──> 创建非核心线程执行任务
      │ 否
      v
  执行拒绝策略 handler.rejectedExecution(task, this)

线程执行流程（Worker）：
  Worker.run()
    -> runWorker(this)
       -> firstTask != null ? 执行 firstTask
       -> while(task = getTask()) != null:
             执行 task.run()
          getTask 返回 null（队列空且超时/线程池关闭）-> Worker 结束

直观示意：
  任务流入 ──────────────────────────────────────────────────────>
  
  [核心线程区]        [工作队列]              [临时线程区]
  ┌──────────┐       ┌──────────────┐       ┌──────────┐
  │ Thread-1 │       │              │       │ Thread-4 │
  │ Thread-2 │       │  Queue(100)  │       │ Thread-5 │
  │ Thread-3 │       │              │       └──────────┘
  └──────────┘       └──────────────┘
  core=3             满了才创建临时线程         最大=5
  
  任务1,2,3 -> 创建核心线程
  任务4~103 -> 进入队列（队列大小100）
  任务104,105 -> 创建临时线程（Thread-4,5）
  任务106+ -> 拒绝策略
```

---

## 10.3 四种拒绝策略

```java
import java.util.concurrent.*;

public class RejectionPolicyDemo {

    // 策略1：AbortPolicy（默认）- 直接抛异常
    static void abortDemo() {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            1, 1, 0, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(1),
            new ThreadPoolExecutor.AbortPolicy()  // 或不写，默认就是这个
        );
        try {
            executor.execute(() -> sleep(1000));
            executor.execute(() -> System.out.println("任务2入队"));
            executor.execute(() -> System.out.println("任务3触发拒绝"));  // 抛出异常
        } catch (RejectedExecutionException e) {
            System.out.println("AbortPolicy: 任务被拒绝，抛出异常: " + e.getMessage());
        }
        executor.shutdown();
    }

    // 策略2：CallerRunsPolicy - 调用者线程执行（降速保护，不丢任务）
    static void callerRunsDemo() {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            1, 1, 0, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(1),
            new ThreadPoolExecutor.CallerRunsPolicy()
        );
        for (int i = 0; i < 5; i++) {
            final int id = i;
            executor.execute(() -> {
                System.out.println(Thread.currentThread().getName()
                    + " 执行任务 " + id);
                sleep(500);
            });
        }
        // 当线程池满时，提交任务的线程（main线程）自己执行，起到背压效果
        executor.shutdown();
    }

    // 策略3：DiscardPolicy - 静默丢弃
    static void discardDemo() {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            1, 1, 0, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(1),
            new ThreadPoolExecutor.DiscardPolicy()
        );
        for (int i = 0; i < 5; i++) {
            final int id = i;
            executor.execute(() -> System.out.println("执行任务 " + id));
        }
        // 任务3,4,5被静默丢弃，无任何通知
        executor.shutdown();
    }

    // 策略4：自定义拒绝策略（推荐：记录日志+监控）
    static void customRejectionDemo() {
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
            2, 4, 60, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(100),
            r -> { Thread t = new Thread(r); t.setName("custom-pool"); return t; },
            (task, pool) -> {
                // 自定义策略：记录日志，发送告警，可以重试或持久化到数据库
                System.err.println("[WARN] 任务被拒绝: " + task
                    + "，线程池状态: 线程数=" + pool.getPoolSize()
                    + "，队列大小=" + pool.getQueue().size());
                // 可以选择：1. 抛异常 2. 重试 3. 降级 4. 持久化到MQ
            }
        );
        executor.shutdown();
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== AbortPolicy ===");
        abortDemo();

        Thread.sleep(500);
        System.out.println("\n=== CallerRunsPolicy ===");
        callerRunsDemo();
    }
}
```

---

## 10.4 线程池状态机

```
ThreadPoolExecutor 状态（存储在 ctl 高3位）：

  ctl = AtomicInteger（高3位=状态，低29位=线程数）

  状态常量：
    RUNNING     = -1 << COUNT_BITS  // 111（二进制）接收新任务，处理队列任务
    SHUTDOWN    =  0 << COUNT_BITS  // 000 不接新任务，但处理队列已有任务
    STOP        =  1 << COUNT_BITS  // 001 不接新任务，不处理队列，中断运行中线程
    TIDYING     =  2 << COUNT_BITS  // 010 所有任务已终止，准备执行 terminated()
    TERMINATED  =  3 << COUNT_BITS  // 011 terminated() 已执行完毕

  状态转换：
  
  RUNNING ──shutdown()──────> SHUTDOWN ──队列清空，线程数=0──> TIDYING
      │                                                              │
      │  shutdownNow()                                              terminated()
      │        │                                                     │
      └────────┴──────────────> STOP ──线程数=0──────────────> TIDYING
                                                                      │
                                                              TERMINATED

  API 对应：
    shutdown():    RUNNING -> SHUTDOWN（优雅关闭，等待任务完成）
    shutdownNow(): RUNNING -> STOP（强制停止，返回未执行的任务列表）
    isShutdown():  != RUNNING
    isTerminated(): == TERMINATED
    awaitTermination(timeout): 等待 TERMINATED，或超时
```

---

## 10.5 工作线程 Worker 执行流程

```java
// Worker 核心逻辑（精简）
private final class Worker extends AbstractQueuedSynchronizer
        implements Runnable {

    final Thread thread;
    Runnable firstTask;

    Worker(Runnable firstTask) {
        setState(-1);  // 初始state=-1，防止在启动前被中断
        this.firstTask = firstTask;
        this.thread = getThreadFactory().newThread(this);
    }

    public void run() {
        runWorker(this);
    }
}

// runWorker：Worker 线程的核心执行循环
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;
    w.unlock();  // 允许中断（state=-1 -> 0）

    boolean completedAbruptly = true;
    try {
        // 核心循环：先执行 firstTask，然后不断从队列取任务
        while (task != null || (task = getTask()) != null) {
            w.lock();  // 任务执行时加锁（防止被 shutdown 中断）
            // 检查线程池状态，如果是 STOP，中断当前线程
            if ((runStateAtLeast(ctl.get(), STOP) ||
                 (Thread.interrupted() && runStateAtLeast(ctl.get(), STOP))) &&
                !wt.isInterrupted())
                wt.interrupt();
            try {
                beforeExecute(wt, task);    // 钩子：任务执行前（可重写）
                Throwable thrown = null;
                try {
                    task.run();             // 执行任务
                } catch (RuntimeException x) { thrown = x; throw x; }
                  catch (Error x)           { thrown = x; throw x; }
                  catch (Throwable x)       { thrown = x; throw new Error(x); }
                  finally { afterExecute(task, thrown); }  // 钩子：执行后
            } finally {
                task = null;
                w.completedTasks++;
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);  // 线程退出处理
    }
}

// getTask：从队列获取任务（控制线程存活）
private Runnable getTask() {
    boolean timedOut = false;
    for (;;) {
        int c = ctl.get();
        int rs = runStateOf(c);

        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;  // 线程池关闭或队列空，Worker 退出
        }

        int wc = workerCountOf(c);
        // 是否需要超时控制？
        // 1. allowCoreThreadTimeOut=true（核心线程也可超时）
        // 2. 当前线程数 > corePoolSize（非核心线程）
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;  // 减少线程数，Worker 退出
            continue;
        }

        try {
            // 有超时控制：poll（超时返回null）；无：take（一直阻塞）
            Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null) return r;
            timedOut = true;  // 超时，下次循环会退出
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

---

## 10.6 四种预设线程池及其问题

```java
import java.util.concurrent.*;

public class PresetThreadPoolDemo {

    // 1. newFixedThreadPool：固定线程数
    static ExecutorService createFixed() {
        // 内部：new ThreadPoolExecutor(n, n, 0L, MILLISECONDS, new LinkedBlockingQueue<>())
        // 问题：LinkedBlockingQueue 无界！任务堆积可能 OOM
        return Executors.newFixedThreadPool(4);
    }

    // 2. newCachedThreadPool：按需创建，无上限
    static ExecutorService createCached() {
        // 内部：new ThreadPoolExecutor(0, MAX_VALUE, 60L, SECONDS, new SynchronousQueue<>())
        // 问题：maximumPoolSize = Integer.MAX_VALUE！短时大量请求可能创建大量线程，OOM
        return Executors.newCachedThreadPool();
    }

    // 3. newSingleThreadExecutor：单线程，顺序执行
    static ExecutorService createSingle() {
        // 内部：new ThreadPoolExecutor(1, 1, 0L, MILLISECONDS, new LinkedBlockingQueue<>())
        // 问题：同 FixedThreadPool，队列无界
        return Executors.newSingleThreadExecutor();
    }

    // 4. newScheduledThreadPool：定时/延迟执行
    static ScheduledExecutorService createScheduled() {
        // 问题：maximumPoolSize = Integer.MAX_VALUE（与 CachedThreadPool 类似问题）
        ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(4);

        // 延迟3秒执行
        scheduler.schedule(() -> System.out.println("延迟任务"), 3, TimeUnit.SECONDS);

        // 固定频率执行（每2秒一次，与上次开始时间计算）
        scheduler.scheduleAtFixedRate(
            () -> System.out.println("固定频率: " + System.currentTimeMillis()),
            1, 2, TimeUnit.SECONDS);

        // 固定延迟执行（每2秒一次，与上次结束时间计算）
        scheduler.scheduleWithFixedDelay(
            () -> System.out.println("固定延迟: " + System.currentTimeMillis()),
            1, 2, TimeUnit.SECONDS);

        return scheduler;
    }

    // 推荐：自定义线程池（阿里Java开发手册规范）
    static ThreadPoolExecutor createCustom() {
        return new ThreadPoolExecutor(
            5,                              // corePoolSize
            20,                             // maximumPoolSize
            60, TimeUnit.SECONDS,           // keepAliveTime
            new ArrayBlockingQueue<>(1000), // 有界队列，防止OOM
            new ThreadFactory() {
                private final java.util.concurrent.atomic.AtomicInteger count =
                    new java.util.concurrent.atomic.AtomicInteger(0);
                @Override
                public Thread newThread(Runnable r) {
                    Thread t = new Thread(r, "my-pool-" + count.getAndIncrement());
                    t.setDaemon(false);
                    t.setUncaughtExceptionHandler((thread, e) ->
                        System.err.println("线程异常: " + thread.getName() + ", " + e));
                    return t;
                }
            },
            new ThreadPoolExecutor.CallerRunsPolicy()  // 调用者执行，起到背压效果
        );
    }
}
```

---

## 10.7 线程池参数最佳实践

```
线程池参数公式（参考，实际需压测验证）：

CPU 密集型任务（大量计算，很少 IO 等待）：
  - corePoolSize = CPU核心数 + 1
  - 多1个是为了当线程因偶发缺页/GC停顿时，有线程顶上
  - maximumPoolSize = corePoolSize（无需扩容）
  - 队列：SynchronousQueue 或较小的 ArrayBlockingQueue

IO 密集型任务（数据库查询、网络请求，大量等待）：
  - corePoolSize = 2 * CPU核心数（或更高）
  - 精确公式：N_cpu * U_cpu * (1 + W/C)
    N_cpu = CPU核心数，U_cpu = 目标CPU利用率（0.7），W/C = 等待时间/计算时间
  - 实际：根据业务测试，常见值 50~200
  - 队列：ArrayBlockingQueue(N)，N根据接口TPS和响应时间估算

混合型任务：
  - 分拆为 CPU 密集和 IO 密集两个线程池
  - 相互隔离，互不影响

超时任务（防止线程被占用太久）：
  - 设置 keepAliveTime
  - 任务内部设置超时
  - Future.get(timeout) 或 CompletableFuture.orTimeout()

动态调整（生产环境推荐）：
  - 使用 ThreadPoolExecutor.setCorePoolSize() 动态调整
  - 配合配置中心（Apollo/Nacos）实现运行时参数调整
  - 实现 ThreadPoolExecutor 的 beforeExecute/afterExecute 进行监控
```

---

## 10.8 线程池监控与优雅关闭

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

public class ThreadPoolMonitor {

    // 自定义监控线程池
    static class MonitoredThreadPool extends ThreadPoolExecutor {
        private final String name;
        private final AtomicLong totalTaskCount = new AtomicLong(0);
        private final AtomicLong failedTaskCount = new AtomicLong(0);

        MonitoredThreadPool(String name, int core, int max, BlockingQueue<Runnable> queue) {
            super(core, max, 60, TimeUnit.SECONDS, queue,
                r -> {
                    Thread t = new Thread(r, name + "-" + r.hashCode());
                    t.setUncaughtExceptionHandler((th, e) ->
                        System.err.println("[" + name + "] 线程异常: " + e));
                    return t;
                },
                new ThreadPoolExecutor.CallerRunsPolicy());
            this.name = name;
        }

        @Override
        protected void beforeExecute(Thread t, Runnable r) {
            super.beforeExecute(t, r);
            totalTaskCount.incrementAndGet();
        }

        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            super.afterExecute(r, t);
            if (t != null) failedTaskCount.incrementAndGet();
        }

        public void printStatus() {
            System.out.printf("[%s] 线程池状态: 核心=%d, 活跃=%d, 总线程=%d, " +
                "队列=%d, 完成=%d, 失败=%d%n",
                name,
                getCorePoolSize(),
                getActiveCount(),
                getPoolSize(),
                getQueue().size(),
                getCompletedTaskCount(),
                failedTaskCount.get()
            );
        }
    }

    // 优雅关闭
    static void gracefulShutdown(ExecutorService executor) {
        executor.shutdown();  // 停止接受新任务，等待已提交任务完成
        try {
            // 等待最多60秒
            if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                // 超时，强制关闭
                executor.shutdownNow();
                // 再等一会儿
                if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
                    System.err.println("线程池未能正常关闭");
                }
            }
        } catch (InterruptedException e) {
            // 等待过程中被中断，强制关闭
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
        System.out.println("线程池已关闭");
    }

    // shutdown vs shutdownNow 区别：
    // shutdown():     软关闭，不接受新任务，等待当前任务和队列中的任务都完成
    // shutdownNow():  硬关闭，interrupt 所有线程，返回未开始执行的任务列表
    //                 注意：不保证所有线程都能停止（线程需响应中断）

    public static void main(String[] args) throws InterruptedException {
        MonitoredThreadPool pool = new MonitoredThreadPool(
            "demo-pool", 2, 5, new ArrayBlockingQueue<>(20));

        for (int i = 0; i < 10; i++) {
            final int id = i;
            pool.execute(() -> {
                try { Thread.sleep(100); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }

        Thread.sleep(500);
        pool.printStatus();
        gracefulShutdown(pool);
    }
}
```

---

## 10.9 自定义线程池完整案例

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

/**
 * 生产可用的自定义线程池：
 * - 有界队列防 OOM
 * - 自定义线程工厂（命名+异常处理）
 * - CallerRunsPolicy 背压
 * - 监控统计
 * - 优雅关闭
 */
public class ProductionThreadPool {

    private final ThreadPoolExecutor executor;
    private final String poolName;

    public ProductionThreadPool(String name, int coreSize, int maxSize, int queueSize) {
        this.poolName = name;
        AtomicInteger threadNum = new AtomicInteger(1);

        this.executor = new ThreadPoolExecutor(
            coreSize,
            maxSize,
            60L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(queueSize),
            r -> {
                Thread t = new Thread(r);
                t.setName(name + "-worker-" + threadNum.getAndIncrement());
                t.setDaemon(false);
                t.setUncaughtExceptionHandler((thread, e) -> {
                    System.err.println("[CRITICAL] 线程[" + thread.getName()
                        + "] 未捕获异常: " + e.getMessage());
                    e.printStackTrace();
                });
                return t;
            },
            (task, pool) -> {
                // 记录拒绝日志（生产环境接入监控告警）
                System.err.println("[WARN] 任务被拒绝! poolName=" + name
                    + ", activeThreads=" + pool.getActiveCount()
                    + ", queueSize=" + pool.getQueue().size());
                // 策略：调用者执行（防止任务丢失）
                if (!pool.isShutdown()) {
                    task.run();
                }
            }
        );

        // 预热核心线程
        executor.prestartAllCoreThreads();
    }

    public void execute(Runnable task) {
        executor.execute(task);
    }

    public <T> Future<T> submit(Callable<T> task) {
        return executor.submit(task);
    }

    public void printStats() {
        System.out.printf("[%s] 核心=%d 最大=%d 活跃=%d 总线程=%d 队列=%d 完成=%d%n",
            poolName,
            executor.getCorePoolSize(),
            executor.getMaximumPoolSize(),
            executor.getActiveCount(),
            executor.getPoolSize(),
            executor.getQueue().size(),
            executor.getCompletedTaskCount()
        );
    }

    public void shutdown() {
        executor.shutdown();
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                System.err.println("[WARN] 线程池未在30秒内关闭，强制关闭");
                executor.shutdownNow();
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }

    // 动态调整参数（配合配置中心使用）
    public void resize(int newCoreSize, int newMaxSize) {
        if (newCoreSize <= newMaxSize) {
            executor.setMaximumPoolSize(newMaxSize);  // 先扩 max
            executor.setCorePoolSize(newCoreSize);
        } else {
            executor.setCorePoolSize(newCoreSize);    // 先缩 core
            executor.setMaximumPoolSize(newMaxSize);
        }
        System.out.println("线程池已调整: core=" + newCoreSize + ", max=" + newMaxSize);
    }

    public static void main(String[] args) throws Exception {
        ProductionThreadPool pool = new ProductionThreadPool("order-pool", 4, 8, 200);

        // 提交任务
        for (int i = 0; i < 20; i++) {
            final int orderId = i;
            pool.execute(() -> {
                System.out.printf("[%s] 处理订单 #%d%n",
                    Thread.currentThread().getName(), orderId);
                try { Thread.sleep(100); } catch (InterruptedException e) {}
            });
        }

        Thread.sleep(500);
        pool.printStats();
        pool.shutdown();
    }
}
```

---
# Part 11: Future 与异步编程

## 11.1 Future / Callable 基本使用

```java
import java.util.concurrent.*;

public class FutureBasicDemo {

    public static void main(String[] args) throws Exception {
        ExecutorService exec = Executors.newFixedThreadPool(3);

        // 提交 Callable，获得 Future
        Future<Integer> f1 = exec.submit(() -> {
            Thread.sleep(1000);
            return 42;
        });

        System.out.println("任务已提交，做其他事...");

        // get() 阻塞直到有结果
        System.out.println("结果: " + f1.get());

        // get(timeout) 超时等待
        try {
            Future<String> f2 = exec.submit(() -> {
                Thread.sleep(5000);
                return "慢任务";
            });
            String result = f2.get(2, TimeUnit.SECONDS);  // 2秒超时
        } catch (TimeoutException e) {
            System.out.println("任务超时！");
        }

        // cancel() 取消任务
        Future<String> f3 = exec.submit(() -> {
            Thread.sleep(3000);
            return "可取消任务";
        });
        f3.cancel(true);  // true=中断正在执行的线程
        System.out.println("已取消: " + f3.isCancelled());

        exec.shutdown();
    }
}
```

## 11.2 FutureTask 实现原理

```
FutureTask 内部状态机：

  NEW(0) ──run()开始执行──> COMPLETING(1)
                                │
              ├─── 正常完成──> NORMAL(2)     set(value) 写入结果
              ├─── 异常结束──> EXCEPTIONAL(3) setException(e)
              └─── 取消开始──> CANCELLED(4)

  NEW ──cancel(false)──> CANCELLED(4)
  NEW ──cancel(true) ──> INTERRUPTING(5) ──> INTERRUPTED(6)

  get() 实现：
    state == NORMAL: 直接返回 outcome
    state == EXCEPTIONAL: 抛出 outcome（包装的异常）
    state < COMPLETING: 加入等待队列（LockSupport.park），等待 run() 完成后唤醒

  run() 完成后：
    调用 finishCompletion()
    唤醒所有等待 get() 的线程
```

```java
import java.util.concurrent.*;

public class FutureTaskDemo {

    public static void main(String[] args) throws Exception {
        // FutureTask = Callable + Future 的组合
        FutureTask<Integer> futureTask = new FutureTask<>(() -> {
            System.out.println("FutureTask 执行中...");
            Thread.sleep(1000);
            return 100;
        });

        // 可以直接传给 Thread 执行（不依赖线程池）
        Thread t = new Thread(futureTask, "FutureTask-Thread");
        t.start();

        // 也可以提交给 ExecutorService
        // executor.submit(futureTask);

        System.out.println("等待结果...");
        System.out.println("结果: " + futureTask.get());

        // 缓存计算结果（多次调用 get 只计算一次）
        System.out.println("再次获取: " + futureTask.get());  // 直接返回缓存值
    }
}
```

---

## 11.3 CompletableFuture（Java 8）

`CompletableFuture` 是 Java 8 引入的**异步编程框架**，支持链式调用、组合、异常处理。

```
CompletableFuture 设计理念：
  1. 链式编程：f.thenApply().thenAccept()
  2. 组合：allOf / anyOf / thenCompose / thenCombine
  3. 异常处理：exceptionally / handle / whenComplete
  4. 默认异步线程池：ForkJoinPool.commonPool()
  5. 也可指定自定义线程池（推荐）

核心方法分类：
  创建：supplyAsync / runAsync
  转换：thenApply（有返回值）/ thenAccept（消费）/ thenRun（无参数无返回值）
  组合：thenCompose（扁平化）/ thenCombine（合并两个）
  聚合：allOf（等全部）/ anyOf（等任一）
  异常：exceptionally / handle / whenComplete
```

```java
import java.util.concurrent.*;

public class CompletableFutureDemo {

    static ExecutorService executor = Executors.newFixedThreadPool(4);

    // ===== 创建 =====
    static void creation() throws Exception {
        // supplyAsync：有返回值的异步任务
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
            System.out.println("supplyAsync 执行: " + Thread.currentThread().getName());
            return "Hello";
        }, executor);  // 推荐指定线程池

        // runAsync：无返回值的异步任务
        CompletableFuture<Void> f2 = CompletableFuture.runAsync(() -> {
            System.out.println("runAsync 执行");
        }, executor);

        // 直接创建已完成的 Future
        CompletableFuture<Integer> f3 = CompletableFuture.completedFuture(42);

        System.out.println("f1=" + f1.get() + ", f3=" + f3.get());
    }

    // ===== 转换链 =====
    static void transformation() throws Exception {
        CompletableFuture<String> result = CompletableFuture
            .supplyAsync(() -> "hello", executor)           // 返回 "hello"
            .thenApply(s -> s.toUpperCase())                // "HELLO"
            .thenApply(s -> s + " WORLD")                  // "HELLO WORLD"
            .thenApply(s -> {
                System.out.println("当前线程: " + Thread.currentThread().getName());
                return s.toLowerCase();
            });                                             // "hello world"

        System.out.println("转换结果: " + result.get());

        // thenAccept：消费结果，无返回值
        CompletableFuture<Void> consumer = CompletableFuture
            .supplyAsync(() -> "data", executor)
            .thenAccept(s -> System.out.println("消费: " + s));
        consumer.join();

        // thenRun：不关心上一步结果，执行后续动作
        CompletableFuture<Void> runner = CompletableFuture
            .supplyAsync(() -> "result", executor)
            .thenRun(() -> System.out.println("任务完成，执行清理"));
        runner.join();

        // thenApplyAsync：在新线程执行（与 thenApply 的区别：是否使用同一线程）
        CompletableFuture<String> asyncTransform = CompletableFuture
            .supplyAsync(() -> "data", executor)
            .thenApplyAsync(s -> {
                System.out.println("async转换线程: " + Thread.currentThread().getName());
                return s + "_processed";
            }, executor);  // 在指定线程池执行
        System.out.println("异步转换: " + asyncTransform.get());
    }

    // ===== 组合 =====
    static void combination() throws Exception {
        // thenCompose：扁平化嵌套（避免 CompletableFuture<CompletableFuture<>>）
        CompletableFuture<String> composed = CompletableFuture
            .supplyAsync(() -> "userId=123", executor)
            .thenCompose(userId -> CompletableFuture.supplyAsync(() -> {
                // 用 userId 查用户信息
                return "User{id=123, name=Alice}";
            }, executor));
        System.out.println("thenCompose: " + composed.get());

        // thenCombine：合并两个独立的 Future
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(
            () -> { sleep(500); return "价格:100"; }, executor);
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(
            () -> { sleep(300); return "库存:50"; }, executor);

        CompletableFuture<String> combined = f1.thenCombine(f2,
            (price, stock) -> price + ", " + stock);
        System.out.println("thenCombine: " + combined.get());
        // 两个任务并行执行，总耗时 max(500,300) = 500ms（而非 800ms）

        // allOf：等待所有完成
        CompletableFuture<String> fa = CompletableFuture.supplyAsync(() -> "A", executor);
        CompletableFuture<String> fb = CompletableFuture.supplyAsync(() -> "B", executor);
        CompletableFuture<String> fc = CompletableFuture.supplyAsync(() -> "C", executor);

        CompletableFuture<Void> all = CompletableFuture.allOf(fa, fb, fc);
        all.join();  // 等待全部完成
        System.out.println("allOf 结果: " + fa.get() + fb.get() + fc.get());

        // anyOf：任意一个完成就返回
        CompletableFuture<Object> any = CompletableFuture.anyOf(
            CompletableFuture.supplyAsync(() -> { sleep(1000); return "慢任务"; }, executor),
            CompletableFuture.supplyAsync(() -> { sleep(100);  return "快任务"; }, executor)
        );
        System.out.println("anyOf 结果: " + any.get());  // "快任务"
    }

    // ===== 异常处理 =====
    static void exceptionHandling() throws Exception {
        // exceptionally：只处理异常，正常情况跳过
        CompletableFuture<String> f1 = CompletableFuture
            .supplyAsync(() -> {
                if (Math.random() > 0.5) throw new RuntimeException("随机失败");
                return "成功";
            }, executor)
            .exceptionally(e -> "默认值（" + e.getMessage() + "）");
        System.out.println("exceptionally: " + f1.get());

        // handle：无论正常还是异常都执行（类似 try-catch-finally）
        CompletableFuture<String> f2 = CompletableFuture
            .supplyAsync(() -> { throw new RuntimeException("故意失败"); }, executor)
            .handle((result, ex) -> {
                if (ex != null) {
                    System.out.println("handle 捕获异常: " + ex.getMessage());
                    return "fallback";
                }
                return result.toString();
            });
        System.out.println("handle: " + f2.get());

        // whenComplete：观察结果（不改变结果，类似 peek）
        CompletableFuture<Integer> f3 = CompletableFuture
            .supplyAsync(() -> 42, executor)
            .whenComplete((result, ex) -> {
                if (ex != null) System.out.println("失败: " + ex);
                else System.out.println("成功: " + result);
                // 不改变 result，后续仍可获取 42
            });
        System.out.println("whenComplete 结果: " + f3.get());  // 42
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }

    public static void main(String[] args) throws Exception {
        System.out.println("=== 创建 ==="); creation();
        System.out.println("\n=== 转换 ==="); transformation();
        System.out.println("\n=== 组合 ==="); combination();
        System.out.println("\n=== 异常处理 ==="); exceptionHandling();
        executor.shutdown();
    }
}
```

---

## 11.4 完整异步编排案例（商品详情页）

```java
import java.util.concurrent.*;

/**
 * 电商商品详情页异步聚合案例
 * 需要并发查询：商品基本信息、库存、评价、推荐商品
 * 使用 CompletableFuture 并行查询，大幅降低响应时间
 */
public class ProductDetailPageDemo {

    static ExecutorService executor = Executors.newFixedThreadPool(8);

    // 模拟数据对象
    record ProductInfo(Long id, String name, double price) {}
    record StockInfo(Long productId, int stock, String warehouse) {}
    record ReviewInfo(Long productId, double rating, int reviewCount) {}
    record RecommendInfo(Long productId, java.util.List<Long> recommendIds) {}
    record ProductDetailVO(ProductInfo product, StockInfo stock,
                           ReviewInfo review, RecommendInfo recommend) {}

    // ===== 模拟各服务调用（实际是 RPC 或 HTTP 请求）=====

    static CompletableFuture<ProductInfo> getProductInfo(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(100);  // 模拟 100ms 查询耗时
            System.out.println("[商品服务] 查询商品: " + productId);
            return new ProductInfo(productId, "iPhone 15", 5999.0);
        }, executor);
    }

    static CompletableFuture<StockInfo> getStockInfo(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(80);
            System.out.println("[库存服务] 查询库存: " + productId);
            return new StockInfo(productId, 100, "上海仓");
        }, executor);
    }

    static CompletableFuture<ReviewInfo> getReviewInfo(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(150);
            System.out.println("[评价服务] 查询评价: " + productId);
            return new ReviewInfo(productId, 4.8, 9999);
        }, executor);
    }

    static CompletableFuture<RecommendInfo> getRecommendInfo(Long productId) {
        return CompletableFuture.supplyAsync(() -> {
            sleep(200);
            System.out.println("[推荐服务] 查询推荐: " + productId);
            return new RecommendInfo(productId, java.util.List.of(2L, 3L, 4L));
        }, executor);
    }

    // ===== 聚合查询（并行） =====
    static CompletableFuture<ProductDetailVO> getProductDetail(Long productId) {
        long start = System.currentTimeMillis();

        // 4个服务并行查询（非阻塞！）
        CompletableFuture<ProductInfo>    cfProduct   = getProductInfo(productId);
        CompletableFuture<StockInfo>      cfStock     = getStockInfo(productId);
        CompletableFuture<ReviewInfo>     cfReview    = getReviewInfo(productId);
        CompletableFuture<RecommendInfo>  cfRecommend = getRecommendInfo(productId);

        // 等待全部完成，聚合结果
        return CompletableFuture.allOf(cfProduct, cfStock, cfReview, cfRecommend)
            .thenApply(v -> {
                long elapsed = System.currentTimeMillis() - start;
                System.out.println("全部查询完成，耗时: " + elapsed + "ms");
                // 注意：allOf 完成时，各 CF 的 get() 不会再阻塞
                try {
                    return new ProductDetailVO(
                        cfProduct.get(),
                        cfStock.get(),
                        cfReview.get(),
                        cfRecommend.get()
                    );
                } catch (Exception e) {
                    throw new RuntimeException("聚合失败", e);
                }
            })
            .exceptionally(e -> {
                System.err.println("查询失败: " + e.getMessage());
                return null;
            });
    }

    // ===== 带超时的聚合（生产环境推荐）=====
    static ProductDetailVO getProductDetailWithTimeout(Long productId) throws Exception {
        CompletableFuture<ProductDetailVO> cf = getProductDetail(productId);

        try {
            // Java 9+ 可用 cf.orTimeout(500, TimeUnit.MILLISECONDS)
            return cf.get(500, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            System.out.println("查询超时，使用降级数据");
            cf.cancel(true);
            // 降级：返回部分数据或默认值
            return new ProductDetailVO(
                new ProductInfo(productId, "加载中...", 0),
                new StockInfo(productId, -1, ""),
                new ReviewInfo(productId, 0, 0),
                new RecommendInfo(productId, java.util.List.of())
            );
        }
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); } catch (InterruptedException e) {}
    }

    public static void main(String[] args) throws Exception {
        System.out.println("=== 商品详情页并发查询 ===");
        System.out.println("理论串行耗时: 100+80+150+200=530ms");
        System.out.println("并行耗时约: max(100,80,150,200)=200ms");
        System.out.println();

        long start = System.currentTimeMillis();
        ProductDetailVO detail = getProductDetail(1L).get();
        long elapsed = System.currentTimeMillis() - start;

        if (detail != null) {
            System.out.println("\n查询结果:");
            System.out.println("  商品: " + detail.product().name()
                + " ¥" + detail.product().price());
            System.out.println("  库存: " + detail.stock().stock()
                + " (" + detail.stock().warehouse() + ")");
            System.out.println("  评分: " + detail.review().rating()
                + " (" + detail.review().reviewCount() + "条)");
            System.out.println("  推荐: " + detail.recommend().recommendIds());
        }
        System.out.println("实际耗时: " + elapsed + "ms");

        executor.shutdown();
    }
}
```

---
# Part 12: ThreadLocal 深度解析

## 12.1 ThreadLocal 使用场景

`ThreadLocal` 提供**线程局部变量**：每个线程有独立的变量副本，互不影响。

**典型使用场景：**
1. 数据库连接/Session 隔离（事务管理）
2. 用户上下文传递（Request 上下文）
3. 日期格式化（SimpleDateFormat 非线程安全）
4. MDC 日志上下文（Slf4j MDC 底层使用 ThreadLocal）

```java
import java.text.SimpleDateFormat;
import java.util.*;

public class ThreadLocalDemo {

    // 场景1：SimpleDateFormat 线程安全问题解决
    // SimpleDateFormat 不是线程安全的，不能共享，用 ThreadLocal 保证每个线程独有一个
    private static final ThreadLocal<SimpleDateFormat> dateFormat =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String formatDate(Date date) {
        return dateFormat.get().format(date);  // 每个线程使用自己的 SDF 实例
    }

    // 场景2：用户上下文传递
    static class UserContext {
        private static final ThreadLocal<String> currentUser = new ThreadLocal<>();

        public static void setUser(String userId) { currentUser.set(userId); }
        public static String getUser() { return currentUser.get(); }
        public static void clear() { currentUser.remove(); }  // 重要！
    }

    // 场景3：数据库连接（事务管理示意）
    static class ConnectionHolder {
        private static final ThreadLocal<Object> connectionHolder = new ThreadLocal<>();

        public static void beginTransaction(Object conn) {
            connectionHolder.set(conn);
        }
        public static Object getConnection() {
            return connectionHolder.get();
        }
        public static void endTransaction() {
            connectionHolder.remove();  // 必须清理！
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // 多线程使用 ThreadLocal，各自独立
        Runnable task = () -> {
            String threadName = Thread.currentThread().getName();
            UserContext.setUser("user-" + threadName);
            System.out.println(threadName + " 设置用户: " + UserContext.getUser());
            try {
                Thread.sleep(100);
                // 100ms 后仍是自己的值，不被其他线程影响
                System.out.println(threadName + " 获取用户: " + UserContext.getUser());
            } catch (InterruptedException e) {}
            finally {
                UserContext.clear();  // 清理，防止内存泄漏
            }
        };

        Thread t1 = new Thread(task, "T1");
        Thread t2 = new Thread(task, "T2");
        t1.start(); t2.start();
        t1.join(); t2.join();
    }
}
```

---

## 12.2 ThreadLocalMap 内部结构

```
ThreadLocal 数据结构：

  Thread 对象内部：
  ┌──────────────────────────────────────────────────────────────────────┐
  │  Thread                                                              │
  │  ...                                                                 │
  │  ThreadLocal.ThreadLocalMap threadLocals;  <- 每个线程都有自己的Map   │
  │  ...                                                                 │
  └──────────────────────────────────────────────────────────────────────┘
  
  ThreadLocalMap 结构（自定义 HashMap）：
  ┌──────────────────────────────────────────────────────────────────────┐
  │  ThreadLocalMap                                                      │
  │                                                                      │
  │  Entry[] table;  // 开放地址法哈希表（线性探测）                       │
  │                                                                      │
  │  table[i] = Entry {                                                  │
  │    WeakReference<ThreadLocal<?>> key;  // 弱引用！                   │
  │    Object value;                       // 强引用                     │
  │  }                                                                   │
  └──────────────────────────────────────────────────────────────────────┘

  ThreadLocal、ThreadLocalMap、Thread、Value 的关系：

  ThreadLocal 实例（可能是静态的）
      |
      |  set(value) / get()
      v
  Thread.threadLocals（ThreadLocalMap）
      |
      |  以 ThreadLocal 弱引用为 key，value 为 value
      v
  Entry { WeakReference<ThreadLocal> -> value }

  关键：ThreadLocalMap 的 key 是 ThreadLocal 的弱引用
       但 value 是强引用！
```

---

## 12.3 内存泄漏原因与解决

```
内存泄漏发生场景（线程池 + ThreadLocal）：

  正常情况（线程生命周期短）：
    线程结束 -> Thread 对象被 GC -> threadLocals 被 GC -> Entry 被 GC -> 无泄漏

  问题场景（线程池中的长生命周期线程）：

  ┌─────────────────────────────────────────────────────────────────────────┐
  │                                                                         │
  │  static ThreadLocal<BigObject> tl = new ThreadLocal<>();               │
  │         ^                                                               │
  │         │ 强引用                                                         │
  │  ThreadLocal 实例                                                        │
  │         ^                                                               │
  │         │ 弱引用（Entry.key）                                             │
  │  ThreadLocalMap.Entry                                                   │
  │         |                                                               │
  │         | 强引用（Entry.value）                                          │
  │         v                                                               │
  │  BigObject（堆内存）                                                     │
  │                                                                         │
  │  如果 tl = null（ThreadLocal 引用被置 null）：                           │
  │    弱引用 key 在下次 GC 时被回收 -> Entry.key = null                     │
  │    但 Entry.value (BigObject) 仍然强引用，不会被 GC！                     │
  │    线程池中的线程长期存活 -> BigObject 一直在内存中 -> 内存泄漏！          │
  │                                                                         │
  └─────────────────────────────────────────────────────────────────────────┘

内存泄漏时序：
  1. ThreadLocal 变量 tl 设置为 null（或方法结束出作用域）
  2. GC 发生，弱引用 key 被回收 -> Entry.key = null，但 Entry.value 还在
  3. 线程池中的线程长期运行，threadLocals 中积累大量 key=null 的 Entry
  4. value 中的大对象无法被 GC 回收 -> 内存泄漏
```

**解决方案：**

```java
public class ThreadLocalMemoryLeakFix {

    private static final ThreadLocal<byte[]> tl = new ThreadLocal<>();

    // 使用线程池的场景
    static ExecutorService executor = Executors.newFixedThreadPool(5);

    // 错误示范（内存泄漏风险）
    static void badPractice() {
        executor.execute(() -> {
            tl.set(new byte[1024 * 1024]);  // 设置 1MB 数据
            // 没有 remove()！线程归还线程池后，value 仍然存在
            doWork();
            // 线程不会结束，tl 中的数据一直保留 -> 内存泄漏
        });
    }

    // 正确做法：使用 try-finally 确保 remove()
    static void goodPractice() {
        executor.execute(() -> {
            try {
                tl.set(new byte[1024 * 1024]);
                doWork();
            } finally {
                tl.remove();  // 必须在 finally 中调用！
                // 清除 Entry，BigObject 可以被 GC
            }
        });
    }

    // 更好的做法：使用 try-with-resources 模式（Java 7+）
    static class ThreadLocalScope<T> implements AutoCloseable {
        private final ThreadLocal<T> threadLocal;

        ThreadLocalScope(ThreadLocal<T> tl, T value) {
            this.threadLocal = tl;
            tl.set(value);
        }

        @Override public void close() {
            threadLocal.remove();
        }
    }

    static void bestPractice() {
        executor.execute(() -> {
            try (var scope = new ThreadLocalScope<>(tl, new byte[1024 * 1024])) {
                doWork();
            }  // close() 自动调用 tl.remove()
        });
    }

    static void doWork() { /* 业务逻辑 */ }

    // ThreadLocalMap 的自动清理（expungeStaleEntries）
    // ThreadLocal.get/set/remove 操作时会顺带清理 key=null 的 Entry
    // 但这依赖 ThreadLocal 被调用，不能完全依赖此机制

    public static void main(String[] args) {
        goodPractice();
        bestPractice();
        executor.shutdown();
    }
}
```

---

## 12.4 InheritableThreadLocal（父子线程值传递）

```java
public class InheritableThreadLocalDemo {

    // 普通 ThreadLocal：子线程无法继承父线程的值
    static ThreadLocal<String> tl = new ThreadLocal<>();

    // InheritableThreadLocal：子线程在创建时继承父线程的值
    static InheritableThreadLocal<String> itl = new InheritableThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        tl.set("父线程ThreadLocal值");
        itl.set("父线程InheritableThreadLocal值");

        Thread child = new Thread(() -> {
            System.out.println("子线程 tl: " + tl.get());    // null（无法继承）
            System.out.println("子线程 itl: " + itl.get());  // 父线程的值

            // 子线程修改不影响父线程
            itl.set("子线程修改后的值");
            System.out.println("子线程修改后 itl: " + itl.get());
        });

        child.start();
        child.join();

        System.out.println("父线程 itl: " + itl.get());  // 不受子线程修改影响

        tl.remove();
        itl.remove();
    }
}
```

**InheritableThreadLocal 的局限性：**
- 值在子线程**创建时**复制，之后父线程修改不会同步给子线程
- **线程池场景下无效**：线程被重用，不会重新继承父线程的值

---

## 12.5 TransmittableThreadLocal（TTL，阿里开源）

`TransmittableThreadLocal`（TTL）是阿里开源的工具类，专门解决**线程池场景下的 ThreadLocal 值传递**问题。

```
TTL 解决的问题：

  问题场景：
    父线程（main/request线程）设置 threadLocal = "user-123"
    提交任务到线程池（线程被复用，不会重新创建）
    线程池中的线程（之前已有其他 threadLocal 值）执行任务
    任务中读取 threadLocal = null 或读到上次任务的值！

  TTL 解决方案：
    在任务提交时（wrap），保存当前父线程的所有 TTL 值
    在任务执行前，将这些值设置到执行线程
    在任务执行后，恢复执行线程原来的 TTL 值

  使用方式：
    1. 引入依赖：com.alibaba:transmittable-thread-local
    2. 将 ThreadLocal 替换为 TransmittableThreadLocal
    3. 任务提交时用 TtlRunnable/TtlCallable 包装（或 TtlExecutors 装饰线程池）

示例代码（需引入 TTL 依赖）：
```

```java
// 使用 TransmittableThreadLocal（需引入 ali ttl 依赖）
// Maven: com.alibaba:transmittable-thread-local:2.14.3

/*
import com.alibaba.ttl.TransmittableThreadLocal;
import com.alibaba.ttl.TtlRunnable;
import com.alibaba.ttl.threadpool.TtlExecutors;

public class TTLDemo {

    // 使用 TransmittableThreadLocal 替代 ThreadLocal
    static TransmittableThreadLocal<String> context = new TransmittableThreadLocal<>();

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executor = TtlExecutors.getTtlExecutorService(
            Executors.newFixedThreadPool(2)  // 用 TTL 装饰线程池
        );

        context.set("request-001");
        System.out.println("主线程: " + context.get());

        // 提交任务（线程池复用线程，但 TTL 保证正确传递上下文）
        executor.submit(() -> {
            System.out.println("子线程读取: " + context.get());  // request-001
        });

        context.set("request-002");
        executor.submit(() -> {
            System.out.println("子线程读取: " + context.get());  // request-002
        });

        Thread.sleep(500);
        executor.shutdown();
        context.remove();
    }
}
*/

// 没有 TTL 依赖时的等价原理演示
public class TTLPrincipleDemo {

    private static final java.util.Map<String, String> threadLocalSnapshot =
        new java.util.concurrent.ConcurrentHashMap<>();

    // 模拟 TTL 的核心：提交任务前快照，执行前恢复，执行后清理
    static Runnable wrapTask(Runnable task) {
        // 提交时快照当前线程的上下文（模拟 TTL 的 capture）
        String capturedContext = Thread.currentThread().getName() + "-context";
        return () -> {
            // 执行前设置上下文（模拟 TTL 的 replay）
            String oldContext = threadLocalSnapshot.put(
                Thread.currentThread().getName(), capturedContext);
            try {
                task.run();
            } finally {
                // 执行后恢复（模拟 TTL 的 restore）
                if (oldContext != null) threadLocalSnapshot.put(
                    Thread.currentThread().getName(), oldContext);
                else threadLocalSnapshot.remove(Thread.currentThread().getName());
            }
        };
    }

    public static void main(String[] args) {
        System.out.println("TTL 原理演示（完整实现需引入 alibaba/transmittable-thread-local）");
    }
}
```

---
# Part 13: 死锁

## 13.1 死锁四个必要条件

死锁是**两个或多个线程互相等待对方持有的资源**，导致永久阻塞的状态。

```
死锁的四个必要条件（缺少任意一个，死锁就不会发生）：

  1. 互斥条件（Mutual Exclusion）
     资源同一时间只能被一个线程持有
     -> 无法避免（锁的本质）

  2. 持有并等待（Hold and Wait）
     线程持有至少一个资源，同时等待其他线程持有的资源
     -> 可预防：一次性申请所有资源，或超时放弃

  3. 不可抢占（No Preemption）
     线程持有的资源不能被强制剥夺，只能主动释放
     -> 可预防：超时机制（tryLock），或等待时释放已持有资源

  4. 循环等待（Circular Wait）
     线程之间形成环形等待链
     T1 等 T2 的锁，T2 等 T3 的锁，T3 等 T1 的锁
     -> 可预防：按固定顺序获取锁（破坏循环）

死锁示意图：
  ┌──────────┐  持有 lockA   ┌──────────┐
  │ Thread1  │ ────────────> │  Lock A  │
  │          │ <──────────── │          │
  └──────────┘  等待 lockB   └──────────┘
       ↓等待                       ↑持有
  ┌──────────┐  持有 lockB   ┌──────────┐
  │ Thread2  │ ────────────> │  Lock B  │
  └──────────┘               └──────────┘
  T1 持有A等待B，T2 持有B等待A -> 死锁！
```

## 13.2 死锁代码示例

```java
public class DeadlockDemo {

    private static final Object lockA = new Object();
    private static final Object lockB = new Object();

    public static void main(String[] args) {
        // 线程1：先获取 lockA，再获取 lockB
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("T1 获得 lockA");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("T1 等待 lockB...");
                synchronized (lockB) {
                    System.out.println("T1 获得 lockB，完成任务");
                }
            }
        }, "T1");

        // 线程2：先获取 lockB，再获取 lockA（顺序相反！）
        Thread t2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("T2 获得 lockB");
                try { Thread.sleep(100); } catch (InterruptedException e) {}
                System.out.println("T2 等待 lockA...");
                synchronized (lockA) {
                    System.out.println("T2 获得 lockA，完成任务");
                }
            }
        }, "T2");

        t1.start();
        t2.start();
        // 程序将永久阻塞在这里！两个线程互相等待
    }
}
```

## 13.3 死锁检测：jstack 分析

```
检测死锁的方法：

1. jstack 命令：
   # 获取 Java 进程 PID
   jps -l
   
   # 生成线程 dump，jstack 会自动检测死锁
   jstack <pid>

   输出示例：
   Found one Java-level deadlock:
   =============================
   "T2":
     waiting to lock monitor 0x00007f...（lockA）
     which is held by "T1"
   "T1":
     waiting to lock monitor 0x00007f...（lockB）
     which is held by "T2"
   
   Java stack information for the threads listed above:
   ===================================================
   "T2":
     at DeadlockDemo.lambda$main$1(DeadlockDemo.java:28)
     - waiting to lock <0x...> (a java.lang.Object)
     - locked <0x...> (a java.lang.Object)
   "T1":
     at DeadlockDemo.lambda$main$0(DeadlockDemo.java:16)
     - waiting to lock <0x...> (a java.lang.Object)
     - locked <0x...> (a java.lang.Object)
   
   Found 1 deadlock.

2. VisualVM / JConsole：图形化工具，有"检测死锁"按钮

3. 代码中检测（ManagementFactory）：
```

```java
import java.lang.management.*;

public class DeadlockDetector {
    public static void detectDeadlock() {
        ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
        long[] deadlockedThreads = mxBean.findDeadlockedThreads();

        if (deadlockedThreads != null) {
            ThreadInfo[] threadInfos = mxBean.getThreadInfo(deadlockedThreads, true, true);
            StringBuilder sb = new StringBuilder("检测到死锁！\n");
            for (ThreadInfo info : threadInfos) {
                sb.append("线程: ").append(info.getThreadName()).append("\n");
                sb.append("等待锁: ").append(info.getLockName()).append("\n");
                sb.append("锁持有者: ").append(info.getLockOwnerName()).append("\n");
            }
            System.err.println(sb);
        } else {
            System.out.println("未检测到死锁");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        // 启动死锁检测定时任务
        java.util.concurrent.Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(DeadlockDetector::detectDeadlock,
                1, 5, java.util.concurrent.TimeUnit.SECONDS);
    }
}
```

## 13.4 死锁预防

```java
import java.util.concurrent.locks.ReentrantLock;
import java.util.concurrent.TimeUnit;

public class DeadlockPreventionDemo {

    private static final ReentrantLock lockA = new ReentrantLock();
    private static final ReentrantLock lockB = new ReentrantLock();

    // 方案1：按固定顺序获取锁（破坏循环等待条件）
    // 所有线程都按 lockA -> lockB 的顺序获取
    static void fixedOrderLocking() {
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {       // 先获取编号小的锁
                synchronized (lockB) {   // 再获取编号大的锁
                    System.out.println("T1 完成（固定顺序）");
                }
            }
        }, "T1");

        Thread t2 = new Thread(() -> {
            synchronized (lockA) {       // 同样先获取 lockA（相同顺序！）
                synchronized (lockB) {
                    System.out.println("T2 完成（固定顺序）");
                }
            }
        }, "T2");

        t1.start(); t2.start();
    }

    // 方案2：使用 tryLock 超时获取（破坏持有并等待条件）
    static void tryLockWithTimeout() {
        Runnable task = () -> {
            String name = Thread.currentThread().getName();
            while (true) {
                boolean gotA = false, gotB = false;
                try {
                    gotA = lockA.tryLock(100, TimeUnit.MILLISECONDS);
                    gotB = lockB.tryLock(100, TimeUnit.MILLISECONDS);
                    if (gotA && gotB) {
                        System.out.println(name + " 获得两把锁，执行任务");
                        break;
                    } else {
                        System.out.println(name + " 未能同时获得两把锁，退让重试");
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                } finally {
                    if (gotB) lockB.unlock();
                    if (gotA) lockA.unlock();
                }
                // 随机等待，避免活锁（两个线程同时退让又同时重试）
                try {
                    Thread.sleep((long)(Math.random() * 50));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    break;
                }
            }
        };

        new Thread(task, "T1").start();
        new Thread(task, "T2").start();
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== 固定顺序获取锁 ===");
        fixedOrderLocking();

        Thread.sleep(500);
        System.out.println("\n=== tryLock 超时机制 ===");
        tryLockWithTimeout();
    }
}
```

## 13.5 活锁与饥饿

```
活锁（Livelock）：
  线程没有阻塞，但因为不断响应对方的动作，无法推进自己的工作
  类比：两人在走廊相遇，都礼让对方，结果都来回侧移，谁也过不去

  解决：引入随机退让时间（如 tryLockWithTimeout 中的随机 sleep）

饥饿（Starvation）：
  某个线程由于优先级低，或竞争资源总被高优先级线程抢占，长时间得不到执行
  
  常见原因：
    - 线程优先级设置不当
    - 非公平锁下某线程一直被插队
    - synchronized 持有时间过长，其他线程长时间等待

  解决：
    - 使用公平锁（ReentrantLock(true)）
    - 合理设置线程优先级
    - 避免长时间持有锁

并发问题汇总：
  死锁（Deadlock）    -> 永久阻塞，线程不运行
  活锁（Livelock）    -> 线程运行但无进展
  饥饿（Starvation）  -> 某线程长时间无法执行
  竞态条件（Race Condition） -> 结果依赖于时序，不可预期
```

---

# Part 14: 完整实战案例

## 14.1 案例1：高并发秒杀（线程池 + 异步下单）

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

/**
 * 简化版秒杀系统核心逻辑
 * 真实场景还需要 Redis 分布式锁，Lua 脚本保证原子性
 */
public class SeckillDemo {

    // 库存（实际项目用 Redis，这里用 AtomicInteger 模拟）
    private final AtomicInteger stock;
    // 异步下单线程池（解耦秒杀逻辑和下单逻辑）
    private final BlockingQueue<Long> orderQueue = new LinkedBlockingQueue<>(10000);
    private final ExecutorService orderExecutor =
        Executors.newFixedThreadPool(4, r -> new Thread(r, "order-worker"));

    public SeckillDemo(int initialStock) {
        this.stock = new AtomicInteger(initialStock);
        startOrderWorker();
    }

    // 秒杀接口（高并发调用）
    public boolean seckill(Long userId) {
        // 1. 扣减库存（CAS，无锁高性能）
        int remaining;
        do {
            remaining = stock.get();
            if (remaining <= 0) {
                return false;  // 库存不足
            }
        } while (!stock.compareAndSet(remaining, remaining - 1));

        // 2. 异步下单（不阻塞秒杀流程）
        boolean queued = orderQueue.offer(userId);
        if (!queued) {
            // 队列满，回补库存
            stock.incrementAndGet();
            return false;
        }

        System.out.printf("[%s] 用户%d 秒杀成功，剩余库存: %d%n",
            Thread.currentThread().getName(), userId, remaining - 1);
        return true;
    }

    // 异步下单处理
    private void startOrderWorker() {
        for (int i = 0; i < 4; i++) {
            orderExecutor.submit(() -> {
                while (!Thread.currentThread().isInterrupted()) {
                    try {
                        Long userId = orderQueue.poll(1, TimeUnit.SECONDS);
                        if (userId != null) {
                            createOrder(userId);
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                }
            });
        }
    }

    private void createOrder(Long userId) {
        // 模拟下单（写数据库等）
        try { Thread.sleep(10); } catch (InterruptedException e) {}
        System.out.printf("[%s] 为用户%d 创建订单成功%n",
            Thread.currentThread().getName(), userId);
    }

    public static void main(String[] args) throws InterruptedException {
        SeckillDemo seckill = new SeckillDemo(100);  // 100件商品

        // 模拟 500 个并发用户抢购
        ExecutorService userPool = Executors.newFixedThreadPool(50);
        AtomicInteger successCount = new AtomicInteger(0);
        CountDownLatch latch = new CountDownLatch(500);

        long start = System.currentTimeMillis();
        for (long i = 1; i <= 500; i++) {
            final long userId = i;
            userPool.submit(() -> {
                try {
                    if (seckill.seckill(userId)) successCount.incrementAndGet();
                } finally {
                    latch.countDown();
                }
            });
        }

        latch.await();
        System.out.printf("%n秒杀结束！成功: %d，耗时: %dms%n",
            successCount.get(), System.currentTimeMillis() - start);

        Thread.sleep(2000);  // 等待异步下单完成
        userPool.shutdown();
        seckill.orderExecutor.shutdown();
    }
}
```

---

## 14.2 案例2：生产者消费者（BlockingQueue）

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class ProducerConsumerBlockingQueue {

    static BlockingQueue<String> queue = new ArrayBlockingQueue<>(50);
    static AtomicBoolean producingDone = new AtomicBoolean(false);

    // 生产者
    static class Producer implements Runnable {
        private final String name;
        private final int count;

        Producer(String name, int count) {
            this.name = name; this.count = count;
        }

        @Override
        public void run() {
            for (int i = 0; i < count; i++) {
                String msg = name + "-msg-" + i;
                try {
                    queue.put(msg);  // 队列满则阻塞
                    System.out.printf("[%s] 生产: %s，队列大小: %d%n",
                        name, msg, queue.size());
                    Thread.sleep((long)(Math.random() * 50));
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
        }
    }

    // 消费者
    static class Consumer implements Runnable {
        private final String name;

        Consumer(String name) { this.name = name; }

        @Override
        public void run() {
            while (!producingDone.get() || !queue.isEmpty()) {
                try {
                    String msg = queue.poll(500, TimeUnit.MILLISECONDS);
                    if (msg != null) {
                        System.out.printf("[%s] 消费: %s，队列大小: %d%n",
                            name, msg, queue.size());
                        Thread.sleep((long)(Math.random() * 100));
                    }
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                    return;
                }
            }
            System.out.println(name + " 退出");
        }
    }

    public static void main(String[] args) throws InterruptedException {
        int producerCount = 3;
        int consumerCount = 2;
        int itemsPerProducer = 10;

        ExecutorService exec = Executors.newFixedThreadPool(producerCount + consumerCount);
        CountDownLatch producerLatch = new CountDownLatch(producerCount);

        // 启动消费者
        for (int i = 0; i < consumerCount; i++)
            exec.submit(new Consumer("Consumer-" + i));

        // 启动生产者
        for (int i = 0; i < producerCount; i++) {
            final int id = i;
            exec.submit(() -> {
                try { new Producer("Producer-" + id, itemsPerProducer).run(); }
                finally { producerLatch.countDown(); }
            });
        }

        // 等待所有生产者完成
        producerLatch.await();
        producingDone.set(true);
        System.out.println("所有生产者完成，等待消费者消费完毕...");

        exec.shutdown();
        exec.awaitTermination(30, TimeUnit.SECONDS);
        System.out.println("全部完成！");
    }
}
```

---

## 14.3 案例3：并发限流（Semaphore + 滑动窗口）

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;
import java.util.ArrayDeque;
import java.util.Deque;

public class RateLimiterDemo {

    // ===== 方案1：Semaphore 简单限流 =====
    static class SemaphoreRateLimiter {
        private final Semaphore semaphore;

        SemaphoreRateLimiter(int maxConcurrent) {
            semaphore = new Semaphore(maxConcurrent, true);
        }

        public boolean tryExecute(Runnable task) {
            if (!semaphore.tryAcquire()) {
                return false;  // 当前并发数已满
            }
            try {
                task.run();
                return true;
            } finally {
                semaphore.release();
            }
        }
    }

    // ===== 方案2：滑动窗口计数限流 =====
    static class SlidingWindowRateLimiter {
        private final int maxRequests;   // 时间窗口内最大请求数
        private final long windowMs;     // 窗口大小（毫秒）
        private final Deque<Long> timestamps = new ArrayDeque<>();

        SlidingWindowRateLimiter(int maxRequests, long windowMs) {
            this.maxRequests = maxRequests;
            this.windowMs = windowMs;
        }

        public synchronized boolean tryAcquire() {
            long now = System.currentTimeMillis();
            long windowStart = now - windowMs;

            // 移除过期的请求记录
            while (!timestamps.isEmpty() && timestamps.peekFirst() <= windowStart) {
                timestamps.pollFirst();
            }

            if (timestamps.size() < maxRequests) {
                timestamps.addLast(now);
                return true;
            }
            return false;  // 超出限流
        }
    }

    public static void main(String[] args) throws InterruptedException {
        System.out.println("=== Semaphore 并发限流（最多3个并发）===");
        SemaphoreRateLimiter limiter1 = new SemaphoreRateLimiter(3);
        ExecutorService exec = Executors.newFixedThreadPool(10);
        AtomicInteger rejected = new AtomicInteger(0);

        for (int i = 0; i < 10; i++) {
            final int id = i;
            exec.submit(() -> {
                boolean ok = limiter1.tryExecute(() -> {
                    System.out.println("处理请求 " + id);
                    try { Thread.sleep(200); } catch (InterruptedException e) {}
                });
                if (!ok) {
                    System.out.println("请求 " + id + " 被限流");
                    rejected.incrementAndGet();
                }
            });
        }

        exec.shutdown();
        exec.awaitTermination(5, TimeUnit.SECONDS);
        System.out.println("被限流请求数: " + rejected.get());

        System.out.println("\n=== 滑动窗口限流（每秒最多5个）===");
        SlidingWindowRateLimiter limiter2 = new SlidingWindowRateLimiter(5, 1000);
        for (int i = 0; i < 15; i++) {
            boolean ok = limiter2.tryAcquire();
            System.out.println("请求 " + i + ": " + (ok ? "通过" : "限流"));
            Thread.sleep(100);  // 每100ms一个请求，1秒内10个
        }
    }
}
```

---

## 14.4 案例4：异步任务编排（CompletableFuture 商品详情页）

（见 Part 11 的 ProductDetailPageDemo，此处不重复）

---

## 14.5 案例5：线程池动态参数调整

```java
import java.util.concurrent.*;
import java.util.concurrent.atomic.*;

public class DynamicThreadPoolDemo {

    static class DynamicThreadPool {
        private final ThreadPoolExecutor executor;
        private volatile int currentCore;
        private volatile int currentMax;

        DynamicThreadPool(int coreSize, int maxSize, int queueSize) {
            this.currentCore = coreSize;
            this.currentMax = maxSize;
            executor = new ThreadPoolExecutor(
                coreSize, maxSize, 60, TimeUnit.SECONDS,
                new ResizableCapacityBlockingQueue<>(queueSize),
                r -> new Thread(r, "dynamic-pool-" + r.hashCode()),
                new ThreadPoolExecutor.AbortPolicy()
            );
        }

        // 动态调整参数（可配合配置中心 Nacos/Apollo 使用）
        public synchronized void resize(int newCore, int newMax, int newQueueCapacity) {
            System.out.printf("调整前: core=%d max=%d queue=%d%n",
                currentCore, currentMax,
                ((ResizableCapacityBlockingQueue<?>)executor.getQueue()).getCapacity());

            // 注意调整顺序：避免 core > max 的非法状态
            if (newCore <= currentMax) {
                executor.setCorePoolSize(newCore);
                executor.setMaximumPoolSize(newMax);
            } else {
                executor.setMaximumPoolSize(newMax);
                executor.setCorePoolSize(newCore);
            }
            ((ResizableCapacityBlockingQueue<?>)executor.getQueue()).setCapacity(newQueueCapacity);

            currentCore = newCore;
            currentMax = newMax;

            System.out.printf("调整后: core=%d max=%d queue=%d%n",
                currentCore, currentMax, newQueueCapacity);
        }

        public void printStats() {
            System.out.printf("[Stats] core=%d max=%d active=%d pool=%d queue=%d completed=%d%n",
                executor.getCorePoolSize(), executor.getMaximumPoolSize(),
                executor.getActiveCount(), executor.getPoolSize(),
                executor.getQueue().size(), executor.getCompletedTaskCount());
        }

        public ExecutorService getExecutor() { return executor; }
        public void shutdown() { executor.shutdown(); }
    }

    // 可动态调整容量的阻塞队列
    static class ResizableCapacityBlockingQueue<E> extends LinkedBlockingQueue<E> {
        private volatile int capacity;

        ResizableCapacityBlockingQueue(int capacity) {
            super(capacity);
            this.capacity = capacity;
        }

        public int getCapacity() { return capacity; }

        public void setCapacity(int capacity) {
            this.capacity = capacity;
        }

        @Override
        public boolean offer(E e) {
            return size() < capacity && super.offer(e);
        }
    }

    public static void main(String[] args) throws InterruptedException {
        DynamicThreadPool pool = new DynamicThreadPool(2, 4, 20);

        // 提交初始任务
        for (int i = 0; i < 5; i++) {
            final int id = i;
            pool.getExecutor().execute(() -> {
                System.out.println(Thread.currentThread().getName() + " 处理任务-" + id);
                try { Thread.sleep(1000); } catch (InterruptedException e) {}
            });
        }

        Thread.sleep(500);
        pool.printStats();

        // 模拟业务高峰，动态扩容
        System.out.println("=== 业务高峰，动态扩容 ===");
        pool.resize(4, 10, 100);
        pool.printStats();

        Thread.sleep(2000);

        // 模拟业务低谷，动态缩容
        System.out.println("=== 业务低谷，动态缩容 ===");
        pool.resize(2, 4, 20);
        pool.printStats();

        pool.shutdown();
    }
}
```

---

# Part 15: 常见面试题 FAQ

## Q1：synchronized 和 ReentrantLock 的区别？

**答：**
- `synchronized` 是 JVM 内置关键字，自动加锁解锁，不可中断等待，不支持公平锁，无法查询锁状态
- `ReentrantLock` 是 Java API，需要手动 `lock()/unlock()`，支持可中断（`lockInterruptibly`）、超时（`tryLock`）、公平锁、多条件变量（`newCondition`）
- JDK 6+ 两者性能相近，简单场景用 `synchronized`，需要高级功能用 `ReentrantLock`

## Q2：volatile 能保证原子性吗？

**答：** 不能。volatile 保证**可见性**（通过 MESI 协议）和**有序性**（通过内存屏障），但不保证原子性。`count++` 是读-改-写三步操作，volatile 无法保证这三步的原子性。需要原子性应使用 `AtomicInteger` 或 `synchronized`。

## Q3：happens-before 是什么？有哪些规则？

**答：** `happens-before` 是 JMM 定义的**内存可见性保证**规则。A happens-before B 意味着 A 的操作结果对 B 可见，且 A 在 B 之前发生。8 条规则：程序顺序规则、监视器锁规则、volatile 变量规则、线程启动规则、线程终止规则、线程中断规则、对象终结规则、传递性规则。

## Q4：CAS 是什么？有什么缺点？

**答：** CAS（Compare-And-Swap）是 CPU 原子指令（x86: cmpxchg），比较并交换。缺点：1）**ABA 问题**（用 `AtomicStampedReference` 解决）；2）**自旋开销**（高并发下失败重试，CPU 空转，用 `LongAdder` 解决）；3）**只能保证单个变量原子性**（多变量需用 `synchronized` 或 `AtomicReference`）。

## Q5：AQS 的原理是什么？

**答：** AQS（`AbstractQueuedSynchronizer`）是 JUC 的基石，基于两个核心：1）`volatile int state`（同步状态）；2）CLH 变体双向等待队列（FIFO）。独占锁获取：`tryAcquire` 失败则通过 `addWaiter` 加入队列，`acquireQueued` 自旋等待并 `LockSupport.park` 挂起。释放时 `unparkSuccessor` 唤醒后继节点。

## Q6：线程池的 7 个参数是什么？执行流程是怎样的？

**答：** `corePoolSize`（核心线程数）、`maximumPoolSize`（最大线程数）、`keepAliveTime`（非核心线程存活时间）、`unit`（时间单位）、`workQueue`（工作队列）、`threadFactory`（线程工厂）、`handler`（拒绝策略）。

执行流程：① 线程数 < core -> 创建核心线程 → ② 进入工作队列 → ③ 队列满且线程数 < max -> 创建非核心线程 → ④ 执行拒绝策略。

## Q7：为什么不推荐使用 Executors 工厂方法创建线程池？

**答：** `newFixedThreadPool` 和 `newSingleThreadExecutor` 使用无界 `LinkedBlockingQueue`，任务堆积可导致 OOM；`newCachedThreadPool` 最大线程数为 `Integer.MAX_VALUE`，短时大量任务可创建大量线程导致 OOM。推荐使用 `ThreadPoolExecutor` 手动指定有界队列和合理的最大线程数。

## Q8：死锁的四个必要条件是什么？如何预防？

**答：** 互斥、持有并等待、不可抢占、循环等待。预防方法：① 固定锁的获取顺序（破坏循环等待）；② 使用 `tryLock` 超时机制（破坏持有并等待）；③ 一次性申请所有资源。

## Q9：ConcurrentHashMap 在 JDK 8 中如何实现线程安全？

**答：** JDK 8 使用数组+链表+红黑树结构，线程安全通过：① 桶为空时用 **CAS** 直接插入（无锁）；② 桶非空时对**桶的头节点** `synchronized` 加锁（分段锁，粒度精确到单个桶）；③ 扩容时多线程**协作迁移**；④ 计数用类似 **LongAdder** 的 `baseCount + CounterCell[]` 机制。

## Q10：ThreadLocal 内存泄漏的原因是什么？如何解决？

**答：** `ThreadLocalMap` 的 `Entry.key` 是 `ThreadLocal` 的**弱引用**，`Entry.value` 是**强引用**。当 `ThreadLocal` 引用被 GC 后，`key = null`，但 `value` 仍然被强引用，无法回收。在线程池中线程长期存活，导致 `value` 无法 GC -> 内存泄漏。

解决：使用完毕后在 `finally` 块中调用 `threadLocal.remove()`。

## Q11：CountDownLatch 和 CyclicBarrier 的区别？

**答：** `CountDownLatch` 是一次性的，用于"等待 N 个操作完成"；`CyclicBarrier` 可重复使用，用于"N 个线程互相等待到达同一点"。`CountDownLatch.countDown()` 可以由任意线程调用；`CyclicBarrier.await()` 必须由屏障参与者调用。

## Q12：AtomicInteger 和 LongAdder 如何选择？

**答：** 低并发且需要实时精确值或 CAS 操作，用 `AtomicInteger/AtomicLong`；高并发计数场景（如统计请求量），用 `LongAdder`（分段 Cell，减少竞争，高并发可快 3~10 倍），但 `sum()` 不是强一致性的实时值。

## Q13：什么是指令重排序？volatile 如何防止？

**答：** 指令重排序是编译器和 CPU 在不改变单线程语义的前提下，调整指令执行顺序的优化。多线程下可能导致可见性问题。`volatile` 写操作前后插入 `StoreStore` 和 `StoreLoad` 内存屏障，volatile 读操作后插入 `LoadLoad` 和 `LoadStore` 屏障，禁止相关重排序。

## Q14：StampedLock 和 ReadWriteLock 有什么区别？

**答：** `StampedLock` 引入乐观读（`tryOptimisticRead`，无锁读取后 validate 验证），在读多写少时性能远超 `ReadWriteLock`。`StampedLock` 不可重入，不支持条件变量，中断处理需注意 CPU 飙升问题。`ReentrantReadWriteLock` 可重入，支持条件变量，更安全易用。

## Q15：ForkJoinPool 和 ThreadPoolExecutor 的区别？

**答：** `ThreadPoolExecutor` 是通用线程池，任务平等竞争一个队列。`ForkJoinPool` 是为递归分治算法设计，每个工作线程有**自己的双端队列**（work-stealing），空闲线程从其他线程队列尾部窃取任务，减少竞争，特别适合递归任务（ForkJoinTask）和并行流操作。

## Q16：如何安全地停止一个线程？

**答：** 推荐方式：① 使用 `interrupt()` 设置中断标志，线程内部检查 `Thread.currentThread().isInterrupted()` 或处理 `InterruptedException`（捕获后必须重新设置中断标志 `Thread.currentThread().interrupt()`）；② 使用 `volatile boolean` 标志位作为退出条件；③ 不要使用已废弃的 `Thread.stop()`（可能导致对象状态不一致）。

## Q17：什么是伪共享（False Sharing）？如何解决？

**答：** CPU 缓存以**缓存行**（Cache Line，通常 64 字节）为单位操作。如果两个变量位于同一缓存行，一个线程修改其中一个，会导致另一个线程的缓存行失效（MESI 协议），造成不必要的缓存同步，称为伪共享。解决：① 填充缓冲字节（`@sun.misc.Contended`）隔离变量；② Java 8 `@Contended` 注解；③ LongAdder 的 Cell 类就用了 `@Contended`。

## Q18：线程池的 submit 和 execute 有什么区别？

**答：** `execute(Runnable)` 没有返回值，异常会直接被线程的 `UncaughtExceptionHandler` 处理（不设置则打印后线程退出）；`submit(Callable/Runnable)` 返回 `Future`，异常被包装在 `Future.get()` 的 `ExecutionException` 中（不调用 get 则异常会被吞掉！）。生产中推荐 `submit + Future.get()` 或给线程池设置 `UncaughtExceptionHandler`。

## Q19：CompletableFuture 的 thenApply 和 thenCompose 有什么区别？

**答：** `thenApply(Function<T, R>)` 将 `T` 同步转换为 `R`，得到 `CompletableFuture<R>`；`thenCompose(Function<T, CompletableFuture<R>>)` 将 `T` 转换为 `CompletableFuture<R>` 并**扁平化**，避免嵌套。类比 Stream 的 `map` vs `flatMap`。当后续操作本身是异步的（返回 CF），应用 `thenCompose` 而非 `thenApply`。

## Q20：synchronized 的锁升级过程是什么？

**答：** JDK 6+ synchronized 引入锁升级优化，从低到高：**无锁 -> 偏向锁 -> 轻量级锁 -> 重量级锁**，只能升级不能降级（GC 特殊情况除外）。偏向锁：同一线程重复加锁零开销（Mark Word 存储线程 ID）；轻量级锁：多线程交替访问，CAS 操作栈帧 Lock Record；重量级锁：真正竞争，操作系统 mutex，线程挂起。JDK 21 已移除偏向锁。

## Q21：什么是 AQS 的 CLH 队列？为什么用双向链表？

**答：** CLH 是一种基于自旋锁的有序公平链式队列（Craig, Landin, Hagersten）。AQS 使用它的变体：节点 park（阻塞）而非自旋，节点内存储线程引用。使用**双向链表**的原因：① 取消节点时需要从后向前找有效前驱（单向链表不能直接访问前驱）；② 释放锁时 `unparkSuccessor` 可能需要从 tail 向前找（因为 next 指针不是原子设置的）。

## Q22：Java 内存模型中的 happens-before 和时间上的先发生是同一概念吗？

**答：** 不同。happens-before 是**对内存操作可见性的规范承诺**，而不是时间上的先后。即使 A 在时间上先于 B 发生，如果没有 happens-before 关系，B 也未必能看到 A 的结果。反之，如果有 happens-before 关系（如 volatile），即使 A 的物理执行时间很长，也能保证 B 看到 A 最新的值。

---

## 附录：JUC 常用类速查

```
锁：
  synchronized          内置锁，最简单
  ReentrantLock         可重入独占锁，功能最丰富
  ReentrantReadWriteLock 读写分离锁
  StampedLock           乐观读锁，最高性能（Java 8）

原子类：
  AtomicInteger/Long/Boolean     基础原子类
  AtomicReference                引用原子类
  AtomicStampedReference         带版本号，解决 ABA
  LongAdder/LongAccumulator      高并发累加（推荐）

同步工具：
  CountDownLatch         倒计时，一次性
  CyclicBarrier          循环屏障，可重用
  Semaphore              信号量，控制并发数
  Exchanger              两线程数据交换
  Phaser                 多阶段同步（Java 7）

并发集合：
  ConcurrentHashMap      高性能并发 Map
  CopyOnWriteArrayList   写时复制 List
  ConcurrentLinkedQueue  无锁并发队列
  ArrayBlockingQueue     有界阻塞队列
  LinkedBlockingQueue    链表阻塞队列
  PriorityBlockingQueue  优先级阻塞队列
  DelayQueue             延迟队列
  SynchronousQueue       无缓冲直传队列

线程池：
  ThreadPoolExecutor     通用线程池（推荐直接使用）
  ScheduledThreadPoolExecutor 定时/延迟执行
  ForkJoinPool           分治并行计算
  CompletableFuture      异步编程（Java 8）

JVM 工具：
  jps                   查看 Java 进程
  jstack <pid>          线程 dump，检测死锁
  jmap -heap <pid>      堆内存分析
  jstat -gc <pid>       GC 统计
  VisualVM / JConsole   图形化监控
```

---

> **学习建议：**
> 1. 先理解 JMM（Part 2），这是所有并发问题的根源
> 2. 深入掌握 AQS（Part 4），JUC 大半组件的基础
> 3. 线程池（Part 10）是生产最常用，必须熟练配置
> 4. CompletableFuture（Part 11）是异步编程的利器
> 5. 通过 jstack 分析真实死锁，加深理解
> 6. 建议阅读 JDK 源码：`AbstractQueuedSynchronizer`、`ThreadPoolExecutor`、`ConcurrentHashMap`
