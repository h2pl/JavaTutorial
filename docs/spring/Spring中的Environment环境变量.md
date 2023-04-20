



如今，致力于帮助开发者用更少的代码、更快地写出生产级系统的 Spring Boot 已然成为 Java 应用开发的事实标准。在 Spring Boot 提供的众多特性中，**自动配置**无疑是对提升开发体验最显著的一个特性，Spring Boot 基于这一特性为开发人员自动声明了若干开箱即用、具备某一功能的 Bean。大多数情况下，自动配置的 Bean 刚好能满足大家的需求，但在某些情况下，不得不完整地覆盖它们，这个时候只需要重新声明相关类型的 Bean 即可，因为绝大多数自动配置的 Bean 都会由`@ConditionalOnMissingBean`注解修饰。幸运的是，如果只是想微调一些细节，比如改改端口号 (server.port) 和数据源 URL (spring.datasource.url) ，那压根没必要重新声明`ServerProperties`和`DataSourceProperties`这俩 Bean 来覆盖自动配置的 Bean。 Spring Boot 为自动配置的 Bean 提供了1000多个用于微调的属性，当需要调整设置时，只需要在环境变量、命令行参数或配置文件 (application.properties/application.yml) 中进行指定即可，这就是 Spring Boot 的`Externalized Configuration` (配置外化) 特性。

当然，外部配置源并不局限于环境变量、命令行参数和配置文件这三种，感兴趣的读者可以自行阅读 Spring Boot 官方文档。在 Spring 中，`BeanFactory`扮演着 Bean 容器的角色，而`Environment`同样定位为一个容器，即外部配置源中的属性都会被添加到 _Environment_ 中。**在微服务大行其道的今天，外部配置源又衍生出了_Disconf_、_Apollo_ 和 _Nacos_ 等分布式配置中心，但在 Spring 的地盘，还是要入乡随俗，从配置中心中读取到的属性依然会被追加到 _Environment_ 中**。

笔者之所以写这篇文章，是受`jasypt`组件的启发。第一次接触它是在2018年，当时就很好奇这玩意儿究竟是如何实现对敏感属性加解密的；现在来看，要想实现这么一个东东，不仅需要熟悉 Bean 的生命周期、IoC 容器拓展点 (IoC Container Extension Points) 和 Spring Boot 的启动流程等知识，还需要掌握 _Environment_。

> jasypt 上手十分简单。首先通过`jasypt-maven-plugin`这一 maven 插件为敏感属性值生成密文，然后用`ENC(密文)`替换敏感属性值即可。如下：
>
> ```
> jasypt.encryptor.password=crimson_typhoon
> 
> spring.datasource.url=jdbc:mysql://HOST:PORT/db_sql_boy?characterEncoding=UTF-8
> spring.datasource.hikari.driver-class-name=com.mysql.cj.jdbc.Driver
> spring.datasource.hikari.username=root
> spring.datasource.hikari.password=ENC(qS8+DEIlHxvhPHgn1VaW3oHkn2twrmwNOHewWLIfquAXiCDBrKwvIhDoqalKyhIF)
> 复制代码
> ```

## 1 认识 Environmnent

在实际工作中，我们与 _Environment_ 打交道的机会并不多；如果业务 Bean 确实需要获取外部配置源中的某一属性值，可以手动将 _Environment_ 注入到该业务 Bean 中，也可以直接实现`EnvironmentAware`接口，得到 _Environment_ 类型的 Bean 实例之后可以通过`getProperty()`获取具体属性值。_Environment_ 接口内容如下所示：

```
public interface Environment extends PropertyResolver {
    String[] getActiveProfiles();
    String[] getDefaultProfiles();
    boolean acceptsProfiles(Profiles profiles);
}

public interface PropertyResolver {
    boolean containsProperty(String key);
    String getProperty(String key);
    String getProperty(String key, String defaultValue);
    <T> T getProperty(String key, Class<T> targetType);
    <T> T getProperty(String key, Class<T> targetType, T defaultValue);
    String resolvePlaceholders(String text);
}
复制代码
```

**大家不要受 _Environment_ 中 _getProperty()_ 方法的误导，外部配置源中的属性并不是以单个属性为维度被添加到 _Environment_ 中的，而是以`PropertySource`为维度**。_PropertySource_ 是对属性源名称和该属性源中一组属性的抽象，`MapPropertySource`是一种最简单的实现，它通过 _Map<String, Object>_ 来承载相关的属性。_PropertySource_ 内容如下：

```
public abstract class PropertySource<T> {
    protected final String name;
    protected final T source;

    public PropertySource(String name, T source) {
        this.name = name;
        this.source = source;
    }

    public String getName() { return this.name; }
    public T getSource() { return this.source; }
    public abstract Object getProperty(String name);
}
复制代码
```

从上述 _PropertySource_ 内容来看，_PropertySource_ 自身是具备根据属性名获取属性值这一能力的。

####  ****getProperty()内部执行逻辑****

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/QQ%E6%88%AA%E5%9B%BE20230416193319.jpg)

