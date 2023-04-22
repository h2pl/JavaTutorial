### 1\. 条件注解及其判断类

在 [springboot 自动装配之加载自动装配类](https://my.oschina.net/funcy/blog/4870868)一文中，我们分析到 springboot 会加载 `META-INF/spring.factories` 文件中定义的自动装配类，加载到这些自动装配类后，这些类中的 bean 就一定会初始化吗？并不是，我们可以在对应的 Bean 生成方法上使用**条件注解**来控制类是否进行初始化！

springboot 提供的条件注解如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0e8d27c887fb6ca142672cad1c60e9de207.png)

这里列举部分如下：

| 注解类型              | 注解类型                                                     | 功能说明                                      |
| --------------------- | ------------------------------------------------------------ | --------------------------------------------- |
| class 条件注解        | `@ConditionalOnClass`/`@ConditionalOnMissingClass`           | 当指定的类**存在 / 缺失**时初始化该 bean      |
| bean 条件注解         | `@ConditionalOnBean`/`@ConditionalOnMissingBean`             | 当指定的 bean **存在 / 缺失**时初始化该 bean  |
| 属性条件注解          | `@ConditionalOnProperty`                                     | 当指定的属性存在初始化该 bean                 |
| Resource 条件注解     | `@ConditionalOnResource`                                     | 当指定的资源存在初始化该 bean                 |
| Web 应用条件注解      | `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | 当前应用**为 / 不为** web 应用时初始化该 bean |
| spring 表达式条件注解 | `@ConditionalOnExpression`                                   | 当表达式结果为 true 时初始化该 bean           |

我们进入 `@ConditionalOnClass` 看看该注解的内容：

```
...
@Conditional(OnClassCondition.class)
public @interface ConditionalOnClass {
    ...
}

```

可以看到，`ConditionalOnClass` 组合了 `@Conditional` 注解的功能，处理类是 `OnClassCondition.class`。

关于 `@Conditional` 注解可以参考 [ConfigurationClassPostProcessor 之处理 @Conditional 注解](https://my.oschina.net/funcy/blog/4873444)，这里我们直接说 `@Conditional` 的使用方式：

1. `@Conditional` 是 spring 处理的条件注解；

2. `@Conditional` 提供了一个属性 `value`，类型为 `Class`，其必须是 `Condition` 的子类：

   ```
   Class<? extends Condition>[] value();
   
   ```

3. `Condition` 是一个接口，其中有一个 `matches(...)` 方法：

   ```
   public interface Condition {
   
       boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata);
   
   }
   
   ```

   只有在 `matches(...)` 方法返回 `true` 时，`ConfigurationClassPostProcessor` 才会将其对应的 bean 注册到 `beanFactory` 的 `beanDefinitionMap` 中.

总结完 `@Conditional` 的使用方式后，我们就明白了：`OnClassCondition.class` 是 `Condition` 的子类，其 `matches(...)` 方法用来处理主要条件规则。同理，其他条件注解的处理方式也类似，这里总结下条件注解的判断类：

| 注解类型              | 注解类型                                                     | 条件判断类                  |
| --------------------- | ------------------------------------------------------------ | --------------------------- |
| class 条件注解        | `@ConditionalOnClass`/`@ConditionalOnMissingClass`           | `OnClassCondition`          |
| bean 条件注解         | `@ConditionalOnBean`/`@ConditionalOnMissingBean`             | `OnBeanCondition`           |
| 属性条件注解          | `@ConditionalOnProperty`                                     | `OnPropertyCondition`       |
| Resource 条件注解     | `@ConditionalOnResource`                                     | `OnResourceCondition`       |
| Web 应用条件注解      | `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | `OnWebApplicationCondition` |
| spring 表达式条件注解 | `@ConditionalOnExpression`                                   | `OnExpressionCondition`     |

接下来，分析目的就很明确了：要分析这些条件注解的判断逻辑，只需要分析对应条件判断类的 `matches(...)` 方法就可以了。

### 2. `SpringBootCondition#matches`

进入 `OnClassCondition#matches` 方法，发现来到的是 `SpringBootCondition`，相关方法如下：

```
public abstract class SpringBootCondition implements Condition {

    private final Log logger = LogFactory.getLog(getClass());

    @Override
    public final boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
        String classOrMethodName = getClassOrMethodName(metadata);
        try {
            // 获取条件匹配结果
            ConditionOutcome outcome = getMatchOutcome(context, metadata);
            // 打印一条日志
            logOutcome(classOrMethodName, outcome);
            // 记录条件评估的发生，简单理解为记录一条条件判断记录吧
            recordEvaluation(context, classOrMethodName, outcome);
            // 这里返回最终结果：true 或 false
            return outcome.isMatch();
        }
        catch (NoClassDefFoundError ex) {
            throw new IllegalStateException(...);
        }
        catch (RuntimeException ex) {
            throw new IllegalStateException(...);
        }
    }

    /**
     * 这是个抽象方式，具体内容由子类实现
     */
    public abstract ConditionOutcome getMatchOutcome(
        ConditionContext context, AnnotatedTypeMetadata metadata);

    ...

}

```

