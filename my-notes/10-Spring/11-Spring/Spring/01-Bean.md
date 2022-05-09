

# [bean 解析、注册、实例化流程源码剖析](https://www.cnblogs.com/grasp/p/11074770.html)

本spring源码的版本：4.3.7

Spring bean的加载主要分为以下6步：

- （1）读取XML配置文件
- （2）XML文件解析为document文档
- （3）解析bean
- （4）注册bean
- （5）实例化bean
- （6）获取bean

### 1 读取XML配置文件

查看源码第一步是找到程序入口，再以入口为突破口，一步步进行源码跟踪。

Java Web应用中的入口就是web.xml。

在web.xml找到ContextLoaderListener ，此Listener负责初始化Spring IOC。

contextConfigLocation参数设置了bean定义文件地址。

```xml
	<!-- 1.指定Spring的配置文件 -->
    <!-- 否则Spring会默认从WEB-INF下寻找配置文件,contextConfigLocation属性是Spring内部固定的 -->
    <!-- 通过ContextLoaderListener的父类ContextLoader中发现CONFIG_LOCATION_PARAM固定为contextConfigLocation -->
	<!-- Sping应用上下文参数：声明应用范围内的初始化参数. context-param元素声明应用范围内的初始化参数. -->
    <context-param>
        <!-- 环境配置位置 -->
        <param-name>contextConfigLocation</param-name>
        <param-value>classpath:spring/spring-context.xml</param-value>
    </context-param>
    <!-- 2.实例化Spring容器 -->
    <!-- 应用启动时,该监听器被执行,它会读取Spring相关配置文件,其默认会到WEB-INF中查找applicationContext.xml -->
    <!-- Spring监听器:ContextLoaderListener的作用就是启动Web容器时,自动装配ApplicationContext的配置信息. -->
    <listener>
        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
    </listener>
```

ContextLoaderListener作用就是负责启动和关闭Spring root WebApplicationContext。

具体WebApplicationContext是什么？开始看源码。

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
    public ContextLoaderListener() {
    }
    public ContextLoaderListener(WebApplicationContext context) {
        super(context);
    }
    /**
     * Initialize the root web application context.
     */
    @Override
    public void contextInitialized(ServletContextEvent event) {
        initWebApplicationContext(event.getServletContext());
    }
    /**
     * Close the root web application context.
     */
    @Override
    public void contextDestroyed(ServletContextEvent event) {
        closeWebApplicationContext(event.getServletContext());
        ContextCleanupListener.cleanupAttributes(event.getServletContext());
    }
}
```

从源码看出此Listener主要有两个函数，一个负责初始化WebApplicationContext，一个负责销毁。

继续看initWebApplicationContext函数。

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
        //初始化Spring容器时如果发现servlet 容器中已存在根Spring容根器则抛出异常，证明rootWebApplicationContext只能有一个。
        if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
            throw new IllegalStateException(
                    "Cannot initialize context because there is already a root application context present - " +
                    "check whether you have multiple ContextLoader* definitions in your web.xml!");
        }

        Log logger = LogFactory.getLog(ContextLoader.class);
        servletContext.log("Initializing Spring root WebApplicationContext");
        if (logger.isInfoEnabled()) {
            logger.info("Root WebApplicationContext: initialization started");
        }
        long startTime = System.currentTimeMillis();

        try {
            // Store context in local instance variable, to guarantee that
            // it is available on ServletContext shutdown.
            if (this.context == null) {
                //1.创建webApplicationContext实例
                this.context = createWebApplicationContext(servletContext);
            }
            if (this.context instanceof ConfigurableWebApplicationContext) {
                ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
                if (!cwac.isActive()) {
                    // The context has not yet been refreshed -> provide services such as
                    // setting the parent context, setting the application context id, etc
                    if (cwac.getParent() == null) {
                        // The context instance was injected without an explicit parent ->
                        // determine parent for root web application context, if any.
                        ApplicationContext parent = loadParentContext(servletContext);
                        cwac.setParent(parent);
                    }
                    //2.配置WebApplicationContext
                    configureAndRefreshWebApplicationContext(cwac, servletContext);
                }
            }
            //3.把生成的webApplicationContext 设置为root webApplicationContext。
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

            ClassLoader ccl = Thread.currentThread().getContextClassLoader();
            if (ccl == ContextLoader.class.getClassLoader()) {
                currentContext = this.context;
            }
            else if (ccl != null) {
                currentContextPerThread.put(ccl, this.context);
            }

            if (logger.isDebugEnabled()) {
                logger.debug("Published root WebApplicationContext as ServletContext attribute with name [" +
                        WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE + "]");
            }
            if (logger.isInfoEnabled()) {
                long elapsedTime = System.currentTimeMillis() - startTime;
                logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
            }

            return this.context;
        }
        catch (RuntimeException ex) {
            logger.error("Context initialization failed", ex);
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
            throw ex;
        }
        catch (Error err) {
            logger.error("Context initialization failed", err);
            servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, err);
            throw err;
        }
    }
```

