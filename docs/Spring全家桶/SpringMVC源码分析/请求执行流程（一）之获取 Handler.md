上一篇文章我们分析 `RequestMapping` 初始化流程，本文将分析 spring mvc 的请求执行流程。

### 1\. 请求的执行入口

在 [spring mvc 之 DispatcherServlet 初始化流程](https://my.oschina.net/funcy/blog/4710330 "spring mvc之DispatcherServlet 初始化流程")一文中，我们深入分析了向 `servlet` 容器添加了 `DispatcherServlet` 后，引发的一系列初始化流程，本文将继续围绕这个 `servlet` 分析 `springmvc` 的请求流程。

#### 1.1 回顾 servlet 的执行入口：

在分析 `DispatcherServlet` 前首先要回顾下 servlet 的执行入口。

在我们实现自定义的 servlet 时，一般是实现 `HttpServlet`，然后重写 `doGet(xxx)`、`doPost()` 方法，而实际上 servlet 为 `HttpServlet#service(ServletRequest, ServletResponse)`：

```
public abstract class HttpServlet extends GenericServlet {
    ...

    // 这个方法仅做了参数类型转换
    @Override
    public void service(ServletRequest req, ServletResponse res)
        throws ServletException, IOException {

        HttpServletRequest  request;
        HttpServletResponse response;

        // 在这里处理参数的类型转换
        try {
            request = (HttpServletRequest) req;
            response = (HttpServletResponse) res;
        } catch (ClassCastException e) {
            throw new ServletException(lStrings.getString("http.non_http"));
        }
        service(request, response);
    }

    /**
     * 在这里处理请求
     * 从代码可以看到，这个类其实是做一个转发：判断请求方法，然后调用具体的方法执行
     */
    protected void service(HttpServletRequest req, HttpServletResponse resp)
        throws ServletException, IOException {

        String method = req.getMethod();

        // 判断的请求方法，然后找到对应的方法去执行
        if (method.equals(METHOD_GET)) {
            long lastModified = getLastModified(req);
            if (lastModified == -1) {
                doGet(req, resp);
            } else {
                ...
                doGet(req, resp);
            }
        } else if (method.equals(METHOD_HEAD)) {
            long lastModified = getLastModified(req);
            maybeSetLastModified(resp, lastModified);
            doHead(req, resp);
        } else if (method.equals(METHOD_POST)) {
            doPost(req, resp);
        } else if (method.equals(METHOD_PUT)) {
            doPut(req, resp);
        } else if (method.equals(METHOD_DELETE)) {
            doDelete(req, resp);
        } else if (method.equals(METHOD_OPTIONS)) {
            doOptions(req,resp);
        } else if (method.equals(METHOD_TRACE)) {
            doTrace(req,resp);
        } else {
            // 没有对应的方法，报错
            ...
        }
    }

}

```

以上是 servlet 源码，方法比较简单，重点部分都做了注释，这里有两个需要再次强调下：

1.  servlet 的执行入口为 `HttpServlet#service(ServletRequest, ServletResponse)`；
2.  `HttpServlet#service(HttpServletRequest, HttpServletResponse)` 方法会根据请求方法找到对应的处理方法执行，一般来说，我们自定义 servlet，只要重写 `doGet(xxx)`、`doPost(xxx)` 等方法即可。

请求流程大概如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-60d0e2afcbdb75a175c6f28471fc467f0e5.png)

#### 1.2 `DispatcherServlet` 的父类：`FrameworkServlet`

了解完 servlet 的请求入口后，接下来就得分析一个不得不提的类了：`FrameworkServlet`。`FrameworkServlet` 是 `HttpServlet` 的子类，实现了 `HttpServlet` 的各种 `doXxx()`，同时也实现了 `service(HttpServletRequest, HttpServletResponse)`：

```
/**
 *  FrameworkServlet继承了HttpServletBean，而HttpServletBean继承了HttpServlet
 *  因此FrameworkServlet也是HttpServlet的子类
 */
public abstract class FrameworkServlet extends HttpServletBean implements ApplicationContextAware {
    @Override
    protected final void doGet(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }

    @Override
    protected final void doPost(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }

    @Override
    protected final void doPut(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }

    @Override
    protected final void doDelete(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        processRequest(request, response);
    }

    @Override
    protected void service(HttpServletRequest request, HttpServletResponse response)
            throws ServletException, IOException {
        HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
        if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
            processRequest(request, response);
        }
        else {
            // GET/POST/PUT/DELETE 等请求方法，还是会调用父类的方法
            super.service(request, response);
        }
    }
}

```