一般，_Environment_ 实现类中会持有一个`PropertyResolver`类型的成员变量，进而交由 _PropertyResolver_ 负责执行 _getProperty()_ 逻辑。_PropertyResolver_ 实现类中又会持有两个成员变量，分别是：`ConversionService`与`PropertySources`；首先，_PropertyResolver_ 遍历 `PropertySources` 中的 _PropertySource_，获取原生属性值；然后委派 _ConversionService_ 对原生属性值进行数据类型转换 (如果有必要的话)。**虽然 PropertySource 自身是具备根据属性名获取属性值这一能力的，但不具备占位符解析与类型转换能力，于是在中间引入具备这两种能力的 PropertyResolver， 这也印证了一个段子：在计算机科学中，没有什么问题是在中间加一层解决不了的，如果有，那就再加一层**。

####  ****PropertySource内部更新逻辑****

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/QQ%E6%88%AA%E5%9B%BE20230416193319.jpg)

_Environment_ 实现类中除了持有`PropertyResolver`类型的成员变量外，还有一个`MutablePropertySources`类型的成员变量，但并不提供直接操作该 _MutablePropertySources_ 的方法，我们只能通过`getPropertySources()`方法获取 _MutablePropertySources_ 实例，然后借助 _MutablePropertySources_ 中的`addFirst()`、`addLast()`和`replace()`等方法去更新 _PropertySource_。_MutablePropertySources_ 是 _PropertySources_ 唯一一个实现类，如下图所示：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416193420.png)

总的来说，_Environment_ 是对 _PropertySource_ 和 _Profile_ 的顶级抽象，下面介绍 _Profile_ 的概念。当应用程序需要部署到不同的运行环境时，一些属性项通常会有所不同，比如，数据源 URL 在开发环境和测试环境就会不一样。Spring 从3.1版本开始支持基于 _Profile_ 的条件化配置。

**Profile in Spring 3.1**

在 Spring 发布3.1版本时，Spring Boot 还未问世，可以说此时的 _Profile_ 特性还是有些**瑕疵**的，但瑕不掩瑜。主要体现在：针对同一类型的 Bean，必须声明多次。一起来感受下这种小瑕疵：

```
@Configuration(proxyBeanMethods = false)
public class DataSourceConfig {
    @Bean
    @Profile("dev")
    public DataSource devDataSource () {
        return DataSourceBuilder.create()
                .driverClassName("com.mysql.jdbc.Driver")
                .url("jdbc:mysql://DEV_HOST:PORT/db_sql_boy?characterEncoding=UTF-8")
                .username("dev")
                .password("dev")
                .build();
    }

    @Bean
    @Profile("test")
    public DataSource testDataSource () {
        return DataSourceBuilder.create()
                .driverClassName("com.mysql.jdbc.Driver")
                .url("jdbc:mysql://TEST_HOST:PORT/db_sql_boy?characterEncoding=UTF-8")
                .username("test")
                .password("test")
                .build();
    }
}
复制代码
```

**Profile in Spring Boot**

Spring Boot 发布后，`@Profile`注解可以扔到九霄云外了。官方开发大佬肯定也意识到 _Profile in Spring 3.1_ 中这种瑕疵，于是在 Spring Boot 的第一版本 _(1.0.0.RELEASE)_ 中就迫不及待地支持为 _application.properties_ 和 _application.yml_ 里的属性项配置 _Profile_ 了。换个口味，一起来感受下这种优雅：

```
@Configuration(proxyBeanMethods = false)
public class DataSourceConfig {
    @Bean
    public DataSource devDataSource (DataSourceProperties dataSourceProperties) {
        return DataSourceBuilder.create()
                .driverClassName(dataSourceProperties.getDriverClassName())
                .url(dataSourceProperties.getUrl())
                .username(dataSourceProperties.getUsername())
                .password(dataSourceProperties.getPassword())
                .build();
    }
}
复制代码
```

_application-dev.properties_ 内容如下：

```
spring.datasource.url=jdbc:mysql://DEV_HOST:PORT/db_sql_boy?characterEncoding=UTF-8
spring.datasource.hikari.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.hikari.password=dev
spring.datasource.hikari.username=dev
复制代码
```

_application-test.properties_ 内容如下：

```
spring.datasource.url=jdbc:mysql://TEST_HOST:PORT/db_sql_boy?characterEncoding=UTF-8
spring.datasource.hikari.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.hikari.password=test
spring.datasource.hikari.username=test
复制代码
```

在原生 Spring 3.1 和 Spring Boot 中，均是通过`spring.profiles.active`来为 _Environment_ 指定激活的 _Profile_，否则_Environment_ 中默认激活的 _Profile_ 名称为`default`。写到这里，笔者脑海中闪现一个问题：一般，`@Profile` 注解主要与 _@Configuration_ 注解或 _@Bean_ 注解搭配使用，如果 _spring.profiles.active_ 的值为 _dev_ 时，那么那些由 _@Configuration_ 或 _@Bean_ 注解标记 (但没有`@Profile`注解的身影哈) 的 Bean 还会被解析为若干`BeanDefinition`实例吗？答案是会的。`ConfigurationClassPostProcessor`负责将 _@Configuration_ 配置类解析为 _BeanDefinition_，在此过程中会执行`ConditionEvaluator`的`shouldSkip()`方法，主要内容如下：

