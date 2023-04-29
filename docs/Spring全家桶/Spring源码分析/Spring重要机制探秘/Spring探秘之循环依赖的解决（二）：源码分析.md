在 [spring 探秘之循环依赖（一）：理论基石](https://my.oschina.net/funcy/blog/4659555 "spring探秘之循环依赖（一）：理论基石")一文中 ，我们提到 spring 解决循环依赖的流程如下:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-dc325a87b321e4c246a1b2f14169821a75a.png)

为了能让以上流程能顺利进行，spring 中又提供了 5 个数据结构来保存流程中产生的关键信息，这 5 个数据结构如下（下文将这 5 个数据结构称为 **5 大结构**）：

| 结构                            | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `singletonObjects`              | **一级缓存**，类型为 `ConcurrentHashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是完整的 `spring bean`，即完成属性注入、初始化的 bean，如果 bean 需要 aop，存储的就是代理对象 |
| `earlySingletonObjects`         | **二级缓存**，类型为 `HashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是实例化完成，但未进行依赖注入的 `bean`，**如果 `bean` 需要 `aop`，这里存储的就是代理对象，只不过代理对象所持有的原始对象并未进行依赖注入** |
| `singletonFactories`            | **三级缓存**，类型为 `HashMap<String, ObjectFactory>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 存储的是一个 `lambda` 表达式：`() -> getEarlyBeanReference(beanName, mbd, bean)`，`getEarlyBeanReference(xxx)` 中的 `bean` 是刚创建完成的 `java bean`，没有进行 spring 依赖注入，也没进行 aop |
| `singletonsCurrentlyInCreation` | 类型为 `SetFromMap<String>`，位于 `DefaultSingletonBeanRegistry`，创建方式为 `Collections.newSetFromMap(new ConcurrentHashMap<>(16))`，表明这是个由 `ConcurrentHashMap` 实现的 set，存储的是正在创建中的对象，可以**用来判断当前对象是否在创建中** |
| `earlyProxyReferences`          | 类型为 `ConcurrentHashMap<Object, Object>`，位于 `AbstractAutoProxyCreator`，存储的是提前进行 aop 的对象，可以**用来判断 bean 是否进行过 aop，保证每个对象只进行一次 aop** |

了解了这些之后，接下来我们就正式开始进行源码分析了。

### 1\. 准备 demo

本文中的示例 demo 位于 [gitee.com/funcy](https://gitee.com/funcy/spring-framework/tree/v5.2.2.RELEASE_learn/spring-learn/src/main/java/org/springframework/learn/explore/demo03 "gitee.com/funcy")，这里仅给出关键代码。

准备两个 service：service1，service2，这两个 service 里都有一个代理方法：

```
@Service
public class Service1 {

    @Autowired
    private Service2 service2;

    public Service1() {
        System.out.println("调用service1的构造方法");
    }

    /**
     * 标注 @AopAnnotation 了，表明这个方法要被代理
     */
    @AopAnnotation
    public void printAutowired() {
        System.out.println("Service1 Autowired:" + service2.getClass());
    }

    @Override
    public String toString() {
        return "Service1:" + getClass();
    }
}

@Component
public class Service2 {

    @Autowired
    private Service1 service1;

    public Service2() {
        System.out.println("调用service2的构造方法");
    }

    /**
     * 标注 @AopAnnotation 了，表明这个方法要被代理
     */
    @AopAnnotation
    public void printAutowired() {
        System.out.println("Service2 Autowired:" + service1.getClass());
    }

    @Override
    public String toString() {
        return "Service2:" + this.getClass();
    }
}

```

这里是主类：

```
public class Demo03Main {

    public static void main(String[] args) {
        ApplicationContext context = 
                new AnnotationConfigApplicationContext(AopAnnotationConfig.class);
        Object obj1 = context.getBean("service1");
        Object obj2 = context.getBean("service2");
        ((Service1)obj1).printAutowired();
        ((Service2)obj2).printAutowired();
    }
}

```

在 `Service1` 中，需要注入属性 `service2`，在在 `Service2` 中，需要注入属性 `service1`，且 `Service1`、`Service2` 都需要进行代理，最终 `main()` 方法执行结果如下：

```
调用service1的构造方法
调用service2的构造方法
Disconnected from the target VM, address: 'localhost:55518', transport: 'socket'
Connected to the target VM, address: '127.0.0.1:55507', transport: 'socket'
@Around: before execute...
Service1 Autowired:class org.springframework.learn.explore.demo03.Service2$$EnhancerBySpringCGLIB$$e7e367ab
@Around: after execute...
@Around: before execute...
Service2 Autowired:class org.springframework.learn.explore.demo03.Service1$$EnhancerBySpringCGLIB$$d447df08
@Around: after execute...

```

得到的 obj1、obj2 分别为

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-201844d13a7b489d1a18a8218d6440d6068.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-3781280ce3f4da91504b0665ebfd3aed05b.png)

分析内容，不难得出如下结论：

*   `service1` 与 `service2` 的 `printAutowired()` 方法都输出了切面方法的内容，表明两者都代理成功了；
*   从容器中获取的的 `service1` 与 `service2` 都是代理对象；
*   `service1` 的代理对象持有 `service1` 的原始对象，`service2` 的代理对象持有 `service2` 的原始对象；
*   `service1` 的原始对象中注入的 `service2` 为代理对象，`service2` 的原始对象中注入的 `service1` 为代理对象。

接下来，我们就从源码分析，看 spring 是如何一步步运行，最终得到这个结果的。

> `service1` 与 `service2` 的代理对象是由 `cglib` 代理产生的，这部分内容与本文无关，就不分析了。

### 2\. 第一次调用 `AbstractBeanFactory#getBean(String)`：获取 `service1`

spring bean 的创建、依赖注入流程是从 `AbstractBeanFactory#getBean(String)` 方法开始，这一点可参考 [spring 启动流程之完成 BeanFactory 的初始化](https://my.oschina.net/funcy/blog/4658230)，我们的源码分析也从这个方法入手。

本文一开始，就提到了 spring 为解决循环依赖的 5 大结构，这里展示如下：

| 结构                            | 内容 |
| ------------------------------- | ---- |
| `singletonObjects`              |      |
| `earlySingletonObjects`         |      |
| `singletonFactories`            |      |
| `singletonsCurrentlyInCreation` |      |
| `earlyProxyReferences`          |      |

一开始，5 大结构中关于 `service1` 与 `service2` 的什么数据都没有 ，接下来，我们一边分析代码，一边关注这几个结构中的内容。

> 事实上，5 大结构是有内容的，那是 spring 内部提供的一些 bean，但由于我们只关注 `service1` 与 `service2` 相关的内容，因此这里不展示。

### 2.1 `AbstractBeanFactory#doGetBean`

获取 `service1` 的代码在 `AbstractBeanFactory#getBean(String)`，但真正的获取操作却是在 `AbstractBeanFactory#doGetBean` 中，调用链如下：

```
|-AbstractBeanFactory#getBean(String)
 |-AbstractBeanFactory#doGetBean

```

代码如下：

```
    protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
            @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

        ...
        // 1\. 检查是否已初始化，这里会把beanName放入singletonsCurrentlyInCreation中
        Object sharedInstance = getSingleton(beanName);
        ...
        // 如果是单例的
        if (mbd.isSingleton()) {
            // 2\. getSingleton(): 创建对象，删除二、三级缓存
            sharedInstance = getSingleton(beanName, () -> {
                try {
                    // 执行创建 Bean
                    return createBean(beanName, mbd, args);
                }
                catch (BeansException ex) {
                    destroySingleton(beanName);
                    throw ex;
                }
            });
            // 这里如果是普通Bean 的话，直接返回，如果是 FactoryBean 的话，
            // 返回它创建的那个实例对象
            bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
        }
        ...
    }

```

以上代码经过了精简，我们只保留了主要流程（后续分析中，也会精简代码，只保留主要流程，省略无关代码），这个方法有两个地方需要分析：

1.  `getSingleton(beanName)`：获取对象，只从三个缓存中获取；
2.  `getSingleton(beanName, () -> { ... })`：获取对象，创建会在这里创建。

#### 2.2 `DefaultSingletonBeanRegistry#getSingleton(String, boolean)`

我们先来看看 `getSingleton(beanName)`，调用链如下：

```
|-AbstractBeanFactory#getBean(String)
 |-AbstractBeanFactory#doGetBean
  |-DefaultSingletonBeanRegistry#getSingleton(String)
   |-DefaultSingletonBeanRegistry#getSingleton(String, boolean)

```

直接看代码：

```
/*
 * beanName: 传入的值为 service1
 * allowEarlyReference：传入的值为 true
 */
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 1\. 从一级缓存中获取，这里是获取不到
    Object singletonObject = this.singletonObjects.get(beanName);
    // 2\. isSingletonCurrentlyInCreation(...) 判断service1是否在创建中，返回 false
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
        // 省略下面的代码
        ...
    }
    return singletonObject;
}

```

对上述代码分析如下：

1. 从三级缓存中获取，此时 `singletonObjects` 里并没有 `service1`，获取不到；

2. 继续，判断 `service1` 是否在创建中，判断方式如下：

   ```
   public boolean isSingletonCurrentlyInCreation(String beanName) {
       return this.singletonsCurrentlyInCreation.contains(beanName);
   }
   
   ```

   显然，`singletonsCurrentlyInCreation` 是没有 `service1` 的，这里也返回 false.

`DefaultSingletonBeanRegistry#getSingleton(String, boolean)` 运行到这里就返回了，这个方法里下面的返回我们就先不分析了。

#### 2.3 `DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)`

让我们回到 `AbstractBeanFactory#doGetBean`，接着分析 `getSingleton(beanName, () -> { ... })`，代码如下：

```
    /**
     * beanName：传入的是service1
     * singletonFactory：传入的是lambda表达式，值为
     *           () -> {
     *               try {
     *                   // 这里就是创建bean的流程
     *                   return createBean(beanName, mbd, args);
     *               }
     *               catch (BeansException ex) {
     *                   destroySingleton(beanName);
     *                   throw ex;
     *               }
     *           }
     */
    public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
        Assert.notNull(beanName, "Bean name must not be null");
        synchronized (this.singletonObjects) {
            // 再次判断bean是否已存在
            Object singletonObject = this.singletonObjects.get(beanName);
            if (singletonObject == null) {
                // 1\. 判断当前正在实例化的bean是否存在正在创建中
                // 这个方法会beanName添加到 singletonsCurrentlyInCreation 中
                beforeSingletonCreation(beanName);
                try {
                    // 2\. 在这里进行bean的创建
                    singletonObject = singletonFactory.getObject();
                }
                catch (...) {
                    ...
                }
                ...
            }
            return singletonObject;
        }
    }

```

按照代码中的注释内容，我们分析主要步骤：

1. 判断当前正在实例化的 bean 是否存在正在创建中，就是判断当前对象是否在 `singletonsCurrentlyInCreation` 中，如果存在则抛出异常，不存在则添加到 `singletonsCurrentlyInCreation` 中；

   运行完之后这一行代码后，5 大结构中的内容如下：

   | 结构                            | 内容     |
      | ------------------------------- | -------- |
   | `singletonObjects`              |          |
   | `earlySingletonObjects`         |          |
   | `singletonFactories`            |          |
   | `singletonsCurrentlyInCreation` | service1 |
   | `earlyProxyReferences`          |          |

2. 进行 bean 的创建过程，`singletonFactory.getObject()` 运行的实际上是传入的 lambda 表达式：

   ```
   () -> {
       try {
           // 执行创建 Bean
           return createBean(beanName, mbd, args);
       }
       catch (BeansException ex) {
           destroySingleton(beanName);
           throw ex;
       }
   }
   
   ```

   也就是 `AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])`，这 个方法是接下来分析的重点；

#### 2.4 `AbstractAutowireCapableBeanFactory#doCreateBean`

从代码上来看，`AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])` 并没有做什么实质性内容，我们就直接来到 `AbstractAutowireCapableBeanFactory#doCreateBean` 了，这其中经历的千山万水如下：

```
|-AbstractBeanFactory#getBean(String)
 |-AbstractBeanFactory#doGetBean
  |-DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)
   |-AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])
    |-AbstractAutowireCapableBeanFactory#doCreateBean

```

`AbstractAutowireCapableBeanFactory#doCreateBean` 内容如下：

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, 
        final @Nullable Object[] args) throws BeanCreationException {

    BeanWrapper instanceWrapper = null;
    if (instanceWrapper == null) {
        // 1\. 实例化 Bean，调用java反射进行实例化
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
                // 调用 AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition 方法
                // 2\. 获取需要注入的属性与方法（标注 @Autowired 注解）
                applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
            }
            catch (Throwable ex) {
                throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                        "Post-processing of merged bean definition failed", ex);
            }
            mbd.postProcessed = true;
        }
    }
    // 解决循环依赖问题, 是否允许循环依赖, allowCircularReferences默认为true，可以关闭
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
            isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        // 3\. 将 bean 添加到 singletonFactories
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }

    Object exposedObject = bean;
    try {
        // 4\. 属性注入，发现需要注入 service2，然后再调用 getBean("service2")
        populateBean(beanName, mbd, instanceWrapper);
        ...
    }
    catch (Throwable ex) {
        ...
    }
}

