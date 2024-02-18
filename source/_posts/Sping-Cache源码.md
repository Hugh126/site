---
title: Sping Cache源码
date: 2023-04-04 10:27:52
tags: 
    - spring cache
    - spring增强
categories: Spring+
---
我们常说读源码一定要带着问题来读。那么你的疑问又是哪些呢？  
就与我而言，在大概明白SpringCache的使用方式后，我想知道  
1. @EnableCaching 这个注解到底干了什么事情  
2. @Cacheable这种方法上的注解，是怎么替换我方法的返回的  
3. 我想对SpringCache有个更深入一层的理解，后续如果有问题，我知道在哪里去切入。
<!--more-->

## EnableCaching注释解读
看源码一定要看注释，注释开头说明了这个注解的作用类似xml中
``` xml
 <cache:annotation-driven/>
```  
它们负责负责注册必要的Spring组件，提供缓存管理，例如`CacheInterceptor`和基于代理的增强。  
然后就给出了很清晰的使用方法：
``` java
 @Configuration
   @EnableCaching
   public class AppConfig {
  
       @Bean
       public MyService myService() {
           // configure and return a class having @Cacheable methods
           return new MyService();
       }
  
       @Bean
       public CacheManager cacheManager() {
           // configure and return an implementation of Spring's CacheManager SPI
           SimpleCacheManager cacheManager = new SimpleCacheManager();
           cacheManager.setCaches(Arrays.asList(new ConcurrentMapCache("default")));
           return cacheManager;
       }
   }
```
CacheManager必须指定，因为居然没有提供默认的Cache。CacheManager是接口，它的实现类就是可支持的Cache实现。通过配置就可以支持的Cache类型可以参考`CacheType`,可以通过spring.cache.type指定。  
为了更精确的使用Cache，可以自定义`CachingConfigurer`的实现，从而指定自定义的 CacheManager, CacheResolver, KeyGenerator, CacheErrorHandler。如果CacheManager和CacheResolver同时指定了，那么CacheManager将被忽略。然后，配置类必须要纳入Spring的Bean管理，推荐使用@Configuration，然后继承 CachingConfigurerSupport 。打开一看，居然是四个类型的空实现，而注释里面声明只有CacheManager必须要指定。那么，必然存在一个位置给他们赋默认值（后面解答）。然后，缓存通知的模式默认为AdviceMode.PROXY，它只支持基于Bean对象的调用，不支持本地方法的调用。毕竟是基于Spring的AOP，跟事务拦截类似。  

## EnableCaching还做了什么
通过`@Import(CachingConfigurationSelector.class)`可以看到Import了一个增强类，其中selectImports方法定义了如果使用`proxy`代理的话，又另外引入了两个类：`AutoProxyRegistrar`和`ProxyCachingConfiguration`。  
其中 AutoProxyRegistrar 的作用是根据当前BeanDefinitionRegistry适当地注册自动代理创建者。参考  
`AopConfigUtils.registerAutoProxyCreatorIfNecessary`
而 ProxyCachingConfiguration 则是注册基于Spring注解的缓存管理的必要类。这是一个核心的类。  
``` java
@Configuration(proxyBeanMethods = false)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

	// 创建一个基于CacheOperationSource可访问bean工厂的增强器
	@Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor(
			CacheOperationSource cacheOperationSource, CacheInterceptor cacheInterceptor) {

		BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
		advisor.setCacheOperationSource(cacheOperationSource);
		advisor.setAdvice(cacheInterceptor);
		if (this.enableCaching != null) {
			advisor.setOrder(this.enableCaching.<Integer>getNumber("order"));
		}
		return advisor;
	}


	// 创建一个默认的AnnotationCacheOperationSource，支持带有Cacheable和CacheEvict注解的公共方法
	// 核心是定义getCacheOperations
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}

	// 使用策略设计模式, 本身只负责调用顺序, CacheAspectSupport 完成具体操作的定义。CacheOperationSource用于确定缓存操作，KeyGenerator将构建缓存键，CacheResolver将解析要使用的实际缓存
	// 前面说了默认组件也在这里制定，分别是 SimpleCacheResolver.of(SupplierUtils.resolve(cacheManager)
	// SimpleKeyGenerator SimpleCacheErrorHandler ,注意这里cacheResolver默认的指定仍然来自于CacheManager，所以再次说明CacheManager必须有
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor(CacheOperationSource cacheOperationSource) {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		interceptor.setCacheOperationSource(cacheOperationSource);
		return interceptor;
	}

}
```

