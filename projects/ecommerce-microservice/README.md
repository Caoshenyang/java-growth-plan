# 电商平台微服务系统

## 项目概述
基于Spring Cloud Alibaba的电商微服务系统，实现用户、商品、订单、支付等核心功能，支持容器化部署和K8s编排。

## 技术栈
- Spring Cloud Alibaba 2021.0.5.0
- Spring Boot 2.7
- Nacos 2（注册中心、配置中心）
- Sentinel 1.8（限流熔断）
- Seata 1.5（分布式事务）
- Gateway 3.1（API网关）
- OpenFeign（服务调用）
- Redis 7（缓存）
- RocketMQ 5（消息队列）
- MySQL 8（数据库）
- Docker（容器化）
- Kubernetes（编排）

## 项目进度
**状态**：⬜ 未开始
**完成度**：0%

## 微服务列表

### 1. 用户服务 (user-service)
**端口**：8001
**功能**：
- [ ] 用户注册
- [ ] 用户登录（JWT）
- [ ] 用户信息管理
- [ ] 收货地址管理

**接口**：
- POST /api/users/register - 注册
- POST /api/users/login - 登录
- GET /api/users/{id} - 获取用户信息
- PUT /api/users/{id} - 更新用户信息

**状态**：⬜ 未开始

---

### 2. 商品服务 (product-service)
**端口**：8002
**功能**：
- [ ] 商品管理
- [ ] 分类管理
- [ ] 库存管理
- [ ] 商品搜索（ES）

**接口**：
- GET /api/products - 商品列表
- GET /api/products/{id} - 商品详情
- POST /api/products - 添加商品
- PUT /api/products/{id} - 更新商品
- DELETE /api/products/{id} - 删除商品

**状态**：⬜ 未开始

---

### 3. 订单服务 (order-service)
**端口**：8003
**功能**：
- [ ] 订单创建
- [ ] 订单查询
- [ ] 订单状态管理
- [ ] 订单支付
- [ ] 分布式事务

**接口**：
- POST /api/orders - 创建订单
- GET /api/orders/{id} - 订单详情
- GET /api/orders - 订单列表
- PUT /api/orders/{id}/status - 更新状态

**状态**：⬜ 未开始

---

### 4. 支付服务 (payment-service)
**端口**：8004
**功能**：
- [ ] 支付创建
- [ ] 支付回调
- [ ] 支付查询
- [ ] 退款处理

**接口**：
- POST /api/payments - 创建支付
- POST /api/payments/callback - 支付回调
- GET /api/payments/{id} - 支付查询

**状态**：⬜ 未开始

---

### 5. 网关服务 (gateway-service)
**端口**：8000
**功能**：
- [ ] 路由转发
- [ ] 鉴权（JWT）
- [ ] 限流（Sentinel）
- [ ] 日志记录
- [ ] 跨域处理

**状态**：⬜ 未开始

---

## 核心技术

### 1. 服务治理
```yaml
# 服务注册与发现
spring:
  cloud:
    nacos:
      discovery:
        server-addr: localhost:8848

# 服务调用
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUser(@PathVariable("id") Long id);
}

# 限流熔断
@SentinelResource(value = "getUser",
    blockHandler = "handleBlock")
public User getUser(Long id) {
    return userClient.getUser(id);
}
```

### 2. 配置中心
```yaml
# bootstrap.yml
spring:
  cloud:
    nacos:
      config:
        server-addr: localhost:8848
        file-extension: yaml
        shared-configs:
          - data-id: common.yaml
```

### 3. 分布式事务
```java
@GlobalTransactional
public void createOrder(Order order) {
    // 扣减库存
    productClient.deductStock(order.getProductId(), order.getCount());
    // 创建订单
    orderMapper.insert(order);
    // 扣减余额
    userClient.deductBalance(order.getUserId(), order.getAmount());
}
```

### 4. 容器化部署
```dockerfile
# Dockerfile
FROM openjdk:11-jre-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 5. K8s编排
```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
    spec:
      containers:
      - name: user-service
        image: user-service:latest
        ports:
        - containerPort: 8001
```

---

## 项目结构
```
ecommerce-microservice/
├── user-service/         # 用户服务
├── product-service/      # 商品服务
├── order-service/        # 订单服务
├── payment-service/      # 支付服务
├── gateway-service/      # 网关服务
├── common/               # 公共模块
│   ├── common-core/      # 核心工具类
│   ├── common-web/       # Web配置
│   ├── common-auth/      # 认证模块
│   └── common-feign/     # Feign配置
├── docker/               # Docker配置
│   ├── docker-compose.yml
│   └── Dockerfile
├── k8s/                  # K8s配置
│   ├── deployments/
│   ├── services/
│   └── ingress/
└── docs/                 # 文档
    ├── design.md
    ├── api.md
    └── deploy.md
```

---

## 快速开始

### 本地开发
```bash
# 1. 启动Nacos
sh nacos/bin/startup.sh -m standalone

# 2. 启动Sentinel Dashboard
java -jar sentinel-dashboard.jar

# 3. 启动基础服务（MySQL、Redis、RocketMQ）
docker-compose -f docker/docker-compose-dev.yml up -d

# 4. 启动微服务
# 按顺序启动：user -> product -> order -> payment -> gateway

# 5. 访问网关
http://localhost:8000
```

### Docker部署
```bash
# 构建镜像
docker build -t user-service:latest ./user-service

# 启动服务
docker-compose -f docker/docker-compose.yml up -d
```

### K8s部署
```bash
# 创建命名空间
kubectl create namespace ecommerce

# 部署服务
kubectl apply -f k8s/deployments/
kubectl apply -f k8s/services/
kubectl apply -f k8s/ingress/

# 查看状态
kubectl get pods -n ecommerce
```

---

## 监控告警

### 监控指标
- QPS、响应时间
- JVM内存、GC
- 数据库连接池
- Redis连接数
- RocketMQ消息堆积

### 告警规则
- QPS异常波动
- 响应时间P99 > 500ms
- 错误率 > 1%
- JVM内存使用 > 80%

---

## 后续计划
- [ ] 第1周：项目搭建、公共模块
- [ ] 第2周：用户服务
- [ ] 第3周：商品服务
- [ ] 第4周：订单服务
- [ ] 第5周：支付服务
- [ ] 第6周：网关服务
- [ ] 第7周：服务治理、分布式事务
- [ ] 第8周：容器化、K8s部署

---

## 学习资源
- 黑马程序员Spring Cloud Alibaba
- 《深入理解Spring Cloud与微服务构建》
- Spring Cloud Alibaba官方文档
