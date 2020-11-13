### URL

dubbo 采用 URL 的方式作为约定的参数类型，即公共契约，我们通过 URL 来进行交互、交流

> protocol://username:password@host:port/path?key=value&key=value
>
> - protocol：指的是 dubbo 中的各种协议，如：dubbo / thrift / http
> - username / password：用户名/密码
> - host/port：主机 / 端口
> - path：接口的名称
> - parameters：参数键值对 

### 配置解析

dubbo 利用了 spring 配置文件扩展了自定义的解析。像 dubbo.xsd 就是用来约束 XML 配置时候的标签和对应的属性用的，然后 Spring 在解析到自定义的标签的时候会查找 spring.schemas 和 spring.handlers。

![](D:/Book/MyNotes/img/dubbo-spring-schemas.png)

![](D:/Book/MyNotes/img/dubbo-spring-handlers.png)

​		spring.schemas 指明了约束文件的路径

​		spring.handlers 指明了利用该 handler 来解析标签

​		我们再来看一下 **DubboNamespaceHandler** 都干了啥。

​		![](D:/Book/MyNotes/img/dubbo-DubboNamespaceHandler.png)



### 服务暴露全流程



![](D:/Book/MyNotes/img/dubbo-1.png)

第一步，检测配置，并且组装成 url

第二步，暴露服务，包括暴露本地服务和远程服务

第三步，注册服务到注册中心

![](D:/Book/MyNotes/img/dubbo-2.png)

第一步，将服务实现类转化为 Invoker

第二步，将 Invoker 通过具体的协议转化为 Exporter

### 源码分析

如配置解析 DubboNamespaceHandler 所示，service 标签其实就是对应 ServiceBean。

![](D:/Book/MyNotes/img/dubbo-servicebean.png)

这里可以看到，它实现了 **ApplicationListener<ContextRefreshedEvent>**，这样就会**在 Spring IOC 刷新完成之后调用 onApplicationEvent 方法，而这个方法里做的就是服务暴露**，这就是服务暴露的启动点。



![](D:/Book/MyNotes/img/dubbo-onApplicationEvent.png)

​		可以看到，如果不是延迟暴露，并且还没暴露过，并且支持暴露的话，就执行 export() 方法，而 export 最终会调用父类的 export 方法。

​		![](D:/Book/MyNotes/img/dubbo-export.png)

​		主要就是检查了一些配置，确认需要暴露的话就暴露服务，doExport() 中都是一些检查配置的过程，重点应该是关注里边的 doExportUrls() 方法。

​		![](D:/Book/MyNotes/img/dubbo-doExportUrls.png)

​		可以看到，dubbo 支持多注册中心，并且支持多协议，一个服务如果支持多协议，那么就都需要暴露。

​		接下来的重点是 doExportUrlsFor1Protocol 方法。

​		![](D:/Book/MyNotes/img/dubbo-doExportUrlsFor1Protocol.png)

​		此时构建出来的 URL 如下

​		![](D:/Book/MyNotes/img/dubbo-doExportUrlsFor1Protocol-result.png)

​		然后就是根据 URL 进行服务暴露了。

​		![](D:/Book/MyNotes/img/dubbo-服务暴露.png)

​		

### 本地暴露

exportLocal 方法是本地暴露，走的是 injvm 协议，可以看到它搞了个新的 URL 修改了协议。

![](img/dubbo-exportLocal.png)

这里的 export 涉及到 SPI 的自定义扩展。

```java
 Exporter<?> exporter = protocol.export(
                    proxyFactory.getInvoker(ref, (Class) interfaceClass, local));
```

Protocol 的 export 方法是实现了 @Adaptive 注解的，因此会生成代理类，然后代理类会根据 invoker 里面的 URL 参数得知具体的协议，然后通过 Dubbo SPI 机制选择对应的实现类进行 export，而这个方法会调用 InjvmProtocol 的 export 方法。

