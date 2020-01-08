---
title: Spring事件监听源码解析
categories: 
  - Spring
tags: 
  - event
---

Spring事件监听机制离不开容器IOC特性提供的支持,比如容器会自动创建事件发布器,自动识别用户注册的监听器并进行管理,在特定的事件发布后会找到对应的事件监听器并对其监听方法进行回调。Spring帮助用户屏蔽了关于事件监听机制背后的很多细节,使用户可以专注于业务层面进行自定义事件开发。然而我们还是忍不住对其背后的实现原理进行一番探讨,比如:
1. 事件发布器ApplicationEventMulticaster是何时被初始化的,初始化过程中都做了什么？
2. 注册事件监听器的过程是怎样的,容器怎么识别出它们并进行管理?
3. 容器发布事件的流程是怎样的?它如何根据发布的事件找到对应的事件监听器,事件和由该事件触发的监听器之间的匹配规则是怎样的？

### 初始化事件发布器流程
真正的事件发布器是ApplicationEventMulticaster,它定义在AbstractApplicationContext中,并在ApplicationContext容器启动的时候进行初始化。在容器启动的refrsh()方法中可以找到初始化事件发布器的入口方法,如下<2>所示
```java
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// Prepare this context for refreshing.
		prepareRefresh();

		// Tell the subclass to refresh the internal bean factory.
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);<1>
			// Allows post-processing of the bean factory in context subclasses.
			postProcessBeanFactory(beanFactory);

			// Invoke factory processors registered as beans in the context.
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			initMessageSource();

			// Initialize event multicaster for this context.
			initApplicationEventMulticaster();<2>

			// Initialize other special beans in specific context subclasses.
			onRefresh();

			// Check for listener beans and register them.
			registerListeners();<3>

			// Instantiate all remaining (non-lazy-init) singletons.
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			finishRefresh();
		}
		catch (BeansException ex) {
			//...
		}
		finally {
			//...
		}
	}
}
```
```java
protected void initApplicationEventMulticaster() {
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		if (logger.isTraceEnabled()) {
			logger.trace("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
		}
	}
	else {
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		if (logger.isTraceEnabled()) {
			logger.trace("No '" + APPLICATION_EVENT_MULTICASTER_BEAN_NAME + "' bean, using " +
					"[" + this.applicationEventMulticaster.getClass().getSimpleName() + "]");
		}
	}
}
```
这里会根据核心容器beanFactory中是否有id为applicationEventMulticaster的bean分两种情况:

1. 容器中已有id为applicationEventMulticaster的bean直接从容器缓存获取或是创建该bean实例,并交由成员变量applicationEventMulticaster保存。当用户自定义了事件发布器并向容器注册时会执行该流程。
2. 容器中不存在applicationEventMulticaster的bean,这是容器默认的执行流程,会创建一个SimpleApplicationEventMulticaster,其仅在实现事件发布器基本功能(管理事件监听器以及发布容器事件)的前提下,增加了可以设置任务执行器Executor和错误处理器ErrorHandler的功能,当设置Executor为线程池时,则会以异步的方式对事件监听器进行回调,而ErrorHandler允许我们在回调方法执行错误时进行自定义处理。默认情况下，这两个变量都为null。
```java
public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {

	@Nullable
	private Executor taskExecutor;

	@Nullable
	private ErrorHandler errorHandler;
```
```java
public abstract class AbstractApplicationEventMulticaster
		implements ApplicationEventMulticaster, BeanClassLoaderAware, BeanFactoryAware {

	private final ListenerRetriever defaultRetriever = new ListenerRetriever(false);

	final Map<ListenerCacheKey, ListenerRetriever> retrieverCache = new ConcurrentHashMap<>(64);

	@Nullable
	private ClassLoader beanClassLoader;

	@Nullable
	private BeanFactory beanFactory;

	private Object retrievalMutex = this.defaultRetriever;
```
之后会调用beanFactory.registerSingleton方法将创建的SimpleApplicationEventMulticaster实例注册为容器的单实例bean。
初始化事件发布器的工作非常简单,一句话总结:由容器实例化用户自定义的事件发布器或者由容器帮我们创建一个简单的事件发布器并交由容器管理。

