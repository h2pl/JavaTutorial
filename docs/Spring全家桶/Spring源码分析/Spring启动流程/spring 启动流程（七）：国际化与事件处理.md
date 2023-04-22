![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-ed6b80d76ba4b2ddb0f8d15e070a0c32df7.png)

接上文，我们继续分析 spring 的启动流程。

### 7\. 国际化: `initMessageSource()`

这个方法是用来初始化 `MessageSource` 的，内容如下：

```
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {

    ...

    /**
     * 初始化 MessageSource
     */
    protected void initMessageSource() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        // 如果beanFactory中存在MessageSource，设置其 ParentMessageSource
        if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
            this.messageSource = beanFactory.getBean(
                    MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
            if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
                HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
                if (hms.getParentMessageSource() == null) {
                    // 设置ParentMessageSource
                    hms.setParentMessageSource(getInternalParentMessageSource());
                }
            }
        }
        // 如果beanFactory中不存在MessageSource，就 创建-设置-注册
        else {
            DelegatingMessageSource dms = new DelegatingMessageSource();
            dms.setParentMessageSource(getInternalParentMessageSource());
            this.messageSource = dms;
            // 设置ParentMessageSource
            beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
        }
    }

    /**
     * 返回父容器的 messageSource
     */
    @Nullable
    protected MessageSource getInternalParentMessageSource() {
        return (getParent() instanceof AbstractApplicationContext ?
                ((AbstractApplicationContext) getParent()).messageSource : getParent());
    }

    ...
}

```

可以看到，整个方法主要是操作 `MessageSource`，主要逻辑为：如果已经存在 `MessageSource` 了，就设置一些属性；否则就创建 `MessageSource`，并设置些 属性，最后注册到 `beanFactory` 中。

关于 `MessageSource` 的具体作用，本文就不展开了。

### 8\. 初始化事件广播器：`initApplicationEventMulticaster()`

`AbstractApplicationContext#initApplicationEventMulticaster` 代码如下：

```
protected void initApplicationEventMulticaster() {
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 如果用户配置了自定义事件广播器，就使用用户的
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
                beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, 
                        ApplicationEventMulticaster.class);
    }
    else {
        // 用户没有配置广播器，就使用默认的事件广播器
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, 
                this.applicationEventMulticaster);
    }
}

```

这块逻辑也很简单，如果已存在事件广播器，就使用已存在的，否则就创建一个。关于 `ApplicationEventMulticaster`，主要就是用来广播事件的，更多关于事件的内容，可以参考 [spring 探秘之 spring 事件机制](https://my.oschina.net/funcy/blog/4713339).

### 9\. 扩展点：onRefresh ()

`AbstractApplicationContext#onRefresh` 是 spring 提供的一个扩展点，方法并无内容：

```
protected void onRefresh() throws BeansException {

}

```

如果需要处理特定的操作，可以在子类中实现。

当前使用的 `ApplicationContext` 是 `AnnotationConfigApplicationContext`，并无 `onRefresh()` 方法，就不过多分析了。

### 10\. 注册事件监听器：registerListeners ()

`AbstractApplicationContext#registerListeners` 相关代码如下：

> AbstractApplicationContext

```
/** 这里就是用来存放监听器的 */
private final Set<ApplicationListener<?>> applicationListeners = new LinkedHashSet<>();

/** 返回当前所有的监听器 */
public Collection<ApplicationListener<?>> getApplicationListeners() {
    return this.applicationListeners;
}

/**
 * 添加监听器
 */
public void addApplicationListener(ApplicationListener<?> listener) {
    Assert.notNull(listener, "ApplicationListener must not be null");
    if (this.applicationEventMulticaster != null) {
        this.applicationEventMulticaster.addApplicationListener(listener);
    }
    this.applicationListeners.add(listener);
}

/**
 * 注册监听器
 */
protected void registerListeners() {
    // 先添加手动set的一些监听器
    // getApplicationListeners() 获取的监听器基本是通过调用 addApplicationListener(...) 添加的
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }
    // 获取取到监听器的名称，设置到广播器
    // 此时获取的监听器是从 beanFactory 中获取的，即是spring通过包扫描得到的
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }
    // 如果存在早期应用事件，广播
    Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
    this.earlyApplicationEvents = null;
    if (earlyEventsToProcess != null) {
        for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
            //  广播早期事件
            getApplicationEventMulticaster().multicastEvent(earlyEvent);
        }
    }
}

```

这个方法的流程大致如下：

1.  添加 `AbstractApplicationContext#applicationListeners` 中的监听器到 `ApplicationEventMulticaster` 中
2.  从 `beanFactory` 获取监听器的 `beanName`，添加到 `ApplicationEventMulticaster` 中
3.  如果有早期事件，就进行广播

关于 spring 的事件，本文并不打算展开，如果想了解更多，可参考 [spring 探秘之 spring 事件机制](https://my.oschina.net/funcy/blog/4713339).

本文的分析就到这里了。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4892120](https://my.oschina.net/funcy/blog/4892120) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_