```

这个方法进行的步骤如下：

1. 实例化 Bean，调用 java 反射进行实例化，这个就不多说了，实例化后的 `service1` 如下，可以看到 `service2` 还是 `null`（表明未进行属性注入）：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8b2f5fdf0efd6beb7f62d4f35f5c5562b83.png)

2. 获取需要注入的属性与方法，**必须在原始对象中才能获取到，代理对象是获取不到的**，查找 `@Autowired` 的 `beanPostProcessor` 是 `AutowiredAnnotationBeanPostProcessor`；

3. 将 bean 添加到 `singletonFactories` 中，添加过程如下：

   ```
   protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
       Assert.notNull(singletonFactory, "Singleton factory must not be null");
       synchronized (this.singletonObjects) {
           // 如果单例池当中不存在才会add
           if (!this.singletonObjects.containsKey(beanName)) {
               // 把工厂对象put到singletonFactories
               this.singletonFactories.put(beanName, singletonFactory);
               // 删除二级缓存中的对象
               this.earlySingletonObjects.remove(beanName);
               this.registeredSingletons.add(beanName);
           }
       }
   }
   
   ```

   加入的对象为

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-3d9a56a0d655fe1e95d8857095fb46aa71a.png)

   此时那 5 个结构中的内容如下：

   | 结构                            | 内容             |
      | ------------------------------- | ---------------- |
   | `singletonObjects`              |                  |
   | `earlySingletonObjects`         |                  |
   | `singletonFactories`            | lambda(service1) |
   | `singletonsCurrentlyInCreation` | service1         |
   | `earlyProxyReferences`          |                  |

   `lambda(service1)` 实际内容为 `() -> getEarlyBeanReference(beanName, mbd, bean)`，这个 `lambda` 表达式所做的就是进行 aop 操作，具体内容等运行的时候我们再分析。

4. 属性注入，在第 2 步中，spring 找到 `service1` 需要注入对象有 `service2`，这里会再调用 `beanFactory.getBean("service2")` 进入 `service2` 的生命周期，这又回到了 `AbstractBeanFactory#getBean(String)` 方法。

   从 `populateBean(xxx)` 到 `getBean(xxx)` 的步骤相当多，调用链如下：

   ```
   |-AbstractBeanFactory#getBean(String)
    |-AbstractBeanFactory#doGetBean
     |-DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)
      |-AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])
       |-AbstractAutowireCapableBeanFactory#doCreateBean
        // 这里进行依赖注入
        |-AbstractAutowireCapableBeanFactory#populateBean
         |-AutowiredAnnotationBeanPostProcessor#postProcessProperties
          |-InjectionMetadata#inject
           |-AutowiredAnnotationBeanPostProcessor.AutowiredFieldElement#inject
            |-DefaultListableBeanFactory#resolveDependency
             |-DefaultListableBeanFactory#doResolveDependency
              |-DependencyDescriptor#resolveCandidate
               |-AbstractBeanFactory#getBean(String)
   
   ```

   看一眼 `DependencyDescriptor#resolveCandidate`，调用的正是 `beanFactory.getBean(String)` 方法：

   ```
   public Object resolveCandidate(String beanName, Class<?> requiredType, BeanFactory beanFactory)
           throws BeansException {
       return beanFactory.getBean(beanName);
   }
   
   ```

