前面的文章中，我们分析了 `DispatcherServlet` 初始化流程，本文将来分析 `RequestMapping` 初始化流程。这里所说的 `RequestMapping` 初始化流程，直观来说，就是 spring 处理 `@RequestMaping` 注解的过程。

### 1\. 再谈 `@EnableWebMvc`

在 [spring mvc 之 springmvc demo 与 @EnableWebMvc 注解 ](https://my.oschina.net/funcy/blog/4696657)一文中提到，spring 通过 `@EnableWebMvc` 注解来启用 mvc 功能，最终通过 `@Import` 注解为项目引入了 `DelegatingWebMvcConfiguration.class`，该类通过 `@Bean` 注解的方法向 spring 中引入大量的 mvc 组件：

*   `public RequestMappingHandlerMapping requestMappingHandlerMapping(...)`
*   `public PathMatcher mvcPathMatcher()`
*   `public UrlPathHelper mvcUrlPathHelper()`
*   ...

这么组件中，与 `@RequestMaping` 注解相关的类就是 `RequestMappingHandlerMapping`.

### 2. `RequestMappingHandlerMapping#afterPropertiesSet` 方法

`RequestMappingHandlerMapping` 是创建是在 `WebMvcConfigurationSupport` 中：

```
@Bean
public RequestMappingHandlerMapping requestMappingHandlerMapping(
        @Qualifier("mvcContentNegotiationManager") ContentNegotiationManager contentNegotiationManager,
        @Qualifier("mvcConversionService") FormattingConversionService conversionService,
        @Qualifier("mvcResourceUrlProvider") ResourceUrlProvider resourceUrlProvider) {

    // 创建bean
    RequestMappingHandlerMapping mapping = createRequestMappingHandlerMapping();
    mapping.setOrder(0);
    mapping.setInterceptors(getInterceptors(conversionService, resourceUrlProvider));
    mapping.setContentNegotiationManager(contentNegotiationManager);
    mapping.setCorsConfigurations(getCorsConfigurations());

    // 处理各种配置，这里就是上一篇文章中提到的getXxx()方法获取配置
    PathMatchConfigurer configurer = getPathMatchConfigurer();
    Boolean useSuffixPatternMatch = configurer.isUseSuffixPatternMatch();
    if (useSuffixPatternMatch != null) {
        mapping.setUseSuffixPatternMatch(useSuffixPatternMatch);
    }
    Boolean useRegisteredSuffixPatternMatch = configurer.isUseRegisteredSuffixPatternMatch();
    if (useRegisteredSuffixPatternMatch != null) {
        mapping.setUseRegisteredSuffixPatternMatch(useRegisteredSuffixPatternMatch);
    }
    Boolean useTrailingSlashMatch = configurer.isUseTrailingSlashMatch();
    if (useTrailingSlashMatch != null) {
        mapping.setUseTrailingSlashMatch(useTrailingSlashMatch);
    }
    UrlPathHelper pathHelper = configurer.getUrlPathHelper();
    if (pathHelper != null) {
        mapping.setUrlPathHelper(pathHelper);
    }
    PathMatcher pathMatcher = configurer.getPathMatcher();
    if (pathMatcher != null) {
        mapping.setPathMatcher(pathMatcher);
    }
    Map<String, Predicate<Class<?>>> pathPrefixes = configurer.getPathPrefixes();
    if (pathPrefixes != null) {
        mapping.setPathPrefixes(pathPrefixes);
    }

    return mapping;
}

// 创建对象
protected RequestMappingHandlerMapping createRequestMappingHandlerMapping() {
    return new RequestMappingHandlerMapping();
}

```

这个方法就是用来创建对象 `RequestMappingHandlerMapping` 对象的，先是创建了一个对象，然后设置了各种属性。对象创建后，继续进行 spring bean 的生命周期，继而调用 `RequestMappingHandlerMapping#afterPropertiesSet` 方法：

```
@Override
public void afterPropertiesSet() {
    // 配置了一些属性
    this.config = new RequestMappingInfo.BuilderConfiguration();
    this.config.setUrlPathHelper(getUrlPathHelper());
    this.config.setPathMatcher(getPathMatcher());
    this.config.setSuffixPatternMatch(this.useSuffixPatternMatch);
    this.config.setTrailingSlashMatch(this.useTrailingSlashMatch);
    this.config.setRegisteredSuffixPatternMatch(this.useRegisteredSuffixPatternMatch);
    this.config.setContentNegotiationManager(getContentNegotiationManager());
    // 调用父类的方法
    super.afterPropertiesSet();
}

```

这个方法先是 配置了一些属性，然后再调用父类的 `afterPropertiesSet()`，继续追下去：

> AbstractHandlerMethodMapping#afterPropertiesSet

```
@Override
public void afterPropertiesSet() {
    initHandlerMethods();
}

protected void initHandlerMethods() {
    // 调用getCandidateBeanNames()获取容器中所有bean的beanName，
    // 然后挨个处理容器中所有 bean
    for (String beanName : getCandidateBeanNames()) {
        if (!beanName.startsWith(SCOPED_TARGET_NAME_PREFIX)) {
            // 逐个 bean 处理，继续往下看
            processCandidateBean(beanName);
        }
    }
    // 这个方法仅打了一个日志，没什么功能
    handlerMethodsInitialized(getHandlerMethods());
}

```

spring 在处理时，获取了容器中所有 bean 的 beanName，然后对 beanName 进行挨个处理，继续看 `AbstractHandlerMethodMapping#processCandidateBean`：

```
// 处理bean的具体逻辑
protected void processCandidateBean(String beanName) {
    // 获取 beanName 对应的 beanType
    // 1\. 如果是cglib代理，beanType 为 Xxx$$EnhancerBySpringCGLIB
    // 2\. 如果是jdk动态代理，beanType 为 com.sum.proxy.$Proxy
    Class<?> beanType = null;
    try {
        beanType = obtainApplicationContext().getType(beanName);
    }
    catch (Throwable ex) {
        ...
    }
    // isHandler: beanType上是否有 @Controller 或 @RequestMapping 注解
    if (beanType != null && isHandler(beanType)) {
        // 处理 handlerMethods
        detectHandlerMethods(beanName);
    }
}

```

这个方法还是比较简单，主要是获取 `beanName` 对应的 `beanType`，然后判断是否有 `@Controller/@RequestMapping` 注解，之后就调用 `AbstractHandlerMethodMapping#detectHandlerMethods` 进一步处理。

不过对于 `isHandler(Class)` 需要着重说明下：

1. 能识别 `@Controller`，同样能识别 `@RestController`，甚至其他标注 `@Controller` 的注解，即能识别如下注解：

   ```
   // 标记了 @Controller
   @Controller
   // 省略其他注解
   public @interface XxxController {
       ...
   }
   
   ```

2. 如果 `beanName` 对应 bean 是 cglib 代理 bean，beanType 为 `Xxx$$EnhancerBySpringCGLIB`，能识别其父类 (也就是目标类) 上的 `@Controller/@ReestMapping`;

3. 如果 `beanName` 对应的 bean 是 jdk 动态代理 bean，beanType 为 `com.sum.proxy.$Proxy`，能识别其父接口上的 `@Controller/@RequestMapping`;

4. 如果 beanType 是 `com.sum.proxy.$Proxy`(jdk 动态代理类)，** 无法识别其目标类上的 `@Controller/@RequestMapping` ** 的；

5. 标注 `@Controller/@RequestMapping` 的类要实现 jdk 动态代理，需要将 `@Controller/@RequestMapping` 放在接口及接口的方法上。

到了这里，传入 `AbstractHandlerMethodMapping#detectHandlerMethods` 的 `beanType` 都是标注了 `@Controller/@RequestMapping` 的类或接口了。

### 3. `AbstractHandlerMethodMapping#detectHandlerMethods`

继续看看 `AbstractHandlerMethodMapping#detectHandlerMethods` 的内容：

```
// 检测handler方法
protected void detectHandlerMethods(Object handler) {
    Class<?> handlerType = (handler instanceof String ?
            obtainApplicationContext().getType((String) handler) : handler.getClass());
    if (handlerType != null) {
        // 1\. 针对cglib代理对象，得到其父类，也就是目标对象的类
        Class<?> userType = ClassUtils.getUserClass(handlerType);
        // 2\. 这里会处理 userType、userType的不包括Object的所有父类及 userType 的所有接口的方法
        Map<Method, T> methods = MethodIntrospector.selectMethods(userType,
                // 3\. 对每个方法，如果有 @RequestMapping，则创建 RequestMappingInfo
                // 将 @RequestMapping 信息封装到该对象中
                (MethodIntrospector.MetadataLookup<T>) method -> {
                    try {
                        // 在这里处理方法上的 @RequestMapping 注解
                        return getMappingForMethod(method, userType);
                    }
                    catch (Throwable ex) {
                        ...
                    }
                });
        methods.forEach((method, mapping) -> {
            Method invocableMethod = AopUtils.selectInvocableMethod(method, userType);
            // 4\. 在这里将handler、mapping与method保存起来
            registerHandlerMethod(handler, invocableMethod, mapping);
        });
    }
}

```

这个方法做了如下几件事：

1.  针对 cglib 代理对象，得到其父类，也就是目标对象的类（**这里为何只处理 cglib 代理对象，而不处理 jdk 动态代理对象呢？** 从上面对 `isHandler(Class)` 的说明可知，spring 并不能识别 jdk 动态代理对象对应类上的 `@Controller/@RequestMapping` 注解，因此不会执行到这里）；
2.  处理 `userType`、`userType` 的不包括 Object 的所有父类及 `userType` 的所有接口的方法；
3.  对每个方法，如果有 `@RequestMapping`，则创建 `RequestMappingInfo`，将 `@RequestMapping` 信息封装到该对象中；
4.  注册，将 `handler`、`mapping` 与 `method` 保存到 Map 中。

#### 3.1 查找方法

在 `detectHandlerMethods` 方法的处理过程中，会查找 `userType`、`userType` 的所有父类（不包括 Object）及 `userType` 的所有接口的方法，过程如下：

> MethodIntrospector#selectMethods(Class, MethodIntrospector.MetadataLookup)

```
public static <T> Map<Method, T> selectMethods(Class<?> targetType, final 
            MetadataLookup<T> metadataLookup) {
    final Map<Method, T> methodMap = new LinkedHashMap<>();
    Set<Class<?>> handlerTypes = new LinkedHashSet<>();
    Class<?> specificHandlerType = null;
    // 非jdk动态代理类
    if (!Proxy.isProxyClass(targetType)) {
        // 如果是cglib代理类，获取其具体的父类 class
        specificHandlerType = ClassUtils.getUserClass(targetType);
        handlerTypes.add(specificHandlerType);
    }
    // 获取类的所有接口，包括接口的父接口.
    handlerTypes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetType));
    for (Class<?> currentHandlerType : handlerTypes) {
        final Class<?> targetClass = (specificHandlerType != null 
                      ? specificHandlerType : currentHandlerType);
        // 处理currentHandlerType、currentHandlerType的不包括Object的所有父类、
        // currentHandlerType的所有接口的方法
        // 处理aop时，调用的也是这个方法
        ReflectionUtils.doWithMethods(currentHandlerType, method -> {
            Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
            T result = metadataLookup.inspect(specificMethod);
            if (result != null) {
                Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
                if (bridgedMethod == specificMethod || 
                             metadataLookup.inspect(bridgedMethod) == null) {
                    methodMap.put(specificMethod, result);
                }
            }
        }, ReflectionUtils.USER_DECLARED_METHODS);
    }
    return methodMap;
}

```

#### 3.2 创建 `RequestMappingInfo`

对每个方法，如果有 `@RequestMapping`，则创建 `RequestMappingInfo`，将 `@RequestMapping` 信息封装到该对象中：

> RequestMappingHandlerMapping#getMappingForMethod

```
protected RequestMappingInfo getMappingForMethod(Method method, Class<?> handlerType) {
    // 处理方法上的 @RequestMapping
    RequestMappingInfo info = createRequestMappingInfo(method);
    if (info != null) {
        // 处理类上的 @RequestMapping
        RequestMappingInfo typeInfo = createRequestMappingInfo(handlerType);
        if (typeInfo != null) {
            // 合并方法与类上的 @RequestMapping 结果
            // 如类的 @RequestMapping("/test")，方法上的 @RequestMapping("/hello")
            // 合并后的结果为 /test/hello
            info = typeInfo.combine(info);
        }
        String prefix = getPathPrefix(handlerType);
        if (prefix != null) {
            info = RequestMappingInfo.paths(prefix).options(this.config).build().combine(info);
        }
    }
    return info;
}

@Nullable
private RequestMappingInfo createRequestMappingInfo(AnnotatedElement element) {
    // 获取 @RequestMapping 注解
    RequestMapping requestMapping = AnnotatedElementUtils
            .findMergedAnnotation(element, RequestMapping.class);
    // 处理请求条件，实际上这个方法为空
    RequestCondition<?> condition = (element instanceof Class ?
            getCustomTypeCondition((Class<?>) element) : getCustomMethodCondition((Method) element));
    return (requestMapping != null ? createRequestMappingInfo(requestMapping, condition) : null);
}

// 创建 RequestMappingInfo，这个 RequestMapping 就是 @RequestMapping 注解
protected RequestMappingInfo createRequestMappingInfo(
        RequestMapping requestMapping, @Nullable RequestCondition<?> customCondition) {
    // 其实就是将 @RequestMapping 注解封装为RequestMappingInfo对象
    RequestMappingInfo.Builder builder = RequestMappingInfo
            .paths(resolveEmbeddedValuesInPatterns(requestMapping.path()))
            // 下面的属性皆来自 @RequestMapping 注解
            .methods(requestMapping.method())
            .params(requestMapping.params())
            .headers(requestMapping.headers())
            .consumes(requestMapping.consumes())
            .produces(requestMapping.produces())
            .mappingName(requestMapping.name());
    if (customCondition != null) {
        builder.customCondition(customCondition);
    }
    return builder.options(this.config).build();
}

```

到了这里，我们就完成了由 `@RequestMapping` 到 `RequestMappingInfo对象`的转化。

再来看看 `RequestMappingInfo` 长什么样：

```
public final class RequestMappingInfo implements RequestCondition<RequestMappingInfo> {
    // 提供了很多属性，对应着 @RequestMapping的属性
    @Nullable
    private final String name;
    private final PatternsRequestCondition patternsCondition;
    private final RequestMethodsRequestCondition methodsCondition;
    private final ParamsRequestCondition paramsCondition;
    private final HeadersRequestCondition headersCondition;
    private final ConsumesRequestCondition consumesCondition;
    private final ProducesRequestCondition producesCondition;
    private final RequestConditionHolder customConditionHolder;

    // 构造方法，就是在
    public RequestMappingInfo(@Nullable String name, @Nullable PatternsRequestCondition patterns,
            @Nullable RequestMethodsRequestCondition methods, @Nullable ParamsRequestCondition params,
            @Nullable HeadersRequestCondition headers, @Nullable ConsumesRequestCondition consumes,
            @Nullable ProducesRequestCondition produces, @Nullable RequestCondition<?> custom) {

        this.name = (StringUtils.hasText(name) ? name : null);
        this.patternsCondition = (patterns != null ? patterns : new PatternsRequestCondition());
        this.methodsCondition = (methods != null ? methods : new RequestMethodsRequestCondition());
        this.paramsCondition = (params != null ? params : new ParamsRequestCondition());
        this.headersCondition = (headers != null ? headers : new HeadersRequestCondition());
        this.consumesCondition = (consumes != null ? consumes : new ConsumesRequestCondition());
        this.producesCondition = (produces != null ? produces : new ProducesRequestCondition());
        this.customConditionHolder = new RequestConditionHolder(custom);
    }

    // builder 构建模式，前面正是使用builder来创建RequestMappingInfo对象的
    private static class DefaultBuilder implements Builder {
        // 省略其他
        ...

        //  使用builder()方法来创建对象
        @Override
        public RequestMappingInfo build() {
            ContentNegotiationManager manager = this.options.getContentNegotiationManager();
            PatternsRequestCondition patternsCondition = new PatternsRequestCondition(
                    this.paths, this.options.getUrlPathHelper(), this.options.getPathMatcher(),
                    this.options.useSuffixPatternMatch(), this.options.useTrailingSlashMatch(),
                    this.options.getFileExtensions());
            // 调用 RequestMappingInfo 构造方法
            return new RequestMappingInfo(this.mappingName, patternsCondition,
                    new RequestMethodsRequestCondition(this.methods),
                    new ParamsRequestCondition(this.params),
                    new HeadersRequestCondition(this.headers),
                    new ConsumesRequestCondition(this.consumes, this.headers),
                    new ProducesRequestCondition(this.produces, this.headers, manager),
                    this.customCondition);
        }
    }
    // 省略其他
    ...
}

```

#### 3.3 注册

封装完 `@RequestMapping` 信息后，接下来就是将接口信息注册到 springmvc 中了：

> RequestMappingHandlerMapping#registerHandlerMethod

```
@Override
protected void registerHandlerMethod(Object handler, Method method, RequestMappingInfo mapping) {
    // 调用父类的方法，注册相关逻辑在这里
    super.registerHandlerMethod(handler, method, mapping);
    updateConsumesCondition(mapping, method);
}
// 处理方法参数的 @RequestBody 注解
private void updateConsumesCondition(RequestMappingInfo info, Method method) {
    ConsumesRequestCondition condition = info.getConsumesCondition();
    if (!condition.isEmpty()) {
        for (Parameter parameter : method.getParameters()) {
            // 处理 方法参数的 @RequestBody 注解，设置 BodyRequired 的值
            MergedAnnotation<RequestBody> annot = MergedAnnotations.from(parameter)
                    .get(RequestBody.class);
            if (annot.isPresent()) {
                condition.setBodyRequired(annot.getBoolean("required"));
                break;
            }
        }
    }
}

```

最终，发现具体的注册逻辑是在 `AbstractHandlerMethodMapping#registerHandlerMethod` 中完成的，看来这个就是最终方法了。在分析这个方法前，我们先来看看这一步得到的 `Map<Method, T> methods`:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-ec6b4b2d2156e0e041982425173feda38fc.png)

