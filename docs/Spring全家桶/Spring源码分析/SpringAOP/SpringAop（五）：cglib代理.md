

[上一篇文章](https://my.oschina.net/funcy/blog/4696654 "上一篇文章")介绍了 spring 的 jdK 动态代理，本文来介绍 spring 的 cglib 代理。

### 1\. cglib 代理简介

jdk 虽然提供了动态代理，但是动态代理有一个不足：**如果类没有实现接口，就无法 jdk 动态代理**。为了解决这个不足，spring 又引入了 cglib 代理。

cglib 底层是基于 asm 的，也就是直接操作字节码，相当于对 asm 进行了一层封装。直接操作字码，需要对 java 指令、字节码文件有深入理解才能进行，但字节码晦涩难懂，一般不建议直接操作。经 cglib 封装后，字节码的操作就变得简单多了，因此**绝大多数情况下都建议使用 cglib 封装好的方法来进行字节码操作**。

spring cglib 操作位于 `spring-core` 模块：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-6b64440344221ca08ed8c822b5b22c1d341.png)

再来看看 asm 与 cglib 包说明：

```
/**
 * Spring's repackaging of
 * ASM 7.0
 * (with Spring-specific patches; for internal use only).
 *
 * <p>This repackaging technique avoids any potential conflicts with
 * dependencies on ASM at the application level or from third-party
 * libraries and frameworks.
 *
 * <p>As this repackaging happens at the class file level, sources
 * and javadocs are not available here.
 */
 package org.springframework.asm;

```

注意第一句：`Spring's repackaging of ASM 7.0`，表明这是 spring 对 `asm7.0` 重新打包.

```
/**
 * Spring's repackaging of
 * CGLIB 3.3
 * (with Spring-specific patches; for internal use only).
 *
 * <p>This repackaging technique avoids any potential conflicts with
 * dependencies on CGLIB at the application level or from third-party
 * libraries and frameworks.
 *
 * <p>As this repackaging happens at the class file level, sources
 * and javadocs are not available here.
 */
package org.springframework.cglib;

```

注意第一句：`Spring's repackaging of CGLIB 3.3`，表明这是 spring 对 `CGLIB 3.3` 重新打包.

何谓重新打包呢？个人理解，就是将 `asm7.0` 与 `CGLIB 3.3` 的源码改个包名，复制到 spring 项目下。因此 spring 并没有在 `gradle` 文件中引入 `asm` 与 `cglib` 的相当 jar 包，而是在项目中直接引入了这两个项目的源码！

### 2\. cglib 代理示例

在正式开始之前，我们先来看 看 cglib 代理是如何进行的。

首先准备一个类：

```
package org.springframework.learn.demo04;

public class CglibProxyService {
    public void hello01() {
        System.out.println("hello01");
    }
}

```

再准备一个 `MethodInterceptor`（类比 jdk 动态代理中的 `InvocationHandler`）：

```
package org.springframework.learn.demo04;

import org.springframework.cglib.proxy.MethodInterceptor;
import org.springframework.cglib.proxy.MethodProxy;

import java.lang.reflect.Method;

public class MyMethodInterceptor implements MethodInterceptor {

    /** 目标对象 */
    private Object target;

    public MyMethodInterceptor(Object target){
        this.target = target;
    }

    @Override
    public Object intercept(Object proxyObj, Method method, Object[] objects, 
                MethodProxy proxy) throws Throwable {
        System.out.println("执行方法为:" + method.getName());
        return proxy.invoke(target, objects);
    }
}

```

最后是主类：

```
package org.springframework.learn.demo04;

import org.springframework.cglib.proxy.Enhancer;

/**
 * ｛这里添加描述｝
 *
 * @author fangchengyan
 * @date 2020-11-01 9:23 下午
 */
public class Demo04Main {

    public static void main(String[] args) {
        CglibProxyService target = new CglibProxyService();
        MyMethodInterceptor interceptor = new MyMethodInterceptor(target);

        Enhancer enhancer = new Enhancer();
        // 设置父类
        enhancer.setSuperclass(CglibProxyService.class);
        // 设置callback，这个callback就是上面提供的 MyMethodInterceptor
        enhancer.setCallback(interceptor);
        // 使用 enhancer 创建代理对象
        CglibProxyService proxy = (CglibProxyService)enhancer.create();
        proxy.hello01();
    }
}

```

