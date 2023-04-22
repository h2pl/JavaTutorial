本文是 **springmvc 请求执行流程**的第二篇文章，在上一篇文章中，我们分析了 `DispatcherServlet#doDispatch` 方法，总结出请求执行分为如下步骤：

1.  获取对应的 `HandlerExecutionChain`, 获取的 `HandlerExecutionChain` 中包含真正地处理器（`Controller` 中的方法）和一组 `HandlerInterceptor` 拦截器；
2.  获取对应的 `handlerAdapter`，该对象用来运行 `handler(xxx)` 方法；
3.  执行 spring 的拦截器，运行 `HandlerInterceptor#preHandle` 方法；
4.  处理请求，也就是通过上面获取到的 `handlerAdapter` 来调用 `handle(xxx)` 方法；
5.  执行 spring 的拦截器，运行 `HandlerInterceptor#postHandle` 方法；
6.  处理返回结果，这里会渲染视图，以及执行 spring 拦截器的 `HandlerInterceptor#afterCompletion`。

接着，我们继续分析 `HandlerExecutionChain` 的获取以及 `handlerAdapter` 的获取，书接上回，本文将继续分析接下来的步骤。

### 5\. 执行 spring 的拦截器：`HandlerInterceptor#preHandle`

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ...
    // 3\. 运行spring的拦截器, 运行 HandlerInterceptor#preHandle 方法
    // 这个mappedHandler，就是第1步获取的HandlerExecutionChain
    if (!mappedHandler.applyPreHandle(processedRequest, response)) {
        return;
    }
    ...
}

```

上面的 `mappedHandler`，就是第 1 步获取的 `HandlerExecutionChain`，进入 `HandlerExecutionChain#applyPreHandle`：

> HandlerExecutionChain#applyPreHandle

```
/**
 * 执行 HandlerInterceptor#preHandle 方法
 */
boolean applyPreHandle(HttpServletRequest request, HttpServletResponse response) 
        throws Exception {
    // 获取所有的拦截器
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 遍历执行 preHandle 方法
        for (int i = 0; i < interceptors.length; i++) {
            HandlerInterceptor interceptor = interceptors[i];
            if (!interceptor.preHandle(request, response, this.handler)) {
                // 失败了，还会继续执行 HandlerInterceptor#afterCompletion 方法
                triggerAfterCompletion(request, response, null);
                return false;
            }
            this.interceptorIndex = i;
        }
    }
    return true;
}

/**
 * 执行 HandlerInterceptor#afterCompletion 方法
 * 为了保证HandlerInterceptor#afterCompletion的执行，
 * 接下来的分析会看到，这个方法会在多个方法调用到
 */
void triggerAfterCompletion(HttpServletRequest request, 
        HttpServletResponse response, @Nullable Exception ex) throws Exception {
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 遍历执行 HandlerInterceptor#afterCompletion 方法
        for (int i = this.interceptorIndex; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            try {
                interceptor.afterCompletion(request, response, this.handler, ex);
            }
            catch (Throwable ex2) {
                logger.error("HandlerInterceptor.afterCompletion threw exception", ex2);
            }
        }
    }
}

```

我们来看一眼 `HandlerExecutionChain`：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-40ba62630b5d83a16ed9aba35aba48af8be.png)

再来看看传入 `HandlerInterceptor` 的 `handler` 是啥：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-6c50a936804c1db4a7b817dbb5ff417034d.png)

这个 `handler` 就是 `HandlerMethod`，里面包含的信息还是挺丰富的，像 `bean`/`beanFactory`/`method`，利用这几个属性，我们可以对其进行各种操作。

### 6\. 方法的执行：`AbstractHandlerMethodAdapter#handle`

我们再回到 `DispatcherServlet#doDispatch`，执行完 `HandlerInterceptor#preHandle` 方法后，就来到所有流程中的重头戏：`handler` 的执行，也就是 `controller` 中，`url` 对应的方法执行：

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ...
    // 4\. 通过上面获取到的handlerAdapter来调用handle
    mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
    ...
}

```

一路跟进去，最终来到了 `RequestMappingHandlerAdapter#invokeHandlerMethod` 方法：

