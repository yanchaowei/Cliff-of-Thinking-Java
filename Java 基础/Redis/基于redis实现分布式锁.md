# 基于redis实现分布式锁

## 一、概述

> SET resource_name my_random_value NX PX 30000

主要依靠上述命令，该命令仅当 Key 不存在时（NX保证）set 值，并且设置过期时间 3000ms （PX保证），值 my_random_value 必须是所有 client 和所有锁请求发生期间唯一的，释放锁的逻辑是：

```
if redis.call("get",KEYS[1]) == ARGV[1] then
    return redis.call("del",KEYS[1])
else
    return 0
end
```

上述实现可以避免释放另一个client创建的锁，如果只有 del 命令的话，那么如果 client1 拿到 lock1 之后因为某些操作阻塞了很长时间，此时 Redis 端 lock1 已经过期了并且已经被重新分配给了 client2，那么 client1 此时再去释放这把锁就会造成 client2 原本获取到的锁被 client1 无故释放了，但现在为每个 client 分配一个 unique 的 string 值可以避免这个问题。至于如何去生成这个 unique string，方法很多随意选择一种就行了。

不懂没关系，下面我们用代码来一点点解释。

## 二、分布式锁的实现要点

 为了实现分布式锁，需要确保锁同时满足以下四个条件：

1. 互斥性。在任意时刻，只有一个客户端能持有锁。
2. 不会发送死锁。即使一个客户端持有锁的期间崩溃而没有主动释放锁，也需要保证后续其他客户端能够加锁成功。
3. 加锁和解锁必须是同一个客户端，客户端自己不能把别人加的锁给释放了。
4. 容错性。只要大部分的Redis节点正常运行，客户端就可以进行加锁和解锁操作。

## 三、Redis实现分布式锁的错误实现方式

### 3.1 加锁错误姿势

在讲解使用Redis实现分布式锁的正确姿势之前，我们有必要来看下错误实现方式。

首先，为了保证**互斥性**和**不会发送死锁**2个条件，所以我们在加锁操作的时候，需要使用`SETNX`指令来保证互斥性——只有一个客户端能够持有锁。为了保证不会发送死锁，需要给锁加一个过期时间，这样就可以保证即使持有锁的客户端期间崩溃了也不会一直不释放锁。

为了保证这2个条件，有些人错误的实现会用如下代码来实现加锁操作：

```java
	/**
     * 实现加锁的错误姿势
     * @param jedis
     * @param lockKey
     * @param requestId
     * @param expireTime
     */
public static void wrongGetLock1(Jedis jedis, String lockKey, String requestId, int expireTime) {
    Long result = jedis.setnx(lockKey, requestId);
    if (result == 1) {
        // 若在这里程序突然崩溃，则无法设置过期时间，将发生死锁
        jedis.expire(lockKey, expireTime);
    }
}
```

对于新手同学来说，乍一看好像没什么问题。但是我们细品一下，setnx 和expire是两条Redis指令，不具备原子性，如果程序在执行完setnx之后突然崩溃，导致没有设置锁的过期时间，从而就导致死锁了。因为这个客户端持有的所有不会被其他客户端释放，持有锁的客户端又崩溃了，也不会主动释放。从而该锁永远不会释放，导致其他客户端也获得不能锁。从而其他客户端一直阻塞**。所以针对该代码正确姿势应该保证setnx和expire原子性**。

实现加锁操作的错误姿势2。具体实现如下代码所示：

```java
	/**
     * 实现加锁的错误姿势2
     * @param jedis
     * @param lockKey
     * @param expireTime
     * @return
     */
    public static boolean wrongGetLock2(Jedis jedis, String lockKey, int expireTime) {
        long expires = System.currentTimeMillis() + expireTime;
        String expiresStr = String.valueOf(expires);
        // 如果当前锁不存在，返回加锁成功
        if (jedis.setnx(lockKey, expiresStr) == 1) {
            return true;
        }

        // 如果锁存在，获取锁的过期时间
        String currentValueStr = jedis.get(lockKey);
        if (currentValueStr != null && Long.parseLong(currentValueStr) < System.currentTimeMillis()) {
            // 锁已过期，获取上一个锁的过期时间，并设置现在锁的过期时间
            // 注意，一只bug小可爱出现了，当即使判断出来锁被一个用户释放，又被另一个人获取，但是你已经把人家的过期时间覆盖掉了(set)，还不想负责，不是耍流氓吗？
            String oldValueStr = jedis.getSet(lockKey, expiresStr);
            if (oldValueStr != null && oldValueStr.equals(currentValueStr)) {
                // 考虑多线程并发的情况，只有一个线程的设置值和当前值相同，它才有权利加锁
                return true;
            }
        }
        // 其他情况，一律返回加锁失败
        return false;
    }
```

