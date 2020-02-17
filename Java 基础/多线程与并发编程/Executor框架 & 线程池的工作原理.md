# Executor框架 & 线程池的工作原理

## 概述

- 在Java 5之后，并发编程引入了一堆新的启动、调度和管理线程的API。
- 其内部使用了线程池机制它在 java.util.cocurrent 包下。
- 通过该框架来控制线程的启动、执行和关闭，可以简化并发编程的操作。
- **优点：**因此，在Java 5之后，通过Executor来启动线程比使用Thread的start方法更好，除了更易管理，效率更好（用线程池实现，节约开销）外，还有关键的一点：有助于避免this逃逸问题——如果我们在构造器中启动一个线程，因为另一个任务可能会在构造器结束之前开始执行，此时可能会访问到初始化了一半的对象用Executor在构造器中。

Executor框架包括：线程池，Executor，Executors，ExecutorService，CompletionService，Future，Callable等。

我们一步步进行讲解

### Executor

Executor接口中之只定义了一个方法：
`void execute(Runnable command);`
该方法接收一个 Runable 实例，它用来执行一个任务，任务即一个实现了 Runnable 接口的类。

### ExecutorService

ExecutorService 接口继承自 Executor 接口:

```java
void shutdown();

List<Runnable> shutdownNow();

boolean isShutdown();

boolean isTerminated();

<T> Future<T> submit(Callable<T> task);

<T> Future<T> submit(Runnable task, T result);

Future<?> submit(Runnable task);

...
```

- ExecutorService的生命周期包括三种状态：**运行、关闭、终止**。
  - 创建后便进入运行状态
  - 当调用了shutdown（）方法时，便进入关闭状态，此时意味着ExecutorService不再接受新的任务，但它还在执行已经提交了的任务，当素有已经提交了的任务执行完后，便到达终止状态。
  - 如果不调用shutdown（）方法，ExecutorService会一直处在运行状态，不断接收新的任务，执行新的任务，服务器端一般不需要关闭它，保持一直运行即可。
- `shutdown()`和 `shutdownNow()`**区别：**一个`ExecutorService`可以关闭，这将导致它拒绝新的任务。 提供了两种不同的方法来关闭ExecutorService 。 `shutdown()`方法将允许先前提交的任务在终止之前执行，而`shutdownNow()`方法可以防止等待任务启动并尝试停止当前正在执行的任务。 一旦终止，执行者没有任务正在执行，没有任务正在等待执行，并且不能提交新的任务。**因此我们一般用该接口来实现和管理多线程。**应关闭未使用的ExecutorService以允许资源的回收。

------

## Executors 和 四类线程池

线程池，从字面含义来看，是指管理一组同构工作线程的资源池。线程池是与工作队列密切相关的，其中在工作队列中保存了所有等待执行的任务。工作者线程的任务很简单：从工作队列中获取一个任务，执行任务，然后返回线程池并等待下一个任务。

“在线程池中执行任务“”比“为每个线程分配一个任务”优势更多。通过重用现有的线程而不是创建线程，可以在处理多个请求时分摊在线程创建和销毁过程中产生的巨大开销。另一个额外的好处是，当请求到达时，工作线程通常已经存在，因此不会由于等待创建线程而延迟任务的执行，从而提高了响应性。通过适当的调整线程池的大小，可以创建足够的线程以便使处理器保持忙碌状态，同时还可以防止过多线程相互竞争资源而使应用程序耗尽内存或失败。

`Executors`提供了一系列工厂方法用于创先线程池，返回的线程池为 ThreadPoolExecutor，实现了 ExecutorService 接口。下面看看集中线程池的区别：

| 线程池                        | 认识                                                         |
| ----------------------------- | ------------------------------------------------------------ |
| `newCachedThreadPool()`       | 创建一个根据需要创建新线程的线程池，但在可用时将重新使用以前构造的线程，并在需要时使用提供的ThreadFactory创建新线程。 |
| `newFixedThreadPool(int)`     | 创建一个线程池，该线程池重用固定数量的从共享无界队列中运行的线程。 |
| `newScheduledThreadPool(int)` | 创建一个线程池，可以调度命令在给定的延迟之后运行，或定期执行。 |
| `SingleThreadExecutor()`      | 创建一个使用从无界队列运行的单个工作线程的执行程序。         |

------

### 自定义线程池

自定义线程池，可以用` ThreadPoolExecutor `类创建，它有多个构造方法来创建线程池，我们先来看一下这个方法，其他几个线程的创建都是基于这个方法进行修改相关参数实现的。

```Java
public ThreadPoolExecutor (int corePoolSize, int maximumPoolSize, long         keepAliveTime, TimeUnit unit,BlockingQueue<Runnable> workQueue)
```

- corePoolSize：线程池中所保存的核心线程数，包括空闲线程。
- maximumPoolSize：池中允许的最大线程数。
- keepAliveTime：线程池中的空闲线程所能持续的最长时间。
- unit：持续时间的单位。
- workQueue：任务执行前保存任务的队列，仅保存由execute方法提交的Runnable任务。

当试图通过 `excute` 方法讲一个Runnable任务添加到线程池中时，按照如下顺序来处理：

- 1、如果线程池中的线程数量少于corePoolSize，即使线程池中有空闲线程，也会创建一个新的线程来执行新添加的任务；
- 2、如果线程池中的线程数量大于等于corePoolSize，但缓冲队列workQueue未满，则将新添加的任务放到workQueue中，按照FIFO的原则依次等待执行（线程池中有线程空闲出来后依次将缓冲队列中的任务交付给空闲的线程执行）；
- 3、如果线程池中的线程数量大于等于corePoolSize，且缓冲队列workQueue已满，但线程池中的线程数量小于maximumPoolSize，则会创建新的线程来处理被添加的任务；
- 4、如果线程池中的线程数量等于了maximumPoolSize，有4种才处理方式（该构造方法调用了含有5个参数的构造方法，并将最后一个构造方法为RejectedExecutionHandler类型，它在处理线程溢出时有4种方式，这里不再细说，要了解的，自己可以阅读下源码）。
- 总结起来，也即是说，当有新的任务要处理时，先看线程池中的线程数量是否大于corePoolSize，再看缓冲队列workQueue是否满，最后看线程池中的线程数量是否大于maximumPoolSize。
- 另外，当线程池中的线程数量大于corePoolSize时，如果里面有线程的空闲时间超过了keepAliveTime，就将其移除线程池，这样，可以动态地调整线程池中线程的数量。

