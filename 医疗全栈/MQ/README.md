# 1. MQ

MQ是消息队列。

如果没有MQ，那么前端需要执行后端服务是，需要阻塞等待后端服务处理完毕，前端才能继续执行。如果服务执行较慢，或者中间服务挂了，整个流程会失败，导致前端体验不好。

有了MQ，前端需要后端服务时，后端可以将请求封装成消息，放到消息队列后即可返回前端“处理中”的信息。后端获取消息后逐步执行，前端可以持续监听，或者通过WebSocket来监听执行情况。

# 2. 业务场景

1. 异步发送短信。注册成功后需要发送验证码、登录提醒、订单通知等，这些非核心业务不应该影响主线程。
2. 异步生成报表。这种涉及大量SQL，不应该同步执行。
3. 数据同步。用户数据修改后，同步到数据库时应使用消息队列实现。
4. 削峰。消息队列能够缓存业务操作，后端根据自身需求来控制执行速度，不会有高峰压力。

但引入了MQ也会出现问题。会有消息丢失、重复消费、顺序消费、消息堆积、延迟、最终一致性等问题。所以MQ适合允许短暂不一致的异步业务，不适合强一致性，必须立刻要结果的业务。

# 3. 基本角色

MQ主要有三类角色。

```
Producer -> Broker -> Consumer
```

Producer是生产者，发送消息。

Consumer是消费者，处理消息。

Broker是消息服务器，负责存储、接收、转发消息。

Message是消息本身，包含消息体、消息ID、Topic等。

Topic是消息的分类，生产者需要将消息发送到某个Topic，消费者从某个Topic中消费消息。

Consumer Group是消费者组，同一组内多个消费者共同消费一份信息。

# 4. RabbitMQ

RabbitMQ的流程如下。

```
Producer -> Exchange -> Queue -> Consumer
```

生产者将消息发送给Exchange，Exchange根据路由规则将消息发送到指定的Queue。消费者监听Queue，从Queue中获取消息。

## 1. Exchange类型

Exchange是交换机，常见的有四种。

1. Direct Exchange。

直接交换机中，需要指定当前交换机的名字，然后绑定队列时，需要指定Routing Key，这样后续发送消息时指定交换机名字和Routing Key，就能发送到对应的交换机的队列。

```java
@Configuration
public class DirectExchangeConfig {

    // 1. 声明Direct交换机
    @Bean
    public DirectExchange directLogExchange() {
        return new DirectExchange("direct.log.exchange", 
                true,  // durable: 持久化
                false  // autoDelete: 不自动删除
        );
    }

    // 2. 声明两个队列
    @Bean
    public Queue errorQueue() {
        return new Queue("error.queue", true); // 持久化队列
    }

    @Bean
    public Queue infoQueue() {
        return new Queue("info.queue", true);
    }

    // 3. 绑定队列到交换机（指定Routing Key）
    @Bean
    public Binding errorBinding() {
        return BindingBuilder
                .bind(errorQueue())
                .to(directLogExchange())
                .with("error");  // Routing Key = "error"
    }

    @Bean
    public Binding infoBinding() {
        return BindingBuilder
                .bind(infoQueue())
                .to(directLogExchange())
                .with("info");
    }

    // 也可以让warn也进入info队列（多重绑定）
    @Bean
    public Binding warnBinding() {
        return BindingBuilder
                .bind(infoQueue())   // 注意：绑定到infoQueue
                .to(directLogExchange())
                .with("warn");
    }
}
```

```java
@Autowired
    private RabbitTemplate rabbitTemplate;

    // 发送error级别日志（只会被error.queue消费）
    public void sendErrorLog(String message) {
        rabbitTemplate.convertAndSend(
                "direct.log.exchange",  // 交换机名称
                "error",                // Routing Key
                "[ERROR] " + message    // 消息体
        );
        System.out.println("发送Error日志: " + message);
    }
```

```java
@Component
public class LogConsumer {

    // 监听error队列（邮件告警）
    @RabbitListener(queues = "error.queue")
    public void handleErrorLog(String message) {
        System.out.println("【告警邮件】收到错误日志: " + message);
        // 实际业务：发送邮件、钉钉通知等
    }

    // 监听info队列（存储数据库）
    @RabbitListener(queues = "info.queue")
    public void handleInfoLog(String message) {
        System.out.println("【数据库存储】收到Info/Warn日志: " + message);
        // 实际业务：插入Elasticsearch或MySQL
    }
}
```

也就是说，需要先定义交换机，定义队列，通过Bind将交换机和队列通过Routing Key绑定起来，生产者通过`convertAndSend(exchange_name, routing_key, message)`来生产消息，消费者通过`@RabbitListener(queues = routing_key)`来监听消息。

2. Fanout Exchange。

广播交换机。不需要Routing Key，而是会将消息广播到每一个队列。

