# 秒杀系统

## 项目概述
高并发秒杀系统，支持10万QPS，实现库存扣减、限流降级、防刷等功能。

## 技术栈
- Spring Boot 2.7
- Spring Cloud Alibaba
- Redis 7
- RocketMQ 5
- MySQL 8
- Nacos 2
- Sentinel 1.8

## 项目进度
**状态**：⬜ 未开始
**完成度**：0%

## 模块列表

### 1. 商品模块 (product-service)
**功能**：
- [ ] 商品管理
- [ ] 库存管理
- [ ] 商品查询
- [ ] 库存预减（Redis）

**状态**：⬜ 未开始

---

### 2. 秒杀模块 (seckill-service)
**功能**：
- [ ] 秒杀活动管理
- [ ] 秒杀商品查询
- [ ] 秒杀下单
- [ ] Redis Lua脚本扣库存
- [ ] 限流（Sentinel）
- [ ] 降级策略

**状态**：⬜ 未开始

---

### 3. 订单模块 (order-service)
**功能**：
- [ ] 订单创建
- [ ] 订单查询
- [ ] 订单状态管理
- [ ] MQ异步下单
- [ ] 分布式事务（Seata）

**状态**：⬜ 未开始

---

### 4. 用户模块 (user-service)
**功能**：
- [ ] 用户注册
- [ ] 用户登录
- [ ] 用户信息
- [ ] 防刷（验证码、限流）

**状态**：⬜ 未开始

---

### 5. 网关模块 (gateway-service)
**功能**：
- [ ] 路由转发
- [ ] 限流（Sentinel）
- [ ] 鉴权
- [ ] 日志记录

**状态**：⬜ 未开始

---

## 核心技术

### 1. 高并发处理
```
- CDN静态资源加速
- 页面缓存（Redis）
- 对象缓存（Redis）
- 页面静态化
- 异步下单（RocketMQ）
```

### 2. 库存扣减
```
- Redis预减库存
- Lua脚本保证原子性
- 分布式锁防止超卖
- MQ异步扣减真实库存
```

### 3. 限流降级
```
- 接入层：CDN限流
- 网关层：Sentinel限流
- 应用层：Redis计数器
- 降级：返回默认值
```

### 4. 防刷策略
```
- 验证码
- IP限流
- 用户限流
- 黑名单
- 设备指纹
```

---

## 项目结构
```
seckill-system/
├── product-service/      # 商品服务
├── seckill-service/      # 秒杀服务
├── order-service/        # 订单服务
├── user-service/         # 用户服务
├── gateway-service/      # 网关服务
├── common/               # 公共模块
└── docs/                 # 文档
    ├── design.md         # 设计文档
    ├── api.md            # API文档
    └── deploy.md         # 部署文档
```

---

## 快速开始

### 环境要求
- JDK 11+
- Maven 3.6+
- Redis 7
- RocketMQ 5
- MySQL 8
- Nacos 2

### 启动步骤
```bash
# 1. 启动基础环境
docker-compose up -d

# 2. 启动Nacos
sh startup.sh -m standalone

# 3. 启动服务
# 按顺序启动：user -> product -> order -> seckill -> gateway

# 4. 访问网关
http://localhost:8080
```

---

## 性能指标

| 指标 | 目标 | 实际 |
|------|------|------|
| QPS | 100,000 | - |
| 响应时间 | < 100ms | - |
| 可用性 | 99.9% | - |
| 并发用户 | 10,000+ | - |

---

## 后续计划
- [ ] 项目搭建（第1周）
- [ ] 商品模块开发（第2周）
- [ ] 秒杀模块开发（第3-4周）
- [ ] 订单模块开发（第5周）
- [ ] 性能优化（第6周）
- [ ] 压测验证（第7周）
- [ ] 文档完善（第8周）

---

## 学习资源
- 尚硅谷秒杀系统实战
- 《亿级流量网站架构核心技术》
- [秒杀系统设计文档](../docs/system-design-docs/seckill-system.md)