`SpringBootCondition` 的 `matches(...)` 关键就两行：

```
...
ConditionOutcome outcome = getMatchOutcome(context, metadata);
...
return outcome.isMatch();

```

而 `SpringBootCondition` 的 `getMatchOutcome(...)` 又是个抽象方法，具体的逻辑由子类提供，`OnClassCondition` 它的实现之一。实际上，上述条件判断类都是 `SpringBootCondition` 的子类，后面我们就直接进入具体类的 `getMatchOutcome(...)` 方法分析了。

`getMatchOutcome(...)` 方法返回的结果是 `ConditionOutcome`，接下来我们来看看 `ConditionOutcome` 是个啥：

```
public class ConditionOutcome {

    private final boolean match;

    private final ConditionMessage message;

    /**
     * 构造方法
     */
    public ConditionOutcome(boolean match, String message) {
        this(match, ConditionMessage.of(message));
    }

    /**
     * 构造方法
     */
    public ConditionOutcome(boolean match, ConditionMessage message) {
        Assert.notNull(message, "ConditionMessage must not be null");
        this.match = match;
        this.message = message;
    }

    /**
     * 返回匹配的结果
     */
    public boolean isMatch() {
        return this.match;
    }

    ...
}

```

从代码来看，这个类就是用来封装比较结果的，内部有两个属性：`match` 与 `message`:

*   `match` 的类型是 `boolean`，这个就是最终匹配成功还是失败的标识
*   `message` 的类型是 `ConditionMessage`，它表示匹配结果的说明

我们再来看看 `ConditionMessage`:

```
public final class ConditionMessage {

    private String message;

    private ConditionMessage() {
        this(null);
    }

    private ConditionMessage(String message) {
        this.message = message;
    }

    ...
}

```

它仅有一个属性：`message`，这表明它就是对说明信息的包装。

### 3. `@ConditionalOnClass`: `OnClassCondition#getMatchOutcome`

接下来我们来分析 `OnClassCondition` 的匹配逻辑，直接进入 `getMatchOutcome` 方法：

```
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    ClassLoader classLoader = context.getClassLoader();
    ConditionMessage matchMessage = ConditionMessage.empty();
    // 1\. 处理 @ConditionalOnClass 注解
    List<String> onClasses = getCandidates(metadata, ConditionalOnClass.class);
    if (onClasses != null) {
        // 1.1 处理条件判断
        List<String> missing = filter(onClasses, ClassNameFilter.MISSING, classLoader);
        if (!missing.isEmpty()) {
            // 1.2 构建返回结果：不匹配的情况
            return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnClass.class)
                    .didNotFind("required class", "required classes").items(Style.QUOTE, missing));
        }
        matchMessage = matchMessage.andCondition(ConditionalOnClass.class)
                .found("required class", "required classes")
                .items(Style.QUOTE, filter(onClasses, ClassNameFilter.PRESENT, classLoader));
    }

    // 2\. 处理 @ConditionalOnMissingClass 注解
    List<String> onMissingClasses = getCandidates(metadata, ConditionalOnMissingClass.class);
    if (onMissingClasses != null) {
        // 2.1 处理条件判断
        List<String> present = filter(onMissingClasses, ClassNameFilter.PRESENT, classLoader);
        if (!present.isEmpty()) {
            // 2.2 构建返回结果：不匹配的情况
            return ConditionOutcome.noMatch(ConditionMessage
                    .forCondition(ConditionalOnMissingClass.class)
                    .found("unwanted class", "unwanted classes").items(Style.QUOTE, present));
        }
        matchMessage = matchMessage.andCondition(ConditionalOnMissingClass.class)
                .didNotFind("unwanted class", "unwanted classes")
                .items(Style.QUOTE, filter(onMissingClasses, ClassNameFilter.MISSING, classLoader));
    }
    // 最后返回匹配的结果
    return ConditionOutcome.match(matchMessage);
}

```

这个方法同时处理了 `@ConditionalOnClass` 与 `@ConditionalOnMissingClass` 两个注解，处理流程极其相似，两个注解的条件判断都是通过 `FilteringSpringBootCondition#filter` 内容如下：

```
protected final List<String> filter(Collection<String> classNames, ClassNameFilter classNameFilter,
        ClassLoader classLoader) {
    if (CollectionUtils.isEmpty(classNames)) {
        return Collections.emptyList();
    }
    List<String> matches = new ArrayList<>(classNames.size());
    for (String candidate : classNames) {
        // 进行条件匹配
        if (classNameFilter.matches(candidate, classLoader)) {
            matches.add(candidate);
        }
    }
    return matches;
}

```

由此可见，传入的 `classNameFilter` 成了关键：

*   处理 `@ConditionalOnClass` 时，`classNameFilter` 为 `ClassNameFilter.MISSING`
*   处理 `@ConditionalOnMissingClass` 时，`classNameFilter` 为 `ClassNameFilter.PRESENT`

让我们进入 `ClassNameFilter` 一探究竟，它是 `FilteringSpringBootCondition` 的子类，内容如下：

