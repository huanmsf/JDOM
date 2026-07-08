# Netty 详解：从入门到精通

> 本文档全面覆盖 Netty 核心知识点，适合面试备战与生产实践参考。

---

## 目录

1. [网络编程基础回顾](#1-网络编程基础回顾)
2. [Netty 整体架构](#2-netty-整体架构)
3. [核心组件深度解析](#3-核心组件深度解析)
4. [编解码器（Codec）](#4-编解码器codec)
5. [常用 Handler](#5-常用-handler)
6. [Netty 实战案例](#6-netty-实战案例)
7. [Netty 高级特性](#7-netty-高级特性)
8. [Netty 在主流框架中的应用](#8-netty-在主流框架中的应用)
9. [性能调优](#9-性能调优)
10. [常见问题排查](#10-常见问题排查)
11. [常见面试题 FAQ](#11-常见面试题-faq)

---

## 1. 网络编程基础回顾

### 1.1 BIO（阻塞 I/O）

BIO（Blocking I/O）是 Java 最早提供的网络编程模型，每个客户端连接都需要一个独立线程处理。

```
BIO 模型示意图：

  客户端1 ──► 线程1 ──► Socket ──► 业务处理
  客户端2 ──► 线程2 ──► Socket ──► 业务处理
  客户端3 ──► 线程3 ──► Socket ──► 业务处理
  ...
  客户端N ──► 线程N ──► Socket ──► 业务处理

  问题：线程数 = 连接数，高并发下内存耗尽
```

**BIO 服务端示例代码：**

```java
public class BioServer {
    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8080);
        System.out.println("BIO 服务端启动，监听端口 8080");

        while (true) {
            // accept() 阻塞等待客户端连接
            Socket socket = serverSocket.accept();
            System.out.println("新客户端连接：" + socket.getRemoteSocketAddress());

            // 每个连接创建一个新线程（生产环境应使用线程池）
            new Thread(() -> handleClient(socket)).start();
        }
    }

    private static void handleClient(Socket socket) {
        try (BufferedReader reader = new BufferedReader(
                new InputStreamReader(socket.getInputStream()));
             PrintWriter writer = new PrintWriter(socket.getOutputStream(), true)) {

            String line;
            while ((line = reader.readLine()) != null) {
                System.out.println("收到消息：" + line);
                writer.println("Echo: " + line);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

**BIO 的缺点：**
- 每个连接占用一个线程，线程资源有限
- 线程切换开销大
- 连接数受限于系统线程数上限
- 适用场景：连接数少、并发度低的场景（如数据库连接）

---

### 1.2 NIO（非阻塞 I/O）

NIO（Non-blocking I/O）于 Java 1.4 引入，核心组件：`Channel`、`Buffer`、`Selector`。

```
NIO 模型示意图：

  客户端1 ──┐
  客户端2 ──┤
  客户端3 ──┼──► Selector ──► 单线程/少量线程处理就绪事件
  ...       │      │
  客户端N ──┘      ├── OP_ACCEPT  → 接受新连接
                   ├── OP_READ    → 读数据
                   ├── OP_WRITE   → 写数据
                   └── OP_CONNECT → 完成连接

  优势：一个线程管理多个连接
```

**NIO 三大核心组件：**

| 组件 | 说明 |
|------|------|
| Channel | 双向数据通道，替代 Stream |
| Buffer | 数据缓冲区，读写均通过 Buffer |
| Selector | 多路复用选择器，监听多个 Channel 事件 |

**NIO 服务端示例代码：**

```java
public class NioServer {
    public static void main(String[] args) throws IOException {
        Selector selector = Selector.open();
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.bind(new InetSocketAddress(8080));
        serverChannel.configureBlocking(false); // 设置非阻塞
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);

        System.out.println("NIO 服务端启动，监听端口 8080");

        while (true) {
            selector.select(); // 阻塞直到有事件就绪
            Set<SelectionKey> selectedKeys = selector.selectedKeys();
            Iterator<SelectionKey> iter = selectedKeys.iterator();

            while (iter.hasNext()) {
                SelectionKey key = iter.next();
                iter.remove();

                if (key.isAcceptable()) {
                    // 处理新连接
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    SocketChannel client = server.accept();
                    client.configureBlocking(false);
                    client.register(selector, SelectionKey.OP_READ);
                    System.out.println("新连接：" + client.getRemoteAddress());

                } else if (key.isReadable()) {
                    // 处理读事件
                    SocketChannel client = (SocketChannel) key.channel();
                    ByteBuffer buffer = ByteBuffer.allocate(1024);
                    int bytesRead = client.read(buffer);
                    if (bytesRead == -1) {
                        client.close();
                    } else {
                        buffer.flip();
                        byte[] data = new byte[buffer.remaining()];
                        buffer.get(data);
                        System.out.println("收到：" + new String(data));
                        // 回写数据
                        ByteBuffer response = ByteBuffer.wrap(("Echo: " + new String(data)).getBytes());
                        client.write(response);
                    }
                }
            }
        }
    }
}
```

---

### 1.3 AIO（异步 I/O）

AIO（Asynchronous I/O）在 Java 7 引入，基于事件和回调机制，真正异步，无需 Selector 轮询。

```
AIO 模型示意图：

  应用程序                    操作系统内核
      │                           │
      │── 发起异步读请求 ─────────►│
      │                           │ 内核等待数据
      │   （继续做其他事）         │
      │                           │ 数据就绪，内核拷贝到用户空间
      │◄── 通知完成（回调/Future）─│
      │                           │
```

```java
public class AioServer {
    public static void main(String[] args) throws Exception {
        AsynchronousServerSocketChannel server =
            AsynchronousServerSocketChannel.open()
                .bind(new InetSocketAddress(8080));

        server.accept(null, new CompletionHandler<AsynchronousSocketChannel, Void>() {
            @Override
            public void completed(AsynchronousSocketChannel client, Void attachment) {
                // 继续接受下一个连接
                server.accept(null, this);

                ByteBuffer buffer = ByteBuffer.allocate(1024);
                client.read(buffer, buffer, new CompletionHandler<Integer, ByteBuffer>() {
                    @Override
                    public void completed(Integer result, ByteBuffer buf) {
                        buf.flip();
                        byte[] data = new byte[buf.remaining()];
                        buf.get(data);
                        System.out.println("AIO 收到：" + new String(data));
                    }

                    @Override
                    public void failed(Throwable exc, ByteBuffer buf) {
                        exc.printStackTrace();
                    }
                });
            }

            @Override
            public void failed(Throwable exc, Void attachment) {
                exc.printStackTrace();
            }
        });

        Thread.sleep(Integer.MAX_VALUE); // 保持主线程存活
    }
}
```

---

### 1.4 I/O 多路复用：select / poll / epoll 对比

```
┌─────────────┬──────────────────────┬──────────────────────┬──────────────────────┐
│    特性      │       select         │        poll          │        epoll         │
├─────────────┼──────────────────────┼──────────────────────┼──────────────────────┤
│ 数据结构     │ fd_set（位图）        │ pollfd 数组          │ 红黑树 + 就绪链表    │
│ 最大连接数   │ 1024（FD_SETSIZE）   │ 无限制               │ 无限制               │
│ fd 拷贝      │ 每次调用都拷贝       │ 每次调用都拷贝       │ 只需注册时拷贝一次   │
│ 事件检测     │ 遍历所有 fd          │ 遍历所有 fd          │ 只返回就绪 fd        │
│ 时间复杂度   │ O(n)                 │ O(n)                 │ O(1)                 │
│ 跨平台       │ 支持                 │ 支持                 │ 仅 Linux             │
│ 适用场景     │ 连接数少             │ 连接数中等           │ 高并发，海量连接     │
└─────────────┴──────────────────────┴──────────────────────┴──────────────────────┘
```

**epoll 工作原理：**

```
epoll 三个核心系统调用：

epoll_create(size)
  └── 创建 epoll 实例，返回 epfd（内核维护红黑树）

epoll_ctl(epfd, op, fd, event)
  └── 向红黑树中增/删/改监听的 fd

epoll_wait(epfd, events, maxevents, timeout)
  └── 阻塞等待，就绪事件放入 events 数组返回

epoll 两种工作模式：
  ├── LT（水平触发，默认）：只要 fd 可读/可写，每次 epoll_wait 都通知
  └── ET（边缘触发）：  只在状态变化时通知一次，需一次性读完所有数据
```

---

### 1.5 零拷贝（Zero-Copy）

传统 I/O 数据流转过程：

```
传统方式（4次拷贝，4次上下文切换）：

  磁盘 ──DMA拷贝──► 内核缓冲区 ──CPU拷贝──► 用户缓冲区
                                              │
                                           应用处理
                                              │
       Socket发送缓冲区 ◄──CPU拷贝──── 用户缓冲区
              │
          网卡──DMA拷贝──► 网络
```

```
sendfile 零拷贝（2次拷贝，2次上下文切换）：

  磁盘 ──DMA拷贝──► 内核缓冲区 ──DMA拷贝──► Socket缓冲区 ──► 网络

  用户空间完全不参与数据拷贝！
```

**Netty 中的零拷贝体现：**
1. `CompositeByteBuf`：逻辑合并多个 ByteBuf，无需内存拷贝
2. `FileRegion`：使用 `transferTo()` 实现文件到网络的零拷贝
3. `slice()`：切片不拷贝数据
4. `DirectByteBuf`：堆外内存，减少 JVM 到内核的拷贝

---

## 2. Netty 整体架构

### 2.1 架构总览

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Netty 架构总览                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                      Protocol Support                         │  │
│  │  HTTP  WebSocket  SSL/TLS  Protobuf  Thrift  自定义协议  ...  │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     Transport Services                        │  │
│  │   NIO      OIO      Local      Embedded      epoll(Linux)    │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──────────────────────┐   ┌──────────────────────────────────┐   │
│  │     Core Components  │   │         Utilities                │   │
│  │  ChannelPipeline     │   │  Bootstrap / ServerBootstrap     │   │
│  │  ChannelHandler      │   │  ByteBuf / ByteBufAllocator      │   │
│  │  EventLoop           │   │  Future / Promise                │   │
│  │  EventLoopGroup      │   │  ChannelFuture                   │   │
│  └──────────────────────┘   └──────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Reactor 模式三种模型

#### 2.2.1 单 Reactor 单线程

```
┌────────────────────────────────────────────┐
│            单 Reactor 单线程                │
│                                            │
│  Client ──► Reactor ──► Acceptor ──► Handler│
│              (Select)                      │
│                                            │
│  所有操作在同一个线程中完成                  │
│  缺点：无法利用多核，Handler 处理慢影响全局  │
└────────────────────────────────────────────┘
```

#### 2.2.2 单 Reactor 多线程

```
┌────────────────────────────────────────────────────────┐
│               单 Reactor 多线程                         │
│                                                        │
│  Client ──► Reactor ──► Acceptor                       │
│              (Select)      │                           │
│                            ▼                           │
│                    ┌───────────────┐                   │
│                    │  Thread Pool  │                   │
│                    │  Handler1     │                   │
│                    │  Handler2     │                   │
│                    │  Handler3     │                   │
│                    └───────────────┘                   │
│                                                        │
│  I/O 在 Reactor 线程，业务处理在线程池                   │
│  缺点：Reactor 单点，高并发时可能成为瓶颈               │
└────────────────────────────────────────────────────────┘
```

#### 2.2.3 主从 Reactor 多线程（Netty 默认）

```
┌──────────────────────────────────────────────────────────────────┐
│                  主从 Reactor 多线程（Netty 默认）                 │
│                                                                  │
│  Client ──► MainReactor ──► Acceptor ──► SubReactor1 ──► Handler│
│             (BossGroup)                  SubReactor2 ──► Handler│
│             处理 ACCEPT                  SubReactor3 ──► Handler│
│                                         (WorkerGroup)           │
│                                         处理 READ/WRITE          │
│                                                                  │
│  BossGroup：专门处理连接建立（通常1个线程）                        │
│  WorkerGroup：处理 I/O 读写（通常 CPU*2 个线程）                  │
│  业务逻辑：在 ChannelHandler 中，可投递到业务线程池               │
└──────────────────────────────────────────────────────────────────┘
```

**Netty 中配置主从 Reactor：**

```java
// BossGroup 处理连接，WorkerGroup 处理 I/O
EventLoopGroup bossGroup = new NioEventLoopGroup(1);
EventLoopGroup workerGroup = new NioEventLoopGroup(); // 默认 CPU*2

ServerBootstrap bootstrap = new ServerBootstrap();
bootstrap.group(bossGroup, workerGroup)
    .channel(NioServerSocketChannel.class)
    .childHandler(new ChannelInitializer<SocketChannel>() {
        @Override
        protected void initChannel(SocketChannel ch) {
            ch.pipeline().addLast(new MyHandler());
        }
    });
```

---

### 2.3 Netty 线程模型详解

```
Netty 线程模型：

  ┌─────────────────────────────────────────────────────┐
  │  NioEventLoopGroup (BossGroup)                      │
  │  ┌─────────────────────────────────────────────┐    │
  │  │  NioEventLoop-0                             │    │
  │  │  ┌──────────────────────────────────────┐  │    │
  │  │  │  Thread: 运行 Selector.select()      │  │    │
  │  │  │  处理 OP_ACCEPT 事件                  │  │    │
  │  │  │  将新 Channel 注册到 WorkerGroup      │  │    │
  │  │  └──────────────────────────────────────┘  │    │
  │  └─────────────────────────────────────────────┘    │
  └─────────────────────────────────────────────────────┘

  ┌─────────────────────────────────────────────────────┐
  │  NioEventLoopGroup (WorkerGroup)                    │
  │  ┌──────────────┐ ┌──────────────┐ ┌─────────────┐ │
  │  │NioEventLoop-0│ │NioEventLoop-1│ │NioEventLoop2│ │
  │  │  Thread      │ │  Thread      │ │  Thread     │ │
  │  │  Selector    │ │  Selector    │ │  Selector   │ │
  │  │  TaskQueue   │ │  TaskQueue   │ │  TaskQueue  │ │
  │  │  Channel1    │ │  Channel3    │ │  Channel5   │ │
  │  │  Channel2    │ │  Channel4    │ │  Channel6   │ │
  │  └──────────────┘ └──────────────┘ └─────────────┘ │
  └─────────────────────────────────────────────────────┘

  每个 NioEventLoop：
  ① 运行 Selector，处理注册在其上的所有 Channel 的 I/O 事件
  ② 处理 TaskQueue 中的任务（如 schedule 任务）
  ③ 单线程，无锁化设计
```

**EventLoop 任务执行逻辑（伪代码）：**

```java
// NioEventLoop.run() 核心逻辑（简化版）
protected void run() {
    for (;;) {
        // 1. 执行 select（带超时）
        int selectedKeys = selector.select(timeoutMs);

        // 2. 处理就绪的 I/O 事件
        if (selectedKeys > 0) {
            processSelectedKeys();
        }

        // 3. 执行任务队列中的任务
        runAllTasks(ioRatio);

        // 4. 检查是否需要关闭
        if (isShuttingDown()) {
            closeAll();
            if (confirmShutdown()) return;
        }
    }
}
```

---

### 2.4 Bootstrap 启动流程

```
服务端 ServerBootstrap 启动流程：

  ServerBootstrap.bind(port)
        │
        ▼
  initAndRegister()
  ├── channelFactory.newChannel()  → 创建 NioServerSocketChannel
  ├── init(channel)                → 设置 options/attrs，添加 ServerBootstrapAcceptor
  └── config().group().register(channel) → 将 channel 注册到 BossGroup 的 EventLoop
        │
        ▼
  doBind0()
  └── channel.bind(localAddress)  → 调用 JDK 底层 bind + listen
        │
        ▼
  ChannelFuture 就绪，服务启动完成
```

```java
// 完整的服务端启动示例
public class NettyServer {
    public static void main(String[] args) throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 128)         // 服务端
             .childOption(ChannelOption.SO_KEEPALIVE, true) // 子 Channel
             .childOption(ChannelOption.TCP_NODELAY, true)
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new LoggingHandler(LogLevel.INFO));
                     p.addLast(new MyBusinessHandler());
                 }
             });

            // 绑定并同步等待
            ChannelFuture f = b.bind(8080).sync();
            System.out.println("Netty 服务端启动成功，监听端口 8080");

            // 等待服务端 Channel 关闭
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

---

---

## 3. 核心组件深度解析

### 3.1 Channel

Channel 是 Netty 对网络连接的抽象，封装了底层 Socket 的操作。

```
Channel 继承体系：

  Channel (interface)
    └── AbstractChannel
          ├── AbstractNioChannel
          │     ├── AbstractNioByteChannel
          │     │     └── NioSocketChannel         （客户端 TCP）
          │     └── AbstractNioMessageChannel
          │           └── NioServerSocketChannel   （服务端 TCP）
          ├── NioDatagramChannel                   （UDP）
          └── EpollSocketChannel                   （Linux epoll）
```

**Channel 重要方法：**

```java
Channel channel = ctx.channel();

// 获取状态
channel.isOpen();       // 是否打开
channel.isActive();     // 是否激活（连接建立）
channel.isWritable();   // 是否可写（高水位控制）

// 获取组件
channel.pipeline();     // 获取 ChannelPipeline
channel.eventLoop();    // 获取绑定的 EventLoop
channel.config();       // 获取 Channel 配置
channel.remoteAddress();// 远端地址

// I/O 操作（返回 ChannelFuture）
channel.read();         // 触发读
channel.write(msg);     // 写到缓冲区（不立即发送）
channel.flush();        // 将缓冲区数据发送出去
channel.writeAndFlush(msg); // write + flush
channel.close();        // 关闭连接
channel.bind(addr);     // 绑定地址
channel.connect(addr);  // 发起连接
```

**ChannelFuture 异步操作：**

```java
// Netty 所有 I/O 操作都是异步的，返回 ChannelFuture
ChannelFuture future = channel.writeAndFlush("Hello Netty");

// 方式1：添加监听器（推荐）
future.addListener(f -> {
    if (f.isSuccess()) {
        System.out.println("发送成功");
    } else {
        System.out.println("发送失败：" + f.cause());
    }
});

// 方式2：同步等待（注意：不要在 EventLoop 线程中调用 sync()）
future.sync(); // 阻塞等待完成
```

---

### 3.2 EventLoop 与 EventLoopGroup

```
EventLoop 核心职责：

  ┌────────────────────────────────────────────────────┐
  │                   NioEventLoop                     │
  │                                                    │
  │  ┌──────────┐  ┌──────────────┐  ┌─────────────┐  │
  │  │ Selector │  │  TaskQueue   │  │ScheduleQueue│  │
  │  │          │  │  (普通任务)   │  │ (定时任务)   │  │
  │  └──────────┘  └──────────────┘  └─────────────┘  │
  │       │               │                │           │
  │       └───────────────┴────────────────┘           │
  │                       │                            │
  │              单线程顺序执行                          │
  └────────────────────────────────────────────────────┘
```

**向 EventLoop 提交任务：**

```java
// 在 ChannelHandler 中向 EventLoop 提交任务
public class MyHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        EventLoop eventLoop = ctx.channel().eventLoop();

        // 1. 提交普通任务（在 EventLoop 线程中执行）
        eventLoop.execute(() -> {
            System.out.println("在 EventLoop 线程执行：" + Thread.currentThread().getName());
            ctx.writeAndFlush(msg);
        });

        // 2. 提交延迟任务
        eventLoop.schedule(() -> {
            System.out.println("5秒后执行");
        }, 5, TimeUnit.SECONDS);

        // 3. 判断当前是否在 EventLoop 线程
        if (eventLoop.inEventLoop()) {
            // 直接执行，不需要提交
            ctx.writeAndFlush(msg);
        } else {
            // 提交到 EventLoop 线程执行
            eventLoop.execute(() -> ctx.writeAndFlush(msg));
        }
    }
}
```

**EventLoopGroup 线程分配：**

```java
// EventLoopGroup 通过轮询将 Channel 分配给 EventLoop
// 源码：DefaultEventExecutorChooserFactory
// 如果线程数是2的幂次，使用位运算优化（idx & (len-1)）
// 否则使用取模运算（idx % len）

EventLoopGroup group = new NioEventLoopGroup(4);
// Channel1 → EventLoop-0
// Channel2 → EventLoop-1
// Channel3 → EventLoop-2
// Channel4 → EventLoop-3
// Channel5 → EventLoop-0（轮询）
```

---

### 3.3 ChannelPipeline 与 ChannelHandler

#### 3.3.1 Pipeline 结构

```
ChannelPipeline 双向链表结构：

  ┌─────────────────────────────────────────────────────────────┐
  │                      ChannelPipeline                        │
  │                                                             │
  │  HeadContext ◄──► Handler1 ◄──► Handler2 ◄──► TailContext  │
  │  (Inbound/      (Decoder)    (BusinessLogic)  (Inbound/    │
  │   Outbound)                                   Outbound)    │
  │                                                             │
  │  Inbound  事件传播方向：  Head ──────────────────► Tail      │
  │  Outbound 事件传播方向：  Tail ◄────────────────── Head     │
  └─────────────────────────────────────────────────────────────┘

  常见 Inbound 事件：channelRegistered, channelActive,
                    channelRead, channelReadComplete,
                    exceptionCaught, channelInactive

  常见 Outbound 事件：bind, connect, write, flush,
                     read, disconnect, close
```

#### 3.3.2 ChannelHandler 类型

```java
// 1. ChannelInboundHandlerAdapter - 处理入站事件
public class MyInboundHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("连接建立：" + ctx.channel().remoteAddress());
        ctx.fireChannelActive(); // 传递给下一个 InboundHandler
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("收到数据：" + msg);
        ctx.fireChannelRead(msg); // 传递给下一个 InboundHandler
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        System.out.println("连接断开：" + ctx.channel().remoteAddress());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close(); // 发生异常，关闭连接
    }
}

// 2. ChannelOutboundHandlerAdapter - 处理出站操作
public class MyOutboundHandler extends ChannelOutboundHandlerAdapter {

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        System.out.println("写出数据：" + msg);
        ctx.write(msg, promise); // 传递给下一个 OutboundHandler
    }

    @Override
    public void flush(ChannelHandlerContext ctx) {
        System.out.println("执行 flush");
        ctx.flush();
    }
}

// 3. ChannelDuplexHandler - 同时处理入站和出站
public class LoggingHandler extends ChannelDuplexHandler {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        System.out.println("[IN]  " + msg);
        ctx.fireChannelRead(msg);
    }

    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        System.out.println("[OUT] " + msg);
        ctx.write(msg, promise);
    }
}
```

#### 3.3.3 SimpleChannelInboundHandler 与自动释放

```java
// SimpleChannelInboundHandler 会自动调用 ReferenceCountUtil.release(msg)
// 适合只读不保留 msg 的场景
public class EchoHandler extends SimpleChannelInboundHandler<ByteBuf> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
        // msg 会在方法返回后自动释放，无需手动调用 release()
        System.out.println("收到：" + msg.toString(CharsetUtil.UTF_8));
        ctx.writeAndFlush(msg.retain()); // 如果需要保留，调用 retain()
    }
}

// 注意：如果使用 ChannelInboundHandlerAdapter，需要手动释放
public class ManualReleaseHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        try {
            if (msg instanceof ByteBuf) {
                ByteBuf buf = (ByteBuf) msg;
                System.out.println("收到：" + buf.toString(CharsetUtil.UTF_8));
            }
        } finally {
            ReferenceCountUtil.release(msg); // 必须释放！
        }
    }
}
```

#### 3.3.4 @Sharable 注解

```java
// 默认情况下，每个 Channel 应该有自己独立的 Handler 实例
// 使用 @Sharable 注解表示该 Handler 可以被多个 Channel 共享

@ChannelHandler.Sharable
public class StatelessHandler extends ChannelInboundHandlerAdapter {
    // 无状态，可以共享
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        ctx.fireChannelRead(msg);
    }
}

// 有状态的 Handler 不能加 @Sharable，否则多 Channel 共享时会有并发问题
public class StatefulHandler extends ChannelInboundHandlerAdapter {
    private int count = 0; // 状态变量，不能共享！

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        count++;
        System.out.println("第 " + count + " 次收到消息");
    }
}
```

---

### 3.4 ByteBuf

ByteBuf 是 Netty 对 JDK ByteBuffer 的增强，更灵活、更高效。

#### 3.4.1 ByteBuf 结构

```
ByteBuf 内部结构：

  0          readerIndex      writerIndex      capacity
  │               │               │               │
  ▼               ▼               ▼               ▼
  ┌───────────────┬───────────────┬───────────────┐
  │  已读（废弃）  │  可读数据区域  │  可写空间区域  │
  └───────────────┴───────────────┴───────────────┘
  │←── discardable ──►│←── readable ──►│←── writable ──►│

  readableBytes()  = writerIndex - readerIndex
  writableBytes()  = capacity - writerIndex
  maxWritableBytes = maxCapacity - writerIndex
```

#### 3.4.2 ByteBuf 分类

```
ByteBuf 分类体系：

  按内存位置：
  ├── HeapByteBuf    （JVM 堆内存，byte[]，GC 管理）
  └── DirectByteBuf  （JVM 堆外内存，NIO 直接内存，减少拷贝）

  按是否使用内存池：
  ├── PooledByteBuf    （内存池，默认，高性能）
  └── UnpooledByteBuf  （每次分配新内存）

  组合：
  ├── PooledHeapByteBuf
  ├── PooledDirectByteBuf      ← 默认（IO读写推荐）
  ├── UnpooledHeapByteBuf
  └── UnpooledDirectByteBuf
```

#### 3.4.3 ByteBuf 基本操作

```java
public class ByteBufDemo {
    public static void main(String[] args) {
        // 创建 ByteBuf
        ByteBuf buf = Unpooled.buffer(256);       // 堆内存
        ByteBuf direct = Unpooled.directBuffer(); // 堆外内存
        ByteBuf copy = Unpooled.copiedBuffer("Hello", CharsetUtil.UTF_8); // 复制内容

        // 写操作（自动移动 writerIndex）
        buf.writeInt(100);
        buf.writeLong(200L);
        buf.writeBytes("Hello Netty".getBytes());

        // 读操作（自动移动 readerIndex）
        int i = buf.readInt();
        long l = buf.readLong();
        byte[] bytes = new byte[11];
        buf.readBytes(bytes);

        // 随机访问（不移动 index）
        byte b = buf.getByte(0);
        buf.setByte(0, (byte) 'H');

        // 索引操作
        System.out.println("readableBytes: " + buf.readableBytes());
        System.out.println("writerIndex: " + buf.writerIndex());
        System.out.println("readerIndex: " + buf.readerIndex());

        // 标记与重置
        buf.markReaderIndex();   // 标记当前 readerIndex
        buf.readInt();           // 读取4字节
        buf.resetReaderIndex();  // 回到标记位置

        // 转为字符串
        String str = buf.toString(CharsetUtil.UTF_8);

        // 切片（零拷贝，共享底层数据）
        ByteBuf slice = buf.slice(0, 5);

        // 合并（零拷贝）
        CompositeByteBuf composite = Unpooled.compositeBuffer();
        ByteBuf header = Unpooled.wrappedBuffer(new byte[]{1, 2, 3});
        ByteBuf body = Unpooled.wrappedBuffer(new byte[]{4, 5, 6});
        composite.addComponents(true, header, body);

        // 释放（引用计数减1）
        buf.release();
    }
}
```

#### 3.4.4 引用计数（Reference Counting）

```java
// ByteBuf 使用引用计数管理内存
ByteBuf buf = ctx.alloc().buffer(); // refCnt = 1

buf.retain();  // refCnt = 2
buf.release(); // refCnt = 1
buf.release(); // refCnt = 0 → 内存回收

// 查看引用计数
int refCnt = buf.refCnt();

// 常见内存泄漏原因：
// 1. 忘记调用 release()
// 2. 在 SimpleChannelInboundHandler 中调用 retain() 后没有对应的 release()
// 3. 异常路径跳过了 release()

// 最佳实践：使用 try-finally 确保释放
ByteBuf buf2 = ctx.alloc().buffer();
try {
    // 使用 buf2
} finally {
    buf2.release();
}
```

#### 3.4.5 ByteBufAllocator

```java
// 通过 Channel 获取分配器（推荐）
ByteBufAllocator alloc = ctx.alloc();
ByteBuf buf = alloc.buffer(256);       // 根据配置分配堆内或堆外
ByteBuf heapBuf = alloc.heapBuffer();  // 强制堆内
ByteBuf directBuf = alloc.directBuffer(); // 强制堆外

// 全局默认分配器
ByteBuf buf2 = ByteBufAllocator.DEFAULT.buffer();

// Unpooled（非池化，适合小对象或测试）
ByteBuf buf3 = Unpooled.buffer(128);
ByteBuf buf4 = Unpooled.copiedBuffer("test", CharsetUtil.UTF_8);
```

---

### 3.5 ChannelHandlerContext

```java
public class ContextDemo extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // ctx 是 Handler 和 Pipeline 之间的桥梁

        // 获取关联的 Channel 和 Pipeline
        Channel channel = ctx.channel();
        ChannelPipeline pipeline = ctx.pipeline();

        // 事件传播：ctx.fireXxx() 从当前节点向后传播
        ctx.fireChannelRead(msg);     // 传给下一个 InboundHandler
        ctx.fireChannelActive();
        ctx.fireExceptionCaught(new RuntimeException("test"));

        // 出站操作：ctx.write() 从当前节点向前传播（到 Head）
        ctx.write(msg);               // 从当前 Handler 向前找 OutboundHandler
        ctx.writeAndFlush(msg);

        // channel.write() 从 Tail 开始向前传播（经过所有 Handler）
        channel.write(msg);           // 从 Tail 开始
        channel.writeAndFlush(msg);

        // 获取分配器
        ByteBufAllocator alloc = ctx.alloc();

        // Handler 名称和引用
        String name = ctx.name();
        ChannelHandler handler = ctx.handler();
    }
}
```

---

### 3.6 Promise 与 Future

```java
// Netty 的 Future 继承自 JDK Future，添加了监听器支持
// Promise 是可写的 Future，用于设置操作结果

public class PromiseDemo {
    public static void main(String[] args) throws Exception {
        EventExecutorGroup group = new DefaultEventExecutorGroup(4);
        EventExecutor executor = group.next();

        // 创建 Promise
        Promise<String> promise = executor.newPromise();

        // 添加监听器
        promise.addListener(future -> {
            if (future.isSuccess()) {
                System.out.println("结果：" + future.getNow());
            } else {
                System.out.println("失败：" + future.cause());
            }
        });

        // 在另一个线程中设置结果
        executor.execute(() -> {
            try {
                Thread.sleep(1000);
                promise.setSuccess("Hello from Promise!");
                // 或者设置失败：promise.setFailure(new RuntimeException("error"));
            } catch (Exception e) {
                promise.setFailure(e);
            }
        });

        // 等待完成
        promise.sync();
        group.shutdownGracefully();
    }
}
```

---

---

## 4. 编解码器（Codec）

### 4.1 粘包与拆包问题

```
粘包/拆包示意图：

  正常情况：
  发送端：[Msg1][Msg2][Msg3]
  接收端：[Msg1][Msg2][Msg3]  ✓

  粘包（多条消息粘在一起）：
  发送端：[Msg1][Msg2][Msg3]
  接收端：[Msg1+Msg2][Msg3]  ✗

  拆包（一条消息被分成多片）：
  发送端：[Msg1][Msg2]
  接收端：[Msg1前半][Msg1后半+Msg2]  ✗

  产生原因：
  ├── TCP 是流式协议，无消息边界
  ├── 发送方：Nagle 算法合并小包
  └── 接收方：缓冲区大小限制
```

**Netty 解决粘包/拆包的4种方案：**

| 方案 | 解码器 | 说明 |
|------|--------|------|
| 固定长度 | `FixedLengthFrameDecoder` | 每条消息固定字节数 |
| 分隔符 | `DelimiterBasedFrameDecoder` | 使用特殊字符分隔消息 |
| 长度字段 | `LengthFieldBasedFrameDecoder` | 消息头包含长度字段（最常用） |
| 自定义 | 继承 `ByteToMessageDecoder` | 完全自定义拆包逻辑 |

---

### 4.2 常用内置解码器

#### 4.2.1 FixedLengthFrameDecoder

```java
// 每条消息固定 100 字节
pipeline.addLast(new FixedLengthFrameDecoder(100));
```

#### 4.2.2 LineBasedFrameDecoder 和 DelimiterBasedFrameDecoder

```java
// 以换行符（\n 或 \r\n）为分隔符，最大帧长度 1024
pipeline.addLast(new LineBasedFrameDecoder(1024));

// 自定义分隔符
ByteBuf delimiter = Unpooled.copiedBuffer("$$", CharsetUtil.UTF_8);
pipeline.addLast(new DelimiterBasedFrameDecoder(1024, delimiter));
```

#### 4.2.3 LengthFieldBasedFrameDecoder（重点）

```
消息格式：

  ┌────────────────┬──────────────────────────────┐
  │  Length (4B)   │         Body (N bytes)        │
  └────────────────┴──────────────────────────────┘
  ↑
  lengthFieldOffset=0, lengthFieldLength=4

  构造参数含义：
  maxFrameLength      - 最大帧长度（防止OOM攻击）
  lengthFieldOffset   - 长度字段相对于消息起始的偏移量
  lengthFieldLength   - 长度字段本身占用的字节数（1/2/3/4/8）
  lengthAdjustment    - 长度字段的值需要加的调整量
  initialBytesToStrip - 解码后跳过的字节数（通常跳过Header）
```

```java
// 场景1：最简单的情况
// 消息格式：[Length(4B)][Body]，Length 表示 Body 的长度
pipeline.addLast(new LengthFieldBasedFrameDecoder(
    65535,  // maxFrameLength
    0,      // lengthFieldOffset: Length 从第0字节开始
    4,      // lengthFieldLength: Length 占4字节
    0,      // lengthAdjustment: 不调整
    4       // initialBytesToStrip: 跳过4字节Length，只保留Body
));

// 场景2：消息格式 [Header(2B)][Length(4B)][Body]
// Length 表示 Body 的长度
pipeline.addLast(new LengthFieldBasedFrameDecoder(
    65535,
    2,      // lengthFieldOffset: Length 从第2字节开始
    4,      // lengthFieldLength: Length 占4字节
    0,      // lengthAdjustment
    0       // initialBytesToStrip: 不跳过任何字节（保留完整消息）
));

// 场景3：消息格式 [Length(4B)][Body]，Length 包含自身长度（即Length值 = 4 + Body长度）
pipeline.addLast(new LengthFieldBasedFrameDecoder(
    65535,
    0,
    4,
    -4,     // lengthAdjustment = -4（因为Length值包含了自身）
    4       // initialBytesToStrip: 跳过Length字段
));

// 对应的编码器
pipeline.addLast(new LengthFieldPrepender(4)); // 在消息前添加4字节长度字段
```

---

### 4.3 自定义编解码器

#### 4.3.1 自定义解码器（ByteToMessageDecoder）

```java
/**
 * 自定义协议解码器
 * 消息格式：[魔数(2B)][版本(1B)][消息类型(1B)][序列号(4B)][长度(4B)][内容(NB)]
 */
public class CustomDecoder extends ByteToMessageDecoder {

    private static final int HEADER_SIZE = 12; // 2+1+1+4+4
    private static final short MAGIC = (short) 0xCAFE;

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 可读字节不足 Header，等待更多数据
        if (in.readableBytes() < HEADER_SIZE) {
            return;
        }

        // 标记读取位置，以便不够数据时回退
        in.markReaderIndex();

        // 读取魔数
        short magic = in.readShort();
        if (magic != MAGIC) {
            ctx.close();
            return;
        }

        byte version = in.readByte();
        byte msgType = in.readByte();
        int serialNo = in.readInt();
        int bodyLength = in.readInt();

        // Body 数据还没到齐，等待
        if (in.readableBytes() < bodyLength) {
            in.resetReaderIndex(); // 回退到 Header 起始位置
            return;
        }

        // 读取 Body
        byte[] body = new byte[bodyLength];
        in.readBytes(body);

        // 组装消息对象
        CustomMessage msg = new CustomMessage();
        msg.setMagic(magic);
        msg.setVersion(version);
        msg.setMsgType(msgType);
        msg.setSerialNo(serialNo);
        msg.setBody(body);

        out.add(msg);
    }
}

// 消息类
public class CustomMessage {
    private short magic;
    private byte version;
    private byte msgType;
    private int serialNo;
    private byte[] body;
    // getter/setter 省略
}
```

#### 4.3.2 自定义编码器（MessageToByteEncoder）

```java
public class CustomEncoder extends MessageToByteEncoder<CustomMessage> {

    private static final short MAGIC = (short) 0xCAFE;
    private static final byte VERSION = 1;

    @Override
    protected void encode(ChannelHandlerContext ctx, CustomMessage msg, ByteBuf out) {
        out.writeShort(MAGIC);
        out.writeByte(VERSION);
        out.writeByte(msg.getMsgType());
        out.writeInt(msg.getSerialNo());
        out.writeInt(msg.getBody().length);
        out.writeBytes(msg.getBody());
    }
}
```

#### 4.3.3 MessageToMessageDecoder（对象转对象）

```java
// 将字节流 → 自定义对象后，再将自定义对象 → 业务对象
public class JsonDecoder extends MessageToMessageDecoder<ByteBuf> {

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) {
        String json = msg.toString(CharsetUtil.UTF_8);
        try {
            // 假设使用 Jackson
            MyRequest request = new ObjectMapper().readValue(json, MyRequest.class);
            out.add(request);
        } catch (Exception e) {
            throw new DecoderException("JSON 解析失败：" + e.getMessage(), e);
        }
    }
}

public class JsonEncoder extends MessageToByteEncoder<MyResponse> {

    @Override
    protected void encode(ChannelHandlerContext ctx, MyResponse response, ByteBuf out) throws Exception {
        String json = new ObjectMapper().writeValueAsString(response);
        out.writeBytes(json.getBytes(CharsetUtil.UTF_8));
    }
}
```

#### 4.3.4 整合编解码器（ByteToMessageCodec）

```java
// 将编解码器合二为一
public class CustomCodec extends ByteToMessageCodec<CustomMessage> {

    @Override
    protected void encode(ChannelHandlerContext ctx, CustomMessage msg, ByteBuf out) {
        // 编码逻辑
        out.writeInt(msg.getSerialNo());
        byte[] body = msg.getBody();
        out.writeInt(body.length);
        out.writeBytes(body);
    }

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        if (in.readableBytes() < 8) return;

        in.markReaderIndex();
        int serialNo = in.readInt();
        int bodyLen = in.readInt();

        if (in.readableBytes() < bodyLen) {
            in.resetReaderIndex();
            return;
        }

        byte[] body = new byte[bodyLen];
        in.readBytes(body);

        CustomMessage msg = new CustomMessage();
        msg.setSerialNo(serialNo);
        msg.setBody(body);
        out.add(msg);
    }
}
```

---

### 4.4 Protobuf 集成

```java
// 在 Pipeline 中添加 Protobuf 编解码器
pipeline.addLast(new ProtobufVarint32FrameDecoder());
pipeline.addLast(new ProtobufDecoder(MyProto.MyMessage.getDefaultInstance()));
pipeline.addLast(new ProtobufVarint32LengthFieldPrepender());
pipeline.addLast(new ProtobufEncoder());
pipeline.addLast(new MyProtobufHandler());

// proto 文件示例
/*
syntax = "proto3";
option java_outer_classname = "MyProto";
option java_package = "com.example.proto";

message MyMessage {
    int32 id = 1;
    string name = 2;
    bytes data = 3;
}
*/
```

---

---

## 5. 常用 Handler

### 5.1 心跳检测：IdleStateHandler

```
心跳机制示意图：

  Client                              Server
    │                                   │
    │──── 正常业务数据 ─────────────────►│
    │                                   │ (超过 readerIdleTime 无数据)
    │                                   │ IdleStateEvent(READER_IDLE)
    │◄─── 心跳请求（PING）──────────────│
    │──── 心跳响应（PONG）─────────────►│
    │                                   │ 重置计时器
    │                                   │
    │  (长时间无响应)                    │
    │◄────────────────────────────────── 关闭连接
```

```java
// 服务端配置心跳
pipeline.addLast(new IdleStateHandler(
    60,   // readerIdleTime: 60秒没收到数据触发 READER_IDLE
    30,   // writerIdleTime: 30秒没发送数据触发 WRITER_IDLE
    0,    // allIdleTime:    0=禁用 ALL_IDLE
    TimeUnit.SECONDS
));
pipeline.addLast(new HeartbeatHandler());

// 心跳处理器
public class HeartbeatHandler extends ChannelInboundHandlerAdapter {

    private static final int MAX_IDLE_COUNT = 3;
    private int idleCount = 0;

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;

            if (event.state() == IdleState.READER_IDLE) {
                idleCount++;
                System.out.println("读空闲次数：" + idleCount);

                if (idleCount >= MAX_IDLE_COUNT) {
                    System.out.println("连接超时，关闭：" + ctx.channel().remoteAddress());
                    ctx.close();
                } else {
                    // 发送心跳包
                    ctx.writeAndFlush(buildPingMessage());
                }
            } else if (event.state() == IdleState.WRITER_IDLE) {
                // 定期发送心跳
                ctx.writeAndFlush(buildPingMessage());
            }
        } else {
            super.userEventTriggered(ctx, evt);
        }
    }

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        idleCount = 0; // 收到任何消息，重置空闲计数
        ctx.fireChannelRead(msg);
    }

    private Object buildPingMessage() {
        // 构建心跳消息（具体格式取决于协议）
        return Unpooled.copiedBuffer("PING", CharsetUtil.UTF_8);
    }
}

// 客户端配置：定期发送心跳
pipeline.addLast(new IdleStateHandler(0, 30, 0, TimeUnit.SECONDS));
pipeline.addLast(new ClientHeartbeatHandler());

public class ClientHeartbeatHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
        if (evt instanceof IdleStateEvent) {
            IdleStateEvent event = (IdleStateEvent) evt;
            if (event.state() == IdleState.WRITER_IDLE) {
                System.out.println("发送心跳...");
                ctx.writeAndFlush(Unpooled.copiedBuffer("PING", CharsetUtil.UTF_8));
            }
        }
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        // 连接断开后重连
        System.out.println("连接断开，准备重连...");
        scheduleReconnect(ctx);
    }

    private void scheduleReconnect(ChannelHandlerContext ctx) {
        ctx.channel().eventLoop().schedule(() -> {
            System.out.println("正在重连...");
            // 重连逻辑
        }, 5, TimeUnit.SECONDS);
    }
}
```

---

### 5.2 SSL/TLS 支持

```java
// 服务端 SSL 配置
SslContext sslCtx = SslContextBuilder
    .forServer(certChainFile, privateKeyFile)
    .clientAuth(ClientAuth.OPTIONAL) // 是否需要客户端证书
    .build();

// 在 Pipeline 中添加 SslHandler（必须在第一位）
pipeline.addFirst(sslCtx.newHandler(ch.alloc()));

// 客户端 SSL 配置
SslContext clientSslCtx = SslContextBuilder
    .forClient()
    .trustManager(caFile)
    .build();

pipeline.addFirst(clientSslCtx.newHandler(ch.alloc(), host, port));

// 自签名证书（用于测试）
SelfSignedCertificate ssc = new SelfSignedCertificate();
SslContext testSslCtx = SslContextBuilder
    .forServer(ssc.certificate(), ssc.privateKey())
    .build();
```

---

### 5.3 HTTP 编解码器

```java
// 服务端 HTTP Pipeline
pipeline.addLast(new HttpServerCodec());             // HTTP 编解码
pipeline.addLast(new HttpObjectAggregator(65536));   // 聚合完整的 HTTP 消息
pipeline.addLast(new ChunkedWriteHandler());         // 支持分块写入（大文件）
pipeline.addLast(new HttpServerHandler());

// HTTP Handler 示例
public class HttpServerHandler extends SimpleChannelInboundHandler<FullHttpRequest> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
        // 获取请求信息
        String uri = request.uri();
        HttpMethod method = request.method();
        HttpHeaders headers = request.headers();
        ByteBuf content = request.content();

        System.out.println("URI: " + uri);
        System.out.println("Method: " + method);
        System.out.println("Content-Type: " + headers.get(HttpHeaderNames.CONTENT_TYPE));

        // 构建响应
        String responseBody = "<html><body><h1>Hello Netty HTTP!</h1></body></html>";
        FullHttpResponse response = new DefaultFullHttpResponse(
            HttpVersion.HTTP_1_1,
            HttpResponseStatus.OK,
            Unpooled.copiedBuffer(responseBody, CharsetUtil.UTF_8)
        );

        response.headers()
            .set(HttpHeaderNames.CONTENT_TYPE, "text/html; charset=UTF-8")
            .setInt(HttpHeaderNames.CONTENT_LENGTH, response.content().readableBytes())
            .set(HttpHeaderNames.CONNECTION, HttpHeaderValues.KEEP_ALIVE);

        ctx.writeAndFlush(response);
    }
}
```

---

### 5.4 WebSocket 支持

```java
// WebSocket 服务端 Pipeline
pipeline.addLast(new HttpServerCodec());
pipeline.addLast(new HttpObjectAggregator(65536));
pipeline.addLast(new WebSocketServerProtocolHandler("/ws")); // WebSocket 握手路径
pipeline.addLast(new WebSocketFrameHandler());

// WebSocket Handler
public class WebSocketFrameHandler extends SimpleChannelInboundHandler<WebSocketFrame> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, WebSocketFrame frame) {
        if (frame instanceof TextWebSocketFrame) {
            String text = ((TextWebSocketFrame) frame).text();
            System.out.println("收到文本：" + text);
            ctx.writeAndFlush(new TextWebSocketFrame("Echo: " + text));

        } else if (frame instanceof BinaryWebSocketFrame) {
            ByteBuf data = frame.content();
            System.out.println("收到二进制数据，长度：" + data.readableBytes());

        } else if (frame instanceof PingWebSocketFrame) {
            ctx.writeAndFlush(new PongWebSocketFrame(frame.content().retain()));

        } else if (frame instanceof CloseWebSocketFrame) {
            ctx.channel().close();
        }
    }
}
```

---

### 5.5 流量整形（TrafficShapingHandler）

```java
// 全局流量整形（单例）
GlobalTrafficShapingHandler globalHandler = new GlobalTrafficShapingHandler(
    workerGroup,
    1024 * 1024,  // writeLimit: 写限速，1MB/s
    1024 * 1024,  // readLimit:  读限速，1MB/s
    1000          // checkInterval: 检测间隔，1秒
);

// Channel 级别流量整形
pipeline.addLast(new ChannelTrafficShapingHandler(
    512 * 1024,  // 写限速 512KB/s
    512 * 1024,  // 读限速 512KB/s
    1000
));
```

---

### 5.6 LoggingHandler

```java
// 日志 Handler（调试神器，记录所有入站/出站事件）
pipeline.addLast(new LoggingHandler(LogLevel.DEBUG));

// 或指定 logger 名称
pipeline.addLast(new LoggingHandler("my.channel.logger", LogLevel.INFO));
```

---

---

## 6. Netty 实战案例

### 6.1 Echo 服务器（完整版）

```java
// === EchoServer.java ===
public class EchoServer {

    private final int port;

    public EchoServer(int port) {
        this.port = port;
    }

    public void start() throws Exception {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .option(ChannelOption.SO_BACKLOG, 256)
             .option(ChannelOption.SO_REUSEADDR, true)
             .childOption(ChannelOption.SO_KEEPALIVE, true)
             .childOption(ChannelOption.TCP_NODELAY, true)
             .childHandler(new EchoServerInitializer());

            ChannelFuture f = b.bind(port).sync();
            System.out.println("Echo 服务端启动成功，端口：" + port);
            f.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws Exception {
        new EchoServer(8080).start();
    }
}

// === EchoServerInitializer.java ===
public class EchoServerInitializer extends ChannelInitializer<SocketChannel> {

    @Override
    protected void initChannel(SocketChannel ch) {
        ChannelPipeline p = ch.pipeline();
        p.addLast(new LoggingHandler(LogLevel.INFO));
        p.addLast(new LineBasedFrameDecoder(1024));
        p.addLast(new StringDecoder(CharsetUtil.UTF_8));
        p.addLast(new StringEncoder(CharsetUtil.UTF_8));
        p.addLast(new EchoServerHandler());
    }
}

// === EchoServerHandler.java ===
@ChannelHandler.Sharable
public class EchoServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("客户端连接：" + ctx.channel().remoteAddress());
        ctx.writeAndFlush("欢迎连接到 Echo 服务器！\n");
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("[" + ctx.channel().remoteAddress() + "] 收到：" + msg);
        ctx.writeAndFlush("Echo: " + msg + "\n");
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        System.out.println("客户端断开：" + ctx.channel().remoteAddress());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}

// === EchoClient.java ===
public class EchoClient {

    public static void main(String[] args) throws Exception {
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap b = new Bootstrap();
            b.group(group)
             .channel(NioSocketChannel.class)
             .option(ChannelOption.TCP_NODELAY, true)
             .handler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 protected void initChannel(SocketChannel ch) {
                     ch.pipeline()
                       .addLast(new LineBasedFrameDecoder(1024))
                       .addLast(new StringDecoder(CharsetUtil.UTF_8))
                       .addLast(new StringEncoder(CharsetUtil.UTF_8))
                       .addLast(new EchoClientHandler());
                 }
             });

            ChannelFuture f = b.connect("localhost", 8080).sync();

            // 从控制台读取输入并发送
            BufferedReader reader = new BufferedReader(new InputStreamReader(System.in));
            String line;
            while ((line = reader.readLine()) != null) {
                f.channel().writeAndFlush(line + "\n");
                if ("quit".equalsIgnoreCase(line)) break;
            }

            f.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }
}
```

---

### 6.2 自定义协议 RPC 框架

```java
// === 协议定义 ===
/**
 * 自定义 RPC 协议格式：
 * ┌──────────┬───────┬──────────┬──────────┬──────────────┐
 * │ 魔数(4B)  │版本(1B)│消息类型(1B)│序列号(8B) │ 消息体长度(4B) │ 消息体(NB)
 * └──────────┴───────┴──────────┴──────────┴──────────────┘
 */
public class RpcMessage {
    public static final int MAGIC = 0xCAFEBABE;
    public static final byte VERSION = 1;
    public static final byte TYPE_REQUEST  = 1;
    public static final byte TYPE_RESPONSE = 2;
    public static final byte TYPE_HEARTBEAT = 3;

    private int magic;
    private byte version;
    private byte messageType;
    private long sequenceId;
    private int bodyLength;
    private byte[] body;

    // getter/setter
    public int getMagic() { return magic; }
    public void setMagic(int magic) { this.magic = magic; }
    public byte getVersion() { return version; }
    public void setVersion(byte version) { this.version = version; }
    public byte getMessageType() { return messageType; }
    public void setMessageType(byte messageType) { this.messageType = messageType; }
    public long getSequenceId() { return sequenceId; }
    public void setSequenceId(long sequenceId) { this.sequenceId = sequenceId; }
    public int getBodyLength() { return bodyLength; }
    public void setBodyLength(int bodyLength) { this.bodyLength = bodyLength; }
    public byte[] getBody() { return body; }
    public void setBody(byte[] body) { this.body = body; }
}

// === RPC 解码器 ===
public class RpcDecoder extends ByteToMessageDecoder {

    private static final int HEADER_SIZE = 4 + 1 + 1 + 8 + 4; // 18字节

    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        if (in.readableBytes() < HEADER_SIZE) return;

        in.markReaderIndex();

        int magic = in.readInt();
        if (magic != RpcMessage.MAGIC) {
            throw new RuntimeException("非法魔数：" + Integer.toHexString(magic));
        }

        byte version = in.readByte();
        byte messageType = in.readByte();
        long sequenceId = in.readLong();
        int bodyLength = in.readInt();

        if (in.readableBytes() < bodyLength) {
            in.resetReaderIndex();
            return;
        }

        byte[] body = new byte[bodyLength];
        in.readBytes(body);

        RpcMessage msg = new RpcMessage();
        msg.setMagic(magic);
        msg.setVersion(version);
        msg.setMessageType(messageType);
        msg.setSequenceId(sequenceId);
        msg.setBodyLength(bodyLength);
        msg.setBody(body);

        out.add(msg);
    }
}

// === RPC 编码器 ===
public class RpcEncoder extends MessageToByteEncoder<RpcMessage> {

    @Override
    protected void encode(ChannelHandlerContext ctx, RpcMessage msg, ByteBuf out) {
        out.writeInt(msg.getMagic());
        out.writeByte(msg.getVersion());
        out.writeByte(msg.getMessageType());
        out.writeLong(msg.getSequenceId());
        out.writeInt(msg.getBody() != null ? msg.getBody().length : 0);
        if (msg.getBody() != null) {
            out.writeBytes(msg.getBody());
        }
    }
}

// === 客户端调用处理（Promise 模式）===
public class RpcClientHandler extends SimpleChannelInboundHandler<RpcMessage> {

    // 存储等待中的请求：sequenceId → Promise
    private final Map<Long, CompletableFuture<RpcMessage>> pendingRequests
        = new ConcurrentHashMap<>();

    private final AtomicLong sequenceIdGenerator = new AtomicLong(0);

    public CompletableFuture<RpcMessage> sendRequest(Channel channel, byte[] requestBody) {
        long seqId = sequenceIdGenerator.incrementAndGet();

        RpcMessage request = new RpcMessage();
        request.setMagic(RpcMessage.MAGIC);
        request.setVersion(RpcMessage.VERSION);
        request.setMessageType(RpcMessage.TYPE_REQUEST);
        request.setSequenceId(seqId);
        request.setBody(requestBody);
        request.setBodyLength(requestBody.length);

        CompletableFuture<RpcMessage> future = new CompletableFuture<>();
        pendingRequests.put(seqId, future);

        channel.writeAndFlush(request).addListener(writeFuture -> {
            if (!writeFuture.isSuccess()) {
                pendingRequests.remove(seqId);
                future.completeExceptionally(writeFuture.cause());
            }
        });

        return future;
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RpcMessage response) {
        long seqId = response.getSequenceId();
        CompletableFuture<RpcMessage> future = pendingRequests.remove(seqId);
        if (future != null) {
            future.complete(response);
        }
    }
}
```

---

### 6.3 WebSocket 聊天室

```java
// === ChatServer.java ===
public class ChatServer {

    // 所有在线用户的 Channel 集合
    public static final ChannelGroup channels =
        new DefaultChannelGroup(GlobalEventExecutor.INSTANCE);

    public static void main(String[] args) throws Exception {
        EventLoopGroup boss = new NioEventLoopGroup(1);
        EventLoopGroup worker = new NioEventLoopGroup();

        try {
            ServerBootstrap b = new ServerBootstrap();
            b.group(boss, worker)
             .channel(NioServerSocketChannel.class)
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 protected void initChannel(SocketChannel ch) {
                     ch.pipeline()
                       .addLast(new HttpServerCodec())
                       .addLast(new HttpObjectAggregator(65536))
                       .addLast(new WebSocketServerProtocolHandler("/chat"))
                       .addLast(new ChatHandler());
                 }
             });

            b.bind(8080).sync().channel().closeFuture().sync();
        } finally {
            boss.shutdownGracefully();
            worker.shutdownGracefully();
        }
    }
}

// === ChatHandler.java ===
public class ChatHandler extends SimpleChannelInboundHandler<TextWebSocketFrame> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        Channel channel = ctx.channel();
        // 广播：新用户上线
        ChatServer.channels.writeAndFlush(
            new TextWebSocketFrame("[系统] " + channel.remoteAddress() + " 加入聊天室")
        );
        ChatServer.channels.add(channel);
        System.out.println("用户上线：" + channel.remoteAddress() + "，在线人数：" + ChatServer.channels.size());
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        Channel channel = ctx.channel();
        ChatServer.channels.remove(channel);
        // 广播：用户下线
        ChatServer.channels.writeAndFlush(
            new TextWebSocketFrame("[系统] " + channel.remoteAddress() + " 离开聊天室")
        );
        System.out.println("用户下线：" + channel.remoteAddress() + "，在线人数：" + ChatServer.channels.size());
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, TextWebSocketFrame msg) {
        Channel sender = ctx.channel();
        String text = msg.text();

        // 广播给其他所有用户
        for (Channel channel : ChatServer.channels) {
            if (channel == sender) {
                channel.writeAndFlush(new TextWebSocketFrame("[我] " + text));
            } else {
                channel.writeAndFlush(new TextWebSocketFrame("[" + sender.remoteAddress() + "] " + text));
            }
        }
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

---

### 6.4 文件传输服务

```java
public class FileServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String fileName) throws Exception {
        File file = new File(fileName);

        if (!file.exists()) {
            ctx.writeAndFlush("ERROR: File not found\n");
            return;
        }

        RandomAccessFile raf = new RandomAccessFile(file, "r");

        // 方式1：使用 ChunkedFile（适合所有平台）
        ctx.pipeline().addBefore(ctx.name(), "chunkedWriter", new ChunkedWriteHandler());
        ctx.writeAndFlush(new ChunkedFile(raf, 0, file.length(), 8192));

        // 方式2：使用 DefaultFileRegion 零拷贝（仅 NIO，不支持 SSL）
        // FileRegion region = new DefaultFileRegion(raf.getChannel(), 0, file.length());
        // ctx.writeAndFlush(region).addListener(future -> raf.close());
    }
}
```

---

### 6.5 简易 IM 系统架构

```
IM 系统架构示意：

  ┌─────────────────────────────────────────────────────────────┐
  │                        IM Server                           │
  │                                                             │
  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
  │  │  接入层       │    │  业务层       │    │  存储层       │  │
  │  │  Netty Server│───►│  消息路由     │───►│  Redis       │  │
  │  │  WebSocket   │    │  用户管理     │    │  MySQL       │  │
  │  │  TCP         │    │  群组管理     │    │  MongoDB     │  │
  │  └──────────────┘    └──────────────┘    └──────────────┘  │
  └─────────────────────────────────────────────────────────────┘

  用户会话管理（在线状态）：
  ┌─────────────────────────────────────────────────────────────┐
  │                  UserSessionManager                         │
  │                                                             │
  │  Map<Long, Channel>  onlineUsers                           │
  │                                                             │
  │  login(userId, channel)   → 用户上线，注册 Channel           │
  │  logout(userId)           → 用户下线，注销 Channel           │
  │  isOnline(userId)         → 查询在线状态                     │
  │  sendTo(userId, message)  → 向指定用户发送消息               │
  └─────────────────────────────────────────────────────────────┘
```

```java
// 用户会话管理器
public class UserSessionManager {

    private static final ConcurrentHashMap<Long, Channel> onlineUsers = new ConcurrentHashMap<>();

    // 用户上线
    public static void online(Long userId, Channel channel) {
        onlineUsers.put(userId, channel);
        // 监听 Channel 关闭，自动下线
        channel.closeFuture().addListener(future -> offline(userId));
        System.out.println("用户 " + userId + " 上线，当前在线：" + onlineUsers.size());
    }

    // 用户下线
    public static void offline(Long userId) {
        onlineUsers.remove(userId);
        System.out.println("用户 " + userId + " 下线，当前在线：" + onlineUsers.size());
    }

    // 发送消息给指定用户
    public static boolean sendMessage(Long targetUserId, Object message) {
        Channel channel = onlineUsers.get(targetUserId);
        if (channel != null && channel.isActive()) {
            channel.writeAndFlush(message);
            return true;
        }
        return false; // 用户不在线，消息需要离线存储
    }

    // 广播给所有在线用户
    public static void broadcast(Object message) {
        onlineUsers.values().forEach(ch -> {
            if (ch.isActive()) {
                ch.writeAndFlush(message);
            }
        });
    }

    public static int getOnlineCount() {
        return onlineUsers.size();
    }
}

// IM 消息处理器
public class ImMessageHandler extends SimpleChannelInboundHandler<ImMessage> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ImMessage msg) {
        switch (msg.getType()) {
            case ImMessage.TYPE_LOGIN:
                // 登录
                Long userId = msg.getUserId();
                UserSessionManager.online(userId, ctx.channel());
                ctx.writeAndFlush(ImMessage.buildLoginAck(userId, "登录成功"));
                break;

            case ImMessage.TYPE_SINGLE_CHAT:
                // 单聊
                Long targetId = msg.getTargetId();
                boolean sent = UserSessionManager.sendMessage(targetId, msg);
                if (!sent) {
                    // 目标用户不在线，存入离线消息
                    saveOfflineMessage(msg);
                }
                // 发送确认给发送方
                ctx.writeAndFlush(ImMessage.buildAck(msg.getMsgId(), sent ? "已送达" : "已存储（对方离线）"));
                break;

            case ImMessage.TYPE_GROUP_CHAT:
                // 群聊：获取群成员列表，逐一发送
                List<Long> members = getGroupMembers(msg.getGroupId());
                members.stream()
                    .filter(uid -> !uid.equals(msg.getUserId())) // 排除发送者
                    .forEach(uid -> UserSessionManager.sendMessage(uid, msg));
                break;

            case ImMessage.TYPE_HEARTBEAT:
                // 心跳响应
                ctx.writeAndFlush(ImMessage.buildPong());
                break;
        }
    }

    private void saveOfflineMessage(ImMessage msg) {
        // 存入 Redis 或数据库
        System.out.println("存储离线消息：" + msg.getMsgId());
    }

    private List<Long> getGroupMembers(Long groupId) {
        // 从数据库或缓存获取群成员
        return new ArrayList<>();
    }
}
```

---

---

## 7. Netty 高级特性

### 7.1 内存池（PooledByteBufAllocator）

```
Netty 内存池架构：

  PooledByteBufAllocator
    │
    ├── PoolArena[] heapArenas    （堆内存池，数组大小 = min(2*CPU, 512)）
    └── PoolArena[] directArenas  （堆外内存池）

  PoolArena 内部结构：
  ┌──────────────────────────────────────────────────────────┐
  │  PoolArena                                               │
  │                                                          │
  │  tinySubpagePools[32]    → 16B, 32B, ..., 512B           │
  │  smallSubpagePools[4]    → 1KB, 2KB, 4KB, 8KB           │
  │                                                          │
  │  PoolChunkList:                                          │
  │  qInit → q000 → q025 → q050 → q075 → q100              │
  │           （按使用率分组，0% 25% 50% 75% 100%）           │
  └──────────────────────────────────────────────────────────┘

  PoolChunk（默认16MB）：
  ├── 内部用完全二叉树管理 Page（8KB）
  ├── 通过伙伴算法分配/释放内存
  └── 支持 Subpage（小于 8KB 的对象）

  ThreadLocal 缓存（PoolThreadCache）：
  └── 每个线程维护本地缓存，避免竞争
      ├── tinyCache
      ├── smallCache
      └── normalCache
```

**内存池配置：**

```java
// 默认使用池化内存分配器（推荐）
ByteBufAllocator alloc = PooledByteBufAllocator.DEFAULT;

// 查看内存使用情况
PooledByteBufAllocator allocator = (PooledByteBufAllocator) ctx.alloc();
PooledByteBufAllocatorMetric metric = allocator.metric();
System.out.println("堆内存使用：" + metric.usedHeapMemory());
System.out.println("堆外内存使用：" + metric.usedDirectMemory());
System.out.println("Arena数量：" + metric.numHeapArenas());

// Bootstrap 中配置
bootstrap.option(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
bootstrap.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);

// JVM 参数调优
// -Dio.netty.allocator.type=pooled             使用池化分配器
// -Dio.netty.allocator.numHeapArenas=16        堆内Arena数量
// -Dio.netty.allocator.numDirectArenas=16      堆外Arena数量
// -Dio.netty.allocator.pageSize=8192           Page大小（8KB）
// -Dio.netty.allocator.maxOrder=11             最大Chunk=pageSize<<maxOrder=16MB
// -Dio.netty.allocator.tinyCacheSize=512       tiny缓存大小
// -Dio.netty.allocator.smallCacheSize=256      small缓存大小
// -Dio.netty.allocator.normalCacheSize=64      normal缓存大小
```

---

### 7.2 零拷贝深度解析

#### 7.2.1 CompositeByteBuf

```java
// 场景：HTTP 响应需要将 Header 和 Body 合并发送
// 传统方式（有拷贝）：
ByteBuf header = ...;
ByteBuf body = ...;
ByteBuf merged = Unpooled.buffer(header.readableBytes() + body.readableBytes());
merged.writeBytes(header);
merged.writeBytes(body);
// 发送 merged

// 零拷贝方式（CompositeByteBuf，不实际拷贝数据）：
CompositeByteBuf composite = ctx.alloc().compositeBuffer();
composite.addComponents(true, header, body);
// addComponents 的第一个参数 true 表示自动推进 writerIndex
ctx.writeAndFlush(composite);
```

#### 7.2.2 slice 和 duplicate

```java
ByteBuf original = Unpooled.copiedBuffer("Hello World", CharsetUtil.UTF_8);

// slice：切片，不拷贝，共享底层内存
ByteBuf slice = original.slice(0, 5);   // "Hello"
// 修改 slice 会影响 original！

// duplicate：复制整个 ByteBuf 的视图，不拷贝，共享底层内存
ByteBuf dup = original.duplicate();

// copy：深拷贝，拷贝数据
ByteBuf copy = original.copy();  // 独立内存

// 注意：slice/duplicate 需要调用 retain() 来维持引用计数
ByteBuf slice2 = original.retainedSlice(0, 5); // 相当于 slice(0,5).retain()
```

#### 7.2.3 FileRegion 文件零拷贝

```java
// 使用 sendfile 系统调用，文件数据直接从内核缓冲区发到网卡
// 不需要经过用户空间，实现真正的零拷贝
public class ZeroCopyFileHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String fileName) throws Exception {
        RandomAccessFile raf = new RandomAccessFile(fileName, "r");
        FileRegion region = new DefaultFileRegion(raf.getChannel(), 0, raf.length());

        // 发送 HTTP Header
        HttpResponse response = new DefaultHttpResponse(HTTP_1_1, OK);
        response.headers().set(CONTENT_LENGTH, raf.length());
        ctx.write(response);

        // 零拷贝发送文件
        ctx.write(region);
        ctx.writeAndFlush(LastHttpContent.EMPTY_LAST_CONTENT)
           .addListener(f -> raf.close());
    }
}
```

---

### 7.3 高性能设计剖析

#### 7.3.1 无锁化设计

```
Netty 无锁化核心：

  ① Channel 绑定到唯一 EventLoop（单线程处理所有事件）
  ② 外部线程提交任务时，封装成 Task 放入 TaskQueue
  ③ EventLoop 串行处理 TaskQueue，无需 synchronized

  判断是否需要投递：
  if (eventLoop.inEventLoop()) {
      // 在 EventLoop 线程，直接执行
      doSomething();
  } else {
      // 在外部线程，投递 Task
      eventLoop.execute(() -> doSomething());
  }
