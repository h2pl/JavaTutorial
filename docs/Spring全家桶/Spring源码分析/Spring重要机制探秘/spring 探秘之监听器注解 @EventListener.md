最近在梳理 `BeanFactoryPostProcessor` 功能时，发现了处理事件监听的另一种处理方式：使用 `@EventListener` 注解，处理类的是 `EventListenerMethodProcessor`，我们先从一个示例出发，再逐步分析 `@EventListener` 的处理过程。

### 1. `@EventListener` 使用示例

先定义一个事件：

```
public class MyApplicationEvent extends ApplicationEvent {

    private static final long serialVersionUID = -1L;

    public MyApplicationEvent(Object source) {
        super(source);
    }
}

```

再准备一个事件监听器，这次使用 `@EventListener` 指定监听器：

```
@Configuration
public class Demo08Config {

    /**
     * 这是个事件监听器
     */
    @EventListener(MyApplicationEvent.class)
    public void listener(MyApplicationEvent event) {
        System.out.println("@EventListener监听到了事件："
                + Thread.currentThread().getName() + " | " + event.getSource());
    }
}

```

然后发布事件：

```
@ComponentScan
public class Demo08Main {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext(Demo08Config.class);
        // 发布事件
        context.publishEvent(new MyApplicationEvent(
            Thread.currentThread().getName() + " | 自定义事件 ..."));
    }
}

```

运行，结果如下：

```
@EventListener监听到了事件：main | main | 自定义事件 ...

```

可以看到，被 `@EventListener` 标记的方法确实完成了事件的监听。

### 2. `@EventListener` 概览

使用 `@EventListener` 让我们免除了实现 `ApplicationListener` 就能实现事件监听，简化了代码的开发。本节我们来看看 `@EventListener` 能为我们做些什么。

`@EventListener` 的代码如下：

```
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface EventListener {

    /**
     * classes 的别名
     * 用来指定监听的事件，可以同时监听多个事件 
     */
    @AliasFor("classes")
    Class<?>[] value() default {};

    /**
     * value 的别名
     * 用来指定监听的事件，可以同时监听多个事件 
     */
    @AliasFor("value")
    Class<?>[] classes() default {};

    /**
     * 可以指定一个条件，条件成立时方法才会执行
     * 支持spring el 表达式
     */
    String condition() default "";

}

```

从代码来看，`@EventListener` 提供了两大功能：

*   指定监听的事件，可以指定多个事件
*   指定一个条件，条件成立时方法才会执行，条件支持 spring EL 表达式

了解 `@EventListener` 提供的功能后，接下来我们来看看 spring 是如何处理这个注解的。

### 3. `@EventListener` 的处理：`EventListenerMethodProcessor`

文章开篇便说过，关于 `@EventListener` 的功能，我是在梳理 `BeanFactoryPostProcessor` 功能时发现的，当时发现 `BeanFactoryPostProcessor` 的实现类 `EventListenerMethodProcessor` 会处理 `@EventListener` 注解，接下来就从代码角度来分析 `EventListenerMethodProcessor` 处理 `@EventListener` 的流程。

首先我们来认识下 `EventListenerMethodProcessor`：

```
public class EventListenerMethodProcessor
        implements SmartInitializingSingleton, ApplicationContextAware, BeanFactoryPostProcessor {
    ...
}

```

它主要是实现了两个接口：

*   `BeanFactoryPostProcessor`：大名鼎鼎的 `BeanFactoryPostProcessor` 啊，可以定制化 `BeanFactory` 的一些行为；
*   `SmartInitializingSingleton`：处理单例 bean 的初始化操作，执行时机是在 `bean` 初始化完成之后。

我们先来看看它对 `BeanFactoryPostProcessor#postProcessBeanFactory` 的实现：

