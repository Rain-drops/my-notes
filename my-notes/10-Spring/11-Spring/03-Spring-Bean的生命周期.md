### 一、Bean 生命周期

Bean 的一次加工、属性设置、二次加工

> 1. 实例化(new)
> 2. 对实例化的Bean进行配置，IOC 依赖注入
> 3. 如果实现了 BeanNameAware 接口，会调用 setBeanName(String)方法
> 4. 如果实现了 BeanFactoryAware 接口，会调用 setBeanFactory(BeanFactory)
> 5. 如果实现了 ApplicationContextAware 接口，会调用 setApplicationContext(ApplicationContext)
> 6. 如果实现了 BeanPostProcessor 接口，会调用 postProcessBeforeInitialization(Object obj, String s)方法，修改 Bean 内容
> 7. init-method
> 8. 如果实现了 BeanPostProcessor 接口，会调用 postProcessAfterInitialization(Object obj, String s)方法，修改 Bean 内容
> 9. destory
> 10. destory-method

### 



### 二、源码解析



### 三、面试题

1、Spring 中的 Bean 是线程安全的吗

2、Spring 中 Bean 的作用域

3、Spring 中 Bean 的生命周期