可以看到，对应的 `T` 就是 `RequestMappingInfo` 了。

### 3. `AbstractHandlerMethodMapping#registerHandlerMethod` 方法

接下来我们来看看 `AbstractHandlerMethodMapping#registerHandlerMethod` 代码：

```
public abstract class AbstractHandlerMethodMapping<T> extends AbstractHandlerMapping 
        implements InitializingBean {

    protected void registerHandlerMethod(Object handler, Method method, T mapping) {
        this.mappingRegistry.register(mapping, handler, method);
    }

    // 省略了好多代码
    ... 

    class MappingRegistry {
        // 信息最全的map，包含mapping, handlerMethod, directUrls, name等信息
        private final Map<T, MappingRegistration<T>> registry = new HashMap<>();
        // 所有的 mapping map，/test/hello、/test/{name}
        private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();
        // 明确的url map，如 /test/hello
        private final MultiValueMap<String, T> urlLookup = new LinkedMultiValueMap<>();

        // 省略了好多代码
        ...

        public void register(T mapping, Object handler, Method method) {
            ...
            // 获取读写锁的写锁
            this.readWriteLock.writeLock().lock();
            try {
                // 1\. 获取了 handlerMethod，其实就是将handler 与 method 包装了一层
                HandlerMethod handlerMethod = createHandlerMethod(handler, method);
                validateMethodMapping(handlerMethod, mapping);
                // 2\. 放入 mappingLookup 中，类型为 LinkedHashMap
                // 这是springmvc中一个重要的map
                this.mappingLookup.put(mapping, handlerMethod);

                // 3\. 获取url，放入urlLookup，类型为MultiValueMap，这个map 同一key可以有多个value
                // 这是springmvc中另一个重要的map
                List<String> directUrls = getDirectUrls(mapping);
                for (String url : directUrls) {
                    this.urlLookup.add(url, mapping);
                }
                String name = null;
                if (getNamingStrategy() != null) {
                    name = getNamingStrategy().getName(handlerMethod, mapping);
                    addMappingName(name, handlerMethod);
                }
                CorsConfiguration corsConfig = initCorsConfiguration(handler, method, mapping);
                if (corsConfig != null) {
                    this.corsLookup.put(handlerMethod, corsConfig);
                }

                // 4\. 将mapping, handlerMethod, directUrls, name等封装放入registry，
                // registry 类型为HashMap，这是springmvc 中接口信息最全的一个map
                this.registry.put(mapping, new MappingRegistration<>(mapping, 
                        handlerMethod, directUrls, name));
            }
            finally {
                this.readWriteLock.writeLock().unlock();
            }
        }
    }
}

```

