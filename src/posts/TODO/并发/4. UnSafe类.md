# Unsafe

Unsafe 位于 sun.misc 包下，主要提供一些用于执行低级别、不安全操作的方法，提升 Java 运行效率、增强 Java 语言底层资源操作能力。正如名称所示使用 Unsafe 类是有风险的，使用时需慎重

```java
private static final Unsafe theUnsafe;

private Unsafe() {
}

static {
    theUnsafe = new Unsafe();
}

@CallerSensitive
public static Unsafe getUnsafe() {
    Class var0 = Reflection.getCallerClass();
    if (!VM.isSystemDomainLoader(var0.getClassLoader())) {
        throw new SecurityException("Unsafe");
    } else {
        return theUnsafe;
    }
}
```

Unsafe 通过饿汉式单例模式创建实例，而且构造方法是私有的，只能在该类中创建对象，提供了 getUnsafe 方法作为唯一获取能获取到实例的方法。getUnsafe 方法是被 @CallerSensitive 修饰的，表示该方法有些敏感，建议开发者考虑好之后再去使用该方法

getUnsafe 方法内部作了判断，必须被启动类的类加载器加载，才可被使用，否则会抛出 SecurityException 异常，所以只有 Java 内部的一些类可以直接调用，但我们开发者可以通过反射直接获取 theUnsafe 字段，也就是 Unsafe 对象

```java
public class Test {

    public static void main(String[] args) throws Exception {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        Unsafe unsafe = (Unsafe) field.get(null);
    }
}
```

## 功能

### CAS 操作

Java 在原子类等并发工具中的 CAS 操作都是由此实现，主要提供了 3 种方法，依次传入要修改的对象、对象中某个字段的偏移量、期望值，新值

- 偏移量可以通过 objectFieldOffset 方法传入某个字段进行获取，也就是该字段的内存地址，主要是为了在更新操作在内存中找到该字段的位置，方便比较

```java
public final native boolean compareAndSwapObject(Object var1, long var2, Object var4, Object var5);

public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);

public final native boolean compareAndSwapLong(Object var1, long var2, long var4, long var6);
```

```java
// 获取某个字段在内存中的偏移量
public native long objectFieldOffset(Field var1);
```

- 例如 AtomicInteger 类

```java
public class AtomicInteger extends Number implements java.io.Serializable {

    private static final long valueOffset;

    private volatile long value;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }
}
```

调用 Unsafe 类的 compareAndSwapInt 方法，传入传入要修改的对象，即 AtomicInteger 对象；传入对象中某个字段的偏移量，即静态代码块里通过 getDeclaredField 获取的 value 字段的偏移量；传入期待值；传入新值

### 内存操作

主要包含堆外内存的分配、拷贝、释放、给定地址值操作等方法

我们在 Java 中创建的对象都在堆内存中，堆内存由 JVM 进行管理。与之相对的是堆外内存，处于 JVM 管控之外的内存区域，Java 中对堆外内存的操作，依赖于 Unsafe 提供的操作堆外内存的 native 方法

- 对垃圾回收停顿的改善：由于堆外内存是直接受操作系统管理而不是 JVM，所以当我们使用堆外内存时，即可保持较小的堆内存规模，从而在 GC 时减少回收停顿对于应用的影响
- 提升 IO 性能：在 IO 通信过程中，会存在堆内内存到堆外内存的数据拷贝操作，对于需要频繁进行内存间数据拷贝且生命周期较短的暂存数据，都建议存储到堆外内存

```java
// 分配内存
public native long allocateMemory(long var1);

// 扩充内存
public native long reallocateMemory(long var1, long var3);

// 内存初始化
public native void setMemory(Object var1, long var2, long var4, byte var6);

// 内存初始化
public void setMemory(long var1, long var3, byte var5) {
    this.setMemory((Object)null, var1, var3, var5);
}

// 内存拷贝，对象浅拷贝
public native void copyMemory(Object var1, long var2, Object var4, long var5, long var7);

// 内存拷贝，对象浅拷贝
public void copyMemory(long var1, long var3, long var5) {
    this.copyMemory((Object)null, var1, (Object)null, var3, var5);
}

// 释放内存
public native void freeMemory(long var1);
```

```java
// 获取给定地址的值（当且仅当该内存地址为allocateMemory分配时，此方法结果才是确定的），以下get方法相同
public native byte getByte(long var1);

// 为给定地址设置值（当且仅当该内存地址为allocateMemory分配时，此方法结果才是确定的），以下put方法相同
public native void putByte(long var1, byte var3);

public native short getShort(long var1);

public native void putShort(long var1, short var3);

public native char getChar(long var1);

public native void putChar(long var1, char var3);

public native int getInt(long var1);

public native void putInt(long var1, int var3);

public native long getLong(long var1);

public native void putLong(long var1, long var3);

public native float getFloat(long var1);

public native void putFloat(long var1, float var3);

public native double getDouble(long var1);

public native void putDouble(long var1, double var3);

public native long getAddress(long var1);

public native void putAddress(long var1, long var3);
```

- 例如

