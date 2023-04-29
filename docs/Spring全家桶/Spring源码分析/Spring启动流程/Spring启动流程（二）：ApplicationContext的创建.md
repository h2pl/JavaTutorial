![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-07194ddebd25cb2b71ee5e422bf84e8a397.png)

在前面一篇文章中，我们分析了 spring 的整体启动流程，从这篇开始 ，我们将对启动过程中的一些关键代码进行分析。

分析依旧是基于的 `demo01`，我们直接进行 `ApplicationContext context = new AnnotationConfigApplicationContext("org.springframework.learn.demo01");` 的执行中：

> AnnotationConfigApplicationContext

```
public AnnotationConfigApplicationContext(String... basePackages) {
    this();
    scan(basePackages);
    refresh();
}

```

这个方法只有三行，每行代码的作用在 [spring 启动流程概览](https://my.oschina.net/funcy/blog/4597493 "spring启动流程概览")已做过说明，这里我们将对这些方法展开，详细分析里面的内容。

### 1. `beanFacotry` 的创建

我们直接进行 `this()` 方法，内容如下：

> AnnotationConfigApplicationContext

```
public AnnotationConfigApplicationContext() {
    // AnnotatedBeanDefinitionReader 解析 @Configuration
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

```

表面上看，代码只有两行，创建了两个对象。但熟悉 java 基础语法的都知道，子类在调用构造方法时，会先调用父类的构造方法再执行子类构造方法的代码，因些我们还需要看看该类的父类构造方法里做了什么：

> GenericApplicationContext

```
public GenericApplicationContext() {
    this.beanFactory = new DefaultListableBeanFactory();
}

```

可以看到，父类的构造方法里仅做了一件事：创建 `beanFactory`。自此，我们可以知道：**`AnnotationConfigApplicationContext` 使用的 `BeanFacotry` 为 `DefaultListableBeanFactory`**。

我们再回到 `AnnotationConfigApplicationContext` 的构造方法：

> AnnotationConfigApplicationContext

```
public AnnotationConfigApplicationContext() {
    // AnnotatedBeanDefinitionReader 解析 @Configuration
    this.reader = new AnnotatedBeanDefinitionReader(this);
    this.scanner = new ClassPathBeanDefinitionScanner(this);
}

```

这个方法虽然只有两行，作用却不小。我们深入 `new AnnotatedBeanDefinitionReader(this);` 看下，由于其中的方法调用并不重要，这里仅提供调用链：

```
AnnotationConfigApplicationContext#AnnotationConfigApplicationContext()
 |-AnnotatedBeanDefinitionReader#AnnotatedBeanDefinitionReader(BeanDefinitionRegistry)
  |-AnnotatedBeanDefinitionReader#AnnotatedBeanDefinitionReader(BeanDefinitionRegistry, Environment)
   |-AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry)
    |-AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry, Object)

```

可以看到，最终是调用了 `AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry, Object)`，我们查看下该方法，为了直观查看，这里省略了不必要的代码，我们仅关注主要的流程即可：

```
public static Set<BeanDefinitionHolder> registerAnnotationConfigProcessors(
        BeanDefinitionRegistry registry, @Nullable Object source) {
    // 获得beanFactory
    DefaultListableBeanFactory beanFactory = unwrapDefaultListableBeanFactory(registry);

    // -------- 往beanFactory中添加处理类
    if (beanFactory != null) {
        if (!(beanFactory.getDependencyComparator() instanceof AnnotationAwareOrderComparator)) {
        beanFactory.setDependencyComparator(AnnotationAwareOrderComparator.INSTANCE);
        }
        if (!(beanFactory.getAutowireCandidateResolver() 
                instanceof ContextAnnotationAutowireCandidateResolver)) {
            beanFactory.setAutowireCandidateResolver(new ContextAnnotationAutowireCandidateResolver());
        }
    }

    Set<BeanDefinitionHolder> beanDefs = new LinkedHashSet<>(8);

    // ------------  往beanFactory中添加beanDefinition
    if (!registry.containsBeanDefinition(CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(ConfigurationClassPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, CONFIGURATION_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(AutowiredAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, AUTOWIRED_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JSR-250 support, and if present add the CommonAnnotationBeanPostProcessor.
    if (jsr250Present && !registry.containsBeanDefinition(COMMON_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(CommonAnnotationBeanPostProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, COMMON_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    // Check for JPA support, and if present add the PersistenceAnnotationBeanPostProcessor.
    if (jpaPresent && !registry.containsBeanDefinition(PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition();
        try {
            def.setBeanClass(ClassUtils.forName(PERSISTENCE_ANNOTATION_PROCESSOR_CLASS_NAME,
                AnnotationConfigUtils.class.getClassLoader()));
        }
        catch (ClassNotFoundException ex) {
            throw new IllegalStateException(...);
        }
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, PERSISTENCE_ANNOTATION_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_PROCESSOR_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(EventListenerMethodProcessor.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_PROCESSOR_BEAN_NAME));
    }

    if (!registry.containsBeanDefinition(EVENT_LISTENER_FACTORY_BEAN_NAME)) {
        RootBeanDefinition def = new RootBeanDefinition(DefaultEventListenerFactory.class);
        def.setSource(source);
        beanDefs.add(registerPostProcessor(registry, def, EVENT_LISTENER_FACTORY_BEAN_NAME));
    }
    return beanDefs;
}

```

这个方法代码虽然有点长，但功能却相当直白，就是向 `beanFactory` 添加 `annotation` 相关的处理器。实际上，在 [spring 启动流程概览](https://my.oschina.net/funcy/blog/4597493 "spring启动流程概览")提到的 beanDefinitionMap 中 4 个默认的 beanDefinition 就是在这里添加的：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-a490052bdc1379ef24a7754c65584214c1c.png)

接着我们再关注 `this.scanner = new ClassPathBeanDefinitionScanner(this)`，这是初始化 `scanner` 对象，类型为 `ClassPathBeanDefinitionScanner`，该类从名字上可以看出，这是与 `classPath` 相关的，`beanDefinition` 扫描器，通俗地说，就是**扫描 classPath 路径，将 java class 文件组装成 `beanDefinition` 对象**。

### 2\. 总结

`AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)` 中 `this()` 的执行就分析到这里了，这一 行代码主要做了这几件事：

1.  创建了类型为 `DefaultListableBeanFactory` 的 `beanFactory`
2.  创建了类型为 `AnnotatedBeanDefinitionReader` 的 reader，在其创建的过程中，会往 `beanFactory` 添加 annotation 相关的处理器
3.  创建了类型为 `ClassPathBeanDefinitionScanner` 的 `scanner`

本文流程图示如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b7a7a01b4d38769419a0e25e8f60037cbb5.png)

本文就先到这里了，接下来的文章我们继续分析后续的代码。