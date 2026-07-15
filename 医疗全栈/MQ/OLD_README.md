下面只讲 Kafka，把它当作 Java 后端岗位中最常见的 MQ 来系统学习。你可以把 Kafka 理解为：一个高吞吐、可持久化、可水平扩展的分布式消息日志系统。它既能做传统 MQ，也能做日志采集、实时数据管道、事件驱动架构、流式处理的基础设施。

------

# 一、为什么需要 MQ

在没有 MQ 的系统中，服务之间通常是同步调用。

例如用户下单：

```text
订单服务 -> 库存服务
订单服务 -> 支付服务
订单服务 -> 短信服务
订单服务 -> 积分服务
订单服务 -> 报表服务
```

如果全部同步调用，会出现几个问题。

第一，调用链变长，响应慢。用户只是想下单，但系统却要等短信、积分、报表全部处理完才能返回。

第二，下游故障会影响主流程。比如短信服务挂了，下单本身不应该失败，但同步调用会导致订单服务被拖垮。

第三，流量高峰时下游扛不住。比如秒杀活动中订单请求瞬间暴涨，库存、积分、报表服务可能被直接打爆。

第四，系统强耦合。订单服务必须知道谁需要订单事件，后面新增一个“优惠券服务”也要改订单服务代码。

引入 MQ 后，架构变成：

```text
订单服务 -> Kafka -> 库存服务
              -> 支付服务
              -> 短信服务
              -> 积分服务
              -> 报表服务
```

订单服务只负责把“订单已创建”这个事件发送到 Kafka，然后快速返回。其他服务自己订阅并处理事件。

所以 MQ 的核心价值是：

```text
异步处理
系统解耦
削峰填谷
广播通知
数据同步
最终一致性
日志采集
流式处理
```

------

# 二、Kafka 的核心定位

Kafka 和传统消息队列有一个重要区别：Kafka 本质上是“分布式提交日志”。

传统队列更像：

```text
消息被消费后就从队列中删除
```

Kafka 更像：

```text
消息被追加到日志文件中，消费者通过 offset 记录自己读到哪里
```

也就是说，Kafka 中的消息不会因为被某个消费者消费了就立即删除，而是根据保留策略删除。

例如：

```text
Topic: order_event

offset=0  订单1创建
offset=1  订单2创建
offset=2  订单3支付成功
offset=3  订单4取消
```

消费者 A 读到 offset=2，消费者 B 读到 offset=1，它们互不影响。Kafka 只保存消息，消费进度由消费者组维护。

这使 Kafka 非常适合：

```text
高吞吐消息队列
订单事件流
日志采集
用户行为埋点
数据同步
实时数仓
流式计算
微服务事件驱动
```

------

# 三、Kafka 整体架构

Kafka 的核心组件包括：

```text
Producer        生产者
Consumer        消费者
Broker          Kafka 服务器节点
Topic           主题
Partition       分区
Replica         副本
Leader          分区主副本
Follower        分区从副本
Consumer Group  消费者组
Offset          消费位移
Controller      集群控制器
```

整体结构如下：

```text
Producer
   |
   v
Kafka Cluster
   |
   |-- Broker 1
   |     |-- Topic A - Partition 0 Leader
   |
   |-- Broker 2
   |     |-- Topic A - Partition 1 Leader
   |
   |-- Broker 3
         |-- Topic A - Partition 2 Leader

Consumer Group
   |-- Consumer 1 消费 Partition 0
   |-- Consumer 2 消费 Partition 1
   |-- Consumer 3 消费 Partition 2
```

Kafka 通过 Topic 做逻辑分类，通过 Partition 做并行与扩展，通过 Replica 做高可用，通过 Consumer Group 做负载均衡和广播。

------

# 四、Producer：生产者

Producer 负责把消息发送到 Kafka 的某个 Topic。

一条 Kafka 消息通常包含：

```text
topic
partition
key
value
headers
timestamp
offset
```

其中最重要的是 key 和 value。

value 是真正的业务数据，比如订单 JSON。

```json
{
  "orderId": "1001",
  "userId": "U001",
  "status": "CREATED",
  "amount": 99.9
}
```

key 用于决定消息发往哪个分区。Kafka 默认会对 key 做 hash，然后映射到某个 partition。

```text
partition = hash(key) % partition_count
```

如果订单事件以 orderId 作为 key，那么同一个订单的所有事件都会进入同一个 partition，从而保证该订单维度的顺序。

例如：

```text
key=order_1001 -> partition 0
key=order_1002 -> partition 1
key=order_1003 -> partition 2
```

如果 key 为 null，Kafka 会使用粘性分区策略，批量地把消息发送到某个分区，以提高吞吐量。

------

# 五、Topic：主题

Topic 是 Kafka 中消息的逻辑分类。

例如：

```text
order-created-topic       订单创建事件
order-paid-topic          订单支付事件
user-register-topic       用户注册事件
sms-send-topic            短信发送事件
log-collect-topic         日志采集事件
excel-export-topic        Excel 导出任务
```

Producer 把消息发送到 Topic，Consumer 从 Topic 订阅消息。

Topic 本身不是物理队列，它下面会被拆成多个 Partition。

------

# 六、Partition：分区

Partition 是 Kafka 的核心设计。

一个 Topic 可以有多个 Partition。

```text
Topic: order_event

Partition 0: offset 0, 1, 2, 3 ...
Partition 1: offset 0, 1, 2, 3 ...
Partition 2: offset 0, 1, 2, 3 ...
```

Partition 的作用有三个：

第一，提高吞吐量。多个 Partition 可以分布在不同 Broker 上，实现并行写入和并行消费。

第二，实现水平扩展。Topic 的数据可以被拆分到多个 Broker 上。

第三，提供局部顺序保证。Kafka 只能保证单个 Partition 内的消息有序，不能保证整个 Topic 全局有序。

这点非常重要。

Kafka 的顺序语义是：

```text
同一个 Partition 内有序
不同 Partition 之间不保证顺序
```

例如订单状态流转：

```text
订单创建 -> 订单支付 -> 订单发货 -> 订单完成
```

如果你要求同一个订单的状态事件有序，应该用 orderId 作为 key，让同一订单进入同一个 partition。

不能让同一个订单的事件随机进入不同分区，否则可能出现：

```text
消费者先收到“订单支付”
后收到“订单创建”
```

------

# 七、Broker：Kafka 节点

Broker 是 Kafka 集群中的服务器节点。

一个 Kafka 集群通常由多个 Broker 组成。

```text
Broker 1
Broker 2
Broker 3
```

每个 Broker 存储若干个 Topic 的若干个 Partition。

例如：

```text
Broker 1: order_event-0, log_event-2
Broker 2: order_event-1, user_event-0
Broker 3: order_event-2, log_event-1
```

Kafka 的高吞吐主要来自几个设计：

```text
顺序写磁盘
Page Cache
零拷贝
批量发送
压缩
分区并行
网络请求聚合
```

很多人以为 Kafka 快是因为“都在内存里”，其实不准确。Kafka 的消息最终是落盘的，但它使用顺序追加写日志文件，配合操作系统 Page Cache，所以吞吐非常高。

------

# 八、Replica：副本机制

为了防止 Broker 宕机导致数据丢失，Kafka 支持分区副本。

例如 Topic 有 3 个分区，每个分区副本数为 3：

```text
order_event-0:
  Leader: Broker 1
  Follower: Broker 2
  Follower: Broker 3

order_event-1:
  Leader: Broker 2
  Follower: Broker 1
  Follower: Broker 3

order_event-2:
  Leader: Broker 3
  Follower: Broker 1
  Follower: Broker 2
```

每个 Partition 有一个 Leader，多个 Follower。

Producer 只向 Leader 写入。Consumer 通常也从 Leader 读取。Follower 从 Leader 同步数据。

如果 Leader 所在 Broker 宕机，Kafka 会从可用 Follower 中选举新的 Leader。

这就是 Kafka 的高可用基础。

------

# 九、ISR：同步副本集合

ISR 全称 In-Sync Replicas，即“与 Leader 保持同步的副本集合”。

例如：

```text
Partition 0:
Leader = Broker 1
ISR = Broker 1, Broker 2, Broker 3
```

如果 Broker 3 同步太慢，就会被踢出 ISR：

```text
ISR = Broker 1, Broker 2
```

ISR 对消息可靠性非常关键。

Producer 的 acks 参数会和 ISR 配合使用。

------