```
public class ConditionEvaluator {
    public boolean shouldSkip(AnnotatedTypeMetadata metadata, ConfigurationCondition.ConfigurationPhase phase) {
        if (metadata == null || !metadata.isAnnotated(Conditional.class.getName())) {
            return false;
        }

        if (phase == null) {
            if (metadata instanceof AnnotationMetadata &&
                    ConfigurationClassUtils.isConfigurationCandidate((AnnotationMetadata) metadata)) {
                return shouldSkip(metadata, ConfigurationCondition.ConfigurationPhase.PARSE_CONFIGURATION);
            }
            return shouldSkip(metadata, ConfigurationCondition.ConfigurationPhase.REGISTER_BEAN);
        }

        List<Condition> conditions = new ArrayList<>();
        for (String[] conditionClasses : getConditionClasses(metadata)) {
            for (String conditionClass : conditionClasses) {
                Condition condition = getCondition(conditionClass, this.context.getClassLoader());
                conditions.add(condition);
            }
        }

        AnnotationAwareOrderComparator.sort(conditions);

        for (Condition condition : conditions) {
            ConfigurationCondition.ConfigurationPhase requiredPhase = null;
            if (condition instanceof ConfigurationCondition) {
                requiredPhase = ((ConfigurationCondition) condition).getConfigurationPhase();
            }
            if ((requiredPhase == null || requiredPhase == phase) && !condition.matches(this.context, metadata)) {
                return true;
            }
        }

        return false;
    }
}
复制代码
```

`shouldSkip()`方法第一行 _if_ 语句就是答案，`@Profile`注解由`@Conditional(ProfileCondition.class)`修饰，那如果一个配置类头上没有`Condition`的身影，直接返回`false`了，那就是不跳过该配置类的意思喽！

_Environment_ 中的这些 _PropertySource_ 究竟有啥用啊？当然是为了填充 _Bean_ 喽，废话不多说，上图。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416193443.png)

> 笔者以前都是用 visio 和 processOn 画图，第一次体验 draw.io，没想到如此优秀，强烈安利一波！

## 2 Environmnent 初始化流程

本节主要介绍 Spring Boot 在启动过程中向 _Environmnt_ 中究竟注册了哪些 _PropertySource_。启动入口位于`SpringApplication`中的`run(String... args)`方法，如下：

```
public class SpringApplication {
    public ConfigurableApplicationContext run(String... args) {
        StopWatch stopWatch = new StopWatch();
        stopWatch.start();
        DefaultBootstrapContext bootstrapContext = createBootstrapContext();
        ConfigurableApplicationContext context = null;
        configureHeadlessProperty();
        SpringApplicationRunListeners listeners = getRunListeners(args);
        listeners.starting(bootstrapContext, this.mainApplicationClass);
        try {
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
            ConfigurableEnvironment environment = prepareEnvironment(listeners, bootstrapContext, applicationArguments);
            configureIgnoreBeanInfo(environment);
            Banner printedBanner = printBanner(environment);
            context = createApplicationContext();
            context.setApplicationStartup(this.applicationStartup);
            prepareContext(bootstrapContext, context, environment, listeners, applicationArguments, printedBanner);
            refreshContext(context);
            afterRefresh(context, applicationArguments);
            stopWatch.stop();
            if (this.logStartupInfo) {
                new StartupInfoLogger(this.mainApplicationClass).logStarted(getApplicationLog(), stopWatch);
            }
            listeners.started(context);
            callRunners(context, applicationArguments);
        } catch (Throwable ex) {
            handleRunFailure(context, ex, listeners);
            throw new IllegalStateException(ex);
        }

        try {
            listeners.running(context);
        } catch (Throwable ex) {
            handleRunFailure(context, ex, null);
            throw new IllegalStateException(ex);
        }
        return context;
    }
}
复制代码
```

可以明显看出，_Environmnt_ 的初始化是在`refreshContext(context)`之前完成的，这是毫无疑问的。_run()_ 方法很复杂，但与本文主题契合的逻辑只有**一**处：

```
prepareEnvironment(listeners, bootstrapContext, applicationArguments);
复制代码
```

下面分别分析这两处核心逻辑。

### 2.1 prepareEnvironment()

显然，核心内容都在`prepareEnvironment()`方法内，下面分小节逐一分析。

```
public class SpringApplication {
    private ConfigurableEnvironment prepareEnvironment(SpringApplicationRunListeners listeners,
                                                       DefaultBootstrapContext bootstrapContext,
                                                       ApplicationArguments applicationArguments) {
        // 2.1.1
        ConfigurableEnvironment environment = getOrCreateEnvironment();
        // 2.1.2
        configureEnvironment(environment, applicationArguments.getSourceArgs());
        // 2.1.3
        ConfigurationPropertySources.attach(environment);
        // 2.1.4
        listeners.environmentPrepared(bootstrapContext, environment);
        DefaultPropertiesPropertySource.moveToEnd(environment);
        bindToSpringApplication(environment);
        ConfigurationPropertySources.attach(environment);
        return environment;
    }
}
复制代码
```

#### 2.1.1 getOrCreateEnvironment()

