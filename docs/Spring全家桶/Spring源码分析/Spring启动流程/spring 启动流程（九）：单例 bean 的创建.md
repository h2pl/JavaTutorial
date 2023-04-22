![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1f1ac8f3d8241fad9693d9684048ab7f3ae.png)

接上文，本文依旧是分析 `finishBeanFactoryInitialization(beanFactory)`，本文将重点分析单例 bean 的创建流程。

在上一篇文章中，我们介绍了 `AbstractApplicationContext#finishBeanFactoryInitialization` 的执行过程，本文将深入细节，分析 spring bean 的创建过程。

```
|-AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#finishBeanFactoryInitialization
   |-DefaultListableBeanFactory#preInstantiateSingletons
    |-AbstractBeanFactory#getBean(java.lang.String)
     |-AbstractBeanFactory#doGetBean
      |-AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])

```

我们直接看 `AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])`，精简后的代码如下：

```
@Override
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
    	throws BeanCreationException {

    RootBeanDefinition mbdToUse = mbd;

    // 确保 BeanDefinition 中的 Class 被加载
    Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
    if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
        mbdToUse = new RootBeanDefinition(mbd);
        mbdToUse.setBeanClass(resolvedClass);
    }

    // 准备方法覆写，如果bean中定义了 <lookup-method /> 和 <replaced-method />
    try {
        mbdToUse.prepareMethodOverrides();
    }
    catch (BeanDefinitionValidationException ex) {
        ...
    }

    try {
        // 如果有代理的话直接返回
        Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
        if (bean != null) {
            return bean;
        }
    }
    catch (Throwable ex) {
        ...
    }

    try {
        // 创建 bean
        Object beanInstance = doCreateBean(beanName, mbdToUse, args);
        return beanInstance;
    }
    catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
        ...
    }
    catch (Throwable ex) {
        ...
    }
}

```

可以看到，该方法做了四件事：

1.  确保 class 被加载
2.  覆写方法
3.  查找代理对象，若有则返回
4.  创建 spring bean

对于前面三个操作，本文不关注，我们仅看第四个操作。进入 `AbstractAutowireCapableBeanFactory#doCreateBean`:

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, 
          final @Nullable Object[] args)  throws BeanCreationException {

    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        //如果是 factoryBean则从缓存删除
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
         // 实例化 Bean，这个方法里面才是终点
         instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    //bean实例
    final Object bean = instanceWrapper.getWrappedInstance();
    //bean类型
    Class<?> beanType = instanceWrapper.getWrappedClass();
    if (beanType != NullBean.class) {
        mbd.resolvedTargetType = beanType;
    }

    synchronized (mbd.postProcessingLock) {
        if (!mbd.postProcessed) {
            try {
                // 循环调用MergedBeanDefinitionPostProcessor的postProcessMergedBeanDefinition方法。例如，
                // 1\. 调用 AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition 为了找到
                //  带有@Autowired、@Value 注解的属性和方法
                // 2\. 调用 CommonAnnotationBeanPostProcessor.postProcessMergedBeanDefinition 为了找到
                // 	带有@Resource 注解的属性和方法
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                ...
            }
            mbd.postProcessed = true;
        }
    }

    // 解决循环依赖问题, 是否允许循环依赖, allowCircularReferences默认为true，可以关闭
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 循环依赖的解决，后面会单独分析
        ...
    }

    Object exposedObject = bean;
    try {
        // 负责属性装配, 也就是我们常常说的自动注入，很重要
        populateBean(beanName, mbd, instanceWrapper);
        // 这里是处理bean初始化完成后的各种回调，
        // 例如init-method、InitializingBean 接口、BeanPostProcessor 接口
        exposedObject = initializeBean(beanName, exposedObject, mbd);
    }
    catch (Throwable ex) {
        ...
    }

    //同样的，如果存在循环依赖
    if (earlySingletonExposure) {
        // 循环依赖的解决，后面会单独分析
        ...
    }

    // 把bean注册到相应的Scope中
    try {
        registerDisposableBeanIfNecessary(beanName, bean, mbd);
    }
    catch (BeanDefinitionValidationException ex) {
        ...
    }

    return exposedObject;
}

```

可以看到，spring 在创建 bean 时，主要做了如下三件事：

1.  创建实例
2.  查找属性
3.  注入属性
4.  初始化 bean

接下来，咱们就主要分析这四个步骤。

#### 1\. 创建实例

spring 创建实例的方法是 `AbstractAutowireCapableBeanFactory#createBeanInstance`，内容如下：