# 十、Producer 消息确认机制：acks

Producer 发送消息后，需要 Kafka 返回确认。

Kafka 的确认级别由 `acks` 控制。

`acks=0` 表示 Producer 发出去就不管了，不等待 Broker 确认。

```text
优点：最快
缺点：最容易丢消息
适合：日志、埋点等允许少量丢失的场景
```

`acks=1` 表示 Leader 写入成功后就返回确认。

```text
优点：性能较好
缺点：如果 Leader 写入后还没同步给 Follower 就宕机，消息可能丢失
适合：一般可靠性场景
```

`acks=all` 或 `acks=-1` 表示 Leader 和 ISR 中足够副本确认后才返回成功。

```text
优点：可靠性最高
缺点：延迟更高
适合：订单、支付、库存等核心业务
```

核心业务通常建议：

```properties
acks=all
enable.idempotence=true
retries=Integer.MAX_VALUE
max.in.flight.requests.per.connection<=5
min.insync.replicas=2
replication.factor=3
```

其中 `min.insync.replicas=2` 表示至少要有 2 个 ISR 副本可用，才能认为写入成功。

如果副本数是 3，`min.insync.replicas=2`，那么可以容忍 1 个副本故障。

------

# 十一、消息发送流程

Producer 发送消息不是每条都立即发到 Broker，而是经过一套缓冲和批量发送机制。

大致流程如下：

```text
业务线程调用 send()
        |
        v
序列化 key/value
        |
        v
根据 topic + key 选择 partition
        |
        v
写入本地 RecordAccumulator 缓冲区
        |
        v
Sender 线程批量发送到 Broker
        |
        v
Broker 写入 Leader 分区日志
        |
        v
Follower 拉取同步
        |
        v
Broker 返回 ack
        |
        v
Producer 回调 callback
```

Kafka Producer 的性能来自批处理。

关键参数包括：

```properties
batch.size=16384
linger.ms=5
buffer.memory=33554432
compression.type=snappy
```

`batch.size` 控制批次大小。

`linger.ms` 控制等待时间。比如设置为 5ms，Producer 会稍微等一下，看是否能攒更多消息一起发送。

`buffer.memory` 是 Producer 端总缓冲区大小。

`compression.type` 是压缩算法，常见有 none、gzip、snappy、lz4、zstd。

一般来说：

```text
低延迟优先：linger.ms 小一些
高吞吐优先：batch.size 大一些，linger.ms 适当增加，开启压缩
```

------

# 十二、Consumer：消费者

Consumer 负责从 Kafka Topic 中拉取消息。

Kafka Consumer 是拉模式，不是推模式。

也就是说，不是 Kafka 主动把消息推给消费者，而是消费者主动 poll 拉取消息。

典型代码结构：

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        // 处理消息
    }
}
```

拉模式的好处是消费者可以根据自己的处理能力控制节奏，不会被 Broker 直接推爆。

但坏处是需要正确处理 poll、offset 提交、重平衡、异常重试等问题。

------

# 十三、Consumer Group：消费者组

消费者组是 Kafka 消费模型的核心。

同一个 Consumer Group 中的消费者共同消费一个 Topic。

规则是：

```text
同一个 Partition 同一时刻只能被同一个 Consumer Group 内的一个 Consumer 消费
一个 Consumer 可以消费多个 Partition
```

例如 Topic 有 3 个 Partition，消费者组有 3 个 Consumer：

```text
Partition 0 -> Consumer A
Partition 1 -> Consumer B
Partition 2 -> Consumer C
```

如果消费者组有 2 个 Consumer：

```text
Partition 0 -> Consumer A
Partition 1 -> Consumer A
Partition 2 -> Consumer B
```

如果消费者组有 5 个 Consumer：

```text
Partition 0 -> Consumer A
Partition 1 -> Consumer B
Partition 2 -> Consumer C
Consumer D 空闲
Consumer E 空闲
```

所以一个 Topic 的并行消费能力上限主要由 Partition 数决定。

如果有多个不同 Consumer Group 订阅同一个 Topic，它们之间是广播关系。

例如：

```text
order_event_topic
   |
   |-- 库存服务 group: inventory-service
   |-- 积分服务 group: point-service
   |-- 短信服务 group: sms-service
   |-- 报表服务 group: report-service