运行，结果如下：

```
执行方法为:hello01
hello01

```

可以看到，最终是在 `MyMethodInterceptor#intercept` 执行目标对象的方法。

同 jdk 动态代理比较后，发现两者代码高度相似：

*   `InvocationHandler` 与 `InvocationHandler`：两者代码形式几乎一致 ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-6f3e8213e743554a7a8ffc587b98b1d7d3a.png)

*   代理对象的创建：一个是使用 `Enhangcer` 进行代理对象创建，一个是使用封装好的方法进行对象创建。 ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8ee847f9cbb7ec7fa6c35b2ad5a5537ef3a.png)

从代理对象创建来看，可以看出 `cglib` 生成对象时，配置的参数较多，功能较丰富。

关于 `Enhangcer` 是如何创建代理对象，以及 `org.springframework.asm` 与 `org.springframework.cglib` 这两个包下的代码，这些属于 cglib 的内容，就不过多分析了。

### 3\. spring 中 `cglib` 创建代理对象

我们再来看看 spring 是如何创建代理对象的：

> CglibAopProxy#getProxy(java.lang.ClassLoader)

```
public Object getProxy(@Nullable ClassLoader classLoader) {
    try {
        Class<?> rootClass = this.advised.getTargetClass();
        Assert.state(rootClass != null, "...");

        Class<?> proxySuperClass = rootClass;
        if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
            proxySuperClass = rootClass.getSuperclass();
            Class<?>[] additionalInterfaces = rootClass.getInterfaces();
            for (Class<?> additionalInterface : additionalInterfaces) {
                this.advised.addInterface(additionalInterface);
            }
        }

        // 对目标类进行检查，主要检查点有三个：
        // 1\. 目标方法不能使用final修饰；
        // 2\. 目标方法不能是private类型的；
        // 3\. 目标方法不能是包访问权限的；
        // 这三个点满足任何一个，当前方法就不能被代理，此时该方法就会被略过
        validateClassIfNecessary(proxySuperClass, classLoader);

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
        // 这里AopProxyUtils.completeProxiedInterfaces()方法的主要目的是为要生成的代理类
        // 增加SpringProxy，Advised，DecoratingProxy三个需要实现的接口。这里三个接口的作用如下：
        // 1\. SpringProxy：是一个空接口，用于标记当前生成的代理类是Spring生成的代理类；
        // 2\. Advised：Spring生成代理类所使用的属性都保存在该接口中，
        //    包括Advisor，Advice和其他相关属性；
        // 3\. DecoratingProxy：该接口用于获取当前代理对象所代理的目标对象的Class类型。
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

        // 添加callback
        Callback[] callbacks = getCallbacks(rootClass);
        Class<?>[] types = new Class<?>[callbacks.length];
        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        // 设置代理类中各个方法将要使用的切面逻辑，这里ProxyCallbackFilter.accept()方法返回
        // 的整型值正好一一对应上面Callback数组中各个切面逻辑的下标，也就是说这里的CallbackFilter
        // 的作用正好指定了代理类中各个方法将要使用Callback数组中的哪个或哪几个切面逻辑
        enhancer.setCallbackFilter(new ProxyCallbackFilter( this.advised.getConfigurationOnlyCopy(),
                this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);

        // 生成代理对象
        return createProxyClassAndInstance(enhancer, callbacks);
    }
    catch (...) {
        ...
    }
}

```

以上代码核心是设置 `Enhancer` 的属性，如 `classLoader`、`superclass`、`callbackFilter` 等，最后调用 `createProxyClassAndInstance(xxx)` 创建代理对象：

> ObjenesisCglibAopProxy#createProxyClassAndInstance

