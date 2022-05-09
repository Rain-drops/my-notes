### 概述

Spring 的出现： 简化 java 开发。

> 基于**依赖注入**和**接口**实现松耦合
>
> 引入 AOP，面向切面编程
>
> 引入模板，减少了样板代码
>
> 基于 POJO，减少代码侵略性

核心技术

> **IOC：**控制反转。Spring 通过 **IOC容器** 来进行 Bean 的管理
>
> **DI：**依赖注入，属性赋值
>
> **AOP：**面向切面编程

BeanFactory：

> 利用 BeanFactory 来规范子类，负责生产和管理 Bean。实现了 BeanFactory 的子类就必须拥有 创建 Bean（ getBean() ）的能力。
>
> BeanFactory 通过 AbstractBeanFactory 来进行 Bean 的创建。
>
> 仅提供最基本的依赖注入支持

FactoryBean

> 为 IoC 容器的中 Bean 的实现提供了更加灵活的方式，可以通过实现该接口定制实例化 Bean 的逻辑

BeanDefinition：

> Bean 定义：Bean 的描述信息（beanClassName、scope、isLazy等等），定义一个类以什么样的方式来创建 bean。

BeanWrapper

> 是对 Bean 的包装，用于封装创建后的对象实例
>
> 通过 BeanWrapper，spring ioc 容器可以用统一的方式来访问 bean 的属性

ApplicationContext：

> 由 BeanFactory 派生而来，提供了更多面向实际应用的功能
>
> 1、国际化(getMesage)
>
> 2、资源管理：可以直接读取一个文件的内容( getResource )
>
> 3、加入 web 框架中(加入一个 servlet 或监听器)
>
> 4、事件处理

BeanDefinitionReader

> 读取 Spring 配置文件中的内容，将其转换为 IoC 容器内部的数据结构 BeanDefinition。

ApplicationContextAware

> 通过解耦方式获得 IoC 容器，从而将 IoC 容器注目标类中

BeanPostProcessor

> 后置处理器，作用是在 Bean 对象在实例化和依赖注入完毕后，在显示调用初始化方法的前后添加我们自己的逻辑。

