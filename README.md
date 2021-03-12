# 《分布式系统模式》中文版

[《分布式系统模式》（Patterns of Distributed Systems）](https://martinfowler.com/articles/patterns-of-distributed-systems/)是 [Unmesh Joshi](https://twitter.com/unmeshjoshi) 编写的一系列关于分布式系统实现的文章。这个系列的文章采用模式介绍的格式，介绍了像 Kafka、Zookeeper 这种分布式系统在实现过程采用的通用模式，是学习分布式系统实现的基础。

## 目录

[概述](content/overview.md)

### 模式

* [一致性内核（Consistent Core）](content/consistent-core.md)
* [世代时钟（Generation Clock）](content/generation-clock.md)
* Gossip Dissemination
* [心跳（HeartBeat）](content/heartbeat.md)
* [高水位标记（High-Water Mark）](content/high-water-mark.md)
* [幂等接收者（Idempotent Receiver）](content/idempotent-receiver.md)
* 领导者和追随者（Leader and Followers）
* 租约（Lease）
* 低水位标记（Low-Water Mark）
* Quorum
* 请求管道（Request Pipeline）
* 分段日志（Segmented Log）
* 单一 Socket 通道（Single Socket Channel）
* 单一更新队列（Singular Update Queue）
* 状态监控（State Watch）
* Versioned Values
* 预写日志（Write-Ahead Log）

## 术语表

| 英文             | 翻译           |
| ---------------- | -------------- |
| durability       | 持久性         |
| Write-Ahead Log  | 预写日志       |
| append           | 追加           |
| hash             | 哈希           |
| replicate        | 复制           |
| failure          | 失效           |
| partition        | 分区           |
| HeartBeat        | 心跳           |
| Quorum           | Quorum         |
| Leader           | 领导者         |
| Follower         | 追随者         |
| High Water Mark  | 高水位标记     |
| Low Water Mark   | 低水位标记     |
| entry            | 条目           |
| propagate        | 传播           |
| disconnect       | 失联、断开连接 |
| Generation Clock | 世代时钟       |
| group membership | 分组成员       |
| partitions       | 分区          |
| liveness         | 活性          |
| round trip       | 往返          |
| in-flight        | 在途          |