```java
@Configuration
public class FanoutExchangeConfig {

    // 1. 声明Fanout交换机（注意：广播模式，Routing Key被忽略）
    @Bean
    public FanoutExchange fanoutUserExchange() {
        return new FanoutExchange("fanout.user.exchange", 
                true,   // durable
                false   // autoDelete
        );
    }

    // 2. 声明多个队列（每个服务独立队列）
    @Bean
    public Queue smsQueue() {
        return new Queue("sms.queue", true);
    }

    @Bean
    public Queue emailQueue() {
        return new Queue("email.queue", true);
    }

    @Bean
    public Queue cacheQueue() {
        return new Queue("cache.queue", true);
    }

    @Bean
    public Queue couponQueue() {
        return new Queue("coupon.queue", true);
    }

    // 3. 绑定队列到Fanout交换机（注意：不需要with()指定Routing Key）
    @Bean
    public Binding smsBinding() {
        return BindingBuilder
                .bind(smsQueue())
                .to(fanoutUserExchange());
    }

    @Bean
    public Binding emailBinding() {
        return BindingBuilder
                .bind(emailQueue())
                .to(fanoutUserExchange());
    }

    @Bean
    public Binding cacheBinding() {
        return BindingBuilder
                .bind(cacheQueue())
                .to(fanoutUserExchange());
    }

    @Bean
    public Binding couponBinding() {
        return BindingBuilder
                .bind(couponQueue())
                .to(fanoutUserExchange());
    }
}
```

```java
 @Autowired
    private RabbitTemplate rabbitTemplate;

    public void sendUserRegisterEvent(Long userId, String username) {
        // 构造用户注册消息（JSON格式）
        String message = String.format(
                "{\"userId\": %d, \"username\": \"%s\", \"timestamp\": %d}",
                userId, username, System.currentTimeMillis()
        );

        // 注意：Fanout交换机忽略Routing Key，传空字符串即可
        rabbitTemplate.convertAndSend(
                "fanout.user.exchange",  // 交换机名称
                "",                      // Routing Key（被忽略）
                message
        );
        
        System.out.println("【广播】用户注册事件已发送: " + username);
    }
```

```java
@Component
public class UserRegisterConsumers {

    // 短信服务
    @RabbitListener(queues = "sms.queue")
    public void handleSms(String message) {
        System.out.println("【短信服务】发送注册验证短信: " + message);
        // 调用短信网关
    }

    // 邮件服务
    @RabbitListener(queues = "email.queue")
    public void handleEmail(String message) {
        System.out.println("【邮件服务】发送欢迎邮件: " + message);
        // 调用邮件服务
    }

    // 缓存刷新服务
    @RabbitListener(queues = "cache.queue")
    public void handleCache(String message) {
        System.out.println("【缓存服务】刷新用户缓存: " + message);
        // 更新Redis缓存
    }

    // 优惠券服务
    @RabbitListener(queues = "coupon.queue")
    public void handleCoupon(String message) {
        System.out.println("【优惠券服务】发放新用户优惠券: " + message);
        // 发放优惠券
    }
}
```

这样，只要生产者发送消息，所有消费者都能够获取消息。

3. Topic Exchange。

主题交换机，支持通配符匹配，一条消息可以发送到指定的队列中。

`*`表示匹配一个单词，`#`表示匹配0个或任意多个单词。

```java
@Bean
public Queue orderQueue() {
    return new Queue("order.service.queue", true);
}

// 支付服务队列（处理所有支付相关）
@Bean
public Queue paymentQueue() {
    return new Queue("payment.service.queue", true);
}

// 数据仓库队列（归档所有消息）
@Bean
public Queue archiveQueue() {
    return new Queue("archive.queue", true);
}
// 中国区：匹配 china.*.* 或 china.#
@Bean
public Binding chinaBinding() {
    return BindingBuilder
            .bind(chinaQueue())
            .to(topicOrderExchange())
            .with("china.#");  // 匹配china开头所有消息
}

// 订单服务：匹配 *.order.* 或 #.order.#
@Bean
public Binding orderBinding() {
    return BindingBuilder
            .bind(orderQueue())
            .to(topicOrderExchange())
            .with("*.order.*");  // 匹配任何地域的订单消息
}

// 支付服务：匹配 *.payment.* 或 #.payment.#
@Bean
public Binding paymentBinding() {
    return BindingBuilder
            .bind(paymentQueue())
            .to(topicOrderExchange())
            .with("*.payment.*");
}

// 数据仓库：匹配所有消息（#）
@Bean
public Binding archiveBinding() {
    return BindingBuilder
            .bind(archiveQueue())
            .to(topicOrderExchange())
            .with("#");  // 匹配所有Routing Key
}
```

在绑定的时候，通过Routing Key指定来进行匹配。

比如上面`china.payment.queue`能够匹配到`chinaBinding`，`paymentBinding`，`archiveBinding`三个队列。那么消费者监听这三个队列即可。

## 2. 确认机制

消息确认主要有两个，生产者确认和消费者确认。

生产者需要知道消息到了MQ，消费者确认需要保证消息被成功处理。

生产者确认包括Confirm和Return。

Confirm机制表示消息是否成功到达交换机，并被Broker接收。

Return表示消息到达Exchange后如果无法路由到任何队列，会将消息返回。

暂不考虑RabbitMQ。

# 5. Kafka

Kafka是分布式提交日志。

