---
layout: post
title: Spring Graceful Shutdown
uuid: c426188a-37f7-11ec-8f9d-6b587e584ce1
---

{{ page.title }}
================

优雅停机是指在停止应用时，执行的一系列保证应用正常关闭的操作。这些操作往往包括等待已有请求执行完成、关闭线程、关闭连接和释放资源等，优雅停机可以避免非正常关闭程序可能造成数据异常或丢失，应用异常等问题。优雅停机本质上是JVM即将关闭前执行的一些额外的处理代码。
  
一般来说，优雅停机主要处理  
* 池化资源的释放：数据库连接池，HTTP 连接池
* 在处理线程的释放：已经被连接的HTTP请求
* 隐形受影响的资源的处理：Akka 的 Actor 的关闭等

# JVM 中的实现
编程语言都会提供监听当前线程终结的函数，比如在Java中，我们可以通过如下操作监听我们的退出事件：
```java
public class Main {

    public static void main(String[] args) {

        Runtime.getRuntime().addShutdownHook(new ExitHook());

        System.out.println("Do something, will exit");
    }
}

class ExitHook extends Thread{
    
    public void run() {
        System.out.println("exiting. clear resources...");
    }
}
```
我们会得到如下的结果：
```
Do something, will exit
exiting. clear resources...

```


那么实际情况下我们如何实现优雅停机呢，以Linux 为例，我们告诉 OS 给程序对应的 pid 发送一个 kill -15 消息
```bash
# like this,  non-violence process shutdown
kill -15 pid
```

但是不能够使用 kill -9，因为这个告诉操作系统直接杀死 pid 对应的 process，好比强行断电，就没有任何进行优雅停机的机会了。 
```bash
# only use this after graceful shutdown timeout
kill -9 pid
```

# Spring 的 Graceful shutdown Pattern
Spring 是一个 IOC 容器，他能够管理的是收到其托管的对象，因此需要定义托管对象的 dispose 函数才能够被 Spring 在退出时释放  
那么，在 Spring 中 DisposableBean 接口就是用来表示这个类是支持 graceful dispose 的，而对应的我们就应该实现这个接口定义的 destory() 方法--用来释放当前 bean 占用的资源，包括线程池、IO、堆外内存（如果你用了一些unsafe对象方法）等。

* Spring 需要知道 Runtime 在退出
* Spring 知道需要释放哪些资源
* Spring 需要知道如何释放资源

因为 Spring 版本繁杂，以 org.springframework.boot:2.1.xx.RELEASE 版本分析为例。

## Spring 的 Runtime Hook
Spring 的 graceful shutdown 同样依赖于 JVM 的 Shuthook，可见 org.springframework.context.support.AbstractApplicationContext#registerShutdownHook 
```java
@Override
public void registerShutdownHook() {
    if (this.shutdownHook == null) {
        // No shutdown hook registered yet.
        this.shutdownHook = new Thread() {
            @Override
            public void run() {
                synchronized (startupShutdownMonitor) {
                    doClose();
                }
            }
        };
        Runtime.getRuntime().addShutdownHook(this.shutdownHook);
    }
}
```

## Spring 知道需要释放哪些资源
Spring Bean Lifecycle
![Spring Bean Lifecycle](/images/20211028_spring_graceful_shutdown/spring_bean_lifecycle.png)

可以看到，当 Spring Context 销毁的时候，会调用 destroy() 函数
```java
// org.springframework.context.support.AbstractApplicationContext#destroy
@Deprecated //Spring 5 即将废弃
public void destroy() {
    close();
}

@Override
public void close() {
    synchronized (this.startupShutdownMonitor) {
        doClose();
        // If we registered a JVM shutdown hook, we don't need it anymore now:
        // We've already explicitly closed the context.
        if (this.shutdownHook != null) {
            try {
                Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
            }
            catch (IllegalStateException ex) {
                // ignore - VM is already shutting down
            }
        }
    }
}
```
... tobe continue