```
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, 
        @Nullable Object[] args) {
    // 确保已经加载了此 class
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    // 校验类的访问权限
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) 
                && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(...);
    }

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
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
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
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
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

从以上代码来看，spring 在实例化 bean 时，有 4 种方式：

1.  使用实例化方法：obtainFromSupplier
2.  使用工厂方法：instantiateUsingFactoryMethod
3.  使用有参构造：autowireConstructor
4.  使用无参构造：instantiateBean

对于前面两个方法，这是开发者指定的方法，没啥好说的，后面两个方法是 spring 根据类的实际情况来选择构造方法进行实例化的。对于一个 bean 来说，如果未提供任何构造方法或提供了无参构造方法，则调用 `AbstractAutowireCapableBeanFactory#instantiateBean` 来进行实例化，否则调用 `AbstractAutowireCapableBeanFactory#autowireConstructor` 进行实例化。实际上，这两个方法最终都会执行到 `BeanUtils#instantiateClass(Constructor<T>, Object...)`:

```
public static <T> T instantiateClass(Constructor<T> ctor, Object... args) 
            throws BeanInstantiationException {
    Assert.notNull(ctor, "Constructor must not be null");
    try {
        ReflectionUtils.makeAccessible(ctor);
        if (KotlinDetector.isKotlinReflectPresent() 
                   && KotlinDetector.isKotlinType(ctor.getDeclaringClass())) {
            return KotlinDelegate.instantiateClass(ctor, args);
        }
        else {
            Class<?>[] parameterTypes = ctor.getParameterTypes();
            Assert.isTrue(args.length <= parameterTypes.length, "...");
            Object[] argsWithDefaultValues = new Object[args.length];
            for (int i = 0 ; i < args.length; i++) {
                if (args[i] == null) {
                    Class<?> parameterType = parameterTypes[i];
                    argsWithDefaultValues[i] = (parameterType.isPrimitive() 
                                        ? DEFAULT_TYPE_VALUES.get(parameterType) : null);
                }
                else {
                    argsWithDefaultValues[i] = args[i];
                }
            }
            // 这里进行实例化
            return ctor.newInstance(argsWithDefaultValues);
        }
    }
    catch (...) {
        ...
    }
}

```

可以看到，最终调用的是 `java.lang.reflect.Constructor#newInstance`，这就是 jdk 的反射了。

另外，当 bean 提供了多 个构造方法时，spring 会进行一套复杂的推断机制，推断出最佳的构造方法，这里就不展开了。

#### 2\. 查找属性

对象创建后，接下来就是进行属性注入了，不过在注入属性前，需要知道类中有哪些属性需要注入。spring 属性查找是在 `AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors` 中调用后置处理器进行操作的：

```
protected void applyMergedBeanDefinitionPostProcessors(RootBeanDefinition mbd, 
        Class<?> beanType, String beanName) {
    for (BeanPostProcessor bp : getBeanPostProcessors()) {
        if (bp instanceof MergedBeanDefinitionPostProcessor) {
            // 调用后置处理器
            MergedBeanDefinitionPostProcessor bdp = (MergedBeanDefinitionPostProcessor) bp;
            bdp.postProcessMergedBeanDefinition(mbd, beanType, beanName);
        }
    }
}

```

在这些后置处理器中，

1.  调用 `AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition` 查找带有 `@Autowired`、`@Value` 注解的属性和方法
2.  调用 `CommonAnnotationBeanPostProcessor.postProcessMergedBeanDefinition` 查找带有 `@Resource` 注解的属性和方法

这里为了方便说明 `@Autowired` 的注入流程，我们在 demo01 的 `org.springframework.learn.demo01.BeanObj1` 添加两行行代码：

```
package org.springframework.learn.demo01;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class BeanObj1 {

    // 以下两行为新加入的代码
    @Autowired
    private BeanObj2 beanObj2;

    public BeanObj1() {
        System.out.println("调用beanObj1的构造方法");
    }

    @Override
    public String toString() {
        return "BeanObj1{}";
    }
}

```

进入 `AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition` 方法：

```
@Override
public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, 
        Class<?> beanType, String beanName) {
    // 查找属性与方法
    InjectionMetadata metadata = findAutowiringMetadata(beanName, beanType, null);
    // 验证查找到的属性与方法，并注册到beanDefinition
    metadata.checkConfigMembers(beanDefinition);
}

```

