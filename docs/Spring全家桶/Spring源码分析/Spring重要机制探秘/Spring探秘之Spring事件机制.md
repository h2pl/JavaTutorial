### 1\. demo 准备

在正式分析事件机制前，我们先来准备一个 demo：

1.  准备一个事件类 `MyApplicationEvent`：

```
public class MyApplicationEvent extends ApplicationEvent {

    private static final long serialVersionUID = -1L;

    public MyApplicationEvent(Object source) {
        super(source);
    }
}

```

1.  准备一个监听器 `MyApplicationEventListener`，对事件 `MyApplicationEvent` 进行监听：

```
@Component
public class MyApplicationEventListener 
        implements ApplicationListener<MyApplicationEvent> {

    @Override
    public void onApplicationEvent(MyApplicationEvent event) {
        System.out.println(Thread.currentThread().getName() + " | " + event.getSource());
    }
}

```

准备一个配置类，先不指定内容：

```
@Configuration
@ComponentScan
public class Demo08Config {

}

```

最后是主类：

```
@ComponentScan
public class Demo08Main {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context 
            = new AnnotationConfigApplicationContext(Demo08Config.class);
        // 发布事件
        context.publishEvent(
            new MyApplicationEvent(Thread.currentThread().getName() + " | 自定义事件 ..."));
    }

}

```

以上代码定义了一个事件 `MyApplicationEvent`，然后定义了一个监听器 `MyApplicationEventListener`，用来监听 `MyApplicationEvent` 事件，然后在 `main()` 方法中，调用 `context.publishEvent(...)` 方法来发布该事件。

运行，结果如下：

```
main | main | 自定义事件 ...

```

可以看到，事件成功监听到了。

从运行可知，事件的发布线程为 `main`，事件的监听处理线程也是 `main`。

### 2\. 事件组件介绍

正式开讲前，我们先来介绍下事件相关的组件。事件相关的组件有 4 个：

1.  事件：发布的内容
2.  发布器：用来发布事件的组件
3.  广播器：接收发布器发布的事件，并将接收到的事件广播给监听器
4.  监听器：监听事件

下面我们一一来分析。

#### 2.1 事件

spring 提供的事件为 `ApplicationEvent`，这是一个抽象类，继承了 jdk 提供的 `EventObject` 类，发布自定义事件时，可继承 `ApplicationEvent`：

```
public abstract class ApplicationEvent extends EventObject {

    private static final long serialVersionUID = 7099057708183571937L;

    /** 增加 timestamp 属性*/
    private final long timestamp;

    public ApplicationEvent(Object source) {
        super(source);
        this.timestamp = System.currentTimeMillis();
    }

    public final long getTimestamp() {
        return this.timestamp;
    }
}

```

`ApplicationEvent` 是 `EventObject`，我们继续：

```
/**
 * EventObject 由jdk提供，位于 java.util 包。
 */
public class EventObject implements java.io.Serializable {

    private static final long serialVersionUID = 5516075349620653480L;

    // 事件内容
    protected transient Object  source;

    public EventObject(Object source) {
        if (source == null)
            throw new IllegalArgumentException("null source");

        this.source = source;
    }

    /**
     * 获取事件内容
     */
    public Object getSource() {
        return source;
    }

    public String toString() {
        return getClass().getName() + "[source=" + source + "]";
    }
}

```

结合 `ApplicationEvent` 与 `EventObject` 来看，`ApplicationEvent` 提供了两个属性：

*   `source`：来自 `EventObject` 的属性，定义了事件的内容；
*   `timestamp`: 时间戳，用来记录是事件的生成时间。

实际上，spring 除了能发布 `ApplicationEvent` 类型的事件外，还可以发布 `Object` 类型的事件，接下面查看发布器就可以发 现这一点。

#### 2.2 发布器

spring 提供的发布器为 `ApplicationEventPublisher`，代码如下：

```
public interface ApplicationEventPublisher {

    /**
     * 发布ApplicationEvent类型的事件
     */
    default void publishEvent(ApplicationEvent event) {
        publishEvent((Object) event);
    }

    /**
     * 发布Object类型的事件
     */
    void publishEvent(Object event);

}

```

