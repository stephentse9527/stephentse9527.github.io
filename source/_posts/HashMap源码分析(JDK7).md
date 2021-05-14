---
title: HashMap源码分析(JDK7)
date: 2021-05-03 16:56:19
tags: ['Java', '源码分析']
---

## HashMap源码分析-JDK1.7

### 一、jdk1.7容器初始化

> 无参构造函数

```java
/**
 * DEFAULT_INITIAL_CAPACITY = 16
 * DEFAULT_LOAD_FACTOR = 0.75
**/
public HashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}
```

> 两个参数构造函数

```java
public HashMap(int initialCapacity, float loadFactor) {
	// 不重要的代码已经直接不放了
    this.loadFactor = loadFactor;
    threshold = initialCapacity;
    init();
}
```

使用 threshold 来暂时保存 initialCapacity，init() 是用来初始化llinkedHashMap的

### 二、jdk1.7添加put

##### 2.1、HashMap的put方法

```JAVA
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        // 初始化table数组
        inflateTable(threshold);
    }
    if (key == null)
        return putForNullKey(value);
    // 获取hash值
    int hash = hash(key);
    // 与数据长度减1进行与操作获得桶下标
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    // 添加方法
    addEntry(hash, key, value, i);
    return null;
}
```

##### 2.2、HashMap的初始化方法

初始化就是把传进来的初始容量变成一个最小的2次幂

```java
// 根据之前传进来的值找到一个的大于等于这个数的二次幂
private void inflateTable(int toSize) {
    // 找到一个2的次幂，大于等于这个size
    int capacity = roundUpToPowerOf2(toSize);
	
    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
}
```

##### 2.3、添加元素方法 addEntry

直接使用头插法添加元素

```java
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        // 超过阈值并且桶不为空会进行扩容
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

// 真正添加元素的方法，直接使用头插法添加
void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

2.4、内部二次hash方法

```java
// 面试这么回答: 对于String类，使用stringHash32来计算hash值
// 而其他类型则采用了异或的方式，对多个地方进行异或
// 先将hash值无符号右移20位进行一次异或，然后右移14位再进行一次，7，4也各来一次
final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}
```

### 三、hashmap扩容

生成两倍的数组，并进行元素的重新hash插入

```java
void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    // 如果超过阈值，就直接返回，不进行扩容
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```

元素转移的实现

> 由于插入和扩容都是头插，可能在多线程的时候导致链表头尾相连，造成死循环

```java
void transfer(Entry[] newTable, boolean rehash) {
    int newCapacity = newTable.length;
    // 对于每一个桶，获取到头节点进行重新的hash，也是使用的头插法
    for (Entry<K,V> e : table) {
        while(null != e) {
            Entry<K,V> next = e.next;
            if (rehash) {
                e.hash = null == e.key ? 0 : hash(e.key);
            }
            int i = indexFor(e.hash, newCapacity);
            e.next = newTable[i];
            newTable[i] = e;
            e = next;
        }
    }
}
```

> 总结：
>
> - 对于jdk1.7的HashMap来说，采用的是数组加链表的形式存储数据，使用Entry来保存每个键值对，虽然提供了多个构造函数，对于传进来的初始容量，都只会把它保存在threadhold中不做任何操作， 第一次put的时，会先检查table数组是否为空，如果为空，则进行table的初始化，根据初始值获得大于等于初始值的2的次方，比如13则为16，创建桶数组
> - 对于put操作，会先获得对象自身的hash值，并进行第二次hash，如果是String，则调用Stringhash32方法获取hash值，而其他类型，则分别对原hash值右移20位，14位，7位，4位进行一个异或操作，获得最终的hash值，并和数组长度-1进行与操作获得数组桶下标
> - 遍历这个桶，如果在这个桶中找到key和要插入的key一致，则覆盖value值，如果不是则使用头插法把节点放到桶的第一个元素上
> - 而对于扩容，当容量超过阈值后，就进入扩容，长度为原数组的两倍，并遍历每个桶，对桶的元素进行重新hash，并插入新table，也是使用的头插法