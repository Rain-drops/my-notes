## SpringCloud

![](../img\spring-cloud.webp)

### 1. 注册中心

1. Eureka

   ![](../img\spring-cloud-eureka.png)

   

   **服务注册：**服务提供者启动时，会通过 Eureka Client 向 Eureka Server 进行注册，它提供自身的元数据，如 IP、端口，运行状态、主页等。Eureka Server 接收到请求后，将元数据信息存储在一个双层结构 Map 中，第一层的 key 是服务名，第二层的 key 是具体服务的实例名。 

   **服务续约：**Eureka Client 会每隔 30 秒发送一次心跳。如果 Eureka Server 90 秒内没有收到心跳，会将实例从其注册表中删除。

   **获取服务：**服务消费者在启动，Eureka Client 向注册中心发送 REST 请求给注册中心，从服务注册中心获取服务注册表。Eureka Server 维护一份只读的注册服务清单返回给客户端，同时该缓存清单每 30 秒更新一次。 

   **服务下线：**当服务操作进行正常关闭操作时，会触发一个服务下线的 REST 请求给 Eureka Server，注册中心收到请求时，服务端将该服务的状态置位下线，同时把该下线状态传播下去。

   **服务剔除：**将当前清单中超过 90 秒没有续约的服务从注册中心剔除。

   **高可用实现：**集群中，将自己注册到其他 Eureka Server 中

   **自我保护机制：**Eureka Server 会统计心跳失败的比例在 15 分钟内是否低于 85%，如果是，则认为客户端与注册中心出现了**网络故障**，Eureka Server 自动进入自我保护机制，此时：

   ​	a. Eureka Server 不再从注册表中移除因为长时间没收到心跳而应该过期的服务。

   ​	b. Eureka Server 仍然能够接收新服务的注册和查询请求，但不会被同步到其他节点。

   ​	c. 当网络稳定时，当前 Eureka Server 新的注册信息会被同步到其他节点。

   **Eureka 工作流程：**

   > 1. Eureka Server 启动成功后，等待服务端注册。如果配置了集群，集群之间定时通过 Replicate 同步注册表，每个 Eureka Server 都存在独立完整的服务注册表信息。
> 2. Eureka Client 启动时根据配置的 Eureka Server 地址去注册中心注册服务。
   > 3. Eureka Client 会每 30 秒向 Eureka Server 发送一次心跳请求，证明客户端正常。
   > 4. 当 Eureka Server 90 秒内没有收到 Eureka Client 的心跳，注册中心则认为该节点失败，会注销实例
   > 5. 单位时间内，Eureka Server 统计到大量 Eureka Client 没有上送心跳，则认为可能为网络异常，进入自我保护机制，不再剔除没有上送心跳的客户端。
   > 6. 当 Eureka Client 心跳请求回复正常后，Eureka Server 自动退出自我保护机制。
   > 7. Eureka Client 定时全量或者增量从注册中心同步注册表，并将获取到的信息缓存到本地。
   > 8. 服务调用时，Eureka Client 会先从本地缓存寻找调取的服务，如果获取不到，则从注册中心获取注册表，再同步到本地缓存。
   > 9. Eureka Client 获取目标服务器信息，发起服务调用
   > 10. Eureka Client 程序关闭时向 Eureka Server 发送取消请求，Eureka Server 将实例从注册表中删除。 
   
   **源码分析：**

   注册的信息会存放在 map 中，而且还是个两层的 `ConcurrentHashMap<String, Map<String, Lease<InstanceInfo>>>`，外层 map 的 key 是 appName，也就是**服务名**，内层 map 的 key 是 instanceId，也就是**实例名**。InstanceInfo ：实例

   

2.  Consul

   

### 2. 配置中心

1. Config

   1. 热加载原理

      ![](../img/SpringCloudConfig-热加载.png)

      > 1. 将修改的配置信息提交到 Git，触发 WebHook。WebHook 用 Http 的形式向 ConfigServer 发送 refresh 请求。
      > 2. ConfigServer 将这个消息发送给 spring cloud bus（用 kafka 或者 rabbitMQ 实现）
      > 3. 每个应用实例中有一个 config-client，将会收到这个 refresh 消息
      > 4. 如果 destination 和自己匹配，就执行刷新动作。 向 config-server 请求配置
      > 5. config-server 拉取配置仓库中的最新配置文件并转成相应的json格式
      > 6. 回传给 config-client，随后 config-client 将内容更新到上下文中。

