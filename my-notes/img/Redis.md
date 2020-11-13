## Redis

#### 1. Redis 数据结构

1. String 

   > 常用命令：set, get, decr, incr, mget, setex
   >
   > 最简单的 key-value 类型；常规 key-value 缓存应用；
   >
   > 常规计数：微博数，粉丝数等

2. Hash

   > 常用命令：hget, hset, hgetall 
   >
   > 适合用于存储对象

3. List

   > 常用命令：lpush, rpush, lpop, rpop, lrange
   >
   > List 的实现是一个双向链表，支持反向查找和遍历；
   >
   > ⽐如微博的关注列表，粉丝列表，消息列表等功能都可以⽤ Redis 的 list 结构来实现  

4. Set

   > 常用命令：sadd, spop, smembers, sunion
   >
   > 类似于 list，区别之处在于是自动去重的，
   >
   > 可以基于 set 实现交集、并集、差集操作

5. ZSet

   > 常用命令：zadd, zrange, zrem, zcard
   >
   > 有序集合，zset 内增加了一个权重参数 score，使得集合中的元素能够按 score 进⾏有序排列 

6. Pub/Sub（发布/订阅）

   > 常用命令：publish、subscribe、unsubscribe
   >
   > 发布消息：publish channel_name message_hello
   >
   > 订阅消息：subscribe channel_name

7. Geo

   > 常用命令：geoadd, geopos, geodist, georadius, georadiusbymember, geohash, zrem
   >
   > 支持存储地理位置信息用来实现诸如附近位置，摇一摇等依赖于地理位置信息的功能。geo 的数据类型为 zset。
   >
   > 获取两个位置间的距离：geodist key member1 member2 [m/km/mi/ft]

8. HyperLoglog

   > 常用命令：pfadd, pfcount, pfmerge
   >
   > 用来做基数（不重复元素）统计的算法，优点是，在输入元素的数量或者体积非常非常大时，计算基数所需的空间总是固定的、并且是很小的
   >
   > 可以用来统计网站的独立访客数（UV）

9. Bitmaps

   > 常用命令：setbit, getbit, bitcount, bitop
   >
   > 基于 String 结构，存储的二进制 bit 数组；位上的值要么是 1 要么是 0；支持位运算。
   >
   > 可以用来统计网站的独立访客数（UV）

#### 2. Redis 数据结构的底层原理

1. SDS（string）

   SDS 具有常数复杂度获取字符串长度，杜绝了缓存区的溢出，**减少了修改字符串长度时所需的内存重分配次数**，以及二进制安全能存储各种类型的文件。

2. 双向链表（list）

   可以保存各种不同类型的值，除了用作列表键，还在发布与订阅、慢查询、监视器等方面发挥作用。

3. 字典（hash、set）

   字典底层使用哈希表实现，每个字典通常有两个哈希表，一个平时使用，另一个用于 rehash 时使用，使用链地址法解决哈希冲突。

4. 跳跃表（zset）

   跳跃表通常是有序集合的底层实现之一，表中的节点按照分值大小进行排序。

5. 整数集合（set）

   整数集合是集合键的底层实现之一，底层由数组构成，升级特性能尽可能的节省内存。

6. 压缩列表（zset、hash、list）

   压缩列表是 Redis 为节省内存而开发的顺序型数据结构，通常作为列表键和哈希键的底层实现之一。

7. 对象

#### 3. Redis 线程模型

Redis 内部使用文件事件处理器 **file event handler**，这个文件事件处理器是单线程的，所以 Redis 才被叫做单线程模型。它采用 IO 多路复用机制同时监听多个 Socket，根据 Socket 上的事件来选择对应是时间处理器进行处理。

> 文件事件处理器的结构包含 4 个部分：
>
> - 多个 Socket
> - IO 多路复用程序
> - 文件事件分派器
> - 事件处理器（连接应答处理器、命令请求处理器、命令回复处理器）  

多个 socket 可能会并发产⽣不同的操作，每个操作对应不同的⽂件事件，但是 IO 多路复⽤程序会监听多个 socket，会将 socket 产⽣的事件放⼊队列中排队，事件分派器每次从队列中取出⼀个事件，把该事件交给对应的事件处理器进⾏处理。  

#### 4. Redis 内存淘汰机制

定期删除 -->> 惰性删除 -->> 内存淘汰机制

1. volatile-lru：从已设置过期时间的数据集中，挑选最近做少使用的数据淘汰
2. volatile-ttl：从已设置过期时间的数据集中，挑选即将要过期的数据淘汰
3. volatile-random：从已设置过期时间的数据集中，任意挑选数据淘汰
4. allkeys-lru：从所有键中，挑选最近最少使用的淘汰（这个是最常用的）
5. allkeys-random：从所有键中，任意挑选数据淘汰
6. on-eviction：禁止删除数据，当内存不足以容纳新数据时，写数据操作会报错
7. volatile-lfu：从已设置过期时间的数据集中，挑选最不经常使用的数据淘汰
8. allkeys-lfu：从所有键中，挑选最不常用的数据淘汰

#### 5. [内存淘汰算法](https://www.jianshu.com/p/c8aeb3eee6bc)

