---
title: Spring Security配置初始化@EnableWebSecurity注解详解
categories:
  - spring security
---

通过EnableSecurity注解导入组件
```java
@Retention(value = java.lang.annotation.RetentionPolicy.RUNTIME)
@Target(value = { java.lang.annotation.ElementType.TYPE })
@Documented
@Import({ WebSecurityConfiguration.class,
		SpringWebMvcImportSelector.class,
		OAuth2ImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
	//...
}
```
### WebSecurityConfiguration配置类
这个类中最主要就是的两个方法，
1. setFilterChainProxySecurityConfigurer，这个方法有@Autowired注解，会先执行，执行时AutowiredWebSecurityConfigurersIgnoreParents通过这个类的getWebSecurityConfigurers()方法会获取所有实现WebSecurityConfigurer的类，如果我们有自己实现了
WebSecurityConfigurerAdapter的类就会被这个方法获取到对应的实例，排序后放到websecurity中。在这个方法中还会创建我们第一个比较重要的对象webSecurity。
2. 接着会调用springSecurityFilterChain()方法，这个方法会判断我们上一个方法中有没有获取到webSecurityConfigurers，没有的话这边会创建一个WebSecurityConfigurerAdapter实例，并追加到websecurity中。接着调用websecurity的build方法。实际调用的是websecurity的父类AbstractSecurityBuilder的build方法

### Websecurity类
Websecurity类主要的继承关系为自Websecurity->AbstractConfiguredSecurityBuilder->AbstractSecurityBuilder-->SecurityBuilder，其中上一步调用的build方法就在AbstractSecurityBuilder中
```java
public final O build() throws Exception {
		if (this.building.compareAndSet(false, true)) {
			this.object = doBuild();
			return this.object;
		}
		throw new AlreadyBuiltException("This object has already been built");
	}
```
具体的doBuild()逻辑在子类AbstractConfiguredSecurityBuilder的doBuild方法中
```java
@Override
protected final O doBuild() throws Exception {
	synchronized (configurers) {
		buildState = BuildState.INITIALIZING;

		beforeInit();
		init();

		buildState = BuildState.CONFIGURING;

		beforeConfigure();
		configure();

		buildState = BuildState.BUILDING;

		O result = performBuild();

		buildState = BuildState.BUILT;

		return result;
	}
}
```
可以看到build过程主要分三步，init->configure->peformBuild
- 第一步 init
在init方法中，会调用设置好的SecurityConfigurer列表中每一个configurer的init方法，这个SecurityConfigurer列表就是我们在WebSecurityConfiguration类中通过调用apply()方法设置进来的，即WebSecurityConfigurerAdapter这个类或者是我们继承了WebSecurityConfigurerAdapter的类。

### WebSecurityConfigurerAdapter类（最重要的一个类，我们定制化也是继承这个类进行设置的）
WebSecurityConfigurerAdapter类主要的继承关系为WebSecurityConfigurerAdapter-->WebSecurityConfigurer->SecurityConfigurer
这个类中主要的是下面的三个方法，
```java
public void init(final WebSecurity web) throws Exception {
	final HttpSecurity http = getHttp();
	web.addSecurityFilterChainBuilder(http).postBuildAction(new Runnable() {
		public void run() {
			FilterSecurityInterceptor securityInterceptor = http
					.getSharedObject(FilterSecurityInterceptor.class);
			web.securityInterceptor(securityInterceptor);
		}
	});
}
```
```java
protected final HttpSecurity getHttp() throws Exception {
	if (http != null) {
		return http;
	}

	DefaultAuthenticationEventPublisher eventPublisher = objectPostProcessor
			.postProcess(new DefaultAuthenticationEventPublisher());
	localConfigureAuthenticationBldr.authenticationEventPublisher(eventPublisher);

	AuthenticationManager authenticationManager = authenticationManager();
	authenticationBuilder.parentAuthenticationManager(authenticationManager);
	authenticationBuilder.authenticationEventPublisher(eventPublisher);
	Map<Class<? extends Object>, Object> sharedObjects = createSharedObjects();

	http = new HttpSecurity(objectPostProcessor, authenticationBuilder,
			sharedObjects);
	if (!disableDefaults) {
		// @formatter:off
		http
			.csrf().and()
			.addFilter(new WebAsyncManagerIntegrationFilter())
			.exceptionHandling().and()
			.headers().and()
			.sessionManagement().and()
			.securityContext().and()
			.requestCache().and()
			.anonymous().and()
			.servletApi().and()
			.apply(new DefaultLoginPageConfigurer<>()).and()
			.logout();
		// @formatter:on
		ClassLoader classLoader = this.context.getClassLoader();
		List<AbstractHttpConfigurer> defaultHttpConfigurers =
				SpringFactoriesLoader.loadFactories(AbstractHttpConfigurer.class, classLoader);

		for (AbstractHttpConfigurer configurer : defaultHttpConfigurers) {
			http.apply(configurer);
		}
	}
	configure(http);
	return http;
}
```
```java
protected void configure(HttpSecurity http) throws Exception {
	logger.debug("Using default configure(HttpSecurity). If subclassed this will potentially override subclass configure(HttpSecurity).");

	http
		.authorizeRequests()
			.anyRequest().authenticated()
			.and()
		.formLogin().and()
		.httpBasic();
}
```
init方法做了两件事，一个就是调用getHttp()方法获取一个http实例，并通过web.addSecurityFilterChainBuilder方法把获取到的实例赋值给WebSecurity的securityFilterChainBuilders属性，这个属性在我们执行build的时候会用到，第二个就是为WebSecurity追加了一个postBuildAction，在build都完成后从http中拿出FilterSecurityInterceptor对象并赋值给WebSecurity。

