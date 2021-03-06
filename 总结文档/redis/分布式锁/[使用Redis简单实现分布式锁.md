# [使用Redis简单实现分布式锁](https://www.cnblogs.com/yanwu0527/p/13210578.html)

分布式锁的几种实现方式：mamcache、redis、zookeeper，本片就redis实现分布式锁进行简单的介绍与实现

##### redis实现分布式锁

###### 加锁

最简单的方法是使用`setnx`命令，`key`是唯一的标志，可以按照业务来命名，而`value`最好的做法是使用`线程ID`

```shell
setnx(key, thread_id)
```

当`setnx`返回`1`说明`key`原本不存在，该线程成功获取锁；当`setnx`返回`0`说明`key`已经存在，该线程获取锁失败。

###### 解锁

当持有锁的线程执行完成后，需要释放锁，以便其他的线程可以进入。释放锁的的方式是`del`命令

```shell
del(key)
```

释放锁之后，其他的线程就可以继续执行`setnx`命令来加锁

###### 锁超时

如果一个线程在执行任务的过程中挂掉，来不及释放锁，这块资源将永远被锁住__（死锁）__，别的线程再也别想进来，所以`setnx`的`key`必须设置一个超时时间，以保证即使没有被显示的释放锁，这把锁在一定的时间之后会自动释放，但是由于`setnx`不支持超时参数，所以需要使用额外的指令

```shell
expire(key, timeout)
```

完整的伪代码如下

```java
if (setnx(key, thread_id) == 1) {
	expire(key, timeout);
    try {
        // ----- 业务代码
    } finally {
        del(key);
    }
}
```

但是上述的做法存在一些问题：

1. `setnx`和`expire`这是两个操作，他们并不是原子性的，所以在极端情况下有可能加锁成功了，但是给锁设置超时时间的时候服务出错了导致设置超时时间失败了，此时还是会变成__死锁__

   所以一般情况下我们都使用`set`指令来替代`setnx`指令，因为`set`指令有可选参数

   ```java
   if (set(key, thread_id, timeout, NX) == 1) {
   	expire(key, timeout);
       try {
           // ----- 业务代码
       } finally {
           del(key);
       }
   }
   ```

2. `del`误删：这又是一个极端场景，加入A线程成功加锁并且设置了超时时间是30秒，如果A业务执行太慢过了30秒还没有执行完，这个时候锁过期了会自动释放，B线程得到了锁，当A线程执行完之后，接着执行`del`指令，但是这个时候B线程还没有执行，A会将锁释放。

   该问题的解决方式也很简单，就是将`set`指令的`value`设置为线程ID，在释放锁之前进行验证，当前线程ID是否正确

   ```java
   if (set(key, thread_id, timeout, NX) == 1) {
   	expire(key, timeout);
       try {
           // ----- 业务代码
       } finally {
       	if (thread_id.equaks(get(key))) {
   	        del(key);
       	}
       }
   }
   ```

   同时将线程Id设置为`value`还可以解决__重入__问题

3. 虽然我们将线程Id作为`value`避免了`key`误删的情况，但是此时同一时间有两个线程在只想业务，仍然是不完美的，这种情况我们可以通过守护线程的方式给锁续航

   让获取锁的线程开启一个守护线程，在锁快要到期的时候，使用守护线程来给锁增加超时时间：

   1. 当持有锁的线程执行完之后，显示的关闭掉守护线程
   2. 当持有锁的线程所在服务挂掉后，守护线程也会挂掉，此时没有续航到时间一样会被释放掉

```java
/***
 * 使用RedisTemplate简单实现分布式锁
 */
@Slf4j
@Component
public class RedisLockUtil {
    /*** 分布式锁固定前缀 ***/
    private static final String REDIS_LOCK = "redis_lock_";
    /*** 分布式锁过期时间 ***/
    private static final Integer EXPIRE_TIME = 30;
    /*** 每次自旋睡眠时间 ***/
    private static final Integer SLEEP_TIME = 50;
    /*** 分布式锁自旋次数 ***/
    private static final Integer CYCLES = 10;
    @SuppressWarnings("all")
    @Resource(name = "redisTemplate")
    private ValueOperations<String, String> lockOperations;

    /**
     * 加锁
     *
     * @param key   加锁唯一标识
     * @param value 释放锁唯一标识（建议使用线程ID作为value）
     */
    public void lock(String key, String value) {
        lock(key, value, EXPIRE_TIME);
    }

    /**
     * 加锁
     * @param key     加锁唯一标识
     * @param value   释放锁唯一标识（建议使用线程ID作为value）
     * @param timeout 超时时间（单位：S）
     */
    public void lock(String key, String value, Integer timeout) {
        Assert.isTrue(StringUtils.isNotBlank(key), "redis locks are identified as null.");
        Assert.isTrue(StringUtils.isNotBlank(value), "the redis release lock is identified as null.");
        int cycles = CYCLES;
        // ----- 尝试获取锁，当获取到锁，则直接返回，否则，循环尝试获取
        while (!tryLock(key, value, timeout)) {
            // ----- 最多循环10次，当尝试了10次都没有获取到锁，抛出异常
            if (0 == (cycles--)) {
                log.error("redis try lock fail. key: {}, value: {}", key, value);
                throw new RuntimeException("redis try lock fail.");
            }
            try {
                TimeUnit.MILLISECONDS.sleep(SLEEP_TIME);
            } catch (Exception e) {
                log.error("history try lock error.", e);
            }
        }
    }

    /**
     * 尝试获取锁
     * @param key     加锁唯一标识
     * @param value   释放锁唯一标识（建议使用线程ID作为value）
     * @param timeout 超时时间（单位：S）
     * @return [true: 加锁成功; false: 加锁失败]
     */
    private boolean tryLock(String key, String value, Integer timeout) {
        Boolean result = lockOperations.setIfAbsent(REDIS_LOCK + key, value, timeout, TimeUnit.SECONDS);
        return result != null && result;
    }

    /**
     * 释放锁
     * @param key   加锁唯一标识
     * @param value 释放锁唯一标识（建议使用线程ID作为value）
     */
    public void unLock(String key, String value) {
        Assert.isTrue(StringUtils.isNotBlank(key), "redis locks are identified as null.");
        Assert.isTrue(StringUtils.isNotBlank(value), "the redis release lock is identified as null.");
        key = REDIS_LOCK + key;
        // ----- 通过value判断是否是该锁：是则释放；不是则不释放，避免误删
        if (value.equals(lockOperations.get(key))) {
            lockOperations.getOperations().delete(key);
        }
    }
```

```
1、setIfAbsent(lockKey,value,expireTime, timeUnit) 可以保证加锁和时间是原子性操作
*                 2、注意redis的版本要2.1之上才有这个方法
*    问题：单节点redis情况下出现的问题，锁问题
*    1、首先是加锁和时间没办法保证原子性 -> redis已经给解决了
*    2、如果一个线程A执行代码的时间过长，导致可能出现A锁已经失效，另一个线程B加锁后，A线程释放了B锁 -> value值可以设置为
*       线程id或者我们通过uuid来实现，进行释放锁时进行对比是否释放了自己的锁
*       虽然解决了没办法删除别人的锁了，但是这个锁还是到时间就释放了，还是出现问题，可以通过守护线程的方式进行时间检测，不断地加时间
     解决：可以通过导入redis
```