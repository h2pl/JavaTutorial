本文是 `ConfigurationClassPostProcessor` 分析的第三篇，主要是分析 spring 对 `@Import` 注解的处理流程。

## 4. spring 是如何处理 @Import 注解的？

承接上文，我们继续分析 spring 对 `@Import` 注解的处理流程。

### 4.1 了解 `@Import` 注解

我们来看下 `@Import` 注解的定义：

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Import {
    /**
     * {@link Configuration @Configuration}, {@link ImportSelector},
     * {@link ImportBeanDefinitionRegistrar}, or regular component classes to import.
     */
    Class<?>[] value();
}
```

`@Import` 里有一个方法：`value()`，支持的类型是 `Class`，从上面的文档来看，可以支持类型有 4 种：

- 被 `@Configuration` 注解标记的类
- 实现了 `ImportSelector` 的类
- 实现了 `ImportBeanDefinitionRegistrar` 的类
- 普通类

接下来，我们通过一个 demo 来展示使用 `@Import` 如何将这四种类导入到 spring 中。

### 4.2 demo 准备

1. 首先准备 4 个 bean：

```
/**
 * Element01
 */
public class Element01 {
    public String desc() {
        return "this is element 01";
    }
}

/**
 * Element02
 */
public class Element02 {
    public String desc() {
        return "this is element 02";
    }
}

/**
 * Element03
 */
public class Element03 {
    public String desc() {
        return "this is element 03";
    }
}

/**
 * Element04
 */
public class Element04 {
    public String desc() {
        return "this is element 04";
    }
}
```

1. 准备实现 `ImportBeanDefinitionRegistrar` 的类，将 `element02` 注入其中

```
public class Element02ImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {

    /**
     * 在 registerBeanDefinitions 中注册element02对应的BeanDefinition
     * 也就是把 Element02 对应的 beanDefinition 手动注册到beanFactory 
     * 的 beanDefinitionMap 中
     */
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
            BeanDefinitionRegistry registry) {
        registry.registerBeanDefinition("element02", new RootBeanDefinition(Element02.class));
    }
}
```

1. 准备实现 `ImportSelector` 的类，在 `selectImports(...)` 方法中，返回 `Element03` 的 "包名。类名"

```
public class Element03Selector implements ImportSelector {
    /**
     * 返回String 为 包名.类名
     * 由于后面要用到反射，因此必须是"包名.类名"
     * @param importingClassMetadata
     * @return
     */
    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        return new String[] {Element03.class.getName()};
    }
}
```

1. 准备一个被 `@Configuration` 注解标记类，通过类中被 `@Bean` 标记的方法返回 `Element04`

```
@Configuration
public class Element04Configuration {
    @Bean
    public Element04 element04() {
        return new Element04();
    }

}
```

1. 定义 `@EnableElement` 注解，其中的 `@Import` 注解依次引入 `Element01.class`、`Element02ImportBeanDefinitionRegistrar.class`、`Element03Selector.class`、`Element04Configuration.class`

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({
        // 普通类 
        Element01.class,
        // 实现了 ImportBeanDefinitionRegistrar 的类
        Element02ImportBeanDefinitionRegistrar.class,
        // 实现了 ImportSelector 的类
        Element03Selector.class,
        // 被 @Configuration 标记的类
        Element04Configuration.class
})
public @interface EnableElement {

}
```

1. 主类：

```
// 只需要 @EnableElement 注解
@EnableElement
public class Demo04Main {

    public static void main(String[] args) {
        // 传入的是 Demo04Main.class
        ApplicationContext context = new AnnotationConfigApplicationContext(Demo04Main.class);

        Element01 element01 = context.getBean(Element01.class);
        System.out.println(element01.desc());

        Element02 element02 = context.getBean(Element02.class);
        System.out.println(element02.desc());

        Element03 element03 = context.getBean(Element03.class);
        System.out.println(element03.desc());

        Element04 element04 = context.getBean(Element04.class);
        System.out.println(element04.desc());
    }
}
```

