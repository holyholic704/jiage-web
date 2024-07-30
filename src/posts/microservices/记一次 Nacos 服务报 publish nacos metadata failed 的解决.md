---
date: 2024-07-29
category:
  - 微服务
tag:
  - Dubbo
  - Nacos
  - 问题解决
excerpt: 不具有普遍意义，仅供参考
order: 99
---

# 记一次 Nacos 服务报 publish nacos metadata failed 的解决

最近某次启动某个服务时，发现 CPU 飙的很高，我当是服务启动时的正常现象，可盯着 CPU 好久就没降下来的趋势，于是就查看服务的启动日志，发现一直在报错，都是 `publish nacos metadata failed`

## 是否需要处理

服务启动了大概几分钟后，错误日志就不再打印了，CPU 也将下来了，也能正常提供服务。是否就放任不管呢，还有一大堆事需要处理呢，况且过一会 CPU 占用也会降下来，只要能正常提供服务就行

但是这只是一个服务启动时就占用了 50% 的 CPU，之后服务多了呢，怎么保证运行时不会再出现同样的问题呢，而且这么多错误日志，看着不扎心吗

所以这是一个需要处理的问题，起码先尝试一下

## 查找解决方案

首先要知道这是个什么问题，从字面意思看是向 Nacos 推送元数据时出现问题，什么是元数据，总结来说就是服务注册时的一些额外配置，版本号、实例的权重等

为什么会出现这个问题呢，上网查呗，第一条就是 [java.lang.RuntimeException: publish nacos metadata failed](https://github.com/apache/dubbo/issues/11726)，看来有可能是数据源的问题，因为 Nacos 集群部署时需要用到数据库，不过我用的是 MySQL，不是 PostgreSQL，再往下看解决方案也是针对 PostgreSQL 的

不过提问者也给出了一个 MySQL 适用的解决方案

```yaml
dubbo:
  registry:
    address: nacos://XXX.XXX.XXX.XXX:8848
    use-as-metadata-center: false
    use-as-config-center: false
```

Dubbo 服务注册时不再使用该地址作为配置中心和元数据中心，看来这个问题有可能是由 Dubbo 注册服务时引起的，但是不能因噎废食，作为备选方案吧，先去看看还有没有其他的解决方案

### 数据源问题？

之后又看了几个相关的帖子，大部分说是 Nacos 的版本问题，不过其中的一个回复提醒到了我，对啊，还有 Nacos 日志呢，我怎么现在才想到

查看了 `nacos.log`，我更懵了，`No DataSource set`？我之前都用的好好的啊，怎么会没有数据源呢，那早就不能用了啊，不过这也让我更确定了是数据源的问题

再翻看其他帖子，有人说 MySQL 8.0 版本需要引入 jar 包，不会吧，那我之前连接的是假数据库吗。照着试了一下，下载了 jar 包，放入 `${nacos.home}/plugins/mysql/` 目录，重启 Nacos，还是一样的报错。不过我在查看 `start.out` 日志还是发现了“意外之喜”，出现了一个特别的报错，不知道是不是引入 jar 包才出现的，但终于有一个明确点的原因了

`Host XXX.XXX.XXX.XXX is blocked because of many connection errors; unblock with mysqladmin flush-hosts`

就是有某个 IP 在短时间内向 MySQL 发出了大量的错误连接，从而被加入到黑名单中，暂时禁止该 IP 的访问。知道了原因，我还是不太理解为什么会发生，怎么会产生这么多错误连接，服务端连数据库的依赖都没引入，所以可以确定是 Nacos 操作数据库时产生的错误连接。大概率是我之前测试时，删除了数据库中某些数据导致的，当时可能就已经产生了错误连接，然后滚雪球，直到被 MySQL ban 了

## 解决

剩下的就好解决了，上网搜索一下对应的问题，基本上就两种解决方案

- 增大错误连接数：`SET GLOBAL max_connect_errors = 100000000;`
  - 默认值 100
- 刷新黑名单缓存：`FLUSH HOSTS;`

增加错误连接数被 ban 的阈值，这个不可取，这个本质上还是为了保障数据库安全，防止大量恶意请求，比如暴力破解等，更何况治标不治本，并没有减少错误连接，今日割五城，明日割十城，无论加到多少，只要源头不解决，总归有使用完的一天，甚至 MySQL 服务早崩了

我选择的就是剩下的刷新缓存，源头已经解决了，我把一些旧数据删除了，刷新缓存后再测试一下，果然没有报错了，所以也基本上可以确定是我之前随意操作数据库导致的

## 总结

- 善用搜索，如何通过关键字找到更贴近自己需要的结果
- 多多查看日志，建立一个监控平台是很有必要的，不能光靠不经意间的关注才能发现问题
- 不要随意操作数据库，即便是在测试、学习中，可能会出现意料之外的问题
- 原来 Nacos 不止支持 MySQL，还支持其他的数据库，如 PostgreSQL、Oracle 等，需要下载对应的插件
  - 之前阅读的一些资料中说 Nacos 支持内置的数据库和 MySQL，更新了我的认知

## 参考

- [java.lang.RuntimeException: publish nacos metadata failed](https://github.com/apache/dubbo/issues/11726)
- [Why MySQL connection is blocked of many connection errors?](https://stackoverflow.com/questions/20014746/why-mysql-connection-is-blocked-of-many-connection-errors)
- [nacos-group / nacos-plugin](https://github.com/nacos-group/nacos-plugin/tree/develop/nacos-datasource-plugin-ext)