### 注册事件监听器流程
注册事件监听器的流程在初始化事件发布器之后<3>,其代码如下所示
```java
protected void registerListeners() {
		// Register statically specified listeners first.
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```
首先遍历beanFactory中所有的bean,获取所有实现ApplicationListener接口的bean的beanName,并将这些beanName注册到ApplicationEventMulticaster中
```java
@Override
	public void addApplicationListenerBean(String listenerBeanName) {
		synchronized (this.retrievalMutex) {
			this.defaultRetriever.applicationListenerBeans.add(listenerBeanName);
			this.retrieverCache.clear();
		}
	}
```
defaultRetriever是定义在抽象类AbstractApplicationEventMulticaster中的成员,用来保存所有事件监听器及其beanName
`private final ListenerRetriever defaultRetriever = new ListenerRetriever(false);`
其以内部类形式定义在AbstractApplicationEventMulticaster中
```java
private class ListenerRetriever {

	public final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();

	public final Set<String> applicationListenerBeans = new LinkedHashSet<>();

	private final boolean preFiltered;

	public ListenerRetriever(boolean preFiltered) {
		this.preFiltered = preFiltered;
	}
```
向事件发布器注册监听器的beanName,其实就是将beanName加入ListenerRetriever的集合中。
其实看到这里会有一个疑问,registerListeners()方法只是找到了所有监听器的beanName并将其保存到了事件发布器ApplicationEventMulticaster中,那么真正的事件监听器实例是何时被创建并被加入到事件发布器中的？
这里我们不得不退回到启动容器的refresh()方法中,在初始化beanFactory之后,初始化事件发布器之前,容器在prepareBeanFactory(beanFactory)<1>方法中又注册了一些重要组件,其中就包括一个特殊的BeanPostProcessor:ApplicationListenerDetector,正如其类名暗示的那样,这是一个事件监听器的探测器。
```java
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
	// Tell the internal bean factory to use the context's class loader etc.
	beanFactory.setBeanClassLoader(getClassLoader());
	beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
	beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

	// Configure the bean factory with context callbacks.
	beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
	beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
	beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
	beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
	beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
	beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

	// BeanFactory interface not registered as resolvable type in a plain factory.
	// MessageSource registered (and found for autowiring) as a bean.
	beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
	beanFactory.registerResolvableDependency(ResourceLoader.class, this);
	beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
	beanFactory.registerResolvableDependency(ApplicationContext.class, this);

	// Register early post-processor for detecting inner beans as ApplicationListeners. 注册探测器
	beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

	// Detect a LoadTimeWeaver and prepare for weaving, if found.
	if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
		beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
		// Set a temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
	}
```
ApplicationListenerDetector实现了BeanPostProcessor接口,可以在容器级别对所有bean的生命周期过程进行增强。这里主要是为了能够在初始化所有bean后识别出所有的事件监听器bean并
将其注册到事件发布器中
```java
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
	if (bean instanceof ApplicationListener) {
		// potentially not detected as a listener by getBeanNamesForType retrieval
		Boolean flag = this.singletonNames.get(beanName);
		if (Boolean.TRUE.equals(flag)) {
			// singleton bean (top-level or inner): register on the fly
			this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
		}
		else if (Boolean.FALSE.equals(flag)) {
			if (logger.isWarnEnabled() && !this.applicationContext.containsBean(beanName)) {
				// inner bean with other scope - can't reliably process events
				logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
						"but is not reachable for event multicasting by its containing ApplicationContext " +
						"because it does not have singleton scope. Only top-level listener beans are allowed " +
						"to be of non-singleton scope.");
			}
			this.singletonNames.remove(beanName);
		}
	}
	return bean;
}
```
在初始化所有的bean后,该ApplicationListenerDetector的postProcessAfterInitialization(Object bean, String beanName)方法会被作用在每一个bean上,通过判断传入的bean
是否是ApplicationListener实例进行过滤,然后将找到的事件监听器bean注册到事件发布器中。