这是个接口，其内定义了两个方法：

*   `void publishEvent(ApplicationEvent event)`: 用来发布 `ApplicationEvent` 类型的事件
*   `void publishEvent(Object event)`: 用来发布 `Object` 类型的事件

`AbstractApplicationContext` 是 `ApplicationEventPublisher` 的子类，因此我们可以直接调用 `AbstractApplicationContext#publishEvent` 来发布事件。

关于 `AbstractApplicationContext` 对这两个方法的实现，我们后面分析发布流程时再具体分析。

#### 2.3 广播器

广播器的作用是接收发布的事件，然后将事件广播给监听器，代码如下：

```
public interface ApplicationEventMulticaster {

    /**
     * 添加监听器
     */
    void addApplicationListener(ApplicationListener<?> listener);

    /**
     * 添加监听器的 beanName
     */
    void addApplicationListenerBean(String listenerBeanName);

    /**
     * 移除监听器
     */
    void removeApplicationListener(ApplicationListener<?> listener);

    /**
     * 移除监听器的 beanName
     */
    void removeApplicationListenerBean(String listenerBeanName);

    /**
     * 移除所有的监听器
     */
    void removeAllListeners();

    /**
     * 广播事件
     */
    void multicastEvent(ApplicationEvent event);

    /**
     * 广播事件
     */
    void multicastEvent(ApplicationEvent event, @Nullable ResolvableType eventType);

}

```

从代码上来看，广播器主要功能有两个：

1.  维护监听器，可以对监听器进行增删操作
2.  广播事件

spring 默认的广播器为 `SimpleApplicationEventMulticaster`，这个我们后面会再分析。

#### 2.4 监听器

监听器用来监听发布的事件，然后做一些处理操作，代码如下：

```
@FunctionalInterface
public interface ApplicationListener<E extends ApplicationEvent> extends EventListener {

    /**
     * 处理事件
     */
    void onApplicationEvent(E event);

}

```

在处理事件监听时，我们需要实现 `ApplicationListener`，然后在 `onApplicationEvent(...)` 方法中编写我们的事件监听逻辑。

### 3\. 回顾：`广播器的初始化`与`监听器的注册`

本节我们来回顾下`事件广播器的初始化`与`监听器的注册`的流程。

在处理 spring 容器启动的 `AbstractApplicationContext#refresh` 方法中，`事件广播器的初始化`与`监听器的注册`分别发生在第 8 步与第 10 步：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-c4d4ac83c23e41a706b7ba545fd8d0f7681.png)

相关代码如下：

```
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {

    /**
     * 初始化广播器
     * 如果已配置了事件广播器，就使用已配置的，否则就使用默认的事件广播器
     */
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

    /**
     * 注册监听器
     */
    protected void registerListeners() {
        // 1\. 先将手动添加的监听器放到广播器中
        // 调用AbstractApplicationContext#addApplicationListener进行添加
        for (ApplicationListener<?> listener : getApplicationListeners()) {
            getApplicationEventMulticaster().addApplicationListener(listener);
        }

        // 2\. 从beanFactory中获取取监听器的名称，添加到广播器中
        String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
        for (String listenerBeanName : listenerBeanNames) {
            getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
        }

        // 3\. 如果存在早期应用事件，发布
        Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
        // 将 earlyApplicationEvents 设置为 null，再发布早期事件
        this.earlyApplicationEvents = null;
        if (earlyEventsToProcess != null) {
            for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
                getApplicationEventMulticaster().multicastEvent(earlyEvent);
            }
        }
    }

    ...

}

```

广播器的初始化逻辑很简单，即如果已配置了事件广播器，就使用已配置的，否则就使用默认的事件广播器，默认的事件广播器是 `SimpleApplicationEventMulticaster`。

注册监听器的流程如下：

1.  先将手动添加的监听器放到广播器中，我们可以调用 `AbstractApplicationContext#addApplicationListener` 添加监听器；
2.  从 `beanFactory` 中获取取监听器的名称，添加到广播器中，注意：此时 `beanFactory` 中的 bean 并没有初始化，因此只能添加 `beanName`;
3.  如果存在早期事件，就发布早期事件。

