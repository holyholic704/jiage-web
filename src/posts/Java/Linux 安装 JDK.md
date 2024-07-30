---
date: 2024-07-29
category:
  - JAVA
tag:
  - 环境配置
excerpt: Linux 下安装 JDK
order: 99
---

# Linux 下安装 JDK

```shell
# 安装 JDK
yum install java-1.8.0-openjdk.x86_64

# 验证是否安装成功
java -version
```

- yum 安装的默认路径为：`/usr/lib/jvm`

配置环境变量

```shell
vim /etc/profile
```

在末尾添加

```shell
#set java environment
JAVA_HOME=/usr/lib/jvm/jre-1.8.0-openjdk-1.8.0.412.b08-1.el7_9.x86_64
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME CLASSPATH PATH
```

重载配置生效

```shell
source /etc/profile
```

## 参考

- [Linux下安装jdk的两种方法](https://www.cnblogs.com/Dr-wei/p/13339957.html)
- [Linux安装jdk的详细步骤](https://blog.csdn.net/qq_41694906/article/details/126372085)
