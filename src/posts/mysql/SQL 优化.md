---
date: 2024-12-04
category:
  - 数据库
tag:
  - MySQL
  - 索引
excerpt: SQL 优化建议
order: 99
---

# SQL 优化

## 减少与数据库的交互

一次交互可以搞定的，就不要多次交互

### 对于多个相同的操作可以使用批量操作

但批量操作所针对的数据量不宜过大，过大所需的处理时间也更长

## 拆分复杂的大 SQL 为多个小 SQL

一般来说越复杂的 SQL 执行所需的时间越长，可以考虑将复杂的 SQL 拆分成多个小 SQL，不仅逻辑更清晰，还可通过并行操作提高效率

## 关联查询

### 将经常一起使用的字段放在同一个表中

避免过多的关联查询，也可适当的添加冗余字段，或者中间表

### 避免多表 `JOIN`，最好不要超过 3 个表

尽量使用小表驱动大表

## 子查询

避免使用子查询，可以把子查询优化为关联查询，或者拆分成多个查询

子查询的结果集会被存储到临时表中，无法使用索引，所以会消耗过多的 CPU 和 IO 资源，产生大量的慢查询

## 尽量使用覆盖索引，避免回表

## 避免使用 `SELECT *`

应指定需要查询的字段，数据太多会导致查询性能下降，尽量确保这些字段都在索引中

## 大数据量应避免 `COUNT` 查询

`COUNT` 会扫描整个表中匹配的行数，当表中的数据特别多时会比较耗时

- 查询的字段尽量加上索引，以提升查询效率
- 使用缓存
- 使用汇总字段或表
- 使用近似值
  - 使用 `EXPLAIN` 获取 ROWS 列的值

## 尽量用 `UNION ALL` 代替 `UNION`

`UNION` 有去重操作，会把两个结果集的所有数据放到临时表中进行去重操作

## `ORDER BY`

应尽可能使用索引，遵守最左匹配原则，且多个字段排序顺序要一致，尽量搭配 `LIMIT` 使用

## 避免产生大事务操作

大事务可能会产生大量的阻塞

## 尽量不要在数据库做运算

复杂运算应移到业务应用里完成

### 让索引列在比较表达式中单独出现

如果索引列在比较表达式中不是以单独列的形式出现，而是以某个表达式，或者函数调用形式出现的话，是用不到索引的

```sql
# 无法使用索引
WHERE my_col * 2 < 4

# 可以使用索引
WHERE my_col < 4/2
```

## 避免隐式转换

查询字段是什么类型的，查询条件就要保持相同的类型。当操作符左右两边的数据类型不一致时，会发生隐式转换，可能会导致索引失效，触发全表扫描

## 遵守最左匹配原则

搜索条件中可以不用包含全部联合索引中的列，可以只包含左边一个或左边连续多个列（顺序不影响）

其实建立索引，也就是进行一个排序操作，先按字段 A 排，字段 A 相同时，再按 B 排，依次类推。如果跳过字段 A，直接使用字段 B 是走不了索引的，因为字段 B 只有在字段 A 相同时，才是有序的

### 不满足最左匹配原则也可以走索引

注意不满足最左匹配原则也不是就不能使用索引，在 **满足索引覆盖** 的前提下，也是可以走索引的，但需要遍历整个索引

虽然遍历整个二级索引近似于全表扫描，但二级索引记录要比聚簇索记录小的多，而且因为满足了索引覆盖条件，也不需要进行回表，所以直接遍历二级索引比直接遍历聚簇索引的成本要小很多

```sql
# 为列 A、B、C 建立联合索引

# 走索引
# 只需要遍历索引中 A=1 的部分
SELECT A, B, C FROM test where A = 1;

# 走索引
# 但需要遍历整个索引
SELECT A, B, C FROM test where B = 1;

# 不能走索引，进行全表扫描
SELECT A, B, C, D FROM test where B = 1;
```

## 避免使用 `%` 开头的模糊查询

对于字符串类型的索引列来说，只匹配它的前缀也是可以快速定位记录，因为字符串在排序时，也是逐个比较字符的大小的

但使用 `%` 开头的模糊查询是不走索引的，类似违反最左匹配原则，当然如果满足索引覆盖，也是可能走索引的

## 使用 `OR`

使用 `OR` 子句时，前后出现的列都要是索引列才可以走索引

## 扫描区间不宜过大

使用 `IN`、`NOT IN`、`IS NULL`、`IS NOT NULL`、`!=`、`>`、`<`、`>=`、`<=`、`BETWEEN`、`LIKE` 这些操作都会产生一个需要扫描的范围，也就是扫描区间

优化器会针对可能使用到的二级索引划分几个扫描区间，然后分别调查这些区间内有多少条记录，在这些扫描区间内的二级索引记录的总和占总记录数量的比例达到某个值时，优化器将放弃使用二级索引执行查询，转而采用全表扫描

也就是说取值范围较大时会导致索引失效，走全表扫描

## 排序

`ORDER BY` 的子句后边的列的顺序，也必须按照索引列的顺序给出，当然也适用最左匹配原则

- 注意不可同一条查询内 `ASC`、`DESC` 混用

## 慢 SQL

```sql
# 查看是否开启了慢 SQL 日志
SHOW VARIABLES LIKE 'slow_query%';

# 查看超过多少秒才算慢 SQL
SHOW VARIABLES LIKE 'long_query_time';
```

可通过命令和修改配置文件来开启慢 SQL 日志，一般推荐修改配置文件

```bash
slow_query_log = ON
slow_query_log_file = /usr/local/mysql/slow.log
long_query_time = 1
```

慢 SQL 日志是逻辑日志，记录了 SQL 语句和相关的执行时间，但数据分布分散，不好分析，因此，一般会使用 `mysqldumpslow` 工具进行分析

```sql
# -s：排序方式
#   at：平均执行时间
#   al：平均锁定时间
#   ar：平均返回行数
#   c：执行数量
#   t：执行时间
#   l：锁定时间
#   r：返回记录数
# -t：输出前多少条数据
# -g：正则匹配
mysqldumpslow [ OPTS... ] [ LOGS... ]

# 查询按照执行时间排序的前十条慢 SQL
mysqldumpslow -s t -t 10 /usr/local/mysql/slow.log
```

## 参考

- [MySQL 是怎样运行的：从根儿上理解 MySQL](https://juejin.cn/book/6844733769996304392)
- 《高性能MySQL（第4版）》
- [MySQL高性能优化规范建议总结](https://javaguide.cn/database/mysql/mysql-high-performance-optimization-specification-recommendations.html)
- [MySQL慢查询（一） - 开启慢查询](https://www.cnblogs.com/luyucheng/p/6265594.html)
- [Mysql如何分析慢查询日志--MysqlDumpSlow详解](https://www.cnblogs.com/yuanwanli/p/9024716.html)
- [MySQL数据库调优进阶详解](https://blog.csdn.net/J080624/article/details/53199410)
- [MySQL调优之服务器参数优化实践](https://blog.csdn.net/J080624/article/details/88065417)