1. LRU 最近最少使用（内存置换算法）

   如果一个数据在最近一段时间内没有被访问到，那么可以认为在将来被访问到的可能性也很小

   1. 新数据插入链表头部

   2. 每当缓存命中时（即缓存数据被访问），则将数据移动到链表头部

   3. 当链表满的时候，将链表尾部的数据丢弃

      **HashMap 实现：**
      
      **LinkedHashMap 实现：**

2. LFU 最近最不经常使用（内存置换算法）

   如果一个数据在最近一段时间内很少被访问到，那么可以认为在将来它被访问的可能性也很小

   LFU 算法的描述：

   

#### 6. Redis 持久化机制

1. RDB（快照）

   Redis 可以通过快照来获的存储在内存中的数据在某个时间点的副本。Redis 创建快照后，可以对快照进行备份，将快照复制到其他服务器上，从而创建具有相同数据的服务器副本，还可以将快照留在原地以便重启服务的时候使用。

   > 配置 redis.conf
   >
   > save 900 1 		# 在 900s（15 分组）内，如果至少有一个 key 发生变化，则触发 BGSAVE 命令创建快照
   >
   > save 300 10 	  # 在 300s（5分组）内，如果至少有 10 个 key 发生变化，则触发 BGSAVE 命令创建快照
   >
   > save 60 1000	 # 在 60s（1分组）内，如果至少有1000个 key 发生变化，则触发 BGSAVE 命令创建快照

2. AOF（之追加文件）

   AOF 持久化实时性更好，默认情况下，Redis 没有开启 AOF 持久化，可以在 redis.conf 中添加 **appendonly yes** 来开启。

   开启 AOF 持久化后，每执行一条会更改 Redis 中数据的命令，Redis 就回将该命令写入硬盘中的 AOF 文件。

   > AOF 持久化方式：
   >
   > ​	appendfsync    always 		# 每次有数据修改发生时，都会写入 AOF 文件，严重影响 Redis 性能
   >
   > ​	appendfsync    everysec      # 每秒钟同步一次
   >
   > ​	appendfsync    no                 # 让操作系统决定何时同步

3. Redis 4.0 对持久化的优化

   Redis 4.0 之后开始支持 RDB 和 AOF 的混合持久化（默认关闭，通过 aof-use-rdb-preamble 开启）。

4. AOF 重写

   通过读取服务器当前的数据库状态来实现的。
   
   **重写时的数据一致性问题：**AOF 重写缓冲区。
   
   这个缓冲区在服务器创建子进程之后开始使用，当 Redis 服务器执行完一个写命令后，他会同时将这个命令发送给 AOF 缓冲区和 AOF 重写缓冲区。
   
5. 集群复制模式

   当服务器运行在复制模式下时，从服务器的过期键删除动作由主服务器控制：

   > **·** 主服务器在删除一个过期键之后，会显式地向所有从服务器发送一个 DEL 命令，告知从服务器删除这个过期键。
   >
   > **·** 从服务器在执行客户端发送的读命令时，即使碰到过期键也不会将过期键删除，而是继续像处理未过期的键一样来处理过期键。
   >
   > **·** 从服务器只有在接到主服务器发来的 DEL 命令之后，才会删除过期键。

   通过由主服务器来控制从服务器统一地删除过期键，可以保证主从服务器数据的一致性，也正是因为这个原因，当一个过期键仍然存在于主服务器的数据库时，这个过期键在从服务器里的复制品也会继续存在。

#### 7. Redis 事务

Redis 通过 MULTI、EXEC、WATCH 等命令来实现事务功能。事务提供了一种将多个请求打包，然后一次性、按顺序的执行多个命令的机制，并且在事务执行期间，服务器不会中断事务而改去执行其他客户端的命令请求。

> Redis 事务中，如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚

#### 8. Redis 主从、哨兵

1. 主、从

2. 哨兵

   > 作用：
   >
   > - 通过发送命令，让 Redis 服务器返回其监控其运行状态，包括主服务器，从服务器
   > - 当哨兵监控到主服务器 master 宕机，会自动将 slave 切换为 master，然后通过**发布/订阅**模式通知其他服务器，修改配置文件，切换主机
   >
   > **故障切换（failover）**过程：
   >
   > 假设主服务器宕机，哨兵 1 先检测到这个结果，系统并不会马上进行 failover，仅仅是哨兵 1 主观认为主服务器不可用，这个现象称为**主观下线**。当后边的哨兵也检测到主服务器不可用，并且数量达到一定值时，那么哨兵之间就会进行一次投票，投票的结果由一个哨兵发起，进行 failover 操作。切换成功后，就灰通过发布/订阅模式，让各个哨兵监控的从服务器实现切换主机，这个称为客观下线。这样对于客户端而言，一切都是透明的。

#### 9. Redis 集群分片原理

16384 个槽（类似一致性 hash？）

#### 10. 缓存雪崩、缓存穿透、缓存击穿

**缓存雪崩：**缓存同一时间大面积失效，导致后面的请求都会落到数据库上，造成数据库短时间内承受大量请求而崩溃。

