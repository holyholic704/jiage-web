---
date: 2024-11-12
category:
  - JAVA
tag:
  - 基础
excerpt: BigDecimal 使用注意
order: 99
---

# BigDecimal 使用注意

## 不要使用构造方法将 double 类型的值转换为 Bigdecimal 对象

浮点数无法通过二进制精确的表示，所以在转化时可能会出现异常情况

```java
System.out.println(BigDecimal.valueOf(.1));
System.out.println(new BigDecimal(.1));
System.out.println(new BigDecimal(".1"));
```

> 0.1
> 0.1000000000000000055511151231257827021181583404541015625
> 0.1

## 使用 `compareTo` 方法进行等值比较

`equals` 方法会比较值和精度，`compareTo` 方法则会忽略精度

```java
BigDecimal b1 = BigDecimal.valueOf(1);
BigDecimal b2 = BigDecimal.valueOf(1.0);
System.out.println(b1.equals(b2));
System.out.println(b1.compareTo(b2));
```

> false
> 0
