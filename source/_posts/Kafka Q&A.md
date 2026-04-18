---
title: Kafka Q&A
date: 2024-10-20
updated : 2024-10-20
categories:
- Kafka
tags: 
  - Kafka
description: Kafka 高频面试题整理，覆盖架构、存储、消费、可靠性等核心问题。
series: Kafka 深度解析
series_order: 2
---

本文整理了 Kafka 高频面试题，覆盖架构设计、存储机制、消费模型、可靠性保障等核心知识点。每道题目从问题本质出发，结合原理进行解答。

## Kafka 如何保证消息不丢失？

消息在 Kafka 中的流转经历三个阶段：生产者发送、Broker 存储、消费者消费。需要在每个阶段分别保障。

1. **生产者端**：
   - 设置 `acks=all`（或 `acks=-1`），要求 Leader 收到消息后等待所有 ISR 副本确认才算写入成功
   - 设置 `retries` 为合理值（配合幂等生产者可设为 `Integer.MAX_VALUE`）
   - 使用回调（Callback）确认发送结果，失败时进行重试或记录
   - 设置 `min.insync.replicas=2`，确保至少有 2 个副本确认写入

2. **Broker 端**：
   - 设置 `replication.factor >= 3`，每个 Partition 至少 3 个副本
   - 设置 `min.insync.replicas=2`，配合 `acks=all` 使用
   - 设置 `unclean.leader.election.enable=false`，禁止非 ISR 副本成为 Leader（防止数据丢失）
   - 适当调整 `log.flush.interval.messages`（一般让 OS 管理）

3. **消费者端**：
   - 关闭自动提交（`enable.auto.commit=false`）
   - 消息处理完成后再手动提交 Offset
   - 使用 `commitSync()` 确保提交成功

```java
// 生产者配置示例
props.put("acks", "all");
props.put("enable.idempotence", "true");
props.put("retries", Integer.MAX_VALUE);

// 消费者配置示例
props.put("enable.auto.commit", "false");
```

## Kafka 如何保证消息顺序性？

Kafka 只保证** Partition 内消息的局部有序**，不保证 Topic 级别的全局有序。

1. **Partition 内有序**：消息按写入顺序追加到日志中，消费者按 Offset 顺序消费，天然有序
2. **保证同一业务实体的消息进入同一 Partition**：通过 Key 路由实现
   - 将订单 ID 作为消息 Key，相同订单的消息始终路由到同一 Partition
   - `partition = hash(key) % numPartitions`
3. **单 Partition 单消费者**：Consumer Group 内每个 Partition 只能被一个消费者消费
4. **全局有序的特殊场景**：将 Topic 的 Partition 数设为 1，所有消息都在一个 Partition 内，但牺牲了并行度

```java
// 通过 Key 保证同一订单的消息有序
producer.send(new ProducerRecord<>("orders", orderId, orderEvent));
```

**注意**：如果需要多 Partition 的全局有序，需要在应用层实现（如使用时间戳排序或版本号机制），但这会牺牲性能和实时性。

## Kafka 的 Rebalance 机制？

Rebalance 是消费者组内 Partition 重新分配的过程。

1. **触发条件**：
   - 消费者加入消费者组（新实例启动）
   - 消费者离开消费者组（实例停止或崩溃）
   - 消费者心跳超时（超过 `session.timeout.ms` 未发送心跳）
   - 消费者处理超时（超过 `max.poll.interval.ms` 未调用 poll）
   - 订阅的 Topic Partition 数变化
   - 订阅的 Topic 被创建或删除

2. **Rebalance 流程**：
   - 所有消费者向 GroupCoordinator 发送 JoinGroupRequest
   - Coordinator 选出 Leader Consumer（第一个加入组的消费者）
   - Leader Consumer 根据分配策略计算分区方案
   - Leader 通过 SyncGroupRequest 将方案发送给 Coordinator
   - Coordinator 将分配结果下发给每个消费者
   - 消费者按新分配开始消费

