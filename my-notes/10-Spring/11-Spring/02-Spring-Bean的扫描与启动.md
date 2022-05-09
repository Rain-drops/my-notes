![](../img/spring/BeanFactory.png) 

## 一、Bean 的扫描与启动（基于注解的方式）



#### refresh() 核心



## 二、其他

### 1. BeanFactory 与 ApplicationContext

####     A、BeanFactory

​    ...

####     B、ApplicationContext

​    ...

####     C、本质区别

​    加载时机不同

​    **BeanFactory 是懒加载的**，在启动时不会去实例化 Bean，只有从容器中拿 Bean 时才会去实例化

​    **ApplicationContext 是非懒加载的**，在启动时就会将所有的 Bean 实例化，可以通过 lazy-init=true 配置来实现懒加载



### 2. Bean 与 Bean定义 的区别

BeanDefinition：是 Bean 在 spring 中的描述，如 类名、构造方法、参数、属性、是否懒加载等等。有了 BeanDefinition 我们就可以创建 Bean。

Bean：实例对象，根据 BeanDefinition 得到的对象就是 Bean。

## 