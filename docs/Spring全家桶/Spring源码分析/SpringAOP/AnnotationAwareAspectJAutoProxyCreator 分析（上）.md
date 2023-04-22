在[上一篇文章](https://my.oschina.net/funcy/blog/4678093 "上一篇文章")的分析中，我们介绍了使用 `@EnableAspectJAutoProxy` 开启用 spring aop 功能，而 `@EnableAspectJAutoProxy` 最终向 spring 中引入了一个类：`AnnotationAwareAspectJAutoProxyCreator`，本文就从该类入手，进 一步分析 spring aop 功能。

首先我们来看看这个类的继承关系：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-53b639dcaacb2480e9098a046407a80a574.png)

从继承关系上来看，`AnnotationAwareAspectJAutoProxyCreator` 是一个 `BeanPostProcessor`，结合前面的分析，spring `BeanPostProcessor` 的执行是在 spring bean 初始化前后，这里我们通过断点调试的方式验证下。

### 1\. 调试查看代理对象的产生

我们将断点打在 `AbstractAutowireCapableBeanFactory#initializeBean(String, Object, RootBeanDefinition)` 方法上，然后在 debug 模式下运行：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-449fe5a3456350ae0d9854c1c13f4a59a80.png)

此时的 `wrappedBean` 的类型还是 `AopBean1`，继续往下，当运行完 `applyBeanPostProcessorsAfterInitialization` 后，`wrappedBean` 的类型就变了样：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8b791334bf1873c652a265db9ab1d0e1441.png)

表现上看还是 `AopBean1`，但前面出现了 `$Poxy19` 字样，且多了一个属性：`JdkDynamicAopProxy`，这表明该动态是 jdk 动态代理生成的对象。

再看看 `AopBean2` 运行到此处的变化：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-4fc5d03af4298f55d69e5c619cd72df1ff0.png)

可以看到，类名中出现了 `SpringCGLIB` 字样，这表示该对象是由 spring cglib 代理生成的。

从以上调试结果来看，spring 是在 `AnnotationAwareAspectJAutoProxyCreator#postProcessAfterInitialization` 中完成对象代理的。

实际上，`AnnotationAwareAspectJAutoProxyCreator` 中并未实现 `postProcessAfterInitialization` 方法，`AnnotationAwareAspectJAutoProxyCreator` 的 `postProcessAfterInitialization` 方法继承自 `AbstractAutoProxyCreator#postProcessAfterInitialization`，因此 spring 对象代理的实际完成是在 `AbstractAutoProxyCreator#postProcessAfterInitialization` 中。

`AnnotationAwareAspectJAutoProxyCreator` 是 `BeanPostProcessor` 的子类，spring 在 bean 的初始化前后会调用 `BeanPostProcessor#postProcessBeforeInitialization` 与 `BeanPostProcessor#postProcessAfterInitialization` 方法，我们将从源码上看看 `AnnotationAwareAspectJAutoProxyCreator` 在 bean 的初始前后是如何完成 aop 操作。

### 2. `AbstractAutoProxyCreator#postProcessBeforeInitialization` 方法

代理对象的生成虽然是在 `BeanPostProcessor#postProcessAfterInitialization` 方法，但正所谓 “做戏做全套”，本文我们先分析 `AbstractAutoProxyCreator#postProcessBeforeInitialization` 方法，看看这个方法做了些什么，至于 `AbstractAutoProxyCreator#postProcessAfterInitialization` 方法，将留到下一篇文章分析。

`AnnotationAwareAspectJAutoProxyCreator` 的 `postProcessBeforeInitialization` 方法继承自 `AbstractAutoProxyCreator`，`AbstractAutoProxyCreator#postProcessBeforeInitialization` 方法如下：

```
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) {
    Object cacheKey = getCacheKey(beanClass, beanName);

    if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
        if (this.advisedBeans.containsKey(cacheKey)) {
            return null;
        }
        //1\. 加载所有增强
        if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
            this.advisedBeans.put(cacheKey, Boolean.FALSE);
            return null;
        }
    }

    // 2\. 如果有自定义的TargetSource，则运行下面的方法来创建代理
    TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
    if (targetSource != null) {
        if (StringUtils.hasLength(beanName)) {
            this.targetSourcedBeans.add(beanName);
        }
        Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
        Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
        this.proxyTypes.put(cacheKey, proxy.getClass());
        return proxy;
    }

    return null;
}

```

这个方法主要功能就是加载项目中的增强方法，处理代码如下：

```
if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {
     ...
}

```

这段代码包含两个方法：`isInfrastructureClass(beanClass)` 与 `shouldSkip(beanClass, beanName)`，先来看 `isInfrastructureClass(beanClass)`

#### 2.1 `isInfrastructureClass(beanClass)` 方法

> AnnotationAwareAspectJAutoProxyCreator#isInfrastructureClass

```
@Override
protected boolean isInfrastructureClass(Class<?> beanClass) {
     // 判断当前类是否为 Advice/Pointcut/Advisor/AopInfrastructureBean 的子类
     return (super.isInfrastructureClass(beanClass) ||
          // 判断当前beanClass是否为切面：包含有@Aspect注解， 且不由ajc编译
         (this.aspectJAdvisorFactory != null && this.aspectJAdvisorFactory.isAspect(beanClass)));
}

```

以上代码就两行，功能注释得很清楚了，简单来说就是判断是否为 aop 自身相关的类，如果是 aop 自身相关的类，就不进行切面操作了。

#### 2.2 `shouldSkip(beanClass, beanName)` 方法

接着，我们来看 `shouldSkip(beanClass, beanName)` 方法：

> AspectJAwareAdvisorAutoProxyCreator#shouldSkip

```
@Override
protected boolean shouldSkip(Class<?> beanClass, String beanName) {
    //查找所有标识了@Aspect注解的类，这个方法很重要 ，下面会继续分析
    List<Advisor> candidateAdvisors = findCandidateAdvisors();
    for (Advisor advisor : candidateAdvisors) {
        // 如果当前 advisor 为 AspectJPointcutAdvisor 的实例且 AspectName 为 beanName，返回true
        if (advisor instanceof AspectJPointcutAdvisor &&
                ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)) {
            return true;
        }
    }
    // 调用父类的方法，判断当前 bean 是否为 OriginalInstance
    return super.shouldSkip(beanClass, beanName);
}

```

这个方法会先查找所有标识了 `@Aspect` 注解的类，然后做一些判断，其中 `findCandidateAdvisors` 很关键，我们来看看这个方法：

> AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors

```
@Override
protected List<Advisor> findCandidateAdvisors() {
    // 调用父类的方法，查找当前beanFactory中的Advisor bean
    List<Advisor> advisors = super.findCandidateAdvisors();
    if (this.aspectJAdvisorsBuilder != null) {
        // 将包含 @Aspect 注解的类构建为 Advisor，这句是关键
        advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
    }
    return advisors;
}

```

首先来看下 `super.findCandidateAdvisors()` 方法，也就是 `AbstractAdvisorAutoProxyCreator#findCandidateAdvisors`:

```
protected List<Advisor> findCandidateAdvisors() {
    Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
    return this.advisorRetrievalHelper.findAdvisorBeans();
}

```

这个调用了 `BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans`，继续下去：

```
public List<Advisor> findAdvisorBeans() {
    String[] advisorNames = this.cachedAdvisorBeanNames;
    if (advisorNames == null) {
        // 1\. 查找当前beanFactory中所有 Advisor 的 bean class
        advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                this.beanFactory, Advisor.class, true, false);
        this.cachedAdvisorBeanNames = advisorNames;
    }
    ...

    List<Advisor> advisors = new ArrayList<>();
    for (String name : advisorNames) {
            ...
            // 2\. 从spring中获取 bean class 对应的 bean，将其放入advisors中
            advisors.add(this.beanFactory.getBean(name, Advisor.class));
            ...

    }
    return advisors;
}

```

以上代码做了两件事：

1.  查找当前 beanFactory 中所有 Advisor 的 bean class，Advisor 可以是用户实现 Advisor 相关接口，也可以是 xml 指定的
2.  从 spring 中获取 bean class 对应的 bean，将其放入 advisors 中

#### 2.3 `aspectJAdvisorsBuilder.buildAspectJAdvisors()` 方法

接着我们再回过头来看看 `aspectJAdvisorsBuilder.buildAspectJAdvisors())` 方法 ：

> BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

```
public List<Advisor> buildAspectJAdvisors() {
    List<String> aspectNames = this.aspectBeanNames;

    // 只有在第一次运行时，才会成立
    if (aspectNames == null) {
        synchronized (this) {
            aspectNames = this.aspectBeanNames;
            if (aspectNames == null) {
                List<Advisor> advisors = new ArrayList<>();
                aspectNames = new ArrayList<>();
                //1\. 获取所有Bean名称
                String[] beanNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
                        this.beanFactory, Object.class, true, false);
                for (String beanName : beanNames) {
                    ...
                    //2\. 判断Bean的Class上是否标识@Aspect注解
                    if (this.advisorFactory.isAspect(beanType)) {
                        aspectNames.add(beanName);
                        AspectMetadata amd = new AspectMetadata(beanType, beanName);
                        if (amd.getAjType().getPerClause().getKind() == PerClauseKind.SINGLETON) {
                            // 创建factory对象，这个对象里包含了beanFactory与@Aspect的beanName
                            MetadataAwareAspectInstanceFactory factory =
                                new BeanFactoryAspectInstanceFactory(this.beanFactory, beanName);
                            // 3\. 解析所有的增强方法, 这个方法是重点的重点
                            List<Advisor> classAdvisors = this.advisorFactory.getAdvisors(factory);
                            if (this.beanFactory.isSingleton(beanName)) {
                                //将解析的Bean名称及类上的增强缓存起来,每个Bean只解析一次
                                this.advisorsCache.put(beanName, classAdvisors);
                            }
                            ...
                        }
                        else {
                            ...
                        }
                    }
                }
                // 对属性赋值，之后就不会再运行了
                this.aspectBeanNames = aspectNames;
                return advisors;
            }
        }
    }

    if (aspectNames.isEmpty()) {
        return Collections.emptyList();
    }
    List<Advisor> advisors = new ArrayList<>();
    for (String aspectName : aspectNames) {
        //从缓存中获取当前Bean的切面实例，如果不为空，则指明当前Bean的Class标识了@Aspect，且有切面方法
        List<Advisor> cachedAdvisors = this.advisorsCache.get(aspectName);
        if (cachedAdvisors != null) {
            advisors.addAll(cachedAdvisors);
        }
        else {
            MetadataAwareAspectInstanceFactory factory = this.aspectFactoryCache.get(aspectName);
            advisors.addAll(this.advisorFactory.getAdvisors(factory));
        }
    }
    return advisors;
}

```

方法虽然有点长，不过只做了 3 件事：

1.  获取所有 beanName：`BeanFactoryUtils.beanNamesForTypeIncludingAncestors`
2.  找出所有标记 Aspect 注解的类：`advisorFactory.isAspect(beanType)`
3.  对标记 Aspect 的类提取增强器：`this.advisorFactory.getAdvisors(factory)`

这里我们重点分析第 3 点，看看 `Advisors` 是如何构建的。

> ReflectiveAspectJAdvisorFactory#getAdvisors

```
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    //获取Aspect类、类名称、并校验
    // aspectInstanceFactory 里包含beanFactory与 @Aspect 的 beanName
    Class<?> aspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    String aspectName = aspectInstanceFactory.getAspectMetadata().getAspectName();
    //校验类的合法性相关
    validate(aspectClass);

    MetadataAwareAspectInstanceFactory lazySingletonAspectInstanceFactory =
                  new LazySingletonAspectInstanceFactoryDecorator(aspectInstanceFactory);
    List<Advisor> advisors = new ArrayList<>();
    // 获取这个类除去 @Pointcut 外的所有方法，看下面的分析
    for (Method method : getAdvisorMethods(aspectClass)) {
          // 生成增强实例，这个 方法是重点，看后面的分析
          // 注意 advisors.size()，这个指定了advisor的顺序，由于获取之后会个添加
          // 操作，因此这个值是递增且是不重复的
          Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, 
                    advisors.size(), aspectName);
          if (advisor != null) {
              // 添加 advisors 列表
              advisors.add(advisor);
          }
    }
    // 如果需要增强且配置了延迟增强,则在第一个位置添加同步实例化增强方法
    if (!advisors.isEmpty() && lazySingletonAspectInstanceFactory
                    .getAspectMetadata().isLazilyInstantiated()) {
        Advisor instantiationAdvisor = new SyntheticInstantiationAdvisor(
                    lazySingletonAspectInstanceFactory);
        advisors.add(0, instantiationAdvisor);
    }
    // 获取属性中配置DeclareParents注解的增强
    for (Field field : aspectClass.getDeclaredFields()) {
        Advisor advisor = getDeclareParentsAdvisor(field);
        if (advisor != null) {
              advisors.add(advisor);
        }
    }
    return advisors;
}

/**
 * 获取指定类除@Pointcut外的所有方法
 */
private List<Method> getAdvisorMethods(Class<?> aspectClass) {
    final List<Method> methods = new ArrayList<>();
    // 递归操作 ：排除@Pointcut标识的方法，获取当前类的的方法、来自接口的默认方法，再对父类进行同样的操作
    // 最终结果：除@Pointcut标识之外的所有方法，得到的方法集合包括：
    // 1\. 继承自父类的方法，
    // 2\. 来自自接口的默认的方法
    // 3\. 不包含继承自Object的方法
    ReflectionUtils.doWithMethods(aspectClass, method -> {
        if (AnnotationUtils.getAnnotation(method, Pointcut.class) == null) {
            methods.add(method);
        }
    }, ReflectionUtils.USER_DECLARED_METHODS);
    //对得到的所有方法排序，
    //如果方法标识了切面注解，则按@Around, @Before, @After, @AfterReturning, @AfterThrowing的顺序排序
    //如果没有标识这些注解，则按方法名称的字符串排序,
    //有注解的方法排在无注解的方法之前
    //最后的排序应该是这样的Around, Before, After, AfterReturning, AfterThrowing
    methods.sort(METHOD_COMPARATOR);
    return methods;
}

```

以上方法主要做了两件事：

1.  获取当前类除 `@Pointcut` 外的所有增强方法，包括继承自父类的方法，继承自 Object 的方法，以及来自接口的默认方法
2.  遍历得到的方法，从符合条件的方法得到 Advisor 实例，保存到集合中

关于第一步，上面的代码已经分析了，主要是根据反射来进行的，就不再深入了，这里我们来看第二步，该操作是在 `ReflectiveAspectJAdvisorFactory#getAdvisor` 方法中：

```
@Override
@Nullable
/**
 * declarationOrderInAspect：指定了Advisor的顺序，来源于 AdvisorMethods 的排序，上面已分析
 */
public Advisor getAdvisor(Method candidateAdviceMethod, MetadataAwareAspectInstanceFactory 
        aspectInstanceFactory, int declarationOrderInAspect, String aspectName) {
    //再次校验类的合法性
    validate(aspectInstanceFactory.getAspectMetadata().getAspectClass());
    //切点表达式的包装类里面包含这些东西：@Around("testAop()") 里的 testAop()
    AspectJExpressionPointcut expressionPointcut = getPointcut(
            candidateAdviceMethod, aspectInstanceFactory.getAspectMetadata().getAspectClass());
    if (expressionPointcut == null) {
        return null;
    }
    //根据方法、切点、AOP实例工厂、类名、序号生成切面实例，下面我们先来看看是如何实例化的
    return new InstantiationModelAwarePointcutAdvisorImpl(expressionPointcut, candidateAdviceMethod,
               this, aspectInstanceFactory, declarationOrderInAspect, aspectName);
}

/**
 * 处理切点表达式
 */
@Nullable
private AspectJExpressionPointcut getPointcut(Method candidateAdviceMethod, 
               Class<?> candidateAspectClass) {
    // 查询方法上的切面注解，根据注解生成相应类型的AspectJAnnotation,
    // 在调用AspectJAnnotation的构造函数的同时，根据注解value或pointcut属性得到切点表达式，
    // 有argNames则设置参数名称
    AspectJAnnotation<?> aspectJAnnotation =
               AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
    // 过滤那些不含@Before, @Around, @After, @AfterReturning, @AfterThrowing注解的方法
    if (aspectJAnnotation == null) {
        return null;
    }

    //生成带表达式的切面切入点，设置其切入点表达式
    AspectJExpressionPointcut ajexp =
               new AspectJExpressionPointcut(candidateAspectClass, new String[0], new Class<?>[0]);
    ajexp.setExpression(aspectJAnnotation.getPointcutExpression());
    if (this.beanFactory != null) {
        ajexp.setBeanFactory(this.beanFactory);
    }
    return ajexp;
}

```

`Advisor` 实例化过程如下：

```
public InstantiationModelAwarePointcutAdvisorImpl(AspectJExpressionPointcut declaredPointcut,
        Method aspectJAdviceMethod, AspectJAdvisorFactory aspectJAdvisorFactory,
        MetadataAwareAspectInstanceFactory aspectInstanceFactory,
        int declarationOrder, String aspectName) {
    // 处理属性设置
    this.declaredPointcut = declaredPointcut;
    this.declaringClass = aspectJAdviceMethod.getDeclaringClass();
    this.methodName = aspectJAdviceMethod.getName();
    this.parameterTypes = aspectJAdviceMethod.getParameterTypes();
    this.aspectJAdviceMethod = aspectJAdviceMethod;
    this.aspectJAdvisorFactory = aspectJAdvisorFactory;
    this.aspectInstanceFactory = aspectInstanceFactory;
    this.declarationOrder = declarationOrder;
    this.aspectName = aspectName;

    if (aspectInstanceFactory.getAspectMetadata().isLazilyInstantiated()) {
        ...
    }
    else {
        this.pointcut = this.declaredPointcut;
        this.lazy = false;
        //初始化对应的增强器，这里是重点
        this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
    }
}

/**
 * 实例化过程
 */
private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
    // getAdvice: 处理 Advice 实例化操作
    Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
            this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
    return (advice != null ? advice : EMPTY_ADVICE);
}

```

这段代码清晰明了，主要是实例化 `Advisor`，所谓的实例化，就是往 `Advisor` 对象里设置了一些属性。这里 `instantiatedAdvice` 属性需要特别说明一下，这个属性封装了切面方法，也就是增强的内容。

*   切面方法 (切面具体的执行代码) 封装为 `Advice`；
*   `Advice` 是 `Advisor` 的一个属性。

#### 2.4 实例化 `Advice`

接下来看看 `Advice` 的流程：

> ReflectiveAspectJAdvisorFactory#getAdvice

```
@Override
@Nullable
public Advice getAdvice(Method candidateAdviceMethod, AspectJExpressionPointcut expressionPointcut,
        MetadataAwareAspectInstanceFactory aspectInstanceFactory,
        int declarationOrder, String aspectName) {

    Class<?> candidateAspectClass = aspectInstanceFactory.getAspectMetadata().getAspectClass();
    //又是一次校验
    validate(candidateAspectClass);

    AspectJAnnotation<?> aspectJAnnotation =
            AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(candidateAdviceMethod);
    if (aspectJAnnotation == null) {
        return null;
    }

    if (!isAspect(candidateAspectClass)) {
        throw new AopConfigException(...);
    }

    AbstractAspectJAdvice springAdvice;
    // 根据注解类型生成不同的通知实例，
    // 对应的注解就是 @Before, @Around, @After, @AfterReturning, @AfterThrowing
    switch (aspectJAnnotation.getAnnotationType()) {
        case AtPointcut:
            return null;
        case AtAround:
            springAdvice = new AspectJAroundAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtBefore:
            springAdvice = new AspectJMethodBeforeAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtAfter:
            springAdvice = new AspectJAfterAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            break;
        case AtAfterReturning:
            springAdvice = new AspectJAfterReturningAdvice(
                    candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterReturning afterReturningAnnotation = (AfterReturning) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterReturningAnnotation.returning())) {
                springAdvice.setReturningName(afterReturningAnnotation.returning());
            }
            break;
        case AtAfterThrowing:
            springAdvice = new AspectJAfterThrowingAdvice(
                   candidateAdviceMethod, expressionPointcut, aspectInstanceFactory);
            AfterThrowing afterThrowingAnnotation = (AfterThrowing) aspectJAnnotation.getAnnotation();
            if (StringUtils.hasText(afterThrowingAnnotation.throwing())) {
                springAdvice.setThrowingName(afterThrowingAnnotation.throwing());
            }
            break;
        default:
            throw new UnsupportedOperationException(...);
    }

    //设置通知方法所属的类
    springAdvice.setAspectName(aspectName);
    //设置通知的序号,同一个类中有多个切面注解标识的方法时,按上方说的排序规则来排序，
    //其序号就是此方法在列表中的序号，第一个就是0
    springAdvice.setDeclarationOrder(declarationOrder);
    //获取通知方法的所有参数
    String[] argNames = this.parameterNameDiscoverer.getParameterNames(candidateAdviceMethod);
    //将通知方法上的参数设置到通知中
    if (argNames != null) {
        springAdvice.setArgumentNamesFromStringArray(argNames);
    }
    //计算参数绑定工作，此方法详解请接着往下看
    springAdvice.calculateArgumentBindings();
    return springAdvice;
}

```

我们知道，切面注解标识的方法第一个参数要求是 `JoinPoint`，如果是 `@Around` 注解，则第一个参数可以是 `ProceedingJoinPoint`，计算参数绑定就是来验证参数类型的。

对于不同的通知，spring 会封装成不同的 `advice`，这里先来看看 `advice` 的继承结构：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-671bd92a16642a059ff722ec42c8ea7ef28.png)

关于 `advice`，后面在分析切面的执行时会详细分析，这里只需知道 “不同的切面类型，最终会封装为不同的 advice” 即可。

再来看看 spring 计算参数绑定的流程：

> AbstractAspectJAdvice

```
/**
 * 处理参数绑定
 */
public final synchronized void calculateArgumentBindings() {
    if (this.argumentsIntrospected || this.parameterTypes.length == 0) {
        return;
    }

    int numUnboundArgs = this.parameterTypes.length;
    Class<?>[] parameterTypes = this.aspectJAdviceMethod.getParameterTypes();
    //切面注解标识的方法第一个参数要求是JoinPoint,或StaticPart，若是@Around注解则也可以是ProceedingJoinPoint
    if (maybeBindJoinPoint(parameterTypes[0]) || maybeBindProceedingJoinPoint(parameterTypes[0]) ||
            maybeBindJoinPointStaticPart(parameterTypes[0])) {
        numUnboundArgs--;
    }

    if (numUnboundArgs > 0) {
        //绑定属性其他属性
        bindArgumentsByName(numUnboundArgs);
    }

    this.argumentsIntrospected = true;
}

```

以上代码验证了切面注解标识的方法第一个参数的类型，此外，如果在方法中定义了多个方法，spring 还会进一步进行参数绑定：

> AbstractAspectJAdvice

```
/**
  * 绑定属性其他属性
  */
private void bindArgumentsByName(int numArgumentsExpectingToBind) {
    if (this.argumentNames == null) {
        //获取方法参数的名称
        this.argumentNames = createParameterNameDiscoverer()
                .getParameterNames(this.aspectJAdviceMethod);
    }
    if (this.argumentNames != null) {
        // 继续绑定
        bindExplicitArguments(numArgumentsExpectingToBind);
    }
    else {
        throw new IllegalStateException(...);
    }
}

/**
 * 针对 @AfterReturning 与 @AfterThrowing 注解，进一步处理其参数
 */
private void bindExplicitArguments(int numArgumentsLeftToBind) {
    Assert.state(this.argumentNames != null, "No argument names available");
    //此属性用来存储方法未绑定的参数名称，及参数的序号
    this.argumentBindings = new HashMap<>();

    int numExpectedArgumentNames = this.aspectJAdviceMethod.getParameterCount();
    if (this.argumentNames.length != numExpectedArgumentNames) {
        throw new IllegalStateException(...);
    }

    // argumentIndexOffset代表第一个未绑定参数的顺序
    int argumentIndexOffset = this.parameterTypes.length - numArgumentsLeftToBind;
    for (int i = argumentIndexOffset; i < this.argumentNames.length; i++) {
        //存储未绑定的参数名称及其顺序的映射关系
        this.argumentBindings.put(this.argumentNames[i], i);
    }

    // 如果是@AfterReturning注解的returningName 有值，验证，解析，同时得到定义返回值的类型
    if (this.returningName != null) {
        if (!this.argumentBindings.containsKey(this.returningName)) {
            throw new IllegalStateException(...);
        }
        else {
            Integer index = this.argumentBindings.get(this.returningName);
            this.discoveredReturningType = this.aspectJAdviceMethod.getParameterTypes()[index];
            this.discoveredReturningGenericType = this.aspectJAdviceMethod
                        .getGenericParameterTypes()[index];
        }
    }
    // 如果是@AfterThrowing注解的throwingName 有值，验证，解析，同时得到抛出异常的类型
    if (this.throwingName != null) {
        if (!this.argumentBindings.containsKey(this.throwingName)) {
            throw new IllegalStateException(...);
        }
        else {
            Integer index = this.argumentBindings.get(this.throwingName);
            this.discoveredThrowingType = this.aspectJAdviceMethod.getParameterTypes()[index];
        }
    }

    configurePointcutParameters(this.argumentNames, argumentIndexOffset);
}

/**
 *  这一步仅是将前面未处理过的参数做一个保存，记录到 pontcut 中，待后续再做处理
 */
private void configurePointcutParameters(String[] argumentNames, int argumentIndexOffset) {
    int numParametersToRemove = argumentIndexOffset;
    if (this.returningName != null) {
        numParametersToRemove++;
    }
    if (this.throwingName != null) {
        numParametersToRemove++;
    }
    String[] pointcutParameterNames = new String[argumentNames.length - numParametersToRemove];
    Class<?>[] pointcutParameterTypes = new Class<?>[pointcutParameterNames.length];
    Class<?>[] methodParameterTypes = this.aspectJAdviceMethod.getParameterTypes();

    int index = 0;
    for (int i = 0; i < argumentNames.length; i++) {
        if (i < argumentIndexOffset) {
            continue;
        }
        if (argumentNames[i].equals(this.returningName) ||
            argumentNames[i].equals(this.throwingName)) {
            continue;
        }
        pointcutParameterNames[index] = argumentNames[i];
        pointcutParameterTypes[index] = methodParameterTypes[i];
        index++;
    }
    //剩余的未绑定的参数会赋值给AspectJExpressionPointcut(表达式形式的切入点)的属性，以备后续使用
    this.pointcut.setParameterNames(pointcutParameterNames);
    this.pointcut.setParameterTypes(pointcutParameterTypes);
}

```

### 3\. 总结

本文主要是分析了 `AnnotationAwareAspectJAutoProxyCreator#postProcessBeforeInitialization` 方法，也就是 `AbstractAutoProxyCreator#postProcessBeforeInitialization` 方法，在该方法的运行过程中，主要是将 `@Aspect` 类封装成一个个 `Advisor`，封装步骤如下：

1.  找到项目中标注了 `@Aspect` 的类；
2.  遍历上一步得到的类，通过反射得到该类的方法，包括父接口的默认方法、父类的方法但不包括 Object 类的方法；
3.  从上述方法中，找到标注了 `@Around`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing` 等的方法，将其封装为 `Advisor` 对象。

关于 `Advisor`，其中有个重要属性为 `Advice`，这个属性就是切面方法的具体内容。关于 `Advice`，在后面分析切面的执行时会详细分析。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4678817](https://my.oschina.net/funcy/blog/4678817) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_