```

#### 7.3.2 FastThreadLocal

```java
// Netty 自己实现的 ThreadLocal，比 JDK ThreadLocal 快
// JDK ThreadLocal 使用线性探测哈希表，有哈希碰撞
// FastThreadLocal 使用数组+下标，直接定位，O(1)

// 使用方式（与 JDK ThreadLocal 相同）
private static final FastThreadLocal<ByteBuf> BUFFER = new FastThreadLocal<ByteBuf>() {
    @Override
    protected ByteBuf initialValue() {
        return Unpooled.buffer(1024);
    }

    @Override
    protected void onRemoval(ByteBuf value) throws Exception {
        value.release(); // 清理时释放内存
    }
};

// 读取
ByteBuf buf = BUFFER.get();

// 重要：必须在 FastThreadLocalThread（或其子类）中使用才有性能优势
// NioEventLoop 使用的线程就是 FastThreadLocalThread
```

#### 7.3.3 HashedWheelTimer（时间轮）

```java
// 适用于大量延迟任务的场景（如心跳超时检测）
// 时间复杂度 O(1)，比 JDK ScheduledExecutorService 高效

HashedWheelTimer timer = new HashedWheelTimer(
    100,              // tickDuration: 每格100ms
    TimeUnit.MILLISECONDS,
    512               // ticksPerWheel: 轮盘格子数
);