完整代码见 [gitee/funcy](https://gitee.com/funcy/spring-framework/tree/v5.2.2.RELEASE_learn/spring-learn/src/main/java/org/springframework/learn/explore/demo04).

运行，结果如下：

```
this is element 01
this is element 02
this is element 03
this is element 04
```

可以看到，4 个 bean 都成功引入了 spring 容器。接下来，我们通过代码来看看 spring 是如何处理的。

注：本文是 `ConfigurationClassPostProcessor` 分析的第三篇，对与前面两篇文章（[处理 @ComponentScan 注解](https://my.oschina.net/funcy/blog/4836178)、[处理 @Bean 注解](https://my.oschina.net/funcy/blog/4492878)）雷同的代码，本文会一笔带过，不会具体分析了。

### 4.3 处理配置类：`ConfigurationClassPostProcessor#processConfigBeanDefinitions`

关于配置类的解流程，前两篇已经多次提到过了，这里我们直接看关键代码：

```
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    ...
    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
        // 1. 解析配置类, 对@Component，@PropertySources，@ComponentScans，
        // @ImportResource等的解析
        parser.parse(candidates);
        parser.validate();
        // 解析完成后，得到的配置类，配置类保存在 parser 的 configurationClasses 属性中
        Set<ConfigurationClass> configClasses 
                = new LinkedHashSet<>(parser.getConfigurationClasses());
        ...

        // 2. 把 @Import 引入的类、配置类中带@Bean的方法、
        // @ImportResource 引入的资源等转换成BeanDefinition
        this.reader.loadBeanDefinitions(configClasses);
        ...
        // 获得注册器里面BeanDefinition的数量 和 candidateNames进行比较
        // 如果大于的话，说明有新的BeanDefinition注册进来了
        if (registry.getBeanDefinitionCount() > candidateNames.length) {
            ...
            for (String candidateName : newCandidateNames) {
                // 过滤本次新增的BeanDefinition
                if (!oldCandidateNames.contains(candidateName)) {
                    BeanDefinition bd = registry.getBeanDefinition(candidateName);
                    // 3. 对新添加的类，如果是配置类，且未解析，添加到candidates中，等待下次循环处理
                    if (ConfigurationClassUtils.checkConfigurationClassCandidate(
                            bd, this.metadataReaderFactory) 
                            &&!alreadyParsedClasses.contains(bd.getBeanClassName())) {
                        candidates.add(new BeanDefinitionHolder(bd, candidateName));
                    }
                }
            }
            candidateNames = newCandidateNames;
        }
    }
    while (!candidates.isEmpty());
    ...
}
```

以上代码经过了一些精简，只保留了处理 `@Import` 关键步骤，总结如下：

1. 解析配置类，对 `@Component`，`@PropertySources`，`@ComponentScans`，`@ImportResource` 等的解析，这个 方法前两篇文章也分析了，这次我们聚集于 `@Import` 会再次分析，在这一步，要解析的配置类只有一个，就是我们在 `main()` 方法中注册的 `Demo04.class`：

   ![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-840c14685be569012b54d94aad67ee48825.png)

2. 把 `@Import` 引入的类、配置类中带 `@Bean` 的方法、`@ImportResource` 引入的资源等转换成 `BeanDefinition`，这个方法在前面分析 `@Bean` 的时候也分析过了，这次我们也会分析到；

3. 对新添加的类，如果是配置类，且未解析，添加到 `candidates` 中，等待下次循环处理。

### 4.4 解析配置类：`ConfigurationClassParser#doProcessConfigurationClass`

我们来看看 `@Import` 是如何解析的，进入 `ConfigurationClassParser#doProcessConfigurationClass`：

```
/**
 * 这个方法才是真正处理解析的方法
 */
protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, 
        SourceClass sourceClass) throws IOException {
    // 1. 如果是 @Component 注解，递归处理内部类，本文不关注
    ...

    // 2. 处理@PropertySource注解，本文不关注
    ...

    // 3. 处理 @ComponentScan/@ComponentScans 注解，本文不关注
    ...

    // 4. 处理@Import注解
    processImports(configClass, sourceClass, getImports(sourceClass), true);

    // 5. 处理@ImportResource注解，本文不关注
    ...

    // 6. 处理@Bean的注解，本文不关注
    ...

    // 7. 返回配置类的父类，会在 processConfigurationClass(...) 方法的下一次循环时解析
    ...
    return null;
}
```

处理 `@Import` 注解调用的是 `processImports(...)` 方法，我们继续：

#### 1. 获取 `@Import` 导入的类

让我们把目标放到 `processImports(...)` 方法：

```
processImports(configClass, sourceClass, getImports(sourceClass), true);
```

这个方法共传入了 4 个参数：

- `configClass`：配置类，就是 `demo04Main` 对应的配置类；
- `sourceClass`：对 `demo04Main` 及其上的注解的包装；
- `getImports(sourceClass)`：`getImports(...)` 用来获取传入 `sourceClass` 的所有 `@Import` 注解引入的类；
- `true`：布尔值，是否检查循环引入。

其中，获取 `@Import` 注解引入的类是 `getImports(...)` 的功能，我们先来看看这个方法是如何获取的：

```
private Set<SourceClass> getImports(SourceClass sourceClass) throws IOException {
    Set<SourceClass> imports = new LinkedHashSet<>();
    Set<SourceClass> visited = new LinkedHashSet<>();
    // 在这里获取
    collectImports(sourceClass, imports, visited);
    return imports;
}

/**
 * 具体的获取操作
 */
private void collectImports(SourceClass sourceClass, Set<SourceClass> imports, 
        Set<SourceClass> visited) throws IOException {
    if (visited.add(sourceClass)) {
        for (SourceClass annotation : sourceClass.getAnnotations()) {
            String annName = annotation.getMetadata().getClassName();
            if (!annName.equals(Import.class.getName())) {
                // 如果annotation的名称不是import，则递归调用 collectImports(...) 方法
                collectImports(annotation, imports, visited);
            }
        }
        // 获取当前类的 @Import 注解
        imports.addAll(sourceClass.getAnnotationAttributes(Import.class.getName(), "value"));
    }
}
```

获取 `@Import` 的方法是 `ConfigurationClassParser#collectImports`，这个方法里会获取获取，操作方式如下：

1. 获取传入类的所有注解；
2. 遍历这些注解，如果是 `@Import` 注解，则获取 `@Import` 的 `value` 值；否则，回到第一步，继续处理。

这样之后，配置类上 `demo04Main` 上的 `@EnableElement` 注解会被获取到，由于这个注解不是 `@Import` 注解，就继续获取 `@EnableElement` 其上的注解，此时发现 `@EnableElement` 有 `@Import` 注解，这时就会获取 `@Import` 的 `value()` 值，也就是 `@Import` 注解引入的类。

这个方法运行后 ，得到的结果如下：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-05c7fb442b4bb84d13ff6688e5fbf31cfa5.png)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-4e7153f9cbb922496e7c303cc3c35e7eceb.png)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b238838641af43b783fc9b14c4226f5c1eb.png)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d7e2063676e6e19850c35893b62f4ab76a9.png)

