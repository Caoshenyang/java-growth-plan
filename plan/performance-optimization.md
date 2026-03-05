# 性能优化实战笔记

## 学习目标
- [ ] 全链路性能优化（网络、应用、数据库、系统）
- [ ] 性能测试（JMeter、K6、wrk）
- [ ] 性能指标（QPS、TPS、响应时间、并发数）
- [ ] 性能瓶颈定位

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. 性能优化方法论
**学习日期：____-____-____**

#### 1.1 性能优化流程
```
1. 确定性能目标
   - QPS目标
   - 响应时间目标
   - 并发数目标

2. 建立性能基准
   - 压测环境准备
   - 基准数据收集
   - 性能指标定义

3. 定位性能瓶颈
   - 网络层分析
   - 应用层分析
   - 数据库层分析
   - 系统层分析

4. 优化实施
   - 制定优化方案
   - 逐步实施
   - 验证效果

5. 持续监控
   - 性能监控
   - 告警配置
   - 优化迭代
```

#### 1.2 性能优化原则
```
1. 不要过早优化
2. 优化热点代码（80/20原则）
3. 先测量后优化
4. 优先优化瓶颈
5. 优化后要验证
```

---

### 2. 网络优化
**学习日期：____-____-____**

#### 2.1 连接池优化
```java
// HTTP连接池
@Configuration
public class HttpClientConfig {
    @Bean
    public RestTemplate restTemplate() {
        HttpComponentsClientHttpRequestFactory factory =
            new HttpComponentsClientHttpRequestFactory();
        factory.setConnectTimeout(5000);        // 连接超时
        factory.setReadTimeout(10000);          // 读取超时
        factory.setConnectionRequestTimeout(5000); // 获取连接超时

        // 连接池配置
        HttpClient httpClient = HttpClientBuilder.create()
            .setMaxConnTotal(200)               // 最大连接数
            .setMaxConnPerRoute(50)             // 每个路由最大连接数
            .evictIdleConnections(30, TimeUnit.SECONDS) // 空闲连接驱逐
            .build();

        factory.setHttpClient(httpClient);
        return new RestTemplate(factory);
    }
}

// 数据库连接池
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/db");
        dataSource.setUsername("root");
        dataSource.setPassword("root");

        // 连接池配置
        dataSource.setMaximumPoolSize(20);       // 最大连接数
        dataSource.setMinimumIdle(5);            // 最小空闲连接
        dataSource.setConnectionTimeout(30000);  // 连接超时
        dataSource.setIdleTimeout(600000);       // 空闲超时
        dataSource.setMaxLifetime(1800000);      // 连接最大生命周期

        // 性能优化
        dataSource.setAutoCommit(false);         // 关闭自动提交
        dataSource.setReadOnly(false);           // 非只读
        dataSource.setConnectionTestQuery("SELECT 1"); // 连接测试

        return dataSource;
    }
}

// Redis连接池
@Configuration
public class RedisConfig {
    @Bean
    public JedisPool jedisPool() {
        JedisPoolConfig config = new JedisPoolConfig();
        config.setMaxTotal(50);                  // 最大连接数
        config.setMaxIdle(10);                   // 最大空闲连接
        config.setMinIdle(5);                    // 最小空闲连接
        config.setMaxWaitMillis(3000);           // 获取连接超时
        config.setTestOnBorrow(true);            // 获取时测试
        config.setTestOnReturn(false);           // 归还时不测试

        return new JedisPool(config, "localhost", 6379);
    }
}
```

#### 2.2 HTTP/2优化
```yaml
# Nginx配置
server {
    listen 443 ssl http2;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # HTTP/2优化
    http2_push_preload on;         # 服务器推送
    http2_max_concurrent_streams 100;
    http2_streams_index_size 10;
    http2_recv_timeout 30000;

    # Gzip压缩
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript;
}
```

#### 2.3 长连接优化
```java
// WebSocket长连接
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {
    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(new WebSocketHandler(), "/ws")
            .setAllowedOrigins("*")
            .withSockJS();  // 支持SockJS降级
    }
}
```

---

### 3. 应用优化
**学习日期：____-____-____**

#### 3.1 池化技术
```java
// 线程池配置
@Configuration
public class ThreadPoolConfig {
    @Bean("taskExecutor")
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();

        // CPU密集型：线程数 = CPU核心数 + 1
        // IO密集型：线程数 = CPU核心数 / (1 - 阻塞系数)
        int cpuCore = Runtime.getRuntime().availableProcessors();
        executor.setCorePoolSize(cpuCore);
        executor.setMaxPoolSize(cpuCore * 2);
        executor.setQueueCapacity(1000);
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("task-");
        executor.setRejectedExecutionHandler(
            new ThreadPoolExecutor.CallerRunsPolicy());

        // 优雅关闭
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(60);

        executor.initialize();
        return executor;
    }
}

// 对象池
public class ObjectPool<T> {
    private final Queue<T> pool = new LinkedList<>();
    private final Supplier<T> supplier;
    private final int maxSize;

    public T borrow() {
        T obj = pool.poll();
        if (obj == null) {
            obj = supplier.get();
        }
        return obj;
    }

    public void returnObject(T obj) {
        if (pool.size() < maxSize) {
            pool.offer(obj);
        }
    }
}
```