可以看到，注册的逻辑最终是在 `AbstractHandlerMethodMapping.MappingRegistry#register` 中完成的。接下来我们就一步步分析注册逻辑。

#### 3.1 获取 `HandlerMethod`

相关代码如下：

```
protected HandlerMethod createHandlerMethod(Object handler, Method method) {
    if (handler instanceof String) {
        return new HandlerMethod((String) handler,
                obtainApplicationContext().getAutowireCapableBeanFactory(), method);
    }
    return new HandlerMethod(handler, method);
}

```

这段方法就是简单地调用了 `HandlerMethod` 的构造方法，继续：

```
public class HandlerMethod {

    // 提供了非常多的属性
    protected final Log logger = LogFactory.getLog(getClass());
    private final Object bean;
    @Nullable
    private final BeanFactory beanFactory;
    private final Class<?> beanType;
    private final Method method;
    private final Method bridgedMethod;
    private final MethodParameter[] parameters;
    @Nullable
    private HttpStatus responseStatus;
    @Nullable
    private String responseStatusReason;
    @Nullable
    private HandlerMethod resolvedFromHandlerMethod;
    @Nullable
    private volatile List<Annotation[][]> interfaceParameterAnnotations;
    private final String description;

    // 构造方法
    public HandlerMethod(Object bean, Method method) {
        Assert.notNull(bean, "Bean is required");
        Assert.notNull(method, "Method is required");
        this.bean = bean;
        this.beanFactory = null;
        this.beanType = ClassUtils.getUserClass(bean);
        this.method = method;
        this.bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
        this.parameters = initMethodParameters();
        evaluateResponseStatus();
        this.description = initDescription(this.beanType, this.method);
    }

    // 省略其他
    ...
}

```

