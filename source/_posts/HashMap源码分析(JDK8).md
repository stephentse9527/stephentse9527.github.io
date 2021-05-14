---
title: HashMap源码分析(JDK8)
date: 2021-05-03 17:00:48
tags: ['Java', '源码分析']
---

## HashMap源码分析(JDK8)

### 一、jdk1.8容器初始化

> 无参构造函数

```java
/**
 * DEFAULT_LOAD_FACTOR = 0.75
**/
public HashMap() {
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
}
```

> 两个参数构造函数

```java
public HashMap(int initialCapacity, float loadFactor) {
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}
```

使用 threshold 来暂时保存 initialCapacity，并直接计算出真正的初始容量

### 二、jdk1.8添加put

##### 2.1、HashMap的 putVal 方法

> 中文总结：putVal时，会先检查数组是否进行初始化，如果没有，就进行初始化
>
> 接下来通过与数组长度-1进行与操作获得桶下标，如果桶为空，直接放在桶上
>
> 如果不为空，则判断是否为红黑树节点，如果是，则用红黑树的putTreeVal来进行插入
>
> 否则就遍历桶数组，如果有key和插入的key一致，则覆盖，否则将节点插在链表末尾
>
> 插入后检查链表长度是否大于8，大于8则进行树化，然后size++，如果size大于阈值，则进行扩容操作

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 在第一次 putVal 时才会去初始化 table 数组
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
```

##### 2.2、HashMap的初始化方法

> 中文总结：1.8的扩容和初始化都在resize方法中进行，会先通过原数组的长度来判断要进行扩容还是初始化
>
> 当原数组长度为0时，说明是没进行初始化，会继续判断原阈值是否为0，当阈值不为0，则使用阈值进行初始化，因为初始长度暂时放在阈值字段中，如果为0，则使用默认的初始容量16，然后创建table数组返回

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    // 传入初始容量的情况
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 后面的扩容代码被我删除了，因为讲解的是初始化
    return newTab;
}
```

##### 2.3、内部二次hash方法

> 总结：获取对象本身的hash值，进行无符号右移16位，使得高16位移动到低16位，再和原来的进行异或操作
>
> 这样高16位就可以和低16位进行特征混合，这样就可以让高十六位也参与到hash值的运算中来，降低碰撞

```java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

### 三、hashmap扩容

> 总结：
>
> - 前面的初始化部分就不赘述了，扩容开始时，遍历每个非空桶数组，如果桶数组只有一个元素，则直接重新计算桶下标放到新数组中
> - 然后判断是否为树节点，如果是，则使用树的split方法进行转移
> - 最后的情况就是链表的转移了，首先会使用两个链表，然后对每个节点的hash值和原数组长度进行与操作，结果要么为0，就放到第一个链表当中，否则放到第二个链表当中，最后这个桶会分成两个链表，第一个结果为0的链表直接放在下标和原数组一样的新数组的桶中，另一个链表则是新数组中原数组下标+原数组长度的那个桶
> - 对每个桶都这样操作，就完成了元素的转移

为什么是这样转移的？

答：因为计算桶下标是和数组长度-1进行与操作，扩容成原来的二倍后，数组长度-1在二进制当中也就是多了一个位从0变成1，这个位正好就是原数组长度值为1的那个位，

而计算桶数组是使用与操作，相当于新的桶数组下标会不会改变取决于hash值在那一位上是否为1，如果hash值在那一位上为0，那么重新计算桶数组下标时，那一位也不会参与运算，和原来是没任何区别的，所以下标肯定是一样的，而为1的话，就参与了运算，那结果自然是多出来了这个位置，而这个位置就是原数组长度，所以最后的下标是原下标+原数组长度

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 扩容逻辑，扩大为原数组的两倍
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {               // zero initial threshold signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    // 创建新数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 开始进行元素的转移
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    // 如果桶数组只有一个元素，则直接重新计算桶下标放到新数组中
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 如果是树节点，则使用树的split方法进行转移
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { // preserve order
                    // 这部分非常关键，具体操作在总结处给出
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}
```

> 由于插入和扩容都是头插，可能在多线程的时候导致链表头尾相连，造成死循环

> 总结：
>
> - 对于jdk1.8的HashMap来说，使用的是数组+链表+红黑树的方式来存储k-v键值对
> - HashMap提供多个构造函数，对于传进来的初始大小，会在构造函数中计算出离这个初始值最近的2的次幂，作为真正的初始值，并放在阈值字段threadhold当中
> - 而初始化操作会在第一次putVal时进行，进行put时如果发现table数组为空并且数组长度等于0，则进行初始化，初始化和扩容都在resize方法中一起完成
> - 在resize方法中，会先判断原数组长度，如果为0，则表示要进行初始化，对于初始化阶段，接下来会判断原threadhold是否为0，如果不为0，则threadhold的值就是初始容量，否则使用默认的初始值16，然后使用这个初始容量来创建数组，然后返回
> - 而对于正常的put操作，先获取对象本身的hashcode，然后对hashcode进行无符号右移16位操作，将高16位移到低16位上，然后和原来的hashcode进行一个异或操作，异或操作是一个无进位加法，这样高16位就和低16位混合在一起，目的是为了使得高16位也参与到hash的计算中。获得hash值后，再和数组长度-1进行与操作，获得桶数组下标，接下来如果这个桶没有元素，就直接将节点放在这个桶，如果这个桶是树节点，就使用树的PutTreeVal方法进行插入，否则就是一个链表的尾插了，具体就是遍历链表，如果中间发现key值一样的节点，那就直接替换，否则到了链表末尾，就放在末尾
> - 插入后如果这个桶的链表长度超过8，就会转换成红黑树，而在插入操作后，如果size操作过了阈值，就进行扩容操作
> - 在扩容操作中，初始化的部分就不提了，会去遍历每个非空桶，如果是树节点，则使用树的split操作进行转移，而如果是链表，就会创建两个链表，然后把桶的每个元素的hash值和原数组长度进行与操作，如果为0，则放在第一个链表中，否则放在第二个链表中。完成遍历后，结果为0的链表会直接放在新数组和原来一样的下标的桶中，而另一个则是放在新数组中原下标+原数组长度的下标的桶中

红黑树的大致实现：

红黑树具有一些特性：它的根节点和叶子节点始终是黑色的，而红色节点的两个子节点都是黑色，也就相当于叶子节点到跟节点之间不会出现两个连续的红色节点，并且含有相同数目的黑色节点，这些性质保证了红黑树在满足平衡二叉树特性的前提下，可以做到从根节点到叶子节点的路径差不会超过两倍，对于插入的新节点都是以红色节点来插入的，插入后如果对树的规则造成了破坏，就通过左旋，右旋，以及变色来向上恢复红黑树



#### 可以使用平衡二叉树来代替红黑树吗

- 因为二叉搜索树在极端情况下会退化成链表，所以出现了二叉平衡树，二叉平衡树要求每个节点的子树高度差不能超过1，而红黑树是是允许超过1的，最多能允许根节点到叶子节点的路径长度的最大差值为二倍
- 二叉平衡树每次删除一个节点或添加一个节点，为了保持规则，需要对整个树进行维护，代价很高，而红黑树在插入和删除某个节点，只需要通过左旋，右旋和变色，维护局部的子树就可以完成恢复，所以代价更小