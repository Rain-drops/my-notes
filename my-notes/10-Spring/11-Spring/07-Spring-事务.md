### 一、事务的原理

​		

```java
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {
	// ......
	@Nullable
    protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass, 
    								TransactionAspectSupport.InvocationCallback invocation) throws Throwable {
    	// 1. 获取@Transactional注解的相关参数
        TransactionAttributeSource tas = this.getTransactionAttributeSource();
        // 2. 获取事务管理器
        TransactionAttribute txAttr = tas != null ? tas.getTransactionAttribute(method, targetClass) : null;
        PlatformTransactionManager tm = this.determineTransactionManager(txAttr);
        String joinpointIdentification = this.methodIdentification(method, targetClass, txAttr);
        Object result;
        if (txAttr != null && tm instanceof CallbackPreferringPlatformTransactionManager) {
            // ......
        } else {
        	// 3. 获取TransactionInfo，包含了tm和TransactionStatus
            TransactionAspectSupport.TransactionInfo txInfo = 
            							this.createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
            result = null;
            try {
            	// 4.执行目标方法
                result = invocation.proceedWithInvocation();
            } catch (Throwable var17) {
            	// 5.回滚
                this.completeTransactionAfterThrowing(txInfo, var17);
                throw var17;
            } finally {
            	// 6.清理当前线程的事务相关信息
                this.cleanupTransactionInfo(txInfo);
            }
			// 7.提交事务
            this.commitTransactionAfterReturning(txInfo);
            return result;
        }
    }
}
```



### 二、事务的传播行为

​		Spring 在 TransactionDefinition 接口中规定了 7 种类型的事务传播行为，事务传播行为是 Spring 框架独有的事务增强特性。

​		事务传播行为是用来描述由某一个事务传播行为修饰的方法被嵌套进另一个方法时的事务如何传播。

​		伪代码：

```java
 public void methodA(){
    methodB();
    //doSomething
 }
 
 @Transaction(Propagation=XXX)
 public void methodB(){
    //doSomething
 }
```

​		代码中`methodA()`方法嵌套调用了`methodB()`方法，`methodB()`的事务传播行为由`@Transaction(Propagation=XXX)`设置决定。这里需要注意的是`methodA()`并没有开启事务，某一个事务传播行为修饰的方法并不是必须要在开启事务的外围方法中调用。

#### 1、事务的七种传播行为

> 1> PROPAGATION_REQUIRED: 支持当前事务，假设当前没有事务，就新建一个事务。
>
> 2> PROPAGATION_SUPPORTS: 支持当前事务，假设当前没有事务，就以非事务方式运行。
>
> 3> PROPAGATION_MANDATORY: 支持当前事务，假设当前没有事务，就抛出异常。 
>
> 4> PROPAGATION_REQUIRES_NEW: 新建事务，假设当前存在事务，把当前事务挂起。
>
> 5> PROPAGATION_NOT_SUPPORTED: 以非事务方式运行操作。假设当前存在事务，就把当前事务挂起。
>
> 6> PROPAGATION_NEVER: 以非事务方式运行，假设当前存在事务，则抛出异常。
>
> 7> PROPAGATION_NESTED: 如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。

#### 2、事务的嵌套

