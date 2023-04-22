在 [springboot 自动装配之条件注解（一）](https://my.oschina.net/funcy/blog/4918863)一文中，在分析 `@ConditionalOnBean/@ConditionalOnMissingBean` 注解的条件判断时，官方强烈建议我们在自动装配类中使用这两个注解，并且 `@ConditionalOnBean/@ConditionalOnMissingBean` 标记的类要在指定的类之后初始化，那 springboot 如何来控制自动装配顺序呢？本文将来研究下。

### 1\. springboot 处理自动装配类的过程

需要明确的是，本文探讨的`自动装配顺序`是指将 `class` 注册到 `beanFactory` 的顺序，springboot 处理自动装配类的大致过程如下：

1.  加载自动装配类，在 [springboot 自动装配之加载自动装配类](https://my.oschina.net/funcy/blog/4870868) 一文中已分析过；
2.  对自动装配类进行排序，这是本文将要分析的内容；
3.  遍历自动装配类，对每个自动装配类逐一进行以下操作：
    1.  根据条件注解判断当前自动装配类是否满足装配条件；
    2.  如果当前自动装配类满嘴装配条件，注册到 `beanFactory` 中。

再回到 `@ConditionalOnBean/@ConditionalOnMissingBean`，对如下两个自动装配类：

```
// A是自动装配类
@Configuration
public class A {
    @Bean
    @ConditionalOnMissingBean("b1")
    public A1 a1() {
        return new A1();
    }
}

// B是自动装配类
@Configuration
public class B {
    @Bean
    public B1 b1() {
        return new b1();
    }
}

```

`a1` 与 `b1` 在两个不同的自动装配类中初始化，且 `a1` 只有在 `b1` 不存在时，才会初始化，根据上面总结的 springboot 处理自动装配类的步骤，我们只要指定 `b1` 在 `a1` 之前初始化就不会产生异常了。

那么，自动装配类的顺序如何指定呢？

### 2\. 自动装配类的顺序控制注解

springboot 为我们提供了两种自动装配类的排序手段：

*   绝对自动装配顺序 ——`@AutoConfigOrder`
*   相对自动装配顺序 ——`@AutoConfigureBefore` 与 `@AutoConfigureAfter`

这三个注解就是用来处理自动装配类的排序的了，`@AutoConfigOrder` 指定了装配顺序，同 spring 提供的 `@Order` 类似，`@AutoConfigureBefore` 与 `@AutoConfigureAfter` 可以指定 `class`，表示在哪个 `class` 之前或之后装配。

回到示例，我们可以这样指定装配顺序：

```
// A是自动装配类
@Configuration
// 在B.class之后自动装配
@AutoConfigureAfter(B.class)
public class A {
    @Bean
    @ConditionalOnMissingBean("b1")
    public A1 a1() {
    ...
    }
}

// B是自动装配类
@Configuration
public class B {
    ...
}

```

### 3\. 自动装配类的排序

前面我们提到，`@AutoConfigOrder`、`@AutoConfigureBefore` 与 `@AutoConfigureAfter` 可以控制自动装配类的装配顺序，那么它们是在哪里进行排序的呢？在 [springboot 自动装配之加载自动装配类](https://my.oschina.net/funcy/blog/4870868) 一文中，我们提总结了获取自动装配类的步骤有 6 步：

1.  调用 `AutoConfigurationImportSelector#getAutoConfigurationEntry(...)` 方法加载自动装配类；
2.  将得到的自动装配类保存到 `autoConfigurationEntries` 中；
3.  得到过滤类，这些过滤类就是由 `@EnableAutoConfiguration` 的 `exclude` 或 `excludeName` 指定的；
4.  将 `autoConfigurationEntries` 转换为 `LinkedHashSet`，结果为 `processedConfigurations`；
5.  去除 `processedConfigurations` 需要过滤的类；
6.  将第 5 步得到的类排序后，返回。

而对自动装配类的排序正是在第 6 步，对应的方法是 `AutoConfigurationImportSelector.AutoConfigurationGroup#sortAutoConfigurations`，代码如下：

```
private List<String> sortAutoConfigurations(Set<String> configurations,
        AutoConfigurationMetadata autoConfigurationMetadata) {
    // 先创建了 AutoConfigurationSorter 对象，
    // 然后调用 AutoConfigurationSorter.getInPriorityOrder 进行排序
    return new AutoConfigurationSorter(getMetadataReaderFactory(), autoConfigurationMetadata)
            .getInPriorityOrder(configurations);
}

```

这个方法的处理分为两步：

1.  创建了 `AutoConfigurationSorter` 对象
2.  调用 `AutoConfigurationSorter.getInPriorityOrder` 进行排序

我们先来看看 `AutoConfigurationSorter` 的创建操作：

```
class AutoConfigurationSorter {

    private final MetadataReaderFactory metadataReaderFactory;

    private final AutoConfigurationMetadata autoConfigurationMetadata;

    /**
     * 构造方法
     * 仅仅只是对传入的参数进行赋值，将他们赋值为成员变量
     */
    AutoConfigurationSorter(MetadataReaderFactory metadataReaderFactory,
            AutoConfigurationMetadata autoConfigurationMetadata) {
        Assert.notNull(metadataReaderFactory, "MetadataReaderFactory must not be null");
        this.metadataReaderFactory = metadataReaderFactory;
        this.autoConfigurationMetadata = autoConfigurationMetadata;
    }

    ...

}

```

可以看到，`AutoConfigurationSorter` 的构造方法并没有做什么实质性的操作，看来排序的关键还得看 `AutoConfigurationSorter.getInPriorityOrder` 方法，该方法的代码如下：

```
List<String> getInPriorityOrder(Collection<String> classNames) {
    // 1\. 将 classNames 包装成 AutoConfigurationClasses
    AutoConfigurationClasses classes = new AutoConfigurationClasses(this.metadataReaderFactory,
            this.autoConfigurationMetadata, classNames);
    List<String> orderedClassNames = new ArrayList<>(classNames);
    // 2\. 按类名排序
    Collections.sort(orderedClassNames);
    // 3\. 使用 @AutoConfigureOrder 排序
    orderedClassNames.sort((o1, o2) -> {
        int i1 = classes.get(o1).getOrder();
        int i2 = classes.get(o2).getOrder();
        return Integer.compare(i1, i2);
    });
    // 4\. 使用 @AutoConfigureBefore，@AutoConfigureAfter 排序
    orderedClassNames = sortByAnnotation(classes, orderedClassNames);
    return orderedClassNames;
}

```

从代码来 看，这个方法的执行步骤如下：

1.  将 `classNames` 包装成 `AutoConfigurationClasses`
2.  按类名排序
3.  使用 `@AutoConfigureOrder` 排序
4.  使用 `@AutoConfigureBefore`，`@AutoConfigureAfter` 排序

这个方法排序共进行了 3 次，都是对 `orderedClassNames` 进行排序，这样一来，后面的排序会打乱前面的排序，最先的排序是按类名排序，也就是说，如果没有指定 `@AutoConfigureOrder`、`@AutoConfigureBefore` 等注解，就会使用类名进行排序。

接下来我们具体分析这几个操作吧。

### 4\. 将 `classNames` 包装成 `AutoConfigurationClasses`

该操作位于 `AutoConfigurationSorter.AutoConfigurationClasses#AutoConfigurationClasses` 方法，代码如下：

```
private static class AutoConfigurationClasses {

    // 保存结果
    private final Map<String, AutoConfigurationClass> classes = new HashMap<>();

    /**
     * 构造方法
     */
    AutoConfigurationClasses(MetadataReaderFactory metadataReaderFactory,
            AutoConfigurationMetadata autoConfigurationMetadata, Collection<String> classNames) {
        // 进行方法调用
        addToClasses(metadataReaderFactory, autoConfigurationMetadata, classNames, true);
    }

    /**
     * 添加类，就是将类包装成 AutoConfigurationClass，添加到名为 classes 的 Map 中
     * classNames 就是去除了排除类的所有自动装配类
     */
    private void addToClasses(MetadataReaderFactory metadataReaderFactory,
            AutoConfigurationMetadata autoConfigurationMetadata, Collection<String> classNames, 
            boolean required) {
        for (String className : classNames) {
            if (!this.classes.containsKey(className)) {
                // 将 className 包装成 AutoConfigurationClass
                AutoConfigurationClass autoConfigurationClass = new AutoConfigurationClass(
                        className, metadataReaderFactory, autoConfigurationMetadata);
                boolean available = autoConfigurationClass.isAvailable();
                // @AutoConfigureBefore 与 @AutoConfigureAfter 标记的类的 required 为 false
                if (required || available) {
                    this.classes.put(className, autoConfigurationClass);
                }
                if (available) {
                    // 递归调用
                    addToClasses(metadataReaderFactory, autoConfigurationMetadata,
                            autoConfigurationClass.getBefore(), false);
                    addToClasses(metadataReaderFactory, autoConfigurationMetadata,
                            autoConfigurationClass.getAfter(), false);
                }
            }
        }
    }
    ...
}

```

从以上代码来看，

*   `AutoConfigurationClasses` 包含一个成员变量：`classes`，类型是 `Map`，`key` 是 `String`（也就是 `className`），`value` 是 `AutoConfigurationClass`（也就是 `className` 的包含类）;
*   `AutoConfigurationClasses` 的构造方法会调用 `addToClasses(...)` 该方法会遍历传入的 `classNames`，将其包装成 `AutoConfigurationClass` 后，再保存到 `classes` 中。

在分析 `addToClasses(...)` 的具体逻辑前，我们先来看看 `AutoConfigurationClass` 是个啥：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-fd9c1a4c335951391a33f86a86fe89ff496.png)

可以看到 ，`AutoConfigurationClass` 是类名的包装，并且还保存了 `@AutoConfigureBefore` 与 `@AutoConfigureAfter` 指定的类，以及提供了跟 `@AutoConfigureOrder`、 `@AutoConfigureBefore`、`@AutoConfigureAfter` 相关的一些方法。

我们再回过头看 `addToClasses(...)` 的执行流程，该方法的执行流程如下：

1.  遍历传入的 `classNames`，对其中每一个 `className`，进行下面的操作；
2.  创建 `AutoConfigurationClass`，传入 `className`；
3.  调用 `AutoConfigurationSorter.AutoConfigurationClass#isAvailable` 方法，得到 `available`；
4.  判断 `available` 与 `required` 的值，如果其一为 ture，就将其添加到 `classes`；
5.  如果 `available` 为 `true`，递归处理 `className` 由 `@AutoConfigureBefore` 与 `@AutoConfigureAfter` 指定的类。

流程看着不复杂，不过有几个就去需要分析下 ：

1.  `AutoConfigurationSorter.AutoConfigurationClass#isAvailable`：判断当前 `class` 是否存在
2.  `AutoConfigurationSorter.AutoConfigurationClass#getBefore`：获取 `class`：当前 `class` 需要在这些 `class` 之前处理
3.  `AutoConfigurationSorter.AutoConfigurationClass#getAfter`：获取 `class`：当前 `class` 需要在这些 `class` 之后处理

接下来我们一一来分析下这几个方法。

#### 4.1 `AutoConfigurationSorter.AutoConfigurationClass#isAvailable`

这个方法是用来判断当前 `class` 是否在当前项目的 `classpath` 中，看代码：

```
boolean isAvailable() {
    try {
        if (!wasProcessed()) {
            getAnnotationMetadata();
        }
        return true;
    }
    catch (Exception ex) {
        return false;
    }
}

```

这个方法代码不多，先是调用 `wasProcessed()` 方法，再调用 `getAnnotationMetadata()`，需要注意的是，`getAnnotationMetadata()` 可能会抛出异常，招聘异常也会返回 `false`.

我们继续跟进 `AutoConfigurationSorter.AutoConfigurationClass#wasProcessed` 方法：

```
private boolean wasProcessed() {
    return (this.autoConfigurationMetadata != null
        // 判断 META-INF/spring-autoconfigure-metadata.properties 文件中是否存在该配置
        && this.autoConfigurationMetadata.wasProcessed(this.className));
}

```

这个方法里主要调用了 `AutoConfigurationMetadataLoader.PropertiesAutoConfigurationMetadata#wasProcessed` 方法来判断：

```
@Override
public boolean wasProcessed(String className) {
    // 判断 properties 是否存在对应的 className
    return this.properties.containsKey(className);
}

```

可以看到，这个方法仅是判断 `properties` 里是否包含传入的 `className`，`properties` 的内容来自于 `META-INF/spring-autoconfigure-metadata.properties`，内容示例如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-069444fdaa952c6fa8c39576e776275982b.png)

需要注意的是，该文件在源码中是不存在的，它是在编译时写入的，关于该文件的写入、加载到 `properties` 的流程，本文就不展开分析了，这里提供个大概思路：

*   文件的写入：在代码编译时，springboot 会将自动装配类的一些信息 (例如，`@ConditionalOnClass` 指定的 `class`，`@ConditionalOnBean` 指定的 `bean`，`@AutoConfigureBefore` 与 `@AutoConfigureAfter` 的指定的 `class` 等) 写入到 `META-INF/spring-autoconfigure-metadata.properties` 文件中，处理类为 `AutoConfigureAnnotationProcessor`，这个类是 `javax.annotation.processing.AbstractProcessor` 的子类，而 `AbstractProcessor` 由 jdk 提供，可以在编译期对注解进行处理；

*   文件的加载：在 `AutoConfigurationImportSelector.AutoConfigurationGroup#process` 方法中调用 `AutoConfigurationImportSelector#getAutoConfigurationEntry` 时，会传入 `AutoConfigurationMetadata`，文件 `META-INF/spring-autoconfigure-metadata.properties` 中的内容就是从这里加载到 `AutoConfigurationMetadataLoader.PropertiesAutoConfigurationMetadata#properties` 中的；

由些可见，`AutoConfigurationMetadataLoader.PropertiesAutoConfigurationMetadata#wasProcessed` 方法实际上就是判断 `META-INF/spring-autoconfigure-metadata.properties` 文件中是否有 `className` 的配置。

让我们回到 `AutoConfigurationSorter.AutoConfigurationClass#isAvailable`，再来看看另一个方法：`getAnnotationMetadata()`，该方法位于 `AutoConfigurationSorter.AutoConfigurationClass` 中，代码如下：

```
private AnnotationMetadata getAnnotationMetadata() {
    if (this.annotationMetadata == null) {
        try {
            // 加载`className`对应的资源，当 className 对应的资源不存在时，会抛出异常
            MetadataReader metadataReader = this.metadataReaderFactory
                    .getMetadataReader(this.className);
            this.annotationMetadata = metadataReader.getAnnotationMetadata();
        }
        catch (IOException ex) {
            throw new IllegalStateException(...);
        }
    }
    return this.annotationMetadata;
}

```

继续进入 `SimpleMetadataReaderFactory#getMetadataReader(String)`：

```
@Override
/**
 * 这个方法会获取 className 对应的 .class 文件
 * 如果 .class 文件不存在，就报异常了：IOException
 */
public MetadataReader getMetadataReader(String className) throws IOException {
    try {
        // 转换名称："classpath:xxx/xxx/Xxx.class"
        String resourcePath = ResourceLoader.CLASSPATH_URL_PREFIX 
                + lassUtils.convertClassNameToResourcePath(className) 
                + ClassUtils.CLASS_FILE_SUFFIX;
        // 获取资源，默认的 resourceLoader 为 classLoader
        Resource resource = this.resourceLoader.getResource(resourcePath);
        // 将 resource 转换成 MetadataReader 对象，不存在就会抛出异常：IOException
        return getMetadataReader(resource);
    }
    catch (FileNotFoundException ex) {
        // 有可能是内部类，再按内部类的命名方式处理一次
        int lastDotIndex = className.lastIndexOf('.');
        if (lastDotIndex != -1) {
            String innerClassName = className.substring(0, lastDotIndex) + '$' 
                    + className.substring(lastDotIndex + 1);
            // 转换名称："classpath:xxx/Xxx$Xxx.class"
            String innerClassResourcePath = ResourceLoader.CLASSPATH_URL_PREFIX 
                    + ClassUtils.convertClassNameToResourcePath(innerClassName) 
                    + ClassUtils.CLASS_FILE_SUFFIX;
            Resource innerClassResource = this.resourceLoader.getResource(innerClassResourcePath);
            // 判断是否存在，不存在还是会报异常的：IOException
            if (innerClassResource.exists()) {
                return getMetadataReader(innerClassResource);
            }
        }
        throw ex;
    }
}

```

这个方法的处理流程如下：

1.  将传入的 `className` 转换为 `classpath:xxx/xxx/Xxx.class` 的形式，然后去加载对应的资源，如果资源不存在即 `className` 对应的`.class` 文件不存在，则抛出异常；
2.  在异常的 `catch` 块中，为了防止 `className` 是内部类，会将 `className` 转换为 `classpath:xxx/Xxx$Xxx.class` 的形式，然后再加载一次资源，如果资源存在，直接返回，否则将异常往外抛；

到了这里，我们就明白了，`getAnnotationMetadata()` 就是用来判断当前 `className` 对应的`.class` 在项目的 `classpath` 路径中是否存在。

对这两个方法，我们总结如下：

*   `AutoConfigurationSorter.AutoConfigurationClass#wasProcessed`：当前 `className` 是否在 `META-INF/spring-autoconfigure-metadata.properties` 文件中
*   `AutoConfigurationSorter.AutoConfigurationClass#isAvailable`：当前 `className` 对应的`.class` 文件是否存在

最终的结论：`AutoConfigurationSorter.AutoConfigurationClass#isAvailable` 就是用来判断当前 `className` 对应的`.class` 文件在项目的 `classpath` 路径中.

#### 4.2 `AutoConfigurationSorter.AutoConfigurationClass#getBefore/getAfter`

接下来我们来看看 `AutoConfigurationSorter.AutoConfigurationClass` 类的两个方法：`getAfter()` 与 `getBefore()`：

```
Set<String> getBefore() {
    if (this.before == null) {
        this.before = (wasProcessed() 
            // 如果存在于 `META-INF/spring-autoconfigure-metadata.properties` 文件中，直接获取值
            ? this.autoConfigurationMetadata.getSet(this.className, "AutoConfigureBefore", 
                    Collections.emptySet()) 
            // 否则从 @AutoConfigureBefore 注解上获取
            : getAnnotationValue(AutoConfigureBefore.class));
    }
    return this.before;
}

Set<String> getAfter() {
    if (this.after == null) {
        this.after = (wasProcessed() 
            // 如果存在于 `META-INF/spring-autoconfigure-metadata.properties` 文件中，直接获取值
            ? this.autoConfigurationMetadata.getSet(this.className, "AutoConfigureAfter", 
                    Collections.emptySet()) 
            // 否则从 @AutoConfigureAfter 注解上获取
            : getAnnotationValue(AutoConfigureAfter.class));
    }
    return this.after;
}

/**
 * 从 @AutoConfigureBefore/@AutoConfigureAfter 注解中获取值：value 与 name 指定的值
 */
private Set<String> getAnnotationValue(Class<?> annotation) {
    Map<String, Object> attributes = getAnnotationMetadata()
            .getAnnotationAttributes(annotation.getName(), true);
    if (attributes == null) {
        return Collections.emptySet();
    }
    Set<String> value = new LinkedHashSet<>();
    Collections.addAll(value, (String[]) attributes.get("value"));
    Collections.addAll(value, (String[]) attributes.get("name"));
    return value;
}

```

这两个方法在代码形式基本一致，先看 `getBefore()` 的流程：

1.  如果当前 `className` 存在于 `META-INF/spring-autoconfigure-metadata.properties` 文件中，直接取值，前面分析也提到，springboot 在编译时，会把一些注解的信息写入到 `META-INF/spring-autoconfigure-metadata.properties` 文件中；

2.  如果第 1 步不成功，则从当前 `class` 的 `@AutoConfigureBefore` 取值；

`getAfter()` 方法的流程与 `getBefore()` 的流程基本一致，就不分析了。

### 5\. 使用 `@AutoConfigureOrder` 排序

让我们回到 `AutoConfigurationSorter#getInPriorityOrder` 方法，我们来看看 `@AutoConfigureOrder` 的排序过程：

```
List<String> getInPriorityOrder(Collection<String> classNames) {
    ...
    orderedClassNames.sort((o1, o2) -> {
        int i1 = classes.get(o1).getOrder();
        int i2 = classes.get(o2).getOrder();
        return Integer.compare(i1, i2);
    });
    ...
}

```

这个排序操作使用的是 `List#sort`，`sort(...)` 里的参数为 `Comparator`，指定了排序规则。从代码来看，通过 `getOrder()` 获取到当前类的顺序后，再使用的是 `Integer` 的比较规则进行排序，因此 `getOrder()` 是排序的关键，它所对就的方法是 `AutoConfigurationSorter.AutoConfigurationClass#getOrder`，代码如下：

```
private int getOrder() {
    // 判断 META-INF/spring-autoconfigure-metadata.properties 文件中是否存在当前 className
    if (wasProcessed()) {
        // 如果存在，就使用文件中指定的顺序，否则就使用默认顺序
        return this.autoConfigurationMetadata.getInteger(this.className, 
                "AutoConfigureOrder", AutoConfigureOrder.DEFAULT_ORDER);
    }
    // 处理不存在的情况：获取 @AutoConfigureOrder 注解指定的顺序
    Map<String, Object> attributes = getAnnotationMetadata()
            .getAnnotationAttributes(AutoConfigureOrder.class.getName());
    // 如果 @AutoConfigureOrder 未配置，就使用默认顺序
    return (attributes != null) ? (Integer) attributes.get("value") 
            : AutoConfigureOrder.DEFAULT_ORDER;
}

```

这个方法还是比较简单的，就是获取 `@AutoConfigureOrder` 注解指定的顺序，如果没有 `@AutoConfigureOrder` 注解，就使用默认顺序，默认顺序 `AutoConfigureOrder.DEFAULT_ORDER` 的值为 0。

### 6\. 使用 `@AutoConfigureBefore`，`@AutoConfigureAfter` 排序

接下来就是最激动人心的 `@AutoConfigureBefore` 与 `@AutoConfigureAfter` 注解的排序了，对应的方法为 `AutoConfigurationSorter#sortByAnnotation`，代码如下：

```
/**
 * 进行排序，
 * 实际上这个方法里只是准备了一些数据，真正干活的是 doSortByAfterAnnotation(...)
 */
private List<String> sortByAnnotation(AutoConfigurationClasses classes, List<String> classNames) {
    // 需要排序的 className
    List<String> toSort = new ArrayList<>(classNames);
    toSort.addAll(classes.getAllNames());
    // 排序好的 className
    Set<String> sorted = new LinkedHashSet<>();
    // 正在排序中的 className
    Set<String> processing = new LinkedHashSet<>();
    while (!toSort.isEmpty()) {
        // 真正处理排序的方法
        doSortByAfterAnnotation(classes, toSort, sorted, processing, null);
    }
    // 存在于集合 sorted 中，但不存在于 classNames 中的元素将会被移除
    sorted.retainAll(classNames);
    return new ArrayList<>(sorted);
}

/**
 * 具体进行排序的方法
 */
private void doSortByAfterAnnotation(AutoConfigurationClasses classes, List<String> toSort, 
        Set<String> sorted, Set<String> processing, String current) {
    if (current == null) {
        current = toSort.remove(0);
    }
    // 使用 processing 来判断是否存在循环比较，比如，类A after 类B，而 类B 又 after 类A
    processing.add(current);
    // classes.getClassesRequestedAfter：当前 className 需要在哪些 className 之后执行
    for (String after : classes.getClassesRequestedAfter(current)) {
        Assert.state(!processing.contains(after),
                "AutoConfigure cycle detected between " + current + " and " + after);
        if (!sorted.contains(after) && toSort.contains(after)) {
            // 递归调用
            doSortByAfterAnnotation(classes, toSort, sorted, processing, after);
        }
    }
    processing.remove(current);
    // 添加到已排序结果中
    sorted.add(current);
}

```

`AutoConfigurationSorter#sortByAnnotation` 提供了保存数据的结构，而 `AutoConfigurationSorter#doSortByAfterAnnotation` 才是真正处理排序的方法，排序操作不太好懂，大致流程如下：

1.  查找当前 `className` 需要在哪些 `className` 之后装配，将其保存为 `afterClasses`，也就是说，`afterClasses` 中的每一个 `className` 都要在当前 `className` 之前装配；

2.  遍历 `afterClasses`，对其中每一个 `className`，继续查找其 `afterClasses`，这样递归下去，不考虑循环比较的情况下，最终必然会存在一个 `className`，它的 `afterClasses` 为空，这里就把 `className` 加入到已完成排序的结构中。

我们再来看看获取 `afterClasses` 的操作，方法为 `AutoConfigurationSorter.AutoConfigurationClasses#getClassesRequestedAfter`，代码如下：

```
Set<String> getClassesRequestedAfter(String className) {
    // 当前类：获取在哪些类之后执行，就是获取 @AutoConfigureAfter 注解指定的类
    Set<String> classesRequestedAfter = new LinkedHashSet<>(get(className).getAfter());
    // 其他类：需要前置执行的类中
    this.classes.forEach((name, autoConfigurationClass) -> {
        if (autoConfigurationClass.getBefore().contains(className)) {
            classesRequestedAfter.add(name);
        }
    });
    return classesRequestedAfter;
}

```

从代码来的来看，这个 `afterClasses` 包含两个内容：

*   获取在哪些类装配完成之后装配，就是获取 `@AutoConfigureAfter` 注解指定的类
*   获取哪些类需要在当前类装配之前进行装配

### 7\. 再来看：`@ConditionalOnBean/@ConditionalOnMissingBean`

前面提到了 `@ConditionalOnBean/@ConditionalOnMissingBean` 的坑，了解完自动装配的顺序后，就能很好规避这些坑了：

1.  两个 `bean` 都是自动装配类：避坑方式是，使用 `@AutoConfigureBefore` / `@AutoConfigureAfter` 或 `@AutoConfigureOrder` 指定条件顺序，保证条件注解中的 `bean` 先装配即可；
2.  一个是普通 `spring bean`，一个是自动装配类：如果条件注解中的 `bean` 是普通 spring bean，另一个是自动装配类，这种情况下不用处理，自动装配的处理类是 `DeferredImportSelector` 的子类，先天决定自动装配类在普通 `spring bean` 之后处理；反之 ，条件注解中的 `bean` 是自动装配类，另一个是普通 `spring bean`，这种一定会出错，不要使用；
3.  两个都是普通 `spring bean`：无避坑方法，`spring bean` 注册到 `beanFactory` 的顺序不可控，不建议在这种情况下使用；

### 8\. 总结

本文总结了自动装配类的装配顺序，主要介绍了如下内容：

1.  对自动装配类排序：`AutoConfigurationImportSelector.AutoConfigurationGroup#sortAutoConfigurations`
2.  指定自动装配类的装配顺序：使用 `@AutoConfigureBefore` / `@AutoConfigureAfter` 或 `@AutoConfigureOrder`
3.  排序方式有三种，依次是：
    1.  按 className 排序，由 `String` 提供排序规则
    2.  根据 `@AutoConfigureOrder` 指定的值进行排序，由 `Integer` 提供排序规则
    3.  根据 `@AutoConfigureBefore` / `@AutoConfigureAfter` 进行排序 需要注意的是，以上三种排序方式先后进行，以最后排序完的结果为最终顺序
4.  关于 `@ConditionalOnBean/@ConditionalOnMissingBean` 避坑指南：
    1.  两个 `bean` 都是自动装配类：避坑方式是，使用 `@AutoConfigureBefore` / `@AutoConfigureAfter` 或 `@AutoConfigureOrder` 指定条件顺序，保证条件注解中的 `bean` 先装配即可；
    2.  一个是普通 `spring bean`，一个是自动装配类：条件注解中的 `bean` 必须为普通的 `spring bean`；
    3.  其他情况不可控，不建议使用。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4921594](https://my.oschina.net/funcy/blog/4921594) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_