---
title: JVM优化总结
date: 2022-05-29 17:00:48
tags: ['Java', 'JVM']
---

# JVM优化总结

## 前置知识-元空间的知识

#### 元空间内存分配规则

Metaspace 空间被分配在本地内存中(非堆上)，默认不限制内存使用，可以使用 MaxMetaspaceSize 指定最大值。MetaspaceSize 指定最小值，默认 `21 `M，在元空间申请的内存会分成一个一个的 Matachunk，以Matachunk为单位分配给类加载器，每个 Metachunk 对应唯一一个类加载器，一个类加载器可以有多个 Metachunk![nq93xzhfzb](jvm优化总结/nq93xzhfzb.png)

- **used**: chunk 中已经使用的 block 内存，这些 block 中都加载了类的数据。
- **capacity**：在使用的 chunk 内存。
- **commited**：所有分配的 chunk 内存，这里包含空闲可以再次被利用的。
- **reserved**：是可以使用的内存大小

#### 元空间出发Full GC时机

参数 `MinMetaspaceFreeRatio`，默认40，也就是40%，

## 案例

一个关于JVM 元空间mataspace OOM的优化实例，服务频繁出现FULL GC

解决过程

- 打开了GC日志，然后发现GC日志当中

  元空间使用信息中，**capacity比use多出来很多，说明分配了很多chunk，占用了很多空间，但是却没有完全使用它**

  所以推断出mataspace中应该是碎片化了

  因为chunk分配是根据类加载器来分配的，所以只用那三个类加载器一般是不会出现碎片化的，所以**初步定位为有大量的自定义类加载器加载了一堆类进来**

- 所以dump jvm内存下来，然后观察Objects使用较高的元素，排除Java内部的那些常见类之后，发现一个叫做`DelegatingClassLoader`数量特别高，然后再分析是谁引用了这个类加载器

-  然后发现是因为反射调用method的方法引用的，所以呢就初步判定了是反射造成的

- 最后发现，原来是Method类中，invoke方法里面会有一个优化，指的是有一个膨胀阈值，为15，当发现一个类中的方法被调用超过阈值15次时，会做一个优化，会使用字节码的形式，为这个方法去定义出一个类，这个类的类加载器叫做`DelegatingClassLoader`，所以创建了大量的类加载器

- 最后在定位到代码上，发现我们需要传递上下文对象到一个排序rank引擎中给召回item做一次精排打分，需要把上下文对象做一次转换，所以大量使用了BeanUtils.copy...的方法，找个方法里面就是通过invoke对象的getSet方法来做属性赋值的，所以问题就定位到了

  

