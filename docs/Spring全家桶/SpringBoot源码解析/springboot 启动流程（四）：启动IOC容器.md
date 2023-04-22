上一篇文章总结 springboot 启动流程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-07a6b491fbe69b8dcbd41e59a8543f06671.png)

接上文，我们继续分析接下来的步骤。

### 3.10 刷新 ioc 容器

接下来我们来看看 `SpringApplication#refreshContext` 方法：

```
private void refreshContext(ConfigurableApplicationContext context) {
    // 启动spring容器
    refresh(context);
    if (this.registerShutdownHook) {
        try {
            // 注册 ShutdownHook
            context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
            // Not allowed in some environments.
        }
    }
}

```

这个方法操作就两个：

1.  `refresh(context)`：启动 spring 容器，也不是调用 `AbstractApplicationContext#refresh` 方法；
2.  `context.registerShutdownHook()`：注册 `ShutdownHook`，可以在 jvm 进程关闭时处理一些特定的操作。

#### 3.10.1 启动 spring 容器

进入 `SpringApplication#refresh`：

```
protected void refresh(ApplicationContext applicationContext) {
    Assert.isInstanceOf(AbstractApplicationContext.class, applicationContext);
    // spring 容器的启动操作了
    ((AbstractApplicationContext) applicationContext).refresh();
}

```

这个方法很简单，先判断 `applicationContext` 的类型是否为 `AbstractApplicationContext`，然后再调用 `AbstractApplicationContext#refresh()`。

关于 `AbstractApplicationContext#refresh()`，那可是大名鼎鼎啊，该方法涵盖了 spring 容器的启动流程。由于本文不是分析 spring 的文章，因此这块就不展开分析了，想要了解的启动流程的小伙伴可以参考以下文章：