在上面的代码中主要有两个功能：

- （1）创建WebApplicationContext实例。
- （2）配置生成WebApplicationContext实例。
- （3）把生成的webApplicationContext 设置为root webApplicationContext。

#### 1.1 创建WebApplicationContext实例

进入CreateWebAPPlicationContext函数

```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
        //得到ContextClass类,默认实例化的是XmlWebApplicationContext类
        Class<?> contextClass = determineContextClass(sc);
        if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
            throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                    "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
        }
        //实例化Context类
        return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
    }
```

进入determineContextClass函数。

```java
protected Class<?> determineContextClass(ServletContext servletContext) {
        String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
        if (contextClassName != null) {
            //若设置了contextClass则使用定义好的ContextClass。
            try {
                return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load custom context class [" + contextClassName + "]", ex);
            }
        }
        else {
            //此处获取的是在Spring源码中ContextLoader.properties中配置的org.springframework.web.context.support.XmlWebApplicationContext类。
            contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
            try {
                return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
            }
            catch (ClassNotFoundException ex) {
                throw new ApplicationContextException(
                        "Failed to load default context class [" + contextClassName + "]", ex);
            }
        }
    }
```

#### 1.2 配置Web ApplicationContext

进入configureAndReFreshWebApplicaitonContext函数。

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
        if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
            // The application context id is still set to its original default value
            // -> assign a more useful id based on available information
            String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
            if (idParam != null) {
                wac.setId(idParam);
            }
            else {
                // Generate default id...
                wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                        ObjectUtils.getDisplayString(sc.getContextPath()));
            }
        }
        //webapplicationContext设置servletContext.
        wac.setServletContext(sc);
        //此处CONFIG_LOCATION_PARAM = "contextConfigLocation"，即读即取web.xm中配设置的contextConfigLocation参数值，获得spring bean的配置文件.
        String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
        if (configLocationParam != null) {
            //webApplicationContext设置配置文件路径设。
            wac.setConfigLocation(configLocationParam);
        }

        // The wac environment's #initPropertySources will be called in any case when the context
        // is refreshed; do it eagerly here to ensure servlet property sources are in place for
        // use in any post-processing or initialization that occurs below prior to #refresh
        ConfigurableEnvironment env = wac.getEnvironment();
        if (env instanceof ConfigurableWebEnvironment) {
            ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
        }

        customizeContext(sc, wac);
        //开始处理bean
        wac.refresh();
    }
```

### 2 解析XML文件

上面wac变量声明为ConfigurableWebApplicationContext类型，ConfigurableWebApplicationContext又继承了WebApplicationContext。

WebApplication Context有很多实现类。 但从上面determineContextClass得知此处wac实际上是XmlWebApplicationContext类，因此进入XmlWebApplication类查看其继承的refresh()方法。

沿方法调用栈一层层看下去。

最终跟踪到实现的地方是AbstractApplicationContext的refresh()方法

```java
public void refresh() throws BeansException, IllegalStateException {
        synchronized (this.startupShutdownMonitor) {
            // Prepare this context for refreshing.
            prepareRefresh();

            // Tell the subclass to refresh the internal bean factory.
            //获取beanFactory
            ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

            // Prepare the bean factory for use in this context.
            prepareBeanFactory(beanFactory);

            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                // Invoke factory processors registered as beans in the context.
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);

                // Initialize message source for this context.
                initMessageSource();

                // Initialize event multicaster for this context.
                initApplicationEventMulticaster();

                // Initialize other special beans in specific context subclasses.
                onRefresh();

                // Check for listener beans and register them.
                registerListeners();

                // Instantiate all remaining (non-lazy-init) singletons.
                // 实例化所有声明为非懒加载的单例bean 
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            }

            catch (BeansException ex) {
                if (logger.isWarnEnabled()) {
                    logger.warn("Exception encountered during context initialization - " +
                            "cancelling refresh attempt: " + ex);
                }

                // Destroy already created singletons to avoid dangling resources.
                destroyBeans();

                // Reset 'active' flag.
                cancelRefresh(ex);

                // Propagate exception to caller.
                throw ex;
            }

            finally {
                // Reset common introspection caches in Spring's core, since we
                // might not ever need metadata for singleton beans anymore...
                resetCommonCaches();
            }
        }
    }
