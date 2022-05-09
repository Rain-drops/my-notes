# [HandlerAdapter解析参数过程之HandlerMethodArgumentResolver](https://www.cnblogs.com/grasp/p/11111873.html)

​		在我们做Web开发的时候，会提交各种数据格式的请求，而我们的后台也会有相应的参数处理方式。SpringMVC就为我们提供了一系列的参数解析器，不管你是要获取Cookie中的值，Header中的值，JSON格式的数据，URI中的值。下面我们分析几个SpringMVC为我们提供的参数解析器。

​    *在SpringMVC中为我们定义了一个参数解析的顶级父类：HandlerMethodArgumentResolver。同时SpringMVC为我们提供了这么多的实现类：*

![img](https://img2018.cnblogs.com/blog/738818/201906/738818-20190630231529954-1773941615.png)

​		这么多的类，看起来眼花缭乱的。下面选择几个常用的参数解析的类型来分析一下。在这之前先说一下HandlerMethodArgumentResolverComposite这个类，这个类是SpringMVC参数解析器的一个集合，在RequestMappingHandlerAdapter中就定义了相关变量来处理参数。

​		接到上篇讲的RequestMappingHandlerAdapter和RequestParam原理分析到getMethodArgumentValues方法中。

​		我们先来看这样的一个请求：http://localhost:8080/testindex?id=2&userName=%E6%B5%8B%E8%AF%95

​		后端的代码如下：

```java
    @RequestMapping("testindex")
    public String testIndex(Long id, String userName) {
        System.out.println(String.format("id:%d,userName:%s", id, userName));
        return "/jsp/index";
    }
```

​		这样的写法相信大家都很熟悉，那么SpringMVC是怎么解析出请求中的参数给InvocableHandlerMethod#doInvoke方法当入参的呢？经过我们debug发现，这里methodArgumentResolver.supportsParameter所匹配到的HandlerMethodArgumentResolver的实现类是RequestParamMethodArgumentResolver。我们进入到RequestParamMethodArgumentResolver中看一下supportsParameter方法：

```java
    @Override
    public boolean supportsParameter(MethodParameter parameter) {
        //方法的参数中是否有RequestParam注解。
        if (parameter.hasParameterAnnotation(RequestParam.class)) {
            //方法的参数是否是Map
            if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
                String paramName = parameter.getParameterAnnotation(RequestParam.class).name();
                return StringUtils.hasText(paramName);
            }
            else {
                return true;
            }
        }
        else {
            //如果有RequestPart注解，直接返回faslse
            if (parameter.hasParameterAnnotation(RequestPart.class)) {
                return false;
            }
            parameter = parameter.nestedIfOptional();
            //是否是文件上传中的值
            if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
                return true;
            }
            else if (this.useDefaultResolution) {
                //默认useDefaultResolution为true
                return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
            }
            else {
                return false;
            }
        }
    }
```

​		在上面的代码中我们可以看到，请求对应的处理方法的中的参数如果带有RequestParam注解，则判断是不是Map类型的参数，如果不是，则直接返回true。如果没有RequestParam注解，则判断如果有RequestPart注解，则直接返回false，接着判断是否是文件上传表单中的参数值，如果不是，则接着判断useDefaultResolution是否为true。这里需要说明一下的是：argumentResolvers中有两个RequestParamMethodArgumentResolver bean，一个useDefaultResolution为false，一个useDefaultResolution为true。当useDefaultResolution为false的bean是用来处理RequestParam注解的，useDefaultResolution为true的bean是用来处理简单类型的bean的。在我们这个例子中，useDefaultResolution的值为true。那么接下来回判断是不是简单类型参数，我们进到BeanUtils.isSimpleProperty这个方法中看一下：

```java
    public static boolean isSimpleProperty(Class<?> clazz) {
        Assert.notNull(clazz, "Class must not be null");
        return isSimpleValueType(clazz) || (clazz.isArray() && isSimpleValueType(clazz.getComponentType()));
    }
```

```java
    public static boolean isSimpleValueType(Class<?> clazz) {
        return (ClassUtils.isPrimitiveOrWrapper(clazz) || clazz.isEnum() ||
                CharSequence.class.isAssignableFrom(clazz) ||
                Number.class.isAssignableFrom(clazz) ||
                Date.class.isAssignableFrom(clazz) ||
                URI.class == clazz || URL.class == clazz ||
                Locale.class == clazz || Class.class == clazz);
    }
```

​		真正进行类型判断的方法是isSimpleValueType这个方法，如果请求对应处理类的方法的参数为枚举类型、String类型、Long、Integer、Float、Byte、Short、Double、Date、URI、URL、Locale、Class、文件上传对象或者参数是数组，数组类型为上面列出的类型则返回true。即我们的请求对应处理类的方法的参数为：枚举类型、String类型、Long、Integer、Float、Byte、Short、Double、Date、URI、URL、Locale、Class、文件上传对象或者参数是数组，数组类型为上面列出的类型，则请求参数处理类为：RequestParamMethodArgumentResolver。我们先看一下RequestParamMethodArgumentResolver的UML类图关系：
![img](https://img2018.cnblogs.com/blog/738818/201906/738818-20190630233040794-1974974072.png)

接着我们看一下resolveArgument的这个方法，我们在RequestParamMethodArgumentResolver这个方法中没有找到对应的resolveArgument方法，但是我们在他的父类中找到了resolveArgument这个方法源码如下：

```java
    @Override
    public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
        //获取请求对应处理方法的参数字段值
        NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
        MethodParameter nestedParameter = parameter.nestedIfOptional();
        //解析之后的请求对应处理方法的参数字段值
        Object resolvedName = resolveStringValue(namedValueInfo.name);
        if (resolvedName == null) {
            throw new IllegalArgumentException(
                    "Specified name must not resolve to null: [" + namedValueInfo.name + "]");
        }
        //解析参数值
        Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
        //如果从请求中得到的参数值为null的话
        if (arg == null) {
            //判断是否有默认值
            if (namedValueInfo.defaultValue != null) {
                arg = resolveStringValue(namedValueInfo.defaultValue);
            }
            //判断这个字段是否是必填，RequestParam注解默认为必填。
            else if (namedValueInfo.required && !nestedParameter.isOptional()) {
                handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
            }
            //处理null值
            arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
        }
        //如果为得到的参数值为null，且有默认的值
        else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
            arg = resolveStringValue(namedValueInfo.defaultValue);
        }
        //参数的校验
        if (binderFactory != null) {
            WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
            try {
                arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
            }
            catch (ConversionNotSupportedException ex) {
                throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
                        namedValueInfo.name, parameter, ex.getCause());
            }
            catch (TypeMismatchException ex) {
                throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
                        namedValueInfo.name, parameter, ex.getCause());
            }
        }
        //空实现
        handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);
        return arg;
    }
```

​		在这个方法中，首先获取到参数名字是什么，这里封装为了NamedValueInfo的一个对象，我们可以去getNamedValueInfo这个方法中看一下：

```java
    private NamedValueInfo getNamedValueInfo(MethodParameter parameter) {
        //先从缓存中获取
        NamedValueInfo namedValueInfo = this.namedValueInfoCache.get(parameter);
        if (namedValueInfo == null) {
            //创建NamedValueInfo对象
            namedValueInfo = createNamedValueInfo(parameter);
            //更新刚才得到的NamedValueInfo对象
            namedValueInfo = updateNamedValueInfo(parameter, namedValueInfo);
            //放入缓存中
            this.namedValueInfoCache.put(parameter, namedValueInfo);
        }
        return namedValueInfo;
    }
```

​		我们看一下createNamedValueInfo这个方法，这个方法在RequestParamMethodArgumentResolver中：

```java
    @Override
    protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
        //判断是否有RequestParam注解
        RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
        return (ann != null ? new RequestParamNamedValueInfo(ann) : new RequestParamNamedValueInfo());
    }
```

​		RequestParamNamedValueInfo对象的源码如下：

```java
    private static class RequestParamNamedValueInfo extends NamedValueInfo {
        public RequestParamNamedValueInfo() {
            super("", false, ValueConstants.DEFAULT_NONE);
        }
        public RequestParamNamedValueInfo(RequestParam annotation) {
            super(annotation.name(), annotation.required(), annotation.defaultValue());
        }
    }
```

​		这两个构造函数的区别是，如果有RequestParam注解的话，则取ReuqestParam注解中的值，否则取默认的值，我们看一下updateNamedValueInfo这个方法的源码：

```java
    private NamedValueInfo updateNamedValueInfo(MethodParameter parameter, NamedValueInfo info) {
        String name = info.name;
        //如果上一步创建的NamedValueInfo中的name为空的话
        if (info.name.isEmpty()) {
            //从MethodParameter中解析出参数的名字
            name = parameter.getParameterName();
            if (name == null) {
                throw new IllegalArgumentException(
                        "Name for argument type [" + parameter.getNestedParameterType().getName() +
                        "] not available, and parameter name information not found in class file either.");
            }
        }
        //转换默认值
        String defaultValue = (ValueConstants.DEFAULT_NONE.equals(info.defaultValue) ? null : info.defaultValue);
        return new NamedValueInfo(name, info.required, defaultValue);
    }
```

 		这个方法的主要作用是获取参数的名字。学过反射的我们都知道通过反射的API只能获取方法的形参的类型，不能获取形参的名称，但是这里很明显我们需要获取到形参的名称，所以这里获取形参的名称不是通过反射的方式获取，而是通过了一个叫ASM的技术来实现的。当我们获取到NamedValueInfo 之后，会对获取到的形成做进一步的处理，这里我们可以先不用关注，直接到resolveName这个方法中看一下：

```java
    @Override
    protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
        //获取HttpServletRequest
        HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
        //判断是不是文件上传的请求
        MultipartHttpServletRequest multipartRequest =
                WebUtils.getNativeRequest(servletRequest, MultipartHttpServletRequest.class);
        //先从文件上传的请求中获取上传的文件对象(Part这种东西现在应该很少用了吧，所以这里就直接忽略了)
        Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
        if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
            return mpArg;
        }

        Object arg = null;
        //如果是文件上传请求，
        if (multipartRequest != null) {
            //则获取文件上传请求中的普通表单项的值
            List<MultipartFile> files = multipartRequest.getFiles(name);
            if (!files.isEmpty()) {
                //如果只有一个值的话，则返回一个值，否则返回数组
                arg = (files.size() == 1 ? files.get(0) : files);
            }
        }
        //说明是普通的请求
        if (arg == null) {
            //则从request.getParameterValues中获取值
            String[] paramValues = request.getParameterValues(name);
            if (paramValues != null) {
                //如果只有一个值的话，则返回一个值，否则返回数组
                arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
            }
        }
        return arg;
    }
```

​		这里同时支持了获取文件上传对象、文件上传请求中的表单项参数值的获取，普通请求参数值的获取。对于普通请求参数值的获取是通过request.getParameterValues来获取的。

​		如果我们没有从请求中获取到参数值的话，则先判断是否是有默认值(用RequestParam注解可以设置默认值)，接着判断这个参数是否是必要的参数(RequestParam默认为必要参数)，如果是必要的参数且这个值为null的话，则处理过程如下：

```java
    @Override
    protected void handleMissingValue(String name, MethodParameter parameter, NativeWebRequest request)
            throws Exception {

        HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);
        if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
            if (!MultipartResolutionDelegate.isMultipartRequest(servletRequest)) {
                throw new MultipartException("Current request is not a multipart request");
            }
            else {
                throw new MissingServletRequestPartException(name);
            }
        }
        else {
            throw new MissingServletRequestParameterException(name,
                    parameter.getNestedParameterType().getSimpleName());
        }
    }
```

​		如果是文件上传请求，则异常信息为：Current request is not a multipart request。普通请求，则异常信息为："Required " + this.parameterType + " parameter '" + this.parameterName + "' is not present"。如果值为null的话，则会对null值进行处理，

```java
    private Object handleNullValue(String name, Object value, Class<?> paramType) {
        if (value == null) {
            if (Boolean.TYPE.equals(paramType)) {
                return Boolean.FALSE;
            }
            else if (paramType.isPrimitive()) {  //int long 等
                throw new IllegalStateException("Optional " + paramType.getSimpleName() + " parameter '" + name +
                        "' is present but cannot be translated into a null value due to being declared as a " +
                        "primitive type. Consider declaring it as object wrapper for the corresponding primitive type.");
            }
        }
        return value;
    }
```

​		如果参数类型为int、long等基本类型，则如果请求参数值为null的话，则会抛出异常，异常信息如下："Optional " + paramType.getSimpleName() + " parameter '" + name +' is present but cannot be translated into a null value due to being declared as a primitive type. Consider declaring it as object wrapper for the corresponding primitive type."。
​		我们在上一步获取到的参数，会在下一步进行数据校验。

​		如果我们的请求换成这个：http://localhost:8080/allRequestFormat/requestParamRequest?id=122
​		后台处理代码如下：

```java
    @RequestMapping("requestParamRequest")
    public String requestParamRequest(@RequestParam("id") Long id) {
        System.out.println("参数ID为：" + id);
        return "这是一个带RequestParam注解的请求";
    }
```

​		这个处理过程，和我们上面说的处理过程基本是一致的。
​		这里再多说一些RequestParam这个注解，使用RequestParam注解的好处是，可以指定所要的请求参数的名称，缩短处理过程，可以指定参数默认值。
​		另外再多说一点：java类文件编译为class文件时，有release和debug模式之分，在命令行中直接使用javac进行编译的时候，默认的是release模式，使用release模式会改变形参中的参数名，如果形成的名称变化的话，我们可能不能正确的获取到请求参数中的值。而IDE都是使用debug模式进行编译的。ant编译的时候，需要在ant的配置文件中指定debug="true"。 如果要修改javac编译类文件的方式的话，需要指定-g参数。即：javac -g 类文件。
​		我们最后再总结一下：如果你的请求对应处理类中的形参类型为：枚举类型、String类型、Long、Integer、Float、Byte、Short、Double、Date、URI、URL、Locale、Class、文件上传对象或者参数是数组，数组类型为上面列出的类型的话，则会使用RequestParamMethodArgumentResolver进行参数解析。