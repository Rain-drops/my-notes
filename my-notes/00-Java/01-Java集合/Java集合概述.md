# 简述

数据存储的常用结构有：栈、队列、数组、链表、树。

1. **栈**

   - 先进后出（即，存进去的元素，要在后它后面的元素依次取出后，才能取出该元素）。

   - 栈的入口、出口的都是栈的顶端位置

   - 压栈：就是存元素。即，把元素存储到栈的顶端位置，栈中已有元素依次向栈底方向移动一个位置。

   - 弹栈：就是取元素。即，把栈的顶端位置元素取出，栈中已有元素依次向栈顶方向移动一个位置。

2. **队列**

   - 先进先出（即，存进去的元素，要在它前面的元素依次取出后，才能取出该元素）。

   - 队列的入口、出口各占一侧。

3. **数组**

   - 查找元素快：通过索引，可以快速访问指定位置的元素
   
   - 增删元素慢：
   
     指定索引位置增加元素：需要创建一个新数组，将指定新元素存储在指定索引位置，再把原数组元素根据索引，复制到新数组对应索引的位置。
   
     指定索引位置删除元素：需要创建一个新数组，把原数组元素根据索引，复制到新数组对应索引的位置，原数组中指定索引位置元素不复制到新数组中。

4. **链表**

   - 多个节点之间，通过地址进行连接。
   
   - 查找元素慢：想查找某个元素，需要通过连接的节点，依次向后查找指定元素
   
   - 增删元素快：
   
     增加元素：只需要修改连接下个元素的地址即可。
   
     删除元素：只需要修改连接下个元素的地址即可。

5. **树**

   - 树结构是一种非线性存储结构，存储的是具有“一对多”关系的数据元素的集合

   

# 1. List

## 1.1 ArrayList

1. 非线程安全的，多线程下会发生数据覆盖

2. 实现于 List、RandomAccess 接口。可以插入空数据，支持随机访问

  3. 有两个重要属性：elementData 数组和 size 大小。

  4. elementData 被 `transient`  关键词修饰，防止被自动序列化。ArrayList 自定义了序列化与反序列化，只序列化被使用的数据。

  5. 底层是动态数组，默认大小是 10，扩容是扩展 1.5 倍。

  6. 支持快速随机访问，即通过元素的序号快速获取元素对象。

  7. 插入和删除的时间复杂度受元素位置的影响，默认队尾插入O(1)。

  8. ArrayList 的空间花费 主要体现在列表的结尾会预留一定的容量空间

     > JDK 1.7 是头插法，resize 时，会造成死循环
     >
     > JDK 1.8 是尾插法，resize 时，不会造成死循环 

## 1.2 LinkedList

1. 非线程安全的

  2. JDK1.6 之前是循环链表，JDK1.7 之后是双向链表

  3. 不支持快速随机访问

  4. LinkedList 的空间花费体现在他每一个元素都需要消耗比 ArrayLIst 更多的空间，用来存放直接后续和直接前驱。

     > 头插（LinkedList.addFirst()） 和 尾插（LinkedList.addLast()）

5. node() 会以 O(n/2) 的性能去获取一个节点

   > 如果索引值大于链表大小的一半，那么尾节点开始遍历

   这样的效率非常低，特别当 index 越接近于中间值时。

## 1.3 Vector

1. 线程安全的
2. 底层数组
3. 直接在方法上加锁
4. 扩容是 2 倍



# 2. Set 

## 2.1 HashSet

  1. 非线程安全的
  2. 基于 HashMap 的 Key 实现
  3. 元素无序且唯一，唯一性是靠所存储元素类型是否重写hashCode()和equals()方法来保证的

  4. 允许且仅允许存储一个 null 元素


### 2.1.1 LinkedHashSet

1. 用一个链表来维护元素的插入顺序
 2. 基于 LinkedHashMap 



## 2.2 TreeSet

1. 元素唯一且有序
2. 基于 TreeMap 实现，本质是 红黑树 原理



# 3. Map