getHttp()方法，这个方法在当我们使用默认配置时（大多数情况下）会为我们追加各种SecurityConfigurer的具体实现类到httpSecurity中，如exceptionHandling()方法会追加一个ExceptionHandlingConfigurer，sessionManagement()方法会追加一个SessionManagementConfigurer,securityContext()方法会追加一个SecurityContextConfigurer对象，这些SecurityConfigurer的具体实现类最终会为我们配置各种具体的filter，这些SecurityConfigurer类是怎么调用到的，下面会讲到
另外getHttp()方法的最后会调用configure(http)，这个方法也是我们继承WebSecurityConfigurerAdapter类后最可能会重写的方法

configure(HttpSecurity http)方法，默认的configure(HttpSecurity http)方法继续向httpSecurity类中追加SecurityConfigurer的具体实现类，如authorizeRequests()方法追加一个ExpressionUrlAuthorizationConfigurer，formLogin()方法追加一个FormLoginConfigurer。
其中ExpressionUrlAuthorizationConfigurer这个实现类后面会进一步探讨，因为他会给我们创建一个非常重要的对象FilterSecurityInterceptor对象，FormLoginConfigurer对象比较简单，但是也会为我们提供一个在安全认证过程中经常用到会用的一个Filter：UsernamePasswordAuthenticationFilter。


- 第二步 configure
configure方法最终也调用到了WebSecurityConfigurerAdapter的configure(WebSecurity web)方法，默认实现中这个是一个空方法，具体应用中也经常重写这个方法来实现特定需求。
- 第三步 peformBuild
具体的实现逻辑在WebSecurity类中
```java
protected Filter performBuild() throws Exception {
	Assert.state(
			!securityFilterChainBuilders.isEmpty(),
			() -> "At least one SecurityBuilder<? extends SecurityFilterChain> needs to be specified. "
					+ "Typically this done by adding a @Configuration that extends WebSecurityConfigurerAdapter. "
					+ "More advanced users can invoke "
					+ WebSecurity.class.getSimpleName()
					+ ".addSecurityFilterChainBuilder directly");
	int chainSize = ignoredRequests.size() + securityFilterChainBuilders.size();
	List<SecurityFilterChain> securityFilterChains = new ArrayList<>(
			chainSize);
	for (RequestMatcher ignoredRequest : ignoredRequests) {
		securityFilterChains.add(new DefaultSecurityFilterChain(ignoredRequest));
	}
	for (SecurityBuilder<? extends SecurityFilterChain> securityFilterChainBuilder : securityFilterChainBuilders) {
		securityFilterChains.add(securityFilterChainBuilder.build());
	}
	FilterChainProxy filterChainProxy = new FilterChainProxy(securityFilterChains);
	if (httpFirewall != null) {
		filterChainProxy.setFirewall(httpFirewall);
	}
	filterChainProxy.afterPropertiesSet();

	Filter result = filterChainProxy;
	if (debugEnabled) {
		logger.warn("\n\n"
				+ "********************************************************************\n"
				+ "**********        Security debugging is enabled.       *************\n"
				+ "**********    This may include sensitive information.  *************\n"
				+ "**********      Do not use in a production system!     *************\n"
				+ "********************************************************************\n\n");
		result = new DebugFilter(filterChainProxy);
	}
	postBuildAction.run();
	return result;
}
```
这个方法中最主要的任务就是遍历securityFilterChainBuilders属性中的SecurityBuilder对象，并调用他的build方法。
这个securityFilterChainBuilders属性我们前面也有提到过，就是在WebSecurityConfigurerAdapter类的init方法中获取http后赋值给了WebSecurity。因此这个地方就是调用httpSecurity的build方法。

### HttpSecurity类

