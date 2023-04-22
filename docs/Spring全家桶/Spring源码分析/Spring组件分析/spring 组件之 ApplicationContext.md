### 1. `ApplicationContext` 简介

我们在启动 spring 容器时，一般像这样启动：

```
ApplicationContext context = new AnnotationConfigApplicationContext(Main.class);

```

或这样：

```
ApplicationContext context = new AnnotationConfigWebApplicationContext();
context.register(MvcConfig.class);
context.refresh();

```

这里的 `AnnotationConfigApplicationContext` 与 `AnnotationConfigWebApplicationContext` 都是 `ApplicationContext`，最终进行会调用 `AbstractApplicationContext#refresh` 方法启动 spring 容器。

`ApplicationContext` 翻译为 **spring 应用上下文**，从这个类里可以获取 spring 运行期间的各种信息，如 `BeanFactory`、`Environment` 等，是 spring 中至关重要的一个类。

`ApplicationContext` 继承的接口如下：

```
public interface ApplicationContext extends EnvironmentCapable, ListableBeanFactory, 
        HierarchicalBeanFactory, MessageSource, ApplicationEventPublisher, 
        ResourcePatternResolver {
    ...
}

```

从这里可以看到，`ApplicationContext` 本身也是其他接口的子接口，这些接口的功能如下：

*   `EnvironmentCapable`：提供了环境配置功能，`applicationContext` 实现中会有一个 `Environment` 类型的成员变量，可以通过 `EnvironmentCapable#getEnvironment()` 方法取得
*   `ListableBeanFactory`：`BeanFactory` 的子接口，提供了列举 `BeanFactory` 中所有 `bean` 的方法
*   `HierarchicalBeanFactory`：`BeanFactory` 的子接口，提供了 `BeanFactory` 类似继承的能力（可以获取父 `BeanFactory`）
*   `MessageSource`：指定消息来源，可以用来实现国际化操作
*   `ApplicationEventPublisher`：事件发布器，用其提供的 `publishEvent(...)` 方法来发布事件
*   `ResourcePatternResolver`：资源解析器，提供了获取资源（`Resource`）的方法：`getResources(...)`

我们来看看 `ApplicationContext` 自身提供的方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-82de66bd51d650aa8a1fda29bdda3efd6b4.png)

可以看到，它自身的方法并不多。

### 2. `ApplicationContext` 继承结构

在 `ApplicationContext` 家族中，`ApplicationContext` 主要分为两大派系，他们及其代表如下：

*   非 web 类型的 `ApplicationContext`：处理普通 java 应用的 `ApplicationContext`，代表类为 `AnnotationConfigApplicationContext`
*   web 类型的 `ApplicationContext`：处理 web 应用的 `ApplicationContext`，代表类为 `AnnotationConfigWebApplicationContext`（这里只考虑 `servlet` 类型的 web，不考虑 `reactive` 的 web）

我们再来看看 `AnnotationConfigApplicationContext` 的继承结构：

![图片来自网络](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-6e20477e8f5948894a5f241cd41038cfa15.png)

我们再来看看 `AnnotationConfigWebApplicationContext` 的继承结构：

![图片来自网络](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-23a6c84d82370afe39245a568cbcf918209.png)

### 3\. 在 bean 中获取 `ApplicationContext`

在 spring bean 中获取 `ApplicationContext`，可以通过 `ApplicationContextAware` 接口来处理：

```
@Component
public class TestBean implements ApplicationContextAware {

    private ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) 
            throws BeansException {
        this.applicationContext = applicationContext;
    }

    // 其他操作

}

```

继承 `ApplicationContextAware` 后，`ApplicationContextAwareProcessor` 会在初始化完成后调用 `setApplicationContext(xxx)` 方法，这样我们只需要在 `TestBean` 中维护一个成员变量，对 `applicationContext` 保存即可。

### 4. `ApplicationContextAwareProcessor`

`ApplicationContextAwareProcessor` 是一个 `BeanPostProcessor`，我们主要关注 `ApplicationContextAwareProcessor#postProcessBeforeInitialization` 方法，代表如下：

