# JVM 内存模型

> 学习时间: 2026-03-04 | 状态: ✅ 已完成

---

## 📚 学习目标

- [ ] 理解 JVM 内存结构
- [ ] 掌握堆、栈、方法区的区别
- [ ] 理解对象创建与内存分配流程
- [ ] 掌握逃逸分析原理

---

## 📖 笔记内容

### JVM 内存结构

```
┌─────────────────────────────────────────────────────────────┐
│                         JVM 内存结构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   堆 (Heap)                          │   │
│  │  ┌──────────────┐  ┌──────────────┐                │   │
│  │  │   新生代     │  │    老年代     │                │   │
│  │  │  (Young)     │  │    (Old)     │                │   │
│  │  │  ┌────┬────┐ │  │              │                │   │
│  │  │  │Eden│S0│S1│ │  │              │                │   │
│  │  │  └────┴────┘ │  │              │                │   │
│  │  └──────────────┘  └──────────────┘                │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌───────────────┐  ┌─────────────────────────────────┐   │
│  │ 虚拟机栈      │  │  方法区 (Method Area)            │   │
│  │ (VM Stack)    │  │  - 类信息                       │   │
│  │  ┌─────────┐  │  │  - 常量池                       │   │
│  │  │ 栈帧    │  │  │  - 静态变量                     │   │
│  │  └─────────┘  │  └─────────────────────────────────┘   │
│  └───────────────┘                                          │
│                                                             │
│  ┌───────────────┐  ┌─────────────────────────────────┐   │
│  │ 本地方法栈    │  │  程序计数器 (PC Register)         │   │
│  │(Native Stack) │  └─────────────────────────────────┘   │
│  └───────────────┘                                          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 关键概念

#### 1. 堆 (Heap)
- 存储对象实例
- 被**所有线程共享**
- GC 的主要区域
- 可通过 `-Xms` 和 `-Xmx` 设置大小

#### 2. 栈 (Stack)
- **虚拟机栈**: 存储 Java 方法调用
- **本地方法栈**: 存储 native 方法调用
- **线程私有**
- 存储局部变量、方法参数、返回值

#### 3. 方法区 (Method Area)
- 存储类信息、常量、静态变量
- **永久代** (JDK 7) → **元空间** (JDK 8+)
- 元空间使用本地内存

#### 4. 程序计数器 (PC Register)
- 记录当前线程执行的字节码行号
- 唯一不会 OOM 的区域

---

## 🔍 对象创建过程

```java
Object obj = new Object();
```

1. **类加载检查**: 检查类是否已加载
2. **分配内存**:
   - 指针碰撞 (内存规整)
   - 空闲列表 (内存不规整)
3. **初始化零值**: 将内存清零
4. **设置对象头**: 类元数据、哈希码、GC年龄等
5. **执行 `<init>`: 构造函数

---

## 🚀 逃逸分析

**定义**: 分析对象的作用域，判断是否逃逸出方法

### 逃逸场景

```java
// 1. 对象赋值给静态变量 - 逃逸
public static Object obj;
public void method() {
    obj = new Object(); // 逃逸
}

// 2. 对象赋值给其他线程 - 逃逸
public void method2() {
    new Thread(() -> {
        Object local = new Object();
    }).start();
}

// 3. 返回对象 - 逃逸
public Object method3() {
    return new Object(); // 逃逸
}

// 4. 未逃逸
public void method4() {
    Object local = new Object(); // 未逃逸
    local.toString();
}
```

### 优化手段

| 优化 | 说明 |
|------|------|
| 栈上分配 | 未逃逸对象分配在栈上，随栈帧回收 |
| 标量替换 | 将对象拆散为基本类型 |
| 锁消除 | 未逃逸对象无需同步 |

---

## 📊 JVM 参数配置

```bash
# 堆内存
-Xms4g          # 初始堆大小
-Xmx4g          # 最大堆大小
-Xmn2g          # 新生代大小
-XX:NewRatio=2  # 新生代:老年代 = 1:2

# 元空间 (JDK 8+)
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m

# 栈内存
-Xss1m          # 线程栈大小

# 逃逸分析
-XX:+DoEscapeAnalysis              # 开启逃逸分析
-XX:+EliminateAllocations          # 开启标量替换
-XX:+EliminateLocks                # 开启锁消除
-XX:+PrintEscapeAnalysis           # 打印逃逸分析结果
```

---

## 💻 实践练习

### 实验1: 使用 JVisualVM 分析对象分布

```bash
# 启动应用并开启 JMX
java -jar app.jar \
  -Dcom.sun.management.jmxremote \
  -Dcom.sun.management.jmxremote.port=9010 \
  -Dcom.sun.management.jmxremote.authenticate=false \
  -Dcom.sun.management.jmxremote.ssl=false
```

### 实验2: 逃逸分析验证

```java
public class EscapeAnalysisTest {
    public static void main(String[] args) {
        long start = System.currentTimeMillis();
        for (int i = 0; i < 100000000; i++) {
            alloc();
        }
        System.out.println(System.currentTimeMillis() - start);
    }

    // 未逃逸对象，可能被优化为栈上分配
    private static void alloc() {
        Object obj = new Object();
    }
}
```

**开启逃逸分析**: ~3ms
**关闭逃逸分析**: ~700ms

---

## 📝 总结

- JVM 内存分为堆、栈、方法区、程序计数器
- 对象创建涉及类加载、内存分配、初始化
- 逃逸分析可优化为栈上分配、标量替换、锁消除
- 合理配置 JVM 参数对性能至关重要

---

## 🔗 相关资源

- [JVM 规范](https://docs.oracle.com/javase/specs/jvms/se8/html/)
- [《深入理解 Java 虚拟机》](https://book.douban.com/subject/34907497/)
