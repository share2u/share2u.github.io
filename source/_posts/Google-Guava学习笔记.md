---
title: Google Guava学习笔记
date: 2018-08-05 09:05:02
categories:
 - java
tags:
 - guava
---

1. 简介

2. 基本工具

3. 集合Collections

4. 缓存Caches

5. 函数式风格

6. 并发Concurrency

7. 字符串处理Strings

8. 原生类型

9. 区间Ranges

10. I/O

11. hash

12. 事件总线EventBus

13. 数学运算Math

14. 反射Reflections

    <!-- more-->

## 简介

### 源码包的简单说明

- com.google.common.annotations：普通注解类型。  　　
- com.google.common.base：基本工具类库和接口。  
  - 使用和避免使用null
  - 前置条件：快速失败
  - 常见的对象方法：简化Object常用方法的实现
  - 排序：fluent comparator比较器，提供多关键字排序
  - Throwable类：简化异常检查和错误传播　　
- com.google.common.cache：缓存工具包，非常简单易用且功能强大的JVM内缓存
- com.google.common.collect：带泛型的集合接口扩展和实现，以及工具类
  - Immutable collections(不变的集合)：不可修改的集合
  - 新集合类型：multisets，multimaps，tables， bidirectional maps等 
  - 集合工具类
  - 拓展工具类：给集合对象添加功能
- com.google.common.eventbus：发布订阅风格的事件总线。  　　
- com.google.common.hash： 哈希工具包。  　　
- com.google.common.io：I/O工具包。  　　
- com.google.common.math：原始算术类型和超大数的运算工具包。  　　
- com.google.common.net：网络工具包。  　　
- com.google.common.primitives：八种原始类型和无符号类型的静态工具包。  　　
- com.google.common.reflect：反射工具包。  　　
- com.google.common.util.concurrent：多线程工具包。 

## 基本工具

### Optional 类：使用和避免使用null

参见java8 Optional

###  前置条件：快速失败



###  常见的对象方法：简化Object常用方法的实现

### 排序：fluent comparator比较器，提供多关键字排序

### Throwable类：简化异常检查和错误传播

## 集合

### Immutable collections(不变的集合)

### 新集合类型： 

### 集合工具类

### 拓展工具类：给集合对象添加功能

## 缓存

### 适用性

guava cache与concurrentmap类似，区别在于不会一直保存所有添加的元素，通常都设定为自动回收元素

适用于：

- 消耗一些内存空间来提升速度

- 某些健会被查询不止一次

  cache实例通过chachebuilder生成器模式获取，但是**自定义缓存**才是最有趣的地方

### 加载

获取的值如何与健关联

- 使用cacheLoader（首选，更容易推断所有缓存内容的一致性）
- 调用get时传入一个callable实例：“get-if absent-compute”的原子语义
- cache.put方法

#### CacheLoader

```java
LoadingCache<String,String> cache = CacheBuilder.newBuilder()
    .maximumSize(100)
    .build(new CacheLoader<String, String>() {
        @Override
        public String load(String s) throws Exception {
            if("1".equals(s)){
                return "1";
            }else{
                return null;
            }

        }
    });
@Test
public void g1(){
    try {
        String s = cache.get("1");
        System.out.println(s);
        String s1 = cache.get("2");//返回null 报异常  CacheLoader returned null for key 2.
        System.out.println(s);
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
```

#### Callable

```java
Cache<String,String> cache = CacheBuilder.newBuilder()
    .maximumSize(100)
    .build();
@Test
public void g1(){
    try {
        String s = cache.get("1", new Callable<String>() {//每次get都有这个计算过程，调用少的话可硬用，但是call返回null依然是有问题的
            @Override
            public String call() throws Exception {
                if("1".equals("1")){
                    return "1";
                }else{
                    return null;
                }
            }
        });
        System.out.println(s);
    } catch (ExecutionException e) {
        e.printStackTrace();
    }
}
```

#### 显式插入

[cache.put(key, value)](http://docs.guava-libraries.googlecode.com/git/javadoc/com/google/common/cache/Cache.html#put%28K,%20V%29) 可以直接向缓存中插入值，直接覆盖掉给定健的值

Cache.asMap()视图提供的任何方法也能修改缓存 ，但任何方法都不能原子执行

使用之前要注意使用putIfAbsent(K，V),等

### 缓存回收

基于容量回收、定时回收、和基于引用回收

#### 基于容量回收

maximumSize(long),

- 超过上限尝试回收最近没有使用或者总体上很少使用的缓存项
- 不同的缓存项有不同的权重，如果你的缓存值，占据完全不同的内存空间，可以使用weigher(weigher)指定一个权重函数（计算key的权重），并且用maxumumnweight(long)指定最大总重

#### 定时回收

1.  expireAfterAccess(long,timeUnit):缓存项在给定时间内灭有被读写访问则回收，缓存的回收顺序和基于大小回收一样
2. expiredAfterwrite(long,timeunit):缓存项在给定时间内没有被写访问，则回收，如果任务缓存数据总是在固定时间后变得陈旧，这种方式是可取的

#### 基于引用回收

通过使用弱引用的健、或弱引用的值、或软引用的值

#### 显示清除

任何时候都可以显示清除

### 监听器

[`CacheBuilder.removalListener(RemovalListener)`](http://docs.guava-libraries.googlecode.com/git-history/release/javadoc/com/google/common/cache/CacheBuilder.html#removalListener%28com.google.common.cache.RemovalListener%29) ：声明一个监听器缓存项被移除时做一些额外的动作

有异步方法

### 清理

不会自动执行清理和回收工作、会在写或者读的时候做一些维护工作，原因在于维护线程要新建，减少与用户操作竞争共享锁

可以自己创建维护线程，以固定的时间调用chche.cleanup（）

### 刷新

chche.refresh（K）为健加载新值,读操作返回原值，与回收不同，回收会等待新值加载完成

### 其他特性

统计缓存命中率，加载新增的平均时间，缓存被回收的总数等

## 函数式风格

## 并发

### [ListenableFuture](https://github.com/google/guava/wiki/ListenableFutureExplained) 

### Service

## 字符串处理

## 原生类型

## 区间

## I/O

## hash

## 事件总线

## 数学运算

## 反射

## 参考文献

1. 官网 https://github.com/google/guava/wiki
2. 并发编程网 http://ifeve.com/google-guava/
3. http://www.cnblogs.com/peida/p/Guava.htm