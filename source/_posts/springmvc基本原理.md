---
title: springmvc基本原理
date: 2018-08-21 12:37:28
categories:
- spring
tags:
- springmvc
---

Spring 面试题

Spring MVC基本原理

总结

<!-- more-->

## Spring 面试题

1. 什么是Spring框架，主要模块
2. 好处
3. IOC,DI
4. BeanFactory和ApplicationContext有什么区别
5. Spring Bean的生命周期
6. Spring框架中的单例bean是线程安全的吗

## Spring MVC基本原理

### 配置阶段

#### web.xml

#### DispatcherServlet:

Springweb开发的入口

#### application.xml

配置spring启动所需要的加载的bean

#### url-pattern

拦截的地址

### 初始化阶段

#### servlet的init方法

由web容器自动调用servlet的init方法，在init方法中，执行初始化操作

#### 加载配置文件

加载application.xml，扫包，将className收集起来

#### 初始化IOC容器

就是一个map<String,Object>

- IOC容器规则

1. key默认都是类名首字母小写
2. 如果用户自定义名字，那么要优先设为该名字
3. 如果是接口，使用接口类型作为key,vlaue为实现类

#### 依赖注入

@Autowried

遍历IOC容器中class的属性，是否有注解，将其与容器中的关联

#### 初始化handlermapping

- map<String,handler>,存储requestmapping配置的url等
- list<Handler>中存储映射关系，包括正则url,参数，

将ioc中被controller注解的class中的url作为baseUrl，其中的方法为url后缀

### 远行阶段

#### servlet.service(Request,Response)

线程阻塞，用户请求，封装参数，反射自动调用方法

#### request.getURL()

获得用户请求的URL

#### 匹配URL和对应的Method

#### 调用method

封装参数，反射动态调用method

#### 利用Response将调用结果输出到浏览器