这个加锁操作咋一看没有毛病对吧。那以上这段代码的问题毛病出在哪里呢？

1. 由于客户端自己生成过期时间，所以需要强制要求分布式环境下所有客户端的时间必须同步。

2. 当锁过期的时候，如果多个客户端同时执行jedis.getSet()方法，虽然最终只有一个客户端加锁，但是这个客户端的锁的过期时间可能被其他客户端覆盖。不具备加锁和解锁必须是同一个客户端的特性。**解决上面这段代码的方式就是为每个客户端加锁添加一个唯一标示，已确保加锁和解锁操作是来自同一个客户端。**

### 3.2 解锁错误姿势

分布式锁的实现无法就2个方法，一个加锁，一个就是解锁。下面我们来看下解锁的错误姿势。

错误姿势1.

```java
	/**
     * 解锁错误姿势1
     * @param jedis
     * @param lockKey
     */
    public static void wrongReleaseLock1(Jedis jedis, String lockKey) {
        jedis.del(lockKey);
    }
```

这是最简单的解锁方式，直接一个`del()`，但是你想想啊，你要应对的是高并发呀，锁本来就是“抢“来的，你忍心被人家随随便便释放吗？

这种不先判断拥有者而直接解锁的方式，会导致任何客户端都可以随时解锁。即使这把锁不是它上锁的。

```java
	/**
     * 解锁错误姿势2
     * @param jedis
     * @param lockKey
     * @param requestId
     */
    public static void wrongReleaseLock2(Jedis jedis, String lockKey, String requestId) {

        // 判断加锁与解锁是不是同一个客户端
        if (requestId.equals(jedis.get(lockKey))) {
            // 若在此时，这把锁突然不是这个客户端的，则会误解锁
            jedis.del(lockKey);
        }
    }
```

这里判断了拥有者，那错误原因又在哪里呢？答案又是原子性上面。因为判断和删除不是一个原子性操作。在并发的时候很可能发生解除了别的客户端加的锁。具体场景有：客户端A加锁，一段时间之后客户端A进行解锁操作时，在执行`jedis.del()`之前，锁突然过期了，此时客户端B尝试加锁成功，然后客户端A再执行del方法，则客户端A将客户端B的锁给解除了。从而不也不满足加锁和解锁必须是同一个客户端特性。**解决思路就是需要保证GET和DEL操作在一个事务中进行，保证其原子性。**

## 四、Redis实现分布式锁的正确姿势

刚刚介绍完了错误的姿势后，从上面错误姿势中，我们可以知道，要使用Redis实现分布式锁。

**加锁操作的正确姿势为：**

1. 使用setnx命令保证互斥性
2. 需要设置锁的过期时间，避免死锁
3. setnx和设置过期时间需要保持原子性，避免在设置setnx成功之后在设置过期时间客户端崩溃导致死锁
4. 加锁的Value 值为一个唯一标示。可以采用UUID作为唯一标示。加锁成功后需要把唯一标示返回给客户端来用来客户端进行解锁操作

**解锁操作的正确姿势为：**

1. 需要拿加锁成功的唯一标示要进行解锁，从而保证加锁和解锁的是同一个客户端

2. 解锁操作需要比较唯一标示是否相等，相等再执行删除操作。这2个操作可以采用Lua脚本方式使2个命令的原子性。

Redis分布式锁实现的正确姿势的实现代码：

先定义一个**锁接口**

```java
public interface DistributedLock {
    /**
     * 获取锁
     * @return 锁标识
     */
    String acquire();

    /**
     * 释放锁
     * @param indentifier
     * @return
     */
    boolean release(String indentifier);
}
```

**锁实现**

