## 一、 volatile 与 内存可见性



## 二、 CAS 与 原子变量



## 三、 并发容器

### 1. ConcurrentHashMap

### 2. CountDownLatch

### 3. CyclicBarrier

### 4. Semaphore



## 四、Callable 与 Future

### 1. Callable 与 Runnable

Runnable 是一个接口，在它里面只声明了一个 run() 方法；且 run() 方法返回值为 void 类型，所以在执行完任务之后无法返回任何结果。

Callable 位于 JUC 包下，它也是一个接口，在它里面也只声明了一个方法，只不过这个方法叫做 call()：且 call() 方法有返回值。Callable 通常需要和线程池结合使用。

```java
class Demo{
    public static void main(String[] args){
        ExecutorService threadPool = Executors.newSingleThreadExecutor();
        Future<String> future = threadPool.submit(new CallableExample());
    }
}
class CallableExample implements Callable {
	@Overread
    public Object call() throws Exception { 
		// TODO 
        return null; 
    }
}

```

### 2. Future

​		当 Callable 的 call() 方法完成时，结果必须存储在主线程已知的对象中，以便主线程可以知道该线程返回的结果。为此，可以使用 Future 对象。将 Future 视为保存结果的对象–它可能暂时不保存结果，但将来会保存（一旦 Callable 返回）。因此，Future 基本上**是主线程可以跟踪进度以及其他线程的结果的一种方式**。要实现此接口，必须重写5种方法，但是由于下面的示例使用了库中的具体实现，因此这里仅列出了重要的方法。

> - **public boolean cancel(boolean mayInterrupt) ：**用于停止任务。如果尚未启动，它将停止任务。如果已启动，则仅在 mayInterrupt 为 true 时才会中断任务。
> - **public Object get() ：**用于获取任务的结果。如果任务完成，它将立即返回结果，否则将等待任务完成，然后返回结果。此方法会**阻塞**直到异步计算完成。
> - **public Object get(Long timeout , TimeUnit unit) ：**获取异步执行结果，如果没有结果可用，此方法会**阻塞**，但是会有时间限制，如果阻塞时间超过设定的timeout时间，该方法将抛出异常。
> - **public boolean isDone() ：**如果任务结束（无论是正常结束或是中途取消还是发生异常），则返回 true，否则返回 false。
> - **public boolean isCanceller() ：**如果任务完成前被取消，则返回 true。

​		可以看到 Callable 和 Future 做两件事 ---- Callable 与 Runnable 类似，因为它封装了要在另一个线程上运行的任务，而 Future 用于存储从另一个线程获得的结果。实际上，future 也可以与 Runnable 一起使用。

​		要创建线程，需要 Runnable。为了获得结果，需要 future。

​		Java 库具有具体的 FutureTask 类型，该类型实现 Runnable 和 Future，并方便地将两种功能组合在一起。
​		可以通过为其构造函数提供 Callable 来创建 FutureTask。然后，将 FutureTask 对象提供给 Thread 的构造函数以创建 Thread 对象。因此，间接地使用 Callable 创建线程。

### 3. FutureTask

FutureTask 实现了 Future 接口和 Runnable 接口（即可以通过 Runnable 接口实现线程，也可以通过 Future 取得线程执行完后的结果）

futureTask中定义了7种状态，代表了7种不同的执行状态：

> private static final int NEW                  = 0; *//任务新建和执行中* 
>
> private static final int COMPLETING   = 1; *//任务将要执行完毕* 
>
> private static final int NORMAL           = 2; *//任务正常执行结束* 
>
> private static final int EXCEPTIONAL   = 3; *//任务异常* 
>
> private static final int CANCELLED       = 4; *//任务取消* 
>
> private static final int INTERRUPTING = 5; *//任务线程即将被中断* 
>
> private static final int INTERRUPTED   = 6; *//任务线程已中断*



## 五、Lock 与 Condition



## 六、ReadWriteLock



## 七、[线程八锁](https://www.cnblogs.com/shamao/p/11045282.html)