```

获取beanFactory。

```java
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
        refreshBeanFactory();
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        if (logger.isDebugEnabled()) {
            logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
        }
        return beanFactory;
    }
```

beanFactory初始化。在AbstractRefreshableApplicationContext中实现

```java
protected final void refreshBeanFactory() throws BeansException {
        if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
        }
        try {
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());
            customizeBeanFactory(beanFactory);
            //加载bean定义
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
        }
    }
```

加载bean。在XmlWebApplicationContext中实现

```java
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // Create a new XmlBeanDefinitionReader for the given BeanFactory.
        //创建XmlBeanDefinitionReader实例来解析XML配置文件
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

        // Configure the bean definition reader with this context's
        // resource loading environment.
        beanDefinitionReader.setEnvironment(getEnvironment());
        beanDefinitionReader.setResourceLoader(this);
        beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

        // Allow a subclass to provide custom initialization of the reader,
        // then proceed with actually loading the bean definitions.
        initBeanDefinitionReader(beanDefinitionReader);
        //解析XML配置文件中的bean。
        loadBeanDefinitions(beanDefinitionReader);
    }
```

读取XML配置文件。

```java
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
        //此处读取的就是之前设置好的web.xml中配置文件地址
        String[] configLocations = getConfigLocations();
        if (configLocations != null) {
            for (String configLocation : configLocations) {
                //调用XmlBeanDefinitionReader读取XML配置文件
                reader.loadBeanDefinitions(configLocation);
            }
        }
    }
```

AbstractBeanDefinitionReader读取XML文件中的bean定义

```java
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
        ResourceLoader resourceLoader = getResourceLoader();
        if (resourceLoader == null) {
            throw new BeanDefinitionStoreException(
                    "Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
        }

        if (resourceLoader instanceof ResourcePatternResolver) {
            // Resource pattern matching available.
            try {
                Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
                int loadCount = loadBeanDefinitions(resources);
                if (actualResources != null) {
                    for (Resource resource : resources) {
                        actualResources.add(resource);
                    }
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
                }
                return loadCount;
            }
            catch (IOException ex) {
                throw new BeanDefinitionStoreException(
                        "Could not resolve bean definition resource pattern [" + location + "]", ex);
            }
        }
        else {
            // Can only load single resources by absolute URL.
            Resource resource = resourceLoader.getResource(location);
            //加载bean
            int loadCount = loadBeanDefinitions(resource);
            if (actualResources != null) {
                actualResources.add(resource);
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
            }
            return loadCount;
        }
    }
```

继续查看loadBeanDefinitons函数调用栈，进入到XmlBeanDefinitioReader类的loadBeanDefinitions方法。

```java
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        Assert.notNull(encodedResource, "EncodedResource must not be null");
        if (logger.isInfoEnabled()) {
            logger.info("Loading XML bean definitions from " + encodedResource.getResource());
        }

        Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        if (currentResources == null) {
            currentResources = new HashSet<EncodedResource>(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }
        if (!currentResources.add(encodedResource)) {
            throw new BeanDefinitionStoreException(
                    "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        }
        try {
             //获取文件流
            InputStream inputStream = encodedResource.getResource().getInputStream();
            try {
                InputSource inputSource = new InputSource(inputStream);
                if (encodedResource.getEncoding() != null) {
                    inputSource.setEncoding(encodedResource.getEncoding());
                }
                 //从文件流中加载定义好的bean。
                return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
            }
            finally {
                inputStream.close();
            }
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "IOException parsing XML document from " + encodedResource.getResource(), ex);
        }
        finally {
            currentResources.remove(encodedResource);
            if (currentResources.isEmpty()) {
                this.resourcesCurrentlyBeingLoaded.remove();
            }
        }
    }
```

最终将XML文件解析成Document文档对象。

```java
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {
        try {
            //XML配置文件解析到Document实例中
            Document doc = doLoadDocument(inputSource, resource);
            //注册bean
            return registerBeanDefinitions(doc, resource);
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (SAXParseException ex) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                    "Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
        }
        catch (SAXException ex) {
            throw new XmlBeanDefinitionStoreException(resource.getDescription(),
                    "XML document from " + resource + " is invalid", ex);
        }
        catch (ParserConfigurationException ex) {
            throw new BeanDefinitionStoreException(resource.getDescription(),
                    "Parser configuration exception parsing XML from " + resource, ex);
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(resource.getDescription(),
                    "IOException parsing XML document from " + resource, ex);
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(resource.getDescription(),
                    "Unexpected exception parsing XML document from " + resource, ex);
        }
    }
```

### 3 解析bean

上一步完成了XML文件的解析工作，接下来将XML中定义的bean注册到webApplicationContext，继续跟踪函数。

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        int countBefore = getRegistry().getBeanDefinitionCount();
        //使用documentRedder实例读取bean定义
        documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        return getRegistry().getBeanDefinitionCount() - countBefore;
    }
```

在DefaultBeanDefinitionDocumentReader中，用BeanDefinitionDocumentReader对象来注册bean。

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
        this.readerContext = readerContext;
        logger.debug("Loading bean definitions");
        //读取document元素
        Element root = doc.getDocumentElement();
        //真正开始注册bean
        doRegisterBeanDefinitions(root);
    }
