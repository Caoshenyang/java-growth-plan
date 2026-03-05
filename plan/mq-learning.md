# 消息队列深度学习笔记

## 学习目标
- [ ] RocketMQ（消息存储、消费模型、事务消息、延迟消息）
- [ ] Kafka（高吞吐设计、分区、副本、消费者组）
- [ ] 消息可靠性（不丢失、不重复、顺序性）
- [ ] 消息积压处理

## 学习周期
- 开始时间：____-____-____
- 预计完成：____-____-____
- 实际完成：____-____-____

## 学习笔记

### 1. RocketMQ深度剖析
**学习日期：____-____-____**

#### 1.1 核心概念
```java
// 1. NameServer（注册中心）
// - 管理Broker路由信息
// - 无状态，可集群部署

// 2. Broker（消息代理）
// - 消息存储、转发
// - 主从架构（Master + Slave）

// 3. Producer（消息生产者）
// - 发送消息到Broker
// - 支持同步/异步/单向发送

// 4. Consumer（消息消费者）
// - 拉取消息消费
// - 集群模式/广播模式
```

#### 1.2 消息存储
```java
// CommitLog（ commit log）
// - 所有消息顺序存储
// - 顺序写，随机读
// - 默认1G一个文件

// ConsumeQueue（消费队列）
// - CommitLog的索引文件
// - 按Topic分目录
// - 包含：消息位置、大小、tagsCode

// IndexFile（索引文件）
// - 为消息建立索引
// - 支持按Key查询消息
```

#### 1.3 消息发送
```java
// 同步发送
DefaultMQProducer producer = new DefaultMQProducer("group");
producer.setNamesrvAddr("localhost:9876");
producer.start();

Message msg = new Message("topic", "tag", "body".getBytes());
SendResult result = producer.send(msg);  // 同步等待

// 异步发送
producer.send(msg, new SendCallback() {
    @Override
    public void onSuccess(SendResult sendResult) {
        // 成功回调
    }

    @Override
    public void onException(Throwable e) {
        // 失败回调
    }
});

// 单向发送（不关心结果）
producer.sendOneway(msg);
```

#### 1.4 消息消费
```java
// 集群模式（负载均衡）
DefaultMQPushConsumer consumer = new DefaultMQPushConsumer("group");
consumer.setNamesrvAddr("localhost:9876");
consumer.subscribe("topic", "*");
consumer.setMessageModel(MessageModel.CLUSTERING);
consumer.registerMessageListener(new MessageListenerConcurrently() {
    @Override
    public ConsumeConcurrentlyStatus consumeMessage(
        List<MessageExt> msgs,
        ConsumeConcurrentlyContext context) {
        // 消息处理
        return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
    }
});
consumer.start();

// 广播模式（所有消费者都消费）
consumer.setMessageModel(MessageModel.BROADCASTING);
```

#### 1.5 事务消息
```java
// 发送事务消息
TransactionMQProducer producer = new TransactionMQProducer("group");
producer.setNamesrvAddr("localhost:9876");
producer.setTransactionListener(new TransactionListener() {
    @Override
    public LocalTransactionState executeLocalTransaction(Message msg, Object arg) {
        // 执行本地事务
        try {
            // 业务逻辑
            return LocalTransactionState.COMMIT_MESSAGE;
        } catch (Exception e) {
            return LocalTransactionState.ROLLBACK_MESSAGE;
        }
    }

    @Override
    public LocalTransactionState checkLocalTransaction(MessageExt msg) {
        // 回查本地事务状态
        return LocalTransactionState.COMMIT_MESSAGE;
    }
});
producer.start();

// 发送事务消息
Message msg = new Message("topic", "body".getBytes());
TransactionSendResult result = producer.sendMessageInTransaction(msg, null);
```

#### 1.6 延迟消息
```java
// 延迟级别：1s 5s 10s 30s 1m 2m 3m 4m 5m 6m 7m 8m 9m 10m 20m 30m 1h 2h
Message msg = new Message("topic", "body".getBytes);
msg.setDelayTimeLevel(3);  // 10s
SendResult result = producer.send(msg);
```

---

### 2. Kafka深度剖析
**学习日期：____-____-____**

#### 2.1 核心概念
```java
// 1. Topic（主题）
// - 消息的分类
// - 分为多个Partition

// 2. Partition（分区）
// - Topic的子集
// - 有序性保证
// - 水平扩展

// 3. Offset（偏移量）
// - 消息在Partition中的位置
// - Consumer维护消费位置

// 4. Consumer Group（消费者组）
// - 组内消费者负载均衡
// - 组间消费者广播消费
```

#### 2.2 高吞吐设计
```java
// 1. 顺序写
// - 磁盘顺序写性能高
// - 避免随机写

// 2. 零拷贝
// - sendfile系统调用
// - 减少数据拷贝次数

// 3. 批量发送
// - 积累一批消息发送
// - 减少网络请求

// 4. 数据压缩
// - 支持GZIP、Snappy、LZ4、Zstd
// - 减少网络传输量
```

