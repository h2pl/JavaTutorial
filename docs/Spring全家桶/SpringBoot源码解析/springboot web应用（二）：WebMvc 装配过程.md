平时工作中，springboot 绝大多数情况下运行的是 web 项目，本文将分析 springboot 自动 springMvc 项目的流程。

### 1\. springMvc 的自动装配类

springMvc 的自动装配类为

```
@Configuration(proxyBeanMethods = false)
// 几个装配条件
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass({ Servlet.class, DispatcherServlet.class, WebMvcConfigurer.class })
// 如果没有自定义WebMvc的配置类，则使用本配置
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
// 装配顺序
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE + 10)
@AutoConfigureAfter({ 
        // DispatcherServlet 的自动装配
        DispatcherServletAutoConfiguration.class, 
        // 线程池的自动装配
        TaskExecutionAutoConfiguration.class,
        // jsr 303 验证框架的自动装配
        ValidationAutoConfiguration.class })
public class WebMvcAutoConfiguration {
    ...
}

```

从上面的 `@AutoConfigureAfter` 注解中，`WebMvcAutoConfiguration` 需要在 `DispatcherServletAutoConfiguration`、`TaskExecutionAutoConfiguration`、`ValidationAutoConfiguration` 等类装配完成之后再装配，对这些类的作用列举如下：

*   `DispatcherServletAutoConfiguration`：`DispatcherServlet` 自动装配
*   `TaskExecutionAutoConfiguration`：任务执行器，其实就是创建了一个线程池
*   `ValidationAutoConfiguration`：jsr 303 验证器的自动装配，验证器用来处理 `@NotNull`、`@NotEmpty` 等注解的验证功能

这 3 个类中，与 springMvc 有关的只有 `DispatcherServletAutoConfiguration`，我们来认识一下它：

```
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@Configuration(proxyBeanMethods = false)
@ConditionalOnWebApplication(type = Type.SERVLET)
@ConditionalOnClass(DispatcherServlet.class)
// 需要在 ServletWebServerFactoryAutoConfiguration 自动装配完成后处理
@AutoConfigureAfter(ServletWebServerFactoryAutoConfiguration.class)
public class DispatcherServletAutoConfiguration {
    ...
}

```

`DispatcherServletAutoConfiguration` 需要等 `ServletWebServerFactoryAutoConfiguration` 自动装配完成才进行装配，这个类是做什么的呢？剧透下，它是处理 servlet 容器（`tomcat`, `jetty`, `undertow` 等）的生成的，我们再来看看 `ServletWebServerFactoryAutoConfiguration`：

```
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
// 引入了一些类
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
        // 3个 web 容器
        ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
        ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
        ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
    ...
}

```

可以看到，这个类中总算是没有 `@AutoConfigureAfter` 注解了，因此这个类 springMvc 就是最初自动装配的类，我们的分析就从这个类开始。

总结下以上几个类的装配顺序：

1.  `ServletWebServerFactoryAutoConfiguration`
2.  `DispatcherServletAutoConfiguration`
3.  `WebMvcAutoConfiguration`

接下来我们的分析也按这样的顺序，逐一分析这些自动装配类。

### 2. `ServletWebServerFactoryAutoConfiguration` 的自动装配

`ServletWebServerFactoryAutoConfiguration` 类如下：

```
@Configuration(proxyBeanMethods = false)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
@ConditionalOnClass(ServletRequest.class)
@ConditionalOnWebApplication(type = Type.SERVLET)
@EnableConfigurationProperties(ServerProperties.class)
// 引入了一些类
@Import({ ServletWebServerFactoryAutoConfiguration.BeanPostProcessorsRegistrar.class,
        // 3个 web 容器
        ServletWebServerFactoryConfiguration.EmbeddedTomcat.class,
        ServletWebServerFactoryConfiguration.EmbeddedJetty.class,
        ServletWebServerFactoryConfiguration.EmbeddedUndertow.class })
public class ServletWebServerFactoryAutoConfiguration {
    ...
}

```