3. **Rebalance 的问题**：
   - Rebalance 期间所有消费者暂停消费（STW）
   - 频繁 Rebalance 严重影响消费性能
   - 分区重新分配导致重复消费

4. **避免不必要 Rebalance 的措施**：
   - `session.timeout.ms` 设置为 25s（默认），`heartbeat.interval.ms` 设置为其 1/3
   - `max.poll.interval.ms` 设置为大于最长处理时间
   - 避免消费者频繁启停
   - 使用 CooperativeStickyAssignor 减少每次 Rebalance 的分区移动

## Kafka 为什么这么快？

Kafka 的高性能是多种技术手段共同作用的结果。

1. **顺序写**：Partition 是 Append-Only Log，消息只追加到文件末尾。磁盘顺序写速度约 600MB/s（SSD），接近内存写入速度，远高于随机写的 100KB/s（HDD）

2. **零拷贝**：使用 Linux `sendfile()` 系统调用，数据从磁盘直接传输到网卡，跳过用户空间拷贝。将传统的 4 次数据拷贝减少为 2 次

3. **PageCache**：利用操作系统页缓存，写入时先写入 PageCache 由 OS 异步刷盘，读取时直接命中 PageCache。避免 JVM GC 问题，进程重启后缓存不丢失

4. **批量发送**：生产者将消息攒批发送（`batch.size` + `linger.ms`），减少网络请求次数。一个 Batch 内的消息共用 Header 信息，减少协议开销

5. **压缩**：在 Producer 端压缩整个 Batch（而非单条消息），Broker 直接存储压缩数据。支持 gzip、snappy、lz4、zstd 等算法

6. **稀疏索引**：每 4KB 数据记录一条索引项，索引文件小可常驻内存，查找时先二分定位索引再小范围扫描

7. **分区并行**：Topic 分为多个 Partition，分布在不同 Broker 上，生产者和消费者可以并行读写

8. **Reactor 网络模型**：Broker 使用 1 个 Acceptor 线程 + N 个 Processor 线程 + M 个 Handler 线程池，高效处理网络请求

## Kafka 与 RocketMQ 的区别？

两者都是优秀的分布式消息队列，设计理念和适用场景有所不同。

1. **架构设计**：
   - Kafka：无 NameServer 概念，Broker 通过 ZooKeeper/KRaft 协调。Topic 分为 Partition
   - RocketMQ：NameServer 独立部署，Broker 分为 Master/Slave。Topic 分为 MessageQueue

2. **消息模型**：
   - Kafka：基于 Topic-Partition 模型，Consumer Group 消费
   - RocketMQ：支持 Topic-Queue 模型，同时原生支持 Tag 过滤

3. **事务消息**：
   - Kafka：通过事务协调器实现跨 Partition 原子写入
   - RocketMQ：通过半消息 + 本地事务回查机制实现，对业务侵入更小

4. **延迟消息**：
   - Kafka：原生不支持延迟消息，需要自行实现
   - RocketMQ：原生支持 18 个延迟等级（1s-2h）

5. **消息过滤**：
   - Kafka：只能通过 Consumer 端代码过滤
   - RocketMQ：支持 Broker 端 Tag 和 SQL92 过滤，减少网络传输

6. **消息回溯**：
   - Kafka：支持按 Offset 和时间戳回溯
   - RocketMQ：支持按时间戳回溯

7. **性能特点**：
   - Kafka：在大数据量、高吞吐场景下表现更好（批处理优化更极致）
   - RocketMQ：在线业务消息场景下延迟更低（同步刷盘、同步主从复制）

8. **适用场景**：
   - Kafka：日志收集、实时数据管道、流处理（大数据生态集成好）
   - RocketMQ：电商交易消息、金融业务消息（功能更丰富）

## Kafka 事务消息原理？