到了这里，`service1` 的获取流程就先停一下，因为 `service1` 要注入 `service2`，接下来就开始获取 `service2` 的流程。

本小节结束时，5 大结构中的内容如下：

| 结构                            | 内容             |
| ------------------------------- | ---------------- |
| `singletonObjects`              |                  |
| `earlySingletonObjects`         |                  |
| `singletonFactories`            | lambda(service1) |
| `singletonsCurrentlyInCreation` | service1         |
| `earlyProxyReferences`          |                  |

### 3\. 第二次调用 `AbstractBeanFactory#getBean(String)`：获取 `service2`

#### 3.1 `AbstractBeanFactory#doGetBean`

这一步跟获取 `service1` 的流程基本一致，不同的是 `beanName` 是 `service2`，不再分析。

#### 3.2 `DefaultSingletonBeanRegistry#getSingleton(String, boolean)`

这一步跟获取 `service1` 的流程基本一致，不同的是 `beanName` 是 `service2`，不再分析。

#### 3.3 `DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)`

这一步跟获取 `service1` 的流程基本一致，不同的是 `beanName` 是 `service2`，具体内容不再分析。

`service2` 会在这一步加入 到 `singletonsCurrentlyInCreation` 结构中，这一步之后，5 大结构中的内容如下：

| 结构                            | 内容               |
| ------------------------------- | ------------------ |
| `singletonObjects`              |                    |
| `earlySingletonObjects`         |                    |
| `singletonFactories`            | lambda(service1)   |
| `singletonsCurrentlyInCreation` | service1, service2 |
| `earlyProxyReferences`          |                    |

