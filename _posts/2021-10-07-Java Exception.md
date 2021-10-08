---
layout: post
title: Java Exception
uuid: 78e9cc40-2771-11ec-9808-5f175bb08b40
---

{{ page.title }}
================

# Item 69: Use exceptions only for exceptional conditions
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
异常内部，什么都不做肯定是有问题的，最少应该打印一条日志
 - 代码语义诡异
 - 实现不可靠，因为没有Jdk开发者保证下一个版本这行代码的行为是一致的
 - try catch 块内的代码会有额外的时间开销
 - exception 是用于异常情况的处理， jvm实现者也无法为其专门做优化； 

但是如果你写成下面这样
```java
for (Mountain m : range)
       m.climb();
```
比较明显，根本就不存在越界问题；其次，代码的语义明确；再者，因pattern固定，jvm实现者可为这种pattern做专门优化；故，此为佳选。

PS： 上次的读书会中，我提到使用  xxx.forEach()  的性能会比  for-each 差一些，这个和版本有关系，虽然在Jdk8下是事实，但却不那么重要
 - xxx.forEach() 在目前jdk8 下仅仅比 for-each 稍慢一点点而已，慢 1/4 不到 
 - 当你在 xxx.forEach() 的闭包中操作的都是 immutable 对象，当然首推使用 xxx.forEach()，不仅直观而简洁，而且因强约束了 immutable 对象，反而更安全； 反之，使用 for-each 就行
 - 传统的 fori 性能是相对最慢的，当然 fori 的好处是最灵活。
这完全可以自行 coding 去测试 （最新的 jdk8， 在最新的 jdk 中如何尚未测过）


## case 2 
good
```java
for (Iterator<Foo> i = collection.iterator(); i.hasNext(); ) {
    Foo foo = i.next();
    //...
}
```

bad (// Do not use this hideous code for iteration over a collection!)
```java
try {
    Iterator<Foo> i = collection.iterator();
    while(true) {
        Foo foo = i.next();
        ...
    }
} catch (NoSuchElementException e) {
}
```

# Item 70: Use checked exceptions for recover able conditions and runtime exceptions for programming errors
Checked exceptions 主要用在可以预期的，并且设计目的是调用者必须要处理的，调用者有义务应该知道怎么处理此种异常的情况下。
Runtime exceptions 主要用来指出编程错误

![Java Exception Design](/images/20211007_java_exceptoin/java_exception_tree_20211008110012.jpg)

# Item 71: Avoid unnecessary use of checked exceptions
## How to use Checked exceptions 
强迫程序员处理异常，可以提高代码的可读性和安全性。  
但是使用 Checked exceptions 也有合理性的要求
 - 如果使用者按照正确方式使用 API，但却无法阻止某种已知的Exception 产生
 - 在这种已知的Exception 产生后，使用此API 的程序员可以立即采取有用的动作，处理这种异常
那么这种情况下使用 Checked exceptions 就是非常合理的。
## Avoid Checked exeptions
除了上述的情况外，其他情况下推荐的还是 unchecked exception 
在 java stream api 中，解决 checked exception 的最简单的方法就是返回 result 的一个 Optional；但是返回0长度的Optional却不会带有额外的info，而使用 exception 却会带有导致这个问题发生的原因信息。

所以我们可以使用进一步，把 throw exception 的 method 分成 2 个 method， 比如 actionPermitted() , action()
 - actionPermitted() 返回一个 boolean，表示是否该抛出异常
 - action() 就是正常流程代码
```java
// Invocation with state-testing method and unchecked exception
   if (obj.actionPermitted(args)) {
       obj.action(args);
   } else {
       ... // Handle exceptional condition
}
```

## Item 72: Favor the use of standard exceptions
专家级程序员与缺乏经验的程序员的区别，专家追求高度代码重用（通常也有能力自己做到这种水平）。 这条通用规则同样适用于异常处理。
关注：
- IllegalArgumentException 调用参数值不合适，该见面了
- IllegalStateException 被传入和使用的对象非法时（就是state 不正常时），该见面了
- NullPointerException 在不该出现null的地方出现了null，该见面了
- IndexOutOfBoundsException 下标越界，该见面了
- ConcurrentModificationException 应该单线程使用或同步排队调用的对象，在被并发修改，该见面了
- UnsupportedOperationException 对象不支持所请求的操作，该见面了
  
不要直接重用 Exception, RuntimeException, Throwable, or Error
太一般化的类型，无法提供足够的可预期的信息

## Item 73: Throw exceptions appropriate to the abstraction
更高层的实现应该捕获下层的异常，同时抛出可以按照高层抽象后进行解释的异常。
比如 SpringMVC 中我们返回错误码的 一种方案，就是按照抽象后定义的高层 异常，转化成错误码，返回。

使用 exception chaining 有助于在高层的exception下，可以获取下层exception 的信息；
 - 有助于调试
 - 有助于日志记录，帮助定位错误

## Item 74: Document all exceptions thrown by each method
这个是一般写代码的时候常常被忽视的，但是可以查看 框架代码的文档，依葫芦画瓢就对了，
这点不是技术问题，而是要求人自我“反懒惰”的去做事，其实比较难坚持，但对于后续维护和别人使用你的方法等等，都大有裨益，体现在
 - 开始就知道会出什么问题，可以提早设计防范策略

## Item 75: Include failure-capture information in detail messages
包含对debug 和错误定位有用的所有非敏感信息，比如参数等等的，但是不要多，恰到好处就行，比如你一般不需要把完整的流水信息全部打印出来。

## Item 76: Strive for failure atomicity
这里的 failure atomicity 指的是， 失败的方法调用应该使对象保持在被调用之前的状态
 - immutable 对象
 - 可变对象，则在执行其他操作之前，进行 valid 操作如参数检验云云
 - 调整代码计算顺序， 让可能发生问题的操作排在其他状态修改操作之前

## Item 77: Don’t ignore exceptions
这个不赘述，之前提到过