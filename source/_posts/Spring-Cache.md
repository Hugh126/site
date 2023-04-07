---
title: Spring Cache
date: 2023-03-31 19:30:47
tags: spring cache
---
SpringCache是Spring基于Spring切面提供的缓存增强技术。它集成了缓存常见的各种操作,如Cacheable/CachePut/CacheEvit等；也支持了ConcurrentMapCache/RedisCache/EhCacheCache等各种缓存实现。它集成简单，支持丰富，实乃是业务开发的利器！  

<!--more-->

## SpringCache组件
先来看看一些接触到的常见注解  
|  名称   | 解释  |
|  ----  | ----  |
| CacheManager  | 缓存管理器 |
| EnableCaching  | 开启基于注解的缓存 |
| @Cacheable  | 根据方法的请求参数对其进行缓存 |
| @CachePut  | 更新缓存 |
| @CacheEvict  | 清除缓存 |
|   |  |
| @CacheConfig  | 统一配置本类的缓存注解的属性 |
| keyGenerator  | 缓存数据时key生成策略 |
| serialize  | 缓存数据时value序列化策略 |

## SpringCache基本使用
1. 引入jar包
``` xml
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-cache</artifactId>
    </dependency>
```

2. 开启Cache支持  
这里`@EnableCaching`可以直接作用于SpringBoot的启动类，但为了后续扩展。  
我们定义一个CachingConfigurerSupport继承类，并加到这里
``` java
@EnableCaching
@Configuration
public class CacheConfig extends CachingConfigurerSupport {
    // 后续扩展
}
```

3. @Cacheable("xxx")作用于方法  
定义一个get方法，并加上Cacheable注解就可以了。  
这里cache需要要有一个key，因此cacheNames必不可少。  


##扩展：使用Reids作为缓存实现 
1. yml配置  
```
spring:
  cache:
    type: redis
```
2. 定义ttl  
``` java
@EnableCaching
@Configuration
public class CacheConfig extends CachingConfigurerSupport {

    @Bean
    public RedisCacheConfiguration buildRedisConfig() {

        return RedisCacheConfiguration.defaultCacheConfig()
                // 如果配置不缓存空，则注意返回对象也不可为空，否则报错
//                .disableCachingNullValues()
                .entryTtl(Duration.ofMinutes(5L))
                .prefixCacheNameWith("My_Prefix_")
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
    }
```
这里，定义了ttl为5分钟，然后使用了自定义的前缀。序列化存储是为了在redisClient中容易参照。  

## 自定义key
其实，如果你使用@Cacheable和@CachePut分别作用于getById和updateById的时候就会发现，为什么CachePut明明会放置结果到缓存，为什么在get的时候获取不到？ 其实，都是因为默认的key不相同导致的。 
``` java
@RequestMapping("/user")
@RestController
@CacheConfig(cacheNames = "userCache")
@Slf4j
public class CacheController {
    private static Map<Integer, String> data_map =
            new HashMap(){{
                put(1, "Lily");
                put(2, "Linger");
            }};
    @RequestMapping("/get/{id}")
    @Cacheable(key = "#id")
    public String get(@PathVariable Integer id) {
      log.warn("get {}", id);
      return data_map.get(id);
    }      

    @RequestMapping("/update")
   @CachePut(key = "#id")
    public String update(@RequestParam Integer id, @RequestParam String name) {
        log.warn("add {} {}", id, name);
        data_map.put(id, name);
        return name;
    }      

}    
```
像上面这样，都定义key为请求的ID，那么自然能达到我们想要的效果。  

## 使用心得
  
1、Spring Cache使用还是很方便的，但是要特别注意，不然容易出bug  
2、使用统一的CacheConfig, 明确定义查增改的key，或使用统一keyGenerator  
3、由于get和update使用了一样的key，那么update方法的返回必须和get一致  
4、注意key对应方法返回值就是key对应value！！尽量少的ttl，不然服务重启依旧取不到最新值  
5、对于一般非联动缓存，直接用@Cacheable即可。方法cacheNames会覆盖类上的  

---
以上例子的代码都已上传到Git： 
https://github.com/Hugh126/daydayup.git  

参考  
https://blog.csdn.net/qq_32448349/article/details/101696892  




