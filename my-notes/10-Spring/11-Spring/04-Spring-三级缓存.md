https://itsoku.blog.csdn.net/article/details/113977261

## 一、简述

1、无法解决构造器注入的循环依赖：加入  singletonFactories 三级缓存的前提是执行了构造器。

2、

## 二、Spring 中 setter 循环依赖注入流程

```java
public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
    /** 一级缓存 Cache of singleton objects: bean name to bean instance. */
	private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
	/** 三级缓存 Cache of singleton factories: bean name to ObjectFactory. */
	private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
    /** 二级缓存 Cache of early singleton objects: bean name to bean instance. */
	private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
    ......
}
```

### 1、Spring 启动过程大致如下：

1. 加载配置文件
  2. 解析配置文件转化为 beanDefination，获取 Bean 的所有属性，依赖及初始化用到的各类处理器等
  3. 创建 BeanFactory 并初始化所有**单例** Bean
  4. 注册所有的单例 Bean 并返回可用的容器，一般为扩展的 ApplicationContext

> spring 在对象 getBean() 时，先从一级缓存拿，拿到直接返回，拿不到就去二级缓存拿，再拿不到就去三级缓存拿 ObjectFactory，拿到了就调用 getObject创建对象，拿不到就返回 null
>
> ```java
> protected Object getSingleton(String beanName, boolean allowEarlyReference) { 
>     //首先检查一级缓存中是否存在 
>     Object singletonObject = this.singletonObjects.get(beanName); 
>     /** 
>      * 如果一级缓存中不存在代表当前 Bean 还未被创建或者正在创建中 
>      * 检查当前 Bean 是否正处于正在创建的状态中(当Bean创建时会将Bean名称存放到 singletonsCurrentlyInCreation 集合中) 
>      */ 
>     if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) { 
>         synchronized (this.singletonObjects) { 
>             //检查二级缓存中是否存在 
>             singletonObject = this.earlySingletonObjects.get(beanName); 
>             /** 
>              * @@如果二级缓存中不存在 并且 允许使用早期依赖 
>              * allowEarlyReference ： 它的含义是是否允许早期依赖 
>              * @@那么什么是早期依赖？ 
>              * 就是当Bean还未成为成熟的Bean时就提前使用它，在实例化流程图中我们看到在添加缓存前刚刚实例化Bean但是还未依赖注入时的状态 
>              */ 
>          	if (singletonObject == null && allowEarlyReference) { 
>                 // 获取三级缓存中的 Bean 
>                 ObjectFactory ObjectFactory singletonFactory = this.singletonFactories.get(beanName); 
>                 // 如果 Bean 对应的 ObjectFactory 存在 
>                 if (singletonFactory != null) { 
>                     // 使用 getObject 方法获取到 Bean 的实例 
>                     singletonObject = singletonFactory.getObject(); 
>                     // 将 bean 从三级缓存提升至二级缓存 
>                     this.earlySingletonObjects.put(beanName, singletonObject); 
>                     this.singletonFactories.remove(beanName); }
>             } 
>         } 
>     } 
>     return singletonObject;
> }
> ```
>
> 

