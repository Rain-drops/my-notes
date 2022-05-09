### 1、简单工厂模式

又叫做静态工厂方法（StaticFactory Method）模式，但不属于23种GOF设计模式之一。 

简单工厂模式的实质是由一个工厂类根据传入的参数，动态决定应该创建哪一个产品类。 

spring 中的 BeanFactory 就是简单工厂模式的体现，根据传入一个唯一的标识来获得 bean 对象，但是否是在传入参数后创建还是传入参数前创建这个要根据具体情况来定。

### **2、工厂方法模式（Factory Method）**

通常由应用程序直接使用 new 创建新的对象，为了将对象的创建和使用相分离，采用工厂模式，即应用程序将对象的创建及初始化职责交给工厂对象。

一般情况下，应用程序有自己的工厂对象来创建bean。如果将应用程序自己的工厂对象交给Spring管理，那么Spring管理的就不是普通的bean，而是工厂Bean。

Spring中的FactoryBean就是典型的工厂方法模式：

### **3、单例模式（Singleton）**

保证一个类仅有一个实例，并提供一个访问它的全局访问点。 
spring中的单例模式完成了后半句话，即提供了全局的访问点 BeanFactory。但没有从构造器级别去控制单例，这是因为spring管理的是是任意的java对象。 

核心提示点：Spring下默认的 bean 均为 singleton，可以通过singleton=“true|false” 或者 scope="?"来指定。

### **4、适配器模式（Adapter）**

**把一个类的接口变换成客户端所期待的另一种接口， Adapter模式使原本因接口不匹配（或者不兼容）而无法在一起工作的两个类能够在一起工作**。

在 Spring 的 Aop 中，使用的 Advice（通知）来增强被代理类的功能。Spring 实现 AOP 功能的原理就使用代理模式（1、JDK动态代理。2、CGLib字节码生成技术代理）对类进行方法级别的切面增强，即，生成被代理类的代理类， 并在代理类的方法前，设置拦截器，通过执行拦截器重的内容增强了代理方法的功能，实现的面向切面编程。

```java
public interface AdvisorAdapter {}
class AfterReturningAdviceAdapter implements AdvisorAdapter, Serializable {}
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {}
class ThrowsAdviceAdapter implements AdvisorAdapter, Serializable {}
```

由于 Advisor 链需要的是 MethodInterceptor（拦截器）对象，所以每一个 Advisor 中的 Advice 都要适配成对应的 MethodInterceptor 对象。

![](../img/spring/设计模式/Adapter.png)

```java
/** 对用户发来的请求来进行处理，根据请求去定位请求的具体处理方法 **/
/** HandlerAdapter 定义了如何处理请求的策略，通过请求url、请求 Method 和处理器的 requestMapping 定义，最终确定使用处理类的哪个方法来处理请求，并检查处理类相应处理方法的参数以及相关的 Annotation 配置，确定如何转换需要的参数传入调用方法，并最终调用返回 ModelAndView **/
public interface HandlerAdapter {}
public class HttpRequestHandlerAdapter implements HandlerAdapter {}
public class SimpleServletHandlerAdapter implements HandlerAdapter {}
```



### **5、包装器模式（Wrapper）/ 装饰模式（Decorator）**

​		在我们的项目中遇到这样一个问题：我们的项目需要连接多个数据库，而且不同的客户在每次访问中根据需要会去访问不同的数据库。我们以往在 spring 和 hibernate 框架中总是配置一个数据源，因而 sessionFactory 的 dataSource 属性总是指向这个数据源并且恒定不变，所有 DAO 在使用 sessionFactory 的时候都是通过这个数据源访问数据库。

​		但是现在，由于项目的需要，我们的 DAO 在访问 sessionFactory 的时候都不得不在多个数据源中不断切换，问题就出现了：如何让 sessionFactory 在执行数据持久化的时候，根据客户的需求能够动态切换不同的数据源？我们能不能在 spring 的框架下通过少量修改得到解决？是否有什么设计模式可以利用呢？ 

​		首先想到在 spring 的 applicationContext 中配置所有的 dataSource。这些 dataSource 可能是各种不同类型的，比如不同的数据库：Oracle、SQL Server、MySQL 等，也可能是不同的数据源：比如 apache 提供的 org.apache.commons.dbcp.BasicDataSource、spring 提供的org.springframework.jndi.JndiObjectFactoryBean 等。然后 sessionFactory 根据客户的每次请求，将 dataSource 属性设置成不同的数据源，以到达切换数据源的目的。

​		spring 中用到的包装器模式在类名上有两种表现：一种是类名中含有 Wrapper，另一种是类名中含有 Decorator。基本上都是动态地给一个对象添加一些额外的职责。



```java
/** 这个类主要是用来处理事务缓存的 **/
/** TransactionAwareCacheDecorator 就是对 Cache 的一个包装 **/
public class TransactionAwareCacheDecorator implements Cache {}
```

```java
/**  **/
public class HttpHeadResponseDecorator extends ServerHttpResponseDecorator {}
```

```java
/** 流 **/
public abstract class InputStream implements Closeable {}
public class FileInputStream extends InputStream{}
public class FilterInputStream extends InputStream{}
```



### **6、代理模式（Proxy）**

​		为其他对象提供一种代理以控制对这个对象的访问。 从结构上来看和 Decorator 模式类似，但 Proxy 是控制，更像是一种对功能的限制，而 Decorator 是增加职责。 
​		Spring 的 Proxy 模式在 aop 中有体现，比如 JdkDynamicAopProxy 和 Cglib2AopProxy。 

