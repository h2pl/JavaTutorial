在 springboot 中，如果我们需要注册 servlet 三大组件：`servlet`、`filter`、`listener`，该怎么做呢，springboot 贴心地为我们提供了 3 种方法，本文就来分析这 3 种方法的源码实现。

### 1\. 注册方式

#### 1.1 使用 `XxxRegistrationBean` 注册

springboot 提供了三个类型的 `RegistrationBean` 来处理 servlet 三大组件的注册，分别是 `ServletRegistrationBean`、`FilterRegistrationBean`、`ServletListenerRegistrationBean`，这里我们简单示意下它们的用法：

```
/**
 * 准备了一个servlet
 */
public class MyServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // 处理一些操作
        ...
    }
}

/**
 * 进行注册操作
 */
@Bean
public ServletRegistrationBean registerServlet() {
    ServletRegistrationBean servletRegistrationBean = new ServletRegistrationBean(
            new MyServlet(), "/myServlet");
    // 处理一些配置操作
    servletRegistrationBean.setXxx();
    ...
    return servletRegistrationBean;
}

```

以上提供了 `servlet` 的注册方式，要注册 `filter`、`listener`，只需使用对应的 `RegistrationBean` 即可，这里就不展示了。

#### 1.2 使用 servlet 注解注册

在 `Servlet 3.0`，servlet 容器提供了 3 个注解来处理 `servlet` 三大组件的注册：

*   `@WebServlet`: 处理 `servlet` 注册
*   `@WebFilter`: 处理 `filter` 注册
*   `@WebListener`: 处理 `listener` 注册

还是以 `servlet` 注册为例，先来看看 `@WebServlet`:

```
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface WebServlet {
    String name() default "";

    String[] value() default {};

    String[] urlPatterns() default {};

    int loadOnStartup() default -1;

    WebInitParam[] initParams() default {};

    boolean asyncSupported() default false;

    String smallIcon() default "";

    String largeIcon() default "";

    String description() default "";

    String displayName() default "";
}

```

可以看到，`@WebServlet` 支持多个属性配置，像指定 servlet 的名称、映射的 url 都可以在这里指定，我们也提供一个示例：

```
@WebServlet(name = "myServlet", urlPatterns = "/myServlet")
public class JavaServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // 处理一些操作
        ...
    }

}

```

这样处理后，还要做一个重要的操作，那就是使用 `@ServletComponentScan` 来开启扫描功能：

```
// 使用 @ServletComponentScan 来开启 servlet 组件的扫描功能
@ServletComponentScan
@SpringBootApplication
public class MyApplication {
    public static void main(String[] args) {
        ...
    }
}

```

#### 1.3 `ServletContextInitializer` 注册

使用这种方式注册，需要实现 `ServletContextInitializer` 接口：

```
/**
 * 准备一个servlet
 */
public class MyServlet extends HttpServlet {

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        // 处理一些操作
        ...
    }
}

/**
 * 实现 ServletContextInitializer 
 */
@Component
public class ServletConfig implements ServletContextInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {
        // 使用 servletContext 进行注册
        ServletRegistration initServlet = servletContext.addServlet("myServlet", MyServlet.class);
        // 可以进行一些配置
        initServlet.addMapping("/myServlet");
    }

}

```

使用这种方式注册，先要实现 `ServletContextInitializer`，然后重写 `ServletContextInitializer#onStartup` 方法，在 `ServletContextInitializer#onStartup` 使用 `ServletContext` 对象进行注册。`ServletContext` 对象由 servlet 容器提供，这个对象就是注册的终极类，不管是使用 `RegistrationBean` 注册，还是使用 `@ServletComponentScan` 扫描注册，最终都是通过 `ServletContext` 注册到 servlet 容器中。

### 2\. 源码实现

了解完如何使用后，接下来我们就 进入源码看看这些流程是如何实现。

#### 2.1 `@ServletComponentScan` 扫描

我们直接进入 `@ServletComponentScan`：

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(ServletComponentScanRegistrar.class)
public @interface ServletComponentScan {
    ...
}

