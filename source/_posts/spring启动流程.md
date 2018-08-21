---
title: spring启动流程
date: 2018-08-21 20:39:45
categories:
- spring
tags:
- spring
---

## Web.xml

web容器为spring提供了宿主环境Servletcontext,启动时读取web.xml/包括ContextLoaderListener启动spring容器、DispathcherServlet springmvc分发器，

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_3_1.xsd"
         version="3.1">
    <!--web项目中上下文初始化参数, name value的形式 -->
    <context-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/applicationContext.xml</param-value>
    </context-param>

    <!--ContextLoaderListener,会通过它的监听启动spring容器-->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>

    <!--DispatherServlet,前端MVC核心，分发器，SpringMVC的核心-->
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <load-on-startup>1</load-on-startup>
    </servlet>
</web-app>
```

web容器的初始化过程：

1. web容器（如tomcat）读取web.xml, 读取文件中两个节点和
2. 容器创建ServletContext，它是web的上下文，整个web项目都会用到它
3. 读取context-param节点，它以 键值对的形式出现。将节点值转化为键值对，传给ServletContext
4. 容器创建中的实例，创建监听器。监听器必须继承ServletContextListener
5. 调用ServletContextListener的contextInitialized()方法，spring容器的创建和初始化就是在这个方法中

##  initWebApplicationContext 

org.springframework.web.context.ContextLoader#initWebApplicationContext

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
//判断是否已经有webApplicationContext容器，如果有抛异常	
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
		throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		log.....
		long startTime = System.currentTimeMillis();
		try {
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
            //创建WebApplicationContext
			if (this.context == null) {
				this.context = createWebApplicationContext(servletContext);
			}
  // 判断context有没有父context，取决于web.xml配置文件中locatorFactorySelector参数，如果有父context，则加载它
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
                    // refresh容器，这一步会创建beans
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
          // 将spring容器context，挂载到servletContext这个web容器全局变量中。ServletContext是web容器的上下文
          //web容器指的是Tomcat等部署web应用的容器，不要和spring容器搞混了  
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);
            // 将spring容器context赋值给currentContext变量，保存下来
			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}
			log.....
			return this.context;
		}
		exception ....
	}
```

initWebApplicationContext()主要做三件事

1. 创建WebApplicationContext，通过createWebApplicationContext()方法
2. 加载spring配置文件，并创建beans。通过configureAndRefreshWebApplicationContext()方法
3. 将spring容器context挂载到ServletContext 这个web容器上下文中。通过servletContext.setAttribute()方法。

###  createWebApplicationContext 创建spring容器

###  configureAndRefreshWebApplicationContext 加载spring配置文件，创建beans











## 参考文献

https://blog.csdn.net/u013510838/article/details/75066884

https://blog.csdn.net/moshenglv/article/details/53517343