**自定义线程池示例程序：**

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class ThreadPoolExecutorTest {
 
    /**
     * 创建一个线程池(完整入参): 
     * 核心线程数为5 (corePoolSize), 
     * 最大线程数为10 (maximumPoolSize), 
     * 存活时间为60分钟(keepAliveTime), 
     * 工作队列为LinkedBlockingQueue (workQueue),
     * 线程工厂为默认的DefaultThreadFactory (threadFactory), 
     * 饱和策略(拒绝策略)为AbortPolicy: 抛出异常(handler).
     */
    private static ExecutorService THREAD_POOL = new ThreadPoolExecutor(5, 10, 60, TimeUnit.MINUTES,
            new LinkedBlockingQueue<Runnable>(), Executors.defaultThreadFactory(),
            new ThreadPoolExecutor.AbortPolicy());
 
    /**
     * 只有一个线程的线程池 没有超时时间, 工作队列使用无界的LinkedBlockingQueue
     */
    private static ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
    // private static ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor(Executors.defaultThreadFactory());
 
    /**
     * 有固定线程的线程池(即corePoolSize = maximumPoolSize) 没有超时时间,
     * 工作队列使用无界的LinkedBlockingQueue
     */
    private static ExecutorService fixedThreadPool = Executors.newFixedThreadPool(5);
//     private static ExecutorService fixedThreadPool = Executors.newFixedThreadPool(5, Executors.defaultThreadFactory());
 
    /**
     * 大小不限的线程池 核心线程数为0, 最大线程数为Integer.MAX_VALUE, 存活时间为60秒 该线程池可以无限扩展,
     * 并且当需求降低时会自动收缩, 工作队列使用同步移交SynchronousQueue.
     */
    private static ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
    // private static ExecutorService cachedThreadPool = Executors.newCachedThreadPool(Executors.defaultThreadFactory());
 
    /**
     * 给定的延迟之后运行任务, 或者定期执行任务的线程池
     */
    private static ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);
    // private static ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5, Executors.defaultThreadFactory());
 
    public static void main(String args[]) throws Exception {
 
        /**
         * 例子1: 没有返回结果的异步任务
         */
        THREAD_POOL.submit(new Runnable() {
            @Override
            public void run() {
                // do something
                System.out.println("没有返回结果的异步任务");
            }
        });
        
        /**
         * 例子2: 有返回结果的异步任务
         */
        Future<List<String>> future = THREAD_POOL.submit(new Callable<List<String>>() {
            @Override
            public List<String> call() {
                List<String> result = new ArrayList<>();
                result.add("ycw");
                return result;
            }
        });
        List<String> result = future.get(); // 获取返回结果
        System.out.println("有返回结果的异步任务: " + result);
        
        /**
         * 例子3: 
         * 有延迟的, 周期性执行异步任务
         * 本例子为: 延迟1秒, 每2秒执行1次
         */
        scheduledThreadPool.scheduleAtFixedRate(new Runnable() {
            @Override
            public void run() {
                System.out.println("this is " + Thread.currentThread().getName());
            }
 
        }, 1, 2, TimeUnit.SECONDS);
        
        /**
         * 例子4: FutureTask的使用
         */
        Callable<String> task = new Callable<String>() {
            public String call() {
                return "ycw";
            }
        };      
        FutureTask<String> futureTo = new FutureTask<String>(task);
        THREAD_POOL.submit(futureTo);
        System.out.println(futureTo.get()); // 获取返回结果
//        System.out.println(futureTo.get(3, TimeUnit.SECONDS));  // 超时时间为3秒
    }
}
```

### 创建线程池源码分析

我们现在来看看Executors创建线程池的源码：
**newCachedThreadPool方法**：

```
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                    0L, TimeUnit.MILLISECONDS,
                                     new LinkedBlockingQueue<Runnable>());
}
```

它将corePoolSize和maximumPoolSize都设定为了nThreads，这样便实现了线程池的大小的固定，不会动态地扩大，另外，keepAliveTime设定为了0，也就是说线程只要空闲下来，就会被移除线程池。

**newCachedThreadPool方法**：

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

它将corePoolSize设定为0，而将maximumPoolSize设定为了Integer的最大值，线程空闲超过60秒，将会从线程池中移除。由于核心线程数为0，因此每次添加任务，都会先从线程池中找空闲线程，如果没有就会创建一个线程（SynchronousQueue决定的，后面会说）来执行新的任务，并将该线程加入到线程池中，而最大允许的线程数为Integer的最大值，因此这个线程池理论上可以不断扩大。

**newSingleThreadExecutor方法**：

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

它将corePoolSize和maximumPoolSize都设定为1，实现了线程池的大小的固定为1。因此每次添加任务，若线程池中无线程，则会创建一个线程，并将该线程加入到线程池中。keepAliveTime设定为了0，也就是说线程只要空闲下来，就会被移除线程池。

### 工作队列

如果新请求的到达速率超过了线程池的处理速率，那么新到来的请求将累积起来。在线程池中，这些请求会在一个由Executor管理的Runnable队列中等待，而不会像线程那样去竞争CPU资源。常见的工作队列有以下几种，前三种用的最多。

+ ArrayBlockingQueue：列表形式的工作队列，必须要有初始队列大小，有界队列，先进先出。
+ LinkedBlockingQueue：链表形式的工作队列，可以选择设置初始队列大小，有界/无界队列，先进先出。
+ SynchronousQueue：SynchronousQueue不是一个真正的队列，而是一种在线程之间移交的机制。要将一个元素放入SynchronousQueue中, 必须有另一个线程正在等待接受这个元素. 如果没有线程等待，并且线程池的当前大小小于最大值，那么ThreadPoolExecutor将创建 一个线程, 否则根据饱和策略，这个任务将被拒绝。使用直接移交将更高效，因为任务会直接移交 给执行它的线程，而不是被首先放在队列中, 然后由工作者线程从队列中提取任务. 只有当线程池是无解的或者可以拒绝任务时，SynchronousQueue才有实际价值.
+ PriorityBlockingQueue：优先级队列，有界队列，根据优先级来安排任务，任务的优先级是通过自然顺序或Comparator（如果任务实现了Comparator）来定义的。
+ DelayedWorkQueue：延迟的工作队列，无界队列。

### 缓冲队列的排队策略

如果线程池中的线程数量大于等于corePoolSize，但缓冲队列workQueue未满，则将新添加的任务放到workQueue中，按照FIFO的原则依次等待执行（线程池中有线程空闲出来后依次将缓冲队列中的任务交付给空闲的线程执行）。下面说说几种排队的策略：

- 1、直接提交。缓冲队列采用 SynchronousQueue，它将任务直接交给线程处理而不保持它们。如果不存在可用于立即运行任务的线程（即线程池中的线程都在工作），则试图把任务加入缓冲队列将会失败，因此会构造一个新的线程来处理新添加的任务，并将其加入到线程池中。直接提交通常要求无界 maximumPoolSizes（Integer.MAX_VALUE） 以避免拒绝新提交的任务。newCachedThreadPool采用的便是这种策略。
- 2、无界队列。使用无界队列（典型的便是采用预定义容量的 LinkedBlockingQueue，理论上是该缓冲队列可以对无限多的任务排队）将导致在所有 corePoolSize 线程都工作的情况下将新任务加入到缓冲队列中。这样，创建的线程就不会超过 corePoolSize，也因此，maximumPoolSize 的值也就无效了。当每个任务完全独立于其他任务，即任务执行互不影响时，适合于使用无界队列。newFixedThreadPool采用的便是这种策略。
- 3、有界队列。当使用有限的 maximumPoolSizes 时，有界队列（一般缓冲队列使用ArrayBlockingQueue，并制定队列的最大长度）有助于防止资源耗尽，但是可能较难调整和控制，队列大小和最大池大小需要相互折衷，需要设定合理的参数。

### 饱和策略（拒绝策略）

当有界队列被填满后，饱和策略开始发挥作用。ThreadPoolExecutor的饱和策略可以通过调用setRejectedExecutionHandler来修改。（如果某个任务被提交到一个已被关闭的Executor时，也会用到饱和策略）。饱和策略有以下四种，一般使用默认的AbortPolicy。

+ AbortPolicy：中止策略。默认的饱和策略，抛出未检查的RejectedExecutionException。调用者可以捕获这个异常，然后根据需求编写自己的处理代码。
+ DiscardPolicy：抛弃策略。当新提交的任务无法保存到队列中等待执行时，该策略会悄悄抛弃该任务。
+ DiscardOldestPolicy：抛弃最旧的策略。当新提交的任务无法保存到队列中等待执行时，则会抛弃下一个将被执行的任务，然后尝试重新提交新的任务。（如果工作队列是一个优先队列，那么“抛弃最旧的”策略将导致抛弃优先级最高的任务，因此最好不要将“抛弃最旧的”策略和优先级队列放在一起使用）。
+ CallerRunsPolicy：调用者运行策略。该策略实现了一种调节机制，该策略既不会抛弃任务，也不会抛出异常，而是将某些任务回退到调用者（调用线程池执行任务的主线程），从而降低新任务的流程。它不会在线程池的某个线程中执行新提交的任务，而是在一个调用了execute的线程中执行该任务。当线程池的所有线程都被占用，并且工作队列被填满后，下一个任务会在调用execute时在主线程中执行（调用线程池执行任务的主线程）。由于执行任务需要一定时间，因此主线程至少在一段时间内不能提交任务，从而使得工作者线程有时间来处理完正在执行的任务。在这期间，主线程不会调用accept，因此到达的请求将被保存在TCP层的队列中。如果持续过载，那么TCP层将最终发现它的请求队列被填满，因此同样会开始抛弃请求。当服务器过载后，这种过载情况会逐渐向外蔓延开来——从线程池到工作队列到应用程序再到TCP层，最终达到客户端，导致服务器在高负载下实现一种平缓的性能降低。

### 线程工厂

每当线程池需要创建一个线程时，都是通过线程工厂方法来完成的。在ThreadFactory中只定义了一个方法newThread，每当线程池需要创建一个新线程时都会调用这个方法。Executors提供的线程工厂有两种，一般使用默认的，当然如果有特殊需求，也可以自己定制。

+ DefaultThreadFactory：默认线程工厂，创建一个新的、非守护的线程，并且不包含特殊的配置信息。
+ PrivilegedThreadFactory：通过这种方式创建出来的线程，将与创建privilegedThreadFactory的线程拥有相同的访问权限、 AccessControlContext、ContextClassLoader。如果不使用privilegedThreadFactory， 线程池创建的线程将从在需要新线程时调用execute或submit的客户程序中继承访问权限。
+ 自定义线程工厂：可以自己实现ThreadFactory接口来定制自己的线程工厂方法。



## ThreadPoolExecutor源码解析

了解这几个点，有助于你阅读下面的源码解释。

+ 下面的源码解读中提到的运行状态就是runState，有效的线程数就是workerCount，内容比较多，所以可能两种写法都用到。
+ 运行状态的一些定义：
  + RUNNING：接受新任务并处理排队任务； 
  + SHUTDOWN：不接受新任务，但处理排队任务； 
  + STOP：不接受新任务，不处理排队任务，并中断正在进行的任务；
  + TIDYING：所有任务已经终止；workerCount为零，线程转换到状态，TIDYING将运行terminate()钩子方法；
  + TERMINATED：terminated()已经完成，该方法执行完毕代表线程池已经完全终止。
  + 运行状态之间并不是随意转换的，大多数状态都只能由固定的状态转换而来，转换关系见第4点~第8点。

+ RUNNING - > SHUTDOWN：在调用shutdown()时，可能隐含在finalize()。
+ (RUNNING or SHUTDOWN) -> STOP：调用shutdownNow()。
+ SHUTDOWN - > TIDYING：当队列和线程池都是空的时。
+ STOP - > TIDYING：当线程池为空时。
+ TIDYING - > TERMINATED：当terminate()方法完成时。

### 基础属性（很重要）

```java
/**
 * 主池控制状态ctl是包含两个概念字段的原子整数: workerCount：指有效的线程数量；
 * runState：指运行状态，运行，关闭等。为了将workerCount和runState用1个int来表示，
 * 我们限制workerCount范围为(2 ^ 29) - 1，即用int的低29位用来表示workerCount，
 * 用int的高3位用来表示runState，这样workerCount和runState刚好用int可以完整表示。
 */