```java
/**
 * @Description
 * @created 2019/1/1 20:32
 */
@Slf4j
public class RedisDistributedLock implements DistributedLock{

    private static final String LOCK_SUCCESS = "OK";
    private static final Long RELEASE_SUCCESS = 1L;
    private static final String SET_IF_NOT_EXIST = "NX";
    private static final String SET_WITH_EXPIRE_TIME = "PX";

    /**
     * redis 客户端
     */
    private Jedis jedis;

    /**
     * 分布式锁的键值
     */
    private String lockKey;

    /**
     * 锁的超时时间 10s
     */
    int expireTime = 10 * 1000;

    /**
     * 锁等待，防止线程饥饿
     */
    int acquireTimeout  = 1 * 1000;

    /**
     * 获取指定键值的锁
     * @param jedis jedis Redis客户端
     * @param lockKey 锁的键值
     */
    public RedisDistributedLock(Jedis jedis, String lockKey) {
        this.jedis = jedis;
        this.lockKey = lockKey;
    }

    /**
     * 获取指定键值的锁,同时设置获取锁超时时间
     * @param jedis jedis Redis客户端
     * @param lockKey 锁的键值
     * @param acquireTimeout 获取锁超时时间
     */
    public RedisDistributedLock(Jedis jedis,String lockKey, int acquireTimeout) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.acquireTimeout = acquireTimeout;
    }

    /**
     * 获取指定键值的锁,同时设置获取锁超时时间和锁过期时间
     * @param jedis jedis Redis客户端
     * @param lockKey 锁的键值
     * @param acquireTimeout 获取锁超时时间
     * @param expireTime 锁失效时间
     */
    public RedisDistributedLock(Jedis jedis, String lockKey, int acquireTimeout, int expireTime) {
        this.jedis = jedis;
        this.lockKey = lockKey;
        this.acquireTimeout = acquireTimeout;
        this.expireTime = expireTime;
    }

    @Override
    public String acquire() {
        try {
            // 获取锁的超时时间，超过这个时间则放弃获取锁
            long end = System.currentTimeMillis() + acquireTimeout;
            // 随机生成一个value
            String requireToken = UUID.randomUUID().toString();
            while (System.currentTimeMillis() < end) {
                // 就是上面的：SET resource_name my_random_value NX PX 30000
                String result = jedis.set(lockKey, requireToken, SET_IF_NOT_EXIST, SET_WITH_EXPIRE_TIME, expireTime);
                if (LOCK_SUCCESS.equals(result)) {
                    return requireToken;
                }
                try {
                    // 如果加锁失败，休息一会儿再尝试
                    Thread.sleep(100);
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        } catch (Exception e) {
            log.error("acquire lock due to error", e);
        }

        return null;
    }

    @Override
    public boolean release(String identify) {
        if(identify == null){
            return false;
        }

        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = new Object();
        try {
            result = jedis.eval(script, Collections.singletonList(lockKey),
                                Collections.singletonList(identify));
            if (RELEASE_SUCCESS.equals(result)) {
                log.info("release lock success, requestToken:{}", identify);
                return true;
            }}catch (Exception e){
            log.error("release lock due to error",e);
        }finally {
            if(jedis != null){
                jedis.close();
            }
        }

        log.info("release lock failed, requestToken:{}, result:{}", identify, result);
        return false;
    }
}
```

**锁测试**

```java
// 下面就以秒杀库存数量为场景，测试下上面实现的分布式锁的效果。具体测试代码如下：

public class RedisDistributedLockTest {
    static int n = 500;
    public static void secskill() {
        System.out.println(--n);
    }

    public static void main(String[] args) {
        Runnable runnable = () -> {
            RedisDistributedLock lock = null;
            String unLockIdentify = null;
            try {
                Jedis conn = new Jedis("127.0.0.1",6379);
                lock = new RedisDistributedLock(conn, "test1");
                unLockIdentify = lock.acquire();
                System.out.println(Thread.currentThread().getName() + "正在运行");
                secskill();
            } finally {
                if (lock != null) {
                    lock.release(unLockIdentify);
                }
            }
        };

        for (int i = 0; i < 10; i++) {
            Thread t = new Thread(runnable);
            t.start();
        }
    }
}
```

上面的实现代码只是针对单机的Redis没问题。但是现实生产中大部分都是集群的或者是主备的。但上面的实现姿势在集群或者主备情况下会有相应的问题。分布式锁需要lua脚本保证。

https://g.co/doodle/7h73n

参考：https://www.cnblogs.com/zhili/p/redisdistributelock.html