```java
public interface AopProxy {}
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {}
class CglibAopProxy implements AopProxy, Serializable {}
```



### **7、观察者模式（[Observer](https://www.cnblogs.com/jmcui/p/11054756.html)）**

定义对象间的一种一对多的依赖关系，当一个对象的状态发生改变时，所有依赖于它的对象都得到通知并被自动更新。

> **Spring 中观察者模式的四个角色：**
>
> **1. 事件（ApplicationEvent）**
>
> ApplicationEvent 是所有事件对象的父类。ApplicationEvent 继承自 jdk 的 EventObject, 所有的事件都需要继承 ApplicationEvent, 并且通过 source 得到事件源。
>
> 下列描述了Spring提供的内置事件：
>
> - ContextRefreshedEvent：事件发布在 ApplicationContext 初始化或刷新时（例如：通过在 ConfigurableApplicationContext 接口使用refresh()方法）。这里,“初始化”意味着所有 bean 加载，post-processor bean 被检测到并且激活,单例预先实例化，ApplicationContext 对象可以使用了。只要上下文没有关闭,可以触发多次刷新, ApplicationContext 提供了一种可选择的支持这种“热”刷新。例如：XmlWebApplicationContext 支持热刷新,但 GenericApplicationContext 并非如此。具体是在 AbstractApplicationContext 的 finishRefresh() 方法中。
> - ContextStartedEvent：事件发布在 ApplicationContext 开始使用 ConfigurableApplicationContext 接口 start() 方法。这里,“开始”意味着所有生命周期 bean 接收到一个明确的起始信号。通常,这个信号用于明确停止后重新启动,但它也可以用于启动组件没有被配置为自动运行（例如：组件还没有开始初始化）。
> - ContextStoppedEvent：事件发布在 ApplicationContext 停止时通过使用 ConfigurableApplicationContext 接口上的 stop() 方法。在这里,“停止”意味着所有生命周期bean接收一个显式的停止信号。停止上下文可以通过重新调用start()方法。
> - ContextClosedEvent：事件发布在 ApplicationContext 关闭时通过关闭 ConfigurableApplicationContext 接口()方法。这里,“封闭”意味着所有单例 bean 被摧毁。一个封闭的环境达到生命的终结。它不能刷新或重启。
> - RequestHandledEvent：一个特定的web事件告诉所有能处理HTTP请求的bean 。这个事件是在请求完成后发布的。这个事件只适用于使用 Spring 的 DispatcherServlet 的web应用程序。
>
> **2. 事件监听（ApplicationListener）**
>
> ApplicationListener 事件监听器，也就是观察者。继承自 jdk 的 EventListener，该类中只有一个方法 onApplicationEvent。当监听的事件发生后该方法会被执行。
>
> **3. 事件发布（ApplicationContext）**
>
> ApplicationContext 是 Spring 中的核心容器，在事件监听中 ApplicationContext 可以作为事件的发布者，也就是事件源。因为 ApplicationContext 继承自 ApplicationEventPublisher。在 ApplicationEventPublisher 中定义了事件发布的方法 — publishEvent(Object event)
>
> **4. 事件管理（ApplicationEventMulticaster）**
>
> ApplicationEventMulticaster 用于事件监听器的注册和事件的广播。监听器的注册就是通过它来实现的，它的作用是把 Applicationcontext 发布的 Event 广播给它的监听器列表。

Spring 中 Observer 模式常用的地方是 listener 的实现。如 ApplicationListener。

```java
public abstract class ApplicationEvent extends EventObject {}
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {
    void onApplicationEvent(E event);
}
```

### **8、策略模式（Strategy）**

​		定义一系列的算法，把它们一个个封装起来，并且使它们可相互替换。本模式使得算法可独立于使用它的客户而变化。 

​		Spring 中在实例化对象的时候用到 Strategy 模式。AOP 动态代理的两种实现也是一种策略模式。

```java
/** 负责根据BeanDefinition对象创建一个Bean实例 **/
/** 仅负责实例化Bean的操作，相当于执行Java语言中new的功能，它并不会参与Bean属性的设置工作 **/
public interface InstantiationStrategy {}
public class SimpleInstantiationStrategy implements InstantiationStrategy {}
```



### **9、模板方法模式（Template Method）**

​		它在一个抽象类中公开定义了执行它的方法的模板，它的子类可以按需重写方法实现，但调用将以抽象类中定义的方式进行。简单来说，模板方法模式定义一个操作中的算法的骨架，将一些步骤延迟到子类中，使得子类可以不改变一个算法的结构，就能重新定义该算法的某些特定步骤。

​		Template Method 使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。Template Method 模式一般是需要继承的。

​		Spring 中的 JdbcTemplate，在用这个类时并不想去继承这个类，因为这个类的方法太多，但是我们还是想用到JdbcTemplate已有的稳定的、公用的数据库连接，那么我们怎么办呢？我们可以把变化的东西抽出来作为一个参数传入 JdbcTemplate 的方法中。但是变化的东西是一段代码，而且这段代码会用到 JdbcTemplate 中的变量。怎么办？那我们就用回调对象吧。

​		在这个回调对象中定义一个操纵 JdbcTemplate 中变量的方法，我们去实现这个方法，就把变化的东西集中到这里了。然后我们再传入这个回调对象到 JdbcTemplate，从而完成了调用。这可能是 Template Method 不需要继承的另一种实现方式。 

以下是一个具体的例子： JdbcTemplate 中的 execute 方法 

![](../img/spring/设计模式/TemplateMethod.png)