```java
public class Test {

    public static void main(String[] args) throws Exception {
        Field field = Unsafe.class.getDeclaredField("theUnsafe");
        field.setAccessible(true);
        Unsafe unsafe = (Unsafe) field.get(null);

        long size = 911;

        // 分配内存
        long address = unsafe.allocateMemory(size);
        // 内存初始化
        unsafe.setMemory(address, size, (byte) 0);
        // 设置值
        unsafe.putInt(address, 1024);
        // 获取值
        System.out.println(unsafe.getInt(address));
        // 释放内存
        unsafe.freeMemory(address);
    }
}
```

### 线程调度

LockSupport 类中的 park、unpark 方法就是通过 Unsafe 类中的 park、unpark 实现阻塞与唤醒的

```java
// 阻塞一个线程，可设置等待时间，0为无限制等待下去
public native void park(boolean var1, long var2);

// 唤醒一个线程
public native void unpark(Object var1);
```

### Class 相关

```java
// 获取给定静态字段的内存地址偏移量，这个值对于给定的字段是唯一且固定不变的
public native long staticFieldOffset(Field var1);

// 获取一个静态类中给定字段的对象指针
public native Object staticFieldBase(Field var1);

// 判断是否需要初始化一个类，通常在获取一个类的静态属性的时候
public native boolean shouldBeInitialized(Class<?> var1);

// 判断给定的类是否已经初始化，通常在获取一个类的静态属性的时候
public native void ensureClassInitialized(Class<?> var1);

// 定义一个类，此方法会跳过JVM的所有安全检查
public native Class<?> defineClass(String var1, byte[] var2, int var3, int var4, ClassLoader var5, ProtectionDomain var6);

// 定义一个匿名类
public native Class<?> defineAnonymousClass(Class<?> var1, byte[] var2, Object[] var3);
```

### 读写相关

- 普通读写，可以读写一个类的属性，包括私有的

```java
public native int getInt(Object var1, long var2);

public native void putInt(Object var1, long var2, int var4);

public native Object getObject(Object var1, long var2);

public native void putObject(Object var1, long var2, Object var4);

public native boolean getBoolean(Object var1, long var2);

public native void putBoolean(Object var1, long var2, boolean var4);

public native byte getByte(Object var1, long var2);

public native void putByte(Object var1, long var2, byte var4);

public native short getShort(Object var1, long var2);

public native void putShort(Object var1, long var2, short var4);

public native char getChar(Object var1, long var2);

public native void putChar(Object var1, long var2, char var4);

public native long getLong(Object var1, long var2);

public native void putLong(Object var1, long var2, long var4);

public native float getFloat(Object var1, long var2);

public native void putFloat(Object var1, long var2, float var4);

public native double getDouble(Object var1, long var2);

public native void putDouble(Object var1, long var2, double var4);
```

- volatile 读写，保证可见性和有序性，相比于普通读写性能低点

```java
public native Object getObjectVolatile(Object var1, long var2);

public native void putObjectVolatile(Object var1, long var2, Object var4);

public native int getIntVolatile(Object var1, long var2);

public native void putIntVolatile(Object var1, long var2, int var4);

public native boolean getBooleanVolatile(Object var1, long var2);

public native void putBooleanVolatile(Object var1, long var2, boolean var4);

public native byte getByteVolatile(Object var1, long var2);

public native void putByteVolatile(Object var1, long var2, byte var4);

public native short getShortVolatile(Object var1, long var2);

public native void putShortVolatile(Object var1, long var2, short var4);

public native char getCharVolatile(Object var1, long var2);

public native void putCharVolatile(Object var1, long var2, char var4);

public native long getLongVolatile(Object var1, long var2);

public native void putLongVolatile(Object var1, long var2, long var4);

public native float getFloatVolatile(Object var1, long var2);

public native void putFloatVolatile(Object var1, long var2, float var4);

public native double getDoubleVolatile(Object var1, long var2);

public native void putDoubleVolatile(Object var1, long var2, double var4);
```

- 有序写入，保证写入的有序性，不保证可见性

```java
public native void putOrderedObject(Object var1, long var2, Object var4);

public native void putOrderedInt(Object var1, long var2, int var4);

public native void putOrderedLong(Object var1, long var2, long var4);
```

### 对象操作

```java
// 返回对象成员属性在内存地址相对于此对象的内存地址的偏移量
public native long objectFieldOffset(Field var1);

// 绕过构造方法、初始化代码来创建对象
public native Object allocateInstance(Class<?> var1) throws InstantiationException;
```

### 数组相关

```java
// 返回数组中第一个元素的偏移地址
public native int arrayBaseOffset(Class<?> var1);

// 返回数组中一个元素占用的大小
public native int arrayIndexScale(Class<?> var1);
```

两者配合起来使用，即可定位数组中每个元素在内存中的位置

### 内存屏障

```java
// 禁止load操作重排序
public native void loadFence();

// 禁止store操作重排序
public native void storeFence();

// 禁止load、store操作重排序
public native void fullFence();
```

> 详见 [7. Java 内存模型 - 内存屏障分类](../JVM/7.%20Java%20内存模型.md)

### 系统相关

```java
// 返回系统指针的大小。返回值为4（32位系统）或 8（64位系统）
public native int addressSize();

// 内存页的大小
public native int pageSize();
```

## 参考

- [Java魔法类：Unsafe应用解析](https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html)
- [从原子类和UNSAFE来理解JAVA内存模型，ATOMICINTEGER的INCREMENTANDGET方法和UNSAFE部分源码介绍，VALUEOFFSET偏移量的理解](https://www.cnblogs.com/theRhyme/p/12129120.html)