## 缓存代理是怎么执行的  
之前在分析相关类的时候就说明了真正缓存操作都是`CacheInterceptor`这里面完成的，这里只有一个`invoke`方法，断点跟进一下。  
``` java
public Object invoke(final MethodInvocation invocation) throws Throwable {
		// invocation 其实是一个cglib的代理类org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation ，指向的是有Cacheable的方法
		Method method = invocation.getMethod();
		
		// 定义一个缓存操作执行方法
		CacheOperationInvoker aopAllianceInvoker = () -> {
			try {
				return invocation.proceed();
			}
			catch (Throwable ex) {
				throw new CacheOperationInvoker.ThrowableWrapper(ex);
			}
		};

		Object target = invocation.getThis();
		Assert.state(target != null, "Target must not be null");
		try {
			// 通过代理对象及方法，执行缓存获取操作
			return execute(aopAllianceInvoker, target, method, invocation.getArguments());
		}
		catch (CacheOperationInvoker.ThrowableWrapper th) {
			throw th.getOriginal();
		}
	}
```
跟下去发现，真正起作用的地方还是`CacheAspectSupport`的excute方法：  
``` java
private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
		// Special handling of synchronized invocation
		// 同步调用特殊处理
		if (contexts.isSynchronized()) {
			CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
			if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
				Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
				Cache cache = context.getCaches().iterator().next();
				try {
					return wrapCacheValue(method, handleSynchronizedGet(invoker, key, cache));
				}
				catch (Cache.ValueRetrievalException ex) {
					// Directly propagate ThrowableWrapper from the invoker,
					// or potentially also an IllegalArgumentException etc.
					ReflectionUtils.rethrowRuntimeException(ex.getCause());
				}
			}
			else {
				// No caching required, only call the underlying method
				return invokeOperation(invoker);
			}
		}

		// Process any early evictions
		// 处理beforeInvocation
		processCacheEvicts(contexts.get(CacheEvictOperation.class), true,
				CacheOperationExpressionEvaluator.NO_RESULT);

		// Check if we have a cached item matching the conditions
		// 就是这里从缓存中获取对象，实际处理了注解中condition、key
		Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

		// Collect puts from any @Cacheable miss, if no cached item is found
		// 判断unless
		List<CachePutRequest> cachePutRequests = new ArrayList<>();
		if (cacheHit == null) {
			collectPutRequests(contexts.get(CacheableOperation.class),
					CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
		}

		Object cacheValue;
		Object returnValue;

		if (cacheHit != null && !hasCachePut(contexts)) {
			// If there are no put requests, just use the cache hit
			cacheValue = cacheHit.get();
			returnValue = wrapCacheValue(method, cacheValue);
		}
		else {
			// Invoke the method if we don't have a cache hit
			returnValue = invokeOperation(invoker);
			cacheValue = unwrapReturnValue(returnValue);
		}

		// Collect any explicit @CachePuts
		collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

		// Process any collected put requests, either from @CachePut or a @Cacheable miss
		// 处理其他CachePut
		for (CachePutRequest cachePutRequest : cachePutRequests) {
			cachePutRequest.apply(cacheValue);
		}

		// Process any late evictions
		// 处理condition
		processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);

		return returnValue;
	}
```

## 总结 
其实，基于以上内容，Spring Cache的大致缓存处理的代码已经涉及到了。 但是，要理解代理是怎么环绕增强的，那么久需要AspectJ相关的知识了。其实，切点的相关概念也是Spring从AspectJ中引入的，然后结合了自身的Spring的Bean管理形成了SpringAop。可以参考：  

https://www.baeldung.com/spring-aop  

https://docs.spring.io/spring-framework/docs/3.2.x/spring-framework-reference/html/aop.html  

发现自己其实对Spring的AOP和AspectJ缺乏更清晰的理解，后面再整理一到两篇跟进一下吧。




