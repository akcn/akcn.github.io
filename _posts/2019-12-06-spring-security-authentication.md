---
title: Spring Security 认证过程
categories:
  - spring
---

最近在学习SpringSecurity的原理，开始看源码总是进去容易，调试着断点却发现越来越复杂，到最后发现出不来，很难理解大佬写代码的那种思维，
源码中有大量设计模式的应用，
比如User对象使用Builder构建模式，WebSecurityConfigurerAdapter使用适配器模式
对设计模式理解得不够深入，就很难能看懂，
所以以后在这方面还需多加强

先理解几个单词的含义:
principal 主要的 主角 代表登录用户
credentials 证件证明 可理解为密码
authorities 权力 授权
authentication 认证

spring Security功能的实现主要是由一系列过滤器链相互配合完成。本文通过源码分析其认证过程 基于Spring Security 5.1.5.RELEASE版本
请求经过 UsernamePasswordAuthenticationFilter 过滤器
此类的doFilter定义在父抽象类 AbstractAuthenticationProcessingFilter 中
```java
public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {

		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		if (!requiresAuthentication(request, response)) {
			chain.doFilter(request, response);

			return;
		}

		//...

		Authentication authResult;

		try {
			authResult = attemptAuthentication(request, response);
			if (authResult == null) {
				// return immediately as subclass has indicated that it hasn't completed
				// authentication
				return;
			}
			sessionStrategy.onAuthentication(authResult, request, response);
		}catch(){
			//...
		}
		// Authentication success
		if (continueChainBeforeSuccessfulAuthentication) {
			chain.doFilter(request, response);
		}

		successfulAuthentication(request, response, chain, authResult); // 认证成功后执行此方法 会在SecurityContext中设置Authentication 然后派发认证成功事件
```
调用实现类UsernamePasswordAuthenticationFilter的attemptAuthentication方法
```java
public Authentication attemptAuthentication(HttpServletRequest request,
			HttpServletResponse response) throws AuthenticationException {
		if (postOnly && !request.getMethod().equals("POST")) {
			throw new AuthenticationServiceException(
					"Authentication method not supported: " + request.getMethod());
		}

		String username = obtainUsername(request);
		String password = obtainPassword(request);

		if (username == null) {
			username = "";
		}

		if (password == null) {
			password = "";
		}

		username = username.trim();

		UsernamePasswordAuthenticationToken authRequest = new UsernamePasswordAuthenticationToken(
				username, password);

		// 将当前的请求信息设置到UsernamePasswordAuthenticationToken中
		// Allow subclasses to set the "details" property
		setDetails(request, authRequest); // WebAuthenticationDetails(请ip session id) OAuth2AuthenticationDetailsSource(client id ...)

		return this.getAuthenticationManager().authenticate(authRequest); // AuthenticationManager的实现类为ProviderManager
	}
```

此时new UsernamePasswordAuthenticationToken使用的构造器
```java
public UsernamePasswordAuthenticationToken(Object principal, Object credentials) {
		super(null); // 调用父类设置授权信息 此时未认证所以为空
		this.principal = principal;
		this.credentials = credentials;
		setAuthenticated(false); // 是否已认证
	}
```
调用ProviderManager.authenticate(authRequest)进行认证
```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		Class<? extends Authentication> toTest = authentication.getClass(); // UsernamePasswordAuthenticationToken
		AuthenticationException lastException = null;
		AuthenticationException parentException = null;
		Authentication result = null;
		Authentication parentResult = null;
		boolean debug = logger.isDebugEnabled();

		for (AuthenticationProvider provider : getProviders()) {
			if (!provider.supports(toTest)) {
				continue; // 遍历内部维护的认证提供者 不支持则跳过
			}

			//...
			try {
				result = provider.authenticate(authentication);

				if (result != null) {
					copyDetails(authentication, result);
					break;
				}
			}
			catch (AccountStatusException e) {
				//...
			}
		}

		if (result == null && parent != null) {
			// Allow the parent to try.
			try {
				result = parentResult = parent.authenticate(authentication); // 如果有父级AuthenticationManager 调用父级的authenticate方法认证
			}
			catch (ProviderNotFoundException e) {
				//...
			}
		}

		if (result != null) {
			if (eraseCredentialsAfterAuthentication
					&& (result instanceof CredentialsContainer)) {
				// Authentication is complete. Remove credentials and other secret data
				// from authentication
				((CredentialsContainer) result).eraseCredentials(); // 擦除密码信息
			}

			// If the parent AuthenticationManager was attempted and successful than it will publish an AuthenticationSuccessEvent
			// This check prevents a duplicate AuthenticationSuccessEvent if the parent AuthenticationManager already published it
			if (parentResult == null) {
				eventPublisher.publishAuthenticationSuccess(result); // 发布
			}
			return result;
		}

		//...
	}
```

