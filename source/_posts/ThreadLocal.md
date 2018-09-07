---
title: ThreadLocal
date: 2018-09-04 14:11:08
categories:
- 多线程
tags:
- threadlocal
---

## 定义

Threadlocal提供了线程本地的变量，这些变量绑定于自己的线程，独立的进行初始化。使用上通常是private static 变量，并且关联一个有状态的变量，如user ID 或Transaction ID.

通常是保存 线程中的状态的变量

例如：

```java
public class ThreadId {
    // Atomic integer containing the next thread ID to be assigned
    private static final AtomicInteger nextId = new AtomicInteger(0);

    // Thread local variable containing each thread's ID
    private static final ThreadLocal<Integer> threadId =
        new ThreadLocal<Integer>() {
        @Override 
        protected Integer initialValue() {
            return nextId.getAndIncrement();
        }
    };

    // Returns the current thread's unique ID, assigning it if necessary
    public static int get() {
        return threadId.get();
    }
}
```

只要这个线程活着，并且threadllocal实例可用，那么每一个线程保持一个引用与threadlocal中变量的拷贝，

## 使用

## 原理分析

## 注意点

## 参考文献

http://www.jasongj.com/java/threadlocal/#comments

https://segmentfault.com/a/1190000014738647