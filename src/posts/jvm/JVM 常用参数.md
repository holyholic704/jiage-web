---
date: 2024-08-04
category:
  - JAVA
tag:
  - JVM
excerpt: JVM 常用的启动参数，GC 相关参数配置
order: 99
---

# JVM 常用参数

## 堆栈

- `-Xms`：堆内存初始大小，堆内存的最小值
  - 默认为物理内存的 1/64，但小于 1G
- `-Xmx`：堆内存的最大值
  - 默认为物理内存的 1/4，但小于 1G
- `-Xmn`：新生代大小
  - 一般会设置为堆内存的一半
  - `堆内存 = 老年代 + 新生代`
  - `新生代 = Eden 区 + Survivor0 区 + Survivor1 区`
- `-XX:NewSize`：新生代大小，常与 `-XX:MaxNewSize` 组合使用
  - `-Xmn` 相当于 `-XX:NewSize` 与 `-XX:MaxNewSize` 设置成相同的大小
- `-XX:NewRatio`：新生代与老年代的比值
  - 默认值为 2，即老年代是新生代的 2 倍
- `-XX:SurvivorRatio`：一个 Survivor 区与 Eden 区的比值
  - 默认值为 8，即 Eden 区是一个 Survivor 区的 8 倍
- `-Xss`：每个线程对应的栈内存大小
  - 默认为 1M

### 为什么建议 `Xms` 和 `Xmx` 设置成同一大小

首先我们要知道堆内存扩容和缩容的阈值

- 扩容：默认空余堆内存 **小于 40%** 时，直到堆内存的最大容量
- 缩容：默认空余堆内存 **大于 70%** 时，直到堆内存的最小限制

如果堆内存初始大小设置的过小，在使用时就可能会频繁的触发 GC，GC 就意味着停顿（STW，Stop The World），过多的 GC 会影响应用的响应速度，甚至可能会引发应用卡顿和响应超时

- Stop The World 无法避免，只能尽可能的减少停顿次数和时长

当 GC 过后空余的堆内存仍无法满足需求，才会进行堆内存的扩容，JVM 需要向操作系统申请额外的内存空间，也需要耗费点时间，甚至可能还会出现没有内存可分配的情况

某些 JVM（例如广泛使用的 OpenJDK）默认不会主动向操作系统退还未用的内存，所以与其让堆内存一步步扩容到最大容量，不如直接将其的初始容量就设置为最大容量

- JVM 不太情愿的原因是归还内存是有一定代价的，根据使用的垃圾收集器不同，代价也不同
  - 例如使用标记-清除算法（Mark-Sweep）的收集器，内存是不规整的，如需归还，还需要一番整理，代价过大

也可通过以下参数，让 JVM 主动释放内存，不过 GC 时会消耗更多的 CPU

```shell
# 堆内存最大空闲率（0 ~ 100），超过该值，JVM 会主动释放未使用的内存
-XX:MaxHeapFreeRatio=70

# 堆内存最小空闲率（0 ~ 100），低于该值，JVM 会扩展堆内存
-XX:MinHeapFreeRatio=40

# 期望 GC 时间占用程序运行时间的比例 1 / (n + 1)，取值 1 ~ 99
# 如果 n 取 99，就表示希望 GC 时间只占用运行时间的 1 / (99 + 1)，即 1%
-XX:GCTimeRatio=99

# 综上，如果希望 JVM 主动释放内存，建议使用如下配置
# JVM 最多会使用 20% 的 CPU 来处理 GC
-XX:MinHeapFreeRatio=5 -XX:MaxHeapFreeRatio=10 -XX:GCTimeRatio=4
```

### 为什么实际上 Eden 区与 Survivor 区的比值不是 8:1 呢

JDK1.8 默认使用 Parallel 收集器，该收集器默认开启了 AdaptiveSizePolicy，会根据 GC 的情况，动态的调整 Eden、From、To 区的大小，以满足 3 个预期目标

- 预期的 GC 停顿时间
  - 如果 GC 停顿时间超过了预期值，会减少内存大小，使用的内存越小，GC 时的耗时也就越少
  - 可通过 `-XX:MaxGCPauseMillis` 设置
- 预期的吞吐量
  - 如果吞吐量小于预期值，会增加内存大小，以减少 GC 频率
  - 可通过 `-XX:GCTimeRatio` 设置
- 尽可能少的内存占用
  - 首先需满足前两个前提

