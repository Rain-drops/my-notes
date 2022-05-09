# [RequestMapping原理分析和RequestMappingHandlerMapping](https://www.cnblogs.com/grasp/p/11100124.html)

源码版本spring-webmvc-4.3.7.RELEASE

使用Spring MVC的同学一般都会以下方式定义请求地址:

```
@Controller
@RequestMapping("/test")
public class TestController {

    @RequestMapping(value = {"/show"})
    public String testTest() {
        return "/jsp/index";
    }
}
```

@Controller注解用来把一个类定义为Controller。

@RequestMapping注解用来把web请求映射到相应的处理函数。

@Controller和@RequestMapping结合起来完成了Spring MVC请求的派发流程。

为什么两个简单的注解就能完成这么复杂的功能呢？又和<context:component-scan base-package="xx.xx.xx"/>的位置有什么关系呢？

## @RequestMapping流程分析

@RequestMapping流程可以分为下面6步：

- 1.注册RequestMappingHandlerMapping bean 。
- 2.实例化RequestMappingHandlerMapping bean。
- 3.获取RequestMappingHandlerMapping bean实例。
- 4.接收requst请求。
- 5.在RequestMappingHandlerMapping实例中查找对应的handler。
- 6.handler处理请求。

为什么是这6步，我们展开分析。

### 1 注册RequestMappingHandlerMapping bean

第一步还是先找程序入口。

使用Spring MVC的同学都知道，要想使@RequestMapping注解生效，必须得在xml配置文件中配置< mvc:annotation-driven/>。因此我们以此为突破口开始分析。

