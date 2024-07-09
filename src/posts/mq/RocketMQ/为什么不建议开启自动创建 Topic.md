---
date: 2024-07-01
category:
  - 消息队列
tag:
  - RocketMQ
excerpt: 为什么不建议开启自动创建 Topic，可能会出现哪些问题
order: 99
---

# 为什么不建议开启自动创建 Topic

在使用未创建的 Topic 发送消息时，由于 NameServer 中没有该 Topic 的路由信息，所以会拉取名为 TBW102 的 Topic 获取路由信息

- 如果开启了自动创建 Topic，Broker 在启动时创建名为 TBW102 的 Topic

```java
public static final String AUTO_CREATE_TOPIC_KEY_TOPIC = "TBW102"; // Will be created at broker when isAutoCreateTopicEnable
```

- 用户指定的读写队列数可能不是预期结果。创建的 Topic 的读队列数和写队列数取值为默认 Topic（TBW102）的读队列数和Produce端设置的队列数的最小值

```java
private volatile int defaultTopicQueueNums = 4;
```