可以看到，`HandlerMethod` 中有非常多的属性，构造方法所做的事也仅仅是赋值而已。由此可看出，`HandlerMethod` 就是对 `handler` 与 `method` 的一个包装。

#### 3.2 验证 mapping 是否重复

在 springmvc 使用中，如果不小心定义了两个相同的 `requestMapping`，会出现如下异常：

```
Caused by: java.lang.IllegalStateException: Ambiguous mapping. Cannot map 
'xxxController' method xxxMethod to /xxx/xxx: There is already 'xxxControllter' 
bean method xxxMethod mapped.

```

这个异常就是在验证 mapping 时，发现了重复了 mapping 而触发的，代码如下：

```
// 所有的 mapping map，/test/hello、/test/{name}
private final Map<T, HandlerMethod> mappingLookup = new LinkedHashMap<>();

private void validateMethodMapping(HandlerMethod handlerMethod, T mapping) {
    // 找到已存在的 method
    HandlerMethod existingHandlerMethod = this.mappingLookup.get(mapping);
    // 已存在的handlerMethod不为空，且不等于当前 handlerMethod
    if (existingHandlerMethod != null && !existingHandlerMethod.equals(handlerMethod)) {
        throw new IllegalStateException(
                "Ambiguous mapping. Cannot map '" + handlerMethod.getBean() + "' method \n" +
                handlerMethod + "\nto " + mapping + ": There is already '" +
                existingHandlerMethod.getBean() + "' bean method\n" +
                existingHandlerMethod + " mapped.");
    }
}

```