### 3. [Ribbon（客户端[消费者端]负载均衡的工具）](https://www.cnblogs.com/king1302217/p/12144623.html)

Ribbon 实现客户端的负载均衡。 Consumer 端获取到了所有的服务列表之后，在其内部使用负载均衡算法，进行多个系统的调用。

**功能：**

1. 通过根据 DNS 和 IP 与服务端通信
2. 可以根据算法从多个服务中选取一个服务进行通信
3. 通过将客户端和服务器分成几个区域（zone）来建立客户端与服务器间的关系。客户端尽量访问和自己在相同区域的服务，减少服务延迟
4. 保留服务器的统计信息，ribbon 可以实现用于避免高延时或频繁访问故障的服务器
5. 保留区域（zone）的统计数据，ribbon 可以实现避免访问失败的区域

**组件：**

​	IRule：负载均衡。[轮询、随机、最小访问数、过滤故障服务与并发数超过阀值的服务、根据响应时间计算权重等]

​	IPing：检查服务列表是否都活。

​			DummyPing：默认返回 true，即认为所有服务永远活着

​			PingUrl：使用 HttpClient 调用服务的一个 URL，调用成功，则程序认为本次心跳成功，服务活着

​			NIWSDiscoveryPing：如果 Discovery Client 认为是在线，则程序认为本次心跳成功，服务活着 

​	ServerList：存储服务列表，分为静态和动态。动态则后台有个线程定时定时刷新个过滤服务列表

​	ServerListFilter：允许过滤配置，或动态获取具有所需特性的服务器列表

​	ServerListUpdater：用于动态更新服务器列表

​	IClientConfig：定义个种配置信息，用来初始化 ribbon 客户端和负载均衡

​	ILoadBalancer：定义软件负载平衡器操作的接口。动态更新一组服务列表及根据指定算法从现有服务器列表中选择一个服务



### 4. [Hystrix](https://blog.csdn.net/eson_15/article/details/86628622)

**雪崩效应：**

​	多个微服务之间调用的时候，假设微服务 A 调用微服务 B 和微服务 C，微服务 B 和微服务 C 又在调用其他微服务，这就是所谓的“扇出”。

​	如果扇出的链路上某个微服务（如：B 调用 C）的调用响应时间过长或者不可用，那么微服务 A 的调用就会阻塞占用越来越多的系统资源，进而引起系统崩溃。

**断路器：**

​	当某个微服务发生故障时，通过断路器，用户再次发起的请求都会被断路，不再调用响应服务，而是走服务降级策略 fallBack，向调用方返回一个预期的、可处理的默认响应，而不是长时间的等待或直接返回一个异常信息。保证服务调用方可以顺利的执行逻辑。	

**服务熔断：**当指定时间窗内，调用者的失败率达到了一定的阀值，那么 Hystrix 则会自动将服务 B 和 C 之间的请求都断了，以免导致雪崩效应。

**服务降级：**降级是为了更好的用户体验，当一个方法调用异常时，通过执行另一种代码逻辑来给用户友好的回复。Hystrix 的后备处理模式。

**资源隔离：**通过限流来限制对某一服务的访问量。默认提供两种隔离技术：**线程池**和**信号量**。

> **线程池：**一般作用于**对依赖服务的网络请求的调用和访问。**通过线程池的大小进行控制。
>
> **信号量：**适合**内部一些比较复杂的的业务逻辑的访问**，不涉及网络。通过信号量的容量来进行限流（同步阻塞的）。

**使用：**

1. Ribbon 中使用

   启动类注解 @EnableCircuitBreaker

   在 Controller Method 上注解 @HystrixCommand(fallbackMethod = "xxxMethod")

   在 xxxMethod 中编写异常后的处理逻辑代码