Kafka 的事务机制支持跨多个 Partition 的原子写入，主要用于 Consume-Transform-Produce 模式。

1. **核心组件**：
   - `transactional.id`：Producer 的全局唯一事务标识，用于跨 Session 恢复事务状态
   - `TransactionCoordinator`：负责管理事务状态的 Broker 节点
   - `__transaction_state`：内部 Topic，存储事务状态日志（类似 Commit Log）
   - `PID`（Producer ID）：每个 Producer 实例的唯一标识，配合 Sequence Number 实现幂等

2. **事务流程**：
   - `initTransactions()`：向 TransactionCoordinator 注册 `transactional.id`，获取或恢复 PID
   - `beginTransaction()`：Producer 本地标记事务开始
   - `send()`：发送消息到目标 Partition，同时向 Coordinator 注册该 Partition
   - `commitTransaction()`：
     a. Coordinator 写入 PREPARE_COMMIT 到 `__transaction_state`
     b. 向每个参与的 Partition 写入 COMMIT 标记（Control Batch）
     c. Coordinator 写入 COMPLETE_COMMIT
   - `abortTransaction()`：类似 commit，但写入 ABORT 标记

3. **消费者端**：
   - `isolation.level=read_committed`：只消费已提交事务的消息
   - Broker 通过 Control Batch 标记过滤未提交或中止的事务消息

4. **与 RocketMQ 事务消息的区别**：
   - Kafka 事务：解决流处理中跨 Partition 原子写入问题
   - RocketMQ 事务：解决本地事务与消息发送的一致性问题（半消息 + 回查）

## Kafka 消息积压如何处理？

消息积压是生产环境的常见问题，需要从发现、应急、根治三个层面处理。

1. **发现积压**：
   - 监控 `RecordsLag`（消费者延迟量）
   - 监控 `ConsumerLag`（Kafka Consumer Lag 指标）
   - 使用 Kafka Manager / CMAK / Burrow 等工具
   - 通过 `kafka-consumer-groups.sh --describe` 查看

2. **应急处理**：
   - **增加消费者实例**：确保消费者数量等于 Partition 数量（超过 Partition 数无意义）
   - **增加 Partition**：如果消费者已满配，增加 Topic 的 Partition 数量（注意 Key 路由变化）
   - **临时扩消费线程**：每个消费者内部增加处理线程
   - **消息转发**：将积压消息转发到一个有更多 Partition 的新 Topic，用更多消费者消费
   - **跳过非关键消息**：对于日志类非关键消息，可设置消费者从最新 Offset 开始消费

3. **长期优化**：
   - 优化消费者处理逻辑（批量处理、异步化、减少 IO）
   - 合理设置 Partition 数量（满足峰值吞吐）
   - 生产者端限流或降级
   - 使用背压（Backpressure）机制

```bash
#​ 查看消费者组延迟
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe --group order-service

#​ 输出示例：
#​ TOPIC           PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
#​ order-events    0          12345           15678           3333
#​ order-events    1          12200           15500           3300
```

## Kafka 分区数如何设定？

分区数的选择需要综合考虑吞吐量、消费者并行度、Broker 负载等因素。

1. **基于吞吐量计算**：
   - 评估单 Partition 的生产吞吐（约 10MB/s）和消费吞吐
   - 目标吞吐量 / 单 Partition 吞吐量 = 最小 Partition 数
   - 再预留 20%-30% 余量

2. **基于消费者数量**：
   - Partition 数应大于等于最大消费者数
   - 多余的消费者会处于空闲状态

3. **不宜过多的原因**：
   - 每个 Partition 占用文件句柄和内存
   - Partition 越多，Leader Election 时间越长
   - 增加 Broker 的恢复时间
   - 影响可用性（单个 Broker 承载更多 Partition）

4. **经验值**：
   - 小规模集群（3-5 Broker）：每个 Topic 6-12 个 Partition
   - 中等规模（10+ Broker）：每个 Topic 30-60 个 Partition
   - 单个 Broker 建议 Partition 总数不超过 1000-2000