首先是根据 `mapping` 从 `mappingLookup` 中查找 `HandlerMethod`，如果找到了且找到的 `handlerMethod` 不是当前 `handlerMethod`，则表示重复，就报异常了。

最后分别来看看判断 `HandlerMethod` 与 `RequestMappingInfo` 是如何判断相等的：

> HandlerMethod#equals

```
@Override
public boolean equals(@Nullable Object other) {
    if (this == other) {
        return true;
    }
    if (!(other instanceof HandlerMethod)) {
        return false;
    }
    HandlerMethod otherMethod = (HandlerMethod) other;
    return (this.bean.equals(otherMethod.bean) && this.method.equals(otherMethod.method));
}

```

> RequestMappingInfo#equals

```
@Override
public boolean equals(@Nullable Object other) {
    if (this == other) {
        return true;
    }
    if (!(other instanceof RequestMappingInfo)) {
        return false;
    }
    RequestMappingInfo otherInfo = (RequestMappingInfo) other;
    return (this.patternsCondition.equals(otherInfo.patternsCondition) &&
            this.methodsCondition.equals(otherInfo.methodsCondition) &&
            this.paramsCondition.equals(otherInfo.paramsCondition) &&
            this.headersCondition.equals(otherInfo.headersCondition) &&
            this.consumesCondition.equals(otherInfo.consumesCondition) &&
            this.producesCondition.equals(otherInfo.producesCondition) &&
            this.customConditionHolder.equals(otherInfo.customConditionHolder));
}

```

