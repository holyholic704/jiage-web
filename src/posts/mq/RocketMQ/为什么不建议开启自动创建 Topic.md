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

一个 Topic 下有多个队列，可分布在不同的 Broker 上，在消息在发送前需要找到 Topic 与 Broker 对应关系，也就是路由信息

在使用未创建的 Topic 发送消息时，由于 NameServer 中没有该 Topic 的路由信息，也就不知道该往哪个 Broker 发送消息，这时生产者会拉取名为 TBW102 的 Topic 以获取路由信息

- 如果开启了自动创建 Topic，Broker 在启动时创建名为 TBW102 的 Topic

```java
// 在发送消息时，通过该方法获取路由信息
private TopicPublishInfo tryToFindTopicPublishInfo(final String topic) {
    // 首先从本地缓存的路由表中查找路由信息
    TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
    // 如果获取不到该 Topic 的路由信息
    if (null == topicPublishInfo || !topicPublishInfo.ok()) {
        this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
        // 主动向 NameServer 拉取该 Topic 的路由信息，很显然，也是没有的
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
    }

    if (topicPublishInfo.isHaveTopicRouterInfo() || topicPublishInfo.ok()) {
        return topicPublishInfo;
    } else {
        // 从 NameServer 上拉取 TBW102 的路由信息
        this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic, true, this.defaultMQProducer);
        topicPublishInfo = this.topicPublishInfoTable.get(topic);
        return topicPublishInfo;
    }
}
```

生产者通过 TBW102 上的路由信息，构建出一个新创建 Topic 的路由信息。目前该 Topic 的路由信息还是存储在本地的，也就是生产者，还未将路由信息注册到 NameServer 中，但现在已经能开始发送消息了，因为可以使用 TBW102 的路由信息，所以接下来就是 Broker 中获取到消息后的处理步骤

```java
public class SendMessageProcessor extends AbstractSendMessageProcessor implements NettyRequestProcessor {

    ...

    // Broker 中的消息处理方法
    public RemotingCommand processRequest(ChannelHandlerContext ctx,
        RemotingCommand request) throws RemotingCommandException {
        SendMessageContext sendMessageContext;
        switch (request.getCode()) {
            case RequestCode.CONSUMER_SEND_MSG_BACK:
                return this.consumerSendMsgBack(ctx, request);
            default:
                SendMessageRequestHeader requestHeader = parseRequestHeader(request);
                if (requestHeader == null) {
                    return null;
                }
                
                ...

                // 这里是判断本次消息是否是批量消息，两个方法内部都调用了 preSend 方法，知道这个就足够了
                if (requestHeader.isBatch()) {
                    response = this.sendBatchMessage(ctx, request, sendMessageContext, requestHeader, mappingContext,
                        (ctx1, response1) -> executeSendMessageHookAfter(response1, ctx1));
                } else {
                    response = this.sendMessage(ctx, request, sendMessageContext, requestHeader, mappingContext,
                        (ctx12, response12) -> executeSendMessageHookAfter(response12, ctx12));
                }

                return response;
        }
    }

    public RemotingCommand sendMessage(final ChannelHandlerContext ctx,
    final RemotingCommand request,
    final SendMessageContext sendMessageContext,
    final SendMessageRequestHeader requestHeader,
    final TopicQueueMappingContext mappingContext,
    final SendMessageCallback sendMessageCallback) throws RemotingCommandException {

      ...

      final RemotingCommand response = preSend(ctx, request, requestHeader);

      ...

    }

    private RemotingCommand preSend(ChannelHandlerContext ctx, RemotingCommand request,
        SendMessageRequestHeader requestHeader) {
        final RemotingCommand response = RemotingCommand.createResponseCommand(SendMessageResponseHeader.class);

        ...

        // 调用父类的 msgCheck 方法
        super.msgCheck(ctx, requestHeader, request, response);

        return response;
    }

    ...

}
```

