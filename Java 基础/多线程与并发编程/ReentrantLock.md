# ReentrantLock

![](E:\myworkspace\Cliff-of-Thinking-Java\Java 基础\images\AQSdepedents.PNG)

**性质**:

+ 独占锁(排它锁):只能有一个线程获取锁

+ 重入锁:一个线程可以多次lock()
+ 公平/非公平锁:只针对上锁过程
  + 非公平锁:尝试获取锁,若成功立刻返回,失败则加入同步队列
  + 公平锁:直接加入同步队列

## ReentrantLock实现思路

同步类在实现时一般都将自定义同步器（`sync`）定义为内部类，供自己使用；而同步类自己（`ReentrantLock`）则实现某个接口(`Lock`)，对外服务。当然，接口的实现要直接依赖`sync`，它们在语义上也存在某种对应关系！！而sync只用实现资源state的获取-释放方式tryAcquire-tryRelelase，至于线程的排队、等待、唤醒等，上层的AQS都已经实现好了，我们不用关心。

同步类的实现方式都差不多，不同的地方就在获取-释放资源的方式tryAcquire-tryRelelase。

## Lock接口

```java
public interface Lock {
	//上锁(不响应Thread.interrupt()直到获取锁)
    void lock();
	//上锁(响应Thread.interrupt())
    void lockInterruptibly() throws InterruptedException;
	//尝试获取锁(以nonFair方式获取锁)
    boolean tryLock();
  	//在指定时间内尝试获取锁(响应Thread.interrupt(),支持公平/二阶段非公平)
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
	//解锁
    void unlock();
	//获取Condition
    Condition newCondition();
}
```

## Lock()

```Java
public void lock() {
	sync.lock();
}
```

**关于sync的实现**

在`ReentrantLock`中, `Sync`是一个抽象类, 有两个子类: `NonfairSync`和`FairSync`. 而变量 sync 是哪一种类型取决于`ReentrantLock`实例在构造时, 看构造器:

```Java
public ReentrantLock() {
    sync = new NonfairSync();	// 默认非公平锁
}

public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();	//可传入true指定公平锁;
}

```

**公平锁**

**ReentrantLock.FairSync.lock()**

```java
final void lock() {
	acquire(1);
}
```

`acquire(1);`使用的是父类`AbstractQueuedSynchronizer`的`acquire(int arg)` 方法:

```Java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

**非公平锁**

**ReentrantLock.NonFairSync.lock()**

```Java
final void lock() {
    // 使用CAS方式在acquire()之前先尝试获取锁
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```

而`compareAndSetState` `setExclusiveOwnerThread` 和 `acquire`均为父类实现.

**AbstractQueuedSynchronizer.acquire(int arg)**

可以看出来公平锁`FairSync.lock()`和非公平锁`NonFairSync.lock()`最终都会调用父类的父类`AbstractQueuedSynchronizer`的`acquire(int arg)` 方法; 

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&		// 尝试获取锁(失败);
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))	// 将该线程封装成Node节点放在线程等待队列尾部,等待是自旋的过程,获取资源后返回.
        selfInterrupt();	
}
```

函数流程如下：

+ tryAcquire()尝试直接去获取资源，如果成功则直接返回（这里体现了非公平锁，每个线程获取锁时会尝试直接抢占加塞一次，而CLH队列中可能还有别的线程在等待）；

+ addWaiter()将该线程加入等待队列的尾部，并标记为独占模式；

+ acquireQueued()使线程阻塞在等待队列中获取资源，一直获取到资源后才返回。如果在整个等待过程中被中断过，则返回true，否则返回false。

+ 如果线程在等待过程中被中断过，它是不响应的。只是获取资源后才再进行自我中断selfInterrupt()，将中断补上。

  

两次都会调用`tryAcquire(arg)`尝试获取锁, 但该方法在`AbstractQueuedSynchronizer`类中是不可用的, 因为具体实现要在根据不同的锁的要求在其子类**自定义同步器**进行重写.

**ReentrantLock.FairSync**

```java
 #java.util.concurrent.locks.ReentrantLock.FairSync
 protected final boolean tryAcquire(int acquires) {
 	//获取当前线程
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {//当前锁没被占用
 	   if (!hasQueuedPredecessors() &&//1.判断同步队列中是否有节点在等待
 		   compareAndSetState(0, acquires)) {//2.如果上面!1成立,修改state值(表明当前锁已被占用)
 		   setExclusiveOwnerThread(current);//3.如果2成立,修改当前占用锁的线程为当前线程
 		   return true;
 	   }
    }
    else if (current == getExclusiveOwnerThread()) {//占用锁线程==当前线程(重入)
 	   int nextc = c + acquires;//
 	   if (nextc < 0)
 		   throw new Error("Maximum lock count exceeded");
 	   setState(nextc);//修改status
 	   return true;
    }
    return false;//直接获取锁失败
}

```

**ReentrantLock.NonfairSync**

```java
#java.util.concurrent.locks.ReentrantLock.NonfairSync
protected final boolean tryAcquire(int acquires) {
 	return nonfairTryAcquire(acquires);
 }
 
#java.util.concurrent.locks.ReentrantLock.Sync
 final boolean nonfairTryAcquire(int acquires) {//这个过程其实和FairSync.tryAcquire()基本一致
	final Thread current = Thread.currentThread();
	int c = getState();
	if (c == 0) {
		//唯一区别: 这里不会去判断队列中是否为空
		if (compareAndSetState(0, acquires)) {
			setExclusiveOwnerThread(current);
			return true;
		}
	}
	else if (current == getExclusiveOwnerThread()) {
		int nextc = c + acquires;
		if (nextc < 0) // overflow
			throw new Error("Maximum lock count exceeded");
		setState(nextc);
		return true;
	}
	return false;
}

```

**unLock()过程**

```java
#java.util.concurrent.locks.ReentrantLock
public void unlock() {
	sync.release(1);
}


#java.util.concurrent.locks.AbstractQueuedSynchronizer
public final boolean release(int arg) {
    if (tryRelease(arg)) {		// 释放锁
        Node h = head;
        if (h != null &&		// head节点为空(非公平锁直接获取锁)
        h.waitStatus != 0)
            unparkSuccessor(h);	// 唤醒同步队列中离head最近的一个waitStatus<=0的节点
        return true;
    }
    return false;
}


#java.util.concurrent.locks.ReentrantLock.Sync
protected final boolean tryRelease(int releases) {
	int c = getState() - releases;
	//持有锁的线程==当前线程
	if (Thread.currentThread() != getExclusiveOwnerThread())
		throw new IllegalMonitorStateException();
	boolean free = false;
	if (c == 0) {		//重入锁全部释放
		free = true;
		//置空持有锁线程
		setExclusiveOwnerThread(null);
	}
	//state==0(此时持有锁,不用cas)
	setState(c);
	return free;
}

```

**tryRelease(int releases)**方法是在抽象类`Sync` 中实现, 两个子类并未重写. 所以两种锁的释放过程是一样的.

## lockInterruptibly()过程

lockInterruptibly()与lock()过程基本相同,区别在于Thread.intterpt()的应对措施不同

```java
//lock()
final boolean acquireQueued(final Node node, int arg) {
	boolean failed = true;
	try {
		//表示是否被打断
		boolean interrupted = false;
		for (;;) {
			//获取node.pre节点
			final Node p = node.predecessor();
			if (p == head //当前节点是否是同步队列中的第二个节点
			&& tryAcquire(arg)) {//获取锁,当前head指向当前节点
				setHead(node);//head=head.next
				p.next = null;//置空 
				failed = false;
				return interrupted;
			}

			if (shouldParkAfterFailedAcquire(p, node) && //是否空转(因为空转唤醒是个耗时操作,进入空转前判断pre节点状态.如果pre节点即将释放锁,则不进入空转)
				parkAndCheckInterrupt())//利用unsafe.park()进行空转(阻塞)
				interrupted = true;//如果Thread.interrupt()被调用,(不会真的被打断,会继续循环空转直到获取到锁)
		}
	} finally {
		if (failed)//tryAcquire()过程出现异常导致获取锁失败,则移除当前节点
			cancelAcquire(node);
	}
}
// lockInterruptibly()
private void doAcquireInterruptibly(int arg)
	throws InterruptedException {
	final Node node = addWaiter(Node.EXCLUSIVE);
	boolean failed = true;
	try {
		for (;;) {
			final Node p = node.predecessor();
			if (p == head && tryAcquire(arg)) {
				setHead(node);
				p.next = null; 
				failed = false;
				return;
			}
			if (shouldParkAfterFailedAcquire(p, node) &&
				parkAndCheckInterrupt())//唯一区别当Thread.intterpt()打断时,直接抛出异常
				throw new InterruptedException();
		}
	} finally {
		if (failed)//然后移除当前节点
			cancelAcquire(node);
	}
}

```

## tryLock()

```java
#java.util.concurrent.locks.ReentrantLock
public boolean tryLock() {
	//尝试获取非公平锁
	return sync.nonfairTryAcquire(1);
}

```

## tryLock(long timeout, TimeUnit unit)

```java
#java.util.concurrent.locks.ReentrantLock
public boolean tryLock(long timeout, TimeUnit unit)
		throws InterruptedException {
	return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}


#java.util.concurrent.locks.AbstractQueuedSynchronizer
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
		throws InterruptedException {
	if (Thread.interrupted())
		throw new InterruptedException();
	return tryAcquire(arg) ||//获取锁(公平/非公平)
		doAcquireNanos(arg, nanosTimeout);//在指定时间内等待锁(空转)
}


private boolean doAcquireNanos(int arg, long nanosTimeout)
		throws InterruptedException {
	...
	final long deadline = System.nanoTime() + nanosTimeout;
	//加入队尾
	final Node node = addWaiter(Node.EXCLUSIVE);
	boolean failed = true;
	try {
		for (;;) {
			final Node p = node.predecessor();
			if (p == head && tryAcquire(arg)) {
				setHead(node);
				p.next = null; 
				failed = false;
				return true;
			}
		  //上面与acquireQueued()相同,重点看这里
		  //计算剩余时间
			nanosTimeout = deadline - System.nanoTime();
			if (nanosTimeout <= 0L)
				return false;
			if (shouldParkAfterFailedAcquire(p, node) &&
				nanosTimeout > spinForTimeoutThreshold)
				//利用parkNanos()指定空转时间
				LockSupport.parkNanos(this, nanosTimeout);
			if (Thread.interrupted())//如果被Thread.interrupt(),则抛异常
				throw new InterruptedException();
		}
	} finally {
		if (failed)//移除节点
			cancelAcquire(node);
	}
}

```



## newCondition()

```java
public Condition newCondition() {
	return sync.newCondition();
}
#java.util.concurrent.locks.ReentrantLock.Sync
final ConditionObject newCondition() {
	return new ConditionObject();
}
```