5. **分区数调整**：
   - 只能增加，不能减少
   - 增加后 Key 的路由会变化（hash % numPartitions 结果改变）
   - 如需保证 Key 顺序性，需提前规划好分区数

## Kafka 的零拷贝原理？

零拷贝技术减少或消除数据在内核空间和用户空间之间的拷贝操作。

1. **传统数据传输（4 次拷贝 + 4 次上下文切换）**：
   ```
   应用调用 read():
     磁盘 -> 内核 PageCache (DMA 拷贝)
     内核 PageCache -> 用户缓冲区 (CPU 拷贝)
   应用调用 write():
     用户缓冲区 -> Socket 缓冲区 (CPU 拷贝)
     Socket 缓冲区 -> 网卡 (DMA 拷贝)
   ```

2. **零拷贝 sendfile()（2 次拷贝 + 2 次上下文切换）**：
   ```
   应用调用 sendfile():
     磁盘 -> 内核 PageCache (DMA 拷贝)
     内核 PageCache -> 网卡 (DMA 拷贝，CPU 不参与)
   ```

3. **Kafka 中的应用**：
   - Kafka Broker 读取消息时调用 `FileChannel.transferTo()`
   - 底层在 Linux 上调用 `sendfile()` 系统调用
   - 在 Java NIO 中，`transferTo()` 直接委托给操作系统的零拷贝机制
   - 消费者拉取消息时，Broker 无需将消息读入 JVM 堆内存，直接从 PageCache 传输到网卡

4. **前提条件**：
   - 操作系统支持 `sendfile()`（Linux 2.4+）或 `mmap()`
   - 数据已缓存在 PageCache 中（Kafka 的消息通常刚写入就被消费，命中率极高）

## Kafka 如何实现 Exactly-Once？

Kafka 的 Exactly-Once 语义通过三个层面协同实现。

1. **幂等生产者（单 Partition 去重）**：
   - 每个 Producer 分配唯一的 PID（Producer ID）
   - 每条消息携带 `<PID, Partition, SequenceNumber>`
   - Broker 检查序号去重：期望值则接受，小于期望值则丢弃重复消息
   - 配置：`enable.idempotence=true`
   - 限制：仅保证单 Partition、单 Producer Session 内去重

2. **事务（跨 Partition 原子写入）**：
   - 通过 `transactional.id` 标识事务 Producer
   - TransactionCoordinator 协调事务状态
   - 事务日志存储在 `__transaction_state` 内部 Topic
   - 支持跨多个 Topic/Partition 的原子写入
   - 支持 Consume-Transform-Produce 模式的 Exactly-Once

3. **消费者端 Read Committed**：
   - 设置 `isolation.level=read_committed`
   - Broker 通过 Control Batch 标记过滤未提交或中止的事务消息
   - 消费者只看到已成功提交的事务消息

4. **完整配置示例**：

```java
// 生产者
props.put("enable.idempotence", "true");
props.put("transactional.id", "tx-producer-1");

// 消费者
props.put("isolation.level", "read_committed");
props.put("enable.auto.commit", "false");

// 事务代码模板
producer.initTransactions();
try {
    producer.beginTransaction();
    // 消费消息 -> 处理 -> 生产新消息
    producer.send(new ProducerRecord<>("output-topic", key, value));
    // 提交消费 Offset（与消息写入在同一个事务中）
    producer.sendOffsetsToTransaction(offsets, groupId);
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}
```

5. **Exactly-Once 的边界**：
   - Kafka 内部的 Exactly-Once：完全保证
   - Kafka 到外部系统的 Exactly-Once：需要外部系统支持幂等或事务
   - 常见模式：Kafka -> 处理 -> Kafka（Exactly-Once） vs Kafka -> 处理 -> MySQL（需要应用层幂等）