```

这个注册上面标记了 `@Import` 注解，引入了一个类：`ServletComponentScanRegistrar`，我们看看这个类究竟做了啥：

```
/**
 * 实现了ImportBeanDefinitionRegistrar
 * 向容器中注册了 ServletComponentRegisteringPostProcessor
 */
class ServletComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    private static final String BEAN_NAME = "servletComponentRegisteringPostProcessor";

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
            BeanDefinitionRegistry registry) {
        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);
        if (registry.containsBeanDefinition(BEAN_NAME)) {
            updatePostProcessor(registry, packagesToScan);
        }
        else {
            // 注册 BeanFactoryPostProcessor
            addPostProcessor(registry, packagesToScan);
        }
    }

    /**
     * 注册 BeanFactoryPostProcessor
     * 注册了 ServletComponentRegisteringPostProcessor
     */
    private void addPostProcessor(BeanDefinitionRegistry registry, Set<String> packagesToScan) {
        GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
        // ServletComponentRegisteringPostProcessor: 处理扫描的 BeanFactoryPostProcessor
        beanDefinition.setBeanClass(ServletComponentRegisteringPostProcessor.class);
        beanDefinition.getConstructorArgumentValues().addGenericArgumentValue(packagesToScan);
        beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
        // ServletComponentScanRegistrar 就是为了注册 ServletComponentRegisteringPostProcessor
        registry.registerBeanDefinition(BEAN_NAME, beanDefinition);
    }

    ...

}

```

可以看到，这个类实现了 `ImportBeanDefinitionRegistrar`，主要是向 spring 容器中注册了 `ServletComponentRegisteringPostProcessor`。我们继续看下去，进入 `ServletComponentRegisteringPostProcessor`：

```
class ServletComponentRegisteringPostProcessor implements BeanFactoryPostProcessor, 
        ApplicationContextAware {

    /**
     * 需要扫描的包.
     */
    private final Set<String> packagesToScan;

    /**
     * 要扫描的包由构造方法传入
     */
    ServletComponentRegisteringPostProcessor(Set<String> packagesToScan) {
        this.packagesToScan = packagesToScan;
    }

    /**
     * 重写了BeanFactoryPostProcessor的方法
     */
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
            throws BeansException {
        // 判断是否运行在内嵌的 web 容器中
        if (isRunningInEmbeddedWebServer()) {
            // 扫描器，配置了扫描规则
            ClassPathScanningCandidateComponentProvider componentProvider 
                    = createComponentProvider();
            for (String packageToScan : this.packagesToScan) {
                // 进行包扫描操作
                scanPackage(componentProvider, packageToScan);
            }
        }
    }

    ...

}

```

可以看到，`ServletComponentRegisteringPostProcessor` 实现了 `BeanFactoryPostProcessor`，在重写的 `BeanFactoryPostProcessor#postProcessBeanFactory` 方法中处理扫描操作，扫描前，先是创建了扫描器 `ClassPathScanningCandidateComponentProvider`，然后再进行扫描。

我们先来看扫描器的创建方法 `createComponentProvider()`：

```
// 处理各种 handler
private static final List<ServletComponentHandler> HANDLERS;

static {
    List<ServletComponentHandler> servletComponentHandlers = new ArrayList<>();
    servletComponentHandlers.add(new WebServletHandler());
    servletComponentHandlers.add(new WebFilterHandler());
    servletComponentHandlers.add(new WebListenerHandler());
    HANDLERS = Collections.unmodifiableList(servletComponentHandlers);
}

/**
 * 创建扫描器
 */
private ClassPathScanningCandidateComponentProvider createComponentProvider() {
    // 创建对象
    ClassPathScanningCandidateComponentProvider componentProvider 
            = new ClassPathScanningCandidateComponentProvider(false);
    componentProvider.setEnvironment(this.applicationContext.getEnvironment());
    componentProvider.setResourceLoader(this.applicationContext);
    for (ServletComponentHandler handler : HANDLERS) {
        // 配置过滤规则
        componentProvider.addIncludeFilter(handler.getTypeFilter());
    }
    return componentProvider;
}

```

