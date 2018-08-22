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

初始化spring容器，使用默认的配置文件，或者context-params中的配置contextConfigLocation

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
  
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				if (!cwac.isActive()) {
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
              // 判断context有没有父context，
               //取决于web.xml配置文件中locatorFactorySelector参数，如果有父context，则加载它
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

初始化root webAppLicationContext，采用默认配置或者自定义的class

```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
   // 获取WebApplicationContext实现类的class对象，WebApplicationContext只是一个接口，需要有具体的实现类，默认的实现类是XmlWebApplicationContext
   Class<?> contextClass = determineContextClass(sc);
  // 自定义WebApplicationContext必须继承自ConfigurableWebApplicationContext
   if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
      throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
            "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
   }
  // 由class对象创建实例对象
   return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

### configureAndRefreshWebApplicationContext 加载spring配置文件，创建beans

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		wac.setId(idParam);
		wac.setServletContext(sc);
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
    //加载属性到enviroment
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}
		//个性化配置
		customizeContext(sc, wac);
		wac.refresh();
	}
```

```java
//读取配置文件、并创建和初始化beans
public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
            //准备工作，设置ApplicationContext中的一些标志位，如closed设为false,activer为true,等
			prepareRefresh();
			// Tell the subclass to refresh the internal bean factory.
            //读取spring xml配置文件，
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// Prepare the bean factory for use in this context.
            //设置容器beanfactory的各种成员属性，比如beanCLassLoader,beanPostProcessors
            //这里的beanPostProcessor都是系统默认的，不是用户自定义的。比如负责注入ApplicationContext引用到各种Aware中的ApplicationContextAwareProcessor容器后处理器。
			prepareBeanFactory(beanFactory);
				// Allows post-processing of the bean factory in context subclasses.
             // 调用默认的容器后处理器，如ServletContextAwareProcessor
				postProcessBeanFactory(beanFactory);
				// Invoke factory processors registered as beans in the context.
            // 初始化并调用所有注册的容器后处理器BeanFactoryPostProcessor
				invokeBeanFactoryPostProcessors(beanFactory);
				// Register bean processors that intercept bean creation.
            	 // 注册bean后处理器，将实现了BeanPostProcessor接口的bean找到。
            //先将实现了PriorityOrdered接口的bean排序并注册到容器BeanFactory中，
            //然后将实现了Ordered接口的排序并注册到容器中，最后注册剩下的。
				registerBeanPostProcessors(beanFactory);
				// Initialize message source for this context.
            // 初始化MessageSource，用来处理国际化。
            //如果有beanName为“messageSource”，则初始化。否则使用默认的。
				initMessageSource();
				// Initialize event multicaster for this context.
            // 初始化ApplicationEventMulticaster，用来进行事件广播。
            //如果有beanName为"applicationEventMulticaster"，则初始化它。否则使用默认的SimpleApplicationEventMulticaster。广播事件会发送给所有监听器，也就是实现了ApplicationListener的类
            //关于spring事件体系，可以参见 http://blog.csdn.net/caihaijiang/article/details/7460888
				initApplicationEventMulticaster();
				// Initialize other special beans in specific context subclasses.
            // 初始化其他特殊的bean。子类可以override这个方法。如WebApplicationContext的themeSource
				onRefresh();
				// Check for listener beans and register them.
             // 注册事件监听器，也就是所有实现了ApplicationListener的类。会将监听器加入到事件广播器ApplicationEventMulticaster中，所以在广播时就可以发送消息给所有监听器了。
				registerListeners();

			 // 初始化所有剩下的singleton bean(没有标注lazy-init的)
				finishBeanFactoryInitialization(beanFactory);

				// 回调LifecycleProcessor，发送ContextRefreshedEvent事件等
				finishRefresh();
			}
		}
	}
```







## 参考文献

https://blog.csdn.net/u013510838/article/details/75066884

https://blog.csdn.net/moshenglv/article/details/53517343