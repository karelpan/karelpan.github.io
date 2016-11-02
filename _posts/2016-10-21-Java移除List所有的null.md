---
layout: post
title: Java移除List所有的null.
uuid: 1ce2422e-9734-11e6-9bf4-005056359bbd
---

 {{ page.title }}
================

<p class="meta">21 Oct 2016 - Su Zhou</p>


## 1. Plain Java
能够使用Java基本库优雅解决的，就不要引入更多的依赖

```java
List<Integer> list = Lists.newArrayList(null, 1, null);
while (list.remove(null));
```

or

```java
List<Integer> list = Lists.newArrayList(null, 1, null);
list.removeAll(Collections.singleton(null));
```

第二种比起第一种效率要高， 建议自己 benchmark 看结果，然后看源码，明白温和 第二种性能高 


## 2. 使用 Google Guava
Guava 是个优秀的库， 他也提供了非常明了的操作API

```java
List<Integer> list = Lists.newArrayList(null, 1, null);
Iterables.removeIf(list, Predicates.isNull());
```

Alternatively, if we don’t want to modify the source list, Guava will allow us to create a new, filter list
不想影响原本的 List， 那就新建一个吧

```java
List<Integer> list = Lists.newArrayList(null, 1, null, 2, 3);
List<Integer> listWithoutNulls = Lists.newArrayList(
Iterables.filter(list, Predicates.notNull()));
```



## 3. 使用 Apache Commons Collections
除了Guava ，Apache 的Commons库也是经常使用的（自己看库，会发现Guava的代码质量和优雅程度，明显优于Apache Commons 库，建议直接看代码，有很直观的了解，建立新Java 代码观 ）

```java
List<Integer> list = Lists.newArrayList(null, 1, 2, null, 3, null);
CollectionUtils.filter(list, PredicateUtils.notNullPredicate());
```



## Da Da Da Da~  Java 8
恩，前面的其实都不能算有美感的解决方案; 但是Java8给我们带来了新的特性 Lamada + 闭包

```java
// 使用 Filter 可以优雅的， 准确的解决问题， 而且配合 Lamada， 其意自现
List<Integer> list = Lists.newArrayList(null, 1, 2, null, 3, null);
List<Integer> listWithoutNulls = 
list.parallelStream().filter(i -> i != null).collect(Collectors.toList());
```

or

```java
List<Integer> list = Lists.newArrayList(null, 1, 2, null, 3, null);
List<Integer> listWithoutNulls = 
list.stream().filter(i -> i != null).collect(Collectors.toList());
```

or

最优雅解决方案

```java
// 相信这个才是最优雅的， 建议多读Java8的源码，特别是 Stream API（Stream API 构筑于 NIO2 之上）
List<Integer> listWithoutNulls = Lists.newArrayList(null, 1, 2, null, 3, null);
listWithoutNulls.removeIf(p -> p == null);
```