`createComponentProvider()` 方法中，先是创建了扫描器对象，然后设置了一些属性，接着就是配置过滤规则，我们这里重点来看下过滤规则的配置，这些规则由 `WebServletHandler`/`WebFilterHandler`/`WebListenerHandler` 的 `getTypeFilter()` 方法提供：

`getTypeFilter()` 方法位于一个抽象方法中：

```
abstract class ServletComponentHandler {

    private final TypeFilter typeFilter;

    /**
     * 传入注解，转换为 AnnotationTypeFilter 对象
     */
    protected ServletComponentHandler(Class<? extends Annotation> annotationType) {
        this.typeFilter = new AnnotationTypeFilter(annotationType);
        ...
    }

    /**
     * 返回 TypeFilter
     */
    TypeFilter getTypeFilter() {
        return this.typeFilter;
    }
    ...
}

```

在 `ServletComponentHandler` 中，有一个成员变量 `typeFilter`，在构造方法中传入注册值后会转换会 `AnnotationTypeFilter`，然后赋值给 `typeFilter`，而 `getTypeFilter()` 方法返回的就是这个 `typeFilter`。

了解完这个 `typeFilter` 的来源后，我们来看看它的几个实现类：

```
/**
 * WebFilterHandler 构造方法传入的参数是 WebFilter
 */
class WebFilterHandler extends ServletComponentHandler {
    WebFilterHandler() {
        super(WebFilter.class);
    }
    ...
}

/**
 * WebListenerHandler 构造方法传入的参数是 WebListener
 */
class WebListenerHandler extends ServletComponentHandler {
    WebListenerHandler() {
        super(WebListener.class);
    }
    ...
}

/**
 * WebServletHandler 构造方法传入的参数是 WebServlet
 */
class WebServletHandler extends ServletComponentHandler {
    WebServletHandler() {
        super(WebServlet.class);
    }
    ...
}

```

由此就明白了，`createComponentProvider()` 得到的 `ClassPathScanningCandidateComponentProvider` 只处理包含 3 个注解的类：

*   `@WebFilter`
*   `@WebListener`
*   `@WebServlet`

我们继续，接下来看看扫描流程，方法为 `ServletComponentRegisteringPostProcessor#scanPackage`:

```
private void scanPackage(ClassPathScanningCandidateComponentProvider componentProvider, 
        String packageToScan) {
    for (BeanDefinition candidate : componentProvider.findCandidateComponents(packageToScan)) {
        if (candidate instanceof AnnotatedBeanDefinition) {
            // 处理得到的 BeanDefinition
            for (ServletComponentHandler handler : HANDLERS) {
                handler.handle(((AnnotatedBeanDefinition) candidate),
                        (BeanDefinitionRegistry) this.applicationContext);
            }
        }
    }
}

```

关于具体的扫描流程（`ClassPathScanningCandidateComponentProvider#findCandidateComponents` 方法），同 spring 的包扫描流程基本一致，这里就不展开细讲了，我们把重点放在 `BeanDefinition` 的处理上，也就是 `ServletComponentHandler#handle` 方法：

```
void handle(AnnotatedBeanDefinition beanDefinition, BeanDefinitionRegistry registry) {
    // 这里的 annotationType 就是构造方法中传入的注解，如@WebFilter，@WebListener 等
    Map<String, Object> attributes = beanDefinition.getMetadata()
            .getAnnotationAttributes(this.annotationType.getName());
    // 判断对应的注解是否存在，存在则处理
    if (attributes != null) {
        doHandle(attributes, beanDefinition, registry);
    }
}

```

在处理扫描得到的 `BeanDefinition` 时，先遍历所有的 `handler`(`WebServletHandler`/`WebFilterHandler`/`WebListenerHandler`)，然后调用 `ServletComponentHandler#handle` 方法进行处理。在 `ServletComponentHandler#handle` 中，又会根据是否存在对应的注解是否存在（使用 `AnnotatedBeanDefinition#getMetadata` 获取对应注册的信息）来决定是否调用子类的 `doHandler()` 方法。