```

解析XML文档。

```java
protected void doRegisterBeanDefinitions(Element root) {
        // Any nested <beans> elements will cause recursion in this method. In
        // order to propagate and preserve <beans> default-* attributes correctly,
        // keep track of the current (parent) delegate, which may be null. Create
        // the new (child) delegate with a reference to the parent for fallback purposes,
        // then ultimately reset this.delegate back to its original (parent) reference.
        // this behavior emulates a stack of delegates without actually necessitating one.
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);

        if (this.delegate.isDefaultNamespace(root)) {
            String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                        profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
                if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (logger.isInfoEnabled()) {
                        logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                    }
                    return;
                }
            }
        }

        //预处理XML 
        preProcessXml(root);
        //解析注册bean
        parseBeanDefinitions(root, this.delegate);
        postProcessXml(root);

        this.delegate = parent;
    }
```

在DefaultBeanDefinitionDocumentReader中，循环解析XML文档中的每个元素。

```java
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

下面是默认命名空间的解析逻辑。

不明白Spring的命名空间的可以网上查一下，其实类似于package，用来区分变量来源，防止变量重名。

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        //解析import元素
        if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            importBeanDefinitionResource(ele);
        }
        //解析alias元素
        else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            processAliasRegistration(ele);
        }
        //解析bean元素
        else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            processBeanDefinition(ele, delegate);
        }
         //解析beans元素
        else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            // recurse
            doRegisterBeanDefinitions(ele);
        }
    }
```

这里我们就不一一跟踪，以解析bean元素为例继续展开。

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        //解析bean
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // Register the final decorated instance.
                //注册bean
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // Send registration event.
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }
```

解析bean元素，最后把每个bean解析为一个包含bean所有信息的BeanDefinitionHolder对象

### 4 注册bean

接下来将解析到的bean注册到webApplicationContext中。接下继续跟踪registerBeanDefinition函数。

```java
public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        // Register bean definition under primary name.
        // 获取beanname
        String beanName = definitionHolder.getBeanName();
        //注册bean
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

        // 注册bean的别名
        // Register aliases for bean name, if any.
        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
    }
```

跟踪registerBeanDefinition函数，此函数将bean信息保存到到webApplicationContext的beanDefinitionMap变量中，该变量为map类型，保存Spring 容器中所有的bean定义。一般是DefaultListableBeanFactory中实现的。