所以如果使用的是 JDK1.8，且没有进行特殊的设置，就经常会出现 Survivor 区很小的情况。如果 Survivor 区空间无法容纳 Young GC 后的存活的对象，这些对象会直接晋升到老年代，老年代的占用越来越大，直到空间不足触发 Full GC

- 目前在使用的被称为 From Survivor 区，GC 时被移动到的为 To Survivor 区，而 Survivor0（S0）、Survivor1（1）是实际上的空间划分（也依旧是逻辑上的划分）

```shell
# 开启自适应调节策略
-XX:+UseAdaptiveSizePolicy

# 关闭自适应调节策略
-XX:-UseAdaptiveSizePolicy
```

- CMS 与 ParNew 都不支持 AdaptiveSizePolicy

## 元空间（方法区）

- `-XX:MetaspaceSize`：元空间默认大小
  - 默认 20.79M
  - 设置的值作为触发 Full GC 的阈值，该值只对触发有效，触发 Full GC 后，JVM 会计算新的值，之后的 Full GC 就跟 MetaspaceSize 没有关系了
- `-XX:MaxMetaspaceSize`：元空间最大值
  - 默认值为 -1，表示无穷大，受到系统内存的限制

## 直接内存

- `-XX:MaxDirectMemorySize`：直接内存可使用的上限
  - 默认大小为堆内存的最大值

## 垃圾收集

- `-XX:+UseSerialGC`：新生代和老年代都使用串行收集器
  - 新生代：Serial
  - 老年代：Serial Old
- `-XX:+UseParNewGC`：新生代使用 ParNew 并行收集器
  - 老年代如无指定，默认使用 Serial Old
- `-XX:+UseParallelGC`：新生代使用 Parallel Scavenge 并行收集器
  - 老年代如无指定，默认使用 Parallel Old
- `-XX:+UseParallelOldGC`：老年代使用 Parallel Old 并行收集器
  - 新生代如无指定，默认使用 Parallel Scavenge
- `-XX:+UseConcMarkSweepGC`：老年代使用 CMS 收集器
  - 新生代如无指定，默认使用 ParNew
- `-XX:+UseG1GC`：使用 G1 收集器

### 并行收集器相关参数

- `-XX:ParallelGCThreads`：并行收集器的线程数，即同时使用多少个线程进行垃圾回收
  - 默认为 CPU 核心数
- `-XX:GCTimeRatio`：期望 GC 时间占用程序运行时间的比例 1 / (n + 1)，取值 1 ~ 99
  - 设置的值越大，表示 GC 时间占用系统运行时间越少
- `-XX:MaxGCPauseMillis`：期望每次 GC 最大的停顿毫秒数
  - 如果无法满足设置的时长，JVM 会自动调整新生代的大小，以满足要求，如果仍无法满足需求，则会牺牲 GCTimeRatio 的要求

### CMS 收集器相关

- `-XX:CMSInitiatingOccupancyFraction`：当老年代使用的内存比例到达设定的阈值，就会开始针对老年代的 GC（Major GC）
  - 默认值为 92
- `-XX:UseCMSCompactAtFullCollection`：在进行 Full GC 时，同时进行内存碎片的合并整理
  - 默认开启
  - JDK 9 已废弃
- `-XX:CMSFullGCsBeforeCompaction`：在多少次 Full GC 后，下一次进行 Full GC 前会进行一次碎片整理
  - 默认值为 0，表示每次进行 Full GC 前会进行一次碎片整理
  - JDK 9 已废弃

### GC 日志

- `-Xloggc:<filename>`：日志文件的输出路径
- `-XX:+PrintGC`：打印 GC 的简要信息
- `-XX:+PrintGCDetails`：打印 GC 的详情

```shell
# PrintGC
[Full GC (Metadata GC Threshold)  36413K->31034K(497664K), 0.1251295 secs]

# PrintGCDetails
[Full GC (Metadata GC Threshold) [PSYoungGen: 14844K->0K(148480K)] [ParOldGen: 21547K->30724K(349696K)] 36392K->30724K(498176K), [Metaspace: 56130K->56130K(1101824K)], 0.1234097 secs] [Times: user=0.20 sys=0.00, real=0.12 secs]

# Full GC (Metadata GC Threshold)：GC 的种类，括号中为触发 GC 的原因
# PSYoungGen: 14844K->0K(148480K)：垃圾收集的区域及 GC 前后内存使用量，括号中为该区域的总大小
# 36392K->30724K(498176K)：GC 前后堆内存的使用量，括号中为堆内存的总大小
# 0.1234097 secs：该次 GC 的时长
# Times: user=0.20 sys=0.00, real=0.12 secs
#   user=0.20：所有 GC 线程消耗的 CPU 时间
#   sys=0.00：系统调用和系统等待事件消耗的时间
#   real=0.12 secs：应用停顿的时间
```