这个类引入了 `BeanPostProcessorsRegistrar`、`EmbeddedTomcat`、`EmbeddedJetty`、`EmbeddedUndertow`，我们来逐一分析吧！

#### 2.1 `BeanPostProcessorsRegistrar`

`BeanPostProcessorsRegistrar` 是 `ServletWebServerFactoryAutoConfiguration` 的内部类，代码如下：

```
public static class BeanPostProcessorsRegistrar 
        implements ImportBeanDefinitionRegistrar, BeanFactoryAware {

    ...

    /**
     * 来自 ImportBeanDefinitionRegistrar 的方法
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata,
            BeanDefinitionRegistry registry) {
        if (this.beanFactory == null) {
            return;
        }
        // 注册组件
        registerSyntheticBeanIfMissing(registry, "webServerFactoryCustomizerBeanPostProcessor",
                WebServerFactoryCustomizerBeanPostProcessor.class);
        registerSyntheticBeanIfMissing(registry, "errorPageRegistrarBeanPostProcessor",
                ErrorPageRegistrarBeanPostProcessor.class);
    }

    /**
     * 具体的注册操作
     */
    private void registerSyntheticBeanIfMissing(BeanDefinitionRegistry registry, 
            String name, Class<?> beanClass) {
        if (ObjectUtils.isEmpty(this.beanFactory.getBeanNamesForType(beanClass, true, false))) {
            RootBeanDefinition beanDefinition = new RootBeanDefinition(beanClass);
            beanDefinition.setSynthetic(true);
            registry.registerBeanDefinition(name, beanDefinition);
        }
    }
}

```

这个类主要是向 spring 容器中注册了两个类：`WebServerFactoryCustomizerBeanPostProcessor`、`ErrorPageRegistrarBeanPostProcessor`，我们来看下它们分别是个啥。

##### 1. `WebServerFactoryCustomizerBeanPostProcessor`

`WebServerFactoryCustomizerBeanPostProcessor` 的代码如下：

```
public class WebServerFactoryCustomizerBeanPostProcessor 
        implements BeanPostProcessor, BeanFactoryAware {

    ...

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) 
            throws BeansException {
        // bean 的类型是 WebServerFactory 才处理
        if (bean instanceof WebServerFactory) {
            postProcessBeforeInitialization((WebServerFactory) bean);
        }
        return bean;
    }

    @SuppressWarnings("unchecked")
    private void postProcessBeforeInitialization(WebServerFactory webServerFactory) {
        LambdaSafe.callbacks(WebServerFactoryCustomizer.class, getCustomizers(), webServerFactory)
                .withLogger(WebServerFactoryCustomizerBeanPostProcessor.class)
                // 处理配置
                .invoke((customizer) -> customizer.customize(webServerFactory));
    }

    /**
     * 获取 WebServerFactoryCustomizer，得到的是一个不可变的 List
     */
    private Collection<WebServerFactoryCustomizer<?>> getCustomizers() {
        if (this.customizers == null) {
            this.customizers = new ArrayList<>(getWebServerFactoryCustomizerBeans());
            this.customizers.sort(AnnotationAwareOrderComparator.INSTANCE);
            this.customizers = Collections.unmodifiableList(this.customizers);
        }
        return this.customizers;
    }

    /**
     * 获取 beanFactory 中所有的 WebServerFactoryCustomizer
     */
    private Collection<WebServerFactoryCustomizer<?>> getWebServerFactoryCustomizerBeans() {
        return (Collection) this.beanFactory.getBeansOfType(
                WebServerFactoryCustomizer.class, false, false).values();
    }

}

```

这个类是用来处理 `WebServerFactory` 的自定义配置的，它实现了 `BeanPostProcessor`， 我们主要关注它的 `postProcessBeforeInitialization(...)` 方法。