```java
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
            throws BeanDefinitionStoreException {

        Assert.hasText(beanName, "Bean name must not be empty");
        Assert.notNull(beanDefinition, "BeanDefinition must not be null");

        if (beanDefinition instanceof AbstractBeanDefinition) {
            try {
                ((AbstractBeanDefinition) beanDefinition).validate();
            }
            catch (BeanDefinitionValidationException ex) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Validation of bean definition failed", ex);
            }
        }

        BeanDefinition oldBeanDefinition;

        oldBeanDefinition = this.beanDefinitionMap.get(beanName);
        if (oldBeanDefinition != null) {
            if (!isAllowBeanDefinitionOverriding()) {
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
                        "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
                        "': There is already [" + oldBeanDefinition + "] bound.");
            }
            else if (oldBeanDefinition.getRole() < beanDefinition.getRole()) {
                // e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
                if (this.logger.isWarnEnabled()) {
                    this.logger.warn("Overriding user-defined bean definition for bean '" + beanName +
                            "' with a framework-generated bean definition: replacing [" +
                            oldBeanDefinition + "] with [" + beanDefinition + "]");
                }
            }
            else if (!beanDefinition.equals(oldBeanDefinition)) {
                if (this.logger.isInfoEnabled()) {
                    this.logger.info("Overriding bean definition for bean '" + beanName +
                            "' with a different definition: replacing [" + oldBeanDefinition +
                            "] with [" + beanDefinition + "]");
                }
            }
            else {
                if (this.logger.isDebugEnabled()) {
                    this.logger.debug("Overriding bean definition for bean '" + beanName +
                            "' with an equivalent definition: replacing [" + oldBeanDefinition +
                            "] with [" + beanDefinition + "]");
                }
            }
            this.beanDefinitionMap.put(beanName, beanDefinition);
        }
        else {
            if (hasBeanCreationStarted()) {
                // Cannot modify startup-time collection elements anymore (for stable iteration)
                synchronized (this.beanDefinitionMap) {
                    this.beanDefinitionMap.put(beanName, beanDefinition);
                    List<String> updatedDefinitions = new ArrayList<String>(this.beanDefinitionNames.size() + 1);
                    updatedDefinitions.addAll(this.beanDefinitionNames);
                    updatedDefinitions.add(beanName);
                    this.beanDefinitionNames = updatedDefinitions;
                    if (this.manualSingletonNames.contains(beanName)) {
                        Set<String> updatedSingletons = new LinkedHashSet<String>(this.manualSingletonNames);
                        updatedSingletons.remove(beanName);
                        this.manualSingletonNames = updatedSingletons;
                    }
                }
            }
            else {
                // Still in startup registration phase
                //把bean信息保存到beanDefinitionMap中
                this.beanDefinitionMap.put(beanName, beanDefinition);
                //把beanName 保存到List 类型的beanDefinitionNames属性中
                this.beanDefinitionNames.add(beanName);
                this.manualSingletonNames.remove(beanName);
            }
            this.frozenBeanDefinitionNames = null;
        }

        if (oldBeanDefinition != null || containsSingleton(beanName)) {
            resetBeanDefinition(beanName);
        }
    }
```

实例化bean的时机有两个。

一个是容器启动时候，另一个是真正调用的时候。

如果bean声明为scope=singleton且lazy-init=false，则容器启动时候就实例化该bean（Spring 默认就是此行为）。否则在调用时候再进行实例化。

相信用过Spring的同学们都知道以上概念，但是为什么呢？

继续从源码角度进行分析，回到之前XmlWebApplication的refresh（）方法。

```java
public void refresh() throws BeansException, IllegalStateException {
   synchronized (this.startupShutdownMonitor) {
      //生成beanFactory,
      ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
      // 实例化所有声明为非懒加载的单例bean 
      finishBeanFactoryInitialization(beanFactory);
 }
```

可以看到获得beanFactory后调用了 finishBeanFactoryInitialization()方法，继续跟踪此方法。

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
        // Initialize conversion service for this context.
        if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
                beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
            beanFactory.setConversionService(
                    beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
        }

        // Register a default embedded value resolver if no bean post-processor
        // (such as a PropertyPlaceholderConfigurer bean) registered any before:
        // at this point, primarily for resolution in annotation attribute values.
        if (!beanFactory.hasEmbeddedValueResolver()) {
            beanFactory.addEmbeddedValueResolver(new StringValueResolver() {
                @Override
                public String resolveStringValue(String strVal) {
                    return getEnvironment().resolvePlaceholders(strVal);
                }
            });
        }

        // Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
        String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
        for (String weaverAwareName : weaverAwareNames) {
            getBean(weaverAwareName);
        }

        // Stop using the temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(null);

        // Allow for caching all bean definition metadata, not expecting further changes.
        beanFactory.freezeConfiguration();

        // Instantiate all remaining (non-lazy-init) singletons.
        // 初始化非懒加载的单例bean   
        beanFactory.preInstantiateSingletons();
    }
