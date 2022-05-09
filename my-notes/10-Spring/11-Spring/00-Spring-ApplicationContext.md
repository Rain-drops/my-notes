## ApplicationContext

ApplicationContext 是 Spring 提供的一个高级的 IoC 容器，它除了能够提供 IoC 容器的基本功能，还为用户提供了以下附加服务。

（1）支持信息源，可以实现国际化（实现 MessageSource 接口）。
（2）访问资源（实现 ResourcePatternResolver 接口）。
（3）支持应用事件（实现 ApplicationEventPublisher 接口）    

## AbstractApplicationContext

ApplicationContext 是该模块的核心接口，它的超类是 BeanFactory。

ApplicationContext 实例化后会自动对所有的单实例Bean进行实例化与依赖关系的装配，使之处于待用状态。

## AnnotationConfigApplicationContext

AnnotationConfigApplicationContext 的使用有两种方式，一种是指定配置类文件，另一种是指定扫描路径。

![](D:/Book/MyNotes/my-notes/img/spring/AnnotationConfigApplicationContext.png)

以这个构造函数为切入点。

> 先了解一下 Bean 与 Bean定义 的区别
>
> BeanDefinition：是 Bean 在 spring 中的描述，如 类名、构造方法、参数、属性、是否懒加载等等。有了 BeanDefinition 我们就可以创建 Bean。
>
> Bean：实例对象，根据 BeanDefinition 得到的对象就是 Bean。

![](D:/Book/MyNotes/my-notes/img/spring/AnnotationConfigApplicationContext-Constructor.jpg)

#### AnnotatedBeanDefinitionReader

读取注解的 Bean定义 读取器。是专门处理配置类的，向容器中注入一些核心的 BeanDefinition，主要是后置处理器。

![](D:/Book/MyNotes/my-notes/img/spring/registerAnnotationConfigProcessors.jpg)

##### 扩展：

看下 ConfigurationClassPostProcessor，在 processConfigBeanDefinitions 方法中，其核心逻辑是：

首先从当前 bean 容器中取出全部的 bean 的定义，然后从中挑选出候选集。什么样的类可以是候选集？@Configuration 标注的 bean 定义，或者带有 @Bean 注解的 bean 定义。然后 new 了一个 parser，针对之前的候选集一个一个 parse，然后从 parser 中取到 parse 后的 bean 定义，交给 reader 注册。

![](D:/Book/MyNotes/my-notes/img/spring/processConfigBeanDefinitions-1.jpg)

![](D:/Book/MyNotes/my-notes/img/spring/processConfigBeanDefinitions-2.jpg)

这里看下 ConfigurationClassParser.parse() ，重点关注下 doProcessConfigurationClass 方法

![](D:/Book/MyNotes/my-notes/img/spring/doProcessConfigurationClass.png)

该方法是处理 @Configuration 的核心方法，针对与 @Configuration 标签可能同时出现的其他标签做处理。

包括 @PropertySource，@ComponentScan，@Import 以及类内部的 @Bean。



#### ClassPathBeanDefinitionScanner

![](D:/Book/MyNotes/my-notes/img/spring/ClassPathBeanDefinitionScanner.jpg)

只关心最后一个构造函数的 registerDefaultFilters() 方法。

该方法用于 注册 spring 扫描类过滤器，加了特定注解的类会被扫描到（@Component、@Repository、@Service、@Controller、@ManagedBean、@Named）

#### register()

解析配置类，将配置类转化成 BeanDefinition，设置一些默认属性，然后注册到 Spring（BeanFactory）容器中。

**注意：**此时配置类，并没有被创建出来，只是解析了一番，相当于给 Spring 写了一份声明，后面 Spring 会根据这份声明，来创建和解析配置类。





## ClassPathXmlApplicationContext