回顾完这两个流程后，我们大脑中需要清楚地明白以下两点：

1.  广播器是在哪里初始化的
2.  监听器是在哪里注册到广播器中的

### 4\. 事件发布流程

经过前面两节的铺垫，我们对事件的组件已经有了个大概的概念，也明白了事件广播器的初始化与监听器的注册流程，接下来 我们就正式分析事件的发布流程了。

在 demo 中，我们调用 `context.publishEvent(...)` 来发布事件，我们跟进这个方法：

```
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {

    @Override
    public void publishEvent(ApplicationEvent event) {
        publishEvent(event, null);
    }

    protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
        Assert.notNull(event, "Event must not be null");
        // 处理事件类型
        ApplicationEvent applicationEvent;
        if (event instanceof ApplicationEvent) {
            applicationEvent = (ApplicationEvent) event;
        }
        else {
            applicationEvent = new PayloadApplicationEvent<>(this, event);
            if (eventType == null) {
                eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
            }
        }

        // earlyApplicationEvents 不为 null，将applicationEvent添加到 earlyApplicationEvents
        // 在注册监听器后，会把 earlyApplicationEvents 设置为 null
        if (this.earlyApplicationEvents != null) {
            this.earlyApplicationEvents.add(applicationEvent);
        }
        else {
            // 这里是发布事件的操作
            getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
        }

        // 存在父容器，一样发布
        if (this.parent != null) {
            if (this.parent instanceof AbstractApplicationContext) {
                ((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
            }
            else {
                this.parent.publishEvent(event);
            }
        }
    }

}

```

这个方法很简单，关键代码为

```
// 这里是发布事件的操作
getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);

```

这行代码先获取了事件广播器，然后广播事件。

前面提到，spring 提供的默认的事件广播器是 `SimpleApplicationEventMulticaster`，我们进入 `SimpleApplicationEventMulticaster#multicastEvent(ApplicationEvent, ResolvableType)` 来看看事件的广播流程：

```
/**
 * 广播事件
 */
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    // 1\. 获取 TaskExecutor
    Executor executor = getTaskExecutor();
    // 2\. getApplicationListeners(...) 获取能监听该事件的监听器
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        // 3\. 遍历监听器，逐一调用 invokeListener(...) 方法
        if (executor != null) {
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
            invokeListener(listener, event);
        }
    }
}

```

以上方法包含 3 个操作：

1.  获取执行器：`getTaskExecutor()`
2.  获取事件的监听器：`getApplicationListeners(...)`
3.  调用监听器的监听方法：`invokeListener(...)`

以上操作包含了事件广播的整个流程，我们下面来好好分析下。

#### 1\. 获取执行器：`getTaskExecutor()`

`taskExecutor` 是 `SimpleApplicationEventMulticaster` 的一个属性，`getTaskExecutor()` 是 `taskExecutor` 的 `getter` 方法：

```
    private Executor taskExecutor;

    public void setTaskExecutor(@Nullable Executor taskExecutor) {
        this.taskExecutor = taskExecutor;
    }

    @Nullable
    protected Executor getTaskExecutor() {
        return this.taskExecutor;
    }

```

spring 为我们提供了两种类型的 `taskExecutor`：

1. `SyncTaskExecutor`：同步的 `taskExecutor`，其 `execute(...)` 方法为：

   ```
    @Override
    public void execute(Runnable task) {
        Assert.notNull(task, "Runnable must not be null");
        task.run();
    }
   
   ```

   可以看到，这确实是个同步方法，直接调用 `Runnable#run` 方法，并没有启动新的线程

