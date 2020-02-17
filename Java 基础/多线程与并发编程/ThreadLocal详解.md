# ThreadLocal详解

## 概述

ThreadLocal 用于提供线程局部变量，在多线程环境可以保证各个线程里的变量独立于其它线程里的变量。也就是说 ThreadLocal 可以为每个线程创建一个**单独的变量副本**，相当于线程的 private static 类型变量。

ThreadLocal 的作用和同步机制有些相反：同步机制是为了保证多线程环境下数据的一致性；而 ThreadLocal 是保证了多线程环境下数据的独立性。

我们来举一个例子，比如：多个用户访问服务端的一段服务代码，在并发访问的时候，每个访问线程都需要维护一些独立的变量，为了保证自己数据的独立性（也就是不被其他用户线程访问修改），就可以使用ThreadLocal。

我们看一个简单的示例代码：

```java
public class ThreadLocalTest {
    private static String strLabel;
    private static ThreadLocal<String> threadLabel = new ThreadLocal<>();

    public static void main(String... args) {
        strLabel = "main";
        threadLabel.set("main");

        Thread thread = new Thread() {

            @Override
            public void run() {
                super.run();
                strLabel = "child";
                threadLabel.set("child");
            }

        };

        thread.start();
        try {
            // 保证线程执行完毕
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

        System.out.println("strLabel = " + strLabel);
        System.out.println("threadLabel = " + threadLabel.get());
    }
}
```

输出结果：

```java
strLabel = child
threadLabel = main
```

主线程和tread线程各持有两个String类型的同名变量`strLabel`和`threadLabel`，区别是`threadLabel`封装在`ThreadLocal`中，而前者只是一个普通的String类型变量。因此，主线程中的`strLabel`变量被thread线程访问并且更改了，而`threadLabel`变量会为主线程和thread各创建一个【单独的变量副本】用于保存自己的变量副本，他们是相互独立的，只允许自己访问，其他线程访问不到。也就是说 ThreadLocal 类型的变量的值在每个线程中是独立的。

## 源码详解

### set(T value)

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}


ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}

#java.lang.Thread.threadLocals
ThreadLocal.ThreadLocalMap threadLocals = null;


void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}

 ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
     table = new Entry[INITIAL_CAPACITY];
     int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
     table[i] = new Entry(firstKey, firstValue);
     size = 1;
     setThreshold(INITIAL_CAPACITY);
 }
```

set(T value) 方法中，首先获取当前线程，然后在获取到当前线程的 ThreadLocalMap，如果 ThreadLocalMap 不为 null，则将 value 保存到 ThreadLocalMap 中，并用当前 ThreadLocal 作为 key；否则创建一个 ThreadLocalMap 并给到当前线程，然后保存 value。

**关于ThreadLocalMap** 

在 set，get，initialValue 和 remove 方法中都会获取到当前线程，然后通过当前线程获取到 ThreadLocalMap，如果 ThreadLocalMap 为 null，则会创建一个 ThreadLocalMap，并给到当前线程。

ThreadLocalMap 相当于一个 HashMap，是ThreadLocal 实例为每个变量“分配”的“容器”，由每个线程所有，是真正保存值的地方，ThreadLocalMap 中维护的值也是属于线程自己的。这就保证了 ThreadLocal 类型的变量在每个线程中是独立的，在多线程环境下不会相互影响。。关于`ThreadLocalMap`的具体实现和操作，可以先做一个这样的初步认识，我们下面会具体讲解。

### get()

```java
public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

同样的，在 get() 方法中也会获取到当前线程的 ThreadLocalMap，如果 ThreadLocalMap 不为 null，则把获取 key 为当前 ThreadLocal 的值；否则调用 setInitialValue() 方法返回初始值，并保存到新创建的 ThreadLocalMap 中。

### setInitialValue()

```java
private T setInitialValue() {
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}

protected T initialValue() {
    return null;
}
```

initialValue() 是 ThreadLocal 的初始值，默认返回 null，子类可以重写改方法，用于设置 ThreadLocal 的初始值。

### remove()

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}


/**
  * Remove the entry for key.
  * 这部分会在稍后讲解ThreadLocalMap部分讲解
  */