在 `WebServerFactoryCustomizerBeanPostProcessor` 的 `postProcessBeforeInitialization(...)` 方法中，如果当前 bean 的类型是 `WebServerFactory`，会先获取 `beanFactory` 中所有类型为 `WebServerFactoryCustomizer` 的 bean，然后将这些 `WebServerFactoryCustomizer` 配置到 `WebServerFactory`bean 中。

如果我们要自定义 `Tomcat` 的配置，可以这样处理：

```
@Component
public class MyCustomizer implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(8091);
        factory.setContextPath("/");
        // 设置其他配置
        ...
    }

}

```

在 `ServletWebServerFactoryAutoConfiguration` 中提供了两个 `WebServerFactoryCustomizer`:

```
public class ServletWebServerFactoryAutoConfiguration {

    @Bean
    public ServletWebServerFactoryCustomizer servletWebServerFactoryCustomizer(
            ServerProperties serverProperties) {
        return new ServletWebServerFactoryCustomizer(serverProperties);
    }

    @Bean
    @ConditionalOnClass(name = "org.apache.catalina.startup.Tomcat")
    public TomcatServletWebServerFactoryCustomizer tomcatServletWebServerFactoryCustomizer(
            ServerProperties serverProperties) {
        return new TomcatServletWebServerFactoryCustomizer(serverProperties);
    }

    ...

}

```

从方法参数来看，这两个类的配置都来自于 `ServerProperties`：

```
@ConfigurationProperties(prefix = "server", ignoreUnknownFields = true)
public class ServerProperties {
    ...
}

```

它上面了 `ConfigurationProperties` 注解，`prefix` 为 "server"，这表明它的配置都是以 `server` 开头，支持的配置如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-586d95875e2eeefd295643c3e6acf0091e1.png)

例如，我们要配置它的端口，只需在 `application.properties` 中这样配置即可：

```
server.port=8080

```

至于其他配置，这里就不多作分析了。

##### 2. `ErrorPageRegistrarBeanPostProcessor`

`ErrorPageRegistrarBeanPostProcessor` 的代码如下：

```
public class ErrorPageRegistrarBeanPostProcessor implements BeanPostProcessor, BeanFactoryAware {

    /**
     * 来自BeanPostProcessor的方法，在bean初始化前执行
     */
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) 
            throws BeansException {
        if (bean instanceof ErrorPageRegistry) {
            // 如果 bean 是 ErrorPageRegistry
            postProcessBeforeInitialization((ErrorPageRegistry) bean);
        }
        return bean;
    }

    private void postProcessBeforeInitialization(ErrorPageRegistry registry) {
        for (ErrorPageRegistrar registrar : getRegistrars()) {
            // 注册错误页
            registrar.registerErrorPages(registry);
        }
    }

    private Collection<ErrorPageRegistrar> getRegistrars() {
        if (this.registrars == null) {
            // 获取所有的错误页
            this.registrars = new ArrayList<>(this.beanFactory.getBeansOfType(
                    ErrorPageRegistrar.class, false, false).values());
            this.registrars.sort(AnnotationAwareOrderComparator.INSTANCE);
            this.registrars = Collections.unmodifiableList(this.registrars);
        }
        return this.registrars;
    }

    ...
}

```

我们主要关注 `postProcessBeforeInitialization` 的流程：

1.  如果 bean 的类型是 `ErrorPageRegistry`，进行第 2 步
2.  获取 beanFactory 中所有 `ErrorPageRegistrar` 类型的 bean，遍历进行第 3 步
3.  调用 `registrar.registerErrorPages(registry)` 进行错误页操作

这里有两个类需要说明：

*   `ErrorPageRegistrar`：错误页注册器（让 `ErrorPageRegistry` 干活的类）
*   `ErrorPageRegistry`：错误的注册器类（实际干活的类）

如果我们想要自定义错误页，可以实现 `ErrorPageRegistry` 接口：