```
abstract class FilteringSpringBootCondition extends SpringBootCondition
        implements AutoConfigurationImportFilter, BeanFactoryAware, BeanClassLoaderAware {
    ...
    /**
     * 如果 classLoader 存在，则调用 ClassLoader#loadClass 方法
     * 否则调用 Class#forName 方法
     */
    protected static Class<?> resolve(String className, ClassLoader classLoader) 
            throws ClassNotFoundException {
        if (classLoader != null) {
            return classLoader.loadClass(className);
        }
        return Class.forName(className);
    }

    /**
     * 处理条件匹配
     */
    protected enum ClassNameFilter {

        PRESENT {

            @Override
            public boolean matches(String className, ClassLoader classLoader) {
                return isPresent(className, classLoader);
            }

        },

        MISSING {

            @Override
            public boolean matches(String className, ClassLoader classLoader) {
                return !isPresent(className, classLoader);
            }

        };

        abstract boolean matches(String className, ClassLoader classLoader);

        /**
         * Class 是否存在
         * 通过捕获类加载时的异常来判断类是否存在，未抛出异常则表示类存在
         */
        static boolean isPresent(String className, ClassLoader classLoader) {
            if (classLoader == null) {
                classLoader = ClassUtils.getDefaultClassLoader();
            }
            try {
                // 通过异常捕获来判断是否存在该class
                resolve(className, classLoader);
                return true;
            }
            catch (Throwable ex) {
                return false;
            }
        }
    }
    ...
}

```

看到这里我们就明白了：判断 `Class` 是否存在，spring 是通过捕获 `ClassLoader.load(String)` 或 `Class.forName(String)` 方法的异常来处理的，如果抛出了异常就表明 `Class` 不存在。

这里总结下 `@ConditionalOnClass`/`@ConditionalOnMissingClass` 的处理方式：**两者的处理类都为 `OnClassCondition`，通过捕获 `ClassLoader.load(String)` 或 `Class.forName(String)` 方法的异常来判断 `Class` 是否存在，如果抛出了异常就表明 `Class` 不存在**。

### 4. `@ConditionalOnBean`: `OnBeanCondition#getMatchOutcome`

继续看看 `@ConditionalOnBean` 的 处理，直接进入 `OnBeanCondition#getMatchOutcome`：

```
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    ConditionMessage matchMessage = ConditionMessage.empty();
    MergedAnnotations annotations = metadata.getAnnotations();
    // 处理 @ConditionalOnBean
    if (annotations.isPresent(ConditionalOnBean.class)) {
        Spec<ConditionalOnBean> spec = new Spec<>(context, metadata, 
                annotations, ConditionalOnBean.class);
        // 处理匹配
        MatchResult matchResult = getMatchingBeans(context, spec);
        // 注意判断条件
        if (!matchResult.isAllMatched()) {
            String reason = createOnBeanNoMatchReason(matchResult);
            return ConditionOutcome.noMatch(spec.message().because(reason));
        }
        matchMessage = spec.message(matchMessage).found("bean", "beans").items(Style.QUOTE,
                matchResult.getNamesOfAllMatches());
    }

    // 处理 @ConditionalOnSingleCandidate
    if (metadata.isAnnotated(ConditionalOnSingleCandidate.class.getName())) {
        Spec<ConditionalOnSingleCandidate> spec 
                = new SingleCandidateSpec(context, metadata, annotations);
        // 处理匹配
        MatchResult matchResult = getMatchingBeans(context, spec);
        // 注意判断条件
        if (!matchResult.isAllMatched()) {
            return ConditionOutcome.noMatch(spec.message().didNotFind("any beans").atAll());
        }
        else if (!hasSingleAutowireCandidate(context.getBeanFactory(), 
                matchResult.getNamesOfAllMatches(), spec.getStrategy() == SearchStrategy.ALL)) {
            return ConditionOutcome.noMatch(spec.message().didNotFind("a primary bean from beans")
                    .items(Style.QUOTE, matchResult.getNamesOfAllMatches()));
        }
        matchMessage = spec.message(matchMessage).found("a primary bean from beans")
                .items(Style.QUOTE, matchResult.getNamesOfAllMatches());
    }

    // 处理 @ConditionalOnMissingBean
    if (metadata.isAnnotated(ConditionalOnMissingBean.class.getName())) {
        Spec<ConditionalOnMissingBean> spec = new Spec<>(context, metadata, annotations,
                ConditionalOnMissingBean.class);
        // 处理匹配
        MatchResult matchResult = getMatchingBeans(context, spec);
        // 注意判断条件
        if (matchResult.isAnyMatched()) {
            String reason = createOnMissingBeanNoMatchReason(matchResult);
            return ConditionOutcome.noMatch(spec.message().because(reason));
        }
        matchMessage = spec.message(matchMessage).didNotFind("any beans").atAll();
    }
    return ConditionOutcome.match(matchMessage);
}

```

