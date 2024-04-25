---
title: 分布式限流sentinel
date: 2024-04-24 18:00:37
tags:
- 限流
- sentinel
categories: 中间件
---
单机限流可以采用以令牌桶、漏桶算法为基础的诸多实现，在分布式限流一般使用的是网关产品附带的一些限流功能。而Sentinel，提供了丰富的限流、熔断、降级等功能，更是提供了控制台系统展示、灵活控制这些规则，相当强大！
<!--more-->
# 一、限流简述
限流定义：只允许指定数量的事件进入系统，超出部分将被拒绝服务、排队、降级等处理  
限流目标： 保证一部分请求流量可以得到正常响应，避免系统雪崩  
常说的限流，一般是单机限流。单机限流可以通过Guava的RateLimiter来实现，它使用的是令牌桶算法。  
分布式限流以前使用的是nginx的限流模块，限制并发连接和限制同一IP的访问频次。可是，在实际引用中还远远不够，一般需要通过LUA来加入一些更灵活实用的功能。  
Sentinel作为AlibabaSrpingCloud的一部分，承接了阿里巴巴近10年的双十一大促流量的核心场景，有丰富的应用场景以及监控+开源生态。个人使用来看，个人认为sentinel的优势在于，除了提供全面的限流降级的接入方式，更在于提供了控制台，便于服务本身接入并自动采集Metrics，以及基于Metics（QPS，链路，内存，CPU等）的各类限流熔断，这些才是真实场景中实用的。  

# 二、同类组件对比
​![image](/images/assets/image-20240425162917-mdlq5qj.png)




# 三、资源与规则

