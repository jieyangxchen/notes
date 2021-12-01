# <center>Redisson分布式锁浅析</center>

> 想要实现分布式锁，可以利用redis实现
>
>原理还是锁本身的原理
>
> 原生redis分布式锁是利用redis内置的lua脚本来实现

但如果自己手撸一个包含了读写分离、续约处理、重试机制的分布式锁的话，
可能会花费大量时间，所以就引入了redisson

## 一、前置操作

1. 依赖

```xml
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson</artifactId>
    <version>3.16.4</version>
</dependency>
```

2. 配置

```java
@Configuration
public class RedissionConfig {
    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.password}")
    private String password;

    private int port = 6379;

    @Bean
    public RedissonClient getRedisson() {
        Config config = new Config();
        config.useSingleServer().
                setAddress("redis://" + redisHost + ":" + port).
                setPassword(password);
        config.setCodec(new JsonJacksonCodec());
        return Redisson.create(config);
    }
}
```

3. 启用Rlock

```java
@Resource
private RedissonClient redissonClient;
...
RLock rLock = redissonClient.getLock(lockName);
try {
    boolean isLocked = rLock.tryLock(expireTime, TimeUnit.MILLISECONDS);
    if (isLocked) {
        // TODO
                }
    } catch (Exception e) {
            rLock.unlock();
    }
...
```



## 二、加锁原理

1.进入trylock方法-->tryAcquireOnceAsync方法

```java
private RFuture<Boolean> tryAcquireOnceAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId) {
        RFuture<Boolean> ttlRemainingFuture;
        if (leaseTime != -1) {
            ttlRemainingFuture = tryLockInnerAsync(waitTime, leaseTime, unit, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
        } else {
            // 尝试获取锁
            ttlRemainingFuture = tryLockInnerAsync(waitTime, internalLockLeaseTime,
                    TimeUnit.MILLISECONDS, threadId, RedisCommands.EVAL_NULL_BOOLEAN);
        }

        ttlRemainingFuture.onComplete((ttlRemaining, e) -> {
            if (e != null) {
                return;
            }

            // lock acquired
            if (ttlRemaining) {
                if (leaseTime != -1) {
                    internalLockLeaseTime = unit.toMillis(leaseTime);
                } else {
                    scheduleExpirationRenewal(threadId);
                }
            }
        });
        return ttlRemainingFuture;
    }
```

- tryAcquireOnceAsync方法内是通过tryLockInnerAsync方法进行锁的获取操作，

- tryLockInnerAsync方法内则是通过执行lua脚本进行锁的操作；

> <font color=blue>ps. lua脚本是redis内置的轻量化脚本，</font>
>
> <font color=blue>而redis的脚本执行是原子性的，脚本执行期间不会执行其他指令，一个脚本执行完才会执行下一个</font>

```java
 <T> RFuture<T> tryLockInnerAsync(long waitTime, long leaseTime, TimeUnit unit, long threadId, RedisStrictCommand<T> command) {
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, command,
                    // 如果key不存在   
                    "if (redis.call('exists', KEYS[1]) == 0) then " +
                        // 新增锁->线程id对应的value（计数器）设为1      
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        // 设置过期时间      
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                    // 如果key和线程id都存在（可重入锁）         
                    "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                        // 线程id对应的计数器自增      
                        "redis.call('hincrby', KEYS[1], ARGV[2], 1); " +
                        // 重置过期时间
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return nil; " +
                        "end; " +
                    "return redis.call('pttl', KEYS[1]);",
                Collections.singletonList(getRawName()), unit.toMillis(leaseTime), getLockName(threadId));
    }
```

><font color=blue>小结：也就是通过hash结构维护了加锁机制</font>

## 三、解锁原理

1.进入unclock方法-->unlockAsync-->unlockInnerAsync方法

```java
protected RFuture<Boolean> unlockInnerAsync(long threadId) {
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                     // 如果key和线程id不存在，返回空
                    "if (redis.call('hexists', KEYS[1], ARGV[3]) == 0) then " +
                        "return nil;" +
                        "end; " +
                     // 计数器减1        
                    "local counter = redis.call('hincrby', KEYS[1], ARGV[3], -1); " +
                     // 减1后如果计数器还是大于0         
                    "if (counter > 0) then " +
                        // 重设过期时间      
                        "redis.call('pexpire', KEYS[1], ARGV[2]); " +
                        "return 0; " +
                    "else " +
                        // 如果计数器小于等于0
                        // 删除该key      
                        "redis.call('del', KEYS[1]); " +
                        // 发出解锁信息      
                        "redis.call('publish', KEYS[2], ARGV[1]); " +
                        "return 1; " +
                        "end; " +
                    "return nil;",
                Arrays.asList(getRawName(), getChannelName()), LockPubSub.UNLOCK_MESSAGE, internalLockLeaseTime, getLockName(threadId));
    }
```

> <font color=blue>小结：通过计数器解锁，并通过LockPubSub通知其他线程解锁状态</font>

## 四、订阅消息

1.通过tryLock(long time, TimeUnit unit)方法没有获取锁的线程，会进行等待，同时在监听到事件后通过LockPubSub进行操作；

```java
@Override
    protected void onMessage(RedissonLockEntry value, Long message) {
        if (message.equals(UNLOCK_MESSAGE)) {
            Runnable runnableToExecute = value.getListeners().poll();
            if (runnableToExecute != null) {
                runnableToExecute.run();
            }

            value.getLatch().release();
        } else if (message.equals(READ_UNLOCK_MESSAGE)) {
            while (true) {
                Runnable runnableToExecute = value.getListeners().poll();
                if (runnableToExecute == null) {
                    break;
                }
                runnableToExecute.run();
            }

            value.getLatch().release(value.getLatch().getQueueLength());
        }
    }
```