```

每个 group 都能完整消费一份订单事件。

------

# 十四、Offset：消费位移

Offset 是 Kafka 中消息在 Partition 内的位置。

每个 Partition 的 offset 独立递增。

```text
Partition 0:
offset 0
offset 1
offset 2
offset 3
```

Consumer 通过 offset 记录自己消费到哪里。

例如：

```text
order-service-group 对 order_event-0 消费到 offset=100
report-service-group 对 order_event-0 消费到 offset=20
```

不同消费者组的 offset 互不影响。

Kafka 会把消费者组的 offset 存储在内部 Topic：

```text
__consumer_offsets
```

注意：Offset 不是消息 ID。Offset 只在某个 Partition 内唯一，不能跨 Partition 全局唯一。

------

# 十五、自动提交与手动提交

Consumer 消费完消息后，需要提交 offset。

Kafka 支持自动提交：

```properties
enable.auto.commit=true
auto.commit.interval.ms=5000
```

自动提交的缺点是可能消息还没真正处理完，offset 就已经提交了。此时服务宕机，会导致消息丢失。

例如：

```text
poll 拉取消息
自动提交 offset
业务处理到一半宕机
重启后从新 offset 开始消费
之前消息不会再消费
```

核心业务通常建议手动提交：

```properties
enable.auto.commit=false
```

处理成功后再提交：

```java
consumer.commitSync();
```

但手动提交也要注意。如果业务处理成功了，但 offset 提交失败，消息会被重复消费。

因此 Kafka 消费端天然更容易实现“至少一次”，即：

```text
消息不会丢，但可能重复
```

所以业务处理必须具备幂等性。

------

# 十六、Kafka 的三种消息语义

Kafka 常见消息语义有三类：

```text
最多一次 At Most Once
至少一次 At Least Once
恰好一次 Exactly Once
```

最多一次：

```text
先提交 offset，再处理消息
可能丢消息，不会重复
```

至少一次：

```text
先处理消息，再提交 offset
不容易丢消息，但可能重复
```

恰好一次：

```text
消息既不丢，也不重复产生业务效果
```

在实际 Java 后端业务中，最常用的是“至少一次 + 业务幂等”。

真正的 Exactly Once 通常要求 Kafka Producer 幂等、事务、消费者 offset 与输出结果在同一事务边界中处理。它更适合 Kafka 到 Kafka 的流处理，而不是所有外部数据库业务都能天然做到。

------

# 十七、重复消费

Kafka 中重复消费非常常见，不是异常现象。

常见原因包括：

```text
消费成功但 offset 提交失败
消费者处理超时导致 rebalance
服务宕机重启
手动 seek offset
重试机制重复投递
Producer 重试导致重复发送
网络抖动导致确认丢失
```

例如：

```text
Consumer 处理订单消息成功
数据库已经插入记录
准备 commit offset
服务宕机
重启后 Kafka 认为 offset 没提交
再次投递同一条消息
```

所以业务必须设计幂等。

常见幂等方案有：

第一，使用业务唯一键。

例如订单支付消息中有：

```text
paymentId
orderId
eventId
```

数据库建立唯一索引：

```sql
CREATE UNIQUE INDEX uk_payment_event ON payment_record(event_id);
```

重复消息再次插入时会违反唯一约束，从而避免重复处理。

第二，建立消息消费表。

```sql
CREATE TABLE message_consume_record (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    message_id VARCHAR(128) NOT NULL,
    consumer_group VARCHAR(128) NOT NULL,
    status VARCHAR(32) NOT NULL,
    create_time DATETIME NOT NULL,
    UNIQUE KEY uk_msg_group(message_id, consumer_group)
);
```

消费前先判断是否处理过。

第三，利用 Redis 去重。

```text
SETNX consume:order_event:event_1001 1
```

设置成功才处理，失败说明已处理过。

第四，业务状态机天然幂等。

例如订单状态只能从：

```text
CREATED -> PAID -> SHIPPED -> FINISHED
```

如果当前订单已经是 PAID，再收到 PAID 消息，直接忽略。

第五，下游接口支持幂等号。

例如调用支付、退款、发券接口时传入 requestId，保证同一个 requestId 只处理一次。

------

# 十八、顺序消费

Kafka 只能保证 Partition 内有序。

要实现顺序消费，需要满足：

```text
相关消息发送到同一个 Topic
相关消息使用同一个 key
同一个 key 落到同一个 Partition
同一个 Partition 同一时刻只被同一个 Consumer 消费
Consumer 内部不能乱序并发处理
```

例如订单状态流转：

```java
producer.send(new ProducerRecord<>("order-event-topic", orderId, eventJson));
```

只要 orderId 相同，就会进入同一个 Partition。

但注意几个坑。

第一，如果 Topic 增加 Partition 数，key 到 Partition 的映射可能变化。后续消息可能进入新 Partition，从而影响严格顺序。

第二，如果消费者内部把消息丢到线程池并发处理，可能导致乱序。

例如：

```java
for (record : records) {
    threadPool.submit(() -> handle(record));
}
```

这种方式会破坏同一 Partition 内的处理顺序。

第三，如果失败消息被跳过，后续消息继续处理，也可能破坏业务顺序。

例如：

```text
订单创建处理失败
订单支付继续处理
```

所以严格顺序场景通常需要：

```text
按 key 分区
单分区顺序处理
失败阻塞或转入有序重试机制
状态机校验
```

实际业务中更常见的是“局部有序”，比如保证同一个订单、同一个用户、同一个设备的事件有序，而不是全局有序。

------

# 十九、消息丢失

Kafka 消息丢失通常发生在三个环节：

```text
Producer 端丢失
Broker 端丢失
Consumer 端丢失
```

Producer 端丢失常见原因：

```text
acks=0
发送失败但没有重试
异步发送没有检查 callback
buffer 满了消息被丢弃或发送超时
序列化失败但业务未感知
```

解决方案：

```properties
acks=all
retries=Integer.MAX_VALUE
enable.idempotence=true
delivery.timeout.ms=120000
request.timeout.ms=30000
```

异步发送时必须处理 callback：

```java
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // 记录日志、告警、重试、落库补偿
    }
});
```

Broker 端丢失常见原因：

```text
副本数太少
min.insync.replicas 设置不合理
unclean leader election 导致落后副本成为 Leader
磁盘故障
消息还未刷盘机器就宕机
```

解决方案：

```properties
replication.factor=3
min.insync.replicas=2
acks=all
unclean.leader.election.enable=false
```

Consumer 端丢失常见原因：

```text
自动提交 offset
先提交 offset 后处理业务
异常被吞掉
批量消费中部分失败但整体提交
```

解决方案：

```text
关闭自动提交
业务处理成功后再提交 offset
异常消息进入重试或 DLQ
批量处理要记录每条消息的处理结果
```

核心业务推荐模式：

```text
Producer: acks=all + 幂等发送 + callback 检查
Broker: 副本数 3 + min ISR 2
Consumer: 手动提交 offset + 业务幂等 + 失败重试 + DLQ
```

------

# 二十、消息确认：Kafka 中的“确认”分两类

很多人说“消息确认”，但 Kafka 中至少有两种确认。

第一类是 Producer 到 Broker 的确认。

```text
Producer 发消息 -> Broker 写入成功 -> 返回 ack
```

这个由 `acks` 控制。

第二类是 Consumer 消费确认。

```text
Consumer 消费消息 -> 业务处理成功 -> 提交 offset
```

这个由 offset commit 控制。

所以面试时要区分：

```text
生产端确认：acks
消费端确认：offset commit
```

不要把它们混为一谈。

------

# 二十一、失败重试

Kafka 本身没有像某些传统 MQ 那样内置完善的消费失败重试队列模型。Java 后端一般自己设计。

常见重试方式有三种。

第一，Consumer 本地立即重试。

```java
for (int i = 0; i < 3; i++) {
    try {
        handle(record);
        break;
    } catch (Exception e) {
        Thread.sleep(1000);
    }
}
```

优点是简单。缺点是阻塞当前 Partition，影响后续消息。

适合短暂异常，比如网络抖动、数据库瞬时连接失败。

第二，重试 Topic。

例如：

```text
order-event-topic
order-event-retry-1m-topic
order-event-retry-5m-topic
order-event-retry-30m-topic
order-event-dlq-topic
```

消费失败后，不是一直阻塞原 Topic，而是把消息发送到 retry topic。

```text
第一次失败 -> retry-1m
第二次失败 -> retry-5m
第三次失败 -> retry-30m
最终失败 -> DLQ
```

消息中需要携带重试次数：

```json
{
  "eventId": "evt_1001",
  "orderId": "1001",
  "retryCount": 2,
  "originalTopic": "order-event-topic",
  "errorMessage": "database timeout"
}
```

第三，数据库任务表重试。

对于特别重要的任务，可以把失败消息落入数据库任务表，由定时任务扫描重试。

```sql
CREATE TABLE mq_retry_task (
    id BIGINT PRIMARY KEY,
    topic VARCHAR(128),
    message_key VARCHAR(128),
    message_body TEXT,
    retry_count INT,
    next_retry_time DATETIME,
    status VARCHAR(32),
    error_message TEXT
);
```

这种方式可靠、可查询、可人工干预，适合订单、支付、退款等核心链路。

------

# 二十二、死信队列 DLQ

死信队列就是专门存放“最终处理失败消息”的 Topic。

Kafka 中一般用普通 Topic 来模拟 DLQ。

例如：

```text
order-event-dlq-topic
sms-send-dlq-topic
excel-export-dlq-topic
```

进入 DLQ 的消息通常包括：

```text
原始 topic
原始 partition
原始 offset
message key
message body
失败原因
异常堆栈
重试次数
失败时间
消费者组
```

示例：

```json
{
  "originalTopic": "order-event-topic",
  "partition": 2,
  "offset": 10923,
  "key": "order_1001",
  "value": {
    "orderId": "1001",
    "status": "PAID"
  },
  "consumerGroup": "inventory-service",
  "retryCount": 5,
  "errorMessage": "inventory not found",
  "failedAt": "2026-06-18 10:00:00"
}
```

DLQ 的作用不是“解决问题”，而是避免坏消息卡住主消费链路，同时保留现场，方便后续补偿。

生产环境必须对 DLQ 做监控。

例如：

```text
DLQ 消息数量 > 0 告警
DLQ 消息堆积超过阈值告警
DLQ 按错误类型聚合分析
提供后台页面人工重放
```

------

# 二十三、延迟队列

Kafka 原生不适合做复杂延迟队列。

Kafka 的消息模型是按 offset 顺序消费，不支持对每条消息设置任意投递时间，然后自动延迟投递。

但可以通过几种方式实现延迟效果。

第一，延迟 Topic 分层。

```text
delay-1m-topic
delay-5m-topic
delay-30m-topic
```

消费者消费 delay topic 时，如果时间未到，就暂停处理或重新投递。

第二，消息中携带 executeTime。

```json
{
  "taskId": "task_1001",
  "executeTime": "2026-06-18 10:30:00",
  "payload": {}
}
```

消费者判断当前时间是否到达执行时间。

但这种方式有个问题：如果队头消息没到时间，可能阻塞后续已经到时间的消息。

第三，数据库延迟任务表 + 定时扫描。

对于订单超时取消这类业务，更常见做法是数据库表配合定时任务：

```sql
SELECT * FROM delay_task
WHERE status = 'INIT'
AND execute_time <= NOW()
LIMIT 100;
```

然后投递到 Kafka 或直接处理。

第四，时间轮。

可以用时间轮管理延迟任务，到期后投递 Kafka。

总结：

```text
Kafka 可以实现延迟效果，但不是天然延迟队列。
复杂延迟任务更适合用任务表、时间轮或专门的延迟消息组件。
```

如果面试官问“Kafka 怎么实现延迟队列”，可以回答：

```text
Kafka 不原生支持任意延迟消息。常见做法是延迟 Topic 分层、消息携带执行时间、数据库任务表扫描、时间轮调度，到期后再投递 Kafka。需要注意队头阻塞、重试次数、消息幂等和 DLQ。
```

------

# 二十四、消息堆积

消息堆积是 Kafka 面试和生产中非常重要的问题。

消息堆积指：

```text
Producer 生产速度 > Consumer 消费速度
```

Kafka 中通常用 lag 表示堆积量。

```text
lag = log end offset - committed offset
```

例如：

```text
Partition 0 最新 offset = 10000
Consumer 已提交 offset = 7000
lag = 3000
```

消息堆积的常见原因：

```text
消费者处理逻辑太慢
消费者实例太少
Partition 数太少
下游数据库慢
外部接口慢
单条消息处理阻塞
消费者频繁 rebalance
消息体太大
消费者异常重试导致卡住
```

解决方案要分层看。

如果是消费实例不够：

```text
增加 Consumer 实例
但实例数不能超过 Partition 数
```

如果 Partition 数太少：

```text
增加 Topic Partition 数
但要注意 key 顺序和分区映射变化
```

如果业务处理慢：

```text
优化 SQL
增加批量处理
异步化下游调用
使用线程池并发处理
引入本地缓冲
减少远程调用
```

如果单条坏消息卡住：

```text
增加重试次数上限
失败进入 DLQ
避免无限重试阻塞 Partition
```

如果下游系统扛不住：

```text
限流
降级
批处理
削峰消费
增加下游容量
```

如果消费线程池乱序风险高：

```text
按 key 分发到固定工作线程
保证同 key 串行处理
```

例如可以使用 key-hash 到 worker：

```text
workerIndex = hash(orderId) % workerCount
```

同一个 orderId 永远进入同一个 worker，从而兼顾并发和局部顺序。

------

# 二十五、削峰填谷

削峰填谷是 MQ 的典型场景。

没有 Kafka 时：

```text
秒杀请求 -> 订单服务 -> 库存服务 -> 数据库
```

瞬间流量可能直接打爆数据库。

引入 Kafka 后：

```text
秒杀请求 -> 快速写入 Kafka -> 后台消费者按能力慢慢处理
```

Kafka 作为缓冲层，把瞬时高峰削平。

```text
高峰期：消息堆积在 Kafka
低峰期：消费者慢慢追上
```

但削峰填谷不是无限承载。Kafka 能抗流量，不代表下游无限处理。

必须配合：

```text
限流
熔断
降级
库存预扣减
消费者扩容
堆积监控
超时处理
```

例如秒杀系统中，通常不是所有请求都进入 Kafka，而是先做资格校验、库存预扣、限流，然后再发送订单创建消息。

------

# 二十六、系统解耦

系统解耦指生产者不需要知道有哪些消费者。

例如订单服务只发送：

```text
OrderCreatedEvent
```

后续库存、短信、积分、报表、风控都可以独立订阅。

```text
订单服务不关心谁消费
消费者上线下线不影响订单服务
新增业务不需要修改订单服务
```

但解耦不是没有成本。引入 Kafka 后，系统会变成最终一致性，需要处理：

```text
消息是否成功发送
消费者是否成功处理
失败如何补偿
重复消费如何幂等
消息顺序如何保证
积压如何监控
DLQ 如何处理
```

所以 MQ 的本质是：用异步和最终一致性换取吞吐、可用性和扩展性。

------

# 二十七、最终一致性

最终一致性是 MQ 最重要的业务思想之一。

例如订单支付成功后：

```text
支付服务更新支付状态
发送 OrderPaidEvent
库存服务扣减库存
积分服务增加积分
短信服务发送通知
报表服务更新统计
```

这些动作不需要在同一个同步事务中完成。只要最终都完成即可。

但是要考虑异常：

```text
支付状态更新成功，但消息发送失败怎么办？
消息发送成功，但库存消费失败怎么办？
库存消费成功，但 offset 提交失败怎么办？
积分重复增加怎么办？
```

常见最终一致性方案有：

```text
本地消息表
事务消息
Outbox Pattern
消费端幂等
失败重试
DLQ
补偿任务
对账任务
```

------

# 二十八、本地消息表 / Outbox Pattern

本地消息表是解决“数据库操作成功，但 Kafka 消息发送失败”的经典方案。

例如订单创建时，不是直接：

```text
插入订单
发送 Kafka
```

而是在同一个数据库事务里：

```text
插入订单表
插入消息表
提交事务
```

表结构：

```sql
CREATE TABLE local_message (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    event_id VARCHAR(128) NOT NULL,
    topic VARCHAR(128) NOT NULL,
    message_key VARCHAR(128),
    message_body TEXT NOT NULL,
    status VARCHAR(32) NOT NULL,
    retry_count INT NOT NULL DEFAULT 0,
    create_time DATETIME NOT NULL,
    update_time DATETIME NOT NULL,
    UNIQUE KEY uk_event_id(event_id)
);
```

流程：

```text
1. 业务数据库事务中写订单表
2. 同一个事务中写 local_message
3. 定时任务扫描未发送消息
4. 发送 Kafka
5. 发送成功后更新消息表状态
6. 失败则重试
```

这样可以保证：

```text
订单创建成功，则消息一定会被记录
消息发送失败，也可以补偿重发
```

这就是 Outbox Pattern。

缺点是需要额外表和定时任务，存在一定延迟。但在金融、订单、支付等场景非常实用。

------

# 二十九、Kafka 事务

Kafka 支持 Producer 事务，可以把多条消息写入多个 Topic/Partition，作为一个原子操作提交或回滚。

典型配置：

```properties
transactional.id=order-tx-producer-1
enable.idempotence=true
acks=all
```

基本流程：

```java
producer.initTransactions();

