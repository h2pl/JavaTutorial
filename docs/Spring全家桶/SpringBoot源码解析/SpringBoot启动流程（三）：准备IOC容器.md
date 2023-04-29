上一篇文章总结 springboot 启动流程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-07a6b491fbe69b8dcbd41e59a8543f06671.png)

接上文，我们继续分析接下来的步骤。

### 3.8 创建 ioc 容器

创建 ioc 容器的代码如下：

```
ConfigurableApplicationContext context = null;
....
// 创建applicationContext
context = createApplicationContext();

```

我们进入 `SpringApplication#createApplicationContext` 方法：

```
/** 默认的 ApplicationContext */
public static final String DEFAULT_CONTEXT_CLASS = "org.springframework.context."
        + "annotation.AnnotationConfigApplicationContext";

/** servlet 应用的的 ApplicationContext */
public static final String DEFAULT_SERVLET_WEB_CONTEXT_CLASS = "org.springframework.boot."
        + "web.servlet.context.AnnotationConfigServletWebServerApplicationContext";

/** Reactive 应用的 ApplicationContext */
public static final String DEFAULT_REACTIVE_WEB_CONTEXT_CLASS = "org.springframework."
        + "boot.web.reactive.context.AnnotationConfigReactiveWebServerApplicationContext";

protected ConfigurableApplicationContext createApplicationContext() {
    Class<?> contextClass = this.applicationContextClass;
    if (contextClass == null) {
        try {
            // 根据应用类型来创建不同的容器
            switch (this.webApplicationType) {
            case SERVLET:
                // 使用的是 AnnotationConfigServletWebServerApplicationContext
                contextClass = Class.forName(DEFAULT_SERVLET_WEB_CONTEXT_CLASS);
                break;
            case REACTIVE:
                contextClass = Class.forName(DEFAULT_REACTIVE_WEB_CONTEXT_CLASS);
                break;
            default:
                // 默认使用的是 AnnotationConfigApplicationContext
                contextClass = Class.forName(DEFAULT_CONTEXT_CLASS);
            }
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(...);
        }
    }
    // 使用反射进行实例化
    return (ConfigurableApplicationContext) BeanUtils.instantiateClass(contextClass);
}

```

这个方法主要就是根据应用类型来创建不同 `ApplicationContext`，使用反射的方法进行实例化，各应用类型对应的 `ApplicationContext` 如下：

1.  `servlet` 应用：`AnnotationConfigServletWebServerApplicationContext`
2.  `reactive` 应用：`AnnotationConfigReactiveWebServerApplicationContext`
3.  以上都不是：`AnnotationConfigApplicationContext`

当前应用的类型是 `servlet`，因此创建的 `ApplicationContext` 是 `AnnotationConfigReactiveWebServerApplicationContext`，来看看它的构造方法：

```
public class AnnotationConfigServletWebServerApplicationContext 
        extends ServletWebServerApplicationContext implements AnnotationConfigRegistry {

    // 用来处理 BeanDefinition 的注册
    private final AnnotatedBeanDefinitionReader reader;

    // 用来处理包的扫描
    private final ClassPathBeanDefinitionScanner scanner;

    ...

    public AnnotationConfigServletWebServerApplicationContext() {
        this.reader = new AnnotatedBeanDefinitionReader(this);
        this.scanner = new ClassPathBeanDefinitionScanner(this);
    }

    ...
}

```

`AnnotationConfigServletWebServerApplicationContext` 的构造方法还是比较简单的，只是设置了两个属性，就不多说了。不过，我们也要把目光放远一点，看看其父类的构造方法，最终在 `GenericApplicationContext` 的构造方法中找到这么一句：

```
public GenericApplicationContext() {
    this.beanFactory = new DefaultListableBeanFactory();
}

```

这行代码创建了 `DefaultListableBeanFactory` 并将其赋值给了 `beanFactory`，这表明我们在 `ApplicationContext` 使用的 `beanFactory` 就是 `DefaultListableBeanFactory`。

### 3.9 准备 ioc 容器

创建完 ioc 容器后，接着就是对容器进行一些准备操作，代码如下：

