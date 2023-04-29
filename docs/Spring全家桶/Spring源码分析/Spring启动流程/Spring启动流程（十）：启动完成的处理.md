

接上文，继续分析 spring 的启动流程。

12. 完成启动操作: finishRefresh()

AbstractApplicationContext#finishRefresh 方法如下：

    protected void finishRefresh() {
        // 看名字就知道了，清理初始化过程中一系列操作使用到的资源缓存
        clearResourceCaches();
        // 初始化LifecycleProcessor
        initLifecycleProcessor();
        // 这个方法的内部实现是启动所有实现了Lifecycle接口的bean
        getLifecycleProcessor().onRefresh();
        // 发布ContextRefreshedEvent事件
        publishEvent(new ContextRefreshedEvent(this));
        // 检查spring.liveBeansView.mbeanDomain是否存在，有就会创建一个MBeanServer
        LiveBeansView.registerApplicationContext(this);
    }


这个方法代码不多，就几个方法，我们分别来看看。

1. 清理资源缓存：clearResourceCaches()

clearResourceCaches() 方法内容如下：

    public class DefaultResourceLoader implements ResourceLoader {
    
        private final Map<Class<?>, Map<Resource, ?>> resourceCaches
            = new ConcurrentHashMap<>(4);
    
        public void clearResourceCaches() {
            this.resourceCaches.clear();
        }
    
        // 省略了这个类的好多代码
        ...
    
    }


这个方法就是用来清理 resourceCaches 的，这是个 Map，里面存入的内容是 Resource。

那什么是 Resource 呢？在前面介绍扫描包的过程中，我们会先把 class 文件读取出来，转换成 Resource 后再进一步处理，常见的 Resource 类型有 FileSystemResource、UrlResource 等，resourceCaches 就是存放这些 Resource 的。

2. 处理 LifecycleProcessor

我们先来看看什么是 LifecycleProcessor：

    /**
     * 处理容器的启动与关闭操作
     */
    public interface LifecycleProcessor extends Lifecycle {
    
        /**
         * 容器启动完成时调用
         */
        void onRefresh();
    
        /**
         * 容器关闭时调用
         */
        void onClose();
    
    }


这个接口用来处理容器处理容器的启动与关闭操作，比如我们自己实现该接口，然后重写 onRefresh() 与 onClose()，以便在容器启动与关闭时做一些操作，像这样：

    @Component
    public class MyLifecycleProcessor implements LifecycleProcessor {
    
        @Override
        public void onRefresh() {
            System.out.println("容器启动");
        }
    
        @Override
        public void onClose() {
            System.out.println("容器关闭");
        }
    }


与 LifecycleProcessor 相关的方法有两个：initLifecycleProcessor()、getLifecycleProcessor()，我们一起一为看看这两个方法：

AbstractApplicationContext

    private LifecycleProcessor lifecycleProcessor;
    
    /**
     * 初始化 LifecycleProcessor
     */
    protected void initLifecycleProcessor() {
        ConfigurableListableBeanFactory beanFactory = getBeanFactory();
        // 存在，直接使用
        if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
            this.lifecycleProcessor =
                    beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
        }
        // 不存在则创建，默认使用DefaultLifecycleProcessor
        else {
            DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
            defaultProcessor.setBeanFactory(beanFactory);
            this.lifecycleProcessor = defaultProcessor;
            beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
        }
    }
    
    /**
     * 返回 lifecycleProcessor
     */
    LifecycleProcessor getLifecycleProcessor() throws IllegalStateException {
        if (this.lifecycleProcessor == null) {
            throw new IllegalStateException(...);
        }
        return this.lifecycleProcessor;
    }


initLifecycleProcessor 所做的就是设置 AbstractApplicationContext#lifecycleProcessor 属性，如果 beanFactory 中存在 initLifecycleProcessor 则直接使用，否则就创建一个。

getLifecycleProcessor() 仅仅只是返回了 AbstractApplicationContext#lifecycleProcessor 属性。

在 getLifecycleProcessor().onRefresh() 中，还调用了 onRefresh() 方法，我们一起来看看 DefaultLifecycleProcessor#onRefresh 做了什么：

    @Override
    public void onRefresh() {
        startBeans(true);
        this.running = true;
    }


从变量来看，这个方法仅仅只是改了一个运行状态。

3. 发布 ContextRefreshedEvent 事件

代码 publishEvent(new ContextRefreshedEvent(this)) 发布了 ContextRefreshedEvent，我们自己也可以来监听该事件。关于事件，本文并不打算深入，关于 spring 事件的详细分析，可以参考 spring 探秘之 spring 事件机制。

13. 清除缓存: resetCommonCaches()

该方法代码如下：

    protected void resetCommonCaches() {
        ReflectionUtils.clearCache();
        AnnotationUtils.clearCache();
        ResolvableType.clearCache();
        CachedIntrospectionResults.clearClassLoader(getClassLoader());
    }


从方法来看，就是执行各种缓存，执行比较简单，就不深究了。

---

本文原文链接：https://my.oschina.net/funcy/blog/4892555 ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。