```
@Override
@Nullable
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    /**
     * applicationContext 是 Environment、ResourceLoader、
     * ApplicationEventPublisher、MessageSource 等的子类，这些类的aware接口的调用，都可以
     * 通过 applicationContext 参数进行
     */
    if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
            bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
            bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
        return bean;
    }
    AccessControlContext acc = null;
    if (System.getSecurityManager() != null) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }
    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
            invokeAwareInterfaces(bean);
            return null;
        }, acc);
    }
    else {
        invokeAwareInterfaces(bean);
    }
    return bean;
}

/**
 * 调用 Aware 接口的方法
 * 除了EmbeddedValueResolverAware外，其余的传入参数都是 this.applicationContext
 */
private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof EmbeddedValueResolverAware) {
        // 注意embeddedValueResolver的获取操作如下：
        // new EmbeddedValueResolver(applicationContext.getBeanFactory());
        ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
    }
    if (bean instanceof ResourceLoaderAware) {
        ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationEventPublisherAware) {
        ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(
                this.applicationContext);
    }
    if (bean instanceof MessageSourceAware) {
        ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
    }
    // 装配 实现了ApplicationContextAware的类的 applicationContext
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
}

```

这个方法还是比较简单的，就是判断 bean 类型，然后转换、调用方法。

### 5. `ApplicationContext` 与 `BeanFactory` 的关系

最后我们再来讨论下 `ApplicationContext` 与 `BeanFactory` 两者的关系。本文一开始就说明了 `ApplicationContext` 继承了 `BeanFactory` 的接口，因为这两者是继承关系。不过，除了继承关系外，他们还是组合关系，`ApplicationContext` 持有 `BeanFactory` 的对象，直接看代码：

对于 `AnnotationConfigApplicationContext`，`beanFactory` 赋值代码如下：

```
public class GenericApplicationContext extends AbstractApplicationContext 
        implements BeanDefinitionRegistry {

    // 这就是持有的 beanFactory 对象
    private final DefaultListableBeanFactory beanFactory;

    public GenericApplicationContext() {
        this.beanFactory = new DefaultListableBeanFactory();
    }

    @Override
    public final ConfigurableListableBeanFactory getBeanFactory() {
        return this.beanFactory;
    }

    ...

}

```

对于 `AnnotationConfigWebApplicationContext`，`beanFactory` 赋值代码如下：

```
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {

    // 这就是持有的 beanFactory 对象
    @Nullable
    private DefaultListableBeanFactory beanFactory;

    @Override
    protected final void refreshBeanFactory() throws BeansException {
        // 判断当前ApplicationContext是否存在BeanFactory，如果存在的话就销毁所有 Bean，关闭 BeanFactory
        if (hasBeanFactory()) {
            destroyBeans();
            closeBeanFactory();
        }
        try {
            // 初始化DefaultListableBeanFactory，看下面的创建方法
            DefaultListableBeanFactory beanFactory = createBeanFactory();
            beanFactory.setSerializationId(getId());

            // 设置 BeanFactory 的两个配置属性：是否允许 Bean 覆盖、是否允许循环引用
            customizeBeanFactory(beanFactory);

            // 加载 Bean 到 BeanFactory 中
            loadBeanDefinitions(beanFactory);
            synchronized (this.beanFactoryMonitor) {
                this.beanFactory = beanFactory;
            }
        }
        catch (IOException ex) {
            ...
        }
    }

    // 创建 beanFactory
    protected DefaultListableBeanFactory createBeanFactory() {
        // 指定父beanFactory
        return new DefaultListableBeanFactory(getInternalParentBeanFactory());
    }

    // 获取 beanFactory
    @Override
    public final ConfigurableListableBeanFactory getBeanFactory() {
        synchronized (this.beanFactoryMonitor) {
            if (this.beanFactory == null) {
                ...
            }
            return this.beanFactory;
        }
    }

    ...

}

```

`BeanFactory` 的相关方法实现如下：