可以看到，以上代码中，有一个方法的出镜率相当高：`FrameworkServlet#processRequest`，不管是 `doXxx(xxx)`，还是 `service(xxx)`，都会调用 `processRequest(xxx)`，接下来我们就来看看这个方法做了什么：

> FrameworkServlet#processRequest

```
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
        throws ServletException, IOException {
    // 记录开始时间
    long startTime = System.currentTimeMillis();
    Throwable failureCause = null;
    // 记录当前线程的信息
    LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
    LocaleContext localeContext = buildLocaleContext(request);
    RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
    ServletRequestAttributes requestAttributes = buildRequestAttributes(
            request, response, previousAttributes);
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), 
             new RequestBindingInterceptor());
    initContextHolders(request, localeContext, requestAttributes);
    try {
        // 核心处理
        doService(request, response);
    }
    catch (ServletException | IOException ex) {
        failureCause = ex;
        throw ex;
    }
    catch (Throwable ex) {
        failureCause = ex;
        throw new NestedServletException("Request processing failed", ex);
    }
    finally {
        // 清除线程绑定信息
        resetContextHolders(request, previousLocaleContext, previousAttributes);
        if (requestAttributes != null) {
            requestAttributes.requestCompleted();
        }
        logResult(request, response, failureCause, asyncManager);
        // 发送事件通知
        publishRequestHandledEvent(request, response, startTime, failureCause);
    }
}

```

这个方法虽然有点长，但大部分与请求处理流程关系不大，与请求处理流程相关的只有几行：

```
    ...
    try {
        // 核心处理
        doService(request, response);
    }
    catch (ServletException | IOException ex) {
        failureCause = ex;
        throw ex;
    }
    ...

```

由此可以看到，实际处理请求的方法是在 `FrameworkServlet#doService` 中。不过，`FrameworkServlet#doService` 是个抽象方法：

```
protected abstract void doService(HttpServletRequest request, 
        HttpServletResponse response) throws Exception;

```

真正的实现是在子类，也就是 `DispatcherServlet#doService` 中。

#### 1.3 `DispatcherServlet#doService`

来看看 `DispatcherServlet#doService` 做了啥事：

```
public class DispatcherServlet extends FrameworkServlet {
    @Override
    protected void doService(HttpServletRequest request, 
            HttpServletResponse response) throws Exception {
        logRequest(request);
        // 省略了一大段属性设置
        ...
        try {
            // 具体的处理
            doDispatch(request, response);
        }
        finally {
            ...
        }
    }
}

```

这人方法也没干什么事实，只是调用了一下 `doDispatch` 方法，然后就没了。事实上，`DispatcherServlet#doDispatch` 就是最终处理请求的逻辑，接下来我们重点分析这个方法 。

这一节我们来总结下 `DispatcherServlet` 的请求流程：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-f854710c25924666cd5ab755c2983a9baf7.png)

### 2\. springmvc 请求分发：`DispatcherServlet#doDispatch`

