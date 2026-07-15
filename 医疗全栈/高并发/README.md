高并发涉及三个指标

* QPS是每秒查询次数，也是吞吐量。
* RT是响应时间，系统处理一个请求需要多少毫秒。
* 并发数。也就是压力，同一时间有多少用户在操作。

在理想情况下，`QPS=并发数/RT`。但由于竞争问题，在RT减小时，虽然理论上QPS会上升，但实际上QPS会下降。而高并发的目标就是让RT尽量不升高，让QPS尽量不下降。

而高并发也会带来三个问题。

1. 流量洪峰。瞬间请求两可能达到平时的几百倍，导致服务器宕机。
2. 资源竞争。大量请求竞争数据库、内存，导致连接池耗尽，大量请求超时。
3. 数据一致性。高并发情况下，多个请求可能同时修改同一条数据，可能导致超卖或者重复扣减。

# 1. 主要方法

高并发的解决思路主要有五个。

## 1. 限流

在请求到来前进行限流，只放行系统能够承受的流量，多余的请求直接拒绝或者排队。

可以通过固定窗口、滑动窗口、漏桶和令牌桶来实现。

## 2. 分流

将流量分到多台机器上。

将服务开启多个节点，通过Nginx或者Gateway实现负载均衡，也可以将多节点注册到Nacos实现服务发现，调用的时候自动进行负载均衡。

如果数据库压力大，可以进行分库分表，将大数据库分为多个小数据库。

或者将网页静态资源放到CDN就近获取，不占用服务端资源。

## 3. 缓存

缓存是高并发比较有用的方法。将经常访问的数据加载到内存，直接返回。

而缓存可以设计多级缓存，针对热点数据缓存到本地缓存，然后到Redis，最后到数据库。这样能够大幅减轻数据库的压力。

## 4. 异步

针对非核心、不需要返回结果的操作，不需要阻塞主线程，可以通过消息队列来进行异步处理。

使用Kafka等将请求暂存起来，后端服务从消息队列中拉取消息，自动消费，主线程可以直接返回处理中，等待处理完成后再更新状态。

## 5. 熔断降级

如果服务出现故障或响应过慢，需要有自我保护机制。

当调用下游服务失败率达到阈值时，需要直接拒绝请求，返回结果，如“系统繁忙”等。熔断达到一定时间，就会开放少部分请求检测下游服务，如果依然失败，就继续熔断，成功则取消熔断。

降级时在流量高峰期，关闭非核心功能，把资源留给核心功能。

整体流程如下。

```
用户请求
CDN 加速静态资源
Nginx 反向代理做限流+负载均衡
Spring Cloud Gateway 路由+限流+熔断
微服务 业务逻辑处理
消息队列 异步处理非核心业务
数据库层 操作数据
```

# 2. 基本原则

处理大量数据时，不应该一次性读取、处理全部数据，而是应该分开处理。

分批处理。一次性处理500条、1000条等，不要加载全部数据。

流式处理。一边读一边处理，一边处理一边释放。

异步处理。大任务不要阻塞执行，而是提交任务，交由后台自行处理。

幂等设计。同一个操作重复多次，应该只有一次执行。

限流和隔离。大任务不能抢光资源，影响正常业务。

# 3. 分页查询

```mysql
SELECT * 
FROM user
ORDER BY id
LIMIT 1000000, 20;
```

这个SQL性能很差，因为数据库需要扫描前1000000行后，再选取20条。

这种LIMIT方式适合普通分页，如前几百行。

```mysql
SELECT id, name, age
FROM user
WHERE status = 1
ORDER BY id DESC
LIMIT 20, 20;
```

> 获取每页大小20，获取第二页的内容

但深分页的时候，性能会很差。

深分页有多种优化方法。

1. 游标分页。

不要通过LIMIT设置偏移量，而是通过id范围查询来跳转到对应的位置。

```mysql
SELECT id, name, age
FROM user
WHERE id > 1000000
ORDER BY id
LIMIT 20;
```

这样，就能通过id索引获取到准确的位置，然后向后读取20条数据。

缺点是这个范围查询的数据不好置顶，需要根据上一页最后的id来指定，适合一直下一页的情况。

2. 延迟关联

```mysql
SELECT u.*
FROM user u
JOIN (
	SELECT id
    FROM user
    WHERE status = 1
    ORDER BY id
    LIMIT 1000000, 20
) tmp ON u.id = tmp.id;
```