`getOrCreateEnvironment()`主要负责构建 _Environment_ 实例。如果当前应用是基于`同步阻塞I/O`模型的，则 _Environment_ 选用`ApplicationServletEnvironment`；相反地，如果当前应用是基于`异步非阻塞I/O`模型的，则 _Environment_ 选用`ApplicationReactiveWebEnvironment`。我们工作中基本都是基于 Spring MVC 开发应用，Spring MVC 是一款构建于`Servlet API`之上、基于同步阻塞 I/O 模型的主流 Java Web 开发框架，这种 I/O 模型意味着一个 HTTP 请求对应一个线程，即每一个 HTTP 请求都是在各自线程上下文中完成处理的。_ApplicationServletEnvironment_ 继承关系如下图所示：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416193515.png)

从上图可以看出 _ApplicationServletEnvironment_ 家族相当庞大，在执行 _ApplicationServletEnvironment_ 构造方法的时候必然会触发各级父类构造方法中的逻辑，**依次为**：

```
public abstract class AbstractEnvironment implements ConfigurableEnvironment {
    public AbstractEnvironment() {
        this(new MutablePropertySources());
    }

    protected AbstractEnvironment(MutablePropertySources propertySources) {
        this.propertySources = propertySources;
        // createPropertyResolver(propertySources)
        // |___ ConfigurationPropertySources.createPropertyResolver(propertySources)
        //      |___ new ConfigurationPropertySourcesPropertyResolver(propertySources)
        this.propertyResolver = createPropertyResolver(propertySources);
        customizePropertySources(propertySources);
    }
}
复制代码
```

```
public class StandardServletEnvironment extends StandardEnvironment implements ConfigurableWebEnvironment {
    @Override
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(new StubPropertySource("servletConfigInitParams"));
        propertySources.addLast(new StubPropertySource("servletContextInitParams"));
        super.customizePropertySources(propertySources);
    }
}
复制代码
```

```
public class StandardEnvironment extends AbstractEnvironment {
    @Override
    protected void customizePropertySources(MutablePropertySources propertySources) {
        propertySources.addLast(
                new PropertiesPropertySource("systemProperties", (Map) System.getProperties()));
        propertySources.addLast(
                new SystemEnvironmentPropertySource("systemEnvironment", (Map) System.getenv()));
    }
}
复制代码
```

随着 _ApplicationServletEnvironment_ 构造方法的执行，此时在 _Environment_ 里 _MutablePropertySources_ 类型的成员变量`propertySources`中已经有了**四**个 _PropertySource_ 了，名称依次是：`servletConfigInitParams`、`servletContextInitParams`、`systemProperties`和`systemEnvironment`。此外，也要记住 _ApplicationServletEnvironment_ 中的两个重要成员变量，即`MutablePropertySources`和`ConfigurationPropertySourcesPropertyResolver`。

#### 2.1.2 configureEnvironment()

`configureEnvironment()`方法中的逻辑也很简单哈。首先，为 _Environment_ 中的 _PropertySourcesPropertyResolver_ 设定 _ConversionService_；然后，向 _Environment_ 中的 _MutablePropertySources_ 追加一个名称为`commandLineArgs`的 _PropertySource_ 实例，注意使用的是`addFirst()`方法哦，这意味着这个名称为`commandLineArgs`的 _PropertySource_ 优先级是最高的。主要逻辑如下：

```
public class SpringApplication {
    protected void configureEnvironment(ConfigurableEnvironment environment, String[] args) {
        if (this.addConversionService) {
            environment.getPropertyResolver().setConversionService(new ApplicationConversionService());
        }
        if (this.addCommandLineProperties && args.length > 0) {
            MutablePropertySources sources = environment.getPropertySources();
            sources.addFirst(new SimpleCommandLinePropertySource(args));
        }
    }
}
复制代码
```

继续`SimpleCommandLinePropertySource`：

```
public class SimpleCommandLinePropertySource extends CommandLinePropertySource<CommandLineArgs> {
    public SimpleCommandLinePropertySource(String... args) {
        // 其父类构造方法为：super("commandLineArgs", source)
        super(new SimpleCommandLineArgsParser().parse(args));
    }
}
复制代码
```

命令行参数还是比较常用的，比如我们在启动 Spring Boot 应用时会这样声明命令行参数：`java -jar app.jar --server.port=8088`。

#### 2.1.3 ConfigurationPropertySources.attach()

`attach()`方法主要就是在 _Environment_ 中 _MutablePropertySources_ 的头部位置插入加一个名称为`configurationProperties`的 _PropertySource_ 实例。主要逻辑如下：

```
public final class ConfigurationPropertySources {
    public static void attach(org.springframework.core.env.Environment environment) {
        MutablePropertySources sources = ((ConfigurableEnvironment) environment).getPropertySources();
        PropertySource<?> attached = getAttached(sources);
        if (attached != null && attached.getSource() != sources) {
            sources.remove(ATTACHED_PROPERTY_SOURCE_NAME);
            attached = null;
        }
        if (attached == null) {
            sources.addFirst(new ConfigurationPropertySourcesPropertySource("configurationProperties", new SpringConfigurationPropertySources(sources)));
        }
    }

    static PropertySource<?> getAttached(MutablePropertySources sources) {
        return (sources != null) ? sources.get("configurationProperties") : null;
    }
}
复制代码
```

