---
title: listener学习笔记
date: 2018-07-30 15:31:16
author: 大东
categories:
- web
tags:
- listener
---



<!-- more -->

## 定义

监听器listener就是在application、session\request三个对象创建、销毁或者往其中添加修改删除属性时自动执行代码的功能组件

## 分类  

|                                 | 类型              | 接口方法                                                     | 接收事件                     |
| :------------------------------ | ----------------- | ------------------------------------------------------------ | ---------------------------- |
| ServletContextListener 接口     | Servlet上下文事件 | contextInitialized() 与 contextDestroyed()                   | ServletContextEvent          |
| ServletContextAttributeListene  | Servlet上下文事件 | attributeAdded() 、 attributeReplaced() 、 attributeRemoved() | ServletContextAttributeEvent |
| HttpSessionListener             | 会话事件          | sessionCreated() 与 sessionDestroyed ()                      | HttpSessionEvent             |
| HttpSessionAttributeListener    | 会话事件          | attributeAdded() 、 attributeReplaced() 、 attributeRemoved() | HttpSessionBindingEvent      |
| HttpSessionActivationListener   | 会话事件          | sessionDidActivate() 与 sessionWillPassivate()               | HttpSessionEvent             |
| HttpSessionBindingListener      | 会话事件          | valueBound() 与   valueUnbound()                             |                              |
| ServletRequestListener          | 请求事件          | requestInitialized() 与 requestDestroyed()                   | RequestEvent                 |
| ServletRequestAttributeListener | 请求事件          | attributeAdded() 、 attributeReplaced() 、 attributeRemoved() | ServletRequestAttributeEvent |

## 举例

在web.xml中添加

```java
< listener >  
< listener -class > com.servlet .listener .YouAchieveListener  < /listener -class >
< /listener >
```

### 在线人数

思路：配置listener实现httpsessionListener,sessionCreated的时候执行

### spring启动

```java
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:spring/applicationContext-*.xml</param-value><!-- 采用的是通配符方式，查找WEB-INF/spring目录下xml文件。如有多个xml文件，以“,”分隔。 -->
</context-param>

<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

​	ContextLoaderListener的作用就是启动Web容器时，自动装配ApplicationContext的配置信息。因为它实现了ServletContextListener这个接口，在web.xml配置这个监听器，启动容器时，就会默认执行它实现的方法。

　　ContextLoaderListener如何查找ApplicationContext.xml的配置位置以及配置多个xml：如果在web.xml中不写任何参数配置信息，默认的路径是"/WEB-INF/applicationContext.xml"，在WEB-INF目录下创建的xml文件的名称必须是applicationContext.xml（在MyEclipse中把xml文件放置在src目录下）。如果是要自定义文件名可以在web.xml里加入contextConfigLocation这个context参数。

## 原理

观察者模式的实现，在web。xml中配置listener的时候就是把一个被观察者放入到观察者对象列表中，当被观察者触发了注册事件时观察者作出相应的反应。在容器container中注册呼叫特定的实现类

## 参考文献

https://my.oschina.net/ydsakyclguozi/blog/398403

https://www.cnblogs.com/hellojava/archive/2012/12/26/2833840.html

http://even2012.iteye.com/blog/1963467