```
public class SpringApplication {

    ...

    private void prepareContext(ConfigurableApplicationContext context, 
            ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, 
            ApplicationArguments applicationArguments, Banner printedBanner) {
        // 将创建好的应用环境设置到IOC容器中
        context.setEnvironment(environment);
        // 设置容器的一些参数
        postProcessApplicationContext(context);
        // 应用Initializer进行初始化操作
        applyInitializers(context);
        //  监听器：SpringApplicationRunListeners的contextPrepared方法
        // （在创建和准备ApplicationContext之后，但在加载之前）
        listeners.contextPrepared(context);
        // 打印个日志
        if (this.logStartupInfo) {
            logStartupInfo(context.getParent() == null);
            logStartupProfileInfo(context);
        }
        // 获取beanFactory
        ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
        // 将运行参数作为bean注册到beanFactory中
        beanFactory.registerSingleton("springApplicationArguments", applicationArguments);
        // 将 banner 作为bean注册到beanFactory中
        if (printedBanner != null) {
            beanFactory.registerSingleton("springBootBanner", printedBanner);
        }
        if (beanFactory instanceof DefaultListableBeanFactory) {
            // 是否允许bean的信息被覆盖
            ((DefaultListableBeanFactory) beanFactory)
                    .setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
        }
        // 处理懒加载
        if (this.lazyInitialization) {
            context.addBeanFactoryPostProcessor(
                    new LazyInitializationBeanFactoryPostProcessor());
        }
        // 获取所有资源
        Set<Object> sources = getAllSources();
        Assert.notEmpty(sources, "Sources must not be empty");
        // 加载 class
        load(context, sources.toArray(new Object[0]));
        // 发布事件
        listeners.contextLoaded(context);
    }
}

```

准备工作的步骤还是很清晰的，重要内容都已在代码中进行了注释，这里对一些操作我们稍微展开下。

#### 1\. 设置 `Environment`

该操作的代码为

```
private void prepareContext(ConfigurableApplicationContext context, 
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, 
        ApplicationArguments applicationArguments, Banner printedBanner) {
    // 将创建好的应用环境设置到IOC容器中
    context.setEnvironment(environment);
    ...
}

```

这个 `environment` 就是前面创建的 `environment`，ioc 容器也是使用这个，我们来看看设置操作：

```
public class AnnotationConfigServletWebServerApplicationContext extends ... {

    ...

    @Override
    public void setEnvironment(ConfigurableEnvironment environment) {
        // 调用父类的方法进行设置
        super.setEnvironment(environment);
        // 也将environment设置到这两个属性中
        this.reader.setEnvironment(environment);
        this.scanner.setEnvironment(environment);
    }

    ....
}

```

#### 2\. 处理 ioc 的部分属性

我们来看看 `postProcessApplicationContext(context);` 所做的工作：

```
public class SpringApplication {

    ...

    protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
        // 设置 beanNameGenerator，用来生成 bean 的名称，这里传入的是null
        if (this.beanNameGenerator != null) {
            context.getBeanFactory().registerSingleton(
                    AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                    this.beanNameGenerator);
        }
        // 设置 resourceLoader的相关参数，由于为null，if里的代码不会执行
        if (this.resourceLoader != null) {
            if (context instanceof GenericApplicationContext) {
                ((GenericApplicationContext) context).setResourceLoader(this.resourceLoader);
            }
            if (context instanceof DefaultResourceLoader) {
                ((DefaultResourceLoader) context).setClassLoader(
                        this.resourceLoader.getClassLoader());
            }
        }
        // 这里会执行
        if (this.addConversionService) {
            // 设置类型转换器，如把String转Number等
            context.getBeanFactory().setConversionService(
                    ApplicationConversionService.getSharedInstance());
        }
    }

    ...

}

```

这部分就是设置 `ApplicationContext` 的几个属性，在我们的 demo 中，`beanNameGenerator` 与 `resourceLoader` 都是 `null`，因此这两块都不会运行，能运行的就只有这块代码：

```
 context.getBeanFactory().setConversionService(
        ApplicationConversionService.getSharedInstance());

```

`ConversionService` 在前面也提到过，它主要是用来进行参数类型转换的。

#### 3\. 应用初始化器：`applyInitializers(context)`

