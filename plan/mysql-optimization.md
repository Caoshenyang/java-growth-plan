# MySQL 深度优化笔记

## 学习目标
- [ ] MySQL索引优化（B+树、执行计划、索引失效场景）
- [ ] 事务隔离级别与锁机制
- [ ] 分库分表方案（水平拆分、垂直拆分）
- [ ] 读写分离、主从复制
- [ ] 慢查询优化

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. 索引优化
**学习日期：____-____-____**

#### 1.1 索引数据结构
```sql
-- B+树特点
-- 1. 非叶子节点只存储索引
-- 2. 叶子节点存储数据
-- 3. 叶子节点通过指针连接（范围查询优势）
-- 4. 高度一般3层（1000万数据）

-- 聚簇索引 vs 非聚簇索引
-- 聚簇索引：主键索引，叶子节点存储完整行数据
-- 非聚簇索引：辅助索引，叶子节点存储主键值（需要回表）
```

#### 1.2 索引类型
```sql
-- 主键索引
PRIMARY KEY (id)

-- 唯一索引
UNIQUE KEY uk_email (email)

-- 普通索引
KEY idx_name (name)

-- 联合索引
KEY idx_name_age (name, age)

-- 全文索引
FULLTEXT KEY ft_content (content)
```

#### 1.3 索引失效场景
```sql
-- 1. 使用函数
SELECT * FROM user WHERE YEAR(create_time) = 2024;

-- 2. 隐式转换
SELECT * FROM user WHERE phone = 13800138000;  -- phone是varchar

-- 3. 模糊查询（前缀可以使用）
SELECT * FROM user WHERE name LIKE '%张%';  -- 失效
SELECT * FROM user WHERE name LIKE '张%';   -- 有效

-- 4. 联合索引不满足最左前缀
-- 索引：idx(a, b, c)
WHERE b = 1 AND c = 2;  -- 失效
WHERE a = 1 AND c = 2;  -- 部分失效（只用a）

-- 5. 范围查询后面的索引失效
-- 索引：idx(a, b, c)
WHERE a = 1 AND b > 10 AND c = 3;  -- c索引失效

-- 6. != 或 <> 操作
SELECT * FROM user WHERE status != 1;

-- 7. IS NULL 或 IS NOT NULL
SELECT * FROM user WHERE name IS NULL;
```

#### 1.4 索引设计原则
```sql
-- 1. 选择性高的字段
-- 区分度计算：
SELECT COUNT(DISTINCT column) / COUNT(*) FROM table;

-- 2. 联合索引顺序（最左前缀原则）
-- 经常查询的字段放前面
-- 区分度高的字段放前面

-- 3. 覆盖索引（避免回表）
-- 索引：idx(name, age)
SELECT id, name, age FROM user WHERE name = '张三';

-- 4. 索引列不要太多（5个以内）
-- 5. 考虑索引的维护成本
```

---

### 2. 执行计划分析
**学习日期：____-____-____**

#### 2.1 EXPLAIN 命令
```sql
EXPLAIN SELECT * FROM user WHERE id = 1;

-- 重要列：
-- id: 查询序列号
-- select_type: 查询类型（SIMPLE, PRIMARY, SUBQUERY, DERIVED等）
-- type: 访问类型（性能：system > const > eq_ref > ref > range > index > ALL）
-- key: 实际使用的索引
-- rows: 扫描行数
-- Extra: 额外信息
```

#### 2.2 type 类型详解
```
system: 表只有一行记录（系统表）
const: 主键或唯一索引查询（最多1行）
eq_ref: 唯一索引扫描（JOIN时使用）
ref: 非唯一索引扫描
range: 范围查询（>, <, BETWEEN, IN）
index: 索引全扫描
ALL: 全表扫描（需要优化）
```

#### 2.3 Extra 重要信息
```sql
-- Using index: 使用覆盖索引（好）
-- Using filesort: 文件排序（需要优化）
-- Using temporary: 使用临时表（需要优化）
-- Using where: 使用WHERE过滤
-- Using index condition: 索引下推
```

---

### 3. 事务与锁
**学习日期：____-____-____**

