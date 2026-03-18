# 消息中间件技术选型

**Kafka vs RocketMQ | 积分消息 + 订单消息**
版本：v1.0 | 日期：2026-03-18

---

## 1. 背景与约束

当前有两个业务服务需要引入消息中间件，需共用同一套 MQ 集群：

| 服务 | JDK 版本 | 框架 | 业务需求 |
|------|----------|------|----------|
| 积分服务 | JDK 8 | Spring Boot 2.x | 积分授予、扣减、过期、兑换事件 |
| 订单服务 | JDK 17 | Spring Boot 3.x | 订单创建、支付、发货、完成、取消事件 |

核心约束：

- 两个服务共用同一个 MQ Broker，不单独部署
- JDK 版本不升降，各服务保持现有运行环境
- 积分场景要求消息不丢，需支持事务消息或 Outbox 模式
- 订单场景需多下游订阅（仓储、物流、BI），要求消息可回溯

---

## 2. JDK 兼容性分析

> Broker 的运行 JDK 与业务服务的客户端 JDK 完全独立。两个方案均可支持 JDK8 / JDK17 客户端同时连接同一个 Broker。

| 中间件 | Broker 运行 JDK | 客户端 JDK8 | 客户端 JDK17 | 说明 |
|--------|----------------|-------------|--------------|------|
| Kafka 3.9 | JDK 11 / 17 / 21 | ✅ 支持 | ✅ 支持 | kafka-clients 编译目标 Java 8 |
| Kafka 4.0+ | JDK 11 / 17 / 21 | ❌ 不支持 | ✅ 支持 | 已移除 JDK8 支持，不可选 |
| RocketMQ 5.x | JDK 8（推荐） | ✅ 支持 | ✅ 支持 | 新版 gRPC 客户端编译目标 Java 8 |

---

## 3. Docker 镜像选型

| 方案 | 推荐镜像 | 对应版本 | 说明 |
|------|----------|----------|------|
| **Kafka（推荐）** | `confluentinc/cp-kafka:7.9.0` | Apache Kafka 3.9 | Confluent Platform 7.9 基于 Kafka 3.9 构建，Broker 内置 JDK21 |
| Kafka（当前） | `confluentinc/cp-kafka:7.6.1` | Apache Kafka 3.6 | 当前使用版本，可平滑升级到 7.9.0 |
| RocketMQ | `apache/rocketmq:5.3.1` | RocketMQ 5.3.1 | 官方镜像，Broker 建议 JDK8 运行 |

---

## 4. 核心功能对比

| 特性 | Kafka 3.9 | RocketMQ 5.x |
|------|-----------|--------------|
| 消息语义 | at-least-once | at-least-once |
| 事务消息 | 有限（Exactly-once 语义） | ✅ 原生支持（两阶段提交） |
| 延迟消息 | ❌ 不支持，需外部调度 | ✅ 原生支持（18 个级别） |
| 消息回溯 | ✅ 支持（按时间 / offset） | ✅ 支持 |
| 多下游订阅 | ✅ 多消费组隔离，各自维护 offset | ✅ 多消费组隔离 |
| 死信队列 | 需手动实现 | ✅ 原生 `%DLQ%{consumerGroup}` Topic |
| 消息顺序 | 分区内有序 | 队列内有序 |
| 吞吐量 | 百万级 TPS | 十万级 TPS |
| 流处理生态 | ✅ Flink / Spark / Kafka Streams 原生集成 | 无 |
| 运维复杂度 | KRaft 模式，无需 ZooKeeper | 需 NameServer |
| ACL 权限 | ✅ 支持 | ✅ 支持 |
| TLS 加密 | ✅ 支持 | ✅ 支持（4.x 起） |

---

## 5. 选型结论

**最终选择：Kafka 3.9 | `confluentinc/cp-kafka:7.9.0`**

选择理由：

1. **JDK 兼容**：两个服务（JDK8 / JDK17）均使用 `kafka-clients 3.9.x`，零改造成本
2. **多下游订阅**：订单消息需仓储、物流、BI 等多个下游独立消费，Kafka 消费组隔离机制天然适配
3. **消息回溯**：BI 服务可按时间重跑历史数据，无需重新推送
4. **运维简洁**：Kafka 3.9 KRaft 模式无 ZooKeeper 依赖，部署和运维更简单
5. **生态扩展**：未来接入 Flink / Spark 流处理无缝集成
6. **积分可靠性**：通过 Outbox 模式 + 幂等消费完整覆盖不丢消息场景

> **备注**：积分场景的延迟消息（批量过期）和原生事务消息是 RocketMQ 的优势点。若后续积分业务有大量延迟消息需求，可评估引入 RocketMQ 作为补充，按业务域拆分部署。