> 创建 Bean
>
> ```java
> protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args) 
>     throws BeanCreationException {   
>     /**    
>      * 我们一开始通过 getSingleton() 方法中获取三级缓存中存放的Bean，这里就是向三级缓存中添加 bean 的地方   
>      * 流程:    
>      * 1.检查当前 bean 是否为单例模式，并且是否允许循环引用[讲解1],并且当前是否正在创建中(在getSingleton方法中添加的)    
>      * 2.如果允许提前曝光[讲解2]，addSingletonFactory() 方法向缓存中添加当前 bean 的 ObjectFactory    
>      *    [讲解1]：当前 Bean 如果不允许循环引用(循环依赖也就是被依赖)，则这里就不会提前曝光，
>      *    对应的 ObjectFactory，则当发生循环依赖时会抛出 BeanCreationException 异常   
>      *    [讲解2]：提前曝光的含义就是说当 bean 还未创建完毕时就先将创建中状态的bean放到指定缓存中，为循环依赖提供支持   
>      */ 
> 	boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences 
>                                       && isSingletonCurrentlyInCreation(beanName));   
>     // 需要提前曝光   
>     if (earlySingletonExposure) {      
>         // 向缓存(三级缓存)中添加当前 bean 的 ObjectFactory      
>         addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));   
>     }   
> 	// 初始化 Bean 实例阶段      
>     Object exposedObject = bean;   
>     try {
> 		// 依赖注入这时会递归调用getBean      
>         populateBean(beanName, mbd, instanceWrapper);      
>         // 调用初始化方法，如：init-method      
>         exposedObject = initializeBean(beanName, exposedObject, mbd);   
>     }catch (Throwable ex) {
>         // ...
>     }    
>     /**    
>      * 当允许提前曝光时进入判断    
>      * @这里做了什么？    
>      * 1.检查当前bean是否经历了一场循环依赖    
>      *   - 通过 getSingleton(beanName,false) 获取缓存中的 bean，传入 false 代表不获取三级缓存中的bean    
>      *   - 为什么说 检查当前bean是否经历了一场循环依赖呢？ 因为上述说了传入 false 代表不获取三级缓存的    
>      *   - 那么什么情况下才会存在与一级缓存和二级缓存呢？答案就是循环依赖后 [解释1] 和bean实例化完成后    
>      *   - 所以如果 getSingleton 返回的 bean 不为空，则这个bean就是刚刚经历了循环依赖
>      * 2.检查提前曝光的bean和当前的Bean是否一致    
>      *   - 下面有个判断 if (exposedObject == bean) ，这个判断从缓存中获取的bean 和 经历过初始化后的 bean    
>      *   - 是否一致，可能我们有点晕，这里解释一下，缓存从的bean是什么时候存进去的？是在 addSingletonFactory 方法(649行)    
>      *   - 然后这里存进去的 bean 只是提前曝光的 bean，还没有依赖注入和初始化，但是在依赖注入和初始化时都是可能直接改变    
>      *   - 当前 bean 的实例的，这意味着什么？意味着经历了依赖注入和初始化的bean很可能和缓存中的bean就已经完全不是一个 bean了    
>      *   下面讲解当一致或不一致时的逻辑：    
>      * 2.1 一致：    
>      *  不是很理解，直接赋值，可是经历了各种 BeanPostProsser 或者依赖注入和初始化后不是就不一样了吗    
>      * 2.2 不一致:    
>      *  看下方对于 else if 代码块的解释    
>      *    
>      * @[解释1]    
>      * 当循环依赖时，A依赖着B，B依赖着A，实例化A首先将A放到三级缓存中然后发现依赖着B，然后去实例化B，发现依赖着A    
>      * 发现A在三级缓存，然后获取三级缓存中的bean并且将A从三级缓存中提升到二级缓存中，实例化B完成，接着实例化A也完成。    
>      * @通俗讲解    
>      * 假设我们业务上对某种数据加了缓存，假设 i 在缓存中存的值为1，当我在数据库中把 i 的值改成 2 时，缓存中的 i 还没有被改变还是 1    
>      * 这时的数据已经和我们的真实数据偏离了，不一致了，这时有两种解决方式：
>      * 1.服务器检查到数据不一致抛出异常。(也就是进入else if 代码块)    
>      * 2.直接使用原始值也就是1(也就是将 allowRawInjectionDespiteWrapping 改成 true),当然这两种方式明显不是我们正常数据库的操作，只是    
>      * 为了说明当前的这个例子而已。    
>      */   
>      if (earlySingletonExposure) {      
>          //获取缓存中(除三级缓存) beanName 对应的 bean      
>          Object earlySingletonReference = getSingleton(beanName, false);      
>          //当经历了一场循环依赖后 earlySingletonReference 就不会为空      
>          if (earlySingletonReference != null) {
>              //如果 exposedObject 没有在初始化方法中被改变，也就是没有被增强         
>              if (exposedObject == bean) {            
>                  //直接赋值? 可是经历了各种 BeanPostProsser 或者依赖注入和初始化后不是就不一样了吗            
>                  exposedObject = earlySingletonReference;         
>          	 }
>              /**
>               * 走到 else if  时说明 当前 Bean 被 BeanPostProessor 增强了          
>               * 判断的条件为：          
>               * 1.如果允许使用被增强的          
>               * 2.检查是否存在依赖当前bean的bean          
>               *          
>               * 如果存在依赖的bean已经被实例化完成的，如果存在则抛出异常          
>               * 为什么抛出异常呢？          
>               * 因为依赖当前bean 的bean 已经在内部注入了当前bean的旧版本，但是通过初始化方法后这个bean的版本已经变成新的了          
>               * 旧的哪个已经不适用了，所以抛出异常        
>               */         
>              else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {            
>                  String[] dependentBeans = getDependentBeans(beanName);            
>                  Set actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);            
>                  for (String dependentBean : dependentBeans) {               
>                      if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {                  
>                          actualDependentBeans.add(dependentBean);               
>                      }
>                  }
>                  if (!actualDependentBeans.isEmpty()) {               
>                     throw new BeanCurrentlyInCreationException(beanName, 
>                       "Bean with name '" + beanName + "' has been injected into other beans [" 
>                        + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) 
>                        + "] in its raw version as part of a circular reference, but has eventually been " 
>                        + "wrapped. This means that said other beans do not use the final version of the " 
>                        + "bean. This is often the result of over-eager type matching - consider using " 
>                        + "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");            
>                 }         
>             } 
>         }   
>     }   
>     try {      
> 		/**       
>          * Register bean as disposable.       
>          * 注册 Bean 的销毁方法拓展      
>          */      
>          registerDisposableBeanIfNecessary(beanName, bean, mbd);   
> 	} catch (BeanDefinitionValidationException ex) {      
> 	 	 throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);   
>   	}   
>     return exposedObject;
> }
> ```
>
> 