```
    @Nullable
    private List<EventListenerFactory> eventListenerFactories;

    /**
     * 这是 BeanFactoryPostProcessor 的 postProcessBeanFactory(...) 方法.
     * 在这个方法里，仅是获取了 EventListenerFactory，然后保存在 eventListenerFactories中.
     * spring 默认提供的 EventListenerFactory 有两个：
     *     1\. DefaultEventListenerFactory：spring 默认的
     *     2\. TransactionalEventListenerFactory：处理事务监听的
     * 这部分并没有做什么。
     */
    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        this.beanFactory = beanFactory;

        Map<String, EventListenerFactory> beans = beanFactory
                .getBeansOfType(EventListenerFactory.class, false, false);
        List<EventListenerFactory> factories = new ArrayList<>(beans.values());
        // 排序
        AnnotationAwareOrderComparator.sort(factories);
        this.eventListenerFactories = factories;
    }

```

这个方法还是比较简单的，就只是从容器中获取了 `EventListenerFactory` 并赋值给了 `eventListenerFactories`。

`EventListenerFactory` 的功能是用来生成 `ApplicationListener`，后面我们会分析 spring 是如何把 `@EventListener` 标记的方法转换成 `ApplicationListener` 对象的。从代码来看，spring 提供的 `EventListenerFactory` 有两个：

*   `DefaultEventListenerFactory`：spring 默认的
*   `TransactionalEventListenerFactory`：处理事务监听的

接下来我们再来看看它对 `SmartInitializingSingleton#afterSingletonsInstantiated()` 的实现：

```
    /**
     * 这个方法是 SmartInitializingSingleton 的 afterSingletonsInstantiated() 方法.
     * 会在bean初始化完成后调用。
     * 在这个方法中，主要是将标记了 @EventListener 的方法转换成 ApplicationListener 对象，并注册到监听器中
     */
    @Override
    public void afterSingletonsInstantiated() {
        ConfigurableListableBeanFactory beanFactory = this.beanFactory;
        Assert.state(this.beanFactory != null, "No ConfigurableListableBeanFactory set");
        String[] beanNames = beanFactory.getBeanNamesForType(Object.class);
        for (String beanName : beanNames) {
            if (!ScopedProxyUtils.isScopedTarget(beanName)) {
                Class<?> type = null;
                try {
                    // 获取 aop 代理对应的目标类，从 beanName 对应的 BeanDefinition 中获取.
                    // 如果获取不到，表示是不是代理对象，使用 beanFactory.getType(beanName) 获取
                    type = AutoProxyUtils.determineTargetClass(beanFactory, beanName);
                }
                catch (Throwable ex) {
                    ...
                }
                if (type != null) {
                    if (ScopedObject.class.isAssignableFrom(type)) {
                        try {
                            Class<?> targetClass = AutoProxyUtils.determineTargetClass(
                                    beanFactory, ScopedProxyUtils.getTargetBeanName(beanName));
                            if (targetClass != null) {
                                type = targetClass;
                            }
                        }
                        catch (Throwable ex) {
                            ...
                        }
                    }
                    try {
                        // 具体的处理操作
                        processBean(beanName, type);
                    }
                    catch (Throwable ex) {
                        ...
                    }
                }
            }
        }
    }

```

这个方法的执行流程如下：

1.  获取当前 `beanFactory` 中的所有 `bean`，调用的是 `beanFactory.getBeanNamesForType(Object.class)` 方法，注意传入的 `class` 是 `Object`，这样就拿出了所有的 `bean`；
2.  遍历这些 `bean`，对每一个 `bean`，如果是代理对象，则调用 `AutoProxyUtils.determineTargetClass(beanFactory, beanName)` 方法获取其目标类，我们知道注解是不能继承的，要获取 `@EventListener` 标记的方法，需要从目标类去获取；如果不是代理对象，则目标类就是 `bean` 对应的类；
3.  调用 `processBean(beanName, type)` 方法进一步处理.

看来关键是在 `EventListenerMethodProcessor#processBean` 方法了：

