自动装配是 springboot 的核心之一，本文将来探究 springboot 是如何加载自动装配类的。

在 [@SpringBootApplication 注解](https://my.oschina.net/funcy/blog/4870882)一文中，我们提到 springboot 处理自动装配的注解是 `@EnableAutoConfiguration`，代码如下：

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

以上代码包含三个部分：

1.  `@AutoConfigurationPackage`：指定自动装配的包；
2.  `@Import(AutoConfigurationImportSelector.class)`：引入自动装配的处理类 `AutoConfigurationImportSelector`，这个类是自动装配的关键所在；
3.  `@EnableAutoConfiguration` 的属性：`@EnableAutoConfiguration` 提供了两个属性：`exclude` 与 `excludeName`，可以用来排除不需要自动装配的类。

本文重点来分析 `AutoConfigurationImportSelector` 类。

### 1. `AutoConfigurationImportSelector.AutoConfigurationGroup`

`AutoConfigurationImportSelector` 实现了 `DeferredImportSelector`，关于 `DeferredImportSelector` 的分析，可以参考 [ConfigurationClassPostProcessor 之处理 @Import 注解](https://my.oschina.net/funcy/blog/4678152)，这里我们直接给出结论：

* `DeferredImportSelector` 是 `ImportSelector` 的子接口，其内部有一个接口 `Group`，该接口定义了两个方法：

  ```
  public interface DeferredImportSelector extends ImportSelector {
      ...
  
      interface Group {
  
          /**
           * 处理导入操作
           */
          void process(AnnotationMetadata metadata, DeferredImportSelector selector);
  
          /**
           * 返回导入类
           */
          Iterable<Entry> selectImports()
      }
  }
  
  ```

  在处理 `DeferredImportSelector` 的导入类时，`DeferredImportSelector.Group#process` 方法会先调用，然后再调用 `DeferredImportSelector.Group#selectImports` 返回导入类；

* `DeferredImportSelector` 可以指定导入类的分组，在处理时，可以按分组处理导入类；

* `DeferredImportSelector` 在处理导入类时，先将导入类按分组放入一个 `map` 中，在处理完其他配置类（spring 的配置类为 `@Component`、`@ComponentScan`、`@Import`、`@Configuration`、`@Bean` 标记的类）后再来处理分组中的导入类，也就是说，`DeferredImportSelector` 导入的类，会在其他类注册到 `beanFactory` 中后，再进行注册（注册前还需判断能否注册到 `beanFactory`，若能才注册）。

我们来看看 `AutoConfigurationImportSelector` 的代码：

```
// 实现了 DeferredImportSelector
public class AutoConfigurationImportSelector implements DeferredImportSelector, 
        BeanClassLoaderAware,ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

    ...

    /**
     * 这里实现了 DeferredImportSelector.Group
     */
    private static class AutoConfigurationGroup implements DeferredImportSelector.Group, 
            BeanClassLoaderAware, BeanFactoryAware, ResourceLoaderAware {

        /**
         * 保存导入的类
         */
        private final List<AutoConfigurationEntry> autoConfigurationEntries = new ArrayList<>();

        /**
         * 处理导入类
         */
        @Override
        public void process(AnnotationMetadata annotationMetadata, 
                DeferredImportSelector deferredImportSelector) {
            Assert.state(deferredImportSelector instanceof AutoConfigurationImportSelector,
                    () -> String.format("Only %s implementations are supported, got %s",
                            AutoConfigurationImportSelector.class.getSimpleName(),
                            deferredImportSelector.getClass().getName()));
            // 1\. 调用 AutoConfigurationImportSelector#getAutoConfigurationEntry(...) 方法，
            // 在这个方法里会加载自动装配类
            AutoConfigurationEntry autoConfigurationEntry = 
                ((AutoConfigurationImportSelector) deferredImportSelector)
                    .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);
            // 2\. 将获取到的 autoConfigurationEntry 保存起来
            this.autoConfigurationEntries.add(autoConfigurationEntry);
            for (String importClassName : autoConfigurationEntry.getConfigurations()) {
                this.entries.putIfAbsent(importClassName, annotationMetadata);
            }
        }

        /**
         * 返回导入类
         */
        @Override
        public Iterable<Entry> selectImports() {
            if (this.autoConfigurationEntries.isEmpty()) {
                return Collections.emptyList();
            }
            // 3\. 得到过滤类
            Set<String> allExclusions = this.autoConfigurationEntries.stream()
                    .map(AutoConfigurationEntry::getExclusions).flatMap(Collection::stream)
                    .collect(Collectors.toSet());
            // 4\. 将 autoConfigurationEntries 转换为 LinkedHashSet
            Set<String> processedConfigurations = this.autoConfigurationEntries.stream()
                    .map(AutoConfigurationEntry::getConfigurations).flatMap(Collection::stream)
                    .collect(Collectors.toCollection(LinkedHashSet::new));
            // 5\. 去除需要过滤的类
            processedConfigurations.removeAll(allExclusions);
            // 6\. 进行排序
            return sortAutoConfigurations(processedConfigurations, getAutoConfigurationMetadata())
                    .stream().map((importClassName) -> new Entry(
                        this.entries.get(importClassName), importClassName))
                    .collect(Collectors.toList());
        }

        ...
    }

}

```

这里我们将 `DeferredImportSelector.Group#process` 与 `DeferredImportSelector.Group#selectImports` 两个方法结合起来看，处理步骤总结如下:

1.  调用 `AutoConfigurationImportSelector#getAutoConfigurationEntry(...)` 方法加载自动装配类；
2.  将得到的自动装配类保存到 `autoConfigurationEntries` 中；
3.  得到过滤类，这些过滤类就是由 `@EnableAutoConfiguration` 的 `exclude` 或 `excludeName` 指定的；
4.  将 `autoConfigurationEntries` 转换为 `LinkedHashSet`，结果为 `processedConfigurations`；
5.  去除 `processedConfigurations` 需要过滤的类；
6.  将第 5 步得到的类排序后，返回。

接下来我们对这些关键步骤进行分析。

> 特别说明：`DeferredImportSelector` 是 `ImportSelector` 的子接口，`ImportSelector` 处理导入类的方法是 `selectImports(...)`，在 `DeferredImportSelector` 中也重写了该方法：
>
> ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-75a2839c622af2d0f374189b2e2765a64d7.png)
>
> 这个方法所做的也是加载自动装配类，返回最终导入的类，但需要注意的是，springboot 的自动导入类**不是**在这里处理的，关于这点，可以在方法内打个断点，然后就会发现这个方法并没有运行到！
>
> 最后再声明下：springboot 的自动导入类**不是**在 `AutoConfigurationImportSelector#selectImports` 方法中处理的，而是在 `AutoConfigurationImportSelector.AutoConfigurationGroup#selectImports` 方法中处理的。

### 2\. 获取装配类：`AutoConfigurationImportSelector#getAutoConfigurationEntry`

自动配置 类的加载代码为：

```
AutoConfigurationEntry autoConfigurationEntry = 
    ((AutoConfigurationImportSelector) deferredImportSelector)
        .getAutoConfigurationEntry(getAutoConfigurationMetadata(), annotationMetadata);

```

该代码就是用来加载自动装配类的，我们直接进入 `AutoConfigurationImportSelector#getAutoConfigurationEntry` 方法：

```
protected AutoConfigurationEntry getAutoConfigurationEntry(
        AutoConfigurationMetadata autoConfigurationMetadata, AnnotationMetadata annotationMetadata) {
    // 又一次判断是否开启自动装配
    if (!isEnabled(annotationMetadata)) {
        return EMPTY_ENTRY;
    }
    // 获取注解的属性
    AnnotationAttributes attributes = getAttributes(annotationMetadata);
    // 1\. 加载候选的自动配置类
    List<String> configurations = getCandidateConfigurations(annotationMetadata, attributes);
    // 2\. 去重，转换成set，再转换成list
    configurations = removeDuplicates(configurations);
    // 3\. 去除需要排除的类，其实就是处理@EnableAutoConfiguration的exclude与excludeName
    Set<String> exclusions = getExclusions(annotationMetadata, attributes);
    checkExcludedClasses(configurations, exclusions);
    configurations.removeAll(exclusions);
    // 4\. 过滤不需要自动装配的类
    configurations = filter(configurations, autoConfigurationMetadata);
    // 5\. 触发 AutoConfigurationImportEvent 事件
    fireAutoConfigurationImportEvents(configurations, exclusions);
    // 6\. 最终返回的值
    return new AutoConfigurationEntry(configurations, exclusions);
}

```

这个方法非常重要，包含了获取自动装配类的全部操作，该操作流程如下：

1. 加载候选的自动装配类，springboot 自动装配的类位于 `classpath` 下的 `META-INF/spring.factories` 文件中，key 为 `org.springframework.boot.autoconfigure.EnableAutoConfiguration`，这个我们后面再详细分析；

2. 去除重复的自动装配类，上一步加载得到的自动装配类可能会有重复，在这里会去除重复的类，去除方式也非常简单，springboot 就只是先转换成 `Set`，再转换成 `List`；

3. 去除排除的类，前面提到 `@EnableAutoConfiguration` 可以通过 `exclude` 与 `excludeName` 指定需要排除的类，这一步就是来处理这两个属性的；

4. 过滤不需要自动装配的类，根据本人调试，发现并没有完成过滤：

   过滤前是 124 个：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-773695c0161c8126f239c2e66529fc8a394.png)

   过滤后还是 124 个：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-7144d971f729b5dfbddfeff2d3319c7705c.png)

