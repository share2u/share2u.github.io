---
title: spring-bean的创建和初始化
date: 2018-08-21 20:41:10
categories:
 - spring
tags:
 - spring
---

反射创建bean以及初始化

<!-- more-->

```java

protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    // I初始化conversionService类型转换bean，它可以服务于其他bean的类型转换
    if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
        beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
        beanFactory.setConversionService(
            beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
    }

    //// 初始化LoadTimeWeaverAware bean
    String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
    for (String weaverAwareName : weaverAwareNames) {
        getBean(weaverAwareName);
    }

    // Stop using the temporary ClassLoader for type matching.
    //// 停止使用临时的ClassLoader，
    beanFactory.setTempClassLoader(null);
    // Allow for caching all bean definition metadata, not expecting further changes.
    beanFactory.freezeConfiguration();

    // Instantiate all remaining (non-lazy-init) singletons.
    //创建和初始化非lazy-init的singleton beans
    beanFactory.preInstantiateSingletons();
}
```

```java
public void preInstantiateSingletons() throws BeansException {
    // Iterate over a copy to allow for init methods which in turn register new bean definitions.
    // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
    // 获取XML配置文件解析时，解析到的所有beanname
    List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

    // 遍历所有没有标注lazy-init的singleton的beanname,创建bean
    for (String beanName : beanNames) {
        //利用beanname获取BeanDefinition，在XML解析时会生成BeanDefinition对象，
        //将XML中的各属性添加到BeanDefinition的相关标志位中，比如abstractFlag，scope等
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 非abstract，非lazy-init的singleton bean才需要在容器初始化阶段创建
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 处理FactoryBean
            if (isFactoryBean(beanName)) {
                 // 获取FactoryBean实例，FactoryBean前面会加一个&符号
                final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                boolean isEagerInit;
                if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                    isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                        @Override
                        public Boolean run() {
                            return ((SmartFactoryBean<?>) factory).isEagerInit();
                        }
                    }, getAccessControlContext());
                }
                else {
                    isEagerInit = (factory instanceof SmartFactoryBean &&
                                   ((SmartFactoryBean<?>) factory).isEagerInit());
                }
                 // 非Factorybean,直接调用getBean方法
                if (isEagerInit) {
                    getBean(beanName);
                }
            }
            else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    // bean创建后，对SmartInitializingSingleton回调afterSingletonsInstantiated()方法
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                    @Override
                    public Object run() {
                        smartSingleton.afterSingletonsInstantiated();
                        return null;
                    }
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}
```

```java
public Object getBean(String name) throws BeansException {
   return doGetBean(name, null, null, false);
}
```

### doGetBean

