# 面试准备笔记

## 学习目标
- [ ] 简历优化
- [ ] 项目梳理
- [ ] 技术知识点复习
- [ ] 系统设计题练习
- [ ] 模拟面试

## 准备周期
- 开始时间：____-____-____
- 目标面试时间：____-____-____

## 学习笔记

### 1. 简历优化
**完成日期：____-____-____**

#### 1.1 个人信息
```
姓名：XXX
手机：138-XXXX-XXXX
邮箱：XXX@example.com
工作年限：8年
求职意向：Java架构师/技术专家
期望薪资：30-35K
```

#### 1.2 专业技能
```
【Java基础】
- 深入理解JVM（内存模型、GC算法、类加载机制）
- 精通并发编程（JUC源码、线程池、锁机制）
- 熟练掌握集合框架、IO/NIO、反射等核心特性

【框架技术】
- 精通Spring、Spring Boot、Spring Cloud Alibaba
- 熟悉MyBatis、Hibernate等ORM框架
- 了解Netty、Tomcat等网络编程框架

【中间件】
- 精通Redis（数据结构、持久化、集群、缓存设计）
- 熟练使用RocketMQ、Kafka等消息队列
- 了解Elasticsearch、Zookeeper等中间件

【数据库】
- 精通MySQL（索引优化、事务隔离、分库分表）
- 熟悉SQL优化、慢查询分析
- 了解NoSQL数据库（MongoDB、HBase）

【分布式系统】
- 理解CAP、BASE理论
- 熟悉分布式事务（2PC、TCC、Saga、本地消息表）
- 掌握分布式锁、分布式ID生成器

【微服务架构】
- 精通Spring Cloud Alibaba全家桶
- 熟悉服务注册发现、配置中心、API网关
- 了解服务网格（Istio）

【云原生技术】
- 熟练使用Docker容器化
- 掌握Kubernetes核心概念和应用
- 了解DevOps（Jenkins、GitLab CI）

【系统设计】
- 能够独立设计高并发、高可用系统
- 熟悉性能优化、系统调优
- 掌握架构设计方法和技术选型
```

#### 1.3 项目经验
```
项目1：电商平台微服务系统
技术栈：Spring Cloud Alibaba + Docker + K8s + Redis + RocketMQ + MySQL
项目描述：负责电商平台微服务架构设计与实现
核心职责：
- 主导微服务拆分（用户、商品、订单、支付服务）
- 设计并实现服务治理（限流、熔断、降级）
- 实现分布式事务（Seata）
- 性能优化（QPS从1K提升到10K）
技术亮点：
- 实现基于Nacos的服务注册与配置中心
- 使用Sentinel实现限流熔断
- 使用Redis实现多级缓存
- 使用RocketMQ实现异步解耦

项目2：秒杀系统
技术栈：Spring Boot + Redis + RocketMQ + MySQL
项目描述：设计并实现高并发秒杀系统
核心职责：
- 负责秒杀系统架构设计
- 实现Redis预减库存
- 使用RocketMQ异步下单
- 实现防刷策略
技术亮点：
- QPS达到10万
- 使用Redis Lua脚本保证原子性
- 使用布隆过滤器防止缓存穿透
- 实现分布式锁解决超卖问题

项目3：分布式任务调度系统
技术栈：Spring Boot + Netty + MySQL + Redis
项目描述：自研分布式任务调度系统
核心职责：
- 设计系统架构
- 实现任务调度核心逻辑
- 实现故障转移和负载均衡
技术亮点：
- 支持CRON表达式
- 支持任务分片执行
- 支持故障自动转移
- 实现任务监控和告警
```

---

### 2. 技术知识点
**复习日期：____-____-____**

#### 2.1 Java基础
```java
// HashMap源码
// 1. 数据结构：数组 + 链表 + 红黑树
// 2. 扰动函数：(h = key.hashCode()) ^ (h >>> 16)
// 3. 容量计算：tableSizeFor(initialCapacity)
// 4. 扩容时机：size > threshold（capacity * loadFactor）
// 5. 扩容方法：resize()
// 6. 红黑树转换：链表长度 > 8 且 容量 > 64

// ConcurrentHashMap源码
// JDK1.7：Segment分段锁
// JDK1.8：Node数组 + CAS + synchronized

// ThreadPoolExecutor源码
// 核心参数：
// - corePoolSize：核心线程数
// - maximumPoolSize：最大线程数
// - workQueue：阻塞队列
// - keepAliveTime：非核心线程存活时间
// - handler：拒绝策略

// 工作流程：
// 1. 核心线程数未满 → 创建核心线程
// 2. 核心线程数已满 → 放入队列
// 3. 队列已满 → 创建非核心线程
// 4. 达到最大线程数 → 执行拒绝策略
```

