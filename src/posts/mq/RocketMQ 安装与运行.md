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

## 参考

- [Docker 部署 RocketMQ](https://rocketmq.apache.org/zh/docs/quickStart/02quickstartWithDocker)
- [腾讯云部署RocketMQ，connect to 10911 failed](https://blog.csdn.net/lifuma/article/details/91129573)
