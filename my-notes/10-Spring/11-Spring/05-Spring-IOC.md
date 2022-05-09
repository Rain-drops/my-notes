## 一、Spring 核心容器类

### 1、BeanFactory

​		BeanFactory 只对 IoC 容器的基本行为做了定义，根本不关心 Bean 是如何定义及怎样加载的。要知道工厂是如何产生对象的，我们需要看具体的 IoC 容器实现，Spring 提供了许多 IoC 容器实现，比如 GenericApplicationContext、ClasspathXmlApplicationContext等 。

​		BeanFactory 实例化后并不会自动实例化 Bean，只有当 Bean 被使用时， BeanFactory 才会对该 Bean 进行实例化与依赖关系的装配 。

### 2、BeanDefinition

​		Spring IoC容器管理我们定义的各种Bean对象及其相互关系， Bean 对象在 Spring 实现中是以 BeanDefinition 来描述的，其继承体系如下图所示。

### 3、BeanDefinitionReader 

​		Bean 的解析过程非常复杂，功能被分得很细，因为这里需要被扩展的地方很多，必须保证足够的灵活性，以应对可能的变化。 Bean 的解析主要就是对Spring 配置文件的解析。这个解析过程主要通过 BeanDefinitionReader 来完成。

















  

IOC 的启动流程