#### 3.4 `AbstractAutowireCapableBeanFactory#doCreateBean`

这一步跟获取 `service1` 的流程基本一致，说明如下：

1. 对象创建，这里会创建 `service2`，创建完成后的对象如下：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-393811afa91cc1ff696b02ce6d683c97640.png)

   同样的，`service2` 中的属性 `service1` 也为 `null`；

2. 获取需要注入的属性与方法，`service2` 的属性 `service1` 会被找到，因为该属性标注有 `@Autowired` 注解；

3. 将 bean 添加到 `singletonFactories` 中，这一步之后，5 个结构中的内容如下：

   | 结构                            | 内容                               |
      | ------------------------------- | ---------------------------------- |
   | `singletonObjects`              |                                    |
   | `earlySingletonObjects`         |                                    |
   | `singletonFactories`            | lambda(service1), lambda(service2) |
   | `singletonsCurrentlyInCreation` | service1, service2                 |
   | `earlyProxyReferences`          |                                    |

   这一步的关键在于，`service2` 添加到三级缓存中了；

4. 属性注入，在第 2 步中，spring 找到 `service2` 需要注入对象有 `service1`，这里会再调用 `getBean("service2")` 方法，又回到了 `AbstractBeanFactory#getBean(String)` 方法了。

到了这里，`service2` 的获取流程也要先停一下了，因为 `service2` 要注入 `service1`，接下来又要开始获取 `service1` 的流程。

本小节结束时，5 个结构中的内容如下：

| 结构                            | 内容                               |
| ------------------------------- | ---------------------------------- |
| `singletonObjects`              |                                    |
| `earlySingletonObjects`         |                                    |
| `singletonFactories`            | lambda(service1), lambda(service2) |
| `singletonsCurrentlyInCreation` | service1, service2                 |
| `earlyProxyReferences`          |                                    |

### 4\. 第三次调用 `AbstractBeanFactory#getBean(String)`：再次获取 `service1`

#### 4.1 `AbstractBeanFactory#doGetBean`

`AbstractBeanFactory#doGetBean` 代码如下：

```
    protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
            @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

        ...
        // 1\. 检查是否已初始化，这里会把beanName放入singletonsCurrentlyInCreation中
        Object sharedInstance = getSingleton(beanName);
        ...
}

```

在运行到 `Object sharedInstance = getSingleton(beanName)` 前，`AbstractBeanFactory#doGetBean` 的运行流程与前面两小节无区别，在 `Object sharedInstance = getSingleton(beanName)` 里就有了变化，接下来我们来看看 `getSingleton(beanName)` 的执行。

#### 4.2 `DefaultSingletonBeanRegistry#getSingleton(String, boolean)`

```
/*
 * beanName: 传入的值为 service1
 * allowEarlyReference：传入的值为 true
 */
protected Object getSingleton(String beanName, boolean allowEarlyReference) {
    // 1\. 从一级缓存中获取，这里是获取不到
    Object singletonObject = this.singletonObjects.get(beanName);
    // 2\. isSingletonCurrentlyInCreation(...) 判断service1是否在创建中，返回 true
    if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
            synchronized (this.singletonObjects) {
                // 3\. 从二级缓存中获取，这里也获取不到
                singletonObject = this.earlySingletonObjects.get(beanName);
                // 4\. 传入的allowEarlyReference是true，继续执行里面的内容
                if (singletonObject == null && allowEarlyReference) {
                    // 5\. 从三级缓存中获取，能获取到
                    ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
                    if (singletonFactory != null) {
                        // 6\. 调用 singletonFactory.getObject()
                        // 最终调用的是 AbstractAutowireCapableBeanFactory.getEarlyBeanReference 方法
                        singletonObject = singletonFactory.getObject();
                        // 7\. 处理缓存
                        // 放到二级缓存中
                        this.earlySingletonObjects.put(beanName, singletonObject);
                        // 从三级缓存中清除
                        this.singletonFactories.remove(beanName);
                    }
                }
            }
    }
    return singletonObject;
}

```

