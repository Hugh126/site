---
title: RedLock
date: 2023-08-01 15:26:24
tags:
---
RedLock是一种分布式锁的实现方式，算法思想很好理解：假设有N个相互独立无任何协作的Redis master，在每个master上使用单机Redis的加锁方式加锁。当有N/2+1个节点加锁成功，即获取分布式锁成功；否则在所有实例上解锁。
<!--more-->

# 客户端行为
1. 获取当前Unix时间，以毫秒为单位。
2. 依次尝试从N个Master实例使用相同的key和随机值获取锁（假设这个key是LOCK_KEY）。当向Redis设置锁时，客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个Redis实例。
3. 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。
4. 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。
5. 如果因为某些原因，获取锁失败（没有在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功）。  
> 参考:    
    [RedLock算法理解](https://zhuanlan.zhihu.com/p/374732293)

# 使用场景
- 存在网络分区
- 集群存在故障转移，需要容错  
  
## 分布式锁说明
一般说的redis分布式锁，其实是针对多个客户端获取锁和释放锁，redis分布式锁需要保证操作的原子性。这种情况下的，一般一个redis集群中就只有1个master，分布式锁的获取只是单机操作（包括redisson的getLock）。  
而如果是cluster集群模式，不同key分布在不同的master上；或者在哨兵模式集群之上加一次负载均衡，串联多个哨兵集群作为一个大集群。针对这种相互独立的多个Master，需要加上一个分布式锁的话，Redis官方推出了一个规范的算法：RedLock。
> 参考：  
    [Redis的三种集群模式](https://zhuanlan.zhihu.com/p/145186839)  

## 通过jedis获取分布式锁
``` java 
import redis.clients.jedis.Jedis;
import redis.clients.jedis.params.SetParams;
import java.util.Collections;

public class RedisLock {

    private static final Long RELEASE_SUCCESS = 1L;
    private static final String LOCK_SUCCESS = "OK";
    private static final String KEY_PREFIX="DE_BATCH_LOCK_";

    /**
     * 释放分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @return 是否释放成功
     */
    public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {

        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(KEY_PREFIX+lockKey), Collections.singletonList(requestId));
        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;
    }


    /**
     * 尝试获取分布式锁
     * @param jedis Redis客户端
     * @param lockKey 锁
     * @param requestId 请求标识
     * @param expireTime 超期时间 s
     * @return 是否获取成功
     */
    public static boolean tryGetDistributedLock(Jedis jedis, String lockKey, String requestId, Integer expireTime) {
        SetParams params = SetParams.setParams().nx();
        if (expireTime != null) {
            params.ex(expireTime);
        }
        String result = jedis.set(KEY_PREFIX+lockKey, requestId, params);
        if (LOCK_SUCCESS.equals(result)) {
            return true;
        }
        String value=jedis.get(KEY_PREFIX+lockKey);
        logger.error("try get lock failed，current value = " + value);
        return false;
    }
}    
```


## 通过Redisson获取分布式锁
``` java
RedissonClient client = Redisson.create(config);
RLock lock = getClient().getLock(lockKey);
lock.tryLock(waitTime, leaseTime, unit);
// lock.lock()是阻塞方法 
```

# 基于Redisson的RedLock实现

``` java
RLock lock1 = redissonClient1.getLock(lockKey);
RLock lock2 = redissonClient2.getLock(lockKey);
RLock lock3 = redissonClient3.getLock(lockKey);
// 向3个master加锁
RedissonRedLock redLock = new RedissonRedLock(lock1, lock2, lock3);
boolean isLock;
try {
    isLock = redLock.tryLock(1, 10, TimeUnit.SECONDS);
    if (isLock) {
        // do something;
    }
} catch (Exception e) {
    e.printStackTrace();
} finally {
    redLock.unlock();
}
```





