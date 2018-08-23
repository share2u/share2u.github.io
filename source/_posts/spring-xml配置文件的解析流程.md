---
title: spring-xml配置文件的解析流程
date: 2018-08-21 20:42:15
categories:
 - spring
tags:
 - spring
---

加载xml配置文件,将bean注册到Map中

<!-- more-->

## obtainFreshBeanFactory

org.springframework.context.support.AbstractApplicationContext#obtainFreshBeanFactory

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}
```

refreshBeanFactory

org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory

```java
protected final void refreshBeanFactory() {
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    beanFactory.setSerializationId(getId());
    customizeBeanFactory(beanFactory);
    //加载xml配置文件，具体子ApplicationContext
    loadBeanDefinitions(beanFactory);
    synchronized (this.beanFactoryMonitor) {
        this.beanFactory = beanFactory;
    }
}
```

### loadBeanDefinitions

#### XmlWebApplicationContext

默认的spring容器

org.springframework.web.context.support.XmlWebApplicationContext#loadBeanDefinitions(org.springframework.beans.factory.support.DefaultListableBeanFactory)

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory){
    // 创建XmlBeanDefinitionReader，用它来读取XML配置文件
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    // 配置beanDefinitionReader的环境和属性等
    beanDefinitionReader.setEnvironment(getEnvironment());
    beanDefinitionReader.setResourceLoader(this);
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // 初始化beanDefinitionReader，子类可以实现这个方法，做一些个性化配置和初始化
    initBeanDefinitionReader(beanDefinitionReader);
    // 开始load xml文件
    loadBeanDefinitions(beanDefinitionReader);
}
```

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) {
    //读取web.xml中的contextConfigLocation元素，没有读默认
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        for (String configLocation : configLocations) {
            //reader读取xml文件
            reader.loadBeanDefinitions(configLocation);
        }
    }
}
```

```java
public int loadBeanDefinitions(EncodedResource encodedResource) {
     //将Resource对象添加到hashSet中
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    if (currentResources == null) {
        currentResources = new HashSet<EncodedResource>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
    }
    InputStream inputStream = encodedResource.getResource().getInputStream();
    InputSource inputSource = new InputSource(inputStream);
    if (encodedResource.getEncoding() != null) {
        inputSource.setEncoding(encodedResource.getEncoding());
    }
     // 加载封装好的inputSource对象，读取XML配置文件
    return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
}
finally {
    currentResources.remove(encodedResource);
    if (currentResources.isEmpty()) {
        this.resourcesCurrentlyBeingLoaded.remove();
    }
}
}
```

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) {
    //将xml转为Document
    Document doc = doLoadDocument(inputSource, resource);
    //Document转换为BeanDefinition并注册到BeanDefinitionMap
    return registerBeanDefinitions(doc, resource);

}
```

```java
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>(64);
```