try {
    producer.beginTransaction();

    producer.send(new ProducerRecord<>("topic-a", key, value1));
    producer.send(new ProducerRecord<>("topic-b", key, value2));

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

Kafka 事务主要解决的是：

```text
多条 Kafka 消息之间的原子性
Kafka 消费 offset 与 Kafka 输出消息之间的原子性
```

例如 Kafka Streams 中：

```text
从 topic-a 消费
处理后写入 topic-b
同时提交 topic-a 的 offset
```

可以做到 Kafka 内部链路的 Exactly Once。

但 Kafka 事务不能自动把 MySQL 事务也包进去。

例如：

```text
更新 MySQL
发送 Kafka 消息
```

这不是单靠 Kafka 事务就能完全解决的，因为 MySQL 和 Kafka 是两个不同的事务系统。

此时更常用的是：

```text
本地消息表 / Outbox Pattern
事务消息思想
对账补偿
```

------

# 三十、Producer 幂等

Producer 幂等用于解决 Producer 重试导致的重复写入问题。

例如：

```text
Producer 发送消息
Broker 实际写入成功
ack 返回时网络异常
Producer 以为失败并重试
```

如果没有幂等，Broker 可能写入两条相同消息。

开启：

```properties
enable.idempotence=true
```

Kafka 会为 Producer 分配 ProducerId，并为每个分区维护序列号，从而识别重复消息。

幂等 Producer 的作用范围是：

```text
同一个 Producer 会话内
同一个 Topic-Partition 内
防止生产端重试导致的重复写入
```

但它不能解决所有业务重复问题。

例如：

```text
业务代码主动调用 send 两次
服务重启后重新发送同一业务消息
消费者处理重复
```

这些仍然需要业务幂等。

------

# 三十一、交换机这个概念在 Kafka 中如何理解

你列出的“交换机”是传统 MQ 里的概念，Kafka 没有 RabbitMQ 那种 Exchange。

Kafka 的路由模型更简单：

```text
Producer 直接发送到 Topic
Topic 再根据 key/partitioner 进入具体 Partition
Consumer 订阅 Topic
```

Kafka 中近似承担“路由”作用的是：

```text
Topic 命名
消息 key
Partitioner 分区器
Consumer Group
Kafka Streams 分流逻辑
```

例如：

```text
订单创建事件 -> order-created-topic
订单支付事件 -> order-paid-topic
订单取消事件 -> order-cancelled-topic
```

或者统一进入一个 Topic：

```text
order-event-topic
```

消息体中带 eventType：

```json
{
  "eventType": "ORDER_PAID",
  "orderId": "1001"
}
```

消费者根据 eventType 处理。

两种设计都可以。

如果事件类型少、消费者边界清晰，可以拆多个 Topic。

如果事件属于同一业务流、需要按订单聚合或保持顺序，可以使用一个统一 Topic，靠 eventType 区分。

------

# 三十二、队列这个概念在 Kafka 中如何理解

Kafka 里没有传统意义上“一个队列被多个消费者抢消息，消费后删除”的模型。

Kafka 的队列感来自：

```text
Topic + Partition + Consumer Group
```

同一个 Consumer Group 内部是队列模式：

```text
一条消息只会被 group 内的一个 consumer 消费
```

不同 Consumer Group 之间是发布订阅模式：

```text
一条消息会被多个 group 各自消费一遍
```

所以 Kafka 同时支持：

```text
队列模式：同一个 group 多消费者负载均衡
发布订阅模式：多个 group 独立订阅同一 Topic
```

------

# 三十三、消息存储机制

Kafka 把每个 Partition 存储为一组日志段文件。

逻辑结构：

```text
topic-order-event/
  partition-0/
    00000000000000000000.log
    00000000000000000000.index
    00000000000000000000.timeindex

    00000000000000100000.log
    00000000000000100000.index
    00000000000000100000.timeindex
```

`.log` 存消息数据。

`.index` 存 offset 到物理位置的稀疏索引。

`.timeindex` 存时间戳索引。

Kafka 使用顺序追加写：

```text
新消息只追加到 log 文件末尾
```

不会频繁随机写，所以磁盘吞吐很高。

Kafka 查找某个 offset 时，不是从头扫描，而是通过 index 找到近似位置，再顺序扫描少量数据。

------

# 三十四、消息保留与删除

Kafka 消息不会因为消费成功就删除，而是根据保留策略删除。

常见配置：

```properties
log.retention.hours=168
log.retention.bytes=1073741824
```

表示保留 7 天，或者超过一定大小后清理。

也可以按 Topic 设置：

```bash
kafka-configs.sh --alter \
  --entity-type topics \
  --entity-name order-event-topic \
  --add-config retention.ms=604800000
```

Kafka 还有日志压缩 Log Compaction。

普通删除策略是 delete：

```text
按时间或大小删除旧日志
```

压缩策略是 compact：

```text
同一个 key 只保留最新 value
```

适合保存状态变更类数据，例如：

```text
userId -> user profile
configKey -> configValue
```

但订单流水、支付流水、日志采集一般不适合 compact，因为历史事件都有价值。

------

# 三十五、Kafka 为什么快

Kafka 高性能来自多个方面。

第一，顺序写磁盘。

随机写磁盘很慢，但顺序写磁盘很快。Kafka 的 Partition 日志是追加写。

第二，Page Cache。

Kafka 大量依赖操作系统 Page Cache。消息写入文件系统后，操作系统会缓存到内存中，消费者读取时也经常命中 Page Cache。

第三，零拷贝。

Kafka 发送文件数据给消费者时，可以使用操作系统的零拷贝能力，减少用户态和内核态之间的数据复制。

第四，批处理。

Producer 会批量发送消息，Broker 批量写入，Consumer 批量拉取。

第五，压缩。

多条消息组成 batch 后一起压缩，可以降低网络传输和磁盘占用。

第六，分区并行。

多个 Partition 可以分布在多个 Broker 上，同时读写。

所以 Kafka 适合高吞吐、大数据量、日志流、事件流场景。

------

# 三十六、Kafka 的可靠性配置

核心业务推荐 Topic 配置：

```properties
replication.factor=3
min.insync.replicas=2
unclean.leader.election.enable=false
```

Producer 配置：

```properties
acks=all
enable.idempotence=true
retries=Integer.MAX_VALUE
delivery.timeout.ms=120000
request.timeout.ms=30000
max.in.flight.requests.per.connection=5
compression.type=snappy
```

Consumer 配置：

```properties
enable.auto.commit=false
auto.offset.reset=earliest
max.poll.records=100
max.poll.interval.ms=300000
session.timeout.ms=10000
heartbeat.interval.ms=3000
```

含义说明：

`replication.factor=3` 表示每个分区有 3 个副本。

`min.insync.replicas=2` 表示至少 2 个同步副本确认才算成功。

`unclean.leader.election.enable=false` 表示不允许落后副本成为 Leader，避免数据丢失。

`enable.idempotence=true` 表示开启生产者幂等。

`enable.auto.commit=false` 表示关闭自动提交 offset。

`max.poll.records` 控制每次 poll 拉取的最大记录数，防止一次拉太多处理不过来。

`max.poll.interval.ms` 表示两次 poll 之间允许的最大间隔。如果业务处理太慢超过该时间，Kafka 会认为消费者失联，触发 rebalance。

------

# 三十七、Rebalance：消费者重平衡

Rebalance 指 Consumer Group 中 Partition 分配发生变化。

触发原因：

```text
消费者加入 group
消费者离开 group
消费者宕机
Topic 分区数变化
消费者长时间不 poll
心跳超时
```

例如原来：

```text
P0 -> C1
P1 -> C1
P2 -> C2
```

新增 C3 后可能变成：

```text
P0 -> C1
P1 -> C2
P2 -> C3
```

Rebalance 的问题：

```text
消费暂停
重复消费
处理中的消息可能被重新分配
频繁 rebalance 会导致吞吐下降
```

优化方式：

```text
控制消费者处理时间
合理设置 max.poll.interval.ms
避免频繁扩缩容
使用静态成员机制
使用 cooperative sticky 分配策略
处理完消息后再提交 offset
```

面试中要知道：消费者不是越多越好。消费者数超过 Partition 数后，多余消费者会空闲，而且频繁变动会引发 rebalance。

------

# 三十八、分区分配策略

Consumer Group 内部需要决定哪个 Consumer 消费哪些 Partition。

常见分配策略包括：

```text
RangeAssignor
RoundRobinAssignor
StickyAssignor
CooperativeStickyAssignor
```

RangeAssignor 按范围分配。可能导致某些消费者负载更高。

RoundRobinAssignor 轮询分配，比较均匀。

StickyAssignor 尽量保持上一次分配结果，减少 rebalance 带来的迁移。

CooperativeStickyAssignor 是增量协作式重平衡，减少全量停止消费的影响。

生产中更推荐 sticky 或 cooperative sticky 这一类更平滑的策略。

------

# 三十九、Kafka 与数据库事务的一致性问题

这是 Java 后端面试高频题。

场景：

```text
订单服务创建订单，然后发送 Kafka 消息通知库存服务
```

伪代码：

```java
@Transactional
public void createOrder(Order order) {
    orderMapper.insert(order);
    kafkaTemplate.send("order-created-topic", order.getId(), json);
}
```

这个代码有隐患。

情况一：

```text
订单插入成功
Kafka 发送失败
数据库事务提交
结果：订单存在，但下游不知道
```

情况二：

```text
Kafka 发送成功
数据库事务回滚
结果：下游收到不存在的订单
```

更好的方案是本地消息表：

```java
@Transactional
public void createOrder(Order order) {
    orderMapper.insert(order);
    localMessageMapper.insert(buildOrderCreatedMessage(order));
}
```

然后异步任务发送消息。

或者使用事务同步，在数据库事务提交后再发送 Kafka：

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override
    public void afterCommit() {
        kafkaTemplate.send("order-created-topic", order.getId(), json);
    }
});
```

但 afterCommit 仍然可能发送失败，所以核心业务仍建议结合本地消息表。

------

# 四十、消息事务与最终一致性示例：订单创建

推荐流程：

```text
1. 用户提交订单
2. 订单服务开启数据库事务
3. 插入订单表，状态 INIT
4. 插入本地消息表，事件 OrderCreated
5. 提交数据库事务
6. 定时任务扫描本地消息表
7. 发送 Kafka
8. 发送成功后标记消息 SENT
9. 库存服务消费消息
10. 库存服务幂等扣减库存
11. 库存扣减成功后提交 offset
12. 失败则重试，多次失败进入 DLQ
```

这个方案能处理：

```text
订单成功但消息失败
消息重复发送
消费者重复消费
库存服务短暂不可用
消息最终补偿
```

------

# 四十一、异步发送短信

场景：

```text
用户注册成功后发送短信
```

不应该同步调用短信服务。

推荐流程：

```text
用户服务注册成功
发送 UserRegisteredEvent 到 Kafka
短信服务消费事件
调用短信供应商
成功后提交 offset
失败则重试
多次失败进入 sms-dlq-topic
```

Topic 设计：

```text
user-registered-topic
sms-send-retry-topic
sms-send-dlq-topic
```

消息示例：

```json
{
  "eventId": "evt_001",
  "userId": "U001",
  "phone": "13800000000",
  "templateCode": "REGISTER_SUCCESS",
  "params": {
    "username": "Tom"
  }
}
```

注意点：

```text
短信允许一定延迟
短信发送必须幂等，避免重复发送
可以用 eventId 或 businessKey 去重
失败要记录供应商返回码
DLQ 需要人工补偿或后台重发
```

如果短信重复发送影响不大，可以接受“至少一次”。如果严格不能重复，需要在短信服务端建立发送记录表，用唯一键控制。

------

# 四十二、异步生成报表

场景：

```text
用户点击生成月度报表
```

报表生成耗时长，不应阻塞 HTTP 请求。

推荐流程：

```text
前端提交报表任务
报表服务写 report_task 表，状态 INIT
发送 report-generate-topic
消费者异步生成报表
生成完成后更新任务状态为 SUCCESS
前端轮询或 WebSocket 通知用户
```

消息示例：

```json
{
  "taskId": "report_1001",
  "userId": "U001",
  "reportType": "MONTHLY_SALES",
  "startDate": "2026-06-01",
  "endDate": "2026-06-30"
}
```

注意点：

```text
任务必须有状态表
不能只依赖 Kafka 消息
生成过程要可重试
重复消费时根据 taskId 判断任务是否已完成
大报表结果存对象存储，不要塞进 Kafka
Kafka 消息只传任务 ID 和参数，不传大文件
```

任务状态可以设计为：

```text
INIT
PROCESSING
SUCCESS
FAILED
CANCELLED
```

------

# 四十三、异步导出 Excel

Excel 导出和报表类似，但更强调大文件处理。

错误设计：

```text
把 Excel 文件内容直接放入 Kafka
```

这是不推荐的。Kafka 消息体不应该过大。

推荐设计：

```text
1. 用户创建导出任务
2. 写 export_task 表
3. 发送 export-excel-topic，只传 taskId
4. 消费者查询任务参数
5. 分页查询数据生成 Excel
6. 上传对象存储
7. 更新下载地址
8. 通知用户
```

消息：

```json
{
  "taskId": "export_1001",
  "userId": "U001"
}
```

注意点：

```text
Kafka 不传大对象
导出任务要分页处理
避免一次性加载大量数据到内存
支持任务状态查询
支持失败重试
支持过期清理
```

------

# 四十四、订单状态流转

订单状态流转是 Kafka 最典型场景之一。

例如：

```text
ORDER_CREATED
ORDER_PAID
ORDER_SHIPPED
ORDER_FINISHED
ORDER_CANCELLED
```

设计方式一：每种事件一个 Topic。

```text
order-created-topic
order-paid-topic
order-shipped-topic
order-cancelled-topic
```

优点是消费者订阅清晰。

缺点是 Topic 多，跨事件顺序不好管理。

设计方式二：统一订单事件 Topic。

```text
order-event-topic
```

消息体：

```json
{
  "eventId": "evt_1001",
  "orderId": "order_1001",
  "eventType": "ORDER_PAID",
  "eventTime": "2026-06-18 10:00:00",
  "payload": {
    "amount": 99.9
  }
}
```

Producer 以 orderId 作为 key：

```java
producer.send(new ProducerRecord<>("order-event-topic", orderId, json));
```

这样同一个订单的事件会进入同一个 Partition，保证同订单有序。

消费端必须做状态机校验。

例如当前订单状态是 CREATED，收到 ORDER_PAID，可以处理。

当前订单状态是 INIT，收到 ORDER_SHIPPED，说明乱序或异常，不能直接处理。

状态流转可以用数据库条件更新保证幂等和合法性：

```sql
UPDATE orders
SET status = 'PAID'
WHERE order_id = 'order_1001'
AND status = 'CREATED';
```

如果影响行数为 0，说明状态不符合预期，可能是重复消息或乱序消息。

------

# 四十五、数据同步

Kafka 很适合做数据同步。

例如：

```text
用户服务修改用户信息
发送 UserUpdatedEvent
搜索服务同步 Elasticsearch
推荐服务同步用户画像
风控服务同步用户基础信息
```

或者使用 CDC：

```text
MySQL binlog -> Kafka -> 下游系统
```

常见技术链路：

```text
MySQL -> Debezium / Canal -> Kafka -> Elasticsearch / Flink / ClickHouse
```

Kafka 在这里作为数据管道。

注意点：

```text
消息顺序
重复事件
幂等更新
字段兼容
Schema 演进
补数据机制
消费延迟监控
```

如果是更新类事件，建议带上版本号：

```json
{
  "userId": "U001",
  "version": 12,
  "eventType": "USER_UPDATED",
  "data": {}
}
```

消费端只接受更高版本的数据，避免旧消息覆盖新数据。

```sql
UPDATE user_index
SET name = ?, version = ?
WHERE user_id = ?
AND version < ?;
```

------

# 四十六、日志采集

Kafka 在日志采集中非常常见。

典型链路：

```text
应用日志 -> Filebeat/Fluent Bit/Logstash -> Kafka -> Flink/Logstash -> Elasticsearch/ClickHouse/HDFS
```

Kafka 的作用是缓冲和解耦。

```text
应用不直接写 ES
日志先进入 Kafka
ES 慢了也不会直接影响应用
下游可以多个系统共同消费
```

日志场景通常吞吐大，但允许少量丢失，所以配置可以更偏性能。

例如：

```properties
acks=1
compression.type=lz4
linger.ms=10
batch.size=65536
```

但如果是审计日志、交易日志，可靠性要求更高，就要使用更强配置。

------

# 四十七、Kafka 中大消息问题

Kafka 不适合传输大消息。

默认情况下，Kafka 对单条消息大小有限制，虽然可以调大，但不建议滥用。

大消息会带来：

```text
Producer 缓冲区压力
Broker 网络压力
Page Cache 污染
Consumer 拉取慢
GC 压力
重试成本高
影响其他小消息延迟
```

推荐做法：

```text
文件、图片、Excel、报表放对象存储
Kafka 只传 URL、文件 ID、任务 ID、元数据
```

例如：

```json
{
  "fileId": "file_1001",
  "ossUrl": "https://xxx/report.xlsx",
  "taskId": "export_1001"
}
```

------

# 四十八、Kafka Schema 设计

Kafka 消息格式不能随便写，否则后期维护困难。

常见格式：

```text
JSON
Avro
Protobuf
```

Java 后端初期常用 JSON，简单易调试。

但大规模系统更推荐 Avro 或 Protobuf，因为：

```text
结构明确
体积更小
序列化更快
支持 Schema 演进
兼容性更好
```

一个通用事件结构可以设计成：

```json
{
  "eventId": "evt_1001",
  "eventType": "ORDER_CREATED",
  "eventTime": "2026-06-18T10:00:00",
  "source": "order-service",
  "version": "1.0",
  "traceId": "trace_abc",
  "key": "order_1001",
  "payload": {}
}
```

建议每条消息都包含：

```text
eventId：事件唯一 ID，用于幂等
eventType：事件类型
eventTime：事件发生时间
source：来源服务
version：消息版本
traceId：链路追踪
payload：业务数据
```

------

# 四十九、Topic 设计原则

Topic 设计不要过粗，也不要过细。

过粗：

```text
all-event-topic
```

所有业务都塞进去，会导致消费者过滤复杂，权限和治理困难。

过细：

```text
order-created-for-sms-topic
order-created-for-report-topic
order-created-for-point-topic
```

Topic 爆炸，维护困难。

比较合理的是按照业务领域和事件类型划分。

例如：

```text
order-event-topic
payment-event-topic
user-event-topic
inventory-event-topic
log-event-topic
```

或者：

```text
order-created-topic
order-paid-topic
order-cancelled-topic
```

选择依据：

如果消费者通常关心整个订单生命周期，可以用统一 `order-event-topic`。

如果消费者只关心单一事件，且事件量大，可以拆成多个 Topic。

如果需要同一业务实体的顺序，倾向使用统一 Topic + key。

------

# 五十、Partition 数设计

Partition 数决定了并行度，但不是越多越好。

Partition 太少：

```text
消费并行度不足
吞吐受限
扩容受限
```

Partition 太多：

```text
文件句柄增加
内存开销增加
Controller 管理压力增大
Rebalance 成本增加
故障恢复变慢
小文件变多
```

设计 Partition 数时考虑：

```text
目标吞吐量
消费者并行度
Broker 数量
消息顺序要求
未来扩容空间
```

例如一个 Topic 峰值需要 12 个消费者并行消费，则 Partition 至少应为 12。

但如果要求严格全局顺序，只能一个 Partition，吞吐就会受限。

实际系统中更常见是按业务 key 局部顺序，因此可以设置多个 Partition。

------

# 五十一、Consumer 并发处理模型

Kafka Consumer 不是线程安全的。一个 Consumer 实例通常只能在一个线程中 poll。

如果要提高处理能力，有几种方式。

第一，增加 Consumer 实例。

```text
多个进程或多个线程，每个线程一个 Consumer
```

这是最标准的方式。

第二，poll 线程拉取消息，业务线程池处理。

```text
Consumer poll -> 分发给 worker 线程池 -> 处理完成后提交 offset
```

这种方式复杂，因为 offset 提交必须保证已提交 offset 之前的消息都处理成功。

否则可能出现：

```text
offset 100 处理慢
offset 101 处理成功
提交 offset 102
服务宕机
offset 100 丢失
```

第三，按 Partition 分发处理。

每个 Partition 单独顺序处理，处理完成后提交该 Partition 的 offset。

这适合需要顺序的业务。

第四，按业务 key 分发到固定 worker。

```text
hash(key) % workerCount
```

保证同 key 串行，不同 key 并行。

------

# 五十二、Spring Boot 集成 Kafka

Java 后端常用 Spring Kafka。

Maven 依赖：

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

配置示例：

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092

    producer:
      acks: all
      retries: 10
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.apache.kafka.common.serialization.StringSerializer
      properties:
        enable.idempotence: true
        compression.type: snappy
        linger.ms: 5
        batch.size: 32768

    consumer:
      group-id: order-service-group
      enable-auto-commit: false
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      properties:
        max.poll.records: 100

    listener:
      ack-mode: manual
```