首先在内部通过聚簇索引，将符合条件的20行数据的id查找出来，外部再根据这20个id查找数据。两次都走索引，性能高。

# 4. 批量插入

如果请求需要向数据库插入10万条数据，普通的插入方法，如10万次循环插入或者一次超大的SQL的VALUE拼接会有非常严重的性能问题。逐条插入需要往返数据库10万次，导致网络耗时非常多，且每插入一条数据，B+树会进行旋转、分裂，开销巨大。并且如果放到一个事务中，10万条数据会锁住大量数据行和间隙，导致其他业务长时间阻塞，且主库写入Binlog大事务，从库一次性应用这个大事务会出现主从严重延迟。

最佳的方案是分治、批量预编译和手动控制事务。

分治是按批次发送，每次发送500到1000条。

```mysql
INSERT INTO user(name, age)
VALUES
('TOM', 10),
('Jerry', 20),
('Alice', 22);
```

预编译。如果每次都要发送完整的SQL，那么数据库都需要检查语法、表结构和索引。而如果发送前先使用SQL模板，那么数据库会提前缓存模板，后面只需要发送参数，即可直接执行，节省大量语法解析和优化器的开销。

```
发送SQL模板
INSERT INTO user VALUES (?, ?);
程序发送参数。
for () {
	("user_1", 21);
	("user_2", 22);
}
```

手动控制事务。

默认情况下，数据库执行一次INSERT，就会执行一次事务。那么可以在代码中关闭自动提交，每一个批次完成插入后再调用connection.commit()。这样既不会有大事务，也不会有频繁开启事务的情况。

```java

public class BatchInsertExample {
    // 数据库连接配置
    private static final String URL = "jdbc:mysql://localhost:3306/your_database?useSSL=false&serverTimezone=UTC&rewriteBatchedStatements=true";
    private static final String USER = "root";
    private static final String PASSWORD = "password";

    public static void main(String[] args) {
        int totalRows = 100000;       // 总数据量：10万条
        int batchSize = 1000;         // 批处理大小：每1000条发送一次

        String sql = "INSERT INTO user_table (username, email) VALUES (?, ?)";

        // 1. 获取连接
        try (Connection connection = DriverManager.getConnection(URL, USER, PASSWORD)) {
            
            // 【核心优化 3】关闭自动提交，开启手动控制事务
            connection.setAutoCommit(false); 

            // 【核心优化 2】使用 PreparedStatement 预编译 SQL 模板
            try (PreparedStatement pstmt = connection.prepareStatement(sql)) {
                
                long startTime = System.currentTimeMillis();

                for (int i = 1; i <= totalRows; i++) {
                    // 填充 SQL 参数
                    pstmt.setString(1, "user_" + i);
                    pstmt.setString(2, "user_" + i + "@example.com");

                    // 【核心优化 1】将单条数据加入“攒批”队列，而不直接发给数据库
                    pstmt.addBatch(); 

                    // 每当攒满 1000 条，或者到达最后一条数据时，执行一次网络发送
                    if (i % batchSize == 0 || i == totalRows) {
                        // 执行这一批的插入操作
                        pstmt.executeBatch(); 
                        
                        // 【核心优化 3】配合批量，手动提交事务，将数据真正刷入磁盘
                        connection.commit(); 
                        
                        // 清空当前批次，为下一批腾出内存空间
                        pstmt.clearBatch(); 
                    }
                }

                long endTime = System.currentTimeMillis();
                System.out.println("成功插入 10 万条数据，耗时: " + (endTime - startTime) + " 毫秒");

            } catch (SQLException e) {
                // 发生异常时回滚未提交的事务，保证数据一致性
                connection.rollback();
                System.err.println("数据插入失败，事务已回滚！");
                e.printStackTrace();
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

# 5. 批量更新

批量更新更复杂，因为每条数据可能更新的值不同。

普通的更新适合更新为相同的值。

```mysql
UPDATE user
SET status = 0
WHERE id IN (1, 2, 3);
```

通过case能够更新不同值。

```mysql
UPDATE user
SET age = CASE id
	WHEN 1 THEN 20
	WHEN 2 THEN 21
	WHEN 3 THEN 22
END
WHERE id IN (1, 2, 3);
```

如果大数据，这些方法都不好用。

最好的方法是分批次更新。先构建临时表，通过批量插入将新数据导入到临时表，通过连表update来实现更新。

```mysql
update user u
JOIN tmp_user t ON u.id = t.id
SET u.age = t.age,
	u.name = t.name;
