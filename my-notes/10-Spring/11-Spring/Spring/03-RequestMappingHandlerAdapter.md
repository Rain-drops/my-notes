# [RequestMappingHandlerAdapter和RequestParam原理分析](https://www.cnblogs.com/grasp/p/11105386.html)

我们要使用定义了RequestMapping方法或者类是，需要先准备好所需要的参数。如何准备参数，我们应该考虑些上面问题。

### 都有哪些参数需要绑定？

除了方法确定的参数，还有两个方法的参数需要绑定，那就是当前处理器相对应注释了@ModelAttribute和注释了@InitBinder的方法。

### 参数的值的来源？

有六个参数的来源：

1. request中相关的参数，主要包括url中的参数、post中过来body中的参数，以及请求头包含的值；
2. cookie中的参数
3. session中给的参数
4. 设置到FlashMap中的参数，这种参数用于redirect的参数传递
5. SessionAttribute传递的参数，这类参数通过@SessionAttribute注释传递
6. 通过相应的注释了@ModelAttribute的方法设置的参数

### 具体进行绑定的方法？

​    参数解析使用HandlerMethodArgumentResolver类型的组件完成的，不同类型的使用不同的ArgumentResolver来解析。有的Resolver内部使用了WebDataBinder，可以通过注释了@InitBinder的方法来初始化，注释了@InitBinder的方法也需要绑定参数，而且也是不确定的，所以@InitBinder注释的方法也需要ArgumentResolver来解析参数，使用的和Handler不同的一套ArgumentResolver，另外注释了ModelAttribute的方法也需要绑定参数，使用的和Handler使用的是同一套ArgumentResolver。

## 1.RequestMappingHandlerAdapter

此类的主要功能：

- 备好处理器所需要的参数（这个最难，参数的不确定性）
- 使用处理器处理请求 （这个比较简单，直接用反射调用handleMethod处理就可以了）
- 处理返回值，也就是将不同类型的返回值统一处理成ModelAndView类