- `-XX:+PrintGCDateStamps`：打印 GC 发生时的具体日期时间
- `-XX:+PrintGCTimeStamps`：打印 GC 发生时的时间，从应用启动到 GC 发生时的时间间隔

```shell
# PrintGCDateStamps
# 也包括从应用启动到 GC 发生时的时间间隔，所以一般使用这个参数就可以了
2024-08-04T14:05:01.066+0800: 119.320: [GC (Metadata GC Threshold) 

# PrintGCTimeStamps
240.742: [GC (Metadata GC Threshold) 
```

- `-XX:+PrintTenuringDistribution`：打印对象分布

```shell
Desired survivor size 59244544 bytes, new threshold 15 (max 15)
- age   1:     963176 bytes,     963176 total
- age   2:     791264 bytes,    1754440 total
- age   3:     210960 bytes,    1965400 total
- age   4:     167672 bytes,    2133072 total
- age   5:     172496 bytes,    2305568 total
- age   6:     107960 bytes,    2413528 total
- age   7:     205440 bytes,    2618968 total
- age   8:     185144 bytes,    2804112 total
- age   9:     195240 bytes,    2999352 total
- age  10:     169080 bytes,    3168432 total
- age  11:     114664 bytes,    3283096 total
- age  12:     168880 bytes,    3451976 total
- age  13:     167272 bytes,    3619248 total
- age  14:     387808 bytes,    4007056 total
- age  15:     168992 bytes,    4176048 total
```

- `-XX:+PrintHeapAtGC`：进行 GC 前后打印出堆的信息

```shell
{Heap before GC invocations=11 (full 3):
 PSYoungGen      total 149504K, used 134656K [0x00000000f5580000, 0x0000000100000000, 0x0000000100000000)
  eden space 134656K, 100% used [0x00000000f5580000,0x00000000fd900000,0x00000000fd900000)
  from space 14848K, 0% used [0x00000000fd900000,0x00000000fd900000,0x00000000fe780000)
  to   space 20480K, 0% used [0x00000000fec00000,0x00000000fec00000,0x0000000100000000)
 ParOldGen       total 349696K, used 31057K [0x00000000e0000000, 0x00000000f5580000, 0x00000000f5580000)
  object space 349696K, 8% used [0x00000000e0000000,0x00000000e1e54498,0x00000000f5580000)
 Metaspace       used 60749K, capacity 64134K, committed 64256K, reserved 1105920K
  class space    used 7488K, capacity 8090K, committed 8192K, reserved 1048576K

Heap after GC invocations=11 (full 3):
 PSYoungGen      total 154112K, used 6311K [0x00000000f5580000, 0x0000000100000000, 0x0000000100000000)
  eden space 133632K, 0% used [0x00000000f5580000,0x00000000f5580000,0x00000000fd800000)
  from space 20480K, 30% used [0x00000000fec00000,0x00000000ff229d20,0x0000000100000000)
  to   space 20480K, 0% used [0x00000000fd800000,0x00000000fd800000,0x00000000fec00000)
 ParOldGen       total 349696K, used 31057K [0x00000000e0000000, 0x00000000f5580000, 0x00000000f5580000)
  object space 349696K, 8% used [0x00000000e0000000,0x00000000e1e54498,0x00000000f5580000)
 Metaspace       used 60749K, capacity 64134K, committed 64256K, reserved 1105920K
  class space    used 7488K, capacity 8090K, committed 8192K, reserved 1048576K
}
```

- `-XX:+PrintGCApplicationStoppedTime`：打印 STW（Stop The World）的时长

```shell
Total time for which application threads were stopped: 0.0002086 seconds, Stopping threads took: 0.0000623 seconds
```

#### GC 日志输出

通过 `-Xloggc` 指定的日志文件，采用的是覆盖写的方式，每次应用启动都会覆盖，推荐使用时间戳命名文件 `-Xloggc:gc-%t.log`，每次启动时会用时间戳命名日志文件。还可以多添加 `%p`，表示启动的进程 ID