得到的结果为 `LinkedHashSet`，一次截图不方便，因此分为了 4 张图。可以看到，`@Import` 注解引入的 4 个类都获取到了：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d615198a8f6dbef1ac6a800526311f36078.png)

#### 2. 处理 `@Import` 导入的类

获取到 `@Import` 注解导入的类后，我们再来看 `processImports(...)` 方法：

```
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, boolean checkForCircularImports) {
    ...
    for (SourceClass candidate : importCandidates) {
        // 1. 引入类的类型是 ImportSelector
        if (candidate.isAssignable(ImportSelector.class)) {
            Class<?> candidateClass = candidate.loadClass();
            // 实例化引入的 ImportSelector，并且会执行 Aware 接口方法
            // 支持的Aware有BeanClassLoaderAware，BeanFactoryAware，EnvironmentAware，ResourceLoaderAware
            ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, 
                    ImportSelector.class, this.environment, this.resourceLoader, this.registry);
            if (selector instanceof DeferredImportSelector) {
                this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
            }
            else {
                // 执行 selectImports 方法，获取引入的类，数组中为引入类的"包名.类名"
                String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames);
                // 递归调用 processImports(...) 方法，再一次处理引入的类
                processImports(configClass, currentSourceClass, importSourceClasses, false);
            }
        }
        // 2. 引入类的类型是 ImportBeanDefinitionRegistrar
        else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
            Class<?> candidateClass = candidate.loadClass();
            // 实例化 ImportBeanDefinitionRegistrar，并且执行 Aware 接口的方法
            // 支持的Aware有BeanClassLoaderAware，BeanFactoryAware，EnvironmentAware，ResourceLoaderAware
            ImportBeanDefinitionRegistrar registrar = ParserStrategyUtils
                    .instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class, 
                    this.environment, this.resourceLoader, this.registry);
            // 将 ImportBeanDefinitionRegistrar 保存起来
            configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
        }
        // 3. 不是以上两者，调用 processConfigurationClass(...) 直接解析
        // 如果引入的类是配置类（包含 @Component、@Configuration、@Import 等注解），会在这里进行解析
        else {
            this.importStack.registerImport(
                    currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
            processConfigurationClass(candidate.asConfigClass(configClass));
        }
    }

    ...
}
```