那么这个 `doHandler()` 方法干了什么呢？我们进入 `WebServletHandler#doHandle`：

```
public void doHandle(Map<String, Object> attributes, AnnotatedBeanDefinition beanDefinition,
        BeanDefinitionRegistry registry) {
    // 注册的是 ServletRegistrationBean 对应的 BeanDefinition
    BeanDefinitionBuilder builder = BeanDefinitionBuilder
            .rootBeanDefinition(ServletRegistrationBean.class);
    builder.addPropertyValue("asyncSupported", attributes.get("asyncSupported"));
    builder.addPropertyValue("initParameters", extractInitParameters(attributes));
    builder.addPropertyValue("loadOnStartup", attributes.get("loadOnStartup"));
    // 获取 servlet 名称，如果指定了名称，就使用指定名称，如果没有指定，就使用是bean的名称
    String name = determineName(attributes, beanDefinition);
    builder.addPropertyValue("name", name);
    builder.addPropertyValue("servlet", beanDefinition);
    builder.addPropertyValue("urlMappings", extractUrlPatterns(attributes));
    builder.addPropertyValue("multipartConfig", determineMultipartConfig(beanDefinition));
    registry.registerBeanDefinition(name, builder.getBeanDefinition());
}

```

可以看到，这个方法主要是处理 `Servlet` 的配置，最终向 spring 容器中注册的是 `ServletRegistrationBean` 对应的 `beanDefinition`。

其他两个 `Handler` 类的 `doHandle()` 方法处理流程也差不多，不过最终向 spring 容器中注册的 `beanDefinition` 有所不同，这里就不细读了。

这里总结下这几个注解最终向 spring 容器中注册的 `beanDefinition`：

*   `@WebServlet`: 注册了 `ServletRegistrationBean` 对应的 `beanDefinition`
*   `@WebFilter`: 注册了 `FilterRegistrationBean` 对应的 `beanDefinition`
*   `@WebListener`: 注册了 `ServletListenerRegistrationBean` 对应的 `beanDefinition`

在使用 `XxxRegistrationBean` 注册时，我们是手动创建了 `XxxRegistrationBean`，然后通过 `@Bean` 注解注册到 spring 容器中，而使用 `@WebServlet`/`@WebFilter`/`@WebListener` 闹了一圈，最终也是回到了 `XxxRegistrationBean`！

#### 2.2 `XxxRegistrationBean` 的注册

无论是使用 `XxxRegistrationBean` 注册，还是使用 `@ServletComponentScan` 扫描注册，最终都会得到 `XxxRegistrationBean` 对应的 bean，接下来我们就来探究下这些 bean 是如何注册到 servlet 容器中的。

从代码上看，`ServletRegistrationBean`、`FilterRegistrationBean` 与 `ServletListenerRegistrationBean` 都是 `ServletContextInitializer` 接口的实现类，`ServletRegistrationBean` 的继承结构如下：

![图片来自网络](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0397d56cfb328683ae349d0822db2320231.png)

`ServletContextInitializer` 中只有一个方法 `onStartup(...)`：

```
@FunctionalInterface
public interface ServletContextInitializer {

    /**
     * 就是这个 servletContext，有了它，就是进行 Servlet，filter，listener 的注册了
     */
    void onStartup(ServletContext servletContext) throws ServletException;

}

```

我们在介绍 `servlet` 三大组件的注册方式时，我们提到可以通过实现 `ServletContextInitializer`，重写其 `onStartup()` 来实现 `servlet` 组件的注册，而 `XxxRegistrationBean` 的底层实现也是这么做的。

以 `ServletRegistrationBean`，我们来看看它的 `onStartup(...)` 方法做了啥：

`ServletRegistrationBean` 没有重写 `onStartup(...)` 方法，直接继承自 `RegistrationBean`:

```
public final void onStartup(ServletContext servletContext) throws ServletException {
    // 获取描述信息 
    String description = getDescription();
    // 是否开启注册，默认为true
    if (!isEnabled()) {
        logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
        return;
    }
    // 进行注册操作
    register(description, servletContext);
}

```

