在[上一篇文章](https://my.oschina.net/funcy/blog/4696657 "上一篇文章")中，我们通过一个简单的 demo 成功启动了 springmvc 应用，在提供的 demo 中，我们知道 tomcat 在启动时会调用 `MyWebApplicationInitializer#onStartup` 方法，然后启动 spring 容器。那么 tomcat 究竟是如何启动 spring 的呢？

### 1\. servlet 初始化：`DispatcherServlet#init`

我们再回忆下 `MyWebApplicationInitializer#onStartup` 方法：

```
@Override
public void onStartup(ServletContext servletContext) {
   System.out.println("webApplicationInitializer ...");
   // 创建 spring 的 applicationContext
   AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
   context.register(MvcConfig.class);

   // 实例化 DispatcherServlet
   DispatcherServlet servlet = new DispatcherServlet(context);

   // 将DispatcherServlet注册到servlet容器
   ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
   registration.setLoadOnStartup(1);
   registration.addMapping("/*");
}

```

这段代码先准备了一个 `AnnotationConfigWebApplicationContext`，将其作为参数传入 `DispatcherServlet` 中，然后往 servlet 容器中添加 `DispatcherServlet`。这样，servlet 容器在启动时，就会启动 spring 容器了。这里最重要的就是 `DispatcherServlet`，接下来我们就来分析这个 servlet.

我们先来看下 `DispatcherServlet` 的继承结构：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1df54493011b3fc9cb9cafbb944f1a88256.png)

从上图可以看到，spring 提供的、跟 servlet 相关的类有三个：`HttpServletBean`、`FrameworkServlet` 与 `DispatcherServlet`。作为 servlet，我们知道其初始化方法为 `GenericServlet#init()`，也就是 servlet 的入口方法，我们的分析也将从这里开始。

由于 `DispatcherServlet` 实现了 `HttpServletBean`、`FrameworkServlet`，`DispatcherServlet#init()` 实际上继承自 `HttpServletBean#init`：

```
@Override
public final void init() throws ServletException {

    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            ...
        }
    }

    // 初始化 servlet bean，spring的相关配置就是在这里进行的
    initServletBean();
}

```

可以看到，这个方法加载了一些配置，然后就调用了 `initServletBean`，并没有做一些 spring 实质性的内容。我们继续跟进，跳过其中不重要的方法，调用链路如下：

```
-HttpServletBean#init
 -FrameworkServlet#initServletBean
  -FrameworkServlet#initWebApplicationContext

```

一直跟到 `FrameworkServlet#initWebApplicationContext`：

```
protected WebApplicationContext initWebApplicationContext() {
    // 获取类型为WebServerApplicationContext的父容器，这里得到的结果为null
    WebApplicationContext rootContext =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;

    if (this.webApplicationContext != null) {
        // 这个webApplicationContext，就是在MyWebApplicationInitializer#onStart方法中
        // 传入的AnnotationConfigWebApplicationContext
        wac = this.webApplicationContext;
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                if (cwac.getParent() == null) {
                    cwac.setParent(rootContext);
                }
                // 在这里调用 AbstractApplicationContext#refresh 启动
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    // wac不为null,这里不会运行
    if (wac == null) {
        wac = findWebApplicationContext();
    }
    // wac不为null,这里不会运行
    if (wac == null) {
        // 创建WebApplicationContext，重新运行AbstractApplicationContext#refresh
        wac = createWebApplicationContext(rootContext);
    }

    // 实际上，refreshEventReceived为true，if块的代码并不执行
    if (!this.refreshEventReceived) {
        synchronized (this.onRefreshMonitor) {
            // 刷新应用上下文，springmvc相关代码在这里运行
            onRefresh(wac);
        }
    }

    if (this.publishContext) {
        // 将 WebApplicationContext视为servletContext 一个属性，加入到 servletContext 中
        // 之后就可以使用
        // WebApplicationContextUtils.getWebApplicationContext(ServletContext, String attrName)
        // 来获取
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}

```

这个方法的相关操作代码中已有注释，实际上这个方法最 重要的代码为