一路跟进 `AutowiredAnnotationBeanPostProcessor#findAutowiringMetadata` 方法，跟到了 `AutowiredAnnotationBeanPostProcessor#buildAutowiringMetadata`:

```
private InjectionMetadata buildAutowiringMetadata(final Class<?> clazz) {
     ...
    // 这里跑了一个循环，找完当前类找当前类的父类，直到Object类
    do {
        final List<InjectionMetadata.InjectedElement> currElements = new ArrayList<>();

        // 查找属性
        ReflectionUtils.doWithLocalFields(targetClass, field -> {
            MergedAnnotation<?> ann = findAutowiredAnnotation(field);
            if (ann != null) {
                if (Modifier.isStatic(field.getModifiers())) {
                    return;
                }
                boolean required = determineRequiredStatus(ann);
                // 属性封装成 AutowiredFieldElement
                currElements.add(new AutowiredFieldElement(field, required));
            }
        });

        // 查找方法
        ReflectionUtils.doWithLocalMethods(targetClass, method -> {
            Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
            if (!BridgeMethodResolver.isVisibilityBridgeMethodPair(method, bridgedMethod)) {
                return;
            }
            MergedAnnotation<?> ann = findAutowiredAnnotation(bridgedMethod);
            if (ann != null && method.equals(ClassUtils.getMostSpecificMethod(method, clazz))) {
                if (Modifier.isStatic(method.getModifiers())) {
                    return;
                }
                if (method.getParameterCount() == 0) {
                }
                boolean required = determineRequiredStatus(ann);
                PropertyDescriptor pd = BeanUtils.findPropertyForMethod(bridgedMethod, clazz);
                // 方法封装成 AutowiredMethodElement
                currElements.add(new AutowiredMethodElement(method, required, pd));
            }
        });

        elements.addAll(0, currElements);
        targetClass = targetClass.getSuperclass();
    }
    while (targetClass != null && targetClass != Object.class);

    return InjectionMetadata.forElements(elements, clazz);
}

```

以上代码就是查找过程，总结如下：

*   循环查询父类，从当前类开始，找完当前类就找当前类的父类，直接 Object 为止；
*   查找属性与方法，属性找到后封装为 `AutowiredFieldElement`，方法找到后封装为 `AutowiredMethodElement`；
*   无论是查找属性还是方法，使用的都是 `findAutowiredAnnotation` 方法

接下来，我们就看看 `findAutowiredAnnotation` 方法是如何进行查找的。进入 `AutowiredAnnotationBeanPostProcessor#findAutowiredAnnotation` 方法，内容如下：

```
private final Set<Class<? extends Annotation>> autowiredAnnotationTypes 
        = new LinkedHashSet<>(4);

// 默认构造方法中指定了@Autowired、@Value注解
@SuppressWarnings("unchecked")
public AutowiredAnnotationBeanPostProcessor() {
    this.autowiredAnnotationTypes.add(Autowired.class);
    this.autowiredAnnotationTypes.add(Value.class);
    ...
}

// 这里就是查找方法
@Nullable
private MergedAnnotation<?> findAutowiredAnnotation(AccessibleObject ao) {
    MergedAnnotations annotations = MergedAnnotations.from(ao);
    // autowiredAnnotationTypes 包含了 @Autowired、@Value注解
    for (Class<? extends Annotation> type : this.autowiredAnnotationTypes) {
        MergedAnnotation<?> annotation = annotations.get(type);
        if (annotation.isPresent()) {
            return annotation;
        }
    }
    return null;
}

```

以上代码注解得很明白了，这里就不多说了。

找到属性与方法后，接着就是注册到 `beanDefinition` 了，这是 `InjectionMetadata#checkConfigMembers` 所做的工作：

```
public void checkConfigMembers(RootBeanDefinition beanDefinition) {
    Set<InjectedElement> checkedElements = new LinkedHashSet<>(this.injectedElements.size());
    for (InjectedElement element : this.injectedElements) {
        Member member = element.getMember();
        if (!beanDefinition.isExternallyManagedConfigMember(member)) {
            // 注册到 beanDefinition
            beanDefinition.registerExternallyManagedConfigMember(member);
            checkedElements.add(element);
        }
    }
    this.checkedElements = checkedElements;
}

```

