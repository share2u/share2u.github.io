---
title: shiro工作流程
date: 2018-11-01 11:09:58
categories:
- shiro
tags:
- shiro
- filter
---

基本描述以及shiro是如何执行的，包含认证和授权流程

## 基本描述

- Apache Shiro是Java的一个安全（权限）框架
- 可以在JavaSE，javaEE环境
- 功能点：认证，授权，加密、会话管理、web集成、缓存等

![shiro功能点](shiro工作流程\shiro基本结构.png)



### 应用程序角度

![shiro外部架构](shiro工作流程\shiro外部架构.png)

### Shiro内部架构

![1532848598409](shiro工作流程\shiro内部结构.png)

## 例子

```java
public class UserRealm extends AuthorizingRealm {
 
    private UserService userService;
 
    public void setUserService(UserService userService) {
        this.userService = userService;
    }
 
	// 从数据库中获取权限信息
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        String username = (String)principals.getPrimaryPrincipal();
 
        SimpleAuthorizationInfo authorizationInfo = new SimpleAuthorizationInfo();
		// 从数据库中查询当前用户所拥有的角色
        authorizationInfo.setRoles(userService.findRoles(username));
		// 从数据库中查询当前用户所拥有的权限
        authorizationInfo.setStringPermissions(userService.findPermissions(username));
        return authorizationInfo;
    }
	
	// 从数据库中获取认证信息
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        String username = (String)token.getPrincipal();
        User user = userService.findByUsername(username);
        if(user == null) {
            throw new UnknownAccountException();//没找到帐号
        }
        if(Boolean.TRUE.equals(user.getLocked())) {
            throw new LockedAccountException(); //帐号锁定
        }
        //交给AuthenticatingRealm使用CredentialsMatcher进行密码匹配，如果觉得人家的不好可以自定义实现
        SimpleAuthenticationInfo authenticationInfo = new SimpleAuthenticationInfo(
                user.getUsername(), //用户名 principal
                user.getPassword(), //密码   hashedCredentials
            	//注册的时候生成盐值   credentialsSalt
                ByteSource.Util.bytes(user.getCredentialsSalt()),//salt=username+salt 
                getName()  //realmName
        );
        return authenticationInfo;
    }
 
}
```

## 工作流程

