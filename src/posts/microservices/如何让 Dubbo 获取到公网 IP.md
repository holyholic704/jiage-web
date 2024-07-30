---
date: 2024-07-29
category:
  - 微服务
tag:
  - Dubbo
  - 环境配置
  - 问题解决
excerpt: Dubbo 服务注册时总是注册内网 IP，如何解决，仅供参考
order: 99
---

# 如何让 Dubbo 获取到公网 IP

## 起因

最近在学习调试 Dubbo，本地服务之间调用一切正常，但将服务端发到云服务器上，本地就死活连不上 Dubbo，甚至应用都无法启动

检查了下日志发现了 `failed to connect to server /172.0.4.13:10086`，这里用的是云服务器的内网 IP，难怪本地死活连不上

## 失败的尝试

于是我上网查资料，如何让 Dubbo 注册服务时注册自己的公网 IP，找到了一篇 [主机绑定](https://cn.dubbo.apache.org/zh-cn/docs/advanced/hostname-binding/)，还是官方的，看着挺有希望的，照着改了 hosts、Dubbo 配置，甚至添加 JVM 运行参数 `-DDUBBO_IP_TO_REGISTRY=公网IP`，还是以失败告终

### 修改网卡配置

既然无法修改服务的 IP，那我修改网卡的 IP 总该行了吧，把网卡的内网 IP 换成公网 IP，这样注册服务的时候不就会用公网 IP 了吗

于是又经过了一番捣鼓，这个看似很有希望的方案还是失败了，甚至都无法访问外网了，后来才发现我的云服务器不支持修改 IP 地址，具体的危害我已经享受到了

### 组内网？

改不了 IP，那我两台服务器组内网也可以啊，只要两台云服务器之间的服务能调用就行了，我本地连不上也无所谓

这次浪费的时间倒是不多，试了几个搜到的组内网的方法都不行。结局其实早就注定了，因为我发现服务器的控制台中有一个内网互联的功能，还是要收费的，这就难怪了。不过，我还试了下这个功能，怎么说呢，完全不知道怎么用，文档写的不清不楚的，况且还是要收费的，还是算了吧

此时，已经浪费了大半天的时间了，而且心里都有点崩溃了，要不，就到这吧，本地测测就完了，其他事情交给大牛处理就行了

## 还有希望吗

或许，我再多搜一搜呢？虽然我之前也搜到过不少与我问题类似的解决方案，诸如关键词 `dubbo 内网ip`、`dubbo 外网ip`，也在官方仓库以 `ip` 为关键词搜索过一番 Issues，但都解决不了我的问题。要不试试专业的，Github 的搜索功能确实够强大，为什么不试一下更强大的 Google 呢

于是我又用 Google 搜索 `site:github.com/apache/dubbo/issues ip`，确实出现了很多没看过的搜索结果，我就像抓住了救命稻草，把没看过的挨个查看了一番，不过，结果还是一样的，能解决问题，但解决不了我的问题

再试试查一下 Stackoverflow 吧，虽然我并不看好，因为国内的一些开源软件在国外使用的并不多，哪怕是大名鼎鼎的 Dubbo，更何况我遇到的还不是一个普遍的问题，不过事已至此，再多试一下也无妨，再次使用 Google 搜索 `site:stackoverflow.com dubbo ip`

## 绝处逢生

谁能料想到我最不看好的地方，却最终解决了我的问题呢

[Why dubbo provider always registers to a wrong address?](https://stackoverflow.com/questions/56153450/why-dubbo-provider-always-registers-to-a-wrong-address)，这位老哥就在抱怨为什么 Dubbo 总是注册错的 IP 地址，看了他的 IP 地址，想必也是遇到了 Dubbo 注册时注册了内网 IP。下面给出的方案也是简单粗暴，禁用环回地址，一般来说也就是 127.0.0.1

```shell
# 查看网卡配置
ifconfig

# lo 开头的就是环回地址
ifconfig lo shutdown

# 禁用之后，再查看网卡配置，lo 就没有了
```

- 注意重启网卡或者重启服务器都会失效，所以要么开机之后手动操作，要么就添加一个开机启动的脚本，具体方法附在最后

禁用了环回地址，再启动服务端，就能看到日志中打印的 `current host: 公网IP`，本地再测试一下，果然能调用到远程服务了

那么，代价是什么？之后就不能再使用 127.0.0.1 了，换成公网 IP 就行了，当然可能还会有些隐性的问题，不过，这也算解决了我的问题，虽然有些瑕疵，但已经足够满足我目前的需求。所以这是一个仅供参考的解决方案，实际开发中还是不要轻易尝试

## 参考

- [主机绑定](https://cn.dubbo.apache.org/zh-cn/docs/advanced/hostname-binding/)
- [主机地址自定义暴露](https://cn.dubbo.apache.org/zh-cn/overview/mannual/java-sdk/advanced-features-and-usage/others/set-host/)
- [Why dubbo provider always registers to a wrong address?](https://stackoverflow.com/questions/56153450/why-dubbo-provider-always-registers-to-a-wrong-address)
- [Linux系统如何设置开机自动运行脚本？](https://www.cnblogs.com/yychuyu/p/13095732.html)
