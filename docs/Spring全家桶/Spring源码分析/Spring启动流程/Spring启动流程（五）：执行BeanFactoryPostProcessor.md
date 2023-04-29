![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-c8c446fca94a382109bdc7b6babd6db4c0d.png)

接上文，我们继续分析。

### 4\. 扩展点：`postProcessBeanFactory(beanFactory)`

这个类是 spring 提供的扩展点，本身并无任何功能，留待子类实现，`AbstractApplicationContext` 的 `postProcessBeanFactory` 方法代码如下：

```
    protected void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    }

```

当前我们使用的 `ApplicationContext` 是 `AnnotationConfigApplicationContext`，并没有实现这个方法。

### 5\. 执行 `BeanFactoryPostProcessors`: `invokeBeanFactoryPostProcessors(beanFactory)`

> `BeanFactoryPostProcessor` 称为 beanFactory 的后置处理，用来修改 beanFactory 的一些行为。关于 `BeanFactoryPostProcessor` 的详细分析，可以参考 [spring 组件之 BeanFactoryPostProcessors](https://my.oschina.net/funcy/blog/4597545)。

关于 `BeanFactoryPostProcessor`，这里提几点：

*   `BeanFactoryPostProcessor` 分为两种：`BeanFactoryPostProcessor` 与 `BeanDefinitionRegistryPostProcessor`
*   `BeanDefinitionRegistryPostProcessor` 是 `BeanFactoryPostProcessor` 的子类
*   先执行 `BeanDefinitionRegistryPostProcessor` 的方法，再执行 `BeanFactoryPostProcessor` 的方法

了解这些后，我们跟进代码，这里对不重要代码依旧只给出调用链，如下：

```
|-AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#invokeBeanFactoryPostProcessors
   |-PostProcessorRegistrationDelegate
     #invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory, List<BeanFactoryPostProcessor>)

```

我们直接看 `invokeBeanFactoryPostProcessors` 方法：

```
public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, 
            List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

    // 所有存在的BeanDefinitionRegistryPostProcessor的名字
    Set<String> processedBeans = new HashSet<>();

    //beanFactory是DefaultListableBeanFactory，是BeanDefinitionRegistry的实现类，所以肯定满足if
    if (beanFactory instanceof BeanDefinitionRegistry) {
        BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;

        //regularPostProcessors 用来存放BeanFactoryPostProcessor
        List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();

        //registryProcessors 用来存放BeanDefinitionRegistryPostProcessor
        //BeanDefinitionRegistryPostProcessor 扩展了 BeanFactoryPostProcessor
        List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

        // 循环传进来的beanFactoryPostProcessors，正常情况下，beanFactoryPostProcessors肯定没有数据
        // 因为beanFactoryPostProcessors是获得手动添加的，而不是spring扫描的
        // 只有手动调用annotationConfigApplicationContext.addBeanFactoryPostProcessor(XXX)才会有数据
        for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
            // 判断postProcessor是不是BeanDefinitionRegistryPostProcessor，
            // 因为BeanDefinitionRegistryPostProcessor 扩展了BeanFactoryPostProcessor，
            // 所以这里先要判断是不是BeanDefinitionRegistryPostProcessor, 是的话，直接执行
            // postProcessBeanDefinitionRegistry方法，然后把对象装到registryProcessors里面去
            if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                BeanDefinitionRegistryPostProcessor registryProcessor =
                        (BeanDefinitionRegistryPostProcessor) postProcessor;
                registryProcessor.postProcessBeanDefinitionRegistry(registry);
                registryProcessors.add(registryProcessor);
            }
            else {
                //不是的话，就装到regularPostProcessors
                regularPostProcessors.add(postProcessor);
            }
        }

        // 一个临时变量，用来装载BeanDefinitionRegistryPostProcessor
        // BeanDefinitionRegistry继承了PostProcessorBeanFactoryPostProcessor
        List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

        // 获得实现BeanDefinitionRegistryPostProcessor接口的类的BeanName
        // 这里包含了spring内部提供的BeanDefinitionRegistryPostProcessor
        // 以及开发者自己实现的BeanDefinitionRegistryPostProcessor
        String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
        for (String ppName : postProcessorNames) {
            if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
               //获得ConfigurationClassPostProcessor类，并且放到currentRegistryProcessors
               //ConfigurationClassPostProcessor是很重要的一个类，它实现了
               //BeanDefinitionRegistryPostProcessor接口，
               //BeanDefinitionRegistryPostProcessor接口又实现了BeanFactoryPostProcessor接口
               //ConfigurationClassPostProcessor是极其重要的类, 里面执行了
               //扫描 @Bean，@Import，@ImportResource 等各种操作
               currentRegistryProcessors.add(beanFactory.getBean(
                   ppName, BeanDefinitionRegistryPostProcessor.class));
               processedBeans.add(ppName);
            }
        }

       //处理排序
       sortPostProcessors(currentRegistryProcessors, beanFactory);

       //合并Processors，为什么要合并，因为registryProcessors是
       //装载BeanDefinitionRegistryPostProcessor的
       //一开始的时候，spring只会执行BeanDefinitionRegistryPostProcessor独有的方法
       //而不会执行BeanDefinitionRegistryPostProcessor父类的方法，即BeanFactoryProcessor的方法
       //所以这里需要把处理器放入一个集合中，后续统一执行父类的方法
       registryProcessors.addAll(currentRegistryProcessors);

       //可以理解为执行ConfigurationClassPostProcessor的postProcessBeanDefinitionRegistry方法
       invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
       //因为currentRegistryProcessors是一个临时变量，所以需要清除
       currentRegistryProcessors.clear();

       // 再次根据BeanDefinitionRegistryPostProcessor获得BeanName，
       // 看这个BeanName是否已经被执行过了，有没有实现Ordered接口
       // 如果没有被执行过，也实现了Ordered接口的话，把对象推送到currentRegistryProcessors，
       // 名称推送到processedBeans
       // 如果没有实现Ordered接口的话，这里不把数据加到currentRegistryProcessors，
       // processedBeans中，后续再做处理
       // 这里才可以获得我们定义的实现了BeanDefinitionRegistryPostProcessor的Bean
       postProcessorNames = beanFactory.getBeanNamesForType(
              BeanDefinitionRegistryPostProcessor.class, true, false);
       for (String ppName : postProcessorNames) {
           if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
              currentRegistryProcessors.add(beanFactory.getBean(
                  ppName, BeanDefinitionRegistryPostProcessor.class));
              processedBeans.add(ppName);
           }
       }
       //处理排序
       sortPostProcessors(currentRegistryProcessors, beanFactory);
       //合并Processors
       registryProcessors.addAll(currentRegistryProcessors);
       //执行我们自定义的BeanDefinitionRegistryPostProcessor
       invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
       //清空临时变量
       currentRegistryProcessors.clear();
       // 上面的代码是执行了实现了Ordered接口的BeanDefinitionRegistryPostProcessor，
       // 下面的代码就是执行没有实现Ordered接口的BeanDefinitionRegistryPostProcessor
       boolean reiterate = true;
       while (reiterate) {
           reiterate = false;
           postProcessorNames = beanFactory.getBeanNamesForType(
                    BeanDefinitionRegistryPostProcessor.class, true, false);
           for (String ppName : postProcessorNames) {
                if (!processedBeans.contains(ppName)) {
                    currentRegistryProcessors.add(beanFactory.getBean(
                            ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                    reiterate = true;
                }
           }
           sortPostProcessors(currentRegistryProcessors, beanFactory);
           registryProcessors.addAll(currentRegistryProcessors);
           invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
           currentRegistryProcessors.clear();
       }

       // registryProcessors集合装载BeanDefinitionRegistryPostProcessor
       // 上面的代码是执行子类独有的方法，这里需要再把父类的方法也执行一次
       invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
       invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
    }

    else {
        //regularPostProcessors装载BeanFactoryPostProcessor，执行BeanFactoryPostProcessor的方法
        //但是regularPostProcessors一般情况下，是不会有数据的，
        //只有在外面手动添加BeanFactoryPostProcessor，才会有数据
        invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
    }

    String[] postProcessorNames =
            beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

    List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();
    //循环BeanName数组
    for (String ppName : postProcessorNames) {
        //如果这个Bean被执行过了，跳过
        if (processedBeans.contains(ppName)) {

        }
        //如果实现了PriorityOrdered接口，加入到priorityOrderedPostProcessors
        else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            priorityOrderedPostProcessors.add(beanFactory
                .getBean(ppName, BeanFactoryPostProcessor.class));
        }
        //如果实现了Ordered接口，加入到orderedPostProcessorNames
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        //如果既没有实现PriorityOrdered，也没有实现Ordered。加入到nonOrderedPostProcessorNames
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // 排序处理priorityOrderedPostProcessors，即实现了PriorityOrdered接口的BeanFactoryPostProcessor
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // 执行priorityOrderedPostProcessors
    invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

    // 执行实现了Ordered接口的BeanFactoryPostProcessor
    List<BeanFactoryPostProcessor> orderedPostProcessors 
            = new ArrayList<>(orderedPostProcessorNames.size());
    for (String postProcessorName : orderedPostProcessorNames) {
        orderedPostProcessors.add(beanFactory.getBean(
            postProcessorName, BeanFactoryPostProcessor.class));
    }
    sortPostProcessors(orderedPostProcessors, beanFactory);
    invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

    // 执行既没有实现PriorityOrdered接口，也没有实现Ordered接口的BeanFactoryPostProcessor
    List<BeanFactoryPostProcessor> nonOrderedPostProcessors 
            = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String postProcessorName : nonOrderedPostProcessorNames) {
        nonOrderedPostProcessors.add(beanFactory.getBean(
                postProcessorName, BeanFactoryPostProcessor.class));
    }
    invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

    beanFactory.clearMetadataCache();
}

```

这个方法非常非常长，但理解起来并不难，执行起来遵循这几条规则：

*   先执行 `BeanDefinitionRegistryPostProcessor`，再执行 `BeanFactoryPostProcessor`
*   执行 `BeanDefinitionRegistryPostProcessor` 的顺序如下：
    1.  执行参数传入的 `BeanDefinitionRegistryPostProcessor`
    2.  执行 spring 内部提供的，这里会执行一个非常重要的 `BeanDefinitionRegistryPostProcessor`——`ConfigurationClassPostProcessor`，它会处理项目中的 `@ComponentScan`、`@Component`、`@Import`、`@Bean` 等注解，加载用户自定义的 `BeanDefinitionRegistryPostProcessor`、`BeanFactoryPostProcessor`
    3.  执行剩下的 `BeanDefinitionRegistryPostProcessor`，也就是在上一步中加载到的 `BeanDefinitionRegistryPostProcessor`
*   执行 `BeanFactoryPostProcessor` 的顺序如下：
    1.  执行实现了 `PriorityOrdered` 接口的 `BeanFactoryPostProcessor`
    2.  执行实现了 `Ordered` 接口的 `BeanFactoryPostProcessor`
    3.  执行剩下的 `BeanFactoryPostProcessor`
*   `BeanDefinitionRegistryPostProcessor` 是 `BeanFactoryPostProcessor` 的子类，同样要执行 `BeanFactoryPostProcessor` 的方法

以上说的执行 `BeanDefinitionRegistryPostProcessor`，是指执行 `PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors`，执行 `BeanFactoryPostProcessor` 是指执行 `BeanFactoryPostProcessor#postProcessBeanFactory`

理解以上内容后，接下来就是对以上方法的详细解释了：

1.  执行 `BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry`:
    1.  执行开发者手动调用 `applicationContext.addBeanFactoryPostProcessor` 添加的 `BeanDefinitionRegistryPostProcessor` 的 `postProcessBeanDefinitionRegistry` 方法（一般情况下，开发者不会手动调用该方法）;
    2.  执行 spring 内部提供的、实现了 `PriorityOrdered` 接口的 `BeanDefinitionRegistryPostProcessor` 的 `postProcessBeanDefinitionRegistry` 方法；
    3.  执行实现了 `Ordered` 接口且未执行过的 `BeanDefinitionRegistryPostProcessor` 的 `postProcessBeanDefinitionRegistry` 方法；
    4.  执行以上未执行过的 `BeanDefinitionRegistryPostProcessor` 的 `postProcessBeanDefinitionRegistry` 方法；
2.  执行 `BeanFactoryPostProcessor#postProcessBeanFactory` 方法：
    1.  执行开发者调用 `applicationContext.addBeanFactoryPostProcessor` 添加的 `BeanDefinitionRegistryPostProcessor` 的 `postProcessBeanFactory` 方法（一般情况下，开发者不会手动调用该方法）;
    2.  执行以上从未执行过的、实现了 `PriorityOrdered` 的 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法；
    3.  执行以上从未执行过的、实现了 `Ordered` 的 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法；
    4.  执行以上从未执行过的 `BeanFactoryPostProcessor` 的 `postProcessBeanFactory` 方法

可以看到，这个方法实际就是为了执行两个方法：`BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry` 与 `BeanFactoryPostProcessor#postProcessBeanFactory`。需要注意的是，`BeanDefinitionRegistryPostProcessor` 是 `BeanFactoryPostProcessor` 的子类，在调用 `BeanFactoryPostProcessor#postProcessBeanFactory` 时 ，实际上也调用了 `BeanDefinitionRegistryPostProcessor` 的 `postProcessBeanFactory` 方法.

在以上的 `BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry` 与 `BeanFactoryPostProcessor#postProcessBeanFactory` 执行中，究竟执行了哪些代码呢？这里我们通过调试 `demo01`，发现执行的代码如下：

1.  在 `1.2` 步骤时，执行了 `ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry`
2.  在 `1.4` 步骤时，执行了 `ConfigurationClassPostProcessor#postProcessBeanFactory`
3.  在 `2.4` 步骤时，执行了 `EventListenerMethodProcessor#postProcessBeanFactory`

接着，我们便对以上三个方法进行展开，看看究竟做了什么。

#### 5.1 `ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry` 的执行流程

我们一跟下去，对不重要的方法只显示调用栈：

```
AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(java.lang.String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#invokeBeanFactoryPostProcessors
   |-PostProcessorRegistrationDelegate
      #invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory, List)
    |-PostProcessorRegistrationDelegate#invokeBeanDefinitionRegistryPostProcessors
     |-ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry
      |-ConfigurationClassPostProcessor#processConfigBeanDefinitions

```

我们直接进入 `ConfigurationClassPostProcessor#processConfigBeanDefinitions`：

```
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    String[] candidateNames = registry.getBeanDefinitionNames();

    //循环candidateNames数组
    for (String beanName : candidateNames) {
    BeanDefinition beanDef = registry.getBeanDefinition(beanName);

    // 内部有两个标记位来标记是否已经处理过了
    // 这里会引发一连串知识盲点
    // 当我们注册配置类的时候，可以不加Configuration注解，
    // 直接使用Component ComponentScan Import ImportResource注解，称之为Lite配置类
    // 如果加了Configuration注解，就称之为Full配置类
    // 如果我们注册了Lite配置类，我们getBean这个配置类，会发现它就是原本的那个配置类
    // 如果我们注册了Full配置类，我们getBean这个配置类，会发现它已经不是原本那个配置类了，
    // 而是已经被cgilb代理的类了
    if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
            if (logger.isDebugEnabled()) {
                logger.debug(...);
            }
        }
        // 判断是否为配置类，这里分两种情况：
       // 1\. 带有 @Configuration 注解 且 proxyBeanMethods != false 的类，spring 称其为 Full 配置类
       // 2\. 带有 @Configuration 注解 且 proxyBeanMethods == false,
       // 或 带有 @Component、@ComponentScan、@Import、@ImportResource、
       // @Bean 其中之一注解的类，spring 称其为 Lite 配置类
       // Full与Lite，beanDef会进行标识
       else if (ConfigurationClassUtils
               .checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
           configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
       }
    }

    // 如果没有配置类，直接返回
    if (configCandidates.isEmpty()) {
         return;
    }

    // 处理排序
    configCandidates.sort((bd1, bd2) -> {
         int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
         int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
         return Integer.compare(i1, i2);
    });

    SingletonBeanRegistry sbr = null;
    // DefaultListableBeanFactory最终会实现SingletonBeanRegistry接口，所以可以进入到这个if
    if (registry instanceof SingletonBeanRegistry) {
        sbr = (SingletonBeanRegistry) registry;
        if (!this.localBeanNameGeneratorSet) {
            //spring中可以修改默认的bean命名方式，这里就是看用户有没有自定义bean命名方式
            BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(
                     AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR);
            if (generator != null) {
                this.componentScanBeanNameGenerator = generator;
                this.importBeanNameGenerator = generator;
            }
        }
    }

    if (this.environment == null) {
         this.environment = new StandardEnvironment();
    }

    ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);

    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
        //解析配置类，这个类做了很多事，
        //如：对@Component，@PropertySources，@ComponentScans，@ImportResource等的处理
        parser.parse(candidates);
        parser.validate();

        Set<ConfigurationClass> configClasses
                = new LinkedHashSet<>(parser.getConfigurationClasses());
        configClasses.removeAll(alreadyParsed);

        if (this.reader == null) {
            this.reader = new ConfigurationClassBeanDefinitionReader(
                registry, this.sourceExtractor, this.resourceLoader, this.environment,
                this.importBeanNameGenerator, parser.getImportRegistry());
        }
        // 直到这一步才把Import的类，@Bean @ImportRosource 转换成BeanDefinition
        this.reader.loadBeanDefinitions(configClasses);
        // 把configClasses加入到alreadyParsed
        alreadyParsed.addAll(configClasses);

        candidates.clear();
        // 获得注册器里面BeanDefinition的数量 和 candidateNames进行比较
        // 如果大于的话，说明有新的BeanDefinition注册进来了
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            String[] newCandidateNames = registry.getBeanDefinitionNames();
            Set<String> oldCandidateNames = new HashSet<>(Arrays.asList(candidateNames));
            Set<String> alreadyParsedClasses = new HashSet<>();
            // 循环alreadyParsed。把类名加入到alreadyParsedClasses
            for (ConfigurationClass configurationClass : alreadyParsed) {
                alreadyParsedClasses.add(configurationClass.getMetadata().getClassName());
            }
            for (String candidateName : newCandidateNames) {
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    if (ConfigurationClassUtils
                                .checkConfigurationClassCandidate(bd, this.metadataReaderFactory) &&
                            !alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());

    if (sbr != null && !sbr.containsSingleton(IMPORT_REGISTRY_BEAN_NAME)) {
        sbr.registerSingleton(IMPORT_REGISTRY_BEAN_NAME, parser.getImportRegistry());
    }

    if (this.metadataReaderFactory instanceof CachingMetadataReaderFactory) {
        ((CachingMetadataReaderFactory) this.metadataReaderFactory).clearCache();
    }
}

```

以上方法还是对 `BeanDefinition` 信息的进一步完善，包括对 `@Configuration`、`@PropertySources`、`@ComponentScans`、`@ImportResource` 等的处理。由于 demo01 没有这些注解，这里我们就不展开了，后面我们再分析。

#### 5.2 执行 `ConfigurationClassPostProcessor#postProcessBeanFactory` 的流程

`ConfigurationClassPostProcessor` 的 `postProcessBeanFactory` 方法比较简单，所做的事还是对 `@Configuration` 的增强：

```
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    // 省略其他代码

    // 对 ConfigurationClasses 的增强
    enhanceConfigurationClasses(beanFactory);
    beanFactory.addBeanPostProcessor(new ImportAwareBeanPostProcessor(beanFactory));
}

public void enhanceConfigurationClasses(ConfigurableListableBeanFactory beanFactory) {
    // 省略其他代码

    // 全配置类：处理代理
    ConfigurationClassEnhancer enhancer = new ConfigurationClassEnhancer();
    for (Map.Entry<String, AbstractBeanDefinition> entry : configBeanDefs.entrySet()) {
        AbstractBeanDefinition beanDef = entry.getValue();
        // If a @Configuration class gets proxied, always proxy the target class
        beanDef.setAttribute(AutoProxyUtils.PRESERVE_TARGET_CLASS_ATTRIBUTE, Boolean.TRUE);
        // Set enhanced subclass of the user-specified bean class
        // 在这里进行增强的
        Class<?> configClass = beanDef.getBeanClass();
        Class<?> enhancedClass = enhancer.enhance(configClass, this.beanClassLoader);
        if (configClass != enhancedClass) {
            beanDef.setBeanClass(enhancedClass);
        }
    }
}

```

由于 demo01 没有配置 `@Configuration`，这里就不展开了，后面我们再分析。

#### 5.3 执行 `EventListenerMethodProcessor#postProcessBeanFactory` 的流程

这个方法是用来处理事件监听器的，我们直接上代码：

```
@Override
public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) {
    this.beanFactory = beanFactory;
    Map<String, EventListenerFactory> beans 
          = beanFactory.getBeansOfType(EventListenerFactory.class, false, false);

    List<EventListenerFactory> factories = new ArrayList<>(beans.values());
    AnnotationAwareOrderComparator.sort(factories);
    this.eventListenerFactories = factories;
}

```

可以看到，这里从 spring 容器中，拿出了所有的 `EventListenerFactory`，然后赋值给 `this.eventListenerFactories`，这个就不展开了。

#### 5.4 总结

本文介绍了 `invokeBeanFactoryPostProcessors` 的执行流程，整个类下来就是为了执行两个方法：`BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry` 与 `BeanFactoryPostProcessor#postProcessBeanFactory`，这两个方法的执行过程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b6188601112b2b2c031b0a70f64f2cc885f.png)

通过调试发现，`invokeBeanFactoryPostProcessors` 一共执行了 `BeanFactoryPostProcessor`：

1.  `ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry`
2.  `ConfigurationClassPostProcessor#postProcessBeanFactory`
3.  `EventListenerMethodProcessor#postProcessBeanFactory`

其中，`ConfigurationClassPostProcessor` 是一个非常非常重要的 `BeanFactoryPostProcessor`，关于它的进一步分析可以参考以下文章：

*   [spring 探秘之 ConfigurationClassPostProcessor 之处理 @ComponentScan 注解](https://my.oschina.net/funcy/blog/4836178)
*   [spring 探秘之 ConfigurationClassPostProcessor 之处理 @Bean 注解](https://my.oschina.net/funcy/blog/4492878)
*   [spring 探秘之 ConfigurationClassPostProcessor 之处理 @Import 注解](https://my.oschina.net/funcy/blog/4678152)
*   [spring 探秘之 ConfigurationClassPostProcessor 之处理 @Conditional 注解](https://my.oschina.net/funcy/blog/4873444)

*   * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4641114](https://my.oschina.net/funcy/blog/4641114) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_