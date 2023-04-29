spring aop 执行时，顺序是怎样的，如何改变执行的优先级？本文将从源码上来探究 aop 执行顺序的秘密。

在 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（上）](https://my.oschina.net/funcy/blog/4678817) 与 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)中，纵观 spring aop 创建与执行过程，我们一共遇到两次关于 aop 的排序操作：

*   `ReflectiveAspectJAdvisorFactory#getAdvisorMethods`：对 `aspect` 中的 `@Around/@Before/@After` 等方法进行排序；
*   `AspectJAwareAdvisorAutoProxyCreator#sortAdvisors`：对 `advisor` 进行排序。

接下来我们重点来分析这两个方法。

#### 1 `ReflectiveAspectJAdvisorFactory#getAdvisorMethods`

第一次排序在 `ReflectiveAspectJAdvisorFactory#getAdvisorMethods`，调用结构如下：

```
|-AbstractAutoProxyCreator#postProcessBeforeInstantiation
  |-AspectJAwareAdvisorAutoProxyCreator#shouldSkip
   |-AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors
    |-BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors
     |-ReflectiveAspectJAdvisorFactory#getAdvisors
      |-ReflectiveAspectJAdvisorFactory#getAdvisorMethods

```

代码如下：

> ReflectiveAspectJAdvisorFactory

```
// 获取 @Aspect 类中的方法
private List<Method> getAdvisorMethods(Class<?> aspectClass) {
    final List<Method> methods = new ArrayList<>();
    // 省略获取方法的操作
    ...

    //对得到的所有方法排序，
    methods.sort(METHOD_COMPARATOR);
    return methods;
}

```

这个方法的操作在 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（上）](https://my.oschina.net/funcy/blog/4678817) 一文已作了详细分析，这里我们仅关注排序规则：

> ReflectiveAspectJAdvisorFactory

```
/**
 * METHOD_COMPARATOR 详解
 */
private static final Comparator<Method> METHOD_COMPARATOR;
static {
    Comparator<Method> adviceKindComparator = new ConvertingComparator<>(
            // 比较器，按传入顺序进行比较
            new InstanceComparator<>(
                    Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class),
            // 转换器，将方法转化为 @Around, @Before, @After, @AfterReturning, @AfterThrowing 等注解
            (Converter<Method, Annotation>) method -> {
                AspectJAnnotation<?> annotation =
                    AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(method);
                return (annotation != null ? annotation.getAnnotation() : null);
            });
    // 转换比较器，
    // 1\. 转换：将传入的方法(Method)转换为方法名(String)
    // 2\. 比较：按传入类型进行比较，这里传入的的类型为String，原因是转换器将传入的Method转换成String了
    Comparator<Method> methodNameComparator = new ConvertingComparator<>(Method::getName);
    /*
     * METHOD_COMPARATOR 比较规则：
     * 1\. 如果方法标识了切面注解, 则按 @Around, @Before, @After, @AfterReturning,
     *       @AfterThrowing 顺序排序 (`adviceKindComparator`)
     * 2\. 如果没有标识这些注解，则按方法名称的字符串排序(`methodNameComparator`)
     */
    METHOD_COMPARATOR = adviceKindComparator.thenComparing(methodNameComparator);
}

```

这里对这个方法的排序规则总结如下：

1.  这个方法排序的对象是同一个 `@Aspect` 中的方法；
2.  对于切面方法，排序如下：`@Around`, `@Before`, `@After`, `@AfterReturning`, `@AfterThrowing`；
3.  对于非切面方法，按方法名（String 的排序规则）排序。

该方法排序前后的变化如下：

排序前：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-23df6d6a46f37badb1017ceee8dcfa6533e.png)

排序后：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-85bdb16953c1c6348d39578de6b6144c1cc.png)

排序后与我们分析的 `@Around, @Before, @After, @AfterReturning,@AfterThrowing` 的顺序一致。

得到 `List<Method>` 后，接着会遍历这些 `method`，将其包装为一个个 `advisor`：

> ReflectiveAspectJAdvisorFactory