在分析代码前，先来看看此时 5 大结构中的内容：

| 结构                            | 内容                               |
| ------------------------------- | ---------------------------------- |
| `singletonObjects`              |                                    |
| `earlySingletonObjects`         |                                    |
| `singletonFactories`            | lambda(service1), lambda(service2) |
| `singletonsCurrentlyInCreation` | service1, service2                 |
| `earlyProxyReferences`          |                                    |

对照着上面的内容，分析如下：

1. 从一级缓存中获取，对照着 5 个结构中的内容，获取不到，返回 `null`;

2. `isSingletonCurrentlyInCreation(...)` 判断 `service1` 是否在创建中，对照着 5 个结构中的内容，`service1` 在 `singletonsCurrentlyInCreation` 中，返回 true，继续下面的流程，**这是与第一次获取 `service1` 的不同之处**；

3. 继续从二级缓存中获取，对照着 5 个结构中的内容，`service1` 不在 `earlySingletonObjects`，依然返回 `null`；

4. `allowEarlyReference` 是传入的参数，为 `true`，继续执行 下面的代码；

5. 继续从从三级缓存中获取，对照着 5 个结构中的内容，`service1` 在 `singletonFactories` 中，这里能获取到，返回的结果如下：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-57677e20855580e5bc2f5163d87ecfda7d2.png)

6. 调用 `singletonFactory.getObject()` 方法，当初我们传入 `singletonFactories` 的还是个 lambda 表达式：`() -> getEarlyBeanReference(beanName, mbd, bean)`，调用 `singletonFactory.getObject()` 最终调用的就是 `AbstractAutowireCapableBeanFactory#getEarlyBeanReference`：

   ```
   protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
       Object exposedObject = bean;
       if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
           for (BeanPostProcessor bp : getBeanPostProcessors()) {
               if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
                   // 调用后置处理器，这里完成了aop操作
                   SmartInstantiationAwareBeanPostProcessor ibp = 
                         (SmartInstantiationAwareBeanPostProcessor) bp;
                   // 调用 `AbstractAutoProxyCreator#getEarlyBeanReference`，提前进行 aop
                   exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
               }
           }
       }
       return exposedObject;
   }
   
   ```

   我们跟进 `AbstractAutoProxyCreator#getEarlyBeanReference` 方法：

   ```
   public Object getEarlyBeanReference(Object bean, String beanName) {
       Object cacheKey = getCacheKey(bean.getClass(), beanName);
       this.earlyProxyReferences.put(cacheKey, bean);
       // 这里生成代理对象
       return wrapIfNecessary(bean, beanName, cacheKey);
   }
   
   ```

   关于 aop 的流程，可参考 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)，这里就不再分析了。执行完 `AbstractAutoProxyCreator#getEarlyBeanReference` 之后，5 个结构中的内容如下：

   | 结构                            | 内容                               |
      | ------------------------------- | ---------------------------------- |
   | `singletonObjects`              |                                    |
   | `earlySingletonObjects`         |                                    |
   | `singletonFactories`            | lambda(service1), lambda(service2) |
   | `singletonsCurrentlyInCreation` | service1, service2                 |
   | `earlyProxyReferences`          | service1                           |

   让我们再回到 `getSingleton` 方法，执行完 `singletonFactory.getObject()` 后，得到的 `singletonObject` 为

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-cf72a8ff3d91c3d2a4e300741ba7f0268e6.png)

   这是一个代理对象，并且 `service2` 为 `null`。

7. 接下来就是处理缓存了，一些 map 的 put 与 remove 操作，就不多说了。

`DefaultSingletonBeanRegistry#getSingleton(String, boolean)` 方法执行完成后，5 个结构中的内容如下：

| 结构                            | 内容               |
| ------------------------------- | ------------------ |
| `singletonObjects`              |                    |
| `earlySingletonObjects`         | service1           |
| `singletonFactories`            | lambda(service2)   |
| `singletonsCurrentlyInCreation` | service1, service2 |
| `earlyProxyReferences`          | service1           |

#### 4.3 再次回到 `AbstractBeanFactory#doGetBean`

执行完 `DefaultSingletonBeanRegistry#getSingleton(String, boolean)` 后，我们再回到 `AbstractBeanFactory#doGetBean`：

```
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {
    ...

    // 检查是否已初始化
    Object sharedInstance = getSingleton(beanName);

    // 1\. 这次能获取到对象，if 里面的代码会执行
    if (sharedInstance != null && args == null) {
        // 2\. 这里如果是普通Bean 的话，直接返回，如果是 FactoryBean 的话，返回它创建的那个实例对象
        // 这里的bean是service1的代理对象
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    } else {
        // 这里的代码不会执行到，就不分析了
        ...
    }

    // 处理bean的转换操作，这里返回false，不会执行到
    if (requiredType != null && !requiredType.isInstance(bean)) {
    ...
    }

    // 3\. 返回从`getSingleton(beanName)`得到的对象
    return bean;

}

```

1. 这次能获取到对象，if 里面的代码会执行；

2. `service1` 不是 `FactoryBean`，与 `getSingleton(beanName)` 得到的对象是同一个对象，内容如下：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0cd9c129bf2387d85a7bee4f4e135143f63.png)

