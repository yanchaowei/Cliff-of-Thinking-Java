# Java并发：同步工具类详解（CountDownLatch、CyclicBarrier、Semaphore）

## 概述

同步工具类可以是任何一个对象，只要它根据其自身的状态来协调线程的控制流。阻塞队列可以作为同步工具类，其他类型的同步工具类还包括**信号量**（Semaphore）、**栅栏**（Barrier）以及**闭锁**（Latch）。本文就目前常用的3种同步工具类进行简单介绍。

## 闭锁

闭锁是一种同步工具类，可以延迟线程的进度直到其到达终止状态。闭锁的作用相当于一扇门：在闭锁到达结束状态之前，这扇门一直是关闭的，并且没有任何线程能够通过，当到达结束状态时，这扇门会打来并允许所有的线程通过。当闭锁到达结束状态后，将不会再改变状态，因此这扇门将永远保持打开状态。闭锁可以用来确保某些活动直到其他活动都完成后才继续执行，例如：

+ 确保某个计算在其需要的所有资源都被初始化之后才继续执行。
+ 确保某个服务在其依赖的所有其他服务都已经启动之后才启动。
+ 等到直到直到某个操作的所有参与者（例如，在多玩家游戏中的所有玩家）都就绪再继续执行。

`CountDownLatch` 是一种灵活的闭锁实现。

+ 闭锁状态包括一个**计数器**，该计数器被初始化为一个正数，表示需要等待的事件数量。
+ `countDown` 方法递减计数器，表示有一个事件已经发生了，
+ 而 `await` 方法等待计数器达到零，这表示所有需要等待的事件都已发生。
+ 如果计数器的值非零，那么await方法会一直阻塞直到计算器为零，或者等待中的线程中断，或者超时。

**下面是CountDownLatch的一个简单例子：**

```java

public class CountDownLatchTest {

    static CountDownLatch timeOutCountDownLatch = new CountDownLatch(1);

    public static void main(String args[]) {
        try {
            new Driver(10);
            testAwaitTimtOut();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    // 测试带超时的await方法
    public static void testAwaitTimtOut() throws InterruptedException {
        System.out.println("before await(long timeout, TimeUnit unit)");
        timeOutCountDownLatch.await(3, TimeUnit.SECONDS);   //等待超时时间为3秒
        System.out.println("after await(long timeout, TimeUnit unit)");
    }

}

class Driver {
    public Driver(int N) throws InterruptedException {
        CountDownLatch startSignal = new CountDownLatch(1); // 定义一个CountDownLatch, 计数器值为1, 也就是每次await(), 需要执行1次countDown(), 才能继续执行await()外面的代码
        CountDownLatch doneSignal = new CountDownLatch(N);  // 定义一个CountDownLatch, 计数器值为N, 也就是每次await(), 需要执行N次countDown(), 才能继续执行await()外面的代码

        for (int i = 0; i < N; ++i) {
            // 创建并启动线程
            new Thread(new Worker(startSignal, doneSignal)).start();
        }
        Thread.sleep(2000); // 睡眠2秒, 可以看到10个线程都在等待startSignal.countDown()执行
        System.out.println();
        startSignal.countDown(); // 解除所有线程的阻塞
        doneSignal.await(); // 等待所有线程执行doneSignal.countDown(), 才通过
        System.out.println();
        System.out.println("Main thread-after:doneSignal.await()------");
    }
}

class Worker implements Runnable {
    private final CountDownLatch startSignal;
    private final CountDownLatch doneSignal;

    Worker(CountDownLatch startSignal, CountDownLatch doneSignal) {
        this.startSignal = startSignal;
        this.doneSignal = doneSignal;
    }

    public void run() {
        try {
            System.out.println(Thread.currentThread().getName() + "-before:startSignal.await()");
            startSignal.await();    // 线程会在此处等待, 直到startSignal.countDown()执行
            System.out.println(Thread.currentThread().getName() + "-after:startSignal.await()");
            doneSignal.countDown();
        } catch (InterruptedException ex) {
        } // return;
    }
}
 
```

## 栅栏

栅栏（Bariier）类似于闭锁，它能阻塞一组线程知道某个事件发生。栅栏与闭锁的关键区别在于，所有的线程必须同时到达栅栏位置，才能继续执行。闭锁用于等待等待时间，而栅栏用于等待线程。栅栏用于实现一些协议，例如几个家庭决定在某个地方集合：“所有人6：00在麦当劳碰头，到了以后要等其他人，之后再讨论下一步要做的事情”。

