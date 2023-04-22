本文来探究 springboot 是如何启动的，本文使用到的 demo 位于?[gitee/funcy](https://gitee.com/funcy/spring-boot/tree/v2.2.2.RELEASE_learn/spring-boot-project/spring-boot-learn/src/main/java/org/springframework/boot/learn/autoconfigure/demo01).

## 1\. 从?`Demo01Application#main(...)`?开始

springboot 的启动方式非常简单，就一行：

```
@SpringBootApplication
public class Demo01Application {

    public static void main(String[] args) {
        // 这一行就是用来启动springboot的
        SpringApplication.run(Demo01Application.class, args);
    }
}

```

然后我们就进入这个方法，看看它干了什么：

```
public class SpringApplication {
    ...
    // primarySource 就是我们传入的 Demo01Application.class，
    // args 就是 main() 方法的参数
    public static ConfigurableApplicationContext run(
            Class<?> primarySource, String... args) {
        // 将 primarySource 包装成数组，继续调用 run(...) 方法
        return run(new Class<?>[] { primarySource }, args);
    }

    // primarySources 就是我们传入的 Demo01Application.class 包装成的数组，
    // args 就是 main() 方法的参数
    public static ConfigurableApplicationContext run(
            Class<?>[] primarySources, String[] args) {
        // 这里开始干正事了
        return new SpringApplication(primarySources).run(args);
    }
    ...
}

```

通过方法一步步追查下去后，最后到了?`SpringApplication#run(Class<?>[], String[])`?方法，关键代码如下：

```
return new SpringApplication(primarySources).run(args);

```

这块代码需要拆开来看，可以拆成以下两部分：

*   构造方法：`SpringApplication#SpringApplication(Class<?>...)`
*   实例方法：`SpringApplication#run(String...)`

看来，这两个方法就是 springboot 的启动所有流程了，接下来我们就来分析这两个方法。

## 2\. 创建?`SpringApplication`：`SpringApplication#SpringApplication(Class<?>...)`

```
public class SpringApplication {
    public SpringApplication(Class<?>... primarySources) {
        // 继续调用
        this(null, primarySources);
    }

    /**
     * 这里就最终调用的构造方法了
     * resourceLoader 为 null
     * primarySources 为 Demo01Application.class
     */
    @SuppressWarnings({ "unchecked", "rawtypes" })
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        // 1\. 将传入的resourceLoader设置到成员变量，这里的值为null
        this.resourceLoader = resourceLoader;
        Assert.notNull(primarySources, "PrimarySources must not be null");
        // 2\. 将传入的primarySources设置到成员变量，这里的值为 Demo01Application.class
        this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
        // 3\. 当前的 web 应用类型，REACTIVE，NONE，SERVLET
        this.webApplicationType = WebApplicationType.deduceFromClasspath();
        // 4\. 设置初始化器，getSpringFactoriesInstances：从 META-INF/spring.factories 中获取配置
        setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));
        // 5\. 设置监听器，getSpringFactoriesInstances：从 META-INF/spring.factories 中获取配置
        setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
        // 6\. 返回包含main()方法的class
        this.mainApplicationClass = deduceMainApplicationClass();
    }
}

```

这个方法所做的事还是比较清晰的，相关内容都已在代码中注释了，不过有些方法还需要展开，接下来我们就来看看。

### 2.1 获取当前 web 应用类型：`WebApplicationType.deduceFromClasspath()`

`WebApplicationType.deduceFromClasspath()`?方法是用来推断当前项目是什么类型的，代码如下：

```
public enum WebApplicationType {
    // 不是 web 应用
    NONE,

    // servlet 类型的 web 应用
    SERVLET,

    // reactive 类型的 web 应用
    REACTIVE;

    ...

    private static final String[] SERVLET_INDICATOR_CLASSES = { 
            "javax.servlet.Servlet",
            "org.springframework.web.context.ConfigurableWebApplicationContext" };

    private static final String WEBMVC_INDICATOR_CLASS 
            = "org.springframework.web.servlet.DispatcherServlet";

    private static final String WEBFLUX_INDICATOR_CLASS 
            = "org.springframework.web.reactive.DispatcherHandler";

    private static final String JERSEY_INDICATOR_CLASS 
            = "org.glassfish.jersey.servlet.ServletContainer";

    static WebApplicationType deduceFromClasspath() {
        // classpath 中仅存在 WEBFLUX 相关类
        if (ClassUtils.isPresent(WEBFLUX_INDICATOR_CLASS, null) 
                && !ClassUtils.isPresent(WEBMVC_INDICATOR_CLASS, null)
                && !ClassUtils.isPresent(JERSEY_INDICATOR_CLASS, null)) {
            return WebApplicationType.REACTIVE;
        }
        // classpath 不存在 SERVLET 相关类
        for (String className : SERVLET_INDICATOR_CLASSES) {
            if (!ClassUtils.isPresent(className, null)) {
                return WebApplicationType.NONE;
            }
        }
        // 默认 web 类型为 SERVLET
        // 也就是说，同时存在 WEBFLUX 与 SERVLET 相关类，最终返回的是 SERVLET
        return WebApplicationType.SERVLET;
    }

    ...
}

```

可以看到，springboot 定义了三种项目类型：`NONE`(不是 web 应用)、`SERVLET`(`servlet`?类型的 web 应用)、`REACTIVE`(`reactive`?类型的 web 应用)，`WebApplicationType.deduceFromClasspath()`?的执行流程如下：

1.  如果?`classpath`?中仅存在?`WEBFLUX`?相关类，则表明当前项目是?`reactive`?类型的 web 应用，返回；
2.  如果?`classpath`?中不存在?`SERVLET`?相关类，则表明当前项目不是 web 应用，返回；
3.  如果以上条件都不满足，则表明当前项目是?`servlet`?类型的 web 应用。

由于 demo 引用了?`spring-boot-starter-web`?相关依赖，因此当前项目是?`servlet`?类型的 web 应用。

### 2.2 设置初始化器：`setInitializers(...)`

这块代码如下：

```
setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));

```

这行代码分为两部分:

*   获取?`ApplicationContextInitializer`：`getSpringFactoriesInstances(ApplicationContextInitializer.class)`
*   设置初始化器：`setInitializers(...)`

我们先来看看获取?`ApplicationContextInitializer`?的流程，代码如下：

```
public class SpringApplication {
    ...

    // type 为 ApplicationContextInitializer.class
    private <T> Collection<T> getSpringFactoriesInstances(Class<T> type) {
        return getSpringFactoriesInstances(type, new Class<?>[] {});
    }

    /**
     * type 为 ApplicationContextInitializer.class
     * parameterTypes 为 ew Class<?>[] {}
     * args 为 null
     */
    private <T> Collection<T> getSpringFactoriesInstances(Class<T> type, 
            Class<?>[] parameterTypes, Object... args) {
        ClassLoader classLoader = getClassLoader();
        // 从 META-INF/spring.factories 加载内容
        Set<String> names = new LinkedHashSet<>(
                SpringFactoriesLoader.loadFactoryNames(type, classLoader));
        // 实例化，使用的反射操作
        List<T> instances = createSpringFactoriesInstances(
                type, parameterTypes, classLoader, args, names);
        // 排序，比较的是 @Order 注解，或实现的 Orderd 接口
        AnnotationAwareOrderComparator.sort(instances);
        return instances;
    }
    ...
}

```

以上代码比较简单，先从?`META-INF/spring.factories`?获取内容，然后使用反射进行实例化，进行排序后再返回。

关于?`SpringFactoriesLoader.loadFactoryNames(...)`，它最终会从?`META-INF/spring.factories`?加载内容，`META-INF/spring.factories`?可以理解为一个配置文件，key 为传入的 type，该方法的详细分析，可以参考?[springboot 自动装配之加载自动装配类](https://my.oschina.net/funcy/blog/4870868)。

最终会有多少个?`ApplicationContextInitializer`?加载进来呢？通过调试，发现一共有 7 个：

![](https://oscimg.oschina.net/oscnet/up-53f764fefeb0c55fcfef6e34634805162f5.png)

对这 7 个?`ApplicationContextInitializer`，说明如下：

*   `ConfigurationWarningsApplicationContextInitializer`：报告 IOC 容器的一些常见的错误配置
*   `ContextIdApplicationContextInitializer`：设置 Spring 应用上下文的 ID
*   `DelegatingApplicationContextInitializer`：加载?`application.properties`?中?`context.initializer.classes`?配置的类
*   `RSocketPortInfoApplicationContextInitializer`：将?`RSocketServer`?实际使用的监听端口写入到?`Environment`?环境属性中
*   `ServerPortInfoApplicationContextInitializer`：将内置 servlet 容器实际使用的监听端口写入到?`Environment`?环境属性中
*   `SharedMetadataReaderFactoryContextInitializer`：创建一个?`SpringBoot`?和?`ConfigurationClassPostProcessor`?共用的?`CachingMetadataReaderFactory`?对象
*   `ConditionEvaluationReportLoggingListener`：将?`ConditionEvaluationReport`?写入日志

获取到?`ApplicationContextInitializer`，我们再来看看?`setInitializers(...)`?方法：

```
public class SpringApplication {
    ...
    public void setInitializers(
            Collection<? extends ApplicationContextInitializer<?>> initializers) {
        this.initializers = new ArrayList<>(initializers);
    }
    ...
}

```

这是一个标准的?`setter`?方法，所做的就只是设置成员变量。

### 2.3 设置监听器：`setListeners(...)`

设置监听器的代码如下：

```
setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));

```

从形式上看，同?`Initializer`?一样，也是先从?`META-INF/spring.factories`?中加载?`ApplicationListener`，然后添加到成员变量中，这里我们直接看能获取到哪些?`listener`：

![](https://oscimg.oschina.net/oscnet/up-0440eb21c69a75686850a1b44eb9f1287c8.png)

可以看到，一共可以获取到 11 个?`listener`，这些?`listener`?的作用如下：

*   `ClearCachesApplicationListener`：应用上下文加载完成后对缓存做清除工作
*   `ParentContextCloserApplicationListener`：监听双亲应用上下文的关闭事件并往自己的子应用上下文中传播
*   `CloudFoundryVcapEnvironmentPostProcessor`：对?`CloudFoundry`?提供支持
*   `FileEncodingApplicationListener`：检测系统文件编码与应用环境编码是否一致，如果系统文件编码和应用环境的编码不同则终止应用启动
*   `AnsiOutputApplicationListener`：根据?`spring.output.ansi.enabled`?参数配置?`AnsiOutput`
*   `ConfigFileApplicationListener`：从常见的那些约定的位置读取配置文件
*   `DelegatingApplicationListener`：监听到事件后转发给?`application.properties`?中配置的?`context.listener.classes`?的监听器
*   `ClasspathLoggingApplicationListener`：对环境就绪事件?`ApplicationEnvironmentPreparedEvent`?和应用失败事件?`ApplicationFailedEvent`?做出响应
*   `LoggingApplicationListener`：配置?`LoggingSystem`，使用?`logging.config`?环境变量指定的配置或者缺省配置
*   `LiquibaseServiceLocatorApplicationListener`：使用一个可以和?`SpringBoot`?可执行 jar 包配合工作的版本替换?`LiquibaseServiceLocator`
*   `BackgroundPreinitializer`：使用一个后台线程尽早触发一些耗时的初始化任务

再来看看?`SpringApplication#setListeners`：

```
public void setListeners(Collection<? extends ApplicationListener<?>> listeners) {
    this.listeners = new ArrayList<>(listeners);
}

```

这也是一个标准的?`setter`?方法。

### 2.4 推断主类：`deduceMainApplicationClass()`

所谓主类，就是包含?`main(String[])`，也就是当前 spring 应用的启动类，`SpringApplication#deduceMainApplicationClass`?代码如下：

```
private Class<?> deduceMainApplicationClass() {
    try {
        // 获取调用栈
        StackTraceElement[] stackTrace = new RuntimeException().getStackTrace();
        // 遍历调用栈，找 到main方法
        for (StackTraceElement stackTraceElement : stackTrace) {
            if ("main".equals(stackTraceElement.getMethodName())) {
                return Class.forName(stackTraceElement.getClassName());
            }
        }
    }
    catch (ClassNotFoundException ex) {
        // Swallow and continue
    }
    return null;
}

```

这里主要是通过?`new RuntimeException().getStackTrace()`?获取调用栈，然后遍历，得到包含?`main`?方法的类，得到的调用栈如下：

![](https://oscimg.oschina.net/oscnet/up-8c04487e6b05f583e7d45b83c293634f42a.png)

可以看到，`main()`?就包含在调用栈中了。

### 2.5 总结

本文主要是介绍?`SpringApplication`?的创建过程，我们重点分析了以下几点：

1.  推断当前 web 应用类型：NONE, SERVLET,REACTIVE；
2.  设置初始化器：`ApplicationContextInitializer`；
3.  设置监听器：`ApplicationListener`；
4.  推断主类。

![](https://oscimg.oschina.net/oscnet/up-e9a43f1c523c0f19d37e4741580ed32ca08.png)

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4877610](https://my.oschina.net/funcy/blog/4877610)?，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_