本文是 `ConfigurationClassPostProcessor` 分析的第二篇，主要是分析 spring 对 `@Bean` 注解的处理流程。

## 3\. spring 是如何处理 `@Bean` 注解的？

承接上文，我们继续分析 spring 对 `@Bean` 注解的处理流程。

### 3.1 demo 准备

为了说明问题，我们直接上代码：

首先准备两个 Bean:

```
public class BeanObj1 {
    public BeanObj1() {
        System.out.println("调用beanObj1的构造方法");
    }

    @Override
    public String toString() {
        return "BeanObj1{}";
    }
}

public class BeanObj2 {
    public BeanObj2() {
        System.out.println("调用beanObj2的构造方法");
    }

    @Override
    public String toString() {
        return "BeanObj2{}";
    }
}

```

注意：以上两个类都没有 `Component`、`@Service` 等注解。

再准备一个配置类，通过 `@Bean` 注解的方法生成两个 bean：

```
@Component
public class BeanConfigs {

    @Bean
    public BeanObj1 beanObj1() {
        return new BeanObj1();
    }

    @Bean
    public BeanObj2 beanObj2() {
        // 这里调用下 beanObj1() 方法
        beanObj1();
        return new BeanObj2();
    }

}

```

最后是启动类：

```
@ComponentScan
public class Demo02Main {

    public static void main(String[] args) {
        ApplicationContext context 
                = new AnnotationConfigApplicationContext(Demo02Main.class);
        Object obj1 = context.getBean("beanObj1");
        Object obj2 = context.getBean("beanObj2");
        System.out.println("obj1:" + obj1);
        System.out.println("obj2:" + obj2);
        System.out.println(context.getBean("beanConfigs"));
    }
}

```

对以上 代码，做以下几点需要说明：