```
/**
 * 处理 bean， 这个方法会将 @EventListener 注解标记的方法转换为 ApplicationListener 对象，并注册到监听器.
 * @param beanName
 * @param targetType
 */
private void processBean(final String beanName, final Class<?> targetType) {
    if (!this.nonAnnotatedClasses.contains(targetType) &&
            AnnotationUtils.isCandidateClass(targetType, EventListener.class) &&
            !isSpringContainerClass(targetType)) {
        Map<Method, EventListener> annotatedMethods = null;

        try {
            // 1\. 找到标记 @EventListener 的方法
            annotatedMethods = MethodIntrospector.selectMethods(targetType,
                (MethodIntrospector.MetadataLookup<EventListener>) method ->
                    AnnotatedElementUtils.findMergedAnnotation(method, EventListener.class));
        }
        catch (Throwable ex) {
            ...
        }

        if (CollectionUtils.isEmpty(annotatedMethods)) {
            this.nonAnnotatedClasses.add(targetType);
        }
        else {
            ConfigurableApplicationContext context = this.applicationContext;
            Assert.state(context != null, "No ApplicationContext set");
            List<EventListenerFactory> factories = this.eventListenerFactories;
            Assert.state(factories != null, "EventListenerFactory List not initialized");
            // 2\. 使用 EventListenerFactory 来产生 ApplicationListener 对象
            for (Method method : annotatedMethods.keySet()) {
                for (EventListenerFactory factory : factories) {
                    // 判断当前 EventListenerFactory 是否支持当前方法
                    if (factory.supportsMethod(method)) {
                        // 如果是代理对象，则得到代理类的方法
                        Method methodToUse = AopUtils.selectInvocableMethod(
                                method, context.getType(beanName));
                        // 生成 ApplicationListener，传入的是代理类的方法
                        ApplicationListener<?> applicationListener = factory
                                .createApplicationListener(beanName, targetType, methodToUse);
                        // 初始化操作，这一步主要是对条件判断器赋值：this.evaluator
                        if (applicationListener instanceof ApplicationListenerMethodAdapter) {
                            ((ApplicationListenerMethodAdapter) applicationListener)
                                    .init(context, this.evaluator);
                        }
                        // 添加到监听器中，保存 Listener 是Set，会自动去重
                        context.addApplicationListener(applicationListener);
                        // 满足其一，后面的 EventListenerFactory 就不会再执行了，
                        // 这种时候就会发现 factories 的顺序很重要
                        // 可以在 EventListenerFactory 中指定顺序
                        break;
                    }
                }
            }
        }
    }
}

```

在 `processBean(...)` 方法的关键处理都作了注释，我们再总结下处理流程：

1.  找到标记 `@EventListener` 的方法，这里使用的是 `MethodIntrospector#selectMethods(...)` 方法进行查找，这个方法会查到当前类的方法、其父类的方法以及接口的默认方法，一直到 Object 为止，这些方法中如果标记了 `@EventListener`，都会被找到，最终会把找到的所有方法放到一个 Map 中；
2.  遍历上述得到的 map，将每个方法转换为 `ApplicationListener` 对象，再添加到 `applicationContext` 的 `ApplicationListener` 中，具体流程如下：
    1.  遍历前面得到的 `EventListenerFactory`；
    2.  遍历当前的 `EventListenerFactory` 是否支持该方法，支持则进行下一步，不支持则不处理；
    3.  对于代理对象，找到当前方法的代理方法，最终执行的也是代理方法；
    4.  创建 `ApplicationListener` 对象，构造方法的参数会传入 `method`，对于代理对象，这个 `method` 是代理对象的 `method`；
    5.  对于 `ApplicationListenerMethodAdapter` 实例，进行初始化操作，主要是赋值：this.evaluator；
    6.  将得到的 `ApplicationListener` 添加到监听器中，保存 `Listener` 是一个 `Set`，会自动去重。

综上所述，被 `@EventListener` 标记的方法之所以能对事件进行监听，是因为 spring 将该方法包装成一个 `ApplicationListener` 添加到监听器了，之后这个监听器就跟实现了 `ApplicationListener` 接口的监听器一样能对事件进行监听了。

### 5. `ApplicationListener` 对象的生成

前面分析了 `@EventListener` 的处理流程，本节将来分析 `ApplicationListener` 对象的生成，对应的代码为：

```
// EventListenerMethodProcessor#processBean
ApplicationListener<?> applicationListener = factory
        .createApplicationListener(beanName, targetType, methodToUse);

```