### 容器事件发布流程
```java
public void publishEvent(Object event) {
	publishEvent(event, null);
}
```
```java
protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
	Assert.notNull(event, "Event must not be null");

	// Decorate event as an ApplicationEvent if necessary
	ApplicationEvent applicationEvent;
	if (event instanceof ApplicationEvent) {
		applicationEvent = (ApplicationEvent) event;
	}
	else {
		applicationEvent = new PayloadApplicationEvent<>(this, event);
		if (eventType == null) {
			eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
		}
	}

	// Multicast right now if possible - or lazily once the multicaster is initialized
	if (this.earlyApplicationEvents != null) {
		this.earlyApplicationEvents.add(applicationEvent);
	}
	else {
		getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType); <4>
	}

	// Publish event via parent context as well...
	if (this.parent != null) {
		if (this.parent instanceof AbstractApplicationContext) {
			((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
		}
		else {
			this.parent.publishEvent(event);
		}
	}
}
```
这里为了简化源码阅读的工作量,对一些细节和分支情形做了忽略,只考虑主流程,如上<4>所示,这里调用了事件发布器的multicastEvent()方法进行事件发布,需要传入事件event和事件类型
eventType作为参数。不过通常这个eventType参数为null,因为事件的类型信息完全可以通过反射的方式从event对象中获得。继续跟进源码
```java
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
	ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
	Executor executor = getTaskExecutor();
	for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
		if (executor != null) {
			executor.execute(() -> invokeListener(listener, event));
		}
		else {
			invokeListener(listener, event);
		}
	}
}
```
首先通过传入的参数或者通过调用resolveDefaultEventType(event)方法获取事件的事件类型信息,之后会通过
getApplicationListeners(event, type)方法得到所有和该事件类型匹配的事件监听器,其实现逻辑后面会细说,这里先跳过。对于匹配的每一个监听器,视事件发布器内是否设置了
任务执行器实例Executor,决定以何种方式对监听器的监听方法进行回调。
- 若执行器实例Executor未设置,则进行同步回调,即在当前线程执行监听器的回调方法
- 若用户设置了Executor实例(通常而言是线程池),则会进行异步回调,监听器的监听方法会交由线程池中的线程去执行。

默认情况下容器不会为用户创建执行器实例,因而对监听器的回调是同步进行的,即所有监听器的监听方法都在推送事件的线程中被执行,通常这也是处理业务逻辑的线程,若其中一个监听器回调执行
阻塞,则会阻塞整个业务处理的线程,造成严重的后果。而异步回调的方式,虽然不会导致业务处理线程被阻塞,但是不能共享一些业务线程的上下文资源,比如类加载器,事务等等。因而究竟选择哪种回调
方式,要视具体业务场景而定。

