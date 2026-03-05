# 分布式系统理论笔记

## 学习目标
- [ ] 分布式理论基础（CAP、BASE）
- [ ] 分布式一致性算法（Paxos、Raft、ZAB）
- [ ] 分布式事务（2PC、3PC、TCC、Saga、本地消息表）
- [ ] 分布式锁（Redis、Zookeeper）
- [ ] 分布式ID（雪花算法、UUID、数据库自增）

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. 分布式理论基础
**学习日期：____-____-____**

#### 1.1 CAP理论
```
C（Consistency）一致性：
- 所有节点在同一时间看到相同的数据
- 强一致性、弱一致性、最终一致性

A（Availability）可用性：
- 每个请求都能得到响应
- 不保证响应是成功的

P（Partition Tolerance）分区容错性：
- 系统在网络分区时仍能继续运行
- 分布式系统必须满足P

CAP取舍：
- CP：强一致性，可能不可用（Zookeeper、HBase）
- AP：高可用，可能不一致（Cassandra、CouchDB）
- CA：理论上不存在（分区时无法同时满足）
```

#### 1.2 BASE理论
```
BA（Basically Available）基本可用：
- 允许损失部分可用性
- 响应时间增加、功能降级

S（Soft State）软状态：
- 允许数据中间状态
- 数据同步存在延迟

E（Eventually Consistent）最终一致性：
- 经过一段时间后最终一致
- 不要求实时一致

最终一致性实现：
- 读时修复：读取时修复不一致
- 写时修复：写入时修复不一致
- 异步修复：定期检查修复
```

#### 1.3 分布式一致性级别
```
强一致性：
- 所有节点数据实时一致
- 性能差，可用性低

弱一致性：
- 不保证数据一致
- 性能好，可用性高

最终一致性：
- 经过一段时间后一致
- 性能和一致性的平衡

因果一致性：
- 因果关系的事件有序

单调读：
- 用户总是读到较新或相同的数据

单调写：
- 用户的写操作按顺序执行

读己之所写：
- 用户总能读到自己的写

会话一致性：
- 会话内保证因果一致性
```

---

### 2. 分布式一致性算法
**学习日期：____-____-____**

#### 2.1 Paxos算法
```
角色：
- Proposer（提议者）：提出提案
- Acceptor（接受者）：投票表决
- Learner（学习者）：获取结果

算法流程：
Phase 1：Prepare阶段
1. Proposer选择提案编号n，发送Prepare请求给Acceptor
2. Acceptor收到>n的Prepare请求，承诺不再接受编号<n的提案

Phase 2：Accept阶段
1. Proposer收到多数Acceptor响应，发送Accept请求
2. Acceptor收到Accept请求，接受提案

Multi-Paxos：
- 优化Paxos性能
- 跳过Prepare阶段（Leader）
- 减少RPC次数
```

#### 2.2 Raft算法
```
角色：
- Leader（领导者）：处理所有写请求
- Follower（追随者）：接收Leader的日志
- Candidate（候选者）：选举时的临时角色

算法流程：

1. 领导者选举
   - Follower超时未收到心跳 → 转为Candidate
   - Candidate投票给自己 + 请求其他节点投票
   - 获得多数票 → 成为Leader
   - 分散投票 → 重新选举

2. 日志复制
   - Leader接收写请求
   - Leader追加日志到本地
   - Leader复制日志到Follower
   - 多数Follower确认 → 提交日志

3. 安全性保证
   - 选举安全性：新Leader包含所有已提交日志
   - 日志匹配特性：日志连续一致
   - 领导者完整性：Leader不会覆盖已提交日志
```