```
protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
    // 创建代理类
    Class<?> proxyClass = enhancer.createClass();
    Object proxyInstance = null;

    // 根据代理类，使用反射生成对象
    if (objenesis.isWorthTrying()) {
        try {
            proxyInstance = objenesis.newInstance(proxyClass, enhancer.getUseCache());
        }
        catch (Throwable ex) {
            ...
        }
    }

    if (proxyInstance == null) {
        try {
            Constructor<?> ctor = (this.constructorArgs != null ?
                    proxyClass.getDeclaredConstructor(this.constructorArgTypes) :
                    proxyClass.getDeclaredConstructor());
            ReflectionUtils.makeAccessible(ctor);
            proxyInstance = (this.constructorArgs != null ?
                    ctor.newInstance(this.constructorArgs) : ctor.newInstance());
        }
        catch (Throwable ex) {
            throw new AopConfigException(...);
        }
    }
    // 设置callback属性
    // 当设置了多个 callback 时，会通过 CallbackFilter 来确定最终使用哪个 callback
    ((Factory) proxyInstance).setCallbacks(callbacks);
    return proxyInstance;
}

```

通过在第二部分的 `demo04` 可以知道，cglib 方法的执行，会经过 `MethodInterceptor#intercept` 方法调用的，也就是 `Enhancer` 的 `callback` 属性，因此接下来我们来看看 `callback` 的获取，相关代码位于 `CglibAopProxy#getCallbacks`：

> CglibAopProxy#getCallbacks

```
private Callback[] getCallbacks(Class<?> rootClass) throws Exception {
    boolean exposeProxy = this.advised.isExposeProxy();
    boolean isFrozen = this.advised.isFrozen();
    boolean isStatic = this.advised.getTargetSource().isStatic();

    // 用户自定义的代理逻辑的callback，即开发中，定义的 @Before、@Around、@After等切面方法，
    // 将在DynamicAdvisedInterceptor进行调用
    Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);

    Callback targetInterceptor;
    // 判断如果要暴露代理对象，如果是，则使用AopContext设置将代理对象设置到ThreadLocal中
    // 用户则可以通过AopContext获取目标对象
    if (exposeProxy) {
        // 判断被代理的对象是否是静态的，如果是静态的，则将目标对象缓存起来，每次都使用该对象即可，
        // 如果目标对象是动态的，则在DynamicUnadvisedExposedInterceptor中每次都生成一个新的
        // 目标对象，以织入后面的代理逻辑
        targetInterceptor = (isStatic ?
                new StaticUnadvisedExposedInterceptor(this.advised.getTargetSource().getTarget()) :
                new DynamicUnadvisedExposedInterceptor(this.advised.getTargetSource()));
    }
    else {
        // 这里的两个类与上面两个的唯一区别就在于是否使用AopContext暴露生成的代理对象
        targetInterceptor = (isStatic ?
                new StaticUnadvisedInterceptor(this.advised.getTargetSource().getTarget()) :
                new DynamicUnadvisedInterceptor(this.advised.getTargetSource()));
    }

    // 当前Callback用于不用被代理的方法
    Callback targetDispatcher = (isStatic ?
            new StaticDispatcher(this.advised.getTargetSource().getTarget()) : new SerializableNoOp());

    // 将获取到的callback组装为一个数组
    Callback[] mainCallbacks = new Callback[] {
            // 用户自己定义的拦截器
            aopInterceptor,  // for normal advice
            // 根据条件是否暴露代理对象的拦截器
            targetInterceptor,  // invoke target without considering advice, if optimized
            // 不做任何操作的拦截器
            new SerializableNoOp(),  // no override for methods mapped to this
            // 用于存储Advised对象的分发器
            targetDispatcher, this.advisedDispatcher,
            // 针对equals方法调用的拦截器
            new EqualsInterceptor(this.advised),
            // 针对hashcode方法调用的拦截器
            new HashCodeInterceptor(this.advised)
    };

    Callback[] callbacks;

    // 如果目标对象是静态的，并且切面逻辑的调用链是固定的，则对目标对象和整个调用链进行缓存
    if (isStatic && isFrozen) {
        Method[] methods = rootClass.getMethods();
        Callback[] fixedCallbacks = new Callback[methods.length];
        this.fixedInterceptorMap = new HashMap<>(methods.length);

        for (int x = 0; x < methods.length; x++) {
            Method method = methods[x];
            // 获取目标对象的切面逻辑
            List<Object> chain = this.advised
                    .getInterceptorsAndDynamicInterceptionAdvice(method, rootClass);
            fixedCallbacks[x] = new FixedChainStaticTargetInterceptor(
                    chain, this.advised.getTargetSource().getTarget(), this.advised.getTargetClass());
            // 对调用链进行缓存
            this.fixedInterceptorMap.put(method, x);
        }

        // 将生成的静态调用链存入Callback数组中
        callbacks = new Callback[mainCallbacks.length + fixedCallbacks.length];
        System.arraycopy(mainCallbacks, 0, callbacks, 0, mainCallbacks.length);
        System.arraycopy(fixedCallbacks, 0, callbacks, mainCallbacks.length, fixedCallbacks.length);
        // 这里fixedInterceptorOffset记录了当前静态的调用链的切面逻辑的起始位置，
        // 这里记录的用处在于后面使用CallbackFilter的时候，如果发现是静态的调用链，
        // 则直接通过该参数获取相应的调用链，而直接略过了前面的动态调用链
        this.fixedInterceptorOffset = mainCallbacks.length;
    }
    else {
        callbacks = mainCallbacks;
    }
    return callbacks;
}

```

