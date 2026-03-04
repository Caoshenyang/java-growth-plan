# 秒杀系统设计

> 实战项目：高并发系统设计

---

## 🎯 项目目标

设计并实现一个完整的秒杀系统，掌握：

- [ ] 高并发架构设计
- [ ] 缓存设计与优化
- [ ] 消息队列应用
- [ ] 分布式锁
- [ ] 限流降级策略
- [ ] 防刷机制

---

## 📋 核心功能

- [ ] 商品管理
- [ ] 秒杀活动配置
- [ ] 秒杀下单
- [ ] 库存扣减
- [ ] 订单支付
- [ ] 防刷限流

---

## 🏗️ 系统架构

```
┌─────────────────────────────────────────────────────────────┐
│                        秒杀系统架构                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  [用户]                                                      │
│    │                                                         │
│    ▼                                                         │
│  [CDN] ─────── 静态资源 (页面、图片、JS)                     │
│    │                                                         │
│    ▼                                                         │
│  [网关] ─────── 限流、防刷、验证码                           │
│    │                                                         │
│    ▼                                                         │
│  [秒杀服务] ───── 令牌桶、排队、库存预检                     │
│    │              │                                         │
│    │              ▼                                         │
│    │         [Redis] ─── 库存预扣减 (Lua)                   │
│    │              │                                         │
│    │              ▼                                         │
│    │         [MQ] ───── 异步下单                            │
│    │              │                                         │
│    ▼              ▼                                         │
│  [订单服务] ← [库存服务]                                    │
│    │              │                                         │
│    ▼              ▼                                         │
│  [DB]          [DB]                                         │
│    │              │                                         │
│    ▼              ▼                                         │
│  [支付服务] ──── 支付回调                                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 📁 项目结构

```
seckill-system/
├── seckill-gateway/          # 网关服务
│   ├── RateLimitFilter.java
│   ├── ValidateFilter.java
│   └── BlacklistFilter.java
├── seckill-activity/         # 活动服务
│   ├── ActivityController.java
│   └── ActivityService.java
├── seckill-core/            # 秒杀核心服务
│   ├── SeckillController.java
│   ├── SeckillService.java
│   ├── Redis/
│   │   └── StockDeductService.java
│   └── Queue/
│       └── QueueService.java
├── seckill-order/           # 订单服务
│   ├── OrderController.java
│   └── OrderService.java
├── seckill-stock/           # 库存服务
│   └── StockService.java
├── seckill-payment/         # 支付服务
│   └── PaymentService.java
└── seckill-common/          # 公共模块
    ├── Lock/
    │   └── DistributedLock.java
    └── Limit/
        └── RateLimiter.java
```

---

## 💡 核心设计

### 1. 库存扣减 (Redis + Lua)

```lua
-- 库存扣减脚本
local stock = redis.call('get', KEYS[1])
if tonumber(stock) >= tonumber(ARGV[1]) then
    return redis.call('decrby', KEYS[1], ARGV[1])
else
    return -1
end
```

### 2. 限流策略

```java
// 多级限流
1. 用户级限流: 每用户每秒最多1次请求
2. 接口级限流: 每秒最多10000次请求
3. IP级限流: 每IP每秒最多10次请求
```

### 3. 防刷机制

```java
1. 验证码: 图形验证码、滑块验证
2. 签名校验: 请求参数签名
3. 时间戳: 防止重放攻击
4. 黑名单: 恶意IP封禁
```

### 4. 消息队列

```java
// 异步下单流程
1. 用户请求 -> 预扣库存 -> 入队
2. 消费队列 -> 创建订单
3. 支付成功 -> 确认扣库存
4. 支付失败 -> 回滚库存
```

---

## 📊 开发计划

| 模块 | 预计时间 | 状态 |
|------|----------|------|
| 需求分析 | 1天 | 📋 |
| 架构设计 | 2天 | 📋 |
| 数据库设计 | 1天 | 📋 |
| 网关服务 | 3天 | 📋 |
| 核心服务 | 5天 | 📋 |
| 订单服务 | 3天 | 📋 |
| 库存服务 | 3天 | 📋 |
| 支付集成 | 2天 | 📋 |
| 压测优化 | 3天 | 📋 |

---

## 🚀 技术栈

- **框架**: Spring Cloud Alibaba
- **注册中心**: Nacos
- **网关**: Spring Cloud Gateway
- **缓存**: Redis Cluster
- **消息队列**: RocketMQ
- **数据库**: MySQL + ShardingSphere
- **限流**: Sentinel
- **监控**: Prometheus + Grafana
- **追踪**: SkyWalking

---

## 📝 学习笔记

相关笔记：[高并发系统设计](../../docs/phase3-architecture/high-concurrency.md)

---

## 🔗 参考资料

- [美团技术团队：秒杀系统设计](https://tech.meituan.com/2016/09/28/seckill-system-design.html)