#### 2.3 ZAB协议（Zookeeper）
```
模式：
- 恢复模式：选举Leader
- 广播模式：同步数据

阶段：
Phase 1：Leader选举
   - myid最大的节点成为Leader
   - 其他节点成为Follower

Phase 2：发现阶段
   - Leader收集Follower的epoch
   - 确定最新的epoch

Phase 3：同步阶段
   - Leader同步数据到Follower
   - 确保多数节点数据一致

Phase 4：广播阶段
   - Leader接收写请求
   - 广播Proposal到Follower
   - 多数Follower确认 → 提交

消息类型：
- REQUEST：请求
- PROPOSAL：提议
- ACK：确认
- COMMIT：提交
- PING：心跳
- CHECKVERSION：版本检查
```

---

### 3. 分布式事务
**学习日期：____-____-____**

#### 3.1 2PC（两阶段提交）
```
参与者：
- 协调者（Coordinator）
- 参与者（Participant）

阶段：
Phase 1：准备阶段
   - 协调者发送准备请求
   - 参与者执行事务但不提交
   - 参与者返回Yes/No

Phase 2：提交阶段
   - 所有参与者Yes → 发送提交
   - 任一参与者No → 发送回滚

缺点：
- 同步阻塞：所有参与者阻塞
- 单点故障：协调者故障导致阻塞
- 数据不一致：网络分区导致不一致
```

#### 3.2 3PC（三阶段提交）
```
改进：
- CanCommit阶段：询问是否可以执行
- PreCommit阶段：预提交（超时机制）
- DoCommit阶段：正式提交

优点：
- 降低阻塞时间
- 减少不一致

缺点：
- 仍然存在阻塞
- 实现复杂
```

#### 3.3 TCC（Try-Confirm-Cancel）
```java
// Try阶段：资源检查和预留
public boolean try(Order order) {
    // 检查库存
    if (stock < order.getCount()) {
        return false;
    }
    // 冻结库存
    stock -= order.getCount();
    frozenStock += order.getCount();
    return true;
}

// Confirm阶段：确认执行业务
public boolean confirm(Order order) {
    // 扣减冻结库存
    frozenStock -= order.getCount();
    return true;
}

// Cancel阶段：取消操作
public boolean cancel(Order order) {
    // 恢复冻结库存
    frozenStock -= order.getCount();
    stock += order.getCount();
    return true;
}

// 特点：
// - 代码侵入大
// - 需要实现三个接口
// - 性能好，无锁
```

#### 3.4 Saga模式
```java
// 定义服务编排
@ServiceOrchestrationFlow
    .addService("reserveInventory", orderService::reserve)
    .addService("createOrder", orderService::create)
    .addService("payment", orderService::payment)
    .addCompensation("cancelPayment", orderService::cancelPayment)
    .addCompensation("cancelOrder", orderService::cancel)
    .addCompensation("releaseInventory", orderService::release)
    .start(order);

// 特点：
// - 长事务支持
// - 状态机管理
// - 最终一致性
```

#### 3.5 本地消息表
```java
// 发送消息
@Transactional
public void createOrder(Order order) {
    // 1. 创建订单
    orderMapper.insert(order);

    // 2. 创建本地消息
    Message message = new Message();
    message.setPayload(JSON.toJSONString(order));
    message.setStatus("PENDING");
    messageMapper.insert(message);

    // 3. 定时任务扫描待发送消息
    // 4. 发送到MQ
    // 5. 更新消息状态
}

// 消费消息
@Transactional
public void handleOrder(Message message) {
    try {
        Order order = JSON.parseObject(message.getPayload(), Order.class);
        // 处理订单
        messageMapper.updateStatus(message.getId(), "CONSUMED");
    } catch (Exception e) {
        // 重试
    }
}

// 特点：
// - 最终一致性
// - 实现简单
// - 可靠性高
```

---

### 4. 分布式锁
**学习日期：____-____-____**

#### 4.1 Redis分布式锁
```java
// 加锁
String lockKey = "lock:product:" + productId;
String lockValue = UUID.randomUUID().toString();
Integer lockExpire = 30;

Boolean locked = redis.setnx(lockKey, lockValue, lockExpire);
if (locked) {
    try {
        // 执行业务逻辑
    } finally {
        // 解锁（Lua脚本）
        String script =
            "if redis.call('get', KEYS[1]) == ARGV[1] then " +
            "    return redis.call('del', KEYS[1]) " +
            "else " +
            "    return 0 " +
            "end";
        redis.eval(script, lockKey, lockValue);
    }
}

// Redisson实现（推荐）
RLock lock = redisson.getLock(lockKey);
try {
    if (lock.tryLock(100, 30, TimeUnit.SECONDS)) {
        // 业务逻辑
    }
} finally {
    lock.unlock();
}
```