​	**解决：**把每个 key 的失效时间都加上一个随机值。

**缓存穿透：**大量请求的 key 根本不存在与缓存中，导致请求直接落到数据库上，跟没没有经过缓存这一层。

​	**解决：**做参数校验，将缓存中对应 key 的 value 设置为 null，布隆过滤器。

**缓存击穿：**指一个热点 key 在不停的扛着大并发，当该缓存失效的瞬间，持续的大并发就会穿破缓存，直接落到数据库。

​	**解决：**设置热点数据永不过期，或者，对请求数据库叫互斥锁。

#### 11. Redis 的并发竞争 Key 问题

所谓 Redis 并发竞争 key 的问题，就是多个客户端同时对一个 key 进行操作，但最后的执行顺序和我们期望的不一致，导致最终的结果不同。

​	**解决：**添加分布式锁。

​	基于 zookeeper 临时有序节点可以实现的分布式锁。

​    ⼤致思想为：每个客户端对某个⽅法加锁时，在 zookeeper 上的与该⽅法对应的指定节点的⽬录下，⽣成⼀个唯⼀的瞬时有序节点。 判断是否获取锁的⽅式很简单，只需要判断有序节点中序号最⼩的⼀个。 当释放锁的时候，只需将这个瞬时节点删除即可。同时，其可以避免服务宕机导致的锁⽆法释放，⽽产⽣的死锁问题。完成业务流程后，删除对应的⼦节点释放锁。  

#### 12. 缓存与数据库双写一致性问题

**经典的缓存 + 数据库读写模式：**

读的时候，先读缓存，缓存没有的话就读数据库，然后取出数据后放入缓存，同时返回响应。

更新的时候，先更新数据库，然后在删除缓存。

**简单情况：**

**问题：**先修改数据库，再删除缓存。如果删除缓存失败了，那么会导致数据库中是新数据，缓存中是旧数据，数据就出现了不一致。

**解决：**先删除缓存，再修改数据库

**复杂情况：**

**问题：**数据发生了变更，先删除了缓存，然后要去修改数据库，此时还没修改。一个请求过来，去读缓存，发现缓存空了，去查询数据库，查到了修改前的旧数据，放到了缓存中。随后数据变更的程序完成了数据库的修改。完了，数据库和缓存中的数据不一样了...

**解决：**

#### 13. 槽 slot

Redis Cluster 中有一个 16384 长度的槽的概念，这个槽是一个虚拟的槽，并不是真正存在的。

#### 14. redlock

SET key value NX PX time

#### 15. 数据恢复

**RDB方式：**

RDB 文件的载入工作是在服务器启动时自动执行的，没有专门用于载入 RDB 文件的命令，只要 Redis 服务器在启动时检测到 RDB 文件存在，它就会自动载入RDB 文件，服务器在载入 RDB 文件期间，会一直处于阻塞状态，直到载入工作完成为止。

**AOF方式：**

服务器在启动时，通过载入和执行 AOF 文件中保存的命令来还原服务器关闭之前的数据库状态，具体过程：

（1）载入 AOF 文件

（2）创建模拟客户端

（3）从 AOF 文件中读取一条命令

（4）使用模拟客户端执行命令

（5）循环读取并执行命令，直到全部完成

如果同时启用了 RDB 和 AOF 方式，AOF 优先，启动时只加载 AOF 文件恢复数据



## 面试题

#### redis 为什么单线程？

#### IO 多路复用？

#### redis 和 memcached 的优缺点？

#### redis 五个对象？

#### 每个对象的底层数据结构？

#### redis 的布隆过滤器(引申个布谷鸟过滤器)？

#### hyperLogLog(基于伯努利实验) ？

#### redis 的过期删除机制？

#### 淘汰机制？

#### redis 的 RDB 和 AOF？

#### redis 的事务？

#### redis 的主从?

#### 哨兵？

#### 集群？

redis 是将一个 16384 个槽分给集群中的节点，每个节点通过 gossip 消息得知其他节点的消息，并在自身的 clusterState 中记录了所有的节点的信息和槽数组的分配情况。

#### 客户端是如何访问集群的？

客户端会先访问集群中的一个节点，如果槽命中直接访问，如果不命中则返回 moved 指令，并告知槽实际存在的节点，然后再去访问（如果槽正在迁移，则返回 ack 指令，客户端会被引导去目标节点查找）。

#### 热点 key 问题？

1. 保障热点 key 过期问题，给不同的热点 key 分配随机的过期时间
2. 在 redis 中设置分布式锁，只有获取到锁的请求才可以穿透到数据库
3. 可以通过 hash 分 key，把一个 key 拆分成多个 key，分不到不同的节点，防止单点过热（类似一致性 hash）
4. 本地缓存，简单点就一个 HashMap 就行，或者 Guava cache 或者 Ehcache。用本地缓存来应对极热数据

#### redis 的分布式锁？

#### redlock？

#### redis 的一些优化？

#### 大 key 拆分？

#### 通过游标分批返回避免 key* 等命令阻塞？

#### 禁止 swap？

#### 考虑内存碎片等等？

#### 手撕 LRU

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

   