以上是经过精简后的代码，我们只看处理 `@Import` 的关键步骤：

1. 如果引入类的类型是 `ImportSelector`，处理流程如下：

    1. 使用反射实例化 `ImportSelector`，之后再执行 `Aware` 接口方法，所以我们在处理 `ImportSelector` 时，还可以实现 `Aware` 接口，支持的 `Aware` 有 `BeanClassLoaderAware`，`BeanFactoryAware`，`EnvironmentAware`，`ResourceLoaderAware`；
    2. 执行 `ImportSelector` 实例的 `selectImports` 方法，这一步是为了获取引入的类，结果为 `Class[]`，引入类的 "包名。类名"，`Element03Selector` 中该方法代码如下：

   ```
   @Override
   public String[] selectImports(AnnotationMetadata importingClassMetadata) {
       return new String[] {Element03.class.getName()};
   }
   ```

   这一步会获取到 `Element03.class`; 3. 将获取到的 `Class` 数组，转换成 `SourceClass` 集合，再一次调用 `processImports(...)`，在第二次调用时，才是真正处理 `Element03`。

2. 如果引入类的类型是 `ImportBeanDefinitionRegistrar`，处理流程如下：

    1. 实例化 `ImportBeanDefinitionRegistrar`，并且执行 `Aware` 接口的方法，这一步同 `ImportSelector` 的实例化操作一样，不再赘述；
    2. 将上一步得到的 `ImportBeanDefinitionRegistrar` 实例保存到 `configClass` 中，后面我们再分析保存起来的实例是如何处理的；

3. 如果引入类的类型不是以上两者，调用 `processConfigurationClass(...)` 直接解析，这个方法前两篇文章已经多次提及了，就是用来解析 `@Component`、`@Import`、`@ComponentScan`、`@Configuration`、`@Bean` 等注解的，这一步是为了解析引入类中的这些注解，引入的 `Element01`、`Element02`、`Element03`(上面第 1 步中提到的，从 `Element03Selector` 中获取到 `Element03.class` 后，将其转换成 `SourceClass`，再次调用 `processImports(...)` 的过程) 都是在这一步解析的。