ProviderManager维护一个AuthenticationProvider集合
`private List<AuthenticationProvider> providers = Collections.emptyList();`
此外使用的Provider为DaoAuthenticationProvider
authenticate方法定义在父抽象类AbstractUserDetailsAuthenticationProvider中
```java
public Authentication authenticate(Authentication authentication)
			throws AuthenticationException {
		Assert.isInstanceOf(UsernamePasswordAuthenticationToken.class, authentication, // authentication必须是UsernamePasswordAuthenticationToken
				() -> messages.getMessage(
						"AbstractUserDetailsAuthenticationProvider.onlySupports",
						"Only UsernamePasswordAuthenticationToken is supported"));

		// Determine username
		String username = (authentication.getPrincipal() == null) ? "NONE_PROVIDED"
				: authentication.getName();
		// 先尝试从缓存中获取
		boolean cacheWasUsed = true;
		UserDetails user = this.userCache.getUserFromCache(username);

		if (user == null) {
			cacheWasUsed = false;

			try {
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication);<1>
			}
			catch (UsernameNotFoundException notFound) {
				//...
			}

			//...
		}

		try {
			// 前置检查
			preAuthenticationChecks.check(user); // 检查账号是否过期是否冻结是否可用
			additionalAuthenticationChecks(user,
					(UsernamePasswordAuthenticationToken) authentication); // 检查密码是否正确
		}
		catch (AuthenticationException exception) {
			if (cacheWasUsed) {
				// There was a problem, so try again after checking
				// we're using latest data (i.e. not from the cache)
				cacheWasUsed = false;
				user = retrieveUser(username,
						(UsernamePasswordAuthenticationToken) authentication); 
				preAuthenticationChecks.check(user);
				additionalAuthenticationChecks(user,
						(UsernamePasswordAuthenticationToken) authentication);
			}
			else {
				throw exception;
			}
		}
		// 后置检查
		postAuthenticationChecks.check(user);

		if (!cacheWasUsed) {
			this.userCache.putUserInCache(user);
		}

		Object principalToReturn = user;

		if (forcePrincipalAsString) {
			principalToReturn = user.getUsername();
		}

		return createSuccessAuthentication(principalToReturn, authentication, user);<2>
	}
```

<1>调用DaoAuthenticationProvider中的retrieveUser方法
```java
protected final UserDetails retrieveUser(String username,
			UsernamePasswordAuthenticationToken authentication)
			throws AuthenticationException {
		prepareTimingAttackProtection();
		try {
			// 注意此处调用UserDetailsService的loadUserByUsername方法查找用户信息返回UserDetails
			UserDetails loadedUser = this.getUserDetailsService().loadUserByUsername(username);
			if (loadedUser == null) {
				throw new InternalAuthenticationServiceException(
						"UserDetailsService returned null, which is an interface contract violation");
			}
			return loadedUser;
		}
		catch (UsernameNotFoundException ex) {
			//...
		}
	}
```
<2>调用DaoAuthenticationProvider中的createSuccessAuthentication方法
```java
protected Authentication createSuccessAuthentication(Object principal,
			Authentication authentication, UserDetails user) {
		boolean upgradeEncoding = this.userDetailsPasswordService != null
				&& this.passwordEncoder.upgradeEncoding(user.getPassword());
		if (upgradeEncoding) {
			String presentedPassword = authentication.getCredentials().toString();
			String newPassword = this.passwordEncoder.encode(presentedPassword);
			user = this.userDetailsPasswordService.updatePassword(user, newPassword);
		}
		return super.createSuccessAuthentication(principal, authentication, user);
	}
```
再调用AbstractUserDetailsAuthenticationProvider中的createSuccessAuthentication
```java
protected Authentication createSuccessAuthentication(Object principal,
			Authentication authentication, UserDetails user) {
		// Ensure we return the original credentials the user supplied,
		// so subsequent attempts are successful even with encoded passwords.
		// Also ensure we return the original getDetails(), so that future
		// authentication events after cache expiry contain the details
		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
				principal, authentication.getCredentials(),
				authoritiesMapper.mapAuthorities(user.getAuthorities()));
		result.setDetails(authentication.getDetails());

		return result;
	}
```
此时会再创建UsernamePasswordAuthenticationToken对象 并最终返回到最外层的doFilter方法 最后调用successfulAuthenticationy方法将此对象设置到本地线程中
使用构造器如下
```java
public UsernamePasswordAuthenticationToken(Object principal, Object credentials,
			Collection<? extends GrantedAuthority> authorities) {
		super(authorities);
		this.principal = principal;
		this.credentials = credentials;
		super.setAuthenticated(true); // must use super, as we override
	}
```

认证成功后
```java
protected void successfulAuthentication(HttpServletRequest request,
			HttpServletResponse response, FilterChain chain, Authentication authResult)
			throws IOException, ServletException {

		if (logger.isDebugEnabled()) {
			logger.debug("Authentication success. Updating SecurityContextHolder to contain: "
					+ authResult);
		}

		SecurityContextHolder中.getContext().setAuthentication(authResult);

		rememberMeServices.loginSuccess(request, response, authResult);

		// Fire event
		if (this.eventPublisher != null) {
			eventPublisher.publishEvent(new InteractiveAuthenticationSuccessEvent(
					authResult, this.getClass()));
		}

		successHandler.onAuthenticationSuccess(request, response, authResult);
	}
```
SecurityContext的唯一实现类SecurityContextImpl中只有一个Authentication属性代表认证的用户信息
SecurityContextHolder中有一个SecurityContextHolderStrategy属性代表持有SecurityContext的策略 此处为ThreadLocalSecurityContextHolderStrategy 即本地线程持有策略
从SecurityContextHolder中getContext调用的是SecurityContextHolderStrategy的getContext即从本地线程中获取
第一次获取会先创建一个空的
```java
if (ctx == null) {
		ctx = createEmptyContext();
		contextHolder.set(ctx);
	}
```
然后再设置进已认证的对象，
在一次请求中可以从SecurityContextHolder中获取到SecurityContext从而获取到Authentication