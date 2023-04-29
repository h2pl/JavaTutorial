[上一篇文章](https://my.oschina.net/funcy/blog/4678817 "上一篇文章")主要分析了 `AbstractAutoProxyCreator#postProcessAfterInitialization` 方法，本文我们来分析 `AbstractAutoProxyCreator#postProcessAfterInitialization` 方法。

我们先来看看这个方法的调用链：

```
|-AnnotationConfigApplicationContext
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#finishBeanFactoryInitialization
   |-ConfigurableListableBeanFactory#preInstantiateSingletons
    |-AbstractBeanFactory#getBean(String)
     |-AbstractBeanFactory#doGetBean
      |-DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)
       |-ObjectFactory#getObject
        |-AbstractBeanFactory#createBean
         |-AbstractAutowireCapableBeanFactory#doCreateBean
          |-AbstractAutowireCapableBeanFactory#initializeBean(String, Object, RootBeanDefinition)
           |-AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization
            |-AbstractAutoProxyCreator#postProcessAfterInitialization

```

实际上这个方法的调用链就是 spring bean 的创建过程，我们进入 `AbstractAutoProxyCreator#postProcessAfterInitialization`：

```
@Override
public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
    if (bean != null) {
        Object cacheKey = getCacheKey(bean.getClass(), beanName);
        if (this.earlyProxyReferences.remove(cacheKey) != bean) {
           // 深入wrapIfNecessary()方法
           return wrapIfNecessary(bean, beanName, cacheKey);
        }
    }
    return bean;
}

```

继续进入 `AbstractAutoProxyCreator#wrapIfNecessary`：

```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    //如果已经处理过
    if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
        return bean;
    }
    //如果当前类是增强类，返回
    if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
        return bean;
    }

    // 重要代码一：判断当前类是否为切面类，该代码在上一篇文章已经分析了，就不多说了
    if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
        this.advisedBeans.put(cacheKey, Boolean.FALSE);
        return bean;
    }

    // 重要代码二：校验此类是否应该被代理，获取这个类的增强
    Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
    //如果获取到了增强则需要针对增强创建代理
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 重要代码三：创建代理
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}

```

这个方法 看着有点长，但大多代码都是在做判断，与 aop 功能关系不大。真正有关系的代码只有三行：

```
// 重要代码一：
// 1\. isInfrastructureClass：判断当前是否为aop相关类，
//    如Advice/Pointcut/Advisor等的子类，是否包含 @AspectJ的注解
// 2\. shouldSkip：查找所有切面类，判断是否被排除
if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
    ...
}

// 重要代码二：校验此类是否应该被代理，获取这个类的增强
Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);

// 重要代码三：创建代理
Object proxy = createProxy(
    bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));

```

对于`重要代码一`，上一篇文章已经分析过，本文我们主要分析`重要代码二`与`重要代码三`。

### 1\. 获取类的增强

> AbstractAdvisorAutoProxyCreator

```
@Override
@Nullable
protected Object[] getAdvicesAndAdvisorsForBean(
        Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
    // 查找符合条件的增强，继续往下看
    List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
    if (advisors.isEmpty()) {
        return DO_NOT_PROXY;
    }
    return advisors.toArray();
}

/**
 * 查找符合条件的增强
 */
protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
    //获取容器中的所有增强，在上一篇文章中已经分析过了
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    //验证beanClass是否该被代理，如果应该，则返回适用于这个bean的增强
    List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
    extendAdvisors(eligibleAdvisors);
    if (!eligibleAdvisors.isEmpty()) {
        eligibleAdvisors = sortAdvisors(eligibleAdvisors);
    }
    return eligibleAdvisors;
}

/**
 * 验证beanClass是否该被代理
 */
protected List<Advisor> findAdvisorsThatCanApply(
        List<Advisor> candidateAdvisors, Class<?> beanClass, String beanName) {
    ProxyCreationContext.setCurrentProxiedBeanName(beanName);
    try {
        // 验证beanClass是否该被代理
        return AopUtils.findAdvisorsThatCanApply(candidateAdvisors, beanClass);
    }
    finally {
        ProxyCreationContext.setCurrentProxiedBeanName(null);
    }
}

```

spring 的方法调用比较深，一路追踪，最终到了 `AopUtils.findAdvisorsThatCanApply` 方法，继续往下看：

> AopUtils

```
public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
    if (candidateAdvisors.isEmpty()) {
        return candidateAdvisors;
    }
    List<Advisor> eligibleAdvisors = new ArrayList<>();
    // 遍历 candidateAdvisors，判断是否满足代理条件
    for (Advisor candidate : candidateAdvisors) {
        //处理增强，重点，再往下看
        if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
            eligibleAdvisors.add(candidate);
        }
    }
    boolean hasIntroductions = !eligibleAdvisors.isEmpty();
    for (Advisor candidate : candidateAdvisors) {
        if (candidate instanceof IntroductionAdvisor) {
            // already processed
            continue;
        }
        //对普通bean的处理
        if (canApply(candidate, clazz, hasIntroductions)) {
            eligibleAdvisors.add(candidate);
        }
    }
    return eligibleAdvisors;
}

/**
 * 判断是否需要增强
 */
public static boolean canApply(Advisor advisor, Class<?> targetClass) {
    // 调用下一个方法
    return canApply(advisor, targetClass, false);
}

/**
 * 判断是否需要增强
 */
public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
    //如果存在排除的配置
    if (advisor instanceof IntroductionAdvisor) {
        return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
    }
    else if (advisor instanceof PointcutAdvisor) {
        PointcutAdvisor pca = (PointcutAdvisor) advisor;
        //进入该方法继续
        return canApply(pca.getPointcut(), targetClass, hasIntroductions);
    }
    else {
        // It doesn't have a pointcut so we assume it applies.
        return true;
    }
}

/**
 * 判断是否需要增强
 */
public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
    Assert.notNull(pc, "Pointcut must not be null");
    //切点上是否存在排除类的配置
    if (!pc.getClassFilter().matches(targetClass)) {
        return false;
    }
    //验证注解的作用域是否可以作用于方法上
    MethodMatcher methodMatcher = pc.getMethodMatcher();
    if (methodMatcher == MethodMatcher.TRUE) {
        // No need to iterate the methods if we're matching any method anyway...
        return true;
    }

    IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
    if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
        introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
    }

    // classes包含targetClass、除Object的所有父类、所有接口
    Set<Class<?>> classes = new LinkedHashSet<>();
    if (!Proxy.isProxyClass(targetClass)) {
        classes.add(ClassUtils.getUserClass(targetClass));
    }
    classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

    // 循环判断方法是否需要代理，
    // 这里可以看出，
    // 1\. 只要一个方法满足代理要求，那么整个类就会被代理
    // 2\. 如果父类有方法需要被代理，那么子类也会被代理
    for (Class<?> clazz : classes) {
         // 获取 clazz 定义的方法
         // 包括当前类的方法、除Object外的所有父类方法、接口的默认方法
        Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
        for (Method method : methods) {
            //获取类所实现的所有接口和所有类层级的方法，循环验证
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

看到这里，基本上就了解了 spring 是如何来判断一个对象是否需要被代理的，这里总结流程如下：

1.  获取到项目中所有的切面对象，将其中的切面方法封装为一个 `List<Advisor>`，具体操作在上一篇文章中有详细分析；
2.  遍历 `Advisor`，对每一个 `Advisor`，利用反射获取当前类的除 `Object` 外的所有父类及接口，结果为 `Set<Class>`；
3.  遍历 `Set<Class>`，对其中每一个 `Class`，利用反射获取该 `Class` 方法、除 Object 外的所有父类的方法、接口的默认方法，结果为 `Method[]`;
4.  遍历 `Method[]`，如果有一个 `method` 满足 `Advisor` 的切面条件，则表示当前 `Advisor` 可以应用到当前 bean，该 bean 就需要被代理，这一步最终得到的结果也是一个 `List<Advisor>`，表示该有多个 `Advisor` 需要应用到该对象。

伪代码类似于：

```
// 1\. 获取所有的Advisor
List<Advisor> advisorList = getAdvisorList();
// 2\. 遍历Advisor
for(Advisor advisor : advisorList) {
    // 获取当前类的除`Object`外的所有父类及接口，classSet也包含targetClass
    Set<Class> classSet = getSuperClassAndInterfaces(targetClass);
    for(Class cls : classSet) {
        // 遍历cls中定义的方法，包括当前类的方法、除Object外的所有父类方法、接口的默认方法
        Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
        //  遍历这些方法
        for (Method method : methods) {
             // 判断method是否满足切面条件
        }
    }
}

```

得到 `List<Advisor>` 后，接下来就是根据 `List<Advisor>` 来创建代理对象了。

### 2\. 创建代理对象

我们来看看 spring 创建代理对象的流程。

> `AbstractAutoProxyCreator#wrapIfNecessary`：

```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
    ...
    if (specificInterceptors != DO_NOT_PROXY) {
        this.advisedBeans.put(cacheKey, Boolean.TRUE);
        // 代理对象就是在这里创建的
        // - specificInterceptors：可以应用到该对象的  Advisor，
        //    specificInterceptors的获取过程，上一部分已经详细分析过了，不再赘述了
        // - SingletonTargetSource：对原始对象的一个包装
        Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, 
                        new SingletonTargetSource(bean));
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }
    this.advisedBeans.put(cacheKey, Boolean.FALSE);
    return bean;
}

```

我们先来看看 `SingletonTargetSource`：

```
public class SingletonTargetSource implements TargetSource, Serializable {

    private static final long serialVersionUID = 9031246629662423738L;

    private final Object target;

    public SingletonTargetSource(Object target) {
        Assert.notNull(target, "Target object must not be null");
        this.target = target;
    }

    @Override
    public Class<?> getTargetClass() {
        return this.target.getClass();
    }

    @Override
    public Object getTarget() {
        return this.target;
    }

    ...
}

```

这个类代码很简单，就是对原始对象做了一层包装。继续往下看：

> AbstractAutoProxyCreator#createProxy

```
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {

    if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
        AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) 
            this.beanFactory, beanName, beanClass);
    }

    ProxyFactory proxyFactory = new ProxyFactory();
    //使用proxyFactory对象copy当前类中的相关属性
    proxyFactory.copyFrom(this);

    // 判断是否使用Cglib动态代理，可以在注解中指定：
    // @EnableAspectJAutoProxy(proxyTargetClass = true)
    if (!proxyFactory.isProxyTargetClass()) {
        // 如果 beanFactory 是 ConfigurableListableBeanFactory，
        // 则可在 BeanDefinition 设置属性，单独控制某一个类使用 cglib 代理
        if (shouldProxyTargetClass(beanClass, beanName)) {
            proxyFactory.setProxyTargetClass(true);
        }
        else {
            // 如果没有配置开启, 则判断bean是否有合适的接口使用JDK的动态代理
            // 注意：JDK动态代理必须是带有接口的类
            // 如果类没有实现任何接口则只能使用Cglib动态代理）
            evaluateProxyInterfaces(beanClass, proxyFactory);
        }
    }

    // 构建Advisor，这里包含两个操作：
    // 1\. 添加公共的 Interceptor
    // 2\. 对于给定的advisor，判断其类型，然后转换为其具体类型
    Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
    //添加所有增强
    proxyFactory.addAdvisors(advisors);
    //设置要代理的类
    proxyFactory.setTargetSource(targetSource);
    //Spring的一个扩展点，默认实现为空。留给我们在需要对代理进行特殊操作的时候实现
    customizeProxyFactory(proxyFactory);

    proxyFactory.setFrozen(this.freezeProxy);
    if (advisorsPreFiltered()) {
        proxyFactory.setPreFiltered(true);
    }
    //使用代理工厂获取代理对象
    return proxyFactory.getProxy(getProxyClassLoader());
}

```

在 `@EnableAspectJAutoProxy` 注解中，可以使用 `proxyTargetClass = true` 来设置项目使用 `cglib` 代理，这在代码中也有体现：

```
// 只有在proxyFactory.isProxyTargetClass()为false时，才会进行下面的判断
// 换言之，当 @EnableAspectJAutoProxy(proxyTargetClass = true) 时
// 下面的代码是不会运行的，默认使用就是cglib代理
if (!proxyFactory.isProxyTargetClass()) {
    // 判断有没在 BeanDefinition 中设置使用 cglib代理
    if (shouldProxyTargetClass(beanClass, beanName)) {
        proxyFactory.setProxyTargetClass(true);
    }
    else {
        // 是否满足代理接口的条件，即是否满足jdk动态代理的条件
        evaluateProxyInterfaces(beanClass, proxyFactory);
    }
}

```

spring 是如何判断一个类是否满足 jdk 动态代理的条件的呢？从我们来认知上来说，实现了接口，就可以使用动态代理，否则就只能使用 cglib 代理，我们来看看 spring 是如何判断的：

> ProxyProcessorSupport#evaluateProxyInterfaces

```
/**
 * 判断是否能使用jdk动态代理
 */
protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
    // 获取类的所有接口
    Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
    boolean hasReasonableProxyInterface = false;
    for (Class<?> ifc : targetInterfaces) {
        // 1.isConfigurationCallbackInterface: 判断ifc是否为InitializingBean，DisposableBean，
        //   Closeable，AutoCloseable，以及包含 Aware
        // 2.isInternalLanguageInterface: 是否为内部语言接口，如groovy，mock等
        // 3.ifc.getMethods().length > 0：接口的方法数必须大于1
        if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
                ifc.getMethods().length > 0) {
            hasReasonableProxyInterface = true;
            break;
        }
    }
    if (hasReasonableProxyInterface) {
         // 需要将所有的接口都设置到proxyFactory
         // 试想一下，如果一个类A 实现了接口 I1 与 I2，
         // 如果类A的对象a  托管到了spring容器，那么无论是使用 beanFactory.get(I1.class)，
         // 还是 beanFactory.get(I1.class)，都应该能获取到a.
         for (Class<?> ifc : targetInterfaces) {
             proxyFactory.addInterface(ifc);
         }
    }
    else {
        proxyFactory.setProxyTargetClass(true);
    }
}

```

从源码上来看，spring 判断是否能使用 jdk 动态代理的过程与我们认知上的差不多，不过并不是实现了任意接口就能使用 jdk 动态代理，spring 会排除 `InitializingBean`，`DisposableBean`，`Closeable`，`AutoCloseable` 等接口，同时也会排除无任何方法的接口。

分析完 spring 如何判断是否使用 jdk 动态代理后，接口我们来看看 spring 是如何创建代理对象的。为了说明问题，首先简化下 `AbstractAutoProxyCreator#createProxy`：

```
protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
        @Nullable Object[] specificInterceptors, TargetSource targetSource) {
    // 省略一些代码
    ...
    ProxyFactory proxyFactory = new ProxyFactory();
    //使用proxyFactory对象copy当前类中的相关属性
    proxyFactory.copyFrom(this);
    // 这里省略了好多的判断
    proxyFactory.setProxyTargetClass(true);
    //添加所有增强
    proxyFactory.addAdvisors(advisors);
    //设置要代理的类
    proxyFactory.setTargetSource(targetSource);
    ...
    //使用代理工厂获取代理对象
    return proxyFactory.getProxy(getProxyClassLoader());
}

```

从这里可以看出，这个方法创建了一个 `ProxyFactory` 对象，然后往该对象的属性里设置了一些值。继续往下看：

> ProxyFactory#getProxy(java.lang.ClassLoader)

```
public Object getProxy(@Nullable ClassLoader classLoader) {
    return createAopProxy().getProxy(classLoader);
}

```

这里有两个方法：`createAopProxy()` 与 `getProxy(classLoader)`，我们先来看 `createAopProxy()`。

> ProxyCreatorSupport#createAopProxy

```
protected final synchronized AopProxy createAopProxy() {
    if (!this.active) {
         activate();
    }
    return getAopProxyFactory().createAopProxy(this);
}

```

继续，

> DefaultAopProxyFactory#createAopProxy

```
/**
 * 判断代理类型
 * 如果能使用jdk动态代理，就返回 JdkDynamicAopProxy
 * 否则就返回 ObjenesisCglibAopProxy
 */
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException(...);
        }
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}

```

到这里，我们就能明白，`JdkDynamicAopProxy` 是用来处理 jdk 动态代理的，`ObjenesisCglibAopProxy` 是用来处理 cglib 代理的。接下来，我们来看看 `getProxy(classLoader)` 方法。

> JdkDynamicAopProxy#getProxy(java.lang.ClassLoader)

```
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    Class<?>[] proxiedInterfaces = AopProxyUtils
            .completeProxiedInterfaces(this.advised, true);
    // 是否有equals()与hashCode()方法
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    // 调用 jdk 方法 创建对象
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}

```

最后我们来看看得到的代理对象是什么样的：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-ee46c03d86755b936862c9e8cde266fca1e.png)

