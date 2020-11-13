### JAVA集合

### 1. ArrayList 与 LinkedList、Vector

#### 1. ArrayList

1. 非线程安全的，多线程下会发生数据覆盖

  2. 底层是数组，默认大小是 10，扩容是扩展 1.5 倍

  3. 插入和删除的时间复杂度受元素位置的影响，默认队尾插入O(1)

  4. 支持快速随机访问，即通过元素的序号快速获取元素对象。

  5. ArrayList 的空间花费 主要体现在列表的结尾会预留一定的容量空间

     > JDK 1.7 是头插法，resize 时，会造成死循环
     >
     > JDK 1.8 是尾插法，resize 时，不会造成死循环 

#### 2. LinkedList

1. 非线程安全的

  2. JDK1.6 之前是循环链表，JDK1.7 之后是双向链表

  3. 不支持快速随机访问

  4. LinkedList 的空间花费体现在他每一个元素都需要消耗比 ArrayLIst 更多的空间，用来存放直接后续和直接前驱。

     > 头插（LinkedList.addFirst()） 和 尾插（LinkedList.addLast()）

#### 3. Vector

1. 线程安全的
2. 底层数组
3. 直接在方法上加锁
4. 扩容是 2 倍

#### 4. RandomAccess

​	RandomAccess 是一个标志接口，表明实现这个接口的 List 集合是支持快速随机访问的。也就是说，实现了这个接口的集合是支持 **快速随机访问** 策略的。

### 2. HashMap 与 HashTable、ConcurrentHashMap

#### 1. HashMap

 1. 非线程安全的

 2. 允许且仅允许一个 key 为 null

 3. 默认大小为 16，负载因子为 0.75，扩容是扩展 2 倍

 4. 底层数据结构

    数组 + 链表 + 红黑树

    当链表长度大于阀值（默认 8）时，将链表转化为红黑树（将链表转化为红黑树之前会先判断，当数组长度小于 64，会选择先进行数组扩容），以减少搜索时间。当红黑树节点数小于 6 时，会退回为链表。

    > **红黑树：**

    

 5. HashMap的长度为什么是 2 的幂次方

    > Hash 值的范围是 -2^31 ~~ 2^31 - 1，
    >
    > 数组下标的计算方法是 “（n - 1）& hash”

 6. 多线程

    > 

 7. 遍历方式

    1. 迭代器（Iterator EntrySet）

    2. 迭代器（Iterator KeySet）

    3. For Each（EntrySet）

    4. For Each（KeySet）

    5. Lambda 表达式

     ```java
     map.foreach((k, v) -> {println(k)}});
     ```

    6. Streams API 单线程

    ```
    map.entrySet().stream.foreach(entry -> {println(entry.getKey())})
    ```

    7. Streams API 多线程

    ```java
     map.entrySet().parallelStream.foreach(entry -> {println(entry.getKey())})
    ```

     

#### 2. ConcurrentHashMap

  1. 数据结构

     1. JDK1.7 以前是采用**分段的数组 + 链表**实现，JDK1.8 之后是采用**数组 + 链表 + 红黑树**实现

     2. 实现线程安全的方式

        1. JDK 1.7 以前对整个数组进行了分割分段（Segment），每一把锁只锁其中一部分数据，多线程访问容器里不同数据段的数据不会存在锁竞争。
        2. JDK 1.8 之后 采用 synchronized 和 CAS 来操作。synchronized 只锁当前链表或红黑树的首节点，只要不hash冲突，就不会产生并发。



#### 3. HashTable



#### 4. Collections

 1. 排序

    ```java
    void reverse(List list) // 反转
    void shuffle(List list) // 随机排序
    void short(List list)   // 升序排序
    void swap(List list, int i, int j)   // 交换两个元素的位置
    void rotate(List list, int distance) // 旋转。
    ```

 2. 查找、替换

    ```java
    int binarySearch(List list, Object key) // 对list进行二分查找，返回索引(list 必须是有序的)
    int max(Collection coll)				// 返回最大的元素
    void fill(List list, Object obj)		// ⽤指定的元素代替指定list中的所有元素。
    int frequency(Collection c, Object o)	// 统计元素出现次数
    boolean replaceAll(List list, Object oldVal, Object newVal) // ⽤新元素替换旧元素
    ```

 3. 同步

    ```java
    synchronizedCollection(Collection<T> c) // 返回指定 collection ⽀持的同步（线程安全的） collection。
    synchronizedList(List<T> list)			// 返回指定列表⽀持的同步（线程安全的）List。
    synchronizedMap(Map<K,V> m) 			// 返回由指定映射⽀持的同步（线程安全的）Map。
    synchronizedSet(Set<T> s) 				// 返回指定
    ```

    

#### 4. 快速失败（fail-fast）

​	快速失败是 Java 集合的一种错误检测机制。

​	**在使用迭代器对集合进行遍历的时候，我们在多线程下操作非安全失败（fail-safe）的集合类可能就会触发 fail-fast 机制，导致抛出ConcurrentModificationException 异常。另外，单线程下。如果在遍历过程中对集合对象的内容进行了修改操作的话也会触发 fail-fast 机制。**

#### 5. 安全失败（fail-safe）

​	采⽤安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，⽽是先复制原有集合内容，在拷⻉的集合上进⾏遍历。所以，在遍历过程中对原集合所作的修改并不能被迭代器检测到，故不会抛ConcurrentModificationException 异常。  

### 3. 迭代器











