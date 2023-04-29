在前面的文章中，我们成功的编译了 spring 源码，也构建了第一个 spring 测试 demo，接下来我们就基于[第一个 spring 源码调试 demo](https://my.oschina.net/funcy/blog/4533250 "第一个spring源码调试demo") 中的代码，来对 spring 源码进行源码分析。

### 1\. spring 启动流程概览

在前面 demo 的 `main()` 方法中，有这么一行：

```
ApplicationContext context =
        new AnnotationConfigApplicationContext("org.springframework.learn.demo01");

```

这短短的一行就是 spring 的整个启动流程了。上面的代码中，声明了一个 `ApplicationContext` 类型的对象 `context`，右边使用其子类 `AnnotationConfigApplicationContext` 实例化，并在构造方法中传入了包名 `org.springframework.learn.demo01`，这个包名就表明了接下来要扫描哪些包。

> 这里我们接触到了 spring 的第一个组件：`ApplicationContext`，关于 `ApplicationContext` 的分析，可以参考我的文章 [spring 组件（一）：ApplicationContext](https://my.oschina.net/funcy/blog/4597456 "spring组件（一）：ApplicationContext")。

进入到 `AnnotationConfigApplicationContext`，代码如下：

> AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)

```
public AnnotationConfigApplicationContext(String... basePackages) {
     // 1\. 调用无参构造函数，会先调用父类GenericApplicationContext的构造函数
     // 2\. 父类的构造函数里面就是初始化DefaultListableBeanFactory，并且赋值给beanFactory
     // 3\. 本类的构造函数里面，初始化了一个读取器：AnnotatedBeanDefinitionReader read，
     //    一个扫描器ClassPathBeanDefinitionScanner scanner
     // 4\. 这个scanner，就是下面 scan(basePackages) 调用的对象
     this();

     //对传入的包进行扫描，扫描完成后，会得到一个 BeanDefinition 的集合
     scan(basePackages);

     //启动spring，在这里完成spring容器的初始化操作，
     //包括bean的实例化、属性注入，将bean保存到spring容器中等
     refresh();
}

```

这个类就三行，相关操作都已在代码中注释了，这里稍微再总结下，这段代码主要做了三件事：

1.  调用无参构造，进行属性初始化
2.  进行包扫描，得到 BeanDefinition
3.  启用 spring 容器。

接着，我们再来看看 spring 启动流程中，做了哪些事：

> AbstractApplicationContext#refresh

```
public void refresh() throws BeansException, IllegalStateException {
    // 使用synchronized是为了避免refresh() 还没结束，再次发起启动或者销毁容器引起的冲突
    synchronized (this.startupShutdownMonitor) {
        // 做一些准备工作，记录容器的启动时间、标记“已启动”状态、检查环境变量等
        prepareRefresh();

        // 初始化BeanFactory容器、注册BeanDefinition, 最终获得了DefaultListableBeanFactory
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

        // 还是一些准备工作:
        // 1\. 设置了一个类加载器
        // 2\. 设置了bean表达式解析器
        // 3\. 添加了属性编辑器的支持
        // 4\. 添加了一个后置处理器：ApplicationContextAwareProcessor
        // 5\. 设置了一些忽略自动装配的接口
        // 6\. 设置了一些允许自动装配的接口，并且进行了赋值操作
        // 7\. 在容器中还没有XX的bean的时候，帮我们注册beanName为XX的singleton bean
        prepareBeanFactory(beanFactory);

        try {
            // Spring的一个扩展点. 如果有Bean实现了BeanFactoryPostProcessor接口，
            // 那么在容器初始化以后，Spring 会负责调用里面的 postProcessBeanFactory 方法。
            // 具体的子类可以在这步的时候添加特殊的 BeanFactoryPostProcessor 的实现类，来做些事
            postProcessBeanFactory(beanFactory);

            // 调用BeanFactoryPostProcessor各个实现类的postProcessBeanFactory(factory) 方法
            invokeBeanFactoryPostProcessors(beanFactory);

            // 扩展点,注册 BeanPostProcessor 的实现类，注意不是BeanFactoryPostProcessor
            registerBeanPostProcessors(beanFactory);

            // 初始化当前 ApplicationContext 的 MessageSource，用在国际化操作中
            initMessageSource();

            // 这个方法主要为初始化当前 ApplicationContext 的事件广播器
            initApplicationEventMulticaster();

            // 这也是spring的一个扩展点
            onRefresh();

            // Check for listener beans and register them.
            // 注册事件监听器
            registerListeners();

            // 初始化所有的 singleton beans
            finishBeanFactoryInitialization(beanFactory);

            // 完成启动，
            finishRefresh();
        }

        catch (BeansException ex) {
            if (logger.isWarnEnabled()) {
                logger.warn("Exception encountered during context initialization - " +
                    "cancelling refresh attempt: " + ex);
            }

            // Destroy already created singletons to avoid dangling resources.
            // 销毁已经初始化的的Bean
            destroyBeans();

            // Reset 'active' flag.
            // 重置 'active' 状态
            cancelRefresh(ex);

            // Propagate exception to caller.
            throw ex;
        }

        finally {
            // Reset common introspection caches in Spring's core, since we
            // might not ever need metadata for singleton beans anymore...
            // 清除缓存
            resetCommonCaches();
        }
    }
}

```

这个方法虽然代码不多，但包含了 spring bean 的整个创建过程，每个方法做了些什么，在代码中都有注释，这里就不赘述了。

实际上，`refresh()` 涵盖了 spring 整个创建 bean 的流程，在后面的文章中，我们也将重点展开这里面的方法来分析，在现阶段只需要大致了解这些方法做了什么事即可。

整个流程总结如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-9307fefa65470e5c36ae6044631b5416aef.png)