`SpringApplication#applyInitializers` 方法如下：

```
public class SpringApplication {

    ...

    @SuppressWarnings({ "rawtypes", "unchecked" })
    protected void applyInitializers(ConfigurableApplicationContext context) {
        // getInitializers()：获取所有的初始化器
        for (ApplicationContextInitializer initializer : getInitializers()) {
            Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(initializer.getClass(),
                    ApplicationContextInitializer.class);
            Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
            // 逐一调用initializer的initialize(...)方法
            initializer.initialize(context);
        }
    }

}

```

这块操作还是很清晰的，就是获取所有的 `Initializer`，然后遍历，逐个调用其 `initialize(...)` 方法。

这里还需要提下，`getInitializers()` 是怎么获取 `Initializer` 的呢？相关代码如下：

```
public class SpringApplication {

    private List<ApplicationContextInitializer<?>> initializers;

    ...

    // 在构造方法中设置的 initializers
    public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
        ...

        // 设置初始化器，getSpringFactoriesInstances：从 META-INF/spring.factories 中获取配置
        setInitializers((Collection) getSpringFactoriesInstances(
                ApplicationContextInitializer.class));

        ...
    }

    // 获取 Initializer 的操作，从属性中获取
    public Set<ApplicationContextInitializer<?>> getInitializers() {
        return asUnmodifiableOrderedSet(this.initializers);
    }

    ...

```

看了代码就明白了，在前面分析 `SpringApplication` 的构造方法时，我们提到 springboot 会从 `META-INF/spring.factories` 获取配置的 `Initializer`，将其设置到 `initializers` 属性，而这里就是使用 `Initializer` 的地方了。

#### 4\. 获取所有资源

代码如下：

```
private void prepareContext(ConfigurableApplicationContext context, 
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, 
        ApplicationArguments applicationArguments, Banner printedBanner) {
    ...
    // 获取所有资源
    Set<Object> sources = getAllSources();
    ...
}

```

进入 `getAllSources()` 方法：

```
// 将 primarySources 放入set中，然后将set转化为不可变的set，返回
public Set<Object> getAllSources() {
    Set<Object> allSources = new LinkedHashSet<>();
    if (!CollectionUtils.isEmpty(this.primarySources)) {
        allSources.addAll(this.primarySources);
    }
    // sources 为空，if不执行
    if (!CollectionUtils.isEmpty(this.sources)) {
        allSources.addAll(this.sources);
    }
    return Collections.unmodifiableSet(allSources);
}

```

这个方法很简单，就一个话：将 `primarySources` 放入 `set` 中，然后将 `set` 转化为不可变的 `set`，返回。

那这个 `primarySources` 是啥呢？这就又要回到 `SpringApplication` 的构造方法了：

```
public SpringApplication(ResourceLoader resourceLoader, Class<?>... primarySources) {
    ...
    // 这里设置了 primarySources
    Assert.notNull(primarySources, "PrimarySources must not be null");
    this.primarySources = new LinkedHashSet<>(Arrays.asList(primarySources));
    ...
}

```

然后继续往上，最后发现 `primarySources` 是我们在 `main` 方法中传入的：

```
@SpringBootApplication
public class Demo01Application {

    public static void main(String[] args) {
        // Demo01Application.class 就是 primarySources
        SpringApplication.run(Demo01Application.class, args);
    }

}

```

我们传入的 `Demo01Application.class` 就是 `primarySources`！

因此，`getAllSources()` 返回了一个 set，set 中只有一个元素：`Demo01Application.class`，我们也可以通过调试来验证：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-29dd8056dc457c56e6fe41020516d02fe80.png)

#### 5\. 加载资源

这个操作的代码如下：

```
private void prepareContext(ConfigurableApplicationContext context, 
        ConfigurableEnvironment environment, SpringApplicationRunListeners listeners, 
        ApplicationArguments applicationArguments, Banner printedBanner) {
    ...
    // 加载 class
    load(context, sources.toArray(new Object[0]));
    ...
}

```

进入 `SpringApplication#load` 方法：