// 初始化时有效的线程数为0, 此时ctl为: 1010 0000 0000 0000 0000 0000 0000 0000 
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0)); 
// 高3位用来表示运行状态，此值用于运行状态向左移动的位数，即29位
private static final int COUNT_BITS = Integer.SIZE - 3;     
// 线程数容量，低29位表示有效的线程数, 0001 1111 1111 1111 1111 1111 1111 1111
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;
 
/**
 * 大小关系：RUNNING < SHUTDOWN < STOP < TIDYING < TERMINATED，
 * 源码中频繁使用大小关系来作为条件判断。
 * 1010 0000 0000 0000 0000 0000 0000 0000 运行
 * 0000 0000 0000 0000 0000 0000 0000 0000 关闭
 * 0010 0000 0000 0000 0000 0000 0000 0000 停止
 * 0100 0000 0000 0000 0000 0000 0000 0000 整理
 * 0110 0000 0000 0000 0000 0000 0000 0000 终止
 */
private static final int RUNNING    = -1 << COUNT_BITS; // 运行
private static final int SHUTDOWN   =  0 << COUNT_BITS; // 关闭
private static final int STOP       =  1 << COUNT_BITS; // 停止
private static final int TIDYING    =  2 << COUNT_BITS; // 整理
private static final int TERMINATED =  3 << COUNT_BITS; // 终止
 