前面我们介绍过 spring 提供的 `EventListenerFactory` 有两个：

*   `DefaultEventListenerFactory`：spring 默认的
*   `TransactionalEventListenerFactory`：处理事务监听的

接下来我们就来分析这两个 `EventListenerFactory`。

#### 5.1 `DefaultEventListenerFactory`

```
public class DefaultEventListenerFactory implements EventListenerFactory, Ordered {
    ...

    /**
     * 不愧是默认的实现，能支持所有的方法
     */
    @Override
    public boolean supportsMethod(Method method) {
        return true;
    }

    /**
     * 创建操作
     */ 
    @Override
    public ApplicationListener<?> createApplicationListener(String beanName, 
            Class<?> type, Method method) {
        return new ApplicationListenerMethodAdapter(beanName, type, method);
    }
}

```

对以上代码说明如下：

1.  `DefaultEventListenerFactory` 是 spring 提供的默认实现，能支持所有标记了 `@EventListener` 的方法；
2.  创建 `ApplicationListener` 的方法是 `createApplicationListener`，它调用的是 `ApplicationListenerMethodAdapter` 的构造方法。

我们继续看 `ApplicationListenerMethodAdapter`：

```
public class ApplicationListenerMethodAdapter implements GenericApplicationListener {

    ...

    public ApplicationListenerMethodAdapter(String beanName, Class<?> targetClass, Method method) {
        this.beanName = beanName;
        // 处理方法
        this.method = BridgeMethodResolver.findBridgedMethod(method);
        this.targetMethod = (!Proxy.isProxyClass(targetClass) ?
                AopUtils.getMostSpecificMethod(method, targetClass) : this.method);
        this.methodKey = new AnnotatedElementKey(this.targetMethod, targetClass);

        EventListener ann = AnnotatedElementUtils
                .findMergedAnnotation(this.targetMethod, EventListener.class);
        // 解析支持的事件
        this.declaredEventTypes = resolveDeclaredEventTypes(method, ann);
        // 注解指定的条件
        this.condition = (ann != null ? ann.condition() : null);
        this.order = resolveOrder(this.targetMethod);
    }

    ...
}

```

`ApplicationListenerMethodAdapter` 的构造方法就是一堆的赋值操作，这里我们重点来关注下事件的处理：

```
public class ApplicationListenerMethodAdapter implements GenericApplicationListener {

    ...

    /**
     * 解析方法能监听的事件类型
     */
    private static List<ResolvableType> resolveDeclaredEventTypes(Method method, 
            @Nullable EventListener ann) {
        // 方法最多只能有一个参数，监听的事件类型由 @EventListener 指定
        int count = method.getParameterCount();
        if (count > 1) {
            throw new IllegalStateException(
                    "Maximum one parameter is allowed for event listener method: " + method);
        }

        if (ann != null) {
            // 事件可以指定多个，获取的是 classes 属性值
            Class<?>[] classes = ann.classes();
            if (classes.length > 0) {
                List<ResolvableType> types = new ArrayList<>(classes.length);
                for (Class<?> eventType : classes) {
                    types.add(ResolvableType.forClass(eventType));
                }
                return types;
            }
        }

        if (count == 0) {
            throw new IllegalStateException(
                    "Event parameter is mandatory for event listener method: " + method);
        }
        return Collections.singletonList(ResolvableType.forMethodParameter(method, 0));
    }
}

```

从代码可以得出如下结论：

1.  被 `@EventListener` 标记的方法，参数最多只能有一个
2.  方法能监听的事件可以有多个，由 `@EventListener` 指定

#### 5.2 `TransactionalEventListenerFactory`

接下来我们再来看看 `TransactionalEventListenerFactory`：

```
public class TransactionalEventListenerFactory implements EventListenerFactory, Ordered {

    ...

    /**
     * 只支持标记了 @TransactionalEventListener 注解的方法
     */
    @Override
    public boolean supportsMethod(Method method) {
        return AnnotatedElementUtils.hasAnnotation(method, TransactionalEventListener.class);
    }

    /**
     * 创建的对象是 ApplicationListenerMethodTransactionalAdapter
     */
    @Override
    public ApplicationListener<?> createApplicationListener(String beanName, 
            Class<?> type, Method method) {
        return new ApplicationListenerMethodTransactionalAdapter(beanName, type, method);
    }
}

```