笔者盯着这玩意儿看了好久，压根没看出这个名称为`configurationProperties`的 _PropertySource_ 究竟有啥用。最后，还是在官方文档中关于`Relaxed Binding` (宽松绑定) 的描述中猜出了些端倪。还是通过代码来解读比较直接。首先，在 _application.properties_ 中追加一个配置项：`a.b.my-first-key=hello spring environment`；然后，通过 _Environment_ 取出这个配置项的值，如下：

```
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext configurableApplicationContext = SpringApplication.run(DemoApplication.class, args);
        ConfigurableWebEnvironment environment = (ConfigurableWebEnvironment)
                configurableApplicationContext.getBean(Environment.class);
        System.out.println(environment.getProperty("a.b.my-first-key"));
    }
}
复制代码
```

启动应用后，控制台打印出了 _hello spring environment_ 字样，这与预期是相符的。可当我们通过`environment.getProperty("a.b.myfirstkey")`或者`environment.getProperty("a.b.my-firstkey")`依然能够获取到配置项的内容。`a.b.myfirstkey`和`a.b.my-firstkey`并不是配置文件中的属性名称，只是相似而已，这的确很**宽松**啊，哈哈。感兴趣的读者可以自行 DEBUG 看看其中的原理。

#### 2.1.4 listeners.environmentPrepared()

敲黑板，各位大佬，这个要考的 ！`environmentPrepared()`方法会广播一个`ApplicationEnvironmentPreparedEvent`事件，接着由`EnvironmentPostProcessorApplicationListener`响应该事件，这应该是典型的**观察者模式**。主要内容如下：

```
public class SpringApplicationRunListeners {
    private final List<SpringApplicationRunListener> listeners;

    void environmentPrepared(ConfigurableBootstrapContext bootstrapContext, ConfigurableEnvironment environment) {
        doWithListeners("spring.boot.application.environment-prepared",
                (listener) -> listener.environmentPrepared(bootstrapContext, environment));
    }

    private void doWithListeners(String stepName, Consumer<SpringApplicationRunListener> listenerAction) {
        StartupStep step = this.applicationStartup.start(stepName);
        this.listeners.forEach(listenerAction);
        step.end();
    }
}

public class EventPublishingRunListener implements SpringApplicationRunListener, Ordered {
    @Override
    public void environmentPrepared(ConfigurableBootstrapContext bootstrapContext,
                                    ConfigurableEnvironment environment) {
        this.initialMulticaster.multicastEvent(
                new ApplicationEnvironmentPreparedEvent(bootstrapContext, this.application, this.args, environment));
    }
}

public class SimpleApplicationEventMulticaster extends AbstractApplicationEventMulticaster {
    @Override
    public void multicastEvent(ApplicationEvent event) {
        multicastEvent(event, resolveDefaultEventType(event));
    }

    @Override
    public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
        ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
        Executor executor = getTaskExecutor();
        for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
            if (executor != null) {
                executor.execute(() -> invokeListener(listener, event));
            } else {
                invokeListener(listener, event);
            }
        }
    }
}
复制代码
```

下面来看一下`EnvironmentPostProcessorApplicationListener`的庐山真面目：

```
public class EnvironmentPostProcessorApplicationListener implements SmartApplicationListener, Ordered {
    @Override
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof ApplicationEnvironmentPreparedEvent) {
            onApplicationEnvironmentPreparedEvent((ApplicationEnvironmentPreparedEvent) event);
        }
        if (event instanceof ApplicationPreparedEvent) {
            onApplicationPreparedEvent();
        }
        if (event instanceof ApplicationFailedEvent) {
            onApplicationFailedEvent();
        }
    }
    private void onApplicationEnvironmentPreparedEvent(ApplicationEnvironmentPreparedEvent event) {
        ConfigurableEnvironment environment = event.getEnvironment();
        SpringApplication application = event.getSpringApplication();
        for (EnvironmentPostProcessor postProcessor : getEnvironmentPostProcessors(application.getResourceLoader(), event.getBootstrapContext())) {
            postProcessor.postProcessEnvironment(environment, application);
        }
    }
}
复制代码
```

`EnvironmentPostProcessor`是 Spring Boot 为 _Environment_ 量身打造的扩展点。这里引用官方文档中比较精炼的一句话：_Allows for customization of the application's Environment prior to the application context being refreshed_。_EnvironmentPostProcessor_ 是一个函数性接口，内容如下：

```
public interface EnvironmentPostProcessor {
    void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application);
}
复制代码
```

在上述 _EnvironmentPostProcessorApplicationListener_ 事件处理逻辑中，`getEnvironmentPostProcessors`负责加载出所有的 _EnvironmentPostProcessor_ 。看一下内部加载逻辑：

```
public interface EnvironmentPostProcessorsFactory {
    static EnvironmentPostProcessorsFactory fromSpringFactories(ClassLoader classLoader) {
        return new ReflectionEnvironmentPostProcessorsFactory(
                classLoader, 
                SpringFactoriesLoader.loadFactoryNames(EnvironmentPostProcessor.class, classLoader)
        );
    }
}
复制代码
```

继续进入`SpringFactoriesLoader`一探究竟：

