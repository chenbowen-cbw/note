# Springboot-Redis 分布式锁

## 一. Redis 分布式锁的实现以及存在的问题

锁是针对某个资源，保证其访问的互斥性，在实际使用中，这个资源一般是个字符串。使用 Redis 实现分锁，主要是将资源放到 redis 中利用其原子性，当其他线程访问时，如果 redis 中已经存在这个资源，就不允许之后的一些操作。spring boot 使用 Redis 的操作主要是通过 RedisTemplate 来实现，一般步骤如下：

1. 将锁资源放入 Redis （注意是当 key 不存在时才能放成功，所以使用 setIfAbsent 方法）：
   > redisTemplate.opsForValue().setIfAbsent("key", "value");
   > setIfAbsent 方法是判断 key 是否存在，如果存在则返回 false，不存在则返回 true，并且设置 key 的值为 value。
2. 设置过期时间
   > redisTemplate.expire("key", 30000, TimeUnit.MILLISECONDS);
3. 释放锁
   > redisTemplate.delete("key");

一般情况下，这样的实现就能够满足锁的需求了，但是如果在调用 setIfAbsent 方法之后线程挂掉了，即没有给锁定的资源设置过期时间，默认是永不过期，那么这个锁就会一直存在。所以需要保证设置锁及其过期时间两个操作的原子性，spring data 的 RedisTemplate 当中并没有这样的方法。

但是在 jedis 中是有这种原子操作方法的，需要通过 redisTemplate 的 execute 方法获取到 jedis 里操作命令的对象，代码如下：

```
String result = redisTemplate.execute(new RedisCallback<String>() {
    @Override
    public String doInRedis(RedisConnection connection) throws DataAccessException {
        JedisCommands commands = (JedisCommands) connection.getNativeConnection();
        return commands.set(key, "锁定的资源", "NX", "PX", expire);
    }
});
```

NX： 表示只有当锁定资源不存在的时候才能 SET 成功。利用 Redis 的原子性，保证了只有第一个请求的线程才能获得锁，而之后的所有线程在锁定资源被释放之前都不能获得锁。

PX： expire 表示锁定的资源的自动过期时间，单位是毫秒。具体过期时间根据实际场景而定

这样在获取锁的时候就能够保证设置 Redis 值和过期时间的原子性，避免前面提到的两次 Redis 操作期间出现意外而导致的锁不能释放的问题。但是这样还是可能会存在一个问题，考虑如下的场景顺序：

线程 T1 获取锁

线程 T1 执行业务操作，由于某些原因阻塞了较长时间

锁自动过期，即锁自动释放了

线程 T2 获取锁

线程 T1 业务操作完毕，释放锁（其实是释放的线程 T2 的锁）

按照这样的场景顺序，线程 T2 的业务操作实际上就没有锁提供保护机制了。所以，每个线程释放锁的时候只能释放自己的锁，即锁必须要有一个拥有者的标记，并且也需要保证释放锁的原子性操作。

因此我们可以通过 Lua 脚本来达到释放锁的原子操作，定义 Lua 脚本如下：

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

具体意思可以参考上面提供的文档地址

使用 RedisTemplate 执行的代码如下：

// 使用 Lua 脚本删除 Redis 中匹配 value 的 key，可以避免由于方法执行时间过长而 redis 锁自动过期失效的时候误删其他线程的锁
// spring 自带的执行脚本方法中，集群模式直接抛出不支持执行脚本的异常，所以只能拿到原 redis 的 connection 来执行脚本

```
Long result = redisTemplate.execute(new RedisCallback<Long>() {
    public Long doInRedis(RedisConnection connection) throws DataAccessException {
        Object nativeConnection = connection.getNativeConnection();
        // 集群模式和单机模式虽然执行脚本的方法一样，但是没有共同的接口，所以只能分开执行
        // 集群模式
        if (nativeConnection instanceof JedisCluster) {
            return (Long) ((JedisCluster) nativeConnection).eval(UNLOCK_LUA, keys, args);
        }

        // 单机模式
        else if (nativeConnection instanceof Jedis) {
            return (Long) ((Jedis) nativeConnection).eval(UNLOCK_LUA, keys, args);
        }
        return 0L;
    }
});
```

