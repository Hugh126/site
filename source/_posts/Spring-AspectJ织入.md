---
title: Spring-AspectJ织入
date: 2023-04-06 14:05:35
tags:
- spring-aop
- AspectJ
categories: Spring+
---
最开始用Spring切面的时候听说是基于AspectJ的，便认为SpringAop是依赖AspectJ实现的。其实不然，SpringAop其实不依赖AspectJ，它本身的AOP功能是基于SpringBean管理的，通过动态代理(JDK或CgLib)实现的。然而，这却不是一个完整的AOP解决方案，所以需要引入AspectJ。
<!--more-->  
# 一 AOP相关概念  
- Aspect	切面
- Pointcut 切点，织入Advice的触发条件，也称为Joinpoint
- Advice 增强，切面中切点处的具体行为
    - Around	环绕增强，目标方法执行前后分别执行一些代码
    - AfterReturning	返回增强，目标方法正常执行完毕时执行
    - Before	前置增强，目标方法执行之前执行
    - AfterThrowing	异常抛出增强，目标方法发生异常的时候执行
    - After	后置增强，不管是抛出异常或者正常退出都会执行  
- Weaving 织入，在切点处执行增强的过程

# 二 织入时期  
- Compile-time weaving  
    > 编译期织入，编译的时候织入代码，运行时直接运行 
- Post-compile weaving
    > 编译期后织入，编译后二进制织入，一般此时织入的是已存在的class或者jar
- Load-time weaving
    > 加载期织入，也是二进制织入，但此时是加载到JVM之前  

# 三 SpringAOP和AspectJ差异
通过以上所述，SpringAop是通过动态代理在运行时织入，而AspectJ是通过字节码技术在编译期（或加载期）织入。用代码抽象表述了下，方便理解：  
``` java
class A{
	methodA{
	// do some thing
	}
}

class AProxy-SpringAop{

	doMethodA {
		// advice before
		methodA();
		// advice after
	}

	methodA{
	// do some thing
	}
}

class AProxy-AspectJ{
	methodA{
	// advice before
	// do some thing
	// advice after
	}
}
```
更多关于两者差异和关联可参考：  
https://www.baeldung.com/spring-aop-vs-aspectj  

https://docs.spring.io/spring-framework/docs/4.3.15.RELEASE/spring-framework-reference/html/aop.html  

# 四 解决SpringAOP方法内调用失效  
用SpringAop不论是完成异步调用、事务、缓存操作，都不能避免内部方法调用失效的尴尬。可以参考的解决方案有：  
- AopContext.currentProxy() 获取当前代理对象再调用目标方法  
- @Autowired self； 通过注入自身来调用  
- 把方法2拆分到新的Bean中，避免类方法调用  

# 五 加载时织入(LTW)  

大致方法是将aspectjweaver.jar作为agent传入JVM，在加载时织入相应Advice。下面以SpringCache中使用AspectJ为例（事务和异步都是一样的操作）:  
1. 新增依赖  
``` xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-aop</artifactId>
    </dependency>
    <dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-aspects</artifactId>
    </dependency>
``` 

2. 修改代理模式  
``` java
@EnableCaching(mode = AdviceMode.ASPECTJ)
``` 

3. 传入agent  
``` bat
# 启动参数新增agent
-javaagent:D:\Tools\aspectjweaver-1.9.19.jar
```

更多关于Spring中使用LTW的可以参考：  
https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#aop-aj-ltw


 