```
public class SpringApplication {
    ...

    protected void load(ApplicationContext context, Object[] sources) {
        // 创建一个BeanDefinition的加载器
        BeanDefinitionLoader loader = createBeanDefinitionLoader(
                getBeanDefinitionRegistry(context), sources);
        // 当前 beanNameGenerator 为 null，不调用
        if (this.beanNameGenerator != null) {
            loader.setBeanNameGenerator(this.beanNameGenerator);
        }
        // 当前 resourceLoader 为 null，不调用
        if (this.resourceLoader != null) {
            loader.setResourceLoader(this.resourceLoader);
        }
        // 当前 environment 为 null，不调用
        // 前面创建的environment并没有赋值到成员变量
        if (this.environment != null) {
            loader.setEnvironment(this.environment);
        }
        loader.load();
    }
    ...
}

```

这个方法先是创建了一个 `BeanDefinitionLoader` 实例，上面获取的 `source` 会在 `BeanDefinitionLoader` 的构造方法中传入到实例中，然后对该 `loader` 进行一系列的属性设置，最后再调用其 `load()` 方法。这里需要说明的是，虽然前面创建了 `environment`，但 `this.environment` 还是为 `null`，原因是前面创建的 `environment` 并没有赋值给 `this.environment`。

我们继续，进入 `BeanDefinitionLoader#load()`：

```
class BeanDefinitionLoader {

    ...

    int load() {
        int count = 0;
        // 这里的 sources 中就只有一个元素：Demo01Application.class
        for (Object source : this.sources) {
            count += load(source);
        }
        return count;
    }

    // 处理加载操作
    private int load(Object source) {
        Assert.notNull(source, "Source must not be null");
        // Class 类型在这里加载
        if (source instanceof Class<?>) {
            // source 为 Demo01Application.class，我们主要关注这个方法
            return load((Class<?>) source);
        }
        if (source instanceof Resource) {
            return load((Resource) source);
        }
        if (source instanceof Package) {
            return load((Package) source);
        }
        if (source instanceof CharSequence) {
            return load((CharSequence) source);
        }
        throw new IllegalArgumentException("Invalid source type " + source.getClass());
    }

    // Class 类型的加载操作
    private int load(Class<?> source) {
        // 处理 grouovy 语言的，不用管
        if (isGroovyPresent() && GroovyBeanDefinitionSource.class.isAssignableFrom(source)) {
            GroovyBeanDefinitionSource loader 
                    = BeanUtils.instantiateClass(source, GroovyBeanDefinitionSource.class);
            load(loader);
        }
        // 是否有 @Component 注解
        if (isComponent(source)) {
            // 在这里生成 BeanDefinition 对象，并注册到spring中
            this.annotatedReader.register(source);
            return 1;
        }
        return 0;
    }
    ...

```

在创建 `BeanDefinitionLoader` 实例时，通过其构造方法将 `sources` 传入了 `BeanDefinitionLoader` 的实例中，`sources` 中就只有一个元素：`Demo01Application.class`，因此我们只需关注 `Class` 类型资源的加载就可以了，最终到了 `BeanDefinitionLoader#load(java.lang.Class<?>)` 方法，该方法所做的操作为：判断传入的 `Class` 是否有 `@Component`，如果有就将其注册到 ioc 容器中。

那么 `Demo01Application.class` 是否有 `@Component` 注解呢？有的，不过隐藏得比较深，其注解层级如下：

```
@SpringBootApplication
public class Demo01Application {
    ...
}

```

进入 `@SpringBootApplication`：

```
...
@SpringBootConfiguration
...
public @interface SpringBootApplication {
    ...
}

```

继续进入 `@SpringBootApplication`：

```
...
@Configuration
public @interface SpringBootConfiguration {
    ...
}

```

继续进入 `@Configuration`：

```
...
@Component
public @interface Configuration {
    ...
}

```

经过重重波折，最终在 `@Configuration` 注解上找到了 `@Component`.

关于 `this.annotatedReader.register(source)` 注册的具体操作，这块在 [spring 分析](https://my.oschina.net/funcy/blog/4527454)中，已经分析过了，这里就不再分析了。

限于篇幅，本文就到这里了，下篇继续分析剩下的流程。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-fba95ddd84c68cdd1757f060ab131478a3c.png)

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4884127](https://my.oschina.net/funcy/blog/4884127) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_