---
title: LinkedHashMap源码分析
date: 2021-05-03 20:28:47
tags: ['Java', '源码分析']
---

### LinkedHashMap源码分析

LinkedHashMap 继承于 HashMap，用来存储数据的Entry也是继承HashMap的Node节点，多了两个引用before，after，用来把节点变成双向链表

```java
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}
```

### 添加put方法

并没有去重写put方法，而是直接使用了HashMap的实现，在把节点插入对应的桶的链表时，才重写了newNode方法，也就是创建新节点，通过linkNodeLast方法将Entry接在双向链表尾部，所以**linkedHashMap是带有插入的顺序性的**

```java
private void linkNodeLast(LinkedHashMap.Entry<K,V> p) {
    LinkedHashMap.Entry<K,V> last = tail;
    tail = p;
    if (last == null)
        head = p;
    else {
        p.before = last;
        last.after = p;
    }
}
```

### 访问顺序的维护

当创建LinkedHashMap时，传入accessOrder参数为true，就可以开启访问顺序维护

```java
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    // 开启访问顺序维护
    this.accessOrder = accessOrder;
}
```

LinkedHashMap重写了get方法 ，如果accesOrder为true，就会调用afterNodeAccess方法，内部实现是将这个节点移动到双链表的**末尾**