5. 触发 `AutoConfigurationImportEvent` 事件；

6. 将第 3 步得到的排除类与第 4 步得到的自动装配类包装成 `AutoConfigurationEntry` 返回。

注意最后一行代码：

```
// 6\. 最终返回的值
return new AutoConfigurationEntry(configurations, exclusions);

```

这里把 `configurations` 与 `exclusions` 都传入了 `AutoConfigurationEntry` 的构造方法，我们来看看 `AutoConfigurationEntry`：

```
protected static class AutoConfigurationEntry {
    // 自动装配类
    private final List<String> configurations;

    // 需要排除的自动装配类
    private final Set<String> exclusions;

    /**
     * 构造方法，对再者进行赋值
     */
    AutoConfigurationEntry(Collection<String> configurations, Collection<String> exclusions) {
        this.configurations = new ArrayList<>(configurations);
        this.exclusions = new HashSet<>(exclusions);
    }

    ...
}

```

由些可见，最终返回的 `AutoConfigurationEntry` 包含两大内容：

*   `configurations`：自动装配类，已经去除了需要排除的类
*   `exclusions`：通过 `@EnableAutoConfiguration` 指定的需要排除的类

整个自动装配类的获取就是这样了，下面我们来看看加载候选的自动装配类的流程。

### 3\. 加载候选的自动装配类

