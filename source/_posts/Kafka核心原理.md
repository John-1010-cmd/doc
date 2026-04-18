---
title: Kafka核心原理
date: 2024-10-05
updated : 2024-10-05
categories:
- Kafka
tags: 
  - Kafka
  - 原理
description: 深入理解 Kafka 核心架构，从分区机制到 Exactly-Once 语义的全面解析。
series: Kafka 深度解析
series_order: 1
---

Kafka 是一个分布式流处理平台，最初由 LinkedIn 开发，后成为 Apache 顶级项目。它以高吞吐、低延迟、高可用著称，广泛应用于消息队列、日志收集、实时数据管道、事件驱动架构等场景。本文从架构设计到存储机制，全面解析 Kafka 的核心原理。

## Kafka 整体架构

### 核心概念

Kafka 的架构围绕以下几个核心概念构建：

- **Broker**：Kafka 服务节点，多个 Broker 组成集群。Broker 负责消息的存储和转发。
- **Topic**：逻辑上的消息分类，生产者向 Topic 发送消息，消费者从 Topic 消费消息。
- **Partition**：Topic 的物理分片，每个 Partition 是一个有序的、不可变的追加日志。Partition 是 Kafka 并行度和扩展性的基本单位。
- **Consumer Group**：消费者组，组内消费者共同消费一个 Topic 的所有 Partition，每个 Partition 只能被组内一个消费者消费。

### 架构图

```
+----------+     +----------+     +----------+
| Producer |     | Producer |     | Producer |
+----+-----+     +----+-----+     +----+-----+
     |                |                |
     v                v                v
+---------------------------------------------+
|              Kafka Cluster                  |
|                                             |
|  +-------+  +-------+  +-------+           |
|  |Broker0|  |Broker1|  |Broker2|           |
|  +---+---+  +---+---+  +---+---+           |
|      |          |          |                |
|  Topic "order-events"                      |
|  +--------+  +--------+  +--------+       |
|  |P0(LF)  |  |P1(LF)  |  |P2(LF)  |       |
|  |  Leader|  |  Leader|  |  Leader|       |
|  +--------+  +--------+  +--------+       |
+---------------------------------------------+
     |                |                |
     v                v                v
+----------+     +----------+     +----------+
|Consumer0 |     |Consumer1 |     |Consumer2 |
+----------+     +----------+     +----------+
         Consumer Group "order-service"
```

### 生产者-消费者模型

```
Producer -> Topic (多个 Partition) -> Consumer Group (多个 Consumer)
```

生产者通过指定 Partition 策略将消息发送到特定 Partition，消费者组内的消费者按照分配策略消费对应的 Partition。

---

## 分区机制

### 分区策略

Kafka 生产者发送消息时，需要决定消息发送到哪个 Partition。分区策略如下：

#### 指定 Partition

```java
ProducerRecord<String, String> record = new ProducerRecord<>(
    "order-events",
    1,                    // 明确指定 Partition 编号
    "order-key",
    "order-payload"
);
producer.send(record);
```

#### Key 路由

当未指定 Partition 但消息有 Key 时，使用 Key 的哈希值对 Partition 数量取模：

```java
// Kafka 默认分区器源码（简化）
public int partition(String topic, Object key, byte[] keyBytes,
                     Object value, byte[] valueBytes, Cluster cluster) {
    List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
    int numPartitions = partitions.size();

    if (keyBytes == null) {
        return stickyPartitionCalculator.partition(topic, cluster);
    }

    // 使用 murmur2 哈希算法
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
}
```

**Key 路由的关键点**：
- 相同 Key 的消息始终进入同一个 Partition
- Partition 数量变化时，Key 的路由会改变
- 如果需要严格的消息顺序，必须使用 Key 路由

#### 无 Key 轮询（Sticky Partitioner）

Kafka 2.4 引入 Sticky Partitioner，改善无 Key 场景下的批量发送性能：

