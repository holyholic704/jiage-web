---
date: 2024-07-08
category:
  - 消息队列
tag:
  - RocketMQ
excerpt: RocketMQ 在 Docker 或 Linux 环境下的安装与运行，包括与 Spring Boot 的整合
order: 99
---

# Rebalance 机制

Rebalance（再均衡）机制指的是将一个 Topic 下的多个队列，在同一个消费者组下的多个消费者实例之间进行重新分配，以提升消息的并行处理能力

- 例如某个 Topic 下有 4 个队列，某个消费者组内有 2 个订阅了该 Topic 消费者，这时组内再加一个订阅 Topic 的消费者，为了能让这个消费者能消费到消息，就会触发 Rebalance 机制，重新分配每个消费者消费的队列

消息消费并不是以消费者为单位，而是以消费者组为单位，消费者组会将 Topic 下的某一个队列分配给某一个消费者，也就是说，一个队列对应一个消费者，一个消费可以对应多个队列。可以说一个消费者组内的消费者越多，并行处理能力就更强，但上限是不能超过队列的数量，多余的消费者是分配不到队列的

RocketMQ 触发 Rebalance 机制的前提是使用的集群（CLUSTERING）消费模式，同一条消息，只允许被组内某一个消费者消费。如果使用的是广播（BROADCASTING），同一条消息，能被组内所有消费者消费，也没必要使用 Rebalance 机制

## 触发方式

- Topic 的队列数量变化
  - 队列的扩容或缩容
  - Topic 队列可以分布在不同的 Broker 上，某个 Broker 不可用时，可用的队列数量也会发生变化
- 消费者组信息变化
  - 消费者的扩容或缩容
  - 某些消费者不可用
  - 订阅的 Topic 发生变化

总之要么是队列数量变了，要么是消费者数量变了

## 危害

### 消费暂停

在 Rebalance 过程中，原消费者可能需要暂停部分队列的消费，等待这些队列分配给新的消费者

### 重复消费

消费者在消费完消息后，会提交消费到的 Offset 到 Broker 中，默认情况下，Offset 是异步提交的，这就可能导致在 Rebalance 过程中发生重复消费

例如消费者 1 当前消费某个队列到 Offset 为 10，但异步提交给 Broker 的 Offset 可能仍为 8，这时触发 Rebalance，消费者 2 被分到了这个队列，就有可能从 Offset 为 8 的位置开始消费，这就出现了重复消费

### 消费突刺

由于 Rebalance 可能导致消费暂停或重复消费，因此在 Rebalance 结束后，可能会出现瞬间需要消费大量消息的情况，即消费突刺。这可能会给系统带来额外的压力，影响系统的稳定性和性能

## 参考

- [4.0 消费者Rebalance机制](http://www.tianshouzhi.com/api/tutorials/rocketmq/409)
- [消费模式及rebalance机制](https://blog.csdn.net/hguhbh/article/details/137172799)
- [RocketMQ-Rebalance限制与危害](https://blog.csdn.net/CSDN877425287/article/details/122052134)
