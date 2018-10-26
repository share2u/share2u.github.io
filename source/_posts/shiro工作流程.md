---
title: shiro工作流程
date: 2018-10-24 09:42:02
categories:
- shiro
tags:
- shiro
- filter
---

web.xml中的shiro入口

```xml
<filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <init-param>
        <param-name>targetFilterLifecycle</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>shiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

DelegatingFilterProxy 是Filter的代理，代理的是spring容器中的filter-name一样的bean,

```xml
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
    <property name="securityManager" ref="securityManager"/>
    <property name="loginUrl" value="/login.jsp"/>
    <property name="successUrl" value="/loginsuccess"/>
    <property name="unauthorizedUrl" value="/unauthorized.jsp"/>
    <!--
        配置哪些页面需要受保护，以及访问这些页面需要的权限  过滤实现
            anon 可以被匿名访问
            authc 必须认证后才可以访问的页面
        -->
    <property name="filterChainDefinitions">
        <value>
            /loginsuccess = anon
            /logout = logout
            /user.jsp = roles[user]
            /** = authc
        </value>
    </property>
</bean>
```

### shiroFilter

![1540385548793](C:\Users\cwm\AppData\Local\Temp\1540385548793.png)

ShiroFilterFactoryBean 实现了spring FactoryBean ,getObject获取实例

```java
public Object getObject() throws Exception {
    if (this.instance == null) {
        this.instance = this.createInstance();
    }

    return this.instance;
}
```

```java
protected AbstractShiroFilter createInstance() throws Exception {
    log.debug("Creating Shiro Filter instance.");
    //1、安全管理器
    SecurityManager securityManager = this.getSecurityManager();
    //2、过滤器链管理器
    FilterChainManager manager = this.createFilterChainManager();
    //3、基于路径匹配的过滤器链解析器
    PathMatchingFilterChainResolver chainResolver = new PathMatchingFilterChainResolver();
    chainResolver.setFilterChainManager(manager);
    //返回shirofilter对象到spring容器中
    return new ShiroFilterFactoryBean.SpringShiroFilter((WebSecurityManager)securityManager, chainResolver);
}

```
property filterChainDefinitions 属性，会调用setFilterChainDefinitions

```java
public void setFilterChainDefinitions(String definitions) {
    Ini ini = new Ini();
    ini.load(definitions);
    //section key为url,value 为filter名称
    Section section = ini.getSection("urls");
    this.setFilterChainDefinitionMap(section);
}
```

#### SecurityManager



#### FilterChainManager

```java
protected FilterChainManager createFilterChainManager() {
    //创建DefaultFilterChainManager，添加默认的filters到manager
    DefaultFilterChainManager manager = new DefaultFilterChainManager();
    //设置默认Filter的基本属性
    Map<String, Filter> defaultFilters = manager.getFilters();
    for (Filter filter : defaultFilters.values()) {
        	//设置相关FilterURL的loginurl,successurl,unauthorizedUrl属性
            applyGlobalPropertiesIfNecessary(filter);
        }
    //获取在spring配置文件中的配置的Filter，eg:logout
    Map<String, Filter> filters = getFilters();
    if (!CollectionUtils.isEmpty(filters)) {
        for (Map.Entry<String, Filter> entry : filters.entrySet()) {
            String name = entry.getKey();
            Filter filter = entry.getValue();
            applyGlobalPropertiesIfNecessary(filter);
            if (filter instanceof Nameable) {
                ((Nameable) filter).setName(name);
            }
            // 将配置的Filter添加至manager中，如果同名Filter已存在则覆盖默认Filter
            manager.addFilter(name, filter, false);
        }
    }
    //配置的FilterChainDefinition
    Map<String, String> chains = this.getFilterChainDefinitionMap();
    if (!CollectionUtils.isEmpty(chains)) {
        for (Map.Entry<String, String> entry : chains.entrySet()) {
            String url = entry.getKey();
            String chainDefinition = entry.getValue();
            // 为配置的每一个URL匹配创建FilterChain定义，
            // 这样当访问一个URL的时候，一旦该URL配置上则就知道该URL需要应用上哪些Filter
            // 由于URL配置符会配置多个，所以以第一个匹配上的为准，所以越具体的匹配符应该配置在前面，越宽泛的匹配符配置在后面
            manager.createChain(url, chainDefinition);
        }
    }
    return manager;
}
```

##### DefaultFilterChainManager

```java
public DefaultFilterChainManager() {
    this.filters = new LinkedHashMap<String, Filter>();
    this.filterChains = new LinkedHashMap<String, NamedFilterList>();
    addDefaultFilters(false);
}
```

```java
protected void addDefaultFilters(boolean init) {
    for (DefaultFilter defaultFilter : DefaultFilter.values()) {
        addFilter(defaultFilter.name(), defaultFilter.newInstance(), init, false);
    }
}
```

###### DefaultFilter 过滤器

11个

```java
anon(AnonymousFilter.class),
authc(FormAuthenticationFilter.class),
authcBasic(BasicHttpAuthenticationFilter.class),
logout(LogoutFilter.class),
noSessionCreation(facion ico.class),
perms(PermissionsAuthorizationFilter.class),
port(PortFilter.class),
rest(HttpMethodPermissionFilter.class),
roles(RolesAuthorizationFilter.class),
ssl(SslFilter.class),
user(UserFilter.class);
```

###### filterChains过滤器链

是一个linkedhashmap,key为配置的url,value为对应的过滤器

#### PathMatchingFilterChainResolver

创建了新的filterchainfilter,然后又被前面创建的覆盖了，有问题！！！

1. 基于ant路径匹配方法匹配配置的url,

2. pathMatchingFilterChainResolver设置创建的FilterChainManager对象，所以URL匹配上后可以获取该URL需要应用的FilterChain了。 

### 执行---doFilter

```java
public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain){
    String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
    if ( request.getAttribute(alreadyFilteredAttributeName) != null ) {
        //已经执行过该过滤器
        filterChain.doFilter(request, response);
    } else if ( !isEnabled(request, response) ||shouldNotFilter(request) ) {
           //该过滤器不适合该请求
            filterChain.doFilter(request, response);
        } else {
            request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
            try {
                //执行该过滤器
                doFilterInternal(request, response, filterChain);
            } finally {
                // Once the request has finished, we're done and we don't
                // need to mark as 'already filtered' any more.
                request.removeAttribute(alreadyFilteredAttributeName);
            }
        }
}
```

shiro通过一系列url匹配符配置URL应用上的Filter，然后在Filter中完成相应的任务。核心逻辑-doFilterInternal

OncePerRequestFilter

​	--AbstractShiroFilter

​	--AdviceFilter

#### AbstractShiroFilter

springfactorybean返回的bean核心就是这个类，封装为shirofilter

构造方法

```java
public void init() throws Exception {
    WebEnvironment env = WebUtils.getRequiredWebEnvironment(getServletContext());
    //SecurityManager
    setSecurityManager(env.getWebSecurityManager());
    //FilterChainResolver
    FilterChainResolver resolver = env.getFilterChainResolver();
    if (resolver != null) {
        setFilterChainResolver(resolver);
    }
}
```

```java
protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain) {
    final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
    final ServletResponse response = prepareServletResponse(request, servletResponse, chain);
    final Subject subject = createSubject(request, response);
    //noinspection unchecked
    subject.execute(new Callable() {
        public Object call() throws Exception {
            updateSessionLastAccessTime(request, response);
            executeChain(request, response, chain);
            return null;
        }
    });
}
```

##### createSubject

每次调用都会创建subject

##### executeChain

```java
protected void executeChain(ServletRequest request, ServletResponse response, FilterChain origChain) {
    //获取当前url匹配的过滤器链
    FilterChain chain = getExecutionChain(request, response, origChain);
    //执行过滤器链中过滤器
    chain.doFilter(request, response);
}
```

```java
protected FilterChain getExecutionChain(ServletRequest request, ServletResponse response, FilterChain origChain) {
    FilterChain chain = origChain;
	//获取过滤器链解析器，即创建的PathMatchingFilterChainResolver对象
    FilterChainResolver resolver = getFilterChainResolver();
    if (resolver == null) {
        log.debug("No FilterChainResolver configured.  Returning original FilterChain.");
        return origChain;
    }
	// 调用其getChain方法，根据URL匹配相应的过滤器链
    FilterChain resolved = resolver.getChain(request, response, origChain);
    if (resolved != null) {
        log.trace("Resolved a configured FilterChain for the current request.");
        chain = resolved;
    } else {
        log.trace("No FilterChain configured for the current request.  Using the default.");
    }
    return chain;
}
```

#### AdviceFilter

这个类很多方法实现了spring中的aop特点，prehandle前置通知，posthandle后置通知，异常不执行，afterCompletion （最终通知，一定会执行），可以根据需求覆写这几个方法

```java
public void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain){
    Exception exception = null;
    try {
        //前置通知，判断该过滤链是否支持该请求
        boolean continueChain = preHandle(request, response);
        if (continueChain) {
            //执行过滤器链
            executeChain(request, response, chain);
        }
        //后置通知
        postHandle(request, response);
    } catch (Exception e) {
        exception = e;
    } finally {
        //最终通知
        cleanup(request, response, exception);
    }
}
```

eg:

- preHandle

1.  根据配置，访问URL:"/authenticated.jsp"时，会匹配上authc(FormAuthenticationFilter)
2.  FormAuthenticationFilter继承自PathMatchingFilter，所以返回true ，而logoutfilter会返回false,啥也没有返回给页面，页面显示空白

```java
protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
    for (String path : this.appliedPaths.keySet()) {
// If the path does match, then pass on to the subclass implementation for specific checks
//(first match 'wins'):
        if (pathsMatch(path, request)) {
log.trace("Current requestURI matches pattern '{}'.  Determining filter chain execution...", path);
            Object config = this.appliedPaths.get(path);
            return isFilterChainContinued(request, response, path, config);
        }
    }
    //no path matched, allow the request to go through:
    return true;
}
```

```java
private boolean isFilterChainContinued(ServletRequest request, ServletResponse response,
                                       String path, Object pathConfig) throws Exception {
    return onPreHandle(request, response, pathConfig);
}
```

org.apache.shiro.web.filter.AccessControlFilter#onPreHandle

```java
public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
    //如果isAccessAllowed方法返回false，则会执行onAccessDenied方法
    return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
}
```

org.apache.shiro.web.filter.authc.AuthenticatingFilter#isAccessAllowed

```java
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
    return super.isAccessAllowed(request, response, mappedValue) ||
            (!isLoginRequest(request, response) && isPermissive(mappedValue));
}
```

org.apache.shiro.web.filter.authc.AuthenticationFilter#isAccessAllowed

```java
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
    Subject subject = getSubject(request, response);
    return subject.isAuthenticated();
}
```

```java
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
    //是否为登陆请求
    if (isLoginRequest(request, response)) {
        //是否是POST请求
        if (isLoginSubmission(request, response)) {
            if (log.isTraceEnabled()) {
                log.trace("Login submission detected.  Attempting to execute login.");
            }
            return executeLogin(request, response);
        } else {
            if (log.isTraceEnabled()) {
                log.trace("Login page view.");
            }
            //allow them to see the login page ;)
            return true;
        }
    } else {
        if (log.isTraceEnabled()) {
            log.trace("Attempting to access a path which requires authentication.  Forwarding to the " +
                      "Authentication url [" + getLoginUrl() + "]");
        }
		//执行跳转到登陆界面login.jsp
        saveRequestAndRedirectToLogin(request, response);
        return false;
    }
}
```

login.jsp也会被拦截，执行到这里然后访问login.jsp,

再然后post请求这个url,就会执行executeLogin

org.apache.shiro.web.filter.authc.AuthenticatingFilter#executeLogin

```java
protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
    AuthenticationToken token = createToken(request, response);
    if (token == null) {
        String msg = "createToken method implementation returned null. A valid non-null AuthenticationToken " +
                "must be created in order to execute a login attempt.";
        throw new IllegalStateException(msg);
    }
    try {
        Subject subject = getSubject(request, response);
        subject.login(token);
        //重定向到上次访问的url,过滤链不再执行
        return onLoginSuccess(token, subject, request, response);
    } catch (AuthenticationException e) {
        //过滤链继续执行，返回到登陆界面
        return onLoginFailure(token, e, request, response);
    }
}
```

参考文献

https://blog.csdn.net/xtayfjpk/article/details/53729135 

http://suichangkele.iteye.com/blog/2277023