可以看到，这个方法一共处理了两个注解的条件匹配：`@ConditionalOnBean`、`@ConditionalOnSingleCandidate` 与 `@ConditionalOnMissingBean`，三者都调用了同一个方法 `getMatchingBeans(...)` 来获取匹配结果，然后使用 `matchResult.isAllMatched()` 或 `matchResult.isAnyMatched()` 来做最终的结果判断。

#### `OnBeanCondition#getMatchingBeans`

`getMatchingBeans(...)` 的代码如下：

```
protected final MatchResult getMatchingBeans(ConditionContext context, Spec<?> spec) {
    ClassLoader classLoader = context.getClassLoader();
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    boolean considerHierarchy = spec.getStrategy() != SearchStrategy.CURRENT;
    Set<Class<?>> parameterizedContainers = spec.getParameterizedContainers();
    if (spec.getStrategy() == SearchStrategy.ANCESTORS) {
        BeanFactory parent = beanFactory.getParentBeanFactory();
        Assert.isInstanceOf(ConfigurableListableBeanFactory.class, parent,
                "Unable to use SearchStrategy.ANCESTORS");
        beanFactory = (ConfigurableListableBeanFactory) parent;
    }
    MatchResult result = new MatchResult();
    // 1\. 获取 ignoreType，只有 @ConditionalOnMissingBean 有这个属性
    Set<String> beansIgnoredByType = getNamesOfBeansIgnoredByType(classLoader, beanFactory, 
            considerHierarchy, spec.getIgnoredTypes(), parameterizedContainers);

    // 2\. 处理 types
    for (String type : spec.getTypes()) {
        Collection<String> typeMatches = getBeanNamesForType(classLoader, considerHierarchy, 
                beanFactory, type, parameterizedContainers);
        typeMatches.removeAll(beansIgnoredByType);
        if (typeMatches.isEmpty()) {
            result.recordUnmatchedType(type);
        }
        else {
            result.recordMatchedType(type, typeMatches);
        }
    }

    // 3\. 处理类上的注解 @ConditionalOnMissingBean 有这个属性
    for (String annotation : spec.getAnnotations()) {
        Set<String> annotationMatches = getBeanNamesForAnnotation(classLoader, beanFactory, 
                annotation, considerHierarchy);
        annotationMatches.removeAll(beansIgnoredByType);
        if (annotationMatches.isEmpty()) {
            result.recordUnmatchedAnnotation(annotation);
        }
        else {
            result.recordMatchedAnnotation(annotation, annotationMatches);
        }
    }

    // 4\. 处理 beanName
    for (String beanName : spec.getNames()) {
        if (!beansIgnoredByType.contains(beanName) && containsBean(beanFactory, beanName, 
                considerHierarchy)) {
            result.recordMatchedName(beanName);
        }
        else {
            result.recordUnmatchedName(beanName);
        }
    }
    return result;
}

```

需要说明的是，这个方法会处理 3 个注解的匹配规则：`@ConditionalOnBean`、`@ConditionalOnSingleCandidate` 与 `@ConditionalOnMissingBean`，处理步骤如下：

1.  获取 `ignoreType`，只有 `@ConditionalOnMissingBean` 有这个属性
2.  处理 `types` 的匹配规则
3.  处理注解（类上的注解）的匹配规则， 只有 `@ConditionalOnMissingBean` 有这个属性
4.  处理 `beanName` 的匹配规则

关于以上步骤的具体细节，本文就不具体展开了，这里仅提供流程：

1.  获取 `ignoreType`：
    1.  使用 `ListableBeanFactory#getBeanNamesForType(Class, boolean, boolean)` 方法获取容器中所有的 `ignoreType` 的 `beanName`
    2.  结果为 `beansIgnoredByType`(类型是 `Set<String>`)
2.  处理 `types` 的匹配规则
    1.  使用 `ListableBeanFactory#getBeanNamesForType(Class, boolean, boolean)` 方法获取容器中所有的 `type` 对应的 `beanName`，结果为 `typeMatches`
    2.  将 `typeMatches` 中的值去除 `ignoreType`
    3.  判断第二步得到的 `typeMatches`，如果内容为空，将当前 `Type` 保存到 `unmatchedTypes` 中，否则保存到 `matchedTypes` 与 `namesOfAllMatches` 中
3.  处理注解的匹配规则
    1.  使用 `ListableBeanFactory#getBeanNamesForAnnotation` 方法获取容器中所有的 `annotation` 对应的 `beanName`，结果为 `annotationMatches`
    2.  将 `annotationMatches` 中的值去除 `ignoreType`
    3.  判断第二步得到的 `annotationMatches`，如果内容为空，将当前 `Annotation` 保存到 `unmatchedAnnotations` 中，否则保存到 `matchedAnnotations` 与 `namesOfAllMatches` 中
4.  处理 `beanName` 的匹配规则
    1.  判断 `beansIgnoredByType` 是否包含 `beanName`
    2.  使用 `BeanFactory#containsBean` 方法判断容器中有该 `beanName`
    3.  如果第 2 步结果为 `false`，第二步结果为 `true`，则将当前 `beanName` 加入到 `matchedNames` 与 `namesOfAllMatches`，否则保存到 `unmatchedNames` 中