#### 3.2 异步处理
```java
// 异步方法
@Service
public class AsyncService {
    @Async("taskExecutor")
    public CompletableFuture<String> asyncMethod() {
        // 异步执行
        return CompletableFuture.completedFuture("result");
    }

    @Async("taskExecutor")
    public void asyncMethod2() {
        // 异步执行，无返回值
    }
}

// 异步调用
@RestController
public class AsyncController {
    @Autowired
    private AsyncService asyncService;

    @GetMapping("/async")
    public CompletableFuture<String> async() {
        return asyncService.asyncMethod();
    }

    @GetMapping("/async2")
    public String async2() {
        asyncService.asyncMethod2();
        return "submitted";
    }
}

// 并行执行
public CompletableFuture<Result> parallel() {
    CompletableFuture<String> f1 = asyncService.method1();
    CompletableFuture<Integer> f2 = asyncService.method2();

    return f1.thenCombine(f2, (r1, r2) ->
        new Result(r1, r2));
}
```

#### 3.3 缓存策略
```java
// 本地缓存
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager cacheManager() {
        CaffeineCacheManager cacheManager = new CaffeineCacheManager();
        cacheManager.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(30, TimeUnit.MINUTES)
            .recordStats());  // 记录统计信息
        return cacheManager;
    }
}

// 使用缓存
@Service
public class UserService {
    @Cacheable(value = "user", key = "#id")
    public User getUserById(Long id) {
        return userMapper.selectById(id);
    }

    @CacheEvict(value = "user", key = "#user.id")
    public void updateUser(User user) {
        userMapper.updateById(user);
    }

    @CachePut(value = "user", key = "#result.id")
    public User saveUser(User user) {
        userMapper.insert(user);
        return user;
    }
}

// 多级缓存
@Service
public class MultiLevelCacheService {
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    private final Cache localCache = Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(5, TimeUnit.MINUTES)
        .build();

    public Object get(String key) {
        // L1：本地缓存
        Object value = localCache.getIfPresent(key);
        if (value != null) {
            return value;
        }

        // L2：Redis缓存
        value = redisTemplate.opsForValue().get(key);
        if (value != null) {
            localCache.put(key, value);
            return value;
        }

        // L3：数据库
        value = loadFromDB(key);
        if (value != null) {
            localCache.put(key, value);
            redisTemplate.opsForValue().set(key, value, 30, TimeUnit.MINUTES);
        }

        return value;
    }
}
```

---

### 4. 数据库优化
**学习日期：____-____-____**

#### 4.1 SQL优化
```sql
-- 1. 避免SELECT *
-- 不推荐
SELECT * FROM user WHERE id = 1;

-- 推荐
SELECT id, name FROM user WHERE id = 1;

-- 2. 使用索引
-- 不推荐
SELECT * FROM user WHERE YEAR(create_time) = 2024;

-- 推荐
SELECT * FROM user WHERE create_time >= '2024-01-01'
  AND create_time < '2025-01-01';

-- 3. 避免隐式转换
-- 不推荐
SELECT * FROM user WHERE phone = 13800138000;  -- phone是VARCHAR

-- 推荐
SELECT * FROM user WHERE phone = '13800138000';

-- 4. 批量操作
-- 不推荐
INSERT INTO user (name) VALUES ('a');
INSERT INTO user (name) VALUES ('b');
INSERT INTO user (name) VALUES ('c');

-- 推荐
INSERT INTO user (name) VALUES ('a'), ('b'), ('c');

-- 5. 分页优化
-- 不推荐（深分页性能差）
SELECT * FROM user ORDER BY id LIMIT 1000000, 10;

-- 推荐（使用游标）
SELECT * FROM user WHERE id > 1000000 ORDER BY id LIMIT 10;
```

#### 4.2 索引优化
```sql
-- 联合索引优化
-- 索引：idx(name, age, status)
SELECT * FROM user WHERE name = '张三' AND age = 20;  -- 使用索引
SELECT * FROM user WHERE age = 20;                     -- 不使用索引

-- 覆盖索引
SELECT id, name FROM user WHERE name = '张三';  -- 覆盖索引，不回表

-- 索引下推
SELECT * FROM user WHERE name LIKE '张%' AND age > 20;  -- 索引下推
```

