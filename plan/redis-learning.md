# Redis 精通学习笔记

## 学习目标
- [ ] 5种基本数据结构 + 4种高级数据结构
- [ ] 持久化机制（RDB、AOF）
- [ ] 主从复制、哨兵、Cluster集群
- [ ] 缓存设计（缓存穿透、击穿、雪崩、热点数据）
- [ ] 分布式锁实现
- [ ] Redis 6.0新特性（多线程IO、客户端缓存）

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. 数据结构深度解析
**学习日期：____-____-____**

#### 1.1 五种基本数据结构
```bash
# 1. String（字符串）
SET key value
GET key
INCR key        # 自增
DECR key        # 自减
INCRBY key num  # 指定增量

# 应用场景：
# - 计数器（点赞数、阅读量）
# - 分布式锁
# - 分布式ID生成器
# - Session共享

# 底层实现：SDS（Simple Dynamic String）
# 优势：二进制安全、O(1)时间复杂度获取长度


# 2. Hash（哈希）
HSET user:1 name "张三"
HGET user:1 name
HGETALL user:1
HINCRBY user:1 age 1

# 应用场景：
# - 对象存储
# - 购物车
# - 用户信息

# 底层实现：ziplist（小对象）或 hashtable（大对象）


# 3. List（列表）
LPUSH list value      # 左插入
RPUSH list value      # 右插入
LPOP list             # 左弹出
RPOP list             # 右弹出
LRANGE list 0 -1      # 获取所有元素
BLPOP list timeout    # 阻塞弹出

# 应用场景：
# - 消息队列
# - 最新列表
# - 关注列表

# 底层实现：quicklist（双向链表 + ziplist）


# 4. Set（集合）
SADD set member
SREM set member
SMEMBERS set
SISMEMBER set member
SINTER set1 set2      # 交集
SUNION set1 set2      # 并集
SDIFF set1 set2       # 差集

# 应用场景：
# - 标签系统
# - 共同关注
# - 抽奖系统

# 底层实现：intset（整数集合）或 hashtable


# 5. ZSet（有序集合）
ZADD zset score member
ZREM zset member
ZRANGE zset 0 -1 WITHSCORES
ZRANK zset member           # 排名
ZSCORE zset member          # 分数
ZRANGEBYSCORE zset min max  # 分数范围查询

# 应用场景：
# - 排行榜
# - 延迟队列
# - 范围查询

# 底层实现：skiplist（跳表）+ hashtable
```

#### 1.2 四种高级数据结构
```bash
# 1. Bitmap（位图）
SETBIT key offset value
GETBIT key offset
BITCOUNT key
BITOP op destkey key1 key2  # 位运算

# 应用场景：
# - 用户签到
# - 在线状态
# - 布隆过滤器


# 2. HyperLogLog（基数统计）
PFADD key element
PFCOUNT key
PFMERGE destkey key1 key2

# 应用场景：
# - UV统计
# - 去重统计
# - 误差率：0.81%


# 3. GEO（地理位置）
GEOADD key longitude latitude member
GEODIST key member1 member2 unit
GEOPOS key member
GEORADIUS key longitude latitude radius unit

# 应用场景：
# - 附近的人
# - 距离计算
# - 位置服务


# 4. Stream（流）
XADD stream * field value
XREAD COUNT 2 STREAMS stream $
XGROUP CREATE stream group $
XREADGROUP GROUP group consumer STREAMS stream >

# 应用场景：
# - 消息队列
# - 事件流
```

---

### 2. 持久化机制
**学习日期：____-____-____**

#### 2.1 RDB（快照持久化）
```bash
# RDB配置
save 900 1      # 900秒内至少1个key变化
save 300 10     # 300秒内至少10个key变化
save 60 10000   # 60秒内至少10000个key变化

# 优点：
# - 文件紧凑，适合备份
# - 恢复速度快
# - 对性能影响小

# 缺点：
# - 可能丢失最后一次快照后的数据
# - fork子进程可能阻塞

# 手动触发
BGSAVE  # 后台保存
SAVE    # 同步保存（阻塞）
```

#### 2.2 AOF（追加文件）
```bash
# AOF配置
appendonly yes
appendfilename "appendonly.aof"
appendfsync everysec  # no/everysec/always

# AOF重写
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

# 优点：
# - 数据安全性高（最多丢失1秒数据）
# - AOF文件可读

# 缺点：
# - 文件体积大
# - 恢复速度慢
# - 性能影响较大

# AOF重写
BGREWRITEAOF
```

#### 2.3 混合持久化（Redis 4.0+）
```bash
# RDB-AOF混合模式
aof-use-rdb-preamble yes

# 原理：
# - AOF文件前半部分是RDB格式
# - 后半部分是AOF格式
# - 结合两者优点
```

---

### 3. 主从复制
**学习日期：____-____-____**

#### 3.1 复制原理
```
1. Slave连接Master，发送SYNC命令
2. Master执行BGSAVE，生成RDB文件
3. Master发送RDB文件给Slave
4. Master发送缓冲区的写命令给Slave
5. Slave加载RDB文件，执行写命令
6. 后续持续同步
```

#### 3.2 全量同步 vs 增量同步
```bash
# 全量同步
# - Slave首次连接或Repl-Backlog不足
# - 传输RDB文件

# 增量同步
# - 复制偏移量在Repl-Backlog内
# - 只传输差异部分
```

#### 3.3 主从配置
```bash
# Master配置
port 6379
bind 0.0.0.0

# Slave配置
port 6380
slaveof master_ip master_port
masterauth master_password
slave-read-only yes
```

---

### 4. 哨兵模式（Sentinel）
**学习日期：____-____-____**

