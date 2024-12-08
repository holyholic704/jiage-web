# Spring Boot 集成 ShardingSphere-JDBC

使用 ShardingSphere-JDBC 只需引入依赖，并配置好配置文件即可使用，不需要编写额外的代码，不过配置较为复杂

```xml
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
    <version>5.2.1</version>
</dependency>

<!-- 如果启动报 snakeyaml 相关的错误，因为与 Spring Boot 的版本有冲突，需显式引入相关依赖 -->
<dependency>
    <groupId>org.yaml</groupId>
    <artifactId>snakeyaml</artifactId>
    <version>1.33</version>
</dependency>
```

## 只分表

```yaml
spring:
  shardingsphere:
    # 数据源，现在只分表所以只添加一个数据库
    datasource:
      # 数据库名
      names: ds0
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:mysql://127.0.0.1:3307/ds0
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: 123456
    # 分库分表规则
    rules:
      sharding:
        tables:
          # 逻辑表名
          user:
            # 实际表名
            # ds0 库下的 user_0、user_1、user_2
            actual-data-nodes: ds0.user_$->{0..2}
            # 主键生成策略
            key-generate-strategy:
              column: id
              key-generator-name: snowflake
            # 分表算法
            table-strategy:
              standard:
                # 分表算法使用的列
                sharding-column: id
                # 分表算法名称
                sharding-algorithm-name: user-inline
        # 分片算法
        sharding-algorithms:
          user-inline:
            type: INLINE
            props:
              # 分片算法
              algorithm-expression: user_$->{id % 3}
        # 主键生产器
        key-generators:
          snowflake:
            type: SNOWFLAKE
    # 是否展示执行的 SQL
    props:
      sql-show: true
```

## 只分库

```yaml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:mysql://127.0.0.1:3307/ds0
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: 123456
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:mysql://127.0.0.1:3307/ds1
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: 123456
    rules:
      sharding:
        # 分库规则
        default-database-strategy:
          standard:
            sharding-column: id
            sharding-algorithm-name: database-inline
        tables:
          user:
            actual-data-nodes: ds$->{0..1}.user
            key-generate-strategy:
              column: id
              key-generator-name: snowflake
        sharding-algorithms:
          database-inline:
            type: INLINE
            props:
              algorithm-expression: ds$->{id % 2}
        key-generators:
          snowflake:
            type: SNOWFLAKE
    props:
      sql-show: true
```

## 分库分表

```yaml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:mysql://127.0.0.1:3307/ds0
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: 123456
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:mysql://127.0.0.1:3307/ds1
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: 123456
    rules:
      sharding:
        default-database-strategy:
          standard:
            sharding-column: id
            sharding-algorithm-name: database-inline
        tables:
          user:
            actual-data-nodes: ds$->{0..1}.user_$->{0..2}
            key-generate-strategy:
              column: id
              key-generator-name: snowflake
            table-strategy:
              standard:
                sharding-column: id
                sharding-algorithm-name: user-inline
        sharding-algorithms:
          database-inline:
            type: INLINE
            props:
              algorithm-expression: ds$->{id % 2}
          user-inline:
            type: INLINE
            props:
              algorithm-expression: user_$->{id % 3}
        key-generators:
          snowflake:
            type: SNOWFLAKE
    props:
      sql-show: true
```

## 读写分离

- ShardingSphere-JDBC 不会处理主从之间的同步，也不支持主库多写

```yaml
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:mysql://127.0.0.1:3307/ds0
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: 123456
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        url: jdbc:mysql://127.0.0.1:3307/ds1
        driver-class-name: com.mysql.cj.jdbc.Driver
        username: root
        password: 123456
    rules:
      readwrite-splitting:
        data-sources:
          # 读写分离规则
          read_write_db:
            load-balancer-name: round_robin
            static-strategy:
              # 读库
              read-data-source-names:
                - ds0
                - ds1
              # 写库
              write-data-source-name: ds0
        # 负载均衡算法
            # 轮询：ROUND_ROBIN
            # 随机：RANDOM
            # 权重：WEIGHT
        load-balancers:
          round_robin:
            type: ROUND_ROBIN
    props:
      sql-show: true
```

## 参考

- [Spring Boot 2.7.5 shardingsphere-jdbc 5.2.1 报错](https://blog.csdn.net/lptnyy/article/details/128124412)
- [Springboot 2.6 + Mybatis Plus 3.5 集成 Sharding-jdbc 5.1 分库分表](https://blog.csdn.net/Mrqiang9001/article/details/124175654)
- [ShardingSphere实现读写分离](https://cloud.tencent.com/developer/article/2351749)
- [Spring Boot 集成 ShardingSphere-JDBC 配置示例](https://www.cnblogs.com/zlnp/p/16666428.html)
- [3.3. 读写分离 - 使用限制](https://shardingsphere.apache.org/document/current/cn/features/readwrite-splitting/limitations/)