#### 1.1 三级缓存：

存储单例 bean 的 ObjectFactory，提前暴露的一个单例工厂，二级缓存中存储的就是从这个工厂中获取到的对象	

> ​		单例对象工厂缓存，存放 Bean 工厂对象。是函数接口，函数回调创建代理，主要解决 AOP。在没有循环依赖的环境下保证bean初始化完成之后才生成代理，而不是实例化之后就生成代理，保证了bean的生命周期。
>
> ​		如果创建的 Bean 是有代理的，那么注入的就应该是代理 Bean，而不是原始的 Bean。但是 Spring 一开始并不知道 Bean 是否会有循环依赖，通常情况下（没有循环依赖的情况下），Spring 都会在完成填充属性，并且执行完初始化方法之后再为其创建代理。但是，如果出现了循环依赖的话，Spring 就不得不为其提前创建代理对象，否则注入的就是一个原始对象，而不是代理对象。因此，这里就涉及到应该在哪里提前创建代理对象？
>
> ​		Spring 的做法就是在 ObjectFactory 中去提前创建代理对象。它会执行 getObject() 方法来获取到 Bean。
>
> **什么时候 bean 被放入 3 级缓存：**早期的 bean 被放入 3 级缓存

#### 1.2 二级缓存：

存储早期的 Bean，也就是说在这个 Map 里的 Bean 不是完整的，是还未进行属性注入及初始化的对象	

> 提前暴露的对象缓存，存放原始的 Bean 对象（尚未填充属性）
>
> **什么时候 bean 会被放入 2 级缓存：**当 beanX 还在创建的过程中，此时被加入当前 beanName 创建列表了，但是这个时候 bean 并没有被创建完毕（bean 被丢到一级缓存才算创建完毕），此时 bean 还是个半成品，这个时候其他 bean 需要用到 beanX，此时会从三级缓存中获取到 beanX，beanX 会从三级缓存中丢到 2 级缓存中。

#### 1.3 一级缓存：

存储的是所有创建好了的单例 Bean