```
@Component
public class MyErrorPage implements ErrorPageRegistrar {

    /**
     * 注册错误页
     */
    @Override
    public void registerErrorPages(ErrorPageRegistry errorPageRegistry) {
        // 最终调用的是 ErrorPageRegistry#addErrorPages 进行注册
        errorPageRegistry.addErrorPages(new ErrorPage("/error/page"));
    }
}

```

#### 2.2 `EmbeddedTomcat`

进入 `ServletWebServerFactoryConfiguration.EmbeddedTomcat` 类：

```
@Configuration(proxyBeanMethods = false)
class ServletWebServerFactoryConfiguration {

    @Configuration(proxyBeanMethods = false)
    // 条件注解，条件存在时才引入
    @ConditionalOnClass({ Servlet.class, Tomcat.class, UpgradeProtocol.class })
    // 注意 ServletWebServerFactory，我们可以自主实现 tomcat 容器的装配
    @ConditionalOnMissingBean(value = ServletWebServerFactory.class, 
        search = SearchStrategy.CURRENT)
    public static class EmbeddedTomcat {

        @Bean
        public TomcatServletWebServerFactory tomcatServletWebServerFactory(
                ObjectProvider<TomcatConnectorCustomizer> connectorCustomizers,
                ObjectProvider<TomcatContextCustomizer> contextCustomizers,
                ObjectProvider<TomcatProtocolHandlerCustomizer<?>> protocolHandlerCustomizers) {
            TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
            // 配置一些参数
            factory.getTomcatConnectorCustomizers()
                    .addAll(connectorCustomizers.orderedStream().collect(Collectors.toList()));
            factory.getTomcatContextCustomizers()
                    .addAll(contextCustomizers.orderedStream().collect(Collectors.toList()));
            factory.getTomcatProtocolHandlerCustomizers()
                    .addAll(protocolHandlerCustomizers.orderedStream()
                    .collect(Collectors.toList()));
            return factory;
        }

    }

    ...
}

```

这个类主要就是返回 `TomcatServletWebServerFactory` bean，可以注入一些 `connectorCustomizers`、`contextCustomizers`、`protocolHandlerCustomizers` 等参数进行自定义配置，这些参数就是从 `BeanPostProcessorsRegistrar` 中来的。

这里有个地方需要提一下，如果不想使用 springboot 提供的 `TomcatServletWebServerFactory`，我们可以自己实现 `TomcatServletWebServerFactory`，像这样：

```
@Bean
public ServletWebServerFactory servletWebServerFactory() {
    TomcatServletWebServerFactory tomcat = new TomcatServletWebServerFactory();
    // 处理自定义的各种配置
    ...
    return tomcat;
}

```

这样处理之后，springboot 提供的 `tomcatServletWebServerFactory` 就不会处理了。

其他两个类 `EmbeddedJetty`、`EmbeddedUndertow`，跟 `EmbeddedTomcat` 的处理基本相似，就不多说了。

### 3. `DispatcherServletAutoConfiguration`

我们再来看看 `DispatcherServletAutoConfiguration`，关键代码如下：

