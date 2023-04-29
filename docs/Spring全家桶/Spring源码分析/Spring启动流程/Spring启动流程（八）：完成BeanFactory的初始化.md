![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1f1ac8f3d8241fad9693d9684048ab7f3ae.png)

接上文，本文继续分析 spring 的启动 流程。

### 11\. 初始化单例 bean: `finishBeanFactoryInitialization(beanFactory)`

本文将分析一个**非常重要**的方法 `AbstractApplicationContext#finishBeanFactoryInitialization` 了。

方法的调用如下：

```
|-AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#finishBeanFactoryInitialization
   |-DefaultListableBeanFactory#preInstantiateSingletons

```

我们直接进入 `DefaultListableBeanFactory#preInstantiateSingletons`:

```
public void preInstantiateSingletons() throws BeansException {
    // this.beanDefinitionNames 保存了所有的 beanNames
    List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

    for (String beanName : beanNames) {
        // 合并父 Bean 中的配置，注意<bean id=""  parent="" /> 中的 parent属性
        RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
        // 不是抽象类、是单例的且不是懒加载的
        if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
            // 处理 FactoryBean
            if (isFactoryBean(beanName)) {
                //在 beanName 前面加上“&” 符号
                Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
                if (bean instanceof FactoryBean) {
                     final FactoryBean<?> factory = (FactoryBean<?>) bean;
                     // 判断当前 FactoryBean 是否是 SmartFactoryBean 的实现
                     boolean isEagerInit;
                     if (System.getSecurityManager() != null 
                            && factory instanceof SmartFactoryBean) {
                        isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
                                ((SmartFactoryBean<?>) factory)::isEagerInit,
                                getAccessControlContext());
                     }
                     else {
                          isEagerInit = (factory instanceof SmartFactoryBean &&
                             ((SmartFactoryBean<?>) factory).isEagerInit());
                     }
                     if (isEagerInit) {
                           // 不是FactoryBean的直接使用此方法进行初始化
                           getBean(beanName);
                     }
                }
            }
            else {
                getBean(beanName);
            }
        }
    }

    // Trigger post-initialization callback for all applicable beans...
    // 如果bean实现了 SmartInitializingSingleton 接口的，那么在这里得到回调
    for (String beanName : beanNames) {
        Object singletonInstance = getSingleton(beanName);
        if (singletonInstance instanceof SmartInitializingSingleton) {
            final SmartInitializingSingleton smartSingleton = 
                    (SmartInitializingSingleton) singletonInstance;
            if (System.getSecurityManager() != null) {
                AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
                    smartSingleton.afterSingletonsInstantiated();
                    return null;
                }, getAccessControlContext());
            }
            else {
                smartSingleton.afterSingletonsInstantiated();
            }
        }
    }
}

```

以上代码，看似很多，但关键代码可简化如下：

```
List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);
for (String beanName : beanNames) {
    RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
    if(... && bd.isSingleton() && ...) {
        getBean(beanName);
    }
}

```

> 以上代码的简化中，省略了许多细节，例如判断是否可以实例化时，需要判断是否为抽象类，是否为单例，是否为懒加载等，同时对 bean 也判断了是否为普通 bean 还是 `FactoryBean`，如果实现了 `SmartInitializingSingleton` 接口的 bean，需要另外处理等。对于我们初次阅读代码来说，把精力聚集主要流程就行了，对于一些特例及细节，可以先不纠结，想了解更多细节，可以在把握主要流程的情况下，后面再看。过于纠结特例及细节，反倒让自己抓不住重点，迷失于源码中。

这样一简化，就可以清晰看出该方法的功能：

1.  获取 `beanFactory` 中的 `beanNames` 并遍历；
2.  通过 `beanName` 获取 `BeanDefinition`，进行条件判断，如是否为单例；
3.  遍历调用 `getBean(beanName)` 创建 bean 并将其添加到 spring 中。

从简化后的代码可以看出，看似平平无奇的 `getBean(beanName)`，就是 spring 实例化 bean 的关键。接下来，我们还是忽略其他代码，只关注主要流程，继续分析下去：

```
|-AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)
 |-AbstractApplicationContext#refresh
  |-AbstractApplicationContext#finishBeanFactoryInitialization
   |-DefaultListableBeanFactory#preInstantiateSingletons
    |-AbstractBeanFactory#getBean(java.lang.String)
     |-AbstractBeanFactory#doGetBean

```

> AbstractBeanFactory#doGetBean