好了,现在可以来探究下困扰我们很久的一个问题了,那就是:如何根据事件类型找到匹配的所有事件监听器？这部分逻辑在getApplicationListeners方法中
```java
protected Collection<ApplicationListener<?>> getApplicationListeners(
			ApplicationEvent event, ResolvableType eventType) {

	Object source = event.getSource();
	Class<?> sourceType = (source != null ? source.getClass() : null);
	ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);

	// Quick check for existing entry on ConcurrentHashMap...
	ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
	if (retriever != null) {
		return retriever.getApplicationListeners();
	}

	if (this.beanClassLoader == null ||
			(ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
					(sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
		// Fully synchronized building and caching of a ListenerRetriever
		synchronized (this.retrievalMutex) {
			retriever = this.retrieverCache.get(cacheKey);
			if (retriever != null) {
				return retriever.getApplicationListeners();
			}
			retriever = new ListenerRetriever(true);
			Collection<ApplicationListener<?>> listeners =
					retrieveApplicationListeners(eventType, sourceType, retriever);
			this.retrieverCache.put(cacheKey, retriever); <7>
			return listeners;
		}
	}
	else {
		// No ListenerRetriever caching -> no synchronization necessary
		return retrieveApplicationListeners(eventType, sourceType, null);
	}
}
```
整个流程可以概括为：
1. 首先从缓存map中查找,这个map定义在事件发布器的抽象类中
`final Map<ListenerCacheKey, ListenerRetriever> retrieverCache = new ConcurrentHashMap<>(64);`
ListenerCacheKey由事件类型eventType和事件源类型sourceType构成,ListenerRetriever内部则维护了一个监听器列表。当所发布的事件类型和事件源类型与Map中的key匹配时,将直接返回value中的监听器列表作为匹配结果,通常这发生在事件不是第一次发布时,能避免遍历所有监听器并进行过滤,如果事件时第一次发布,则会执行流程2。
2. 遍历所有的事件监听器,并根据事件类型和事件源类型进行匹配。
```java
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
			ResolvableType eventType, @Nullable Class<?> sourceType, @Nullable ListenerRetriever retriever) {

	List<ApplicationListener<?>> allListeners = new ArrayList<>();
	Set<ApplicationListener<?>> listeners;
	Set<String> listenerBeans;
	synchronized (this.retrievalMutex) {
		listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
		listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
	}
	for (ApplicationListener<?> listener : listeners) {
		if (supportsEvent(listener, eventType, sourceType)) {
			if (retriever != null) {
				retriever.applicationListeners.add(listener);
			}
			allListeners.add(listener);
		}
	}
	if (!listenerBeans.isEmpty()) {
		BeanFactory beanFactory = getBeanFactory();
		for (String listenerBeanName : listenerBeans) {
			try {
				Class<?> listenerType = beanFactory.getType(listenerBeanName);
				if (listenerType == null || supportsEvent(listenerType, eventType)) {
					ApplicationListener<?> listener =
							beanFactory.getBean(listenerBeanName, ApplicationListener.class);
					if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
						if (retriever != null) {
							if (beanFactory.isSingleton(listenerBeanName)) {
								retriever.applicationListeners.add(listener);
							}
							else {
								retriever.applicationListenerBeans.add(listenerBeanName);
							}
						}
						allListeners.add(listener);
					}
				}
			}
			catch (NoSuchBeanDefinitionException ex) {
				// Singleton listener instance (without backing bean definition) disappeared -
				// probably in the middle of the destruction phase
			}
		}
	}
	AnnotationAwareOrderComparator.sort(allListeners);
	if (retriever != null && retriever.applicationListenerBeans.isEmpty()) {
		retriever.applicationListeners.clear();
		retriever.applicationListeners.addAll(allListeners);
	}
	return allListeners;
}
```
判断监听器是否匹配的逻辑在supportsEvent(listener, eventType, sourceType)中,
```java
protected boolean supportsEvent(
			ApplicationListener<?> listener, ResolvableType eventType, @Nullable Class<?> sourceType) {

	GenericApplicationListener smartListener = (listener instanceof GenericApplicationListener ?
			(GenericApplicationListener) listener : new GenericApplicationListenerAdapter(listener));
	return (smartListener.supportsEventType(eventType) && smartListener.supportsSourceType(sourceType));
}
```
首先对原始的ApplicationListener进行一层适配器包装成GenericApplicationListener,便于后面使用该接口中定义的方法判断监听器是否支持传入的事件类型或事件源类型
```java
public interface GenericApplicationListener extends ApplicationListener<ApplicationEvent>, Ordered {
	// 判断是否支持该事件类型
	boolean supportsEventType(ResolvableType eventType);

	// 判断是否支持该事件源类型
	default boolean supportsSourceType(@Nullable Class<?> sourceType) {
		return true; <5>
	}

	@Override
	default int getOrder() {
		return LOWEST_PRECEDENCE;
	}

}
```
smartListener.supportsEventType(eventType)用来判断监听器是否支持该事件类型,因为我们的监听器实例通常都不是SmartApplicationListener类型<6>
```java
@Override
@SuppressWarnings("unchecked")
public boolean supportsEventType(ResolvableType eventType) {
	if (this.delegate instanceof SmartApplicationListener) {
		Class<? extends ApplicationEvent> eventClass = (Class<? extends ApplicationEvent>) eventType.resolve();
		return (eventClass != null && ((SmartApplicationListener) this.delegate).supportsEventType(eventClass));
	}
	else {
		return (this.declaredEventType == null || this.declaredEventType.isAssignableFrom(eventType));<6>
	}
}
```
declaredEventType是监听器泛型的实际类型,而eventType是发布的事件的类型
declaredEventType.isAssignableFrom(eventType)当以下两种情况返回true
1. declaredEventType和eventType类型相同
2. declaredEventType是eventType的父类型
只要监听器泛型的实际类型和发布的事件类型一样或是它的父类型,则该监听器将被成功匹配。
而对于事件源类型而言,通常默认会直接返回true,也就是说事件源的类型通常对于判断匹配的监听器没有意义。<5>

这里查找到所有匹配的监听器返回后会将匹配关系缓存到retrieverCache这个map中<7>