通过xml解析初始化过程如[RequestMapping原理分析和RequestMappingHandlerMapping](https://www.cnblogs.com/grasp/p/11100124.html)一般，重复部分就不在分析。

#### 1.1RequestMappingHandlerAdapter继承图

![img](https://img2018.cnblogs.com/blog/738818/201906/738818-20190628225703915-1493701591.png)

上图信息比较多，我们查找关键信息。可以看到这个类间接实现了HandlerAdapter接口，是HandlerAdapter类型的实例。

除此之外还实现了ApplicationContextAware和IntitalzingBean 这两个接口。

#### 1.2RequestMappingHandlerAdapter类源码分析

既然RequestMappingHandlerAdapter实现了ApplicationContextAware接口，那实例化时候肯定会执行setApplicationContext方法，我们查看其实现逻辑。

```
    @Override
    public final void setApplicationContext(ApplicationContext context) throws BeansException {
        //isContextRequired()方法返回tru
        if (context == null && !isContextRequired()) {
            // Reset internal context state.
            this.applicationContext = null;
            this.messageSourceAccessor = null;
        }
        else if (this.applicationContext == null) {
            // Initialize with passed-in context.
            //所传入的context如果不能被实例化，则抛出异常
            if (!requiredContextClass().isInstance(context)) {
                throw new ApplicationContextException(
                        "Invalid application context: needs to be of type [" + requiredContextClass().getName() + "]");
            }
            this.applicationContext = context;
            //国际化
            this.messageSourceAccessor = new MessageSourceAccessor(context);
            //初始化ApplicationContext
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



initApplicationContext里面主要就是初始化ServletContext，就不在去分析了

RequestMappingHandlerAdapter也实现了InitializingBean接口，当设置完属性后肯定会回调afterPropertiesSet方法，我们重点分析一下afterPropertiesSet的方法，它的源码内容如下，下面我们一点一点的分析这个类：



```
    @Override
    public void afterPropertiesSet() {
        // Do this first, it may add ResponseBody advice beans
        //初始化注释了@ControllerAdvice的类，分别用于缓存@ControllerAdvice注释的类里面注释了@ModelAttribute和@InitBinder方法，也就是全局的@ModelAttribute和InitBinder方法。
        initControllerAdviceCache();

        if (this.argumentResolvers == null) {
            //初始化argumentResolvers，用于处理器方法和注释了@ModelAttribute的方法设置参数
            List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
            this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
        }
        if (this.initBinderArgumentResolvers == null) {
            //初始化initBinderArgumentResolvers，用于给注释了@initBinder的方法设置参数,使用得较少
            List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
            this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
        }
        if (this.returnValueHandlers == null) {
            //初始化returnValueHandlers，用于将处理器的返回值处理为ModelAndView的类型
            List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
            this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
        }
    }
```



我们先看initControllerAdviceCache这个方法：



```
    private void initControllerAdviceCache() {
        if (getApplicationContext() == null) {
            return;
        }
        if (logger.isInfoEnabled()) {
            logger.info("Looking for @ControllerAdvice: " + getApplicationContext());
        }

        //获取到所有注释了@ControllerAdvice的bean
        List<ControllerAdviceBean> beans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
        //对获取到的ControllerAdvice注解的类进行排序，排序的规则是基于实现PriorityOrdered接口或者带有Order注解
        AnnotationAwareOrderComparator.sort(beans);

        List<Object> requestResponseBodyAdviceBeans = new ArrayList<Object>();

        for (ControllerAdviceBean bean : beans) {
            //查找注释了@ModelAttribute而且没有注释@RequestMapping的方法
            Set<Method> attrMethods = MethodIntrospector.selectMethods(bean.getBeanType(), MODEL_ATTRIBUTE_METHODS);
            if (!attrMethods.isEmpty()) {
                this.modelAttributeAdviceCache.put(bean, attrMethods);
                if (logger.isInfoEnabled()) {
                    logger.info("Detected @ModelAttribute methods in " + bean);
                }
            }
            //获取所有带InitBinder注解的方法
            Set<Method> binderMethods = MethodIntrospector.selectMethods(bean.getBeanType(), INIT_BINDER_METHODS);
            if (!binderMethods.isEmpty()) {
                this.initBinderAdviceCache.put(bean, binderMethods);
                if (logger.isInfoEnabled()) {
                    logger.info("Detected @InitBinder methods in " + bean);
                }
            }
            //如果实现了RequestBodyAdvice接口
            if (RequestBodyAdvice.class.isAssignableFrom(bean.getBeanType())) {
                requestResponseBodyAdviceBeans.add(bean);
                if (logger.isInfoEnabled()) {
                    logger.info("Detected RequestBodyAdvice bean in " + bean);
                }
            }
            //如果实现了ResponseBodyAdvice接口
            if (ResponseBodyAdvice.class.isAssignableFrom(bean.getBeanType())) {
                requestResponseBodyAdviceBeans.add(bean);
                if (logger.isInfoEnabled()) {
                    logger.info("Detected ResponseBodyAdvice bean in " + bean);
                }
            }
        }

        // 将实现了RequestBodyAdvice和ResponseBodyAdvice接口的类放入requestResponseBodyAdviceBeans
        // 这里是放入顶部,说明通过@@ControllerAdvice注解实现接口的处理优先级最高
        if (!requestResponseBodyAdviceBeans.isEmpty()) {
            this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
        }
    }
```



看getDefaultArgumentResolvers主要作用是解析传入的参数： 



```
// 获取默认的 HandlerMethodArgumentResolver
private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() { 
    List<HandlerMethodArgumentResolver> resolvers = new ArrayList<HandlerMethodArgumentResolver>();
    // 1.基于注解的参数解析 <-- 解析的数据来源主要是 HttpServletRequest | ModelAndViewContainer
    // Annotation-based argument resolution
    // 解析被注解 @RequestParam, @RequestPart 修饰的参数, 数据的获取通过 HttpServletRequest.getParameterValues
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
    // 解析被注解 @RequestParam 修饰, 且类型是 Map 的参数, 数据的获取通过 HttpServletRequest.getParameterMap
    resolvers.add(new RequestParamMapMethodArgumentResolver());
    // 解析被注解 @PathVariable 修饰, 数据的获取通过 uriTemplateVars, 而 uriTemplateVars 却是通过 RequestMappingInfoHandlerMapping.handleMatch 生成, 其实就是 uri 中映射出的 key <-> value
    resolvers.add(new PathVariableMethodArgumentResolver());
    // 解析被注解 @PathVariable 修饰 且数据类型是 Map, 数据的获取通过 uriTemplateVars, 而 uriTemplateVars 却是通过 RequestMappingInfoHandlerMapping.handleMatch 生成, 其实就是 uri 中映射出的 key <-> value
    resolvers.add(new PathVariableMapMethodArgumentResolver());
    // 解析被注解 @MatrixVariable 修饰, 数据的获取通过 URI提取了;后存储的 uri template 变量值
    resolvers.add(new MatrixVariableMethodArgumentResolver());
    // 解析被注解 @MatrixVariable 修饰 且数据类型是 Map, 数据的获取通过 URI提取了;后存储的 uri template 变量值
    resolvers.add(new MatrixVariableMapMethodArgumentResolver());
    // 解析被注解 @ModelAttribute 修饰, 且类型是 Map 的参数, 数据的获取通过 ModelAndViewContainer 获取, 通过 DataBinder 进行绑定
    resolvers.add(new ServletModelAttributeMethodProcessor(false));
    // 解析被注解 @RequestBody 修饰的参数, 以及被@ResponseBody修饰的返回值, 数据的获取通过 HttpServletRequest 获取, 根据 MediaType通过HttpMessageConverter转换成对应的格式, 在处理返回值时 也是通过 MediaType 选择合适HttpMessageConverter, 进行转换格式, 并输出
    resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    // 解析被注解 @RequestPart 修饰, 数据的获取通过 HttpServletRequest.getParts()
    resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
    // 解析被注解 @RequestHeader 修饰, 数据的获取通过 HttpServletRequest.getHeaderValues()
    resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
    // 解析被注解 @RequestHeader 修饰且参数类型是 Map, 数据的获取通过 HttpServletRequest.getHeaderValues()
    resolvers.add(new RequestHeaderMapMethodArgumentResolver());
    // 解析被注解 @CookieValue 修饰, 数据的获取通过 HttpServletRequest.getCookies()
    resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
    // 解析被注解 @Value 修饰, 数据在这里没有解析
    resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
    // 解析被注解 @SessionAttribute 修饰, 数据的获取通过 HttpServletRequest.getAttribute(name, RequestAttributes.SCOPE_SESSION)
    resolvers.add(new SessionAttributeMethodArgumentResolver());
    // 解析被注解 @RequestAttribute 修饰, 数据的获取通过 HttpServletRequest.getAttribute(name, RequestAttributes.SCOPE_REQUEST)
    resolvers.add(new RequestAttributeMethodArgumentResolver());

    // 2.基于类型的参数解析器
    // Type-based argument resolution
    // 解析固定类型参数(比如: ServletRequest, HttpSession, InputStream 等), 参数的数据获取还是通过 HttpServletRequest
    resolvers.add(new ServletRequestMethodArgumentResolver());
    // 解析固定类型参数(比如: ServletResponse, OutputStream等), 参数的数据获取还是通过 HttpServletResponse
    resolvers.add(new ServletResponseMethodArgumentResolver());
    // 解析固定类型参数(比如: HttpEntity, RequestEntity 等), 参数的数据获取还是通过 HttpServletRequest
    resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
    // 解析固定类型参数(比如: RedirectAttributes), 参数的数据获取还是通过 HttpServletResponse
    resolvers.add(new RedirectAttributesMethodArgumentResolver());
    // 解析固定类型参数(比如: Model等), 参数的数据获取通过 ModelAndViewContainer
    resolvers.add(new ModelMethodProcessor());
    // 解析固定类型参数(比如: Model等), 参数的数据获取通过 ModelAndViewContainer
    resolvers.add(new MapMethodProcessor());
    // 解析固定类型参数(比如: Errors), 参数的数据获取通过 ModelAndViewContainer
    resolvers.add(new ErrorsMethodArgumentResolver());
    // 解析固定类型参数(比如: SessionStatus), 参数的数据获取通过 ModelAndViewContainer
    resolvers.add(new SessionStatusMethodArgumentResolver());
    // 解析固定类型参数(比如: UriComponentsBuilder), 参数的数据获取通过 HttpServletRequest
    resolvers.add(new UriComponentsBuilderMethodArgumentResolver());
    // 3.自定义参数解析器
    // Custom arguments
    if (getCustomArgumentResolvers() != null) {
        resolvers.addAll(getCustomArgumentResolvers());
    }
    // Catch-all
    //这两个解析器可以解析所有类型的参数
    resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
    resolvers.add(new ServletModelAttributeMethodProcessor(true));
    return resolvers;
}
```



getDefaultArgumentResolvers中有四类解析器:

1. 基于注解的参数解析器
2. 基于类型的参数解析器
3. 自定义参数解析器
4. 可解析所有类型的解析器

从源码中可以看出,自定义的解析器是在前面两种都无法解析是才会使用到,这个顺序是无法改变的,例如如果想自己写一个解析器来解析@PathVariable注释的PathVariable参数,是无法实现的,即使写出来并注册到RequestMappingHanderAdapter中也不会被调用

getDefaultInitBinderArgumentResolvers用得较少，原理也是类似的，就不分析了。

分析getDefaultReturnValueHandlers方法：



```
private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
    List<HandlerMethodReturnValueHandler> handlers = new ArrayList<HandlerMethodReturnValueHandler>();

    // Single-purpose return value types
    // 支持 ModelAndView 类型的 HandlerMethodReturnValueHandler, 最后将数据写入 ModelAndViewContainer
    handlers.add(new ModelAndViewMethodReturnValueHandler());
    // 支持 Map 类型的 HandlerMethodReturnValueHandler, 最后将数据写入 ModelAndViewContainer
    handlers.add(new ModelMethodProcessor());
    // 支持 View 类型的 HandlerMethodReturnValueHandler, 最后将数据写入 ModelAndViewContainer
    handlers.add(new ViewMethodReturnValueHandler());
    // 支持 ResponseEntity 类型的 HandlerMethodReturnValueHandler, 最后将数据写入 HttpServletResponse 的数据流中 OutputStream
    handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters()));
    // 支持 ResponseEntity | StreamingResponseBody 类型的 HandlerMethodReturnValueHandler, 最后将数据写入 HttpServletResponse 的数据流中 OutputStream
    handlers.add(new StreamingResponseBodyReturnValueHandler());
    // 支持 HttpEntity | !RequestEntity 类型的 HandlerMethodReturnValueHandler, 最后将数据写入 HttpServletResponse 的数据流中 OutputStream
    handlers.add(new HttpEntityMethodProcessor(getMessageConverters(),this.contentNegotiationManager, this.requestResponseBodyAdvice));
    // 支持 HttpHeaders 类型的 HandlerMethodReturnValueHandler, 最后将数据写入 HttpServletResponse 的数据头部
    handlers.add(new HttpHeadersReturnValueHandler());
    // 支持 Callable 类型的 HandlerMethodReturnValueHandler
    handlers.add(new CallableMethodReturnValueHandler());
    handlers.add(new DeferredResultMethodReturnValueHandler());
    // 支持 WebAsyncTask 类型的 HandlerMethodReturnValueHandler
    handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

    // Annotation-based return value types
    // 将数据加入 ModelAndViewContainer 的 HandlerMethodReturnValueHandler
    handlers.add(new ModelAttributeMethodProcessor(false));
    // 返回值被 ResponseBody 修饰的返回值, 并且根据 MediaType 通过 HttpMessageConverter 转化后进行写入数据流中
    handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(),this.contentNegotiationManager, this.requestResponseBodyAdvice));

    // Multi-purpose return value types
    // 支持返回值为 CharSequence 类型, 设置 ModelAndViewContainer.setViewName
    handlers.add(new ViewNameMethodReturnValueHandler());
    // 支持返回值为 Map, 并将结果设置到 ModelAndViewContainer
    handlers.add(new MapMethodProcessor());
    // Custom return value types
    if (getCustomReturnValueHandlers() != null) {
        handlers.addAll(getCustomReturnValueHandlers());
    }
    // Catch-all
    if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
        handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
    }
    else {
        handlers.add(new ModelAttributeMethodProcessor(true));
    }
    return handlers;
}
```



在这里就初始化完成了，有了这些工具类后我们就可以对收到请求的参数和处理后返回不同的类型进行处理了。

DispatchServlet继承自Servlet，那所有的请求都会在service()方法中进行处理。最后在DispatcherServlet#doDispatch中处理请求后。根据不同的请求会找到对应的HandlerMapping，然后在找到HandlerAdapter进行处理。

如果处理的HandlerAdapter是RequestMappingHandlerAdapter，最后会走到RequestMappingHandlerAdapter#handleInternal：



```
    @Override
    protected ModelAndView handleInternal(HttpServletRequest request,
            HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
            ......
            mav = invokeHandlerMethod(request, response, handlerMethod);
            ......
    }