2. `SimpleAsyncTaskExecutor`：异步的 `taskExecutor`，其 `execute(...)` 方法为：

   ```
   @Override
    public void execute(Runnable task, long startTimeout) {
        Assert.notNull(task, "Runnable must not be null");
        Runnable taskToUse = (this.taskDecorator != null 
                ? this.taskDecorator.decorate(task) : task);
        // doExecute(...) 才是真正干活的方法
        if (isThrottleActive() && startTimeout > TIMEOUT_IMMEDIATE) {
            this.concurrencyThrottle.beforeAccess();
            doExecute(new ConcurrencyThrottlingRunnable(taskToUse));
        }
        else {
            doExecute(taskToUse);
        }
    }
   
    /**
     * 真正干活的方法
     * 从代码来看，这个方法会创建新创建来执行任务，但并没有使用线程池
     */
    protected void doExecute(Runnable task) {
        // 可以看到，这里创建了线程，并且启动了线程
        Thread thread = (this.threadFactory != null 
                ? this.threadFactory.newThread(task) : createThread(task));
        thread.start();
    }
   
   ```

   可以看，在 `SimpleAsyncTaskExecutor` 中，会创建新的线程来执行任务。

这里能不能获取到执行器呢？通过调试发现，执行器的获取结果为 `null`：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-2e6ba6389e603a6373287dac26a89a24109.png)

因此在下面执行 `invokeListener(...)` 方法时，是直接调用的：

```
...
        else {
            invokeListener(listener, event);
        }
...

```

#### 2\. 获取事件的监听器：`getApplicationListeners(...)`

准备来说，这里获取的是能监听传入事件的监听器，方法为 `AbstractApplicationEventMulticaster#getApplicationListeners(ApplicationEvent, ResolvableType)`：

```
/**
 * 这个方法就两个步骤：
 * 1\. 从缓存中获取，能获取到，直接返回
 * 2\. 不能从缓存中获取，调用 retrieveApplicationListeners(...) 方法获取
 */
protected Collection<ApplicationListener<?>> getApplicationListeners(
        ApplicationEvent event, ResolvableType eventType) {
    Object source = event.getSource();
    Class<?> sourceType = (source != null ? source.getClass() : null);
    ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
    // 1\. 从缓存中获取
    ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
    if (retriever != null) {
        return retriever.getApplicationListeners();
    }
    if (this.beanClassLoader == null ||
            (ClassUtils.isCacheSafe(event.getClass(), this.beanClassLoader) &&
            (sourceType == null || ClassUtils.isCacheSafe(sourceType, this.beanClassLoader)))) {
        synchronized (this.retrievalMutex) {
            retriever = this.retrieverCache.get(cacheKey);
            if (retriever != null) {
                return retriever.getApplicationListeners();
            }
            retriever = new ListenerRetriever(true);
            // 2\. 获取操作，这里才是关键
            Collection<ApplicationListener<?>> listeners =
                    retrieveApplicationListeners(eventType, sourceType, retriever);
            this.retrieverCache.put(cacheKey, retriever);
            return listeners;
        }
    }
    else {
        return retrieveApplicationListeners(eventType, sourceType, null);
    }
}

```

这个方法看着长，但关键操作就两个：

1.  从缓存中获取，能获取到，直接返回
2.  不能从缓存中获取，调用 `retrieveApplicationListeners(...)` 方法获取

我们继续进入 `retrieveApplicationListeners(...)`：