// 添加延迟任务
Timeout timeout = timer.newTimeout(t -> {
    System.out.println("5秒后执行");
}, 5, TimeUnit.SECONDS);

// 取消任务
timeout.cancel();

// 停止时间轮
timer.stop();
```

```
时间轮示意图（ticksPerWheel=8）：

      0       1       2       3
  ┌───────┬───────┬───────┬───────┐
  │ Task1 │       │ Task2 │       │
  └───────┴───────┴───────┴───────┘
      4       5       6       7
  ┌───────┬───────┬───────┬───────┐
  │       │ Task3 │       │ Task4 │
  └───────┴───────┴───────┴───────┘
           ↑
         当前指针（每 tick 移动一格）
```

#### 7.3.4 对象池（Recycler）

```java
// Netty 使用 Recycler 实现对象池，减少 GC 压力
// 内部使用 Stack + WeakOrderQueue 实现无锁回收

public class MyObject {
    // Recycler 实例（static final）
    private static final Recycler<MyObject> RECYCLER = new Recycler<MyObject>() {
        @Override
        protected MyObject newObject(Handle<MyObject> handle) {
            return new MyObject(handle);
        }
    };

    private final Recycler.Handle<MyObject> handle;

    private MyObject(Recycler.Handle<MyObject> handle) {
        this.handle = handle;
    }