到了这里，我们发现，除了 `ImportBeanDefinitionRegistrar` 的方式外，其他三种引入方式（普通类、配置类、`ImportSelector` 的实现类）都是一样的处理方式，都是按**配置类的解析方式来处理**，最终调用的是 `processConfigurationClass(...)` 方法！这一 块我们前两篇文章已经多次分析了，这里就不再分析了，接下来我们来继续来看 `ImportBeanDefinitionRegistrar` 的处理。

### 4.5 加载 `BeanDefinitions`：`ConfigurationClassBeanDefinitionReader#loadBeanDefinitions`

在分析 `@Bean` 方法时，我们分析过 `ConfigurationClassBeanDefinitionReader#loadBeanDefinitions` 加载 `@Bean` 方法的流程，这里我们来看加载 `@Import` 的流程，我们直接进入关键方法 `ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass`：

```
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
    ...

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
    // 前面保存在configClass中的ImportBeanDefinitionRegistrars，在这里使用了
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

处理 `@Import` 的地方有两处：

1. 处理 `@Import` 引入的配置类
2. 处理 `@Import` 引入的 `ImportBeanDefinitionRegistrar`

我们分别来看看这两处是如何处理的。

#### 1. 处理 `@Import` 引入的配置类

处理 `@Import` 引入的配置类的相关代码为：

```
    // 处理 @Import 引入的配置类
    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
```

我们进入其中一探究竟：

```
private void registerBeanDefinitionForImportedConfigurationClass(ConfigurationClass configClass) {
    AnnotationMetadata metadata = configClass.getMetadata();
    // 创建 BeanDefinition，类型为 AnnotatedGenericBeanDefinition
    AnnotatedGenericBeanDefinition configBeanDef = new AnnotatedGenericBeanDefinition(metadata);
    // 处理一系列的属性
    ScopeMetadata scopeMetadata = scopeMetadataResolver.resolveScopeMetadata(configBeanDef);
    configBeanDef.setScope(scopeMetadata.getScopeName());
    String configBeanName = this.importBeanNameGenerator.generateBeanName(configBeanDef, this.registry);
    AnnotationConfigUtils.processCommonDefinitionAnnotations(configBeanDef, metadata);
    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(configBeanDef, configBeanName);
    definitionHolder = AnnotationConfigUtils
            .applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
    // 注册
    this.registry.registerBeanDefinition(definitionHolder.getBeanName(), 
            definitionHolder.getBeanDefinition());
    configClass.setBeanName(configBeanName);
}
```

这就是一个将类注册到 `BeanDefinition` 中的过程，使用的 `BeanDefinition` 是 `AnnotatedGenericBeanDefinition`。

#### 2. 处理 `@Import` 引入的 `ImportBeanDefinitionRegistrar`

处理该过程的方法为 `loadBeanDefinitionsFromRegistrars(...)`，代码如下：

```
private void loadBeanDefinitionsFromRegistrars(Map<ImportBeanDefinitionRegistrar, 
        AnnotationMetadata> registrars) {
    registrars.forEach((registrar, metadata) ->
            registrar.registerBeanDefinitions(metadata, this.registry, 
                    this.importBeanNameGenerator));
}
```

该方法先是遍历传入的 `ImportBeanDefinitionRegistrar` 集合，然后逐一调用其中的 `ImportBeanDefinitionRegistrar#registerBeanDefinitions(AnnotationMetadata, BeanDefinitionRegistry, BeanNameGenerator)` 方法，该方法位于 `ImportBeanDefinitionRegistrar` 接口，代码如下：

```
public interface ImportBeanDefinitionRegistrar {

    /**
     * 默认方法，默认实现仅仅做了一个调用
     */
    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
            BeanDefinitionRegistry registry,BeanNameGenerator importBeanNameGenerator) {
        registerBeanDefinitions(importingClassMetadata, registry);
    }

    /**
     * 在 Element02ImportBeanDefinitionRegistrar 中实现的方法
     */
    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
            BeanDefinitionRegistry registry) {
    }

}
```