*   `BeanConfigs` 类使用的注解的是 `@Component`，根据[上一篇文章](https://my.oschina.net/funcy/blog/4836178)的分析，`@Component` 也属于配置类，解析步骤同 `@Configuration` 注解；
*   在 `Demo02Main` 中，传入 `AnnotationConfigApplicationContext` 类为 `Demo02Main`，其上有一个注解 `@ComponentScan`，这个注解没指定包扫描路径，根据[上一篇文章](https://my.oschina.net/funcy/blog/4836178)的分析，不指定包扫描路径，spring 会默认扫描配置类所在包；
*   我们并没有直接把 `BeanConfigs` 注册到容器中（像 `new AnnotationConfigApplicationContext(BeanConfigs.class)` 这样），从[上一篇文章](https://my.oschina.net/funcy/blog/4836178)的分析可知，spring 会先解析 `Demo02Main` 类，处理其上的 `@Component` 注解，从而扫描到 `BeanConfigs` 类，然后会解析 `BeanConfigs`，处理内部的 `@Bean` 方法，这个流程我们接下来也会通过调试的方式进行验证。

运行以上代码，结果如下：

```
调用beanObj1的构造方法
调用beanObj1的构造方法
调用beanObj2的构造方法
obj1:BeanObj1{}
obj2:BeanObj2{}
org.springframework.learn.explore.demo05.BeanConfigs@2b71e916

```

接下来，就以这个 demo 进行分析。

**注意**：本文是 `ConfigurationClassPostProcessor` 分析的第二篇，对与[第一篇](https://my.oschina.net/funcy/blog/4836178)雷同的代码，本文只一笔带过，不会再进行详细分析。

### 3.2 处理配置类：ConfigurationClassPostProcessor#processConfigBeanDefinitions

我们直接进入 `ConfigurationClassPostProcessor#processConfigBeanDefinitions` 方法，调用链如下：

```
AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(Class)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#invokeBeanFactoryPostProcessors
   |-PostProcessorRegistrationDelegate
      #invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory, List)
    |-PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
     |-ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
      |-ConfigurationClassPostProcessor#processConfigBeanDefinitions

```

此时的 `candidates` 只有一个 元素：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0d50f32d5ea1d2b79b8e17612d30693f753.png)

### 3.2 解析 `demo02Main`：ConfigurationClassParser#doProcessConfigurationClass

这一块就是解析 `@ComponentScan` 注解的过程，[第一篇](https://my.oschina.net/funcy/blog/4836178)已详细分析过，这里只给出调用栈：

```
AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(Class)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#invokeBeanFactoryPostProcessors
   |-PostProcessorRegistrationDelegate
      #invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory, List)
    |-PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
     |-ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
      |-ConfigurationClassPostProcessor#processConfigBeanDefinitions
       |-ConfigurationClassParser#parse(Set<BeanDefinitionHolder>)
        |-ConfigurationClassParser#parse(AnnotationMetadata, String)
          |-ConfigurationClassParser#processConfigurationClass
           |-ConfigurationClassParser#doProcessConfigurationClass

```

处理 `Demo02Main` 的 `@ComponentScan` 之后，可以看到 `beanConfigs` 已经扫描到了：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-7fef6007424562f63dda7f2a0b25e9c293b.png)

由于 `beanConfigs` 是配置类，因此会对其进行解析：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-fba3821df9cf2926b16bd064cf164e83471.png)

最终还是会回到 `ConfigurationClassParser#doProcessConfigurationClass`，其中的调用链如下：

```
AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(Class)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#invokeBeanFactoryPostProcessors
   |-PostProcessorRegistrationDelegate
      #invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory, List)
    |-PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
     |-ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
      |-ConfigurationClassPostProcessor#processConfigBeanDefinitions
       |-ConfigurationClassParser#parse(Set<BeanDefinitionHolder>)
        |-ConfigurationClassParser#parse(AnnotationMetadata, String)
          |-ConfigurationClassParser#processConfigurationClass
           |-ConfigurationClassParser#doProcessConfigurationClass
            |-ConfigurationClassParser#parse(String, String)
             |-ConfigurationClassParser#processConfigurationClass
              |-ConfigurationClassParser#doProcessConfigurationClass

```

此时的 `ConfigurationClassParser#doProcessConfigurationClass`，主角就不再是 `demo02Main`，而是 `beanConfigs` 了。

### 3.3 解析 `beanConfigs`：ConfigurationClassParser#doProcessConfigurationClass

接下来我们来看看 `ConfigurationClassParser#doProcessConfigurationClass` 对 `@Bean` 注解的处理，代码如下：

```
/**
 * 这个方法才是真正处理解析的方法
 */
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, 
        SourceClass sourceClass) throws IOException {
    // 1\. 如果是 @Component 注解，递归处理内部类，本文不关注
    ...

    // 2\. 处理@PropertySource注解，本文不关注
    ...

    // 3\. 处理 @ComponentScan/@ComponentScans 注解，本文不关注
    ...

    // 4\. 处理@Import注解，本文不关注
    ...

    // 5\. 处理@ImportResource注解，本文不关注
    ...

    // 6\. 处理@Bean的注解
    // 具体的解析代码
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        // 添加到 configClass 中，后面再处理
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // 7\. 返回配置类的父类，会在 processConfigurationClass(...) 方法的下一次循环时解析
    ...
    return null;
}

```

获取 `@Bean` 的方法调用的是 `retrieveBeanMethodMetadata(...)`，我们跟进去：

```
private Set<MethodMetadata> retrieveBeanMethodMetadata(SourceClass sourceClass) {
    AnnotationMetadata original = sourceClass.getMetadata();
    // 获取包含 @Bean 注解的方法
    Set<MethodMetadata> beanMethods = original.getAnnotatedMethods(Bean.class.getName());
    ...
    return beanMethods;
}

```

再跟进去，最终调用的是 `StandardAnnotationMetadata#getAnnotatedMethods`:

```
public Set<MethodMetadata> getAnnotatedMethods(String annotationName) {
    Set<MethodMetadata> annotatedMethods = null;
    if (AnnotationUtils.isCandidateClass(getIntrospectedClass(), annotationName)) {
        try {
            // 1\. 通过反射类的获取所有的方法
            Method[] methods = ReflectionUtils.getDeclaredMethods(getIntrospectedClass());
            for (Method method : methods) {
                // 2\. 判断是否有 @Bean 注解
                if (isAnnotatedMethod(method, annotationName)) {
                    if (annotatedMethods == null) {
                        annotatedMethods = new LinkedHashSet<>(4);
                    }
                    annotatedMethods.add(new StandardMethodMetadata(
                        method, this.nestedAnnotationsAsMap));
                }
            }
        }
        catch (Throwable ex) {
            throw new IllegalStateException(。。。);
        }
    }
    return annotatedMethods != null ? annotatedMethods : Collections.emptySet();
}

```

这个方法很好理解，关键就两步：

1.  通过反射获取类中所有方法；
2.  遍历得到的方法，逐一判断该方法是否有 `@Bean` 注解；

到了这里，`beanConfigs` 中的两个方法终于获取到了（保存在 `ConfigurationClass` 对象的 `beanMethods` 属性）：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-121f4d518fa57c8932e6b7fd3e7def8704b.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-a4d91c1f52703b8c911bdb9d9afaf697d92.png)

### 3.4 将被 `@Bean` 标记的方法加载到 `BeanDefinitionMap`

上面获取到的 `beanMethod` 此时还只是在 `ConfigurationClass` 对象的 `beanMethods` 属性中，并没有加载到 `beanFactory` 的 `BeanDefinitionMap` 中，本小节来探究下它们是何时放入到 `BeanDefinitionMap` 中。

还记得 `ConfigurationClassPostProcessor#processConfigBeanDefinition` 吗，其中有这么一行代码：

```
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    ...
    // 处理本次解析的类
    // 把 @Import 引入的类、配置类中带@Bean的方法、@ImportResource 引入的资源等转换成BeanDefinition
    this.reader.loadBeanDefinitions(configClasses);
    ...
}

```

这就是加载引入的 `BeanDefinition` 的地方，即把 `@Import` 引入的类、配置类中带 `@Bean` 的方法、`@ImportResource` 引入的资源等转换成 `BeanDefinition`，本文重点关注 `@Bean` 方法的处理，代码如下：

> ConfigurationClassBeanDefinitionReader

```
    public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
        TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
        // 遍历处理传入的 configurationModel
        for (ConfigurationClass configClass : configurationModel) {
            loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
        }
    }

    /**
     * 这个方法会加载各种 ConfigurationClass 引入的 BeanDefinition
     * 1\. @Import 引入的类
     * 2\. 配置类中的 @Bean 方法
     * 3\. @ImportResource 引入的资源
     * 4\. @Import 引入的 ImportBeanDefinitionRegistrar
     */
    private void loadBeanDefinitionsForConfigurationClass(
            ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

        if (trackedConditionEvaluator.shouldSkip(configClass)) {
            ...
        }

        // 处理 @Import 引入的配置类
        if (configClass.isImported()) {
            registerBeanDefinitionForImportedConfigurationClass(configClass);
        }
        // 处理 @Bean 方法
        for (BeanMethod beanMethod : configClass.getBeanMethods()) {
            loadBeanDefinitionsForBeanMethod(beanMethod);
        }
        // 处理 @ImportResource 引入的资源
        loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
        // 处理 @Import 引入的 ImportBeanDefinitionRegistrar
        loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
    }

    /**
     * 处理 @Bean 方法
     * 1\. 创建BeanDefinition，beanMethod 使用的是 ConfigurationClassBeanDefinition
     * 2\. 处理 @Bean 的各种属性，设置到 BeanDefinition 中
     * 3\. 将 BeanDefinition 注册到 beanFactory 中
     */
    private void loadBeanDefinitionsForBeanMethod(BeanMethod beanMethod) {
        ConfigurationClass configClass = beanMethod.getConfigurationClass();
        MethodMetadata metadata = beanMethod.getMetadata();
        String methodName = metadata.getMethodName();

        ...

        // 1\. beanMethod使用的是ConfigurationClassBeanDefinition
        ConfigurationClassBeanDefinition beanDef = 
                new ConfigurationClassBeanDefinition(configClass, metadata);
        beanDef.setResource(configClass.getResource());
        beanDef.setSource(this.sourceExtractor.extractSource(metadata, configClass.getResource()));

        // 2\. 处理 @Bean 的各种属性
        if (metadata.isStatic()) {
            // 静态 @Bean 方法
            if (configClass.getMetadata() instanceof StandardAnnotationMetadata) {
                beanDef.setBeanClass(
                    ((StandardAnnotationMetadata) configClass.getMetadata()).getIntrospectedClass());
            }
            else {
                beanDef.setBeanClassName(configClass.getMetadata().getClassName());
            }
            beanDef.setUniqueFactoryMethodName(methodName);
        }
        else {
            // 普通的 @Bean 方法
            beanDef.setFactoryBeanName(configClass.getBeanName());
            beanDef.setUniqueFactoryMethodName(methodName);
        }

        if (metadata instanceof StandardMethodMetadata) {
            beanDef.setResolvedFactoryMethod(
                ((StandardMethodMetadata) metadata).getIntrospectedMethod());
        }

        beanDef.setAutowireMode(AbstractBeanDefinition.AUTOWIRE_CONSTRUCTOR);
        beanDef.setAttribute(
                org.springframework.beans.factory.annotation.RequiredAnnotationBeanPostProcessor.
                SKIP_REQUIRED_CHECK_ATTRIBUTE, Boolean.TRUE);

        AnnotationConfigUtils.processCommonDefinitionAnnotations(beanDef, metadata);

        Autowire autowire = bean.getEnum("autowire");
        if (autowire.isAutowire()) {
            beanDef.setAutowireMode(autowire.value());
        }

        boolean autowireCandidate = bean.getBoolean("autowireCandidate");
        if (!autowireCandidate) {
            beanDef.setAutowireCandidate(false);
        }

        String initMethodName = bean.getString("initMethod");
        if (StringUtils.hasText(initMethodName)) {
            beanDef.setInitMethodName(initMethodName);
        }

        String destroyMethodName = bean.getString("destroyMethod");
        beanDef.setDestroyMethodName(destroyMethodName);

        ...

        // 3\. 将BeanDefinition注册到beanFactory中
        this.registry.registerBeanDefinition(beanName, beanDefToRegister);
    }

```

`ConfigurationClassBeanDefinitionReader#loadBeanDefinitions` 又调用了两个方法，最终处理是在 `ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForBeanMethod`，这部分逻辑如下：

1.  遍历传入的 `configClass` 集合，对每个 `configClass` 进行 `Definitions` 加载处理；
2.  对 `configClass` 进行 `Definitions` 加载处理时，会逐一处理 `@Import`/`@Bean`/`@ImportResource` 等注解，本小节我们仅关注 `@Bean` 的处理；
3.  处理 `@Bean` 方法时，先创建 `BeanDefinition`(`@Bean` 方法对应的 `BeanDefinition` 是 `ConfigurationClassBeanDefinition`)，然后解析 `@Bean` 的属性，设置到 `BeanDefinition` 中，最终是把 `BeanDefinition` 注册到 `beanFactory` 中.

好 了，执行完 `ConfigurationClassBeanDefinitionReader#loadBeanDefinitions` 后，`BeanDefinition` 就加载到 `beanFactory` 了，对应的 `BeanDefinition` 类型是 `ConfigurationClassBeanDefinition`.

### 3.5 `@Bean` 创建实例

实例的创建流程同普通的 `@Component` 类一致，不同的是普通的 `@Component` 类调用的是构造方法，而 `@Bean` 使用的是 `factoryMethod`，代码如下：

> AbstractAutowireCapableBeanFactory#createBeanInstance

```
/**
 * 实例的创建方式
 * 1\. 使用 instanceSupplier，Supplier是java8提供的类，可以传入一个lambda表达式
 * 2\. 使用工厂方法，如 @Bean 注解对应的方法
 * 3\. 使用的是构造方法注入，即构造方法上有 @Autowired 注解
 * 4\. 构造方法注入，可以是无参构造，也可以是有参构造
 *
 */
protected BeanWrapper createBeanInstance(String beanName, 
        RootBeanDefinition mbd, @Nullable Object[] args) {
    // 确保已经加载了此 class
    Class<?> beanClass = resolveBeanClass(mbd, beanName);
    ...

    // 是否设置了bean创建的Supplier，Supplier是java8提供的类，可以传入一个lambda表达式
    // 调用 AbstractBeanDefinition#setInstanceSupplier 指定
    Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
    if (instanceSupplier != null) {
        return obtainFromSupplier(instanceSupplier, beanName);
    }
    if (mbd.getFactoryMethodName() != null) {
        // 采用工厂方法实例化
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }
    // 是否第一次
    boolean resolved = false;
    // 是否采用构造函数注入
    boolean autowireNecessary = false;
    ...
    if (resolved) {
        if (autowireNecessary) {
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            // 无参构造函数
            return instantiateBean(beanName, mbd);
        }
    }
    // Candidate constructors for autowiring?
    // 判断是否采用有参构造函数
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(
            beanClass, beanName);
    if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
            mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
        return autowireConstructor(beanName, mbd, ctors, args);
    }
    ctors = mbd.getPreferredConstructors();
    if (ctors != null) {
        // 构造函数依赖注入
        return autowireConstructor(beanName, mbd, ctors, null);
    }
    // 调用无参构造函数
    return instantiateBean(beanName, mbd);
}

```

从代码上来看，spring 实例对象的方式有 4 种：

1.  使用 `instanceSupplier`，`Supplier` 是 `java8` 提供的类，可以传入一个 `lambda` 表达式
2.  使用工厂方法，如 `@Bean` 注解对应的方法
3.  使用的是构造方法注入，即构造方法上有 `@Autowired` 注解
4.  构造方法注入，可以是无参构造，也可以是有参构造

这里我们主要关注 `@Bean` 的实例方式，也就是工厂方法实例化方式，我们进去看下：

```
public BeanWrapper instantiateUsingFactoryMethod(String beanName, 
             RootBeanDefinition mbd, @Nullable Object[] explicitArgs) {
    BeanWrapperImpl bw = new BeanWrapperImpl();
    this.beanFactory.initBeanWrapper(bw);
    Object factoryBean;
    Class<?> factoryClass;
    boolean isStatic;
    String factoryBeanName = mbd.getFactoryBeanName();
    if (factoryBeanName != null) {
        factoryBean = this.beanFactory.getBean(factoryBeanName);
        factoryClass = factoryBean.getClass();
        isStatic = false;
    }
    ...

    Method factoryMethodToUse = null;
    ArgumentsHolder argsHolderToUse = null;
    Object[] argsToUse = null;
    // 处理 factoryMethod 的参数
    if (explicitArgs != null) {
        argsToUse = explicitArgs;
    }
    else {
        ...
    }

    if (factoryMethodToUse == null || argsToUse == null) {
        factoryClass = ClassUtils.getUserClass(factoryClass);
        List<Method> candidates = null;
        if (mbd.isFactoryMethodUnique) {
            if (factoryMethodToUse == null) {
                factoryMethodToUse = mbd.getResolvedFactoryMethod();
            }
            if (factoryMethodToUse != null) {
                candidates = Collections.singletonList(factoryMethodToUse);
            }
        }
        // 省略了好多代码
        ...
    }
    bw.setBeanInstance(instantiate(beanName, mbd, 
            factoryBean, factoryMethodToUse, argsToUse));
    return bw;
}

```

以上方法是经精简后的代码，原本方法代码比较多，大部分代码是在在处理 `argsToUse` 与 `factoryMethodToUse` 参数，细节非常多，就不展开分析了，这里我们主要关注以下几个变量：

1.  `factoryBean`： `@Bean` 方法所在类的实例，这里是 `beanConfig`;
2.  `factoryMethodToUse`: 实例化所用的方法，也就是被 `@Bean` 注解的方法，这里是 `BeanConfigs#beanObj1`;
3.  `argsToUse`：被 `@Bean` 注解的方法要用的参数，由于 `BeanConfigs#beanObj1` 没有指定参数，这里是 null;

这三个变量是用来进行实例化的变量，实例化方式大致也能想到了，实例、方法及方法参数都有了，接下来就是调用反射进行实例化了：

> ConstructorResolver#instantiate(...)

```
private Object instantiate(String beanName, RootBeanDefinition mbd,
        @Nullable Object factoryBean, Method factoryMethod, Object[] args) {
    try {

        return this.beanFactory.getInstantiationStrategy().instantiate(
                mbd, beanName, this.beanFactory, factoryBean, factoryMethod, args);
    }
    catch (Throwable ex) {
        ...
    }
}

```

> SimpleInstantiationStrategy#instantiate(...)

```
@Override
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
        @Nullable Object factoryBean, final Method factoryMethod, Object... args) {
    try {
        ...
        try {
            currentlyInvokedFactoryMethod.set(factoryMethod);
            // 这里就相当于调用 beanConfigs.beanObj1()
            Object result = factoryMethod.invoke(factoryBean, args);
            if (result == null) {
                result = new NullBean();
            }
            return result;
        }
        finally {
            ...
        }
    }
    catch (...) {
        ...
    }
}

```

最终是在 `SimpleInstantiationStrategy#instantiate(...)` 方法中进行反射实例化了！

实例化完成后，后面依赖注入、初始化等都与普通的 spring bean 一致，这里就不再分析了。

### 3.6 `@Configuration` 与 `@Bean` 组合使用

上面介绍了 `@Component` 与 `@Bean` 使用时的代码分析，即

```
@Component
public class BeanConfigs {
    @Bean
    public Xxx xxx() {
        ...
    }
}

```

实际上，大多数情况下我们使用的是 `@Configuration` 与 `@Bean` 的组合：

```
@Configuration
public class BeanConfigs {
    @Bean
    public Xxx xxx() {
        ...
    }
}

```

这与我们前面使用的 `@Component` 有何差别呢？本节我们就来分析下。

#### 1\. demo 准备

demo 准备：

```
//@Component
@Configuration
public class BeanConfigs {

    @Bean
    public BeanObj1 beanObj1() {
        return new BeanObj1();
    }

    @Bean
    public BeanObj2 beanObj2() {
        // 这里调用下 beanObj1() 方法
        beanObj1();
        return new BeanObj2();
    }

}

```

这个 demo 仅仅只是将 `@Component` 替换为 `@Configuration`，执行下，结果如下：

```
调用beanObj1的构造方法
调用beanObj2的构造方法
obj1:BeanObj1{}
obj2:BeanObj2{}
org.springframework.learn.explore.demo02.BeanConfigs$$EnhancerBySpringCGLIB$$dca1c55b@75c072cb

```

看出区别了吗，我们将之前的执行也放在这里：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-88e5047479e4f30e70d09397d2631c0c6a5.png)

