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