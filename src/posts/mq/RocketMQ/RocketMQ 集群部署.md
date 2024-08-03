---
date: 2024-08-01
category:
  - 消息队列
tag:
  - RocketMQ
  - 环境配置
excerpt: Linux 下 RocketMQ 集群部署，自动主从切换的配置
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

## 自动主从切换

照着官方文档来就行了 [自动主从切换快速开始](https://github.com/apache/rocketmq/blob/develop/docs/cn/controller/quick_start.md)，需要部署一个自动主从切换的 Controller（最好还是集群），可以独立部署也可以内嵌在 NameServer 中，我选择了内嵌的方式

在 `conf/controller` 中有相关的配置

- `cluster-3n-independent` 文件夹：独立部署
- `cluster-3n-namesrv-plugin` 文件夹：内嵌部署
- `controller-standalone.conf`：单节点部署

所以我们现在只需要关心 `cluster-3n-namesrv-plugin` 文件夹中的配置，进入文件夹，里面有 3 个已给好的配置文件

```shell
namesrv-n0.conf
namesrv-n1.conf
namesrv-n2.conf
```

现在假设有三台机器

- 机器 A：公网 IP 为 192.168.16.101
- 机器 B：公网 IP 为 192.168.16.102
- 机器 C：公网 IP 为 192.168.16.103

我们将 Controller 在三台机器上分别部署一个节点，构成一个集群，当然你想部署在同一台机器，或者部署在更多的机器上都行

先修改机器 A 上的 `namesrv-n0.conf`

```properties
#Namesrv config
# NameServer 的端口
listenPort = 9876
# 开启 Controller
enableControllerInNamesrv = true

#controller config
# 同一 Controller 集群相同
controllerDLegerGroup = group1
# 同一 Controller 集群相同，各节点的地址
controllerDLegerPeers = n0-192.168.16.101:9878;n1-192.168.16.102:9868;n2-192.168.16.103:9858
# 节点 ID，同一 Controller 唯一
controllerDLegerSelfId = n0
```

修改机器 B 上的 `namesrv-n1.conf`

```properties
#Namesrv config
# 如果是部署在不同的机器上，这里填 9876 也是可以的，官方给出的配置是部署在同一台机器的情况，所以端口要不同
listenPort = 9886
enableControllerInNamesrv = true

#controller config
controllerDLegerGroup = group1
controllerDLegerPeers = n0-192.168.16.101:9878;n1-192.168.16.102:9868;n2-192.168.16.103:9858
controllerDLegerSelfId = n1
```

修改机器 C 上的 `namesrv-n2.conf`

```properties
#Namesrv config
listenPort = 9896
enableControllerInNamesrv = true

#controller config
controllerDLegerGroup = group1
controllerDLegerPeers = n0-192.168.16.101:9878;n1-192.168.16.102:9868;n2-192.168.16.103:9858
controllerDLegerSelfId = n2
```

修改好配置，挨个启动就行了，记得开放 9878、9868、9858 端口

```shell
# 机器 A 运行
nohup bin/mqcontroller -c conf/controller/cluster-3n-namesrv-plugin/namesrv-n0.conf &

# 机器 B 运行
nohup bin/mqcontroller -c conf/controller/cluster-3n-namesrv-plugin/namesrv-n1.conf &

# 机器 C 运行
nohup bin/mqcontroller -c conf/controller/cluster-3n-namesrv-plugin/namesrv-n2.conf &
```

可通过下面的命令查看集群信息，如果没有节点列表就说明你配置有问题

```shell
# -a 跟上 Controller 集群任一节点的 IP 和端口
sh bin/mqadmin getControllerMetaData -a 192.168.16.101:9878
```

配置完这些就足够了吗，并不是，我之前意味配置完这些已经足够了，但总是无法自动切换，后来才发现还要修改 Broker 的配置，在这个文档 [部署和升级指南](https://github.com/apache/rocketmq/blob/develop/docs/cn/controller/deploy.md) 中有提到需要修改的配置，在上面那个快速开始的文档竟然提都没提，不愧是阿里开源啊

Broker 的配置也很简单，只需要添加两个配置

```properties
# 开启 Controller 模式
enableControllerMode=true

# Controller 所有节点的地址
controllerAddr=192.168.16.101:9878;192.168.16.102:9868;192.168.16.103:9858
```

之后重启，就可以测试主从切换了，关掉主节点，过 10 秒就可以看到某个节点晋升为主节点了，也可通过 `controllerHeartBeatTimeoutMills` 配置，在这期间 RocketMQ 集群是无法提供服务的，所以实际使用时记得添加重试机制

## 参考

- [运维管理](https://github.com/apache/rocketmq/blob/develop/docs/cn/operation.md)
- [Apache RocketMQ 集群搭建（两主两从）](https://cloud.tencent.com/developer/article/1412653)
- [RocketMQ 集群部署与使用](https://xiangflight.github.io/rocketmq-cluster-deployment/)
- [RocketMQ 4.7.1 环境搭建、集群、SpringBoot整合MQ](https://www.cnblogs.com/chenyanbin/p/13798952.html)
- [0129---rocketMQ设置为内外网皆可访问](https://blog.csdn.net/gmriwyf/article/details/122746146)
- [自动主从切换快速开始](https://github.com/apache/rocketmq/blob/develop/docs/cn/controller/quick_start.md)
- [部署和升级指南](https://github.com/apache/rocketmq/blob/develop/docs/cn/controller/deploy.md)