    // 获取对象（从对象池中取或新建）
    public static MyObject newInstance() {
        return RECYCLER.get();
    }

    // 使用完后回收
    public void recycle() {
        handle.recycle(this);
    }
}

// 使用示例
MyObject obj = MyObject.newInstance();
try {
    // 使用 obj 处理业务
} finally {
    obj.recycle(); // 回收到对象池
}
```

---

### 7.4 Netty 内存泄漏检测

```java
// Netty 内置内存泄漏检测机制（基于 WeakReference + ReferenceQueue）
// 检测级别：
// DISABLED → SIMPLE → ADVANCED → PARANOID

// JVM 参数设置检测级别
// -Dio.netty.leakDetectionLevel=advanced

// 代码设置
ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.PARANOID); // 开发/测试环境
ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.SIMPLE);   // 生产环境

// 泄漏日志示例：
// LEAK: ByteBuf.release() was not called before it's garbage-collected.
// Recent access records:
// #1: ...
// Created at: ...

// 常见泄漏场景及修复
// 1. 在 ChannelInboundHandlerAdapter.channelRead() 中忘记调用 release()
public class LeakFixHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            try {
                // 处理数据...
                String data = buf.toString(CharsetUtil.UTF_8);
                ctx.writeAndFlush(Unpooled.copiedBuffer("OK", CharsetUtil.UTF_8));
            } finally {
                buf.release(); // ← 关键！必须在 finally 中释放
            }
        }
    }
}