```

在DefaultListableBeanFactory中实现，预先实例化单例类逻辑。

```java
public void preInstantiateSingletons() throws BeansException {
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Pre-instantiating singletons in " + this);
        }

        // Iterate over a copy to allow for init methods which in turn register new bean definitions.
        // While this may not be part of the regular factory bootstrap, it does otherwise work fine.
        // 获取所有注册的bean
        List<String> beanNames = new ArrayList<String>(this.beanDefinitionNames);

        // Trigger initialization of all non-lazy singleton beans...
        // 遍历bean 
        for (String beanName : beanNames) {
            RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
            //如果bean是单例且非懒加载，则获取实例
            if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
                if (isFactoryBean(beanName)) {
                    final FactoryBean<?> factory = (FactoryBean<?>) getBean(FACTORY_BEAN_PREFIX + beanName);
                    boolean isEagerInit;
                    if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged(new PrivilegedAction<Boolean>() {
                            @Override
                            public Boolean run() {
                                return ((SmartFactoryBean<?>) factory).isEagerInit();
                            }
                        }, getAccessControlContext());
                    }
                    else {
                        isEagerInit = (factory instanceof SmartFactoryBean &&
                                ((SmartFactoryBean<?>) factory).isEagerInit());
                    }
                    if (isEagerInit) {
                        getBean(beanName);
                    }
                }
                else {
                    getBean(beanName);
                }
            }
        }

        // Trigger post-initialization callback for all applicable beans...
        for (String beanName : beanNames) {
            Object singletonInstance = getSingleton(beanName);
            if (singletonInstance instanceof SmartInitializingSingleton) {
                final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
                if (System.getSecurityManager() != null) {
                    AccessController.doPrivileged(new PrivilegedAction<Object>() {
                        @Override
                        public Object run() {
                            smartSingleton.afterSingletonsInstantiated();
                            return null;
                        }
                    }, getAccessControlContext());
                }
                else {
                    smartSingleton.afterSingletonsInstantiated();
                }
            }
        }
    }
```

获取bean。

```java
 public Object getBean(String name) throws BeansException {
        return doGetBean(name, null, null, false);
    }
```

doGetBean中处理的逻辑很多，为了减少干扰，下面只显示了创建bean的函数调用栈。

```java
protected <T> T doGetBean(
      final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
      throws BeansException {
    //创建bean
       createBean(beanName, mbd, args);
}
```

创建bean。

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
        /创建bean。
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);

        return beanInstance;
    }
```

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
      throws BeanCreationException {
      // 实例化bean   
      instanceWrapper = createBeanInstance(beanName, mbd, args);
}
```

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
   //实例化bean
   return instantiateBean(beanName, mbd);
}
```

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
      //调用实例化策略进行实例化
      beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
}
```

判断哪种动态代理方式实例化bean。

```java
public Object instantiate(RootBeanDefinition bd, String beanName, BeanFactory owner) {
        // Don't override the class with CGLIB if no overrides.
        //使用JDK动态代理
        if (bd.getMethodOverrides().isEmpty()) {
            Constructor<?> constructorToUse;
            synchronized (bd.constructorArgumentLock) {
                constructorToUse = (Constructor<?>) bd.resolvedConstructorOrFactoryMethod;
                if (constructorToUse == null) {
                    final Class<?> clazz = bd.getBeanClass();
                    if (clazz.isInterface()) {
                        throw new BeanInstantiationException(clazz, "Specified class is an interface");
                    }
                    try {
                        if (System.getSecurityManager() != null) {
                            constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor<?>>() {
                                @Override
                                public Constructor<?> run() throws Exception {
                                    return clazz.getDeclaredConstructor((Class[]) null);
                                }
                            });
                        }
                        else {
                            constructorToUse =    clazz.getDeclaredConstructor((Class[]) null);
                        }
                        bd.resolvedConstructorOrFactoryMethod = constructorToUse;
                    }
                    catch (Throwable ex) {
                        throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                    }
                }
            }
            return BeanUtils.instantiateClass(constructorToUse);
        }
        else {
            // Must generate CGLIB subclass.
            //使用CGLIB动态代理
            return instantiateWithMethodInjection(bd, beanName, owner);
        }
    }