```
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    // 省略了一些代码
    ...

    List<Advisor> advisors = new ArrayList<>();
    //获取这个类所有的增强方法
    for (Method method : getAdvisorMethods(aspectClass)) {
        // 生成增强实例，advisors.size() 依次为 0，1，2，... 这是
        // declarationOrderInAspect 的值，后面排序会用到
        Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, 
                advisors.size(), aspectName);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    // 省略了一些代码
    ...
}

```

包装成 `Advisor` 时，会传入 `declarationOrderInAspect` 值，该值为 `advisors.size()`，依次为 `0，1，2，...`，这个值在后面的排序中会用到。

### 2\. advisor 的排序：`AspectJAwareAdvisorAutoProxyCreator#sortAdvisors`

`@Aspect` 类中的切面方法包装成 `advisor` 后，或者获取完自定义 `advisor` 后，接着就进行了第二次排序：`AspectJAwareAdvisorAutoProxyCreator#sortAdvisors`，方法的调用链如下：

```
|-AbstractAutoProxyCreator#postProcessAfterInitialization
 |-AbstractAutoProxyCreator#wrapIfNecessary
  |-AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean
   |-AbstractAdvisorAutoProxyCreator#findEligibleAdvisors
    |-AspectJAwareAdvisorAutoProxyCreator#sortAdvisors

```

> AspectJAwareAdvisorAutoProxyCreator

```
protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
    List<PartiallyComparableAdvisorHolder> partiallyComparableAdvisors 
            = new ArrayList<>(advisors.size());
    for (Advisor element : advisors) {
        partiallyComparableAdvisors.add(
            // 比较规则为 DEFAULT_PRECEDENCE_COMPARATOR,其实是AspectJPrecedenceComparator
            new PartiallyComparableAdvisorHolder(element, DEFAULT_PRECEDENCE_COMPARATOR));
    }
    // 具体的比较操作，比较规则由 AspectJPrecedenceComparator 提供
    List<PartiallyComparableAdvisorHolder> sorted 
            = PartialOrder.sort(partiallyComparableAdvisors);
    if (sorted != null) {
        List<Advisor> result = new ArrayList<>(advisors.size());
        for (PartiallyComparableAdvisorHolder pcAdvisor : sorted) {
            result.add(pcAdvisor.getAdvisor());
        }
        return result;
    }
    else {
        return super.sortAdvisors(advisors);
    }
}

```

这里进行了两个排序，一个是 `PartialOrder.sort(...)`，一个是 `super.sortAdvisors(...)`，我们先来分析 `PartialOrder.sort(...)`。

#### 2.1 `PartialOrder.sort(...)` 的比较器：`AspectJPrecedenceComparator`

实际上，`PartialOrder.sort(...)` 只要做了一个排序而已，这个方法没啥分析的，我们真正要分析的应该是传入的排序规则，也就是 `DEFAULT_PRECEDENCE_COMPARATOR`：

```
private static final Comparator<Advisor> DEFAULT_PRECEDENCE_COMPARATOR 
        = new AspectJPrecedenceComparator();

```

`DEFAULT_PRECEDENCE_COMPARATOR` 的类型是 `AspectJPrecedenceComparator`，我们直接查看其 `compare(xxx)` 方法：

> AspectJPrecedenceComparator

```
@Override
public int compare(Advisor o1, Advisor o2) {
    // 比较规则：AnnotationAwareOrderComparator
    int advisorPrecedence = this.advisorComparator.compare(o1, o2);
    // 顺序相同，且来源于同一 aspect，调用 comparePrecedenceWithinAspect 再次比较
    if (advisorPrecedence == SAME_PRECEDENCE && declaredInSameAspect(o1, o2)) {
        // 比较声明顺序，如果其中有一个是after通知，则后声明的优先级高；否则先声明的优先级高
        advisorPrecedence = comparePrecedenceWithinAspect(o1, o2);
    }
    return advisorPrecedence;
}

```

`AspectJPrecedenceComparator#compare` 比较简单，过程如下：

1.  调用 `advisorComparator.compare` 进行比较，这个比较规则我们接下来会分析；
2.  如果由上述比较规则得到的优先级相同，且两个 `advisor` 是在同一 aspect 中定义的，则调用 `comparePrecedenceWithinAspect` 继续比较.

##### `this.advisorComparator.compare`

我们来看看 `this.advisorComparator.compare` 的比较规则：

```
private final Comparator<? super Advisor> advisorComparator;

public AspectJPrecedenceComparator() {
    this.advisorComparator = AnnotationAwareOrderComparator.INSTANCE;
}

public int compare(Advisor o1, Advisor o2) {
    // 比较规则：AnnotationAwareOrderComparator
    int advisorPrecedence = this.advisorComparator.compare(o1, o2);
    ...
}

```

`this.advisorComparator.compare` 的比较规则由 `AnnotationAwareOrderComparator` 提供：

```
public int compare(@Nullable Object o1, @Nullable Object o2) {
    return doCompare(o1, o2, null);
}

/**
 * 具体的比较操作，先比较 PriorityOrdered，再比较 Ordered
 */
private int doCompare(@Nullable Object o1, @Nullable Object o2, 
        @Nullable OrderSourceProvider sourceProvider) {
    // 两者之一为 PriorityOrdered，谁是PriorityOrdered，谁的优先级高
    boolean p1 = (o1 instanceof PriorityOrdered);
    boolean p2 = (o2 instanceof PriorityOrdered);
    if (p1 && !p2) {
        return -1;
    }
    else if (p2 && !p1) {
        return 1;
    }
    // 查找order的值，先查找Ordered接口，如果没找到，再查找 @Order 注解
    int i1 = getOrder(o1, sourceProvider);
    int i2 = getOrder(o2, sourceProvider);
    // 按Integer规则进行比较
    return Integer.compare(i1, i2);
}

```

从上面的代码可知，先比较 `PriorityOrdered`，再比较 `Ordered`，比较规则如下：

1.  `PriorityOrdered` 比较：两者之中，只有其一实现了 `PriorityOrdered` 接口，那么类型为 `PriorityOrdered` 优先级高， 其他情况则按 `Ordered` 的规则比较；
2.  `Ordered` 比较规则：
    1.  如果实现了 `Ordered` 或 `PriorityOrdered` 接口，则根据 `getOrder()` 返回值进行比较，值越小优先级越高；
    2.  如果标注了 `@Order/@Priority` 注解，则根据其 `value()` 返回值进行比较，值越小优先级越高；
    3.  如果没有实现 `Ordered/PriorityOrdered`，也没有标注 `@Order/@Priority` 注解，则为最低优先级 (`Integer.MAX_VALUE`).

##### `comparePrecedenceWithinAspect`

对于 `@Aspect` 标注的类，如果同一 `aspect` 里定义了同样的 `advice`，spring aop 也提供了一套比较规则：

```
/**
 * 针对 @Aspect， 同一aspect里定义了同样的 advice，再次比较
 */
private int comparePrecedenceWithinAspect(Advisor advisor1, Advisor advisor2) {
    boolean oneOrOtherIsAfterAdvice = (AspectJAopUtils.isAfterAdvice(advisor1) 
            || AspectJAopUtils.isAfterAdvice(advisor2));
    int adviceDeclarationOrderDelta = getAspectDeclarationOrder(advisor1) 
            - getAspectDeclarationOrder(advisor2);
    // 其中有一个是after通知，declarationOrderInAspect大的优先级高
    if (oneOrOtherIsAfterAdvice) {
        if (adviceDeclarationOrderDelta < 0) {
            return LOWER_PRECEDENCE;
        }
        else if (adviceDeclarationOrderDelta == 0) {
            return SAME_PRECEDENCE;
        }
        else {
            return HIGHER_PRECEDENCE;
        }
    }
    // 两者都不是after通知，declarationOrderInAspect小的优先级高
    else {
        if (adviceDeclarationOrderDelta < 0) {
            return HIGHER_PRECEDENCE;
        }
        else if (adviceDeclarationOrderDelta == 0) {
            return SAME_PRECEDENCE;
        }
        else {
            return LOWER_PRECEDENCE;
        }
    }
}

```

比较规则如下：比较两者的 `declarationOrderInAspect` 值，如果两者之一为 `after` 通知，`declarationOrderInAspect` 大的优先级高；如果两者都不是 `after` 通知，`declarationOrderInAspect` 小的优先级高。