// 2. 使用 SimpleChannelInboundHandler 自动释放（推荐）
public class AutoReleaseHandler extends SimpleChannelInboundHandler<ByteBuf> {
    @Override
    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) {
        // 方法返回后自动调用 release()，无需手动释放
        String data = msg.toString(CharsetUtil.UTF_8);
        ctx.writeAndFlush(Unpooled.copiedBuffer("OK", CharsetUtil.UTF_8));
    }
}
```

---

---

## 8. Netty 在主流框架中的应用

### 8.1 Dubbo 中的 Netty

```
Dubbo 网络层架构：

  Consumer（消费者）                    Provider（提供者）
      │                                       │
  DubboProtocol                          DubboProtocol
      │                                       │
  HeaderExchangeClient                  HeaderExchangeServer
      │                                       │
  NettyClient ──── TCP 连接 ────────► NettyServer
      │                                       │
  NettyChannel                           NettyChannel
      │                                       │
  NettyCodecAdapter                      NettyCodecAdapter
  （编解码：Dubbo协议/HTTP2）            （编解码：Dubbo协议/HTTP2）

  Dubbo 协议帧格式：
  ┌────────┬────────┬────────┬──────────┬────────────┐
  │Magic(2)│Flag(1) │Status(1)│Request ID │ Body Length │ Body
  │0xDABB  │        │        │   (8B)    │    (4B)     │
  └────────┴────────┴────────┴──────────┴────────────┘
```

**Dubbo 中 Netty 关键配置：**

```java
// Dubbo 服务提供者配置 Netty 参数
<dubbo:provider
    threads="200"          // 业务线程池大小
    iothreads="4"          // Netty I/O 线程数（workerGroup）
    accepts="500"          // 最大连接数
    payload="8388608"      // 最大请求体大小（8MB）
/>

// Dubbo 使用 Netty 的地方：
// 1. NettyServer: 服务端 ServerBootstrap
// 2. NettyClient: 客户端 Bootstrap（带重连）
// 3. NettyChannel: 封装 io.netty.channel.Channel
// 4. NettyCodecAdapter: 集成 Dubbo 编解码器
// 5. NettyTransporter: SPI 接口实现

// 关键代码片段（Dubbo 源码简化版）
public class NettyServer extends AbstractServer {
    private ServerBootstrap bootstrap;

    @Override
    protected void doOpen() throws Throwable {
        bootstrap = new ServerBootstrap();
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup(getIOThreads());

        bootstrap.group(bossGroup, workerGroup)
            .channel(NioServerSocketChannel.class)
            .childHandler(new ChannelInitializer<NioSocketChannel>() {
                @Override
                protected void initChannel(NioSocketChannel ch) {
                    NettyCodecAdapter adapter = new NettyCodecAdapter(getCodec(), getUrl(), NettyServer.this);
                    ch.pipeline()
                        .addLast("decoder", adapter.getDecoder())
                        .addLast("encoder", adapter.getEncoder())
                        .addLast("handler", nettyServerHandler);
                }
            });

        bootstrap.bind(getUrl().getHost(), getUrl().getPort());
    }
}
```

---

### 8.2 RocketMQ 中的 Netty

```
RocketMQ 网络通信架构：

  Producer / Consumer
       │
       │ RemotingClient（NettyRemotingClient）
       │ ├── Bootstrap（单 EventLoopGroup）
       │ ├── 连接池管理
       │ └── 请求-响应匹配（ConcurrentHashMap<opaque, ResponseFuture>）
       │
       │ TCP 连接
       ▼
  Broker（NettyRemotingServer）
       ├── ServerBootstrap
       ├── bossGroup（1线程，处理ACCEPT）
       ├── workerGroup（nettyServerSelectorThreads，处理I/O）
       └── businessThreadPool（处理实际业务逻辑）

  RocketMQ RemotingCommand 格式：
  ┌──────────┬──────────┬────────────┬────────────┐
  │ 总长度(4B)│ Header长度│ Header(JSON)│  Body(NB)  │
  └──────────┴──────────┴────────────┴────────────┘
```

```java
// RocketMQ 中 Netty Pipeline 配置（简化版）
pipeline.addLast(
    new NettyEncoder(),           // 编码 RemotingCommand
    new NettyDecoder(),           // 解码 RemotingCommand
    new IdleStateHandler(0, 0, 120, TimeUnit.SECONDS), // 120秒空闲
    new NettyConnetManageHandler(),  // 连接管理
    new NettyServerHandler()         // 业务处理
);

// 异步请求-响应模式
public ResponseFuture invokeAsync(Channel channel, RemotingCommand request, long timeout,
                                   InvokeCallback callback) {
    int opaque = request.getOpaque();
    ResponseFuture responseFuture = new ResponseFuture(opaque, timeout, callback);
    responseTable.put(opaque, responseFuture); // 注册到等待响应表

    channel.writeAndFlush(request).addListener(future -> {
        if (future.isSuccess()) {
            responseFuture.setSendRequestOK(true);
        } else {
            responseTable.remove(opaque);
            responseFuture.setCause(future.cause());
            responseFuture.executeInvokeCallback();
        }
    });
    return responseFuture;
}
```

---

### 8.3 gRPC 中的 Netty

```
gRPC-Java 与 Netty 的关系：

  gRPC 应用层
       │
  gRPC Core（AbstractStub, Channel, ServerServiceDefinition）
       │
  NettyClientTransport / NettyServerTransport
       │
  HTTP/2（Netty HTTP2 codec）
       │
  Netty NIO Transport
       │
  TCP/IP

  gRPC Netty 传输层特性：
  ├── 使用 HTTP/2 多路复用（一个连接处理多个 RPC）
  ├── 流控（Flow Control）
  ├── Header 压缩（HPACK）
  └── 全双工流式通信（Streaming RPC）
