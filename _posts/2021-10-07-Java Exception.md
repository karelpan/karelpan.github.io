---
layout: post
title: Java Exception
uuid: 78e9cc40-2771-11ec-9808-5f175bb08b40
---

{{ page.title }}
================

# Use exceptions only for exceptional conditions
anti-patterns 

## case 1 
```java
// Horrible abuse of exceptions. Don't ever do this!
try {
    int i = 0;
       while(true)
           range[i++].climb();
} catch (ArrayIndexOutOfBoundsException e) {
}
```
像啥都不做问题大
 - 代码语义诡异
 - 实现不可靠，因为没有Jdk开发者保证下一个版本这行代码的行为是一致的
 - try cache 块内的代码会有额外的时间开销
 - exception 是用于异常情况的处理， jvm实现者也无法为其专门做优化； 

但是如果你写成下面这样
```java
for (Mountain m : range)
       m.climb();
```
很明显，根本就不存在越界问题；其次，代码的语义明确；再者，因pattern固定，jvm实现者可以为这种pattern做专门优化

这里额外讲一点，上次的读书会中，我提到 使用  xxx.forEach()  的性能会比  for-each 差一些，这个虽然是事实，
 - 但是 xxx.forEach() 在目前jdk8 下仅仅比 for-each 稍慢一点点而已
 - 当你在 xxx.forEach() 的闭包中操作的都是 immutable 对象，那当然是首推使用 xxx.forEach()，因为直观而简洁，而且因为强约束了 immutable 对象，反而更安全； 反之，使用 for-each 就行
 - 传统的 fori 性能是相对最慢的，当然 fori 的好处是最灵活。
这完全可以自行 coding 去测试 （最新的 jdk8）

## case 2 

