# GC 垃圾回收调优

> 学习时间: | 状态: 🔄 进行中

---

## 📚 学习目标

- [ ] 理解 GC roots 和可达性分析
- [ ] 掌握 G1/ZGC 收集器原理
- [ ] GC 日志分析与问题定位
- [ ] GC 调优实战

---

## 📖 笔记内容

### 判断对象存活

#### GC Roots

可作为 GC Roots 的对象：
1. 虚拟机栈中引用的对象
2. 方法区中静态属性引用的对象
3. 方法区中常量引用的对象
4. 本地方法栈中 JNI 引用的对象

### 引用类型

```java
// 强引用 - 只要引用存在，就不会被 GC
Object strong = new Object();

// 软引用 - 内存不足时回收
SoftReference<Object> soft = new SoftReference<>(new Object());

// 弱引用 - 每次 GC 都会回收
WeakReference<Object> weak = new WeakReference<>(new Object());

// 虚引用 - 无法通过引用获取对象，用于跟踪对象回收
ReferenceQueue<Object> queue = new ReferenceQueue<>();
PhantomReference<Object> phantom = new PhantomReference<>(new Object(), queue);
```

---

## 🗑️ 垃圾收集器

### G1 收集器

**适用场景**: 大堆内存 (4G+)，低停顿要求

```
┌─────────────────────────────────────────────────────────────┐
│                      G1 内存布局                             │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐         │
│  │ R1 │ │ R2 │ │ R3 │ │ R4 │ │ R5 │ │ R6 │ │ R7 │ ...   │
│  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘         │
│                                                             │
│  Region: 大小固定 (1MB-32MB，可配置)                        │
│  - Eden Region                                             │
│  - Survivor Region                                         │
│  - Old Region                                              │
│  - Humongous Region (巨型对象)                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**GC 类型**:
- **Young GC**: 只回收新生代 Region
- **Mixed GC**: 回收新生代 + 部分老年代 Region
- **Full GC**: 整堆回收 (STW 时间长，尽量避免)

**Mixed GC 触发条件**:
```bash
-XX:InitiatingHeapOccupancyPercent=45  # 老年代占用超过 45%
```

### ZGC 收集器 (JDK 11+)

**特点**:
- 停顿时间 < 10ms
- 支持 TB 级堆内存
- 染色指针 + 读屏障

```bash
# 开启 ZGC (JDK 15+)
-XX:+UseZGC
```

---

## 📊 GC 日志分析

### JDK 8 日志参数

```bash
# GC 日志
-XX:+PrintGC
-XX:+PrintGCDetails
-XX:+PrintGCTimeStamps
-XX:+PrintGCApplicationStoppedTime
-XX:+PrintGCApplicationConcurrentTime

# 文件输出
-Xloggc:/path/to/gc.log
-XX:+UseGCLogFileRotation
-XX:NumberOfGCLogFiles=10
-XX:GCLogFileSize=10M
```

### JDK 9+ 统一日志

```bash
# GC 详细日志
-Xlog:gc*:file=/path/to/gc.log:time,level,tags:filecount=10,filesize=100m

# 简化版
-Xlog:gc*
```

### 日志分析工具

| 工具 | 说明 |
|------|------|
| [GCViewer](https://github.com/chewiebug/GCViewer) | GUI 分析工具 |
| [GCEasy](https://gceasy.io/) | 在线分析 |
| [CMS GC](https://gceasy.io/) | 推荐 |

---

## 🔧 GC 调优实战

### 调优流程

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│  1. 确定目标  │ -> │  2. 分析日志   │ -> │  3. 调整参数  │
│  - 停顿时间   │    │  - GC 频率     │    │  - 堆大小      │
│  - 吞吐量     │    │  - 停顿时间    │    │  - GC 算法     │
└───────────────┘    └───────────────┘    └───────────────┘
```

### 常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 频繁 Young GC | Eden 太小 | 增大新生代 |
| Full GC 频繁 | 老年代不足 | 增大堆内存/检查内存泄漏 |
| GC 停顿长 | Region 太多 | 减小 Region 大小 |
| OOM: GC overhead | 内存不足/泄漏 | 增大堆/分析 heap dump |

### G1 调优参数

```bash
# 目标最大停顿时间 (默认 200ms)
-XX:MaxGCPauseMillis=200

# Region 大小 (自动计算，可手动指定)
-XX:G1HeapRegionSize=16m

# 触发 Mixed GC 的堆占用比例
-XX:InitiatingHeapOccupancyPercent=45

# 并行标记线程数
-XX:ConcGCThreads=4
-XX:ParallelGCThreads=8
```

---

## 💻 实践练习

### 实验: G1 GC 日志分析

```java
public class GCTest {
    public static void main(String[] args) {
        List<byte[]> list = new ArrayList<>();
        for (int i = 0; i < 1000; i++) {
            list.add(new byte[1024 * 1024]); // 1MB
        }
    }
}
```

**启动参数**:
```bash
java -Xms512m -Xmx512m -XX:+UseG1GC \
     -Xlog:gc*:file=gc.log:time,level,tags \
     GCTest
```

**关键日志解读**:
```
[GC pause (G1 Evacuation Pause) (young), 0.0234567 secs]
   [Parallel Time: 18.5 ms]
   -> 256M->128M(512M) 128M->64M(512M)

[GC pause (G1 Evacuation Pause) (mixed), 0.0456789 secs]
   [Parallel Time: 42.3 ms]
```

---

## 📝 总结

- GC Roots 判断对象存活
- G1 适合大堆低延迟场景
- ZGC 是未来的方向
- 调优要先确定目标，再分析日志

---

## 🔗 相关资源

- [G1 GC 官方文档](https://docs.oracle.com/en/java/javase/17/gctuning/garbage-first-garbage-collector.html)
- [ZGC 官方文档](https://docs.oracle.com/en/java/javase/17/gctuning/z-garbage-collector.html)