经过比较，发现有两处不同：

1.  `beanObj1` 的构造方法仅调用了一次；
2.  `beanConfigs` 对应的类是 `BeanConfigs$$EnhancerBySpringCGLIB$$dca1c55b@75c072cb`，说明它是一个代理对象，使用了 cglib 代理。

实际上，以上两个不同点都可以归结为一个原因：spring 对 `beanConfigs` 进行了代理，调用 `BeanConfigs#beanObj1` 实际调用的是代理方法，即 **spring 会对被 `@Configuration` 标记的类进行 cglib 代理**！

那么代理是怎么创建及运行的呢？我们继续探究。

#### 2\. 代理类的创建：`ConfigurationClassPostProcessor#enhanceConfigurationClasses`

在[上一篇文章](https://my.oschina.net/funcy/blog/4836178)的开篇，我们就提到过 `ConfigurationClassPostProcessor` 在执行时会调用到两个方法：`processConfigBeanDefinitions(...)` 与 `enhanceConfigurationClasses(...)`，`processConfigBeanDefinitions(...)` 引起了 spring 对 `@Import`、`@Configuration` 等注解的解析，前面已经分析过了；而 `enhanceConfigurationClasses(...)` 就是被 `@Configuration` 标记的类代理产生的关键所在！

`enhanceConfigurationClasses(...)` 方法代码如下：

> ConfigurationClassPostProcessor#enhanceConfigurationClasses

```
public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
    Map<String, AbstractBeanDefinition> configBeanDefs = new LinkedHashMap<>();
    for (String beanName : beanFactory.getBeanDefinitionNames()) {
        BeanDefinition beanDef = beanFactory.getBeanDefinition(beanName);
        Object configClassAttr = beanDef.getAttribute(
            ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE);
        ...
        // 1\. 判断是否为一个全配置类
        if (ConfigurationClassUtils.CONFIGURATION_CLASS_FULL.equals(configClassAttr)) {
            ...
            configBeanDefs.put(beanName, (AbstractBeanDefinition) beanDef);
        }
    }
    // 全配置类：处理代理
    ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
    for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
        AbstractBeanDefinition beanDef = entry.getValue();
        beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
        // 处理 BeanClass
        Class<?> configClass = beanDef.getBeanClass();
        // 2\. 生成 enhancedClass
        Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
        if (configClass != enhancedClass) {
            // 3\. 设置 BeanClass，值为enhancedClass
            beanDef.setBeanClass(enhancedClass);
        }
    }
}

```

这个方法比较简单，步骤如下：

1.  判断配置类是否为全配置类，在在[上一篇文章](https://my.oschina.net/funcy/blog/4836178)文章中，我们提到 spring 会把带有 `@Configuration` 注解且 `proxyBeanMethods != false` 的类标记为 `Full` 配置类，这里正是根据前面的标记来判断否为全配置类，很明显，此时的 `beanConfigs` 就是一个全配置类；
2.  对全配置类，会根据其 `configClass` 生成对应的 `enhancedClass`；
3.  将生成的 `enhancedClass` 设置到 `beanDefinition` 的 `beanClass` 中。

执行完此方法后，`beanConfigs` 对应的 `beanDefinition` 的 `beanClass` 就是代理类了：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-f617fc93f72b13b8f785ccf5a12624dec0b.png)

后面创建的 `beanConfigs` 就是这个代理类的实例了。

#### 3\. 执行代理对象的方法

生成代理对象后，代理方法是如何执行的呢？即 spring 是如何执行 `beanConfigs.beanObj1()` 的呢？说起这个，就需要谈到 cglib 代理对象的方法执行了。我们直接来看代理的生成，进入 `enhancer.enhance(configClass, this.beanClassLoader)`：

```
public Class<?> enhance(Class<?> configClass, @Nullable ClassLoader classLoader) {
    if (EnhancedConfiguration.class.isAssignableFrom(configClass)) {
        return configClass;
    }
    // newEnhancer(...) 方法才是关键
    Class<?> enhancedClass = createClass(newEnhancer(configClass, classLoader));
    return enhancedClass;
}

```

继续进入：

```
private static final Callback[] CALLBACKS = new Callback[] {
        // 这个类用来保证 @Bean 方法的单例
        new BeanMethodInterceptor(),
        new BeanFactoryAwareMethodInterceptor(),
        NoOp.INSTANCE
};

// 生成 CallbackFilter，传入的对象为Callback
private static final ConditionalCallbackFilter CALLBACK_FILTER 
        = new ConditionalCallbackFilter(CALLBACKS);

// 生成cglib增强
private Enhancer newEnhancer(Class<?> configSuperClass, @Nullable ClassLoader classLoader) {
    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(configSuperClass);
    enhancer.setInterfaces(new Class<?>[] {EnhancedConfiguration.class});
    enhancer.setUseFactory(false);
    enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
    enhancer.setStrategy(new BeanFactoryAwareGeneratorStrategy(classLoader));
    // 代理部分在callbackFilter
    enhancer.setCallbackFilter(CALLBACK_FILTER);
    enhancer.setCallbackTypes(CALLBACK_FILTER.getCallbackTypes());
    return enhancer;
}

```

关于 cglib 代理的内容，在 [spring aop 之 cglib 代理](https://my.oschina.net/funcy/blog/4696655)一文中已详细分析过了，这不再分析，我们直接说结论：cglib 执行代理方法时，执行的是 `Enhancer` 中 `callbackFilter` 属性的 `MethodInterceptor#intercept` 方法，即 `CALLBACKS` 数组中的 `BeanMethodInterceptor`，下面我们就来看下它的内容：

> ConfigurationClassEnhancer.BeanMethodInterceptor

```
private static class BeanMethodInterceptor implements MethodInterceptor, ConditionalCallback {
    @Override
    @Nullable
    public Object intercept(Object enhancedConfigInstance, Method beanMethod, Object[] beanMethodArgs,
                MethodProxy cglibMethodProxy) throws Throwable {
        ConfigurableBeanFactory beanFactory = getBeanFactory(enhancedConfigInstance);
        String beanName = BeanAnnotationHelper.determineBeanNameFor(beanMethod);
        ...
        // 如果是调用当前的 factoryMethod 方法，直接调用父类的方法
        if (isCurrentlyInvokedFactoryMethod(beanMethod)) {
            // 调用父类的方法，也就是目标方法
            return cglibMethodProxy.invokeSuper(enhancedConfigInstance, beanMethodArgs);
        }
        // 否则直接获取 beanFactory中已有的对象
        return resolveBeanReference(beanMethod, beanMethodArgs, beanFactory, beanName);
    }

    private Object resolveBeanReference(Method beanMethod, Object[] beanMethodArgs,
            ConfigurableBeanFactory beanFactory, String beanName) {
        try {
            ...
            // 调用的的是 beanFactory.getBean(...) 方法，这个方法我们已经非常熟悉了
            Object beanInstance = (useArgs ? beanFactory.getBean(beanName, beanMethodArgs) :
                    beanFactory.getBean(beanName));
            ...
            return beanInstance;
        }
        finally {
            ...
        }
    }

    /**
     * 判断能否执行当前 MethodInterceptor
     */
    @Override
    public boolean isMatch(Method candidateMethod) {
        return (candidateMethod.getDeclaringClass() != Object.class &&
                !BeanFactoryAwareMethodInterceptor.isSetBeanFactory(candidateMethod) &&
                BeanAnnotationHelper.isBeanAnnotated(candidateMethod));
    }
}

```

`BeanMethodInterceptor` 实现了 `MethodInterceptor` 与 `ConditionalCallback`，`ConditionalCallback#isMatch` 用来判断当前 `MethodInterceptor` 能否执行，`MethodInterceptor#intercept` 就是执行的方法内容，执行逻辑为：

1.  如果是直接调用当前的 `factoryMethod` 方法，直接调用父类的方法，也就是 `beanConfigs.beanObj1()`，这个过程会实例化 `beanObj1` 时被调用；
2.  如果不是直接调用当前的 `factoryMethod` 方法（比如在别的方法中调用），则调用 `beanFactory.getBean(...)` 获取 bean，这个过程会在实例化 `beanObj1` 时会 被调用。

以上就是为什么 `beanObj1` 的构造方法只调用了一次，以及为什么 `beanConfigs` 是代理类的原因所在了。

最后再提一句，`@Configuration` 提供了 `proxyBeanMethods()` 方法来让我们选择是否开启配置类的代理，默认值是 true，如果像这样设置：

```
@Configuration(proxyBeanMethods=false)
public class BeanConfigs {
    ...
}

```

`BeanConfigs` 就不会进行代理了，运行结果同 `@Component` 注解一样，这里就不展示了。

#### 4\. 几个小问题

##### 1\. 内部方法调用也能被代理吗？

在示例中，我们是这样调用的：

```
    @Bean
    public BeanObj2 beanObj2() {
        // 这里调用下 beanObj1() 方法
        beanObj1();
        return new BeanObj2();
    }

```

即在 `beanObj2()` 中调用了 `beanObj1()`，这明显是内部方法调用，`beanObj1()` 也能被代理吗？

**回答**：cglib 代理的调用方法有两种：

```
@Override
public Object intercept(Object proxyObj, Method method, Object[] objects, 
            MethodProxy proxy) throws Throwable {
    // 方案1： 使用目标对象，直接调用目标对象的方法
    // return proxy.invoke(target, objects);
    // 方案2： 使用代理对象，调用其父类的方法
    return proxy.invokeSuper(proxyObj, objects);
}

```

`beanObj2()` 的调用使用是`方案2`，也就是使用代理对象调用 `beanObj2()`，`beanObj2()` 的 `this` 为代理对象：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-7ef891450ace815d22a5bf5c8e78e65c2db.png)

因此在 `beanObj2()` 中直接调用 `beanObj1()`，就相当于使用代理对象调用 `beanObj1()`，当然能被代理了。

在我们印象中，方法的内部调用不能被代理，那是因为 spring 在处理 Aop 时，使用的是方案 1 的调用方式，此时的 `this` 为原始对象，当然不能被代理了。

##### 2\. 私有属性如何注入？

比如，我们现在有一个 `BeanObj3`:

```
@Component
public class BeanObj3 {

    public BeanObj3() {
        System.out.println("调用beanObj3的构造方法");
    }

    @Override
    public String toString() {
        return "BeanObj3{}";
    }
}

```

然后在 `BeanConfigs` 中注入：

```
@Configuration
public class BeanConfigs {

    @Autowired
    private BeanObj3 beanObj3;

    @Bean
    public BeanObj1 beanObj1() {
        return new BeanObj1();
    }

    @Bean
    public BeanObj2 beanObj2() {
        // 这里调用下 beanObj1() 方法
        beanObj1();
        System.out.println("beanObj3：" + this.beanObj3);
        return new BeanObj2();
    }

}

```

在 `BeanConfigs` 中自动注入了 `beanObj3` 属性，然后在 `beanObj2()` 中又打印了 `beanObj3` 属性。运行，结果如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-e21bf76da41fcf12e674fa155de4b636253.png)