> 在步骤 3 中，所有**单例**的 Bean **初始化完成后**会存放在 Map(singletonObjects) 中，beanName 为 key，单例 Bean 为 value。
>
> **什么时候 bean 会被放入 1 级缓存：**bean 实例化完毕，初始化完毕，属性注入完毕，bean 完全组装完毕之后，才会被丢到 1 级缓存
>
> 从该缓存中取出的 Bean 可以直接使用。

### 2、A、B 类 setter 循环依赖的创建过程

> 1、getSingleton("a", true) 获取 a：会依次从 3 个级别的缓存中找 a，此时 3 个级别的缓存中都没有 a
>
> 2、将 a 丢到正在创建的 beanName 列表中（Set<String> singletonsCurrentlyInCreation）
>
> 3、实例化 a：A a = new A();这个时候 a 对象是早期的 a，属于半成品
>
> 4、将早期的 a 丢到三级缓存中（Map<String, ObjectFactory<?> > singletonFactories）
>
> 5、调用 populateBean 方法，注入依赖的对象，发现 setB 需要注入 b
>
> 6、调用 getSingleton("b", true) 获取 b：会依次从 3 个级别的缓存中找 a，此时 3 个级别的缓存中都没有 b
>
> 7、将 b 丢到正在创建的 beanName 列表中
>
> 8、实例化 b：B b = new B();这个时候 b 对象是早期的 b，属于半成品
>
> 9、将早期的 b 丢到三级缓存中（Map<String, ObjectFactory<?> > singletonFactories）
>
> 10、调用 populateBean 方法，注入依赖的对象，发现 setA 需要注入 a
>
> 11、调用 getSingleton("a", true) 获取 a：此时 a 会从第 3 级缓存中被移到第 2 级缓存，然后将其返回给 b 使用，此时 a 是个半成品（属性还未填充完毕）
>
> 12、b 通过 setA 将 11 中获取的 a 注入到 b 中
>
> 13、b 被创建完毕，此时 b 会从第 3 级缓存中被移除，然后被丢到 1 级缓存
>
> 14、b 返回给 a，然后 b 被通过 A 类中的 setB 注入给 a
>
> 15、a 的 populateBean 执行完毕，即：完成属性填充，到此时 a 已经注入到 b 中了
>
> 16、调用a= initializeBean("a", a, mbd)对 a 进行处理，这个内部可能对 a 进行改变，有可能导致 a 和原始的 a 不是同一个对象了
>
> 17、调用getSingleton("a", false)获取 a，注意这个时候第二个参数是 false，这个参数为 false 的时候，只会从前 2 级缓存中尝试获取 a，而 a 在步骤 11 中已经被丢到了第 2 级缓存中，所以此时这个可以获取到 a，这个 a 已经被注入给 b 了
>
> 18、此时判断注入给 b 的 a 和通过initializeBean方法产生的 a 是否是同一个 a，不是同一个，则弹出异常



**从上面的过程中我们可以得到一个非常非常重要的结论**

​		当某个 bean 进入到 2 级缓存的时候，说明这个 bean 的早期对象被其他 bean 注入了，也就是说，这个 bean 还是半成品，还未完全创建好的时候，已经被别人拿去使用了，所以必须要有 3 级缓存，2 级缓存中存放的是早期的被别人使用的对象，如果没有 2 级缓存，是无法判断这个对象在创建的过程中，是否被别人拿去使用了。

​		3 级缓存是为了解决一个非常重要的问题：早期被别人拿去使用的 bean 和最终成型的 bean 是否是一个 bean，如果不是同一个，则会产生异常，所以以后面试的时候被问到为什么需要用到 3 级缓存的时候，你只需要这么回答就可以了：三级缓存是为了判断循环依赖的时候，早期暴露出去已经被别人使用的 bean 和最终的 bean 是否是同一个 bean，如果不是同一个则弹出异常，如果早期的对象没有被其他 bean 使用，而后期被修改了，不会产生异常，如果没有三级缓存，是无法判断是否有循环依赖，且早期的 bean 被循环依赖中的 bean 使用了。



## 三、若只使用 2 级缓存会产生什么后果？