得到 `matchedTypes`、`unmatchedNames` 等内容后，`matchResult.isAllMatched()` 或 `matchResult.isAnyMatched()` 最终的判断结果就是判断这些结构是否空：

```
boolean isAllMatched() {
    return this.unmatchedAnnotations.isEmpty() && this.unmatchedNames.isEmpty()
            && this.unmatchedTypes.isEmpty();
}

boolean isAnyMatched() {
    return (!this.matchedAnnotations.isEmpty()) || (!this.matchedNames.isEmpty())
            || (!this.matchedTypes.isEmpty());
}

```

看来，`@ConditionalOnBean`/`@ConditionalOnMissingBean` 的关键，就是使用 `ListableBeanFactory#getBeanNamesForType` 或 `BeanFactory#containsBean` 来判断 `beanName`、`beanType` 是否存在了。

在使用 `@ConditionalOnBean`/`@ConditionalOnMissingBean` 时，有一个坑需要特别注意：条件注解的执行时机是在 spring 的 `ConfigurationClassPostProcessor` 中的，确切地说，是在将 `bean` 加入到 `beanFactory` 的 `beanDefinitionMap` 之前判断的，如果满足条件则添加到 `beanDefinitionMap` 中，否则就不添加。这样就导致了一个问题：如果在 `@ConditionalOnBean`/`@ConditionalOnMissingBean` 的 `bean` 在该 `bean` 之后加入到 `beanDefinitionMap` 中，就有可能出现误判，举例说明：

现在有两个类：

```
@Component
@ConditionalOnMissingBean("b")
public class A {

}

@Component
public class B {

}

```

其中 `A` 与 `B` 都添加了 `@Component`，表明这是 spring bean，然后在 `A` 上添加了注解 `@ConditionalOnMissingBean("b")`，表明在 `b` 不存在时，`A` 才进行初始化。有了这些前提，我们再来看看两种情况：

1.  如果 `b` 先添加到 `beanDefinitionMap` 中，在将 `a` 添加到 `beanDefinitionMap` 时，发现 `b` 已经存在了，于是就不添加了，符合我们的预期；
2.  如果 `a` 先被处理，在添加时，发现 `beanDefinitionMap` 中并没有 `b`，于是 `a` 被添加到 `beanDefinitionMap` 中，再处理 `b`，`b` 也会被添加到 `beanDefinitionMap`，这样一来，`a` 与 `b` 同时存在于 `beanDefinitionMap` 中，最终都会被初始化成 spring bean，这与我们的预期不符。

那么 springboot 如何解决以上问题呢？我们来看看 `@ConditionalOnBean`/`@ConditionalOnMissingBean` 的说明：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-30df67cb01c73a6b201695298aad14fd0a5.png)

稍微翻译如下：

该条件只能匹配到目前为止的应用程序上下文中的 bean 存在情况，因此，强烈建议仅于自动配置类中使用。如果候选 bean 要在另一种自动配置下创建，请确保使用此条件的配置在此之后运行。

对以上内容，我的解读如下：

*   被 `@ConditionalOnBean`/`@ConditionalOnMissingBean` 标记 `bean` 加入到 `beanDefinitionMap` 那一刻，仅匹配目前为止 `beanDefinitionMap` 中已存在的 bean，对之后加入的 bean 不考虑，这就有可能造成误判，可以参考上面举的 `a` 与 `b` 的例子
*   强烈建议仅在自动配置类中使用 `@ConditionalOnBean`/`@ConditionalOnMissingBean` 这两个注解，也就是说在自动配置类中使用的话，能正确处理匹配
*   还是拿上面的 `a` 与 `b` 举例，如果 `a` 与 `b` 分别位于不同的自动配置类中，那么 `a` 需要在 `b` 之后加载到 `beanDefinitionMap` 中，这个可以通过 `@AutoConfigureAfter`、`@AutoConfigureBefore`、`@AutoConfigureOrder` 等注解来指定

关于自动配置类的加载顺序，后面再做分析吧。

限于篇幅，本文就先到这里了，下篇继续分析剩下的条件注解。

* * *

本文是 springboot 条件注解分析的第二篇，上文我们总结了 springboot 的几个条件总结：

| 注解类型              | 注解类型                                                     | 条件判断类                  |
| --------------------- | ------------------------------------------------------------ | --------------------------- |
| class 条件注解        | `@ConditionalOnClass`/`@ConditionalOnMissingClass`           | `OnClassCondition`          |
| bean 条件注解         | `@ConditionalOnBean`/`@ConditionalOnMissingBean`             | `OnBeanCondition`           |
| 属性条件注解          | `@ConditionalOnProperty`                                     | `OnPropertyCondition`       |
| Resource 条件注解     | `@ConditionalOnResource`                                     | `OnResourceCondition`       |
| Web 应用条件注解      | `@ConditionalOnWebApplication` / `@ConditionalOnNotWebApplication` | `OnWebApplicationCondition` |
| spring 表达式条件注解 | `@ConditionalOnExpression`                                   | `OnExpressionCondition`     |

本文继续分析条件判断。