`RequestMappingInfo` 需要各属性相同才判断相等，因此像下面这样的 `@RequestMapping`，得到的 `RequestMappingInfo` 并不相等：

```
// 以下三个 @RequestMapping，虽然请求路径都是“/hello”，但支持的请求方法各不相同
// 因此得到的 RequestMappingInfo 并不相等

@RequestMapping(path = "/hello")
public String hello1() {
    ...
}

@RequestMapping(path = "/hello", method = RequestMethod.GET)
public String hello2() {
    ...
}

@RequestMapping(path = "/hello", method = RequestMethod.POST)
public String hello3() {
    ...
}

```

#### 3.3 获取 `directUrls`

springmvc 中，有两种 url 类型:

1. 明确的 url，如

   ```
   @RequestMapping("/hello")
   public String hello() {
      ...
   }
   
   ```

2. 不明确的 url，如

   ```
   @RequestMapping("/{name}")
   public String hello(@PathVariable("name") String name) {
      ...
   }
   
   ```

springmvc 提供了专门的 `urlLookup` 来保存明确的 url，结构如下：

```
MultiValueMap<String, LinkedList<RequestMappingInfo>>

```

再来看看 springmvc 是如何获取明确的 url 的:

```
List<String> directUrls = getDirectUrls(mapping);

/**
 * AbstractHandlerMethodMapping.MappingRegistry#getDirectUrls
 * 获取明确的url
 */
private List<String> getDirectUrls(T mapping) {
    List<String> urls = new ArrayList<>(1);
    // 从RequestMappingInfo获取所有的 MappingPathPattern
    for (String path : getMappingPathPatterns(mapping)) {
        // 判断得到的 MappingPathPattern 是否为明确的url
        if (!getPathMatcher().isPattern(path)) {
            urls.add(path);
        }
    }
    return urls;
}

/**
 * RequestMappingInfoHandlerMapping#getMappingPathPatterns
 * 获取 patterns, 从 RequestMappingInfo 获取
 * 实际上就是 @RequestMapping 中的 path() 值
 */
@Override
protected Set<String> getMappingPathPatterns(RequestMappingInfo info) {
    return info.getPatternsCondition().getPatterns();
}

/**
 * AntPathMatcher#isPattern
 * 判断是否为明确的 url
 * 只要包含了 *、？之一，或同时包含{、}，就不是明确的url
 */
@Override
public boolean isPattern(@Nullable String path) {
    if (path == null) {
        return false;
    }
    boolean uriVar = false;
    for (int i = 0; i < path.length(); i++) {
        char c = path.charAt(i);
        if (c == '*' || c == '?') {
            return true;
        }
        if (c == '{') {
            uriVar = true;
            continue;
        }
        if (c == '}' && uriVar) {
            return true;
        }
    }
    return false;
}

```