在b[ean 解析、注册、实例化流程源码剖析 ](https://www.cnblogs.com/grasp/p/11074770.html)文中我们知道xml配置文件解析完的下一步就是解析bean。在这里我们继续对那个方法展开分析。如下：

```
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        //如果该元素属于默认命名空间走此逻辑。Spring的默认namespace为：http://www.springframework.org/schema/beans“
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                     //对document中的每个元素都判断其所属命名空间，然后走相应的解析逻辑
                    if (delegate.isDefaultNamespace(ele)) {
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        else {
            //如果该元素属于自定义namespace走此逻辑 ，比如AOP，MVC等。
            delegate.parseCustomElement(root);
        }
    }
```

方法中根据元素的命名空间来进行不同的逻辑处理，如bean、beans等属于默认命名空间执行parseDefaultElement()方法，其它命名空间执行parseCustomElement()方法。

< mvc:annotation-driven/>元素属于mvc命名空间，因此进入到 parseCustomElement()方法。

```
    public BeanDefinition parseCustomElement(Element ele) {
        return parseCustomElement(ele, null);
    }
```

进入parseCustomElement(ele, null)方法。

```
    public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
        //获取该元素namespace url
        String namespaceUri = getNamespaceURI(ele);
        //得到NamespaceHandlerSupport实现类解析元素
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler == null) {
            error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        }
        return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
    }
```

进入NamespaceHandlerSupport类的parse()方法。

```
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        //此处得到AnnotationDrivenBeanDefinitionParser类来解析该元素
        return findParserForElement(element, parserContext).parse(element, parserContext);
    }
```

上面方法分为两步，（1）获取元素的解析类。（2）解析元素。

（1）获取解析类。

```
    private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
        String localName = parserContext.getDelegate().getLocalName(element);
        BeanDefinitionParser parser = this.parsers.get(localName);
        if (parser == null) {
            parserContext.getReaderContext().fatal(
                    "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
        }
        return parser;
    }
```

Spring MVC中含有多种命名空间，此方法会根据元素所属命名空间得到相应解析类，参考[spring xml 配置文件中标签的解析](https://www.cnblogs.com/grasp/p/11079748.html)，其中< mvc:annotation-driven/>对应的是AnnotationDrivenBeanDefinitionParser解析类。

（2）解析< mvc:annotation-driven/>元素

进入AnnotationDrivenBeanDefinitionParser类的parse()方法。

```
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        Object source = parserContext.extractSource(element);
        XmlReaderContext readerContext = parserContext.getReaderContext();

        CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
        parserContext.pushContainingComponent(compDefinition);

        RuntimeBeanReference contentNegotiationManager = getContentNegotiationManager(element, source, parserContext);

        //生成RequestMappingHandlerMapping bean信息
        RootBeanDefinition handlerMappingDef = new RootBeanDefinition(RequestMappingHandlerMapping.class);
        handlerMappingDef.setSource(source);
        handlerMappingDef.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        handlerMappingDef.getPropertyValues().add("order", 0);
        handlerMappingDef.getPropertyValues().add("contentNegotiationManager", contentNegotiationManager);

        ......
        
        //此处HANDLER_MAPPING_BEAN_NAME值为:RequestMappingHandlerMapping类名
        //容器中注册name为RequestMappingHandlerMapping类名
        parserContext.registerComponent(new BeanComponentDefinition(handlerMappingDef, HANDLER_MAPPING_BEAN_NAME));
        
        ......
    }
```

可以看到上面方法在Spring MVC容器中注册了一个名为“HANDLER_MAPPING_BEAN_NAME”,类型为RequestMappingHandlerMapping的bean(看此函数的其它代码，得到同时也注册了RequestMappingHandlerAdapter等)。

至于这个bean能干吗，继续往下分析。

### 2. RequestMappingHandlerMapping bean实例化

bean注册完后的下一步就是实例化。

在开始分析实例化流程之前，我们先介绍一下RequestMappingHandlerMapping是个什么样类。

#### 2.1 RequestMappingHandlerMapping继承图

![img](https://img2018.cnblogs.com/blog/738818/201906/738818-20190627214336170-1548778266.png)

上图信息比较多，我们查找关键信息。可以看到这个类间接实现了HandlerMapping接口，是HandlerMapping类型的实例。

除此之外还实现了ApplicationContextAware和IntitalzingBean 这两个接口。

#### 2.2 ApplicationContextAware接口

下面是[官方介绍](https://link.juejin.im/?target=https%3A%2F%2Fdocs.spring.io%2Fspring-framework%2Fdocs%2Fcurrent%2Fjavadoc-api%2Forg%2Fspringframework%2Fcontext%2FApplicationContextAware.html)：

在这里简要介绍一下这两个接口：

```
public interface ApplicationContextAware
extends Aware
```

Interface to be implemented by any object that wishes to be notified of the [`ApplicationContext`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/context/ApplicationContext.html) that it runs in.

该接口只包含以下方法：

```
void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
* Set the ApplicationContext that this object runs in.
* Normally this call will be used to initialize the object.
```

概括一下上面表达的信息：如果一个类实现了ApplicationContextAware接口，Spring容器在初始化该类时候会自动回调该类的setApplicationContext()方法。这个接口主要用来让实现类得到Spring 容器上下文信息。

#### 2.3 InitializingBean接口

 下面是[官方介绍](https://link.juejin.im/?target=https%3A%2F%2Fdocs.spring.io%2Fspring-framework%2Fdocs%2Fcurrent%2Fjavadoc-api%2Forg%2Fspringframework%2Fbeans%2Ffactory%2FInitializingBean.html)：

```
public interface InitializingBean
```

Interface to be implemented by beans that need to react once all their properties have been set by a [`BeanFactory`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/beans/factory/BeanFactory.html): e.g. to perform custom initialization, or merely to check that all mandatory properties have been set.

该接口只包含以下方法：

```
void afterPropertiesSet() throws Exception;
* Invoked by a BeanFactory after it has set all bean properties supplied
* (and satisfied BeanFactoryAware and ApplicationContextAware).
```

概括一下上面表达的信息：如果一个bean实现了该接口，Spring 容器初始化bean时会回调afterPropertiesSet()方法。这个接口的主要作用是让bean在初始化时可以实现一些自定义的操作。

介绍完RequestMappingHandlerMapping类后我们开始对这个类的源码进行分析。

#### 2.2.4 RequestMappingHandlerMapping类源码分析

既然RequestMappingHandlerMapping实现了ApplicationContextAware接口，那实例化时候肯定会执行setApplicationContext方法，我们查看其实现逻辑。

```
    public final void setApplicationContext(ApplicationContext context) throws BeansException {
        if (context == null && !isContextRequired()) {
            // Reset internal context state.
            this.applicationContext = null;
            this.messageSourceAccessor = null;
        }
        else if (this.applicationContext == null) {
            // Initialize with passed-in context.
            if (!requiredContextClass().isInstance(context)) {
                throw new ApplicationContextException(
                        "Invalid application context: needs to be of type [" + requiredContextClass().getName() + "]");
            }
            this.applicationContext = context;
            this.messageSourceAccessor = new MessageSourceAccessor(context);
            initApplicationContext(context);
        }
        else {
            // Ignore reinitialization if same context passed in.
            if (this.applicationContext != context) {
                throw new ApplicationContextException(
                        "Cannot reinitialize with different application context: current one is [" +
                        this.applicationContext + "], passed-in one is [" + context + "]");
            }
        }
    }
```

可以看到此方法把容器上下文赋值给applicationContext变量，因为现在是Spring MVC容器创建流程，因此此处设置的值就是Spring MVC容器 。

RequestMappingHandlerMapping也实现了InitializingBean接口，当设置完属性后肯定会回调afterPropertiesSet方法，再看afterPropertiesSet方法逻辑。

```
    public void afterPropertiesSet() {
        this.config = new RequestMappingInfo.BuilderConfiguration();
        this.config.setUrlPathHelper(getUrlPathHelper());
        this.config.setPathMatcher(getPathMatcher());
        this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
        this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
        this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
        this.config.setContentNegotiationManager(getContentNegotiationManager());

        super.afterPropertiesSet();
    }
```

上面调用了父类的afterPropertiesSet()方法，沿调用栈继续查看。

```
    public void afterPropertiesSet() {
        //初始化handler函数
        initHandlerMethods();
    }
```

进入initHandlerMethods初始化方法查看逻辑。

```
    protected void initHandlerMethods() {
        if (logger.isDebugEnabled()) {
            logger.debug("Looking for request mappings in application context: " + getApplicationContext());
        }
        //1.获取容器中所有bean 的name。
        //根据detectHandlerMethodsInAncestorContexts bool变量的值判断是否获取父容器中的bean，默认为false。因此这里只获取Spring MVC容器中的bean，不去查找父容器
        String[] beanNames = (this.detectHandlerMethodsInAncestorContexts ?
                BeanFactoryUtils.beanNamesForTypeIncludingAncestors(getApplicationContext(), Object.class) :
                getApplicationContext().getBeanNamesForType(Object.class));
        //循环遍历bean
        for (String beanName : beanNames) {
            if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
                Class<?> beanType = null;
                try {
                    beanType = getApplicationContext().getType(beanName);
                }
                catch (Throwable ex) {
                    // An unresolvable bean type, probably from a lazy bean - let's ignore it.
                    if (logger.isDebugEnabled()) {
                        logger.debug("Could not resolve target class for bean with name '" + beanName + "'", ex);
                    }
                }
                //2.判断bean是否含有@Controller或者@RequestMappin注解
                if (beanType != null && isHandler(beanType)) {
                    //3.对含有注解的bean进行处理，获取handler函数信息。
                    detectHandlerMethods(beanName);
                }
            }
        }
        handlerMethodsInitialized(getHandlerMethods());
    }
```

上面函数分为3步。

（1）获取Spring MVC容器中的bean。

（2）找出含有含有@Controller或者@RequestMappin注解的bean。

（3）对含有注解的bean进行解析。

第1步很简单就是获取容器中所有的bean name，我们对第2、3步作分析。

查看isHandler()方法实现逻辑。

```
    @Override
    protected boolean isHandler(Class<?> beanType) {
        return (AnnotatedElementUtils.hasAnnotation(beanType, Controller.class) ||
                AnnotatedElementUtils.hasAnnotation(beanType, RequestMapping.class));
    }
```

上面逻辑很简单，就是判断该bean是否有@Controller或@RequestMapping注解，然后返回判断结果。

如果含有这两个注解之一就进入detectHandlerMethods（）方法进行处理。

查看detectHandlerMethods（）方法。

```
    protected void detectHandlerMethods(final Object handler) {
        //1.获取bean的类信息
        Class<?> handlerType = (handler instanceof String ?
                getApplicationContext().getType((String) handler) : handler.getClass());
        final Class<?> userType = ClassUtils.getUserClass(handlerType);

        //2.遍历函数获取有@RequestMapping注解的函数信息
        Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
                new MethodIntrospector.MetadataLookup<T>() {
                    @Override
                    public T inspect(Method method) {
                        try {
                            //如果有@RequestMapping注解，则获取函数映射信息
                            return getMappingForMethod(method, userType);
                        }
                        catch (Throwable ex) {
                            throw new IllegalStateException("Invalid mapping on handler class [" +
                                    userType.getName() + "]: " + method, ex);
                        }
                    }
                });

        if (logger.isDebugEnabled()) {
            logger.debug(methods.size() + " request handler methods found on " + userType + ": " + methods);
        }
        //3.遍历映射函数列表，注册handler
        for (Map.Entry<Method, T> entry : methods.entrySet()) {
            Method invocableMethod = AopUtils.selectInvocableMethod(entry.getKey(), userType);
            T mapping = entry.getValue();
            //注册handler函数
            registerHandlerMethod(handler, invocableMethod, mapping);
        }
    }
```



上面方法中用了几个回调，可能看起来比较复杂，其主要功能就是获取该bean和父接口中所有用@RequestMapping注解的函数信息，并把这些保存到methodMap变量中。

我们对上面方法进行逐步分析，看看如何对有@RequestMapping注解的函数进行解析。

先进入selectMethods()方法查看实现逻辑。

```
    public static <T> Map<Method, T> selectMethods(Class<?> targetType, final MetadataLookup<T> metadataLookup) {
        final Map<Method, T> methodMap = new LinkedHashMap<Method, T>();
        Set<Class<?>> handlerTypes = new LinkedHashSet<Class<?>>();
        Class<?> specificHandlerType = null;
        //把自身类添加到handlerTypes中
        if (!Proxy.isProxyClass(targetType)) {
            handlerTypes.add(targetType);
            specificHandlerType = targetType;
        }
        //获取该bean所有的接口，并添加到handlerTypes中
        handlerTypes.addAll(Arrays.asList(targetType.getInterfaces()));

        /对自己及所有实现接口类进行遍历
        for (Class<?> currentHandlerType : handlerTypes) {
            final Class<?> targetClass = (specificHandlerType != null ? specificHandlerType : currentHandlerType);
            //获取函数映射信息
            ReflectionUtils.doWithMethods(currentHandlerType, new ReflectionUtils.MethodCallback() {
                //循环获取类中的每个函数，通过回调处理
                @Override
                public void doWith(Method method) {
                    //对类中的每个函数进行处理
                    Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
                    //回调inspect（）方法给个函数生成RequestMappingInfo  
                    T result = metadataLookup.inspect(specificMethod);
                    if (result != null) {
                        Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
                        if (bridgedMethod == specificMethod || metadataLookup.inspect(bridgedMethod) == null) {
                            //将生成的RequestMappingInfo保存到methodMap中
                            methodMap.put(specificMethod, result);
                        }
                    }
                }
            }, ReflectionUtils.USER_DECLARED_METHODS);
        }
        //返回保存函数映射信息后的methodMap
        return methodMap;
    }
```

上面逻辑中doWith()回调了inspect(),inspect()又回调了getMappingForMethod（）方法。

我们看看getMappingForMethod()是如何生成函数信息的。

```
    @Override
    protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
        //创建函数信息
        RequestMappingInfo info = createRequestMappingInfo(method);
        if (info != null) {
            RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
            if (typeInfo != null) {
                info = typeInfo.combine(info);
            }
        }
        return info;
    }
```

查看createRequestMappingInfo()方法。

```
    private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
        //如果该函数含有@RequestMapping注解,则根据其注解信息生成RequestMapping实例，
        //如果该函数没有@RequestMapping注解则返回空
        RequestMapping requestMapping = AnnotatedElementUtils.findMergedAnnotation(element, RequestMapping.class);
        RequestCondition<?> condition = (element instanceof Class ?
                getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
        //如果requestMapping不为空，则生成函数信息MAP后返回
        return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
    }
```

看看createRequestMappingInfo是如何实现的。

```
    protected RequestMappingInfo createRequestMappingInfo(
            RequestMapping requestMapping, RequestCondition<?> customCondition) {

        return RequestMappingInfo
                .paths(resolveEmbeddedValuesInPatterns(requestMapping.path()))
                .methods(requestMapping.method())
                .params(requestMapping.params())
                .headers(requestMapping.headers())
                .consumes(requestMapping.consumes())
                .produces(requestMapping.produces())
                .mappingName(requestMapping.name())
                .customCondition(customCondition)
                .options(this.config)
                .build();
    可以看到上面把RequestMapping注解中的信息都放到一个RequestMappingInfo实例中后返回。当生成含有@RequestMapping注解的函数映射信息后，最后一步是调用registerHandlerMethod 注册handler和处理函数映射关系。    protectedvoid registerHandlerMethod(Object handler, Method method, T mapping) {
```

```
this.mappingRegistry.register(mapping, handler, method);
    }
```

看到把所有的handler方法都注册到了mappingRegistry这个变量中。

到此就把RequestMappingHandlerMapping bean的实例化流程就分析完了。

### 3 获取RequestMapping bean

这里我们回到Spring MVC容器初始化流程，查看在FrameworkServlet#initWebApplicationContext方法。

```
    protected WebApplicationContext initWebApplicationContext() {
        //1.获得rootWebApplicationContext
        WebApplicationContext rootContext =
                WebApplicationContextUtils.getWebApplicationContext(getServletContext());
        WebApplicationContext wac = null;

        ......
        
        if (wac == null) {
            // No context instance is defined for this servlet -> create a local one
            //2.创建 Spring 容器 
            wac = createWebApplicationContext(rootContext);
        }

        if (!this.refreshEventReceived) {
            // Either the context is not a ConfigurableApplicationContext with refresh
            // support or the context injected at construction time had already been
            // refreshed -> trigger initial onRefresh manually here.
            //3.初始化容器
            onRefresh(wac);
        }

        ......

       return wac;
    }
```

前两步我们在[spring与springmvc父子容器](https://www.cnblogs.com/grasp/p/11042580.html)一文中分析过，主要是创建Spring MVC容器，这里我们重点看第3步。

进入onRefresh()方法。

```
    @Override
    protected void onRefresh(ApplicationContext context) {
        //执行初始化策略 
        initStrategies(context);
    }
```

进入initStrategies方法，该方法进行了很多初始化行为，为减少干扰我们只过滤出与本文相关内容。

```
    protected void initStrategies(ApplicationContext context) {
        ......
        //初始化HandlerMapping
        initHandlerMappings(context);
        ......
    }
```

进入initHandlerMappings()方法。

```
    private void initHandlerMappings(ApplicationContext context) {
        this.handlerMappings = null;

        if (this.detectAllHandlerMappings) {
            // Find all HandlerMappings in the ApplicationContext, including ancestor contexts.
            //容器中查找HandlerMapping的实例
            Map<String, HandlerMapping> matchingBeans =
                    BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
            if (!matchingBeans.isEmpty()) {
                //把找到的bean放到hanlderMappings中。
                this.handlerMappings = new ArrayList<HandlerMapping>(matchingBeans.values());
                // We keep HandlerMappings in sorted order.
                AnnotationAwareOrderComparator.sort(this.handlerMappings);
            }
        }
        else {
            try {
                HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
                this.handlerMappings = Collections.singletonList(hm);
            }
            catch (NoSuchBeanDefinitionException ex) {
                // Ignore, we'll add a default HandlerMapping later.
            }
        }

        // Ensure we have at least one HandlerMapping, by registering
        // a default HandlerMapping if no other mappings are found.
        if (this.handlerMappings == null) {
            this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
            if (logger.isDebugEnabled()) {
                logger.debug("No HandlerMappings found in servlet '" + getServletName() + "': using default");
            }
        }
    }
```

默认有多个实例，其中就有前面注册并实例化了的RequestMappingHandlerMapping bean

![img](https://img2018.cnblogs.com/blog/738818/201906/738818-20190627222753202-369192074.png)

### 4 接收请求

DispatchServlet继承自Servlet，那所有的请求都会在service()方法中进行处理。

查看service()方法。

```
    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {

        //获取请求方法
        HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
        //若是patch请求执行此逻辑
        if (HttpMethod.PATCH == httpMethod || httpMethod == null) {
            processRequest(request, response);
        }
        else {
            //其它请求走此逻辑
            super.service(request, response);
        }
    }
```

我们跟着源码，得到不管get、post最后都会执行到DispatcherServlet#doDispatch(request, response);

### 5 获取handler

最终所有的web请求都由doDispatch()方法进行处理，查看其逻辑。

```
    protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
        HttpServletRequest processedRequest = request;
        HandlerExecutionChain mappedHandler = null;
        ......
        //根据请求获得真正处理的handler
        mappedHandler = getHandler(processedRequest);
        ......
    }
```

查看getHandler()。

```
    protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        //获取HandlerMapping实例
        for (HandlerMapping hm : this.handlerMappings) {
            if (logger.isTraceEnabled()) {
                logger.trace(
                        "Testing handler map [" + hm + "] in DispatcherServlet with name '" + getServletName() + "'");
            }
             //得到处理请求的handler
            HandlerExecutionChain handler = hm.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
        return null;
    }
```

这里遍历handlerMappings获得所有HandlerMapping实例，还记得handlerMappings变量吧，这就是前面initHandlerMappings()方法中设置进去的值。

可以看到接下来调了用HandlerMapping实例的getHanlder()方法查找handler，看其实现逻辑。

```
    @Override
    public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
        Object handler = getHandlerInternal(request);
        ......
        HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
        ......
        return executionChain;
    }
```

进入getHandlerInternal()方法。

```
    @Override
    protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
        //获取函数url
        String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
        ......
        try {
            //查找HandlerMethod 
            HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
            ......
        }
        finally {
            this.mappingRegistry.releaseReadLock();
        }
    }
```

进入lookupHandlerMethod()。

```
    protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) throws Exception {
        List<Match> matches = new ArrayList<Match>();
        List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
        ......
    }
```

可以看到上面方法中从mappingRegistry获取handler，这个mappingRegistry的值还记得是从哪里来的吗？

就是前面RequestMappingHandlerMapping 实例化过程的最后一步调用registerHandlerMethod()函数时设置进去的。

### 6 handler处理请求

获取到相应的handler后剩下的事情就是进行业务逻辑。处理后返回结果，这里基本也没什么好说的。

到此整个@RequestMapping的流程也分析完毕。

## 3.小结

认真读完上面深入分析@RequestMapping注解流程的同学，相信此时肯定对Spring MVC有了更深一步的认识。

在@ReqestMapping解析过程中，initHandlerMethods()函数只是对Spring MVC 容器中的bean进行处理的，并没有去查找父容器的bean。因此不会对父容器中含有@RequestMapping注解的函数进行处理，更不会生成相应的handler。

所以当请求过来时找不到处理的handler，导致404。

## 4.尾声

从上面的分析中，我们知道要使用@RequestMapping注解，必须得把含有@RequestMapping的bean定义到spring-mvc.xml中。

这里也给大家个建议：

因为@RequestMapping一般会和@Controller搭配使。为了防止重复注册bean，建议在spring-mvc.xml配置文件中只扫描含有Controller bean的包，其它的共用bean的注册定义到spring.xml文件中。写法如下：

spring-mvc.xml

```
<!-- 只扫描@Controller注解 -->
<context:component-scan base-package="com.xxx.controller" use-default-filters="false">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller" />
</context:component-scan>
```

spring.xml

```
<!-- 配置扫描注解,不扫描@Controller注解 -->
<context:component-scan base-package="com.xxx">
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller" />
</context:component-scan>。
```

use-default-filters属性默认为true，会扫描所有注解类型的bean 。如果配置成false，就只扫描白名单中定义的bean注解。