### 5. `@ConditionalOnProperty`：`OnPropertyCondition#getMatchOutcome`

我们再来看看 `@ConditionalOnProperty` 的处理，进入 `OnPropertyCondition#getMatchOutcome` 方法：

```
class OnPropertyCondition extends SpringBootCondition {

    @Override
    public ConditionOutcome getMatchOutcome(ConditionContext context, 
            AnnotatedTypeMetadata metadata) {
        // 获取 @ConditionalOnProperty 的属性值
        List<AnnotationAttributes> allAnnotationAttributes = annotationAttributesFromMultiValueMap(
                metadata.getAllAnnotationAttributes(ConditionalOnProperty.class.getName()));
        List<ConditionMessage> noMatch = new ArrayList<>();
        List<ConditionMessage> match = new ArrayList<>();
        for (AnnotationAttributes annotationAttributes : allAnnotationAttributes) {
            // 在 determineOutcome(...) 方法中进行判断，注意参数：context.getEnvironment()
            ConditionOutcome outcome = determineOutcome(annotationAttributes, 
                    context.getEnvironment());
            (outcome.isMatch() ? match : noMatch).add(outcome.getConditionMessage());
        }
        if (!noMatch.isEmpty()) {
            return ConditionOutcome.noMatch(ConditionMessage.of(noMatch));
        }
        return ConditionOutcome.match(ConditionMessage.of(match));
    }

    ...

}

```

这个方法还是比较简单的，先是获取 `@ConditionalOnProperty` 的属性值，再调用 `determineOutcome(...)` 方法进行处理，让我们再进行 `OnPropertyCondition#determineOutcome` 方法：

```
/**
 * 处理结果
 * 注意：resolver 传入的的是 Environment，这就是 applicationContext 中的 Environment
 */
private ConditionOutcome determineOutcome(AnnotationAttributes annotationAttributes, 
        PropertyResolver resolver) {
    Spec spec = new Spec(annotationAttributes);
    List<String> missingProperties = new ArrayList<>();
    List<String> nonMatchingProperties = new ArrayList<>();
    // 处理操作
    spec.collectProperties(resolver, missingProperties, nonMatchingProperties);
    // 判断结果
    if (!missingProperties.isEmpty()) {
        return ConditionOutcome.noMatch(ConditionMessage
            .forCondition(ConditionalOnProperty.class, spec)
            .didNotFind("property", "properties").items(Style.QUOTE, missingProperties));
    }
    // 判断结果
    if (!nonMatchingProperties.isEmpty()) {
        return ConditionOutcome.noMatch(ConditionMessage
            .forCondition(ConditionalOnProperty.class, spec)
            .found("different value in property", "different value in properties")
            .items(Style.QUOTE, nonMatchingProperties));
    }
    // 判断结果
    return ConditionOutcome.match(ConditionMessage
        .forCondition(ConditionalOnProperty.class, spec).because("matched"));
}

/**
 * 处理属性
 */
private void collectProperties(PropertyResolver resolver, List<String> missing, 
        List<String> nonMatching) {
    for (String name : this.names) {
        String key = this.prefix + name;
        // resolver 传入的 environment
        // properties 条件判断就是判断 environment 里有没有相应属性
        if (resolver.containsProperty(key)) {
            if (!isMatch(resolver.getProperty(key), this.havingValue)) {
                nonMatching.add(name);
            }
        }
        else {
            if (!this.matchIfMissing) {
                missing.add(name);
            }
        }
    }
}

```

可以看到，`@ConditionalOnProperty` 最终是通过判断 `environment` 中是否有该属性来处理条件判断的。

### 6. `@ConditionalOnResource`：`OnResourceCondition#getMatchOutcome`

我们再来看看 `@ConditionalOnResource` 的处理，一般我们这样使用：

```
@Bean
@ConditionalOnResource(resources = "classpath:config.properties")
public Config config() {
    return config;
}

```

表示当 `classpath` 中存在 `config.properties` 时，`config` 才会被初始化 springbean。

再进入 `OnResourceCondition#getOutcomes` 方法：

```
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context, AnnotatedTypeMetadata metadata) {
    MultiValueMap<String, Object> attributes = metadata
            .getAllAnnotationAttributes(ConditionalOnResource.class.getName(), true);
    // 获取 ResourceLoader
    ResourceLoader loader = context.getResourceLoader();
    List<String> locations = new ArrayList<>();
    collectValues(locations, attributes.get("resources"));
    Assert.isTrue(!locations.isEmpty(),
            "@ConditionalOnResource annotations must specify at least one resource location");
    List<String> missing = new ArrayList<>();
    // 遍历判断资源是否存在
    for (String location : locations) {
        // location 中可能有占位符，在这里处理
        String resource = context.getEnvironment().resolvePlaceholders(location);
        // 判断 resource 是否存在
        if (!loader.getResource(resource).exists()) {
            missing.add(location);
        }
    }
    // 处理结果
    if (!missing.isEmpty()) {
        return ConditionOutcome.noMatch(ConditionMessage.forCondition(ConditionalOnResource.class)
                .didNotFind("resource", "resources").items(Style.QUOTE, missing));
    }
    return ConditionOutcome.match(ConditionMessage.forCondition(ConditionalOnResource.class)
            .found("location", "locations").items(locations));
}

```

