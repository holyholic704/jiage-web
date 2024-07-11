## HashSet

### 成员变量

HashSet 就是利用 HashMap 实现的，创建一个 HashSet 其实就是创建一个 HashMap

```java
// 存放元素的集合
private transient HashMap<E,Object> map;

// 作为value的值，所有元素共用一个
private static final Object PRESENT = new Object();
```

```java
public class Test {

    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException {
        Set<String> set = new HashSet<>();
        set.add("one");
        set.add("two");
        set.add("three");

        Field field = HashSet.class.getDeclaredField("map");
        field.setAccessible(true);

        System.out.println(field.get(set));
    }
}
```

> {one=java.lang.Object@2a33fae0, two=java.lang.Object@2a33fae0, three=java.lang.Object@2a33fae0}

可以看出所有的 value 用的都是同一个对象

### 构造方法

构造方法也与 HashMap 类似

```java
public HashSet() {
    map = new HashMap<>();
}

public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}

public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}

public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}

// dummy参数没有具体的意思，只是一个标记，用来区分HashSet(int initialCapacity, float loadFactor)
// 该构造方法没有被public修饰，所以是供内部使用的，用于对LinkedHashSet的支持
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

### 常用方法

```java
public Iterator<E> iterator() {
    return map.keySet().iterator();
}

public int size() {
    return map.size();
}

public boolean isEmpty() {
    return map.isEmpty();
}

public boolean contains(Object o) {
    return map.containsKey(o);
}

public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}

public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}

public void clear() {
    map.clear();
}
```

## LinkedHashSet

与 LinkedHashMap 继承自 HashMap 类似的，LinkedHashSet 也是继承自 HashSet，这时 HashSet 中的 `HashSet(int initialCapacity, float loadFactor, boolean dummy)` 构造方法就派上用处了

```java
public class LinkedHashSet<E>
    extends HashSet<E>
    implements Set<E>, Cloneable, java.io.Serializable {

    public LinkedHashSet(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor, true);
    }

    public LinkedHashSet(int initialCapacity) {
        super(initialCapacity, .75f, true);
    }

    public LinkedHashSet() {
        super(16, .75f, true);
    }

    public LinkedHashSet(Collection<? extends E> c) {
        super(Math.max(2*c.size(), 11), .75f, true);
        addAll(c);
    }

    @Override
    public Spliterator<E> spliterator() {
        return Spliterators.spliterator(this, Spliterator.DISTINCT | Spliterator.ORDERED);
    }
}
```

## TreeSet

TreeSet 使用了可排序的 Map，也就是 TreeMap

```java
public class TreeSet<E> extends AbstractSet<E>
    implements NavigableSet<E>, Cloneable, java.io.Serializable
{

    private transient NavigableMap<E,Object> m;

    private static final Object PRESENT = new Object();

    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }

    public TreeSet() {
        this(new TreeMap<E,Object>());
    }

    public TreeSet(Comparator<? super E> comparator) {
        this(new TreeMap<>(comparator));
    }

    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }

    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
}
```
