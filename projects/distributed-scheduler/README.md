# 分布式任务调度系统

## 项目概述
自研分布式任务调度系统，支持定时任务调度、分布式执行、故障转移、任务监控等功能。

## 技术栈
- Spring Boot 2.7
- Netty 4.1
- MySQL 8
- Redis 7
- Zookeeper 3.8（可选）
- Vue 3（管理平台）

## 项目进度
**状态**：⬜ 未开始
**完成度**：0%

## 系统架构

### 架构图
```
┌─────────────┐
│  管理平台    │  Vue3 + Element Plus
└──────┬──────┘
       │ HTTP
┌──────▼──────┐
│  调度服务器  │  Spring Boot + Netty
│  (Master)   │  - 任务调度
│             │  - 任务管理
└──────┬──────┘  - 监控告警
       │ TCP
┌──────▼──────┐
│  执行器集群  │  Spring Boot + Netty
│ (Workers)   │  - 任务执行
│  [Node1]    │  - 心跳上报
│  [Node2]    │  - 故障转移
│  [Node3]    │
└──────┬──────┘
       │
┌──────▼──────┐
│  MySQL      │  - 任务信息
│  Redis      │  - 分布式锁
│  Zookeeper  │  - 选举协调
└─────────────┘
```

---

## 核心模块

### 1. 调度服务器 (scheduler-server)
**端口**：9000
**功能**：
- [ ] 任务管理（CRUD）
- [ ] 任务调度（CRON表达式）
- [ ] 任务分发
- [ ] 监控告警
- [ ] 日志管理

**状态**：⬜ 未开始

---

### 2. 执行器 (executor)
**端口**：9001+
**功能**：
- [ ] 任务执行
- [ ] 心跳上报
- [ ] 结果反馈
- [ ] 故障转移
- [ ] 任务分片

**状态**：⬜ 未开始

---

### 3. 管理平台 (admin-web)
**端口**：8080
**功能**：
- [ ] 任务管理界面
- [ ] 执行器管理
- [ ] 监控大屏
- [ ] 日志查询

**状态**：⬜ 未开始

---

## 核心技术

### 1. 任务调度
```java
// CRON表达式解析
public class CronExpression {
    public boolean matches(Date date) {
        // 解析CRON表达式
        // 判断是否匹配
    }
}

// 任务调度器
public class TaskScheduler {
    public void schedule(Task task) {
        // 计算下次执行时间
        // 添加到时间轮
    }

    public void trigger() {
        // 触发到期任务
        // 分配给执行器
    }
}
```

### 2. 任务分发
```java
// 分发策略
public interface DispatchStrategy {
    Executor select(List<Executor> executors, Task task);
}

// 轮询分发
public class RoundRobinStrategy implements DispatchStrategy {
    private AtomicInteger index = new AtomicInteger(0);

    @Override
    public Executor select(List<Executor> executors, Task task) {
        int i = index.getAndIncrement() % executors.size();
        return executors.get(i);
    }
}

// 负载最低
public class LeastLoadStrategy implements DispatchStrategy {
    @Override
    public Executor select(List<Executor> executors, Task task) {
        return executors.stream()
            .min(Comparator.comparing(Executor::getLoad))
            .orElse(null);
    }
}
```

### 3. 任务执行
```java
// 执行器注册
public class ExecutorRegistry {
    private Map<String, Executor> executors = new ConcurrentHashMap<>();

    public void register(Executor executor) {
        executors.put(executor.getId(), executor);
    }

    public void unregister(String executorId) {
        executors.remove(executorId);
    }
}

// 任务执行
public class TaskExecutor {
    public void execute(Task task) {
        try {
            // 执行任务
            Object result = task.getAction().run();
            // 反馈结果
            feedback(task.getId(), Result.success(result));
        } catch (Exception e) {
            feedback(task.getId(), Result.failure(e));
        }
    }
}
```

### 4. 故障转移
```java
// 心跳检测
public class HealthChecker {
    public void check() {
        executors.forEach(executor -> {
            if (executor.isTimeout()) {
                // 标记为不可用
                executor.markUnavailable();
                // 重新分配任务
                reschedule(executor.getTasks());
            }
        });
    }
}

// 任务重试
public class TaskRetry {
    public void retry(Task task) {
        if (task.getRetryCount() < task.getMaxRetry()) {
            // 重新分配
            dispatch(task);
        } else {
            // 标记失败
            task.markFailed();
        }
    }
}
```

### 5. 分布式锁
```java
// Redis分布式锁
public class DistributedLock {
    public boolean lock(String key, int expireTime) {
        String value = UUID.randomUUID().toString();
        Boolean locked = redis.setnx(key, value, expireTime);
        return locked;
    }

    public void unlock(String key) {
        // Lua脚本保证原子性
        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then " +
                        "    return redis.call('del', KEYS[1]) " +
                        "else " +
                        "    return 0 " +
                        "end";
        redis.eval(script, key, value);
    }
}
```

