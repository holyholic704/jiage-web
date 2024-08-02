---
date: 2024-07-29
category:
  - 微服务
tag:
  - Nacos
  - 环境配置
excerpt: Nacos 的集群部署，Spring Boot 如何集成 Nacos
order: 99
---

# Nacos 集群部署及 Spring Boot 集成

- 需要配置好 Java 环境
- 注意版本的对应关系
- 建议服务器配置：2 核 4 G 及以上

理论上应该选用最新的稳定版本，但由于我目前使用 Java 版本还是 JDK1.8，不支持 Spring Boot 3.0，所以 Spring Boot 使用的版本是 2.7.6，对应的 Spring Cloud Alibaba 版本是 2021.0.5.0

所以我最终选用的 Nacos 版本是 2.2.0，详见 [版本说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)

## 安装

先到仓库上下载压缩包 [alibaba / nacos](https://github.com/alibaba/nacos/releases)，再将压缩包上传到服务器解压即可

## 部署

Nacos 默认使用嵌入式数据库来实现数据的存储，此时多个默认配置下的 Nacos 节点，数据存储就会存在一致性的问题

Nacos 采用了集中式存储的方式来支持集群化部署，目前只支持 MySQL 的存储

进入 `nacos/conf` 目录修改 `application.properties`

```properties
spring.datasource.platform=mysql
db.num=1
db.url.0=jdbc:mysql://127.0.0.1:3306/nacos?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true&useUnicode=true&useSSL=false&serverTimezone=UTC
db.user.0=账号
db.password.0=密码
```

以下是可选项，主要是为了保障 Nacos 的安全使用

```properties
# 建议不要使用默认端口，以减少被扫的频次
server.port=8848

# 建议开启鉴权开关
# 开启 Nacos 鉴权能力，就会拦截检查所有访问请求
nacos.core.auth.enabled=true

# 修改默认的 identity 键值对
nacos.core.auth.server.identity.key=tianwanggaidihu
nacos.core.auth.server.identity.value=baotazhenheyao

# 修改默认的 token.secret.key
# 长度至少 32 字符
nacos.core.auth.plugin.nacos.token.secret.key=5ZOl5YS/5Lus5ZCN5Y+r5LiB55yfIFNtb2tpbmcgcm91bmQsZS1jaWdhcmV0dGVzIG5ldyDmiJHnmoTlsI/pqazlkI3lrZflj6vnj43nj6Ag5YGH54Of5Y+R546w5bCx6LeR6LevIEJhYnkgSSBhaW4ndCBzbW9raW5nIGJ5IHlvdXIgcnVsZXMg6Iet6KaB6aWt55qE5Yir5oyh5oiR6LSi6LevIFdoeSB5b3UgYWx3YXlzIHNvIHBvb3Ig5Zug5Li65L2g5rKh5YWJ6aG+5oiR5bqX6ZO6IEkgYWluJ3QgdHJ5bmEgdGVsbCB5b3Ugd2hhdCB0byBkbyDkvaDoh6rlt7Hlv4Pph4zmnInmlbA=
```

配置完 `application.properties`，再进入 `nacos/conf` 目录修改 `cluster.conf`

- 如果没有 `cluster.conf` 文件，一般都会有一个 `cluster.conf.example` 文件，复制一份或者直接对其修改都行

```shell
# 理论上一个集群应该创建 3 个节点
192.168.16.101:8848
192.168.16.102:8848
192.168.16.103:8848
```

## 创建数据库

运行 `nacos/conf` 目录下的 `nacos-mysql.sql` 或 `mysql-schema.sql`

## 启动

运行 `nacos/bin` 目录下的 `startup.sh` 即可，如果没有指定以 standalone 模式运行，默认就是 cluster 模式

如遇启动问题，可查看 `nacos/logs` 目录下的 `start.out` 日志

## 开放端口

有 4 个端口需要开放

| 端口 | 与主端口的偏移量 | 描述 |
| :-: | :-: | :- |
| 8848 |  | 主端口 |
| 9848 | 1000 | 客户端 gRPC 请求服务端端口，用于客户端向服务端发起连接和请求 |
| 9849 | 1001 | 服务端 gRPC 请求服务端端口，用于服务间同步等 |
| 7848 | -1000 | Jraft 请求服务端端口，用于处理服务端间的 Raft 相关请求 |

## 优化内存占用

如果觉得 Nacos 内存占用过大，可以考虑修改 JVM 堆内存配置

进入 `nacos/bin` 目录修改 `startup.sh`

```shell
#===========================================================================================
# JVM Configuration
#===========================================================================================
if [[ "${MODE}" == "standalone" ]]; then
    JAVA_OPT="${JAVA_OPT} -Xms512m -Xmx512m -Xmn256m"
    JAVA_OPT="${JAVA_OPT} -Dnacos.standalone=true"
else
    if [[ "${EMBEDDED_STORAGE}" == "embedded" ]]; then
        JAVA_OPT="${JAVA_OPT} -DembeddedStorage=true"
    fi
    # 将集群模式下的堆内存
    JAVA_OPT="${JAVA_OPT} -server -Xms512m -Xmx512m -Xmn256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
    JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=${BASE_DIR}/logs/java_heapdump.hprof"
    JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages"

fi
```

## 不要使用默认的用户名和密码

Nacos 默认的用户名和密码为 `nacos / nacos`，为了安全保障，强烈建议使用自定义的用户名和密码

Nacos 有两种添加用户的方式，一种是启动后通过控制台创建，这个简单，不多赘述；另一种是直接在数据库中添加

```sql
> select * from users;
+----------+--------------------------------------------------------------+---------+
| username | password                                                     | enabled |
+----------+--------------------------------------------------------------+---------+
| nacos    | $2a$10$1E5ZSoDsbTerytQsCAc/j.r6IadfMkVr1Cz1o0h3zfXtD2pnLdSBO |       1 |
+----------+--------------------------------------------------------------+---------+

> select * from roles;
+----------+------------+
| username | role       |
+----------+------------+
| nacos    | ROLE_ADMIN |
+----------+------------+
```

具体方法就是在 users 表中添加自定义用户，或者直接将 username 修改成自定义的用户名，并且在 roles 表中进行授权

- 注意密码是需要 Bcrypt 加密的，网上找个在线的或者自己通过编程的手段进行加密

## 集成 Spring Boot

引入 Nacos 依赖

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.0.5.0</version>
</dependency>
```

添加配置

```yaml
spring:
  application:
    name: nacos-test
  cloud:
    nacos:
      discovery:
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

最后在启动类上添加 `@EnableDiscoveryClient` 注解就大功告成了

## 参考

- [版本说明](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)
- [Nacos部署环境](https://nacos.io/zh-cn/docs/deployment.html)
- [nacos占用内存高的问题](https://blog.csdn.net/qq_44403239/article/details/137780323)
- [Nacos 安全使用最佳实践 - 访问控制实践](https://nacos.io/blog/case-authorization/)
- [新版本部署](https://nacos.io/zh-cn/docs/v2/upgrading/2.0.0-compatibility.html)