private void remove(ThreadLocal<?> key) {
    Entry[] tab = table;
    int len = tab.length;
    int i = key.threadLocalHashCode & (len-1);
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        if (e.get() == key) {
            e.clear();
            expungeStaleEntry(i);
            return;
        }
    }
}
```

ThreadLocal 还有一个 remove() 方法，用来移除当前 ThreadLocal 对应的值。同样也是同过当前线程的 ThreadLocalMap 来移除相应的值。





## ThreadLocalMap

### 存储结构

```java
// 初始容量，必须是 2 的幂
private static final int INITIAL_CAPACITY = 16;

// 存储数据的哈希表
private Entry[] table;

// table 中已存储的条目数
private int size = 0;

// 表示一个阈值，当 table 中存储的对象达到该值时就会扩容
private int threshold;

// 设置 threshold 的值
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

### 构造器

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    table = new Entry[INITIAL_CAPACITY];
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    table[i] = new Entry(firstKey, firstValue);
    size = 1;
    setThreshold(INITIAL_CAPACITY);
}

static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

ThreadLocal保存变量会先将变量封装到一个`Entry`节点中，`Entry`非常简单，就是一个键值对。

ThreadLocalMap构造方法中会新建一个`Entry`数组，并将将第一次需要保存的`Entry`键值存储到一个数组中，完成一些初始化工作。

### set(ThreadLocal key, Object value)

```java
private void set(ThreadLocal key, Object value) {

    Entry[] tab = table;
    int len = tab.length;
    // 计算要存储的索引位置
    int i = key.threadLocalHashCode & (len-1);

    // 循环判断要存放的索引位置是否已经存在 Entry，若存在，进入循环体
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        ThreadLocal k = e.get();

        // 若索引位置的 Entry 的 key 和要保存的 key 相等，则更新该 Entry 的值
        if (k == key) {
            e.value = value;
            return;
        }

        // 若索引位置的 Entry 的 key 为 null（key 已经被回收了），
        // 表示该位置的 Entry 已经无效，用要保存的键值替换该位置上的 Entry
        if (k == null) {
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // 要存放的索引位置没有 Entry，将当前键值作为一个 Entry 保存在该位置
    tab[i] = new Entry(key, value);
    // 增加 table 存储的条目数
    int sz = ++size;
    // 清除一些无效的条目并判断 table 中的条目数是否已经超出阈值，若超出则进行扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash(); // 调整 table 的容量，并重新摆放 table 中的 Entry
}
```

首先使用 key（当前 ThreadLocal）的 threadLocalHashCode 来计算要存储的索引位置 i。threadLocalHashCode 的值由 ThreadLocal 类管理，每创建一个 ThreadLocal 对象都会自动生成一个相应的 threadLocalHashCode 值，其实现如下：

