# 消息中间件

> 本目录收录消息中间件相关的技术选型、方案设计与面试参考资料，基于真实项目场景整理。

## 文档列表

| 文档 | 说明 |
|------|------|
| [关于 Kafka 和 RocketMQ 的选型](./关于Kafka和RocketMQ的选型.md) | 基于积分消息 + 订单消息场景的完整选型报告，涵盖 JDK 兼容性、功能对比、升级方案、可靠性设计、安全配置与监控告警 |
| [RocketMQ 高频面试题](./RocketMQ高频面试题.md) | 消息可靠性、性能调优、架构原理、死信处理等高频面试题及答案，含结合项目的加分回答模板 |

## 核心结论速览

**选型决策：Kafka 3.9**（`confluentinc/cp-kafka:7.9.0`）

- 两个服务（JDK 8 / JDK 17）均使用 `kafka-clients 3.9.x`，零改造成本
- 订单消息多下游订阅（仓储、物流、BI），Kafka 消费组隔离机制天然适配
- KRaft 模式无 ZooKeeper 依赖，运维更简洁
- 未来接入 Flink / Spark 流处理无缝集成

**积分消息可靠性保障**

```
业务操作 + 写 PointEventOutbox   ← 同一本地事务
        ↓
定时扫描 PENDING 记录
        ↓
发送到 Kafka → 成功标记 SENT / 失败指数退避重试
        ↓
消费者幂等消费（Redis SETNX，TTL 7 天）
```

**积分过期实现方案**：DB 定时扫描（`FOR UPDATE SKIP LOCKED`）→ 批量发 `points_expired` 事件 → 消费者执行扣减，调度与执行完全解耦。

---

> 相关项目代码见 `twoblack` 仓库其他模块。