```java
public abstract class AbstractSendMessageProcessor implements NettyRequestProcessor {

    ...

    protected RemotingCommand msgCheck(final ChannelHandlerContext ctx,
        final SendMessageRequestHeader requestHeader, final RemotingCommand request,
        final RemotingCommand response) {

        // 以下是对该 Topic 的一些校验，不是重点，不需要过多关注
        if (!PermName.isWriteable(this.brokerController.getBrokerConfig().getBrokerPermission())
            && this.brokerController.getTopicConfigManager().isOrderTopic(requestHeader.getTopic())) {
            response.setCode(ResponseCode.NO_PERMISSION);
            response.setRemark("the broker[" + this.brokerController.getBrokerConfig().getBrokerIP1()
                + "] sending message is forbidden");
            return response;
        }

        TopicValidator.ValidateTopicResult result = TopicValidator.validateTopic(requestHeader.getTopic());
        if (!result.isValid()) {
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark(result.getRemark());
            return response;
        }
        if (TopicValidator.isNotAllowedSendTopic(requestHeader.getTopic())) {
            response.setCode(ResponseCode.NO_PERMISSION);
            response.setRemark("Sending message to topic[" + requestHeader.getTopic() + "] is forbidden.");
            return response;
        }

        // 从本地获取该 Topic 的相关配置，但目前是没有的
        TopicConfig topicConfig =
            this.brokerController.getTopicConfigManager().selectTopicConfig(requestHeader.getTopic());
        if (null == topicConfig) {
            int topicSysFlag = 0;
            if (requestHeader.isUnitMode()) {
                if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    topicSysFlag = TopicSysFlag.buildSysFlag(false, true);
                } else {
                    topicSysFlag = TopicSysFlag.buildSysFlag(true, false);
                }
            }
            
            LOGGER.warn("the topic {} not exist, producer: {}", requestHeader.getTopic(), ctx.channel().remoteAddress());
            // 根据发过来的 Topic，创建相关的配置信息
            topicConfig = this.brokerController.getTopicConfigManager().createTopicInSendMessageMethod(
                requestHeader.getTopic(),
                requestHeader.getDefaultTopic(),
                RemotingHelper.parseChannelRemoteAddr(ctx.channel()),
                requestHeader.getDefaultTopicQueueNums(), topicSysFlag);

            if (null == topicConfig) {
                if (requestHeader.getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    topicConfig =
                        this.brokerController.getTopicConfigManager().createTopicInSendMessageBackMethod(
                            requestHeader.getTopic(), 1, PermName.PERM_WRITE | PermName.PERM_READ,
                            topicSysFlag);
                }
            }

            if (null == topicConfig) {
                response.setCode(ResponseCode.TOPIC_NOT_EXIST);
                response.setRemark("topic[" + requestHeader.getTopic() + "] not exist, apply first please!"
                    + FAQUrl.suggestTodo(FAQUrl.APPLY_TOPIC_URL));
                return response;
            }
        }

        int queueIdInt = requestHeader.getQueueId();
        int idValid = Math.max(topicConfig.getWriteQueueNums(), topicConfig.getReadQueueNums());
        if (queueIdInt >= idValid) {
            String errorInfo = String.format("request queueId[%d] is illegal, %s Producer: %s",
                queueIdInt,
                topicConfig,
                RemotingHelper.parseChannelRemoteAddr(ctx.channel()));

            LOGGER.warn(errorInfo);
            response.setCode(ResponseCode.SYSTEM_ERROR);
            response.setRemark(errorInfo);

            return response;
        }
        return response;
    }

    ...

}
```

