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

**通常是保存 线程中的状态的变量**

## 使用

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

ThreadLocal()

无参构造器

withInitial(Supplier<? extends S>):ThreadLocal<S>

```java
public static <S> ThreadLocal<S> withInitial(Supplier<? extends S> supplier) {
    return new SuppliedThreadLocal<>(supplier);
}

static final class SuppliedThreadLocal<T> extends ThreadLocal<T> {

    private final Supplier<? extends T> supplier;

    SuppliedThreadLocal(Supplier<? extends T> supplier) {
        this.supplier = Objects.requireNonNull(supplier);
    }

    @Override
    protected T initialValue() {
        return supplier.get();
    }
}
```

get():T

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    //如果没有值，那么就返回初始化值
    return setInitialValue();
}
```

set(T):void

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

remove():void

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

## 原理分析

### ThreadLocalMap 

每个线程都有一个threadlocalmap成员变量

threalLocalMap底层是一个散列表(数组链表)

tail -f /opt/web/dianshangwuxian_order-server/logs/catalina.log.2018-09-12

tail -f /opt/web/dianshangwuxian_order-server/logs/order-server.log

### entry

#### entry定位

## 注意点

## 参考文献

http://www.jasongj.com/java/threadlocal/#comments

https://segmentfault.com/a/1190000014738647