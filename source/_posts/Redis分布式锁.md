---
title: Redis分布式锁
date: 2023-07-01 11:56:54
tags: 
- redis
categories: 
- Java
---
> SETNX key value
> 
> SETEX key seconds value
> 
> setnx (SET if Not exists), 如果不存在set, 成功返回1, 已存在则返回0, 是一个原子性(atomic)操作, 
> Redisson 底层也是通过这两个命令结合lua脚本原子性执行完成redis分布式锁
> 
> ![redission分布式锁原理](..\img\post\redission分布式锁原理.png)


## 方案一：推荐直接使用Redisson，框架做了很多细节处理
调用测试
~~~java
@Autowired
private Redisson redisson;

@Test
private void testRedisson() {
        // Redisson
        long time5 = System.currentTimeMillis();
        RLock redissonLock = redisson.getLock("kkk");
        boolean redissonTryLock = redissonLock.tryLock(5L, TimeUnit.SECONDS);
        if (redissonTryLock) {
            try {
                System.out.println("拿到了redissonTryLock");
            } finally {
                redissonLock.unlock();
            }
        }
        long time6 = System.currentTimeMillis();
        System.out.println(time6 - time5); // 耗时20毫秒
}
~~~
~~~xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.11.4</version>
</dependency>
~~~
## 方案二：(能用，但是不好用)
~~~java
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.DefaultRedisScript;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.stereotype.Component;

import javax.annotation.Resource;
import java.time.Duration;
import java.util.Collections;
import java.util.Objects;

/**
 * Redis 分布式锁
 **/
@Component
public class RedisLockUtils {
    @Resource
    private RedisTemplate<String, String> redisTemplate;
    // 分布式锁过期时间 s
    private static final Long LOCK_REDIS_TIMEOUT = 10L;
    /**
     * 加锁
     **/
    public Boolean getLock(String key, String value) {
        return this.redisTemplate.opsForValue().setIfAbsent(key, value, Duration.ofSeconds(LOCK_REDIS_TIMEOUT));
    }
    /**
     * 释放锁
     **/
    public boolean releaseLock(String key, String value) {
        String luaScript = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        RedisScript<Long> redisScript = new DefaultRedisScript<>(luaScript, Long.class);
        return !Objects.equals(this.redisTemplate.execute(redisScript, Collections.singletonList(key), value), 0L);
    }
}
~~~
调用测试
~~~java
        // 自定义的redis锁
        UUID uuid = UUID.randomUUID();
        long time1 = System.currentTimeMillis();
        Boolean isGetLock = redisLockUtils.getLock("redisKey", uuid.toString());
        try {
            if (isGetLock) {
                System.out.println("拿到了自定义的redis锁");
            }
        } finally {
            System.out.println(redisLockUtils.releaseLock("redisKey", uuid.toString()));
        }
        long time2 = System.currentTimeMillis();
        System.out.println(time2 - time1); // 458毫秒，而redisson耗时20毫秒
~~~