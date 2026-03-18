# RocketMQ 面试题库

## 消息可靠性

### 如何保证消息不丢失？
**频率：高频**

三端保障：

生产者用同步发送 + 重试（`retryTimesWhenSendFailed=2`）或事务消息；Broker 开启同步刷盘（`flushDiskType=SYNC_FLUSH`）+ 主从同步复制（`brokerRole=SYNC_MASTER`）；消费者手动 ACK，消费成功再提交。

---

### RocketMQ 事务消息原理？
**频率：高频**

两阶段提交：

1. Producer 发 half 消息到 Broker（消费者不可见）
2. 执行本地事务
3. commit → 消息可消费；rollback → 删除

Broker 若长时间未收到确认，会回查 `TransactionListener.checkLocalTransaction()` 接口。

---

### 消息重复消费怎么处理（幂等）？
**频率：高频**

RocketMQ 是 at-least-once 语义，重复消费不可避免。业务侧保幂等：

1. 唯一 eventId 写 Redis（SETNX，TTL 7天）
2. 数据库唯一索引兜底
3. 状态机判断（已完成则跳过）

---

### Outbox 模式解决什么问题？
**频率：中频**

解决「DB 写成功但 MQ 发失败」的数据不一致。业务 DB 操作与 Outbox 记录在同一本地事务，定时扫描补发。RocketMQ 原生事务消息可覆盖大部分场景，Outbox 作为额外兜底层。

---

## 性能

### RocketMQ 为什么高性能？
**频率：高频**

1. 顺序写 CommitLog（磁盘顺序 IO 接近内存速度）
2. 零拷贝（mmap + sendfile）
3. ConsumeQueue 索引文件分离读写
4. 异步刷盘默认配置，吞吐可达十万级 TPS
5. 批量发送与消费

---

### 如何提升消费吞吐？
**频率：高频**

1. 增加消费者实例数，但不超过队列数（多余实例空跑）
2. 提高单实例并发线程数（`consumeThreadMax`）
3. 批量消费（`consumeMessageBatchMaxSize`）
4. 消费逻辑异步化
5. 适当增加 Queue 数

> 注意：消费者数量超过 Queue 数后多余实例永远分不到队列，是资源浪费。正确做法是先增加 Queue 数，再横向扩消费者实例。

---

### 同步刷盘 vs 异步刷盘怎么选？
**频率：中频**

| 模式 | 性能 | 可靠性 | 适用场景 |
|------|------|--------|----------|
| 异步刷盘（默认） | 高 | Broker 宕机可能丢最近数据 | 日志、行为数据 |
| 同步刷盘 | 约降 30% | 每条消息落盘才返回 ACK | 金融、积分、订单 |

---

### 消息堆积了怎么办？
**频率：中频**

1. 排查消费者是否有阻塞/异常
2. 临时扩容消费者实例
3. 对不重要的历史消息跳过（重置 offset 到最新）
4. 新建临时 Topic + 多消费者快速消费老堆积
5. 根本解：优化消费逻辑，降低单条处理时间

---

## 架构

### NameServer 的作用？为什么不用 ZooKeeper？
**频率：高频**

NameServer 是无状态的路由注册中心，Broker 定时上报心跳，Producer/Consumer 拉取路由表。

不用 ZooKeeper 的原因：

1. 无需强一致，最终一致即可
2. 单机简单无外部依赖
3. NameServer 之间无数据同步，天然 AP

---

### Topic、Queue、消费组的关系？
**频率：高频**

- 一个 Topic 分散在多个 Broker 上，每个 Broker 持有若干 Queue（默认 4 个）
- 消费组内多实例按 Queue 分配（负载均衡），每个 Queue 同时只被一个实例消费（消费组内有序）
- 不同消费组独立消费，互不影响

---

### 顺序消息如何实现？
**频率：中频**

**全局有序**：只用 1 个 Queue，牺牲吞吐。

**局部有序（推荐）**：同一业务 key（如 orderId）Hash 到同一个 Queue，消费者对该 Queue 串行处理。

代码上：send 时指定 `MessageQueueSelector`，消费时用 `MessageListenerOrderly`。

---

## 死信与监控

### 死信队列是什么？如何处理死信？
**频率：高频**

消息重试超过 `maxReconsumeTimes` 次后自动路由到 `%DLQ%{consumerGroup}`。

处理方式：

1. 订阅 DLQ Topic 持久化到 DB
2. 触发告警（钉钉/企业微信）
3. 运维界面支持重新投递或人工补偿
4. 根因分析修复消费者后批量重投

---

### 如何监控 RocketMQ 健康状态？
**频率：中频**

1. 消费者 lag（队列堆积数）—— 超阈值告警
2. Broker 磁盘使用率 > 85% 预警
3. 死信 Topic 有新消息立即报警
4. 发送失败率监控
5. 用结构化日志（`[MQ-METRICS]`）对接 ELK/Loki，无需 Prometheus

---

## 安全

### RocketMQ 有哪些安全配置？
**频率：中频**

1. **ACL**：配置 `aclEnable=true`，为每个 Producer/Consumer 分配 accessKey + secretKey，精确控制 Topic 读写权限
2. **TLS 加密传输**（4.x 起支持），防止消息被截获
3. **网络隔离**：Broker 不对外暴露，仅内网可达
4. NameServer 地址不写死，走配置中心

---

### 如何防止消息被非授权消费？
**频率：中频**

开启 ACL 后每个消费组只能订阅授权的 Topic + Tag。

代码侧：

1. 消息内容加密（AES），消费方持有密钥解密
2. 消息头携带签名，消费者验签
3. 不同业务域隔离到不同 Topic，禁止跨域订阅

---

### 敏感数据（如积分金额）在消息中如何保护？
**频率：低频**

1. 不在消息体中传明文敏感数据，只传业务 ID，消费方查 DB 获取详情
2. 必须传时对 payload 做 AES-256 对称加密
3. 消息落库（Outbox）时，message_body 字段加密存储或只存非敏感摘要
4. 定期轮换加密密钥

---

## 结合项目的加分回答

面试官问「你们怎么保证积分消息不丢」，可以这样回答：

> 生产者用 `sendSync` 同步发送，失败了写 `PointEventOutbox` 表，与业务操作在同一事务，定时扫描补发。Broker 侧用同步刷盘 + 主从同步复制，消费者 `onMessage` 里异常抛 `RuntimeException` 触发 RocketMQ 自动重试，超次数进 `%DLQ%` 死信 Topic，有单独的 `PointsDlqConsumer` 订阅死信触发钉钉告警。

这个回答涵盖了三端保障，比背书本的答案有说服力得多。