### 入口：DelegatingFilterProxy 

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
            /loginsuccess = anon   ===AnonymousFilter
            /logout = logout
            /user.jsp = roles[user]====RolesAuthorizationFilter
            /** = authc            ====FormAuthenticationFilter
        </value>
    </property>
</bean>
```

### 初始化：shiroFilter

![1540385548793](..\..\images\blog\shiroFilter对象.png)

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

### filter执行---匹配filter

(错误)通过请求的url,获取要走的过滤链，或者说，所有请求都走一样的过滤链，过滤器中由判断自己是否支持该请求

（正解）,所有的请求走一个过滤器org.springframework.web.filter.DelegatingFilterProxy#doFilter 

```java
org.apache.shiro.web.servlet.OncePerRequestFilter#doFilter==执行被代理类doFilter
	org.apache.shiro.web.servlet.AbstractShiroFilter#doFilterInternal	
		createSubject  AbstractShiroFilter#createSubject== 创建subject
		updateSessionLastAccessTime                     == 更新session最后访问时间
		executeChain AbstractShiroFilter#executeChain   ==获取过滤链，执行过滤方法
```

```java
protected void executeChain(ServletRequest request, ServletResponse response, FilterChain origChain){
    FilterChain chain = getExecutionChain(request, response, origChain);
    //获取过滤链：如FormAuthenticationFilter
    chain.doFilter(request, response);
}
```

```java
eg:FormAuthenticationFilter#doFilter===父类OncePerRequestFilter的doFilter
		OncePerRequestFilter#doFilter---doFilterInternal---AdviceFilter#doFilterInternal
```

```java
AdviceFilter#doFilterInternal
	preHandle
			AdviceFilter#preHandle  true
			LogoutFilter#preHandle  false
			PathMatchingFilter#preHandle--isFilterChainContinued--onPreHandle
				AnonymousFilter#onPreHandle true
				NoSessionCreationFilter#onPreHandle true
				PathMatchingFilter#onPreHandle  true
				AccessControlFilter#onPreHandle
					isAccessAllowed
					onAccessDenied
			executeChain 							==其他过滤器执行
			postHandle   							==没有实现类
			cleanup
				AdviceFilter#cleanup                  ==没实现
				AuthenticatingFilter#cleanup
					onAccessDenied   
						FormAuthenticationFilter#onAccessDenied   
						BasicHttpAuthenticationFilter#onAccessDenied
```



filter 执行顺序

https://www.cnblogs.com/ljdblog/p/6237683.html

https://www.cnblogs.com/q95265/p/6928081.html

### filter执行---doFilter

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

OncePerRequestFilter：保证每个filter都执行一次

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

isAccessAllowed：表示是否允许访问；mappedValue就是[urls]配置中拦截器参数部分，如果允许访问返回true，否则false；

onAccessDenied：表示当访问拒绝时是否已经处理了；如果返回true表示需要继续处理；如果返回false表示该拦截器实例已经处理了，将直接返回即可

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

## 认证流程

```java
public void login(AuthenticationToken token) throws AuthenticationException {
    clearRunAsIdentitiesInternal();
    //securityManager 登录
    Subject subject = securityManager.login(this, token);

    PrincipalCollection principals;

    String host = null;

    if (subject instanceof DelegatingSubject) {
        DelegatingSubject delegating = (DelegatingSubject) subject;
        //we have to do this in case there are assumed identities - we don't want to lose the 'real' principals:
        principals = delegating.principals;
        host = delegating.host;
    } else {
        principals = subject.getPrincipals();
    }

    if (principals == null || principals.isEmpty()) {
        String msg = "Principals returned from securityManager.login( token ) returned a null or " +
            "empty value.  This value must be non null and populated with one or more elements.";
        throw new IllegalStateException(msg);
    }
    this.principals = principals;
    this.authenticated = true;
    if (token instanceof HostAuthenticationToken) {
        host = ((HostAuthenticationToken) token).getHost();
    }
    if (host != null) {
        this.host = host;
    }
    Session session = subject.getSession(false);
    if (session != null) {
        this.session = decorate(session);
    } else {
        this.session = null;
    }
}
```

```java
public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
    AuthenticationInfo info;
    try {
        //关键逻辑
        info = authenticate(token);
    } catch (AuthenticationException ae) {
        try {
            onFailedLogin(token, ae, subject);
        } catch (Exception e) {
            if (log.isInfoEnabled()) {
                log.info("onFailedLogin method threw an " +
                         "exception.  Logging and propagating original AuthenticationException.", e);
            }
        }
        throw ae; //propagate
    }
    //认证成功，重新创建subject
    Subject loggedIn = createSubject(token, info, subject);
    //处理rememberme逻辑
    onSuccessfulLogin(token, info, loggedIn);

    return loggedIn;
}
```

org.apache.shiro.mgt.AuthenticatingSecurityManager#authenticate

```java
public AuthenticationInfo authenticate(AuthenticationToken token) {
    return this.authenticator.authenticate(token);
}
```

org.apache.shiro.authc.AbstractAuthenticator#authenticate

​	org.apache.shiro.authc.pam.ModularRealmAuthenticator#authenticate

模板方法模式

```java
 protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
        assertRealmsConfigured();
        Collection<Realm> realms = getRealms();
        if (realms.size() == 1) {
            return doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
        } else {
            return doMultiRealmAuthentication(realms, authenticationToken);
        }
    }
```

```java
protected AuthenticationInfo doMultiRealmAuthentication(Collection<Realm> realms, AuthenticationToken token) {
    //多realm下的认证策略默认实现为AtLeastOneSuccessfulStrategy
    AuthenticationStrategy strategy = getAuthenticationStrategy();
    AuthenticationInfo aggregate = strategy.beforeAllAttempts(realms, token);
    for (Realm realm : realms) {
        aggregate = strategy.beforeAttempt(realm, token, aggregate);
        if (realm.supports(token)) {
            //关键实现方法
            AuthenticationInfo info = realm.getAuthenticationInfo(token);
            aggregate = strategy.afterAttempt(realm, token, info, aggregate, t);
        } else {
            log.debug("Realm [{}] does not support token {}.  Skipping realm.", realm, token);
        }
    }
    aggregate = strategy.afterAllAttempts(token, aggregate);
    return aggregate;
}
```

```java
public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
    AuthenticationInfo info = getCachedAuthenticationInfo(token);
    if (info == null) {
        //覆盖实现核心认证信息
        info = doGetAuthenticationInfo(token);
        if (token != null && info != null) {
            cacheAuthenticationInfoIfPossible(token, info);
        }
    }
    if (info != null) {
        assertCredentialsMatch(token, info);
    }
    return info;
}
```

eg:自定义realmA实现AuthenticatingRealm

```java
public class ShiroRealmA extends AuthenticatingRealm {
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        System.out.println("===>ShiroRealmA<=====" + JSON.toJSONString(token));
        /**
         * 1. 吧AuthenticationToken转为usernamepasswordtoken
         * 2. 获取用户名与密码
         * 3. 调用数据库的方法
         * 4. 根据数据库用户情况抛出异常或构建AuthenticationInfo
         */
        UsernamePasswordToken usernamePasswordToken = (UsernamePasswordToken) token;
        String username = usernamePasswordToken.getUsername();
        // 数据库获取的密码，，前端加密 密码加密
        //盐值一般是唯一值，可以是用户名
        ByteSource salt = ByteSource.Util.bytes(username);
        Object credentials = "123";
        Object credentialsMD5 = new SimpleHash("MD5", credentials, salt, 2);

        if ("zs".equals(username)) {
            return new SimpleAuthenticationInfo(username,credentialsMD5, salt, this.getName());
        } else {
            throw new AuthenticationException();
        }
    }
}
```

## 授权流程

perms(PermissionsAuthorizationFilter.class) ：url是否有权限

roles(RolesAuthorizationFilter.class), ：url是否有该角色

```java
public boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws IOException {
    Subject subject = getSubject(request, response);
    //访问需要的权限
    String[] perms = (String[]) mappedValue;
    //subject判断是否有权限
    boolean isPermitted = true;
    if (perms != null && perms.length > 0) {
        if (perms.length == 1) {
            if (!subject.isPermitted(perms[0])) {
                isPermitted = false;
            }
        } else {
            if (!subject.isPermittedAll(perms)) {
                isPermitted = false;
            }
        }
    }
    return isPermitted;
}
```

ModularRealmAuthorizer.isPermitted 

```java
public boolean isPermitted(PrincipalCollection principals, String permission) {
    assertRealmsConfigured();
    for (Realm realm : getRealms()) {
        if (!(realm instanceof Authorizer)) continue;
        // 调用Realm的isPermitted方法
        if (((Authorizer) realm).isPermitted(principals, permission)) {
            return true;
        }
    }
    return false;
}
```

```java
public boolean isPermitted(String permission) {
    return hasPrincipals() && securityManager.isPermitted(getPrincipals(), permission);
}
```

```java
public boolean isPermitted(PrincipalCollection principals, String permissionString) {
    return this.authorizer.isPermitted(principals, permissionString);
```

```java
  public boolean isPermitted(PrincipalCollection principals, String permission) {
        Permission p = getPermissionResolver().resolvePermission(permission);
        return isPermitted(principals, p);
    }

    public boolean isPermitted(PrincipalCollection principals, Permission permission) {
        AuthorizationInfo info = getAuthorizationInfo(principals);
        return isPermitted(permission, info);
    }
```

```java
protected AuthorizationInfo getAuthorizationInfo(PrincipalCollection principals) {

    AuthorizationInfo info = null;
	//缓存中获取授权信息
    Cache<Object, AuthorizationInfo> cache = getAvailableAuthorizationCache();
    if (cache != null) {
        Object key = getAuthorizationCacheKey(principals);
        info = cache.get(key);
    }
    if (info == null) {
        // 继承覆盖实现获取授权信息
        info = doGetAuthorizationInfo(principals);
        if (info != null && cache != null) {
            Object key = getAuthorizationCacheKey(principals);
            cache.put(key, info);
        }
    }

    return info;
}
```

ModularRealmAuthorizer.isPermittedAll 

AuthorizationFilter#onAccessDenied

```java
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws IOException {
    Subject subject = getSubject(request, response);
    //未登陆，跳转到登陆界面
    if (subject.getPrincipal() == null) {
        saveRequestAndRedirectToLogin(request, response);
    } else {
        //登陆未授权、返回未授权界面或者401状态码
        String unauthorizedUrl = getUnauthorizedUrl();
        if (StringUtils.hasText(unauthorizedUrl)) {
            WebUtils.issueRedirect(request, response, unauthorizedUrl);
        } else {
            WebUtils.toHttp(response).sendError(HttpServletResponse.SC_UNAUTHORIZED);
        }
    }
    return false;
}
```



## FormAuthenticationFilter

Onceperrequestfilter#doFilter

adviceFilter#doFilterInternal

AdviceFilter#preHandle  true

AuthenticatingFilter#cleanup

FormAuthenticationFilter#onAccessDenied

```java
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
    //是否为登陆请求
    if (isLoginRequest(request, response)) {
        //是否是post请求
        if (isLoginSubmission(request, response)) {
            return executeLogin(request, response);
        } else {
            return true;
        }
    } else {
        saveRequestAndRedirectToLogin(request, response);
        return false;
    }
}
```

org.apache.shiro.web.filter.authc.AuthenticatingFilter#executeLogin

```java
protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
    AuthenticationToken token = createToken(request, response);
    try {
        Subject subject = getSubject(request, response);
        subject.login(token);
        return onLoginSuccess(token, subject, request, response);
    } catch (AuthenticationException e) {
        return onLoginFailure(token, e, request, response);
    }
}
```

go to 认证流程

## PermissionsAuthorizationFilter

Onceperrequestfilter#doFilter

adviceFilter#doFilterInternal

PathMatchingFilter#preHandle

AccessControlFilter#onPreHandle

go to 授权流程

​	PermissionsAuthorizationFilter#isAccessAllowed

​	AuthorizationFilter#onAccessDenied

## RolesAuthorizationFilter

与PermissionsAuthorizationFilter类似

## UserFilter

是登录页面或者必须登录后获得principalCollection，才能通过

```java
protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {  
        if (isLoginRequest(request, response)) {  
            return true;  
        } else {  
            Subject subject = getSubject(request, response);  
            return subject.getPrincipal() != null;  
        }  
}
```

```java
protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {  
        saveRequestAndRedirectToLogin(request, response);  
        return false;  
} 
```



参考文献

https://blog.csdn.net/xtayfjpk/article/details/53729135 

http://suichangkele.iteye.com/blog/2277023 