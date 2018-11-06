---
title: servlet-filter-listener-interceptor
date: 2018-10-25 19:08:40
categories:
- web
tags:
- servlet
- filter
- listener
- interceptor
---

## 概念

### servlet

- 是一种运行再服务端的java应用程序，具有独立与平台和协议的特性，并且可以动态的生成web页面，它工作在客户端请求和服务端响应的中间层
- 创建并返回给客户端完整的html页面

### filter

- 过滤器，是一个可以复用的代码片段、可以用来转换http请求、响应和头信息。不像servlet,它不能产生一个请求或响应、只能修改某一资源的请求、或者修改某一资源的响应
- 能够在一个请求到达servlet之前预处理用户请求，也可以在离开servlet时处理http响应
- url传来后，检查之后，可保持原来的流程继续向下执行，被下一个filter处理,servlet接收后，不会向下传递。

### listenter

- 监听器、通过listener可以监听web服务器中的某一个执行动作、并根据其要求做出相应的响应。再application\session\request三个对象创建消亡或者添加修改属性时自动执行代码的功能组件
- servlet、filter时针对url之类的，而listener是针对对象的操作，在这些对象变化的时候做一些操作

### interceptor

- 面向切面编程中，就是再你的servcie或者一个方法前后，比如动态代理就是拦截器的简单实现，只能对controller请求进行拦截
- 类似与框架带的filter
  - 不在web.xml中配置，框架配置文件中
  - 由action自己指定用哪个interceptor来接受处理，并且访问action上下文，过滤器可以对几乎所有的请求起作用
  - 拦截器是基于java 反射的，过滤器是基于函数回调的
  - 拦截器不依赖与serlvet容器，过滤器依赖于servlet容器

## 生命周期

### servlet

它被装入web服务器的内存，并在web服务器终止或者重新装入servlet时结束。servlet一但装入web服务器，一般就不会在web服务器内存中删除、直到web服务器关闭或重新结束。

1. 装入：启动服务器时加载servlet的实例
2. 初始化：web服务器启动时或者接受到请求时，初始化工作init()方法负责执行完成
3. 调用：从第一次到以后的多次访问，都只调用doget（）或dopost方法
4. 销毁：停止服务器时调用destory（）方法，销毁实例

### filter

Filter接口方法有init().doFilter(),dodestory()

1. 启动服务器时加载过滤器实例，并调用init()方法来初始化实例
2. 每一次请求时都只调用方法doFilter()进行处理，调用下一过滤器chain.doFilter
3. 停止服务器时调用destory()方法，销毁实例

### listener



### interceptor

- struts 的拦截器在加载xml后，初始化相应拦截器，当action请求来时调用intercept方法，服务器停止销毁interceptor
- prehandle(action执行之前)，posthandle(方法执行之后，不一定会执行)，aftercompletion（最终执行）

## 顺序

web.xml的加载顺序context-param->listener->filter->servlet

执行顺序：

filter1--------------------------web.xml中的配置顺序

​	->interceptor1(prehandle)--框架中的拦截器

​		->servlet1

​	->interceptor1(posthandle,aftercompletion)

->filter1

![1540519294642](servlet-filter-listener-interceptor\struts结构.png)



![springmvc拦截器](servlet-filter-listener-interceptor\springmvc结构.png)

## 应用举例

### filter

在过滤器中修改字符编码，在过滤器中修改request的参数（链路跟踪）

### interceptor

日志，在方法调用前后打印出字符串

## 参考文献

https://blog.csdn.net/sundenskyqq/article/details/8549932

https://blog.csdn.net/tanga842428/article/details/52175683

https://blog.csdn.net/zxd1435513775/article/details/80556034

https://www.cnblogs.com/jzb-blog/p/6717349.html