自动装配类的加载位于 `AutoConfigurationImportSelector#getCandidateConfigurations`，代码如下：

```
protected List<String> getCandidateConfigurations(AnnotationMetadata metadata, 
        AnnotationAttributes attributes) {
    // 调用的是 spring 提供的方法：SpringFactoriesLoader.loadFactoryNames(...)
    // getSpringFactoriesLoaderFactoryClass() 返回的是EnableAutoConfiguration
    List<String> configurations = SpringFactoriesLoader
            .loadFactoryNames(getSpringFactoriesLoaderFactoryClass(), getBeanClassLoader());
    Assert.notEmpty(configurations, "...");
    return configurations;
}

protected Class<?> getSpringFactoriesLoaderFactoryClass() {
    return EnableAutoConfiguration.class;
}

```

继续进入 `SpringFactoriesLoader#loadFactoryNames`：

```
public final class SpringFactoriesLoader {

    public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";

    public static List<String> loadFactoryNames(Class<?> factoryType, @Nullable ClassLoader classLoader) {
        // 得到的 factoryTypeName 是 org.springframework.boot.autoconfigure.EnableAutoConfiguration
        String factoryTypeName = factoryType.getName();
        return loadSpringFactories(classLoader).getOrDefault(factoryTypeName, Collections.emptyList());
    }

    /**
     * 在这里进行加载，加载的是 META-INF/spring.factories 中的属性
     */
    private static Map<String, List<String>> loadSpringFactories(@Nullable ClassLoader classLoader) {
        MultiValueMap<String, String> result = cache.get(classLoader);
        if (result != null) {
            return result;
        }

        try {
            // 加载 META-INF/spring.factories 的内容
            Enumeration<URL> urls = (classLoader != null ?
                    classLoader.getResources(FACTORIES_RESOURCE_LOCATION) :
                    ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));
            result = new LinkedMultiValueMap<>();
            while (urls.hasMoreElements()) {
                URL url = urls.nextElement();
                UrlResource resource = new UrlResource(url);
                // 将 META-INF/spring.factories 的内容转换为 Properties 对象
                Properties properties = PropertiesLoaderUtils.loadProperties(resource);
                for (Map.Entry<?, ?> entry : properties.entrySet()) {
                    String factoryTypeName = ((String) entry.getKey()).trim();
                    // StringUtils.commaDelimitedListToStringArray(...) 逗号分割为数组
                    for (String factoryImplementationName : 
                                StringUtils.commaDelimitedListToStringArray((String) entry.getValue())) {
                        result.add(factoryTypeName, factoryImplementationName.trim());
                    }
                }
            }
            cache.put(classLoader, result);
            return result;
        }
        catch (IOException ex) {
            throw new IllegalArgumentException("Unable to load factories from location [" +
                    FACTORIES_RESOURCE_LOCATION + "]", ex);
        }
    }
    ...
}

```

