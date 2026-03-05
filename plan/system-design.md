# 系统设计学习笔记

## 学习目标
- [ ] 高并发系统设计
- [ ] 高可用系统设计
- [ ] 可扩展架构设计
- [ ] 数据一致性设计
- [ ] 安全性设计
- [ ] 50道系统设计题

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. 高并发系统设计
**学习日期：____-____-____**

#### 1.1 负载均衡策略
```
四层负载均衡（L4）：
- 基于IP+端口
- 性能好，功能简单
- 工具：LVS、Nginx Stream

七层负载均衡（L7）：
- 基于应用层协议
- 功能丰富，性能略差
- 工具：Nginx、HAProxy

负载均衡算法：
- 轮询（Round Robin）
- 加权轮询（Weighted Round Robin）
- 最少连接（Least Connections）
- 一致性Hash（Consistent Hash）
```

#### 1.2 限流算法
```java
// 1. 固定窗口算法
public class FixedWindowRateLimiter {
    private final int limit;
    private final AtomicInteger counter = new AtomicInteger(0);
    private volatile long lastResetTime;

    public boolean allow() {
        long now = System.currentTimeMillis();
        if (now - lastResetTime >= 1000) {
            synchronized (this) {
                if (now - lastResetTime >= 1000) {
                    counter.set(0);
                    lastResetTime = now;
                }
            }
        }
        return counter.incrementAndGet() <= limit;
    }
}

// 2. 滑动窗口算法
public class SlidingWindowRateLimiter {
    private final int limit;
    private final CircularQueue<Long> queue = new CircularQueue<>(limit);

    public boolean allow() {
        long now = System.currentTimeMillis();
        synchronized (this) {
            while (!queue.isEmpty() && now - queue.peek() > 1000) {
                queue.poll();
            }
            if (queue.size() < limit) {
                queue.offer(now);
                return true;
            }
            return false;
        }
    }
}

// 3. 令牌桶算法
public class TokenBucketRateLimiter {
    private final int capacity;      // 桶容量
    private final int rate;          // 令牌生成速率
    private int tokens;              // 当前令牌数
    private long lastTime;

    public boolean allow() {
        long now = System.currentTimeMillis();
        synchronized (this) {
            // 生成令牌
            tokens += (int)((now - lastTime) * rate / 1000.0);
            if (tokens > capacity) tokens = capacity;
            lastTime = now;

            // 消费令牌
            if (tokens > 0) {
                tokens--;
                return true;
            }
            return false;
        }
    }
}

// 4. 漏桶算法
public class LeakyBucketRateLimiter {
    private final int capacity;      // 桶容量
    private final int rate;          // 漏水速率
    private int water;               // 当前水量
    private long lastTime;

    public boolean allow() {
        long now = System.currentTimeMillis();
        synchronized (this) {
            // 漏水
            water -= (int)((now - lastTime) * rate / 1000.0);
            if (water < 0) water = 0;
            lastTime = now;

            // 加水
            if (water < capacity) {
                water++;
                return true;
            }
            return false;
        }
    }
}
```

#### 1.3 降级策略
```java
// 自动降级
@HystrixCommand(fallbackMethod = "fallback")
public String getData() {
    // 业务逻辑
}

public String fallback() {
    // 降级逻辑：返回默认值、缓存、空值等
    return "default";
}

// 人工降级
@Configuration
public class degradeConfig {
    @Value("${degrade.switch:false}")
    private boolean degradeSwitch;

    public String getData() {
        if (degradeSwitch) {
            return "degraded";
        }
        // 正常逻辑
    }
}

// 降级策略：
// - 返回默认值
// - 返回缓存数据
// - 返回静态页面
// - 跳转到降级页面
```

---

### 2. 高可用系统设计
**学习日期：____-____-____**

#### 2.1 集群模式
```
主备模式：
- 主节点提供服务，备节点待命
- 故障切换时间较长
- 成本低

主从模式：
- 主节点写入，从节点读取
- 读写分离，提升性能
- 从节点可能延迟

多主模式：
- 多个主节点同时读写
- 性能好，数据一致性复杂

集群模式：
- 多个节点协同工作
- 自动故障转移
- 成本高
```

#### 2.2 多活架构
```
同地多活：
- 同一地域多个机房
- 机房级别容灾
- 成本适中

异地多活：
- 不同地域多个机房
- 地域级别容灾
- 成本高，数据一致性复杂

单元化：
- 按用户维度拆分单元
- 单元内自闭环
- 扩展性好

接入层多活：
- DNS智能解析
- 负载均衡全局调度
- 就近接入
```

#### 2.3 故障转移
```java
// 健康检查
@Configuration
public class HealthCheckConfig {
    @Scheduled(fixedRate = 5000)
    public void healthCheck() {
        // 检查服务健康状态
        // 不健康则摘除
    }
}

// 自动故障转移
@Configuration
public class FailoverConfig {
    private List<String> servers = Arrays.asList(
        "server1", "server2", "server3"
    );
    private int currentIndex = 0;

    public String callServer() {
        for (int i = 0; i < servers.size(); i++) {
            String server = servers.get(currentIndex);
            try {
                // 调用服务
                return server;
            } catch (Exception e) {
                // 故障切换
                currentIndex = (currentIndex + 1) % servers.size();
            }
        }
        throw new ServiceException("All servers failed");
    }
}

// 隔离熔断
@HystrixCommand(
    commandProperties = {
        @HystrixProperty(name =
            "circuitBreaker.requestVolumeThreshold", value = "10"),
        @HystrixProperty(name =
            "circuitBreaker.errorThresholdPercentage", value = "50"),
        @HystrixProperty(name =
            "circuitBreaker.sleepWindowInMilliseconds", value = "10000")
    }
)
public String callService() {
    // 调用服务
}
```