所谓的注册，其实也相当简单，就是往 `beanDefinition` 持有的 `externallyManagedConfigMembers` 集合中添加一条记录：

> RootBeanDefinition#registerExternallyManagedConfigMember

```
public void registerExternallyManagedConfigMember(Member configMember) {
    synchronized (this.postProcessingLock) {
        if (this.externallyManagedConfigMembers == null) {
            this.externallyManagedConfigMembers = new HashSet<>(1);
        }
        // 往set集合中添加一条记录
        this.externallyManagedConfigMembers.add(configMember);
    }
}

```

#### 3\. 注入属性

spring 的属性注入是在方法 `AbstractAutowireCapableBeanFactory#populateBean` 中处理的，内容如下：

```
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    if (bw == null) {
        if (mbd.hasPropertyValues()) {
            throw new BeanCreationException(...);
        }
        else {
            return;
        }
    }

    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                // 调用具体的后置处理器完成属性填充
                // AutowiredAnnotationBeanPostProcessor.postProcessProperties 处理 @Autowired、@Value 注解
                // CommonAnnotationBeanPostProcessor.postProcessProperties 处理 @Resource注解
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                // 如果返回 false，代表不需要进行后续的属性设值，也不需要再经过其他的 BeanPostProcessor 的处理
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    return;
                }
            }
        }
    }

    // bean的所有属性
    PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    int resolvedAutowireMode = mbd.getResolvedAutowireMode();
    // 属性注入类型为byType还是byName，注：@Autowired 既不是byType也不是byName,因此这里返回false
    if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
        // 通过名字找到所有属性值，如果是 bean 依赖，先初始化依赖的 bean。记录依赖关系
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }
        // 通过类型装配。复杂一些
        if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }
        pvs = newPvs;
    }

    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

    PropertyDescriptor[] filteredPds = null;
    if (hasInstAwareBpps) {
        if (pvs == null) {
            pvs = mbd.getPropertyValues();
        }
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
            // 调用后置处理器完成注释的属性填充
            // 处理@Autowired注解的是 AutowiredAnnotationBeanPostProcessor
            InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
            PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
            if (pvsToUse == null) {
                if (filteredPds == null) {
                    filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
                }
                // 这里就是上方曾经提到过得对@Autowired处理的一个BeanPostProcessor了
                // 它会对所有标记@Autowired、@Value 注解的属性进行设值
                pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, 
                        bw.getWrappedInstance(), beanName);
                if (pvsToUse == null) {
                    return;
                }
            }
            pvs = pvsToUse;
            }
        }
    }
    if (needsDepCheck) {
        if (filteredPds == null) {
             filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        }
        checkDependencies(beanName, mbd, filteredPds, pvs);
    }

    if (pvs != null) {
        // 设置 bean 实例的属性值
        applyPropertyValues(beanName, mbd, bw, pvs);
    }
}

```

spring 的属性填充是在后置处理器中进行的，`@Autowired` 的属性填充方法为 `AutowiredAnnotationBeanPostProcessor#postProcessProperties`:

```
public PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) {
     // 调用 findAutowiringMetadata 方法，确保标注 @Autowired @Value 注解的属性与方法获取成功
     // findAutowiringMetadata 可以发现，里面做了一个缓存，只有第一次才会真正去获取，之后都从缓存中获取
     InjectionMetadata metadata = findAutowiringMetadata(beanName, bean.getClass(), pvs);
     try {
         // 这里进行属性注入
         metadata.inject(bean, beanName, pvs);
     }
     catch (...) {
         ...
     }
}

```

我们再进入 `InjectionMetadata#inject`：

```
public void inject(Object target, @Nullable String beanName, @Nullable PropertyValues pvs) 
        throws Throwable {
    Collection<InjectedElement> checkedElements = this.checkedElements;
    Collection<InjectedElement> elementsToIterate =
            (checkedElements != null ? checkedElements : this.injectedElements);
    if (!elementsToIterate.isEmpty()) {
        // 这里的 InjectedElement，就是在 AutowiredAnnotationBeanPostProcessor#findAutowiringMetadata
        // 中查找到的，field 会封装为 AutowiredFieldElement，method 会会封装为 AutowiredMethodElement
        for (InjectedElement element : elementsToIterate) {
            element.inject(target, beanName, pvs);
        }
    }
}

```

我们再跟进 `element.inject(target, beanName, pvs)`，这里调用到的是 `AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject`：

