

上一篇文章的最后，我们分析到 spring 终于创建了代理对象，其中代理对象的方式为 `jdk动态代理`与 `cglib代理`，本文我们将分析 spring 的动态代理。

### 1. jdk 动态代理介绍

来分析 spring 的动态代理前，我们先来了解下 jdk 的动态代理。jdk 动态代理需要接口，为此我们先准备两个接口：

> IJdkDynamicProxy01

```java
package org.springframework.learn.demo03;

public interface IJdkDynamicProxy01 {
    void hello01();
}
```

> IJdkDynamicProxy02

```java
package org.springframework.learn.demo03;

public interface IJdkDynamicProxy02 {
    void hello02();
}
```

再来准备两个实现类：

> JdkDynamicProxyImpl01

```java
package org.springframework.learn.demo03;

public class JdkDynamicProxyImpl01 implements IJdkDynamicProxy01, IJdkDynamicProxy02{
    @Override
    public void hello01() {
        System.out.println("hello01");
    }

    @Override
    public void hello02() {
        System.out.println("hello02");
    }
}
```

> JdkDynamicProxyImpl02

```java
package org.springframework.learn.demo03;

public class JdkDynamicProxyImpl02 implements IJdkDynamicProxy01 {

    @Override
    public void hello01() {
        System.out.println("hello01");
    }

}
```

这里需要注意的是，`JdkDynamicProxyImpl01` 实现了 `IJdkDynamicProxy01` 与 `IJdkDynamicProxy02` 两个接口，`JdkDynamicProxyImpl02` 只实现了 `IJdkDynamicProxy01` 一个 接口。

接着准备一个 `InvocationHandler`:

> MyInvocationHandler

```java
package org.springframework.learn.demo03;

import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;

public class MyInvocationHandler implements InvocationHandler {

     /** 目标对象 */
     private Object target;

    public MyInvocationHandler(Object target){
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("执行方法为:" + method.getName());
        // 方法的真正执行在这里
        Object rs = method.invoke(target,args);
        return rs;
    }

}
```

最后是主类:

```java
package org.springframework.learn.demo03;

import java.lang.reflect.Proxy;

public class Demo03Main {

    public static void main(String[] args) {
        System.out.println("------------bean01------------");
        JdkDynamicProxyImpl01 bean01 = new JdkDynamicProxyImpl01();
        Object obj1 = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                // JdkDynamicProxyImpl01实现了 IJdkDynamicProxy01, IJdkDynamicProxy02
                // 传入的class为 IJdkDynamicProxy01, IJdkDynamicProxy02
                new Class<?>[]{ IJdkDynamicProxy01.class, IJdkDynamicProxy02.class },
                new MyInvocationHandler(bean01));
        // 可以进行类型强制转换
        ((IJdkDynamicProxy01) obj1).hello01();
        ((IJdkDynamicProxy02) obj1).hello02();

        System.out.println("------------bean01------------");
        Object obj2 = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                 // JdkDynamicProxyImpl01实现了 IJdkDynamicProxy01, IJdkDynamicProxy02
                 // 传入的class为 IJdkDynamicProxy01
                 new Class<?>[]{ IJdkDynamicProxy01.class },
                 new MyInvocationHandler(bean01));
        ((IJdkDynamicProxy01) obj2).hello01();
        // 报异常：java.lang.ClassCastException: class com.sun.proxy.$Proxy1 cannot be cast to class xxx
        //((IJdkDynamicProxy02) obj2).hello02();

        System.out.println("-----------bean02-------------");
        JdkDynamicProxyImpl02 bean02 = new JdkDynamicProxyImpl02();
        Object obj3 = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                 // JdkDynamicProxyImpl01实现了 IJdkDynamicProxy01
                 // 传入的class为 IJdkDynamicProxy01, IJdkDynamicProxy02
                 new Class<?>[]{ IJdkDynamicProxy01.class, IJdkDynamicProxy02.class },
                 new MyInvocationHandler(bean02));
        ((IJdkDynamicProxy01) obj3).hello01();
        IJdkDynamicProxy02 proxy02 = (IJdkDynamicProxy02) obj3;
        // 报异常：java.lang.IllegalArgumentException: object is not an instance of declaring class
        //proxy02.hello02();

    }
}
```

