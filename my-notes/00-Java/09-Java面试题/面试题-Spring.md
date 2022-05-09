### Spring

#### 1. 如何理解 Spring 中的 AOP 和 IOC/DI

**AOP: **面向切面编程，即把业务逻辑功能抽取出来，然后动态把这个功能注入到需要的方法中，这样从业务角度上看，AOP 达到了更进一步的解耦。

AOP 使用**横切**技术，把系统分为两个部分：**核心关注点**和**横切关注点**。业务处理的流程为核心关注点，与之关系不大的为横切关注点（如 权限认证，日志，事务处理）。

AOP 相关术语：

​	通知（Advice）：

​	切入点（JoinPoint）：

​	切入点（Pointcut）：

​	切面（Aspect）：

​	引入（introduction）：

​	目标（target）：

​	代理（proxy）:

​	织入（weaving）：

**IOC/DI：**控制反转/依赖注入

> 在 A 类中调用 B 类的方法，即 A 依赖 B，B 为被依赖对象。
>
> **传统做法：**
>
> (1) 直接在 A（方法）中 new 出 B 对象，然后调用 B 类的方法
>
> (2) 通过简单工厂获取 B 类对象，然后调用 B 类的方法 -- 摆脱了与 B 的耦合，却又与工厂产生了耦合
>
> **Spring 框架：**
>
> 在 Spring 中 B 的实例对象被看作 Bean 对象，这个 Bean 对象是由 Spring 容器进行创建和管理，Spring 会通过配置文件自动执行在 A 中对 B 对象的 setter 方法（即 A 中需要有对 B 对象的 setter 方法），如此，A 获取 B 的实例对象就不是由自己主动去获取，而是被动接受 Spring 给它设值，那么，这个主动变为被动，就可以理解为**控制反转**。 
>
> 从spring容器的角度上看，它负责把A的依赖对象B（B是被依赖对象）注入给了A，所以我们可以理解为**依赖注入**

#### 2. spring bean 的创建时间

> ​	对于 Spring BeanFactory，由于他的应用场合（内存或其他资源受限的场合），使用了延迟加载机制，只有在用户 getBean() 方法时，使用的 JavaBean 实例才会被创建。
>
> ​	对于 Spring ApplicationContext，一旦 ContextLoaderServlet 或者 ContextLoaderListener 初始化成功，所有的类型为 singleton，非延时加载的 JavaBean 实例将会被创建，prototype 类型的 JavaBean 和使用了延迟加载的 singleton 类型的 bean 在使用 getBean 方法时被创建。
>
> ​	实现了 BeanPostProcessor 接口的类会在容器启动的时候初始化。  

#### 3. spring 运行时修改 redis 的密码

> BeanPostProcessor 后置处理器

#### 4. spring cloud Config、动态修改配置文件



#### 5. spring cloud Eureka 服务暴露过程



#### 6. spring cloud Eureka 服务注册与发现的流程



#### 7. spring cloud Hystrix 断路器

##### 1. hystrix 功能  



##### 2. hystrix 怎么检测断路器是否要开启/关闭 

**断路器开启：**

1. 整个链路达到一定的阀值，默认情况下，10 秒内达到 20 个请求
2. 请求的错误百分比大于阀值，默认 50%

**断路器关闭：**

​	1. 断路器打开后，在一段时间内，命令不会再执行（一直触发回退），这段时间被称为“休眠期”（默认 5 秒）。“休眠期”结束后， Hystrix 会尝试性执行一次命令，如果执行成功，则关闭断路器并清空链路信息；如果执行失败。断路器则继续保持打开状态。

##### 3. hystrix 实现原理

1. 使用命令模式将所有对外部服务（或依赖关系）的调用包装在 HystrixCommand 或 HystrixObservableCommand 对象中，并将该对象放在单独的线程中执行；
2. 每个依赖都维护着一个线程池（或信号量），线程池被耗尽在拒绝请求（而不是让请求排队）；
3. 记录请求成功、失败、超时和线程拒绝；
4. 服务错误百分比超过了阈值，熔断器开关自动打开，一段时间内停止对该服务的所有请求；
5. 请求失败，被拒绝，超时或熔断时执行降级逻辑；
6. 近实时的监控指标和配置的修改；

##### 4. 源码解析

#### 8. spring cloud Gateway 网关



#### 9. spring cloud 关键注解

##### 1. 如何理解网关

##### 2. 网关带来的好处和坏处，如何解决



#### 10. SpringBootApplication 注解包含哪些注解

> @ComponentScan：开启包扫描，默认扫描同级及当前包下内容
>
> @EnableAutoConfiguration：开启自动配置
>
> @SpringBootConfiguration：声明当前类是一个配置类
>
> **四个元注解**
>
> @Target(ElementType.TYPE)：标志该注解可以作用于interface、class、enum。
>
> @Retention(RetentionPolicy.RUNTIME)：注解的保留策略；
>
> 这里的策略是：
> 当前注解会出现在.class字节码文件中，
> 并且在运行的时候可以通过反射被获得。
>
> @Documented：标志该注解可以被javadoc工具所引用。
>
> @Inherited：标志该注解可以被继承



#### 