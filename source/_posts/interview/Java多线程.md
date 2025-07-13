---
title: Java 多线程详解
category: interview
---

# Java 创建线程

```java
public class thread {
    public static void main(String[] args) {
        Thread thread = new Thread(() -> System.out.println("thread start"));
        thread.start();
    }
}
```





# Java 同步工具

## `synchronized`

## Lock + Condition

## Sem

**创建不同类型的线程池**： Executors 提供了多种工厂方法来创建 ExecutorService 实现，每种线程池适用于特定场景：

- newFixedThreadPool(int nThreads)：
  - 创建一个固定大小的线程池，维护指定数量的线程，适合需要限制线程数的任务。
  - 作用：确保线程资源可控，适用于长期运行的任务或负载均衡的场景。
- newCachedThreadPool()：
  - 创建一个可根据需要动态创建和回收线程的线程池，适合执行大量短期任务。
  - 作用：提高短任务的响应速度，但在高负载下可能创建过多线程。
- newSingleThreadExecutor()：
  - 创建一个单线程的线程池，按顺序执行任务，适合需要严格顺序执行的场景。
  - 作用：保证任务按提交顺序执行，类似单线程模型但支持异步提交。
- newScheduledThreadPool(int corePoolSize)：
  - 创建一个支持定时和周期性任务的线程池，适合定时任务或延迟执行。
  - 作用：用于调度定期任务，如定时刷新缓存或心跳检测。
- newVirtualThreadPerTaskExecutor()（JDK 21 及以上）：
  - 创建一个为每个任务分配一个虚拟线程的线程池，适合高并发的 I/O 密集型任务。
  - 作用：利用虚拟线程的轻量级特性，支持大规模并发，简化同步编程模型。

```java
public class VirtualThreadExample {
    public static void main(String[] args) throws InterruptedException {
        // 方式 1: 使用 Thread.ofVirtual()
        Thread.ofVirtual().start(() -> {
            System.out.println("Virtual thread 1: " + Thread.currentThread());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        // 方式 2: 使用 ExecutorService
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            executor.submit(() -> {
                System.out.println("Virtual thread 2: " + Thread.currentThread());
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            });
        }

        // 方式 3: 使用 Thread.startVirtualThread()
        Thread.startVirtualThread(() -> {
            System.out.println("Virtual thread 3: " + Thread.currentThread());
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        });

        // 等待任务完成
        Thread.sleep(2000);
    }
}
```





练习题

[按序打印](https://leetcode.cn/problems/print-in-order/description/)

信号量

```java
class Foo {

    private final Semaphore semaphore1 = new Semaphore(0);
    private final Semaphore semaphore2 = new Semaphore(0);

    public Foo() {

    }

    public void first(Runnable printFirst) throws InterruptedException {
        // printFirst.run() outputs "first". Do not change or remove this line.
        printFirst.run();
        semaphore1.release(); // 释放第一个信号量，允许第二个线程执行
    }

    public void second(Runnable printSecond) throws InterruptedException {
        semaphore1.acquire(); // 等待第一个线程完成
        // printSecond.run() outputs "second". Do not change or remove this line.
        printSecond.run();
        semaphore2.release(); // 释放第二个信号量，允许第三个线程执行
    }

    public void third(Runnable printThird) throws InterruptedException {
        semaphore2.acquire(); // 等待第二个线程完成
        // printThird.run() outputs "third". Do not change or remove this line.
        printThird.run();
        semaphore2.release(); // 释放第三个信号量，允许其他线程执行
    }
}
```

CountDownLatch

```java
class Foo {

    private final CountDownLatch firstLatch = new CountDownLatch(1);
    private final CountDownLatch secondLatch = new CountDownLatch(1);

    public Foo() {

    }

    public void first(Runnable printFirst) throws InterruptedException {

        // printFirst.run() outputs "first". Do not change or remove this line.
        try {
            printFirst.run();
        } finally {
            firstLatch.countDown();
        }
    }

    public void second(Runnable printSecond) throws InterruptedException {
        firstLatch.await();
        try {
            // printSecond.run() outputs "second". Do not change or remove this line.
            printSecond.run();
        } finally {
            secondLatch.countDown();
        }
    }

    public void third(Runnable printThird) throws InterruptedException {
        secondLatch.await();
        // printThird.run() outputs "third". Do not change or remove this line.
        printThird.run();

    }
}
```