---

## 6. 升级方案（7.6.1 → 7.9.0）

### 6.1 镜像变更

| | 旧版本 | 新版本 |
|-|--------|--------|
| Docker 镜像 | `confluentinc/cp-kafka:7.6.1` | `confluentinc/cp-kafka:7.9.0` |
| Kafka 版本 | Apache Kafka 3.6 | Apache Kafka 3.9 |
| Broker 内置 JDK | JDK 11 | JDK 21 |

### 6.2 客户端依赖变更

**积分服务（JDK8，Spring Boot 2.x）**

```xml
<!-- 显式覆盖 Spring Boot 内置的 kafka-clients 版本 -->
<properties>
    <kafka.version>3.9.0</kafka.version>
    <spring-kafka.version>2.9.13</spring-kafka.version>
</properties>

<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.9.0</version>
</dependency>
```

**订单服务（JDK17，Spring Boot 3.x）**

```xml
<!-- Spring Boot 3.x 默认拉取旧版 kafka-clients，需显式覆盖 -->
<properties>
    <kafka.version>3.9.0</kafka.version>
</properties>

<dependency>
    <groupId>org.apache.kafka</groupId>
    <artifactId>kafka-clients</artifactId>
    <version>3.9.0</version>
</dependency>
```

### 6.3 JDK17 启动参数

JDK17 对模块系统访问限制更严，需在启动参数中添加：

```
--add-opens java.base/sun.nio.ch=ALL-UNNAMED
--add-opens java.base/java.nio=ALL-UNNAMED
```

Spring Boot 项目可配置在 `JAVA_TOOL_OPTIONS` 环境变量或 `pom.xml` 的 `maven-surefire-plugin` 中。

### 6.4 Broker 配置注意事项

从 7.6 升级到 7.9，若当前使用 ZooKeeper 模式，镜像版本直接替换即可，无需改动 Broker 配置。

如需切换到 KRaft 模式（推荐），去掉 ZooKeeper 容器，替换以下环境变量：

```yaml
# 移除
KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181

# 新增 KRaft 配置
KAFKA_PROCESS_ROLES: broker,controller
KAFKA_NODE_ID: 1
KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
CLUSTER_ID: <生成一个唯一 UUID>
```

---

## 7. 积分消息方案设计

### 7.1 Topic 与 Tags 规划

| Topic | Tags | 说明 |
|-------|------|------|
| `points-event-topic` | `points_granted` | 积分授予 |
| `points-event-topic` | `points_revoked` | 积分撤销 |
| `points-event-topic` | `points_expired` | 积分过期 |
| `points-event-topic` | `points_redeemed` | 积分兑换 |

> Kafka 无原生 Tags 过滤，统一由 `eventType` 字段在消费者侧路由。

### 7.2 可靠性保障（Outbox 模式）

```
业务操作 + 写 PointEventOutbox   ←── 同一本地事务
        ↓
定时扫描 PENDING 记录
        ↓
发送到 Kafka（kafka-clients syncSend）
        ↓
发送成功 → 标记 SENT
发送失败 → 指数退避重试（10s → 30s → 120s → 600s）
超过最大重试次数 → 标记 FAILED，触发告警
```

### 7.3 幂等消费

消费者处理消息前，以 `eventId` 为 key 做 Redis SETNX（TTL 7 天）：

- 首次消费：Redis 返回 true，正常处理
- 重复消费：Redis 返回 false，直接跳过并 ACK

---

## 8. 订单消息方案设计

### 8.1 Topic 规划

| Topic | 消费组 | 消费方 | 说明 |
|-------|--------|--------|------|
| `order-event-topic` | `order-warehouse-group` | 仓储服务 | 监听 `ORDER_PAID`、`ORDER_CANCELLED` |
| `order-event-topic` | `order-logistics-group` | 物流服务 | 监听 `ORDER_SHIPPED` |
| `order-event-topic` | `order-bi-group` | BI 服务 | 监听全部事件，支持回溯重跑 |
| `order-event-topic` | `order-points-group` | 积分服务 | 监听 `ORDER_COMPLETED`，触发积分授予 |

### 8.2 多下游消费隔离

每个下游服务使用独立消费组，各自维护消费 offset，互不影响：

- 仓储服务消费失败不影响物流服务消费进度
- BI 服务可将 offset 重置到任意历史时间点，独立重跑数据
- 新增下游服务只需新建消费组，无需修改生产者

### 8.3 消息回溯

```bash
# BI 服务需要重跑昨天的订单数据时，重置 offset 到昨天 0 点
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group order-bi-group \
  --topic order-event-topic \
  --reset-offsets \
  --to-datetime 2026-03-17T00:00:00.000 \
  --execute
```

