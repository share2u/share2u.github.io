---
title: fastjson
date: 2018-09-29 11:45:18
categories:
- json
tags:
- fastjson
- json
---

json基本格式，以及fastjson基本用法，包括序列化与反序列化

<!--more-->

## JSON 是什么

是一种文本方式展示结构化数据的方式

#### 结构和类型

对象用{}，内部是“key”:"value"

数组用[]表示，不同值用逗号分开

#### 优缺点

优点：平台无关性

缺点：性能一般，缺少schema

## API 

### 序列化

#### 基本序列化

```java
User u = new User();
JSON.toJSONString(u);
```

#### 使用WriteClassName特性

序列化的时候写入类型信息，反序列化的时候，可以根据类型自动识别

```java
Color color = Color.RED;

String text = JSON.toJSONString(color, SerializerFeature.WriteClassName);

System.out.println(text);

{"@type":"java.awt.Color","r":255,"g":0,"b":0,"alpha":255}

Color color = (Color) JSON.parse(text);
```

#### 循环引用

```java
A a = new A();
B b = new B();
a.setB(b);
b.setA(a);
System.out.println(JSON.toJSONString(a));
//{"b":{"a":{"$ref":".."}}}  使用toJSON会栈溢出
A a1 = JSON.parseObject(JSON.toJSONString(a), A.class);
System.out.println(a1 == a1.getB().getA());
//true
```

#### 使用@JSONField Annotation

```java
@JSONField(name ="hah")
private String name;

A a = new A();
a.setName("xxx");
System.out.println(JSON.toJSONString(a));
//{"hah":"xxx"}
```



### 反序列化

#### 指定CLass信息反序列化

```java
JSON.parseObject(s，User.class);
```

#### 类型集合的反序列化

```java
List<User> users = JSON.parseArray(s,User.class);
```

#### 泛型集合的反序列化

```java
Map<String,User> userMap = JSON.parseObject(s,new TypeReference<Map<String,User>>(){});
```



## 参考文献

http://kimmking.github.io/2017/06/06/json-best-practice/

https://blog.csdn.net/qq_35873847/article/details/78850528

http://zyjustin9.iteye.com/blog/2020533