传统队列是消息被消费后就删除，而Kafka是消息追加到日志文件，消费者通过offset记录读到了哪里。

```
Topic: order_event

offset=0  订单1创建
offset=1  订单2创建
offset=2  订单3支付成功
offset=3  订单4取消
```

不同消费者组的消费者的读取位置不一致，互不影响。

## 1. 组件

```
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

## 2. 流程

首先Producer生产消息。

消息经过Cluster分配到指定Broker，然后交到这个Broker下对应的Topic。Topic下又需要选择分区，将消息存储在该分区下。

消费者根据自己的topic、group.id、offset来消费消息。

## 3. 生产者

消息通常包含下面信息。

```
topic
partition
key
value
headers
timestamp
offset
```

其中value包含业务数据。

```json
{
  "orderId": "1001",
  "userId": "U001",
  "status": "CREATED",
  "amount": 99.9
}
```

而key决定发往哪个分区。

```
partition = hash(key) % partition_count
```

如果根据id来作为key，那么同一个id的所有事件都会进入同一个partition保证顺序。

如果key为null，那么就会采用粘性分区策略。

粘性分区：随机选择一个分区，将后续消息发往这个分区，直到消息批次被填满或者达到发送时间。

而topic是消息的主题，用于分类。

Producer将消息发送到Topic中，Consumer从Topic订阅消息。

而Topic不是物理结构，它会被拆分成多个Partition。

## 4. 分区

一个Topic可以有多个分区。

```
Topic: order_event

Partition 0: offset 0, 1, 2, 3 ...
Partition 1: offset 0, 1, 2, 3 ...
Partition 2: offset 0, 1, 2, 3 ...
```

Partition能够提高吞吐量。消息能够装入多个Partition中，让多个服务监听多个Partition。

同时，Partition能够保存到不同的Broker中，实现水平拓展。

由于Partition的原因，连续的消息不会连续存放到某一个Partition中，但一个Partition中能够保证按顺序存储从Topic中读取的消息。也就是Partition能够保证内部有序，但不能保证全局有序。

如果有事件A和事件B按顺序进入Topic，分到不同的Partition中，可能会出现先完成B，再完成A的情况。但分到同一个Parititon中就不会出现这种情况。

## 5. Broker

Broker是Kafka中的服务器节点。

每个Broker中保存多个Topic的多个Partition。

```
Broker 1: order_event-0, log_event-2
Broker 2: order_event-1, user_event-0
Broker 3: order_event-2, log_event-1
```

## 6. 吞吐量大

Kafka以吞吐量大而著称。

1. 顺序读写。传统IO读写慢的原因是数据随机存储，导致读取的时候采用随机读写的方法。但Kafka的分区是物理文件，新消息会加到文件的末尾，保证读取的时候能够按顺序读取数据。
2. 零拷贝。传统IO需要经过`磁盘-内核缓冲区-应用程序内存-Socket网络缓冲区-网卡`，出现了多次拷贝。而Kafka只需要`磁盘-内核缓冲区-网卡`，用了直接内存。减少了两次拷贝和上下文切换。
3. 批量发送。Producer发送消息的时候，不是一有消息就传，而是在内存中存储一定批量后再一次性发送，减少了网络开销。
4. 压缩。因为消息按顺序存储在批次中，压缩的时候效率很高。
5. 分区并行。一个Topic拆成多个Partition，导致消息发送的时候，不同的Partition可以并行处理。

## 7. Replica

为了防止Broker宕机导致消息丢失，Kafka支持副本。

假如Topic有三个分区，每个分区的副本为3。

```
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

每个Partition有一个Leader和多个Follower。

producer只会向Leader写入，Consumer也会向Leader读取，Follower从Leader中同步数据。

如果Leader宕机，那么Follower会选举出新的Leader。

这里有HW概念来保证数据一致性。

HW是指确保消息一致性的位置。

Follower会持续向Leader发起请求来获取新消息。

Leader会跟踪每个Follower的同步进度，有新消息的时候先写入，然后所有的Follower写入成功，自己才会更新HW。

为了数据一致性，消费者只能读取HW以下的消息。新消息如果Leader未确认Follower已写入，是无法读取的。

## 8. ISR

ISR是同步副本集合，只有在ISR里面的Follower才能选举为Leader。

如果Follower因为网络等原因落后太多，Leader会将这个Follower踢出ISR。

## 9. 容灾

如果Leader所在的Broker宕机，那么副本机制会立刻启动容灾。

1. 控制器检测到Leader不可用。
2. 从ISR中选取Follower，提升为新的leader。
3. 旧Leader恢复后，发现自己成为Follower，那么会自动找到新的Leader，同步自己的数据。

## 10. 消息确认

为了防止消息丢失，发送消息后需要进行确认，通过acks来控制。

如果acks设置为0，表示不需要确认。

acks=1，表示Leader写入成功后确认。

acks=all或者-1，表示Leader和ISR足够的副本确认后才返回成功。

## 11. 一般配置

