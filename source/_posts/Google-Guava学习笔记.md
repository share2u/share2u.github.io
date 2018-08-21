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