3. 返回从 `getSingleton(beanName)` 得到的对象，在 `getSingleton(beanName)` 中返回的对象不为空后，就不再执行 bean 的创建流程了，最终返回的是第 2 步中得到的 bean 了。

到了这里，我们终于获取到了 `service1` 的代理对象，尽管代理对象对应的原始对象并没有完成依赖注入，但我们依然可以进行后续的流程，即获取到 `service1` 的代理对象后，`service2` 就可以进行依赖注入，继续接下来的流程了。

最后，我们再来看看本节执行完之后，5 大结构中的内容：

| 结构                            | 内容                            |
| ------------------------------- | ------------------------------- |
| `singletonObjects`              |                                 |
| `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
| `singletonFactories`            | lambda(service2)                |
| `singletonsCurrentlyInCreation` | service1, service2              |
| `earlyProxyReferences`          | service1                        |

获取到 `service1` 后，接下来就继续 `service2` 的获取流程了。

### 5 继续 `service2` 的获取流程

在第 3 节中，因为 `service2` 需要注入 `service1`，才有了第 4 部分的再次获取 `service1` 的流程，最终也成功地获取到了 `service1` 的代理对象 `service1$$EnhancerBySpringCGLIB`，这才让 `service2` 得以完成依赖注入。

接下来，让我们延续 第 3 节 的流程，回到 `AbstractAutowireCapableBeanFactory#doCreateBean`，继续 `service2` 的流程。

#### 5.1 回到 `AbstractAutowireCapableBeanFactory#doCreateBean`

代码如下：

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, 
            final @Nullable Object[] args) throws BeanCreationException {

        // 上面的代码在 3.2 中已经分析过了，就不再分析了
        ...

        Object exposedObject = bean;
        try {
            // 1\. 负责属性装配, 也就是我们常常说的依赖注入
            populateBean(beanName, mbd, instanceWrapper);
            // 2\. 初始化，在这里会处理 aop 操作
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
        catch (Throwable ex) {
            ...
        }

        //同样的，如果存在循环依赖
        if (earlySingletonExposure) {
            // 3\. 从缓存中获取 beanName 对应的bean
            Object earlySingletonReference = getSingleton(beanName, false);
            // 4\. earlySingletonReference 为 null，if 里面的内容不会执行
            if (earlySingletonReference != null) {
                ...
            }
        }

        // 省略不重要的代码
        ...

        return exposedObject;
    }

```

代码说明如下：

1. 处理依赖注入，就是这里触发了第 4 部分的流程，执行完得到的 `service2` 如下：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b294a02be5431c0aa66e5f126a6c6abbb92.png)

   可以看到，此时注入到 `service2` 中的 `service1` 就是代理对象了；

2. 初始化，在这里会处理 aop 操作，执行 aop 的方法为 `AbstractAutoProxyCreator#postProcessAfterInitialization`，代码如下：

   ```
   // 传入的bean为service2，beanName为“service2”
   public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
       if (bean != null) {
           Object cacheKey = getCacheKey(bean.getClass(), beanName);
           // 1\. earlyProxyReferences 中并没有 service2，if里的代码会执行到
           if (this.earlyProxyReferences.remove(cacheKey) != bean) {
               // 2\. 在这里处理aop操作
               return wrapIfNecessary(bean, beanName, cacheKey);
           }
       }
       return bean;
   }
   
   ```

   以上代码先从 `earlyProxyReferences` 移出 `service2`，然后再与传入的 bean 做比较，对照着 5 大结构中的内容，我们发现 `service2` 并不在 `earlyProxyReferences` 中；

   | 结构                            | 内容                            |
      | ------------------------------- | ------------------------------- |
   | `singletonObjects`              |                                 |
   | `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
   | `singletonFactories`            | lambda(service2)                |
   | `singletonsCurrentlyInCreation` | service1, service2              |
   | `earlyProxyReferences`          | service1                        |

   因此，这里返回 `null`，显然不等于传入的 bean，因此 if 中的代码会执行，`service2` 进行 aop。执行完这一步后，得到的 `exposedObject` 为：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d62ebdbb71b76ada963a01c9b8c84b88b00.png)

   `service2` 也变成了代理对象，且已完成了依赖注入。

3. 从缓存中获取 beanName 对应的 bean，执行的是 `DefaultSingletonBeanRegistry#getSingleton(String, boolean)`，代码如下：

   ```
   /**
    * beanName: service2
    * allowEarlyReference：false
    */
   protected Object getSingleton(String beanName, boolean allowEarlyReference) {
       // 1\. 从一级缓存中获取 service2，获取不到，返回null
       Object singletonObject = this.singletonObjects.get(beanName);
       // 2\. 判断 service2 是否在创建中，返回true，执行if中的代码
       if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
           synchronized (this.singletonObjects) {
               // 3\. 从二级缓存中获取 service2，获取不到，返回null
               singletonObject = this.earlySingletonObjects.get(beanName);
               // 4\. 此时传入的 allowEarlyReference 为false，if块的代码不会执行
               if (singletonObject == null && allowEarlyReference) {
                   ...
               }
           }
       }
       return singletonObject;
   }
   
   ```

   我们还是来看 一眼 5 大结构中的内容，对这个方法的执行就一目了然了：

   | 结构                            | 内容                            |
      | ------------------------------- | ------------------------------- |
   | `singletonObjects`              |                                 |
   | `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
   | `singletonFactories`            | lambda(service2)                |
   | `singletonsCurrentlyInCreation` | service1, service2              |
   | `earlyProxyReferences`          | service1                        |

   方法的执行，在代码中已经注释得很明白了，就多说了；

4. 第 3 步返回的 `earlySingletonReference` 为 null，if 里面的内容不会执行。

到这里，得到的 `service2` 就返回了。

#### 5.2 回到 `DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)`

得到 `service2` 的 bean 后，我们回到 `DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)`，继续 `service2` 的流程，代码如下：

```
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    Assert.notNull(beanName, "Bean name must not be null");
    synchronized (this.singletonObjects) {
        ...
            try {
                // 1\. 在这里进行bean的创建
                singletonObject = singletonFactory.getObject();
                newSingleton = true;
            }
            catch (...) {
                ...
            }
            finally {
                ...
                // 2\. 创建完成后，做一些检查操作
                // 这里会把 service2 从 singletonsCurrentlyInCreation 移除
                afterSingletonCreation(beanName);
            }
            if (newSingleton) {
                // 3\. 将该对象添加到 beanFactory 中，这个方法会删除二三级缓存
                addSingleton(beanName, singletonObject);
            }
        ...
        return singletonObject;
    }
}