可以看到，这里加载的是 `classpath` 下的 `META-INF/spring.factories` 文件，注意：这个文件可能会有多个，位于不同的 jar 包中。

springboot 自带的 `META-INF/spring.factories` 位于 `spring-boot-autoconfigure` 模块下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-ace5e83645626966eae1e62a50752f2417d.png)

我们来看一眼 `spring.factories`：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d7de9ecd19345f0dc77cc304843c588fe4d.png)

这个文件定义了许多的配置类，以 `key-value` 的形式保存，多个值之间使用 “,” 分开，上面提到的自动装配类的 key 是 `org.springframework.boot.autoconfigure.EnableAutoConfiguration`，对应的 `value` 非常多，这里就不展示了。

这一步之后，自动装配类就被注册到 spring 容器中了。注：此时加载到 spring 容器中的还是 `BeanDefinition`，要想成为 spring bean，还得经过 `ConditionalOnBean`、`ConditionalOnClass` 等注解的考验，这些我们后面再分析。

### 4\. 获取自动装配类后的处理

让我们再回到 `AutoConfigurationImportSelector.AutoConfigurationGroup`，在第 1 节我们总结的流程如下：

1.  调用 `AutoConfigurationImportSelector#getAutoConfigurationEntry(...)` 方法加载自动装配类；
2.  将得到的自动装配类保存到 `autoConfigurationEntries` 中；
3.  得到过滤类，这些过滤类就是由 `@EnableAutoConfiguration` 的 `exclude` 或 `excludeName` 指定的；
4.  将 `autoConfigurationEntries` 转换为 `LinkedHashSet`，结果为 `processedConfigurations`；
5.  去除 `processedConfigurations` 需要过滤的类；
6.  将第 5 步得到的类排序后，返回。

以上第 2 节与第 3 节，分析的是自动加载类的加载过程，我们再来看看接下来的步骤。

对照着代码，我们会发现接下来的步骤都比较简单，这里也逐一说明下吧。

*   第 2 步，保存得到的自动装配类，这个操作仅仅只是调用了 `List#add(...)` 方法，将得到的 `autoConfigurationEntry` 保存到 `autoConfigurationEntries`，这个结构是 `AutoConfigurationGroup` 的成员变量，在 `AutoConfigurationImportSelector.AutoConfigurationGroup#selectImports` 方法中会用到；