```
public final class SpringFactoriesLoader {

    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

    public static List<String> loadFactoryNames(Class<?> factoryType, ClassLoader classLoader) {
        ClassLoader classLoaderToUse = classLoader;
        if (classLoaderToUse == null) {
            classLoaderToUse = SpringFactoriesLoader.class.getClassLoader();
        }
        String factoryTypeName = factoryType.getName();
        return loadSpringFactories(classLoaderToUse).getOrDefault(factoryTypeName, Collections.emptyList());
    }

    private static Map<String, List<String>> loadSpringFactories(ClassLoader classLoader) {
        Map<String, List<String>> result = cache.get(classLoader);
        if (result != null) {
            return result;
        }

        result = new HashMap<>();
        try {
            Enumeration<URL> urls = classLoader.getResources(FACTORIES_RESOURCE_LOCATION);
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                UrlResource resource = new UrlResource(url);
                Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                for (Map.Entry<?, ?> entry : properties.entrySet()) {
                    String factoryTypeName = ((String) entry.getKey()).trim();
                    String[] factoryImplementationNames = StringUtils.commaDelimitedListToStringArray((String) entry.getValue());
                    for (String factoryImplementationName : factoryImplementationNames) {
                        result.computeIfAbsent(factoryTypeName, key -> new ArrayList<>())
                                .add(factoryImplementationName.trim());
                    }
                }
            }
            result.replaceAll((factoryType, implementations) -> implementations.stream().distinct()
                    .collect(Collectors.collectingAndThen(Collectors.toList(), Collections::unmodifiableList)));
            cache.put(classLoader, result);
        } catch (IOException ex) {
            throw new IllegalArgumentException("Unable to load factories from location [" + FACTORIES_RESOURCE_LOCATION + "]", ex);
        }
        return result;
    }
}
复制代码
```

> **Spring SPI**
>
> > _SpringFactoriesLoader_ 这一套逻辑就是 Spring 中的`SPI`机制；直白点说，就是从`classpath`下的`META-INF/spring.factories` 文件中加载 _EnvironmentPostProcessor_ ，如果大家有需求就将自己实现的 _EnvironmentPostProcessor_ 放到该文件中就行了。其实与`JDK`中的`SPI`机制很类似哈。

在当前版本，Spring Boot 内置了7个 _EnvironmentPostProcessor_ 实现类。接下来挑几个比较典型的分析下。

**RandomValuePropertySourceEnvironmentPostProcessor**

`RandomValuePropertySourceEnvironmentPostProcessor`向 _Environment_ 中追加了一个名称为`random`的 _PropertySource_，即`RandomValuePropertySource`。内容如下：

```
public class RandomValuePropertySourceEnvironmentPostProcessor implements EnvironmentPostProcessor, Ordered {
    public static final int ORDER = Ordered.HIGHEST_PRECEDENCE + 1;
    private final Log logger;

    public RandomValuePropertySourceEnvironmentPostProcessor(Log logger) {
        this.logger = logger;
    }

    @Override
    public int getOrder() {
        return ORDER;
    }

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        RandomValuePropertySource.addToEnvironment(environment, this.logger);
    }
}
复制代码
```

那么这个 _RandomValuePropertySource_ 有啥作用呢？主要就是用于生成随机数，比如：`environment.getProperty("random.int(5,10)")`可以获取一个随机数。以`random.int`为属性名可以获取一个 _int_ 类型的随机数；以`random.long`为属性名可以获取一个 _long_ 类型的随机数；以`random.int(5,10)`为属性名可以获取一个 _[5, 10}_ 区间内 _int_ 类型的随机数，更多玩法大家自行探索。

_SystemEnvironmentPropertySourceEnvironmentPostProcessor_

当前，_Environment_ 中已经存在一个名称为`systemEnvironment`的 _PropertySource_，即`SystemEnvironmentPropertySource`。`SystemEnvironmentPropertySourceEnvironmentPostProcessor`用于将该 _SystemEnvironmentPropertySource_ 替换为`OriginAwareSystemEnvironmentPropertySource`，咋有点“脱裤子放屁，多此一举”的感觉呢，哈哈。

```
public class SystemEnvironmentPropertySourceEnvironmentPostProcessor implements EnvironmentPostProcessor, Ordered {
    public static final int DEFAULT_ORDER = SpringApplicationJsonEnvironmentPostProcessor.DEFAULT_ORDER - 1;
    private int order = DEFAULT_ORDER;

    @Override
    public int getOrder() {
        return this.order;
    }

    @Override
    public void postProcessEnvironment(ConfigurableEnvironment environment, SpringApplication application) {
        String sourceName = "systemEnvironment";
        PropertySource<?> propertySource = environment.getPropertySources().get(sourceName);
        if (propertySource != null) {
            replacePropertySource(environment, sourceName, propertySource, application.getEnvironmentPrefix());
        }
    }
    private void replacePropertySource(ConfigurableEnvironment environment, String sourceName,
                                       PropertySource<?> propertySource, String environmentPrefix) {
        Map<String, Object> originalSource = (Map<String, Object>) propertySource.getSource();
        SystemEnvironmentPropertySource source = new OriginAwareSystemEnvironmentPropertySource(sourceName, originalSource, environmentPrefix);
        environment.getPropertySources().replace(sourceName, source);
    }
}
复制代码
```

**SpringApplicationJsonEnvironmentPostProcessor**