Producer 示例：

```java
@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, String> kafkaTemplate;

    public OrderEventProducer(KafkaTemplate<String, String> kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void sendOrderEvent(String orderId, String messageJson) {
        kafkaTemplate.send("order-event-topic", orderId, messageJson)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        // 生产环境：记录日志、告警、落库补偿
                        System.err.println("Send failed: " + ex.getMessage());
                    } else {
                        System.out.println("Send success, topic="
                                + result.getRecordMetadata().topic()
                                + ", partition="
                                + result.getRecordMetadata().partition()
                                + ", offset="
                                + result.getRecordMetadata().offset());
                    }
                });
    }
}
```

Consumer 示例：

```java
@Component
public class OrderEventConsumer {

    @KafkaListener(
            topics = "order-event-topic",
            groupId = "inventory-service-group"
    )
    public void consume(String message, Acknowledgment ack) {
        try {
            // 1. 反序列化消息
            // 2. 根据 eventId 做幂等判断
            // 3. 执行业务逻辑
            // 4. 业务成功后手动提交 offset
            System.out.println("Consume message: " + message);

            ack.acknowledge();
        } catch (Exception e) {
            // 不要直接吞异常
            // 可以记录失败、发送 retry topic 或 DLQ
            System.err.println("Consume failed: " + e.getMessage());
            throw e;
        }
    }
}
```