```
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    // 包装reques与request对象
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        // 获取 @InitBinder 注解的方法，
        // 包含当前controller与 @ControllerAdvice 标注的类里的 @InitBinder 注解的方法
        WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
        // 获取 @ModelAttribute 注解的方法，
        // 包含当前controller与 @ControllerAdvice 标注的类里的 @ModelAttribute 注解的方法
        ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);
        // 创建方法执行对象
        ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
        if (this.argumentResolvers != null) {
            invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
        }
        if (this.returnValueHandlers != null) {
            invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
        }
        invocableMethod.setDataBinderFactory(binderFactory);
        invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
        // 创建ModelAndView的容器
        ModelAndViewContainer mavContainer = new ModelAndViewContainer();
        mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
        modelFactory.initModel(webRequest, mavContainer, invocableMethod);
        mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);
        // 处理异步请求
        AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
        asyncWebRequest.setTimeout(this.asyncRequestTimeout);
        WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
        asyncManager.setTaskExecutor(this.taskExecutor);
        asyncManager.setAsyncWebRequest(asyncWebRequest);
        asyncManager.registerCallableInterceptors(this.callableInterceptors);
        asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);
        if (asyncManager.hasConcurrentResult()) {
            Object result = asyncManager.getConcurrentResult();
            mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
            asyncManager.clearConcurrentResult();
            LogFormatUtils.traceDebug(logger, traceOn -> {
                String formatted = LogFormatUtils.formatValue(result, !traceOn);
                return "Resume with async result [" + formatted + "]";
            });
            invocableMethod = invocableMethod.wrapConcurrentResult(result);
        }
        // 执行Controller的方法（重点）
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }
        // 处理返回结果
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}

```

这个方法有点长，但重点方法只有一行：`invocableMethod.invokeAndHandle(webRequest, mavContainer);`，在这个方法前的部分都是在做方法执行前的准备工作，如获取 `@InitBinder` 注解的方法、获取 `@ModelAttribute` 注解的方法，准备方法参数 `webRequest`(包装 `request` 与 `response` 对象) 与 `mavContainer`(`ModelAndView` 包装对象) 等。这里我们直接进入 `invocableMethod.invokeAndHandle`，看看方法是如何执行的：

> ServletInvocableHandlerMethod#invokeAndHandle

```
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {
    // 执行handler方法
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    // 设置 RequestHandled 属性值，springmvc会根据该值判断要不要跳转
    if (returnValue == null) {
        if (isRequestNotModified(webRequest) || getResponseStatus() != null 
                || mavContainer.isRequestHandled()) {
            disableContentCachingIfNecessary(webRequest);
            mavContainer.setRequestHandled(true);
            return;
        }
    }
    else if (StringUtils.hasText(getResponseStatusReason())) {
        mavContainer.setRequestHandled(true);
        return;
    }
    mavContainer.setRequestHandled(false);
    Assert.state(this.returnValueHandlers != null, "No return value handlers");
    try {
        // 处理返回结果
        this.returnValueHandlers.handleReturnValue(
                returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    }
    catch (Exception ex) {
        throw ex;
    }
}

```

这里方法先是调用了 `invokeForRequest` 执行方法，然后再根据方法的返回值来设置 `mavContainer` 的 `RequestHandled` 值，最后处理返回结果。继续跟进 `invokeForRequest` 方法：

> InvocableHandlerMethod#invokeForRequest

```
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainermavContainer,
        Object... providedArgs) throws Exception {
    // 处理参数绑定
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    // 反射调用方法
    return doInvoke(args);
}

```

这个方法里先是处理了参数解析，然后使用反射进行方法调用。事实上，整个执行流程中，最核心的就是参数解析了，在 controller 里的 handler 方法中，我们可以这样指定参数：

```
// 直接传参
@RequestMapping("xxx")
public Object test(String name) {
    ...
}

// 在参数上标有@RequestParam、@RequestHeader等注解
@RequestMapping("xxx")
public Object test(@RequestParam("name") String name, 
                   @RequestHeader("uid") String uid) {
    ...
}

// 将传入的参数封装为对象
@RequestMapping("xxx")
public Object test(User user) {
    ...
}

// 上面的方法都是使用form表单传参(也就是k1=v1&2=v2&...的方法)
// 这个方法使用 RequestBody 方式传参，将参数内容放入消息体中
@RequestMapping("xxx")
public Object test(@RequestBody User user) {
    ...
}

...

```