`ImportBeanDefinitionRegistrar#registerBeanDefinitions(AnnotationMetadata, BeanDefinitionRegistry, BeanNameGenerator)` 仅是做了一个调用，最终调用的是 `ImportBeanDefinitionRegistrar#registerBeanDefinitions(AnnotationMetadata, BeanDefinitionRegistry)`，而我们的 `Element02ImportBeanDefinitionRegistrar` 正是实现了该方法：

```
public class Element02ImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
            BeanDefinitionRegistry registry) {
        registry.registerBeanDefinition("element02", new RootBeanDefinition(Element02.class));
    }
}
```

绕了大半天，最终发现 `ImportBeanDefinitionRegistrar` 注册到 `beanDefinitionMap` 的逻辑是我们自己写的！

#### 3. 从容器中获取 `ElementXx`

到了这里，`Element01`、`Element02`、`Element03`、`Element04` 就到注册到 `beanDefinitionMap` 中了，让我们看一眼 `beanDefinitionNames` 中的内容：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-68171a4af503b3e80b23d10a91834e231aa.png)

可以发现，`Element01` 与 `Element03` beanName 不同寻常，这两个 bean 的引入方式为

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0e07154a732c2a884803531e54aa10a0ef9.png)

这点在使用 `beanFactory.get(“beanName”)` 时需要注意：

```
// 获取不到，会报错
beanFactory.get("element01");
// 能获取到
beanFactory.get("element02");
// 获取不到，会报错
beanFactory.get("element03");
// 能获取到
beanFactory.get("element04");
```

使用 `beanFactory.get(“beanName”)` 获取 `element01` 与 `element03` 需要这样获取：

```
// 能获取到
beanFactory.get("org.springframework.learn.explore.demo04.element.Element01");
// 能获取到
beanFactory.get("org.springframework.learn.explore.demo04.element.Element03");
```

当然，我们也可以使用 `beanFactory.get(Class)` 的方式获取：

```
// 能获取到
beanFactory.get(Element01.class);
// 能获取到
beanFactory.get(Element02.class);
// 能获取到
beanFactory.get(Element03.class);
// 能获取到
beanFactory.get(Element04.class);
```

### 4.6 补充：`DeferredImportSelector` 的处理

在分析 `ConfigurationClassParser#processImports` 方法时，处理 `ImportSelector` 的类型时，有这么一段代码：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-4fb0dddcbf45d84ad5260c43a5b55e85a0a.png)

这段代码会先判断 `selector` 是否为 `DeferredImportSelector` 的实例，如果是就按 `DeferredImportSelector` 类型的实例进行处理，否则就按普通的 `ImportSelector` 类型来处理。本节我们来看看 `DeferredImportSelector` 与普通的 `ImportSelector` 有何不同。

`DeferredImportSelector` 的代码如下：

```
/**
 * DeferredImportSelector 是 ImportSelector的子接口
 */
public interface DeferredImportSelector extends ImportSelector {

    /**
     * 返回导入分组
     */
    @Nullable
    default Class<? extends Group> getImportGroup() {
        return null;
    }


    /**
     * 分组的定义
     */
    interface Group {

        /**
         * 该分组的处理方法
         */
        void process(AnnotationMetadata metadata, DeferredImportSelector selector);

        /**
         * 返回该分组的内容
         */
        Iterable<Entry> selectImports();


        /**
         * 分组的元素定义
         */
        class Entry {

            /**
             * 注解数据
             */
            private final AnnotationMetadata metadata;

            /**
             * 导入的类名
             */
            private final String importClassName;

            public Entry(AnnotationMetadata metadata, String importClassName) {
                this.metadata = metadata;
                this.importClassName = importClassName;
            }

            // 省略 get/set 方法，省略 equals/toString/hashCode 方法
            ...

        }
    }
}
```

从上面的代码可以看出，

- `DeferredImportSelector` 是 `ImportSelector` 的子接口，具备 `ImportSelector` 的功能
- `DeferredImportSelector` 提供了一个方法：`Class<? extends Group> getImportGroup()`，该方法返回的是当前 `DeferredImportSelector` 实例所在的分组。