我们在通过`java -jar -Dspring.application.json={"name":"duxiaotou"} app.jar`启动 Spring Boot 应用的时候，该属性会被自动添加到 JVM 系统属性中 (其实 _-Dkey=value_ 这种形式的属性均是如此)，其等效于`System.setProperty(key, value)`；而当存在`SPRING_APPLICATION_JSON`这一系统变量时，自然也会在`System.getenv()`中出现。前面曾经提到过`System.getProperties()`代表的是`systemProperties`这一 _PropertySource_，而`System.getenv()`则代表的是`systemEnvironment`这一 _PropertySource_。`SpringApplicationJsonEnvironmentPostProcessor`就是用于从这两个 _PropertySource_ 中抽取出 _spring.application.json_ 或 _SPRING_APPLICATION_JSON_ 的 _JSON_ 串，进而单独向 _Environment_ 中追加一个名称为`spring.application.json`的 _PropertySource_，即`JsonPropertySource`。

**ConfigDataEnvironmentPostProcessor**

`ConfigDataEnvironmentPostProcessor`负责将`optional:classpath:/`、`optional:classpath:/config/`、`optional:file:./`、`optional:file:./config/`和`optional:file:./config/*/`这些目录下的 _application.properties_ 配置文件加载出来；如果还指定了 _spring.profiles.active_的话，同时也会将这些目录下的 _application-{profile}.properties_ 配置文件加载出来。最终，_ConfigDataEnvironmentPostProcessor_ 将会向 _Environment_ 中追加两个`OriginTrackedMapPropertySource`，这俩 _PropertySource_ 位于 _Environment_ 的尾部；其中 _application-{profile}.properties_ 所代表的 _OriginTrackedMapPropertySource_ 是排在 _application.properties_ 所代表的 _OriginTrackedMapPropertySource_ 前面的，这一点挺重要。

## 3 jasypt 核心原理解读

> `jasypt`基础组件库与`jasypt-spring-boot-starter`是不同作者写的，后者只是为 jasypt 组件开发了 Spring Boot 的起步依赖组件而已。本文所分析的其实就是这个起步依赖组件。

_application.properties_ 配置文件中关于数据源的密码是一个加密后的密文，如下：

```
spring.datasource.hikari.password=ENC(4+t9a5QG8NkNdWVS6UjIX3dj18UtYRMqU6eb3wUKjivOiDHFLZC/RTK7HuWWkUtV)
复制代码
```

当`HikariDataSource`完成属性填充操作后，该 Bean 中 _password_ 字段的值咋就变为解密后的 _qwe@1234_ 这一明文了呢？显然，Spring Boot 为 _Environment_ 提供的`EnvironmentPostProcessor`这一拓展点可以实现偷天换日！但作者没有用它，而是使用了 Spring 中的一个 _IoC 拓展点_，即`BeanFactoryPostProcessor`，这也是完全可以的，因为当执行到 _BeanFactoryPostProcessor_ 中的`postProcessBeanFactory()`逻辑时，只是完成了所有`BeanDefinition`的加载，但还没有实例化 _BeanDefinition_ 各自所对应的 Bean。

下面看一下`EnableEncryptablePropertiesBeanFactoryPostProcessor`中的内容：

```
public class EnableEncryptablePropertiesBeanFactoryPostProcessor implements BeanFactoryPostProcessor, Ordered {

    private final ConfigurableEnvironment environment;
    private final EncryptablePropertySourceConverter converter;

    public EnableEncryptablePropertiesBeanFactoryPostProcessor(ConfigurableEnvironment environment, EncryptablePropertySourceConverter converter) {
        this.environment = environment;
        this.converter = converter;
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
        MutablePropertySources propSources = environment.getPropertySources();
        converter.convertPropertySources(propSources);
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE - 100;
    }
}
复制代码
```

上述源码表明该 _BeanFactoryPostProcessor_ 借助`EncryptablePropertySourceConverter`对 _MutablePropertySources_ 做了一层转换，那么转换成啥了呢？

接着，跟进 _EncryptablePropertySourceConverter_，核心内容如下：

```
public class EncryptablePropertySourceConverter {

    public void convertPropertySources(MutablePropertySources propSources) {
        propSources.stream()
                .filter(ps -> !(ps instanceof EncryptablePropertySource))
                .map(this::makeEncryptable)
                .collect(toList())
                .forEach(ps -> propSources.replace(ps.getName(), ps));
    }

    public <T> PropertySource<T> makeEncryptable(PropertySource<T> propertySource) {
        if (propertySource instanceof EncryptablePropertySource 
                || skipPropertySourceClasses.stream().anyMatch(skipClass -> skipClass.equals(propertySource.getClass()))) {
            return propertySource;
        }
        PropertySource<T> encryptablePropertySource = convertPropertySource(propertySource);
        return encryptablePropertySource;
    }

    private <T> PropertySource<T> convertPropertySource(PropertySource<T> propertySource) {
        PropertySource<T> encryptablePropertySource;
        if (propertySource instanceof SystemEnvironmentPropertySource) {
            encryptablePropertySource = (PropertySource<T>) new EncryptableSystemEnvironmentPropertySourceWrapper((SystemEnvironmentPropertySource) propertySource, propertyResolver, propertyFilter);
        } else if (propertySource instanceof MapPropertySource) {
            encryptablePropertySource = (PropertySource<T>) new EncryptableMapPropertySourceWrapper((MapPropertySource) propertySource, propertyResolver, propertyFilter);
        } else if (propertySource instanceof EnumerablePropertySource) {
            encryptablePropertySource = new EncryptableEnumerablePropertySourceWrapper<>((EnumerablePropertySource) propertySource, propertyResolver, propertyFilter);
        } else {
            encryptablePropertySource = new EncryptablePropertySourceWrapper<>(propertySource, propertyResolver, propertyFilter);
        }
        return encryptablePropertySource;
    }
}
复制代码
```