先是通过 `OnResourceCondition#getOutcomes` 方法来获取 `ResourceLoader`，通过调试方式发现当前的 `ResourceLoader` 为 `AnnotationConfigServletWebServerApplicationContext`：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8f1de99f757eca7aeb0307b06b28d1020d2.png)

获取到 `ResourceLoader` 后，调用 `ResourceLoader#getResource(String)` 来获取资源，然后调用 `Resource#exists` 来判断资源是否存在，最后处理匹配结果。

整个流程的关键是在 `ResourceLoader#getResource(String)`，我们来看看该方法的处理，进入到 `GenericApplicationContext#getResource` 方法：

```
@Override
public Resource getResource(String location) {
    if (this.resourceLoader != null) {
        return this.resourceLoader.getResource(location);
    }
    return super.getResource(location);
}

```

这里的 `this.resourceLoader` 为 `null`，进入父类的方法 `DefaultResourceLoader#getResource`：

```
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");
    for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
        Resource resource = protocolResolver.resolve(location, this);
        if (resource != null) {
            return resource;
        }
    }
    // 处理/开头的资源
    if (location.startsWith("/")) {
        return getResourceByPath(location);
    }
    else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        // 处理classpath开头的资源
        return new ClassPathResource(
            location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
        try {
            // 以上都不满足，使用 url 来解析
            URL url = new URL(location);
            return (ResourceUtils.isFileURL(url) 
                ? new FileUrlResource(url) : new UrlResource(url));
        }
        catch (MalformedURLException ex) {
            // url解析出了问题，最终还是用 getResourceByPath(...) 来解析
            return getResourceByPath(location);
        }
    }
}

/**
 * 通过路径得到 Resource
 */
protected Resource getResourceByPath(String path) {
    return new ClassPathContextResource(path, getClassLoader());
}

```

可以看到，`DefaultResourceLoader#getResource` 通过判断 `location` 的前缀，得到了 4 种 `Resource`：

*   `ClassPathContextResource`
*   `FileUrlResource`
*   `UrlResource`

得到 `Resource` 后，接着就是判断该 `Resource` 是否存在了，我们先来看看 `ClassPathContextResource#exist` 方法，该方法在 `ClassPathResource#exists`：

```
/**
 * 判断 Resource 是否存在
 */
@Override
public boolean exists() {
    return (resolveURL() != null);
}

/**
 * 资源能获取到，则返回资源对应的url，否则返回null
 */
@Nullable
protected URL resolveURL() {
    if (this.clazz != null) {
        // 使用当前的 class 对应的 classLoader 来获取
        return this.clazz.getResource(this.path);
    }
    else if (this.classLoader != null) {
        // 使用指定的 classLoader 来获取
        return this.classLoader.getResource(this.path);
    }
    else {
        // 获取系统类加载器获取
        return ClassLoader.getSystemResource(this.path);
    }
}

```

从代码可以看到，最终是通过 `classLoader` 获取文件的 `url`，通过判断文件 `url` 是否为 `null` 来判断 `resource` 是否存在。

再来看看 `FileUrlResource` 的判断，实际上 `FileUrlResource` 与 `UrlResource` 的 `exist()` 方法都是 `AbstractFileResolvingResource#exists`，这里统一分析就可以了，该方法内容如下：

```
public boolean exists() {
    try {
        URL url = getURL();
        if (ResourceUtils.isFileURL(url)) {
            // 如果是文件，直接判断文件是否存在
            return getFile().exists();
        }
        else {
            // 否则使用网络文件来处理
            URLConnection con = url.openConnection();
            customizeConnection(con);
            HttpURLConnection httpCon =
                    (con instanceof HttpURLConnection ? (HttpURLConnection) con : null);
            // 如果是http，则判断看看链接返回的状态码
            if (httpCon != null) {
                int code = httpCon.getResponseCode();
                if (code == HttpURLConnection.HTTP_OK) {
                    return true;
                }
                else if (code == HttpURLConnection.HTTP_NOT_FOUND) {
                    return false;
                }
            }
            // 连接 contentLengthLong 大于0，也当成是true
            if (con.getContentLengthLong() > 0) {
                return true;
            }
            if (httpCon != null) {
                httpCon.disconnect();
                return false;
            }
            else {
                getInputStream().close();
                return true;
            }
        }
    }
    catch (IOException ex) {
        return false;
    }
}

```

如果是本地文件，直接使用 `File#exists()` 方法判断文件是否存在，否则就判断网络文件是否存在，判断方式这里就不细说了。

总的来说，springboot 对 `@ConditionalOnResource` 的判断还是有些复杂的，这里总结如下：

1.  如果是 `classpath` 文件，通过 `classloader` 获取文件对应的 `url` 是否为 `null` 来判断文件是否存在；
2.  如果是普通文件，则直接 `File#exists()` 方法判断文件是否存在；
3.  如果是网络文件，先打开一个网络连接，判断文件是否存在。

