### 1. 简述

Spring 的最高级抽象是 BeanFactory 接口，他是工厂模式的实现，允许通过名称创建和检索对象。

BeanFactory 底层支持两个对象模型

> （1）单例模型：提供了具有特定名称的全局共享实例对象，可以在查询时对其进行检索。 Singleton 是默认的、也是最常用的单例模型 。
>
> （2）原型模型：确保每次检索都会创建单独的实例对象。在每个用户都需要自己的对象时，采用原型模型。  

BeanFactory 实例化后并不会自动实例化 Bean，只有当 Bean 被使用时， BeanFactory 才会对该 Bean 进行实例化与依赖关系的装配 。



### 2. BeanFactory

BeanFactory 只对 IoC 容器的基本行为做了定义，根本不关心 Bean 是如何定义及怎样加载的。要知道工厂是如何产生对象的，我们需要看具体的 IoC 容器实现，Spring 提供了许多 IoC 容器实现，比如 GenericApplicationContext、ClasspathXmlApplicationContext等 。

```java
public interface BeanFactory {
    // 对 FactoryBean 的转义定义，因为如果使用 Bean 的名称检索 FactoryBean 得到的对象是工厂生成的对象。
    // 如果需要得到工厂本身，需要转义。
    String FACTORY_BEAN_PREFIX = "&";
	// 根据 Bean 的名字，获取在 IOC 容器中得到的 Bean 的实例
    Object getBean(String var1) throws BeansException;
	// 根据 Bean 的名字和 Class 类型，获取 Bean 的实例，增加了类型安全验证机制
    <T> T getBean(String var1, Class<T> var2) throws BeansException;
    Object getBean(String var1, Object... var2) throws BeansException;
    <T> T getBean(Class<T> var1) throws BeansException;
    <T> T getBean(Class<T> var1, Object... var2) throws BeansException;
	
    <T> ObjectProvider<T> getBeanProvider(Class<T> var1);
    <T> ObjectProvider<T> getBeanProvider(ResolvableType var1);
	// 提供对 Bean 的检索，看看在 IOC 容器中是否有这个名字的 Bean
    boolean containsBean(String var1);
	// 判断这个 Bean 是不是单例
    boolean isSingleton(String var1) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String var1) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String var1, ResolvableType var2) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String var1, Class<?> var2) throws NoSuchBeanDefinitionException;
	// 得到 Bean 实例的 Class 类型
    @Nullable
    Class<?> getType(String var1) throws NoSuchBeanDefinitionException;
	// 得到 Bean 的别名，如果根据别名检索，那么其原名也会被检索出来
    String[] getAliases(String var1);
}
```

AbstractBeanFactory



DefaultListableBeanFactory