2. Feign 中使用

   定义 ClientService（即 controller 调用的 service 接口），类上注解 @FeignClient(value = "XXXXX", fallbackFactory = xxxFallbackFactory.class)

   定义一个 Hystrix 处理类（xxxFallbackFactory），实现 FallBackFactory<T> 接口（T 即 ClientService 接口），重写 create 方法，返回熔断后的处理

   配置文件中开启熔断（application.yml）

   > feign: 
   >
   > ​	hystrix: 
   >
   > ​		enabled: true

**原理：**

​	[RxJava](https://zhuanlan.zhihu.com/p/20687178)

### 5. Feign

一种声明式的、模板化的 Web Service 客户端，使得在请求远程服务时与调用本地方法有一样的编码体验，更快捷。

### 6. 网关

系统唯一的对外入口，介于客户端与服务端之间，用于对请求进行鉴权、限流、路由、监控等。

#### 1. Zuul

##### A. 说明

1. Zuul 原本采用**同步阻塞架构**，转型后叫作 Zuul2，采用**异步非阻塞架构**。

2. **核心**是一系列的 Filter，其本质类比于 Servlet 。

3. **限流：**自定义一个过滤器，继承 ZuulFilter，然后通过 GoogleGuava 限流方案。

4. **Zuul 提供了一个框架，可以对过滤器进行动态的加载，编译，运行。**
5. 

##### B. [Zuul 核心框架](https://blog.csdn.net/haha7289/article/details/54312043)

![](../img/spring-cloud-zull-核心框架.png) 

###### 	a. ZuulFilter

​		zuul 支持动加载 Filter 类文件。

​		实现原理是监控存放 Filter 文件的目录，定期扫描这些目录，如果发现有新 Filter 源码文件或者 Filter 源码文件有改动，则对文件进行编译加载。

###### 	b. Request 生命周期

​		![](../img/spring-cloud-zull-request生命周期.png)

###### 	c. ZuulServlet

​		Zuul 基于 Servlet 框架，ZuulServlet 用于处理所有的 Request。其可以认 Http Request 的入口。

###### 	d. Servlet 生命周期

​		Servlet 的生命周期有四个阶段：**加载并实例化**、**初始化**、**请求处理**、**销毁**。主要涉及到的方法有`init`、`service`、`doGet`、`doPost`、`destory`等

#### 2. Gateway

**核心概念：**

​	Route：网关的基础元素，由 ID，目标 URI、断言、过滤器组成。当请求到达网关时，由 Gateway Handler Mapping 通过断言进行路由匹配，当断言为真时匹配到路由。

​	Predicate：允许匹配来着 HTTP 的请求，例如请求头或者请求参数。简单来说就是匹配条件。

​	Filter：过滤器，可以在请求发出前后进行一些业务上的逻辑。

**限流：**Redis + Lua：技术实现高并发和高性能的限流方案。

**限流算法：**	

​	**计数器：**

​	**令牌桶：**

1. 假如用户配置的平均发送速率为 r，则每隔 1/r 秒一个令牌被加入到桶中；

 	2. 假设桶最多可以存发 b 个令牌。如果令牌到达时令牌桶已经满了，那么这个令牌会被丢弃；
 	3. 当一个 n 个字节的[数据包](https://baike.baidu.com/item/数据包)到达时，就从令牌桶中删除 n 个令牌，并且数据包被发送到网络；
 	4. 如果令牌桶中少于 n 个令牌，那么不会删除令牌，并且认为这个数据包在流量限制之外；

​	**漏桶：**

​	**滑动时间窗口：**

​	**三色速率标记法：**

### 7. Archaius



### 8. Bus

消息总栈。



### 9. Security

安全框架，提供认证和授权功能。

### 10. Sleuth



### 11. Admin



### 12. spring cloud 与 dubbo 的对比

![](../img/spring-cloud-与-dubbo-核心比较.jpg)

> **整体比较：**
>
> 1. dubbo 是二进制传输，占用带宽较小
> 2. springCloud 是 http 传输，一般会使用 json 报文，带宽消耗更大
> 3. dubbo 的开发难度较大，原因是 dubbo 的 jar 包依赖问题很多大型工程无法解决
> 4. springcloud的接口协议约定比较自由且松散，需要有强有力的行政措施来限制接口无序升级
> 5. dubbo 的注册中心可以选择 zk、redis 等，springCloud 可以选择 Eureka、Consul 等
>