```java
#java.lang.ThreadLocal.ThreadLocalMap
    
// ThreadLocal 对象的 HashCode
private final int threadLocalHashCode = nextHashCode();

// 使用 AtomicInteger 保证多线程环境下的同步
private static AtomicInteger nextHashCode =
    new AtomicInteger();

// 每次创建 ThreadLocal 对象是 HashCode 的增量
private static final int HASH_INCREMENT = 0x61c88647;

// 计算 ThreadLocal 对象的 HashCode
private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

在保存数据时，如果索引位置有 Entry，且该 Entry 的 key 为 null，表示该位置的 Entry 已经无效，用要保存的键值替换该位置上的 Entry。若索引位置的 Entry 的 key 和要保存的 key 相等，则更新该 Entry 的值。

如果索引位置没有 Entry，将当前键值作为一个 Entry 保存在该位置。利用`cleanSomeSlots(int i, int n)`方法清除一些无效的条目并判断 table 中的条目数是否已经超出阈值，若超出则进行扩容。因为 Entry 的 key 使用的是弱引用的方式，key 如果被回收（即 key 为 null），这时就无法再访问到 key 对应的 value，需要把这样的无效 Entry 清除掉来腾出空间。

### getEntry(ThreadLocal<?> key)

```java
private Entry getEntry(ThreadLocal<?> key) {
    // 使用指定的 key 的 HashCode 计算索引位置
    int i = key.threadLocalHashCode & (table.length - 1);
    // 获取当前位置的 Entry
    Entry e = table[i];
    // 如果 Entry 不为 null 且 Entry 的 key 和 指定的 key 相等，则返回该 Entry
    // 否则调用 getEntryAfterMiss(ThreadLocal key, int i, Entry e) 方法
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    // 索引位置上的 Entry 不为 null 进入循环，为 null 则返回 null
    while (e != null) {
        ThreadLocal k = e.get();
        // 如果 Entry 的 key 和指定的 key 相等，则返回该 Entry
        if (k == key)
            return e;
        // 如果 Entry 的 key 为 null （key 已经被回收了），清除无效的 Entry
        // 否则获取下一个位置的 Entry，循环判断
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

因为可能存在哈希冲突，key 对应的 Entry 的存储位置可能不在通过 key 计算出的索引位置上，也就是说索引位置上的 Entry 不一定是 key 对应的 Entry。所以需要调用 getEntryAfterMiss(ThreadLocal key, int i, Entry e) 方法获取。



### remove(ThreadLocal key)

移除指定的 Entry

```java
private void remove(ThreadLocal key) {
    Entry[] tab = table;
    int len = tab.length;
    // 使用指定的 key 的 HashCode 计算索引位置
    int i = key.threadLocalHashCode & (len-1);
    // 循环判断索引位置的 Entry 是否为 null
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
        // 若 Entry 的 key 和指定的 key 相等，执行删除操作
        if (e.get() == key) {
            // 清除 Entry 的 key 的引用
            e.clear();
            // 清除无效的 Entry
            expungeStaleEntry(i);
            return;
        }
    }
}
```

## **内存泄漏**

> **内存泄漏**指由于疏忽或错误造成程序未能释放已经不再使用的[内存](https://zh.wikipedia.org/wiki/内存)。内存泄漏并非指内存在物理上的消失，而是应用程序分配某段内存后，由于设计错误，导致在释放该段内存之前就失去了对该段内存的控制，从而造成了内存的浪费。

> **弱引用**与[强引用](https://zh.wikipedia.org/w/index.php?title=强引用&action=edit&redlink=1)相对，是指不能确保其引用的[对象](https://zh.wikipedia.org/wiki/对象_(计算机科学))不会被[垃圾回收器](https://zh.wikipedia.org/wiki/垃圾回收器)回收的引用。一个对象若只被弱引用所引用，则被认为是[不可访问](https://zh.wikipedia.org/wiki/不可访问内存)（或弱可访问）的，并因此可能在任何时刻被回收。
>
> Java是第一个将[强引用](https://zh.wikipedia.org/w/index.php?title=强引用&action=edit&redlink=1)作为默认对象引用的主流语言。如果创建了一个弱引用，然后在代码的其它地方用 `get()`获得真实对象，由于弱引用无法阻止垃圾回收，`get()`随时有可能开始返回`null`（假如对象没有被强引用）。

在 ThreadLocalMap 的 set()，get() 和 remove() 方法中，都有清除无效 Entry 的操作，这样做是为了降低内存泄漏发生的可能。

Entry 中的 key 使用了弱引用的方式，这样做是为了降低内存泄漏发生的概率，但不能完全避免内存泄漏。

好的，下面可以分析以下。

假设 Entry 的 key 没有使用弱引用的方式，而是使用了强引用：由于 ThreadLocalMap 的生命周期和当前线程一样长，那么当引用 ThreadLocal 的对象被回收后，由于 ThreadLocalMap 还持有 ThreadLocal 和对应 value 的强引用，ThreadLocal 和对应的 value 是不会被回收的，这就导致了内存泄漏。所以 Entry 以弱引用的方式避免了 ThreadLocal 没有被回收而导致的内存泄漏，但是此时 value 仍然是无法回收的，依然会导致内存泄漏。

ThreadLocalMap 已经考虑到这种情况，并且有一些防护措施：在调用 ThreadLocal 的 get()，set() 和 remove() 的时候都会清除当前线程 ThreadLocalMap 中所有 key 为 null 的 value。这样可以降低内存泄漏发生的概率。所以我们在使用 ThreadLocal 的时候，每次用完 ThreadLocal 都调用 remove() 方法，清除数据，防止内存泄漏。