---

### 3. 可扩展架构设计
**学习日期：____-____-____**

#### 3.1 垂直扩展 vs 水平扩展
```
垂直扩展（Scale Up）：
- 升级硬件配置
- 简单快速
- 有上限，成本高

水平扩展（Scale Out）：
- 增加机器数量
- 无限扩展
- 需要架构支持
```

#### 3.2 分层架构
```
接入层：
- DNS
- CDN
- 负载均衡

网关层：
- API网关
- 路由转发
- 限流熔断

应用层：
- 业务服务
- 微服务集群

数据层：
- 缓存
- 数据库
- 消息队列
```

#### 3.3 微服务拆分
```
拆分原则：
- 单一职责
- 高内聚低耦合
- 业务边界清晰

拆分维度：
- 按业务领域
- 按功能模块
- 按数据维度

拆分步骤：
1. 识别业务边界
2. 定义服务接口
3. 数据拆分
4. 逐步迁移
5. 验证上线
```

---

### 4. 系统设计面试题
**学习日期：____-____-____**

#### 题目1：设计秒杀系统
```
核心问题：
- 超高并发（百万级QPS）
- 库存扣减（一致性）
- 防刷（安全性）

解决方案：
1. 分层限流
   - 接入层：CDN限流
   - 网关层：令牌桶限流
   - 应用层：Redis计数器

2. 库存设计
   - Redis预减库存
   - MQ异步下单
   - 数据库最终一致

3. 防刷策略
   - 验证码
   - 限流（IP、用户）
   - 黑名单

4. 技术选型
   - 缓存：Redis
   - MQ：RocketMQ
   - 数据库：MySQL分库分表
```

#### 题目2：设计微博Feed流
```
核心问题：
- 写扩散 vs 读扩散
- 推拉结合
- 存储优化

解决方案：
1. 推模式（写扩散）
   - 发微博时推送到粉丝收件箱
   - 适合粉丝少场景

2. 拉模式（读扩散）
   - 读取时拉取关注人微博
   - 适合粉丝多场景

3. 推拉结合
   - 大V：拉模式
   - 普通用户：推模式
   - 热点缓存

4. 存储优化
   - Redis存Feed
   - 定期归档
   - 分页加载
```

#### 题目3：设计短链接系统
```
核心问题：
- 短链接生成
- 长短映射
- 跳转统计

解决方案：
1. 短链接生成
   - ID自增 + Base62编码
   - 雪花算法 + Base62编码
   - Hash算法（冲突处理）

2. 存储设计
   - Redis缓存热点数据
   - MySQL持久化
   - 布隆过滤器判断存在

3. 性能优化
   - CDN缓存
   - 异步统计
   - 批量写入

4. 扩展性
   - 按ID分片
   - 按Hash分片
```

---

## 实战项目

### 项目：秒杀系统设计
**项目地址**：`projects/seckill-system/`

#### 架构设计
- [ ] 架构图绘制
- [ ] 技术选型文档
- [ ] 接口设计文档
- [ ] 数据库设计文档

---

## 学习资源

### 书籍
- 《凤凰架构》周志明
- 《系统设计面试》

### 网站
- [系统设计面试](https://github.com/donnemartin/system-design-primer)
- [阿里技术](https://developer.aliyun.com/)

---

## 50道系统设计题

### 初级（1-20）
- [ ] 1. 设计URL短链服务
- [ ] 2. 设计聊天室系统
- [ ] 3. 设计文件存储系统
- [ ] 4. 设计新闻Feed流系统
- [ ] 5. 设计日志收集系统
- [ ] 6. 设计爬虫系统
- [ ] 7. 设计搜索引擎
- [ ] 8. 设计推荐系统
- [ ] 9. 设计限流系统
- [ ] 10. 设计配置中心
- [ ] 11. 设计任务调度系统
- [ ] 12. 设计分布式锁
- [ ] 13. 设计分布式事务
- [ ] 14. 设计监控告警系统
- [ ] 15. 设计灰度发布系统
- [ ] 16. 设计AB测试系统
- [ ] 17. 设计会员积分系统
- [ ] 18. 设计优惠券系统
- [ ] 19. 设计评论系统
- [ ] 20. 设计点赞系统

### 中级（21-35）
- [ ] 21. 设计秒杀系统
- [ ] 22. 设计抢红包系统
- [ ] 23. 设计IM即时通讯
- [ ] 24. 设计视频直播系统
- [ ] 25. 设计支付系统
- [ ] 26. 设计订单系统
- [ ] 27. 设计购物车系统
- [ ] 28. 设计库存系统
- [ ] 29. 设计分库分表系统
- [ ] 30. 设计读写分离系统
- [ ] 31. 设计API网关
- [ ] 32. 设计服务注册中心
- [ ] 33. 设计配置中心
- [ ] 34. 设计链路追踪系统
- [ ] 35. 设计日志分析系统

### 高级（36-50）
- [ ] 36. 设计分布式数据库
- [ ] 37. 设计分布式缓存
- [ ] 38. 设计分布式文件系统
- [ ] 39. 设计分布式搜索引擎
- [ ] 40. 设计分布式消息队列
- [ ] 41. 设计分布式任务调度
- [ ] 42. 设计分布式事务协调器
- [ ] 43. 设计分布式ID生成器
- [ ] 44. 设计分布式限流系统
- [ ] 45. 设计分布式配置中心
- [ ] 46. 设计服务网格
- [ ] 47. 设计云原生应用平台
- [ ] 48. 设计容器编排系统
- [ ] 49. 设计Serverless平台
- [ ] 50. 设计多云管理平台

---

## 学习总结

### 核心收获

### 待深入内容