```
@Override
protected void inject(Object bean, @Nullable String beanName, 
                @Nullable PropertyValues pvs) throws Throwable {
        Field field = (Field) this.member;
        Object value;
        if (this.cached) {
            value = resolvedCachedArgument(beanName, this.cachedFieldValue);
        }
        else {
            DependencyDescriptor desc = new DependencyDescriptor(field, this.required);
            desc.setContainingClass(bean.getClass());
            Set<String> autowiredBeanNames = new LinkedHashSet<>(1);
            Assert.state(beanFactory != null, "No BeanFactory available");
            TypeConverter typeConverter = beanFactory.getTypeConverter();
            try {
                // 这一行就是获取注入的bean
                value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter);
            }
            catch (BeansException ex) {
                ...
            }
            ...
        }
        // 通过反射给属性设置值
        if (value != null) {
            ReflectionUtils.makeAccessible(field);
            field.set(bean, value);
        }
    }
}

```

以上代码有点多，但我们只要主要关注两行代码就行了：

1.  获取需要注入的 bean: `value = beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter)`
2.  使用反射设置 bean：`ReflectionUtils.makeAccessible(field);field.set(bean, value);`

对于第 2 点，最终调用的是 jdk 提供的反射方法，没什么好说的。这里我们重点来看看 spring 是如何获取需要注入的 bean。

跟进 `beanFactory.resolveDependency(desc, beanName, autowiredBeanNames, typeConverter)`，这里对不重要的代码仅给出调用链，调用如下：

```
|-AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#finishBeanFactoryInitialization
   |-DefaultListableBeanFactory#preInstantiateSingletons
    |-AbstractBeanFactory#getBean(String)
     |-AbstractBeanFactory#doGetBean
      |-DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)
       |-AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])
        |-AbstractAutowireCapableBeanFactory#doCreateBean
         |-AbstractAutowireCapableBeanFactory#populateBean
          |-AutowiredAnnotationBeanPostProcessor#postProcessProperties
           |-InjectionMetadata#inject
            |-AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject
             |-DefaultListableBeanFactory#resolveDependency
              |-DefaultListableBeanFactory#doResolveDependency

```

最终运行到 `DefaultListableBeanFactory#doResolveDependency`:

```
@Nullable
public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
         @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) 
         throws BeansException {

    InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
    try {
        ...
        // 1\.  根据类型查找所有的beanClass，map中保存的内容为：beanName -> Class
        Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
           ...
        String autowiredBeanName;
        Object instanceCandidate;
        // 2\. 如果找到的bean有多个，即数量大于1
        if (matchingBeans.size() > 1) {
            // 则获取注入的属性名
            autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
            if (autowiredBeanName == null) {
                if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
                    return descriptor.resolveNotUnique(descriptor.getResolvableType(), matchingBeans);
                }
                else {
                    return null;
                }
            }
            // 3\. 根据注入的属性名从 根据类型找出来的map<beanName, Class>当中找，
            // 得到bean的Class对象
            instanceCandidate = matchingBeans.get(autowiredBeanName);
        }
        else {
            Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
            autowiredBeanName = entry.getKey();
            instanceCandidate = entry.getValue();
        }
        if (autowiredBeanNames != null) {
            autowiredBeanNames.add(autowiredBeanName);
        }
        if (instanceCandidate instanceof Class) {
            // 根据class 获取 bean，如果bean不存在，则会创建bean
            instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
        }
        Object result = instanceCandidate;
        if (result instanceof NullBean) {
            if (isRequired(descriptor)) {
                raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
            }
            result = null;
        }
        if (!ClassUtils.isAssignableValue(type, result)) {
            throw new BeanNotOfRequiredTypeException(...);
        }
        return result;
    }
    finally {
        ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
    }
}

```

对于 bean 的获取，这里再总结下：

1.  根据 bean 类型从 spring 中获取 bean 的 Class 对象，注意：这里得到的对象为 Class，且获取到的 Class 对象从 spring 中获取 beanClass 可能有多个；
2.  如果得到的 Class 对象有多个，则获取注入的属性名，根据属性名查找从第 1 步得到的 Class 对象中查找符合条件的 1 个 Class 对象；
3.  根据 Class 对象从 spring 中获取 bean.

例如，现在接口 `Inter`、实现类 `A` 与 `B`，三者关系如下：

