---
title: SpringEvent应用
date: 2024-04-29 10:32:14
tags: spring event
categories: Spring+
---
让复杂业务解耦，让代码同时保持健壮与优雅是每个coder不懈的追求。或许你会直接上消息队列，可是
<!--more-->

# Spring event

# 一  使用

基本使用很简单，定义事件A，注册一个对A的监听器，然后发布这个事件。示意图如下：

​![image](/images/assets/image-20240430164433-bng4qnr.png)​

具体代码和步骤：

1. 定义事件

    ```java
    import org.springframework.context.ApplicationEvent;
    public class MyEvent extends ApplicationEvent {
        private String name;

        public MyEvent(Object source, String name) {
            super(source);
            this.name = name;
        }
    }
    ```
2. 注册一个监听器

    ```java
    @Component
    public class MyEventListener implements ApplicationListener<MyEvent> {
        @Override
        public void onApplicationEvent(MyEvent event) {
            System.out.println("event = " + event);
        }
    }
    ```
3. 定义自己的上下文

    ```java
    import org.springframework.beans.BeansException;
    import org.springframework.context.ApplicationContext;
    import org.springframework.context.ApplicationContextAware;
    import org.springframework.stereotype.Component;

    @Component
    public class ApplicationContextProvider implements ApplicationContextAware {

        private static ApplicationContext context;

        @Override
        public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
            context = applicationContext;
        }

        public static ApplicationContext getContext() {
            return context;
        }
    }
    ```

4. 发布事件

    ```java
    // 在需要调用的地方
     ApplicationContextProvider.getApplicationContext().publishEvent(new MyEvent("Source", "enventName"));
    ```

# 二 源码分析

springEvent使用基本可以两步， 注册 + 发布，核心代码其实都在 ApplicationEventMulticaster 类

关键类：

1. ApplicationListener （监听）
2. ApplicationEventPublisher （发布）
3. ApplicationEventMulticaster （管理）

‍

### 注册

基本有两种便利的实现方式：1，实现实现ApplicationListener接口 ； 2，定义Bean中的方法，并使用EventListener注解

* 实现ApplicationListener方式注册

  ```java
  // org.springframework.context.support.ApplicationListenerDetector#postProcessAfterInitialization
  // 后置处理所有的ApplicationListener
  	@Override
  	public Object postProcessAfterInitialization(Object bean, String beanName) {
  		if (bean instanceof ApplicationListener) {
  			Boolean flag = this.singletonNames.get(beanName);
  			if (Boolean.TRUE.equals(flag)) {
  				// 注册，最终调用是 this.applicationEventMulticaster.addApplicationListener(listener)
  				// ApplicationEventMulticaster是SpringEvent的管理类
  				this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
  			}
  			else if (Boolean.FALSE.equals(flag)) {
  				// 不支持非单例Bean
  			}
  		}
  		return bean;
  	}
  ```

‍

* 基于注解@EventListener的注册

  ```java

  // org.springframework.context.event.EventListenerMethodProcessor#processBean

  	private void processBean(final String beanName, final Class<?> targetType) {
  		// 获取有EventListener注解的方法
  		Map<Method, EventListener> annotatedMethods = MethodIntrospector.selectMethods(targetType,
  		(MethodIntrospector.MetadataLookup<EventListener>) method -> AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
  		ConfigurableApplicationContext context = this.applicationContext;
  		// 所有
  		List<EventListenerFactory> factories = this.eventListenerFactories;
  		for (Method method : annotatedMethods.keySet()) {
  			for (EventListenerFactory factory : factories) {
  				if (factory.supportsMethod(method)) {
  					Method methodToUse = AopUtils.selectInvocableMethod(method, context.getType(beanName));
  					// 将方法封装成ApplicationListenerMethodAdapter
  					ApplicationListener<?> applicationListener =
  							factory.createApplicationListener(beanName, targetType, methodToUse);
  					if (applicationListener instanceof ApplicationListenerMethodAdapter) {
  						((ApplicationListenerMethodAdapter) applicationListener).init(context, this.evaluator);
  					}
  					// 注册到ApplicationContext，同上面使用方法注解一致
  					context.addApplicationListener(applicationListener);
  					break;
  				}
  			}
  		}
  	}

  ```

‍

### 发布

```java
// 发布调用
// org.springframework.context.support.AbstractApplicationContext#publishEvent(java.lang.Object, org.springframework.core.ResolvableType)
	// 核心是调用ApplicationEventMulticaster的多播方法
	getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);

// org.springframework.context.event.SimpleApplicationEventMulticaster#multicastEvent(org.springframework.context.ApplicationEvent, org.springframework.core.ResolvableType)

	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		// 事件类型转换为ResolvableType 
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		// 如果不指定执行器，那么就是顺序执行，可能会阻塞
		// 可以自定义SimpleApplicationEventMulticaster，通过线程池来异步执行
		Executor executor = getTaskExecutor();
		// 通过事件类型寻找监听器，有缓存优化
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

‍

# 三、注意事项

1. 可以从源码看到，SpringEvent是依赖SpringIOC的，在启动（init）和关闭（destroy）阶段，不要使用
2. 如果不指定事件类型，会接受都爱全部事件。SpringBoot中自有事件依照启动顺序依次有：

    1. ServletWebServerInitializedEvent
    2. ContextRefreshedEvent
    3. ApplicationStartedEvent
    4. AvailabilityChangeEvent
    5. ApplicationReadyEvent
3. SpringEvent默认顺序执行，如果外层有事务，Event中异常可能会导致事务回滚。因此建议使用线程池来异步执行事件。自定义线程池方法：

    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.context.annotation.Configuration;
    import org.springframework.context.event.SimpleApplicationEventMulticaster;
    import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

    @Configuration
    public class EventConfig {
        @Bean
        SimpleApplicationEventMulticaster applicationEventMulticaster() {
            SimpleApplicationEventMulticaster multicaster = new SimpleApplicationEventMulticaster();
            ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
            //核心线程池数量
            executor.setCorePoolSize(Runtime.getRuntime().availableProcessors());
            //最大线程数量
            executor.setMaxPoolSize(Runtime.getRuntime().availableProcessors() * 5);
            //线程池的队列容量
            executor.setQueueCapacity(Runtime.getRuntime().availableProcessors() * 2);
            //线程名称的前缀
            executor.setThreadNamePrefix("springEvent-executor-");
            executor.initialize();
            multicaster.setTaskExecutor(executor);
            return multicaster;
        }
    }
    ```

4. SpringEvent适用于最终一致性，如果需要重试，推荐[Spring-retry](https://springdoc.cn/spring-retry-guide/)

‍