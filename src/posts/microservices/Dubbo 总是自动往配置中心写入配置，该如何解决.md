---
date: 2024-07-29
category:
  - 微服务
tag:
  - Dubbo
  - Nacos
  - 问题解决
excerpt: 配置中心有大量 Dubbo 自动上传的配置，该如何解决
order: 99
---

# Dubbo 总是自动往配置中心写入配置，该如何解决

起初只是觉得 Dubbo 硬往配置中心塞这么多配置看着心烦，搜索之后知道这些配置是 Dubbo 注册时上传的一些元数据，也是可以删的，自己手动删了几个也还是可以正常提供服务

Dubbo 会默认将注册中心作为元数据中心，也有人提到过，使用 Nacos 作为元数据中心对 Nacos 的性能有影响，虽然使用元数据中心的本就有减轻注册中心压力的目的，但如果注册中心和元数据中心是同一个，就需要多做权衡了

以下给出一个测试过的解决方法，还是那句话，仅供参考

```yaml
dubbo:
  registry:
    address: nacos://XXX.XXX.XXX.XXX:8848
    # 同时作为配置中心
    use-as-config-center: false
    # 同时作为元数据中心
    use-as-metadata-center: false
  consumer:
    # 关闭所有服务启动时检查（没有提供者时报错）
    check: false
```

## 参考

- [dubbo使用nacos的配置中心之后会疯狂往配置中心写入配置, 这个正常吗?](https://github.com/apache/dubbo/issues/9306)
- [配置禁用元数据中心后，comsumer启动报错，我该怎么配置？](https://github.com/apache/dubbo/issues/9293)
- [15-Dubbo的三大中心之元数据中心源码解析](https://cn.dubbo.apache.org/zh-cn/blog/2022/08/15/15-dubbo%e7%9a%84%e4%b8%89%e5%a4%a7%e4%b8%ad%e5%bf%83%e4%b9%8b%e5%85%83%e6%95%b0%e6%8d%ae%e4%b8%ad%e5%bf%83%e6%ba%90%e7%a0%81%e8%a7%a3%e6%9e%90/)
