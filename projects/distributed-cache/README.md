# 分布式缓存框架

> 实战项目：多级缓存实现

---

## 🎯 项目目标

实现一个高性能的分布式缓存框架，掌握：

- [ ] 多级缓存设计
- [ ] 缓存一致性
- [ ] 缓存穿透/击穿/雪崩解决方案
- [ ] 热点数据处理
- [ ] 缓存监控

---

## 📋 核心功能

- [ ] 本地缓存 (Caffeine)
- [ ] 分布式缓存 (Redis)
- [ ] 缓存一致性
- [ ] 缓存预热
- [ ] 缓存监控

---

## 🏗️ 架构设计

```
┌─────────────────────────────────────────────────────────────┐
│                       多级缓存架构                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [应用层]                                                    │
│      │                                                      │
│      ├──> [L1: 本地缓存] (Caffeine)                         │
│      │     ├── 容量: 1000                                  │
│      │     ├── TTL: 5min                                   │
│      │     └── 淘汰策略: LRU                                │
│      │                                                      │
│      │     Miss ─────────┐                                 │
│      │                   │                                 │
│      ▼                   ▼                                 │
│  [L2: Redis] ───────── [热点缓存]                          │
│    │                   │                                   │
│    │     Miss ─────────┘                                   │
│    │                                                      │
│    ▼                                                      │
│  [数据库]                                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 💡 核心设计

### 1. 缓存读取

```java
public Object get(String key) {
    // L1: 本地缓存
    Object value = localCache.get(key);
    if (value != null) {
        return value;
    }

    // L2: Redis 缓存
    value = redisCache.get(key);
    if (value != null) {
        localCache.put(key, value);
        return value;
    }

    // DB: 数据库
    value = db.get(key);
    if (value != null) {
        redisCache.put(key, value);
        localCache.put(key, value);
    }

    return value;
}
```

### 2. 缓存更新

```java
public void update(String key, Object value) {
    // 1. 更新数据库
    db.update(key, value);

    // 2. 删除 Redis 缓存
    redisCache.delete(key);

    // 3. 删除本地缓存 (或通过消息通知其他节点)
    localCache.delete(key);
    publishCacheInvalidation(key);
}
```

### 3. 缓存穿透解决方案

```java
public Object getWithPenetrationProtect(String key) {
    // 1. 布隆过滤器判断
    if (!bloomFilter.mightContain(key)) {
        return null;
    }

    // 2. 缓存空值
    Object value = redisCache.get(key);
    if (value != null) {
        return value == NULL_VALUE ? null : value;
    }

    // 3. 查询数据库
    value = db.get(key);
    if (value == null) {
        redisCache.put(key, NULL_VALUE, 5); // 缓存空值5分钟
    } else {
        redisCache.put(key, value);
    }

    return value;
}
```

### 4. 缓存击穿解决方案

```java
public Object getWithBreakdownProtect(String key) {
    // 互斥锁
    String lockKey = "lock:" + key;
    try {
        // 尝试获取锁
        if (redisCache.setNx(lockKey, 1, 10)) {
            // 获取锁成功，查库并回写缓存
            Object value = db.get(key);
            if (value != null) {
                redisCache.put(key, value);
            }
            return value;
        } else {
            // 获取锁失败，短暂休眠后重试
            Thread.sleep(50);
            return get(key);
        }
    } finally {
        redisCache.delete(lockKey);
    }
}
```

### 5. 缓存雪崩解决方案

```java
// 1. 缓存过期时间加随机值
redisCache.put(key, value, BASE_TTL + Random.nextInt(300));

// 2. 熔断降级
if (circuitBreaker.isOpen()) {
    return fallbackValue;
}

// 3. 多级缓存
// 详见架构设计
```

---

## 📊 开发计划

| 模块 | 预计时间 | 状态 |
|------|----------|------|
| 基础框架 | 2天 | 📋 |
| 本地缓存集成 | 1天 | 📋 |
| Redis 集成 | 1天 | 📋 |
| 一致性保证 | 3天 | 📋 |
| 监控面板 | 2天 | 📋 |
| 压测优化 | 2天 | 📋 |

---

## 🚀 技术栈

- **本地缓存**: Caffeine
- **分布式缓存**: Redis
- **消息队列**: RocketMQ (缓存失效通知)
- **监控**: Micrometer + Prometheus

---

## 📝 学习笔记

相关笔记：[Redis 深入](../../docs/phase2-distributed/redis.md)

---

## 🔗 参考资料

- [Caffeine 官方文档](https://github.com/ben-manes/caffeine)
- [Redis 缓存设计](https://redis.io/docs/manual/patterns/)