> <font color=blue>小结：Redisson通过**LockPubSub**监听解锁消息，执行监听回调和释放信号量通知等待线程可以重新抢锁</font>

2.回到tryLock(long time, TimeUnit unit)方法

```java
// 订阅释放锁消息
RFuture<RedissonLockEntry> subscribeFuture = subscribe(threadId);
// 如果await为false，取消订阅，并返回获取锁失败
if (!subscribeFuture.await(time, TimeUnit.MILLISECONDS)) {
            if (!subscribeFuture.cancel(false)) {
                subscribeFuture.onComplete((res, e) -> {
                    if (e == null) {
                        unsubscribe(subscribeFuture, threadId);
                    }
                });
            }
            acquireFailed(waitTime, unit, threadId);
            return false;
}
// 如果为ture，进入循环尝试获取锁
try {
   					//计算获取锁的总耗时，如果大于等于最大等待时间，则获取锁失败
            time -= System.currentTimeMillis() - current;
            if (time <= 0) {
                acquireFailed(waitTime, unit, threadId);
                return false;
            }
        
            while (true) {
                long currentTime = System.currentTimeMillis();
              	// 尝试获取锁
                ttl = tryAcquire(waitTime, leaseTime, unit, threadId);
                // lock acquired
                if (ttl == null) {
                    return true;
                }
								// 超过最大等待时间则返回false结束循环，获取锁失败
                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(waitTime, unit, threadId);
                    return false;
                }
								// 阻塞等待锁（通过信号量(共享锁)阻塞,等待解锁消息）
                // waiting for message
                currentTime = System.currentTimeMillis();
                if (ttl >= 0 && ttl < time) {
                    subscribeFuture.getNow().getLatch().tryAcquire(ttl, TimeUnit.MILLISECONDS);
                } else {
                    subscribeFuture.getNow().getLatch().tryAcquire(time, TimeUnit.MILLISECONDS);
                }

                time -= System.currentTimeMillis() - currentTime;
                if (time <= 0) {
                    acquireFailed(waitTime, unit, threadId);
                    return false;
                }
            }
} finally {
  					// 无论是否获得锁,都要取消订阅解锁消息
            unsubscribe(subscribeFuture, threadId);
 }
```

## 五、续约

> watchDog看门狗

> 打个比方，你自己写了个redis锁，
> 
> 1.你没有设置锁的默认过期时间，那么当这个线程死掉了就会造成死锁;
> 
> 2.你设置了一个默认过期时间，那么有可能会导致线程还没运行完成，锁失效了;
> 
> 于是这个时候你就想到了要一个能监控这个线程的dog，在线程未完成运行时，给他的锁续个命;
> 
> 如果这个线程死掉了，那么在过期时间后，其他线程也可以活的锁，就不会造成死锁;

1.续约

```java
private void renewExpiration() {
        ExpirationEntry ee = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (ee == null) {
            return;
        }
        
        Timeout task = commandExecutor.getConnectionManager().newTimeout(new TimerTask() {
            @Override
            public void run(Timeout timeout) throws Exception {
                ExpirationEntry ent = EXPIRATION_RENEWAL_MAP.get(getEntryName());
                if (ent == null) {
                    return;
                }
                Long threadId = ent.getFirstThreadId();
                if (threadId == null) {
                    return;
                }
                
                RFuture<Boolean> future = renewExpirationAsync(threadId);
                future.onComplete((res, e) -> {
                    if (e != null) {
                        log.error("Can't update lock " + getRawName() + " expiration", e);
                        EXPIRATION_RENEWAL_MAP.remove(getEntryName());
                        return;
                    }
                    
                    if (res) {
                        // reschedule itself
                        renewExpiration();
                    } else {
                        cancelExpirationRenewal(null);
                    }
                });
            }
        }, internalLockLeaseTime / 3, TimeUnit.MILLISECONDS);
        
        ee.setTimeout(task);
    }
```

- 通过lua脚本进行续时

```java
protected RFuture<Boolean> renewExpirationAsync(long threadId) {
        return evalWriteAsync(getRawName(), LongCodec.INSTANCE, RedisCommands.EVAL_BOOLEAN,
                "if (redis.call('hexists', KEYS[1], ARGV[2]) == 1) then " +
                        "redis.call('pexpire', KEYS[1], ARGV[1]); " +
                        "return 1; " +
                        "end; " +
                        "return 0;",
                Collections.singletonList(getRawName()),
                internalLockLeaseTime, getLockName(threadId));
    }
```

2.解除续约(回到解锁方法unlockAsync-->cancelExpirationRenewal)

```java
protected void cancelExpirationRenewal(Long threadId) {
        ExpirationEntry task = EXPIRATION_RENEWAL_MAP.get(getEntryName());
        if (task == null) {
            return;
        }
        
        if (threadId != null) {
            task.removeThreadId(threadId);
        }

        if (threadId == null || task.hasNoThreads()) {
            Timeout timeout = task.getTimeout();
            if (timeout != null) {
                // 取消定时器
                timeout.cancel();
            }
            EXPIRATION_RENEWAL_MAP.remove(getEntryName());
        }
    }
```

> <font color=blue>小结：Redisson锁续约是使用定时器实现的，通过维护定时器来保证续约逻辑；</font>

## 六、总结

打个比方：

1. A、B线程争抢一把锁，A获取到后，B阻塞
2. B线程阻塞时并非主动CAS，而是PubSub方式订阅该锁的广播消息
3. A操作完成释放了锁，B线程收到订阅消息通知
4. B被唤醒开始继续抢锁，拿到锁