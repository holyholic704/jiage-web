---
date: 2024-06-03
category:
  - 数据库
tag:
  - MySQL
excerpt: COUNT 使用注意
order: 99
---

# COUNT

对于 `COUNT(*)`、`COUNT(常数)`、`COUNT(主键)` 来说，优化器可以选择最小的索引执行查询，从而提升效率，它们的执行过程是一样的，只不过对于 `NULL` 值有不同的判断方式，但这个判断为 `NULL` 的过程的代价可以忽略不计，所以可以认为 `COUNT(*)`、`COUNT(常数)`、`COUNT(主键)` 所需要的代价是相同的

对于 `COUNT(非主键列)` 来说，server 层必须要从 InnoDB 中读到包含非主键列的记录，所以优化器并不能随心所欲的选择最小的索引去执行

- `COUNT(*)`：包含 NULL 值
- `COUNT(常数)`：包含 NULL 值
- `COUNT(列名)`：不包含 NULL 值