```
acks=all   # 必须等待ISR所有副本同步后才返回确认
enable.idempotence=true # 开启幂等，生产者重复发送消息时，会根据消息的PID去重
retries=Integer.MAX_VALUE # 发送失败，无限重试
max.in.flight.requests.per.connection<=5 # 限制单个连接中最多5个请求未确认
min.insync.replicas=2 # 每个分区在3台Broker中保存副本，也就是共4个
replication.factor=3 # 消息至少有3个Broker写入，才能返回确认
```

> 这里的replication.factor是用于保证副本数量过少导致不稳定，需要一定数量的副本，才能保证数据不丢失。

## 12. 流程

1. 首先是业务调用send方法，生产者发送消息。此时会返回Future对象，能够获取结果。
2. 对消息进行序列化，获取key和value。
3. 根据Topic和key选择需要发送的Partition。
4. 消息首先会在内存中累积，不同Partition的消息放到不同的内存累加器RecordAccumulator中，这个结构是双端队列。
5. Sender线程不断轮询RecordAccumulator，找到攒满或者超时的批次，取出并发送。按照发送的Broker来进行分组，封装成ProduceRequest请求发送到指定Broker。
6. 发送后，Broker需要写入Leader的分区日志。根据acks来决定是否返回确认。并且Follower会拉取最新信息，写入到副本中，写入成功后，Leader就会更新HW。
7. 如果发送时注册了回调函数，那么Sender收到Broker确认后，会触发回调。如果没有回调，可以通过Future对象的get来获取结果。

## 13. 消费者

Kafka中，消费者需要从Topic中拉取消息。这能够保证消费者不会被大量消息涌入而失去可用性。

```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(1000));
    for (ConsumerRecord<String, String> record : records) {
        // 处理消息
    }
}
```

消费者通常会通过while循环来读取消息。

## 14. 消费者组

同一个消费者组下的消费者共同消费同一个Topic。

通过消费者组，可以实现负载均衡和广播。

1. 负载均衡

也就是队列模式，如一个订单只需要被处理一次，那么就会从消费者组中挑选一个实例来处理。

2. 广播

也就是发布订阅模式，如订单消息要给大数据平台，又要记录日志等。让不同系统使用不同的消费者组，每个组都能收到Topic的所有消息。

默认情况下，消费者必须配置消费者组id。

## 15. Offset

Offset是消息在Partition的位置，每个Partition的offset独立递增。

Consumer通过offset记录自己消费到哪里。

不同消费者组的offset互不影响。

由于同一消费者组里的多个消费者需要同步消息消费位置，所以这个offset不能保存在消费者内部，于是Kafka将消费者组的offset存储到内部Topic中。

也就是说，消费者在读取消息之前，会自动调用fetchCoordinator和fetchOffset，首先查找到自己组的协调者，协调者会去名为`_consumer_offsets`中，根据自己的Topic、group-id、Partition中获取offset，然后才根据这个offset，去消息队列中查看并消费消息。

这个`_consumer_offsets`保存的是消费者提交的偏移量。消费者内部维护一个ConcurrentOffset变量，每次poll后更新。这个ConcurrentOffset会通过commitSync或者自动提交写入`_consumer_offsets`中，消费者读取`_consumer_offsets`的上一次offset。而拉取消息后，ConcurrentOffset会自动更新为下一次Offset。

## 16. 自动提交和手动提交

自动提交比较简单。

```
enable.auto.commit=true
auto.commit.interval.ms=5000
```

> 每5秒自动提交当前poll的最大Offset。

如果拉取消息后立即提交了offset，但处理过程中宕机，恢复后重新拉取，只能拉取到下一条消息。

如果拉取消息处理完后宕机，恢复后向`_consumer_offsets`拉取消息，此时因为未提交，所以拉取到的是上一次的offset，导致消息重复消费。

手动提交能够控制offset的提交时间。

```
enable.auto.commit=true
```

消费者处理完消息后，调用`consumer.commitSync()`就能提交Offset。

优点是可靠性高，消息处理成功才提交。并且可以控制提交频率，每1000条提交一次等，能够提高可靠性。

缺点是代码复杂度提高，需要处理提交失败的重试、异常处理等。如果频繁提交，会增加网络开销。

## 17. 消息语义/

消息语义指消息在生产、存储、消费的过程中，系统对消息可靠性的保证程度。

1. 最多一次(At-most-once)。

消息可能被处理一次，也可能一次都不处理。这里生产者使用`acks=0`不做确认，消费者拉取消息后、处理业务前提交offset来实现。

如果发生异常，消息不可再次消费。

2. 至少一次(At-least-once)。

默认语义，消息至少处理一次，不会丢失，但可能被重复处理多次。

生产者使用`acks=1`或者`acks=all`，确保写入成功，消费者不设置自动提交，在业务处理完成后再手动提交。

这能够确保数据不丢，但只适合业务本身是幂等的情况。

3. 精确一次(Exactily-once)。

每条消息保证被处理且仅被处理一次。

生产者设置`enable.idempotence=true, acks=all, max.in.flight.requests.per.connection <= 5`。消费者配置事务。

适合不幂等且不允许重复或丢失的业务。

## 18. 重复消费

重复消费有很多种情况。