```

说明如下：

1. 历经千难万险，`singletonFactory.getObject()` 终于执行完了，在诸多曲折之后，最终还是得到了 `service2` 的代理对象；

2. `service2` 创建完成后，就会将其从 `singletonsCurrentlyInCreation` 移除，方法比较简单，就不多说了，这一步得到的 5 大结构如下：

   | 结构                            | 内容                            |
      | ------------------------------- | ------------------------------- |
   | `singletonObjects`              |                                 |
   | `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
   | `singletonFactories`            | lambda(service2)                |
   | `singletonsCurrentlyInCreation` | service1                        |
   | `earlyProxyReferences`          | service1                        |

3. 接着，就是缓存的处理，方法为 `DefaultSingletonBeanRegistry#addSingleton`：

   ```
   /*
    * beanName：service2
    * singletonObject：service2$$EnhancerBySpringCGLIB(service2的代理对象)
    */ 
   protected void addSingleton(String beanName, Object singletonObject) {
       synchronized (this.singletonObjects) {
           // 对三个缓存进行put、remove操作，最终只在一级缓存中有该对象
           this.singletonObjects.put(beanName, singletonObject);
           this.singletonFactories.remove(beanName);
           this.earlySingletonObjects.remove(beanName);
           this.registeredSingletons.add(beanName);
       }
   }
   
   ```

   方法比较简单，就不多说了，最终 5 大结构为：

   | 结构                            | 内容                            |
      | ------------------------------- | ------------------------------- |
   | `singletonObjects`              | service2$$EnhancerBySpringCGLIB |
   | `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
   | `singletonFactories`            |                                 |
   | `singletonsCurrentlyInCreation` | service1                        |
   | `earlyProxyReferences`          | service1                        |

到此，`service2` 获取完成，最终保存到 `singletonObjects` 中的 bean 为：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-763dad35376ed1d88086c45f01ee9c4cedd.png)

本节的最后，我们也来看下 5 个结构中的内容：

| 结构                            | 内容                            |
| ------------------------------- | ------------------------------- |
| `singletonObjects`              | service2$$EnhancerBySpringCGLIB |
| `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
| `singletonFactories`            |                                 |
| `singletonsCurrentlyInCreation` | service1                        |
| `earlyProxyReferences`          | service1                        |

至此，`service2` 获取完成，接着让我们继续 `service1` 的获取流程。

### 6 继续 `service1` 的获取流程

在第 2 节中，因为 `servic1` 需要注入 `service2`，才有了第 3 节获取 `service2` 的流程，然后又经过一系列的操作后，最终在第 5 节成功地获取到了 `service2` 的代理对象 `service2$$EnhancerBySpringCGLIB`，这才让 `service2` 得以完成依赖注入。

接下来，让我们延续第 2 节，回到 `AbstractAutowireCapableBeanFactory#doCreateBean` 方法，继续 `service1` 的流程。

#### 6.1 回到 `AbstractAutowireCapableBeanFactory#doCreateBean`