#### 4.1 哨兵原理
```
1. 哨兵定期ping Master和Slave
2. 多数哨兵确认Master主观下线（SDOWN）
3. 哨兵之间投票确认客观下线（ODOWN）
4. 选举Leader哨兵
5. Leader执行故障转移
   - 从Slave中选举新Master
   - 其他Slave指向新Master
   - 通知客户端新Master地址
```

#### 4.2 哨兵配置
```bash
# sentinel.conf
port 26379
sentinel monitor mymaster master_ip 6379 quorum
sentinel auth-pass mymaster password
sentinel down-after-milliseconds mymaster 5000
sentinel parallel-syncs mymaster 1
sentinel failover-timeout mymaster 10000
```

---

### 5. Cluster集群
**学习日期：____-____-____**

#### 5.1 集群原理
```
1. 数据分片（16384个槽位）
2. 每个节点负责部分槽位
3. 节点之间通过Gossip协议通信
4. 自动故障转移
5. 支持在线扩容缩容
```

#### 5.2 槽位分配
```bash
# 查看槽位分配
CLUSTER NODES
CLUSTER SLOTS

# 手动分配槽位
CLUSTER ADDSLOTS slot1 [slot2 ...]
CLUSTER DELSLOTS slot1 [slot2 ...]
CLUSTER SETSLOT slot IMPORTING node_id
CLUSTER SETSLOT slot MIGRATING node_id
CLUSTER GETKEYSINSLOT slot count
CLUSTER KEYSLOT key
```

#### 5.3 集群配置
```bash
# redis.conf
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 5000
cluster-require-full-coverage yes

# 创建集群
redis-cli --cluster create ip1:port1 ip2:port2 ip3:port3 \
  --cluster-replicas 1
```

---

### 6. 缓存设计
**学习日期：____-____-____**

#### 6.1 缓存穿透（查询不存在的数据）
```java
// 解决方案1：布隆过滤器
BloomFilter<String> filter = BloomFilter.create(
    Funnels.stringFunnel(),
    10000,
    0.01
);

// 解决方案2：缓存空对象
String value = redis.get(key);
if (value == null) {
    value = db.get(key);
    if (value == null) {
        redis.setex(key, 30, "");  // 缓存空对象
    } else {
        redis.set(key, value);
    }
}
```

#### 6.2 缓存击穿（热点数据过期）
```java
// 解决方案1：互斥锁
String value = redis.get(key);
if (value == null) {
    if (redis.setnx(lock_key, 1)) {
        redis.expire(lock_key, 30);
        value = db.get(key);
        redis.set(key, value);
        redis.del(lock_key);
    } else {
        Thread.sleep(100);
        return get(key);  // 重试
    }
}

// 解决方案2：热点数据永不过期
// 逻辑过期时间
```

#### 6.3 缓存雪崩（大量数据同时过期）
```java
// 解决方案1：随机过期时间
int expireTime = baseTime + new Random().nextInt(300);
redis.setex(key, expireTime, value);

// 解决方案2：缓存预热
// 解决方案3：高可用集群
// 解决方案4：限流降级
```

#### 6.4 热点数据发现
```java
// 使用LFU（Least Frequently Used）策略
redis.set(key, value);
redis.object("freq", key);  // 查看访问频率

// 热点Key自动发现
// Redis 4.0+ 提供
CONFIG SET hotspot-key-detector yes
CONFIG SET hotspot-key-sample-num 10
```

---

### 7. 分布式锁
**学习日期：____-____-____**

#### 7.1 基本实现
```java
// 加锁
String lockKey = "lock:product:" + productId;
String lockValue = UUID.randomUUID().toString();

Boolean locked = redis.setnx(lockKey, lockValue);
if (locked) {
    redis.expire(lockKey, 30);  // 设置过期时间
    // 执行业务逻辑
}

// 解锁
String script =
    "if redis.call('get', KEYS[1]) == ARGV[1] then " +
    "    return redis.call('del', KEYS[1]) " +
    "else " +
    "    return 0 " +
    "end";
redis.eval(script, Collections.singletonList(lockKey),
           Collections.singletonList(lockValue));
```

#### 7.2 Redisson实现
```java
// 使用Redisson分布式锁
RedissonClient redisson = Redisson.create(config);
RLock lock = redisson.getLock("lock:product:" + productId);

try {
    // 尝试加锁，最多等待100秒，锁定30秒
    if (lock.tryLock(100, 30, TimeUnit.SECONDS)) {
        // 执行业务逻辑
    }
} finally {
    lock.unlock();
}
```

---

### 8. Redis 6.0新特性
**学习日期：____-____-____**

#### 8.1 多线程IO
```bash
# 配置
io-threads 4
io-threads-do-reads yes

# 特点：
# - 只在网络IO阶段使用多线程
# - 命令执行仍是单线程
# - 性能提升约2倍
```

#### 8.2 客户端缓存
```bash
# 服务端配置
tracking on  # 开启跟踪
client-caching yes

# 使用场景：
# - 减少网络开销
# - 提升读取性能
```

---

## 实战项目

### 项目：缓存系统设计
**项目地址**：`projects/redis-cache-system/`

#### 项目目标
- [ ] 实现多级缓存
- [ ] 实现缓存预热
- [ ] 实现缓存监控
- [ ] 解决缓存常见问题

---

## 学习资源

### 书籍
- 《Redis设计与实现》黄健宏
- 《Redis实战》Josiah L. Carlson

### 视频
- 尚硅谷Redis6

### 官方文档
- [Redis官方文档](https://redis.io/documentation)

---

## 学习总结

### 核心收获

### 待深入内容
