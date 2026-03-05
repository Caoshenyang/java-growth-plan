# Java 并发编程精通笔记

## 学习目标
- [ ] JUC源码分析（AQS、CAS、volatile、synchronized）
- [ ] 线程池原理与调优（ThreadPoolExecutor、ForkJoinPool）
- [ ] 锁机制（乐观锁、悲观锁、分布式锁）
- [ ] 并发容器（ConcurrentHashMap、CopyOnWriteArrayList）
- [ ] 异步编程（CompletableFuture、Reactor）

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. JUC 核心原理
**学习日期：____-____-____**

#### 1.1 volatile 关键字
```java
// volatile 特性
public class VolatileExample {
    private volatile boolean flag = false;

    // 可见性：一个线程修改，其他线程立即可见
    // 有序性：禁止指令重排序
    // 原子性：不保证（仅保证单次读写的原子性）
}
```

#### 1.2 synchronized 原理
```java
// 对象锁 vs 类锁
public class SyncExample {
    // 对象锁
    public synchronized void method1() {}

    // 类锁
    public static synchronized void method2() {}

    // 代码块锁
    public void method3() {
        synchronized(this) {}
    }
}
```

**锁升级过程**：偏向锁 → 轻量级锁 → 重量级锁

#### 1.3 CAS（Compare And Swap）
```java
// CAS 原理（无锁算法）
public class CASExample {
    private AtomicInteger atomicInt = new AtomicInteger(0);

    public void increment() {
        // CAS + 自旋
        atomicInt.incrementAndGet(); // 底层使用 CAS 实现
    }
}

// CAS 存在的问题：
// 1. ABA 问题 → 解决：AtomicStampedReference
// 2. 循环时间长开销大
// 3. 只能保证一个共享变量的原子性
```

#### 1.4 AQS（AbstractQueuedSynchronizer）
```java
// AQS 核心原理
// 1. state 变量（同步状态）
// 2. CLH 队列（等待队列）
// 3. 独占/共享模式

public class AQSLock implements Lock {
    private final Sync sync = new Sync();

    private static class Sync extends AbstractQueuedSynchronizer {
        @Override
        protected boolean tryAcquire(int arg) {
            return compareAndSetState(0, 1);
        }

        @Override
        protected boolean tryRelease(int arg) {
            setState(0);
            return true;
        }
    }

    // ... 实现 Lock 接口方法
}
```

---

### 2. 线程池深度剖析
**学习日期：____-____-____**

#### 2.1 ThreadPoolExecutor 原理
```java
// 线程池核心参数
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    corePoolSize,      // 核心线程数
    maximumPoolSize,   // 最大线程数
    keepAliveTime,     // 非核心线程存活时间
    TimeUnit.SECONDS,  // 时间单位
    workQueue,         // 阻塞队列
    threadFactory,     // 线程工厂
    rejectedHandler    // 拒绝策略
);

// 工作流程：
// 1. 核心线程数未满 → 创建核心线程
// 2. 核心线程数已满 → 放入队列
// 3. 队列已满 → 创建非核心线程
// 4. 达到最大线程数 → 执行拒绝策略
```

#### 2.2 四种拒绝策略
| 策略 | 说明 | 使用场景 |
|------|------|---------|
| AbortPolicy | 抛异常 | 默认策略，明确拒绝 |
| CallerRunsPolicy | 调用者执行 | 需要降级处理 |
| DiscardPolicy | 直接丢弃 | 允许丢失任务 |
| DiscardOldestPolicy | 丢弃最老任务 | 优先执行新任务 |

#### 2.3 线程池调优
```java
// CPU 密集型：线程数 = CPU 核心数 + 1
int cpuTasks = Runtime.getRuntime().availableProcessors() + 1;

// IO 密集型：线程数 = CPU 核心数 / (1 - 阻塞系数)
// 阻塞系数一般在 0.8-0.9 之间
int ioTasks = (int)(cpuCoreCount / (1 - 0.9));
```

#### 2.4 ForkJoinPool
```java
// 分治任务框架
ForkJoinPool forkJoinPool = new ForkJoinPool();

class FibonacciTask extends RecursiveTask<Integer> {
    private int n;

    @Override
    protected Integer compute() {
        if (n <= 1) return n;

        FibonacciTask f1 = new FibonacciTask(n - 1);
        FibonacciTask f2 = new FibonacciTask(n - 2);

        f1.fork();  // 异步执行
        return f2.compute() + f1.join();  // 等待结果
    }
}
```

---

### 3. 锁机制
**学习日期：____-____-____**