/**
 * 得到运行状态:入参c为ctl的值，~CAPACITY高3位为1低29位全为0, 
 * 因此运算结果为ctl的高3位, 也就是运行状态
 */
private static int runStateOf(int c)     { return c & ~CAPACITY; }  
/**
 * 得到有效的线程数:入参c为ctl的值, CAPACITY高3为为0, 
 * 低29位全为1, 因此运算结果为ctl的低29位, 也就是有效的线程数
 */
private static int workerCountOf(int c)  { return c & CAPACITY; }   
/**
 * 得到ctl的值：高3位的运行状态和低29位的有效线程数进行或运算, 
 * 组合成一个完成的32位数
 */
private static int ctlOf(int rs, int wc) { return rs | wc; }    
 
// 状态c是否小于s
private static boolean runStateLessThan(int c, int s) { 
    return c < s;
}
// 状态c是否大于等于s
private static boolean runStateAtLeast(int c, int s) {
    return c >= s;
}
// 状态c是否为RUNNING（小于SHUTDOWN的状态只有RUNNING）
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
}
 
// 使用CAS增加一个有效的线程
private boolean compareAndIncrementWorkerCount(int expect) {    
    return ctl.compareAndSet(expect, expect + 1);
}
 
// 使用CAS减少一个有效的线程
private boolean compareAndDecrementWorkerCount(int expect) {    
    return ctl.compareAndSet(expect, expect - 1);
}
 
// 减少一个有效的线程
private void decrementWorkerCount() {
    do {} while (! compareAndDecrementWorkerCount(ctl.get()));
}
 
// 工作队列
private final BlockingQueue<Runnable> workQueue;    
 
// 锁
private final ReentrantLock mainLock = new ReentrantLock(); 
 
// 包含线程池中的所有工作线程,只有在mainLock的情况下才能访问,Worker集合
private final HashSet<Worker> workers = new HashSet<Worker>();
 
private final Condition termination = mainLock.newCondition();
 
// 跟踪线程池的最大到达大小，仅在mainLock下访问
private int largestPoolSize;
 
// 总的完成的任务数
private long completedTaskCount;
 
// 线程工厂，用于创建线程
private volatile ThreadFactory threadFactory;
 
// 拒绝策略
private volatile RejectedExecutionHandler handler;
 
 
/**
 * 线程超时时间，当线程数超过corePoolSize时生效, 
 * 如果有线程空闲时间超过keepAliveTime, 则会被终止
 */
private volatile long keepAliveTime;    
 
// 是否允许核心线程超时，默认false，false情况下核心线程会一直存活。
private volatile boolean allowCoreThreadTimeOut;
 
// 核心线程数
private volatile int corePoolSize;
 
// 最大线程数
private volatile int maximumPoolSize;
 
// 默认饱和策略（拒绝策略）, 抛异常
private static final RejectedExecutionHandler defaultHandler = 
    new AbortPolicy();
 
private static final RuntimePermission shutdownPerm =
    new RuntimePermission("modifyThread");
 
/**
 * Worker类，每个Worker包含一个线程、一个初始任务、一个任务计算器
 */
