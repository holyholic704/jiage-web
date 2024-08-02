---
date: 2024-08-01
category:
  - 消息队列
tag:
  - RocketMQ
  - 环境配置
excerpt: Linux 下 RocketMQ 集群部署
order: 99
---

# RocketMQ 集群部署

由于 NameServer 是去中心化的，所以直接启动就行 `nohup sh bin/mqnamesrv &`

- 如果启动失败，大概率是默认设置的堆内存过大

```shell
# 修改 runserver.sh，找到下面这句并修改为适合的参数，runbroker.sh 同理
JAVA_OPT="${JAVA_OPT} -server -Xms4g -Xmx4g -Xmn2g -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
```

然后就是重头戏，Broker 集群的配置，进入 `conf` 目录，你能看到 3 个文件夹

- `2m-2s-async`：两主两从，一个主节点对应一个从节点，异步复制
- `2m-2s-sync`：两主两从，一个主节点对应一个从节点，同步复制
- `2m-noslave`：两主，没有从节点

你也可以自定义集群的节点配置，但第一次使用集群时，建议先从两主两从配置开始，我这里选择的是 `2m-2s-async`

进入文件夹，你会看到 4 个已经给好的配置文件

```shell
# 主节点 a
broker-a.properties
# 主节点 a 的从节点
broker-a-s.properties
# 主节点 b
broker-b.properties
# 主节点 b 的从节点
broker-b-s.properties
```

- 注意名称只是方便区分，对集群部署并没有什么影响

假设部署的两台机器

- 机器 A：公网 IP 为 192.168.16.101
- 机器 B：公网 IP 为 192.168.16.102

建议主从节点不要部署在同一台机器上，所以将做如下部署

- 机器 A：主节点 a、主节点 b 的从节点
- 机器 B：主节点 b、主节点 a 的从节点

先修改主节点 `broker-a.properties`

```properties
# NameServer 集群地址，使用分号分割
namesrvAddr=192.168.16.101:9876;192.168.16.102:9876
# 所属集群
brokerClusterName=DefaultCluster
# Broker 名称，主从节点相同
brokerName=broker-a
# 0 表示主节点，非 0 表示从节点
brokerId=0
# 删除文件的时间节点，默认为 4 点
deleteWhen=04
# 文件保留时间，默认为 48 小时
fileReservedTime=48
# 节点角色
brokerRole=ASYNC_MASTER
# 刷盘策略
flushDiskType=ASYNC_FLUSH
# 监听端口，同一台机器上的端口都要不同，且相互间距离要大
listenPort=10911
# 开放内外网访问
brokerIP1=192.168.16.101
brokerIP2=机器A的内网IP
# 存储路径，每个节点分配单独的空间
storePathRootDir=/usr/local/rocketmq/data/store
```

修改从节点 `broker-a-s.properties`

```properties
namesrvAddr=192.168.16.101:9876;192.168.16.102:9876
brokerClusterName=DefaultCluster
brokerName=broker-a
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
listenPort=10921
autoCreateSubscriptionGroup=true
brokerIP1=192.168.16.102
brokerIP2=机器B的内网IP
storePathRootDir=/usr/local/rocketmq/rocketmq/data/store/slave
```

`broker-b.properties` 同理

```properties
namesrvAddr=192.168.16.101:9876;192.168.16.102:9876
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
listenPort=10911
brokerIP1=192.168.16.102
brokerIP2=机器B的内网IP
# 存储路径，每个节点分配单独的空间
storePathRootDir=/usr/local/rocketmq/data/store
```

`broker-b-s.properties` 同理

```properties
namesrvAddr=192.168.16.101:9876;192.168.16.102:9876
brokerClusterName=DefaultCluster
brokerName=broker-b
brokerId=1
deleteWhen=04
fileReservedTime=48
brokerRole=SLAVE
flushDiskType=ASYNC_FLUSH
listenPort=10921
autoCreateSubscriptionGroup=true
brokerIP1=192.168.16.101
brokerIP2=机器A的内网IP
storePathRootDir=/usr/local/rocketmq/rocketmq/data/store/slave
```

配置好文件，还要记得开放端口，除了 NameServer 的 9876 端口，每个节点都要开放 3 个端口

- 10911：Broker 对外服务的监听端口
- 10909：偏移 -2，Broker 对外服务的监听端口
- 10912：偏移 +1，用于 Broker 主从同步

一台机器上两个节点，所以总共要开放 6 个节点，下方列举使用上方配置一台机器所需开放的所有端口

- 9876
- 10911、10909、10912
- 10921、10919、10922

准备工作做完就可以启动了，并且不需要再额外复制一份 RocketMQ，启动时加上 `-c`，再跟上配置文件，就可以根据不同的配置文件启动不同的实例了

先启动机器 A 的节点

```shell
# 启动主节点 a
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-a.properties &

# 启动主节点 b 的从节点
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-b-s.properties &
```

再启动机器 B 的节点

```shell
# 启动主节点 b
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-b.properties &

# 启动主节点 a 的从节点
nohup sh bin/mqbroker -c conf/2m-2s-async/broker-a-s.properties &
```

启动没有出问题的话，就可通过 `sh bin/mqadmin clusterlist -n XXX.XXX.XXX.XXX:9876` 查看集群信息了，如果没有列出集群列表，就说明你的配置有问题

## 参考

- [运维管理](https://github.com/apache/rocketmq/blob/develop/docs/cn/operation.md)
- [Apache RocketMQ 集群搭建（两主两从）](https://cloud.tencent.com/developer/article/1412653)
- [RocketMQ 集群部署与使用](https://xiangflight.github.io/rocketmq-cluster-deployment/)
- [RocketMQ 4.7.1 环境搭建、集群、SpringBoot整合MQ](https://www.cnblogs.com/chenyanbin/p/13798952.html)
- [0129---rocketMQ设置为内外网皆可访问](https://blog.csdn.net/gmriwyf/article/details/122746146)