### 2\. spring 启动中 `beanFactory` 的变化

本文中的源码解读就到这里了，接下来我们来看看，spring 启动中 `beanFactory` 有些什么变化。

> `beanFactory` 是 spring 的重要组件之一，直译为 spring bean 工厂，是 spring 生产 bean 与保存 bean 的地方，关于 `beanFactory` 的详细分析，可以查看 [spring BeanFactory 分析](https://my.oschina.net/funcy/blog/4597529 "spring BeanFactory分析")。

我们将断点打在 `AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)` 的 `this()` 方法上，然后运行 demo01 的 `main()` 方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-c3c672a675d9b06f03ea29cb31f6ed5d012.png)

此时的变量中，并没有 `beanFactory`，我们自己添加 `beanFactory` 到调度窗口的变量列表中：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-e9d8ae8fdd3b02b2279376303e3eae4cf2f.png)

这样就能看到对应的值了：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-44d1ade26f0cb667425b0dd99d82666877f.png)

可以看到，此时的 `beanFactory` 为 null，表明 `beanFactory` 并未实例化，我们继续运行：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-664780be9dfef73c12a3f163b349e7e54d8.png)

当运行完 `this()` 后，发现 `beanFactory` 已经有值了，类型为 `DefaultListableBeanFactory`。但是，在查看 `beanFactory` 对象时，发现 `beanFactory` 的属性太多了，我们应该重点关注啥呢？

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1b617cf7edda29c652a7661d4be3779ec85.png)

我们这部分主要关注 spring bean 的创建，因此只需要关注 `beanFactory` 的两个属性就可以了：

*   beanDefinitionMap：存放 beanDefinition 的 map.
*   singletonObjects：存放 spring bean 的 map，spring bean 创建后都存放在这里，也即直观上理解的 `spring 容器`.

> `BeanDefinition` 是 spring 重要组件之一，为‘spring bean 的描述’，简单来说，就是说明了一个 spring bean 应该如何创建。关于 `BeanDefinition` 的详细分析，可以查看 [spring BeanDefinition 分析](https://my.oschina.net/funcy/blog/4597536)。

我们手动添加变量，如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0c6368478258f8b9f76b47fc1c85b02f13f.png)

可以看到，此时的 `beanDefinitionMap` 中已经有 4 个对象了，显然是在 `this()` 方法中添加的，关于这块我们后面会分析。

接着运行，发现 `beanDefinitionMap` 又多了两个：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-a493061fbd4b4066f9a4d91e91ff61e8c4e.png)

这里的 `beanObj1` 与 `beanObj2` 就是我们自己的类了，由此可以判断出 **spring 就是在 `AnnotationConfigApplicationContext#scan` 方法中对包进行扫描的**。

接下来，代码执行进入 `AbstractApplicationContext#refresh` 方法，我们一行行运行下去，发现运行到 `prepareBeanFactory(beanFactory);` 时，`singletonObjects` 中第一次出现了对象：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8cebcb82f5a8754fd1bb4bb3eb3c57dda2d.png)

可以看到，这里出现了 3 个类，基本都跟系统、环境相关，如 `environment` 是 spring 当前使用的环境 (`profile`)，`systemProperties` 当前系统的属性（操作系统、操作系统版本等）。

继续往下运行，发现代码运行到 `invokeBeanFactoryPostProcessors(beanFactory)` 时，又多了 4 个类：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-baa09e51272baa384418cb2c82b9dfb079b.png)

关于这几个类的作用，我们后面的文章中会分析，这里先不必管。继续往下运行，发现在 `registerBeanPostProcessors(beanFactory);` 中，又多了一个对象：

```
org.springframework.context.annotation.internalAutowiredAnnotationProcessor

```

这里我们依旧不用管这个对象，接着运行下去，可以看到在运行 `initMessageSource()` 时，又多了一个对象：

```
messageSource -> {DelegatingMessageSource@1847} "Empty MessageSource"

```

显然，这个对象是用来处理国际化问题的，不过由于 demo01 中并没有用到国际化，所以这里显示 `Empty MessageSource`。继续运行，发现运行到 `initApplicationEventMulticaster();` 时，又多了一个对象：

```
applicationEventMulticaster -> {SimpleApplicationEventMulticaster@1869} 

```

显然，这个对象是用来处理 `ApplicationContext` 的广播事件的，我们的 demo 中并没有用到，暂时不必理会。继续下去，发现在运行完 `finishBeanFactoryInitialization(beanFactory);`，`singletonObjects` 中终于出现了我们期待的对象：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-68b1ee71e468ef8cf839230c07b64c45563.png)

由此可见，对象就是在该方法中创建的。

### 总结

1.  spring 包的描述：`AnnotationConfigApplicationContext#scan`
2.  spring bean 的创建：`AbstractApplicationContext#finishBeanFactoryInitialization`

本文主要是了解 spring 启动流程，从整体上把握 spring 启动过程中的 beanFactory 的变化。本文意在了解 spring 的整体启动流程，后续的分析中，我们将对这些流程进行展开分析。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4597493](https://my.oschina.net/funcy/blog/4597493) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_