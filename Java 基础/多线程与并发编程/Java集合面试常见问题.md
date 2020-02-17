# Java集合面试常见问题

####   常见问题 

-    hashmap如何解决hash冲突，为什么hashmap中的链表需要转成红黑树？    
-    hashmap什么时候会触发扩容？    
-    jdk1.8之前并发操作hashmap时为什么会有死循环的问题？    
-    hashmap扩容时每个entry需要再计算一次hash吗？    
-    hashmap的数组长度为什么要保证是2的幂？    
-    如何用LinkedHashMap实现LRU？    
-    如何用TreeMap实现一致性hash？

#### hashmap如何解决hash冲突，为什么hashmap中的链表需要转成红黑树？    

> 当 HashMap 中有大量的元素都存放到同一个桶中时，这个桶下有一条长长的链表，这个时候 HashMap 就相当于一个单链表，假如单链表有 n 个元素，遍历的时间复杂度就是 O(n)，如果 hash 冲突严重，由这里产生的性能问题尤为突显。
> JDK 1.8 中引入了红黑树，当链表长度 >= TREEIFY_THRESHOLD（8） & tab.length >= MIN_TREEIFY_CAPACITY（64）时，链表就会转化为红黑树，它的查找时间复杂度为 O(logn)，以此来优化这个问题。
>
> **关于为什么是6和8？**
>
> 因为红黑树的平均查找长度是log(n)，长度为8的时候，平均查找长度为3，如果继续使用链表，平均查找长度为8/2=4，这才有转换为树的必要。链表长度如果是小于等于6，6/2=3，虽然速度也很快的，但是转化为树结构和生成树的时间并不会太短。
>
> 还有选择6和8，中间有个差值7可以有效防止链表和树频繁转换。假设一下，如果设计成链表个数超过8则链表转换成树结构，链表个数小于8则树结构转换成链表，如果一个HashMap不停的插入、删除元素，链表个数在8左右徘徊，就会频繁的发生树转链表、链表转树，效率会很低。

   

#### hashmap什么时候会触发扩容？   

> 当向容器添加元素的时候，会判断当前容器的元素个数，如果大于等于阈值—即当前数组的长度乘以加载因子的值的时候，就要自动扩容啦。



#### jdk1.8之前并发操作hashmap时为什么会有死循环的问题？    

> 在 rehash 时，链表以倒序进行重新排列。
>
> ```java
> void transfer(Entry[] newTable, boolean rehash) {
>     int newCapacity = newTable.length;
>     for (Entry<K,V> e : table) {
>         while(null != e) {
>             Entry<K,V> next = e.next;
>             if (rehash) {
>                 e.hash = null == e.key ? 0 : hash(e.key);
>             }
>             int i = indexFor(e.hash, newCapacity);
>             e.next = newTable[i];
>             newTable[i] = e; 	// 每次都将节点放在链表头，导致最终链表顺序和原链表顺序相反
>             e = next;
>         }
>     }
> }
> ```
>
> 在JDK1.8后，这个问题得到了优化：
>
> ```Java
> for (int j = 0; j < oldCap; ++j) {
>     Node<K,V> e;
>     if ((e = oldTab[j]) != null) {
>         oldTab[j] = null;
>         if (e.next == null)
>             newTab[e.hash & (newCap - 1)] = e;
>         else if (e instanceof TreeNode)
>             ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
>         else { // preserve order
>             Node<K,V> loHead = null, loTail = null;		// 1、
>             Node<K,V> hiHead = null, hiTail = null;
>             Node<K,V> next;
>             do {
>                 next = e.next;
>                 if ((e.hash & oldCap) == 0) {
>                     if (loTail == null)
>                         loHead = e;
>                     else
>                         loTail.next = e;				// 2、
>                     loTail = e;
>                 }
>                 else {
>                     if (hiTail == null)
>                         hiHead = e;
>                     else
>                         hiTail.next = e;				// 2、
>                     hiTail = e;
>                 }
>             } while ((e = next) != null);
>             if (loTail != null) {
>                 loTail.next = null;
>                 newTab[j] = loHead;						// 3、
>             }
>             if (hiTail != null) {
>                 hiTail.next = null;
>                 newTab[j + oldCap] = hiHead;
>             }
>         }
>     }
> }
> ```
>
> 

#### hashmap扩容时每个entry需要再计算一次hash吗？    

> 不用计算；

#### hashmap的数组长度为什么要保证是2的幂？    