```java
public class TopicConfigManager extends ConfigManager {

    ...

    public TopicConfig createTopicInSendMessageMethod(final String topic, final String defaultTopic,
        final String remoteAddress, final int clientDefaultTopicQueueNums, final int topicSysFlag) {
        TopicConfig topicConfig = null;
        // 是否创建新 Topic 成功
        boolean createNew = false;

        try {
            if (this.topicConfigTableLock.tryLock(LOCK_TIMEOUT_MILLIS, TimeUnit.MILLISECONDS)) {
                try {
                    topicConfig = getTopicConfig(topic);
                    if (topicConfig != null) {
                        return topicConfig;
                    }

                    // 获取默认 Topic 的配置信息，也就是 TBW102
                    TopicConfig defaultTopicConfig = getTopicConfig(defaultTopic);
                    if (defaultTopicConfig != null) {
                        if (defaultTopic.equals(TopicValidator.AUTO_CREATE_TOPIC_KEY_TOPIC)) {
                            if (!this.brokerController.getBrokerConfig().isAutoCreateTopicEnable()) {
                                defaultTopicConfig.setPerm(PermName.PERM_READ | PermName.PERM_WRITE);
                            }
                        }

                        // 创建新的 Topic 配置信息
                        if (PermName.isInherited(defaultTopicConfig.getPerm())) {
                            topicConfig = new TopicConfig(topic);

                            // 队列数量取默认队列数量 4，与默认 Topic 的写队列数量之间的最小值，即 4
                            int queueNums = Math.min(clientDefaultTopicQueueNums, defaultTopicConfig.getWriteQueueNums());

                            if (queueNums < 0) {
                                queueNums = 0;
                            }

                            topicConfig.setReadQueueNums(queueNums);
                            topicConfig.setWriteQueueNums(queueNums);
                            int perm = defaultTopicConfig.getPerm();
                            perm &= ~PermName.PERM_INHERIT;
                            topicConfig.setPerm(perm);
                            topicConfig.setTopicSysFlag(topicSysFlag);
                            topicConfig.setTopicFilterType(defaultTopicConfig.getTopicFilterType());
                        } else {
                            log.warn("Create new topic failed, because the default topic[{}] has no perm [{}] producer:[{}]",
                                defaultTopic, defaultTopicConfig.getPerm(), remoteAddress);
                        }
                    } else {
                        log.warn("Create new topic failed, because the default topic[{}] not exist. producer:[{}]",
                            defaultTopic, remoteAddress);
                    }

                    if (topicConfig != null) {
                        log.info("Create new topic by default topic:[{}] config:[{}] producer:[{}]",
                            defaultTopic, topicConfig, remoteAddress);

                        // 添加到本地的 Topic 配置表
                        putTopicConfig(topicConfig);

                        long stateMachineVersion = brokerController.getMessageStore() != null ? brokerController.getMessageStore().getStateMachineVersion() : 0;
                        dataVersion.nextVersion(stateMachineVersion);

                        createNew = true;

                        this.persist();
                    }
                } finally {
                    this.topicConfigTableLock.unlock();
                }
            }
        } catch (InterruptedException e) {
            log.error("createTopicInSendMessageMethod exception", e);
        }

        // 向 NameServer 发送新建 Topic 的注册信息
        if (createNew) {
            registerBrokerData(topicConfig);
        }

        return topicConfig;
    }

    ...

}
```

跟着这个流程走一遍，想必你已经能看出来一些问题了

## 读写队列数量

自动创建的主题的读写队列数量分别为 4，而手动创建的默认的读写队列数量分别为 16，虽然也可以后期修改，但不建议，可能会导致消息丢失、重复消费等异常情况

## 可能只有一个 Broker 上有该 Topic 的路由信息

自动创建 Topic 时，由于没有路由信息，会从 TBW102 的路由信息中找到一个 Broker，并发送消息。该 Broker 收到消息后，发现本地缓存和 NameSevrer 上都没有该 Topic 的信息，就会在本地创建一个该 Topic 的配置信息，创建完成后再向 NameServer 发送一个注册信息。除此之外，生产者默认每隔 30 秒向 NameServer 拉取一次路由信息，如果在下一次拉取路由信息之前，没有生产者再往该 Topic 发送消息，那么该 Topic 的路由信息就只会包含创建他的那个 Broker，之后往该 Topic 发送消息时，就只会向这一个 Broker 发送

这样就无法保证高可用了，如果这个 Broker 宕机了，就无法向这个 Topic 发送消息了，什么负载均衡、高吞吐量也更别谈了

如果在自动创建过程中，在所有生产者向 NameServer 拉取一次路由信息之前，不断地有消息发送到待创建的 Topic 上，还是有可能在所有的 Broker 上都创建出该 Topic 的路由信息，但谁又能保证会有足够多的消息，消息会发到所有的 Broker 上呢

## 参考

- [RocketMQ生产环境开启自动创建topic会有什么影响?](https://blog.csdn.net/meser88/article/details/122344206)
- [RocketMQ实战：生产环境中，autoCreateTopicEnable为什么不能设置为true](https://blog.csdn.net/2401_84103441/article/details/138739993)
- [记一次RokcetMQ Topic自动创建问题](https://juejin.cn/post/7298966889359212596)