```



进入invokeHandlerMethod：



```
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    // 构建 ServletWebRequest <-- 主要由 HttpServletRequest, HttpServletResponse
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        // 构建 DataBinder 工厂
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        // binderFactory 中存储着被 @InitBinder, @ModelAttribute 修饰的方法 <- 最终包裹成 InvocableHandlerMethod
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
        // 构建一个 ServletInvocableHandlerMethod
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        // 设置方法参数解析器 HandlerMethodArgumentValueResolver
        invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        // 返回值处理器 HandlerMethodReturnValueHandler
        invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        // 设置 WebDataBinderFactory
        invocableMethod.setDataBinderFactory(binderFactory);
        // 设置 参数名解析器
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);

        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        // 获取 HttpServletRequest 中存储的 FlashMap
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        // 这里是激活 @ModelAttribute, @InitBinder 方法, 并将返回值放入 ModelAndViewContainer
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
        // 将 HttpServletRequest 转换成方法的参数, 激活方法, 最后 通过 HandlerMethodReturnValueHandler 来处理返回值
        invocableMethod.invokeAndHandle(webRequest, mavContainer);      
        // 生成 ModelAndView
        return getModelAndView(mavContainer, modelFactory, webRequest); 
    }
    finally {
        webRequest.requestCompleted(); // 标志请求已经结束, 进行一些生命周期回调函数的激活
    }
}
```



在上面的代码中我们创建了一个ServletInvocableHandlerMethod对象，在这个对象中设置了参数解析器、返回值处理器、数据校验工厂类等。接着我们进入到invocableMethod.invokeAndHandle这个方法中看一下：

```
    public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {

        Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
        ......
    }