```shell
# 使用 gc-%t-%p.log
gc_2024-08-04_14-04-55_pid8931.log
```

不过日志总是越来越大的，不仅会占用更大的空间，过多的日志也会增加分析时的难度，通常来说，我们查看日志时只会关心最新的几条

我们可以为 GC 日志添加一个分割规则，满足条件时自动对日志进行分割，这样单个文件的占用的空间就小多了，我们也只需要关注最新的文件，旧的文件可以通过脚本进行压缩存放

```shell
# 开启日志文件分割
-XX:+UseGCLogFileRotation

# 分割的文件上限，超过上限就会从头开始写，所以推荐在为日志命名时添加 %t 参数，这样就不会覆盖旧日志了
# 默认值为 1
-XX:NumberOfGCLogFiles=10

# 单个文件的上限
# 默认为 512K，即单个文件大于 512K 时会自动分割
-XX:GCLogFileSize=20M
```

## 参考

- [Why to set -Xms and -Xmx to the same value?](https://developer.jboss.org/message/532435#532435)
- [Is there any advantage in setting Xms and Xmx to the same value?](https://stackoverflow.com/questions/43651167/is-there-any-advantage-in-setting-xms-and-xmx-to-the-same-value)
- [JVM的Xms和Xmx参数设置为相同值有什么好处？](https://www.choupangxia.com/2020/09/07/jvm-xms-xmx/)
- [JVM优化关键：SafePoint与Stop-The-World深度解析](https://cloud.baidu.com/article/3316534)
- [7.4.2.2. 了解如何促使 JVM 向操作系统释放未用的内存](https://docs.redhat.com/zh_hans/documentation/openshift_container_platform/4.7/html/nodes/nodes-cluster-resource-configure-jdk-unused_nodes-cluster-resource-configure)
- [jvm内存释放给os](https://blog.csdn.net/shengruxiahua2571/article/details/129485657)
- [Does GC release back memory to OS?](https://stackoverflow.com/questions/30458195/does-gc-release-back-memory-to-os)
- [小知识点 之 JVM -XX:MaxGCPauseMillis 与 -XX:GCTimeRatio](https://www.cnblogs.com/hellxz/p/14056403.html)
- [运维：你们 JAVA 服务内存占用太高，还只增不减！告警了，快来接锅](https://segmentfault.com/a/1190000040050819)
- [JVM源码分析之MetaspaceSize和MaxMetaspaceSize的区别](https://blog.csdn.net/duqi_2009/article/details/102097093)
- [Java 垃圾回收权威指北](https://liujiacai.net/blog/2019/01/09/java-gc-definitive-guide/)
- [JVM性能——垃圾回收器的优化策略](https://blog.csdn.net/qq330983778/article/details/127644441)
- [JVM内存管理------垃圾搜集器参数精解](https://www.cnblogs.com/zuoxiaolong/p/jvm9.html)
- [JVM调优-常见的垃圾回收器](https://juejin.cn/post/7127268544743473160)
- [JVM参数之UseAdaptiveSizePolicy](https://www.cnblogs.com/dengq/p/13687596.html)
- [见了鬼，我JVM的Survivor区怎么只有20M了？](https://juejin.cn/post/6844904192537018381)
- [Dead code related to UseAdaptiveSizePolicy in ParNewGeneration](https://bugs.openjdk.org/browse/JDK-8213113)
- [JVM参数配置说明](https://help.aliyun.com/zh/sae/use-cases/jvm-options)
- [最重要的JVM参数总结](https://javaguide.cn/java/jvm/jvm-parameters-intro.html)
- [How can I add timestamps GC log file names in a Java Service under Windows?](https://stackoverflow.com/questions/27822477/how-can-i-add-timestamps-gc-log-file-names-in-a-java-service-under-windows)
- [求你了，GC 日志打印别再瞎配置了](https://segmentfault.com/a/1190000039806436)
- [Rolling garbage collector logs in java](https://stackoverflow.com/questions/3822097/rolling-garbage-collector-logs-in-java)
- [Try to Avoid -XX:+UseGCLogFileRotation](https://dzone.com/articles/try-to-avoid-xxusegclogfilerotation)
- [GC日志解读，这次别再说看不懂GC日志了](https://juejin.cn/post/7029130033268555807)