这个方法并不复杂，先获取了一下描述信息，然后判断是否开启了注册，接着就是进行注册操作了。我们直接查看注册操作，进入 `DynamicRegistrationBean#register`:

```
protected final void register(String description, ServletContext servletContext) {
    D registration = addRegistration(description, servletContext);
    if (registration == null) {
        logger.info(...);
        return;
    }
    // 处理配置
    configure(registration);
}

```

这个方法主要做了两个件事：注册 `servlet` 与处理配置，我们先来看看注册操作，进入 `ServletRegistrationBean#addRegistration` 方法:

```
protected ServletRegistration.Dynamic addRegistration(String description, 
        ServletContext servletContext) {
    String name = getServletName();
    // 注册
    return servletContext.addServlet(name, this.servlet);
}

```

注册操作还是比较简单的，直接调用 `ServletContext#addServlet` 进行。

继续查看配置处理，进入 `ServletRegistrationBean#configure` 方法：

```
protected void configure(ServletRegistration.Dynamic registration) {
    // 调用父类
    super.configure(registration);
    // 配置urlMapping
    String[] urlMapping = StringUtils.toStringArray(this.urlMappings);
    if (urlMapping.length == 0 && this.alwaysMapUrl) {
        urlMapping = DEFAULT_MAPPINGS;
    }
    if (!ObjectUtils.isEmpty(urlMapping)) {
        registration.addMapping(urlMapping);
    }
    // 配置loadOnStartup
    registration.setLoadOnStartup(this.loadOnStartup);
    // 还处理了一些其他配置
    if (this.multipartConfig != null) {
        registration.setMultipartConfig(this.multipartConfig);
    }
}

```

这个方法先是调用了父类的方法，然后就是配置处理了，在这个方法里主要是处理了 `urlMapping` 与 `loadOnStartup`，就不多做分析了。

我们再来看看 `super.configure(...)` 配置了啥，进入 `DynamicRegistrationBean#configure`:

```
/**
 * 也是处理一些配置
 */
protected void configure(D registration) {
    registration.setAsyncSupported(this.asyncSupported);
    // 配置初始化参数
    if (!this.initParameters.isEmpty()) {
        registration.setInitParameters(this.initParameters);
    }
}

```

这个方法主要处理了初始参数的配置。

从上面的分析来看，`ServletRegistrationBean` 的 `onStartup(...)` 方法主要处理了两个操作：

1.  将 `servlet` 添加到容器中
2.  处理 `servlet` 参数配置

`FilterRegistrationBean` 与 `ServletListenerRegistrationBean` 的注册操作类似，这里就不多说了。

#### 2.3 `ServletContextInitializer#onStartup` 的执行

注册流程已经捋完了，我们再来看看 `ServletContextInitializer#onStartup` 是在哪里执行的。注：这一步的流程比较复杂，会涉及到 tomcat 的启动流程，因此这部分只关注重点代码，不具体分析一步步流程。

以 tomcat 容器为例，经过一系列的调试与代码追踪，发现它是在 `TomcatStarter` 中运行的，代码如下：

```
class TomcatStarter implements ServletContainerInitializer {

    private final ServletContextInitializer[] initializers;

    TomcatStarter(ServletContextInitializer[] initializers) {
        this.initializers = initializers;
    }

    @Override
    public void onStartup(Set<Class<?>> classes, ServletContext servletContext) 
            throws ServletException {
        try {
            for (ServletContextInitializer initializer : this.initializers) {
                // 这里执行 ServletContextInitializer#onStartup方法
                initializer.onStartup(servletContext);
            }
        }
        catch (Exception ex) {
            this.startUpException = ex;
            ...
        }
    }

    ...

}

```

`TomcatStarter` 是 springboot 提供的类，它实现了 `ServletContainerInitializer`，区别于 `ServletContextInitializer`，`ServletContainerInitializer` 是由 tomcat 提供的，在 tomcat 启动时，会执行 `ServletContainerInitializer#onStartup` 方法（`servlt 3.0` 规范）。