```java
protected <T> T doGetBean(
    final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
    throws BeansException {
    //beanname转换，去掉FactoryBean的&前缀，处理alias声明
    final String beanName = transformedBeanName(name);
    Object bean;
    // Eagerly check singleton cache for manually registered singletons.
    Object sharedInstance = getSingleton(beanName);
    if (sharedInstance != null && args == null) {
        // 判断singleton bean是否已经创建好了，创建好了则直接从内存取出。
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        // Check if bean definition exists in this factory.
        //检查是否有beanname对应的BeanDefinition
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // 没有找到BeanDefinition，看看parent工厂中有没有，调用parent工厂的getBean
            // 获取原始的name，包含了FactoryBean前缀，&符号
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // Delegation to parent with explicit args.
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // No args -> delegate to standard getBean method.
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }
        if (!typeCheckOnly) {
            markBeanAsCreated(beanName);
        }
        // 找到了beanname对应的BeanDefinition，合并parent的BeanDefinition(XML中的parent属性)
        final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
        checkMergedBeanDefinition(mbd, beanName, args);

        // Guarantee initialization of beans that the current bean depends on.
        // 处理dependsOn属性
        String[] dependsOn = mbd.getDependsOn();
        if (dependsOn != null) {
            // 遍历所有的dependOn bean，要先注册和创建依赖的bean
            for (String dependsOnBean : dependsOn) {
                 // check是否两个bean是循环依赖，spring不能出现bean的循环依赖
                if (isDependent(beanName, dependsOnBean)) {
                    throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                                    "Circular depends-on relationship between '" + beanName + "' and '" + dependsOnBean + "'");
                }
                 // 注册并创建依赖的bean
                registerDependentBean(dependsOnBean, beanName);
                getBean(dependsOnBean);
            }
        }

        // Create bean instance.
        // 处理scope属性
        if (mbd.isSingleton()) {
            // singleton, 必须保证线程安全情况下创建bean，保证单例
            sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                @Override
                public Object getObject() throws BeansException {
                    // 反射创建bean实例
                    return createBean(beanName, mbd, args);
                }
            });
            // 获取bean实例
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }else if (mbd.isPrototype()) {
            // It's a prototype -> create a new instance.
            Object prototypeInstance = null;
            try {
                // 创建前的回调
                beforePrototypeCreation(beanName);
                // 反射创建bean实例
                prototypeInstance = createBean(beanName, mbd, args);
            }finally {
                // 创建后的回调，清除inCreation的标志
                afterPrototypeCreation(beanName);
            }
            bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
        }else {
             // 其他scope值
            String scopeName = mbd.getScope();
            final Scope scope = this.scopes.get(scopeName);
            Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                @Override
                public Object getObject() throws BeansException {
                    beforePrototypeCreation(beanName);
                    try {
                        return createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                }
            });
            bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
        }
    }
}
}
// check 创建的bean是否是requiredType指明的类型。如果不是，先做转换，转换不成的话只能类型不匹配抛出异常了
if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
    // 尝试将创建的bean转换为requiredType指明的类型
    return getTypeConverter().convertIfNecessary(bean, requiredType);  
}
return (T) bean;
}
```

```java
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args){
    // Make sure bean class is actually resolved at this point.
    // 拷贝一个新的RootBeanDefinition供创建bean使用
    resolveBeanClass(mbd, beanName);
    // Prepare method overrides.
    //处理bean中定义的覆盖方法，主要是xml:lookup-method或replace-method。
    //标记override的方法为已经加载过的，避免不必要的参数检查开销
    mbd.prepareMethodOverrides();
    // 调用BeanPostProcessors bean后处理器，使得bean后处理器可以返回一个proxy bean，
    //从而代替我们要创建的bean。回调后处理器的postProcessBeforeInstantiation()方法，如果这个方法中返回了一个bean，也就是使用了proxy，则再回调postProcessAfterInitialization()方法。之后返回这个Proxy bean即可。
    Object bean = resolveBeforeInstantiation(beanName, mbd);
    if (bean != null) {
        return bean;
    }
    // doCreateBean创建bean实例
    Object beanInstance = doCreateBean(beanName, mbd, args);
    return beanInstance;
}
```

