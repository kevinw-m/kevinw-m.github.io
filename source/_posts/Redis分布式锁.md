---
title: Redis分布式锁
date: 2023-07-01 11:56:54
tags: 
- redis
categories: 
- Java
---
## 方案一：推荐直接使用Redisson
~~~java
@Autowired
private Redisson redisson;

private void test() {
        RLock lock = redisson.getLock("redis key");
        try {
            if (lock.tryLock(3L, TimeUnit.SECONDS)) {
                try {
                    // do something...
                } finally {
                    lock.unlock();
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
}
~~~
~~~xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.11.4</version>
</dependency>
~~~
## 方案二：
~~~java
/**
 * Redis 分布式锁
 *
 **/
@Component
public class RedisLockUtils {
 
    @Autowired
    private RedisTemplate redisTemplate;
 
    //分布式锁过期时间 s  可以根据业务自己调节
    private static final Long LOCK_REDIS_TIMEOUT = 10L;
    //分布式锁休眠 至 再次尝试获取 的等待时间 ms 可以根据业务自己调节
    public static final Long LOCK_REDIS_WAIT = 500L;
 
 
    /**
     *  加锁
     **/
    public Boolean getLock(String key,String value){
        Boolean lockStatus = this.redisTemplate.opsForValue().setIfAbsent(key,value, Duration.ofSeconds(LOCK_REDIS_TIMEOUT));
        return lockStatus;
    }
 
    /**
     *  释放锁
     **/
    public Long releaseLock(String key,String value){
        String luaScript = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        RedisScript<Long> redisScript = new DefaultRedisScript<>(luaScript,Long.class);
        Long releaseStatus = (Long)this.redisTemplate.execute(redisScript, Collections.singletonList(key),value);
        return releaseStatus;
    }
}
~~~
