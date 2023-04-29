**学习目标**

1.  Gateway核心原理分析
    **第1章 Bean的准备**
    前面也讲了这么多组件了，这会儿我们集成
    spring-cloud-starter-gateway组件发现，又是一个starter组件，二话不说，先去找spring.factories文件，分析一下有哪些重要的bean被自动装配进IoC容器里面了。

![SpringCloud系列—Spring Cloud 源码分析之Gateway网关-开源基础软件社区](https://dl-harmonyos.51cto.com/images/202207/788f3c1494307a2ad7d935811c9e62bab2c435.jpg "SpringCloud系列—Spring Cloud 源码分析之Gateway网关-开源基础软件社区")1.先来看
GatewayClassPathWarningAutoConfiguration这个配置类



```
@Configuration(proxyBeanMethods = false)
//当前配置类在GatewayAutoConfiguration这个核心配置类之前加载
@AutoConfigureBefore(GatewayAutoConfiguration.class)
public class GatewayClassPathWarningAutoConfiguration {
	...      
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(name = "org.springframework.web.servlet.DispatcherServlet")
	protected static class SpringMvcFoundOnClasspathConfiguration {
		public SpringMvcFoundOnClasspathConfiguration() {
			log.warn(BORDER
					+ "Spring MVC found on classpath, which is incompatible with Spring Cloud Gateway at this time. "
					+ "Please remove spring-boot-starter-web dependency." + BORDER);
		}

	}
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnMissingClass("org.springframework.web.reactive.DispatcherHandler")
	protected static class WebfluxMissingFromClasspathConfiguration {

		public WebfluxMissingFromClasspathConfiguration() {
			log.warn(BORDER + "Spring Webflux is missing from the classpath, "
					+ "which is required for Spring Cloud Gateway at this time. "
					+ "Please add spring-boot-starter-webflux dependency." + BORDER);
		}
	}
}
```









从这个配置类能看出来，它实际上就通过ConditionOnClass和ConditionOnMissingClass两个做了两个日志打印的功能；如果ClassPath下有
org.springframework.web.servlet.DispatcherServlet类的话，则实例第一个Bean对象，然后打印日志：不能依赖spring-boot-starter-web这个包。然后再检查ClassPath下是否有正确的配置webflux，如果没有，则打印日志：加spring-boot-starter-webflux依赖。

2.核心配置类GatewayAutoConfiguration

因为代码太长，这里就不展示了，这里就列举几个比较重要的

*   PropertiesRouteDefinitionLocator：用于从配置文件（yml/properties）中读取路由配置信息！
*   RouteDefinitionLocator：把 RouteDefinition 转化为 Route
*   RoutePredicateHandlerMapping：类似于 mvc 的HandlerMapping，不过这里是 Gateway实现的。用于匹配对应的请求route
*   GatewayProperties：yml配置信息封装在 GatewayProperties 对象中
*   AfterRoutePredicateFactory：各种路由断言工厂，正是这些断言工厂在启动时已经生成对应的bean，我们才可以在 yml 中配置一下，即可生效
*   RetryGatewayFilterFactory：各种 Gateway 过滤器，正是这些过滤器在启动时已经生成对应的bean，我们才可以在 yml 中配置一下，即可生效
*   GlobalFilter实现类：全局过滤器

3.HttpHandlerAutoConfiguration和WebFluxAutoConfiguration配置类，在GatewayAutoConfiguration之后实例化，分别实例化了HttpHandler和WebFluxConfigBean

**第2章 执行流程**
上一文中讲到Hystrix的原理，在Hystrix中核心业务逻辑都是通过响应式编程完成的，事实上，在Gateway中也都是基于同样的编程风格。同样的，Gateway的流程同SpringMVC流程也非常相似。

当前端有请求进来的时候，大体的流程如下：

1.  首先被DispatcherHandler给捕获拦截，然后对请求的URI进行解析
2.  然后根据URI去调用HandlerMapping，获取真正要执行的WebHandler
3.  然后选择一个合适的适配器HandlerAdapter执行
4.  执行WebHandler
    当请求gateway服务时，所有的请求都会进入到DispatcherHandler中的handle方法，下面我们一起看看这个方法



```
@Override
public Mono<Void> handle(ServerWebExchange exchange) {
    if (this.handlerMappings == null) {
        return createNotFoundError();
    }
    //这里就是webFlux的响应式编程
    return Flux
        // 1.这里就是遍历所有的 handlerMapping
        .fromIterable(this.handlerMappings)
        // 2.获取对应的handlerMapping ，比如常用的 RequestMappingHandlerMapping、RoutePredicateHandlerMapping
        .concatMap(mapping -> mapping.getHandler(exchange))
        .next()
        .switchIfEmpty(createNotFoundError())
        // 3.获取对应的适配器，调用对应的处理器
        .flatMap(handler -> invokeHandler(exchange, handler))
        // 4.返回处理结果
        .flatMap(result -> handleResult(exchange, result));
}
```









**2.1 getHandler**
我们先来看看getHandler方法，它就是Gateway的核心逻辑所在，再getHandler中获取对应的HandlerMapping。

下面是
AbstractHandlerMapping.getHandler的源码



```
@Override
public Mono<Object> getHandler(ServerWebExchange exchange) {
    //这一步会获取路由的实现类，会进入到RoutePredicateHandlerMapping
    return getHandlerInternal(exchange).map(handler -> {
        if (logger.isDebugEnabled()) {
            logger.debug(exchange.getLogPrefix() + "Mapped to " + handler);
        }
        ServerHttpRequest request = exchange.getRequest();
        if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
            CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(exchange) : null);
            CorsConfiguration handlerConfig = getCorsConfiguration(handler, exchange);
            config = (config != null ? config.combine(handlerConfig) : handlerConfig);
            if (!this.corsProcessor.process(config, exchange) || CorsUtils.isPreFlightRequest(request)) {
                return REQUEST_HANDLED_HANDLER;
            }
        }
        return handler;
    });
}
```











```
@Override
protected Mono<?> getHandlerInternal(ServerWebExchange exchange) {
    // don't handle requests on management port if set and different than server port
    if (this.managementPortType == DIFFERENT && this.managementPort != null
        && exchange.getRequest().getURI().getPort() == this.managementPort) {
        return Mono.empty();
    }
    exchange.getAttributes().put(GATEWAY_HANDLER_MAPPER_ATTR, getSimpleName());
    //寻找并匹配路由
    return lookupRoute(exchange)
        // .log("route-predicate-handler-mapping", Level.FINER) //name this
        .flatMap((Function<Route, Mono<?>>) r -> {
            //移除上下文中旧的属性
            exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
            if (logger.isDebugEnabled()) {
                logger.debug(
                    "Mapping [" + getExchangeDesc(exchange) + "] to " + r);
            }
            //把该路由与上下文绑定，后续负载均衡会用
            exchange.getAttributes().put(GATEWAY_ROUTE_ATTR, r);
            //返回 webHandler
            return Mono.just(webHandler);
        }).switchIfEmpty(Mono.empty().then(Mono.fromRunnable(() -> {
        exchange.getAttributes().remove(GATEWAY_PREDICATE_ROUTE_ATTR);
        if (logger.isTraceEnabled()) {
            logger.trace("No RouteDefinition found for ["
                         + getExchangeDesc(exchange) + "]");
        }
    })));
}
```









其中lookupRoute方法会找到yml中配置的所有的路由断言工厂（Before、After、Path等等），并执行apply方法，进行路由匹配，判断是否允许请求通过！执行顺序由springboot自动配置时自己制定



```
protected Mono<Route> lookupRoute(ServerWebExchange exchange) {
    // getRoutes 获取所有的断言工厂
    return this.routeLocator.getRoutes()
        .concatMap(route -> Mono.just(route).filterWhen(r -> {
            exchange.getAttributes().put(GATEWAY_PREDICATE_ROUTE_ATTR, r.getId());
            // 先获取Route内部的predicate属性
            //然后调用apply方法 执行断言！判断请求是否通过
            return r.getPredicate().apply(exchange);
        }).doOnError(e -> logger.error(
                       "Error applying predicate for route: " + route.getId(),
                       e))
                   .onErrorResume(e -> Mono.empty()))
        .next()
        .map(route -> {
            if (logger.isDebugEnabled()) {
                logger.debug("Route matched: " + route.getId());
            }
            validateRoute(route, exchange);
            return route;
        });
}
```









其中getRoutes()方法就是通过
RouteDefinitionRouteLocator从配置文件中获取所有路由的，然后把找到的路由转换成Route



```
@Override
public Flux<Route> getRoutes() {
    // getRouteDefinitions() 从配置文件中获取所有路由
    Flux<Route> routes = this.routeDefinitionLocator.getRouteDefinitions()
        // convertToRoute()：把找到的路由转换成Route
        .map(this::convertToRoute);
    ...
}
```











```
public class Route implements Ordered {
	private final String id;
	private final URI uri;
	private final int order;
	private final AsyncPredicate<ServerWebExchange> predicate;
	private final List<GatewayFilter> gatewayFilters;
	private final Map<String, Object> metadata;	
    ...
}
```









**2.2 invokeHandler**
Gateway由于在上一步匹配路由后返回的是webHandler类型的，所以也需要找到对应的HandlerAdaptor，进入获取对应的适配器方法 invokeHandler(exchange, handler)中



```
private Mono<HandlerResult> invokeHandler(ServerWebExchange exchange, Object handler) {
    if (this.handlerAdapters != null) {
        //找到所有的HandlerAdapter去匹配WebFlux类型
        for (HandlerAdapter handlerAdapter : this.handlerAdapters) {
            if (handlerAdapter.supports(handler)) {
                return handlerAdapter.handle(exchange, handler);
            }
        }
    }
    return Mono.error(new IllegalStateException("No HandlerAdapter: " + handler));
}
```









SimpleHandlerAdapter 中的handle方法如下



```
@Override
public Mono<HandlerResult> handle(ServerWebExchange exchange, Object handler) {
    //处理WebHandler 类型
    WebHandler webHandler = (WebHandler) handler;
    Mono<Void> mono = webHandler.handle(exchange);
    return mono.then(Mono.empty());
}
```









其中webHandler.handle方法就是处理所有过滤器链的方法，该过滤器链包括globalFilters和gatewayFilters



```
@Override
public Mono<Void> handle(ServerWebExchange exchange) {
    // 1\. 根据路由与上下文绑定关系，获取对应的路由Route
    Route route = exchange.getRequiredAttribute(GATEWAY_ROUTE_ATTR);
    List<GatewayFilter> gatewayFilters = route.getFilters();
    // 2\. 收集所有的 globalFilters 并放入List<GatewayFilter>
    //注意这里使用了适配器模式
    List<GatewayFilter> combined = new ArrayList<>(this.globalFilters);
    // 3\. 把 gatewayFilters 也放入List<GatewayFilter>，形成一条过滤器立案
    combined.addAll(gatewayFilters);
    // 4\. 根据order排序
    AnnotationAwareOrderComparator.sort(combined);
    if (logger.isDebugEnabled()) {
        logger.debug("Sorted gatewayFilterFactories: " + combined);
    }
    // 5\. 执行过滤器链中的每一个过滤器方法！
    return new DefaultGatewayFilterChain(combined).filter(exchange);
}
```









注意：在组装过滤器链的时候，是把globalFilters和gatewayFilters两种过滤器都放进了List<GatewayFilter>中，这是怎么做的呢？

这其实用到了一种 适配器 的设计模式！

*   如果放入的是globalFilters，会先把globalFilters转化成GatewayFilterAdapter。 GatewayFilterAdapter在内部集成了GlobalFilter，同时也实现了GatewayFilter，使 globalFilters和gatewayFilters在 适配器 类GatewayFilterAdapter中共存！
*   如果放入的是gatewayFilters，直接放入即可！
    **第3章 负载均衡流程**
    Gateway的负载均衡只需要在yml中配置 uri: lb://user即可实现负载均衡，底层是由全局过滤器LoadBalancerClientFilter的filter方法去做的！

以订单服务的
http://localhost:9527/get/3为例！9527为网关Gateway的端口



```
public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
    // 1\. 根据路由与上下文绑定关系
    // 获取原始的url：http://localhost:9527/get/3
    URI url = exchange.getAttribute(GATEWAY_REQUEST_URL_ATTR);
    String schemePrefix = exchange.getAttribute(GATEWAY_SCHEME_PREFIX_ATTR);
    if (url == null
        || (!"lb".equals(url.getScheme()) && !"lb".equals(schemePrefix))) {
        return chain.filter(exchange);
    }
    addOriginalRequestUrl(exchange, url);
    if (log.isTraceEnabled()) {
        log.trace("LoadBalancerClientFilter url before: " + url);
    }
    // 2\. 通过ribbon的负载均衡算法，根据服务名去nacos或者Eureka选择一个实例！
    // 该实例就有user服务真正的 url 地址：http://localhost:8080/get/3
    final ServiceInstance instance = choose(exchange);
    if (instance == null) {
        throw NotFoundException.create(properties.isUse404(),
                                       "Unable to find instance for " + url.getHost());
    }
    // 3\. 拿到原生的 uri ：http://localhost:9527/get/3
    URI uri = exchange.getRequest().getURI();
    String overrideScheme = instance.isSecure() ? "https" : "http";
    if (schemePrefix != null) {
        overrideScheme = url.getScheme();
    }
    // 4\. 拿服务实例instance的uri替换原生的uri地址 得到 新的url
    // 新的url: http://localhost:8080/get/3
    URI requestUrl = loadBalancer.reconstructURI(
        new DelegatingServiceInstance(instance, overrideScheme), uri);
    if (log.isTraceEnabled()) {
        log.trace("LoadBalancerClientFilter url chosen: " + requestUrl);
    }
    // 5\. 再次记录上下文关系
    exchange.getAttributes().put(GATEWAY_REQUEST_URL_ATTR, requestUrl);
    // 6\. 执行过滤器链中的其他过滤请求
    return chain.filter(exchange);
}
```

# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning