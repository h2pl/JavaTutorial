在 [spring 事务之认识事务组件](https://my.oschina.net/funcy/blog/4773454)一文中，我们通过一个 demo 演示了如何使用 spring 事务管理功能，然后分析了 `@EnableTransactionManagement` 注解的功能，本文将继续分析 spring 事务相关代码。

### 1\. 代理对象创建流程

spring 事务管理功能是基于 aop 的，使用代理对象来进行事务的一系列操作，本文将通过调试的方式来分析代理对象的创建过程。

在 [spring 事务之认识事务组件](https://my.oschina.net/funcy/blog/4773454)中我们通过分析 `@EnableTransactionManagement` 注解，发现该注解会向 spring 容器中注册 `InfrastructureAdvisorAutoProxyCreator`，这个类是 `AbstractAdvisorAutoProxyCreator` 的子类，用来生成代理对象的，本节将基于 `InfrastructureAdvisorAutoProxyCreator` 来分析对象的创建过程。

> 关于 `AbstractAdvisorAutoProxyCreator` 的分析以及代理对象的生成，在 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（上）](https://my.oschina.net/funcy/blog/4678817) 与 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)已经作了详细分析，这里我们主要分析与 aop 有差异的地方，需要详细了解 spring aop 代理对象如何产生的小伙伴，可以阅读上述两篇文章。

我们依旧是进入 `AbstractAutoProxyCreator#postProcessBeforeInitialization`：

```
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    ...
    if (...) {
        //1\. shouldSkip:
        // - AspectJAwareAdvisorAutoProxyCreator 的 shouldSkip 方法会处理 @Aspect 注解的类，
        //   将其中的@Before/@After/@Around等注解包装为Advisor，再调用父类(也就是
        //   AbstractAutoProxyCreator)的shouldSkip方法
        // - InfrastructureAdvisorAutoProxyCreator直接执行AbstractAutoProxyCreator的shouldSkip方法
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }
    if(...)  {
        ...

        // 2\. getAdvicesAndAdvisorsForBean：获取适用用于当前对象的advisor
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(
            beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        ...
        return proxy;
    }
    return null;
}

```

这个方法有两处不同点，代码中已经注明了其中的差异，关于 `shouldSkip`，没啥好说的，我们重点展开 `getAdvicesAndAdvisorsForBean(...)` 方法。

#### 1.1 `BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans`

一路跟进 `getAdvicesAndAdvisorsForBean(...)` 方法，其中的操作与 `AspectJAwareAdvisorAutoProxyCreator` 的操作并不太大区别，不过有个方法个人认为需要强调下，方法如下：

> BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans

```
public List<Advisor> findAdvisorBeans() {
    String[] advisorNames = this.cachedAdvisorBeanNames;
    if (advisorNames == null) {
        // 查找当前beanFactory中所有 Advisor 的 bean class
        // Advisor可以是用户实现Advisor相关接口，也可以是xml指定的
        advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                this.beanFactory, Advisor.class, true, false);
        this.cachedAdvisorBeanNames = advisorNames;
    }
    ...
    List<Advisor> advisors = new ArrayList<>();
    for (String name : advisorNames) {
        ...
        // 根据advisor的bean name，从spring容器中获取 bean
        advisors.add(this.beanFactory.getBean(name, Advisor.class));
        ...
    }
    ...
    return advisors;
}

```

这个方法主要的作用是获取 spring 容器中的所有 `advisor`，其实在 `AnnotationAwareAspectJAutoProxyCreator` 中也是这么获取的，只不过在获取前，`AnnotationAwareAspectJAutoProxyCreator` 会在 `shouldSkip(...)` 方法中把 `@Aspect` 类中包含 `@Befor/@After/@Around` 等注解的方法包装成对应的 `Advisor`，而 `InfrastructureAdvisorAutoProxyCreator` 则不会，本节一开始也提到过了。

在 [spring 事务之认识事务组件](https://my.oschina.net/funcy/blog/4773454)一文中，分析 `@EnableTransactionManagement` 注解功能时，我们分析到该注解会通过 `@Bean` 注解向 spring 中引入 `BeanFactoryTransactionAttributeSourceAdvisor`，这个 bean 就会在 `BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans` 被获取到。

#### 1.2 `AopUtils#canApply(...)`

跟着方法一路往下走，接着就来到了判断 `advisor` 能否适用于目标 `class` 的地方了：

```
/**
 * 判断advisor能否适用于目标class
 */
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
    ...
    // 判断是否为 PointcutAdvisor，处理事务的advisor为BeanFactoryTransactionAttributeSourceAdvisor，
    // 它实现了PointcutAdvisor，因此下面的代码会执行
    else if (advisor instanceof PointcutAdvisor) {
        PointcutAdvisor pca = (PointcutAdvisor) advisor;
        //使用 PointcutAdvisor 继续判断
        return canApply(pca.getPointcut(), targetClass, hasIntroductions);
    }
    ...
}

/**
 * 判断advisor能否适用于目标class
 */
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
    Assert.notNull(pc, "Pointcut must not be null");
    //1\. 切点上是否存在排除类的配置
    if (!pc.getClassFilter().matches(targetClass)) {
        return false;
    }
    // 获取方法匹配对象，MethodMatcher.TRUE 为默认的 MethodMatcher 对象
    MethodMatcher methodMatcher = pc.getMethodMatcher();
    if (methodMatcher == MethodMatcher.TRUE) {
        return true;
    }
    ...
    // classes包含targetClass、除Object的所有父类、所有接口
    Set<Class<?>> classes = new LinkedHashSet<>();
    // 省略获取targetClass的父类步骤
    ...
    for (Class<?> clazz : classes) {
        // 获取 clazz 定义的方法，包括当前类的方法、除Object外的所有父类方法、接口的默认方法
        Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
        for (Method method : methods) {
            // 2\. 匹配的关键在这里
            if (introductionAwareMethodMatcher != null ?
                    introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
                    methodMatcher.matches(method, targetClass)) {
                return true;
            }
        }
    }
    return false;
}

```

这一块的代码与 `AnnotationAwareAspectJAutoProxyCreator` 一模一样，他们都是调用同样的方法来判断，但由于传入的的 `advisor` 不同，最终调用到的具体的匹配规则也不相同。

从代码的分析来看，匹配的逻辑都在 `Pointcut` 中，而 `Pointcut` 又来自于 `Advisor`，可见 `Advisor` 十分关键。处理事务的 `Advisor` 为 `BeanFactoryTransactionAttributeSourceAdvisor`，接下来我们就来分析这个类。

#### 1.3 `BeanFactoryTransactionAttributeSourceAdvisor` 匹配规则分析

从上一小 节的分析中，我们知道判断 `targetClass` 能否应用当前 `advisor` 的规则来源于 `advisor` 的 `pointcut`，`pointcut` 有两个地方包含了判断规则：

*   匹配类：`pc.getClassFilter().matches(targetClass)`
*   匹配方法：`pc.getMethodMatcher().matches(method, targetClass)`

这一小 节我们从 `BeanFactoryTransactionAttributeSourceAdvisor` 入手，一步步分析匹配规则。

```
public class BeanFactoryTransactionAttributeSourceAdvisor 
        extends AbstractBeanFactoryPointcutAdvisor {

    @Nullable
    private TransactionAttributeSource transactionAttributeSource;

    /**
     * 这个就是 pointcut
     */
    private final TransactionAttributeSourcePointcut pointcut = 
            new TransactionAttributeSourcePointcut() {
        @Override
        @Nullable
        protected TransactionAttributeSource getTransactionAttributeSource() {
            return transactionAttributeSource;
        }
    };

    /**
     * 设置 transactionAttributeSource
     */
    public void setTransactionAttributeSource(TransactionAttributeSource 
            transactionAttributeSource) {
        this.transactionAttributeSource = transactionAttributeSource;
    }

    /**
     * 设置 ClassFilter
     */
    public void setClassFilter(ClassFilter classFilter) {
        this.pointcut.setClassFilter(classFilter);
    }

    /**
     * 获取 pointcut
     */
    @Override
    public Pointcut getPointcut() {
        return this.pointcut;
    }
}

```

上面的代码关键部分已经注释了，这里来总结下：`BeanFactoryTransactionAttributeSourceAdvisor#getPointcut` 得到的 `pointcut` 为 `TransactionAttributeSourcePointcut`，就是在 `private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {...}` 中创建的。

`BeanFactoryTransactionAttributeSourceAdvisor` 的 `transactionAttributeSource` 是什么呢？回忆下 `ProxyTransactionManagementConfiguration` 中创建 `transactionAdvisor` 的代码：

```
public class ProxyTransactionManagementConfiguration 
        extends AbstractTransactionManagementConfiguration {

    // 省略其他
    ...

    /**
     * 读取Spring的 @Transactional 注解，并将相应的事务属性公开给Spring的事务基础结构
     */
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource();
    }

    /**
     * 事务增强器.
     * transactionAttributeSource：transactionAttributeSource() 返回的对象
     */
    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
            TransactionAttributeSource transactionAttributeSource,
            TransactionInterceptor transactionInterceptor) {
        BeanFactoryTransactionAttributeSourceAdvisor advisor = 
                new BeanFactoryTransactionAttributeSourceAdvisor();
        // 设置事务属性类，用来保存 @Transactional 的属性
        advisor.setTransactionAttributeSource(transactionAttributeSource);
        ...
        return advisor;
    }

}

```

由此可知，`BeanFactoryTransactionAttributeSourceAdvisor` 的 `transactionAttributeSource` 属性为 `AnnotationTransactionAttributeSource`.

我们再回到 `BeanFactoryTransactionAttributeSourceAdvisor`，从上面的分析可知，`getPointcut()` 得到的是 `TransactionAttributeSourcePointcut` 对象，然后进入这个类：

```
abstract class TransactionAttributeSourcePointcut 
        extends StaticMethodMatcherPointcut implements Serializable {

    protected TransactionAttributeSourcePointcut() {
        // 在构造方法中设置 ClassFilter
        setClassFilter(new TransactionAttributeSourceClassFilter());
    }

    /**
     * pointcut 的 matches 方法
     */
    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        // 得到的结果为AnnotationTransactionAttributeSource
        TransactionAttributeSource tas = getTransactionAttributeSource();
        return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
    }

    /**
     * 在 BeanFactoryTransactionAttributeSourceAdvisor 中已指定
     */
    @Nullable
    protected abstract TransactionAttributeSource getTransactionAttributeSource();

    /**
     * 内部类，实现了 ClassFilter
     */
    private class TransactionAttributeSourceClassFilter implements ClassFilter {

        /**
         * ClassFilter 的 matches
         */
        @Override
        public boolean matches(Class<?> clazz) {
            // 是否为TransactionalProxy、PlatformTransactionManager、PersistenceExceptionTranslator的实现类
            if (TransactionalProxy.class.isAssignableFrom(clazz) ||
                    PlatformTransactionManager.class.isAssignableFrom(clazz) ||
                    PersistenceExceptionTranslator.class.isAssignableFrom(clazz)) {
                return false;
            }
            //判断 TransactionAttributeSource 获取事务属性是否为空
            // 得到的结果为AnnotationTransactionAttributeSource
            TransactionAttributeSource tas = getTransactionAttributeSource();
            return (tas == null || tas.isCandidateClass(clazz));
        }
    }

}

```

从上面的方法，我们得到了一个重要的规则：

*   匹配类：`pc.getClassFilter().matches(targetClass)`，`ClassFilter` 为 `TransactionAttributeSourceClassFilter`

匹配类的规则是找到了，那匹配方法的规则呢？我们进入 `TransactionAttributeSourcePointcut#getMethodMatcher()` 方法，进入的是 `StaticMethodMatcherPointcut`：

```
public abstract class StaticMethodMatcherPointcut 
        extends StaticMethodMatcher implements Pointcut {
    // 省略了一些代码
    ...

    @Override
    public final MethodMatcher getMethodMatcher() {
        return this;
    }
}

```

返回的竟然是 `this`！这是个啥？不要慌，再细看 `TransactionAttributeSourcePointcut`，发现它继承了 `StaticMethodMatcherPointcut`：

```
abstract class TransactionAttributeSourcePointcut 
        extends StaticMethodMatcherPointcut implements Serializable {
    // 省略了一些代码
    ...
}

```

所以，`pc.getMethodMatcher()` 得到的就是 `TransactionAttributeSourcePointcut`，其 `mathes(...)` 方法就是 `TransactionAttributeSourcePointcut#matches`.

在本小节的最后，我们来总结下分析的结果：

*   匹配类：`pc.getClassFilter().matches(targetClass)`，`ClassFilter` 为 `TransactionAttributeSourceClassFilter`；
*   匹配方法：`pc.getMethodMatcher().matches(method, targetClass)`，`methodMatcher` 为 `TransactionAttributeSourcePointcut`；
*   在上面两个规则中，都会调用 `TransactionAttributeSourcePointcut#getTransactionAttributeSource`，这个方法返回的结果为 `AnnotationTransactionAttributeSource`.

#### 1.4 匹配流程

在 1.2 部分，我们知道，当前 `advisor` 能否应用于目标 class，需要同时满足两个匹配规则：

*   匹配类：`pc.getClassFilter().matches(targetClass)`，`ClassFilter` 为 `TransactionAttributeSourceClassFilter`；
*   匹配方法：`pc.getMethodMatcher().matches(method, targetClass)`，`methodMatcher` 为 `TransactionAttributeSourcePointcut`；

我们来看下这两个方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-133740c93d470e16ec8dc9a34106adb8fc8.png)

*   `TransactionAttributeSourceClassFilter#matches`：先判断当前类是否为是否为 `TransactionalProxy`、`PlatformTransactionManager`、`PersistenceExceptionTranslator` 或其实现类，然后调用 `AnnotationTransactionAttributeSource#isCandidateClass` 继续判断；
*   `TransactionAttributeSourcePointcut#matches`：调用 `AnnotationTransactionAttributeSource#getTransactionAttribute`（由于继承关系，实际调用的是 `AbstractFallbackTransactionAttributeSource#getTransactionAttribute`）判断。

接下来我们就来分析下具体匹配流程。

##### `AnnotationTransactionAttributeSource#isCandidateClass`

让我们直奔主题，进入 `isCandidateClass` 方法：

> AnnotationTransactionAttributeSource#isCandidateClass

```
@Override
public boolean isCandidateClass(Class<?> targetClass) {
    // 找到所有的annotationParsers，循环匹配
    for (TransactionAnnotationParser parser : this.annotationParsers) {
        if (parser.isCandidateClass(targetClass)) {
            return true;
        }
    }
    return false;
}

```

可以看到，这是在循环里调用 `TransactionAnnotationParser` 的 `isCandidateClass` 方法。`this.annotationParsers` 是啥呢？通过调试，内容如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-5e16d6769e5a74c2f3064afdd090de08d42.png)

`this.annotationParsers` 中只有 `SpringTransactionAnnotationParser`，我们进入其 `isCandidateClass` 方法：

```
public class SpringTransactionAnnotationParser 
        implements TransactionAnnotationParser, Serializable {

    /**
     * 判断类上是否有 @Transactional 注解
     */
    @Override
    public boolean isCandidateClass(Class<?> targetClass) {
        return AnnotationUtils.isCandidateClass(targetClass, Transactional.class);
    }
}

```

这个方法最终调用的是 `AnnotationUtils.isCandidateClass`，用来判断指定的类上是否有 `@Transactional` 注解。

到这里，我们就明白了，`TransactionAttributeSourceClassFilter#matches` 先排除一些类 (`TransactionalProxy`/`PlatformTransactionManager`/`PersistenceExceptionTranslator` 及其子类) 后，最终会匹配带有 `@Transactional` 注解的类。

##### `AnnotationTransactionAttributeSource#getTransactionAttribute`

上面的方法匹配成功后，并不能表示成功匹配，还得匹配 `TransactionAttributeSourcePointcut#matches`，两者同时满足才会匹配成功。`TransactionAttributeSourcePointcut#matches` 调用 `AnnotationTransactionAttributeSource#getTransactionAttribute` 完成匹配的，我们跟进去：

```
public abstract class AbstractFallbackTransactionAttributeSource 
        implements TransactionAttributeSource {

    /**
     * 获取 @Transactional 注解的属性
     */
    public TransactionAttribute getTransactionAttribute(Method method, 
            @Nullable Class<?> targetClass) {
        if (method.getDeclaringClass() == Object.class) {
            return null;
        }

        // 省略从缓存中获取
        ...
        else {
            // 获取 Transaction 属性，即 @Transactional 注解的属性
            TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
            // 省略放入缓存操作
            ...
            return txAttr;
        }
    }
}

```

`AnnotationTransactionAttributeSource` 的 `getTransactionAttribute` 是继承自 `AbstractFallbackTransactionAttributeSource` 的，因此我们进入的方法是 `AbstractFallbackTransactionAttributeSource#getTransactionAttribute`，这个方法用来获取方法上的 `@Transactional` 注解的属性，我们跟进 `computeTransactionAttribute(...)`：

> AbstractFallbackTransactionAttributeSource

```
protected TransactionAttribute computeTransactionAttribute(Method method, 
        @Nullable Class<?> targetClass) {
    // 默认必须要 public 方法才支持事务
    if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
        return null;
    }
    // 1\. 获取确切的方法，例如传入的class是IFoo，实际的的class是DefaultFoo，
    //    那么应该将 IFoo#method 转换为 DefaultFoo#method
    Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);
    // 2\. 从方法上获取 @Transactional 的属性
    TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
    if (txAttr != null) {
        return txAttr;
    }
    // 3\. 从类上获取 @Transaction 的属性
    txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
    if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
        return txAttr;
    }
    if (specificMethod != method) {
        // 4\. 确切的方法上找不到，就找传入的方法上的
        txAttr = findTransactionAttribute(method);
        if (txAttr != null) {
            return txAttr;
        }
        // 5\. 以上都没找到，就找确切的类上的
        txAttr = findTransactionAttribute(method.getDeclaringClass());
        if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
            return txAttr;
        }
    }
    // 6\. 没有获取到，最终返回null
    return null;
}

```

从以上方法的流程，可总结获取 `@Transactional` 属性流程如下：

1.  将传入的方法转换为确切的方法，例如传入的 `class` 是 `IFoo`，实际的的 `class` 是 `DefaultFoo`，这里就会将 `IFoo#method` 转换为 `DefaultFoo#method`
2.  从确切的方法上获取 `@Transactional` 的属性
3.  如果没有获取到，就从确切的从类上获取 `@Transaction` 的属性
4.  如果没有获取到，就传入的方法上获取 `@Transaction` 的属性
5.  如果没有获取到，就传入的类上获取 `@Transaction` 的属性
6.  如果以上都没有获取到，就返回 `null`

spring 又是如何从方法或类上获取 `@Transactional` 的属性呢？继续看下去：

> AnnotationTransactionAttributeSource

```
    // 从方法上获取 @Transactional 属性
    protected TransactionAttribute findTransactionAttribute(Method method) {
        return determineTransactionAttribute(method);
    }

    // 从类上获取 @Transactional 属性
    protected TransactionAttribute findTransactionAttribute(Class<?> clazz) {
        return determineTransactionAttribute(clazz);
    }

    // 最终调用的方法
    protected TransactionAttribute determineTransactionAttribute(AnnotatedElement element) {
        for (TransactionAnnotationParser parser : this.annotationParsers) {
            // 解析 @Transactional 注解的属性
            TransactionAttribute attr = parser.parseTransactionAnnotation(element);
            if (attr != null) {
                return attr;
            }
        }
        return null;
    }

```

最终他们都是调用 `AnnotationTransactionAttributeSource#determineTransactionAttribute` 来获取的，而在 `AnnotationTransactionAttributeSource#determineTransactionAttribute` 调用了 `TransactionAnnotationParser#parseTransactionAnnotation` 方法。对于 `this.annotationParsers` 还眼熟吗，在前面已经分析过了，里面只有一个类：`SpringTransactionAnnotationParser`，我们跟进去：

> SpringTransactionAnnotationParser

```
    /**
     * 获取 Transactional 注解，存在则继续解析，不存在则返回 null
     */
    public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
        // 获取 Transactional 注解，存在则继续解析，不存在则返回 null
        AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(
                element, Transactional.class, false, false);
        if (attributes != null) {
            return parseTransactionAnnotation(attributes);
        }
        else {
            return null;
        }
    }

    /**
     * 解析 Transactional 注解的具体操作
     */
    protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
        RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();
        // 事务的传播方式
        Propagation propagation = attributes.getEnum("propagation");
        rbta.setPropagationBehavior(propagation.value());
        // 事务的隔离级别
        Isolation isolation = attributes.getEnum("isolation");
        rbta.setIsolationLevel(isolation.value());
        // 事务的超时时间
        rbta.setTimeout(attributes.getNumber("timeout").intValue());
        // 是否为只读
        rbta.setReadOnly(attributes.getBoolean("readOnly"));
        rbta.setQualifier(attributes.getString("value"));
        // 处理回滚异常
        List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
        for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
            rollbackRules.add(new RollbackRuleAttribute(rbRule));
        }
        for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
            rollbackRules.add(new RollbackRuleAttribute(rbRule));
        }
        // 处理不回滚异常
        for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
            rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
        }
        for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
            rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
        }
        rbta.setRollbackRules(rollbackRules);

        return rbta;
    }

```

可以看到，`Transactional` 注解的各属性解析成了 `RuleBasedTransactionAttribute`.

至此，我们就明白了，`TransactionAttributeSourcePointcut#matches` 就是用来判断类或方法上有没有 `Transactional` 注解。

#### 1.5 代理对象的创建

代理对象的创建在 `AbstractAutoProxyCreator#postProcessAfterInitialization` 方法中完成的，这个流程同 aop 流程真的是一模一样，这里就不再分析了，想了解的小伙伴可查看 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)。

### 2\. 方法的执行

方法的执行方面，事务与 aop 的执行流程并无区别，一样是通过 `Advisor` 找到对应的 `Advice`，再通过 `Advice` 找到对应的 `methodInterceptor`，最终执行的是 `MethodInterceptor#invoke` 方法，在事务这里 `MethodInterceptor` 为 `TransactionInterceptor`，这个类是在 `ProxyTransactionManagementConfiguration` 中通过 `@Bean` 注解引入的。

关于 aop 的流程，我们并不打算分析，相关分析在 [spring aop 之 jdk 动态代理](https://my.oschina.net/funcy/blog/4696654) 与[ spring aop 之 cglib 代理](https://my.oschina.net/funcy/blog/4696655) 已有详细分析，感兴趣的小伙伴可自行查阅，这里我们直接来看 `TransactionInterceptor#invoke` 的执行流程。

事务的处理的流程在 `TransactionInterceptor#invoke` 方法中：

> TransactionInterceptor#invoke

```
public Object invoke(MethodInvocation invocation) throws Throwable {
    Class<?> targetClass = (invocation.getThis() != null 
        ? AopUtils.getTargetClass(invocation.getThis()) : null);
    // 继续往下看
    return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
}

```

真正的处理逻辑在 `TransactionAspectSupport#invokeWithinTransaction` 方法中：

> TransactionAspectSupport#invokeWithinTransaction

```
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {
    TransactionAttributeSource tas = getTransactionAttributeSource();
    // 获取@Transactional的属性配置
    final TransactionAttribute txAttr = (tas != null 
        ? tas.getTransactionAttribute(method, targetClass) : null);
    // 获取事务管理器（IOC容器中获取）
    final TransactionManager tm = determineTransactionManager(txAttr);

    // 省略 ReactiveTransactionManager 的处理
    ...

    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    // 获取方法的全限定名，格式为："包名.类型.方法名"
    final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

    // 事务的处理逻辑，这也是我们接下来主要分析的地方
    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        // 1\. 开启事务
        TransactionInfo txInfo = createTransactionIfNecessary(ptm, txAttr, joinpointIdentification);
        Object retVal;
        try {
            // 2\. 执行具体的业务
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // 3\. 异常回滚
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            // 重置事务信息
            cleanupTransactionInfo(txInfo);
        }
        if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
            TransactionStatus status = txInfo.getTransactionStatus();
            if (status != null && txAttr != null) {
                retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
            }
        }
        // 4\. 提交事务，代码中会判断是否能支持
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
    else {
        // 省略其他
        ...
    }
}

```

以上方法就是事务的全部流程了，步骤如下：

1.  开启事务
2.  执行业务代码
3.  异常回滚
4.  提交事务

关于事务的具体分析，我们将在下一篇文章分析，本文只需要对事务的处理流程有大致了解即可。

### 3\. 总结

本文主要分析了事务中代理对象的创建及执行流程，实际上这些步骤同 aop 基本一致，本文着重分析了与 aop 不同的部分：

*   在代理对象的创建方面，分析了如何判断当前方法能否使用 `BeanFactoryTransactionAttributeSourceAdvisor`，重点分析了 `TransactionAttributeSourceClassFilter#matches` 与 `TransactionAttributeSourcePointcut#matches` 方法，这是判断的核心所在；

*   在方法的执行上的，粗略分析了 `TransactionInterceptor#invoke` 的执行流程，这些事务的开启、提交、异常回滚等流程跟我们平常使用的差别不大，不过里面的具体细节我们并没有分析。

本文重点是代理对象的创建及执行流程，事务执行的具体细节我们下篇再分析。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4773457](https://my.oschina.net/funcy/blog/4773457) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_