上一节的最后，我们发现 springmvc 处理请求的方法是 `DispatcherServlet#doDispatch`，本节就从这个方法入手，看看这个方法的逻辑：

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) 
            throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    try {
        ModelAndView mv = null;
        Exception dispatchException = null;
        try {
            //如果是文件上传请求则进行特殊处理
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);
            // 1\. 获取对应的handler, 
            // Handler中包含真正地处理器（Controller中的方法）和一组HandlerInterceptor拦截器
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                // 如果没找到，报个404
                noHandlerFound(processedRequest, response);
                return;
            }
            // 2\. 获取对应的handlerAdapter，用来运行 handler(xxx)
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
            // 处理last-modified情况
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
            // 3\. 运行spring的拦截器, 运行 HandlerInterceptor#preHandle 方法
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }
            // 4\. 通过上面获取到的handlerAdapter来调用handle
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
            // 如果函数调用没有返回视图则使用默认的
            applyDefaultViewName(processedRequest, mv);
            // 5\. 执行拦截器，运行 HandlerInterceptor#postHandle 方法
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // 6\. 处理返回结果，在这个方法里会渲染视图，以及执行 HandlerInterceptor#afterCompletion
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (...) {
        // 这里会执行 HandlerInterceptor#afterCompletion
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            if (mappedHandler != null) {
                // 回调拦截器，执行方法 AsyncHandlerInterceptor#afterConcurrentHandlingStarted
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}

```

这个方法在点长，不过流程很清晰，springmvc 的整个请求流程都在这里了，这里把关键步骤展示如下：

1.  获取对应的 `HandlerExecutionChain`, 获取的 `HandlerExecutionChain` 中包含真正地处理器（`Controller` 中的方法）和一组 `HandlerInterceptor` 拦截器；
2.  获取对应的 `handlerAdapter`，该对象用来运行 `handler(xxx)` 方法；
3.  执行 spring 的拦截器，运行 `HandlerInterceptor#preHandle` 方法；
4.  处理请求，也就是通过上面获取到的 `handlerAdapter` 来调用 `handle(xxx)` 方法；
5.  执行 spring 的拦截器，运行 `HandlerInterceptor#postHandle` 方法；
6.  处理返回结果，这里会渲染视图，以及执行 spring 拦截器的 `HandlerInterceptor#afterCompletion`。

总的流程梳理清楚了，接下来就是逐个流程分析了。

### 3\. 获取 `HandlerExecutionChain`

获取 `HandlerExecutionChain` 的方法在 `DispatcherServlet#getHandler` 中：

```
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        // 遍历所有的handlerMapping，
        // 这里的 handlerMapping 是在WebMvcConfigurationSupport中引入的
        for (HandlerMapping mapping : this.handlerMappings) {
            // 这里调用具体的handler，哪个handler能够处理就直接返回
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}

```

这里的 `handlerMappings` 是在 `WebMvcConfigurationSupport` 中引入的，关于这一块的分析，可能参考 [springmvc demo 与 @EnableWebMvc 注解](https://my.oschina.net/funcy/blog/4678093 "springmvc demo 与 @EnableWebMvc 注解")一文，这里来看看这个 `handlerMappings` 有些啥：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-4465a638ca6c52f24f8c9343986be0bbee9.png)

对于 `RequestMappingHandlerMapping` 相信大家已经很熟悉，对于 `@Controller`/`@RequestMapping` 方式实现的 `controller`，对应的 `HandlerMapping` 就是 `RequestMappingHandlerMapping`。至于另外的两个 `HandlerMapping`，则分别对应不同方式实现的 `controller`，关于这一点，感兴趣的小伙伴可以自行百度，这里就不展开了。

我们继续看 `AbstractHandlerMapping#getHandler` 方法：

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 1\. 调用具体的实现去获取handler
    Object handler = getHandlerInternal(request);
    // 如果为空使用默认的
    if (handler == null) {
        handler = getDefaultHandler();
    }
    // 没有默认的返回空
    if (handler == null) {
        return null;
    }
    // 尝试通过BeanName去获取handler
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }
    // 2\. 获取 executionChain，其实就是找到 uri 对应的 Interceptors,
    // 然后与上面找到的handler一起封装到HandlerExecutionChain对象中
    // 这里的Interceptors，也是在WebMvcConfigurationSupport中配置的
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    // 3\. 处理路域相关的配置：CorsHandlerExecutionChain
    // 这里可以看到，所谓的cors跨域配置，也是由拦截器实现的
    if (hasCorsConfigurationSource(handler)) {
        CorsConfiguration config = (this.corsConfigurationSource != null 
                ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        config = (config != null ? config.combine(handlerConfig) : handlerConfig);
        // 将跨域相关的配置添加到 Interceptors，加到拦截器List的第一个中
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }
    return executionChain;
}

```

这个方法主要做了三件事：

1.  调用具体的实现去获取 handler，这个方法是重点，下面会继续讲；
2.  获取 `executionChain`，这个 `executionChain` 除了包含了上一步的 `handler` 外，还包含 `uri` 对应的 `Interceptors`，获取方法为获取所有的 `Interceptors` 配置（在 `WebMvcConfigurationSupport` 中配置的），再逐一判断 uri 是否符合 `Interceptor` 的 uri 配置；
3.  获取 cors 跨域配置，然后添加到 `executionChain` 中的 `Interceptors` 列表的第一位。嗯，没错，cors 跨域配置也是在 `WebMvcConfigurationSupport` 中配置的。

#### 3.1 查找 `HandlerMethod`

我们进入 `getHandlerInternal(xxx)` 方法：

> AbstractHandlerMethodMapping#getHandlerInternal

```
@Override
protected HandlerMethod getHandlerInternal(HttpServletRequest request) throws Exception {
    // 获取请求的url
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    request.setAttribute(LOOKUP_PATH, lookupPath);
    this.mappingRegistry.acquireReadLock();
    try {
        // 在这里查找uri对应的handlerMethod
        HandlerMethod handlerMethod = lookupHandlerMethod(lookupPath, request);
        // 如果handlerMethod不为空，则重新创建一个HandlerMethod返回
        return (handlerMethod != null ? handlerMethod.createWithResolvedBean() : null);
    }
    finally {
        this.mappingRegistry.releaseReadLock();
    }
}

```

这里还是调用 `lookupHandlerMethod(xxx)` 来查找 `handlerMethod`，继续

> AbstractHandlerMethodMapping#lookupHandlerMethod

```
protected HandlerMethod lookupHandlerMethod(String lookupPath, HttpServletRequest request) 
        throws Exception {
    List<Match> matches = new ArrayList<>();
    // 先从urlLookup中找，urlLookup是一个map，key是url，value是LinkedList<RequestMappingInfo>
    List<T> directPathMatches = this.mappingRegistry.getMappingsByUrl(lookupPath);
    if (directPathMatches != null) {
        // 由于返回的是一个 list，这里会把所有的匹配的结果放入一个matches中
        addMatchingMappings(directPathMatches, matches, request);
    }
    if (matches.isEmpty()) {
        // 如果通过url没找到，则遍历所有的 mappings 匹配，匹配类似于 /test/{name} 的url
        // mappings也是一个map，key是RequestMappingInfo， value是HandlerMethod
        addMatchingMappings(this.mappingRegistry.getMappings().keySet(), matches, request);
    }
    // 找到最佳匹配的mapping,返回其对应的HandlerMethod
    // 比较规则来自于 RequestMappingInfo#compareTo方法
    if (!matches.isEmpty()) {
        Comparator<Match> comparator = new MatchComparator(getMappingComparator(request));
        matches.sort(comparator);
        Match bestMatch = matches.get(0);
        if (matches.size() > 1) {
            if (CorsUtils.isPreFlightRequest(request)) {
                return PREFLIGHT_AMBIGUOUS_MATCH;
            }
            Match secondBestMatch = matches.get(1);
            // 找到了两个最佳匹配，抛出异常
            if (comparator.compare(bestMatch, secondBestMatch) == 0) {
                Method m1 = bestMatch.handlerMethod.getMethod();
                Method m2 = secondBestMatch.     .,m.bvc .getMethod();
                String uri = request.getRequestURI();
                throw new IllegalStateException(...);
            }
        }W
        request.setAttribute(BEST_MATCHING_HANDLER_ATTRIBUTE, bestMatch.handlerMethod);
        handleMatch(bestMatch.mapping, lookupPath, request);
        return bestMatch.handlerMethod;
    }
    else {
        return handleNoMatch(this.mappingRegistry.getMappings().keySet(), lookupPath, request);
    }
}

```

这个方法就是处理 handler 的获取了。这里的获取分为几个步骤：

1.  先从 `urlLookup` 中找，`urlLookup` 是一个 `map`，`key` 是 `url`，`value` 是 `LinkedList<RequestMappingInfo>`，这个操作就是 `map.get(xxx)` 方法；
2.  如果通过 `url` 没找到，则遍历所有的 `mappings` 匹配，匹配类似于 `/test/{name}` 的 `url`，`mappings` 也是一个 `map`，`key` 是 `RequestMappingInfo`， `value` 是 `HandlerMethod`；
3.  如果找到了多个 `HandlerMethod`，则根据 `RequestMappingInfo#compareTo` 方法提供的方法，找到最佳的 `RequestMappingInfo` 对应的 `HandlerMethod`。

我们来看看在 `mappings` 里是如何找到匹配的 `RequestMappingInfo` 的：

> AbstractHandlerMethodMapping#addMatchingMappings

```
private void addMatchingMappings(Collection<T> mappings, List<Match> matches, 
            HttpServletRequest request) {
    for (T mapping : mappings) {
        // 匹配其他条件，找到其中所有符合条件的 mappings
        T match = getMatchingMapping(mapping, request);
        if (match != null) {
            matches.add(new Match(match, this.mappingRegistry.getMappings().get(mapping)));
        }
    }
}

```

最终发现匹配的处理是在 `RequestMappingInfo#getMatchingCondition` 方法中，`RequestMappingInfo` 还有一个 `compareTo` 方法，我们也一并查看下：

> RequestMappingInfo

```
/**
 * 匹配规则
 * 会分别匹配 请求方法(get,post等)、请求参数、请求头等
 */
public RequestMappingInfo getMatchingCondition(HttpServletRequest request) {
    RequestMethodsRequestCondition methods = this.methodsCondition.getMatchingCondition(request);
    if (methods == null) {
        return null;
    }
    ParamsRequestCondition params = this.paramsCondition.getMatchingCondition(request);
    if (params == null) {
        return null;
    }
    HeadersRequestCondition headers = this.headersCondition.getMatchingCondition(request);
    if (headers == null) {
        return null;
    }
    ConsumesRequestCondition consumes = this.consumesCondition.getMatchingCondition(request);
    if (consumes == null) {
        return null;
    }
    ProducesRequestCondition produces = this.producesCondition.getMatchingCondition(request);
    if (produces == null) {
        return null;
    }
    PatternsRequestCondition patterns = this.patternsCondition.getMatchingCondition(request);
    if (patterns == null) {
        return null;
    }
    RequestConditionHolder custom = this.customConditionHolder.getMatchingCondition(request);
    if (custom == null) {
        return null;
    }
    return new RequestMappingInfo(this.name, patterns,
            methods, params, headers, consumes, produces, custom.getCondition());
}

/**
 * 比较规则，找到最佳匹配
 * 会分别比较 请求方法(get,post等)、请求参数、请求头等
 */
public int compareTo(RequestMappingInfo other, HttpServletRequest request) {
    int result;
    if (HttpMethod.HEAD.matches(request.getMethod())) {
        result = this.methodsCondition.compareTo(other.getMethodsCondition(), request);
        if (result != 0) {
            return result;
        }
    }
    result = this.patternsCondition.compareTo(other.getPatternsCondition(), request);
    if (result != 0) {
        return result;
    }
    result = this.paramsCondition.compareTo(other.getParamsCondition(), request);
    if (result != 0) {
        return result;
    }
    result = this.headersCondition.compareTo(other.getHeadersCondition(), request);
    if (result != 0) {
        return result;
    }
    result = this.consumesCondition.compareTo(other.getConsumesCondition(), request);
    if (result != 0) {
        return result;
    }
    result = this.producesCondition.compareTo(other.getProducesCondition(), request);
    if (result != 0) {
        return result;
    }
    result = this.methodsCondition.compareTo(other.getMethodsCondition(), request);
    if (result != 0) {
        return result;
    }
    result = this.customConditionHolder.compareTo(other.customConditionHolder, request);
    if (result != 0) {
        return result;
    }
    return 0;
}

```

无论匹配，还是比较，都会对请求方法 (get,post 等)、请求参数、请求头等一一进行处理。

到这里，我们就明白了 springmvc 是如何找到 `HandlerMethod` 的了。

#### 3.2 查找 `Interceptors`

我们回到 `AbstractHandlerMapping#getHandler`，看看是如何获取 `Interceptor` 的：

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    ...
    // 2\. 获取 executionChain，其实就是找到 uri 对应的 Interceptors,
    // 然后与上面找到的handler一起封装到HandlerExecutionChain对象中
    // 这里的Interceptors，也是在WebMvcConfigurationSupport中配置的
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    ...
    return executionChain;
}