这里的 `declarationOrderInAspect` 是什么呢？这是上一小节提到的 `advisor.size()`，代码如下：

> ReflectiveAspectJAdvisorFactory

```
public List<Advisor> getAdvisors(MetadataAwareAspectInstanceFactory aspectInstanceFactory) {
    // 省略了一些代码
    ...

    List<Advisor> advisors = new ArrayList<>();
    //获取这个类所有的增强方法
    for (Method method : getAdvisorMethods(aspectClass)) {
        // 生成增强实例，advisors.size() 依次为 0，1，2，... 这是
        // declarationOrderInAspect 的值，后面排序会用到
        Advisor advisor = getAdvisor(method, lazySingletonAspectInstanceFactory, 
                advisors.size(), aspectName);
        if (advisor != null) {
            advisors.add(advisor);
        }
    }

    // 省略了一些代码
    ...
}

```

特别强调的是，这个规则只适用于同一 `@Aspect` 类定义的、同样的通知方法，如：

```
@Aspect
public class AspectTest {
    @Before
    public void before1() {
        ...
    }

    @Before
    public void before2() {
        ...
    }

}

```

这里的 `before1()` 与 `before2()` 对应的 `advisor` 适用于 `comparePrecedenceWithinAspect` 排序，而以下代码就不适用了，原因是在不同的 `@Aspect` 类中定义的：

```
@Aspect
public class AspectTest1 {
    @Before
    public void before() {
        ...
    }

}

@Aspect
public class AspectTest2 {
    @Before
    public void before() {
        ...
    }

}

```

#### 2. `super.sortAdvisors`

我们再回过头来看 `super.sortAdvisors(advisors)`:

> AspectJAwareAdvisorAutoProxyCreator

```
protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
    ...
    else {
        return super.sortAdvisors(advisors);
    }
}

```

我们跟进去：

> AbstractAdvisorAutoProxyCreator

```
 protected List<Advisor> sortAdvisors(List<Advisor> advisors) {
     AnnotationAwareOrderComparator.sort(advisors);
     return advisors;
 }

```

发现最后使用的是 `AnnotationAwareOrderComparator.sort(advisors)`，实际上，这个就是上面分析的 `this.advisorComparator.compare` 的比较规则，这里就不再分析了。

### 3. `getOrder()` 值的由来

#### `BeanFactoryTransactionAttributeSourceAdvisor#getOrder()`

`BeanFactoryTransactionAttributeSourceAdvisor` 没有 `@Order/@Priority`，但它实现了 `Ordered` 接口，因此它的执行顺序由 `getOrder()` 方法的返回值决定，对应的 `getOrder()` 方法如下：

```
    /**
     * 获取 order，方法如下：
     * 1\. 如果已指定了 order，直接返回；
     * 2\. 获取 advisor 的 advice，如果 advice 实现了 Ordered 接口，调用 getOrder()；
     * 3\. 如果以上都不满足，则返回 Ordered.LOWEST_PRECEDENCE (最低优先级)。
     * @return
     */
    @Override
    public int getOrder() {
        if (this.order != null) {
            return this.order;
        }
        Advice advice = getAdvice();
        if (advice instanceof Ordered) {
            return ((Ordered) advice).getOrder();
        }
        return Ordered.LOWEST_PRECEDENCE;
    }

```

`Ordered.LOWEST_PRECEDENCE` 为 `Integer.MAX_VALUE`，即 `2147483647`，我们再看看 `BeanFactoryTransactionAttributeSourceAdvisor` 的 `getOrder()` 方法返回的值：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d05044b7dcd41855ab6e59727c4be74bdb9.png)

可见，`BeanFactoryTransactionAttributeSourceAdvisor` 的执行顺序是默认的 `Integer.MAX_VALUE`。如果调度的话，发现这个值是在 `return this.order` 返回的：

```
public int getOrder() {
    // 通过调度发现，this.order 并不为null
    if (this.order != null) {
        return this.order;
    }
    // 省略一些代码
    ...
}

```

那么这值是从哪里来的呢？经过重重分析，发现是在创建 `BeanFactoryTransactionAttributeSourceAdvisor` 对象时，调用 `BeanFactoryTransactionAttributeSourceAdvisor#setOrder` 设置的：