```

```java
// gRPC 服务端使用 Netty（简化）
Server server = NettyServerBuilder.forPort(50051)
    .addService(new GreeterImpl())
    .bossEventLoopGroup(new NioEventLoopGroup(1))
    .workerEventLoopGroup(new NioEventLoopGroup())
    .executor(Executors.newFixedThreadPool(10)) // 业务线程池
    .maxInboundMessageSize(16 * 1024 * 1024)   // 最大消息16MB
    .build()
    .start();

// gRPC 客户端使用 Netty
ManagedChannel channel = NettyChannelBuilder.forAddress("localhost", 50051)
    .eventLoopGroup(new NioEventLoopGroup())
    .usePlaintext()   // 不使用TLS（生产应该用SSL）
    .maxInboundMessageSize(16 * 1024 * 1024)
    .build();

GreeterGrpc.GreeterBlockingStub stub = GreeterGrpc.newBlockingStub(channel);
HelloReply reply = stub.sayHello(HelloRequest.newBuilder().setName("Netty").build());
```

---

### 8.4 Spring WebFlux 中的 Netty

```
Spring WebFlux 与 Netty：

  HTTP 请求
       │
  ReactorHttpHandlerAdapter
       │
  HttpWebHandlerAdapter
       │
  DispatcherHandler（WebFlux）
       │
  ├── RouterFunction（函数式路由）
  └── @Controller（注解式控制器）
       │
  ReactorNettyHttpServer
       │
  Netty（reactor-netty 封装）
```

```java
// Spring Boot WebFlux 使用 Netty 作为默认服务器
// application.properties
// server.port=8080
// spring.main.web-application-type=reactive

// 配置 Netty 参数
@Configuration
public class NettyConfig {

    @Bean
    public NettyReactiveWebServerFactory nettyReactiveWebServerFactory() {
        NettyReactiveWebServerFactory factory = new NettyReactiveWebServerFactory();
        factory.addServerCustomizers(httpServer -> httpServer
            .option(ChannelOption.SO_BACKLOG, 1024)
            .childOption(ChannelOption.SO_KEEPALIVE, true)
            .childOption(ChannelOption.TCP_NODELAY, true)
        );
        return factory;
    }
}

// WebFlux Controller
@RestController
@RequestMapping("/api")
public class ReactiveController {

    @GetMapping("/stream")
    public Flux<String> stream() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(i -> "消息 " + i)
            .take(10);
    }

    @PostMapping("/echo")
    public Mono<String> echo(@RequestBody Mono<String> body) {
        return body.map(text -> "Echo: " + text);
    }
}
```

---

### 8.5 Elasticsearch 中的 Netty

```
Elasticsearch 网络层（基于 Netty）：

  ┌─────────────────────────────────────────────────┐
  │  Elasticsearch 节点                              │
  │                                                 │
  │  ┌─────────────────┐  ┌─────────────────────┐  │
  │  │  HTTP Transport  │  │  Transport（节点通信）│  │
  │  │  （REST API）    │  │  （集群内部通信）    │  │
  │  │  Netty HTTP      │  │  Netty TCP           │  │
  │  │  端口：9200       │  │  端口：9300           │  │
  │  └─────────────────┘  └─────────────────────┘  │
  └─────────────────────────────────────────────────┘

  ES 中的 Netty 特点：
  ├── 自定义线程池（避免在 Netty I/O 线程中做耗时操作）
  ├── 页缓存（PageCacheRecycler）类似于 Netty 内存池
  └── 使用 Netty 的 ByteBuf 作为数据传输容器
```

---

---

## 9. 性能调优

### 9.1 线程数配置

```java
// BossGroup：通常1个线程就够（只处理 ACCEPT）
EventLoopGroup bossGroup = new NioEventLoopGroup(1);

// WorkerGroup：推荐 CPU核数 * 2
int cpuCores = Runtime.getRuntime().availableProcessors();
EventLoopGroup workerGroup = new NioEventLoopGroup(cpuCores * 2);

// 业务线程池：避免在 I/O 线程中执行耗时操作
EventExecutorGroup businessGroup = new DefaultEventExecutorGroup(
    cpuCores * 4,                           // 线程数
    new DefaultThreadFactory("business")    // 线程名前缀
);

// 在 Pipeline 中为耗时 Handler 指定业务线程池
pipeline.addLast(businessGroup, "businessHandler", new MyBusinessHandler());
// 这样 MyBusinessHandler 的回调会在 businessGroup 的线程中执行
// 而不占用 Netty I/O 线程

// 何时使用业务线程池：
// ✓ 数据库查询
// ✓ RPC 调用
// ✓ 磁盘 I/O
// ✓ 耗时计算（>1ms）
// ✗ 纯内存操作（直接在 EventLoop 线程中执行）
```

---

### 9.2 ChannelOption 参数详解

```java
ServerBootstrap b = new ServerBootstrap();

// 服务端 ServerSocketChannel 参数
b.option(ChannelOption.SO_BACKLOG, 1024)
    // TCP 全连接队列大小（第三次握手后等待 accept() 的连接队列）
    // 建议值：根据 QPS 和 accept() 处理速度设置，通常 1024-8192

b.option(ChannelOption.SO_REUSEADDR, true)
    // 允许端口重用（TIME_WAIT 状态下可以立即重启）

// 子 Channel（SocketChannel）参数
b.childOption(ChannelOption.SO_KEEPALIVE, true)
    // TCP KeepAlive，保活探测（默认间隔2小时，建议配合 IdleStateHandler 使用）

b.childOption(ChannelOption.TCP_NODELAY, true)
    // 禁用 Nagle 算法（小包立即发送，降低延迟）
    // 对于延迟敏感的应用（如游戏、IM）必须开启

b.childOption(ChannelOption.SO_SNDBUF, 65536)
    // TCP 发送缓冲区大小（字节），64KB
    // 默认值由系统决定（Linux：4KB-512KB）

b.childOption(ChannelOption.SO_RCVBUF, 65536)
    // TCP 接收缓冲区大小，64KB

b.childOption(ChannelOption.SO_LINGER, 0)
    // close() 时的行为：0=立即发RST，>0=等待N秒, -1=默认（四次挥手）

b.childOption(ChannelOption.WRITE_BUFFER_WATER_MARK,
    new WriteBufferWaterMark(32 * 1024, 64 * 1024))
    // 写缓冲区水位线
    // 低水位（32KB）：Channel 变为可写，触发 channelWritabilityChanged(true)
    // 高水位（64KB）：Channel 变为不可写，触发 channelWritabilityChanged(false)
    // 用于背压控制，防止写缓冲区无限增长导致OOM

b.childOption(ChannelOption.RCVBUF_ALLOCATOR,
    new AdaptiveRecvByteBufAllocator(64, 1024, 65536))
    // 自适应接收缓冲区分配器
    // min=64, initial=1024, max=65536
    // 根据实际接收数据量动态调整 ByteBuf 大小

b.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT)
    // 使用池化内存分配器（默认已是池化，此处为显式指定）

b.childOption(ChannelOption.AUTO_READ, true)
    // 是否自动读取数据
    // 设为 false 可暂停读取（配合流控使用）
```

---

### 9.3 背压（Back Pressure）控制

```java
// 当写入速度大于发送速度时，Channel 写缓冲区会堆积
// 使用高低水位线实现背压

@ChannelHandler.Sharable
public class BackPressureHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelWritabilityChanged(ChannelHandlerContext ctx) {
        Channel channel = ctx.channel();

        if (!channel.isWritable()) {
            // 写缓冲区超过高水位，暂停读取
            System.out.println("写缓冲区满，停止读取");
            channel.config().setAutoRead(false);
        } else {
            // 写缓冲区降到低水位，恢复读取
            System.out.println("写缓冲区恢复，继续读取");
            channel.config().setAutoRead(true);
        }

        ctx.fireChannelWritabilityChanged();
    }

    // 发送前检查是否可写
    public static void writeIfPossible(Channel channel, Object msg) {
        if (channel.isWritable()) {
            channel.writeAndFlush(msg);
        } else {
            // 丢弃消息或缓存到业务队列
            System.out.println("通道不可写，消息丢弃或入队：" + msg);
            ReferenceCountUtil.release(msg);
        }
    }
}
```

---

### 9.4 ByteBuf 优化建议

```java
// 1. 优先使用 Direct ByteBuf（I/O 操作减少一次内存拷贝）
bootstrap.childOption(ChannelOption.ALLOCATOR, PooledByteBufAllocator.DEFAULT);
// 注意：PooledByteBufAllocator 默认根据是否有 Unsafe 决定使用 Direct

// 2. 避免频繁创建 ByteBuf，使用 alloc() 从池中获取
ByteBuf buf = ctx.alloc().buffer(256); // 不要使用 Unpooled 除非特殊情况

// 3. 合理设置初始容量，减少扩容次数
// 扩容策略：当容量不足时，按4MB为界：小于4MB倍增，大于4MB每次增加4MB
ByteBuf buf2 = ctx.alloc().buffer(1024); // 根据预期数据大小设置

// 4. 避免在 Handler 中保留 ByteBuf 引用（除非 retain() 了）
// 错误示例：
List<ByteBuf> bufs = new ArrayList<>();
// channelRead 中：
bufs.add((ByteBuf) msg); // 错误！msg 会被释放

// 正确示例：
// channelRead 中：
ByteBuf data = ((ByteBuf) msg).copy(); // 深拷贝后保存
// 或者：
((ByteBuf) msg).retain();  // 增加引用计数
bufs.add((ByteBuf) msg);
// 使用完后：
bufs.forEach(ByteBuf::release);

// 5. 使用 discardReadBytes() 回收已读空间（谨慎使用，会触发内存拷贝）
// 只有当浪费空间超过一半时才值得调用
if (buf.readerIndex() > buf.capacity() / 2) {
    buf.discardReadBytes();
}
```

---

### 9.5 连接数优化

```java
// 1. 限制最大连接数（防止 OOM）
public class ConnectionLimitHandler extends ChannelInboundHandlerAdapter {

    private static final AtomicInteger connectionCount = new AtomicInteger(0);
    private static final int MAX_CONNECTIONS = 10000;

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        int count = connectionCount.incrementAndGet();
        if (count > MAX_CONNECTIONS) {
            connectionCount.decrementAndGet();
            ctx.close();
            System.out.println("连接数超限，拒绝连接：" + ctx.channel().remoteAddress());
        } else {
            System.out.println("当前连接数：" + count);
            ctx.fireChannelActive();
        }
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        connectionCount.decrementAndGet();
        ctx.fireChannelInactive();
    }
}

// 2. 设置 SO_BACKLOG 防止半连接队列溢出
b.option(ChannelOption.SO_BACKLOG, 4096);

// 3. Linux 内核参数调优
// net.core.somaxconn=65535           # 全连接队列上限
// net.ipv4.tcp_max_syn_backlog=8096  # 半连接队列上限
// net.core.netdev_max_backlog=10000  # 网卡接收队列长度
// net.ipv4.tcp_syn_retries=2         # SYN 重传次数
// net.ipv4.tcp_synack_retries=2      # SYN-ACK 重传次数
// ulimit -n 1048576                  # 文件描述符上限
```

---

### 9.6 GC 调优建议

```
Netty GC 相关优化建议：

  ① 使用 Direct Memory（堆外内存）减少 GC 压力
     -Xmx4g -XX:MaxDirectMemorySize=4g
     注意：堆外内存也需要监控，泄漏同样致命

  ② 启用池化内存分配器，减少 GC 频率
     -Dio.netty.allocator.type=pooled

  ③ G1GC 适合大堆（>4GB）
     -XX:+UseG1GC
     -XX:G1HeapRegionSize=4m
     -XX:MaxGCPauseMillis=200

  ④ ZGC（低延迟，JDK 15+，推荐高并发场景）
     -XX:+UseZGC
     -Xmx16g

  ⑤ 减少对象创建
     ├── 使用 FastThreadLocal 代替 ThreadLocal
     ├── 使用 Recycler 对象池
     └── 复用 StringBuilder、byte[] 等

  ⑥ 监控指标
     ├── GC 频率和停顿时间（JVM GC 日志）
     ├── 堆外内存使用量（BufferPoolMXBean）
     └── Netty 内存池使用（PooledByteBufAllocatorMetric）
```

---

### 9.7 Epoll Transport（Linux 专属优化）

```java
// 在 Linux 上使用 Epoll Transport，性能优于 NIO
// 需要引入依赖：netty-transport-native-epoll

// 自动选择最优 Transport
EventLoopGroup bossGroup;
EventLoopGroup workerGroup;
Class<? extends ServerChannel> channelClass;

if (Epoll.isAvailable()) {
    // Linux，使用 epoll
    bossGroup = new EpollEventLoopGroup(1);
    workerGroup = new EpollEventLoopGroup();
    channelClass = EpollServerSocketChannel.class;
    System.out.println("使用 Epoll Transport");
} else {
    // 其他平台，使用 NIO
    bossGroup = new NioEventLoopGroup(1);
    workerGroup = new NioEventLoopGroup();
    channelClass = NioServerSocketChannel.class;
    System.out.println("使用 NIO Transport");
}

ServerBootstrap b = new ServerBootstrap();
b.group(bossGroup, workerGroup)
 .channel(channelClass)
 // Epoll 专有参数
 .childOption(EpollChannelOption.EPOLL_MODE, EpollMode.EDGE_TRIGGERED) // ET 模式
 .childOption(EpollChannelOption.TCP_KEEPIDLE, 60)   // 60s 后开始探活
 .childOption(EpollChannelOption.TCP_KEEPINTVL, 10)  // 探活间隔10s
 .childOption(EpollChannelOption.TCP_KEEPCNT, 3);    // 失败3次断开
```

---

---

## 10. 常见问题排查

### 10.1 内存泄漏排查

```
内存泄漏排查步骤：

  Step 1: 开启泄漏检测
  -Dio.netty.leakDetectionLevel=advanced

  Step 2: 观察日志
  LEAK: ByteBuf.release() was not called before...

  Step 3: 分析堆外内存增长
  jcmd <pid> VM.native_memory
  # 或使用 pmap / smaps

  Step 4: 常见原因检查列表
  ✗ ChannelInboundHandlerAdapter.channelRead() 未调用 release()
  ✗ 将 ByteBuf 传入线程池后未 retain()
  ✗ 异常路径跳过了 release()
  ✗ CompositeByteBuf 添加组件后未正确释放
  ✗ 在 Channel 关闭后仍保留 ByteBuf 引用
```

```java
// 常见泄漏场景修复

// 场景1：异常路径遗漏
// 错误：
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf buf = (ByteBuf) msg;
    if (someCondition) {
        throw new RuntimeException("error"); // buf 未释放！
    }
    // ...
    buf.release();
}

// 正确：
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ByteBuf buf = (ByteBuf) msg;
    try {
        if (someCondition) {
            throw new RuntimeException("error");
        }
        // ...
    } finally {
        buf.release(); // 无论如何都会释放
    }
}

// 场景2：投递到线程池前忘记 retain
// 错误：
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    executor.submit(() -> {
        ByteBuf buf = (ByteBuf) msg; // msg 可能已经被释放！
        process(buf);
    });
    // channelRead 返回后，msg 可能被释放
}