### 6. 任务分片
```java
// 分片任务
public class ShardingTask {
    public void execute(Task task) {
        // 获取可用执行器
        List<Executor> executors = getAvailableExecutors();
        // 计算分片
        int shardCount = executors.size();
        List<Shard> shards = shard(task.getData(), shardCount);
        // 分配分片
        for (int i = 0; i < shards.size(); i++) {
            Executor executor = executors.get(i);
            executor.execute(new ShardTask(task, shards.get(i), i));
        }
    }
}
```

---

## 数据库设计

### 任务表 (task)
```sql
CREATE TABLE `task` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `name` VARCHAR(100) NOT NULL COMMENT '任务名称',
  `cron` VARCHAR(50) NOT NULL COMMENT 'CRON表达式',
  `action_class` VARCHAR(200) COMMENT '任务执行类',
  `params` TEXT COMMENT '任务参数',
  `status` TINYINT DEFAULT 0 COMMENT '状态 0-正常 1-暂停',
  `next_time` DATETIME COMMENT '下次执行时间',
  `last_time` DATETIME COMMENT '上次执行时间',
  `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP,
  `update_time` DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 执行器表 (executor)
```sql
CREATE TABLE `executor` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `name` VARCHAR(100) NOT NULL COMMENT '执行器名称',
  `host` VARCHAR(50) NOT NULL COMMENT '主机',
  `port` INT NOT NULL COMMENT '端口',
  `status` TINYINT DEFAULT 0 COMMENT '状态 0-在线 1-离线',
  `load` INT DEFAULT 0 COMMENT '负载数',
  `heartbeat` DATETIME COMMENT '最后心跳时间',
  `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

### 任务日志表 (task_log)
```sql
CREATE TABLE `task_log` (
  `id` BIGINT PRIMARY KEY AUTO_INCREMENT,
  `task_id` BIGINT NOT NULL COMMENT '任务ID',
  `executor_id` BIGINT COMMENT '执行器ID',
  `status` TINYINT COMMENT '状态 0-成功 1-失败',
  `start_time` DATETIME COMMENT '开始时间',
  `end_time` DATETIME COMMENT '结束时间',
  `cost` INT COMMENT '耗时(ms)',
  `result` TEXT COMMENT '执行结果',
  `exception` TEXT COMMENT '异常信息',
  `create_time` DATETIME DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## API接口

### 任务管理
```java
POST   /api/tasks           # 创建任务
GET    /api/tasks           # 任务列表
GET    /api/tasks/{id}      # 任务详情
PUT    /api/tasks/{id}      # 更新任务
DELETE /api/tasks/{id}      # 删除任务
POST   /api/tasks/{id}/pause  # 暂停任务
POST   /api/tasks/{id}/start  # 启动任务
POST   /api/tasks/{id}/trigger # 立即执行
```

### 执行器管理
```java
GET   /api/executors        # 执行器列表
GET   /api/executors/{id}   # 执行器详情
POST  /api/executors/{id}/disable # 禁用执行器
POST  /api/executors/{id}/enable  # 启用执行器
```

---

## 项目结构
```
distributed-scheduler/
├── scheduler-server/       # 调度服务器
│   ├── task/              # 任务管理
│   ├── schedule/          # 任务调度
│   ├── dispatch/          # 任务分发
│   └── monitor/           # 监控告警
├── executor/              # 执行器
│   ├── execute/           # 任务执行
│   ├── heartbeat/         # 心跳上报
│   └── failover/          # 故障转移
├── common/                # 公共模块
│   ├── protocol/          # 通信协议
│   └── lock/              # 分布式锁
├── admin-web/             # 管理平台
│   ├── task/              # 任务管理
│   ├── executor/          # 执行器管理
│   └── monitor/           # 监控大屏
└── sql/                   # SQL脚本
```

---

## 快速开始

### 启动调度服务器
```bash
cd scheduler-server
mvn spring-boot:run
```

### 启动执行器
```bash
cd executor
mvn spring-boot:run
```

### 启动管理平台
```bash
cd admin-web
npm install
npm run dev
```

---

## 后续计划
- [ ] 第1周：项目搭建、协议定义
- [ ] 第2-3周：调度服务器开发
- [ ] 第4-5周：执行器开发
- [ ] 第6周：管理平台开发
- [ ] 第7周：测试优化
- [ ] 第8周：文档完善

---

## 学习资源
- XXL-Job源码
- Elastic-Job源码
- 《分布式系统原理与范型》