* 消费成功但offset没提交。
* 消费车处理超时并且Rebalance，新消费者会出现重复消费的问题。
* 手动设置offset。
* 重试机制导致重复消费。
* Producer重试导致重复发送。

因此，业务需要设置幂等。

1. 可以在消息中设置唯一id。例如订单消息中，会有orderId。消费者处理消息时，先看id是否已经被处理过。
2. 利用Redis去重。可以通过`SETNX consume:order_event:event_1001 1`，只有设置成功才处理。
3. 使用精确一次语义。通过kafka事务，将offset提交和业务操作放到事务中。确保消息提交的时候，业务操作已执行。

## 19. 顺序消费

Kafka只保证分区内有序，不保证跨分区有序。

但如果Topic增加了Partition，Key到Partition的映射可能发生变化，后续消息可能进入新的Partition。

如果消费者把消息丢到线程池处理，也会影响排序。

如果不注意顺序，其中有消息执行失败，可能会破坏业务。

```
订单修改为已支付
订单创建失败
```

> 上面就是一种乱序的情况，导致业务失败。

1. 将业务唯一标识作为消息的Key。保证同一个Key的消息永远进入同一个Partition。
2. 只使用单Paritition。
3. 消费者应该保证Partition内单线程处理来保证顺序。

## 20. 消息丢失

消息丢失发生在Producer丢失、Broker丢失和Consumer丢失三个场景。

1. Producer丢失

主要原因有acks=0，发送失败但没有重试，异步发送没有检查回调，buffer满了导致消息被丢弃等。

解决方法如下。

```
acks=all
retries=Integer.MAX_VALUE
enable.idempotence=true
delivery.timeout.ms=120000
request.timeout.ms=30000
min.insync.replicas=2
```

> 需要保证所有副本写入，并至少有两个副本确认，开启幂等，失败后无限重试等机制。

2. Broker丢失

主要有两个原因。

`unclean.leader.election.enbale=true`导致落后的Follower也能选为新的Leader，或者副本数太少，导致出现所有副本损坏的情况。

或者消息更新到磁盘时就宕机。可以设置刷新频率。

3. Consumer丢失

最主要的原因是自动提交，但业务处理失败。导致失败的消息无法被重复读取。需要关闭自动提交，在业务成功后手动提交。

还有异步处理的时候，没有对可能的异常做处理，导致失败消息没有被重试。丢失的消息可以重试或写入死信队列。

## 21. 失败重试

常见的重试方法有三种。

1. Consumer本地重试。达到次数后发送到死信队列。

```java
Map<String, Integer> retryCountMap = new HashMap<>();

public void processWithRetry(ConsumerRecord record) {
    String key = record.topic() + "-" + record.partition() + "-" + record.offset();
    int retryCount = retryCountMap.getOrDefault(key, 0);
    
    try {
        doBusiness(record);
        consumer.commitSync();
        retryCountMap.remove(key); // 成功则移除计数
    } catch (Exception e) {
        if (retryCount < MAX_RETRIES) {
            retryCountMap.put(key, retryCount + 1);
            // 不提交位移，等待下次 poll 重试
        } else {
            // 超过重试次数，发送到死信队列
            deadLetterProducer.send(deadLetterTopic, record);
            consumer.commitSync(); // 提交位移，跳过这条坏消息
            retryCountMap.remove(key);
        }
    }
}
```

2. 数据库任务表重试。

如果特别重要，可以将失败消息存放到数据库，由定时任务进行重试。

## 22. 死信队列

DeadLetterQueue是存放最终失败消息的Topic。本质上是一个普通的Topic。

死信队列的信息包含topic、partition、offset、key、body、失败原因、堆栈、重试次数、失败时间、消费者组等信息。

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

死信队列可以被监控，如果存有消息，或者消息堆积，就会告警。

## 23. 延迟队列

Kafka本身没有复杂的延迟队列。本身是将消息按offset的顺序进行消费，不支持对消息设置投递时间。

但可以设置多种方式。

1. 设置延迟Topic。

针对特定的Topic，如果消费者要消费就需要看时间，如果时间未到，就暂停处理。

2. 消息携带执行时间。

消息中可以设置`"executeTime"`，消费者收到消息时如果发现时间未到，可以等待。但可能会导致后续执行时间较前的消息无法准时消费。

3. 数据库延时任务表。

消息应该先保存到数据库，并配备定时任务轮询数据库，选取到达执行时间的任务再放到消息队列。

```sql
SELECT * FROM delay_task
WHERE status = 'INIT'
AND execute_time <= NOW()
LIMIT 100;
```

4. 时间轮。

消费者获取消息后，放到本地的DelayQueue，设置好延迟时间，到了再处理。或者生产者生产消息后放到本地的DelayQueue，设置延迟时间，到了再投递。

## 24. 消息堆积

消息堆积是指生产者不断生产消息，而消费者的消费速度较慢，导致队列堆积大量消息，最终导致服务不可用。

通常使用lag来表示堆积量。

```
lag = log end offset - committed offset
```

主要原因有5个。