*   [【spring 源码分析】spring 启动流程（四）：启动前的准备工作](https://my.oschina.net/funcy/blog/4633169)
*   [【spring 源码分析】spring 启动流程（五）：执行 BeanFactoryPostProcessor](https://my.oschina.net/funcy/blog/4641114)
*   [【spring 源码分析】spring 启动流程（六）：注册 BeanPostProcessor](https://my.oschina.net/funcy/blog/4657181)
*   [【spring 源码分析】spring 启动流程（七）：国际化与事件处理](https://my.oschina.net/funcy/blog/4892120)
*   [【spring 源码分析】spring 启动流程（八）：完成 BeanFactory 的初始化](https://my.oschina.net/funcy/blog/4658230)
*   [【spring 源码分析】spring 启动流程（九）：单例 bean 的创建](https://my.oschina.net/funcy/blog/4659524)
*   [【spring 源码分析】spring 启动流程（十）：启动完成的处理](https://my.oschina.net/funcy/blog/4892555)

在 `AbstractApplicationContext#refresh()` 中，spring 提供了几个扩展点：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b86d0fb2e3790f63b5c6590884be9401354.png)

我们当前使用的 `applicationContext` 为 `AnnotationConfigServletWebServerApplicationContext`，其中也使用了这些扩展点，我们主要关注这些扩展点的应用。

##### 1\. 启动前准备：`prepareRefresh()`

经过调试发现，`initPropertySources()` 方法会运行到，调用链如下：

```
AbstractApplicationContext#refresh
 |- AnnotationConfigServletWebServerApplicationContext#prepareRefresh
  |- AbstractApplicationContext#prepareRefresh
   |- GenericWebApplicationContext#initPropertySources

```

最终调用的是 `GenericWebApplicationContext#initPropertySources`：

```
protected void initPropertySources() {
    ConfigurableEnvironment env = getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(this.servletContext, null);
    }
}

```

这个方法里先获取 `Environment`，然后判断是否为 `ConfigurableWebEnvironment` 的实例，在前面分析**准备运行时环境**时，我们得到的 `Environment` 为 `StandardServletEnvironment`，是 `ConfigurableWebEnvironment` 的符合，然后调用 `ConfigurableWebEnvironment#initPropertySources` 方法，结果到了 `StandardServletEnvironment#initPropertySources`：

```
public void initPropertySources(@Nullable ServletContext servletContext, 
        @Nullable ServletConfigservletConfig) {
    // 替换上面设置的 servletContextInitParams 为 servletContext
    // 替换上面设置的 servletConfigInitParams 为 servletConfig
    WebApplicationContextUtils.initServletPropertySources(getPropertySources(), 
        servletContext, servletConfig);
}

```

这个方法还是很简单，只是将 `servletContext` 与 `servletConfig` 设置到了 `Environment` 中。

##### 2\. 获取 `beanFactory`: `obtainFreshBeanFactory()`

当前 `applicationContext` 对该方法无扩展，不分析。

##### 3\. 准备 `beanFactory`: `prepareBeanFactory(beanFactory)`

当前 `applicationContext` 对该方法无扩展，不分析。

##### 4\. 扩展点：`postProcessBeanFactory(beanFactory)`

`AnnotationConfigServletWebServerApplicationContext` 重写了这个方法：

```
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 调用父类的方法
    super.postProcessBeanFactory(beanFactory);
    // 进行包扫描，这里的包并不存在
    if (this.basePackages != null && this.basePackages.length > 0) {
        this.scanner.scan(this.basePackages);
    }
    // 注册bean，为空
    if (!this.annotatedClasses.isEmpty()) {
        this.reader.register(ClassUtils.toClassArray(this.annotatedClasses));
    }
}

```

这个方法的执行过程如下：

1.  调用了父类的方法 `super.postProcessBeanFactory(beanFactory)`
2.  进行包扫描，通过调试发现，这里的 `basePackages` 为 nul
3.  注册 `annotatedClasses`，这里的 `annotatedClasses` 为空

我们主要来看看 `super.postProcessBeanFactory(beanFactory)`，该方法在 `ServletWebServerApplicationContext` 中：

```
@Override
protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 添加一个 BeanPostProcessor
    beanFactory.addBeanPostProcessor(
            new WebApplicationContextServletContextAwareProcessor(this));
    // 忽略 ServletContextAware 的自动注入
    beanFactory.ignoreDependencyInterface(ServletContextAware.class);
    // 注册 web bean 的范围，这里会注册request、session、globalSession的作用域
    registerWebApplicationScopes();
}

```

这个方法内容比较简单，主要是注册 `BeanPostProcessor` 以及注册 `web bean` 的作用范围。这里我们主要看下 `WebApplicationContextServletContextAwareProcessor` 的作用，代码如下：

```
public class WebApplicationContextServletContextAwareProcessor 
        extends ServletContextAwareProcessor {

    private final ConfigurableWebApplicationContext webApplicationContext;

    public WebApplicationContextServletContextAwareProcessor(
            ConfigurableWebApplicationContext webApplicationContext) {
        Assert.notNull(webApplicationContext, "WebApplicationContext must not be null");
        this.webApplicationContext = webApplicationContext;
    }

    /**
     * 获取 ServletContext
     */
    @Override
    protected ServletContext getServletContext() {
        ServletContext servletContext = this.webApplicationContext.getServletContext();
        return (servletContext != null) ? servletContext : super.getServletContext();
    }

    /**
     * 获取 ServletConfig
     */
    @Override
    protected ServletConfig getServletConfig() {
        ServletConfig servletConfig = this.webApplicationContext.getServletConfig();
        return (servletConfig != null) ? servletConfig : super.getServletConfig();
    }

}

```

这个类似乎并没有做什么，我们再跟进父类，由于它是个 `BeanPostProcessor`，我们主要关注它的 `postProcessBeforeInitialization()` 与 `postProcessAfterInitialization()` 两个方法：

```
public class ServletContextAwareProcessor implements BeanPostProcessor {

    ...

    public Object postProcessBeforeInitialization(Object bean, 
            String beanName) throws BeansException {
        // 设置 ServletContext
        if (getServletContext() != null && bean instanceof ServletContextAware) {
            ((ServletContextAware) bean).setServletContext(getServletContext());
        }
        // 设置 ServletConfig
        if (getServletConfig() != null && bean instanceof ServletConfigAware) {
            ((ServletConfigAware) bean).setServletConfig(getServletConfig());
        }
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        return bean;
    }

}

```

可以看到，这个 `BeanPostProcessor` 是用来处理 `ServletContextAware` 与 `ServletConfigAware` 两个 `Aware` 接口的，套路同处理 `ApplicationAware`、`BeanFactoryAware` 等一样。

##### 5\. 执行 `BeanFactoryPostProcessors`: `invokeBeanFactoryPostProcessors(beanFactory)`

当前 `applicationContext` 对该方法无扩展，不分析。

值得一提的是，在这个方法中，有个重的 `BeanFactoryPostProcessor` 会被执行：`ConfigurationClassPostProcessor`，springboot 的自动装配的启用注解 `@EnableAutoConfiguration` 会在这里处理，自动装配类的加载、条件注解也是在 `ConfigurationClassPostProcessor` 中。

##### 6\. 注册 `BeanPostProcessor`: `registerBeanPostProcessors(beanFactory)`

当前 `applicationContext` 对该方法无扩展，不分析。

##### 7\. 初始化` MessageSource`(用于国际化操作): `initMessageSource()`

当前 `applicationContext` 对该方法无扩展，不分析。

##### 8\. 初始化事件广播器：`initApplicationEventMulticaster()`

当前 `applicationContext` 对该方法无扩展，不分析。

##### 9\. 扩展点：`onRefresh()`

当前 `applicationContext` 对该方法的扩展为 `ServletWebServerApplicationContext#onRefresh` 方法，代码如下：

```
@Override
protected void onRefresh() {
    // 调用父类方法
    super.onRefresh();
    try {
        // 创建web服务器，如tomcat,jetty等
        createWebServer();
    }
    catch (Throwable ex) {
        throw new ApplicationContextException(...);
    }
}

```

可以 看到，web 服务器是在这个方法中创建的。不过 web 服务器的创建并不简单，需要经过多种条件判断，关于这点我们后面再详细说明。

##### 10\. 注册事件监听器：`registerListeners()`

当前 `applicationContext` 对该方法无扩展，不分析。

##### 11\. 初始化单例 `bean`: `finishBeanFactoryInitialization(beanFactory)`

当前 `applicationContext` 对该方法无扩展，不分析。

##### 12\. 完成启动操作: `finishRefresh()`

当前 `applicationContext` 对该方法的扩展为 `ServletWebServerApplicationContext#finishRefresh` 方法，代码如下：

```
@Override
protected void finishRefresh() {
    super.finishRefresh();
    // 启动web容器
    WebServer webServer = startWebServer();
    if (webServer != null) {
        // 发布 ServletWebServerInitializedEvent 事件
        publishEvent(new ServletWebServerInitializedEvent(webServer, this));
    }
}

/**
 * 启动web容器
 */
private WebServer startWebServer() {
    WebServer webServer = this.webServer;
    if (webServer != null) {
        webServer.start();
    }
    return webServer;
}

```

可以看到，这里才是真正启动 web 容器。

##### 13\. 清除缓存: `resetCommonCaches()`

当前 `applicationContext` 对该方法无扩展，不分析。

#### 3.10.2 注册 `ShutdownHook`

我们再来看 `context.registerShutdownHook()`，该方法由 `AbstractApplicationContext#registerShutdownHook` 提供：

```
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {
    ...
    @Override
    public void registerShutdownHook() {
        if (this.shutdownHook == null) {
            // 指定线程的名字
            this.shutdownHook = new Thread(SHUTDOWN_HOOK_THREAD_NAME) {
                @Override
                public void run() {
                    synchronized (startupShutdownMonitor) {
                        // 这里就是 ShutdownHook 的内容
                        doClose();
                    }
                }
            };
            Runtime.getRuntime().addShutdownHook(this.shutdownHook);
        }
    }

    /**
     * 处理容器的关闭操作
     */
    protected void doClose() {
        // Check whether an actual close attempt is necessary...
        if (this.active.get() && this.closed.compareAndSet(false, true)) {
            LiveBeansView.unregisterApplicationContext(this);

            try {
                // 发布关闭事件
                publishEvent(new ContextClosedEvent(this));
            }
            catch (Throwable ex) {
                logger.warn(...);
            }

            // 调用 lifecycle 的 onClose() 方法
            if (this.lifecycleProcessor != null) {
                try {
                    this.lifecycleProcessor.onClose();
                }
                catch (Throwable ex) {
                    logger.warn(...);
                }
            }

            // 销毁 bean
            destroyBeans();

            // 关闭容器
            closeBeanFactory();

            // 扩展点，待子类实现
            onClose();

            // 清除监听器
            if (this.earlyApplicationListeners != null) {
                this.applicationListeners.clear();
                this.applicationListeners.addAll(this.earlyApplicationListeners);
            }

            // 设置 active 标识
            this.active.set(false);
        }
    }

    ...

}

```

可以看到，`context.registerShutdownHook()` 实际上是运行了 `doClose()` 方法，用来处理容器的关闭操作。关闭 spring 容器的关闭，注释已经相当清楚了，这里就不深入了。

好了，容器的启动就分析到这里了，从流程上来讲，与 spring 容器启动的最大扩展在于 `onRefresh()` 与 `finishRefresh()`，前者创建了 `webServer` 容器，后者启动了 `webServer` 容器。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-81033ec78641ad875623cf452ef9cd62eb6.png)

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4888129](https://my.oschina.net/funcy/blog/4888129) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_