private final class Worker   
    extends AbstractQueuedSynchronizer
    implements Runnable
{
    private static final long serialVersionUID = 6138294804551838833L;
 
    final Thread thread;    // Worker对应的线程
    Runnable firstTask; // 运行的初始任务。
    volatile long completedTasks;   // 每个线程的任务计数器
 
    Worker(Runnable firstTask) {
        setState(-1); // 禁止中断，直到runWorker
        this.firstTask = firstTask; // 设置为初始任务
        // 使用当前线程池的线程工厂创建一个线程
        this.thread = getThreadFactory().newThread(this);  
    }
 
    // 将主运行循环委托给外部runWorker
    public void run() {
        runWorker(this);
    }
 
    // Lock methods
    //
    // The value 0 represents the unlocked state.
    // The value 1 represents the locked state.
    /**
     * 通过AQS的同步状态来实现锁机制。state为0时代表锁未被获取（解锁状态），
     * state为1时代表锁已经被获取（加锁状态）。
     */
    protected boolean isHeldExclusively() { // 
        return getState() != 0; 
    }
    protected boolean tryAcquire(int unused) {  // 尝试获取锁
        if (compareAndSetState(0, 1)) { // 使用CAS尝试将state设置为1，即尝试获取锁
            // 成功将state设置为1，则当前线程拥有独占访问权
            setExclusiveOwnerThread(Thread.currentThread());    
            return true;
        }
        return false;
    }
    protected boolean tryRelease(int unused) {  // 尝试释放锁
        setExclusiveOwnerThread(null);  // 释放独占访问权：即将独占访问线程设为null
        setState(0);    // 解锁：将state设置为0
        return true;
    }
    public void lock()        { acquire(1); }   // 加锁
    public boolean tryLock()  { return tryAcquire(1); } // 尝试加锁
    public void unlock()      { release(1); }   // 解锁
    public boolean isLocked() { return isHeldExclusively(); }  // 是否为加锁状态 
    void interruptIfStarted() { // 如果线程启动了，则进行中断
        Thread t;
        if (getState() >= 0 && (t = thread) != null && !t.isInterrupted()) {
            try {
                t.interrupt();
            } catch (SecurityException ignore) {
            }
        }
    }
}
```



### execute(Runnable command)

使用线程池的`submit()`方法提交任务时，会走到该方法，该方法也是线程池最重要的方法。

```java
public void execute(Runnable command) {
    if (command == null)    // 为空校验
        throw new NullPointerException();
 
    int c = ctl.get();  // 拿到当前的ctl值
    if (workerCountOf(c) < corePoolSize) {  // 如果有效的线程数小于核心线程数
        if (addWorker(command, true))   // 则新建一个线程来处理任务（核心线程）
            return;
        c = ctl.get();  // 拿到当前的ctl值
    }
    // 走到这里说明有效的线程数已经 >= 核心线程数
    if (isRunning(c) && workQueue.offer(command)) {// 如果当前状态是运行, 尝试将任务放入工作队列
        int recheck = ctl.get();    // 再次拿到当前的ctl值
        // 如果再次检查状态不是运行, 则将刚才添加到工作队列的任务移除
        if (! isRunning(recheck) && remove(command)) 
            reject(command);    // 并调用拒绝策略
        else if (workerCountOf(recheck) == 0) // 如果再次检查时,有效的线程数为0, 
            addWorker(null, false); // 则新建一个线程(非核心线程)
    }
    // 走到这里说明工作队列已满
    else if (!addWorker(command, false))//尝试新建一个线程来处理任务(非核心)
        reject(command);    // 如果失败则调用拒绝策略
}
```

### addWorker(Runnable firstTask, boolean core)

新建一个线程来处理任务，指定Runnable类型的任务，和是否创建核心线程。

```java
/**
 * 添加一个Worker，Worker包含一个线程和一个任务，由这个线程来执行该任务。
 */