```
public abstract class AbstractApplicationContext extends DefaultResourceLoader
        implements ConfigurableApplicationContext {### 1\. 什么是 `BeanDefinition`

`BeanDefinition` 从名称上来看，就是 `bean定义`，用来定义 spring bean 的信息。

在 java 中，定义一个类的元信息（构造方法，成员变量，成员方法等），使用的是 `Class` 类，一个`.class` 文件加载到 jvm 后，都会生成一个 `Class` 对象，在对象实例化时，就根据这个 `Class` 对象的信息来生成。

在 spring 中，也有这么一个类来定义 bean 的信息，这个类就是 `BeanDefinition`，它定义了 spring bean 如何生成，如何初始化，如何销毁等，我们来看看它支持的部分方法：

```
public interface BeanDefinition extends AttributeAccessor, BeanMetadataElement {

    /**
     * 设置父 BeanDefinition
     * BeanDefinition 有个类似继承的概念，这里指定了父BeanDefinition
     * 实例化bean时，会合并父BeanDefinition
     */
    void setParentName(@Nullable String parentName);

    /**
     * 获取父Bean
     */
    @Nullable
    String getParentName();

    /**
     * 设置beanClass名称
     * 实例化时，会实例化成这个 Class 的对象
     */
    void setBeanClassName(@Nullable String beanClassName);

    /**
     * 获取beanClass名称
     */
    @Nullable
    String getBeanClassName();

    /**
     * 设置bean的作用范围，单例或原型
     */
    void setScope(@Nullable String scope);

    /**
     * 获取bean的作用范围，单例或原型
     */
    @Nullable
    String getScope();

    /**
     * 设置懒加载
     */
    void setLazyInit(boolean lazyInit);

    /**
     * 是否为懒加载
     */
    boolean isLazyInit();

    /**
     * 设置该Bean依赖的所有Bean
     * 即 @DependsOn 指定的bean
     */
    void setDependsOn(@Nullable String... dependsOn);

    /**
     * 返回该Bean的所有依赖。
     */
    @Nullable
    String[] getDependsOn();

    /**
     * 设置是否作为自动注入的候选对象。
     */
    void setAutowireCandidate(boolean autowireCandidate);

    /**
     * 是否作为自动注入的候选对象
     */
    boolean isAutowireCandidate();

    /**
     * 设置是否为主要的，当容器中存在多个同类型的bean时，可以只返回主要的
     * 就是 @Primary 的作用
     */
    void setPrimary(boolean primary);

    /**
     * 是否为主要的bean
     */
    boolean isPrimary();

    /**
     * 针对factoryBean
     * 指定factoryBean的名称
     */
    void setFactoryBeanName(@Nullable String factoryBeanName);

    /**
     * 针对factoryBean
     * 获取factoryBean的名称
     */
    @Nullable
    String getFactoryBeanName();

    /**
     * 设置工厂方法的名称
     * 例如 @Bean 标记的方法
     */
    void setFactoryMethodName(@Nullable String factoryMethodName);

    /**
     * 返回工厂方法的名称
     * 例如 @Bean 标记的方法
     */
    @Nullable
    String getFactoryMethodName();

    /**
     * 获取构造就去的参数值
     */
    ConstructorArgumentValues getConstructorArgumentValues();

    /**
     * 构造方法是否有参数
     */
    default boolean hasConstructorArgumentValues() {
        return !getConstructorArgumentValues().isEmpty();
    }

    /**
     * 获取属性值
     * 这里可以自主设置值作为构造方法、工厂方法的参数
     */
    MutablePropertyValues getPropertyValues();

    /**
     * 是否有属性值
     */
    default boolean hasPropertyValues() {
        return !getPropertyValues().isEmpty();
    }

    /**
     * 设置初始化方法名称
     */
    void setInitMethodName(@Nullable String initMethodName);

    /**
     * 获取初始化方法名称
     */
    @Nullable
    String getInitMethodName();

    /**
     * 设置销毁方法名称
     */
    void setDestroyMethodName(@Nullable String destroyMethodName);

    /**
     * 获取销毁方法名称
     */
    @Nullable
    String getDestroyMethodName();

    /**
     * 是否为单例bean
     */
    boolean isSingleton();

    /**
     * 是否为原型bean
     */
    boolean isPrototype();

    /**
     * 如果这个 Bean 是被设置为 abstract，那么不能实例化，常用于作为 父bean 用于继承
     */
    boolean isAbstract();

    ...

}