// 正确：
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    ((ByteBuf) msg).retain(); // 增加引用计数
    executor.submit(() -> {
        ByteBuf buf = (ByteBuf) msg;
        try {
            process(buf);
        } finally {
            buf.release(); // 线程池任务完成后释放
        }
    });
}
```

---

### 10.2 CPU 空转（Epoll Bug）

```
JDK NIO Epoll Bug：

  表现：Selector.select() 不断返回 0（没有事件但立即返回）
        导致线程 CPU 占用 100%

  历史：JDK 6/7 上存在，JDK 8 有改善但不彻底

  Netty 的解决方案（NioEventLoop 源码）：

  ① 统计 select() 空转次数（SELECTOR_AUTO_REBUILD_THRESHOLD，默认512）
  ② 超过阈值时，重建 Selector
  ③ 将旧 Selector 上的所有 Channel 注册到新 Selector
  ④ 旧 Selector 关闭

  关键代码（NioEventLoop.run()简化）：
```

```java
// Netty 防 Epoll Bug 机制（简化版）
private void rebuildSelector() {
    Selector oldSelector = selector;
    Selector newSelector = openSelector();

    // 将所有 Channel 从旧 Selector 迁移到新 Selector
    for (SelectionKey key : oldSelector.keys()) {
        Object attachment = key.attachment();
        int ops = key.interestOps();
        key.cancel();
        key.channel().register(newSelector, ops, attachment);
    }

    selector = newSelector;
    oldSelector.close();
    System.out.println("Selector 已重建（Epoll Bug 规避）");
}

// 检测空转（NioEventLoop.run() 中的逻辑）
int selectCnt = 0;
for (;;) {
    int selectedKeys = selector.select(timeoutNanos);
    selectCnt++;

    if (selectedKeys != 0 || hasTasks()) {
        selectCnt = 0; // 有事件，重置计数
        processSelectedKeys();
        runAllTasks();
    } else if (selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {
        rebuildSelector();
        selectCnt = 0;
    }
}
```

---

### 10.3 连接数过多导致 OOM

```
排查步骤：

  1. 查看当前连接数
  ss -s              # 查看 TCP 连接统计
  ss -nt | wc -l     # 当前连接总数
  netstat -an | grep ESTABLISHED | wc -l

  2. 查看 TIME_WAIT 是否过多
  netstat -an | grep TIME_WAIT | wc -l
  # TIME_WAIT 过多可调整：
  # net.ipv4.tcp_tw_reuse=1
  # net.ipv4.tcp_fin_timeout=30

  3. 查看文件描述符限制
  ulimit -n          # 当前进程限制
  cat /proc/sys/fs/file-max  # 系统级限制

  4. 调整文件描述符
  ulimit -n 1048576  # 临时修改
  # /etc/security/limits.conf 永久修改

  5. 代码层面检查
  - 是否及时关闭不需要的连接
  - IdleStateHandler 是否正确配置
  - 是否有连接池泄漏
```

---

### 10.4 消息发送慢/堆积

```
排查方向：

  1. 写缓冲区是否堆积（监控 channel.bytesBeforeUnwritable()）
  2. 是否在 I/O 线程中执行了耗时操作
  3. 接收方处理慢，导致 TCP 窗口关闭
  4. 网络带宽是否打满

  常见代码问题：
```

```java
// 问题：在 I/O 线程中执行数据库查询
public void channelRead(ChannelHandlerContext ctx, Object msg) {
    // 错误：此时在 EventLoop 线程，会阻塞所有 Channel 的 I/O！
    User user = userDao.findById(userId); // 数据库查询，可能耗时数百ms
    ctx.writeAndFlush(user);
}

// 修复：使用独立业务线程池
private static final ExecutorService DB_POOL = Executors.newFixedThreadPool(50);

public void channelRead(ChannelHandlerContext ctx, Object msg) {
    DB_POOL.submit(() -> {
        User user = userDao.findById(userId); // 在业务线程中执行
        ctx.writeAndFlush(user); // 写回操作线程安全（Netty 会投递到 EventLoop）
    });
}

// 或者在 Pipeline 中为 Handler 指定 EventExecutorGroup
pipeline.addLast(businessExecutors, "dbHandler", new DbHandler());

// 监控写缓冲区积压
@Override
public void channelWritabilityChanged(ChannelHandlerContext ctx) {
    Channel channel = ctx.channel();
    long pendingBytes = channel.unsafe().outboundBuffer().totalPendingWriteBytes();
    System.out.println("写缓冲区积压：" + pendingBytes + " bytes");

    if (!channel.isWritable()) {
        System.out.println("流量控制：暂停读取");
        channel.config().setAutoRead(false);
    } else {
        channel.config().setAutoRead(true);
    }
}
```

---

### 10.5 Handler 执行顺序异常

```java
// 常见错误：Pipeline 中 Handler 顺序不当

// 错误示例（解码器在业务 Handler 之后）：
pipeline.addLast(new MyBusinessHandler());  // 错误！收到的是 ByteBuf，不是业务对象
pipeline.addLast(new LengthFieldBasedFrameDecoder(...));
pipeline.addLast(new MyDecoder());

// 正确示例：
pipeline.addLast(new LengthFieldBasedFrameDecoder(...)); // 1. 先拆包
pipeline.addLast(new MyDecoder());                        // 2. 再解码
pipeline.addLast(new MyBusinessHandler());                // 3. 最后业务处理

// 出站顺序（从 Tail 到 Head）：
// Handler1 → Handler2 → Handler3(Encoder) → Head
// 注意：ctx.write() 从当前位置向前找 OutboundHandler
// channel.write() 从 Tail 开始向前找

// 查看 Pipeline 当前结构（调试用）
channel.pipeline().forEach(entry ->
    System.out.println(entry.getKey() + " -> " + entry.getValue().getClass().getSimpleName())
);
```

---

### 10.6 常见异常及处理

```java
// 1. java.io.IOException: Connection reset by peer
// 原因：对端强制关闭了连接（发送 RST）
// 处理：在 exceptionCaught 中关闭连接，通常无需额外处理

// 2. java.nio.channels.ClosedChannelException
// 原因：向已关闭的 Channel 写数据
// 处理：写操作前判断 channel.isActive()

// 3. io.netty.handler.codec.TooLongFrameException
// 原因：消息超过 maxFrameLength 限制
// 处理：增大 maxFrameLength 或检查数据合法性

// 4. io.netty.util.internal.OutOfDirectMemoryError
// 原因：堆外内存耗尽（内存泄漏或 MaxDirectMemorySize 设置过小）
// 处理：开启泄漏检测，增大 -XX:MaxDirectMemorySize

@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
    if (cause instanceof IOException) {
        // 网络异常，通常是对端断开，记录日志但不打印堆栈
        System.out.println("网络异常，关闭连接：" + cause.getMessage());
    } else if (cause instanceof TooLongFrameException) {
        // 非法消息，关闭连接
        System.out.println("消息过长，关闭连接");
        ctx.close();
    } else {
        // 其他未知异常，打印堆栈并关闭连接
        cause.printStackTrace();
        ctx.close();
    }
}
```

---

---

## 11. 常见面试题 FAQ

### Q1：Netty 是什么？为什么不直接使用 JDK NIO？

**答：**

Netty 是一个基于 Java NIO 的高性能、异步事件驱动的网络应用框架。

**直接使用 JDK NIO 的痛点：**

| 问题 | 说明 |
|------|------|
| API 复杂 | ByteBuffer 使用繁琐，需手动切换 read/write 模式，容易出错 |
| Epoll Bug | JDK NIO 在 Linux 下存在 Selector 空轮询 Bug，导致 CPU 100% |
| 粘包拆包 | 需要自己处理 TCP 流的粘包/拆包问题 |
| 无连接管理 | 需要自己管理连接断开、重连、心跳检测 |
| 性能差 | 没有内存池，ByteBuffer 每次分配新内存 |
| 编解码 | 需要自己实现各种协议的编解码 |

**Netty 的解决方案：**
- ByteBuf：比 ByteBuffer 更易用，支持动态扩容，支持内存池
- 自动规避 Epoll Bug（重建 Selector）
- 内置各种编解码器（长度字段、分隔符、HTTP、WebSocket...）
- IdleStateHandler 处理心跳和超时
- PooledByteBufAllocator 高效内存管理

---

### Q2：Netty 的线程模型是什么？

**答：**

Netty 采用**主从 Reactor 多线程模型**：

```
BossGroup（主 Reactor）
├── 通常1个线程
├── 运行 Selector，监听 OP_ACCEPT 事件
├── 负责建立连接（三次握手）
└── 将建立好的 Channel 注册到 WorkerGroup

WorkerGroup（从 Reactor）
├── 默认 CPU核数 * 2 个线程
├── 每个线程运行一个 Selector
├── 负责监听 OP_READ / OP_WRITE 事件
└── 在 ChannelPipeline 中执行 Handler 链

业务线程池（可选）
└── 处理耗时的业务逻辑，避免阻塞 I/O 线程
```

**关键特性：**
1. 每个 Channel 绑定唯一 EventLoop（整个生命周期不变）
2. 单个 EventLoop 线程处理其上所有 Channel 的事件（无锁化）
3. 外部线程通过 TaskQueue 提交任务给 EventLoop

---

### Q3：Netty 如何解决粘包/拆包问题？

**答：**

TCP 是流式协议，没有消息边界。Netty 提供4种解决方案：

1. **FixedLengthFrameDecoder**：固定长度，每条消息 N 字节
2. **DelimiterBasedFrameDecoder**：特殊分隔符（如 `\n`）标识消息结束
3. **LengthFieldBasedFrameDecoder**：消息头包含 Body 长度字段（最通用）
4. **自定义**：继承 `ByteToMessageDecoder` 实现自定义拆包逻辑

最常用的是 `LengthFieldBasedFrameDecoder`，配合 `LengthFieldPrepender` 使用。

---

### Q4：Netty 的零拷贝体现在哪些地方？

**答：**

Netty 的零拷贝分为两层：

**操作系统层面（真正的零拷贝）：**
- 使用 `FileRegion`（底层调用 `sendfile`），文件数据不经过用户空间直接从内核发送到网卡

**用户空间层面（减少内存拷贝）：**
- `CompositeByteBuf`：逻辑合并多个 ByteBuf，不拷贝数据
- `ByteBuf.slice()`：切片共享底层内存，不拷贝
- `ByteBuf.duplicate()`：复制视图，不拷贝数据
- `Unpooled.wrappedBuffer(byte[])`：包装 byte[]，不拷贝
- `DirectByteBuf`：堆外内存，减少 JVM→内核的拷贝

---

### Q5：EventLoop 线程如何保证线程安全？

**答：**

Netty 通过以下机制保证线程安全：

1. **单线程执行**：每个 EventLoop 只有一个线程，Channel 的所有事件都在这个线程中串行处理

2. **任务队列**：非 EventLoop 线程的操作被封装成 `Runnable` 投递到 `taskQueue`，由 EventLoop 线程串行执行

3. **inEventLoop 判断**：
```java
if (eventLoop.inEventLoop()) {
    // 当前就是 EventLoop 线程，直接执行
    doSomething();
} else {
    // 外部线程，提交任务
    eventLoop.execute(() -> doSomething());
}
```

4. **Pipeline 链的无锁化**：Handler 中的状态只在 EventLoop 线程访问，无需加锁（前提是 Handler 没有 `@Sharable` 且有状态）

---

### Q6：ByteBuf 与 JDK ByteBuffer 的区别？

**答：**

| 对比项 | JDK ByteBuffer | Netty ByteBuf |
|--------|----------------|---------------|
| 读写指针 | 共用 position（需 flip） | readerIndex + writerIndex 分离 |
| 扩容 | 不支持，需手动创建更大的 | 自动扩容（到 maxCapacity） |
| 内存池 | 无 | PooledByteBufAllocator（默认） |
| 引用计数 | 无 | 有，支持手动管理生命周期 |
| 零拷贝 | 有限支持 | CompositeByteBuf、slice、wrap 等 |
| 类型读写 | 手动处理 | readInt/readLong/readBytes 等 |
| 堆外内存 | DirectByteBuffer | DirectByteBuf（更好用） |
| API 易用性 | 繁琐 | 链式调用，更易用 |

---

### Q7：ChannelPipeline 中 Handler 的执行顺序是什么？

**答：**

Pipeline 是**双向链表**，有 HeadContext 和 TailContext 两个哨兵节点。

- **Inbound 事件**（如 channelRead）：从 HeadContext → 向后传播 → TailContext
- **Outbound 操作**（如 write）：从 TailContext → 向前传播 → HeadContext

```
数据入站：
  I/O → HeadContext → Decoder → BusinessHandler → TailContext

数据出站：
  TailContext → Encoder → HeadContext → I/O
```

**注意：**
- `ctx.fireChannelRead(msg)` ：从当前节点向后传播（Inbound）
- `ctx.write(msg)` ：从当前节点向前传播（Outbound）
- `channel.write(msg)` ：从 Tail 开始向前传播（经过所有 Handler）

---

### Q8：Netty 如何实现心跳检测？

**答：**

使用 `IdleStateHandler` + 自定义处理器：

```java
// 1. 添加 IdleStateHandler
pipeline.addLast(new IdleStateHandler(60, 30, 0, TimeUnit.SECONDS));
// readerIdleTime=60s：60秒没收到数据
// writerIdleTime=30s：30秒没发送数据
// allIdleTime=0：禁用

// 2. 在自定义 Handler 中处理 IdleStateEvent
@Override
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
    if (evt instanceof IdleStateEvent) {
        IdleStateEvent e = (IdleStateEvent) evt;
        if (e.state() == IdleState.READER_IDLE) {
            ctx.close(); // 服务端：超时关闭连接
        } else if (e.state() == IdleState.WRITER_IDLE) {
            ctx.writeAndFlush(HEARTBEAT_PING); // 客户端：发送心跳
        }
    }
}
```

---

### Q9：Netty 的 ChannelHandler 是线程安全的吗？

**答：**

**取决于 Handler 的设计：**

1. **无状态 Handler**（`@Sharable`）：可以安全地被多个 Channel 共享，因为没有实例变量
2. **有状态 Handler**（无 `@Sharable`）：每个 Channel 应创建独立实例，不能共享

**关键原则：**
- 同一个 Channel 的事件总在同一个 EventLoop 线程中处理（串行），Handler 内不需要同步
- 如果 Handler 被多个 Channel 共享（`@Sharable`），且有状态变量，必须用线程安全的数据结构

```java
// 正确：有状态 Handler 每次创建新实例
pipeline.addLast(new StatefulHandler()); // 每次 new

// 正确：无状态 Handler 可以复用同一实例
@Sharable
public class StatelessHandler extends ChannelInboundHandlerAdapter { ... }
private static final StatelessHandler INSTANCE = new StatelessHandler();
pipeline.addLast(INSTANCE); // 复用
```

---

### Q10：Netty 服务端启动流程？

**答：**

```
1. 创建 ServerBootstrap，配置 BossGroup 和 WorkerGroup

2. 调用 bind(port) 触发启动流程：
   ├── channelFactory.newChannel()
   │   └── 创建 NioServerSocketChannel
   │       └── 创建 Java NIO ServerSocketChannel
   ├── init(channel)
   │   └── 设置 options/attrs
   │   └── 向 Pipeline 添加 ServerBootstrapAcceptor
   └── config().group().register(channel)
       └── 将 Channel 注册到 BossGroup 的 EventLoop
       └── 调用 SelectableChannel.register(selector, 0)（初始 interestOps=0）

