完成包的扫描后，接着就开始了 spring 的启动了，即 `AbstractApplicationContext#refresh` 方法，该方法一共包含 13 个操作，涵盖也 spring 启动的整个流程：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-88c3a2486c24ccd0ad390ba9b62b986a6b2.png)

本系列从本文开始，逐步分析这 13 个方法，探索 spring 的启动流程。

### 1\. 启动前准备：`prepareRefresh()`

跟进 `prepareRefresh()` 方法，调用链如下：

```
|-AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#prepareRefresh

```

代码如下：

```
protected void prepareRefresh() {
    // Switch to active.
    this.startupDate = System.currentTimeMillis();
    this.closed.set(false);
    this.active.set(true);

    // 初始化加载配置文件方法，并没有具体实现，一个留给用户的扩展点
    initPropertySources();

    // 检查环境变量
    getEnvironment().validateRequiredProperties();

    if (this.earlyApplicationListeners == null) {
        this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
    } else {
        this.applicationListeners.clear();
        this.applicationListeners.addAll(this.earlyApplicationListeners);
    }

    this.earlyApplicationEvents = new LinkedHashSet<>();
}

```

这段代码比较简单，就是设置了下启动时间、容器的启动状态、环境变量的检查、属性的初始化等。

### 2\. 获取 `beanFactory: obtainFreshBeanFactory()`

我们再跟进 `obtainFreshBeanFactory()` 方法，内容如下：

> AbstractApplicationContext#obtainFreshBeanFactory

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    refreshBeanFactory();
    // 返回刚刚创建的 BeanFactory
    return getBeanFactory();
}

```

这个方法先是 `refresh` 了 `BeanFactory`，然后再返回了 `BeanFactory`，我们继续跟进 `refreshBeanFactory()`：

> GenericApplicationContext#refreshBeanFactory

```
@Override
protected final void refreshBeanFactory() throws IllegalStateException {
    // 省略了一些判断代码
    this.beanFactory.setSerializationId(getId());
}

```

这个方法关键代码只有一行，作用是用来设置 beanFactory 的 `SerializationId`.

我们再回过头来看看 `getBeanFactory()` 方法：

> GenericApplicationContext#getBeanFactory

```
public final ConfigurableListableBeanFactory getBeanFactory() {
    return this.beanFactory;
}

```

这个方法就更简单了，仅仅返回了当前类的 `beanFactory`，这个 `beanFactory` 就是我们在分析 `AnnotationConfigApplicationContext` 构造方法时创建的，类型为 `DefaultListableBeanFactory`.

### 3\. 准备 `beanFactory: prepareBeanFactory(beanFactory)`

我们继续，进入 `prepareBeanFactory(beanFactory)` 方法：

> AbstractApplicationContext#prepareBeanFactory

```
protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 设置为加载当前ApplicationContext类的类加载器
    beanFactory.setBeanClassLoader(getClassLoader());
    // 设置 BeanExpressionResolver——bean表达式解析器
    beanFactory.setBeanExpressionResolver(
            new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
    // 属性编辑器支持
    beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

    // 这里是Spring的又一个扩展点
    // 在所有实现了Aware接口的bean在初始化的时候，这个 processor负责回调，
    // 这个我们很常用，如我们会为了获取 ApplicationContext 而 implement ApplicationContextAware
    // 注意：它不仅仅回调 ApplicationContextAware，还会负责回调 EnvironmentAware、ResourceLoaderAware 等
    beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));

    // 下面几行的意思就是，如果某个 bean 依赖于以下几个接口的实现类，
    // 在自动装配的时候忽略它们，Spring 会通过其他方式来处理这些依赖。
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

    // 下面几行就是为特殊的几个 bean 赋值，如果有 bean 依赖了以下几个，会注入这边相应的值
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);

     // 添加一个后置处理器：ApplicationListenerDetector，此后置处理器实现了BeanPostProcessor接口
     beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

    // 如果存在bean名称为loadTimeWeaver的bean则注册一个BeanPostProcessor
    if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
        beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
        // Set a temporary ClassLoader for type matching.
        beanFactory.setTempClassLoader(
                new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
    }

    // 如果没有定义 "environment" 这个 bean，那么 Spring 会 "手动" 注册一个
    if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
    }
    // 如果没有定义 "systemProperties" 这个 bean，那么 Spring 会 "手动" 注册一个
    if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, 
                getEnvironment().getSystemProperties());
    }
    // 如果没有定义 "systemEnvironment" 这个 bean，那么 Spring 会 "手动" 注册一个
    if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
        beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, 
                getEnvironment().getSystemEnvironment());
    }
}

```

这个方法是对 beanFactory 的一些准备，设置一些属性，添加一些 bean 处理等，代码都有注解，这里就不重复说了。

关于往 beanFactory 中添加 `ApplicationListenerDetector` 需要提下，相关代码为 `beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));`，我们看下 `ApplicationListenerDetector` 这个类：

> org.springframework.context.support.ApplicationContextAwareProcessor

```
class ApplicationContextAwareProcessor implements BeanPostProcessor {
    @Override
    @Nullable
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    // 省略了一些代码
    AccessControlContext acc = null;
    if (System.getSecurityManager() != null) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }
    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
        }, acc);
    } else {
        invokeAwareInterfaces(bean);
    }
        return bean;
    }

    // 回调 Aware接口
    private void invokeAwareInterfaces(Object bean) {
        // 调用 EnvironmentAware#setEnvironment 方法
        if (bean instanceof EnvironmentAware) {
            ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
        }
        // 调用 EmbeddedValueResolverAware#setEmbeddedValueResolver 方法
        if (bean instanceof EmbeddedValueResolverAware) {
            ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
        }
        // 调用 ResourceLoaderAware#setResourceLoader 方法
        if (bean instanceof ResourceLoaderAware) {
            ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
        }
        // 调用 ApplicationEventPublisherAware#setApplicationEventPublisher 方法
        if (bean instanceof ApplicationEventPublisherAware) {
            ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
        }
        // 调用 MessageSourceAware#setMessageSource 方法
        if (bean instanceof MessageSourceAware) {
            ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
        }
        // 调用 ApplicationContextAware#setApplicationContext 方法
        if (bean instanceof ApplicationContextAware) {
            ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
        }
    }

    // 省略其他代码
}

```

可以看到，

1.  这个类实现了 `BeanPostProcessor` 接口；
2.  方法 `postProcessBeforeInitialization` 由 `BeanPostProcessor` 提供，最为关键的代码是` invokeAwareInterfaces(bean);`；
3.  `invokeAwareInterfaces` 只是一系列的方法调用

关于 `BeanPostProcessor` 的的分析，可以参考 [spring 组件之 BeanPostProcessors ](https://my.oschina.net/funcy/blog/4597551)，关于该类的作用，后续会继续讲到。

好了，本文的分析就到这里了，本文仅分析了 spring 启动时对 beanFactory 的准备，内容较简单，最后用一幅图来总结下本文内容：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1e10d7aff080b2e0bbfbef5d79c56cc54c9.png)

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4633169](https://my.oschina.net/funcy/blog/4633169) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_