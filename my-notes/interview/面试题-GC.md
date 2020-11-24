### Java GC

#### 1. Java 运行时区域 及 各个区域的作用、java内存模型 及 为什么要这么设计

。。。。。。

#### 2. GC 算法

。。。。。。

#### 3. 谈下对 GC 的了解，何为垃圾，有哪些垃圾回收器，cms 和 g1 的区别，cms 的源码

**CMS：**

**G1：**

#### 4. 有没有排查过线上 oom 的问题，如何排查的



#### 5. 有没有使用过 jvm 自带的工具，如何使用的

jstate：监视虚拟机运行时状态信息的命令

> jstate -gc pid
>
> jstate -gcutil pid

jmap：

> jmap -dump:format=b,file=/opt/heap.hprof pid

jstack：

> jstack -l pid > /opt/deadlock.txt



#### 6. 遇到过线上服务器 cpu 飙高的情况没有，如何处理的

top -Hp pid -->> printf %d pid -->> jstack -l pid



#### 7. 假设有下图所示的一个 full gc 的图，纵向是内存使用情况，横向是时间，你如何排查这个 full gc 的问题，怎么去解决你说出来的这些问题

![](D:/Book/MyNotes/my-notes/img/面试题-FullGC-图1.png)

频繁创建大对象且对象存活时间短。