#### 3.1 乐观锁 vs 悲观锁
```java
// 悲观锁（synchronized、ReentrantLock）
public void pessimisticLock() {
    synchronized (this) {
        // 悲观认为会发生冲突，先加锁再操作
    }
}

// 乐观锁（CAS、版本号）
public boolean optimisticLock(int version) {
    // 乐观认为不会冲突，先操作再验证
    return atomicCompareAndSet(expectedValue, newValue);
}
```

#### 3.2 ReentrantLock
```java
ReentrantLock lock = new ReentrantLock();

public void method() {
    lock.lock();
    try {
        // 业务逻辑
    } finally {
        lock.unlock();
    }
}

// 特性：
// 1. 可中断
// 2. 可超时
// 3. 公平锁/非公平锁
// 4. 支持多个 Condition
```

#### 3.3 读写锁（ReadWriteLock）
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();

public void read() {
    rwLock.readLock().lock();
    try {
        // 读操作（共享锁）
    } finally {
        rwLock.readLock().unlock();
    }
}

public void write() {
    rwLock.writeLock().lock();
    try {
        // 写操作（排他锁）
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

---

### 4. 并发容器
**学习日期：____-____-____**

#### 4.1 ConcurrentHashMap
```java
// JDK 1.7：分段锁（Segment）
// JDK 1.8：CAS + synchronized（Node数组）

ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();

// 常用方法
map.put(key, value);
map.get(key);
map.putIfAbsent(key, value);
map.compute(key, (k, v) -> v + "updated");
```

#### 4.2 CopyOnWriteArrayList
```java
// 写时复制：适合读多写少
CopyOnWriteArrayList<String> list = new CopyOnWriteArrayList<>();

// 原理：写操作复制新数组，读操作无需加锁
// 缺点：写操作性能差，内存占用大
```

#### 4.3 BlockingQueue
```java
// 常用实现
ArrayBlockingQueue<String> arrayQueue = new ArrayBlockingQueue<>(100);
LinkedBlockingQueue<String> linkedQueue = new LinkedBlockingQueue<>(100);
PriorityBlockingQueue<String> priorityQueue = new PriorityBlockingQueue<>();
SynchronousQueue<String> syncQueue = new SynchronousQueue<>();

// 生产者-消费者模式
BlockingQueue<String> queue = new ArrayBlockingQueue<>(10);

// 生产者
new Thread(() -> {
    try {
        queue.put("product");  // 队列满时阻塞
    } catch (InterruptedException e) {}
}).start();

// 消费者
new Thread(() -> {
    try {
        String item = queue.take();  // 队列空时阻塞
    } catch (InterruptedException e) {}
}).start();
```

---

### 5. 异步编程
**学习日期：____-____-____**

#### 5.1 CompletableFuture
```java
// 创建异步任务
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // 异步执行
    return "result";
});

// 链式调用
future
    .thenApply(result -> result + " processed")
    .thenAccept(result -> System.out.println(result))
    .exceptionally(ex -> {
        System.err.println("Error: " + ex.getMessage());
        return null;
    });

// 组合多个 Future
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> "Hello");
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> "World");

CompletableFuture<String> combined = f1.thenCombine(f2, (r1, r2) -> r1 + " " + r2);

// 等待所有完成
CompletableFuture<Void> allOf = CompletableFuture.allOf(f1, f2);
```

#### 5.2 Reactor（响应式编程）
```java
// Mono（0-1个元素）
Mono<String> mono = Mono.just("value");

// Flux（0-N个元素）
Flux<String> flux = Flux.just("a", "b", "c");

// 操作符
flux
    .map(String::toUpperCase)
    .filter(s -> s.startsWith("A"))
    .subscribe(System.out::println);
```

---

## 实战项目

### 项目：并发工具库实现
**项目地址**：`projects/concurrent-utils/`

#### 项目目标
- [ ] 实现自定义线程池
- [ ] 实现分布式锁
- [ ] 实现限流器（令牌桶、漏桶）

---

## 学习资源

### 书籍
- 《Java并发编程的艺术》方腾飞
- 《Java并发编程实战》Brian Goetz

### 视频
- 极客时间《Java并发编程实战》

### 源码
- JUC（java.util.concurrent）

---

## 面试题

### 常见面试题
1. synchronized 和 ReentrantLock 的区别？
2. ThreadPoolExecutor 的工作原理？
3. ConcurrentHashMap 的实现原理？
4. CAS 的 ABA 问题如何解决？
5. 线程池的拒绝策略有哪些？

---

## 学习总结

### 核心收获

### 待深入内容
