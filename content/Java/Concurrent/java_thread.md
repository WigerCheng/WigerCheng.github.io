---
tags:
  - Java/Concurrent
title: Java 线程
draft: true
---
## Thread.start

```java
    public void start() {
        synchronized (this) {
            // zero status corresponds to state "NEW".
            if (holder.threadStatus != 0)
                throw new IllegalThreadStateException();
            start0();
        }
    }
```

>[!warning] 
>
>不能两次启动THread，否则会出现IllegalThreadStateException异常。

## 线程名字

线程有自己的名字。可以通过构造函数new Thread(String name)的时候传入，也可通过`setName`来设置线程名字。缺省名字是`"Thread-XX"`，XX是自增数字。

## 线程的父子关系

在Thread的构造方法里，会调用`Thread parent = currentThread();`。其中`currentThread`就是获取当前的线程，但根据[[java_thread_lifecycle|线程生命周期]]可知，在构造方法的时候，线程是处于NEW状态，所以`currentThread`获取的就是创建这个线程的线程。

由此可知：

1. 一个线程的创建是由另一个线程完成的。
2. 被创建线程的父线程就是创建它的线程。

>[!note]
>
>main函数所在的线程由JVM创建，也就是main线程。

