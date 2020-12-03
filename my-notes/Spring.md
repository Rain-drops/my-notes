## 框架

### 1. Bean 生命周期

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

### 2. 代理

​	代理模式：指一个对象 A 通过持有另一个对象 B，可以具有 B 同样的行为模式。为了对外开放协议，B 往往实现了一个接口，A也会去实现接口。但是 B 是“真正”实现类，A 则比较“虚”，他借用了 B 的方法去实现接口的方法。

#### 1. 静态代理：

#### 2. 动态代理：

##### 	JDK 动态代理：

​	利用反射机制实现代理接口的匿名类，在调用具体方法前调用 InvokeHandler 来处理。需要指定一个类加载器，然后生成代理对象实现类的接口或类的类型。

​	**基于接口的。**

##### 	CGLIB：

​	基于 asm 的开源包，对代理对象的 Class 文件加载进来，通过修改其字节码生成的子类来处理。

​	**基于继承父类生成的代理类。**

##### 	区别：

1. JDK 动态代理只能对实现了接口的类生成代理，不能针对类。

 	2. CGLIB 针对类实现代理，主要是对指定的类生成一个子类，覆盖其中的方法。不能对声明为 final 的方法进行代理。

### 3. 反射

在运行时，获取对象的方法和属性。

jvm 通过字节码 class 文件，反编译生成相应的对象。

Proxy：

InvocationHandler：

#### 1. 实现方式

1. Class.forName("类的全路径")

 	2. 类名.class
 	3. 对象名.getClass()

### 4. 类加载机制

加载：这个阶段会在内存中生成一个代表这个类的 java.lang.Class 对象，作为方法区这个类的各种数据的入口。

验证：确保 Class 文件的字节流中包含的信息是否符合当前虚拟机的要求，并且不会危害到虚拟机安全。

准备：为类变量分配内存并设置类变量的初始值

> **注意：**
>
> public static int i = 8080
>
> 该变量，准备阶段过后，初始值是 0，将 i 赋值为 8080 的 put static 指令是程序被编译后，存放于类构造器 <client> 方法中的。
>
> public static **final** int i = 8080 
>
> 在编译阶段会为 i 生成 ConstantValue 属性，在准备阶段，虚拟机会根据 ConstantValue 属性为 i 赋值为 8080。

解析：虚拟机将常量池中的符合引用替换为直接引用的过程。

初始化：执行类构造器 <client> 方法的过程。<client> 方法是由编译器自动收集类中的类变量的赋值操作和静态语句块中的语句合并而成的。

使用：

卸载：

### 5. [ClassLoader](https://blog.csdn.net/jiangmaodong/article/details/105513954) 

双亲委派机制：避免类的重复加载 、安全（防止核心库被篡改）

![class-loader](img\class-loader.png)

### 6. BeanPostProcessor

​	前置处理器，修改 bean 内容。

### 7. IOC

### 8. AOP

#### 1. 作用

简单的说就是将跟业务无关的代码，封装起来，减少系统重复代码，降低模块间的耦合。

#### 2. 基本概念：

> - (1) Aspect(切面)：通常是一个类，里面可以定义切入点和通知
> - (2) JointPoint(连接点)：程序执行过程中明确的点，一般是方法的调用
> - (3) Advice(通知)：AOP 在特定的切入点上执行的增强处理，有before, after, afterReturning, afterThrowing, around
> - (4) Pointcut(切入点)：就是带有通知的连接点，在程序中主要体现为书写切入点表达式
> - (5) AOP 代理：AOP 框架创建的对象，代理就是目标对象的加强。Spring 中的 AOP 代理可以使 JDK 动态代理，也可以是 CGLIB 代理，前者基于接口，后者基于子类

**通知方法:**

1. 前置通知：在我们执行目标方法之前运行(**@Before**)
2. 后置通知：在我们目标方法运行结束之后 ,不管有没有异常**(@After)**
3. 返回通知：在我们的目标方法正常返回值后运行**(@AfterReturning)**
4. 异常通知：在我们的目标方法出现异常后运行**(@AfterThrowing)**
5. 环绕通知：动态代理, 需要手动执行joinPoint.procced()(其实就是执行我们的目标方法执行之前相当于前置通知, 执行之后就相当于我们后置通知**(@Around)**

#### 3. 源码解析

​	

### 9. 循环依赖

### 10. 三级缓存

​	**无法解决构造器注入的循环依赖**：加入  singletonFactories 三级缓存的前提是执行了构造器。

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    /** 一级缓存 Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
    /** 二级缓存 Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
	/** 三级缓存 Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    ......
}
```

​	**Spring 启动过程：**

1. 加载配置文件

 	2. 解析配置文件转化为 beanDefination，获取 Bean 的所有属性，依赖及初始化用到的各类处理器等
 	3. 创建 BeanFactory 并初始化所有**单例** Bean
 	4. 注册所有的单例 Bean 并返回可用的容器，一般为扩展的 ApplicationContext

#### 一级缓存：

​	在步骤 3 中，所有**单例**的 Bean **初始化完成后**会存放在 Map(singletonObjects) 中，beanName 为 key，单例 Bean 为 value。

​	**从该缓存中去除的 Bean 可以直接使用**。

#### 二级缓存：

​	提前暴露的对象缓存，存放原始的 Bean 对象（尚未填充属性）

#### 三级缓存：

​	单例对象工厂缓存，存放 Bean 工厂对象。**代理**

### 11. session 共享

### 12. 事务

### 13. Spring boot Bean装配失败

​	**Bean 装配顺序：**

1. 同是 @ComponentScan 扫描注册的 Bean，按 Class 文件名顺序，排在前面的先注册，所以先实例化，例如 ATest 类实例化先于 BTest 类。

 	2. @Conguration 配置类实例化先于其内部类定义的 @Bean 方法执行实例化。

**@Import 注解用来导入配置类或者一些需要前置加载的类。**