```Java
/*
 * 题目：判断打印的 "one" or "two" ？
 * 
 * 1. 两个普通同步方法, 两个线程, 标准打印, 打印?       //one  two
 * 2. 新增 Thread.sleep() 给 getOne(), 打印?       //one  two
 * 3. 新增普通方法 getThree(),          打印?       //three  one   two
 * 4. 两个普通同步方法，两个 Number 对象, 打印?        //two  one
 * 5. 修改 getOne() 为静态同步方法，     打印?        //two   one
 * 6. 修改两个方法均为静态同步方法，一个 Number 对象?         //one   two
 * 7. 一个静态同步方法，一个非静态同步方法，两个 Number 对象?  //two  one
 * 8. 两个静态同步方法，两个 Number 对象?                  //one  two
 * 
 * 线程八锁的关键：
 * 1. 非静态方法的锁默认为 this, 静态方法的锁为 对应的 Class 实例
 * 2. 某一个时刻内，只能有一个线程持有锁，无论几个方法。
 */
```



## 八、线程池

newFixedThreadPool：建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。

newSingleThreadExecutor：一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

newScheduledThreadPool：创建一个可定期或者延时执行任务的定长线程池，支持定时及周期性任务执行。 

newCachedThreadPool：创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。

```java
/**
 * corePoolSize 线程池基本大小
 * maximumPoolSize 线程池最大大小
 * keepAliveTime 线程存活保持时间
 * unit 时间单位
 * workQueue 任务队列
 * 		ArrayBlockingQueue：规定大小的 BlockingQueue，其构造必须指定大小。其所含的对象是FIFO顺序排序的。
 * 		LinkedBlockingQueue：大小不固定的 BlockingQueue，
 *			若其构造时指定大小，生成的BlockingQueue有大小限制，
 *			不指定大小，其大小有Integer.MAX_VALUE来决定。其所含的对象是FIFO顺序排序的。
 *      PriorityBlockingQueue：类似于 LinkedBlockingQueue，但是其所含对象的排序不是FIFO，而是依据对象的自然顺序或者构造函数的Comparator决定。
 * 		SynchronizedQueue：特殊的 BlockingQueue，对其的操作必须是放和取交替完成。
 * threadFactory 线程线程工厂
 * handler 线程饱和/拒绝策略
 * 		丢弃任务并抛出异常
 * 		丢弃任务但不抛出异常
 * 		丢弃队列最前面的任务，然后重新提交被拒绝的任务
 * 		由调用线程处理(执行)该任务
 */
public ThreadPoolExecutor
    (int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) 
{
    this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, Executors.defaultThreadFactory(), defaultHandler);
}
```



## 九、线程调度

### 1. sleep 休眠

​		sleep 方法是属于 **Thread** 类中的，sleep 过程中线程**不会释放锁**，只会**阻塞线程**，让出 cpu 给其他线程，但是他的监控状态依然保持着，当指定的时间到了又会自动恢复运行状态，**可中断**，sleep 给其他线程运行机会时不考虑线程的优先级，因此**会给低优先级的线程以运行的机会**

### 2. yield 暂停

​		和 sleep 一样都是 Thread 类的方法，都是暂停当前正在执行的线程对象，**不会释放资源锁**，和 sleep 不同的是 yield方法并不会让线程进入阻塞状态，而是让线程重回就绪状态，它只需要等待重新获取 CPU 执行时间，所以执行 yield() 的线程有可能在进入到可执行状态后马上又被执行。还有一点和 sleep 不同的是 yield 方法只能使同优先级或更高优先级的线程有执行的机会

### 3. join 挂起

​		等待调用 join 方法的线程结束之后，程序再继续执行，一般用于**等待异步线程执行完结果之后才能继续运行的场景**。例如：主线程创建并启动了子线程，如果子线程中药进行大量耗时运算计算某个数据值，而主线程要取得这个数据值才能运行，这时就要用到 join 方法了

### 4. wait 释放

​		wait 方法是属于 Object 类中的，wait 过程中线程会释放对象锁，只有当其他线程调用 notify 才能唤醒此线程。wait 使用时必须先获取对象锁，即必须在 synchronized 修饰的代码块中使用，那么相应的 notify 方法同样必须在 synchronized 修饰的代码块中使用，如果没有在synchronized 修饰的代码块中使用时运行时会抛出IllegalMonitorStateException的异常


## 十、Fork/Join 框架

把一个线程拆分为多个子线程执行，然后再汇总子线程的执行结果



## 十一、AQS