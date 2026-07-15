下面给你一份 **ZooKeeper 完整学习教程**，重点面向 Java 后端开发。内容会覆盖：ZooKeeper 是什么、核心模型、集群架构、一致性机制、Watcher、Session、ACL、Java 原生 API、Curator、高频分布式场景、运维配置、故障排查和面试重点。

Apache 官方对 ZooKeeper 的定位是：用于维护配置信息、命名服务、分布式同步和集群成员管理的集中式协调服务。也就是说，ZooKeeper 本身不是数据库、不是缓存、不是消息队列，而是一个 **分布式协调系统**。([Apache ZooKeeper](https://zookeeper.apache.org/?utm_source=chatgpt.com))

当前 Apache ZooKeeper 官方发布页显示，ZooKeeper 社区维护 stable 和 current 两个分支；截至查询时，current release 是 3.9.5，latest stable release 是 3.8.6。实际项目里一般建议使用稳定分支或与现有中间件兼容的版本。([Apache ZooKeeper](https://zookeeper.apache.org/releases.html?utm_source=chatgpt.com))

------

# 1. ZooKeeper 是什么

ZooKeeper 是一个为分布式系统提供协调能力的服务。它通常用于：

1. 服务注册与发现
2. 分布式锁
3. 配置中心
4. 主节点选举
5. 集群成员管理
6. 分布式队列
7. 分布式屏障
8. 元数据管理
9. 分布式 ID 或顺序节点生成
10. 多进程状态同步

它解决的问题不是“存大量数据”，而是“让多个节点对某些关键状态达成一致”。

例如，一个分布式系统中有多个服务实例：

```text
Service-A-1
Service-A-2
Service-A-3
```

现在需要知道哪些实例还活着、谁是主节点、配置是否变化、某个任务是否已经被抢占。ZooKeeper 就可以提供这些协调能力。

------

# 2. ZooKeeper 不适合做什么

很多初学者会误用 ZooKeeper。需要先明确它的边界。

ZooKeeper 不适合：

1. 存储大量业务数据
2. 高频写入数据
3. 替代 Redis 缓存
4. 替代 MySQL 数据库
5. 替代 Kafka 消息队列
6. 存储大文件
7. 做复杂查询
8. 做长文本配置管理

ZooKeeper 的单个 znode 数据通常建议保持很小。它适合存储元数据、状态、配置、服务地址等轻量级信息。

------

# 3. ZooKeeper 的核心思想

ZooKeeper 的核心抽象是一个类似文件系统的树形结构。

```text
/
├── app
│   ├── config
│   ├── services
│   │   ├── order-service
│   │   └── user-service
│   └── locks
└── zookeeper
```

树上的每一个节点叫做 **znode**。

官方文档中说明，ZooKeeper 树中的每个节点都称为 znode，znode 会维护 stat 结构，包括版本号、时间戳等信息；当 znode 数据发生变化时，其版本号会递增，客户端更新或删除节点时可以基于版本号做并发控制。([Apache ZooKeeper](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html?utm_source=chatgpt.com))

------

# 4. znode 类型

ZooKeeper 中的节点主要有 4 类。

## 4.1 持久节点 Persistent Node

创建后一直存在，除非主动删除。

```text
/app/config
```

适合保存配置、目录结构、长期元数据。

Java 示例：

```java
zooKeeper.create(
        "/app",
        "hello".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE,
        CreateMode.PERSISTENT
);
```

------

## 4.2 临时节点 Ephemeral Node

临时节点与客户端 Session 绑定。

客户端断开并且 Session 过期后，临时节点会自动删除。

```text
/services/order-service/instance-1
```

适合做服务注册、心跳检测、在线状态管理。

示例：

```java
zooKeeper.create(
        "/services/order-service/instance-1",
        "192.168.1.10:8080".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE,
        CreateMode.EPHEMERAL
);
```

注意：临时节点下面不能创建子节点。

------

## 4.3 持久顺序节点 Persistent Sequential Node

创建后不会自动删除，并且 ZooKeeper 会在节点名后自动追加递增序号。

例如你创建：

```text
/tasks/task-
```

实际生成：

```text
/tasks/task-0000000001
/tasks/task-0000000002
/tasks/task-0000000003
```

示例：

```java
String path = zooKeeper.create(
        "/tasks/task-",
        "task data".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE,
        CreateMode.PERSISTENT_SEQUENTIAL
);

System.out.println(path);
```

适合做分布式队列、任务排序。

------

## 4.4 临时顺序节点 Ephemeral Sequential Node

节点既是临时的，又带有自动递增编号。

```text
/locks/order-lock/lock-0000000001
/locks/order-lock/lock-0000000002
```

这是实现分布式锁、主节点选举的核心机制。

示例：

```java
String lockNode = zooKeeper.create(
        "/locks/order-lock/lock-",
        new byte[0],
        ZooDefs.Ids.OPEN_ACL_UNSAFE,
        CreateMode.EPHEMERAL_SEQUENTIAL
);
```

------

# 5. ZooKeeper 集群架构

ZooKeeper 通常以集群方式部署，叫做 ensemble。

一个 ZooKeeper 集群通常有 3、5、7 个节点，推荐奇数个节点。

```text
Client
  |
  |----------------------------
  |            |              |
Server-1     Server-2       Server-3
Follower     Leader         Follower
```

ZooKeeper 集群中有三类角色：

## 5.1 Leader

Leader 负责处理写请求、事务提案、事务提交。

## 5.2 Follower

Follower 可以处理读请求，也参与 Leader 选举和写请求投票。

## 5.3 Observer

Observer 不参与投票，只同步数据并处理读请求。

Observer 的作用是提升读扩展能力，但不增加写入投票压力。

------

# 6. 为什么 ZooKeeper 集群通常用奇数台

ZooKeeper 写请求需要多数派确认。

假设集群有 N 台机器，能容忍的故障数为：

```text
floor((N - 1) / 2)
```

| 节点数 | 多数派数量 | 可容忍故障数 |
| ------ | ---------- | ------------ |
| 1      | 1          | 0            |
| 2      | 2          | 0            |
| 3      | 2          | 1            |
| 4      | 3          | 1            |
| 5      | 3          | 2            |
| 6      | 4          | 2            |
| 7      | 4          | 3            |

可以看到，3 台和 4 台都只能容忍 1 台故障，所以 4 台没有明显收益，反而增加成本。因此常见部署是 3 台或 5 台。

------

# 7. ZooKeeper 的一致性模型

ZooKeeper 不是强线性一致性读写模型，但它提供了非常重要的协调一致性保证。

常见理解：

1. 写请求由 Leader 顺序处理。
2. 写操作具有全局顺序。
3. 客户端会看到自己写入之后的结果。
4. Watcher 事件有顺序保证。
5. 不同客户端连接到不同 Follower 时，读可能存在短暂延迟。
6. 如果需要读到最新数据，可以调用 `sync()`。

示例：

```java
zooKeeper.sync("/app/config", (rc, path, ctx) -> {
    try {
        byte[] data = zooKeeper.getData("/app/config", false, null);
        System.out.println(new String(data));
    } catch (Exception e) {
        e.printStackTrace();
    }
}, null);
```

------

# 8. 安装 ZooKeeper

可以本地单机安装学习，也可以用 Docker。

## 8.1 Docker 单机启动

```bash
docker run -d \
  --name zookeeper \
  -p 2181:2181 \
  zookeeper:3.8
```

连接测试：

```bash
docker exec -it zookeeper zkCli.sh
```

进入 CLI 后：

```bash
ls /
create /test hello
get /test
set /test world
delete /test
```

------

## 8.2 Docker Compose 三节点集群

```yaml
version: '3.8'

services:
  zk1:
    image: zookeeper:3.8
    hostname: zk1
    container_name: zk1
    ports:
      - "2181:2181"
    environment:
      ZOO_MY_ID: 1
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=zk3:2888:3888;2181

  zk2:
    image: zookeeper:3.8
    hostname: zk2
    container_name: zk2
    ports:
      - "2182:2181"
    environment:
      ZOO_MY_ID: 2
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=zk3:2888:3888;2181

  zk3:
    image: zookeeper:3.8
    hostname: zk3
    container_name: zk3
    ports:
      - "2183:2181"
    environment:
      ZOO_MY_ID: 3
      ZOO_SERVERS: server.1=zk1:2888:3888;2181 server.2=zk2:2888:3888;2181 server.3=zk3:2888:3888;2181
```

启动：

```bash
docker compose up -d
```

连接：

```bash
docker exec -it zk1 zkCli.sh -server zk1:2181
```

查看状态：

```bash
docker exec -it zk1 zkServer.sh status
docker exec -it zk2 zkServer.sh status
docker exec -it zk3 zkServer.sh status
```

------

# 9. ZooKeeper CLI 常用命令

进入客户端：

```bash
zkCli.sh -server localhost:2181
```

常用命令：

```bash
ls /
create /app hello
get /app
set /app world
stat /app
delete /app
```

递归删除：

```bash
deleteall /app
```

创建顺序节点：

```bash
create -s /queue/task- task1
```

创建临时节点：

```bash
create -e /online/user1 online
```

创建临时顺序节点：

```bash
create -e -s /locks/lock- ""
```

查看 ACL：

```bash
getAcl /app
```

设置 ACL：

```bash
setAcl /app world:anyone:r
```

------

# 10. Java 原生 API 入门

## 10.1 Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.zookeeper</groupId>
        <artifactId>zookeeper</artifactId>
        <version>3.8.6</version>
    </dependency>
</dependencies>
```

------

## 10.2 创建连接

```java
package com.example.zk;

import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;

import java.util.concurrent.CountDownLatch;

public class ZkConnectDemo {

    public static void main(String[] args) throws Exception {
        String connectString = "127.0.0.1:2181";
        int sessionTimeout = 5000;

        CountDownLatch connectedSignal = new CountDownLatch(1);

        ZooKeeper zooKeeper = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent event) {
                System.out.println("Event: " + event);

                if (event.getState() == Event.KeeperState.SyncConnected) {
                    connectedSignal.countDown();
                }
            }
        });

        connectedSignal.await();

        System.out.println("ZooKeeper connected.");
        zooKeeper.close();
    }
}
```

重点：

```java
new ZooKeeper(connectString, sessionTimeout, watcher)
```

连接是异步建立的，所以通常需要 `CountDownLatch` 等待连接完成。

------

# 11. 创建节点

```java
package com.example.zk;

import org.apache.zookeeper.*;
import java.util.concurrent.CountDownLatch;

public class CreateNodeDemo {

    private static ZooKeeper zk;

    public static void main(String[] args) throws Exception {
        connect();

        String path = zk.create(
                "/java-demo",
                "hello zookeeper".getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.PERSISTENT
        );

        System.out.println("Created: " + path);

        zk.close();
    }

    private static void connect() throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        zk = new ZooKeeper("127.0.0.1:2181", 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();
    }
}
```

------

# 12. 判断节点是否存在

```java
Stat stat = zk.exists("/java-demo", false);

if (stat != null) {
    System.out.println("Node exists.");
} else {
    System.out.println("Node does not exist.");
}
```

带 Watcher：

```java
Stat stat = zk.exists("/java-demo", event -> {
    System.out.println("Node event: " + event);
});
```

------

# 13. 获取节点数据

```java
byte[] data = zk.getData("/java-demo", false, null);
System.out.println(new String(data));
```

获取数据和 stat：

```java
Stat stat = new Stat();
byte[] data = zk.getData("/java-demo", false, stat);

System.out.println("Data: " + new String(data));
System.out.println("Version: " + stat.getVersion());
System.out.println("Created time: " + stat.getCtime());
System.out.println("Modified time: " + stat.getMtime());
```

------

# 14. 修改节点数据

```java
Stat stat = zk.setData(
        "/java-demo",
        "new data".getBytes(),
        -1
);
```

第三个参数是版本号。

```java
-1
```

表示忽略版本。

如果要做 CAS 乐观锁控制：

```java
Stat stat = new Stat();
zk.getData("/java-demo", false, stat);

int currentVersion = stat.getVersion();

zk.setData(
        "/java-demo",
        "updated safely".getBytes(),
        currentVersion
);
```

如果版本不匹配，会抛出：

```text
KeeperException.BadVersionException
```

------

# 15. 删除节点

```java
zk.delete("/java-demo", -1);
```

带版本删除：

```java
Stat stat = zk.exists("/java-demo", false);

if (stat != null) {
    zk.delete("/java-demo", stat.getVersion());
}
```

注意：ZooKeeper 不能直接删除有子节点的节点，需要先删除子节点。

------

# 16. 获取子节点

```java
List<String> children = zk.getChildren("/app", false);

for (String child : children) {
    System.out.println(child);
}
```

带 Watcher：

```java
List<String> children = zk.getChildren("/app", event -> {
    System.out.println("Children changed: " + event);
});
```

------

# 17. Watcher 机制

Watcher 是 ZooKeeper 非常核心的机制。

Watcher 可以监听：

1. 节点创建
2. 节点删除
3. 节点数据变化
4. 子节点列表变化
5. 连接状态变化

官方文档说明，ZooKeeper 的 Watch 是一种通知机制，客户端可以在读取数据时设置 watch，当 znode 发生变化时收到通知；同时 znode 还维护版本号，便于客户端判断数据变化。([Apache ZooKeeper](https://zookeeper.apache.org/doc/current/zookeeperProgrammers.html?utm_source=chatgpt.com))

------

## 17.1 Watcher 的重要特性

必须记住：

1. Watcher 是一次性的。
2. 触发后需要重新注册。
3. Watcher 通知只告诉你“发生了变化”，不携带完整新数据。
4. Watcher 事件是异步的。
5. Watcher 与 Session 绑定。
6. 如果 Session 过期，Watcher 也失效。
7. Watcher 不能保证每一次中间状态都被观察到。

例如：

```text
客户端 watch /config
/config 从 A 改成 B
/config 从 B 改成 C
客户端收到通知后再读取，可能直接读到 C
```

Watcher 不适合做精确事件流，适合做状态变化提醒。

------

## 17.2 监听节点数据变化

```java
package com.example.zk;

import org.apache.zookeeper.*;
import java.util.concurrent.CountDownLatch;

public class DataWatchDemo {

    private static ZooKeeper zk;
    private static final String PATH = "/config";

    public static void main(String[] args) throws Exception {
        connect();

        if (zk.exists(PATH, false) == null) {
            zk.create(PATH, "v1".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

        watchData();

        Thread.sleep(Long.MAX_VALUE);
    }

    private static void watchData() throws Exception {
        byte[] data = zk.getData(PATH, event -> {
            if (event.getType() == Watcher.Event.EventType.NodeDataChanged) {
                System.out.println("Data changed.");
                try {
                    watchData(); // 重新注册 Watcher
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, null);

        System.out.println("Current data: " + new String(data));
    }

    private static void connect() throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        zk = new ZooKeeper("127.0.0.1:2181", 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();
    }
}
```

在 CLI 中修改：

```bash
set /config v2
set /config v3
```

Java 程序会收到变化通知。

------

## 17.3 监听子节点变化

```java
package com.example.zk;

import org.apache.zookeeper.*;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class ChildrenWatchDemo {

    private static ZooKeeper zk;
    private static final String PATH = "/services";

    public static void main(String[] args) throws Exception {
        connect();

        if (zk.exists(PATH, false) == null) {
            zk.create(PATH, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

        watchChildren();

        Thread.sleep(Long.MAX_VALUE);
    }

    private static void watchChildren() throws Exception {
        List<String> children = zk.getChildren(PATH, event -> {
            if (event.getType() == Watcher.Event.EventType.NodeChildrenChanged) {
                System.out.println("Children changed.");
                try {
                    watchChildren();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });

        System.out.println("Current children: " + children);
    }

    private static void connect() throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        zk = new ZooKeeper("127.0.0.1:2181", 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();
    }
}
```

CLI 测试：

```bash
create /services/order-service ""
create /services/user-service ""
delete /services/order-service
```

------

# 18. Session 机制

Session 是 ZooKeeper 客户端与服务端之间的会话。

客户端连接 ZooKeeper 后，会建立 Session。Session 有超时时间，例如 5000ms。

如果客户端短暂断开，但是在 Session 过期之前重新连接，临时节点不会删除。

如果 Session 过期：

1. 临时节点删除
2. Watcher 失效
3. 客户端需要重新建立连接
4. 原来基于临时节点实现的锁、注册信息、主节点身份全部失效

典型场景：

```text
服务实例启动
  |
  | 创建 EPHEMERAL 节点
  |
服务进程挂掉
  |
  | Session 过期
  |
ZooKeeper 删除临时节点
  |
其他客户端 watch 到服务下线
```

------

# 19. Stat 结构详解

ZooKeeper 每个 znode 都有一个 Stat。

常用字段：

```java
Stat stat = new Stat();
byte[] data = zk.getData("/config", false, stat);

System.out.println(stat.getCzxid());
System.out.println(stat.getMzxid());
System.out.println(stat.getCtime());
System.out.println(stat.getMtime());
System.out.println(stat.getVersion());
System.out.println(stat.getCversion());
System.out.println(stat.getAversion());
System.out.println(stat.getEphemeralOwner());
System.out.println(stat.getDataLength());
System.out.println(stat.getNumChildren());
```

含义：

| 字段           | 含义                        |
| -------------- | --------------------------- |
| czxid          | 创建该节点的事务 ID         |
| mzxid          | 最后一次修改该节点的事务 ID |
| ctime          | 创建时间                    |
| mtime          | 修改时间                    |
| version        | 数据版本                    |
| cversion       | 子节点版本                  |
| aversion       | ACL 版本                    |
| ephemeralOwner | 临时节点所属 Session ID     |
| dataLength     | 数据长度                    |
| numChildren    | 子节点数量                  |

------

# 20. ZooKeeper 的 zxid

zxid 是 ZooKeeper 的事务 ID。

每次写操作都会生成一个 zxid。

zxid 可以理解为全局事务顺序号。

```text
create /a
set /a
delete /a
```

每次写都会对应一个递增的 zxid。

ZooKeeper 依赖 zxid 来保证事务顺序、Leader 恢复、数据同步。

------

# 21. ACL 权限控制

ZooKeeper 支持 ACL。

ACL 权限包括：

| 权限   | 含义                 |
| ------ | -------------------- |
| CREATE | 创建子节点           |
| READ   | 读取节点数据和子节点 |
| WRITE  | 修改节点数据         |
| DELETE | 删除子节点           |
| ADMIN  | 修改 ACL             |

Java 中对应：

```java
ZooDefs.Perms.CREATE
ZooDefs.Perms.READ
ZooDefs.Perms.WRITE
ZooDefs.Perms.DELETE
ZooDefs.Perms.ADMIN
ZooDefs.Perms.ALL
```

------

## 21.1 常见 ACL 模式

### world:anyone

任何人都可访问。

```java
ZooDefs.Ids.OPEN_ACL_UNSAFE
```

### digest

用户名密码认证。

```text
user:password
```

### ip

基于 IP 授权。

### sasl

基于 Kerberos / SASL。

------

## 21.2 Java Digest ACL 示例

```java
package com.example.zk;

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.ACL;
import org.apache.zookeeper.data.Id;

import java.security.MessageDigest;
import java.util.Base64;
import java.util.Collections;
import java.util.concurrent.CountDownLatch;

public class AclDemo {

    public static void main(String[] args) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();

        zk.addAuthInfo("digest", "admin:123456".getBytes());

        ACL acl = new ACL(
                ZooDefs.Perms.ALL,
                new Id("auth", "")
        );

        if (zk.exists("/secure", false) == null) {
            zk.create(
                    "/secure",
                    "secret".getBytes(),
                    Collections.singletonList(acl),
                    CreateMode.PERSISTENT
            );
        }

        byte[] data = zk.getData("/secure", false, null);
        System.out.println(new String(data));

        zk.close();
    }
}
```

说明：

```java
new Id("auth", "")
```

表示使用当前已经认证的用户。

------

# 22. 异步 API

ZooKeeper Java 原生 API 支持异步调用。

同步创建：

```java
zk.create(path, data, acl, createMode);
```

异步创建：

```java
zk.create(
        "/async-node",
        "hello".getBytes(),
        ZooDefs.Ids.OPEN_ACL_UNSAFE,
        CreateMode.PERSISTENT,
        (rc, path, ctx, name) -> {
            System.out.println("Return code: " + rc);
            System.out.println("Path: " + path);
            System.out.println("Name: " + name);
            System.out.println("Context: " + ctx);
        },
        "my-context"
);
```

异步读取：

```java
zk.getData(
        "/async-node",
        false,
        (rc, path, ctx, data, stat) -> {
            System.out.println("Data: " + new String(data));
        },
        null
);
```

异步 API 适合高并发客户端，避免阻塞业务线程。

------

# 23. Curator 简介

原生 ZooKeeper API 偏底层，需要自己处理：

1. 连接重试
2. Session 过期
3. Watcher 重复注册
4. 分布式锁边界条件
5. 节点递归创建
6. 异常处理

生产项目中更常用 Apache Curator。

Curator 是 ZooKeeper 的高级客户端库，封装了：

1. CuratorFramework
2. RetryPolicy
3. Recipes
4. Distributed Lock
5. Leader Election
6. NodeCache / PathChildrenCache / CuratorCache
7. Service Discovery

------

# 24. Curator Maven 依赖

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-framework</artifactId>
        <version>5.6.0</version>
    </dependency>

    <dependency>
        <groupId>org.apache.curator</groupId>
        <artifactId>curator-recipes</artifactId>
        <version>5.6.0</version>
    </dependency>
</dependencies>
```

版本需要根据 ZooKeeper 版本兼容性调整。实际项目中建议统一由 Spring Boot、Spring Cloud 或中间件 BOM 管理依赖版本。

------

# 25. Curator 创建连接

```java
package com.example.curator;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class CuratorConnectDemo {

    public static void main(String[] args) throws Exception {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);

        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181")
                .sessionTimeoutMs(5000)
                .connectionTimeoutMs(3000)
                .retryPolicy(retryPolicy)
                .namespace("demo")
                .build();

        client.start();

        System.out.println("Curator started.");

        client.close();
    }
}
```

注意：

```java
.namespace("demo")
```

表示所有操作都会自动加上 `/demo` 前缀。

例如：

```java
client.create().forPath("/config")
```

实际路径是：

```text
/demo/config
```

------

# 26. Curator CRUD 示例

```java
package com.example.curator;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;

public class CuratorCrudDemo {

    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                "127.0.0.1:2181",
                5000,
                3000,
                new ExponentialBackoffRetry(1000, 3)
        );

        client.start();

        String path = "/curator/config";

        if (client.checkExists().forPath(path) == null) {
            client.create()
                    .creatingParentsIfNeeded()
                    .forPath(path, "v1".getBytes());
        }

        byte[] data = client.getData().forPath(path);
        System.out.println(new String(data));

        client.setData().forPath(path, "v2".getBytes());

        byte[] newData = client.getData().forPath(path);
        System.out.println(new String(newData));

        client.delete()
                .deletingChildrenIfNeeded()
                .forPath("/curator");

        client.close();
    }
}
```

Curator 相比原生 API 的优势很明显：

```java
.creatingParentsIfNeeded()
.deletingChildrenIfNeeded()
```

原生 API 需要自己递归处理。

------

# 27. 使用 ZooKeeper 实现配置中心

## 27.1 思路

```text
/config/order-service
```

节点保存 JSON 配置：

```json
{
  "timeout": 3000,
  "maxRetry": 3
}
```

应用启动时读取配置，并 watch 配置节点。一旦配置变化，重新读取。

------

## 27.2 原生 API 实现配置监听

```java
package com.example.zk.config;

import org.apache.zookeeper.*;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicReference;

public class ZkConfigCenter {

    private static final String CONFIG_PATH = "/config/order-service";

    private final ZooKeeper zk;
    private final AtomicReference<String> config = new AtomicReference<>();

    public ZkConfigCenter(String connectString) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        this.zk = new ZooKeeper(connectString, 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();
    }

    public void init() throws Exception {
        if (zk.exists("/config", false) == null) {
            zk.create("/config", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

        if (zk.exists(CONFIG_PATH, false) == null) {
            zk.create(CONFIG_PATH, "{}".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

        loadAndWatch();
    }

    private void loadAndWatch() throws Exception {
        byte[] data = zk.getData(CONFIG_PATH, event -> {
            if (event.getType() == Watcher.Event.EventType.NodeDataChanged) {
                try {
                    loadAndWatch();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }, null);

        String value = new String(data);
        config.set(value);
        System.out.println("Loaded config: " + value);
    }

    public String getConfig() {
        return config.get();
    }

    public void close() throws InterruptedException {
        zk.close();
    }

    public static void main(String[] args) throws Exception {
        ZkConfigCenter center = new ZkConfigCenter("127.0.0.1:2181");
        center.init();

        Thread.sleep(Long.MAX_VALUE);
    }
}
```

CLI 修改：

```bash
set /config/order-service '{"timeout":5000,"maxRetry":5}'
```

------

# 28. 服务注册与发现

## 28.1 思路

服务启动时创建临时节点：

```text
/services/order-service/instance-0000000001
/services/order-service/instance-0000000002
```

节点数据保存服务地址：

```text
192.168.1.10:8080
```

服务下线或崩溃后，Session 过期，临时节点自动删除。

消费者 watch：

```text
/services/order-service
```

子节点变化后重新拉取服务列表。

------

## 28.2 服务注册代码

```java
package com.example.zk.discovery;

import org.apache.zookeeper.*;

import java.util.concurrent.CountDownLatch;

public class ServiceRegistry {

    private final ZooKeeper zk;

    public ServiceRegistry(String connectString) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        this.zk = new ZooKeeper(connectString, 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();
    }

    public void register(String serviceName, String address) throws Exception {
        String root = "/services";
        String servicePath = root + "/" + serviceName;

        createIfAbsent(root);
        createIfAbsent(servicePath);

        String instancePath = servicePath + "/instance-";

        String actualPath = zk.create(
                instancePath,
                address.getBytes(),
                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL
        );

        System.out.println("Registered: " + actualPath + " -> " + address);
    }

    private void createIfAbsent(String path) throws Exception {
        if (zk.exists(path, false) == null) {
            try {
                zk.create(path, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            } catch (KeeperException.NodeExistsException ignored) {
            }
        }
    }

    public static void main(String[] args) throws Exception {
        ServiceRegistry registry = new ServiceRegistry("127.0.0.1:2181");
        registry.register("order-service", "127.0.0.1:8080");

        Thread.sleep(Long.MAX_VALUE);
    }
}
```

------

## 28.3 服务发现代码

```java
package com.example.zk.discovery;

import org.apache.zookeeper.*;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.atomic.AtomicReference;

public class ServiceDiscovery {

    private final ZooKeeper zk;
    private final String servicePath;
    private final AtomicReference<List<String>> instances = new AtomicReference<>(new ArrayList<>());

    public ServiceDiscovery(String connectString, String serviceName) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        this.zk = new ZooKeeper(connectString, 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();

        this.servicePath = "/services/" + serviceName;
    }

    public void start() throws Exception {
        watchInstances();
    }

    private void watchInstances() throws Exception {
        List<String> children = zk.getChildren(servicePath, event -> {
            if (event.getType() == Watcher.Event.EventType.NodeChildrenChanged) {
                try {
                    watchInstances();
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        });

        List<String> currentInstances = new ArrayList<>();

        for (String child : children) {
            String path = servicePath + "/" + child;
            byte[] data = zk.getData(path, false, null);
            currentInstances.add(new String(data));
        }

        instances.set(currentInstances);
        System.out.println("Instances: " + currentInstances);
    }

    public List<String> getInstances() {
        return instances.get();
    }

    public static void main(String[] args) throws Exception {
        ServiceDiscovery discovery = new ServiceDiscovery("127.0.0.1:2181", "order-service");
        discovery.start();

        Thread.sleep(Long.MAX_VALUE);
    }
}
```

------

# 29. 分布式锁

ZooKeeper 分布式锁是最经典的使用场景之一。

官方 Recipes 文档也描述了基于顺序节点实现分布式锁、队列等协调模式。锁的核心思路是：客户端在锁目录下创建临时顺序节点，编号最小者获得锁；未获得锁的客户端监听自己前一个节点，前一个节点删除后再尝试获取锁。([Apache ZooKeeper](https://zookeeper.apache.org/doc/current/recipes.html?utm_source=chatgpt.com))

------

## 29.1 错误的锁实现

错误做法：

```text
所有客户端都 watch /lock
```

问题：

```text
锁释放时，所有客户端都被唤醒
```

这会造成羊群效应。

------

## 29.2 正确的锁实现

正确做法：

```text
1. 每个客户端创建 EPHEMERAL_SEQUENTIAL 节点
2. 获取所有子节点并排序
3. 如果自己是最小节点，获得锁
4. 否则 watch 自己前一个节点
5. 前一个节点删除后，再判断自己是否最小
```

示意：

```text
/locks/order-lock
├── lock-0000000001  ← 获得锁
├── lock-0000000002  ← watch lock-0000000001
├── lock-0000000003  ← watch lock-0000000002
└── lock-0000000004  ← watch lock-0000000003
```

这样每次释放锁，只唤醒一个客户端。

------

## 29.3 原生 Java 实现分布式锁

```java
package com.example.zk.lock;

import org.apache.zookeeper.*;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class ZkDistributedLock {

    private final ZooKeeper zk;
    private final String lockRoot;
    private String currentNode;

    public ZkDistributedLock(String connectString, String lockRoot) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        this.zk = new ZooKeeper(connectString, 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();

        this.lockRoot = lockRoot;

        if (zk.exists(lockRoot, false) == null) {
            try {
                zk.create(lockRoot, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            } catch (KeeperException.NodeExistsException ignored) {
            }
        }
    }

    public void lock() throws Exception {
        currentNode = zk.create(
                lockRoot + "/lock-",
                new byte[0],
                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL
        );

        attemptLock();
    }

    private void attemptLock() throws Exception {
        while (true) {
            List<String> children = zk.getChildren(lockRoot, false);
            Collections.sort(children);

            String currentNodeName = currentNode.substring(lockRoot.length() + 1);
            int index = children.indexOf(currentNodeName);

            if (index == -1) {
                throw new IllegalStateException("Current node not found: " + currentNode);
            }

            if (index == 0) {
                System.out.println(Thread.currentThread().getName() + " acquired lock: " + currentNode);
                return;
            }

            String previousNodeName = children.get(index - 1);
            String previousNodePath = lockRoot + "/" + previousNodeName;

            CountDownLatch waitLatch = new CountDownLatch(1);

            Stat stat = zk.exists(previousNodePath, event -> {
                if (event.getType() == Watcher.Event.EventType.NodeDeleted) {
                    waitLatch.countDown();
                }
            });

            if (stat != null) {
                waitLatch.await();
            }
        }
    }

    public void unlock() throws Exception {
        if (currentNode != null) {
            zk.delete(currentNode, -1);
            System.out.println(Thread.currentThread().getName() + " released lock: " + currentNode);
            currentNode = null;
        }
    }

    public void close() throws InterruptedException {
        zk.close();
    }

    public static void main(String[] args) throws Exception {
        ZkDistributedLock lock = new ZkDistributedLock("127.0.0.1:2181", "/locks/order-lock");

        lock.lock();

        try {
            System.out.println("Doing business logic...");
            Thread.sleep(5000);
        } finally {
            lock.unlock();
            lock.close();
        }
    }
}
```

------

## 29.4 Curator 分布式锁

生产环境更推荐 Curator。

```java
package com.example.curator.lock;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.apache.curator.retry.ExponentialBackoffRetry;

import java.util.concurrent.TimeUnit;

public class CuratorLockDemo {

    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                "127.0.0.1:2181",
                5000,
                3000,
                new ExponentialBackoffRetry(1000, 3)
        );

        client.start();

        InterProcessMutex lock = new InterProcessMutex(client, "/locks/payment-lock");

        if (lock.acquire(10, TimeUnit.SECONDS)) {
            try {
                System.out.println("Lock acquired.");
                Thread.sleep(3000);
            } finally {
                lock.release();
                System.out.println("Lock released.");
            }
        } else {
            System.out.println("Failed to acquire lock.");
        }

        client.close();
    }
}
```

Curator 的 `InterProcessMutex` 是可重入分布式锁。

------

# 30. 主节点选举

主节点选举和分布式锁类似。

## 30.1 实现思路

```text
/election
├── candidate-0000000001  ← leader
├── candidate-0000000002
├── candidate-0000000003
```

编号最小的节点成为 Leader。

Leader 挂掉后，其临时节点消失，后面的候选者重新判断。

------

## 30.2 原生 Java 实现 Leader 选举

```java
package com.example.zk.election;

import org.apache.zookeeper.*;

import java.util.Collections;
import java.util.List;
import java.util.concurrent.CountDownLatch;

public class LeaderElection {

    private final ZooKeeper zk;
    private final String electionRoot = "/election";
    private String currentNode;

    public LeaderElection(String connectString) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        this.zk = new ZooKeeper(connectString, 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();

        if (zk.exists(electionRoot, false) == null) {
            try {
                zk.create(electionRoot, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            } catch (KeeperException.NodeExistsException ignored) {
            }
        }
    }

    public void volunteer() throws Exception {
        currentNode = zk.create(
                electionRoot + "/candidate-",
                new byte[0],
                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL
        );

        electLeader();
    }

    private void electLeader() throws Exception {
        List<String> children = zk.getChildren(electionRoot, false);
        Collections.sort(children);

        String currentNodeName = currentNode.substring(electionRoot.length() + 1);
        String smallestNode = children.get(0);

        if (currentNodeName.equals(smallestNode)) {
            System.out.println("I am leader: " + currentNodeName);
        } else {
            int index = children.indexOf(currentNodeName);
            String previousNode = children.get(index - 1);
            String previousPath = electionRoot + "/" + previousNode;

            System.out.println("I am follower: " + currentNodeName + ", watching: " + previousNode);

            Stat stat = zk.exists(previousPath, event -> {
                if (event.getType() == Watcher.Event.EventType.NodeDeleted) {
                    try {
                        electLeader();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }
                }
            });

            if (stat == null) {
                electLeader();
            }
        }
    }

    public static void main(String[] args) throws Exception {
        LeaderElection election = new LeaderElection("127.0.0.1:2181");
        election.volunteer();

        Thread.sleep(Long.MAX_VALUE);
    }
}
```

------

## 30.3 Curator LeaderLatch

```java
package com.example.curator.election;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.framework.recipes.leader.LeaderLatch;
import org.apache.curator.retry.ExponentialBackoffRetry;

import java.util.UUID;

public class CuratorLeaderLatchDemo {

    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorFrameworkFactory.newClient(
                "127.0.0.1:2181",
                5000,
                3000,
                new ExponentialBackoffRetry(1000, 3)
        );

        client.start();

        String id = UUID.randomUUID().toString();

        LeaderLatch leaderLatch = new LeaderLatch(client, "/leader/order-service", id);
        leaderLatch.start();

        while (true) {
            if (leaderLatch.hasLeadership()) {
                System.out.println(id + " is leader.");
            } else {
                System.out.println(id + " is follower.");
            }

            Thread.sleep(3000);
        }
    }
}
```

------

# 31. 分布式队列

官方 Recipes 中也介绍了队列方案：客户端在队列节点下创建带顺序号的子节点，消费者通过 `getChildren()` 获取队列元素，并按最小序号处理。([Apache ZooKeeper](https://zookeeper.apache.org/doc/current/recipes.html?utm_source=chatgpt.com))

## 31.1 队列结构

```text
/queue
├── item-0000000001
├── item-0000000002
└── item-0000000003
```

------

## 31.2 生产者

```java
public void offer(String value) throws Exception {
    zk.create(
            "/queue/item-",
            value.getBytes(),
            ZooDefs.Ids.OPEN_ACL_UNSAFE,
            CreateMode.PERSISTENT_SEQUENTIAL
    );
}
```

------

## 31.3 消费者

```java
public String poll() throws Exception {
    while (true) {
        List<String> children = zk.getChildren("/queue", false);

        if (children.isEmpty()) {
            return null;
        }

        Collections.sort(children);

        String first = children.get(0);
        String path = "/queue/" + first;

        try {
            byte[] data = zk.getData(path, false, null);
            zk.delete(path, -1);
            return new String(data);
        } catch (KeeperException.NoNodeException ignored) {
            // 被其他消费者抢先删除，重试
        }
    }
}
```

注意：ZooKeeper 可以实现简单队列，但不建议替代 Kafka、RabbitMQ 这类专业消息系统。

------

# 32. 分布式屏障 Barrier

屏障用于等待多个进程都到达某个状态后再继续执行。

例如 5 个服务实例都准备完毕后，统一开始任务。

## 32.1 思路

```text
/barrier/task-1
├── worker-1
├── worker-2
├── worker-3
├── worker-4
└── worker-5
```

当子节点数量达到指定值，所有 worker 开始执行。

------

## 32.2 示例代码

```java
package com.example.zk.barrier;

import org.apache.zookeeper.*;

import java.util.List;
import java.util.concurrent.CountDownLatch;

public class DistributedBarrier {

    private final ZooKeeper zk;
    private final String barrierPath;
    private final int parties;
    private String currentNode;

    public DistributedBarrier(String connectString, String barrierPath, int parties) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        this.zk = new ZooKeeper(connectString, 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();

        this.barrierPath = barrierPath;
        this.parties = parties;

        if (zk.exists(barrierPath, false) == null) {
            try {
                zk.create(barrierPath, new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            } catch (KeeperException.NodeExistsException ignored) {
            }
        }
    }

    public void await() throws Exception {
        currentNode = zk.create(
                barrierPath + "/worker-",
                new byte[0],
                ZooDefs.Ids.OPEN_ACL_UNSAFE,
                CreateMode.EPHEMERAL_SEQUENTIAL
        );

        while (true) {
            CountDownLatch waitLatch = new CountDownLatch(1);

            List<String> children = zk.getChildren(barrierPath, event -> {
                if (event.getType() == Watcher.Event.EventType.NodeChildrenChanged) {
                    waitLatch.countDown();
                }
            });

            if (children.size() >= parties) {
                System.out.println("Barrier passed.");
                return;
            }

            waitLatch.await();
        }
    }

    public static void main(String[] args) throws Exception {
        DistributedBarrier barrier = new DistributedBarrier("127.0.0.1:2181", "/barrier/task1", 3);
        barrier.await();

        System.out.println("Start task.");
        Thread.sleep(Long.MAX_VALUE);
    }
}
```

------

# 33. ZooKeeper 事务 multi

ZooKeeper 支持多个操作组成事务。

要么全部成功，要么全部失败。

示例：

```java
zk.multi(List.of(
        Op.create("/tx/a", "a".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT),
        Op.create("/tx/b", "b".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT),
        Op.setData("/tx/c", "c".getBytes(), -1)
));
```

完整示例：

```java
package com.example.zk.transaction;

import org.apache.zookeeper.*;

import java.util.List;
import java.util.concurrent.CountDownLatch;

public class MultiDemo {

    public static void main(String[] args) throws Exception {
        CountDownLatch latch = new CountDownLatch(1);

        ZooKeeper zk = new ZooKeeper("127.0.0.1:2181", 5000, event -> {
            if (event.getState() == Watcher.Event.KeeperState.SyncConnected) {
                latch.countDown();
            }
        });

        latch.await();

        if (zk.exists("/tx", false) == null) {
            zk.create("/tx", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

        zk.multi(List.of(
                Op.create("/tx/a", "a".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT),
                Op.create("/tx/b", "b".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT)
        ));

        System.out.println("Transaction success.");

        zk.close();
    }
}
```

------

# 34. ZooKeeper 典型应用场景总结

## 34.1 配置中心

```text
/config/payment-service
```

优点：

1. 配置集中管理
2. Watcher 自动通知
3. 客户端本地缓存

缺点：

1. 不适合大配置
2. Watcher 一次性，需要重新注册
3. 不适合频繁变更

------

## 34.2 服务发现

```text
/services/user-service/instance-xxx
```

优点：

1. 临时节点自动感知下线
2. 子节点监听实现服务列表刷新
3. 适合早期 RPC 框架

缺点：

1. 大规模服务实例下 watch 压力较大
2. 现代系统更常用 Nacos、Consul、Eureka、Kubernetes Service

------

## 34.3 分布式锁

```text
/locks/resource/lock-0000000001
```

优点：

1. 可靠性较高
2. 锁释放可由 Session 过期兜底
3. 顺序节点避免锁竞争混乱

缺点：

1. 性能不如 Redis 锁
2. 实现细节复杂
3. Session 超时参数设置不当会导致锁误释放

------

## 34.4 Leader 选举

```text
/election/candidate-0000000001
```

适合：

1. 主从调度系统
2. 定时任务唯一执行者
3. 分布式控制器
4. 集群管理器

------

## 34.5 集群成员管理

```text
/members/node-xxx
```

节点上线创建临时节点，节点下线自动删除。

------

# 35. ZooKeeper 配置文件详解

典型 `zoo.cfg`：

```properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/var/lib/zookeeper
dataLogDir=/var/log/zookeeper
clientPort=2181

server.1=zk1:2888:3888
server.2=zk2:2888:3888
server.3=zk3:2888:3888
```

字段解释：

| 配置       | 说明                                      |
| ---------- | ----------------------------------------- |
| tickTime   | 基础时间单位，毫秒                        |
| initLimit  | Follower 初始化连接 Leader 的最大 tick 数 |
| syncLimit  | Follower 与 Leader 同步的最大 tick 数     |
| dataDir    | 快照数据目录                              |
| dataLogDir | 事务日志目录                              |
| clientPort | 客户端连接端口                            |
| server.X   | 集群节点配置                              |

`server.X`：

```properties
server.1=host:2888:3888
```

含义：

```text
2888: Follower 和 Leader 通信端口
3888: Leader 选举端口
```

每个节点还需要在 `dataDir` 下创建 `myid` 文件。

例如 zk1：

```bash
echo 1 > /var/lib/zookeeper/myid
```

zk2：

```bash
echo 2 > /var/lib/zookeeper/myid
```

zk3：

```bash
echo 3 > /var/lib/zookeeper/myid
```

------

# 36. 四字命令

ZooKeeper 支持一些四字命令，用于运维诊断。

需要在配置中开启白名单：

```properties
4lw.commands.whitelist=stat,ruok,conf,cons,mntr,srvr,wchs,wchc,dirs
```

常用命令：

```bash
echo ruok | nc 127.0.0.1 2181
```

返回：

```text
imok
```

查看状态：

```bash
echo stat | nc 127.0.0.1 2181
```

查看监控指标：

```bash
echo mntr | nc 127.0.0.1 2181
```

查看服务器配置：

```bash
echo conf | nc 127.0.0.1 2181
```

查看连接：

```bash
echo cons | nc 127.0.0.1 2181
```

------

# 37. 重要异常

Java 开发中常见异常：

## 37.1 NodeExistsException

节点已存在。

```java
try {
    zk.create("/app", new byte[0], ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
} catch (KeeperException.NodeExistsException e) {
    // ignore
}
```

------

## 37.2 NoNodeException

节点不存在。

```java
try {
    zk.getData("/not-exists", false, null);
} catch (KeeperException.NoNodeException e) {
    System.out.println("Node not exists.");
}
```

------

## 37.3 BadVersionException

版本不匹配。

通常是并发修改导致。

```java
zk.setData("/config", "v2".getBytes(), oldVersion);
```

如果别人已经改过，版本会不一致。

------

## 37.4 ConnectionLossException

客户端与服务端连接丢失。

注意：连接丢失时，不能简单认为操作失败。

例如：

```text
客户端发送 create 请求
服务端创建成功
响应返回前网络断开
客户端收到 ConnectionLossException
```

此时节点可能已经创建成功。

所以生产代码要具备幂等性。

------

## 37.5 SessionExpiredException

Session 已过期。

这通常意味着：

1. 临时节点已经丢失
2. 锁已经失效
3. Watcher 已失效
4. 客户端需要重新初始化状态

------

# 38. ZooKeeper 的 CAP 理解

ZooKeeper 更偏向 CP。

它优先保证一致性和分区容错性。

当集群无法形成多数派时，ZooKeeper 不能继续处理写请求。

例如 3 节点集群：

```text
zk1, zk2, zk3
```

如果只剩 1 个节点可用：

```text
zk1
```

由于不能形成多数派，不能对外提供正常写服务。

------

# 39. ZooKeeper 与 Redis 分布式锁对比

| 对比项   | ZooKeeper 锁         | Redis 锁         |
| -------- | -------------------- | ---------------- |
| 基础机制 | 临时顺序节点         | SET NX PX        |
| 锁释放   | Session 过期自动删除 | TTL 过期         |
| 公平性   | 容易实现公平锁       | 通常非公平       |
| 惊群问题 | 可监听前驱节点避免   | 需要额外设计     |
| 性能     | 较低                 | 较高             |
| 复杂度   | 较高                 | 较低             |
| 一致性   | 较强                 | 取决于部署和算法 |
| 适用场景 | 强协调、主选举       | 高频轻量锁       |

结论：

1. 要强协调、公平锁、主节点选举，用 ZooKeeper。
2. 要高性能、短时间互斥，用 Redis。
3. 要数据库资源互斥，也可以考虑数据库唯一索引或悲观锁。
4. 生产中不要手写复杂锁，优先用 Curator。

------

# 40. ZooKeeper 与 Eureka、Nacos、Consul 对比

| 组件      | 定位                   | 一致性倾向 | 典型用途              |
| --------- | ---------------------- | ---------- | --------------------- |
| ZooKeeper | 分布式协调             | CP         | 锁、选举、元数据      |
| Eureka    | 服务注册发现           | AP         | Spring Cloud 服务发现 |
| Nacos     | 配置中心、服务发现     | 可配置     | 微服务注册配置        |
| Consul    | 服务发现、KV、健康检查 | CP         | 服务治理              |
| etcd      | 分布式 KV、协调        | CP         | Kubernetes 元数据     |

ZooKeeper 是通用协调系统，但现代云原生体系中，很多场景会使用 etcd、Nacos 或 Kubernetes 原生机制替代。

------

# 41. 生产环境注意事项

## 41.1 不要存大数据

ZooKeeper 适合小数据。

错误：

```text
/config/large-json  5MB
```

正确：

```text
/config/order-service  几 KB
```

------

## 41.2 Watcher 不要滥用

大量客户端 watch 同一个节点，会造成通知风暴。

要避免：

```text
10000 个客户端 watch /config/global
```

可以考虑：

1. 分层配置
2. 本地缓存
3. 灰度刷新
4. 使用专业配置中心

------

## 41.3 Session Timeout 要合理

太短：

```text
网络抖动导致 Session 过期，临时节点误删
```

太长：

```text
故障节点长时间无法被感知
```

常见设置：

```text
5s ~ 30s
```

实际要结合网络、业务容忍度、部署环境决定。

------

## 41.4 不要让业务线程直接阻塞在 ZooKeeper 操作上

建议：

1. ZooKeeper 结果本地缓存
2. 后台线程 watch 变化
3. 业务线程读取本地内存状态

例如配置中心：

```text
ZooKeeper → Watcher → 本地 AtomicReference → 业务读取
```

------

## 41.5 所有操作都要考虑重试与幂等

尤其是：

```text
ConnectionLossException
```

因为请求可能已经在服务端成功，只是客户端没收到响应。

------

## 41.6 锁业务必须小心 Session 过期

如果客户端持有锁期间发生长时间 GC，可能导致 Session 过期，锁节点被删除，其他客户端获得锁。

但原客户端恢复后可能还以为自己持有锁。

所以锁保护的业务要设计：

1. fencing token
2. 版本号
3. 数据库乐观锁
4. 业务幂等
5. 超时中断机制

------

# 42. fencing token 是什么

在分布式锁中，只靠“我拿到了锁”还不够。

假设：

```text
Client A 获得锁 lock-0000000001
Client A 长时间 GC
Session 过期，锁释放
Client B 获得锁 lock-0000000002
Client A GC 恢复，以为自己还持有锁，继续写数据库
```

这就出问题了。

解决方法是 fencing token。

ZooKeeper 的顺序节点编号可以作为 token：

```text
lock-0000000001 → token = 1
lock-0000000002 → token = 2
```

下游资源只接受更大的 token。

例如数据库表：

```sql
UPDATE resource
SET value = ?, token = ?
WHERE id = ? AND token < ?
```

这样旧锁持有者即使恢复，也无法覆盖新锁持有者的写入。

------

# 43. Spring Boot 集成 Curator

## 43.1 Maven

```xml
<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-framework</artifactId>
    <version>5.6.0</version>
</dependency>

<dependency>
    <groupId>org.apache.curator</groupId>
    <artifactId>curator-recipes</artifactId>
    <version>5.6.0</version>
</dependency>
```

------

## 43.2 配置类

```java
package com.example.config;

import org.apache.curator.RetryPolicy;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CuratorConfig {

    @Bean(initMethod = "start", destroyMethod = "close")
    public CuratorFramework curatorFramework() {
        RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);

        return CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181")
                .sessionTimeoutMs(10000)
                .connectionTimeoutMs(3000)
                .retryPolicy(retryPolicy)
                .namespace("my-app")
                .build();
    }
}
```

------

## 43.3 分布式锁 Service

```java
package com.example.service;

import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.locks.InterProcessMutex;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class OrderService {

    private final CuratorFramework curatorFramework;

    public OrderService(CuratorFramework curatorFramework) {
        this.curatorFramework = curatorFramework;
    }

    public void createOrder(Long productId) throws Exception {
        String lockPath = "/locks/product-" + productId;
        InterProcessMutex lock = new InterProcessMutex(curatorFramework, lockPath);

        boolean acquired = lock.acquire(5, TimeUnit.SECONDS);

        if (!acquired) {
            throw new IllegalStateException("Failed to acquire lock.");
        }

        try {
            System.out.println("Check stock.");
            System.out.println("Deduct stock.");
            System.out.println("Create order.");
        } finally {
            lock.release();
        }
    }
}
```

------

# 44. ZooKeeper 面试重点

## 44.1 ZooKeeper 是什么？

ZooKeeper 是一个分布式协调服务，用于配置管理、服务发现、分布式锁、主节点选举、集群成员管理等场景。

------

## 44.2 znode 有哪些类型？

1. 持久节点
2. 临时节点
3. 持久顺序节点
4. 临时顺序节点

------

## 44.3 临时节点什么时候删除？

当客户端 Session 过期时，临时节点会自动删除。普通断连不会立刻删除，只要 Session 未过期并重新连接，临时节点还在。

------

## 44.4 Watcher 是什么？

Watcher 是 ZooKeeper 的事件通知机制。客户端在读取节点数据或子节点时注册 Watcher，节点变化后服务端通知客户端。

特点：

1. 一次性
2. 异步
3. 需要重复注册
4. 只通知变化，不携带完整数据

------

## 44.5 ZooKeeper 如何实现分布式锁？

使用临时顺序节点。

流程：

1. 在锁目录下创建临时顺序节点。
2. 获取所有子节点并排序。
3. 如果自己编号最小，获得锁。
4. 否则监听前一个节点。
5. 前一个节点删除后重新竞争。
6. 执行业务后删除自己的节点释放锁。

------

## 44.6 为什么监听前一个节点，而不是监听锁目录？

为了避免羊群效应。

如果所有客户端都监听锁目录，锁释放时所有客户端都会被唤醒。监听前一个节点可以保证每次只唤醒下一个等待者。

------

## 44.7 ZooKeeper 如何实现服务注册？

服务启动时创建临时节点，节点数据保存服务地址。消费者监听服务目录的子节点变化，服务挂掉后临时节点自动删除，消费者感知下线。

------

## 44.8 ZooKeeper 如何实现 Leader 选举？

所有候选者创建临时顺序节点，编号最小的成为 Leader。Leader 宕机后临时节点消失，后续编号最小者成为新 Leader。

------

## 44.9 ZooKeeper 是 CP 还是 AP？

一般认为 ZooKeeper 是 CP 系统。它优先保证一致性和分区容错性。当集群无法形成多数派时，不能继续正常处理写请求。

------

## 44.10 ZooKeeper 和 Redis 锁有什么区别？

ZooKeeper 锁一致性更强，更容易实现公平锁和自动释放。Redis 锁性能更高，但一致性和锁续期问题需要额外设计。

------

# 45. 推荐学习路线

第一阶段：基础使用

1. 安装 ZooKeeper
2. 熟悉 CLI
3. 理解 znode
4. 掌握 create、get、set、delete、ls
5. 理解持久节点、临时节点、顺序节点

第二阶段：Java API

1. 创建连接
2. CRUD
3. Watcher
4. Session
5. Stat
6. ACL
7. 异步 API
8. multi 事务

第三阶段：核心原理

1. Leader / Follower
2. zxid
3. Zab 协议思想
4. 多数派机制
5. Watcher 机制
6. Session 机制
7. 数据同步
8. Leader 选举

第四阶段：实战场景

1. 配置中心
2. 服务注册发现
3. 分布式锁
4. Leader 选举
5. 分布式队列
6. 分布式屏障

第五阶段：生产实践

1. Curator
2. 连接重试
3. Session 过期处理
4. ACL 安全配置
5. 监控指标
6. 四字命令
7. 性能调优
8. 故障恢复

------

# 46. 最小完整练习项目

建议你最后做一个小项目：

```text
zk-learning-demo
├── config-center
│   ├── ConfigWatcher.java
│   └── ConfigUpdater.java
├── service-discovery
│   ├── ServiceRegistry.java
│   └── ServiceDiscovery.java
├── distributed-lock
│   ├── ZkDistributedLock.java
│   └── LockTest.java
├── leader-election
│   └── LeaderElection.java
└── curator-demo
    ├── CuratorCrudDemo.java
    ├── CuratorLockDemo.java
    └── CuratorLeaderLatchDemo.java
```

你真正掌握 ZooKeeper 的标志不是会背概念，而是能自己实现这几个东西：

1. 用临时节点实现服务注册。
2. 用子节点 Watcher 实现服务发现。
3. 用临时顺序节点实现公平分布式锁。
4. 用临时顺序节点实现 Leader 选举。
5. 能解释 Session 过期对锁和注册中心的影响。
6. 能解释为什么 Watcher 是一次性的。
7. 能解释为什么监听前驱节点可以避免羊群效应。
8. 能解释 ZooKeeper 为什么通常部署奇数台。