private boolean addWorker(Runnable firstTask, boolean core) {   
    retry:
    for (;;) {		// 自旋
        int c = ctl.get();  // c赋值为ctl
        int rs = runStateOf(c); // rs赋值为运行状态
        /**
         * 1.如果池停止或有资格关闭，则此方法返回false；
         * 如果线程工厂在被询问时未能创建线程，它也返回false。 
         * 包括以下5种情况：
         * 1).rs为RUNNING，通过校验。
         * 2).rs为STOP或TIDYING或TERMINATED，返回false。
         * （STOP、TIDYING、TERMINATED：已经停止进入最后清理终止，不接受任务不处理队列任务）
         * 3).rs为SHUTDOWN，提交的任务不为空，返回false。
         * （SHUTDOWN：不接受任务但是处理队列任务，因此任务不为空返回false）
         * 4).rs为SHUTDOWN，提交的任务为空，并且工作队列为空，返回false。
         * （状态为SHUTDOWN、提交的任务为空、工作队列为空，则线程池有资格关闭，直接返回false）
         * 5).rs为SHUTDOWN，提交的任务为空，并且工作队列不为空，通过校验。
         * （因为SHUTDOWN状态下刚好可以处理队列任务）
         */
        if (rs >= SHUTDOWN &&
            ! (rs == SHUTDOWN &&
               firstTask == null &&
               ! workQueue.isEmpty()))
            return false;
 
        for (;;) {
            int wc = workerCountOf(c);  // 拿到有效的线程数
            // 校验有效的线程数是否超过阈值
            if (wc >= CAPACITY ||
                wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // 使用CAS将workerCount+1, 修改成功则跳出循环，否则进入下面的状态判断
            if (compareAndIncrementWorkerCount(c))
                break retry;
            c = ctl.get();  // 重新读取ctl
            // 判断当前运行状态，如果不等于上面获取的运行状态rs，
            // 说明rs被其他线程修改了，跳到retry重新校验线程池状态
            if (runStateOf(c) != rs)
                continue retry;
            // 走到这里说明compareAndIncrementWorkerCount失败; 
            // 重试内部循环（状态没变，则继续内部循环，尝试使用CAS修改workerCount）
        }
    }
 
    boolean workerStarted = false;  // Worker的线程是否启动
    boolean workerAdded = false;    // Worker是否成功增加
    Worker w = null;
    try {
        w = new Worker(firstTask);  // 用firstTask和当前线程创建一个Worker
        final Thread t = w.thread;  // 拿到Worker对应的线程
        if (t != null) {
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();    // 加锁
            try {
                // Recheck while holding lock.
                // Back out on ThreadFactory failure or if
                // shut down before lock acquired.
                int rs = runStateOf(ctl.get()); // 加锁的情况下重新获取当前的运行状态
 
                // 如果当前的运行状态为RUNNING，
                // 或者当前的运行状态为SHUTDOWN并且firstTask为空，则通过校验
                if (rs < SHUTDOWN ||
                    (rs == SHUTDOWN && firstTask == null)) {
                    if (t.isAlive())    // 预先校验线程是可以启动的
                        throw new IllegalThreadStateException();
                    workers.add(w); // 将刚创建的worker添加到工作者列表
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                mainLock.unlock();
            }
            if (workerAdded) {  // 如果Worker添加成功，则启动线程执行
                t.start();
                workerStarted = true;
            }
        }
    } finally {
        if (! workerStarted)    // 如果Worker的线程没有成功启动
            addWorkerFailed(w); // 则进行回滚, 移除之前添加的Worker
    }
    return workerStarted;
}
```

该方法主要目的就是使用入参中的firstTask和当前线程添加一个Worker，前面的for循环主要是对当前线程池的运行状态和有效的线程数进行一些校验。该方法涉及到的其他方法有`addWorkerFailed()`（见下文`addWorkerFailed()`源码解读）；还有就是Worker的线程启动时，会调用Worker里的run方法，执行runWorker(this)方法（见下文runWorker源码解读）。

### addWorkerFailed(Worker w)

```java

private void addWorkerFailed(Worker w) {    // 回滚Worker的添加，就是将Worker移除
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        if (w != null)
            workers.remove(w);  // 移除Worker
        decrementWorkerCount(); // 有效线程数-1
        tryTerminate(); // 有worker线程移除，可能是最后一个线程退出需要尝试终止线程池
    } finally {
        mainLock.unlock();
    }
}
```

该方法很简单，就是移除入参中的Worker并将workerCount-1，最后调用tryTerminate尝试终止线程池，tryTerminate见下文对应方法源码解读。

### runWorker(Worker w)

```java
/**
 * Worker的线程开始执行任务
 */
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread(); // 获取当前线程
    Runnable task = w.firstTask;    // 拿到Worker的初始任务
    w.firstTask = null;
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;   // Worker是不是因异常而死亡
    try {
        while (task != null || (task = getTask()) != null) {// Worker取任务执行
            w.lock();   // 加锁
            /**如果线程池停止，确保线程中断; 如果不是，确保线程不被中断。
             * 在第二种情况下进行重新检查，以便在清除中断的同时处理shutdownNow竞争
             * 线程池停止指运行状态为STOP/TIDYING/TERMINATED中的一种
             */
            if ((runStateAtLeast(ctl.get(), STOP) ||    // 判断线程池运行状态
                 (Thread.interrupted() &&   // 重新检查
                  runStateAtLeast(ctl.get(), STOP))) && // 再次判断线程池运行状态
                !wt.isInterrupted())// 走到这里代表线程池运行状态为停止,检查wt是否中断
                wt.interrupt(); // 线程池的状态为停止并且wt不为中断, 则将wt中断
            try {
                beforeExecute(wt, task);// 执行beforeExecute（默认空，需要自己重写）
                Throwable thrown = null;
                try {
                    task.run(); // 执行任务
                } catch (RuntimeException x) {
                    thrown = x; throw x; //如果抛异常,则completedAbruptly为true
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);// 执行afterExecute（需要自己重写）
                }
            } finally {
                task = null;    // 将执行完的任务清空
                w.completedTasks++; // Worker完成任务数+1
                w.unlock();
            }
        }
        completedAbruptly = false;  // 如果执行到这里，则worker是正常退出
    } finally {
        processWorkerExit(w, completedAbruptly);// 调用processWorkerExit方法
    }
}
```

该方法为Worker线程开始执行任务，首先执行当初创建Worker时的初始任务，接着从工作队列中获取任务执行。主要涉及两个方法：获取任务的方法getTask（见下文getTask源码解读）和执行Worker退出的方法processWorkerExit（见下文processWorkerExit源码解读）。注：processWorkerExit在处理正常Worker退出时，没有对workerCount-1，而是在getTask方法中进行workerCount-1。

### getTask()

```java
private Runnable getTask() {    // Worker从工作队列获取任务
    boolean timedOut = false; // poll方法取任务是否超时
 
    for (;;) {  // 无线循环
        int c = ctl.get();  // ctl
        int rs = runStateOf(c); // 当前运行状态
 
        // 如果线程池运行状态为停止，或者可以停止（状态为SHUTDOWN并且队列为空）
        // 则返回null，代表当前Worker需要移除
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {    
            decrementWorkerCount(); // 将workerCount - 1
            // 返回null前将workerCount - 1,
            // 因此processWorkerExit中completedAbruptly＝false时无需再减
            return null;
        }
 
        int wc = workerCountOf(c);  // 当前的workerCount
 
        // 判断当前Worker是否可以被移除, 即当前Worker是否可以一直等待任务。
        // 如果allowCoreThreadTimeOut为true，或者workerCount大于核心线程数，
        // 则当前线程是有超时时间的（keepAliveTime），无法一直等待任务。
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;    
 
        // 如果wc超过最大线程数 或者 当前线程会超时并且已经超时，
        // 并且wc > 1 或者 工作队列为空，则返回null，代表当前Worker需要移除
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {   // 确保有Worker可以移除 
            if (compareAndDecrementWorkerCount(c))
                // 返回null前将workerCount - 1，
                // 因此processWorkerExit中completedAbruptly＝false时无需再减
                return null;    
            continue;
        }
 
        try {
            // 根据线程是否会超时调用相应的方法，poll为带超时的获取任务方法
            // take()为不带超时的获取任务方法，会一直阻塞直到获取到任务
            Runnable r = timed ? 
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
            if (r != null)
                return r;
            timedOut = true;    // 走到这代表当前线程获取任务超时
        } catch (InterruptedException retry) {
            timedOut = false;   // 被中断
        }
    }
}
```

Worker从工作队列获取任务，如果allowCoreThreadTimeOut为false并且  workerCount<=corePoolSize，则这些核心线程永远存活，并且一直在尝试获取工作队列的任务；否则，线程会有超时时间（keepAliveTime），当在keepAliveTime时间内获取不到任务，该线程的Worker会被移除。 
Worker移除的过程：getTask方法返回null，导致runWorker方法中跳出while循环，调用processWorkerExit方法将Worker移除。注意：在返回null的之前，已经将workerCount-1，因此在processWorkerExit中，completedAbruptly=false的情况（即正常超时退出）不需要再将workerCount-1。

### processWorkerExit(Worker , boolean)

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {   // Worker的退出
    // 如果Worker是异常死亡（completedAbruptly=true），则workerCount-1；
    // 如果completedAbruptly为false的时候（正常超时退出），则代表task=getTask()等于null，
    // getTask()方法中返回null的地方，都已经将workerCount - 1，所以此处无需再-1
    if (completedAbruptly) 
        decrementWorkerCount();
 
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();    // 加锁
    try {
        completedTaskCount += w.completedTasks; // 该Worker完成的任务数加到总完成的任务数
        workers.remove(w);  // 移除该Worker
    } finally {
        mainLock.unlock();
    }
 
    tryTerminate(); // 有Worker线程移除，可能是最后一个线程退出，需要尝试终止线程池
 
    int c = ctl.get();  // 获取当前的ctl
    if (runStateLessThan(c, STOP)) {  // 如果线程池的运行状态还没停止（RUNNING或SHUTDOWN）
        if (!completedAbruptly) {   // 如果Worker不是异常死亡
            // min为线程池的理论最小线程数:如果允许核心线程超时则min为0,否则min为核心线程数
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;    
            // 如果min为0,工作队列不为空,将min设置为1,确保至少有1个Worker来处理队列里的任务 
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;
            // 当前有效的线程数>=min，直接返回;
            if (workerCountOf(c) >= min)
                return; // replacement not needed 
            // 如果代码走到这边，代表workerCountOf(c) < min，此时会走到下面的addWorker方法。
            // 通过getTask方法我们知道，当allowCoreThreadTimeOut为false
            // 并且workerCount<=corePoolSize时，是不会走到processWorkerExit方法的。
            // 因此走到这边只可能是当前移除的Worker是最后一个Worker，但是此时工作
            // 队列还不为空，因此min被设置成了1，所以需要在添加一个Worker来处理工作队列。
        }
        addWorker(null, false); // 添加一个Worker
    }
}
```