- 生产者将一批消息"粘"到同一个 Partition，积累到 `batch.size` 或 `linger.ms` 后发送
- 然后切换到下一个 Partition
- 相比传统 Round Robin，减少了小批量请求，提升了吞吐量

#### 自定义 Partitioner

```java
public class RegionPartitioner implements Partitioner {
    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (key instanceof String) {
            String region = extractRegion((String) key);
            switch (region) {
                case "north": return 0;
                case "south": return 1;
                case "east":  return 2;
                case "west":  return 3;
                default:      return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
            }
        }
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }

    @Override
    public void configure(Map<String, ?> configs) {}

    @Override
    public void close() {}
}
```

### 分区数量选择

分区数直接影响并行度和吞吐量，选择原则：

- 单个 Partition 的吞吐量约为 10MB/s（生产 + 消费）
- 目标吞吐量 / 单 Partition 吞吐量 = 最小 Partition 数
- 考虑消费者数量：Partition 数应大于等于最大消费者数
- 不宜过多：分区过多会增加 Leader 选举开销、文件句柄、内存占用

---

## 副本机制

### Leader 与 Follower

Kafka 通过副本（Replica）机制实现高可用：

- **Leader Replica**：处理该 Partition 的所有读写请求
- **Follower Replica**：从 Leader 异步复制数据，不处理客户端请求
- **ISR（In-Sync Replicas）**：与 Leader 保持同步的副本集合，包含 Leader 自身

```
Partition 0:
  Leader:   Broker-1 (ISR)
  Follower: Broker-2 (ISR)  -- 同步延迟在阈值内
  Follower: Broker-3 (OSR)  -- 同步延迟超过阈值，被踢出 ISR
```

### HW 与 LEO

理解 HW 和 LEO 是理解 Kafka 数据一致性的关键：

- **LEO（Log End Offset）**：每个副本日志的下一条消息偏移量（日志末端位置）
- **HW（High Watermark）**：所有 ISR 副本中最小的 LEO，消费者只能看到 HW 之前的消息

```
时间线：

Leader:      [0] [1] [2] [3] [4] [5]    LEO=6
Follower-1:  [0] [1] [2] [3] [4] [5]    LEO=6  (ISR)
Follower-2:  [0] [1] [2] [3]            LEO=4  (OSR)

HW = min(6, 6, 4) = 4  -> 消费者只能消费 offset 0-3 的消息
```

**HW 更新流程**：

1. Follower 从 Leader 拉取数据，携带自己的 LEO
2. Leader 收到所有 ISR 的 LEO 后，取最小值更新 HW
3. Follower 拉取数据时获取 Leader 的 HW，更新自己的 HW

**Leader 崩溃恢复**：
- 新 Leader 从 ISR 中选举（第一个存活的 ISR 副本）
- 新 Leader 的 LEO 可能低于旧 Leader 的 HW
- Kafka 通过 Leader Epoch 机制避免数据不一致

---

## 消费者组

### Rebalance 机制

Rebalance 是消费者组内的分区重新分配过程，触发条件：

1. 消费者加入或离开组
2. 消费者心跳超时（`session.timeout.ms`，默认 45s）
3. 订阅的 Topic Partition 数量变化
4. 订阅的 Topic 被创建或删除

**Rebalance 过程**：

```
1. 所有消费者向 Coordinator 发送 JoinGroupRequest
2. Coordinator 选出 Leader Consumer
3. Leader Consumer 根据分配策略计算分区分配方案
4. Leader 发送 SyncGroupRequest（携带分配方案）
5. 其他消费者发送 SyncGroupRequest
6. Coordinator 将分配结果下发给每个消费者
```

**Rebalance 的问题**：Rebalance 期间所有消费者暂停消费（Stop The World），频繁 Rebalance 会严重影响消费性能。