```

这样内部会通过Join来实现更新。

# 6. 流式处理

```java
public void streamProcessLargeData() {
    Connection conn = null;
    PreparedStatement ps = null;
    ResultSet rs = null;
    
    try {
        conn = dataSource.getConnection();
        // 1. 关键点：设置游标类型为只读向前滚动
        ps = conn.prepareStatement("SELECT id, data FROM large_table WHERE status = 0",
                                   ResultSet.TYPE_FORWARD_ONLY, 
                                   ResultSet.CONCUR_READ_ONLY);
        
        // 2. 关键点：设置每次从数据库读取的行数（MySQL叫 FetchSize，设置为 Integer.MIN_VALUE 强制流式）
        ps.setFetchSize(Integer.MIN_VALUE);
        
        // 3. 关键点：关闭自动提交（配合游标，防止长事务锁表）
        conn.setAutoCommit(false);
        
        rs = ps.executeQuery();
        
        int count = 0;
        List<Long> batchIds = new ArrayList<>(1000);
        
        while (rs.next()) {
            Long id = rs.getLong("id");
            String data = rs.getString("data");
            
            // 模拟复杂业务计算
            String newData = complexCalculate(data);
            
            // 收集ID，准备后续批量更新（这里只做示例，实际用临时表Join更好）
            batchIds.add(id);
            count++;
            
            // 每1000条，处理一批（比如更新状态，或者发送到MQ）
            if (count % 1000 == 0) {
                // 执行批量更新逻辑（参考上一讲的Case When）
                batchUpdateByIds(batchIds, newStatus);
                batchIds.clear();
                
                // 流式处理中，每批提交一次事务，释放锁
                conn.commit();
            }
        }
        
        // 处理尾巴
        if (!batchIds.isEmpty()) {
            batchUpdateByIds(batchIds, newStatus);
            conn.commit();
        }
        
    } catch (SQLException e) {
        e.printStackTrace();
        if (conn != null) {
            try { conn.rollback(); } catch (SQLException ex) { ex.printStackTrace(); }
        }
    } finally {
        // 关闭资源
    }
}
```

通过设置ResultSet.TYPE_FORWARD_ONLY和ResultSet.CONCUR_READ_ONLY，以及设置setFatchSize为最大值强制流动，以及关闭自动提交即可。

# 7. 任务拆分

大任务不能单线程地直接运行，最好可以拆分成小任务。

例如需要处理id从1到1000000的数据，可以以100000为单位，拆分成10组，分别进行处理。

拆分的方式有多种。

1. 按照id拆分。可以根据id的范围来拆分任务。但缺点是可能分布不均，有的子任务多，有的少。
2. 根据事件范围拆分。如每个子任务负责一天。这符合业务归档的习惯，但可能出现某一天热点数据多导致任务依然很大的情况。
3. 按hash拆分。对数据取哈希然后取模，取n的模就可以拆分成n个子任务。优点是分布均匀，每个子任务数量大致相等，但不适合范围查询。

# 8. 线程池隔离

不同的业务不应该使用同一个线程池，避免相互竞争。

可以一个服务配置多个线程池，核心业务走专属的线程池，非核心业务走另一个线程池。

因此，可以在不同的微服务使用独立的线程池，并且同一服务内的不同接口可以使用不同的线程池，也可以按照用户的等级来区分线程池，保证高价值用户的质量。

# 9. 读写分离

读写分离是写操作走主库，读操作走从库。从库通过主从复制来同步数据。这能够分摊数据库的读压力，适合读多写少，但主从复制会存在延迟，如果刚写完就读从库，可能读不不了数据。

## 1. 主从延迟

主库写入Binlog，从库拉取并更新时，此时读操作可能读取不到新数据。如果不要求强一致性就没关系，如果要求强一致性，需要从主库中读取，或者主库更新后，将数据写入到Redis。

## 2. 从库负载不均

多个从库硬件不同，性能有差异，可以通过负载均衡算法，通过权重轮询等方法来定义均衡策略。

## 3. 从库故障

如果从库发生故障，那么查询会打到主库，主库压力增加，需要使用自动故障转移，从库故障时需要更换另一个从库来读取数据。

## 4. 事务不可重复读

一个事务如果更新了数据，然后再查询两次，可能存在第一次查询到旧数据，第二次从库更新了，查询到新数据。这样的话，事务需要绑定主库，确保事务操作在主库执行。

# 10. 索引

如果要优化SQL，首先要建立索引。对查询条件、排序等，以及联合查询的字段都需要构建索引。

```mysql
CREATE INDEX idx_order_user_status_time
ON orders(user_id, status, created_at);
```

> 联合索引

适合以id、状态和创建时间进行查询的SQL语句。

索引有很多情况失效。

1. 联合索引违反最左匹配原则。
2. 在索引上进行计算或者使用函数。
3. 隐式类型转换。
4. 左模糊查询。
5. OR查询条件存在非索引。
6. 数据量太小，导致回表性能比全表扫描低。

通过EXPLAIN能够查看SQL语句的执行情况。

如果type是ALL，说明进行了全盘扫描，索引失效。如果key是null，也说明没有使用索引。还有rows如果太大，说明扫描了很多行。

# 11. 接口超时

高并发系统需要设置接口超时。如果一个请求的处理时间超过阈值，就会中断请求，释放资源。因为占用的时间越长，其他请求等待的时间也就越长，导致其他请求也无法执行。

设置超时时间在Nginx中设置。

```
nginx: proxy_read_timeout 30s;
```

也在Spring Boot中设置。

```
spring.mvc.async.request-timeout=30000
```

业务层也需要设置。

```
CompletableFuture.orTimeout(30, TimeUnit.SECONDS)
```

```java
@Service
public class FileImportService {
    
