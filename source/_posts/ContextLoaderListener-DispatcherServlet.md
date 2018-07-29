---
title: ContextLoaderListener & DispatcherServlet
date: 2018-07-29 11:31:16
author: 阿润
categories:
- spring framework
tags:
- spring
- servlet
- listener
---

ContextLoaderListener, DispatcherServlet都会加载配置文件, 那么二者之间有什么区别？  

1. 两个配置文件之间的差异，各定义什么bean
2. 两个类的继承体系结构
3. 两个配置文件生成的容器之间的关系

<!-- more -->

## Spring 与 web.xml

spring 提供了强大的IOC能力，我们往往会配置业务层，数据层的bean在容器中；除此之外，spring 特提供了spring mvc，控制前段发送的请求，在web.xml中往往会有以下的代码片段。

```xml
<listener>
     <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
           
<context-param>
     <param-name>contextConfigLocation</param-name>
     <param-value>classpath:*-context.xml</param-value>
</context-param>

<servlet>
    <servlet-name>platform-services</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:platform-services-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>

<servlet-mapping>...
```

- ContextLoadListener ContextLoaderListener加载<context-param>设置的配置文件, 联系ServletContextListener监听器
- DispatcherServlet DispatcherServlet记载<init-param>设置的配置文件, 联系ServletConfig

## ContextLoaderListener

​	继承体系结构 ContextLoaderListener继承ContextLoader, 实现了ServletContextListener接口. 因此在Web容器启动的时候, 其会监听到ServletContext创建事件, 进而进行初始化操作, 其加载配置文件的本质工作是通过ContextLoader来实现的。

​	initWebApplicationContext()根据<context-param>指定的xml文件初始化root WebAppicationContext. 在没有设置<context-class>的,即自定有 上下文初始化类的情况下, 默认采用XmlWebApplicationContex来初始化Spring容器. 此容器是root容器, 里面一般包含应用程序可以共享的bean, 例如 业务层, 数据层, 工具层等等, 往往不会涉及到Web相关的组件, 如视图解析器, 控制器..... root容器被设置在ServletContext中 

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}

	/**
	 * Initialize the root web application context.
	 */
	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}    
}
```

## DispatcherServlet

​	集成体系结构 从集成体系结构可以看出, DispatcherServlet本质上市基于Servlet, 因此也就包含Servlet的生命周期方法, 例如, init(), getServletConfig(), service()

1. HttpServletBean读取web.xml中的DispatcherServlet的配置信息, <init-param>; 同时留给子类一些接口实现其他的功能
2. FramewordServlet则是初始化web application context, 每一个DIspatcherServlet都有一个与之对应的web application context. 其父类是ContextLoader初始化的 Spring上下文(root web application context)
3. DispatcherServlet则用户初始化Spring MVC框架其他的组件, 如HandlerMapping....

```yaml
 HttpServlet
        HttpServletBean
            FramewordServlet
                DispactherServlet
```

## 两者初始化的上下文之间的关系

​	上下文的关系如下 父子关系, Spring允许用户建立容器之间的多级关系, 子容器(DispatcherServlet初始化的容器)在本容器中找不到Bean时, 将会去父容器中查找. 父容器中的bean将 在所有子容器之间共享。

## 常见问题

- 同一个Bean初始化多遍

1. 将root web application context与web application context的配置信息放在一起
2. 一个bean不仅使用注解, 也使用xml配置以便
3. 在拦截器或者监听器中使用XmlWebApplicationContext加载配置文件....