```
/**
 * 在这里真正获取监听器
 *
 */
private Collection<ApplicationListener<?>> retrieveApplicationListeners(
        ResolvableType eventType, @Nullable Class<?> sourceType, 
        @Nullable ListenerRetriever retriever) {
    List<ApplicationListener<?>> allListeners = new ArrayList<>();
    Set<ApplicationListener<?>> listeners;
    Set<String> listenerBeans;
    synchronized (this.retrievalMutex) {
        listeners = new LinkedHashSet<>(this.defaultRetriever.applicationListeners);
        listenerBeans = new LinkedHashSet<>(this.defaultRetriever.applicationListenerBeans);
    }
    // 从 listeners 中获取能处理当前事件的 lister
    for (ApplicationListener<?> listener : listeners) {
        // 在这里判断当前listener是否支持传入event
        if (supportsEvent(listener, eventType, sourceType)) {
            if (retriever != null) {
                retriever.applicationListeners.add(listener);
            }
            allListeners.add(listener);
        }
    }
    // 从 listenerBeans 中获取能处理当前事件的 lister
    if (!listenerBeans.isEmpty()) {
        ConfigurableBeanFactory beanFactory = getBeanFactory();
        for (String listenerBeanName : listenerBeans) {
            try {
                // 这里判断当前 listenerBeanName 是否能监听传入事件
                if (supportsEvent(beanFactory, listenerBeanName, eventType)) {
                    // 从容器中获取对应的bean，对于早期事件，会引起监听器提前初始化
                    ApplicationListener<?> listener = beanFactory.getBean(
                            listenerBeanName, ApplicationListener.class);
                    if (!allListeners.contains(listener) 
                            && supportsEvent(listener, eventType, sourceType)) {
                        if (retriever != null) {
                            // 注意单例与非单例的区别
                            if (beanFactory.isSingleton(listenerBeanName)) {
                                retriever.applicationListeners.add(listener);
                            }
                            else {
                                retriever.applicationListenerBeans.add(listenerBeanName);
                            }
                        }
                        allListeners.add(listener);
                    }
                }
                else {
                    Object listener = beanFactory.getSingleton(listenerBeanName);
                    if (retriever != null) {
                        retriever.applicationListeners.remove(listener);
                    }
                    allListeners.remove(listener);
                }
            }
            catch (NoSuchBeanDefinitionException ex) {
            }
        }
    }
    // 排序
    AnnotationAwareOrderComparator.sort(allListeners);
    if (retriever != null && retriever.applicationListenerBeans.isEmpty()) {
        retriever.applicationListeners.clear();
        retriever.applicationListeners.addAll(allListeners);
    }
    return allListeners;
}
...

}

```

在 `AbstractApplicationContext#registerListeners` 方法中，我们在注册监听器时 ，注册了两大类监听器：

1.  注册手动添加的监听器，它们是一个个实例；
2.  从容器中获取监听器的 beanName，进行注册；

`retrieveApplicationListeners()` 中出现的 `listeners` 与 `listenerBeans` 就是处理这两种类型的监听器的。

以上方法虽然有点长，但逻辑非常清晰：

1.  遍历 `listeners`，逐一调用 `supportsEvent(listener, eventType, sourceType)` 来判断当前 listener 能否监听传入事件；
2.  遍历 `listenerBeans`，逐一调用 `supportsEvent(beanFactory, listenerBeanName, eventType)` 来判断当前 listener 能否监听传入事件；

关于这两个方法的处理比较复杂，这里就不一一进行分析了，这里给出处理的大致思路：

1. 从传入的 `listener` 或 `listenerBeanName` 获取 `listener` 的 Class，`listener` 只需 `listener.getClass()` 即可，`listenerBeanName` 可通过 `beanFactory.getType(listenerBeanName)` 获取；

2. 获取 `listener` 监听的事件类型，一个监听器是这样定义的：

   ```
   public class MyApplicationEventListener 
           implements ApplicationListener<MyApplicationEvent> {
   
       @Override
       public void onApplicationEvent(MyApplicationEvent event) {
           ...
       }
   }
   
   ```

   我们可以这样获取到 `MyApplicationEvent`：

   ```
   // 这些类都是jdk提供的
   ParameterizedType parameterizedType = (ParameterizedType) 
           MyApplicationEventListener.class.getGenericInterfaces()[0];
       Class<?> type = (Class)parameterizedType.getActualTypeArguments()[0];
   
   ```

   当然，spring 在处理这部分逻辑时特别复杂，毕竟如果 `MyApplicationEventListener` 同时实现了多个接口，或者 `MyApplicationEventListener` 的父 - 父 - 父 -... 接口才是 `ApplicationListener`，这些情况都要考虑到。

3. 获取到监听器能监听的事件了，就能判断当前判断监听是否能监听传入事件了，spring 处理匹配的方法为 `ResolvableType#isAssignableFrom(ResolvableType)`，从注释上来看，它实现了类似 `Class.isAssignableFrom` 方法的功能，同时还对传入类上的泛型也实现了类似 `Class.isAssignableFrom` 的功能。

#### 3\. 调用监听方法：`invokeListener(...)`