接下来我们来关注这行代码：

```
this.deferredImportSelectorHandler.handle(configClass, (DeferredImportSelector) selector);
```

来看看这行代码做做了什么，进入 `ConfigurationClassParser#handler方法`：

```
class ConfigurationClassParser {

    ...

    private class DeferredImportSelectorHandler {

        @Nullable
        private List<DeferredImportSelectorHolder> deferredImportSelectors = new ArrayList<>();

        public void handle(ConfigurationClass configClass, DeferredImportSelector importSelector) {
            // 将 configClass 与 importSelector 包装成 DeferredImportSelectorHolder
            DeferredImportSelectorHolder holder = new DeferredImportSelectorHolder(
                    configClass, importSelector);
            if (this.deferredImportSelectors == null) {
                DeferredImportSelectorGroupingHandler handler 
                    = new DeferredImportSelectorGroupingHandler();
                handler.register(holder);
                handler.processGroupImports();
            }
            else {
                // 添加到 deferredImportSelectors 中
                this.deferredImportSelectors.add(holder);
            }
        }
        ...
    }
    ...
}
```

可以看到，这个方洗先将 `configClass` 与 `importSelector` 包装成 `DeferredImportSelector`，然后再添加到 `deferredImportSelectors`。

到目前为止，`DeferredImportSelector` 引入的类并没有进行处理，那么 `DeferredImportSelector` 引入的类是在哪里处理的呢？让我们回到 `ConfigurationClassParser#parse(Set<BeanDefinitionHolder>)` 方法：

```
public class  ConfigurationClassParser {
    public void parse(Set<BeanDefinitionHolder> configCandidates) {
        // 循环传进来的配置类
        for (BeanDefinitionHolder holder : configCandidates) {
            BeanDefinition bd = holder.getBeanDefinition();
            try {
                // 如果获得BeanDefinition是AnnotatedBeanDefinition的实例
                if (bd instanceof AnnotatedBeanDefinition) {
                    parse(((AnnotatedBeanDefinition) bd).getMetadata(), holder.getBeanName());
                }
                ...
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                ...
            }
        }
        // 在这里处理 DeferredImportSelector
        this.deferredImportSelectorHandler.process();
    }
    ...
}
```

可以看到，在处理完配置类的解析后，最后再来处理 `DeferredImportSelector`，也就是上面添加到 `deferredImportSelectors` 的内容，调用的是 `deferredImportSelectorHandler.process()`.

我们继续：

```
class ConfigurationClassParser {

    private class DeferredImportSelectorHandler {
        ...
        public void process() {
            List<DeferredImportSelectorHolder> deferredImports = this.deferredImportSelectors;
            this.deferredImportSelectors = null;
            try {
                if (deferredImports != null) {
                    DeferredImportSelectorGroupingHandler handler 
                        = new DeferredImportSelectorGroupingHandler();
                    // 排序，DeferredImportSelector 可以指定处理顺序，@Order/Orderd
                    deferredImports.sort(DEFERRED_IMPORT_COMPARATOR);
                    // 遍历调用 DeferredImportSelectorGroupingHandler#register 方法
                    deferredImports.forEach(handler::register);
                    // 处理导入
                    handler.processGroupImports();
                }
            }
            finally {
                this.deferredImportSelectors = new ArrayList<>();
            }
        }
        ...
    }
    ...
}
```

`process()` 方法步骤如下：

1. 排序，这个主要是根据 `@Order` 注解，或者实现了 `Orderd` 接口来排序；
2. 遍历调用 `DeferredImportSelectorGroupingHandler#register` 方法，其实就是将 `deferredImports` 中的元素注册到 `handler` 中；
3. 调用 `handler.processGroupImports()` 方法来处理导入。

我们再来看看 `handler.processGroupImports()` 方法：

