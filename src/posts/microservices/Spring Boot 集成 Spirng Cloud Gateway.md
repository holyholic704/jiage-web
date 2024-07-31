---
date: 2024-07-30
category:
  - 微服务
tag:
  - Nacos
  - 环境配置
excerpt: Spring Boot 集成 Spirng Cloud Gateway
order: 99
---

# Spring Boot 集成 Spirng Cloud Gateway

Spirng Cloud Gateway 没有明确的版本对应关系，反正我是不知道，所以如果你遇到了如下错误，大概率是你的版本不对

```shell
java.lang.NoSuchMethodError: reactor.netty.http.client.HttpClient.noChunkedTransfer()Lreactor/netty/http/client/HttpClient;
```

所以你最好从一个能正常运行的项目中找，或者使用下面提供的版本

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
    <version>3.1.9</version>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    <version>3.1.8</version>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.0.5.0</version>
</dependency>
```

- 注意不要引入 web 依赖

引入完依赖，就可以开始编写配置文件

```yaml
server:
  port: 9000
spring:
  application:
    name: gateway-test
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
   gateway:
     routes:
         # id 唯一就行，无特殊要求
       - id: suiyitianxie
         # 匹配后提供服务的路由地址
         # lb 代表负载均衡，也可以直接使用 http
         # 地址可以写服务名，也可用 IP 加端口号
         uri: lb://mougefuwu
         # 断言，路径匹配的进行路由
         predicates:
           - Path=/test
```

最后别忘了在启动类加上注解 `@EnableDiscoveryClient`

现在 gateway 已经可以使用了，假如你的某个服务名为 mougefuwu，也就是配置文件里填写的服务名，端口号为 9001，该服务内有个 test 接口

之前你是通过 `localhost:9001/test` 请求 test 接口，想要使用 gateway 进行调用就要改为 `localhost:9000/test`，这样你就能通过 gateway 进行路由转发了

- 如果调用出现 503 错误，大概率是你没引入 loadbalancer 的依赖

## 动态路由

如果一个一个配路由，那不得累死了，所以 gateway 也提供了动态路由的功能

```yaml
spring:
  cloud:
    gateway:
      discovery:
        locator:
          # 开启从注册中心动态创建路由的功能，使用服务名进行路由
          enabled: true
```

再拿上面的例子来说，现在要想通过 gateway 进行路由转发，就要改为使用 `localhost:9000/mougefuwu/test` 进行请求，也就是再加上服务名

## 参考

- [spring cloud gateway fails to route due to netty issue](https://stackoverflow.com/questions/53261357/spring-cloud-gateway-fails-to-route-due-to-netty-issue)
- [SpringCloud gateway 路由转发 503 错误踩坑](https://www.cnblogs.com/cgsdg/p/16851016.html)
- [SpringCloud集成Gateway新一代服务网关](https://www.cnblogs.com/xfeiyun/p/16222605.html)
