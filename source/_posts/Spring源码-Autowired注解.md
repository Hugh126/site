---
title: Spring源码-Autowired注解
date: 2023-10-11 13:19:59
tags: 
---
古人说“绝知此事要躬行”，对于学习框架源码更是如此（本篇介绍源码阅读基础）
<!--more-->

# 一 新建干净的项目
一个较多业务逻辑的项目对于源码阅读初入门的人，或多或少有一些干扰。  

*从模板新建maven项目*    
![](/images/Autowired1.png)

*添加maven依赖*
``` xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-context</artifactId>
  <version>5.2.8.RELEASE</version>
</dependency>
```

*准备Bean*
``` java
// 新建类B
@Component
public class B {
}
// 新建类A，注入属性B
@Component
public class A  {
    @Autowired
    private B b;
    public void test() {
        System.out.println("b.info=" + b);
    }
}
```

*启动Main方法*
``` java
public class App 
{
    public static void main( String[] args )
    {
        AbstractApplicationContext context = new AnnotationConfigApplicationContext("org.example");
        context.getBean(A.class).test();
        context.close();
    }
}
```

# 二 Spring初始化流程认知  

## refresh流程
大致是如下流程:  
1. 注解类或xml配置文件读取为BeanDefinition
2. 创建BeanFactory
3. 通过BeanDefinition为Bean工厂添加各种后置处理
4. 遍历BeanDefinition中定义的BeanName，创建->初始化（填充属性、bean前置、自定义初始化、bean后置）  

曾经看过马士兵的一次讲课，觉得很清晰，这里借用一下原图：  
![](/images/Autowired2.png)

## Bean的生命周期  
参考BeanFactory接口注释  
![](/images/Autowired3.png)

## Bean工厂的默认实现  

Spring对ConfigurableListableBeanFactory和BeanDefinitionRegistry接口的默认实现:基于bean定义元数据的成熟bean工厂，可通过后处理器扩展。  

## 条件断点  
这里面设置条件断点：
DefaultListableBeanFactory.doCreateBean
    - createBeanInstance(beanName, mbd, args) // 创建Bean
    - populateBean(beanName, mbd, instanceWrapper) // 填充Bean属性
    	-  ibp.postProcessProperties  // 这一行，注意ibp实现类为AutowiredAnnotationBeanPostProcessor
  - metadata.inject(bean, beanName, pvs);
内部调用顺序：
  InjectionMetadata. inject -> AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement.inject


  # 三 源码Trace  

*在实例化和初始化位置打条件断点*  

![](/images/Autowired4.png)  
  
*在Bean的填充属性位置打条件断点，注意此处处理BP类型*  
![](/images/Autowired5.png) 

*获取到需要注入的元数据后，执行注入*  
![](/images/Autowired6.png) 


*然后是调用属性类的注入方法*  
注意这里是继承内部类后复写的方法 ，OMG
``` java
private class AutowiredFieldElement extends InjectionMetadata.InjectedElement
``` 
![](/images/Autowired7.png) 


*解决依赖 resolveDependency*  
![](/images/Autowired8.png)

*获取实例化对象b后，反射调用set注入*  
![](/images/Autowired9.png)


*递归调用b的属性填充*  
对于没有下级依赖的属性，在填属性（populateBean）的时候依旧会走到inject方法，但是它的对象是InjectionMetadata.EMPTY ，会直接返回  
![](/images/Autowired10.png)

# 四 总结  

Autowired的逻辑主要是在填充属性(populateBean)的时候，doResolveDependency进行处理的。  
1. 首先会通过findAutowireCandidates查找所有类型匹配的类， 如果找不到直接异常  
2. 如果有多个候选，那么determineAutowireCandidate会进行决定最终的候选，逻辑：
    1. 是否有@Primary来赋予bean更高的优先级
    2. 看是否实现了OrderComparator，通过更高优先级来匹配
    3. 通过给定的名称匹配  

--- 
在熟悉既有逻辑后，不妨思考一下以前学习Autowired自动装配的规则(优先类型匹配，然后名称匹配)：  
可以将类型变更为接口，或者Object，又或者改下默认的field名称，看是否可以匹配上。对应的源码又是怎么走的。  