注意：实际生产中不要只 `System.out.println`，需要结构化日志、traceId、告警、DLQ。

------

# 五十三、手动提交 offset 的注意点

Spring Kafka 的 ack-mode 有多种：

```text
record
batch
time
count
manual
manual_immediate
```

常见核心业务使用 manual 或 manual_immediate。

如果你批量消费：

```java
@KafkaListener(topics = "order-event-topic")
public void consume(List<String> messages, Acknowledgment ack) {
    for (String message : messages) {
        handle(message);
    }
    ack.acknowledge();
}
```

要注意：只有全部消息处理成功后才能提交。

如果中间某条失败：

```text
前几条已经处理成功
中间一条失败
后面几条未处理
offset 未提交
下次会从这一批重新消费
```

所以处理成功的前几条可能重复消费。

这要求业务幂等。

------

# 五十四、Spring Kafka 中的重试和 DLQ

Spring Kafka 可以配置错误处理器，把失败消息发送到 DLT，也就是 Dead Letter Topic。

概念上是：

```text
主 Topic 消费失败
重试若干次
仍失败则发送到 DLT
```

DLT 命名通常类似：

```text
order-event-topic.DLT
```

实际业务中，是否使用框架自动 DLT，要看团队规范。有些团队更倾向自己实现 retry topic 和 dlq topic，因为可控性更强。

