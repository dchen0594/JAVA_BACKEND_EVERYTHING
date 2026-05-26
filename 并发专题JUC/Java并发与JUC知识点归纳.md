# Java 并发与 JUC 知识点归纳

> 本文覆盖 Java 并发编程与 `java.util.concurrent`（JUC）中高频考点与工程实践，含原理说明与可运行示例。  
> **目录跳转**：各章节标题使用显式 `id`，避免中文标题自动生成锚点不一致导致无法跳转。

---

## 目录

- [1. 并发与并行、线程基础](#sec-01-basics)
- [2. 线程的创建与生命周期](#sec-02-thread-lifecycle)
- [3. synchronized 与内置锁](#sec-03-synchronized)
- [4. volatile 与 happens-before](#sec-04-volatile)
- [5. Java 内存模型（JMM）](#sec-05-jmm)
- [6. 原子类与 CAS](#sec-06-atomic-cas)
- [7. Lock、ReentrantLock 与 AQS](#sec-07-lock-aqs)
- [8. 读写锁与 StampedLock](#sec-08-rw-stamped)
- [9. Condition 与等待/通知](#sec-09-condition)
- [10. 线程间协作工具](#sec-10-coordination)
- [11. 线程池与 Executor 框架](#sec-11-executor)
- [12. 并发容器](#sec-12-concurrent-collections)
- [13. BlockingQueue 与生产者-消费者](#sec-13-blocking-queue)
- [14. ThreadLocal 与内存泄漏](#sec-14-threadlocal)
- [15. CompletableFuture 与异步编排](#sec-15-completable-future)
- [16. Fork/Join 与并行流](#sec-16-fork-join)
- [17. 死锁、活锁、饥饿与排查](#sec-17-deadlock)
- [18. 常见面试题与工程建议](#sec-18-interview-tips)

---

<h2 id="sec-01-basics">1. 并发与并行、线程基础</h2>

### 1.1 概念区分

| 概念 | 含义 |
|------|------|
| **并发（Concurrency）** | 多个任务在时间上交替推进（单核也可“同时”处理多个任务） |
| **并行（Parallelism）** | 多个任务在同一时刻真正同时执行（通常需要多核） |
| **进程** | 资源分配的基本单位，拥有独立内存空间 |
| **线程** | CPU 调度的基本单位，共享进程堆、方法区等，拥有独立栈 |

### 1.2 为什么需要多线程

- 提高 CPU 利用率（I/O 阻塞时切换线程）
- 降低响应时间（异步处理、分片计算）
- 简化某些模型（每个连接一个线程，需注意资源上限）

### 1.3 使用案例：异步记录日志

```java
public class AsyncLogDemo {
    public static void main(String[] args) throws InterruptedException {
        ExecutorService pool = Executors.newSingleThreadExecutor(r -> {
            Thread t = new Thread(r, "log-writer");
            t.setDaemon(true);
            return t;
        });
        pool.submit(() -> writeLog("order created: 10001"));
        System.out.println("main continues without waiting for disk IO");
        pool.shutdown();
        pool.awaitTermination(2, TimeUnit.SECONDS);
    }

    private static void writeLog(String msg) {
        try { Thread.sleep(200); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        System.out.println(msg);
    }
}
```

---

<h2 id="sec-02-thread-lifecycle">2. 线程的创建与生命周期</h2>

### 2.1 创建线程的方式

| 方式 | 特点 |
|------|------|
| 继承 `Thread` | 不推荐，Java 单继承限制 |
| 实现 `Runnable` | 推荐，任务与线程解耦 |
| 实现 `Callable<V>` + `Future` | 可有返回值、可抛受检异常 |
| 线程池提交任务 | 生产环境首选，复用线程、统一管理 |

```java
// Runnable
new Thread(() -> System.out.println("run")).start();

// Callable + FutureTask
FutureTask<Integer> task = new FutureTask<>(() -> 1 + 1);
new Thread(task).start();
System.out.println(task.get()); // 2
```

### 2.2 线程状态（`Thread.State`）

```
NEW → RUNNABLE ⇄ BLOCKED / WAITING / TIMED_WAITING → TERMINATED
```

| 状态 | 触发场景 |
|------|----------|
| `NEW` | 未 `start()` |
| `RUNNABLE` | 可运行（含就绪与运行，OS 层面可能阻塞） |
| `BLOCKED` | 等待进入 `synchronized` 监视器 |
| `WAITING` | `wait()`、`join()`、`LockSupport.park()` 等无超时等待 |
| `TIMED_WAITING` | `sleep()`、带超时的 `wait`/`join` |
| `TERMINATED` | 执行结束 |

### 2.3 常用 API

```java
Thread t = new Thread(() -> {}, "worker");
t.start();                          // 启动
t.join(1000);                       // 等待结束，最多 1s
Thread.sleep(500);                  // 不释放锁，仅让出 CPU 时间片
Thread.yield();                     // 提示调度器让出，不保证
t.interrupt();                      // 设置中断标志
Thread.currentThread().isInterrupted(); // 检查中断
```

### 2.4 使用案例：可中断的耗时任务

```java
public class InterruptDemo {
    public static void main(String[] args) throws InterruptedException {
        Thread worker = new Thread(() -> {
            while (!Thread.currentThread().isInterrupted()) {
                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    // 捕获后中断标志会被清除，需重新设置
                    Thread.currentThread().interrupt();
                }
            }
            System.out.println("worker stopped gracefully");
        });
        worker.start();
        Thread.sleep(1200);
        worker.interrupt();
        worker.join();
    }
}
```

---

<h2 id="sec-03-synchronized">3. synchronized 与内置锁</h2>

### 3.1 作用

- **互斥**：同一时刻只有一个线程持有监视器锁
- **可见性**：释放锁会把工作内存刷新到主内存；获取锁会从主内存加载
- **可重入**：同一线程可多次获取同一把锁

### 3.2 三种用法

```java
public class SyncDemo {
    private final Object lock = new Object();
    private static int staticCount = 0;
    private int instanceCount = 0;

    public synchronized void instanceMethod() { instanceCount++; }

    public static synchronized void staticMethod() { staticCount++; }

    public void blockSync() {
        synchronized (lock) { instanceCount++; }
    }
}
```

### 3.3 锁升级（JDK 6+，了解即可）

无锁 → **偏向锁**（同一线程反复进入）→ **轻量级锁**（少量竞争，CAS）→ **重量级锁**（激烈竞争，OS Mutex）

### 3.4 使用案例：线程安全的单例（双重检查锁）

```java
public class Singleton {
    private static volatile Singleton instance;

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

> `volatile` 防止指令重排导致返回未初始化完成的对象。

---

<h2 id="sec-04-volatile">4. volatile 与 happens-before</h2>

### 4.1 volatile 能保证什么

- **可见性**：写后立即对其他线程可见
- **有序性**：禁止特定重排（建立 happens-before）
- **不保证原子性**：`i++` 仍需 `synchronized` 或原子类

### 4.2 适用场景

- 状态标志位（`boolean flag`）
- 双重检查锁中的实例引用
- 一写多读的配置变量

```java
public class VolatileFlag {
    private volatile boolean running = true;

    public void stop() { running = false; }

    public void loop() {
        while (running) {
            // do work
        }
    }
}
```

### 4.3 happens-before 规则（常考）

1. 程序顺序规则  
2. 监视器锁规则：`unlock` happens-before 后续 `lock`  
3. `volatile` 写 happens-before 后续对同一变量的读  
4. 线程 `start()` happens-before 线程内动作  
5. 线程内动作 happens-before `join()` 返回  
6. 传递性  

---

<h2 id="sec-05-jmm">5. Java 内存模型（JMM）</h2>

### 5.1 结构

每个线程有**工作内存**（本地缓存），共享数据在主内存。线程对变量的操作在工作内存进行，再同步到主内存。

### 5.2 三大特性

| 特性 | 说明 |
|------|------|
| 原子性 | 不可分割（`synchronized`、`Lock`、`Atomic`） |
| 可见性 | 一个线程修改对其他线程可见（`volatile`、`synchronized`） |
| 有序性 | 禁止重排或建立 happens-before |

### 5.3 使用案例：没有同步时的可见性问题

```java
public class VisibilityBug {
    private static boolean ready = false; // 未 volatile 可能永远看不到 true
    private static int value = 0;

    public static void main(String[] args) throws InterruptedException {
        new Thread(() -> {
            while (!ready) { }
            System.out.println("value = " + value); // 可能打印 0
        }).start();
        value = 42;
        ready = true;
    }
}
```

修复：将 `ready` 改为 `volatile`，或使用 `AtomicBoolean`。

---

<h2 id="sec-06-atomic-cas">6. 原子类与 CAS</h2>

### 6.1 CAS（Compare-And-Swap）

CPU 原子指令：比较内存值与期望值，相等则更新。  
**ABA 问题**：值从 A→B→A，仅比较值会误以为未变。可用 `AtomicStampedReference` 带版本号解决。

### 6.2 常用原子类

| 类 | 用途 |
|----|------|
| `AtomicInteger/Long` | 计数器 |
| `AtomicReference` | 引用原子更新 |
| `AtomicIntegerArray` | 数组元素原子操作 |
| `LongAdder` | 高并发累加，分段减少竞争 |
| `AtomicStampedReference` | 带版本 CAS |

### 6.3 使用案例：高并发计数器

```java
public class CounterDemo {
    private final LongAdder adder = new LongAdder();

    public void increment() { adder.increment(); }

    public long sum() { return adder.sum(); }

    public static void main(String[] args) throws InterruptedException {
        CounterDemo demo = new CounterDemo();
        int threads = 10;
        CountDownLatch latch = new CountDownLatch(threads);
        ExecutorService pool = Executors.newFixedThreadPool(threads);
        for (int i = 0; i < threads; i++) {
            pool.submit(() -> {
                for (int j = 0; j < 100_000; j++) demo.increment();
                latch.countDown();
            });
        }
        latch.await();
        System.out.println(demo.sum()); // 1000000
        pool.shutdown();
    }
}
```

### 6.4 自旋 vs 阻塞

CAS 失败会自旋重试，适合竞争不激烈场景；竞争激烈时 `Lock`/`synchronized` 可能更合适。

---

<h2 id="sec-07-lock-aqs">7. Lock、ReentrantLock 与 AQS</h2>

### 7.1 Lock 接口

```java
void lock();
void unlock();  // 必须在 finally 中释放
boolean tryLock();
boolean tryLock(long time, TimeUnit unit);
Condition newCondition();
```

### 7.2 ReentrantLock

- 可中断锁：`lockInterruptibly()`
- 公平/非公平构造：`new ReentrantLock(true)`
- 支持多个 `Condition`

```java
public class ReentrantLockDemo {
    private final ReentrantLock lock = new ReentrantLock();

    public void transfer() {
        lock.lock();
        try {
            // 临界区
        } finally {
            lock.unlock();
        }
    }
}
```

### 7.3 AQS（AbstractQueuedSynchronizer）

JUC 锁与同步器的基础框架：

- **state**：同步状态（重入次数、许可数等）
- **CLH 队列**：获取锁失败则入队阻塞
- 模板方法：`tryAcquire` / `tryRelease` 由子类实现

基于 AQS 的实现：`ReentrantLock`、`Semaphore`、`CountDownLatch`、`ReentrantReadWriteLock` 等。

### 7.4 synchronized vs ReentrantLock

| 对比项 | synchronized | ReentrantLock |
|--------|--------------|---------------|
| 释放 | 自动 | 手动 `unlock` |
| 公平性 | 非公平 | 可选公平 |
| 条件变量 | 单个（wait/notify） | 多个 Condition |
| 中断 | 不支持 | `lockInterruptibly` |

---

<h2 id="sec-08-rw-stamped">8. 读写锁与 StampedLock</h2>

### 8.1 ReentrantReadWriteLock

- **读锁**：共享，多线程可同时读
- **写锁**：独占，与读/写互斥
- 适合**读多写少**

```java
public class CacheWithRwLock {
    private final Map<String, String> cache = new HashMap<>();
    private final ReentrantReadWriteLock rw = new ReentrantReadWriteLock();

    public String get(String key) {
        rw.readLock().lock();
        try {
            return cache.get(key);
        } finally {
            rw.readLock().unlock();
        }
    }

    public void put(String key, String value) {
        rw.writeLock().lock();
        try {
            cache.put(key, value);
        } finally {
            rw.writeLock().unlock();
        }
    }
}
```

### 8.2 StampedLock（JDK 8+）

- 乐观读：`tryOptimisticRead()`，读后验证 `validate(stamp)`
- 不适合重度写竞争；不可重入，编码更复杂

---

<h2 id="sec-09-condition">9. Condition 与等待/通知</h2>

### 9.1 对比 wait/notify

`Condition` 绑定 `Lock`，可创建多个条件队列（如“非满”“非空”）。

```java
public class BoundedBuffer {
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();
    private final Object[] items = new Object[10];
    private int count, putIndex, takeIndex;

    public void put(Object x) throws InterruptedException {
        lock.lock();
        try {
            while (count == items.length) notFull.await();
            items[putIndex] = x;
            if (++putIndex == items.length) putIndex = 0;
            count++;
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public Object take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) notEmpty.await();
            Object x = items[takeIndex];
            if (++takeIndex == items.length) takeIndex = 0;
            count--;
            notFull.signal();
            return x;
        } finally {
            lock.unlock();
        }
    }
}
```

> 生产环境更常用 `ArrayBlockingQueue` 等现成实现。

---

<h2 id="sec-10-coordination">10. 线程间协作工具</h2>

### 10.1 CountDownLatch

一次性倒数，归零后等待线程继续。

```java
// 主线程等待 3 个子任务完成
CountDownLatch latch = new CountDownLatch(3);
for (int i = 0; i < 3; i++) {
    pool.submit(() -> {
        try { doWork(); } finally { latch.countDown(); }
    });
}
latch.await();
```

### 10.2 CyclicBarrier

多线程互相等待到齐，可**重复使用**（分阶段计算）。

```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> System.out.println("phase done"));
// 每个线程 barrier.await();
```

### 10.3 Semaphore

控制同时访问资源的线程数（限流、连接池）。

```java
Semaphore sem = new Semaphore(5); // 最多 5 个并发
sem.acquire();
try {
    accessResource();
} finally {
    sem.release();
}
```

### 10.4 Phaser

更灵活的分阶段屏障，支持动态注册参与者。

### 10.5 Exchanger

两个线程在同步点交换数据。

### 10.6 对比小结

| 工具 | 可重用 | 典型场景 |
|------|--------|----------|
| CountDownLatch | 否 | 等待多个任务完成 |
| CyclicBarrier | 是 | 多阶段并行计算 |
| Semaphore | 是 | 限流、资源池 |
| Phaser | 是 | 复杂多阶段协作 |

---

<h2 id="sec-11-executor">11. 线程池与 Executor 框架</h2>

### 11.1 为什么用线程池

- 降低创建/销毁线程开销
- 控制并发度，防止 OOM
- 统一管理、监控、拒绝策略

### 11.2 ThreadPoolExecutor 核心参数

```java
new ThreadPoolExecutor(
    corePoolSize,      // 核心线程数
    maximumPoolSize,   // 最大线程数
    keepAliveTime,     // 非核心线程空闲存活时间
    unit,
    workQueue,         // 任务队列
    threadFactory,
    handler            // 拒绝策略
);
```

**任务提交流程**：

1. 线程数 < core → 创建核心线程执行  
2. 否则入队；队列满且线程数 < max → 创建非核心线程  
3. 队列满且线程数 = max → 执行拒绝策略  

### 11.3 常见队列

| 队列 | 特点 |
|------|------|
| `LinkedBlockingQueue` | 有界/无界链表，常用 |
| `ArrayBlockingQueue` | 有界数组，公平可选 |
| `SynchronousQueue` | 不存储，直接交接 |
| `DelayedWorkQueue` | `ScheduledThreadPoolExecutor` 内部使用 |

### 11.4 拒绝策略

- `AbortPolicy`：抛异常（默认）
- `CallerRunsPolicy`：调用者线程执行
- `DiscardPolicy`：静默丢弃
- `DiscardOldestPolicy`：丢弃队列最老任务

### 11.5 使用案例：业务线程池（推荐显式配置）

```java
public class OrderThreadPool {
    private static final ThreadPoolExecutor EXECUTOR = new ThreadPoolExecutor(
        4, 8,
        60, TimeUnit.SECONDS,
        new ArrayBlockingQueue<>(200),
        r -> {
            Thread t = new Thread(r);
            t.setName("order-pool-" + t.getId());
            return t;
        },
        new ThreadPoolExecutor.CallerRunsPolicy()
    );

    public static void submitOrder(Runnable task) {
        EXECUTOR.execute(task);
    }
}
```

### 11.6 Executors 工厂方法（了解风险）

| 方法 | 风险 |
|------|------|
| `newFixedThreadPool` | 无界队列可能堆积大量任务 OOM |
| `newCachedThreadPool` | 线程数无上限 |
| `newSingleThreadExecutor` | 无界队列 |
| `newScheduledThreadPool` | 需合理设置核心数 |

**阿里规约建议**：使用 `ThreadPoolExecutor` 显式传参。

### 11.7 ForkJoinPool 与 work-stealing

分治任务，空闲线程从其他队列尾部“偷”任务。见 [第 16 节](#sec-16-fork-join)。

---

<h2 id="sec-12-concurrent-collections">12. 并发容器</h2>

### 12.1 ConcurrentHashMap（JDK 8+）

- 结构：数组 + 链表 + 红黑树
- 锁粒度：对桶头节点 `synchronized`（或 CAS 插入），非整表锁
- `size()` 使用 `baseCount` + `CounterCell` 类似 LongAdder

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.computeIfAbsent("b", k -> 2);
map.merge("a", 1, Integer::sum); // 原子合并
```

### 12.2 CopyOnWriteArrayList

写时复制整个数组，读无锁，适合**读极多写极少**（监听器列表、配置快照）。

```java
CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();
listeners.add("handler1"); // 写操作复制数组，开销大
```

### 12.3 ConcurrentLinkedQueue

无界非阻塞链表队列，CAS 实现。

### 12.4 对比 Hashtable / Collections.synchronizedXxx

旧 API 整表/整对象锁，并发度低；新代码优先 JUC 容器。

---

<h2 id="sec-13-blocking-queue">13. BlockingQueue 与生产者-消费者</h2>

### 13.1 常用实现

| 类 | 有界 | 特点 |
|----|------|------|
| `ArrayBlockingQueue` | 是 | 数组，一把锁 |
| `LinkedBlockingQueue` | 可选 | 两把锁提高吞吐 |
| `PriorityBlockingQueue` | 无界 | 堆排序 |
| `DelayQueue` | 无界 | 延迟元素 |
| `SynchronousQueue` | 0 容量 | 直接传递 |

### 13.2 使用案例：经典生产者-消费者

```java
public class ProducerConsumer {
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(5);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 0; i < 20; i++) {
                    queue.put(i);
                    System.out.println("produced " + i);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        Thread consumer = new Thread(() -> {
            try {
                while (true) {
                    int v = queue.take();
                    System.out.println("consumed " + v);
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });

        producer.start();
        consumer.start();
    }
}
```

---

<h2 id="sec-14-threadlocal">14. ThreadLocal 与内存泄漏</h2>

### 14.1 原理

每个 `Thread` 持有 `ThreadLocalMap`，以 `ThreadLocal` 为 key（弱引用），值为强引用。

### 14.2 典型用途

- 存放用户上下文（`userId`、租户 ID）
- `SimpleDateFormat` 非线程安全时的线程隔离
- Spring 事务同步资源绑定

```java
public class UserContext {
    private static final ThreadLocal<String> USER = new ThreadLocal<>();

    public static void set(String userId) { USER.set(userId); }
    public static String get() { return USER.get(); }
    public static void clear() { USER.remove(); } // 线程池场景必须清理
}
```

### 14.3 内存泄漏原因

- `ThreadLocal` 被回收后，key 为 null，但 value 仍被线程强引用
- **线程池**中线程长期存活 → value 无法回收  

**规范**：`try-finally` 中调用 `remove()`。

### 14.4 InheritableThreadLocal

子线程可继承父线程值；线程池中无效（应用 `TransmittableThreadLocal` TTL 等方案）。

---

<h2 id="sec-15-completable-future">15. CompletableFuture 与异步编排</h2>

### 15.1 常用 API

```java
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> fetchUser());
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> fetchOrder());

CompletableFuture<String> combined = f1.thenCombine(f2, (u, o) -> u + "|" + o);

f1.thenAccept(System.out::println);
f1.exceptionally(ex -> { log.error(ex); return "default"; });
CompletableFuture.allOf(f1, f2).join();
```

### 15.2 使用案例：并行调用下游再聚合

```java
public class AggregateService {
    private final ExecutorService pool = Executors.newFixedThreadPool(8);

    public String loadDashboard(String userId) throws Exception {
        CompletableFuture<String> profile = CompletableFuture
            .supplyAsync(() -> rpcGetProfile(userId), pool);
        CompletableFuture<Integer> points = CompletableFuture
            .supplyAsync(() -> rpcGetPoints(userId), pool);

        return profile.thenCombine(points, (p, pt) -> p + ", points=" + pt)
            .get(3, TimeUnit.SECONDS);
    }

    private String rpcGetProfile(String id) { return "name=Tom"; }
    private int rpcGetPoints(String id) { return 100; }
}
```

### 15.3 注意

- 指定线程池，避免默认 `ForkJoinPool.commonPool()` 被业务占满
- 处理好超时与异常链

---

<h2 id="sec-16-fork-join">16. Fork/Join 与并行流</h2>

### 16.1 RecursiveTask / RecursiveAction

```java
public class SumTask extends RecursiveTask<Long> {
    private final int[] arr;
    private final int start, end;
    private static final int THRESHOLD = 10_000;

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += arr[i];
            return sum;
        }
        int mid = (start + end) >>> 1;
        SumTask left = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();
        long rightResult = right.compute();
        long leftResult = left.join();
        return leftResult + rightResult;
    }
}
```

### 16.2 parallelStream

底层共用 `ForkJoinPool.commonPool()`：

```java
long sum = Arrays.stream(arr).parallel().sum();
```

CPU 密集型、无共享可变状态时可考虑；有锁/IO 时慎用。

---

<h2 id="sec-17-deadlock">17. 死锁、活锁、饥饿与排查</h2>

### 17.1 死锁四个必要条件

互斥、占有且等待、不可抢占、循环等待。  
**预防**：固定加锁顺序、超时 `tryLock`、银行家算法（理论）。

```java
// 错误：不同顺序获取锁
// 线程1: lockA -> lockB
// 线程2: lockB -> lockA

// 正确：全局顺序
synchronized (lockA) {
    synchronized (lockB) { /* ... */ }
}
```

### 17.2 排查工具

```bash
jstack <pid>   # 查找 Found one Java-level deadlock
jcmd <pid> Thread.print
```

### 17.3 活锁与饥饿

- **活锁**：线程未阻塞，但互相让步导致都无法推进
- **饥饿**：低优先级或非公平锁导致部分线程长期得不到资源

---

<h2 id="sec-18-interview-tips">18. 常见面试题与工程建议</h2>

### 18.1 高频面试题速览

1. `synchronized` 与 `volatile` 区别？  
2. `ReentrantLock` 与 `synchronized` 区别？  
3. `ConcurrentHashMap` 1.7 与 1.8 实现差异？  
4. 线程池参数如何设置？  
5. 如何避免死锁？  
6. `ThreadLocal` 原理与泄漏？  
7. AQS 工作流程？  
8. `Callable` 与 `Runnable`？  
9. `sleep` 与 `wait`？  
10. `submit` 与 `execute`？（`submit` 吞异常需注意 `Future.get()`）

### 18.2 工程实践清单

- [ ] 优先使用 JUC 工具类，少手写 wait/notify  
- [ ] 线程池显式配置，禁止 `Executors` 无界池  
- [ ] 锁粒度尽量小，避免在锁内做 RPC/IO  
- [ ] 中断标志正确处理，不要吞 `InterruptedException`  
- [ ] 线程池 + ThreadLocal 必须 `remove()`  
- [ ] 异步用 `CompletableFuture` 并指定线程池与超时  
- [ ] 高并发计数用 `LongAdder`  
- [ ] 用 `jstack`/APM 排查死锁与线程堆积  

### 18.3 JUC 包结构速查

```
java.util.concurrent
├── locks (Lock, ReentrantLock, StampedLock, Condition)
├── atomic (AtomicXxx, LongAdder)
├── 线程池 (Executor, ThreadPoolExecutor, Scheduled...)
├── 容器 (ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue...)
├── 协作 (CountDownLatch, CyclicBarrier, Semaphore, Phaser)
└── CompletableFuture
```

---

## 附录：与本文目录对应的锚点 ID 一览

| 章节 | 锚点 ID |
|------|---------|
| 1 | `sec-01-basics` |
| 2 | `sec-02-thread-lifecycle` |
| 3 | `sec-03-synchronized` |
| 4 | `sec-04-volatile` |
| 5 | `sec-05-jmm` |
| 6 | `sec-06-atomic-cas` |
| 7 | `sec-07-lock-aqs` |
| 8 | `sec-08-rw-stamped` |
| 9 | `sec-09-condition` |
| 10 | `sec-10-coordination` |
| 11 | `sec-11-executor` |
| 12 | `sec-12-concurrent-collections` |
| 13 | `sec-13-blocking-queue` |
| 14 | `sec-14-threadlocal` |
| 15 | `sec-15-completable-future` |
| 16 | `sec-16-fork-join` |
| 17 | `sec-17-deadlock` |
| 18 | `sec-18-interview-tips` |

> 在 GitHub、VS Code、Cursor 预览中，目录链接格式为 `[标题](#sec-xx-xxx)`，与上文 `<h2 id="sec-xx-xxx">` 一一对应，可稳定跳转。
