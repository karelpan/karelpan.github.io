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

定位到最终的释放资源的地方是 org.springframework.context.support.AbstractApplicationContext#doClose 函数，下面分析一下。
```java
protected void doClose() {
    LiveBeansView.unregisterApplicationContext(this);

    try {
        // Publish shutdown event.
        publishEvent(new ContextClosedEvent(this)); // 1
    }
    // Stop all Lifecycle beans, to avoid delays during individual destruction.
    if (this.lifecycleProcessor != null) {
        try {
            this.lifecycleProcessor.onClose(); // 2
        }
        catch (Throwable ex) {
            logger.warn("Exception thrown from LifecycleProcessor on context close", ex);
        }
    }

    // Destroy all cached singletons in the context's BeanFactory.
    destroyBeans(); // 3

    // Close the state of this context itself.
    closeBeanFactory(); // 4

    // Let subclasses do some final clean-up if they wish...
    onClose(); // 5

    // Reset local application listeners to pre-refresh state.
    if (this.earlyApplicationListeners != null) {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    // Switch to inactive.
    this.active.set(false);

}
```
cut 无关的代码后，我们可以发现对于 Spring 来说，关闭的顺序是
1. 发布一个关闭事件
2. 调用生命周期处理器
3. 销毁所有的Bean
4. 关闭Bean工厂
5. 调用子类的Close函数

1、2 是回调，不涉及对象销毁；5 是让子类进行额外的回收动作； 容器内的 bean 的移除销毁第 3 步。 

受到 Spring 托管的对象很多，不一定所有的对象都需要销毁行为。从下面的 spring 框架代码可知，所有的需要带有自己销毁 method 的对象都实现了 DisposableBean Interface
```java
public void destroySingleton(String beanName) {
    this.removeSingleton(beanName);
    DisposableBean disposableBean;
    synchronized(this.disposableBeans) {
        disposableBean = (DisposableBean)this.disposableBeans.remove(beanName);
    }

    this.destroyBean(beanName, disposableBean);
}
```
Spring 会在关闭的时候销毁这些实现了 DisposableBean Interface 类型的 Bean对象。


## Spring 需要知道如何释放资源
```java
public interface DisposableBean {
    void destroy() throws Exception;
}
```
实现此方法，Spring会在销毁对象的时候，自动调用


## 组合在一起
Spring Web Server 是如何优雅的停机的
* Web Server内部的资源都是 DisposableBean，并且受 Spring 托管

举个例子，比如数据库的资源 ```org.springframework.orm.jpa.AbstractEntityManagerFactoryBean#destroy``` 在销毁的阶段会将 Entity 对象进行销毁。

对于收到 Spring 托管的对象的优雅停机的路径是：
```
Runtine Shutdown Hook -> Context:destory() -> DisposableBean:destroy()
```
对于大部分的资源比如数据库，服务发现，等等都是这样的销毁方式。


#  Web 容器
## Q1：普通的 bean 的销毁我们已然了解，那么 web server 呢？
Web Server 并不是在 destroySingleton 阶段进行销毁的

除了销毁Beans 之外，还有最后一个 close() 函数可以调用，对于 Tomcat 等这些 web容器来说，本身是作为 ApplicationContext 的一个实现，并非为Spring Bean 一部分（Spring 本身是一个 Servlet，托管于 Tomcat 这样的 Servlet 容器）

对于 Spring 而言，可以调用 ```org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext``` 的 close() 函数来告诉 Tomcat 开始关闭
```java
protected void onClose() {
    super.onClose();
    this.stopAndReleaseWebServer();
}
```

## Q2: 如果一个 web 服务已经触发了 Shutdown 动作后， 此时还可以接受请求吗？
* 在老版本的 Tomcat 比如 8 内是不支持的（在9.0.33+的 tomcat 配合 spring-boot-2-3-0+ 可以提供默认支持）， 我们实际上是无法获得正确的返回的。
```
+------------------+          +---------------------------+         +--------------------------------------+
|   controller     |          |       services            |         |               dao                    |
|                  +--------->+                           +--------->                                      |
|                  |          |                           | 2. Query DB                          Entitymanger
+------------------+          +---------------------------+         +---------------------------------^----+
                                                                                                      |
                                                                                                      |
                                                                                                      |
                                 +--------------------------------------+                             |1. destory()
                                 |                                      |                             |
                                 |    Destory                           +-----------------------------+
                                 |                                      |
                                 +--------------------------------------+
```
但是我们可以通过，添加 静态状态维护类，并使用内存强同步语义的变量来标识，已经进入 Shutdown hook， 通过 controller 的 AOP Before 切面拦截，阻止后续请求并直接 failed

# Q3: 如果仅仅使用 haproxy 这样的均衡器时，想要及时发现已经不服务的系统，如何做？
对于 一个已经进入 shutdown 流程的 Spring 管理的 web 应用。 
1. 首先 haproxy 和对应的 web app ，本身应该有 健康检查这样的机制， 一般为一个仅有head 数据的请求
2. 在 web app 触发了 shutdown 后， 给予 Q2 中的的切面拦截的方法，应让 健康检查的接口直接返回快速失败，那么 haproxy 就知道应该从转发列表中摘除此 node （直到其健康检查接口重新正确返回--指的就是服务重新正确服务）

# Q4: 如果我希望，在触发了 shutdown 操作后， 已经接受的请求优先处理完后，再进行后续的 graceful shutdown，假设在 spring 内部如何做呢？
1. 实现自己 EmbeddedWebApplicationContext ，比如 实现一个  ```GracefulShutdownGenericWebApplicatonContext implements EmbeddedWebApplicationContext```
2. 实现自己的 doClose() 方法，在未处理完任务前，不关闭 web server
3. 在创建 Spring 上下文的时候，使用这个 ```GracefulShutdownGenericWebApplicatonContext.class```
4. 对于有自定义销毁流程的 bean 实现 SmartLifecycle ，比如实现(我随便命名了)一个 ```ActorSystemSmartLifecycle implements SmartLifecycle``` , 通过设置 phase 来设置优先级
5. 在启动 Spring 创建好对应的 SmartLifecycle 的 Bean 


# 在 Docker 容器中 Java程序的 gracelful shutdown 
docker stop 加上一个 timeout 命令默认触发的是 SIGNAL 15（KILL -15）， 待超时后才会触发 SIGNAL 9 (KILL -9) 强制杀死 docker 中 pid 为 1 的进程

那么这个 SIGNAL 会被 docker 内的 vm 讲信号转发给 pid 为 1 的 jvm 进程， 由于 springboot(tomcat + spring)  默认注册了 shutdownHook， 那么就会触发这个回调，开始执行关闭流程 
   

# 最 Pure 的测试方式
```java
java -jar xxxxx-xxxx.jar
```
此时这个程序是前台程序，console 输出中可以看到完整的log，此时 shell 的输入输出也被binding在 jvm 进程所对应的这个标准输入输出， 此时只要触发 SIGNAL 2 只需要 Ctrl + C 即可， 可以通过日志看到关闭流程。


## 会触发 jvm shutdown hook 的 OS 信号
* SIGINT   2       中断
* SIGTERM  15      终止信号
上面两个信号会触发  shutdownHook 


而 SIGKILL  9  kill信号  是强制执行， 此 signal 不能被对应被 kill 的 process 忽略、处理和阻塞