#### 2.2 JVM
```
// 内存模型
// - 堆：对象实例
// - 栈：方法调用、局部变量
// - 方法区：类信息、常量、静态变量
// - 程序计数器：字节码行号
// - 本地方法栈：Native方法

// GC算法
// - Serial：单线程，STW长
// - ParNew：多线程，STW较长
// - CMS：并发标记，有碎片
// - G1：Region，可预测停顿
// - ZGC：并发整理，极低停顿

// 类加载
// - 加载：获取二进制流
// - 验证：文件格式、字节码、符号引用
// - 准备：静态变量赋初值
// - 解析：符号引用转直接引用
// - 初始化：静态变量赋真值

// 双亲委派模型
// - Bootstrap ClassLoader（启动类加载器）
// - Extension ClassLoader（扩展类加载器）
// - Application ClassLoader（应用类加载器）
```

#### 2.3 并发编程
```java
// synchronized原理
// - 对象头：Mark Word（锁状态）
// - 锁升级：偏向锁 → 轻量级锁 → 重量级锁
// - Monitor：管程对象

// volatile原理
// - 内存屏障（Memory Barrier）
// - 可见性：MESI缓存一致性协议
// - 有序性：禁止指令重排序

// AQS原理
// - state：同步状态
// - CLH队列：等待队列
// - 独占/共享模式
// - 公平/非公平获取

// CAS原理
// - Compare And Swap
// - Unsafe类
// - CPU指令：cmpxchg
// - ABA问题：AtomicStampedReference
```

---

### 3. 系统设计题
**练习日期：____-____-____**

#### 题目1：设计限流系统
```
需求：
- 支持QPS限流
- 支持并发数限流
- 支持IP、用户维度限流
- 支持动态配置

方案：
1. 限流算法
   - 固定窗口
   - 滑动窗口
   - 令牌桶
   - 漏桶

2. 技术选型
   - Redis + Lua脚本
   - Sentinel
   - 自研限流组件

3. 架构设计
   - 配置中心
   - 限流规则引擎
   - 限流执行器
   - 监控告警
```

#### 题目2：设计分布式缓存系统
```
需求：
- 支持分布式缓存
- 支持缓存过期
- 支持缓存更新
- 支持缓存统计

方案：
1. 缓存架构
   - 本地缓存（Caffeine）
   - 分布式缓存（Redis）
   - 多级缓存

2. 缓存策略
   - Cache Aside
   - Read Through
   - Write Through
   - Write Behind

3. 问题解决
   - 缓存穿透：布隆过滤器
   - 缓存击穿：互斥锁
   - 缓存雪崩：随机过期时间
```

---

### 4. 常见面试题
**复习日期：____-____-____**

#### 4.1 Java基础
1. HashMap的实现原理？
2. ConcurrentHashMap的实现原理？
3. 线程池的参数及工作流程？
4. synchronized和ReentrantLock的区别？
5. volatile的作用及原理？
6. CAS的原理及ABA问题？
7. 类加载过程及双亲委派模型？
8. JVM内存模型及GC算法？
9. OOM排查思路？
10. CPU飙高排查思路？

#### 4.2 Spring框架
1. Spring IOC的实现原理？
2. Spring AOP的实现原理？
3. Spring Bean的生命周期？
4. Spring事务的传播特性？
5. Spring循环依赖的解决？
6. Spring Boot自动配置原理？

#### 4.3 数据库
1. MySQL索引的实现原理？
2. 聚簇索引和非聚簇索引的区别？
3. 事务隔离级别及MVCC？
4. MySQL锁机制？
5. 分库分表方案？
6. 慢查询优化思路？

#### 4.4 Redis
1. Redis的数据结构及应用场景？
2. Redis的持久化机制？
3. Redis的主从复制原理？
4. Redis的集群方案？
5. 缓存穿透、击穿、雪崩的解决方案？
6. 如何实现分布式锁？

#### 4.5 分布式系统
1. CAP理论和BASE理论？
2. 分布式事务的解决方案？
3. 如何实现分布式锁？
4. 如何生成分布式ID？
5. 分布式一致性算法？

---

### 5. 模拟面试
**完成日期：____-____-____**

#### 面试记录1
```
面试公司：
面试岗位：
面试时间：
面试官：

面试题目：
1.
2.
3.

回答情况：
- 好的方面：
- 需要改进：

改进计划：
```

---

## 学习资源

### 网站
- [牛客网](https://www.nowcoder.com/)
- [LeetCode](https://leetcode.cn/)

### 书籍
- 《剑指Offer》
- 《编程珠玑》

---

## 学习总结

### 核心收获

### 待深入内容