从代码来看，`TransactionalEventListenerFactory` 与 `DefaultEventListenerFactory` 差别如下：

*   `ApplicationListenerMethodTransactionalAdapter` 只支持标记了 `@TransactionalEventListener` 的方法；
*   创建的 `ApplicationListener` 实际类型为 `ApplicationListenerMethodTransactionalAdapter`。

我们先来看看 `@TransactionalEventListener` 是个啥：

```
@Target({ElementType.METHOD, ElementType.ANNOTATION_TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
/**
 * 标记了 @EventListener 注解
 */
@EventListener
public @interface TransactionalEventListener {
    /**
     * 事务处理
     */
    TransactionPhase phase() default TransactionPhase.AFTER_COMMIT;

    /**
     * 事务处理
     */
    boolean fallbackExecution() default false;

    /**
     * 指定监听事件
     */
    @AliasFor(annotation = EventListener.class, attribute = "classes")
    Class<?>[] value() default {};

    /**
     * 指定监听事件
     */
    @AliasFor(annotation = EventListener.class, attribute = "classes")
    Class<?>[] classes() default {};

    /**
     * 指定条件
     */
    String condition() default "";
}

```

可以看到，这个注解标记了 `@EventListener`，因此具有与 `@EventListener` 相同的功能。另外，与 `@EventListener` 相比，多了两个属性：`phase()` 与 `fallbackExecution()`，看来是用来控制事务的。

了解了 `TransactionalEventListener` 之后，我们再来看看 `ApplicationListenerMethodTransactionalAdapter`：

```
/**
 * 继承了 ApplicationListenerMethodAdapter
 */
class ApplicationListenerMethodTransactionalAdapter extends ApplicationListenerMethodAdapter {
    ...

    private final TransactionalEventListener annotation;

    public ApplicationListenerMethodTransactionalAdapter(String beanName, 
            Class<?> targetClass, Method method) {
        // 调用父类的方法
        super(beanName, targetClass, method);
        // 这里的注解是 @TransactionalEventListener
        TransactionalEventListener ann = AnnotatedElementUtils
                .findMergedAnnotation(method, TransactionalEventListener.class);
        if (ann == null) {
            throw new IllegalStateException(...);
        }
        this.annotation = ann;
    }

    ...
}

```

从代码上来看，`ApplicationListenerMethodTransactionalAdapter` 继承了 `ApplicationListenerMethodAdapter`，其构造方法也是先调用 `ApplicationListenerMethodAdapter` 的构造方法，然后再给 `annotation` 赋值。

### 6\. 事件监听

接下来我们来看看事件的监听操作。

#### 6.1 `ApplicationListenerMethodAdapter` 监听事件

在前面的文章中，我们知道 `ApplicationListener` 要处理一个事件，必须包含两个操作：

1.  判断当前 `ApplicationListener` 是否支持当前事件
2.  如果支持，则进行事件处理

我们来看看 `ApplicationListenerMethodAdapter` 的这两个操作：

```
public class ApplicationListenerMethodAdapter implements GenericApplicationListener {

    ...

    /**
     * 当前 listener 是否支持传入的事件.
     * 使用的方法是 ResolvableType#isAssignableFrom(ResolvableType)，可以将该方法简单理解
     * 为是对 Class#isAssignableFrom 功能的扩展
     */
    @Override
    public boolean supportsEventType(ResolvableType eventType) {
        for (ResolvableType declaredEventType : this.declaredEventTypes) {
            if (declaredEventType.isAssignableFrom(eventType)) {
                return true;
            }
            if (PayloadApplicationEvent.class.isAssignableFrom(eventType.toClass())) {
                ResolvableType payloadType = eventType
                        .as(PayloadApplicationEvent.class).getGeneric();
                if (declaredEventType.isAssignableFrom(payloadType)) {
                    return true;
                }
            }
        }
        return eventType.hasUnresolvableGenerics();
    }

    /**
     * 事件处理
     */
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        processEvent(event);
    }

    public void processEvent(ApplicationEvent event) {
        // 解析参数
        Object[] args = resolveArguments(event);
        // shouldHandle：判断条件，由 EventListener.condition() 提供
        if (shouldHandle(event, args)) {
            // 反射调用方法
            Object result = doInvoke(args);
            if (result != null) {
                // 处理执行结果，将返回的结果当作事件发布出去
                handleResult(result);
            }
            else {
                logger.trace("No result object given - no result to handle");
            }
        }
    }

}

```