### 7. `@ConditionalOnWebApplication`：`OnWebApplicationCondition#getMatchOutcome`

我们再来看看 `@ConditionalOnWebApplication` 的处理，进入 `OnWebApplicationCondition#getOutcomes` 方法：

```
@Override
protected ConditionOutcome[] getOutcomes(String[] autoConfigurationClasses,
        AutoConfigurationMetadata autoConfigurationMetadata) {
    ConditionOutcome[] outcomes = new ConditionOutcome[autoConfigurationClasses.length];
    for (int i = 0; i < outcomes.length; i++) {
        String autoConfigurationClass = autoConfigurationClasses[i];
        if (autoConfigurationClass != null) {
            // 处理结果
            outcomes[i] = getOutcome(autoConfigurationMetadata.get(autoConfigurationClass, 
                "ConditionalOnWebApplication"));
        }
    }
    return outcomes;
}

/**
 * 处理结果
 * springboot支持的web类型有两种：SERVLET，REACTIVE
 */
private ConditionOutcome getOutcome(String type) {
    if (type == null) {
        return null;
    }
    ConditionMessage.Builder message = ConditionMessage
            .forCondition(ConditionalOnWebApplication.class);
    // 如果指定的类型是 SERVLET
    if (ConditionalOnWebApplication.Type.SERVLET.name().equals(type)) {
        if (!ClassNameFilter.isPresent(SERVLET_WEB_APPLICATION_CLASS, getBeanClassLoader())) {
            return ConditionOutcome.noMatch(
                message.didNotFind("servlet web application classes").atAll());
        }
    }
    // 如果指定的类型是 REACTIVE
    if (ConditionalOnWebApplication.Type.REACTIVE.name().equals(type)) {
        if (!ClassNameFilter.isPresent(REACTIVE_WEB_APPLICATION_CLASS, getBeanClassLoader())) {
            return ConditionOutcome.noMatch(
                message.didNotFind("reactive web application classes").atAll());
        }
    }
    // 如果没有指定web类型
    if (!ClassNameFilter.isPresent(SERVLET_WEB_APPLICATION_CLASS, getBeanClassLoader())
            && !ClassUtils.isPresent(REACTIVE_WEB_APPLICATION_CLASS, getBeanClassLoader())) {
        return ConditionOutcome.noMatch(
            message.didNotFind("reactive or servlet web application classes").atAll());
    }
    return null;
}

```

这个方法很简单，处理逻辑为：根据 `@ConditionalOnWebApplication` 中指定的类型，判断对应的类是否存在，判断方式与 `@ConditionalOnClass` 判断类是否存在一致，而两种类型对应的类如下：

*   Servlet：`org.springframework.web.context.support.GenericWebApplicationContext`
*   Reactive：`org.springframework.web.reactive.HandlerResult`

### 8. `@ConditionalOnExpression`：`OnExpressionCondition#getMatchOutcome`

我们再来看看 `@ConditionalOnExpression` 的处理，进入 `OnExpressionCondition#getOutcomes` 方法：

```
/**
 * 处理匹配结果
 */
@Override
public ConditionOutcome getMatchOutcome(ConditionContext context, 
        AnnotatedTypeMetadata metadata) {
    // 获取表达式
    String expression = (String) metadata.getAnnotationAttributes(
            ConditionalOnExpression.class.getName()).get("value");
    expression = wrapIfNecessary(expression);
    ConditionMessage.Builder messageBuilder = ConditionMessage
            .forCondition(ConditionalOnExpression.class, "(" + expression + ")");
    // 处理占位符
    expression = context.getEnvironment().resolvePlaceholders(expression);
    ConfigurableListableBeanFactory beanFactory = context.getBeanFactory();
    if (beanFactory != null) {
        // 计算表达式的值
        boolean result = evaluateExpression(beanFactory, expression);
        return new ConditionOutcome(result, messageBuilder.resultedIn(result));
    }
    return ConditionOutcome.noMatch(messageBuilder.because("no BeanFactory available."));
}

/**
 * 计算表达式的值
 */
private Boolean evaluateExpression(ConfigurableListableBeanFactory beanFactory, 
        String expression) {
    BeanExpressionResolver resolver = beanFactory.getBeanExpressionResolver();
    if (resolver == null) {
        resolver = new StandardBeanExpressionResolver();
    }
    // 在这里解析表达式的值
    BeanExpressionContext expressionContext = new BeanExpressionContext(beanFactory, null);
    Object result = resolver.evaluate(expression, expressionContext);
    return (result != null && (boolean) result);
}

```

可以看到，springboot 最终是通过 `BeanExpressionResolver#evaluate` 方法来计算表达式结果，关于 spring 表达式，本文就不展开分析了。

好了，spring 条件注解的分析就到这里了，需要说明的是，springboot 还 有其他条件注解：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0e8d27c887fb6ca142672cad1c60e9de207.png)

这些注解的判断方式与本文的方式相类似，就不一一进行分析了。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4921590](https://my.oschina.net/funcy/blog/4921590) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_