代码中分为集群模式和单机模式，并且两者的方法、参数都一样，原因是 spring 封装的执行脚本的方法中（ RedisConnection 接口继承于 RedisScriptingCommands 接口的 eval 方法），集群模式的方法直接抛出了不支持执行脚本的异常（虽然实际是支持的），所以只能拿到 Redis 的 connection 来执行脚本，而 JedisCluster 和 Jedis 中的方法又没有实现共同的接口，所以只能分开调用。

spring 封装的集群模式执行脚本方法源码：

```
 JedisClusterConnection.java
/**
 * (non-Javadoc)
 * @see org.springframework.data.redis.connection.RedisScriptingCommands#eval(byte[], org.springframework.data.redis.connection.ReturnType, int, byte[][])
 */
@Override
public <T> T eval(byte[] script, ReturnType returnType, int numKeys, byte[]... keysAndArgs) {
    throw new InvalidDataAccessApiUsageException("Eval is not supported in cluster environment.");
}
```

至此，我们就完成了一个相对可靠的 Redis 分布式锁，但是，在集群模式的极端情况下，还是可能会存在一些问题，比如如下的场景顺序（ 本文暂时不深入开展 ）：

线程 T1 获取锁成功

Redis 的 master 节点挂掉，slave 自动顶上

线程 T2 获取锁，会从 slave 节点上去判断锁是否存在，由于 Redis 的 master slave 复制是异步的，所以此时线程 T2 可能成功获取到锁

为了可以以后扩展为使用其他方式来实现分布式锁，定义了接口和抽象类，所有的源码如下：

```
# DistributedLock.java 顶级接口
/**
 * @author fuwei.deng
 * @date 2017年6月14日 下午3:11:05
 * @version 1.0.0
 */
public interface DistributedLock {

    public static final long TIMEOUT_MILLIS = 30000;

    public static final int RETRY_TIMES = Integer.MAX_VALUE;

    public static final long SLEEP_MILLIS = 500;

    public boolean lock(String key);

    public boolean lock(String key, int retryTimes);

    public boolean lock(String key, int retryTimes, long sleepMillis);

    public boolean lock(String key, long expire);

    public boolean lock(String key, long expire, int retryTimes);

    public boolean lock(String key, long expire, int retryTimes, long sleepMillis);

    public boolean releaseLock(String key);
}
```

```
# AbstractDistributedLock.java 抽象类，实现基本的方法，关键方法由子类去实现
/**
 * @author fuwei.deng
 * @date 2017年6月14日 下午3:10:57
 * @version 1.0.0
 */
public abstract class AbstractDistributedLock implements DistributedLock {

    @Override
    public boolean lock(String key) {
        return lock(key, TIMEOUT_MILLIS, RETRY_TIMES, SLEEP_MILLIS);
    }

    @Override
    public boolean lock(String key, int retryTimes) {
        return lock(key, TIMEOUT_MILLIS, retryTimes, SLEEP_MILLIS);
    }

    @Override
    public boolean lock(String key, int retryTimes, long sleepMillis) {
        return lock(key, TIMEOUT_MILLIS, retryTimes, sleepMillis);
    }

    @Override
    public boolean lock(String key, long expire) {
        return lock(key, expire, RETRY_TIMES, SLEEP_MILLIS);
    }

    @Override
    public boolean lock(String key, long expire, int retryTimes) {
        return lock(key, expire, retryTimes, SLEEP_MILLIS);
    }

}
```