当我们按规范传入参数时，springmvc 都能正常处理，我们来看看 springmvc 是如何做到这一点的。

#### 参数解析

继续，进入 `InvocableHandlerMethod#getMethodArgumentValues` 方法：

> InvocableHandlerMethod#getMethodArgumentValues

```
protected Object[] getMethodArgumentValues(NativeWebRequest request, 
        @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
    // 获取方法的所有参数，可以简单理解为利用反射获取handler方法参数，然后包装为 MethodParameter
    MethodParameter[] parameters = getMethodParameters();
    if (ObjectUtils.isEmpty(parameters)) {
        return EMPTY_ARGS;
    }
    Object[] args = new Object[parameters.length];
    // 依次处理每个参数
    for (int i = 0; i < parameters.length; i++) {
        MethodParameter parameter = parameters[i];
        parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
        args[i] = findProvidedArgument(parameter, providedArgs);
        if (args[i] != null) {
            continue;
        }
        // 判断是否有参数解析类支持当前参数的解析
        if (!this.resolvers.supportsParameter(parameter)) {
            throw new IllegalStateException(formatArgumentError(parameter, "No suitable resolver"));
        }
        try {
            // 处理参数解析
            args[i] = this.resolvers.resolveArgument(parameter, mavContainer, request, 
                    this.dataBinderFactory);
        }
        catch (Exception ex) {
            throw ex;
        }
    }
    return args;
}

```

这个方法先获取了 handler 方法的参数 (可以简单理解为利用反射获取 handler 方法参数，然后包装为 `MethodParameter`)，然后对这些参数逐个解析。解析时，会用到两个重要的方法：`resolvers.supportsParameter(...)` 与 `resolvers.resolveArgument(...)`，这两个方法最终调用的方法在 `HandlerMethodArgumentResolverComposite` 中：

> HandlerMethodArgumentResolverComposite

```
@Nullable
public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainermavContainer,
        NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
    // 获取一个解析器
    HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
    if (resolver == null) {
        throw new IllegalArgumentException(...);
    }
    // 解析参数，调用 HandlerMethodArgumentResolver#resolveArgument 方法
    return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
}

private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
    HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
    if (result == null) {
        // 遍历所有的解析器
        for (HandlerMethodArgumentResolver resolver : this.argumentResolvers) {
            // 找到其中一个解析器，返回
            // 调用 HandlerMethodArgumentResolver#supportsParameter 方法
            if (resolver.supportsParameter(parameter)) {
                result = resolver;
                this.argumentResolverCache.put(parameter, result);
                break;
            }
        }
    }
    return result;
}

```

这两个方法主要是遍历解析器，然后调用 `HandlerMethodArgumentResolver#supportsParameter` 与 `HandlerMethodArgumentResolver#resolveArgument` 处理具体的操作。`HandlerMethodArgumentResolver` 是个接口，里面仅有两个方法：

```
public interface HandlerMethodArgumentResolver {
    /**
     * 当前是否支持处理当前参数
     */
    boolean supportsParameter(MethodParameter parameter);

    /**
     * 具体的解析操作
     */
    @Nullable
    Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
            NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
}

```

这个就是真正的参数解析器了。在 springmvc 中，提供了多少种参数解析器呢？经过本人的调试，发现多达 26 个：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-fd30bc847fc2cf7082de3731e68df6ae5f9.png) ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-894b391bbe4567734ee7237d900bc55ee31.png)

正是在这些参数解析器的帮助下，springmvc 能支持多种参数接收方式！对于参数解析器，由于设计比较复杂，涉及多种传参方式，本文并不想展开讨论，感兴趣的小伙伴可自行查阅相关文档。

#### 执行 handler 方法

处理完参数解析后，就开始执行 handler 方法了。让我们回到 `InvocableHandlerMethod#invokeForRequest` 方法：

> InvocableHandlerMethod#invokeForRequest