```

不管哪种方式最终都是通过反射的形式完成了bean的实例化。

```java
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) 
      ReflectionUtils.makeAccessible(ctor);
      return ctor.newInstance(args);
}
```

### 6 获取bean

我们继续回到doGetBean函数，分析获取bean的逻辑。

```java
protected <T> T doGetBean(
            final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
            throws BeansException {
        //获取beanName
        final String beanName = transformedBeanName(name);
        Object bean;

        // Eagerly check singleton cache for manually registered singletons.
        // 先检查该bean是否为单例且容器中是否已经存在例化的单例类
        Object sharedInstance = getSingleton(beanName);
        //如果已存在该bean的单例类
        if (sharedInstance != null && args == null) {
            if (logger.isDebugEnabled()) {
                if (isSingletonCurrentlyInCreation(beanName)) {
                    logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                            "' that is not fully initialized yet - a consequence of a circular reference");
                }
                else {
                    logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
                }
            }
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
        }

        else {
            // Fail if we're already creating this bean instance:
            // We're assumably within a circular reference.
            if (isPrototypeCurrentlyInCreation(beanName)) {
                throw new BeanCurrentlyInCreationException(beanName);
            }

            // Check if bean definition exists in this factory.
            // 获取父BeanFactory
            BeanFactory parentBeanFactory = getParentBeanFactory();
            //先判断该容器中是否注册了此bean，如果有则在该容器实例化bean，否则再到父容器实例化bean
            if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
                // Not found -> check parent.
                String nameToLookup = originalBeanName(name);
                if (args != null) {
                    // Delegation to parent with explicit args.
                    // 如果父容器有该bean，则调用父beanFactory的方法获得该bean
                    return (T) parentBeanFactory.getBean(nameToLookup, args);
                }
                else {
                    // No args -> delegate to standard getBean method.
                    return parentBeanFactory.getBean(nameToLookup, requiredType);
                }
            }

            if (!typeCheckOnly) {
                markBeanAsCreated(beanName);
            }

            try {
                final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
                checkMergedBeanDefinition(mbd, beanName, args);

                // Guarantee initialization of beans that the current bean depends on.
                //如果该bean有依赖bean，先实递归例化依赖bean。
                String[] dependsOn = mbd.getDependsOn();
                if (dependsOn != null) {
                    for (String dep : dependsOn) {
                        if (isDependent(beanName, dep)) {
                            throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                    "Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
                        }
                        registerDependentBean(dep, beanName);
                        getBean(dep);
                    }
                }

                // Create bean instance.
                //如果scope为Singleton执行此逻辑
                if (mbd.isSingleton()) {
                    sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                        @Override
                        public Object getObject() throws BeansException {
                            try {
                                //调用创建bean方法
                                return createBean(beanName, mbd, args);
                            }
                            catch (BeansException ex) {
                                // Explicitly remove instance from singleton cache: It might have been put there
                                // eagerly by the creation process, to allow for circular reference resolution.
                                // Also remove any beans that received a temporary reference to the bean.
                                destroySingleton(beanName);
                                throw ex;
                            }
                        }
                    });
                    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
                }
                //如果scope为Prototype执行此逻辑，每次获取时候都实例化一个bean
                else if (mbd.isPrototype()) {
                    // It's a prototype -> create a new instance.
                    Object prototypeInstance = null;
                    try {
                        beforePrototypeCreation(beanName);
                        prototypeInstance = createBean(beanName, mbd, args);
                    }
                    finally {
                        afterPrototypeCreation(beanName);
                    }
                    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
                }
                 //如果scope为Request,Session,GolbalSession执行此逻辑
                else {
                    String scopeName = mbd.getScope();
                    final Scope scope = this.scopes.get(scopeName);
                    if (scope == null) {
                        throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
                    }
                    try {
                        Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                            @Override
                            public Object getObject() throws BeansException {
                                beforePrototypeCreation(beanName);
                                try {
                                    return createBean(beanName, mbd, args);
                                }
                                finally {
                                    afterPrototypeCreation(beanName);
                                }
                            }
                        });
                        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                    }
                    catch (IllegalStateException ex) {
                        throw new BeanCreationException(beanName,
                                "Scope '" + scopeName + "' is not active for the current thread; consider " +
                                "defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                ex);
                    }
                }
            }
            catch (BeansException ex) {
                cleanupAfterBeanCreationFailure(beanName);
                throw ex;
            }
        }

        // Check if required type matches the type of the actual bean instance.
        if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
            try {
                return getTypeConverter().convertIfNecessary(bean, requiredType);
            }
            catch (TypeMismatchException ex) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Failed to convert bean '" + name + "' to required type '" +
                            ClassUtils.getQualifiedName(requiredType) + "'", ex);
                }
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
        }
        return (T) bean;
    }
```

上面方法中首先调用getSingleton(beanName)方法来获取单例bean，如果获取到则直接返回该bean。方法调用栈如下：

```java
public Object getSingleton(String beanName) {
   return getSingleton(beanName, true);
}
```

```java
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
```

getSingleton方法先从singletonObjects属性中获取bean 对象,如果不为空则返回该对象，否则返回null。

那 singletonObjects保存的是什么？什么时候保存的呢？

回到doGetBean（）函数继续分析。如果singletonObjects没有该bean的对象，进入到创建bean的逻辑。处理逻辑如下：

```java
//获取父beanFactory
BeanFactory parentBeanFactory = getParentBeanFactory();
//如果该容器中没有注册该bean，且父容器不为空，则去父容器中获取bean后返回
if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
      return parentBeanFactory.getBean(nameToLookup, requiredType);
}
```

下面是判断容器中有没有注册bean的逻辑，此处beanDefinitionMap相信大家都不陌生，在注册bean的流程里已经说过所有的bean信息都会保存到该变量中。

```
 public boolean containsBeanDefinition(String beanName) {
        Assert.notNull(beanName, "Bean name must not be null");
        return this.beanDefinitionMap.containsKey(beanName);
    }