* 消费者处理太慢。
* 消费者实例太少。
* 消费者出现故障。
* 生产者突发流量。
* 消费者组经常变化，导致分区重分配，消费效率低。

查看堆积原因可以首先看Lag，如果不断增大，说明堆积在恶化。然后监控Topic的`MessagesInperSec`和`MessagesOutPerSec`，如果生产速率持续大于消费速率，说明消费者性能低。接下来就查看消费者的线程状态，看是否有死锁，或者调用API出现网络问题等。

消息堆积有多种解决方法。

1. 添加消费者。能够提升消费能力。
2. 添加分区。分区多能够保证消息被并行执行。
3. 优化消费逻辑。减少消息处理时间。
4. 降级处理。跳过非核心消息，或者写入死信队列。
5. 限流、熔断、降级。

## 25. MQ使用场景

1. 削峰填谷。

如果没有MQ，商品流程如下。

```
秒杀请求-订单服务-库存服务-数据库
```

这样会导致流量直接打到缓存和数据库。

使用MQ后如下。

```
秒杀请求-写入MQ-返回
```

请求写入MQ后，消费者可以根据自己的能力来处理业务。

2. 解耦。

生产者不一定知道有哪些消费者。

如订单服务不关心谁消费，而且消费者上线下线不影响订单服务。但如果引入了MQ，这里的业务就会变成最终一致性，有短暂的数据不一致。

## 26. 本地消息表

本地方法表是用来解决业务处理成功，但消息发送失败的情况。

流程如下。

1. 在业务处理的时候，业务数据库中创建本地消息表。业务操作执行的时候，顺便将消息写入到这个消息表。如果业务失败，消息不会写入。业务成功，消息必然写入。
2. 后台启动定时任务，不断扫描本地消息表，获取消息数据并发送到消息队列。
3. 这样，就能够保证消息执行后一定会被记录，并且向下游发送消息的时候，能够保证这个消息已持久化，能够重发。

## 27. Kafka事务

事务能够将多个消息写入多个Topic和Partition，保证为原子操作。

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

这能够保证多消息之间的原子性。

```java
// 消费者配置（需要设置为读已提交）
props.put("isolation.level", "read_committed");

// 事务性生产者和消费者结合
producer.beginTransaction();
try {
    // 1. 从 Kafka 消费消息
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    
    // 2. 业务处理（如计算、转换）
    for (ConsumerRecord<String, String> record : records) {
        String result = process(record);
        // 3. 发送结果消息
        producer.send(new ProducerRecord<>("result-topic", result));
    }
    
    // 4. 提交消费位移
    producer.sendOffsetsToTransaction(
        getOffsets(records), 
        consumer.groupMetadata()
    );
    
    // 5. 提交事务
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

消费者也一样，能够保证从消费消息、业务处理到提交位移的原子操作。

## 28. Producer幂等

Producer可以通过`enable.idempotence=true`来开启幂等。

生产者发送消息的时候，会带上(PID, TopicPartition, SequenceNumber)。Broker收到消息后，会检查Partition的最新SequenceNumber。

如果SequenceNumber = 最新SequenceNumber，说明当前是新消息，直接写入。

如果SequenceNumber <= 最新SequenceNumber，认为是重复消息，直接返回成功。

如果SequenceNumber > 最新SequenceNumber，说明中间有消息丢失，抛出`OutOfOrderSequenceException`。

## 29. 消息存储机制

Kafka会将每个Partition存储为日志段文件。

```
topic-order-event/
	partition-0/
		00000000000000000000.log
        00000000000000000000.index
        00000000000000000000.timeindex

        00000000000000100000.log
        00000000000000100000.index
        00000000000000100000.timeindex