### doCreateBean

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {
    // 创建bean实例，如果是singleton，先尝试从缓存中取，取不到则创建
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 反射创建bean实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);
    Class<?> beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);

    // Allow post-processors to modify the merged bean definition.
    // 回调MergedBeanDefinitionPostProcessor.postProcessMergedBeanDefinition，它可以修改bean属性
    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            mbd.postProcessed = true;
        }
    }

    // Eagerly cache singletons to be able to resolve circular references
    // even when triggered by lifecycle interfaces like BeanFactoryAware.
    // 曝光单例对象的引用，主要是为了解决单例间的循环依赖问题，以及依赖的bean比较复杂时的初始化性能问题
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                         "' to allow for resolving potential circular references");
        }
        addSingletonFactory(beanName, new ObjectFactory<Object>() {
            @Override
            public Object getObject() throws BeansException {
                return getEarlyBeanReference(beanName, mbd, bean);
            }
        });
    }
    // 初始化bean
    Object exposedObject = bean;
    populateBean(beanName, mbd, instanceWrapper);
    if (exposedObject != null) {
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    // 单例曝光对象的处理
    if (earlySingletonExposure) {
        Object earlySingletonReference = getSingleton(beanName, false);
        if (earlySingletonReference != null) {
            if (exposedObject == bean) {
                exposedObject = earlySingletonReference;
            }
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
                String[] dependentBeans = getDependentBeans(beanName);
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);
                for (String dependentBean : dependentBeans) {
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
                        actualDependentBeans.add(dependentBean);
                    }
                }
                if (!actualDependentBeans.isEmpty()) {
                    throw new BeanCurrentlyInCreationException;
                }
            }
        }
    }
    // Register bean as disposable.
     // 注册bean为可销毁的bean，bean销毁时，会回调destroy-method
    registerDisposableBeanIfNecessary(beanName, bean, mbd);
    return exposedObject;
}
```

#### createBeanInstance

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // Make sure bean class is actually resolved at this point.
    // 先创建class对象，反射的套路。利用bean的class属性进行反射，所以class属性一定要是bean的实现类
    Class<?> beanClass = resolveBeanClass(mbd, beanName);
	// class如果不是public的，则抛出异常。因为没法进行实例化
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                        "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }
// 使用FactoryBean的factory-method来创建，支持静态工厂和实例工厂
    if (mbd.getFactoryMethodName() != null)  {
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // Shortcut when re-creating the same bean...
    // 无参数情况时，创建bean。调用无参构造方法
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            // autoWire创建 自动装配
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
             // 普通创建
            return instantiateBean(beanName, mbd);
        }
    }

    // Need to determine the constructor...
    // 有参数情况时，创建bean。先利用参数个数，类型等，确定最精确匹配的构造方法。
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // No special handling: simply use no-arg constructor.
    // 有参数时，又没获取到构造方法，则只能调用无参构造方法来创建实例了(兜底方法)
    return instantiateBean(beanName, mbd);
}
```

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    Object beanInstance;
    final BeanFactory parent = this;
    if (System.getSecurityManager() != null) {
        beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                // 创建实例
                return getInstantiationStrategy().instantiate(mbd, beanName, parent);
            }
        }, getAccessControlContext());
    }
    else {
        beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
    }
    BeanWrapper bw = new BeanWrapperImpl(beanInstance);
    initBeanWrapper(bw);
    return bw;
}
```

##### instantiate

```java
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
    // Don't override the class with CGLIB if no overrides.
    if (bd.getMethodOverrides().isEmpty()) {
        Constructor<?> constructorToUse;
        // 保证线程安全情况下，获取Constructor
        synchronized (bd.constructorArgumentLock) {
            // 获取构造方法或factory-method
            constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
            if (constructorToUse == null) {
                // BeanDefinition中如果没有Constructor或者factory-method，则直接使用默认无参构造方法
                final Class<?> clazz = bd.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                if (System.getSecurityManager() != null) {
                    constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
                        @Override
                        public Constructor<?> run() throws Exception {
                            return clazz.getDeclaredConstructor((Class[]) null);
                        }
                    });
                }
                else {
                    // 获取默认无参构造方法
                    constructorToUse = clazz.getDeclaredConstructor((Class[]) null);
                }
                bd.resolvedConstructorOrFactoryMethod = constructorToUse;

            }
        }
        // 使用上一步得到的Constructor，反射获取bean实例
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // Must generate CGLIB subclass.
        return instantiateWithMethodInjection(bd, beanName, owner);
    }
}
```

instantiate方法主要做两件事

1. 确定Constructor或者factory-method
2. 利用Constructor，反射创建bean实例

#### initializeBean

bean创建完后，容器会对它进行初始化，包括后处理的调用，init-method的调用等 

```java
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
    // 回调各种aware method，如BeanNameAware， BeanFactoryAware等
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged(new PrivilegedAction<Object>() {
            @Override
            public Object run() {
                invokeAwareMethods(beanName, bean);
                return null;
            }
        }, getAccessControlContext());
    }
    else {
        // 回调init-method
        invokeAwareMethods(beanName, bean);
    }
    // 回调beanPostProcessor的postProcessBeforeInitialization()方法
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }
    invokeInitMethods(beanName, wrappedBean, mbd);
    if (mbd == null || !mbd.isSynthetic()) {
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }
    return wrappedBean;
}
```

由此可见，initializeBean(),也就是bean的初始化流程为

1. 回调各种aware method，如BeanNameAware，将容器中相关引用注入到bean中，供bean使用
2. 回调beanPostProcessor的postProcessBeforeInitialization(), 后处理器的初始化前置调用
3. 回调init-method， 注解和XML中都可以声明
4. 回调beanPostProcessor的postProcessAfterInitialization()方法，后处理器的初始化后置调用。