```

可以看到，`BeanDefinition` 支持的方法非常多，有很多是我们平时使用时指定的：

*   `setScope(...)`：设置 bean 的作用范围，由 `@Scope` 指定
*   `setLazyInit(...)`：设置懒加载，由 `@Lazy` 指定
*   `setDependsOn(...)`：设置 bean 依赖，由 `@DependsOn` 指定
*   `setPrimary(...)`：设置为主要 bean，由 `@Primary` 指定
*   `setFactoryMethodName(...)`：设置工厂方法名称，例如由 `@Bean` 标记的方法

以上是在注解时代可以指定的，还有些是在 `xml` 时代指定的，如：

*   `setInitMethodName(...)`：设置初始化方法 ，由 `init-method` 指定
*   `setDestroyMethodName(...)`：设置销毁方法，由 `destroy-method` 指定

### 2\. spring 提供了哪些 `BeanDefinition`

`BeanDefinition` 是一个接口，我们当然不能直接使用，接下来我们来看看 Spring 提供了哪些 `BeanDefinition`：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-49cbc9cb32badc1db52717cd19a9447eca7.png)

spring 提供的 `BeanDefinition` 基本就是上图所示的几种了，这里我们主要看这向种：

*   `RootBeanDefinition`
*   `ChildBeanDefinition`
*   `GenericBeanDefinition`
*   `ScannedGenericBeanDefinition`
*   `AnnotatedGenericBeanDefinition`

#### 2.1 `RootBeanDefinition` 和 `ChildBeanDefinition`

在前面提到了 `BeanDefinition` 继子的概念，这里就是用来处理继承的，一般来说，我们可以在 `RootBeanDefinition` 定义公共参数，然后在 `ChildBeanDefinition` 中定义各自的内容，示例如下：

```
public static void main(String[] args) {
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
// RootBeanDefinition
RootBeanDefinition root = new RootBeanDefinition();
root.setBeanClass(User.class);
root.getPropertyValues().add("name", "123");
// 这种注册方法仅做展示，实际项目中不建议使用
// 使用项目建议使用的 BeanDefinitionRegistryPostProcessor 提供的方法
context.registerBeanDefinition("root", root);

    // ChildBeanDefinition
    ChildBeanDefinition child1 = new ChildBeanDefinition("root");
    child1.getPropertyValues().add("age", "11");
    // 这种注册方法仅做展示，实际项目中不建议使用
    // 使用项目建议使用的 BeanDefinitionRegistryPostProcessor 提供的方法
    context.registerBeanDefinition("child1", child1);

    // ChildBeanDefinition
    ChildBeanDefinition child2 = new ChildBeanDefinition("root");
    child2.getPropertyValues().add("age", "12");
    // 这种注册方法仅做展示，实际项目中不建议使用
    // 使用项目建议使用的 BeanDefinitionRegistryPostProcessor 提供的方法
    context.registerBeanDefinition("child2", child2);
    // 启动容器
    context.refresh();

    User rootUser = (User) context.getBean("root");
    User child1User = (User) context.getBean("child1");
    User child2User = (User) context.getBean("child2");
    System.out.println(rootUser);
    System.out.println(child1User);
    System.out.println(child2User);
}

```

运行结果：

```
User{name='123', age=null}
User{name='123', age=11}
User{name='123', age=12}

```

可以看到，`child1` 与 `child1` 进行成功地从 `RootBeanDefinition` 继承到了属性。

#### 2.2 `GenericBeanDefinition`

这是个通用的 `BeanDefinition`，直接继承了 `AbstractBeanDefinition`，它自身提供的方法如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-5ed980070b301dc84926fac2494093d4fc6.png)

可以看到，它自身提供的方法并不多，其操作基本继承 `AbstractBeanDefinition`，一般情况下，我们要生成自己的 `BeanDefinition` 时，只需要使用这个类就可以了，这里也提供一个示例：

```
public static void main(String[] args) {
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();

    GenericBeanDefinition userBeanDefinition = new GenericBeanDefinition();
    userBeanDefinition.setBeanClass(User.class);
    userBeanDefinition.getPropertyValues().add("name", "123");
    userBeanDefinition.getPropertyValues().add("age", "11");
    // 这种注册方法仅做展示，实际项目中不建议使用
    // 使用项目建议使用的 BeanDefinitionRegistryPostProcessor 提供的方法
    context.registerBeanDefinition("user", userBeanDefinition);

    // 启动容器
    context.refresh();

    User user = (User) context.getBean("user");
    System.out.println(user);
}