```

如果该容器中已经注册过bean，继续往下走。先获取该bean的依赖bean，如果依赖bean，则先递归获取相应的依赖bean。

```
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
    for (String dep : dependsOn) {
         registerDependentBean(dep, beanName);
         getBean(dep);
    }
}   
```

依赖bean创建完成后，接下来就是创建自身bean实例了。

获取bean实例的处理逻辑有三种，即Singleton、Prototype、其它(request、session、global session)，下面一一说明。

#### 6.1 Singleton

```
if (mbd.isSingleton()) {
       sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
          @Override
          public Object getObject() throws BeansException {
            //创建bean回调
            return createBean(beanName, mbd, args); 
       });
      bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

获取单例bean，如果已经有该bean的对象直接返回。如果没有则创建单例bean对象，并添加到容器的singletonObjects Map中，以后直接从singletonObjects直接获取bean。

如果singleton的缓存earlySingletonObjects中,一旦对象最终创建好，此引用信息将删除。

```
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
        //如果singletonObjects中没有该bean
        Object singletonObject = this.singletonObjects.get(beanName);
        if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                singletonObject = this.earlySingletonObjects.get(beanName);
                //如果singleton的缓存earlySingletonObjects中没有该bean,一旦对象最终创建好，此引用信息将删除
                if (singletonObject == null && allowEarlyReference) {
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        //回调参数传进来的ObjectFactory的getObject方法，即调用createBean方法创建bean实例
                        singletonObject = singletonFactory.getObject();
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
        }
        return (singletonObject != NULL_OBJECT ? singletonObject : null);
    }
```

#### 6.2 Prototype

```
else if (mbd.isPrototype()) {
    Object prototypeInstance = null;
    //创建bean
    prototypeInstance = createBean(beanName, mbd, args);
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
```

#### 6.3 request、session、global session

```
else {
    //获取该bean的scope   
    String scopeName = mbd.getScope();
    //获取相应scope
    final Scope scope = this.scopes.get(scopeName);
    //获取相应scope的实例化对象
    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
             @Override
             public Object getObject() throws BeansException {
                   return createBean(beanName, mbd, args);
             }
    });
    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
   }
```

从相应scope获取对象实例。实现的地方AbstractRequestAttributesScope

```
    public Object get(String name, ObjectFactory<?> objectFactory) {
        RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();
        //先从指定scope中获取bean实例，如果没有则新建，如果已经有直接返回
        Object scopedObject = attributes.getAttribute(name, getScope());
        if (scopedObject == null) {
            //回调函数调用createBean创建实例
            scopedObject = objectFactory.getObject();
            //创建实例后保存到相应scope中
            attributes.setAttribute(name, scopedObject, getScope());
        }
        return scopedObject;
    }
```

判断scope，获取实例函数逻辑。

```
public Object getAttribute(String name, int scope) {
        if (scope == SCOPE_REQUEST) {
            if (!isRequestActive()) {
                throw new IllegalStateException(
                        "Cannot ask for request attribute - request is not active anymore!");
            }
            //从request中获取实例
            return this.request.getAttribute(name);
        }
        else {
            HttpSession session = getSession(false);
            if (session != null) {
                try {
                    //从session中获取实例
                    Object value = session.getAttribute(name);
                    if (value != null) {
                        this.sessionAttributesToUpdate.put(name, value);
                    }
                    return value;
                }
                catch (IllegalStateException ex) {
                    // Session invalidated - shouldn't usually happen.
                }
            }
            return null;
        }
    }
```

在相应scope中设置实例函数逻辑。

```
public void setAttribute(String name, Object value, int scope) {
        if (scope == SCOPE_REQUEST) {
            if (!isRequestActive()) {
                throw new IllegalStateException(
                        "Cannot set request attribute - request is not active anymore!");
            }
            this.request.setAttribute(name, value);
        }
        else {
            HttpSession session = getSession(true);
            this.sessionAttributesToUpdate.remove(name);
            session.setAttribute(name, value);
        }
    }
```

以上就是Spring bean从无到有的整个逻辑。