## 3.1 HashMap

 1. 非线程安全的

    > 主要表现在 put 时，线程 A 计算完数据要落到的桶的索引坐标，并且获取到该桶里边的链表头结点，此时线程 A 的时间片用完了，切换到线程 B，线程 B 执行同样的操作，但是数据记录成功，此时切换回线程 A，A 继续执行，会导致线程 B 插入的数据被覆盖，导致数据丢失。

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

 6. [多线程](https://blog.csdn.net/qq_22158743/article/details/88932176)

    > **JDK 1.7 下：**
    >
    > **数组(Hash) + 链表(Entry)**
    >
    > 并发场景下容易出现死循环：当两个线程同时对原有的旧的 Hash 表进行扩容时，当其中一个线程正在扩容时(遍历单向链表)，CPU 切换到另一个线程进行扩容操作，并且线程二中完成了所有扩容的操作，此时再切换的线程一中，就可能造成单向链表形成一个环，从而下一次查询操作时就有可能发生死循环。
    >
    > HashMap 在扩容时会调用 transfer 方法，就是将原 Hash 表上的元素(Entry)全部转移到新的 Hash 表上，且采用**头插法**。
    >
    > **单线程下 rehash 过程：**
    >
    > **原数据：**
    >
    > ![](../img/hashmap-rehash-data.png)
    >
    > **rehash：**
    >
    > ![](../img/hashmap-rehash-1.7.png)

    

 7. 遍历方式

    1. 迭代器（Iterator EntrySet）

       ```java
       Iterator<Map.Entry<String, Integer>> entryIterator = map.entrySet().iterator();
       while (entryIterator.hasNext()) {
           Map.Entry<String, Integer> next = entryIterator.next();
           System.out.println("key=" + next.getKey() + " value=" + next.getValue());
       }
       ```

    2. 迭代器（Iterator KeySet）

       ```java
       Iterator<String> iterator = map.keySet().iterator();
       while (iterator.hasNext()){
           String key = iterator.next();
           System.out.println("key=" + key + " value=" + map.get(key));
       }
       ```

    3. For Each（EntrySet）

    4. For Each（KeySet）

    5. Lambda 表达式

       ```java
       map.foreach((k, v) -> {println(k)}});
       ```

    6. Streams API 单线程

       ```java
       map.entrySet().stream.foreach(entry -> {println(entry.getKey())})
       ```

    7. Streams API 多线程

       ```java
       map.entrySet().parallelStream.foreach(entry -> {println(entry.getKey())})
       ```

 8. put 方法解析

    ```java
    public V put(K key, V value) {
            /**
             * 四个参数，第一个hash值，第四个参数表示如果该key存在值，如果为null的话，则插入新的value，
             * 最后一个参数，在hashMap中没有用，可以不用管，使用默认的即可
             **/
            return putVal(hash(key), key, value, false, true);
        }
     
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        // tab 哈希数组，p 该哈希桶的首节点，n hashMap的长度，i 计算出的数组下标
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 获取长度并进行扩容，使用的是懒加载，table一开始是没有加载的，等put后才开始加载
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        /**
         * 如果计算出的该哈希桶的位置没有值，则把新插入的key-value放到此处，此处就算没有插入成功，也就是发生哈希冲突时也会把哈希桶的首节点赋予p
         **/
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        // 发生哈希冲突的几种情况
        else {
            // e 临时节点的作用， k 存放该当前节点的key 
            Node<K,V> e; K k;
            // 第一种，插入的key-value的hash值，key都与当前节点的相等，e = p，则表示为首节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 第二种，hash值不等于首节点，判断该p是否属于红黑树的节点
            else if (p instanceof TreeNode)
                /**
                 * 为红黑树的节点，则在红黑树中进行添加，如果该节点已经存在，则返回该节点（不为null），
                 * 该值很重要，用来判断put操作是否成功，如果添加成功返回null
                 **/
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            // 第三种，hash值不等于首节点，不为红黑树的节点，则为链表的节点
            else {
                // 遍历该链表
                for (int binCount = 0; ; ++binCount) {
                    // 如果找到尾部，则表明添加的key-value没有重复，在尾部进行添加
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 判断是否要转换为红黑树结构
                        if (binCount >= TREEIFY_THRESHOLD - 1) 
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 如果链表中有重复的key，e则为当前重复的节点，结束循环
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // 有重复的key，则用待插入值进行覆盖，返回旧值。
            if (e != null) { 
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        // 到了此步骤，则表明待插入的key-value是没有key的重复，因为插入成功e节点的值为null
        // 修改次数+1
        ++modCount;
        // 实际长度+1，判断是否大于临界值，大于则扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        // 添加成功
        return null;
    }
    ```


### 2.1.1 LinkedHashMap

1. 有序的

   > **排序方式有两种：**
   >
   > 1. 根据写入顺序排序
   > 2. 根据访问顺序排序：每次 get 都会将访问的值移动到链表末尾。

2. 底层是继承于 HashMap 实现，由一个双向链表构成

4. 数据结构

   ```java
   public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>{
   	static class Entry<K,V> extends HashMap.Node<K,V> {
           Entry<K,V> before, after;
           Entry(int hash, K key, V value, Node<K,V> next) {
               super(hash, key, value, next);
           }
       }
       // 指针的 头结点
       private transient Entry<K,V> header;
       // 尾结点
       transient LinkedHashMap.Entry<K,V> tail;
       // 访问顺序
       private final boolean accessOrder;
   }
   ```

4. 



### 2.2.2 WeakHashMap



## 2.2 HashTable

1. Hashtable 继承自 Dictionary 类，而 HashMap 继承自 AbstractMap 类。

2. 同样基于 hash 表实现，都实现了 Map 接口。也是 k - v 结构。同样通过单链表解决冲突问题。

3. 采用**拉链法**实现哈希表。

   > **table**：为一个 Entry[] 数组类型，Entry 代表了“拉链”的节点，每一个 Entry 代表了一个键值对，哈希表的"key-value 键值对"都是存储在 Entry 数组中的。
   >
   > **count**：HashTable 的大小，注意这个大小并不是 HashTable 的容器大小，而是他所包含 Entry 键值对的数量。
   >
   > **threshold**：Hashtable 的阈值，用于判断是否需要调整 Hashtable 的容量。threshold 的值="容量*加载因子"。
   >
   > **loadFactor**：加载因子。
   >
   > **modCount**：用来实现“fail-fast”机制的（也就是快速失败）。所谓快速失败就是在并发集合中，其进行迭代操作时，若有其他线程对其进行结构性的修改，这时迭代器会立马感知到，并且立即抛出 ConcurrentModificationException 异常，而不是等到迭代完成之后才告诉你（你已经出错了）

 4. 使用 synchronized 修饰

  5. 哈希值的使用不同，HashTable 直接使用对象的 hashCode。而 HashMap 重新计算 hash 值

  6. hash 数组默认大小是 11，增加的方式是 old * 2 + 1

  7. key 和 value 都不允许为 null



## 3.3 ConcurrentHashMap

  1. JDK 1.7

     1. 数据结构

        JDK1.7 以前是采用**分段的数组(Segment) + 链表(HashEntry)**实现

        其中 Segment 继承与 ReentrantLock。

        HashEntry 中的 value 和 next 属性使用 volatile 关键词修饰，保证内存可见性。

     2. 实现线程安全的方式

        整个数组进行了分割分段（Segment），每一把锁只锁其中一部分数据，多线程访问容器里不同数据段的数据不会存在锁竞争。

     3. get 方法

        HashEntry 中的 value 和 next 属性使用 volatile 关键词修饰，保证内存可见性，所以**每次获取值时都是最新值**。

     4. put 方法 

        仍然需要加锁处理。

     5. size 方法

  2. JDK1.8 

     1. 数据结构

        采用**数组 + 链表(Node) + 红黑树**实现

        Node 中的 value 和 next 属性使用 volatile 关键词修饰，保证内存可见性。

     2. 实现线程安全的方式

        采用 synchronized 和 CAS 来操作。synchronized 只锁当前链表或红黑树的首节点，只要不 hash 冲突，就不会产生并发。

     3. get 方法

     4. put 方法

     5. size 方法


## 3.4 TreeMap

1. 不允许出现重复的 key

2. 基于红黑树实现

3. 可以插入 null 键，null 值；

4. 可以对元素进行排序；

5. 无序集合（插入和遍历顺序不一致）

   

## 3.5 IdentifyHashMap



# 4. 迭代器

## 1. 快速失败（fail-fast）

​	    快速失败是 Java 集合的一种错误检测机制。

​		迭代器在遍历时直接访问集合中的内容，并且在遍历过程中使用一个 modCount 变量。集合在被遍历期间如果内容发生变化，就会改变 modCount 的值。每当迭代器使用 hashNext()/next() 遍历下一个元素之前，都会检测 modCount 变量是否为 expectedmodCount 值，是的话就返回遍历；否则抛出异常，终止遍历。

​	    **在使用迭代器对集合进行遍历的时候，我们在多线程下操作非安全失败（fail-safe）的集合类可能就会触发 fail-fast 机制，导致抛出ConcurrentModificationException 异常。另外，单线程下。如果在遍历过程中对集合对象的内容进行了修改操作的话也会触发 fail-fast 机制。**

## 2. 安全失败（fail-safe）

​	    采⽤安全失败机制的集合容器，在遍历时不是直接在集合内容上访问的，⽽是先复制原有集合内容，在拷⻉的集合上进⾏遍历。所以，在遍历过程中对原集合所作的修改并不能被迭代器检测到，故不会抛 ConcurrentModificationException 异常。  

​		基于拷贝内容的优点是避免了 Concurrent Modification Exception，但同样地，迭代器并不能访问到修改后的内容，即：迭代器遍历的是开始遍历那一刻拿到的集合拷贝，在遍历期间原集合发生的修改迭代器是不知道的。

​		java.util.concurrent 包下的容器都是安全失败，可以在多线程下并发使用，并发修改



# 5. Collections

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