**避免不必要的 Rebalance**：
- 合理设置 `session.timeout.ms`（心跳超时）和 `heartbeat.interval.ms`（心跳间隔，建议为超时的 1/3）
- 设置 `max.poll.interval.ms` 大于消息处理最长时间
- 避免消费者频繁启停

### 分区分配策略

#### Range Assignor（默认）

按 Partition 编号范围连续分配：

```
Topic: order-events (7 个 Partition)
Consumer Group: 3 个消费者 (C0, C1, C2)

每个消费者分配: 7 / 3 = 2，余 1
C0: [0, 1, 2]  // 前面的消费者多分配一个
C1: [3, 4]
C2: [5, 6]
```

**问题**：当消费者订阅多个 Topic 时，编号靠前的消费者总是多分配，导致负载不均衡。

#### RoundRobin Assignor

将所有 Topic 的 Partition 逐一轮询分配：

```
TopicA: P0, P1, P2
TopicB: P0, P1

C0: TopicA-P0, TopicB-P1
C1: TopicA-P1, TopicB-P0
C2: TopicA-P2
```

**问题**：当消费者订阅不同 Topic 时，可能分配不均。

#### Sticky Assignor

满足两个目标：
1. 分配尽可能均匀
2. Rebalance 时保留原有的分配，尽量少移动

```
初始分配：
C0: [P0, P1, P2]
C1: [P3, P4, P5]
C2: [P6]

C2 离线后 Rebalance：
C0: [P0, P1, P2, P6]  // P6 原来是 C2 的
C1: [P3, P4, P5]       // 保持不变
```

---

## Offset 管理

### 自动提交 vs 手动提交

#### 自动提交

```java
Properties props = new Properties();
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "5000"); // 每 5 秒自动提交

// 风险：消息消费后、提交前崩溃 -> 重复消费
// 或提交后、消费前崩溃 -> 消息丢失
```

#### 手动提交

```java
props.put("enable.auto.commit", "false");

// 同步提交
consumer.commitSync();

// 异步提交
consumer.commitAsync((offsets, exception) -> {
    if (exception != null) {
        log.error("异步提交失败", exception);
        // 可选择重试或记录
    }
});

// 最佳实践：异步 + 同步组合
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        for (ConsumerRecord<String, String> record : records) {
            processMessage(record);
        }
        consumer.commitAsync(); // 正常情况异步提交（高性能）
    }
} finally {
    consumer.commitSync(); // 关闭前同步提交（确保不丢）
    consumer.close();
}
```

#### 指定 Offset 提交

```java
// 按分区粒度提交
Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
for (ConsumerRecord<String, String> record : records) {
    processMessage(record);
    offsets.put(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1)
    );
}
consumer.commitSync(offsets);
```

### __consumer_offsets

Kafka 使用内部 Topic `__consumer_offsets`（默认 50 个 Partition）存储消费者组的提交 Offset：

- Key：`group + topic + partition` 的哈希值
- Value：Offset 值 + 元数据（Leader Epoch、提交时间戳）
- 每个 Consumer Group 对应一个特定的 Partition（取 hash % 50）

---

## Exactly-Once 语义

Kafka 的消息传递语义有三个级别：

| 语义 | 说明 | 场景 |
|------|------|------|
| At Most Once | 最多一次，可能丢失 | 允许丢数据的日志 |
| At Least Once | 至少一次，可能重复 | 默认语义 |
| Exactly Once | 精确一次 | 金融交易、订单处理 |

### 幂等生产者

Kafka 0.11 引入幂等生产者，防止生产者重试导致消息重复：

```java
props.put("enable.idempotence", "true");
// 等价于同时设置：
// acks = all
// retries = Integer.MAX_VALUE
// max.in.flight.requests.per.connection = 5 (Kafka 3.0+ 自动调整)
```