运行结果：

```
执行方法为:hello01
hello01
执行方法为:hello02
hello02
------------bean01------------
执行方法为:hello01
hello01
-----------bean02-------------
执行方法为:hello01
hello01
```

对结果分析如下：

1. `Proxy#newProxyInstance(ClassLoader, Class<?>[], InvocationHandler)` 的第二个参数传入的接口，声明的是代理对象所属的接口类型，第三个参数是执行器，代理类对象的执行为其 `invoke()` 方法；
2. `JdkDynamicProxyImpl01` 同时实现了 `IJdkDynamicProxy01` 与 `IJdkDynamicProxy02` 接口，但传入接口类型时，只传入了 `IJdkDynamicProxy01`，当代理对象 obj2 强转为 `IJdkDynamicProxy02` 时，就会报 `ClassCastException`，强转失败，这表明 `Proxy#newProxyInstance(ClassLoader, Class<?>[], InvocationHandler)` 声明的是代理对象所属的接口类型；
3. `JdkDynamicProxyImpl02` 只实现了 `IJdkDynamicProxy01` 接口，但传入接口类型时，传入了 `IJdkDynamicProxy01` 与 `IJdkDynamicProxy02`，当代理对象 `obj3` 强转为 `IJdkDynamicProxy02` 时，并未报异常，但是在执行 `proxy02.hello02()` 时，却报了 `java.lang.IllegalArgumentException: object is not an instance of declaring class`，同样表明了 `Proxy#newProxyInstance(ClassLoader, Class<?>[], InvocationHandler)` 声明的是代理对象所属的接口类型，跟目标对象的类型无关。

### 2. 再次分析 spring jdk 动态代理对象的创建

有了上面的分析，我们再来看看 spring 是如何创建代理对象的：

```java
@Override
public Object getProxy(@Nullable ClassLoader classLoader) {
    // 获取目标对象实现的接口
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    // 是否有equals()与hashCode()方法
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    // 调用 jdk 方法 创建对象
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

1. 传入的接口为 `proxiedInterfaces`，这个值包含了目标类实现的所有接口，同时 spring 也会添加自身的接口，如 `SpringProxy`、`Advised`，这些在上一篇文章已经详细分析过了；
2. 指定 `InvocationHandler` 为 `this`，也就是 `JdkDynamicAopProxy` 的对象，实际上 `JdkDynamicAopProxy` 实现了 `InvocationHandler`.

由第一部分的分析可知，jdk 动态代理对象的方法最终是在 `java.lang.reflect.InvocationHandler#invoke` 中执行的，也就是 `JdkDynamicAopProxy#invoke`，接下来我们就来分析 `JdkDynamicAopProxy#invoke` 方法，来看看 spring 是如何执行代理方法的。

### 3. jdk 动态代理方法的执行

spring jdk 动态代理方法的执行在 `JdkDynamicAopProxy#invoke`：

> JdkDynamicAopProxy#invoke

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;

    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    try {
        // 若执行的 equals 方法，不需要代理执行
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            return equals(args[0]);
        }
        // 若执行的 hashCode 方法，不需要代理执行
        else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            return hashCode();
        }
        // 如果执行的class对象是DecoratingProxy，也不需要代理执行
        else if (method.getDeclaringClass() == DecoratingProxy.class) {
            return AopProxyUtils.ultimateTargetClass(this.advised);
        }
        else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        Object retVal;

        // 判断 advised的exposeProxy 值是否为 true
        // advised的exposeProxy来源于 @EnableAspectJAutoProxy 的 exposeProxy
        // 即 像这样指定时，@EnableAspectJAutoProxy(exposeProxy = true)，以下代码执行
        if (this.advised.exposeProxy) {
            // 将当前的 proxy 对象放到 threadLocal 中
            // 后续可以 (UserService (AopContext.currentProxy)).getUser() 方式调用
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // 获取目标对象及目标对象的class
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        // 将 aop 的 advisor 转化为拦截器，在这里判断该方法可以使用哪些切面方法
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
                method, targetClass);
        if (chain.isEmpty()) {
            // 如果加入的拦截器链为空，表明该方法没有被拦截，通过反射直接执行
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        }
        else {
            // 创建一个方法调用对象
            MethodInvocation invocation =
                   new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            // 调用执行，重点
            retVal = invocation.proceed();
        }

        Class<?> returnType = method.getReturnType();
        if (retVal != null && retVal == target &&
                returnType != Object.class && returnType.isInstance(proxy) &&
                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
            retVal = proxy;
        }
        else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
            throw new AopInvocationException(...);
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