```

> .log是消息数据文件，存储消息的物理内容。.index是偏移量索引文件，记录Offset到物理位置的映射。.timeindex保存Timestamp到Offset的映射。

上面显示了两个段。第一个消息的文件名是`00000000000000000000`，表示这个段从Offset 0开始。如果段大小达到指定值，或者达到存活时间，就会从新的段开始。如上面第二个的文件`00000000000000100000`。

Kafka写入是从.log文件的末尾添加新的消息，没有随机IO，所以磁盘吞吐量大。并且会通过index找到近似位置再找，速度很快。

## 30. 消息保留与删除

Kafka消息不会因为消费成功就删除，而是有保留策略。

```
log.retention.hours=168
log.retention.bytes=1073741824
```

> 可以通过时间，168表示7天后删除。或者通过空间，达到一定大小后删除。

还有日志压缩。

通过`log.cleanup.policy`控制，有两种策略。

1. 删除。超过保留时间或大小，直接删除.log文件。
2. 压缩。对同一个Key，只保留最新的消息，旧版本会被清理。

此外，删除消息的单位是段，后台的Cleaner线程会定期检查，如果某个段的最晚时间超出了配置的保留策略时间，就会将整个段删除。

## 31. 消费者重平衡

Rebalance是消费者组内部，组内成员发生变化的时候，需要重新分配Topic给消费者的过程。例如消费者增加或者减少，分区数变化，消费者订阅的Topic发生变化时，都需要Rebalance。

1. 流程

首先，所有消费者都要暂停拉取消息，向协调者发送JoinGroup请求。

协调者从所有消费者中选举Leader。

Leader根据配置的策略，计算每个消费者负责的分区。

Leader通过SyncGroup将请求方案发送给协调者，协调者发送给所有消费者。

消费者收到分配结果，从新分区拉取消息。

2. 分区分配策略

Leader有多种分配分区的策略。

* Range。按分区号连续分段分配。
* RoundRobin轮询。将所有分区循环分配给消费者。
* Sticky。保持均匀的同时，不移动分配好的分区。
* Cooperative。分量阶段完成Rebalance，先撤销部分分区，再分配新分区。

Range首先会计算每个Topic的分区总数和消费者总数，得到平均分区数。将分区按照序号排序，消费者按照序号，拿起平均分区数的分区。

RoundRobin是轮询，将所有Partition进行排序，将排序后的Partition轮流分给消费者，直到分配完毕。

Sticky是保证分配均匀的情况下，保留消费者已经拥有的分区。如果消费者增减导致不均衡，将必须移动的分区从负载重的消费者移动到新消费者。

Cooperative Sticky不要求所有消费者同一时刻停止消费。

首先协调七计算需要撤销的分区，通知消费者释放分区。期间，消费者依然能够消费未撤销的分区。然后被释放的分区会进入分配池，协调器将它们分配给目标消费者。

## 32. 一致性

首先订单服务执行业务，然后发送消息通知库存服务。

```java
@Transactional
public void createOrder(Order order) {
    orderMapper.insert(order);
    kafkaTemplate.send("order-created-topic", order.getId(), json);
}
```

这里会有问题。如果订单插入成功，消息发送失败，那么下游服务可能会不知道这里的业务，后续不会通知库存服务。

还有消息发送成功，但插入失败导致事务回滚，下游服务会收到不存在的订单。

因此，需要使用本地消息表。首先确保消费者成功执行了业务后，将消息保存到本地消息表。由于是两次数据库操作，能够保证原子性。后台通过定时任务轮询本地消息表，读取本地消息表的消息并查看状态，如果状态为PENDING，就将消息发送到消息队列，通知下游服务执行，并改为SENT，不再被定时任务读取。如果发送失败，定时任务会重试直到成功。

## 33. Schema设计

消息格式不能随便写，否则后期维护困难。

常见的格式有JSON、Avro、Protobuf。

JSON能够直接发送，按照key/value的格式来发送。

```json
{
  "orderId": "123",
  "userId": "456",
  "amount": 100.50
}
```

但Avro更适合。这是二进制文件，需要配合Schema注册才能使用。

首先需要定义Schema文件。后缀是.avsc。

```json
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.example.avro",
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "userId", "type": "string" },
    { "name": "amount", "type": "double" },
    { "name": "createTime", "type": "long" }
  ]
}
```

这里定义好后，Java能够自动生成名为OrderCreated的类。可以直接使用这个类，并设置字段值。

```java
// 1. 创建 Avro 对象（使用生成的 Specific Record）
OrderCreated order = OrderCreated.newBuilder()
    .setOrderId("123")
    .setUserId("456")
    .setAmount(100.50)
    .setCreateTime(System.currentTimeMillis())
    .build();

// 2. 配置 Kafka 生产者（使用 Avro 序列化器）
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "io.confluent.kafka.serializers.KafkaAvroSerializer");
props.put("schema.registry.url", "http://schema-registry:8081");

KafkaProducer<String, OrderCreated> producer = new KafkaProducer<>(props);

// 3. 发送消息
producer.send(new ProducerRecord<>("order-topic", "key-123", order));
```

消费者需要设置avro模式，才能读取。

```java
// 配置消费者（使用 Avro 反序列化器）
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "order-consumer");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "io.confluent.kafka.serializers.KafkaAvroDeserializer");
props.put("schema.registry.url", "http://schema-registry:8081");
// 关键：使用 Specific Record 模式
props.put("specific.avro.reader", "true");

KafkaConsumer<String, OrderCreated> consumer = new KafkaConsumer<>(props);
consumer.subscribe(List.of("order-topic"));