```
@Nullable
public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainermavContainer,
        Object... providedArgs) throws Exception {
    // 处理参数绑定
    Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
    // 反射调用方法
    return doInvoke(args);
}

```

进入 `doInvoke` 方法：

```
@Nullable
protected Object doInvoke(Object... args) throws Exception {
    // 使用反射执行方法
    ReflectionUtils.makeAccessible(getBridgedMethod());
    try {
        // 这里就是反射操作
        return getBridgedMethod().invoke(getBean(), args);
    }
    catch (...) {
        ...
    }
}

```

这个方法很简单，就是利用反射来进行方法执行的，就不过多分析了。

#### 处理返回参数

让我们回到 `ServletInvocableHandlerMethod#invokeAndHandle` 方法：

> ServletInvocableHandlerMethod#invokeAndHandle

```
public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
        Object... providedArgs) throws Exception {
    // 执行handle方法
    Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
    ...
    try {
        // 处理返回结果
        this.returnValueHandlers.handleReturnValue(
                returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
    }
    catch (Exception ex) {
        throw ex;
    }
}

```

处理完方法执行后，接着就处理返回结果：

> HandlerMethodReturnValueHandlerComposite#handleReturnValue

```
@Override
public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
        ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
    // 根据返回类型找一个合适的handler来处理返回数据
    HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
    if (handler == null) {
        throw new IllegalArgumentException(...);
    }
    handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
}

@Nullable
private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameterreturnType) {
    boolean isAsyncValue = isAsyncReturnValue(value, returnType);
    // 遍历，逐个判断是否能处理当前返回类型
    for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
        if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
            continue;
        }
        if (handler.supportsReturnType(returnType)) {
            return handler;
        }
    }
    return null;
}

```

获取 `ReturnValueHandler` 的套路与前面获取 `ArgumentResolver` 的套路相关无几，`ReturnValueHandler` 是 `HandlerMethodReturnValueHandler` 的子类，`HandlerMethodReturnValueHandler` 代码如下：

```
public interface HandlerMethodReturnValueHandler {

    /**
     * 判断当前ReturnValueHandler能否处理传入的returnType
     */
    boolean supportsReturnType(MethodParameter returnType);

    /**
     * 具体的处理逻辑
     */
    void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
            ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;

}

```

同前面参数解析器一样，这个接口里也有两个方法：

*   `boolean supportsReturnType(xxx)`：判断当前 `ReturnValueHandler` 能否处理传入的 `returnType`
*   `void handleReturnValue(xxx)`：具体的处理逻辑

同样地，springmvc 也提供了非常多的实现来处理返回参数：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8e3c1fd4dca7c13efbceac5800e26145963.png)

对于参数的返回，我们来看一个简单的示例：

如果我们像这样返回值时：

```
@Controller
@RequestMapping("/xxx")
public class XxxController {

    @RequestMapping("/index")
    public String index() {
        return "index";
    }
}

```

返回到页面的将是一个视图，对应的 `HandlerMethodReturnValueHandler` 为 `ViewNameMethodReturnValueHandler`：

```
public class ViewNameMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

    @Nullable
    private String[] redirectPatterns;

    // 省略 redirectPatterns 的setter与getter方法
    ...

    @Override
    public boolean supportsReturnType(MethodParameter returnType) {
        Class<?> paramType = returnType.getParameterType();
        // 支持处理的返回值类型：返回值类型为void，或者为字符串类型
        return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
    }

    /**
     * 具体的处理逻辑
     */
    @Override
    public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
            ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

        if (returnValue instanceof CharSequence) {
            // viewName 就是返回值
            String viewName = returnValue.toString();
            mavContainer.setViewName(viewName);
            if (isRedirectViewName(viewName)) {
                // 是否需要跳转
                mavContainer.setRedirectModelScenario(true);
            }
        }
        else if (returnValue != null) {
            throw new UnsupportedOperationException(...);
        }
    }

    /**
     * 判断是否需要跳转
     */
    protected boolean isRedirectViewName(String viewName) {
        // this.redirectPatterns 默认为null，可以自行调用 setter 方法设置
        return (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) 
            // 匹配 redirect: 开头，也就是说，如果我们返回 "redirect:index"，就表明该结果需要跳转
            || viewName.startsWith("redirect:"));
    }

}

```