#### 2.3 分区策略
```java
// 1. 轮询分区（Round-robin）
// 2. 随机分区（Random）
// 3. Key分区（Hash）
Properties props = new Properties();
props.put("partitioner.class", CustomPartitioner.class);

// 4. 自定义分区
public class CustomPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        // 自定义分区逻辑
        return 0;
    }
}
```

#### 2.4 副本机制
```java
// 副本作用：
// - 高可用
// - 数据冗余

// ISR（In-Sync Replicas）
// - 与Leader保持同步的副本集合
// - 只有ISR中的副本才能被选为新的Leader

// HW（High Watermark）
// - ISR所有副本都已复制的消息位置
// - 消费者只能消费到HW之前的消息
```

#### 2.5 消息消费
```java
// 自动提交
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "1000");

// 手动提交
props.put("enable.auto.commit", "false");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        // 处理消息
    }
    consumer.commitSync();  // 同步提交
    // consumer.commitAsync();  // 异步提交
}
```

#### 2.6 Rebalance（重平衡）
```java
// 触发条件：
// 1. 消费者数量变化
// 2. 订阅Topic数量变化
// 3. Topic分区数量变化

// Rebalance策略：
// - Range（范围）
// - RoundRobin（轮询）
// - Sticky（粘性）

// 避免Rebalance：
// 1. 增大session.timeout.ms
// 2. 增大max.poll.interval.ms
// 3. 减少max.poll.records
```

---

### 3. 消息可靠性保证
**学习日期：____-____-____**

#### 3.1 消息不丢失
```java
// 生产端：
// 1. 使用同步发送
// 2. 设置重试次数
producer.setRetryTimesWhenSendFailed(3);

// 3. 使用事务或ACK机制
props.put("acks", "all");  // Kafka（0/1/all）

// 服务端：
// 1. 刷盘策略（RocketMQ）
flushDiskType = SYNC_FLUSH  // 同步刷盘

// 2. 副本同步（RocketMQ）
brokerRole = SYNC_MASTER  // 同步复制

// 3. 多副本（Kafka）
replication.factor = 3

// 消费端：
// 1. 手动提交offset
// 2. 消息处理完成后再提交
consumer.commitSync();
```

#### 3.2 消息不重复
```java
// 幂等性设计：
// 1. 生产端幂等（开启幂等）
props.put("enable.idempotence", "true");

// 2. 消费端幂等
// - 业务去重表
// - Redis去重
// - 状态机去重

// Redis去重示例
String key = "msg:" + msgId;
Boolean exists = redis.setnx(key, "1");
if (exists) {
    redis.expire(key, 86400);
    // 处理消息
} else {
    // 消息已处理，跳过
}
```

#### 3.3 消息顺序性
```java
// 全局顺序：
// - 单分区、单消费者
// - 性能差

// 部分顺序：
// - 同一业务发到同一分区
// - 同一分区单消费者

// RocketMQ顺序消费
MessageQueueSelector selector = new MessageQueueSelector() {
    @Override
    public MessageQueue select(List<MessageQueue> mqs, Message msg, Object arg) {
        Integer id = (Integer) arg;
        int index = id % mqs.size();
        return mqs.get(index);
    }
};
producer.send(msg, selector, orderId);

// 顺序消费（MessageListenerOrderly）
consumer.registerMessageListener(new MessageListenerOrderly() {
    @Override
    public ConsumeOrderlyStatus consumeMessage(
        List<MessageExt> msgs,
        ConsumeOrderlyContext context) {
        // 顺序处理
        return ConsumeOrderlyStatus.SUCCESS;
    }
});
```

---

### 4. 消息积压处理
**学习日期：____-____-____**

#### 4.1 积压原因
```java
// 1. 消费速度 < 生产速度
// 2. 消费者故障
// 3. 消费逻辑耗时
// 4. 消费者数量不足
```

#### 4.2 解决方案
```java
// 1. 增加消费者（Consumer数量 <= Partition数量）

// 2. 优化消费逻辑
// - 批量处理
// - 异步处理
// - 缩短处理时间

// 3. 临时扩容方案
// - 创建新Topic（Partition数量翻倍）
// - 写程序将积压消息转发到新Topic
// - 部署大量消费者消费新Topic
// - 恢复正常后修改回原Topic

// 4. 丢弃部分消息（非关键业务）
// - 修改消费offset
// - 跳过积压消息
```

---

## 实战项目

### 项目：简易消息队列实现
**项目地址**：`projects/simple-mq/`

#### 项目目标
- [ ] 实现基本的消息发送/接收
- [ ] 实现消息持久化
- [ ] 实现消息确认机制
- [ ] 实现消费组管理

---

## 学习资源

### 书籍
- 《Kafka权威指南》
- 《RocketMQ技术内幕》

### 官方文档
- [RocketMQ文档](https://rocketmq.apache.org/zh/)
- [Kafka文档](https://kafka.apache.org/documentation/)

---

## 学习总结

### 核心收获

### 待深入内容
