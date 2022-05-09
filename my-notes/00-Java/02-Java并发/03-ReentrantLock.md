### 一、使用

​		synchronized 是在**软件层面**依赖 JVM，而 j.u.c.Lock 是在**硬件层面**依赖特殊的 CPU 指令。

​		非公平锁加锁过程：判断如果当前 state 为 0，说明当前没有线程占用锁，那么只有一个线程会 CAS 获得锁。那么其他线程会调用 acquire 方法来竞争锁（后续会加到同步队列中自旋或挂起）。当有其他线程 A 进来想要获得锁时，恰好此前的某一线程释放锁，那么 A 会在同步队列中所有等待获取锁的线程之前抢先获得锁。

​		公平锁加锁过程：所有来的线程必须扔到队列尾部。acquire 方法会向非公平锁一下先调用 tryAcquire 获取锁，但是只有队列为空或者本身就是 head 时，才会成功，如果队列非空则被放到队列尾部。

#### 1. [Condition](https://www.cnblogs.com/gemine/p/9039012.html) 条件变量

> 1. Synchronized 中，所有的线程都在同一个 object 的条件队列上等待。而 ReentrantLock 中，每个 condition 都维护了一个条件队列。
>
> 2. 每一个 **Lock** 可以有任意数据的 **Condition** 对象，**Condition** 是与 **Lock** 绑定的，所以就有 **Lock** 的公平性特性：如果是公平锁，线程为按照 FIFO 的顺序从 *Condition.await* 中释放，如果是非公平锁，那么后续的锁竞争就不保证 FIFO 顺序了。
>
> 3. Condition 接口定义的方法，**await** 对应于 **Object.wait**，**signal** 对应于 **Object.notify**，**signalAll** 对应于 **Object.notifyAll**。特别说明的是Condition 的接口改变名称就是为了避免与 Object 中的 *wait/notify/notifyAll* 的语义和使用上混淆。

##### 1.1 与 Lock 配合实现**等待/通知**模式。

```java
class DemoLock{
    private Lock lock = new ReentrantLocK();
    private Condition condition = lock.newCondition();    
    private void conditionWait(){
        lock.lock();
        try{
	        System.out.println("获取锁");
            System.out.println("等待获取信号");
            condition.await();
            System.out.println("获取到信号");
        }catch(Exection ex){
        }finally{
            lock.unlock();
        }
    }
    private void conditionSignal(){
        lock.lock();
        try{
	        System.out.println("获取锁");
            condition.await();
            System.out.println("发出信号");
        }catch(Exection ex){
        }finally{
            lock.unlock();
        }
    }
}
```



#### 2. await 实现逻辑：

> 1. 将线程 A 加入到条件等待队列中，如果最后一个节点是取消状态，则从对列中删除。
> 2. 线程 A 释放锁，实质上是线程 A 修改 AQS 的状态 state 为 0，并唤醒 AQS 等待队列中的线程 B，线程 B 被唤醒后，尝试获取锁，接下去的过程就不重复说明了。
> 3. 线程 A 释放锁并唤醒线程 B 之后，如果线程 A 不在 AQS 的同步队列中，线程 A 将通过 LockSupport.park 进行挂起操作。
> 4. 随后，线程 A 等待被唤醒，当线程 A 被唤醒时，会通过 acquireQueued 方法竞争锁，如果失败，继续挂起。如果成功，线程 A 从 await 位置恢复。

#### 3. signal 实现逻辑：

> 1. 接着上述场景，线程B执行了signal方法，取出条件队列中的第一个非CANCELLED节点线程，即线程A。另外，signalAll就是唤醒条件队列中所有非CANCELLED 节点线程。遇到CANCELLED线程就需要将其从队列中删除。
>
> 2. 通过CAS修改线程A的waitStatus为0，表示该节点已经不是等待条件状态，并将线程A插入到AQS的等待队列中。
>
> 3. 唤醒线程A，线程A和别的线程进行锁的竞争。

### 二、原理

ReentrantLock 主要利用 CAS + AQS 队列来实现。

**CAS**：Compare and Swap，比较并交换。CAS 有 3 个操作数：内存值 V、预期值 A、要修改的新值 B。当且仅当预期值 A 和内存值 V 相同时，将内存值 V 修改为B，否则什么都不做。该操作是一个原子操作，被广泛的应用在 Java 的底层实现中。在 Java 中，CAS 主要是由 sun.misc.Unsafe 这个类通过 JNI 调用 CPU 底层指令实现。

**AQS：**是一个用于构建锁和同步容器的框架。事实上 concurrent 包内许多类都是基于 AQS 构建，例如 ReentrantLock，Semaphore，CountDownLatch，ReentrantReadWriteLock，FutureTask 等。

​		AQS 使用一个 FIFO 的队列表示排队等待锁的线程，队列头节点称作“哨兵节点”或者“哑节点”，它不与任何线程关联。其他的节点与等待线程关联，每个节点维护一个等待状态 waitStatus。