**原理**：
- 每个 Producer 实例分配唯一的 `PID`（Producer ID）
- 每条消息携带 `<PID, Sequence Number>` 序号
- Broker 收到消息后检查序号：
  - 序号等于期望值 -> 接受
  - 序号小于期望值 -> 重复消息，丢弃
  - 序号大于期望值 -> 有消息丢失，异常

**限制**：幂等性仅保证单 Partition 内、单 Producer Session 内的去重，跨 Session 或跨 Partition 不保证。

### 事务

Kafka 事务支持跨多个 Partition 的原子写入：

```java
props.put("transactional.id", "order-tx-producer-1");

KafkaProducer<String, String> producer = new KafkaProducer<>(props);
producer.initTransactions();

try {
    producer.beginTransaction();

    // 向多个 Topic/Partition 发送消息（原子性）
    producer.send(new ProducerRecord<>("orders", "order-1", "{...}"));
    producer.send(new ProducerRecord<>("payments", "pay-1", "{...}"));

    // 提交消费者的 Offset（Consume-Transform-Produce 模式）
    producer.sendOffsetsToTransaction(offsets, consumerGroupId);

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

**事务协调流程**：

```
Producer -> TransactionCoordinator (特殊的 Broker 节点)
    1. initTransactions() -> 获取 PID，恢复或初始化事务状态
    2. beginTransaction() -> 本地标记事务开始
    3. send() -> 向 TransactionCoordinator 注册目标 Partition
    4. commitTransaction() ->
        a. TransactionCoordinator 写入 PREPARE_COMMIT 到 __transaction_state
        b. 向每个目标 Partition 写入 COMMIT 标记
        c. TransactionCoordinator 写入 COMPLETE_COMMIT
```

### Read Committed

消费者端通过 `isolation.level` 控制事务可见性：

```java
props.put("isolation.level", "read_committed");
// 只消费已提交的事务消息
// 未提交的消息和 Abort 的消息被过滤

// 默认为 read_uncommitted
// 消费所有消息，包括未提交的事务消息
```

Kafka 通过在日志中写入 **Control Batch**（控制消息）来标记事务的提交或中止。消费者在读取时跳过 Abort 的事务消息。

---

## 存储机制

### 日志分段

每个 Partition 在磁盘上对应一个目录，目录下包含多个日志分段（LogSegment）：

```
/var/kafka/data/order-events-0/
    00000000000000000000.log    # 消息数据文件
    00000000000000000000.index  # 偏移量索引文件
    00000000000000000000.timeindex # 时间戳索引文件
    00000000000005367851.log    # 第二个分段（baseOffset = 5367851）
    00000000000005367851.index
    00000000000005367851.timeindex
```

- 每个 LogSegment 由 `.log`（数据）、`.index`（偏移量索引）、`.timeindex`（时间戳索引）三个文件组成
- 当 `.log` 文件达到 `log.segment.bytes`（默认 1GB）或 `log.segment.ms` 时间阈值时，创建新分段
- 文件名即该分段的 Base Offset

### 稀疏索引

Kafka 使用稀疏索引而非稠密索引，每写入 4KB 数据（`log.index.interval.bytes`）才记录一条索引项：

```
偏移量索引 (.index):
    Offset     Position
    0          0
    128        1372      <-- 不是每条消息都有索引
    256        2744
    ...

查找 offset=200 的消息:
    1. 二分查找 index 文件，找到 offset <= 200 的最大索引项 (offset=128, pos=1372)
    2. 从 log 文件的 pos=1372 开始顺序扫描
    3. 最多扫描 4KB 数据即可找到目标消息
```

**稀疏索引的优势**：索引文件小，可以常驻内存，查找速度快。

### 零拷贝

Kafka 使用零拷贝技术大幅提升消息传输性能。

#### 传统数据发送（4 次拷贝）

```
磁盘 -> 内核缓冲区 -> 用户空间缓冲区 -> Socket 缓冲区 -> 网卡
          (DMA)          (CPU拷贝)          (CPU拷贝)       (DMA)