终于到调用监听方法了，代码如下：

```
/**
 * 执行监听器
 */
protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
    ErrorHandler errorHandler = getErrorHandler();
    // 最终调用的是 doInvokeListener(...)
    if (errorHandler != null) {
        try {
            doInvokeListener(listener, event);
        }
        catch (Throwable err) {
            errorHandler.handleError(err);
        }
    }
    else {
        doInvokeListener(listener, event);
    }
}

/**
 * 具体的执行操作，最终调用的是 ApplicationListener#onApplicationEvent 方法
 */
private void doInvokeListener(ApplicationListener listener, ApplicationEvent event) {
    try {
        listener.onApplicationEvent(event);
    }
    catch (ClassCastException ex) {
        String msg = ex.getMessage();
        if (msg == null || matchesClassCastMessage(msg, event.getClass())) {
            // 省略日志打印
        }
        else {
            throw ex;
        }
    }
}

```

这一块很简单，就是遍历上一步获取的监听器，逐一调用其 `onApplicationEvent(...)` 方法。

这里需要注意的是，在前面`获取执行器`的分析中，我们提到 `getTaskExecutor()` 的结果为 `null`，这表明 `invokeListener(...)` 是直接执行的，并没有另起一个线程中执行，这点需要格外注意。

### 5\. 事件的应用

#### 5.1 监听容器启动完成事件

在 spring 启动流程中，容器启动完成会发布 `ContextRefreshed` 事件：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-f60ea2710c59e2ce9e1401f1b54cbd9d700.png)

我们要监听器该事件也十分简单，相应的 `Listener` 如下：

```
@Component
public class ContextRefreshedListener 
        implements ApplicationListener<ContextRefreshedEvent> {

    @Override
    public void onApplicationEvent(ContextRefreshedEvent event) {
        System.out.println("容器启动完成");
    }

}

```

#### 5.2 发布早期事件

一开始，我以为早期事件是这样发布：

```
AnnotationConfigApplicationContext context 
        = new AnnotationConfigApplicationContext();
context.register(Demo08Config.class);
// 发布事件，在容器启动前发布
context.publishEvent(new MyApplicationEvent(
        Thread.currentThread().getName() + " | 自定义事件 ..."));
context.refresh();

```

运行后，会报错： ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-4fc18c95f69d8c3d65be2a0f603d42e19ce.png)

从错误信息来看，是说广播器未初始化。

那么广播器是在哪里初始化的呢？回忆下前面的分析，很清楚地知道是在 `refresh()` 过程的第 8 步，而早期事件的发布又在第 10 步，这样一来，早期事件就只能在第 9 步了：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1fecd516c6d58d628937b20ebd806065891.png)

从这里可以看出，早期事件就是为 `onRefresh()` 这个扩展点准备的！

早期事件的发布也就简单了：

```
ApplicationContext context = 
        new AnnotationConfigApplicationContext(Demo08Config.class) {
    @Override
    public void onRefresh() {
        // 这里发布就是早期事件
        publishEvent(new MyApplicationEvent(
                Thread.currentThread().getName() + " | 自定义事件 ..."));
    }
};

```

#### 5.3 异步广播事件

从前面的源码分析来看，事件的执行是在同一个线程中进行的，这点在 demo 的运行结果中也能看出来：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-afed97c136df717ba3d9397ff142ae0675f.png)

前面已经分析过，广播事件时，会先获取 `executor`，如果 `executor` 存在，在 `executor` 执行，否则就直接执行，而广播器中的 `executor` 为 `null`，因此很容易结论：要想实现异步执行，需要在广播器中设置 `executor` 属性。

前面同样也分析过，spring 在初始化广播器时，会先判断容易中是否存在广播器（`beanName` 为 `applicationEventMulticaster`），不存在则使用 `SimpleApplicationEventMulticaster` 创建默认的广播器。

如此一来，我们只需要自定义 `beanName` 为 `applicationEventMulticaster` 的 bean 不就行，像这样：