---

## 9. 安全配置要求

### 9.1 ACL 权限控制

生产环境必须开启 ACL，为每个 Producer / Consumer 配置最小权限：

```properties
# Broker 开启 ACL
authorizer.class.name=kafka.security.authorizer.AclAuthorizer
allow.everyone.if.no.acl.found=false
```

```bash
# 授予积分服务生产者权限
kafka-acls.sh --add --allow-principal User:points-producer \
  --operation Write --topic points-event-topic

# 授予 BI 服务消费者权限
kafka-acls.sh --add --allow-principal User:bi-consumer \
  --operation Read --topic order-event-topic \
  --group order-bi-group
```

### 9.2 敏感数据保护

- 消息体中不传明文敏感数据（积分金额、订单金额等），只传业务 ID，消费方查 DB 获取详情
- 必须传输时对 payload 做 AES-256 加密
- Outbox 表的 `message_body` 字段存储加密后的内容或只存非敏感摘要

---

## 10. 监控与告警

### 10.1 关键监控指标

| 指标 | 告警阈值 | 说明 |
|------|----------|------|
| 消费者 lag（堆积数） | > 10000 条持续 5 分钟 | 消费者可能阻塞或异常 |
| Broker 磁盘使用率 | > 85% | 预警，避免磁盘写满 |
| 发送失败率 | > 1% | 生产者侧异常 |
| Outbox FAILED 记录 | > 0 | 需人工介入处理 |

### 10.2 结构化日志格式

统一使用以下格式输出指标日志，便于 ELK / Loki 解析：

```
[MQ-METRICS] action=send_ok   topic=xxx eventType=yyy cost_ms=N total=N
[MQ-METRICS] action=send_fail topic=xxx eventType=yyy total=N
[MQ-METRICS] action=consume_ok topic=xxx eventType=yyy cost_ms=N total=N
[MQ-METRICS] action=dlq       topic=xxx total=N
[MQ-METRICS] action=outbox_pending topic=xxx count=N
```

### 10.3 告警渠道

告警通过钉钉 Webhook 发送，配置：

```yaml
mq:
  alert:
    dingtalk-webhook: https://oapi.dingtalk.com/robot/send?access_token=xxx
```

未配置 webhook 时自动降级为只打日志，不影响业务。

---

## 11. 面试高频问题

### 消息可靠性

**Q：如何保证消息不丢失？**

三端保障：生产者用同步发送 + 重试 + Outbox 兜底；Broker 开启同步刷盘（`flushDiskType=SYNC_FLUSH`）+ 主从同步复制；消费者手动 ACK，消费成功再提交。

**Q：消息重复消费怎么处理（幂等）？**

Kafka 是 at-least-once 语义，重复消费不可避免。业务侧保幂等：唯一 eventId 写 Redis（SETNX，TTL 7 天）；数据库唯一索引兜底；状态机判断（已完成则跳过）。

**Q：Outbox 模式解决什么问题？**

解决「DB 写成功但 MQ 发失败」的数据不一致。业务 DB 操作与 Outbox 记录在同一本地事务，定时扫描补发，保证生产者侧最终一致性。

### 性能

**Q：消息堆积了怎么办？**

排查消费者阻塞 → 临时扩容消费者实例（不超过分区数）→ 对不重要的历史消息跳过 offset → 新建临时 Topic 快速消费老堆积 → 根本解：优化消费逻辑降低单条处理时间。

> 注意：消费者实例数超过分区数后，多余实例空跑，是资源浪费。应先增加分区数，再横向扩消费者实例。

**Q：同步刷盘 vs 异步刷盘怎么选？**

异步刷盘（默认）性能高，Broker 宕机可能丢最近数据，适合日志、行为数据场景。同步刷盘每条消息落盘才返回 ACK，性能约降 30%，但可靠性高，适合金融、积分、订单场景。

### 架构

**Q：Kafka 消费组的隔离原理？**

每个消费组独立维护各分区的消费 offset，不同消费组消费同一 Topic 互不影响。新增消费组从最早或最新 offset 开始消费，不影响已有消费组的进度。这是订单消息多下游消费的核心机制。

**Q：为什么选 Kafka 而不是 RocketMQ 处理订单消息？**

订单消息的核心诉求是多下游独立消费 + 消息回溯 + 流处理生态。Kafka 的消费组隔离、offset 按时间重置、Flink/Spark 原生集成在这三点上均优于 RocketMQ。RocketMQ 在事务消息和延迟消息上有优势，更适合积分授予等金融一致性场景。