`CyclicBarrier` 可以使一定数量的参与方反复的在栅栏位置汇聚，它在并行迭代算法中非常有用：将一个问题拆成一系列相互独立的子问题。当线程到达栅栏位置时，调用await() 方法，这个方法是阻塞方法，直到所有线程到达了栅栏位置，那么栅栏被打开，此时所有线程被释放，而栅栏将被重置以便下次使用。如果对await的调用超时，或者await阻塞的线程被中断，那么栅栏就认为是打破了，所有阻塞的await调用都将终止并抛出BrokenBarrierException。CycleBarrier还可以使你将一个栅栏操作传递给构造函数，这是一个Runnable，当成功通过栅栏会（在一个子任务线程中）执行它，但在阻塞线程被释放之前是不能执行的。

下面是CycleBarrier的一个简单例子：

```java
class Solver {
	final CyclicBarrier barrier;
 
	class Worker implements Runnable {
		public void run() {
			try {
				System.out.println(Thread.currentThread().getName() + ":before barrier.await()");
				Thread.sleep(1000); // 睡眠1秒, 便于更好的观察
				barrier.await();    // 当所有线程到达此处时, 栅栏打开, 先执行定义栅栏时自带的Runnable方法, 所有线程得以往下执行    
				System.out.println(Thread.currentThread().getName() + ":after barrier.await()");
			} catch (InterruptedException ex) {
				return;
			} catch (BrokenBarrierException ex) {
				return;
			}
		}
	}
 
	public Solver(int key) throws InterruptedException {
	    // 定义一个CyclicBarrier, 等待的线程数为key个, 并且自带一个Runnable方法
		barrier = new CyclicBarrier(key, new Runnable() {
			public void run() {
				try {
	                System.out.println("所有线程执行await()后, 执行本run方法, 也就是栅栏打开后会先执行本方法");
                    Thread.sleep(1000); // 睡眠1秒, 更好的观察此方法的执行顺序
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
			}
		});
		// 第一次: 创建key个线程执行, 与上文定义CyclicBarrier的数量相同（都为key）
		for (int i = 0; i < key ; ++i){
			new Thread(new Worker()).start();
		}
		Thread.sleep(3000);   // 睡眠3秒, 等待上面的所有线程执行完毕
		System.out.println();
		// 第二次: 创建key个线程执行, 测试栅栏是可以反复使用的
		for (int i = 0; i < key ; ++i){ // 
            new Thread(new Worker()).start();
        }
	}
 
}
public class CyclicBarrierTest {
	public static void main(String args[]) throws InterruptedException{
		new Solver(10);
	}
}
```

## 信号量

计数信号量（counting semaphore）用来控制同时访问某个特定资源的操作数量，或者同时执行某个制定操作的数量。计数信号量还可以实现某种资源池，或者对容器施加边界。
Semaphore中管理着一组虚拟的许可（permit），许可的初始数量可通过构造函数来指定。在执行操作时可以首先获得许可（只要还有剩余的许可），并在使用以后释放许可。如果没有许可，那么acquire将阻塞直到有许可（或者直到被中断或者操作超时）。release方法将返回一个许可给信号量。计算信号量的一种简化形式是二值信号量，即初始值为1的Semaphore。二值信号量可以用作互斥体（mutex），并具备不可重入的加锁语义：谁拥有这个唯一的许可，谁就拥有可互斥锁。

Semaphore可以用于实现资源池，例如数据库连接池。我们可以构造一个固定长度的资源池，当池为空时，请求资源将会失败，但你真正希望看到的行为是阻塞而不是失败，并且当池非空时解除阻塞。如果将Semaphore的计数值初始化为池的大小，并在从池中获取一个资源之前首先调用acquire方法获取一个许可，在将资源返回给池之后调用release释放许可，那么acquire将一直阻塞直到资源池不为空。

```java
package com.joonwhee.imp;
 
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
 
/**
 * Semaphore的简单例子
 * @author JoonWhee
 * @Date 2018年1月27日
 */
public class SemaphoreTest {
 
    public static final ExecutorService THREAD_POOL = Executors.newFixedThreadPool(5);
    // 定义一个信号量, 许可数量只有1, 也就是同时最多只能有1个线程访问, 可以通过修改许可数量来观察输出有什么不同
    private static final Semaphore available = new Semaphore(1);
 
    public static void doSomeThing() throws InterruptedException {
        available.acquire();    // 尝试获取一个许可, 阻塞直到有一个可用, 或者被打断
        // 获取许可用, 进行自己的逻辑处理, 此处只输出一句话
        System.out.println(Thread.currentThread().getName() + "-doSomeThing");  
        Thread.sleep(1000); // 为了看的更清楚, 睡眠1秒
        available.release();    // 释放许可
    }
 
    public static void main(String args[]) throws InterruptedException {
        // 开启5个线程执行doSomeThing()
        for (int i = 0; i < 5; i++) {
            THREAD_POOL.submit(new Runnable() {
                @Override
                public void run() {
                    try {
                        doSomeThing();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
        
    }
}
```