```

在invokeForRequest这个方法中，主要干了两件事，一是解析请求参数，二是调用Controller中的请求方法。这里我们主要关注的是参数解析的部分：



```
    public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {

        //解析请求参数
        Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
        if (logger.isTraceEnabled()) {
            logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
                    "' with arguments " + Arrays.toString(args));
        }
        //调用Controller中的请求方法
        Object returnValue = doInvoke(args);
        if (logger.isTraceEnabled()) {
            logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
                    "] returned [" + returnValue + "]");
        }
        return returnValue;
    }
```



进入getMethodArgumentValues：



```
    private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
            Object... providedArgs) throws Exception {
        //这里的 getMethodParameters 其实是在构造 MethodParameters 时创建的
        MethodParameter[] parameters = getMethodParameters();
        Object[] args = new Object[parameters.length];
        //从这里开始对参数进行一个一个解析 <- 主要是通过 HandlerMethodArgumentResolver
        for (int i = 0; i < parameters.length; i++) {
            MethodParameter parameter = parameters[i];
            parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
            //如果之前有预先设置值的话，则取预先设置好的值
            args[i] = resolveProvidedArgument(parameter, providedArgs);
            if (args[i] != null) {
                continue;
            }
            //获取能解析出方法参数值的参数解析器类
            if (this.argumentResolvers.supportsParameter(parameter)) {
                try {
                    //通过 argumentResolvers 解析 HandlerMethod 里面对应的参数内容
                    args[i] = this.argumentResolvers.resolveArgument(
                            parameter, mavContainer, request, this.dataBinderFactory);
                    continue;
                }
                catch (Exception ex) {
                    if (logger.isDebugEnabled()) {
                        logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
                    }
                    throw ex;
                }
            }
            //如果没有能解析方法参数的类，抛出异常
            if (args[i] == null) {
                throw new IllegalStateException("Could not resolve method parameter at index " +
                        parameter.getParameterIndex() + " in " + parameter.getMethod().toGenericString() +
                        ": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
            }
        }
        return args;
    }
```



这里需要说的是argumentResolvers这个对象是HandlerMethodArgumentResolverComposite这个类。所有参数的解析都是委托这个类来完成的，这个类会调用真正的请求参数的解析的类：

```
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        return (getArgumentResolver(parameter) != null);
    }
```



```
    private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
        //先看之前有没有解析过这个方法参数，如果解析过，则从缓存中取
        HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
        if (result == null) {
            //循环所有的参数解析类，匹配真正参数解析的类
            for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
                if (logger.isTraceEnabled()) {
                    logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
                            parameter.getGenericParameterType() + "]");
                }
                if (methodArgumentResolver.supportsParameter(parameter)) {
                    result = methodArgumentResolver;
                    //放到缓存中
                    this.argumentResolverCache.put(parameter, result);
                    break;
                }
            }
        }
        return result;
    }
```



下一篇分析具体从URL参数中传的值，如何绑定到对应controller参数上面。