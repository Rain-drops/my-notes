### 1. SpringBoot启动过程

启动流程主要分为三个部分，

第一部分进行 SpringApplication 的初始化模块，配置一些基本的环境变量、资源、构造器、监听器；

第二部分实现了应用具体的启动方案，包括启动流程的监听模块、加载配置环境模块、及核心的创建上下文环境模块；

第三部分是自动化配置模块，该模块作为 SpringBoot 自动配置核心；

![](../img/spring/springboot/springboot启动过程.png)

​		每个SpringBoot程序都有一个主入口，也就是 main 方法，main 里面调用 SpringApplication.run() 启动整个 spring-boot 程序，该方法所在类需要使用 @SpringBootApplication 注解，以及 @ImportResource 注解 (if need)

​		@SpringBootApplication 包括三个注解，功能如下：

​	（1）@EnableAutoConfiguration根据应用所声明的依赖来对Spring框架进行自动配置

​	（2）@SpringBootConfiguration(内部为@Configuration)：被标注的类等于在 spring 的 XML 配置文件中(applicationContext.xml)，装配所有bean事务，提供了一个spring的上下文环境

​	（3）@ComponentScan：组件扫描，可自动发现和装配Bean，默认扫描SpringApplication的run方法里的Booter.class所在的包路径下文件，所以最好将该启动类放到根包路径下