---
date: 2024-07-29
category:
  - 微服务
tag:
  - Nacos
  - 环境配置
excerpt: Spring Boot 集成 Nacos 配置中心
order: 99
---

# Spring Boot 集成 Nacos 配置中心

坑较多，毕竟阿里开源嘛 `^_^`

首先引入依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2021.0.5.0</version>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-bootstrap</artifactId>
    <version>3.1.5</version>
</dependency>
```

如不引入 `spring-cloud-starter-bootstrap` 启动时可能会报：`No spring.config.import property has been defined`

- 注意与 Spring Boot 的对应关系
  - Spring Boot 3.0 对应 4.0 及以上版本
  - Spring Boot 2.0 对应 3.0 版本

之后添加配置文件，注意必须以 bootstrap 命名，否则可能会读取不到配置中心的配置

- `bootstrap.yaml`

```yaml
spring:
  profiles:
    active: dev
```

- `bootstrap-dev.yaml`

```yaml
server:
  port: 10086
spring:
  application:
    name: nacos-config-test
  cloud:
    nacos:
      config:
        # 默认是 properties 格式
        file-extension: yaml
        # 依次添加节点地址，逗号分割
        server-addr: 192.168.16.101:8848, 192.168.16.102:8848, 192.168.16.103:8848
        # 默认 pulic
        # 如需使用自定义的 namespace，需到控制台中添加，并在这里填写命名空间 ID，注意不是命名空间名称哦
        namespace: 68a0e6f6-210a-4d12-85ff-533d6daa9264
        # 默认 DEFAULT_GROUP
        group: dev_group
        # 再多说一遍，不要使用默认的用户名和密码
        username: nishuishui
        password: woshinidie
```

现在就可以在 Nacos 中添加配置了，注意 namespace、group 要与配置文件中的相同

另外再注意一下 Data ID 的填写规则：`${prefix}-${spring.profile.active}.${file-extension}`

```shell
${prefix}-${spring.profiles.active}.${file-extension}

# prefix：默认为 `spring.application.name` 的值，也可通过配置项 spring.cloud.nacos.config.prefix 来配置
# spring.profiles.active：当前启用的环境，没有即为空，对应的连接符 - 也就不需要添加了
# file-extension：文件格式
```

所以我最终在配置中心中创建的配置的 Data ID 就为 `nacos-config-test-dev.yaml`

接下来就可以愉快的使用了，通过 `@Value` 或 `@ConfigurationProperties` 获取配置，并且可以在类上加入 `@RefreshScope` 以实现热更新

## 共享配置和扩展配置

```yaml
spring:
  cloud:
    nacos:
      config:
        extension-configs:
          - data-id: extension1.yaml
            group: normal
            refresh: true
          - data-id: extension2.yaml
            group: normal
            refresh: true
        shared-configs:
          - data-id: common1.yaml
            group: DEFAULT_GROUP
            refresh: true
          - data-id: common2.yaml
            group: DEFAULT_GROUP
            refresh: true
```

通过 `extension-configs` 或 `shared-configs` 都可以多个配置，但他们的侧重点不同

- `extension-configs` 侧重于额外添加的配置
- `shared-configs` 侧重于一个共享的配置，一般也把他的 group 设置为 DEFAULT_GROUP

## 参考

- [SpringBoot项目如何从Nacos配置中心动态读取配置信息](https://www.cnblogs.com/gaopengpy/p/14628360.html)
- [Nacos Spring Cloud配置管理指定file-extension的格式为yaml不生效](https://blog.csdn.net/hejunfei/article/details/123082936)
- [Spring Cloud + Nacos 项目启动失败【No spring.config.import property has been defined】](https://blog.csdn.net/qq_41460383/article/details/134598168)
- [Nacos配置中心、注册中心详解（配置文件命名规则、extension-configs、shared-configs的作用、加载优先级）](https://blog.csdn.net/qq_43331014/article/details/131317715)