![](img/dubbo-Protocol#export.png)

![](img/dubbo-InjvmProtocol#export.png)

我们再来看看转换得到的 export 到底长什么样子。

![](img/dubbo-exportLocal-export.png)

从图中可以看到，实际上就是具体实现类层层封装，invoker 其实是由 Javassist创建的，具体创建过程详见proxyFactory.getInvoker。

**为什么要封装成 invoker：**其实就是想屏蔽调用的细节，统一暴露出一个可执行体。

**为什么要搞个本地暴露：**因为可能存在同一个 JVM 内部引用自身服务的情况，因此暴露的本地服务在内部调用的时候可以直接消费同一个 JVM 的服务避免了网络间的通信。

汇总图如下：

![](img/dubbo-本地暴露.png)

再来一波时序图

![](img/dubbo-exportLocal-时序图.png)



### 远程暴露

和本地暴露一样，同样需要封装成 invoker。不过这里相对而言比较复杂一些，这里会通过 ` registryURL.addParameterAndEncoded(Constants.EXPORT_KEY, url.toFullString())` 将 URL 拼接成以下样子

> registry://127.0.0.1:2181/com.alibaba.dubbo.registry.RegistryService?application=demo-provider&dubbo=2.0.2&export=dubbo://192.168.1.17:20880/com.alibaba.dubbo.demo.DemoService......

可以看到走 registry 协议，然后参数里又有 export=dubbo://，这个走 dubbo 协议。索引我们可以得知，会先通过 registry 协议找到 RegistryProtocol 进行 export，并且在此方法里还会根据 export 字段得到值，然后执行 DubboProtocol 的 export 方法。 

现在进入 RegistryProtocol#export 方法，看一下整体流程。

![](img/dubbo-RegistryProtocol#export.png)

可以看到，这一步是将上面的 export=dubbo://... 先转换成 exporter，然后获取注册中心的相关配置，如果需要注册则向注册中心注册，并且在 ProviderConsumerRegTable 这个表中记录服务提供者。其实就是往一个 ConcurrentHashMap 中将塞入 invoker，key 就是服务接口全限定名，value 是一个 set，set 里面会存包装过的 invoker 。

![](img/dubbo-providerInvokers.png)

再继续看 doLocalExport 方法内部

![](img/dubbo-doLocalExport.png)

这个方法主要就是根据 URL 上的 Dubbo 协议暴露出 exporter，接下来就看下 DubboProtocol#export 方法。

![](img/dubbo-DubboProtocol#export.png)

可以看到这里的关键其实就是打开 Server ，RPC 肯定需要远程调用，这里我们用的是 NettyServer 来监听服务。

![](img/dubbo-doLocalExport-createServer.png)

总结一下，Dubbo 协议的 export 主要就是根据 URL 构建出 key（例如有分组、接口名、端口等），然后根据 key 和 invoker 关联，关联之后存储到 DubboProtocol 的 exporterMap 中，然后如果是服务初次暴露则会创建监听服务器，默认是 NettyServer，并且会初始化各种 Handler 比如心跳、编解码等。

其实上面的 Protocol 是个代理类，在内部会通过 SPI 机制找到具体的实现类。

![](img/dubbo-protocol$Adaptive.png)

这里可以看到 export 的具体实现

![](img/dubbo-Protocol#export-impl.png)

通过 DUbbo SPI 扫包会把 wrapper 结尾的类缓存起来，然后当加载具体实现类的时候会包装实现类来实现 Dubbo 的 AOP。

DubboProtocol 有两个包装类，分别是 ProtocolFilterWrapper 和 ProtocolListenerWrapper。

对于所有的 Protocol 实现类来说调用链如下：

![](img/dubbo-Protocol实现类调用链.png)

而在 ProtocolFilterWrapper 的 export 里面就会把 invoker 组装上各种 Filter。

![](img/dubbo-ProtocolFilterWrapper#export.png)

我们再来看下 zookeeper 里面现在是怎么样的，关注 dubbo 目录。

![](img/dubbo-zookeeper.png)

两个 service 占用了两个目录，分别有 configurators 和 providers 文件夹，文件夹里面记录的就是 URL 的那一串，值是服务提供者 ip。

Dubbo 服务暴露完整流程图：

![Dubbo 服务暴露完整流程图](img/dubbo-服务暴露-完整流程图.png)