```
public class DispatcherServletAutoConfiguration {

    public static final String DEFAULT_DISPATCHER_SERVLET_BEAN_NAME = "dispatcherServlet";

    public static final String DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME 
            = "dispatcherServletRegistration";

    @Configuration(proxyBeanMethods = false)
    @Conditional(DefaultDispatcherServletCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties({ HttpProperties.class, WebMvcProperties.class })
    protected static class DispatcherServletConfiguration {

        /**
         * 大名鼎鼎的 DispatcherServlet.
         * @param httpProperties http属性.
         * @param webMvcProperties webMvc 属性.
         * @return 返回对象
         */
        @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServlet dispatcherServlet(HttpProperties httpProperties, 
                WebMvcProperties webMvcProperties) {
            DispatcherServlet dispatcherServlet = new DispatcherServlet();
            dispatcherServlet.setDispatchOptionsRequest(
                    webMvcProperties.isDispatchOptionsRequest());
            dispatcherServlet.setDispatchTraceRequest(webMvcProperties.isDispatchTraceRequest());
            dispatcherServlet.setThrowExceptionIfNoHandlerFound(
                    webMvcProperties.isThrowExceptionIfNoHandlerFound());
            dispatcherServlet.setPublishEvents(webMvcProperties.isPublishRequestHandledEvents());
            dispatcherServlet.setEnableLoggingRequestDetails(httpProperties.isLogRequestDetails());
            return dispatcherServlet;
        }

        /**
         * 文件上传组件.
         * @param resolver 参数.
         * @return 返回值.
         */
        @Bean
        @ConditionalOnBean(MultipartResolver.class)
        @ConditionalOnMissingBean(name = DispatcherServlet.MULTIPART_RESOLVER_BEAN_NAME)
        public MultipartResolver multipartResolver(MultipartResolver resolver) {
            return resolver;
        }

    }

    /**
     * 生成 DispatcherServletRegistrationBean
     * 它会将dispatcherServlet注册到servlet容器
     */
    @Configuration(proxyBeanMethods = false)
    @Conditional(DispatcherServletRegistrationCondition.class)
    @ConditionalOnClass(ServletRegistration.class)
    @EnableConfigurationProperties(WebMvcProperties.class)
    @Import(DispatcherServletConfiguration.class)
    protected static class DispatcherServletRegistrationConfiguration {

        @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
        @ConditionalOnBean(value = DispatcherServlet.class, 
                name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServletRegistrationBean dispatcherServletRegistration(
                DispatcherServlet dispatcherServlet, WebMvcProperties webMvcProperties, 
                ObjectProvider<MultipartConfigElement> multipartConfig) {
            // 生成 DispatcherServletRegistrationBean（它会将dispatcherServlet注册到servlet容器）
            DispatcherServletRegistrationBean registration = 
                    new DispatcherServletRegistrationBean(dispatcherServlet, 
                    webMvcProperties.getServlet().getPath());
            registration.setName(DEFAULT_DISPATCHER_SERVLET_BEAN_NAME);
            registration.setLoadOnStartup(webMvcProperties.getServlet().getLoadOnStartup());
            multipartConfig.ifAvailable(registration::setMultipartConfig);
            return registration;
        }

    }

    ...

}

```

它主要是注册了 3 个 bean：

