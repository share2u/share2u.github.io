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

有不同的实现类

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

创建一个转换器实例然后调用注册benaDefinitions

```java
public int registerBeanDefinitions(Document doc, Resource resource) {
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    documentReader.setEnvironment(getEnvironment());
    int countBefore = getRegistry().getBeanDefinitionCount();
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

```java
protected void doRegisterBeanDefinitions(Element root) {
    BeanDefinitionParserDelegate parent = this.delegate;
    this.delegate = createDelegate(getReaderContext(), root, parent);
    if (this.delegate.isDefaultNamespace(root)) {
        String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
        if (StringUtils.hasText(profileSpec)) {
            String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
            if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                return;
            }
        }
    }
    preProcessXml(root);
    //装换文档中的标签
    parseBeanDefinitions(root, this.delegate);
    postProcessXml(root);

    this.delegate = parent;
}
```

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    if (delegate.isDefaultNamespace(root)) {
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                if (delegate.isDefaultNamespace(ele)) {
                    parseDefaultElement(ele, delegate);
                }
                else {
                    delegate.parseCustomElement(ele);
                }
            }
        }
    }
    else {
        delegate.parseCustomElement(root);
    }
}
```

```java
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
    String namespaceUri = getNamespaceURI(ele);
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

各种方法的标签解析，如何选择的呢？

```java
NamespaceHandler
	SimplePropertyNamespaceHandler
	SimpleConstructorNamespaceHandler
	NamespaceHandlerSupport
		JeeNamespaceHandler
		AopNamespaceHandler
		ContextNamespaceHandler
		LangNamespaceHandler
		UtilNamespaceHandler
		MvcNamespaceHandler
		TaskNamespaceHandler
		CacheNamespaceHandler
		TxNamespaceHandler
```