那么 `TomcatStarter` 是如何添加到 tomcat 容器中的呢？虽然 `servlt 3.0` 规范可以通过 `spi` 技术扫描到 `ServletContainerInitializer` 的实现，但是这里明显不是这样做的，因为如果由 tomcat 通过 `spi` 扫描得到 `TomcatStarter` 的实例，那它的成员变量 `initializers` 就无法赋值了，所以在添加到 tomcat 前，`TomcatStarter` 就要实例化并且 `initializers` 就要被赋值。

经过多次调试，发现 `TomcatStarter` 是在 `TomcatServletWebServerFactory#configureContext` 中添加到 tomcat 容器的，关键代码如下:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-6625407ce4d1dad8de736db00db124a2a99.png)

可以看到，`initializers` 被当作构造参数传入到 `TomcatStarter` 的构造方法中，得到 `TomcatStarter` 的实例，再手动添加到 tomcat 容器中了。

那么，这个 `initializers` 是在哪里获取到的呢？事实上，我们的 `XxxRegistrationBean` 都要 spring 容器中，要获取的话，只要调用 `beanFactory.getBeansOfType(...)` 就可以了，`ServletContextInitializerBeans#addServletContextInitializerBean(String, ServletContextInitializer, ListableBeanFactory)` 就是干这件事的：

```
private void addServletContextInitializerBeans(ListableBeanFactory beanFactory) {
    for (Class<? extends ServletContextInitializer> initializerType : this.initializerTypes) {
        // 获取 ServletContextInitializer: getOrderedBeansOfType(beanFactory, initializerType)
        for (Entry<String, ? extends ServletContextInitializer> initializerBean 
                : getOrderedBeansOfType(beanFactory, initializerType)) {
            addServletContextInitializerBean(initializerBean.getKey(), 
                initializerBean.getValue(), beanFactory);
        }
    }
}

/**
 * 处理添加操作
 */
private void addServletContextInitializerBean(String beanName, 
        ServletContextInitializer initializer, ListableBeanFactory beanFactory) {
    // 添加 ServletRegistrationBean
    if (initializer instanceof ServletRegistrationBean) {
        Servlet source = ((ServletRegistrationBean<?>) initializer).getServlet();
        addServletContextInitializerBean(Servlet.class, beanName, initializer, 
                beanFactory, source);
    }
    // 添加 FilterRegistrationBean
    else if (initializer instanceof FilterRegistrationBean) {
        Filter source = ((FilterRegistrationBean<?>) initializer).getFilter();
        addServletContextInitializerBean(Filter.class, beanName, initializer, 
                beanFactory, source);
    }
    // 添加 DelegatingFilterProxyRegistrationBean
    else if (initializer instanceof DelegatingFilterProxyRegistrationBean) {
        String source = ((DelegatingFilterProxyRegistrationBean) initializer).getTargetBeanName();
        addServletContextInitializerBean(Filter.class, beanName, initializer, beanFactory, source);
    }
    // 添加 ServletListenerRegistrationBean
    else if (initializer instanceof ServletListenerRegistrationBean) {
        EventListener source = ((ServletListenerRegistrationBean<?>) initializer).getListener();
        addServletContextInitializerBean(EventListener.class, beanName, initializer, 
                beanFactory, source);
    }
    else {
        // 其他的 ServletContextInitializer Bean
        addServletContextInitializerBean(ServletContextInitializer.class, beanName, 
                initializer, beanFactory, initializer);
    }
}

```

### 3\. 总结

本文分析了 springboot 注册 servlet 三大组件的流程：

1.  以 `Servlet` 为例，介绍了 3 种注册方式：使用 `XxxRegistrationBean` 注册、使用 `servlet` 注解 (`@WebServlet`/`@WebFilter`/`@WebListener`) 注册，以及实现 `ServletContextInitializer` 接口手动注册；
2.  分析了 `@ServletComponentScan` 注册的扫描流程
3.  以 `ServletRegistrationBean` 为例，分析了将 `ServletRegistrationBean` 注册到 servlet 的流程
4.  分析了 `ServletContainerInitializer#onStartup` 的执行

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4951050](https://my.oschina.net/funcy/blog/4951050) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_