以上代码比较长，主要作用就是获取 `callback`，虽然 spring 提供了众多 `callback`，但与我们自定义通知相关的 callback 只有一个： `DynamicAdvisedInterceptor`，在这个 `callback` 的 `CglibAopProxy.DynamicAdvisedInterceptor#intercept` 方法中，我们在代码中的自定通知就是在这里执行的。

这一步我们得到 cglib 的代理对象后，接下来就来看看切面方法是如何执行的。

### 4\. cglib 切面方法的执行

cglib 切面方法的执行 `CglibAopProxy.DynamicAdvisedInterceptor#intercept` 方法：

> `CglibAopProxy.DynamicAdvisedInterceptor#intercept`

```
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) 
        throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Object target = null;
    // 通过TargetSource获取目标对象
    TargetSource targetSource = this.advised.getTargetSource();
    try {
        // 判断如果需要暴露代理对象，则将当前代理对象设置到ThreadLocal中
        if (this.advised.exposeProxy) {
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);
        // 获取目标对象切面逻辑的调用链
        List<Object> chain = this.advised
                .getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        Object retVal;
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            // 没有拦截器，直接调用目标对象的方法
            retVal = methodProxy.invoke(target, argsToUse);
        }
        else {
            // 这里有两个步骤：
            // 1\. 创建执行器：new CglibMethodInvocation()
            // 2\. 执行拦截器链：CglibMethodInvocation#proceed
            retVal = new CglibMethodInvocation(proxy, target, method, args, 
                    targetClass, chain, methodProxy).proceed();
        }
        // 对返回值进行处理，如果返回值就是当前目标对象，那么将代理生成的代理对象返回；
        // 如果返回值为空，并且返回值类型是非void的基本数据类型，则抛出异常；
        // 如果上述两个条件都不符合，则直接将生成的返回值返回
        retVal = processReturnType(proxy, target, method, retVal);
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

最终执行是在 `new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed()`，继续看下去：

> CglibAopProxy.CglibMethodInvocation#proceed

```
public Object proceed() throws Throwable {
    try {
        return super.proceed();
    }
    catch (...) {
        ....
    }
}

```

直接调用的是父类的方法，`CglibAopProxy.CglibMethodInvocation` 的父类是谁呢？一看下去，发现竟然是 `ReflectiveMethodInvocation`，`super.proceed()` 调用的是 `ReflectiveMethodInvocation#proceed`！

在[上一 篇文章](https://my.oschina.net/funcy/blog/4696654 "上一 篇文章")中，我们就详细分析了 `ReflectiveMethodInvocation#proceed` 的调用过程，而现在，在 cglib 代理里，最终执行的也是同样的代码，这一块的执行过程就不重复分析了。

最后用一张图来说明切面通知的执行过程：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-1c31c8e6279af4c150df18ebbd345c7f110.png)

最终的执行顺序：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-a1f3ccebe3b7335eaef88b8a39ca36b9e05.png)

### 5\. 总结

本文分析了 cglib 代理的执行过程，切面的执行流程位于 `CglibAopProxy.DynamicAdvisedInterceptor#intercept`，最终调用的是 `ReflectiveMethodInvocation#proceed`，这与 jdk 动态代理的执行流程相同。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4696655](https://my.oschina.net/funcy/blog/4696655) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_