```
@Configuration
@ComponentScan
public class Demo08Config {

    /**
     * 生成自定义广播器
     * 注意：名称必须为 applicationEventMulticaster
     */
    @Bean
    public ApplicationEventMulticaster applicationEventMulticaster() {
        SimpleApplicationEventMulticaster applicationEventMulticaster
                = new SimpleApplicationEventMulticaster();
        // SimpleAsyncTaskExecutor 是spring中提供的异步任务执行器
        applicationEventMulticaster.setTaskExecutor(new SimpleAsyncTaskExecutor());
        return applicationEventMulticaster;
    }
}

```

`SimpleApplicationEventMulticaster` 使用了 spring 提供的异步任务执行器：`SimpleAsyncTaskExecutor`，运行，结果如下：

```
SimpleAsyncTaskExecutor-2 | main | 自定义事件 ...

```

可以看到，发布线程与监听线程不再是同一个了。

但事情到这还算完，前面已经分析过 `SimpleAsyncTaskExecutor` 的任务执行过程，在执行时它会不断创建新线程：

```
   protected void doExecute(Runnable task) {
        // 可以看到，这里创建了线程，并且启动了线程
        Thread thread = (this.threadFactory != null 
                ? this.threadFactory.newThread(task) : createThread(task));
        thread.start();
    }

```

如果我们连续发布 100 个事件：

```
for(int i = 0; i < 100; i++) {
    context.publishEvent(new MyApplicationEvent(
            Thread.currentThread().getName() + " | 自定义事件 ..."));
}

```

运行结果会这样：

```
SimpleAsyncTaskExecutor-2 | main | 自定义事件 ...
SimpleAsyncTaskExecutor-3 | main | 自定义事件 ...
SimpleAsyncTaskExecutor-4 | main | 自定义事件 ...
...
SimpleAsyncTaskExecutor-99 | main | 自定义事件 ...
SimpleAsyncTaskExecutor-100 | main | 自定义事件 ...
SimpleAsyncTaskExecutor-101 | main | 自定义事件 ...

```

可以看到，对于每一个事件，都会创建一个新线程来执行，线程本就是如此宝贵的资源，这样频繁地创建就是一个极大的浪费，那如何重用线程呢？答案是线程池。

我们看下 `SimpleApplicationEventMulticaster#setTaskExecutor` 方法的参数，发现它是 `java.util.concurrent.Executor`，接下来事情就变得简单了，我们直接使用 jdk 提供的线程池：

```
@Bean
public ApplicationEventMulticaster applicationEventMulticaster() {
    SimpleApplicationEventMulticaster applicationEventMulticaster
            = new SimpleApplicationEventMulticaster();
    // 使用jdk提供的线程池
    applicationEventMulticaster.setTaskExecutor(Executors.newFixedThreadPool(4));
    return applicationEventMulticaster;
}

```

这里使用的是 jdK 提供的 `newFixedThreadPool`，设置了 4 个线程，运行，结果如下：

```
pool-1-thread-2 | main | 自定义事件 ...
pool-1-thread-3 | main | 自定义事件 ...
pool-1-thread-4 | main | 自定义事件 ...
pool-1-thread-4 | main | 自定义事件 ...
pool-1-thread-1 | main | 自定义事件 ...
pool-1-thread-3 | main | 自定义事件 ...
pool-1-thread-3 | main | 自定义事件 ...
pool-1-thread-2 | main | 自定义事件 ...
pool-1-thread-1 | main | 自定义事件 ...
pool-1-thread-3 | main | 自定义事件 ...
...

```

可以看到，自始至终只有 4 个线程在广播事件。

在有些代码规范中，是禁止直接使用 jdk 提供的线程的池，他们更提倡自定义线程池，关于这点，由于本文并不是讨论线程池技术的，因此就简单地使用 jdk 提供的线程示例下。

#### 5.4 自定义事件

实际上，前面提供的示例 demo 就是自定义事件，这里就不重复了。

### 6\. 总结

本文分析了 spring 事件机制，介绍了事件的四大基本组件：事件，发布器，广播器，监听器，并且从源码分析了组件的初始化、事件的发布流程。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4713339](https://my.oschina.net/funcy/blog/4713339) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_