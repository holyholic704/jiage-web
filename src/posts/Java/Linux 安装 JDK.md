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
# 查看是否已安装 JDK
# 如需卸载请跳到下方卸载章节
yum list installed|grep jdk

# 安装 JDK
# -y 表示对安装时的所有询问都回答 yes
yum install java-1.8.0-openjdk-devel.x86_64 -y

# 验证是否安装成功
java -version
```

- 也可以使用 `yum search jdk` 搜索自己想要的版本
- 推荐下载带有 devel 后缀的包，包含了 jstat、jps、jmap 等调试工具
- yum 安装的默认路径为：`/usr/lib/jvm`
  - 如果忘了也可通过 `whereis java` 进行搜索

配置环境变量

```shell
vim /etc/profile
```

在末尾添加

```shell
#set java environment
JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.412.b08-1.el7_9.x86_64
PATH=$PATH:$JAVA_HOME/bin
CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export JAVA_HOME CLASSPATH PATH
```

重载配置生效

```shell
source /etc/profile
```

## 卸载

```shell
# 卸载所有 openjdk 相关文件输入
yum -y remove java-1.8.0-openjdk*

# 卸载 tzdata-java
yum -y remove tzdata-java.noarch  
```

## 参考

- [Linux下安装jdk的两种方法](https://www.cnblogs.com/Dr-wei/p/13339957.html)
- [Linux安装jdk的详细步骤](https://blog.csdn.net/qq_41694906/article/details/126372085)
- [openjdk没有jstack、jstat、jps等命令解决办法](https://blog.csdn.net/tl4832194/article/details/107690496)
- [CentOS 7 yum卸载jdk、安装jdk以及配置jdk环境](https://cloud.tencent.com/developer/article/2091158)
- [yum -y与 yum有什么区别](https://blog.csdn.net/OUCFSB/article/details/80106993)
