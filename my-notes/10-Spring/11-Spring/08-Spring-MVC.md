### 一、SpringMVC

#### 1、请求处理流程初探

![](../img/spring/SpringMVC-请求处理流程.jpg)

> 1> ① DispatcherServlet 是 SpringMVC 中的前端控制器（FrontController），负责接收 Request 并将 Request 转发给对应的处理组件。
> 2> ②、③ HanlerMapping 是 SpringMVC 中完成 URL 到 Controller 映射的组件。DispatcherServlet 接收 Request，然后从 HandlerMapping 查找处理 Request 的Controller。③ 会返回一个执行链 HandlerExceptionChain
> 3> ④ 请求适配器执行 Handler
> 4> ⑤、⑥、⑦ Controller 处理 Request，并返回 ModelAndView 对象。Controller 是 SpringMVC 中负责处理 Request 的组件，ModelAndView 是封装结果视图的组件。
> 4> ⑧、⑨ 是视图解析器解析 ModelAndView 对象并返回对应的视图给客户端的过程。⑧请求进行视图解析并返回一个 View；⑨ 视图渲染，将模型数据Model填充到request域。
> 
> 		容器初始化时会建立所有 URL 和 Controller 中方法的对应关系，保存到 HandlerMapping 中，用户请求时根据请求的 URL 快速定位到 Controller 中的某个方法。在 Spring 中先将 URL 和 Controller 的对应关系保存到 Map＜url，Controller＞中。 Web 容器启动时会通知 Spring 初始化容器（加载 Bean 的定义信息和初始化所有单例 Bean），然后 SpringMVC 会遍历容器中的 Bean，获取每一个 Controller 中的所有方法访问的 URL，将 URL 和 Controller 保存到一个 Map 中。这样就可以根据请求快速定位到 Controller，因为最终处理请求的是 Controller 中的方法，Map 中只保留了 URL 和 Controller 的对应关系，所以要根据请求的URL进一步确认 Controller 中的方法。其原理就是拼接 Controller 的 URL （Controller 上@RequestMapping 的值）和方法的URL（Method 上 @RequestMapping 的值），与请求的 URL 进行匹配，找到匹配的方法。确定处理请求的方法后，接下来的任务就是参数绑定，把请求中的参数绑定到方法的形式参数上，这是整个请求处理过程中最复杂的一步。

#### 2、SpringMVC 详解

![](../img/spring/SpringMVC-时序图.jpg)

##### 2.1、初始化阶段	

> 1> init()：将配置参数映射到这个 servlet 的 bean 属性上，并调用子类初始化。[加载配置信息，初始化 BeanWrapper]。
>
> ​		BeanWrapper 是对 Bean 的包装，其接口中定义的功能很简单，包括设置获取被包装的对象，获取被包装 Bean 的属性描述器。通过 BeanWrapper，Spring IoC 容器可以用统一的方式来访问 Bean 属性。
>
> 2> initServletBean()：真正完成容器的初始化动作。默认实现为空，通过子类重写来完成自定义初始化。在调用此方法之前，该 servlet 的所有 bean 属性都将被设置。
>
> 3> initWebApplicationContext()：初始化并发布这个 Servlet 的 WebApplicationContext。将 Servlet 上下文与 Spring 容器上下文关联。
>
> ​		ServletContext：通过自定义 contextListener 获取 web.xml 中配置的参数，
>
> 4> onRefresh()：
>
> 5> initStrategies()：初始化 SpringMVC 的九大组件

##### 2.2、调用阶段

> 
>
> 

### 二、九大组件

> 1. HandlerMappings
>
>    URI 跟 Handler(具体处理方法) 建立的一对一的对应关系。标注了 @RequestMapping 的每个方法都可以看成一个 Handler。Handler 负责实际的请求处理，在请求到达后， HandlerMapping 的作用便是找到请求相应的处理器 Handler 和 Interceptor。  
>
> 2. HandlerAdapters
>
>    参数适配器，将 Servlet 中 Request 中的参数转换成 Handler 的参数
>
> 3. HandlerExceptionResolvers
>
>    异常拦截器 处理异常产生之后渲染(不同的异常对应不同的页面，如 404)的规则的转化器，解耦
>
> 4. ViewResolvers
>
>    视图解析器，将 String 类型的视图名解析为 View 类型的视图。
>
> 5. RequestToViewNameTranslator
>
>    视图预处理器，在处理器返回的 View 为空时使用这个接口的实现，获得 viewName。
>
> 6. LocaleResolver
>
>    语言解析器，根据 Request 获取浏览器的本地语言设置，实现国际化
>
> 7. ThemeResolver
>
>    主题解析器，实现换肤
>
> 8. MultipartResolver
>
>    多文件上传处理
>
> 9. FlashMapManager
>
>    处理重定向时的参数中转，避免重定向时参数携带到 URL。