这个类比较简单，关键部分已在代码中作了注释，这里就不多说了。值得一提的是，在 `handleReturnValue(xxx)` 方法中，springmvc 将返回的字符串设置为 `viewName` 需要注意下，后面在处理视图时，会利用这个 `viewName` 拿到对应的 `View`。

#### 获取 ModelAndView

让我们回到 `RequestMappingHandlerAdapter#invokeHandlerMethod` 方法：

```
@Nullable
protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
        HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
    ServletWebRequest webRequest = new ServletWebRequest(request, response);
    try {
        ...
        // 执行Controller的方法
        invocableMethod.invokeAndHandle(webRequest, mavContainer);
        if (asyncManager.isConcurrentHandlingStarted()) {
            return null;
        }
        // 处理执行结果，从中拿到 ModelAndView
        return getModelAndView(mavContainer, modelFactory, webRequest);
    }
    finally {
        webRequest.requestCompleted();
    }
}

```

执行完 `invocableMethod.invokeAndHandle(webRequest, mavContainer)` 方法后接着就是从执行结果中拿到 `ModelAndView` 了，进入 `RequestMappingHandlerAdapter#getModelAndView` 方法：

```
private ModelAndView getModelAndView(ModelAndViewContainer mavContainer,
        ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
    // 这个 ModelFactory，就是前面获取的 包含@ModelAttribute 注解的方法
    modelFactory.updateModel(webRequest, mavContainer);
    if (mavContainer.isRequestHandled()) {
        return null;
    }
    ModelMap model = mavContainer.getModel();
    // 创建视图对象，把mavContainer.getViewName()传入到ModelAndView的构造方法中
    ModelAndView mav = new ModelAndView(mavContainer.getViewName(), 
            model, mavContainer.getStatus());
    if (!mavContainer.isViewReference()) {
        mav.setView((View) mavContainer.getView());
    }
    // 处理重定向参数
    if (model instanceof RedirectAttributes) {
        Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
        HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
        if (request != null) {
            RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
        }
    }
    return mav;
}

```

可以看到，`ModelAndView` 就是从前面执行结果的 `viewName` 得到的。

至此，`AbstractHandlerMethodAdapter#handle` 执行完毕。

### 7\. 执行拦截器：`HandlerInterceptor#postHandle`

让我们再回到 `DispatcherServlet#doDispatch` 方法：

> DispatcherServlet#doDispatch

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    ...

    try {
        ...
        try {
            ...
            // 4.通过上面获取到的handlerAdapter来调用handle
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            // 5.如果函数调用没有返回视图则使用默认的
            applyDefaultViewName(processedRequest, mv);
            // 6\. 执行拦截器，运行 HandlerInterceptor#postHandle 方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (...) {
            ...
        }
        // 7\. 处理返回结果，在这个方法里会执行 HandlerInterceptor.afterCompletion
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (...) {
        ...
    }
}

```

执行完 `handler` 方法后，就开始执行拦截器的 `HandlerInterceptor#postHandle` 方法了：

> HandlerExecutionChain#applyPostHandle

```
void applyPostHandle(HttpServletRequest request, HttpServletResponse response, 
        @NullableModelAndView mv) throws Exception {
    HandlerInterceptor[] interceptors = getInterceptors();
    if (!ObjectUtils.isEmpty(interceptors)) {
        // 遍历执行 postHandle(...) 方法
        for (int i = interceptors.length - 1; i >= 0; i--) {
            HandlerInterceptor interceptor = interceptors[i];
            interceptor.postHandle(request, response, this.handler, mv);
        }
    }
}

```

这个同前面执行拦截器方法类似，这里就不过多分析了。需要强调的是，`HandlerInterceptor#postHandle` 的执行时机是**在执行完 `handler` 方法后之后，在视图解析之前**。因此，我们可以在拦截器的这个方法里处理一些额外的渲染参数。

限于篇幅，本文就先到这里了，request 执行的后续流程，我们下篇文章再分析。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4741104](https://my.oschina.net/funcy/blog/4741104) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_