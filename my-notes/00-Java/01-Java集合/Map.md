## 一、HashMap

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
    > ![](D:/Book/MyNotes/my-notes/img/hashmap-rehash-data.png)
    >
    > **rehash：**
    >
    > ![](D:/Book/MyNotes/my-notes/img/hashmap-rehash-1.7.png)

 7. HashMap [的扩容机制](https://zhuanlan.zhihu.com/p/114363420)

    默认大小 16；负载因子 0.75，即阀值为 16 * 0.75 = 12 ；扩容大小 1 倍，即 32。

    初始化时，比如有 1000 个元素，那么，为了让 0.75 * size > 1000，应该 new HashMap(2048)

    > HashMap 的容量变化：
    >
    > 1. 空参构造函数：实例化的 HashMap 默认内部数组是 null，即没有实例化，第一次调用 put 方法时，则会开始第一次初始化扩容，长度为 16。
    > 2. 有参构造函数：会根据指定的正整数找到**不小于指定容量的 2 的幂数**，将这个数赋值给阀值（threshold），第一次调用 put 时，会将阀值赋值给容量，然后让**阀值 = 容量 * 负载因子**
    > 3. 如果不是第一次扩容，则容量变为原来的两倍，阀值也变为原来的两倍。
    >
    > 元素迁移：
    >
    > 1. 由于数组的容量是以 2 的幂次方扩容的，那么新的位置要么在**原位置**，要么**在原长度 + 原位置**，如下图：
    >
    >    ![](D:/Book/MyNotes/my-notes/img/hashmap.png)
    >
    >    数组长度变为原来的两倍，表现在二进制上就是**多了一个高位参与数组下标确定**。此时，一个元素通过 hash 转换坐标的方法计算后，恰好出现一个现象：最高位是 0 则坐标不变，最高位是 1 则坐标变为 “10000 + 原坐标”，即 “原长度 + 原坐标”。如下图：
    >
    >    ![](D:/Book/MyNotes/my-notes/img/hashmap-2.jpg)
    >
    >    因此，在扩容时，不需要重新计算元素的 hash 了，只需要判断最高位是 1 还是 0 就好了。
    >
    >    JDK8 在迁移元素时是正序的，不会出现链表转置的发生。
    >
    >    如果某个桶内的元素超过 8 个，并且容量大于 64，则会将链表转化成红黑树，加快数据查询效率。
    >
    >    如果某个桶内的元素小于 6 个，则会将红黑树转化回链表
    >
    >    *( 图片来源于文末的参考链接 )*

    

 8. 遍历方式

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

 9. put 方法解析

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


### 1、LinkedHashMap

1. 有序的

   > **排序方式有两种：**
   >
   > 1. 根据写入顺序排序
   > 2. 根据访问顺序排序：每次 get 都会将访问的值移动到链表末尾。

2. 底层是继承于 HashMap 实现，由一个双向链表构成

3. 数据结构

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




### 2、WeakHashMap



## 二、HashTable

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



## 三、ConcurrentHashMap

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


## 四、TreeMap

1. 不允许出现重复的 key

2. 基于红黑树实现

3. 可以插入 null 键，null 值；

4. 可以对元素进行排序；

5. 无序集合（插入和遍历顺序不一致）

   

## 五、IdentityHashMap

