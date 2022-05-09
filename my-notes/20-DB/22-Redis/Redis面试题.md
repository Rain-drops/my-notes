## 面试题

### redis 为什么单线程？

### IO 多路复用？

### redis 和 memcached 的优缺点？

### redis 五个对象？

### 每个对象的底层数据结构？

### redis 的布隆过滤器(引申个布谷鸟过滤器)？

### 布隆过滤器

1. 先说下背景，肯定遇到这种情况，判断元素在不在一个集合里面，如果，集合里面的元素非常大，这个判断过程是非常耗时的，而且集合占用空间也很大。

2. 应用场景，网页黑名单，垃圾邮件过滤，电话黑名单，url去重，内容推荐等。

3. 原理：布隆过滤器实际上就是一个字节数组，字节数组的值是 0 或 1，在添加元素的时候，对值通过多个 hash 函数的计算，得到多个 0，1 然后在字节数组里面在相应的位置设置值。这样处理完所有的值之后，一个完整的布隆过滤器就完成了。之后就进入应用阶段了，判断值在不在布隆过滤器里面了，如果新输出的对象是之前处理放在布隆过滤器里面的，那就一定是存在，因为两次计算得到的hash值是一样的，肯定在，那对于新的对象了，这时就有可能会出现误杀了，新的值的 hash 值可能与老的值 hash 一样，于是布隆过滤器就认为，这个值是黑名单里的了，会造成误杀的结果。相当于就是宁愿杀错一个，不愿放过一个。

4. 改进：通常误杀的话，可以通过两个方法去补救，再建立一个白名单，从布隆器本身去优化，降低误杀率。

5. 再举例，头条给你推荐内容的时候，肯定要去查询一个的你的历史阅读记录，你看过的内容，一定是存在你的记录中的，新内容会有很小的机率认为是你之前看过的。

### 布谷鸟过滤器

布谷鸟过滤器源于布谷鸟Hash算法，布谷鸟Hash表有两张，分别两个Hash函数，当有新的数据插入的时候，它会计算出这个数据在两张表中对应的两个位置，这个数据一定会被存在这两个位置之一（表1或表2）。一旦发现其中一张表的位置被占，就将改位置原来的数据踢出，被踢出的数据就去另一张表找对应的位置。通过不断的踢出数据，最终所有数据都找到了自己的归宿。
但仍会有数据不断的踢出，最终形成循环，总有一个数据一直没办法找到落脚的位置，这代表布谷Hash表走到了极限，需要将Hash算法优化或Hash表扩容。

布谷鸟过滤器只会存储元素的指纹信息（几个bit，类似于布隆过滤器），由于不是存储了数据的全部信息，会有误判的可能。

由于布谷鸟过滤器在踢出数据时，需要再次计算原数据在另一种表的Hash值，因此作者设计Hash算法时将两个Hash函数变成了一个Hash函数，第一张表的备选位置是Hash(x),第二张表的备选位置是Hash(x)⊕hash(fingerprint(x))，即第一张表的位置与存储的指纹的Hash值做异或运算。这样可以直接用指纹的值 异或 原来位置的Hash值来计算出其另一张表的位置。

#### 优缺点

布谷鸟过滤器在错误率小于3%的时候空间性能是优于布隆过滤器的，而这个条件在实际应用中常常满足，所以一般来说它的空间性能是要优于布隆过滤器的。布谷鸟过滤器按照普通设计，只有两个Hash表，在查找的时候可以确保两次访存就可以做完，相比于布隆过滤器的K个Hash函数K次访存，在数据量很大不能全部装载在内存中的情况下，多次访问内存性能较差。当然，布谷过滤器也有其相应的缺点，当装填数据过多的时候，容易出现循环的问题，即插入失败的情况。另外，它还跟布隆过滤器共有的一个缺点，就是访问空间地址不连续，内存寻址消耗大。

> 布谷鸟过滤器优点：
>
> - 访问内存次数低
> - Hash函数计算简单
>
> 缺点：
>
> - 内存空间不联系，CPU消耗大
> - 容易出现装填循环问题
> - 删除数据时，Hash冲突会引起误删

### hyperLogLog(基于伯努利实验) ？

### redis 的过期删除机制？

### 淘汰机制？

### redis 的 RDB 和 AOF？

### redis 的事务？

### redis 的主从?

### 哨兵？

### 集群？

redis 是将一个 16384 个槽分给集群中的节点，每个节点通过 gossip 消息得知其他节点的消息，并在自身的 clusterState 中记录了所有的节点的信息和槽数组的分配情况。

### 客户端是如何访问集群的？