流程如下：

1.  获取当前 `mapping` 的所有 `path`，也就是 `@RequestMapping` 的 `path()` 值；
2.  遍历得到的 `path`，逐一判断是否为明确的 url (只要包含了 `*`、`?` 之一，或同时包含 `{`、`}`，就不是明确的 url).

#### 3.4 注册接口信息

注册接口信息就比较简单了，其实就是往 map 中添加数据，这里介绍三个 map：

*   `urlLookup`：明确的 `url map`，类型为 `MultiValueMap<String, LinkedList<RequestMappingInfo>>`，`key` 为明确的 url，如 `/test/hello`，`value` 为 `LinkedList<RequestMappingInfo>`；

*   `mappingLookup`：所有的 `mapping map`，所有的 `@RequestMapping` 对应的 `RequestMappingInfo` 都能在这里找到，包括 `/test/hello`、`/test/{name}` 对应的 `RequestMappingInfo`，类型为 `Map<RequestMappingInfo, HandlerMethod>`；

*   `registry`：信息最全的 map，类型为 `Map<RequestMappingInfo, MappingRegistration<RequestMappingInfo>>`，包含所有的 `RequestMappingInfo`，key 为 `RequestMappingInfo`，value 为 `MappingRegistration<RequestMappingInfo>`，而 `MappingRegistration` 为 `mapping`, `handlerMethod`, `directUrls`, `name` 的包装类，也就是说 `MappingRegistration` 包含了 `mapping`, `handlerMethod`, `directUrls`, `name` 等信息。