*   `dispatcherServlet`：springMvc 的请求入口，url 请求由此进入，然后转化到 `requestMapping`
*   `multipartResolver`：处理文件上传
*   `dispatcherServletRegistration`：处理 `dispatcherServlet` 的注册，它会将 `dispatcherServlet` 注册到 servlet 容器中（关于 springboot 注册 servlet 组件的内容，可以参考 [springboot web 应用之 servlet 组件的注册流程](https://my.oschina.net/funcy/blog/4951050)）

### 4. `WebMvcAutoConfiguration`

继续看 `WebMvcAutoConfiguration`:

```
@Configuration(proxyBeanMethods = false)
...
// 如果没有自定义WebMvc的配置类，则使用本配置
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)
...
public class WebMvcAutoConfiguration {
    ...
}

```

`WebMvcAutoConfiguration` 上有个注解需要注意下：

```
@ConditionalOnMissingBean(WebMvcConfigurationSupport.class)

```

这行代码表明了 `WebMvcAutoConfiguration` 只有在 `WebMvcConfigurationSupport` 类型的 bean 不存在时，它才会进行自动装配。

`WebMvcConfigurationSupport` 是个啥呢？在 [springmvc demo 与 @EnableWebMvc 注解](https://my.oschina.net/funcy/blog/4696657)一文中，我们提到引入 springMvc 组件的两种方式：

*   使用 `@EnableWebMvc` 注解
*   继承 `WebMvcConfigurationSupport` 类

而这种方法最终都会向 spring 容器中引入类型为 `WebMvcConfigurationSupport` 的 bean，因此在 `springboot` 项目中，如果我们进行了以上两种操作之一，那么就引入了 `WebMvcConfigurationSupport`，`WebMvcAutoConfiguration` 的自动装配就不执行了。

我们再来看看 `WebMvcAutoConfiguration` 装配的 bean。

#### 4.1 `WebMvcAutoConfigurationAdapter`

`WebMvcAutoConfigurationAdapter` 是 `WebMvcAutoConfiguration` 的内部类，定义如下：

```
@Configuration(proxyBeanMethods = false)
// 引入了 EnableWebMvcConfiguration
@Import(EnableWebMvcConfiguration.class)
@EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
@Order(0)
public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
    ...
}

```

它实现了 `WebMvcConfigurer`，且引入了 `EnableWebMvcConfiguration`。`WebMvcConfigurer` 可以用来处理 springMvc 的配置，只需要重写其中对应的方法即可，`EnableWebMvcConfiguration` 从名称来看，是 “启用 webMvc 配置”，它也是 `WebMvcAutoConfiguration` 的内部类，我们来看看它做了啥：

```
@Configuration(proxyBeanMethods = false)
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration 
        implements ResourceLoaderAware {
    ...
}

```

可以看到，它是 `DelegatingWebMvcConfiguration` 的子类，而 `DelegatingWebMvcConfiguration` 又是个啥呢？我们也来看看它的定义：

```
@Configuration(proxyBeanMethods = false)
public class DelegatingWebMvcConfiguration extends WebMvcConfigurationSupport {
    ...
}

```

`DelegatingWebMvcConfiguration` 是由 springMvc 提供的，可以看到，它实现了 `WebMvcConfigurationSupport`，因此，springboot 通过 `@Import(EnableWebMvcConfiguration.class)` 的方式向 spring 容器中引入了 `WebMvcConfigurationSupport` 类型的 bean。

事实上，`DelegatingWebMvcConfiguration` 正是 `@EnableWebMvc` 引入的 bean：

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
// 引入了 DelegatingWebMvcConfiguration
@Import(DelegatingWebMvcConfiguration.class)
public @interface EnableWebMvc {
    ...
}

```

springbot 不直接引入 `DelegatingWebMvcConfiguration` 而是引入它的子类 `EnableWebMvcConfiguration`，虽然它是做了一些自定义配置的。

关于 `EnableWebMvcConfiguration` 配置了些啥，我们一会再分析，继续分析 `WebMvcAutoConfigurationAdapter` 的引入 bean：

```
/**
 * http 消息转换器.
 */
@Override
public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    this.messageConvertersProvider.ifAvailable(
            (customConverters) -> converters.addAll(customConverters.getConverters()));
}

/**
 * 视图解析器.
 */
@Bean
@ConditionalOnMissingBean
public InternalResourceViewResolver defaultViewResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix(this.mvcProperties.getView().getPrefix());
    resolver.setSuffix(this.mvcProperties.getView().getSuffix());
    return resolver;
}

@Bean
@ConditionalOnBean(View.class)
@ConditionalOnMissingBean
public BeanNameViewResolver beanNameViewResolver() {
    BeanNameViewResolver resolver = new BeanNameViewResolver();
    resolver.setOrder(Ordered.LOWEST_PRECEDENCE - 10);
    return resolver;
}

@Bean
@ConditionalOnBean(ViewResolver.class)
@ConditionalOnMissingBean(name = "viewResolver", value = ContentNegotiatingViewResolver.class)
public ContentNegotiatingViewResolver viewResolver(BeanFactory beanFactory) {
    ContentNegotiatingViewResolver resolver = new ContentNegotiatingViewResolver();
    resolver.setContentNegotiationManager(beanFactory.getBean(ContentNegotiationManager.class));
    resolver.setOrder(Ordered.HIGHEST_PRECEDENCE);
    return resolver;
}