```

进入 `getHandlerExecutionChain` 方法：

> AbstractHandlerMapping#getHandlerExecutionChain

```
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, 
            HttpServletRequest request) {
    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?
            (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));
    // 获取当前的请求路径
    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);
    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {
        if (interceptor instanceof MappedInterceptor) {
            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;
            // 判断当前请求路径是否满足interceptor里配置的路径
            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {
                chain.addInterceptor(mappedInterceptor.getInterceptor());
            }
        }
        else {
            chain.addInterceptor(interceptor);
        }
    }
    return chain;
}

```

这个方法比较简单，相关内容已经在代码中做了注释，就不多说了 。

#### 3.3 处理 cors 跨域配置

我们再来看看跨域配置的处理：

```
public final HandlerExecutionChain getHandler(HttpServletRequest request) 
        throws Exception {
    ...
    // 3\. 处理路域相关的配置：CorsHandlerExecutionChain
    // 这里可以看到，所谓的cors跨域配置，也是由拦截器实现的
    if (hasCorsConfigurationSource(handler)) {
        // 获取跨域配置
        CorsConfiguration config = (this.corsConfigurationSource != null 
                ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        config = (config != null ? config.combine(handlerConfig) : handlerConfig);
        // 将跨域相关的配置添加到 Interceptors，加到拦截器List的第一个中
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }
    return executionChain;
}

```

跨域相关配置也可以 `WebMvcConfigurationSupport` 中配置：

```
protected void addCorsMappings(CorsRegistry registry) {
    ...
}

```

springmvc 获取到跨域配置后，会把相关配置添加到 `HandlerExecutionChain` 中：

```
# AbstractHandlerMapping#getCorsHandlerExecutionChain
protected HandlerExecutionChain getCorsHandlerExecutionChain(HttpServletRequest request,
        HandlerExecutionChain chain, @Nullable CorsConfiguration config) {
    if (CorsUtils.isPreFlightRequest(request)) {
        HandlerInterceptor[] interceptors = chain.getInterceptors();
        chain = new HandlerExecutionChain(new PreFlightHandler(config), interceptors);
    }
    else {
        // 添加到Interceptors的首位
        chain.addInterceptor(0, new CorsInterceptor(config));
    }
    return chain;
}

# HandlerExecutionChain#addInterceptor(int, HandlerInterceptor)
public void addInterceptor(int index, HandlerInterceptor interceptor) {
    // 其实就是操作一个list
    initInterceptorList().add(index, interceptor);
}

```

在 `HandlerExecutionChain` 中，有一个 `List` 用来存入 `Interceptor`，获取到的跨域配置，会添加到这个 `List` 的 `index=0` 的位置。

到这里，`handler` 就获取完成了，这个 `handler` 包含两部分：

*   `HandlerMethod`: 处理请求的方法，由于本文只分析 `@Controller` 方式的 controller，可以简单理解为有 `@RequestMapping` 注解的方法；
*   `List<Interceptor>`: 拦截器链，如果有跨域配置，那么跨域配置会放在这个 List 的第一位。

### 4\. 获取 `HandlerAdapter`

再回到 `DispatcherServlet#doDispatch` 方法，我们来看看获取 `HandlerAdapter` 的方法：

```
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) 
        throws Exception {
            ...
    // 2\. 获取对应的handlerAdapter，用来运行 handler(xxx)
    HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
    ...

```

进入 `getHandlerAdapter(xxx)` 方法：

> DispatcherServlet#getHandlerAdapter

```
protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
    // handlerAdapters 里的bean，也是由WebMvcConfigurationSupport引入的
    if (this.handlerAdapters != null) {
        for (HandlerAdapter adapter : this.handlerAdapters) {
            // 不同的handlerAdapter的判断方法不同
            if (adapter.supports(handler)) {
                return adapter;
            }
        }
    }
    throw new ServletException(...);
}

```

可以看到，这里会找到当前所有的 `adapter`，然后遍历，逐个判断是否能处理当前的 `handler`，所有的 `adapter` 如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-dedb2d58b9e99091aeb559f6c5a4aab4ccc.png)

再来看看如何判断是否能处理当前的 `handler` 的，我们看其中一个 `handler`，进入 `AbstractHandlerMethodAdapter#supports` 方法：

> AbstractHandlerMethodAdapter#supports

```
@Override
public final boolean supports(Object handler) {
    // 判断handler是否为HandlerMethod的实例
    return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
}

```

这里仅做了一个简单的判断，然后再调用 `supportsInternal` 方法，继续：

> RequestMappingHandlerAdapter#supportsInternal

```
protected boolean supportsInternal(HandlerMethod handlerMethod) {
    return true;
}

```

这个方法直接返回 true, 由于可见，如果 `handler` 的实例是 `HandlerMethod`，那么就会返回 `RequestMappingHandlerAdapter`.

这一步找到的 `adapter` 为 `RequestMappingHandlerAdapter`，这个 `adapter` 有什么用呢？限于篇幅，本文就先到这里了，剩下的流程下篇文章继续分析。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4717420](https://my.oschina.net/funcy/blog/4717420) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_