```
interface Inter {

}

@Service
class A implements Inter {

}

@Service
class B implements Inter {

}

```

在类 `C` 中，注入 `Inter` 类型：

```
@Service
class C {
    @Autowired
    private Inter b;
}

```

在注入时，

1.  spring 会通过 `Inter.class` 查找，得到两个类：`A.class` 与 `B.class`，出现了多个 bean Class 对象；
2.  得到注入的属性名，这里是 `b`，然后使用 `b` 在以上得到的 bean Class 对象找到符合的 bean Class 对象，显然，`B.class` 符合；
3.  从 spring 容器中获取 `B.class` 对应的 bean.

这里会有个小问题：如果在 `C` 中这样注入:

```
@Service
class C {
    @Autowired
    private Inter inter;
}

```

即 `Inter` 的属性名为 `inter`，而 spring 中又没有名称为 `inter` 的 bean Class 对象，这样能注入成功吗？

事实上，这样注入不会成功，且 spring 会报异常，感兴趣的小伙伴可以自己试试。

最后，我们再来看看如何根据 class 从 spring 中获取对应的 bean，即 `instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this)`，跟进去：

> DependencyDescriptor#resolveCandidate

```
public Object resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory)
        throws BeansException {
    // 调用的上 AbstractBeanFactory#getBean(String)
    return beanFactory.getBean(beanName);
}

```

这个方法相当简单，就只有一行：`beanFactory.getBean(beanName)`，回顾下调用链：

```
|-AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#finishBeanFactoryInitialization
   |-DefaultListableBeanFactory#preInstantiateSingletons
    // 这里第一次调用了AbstractBeanFactory#getBean
    // 此时传入的参数为beanObj1
    |-AbstractBeanFactory#getBean(String)
     |-AbstractBeanFactory#doGetBean
      |-DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)
       |-AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])
        |-AbstractAutowireCapableBeanFactory#doCreateBean
         |-AbstractAutowireCapableBeanFactory#populateBean
          |-AutowiredAnnotationBeanPostProcessor#postProcessProperties
           |-InjectionMetadata#inject
            |-AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject
             |-DefaultListableBeanFactory#resolveDependency
              |-DefaultListableBeanFactory#doResolveDependency
               |-DependencyDescriptor#resolveCandidate
                // 这里第一次调用了AbstractBeanFactory#getBean，
                // 此时传入的参数为beanObj2
                |-AbstractBeanFactory#getBean(String)
                 .. 下面开始进行`beanObj2`的实例化与初始化操作

```

结合调用链及前面分析可以看出，

*   在获取 `beanObj1` 时第一次调用 `AbstractBeanFactory#getBean(String)`，由于此时 spring 中不存在 `beanObj1`，就开始了 `beanObj1` 的创建；
*   `beanObj1` 实例化完成后，接着就进行属性注入，`beanObj1` 中有个属性 `BeanObj2 beanObj2`，同样调用 `AbstractBeanFactory#getBean(String)` 从 spring 中获取 `beanObj2` 对象；
*   由于此时 spring 中不存在 `beanObj2`，接着就进入了 `beanObj2` 对象的创建过程；创建对象后，同样也会对 `beanObj2` 属性注入、运行初始化方法等操作，这与 `beanObj1` 的操作类似；
*   得到 `beanObj2` 对象后，注入到 `beanObj1` 的 `BeanObj2 beanObj2` 属性中，然后进行 `beanObj1` 的后续操作。

从这里可以看出，属性注入时，可能会使 bean 初始化提前；同时，对于依赖层次多的情况，调用链将会非常长，如 A 需要注入 B，B 要注入 C，那么在 A 的属性注入时，将会初始化 B，而 B 在初始化时，又会初始化 C。

对于 bean 的实例化与初始化，这里需要明确下：

*   实例化：仅指创建对象的操作，可以使用 new 或反射机制，未处理 spring 属性注入及之后的运行初始化方法、保存到 spring 容器等一系列操作；
*   初始化：对实例化后的对象进行属性注入、运行初始化方法，最终保存到 spring 容器；

#### 4\. 初始化 bean

spring 初始化 bean 的方法是 `AbstractAutowireCapableBeanFactory#initializeBean(String, Object, RootBeanDefinition)`:

```
protected Object initializeBean(final String beanName, final Object bean, 
             @Nullable RootBeanDefinition mbd) {
    // 1\. 运行 invokeAwareMethods
    if (System.getSecurityManager() != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
             invokeAwareMethods(beanName, bean);
             return null;
        }, getAccessControlContext());
    }
    else {
        invokeAwareMethods(beanName, bean);
    }

    // 2\. 运行 applyBeanPostProcessorsBeforeInitialization
    Object wrappedBean = bean;
    if (mbd == null || !mbd.isSynthetic()) {
        // 执行 spring 当中的内置处理器——xxxPostProcessor-------@PostConstruct
        wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
    }

    try {
        // 3\. 执行 InitializingBean 初始化
        invokeInitMethods(beanName, wrappedBean, mbd);
    }
    catch (Throwable ex) {
        ...
    }
    if (mbd == null || !mbd.isSynthetic()) {
        // 4\. 运行 applyBeanPostProcessorsAfterInitialization
        wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
    }

    return wrappedBean;
}

```

以上代码主要是用来运行 4 个方法：

1.  invokeAwareMethods(beanName, bean);
2.  applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
3.  invokeInitMethods(beanName, wrappedBean, mbd);
4.  applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);

##### 4.1 调用 Aware bean 的方法

`invokeAwareMethods` 就是执行 Aware bean 的相关方法：

```
private void invokeAwareMethods(final String beanName, final Object bean) {
    if (bean instanceof Aware) {
        if (bean instanceof BeanNameAware) {
            ((BeanNameAware) bean).setBeanName(beanName);
        }
        if (bean instanceof BeanClassLoaderAware) {
            ClassLoader bcl = getBeanClassLoader();
            if (bcl != null) {
                ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
            }
        }
        if (bean instanceof BeanFactoryAware) {
            ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
        }
    }
}

```

以上方法比较简单，就不多说了。

##### 4.2 运行后置处理器的 `postProcessBeforeInitialization` 方法

接着来看看 `AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization` 的运行：

```
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
         throws BeansException {
    Object result = existingBean;
    for (BeanPostProcessor processor : getBeanPostProcessors()) {
        // 调用后置处理器
        Object current = processor.postProcessBeforeInitialization(result, beanName);
        if (current == null) {
            return result;
        }
        result = current;
    }
    return result;
}

```

这里主要运行 `ApplicationContextAwareProcessor#postProcessBeforeInitialization` 方法，内容如下：

```
@Override
@Nullable
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
    if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
              bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
              bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
        return bean;
    }
    AccessControlContext acc = null;
    if (System.getSecurityManager() != null) {
        acc = this.applicationContext.getBeanFactory().getAccessControlContext();
    }
    if (acc != null) {
        AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
           invokeAwareInterfaces(bean);
           return null;
        }, acc);
    }
    else {
        // 运行方法
        invokeAwareInterfaces(bean);
    }
    return bean;
}

private void invokeAwareInterfaces(Object bean) {
    if (bean instanceof EnvironmentAware) {
        ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
    }
    if (bean instanceof EmbeddedValueResolverAware) {
        ((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
    }
    if (bean instanceof ResourceLoaderAware) {
        ((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
    }
    if (bean instanceof ApplicationEventPublisherAware) {
        ((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
    }
    if (bean instanceof MessageSourceAware) {
        ((MessageSourceAware) bean).setMessageSource(this.applicationContext);
    }
    // 装配 实现了ApplicationContextAware的类的 applicationContext
    if (bean instanceof ApplicationContextAware) {
        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
    }
}

```

其实这个方法就是给各种 `XxxAware` 赋值，比如大名鼎鼎的 `ApplicationContextAware` 赋值也是在这里进行。

顺带一提的是，如果在 bean 中，方法上有 `@PostConstruct` 注解：

```
@PostConstruct
public void test() {
    System.out.println("PostConstruct");
}

```

该方法将会在 `InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization` 中进行调用，代码比较简单，也是利用了 jdk 的反射机制，这里就不多说了。

##### 4.3 调用 bean 的初始化方法

接着来看看 `AbstractAutowireCapableBeanFactory#invokeInitMethods` 方法:

```
protected void invokeInitMethods(String beanName, final Object bean, 
        @Nullable RootBeanDefinition mbd)  throws Throwable {
    // 执行 InitializingBean#afterPropertiesSet 方法
    boolean isInitializingBean = (bean instanceof InitializingBean);
    if (isInitializingBean && (mbd == null || 
                 !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
        if (System.getSecurityManager() != null) {
            try {
                AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
                    ((InitializingBean) bean).afterPropertiesSet();
                    return null;
                }, getAccessControlContext());
            }
            catch (PrivilegedActionException pae) {
                throw pae.getException();
            }
        }
        else {
            // 如果bean实现了 InitializingBean 接口，则调用 afterPropertiesSet()
            ((InitializingBean) bean).afterPropertiesSet();
        }
    }

    // 执行自定义的初始化方法，即在xml中，通过 init-method 指定的方法
    if (mbd != null && bean.getClass() != NullBean.class) {
        String initMethodName = mbd.getInitMethodName();
        if (StringUtils.hasLength(initMethodName) &&
                !(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
                !mbd.isExternallyManagedInitMethod(initMethodName)) {
            invokeCustomInitMethod(beanName, bean, mbd);
        }
    }
}

```

这里主要是运行两个方法：

1.  如果 bean 实现了 InitializingBean 接口，则调用 afterPropertiesSet ()
2.  如果指定了 init-method，则在 `invokeCustomInitMethod` 中运行，也是通过 jdk 反射机制来运行的，就不多说了。

##### 4.4 运行后置处理器的 `postProcessAfterInitialization` 方法

这一步主要是运行 `BeanPostProcessor#postProcessAfterInitialization`，所运行的 `BeanPostProcessor` 是 `ApplicationListenerDetector`:

```
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) {
    if (bean instanceof ApplicationListener) {
        Boolean flag = this.singletonNames.get(beanName);
        if (Boolean.TRUE.equals(flag)) {
            this.applicationContext.addApplicationListener((ApplicationListener<?>) bean);
        }
        else if (Boolean.FALSE.equals(flag)) {
            this.singletonNames.remove(beanName);
        }
    }
    return bean;
}

```

这个方法是用来处理 `ApplicationListener` 的，如果有 bean 实现了 `ApplicationListener`，就是在这里添加到 spring 的监听器列表的。

#### 5\. 总结

本文主要介绍了 spring bean 创建过程，相关流程总结如下：

1.  实例化 bean，指定了初始化方法、指定了 `factory method`、自主推断构造方法等方式；

2.  查询待注入的属性或方法，即哪些属性或方法上标注了 `@Autowird`/`@Value`/`@Resource`

    *   `AutowiredAnnotationBeanPostProcessor.postProcessMergedBeanDefinition` 用来查找带有 @Autowired、@Value 注解的属性和方法
    *   `CommonAnnotationBeanPostProcessor.postProcessMergedBeanDefinition` 用来查找带有 @Resource 注解的属性和方法
3.  属性填充

    *   `AutowiredAnnotationBeanPostProcessor.postProcessProperties` 处理 `@Autowired`、`@Value` 注解，`CommonAnnotationBeanPostProcessor.postProcessProperties` 处理 `@Resource` 注解
    *   处理 `@Autowired` 时，先根据类型找到所有的 `Class`，如果有多个，再根据注入属性的名称查找符合的 `Class`，最后调用 `beanFactory.getBean(...)` 从 spring 获取要注入的对象，通过反射机制为对应的属性设置值；
4.  初始化 bean

    1.  执行 Aware bean 的相关方法
    2.  调用 `BeanPostProcessor#postProcessBeforeInitialization` 方法
        *   在 `ApplicationContextAwareProcessor#postProcessBeforeInitialization` 中执行 Aware bean 的相关方法
        *   在 `InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization` 中处理 `@PostConstruct` 注解
    3.  执行初始化方法
        *   执行 `InitializingBean#afterPropertiesSet` 方法
        *   执行自定义的 `init-method` 方法
    4.  运行 `BeanPostProcessor#postProcessAfterInitialization` 方法
        *   在 `ApplicationListenerDetector#postProcessAfterInitialization` 中处理 spring 监听器

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b7a4d4bc1bfbd76537e40cf843b0d18df93.png)

关于 spring bean 的创建流程就先分析到这里了，不过 spring 在创建 bean 的过程中，有一个无法忽视的情况：循环依赖，关于这方面的处理，可以参考：

*   [spring 探秘之循环依赖的解决（一）：理论基石](https://my.oschina.net/funcy/blog/4659555)
*   [spring 探秘之循环依赖的解决（二）：源码分析](https://my.oschina.net/funcy/blog/4815992)

*   * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4659524](https://my.oschina.net/funcy/blog/4659524) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_