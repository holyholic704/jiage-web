---
date: 2024-07-29
category:
  - 微服务
tag:
  - Dubbo
  - 环境配置
excerpt: Spring Boot 集成 Dubbo
order: 99
---

# Spring Boot 集成 Dubbo

首先是喜闻乐见的版本选择环节，不过好在 Dubbo 声明了会兼容老版本，并且用户无需做任何改造，事实也确实如此，所以大胆的选择最新版即可

使用 Dubbo 需要创建三个模块

- 共享 API
- 服务端
- 消费端

> 当然，你也可以把服务端和消费端放一起，虽然失去了远程调用的意义，但作为测试、学习也无伤大雅

由于共享 API 充当了服务端与消费端的中间模块，所以我们在这个模块里引入依赖就可以了

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <artifactId>test-api</artifactId>
    <groupId>com.test</groupId>
    <version>0.0.1-SNAPSHOT</version>
    <name>test-api</name>
    <description>api</description>

    <dependencies>
        <dependency>
            <groupId>org.apache.dubbo</groupId>
            <artifactId>dubbo-spring-boot-starter</artifactId>
            <version>3.2.14</version>
        </dependency>
    </dependencies>
</project>
```

此外，Dubbo 依赖注册中心进行服务发现与治理，所以将 Nacos 的依赖一并加入

- 可以自行选择喜欢的或擅长的注册中心
- Dubbo 2.7 之前自带一个简单的注册中心，不过已被移除了

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.0.5.0</version>
</dependency>
```

接下来就在该模块定义一个简单的 API

```java
public interface TestService {
    String getStr();
}
```

创建服务端，并在 Maven 中引入共享 API 模块，消费端同理，下面就不多赘述了

```xml
<dependency>
    <groupId>com.test</groupId>
    <artifactId>test-api</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```

添加配置文件

```yaml
server:
  port: 9001
spring:
  application:
    name: producer-test
  cloud:
    nacos:
      discovery:
        # 依次添加节点地址，逗号分割
        # 不是集群部署的填一个足矣
        server-addr: 192.168.16.101:8848, 192.168.16.102:8848, 192.168.16.103:8848
        # 默认 pulic
        # 如需使用自定义的 namespace，需到控制台中添加，并在这里填写命名空间 ID，注意不是命名空间名称哦
        namespace: 68a0e6f6-210a-4d12-85ff-533d6daa9264
        # 默认 DEFAULT_GROUP
        group: dev_group
        username: woshishui
        password: woshinidie
dubbo:
  # Dubbo 的应用名
  application:
    name: producer-test-provider
  # Dubbo 协议信息
  protocol:
    name: dubbo
    # 默认端口 -1，建议指定一个固定的端口
    port: 10086
  # 注册中心地址
  registry:
    address: nacos://192.168.16.101:8848,192.168.16.102:8848,192.168.16.103:8848/?username=woshishui&password=woshinidie&group=dev_group&namespace=68a0e6f6-210a-4d12-85ff-533d6daa9264
```

实现 TestService 接口

```java
// 标记为 Dubbo 服务
@DubboService
public class TestServiceImpl implements TestService {

    @Override
    public String getStr() {
        return "你成功了";
    }
}
```

最后在启动类上加上两个注解，消费端同理，下面就不多赘述了

- `@EnableDiscoveryClient`：开启 Nacos 的服务注册与发现
  - 根据自己引入的注册中心进行修改
- `@EnableDubbo`：开启 Dubbo 服务

现在来到消费端，所需的配置与服务端几乎相同，配置文件需要稍作修改

```yaml
server:
  port: 9002
spring:
  application:
    name: consumer-test
  cloud:
    nacos:
      discovery:
        # 依次添加节点地址，逗号分割
        # 不是集群部署的填一个足矣
        server-addr: 192.168.16.101:8848, 192.168.16.102:8848, 192.168.16.103:8848
        # 默认 pulic
        # 如需使用自定义的 namespace，需到控制台中添加，并在这里填写命名空间 ID，注意不是命名空间名称哦
        namespace: 68a0e6f6-210a-4d12-85ff-533d6daa9264
        # 默认 DEFAULT_GROUP
        group: dev_group
        username: woshishui
        password: woshinidie
dubbo:
  # Dubbo 的应用名
  application:
    name: consumer-test-provider
  # Dubbo 协议信息
  protocol:
    name: dubbo
    # 默认端口 -1，建议指定一个固定的端口
    port: 10086
  # 注册中心地址
  registry:
    address: nacos://192.168.16.101:8848,192.168.16.102:8848,192.168.16.103:8848/?username=woshishui&password=woshinidie&group=dev_group&namespace=68a0e6f6-210a-4d12-85ff-533d6daa9264
```

接下来就只剩下如何远程调用了，在远程服务 API 上加上 `@DubboReference` 即可

```java
@Service
public class RpcTestService {

    @DubboReference
    TestService testService;

    public String test() {
        return testService.getStr();
    }
}
```

## 参考

- [Dubbo Spring Boot Starter 开发微服务应用](https://cn.dubbo.apache.org/zh-cn/overview/quickstart/java/spring-boot/)