该方法就是执行Worker的退出：统计完成的任务数，将Worker移除，并尝试终止线程池，最后根据情况决定是否创建一个新的Worker。两种情况下会创建一个新的Worker：1）被移除的Worker是由于异常而死亡；2）被移除的Worker是最后一个Worker，但是工作队列还有任务。completedAbruptly=false时，没有将workerCount-1是因为已经在getTask方法中将workerCount-1。

### tryTerminate()

```java
final void tryTerminate() { // 尝试终止线程池
    for (;;) {
        int c = ctl.get();
        // 只有当前状态为STOP 或者 SHUTDOWN并且队列为空，才会尝试整理并终止
        // 1: 当前状态为RUNNING，则不尝试终止，直接返回
        // 2: 当前状态为TIDYING或TERMINATED，代表有其他线程正在执行终止，直接返回
        // 3: 当前状态为SHUTDOWN 并且 workQueue不为空，则不尝试终止，直接返回
        if (isRunning(c) || // 1
            runStateAtLeast(c, TIDYING) ||  // 2
            (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))   // 3
            return;
        // 走到这代表线程池可以终止（通过上面的校验）
        // 如果此时有效线程数不为0， 将中断一个空闲的Worker，以确保关闭信号传播
        if (workerCountOf(c) != 0) { // Eligible to terminate 
            interruptIdleWorkers(ONLY_ONE);
            return;
        }
 
        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();    // 加锁，终止线程池
        try {
            // 使用CAS将ctl的运行状态设置为TIDYING，有效线程数设置为0
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {  
                try {
                    terminated();   // 供用户重写的terminated方法，默认为空
                } finally {
                    // 将ctl的运行状态设置为TERMINATED，有效线程数设置为0
                    ctl.set(ctlOf(TERMINATED, 0));  
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
        // else retry on failed CAS
    }
}
```

该方法用来尝试终止线程池，主要在移除Worker后会调用此方法。首先进行一些状态的校验，如果通过校验，则在加锁的条件下，使用CAS将运行状态设为TERMINATED，有效线程数设为0。



出处：http://blog.csdn.net/ns_code/article/details/17465497

## Callable、Future和FutureTask

创建线程的2种方式，一种是直接继承Thread，另外一种就是实现Runnable接口。
这2种方式都有一个缺陷就是：在执行完任务之后无法获取执行结果。
如果需要获取执行结果，就必须通过共享变量或者使用线程通信的方式来达到效果，这样使用起来就比较麻烦。
而自从Java 1.5开始，就提供了 **Callable** 和 **Future** ，通过它们可以在任务执行完毕之后得到任务执行结果。

**目录**