/**
 * 引入国际化配置.
 */
@Bean
@ConditionalOnMissingBean
@ConditionalOnProperty(prefix = "spring.mvc", name = "locale")
public LocaleResolver localeResolver() {
    if (this.mvcProperties.getLocaleResolver() == WebMvcProperties.LocaleResolver.FIXED) {
        return new FixedLocaleResolver(this.mvcProperties.getLocale());
    }
    AcceptHeaderLocaleResolver localeResolver = new AcceptHeaderLocaleResolver();
    localeResolver.setDefaultLocale(this.mvcProperties.getLocale());
    return localeResolver;
}

/**
 * 静态资源映射
 */
@Override
public void addResourceHandlers(ResourceHandlerRegistry registry) {
    if (!this.resourceProperties.isAddMappings()) {
        logger.debug("Default resource handling disabled");
        return;
    }
    // 映射webjars
    Duration cachePeriod = this.resourceProperties.getCache().getPeriod();
    CacheControl cacheControl = this.resourceProperties.getCache()
            .getCachecontrol().toHttpCacheControl();
    if (!registry.hasMappingForPattern("/webjars/**")) {
        customizeResourceHandlerRegistration(registry.addResourceHandler("/webjars/**")
                .addResourceLocations("classpath:/META-INF/resources/webjars/")
                .setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
    }
    // 映射静态资源路径
    String staticPathPattern = this.mvcProperties.getStaticPathPattern();
    if (!registry.hasMappingForPattern(staticPathPattern)) {
        customizeResourceHandlerRegistration(registry.addResourceHandler(staticPathPattern)
                // staticLocations 默认为 classpath:[/META-INF/resources/,
                // /resources/, /static/, /public/]
                .addResourceLocations(getResourceLocations(
                    this.resourceProperties.getStaticLocations()))
                .setCachePeriod(getSeconds(cachePeriod)).setCacheControl(cacheControl));
    }
}

```

可以看到，`WebMvcAutoConfigurationAdapter` 引入了三类 bean:

1.  http 消息转换器
2.  视图解析器
3.  国际化配置

这里我们重点来看看 `http消息转换器`。在使用 springMvc 时，通过在 `Controller` 类或其中的方法上标记 `@ResponseBody`，返回参数就会转换为 json，这就是 `http消息转换器`所做的工作了。springboot 默认的 jaon 处理是 `jackson`，它的实例化在 `JacksonAutoConfiguration` 中，装配为 `HttpMessageConverter` 的处理在 `JacksonHttpMessageConvertersConfiguration`，这里就不展开了。

在上面 的分析中，多次出现了 `WebMvcProperties` 配置，我们来看看它是个啥：

```
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
    ...
}