可以看到，注入的 `beanObj3` 也能获取到了。这里就有个问题了：`beanObj3` 是属于目标对象的，而 `this` 是代理对象，难不成代理对象能拿到目标对象的私有属性？

首先，添加到 `beanFactory` 的 `beanDefinitionMap` 中的类是 `BeanConfigs$$EnhancerBySpringCGLIB$$Xxx` 类（代理类），而不是 `BeanConfigs`，spring 在进行属性注入时，会查找当前类及其父类的所有等注入属性进行注入，因此，虽然添加到 spring 容器中的 是 `BeanConfigs$$EnhancerBySpringCGLIB$$Xxx` 类，但 `BeanConfigs` 中的 `beanObj3` 一样会被注入，至于原因嘛，由于 cglib 的代理关系，`BeanConfigs` 是 `BeanConfigs$$EnhancerBySpringCGLIB$$Xxx` 的父类。

那 `BeanConfigs$$EnhancerBySpringCGLIB$$Xxx` 会继承 `beanObj3` 属性吗？这里直接看运行结果吧：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-27ebdebfd8c958f1de362ca1b30f2d0fc7c.png)

这是最终得到的 `beanConfigs` 对象，可以看到，它里面就有一个 `beanObj3` 属性，并且还有值。

### 3.7 总结