```
# RedisDistributedLock.java Redis分布式锁的实现
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.dao.DataAccessException;
import org.springframework.data.redis.connection.RedisConnection;
import org.springframework.data.redis.core.RedisCallback;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.util.StringUtils;

import redis.clients.jedis.Jedis;
import redis.clients.jedis.JedisCluster;
import redis.clients.jedis.JedisCommands;

/**
 * @author fuwei.deng
 * @date 2017年6月14日 下午3:11:14
 * @version 1.0.0
 */
public class RedisDistributedLock extends AbstractDistributedLock {

    private final Logger logger = LoggerFactory.getLogger(RedisDistributedLock.class);

    private RedisTemplate<Object, Object> redisTemplate;

    private ThreadLocal<String> lockFlag = new ThreadLocal<String>();

    public static final String UNLOCK_LUA;

    static {
        StringBuilder sb = new StringBuilder();
        sb.append("if redis.call(\"get\",KEYS[1]) == ARGV[1] ");
        sb.append("then ");
        sb.append("    return redis.call(\"del\",KEYS[1]) ");
        sb.append("else ");
        sb.append("    return 0 ");
        sb.append("end ");
        UNLOCK_LUA = sb.toString();
    }

    public RedisDistributedLock(RedisTemplate<Object, Object> redisTemplate) {
        super();
        this.redisTemplate = redisTemplate;
    }

    @Override
    public boolean lock(String key, long expire, int retryTimes, long sleepMillis) {
        boolean result = setRedis(key, expire);
        // 如果获取锁失败，按照传入的重试次数进行重试
        while((!result) && retryTimes-- > 0){
            try {
                logger.debug("lock failed, retrying..." + retryTimes);
                Thread.sleep(sleepMillis);
            } catch (InterruptedException e) {
                return false;
            }
            result = setRedis(key, expire);
        }
        return result;
    }

    private boolean setRedis(String key, long expire) {
        try {
            String result = redisTemplate.execute(new RedisCallback<String>() {
                @Override
                public String doInRedis(RedisConnection connection) throws DataAccessException {
                    JedisCommands commands = (JedisCommands) connection.getNativeConnection();
                    String uuid = UUID.randomUUID().toString();
                    lockFlag.set(uuid);
                    return commands.set(key, uuid, "NX", "PX", expire);
                }
            });
            return !StringUtils.isEmpty(result);
        } catch (Exception e) {
            logger.error("set redis occured an exception", e);
        }
        return false;
    }

    @Override
    public boolean releaseLock(String key) {
        // 释放锁的时候，有可能因为持锁之后方法执行时间大于锁的有效期，此时有可能已经被另外一个线程持有锁，所以不能直接删除
        try {
            List<String> keys = new ArrayList<String>();
            keys.add(key);
            List<String> args = new ArrayList<String>();
            args.add(lockFlag.get());

            // 使用lua脚本删除redis中匹配value的key，可以避免由于方法执行时间过长而redis锁自动过期失效的时候误删其他线程的锁
            // spring自带的执行脚本方法中，集群模式直接抛出不支持执行脚本的异常，所以只能拿到原redis的connection来执行脚本

            Long result = redisTemplate.execute(new RedisCallback<Long>() {
                public Long doInRedis(RedisConnection connection) throws DataAccessException {
                    Object nativeConnection = connection.getNativeConnection();
                    // 集群模式和单机模式虽然执行脚本的方法一样，但是没有共同的接口，所以只能分开执行
                    // 集群模式
                    if (nativeConnection instanceof JedisCluster) {
                        return (Long) ((JedisCluster) nativeConnection).eval(UNLOCK_LUA, keys, args);
                    }

                    // 单机模式
                    else if (nativeConnection instanceof Jedis) {
                        return (Long) ((Jedis) nativeConnection).eval(UNLOCK_LUA, keys, args);
                    }
                    return 0L;
                }
            });

            return result != null && result > 0;
        } catch (Exception e) {
            logger.error("release lock occured an exception", e);
        }
        return false;
    }

}
```

二. 基于 AOP 的 Redis 分布式锁
在实际的使用过程中，分布式锁可以封装好后使用在方法级别，这样就不用每个地方都去获取锁和释放锁，使用起来更加方便。

首先定义个注解：

```
import java.lang.annotation.ElementType;
import java.lang.annotation.Inherited;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

/**
 * @author fuwei.deng
 * @date 2017年6月14日 下午3:10:36
 * @version 1.0.0
 */
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
public @interface RedisLock {

    /** 锁的资源，redis的key*/
    String value() default "default";

    /** 持锁时间,单位毫秒*/
    long keepMills() default 30000;

    /** 当获取失败时候动作*/
    LockFailAction action() default LockFailAction.CONTINUE;

    public enum LockFailAction{
        /** 放弃 */
        GIVEUP,
        /** 继续 */
        CONTINUE;
    }

    /** 重试的间隔时间,设置GIVEUP忽略此项*/
    long sleepMills() default 200;

    /** 重试次数*/
    int retryTimes() default 5;
}
```