```

#### 零拷贝（2 次拷贝）

```java
// Kafka 使用 Java FileChannel.transferTo() -> 底层调用 sendfile()
FileChannel fileChannel = new FileInputStream("00000000000000000000.log").getChannel();
fileChannel.transferTo(position, count, socketChannel);

// Linux sendfile() 系统调用
// 磁盘 -> 内核缓冲区 -> 网卡 (直接由 DMA 传输到网卡)
```

```
磁盘 -> 内核 PageCache -> 网卡
          (DMA)           (DMA)
```

**减少的两次拷贝**：内核缓冲区 -> 用户空间 -> Socket 缓冲区，全程零 CPU 拷贝。

---

## 高性能设计

### 批量发送

Kafka 生产者将消息批量发送，而非逐条发送：

```java
props.put("batch.size", 16384);      // 批量大小（字节），默认 16KB
props.put("linger.ms", 5);           // 等待时间（毫秒），默认 0
props.put("buffer.memory", 33554432); // 缓冲区总大小，默认 32MB

// 工作流程：
// 1. 消息先写入 RecordAccumulator 的对应 Partition 批次中
// 2. 批次满了 (batch.size) 或超时 (linger.ms) 后发送
// 3. Sender 线程将多个批次合并为一个请求发送给 Broker
```

**最佳实践**：
- `linger.ms` 设为 5-10ms，小幅增加延迟换取大幅提升的吞吐
- `batch.size` 根据消息大小调整，目标是一批消息在 16KB-1MB 之间

### 压缩

Kafka 支持在 Producer 端压缩整个 Batch，Broker 直接存储压缩后的数据：

```java
props.put("compression.type", "lz4");
// 支持: none, gzip, snappy, lz4, zstd
```

| 压缩算法 | 压缩比 | 压缩速度 | 解压速度 | 适用场景 |
|---------|--------|---------|---------|---------|
| none | 1:1 | - | - | 网络带宽充足 |
| gzip | 高 | 慢 | 中 | 带宽受限、冷数据 |
| snappy | 中 | 快 | 快 | 实时处理 |
| lz4 | 中 | 最快 | 最快 | 低延迟场景 |
| zstd | 较高 | 中 | 中 | 平衡型 |

### PageCache

Kafka 大量利用操作系统的 PageCache：

- **写入**：消息先写入 PageCache，由 OS 异步刷盘（而非每条消息 fsync）
- **读取**：直接从 PageCache 读取（命中率高，因为写入后通常很快被消费）
- **优势**：避免 JVM GC 问题，利用 OS 的 LRU 缓存策略，进程重启后缓存不丢失

**配置建议**：
- `log.flush.interval.messages`：不建议修改（让 OS 管理刷盘）
- 避免给 Broker 分配过大堆内存，留更多内存给 PageCache

### 顺序写

Kafka 的消息追加采用顺序写，这是磁盘最高效的写入方式：

- 顺序写性能接近内存写入（约 600MB/s on SSD）
- 随机写性能极差（约 100KB/s on HDD）
- Kafka 的 Partition 就是一个 Append-Only Log，天然顺序写

```
传统数据库：随机写（B+ Tree 更新） -> 性能瓶颈
Kafka：顺序写（追加日志）         -> 极高性能
```

---

## 总结

Kafka 的高性能和高可靠性不是单一技术实现的，而是多种技术的组合：

| 维度 | 技术手段 |
|------|---------|
| 高吞吐 | 批量发送、压缩、PageCache、顺序写、零拷贝 |
| 高可用 | 副本机制、ISR、Leader Election |
| 消息顺序 | Partition 内有序、Key 路由 |
| 精确一次 | 幂等生产者、事务、Read Committed |
| 扩展性 | Partition 分片、Consumer Group 并行消费 |

理解这些核心原理有助于在实际工作中做出正确的架构决策，如 Partition 数量规划、消费者参数调优、消息可靠性保障等。