```

### 2.3 `ScannedGenericBeanDefinition`

`ScannedGenericBeanDefinition` 继承了 `GenericBeanDefinition`，同时也实现了 `AnnotatedBeanDefinition` 接口，本身提供的方法并不多：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-9ed3925090a9ae5fc1d4218ed5a59c2ea19.png)

其操作基本来自 `GenericBeanDefinition`，这里就不提供示例了。

### 2.4 `AnnotatedGenericBeanDefinition`

`AnnotatedGenericBeanDefinition` 继承了 `GenericBeanDefinition`，同时也实现了 `AnnotatedBeanDefinition` 接口，本身提供的方法并不多：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b2e7f49fcdcae4f7272e2670b4fbb92766a.png)

其操作基本来自 `GenericBeanDefinition`，这里就不提供示例了。

### 3\. 操作 spring 容器中已有的 `BeanDefinition`

上面的例子都是往 spring 容器中添加 `BeanDefinition`，我们要如何操作 spring 容器中已有的 `BeanDefinition` 呢？

#### 3.1 demo 准备

这里首先准备一个 demo：

首先准备两个 `service`：

```
@Service
public class Service01 {

    private String name;

    public void hello() {
        System.out.println("hello " + name + ", from service01");
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

@Service
public class Service02 {

    private String name;

    public void hello() {
        System.out.println("hello " + name + ", from service02");
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

```

接着是主要类：

```
@ComponentScan
public class Demo02Main {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext context
                = new AnnotationConfigApplicationContext();
        context.register(Demo02Main.class);
        context.refresh();

        Service01 service01 = (Service01) context.getBean("service01");
        Service02 service02 = (Service02) context.getBean("service02");
        service01.hello();
        service02.hello();

    }
}

```

运行 ，结果如下：

```
hello null, from service01
hello null, from service02

```

这说明我们的容器已经启动成功了，`service01` 与 `service02` 也初始化成功了。

#### 3.2 不成功的尝试

这里我们反推下，`service01` 与 `service02` 已经初始化成功了，就说明容器中必然 `service01` 与 `service02` 对应的 `beanDefifnition`，我们想操作这个 `beanDefifnition`，就必须要先获取这个 `beanDefifnition`。

如何获取 spring 中已经存在的 `beanDefifnition` 呢？参考第 2 节的示例，如果你认为在 `context.refresh()` 前获取，像这样：

```
public static void main(String[] args) {
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.register(Demo02Main.class);
// 在这里获取 beanDefinition，会报错
BeanDefinition service01Bd = context.getBeanDefinition("service01");
service01Bd.getPropertyValues().addPropertyValue("name", "123");

    context.refresh();

    Service01 service01 = (Service01) context.getBean("service01");
    Service02 service02 = (Service02) context.getBean("service02");
    service01.hello();
    service02.hello();
}

```

运行，发现会报错：

```
Exception in thread "main" org.springframework.beans.factory
.NoSuchBeanDefinitionException: No bean named 'service01' available

```

聪明如你，一定会想到，在 `context.refresh()` 前获取会报错，那在之后呢？代码像这样：

```
public static void main(String[] args) {
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.register(Demo02Main.class);
context.refresh();

    // 获取 beanDefinition，修改不起作用
    BeanDefinition service01Bd = context.getBeanDefinition("service01");
    service01Bd.getPropertyValues().addPropertyValue("name", "123");

    Service01 service01 = (Service01) context.getBean("service01");
    Service02 service02 = (Service02) context.getBean("service02");
    service01.hello();
    service02.hello();

}

```

运行，结果如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-268bb546197c4eb9d1686118a12fabb0009.png)

确实没有报错，但是我们的修改也没起作用。在代码里，我们给 `service01` 的 `name` 属性指定值为 `123`，运行结果还是 `null`，没起作用的原因是 `service01` 是在 `context.refresh()` 进行初始化 的，后面再怎么对它的 `BeanDefinition` 修改，也体现不到它身上。

那么究竟要怎么做呢？

#### 3.2 `BeanDefinitionRegistryPostProcessor`：定制化 `beanDefinition`

接下来我们要放出大招了，这个大招就是：`BeanFactoryPostProcessor`，关于这个的介绍，可以它的介绍，可以参考 [spring 组件之 BeanFactoryPostProcessor](https://my.oschina.net/funcy/blog/4597545)，这里直接给下结论：

> `BeanFactoryPostProcessor` 中文名叫 `spring beanFactory 的后置处理器`，可以用来定制化 `beanFactory` 的一些行为。spring 为我们提供了两种 `BeanFactoryPostProcessor`：
>
> *   `BeanFactoryPostProcessor`：定制化 `beanFactory` 的行为
> *   `BeanDefinitionRegistryPostProcessor`：定制化 `beanDefinition` 的行为

很明显，我们应该使用 `BeanDefinitionRegistryPostProcessor`，直接实现这个接口：

```
@Component
public class MyBeanDefinitionRegistryPostProcessor
implements BeanDefinitionRegistryPostProcessor {

    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 
            throws BeansException {
        BeanDefinition service01Bd = registry.getBeanDefinition("service01");
        service01Bd.getPropertyValues().addPropertyValue("name", "123");
    }

    @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
            throws BeansException {
        // BeanDefinitionRegistryPostProcessor 是 BeanFactoryPostProcessor 的子接口
        // postProcessBeanFactory(...) 来自 BeanFactoryPostProcessor，我们这里不处理
    }
}

```

`main` 方法跟最初保持一致：

```
public static void main(String[] args) {
AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
context.register(Demo02Main.class);
context.refresh();

    Service01 service01 = (Service01) context.getBean("service01");
    Service02 service02 = (Service02) context.getBean("service02");
    service01.hello();
    service02.hello();

}

```

运行，结果如下：

```
hello 123, from service01
hello null, from service02

```

可以看到 `service01` 的 `name` 确实变成 `123` 了。

实际上，`BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry` 主要就是使用 `BeanDefinitionRegistry` 来完成 `BeanDefinition` 的操作，它支持的方法如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-5812a2cac994c5940d57c7e6ab55c23a63e.png)

重要操作如下：

*   `getBeanDefinition(...)`：获取 `BeanDefinition`，存在则返回，不存在则报错，得到 `BeanDefinition` 后，就可以对其进行各种操作了
*   `registerBeanDefinition(...)`：注册 `BeanDefinition`，可以自定义 `BeanDefinition` 对象，然后调用该方法注册到容器中，前面的例子中，我们是在 `context.refresh()` 前调用 `context.registerBeanDefinition`，这里需要说的是忘了前面的方法吧，并不是所有的情况下，都能拿到 `context.refresh()` 的 `context`（例如 `springboot` 中），因此推荐使用 `BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry` 方法注册 `BeanDefinition`
*   `removeBeanDefinition(...)`：移除 `BeanDefinition`，一般情况下应该不会用到
*   `containsBeanDefinition(...)`：判断是否包含某个 `BeanDefinition`

### 4\. 总结

本文主要介绍了 `BeanDefinition` 的作用：

1.  `BeanDefinition` 的方法
2.  几个类型的 `BeanDefinition` 的使用
3.  使用 `BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry` 操作 `BeanDefinition`，主要是注册、修改 `BeanDefinition`

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4597536](https://my.oschina.net/funcy/blog/4597536) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_

    ...
    @Override
    public <T> T getBean(Class<T> requiredType) throws BeansException {
        assertBeanFactoryActive();
        return getBeanFactory().getBean(requiredType);
    }

    @Override
    public <T> T getBean(Class<T> requiredType, Object... args) throws BeansException {
        assertBeanFactoryActive();
        return getBeanFactory().getBean(requiredType, args);
    }

    ...
}

```

这些方法在实现时，都是调用 `getBeanFactory()` 获取到 `BeanFactory` 对象，然后直接调用 `BeanFactory` 的方法，`getBeanFactory()` 调用的就是 `GenericApplicationContext` 或 `AnnotationConfigWebApplicationContext` 的 `getBeanFactory()` 方法。

关于 `ApplicationContext` 的内容就介绍到这里了。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4597456](https://my.oschina.net/funcy/blog/4597456) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_