可以看到，代理对象的 `h` 属性保存的就是 `JdkDynamicAopProxy` 对象，`JdkDynamicAopProxy` 对象的 `advised` 属性保存了代理的代理的相关信息。

> CglibAopProxy#getProxy(java.lang.ClassLoader)

```
public Object getProxy(@Nullable ClassLoader classLoader) {
    try {
        Class<?> rootClass = this.advised.getTargetClass();
        Assert.state(rootClass != null, "xxx");

        Class<?> proxySuperClass = rootClass;
        if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
            proxySuperClass = rootClass.getSuperclass();
            Class<?>[] additionalInterfaces = rootClass.getInterfaces();
            for (Class<?> additionalInterface : additionalInterfaces) {
                this.advised.addInterface(additionalInterface);
            }
        }

        validateClassIfNecessary(proxySuperClass, classLoader);

        // 创建 Enhancer 对象，并set一些属性
        Enhancer enhancer = createEnhancer();
        if (classLoader != null) {
            enhancer.setClassLoader(classLoader);
            if (classLoader instanceof SmartClassLoader &&
                    ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                enhancer.setUseCache(false);
            }
        }
        // Superclass就是要代理的类
        enhancer.setSuperclass(proxySuperClass);
        // 设置接口，如 SpringProxy，Advised
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

        Callback[] callbacks = getCallbacks(rootClass);
        Class<?>[] types = new Class<?>[callbacks.length];
        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        enhancer.setCallbackFilter(new ProxyCallbackFilter(
                this.advised.getConfigurationOnlyCopy(), 
                this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);

        return createProxyClassAndInstance(enhancer, callbacks);
    }
    catch (CodeGenerationException | IllegalArgumentException ex) {
        throw new AopConfigException(...);
    }
    catch (Throwable ex) {
        throw new AopConfigException("Unexpected AOP exception", ex);
    }
}

```

spring 使用 cglib 创建对象，主要用到了 `Enhancer` 类，关于这一块后面会再分析。

最后我们也来看看创建后得到的对象：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-97d647a818f62fa0979dabd58f3aa19e473.png)

### 3\. 总结

本文主要分析了 `AbstractAutoProxyCreator#postProcessAfterInitialization`，该方法主要做了三件事：

1.  获取项目中所有切面类，将切点方法包装为 `List`：实际上，这一步的操作也会在 `AbstractAutoProxyCreator#postProcessBeforeInitialization` 中执行，然后将结果缓存了起来，这一步其实是直接在缓存中拿结果；
2.  获取当前对象的所有增强：这一步就是判断哪些增强可以用于当前对象，判断时先获取当前类的所有接口与不包括 Object 的父类，然后逐一判断这些接口与类中的方法是否满足增强条件，只要有一个方法满足，就表示当前对象需要被代理；
3.  创建代理对象：创建代理对象时，默认情况下，会根据是否实现了接口来选择使用 jdk 动态代理还是 cglib，可应用于当前 bean 的 `List` 也会封装进代理对象中。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4687961](https://my.oschina.net/funcy/blog/4687961) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_