while (true) {
    ConsumerRecords<String, OrderCreated> records = consumer.poll(Duration.ofSeconds(1));
    for (ConsumerRecord<String, OrderCreated> record : records) {
        OrderCreated order = record.value();
        String orderId = order.getOrderId();  // 类型安全，IDE 自动补全
        double amount = order.getAmount();
        System.out.println("收到订单: " + orderId + ", 金额: " + amount);
    }
}
```

如果添加字段，可以在avsc文件直接添加字段，但是需要使用default修饰，给定初始值。

```json
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.example.avro",
  "version": 2,
  "fields": [
    { "name": "orderId", "type": "string" },
    { "name": "userId", "type": "string" },
    { "name": "amount", "type": "double" },
    { "name": "createTime", "type": "long" },
    { "name": "couponCode", "type": ["null", "string"], "default": null }
  ]
}
```

如果需要删除字段，只能删除default为null的字段。而没有default的字段不可删除。

如果给出与avsc格式不同的消息，会拒绝注册。

最后，消息格式最好包含下面的字段。

```
eventId：事件ID，用于幂等
eventType：事件类型
eventTime：发生时间
source：来源服务
version：消息版本
traceId：链路追踪
payload：业务数据
```

## 34. Partition数量设计

分区数需要符合下面的规则。

1. 分区数大于消费者数。必须保证消费者至少有一个分区。
2. 分区数就是并行上限。有多少个分区，后台服务就能同时处理多少个消息。
3. 分区数影响顺序行，需要保证顺序的消息需要放到同一个分区。
4. 分区数只能增加，不能减少，需要谨慎增加。

分区数可以按照公式来计算出来。

```
分区数=max(生产吞吐量/单分区生产上限，消费吞吐量/单分区消费上限)
```

## 35. 线程安全

生产者实例是线程安全的。因为它只负责发送消息，消息内容在生产者内部定义，不依赖之前的结果。所以多个节点或线程可以使用同一个生产者实例。

```java
// 多个线程共享同一个生产者实例
KafkaProducer<String, String> producer = new KafkaProducer<>(props);

// 线程1 和 线程2 可以同时调用 producer.send()
producer.send(new ProducerRecord<>("topic", "key1", "value1"));
producer.send(new ProducerRecord<>("topic", "key2", "value2"));
```

消费者实例是非线程安全的。消费者有状态，需要维护分区分配、位移等上下文。

```java
// ❌ 错误：多个线程共享同一个消费者实例
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
// 线程1 和 线程2 同时调用 consumer.poll()
// 会抛出 ConcurrentModificationException
```

因此最好的方法是每个线程独立创建消费者实例，配置相同的消费者组即可。

## 36. Spring集成Kafka

Producer实例。

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

Consumer实例。

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

## 37. 常见问题

1. 消息堆积。

如果出现消息堆积，可以看Lag，看看最新offset和读取的offset的对比，以及看消费者是否存活，处理事件，还有是否有rebalance，下游服务的API是否慢了等。

2. 消息重复。

需要检查offset的提交时机，并设置acks=-1来避免。同时可以检查消费者是否重启，是否发生了rebalance，生产者是否重复发了消息，检查业务是否需要幂等等。

3. 消息丢失。

首先检查acks，不能为0。然后检查重试次数，需要加一点，还有发送后的回调函数，还有`min.insync.replicas`是否设置够，是否自动提交了offset等，或者是否为先提交后处理导致的问题。

## 38. 面试题

### 1. Kafka为什么快

顺序写，零拷贝，批量发送，压缩，分区并行。

### 2. Kafka如何保证消息不丢

Producer使用acks=1或者all，开启重试，处理回调函数等。

Broker设置副本数和最小副本数，关闭unclean leader election。

Consumer关闭自动提交，手动提交offset。

失败消息重试或者进入DLQ。

### 3. 怎么解决重复消费

Kafka的语义通常保证至少一次，因此会出现重复消费的情况。可以通过业务幂等来解决，如消费消息时看eventID是否已经执行，也可以保存消费记录表，或者在Redis中SETNX来保存ID等。

### 4. Kafka消息堆积怎么办

消息堆积需要看Lag和消费耗时，如果消费慢，可以添加分区，或者添加消费者节点，如果是下游API调用慢，需要优化业务逻辑，或者进行限流，消息重试后放入死信队列等。

### 5. 怎么保证顺序消费

Partition内有序，Partition间无序。所以如果要保证顺序的话，需要指定id为key，让同一个业务的消息分发到同一个Partition内，Consumer也不能使用多线程来处理，需要串行来保证顺序。

### 6. Offset是什么

Kafka保存消息是通过日志段文件来保存的，而Offset是指当前消息在段文件里的偏移量。每个消费者组独立维护自己的偏移量，用来读取消息队列里的指定位置的消息。

### 7. 如何实现延迟队列

Kafka没有延迟队列，需要通过外部逻辑来实现，比如在消息中写入ExecuteTime，消费者获取消息的时候就能够等待后执行，或者先不写入消息队列，先写入数据库的任务表，开启定时任务轮询任务表，发现时间到了，就发送到消息队列。还有时间轮，生产者先把消息放到延迟队列，时间到了就放到消息队列。或者消费者把获取到的消息放到延迟队列，时间到了就执行。

### 8. Kafka事务能解决MySQL+Kafka的一致性吗

Kafka事务用于解决多消息发布的原子性，与数据库事务不同。如果要保证一致性，需要使用本地消息表等方法。

### 9. acks=all是否不丢失

acks=all不一定不丢失，如果确保副本数量足够，最小副本数量，以及不允许不在ISR的Follower成为Leader，Producer需要重试和处理回调函数，Consumer也需要正确提交offset。

### 10. 消费者组是什么

在同一个消费者组内，所有的消费者共同消费一个Topic，一个消息只能被消费者组的一个消费者处理。不同消费者组相互独立，相当于广播。

### 11. 如何保证消息能够写入到相同的分区

通过消息键和分区器来实现。如果指定key的话，那么同一个key算出来的分区的相同的，能够保证消息写入到同一个分区。

