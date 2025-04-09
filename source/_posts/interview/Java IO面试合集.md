---
title: Java IO面试合集	
category: interview
---

# IO流（BIO）

根据数据传输方式，可以将 IO 类分为:

* **字节流**：字节流读取单个字节，字节流用来处理二进制文件(图片、MP3、视频文件)。

  <img src="https://pdai.tech/images/io/java-io-category-1.png" style="zoom: 80%;" />

* **字符流**：字符流读取单个字符(一个字符根据编码的不同，对应的字节也不同，如 UTF-8 编码中文汉字是 3 个字节，GBK编码中文汉字是 2 个字节。)，字符流用来处理文本文件(可以看做是特殊的二进制文件，使用了某种编码，人可以阅读)。

  <img src="https://pdai.tech/images/io/java-io-category-2.png" style="zoom: 80%;" />

根据数据操作对象，可以将 IO 类分为:

<img src="https://pdai.tech/images/io/java-io-category-3.png" style="zoom:80%;" />

# 设计模式

> **装饰者模式（Decorator）**：把装饰者套在被装饰者之上，从而动态扩展被装饰者的功能，装饰器模式通过**组合**替代继承来扩展原始类的功能。即可以在不改变原有对象的情况下拓展其功能。

<img src="https://pdai.tech/images/pics/DP-Decorator-java.io.png" style="zoom:80%;" />

```java
BufferedInputStream bis = new BufferedInputStream(new FileInputStream(fileName));
ZipInputStream zis = new ZipInputStream(bis);

BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream(fileName));
ZipOutputStream zipOut = new ZipOutputStream(bos);
```

# 操作系统IO模型

异步IO

|            | Linux                                | MacOS    | Window |
| ---------- | ------------------------------------ | -------- | ------ |
| 事件驱动IO | `select` `poll` `epoll`              | `kqueue` | 无     |
| 异步IO     | `POSIX`<br />`io_uring` (kernel5.1+) | 无       | `IOCP` |

事件驱动IO / IO多路复用

| IO模型   | 相对性能 | 关键思路         | 操作系统      | JAVA支持情况                                                 |
| -------- | -------- | ---------------- | ------------- | ------------------------------------------------------------ |
| `select` | 较高     | Reactor          | windows/Linux | 支持,Reactor模式(反应器设计模式)。Linux操作系统的 kernels 2.4内核版本之前，默认使用select；而目前windows下对同步IO的支持，都是select模型 |
| `poll`   | 较高     | Reactor          | Linux         | Linux下的JAVA NIO框架，Linux kernels 2.6内核版本之前使用poll进行支持。也是使用的Reactor模式 |
| `epoll`  | 高       | Reactor/Proactor | Linux         | Linux kernels 2.6内核版本及以后使用epoll进行支持；Linux kernels 2.6内核版本之前使用poll进行支持；另外一定注意，由于Linux下没有Windows下的IOCP技术提供真正的 异步IO 支持，所以Linux下使用epoll模拟异步IO |
| `kqueue` | 高       | Proactor         | FreeBSD/MacOS | 目前JAVA的版本不支持                                         |

# Java IO模型

## BIO (Blocking I/O)

BIO 属于同步阻塞 IO 模型，同步阻塞 IO 模型中，应用程序发起 read 调用后，会一直阻塞，直到内核把数据拷贝到用户空间。在客户端连接数量不高的情况下，是没问题的。但是，当面对十万甚至百万级连接的时候，传统的 BIO 模型是无能为力的。因此，我们需要一种更高效的 I/O 处理模型来应对更高的并发量。

## NIO (Non-blocking/New I/O)

Java 中的 NIO 于 Java 1.4 中引入，对应 `java.nio` 包，提供了 `Channel` , `Selector`，`Buffer` 等抽象。它是支持面向缓冲的，基于通道的 I/O 操作方法。 对于高负载、高并发的（网络）应用，应使用 NIO。

Java 异步 I/O 机制：

- NIO（java.nio）：
  - 引入了非阻塞 I/O 和选择器（Selector），基于操作系统的多路复用机制（如 Linux 的 `select` `poll` `epoll`）。
  - 通过 Selector 和 Channel（如 `SocketChannel` `ServerSocketChannel` `DatagramChannel`），Java 可以实现高效的网络 I/O。
  - 但这是 就绪通知（readiness-based）模型，而不是真正的完成型异步 I/O。
- NIO.2（Asynchronous I/O，Java 7+）：
  - 在 java.nio.channels 包中提供了异步通道类，如 `AsynchronousSocketChannel` `AsynchronousFileChannel`
  - 支持通过 Future 或回调（CompletionHandler）处理 I/O 操作的结果。
  - 底层实现依赖操作系统的异步 I/O 接口：
    - 在 Linux 上，`AsynchronousFileChannel` 使用 `POSIX AIO` 作为其底层实现机制。`AsynchronousSocketChannel` 使用 `epoll`
    - 在 Windows 上，使用 IOCP（I/O Completion Ports）。
    - 在 macOS 上，使用线程池或 kqueue。

Java 原生提供的NIO过于底层，编写复杂度很高，不推荐程序员直接使用，可以通过Netty框架来进行网络编程。

## AIO (Asynchronous I/O)

Java

* 在Window上使用IOCP，是完全异步的模型。

* 在Linux和BSD中 `AsynchronousFileChannel` 使用 `POSIX AIO` 作为其底层实现机制是完全异步的操作，但`AsynchronousSocketChannel` 使用 `epoll` 加线程池模拟异步行为。

目前Java还没有支持Linux的 `io_uring` 完全异步IO模型。项目中直接使用AIO的情景很少，主要还是使用NIO，如Netty。



总结：目前主流的Socket IO基本上使用NIO（事件驱动IO），需要自行编写网络IO可使用Netty，需要使用Spring就使用Spring WebFlux（对Netty封装）