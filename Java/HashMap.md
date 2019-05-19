# HashMap

HashMap 底层是基于 **`数组 + 链表`** 组成的，不过在 jdk1.7 和 1.8 中具体实现稍有不同。

![img](https://i.loli.net/2019/05/08/5cd1d2be77958.jpg)

 HashMap 中比较核心的几个成员变量；

1. 初始化桶大小，因为底层是数组，所以这是数组默认的大小。
2. 桶最大值。
3. 默认的负载因子（0.75）
4. `table` 真正存放数据的数组。
5. `Map` 存放数量的大小。
6. 桶大小，可在初始化时显式指定。
7. 负载因子，可在初始化时显式指定。

给定的默认容量为 16，负载因子为 0.75。Map 在使用过程中不断的往里面存放数据，当数量达到了 `16 * 0.75 = 12` 就需要将当前 16 的容量进行扩容，而**扩容这个过程涉及到 rehash、复制数据等操作，所以非常消耗性能**。

因此通常建议能提前预估 HashMap 的大小最好，尽量的减少扩容带来的性能损耗。

## 1.7 HashMap缺点

> 当 Hash 冲突严重时，在桶上形成的链表会变的越来越长，这样在查询时的效率就会越来越低；时间复杂度为 `O(N)`。

因此 1.8 中重点优化了这个查询效率。

##  1.8 HashMap结构

![img](https://i.loli.net/2019/05/08/5cd1d2c1c1cd7.jpg)

原理上来说：ConcurrentHashMap 采用了分段锁技术，其中 Segment 继承于 ReentrantLock。不会像 HashTable 那样不管是 put 还是 get 操作都需要做同步处理，理论上 ConcurrentHashMap 支持 CurrencyLevel (Segment 数组数量)的线程并发。每当一个线程占用锁访问一个 Segment 时，不会影响到其他的 Segment。

## 