- [Runnable](chrome-extension://kidnkfckhbdkfgbicccmdggmpgogehop/index_zh.html#runnable)
- [Callable](chrome-extension://kidnkfckhbdkfgbicccmdggmpgogehop/index_zh.html#callable)
- [Future](chrome-extension://kidnkfckhbdkfgbicccmdggmpgogehop/index_zh.html#Future)
- [FutureTask](chrome-extension://kidnkfckhbdkfgbicccmdggmpgogehop/index_zh.html#FutureTask)
- [使用实例](chrome-extension://kidnkfckhbdkfgbicccmdggmpgogehop/index_zh.html#使用实例)

### Runnable

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

这是一个函数式接口，由于run()方法返回值为void类型，所以在执行完任务之后无法返回任何结果。

------

### Callable

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

Callable位于java.util.concurrent包下，它也是一个函数式接口，在它里面也的这个方法叫做call()。可以看到，这是一个泛型接口，call()函数返回的类型就是传递进来的V类型。
那么怎么**使用Callable**呢？一般情况下是配合ExecutorService来使用的，在ExecutorService接口中声明了若干个submit方法的重载版本：

```java
<T> Future<T> submit(Callable<T> task);
<T> Future<T> submit(Runnable task, T result);
Future<?> submit(Runnable task);
```

### Future

```java
public interface Future<V> {
    boolean cancel(boolean mayInterruptIfRunning);
    boolean isCancelled();
    boolean isDone();
    V get() throws InterruptedException, ExecutionException;
    V get(long timeout, TimeUnit unit)
      throws InterruptedException, ExecutionException, TimeoutException;
}
```

Future类位于java.util.concurrent包下，它是一个接口。
在Future接口中声明了5个方法，下面依次解释每个方法的作用：

- **cancel方法用来取消任务**，如果取消任务成功则返回true，如果取消任务失败则返回false。参数mayInterruptIfRunning表示是否允许取消正在执行却没有执行完毕的任务，如果设置true，则表示可以取消正在执行过程中的任务。如果任务已经完成，则无论mayInterruptIfRunning为true还是false，此方法肯定返回false，即如果取消已经完成的任务会返回false；如果任务正在执行，若mayInterruptIfRunning设置为true，则返回true，若mayInterruptIfRunning设置为false，则返回false；如果任务还没有执行，则无论mayInterruptIfRunning为true还是false，肯定返回true。
  isCancelled方法表示任务是否被取消成功，如果在任务正常完成前被取消成功，则返回 true。
- **isDone方法**表示任务是否已经完成，若任务完成，则返回true；
- **get()方法**用来获取执行结果，这个方法会产生阻塞，会一直等到任务执行完毕才返回；
- **get(long timeout, TimeUnit unit)**用来获取执行结果，如果在指定时间内，还没获取到结果，就直接返回null。
  也就是说Future提供了**三种功能：**
- 1）判断任务是否完成；
- 2）能够中断任务；
- 3）能够获取任务执行结果。

因为Future只是一个接口，所以是无法直接用来创建对象使用的，因此就有了下面的FutureTask。

### FutureTask

```java
public class FutureTask<V> implements RunnableFuture<V> {
    private volatile int state;
    private static final int NEW          = 0;
    private static final int COMPLETING   = 1;
    private static final int NORMAL       = 2;
    private static final int EXCEPTIONAL  = 3;
    private static final int CANCELLED    = 4;
    private static final int INTERRUPTING = 5;
    private static final int INTERRUPTED  = 6;
    private Callable<V> callable;
    private Object outcome; 
    private volatile Thread runner;
    private volatile WaitNode waiters;
```

我们先来看一下FutureTask的实现：

```java
public class FutureTask<V> implements RunnableFuture<V>
```

FutureTask类实现了RunnableFuture接口，我们看一下RunnableFuture接口的实现：

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```

从这里我们可以看出RunnableFuture继承了 **Runnable接口** 和 **Future接口**，而FutureTask实现了RunnableFuture接口。所以它既可以作为Runnable被线程执行，又可以作为Future得到Callable的返回值。

FutureTask提供了2个构造器：

```java
public FutureTask(Callable<V> callable) {
}
public FutureTask(Runnable runnable, V result) {
}
```

### 使用实例

上文说到，可以通过Executor框架来控制线程的启动，执行，关闭，简化并发编程。Executor框架把任务提交和执行解耦，要执行任务的人只需要把任务描述清楚提交即可，任务的执行提交人不需要去关心。下面我们两个实例的区别其实就是向 `ExecutorService` 提交任务的区别，表现在源码上就是下面两个方法的区别：

```java
<T> Future<T> submit(Callable<T> task);
Future<?> submit(Runnable task);
```

1、使用 **Callable+Future** 获取执行结果

```java
public class Test {
    public static void main(String[] args) {
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        Future<Integer> result = executor.submit(task);
        executor.shutdown();

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }

        System.out.println("主线程在执行任务");

        try {
            System.out.println("task运行结果"+result.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        System.out.println("所有任务执行完毕");
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++)
            sum += i;
        return sum;
    }
}
```

2.使用 **Callable+FutureTask** 获取执行结果

```java
public class Test {
    public static void main(String[] args) {
        //第一种方式
        ExecutorService executor = Executors.newCachedThreadPool();
        Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        executor.submit(futureTask);
        executor.shutdown();

        //第二种方式，注意这种方式和第一种方式效果是类似的，只不过一个使用的是ExecutorService，一个使用的是Thread
        /*Task task = new Task();
        FutureTask<Integer> futureTask = new FutureTask<Integer>(task);
        Thread thread = new Thread(futureTask);
        thread.start();*/

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e1) {
            e1.printStackTrace();
        }

        System.out.println("主线程在执行任务");

        try {
            System.out.println("task运行结果"+futureTask.get());
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (ExecutionException e) {
            e.printStackTrace();
        }

        System.out.println("所有任务执行完毕");
    }
}
class Task implements Callable<Integer>{
    @Override
    public Integer call() throws Exception {
        System.out.println("子线程在进行计算");
        Thread.sleep(3000);
        int sum = 0;
        for(int i=0;i<100;i++)
            sum += i;
        return sum;
    }
}
```

出处：http://www.cnblogs.com/dolphin0520/