装配分布式锁的 bean

```
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.condition.ConditionalOnBean;
import org.springframework.boot.autoconfigure.data.redis.RedisAutoConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.core.RedisTemplate;

import com.itopener.lock.redis.spring.boot.autoconfigure.lock.DistributedLock;
import com.itopener.lock.redis.spring.boot.autoconfigure.lock.RedisDistributedLock;

/**
 * @author fuwei.deng
 * @date 2017年6月14日 下午3:11:31
 * @version 1.0.0
 */
@Configuration
@AutoConfigureAfter(RedisAutoConfiguration.class)
public class DistributedLockAutoConfiguration {

    @Bean
    @ConditionalOnBean(RedisTemplate.class)
    public DistributedLock redisDistributedLock(RedisTemplate<Object, Object> redisTemplate){
        return new RedisDistributedLock(redisTemplate);
    }

}
```

定义切面（spring boot 配置方式）

```
import java.lang.reflect.Method;
import java.util.Arrays;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.autoconfigure.AutoConfigureAfter;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.context.annotation.Configuration;
import org.springframework.util.StringUtils;

import com.itopener.lock.redis.spring.boot.autoconfigure.annotations.RedisLock;
import com.itopener.lock.redis.spring.boot.autoconfigure.annotations.RedisLock.LockFailAction;
import com.itopener.lock.redis.spring.boot.autoconfigure.lock.DistributedLock;

/**
 * @author fuwei.deng
 * @date 2017年6月14日 下午3:11:22
 * @version 1.0.0
 */
@Aspect
@Configuration
@ConditionalOnClass(DistributedLock.class)
@AutoConfigureAfter(DistributedLockAutoConfiguration.class)
public class DistributedLockAspectConfiguration {

    private final Logger logger = LoggerFactory.getLogger(DistributedLockAspectConfiguration.class);

    @Autowired
    private DistributedLock distributedLock;

    @Pointcut("@annotation(com.itopener.lock.redis.spring.boot.autoconfigure.annotations.RedisLock)")
    private void lockPoint(){

    }

    @Around("lockPoint()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable{
        Method method = ((MethodSignature) pjp.getSignature()).getMethod();
        RedisLock redisLock = method.getAnnotation(RedisLock.class);
        String key = redisLock.value();
        if(StringUtils.isEmpty(key)){
            Object[] args = pjp.getArgs();
            key = Arrays.toString(args);
        }
        int retryTimes = redisLock.action().equals(LockFailAction.CONTINUE) ? redisLock.retryTimes() : 0;
        boolean lock = distributedLock.lock(key, redisLock.keepMills(), retryTimes, redisLock.sleepMills());
        if(!lock) {
            logger.debug("get lock failed : " + key);
            return null;
        }

        //得到锁,执行方法，释放锁
        logger.debug("get lock success : " + key);
        try {
            return pjp.proceed();
        } catch (Exception e) {
            logger.error("execute locked method occured an exception", e);
        } finally {
            boolean releaseResult = distributedLock.releaseLock(key);
            logger.debug("release lock : " + key + (releaseResult ? " success" : " failed"));
        }
        return null;
    }
}
```

spring boot starter 还需要在 resources/META-INF 中添加 spring.factories 文件

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.itopener.lock.redis.spring.boot.autoconfigure.DistributedLockAutoConfiguration,\
com.itopener.lock.redis.spring.boot.autoconfigure.DistributedLockAspectConfiguration
```

这样封装之后，使用 spring boot 开发的项目，直接依赖这个 starter，就可以在方法上加 RedisLock 注解来实现分布式锁的功能了，当然如果需要自己控制，直接注入分布式锁的 bean 即可

```
@Autowired
private DistributedLock distributedLock;
```

如果需要使用其他的分布式锁实现，继承 AbstractDistributedLock 后实现获取锁和释放锁的方法即可