```
class ConfigurationClassParser {

    ...

    private class DeferredImportSelectorGroupingHandler {
        ...
        /**
         * 最终的处理类
         * 在这里处理分组导入
         */
        public void processGroupImports() {
            for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
                // 遍历分组， grouping.getImports()是关键
                grouping.getImports().forEach(entry -> {
                    ConfigurationClass configurationClass = this.configurationClasses.get(
                            entry.getMetadata());
                    try {
                        // 同ImportSelector的实现类一样，最终也是调用 processImports(...) 处理导入
                        // 注意 entry.getImportClassName()，这一次调用 processImports(...) 的参
                        // 数是ImportSelector引入的类
                        processImports(configurationClass, asSourceClass(configurationClass),
                                asSourceClasses(entry.getImportClassName()), false);
                    }
                    catch (...) {
                        ...
                    }
                });
            }
        }
        ...
    }
    ...

}
```

`processGroupImports(...)` 主要逻辑如下：

1. 遍历分组
2. 调用 `grouping.getImports()` 获取分组的类
3. 调用 `processImports` 处理分组类的导入，这点同处理 `ImportSelector` 接口流程是一样的

我们来看看 `grouping.getImports()` 方法，该方法为 `ConfigurationClassParser.DeferredImportSelectorGrouping#getImports`，代码如下：

```
public Iterable<Group.Entry> getImports() {
    for (DeferredImportSelectorHolder deferredImport : this.deferredImports) {
        // 执行 Group#process
        this.group.process(deferredImport.getConfigurationClass().getMetadata(),
                deferredImport.getImportSelector());
    }
    // 执行 Group#selectImports
    return this.group.selectImports();
}
```

在这个方法里，会调用 `DeferredImportSelector.Group` 的两个方法：

- `DeferredImportSelector.Group#process`
- `DeferredImportSelector.Group#selectImports`

关于这两个方法，本小节的一开始就贴出了其代码及其注释，而引入类正是由 `DeferredImportSelector.Group#selectImports` 方法返回的.

分析了老半天，总结下 `DeferredImportSelector` 与 `ImportSelector` 两者的区别：

- `DeferredImportSelector` 可以指定分组：在处理时，可以根据分组统一处理；
- `DeferredImportSelector` 处理时机：会在所在配置类解析完之后再处理；
- `DeferredImportSelector` 引入类的返回：不同于 `ImportSelector`（其引入类由 `ImportSelector#selectImports` 方法返回），它的引入类由 `DeferredImportSelector.Group#selectImports` 方法返回。

### 4.7 总结

本文主要分析了 `ConfigurationClassPostProcessor` 对 `@Import` 注解的处理，总结如下:

1. `@Import` 可导入的类有 4 种，分别是普通类、配置类、实现了 `ImportSelector` 的类以及实现了 `ImportBeanDefinitionRegistrar` 的类；
2. 获取 `@Import` 注解：spring 在获取类上的 `@Import` 时，先获取类上的所有注解，然后逐一判断，如果当前注解是 `@Import`，则获取 `@Import` 的引入的类 (方法为 `Import#value`)，否则获取当前注解上的所有注解，重复以上处理；
3. 处理 `@Import` 引入的类：普通类、配置类统一按配置类来解析；实现了 `ImportSelector` 的类，会从 `selectImports(...)` 方法返回的 “包名。类名” 中拿到 class，然后也是按配置类解析；实现 `ImportBeanDefinitionRegistrar` 的类，会把对应类的实例保存到当前配置类中，后面注册到 `beanDefinitionMap` 时，就是从这里获取的；
4. 将引入类注册到 `beanDefinitionMap`：实现了 `ImportSelector` 的类最终得到的是一个普通类或配置类，同引入的普通类与配置类一样，直接注册；实现 `ImportBeanDefinitionRegistrar` 的类，调用其 `ImportBeanDefinitionRegistrar#registerBeanDefinitions(AnnotationMetadata, BeanDefinitionRegistry)` 方法进行注册，注册逻辑由实现类自行定义。

------

*本文原文链接：https://my.oschina.net/funcy/blog/4678152 ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。*