```
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
         @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // 主要逻辑就是如果是FactoryBean就把&去掉,如果是别名就把根据别名获取真实名称
    final String beanName = transformedBeanName(name);
    //最后的返回值
    Object bean;

    // 检查是否已初始化
    Object sharedInstance = getSingleton(beanName);
    // 如果已经初始化过了，且没有传args参数就代表是get，直接取出返回
    if (sharedInstance != null && args == null) {
        // 这里如果是普通Bean 的话，直接返回，如果是 FactoryBean 的话，返回它创建的那个实例对象
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    else {
        // 如果存在prototype类型的这个bean
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }

        // 如果当前BeanDefinition不存在这个bean且具有父BeanFactory
        BeanFactory parentBeanFactory = getParentBeanFactory();
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            String nameToLookup = originalBeanName(name);
            if (parentBeanFactory instanceof AbstractBeanFactory) {
                return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
                   nameToLookup, requiredType, args, typeCheckOnly);
            }
            else if (args != null) {.
                 // 返回父容器的查询结果
                 return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else if (requiredType != null) {
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
            else {
                return (T) parentBeanFactory.getBean(nameToLookup);
            }
        }

        if (!typeCheckOnly) {
            // typeCheckOnly 为 false，将当前 beanName 放入一个 alreadyCreated 的 Set 集合中。
            markBeanAsCreated(beanName);
        }

        // 到这就要创建bean了
        try {
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);    
            // 先初始化依赖的所有 Bean， depends-on 中定义的依赖
            String[] dependsOn = mbd.getDependsOn();
            if (dependsOn != null) {
                for (String dep : dependsOn) {
                    // 检查是不是有循环依赖
                    if (isDependent(beanName, dep)) {
                        throw new BeanCreationException(...);
                    }
                    // 注册一下依赖关系
                    registerDependentBean(dep, beanName);
                    try {
                        // 先初始化被依赖项
                        getBean(dep);
                    }
                    catch (NoSuchBeanDefinitionException ex) {
                        throw new BeanCreationException(...);
                    }
                }
            }

            // 如果是单例的
            if (mbd.isSingleton()) {
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
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }
            // 如果是prototype
            else if (mbd.isPrototype()) {
                Object prototypeInstance = null;
                try {
                    beforePrototypeCreation(beanName);
                    // 执行创建 Bean
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    afterPrototypeCreation(beanName);
                }
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }
            // 如果不是 singleton 和 prototype, 那么就是自定义的scope(例如Web项目中的session等类型)，
            // 这里就交给自定义scope的应用方去实现
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                if (scope == null) {
                  throw new IllegalStateException(...);
                }
                try {
                    Object scopedInstance = scope.get(beanName, () -> {
                        beforePrototypeCreation(beanName);
                        try {
                            // 执行创建 Bean
                            return createBean(beanName, mbd, args);
                        }
                        finally {
                            afterPrototypeCreation(beanName);
                        }
                    });
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                  throw new BeanCreationException(...);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    //检查bean的类型
    if (requiredType != null && !requiredType.isInstance(bean)) {
        try {
            T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
            if (convertedBean == null) {
                throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
            }
            return convertedBean;
        }
        catch (TypeMismatchException ex) {
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}

```

上面的代码基本上给出了注释， 在这就不多做 解释了。spring 功能比较复杂，考虑的东西也比较多，因此上述代码会多做多种判断，应对多种情况。如果我们仅考虑 demo01 的情况 (`singleton` 情况)，以上代码关键如下：

```
//最后的返回值
Object bean;

// 这里仅给出单例的情况
// 1\. 获取单例对象，同时也提供了一个lambda表达式，用来进行bean的创建
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

// 2\. 进一步处理sharedInstance，然后返回bean。事实上，这个方法主要处理的是，
// 如果是FactoryBean，就返回返回它创建的那个实例对象，否则就直接返回
bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);

return (T) bean;

```

精简后的代码，就是 spring 创建 bean 的流程。接着我们就分别看看这两个方法的内容，这里删除了一些不必要的代码：

> `DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory<?>)`

```
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
    synchronized (this.singletonObjects) {
        boolean newSingleton = false;
        try {
            // 对象就是在这里进行创建的
            singletonObject = singletonFactory.getObject();
            newSingleton = true;
        }
        catch (... ex) {
            ...
        }
        finally {
            // 创建完成后，做一些判断操作，与创建过程关系不大
            afterSingletonCreation(beanName);
        }
        if (newSingleton) {
            // 添加到  beanFactory 缓存
            addSingleton(beanName, singletonObject);
        }
        return singletonObject;
    }
}

```

从以上代码可以看出，bean 的创建就是在 `singletonFactory.getObject()`，关于这个方法执行了什么，我们还应该结合 `AbstractBeanFactory#doGetBean`.

首先，我们进入 `ObjectFactory#getObject`，发现代码如下：

```
@FunctionalInterface
public interface ObjectFactory<T> {
    T getObject() throws BeansException;
}

```

这是一个函数式编程接口，jdk8 提供的新语法。再看看 `AbstractBeanFactory#doGetBean` 方法传入的对象：

```
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

```

这里传入的是一个 lambda 表达式，当代码执行 `singletonFactory.getObject()` 时，实际上执行的是

```
try {
    // 执行创建 Bean
    return createBean(beanName, mbd, args);
}
catch (BeansException ex) {
    destroySingleton(beanName);
    throw ex;
}

```

即 `AbstractAutowireCapableBeanFactory#createBean(String, RootBeanDefinition, Object[])`，前面弯弯绕绕了那么多，spring 终于要进入 bean 的创建了！

关于 spring 对象的创建，我们会在后面的文章继续分析，本文我们只关注 `AbstractBeanFactory#getBean` 与 `DefaultSingletonBeanRegistry#getSingleton`，结合以上分析，我们对这两个方法做一个总结：

*   `AbstractBeanFactory#getBean`：对于 scope 为 `PropertyType` 的 bean 来说，该方法会直接创建 bean；对于 scope 为 `singleton` 的 bean 来说，该方法会先判断 `beanFactory` 是否存在该 bean，若存在则直接返回，否则就先创建再返回。

*   `DefaultSingletonBeanRegistry#getSingleton`：这个方法就是从 `beanFactory` 获取 singleton bean 的方法：若存在则直接返回，否则就先创建再返回。

本文就先分析到这里了，下篇文章我们再分析 spring bean 创建过程。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4658230](https://my.oschina.net/funcy/blog/4658230) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_