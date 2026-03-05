# Spring Cloud 微服务架构学习笔记

## 学习目标
- [ ] Spring Cloud Alibaba全家桶
- [ ] 服务注册与发现（Nacos、Eureka）
- [ ] 配置中心（Nacos、Apollo）
- [ ] API网关（Gateway、Zuul）
- [ ] 服务调用（OpenFeign、Dubbo）
- [ ] 熔断降级（Sentinel、Hystrix）
- [ ] 分布式事务（Seata）

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. 服务注册与发现
**学习日期：____-____-____**

#### 1.1 Nacos注册中心
```java
// 服务端配置（Docker）
docker run -d \
  -e MODE=standalone \
  -p 8848:8848 \
  --name nacos \
  nacos/nacos-server:latest

// 客户端依赖
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>

// 客户端配置
spring:
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848
        namespace: public
        group: DEFAULT_GROUP

// 服务注册
@EnableDiscoveryClient
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

#### 1.2 Nacos核心概念
```yaml
# Namespace（命名空间）
# - 环境隔离（dev、test、prod）

# Group（分组）
# - 业务隔离

# Cluster（集群）
# - 同一服务不同实例的集群
# - 就近访问

# Instance（实例）
# - 具体的服务实例
# - 包含IP、Port、权重等
```

#### 1.3 服务健康检查
```yaml
spring:
  cloud:
    nacos:
      discovery:
        heart-beat-interval: 5000  # 心跳间隔
        heart-beat-timeout: 15000  # 心跳超时
        ip-delete-timeout: 30000   # IP删除超时
```

---

### 2. 配置中心
**学习日期：____-____-____**

#### 2.1 Nacos配置中心
```java
// 依赖
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
</dependency>

// bootstrap.yml
spring:
  application:
    name: user-service
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        file-extension: yaml
        namespace: public
        group: DEFAULT_GROUP
        shared-configs:
          - data-id: common.yaml
            group: DEFAULT_GROUP
            refresh: true

// 动态刷新
@RefreshScope
@RestController
public class ConfigController {
    @Value("${config.value}")
    private String value;
}
```

#### 2.2 配置管理
```yaml
# Data ID（数据ID）
# - 格式：${spring.application.name}-${profile}.${file-extension}
# - 示例：user-service-dev.yaml

# 配置格式
# - YAML
# - Properties
# - JSON
# - XML

# 配置版本
# - 支持版本回滚
# - 灰度发布
```

---

### 3. API网关
**学习日期：____-____-____**

#### 3.1 Spring Cloud Gateway
```java
// 依赖
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>