```

很明显，它是个配置类 ，可以看到，它的配置前缀为 `spring.mvc`，支持的配置如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-062f490afbb74e496f50f4cad4bd13ede82.png)

#### 4.2 `EnableWebMvcConfiguration`

我们来看看 `EnableWebMvcConfiguration` 自定义的配置：

```
public static class EnableWebMvcConfiguration extends DelegatingWebMvcConfiguration 
        implements ResourceLoaderAware {

    /**
     * 配置适配器
     */
    @Bean
    @Override
    public RequestMappingHandlerAdapter requestMappingHandlerAdapter(
            @Qualifier("mvcContentNegotiationManager") 
                ContentNegotiationManager contentNegotiationManager,
            @Qualifier("mvcConversionService") FormattingConversionService conversionService,
            @Qualifier("mvcValidator") Validator validator) {
        RequestMappingHandlerAdapter adapter = super.requestMappingHandlerAdapter(
                contentNegotiationManager, conversionService, validator);
        adapter.setIgnoreDefaultModelOnRedirect(this.mvcProperties == null 
                || this.mvcProperties.isIgnoreDefaultModelOnRedirect());
        return adapter;
    }

    /**
     * 路径映射
     */
    @Bean
    @Primary
    @Override
    public RequestMappingHandlerMapping requestMappingHandlerMapping(
            @Qualifier("mvcContentNegotiationManager") 
                    ContentNegotiationManager contentNegotiationManager,
            @Qualifier("mvcConversionService") FormattingConversionService conversionService,
            @Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {
        // Must be @Primary for MvcUriComponentsBuilder to work
        return super.requestMappingHandlerMapping(contentNegotiationManager, conversionService,
                resourceUrlProvider);
    }

    /**
     * 欢迎页
     */
    @Bean
    public WelcomePageHandlerMapping welcomePageHandlerMapping(
            ApplicationContext applicationContext, 
            FormattingConversionService mvcConversionService, 
            ResourceUrlProvider mvcResourceUrlProvider) {
        WelcomePageHandlerMapping welcomePageHandlerMapping = new WelcomePageHandlerMapping(
                new TemplateAvailabilityProviders(applicationContext), applicationContext, 
                getWelcomePage(), this.mvcProperties.getStaticPathPattern());
        welcomePageHandlerMapping.setInterceptors(getInterceptors(
                mvcConversionService, mvcResourceUrlProvider));
        return welcomePageHandlerMapping;
    }

    /**
     * mvc配置，添加了日期格式的处理
     */
    @Bean
    @Override
    public FormattingConversionService mvcConversionService() {
        WebConversionService conversionService 
            = new WebConversionService(this.mvcProperties.getDateFormat());
        addFormatters(conversionService);
        return conversionService;
    }

    /**
     * 参数校验器
     */
    @Bean
    @Override
    public Validator mvcValidator() {
        if (!ClassUtils.isPresent("javax.validation.Validator", getClass().getClassLoader())) {
            return super.mvcValidator();
        }
        return ValidatorAdapter.get(getApplicationContext(), getValidator());
    }

    /**
     * 异常处理
     */
    @Override
    protected ExceptionHandlerExceptionResolver createExceptionHandlerExceptionResolver() {
        if (this.mvcRegistrations != null 
                && this.mvcRegistrations.getExceptionHandlerExceptionResolver() != null) {
            return this.mvcRegistrations.getExceptionHandlerExceptionResolver();
        }
        return super.createExceptionHandlerExceptionResolver();
    }

    @Override
    protected void extendHandlerExceptionResolvers(
                List<HandlerExceptionResolver> exceptionResolvers) {
        super.extendHandlerExceptionResolvers(exceptionResolvers);
        if (this.mvcProperties.isLogResolvedException()) {
            for (HandlerExceptionResolver resolver : exceptionResolvers) {
                if (resolver instanceof AbstractHandlerExceptionResolver) {
                    ((AbstractHandlerExceptionResolver) resolver).setWarnLogCategory(
                            resolver.getClass().getName());
                }
            }
        }
    }
    ...
}

```

相比于原生的 springMvc，它额外配置了适配器、路径映射、参数校验、异常处理等。

### 5\. 总结

1.  `webMvc` 在装配时，先装配 `ServletWebServerFactoryAutoConfiguration`，再装配 `DispatcherServletAutoConfiguration`，最后装配 `WebMvcAutoConfiguration`
2.  `ServletWebServerFactoryAutoConfiguration` 处理 servlet 容器的装配，`DispatcherServletAutoConfiguration` 处理 `DispatcherServlet` 的装配，`WebMvcAutoConfiguration` 处理 `webMvc` 组件（消息转换器、视图解析器、静态资源映射等）的装配
3.  如果向 spring 容器中引入了 `WebMvcConfigurationSupport`， `WebMvcAutoConfiguration` 的装配操作将不会执行
4.  `servlet` 的相关配置以 `servlet` 为前缀，`webMvc` 的配置以 `spring.mvc` 为前缀，我们可以在配置文件（一般为 `application.properties/application.yml`）中进行配置

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4921595](https://my.oschina.net/funcy/blog/4921595) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_