> 源码中他们采用了**延迟初始化操作**，也就是table只有在用到的时候才初始化，如果你不对他进行`put`等操作的话，table的长度永远为"零"。
>
> + **分布均匀**：如果不是2的n次方，那么有些位置上是永远不会被用到；
>   + table长度如果不是2的幂，则`table.length`的二进制有些位上会出现0，在计算index（=HashCode（Key） & （Length - 1）)时，就会出现有些index结果的出现几率会更大，而有些index结果永远不会出现。
> + **方便计算**，扩容后计算新位置，非常方便
>   + 当容量一定是2^n时，h & (length - 1) == h % length；
>
> + **计算索引需要**
>   + `hash&(newTable.length-1)` ==  `hash&(oldTable.length-1)+hash&oldTable.length`
>   + 因为table的长度一定是2的n次方，也就是oldCap 一定是2的n次方，也就是说 oldCap有且只有一位是1，而且这个位置在最高位；所以`hash&oldTable.length==oldTable.length`
>   + 扩容代码中，使用 e 节点的 hash 值跟 oldCap 进行位与运算，以此决定将节点分布到 “原索引位置” 或者 “原索引 + oldCap 位置” 上。
>
> **tableSizeFor的功能**（不考虑大于最大容量的情况）
>
> + 返回**大于输入参数且最近的2的整数次幂的数**。
>
> + ```java
>   static final int tableSizeFor(int cap) {
>       int n = cap - 1;
>       n |= n >>> 1;
>       n |= n >>> 2;
>       n |= n >>> 4;
>       n |= n >>> 8;
>       n |= n >>> 16;
>       return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
>   }
>   ```

#### 如何用LinkedHashMap实现LRU？    

> **LinkedHashMap介绍**
>
> 从LinkedHashMap的定义里面可以看到它单独维护了一个双向链表，用于记录元素插入的顺序。这也是为什么我们打印LinkedHashMap的时候可以按照插入顺序打印的支撑。而accessOrder属性则指明了进行遍历时是按照什么顺序进行访问,我们可以通过它的构造方法自己指定顺序。
>
> **当accessOrder=true**
>
> 在插入的时候LinkedHashMap复写了HashMap的newNode以及newTreeNode方法,并且在方法内部更新了双向链表的指向关系。
>
> 同时插入的时候调用了afterNodeAccess()方法以及afterNodeInsertion()方法，在HashMap中这两个方法是空实现，而在LinkedHashMap中则有具体实现,这两个方法也是专门给LinkedHashMap进行回调处理的。
>
> afterNodeAccess()方法中如果accessOrder=true时会移动节点到双向链表尾部。当我们在调用map.get()方法的时候如果accessOrder=true也会调用这个方法，这就是为什么访问顺序输出时访问到的元素移动到链表尾部的原因。
>
> **afterNodeInsertion(boolean evict)**
>
> ```java
> // evict如果为false，则表处于创建模式,当我们new HashMap(Map map)的时候就处于创建模式
> void afterNodeInsertion(boolean evict) { // possibly remove eldest
>   LinkedHashMap.Entry<K,V> first;
> 
>   // removeEldestEntry 总是返回false,所以下面的代码不会执行。
>   if (evict && (first = head) != null && removeEldestEntry(first)) {
>       K key = first.key;
>       removeNode(hash(key), key, null, false, true);
>   }
> }
> ```
>
> **实现LRU**
>
> ```java
> public class LRUCache<K,V> extends LinkedHashMap<K,V> {
>     
>   private int cacheSize;
>   
>   public LRUCache(int cacheSize) {
>       super(16,0.75f,true);
>       this.cacheSize = cacheSize;
>   }
> 
>   /**
>    * 判断元素个数是否超过缓存容量
>    */
>   @Override
>   protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
>       return size() > cacheSize;
>   }
> }
> ```
>
> 

#### 如何用TreeMap实现一致性hash？

> **普通的 hash 算法**通常都是对机器数量进行取余.
>
> 当使用负载均衡的时候，负载均衡器根据对象的 key 对机器进行取余，这个时候，原有的 key 取余现有的机器数 4 就找不到那台机器了！笨一点的办法，就是在增加机器的时候，清除所有缓存，但这会导致缓存击穿甚至缓存雪崩，严重情况下引发 DB 宕机。
>
> **一致性hash**
>
> 可以假设有一个 2 的 32 次方的环形，缓存节点通过 hash 落在环上。而对象的添加也是使用 hash，但很大的几率是 hash 不到缓存节点的。怎么办呢？**找离他最近的那个节点。** 比如顺时针找前面那个节点。
>
> **一致性 hash 有什么问题呢？**
>
> 集群环境负载不够均衡。
>
> 原因是：如果缓存节点分布不均匀。
>
> 怎么办？
>
> 可以在不均的地方给他弄均匀。在空闲的地方加入 **虚拟节点**，这些节点的数据映射到真实节点上，就可以了。