// 配置
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service
          predicates:
            - Path=/api/user/**
          filters:
            - StripPrefix=1
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/order/**

// 自定义过滤器
@Component
public class AuthFilter implements GlobalFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange,
                             GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders()
                                 .getFirst("Authorization");
        if (token == null) {
            exchange.getResponse().setStatusCode(
                HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }
}
```

#### 3.2 Gateway核心概念
```yaml
# Route（路由）
# - ID：唯一标识
# - URI：目标地址
# - Predicate：断言
# - Filter：过滤器

# Predicate（断言）
# - Path：路径匹配
# - Method：方法匹配
# - Header：请求头匹配
# - Query：参数匹配
# - Time：时间匹配

# Filter（过滤器）
# - Pre Filter：前置处理
# - Post Filter：后置处理
```

---

### 4. 服务调用
**学习日期：____-____-____**

#### 4.1 OpenFeign
```java
// 依赖
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>

// 启用Feign
@EnableFeignClients
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

// Feign客户端
@FeignClient(name = "user-service", fallback = UserFallback.class)
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);

    @PostMapping("/api/users")
    User createUser(@RequestBody User user);
}

// 降级处理
@Component
public class UserFallback implements UserClient {
    @Override
    public User getUser(Long id) {
        return User.defaultUser();
    }

    @Override
    public User createUser(User user) {
        throw new ServiceException("服务不可用");
    }
}
```

#### 4.2 Feign配置
```yaml
feign:
  client:
    config:
      default:
        connectTimeout: 5000
        readTimeout: 5000
      user-service:
        connectTimeout: 3000
        readTimeout: 3000
  compression:
    request:
      enabled: true
      mime-types: text/xml,application/xml,application/json
      min-request-size: 2048
    response:
      enabled: true
```

---

### 5. 熔断降级
**学习日期：____-____-____**

#### 5.1 Sentinel
```java
// 依赖
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>

// 配置
spring:
  cloud:
    sentinel:
      transport:
        dashboard: localhost:8080
        port: 8719

// 流量控制
@GetMapping("/api/users")
@SentinelResource(value = "getUsers",
    blockHandler = "handleBlock",
    fallback = "handleFallback")
public List<User> getUsers() {
    return userService.list();
}

// 限流处理
public List<User> handleBlock(BlockException ex) {
    return Collections.emptyList();
}

// 降级处理
public List<User> handleFallback(Throwable ex) {
    return Collections.emptyList();
}
```

#### 5.2 Sentinel规则
```yaml
# 流量控制规则
# - QPS：每秒请求数
# - 线程数：并发线程数
# - 流控模式：直接、关联、链路
# - 流控效果：快速失败、Warm Up、排队等待

# 熔断降级规则
# - 慀调用比例
# - 异常比例
# - 异常数
# - 熔断时长
# - 最小请求数

# 热点参数规则
# - 参数索引
# - 单机阈值
# - 参数例外项
```

---

### 6. 分布式事务
**学习日期：____-____-____**

#### 6.1 Seata AT模式
```java
// 依赖
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-seata</artifactId>
</dependency>

// 配置
seata:
  enabled: true
  application-id: order-service
  tx-service-group: my_test_tx_group
  service:
    vgroup-mapping:
      my_test_tx_group: default
  registry:
    type: nacos
    nacos:
      server-addr: localhost:8848
      namespace: public
      group: SEATA_GROUP

// 全局事务
@GlobalTransactional
public void createOrder(Order order) {
    // 扣减库存
    stockService.deduct(order.getProductId(), order.getCount());
    // 创建订单
    orderMapper.insert(order);
    // 扣减余额
    accountService.deduct(order.getUserId(), order.getAmount());
}
```

#### 6.2 Seata模式对比
```
AT模式：
- 优点：代码侵入小，两阶段提交
- 缺点：锁竞争，性能一般
- 场景：常规业务

TCC模式：
- 优点：性能好，无锁
- 缺点：代码侵入大，要实现三个接口
- 场景：高性能要求

Saga模式：
- 优点：长事务支持
- 缺点：状态机复杂
- 场景：业务流程长
```

---

### 7. 链路追踪
**学习日期：____-____-____**

#### 7.1 Spring Cloud Sleuth + Zipkin
```java
// 依赖
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-sleuth-zipkin</artifactId>
</dependency>

// 配置
spring:
  sleuth:
    zipkin:
      base-url: http://localhost:9411
    sampler:
      probability: 1.0  # 采样率
```

---

## 实战项目

### 项目：电商微服务系统
**项目地址**：`projects/ecommerce-microservice/`

#### 服务列表
- [ ] user-service（用户服务）
- [ ] product-service（商品服务）
- [ ] order-service（订单服务）
- [ ] payment-service（支付服务）
- [ ] gateway-service（网关服务）

#### 技术栈
- Spring Cloud Alibaba
- Nacos
- Sentinel
- Seata
- Gateway
- OpenFeign

---

## 学习资源

### 视频
- 黑马程序员Spring Cloud Alibaba

### 官方文档
- [Spring Cloud Alibaba](https://github.com/alibaba/spring-cloud-alibaba)
- [Nacos](https://nacos.io/zh-cn/)
- [Sentinel](https://sentinelguard.io/zh-cn/)

---

## 学习总结

### 核心收获

### 待深入内容