理解这些 map 后，注册就相当简单了，就是简单地调用下 `Map#put` 方法，就不多说了。

### 4\. 总结

本文分析了 spring 处理处理 `@RequestMapping` 注解的流程，这部分流程在 `RequestMappingHandlerMapping#afterPropertiesSet` 方法中，流程如下：

1.  获取容器中所有 bean 的 `beanName`，逐个处理，处理方法见第 2 步；
2.  找到与 `beanName` 对应的 `beanType`，判断其上是否有 `@Controller/@RequestMapping` 注解；
3.  对包含 `@Controller/@RequestMapping` 的 `beanType`，找到有 `@RequestMapping` 注解的方法，将其 `@RequestMapping` 注解封装为 `RequestMappingInfo`，这一步得到的结果为一个 map：`Map<Method, RequestMappingInfo>`；
4.  将 `beanName`、`beanType` 与 `Map<Method, RequestMappingInfo>` 注册到 springmvc 中，这里有三个比较重要的 map：
   *   `MultiValueMap<String, LinkedList<RequestMappingInfo>>`、
   *   `Map<RequestMappingInfo, HandlerMethod>`(`HandlerMethod` 为 `Method` 的包装类)
   *   `Map<RequestMappingInfo, MappingRegistration<RequestMappingInfo>>`(`MappingRegistration` 为 `RequestMappingInfo`, `HandlerMethod`, `directUrls`, `beanName` 的包装类)；

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-fe807564e60040b262b28977a46a74f60e9.png)

总的来说，`RequestMapping` 的处理流程隐藏比较深，本人也是调试了好多次才找到。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4715079](https://my.oschina.net/funcy/blog/4715079) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_