代码如下：

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, 
            final @Nullable Object[] args) throws BeanCreationException {

        // 上面的代码在 3.1 中已经分析过了，就不再分析了
        ...

        Object exposedObject = bean;
        try {
            // 1\. 负责属性装配, 也就是我们常常说的依赖注入
            populateBean(beanName, mbd, instanceWrapper);
            // 2\. 初始化，在这里会处理 aop 操作
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
        catch (Throwable ex) {
            ...
        }

        //同样的，如果存在循环依赖
        if (earlySingletonExposure) {
            // 3\. 从缓存中获取 beanName 对应的bean
            Object earlySingletonReference = getSingleton(beanName, false);
            // 4\. earlySingletonReference 不为 null，if 里面的内容会执行
            if (earlySingletonReference != null) {
                if (exposedObject == bean) {
                    // 5\. 将返回的对象赋值给 exposedObject，返回对象为代理对象
                    exposedObject = earlySingletonReference;
                }
            }
        }

        // 省略不重要的代码
        ...

        return exposedObject;
    }

```

代码说明如下：

1. 处理依赖注入，就是这里触发了第 3 节的流程，执行完得到的 `service1` 如下：

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-3883285be5904e056bc517d35bfd44c47a0.png)

   可以看到，`service1` 已经完成了依赖注入，且此时注入到 `service1` 中的 `service2` 已经是代理对象了，不过 `service1` 此时还是原始对象，我们继续往下看；

2. 初始化，在这里会处理 aop 操作，执行 aop 的方法为 `AbstractAutoProxyCreator#postProcessAfterInitialization`，代码如下：

   ```
   // 传入的bean为service1，beanName为“service1”
   public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) {
       if (bean != null) {
           Object cacheKey = getCacheKey(bean.getClass(), beanName);
           // 1\. earlyProxyReferences 中有 service1，if里的代码不会执行到
           if (this.earlyProxyReferences.remove(cacheKey) != bean) {
               // 2\. 在这里处理aop操作，service1 已经执行过aop了
               return wrapIfNecessary(bean, beanName, cacheKey);
           }
       }
       return bean;
   }
   
   ```

   以上代码先从 `earlyProxyReferences` 移出 `service1`，然后再与传入的 bean 做比较，对照着 5 大结构中的内容，我们发现 `service1` 此时就在 `earlyProxyReferences` 中；

   | 结构                            | 内容                            |
      | ------------------------------- | ------------------------------- |
   | `singletonObjects`              | service2$$EnhancerBySpringCGLIB |
   | `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
   | `singletonFactories`            |                                 |
   | `singletonsCurrentlyInCreation` | service1                        |
   | `earlyProxyReferences`          | service1                        |

   因此，这里会返回 `service1`，if 中的代码不会执行。执行完这一步后，5 个结构中的内容为：

   | 结构                            | 内容                            |
      | ------------------------------- | ------------------------------- |
   | `singletonObjects`              | service2$$EnhancerBySpringCGLIB |
   | `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
   | `singletonFactories`            |                                 |
   | `singletonsCurrentlyInCreation` | service1                        |
   | `earlyProxyReferences`          |                                 |

3. 从缓存中获取 beanName 对应的 bean，执行的是 `DefaultSingletonBeanRegistry#getSingleton(String, boolean)`，代码如下：

   ```
   /**
    * beanName: service1
    * allowEarlyReference：false
    */
   protected Object getSingleton(String beanName, boolean allowEarlyReference) {
       // 1\. 从一级缓存中获取 service1，获取不到，返回null
       Object singletonObject = this.singletonObjects.get(beanName);
       // 2\. 判断 service1 是否在创建中，返回true，执行if中的代码
       if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
           synchronized (this.singletonObjects) {
               // 3\. 从二级缓存中获取 service1，能获取到
               singletonObject = this.earlySingletonObjects.get(beanName);
               // 4\. singletonObject 不为null，if块的代码不会执行
               if (singletonObject == null && allowEarlyReference) {
                   ...
               }
           }
       }
       return singletonObject;
   }
   
   ```

   我们看 一眼 5 大结构中的内容，对这个方法的执行就一目了然了：

   | 结构                            | 内容                            |
      | ------------------------------- | ------------------------------- |
   | `singletonObjects`              | service2$$EnhancerBySpringCGLIB |
   | `earlySingletonObjects`         | service1$$EnhancerBySpringCGLIB |
   | `singletonFactories`            |                                 |
   | `singletonsCurrentlyInCreation` | service1                        |
   | `earlyProxyReferences`          |                                 |

   方法的执行，在代码中已经注释得很明白了，就多说了，这一步会把 `service1` 的代理对象返回；

4. 上一步得到的 `earlySingletonReference` 为

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-425636328820897c75890f01614d94fe9dd.png)

   代理对象已经返回来了，可以继续执行 if 块的代码了；

5. 最后一步是赋值操作，将返回的对象赋值给 `exposedObject`，然后返回 `exposedObject`，最终返回的对象为 `service1` 的代理对象。

到了这一步，`service1` 的代理对象也获取完成了。

#### 6.2 回到 `DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)`

这一步的操作是清理一些缓存，就不再分析了，最终 5 个结构中的内容如下：

| 结构                            | 内容                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `singletonObjects`              | service2<nobr aria-hidden="true">EnhancerBySpringCGLIB,service1</nobr><math xmlns="http://www.w3.org/1998/Math/MathML" display="block"><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mo>,</mo><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mi>?</mi><mn>1</mn></math><mi>E</mi><mi>n</mi><mi>h</mi><mi>a</mi><mi>n</mi><mi>c</mi><mi>e</mi><mi>r</mi><mi>B</mi><mi>y</mi><mi>S</mi><mi>p</mi><mi>r</mi><mi>i</mi><mi>n</mi><mi>g</mi><mi>C</mi><mi>G</mi><mi>L</mi><mi>I</mi><mi>B</mi><mo>,</mo><mi>s</mi><mi>e</mi><mi>r</mi><mi>v</mi><mi>i</mi><mi>c</mi><mi>e</mi><mn>1</mn></math>" role="presentation">EnhancerBySpringCGLIB |
| `earlySingletonObjects`         |                                                              |
| `singletonFactories`            |                                                              |
| `singletonsCurrentlyInCreation` |                                                              |
| `earlyProxyReferences`          |                                                              |

通过调试，查看得到 `singletonObjects` 中对象如下：

1. service1

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b6ba35a7a701056ae55a8f9b2ef9345cac5.png)

2. service2

   ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-563d6e323ef8fc5e9fd02f7a938ddbac655.png)

可以看到，`singletonObjects` 中两者都是代理对象，彼此注入的，也是代理对象，循环依赖得到了解决。

### 4\. 总结

spring 循环依赖的分析到这里就结束了，循环依赖有解的核心在于**对象可以提前进行 aop**，spring 正是利用这一点，再配合 5 大结构存储必要的信息，一步步解决循环依赖问题。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4815992](https://my.oschina.net/funcy/blog/4815992) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_