---
date: 2024-07-01
category:
  - 消息队列
tag:
  - RocketMQ
excerpt: RocketMQ 在 Docker 或 Linux 环境下的安装与运行，包括与 Spring Boot 的整合
order: 99
---

# RocketMQ 安装与运行

## Windows 环境下安装

Windows 环境下，下载压缩包解压运行即可，不多做赘述

## 使用 Docker 安装

官网有详细的安装流程，但也有几个需要注意的地方

首先拉取镜像

```shell
docker pull apache/rocketmq:5.2.0
```

创建容器共享网络

```shell
docker network create rocketmq
```

启动 NameServer

```shell
docker run -d --name rmqnamesrv -p 9876:9876 --network rocketmq apache/rocketmq:5.2.0 sh mqnamesrv

# 验证 NameServer 是否启动成功
docker logs -f rmqnamesrv
```

从 NameServer 中复制一份 `borker.conf` 出来，或者按照下面给出配置，自己创建一个

```shell
# docker cp 容器ID:容器内文件位置 本地位置
docker cp 8d3b072f047f:/home/rocketmq/rocketmq-5.2.0/conf/broker.conf D:/conf/broker.conf
```

```shell
# 以下为默认配置
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
# 以上为默认配置

# Broker节点所在地址
brokerIP1 = 外网IP
# NameServer地址
namesrvAddr = 外网IP:9876
# 是否自动创建Topic，生产环境中不建议开启
autoCreateTopicEnable = true
```

- 如果运行时报 `connect to 10911 failed`，记得加上 brokerIP1 与 namesrvAddr ，并且使用外网地址

启动 Broker

```shell
docker run -d --name rmqbroker --network rocketmq -p 10912:10912 -p 10911:10911 -p 10909:10909 -e "NAMESRV_ADDR=rmqnamesrv:9876" -v D:/conf/broker.conf:/home/rocketmq/rocketmq-5.2.0/conf/broker.conf apache/rocketmq:5.2.0 sh mqbroker -c /home/rocketmq/rocketmq-5.2.0/conf/broker.conf
```

- 注意要使用 RocketMQ 的 `-c` 选项，表示使用自定义的配置，否则仍会加载默认的配置

## Linux 环境下安装

建议下载二进制包，解压后即可使用

启动 NameServer

```shell
$ nohup sh bin/mqnamesrv &
 
# 验证namesrv是否启动成功
$ tail -f ~/logs/rocketmqlogs/namesrv.log
The Name Server boot success...
```

启动 Broker

```shell
nohup sh bin/mqbroker -n localhost:9876 &

# 验证broker是否启动成功
tail -f ~/logs/rocketmqlogs/proxy.log 
```

记得放开 10912、10911、10909 这 3 个端口

## 与 Spring Boot 整合

引入依赖

```xml
<dependency>
    <groupId>org.apache.rocketmq</groupId>
    <artifactId>rocketmq-spring-boot-starter</artifactId>
    <version>2.3.0</version>
</dependency>
```

添加配置文件

```yaml
rocketmq:
  name-server: localhost:9876
  producer:
    group: my-group
```

添加配置类

```java
@Configuration
public class RocketConfig {

    @Value("${rocketmq.name-server}")
    private String nameServer;

    @Value("${rocketmq.producer.group}")
    private String producerGroup;

    @Bean
    RocketMQTemplate rocketmqTemplate() {
        RocketMQTemplate rocketMQTemplate = new RocketMQTemplate();
        DefaultMQProducer defaultMQProducer = new DefaultMQProducer();
        defaultMQProducer.setNamesrvAddr(nameServer);
        defaultMQProducer.setProducerGroup(producerGroup);
        rocketMQTemplate.setProducer(defaultMQProducer);
        return rocketMQTemplate;
    }
}
```

编写生产者

```java
@RestController
public class TestController {

    @Autowired
    private RocketMQTemplate rocketmqTemplate;

    @GetMapping("send")
    public String send(@RequestParam String topic, @RequestParam String msg) {
        Message<String> message = MessageBuilder.withPayload(msg).build();
        rocketmqTemplate.send(topic, message);
        return "SUCCESS";
    }
}
```

也可以选择直接使用 DefaultMQProducer

```java
@Component
public class TestProducer implements ApplicationRunner {

    @Value("${rocketmq.name-server}")
    private String nameServer;

    @Value("${rocketmq.producer.group}")
    private String producerGroup;

    private DefaultMQProducer defaultMQProducer = new DefaultMQProducer();

    public void init() throws MQClientException {
        defaultMQProducer.setProducerGroup(producerGroup);
        defaultMQProducer.setNamesrvAddr(nameServer);
        defaultMQProducer.start();
        System.out.println("启动");
    }

    public void send(String topic, String message) throws MQBrokerException, RemotingException, InterruptedException, MQClientException {
        Message msg = new Message(topic, message.getBytes());
        defaultMQProducer.send(msg);
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        init();
    }
}
```

编写消费者，实现 RocketMQListener 接口，并添加 RocketMQMessageListener 注解

```java
@Component
@RocketMQMessageListener(topic = "nice", consumerGroup = "ah-yeah")
public class TestConsumer implements RocketMQListener<String> {

    @Override
    public void onMessage(String message) {
        System.out.println(message);
    }
}
```

当然也可以选择直接使用 DefaultMQPushConsumer

```java
@Component
public class ManualConsumer implements ApplicationRunner {

    @Value("${rocketmq.name-server}")
    private String nameServer;

    private final DefaultMQPushConsumer consumer = new DefaultMQPushConsumer();

    public void init() throws MQClientException {
        consumer.setNamesrvAddr(nameServer);
        consumer.setConsumerGroup("ah-yeah");
        consumer.subscribe(MQConstant.NICE, "*");
        consumer.registerMessageListener((MessageListenerConcurrently) (msg, context) -> {
            for (MessageExt messageExt : msg) {
                System.out.println(new String(messageExt.getBody()));
            }
            return ConsumeConcurrentlyStatus.CONSUME_SUCCESS;
        });
        consumer.start();
        System.out.println("启动");
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        init();
    }
}
```

## 参考

- [Docker 部署 RocketMQ](https://rocketmq.apache.org/zh/docs/quickStart/02quickstartWithDocker)
- [腾讯云部署RocketMQ，connect to 10911 failed](https://blog.csdn.net/lifuma/article/details/91129573)
- [SpringBoot集成RocketMQ实现三种消息发送方式](https://blog.csdn.net/m0_37999219/article/details/131109507)
- [RocketMQ笔记（二）SpringBoot整合RocketMQ发送异步消息](https://blog.csdn.net/Alian_1223/article/details/136591837)
- [SpringBoot整合RocketMQ，老鸟们都是这么玩的！](https://www.cnblogs.com/jianzh5/p/17301690.html)
- [Springboot整合RocketMQ简单使用](https://www.cnblogs.com/qlqwjy/p/16175864.html)
