---
title: servlet学习笔记
date: 2018-07-30 15:31:16
author: 大东
categories:
- web
tags:
- servlet
---



<!-- more -->

## servlet容器

常见的Servlet容器是tomcat，jetty等

### tomcat容器模型

![1532928329345](C:\Users\cwm\Documents\tmp\web\tomcat容器模型.png)

tomcat容器分为四个等级，真正管理servlet的容器是Context容器，一个Context对应一个web工程

### tomcat添加一个web项目

```java
  public Context addWebapp(Host host, String url, String path) {
        //silence(url);
        Context ctx = new StandardContext();
        ctx.setPath( url );//设置访问路径
        ctx.setDocBase(path);//设置项目路径
        ctx.setRealm(new NullRealm());//设置权限
        ctx.addLifecycleListener(new Tomcat.DefaultWebXmlListener());
        ContextConfig ctxCfg = new ContextConfig();
        ctx.addLifecycleListener(ctxCfg);
        ctxCfg.setDefaultWebXml("org/apache/catalin/startup/NO_DEFAULT_XML");
        host.addChild(ctx);//添加到host父容器中
        return ctx;
    }
```

### tomcat 启动

#### 容器初始化

#### 应用初始化

1. 主要是解析web.xml文件，解析文件中的属性保存到WebXML对象中，
2. 然后将WebXML对象中的属性设置到Context中，这里包括Servlet，filter,listener等
3. Servlet将被包装成Context容器中的StandardWrapper，其具有容器的特征，转换将使得开发者不需要强耦合tomcat

#### 创建servlet实例

1. 创建Servlet对象

   如果Servlet的load-on-startup配置项大于0，那么context容器启动的时候就会被实例化，例如DefaultServlet和JSPServlet

   调用wrapper.loadServlet方法：获取servletclasss交给instanceManager去创建一个机遇servletclass.class对象

2. 初始化Servlet

   调用servlet的init方法，同时把包装了Standandwrapper对象de standardwrapperfacade作为servletconfig传给Servlet

## Servlet体系结构

![1532954375886](C:\Users\cwm\Documents\tmp\web\Servlet体系结构.png)

## 参考文献

Servlet 工作原理解析

https://www.ibm.com/developerworks/cn/java/j-lo-servlet/index.html