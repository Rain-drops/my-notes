## 一、 volatile 与 内存可见性



## 二、 CAS 与 原子变量



## 三、 并发容器

### 1. ConcurrentHashMap

### 2. CountDownLatch

### 3. CyclicBarrier

### 4. Semaphore



## 四、Callable 与 FutureTask



## 五、Lock 与 Condition



## 六、ReadWriteLock



## 七、线程八锁

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



## 九、线程调度



## 十、Fork/Join 框架

把一个线程拆分为多个子线程执行，然后再汇总子线程的执行结果



## 十一、AQS