#### 3.1 事务隔离级别
```sql
-- 查看隔离级别
SELECT @@transaction_isolation;

-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 四种隔离级别：
-- 1. READ UNCOMMITTED（读未提交）：脏读、不可重复读、幻读
-- 2. READ COMMITTED（读已提交）：不可重复读、幻读
-- 3. REPEATABLE READ（可重复读）：幻读（MySQL默认级别）
-- 4. SERIALIZABLE（串行化）：无并发问题
```

#### 3.2 锁机制
```sql
-- 行锁（InnoDB支持）
-- 共享锁（S锁）：SELECT ... LOCK IN SHARE MODE
-- 排他锁（X锁）：SELECT ... FOR UPDATE

-- 间隙锁（Next-Key Lock）
-- 解决幻读问题
-- 锁定索引记录之间的间隙

-- 表锁（MyISAM支持）
-- 读锁：LOCK TABLE user READ;
-- 写锁：LOCK TABLE user WRITE;
```

#### 3.3 死锁处理
```sql
-- 查看死锁日志
SHOW ENGINE INNODB STATUS;

-- 死锁预防：
-- 1. 固定加锁顺序
-- 2. 尽量使用相等条件
-- 3. 添加合理的索引
-- 4. 使用较小的事务
```

---

### 4. 主从复制
**学习日期：____-____-____**

#### 4.1 复制原理
```
1. Master记录二进制日志（Binary Log）
2. Slave连接Master，请求指定位置之后的日志
3. Master执行Dump Thread，发送日志给Slave
4. Slave的I/O Thread接收日志，写入Relay Log
5. Slave的SQL Thread重放Relay Log中的操作
```

#### 4.2 复制模式
```sql
-- 异步复制（默认）
-- 主库执行完直接返回，不等待从库确认

-- 半同步复制
-- 主库等待至少一个从库确认后才返回
SET GLOBAL rpl_semi_sync_master_enabled = 1;

-- 全同步复制
-- 主库等待所有从库确认（性能差）
```

#### 4.3 主从配置
```sql
-- Master配置
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=ROW

-- Slave配置
[mysqld]
server-id=2
relay-log=relay-bin
read-only=1
```

---

### 5. 分库分表
**学习日期：____-____-____**

#### 5.1 垂直拆分
```sql
-- 垂直分表（大表拆小表）
-- 将不常用的字段、大字段拆分到新表

-- 垂直分库
-- 按业务模块拆分（用户库、订单库、商品库）
```

#### 5.2 水平拆分
```sql
-- 水平分表
-- 1. RANGE分片：按时间、ID范围
-- 2. HASH分片：取模、一致性Hash
-- 3. LIST分片：按枚举值

-- 分表示例
CREATE TABLE user_0 LIKE user;
CREATE TABLE user_1 LIKE user;
...

-- 路由规则
shardIndex = userId % tableCount;
```

#### 5.3 分库分表中间件
- ShardingSphere
- MyCAT
- Cobar
- Vitess

---

### 6. 慢查询优化
**学习日期：____-____-____**

#### 6.1 慢查询配置
```sql
-- 开启慢查询日志
SET GLOBAL slow_query_log = ON;
SET GLOBAL long_query_time = 1;  -- 超过1秒记录

-- 查看慢查询日志位置
SHOW VARIABLES LIKE 'slow_query_log_file';
```

#### 6.2 慢查询分析
```bash
# mysqldumpslow 工具
mysqldumpslow -s t -t 10 /var/log/mysql/slow.log

# pt-query-digest 工具
pt-query-digest /var/log/mysql/slow.log
```

#### 6.3 优化步骤
```
1. 开启慢查询日志
2. 分析慢SQL
3. 使用EXPLAIN分析执行计划
4. 优化索引或SQL语句
5. 验证优化效果
```

---

## 实战项目

### 项目：慢查询优化实践
**项目地址**：`projects/mysql-optimization/`

#### 优化案例
- [ ] 优化10个真实慢查询
- [ ] 生成优化报告
- [ ] 性能对比测试

---

## 学习资源

### 书籍
- 《高性能MySQL》Baron Schwartz
- 《MySQL技术内幕：InnoDB存储引擎》姜尧

### 视频
- 尚硅谷MySQL高级

---

## 学习总结

### 核心收获

### 待深入内容