本文主要分析了 `ConfigurationClassPostProcessor` 处理 `@Bean` 注解的过程，总结如下：

1.  解析配置类，通过反射获取配置类里所有被 `@Bean` 标记的方法；
2.  遍历这些方法，将其封装成一个个 `BeanDefinition` 注册到 `beanFactory` 中，对应 `BeanDefinition` 具体类型为 `ConfigurationClassBeanDefinition`；
3.  如果配置类是全配置类，会对配置类进行 cglib 代理；
4.  实例化时，使用反射调用对应的方法生成实例（得到实例后，spring 会再对其进行依赖注入、初始化等）；
5.  在别的 `@Bean` 方法中调用当前 `@Bean` 方法时，如果当前 `@Bean` 方法所在的类是全配置类，则会去 `beanFactory` 中查找对应的 `bean`(查找的过程是，找到则返回，找不到则创建再返回，**返回的 bean 有完整的 spring bean 的生命周期**)，这个操作是由 cglib 代理完成；如果当前 `@Bean` 方法所在的类不是全配置类，则会按照普通的方法调用，生成 bean 的实例返回（**返回的 bean 没有完整的 spring bean 的生命周期**）。

本文的分析就到这里了，接下来我们继续分析 `ConfigurationClassPostProcessor` 处理其他注解的流程。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4492878](https://my.oschina.net/funcy/blog/4492878) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_