    public void importLargeFile(MultipartFile file) {
        CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
            // 耗时的批量插入逻辑
            doBatchInsert(file);
        }, batchExecutor);
        
        // 设置业务超时：30秒没处理完，主动中断
        future.orTimeout(30, TimeUnit.SECONDS)
              .exceptionally(ex -> {
                  if (ex instanceof TimeoutException) {
                      log.error("批量导入超时，任务取消");
                      // 触发补偿逻辑：记录失败状态，人工介入
                  }
                  return null;
              });
    }
}
```

# 12. 反压

反压是系统处理能力饱和时，主动向上游服务反馈压力，让上游暂停发送请求或者减缓速度。

如前端发送100个任务，后端只能处理10个，导致消费者被压死。

前端发送到10个的时候，后端处理10个，此时消费者告诉生产者压力，生产者就能降低速率或者暂停发送，让后端处理完毕。

反压有多种实现方式。

1. 滑动窗口。消费者处理慢的时候，窗口缩小，那么生产端就会减速。
2. 消息队列反压。MQ能够通过配置控制消费者获取消息的速度。
3. 线程池拒绝策略。如果线程池接收太多任务导致线程满、队列满，可以通过拒绝策略来限制任务数量。

# 13. N+1查询

在ORM框架中，如果查询一个对象列表，如10个用户，那么ORM会执行1次SQL查主表，然后在循环中对每个对象执行一次查询来查关联表，总共N+1次查询。

如需要查询用户和订单。

```java
// 你写的代码（MyBatis-Plus / JPA）
List<User> users = userMapper.selectList(null);  // 查出10个用户
for (User user : users) {
    // 这里会触发N次查询
    List<Order> orders = orderMapper.selectByUserId(user.getId());
    System.out.println(user.getName() + " 的订单数：" + orders.size());
}
```

这里实际的SQL如下。

```mysql
-- 第1次查询：查出所有用户
SELECT * FROM user;

-- 第N次查询：对每个用户查一次订单
SELECT * FROM order WHERE user_id = 1;
SELECT * FROM order WHERE user_id = 2;
SELECT * FROM order WHERE user_id = 3;
... (共10次)
```

需要查询多少数据，就需要多少次查询。

这是因为ORM的延迟加载机制，ORM查询主对象时，默认不会查询关联的子对象。只有真正需要使用的时候才会查询，导致需要新的SQL去数据库获取。

这种会导致数据库连接池耗尽，且频繁传输数据会导致大量网络IO，数据库压力很大。

需要解决的话，可以使用join来查询。

```mysql
SELECT u.*, o.*
	FROM user u
	JOIN order o ON u.id = o.user_id;
```

这样能够一次性将关联数据查询出来。

或者使用批量查询，先查询所有用户，再一次性查询所有关联订单。

```mysql
SELECT * FROM order WHERE user_id IN (1,2,3...10000);
```

# 15. 数据冗余

数据冗余是在同一张表中存储冗余字段。

例如订单表可以存储user_name，这样查询订单的时候，能够直接获取用户名，而不需要通过join来查询。

但数据冗余违反了第第二范式，且会带来一致性问题。一旦这个冗余字段可变，那么数据更新的时候，需要考虑其他数据的冗余字段是否需要改变。