3. 注册完成后，触发 channelRegistered 事件

4. 执行 bind：
   └── channel.bind(localAddress)
       └── AbstractUnsafe.bind()
           └── javaChannel.bind(localAddress)（系统调用 bind + listen）
           └── 修改 interestOps 添加 OP_ACCEPT

5. ChannelFuture 完成，服务启动完毕，等待客户端连接
```

---

### Q11：Netty 如何优雅关闭？

**答：**

```java
// 优雅关闭步骤：
EventLoopGroup bossGroup = new NioEventLoopGroup();
EventLoopGroup workerGroup = new NioEventLoopGroup();

// ... 启动服务 ...

// 关闭时：
// 1. 停止接受新连接（先关闭 Boss）
// 2. 等待所有 Channel 的事件处理完成
// 3. 关闭 Worker 线程

Future<?> bossFuture = bossGroup.shutdownGracefully(
    100,         // quietPeriod：100ms 内无新任务则关闭
    3000,        // timeout：最多等待 3s 强制关闭
    TimeUnit.MILLISECONDS
);
Future<?> workerFuture = workerGroup.shutdownGracefully(100, 3000, TimeUnit.MILLISECONDS);

bossFuture.sync();
workerFuture.sync();
System.out.println("Netty 已优雅关闭");
```

---

### Q12：Netty 中如何进行连接重连？

**答：**

```java
public class ReconnectHandler extends ChannelInboundHandlerAdapter {

    private final Bootstrap bootstrap;
    private final String host;
    private final int port;
    private volatile Channel channel;

    public ReconnectHandler(Bootstrap bootstrap, String host, int port) {
        this.bootstrap = bootstrap;
        this.host = host;
        this.port = port;
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        System.out.println("连接断开，5秒后重连...");
        ctx.channel().eventLoop().schedule(this::reconnect, 5, TimeUnit.SECONDS);
    }

    private void reconnect() {
        bootstrap.connect(host, port).addListener((ChannelFutureListener) future -> {
            if (future.isSuccess()) {
                channel = future.channel();
                System.out.println("重连成功：" + host + ":" + port);
            } else {
                System.out.println("重连失败，继续重试...");
                future.channel().eventLoop().schedule(this::reconnect, 5, TimeUnit.SECONDS);
            }
        });
    }
}
```

---

### Q13：Netty 的 Channel 和 Java NIO Channel 有什么关系？

**答：**

Netty 的 `Channel` 是对 Java NIO `Channel` 的封装和增强：

| 特性 | Java NIO Channel | Netty Channel |
|------|-----------------|---------------|
| 关联 | 底层实现 | 封装并增强 NIO Channel |
| 线程绑定 | 无 | 绑定唯一 EventLoop |
| Pipeline | 无 | 有，支持 Handler 链 |
| 读写方式 | 直接操作 ByteBuffer | 通过 Pipeline 处理 ByteBuf |
| 状态查询 | 有限 | isOpen/isActive/isWritable |
| 异步 | 需要 Selector | 天然异步，返回 ChannelFuture |

Netty Channel 内部通过 `Unsafe` 接口持有 Java NIO `SelectableChannel` 引用，所有底层 I/O 操作委托给 NIO Channel 执行。

---

### Q14：Netty 中的 Promise 和 Future 是什么关系？

**答：**

```
JDK Future（只读，无法设置结果）
    └── Netty Future（添加监听器 addListener）
            └── Netty Promise（可写，setSuccess/setFailure）

关系：
  Future ≈ 承诺的凭证（消费方视角，只能查询/等待结果）
  Promise ≈ 承诺本身（生产方视角，可以设置结果）

使用场景：
  异步操作返回 ChannelFuture（Future 的子接口）
  内部实现使用 DefaultChannelPromise（Promise 的实现类）
```

---

### Q15：Netty 的 FastThreadLocal 比 JDK ThreadLocal 快在哪？

**答：**

| 对比项 | JDK ThreadLocal | Netty FastThreadLocal |
|--------|----------------|----------------------|
| 存储结构 | 线性探测哈希表 | 对象数组（InternalThreadLocalMap） |
| 查找方式 | 哈希计算 + 碰撞处理 | 数组下标直接访问（O(1)） |
| 内存布局 | 可能碰撞导致缓存行失效 | 连续数组，缓存友好 |
| 清理 | 弱引用 GC 回收 | 主动 removeAll()，提供 onRemoval 回调 |
| 适用线程 | 所有线程 | 需配合 FastThreadLocalThread（EventLoop 线程） |

FastThreadLocal 每个变量都有一个唯一的 `index`（原子递增），通过 `array[index]` 直接访问，避免了哈希冲突。

---

### Q16：如何处理 Netty 中的半关闭（Half-Close）？

**答：**

```java
// TCP 支持半关闭：一端关闭写（发送 FIN），另一端仍可写
// 默认情况下，Netty 不支持半关闭

// 开启半关闭支持
pipeline.addFirst(new HalfCloseHandler());

// 或者在 Channel 配置
ch.config().setAllowHalfClosure(true);

// 处理半关闭事件
public class HalfCloseHandler extends ChannelInboundHandlerAdapter {
    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt == ChannelInputShutdownEvent.INSTANCE) {
            // 对端关闭了写（发送了 FIN）
            System.out.println("对端停止发送数据");
            // 完成我方的写后，主动关闭
            ctx.writeAndFlush(lastMessage).addListener(ChannelFutureListener.CLOSE);
        }
    }
}
```

---

### Q17：Netty 如何实现高效的定时任务？

**答：**

Netty 提供两种定时任务机制：

**1. EventLoop 的 schedule 方法：**
```java
// 延迟执行（适合少量定时任务）
ctx.channel().eventLoop().schedule(() -> {
    System.out.println("5秒后执行");
}, 5, TimeUnit.SECONDS);

// 周期执行
ctx.channel().eventLoop().scheduleAtFixedRate(() -> {
    System.out.println("每隔1秒执行");
}, 0, 1, TimeUnit.SECONDS);
```

**2. HashedWheelTimer（大量定时任务）：**
```java
// 用于超时检测、连接回收等场景（比 ScheduledExecutorService 高效）
HashedWheelTimer timer = new HashedWheelTimer(100, TimeUnit.MILLISECONDS, 512);
Timeout timeout = timer.newTimeout(t -> cleanup(conn), 30, TimeUnit.SECONDS);
// 连接活跃时取消超时
timeout.cancel();
```

---

### Q18：Netty 支持哪些序列化协议？

**答：**

Netty 内置支持：
- **Protobuf**：`ProtobufDecoder` / `ProtobufEncoder`（二进制，高效）
- **Java Serialization**：`ObjectDecoder` / `ObjectEncoder`（不推荐，慢且存在安全问题）

第三方集成：
- **JSON**（Jackson/Gson）：自定义 `MessageToByteEncoder`
- **MessagePack**：紧凑二进制格式
- **Kryo**：高性能 Java 序列化
- **FST**：快速序列化

**推荐：Protobuf（性能最好，跨语言）**
```java
// Maven 依赖
// <dependency>
//   <groupId>io.netty</groupId>
//   <artifactId>netty-codec-protobuf</artifactId>
// </dependency>

pipeline.addLast(new ProtobufVarint32FrameDecoder());
pipeline.addLast(new ProtobufDecoder(MyProto.Message.getDefaultInstance()));
pipeline.addLast(new ProtobufVarint32LengthFieldPrepender());
pipeline.addLast(new ProtobufEncoder());
```

---

### Q19：Netty 的 Channel 状态机是什么？

**答：**

```
Channel 状态转换：

  UNREGISTERED
       │
       │ register()
       ▼
  REGISTERED ──── channelRegistered 事件
       │
       │ bind()/connect()
       ▼
    ACTIVE ─────── channelActive 事件
       │
       │ close()/disconnect()
       ▼
   INACTIVE ────── channelInactive 事件
       │
       │ deregister()
       ▼
  UNREGISTERED

  关键方法：
  channel.isRegistered() → REGISTERED 状态
  channel.isOpen()       → 未调用 close()
  channel.isActive()     → 已连接（TCP）或已绑定（UDP）
  channel.isWritable()   → 写缓冲区未超高水位
```

---

### Q20：如何在 Netty 中实现请求-响应模式（类似 RPC）？

**答：**

核心思路：使用 `sequenceId` 关联请求和响应，使用 `CompletableFuture` 或 `Promise` 等待结果。

```java
// 客户端 Handler
public class RpcClientHandler extends SimpleChannelInboundHandler<RpcResponse> {

    // 等待响应的请求表：seqId → Future
    private static final ConcurrentHashMap<Long, CompletableFuture<RpcResponse>>
        pendingMap = new ConcurrentHashMap<>();

    private static final AtomicLong seqGen = new AtomicLong();

    // 发送请求并返回 Future
    public static CompletableFuture<RpcResponse> send(Channel ch, RpcRequest req) {
        long seqId = seqGen.incrementAndGet();
        req.setSeqId(seqId);

        CompletableFuture<RpcResponse> future = new CompletableFuture<>();
        pendingMap.put(seqId, future);

        ch.writeAndFlush(req).addListener(f -> {
            if (!f.isSuccess()) {
                pendingMap.remove(seqId);
                future.completeExceptionally(f.cause());
            }
        });

        // 超时处理
        ch.eventLoop().schedule(() -> {
            CompletableFuture<RpcResponse> f2 = pendingMap.remove(seqId);
            if (f2 != null) {
                f2.completeExceptionally(new TimeoutException("RPC 超时"));
            }
        }, 5, TimeUnit.SECONDS);

        return future;
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, RpcResponse response) {
        CompletableFuture<RpcResponse> future = pendingMap.remove(response.getSeqId());
        if (future != null) {
            future.complete(response);
        }
    }
}

// 调用示例
CompletableFuture<RpcResponse> future = RpcClientHandler.send(channel, request);
RpcResponse response = future.get(5, TimeUnit.SECONDS);
```

---

### Q21：Netty 内存池的核心数据结构是什么？

**答：**

```
Netty 内存池采用 jemalloc 算法思想：

  PoolArena（内存竞技场，减少线程竞争）
    ├── 按大小分级管理：
    │   ├── Tiny（< 512B）：tinySubpagePools[32]，16B步进
    │   ├── Small（512B~8KB）：smallSubpagePools[4]
    │   ├── Normal（8KB~16MB）：PoolChunkList
    │   └── Huge（> 16MB）：直接分配，不入池
    │
    └── PoolChunkList（按使用率组织的 PoolChunk 链表）
          ├── qInit（0%~25%）
          ├── q000（1%~50%）
          ├── q025（25%~75%）
          ├── q050（50%~100%）
          ├── q075（75%~100%）
          └── q100（100%）

  PoolChunk（默认 16MB）
    └── 使用完全二叉树（buddy system）管理 2048 个 Page（8KB）

  PoolSubpage（解决内部碎片）
    └── 一个 Page（8KB）划分成多个等大小的小块

  PoolThreadCache（线程本地缓存，无锁）
    └── 每个线程维护 tiny/small/normal 三级缓存
        用于快速分配/回收，避免锁竞争
```

---

### Q22：Netty 的 write 和 writeAndFlush 有什么区别？

**答：**

```java
// write(msg)：
// 1. 将消息编码后写入 ChannelOutboundBuffer（写缓冲区）
// 2. 不会立即发送到 Socket 发送缓冲区
// 3. 适合批量写入，最后调用一次 flush

// flush()：
// 1. 将 ChannelOutboundBuffer 中所有消息写入 Socket 发送缓冲区
// 2. 触发系统调用，数据交给内核

// writeAndFlush(msg) = write(msg) + flush()
// 适合单条消息立即发送的场景

// 最佳实践：批量写入性能更好
for (MyMessage msg : messages) {
    ctx.write(msg);  // 只写入缓冲区
}
ctx.flush();         // 批量发送

// 等价于：
ctx.writeAndFlush(Unpooled.EMPTY_BUFFER); // 也可以用空消息触发 flush

// 注意：频繁调用 writeAndFlush 每次都有系统调用开销
// 高吞吐场景应积累一批后统一 flush
```

---

### Q23：如何监控 Netty 应用的性能？

**答：**

```java
// 1. Netty 内置监控：Channel 统计
ChannelTrafficShapingHandler trafficHandler = new ChannelTrafficShapingHandler(0, 0, 1000);
pipeline.addFirst(trafficHandler);

// 查看流量统计
TrafficCounter counter = trafficHandler.trafficCounter();
System.out.println("读取字节：" + counter.cumulativeReadBytes());
System.out.println("写入字节：" + counter.cumulativeWrittenBytes());
System.out.println("读取速率：" + counter.lastReadThroughput() + " B/s");

// 2. JMX 监控（通过 Micrometer）
// 注册 Netty 指标到 Prometheus
MeterRegistry registry = new PrometheusMeterRegistry(PrometheusConfig.DEFAULT);
// 监控 EventLoop 任务队列
// 监控连接数
// 监控 ByteBuf 内存使用

// 3. 内存池监控
PooledByteBufAllocator allocator = PooledByteBufAllocator.DEFAULT;
PooledByteBufAllocatorMetric metric = allocator.metric();
System.out.println("堆内存使用：" + metric.usedHeapMemory() + " bytes");
System.out.println("堆外内存使用：" + metric.usedDirectMemory() + " bytes");

// 4. 常用监控指标
// - 活跃连接数（自定义 ChannelGroup）
// - 消息处理 QPS（Counter）
// - 消息处理延迟 P99/P999（Histogram）
// - ByteBuf 泄漏告警（ResourceLeakDetector）
// - EventLoop 任务队列积压
// - GC 频率和停顿
```

---

## 附录：Netty 核心依赖

```xml
<!-- Maven 引入 Netty -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>4.1.96.Final</version>
</dependency>

<!-- 或者按需引入 -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport</artifactId>
    <version>4.1.96.Final</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-codec</artifactId>
    <version>4.1.96.Final</version>
</dependency>
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-handler</artifactId>
    <version>4.1.96.Final</version>
</dependency>

<!-- Linux epoll 优化（仅 Linux 使用） -->
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport-native-epoll</artifactId>
    <version>4.1.96.Final</version>
    <classifier>linux-x86_64</classifier>
</dependency>
```

---

## 附录：常用 JVM 参数

```bash
# 内存配置
-Xmx4g -Xms4g
-XX:MaxDirectMemorySize=4g          # 堆外内存上限

# GC 配置（推荐 G1 或 ZGC）
-XX:+UseG1GC
-XX:G1HeapRegionSize=4m
-XX:MaxGCPauseMillis=200
-XX:+PrintGCDetails -XX:+PrintGCDateStamps
-Xloggc:/var/log/app/gc.log

# Netty 配置
-Dio.netty.leakDetectionLevel=simple           # 生产环境内存泄漏检测
-Dio.netty.allocator.type=pooled               # 池化内存分配器
-Dio.netty.recycler.maxCapacityPerThread=4096  # 对象池每线程最大容量
-Dio.netty.noUnsafe=false                      # 允许使用 Unsafe（性能更好）

# 线程配置
-Dio.netty.eventLoopThreads=16      # EventLoop 线程数（workerGroup）
```

---

*文档涵盖 Netty 4.x 版本，内容包含基础原理、核心组件、实战案例、性能调优与面试高频知识点。*