可以看到，判断当前当前 `ApplicationListener` 是否支持当前事件时，使用的是 `ResolvableType#isAssignableFrom(ResolvableType)`，关于这个方法本文并不打算深入分析，可以简单地将该方法理解为是对 `Class#isAssignableFrom` 功能的扩展。

`ApplicationListenerMethodAdapter` 处理事件监听的流程如下：

1.  解析参数，将传入的 `event` 转换为方法参数中具体的值
2.  通过反射调用被 `@EventListener` 标记的方法
3.  处理 `@EventListener` 方法的返回结果，在 `handleResult(...)` 方法里可以将返回的结果当作事件再发布出去

关于 `ApplicationListenerMethodAdapter` 处理事件监听就分析到这里了。

#### 6.2 `ApplicationListenerMethodTransactionalAdapter` 监听事件

`ApplicationListenerMethodTransactionalAdapter` 的事件监听比 `ApplicationListenerMethodAdapter` 要简单一些：

```
/**
 * 继承了 ApplicationListenerMethodAdapter
 */
class ApplicationListenerMethodTransactionalAdapter extends ApplicationListenerMethodAdapter {

    ...

    // supportsEventType(...) 也是继承自你类的方法，自身并无实现

    /**
     * 事件处理
     */
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        // 大概是在处理事务的操作，后面有机会再研究
        if (TransactionSynchronizationManager.isSynchronizationActive()
                && TransactionSynchronizationManager.isActualTransactionActive()) {
            TransactionSynchronization transactionSynchronization 
                    = createTransactionSynchronization(event);
            TransactionSynchronizationManager.registerSynchronization(transactionSynchronization);
        }
        else if (this.annotation.fallbackExecution()) {
            // 调用的是父类的方法
            processEvent(event);
        }
        else {
            // 只是log 打印，省略
            ...
        }
    }

    ...
}

```

对以上代码说明如下：

1.  `ApplicationListenerMethodTransactionalAdapter` 是 `ApplicationListenerMethodAdapter` 的子类，会继承来自 `ApplicationListenerMethodAdapter` 的方法
2.  `ApplicationListenerMethodTransactionalAdapter` 并没有重写 `supportsEventType(...)` 方法，因此也是使用 `ApplicationListenerMethodAdapter` 的 `supportsEventType(...)` 方法来判断事件的支持情况
3.  在处理事件时，`ApplicationListenerMethodTransactionalAdapter` 会进行事务相关的处理，具体的处理逻辑本文就不深入了

### 7\. 总结

本文主要分析了 `@EventListener` 的处理流程，总结如下：

1.  `@EventListener` 可以指定监听的事件、事件处理的条件
2.  处理 `@EventListener` 的类是 `EventListenerMethodProcessor`，该类会把被 `@EventListener` 标记的方法转换成一个 `ApplicationListener` 对象，然后添加到 `ApplicationContext` 的监听器中
3.  方法转换成 `ApplicationListener` 对象的操作由 `EventListenerFactory` 提供，spring 提供了两个 `EventListenerFactory`：
    *   `DefaultEventListenerFactory`：spring 默认的，转换成的 `ApplicationListener` 为 `ApplicationListenerMethodAdapter`
    *   `TransactionalEventListenerFactory`：处理事务监听的，转换成的 `ApplicationListener` 为 `ApplicationListenerMethodTransactionalAdapter`

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4926344](https://my.oschina.net/funcy/blog/4926344) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_