#### 4.3 表设计优化
```sql
-- 垂直分表
-- 原表
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    intro TEXT,  -- 大字段
    create_time DATETIME
);

-- 分表
CREATE TABLE user (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50),
    age INT,
    create_time DATETIME
);

CREATE TABLE user_ext (
    user_id BIGINT PRIMARY KEY,
    intro TEXT
);

-- 水平分表
CREATE TABLE user_0 (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50)
);
CREATE TABLE user_1 (
    id BIGINT PRIMARY KEY,
    name VARCHAR(50)
);
```

---

### 5. 系统优化
**学习日期：____-____-____**

#### 5.1 内核参数优化
```bash
# /etc/sysctl.conf

# 网络优化
net.core.somaxconn = 32768                    # 监听队列长度
net.ipv4.tcp_max_syn_backlog = 8192          # SYN队列长度
net.core.netdev_max_backlog = 16384          # 网卡队列长度

# TCP优化
net.ipv4.tcp_fin_timeout = 30                # FIN等待时间
net.ipv4.tcp_keepalive_time = 1200           # Keepalive时间
net.ipv4.tcp_max_tw_buckets = 5000           # TIME_WAIT数量
net.ipv4.tcp_tw_reuse = 1                    # 复用TIME_WAIT
net.ipv4.tcp_tw_recycle = 0                  # 关闭TIME_WAIT快速回收

# 文件句柄
fs.file-max = 65535

# 应用
# /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
```

#### 5.2 JVM参数优化
```bash
# 堆内存
-Xms4g -Xmx4g                    # 初始堆和最大堆相同

# 新生代
-Xmn2g                          # 新生代大小
-XX:SurvivorRatio=8             # Eden:S0:S1 = 8:1:1

# 元空间
-XX:MetaspaceSize=256m
-XX:MaxMetaspaceSize=256m

# GC选择
-XX:+UseG1GC                    # 使用G1
-XX:MaxGCPauseMillis=200        # 最大停顿时间

# GC日志
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:/path/to/gc.log

# 性能优化
-XX:+UseTLAB                    # 线程本地分配
-XX:+UseBiasedLocking           # 偏向锁
-XX:+UseStringDeduplication     # 字符串去重
```

---

### 6. 性能测试
**学习日期：____-____-____**

#### 6.1 JMeter测试
```xml
<!-- JMeter测试计划 -->
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2">
  <hashTree>
    <TestPlan>
      <stringProp name="TestPlan.comments">性能测试</stringProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
    </TestPlan>
    <hashTree>
      <!-- 线程组 -->
      <ThreadGroup>
        <stringProp name="ThreadGroup.num_threads">100</stringProp>
        <stringProp name="ThreadGroup.ramp_time">10</stringProp>
        <stringProp name="ThreadGroup.duration">60</stringProp>
      </ThreadGroup>
      <hashTree>
        <!-- HTTP请求 -->
        <HTTPSamplerProxy>
          <stringProp name="HTTPSampler.domain">example.com</stringProp>
          <stringProp name="HTTPSampler.path">/api/users</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
        </HTTPSamplerProxy>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

#### 6.2 K6测试
```javascript
// k6测试脚本
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  stages: [
    { duration: '1m', target: 100 },   // 1分钟到100用户
    { duration: '3m', target: 100 },   // 保持100用户
    { duration: '1m', target: 200 },   // 1分钟到200用户
    { duration: '3m', target: 200 },   // 保持200用户
    { duration: '1m', target: 0 },     // 降低到0
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],  // 95%请求在500ms内
    http_req_failed: ['rate<0.01'],    // 错误率小于1%
  },
};

export default function () {
  let res = http.get('http://example.com/api/users');
  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });
  sleep(1);
}
```

#### 6.3 wrk测试
```bash
# wrk测试命令
wrk -t12 -c400 -d30s http://example.com/api/users

# 参数说明：
# -t: 线程数
# -c: 连接数
# -d: 测试时长
# -s: Lua脚本
# --latency: 输出延迟统计

# 使用Lua脚本
wrk -t12 -c400 -d30s -s script.lua http://example.com
```

---

## 实战项目

### 项目：性能优化实战
**项目地址**：`projects/performance-optimization/`

#### 优化目标
- [ ] QPS从1K提升到10K
- [ ] 响应时间P99从500ms降低到100ms
- [ ] 数据库连接池优化
- [ ] 缓存优化
- [ ] SQL优化

---

## 学习资源

### 书籍
- 《Java性能权威指南》
- 《性能之巅》

### 工具
- JProfiler
- Arthas
- SkyWalking

---

## 学习总结

### 核心收获

### 待深入内容