不管哪种方式，核心原则一样：

```text
失败不能无限阻塞主消费
失败原因要可观测
DLQ 要有告警和补偿机制
```

------

# 五十五、Kafka Stream 简介

Kafka 不只是 MQ，还支持流式处理。

Kafka Streams 是 Kafka 官方提供的 Java 流处理库。

它可以做：

```text
过滤
转换
聚合
窗口统计
Join
分组
状态存储
```

例如：

```text
订单事件流 -> 按用户统计 5 分钟内订单金额
用户行为流 -> 实时计算点击量
日志流 -> 实时告警
```

概念：

```text
KStream：事件流，每条记录都是独立事件
KTable：变更日志表，同一个 key 保留最新状态
GlobalKTable：每个实例持有全局表副本
```

例如：

```text
order_event_topic -> Kafka Streams -> order_stat_topic
```

普通 Java 后端面试中，不一定要求深入 Kafka Streams，但知道 Kafka 可以从消息队列扩展到流处理，会更完整。

------

# 五十六、Kafka Connect 简介

Kafka Connect 是 Kafka 的数据集成框架。

它用于把外部系统和 Kafka 连接起来。

常见 Source Connector：

```text
MySQL -> Kafka
PostgreSQL -> Kafka
MongoDB -> Kafka
日志文件 -> Kafka
```

常见 Sink Connector：

```text
Kafka -> Elasticsearch
Kafka -> ClickHouse
Kafka -> HDFS
Kafka -> S3
Kafka -> MySQL
```

例如 CDC 数据同步：

```text
MySQL binlog -> Debezium Connector -> Kafka -> Elasticsearch Sink
```

Java 后端只要知道它解决的是“数据接入和数据输出”的问题即可。

------

# 五十七、Kafka 安全

生产环境 Kafka 需要考虑安全。

常见内容：

```text
认证 Authentication
授权 Authorization
加密 Encryption
审计 Audit
```

认证方式：

```text
SASL/PLAIN
SASL/SCRAM
SSL
Kerberos
```

授权通常使用 ACL：

```text
某个用户可以写 order-event-topic
某个用户可以读 user-event-topic
某个用户不能删除 Topic
```

传输加密使用 SSL/TLS。

如果是公司内部系统，面试一般不会深入到安全配置，但知道 Kafka 不是默认完全安全的，生产环境需要认证、授权、加密。

------

# 五十八、Kafka 监控指标

Kafka 生产环境必须监控。

核心指标包括：

Producer：

```text
发送成功率
发送失败率
请求延迟
重试次数
batch size
buffer 可用空间
record error rate
```

Broker：

```text
Broker 存活
磁盘使用率
网络吞吐
请求处理延迟
Under Replicated Partitions
Offline Partitions
ISR 变化
Controller 状态
Page Cache 命中情况
```

Consumer：

```text
consumer lag
消费速率
处理耗时
commit 失败次数
rebalance 次数
poll 间隔
DLQ 数量
```

最重要的报警指标之一是：

```text
Consumer Lag
```

Lag 持续增长说明消费者处理不过来。

另一个重要指标是：

```text
Under Replicated Partitions
```

表示有分区副本不同步，可能影响可靠性。

------

# 五十九、Kafka 常见故障排查

消息堆积：

```text
看 lag
看消费者是否存活
看消费者处理耗时
看是否频繁 rebalance
看下游 DB/API 是否慢
看是否有坏消息卡住
```

消息重复：

```text
检查 offset 提交时机
检查消费者是否重启
检查是否发生 rebalance
检查生产者是否重试
检查业务是否缺少幂等
```

消息丢失：

```text
检查 acks
检查 retries
检查 callback
检查 min.insync.replicas
检查是否自动提交 offset
检查是否先提交后处理
```

消费不到消息：

```text
检查 topic 名称
检查 groupId
检查 offset 位置
检查 auto.offset.reset
检查消费者是否订阅成功
检查 ACL 权限
```

顺序错乱：

```text
检查 key 是否一致
检查是否增加过 partition
检查消费者是否线程池并发乱序
检查失败重试是否跳过前置消息
```

------

# 六十、Kafka CLI 常用命令

创建 Topic：

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create \
  --topic order-event-topic \
  --partitions 3 \
  --replication-factor 3
```

查看 Topic：

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

查看 Topic 详情：

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe \
  --topic order-event-topic
```

生产消息：

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --topic order-event-topic
```

消费消息：

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic order-event-topic \
  --from-beginning
```