*   第 3 步，得到的是所有的过滤类，该过滤类是遍历 `autoConfigurationEntries`，然后通过 `autoConfigurationEntry#getExclusions` 方法得到的 ，前面我们也提到过，`autoConfigurationEntry` 只包含两个成员变量：`configurations`(去除排除类后的自动装配类) 与 `exclusions`(通过 `@EnableAutoConfiguration` 指定的排除类)；

*   第 4 步，将 `List` 转换为 `LinkedHashSet`，不分析；

*   第 5 步，对所有的自动装配类再进行一次去除排除类的操作，排除的对象是所有的排除类，这种情况应该是会对同一项目中有多个 `@EnableAutoConfiguration` 的情况，比如第一个 `@EnableAutoConfiguration` 注解排除 `A`、`B` 两个类，第二个 `@EnableAutoConfiguration` 注解排除 `C`，`D` 两个类，那最终排除的是 `A`、`B`、`C`，`D` 四个类；

*   第 6 步，这一步的主要操作是排序，这个顺序决定了自动装配类注册到 `beanFactory` 中的顺序，`AutoConfigureOrder`、`@AutoConfigureAfter` 与 `@AutoConfigureBefore` 就是在这里处理的，关于这块内容，可以参考 [springboot 自动装配之自动装配顺序](https://my.oschina.net/funcy/blog/4921594).

经过这些步骤后，自动装配的获取就完成了。

### 5\. 自定义自动装配类

了解完自动装配类的加载过程后，我们也可以自定义一个自动装配类。

1.  准备一个自动装配类

```
@Configuration
public class MyAutoConfiguration {

    @Bean
    public Object object() {
        System.out.println("create object");
        return new Object();
    }
}

```

这个类很简单，就是一个标记了 `@Configuration` 的类，类中使用 `@Bean` 注解创建了一个 bean，在创建 bean 的过程中会 打印 "create object"。

1.  准备 `META-INF/spring.factories` 内容如下：

```
# Auto Configure
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
org.springframework.boot.learn.autoconfigure.demo01.configure.MyAutoConfiguration

```

1.  主类

```
@SpringBootApplication
public class AutoconfigureDemo01Application {

    public static void main(String[] args) {
        SpringApplication.run(AutoconfigureDemo01Application.class, args);
    }

}

```

运行结果如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-9337a7ac4ce4ff7d71e69bc952bfac30b12.png)

可以看到，`create object` 成功打印了。

那这个 `bean` 是通过包扫描创建的，还是自动装配导入的呢？我们通过调试的方式来看下自动装配得到的类：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-27bea49ddeae2f0f720ea338914cc443aec.png)

可以看到，`MyAutoConfiguration` 就在自动装配类的列表中了。

注意到，`MyAutoConfiguration` 加了 `@Configuration` 注解， 那么它究竟是由 sping 容器扫描到的，还是由自动装配得到的呢？

在[【springboot 源码分析】@SpringBootApplication 注解](https://my.oschina.net/funcy/blog/4870882)一文中，我们提到 `SpringBootApplication` 注解中的 `@ComponentScan` 会指定一个过滤器：`AutoConfigurationExcludeFilter`，这个过滤器会过滤自动装配类，这里我们看下目前为止 `beanFactory` 都有哪些 `beanName`：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1725008451f5516ab540bcc3ae13d1f37ee.png)

可以看到，并没有 `MyAutoConfiguration`，因此此时它还没被扫描进 `beanFactory` 中。

当然，我们也可以把 `MyAutoConfiguration` 上面的 `@Configuration` 注解去掉，这样就不会有这个困惑了。

### 6\. 总结

本文从 `@EnableAutoConfiguration` 注解出发，分析了自动装配类的加载流程，加载流程在 `AutoConfigurationImportSelector#getAutoConfigurationEntry` 方法中，最终加载的是 `META-INF/spring.factories` 文件中 key 是 `org.springframework.boot.autoconfigure.EnableAutoConfiguration` 的类。

得到自动装配类后，spring 会将其注册到容器中，此时它们还是一个的 `BeanDefinition`，要想成为 spring bean，还得经过 `ConditionalOnBean`、`ConditionalOnClass` 等注解的考验，这些我们后面再分析。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4870868](https://my.oschina.net/funcy/blog/4870868) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_