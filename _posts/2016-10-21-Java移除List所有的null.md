---
layout: post
title: Java移除List所有的null.
uuid: 1ce2422e-9734-11e6-9bf4-005056359bbd
---

 {{ page.title }}
================

<p class="meta">21 Oct 2016 - Su Zhou</p>


## Plain Java

```java
List<Integer> list = Lists.newArrayList(null, 1, null);
while (list.remove(null));
```

or

```java
List<Integer> list = Lists.newArrayList(null, 1, null);
list.removeAll(Collections.singleton(null));
```



## Use Google Guava

```java
List<Integer> list = Lists.newArrayList(null, 1, null);
Iterables.removeIf(list, Predicates.isNull());
```

Alternatively, if we don’t want to modify the source list, Guava will allow us to create a new, filter list

```java
List<Integer> list = Lists.newArrayList(null, 1, null, 2, 3);
List<Integer> listWithoutNulls = Lists.newArrayList(
Iterables.filter(list, Predicates.notNull()));
```



## Use Apache Commons Collections

```java
List<Integer> list = Lists.newArrayList(null, 1, 2, null, 3, null);
CollectionUtils.filter(list, PredicateUtils.notNullPredicate());
```



## Da Da Da Da~  Java 8

```java
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

```java
List<Integer> listWithoutNulls = Lists.newArrayList(null, 1, 2, null, 3, null);
listWithoutNulls.removeIf(p -> p == null);
```


Look! The last Java8 solution is clearest.