> ProxyTransactionManagementConfiguration

```
@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
    BeanFactoryTransactionAttributeSourceAdvisor advisor 
            = new BeanFactoryTransactionAttributeSourceAdvisor();
    advisor.setTransactionAttributeSource(transactionAttributeSource());
    advisor.setAdvice(transactionInterceptor());
    if (this.enableTx != null) {
        // 这里获取的是 @EnableTransactionManagement 注解的 order() 值
        advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
    }
    return advisor;
}

```

到这里我们就明白了，事务 advisor 的执行顺序可以在 `@EnableTransactionManagement` 中指定：

```
public @interface EnableTransactionManagement {

    boolean proxyTargetClass() default false;

    AdviceMode mode() default AdviceMode.PROXY;

    /**
     * 这里指定advisor执行顺序，默认是最低优先级
     */
    int order() default Ordered.LOWEST_PRECEDENCE;

}

```

结论：`@EnableTransactionManagement` 注解的 `order()` 方法可以指定 `advisor` 的执行顺序。

#### `InstantiationModelAwarePointcutAdvisorImpl#getOrder()`

从前面的分析可知，`@Aspect` 类中的每一个方法最终都会转化为 `advisor`，类型为 `InstantiationModelAwarePointcutAdvisorImpl`，它也实现了 `Ordered` 接口，因此执行顺序也是由 `InstantiationModelAwarePointcutAdvisorImpl#getOrder()` 方法决定，它的 `getOrder()` 方法如下：

> InstantiationModelAwarePointcutAdvisorImpl

```
@Override
public int getOrder() {
    return this.aspectInstanceFactory.getOrder();
}

```

在 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（上）](https://my.oschina.net/funcy/blog/4678817)我们已经详细分析了 `method` 到 `advisor` 的转变过程，能从代码上轻松找到 `aspectInstanceFactory` 的类型，这里我们就不再一步步分析源码了，直接通过调试的方法获取 `aspectInstanceFactory` 的类型：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-e2784fc573474f56e8555ef60d57a07bcbf.png)

从调试的结果来看，`aspectInstanceFactory` 类型为 `LazySingletonAspectInstanceFactoryDecorator`，我们跟进其 `getOrder()` 方法：

> LazySingletonAspectInstanceFactoryDecorator

```
@Override
public int getOrder() {
    return this.maaif.getOrder();
}

```

我们依然使用调试的方式获取 `maaif` 的类型：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-2a4b87b6c8c824c55c4a3d86d736796def0.png)

`maaif` 类型为 `BeanFactoryAspectInstanceFactory`，我们继续跟进：

> BeanFactoryAspectInstanceFactory

```
public int getOrder() {
    // this.name 指的是标注了 @Aspect 注解的类
    Class<?> type = this.beanFactory.getType(this.name);
    if (type != null) {
        // 如果实现了 Ordered 接口，就调用 getOrder() 方法获取
        // PriorityOrdered 是 Ordered 的子接口，也有 getOrder() 方法，因此这里也会获取到
        if (Ordered.class.isAssignableFrom(type) && this.beanFactory.isSingleton(this.name)) {
            return ((Ordered) this.beanFactory.getBean(this.name)).getOrder();
        }
        // 1\. 查找类是是否有 @Order 注解，如果有，则返回 @Order 注解指定的值；
        // 2\. 否则查询类是否有 @Priority 注解，如果有，则返回 @Priority 注解指定的值；
        // 3\. 如果以上都不满足，返回 Ordered.LOWEST_PRECEDENCE，值为 Integer.MAX_VALUE
        return OrderUtils.getOrder(type, Ordered.LOWEST_PRECEDENCE);
    }
    return Ordered.LOWEST_PRECEDENCE;
}

```

从代码上来看，`getOrder()` 逻辑如下：

1.  通过名称获取切面类，也就是标注了 `@Aspect` 的类；
2.  如果切面类实现了 `Ordered` 接口，就调用 `getOrder()` 方法获取，返回（值得一提的是，`PriorityOrdered` 是 `Ordered` 的子接口，也有 `getOrder()` 方法，这里也会获取到）；
3.  如果上面没有获取到，则查找切面类是是否有 `@Order` 注解，如果有，则返回 `@Order` 注解指定的值；如果没有，查找切面类是否有 `@Priority` 注解，如果有，则返回 `@Priority` 注解指定的值；
4.  如果以上没有获取到值，就返回默认值：`Ordered.LOWEST_PRECEDENCE`。