HttpSecurity和WebSecurity两个类继承关系都一样，所以httpSecurity的build也是分三步
init->configure->performBuild,只不过在webSecurity中SecurityConfigurer列表在默认实现下只有WebSecurityConfigurerAdapter一个，而httpSecurity中的SecurityConfigurer列表我们在WebSecurityConfigurerAdapter的init方法中设置了很多，这时就会依次调用我们设置的SecurityConfigurer的具体实现类。
我在debug时看了，默认情况下springSecurity有下面的这些
```bash
[org.springframework.security.config.annotation.web.configurers.ExceptionHandlingConfigurer@350a94ce,
 org.springframework.security.config.annotation.web.configurers.HeadersConfigurer@7e00ed0f,
 org.springframework.security.config.annotation.web.configurers.SessionManagementConfigurer@b0fc838,
 org.springframework.security.config.annotation.web.configurers.SecurityContextConfigurer@3964d79,
 org.springframework.security.config.annotation.web.configurers.RequestCacheConfigurer@62db0521,
 org.springframework.security.config.annotation.web.configurers.AnonymousConfigurer@1b4ae4e0,
 org.springframework.security.config.annotation.web.configurers.ServletApiConfigurer@6ef1a1b9,
 org.springframework.security.config.annotation.web.configurers.DefaultLoginPageConfigurer@5fbdc49b,
 org.springframework.security.config.annotation.web.configurers.LogoutConfigurer@65753040,
 org.springframework.security.config.annotation.web.configurers.ExpressionUrlAuthorizationConfigurer@2954b5ea,
 org.springframework.security.config.annotation.web.configurers.FormLoginConfigurer@4acb2510,
 org.springframework.security.config.annotation.web.configurers.RememberMeConfigurer@7be3a9ce]
```

上面这些配置每个都完成一段逻辑，如RememberMeConfigurer会向HttpSecurity中追加一个RememberMeAuthenticationFilter实例，一个RememberMeAuthenticationProvider实例，用来做实际的认证。因为这些类做的事情都比较单一明确，从类名就可以猜测出来这些类做的工作，这里就不一个一个讲解，下面简单以ExpressionUrlAuthorizationConfigurer这个类为例来做个说明，因为这个类会为我们创建一个非常重要的对象FilterSecurityInterceptor

http.authorizeRequests()调用此方法会实例化ExpressionUrlAuthorizationConfigurer
ExpressionUrlAuthorizationConfigurer的继承关系
ExpressionUrlAuthorizationConfigurer->AbstractInterceptUrlConfigurer->AbstractHttpConfigurer->SecurityConfigurerAdapter-->SecurityConfigurer
对应的init方法在SecurityConfigurerAdapter类中，是个空实现，什么也没有做，configure方法在SecurityConfigurerAdapter类中也有一个空实现，在AbstractInterceptUrlConfigurer类中进行了重写
```java
@Override
public void configure(H http) throws Exception {
	FilterInvocationSecurityMetadataSource metadataSource = createMetadataSource(http);
	if (metadataSource == null) {
		return;
	}
	FilterSecurityInterceptor securityInterceptor = createFilterSecurityInterceptor(
			http, metadataSource, http.getSharedObject(AuthenticationManager.class));
	if (filterSecurityInterceptorOncePerRequest != null) {
		securityInterceptor
				.setObserveOncePerRequest(filterSecurityInterceptorOncePerRequest);
	}
	securityInterceptor = postProcess(securityInterceptor);
	http.addFilter(securityInterceptor);
	http.setSharedObject(FilterSecurityInterceptor.class, securityInterceptor);
}

private AccessDecisionManager createDefaultAccessDecisionManager(H http) {
	AffirmativeBased result = new AffirmativeBased(getDecisionVoters(http));
	return postProcess(result);
}

private FilterSecurityInterceptor createFilterSecurityInterceptor(H http,
			FilterInvocationSecurityMetadataSource metadataSource,
			AuthenticationManager authenticationManager) throws Exception {
	FilterSecurityInterceptor securityInterceptor = new FilterSecurityInterceptor();
	securityInterceptor.setSecurityMetadataSource(metadataSource);
	securityInterceptor.setAccessDecisionManager(getAccessDecisionManager(http));
	securityInterceptor.setAuthenticationManager(authenticationManager);
	securityInterceptor.afterPropertiesSet();
	return securityInterceptor;
}
```
在这个类的configure中创建了一个FilterSecurityInterceptor，并且也可以明确看到spring security默认给我们创建的AccessDecisionManager是AffirmativeBased。

最后再看下HttpSecurity类执行build的最后一步 performBuild，这个方法就是在HttpSecurity中实现的
```java
@Override
protected DefaultSecurityFilterChain performBuild() throws Exception {
	Collections.sort(filters, comparator);
	return new DefaultSecurityFilterChain(requestMatcher, filters);
}
```


可以看到，这个类只是把我们追加到HttpSecurity中的security进行了排序，用的排序类是FilterComparator，从而保证我们的filter按照正确的顺序执行。接着将filters构建成filterChian返回。在前面WebSecurity的performBuild方法中，这个返回值会被包装成FilterChainProxy，并作为WebSecurity的build方法的放回值。从而以springSecurityFilterChain这个名称注册到springContext中（在WebSecurityConfiguration中做的）

在WebSecurity的performBuild方法的最后一步还执行了一个postBuildAction.run，这个方法也是spring security给我们提供的一个hooks，可以在build完成后再做一些事情，比如我们在WebSecurityConfigurerAdapter类的init方法中我们利用这个hook在构建完成后将FilterSecurityInterceptor赋值给了webSecurity类的filterSecurityInterceptor属性