客户端会先访问集群中的一个节点，如果槽命中直接访问，如果不命中则返回 moved 指令，并告知槽实际存在的节点，然后再去访问（如果槽正在迁移，则返回 ack 指令，客户端会被引导去目标节点查找）。

### 热点 key 问题？

1. 保障热点 key 过期问题，给不同的热点 key 分配随机的过期时间
2. 在 redis 中设置分布式锁，只有获取到锁的请求才可以穿透到数据库
3. 可以通过 hash 分 key，把一个 key 拆分成多个 key，分不到不同的节点，防止单点过热（类似一致性 hash）
4. 本地缓存，简单点就一个 HashMap 就行，或者 Guava cache 或者 Ehcache。用本地缓存来应对极热数据

### redis 的分布式锁？

### redlock？

### redis 的一些优化？

### 大 key 拆分？

### 通过游标分批返回避免 key* 等命令阻塞？

### 禁止 swap？

### 考虑内存碎片等等？

### 手撕 LRU

**最久未使用算法：**最近使用的数据会在未来一段时间内仍然被使用，已经很久没有使用的数据可能在未来较长一段时期内仍然不会被使用。

1. 用 LinkedHashMap，我继承 LinkedHashMap，重写 removeEldestEntry

2. 用 HashMap，存储 key 和 node 的之间的关系，并且记录头尾节点的引用，并且链表是双向链表

   ```java
   /**
    *
    */
   public class Node {
       //键
       Object key;
       //值
       Object value;
       //上一个节点
       Node pre;
       //下一个节点
       Node next;
    
       public Node(Object key, Object value) {
           this.key = key;
           this.value = value;
       }
   }
   /**
    * 链表定义
    */
   public class LRU<K, V> {
       private int currentSize;//当前的大小
       private int capcity;//总容量
       private HashMap<K, Node> caches;//所有的node节点
       private Node first;//头节点
       private Node last;//尾节点
    
       public LRU(int size) {
           currentSize = 0;
           this.capcity = size;
           caches = new HashMap<K, Node>(size);
       }
    	/**
        * 添加元素
        * @param key
        * @param value
        */
       public void put(K key, V value) {
           Node node = caches.get(key);
           //如果新元素
           if (node == null) {
               //如果超过元素容纳量
               if (caches.size() >= capcity) {
                   //移除最后一个节点
                   caches.remove(last.key);
                   removeLast();
               }
               //创建新节点
               node = new Node(key,value);
               caches.put(key, node);
               currentSize++;
           }else {
               //已经存在的元素覆盖旧值
               node.value = value;
           }
           //把元素移动到首部
           moveToHead(node);
       }
       /**
        * 通过key获取元素
        * @param key
        * @return
        */
       public Object get(K key) {
           Node node = caches.get(key);
           if (node == null) {
               return null;
           }
           //把访问的节点移动到首部
           moveToHead(node);
           return node.value;
       }
       /**
        * 根据key移除节点
        * @param key
        * @return
        */
       public Object remove(K key) {
           Node node = caches.get(key);
           if (node != null) {
               if (node.pre != null) {
                   node.pre.next = node.next;
               }
               if (node.next != null) {
                   node.next.pre = node.pre;
               }
               if (node == first) {
                   first = node.next;
               }
               if (node == last) {
                   last = node.pre;
               }
           }
           return caches.remove(key);
       }
       /**
        * 把当前节点移动到首部
        * @param node
        */
       private void moveToHead(Node node) {
           if (first == node) {
               return;
           }
           if (node.next != null) {
               node.next.pre = node.pre;
           }
           if (node.pre != null) {
               node.pre.next = node.next;
           }
           if (node == last) {
               last = last.pre;
           }
           if (first == null || last == null) {
               first = last = node;
               return;
           }
           node.next = first;
           first.pre = node;
           first = node;
           first.pre = null;
       }
   }
   
   /**
    * 测试
    */
   public class TestApp {    
       public static void main(String[] args) {
           LRU<Integer, String> lru = new LRU<Integer, String>(5);
           lru.put(1, "a");
           lru.put(2, "b");
           lru.put(3, "c");
           lru.put(4, "d");
           lru.put(5, "e");
           System.out.println("原始链表为：" + lru.toString());
   
           lru.get(4);
           System.out.println("获取key为4的元素之后的链表：" + lru.toString());
   
           lru.put(6,"f");
           System.out.println("新添加一个key为6之后的链表：" + lru.toString());
   
           lru.remove(3);
           System.out.println("移除key=3的之后的链表" + lru.toString());
       }
   }
   ```

   