查看消费者组：

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
```

查看消费进度：

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe \
  --group inventory-service-group
```

重置 offset：

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group inventory-service-group \
  --topic order-event-topic \
  --reset-offsets \
  --to-earliest \
  --execute
```

------

# 六十一、Kafka Docker Compose 本地环境

本地学习可以用 Docker Compose 搭 Kafka。不同镜像配置略有差异，下面给一个概念化配置，实际使用时按镜像文档调整。

```yaml
services:
  kafka:
    image: apache/kafka:latest
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@localhost:9093
      KAFKA_LOG_DIRS: /tmp/kraft-combined-logs
```

现在 Kafka 新版本可以使用 KRaft 模式，不再强依赖 ZooKeeper。学习时你需要知道：

```text
旧架构：Kafka + ZooKeeper 管理元数据
新架构：Kafka KRaft 自己管理元数据
```

Java 后端面试中通常不需要深入 KRaft Raft 细节，但要知道 Kafka 的集群元数据、Leader 选举、Controller 管理这些概念。

------

# 六十二、Kafka 在微服务中的事件设计

一个好的事件不是简单的“数据库表变化通知”，而是表达业务事实。

不推荐：

```text
order_table_updated
```

更推荐：

```text
OrderCreated
OrderPaid
OrderCancelled
OrderShipped
```

事件应该表示“已经发生的事实”，通常用过去式。

```text
UserRegistered
PaymentSucceeded
InventoryDeducted
CouponIssued
```

事件中应该携带足够的上下文，但不要携带过大的数据。

例如订单支付事件：

```json
{
  "eventId": "evt_1001",
  "eventType": "OrderPaid",
  "orderId": "order_1001",
  "userId": "user_001",
  "paymentId": "pay_1001",
  "amount": 99.9,
  "paidAt": "2026-06-18T10:00:00",
  "traceId": "trace_abc"
}
```

消费者拿到事件后，不一定需要再查订单服务，减少同步依赖。

但也不能把整个订单大对象全部塞进去。需要在“信息完整”和“消息轻量”之间平衡。

------

# 六十三、事件驱动架构 EDA

Kafka 常用于事件驱动架构。

传统命令式调用：

```text
订单服务调用库存服务扣库存
```

事件驱动：

```text
订单服务发布 OrderCreated
库存服务订阅事件后扣库存
```

区别在于：

```text
命令 Command：要求别人做什么
事件 Event：通知别人发生了什么
```

例如：

```text
Command: DeductInventory
Event: OrderCreated
```

在微服务中，事件更利于解耦。但如果业务需要立即返回明确结果，仍然需要同步调用。

所以不是所有场景都应该用 MQ。

适合 Kafka 的场景：

```text
异步任务
非核心链路
多下游订阅
高吞吐数据流
日志采集
最终一致性可接受
```

不适合 Kafka 的场景：

```text
需要立即强一致结果
请求响应式查询
低吞吐但复杂延迟调度
严格全局顺序
小系统中引入后复杂度大于收益
```

------

# 六十四、Kafka 和 HTTP/RPC 的关系

Kafka 不是用来完全替代 HTTP/RPC 的。

HTTP/RPC 适合：

```text
同步查询
立即返回结果
强交互
调用方需要知道成功失败
```

Kafka 适合：

```text
异步通知
事件广播
削峰填谷
流式数据
最终一致性
```

例如：

```text
查询用户余额：HTTP/RPC
用户注册后发短信：Kafka
支付前校验订单状态：HTTP/RPC
支付成功后通知积分系统：Kafka
```

系统设计时经常是二者结合，而不是二选一。

------

# 六十五、Kafka 面试高频题

问题一：Kafka 为什么快？

回答重点：

```text
顺序写磁盘、Page Cache、零拷贝、批量发送、压缩、分区并行。
```

问题二：Kafka 如何保证消息不丢？

回答重点：

```text
Producer 使用 acks=all、开启重试和幂等、处理 callback；
Broker 设置 replication.factor=3、min.insync.replicas=2、关闭 unclean leader election；
Consumer 关闭自动提交，业务处理成功后手动提交 offset；
失败消息进入重试和 DLQ。
```

问题三：Kafka 如何解决重复消费？

回答重点：

```text
Kafka 通常保证至少一次，因此重复消费正常。通过业务幂等解决，比如 eventId 唯一索引、消费记录表、Redis SETNX、状态机幂等、下游幂等号。
```

问题四：Kafka 如何保证顺序消费？

回答重点：

```text
Kafka 只保证单 Partition 内有序。需要用同一个业务 key，例如 orderId，发送到同一个 Partition；Consumer 不要并发乱序处理；失败重试不能跳过前置消息；如果增加 Partition，要注意 key 映射变化。
```

问题五：Kafka 消息堆积怎么办？

回答重点：

```text
先看 lag 和消费耗时，判断是消费者慢、下游慢、Partition 不够、坏消息卡住还是频繁 rebalance。解决方案包括增加消费者、增加 Partition、优化业务逻辑、批处理、限流下游、失败进 DLQ、调整 poll 参数。
```

问题六：Kafka 的 offset 是什么？

回答重点：

```text
offset 是消息在 Partition 内的位置。每个 Consumer Group 独立维护自己的 offset。Kafka 把 offset 存在 __consumer_offsets 中。
```

问题七：Kafka 的消费者组是什么？

回答重点：

```text
同一个消费者组内多个消费者共同消费 Topic，一个 Partition 同一时刻只能被组内一个消费者消费。不同消费者组之间相互独立，相当于广播订阅。
```

问题八：Kafka 如何实现延迟队列？

回答重点：

```text
Kafka 原生不适合任意延迟消息。可通过延迟 Topic、消息 executeTime、数据库任务表、时间轮实现。要注意队头阻塞、重试、幂等和 DLQ。
```

问题九：Kafka 事务能解决 MySQL + Kafka 的一致性吗？

回答重点：

```text
Kafka 事务主要解决 Kafka 内部多消息、多分区以及消费 offset 与输出消息的原子性，不能直接把 MySQL 事务也纳入同一事务。MySQL + Kafka 更常用本地消息表、Outbox Pattern、补偿和对账。
```

问题十：Kafka 中 acks=all 是否一定不丢消息？

回答重点：

```text
不一定。acks=all 还需要配合 replication.factor、min.insync.replicas、unclean.leader.election.enable=false、Producer 重试和正确处理 callback。Consumer 端也要正确提交 offset。
```

------

# 六十六、Kafka 学习优先级

按照 Java 后端岗位要求，建议按这个顺序掌握：

第一层，基础概念：

```text
Producer
Consumer
Broker
Topic
Partition
Offset
Consumer Group
Replica
Leader/Follower
ISR
```

第二层，可靠性：

```text
acks
retries
idempotence
replication.factor
min.insync.replicas
offset commit
手动提交
消息丢失
重复消费
幂等
```

第三层，业务能力：

```text
异步任务
削峰填谷
系统解耦
最终一致性
重试 Topic
DLQ
延迟任务
本地消息表
Outbox Pattern
```

第四层，性能与运维：

```text
批量发送
压缩
linger.ms
batch.size
Consumer Lag
消息堆积
Rebalance
Partition 设计
监控告警
```

第五层，进阶：

```text
Kafka 事务
Exactly Once
Kafka Streams
Kafka Connect
Schema Registry
CDC
KRaft
安全认证授权
```

------

# 六十七、总结

Kafka 的核心不是“会发消息、会收消息”，而是理解它在分布式系统中的作用。

你需要建立这套思维：

```text
Producer 负责可靠发送
Broker 负责持久化和高可用
Topic/Partition 负责分类与并行
Consumer Group 负责负载均衡
Offset 负责消费进度
acks/ISR/副本 负责生产端可靠性
手动提交/幂等/DLQ 负责消费端可靠性
重试/补偿/本地消息表 负责最终一致性
监控 lag 负责生产可观测性
```

对于 Java 后端面试，Kafka 最重要的不是 API，而是这些问题：

```text
为什么要用 MQ？
Kafka 如何削峰填谷？
Kafka 如何解耦？
Kafka 如何保证消息不丢？
Kafka 如何处理重复消费？
Kafka 如何保证顺序？
Kafka 如何做失败重试和死信队列？
Kafka 如何处理消息堆积？
Kafka 如何实现最终一致性？
Kafka 和数据库事务如何配合？
```

掌握这些，基本就覆盖了 Kafka 在 Java 后端岗位中的核心要求。