以上方法流程如下：

1. 判断要执行的方法是否为 `equals`、`hashcode` 等，这些方法不需要代理；
2. 获取可用于要执行的方法的所有切面方法，得到一个拦截器集合；
3. 调用切面方法集合与及目标方法。

这里我们重点关注切面方法与目标方法的执行，关键代码如下：

```java
// 将 aop 的 advisor 转化为拦截器，在这里判断该方法可以使用哪些切面方法
List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(
        method, targetClass);
// 创建一个方法调用对象
MethodInvocation invocation =
       new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
// 调用执行，重点
retVal = invocation.proceed();
```

#### 获取 `MethodInterceptor`

在分析方法的执行前，我们先来看看 `getInterceptorsAndDynamicInterceptionAdvice(...)`，这个方法是用来获取执行的切面方法的，也就是 `MethodInterceptor`：

> AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, 
        @Nullable Class<?>targetClass) {
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    List<Object> cached = this.methodCache.get(cacheKey);
    if (cached == null) {
        // 在这个方法里继续获取，下面会再分析
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}
```

继续，

```java
/**
 * 获取 Interceptor，过程如下：
 */
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
        Advised config, Method method, @Nullable Class<?> targetClass) {
    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
    // 获取 advisors，aop的advisors如下：
    Advisor[] advisors = config.getAdvisors();
    List<Object> interceptorList = new ArrayList<>(advisors.length);
    Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
    Boolean hasIntroductions = null;
    for (Advisor advisor : advisors) {
        // 如果advisor是PointcutAdvisor，则使用PointcutAdvisor里的Pointcut进行匹配
        if (advisor instanceof PointcutAdvisor) {
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
            // 这里判断切面逻辑的调用链是否提前进行过过滤，如果进行过，则不再进行目标方法的匹配，
            // 如果没有，则再进行一次匹配。
            if (config.isPreFiltered() 
                    || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                boolean match;
                if (mm instanceof IntroductionAwareMethodMatcher) {
                    if (hasIntroductions == null) {
                        hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
                    }
                    match = ((IntroductionAwareMethodMatcher) mm)
                            .matches(method, actualClass, hasIntroductions);
                }
                else {
                    match = mm.matches(method, actualClass);
                }
                if (match) {
                    // 将Advisor对象转换为MethodInterceptor数组
                    MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                    if (mm.isRuntime()) {
                        for (MethodInterceptor interceptor : interceptors) {
                            // 将 interceptor与methodMatcher包装成InterceptorAndDynamicMethodMatcher
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
                        }
                    }
                    else {
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        }
        else if (advisor instanceof IntroductionAdvisor) {
            // 判断如果为IntroductionAdvisor类型的Advisor，则将调用链封装为Interceptor数组
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }
        else {
            // 这里是提供的使用自定义的转换器对Advisor进行转换的逻辑，因为getInterceptors()方法中
            // 会使用相应的Adapter对目标Advisor进行匹配，如果能匹配上，通过其getInterceptor()方法
            // 将自定义的Advice转换为MethodInterceptor对象
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }
    return interceptorList;
}
```

我们根据以上方法来总结下获取 `MethodInterceptor` 的过程过程如下：

1. 获取项目中所有的 `advisors`

2. 遍历，对每个



   ```
   advisor
   ```



继续按以下流程处理：

1. 如果 `advisor` 是 `PointcutAdvisor`，则使用其中的 `Pointcut` 进行匹配，匹配成功后，获取 `MethodInterceptor` 返回；
2. 如果 `advisor` 是 `IntroductionAdvisor`，则使用其中的 `ClassFilter` 进行匹配，匹配成功后，获取 `MethodInterceptor` 返回；
3. 如果以上条件不满足，直接获取 `MethodInterceptor` 返回；

那么 `MethodInterceptor` 是如何获取的呢？我们继续往下看：

```java
// 存放 AdvisorAdapter 的地方
private final List<AdvisorAdapter> adapters = new ArrayList<>(3);

// 添加 adapter
public DefaultAdvisorAdapterRegistry() {
    // @Before
    registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
    // @AfterReturning
    registerAdvisorAdapter(new AfterReturningAdviceAdapter());
    // @AfterThrowing
    registerAdvisorAdapter(new ThrowsAdviceAdapter());
}

/**
 * 获取advisor对应的MethodInterceptor
 */
@Override
public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
    List<MethodInterceptor> interceptors = new ArrayList<>(3);
    // 获取当前advisor里的MethodInterceptor
    Advice advice = advisor.getAdvice();
    // 如果 advice 是 MethodInterceptor的实例，添加
    if (advice instanceof MethodInterceptor) {
        interceptors.add((MethodInterceptor) advice);
    }
    // 
    // 使用 AdvisorAdapter 将 advice 转换为 MethodInterceptor
    // 如果advice满足adapter，调用 adapter.getInterceptor 获取 MethodInterceptor
    for (AdvisorAdapter adapter : this.adapters) {
        if (adapter.supportsAdvice(advice)) {
            interceptors.add(adapter.getInterceptor(advisor));
        }
    }
    if (interceptors.isEmpty()) {
        throw new UnknownAdviceTypeException(advisor.getAdvice());
    }
    return interceptors.toArray(new MethodInterceptor[0]);
}
```

对这个方法的流程总结如下：

1. 如果 `advice` 是 `MethodInterceptor`，直接将其转换成 `MethodInterceptor`；
2. 如果以上不满足，则使用 `AdvisorAdapter` 将 advice 转换成 `MethodInterceptor`.

关于 `adapters`，spring 为提供了三个 `Adapter`：

- MethodBeforeAdviceAdapter：处理 `@Before`
- AfterReturningAdviceAdapter：处理 `@AfterReturning`
- ThrowsAdviceAdapter：处理 `@AfterThrowing`

这三个 `Adapter` 就只有一个功能：返回 `advice` 对应的 `MethodInterceptor`，我们来看下 `MethodBeforeAdviceAdapter` 的代码就明白了：

```java
class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {
    /**
     * 是否能处理当前advice
     */
    @Override
    public boolean supportsAdvice(Advice advice) {
        return (advice instanceof MethodBeforeAdvice);
    }

    /**
     * 返回对应的MethodInterceptor
     */
    @Override
    public MethodInterceptor getInterceptor(Advisor advisor) {
        MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
        return new MethodBeforeAdviceInterceptor(advice);
    }
}
```

其他两个 `Adapter` 的功能极其相似，就不分析了，这里总结下各注解对应的 `advice`、`methodInterceptor`：

| 注解            | advice                      | methodInterceptor               |
| --------------- | --------------------------- | ------------------------------- |
| @Before         | AspectJMethodBeforeAdvice   | MethodBeforeAdviceInterceptor   |
| @After          | AspectJAfterAdvice          | AspectJAfterAdvice              |
| @Around         | AspectJAroundAdvice         | AspectJAroundAdvice             |
| @AfterReturning | AspectJAfterReturningAdvice | AfterReturningAdviceInterceptor |
| @AfterThrowing  | AspectJAfterThrowingAdvice  | ThrowsAdviceInterceptor         |

#### ReflectiveMethodInvocation#proceed

获取完 `MethodInterceptor` 后，就开始进行方法的执行了，我们直接进入 `ReflectiveMethodInvocation#proceed` 方法：

```java
public Object proceed() throws Throwable {
    // 执行完所有的增强后执行目标方法
    // 这里使用了责任链模式，这个方法在责任链中调用，满足条件表示当前责任链已经执行到最后了
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    // 获取下一个要执行的拦截器
    Object interceptorOrInterceptionAdvice =
           this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        Class<?> targetClass = (this.targetClass != null 
                ? this.targetClass : this.method.getDeclaringClass());
        if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
            // 匹配，就调用拦截器的方法，也就是切面方法
            // 在 MethodInterceptor#invoke里，会再次 ReflectiveMethodInvocation#proceed，又调回了当前方法
            return dm.interceptor.invoke(this);
        }
        else {
            // 不匹配，则递归调用当前方法
            return proceed();
        }
    }
    else {
        // 注意，这个方法传入的参数是 this，表示当前对象
        // 在 MethodInterceptor#invoke里，会再次 ReflectiveMethodInvocation#proceed，又调回了当前方法
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}

/**
 * 调用目标方法
 */
protected Object invokeJoinpoint() throws Throwable {
    // 使用反射调用目标对象方法，注意这里传入的应该是目标对象，而不是代理对象
    return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
}
```

以上代码的调用使用了责任链模式，执行逻辑如下：

1. 判断是否执行完所有的切面方法，若是，则执行目标方法，否则执行下一步；
2. 获取下一个拦截器，判断能否执行，若能，则调用拦截器方法，否则执行第 一步操作；

以上逻辑看着挺简单，但具体是怎么执行的呢？在 spring 中，切面通知有五种类型：`@Before`、`@After`、`@AfterReturning`、`@AfterThrowing` 与 `@Around`，这里我们一一来看看这五种通知是如何调用的。

#### 1. `@Before`

> MethodBeforeAdviceInterceptor#invoke

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    // 执行前置通知，看下面
    this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
    // 继续执行下一个拦截器
    return mi.proceed();
}
```

进入 `advice.before(xxx)` 方法：

> AspectJMethodBeforeAdvice#before

```java
@Override
public void before(Method method, Object[] args, @Nullable Object target) throws Throwable {
    invokeAdviceMethod(getJoinPointMatch(), null, null);
}
```

继续跟下去：

> AbstractAspectJAdvice#invokeAdviceMethod(JoinPointMatch, Object, Throwable)

```java
protected Object invokeAdviceMethod(
         @Nullable JoinPointMatch jpMatch, @Nullable Object returnValue, @Nullable Throwable ex)
         throws Throwable {

     return invokeAdviceMethodWithGivenArgs(argBinding(getJoinPoint(), jpMatch, returnValue, ex));
}

/**
  *  调用反射执行
  */
protected Object invokeAdviceMethodWithGivenArgs(Object[] args) throws Throwable {
    Object[] actualArgs = args;
    if (this.aspectJAdviceMethod.getParameterCount() == 0) {
        actualArgs = null;
    }
    try {
        // 熟悉的jdk反射代码
        ReflectionUtils.makeAccessible(this.aspectJAdviceMethod);
        return this.aspectJAdviceMethod.invoke(
                this.aspectInstanceFactory.getAspectInstance(), actualArgs);
    }
    catch (...) {
        ...
}
```

可以看到，最终是调用 jdk 反射来调用的。

#### 2. `@After`

> AspectJAfterAdvice#invoke

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
        // 继续执行下一个拦截器
        return mi.proceed();
    }
    finally {
        // 调用切面方法：放在 finally 块，表示一定会执行，最终也是使用反射调用
        invokeAdviceMethod(getJoinPointMatch(), null, null);
    }
}
```

#### 3. `@AfterReturning`

> AfterReturningAdviceInterceptor#invoke

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    // 继续执行下一个拦截器
    Object retVal = mi.proceed();
    // 调用切面方法，继续往下看
    this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
    return retVal;
}
```

> AspectJAfterReturningAdvice#afterReturning

```java
public void afterReturning(@Nullable Object returnValue, Method method, 
            Object[] args, @Nullable Object target) throws Throwable {
    if (shouldInvokeOnReturnValueOf(method, returnValue)) {
        // 调用切面方法，依然是调用反射执行
        invokeAdviceMethod(getJoinPointMatch(), returnValue, null);
    }
}
```

#### 4. `@AfterThrowing`

> AspectJAfterThrowingAdvice#invoke

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
        // 调用 ReflectiveMethodInvocation#proceed
        return mi.proceed();
    }
    catch (Throwable ex) {
        if (shouldInvokeOnThrowing(ex)) {
            // 调用切面方法，只有在抛出异常时才会被调用
            invokeAdviceMethod(getJoinPointMatch(), null, ex);
        }
        throw ex;
    }
}
```

#### 5. `@Around`

> AspectJAroundAdvice#invoke

```java
@Override
public Object invoke(MethodInvocation mi) throws Throwable {
    if (!(mi instanceof ProxyMethodInvocation)) {
        throw new IllegalStateException(...);
    }
    ProxyMethodInvocation pmi = (ProxyMethodInvocation) mi;
    ProceedingJoinPoint pjp = lazyGetProceedingJoinPoint(pmi);
    JoinPointMatch jpm = getJoinPointMatch(pmi);
    // 调用切面方法
    return invokeAdviceMethod(pjp, jpm, null, null);
}
```

我们在实现环绕通知时，一般像这样实现：

```java
@Around(xxx)
public Object around(ProceedingJoinPoint p){
    // 执行目标方法前的操作
    ...

    // 执行目标方法，这一句是关键
    // 实际这里并不仅仅是执行目标方法，最终调用的是 ReflectiveMethodInvocation#proceed
    // 即如果有拦截器，则继续执行下一个拦截器，否则执行目标方法
    Object o = p.proceed();

    // 执行目标方法后的操作
    ...
    return o;
}
```

spring 对以上五种通知的执行，主要分为两个部分：

1. 使用反射方式执行切面方法，也就是所谓的 “增强”；
2. 调用 `ReflectiveMethodInvocation#proceed` 继续执行下一个拦截器，或执行目标方法。

以上两个部分的差别仅在于执行次序及执行的位置，如

- `@Before` 通知，`1` 在前，`2` 在后；
- `@AfterReturning` 通知，`2` 在前，`1` 在后，如果执行 `2` 时发生了异常，`1` 就不会执行了；
- `@AfterThrowing` 通知，`2` 在前，`1` 在后，且 `1` 是放在 `catch` 块中执行，即只有发生了异常，`1` 才会执行；
- `@After` 通知，`2` 在前，`1` 在后，`1` 是放在 `finally` 块中执行，由于 `finally` 的特殊性，即使发生了异常，`1` 同样会执行；
- `@Around` 通知，在切面方法中自行指定了 `2` 的执行时机。

注意区别 `@AfterReturning`、`@AfterThrowing` 与 `@After` 通知的执行时机。

最后，我们再来看看这几个通知是如何执行的。

通过调试的方式，发现 spring 执行通知的顺序如下：

1. 首先执行 `@AfterThrowing` 通知，先调用 `mi.proceed()` 执行下一个拦截器，然后在 `catch` 块中执行切面方法，表明只有出现异常时，该切面方法才会执行；
2. 上一步操作中，调用 `mi.proceed()` 时会执行 `@AfterReturning` 通知，执行时先调用 `mi.proceed()` 执行下一个拦截器，然后再执行切面方法；
3. 上一步操作中，调用 `mi.proceed()` 时会执行 `@After` 通知，执行时先调用 `mi.proceed()` 执行下一个拦截器，然后在 `finally` 块中执行切面方法，表明即使出现异常，切面方法还是会执行；
4. 上一步操作中，调用 `mi.proceed()` 时会执行 `@Around` 通知，执行时直接执行切面方法，由于 `@Around` 通知的切面方法里会调用 `ProceedingJoinPoint#proceed()`，最终还是会执行下一个拦截器；
5. 上一步操作中，调用 `mi.proceed()` 时会执行 `@Before` 通知，执行时会先执行切面方法，再调用 `mi.proceed()` 执行下一个拦截器；
6. 拦截器链执行到最后，发现没有可执行的拦截器了，此时就开始执行目标方法。

画图示意通知的执行过程如下：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-e88e0cec49c47648005e5d3160663425739.png)

最终的执行顺序：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-97026239d0b2dc02abbe87a9b76325c3cc0.png)

### 4. 总结

本文主要分析了 jdk 动态代理的执行过程，梳理了各个切面通知的执行顺序。本文就先到这里了，下一篇文章将介绍 cglib 的执行过程。

------

*本文原文链接：https://my.oschina.net/funcy/blog/4696654 ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。*