#### 4.2 Zookeeper分布式锁
```java
// 创建临时顺序节点
String lockPath = "/locks/product/" + productId;
String currentPath = zk.create(lockPath + "/lock-", null,
    ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

// 获取所有子节点
List<String> children = zk.getChildren(lockPath, false);
Collections.sort(children);

// 判断是否是最小节点
if (currentPath.equals(children.get(0))) {
    // 获得锁
} else {
    // 监听前一个节点
    String prevNode = children.get(Collections.binarySearch(
        children, currentPath.substring(currentPath.lastIndexOf('/') + 1)) - 1);
    final CountDownLatch latch = new CountDownLatch(1);
    zk.exists(lockPath + "/" + prevNode, event -> {
        if (event.getType() == Event.EventType.NodeDeleted) {
            latch.countDown();
        }
    });
    latch.await();
    // 获得锁
}

// Curator实现（推荐）
InterProcessMutex lock = new InterProcessMutex(zkClient, lockPath);
try {
    lock.acquire();
    // 业务逻辑
} finally {
    lock.release();
}
```

#### 4.3 Redis vs Zookeeper
```
Redis：
- 优点：性能高，实现简单
- 缺点：客户端超时，主从切换可能丢失锁

Zookeeper：
- 优点：可靠性高，Watch机制
- 缺点：性能较低，实现复杂
```

---

### 5. 分布式ID
**学习日期：____-____-____**

#### 5.1 雪花算法（Snowflake）
```java
public class SnowflakeIdGenerator {
    private final long twepoch = 1288834974657L;
    private final long workerIdBits = 5L;
    private final long datacenterIdBits = 5L;
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);
    private final long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);
    private final long sequenceBits = 12L;
    private final long workerIdShift = sequenceBits;
    private final long datacenterIdShift = sequenceBits + workerIdBits;
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    private long workerId;
    private long datacenterId;
    private long sequence = 0L;
    private long lastTimestamp = -1L;

    public synchronized long nextId() {
        long timestamp = timeGen();

        if (timestamp < lastTimestamp) {
            throw new RuntimeException("时钟回拨");
        }

        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            if (sequence == 0) {
                timestamp = tilNextMillis(lastTimestamp);
            }
        } else {
            sequence = 0L;
        }

        lastTimestamp = timestamp;

        return ((timestamp - twepoch) << timestampLeftShift)
                | (datacenterId << datacenterIdShift)
                | (workerId << workerIdShift)
                | sequence;
    }

    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    protected long timeGen() {
        return System.currentTimeMillis();
    }
}

// ID结构：
// - 1位符号位
// - 41位时间戳
// - 5位数据中心ID
// - 5位机器ID
// - 12位序列号
```

#### 5.2 其他方案对比
```
UUID：
- 优点：简单，无冲突
- 缺点：无序，过长

数据库自增：
- 优点：简单，递增
- 缺点：性能差，分库分表问题

Redis自增：
- 优点：性能好
- 缺点：依赖Redis，集群问题

号段模式：
- 优点：性能好，可扩展
- 缺点：ID不连续，可能浪费

雪花算法：
- 优点：性能好，有序，可扩展
- 缺点：依赖时钟，时钟回拨问题
```

---

## 学习资源

### 书籍
- 《数据密集型应用系统设计》
- 《从Paxos到Zookeeper》

### 文章
- [Paxos算法详解](https://zhuanlan.zhihu.com/p/31780543)
- [Raft算法详解](https://github.com/maemual/raft-zh_cn)

---

## 学习总结

### 核心收获

### 待深入内容