```
protected WebApplicationContext initWebApplicationContext() {
    ...
    // 在这里调用 AbstractApplicationContext#refresh 启动
    configureAndRefreshWebApplicationContext(cwac);
    ...
    return wac;
}

```

相关分析如下：

> FrameworkServlet#configureAndRefreshWebApplicationContext

```
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        if (this.contextId != null) {
            wac.setId(this.contextId);
        }
        else {
            // Generate default id...
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(getServletContext().getContextPath()) + 
                    '/' + getServletName());
        }
    }
    wac.setServletContext(getServletContext());
    wac.setServletConfig(getServletConfig());
    wac.setNamespace(getNamespace());
    // 添加事件监听器，监听spring启动完成事件
    // 这个监听器十分重要，后面会分析
    wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(getServletContext(), getServletConfig());
    }
    // 扩展点，没有什么功能，待以后扩展
    postProcessWebApplicationContext(wac);
    applyInitializers(wac);
    // 调用就是 AbstractApplicationContext.refresh
    wac.refresh();
}

```

这个方法实际上还是配置 `ConfigurableWebApplicationContext` 的一些属性，最后调用 `AbstractApplicationContext#refresh` 来启动 spring 容器。关于 `AbstractApplicationContext#refresh` 的分析，可以参考 [spring 启动流程之启动前的准备工作](https://my.oschina.net/funcy/blog/4633169 "spring启动流程之启动前的准备工作")。

到这里，spring 容器就真正启动了。

### 2. `SourceFilteringListener`：启动事件监听器

这里有个问题：在 springmvc 中，我们知道 spring 会识别 `@Controller`，将 `RequestMapping`/`@PostMapping`/`@GetMapping` 等注解中的路径封装为一个 uri，等待外部访问，但是我们一路看来，似乎 spring 并没有做这些工作，那么这部分的工作是在哪里进行的呢？

实际上，spring 这部分的工作是在启动监听器中完成的，也就是

```
wac.addApplicationListener(new SourceFilteringListener(wac, new ContextRefreshListener()));

```

这个监听器监听的是 spring 的启动 事件，将在 spring 容器启动完成后调用。

关于 spring 的事件相关内容，可以参考 [spring 探秘之 spring 事件机制](https://my.oschina.net/funcy/blog/4713339 "spring探秘之spring 事件机制")，这里直接说结论：spring 中提供了 `ApplicationEventPublisher#publishEvent(Object)`（事件发布器）、`ApplicationEvent`（事件）与 `ApplicationListener` （事件监听器），当 spring 通过 `ApplicationEventPublisher#publishEvent(Object)` 发布 `ApplicationEvent`（事件）时，`ApplicationListener` （事件监听器）将会监听到。

我们来看看 `SourceFilteringListener`：

```
public class SourceFilteringListener implements GenericApplicationListener, SmartApplicationListener {

    private final Object source;

    @Nullable
    private GenericApplicationListener delegate;

    /**
     * 构造方法，传入 event 与 listener
     */
    public SourceFilteringListener(Object source, ApplicationListener<?> delegate) {
        this.source = source;
        this.delegate = (delegate instanceof GenericApplicationListener ?
                (GenericApplicationListener) delegate : new GenericApplicationListenerAdapter(delegate));
    }

    /**
     * 事件监听方法
     */
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event.getSource() == this.source) {
            // 调用真正的事件处理
            onApplicationEventInternal(event);
        }
    }

    /**
     *  处理事件
     */
    protected void onApplicationEventInternal(ApplicationEvent event) {
        if (this.delegate == null) {
            throw new IllegalStateException(...);
        }
        // 最终还是调用传入的事件监听器的onApplicationEvent方法
        this.delegate.onApplicationEvent(event);
    }

    // 省略了一些代码
    ...

```

可以看到，`SourceFilteringListener` 通过构造方法传入了 `ContextRefreshListener` 的实例，然后在 `SourceFilteringListener#onApplicationEvent` 方法中，最终调用的是 `ContextRefreshListener#onApplicationEvent` 方法。

接下来我们再来看 `ContextRefreshListener`：

> FrameworkServlet.ContextRefreshListener

```
private class ContextRefreshListener implements ApplicationListener<ContextRefreshedEvent> {

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        FrameworkServlet.this.onApplicationEvent(event);
    }
}

```

这个类是 `FrameworkServlet` 的内部类，代码很简单，最终调用的是 `FrameworkServlet#onApplicationEvent`：

```
public void onApplicationEvent(ContextRefreshedEvent event) {
    // 修改状态，这个很关键，运行了这行后，
    // FrameworkServlet#initWebApplicationContext里的onRefresh(...)就不会运行
    this.refreshEventReceived = true;
    synchronized (this.onRefreshMonitor) {
         // 具体的逻辑处理
         onRefresh(event.getApplicationContext());
    }
}

```

接下来就是 `DispatcherServlet#onRefresh` 方法了：

```
@Override
protected void onRefresh(ApplicationContext context) {
    initStrategies(context);
}

/**
 * springmvc的终极奥秘就在这里了
 * 这个方法中，初始化了springmvc的各种组件
 */
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}

```

可以看到，最终运行的方法是 `DispatcherServlet#initStrategies`，这个方法虽只寥寥数行，且初始化了 springmvc 各依赖组件！

### 3. `DispatcherServlet#initStrategies`：初始化 springmvc 组件

spring 在启动完成后，会发布启动完成事件，然后由监听器 `SourceFilteringListener` 监听到该事件后，执行监听逻辑，最终调用到 `DispatcherServlet#initStrategies`。本节我们将来分析 `DispatcherServlet#initStrategies` 的执行过程。

其实这个方法很简单，里面有 9 行代码，每行代码都初始化了 springmvc 的一个组件，如 `initMultipartResolver`：

```
public static final String MULTIPART_RESOLVER_BEAN_NAME = "multipartResolver";

private void initMultipartResolver(ApplicationContext context) {
    try {
        // 从spring容器中获取multipartResolver对象
        this.multipartResolver = context.getBean(MULTIPART_RESOLVER_BEAN_NAME, MultipartResolver.class);
    }
    catch (NoSuchBeanDefinitionException ex) {
        // 获取失败，默认为null
        this.multipartResolver = null;
        }
    }
}

```

`multipartResolver` 是用来处理文件上传的 bean，在 spring 中，我们处理文件上传时，一般会像这样引入 `multipartResolver` bean；

```
@Bean(name = "multipartResolver")
public MultipartResolver multipartResolver() {
    CommonsMultipartResolver resolver = new CommonsMultipartResolver();
    resolver.setDefaultEncoding("UTF-8");
    resolver.setResolveLazily(true);
    resolver.setMaxInMemorySize(40960);
    //允许上传文件最大为1G
    resolver.setMaxUploadSize(1024 * 1024 * 1024);
    return resolver;
}

```

如果未引入 `multipartResolver` bean，spring 默认为 null，就不能进行文件上传了。

再来看看 `springmvc` `HandlerMappings` 的初始化过程：

> DispatcherServlet

```
public static final String HANDLER_MAPPING_BEAN_NAME = "handlerMapping";

private static final String DEFAULT_STRATEGIES_PATH = "DispatcherServlet.properties";

private static final Properties defaultStrategies;

static {
    try {
        // 在static块中加载DispatcherServlet.properties文件
        ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, 
                DispatcherServlet.class);
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    }
    catch (IOException ex) {
        throw new IllegalStateException(...);
    }
}

/**
 * handlerMappings 属性
 */
@Nullable
private List<HandlerMapping> handlerMappings;

/**
 * 初始化 HandlerMappings
 * 1\. 从spring 容器中获取 HandlerMapping bean，
 *     如果获取成功，则把得到的结果赋值给handlerMappings
 * 2\. 如果未获得，则获取默认的 HandlerMapping bean
 */
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;
    if (this.detectAllHandlerMappings) {
        // 加载所有实现HandlerMapping接口的bean
        Map<String, HandlerMapping> matchingBeans = BeanFactoryUtils
                .beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        // 这里不为空，会运行
        if (!matchingBeans.isEmpty()) {
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            // 排序，Spring处理请求就是根据这个排序的结果进行处理，
            // 如果当前handlerMapping不可以处理则抛给下一个
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
    if (this.handlerMappings == null) {
        // 如果未添加handlerMappings，则获取默认的 handlerMappings
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
    }
}

/**
 * 获取默认的策略
 */
protected <T> List<T> getDefaultStrategies(ApplicationContext context, Class<T> strategyInterface) {
    String key = strategyInterface.getName();
    // 获取配置文件DispatcherServlet.properties中默认的 class 配置
    String value = defaultStrategies.getProperty(key);
    if (value != null) {
        String[] classNames = StringUtils.commaDelimitedListToStringArray(value);
        List<T> strategies = new ArrayList<>(classNames.length);
        for (String className : classNames) {
            try {
                // 使用反射创建bean
                Class<?> clazz = ClassUtils.forName(className, DispatcherServlet.class.getClassLoader());
                Object strategy = createDefaultStrategy(context, clazz);
                strategies.add((T) strategy);
            }
            catch (ClassNotFoundException ex) {
                throw new BeanInitializationException(...);
            }
            catch (LinkageError err) {
                throw new BeanInitializationException(...);
            }
        }
        return strategies;
    }
    else {
        return new LinkedList<>();
    }
}

```

初始化 `HandlerMappings` 时，

1.  先从 spring 容器中获取 `HandlerMapping` bean，如果获取成功（实际上这里也能获得），则把得到的结果赋值给 `DispatcherServlet` 的 `handlerMappings` 属性；
2.  如果未失败，表明 spring 容器中未存在 `HandlerMapping` ，则获取默认的 `HandlerMapping` bean.
3.  获取默认的 `HandlerMapping` bean 时，读取 `DispatcherServlet.properties` 配置，然后使用反射实例化。

我们来看看 `DispatcherServlet.properties` 文件，该文件位于` spring-webmvc/src/main/resources/org/springframework/web/servlet/DispatcherServlet.properties`，部分内容如下：

```
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
    org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
    org.springframework.web.servlet.function.support.RouterFunctionMapping
...

```

`DispatcherServlet#initStrategies` 中其他 `initXxx()` 内容类似，这里就不一一分析了。

### 4\. 总结

本文主要分析了 springmvc 的启动过程，总结如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-7ab3fae1e39f545c1d7c1811351227fa434.png)

1.  servlet 容器（这里是 tomcat）启动时，通过 spi 机制执行 `ServletContainerInitializer#onStartup` 方法，而 springmvc 对提供的 `SpringServletContainerInitializer` 对其进行了实现，于是 `SpringServletContainerInitializer#onStartup` 方法会被调用；
2.  `SpringServletContainerInitializer#onStartup` 方法中，spring 会调用 `WebApplicationInitializer#onStartup` 方法，而 `MyWebApplicationInitializer` 对其进行了实现，于是 `MyWebApplicationInitializer#onStartup` 会被调用；
3.  在 `MyWebApplicationInitializer#onStartup` 方法中 ，我们创建了一个 `applicationContext` 对象，将其与 `DispatcherServlet` 绑定，然后将 `DispatcherServlet` 注册到 servlet 容器中（这里是 tomcat）；
4.  `DispatcherServlet` 注册到 servlet 容器中（这里是 tomcat）后，根据 servlet 生命周期，`DispatcherServlet#init` 将会被调用；
5.  `DispatcherServlet#init` 中会执行 spring 容器的启动过程，spring 容器启动后，会发布启动完成事件；
6.  spring 启动完成后，`ContextRefreshListener` 将会监听 spring 启动完成事件，`FrameworkServlet.ContextRefreshListener#onApplicationEvent` 方法会被调用，调用调用到 `DispatcherServlet#initStrategies`；
7.  spring 最终在 `DispatcherServlet#initStrategies` 中初始化 `MultipartResolver`、`LocaleResolver` 等组件，所谓的初始化，其实是获取或创建对应的 bean，然后赋值给 `DispatcherServlet` 的属性。

到此，springmvc 整个启动流程就完成了。不过到此我们都没有看到 **spring 处理 `@RequestMapping` 的相关流程**，那么 spring 是如何处理这个流程呢 ，下一篇文章将揭晓。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4710330](https://my.oschina.net/funcy/blog/4710330) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_