springboot 在启动类上会标注一个注解：`@SpringBootApplication`，本人将从源码解析分析这 个注解的作用。

`@SpringBootApplication` 代码如下：

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { 
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {

    /**
     * 自动装配要排除的类，功能来自于 @EnableAutoConfiguration
     */
    @AliasFor(annotation = EnableAutoConfiguration.class)
    Class<?>[] exclude() default {};

    /**
     *  自动装配要排除的类名，功能来自于 @EnableAutoConfiguration
     */
    @AliasFor(annotation = EnableAutoConfiguration.class)
    String[] excludeName() default {};

    /**
     * 配置扫描的包，功能来自于 @ComponentScan
     */
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackages")
    String[] scanBasePackages() default {};

    /**
     * 配置扫描的class，该class所在的包都会被扫描，功能来自于 @ComponentScan
     */
    @AliasFor(annotation = ComponentScan.class, attribute = "basePackageClasses")
    Class<?>[] scanBasePackageClasses() default {};

    /**
     * 是否启用 @Bean 方法代理，功能来自于 @Configuration
     */
    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;

}

```

1.  `@SpringBootApplication` 是一个组合注解，包含了 `@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan` 三个注解的功能；
2.  `@SpringBootApplication` 中也提供了一些配置属性，而这些属性来自于以上三个注解。

接下来我们来看看这三个注解的作用分别是什么。

### 1. `@SpringBootConfiguration`

进入 `@SpringBootConfiguration`，代码如下：

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Configuration
public @interface SpringBootConfiguration {

    @AliasFor(annotation = Configuration.class)
    boolean proxyBeanMethods() default true;

}

```

这个注解比较简单，上面标记了 `@Configuration`，然后是一个属性 `proxyBeanMethods()`，它来自于 `@Configuration`。因此，`@SpringBootConfiguration` 并没有做什么，仅仅只是将 `@Configuration` 使用了 `@Configuration` 的功能。

关于 `@Configuration`，它来自于 spring，能被 spring 识别为 `Component`，且 `proxyBeanMethods != false` 时，会被 spring 标记为 `Full` 配置类，在后续对其中的 `@Bean` 方法处理时，会进行 cglib 代理，关于这方面的内容，可参考 [ConfigurationClassPostProcessor（二）：处理 @Bean 注解](https://my.oschina.net/funcy/blog/4492878).

### 2. `@EnableAutoConfiguration`

`@EnableAutoConfiguration` 主要 用来开启自动装配功能，代码如下：

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
// 自动装配的包
@AutoConfigurationPackage
// 引入的自动装配类
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {

    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";

    /**
     * 可自行定义排除自动装配的类
     */
    Class<?>[] exclude() default {};

    /**
     * 可自行定义排除自动装配的类名
     */
    String[] excludeName() default {};

}

```

从代码中可以看到，

1.  该注解组合了 `@AutoConfigurationPackage` 注解的功能，该注解用来指定自动装配的包；
2.  该注解通过 `@Import` 注解引入了一个类 `AutoConfigurationImportSelector`，这个类是自动装配的关键；
3.  该注解提供了两个配置，用来排除指定的自动装配类，可以根据类来排除 (`Class` 对象)，也可以根据类名 (`包名.类名`) 排除。

接下来我们来关注 `@AutoConfigurationPackage` 及引入的 `AutoConfigurationImportSelector`。

#### 2.1 `@AutoConfigurationPackage`

`@AutoConfigurationPackage` 指定了自动装配的包，代码如下：

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Import(AutoConfigurationPackages.Registrar.class)
public @interface AutoConfigurationPackage {

}

```

这个注解的内容非常简单，仅使用 `@Import` 注解引入了 `AutoConfigurationPackages.Registrar`，我们来看下它的内容：

```
public abstract class AutoConfigurationPackages {

    private static final String BEAN = AutoConfigurationPackages.class.getName();

    static class Registrar implements ImportBeanDefinitionRegistrar, DeterminableImports {

        /**
         * 根据 ImportBeanDefinitionRegistrar 的处理，spring将调用 registerBeanDefinitions() 注册内容
         */
        @Override
        public void registerBeanDefinitions(AnnotationMetadata metadata, 
                    BeanDefinitionRegistry registry) {
            register(registry, new PackageImport(metadata).getPackageName());
        }

        @Override
        public Set<Object> determineImports(AnnotationMetadata metadata) {
            return Collections.singleton(new PackageImport(metadata));
        }

    }

    /**
     * 处理具体的注册操作
     * 1\. 如果 beanFacotry 中包含 BEAN，则将传入的包名添加到 BEAN 对应的 BeanDefinition 的构造方法参数值上；
     * 2\. 如果 beanFacotry 中不包含 BEAN，则创建 beanDefinition，设置参数值，然后将其注册到 beanFacotry。
     * 注册到beanFacotry中的bean为BasePackages
     */
    public static void register(BeanDefinitionRegistry registry, String... packageNames) {
        if (registry.containsBeanDefinition(BEAN)) {
            BeanDefinition beanDefinition = registry.getBeanDefinition(BEAN);
            // bean 是 BasePackages，构造方法是 BasePackages(String... names)，这里获取原本的构造参数的值
            ConstructorArgumentValues constructorArguments 
                    = beanDefinition.getConstructorArgumentValues();
            // 将原本的构造参数值，以及传入的 packageNames 统一添加到构造方法的第0个参数值上
            constructorArguments.addIndexedArgumentValue(0, 
                    addBasePackages(constructorArguments, packageNames));
        }
        else {
            GenericBeanDefinition beanDefinition = new GenericBeanDefinition();
            // 设置BeanClass为BasePackages.class
            beanDefinition.setBeanClass(BasePackages.class);
            // 设置构造方法的参数值
            beanDefinition.getConstructorArgumentValues().addIndexedArgumentValue(0, packageNames);
            beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
            registry.registerBeanDefinition(BEAN, beanDefinition);
        }
    }

    /**
     * packageName 的包装类
     * packageName 是传入类所在的包名，在PackageImport的构造方法中获取
     */
    private static final class PackageImport {

        private final String packageName;

        PackageImport(AnnotationMetadata metadata) {
            // 获取传入类所在包名
            this.packageName = ClassUtils.getPackageName(metadata.getClassName());
        }

        String getPackageName() {
            return this.packageName;
        }

        // 省略 equals/toString/hashCode 方法
        ...

    }

    /**
     * 注解到 beanFactory 中的类
     * 该类中有一个List结构，用来保存包扫描路径
     */
    static final class BasePackages {
        // 包扫描路径在这里保存
        private final List<String> packages;

        private boolean loggedBasePackageInfo;

        BasePackages(String... names) {
            List<String> packages = new ArrayList<>();
            for (String name : names) {
                if (StringUtils.hasText(name)) {
                    packages.add(name);
                }
            }
            this.packages = packages;
        }

        // 省略了一些代码
        ...
    }

}

```

代码有点长，但逻辑并不复杂，流程如下：

1.  `AutoConfigurationPackages.Registrar` 实现了 `ImportBeanDefinitionRegistrar`，`registerBeanDefinitions(...)` 方法向 spring 中注册了 `BasePackages`，注册逻辑在 `AutoConfigurationPackages#register` 方法中；
2.  `AutoConfigurationPackages#register` 方法的注册逻辑为，先判断是否已注册了 `BasePackages`，如果注册了，就将当前类所在的包添加到 `BasePackages` 的构造方法参数值中，否则就创建 `BeanDefinition`，设置构造方法的参数值，然后注册到 spring 中；

#### 2.2 `AutoConfigurationImportSelector`

`AutoConfigurationImportSelector` 是处理自动配置的关键，代码如下：

```
public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,

    ...

}

```

`AutoConfigurationImportSelector` 实现了 `DeferredImportSelector`，这是一个 `ImportSelector` 类，但处理的优先级最低 (在 `@ComponentScan`、`@Component`、`@Bean`、`@Configuration` 及其他 `@Import` 注解处理完之后再处理)，在 `AutoConfigurationImportSelector` 类中会处理自动配置类的加载流程，正是通过这种方式，将自动配置类引入了 spring 容器中。

关于 spring 对 `@Import` 的处理，可以参考 [ConfigurationClassPostProcessor 之处理 @Import 注解](https://my.oschina.net/funcy/blog/4678152).

关于 `AutoConfigurationImportSelector` 获取自动配置类的流程，将在后面的文章中具体分析，本文就不展开了。

### 3. `@ComponentScan`

这个注解想必大家已经很熟悉了，它指定了包扫描路径，如果不指定，就扫描所在类的包，关于这些，在 [ConfigurationClassPostProcessor 之处理 @ComponentScan 注解](https://my.oschina.net/funcy/blog/4836178)一文中已经详细分析过了，就不再分析了。

本文我们来分析这个注解属性中使用的 2 个类：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-19d14d3d8262eead434d5ca09369e1789d5.png)

#### 3.1 `TypeExcludeFilter`

这个类表示在进行包扫描时，可以排除一些类，代码如下：

```
public class TypeExcludeFilter implements TypeFilter, BeanFactoryAware {

    private BeanFactory beanFactory;

    private Collection<TypeExcludeFilter> delegates;

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }

    @Override
    public boolean match(MetadataReader metadataReader, 
            MetadataReaderFactory metadataReaderFactory) throws IOException {
        if (this.beanFactory instanceof ListableBeanFactory 
                && getClass() == TypeExcludeFilter.class) {
            // getDelegates() 获取当前容器中所有的 TypeExcludeFilter 实例
            // 可以自主继承 TypeExcludeFilter，自定义匹配规则
            for (TypeExcludeFilter delegate : getDelegates()) {
                if (delegate.match(metadataReader, metadataReaderFactory)) {
                    return true;
                }
            }
        }
        return false;
    }

    private Collection<TypeExcludeFilter> getDelegates() {
        Collection<TypeExcludeFilter> delegates = this.delegates;
        if (delegates == null) {
            delegates = ((ListableBeanFactory) this.beanFactory)
                    .getBeansOfType(TypeExcludeFilter.class).values();
            this.delegates = delegates;
        }
        return delegates;
    }

    ....

```

从代码上来看，如果要排除一些 类，我们可以自主继承 `TypeExcludeFilter` 类，然后重写 `match(...)` 方法，在其中定义匹配逻辑。

#### 3.1 `AutoConfigurationExcludeFilter`

`AutoConfigurationExcludeFilter` 用来排除自动配置类，也就是说，spring 在进行包扫描时，不会扫描自动配置类，代码如下：

```
public class AutoConfigurationExcludeFilter implements TypeFilter, BeanClassLoaderAware {

    private ClassLoader beanClassLoader;

    private volatile List<String> autoConfigurations;

    @Override
    public void setBeanClassLoader(ClassLoader beanClassLoader) {
        this.beanClassLoader = beanClassLoader;
    }

    @Override
    public boolean match(MetadataReader metadataReader, 
            MetadataReaderFactory metadataReaderFactory) throws IOException {
        // isConfiguration(...)：当前类是否被 @Configuration 标记
        // isAutoConfiguration(...)：当前类是否为自动配置类
        return isConfiguration(metadataReader) && isAutoConfiguration(metadataReader);
    }

    private boolean isConfiguration(MetadataReader metadataReader) {
        return metadataReader.getAnnotationMetadata().isAnnotated(Configuration.class.getName());
    }

    private boolean isAutoConfiguration(MetadataReader metadataReader) {
        // 获取所有的自动配置类，然后判断当前类是否存在于其中
        return getAutoConfigurations().contains(metadataReader.getClassMetadata().getClassName());
    }

    protected List<String> getAutoConfigurations() {
        if (this.autoConfigurations == null) {
            this.autoConfigurations = SpringFactoriesLoader
                    .loadFactoryNames(EnableAutoConfiguration.class, this.beanClassLoader);
        }
        return this.autoConfigurations;
    }

}

```

我们主要看 `match(...)` 方法，它的匹配的类为：

1.  被 `@Configuration` 标记；
2.  是自动配置类。

满足以上两个条件，spring 就不会对其进行扫描处理。

那什么是自动配置类呢？从 `isAutoConfiguration(...)` 可以看到，在判断是否为自动配置类上，springboot 先使用 `SpringFactoriesLoader` 加载所有配置类，然后再判断传入的类是否为其中之一。从这里可以看出，自动配置类并不进行包扫描操作。

关于 `SpringFactoriesLoader` 如何加载配置类，后面的文章会详细分析。

### 4\. 总结

本文主要分析 `@SpringBootApplication` 的功能，总结如下：

1.  `@SpringBootApplication` 是一个组合注解，包含了 `@SpringBootConfiguration`、`@EnableAutoConfiguration`、`@ComponentScan` 三个注解的功能，同时提供了一些属性配置，也是来自于以上 3 个注解；
2.  `@SpringBootConfiguration` 包含了 `Configuration` 注解的功能；
3.  `@EnableAutoConfiguration` 是开启自动装配的关键注解，其中标记了 `@AutoConfigurationPackage`，会将被 `@SpringBootApplication` 标记的类所在的包，包装成 `BasePackages`，然后注册到 spring 容器中；`@EnableAutoConfiguration` 还通过 `@Import` 注解向容器中引入了 `AutoConfigurationImportSelector`，该类会将当前项目支持的自动配置类添加到 spring 容器中；
4.  `@ComponentScan` 定义了包扫描路径，其 `excludeFilters` 值可以用来排除类的扫描，springboot 指定了 `TypeExcludeFilter`，表明我们可以继承该类来自主定义排除的类 ；同时也指定了 `AutoConfigurationExcludeFilter` ，该 `Filter` 可以用来排除自动配置类，也就是说，自动配置类不会进行包描述操作。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4870882](https://my.oschina.net/funcy/blog/4870882) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_