显然，它将相关原生 _PropertySource_ 转换为了一个`EncryptablePropertySourceWrapper`，那这个肯定可以实现密文解密，必须的！

继续，跟进`EncryptablePropertySourceWrapper`，内容如下：

```
public class EncryptablePropertySourceWrapper<T> extends PropertySource<T> implements EncryptablePropertySource<T> {
    private final CachingDelegateEncryptablePropertySource<T> encryptableDelegate;

    public EncryptablePropertySourceWrapper(PropertySource<T> delegate, EncryptablePropertyResolver resolver, EncryptablePropertyFilter filter) {
        super(delegate.getName(), delegate.getSource());
        encryptableDelegate = new CachingDelegateEncryptablePropertySource<>(delegate, resolver, filter);
    }

    @Override
    public Object getProperty(String name) {
        return encryptableDelegate.getProperty(name);
    }

    @Override
    public PropertySource<T> getDelegate() {
        return encryptableDelegate;
    }
}
复制代码
```

失望！没看出啥解密逻辑，但从其 _getProperty_ 方法来看，将具体解析逻辑委派给了`CachingDelegateEncryptablePropertySource`。

没办法，只能到 _CachingDelegateEncryptablePropertySource_ 中一探究竟了：

```
public class CachingDelegateEncryptablePropertySource<T> extends PropertySource<T> implements EncryptablePropertySource<T> {
    private final PropertySource<T> delegate;
    private final EncryptablePropertyResolver resolver;
    private final EncryptablePropertyFilter filter;
    private final Map<String, Object> cache;

    public CachingDelegateEncryptablePropertySource(PropertySource<T> delegate, EncryptablePropertyResolver resolver, EncryptablePropertyFilter filter) {
        super(delegate.getName(), delegate.getSource());
        this.delegate = delegate;
        this.resolver = resolver;
        this.filter = filter;
        this.cache = new HashMap<>();
    }

    @Override
    public PropertySource<T> getDelegate() {
        return delegate;
    }

    @Override
    public Object getProperty(String name) {
        if (cache.containsKey(name)) {
            return cache.get(name);
        }
        synchronized (name.intern()) {
            if (!cache.containsKey(name)) {
                Object resolved = getProperty(resolver, filter, delegate, name);
                if (resolved != null) {
                    cache.put(name, resolved);
                }
            }
            return cache.get(name);
        }
    }
}
复制代码
```

终于，跟进到`EncryptablePropertySource`中看到了解密的最终逻辑。其中，`EncryptablePropertyDetector`负责探测相关属性是否需要对其解密，主要通过判断该属性值是否由`ENC()`包裹。

```
public interface EncryptablePropertySource<T> extends OriginLookup<String> {
    default Object getProperty(EncryptablePropertyResolver resolver, EncryptablePropertyFilter filter, PropertySource<T> source, String name) {
        Object value = source.getProperty(name);
        if (value != null && filter.shouldInclude(source, name) && value instanceof String) {
            String stringValue = String.valueOf(value);
            return resolver.resolvePropertyValue(stringValue);
        }
        return value;
    }
}

public class DefaultPropertyResolver implements EncryptablePropertyResolver {

    private final Environment environment;
    private StringEncryptor encryptor;
    private EncryptablePropertyDetector detector;

    @Override
    public String resolvePropertyValue(String value) {
        return Optional.ofNullable(value)
                .map(environment::resolvePlaceholders)
                .filter(detector::isEncrypted)
                .map(resolvedValue -> {
                    try {
                        String unwrappedProperty = detector.unwrapEncryptedValue(resolvedValue.trim());
                        String resolvedProperty = environment.resolvePlaceholders(unwrappedProperty);
                        return encryptor.decrypt(resolvedProperty);
                    } catch (EncryptionOperationNotPossibleException e) {
                        throw new DecryptionException("Unable to decrypt property: " + value + " resolved to: " + resolvedValue + ". Decryption of Properties failed,  make sure encryption/decryption " +
                                "passwords match", e);
                    }
                })
                .orElse(value);
    }
}
复制代码
```

## 4 总结

总结性的文字就不再说了，笔者现在文思泉涌，否则又能水300字。最后，希望大家记住在当前 Spring Boot 版本中，由`ApplicationServletEnvironment`扮演 _Environment_，其最终将委派`ConfigurationPropertySourcesPropertyResolver`去获取属性值。



作者：程序猿杜小头
链接：https://juejin.cn/post/7098299623759937543
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

# 参考文章

https://www.w3cschool.cn/wkspring
https://www.runoob.com/w3cnote/basic-knowledge-summary-of-spring.html
http://codepub.cn/2015/06/21/Basic-knowledge-summary-of-Spring
https://dunwu.github.io/spring-tutorial
https://mszlu.com/java/spring
http://c.biancheng.net/spring/aop-module.html