到这里就明白了，`@Aspect` 类可以通过实现 `Ordered/PriorityOrdered` 接口来指定执行优先级，也可以通过 `@Order/@Priority` 注解来指定执行优先级。

**需要特别指出的是**，从 `getOrder()` 代码来看，这部分 代码只 是把 `PriorityOrdered/@Priority` 当作 `Order` 来处理，优先级并不比 `Ordered/@Order` 高。也就是说，如果 `AspectA` 标注了的了 `@Priority`，`AspectB` 标注了的了 `@Order`，`AspectA` 的优先级并不一定比 `AspectB` 高，真正决定优先级的是注解 里的 `value()` 值。

### 4\. 如何自定义优先级

我们在自己写代码时，如何指定优先级呢？

1. 如果是自主实现 `advisor`，可实现 `Ordered` 接口，也可以在 `advisor` 类上标注 `@Order` 注解：

   ```
   public class MyAdvisor extends AbstractBeanFactoryPointcutAdvisor implements Ordered {
   
       @Override
       public int getOrder() {
           return xxx;
       }
   
   }
   
   @Order(xxx)
   public class MyAdvisor extends AbstractBeanFactoryPointcutAdvisor {
   
       @Override
       public int getOrder() {
           return xxx;
       }
   
   }
   
   ```

2. 如果是单个切面类 (`@Aspect` 标注的类)，且无重复的 `@Around/@Before/@After` 等

   ```
   @Aspect
   public class MyAspectj {
   
       @Around("xxx")
       public Object around(ProceedingJoinPoint p){
           ...
       }
   
       @Before("xxx")
       public void before(JoinPoint p) {
           ...
       }
   
       @After("xxx")
       public void after(JoinPoint p) {
           ...
       }
   
       @AfterReturning("xxx")
       public void afterReturning(JoinPoint p) {
           ...
       }
   
       @AfterThrowing("xxx")
       public void afterThrowing(JoinPoint p) {
           ...
       }
   }
   
   ```

   对于同一切面的不同通知，spring 已经帮我们设置好了执行顺序，我们无从更改，执行顺序依次为 `Around, Before, After, AfterReturning, AfterThrowing`.

3. 单个切面类 (`@Aspect` 标注的类) 内有重复的 `@Around/@Before/@After` 等，情况如下：

   ```
   @Aspect
   public class MyAspectj {
   
       @Around("xxx")
       public Object around(ProceedingJoinPoint p){
           ...
       }
   
       @Before("xxx")
       public void before(JoinPoint p) {
           ...
       }
   
       @Around("xxx")
       public Object around(ProceedingJoinPoint p){
           ...
       }
   
       @Before("xxx")
       public void before(JoinPoint p) {
           ...
       }
   
   }
   
   ```

   这种情况我们在 `AspectJPrecedenceComparator#comparePrecedenceWithinAspect` 方法时有分析过，得到的结论是：比较两者的 `declarationOrderInAspect` 值，如果两者之一为 `after` 通知，`declarationOrderInAspect` 大的优先级高；如果两者都不是 `after` 通知，`declarationOrderInAspect` 小的优先级高。这个 `declarationOrderInAspect` 完全依赖于 jdk 的反射机制，先获取到的是哪个方法，哪个方法的 `declarationOrderInAspect` 就小，不同 jdK 版本之间，难以保证获得的顺序一致。

4. 多个切面类 (`@Aspect` 标注的类) 的执行顺序可以通过 `@Order` 注解，或实现 `Ordered` 接口来指定：

   ```
   @Order(xxx)
   public class MyAspectj1 {
       ...
   }
   
   @Order(xxx)
   public class MyAspectj2 {
       ...
   }
   
   ```

另外，`getOrder()` 返回的值或 `@Order(xxx)` 指定的值越小，表明优先级越高。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4784828](https://my.oschina.net/funcy/blog/4784828) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_