[basic-api-resource-rule | Sentinel (sentinelguard.io)](https://sentinelguard.io/zh-cn/docs/basic-api-resource-rule.html)

‍

### 定义资源

* [主流框架接入](https://sentinelguard.io/zh-cn/docs/open-source-framework-integrations.html)

* 使用注解@SentinelResource
* 自定义定义，包括 抛出异常、布尔值判断、异步异常捕获

  ```java
  /**
   ----------1 抛出异常------------------
  */
  try (Entry entry = SphU.entry("resourceName")) {
    // 被保护的业务逻辑
    // do something here...
  } catch (BlockException ex) {
    // 资源访问阻止，被限流或被降级
    // 在此处进行相应的处理操作
  }


  /**
    ----------1 布尔值判断------------------
  */
    // 资源名可使用任意有业务语义的字符串
    if (SphO.entry("自定义资源名")) {
      // 务必保证finally会被执行
      try {
        /**
        * 被保护的业务逻辑
        */
      } finally {
        SphO.exit();
      }
    } else {
      // 资源访问阻止，被限流或被降级
      // 进行相应的处理操作
    }
  ```

‍

### 定义规则

* 流量控制规则 (FlowRule)

  基本场景，可针对QPS、线程数等资源，按照限流策略（直接、链路、关联），达到流控效果（直接拒绝 / 排队等待 / 慢启动模式）。
* 熔断降级规则 (DegradeRule)

  对于高并发场景的异常接口情况（慢调用比例 异常比例 异常数）进行限制，达到服务降级效果
* 系统保护规则 (SystemRule)

  结合应用的 Load、CPU 使用率、总体平均 RT、入口 QPS 和并发线程数等几个维度的监控指标，通过自适应的流控策略，让系统的入口流量和系统的负载达到一个平衡
* 访问控制规则 (AuthorityRule)

  根据调用方来限制资源是否通过，这个更多像一个网关的功能
* 热点规则 (ParamFlowRule)

  利用 LRU 策略统计最近最常访问的热点参数，结合令牌桶算法来进行参数级别的流控。这个针对高并发的细节控制有意想不到效果。

经历过活动高并发的大流量直接把服务挂掉的经验，熔断、系统保护是必要的的，流控最为实用。

‍

#### 流控规则

​![image](/images/assets/image-20240226173616-kau7kcb.png)​

‍

‍

[流控规则详解](https://zhuanlan.zhihu.com/p/439916935)

#### 流控模式

1. 直接
2. 关联
    如果配置流控规则为关联模式，那么当 关联资源接口超过阈值过后，就会对 配置资源接口触发流控规则（*关联资源本身不受影响*）。
3. 链路

    设想存在以下业务场景：
    链路1（接口1 -> 公共资源1）
    链路2（接口2 -> 公共资源1）

    如果只想对链路1部分的公共资源进行限制，那么需要用到链路模式。
    >  [链路模式不生效](https://github.com/alibaba/sentinel/issues/1213)
    >
    > 如果修改完成不生效，@SentinelResource 注解生效基于SpringAop，那么就看看是否引用方式不对，类似事务注解不生效的场景。解决方式也类似。
    >
    所以，我们在设置链路流控规则的时候一定要设置 fallbackFactory。不然无法处理 FlowExecption 异常信息，造成系统出错

#### 流控效果

1. 快速失败  
    抛出异常
2. Warm Up
    流量会被限制到设定QPS/3（coldFactor默认为3），经过设置的预热时间（s）后才达到设定QPS阈值。
3. 排队等待（用的最多）
	丢到队列中异步执行，并设置最大等待时间

#### 熔断规则
​![image](/images/assets/image-20240226173645-8641hbr.png)​

#### 测试代码支持  
https://github.com/Hugh126/daydayup/tree/master/sentinel

# 四、控制台  

## 控制台使用:  

```json
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -Dproject.name=sentinel-dashboard -jar sentinel-dashboard-1.8.6.jar
// 如果希望换个端口， 且不希望控制台本身作为一个客户端接入
java -Dserver.port=8989 -jar sentinel-dashboard-1.8.6.jar
```
> 注意： Point有流量请求才会显示在控制台中

## 控制台规则推送模式

[在生产环境中使用 Sentinel · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/%E5%9C%A8%E7%94%9F%E4%BA%A7%E7%8E%AF%E5%A2%83%E4%B8%AD%E4%BD%BF%E7%94%A8-Sentinel#%E8%A7%84%E5%88%99%E7%AE%A1%E7%90%86%E5%8F%8A%E6%8E%A8%E9%80%81)

‍

**Sentinel Dashboard 通过客户端自带的规则 API来实时查询和更改内存中的规则。**

如果使用原始模式，非常的不方便，每次重启服务都把规则搞没了，要重新建立连接、重建规则。==>对接nacos，持久化到nacos，并动态推送

[动态规则扩展 · alibaba/Sentinel Wiki (github.com)](https://github.com/alibaba/Sentinel/wiki/%E5%8A%A8%E6%80%81%E8%A7%84%E5%88%99%E6%89%A9%E5%B1%95#%E6%8E%A8%E6%A8%A1%E5%BC%8F%E4%BD%BF%E7%94%A8-nacos-%E9%85%8D%E7%BD%AE%E8%A7%84%E5%88%99)

‍

## 问题

官方文档有点延迟， 说明也不够清晰。

* 如果需要控制台新建规则持久化到Nacos，则需要对dashboard进行改造。 具体参考：

[https://blog.csdn.net/qq_44870331/article/details/129886930](https://blog.csdn.net/qq_44870331/article/details/129886930)  


* 如果只需要从Nacos数据源读取配置：

  ```java
          ReadableDataSource<String, List<FlowRule>> flowRuleDataSource =
                  new NacosDataSource<>("localhost", "DEFAULT_GROUP", "test",
                  source -> JSON.parseObject(source, new TypeReference<List<FlowRule>>() {}));
          FlowRuleManager.register2Property(flowRuleDataSource.getProperty());
  ```

---
‍
本人fork了sentinel，并对1.8.6分支进行了改造，可以直接使用支持集成Nacos的控制台：  
 [改造后项目地址](https://github.com/Hugh126/Sentinel/tree/Branch_1.8.6)

‍
