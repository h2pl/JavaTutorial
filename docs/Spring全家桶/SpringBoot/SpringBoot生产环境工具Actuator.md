

# 生产就绪功能

[Back to index](https://springdoc.cn/spring-boot/index.html)

*   [1\. 启用生产就绪的功能](https://springdoc.cn/spring-boot/actuator.html#actuator.enabling)
*   [2\. 端点（Endpoint）](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints)
*   [3\. 通过HTTP进行监控和管理](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring)
*   [4\. 通过JMX进行监控和管理](https://springdoc.cn/spring-boot/actuator.html#actuator.jmx)
*   [5\. 可观测性（Observability）](https://springdoc.cn/spring-boot/actuator.html#actuator.observability)
*   [6\. 日志记录器（Logger）](https://springdoc.cn/spring-boot/actuator.html#actuator.loggers)
*   [7\. 指标（Metrics）](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics)
*   [8\. 追踪（Tracing）](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing)
*   [9\. 审计](https://springdoc.cn/spring-boot/actuator.html#actuator.auditing)
*   [10\. 记录 HTTP Exchange](https://springdoc.cn/spring-boot/actuator.html#actuator.http-exchanges)
*   [11\. 进程监控](https://springdoc.cn/spring-boot/actuator.html#actuator.process-monitoring)
*   [12\. Cloud Foundry 的支持](https://springdoc.cn/spring-boot/actuator.html#actuator.cloud-foundry)
*   [13\. 接下来读什么](https://springdoc.cn/spring-boot/actuator.html#actuator.whats-next)













|  | 本站([springdoc.cn](https://springdoc.cn/))中的内容来源于 [spring.io](https://spring.io/) ，原始版权归属于 [spring.io](https://spring.io/)。由 [springboot.io - Spring Boot中文社区](https://springboot.io/) 进行翻译，整理。可供个人学习、研究，未经许可，不得进行任何转载、商用或与之相关的行为。 商标声明：Spring 是 Pivotal Software, Inc. 在美国以及其他国家的商标。 |
| --- | --- |





Spring Boot包括一些额外的功能，以帮助你在将应用程序发布到生产时监控和管理你的应用程序。 你可以选择通过使用HTTP端点或使用JMX来管理和监控你的应用程序。 审计、健康和指标收集也可以自动应用于你的应用程序。









## [](https://springdoc.cn/spring-boot/actuator.html#actuator.enabling)1\. 启用生产就绪的功能





[`spring-boot-actuator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator) 模块提供了所有Spring Boot的生产就绪功能。 启用这些功能的推荐方法是添加对 `spring-boot-starter-actuator` “Starter” 的依赖。







Actuator的定义



actuator（执行器） 是一个制造术语，指的是用于移动或控制某物的机械装置。actuator 可以从一个小的变化中产生大量的运动。









要在基于Maven的项目中添加actuator，请添加以下 “Starter” 依赖。







```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-actuator
    </dependency>
</dependencies>
```







对于Gradle，使用以下声明。







```
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-actuator'
}
```











## [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints)2\. 端点（Endpoint）





Actuator 端点（endpoint）让你可以监控并与你的应用程序互动。 Spring Boot包括一些内置的端点，并允许你添加自己的端点。 例如，`health` 端点提供基本的应用程序健康信息。





你可以[启用或禁用](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.enabling)每个单独的端点，并[通过HTTP或JMX公开它们（使它们可以远程访问）](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.exposing)。当一个端点被启用和暴露时，它被认为是可用的。内置的端点只有在它们可用时才会被自动配置。大多数应用程序选择通过HTTP暴露，其中端点的ID和 `/actuator` 的前缀被映射到一个URL。例如，默认情况下，`health` 端点被映射到 `/actuator/health`。





|  | 要了解更多关于actuator的端点以及它们的请求和响应格式，请看单独的API文档（ [HTML](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/actuator-api/htmlsingle) 或 [PDF](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/actuator-api/pdf/spring-boot-actuator-web-api.pdf)）。 |
| --- | --- |





以下是技术无关的终端。



<colgroup><col><col></colgroup>
| ID | 说明 |
| --- | --- |
| `auditevents` | 公开当前应用程序的审计事件信息。 需要一个 `AuditEventRepository` bean。 |
| `beans` | 显示你的应用程序中所有Spring Bean的完整列表。 |
| `caches` | 显示可用的缓存。 |
| `conditions` | 显示对配置和自动配置类进行评估的条件，以及它们符合或不符合的原因。 |
| `configprops` | 显示所有 `@ConfigurationProperties` 的整理列表。 |
| `env` | 暴露Spring的 `ConfigurableEnvironment` 中的属性。 |
| `flyway` | 显示任何已经应用的Flyway数据库迁移。 需要一个或多个 `Flyway` bean。 |
| `health` | 显示应用程序的健康信息。 |
| `httpexchanges` | 显示 HTTP exchange 信息（默认情况下，最后 100 个 HTTP request/response exchange）。 需要一个 `HttpExchangeRepository` bean。 |
| `info` | 显示任意的应用程序信息。 |
| `integrationgraph` | 显示Spring集成图。 需要依赖 `spring-integration-core`。 |
| `loggers` | 显示和修改应用程序中logger的配置。 |
| `liquibase` | 显示任何已经应用的Liquibase数据库迁移。 需要一个或多个 `Liquibase` Bean。 |
| `metrics` | 显示当前应用程序的 “metrics” 信息。 |
| `mappings` | 显示所有 `@RequestMapping` 路径的整理列表。 |
| `quartz` | 显示有关Quartz Scheduler Job的信息。 |
| `scheduledtasks` | 显示你的应用程序中的计划任务。 |
| `sessions` | 允许从Spring Session支持的会话存储中检索和删除用户会话。 需要一个使用Spring Session的基于Servlet的Web应用程序。 |
| `shutdown` | 让应用程序优雅地关闭。只在使用jar打包时有效。默认情况下是禁用的。 |
| `startup` | 显示由 `ApplicationStartup` 收集的[启动步骤数据](https://springdoc.cn/spring-boot/features.html#features.spring-application.startup-tracking)。要求 `SpringApplication` 被配置为 `BufferingApplicationStartup`。 |
| `threaddump` | Performs a thread dump. |



如果你的应用程序是一个Web应用程序（Spring MVC、Spring WebFlux或Jersey），你可以使用以下额外的端点。



<colgroup><col><col></colgroup>
| ID | 说明 |
| --- | --- |
| `heapdump` | 返回一个堆dump文件。 在HotSpot JVM上，返回一个 `HPROF` 格式的文件。 在OpenJ9 JVM上，返回一个 `PHD` 格式的文件。 |
| `logfile` | 返回日志文件的内容（如果 `logging.file.name` 或 `logging.file.path` 属性已被设置）。 支持使用HTTP `Range` 头来检索日志文件的部分内容。 |
| `prometheus` | 以可被 Prometheus 服务器抓取的格式展示度量（metric）。 依赖于 `micrometer-registry-prometheus`。 |



### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.enabling)2.1\. 启用端点



默认情况下，除 `shutdown` 外的所有端点都被启用。 要配置一个端点的启用，请使用其 `management.endpoint.<id>.enabled` 属性。 下面的例子启用了 `shutdown` 端点。







Properties

Yaml





```
management.endpoint.shutdown.enabled=true

```







如果你希望端点启用是“选择启用”而不是“选择禁用”，请将 `management.endpoints.enabled-by-default` 属性设置为 `false`，并使用单个端点的 `enabled` 属性来选择重新启用。 下面的例子启用了 `info` 端点，并禁用了所有其他端点。







Properties

Yaml





```
management.endpoints.enabled-by-default=false
management.endpoint.info.enabled=true

```







|  | 被禁用的端点会从应用程序上下文中完全删除。如果你想只改变暴露端点的技术，请使用 [`include` 和 `exclude` 属性](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.exposing)来代替。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.exposing)2.2\. 暴露端点



默认情况下，只有health端点是通过HTTP和JMX暴露的。 由于端点可能包含敏感信息，你应该仔细考虑何时暴露它们。





要改变哪些端点被暴露，请使用以下特定技术的 `include` 和 `exclude` 属性。



<colgroup><col><col></colgroup>
| 属性 | 默认 |
| --- | --- |
| `management.endpoints.jmx.exposure.exclude` |  |
| `management.endpoints.jmx.exposure.include` | `health` |
| `management.endpoints.web.exposure.exclude` |  |
| `management.endpoints.web.exposure.include` | `health` |



`include` 属性列出了被暴露的端点的ID。 `exclude` 属性列出了不应该被公开的端点的ID。 `exclude` 属性优先于 `include` 属性。 你可以用一个端点ID列表来配置 `include` 和 `exclude` 属性。





例如，要通过JMX只公开 `health` 和 `info` 端点，请使用以下属性。







Properties

Yaml





```
management.endpoints.jmx.exposure.include=health,info

```







`*` 可以用来选择所有端点。 例如，要通过HTTP公开所有的东西，除了 `env` 和 `beans` 端点，使用以下属性。







Properties

Yaml





```
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=env,beans

```







|  | `*` 在YAML中具有特殊含义，所以如果你想包括（或排除）所有的端点，一定要加引号。 |
| --- | --- |





|  | 如果你的应用程序是公开暴露的，我们强烈建议你也[保护你的端点](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.security)。 |
| --- | --- |





|  | 如果你想在端点暴露时实施自己的策略，你可以注册一个 `EndpointFilter` bean。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.security)2.3\. 安全（Security）



为了安全起见，默认情况下只有 `/health` 端点通过HTTP公开。 你可以使用 `management.endpoints.web.exposure.include` 属性来配置被暴露的端点。





|  | 在设置 `management.endpoints.web.exposure.include` 之前，请确保暴露的执行器不包含敏感信息，将其置于防火墙之后，或由Spring Security之类的东西来保证安全。 |
| --- | --- |





如果Spring Security在classpath上，并且没有其他 `SecurityFilterChain` bean，那么除了 `/health` 之外的所有执行器（actuator）都由Spring Boot自动配置来保证安全。 如果你定义了一个自定义的 `SecurityFilterChain` bean，Spring Boot自动配置就会退缩，让你完全控制执行器的访问规则。





如果你想为HTTP端点配置自定义安全（例如，只允许具有某种角色的用户访问），Spring Boot提供了一些方便的 `RequestMatcher` 对象，你可以与Spring Security结合使用。





一个典型的Spring Security配置可能看起来像下面的例子。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MySecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher(EndpointRequest.toAnyEndpoint());
        http.authorizeHttpRequests((requests) -> requests.anyRequest().hasRole("ENDPOINT_ADMIN"));
        http.httpBasic(withDefaults());
        return http.build();
    }

}

```







前面的例子使用 `EndpointRequest.toAnyEndpoint()` 来匹配一个请求到任何端点，然后确保所有的端点都有 `ENDPOINT_ADMIN` 的角色。 `EndpointRequest` 上还有其他几个匹配器方法。 详情见API文档（ [HTML](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/actuator-api/htmlsingle) 或 [PDF](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/actuator-api/pdf/spring-boot-actuator-web-api.pdf)）。





如果你在防火墙后面部署应用程序，你可能希望你的所有执行器端点都能被访问，而不需要认证。 你可以通过改变 `management.endpoints.web.exposure.include` 属性来做到这一点，如下所示。







Properties

Yaml





```
management.endpoints.web.exposure.include=*

```







此外，如果存在Spring Security，你需要添加自定义安全配置，允许未经认证的访问端点，如下例所示。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MySecurityConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http.securityMatcher(EndpointRequest.toAnyEndpoint());
        http.authorizeHttpRequests((requests) -> requests.anyRequest().permitAll());
        return http.build();
    }

}

```







|  | 在前面的两个例子中，配置只适用于actuator端点。 由于Spring Boot的安全配置在有任何 `SecurityFilterChain` bean的情况下都会完全退出，所以你需要配置一个额外的 `SecurityFilterChain` bean，其规则适用于应用程序的其他部分。 |
| --- | --- |





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.security.csrf)2.3.1\. 跨网站请求伪造保护（CSRF）



由于Spring Boot依赖Spring Security的默认值，CSRF保护在默认情况下被打开。 这意味着在使用默认安全配置时，需要 `POST`（shutdown和loggers端点）、`PUT` 或 `DELETE` 的actuator端点会出现403（禁止）的错误。





|  | 我们建议只有在你创建的服务被非浏览器客户端使用时才完全禁用CSRF保护。 |
| --- | --- |





你可以在 [Spring安全参考指南](https://docs.spring.io/spring-security/reference/6.1.0-M1/features/exploits/csrf.html) 中找到关于CSRF保护的其他信息。









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.caching)2.4\. 配置端点



端点会自动缓存不需要任何参数的读取操作的响应。 要配置端点缓存响应的时间，请使用其 `cache.time-to-live` 属性。 下面的例子将 `beans` 端点的缓存生存时间设置为10秒。







Properties

Yaml





```
management.endpoint.beans.cache.time-to-live=10s

```







|  | `management.endpoint.<name>` 前缀唯一地标识了正在配置的端点。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.hypermedia)2.5\. 用于 Actuator Web 端点的超媒体（Hypermedia）



一个 “discovery page” 被添加到所有端点的链接中。 默认情况下，“discovery page” 在 `/actuator` 上是可用的。





要禁用 “discovery page”，请在你的应用程序属性中添加以下属性。







Properties

Yaml





```
management.endpoints.web.discovery.enabled=false

```







当配置了一个自定义的管理上下文路径时，“discovery page” 会自动从 `/actuator` 移到管理上下文的根部。 例如，如果管理上下文路径是 `/management`，discovery page可以从 `/management` 获得。 当管理上下文路径被设置为 `/` 时，发现页被禁用，以防止与其他mapping发生冲突的可能性。







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.cors)2.6\. CORS的支持



[跨源资源共享](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing)（CORS）是 [W3C的一个规范](https://www.w3.org/TR/cors/)，可以让你以灵活的方式指定哪种跨域请求被授权。如果你使用Spring MVC或Spring WebFlux，你可以配置Actuator的Web端点来支持这种情况。





CORS支持在默认情况下是禁用的，只有在你设置了 `management.endpoints.web.cors.allowed-origins` 属性后才会启用。 下面的配置允许从 `example.com` 域名中调用 `GET` 和 `POST`。







Properties

Yaml





```
management.endpoints.web.cors.allowed-origins=https://example.com
management.endpoints.web.cors.allowed-methods=GET,POST

```







|  | 参见 [`CorsEndpointProperties`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator-autoconfigure/src/main/java/org/springframework/boot/actuate/autoconfigure/endpoint/web/CorsEndpointProperties.java) 以获得完整的选项列表。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom)2.7\. 实现自定义端点



如果你添加了一个带有 `@Endpoint` 注解的 `@Bean`，任何带有 `@ReadOperation`、`@WriteOperation` 或 `@DeleteOperation` 注释的方法都会自动通过JMX公开，在Web应用程序中也会通过HTTP公开。 通过使用Jersey、Spring MVC或Spring WebFlux，端点可以通过HTTP暴露。 如果Jersey和Spring MVC都可用，则使用Spring MVC。





下面的例子暴露了一个读操作，它返回一个自定义对象。







Java

Kotlin





```
@ReadOperation
public CustomData getData() {
    return new CustomData("test", 5);
}

```







你也可以通过使用 `@JmxEndpoint` 或 `@WebEndpoint` 来编写特定技术的端点。 这些端点被限制在它们各自的技术上。 例如，`@WebEndpoint` 只通过HTTP暴露，而不是通过JMX。





你可以通过使用 `@EndpointWebExtension` 和 `@EndpointJmxExtension` 来编写特定的技术扩展。 这些注解让你提供特定技术的操作，以增强现有的端点。





最后，如果你需要访问Web框架的特定功能，你可以实现servlet或Spring的 `@Controller` 和 `@RestController` 端点，代价是它们不能通过JMX或使用不同的Web框架时可用。





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.input)2.7.1\. 接收输入



端点上的操作通过其参数接收输入。 当通过web公开时，这些参数的值来自URL的查询参数和JSON请求体。 当通过JMX公开时，参数被映射到MBean操作的参数中。 默认情况下，参数是必需的。 它们可以通过使用 `@javax.annotation.Nullable` 或 `@org.springframework.lang.Nullable` 进行注解而成为可选项。





你可以将JSON请求体中的每个根属性映射到端点的一个参数。 考虑一下下面的JSON请求体。







```
{
    "name": "test",
    "counter": 42
}
```







你可以用它来调用一个写操作，该操作需要 `String name` 和 `int counter` 参数，如下面的例子所示。







Java

Kotlin





```
@WriteOperation
public void updateData(String name, int counter) {
    // injects "test" and 42
}

```







|  | 因为端点是技术不可知的，在方法签名中只能指定简单的类型。 特别是，不支持用 `CustomData` 类型声明一个定义了 `name` 和 `counter` 属性的单一参数。 |
| --- | --- |





|  | 为了让输入映射到操作方法的参数，实现端点的Java代码应该用 `-parameters` 编译，而实现端点的Kotlin代码应该用 `-java-parameters` 编译。 如果你使用Spring Boot的Gradle插件或使用Maven和 `spring-boot-starter-parent`，这将自动发生。 |
| --- | --- |





##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.input.conversion)输入类型转换



如果有必要，传递给端点操作方法的参数会自动转换为所需类型。 在调用操作方法之前，通过JMX或HTTP收到的输入被转换为所需的类型，方法是使用 `ApplicationConversionService` 的实例以及任何 `Converter` 或 `GenericConverter` Bean，并以 `@EndpointConverter` 限定。









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web)2.7.2\. 自定义WEB端点



对 `@Endpoint`、`@WebEndpoint` 或 `@EndpointWebExtension` 操作会自动使用Jersey、Spring MVC或Spring WebFlux通过HTTP公开。 如果Jersey和Spring MVC都可用，则使用Spring MVC。





##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.request-predicates)WEB端点请求谓词（Predicates）



一个请求谓词会为web暴露的端点上的每个操作（operation）自动生成。







##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.path-predicates)Path



path谓词由端点的ID和网络暴露的端点的基本路径决定。 默认的基本路径是 `/actuator`。 例如，一个ID为 `sessions` 的端点在谓词中使用 `/actuator/sessions` 作为其路径。





你可以通过用 `@Selector` 注解操作方法的一个或多个参数来进一步定制路径。 这样的参数会作为一个路径变量添加到路径谓词中。 在调用端点操作时，该变量的值会被传入操作方法。 如果你想捕获所有剩余的路径元素，你可以在最后一个参数上添加 `@Selector(Match=ALL_REMAINING)`，并使其成为一个与 `String[]` 转换兼容的类型。







##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.method-predicates)HTTP method（方法）



HTTP method谓词是由操作类型决定的，如下表所示。



<colgroup><col><col></colgroup>
| Operation | HTTP method |
| --- | --- |
| `@ReadOperation` | `GET` |
| `@WriteOperation` | `POST` |
| `@DeleteOperation` | `DELETE` |





##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.consumes-predicates)Consumes



对于使用request body的 `@WriteOperation`（HTTP `POST`），谓词的 `consumes` 子句是 `application/vnd.spring-boot.actuator.v2+json, application/json`。 对于所有其他操作，`consumes` 子句是空的。







##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.produces-predicates)Produces



谓词的 `produces` 子句可以由 `@DeleteOperation`、`@ReadOperation` 和 `@WriteOperation` 注释的 `produces` 属性决定。 该属性是可选的。 如果不使用它，`produces` 子句会自动确定。





如果操作方法返回 `void` 或 `Void`，则 `produces` 子句为空。 如果操作方法返回 `org.springframework.core.io.Resource`，`produces` 子句是 `application/octet-stream`。 对于所有其他操作，`produces` 子句是 `application/vnd.spring-boot.actuator.v2+json, application/json`。







##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.response-status)WEB端点响应状态



端点操作的默认响应状态取决于操作类型（读、写或删除）和操作返回的内容（如果有的话）。





如果 `@ReadOperation` 返回一个值，响应状态将是200(Ok)。 如果它没有返回一个值，响应状态将是404(Not Found)。





如果 `@WriteOperation` 或 `@DeleteOperation` 返回一个值，响应状态将是200（OK）。 如果它没有返回一个值，响应状态将是204（No Content）。





如果一个操作在调用时没有所需的参数，或者参数不能被转换为所需的类型，操作方法就不会被调用，响应状态将是400（Bad Request）。







##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.range-requests)WEB端点 Range 请求



你可以使用HTTP range请求来请求一个HTTP资源的一部分。 当使用Spring MVC或Spring Web Flux时，返回 `org.springframework.core.io.Resource` 的操作自动支持范围请求。





|  | 使用Jersey时不支持 Range 请求。 |
| --- | --- |







##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.web.security)Web端点的安全



在web端点或web特定端点扩展上的操作可以接收当前的 `java.security.Principal` 或 `org.springframework.boot.actuate.endpoint.SecurityContext` 作为方法参数。 前者通常与 `@Nullable` 一起使用，为已认证和未认证的用户提供不同的行为。 后者通常用于通过使用其 `isUserInRole(String)` 方法来执行授权检查。









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.servlet)2.7.3\. Servlet 端点



一个Servlet可以作为一个端点暴露出来，方法是实现一个用 `@ServletEndpoint` 注解的类，同时实现 `Supplier<EndpointServlet>`。 Servlet端点提供了与servlet容器的更深层次的整合，但却牺牲了可移植性。 它们的目的是用来将现有的Servlet作为一个端点来公开。 对于新的端点，应尽可能选择 `@Endpoint` 和 `@WebEndpoint` 注解。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.implementing-custom.controller)2.7.4\. Controller 端点



你可以使用 `@ControllerEndpoint` 和 `@RestControllerEndpoint` 来实现一个仅由Spring MVC或Spring WebFlux公开的端点。 方法通过使用Spring MVC和Spring WebFlux的标准注解进行映射，如 `@RequestMapping` 和 `@GetMapping`，端点的ID被用作路径的前缀。 控制器端点提供了与Spring的Web框架更深入的集成，但却牺牲了可移植性。 应尽可能选择 `@Endpoint` 和 `@WebEndpoint` 注解。









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health)2.8\. 健康信息



你可以使用健康信息来检查你运行的应用程序的状态。 它经常被监控软件用来在生产系统发生故障时提醒别人。 `health` 端点暴露的信息取决于 `management.endpoint.health.show-details` 和 `management.endpoint.health.show-components` 属性，它们可以配置为以下值之一。



<colgroup><col><col></colgroup>
| 值 | 说明 |
| --- | --- |
| `never` | 细节从不显示。 |
| `when-authorized` | 细节只显示给授权用户。 授权的角色可以通过使用 `management.endpoint.health.roles` 进行配置。 |
| `always` | 详情显示给所有用户。 |



默认值是 `never`。 当用户处于端点的一个或多个角色中时，他们被认为是被授权的。 如果端点没有配置角色（默认值），所有认证的用户都被认为是授权的。 你可以通过使用 `management.endpoint.health.roles` 属性来配置角色。





|  | 如果你已经保护了你的应用程序并希望使用 `always`，你的安全配置（security configuration）必须允许认证和非认证用户访问health端点。 |
| --- | --- |





健康信息是从 [`HealthContributorRegistry`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthContributorRegistry.java) 的内容中收集的（默认情况下，所有 [`HealthContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthContributor.java) 实例都定义在你的 `ApplicationContext` 中）。 Spring Boot包括一些自动配置的 `HealthContributor`，你也可以编写自己的。





一个 `HealthContributor` 可以是一个 `HealthIndicator` 或一个 `CompositeHealthContributor`。 一个 `HealthIndicator` 提供实际的健康信息，包括 `Status`。 一个 `CompositeHealthContributor` 提供其他 `HealthContributors` 的组合。 综合起来，contributor形成一个树状结构来表示整个系统的健康状况。





默认情况下，最终的系统健康状况是由一个 `StatusAggregator` 得出的，它根据一个有序的状态列表对每个 `HealthIndicator` 的状态进行排序。 排序列表中的第一个状态被用作整体健康状态。 如果没有 `HealthIndicator` 返回的状态是 `StatusAggregator` 所知道的，就会使用 `UNKNOWN` 状态。





|  | 你可以使用 `HealthContributorRegistry` 来在运行时注册和取消注册健康指标。 |
| --- | --- |





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.auto-configured-health-indicators)2.8.1\. 自动配置的HealthIndicators



在适当的时候，Spring Boot会自动配置下表中列出的 `HealthIndicators`。 你也可以通过配置 `management.health.key.enabled` 来启用或停用所选指标。 和下表中列出的 `key`。



<colgroup><col><col><col></colgroup>
| Key | Name | 说明 |
| --- | --- | --- |
| `cassandra` | [`CassandraDriverHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/cassandra/CassandraDriverHealthIndicator.java) | 检查Cassandra数据库是否已经启动。 |
| `couchbase` | [`CouchbaseHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/couchbase/CouchbaseHealthIndicator.java) | 检查Couchbase集群是否已经启动。 |
| `db` | [`DataSourceHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/jdbc/DataSourceHealthIndicator.java) | 检查是否可以获得与`DataSource`的连接。 |
| `diskspace` | [`DiskSpaceHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/system/DiskSpaceHealthIndicator.java) | 检查磁盘空间是否不足。 |
| `elasticsearch` | [`ElasticsearchRestHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/elasticsearch/ElasticsearchRestHealthIndicator.java) | 检查Elasticsearch集群是否已经启动。 |
| `hazelcast` | [`HazelcastHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/hazelcast/HazelcastHealthIndicator.java) | 检查Hazelcast服务器是否已经启动。 |
| `influxdb` | [`InfluxDbHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/influx/InfluxDbHealthIndicator.java) | 检查InfluxDB服务器是否已经启动。 |
| `jms` | [`JmsHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/jms/JmsHealthIndicator.java) | 检查一个JMS代理是否已经启动。 |
| `ldap` | [`LdapHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/ldap/LdapHealthIndicator.java) | 检查一个LDAP服务器是否正常。 |
| `mail` | [`MailHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/mail/MailHealthIndicator.java) | 检查一个邮件服务器是否正常。 |
| `mongo` | [`MongoHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/data/mongo/MongoHealthIndicator.java) | 检查Mongo数据库是否已经启动。 |
| `neo4j` | [`Neo4jHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/neo4j/Neo4jHealthIndicator.java) | 检查Neo4j数据库是否已经启动。 |
| `ping` | [`PingHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/PingHealthIndicator.java) | 总是响应 `UP` 。 |
| `rabbit` | [`RabbitHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/amqp/RabbitHealthIndicator.java) | 检查一个Rabbit服务器是否已经启动。 |
| `redis` | [`RedisHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/data/redis/RedisHealthIndicator.java) | 检查Redis服务器是否已经启动。 |



|  | 你可以通过设置 `management.health.defaults.enabled` 属性来禁用它们。 |
| --- | --- |





额外的 `HealthIndicators` 是可用的，但在默认情况下不启用。



<colgroup><col><col><col></colgroup>
| Key | Name | 说明 |
| --- | --- | --- |
| `livenessstate` | [`LivenessStateHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/availability/LivenessStateHealthIndicator.java) | 显示 “Liveness” 应用程序的可用性状态。 |
| `readinessstate` | [`ReadinessStateHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/availability/ReadinessStateHealthIndicator.java) | 暴露 “Readiness” 应用程序的可用性状态。 |





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.writing-custom-health-indicators)2.8.2\. 编写自定义HealthIndicators



为了提供自定义的健康信息，你可以注册实现 [`HealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthIndicator.java) 接口的Spring Bean。 你需要提供一个 `health()` 方法的实现，并返回一个 `Health` 响应。 `Health` 响应应该包括一个status，并可以选择包括要显示的其他细节。 下面的代码显示了一个 `HealthIndicator` 的实现样本。







Java

Kotlin





```
@Component
public class MyHealthIndicator implements HealthIndicator {

    @Override
    public Health health() {
        int errorCode = check();
        if (errorCode != 0) {
            return Health.down().withDetail("Error Code", errorCode).build();
        }
        return Health.up().build();
    }

    private int check() {
        // perform some specific health check
        return ...
    }

}

```







|  | 一个给定的 `HealthIndicator` 的标识符（ID）是没有 `HealthIndicator` 后缀的Bean的名字，如果它存在的话。 在前面的例子中，健康信息可以在一个名为 `my` 的条目中找到。 |
| --- | --- |





|  | 健康指标通常是通过HTTP调用的，需要在任何连接超时之前做出响应。 如果任何健康指标的响应时间超过10秒，Spring Boot将记录一条警告信息。 如果你想配置这个阈值，你可以使用 `management.endpoint.health.logging.slow-indicator-threshold` 属性。 |
| --- | --- |





除了Spring Boot预定义的 [`Status`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/Status.java) 类型外，`Health` 可以返回代表新系统状态的自定义 `Status`。 在这种情况下，你还需要提供 `StatusAggregator` 接口的自定义实现，或者你必须通过使用 `management.endpoint.health.status.order` 配置属性来配置默认实现。





例如，假设在你的一个 `HealthIndicator` 实现中使用了一个代码为 `FATAL` 的新 `Status`。 为了配置严重性顺序，在你的应用程序属性中添加以下属性。







Properties

Yaml





```
management.endpoint.health.status.order=fatal,down,out-of-service,unknown,up

```







响应中的HTTP状态代码反映了整体健康状态。 默认情况下，`OUT_OF_SERVICE` 和 `DOWN` 映射到503。 任何未映射的健康状态，包括 `UP`，都映射为200。 如果你通过HTTP访问健康端点，你可能还想注册自定义状态映射。 配置自定义映射会禁用 `DOWN` 和 `OUT_OF_SERVICE` 的默认映射。 如果你想保留默认映射，你必须明确地配置它们，以及任何自定义映射。 例如，下面的属性将 `FATAL` 映射为503（服务不可用），并保留了 `DOWN` 和 `OUT_OF_SERVICE` 的默认映射。







Properties

Yaml





```
management.endpoint.health.status.http-mapping.down=503
management.endpoint.health.status.http-mapping.fatal=503
management.endpoint.health.status.http-mapping.out-of-service=503

```







|  | 如果你需要更多的控制，你可以定义你自己的 `HttpCodeStatusMapper` bean。 |
| --- | --- |





下表显示了内置状态的默认状态映射。



<colgroup><col><col></colgroup>
| Status | Mapping |
| --- | --- |
| `DOWN` | `SERVICE_UNAVAILABLE` (`503`) |
| `OUT_OF_SERVICE` | `SERVICE_UNAVAILABLE` (`503`) |
| `UP` | 默认情况下没有映射，所以HTTP状态为 `200`。 |
| `UNKNOWN` | 默认情况下没有映射，所以HTTP状态为 `200`。 |





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.reactive-health-indicators)2.8.3\. 响应式健康指标



对于响应式应用程序，例如那些使用Spring WebFlux的应用程序，`ReactiveHealthContributor` 提供了一个非阻塞的契约来获取应用程序的健康状况。 与传统的 `HealthContributor` 类似，健康信息从 [`ReactiveHealthContributorRegistry`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/ReactiveHealthContributorRegistry.java) 的内容中收集（默认情况下，所有 [`HealthContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/HealthContributor.java) 和 [`ReactiveHealthContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/ReactiveHealthContributor.java) 的实例定义在你的 `ApplicationContext` 里）。





不对响应式API进行检查的常规 `HealthContributors` 在弹性调度器上执行。





|  | 在一个响应式应用程序中，你应该使用 `ReactiveHealthContributorRegistry` 来在运行时注册和取消注册健康指标。 如果你需要注册一个普通的 `HealthContributor`，你应该用 `ReactiveHealthContributor#adapt` 来包装它。 |
| --- | --- |





为了从响应式API中提供自定义的健康信息，你可以注册实现 [`ReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/health/ReactiveHealthIndicator.java) 接口的Spring Bean。 下面的代码显示了一个 `ReactiveHealthIndicator` 的示例实现。







Java

Kotlin





```
@Component
public class MyReactiveHealthIndicator implements ReactiveHealthIndicator {

    @Override
    public Mono<Health> health() {
        return doHealthCheck().onErrorResume((exception) ->
            Mono.just(new Health.Builder().down(exception).build()));
    }

    private Mono<Health> doHealthCheck() {
        // perform some specific health check
        return ...
    }

}

```







|  | 为了自动处理错误，可以考虑从 `AbstractReactiveHealthIndicator` 中扩展。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.auto-configured-reactive-health-indicators)2.8.4\. 自动配置的 ReactiveHealthIndicators



在适当的时候，Spring Boot会自动配置以下的 `ReactiveHealthIndicators`。



<colgroup><col><col><col></colgroup>
| Key | Name | 说明 |
| --- | --- | --- |
| `cassandra` | [`CassandraDriverReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/cassandra/CassandraDriverReactiveHealthIndicator.java) | 检查Cassandra数据库是否已经启动。 |
| `couchbase` | [`CouchbaseReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/couchbase/CouchbaseReactiveHealthIndicator.java) | 检查Couchbase集群是否已经启动。 |
| `elasticsearch` | [`ElasticsearchReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/data/elasticsearch/ElasticsearchReactiveHealthIndicator.java) | 检查Elasticsearch集群是否已经启动。 |
| `mongo` | [`MongoReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/data/mongo/MongoReactiveHealthIndicator.java) | 检查Mongo数据库是否已经启动。 |
| `neo4j` | [`Neo4jReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/neo4j/Neo4jReactiveHealthIndicator.java) | 检查Neo4j数据库是否已经启动。 |
| `redis` | [`RedisReactiveHealthIndicator`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/data/redis/RedisReactiveHealthIndicator.java) | 检查Redis服务器是否已经启动。 |



|  | 如果有必要，响应式指标会取代常规指标。 另外，任何没有被明确处理的 `HealthIndicator` 都会被自动包装起来。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.groups)2.8.5\. Health分组（Health Groups）



有时，将健康指标组织成可以用于不同目的的组是很有用的。





要创建一个健康指标组，你可以使用 `management.endpoint.health.group.<name>` 属性，并指定一个健康指标ID列表来 `include` 或 `exclude`。 例如，要创建一个只包括数据库指标的组，你可以定义如下。







Properties

Yaml





```
management.endpoint.health.group.custom.include=db

```







然后你可以通过点击 `[localhost:8080/actuator/health/custom](http://localhost:8080/actuator/health/custom)` 来检查结果。





同样，要创建一个组，将数据库指标排除在该组之外，并包括所有其他指标，你可以定义如下。







Properties

Yaml





```
management.endpoint.health.group.custom.exclude=db

```







默认情况下，组继承了与系统健康相同的 `StatusAggregator` 和 `HttpCodeStatusMapper` 设置。 然而，你也可以在每个组的基础上定义这些。 如果需要，你也可以覆盖 `show-details` 和 `roles` 属性。







Properties

Yaml





```
management.endpoint.health.group.custom.show-details=when-authorized
management.endpoint.health.group.custom.roles=admin
management.endpoint.health.group.custom.status.order=fatal,up
management.endpoint.health.group.custom.status.http-mapping.fatal=500
management.endpoint.health.group.custom.status.http-mapping.out-of-service=500

```







|  | 如果你需要注册自定义的 `StatusAggregator` 或 `HttpCodeStatusMapper` Bean以用于组，你可以使用 `@Qualifier("groupname")`。 |
| --- | --- |





一个健康组也可以包括/排除一个 `CompositeHealthContributor`。 您也可以只包括/排除一个 `CompositeHealthContributor` 的某个组件。 这可以使用组件的完全名称来完成，如下所示。







```
management.endpoint.health.group.custom.include="test/primary"
management.endpoint.health.group.custom.exclude="test/primary/b"
```







在上面的例子中，`custom` 组将包括名称为 `primary` 的 `HealthContributor`，它是复合 `test` 的一个组成部分。 在这里，`primary` 本身就是一个复合体，名字为 `b` 的 `HealthContributor` 将被排除在 `custom` 组之外。





健康组可以在主端口或管理端口的额外路径上提供。 这在Kubernetes等云环境中很有用，在这些环境中，出于安全考虑，为执行器端点使用一个单独的管理端口是很常见的。 有一个单独的端口可能导致不可靠的健康检查，因为即使健康检查成功，主应用程序也可能无法正常工作。 健康组可以用一个额外的路径进行配置，如下所示。







```
management.endpoint.health.group.live.additional-path="server:/healthz"
```







这将使 `live` 健康组在主服务器端口 `/healthz` 上可用。 前缀是强制性的，必须是 `server:`（代表主服务器端口）或 `management:`（代表管理端口，如果已配置）。 路径必须是一个单一的路径段。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.datasource)2.8.6\. 数据源健康



`DataSource` 健康指标显示标准数据源和路由数据源Bean的健康状况。 路由数据源的健康状况包括其每个目标数据源的健康状况。 在健康端点的响应中，路由数据源的每个目标都是通过使用其路由键来命名的。 如果你不希望在指标的输出中包括路由数据源，请将 `management.health.db.ignore-routing-data-sources` 设置为 `true`。









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.kubernetes-probes)2.9\. Kubernetes 探针



部署在Kubernetes上的应用程序可以通过 [容器探针](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes) 提供有关其内部状态的信息。根据 [你的Kubernetes配置](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)，kubelet会调用这些探针并对结果做出反应。





默认情况下，Spring Boot会管理你的[应用可用性状态](https://springdoc.cn/spring-boot/features.html#features.spring-application.application-availability)。 如果部署在Kubernetes环境中，actuator从 `ApplicationAvailability` 接口中收集 “Liveness” 和 “Readiness” 信息，并在专用[健康指标](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.auto-configured-health-indicators)中使用这些信息。`LivenessStateHealthIndicator` 和 `ReadinessStateHealthIndicator`。 这些指标显示在全局健康端点（`"/actuator/health"`）。 它们也可以通过使用[健康组](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.groups)作为单独的HTTP探针：`"/actuator/health/liveness"` 和 `"/actuator/health/readiness"`。





然后，你可以用以下端点信息配置你的Kubernetes基础设施。







```
livenessProbe:
  httpGet:
    path: "/actuator/health/liveness"
    port: 
  failureThreshold: ...
  periodSeconds: ...

readinessProbe:
  httpGet:
    path: "/actuator/health/readiness"
    port: 
  failureThreshold: ...
  periodSeconds: ...
```







|  | `` 应该被设置为执行器端点可用的端口。 它可以是主Web服务器的端口，也可以是一个单独的管理端口，如果 `"management.server.port"` 属性已经被设置。 |
| --- | --- |





只有当应用程序[在Kubernetes环境中运行时](https://springdoc.cn/spring-boot/deployment.html#deployment.cloud.kubernetes)，这些健康组才会自动启用。 你可以通过使用 `management.endpoint.health.probes.enabled` 配置属性在任何环境中启用它们。





|  | 如果一个应用程序的启动时间超过了配置的有效期，Kubernetes 会提到 `"startupProbe"` 作为一个可能的解决方案。一般来说，这里不一定需要 `"startupProbe"`，因为 `"readinessProbe"` 会在所有启动任务完成之前失效。这意味着你的应用程序在准备好之前不会收到流量。然而，如果你的应用程序需要很长时间才能启动，可以考虑使用 `"startupProbe"` 来确保Kubernetes不会在你的应用程序启动过程中杀死它。请参阅描述[探针在应用程序生命周期中的行为](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.kubernetes-probes.lifecycle)的部分。 |
| --- | --- |





如果你的Actuator端点被部署在一个单独的管理上下文中，那么这些端点就不会像主程序那样使用相同的网络基础设施（端口、连接池、框架组件）。 在这种情况下，即使主程序不能正常工作（例如，它不能接受新的连接），探测检查也可能成功。 由于这个原因，在主服务器端口上设置 `liveness` 和 `readiness` 健康组是个好主意。 这可以通过设置以下属性来实现。







```
management.endpoint.health.probes.add-additional-paths=true
```







这将使 `liveness` 在 `/livez` 可用，`readiness` 在 `readyz` 的主服务器端口可用。





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.kubernetes-probes.external-state)2.9.1\. 用Kubernetes探针检查外部状态



执行器将 “liveness” 和 “readiness” 探针配置为健康组。这意味着所有的[健康组的功能](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.health.groups)对它们都是可用的。例如，你可以配置额外的健康指标。







Properties

Yaml





```
management.endpoint.health.group.readiness.include=readinessState,customCheck

```







默认情况下，Spring Boot不会向这些组添加其他健康指标。





“liveness” 探针不应该依赖于外部系统的健康检查。如果[应用程序的有效性状态](https://springdoc.cn/spring-boot/features.html#features.spring-application.application-availability.liveness)被破坏，Kubernetes会尝试通过重新启动应用程序实例来解决这个问题。这意味着，如果一个外部系统（如数据库、Web API或外部缓存）出现故障，Kubernetes可能会重新启动所有应用程序实例，并产生级联故障。





至于 “readiness” 探测，检查外部系统的选择必须由应用程序开发人员谨慎作出。出于这个原因，Spring Boot在准备状态探测中不包括任何额外的健康检查。如果[应用程序实例的readiness state是unready](https://springdoc.cn/spring-boot/features.html#features.spring-application.application-availability.readiness)，Kubernetes就不会将流量路由到该实例。一些外部系统可能不被应用实例所共享，在这种情况下，它们可以被包括在准备状态探测中。其他外部系统可能不是应用程序的关键（应用程序可能有断路器和回退），在这种情况下，它们绝对不应该被包括在内。不幸的是，一个被所有应用实例共享的外部系统很常见，你必须做出判断。把它包括在准备就绪探针中，并期望当外部服务发生故障时，应用程序会被停止服务，或者不包括它，并处理堆栈中更高层次的故障，也许通过在调用者中使用断路器。





|  | 如果一个应用程序的所有实例都没有准备好，`type=ClusterIP` 或 `NodePort` 的Kubernetes服务不接受任何传入连接。没有HTTP错误响应（503等），因为没有连接。`type=LoadBalancer` 的服务可能接受也可能不接受连接，这取决于提供者。一个有明确 [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) 的服务也会以一种取决于实现的方式进行响应—?入口服务本身必须决定如何处理来自下游的 “connection refused”。在负载均衡器和入口的情况下，HTTP 503是很有可能的。 |
| --- | --- |





另外，如果一个应用程序使用 Kubernetes [autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)，它可能会对应用程序被从负载平衡器中取出做出不同的反应，这取决于其autoscaler的配置。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.kubernetes-probes.lifecycle)2.9.2\. 应用程序生命周期和探针状态



Kubernetes Probes支持的一个重要方面是它与应用程序生命周期的一致性。 `AvailabilityState`（即应用程序的内存内部状态）和实际的探针（暴露该状态）之间存在着显著的区别。 和实际的探针（暴露了该状态）之间有很大的区别。 根据应用程序生命周期的不同阶段，探针可能无法使用。





Spring Boot在[启动和关闭期间发布application event](https://springdoc.cn/spring-boot/features.html#features.spring-application.application-events-and-listeners)，探针可以监听这些事件并暴露出 `AvailabilityState` 信息。





下表显示了不同阶段的 `AvailabilityState` 和HTTP connector的状态。





当一个Spring Boot应用程序启动时。



<colgroup><col><col><col><col><col></colgroup>
| 启动阶段 | LivenessState | ReadinessState | HTTP server | 备注 |
| --- | --- | --- | --- | --- |
| Starting | `BROKEN` | `REFUSING_TRAFFIC` | 未启动 | Kubernetes检查 "liveness" 探针，如果时间过长，就重新启动应用程序。 |
| Started | `CORRECT` | `REFUSING_TRAFFIC` | 拒绝请求 | 应用程序上下文被刷新。应用程序执行启动任务，还没有收到流量。 |
| Ready | `CORRECT` | `ACCEPTING_TRAFFIC` | 接受请求 | 启动任务已经完成。该应用程序正在接收流量。 |



当一个Spring Boot应用程序关闭时。



<colgroup><col><col><col><col><col></colgroup>
| 停机阶段 | Liveness State | Readiness State | HTTP server | 备注 |
| --- | --- | --- | --- | --- |
| Running | `CORRECT` | `ACCEPTING_TRAFFIC` | 接受请求 | 已要求关闭。 |
| Graceful shutdown | `CORRECT` | `REFUSING_TRAFFIC` | 新的请求被拒绝 | 如果启用， [优雅关机会处理“处理中”的请求](https://springdoc.cn/spring-boot/web.html#web.graceful-shutdown)。 |
| Shutdown complete | N/A | N/A | 服务器被关闭 | 应用程序上下文被关闭，应用程序被关闭。 |



|  | 关于Kubernetes部署的更多信息，请参见[Kubernetes容器生命周期](https://springdoc.cn/spring-boot/deployment.html#deployment.cloud.kubernetes.container-lifecycle)部分。 |
| --- | --- |









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info)2.10\. 应用信息



应用程序信息公开了从你的 `ApplicationContext` 中定义的所有 [`InfoContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/InfoContributor.java) Bean收集的各种信息。 Spring Boot包括一些自动配置的 `InfoContributor` Bean，你也可以编写自己的。





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info.auto-configured-info-contributors)2.10.1\. 自动配置的 InfoContributor



在适当的时候，Spring会自动配置以下 `InfoContributor` Bean。



<colgroup><col><col><col><col></colgroup>
| ID | Name | 说明 | 前提条件 |
| --- | --- | --- | --- |
| `build` | [`BuildInfoContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/BuildInfoContributor.java) | 暴露了构建信息。 | 一个 `META-INF/build-info.properties` 资源。 |
| `env` | [`EnvironmentInfoContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/EnvironmentInfoContributor.java) | 暴露 `Environment` 中名称以 `info.` 开头的任何属性。 | None. |
| `git` | [`GitInfoContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/GitInfoContributor.java) | 暴露了git信息。 | 一个 `git.properties` 资源。 |
| `java` | [`JavaInfoContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/JavaInfoContributor.java) | 暴露Java运行时（Runtime）信息。 | None. |
| `os` | [`OsInfoContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/OsInfoContributor.java) | 暴露操作系统信息。 | None. |



个人贡献者（contributor）是否被启用是由其 `management.info.<id>.enabled` 属性控制。 不同的contributor对这个属性有不同的默认值，这取决于他们的先决条件和他们所暴露的信息的性质。





由于没有先决条件表明它们应该被启用，`env`、`java` 和 `os` contributor 默认是禁用的。 可以通过设置 `management.info.<id>.enabled` 属性为 `true` 来启用它们。





`build` 和 `git` 信息contributor默认是启用的。 可以通过将其 `management.info.<id>.enabled` 属性设置为 `false` 来禁用。 另外，要禁用每一个默认启用的contributor，请将 `management.info.defaults.enabled` 属性设为 `false`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info.custom-application-information)2.10.2\. 自定义应用信息（Application Information）



当 `env` contributor 被启用时，你可以通过设置 `info.*` Spring属性来定制 `info` 端点所暴露的数据。 `info` key下的所有 `Environment` 属性都会自动暴露。 例如，你可以在你的 `application.properties` 文件中添加以下设置。







Properties

Yaml





```
info.app.encoding=UTF-8
info.app.java.source=17
info.app.java.target=17

```







|  | 与其硬编码这些值，你还可以 [在构建时扩展信息属性](https://springdoc.cn/spring-boot/howto.html#howto.properties-and-configuration.expand-properties)。假设你使用Maven，你可以将前面的例子改写如下。PropertiesYaml```info.app.encoding=@project.build.sourceEncoding@info.app.java.source=@java.version@info.app.java.target=@java.version@``` |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info.git-commit-information)2.10.3\. Git Commit 信息



`info` 端点的另一个有用的功能是它能够发布关于你的 `git` 源代码库在项目建立时的状态的信息。 如果有一个 `GitProperties` bean，你可以使用 `info` 端点来公布这些属性。





|  | 如果classpath的根部有 `git.properties` 文件，`GitProperties` Bean就会被自动配置。更多细节见 "[如何生成git信息](https://springdoc.cn/spring-boot/howto.html#howto.build.generate-git-info)"。 |
| --- | --- |





默认情况下，端点会暴露 `git.branch`、`git.commit.id` 和 `git.commit.time` 属性（如果存在）。 如果你不想让这些属性出现在端点响应中，需要从 `git.properties` 文件中排除它们。 如果你想显示完整的git信息（即 `git.properties` 的全部内容），使用 `management.info.git.mode` 属性，如下所示。







Properties

Yaml





```
management.info.git.mode=full

```







要从 `info` 端点完全禁用git提交信息，请 `management.info.git.enabled` 属性设为 `false`，如下所示。







Properties

Yaml





```
management.info.git.enabled=false

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info.build-information)2.10.4\. 构建信息



如果 `BuildProperties` Bean是可用的，`info` 端点也可以发布关于你的构建信息。如果classpath中的 `META-INF/build-info.properties` 文件可用，就会发生这种情况。





|  | Maven和Gradle插件都可以生成该文件。详见 "[如何生成构建信息](https://springdoc.cn/spring-boot/howto.html#howto.build.generate-info)"。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info.java-information)2.10.5\. Java信息



`info` 端点发布了关于你的Java运行环境的信息，更多细节见 [`JavaInfo`](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/api/org/springframework/boot/info/JavaInfo.html)。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info.os-information)2.10.6\. 操作系统（OS）信息



`info` 端点发布关于你的操作系统的信息，更多细节见 [`OsInfo`](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/api/org/springframework/boot/info/OsInfo.html)`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.info.writing-custom-info-contributors)2.10.7\. 编写自定义 InfoContributor



为了提供自定义的应用程序信息，你可以注册实现 [`InfoContributor`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-actuator/src/main/java/org/springframework/boot/actuate/info/InfoContributor.java) 接口的Spring Bean。





下面的例子贡献了一个只有一个值的 `example` 条目。







Java

Kotlin





```
@Component
public class MyInfoContributor implements InfoContributor {

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("example", Collections.singletonMap("key", "value"));
    }

}

```







如果你请求 `info` 端点，你应该看到一个包含以下额外条目的响应。







```
{
    "example": {
        "key" : "value"
    }
}
```















## [](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring)3\. 通过HTTP进行监控和管理





如果你正在开发一个Web应用程序，Spring Boot Actuator会自动配置所有启用的端点，使其通过HTTP公开。 默认的惯例是使用端点的 `id` 和 `/actuator` 的前缀作为URL路径。 例如，`health` 以 `/actuator/health` 的形式公开。





|  | Actuator 支持 Spring MVC、Spring WebFlux和Jersey。 如果Jersey和Spring MVC都可用，则会使用Spring MVC。 |
| --- | --- |





|  | 为了获得API文档（ [HTML](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/actuator-api/htmlsingle) 或 [PDF](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/actuator-api/pdf/spring-boot-actuator-web-api.pdf) ）中记载的正确的JSON响应，Jackson是一个必要的依赖。 |
| --- | --- |





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring.customizing-management-server-context-path)3.1\. 定制管理端点路径



有时，为管理端点定制前缀是很有用的。 例如，你的应用程序可能已经将 `/actuator` 用于其他目的。 你可以使用 `management.endpoints.web.base-path` 属性来改变管理端点的前缀，如下面的例子所示。







Properties

Yaml





```
management.endpoints.web.base-path=/manage

```







前面的 `application.properties` 例子将端点从 `/actuator/{id}` 改为 `/manage/{id}` （例如，`/manage/info`）。





|  | 除非管理端口被配置为[使用不同的HTTP端口](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring.customizing-management-server-port)来暴露端点，否则 `management.endpoints.web.base-path` 是相对于 `server.servlet.context-path` （用于Servlet Web应用）或 `spring.webflux.base-path` （用于reactive Web应用）。如果配置了 `management.server.port`， `management.endpoints.web.base-path` 是相对于 `management.server.base-path` 的。 |
| --- | --- |





如果你想把端点映射到不同的路径，你可以使用 `management.endpoints.web.path-mapping` 属性。





下面的例子将 `/actuator/health` 重新映射为 `/healthcheck`。







Properties

Yaml





```
management.endpoints.web.base-path=/
management.endpoints.web.path-mapping.health=healthcheck

```









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring.customizing-management-server-port)3.2\. 定制管理服务器端口



对于基于云的部署来说，通过使用默认的HTTP端口来暴露管理端点是一个明智的选择。 然而，如果你的应用程序在你自己的数据中心内运行，你可能更喜欢使用不同的HTTP端口来暴露端点。





你可以设置 `management.server.port` 属性来改变HTTP端口，如下面的例子所示。







Properties

Yaml





```
management.server.port=8081

```







|  | 在 Cloud Foundry 上，默认情况下，应用程序仅在端口 8080 上接收 HTTP 和 TCP 路由的请求。 如果您想在 Cloud Foundry 上使用自定义管理端口，您需要明确设置应用程序的路由以将流量转发到自定义端口。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring.management-specific-ssl)3.3\. 配置针对Management（管理）层的SSL



当配置为使用自定义端口时，你也可以通过使用各种 `management.server.ssl.*` 属性来配置管理服务器的SSL。 例如，这样做可以让管理服务器通过HTTP提供服务，而主应用程序使用HTTPS，如以下属性设置所示。







Properties

Yaml





```
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:store.jks
server.ssl.key-password=secret
management.server.port=8080
management.server.ssl.enabled=false

```







或者，主服务器和管理服务器都可以使用SSL，但使用不同的密钥存储，如下所示。







Properties

Yaml





```
server.port=8443
server.ssl.enabled=true
server.ssl.key-store=classpath:main.jks
server.ssl.key-password=secret
management.server.port=8080
management.server.ssl.enabled=true
management.server.ssl.key-store=classpath:management.jks
management.server.ssl.key-password=secret

```









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring.customizing-management-server-address)3.4\. 定制Management（管理）服务器地址



你可以通过设置 `management.server.address` 性来定制管理端点的可用地址。 如果你想只在内部或面向运维的网络上监听，或者只监听来自 `localhost` 的连接，这样做会很有用。





|  | 只有当端口与主服务器端口不同时，你才能在不同的地址上进行监听。 |
| --- | --- |





下面的例子 `application.properties` 不允许远程管理连接。







Properties

Yaml





```
management.server.port=8081
management.server.address=127.0.0.1

```









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.monitoring.disabling-http-endpoints)3.5\. 禁用HTTP端点



如果你不想通过HTTP暴露端点，你可以把管理端口设置为 `-1`，如下例所示。







Properties

Yaml





```
management.server.port=-1

```







你也可以通过使用 `management.endpoints.web.exposure.exclude` 属性来实现，如下例所示。







Properties

Yaml





```
management.endpoints.web.exposure.exclude=*

```













## [](https://springdoc.cn/spring-boot/actuator.html#actuator.jmx)4\. 通过JMX进行监控和管理





Java管理扩展（JMX）提供了一个标准的机制来监控和管理应用程序。 默认情况下，该功能未被启用。 你可以通过设置 `spring.jmx.enabled` 配置属性为 `true` 来打开它。 Spring Boot将最合适的 `MBeanServer` 作为ID为 `mbeanServer` 的Bean公开。 你的任何带有Spring JMX注解的Bean（`@ManagedResource`、`@ManagedAttribute` 或 `@ManagedOperation`）都会暴露给它。





如果你的平台提供了一个标准的 `MBeanServer` ，Spring Boot会使用它，并在必要时默认为VM `MBeanServer`。 如果所有这些都失败了，就会创建一个新的 `MBeanServer`。





更多细节见 [`JmxAutoConfiguration`](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/jmx/JmxAutoConfiguration.java) 类。





默认情况下，Spring Boot也将管理端点作为JMX MBeans在 `org.springframework.boot` 域下公开。 要完全控制JMX域中的端点注册，可以考虑注册你自己的 `EndpointObjectNameFactory` 实现。





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.jmx.custom-mbean-names)4.1\. 自定义MBean名称



MBean的名字通常由端点的 `id` 生成。 例如， `health` 端点被暴露为 `org.springframework.boot:type=Endpoint,name=Health`。





如果你的应用程序包含一个以上的Spring `ApplicationContext`，你可能会发现名字发生冲突。 为了解决这个问题，你可以将 `spring.jmx.unique-names` 属性设置为 `true`，这样MBean的名字就总是唯一的。





你还可以自定义暴露端点的JMX域。 下面的设置显示了在 `application.properties` 中这样做的一个例子。







Properties

Yaml





```
spring.jmx.unique-names=true
management.endpoints.jmx.domain=com.example.myapp

```









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.jmx.disable-jmx-endpoints)4.2\. 禁用JMX端点



如果你不想通过JMX暴露端点，你可以把 `management.endpoints.jmx.exposure.exclude` 属性设置为 `*`，如下例所示。







Properties

Yaml





```
management.endpoints.jmx.exposure.exclude=*

```













## [](https://springdoc.cn/spring-boot/actuator.html#actuator.observability)5\. 可观测性（Observability）





可观察性是指从外部观察一个运行中的系统的内部状态的能力。它由三个支柱组成：日志、度量和跟踪。





对于度量和跟踪，Spring Boot使用 [Micrometer Observation](https://micrometer.io/docs/observation)。要创建你自己的观察（这将导致度量和跟踪），你可以注入一个 `ObservationRegistry`。







```
@Component
public class MyCustomObservation {

    private final ObservationRegistry observationRegistry;

    public MyCustomObservation(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    public void doSomething() {
        Observation.createNotStarted("doSomething", this.observationRegistry)
                .lowCardinalityKeyValue("locale", "en-US")
                .highCardinalityKeyValue("userId", "42")
                .observe(() -> {
                    // Execute business logic here
                });
    }

}

```







|  | 低卡度的标签将被添加到指标和追踪中，而高卡度的标签将只被添加到追踪中。 |
| --- | --- |





`ObservationPredicate`、`GlobalObservationConvention` 和 `ObservationHandler` 类型的 Bean 将被自动注册到 `ObservationRegistry` 上。你可以另外注册任意数量的 `ObservationRegistryCustomizer` Bean来进一步配置注册表。





更多细节请见 [Micrometer Observation 文档](https://micrometer.io/docs/observation)。





|  | JDBC和R2DBC的可观察性（Observability）可以使用单独的项目进行配置。对于 JDBC， [Datasource Micrometer 项目](https://github.com/jdbc-observations/datasource-micrometer) 提供了一个 Spring Boot Starter，在调用JDBC操作时自动创建观察。在 [参考文档](https://jdbc-observations.github.io/datasource-micrometer/docs/current/docs/html/)中阅读更多关于它的信息。对于R2DBC， [R2DBC观察的Spring Boot自动配置](https://github.com/spring-projects-experimental/r2dbc-micrometer-spring-boot) 可以为R2DBC查询调用创建观察。 |
| --- | --- |





接下来的章节将提供关于日志、指标和追踪的更多细节。









## [](https://springdoc.cn/spring-boot/actuator.html#actuator.loggers)6\. 日志记录器（Logger）





Spring Boot Actuator包括在运行时查看和配置应用程序的日志级别的功能。 你可以查看整个列表或单个日志记录器的配置，它由明确配置的日志级别以及日志框架赋予它的有效日志级别组成。 这些级别可以是以下之一。





*   `TRACE`

*   `DEBUG`

*   `INFO`

*   `WARN`

*   `ERROR`

*   `FATAL`

*   `OFF`

*   `null`





`null` 表示没有明确的配置。





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.loggers.configure)6.1\. 配置一个 Logger



要配置一个给定的记录器，`POST` 一个部分实体到资源的URI，如下面的例子所示。







```
{
    "configuredLevel": "DEBUG"
}
```







|  | 要 “reset” （重置）记录器的特定级别（并使用默认配置），你可以传递一个 `null` 的值作为 `configuredLevel`。 |
| --- | --- |











## [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics)7\. 指标（Metrics）





Spring Boot Actuator为 [Micrometer](https://micrometer.io/) 提供了依赖管理和自动配置，Micrometer是一个支持 [众多监控系统](https://micrometer.io/docs) 的应用程序指标接口，包括。





*   [AppOptics](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.appoptics)

*   [Atlas](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.atlas)

*   [Datadog](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.datadog)

*   [Dynatrace](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.dynatrace)

*   [Elastic](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.elastic)

*   [Ganglia](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.ganglia)

*   [Graphite](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.graphite)

*   [Humio](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.humio)

*   [Influx](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.influx)

*   [JMX](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.jmx)

*   [KairosDB](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.kairos)

*   [New Relic](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.newrelic)

*   [OpenTelemetry](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.otlp)

*   [Prometheus](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.prometheus)

*   [SignalFx](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.signalfx)

*   [Simple (in-memory)](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.simple)

*   [Stackdriver](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.stackdriver)

*   [StatsD](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.statsd)

*   [Wavefront](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.wavefront)





|  | 要了解更多关于Micrometer的功能，请参见其 [参考文档](https://micrometer.io/docs)，特别是 [概念部分](https://micrometer.io/docs/concepts)。 |
| --- | --- |





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.getting-started)7.1\. 入门



Spring Boot会自动配置一个复合的 `MeterRegistry`，并为它在classpath上发现的每个支持的实现添加一个注册表。 在你的运行时classpath中有对 `micrometer-registry-{system}` 的依赖就足以让Spring Boot配置注册表了。





大多数注册表有共同的特点。 例如，即使 Micrometer 注册表的实现在classpath上，你也可以禁用一个特定的注册表。 下面的例子禁用了Datadog。







Properties

Yaml





```
management.datadog.metrics.export.enabled=false

```







你也可以禁用所有的注册表，除非注册表特定属性另有说明，如下例所示。







Properties

Yaml





```
management.defaults.metrics.export.enabled=false

```







Spring Boot还将任何自动配置的注册表添加到 `Metrics` 类上的全局静态复合注册表，除非你明确告诉它不要这样做。







Properties

Yaml





```
management.metrics.use-global-registry=false

```







你可以注册任意数量的 `MeterRegistryCustomizer` Bean来进一步配置注册表，比如在任何表被注册之前应用普通标签。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MyMeterRegistryConfiguration {

    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return (registry) -> registry.config().commonTags("region", "us-east-1");
    }

}

```







你可以通过更具体的泛型将定制应用于特定的注册表实现。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MyMeterRegistryConfiguration {

    @Bean
    public MeterRegistryCustomizer<GraphiteMeterRegistry> graphiteMetricsNamingConvention() {
        return (registry) -> registry.config().namingConvention(this::name);
    }

    private String name(String name, Meter.Type type, String baseUnit) {
        return ...
    }

}

```







Spring Boot还 [配置了内置 instrumentation](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported)，你可以通过配置或专用注解标记来控制。







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export)7.2\. 支持的监控系统



本节简要介绍每个支持的监控系统。





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.appoptics)7.2.1\. AppOptics



默认情况下，AppOptics注册中心会定期将指标推送到 `[api.appoptics.com/v1/measurements](https://api.appoptics.com/v1/measurements)`。要将指标导出到 SaaS [AppOptics](https://micrometer.io/docs/registry/appOptics)，必须提供你的API令牌。







Properties

Yaml





```
management.appoptics.metrics.export.api-token=YOUR_TOKEN

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.atlas)7.2.2\. Atlas



默认情况下，指标会被导出到运行在你本地机器上的 [Atlas](https://micrometer.io/docs/registry/atlas)。你可以提供 [Atlas server](https://github.com/Netflix/atlas) 的位置。







Properties

Yaml





```
management.atlas.metrics.export.uri=https://atlas.example.com:7101/api/v1/publish

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.datadog)7.2.3\. Datadog



一个Datadog注册中心会定期将指标推送到 [datadoghq](https://www.datadoghq.com/)。 要导出指标到 [Datadog](https://micrometer.io/docs/registry/datadog)，你必须提供你的API密钥。







Properties

Yaml





```
management.datadog.metrics.export.api-key=YOUR_KEY

```







如果你另外提供一个应用密钥（可选），那么元数据，如仪表描述、类型和基本单位也将被导出。







Properties

Yaml





```
management.datadog.metrics.export.api-key=YOUR_API_KEY
management.datadog.metrics.export.application-key=YOUR_APPLICATION_KEY

```







默认情况下，指标被发送到Datadog美国 [site](https://docs.datadoghq.com/getting_started/site) （`[api.datadoghq.com](https://api.datadoghq.com/)`）。 如果你的Datadog项目托管在其他网站上，或者你需要通过代理发送指标，请相应配置URI。







Properties

Yaml





```
management.datadog.metrics.export.uri=https://api.datadoghq.eu

```







你还可以改变向Datadog发送指标的时间间隔。







Properties

Yaml





```
management.datadog.metrics.export.step=30s

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.dynatrace)7.2.4\. Dynatrace



Dynatrace提供了两个指标摄取API，都是为 [Micrometer](https://micrometer.io/docs/registry/dynatrace) 实现的。你可以在 [这里](https://www.dynatrace.com/support/help/how-to-use-dynatrace/metrics/metric-ingestion/ingestion-methods/micrometer) 找到Dynatrace关于Micrometer指标摄入的文档。`v1` 命名空间中的配置属性只适用于导出到 [Timeseries v1 API](https://www.dynatrace.com/support/help/dynatrace-api/environment-api/metric-v1/) 时。`v2` 命名空间中的配置属性只适用于导出到 [Metrics v2 API](https://www.dynatrace.com/support/help/dynatrace-api/environment-api/metric-v2/post-ingest-metrics/) 时。请注意，该集成每次只能导出到API的 `v1` 或 `v2` 版本，`v2` 版本是首选。如果 `device-id`（v1版需要，但在v2版中不使用）在 `v1` 版命名空间中被设置，那么metric将被导出到 `v1` 版端点。否则，就假定是 `v2` 版本。





##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.dynatrace.v2-api)v2 API



你可以通过两种方式使用v2 API。





###### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.dynatrace.v2-api.auto-config)自动配置



Dynatrace自动配置适用于由OneAgent或Dynatrace Operator for Kubernetes监控的主机。





**本地OneAgent：**如果主机上运行OneAgent，指标会自动输出到 [local OneAgent ingest endpoint](https://www.dynatrace.com/support/help/how-to-use-dynatrace/metrics/metric-ingestion/ingestion-methods/local-api/) 。 摄取端点将指标转发到Dynatrace后端。





**Dynatrace Kubernetes Operator：**在安装了Dynatrace Operator的Kubernetes中运行时，注册表将自动从操作员那里获取你的端点URI和API令牌。





这是默认行为，除了依赖 `io.micrometer:micrometer-registry-dynatrace` 之外，不需要特别的设置。







###### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.dynatrace.v2-api.manual-config)手动配置



如果没有自动配置，则需要 [Metrics v2 API](https://www.dynatrace.com/support/help/dynatrace-api/environment-api/metric-v2/post-ingest-metrics/) 的端点和一个 API 令牌。API令牌必须有 “Ingest metrics” （`metrics.ingest`）的权限设置。我们建议将令牌的范围限制在这一个权限上。你必须确保端点URI包含路径（例如，`/api/v2/metrics/ingest`）。





Metrics API v2摄取端点的URL根据你的部署选项而不同。





*   SaaS: `https://{your-environment-id}.live.dynatrace.com/api/v2/metrics/ingest`

*   Managed deployments: `https://{your-domain}/e/{your-environment-id}/api/v2/metrics/ingest`





下面的例子是用 `example` environment id 配置度量值导出。







Properties

Yaml





```
management.dynatrace.metrics.export.uri=https://example.live.dynatrace.com/api/v2/metrics/ingest
management.dynatrace.metrics.export.api-token=YOUR_TOKEN

```







在使用Dynatrace v2 API时，可以使用以下可选功能（更多细节可在 [Dynatrace文档](https://www.dynatrace.com/support/help/how-to-use-dynatrace/metrics/metric-ingestion/ingestion-methods/micrometer#dt-configuration-properties) 中找到）。





*   Metric key 的前缀。设置一个前缀，该前缀将被添加到所有导出的metric key中。

*   用Dynatrace元数据来充实。如果OneAgent或Dynatrace操作员正在运行，用额外的元数据（例如，关于主机、进程或Pod）来丰富指标。

*   默认维度。指定添加到所有导出度量的键值对。 如果用Micrometer指定了具有相同键的标签，它们将覆盖默认dimension。

*   使用Dynatrace Summary instrument。在某些情况下，Micrometer Dynatrace注册表创建的指标被拒绝。 在Micrometer 1.9.x中，通过引入Dynatrace特定的摘要工具来解决这个问题。 把这个开关设置为 `false` 会迫使Micrometer回到1.9.x之前的默认行为。 只有在从Micrometer 1.8.x迁移到1.9.x时遇到问题时才可以使用。





可以不指定URI和API令牌，如以下例子所示。 在这种情况下，会使用自动配置的端点。







Properties

Yaml





```
management.dynatrace.metrics.export.v2.metric-key-prefix=your.key.prefix
management.dynatrace.metrics.export.v2.enrich-with-dynatrace-metadata=true
management.dynatrace.metrics.export.v2.default-dimensions.key1=value1
management.dynatrace.metrics.export.v2.default-dimensions.key2=value2
management.dynatrace.metrics.export.v2.use-dynatrace-summary-instruments=true

```











##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.dynatrace.v1-api)v1 API (Legacy)



Dynatrace v1 API指标注册表通过使用 [Timeseries v1 API](https://www.dynatrace.com/support/help/dynatrace-api/environment-api/metric-v1/) 定期将指标推送到配置的URI。 为了向后兼容现有的设置，当 `device-id` 被设置时（v1需要，但在v2中不使用），指标被导出到Timeseries v1端点。 要向 [Dynatrace](https://micrometer.io/docs/registry/dynatrace) 导出指标，必须提供你的API令牌、设备ID和URI。







Properties

Yaml





```
management.dynatrace.metrics.export.uri=https://{your-environment-id}.live.dynatrace.com
management.dynatrace.metrics.export.api-token=YOUR_TOKEN
management.dynatrace.metrics.export.v1.device-id=YOUR_DEVICE_ID

```







对于v1版API，你必须指定基本环境URI，而不指定路径，因为v1版的端点路径会自动添加。







##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.dynatrace.version-independent-settings)与版本无关的设置



除了API端点和令牌外，你还可以改变向Dynatrace发送指标的间隔时间。 默认的导出时间间隔是 `60s`。 下面的例子将导出时间间隔设置为30秒。







Properties

Yaml





```
management.dynatrace.metrics.export.step=30s

```







你可以在 [Micrometer文档](https://micrometer.io/docs/registry/dynatrace) 和 [Dynatrace文档](https://www.dynatrace.com/support/help/how-to-use-dynatrace/metrics/metric-ingestion/ingestion-methods/micrometer) 中找到关于如何为Micrometer设置Dynatrace exporter（导出器）的更多信息。









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.elastic)7.2.5\. Elastic



默认情况下，指标被导出到运行在你本地机器上的 [Elastic](https://micrometer.io/docs/registry/elastic) 。 你可以通过使用以下属性提供要使用的Elastic服务器的位置。







Properties

Yaml





```
management.elastic.metrics.export.host=https://elastic.example.com:8086

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.ganglia)7.2.6\. Ganglia



默认情况下，指标被导出到运行在你本地机器上的 [Ganglia](https://micrometer.io/docs/registry/ganglia) 。你可以提供 [Ganglia server](http://ganglia.sourceforge.net/) 的主机和端口，如下例所示。







Properties

Yaml





```
management.ganglia.metrics.export.host=ganglia.example.com
management.ganglia.metrics.export.port=9649

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.graphite)7.2.7\. Graphite



默认情况下，指标会被导出到运行在你本地机器上的 [Graphite](https://micrometer.io/docs/registry/graphite) 。你可以提供 [Graphite server](https://graphiteapp.org/) 的主机和端口，如下例所示。







Properties

Yaml





```
management.graphite.metrics.export.host=graphite.example.com
management.graphite.metrics.export.port=9004

```







Micrometer提供了一个默认的 `HierarchicalNameMapper`，管理dimensional meter ID如何https://micrometer.io/docs/registry/graphite#_hierarchical_name_mapping[映射到 flat hierarchical name]。





|  | 要控制这种行为，请定义你的 `GraphiteMeterRegistry` 并提供你自己的 `HierarchicalNameMapper`。 除非你自己定义，否则会提供一个自动配置的 `GraphiteConfig` 和 `Clock` Bean。JavaKotlin```@Configuration(proxyBeanMethods = false)public class MyGraphiteConfiguration {    @Bean    public GraphiteMeterRegistry graphiteMeterRegistry(GraphiteConfig config, Clock clock) {        return new GraphiteMeterRegistry(config, clock, this::toHierarchicalName);    }    private String toHierarchicalName(Meter.Id id, NamingConvention convention) {        return ...    }}``` |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.humio)7.2.8\. Humio



默认情况下，Humio注册中心会定期将指标推送到 [cloud.humio.com](https://cloud.humio.com/) 。 要将指标导出到SaaS [Humio](https://micrometer.io/docs/registry/humio)，你必须提供你的API令牌。







Properties

Yaml





```
management.humio.metrics.export.api-token=YOUR_TOKEN

```







你还应该配置一个或多个标签，以确定推送指标的数据源。







Properties

Yaml





```
management.humio.metrics.export.tags.alpha=a
management.humio.metrics.export.tags.bravo=b

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.influx)7.2.9\. Influx



默认情况下，指标会被导出到运行在本地机器上的 [Influx](https://micrometer.io/docs/registry/influx) v1实例，并采用默认配置。要导出指标到InfluxDB v2，请配置 `org`、`bucket` 和用于写入指标的authentication `token`。你可以通过以下方式提供要使用的 [Influx server](https://www.influxdata.com/) 的位置。







Properties

Yaml





```
management.influx.metrics.export.uri=https://influx.example.com:8086

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.jmx)7.2.10\. JMX



Micrometer提供了对 [JMX](https://micrometer.io/docs/registry/jmx) 的分层映射，主要是作为一种廉价和可移植的方式来查看本地的度量。 默认情况下，指标被导出到 `metrics` JMX域。 你可以通过以下方式提供要使用的域。







Properties

Yaml





```
management.jmx.metrics.export.domain=com.example.app.metrics

```







Micrometer提供了一个默认的 `HierarchicalNameMapper`，管理dimensional meter ID 如何 [映射到 flat hierarchical name](https://micrometer.io/docs/registry/jmx#_hierarchical_name_mapping)。





|  | 要控制这种行为，请定义你的 `JmxMeterRegistry` 并提供你自己的 `HierarchicalNameMapper`。 除非你自己定义，否则会提供一个自动配置的 `JmxConfig` 和 `Clock` Bean。JavaKotlin```@Configuration(proxyBeanMethods = false)public class MyJmxConfiguration {    @Bean    public JmxMeterRegistry jmxMeterRegistry(JmxConfig config, Clock clock) {        return new JmxMeterRegistry(config, clock, this::toHierarchicalName);    }    private String toHierarchicalName(Meter.Id id, NamingConvention convention) {        return ...    }}``` |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.kairos)7.2.11\. KairosDB



默认情况下，指标被导出到运行在你本地机器上的 [KairosDB](https://micrometer.io/docs/registry/kairos) 。你可以通过以下方式提供要使用的 [KairosDB server](https://kairosdb.github.io/) 的位置。







Properties

Yaml





```
management.kairos.metrics.export.uri=https://kairosdb.example.com:8080/api/v1/datapoints

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.newrelic)7.2.12\. New Relic



New Relic注册中心会定期将指标推送到 [New Relic](https://micrometer.io/docs/registry/new-relic)。要导出指标到 [New Relic](https://newrelic.com/)，你必须提供你的API密钥和账户ID。







Properties

Yaml





```
management.newrelic.metrics.export.api-key=YOUR_KEY
management.newrelic.metrics.export.account-id=YOUR_ACCOUNT_ID

```







你还可以改变向New Relic发送指标的时间间隔。







Properties

Yaml





```
management.newrelic.metrics.export.step=30s

```







默认情况下，指标是通过REST调用发布的，但如果你在classpath上有Java Agent API，你也可以使用它。







Properties

Yaml





```
management.newrelic.metrics.export.client-provider-type=insights-agent

```







最后，你可以通过定义你自己的 `NewRelicClientProvider` 豆来完全控制。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.otlp)7.2.13\. OpenTelemetry



默认情况下，指标被导出到运行在你本地机器上的 [OpenTelemetry](https://micrometer.io/docs/registry/otlp) 。你可以通过以下方式提供要使用的 [OpenTelemtry metric endpoint](https://opentelemetry.io/) 的位置。







Properties

Yaml





```
management.otlp.metrics.export.url=https://otlp.example.com:4318/v1/metrics

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.prometheus)7.2.14\. Prometheus



[Prometheus](https://micrometer.io/docs/registry/prometheus) 希望scrape或轮询单个应用程序实例的指标。Spring Boot在 `/actuator/prometheus` 提供了一个actuator端点，以便以适当的格式呈现 [Prometheus scrape](https://prometheus.io/)。





|  | 默认情况下，该端点是不可用的，必须被暴露。更多细节请参见[暴露端点](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.exposing)。 |
| --- | --- |





下面的例子 `scrape_config` 添加到 `prometheus.yml`。







```
scrape_configs:
  - job_name: "spring"
    metrics_path: "/actuator/prometheus"
    static_configs:
      - targets: ["HOST:PORT"]
```







也支持 [Prometheus Exemplars](https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage)。要启用这个功能，应该有一个 `SpanContextSupplier` Bean。如果你使用 [Micrometer Tracing](https://micrometer.io/docs/tracing)，这将为你自动配置，但如果你想，你总是可以创建自己的。请查看 [Prometheus 文档](https://prometheus.io/docs/prometheus/latest/feature_flags/#exemplars-storage) ，因为这个功能需要在Prometheus这边明确启用，而且只支持使用 [OpenMetrics](https://github.com/OpenObservability/OpenMetrics/blob/v1.0.0/specification/OpenMetrics.md#exemplars) 格式。





对于短暂的或批处理的作业，可能存在的时间不够长，无法被刮取，你可以使用 [Prometheus Pushgateway](https://github.com/prometheus/pushgateway) 支持，将指标暴露给Prometheus。要启用Prometheus Pushgateway支持，请在你的项目中添加以下依赖。







```
<dependency>
    <groupId>io.prometheus</groupId>
    simpleclient_pushgateway
</dependency>
```







当Prometheus Pushgateway的依赖出现在classpath上，并且 `management.prometheus.metrics.export.pushgateway.enabled` 属性被设置为 `true` 时，一个 `PrometheusPushGatewayManager` bean就被自动配置了。 它负责管理向Prometheus Pushgateway推送指标的工作。





你可以通过使用 `management.prometheus.metrics.export.pushgateway` 下的属性来调整 `PrometheusPushGatewayManager`。 对于高级配置，你也可以提供你自己的 `PrometheusPushGatewayManager` bean。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.signalfx)7.2.15\. SignalFx



SignalFx注册中心会定期将指标推送到 [SignalFx](https://micrometer.io/docs/registry/signalFx)。要导出指标到 [SignalFx](https://www.signalfx.com/)，你必须提供你的access token。







Properties

Yaml





```
management.signalfx.metrics.export.access-token=YOUR_ACCESS_TOKEN

```







你也可以改变向SignalFx发送指标的时间间隔。







Properties

Yaml





```
management.signalfx.metrics.export.step=30s

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.simple)7.2.16\. Simple



Micrometer提供了一个简单的、内存中的后端，如果没有配置其他注册表，该后端会自动作为备用。这可以让你看到在 [metrics endpoint](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.endpoint) 中收集了哪些度量。





一旦你使用任何其他可用的后端，内存中的后端就会自动关闭。你也可以明确地禁用它。







Properties

Yaml





```
management.simple.metrics.export.enabled=false

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.stackdriver)7.2.17\. Stackdriver



Stackdriver注册中心会定期向 [Stackdriver](https://cloud.google.com/stackdriver/) 推送指标。要导出指标到SaaS [Stackdriver](https://micrometer.io/docs/registry/stackdriver)，你必须提供你的Google Cloud project ID。







Properties

Yaml





```
management.stackdriver.metrics.export.project-id=my-project

```







你还可以改变向Stackdriver发送指标的时间间隔。







Properties

Yaml





```
management.stackdriver.metrics.export.step=30s

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.statsd)7.2.18\. StatsD



StatsD注册表急切地将指标通过UDP推送给StatsD agent。默认情况下，指标被导出到运行在你本地机器上的 [StatsD](https://micrometer.io/docs/registry/statsD) agent。你可以通过以下方式提供StatsD代理的主机、端口和协议，以便使用。







Properties

Yaml





```
management.statsd.metrics.export.host=statsd.example.com
management.statsd.metrics.export.port=9125
management.statsd.metrics.export.protocol=udp

```







你还可以改变要使用的StatsD线路协议（默认为Datadog）。







Properties

Yaml





```
management.statsd.metrics.export.flavor=etsy

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.export.wavefront)7.2.19\. Wavefront



Wavefront注册表定期将指标推送到 [Wavefront](https://micrometer.io/docs/registry/wavefront)。如果你直接将指标导出到 [Wavefront](https://www.wavefront.com/)，你必须提供你的API token。







Properties

Yaml





```
management.wavefront.api-token=YOUR_API_TOKEN

```







另外，您可以在您的环境中使用Wavefront sidecar或内部代理来转发指标数据到Wavefront API主机。







Properties

Yaml





```
management.wavefront.uri=proxy://localhost:2878

```







如果您将指标发布到Wavefront代理（如 [Wavefront文档](https://docs.wavefront.com/proxies_installing.html) 中所述），主机必须是 `proxy://HOST:PORT` 格式。





你也可以改变向Wavefront发送指标的时间间隔。







Properties

Yaml





```
management.wavefront.metrics.export.step=30s

```











### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported)7.3\. 支持的指标 （Metric）和度量（Meter）



Spring Boot为各种各样的技术提供了自动计量器注册。 在大多数情况下，默认值提供了合理的指标，可以发布到任何支持的监控系统中。





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.jvm)7.3.1\. JVM指标



自动配置通过使用核心 Micrometer 类启用JVM度量。 JVM指标在 `jvm.` meter name 下发布。





提供以下JVM指标。





*   各种内存和缓冲池细节

*   与垃圾收集有关的统计数据

*   线程利用率

*   加载和卸载的类的数量

*   JVM的版本信息

*   JIT 编译时间







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.system)7.3.2\. 系统指标



自动配置通过使用核心Micrometer类来实现系统度量。 系统指标在 `system.`、`process.` 和 `disk.` meter 名下发布。





提供以下系统指标。





*   CPU指标

*   文件描述符指标

*   正常运行时间指标（包括应用程序已经运行的时间和绝对启动时间的固定测量）。

*   可用的磁盘空间







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.application-startup)7.3.3\. 应用程序启动指标



自动配置暴露了应用程序的启动时间指标。





*   `application.started.time`: 启动应用程序的时间。

*   `application.ready.time`：应用程序准备好为请求提供服务所需的时间。





指标是由应用类的完全名称来标记的。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.logger)7.3.4\. 日志记录器指标



自动配置启用了Logback和Log4J2的事件度量。 细节在 `log4j2.events.` 或 `logback.events.` meter名下公布。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.tasks)7.3.5\. 任务执行和调度指标



自动配置使所有可用的 `ThreadPoolTaskExecutor` 和 `ThreadPoolTaskScheduler` Bean都能被测量，只要底层的 `ThreadPoolExecutor` 可用。 指标由executor的名称来标记，executor的名称来自于Bean的名称。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.spring-mvc)7.3.6\. Spring MVC 指标



自动配置能够对 Spring MVC Controller和编程式handler处理的所有请求进行度量。 默认情况下，指标是以 `http.server.requests` 为名称生成的。 你可以通过设置 `management.observations.http.server.requests.name` 属性来定制该名称。





关于产生的观察结果（observation）的更多信息，请参见 [Spring Framework 参考文档](https://docs.spring.io/spring-framework/docs/6.0.5/reference/html/integration.html#integration.observability.http-server.servlet)。





要添加到默认标签中，请提供一个继承了 `org.springframework.http.server.observation` 包中的 `DefaultServerRequestObservationConvention` 的 `@Bean`。要替换默认标签，请提供一个实现 `ServerRequestObservationConvention` 的 `@Bean`。





|  | 在某些情况下，Web控制器中处理的异常不会被记录为请求度量标签。应用程序可以选择加入并通过将[handled exception 设置为 request attribute](https://springdoc.cn/spring-boot/web.html#web.servlet.spring-mvc.error-handling)来记录异常。 |
| --- | --- |





默认情况下，所有请求都被处理。 要自定义过滤器，请提供一个实现 `FilterRegistrationBean<WebMvcMetricsFilter>` 的 `@Bean`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.spring-webflux)7.3.7\. Spring WebFlux 指标



自动配置能够对Spring WebFlux controller和编程式handler的所有请求进行度量。 默认情况下，指标是以 `http.server.requests` 为名生成的。 你可以通过设置 `management.observations.http.server.requests.name` 属性来定制该名称。





关于产生的观察结果（observation）的更多信息，请参见 [Spring Framework 参考文档](https://docs.spring.io/spring-framework/docs/6.0.5/reference/html/integration.html#integration.observability.http-server.reactive)。





要添加到默认标签中，请提供继承了 `org.springframework.http.server.reactive.observation` 包中的 `DefaultServerRequestObservationConvention` 的 `@Bean`。要替换默认标签，请提供一个实现 `ServerRequestObservationConvention` 的 `@Bean`。





|  | 在某些情况下，控制器和处理函数中处理的异常不会被记录为请求度量标签。应用程序可以选择加入并通过[将handled exception设置为request attribute](https://springdoc.cn/spring-boot/web.html#web.reactive.webflux.error-handling)来记录异常。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.jersey)7.3.8\. Jersey Server 指标



自动配置使Jersey JAX-RS实现所处理的所有请求都能被测量。 默认情况下，指标是以 `http.server.requests` 为名称生成的。 你可以通过设置 `management.observations.http.server.requests.name` 属性来定制名称。





默认情况下，Jersey服务器指标被标记为以下信息。



<colgroup><col><col></colgroup>
| Tag | 说明 |
| --- | --- |
| `exception` | 处理请求时抛出的任何异常的简单类名。 |
| `method` | 请求的方法（例如，`GET` 或 `POST`）。 |
| `outcome` | 请求的结果，基于响应的状态代码。 1xx是 `INFORMATIONAL`，2xx是 `SUCCESS`，3xx是 `REDIRECTION`，4xx是 `CLIENT_ERROR`，5xx是 `SERVER_ERROR`。 |
| `status` | 响应的HTTP状态代码（例如，`200` 或 `500`）。 |
| `uri` | 如果可能的话，在进行变量替换之前，请求的URI模板（例如：`/api/person/{id}`）。 |



要定制标签，请提供一个实现 `JerseyTagsProvider` 的 `@Bean`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.http-clients)7.3.9\. HTTP Client 指标



Spring Boot Actuator负责管理 `RestTemplate` 和 `WebClient` 的工具。为此，你必须注入自动配置的构建器并使用它来创建实例。





*   `RestTemplateBuilder` 用于 `RestTemplate`

*   `WebClient.Builder` 用于 `WebClient`





你也可以手动应用负责这个工具的customizer，即 `ObservationRestTemplateCustomizer` 和 `ObservationWebClientCustomizer`。





默认情况下，指标是以 `http.client.requests` 这个名字生成的。





你可以通过设置 `management.observations.http.client.requests.name` 属性来定制这个名字。





关于产生的观察结果（observation）的更多信息，请参见 [Spring Framework 参考文档](https://docs.spring.io/spring-framework/docs/6.0.5/reference/html/integration.html#integration.observability.http-client)。





要在使用 `RestTemplate` 时定制标签，请提供一个实现了 `org.springframework.http.client.observation` 包中 `ClientRequestObservationConvention` 的 `@Bean`。要在使用 `WebClient` 时自定义标签，请提供一个实现了 `org.springframework.web.reactive.function.client` 包中 `ClientRequestObservationConvention` 的 `@Bean`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.tomcat)7.3.10\. Tomcat 指标



自动配置仅在 `MBeanRegistry` 被启用时才会启用Tomcat的仪器。 默认情况下，`MBeanRegistry` 是禁用的，但你可以通过设置 `server.tomcat.mbeanregistry.enabled` 为 `true` 来启用它。





Tomcat的指标在 `tomcat.` meter 名下发布。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.cache)7.3.11\. Cache 指标



自动配置可以在启动时对所有可用的 `Cache` 实例进行检测，其指标以 `cache` 为前缀。





缓存仪表是标准化的基本指标集。





也可以使用额外的、针对缓存的指标。





支持以下缓存库。





*   Cache2k

*   Caffeine

*   Hazelcast

*   任何兼容的JCache（JSR-107）实现

*   Redis





指标由缓存的名称和 `CacheManager` 的名称来标记，`CacheManager` 的名称是由Bean名称派生的。





|  | 只有在启动时配置的缓存被绑定到注册表。 对于没有在缓存配置中定义的缓存，例如在启动阶段后临时创建的缓存或以编程方式创建的缓存，需要明确注册。 一个 `CacheMetricsRegistrar` Bean可以使这个过程更容易。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.spring-graphql)7.3.12\. Spring GraphQL 指标



参见 [Spring GraphQL 参考文档](https://docs.spring.io/spring-graphql/docs/1.1.2/reference/html/)。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.jdbc)7.3.13\. DataSource 指标



自动配置使所有可用的 `DataSource` 对象的仪器化，指标前缀为 `jdbc.connections`。 数据源检测的结果是表示池中当前活动的、空闲的、最大允许的和最小允许的连接数的仪表。





衡量标准也由基于bean名称计算的 `DataSource` 的名称来标记。





|  | 默认情况下，Spring Boot为所有支持的数据源提供元数据。 如果你喜欢的数据源不被支持，你可以添加额外的 `DataSourcePoolMetadataProvider` Bean。 请参阅 `DataSourcePoolMetadataProvidersConfiguration` 以了解实例。 |
| --- | --- |





此外，Hikari特定的指标以 `hikaricp` 前缀暴露。 每个指标都被pool的名字所标记（你可以用 `spring.datasource.name` 来控制）。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.hibernate)7.3.14\. Hibernate 指标



如果 `org.hibernate.orm:hibernate-micrometer` 在classpath上，所有启用了统计功能的Hibernate `EntityManagerFactory` 实例都会被一个名为 `hibernate` 的指标所检测。





衡量标准也由 `EntityManagerFactory` 的名称来标记，该名称是由Bean名称派生的。





要启用统计，标准JPA属性 `hibernate.generate_statistics` 必须设置为 `true`。 你可以在自动配置的 `EntityManagerFactory` 上启用。







Properties

Yaml





```
spring.jpa.properties[hibernate.generate_statistics]=true

```









#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.spring-data-repository)7.3.15\. Spring Data Repository 指标



自动配置能够对所有Spring Data `Repository` 方法的调用进行度量。 默认情况下，指标以 `spring.data.repository.invocations` 为名生成。 你可以通过设置 `management.metrics.data.repository.metric-name` 属性来自定义名称。





`io.micrometer.core.annotation` 包中的 `@Timed` 注解支持 `Repository` 接口和方法。如果你不想记录所有 `Repository` 调用的度量（metric），你可以将 `management.metrics.data.repository.autotime.enabled` 设置为 `false`，而专门使用 `@Timed` 注解。





|  | 一个带有 `longTask = true` 的 `@Timed` 注解可以为该方法启用一个长任务计时器。长任务定时器需要一个单独的 metric name，并且可以与短任务定时器（task timer）叠加。 |
| --- | --- |





默认情况下，repository调用相关的度量标准被标记为以下信息。



<colgroup><col><col></colgroup>
| Tag | 说明 |
| --- | --- |
| `repository` | 源 `Repository` 的简单类名。 |
| `method` | 被调用的 `Repository` 方法的名称。 |
| `state` | 结果状态（`SUCCESS`, `ERROR`, `CANCELED`, `RUNNING`）。 |
| `exception` | 调用中抛出的任何异常的简单类名。 |



要替换默认标签，需要提供一个实现了 `RepositoryTagsProvider` 的 `@Bean`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.rabbitmq)7.3.16\. RabbitMQ 指标



自动配置使所有可用的 RabbitMQ 连接工厂的仪器化，其指标名为 `rabbitmq`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.spring-integration)7.3.17\. Spring Integration 指标



只要有 `MeterRegistry` bean，Spring Integration就会自动提供 [Micrometer support](https://docs.spring.io/spring-integration/docs/6.1.0-M1/reference/html/system-management.html#micrometer-integration)。 度量标准在 `spring.integration.` meter 名称下发布。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.kafka)7.3.18\. Kafka 指标



自动配置为自动配置的消费者工厂和生产者工厂分别注册了一个 `MicrometerConsumerListener` 和 `MicrometerProducerListener`。 它还为 `StreamsBuilderFactoryBean` 注册了一个 `KafkaStreamsMicrometerListener`。 更多细节，请参阅Spring Kafka文档中的 [Micrometer Native Metrics](https://docs.spring.io/spring-kafka/docs/3.0.3/reference/html/#micrometer-native) 部分。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.mongodb)7.3.19\. MongoDB 指标



本节简要介绍MongoDB的可用度量。





##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.mongodb.command)MongoDB命令指标



自动配置将 `MongoMetricsCommandListener` 与自动配置的 `MongoClient` 注册。





一个名为 `mongodb.driver.commands` 的timer指标被创建，用于向底层MongoDB driver发出的每条命令。 默认情况下，每个指标都被标记为以下信息。



<colgroup><col><col></colgroup>
| Tag | 说明 |
| --- | --- |
| `command` | 发出的命令的名称。 |
| `cluster.id` | 发送该命令的集群的标识符。 |
| `server.address` | 发送命令的服务器的地址。 |
| `status` | 命令的结果（`SUCCESS` 或 `FAILED）。 |



为了替换默认的度量衡标签，定义一个 `MongoCommandTagsProvider` bean，如下例所示。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MyCommandTagsProviderConfiguration {

    @Bean
    public MongoCommandTagsProvider customCommandTagsProvider() {
        return new CustomCommandTagsProvider();
    }

}

```







要禁用自动配置的命令度量，请设置以下属性。







Properties

Yaml





```
management.metrics.mongo.command.enabled=false

```









##### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.mongodb.connection-pool)MongoDB 连接池指标



自动配置将 `MongoMetricsConnectionPoolListener` 与自动配置的 `MongoClient` 注册。





以下是为连接池创建的测量指标。





*   `mongodb.driver.pool.size` 报告连接池的当前大小，包括空闲和正在使用的成员。

*   `mongodb.driver.pool.checkedout` 报告当前使用中的连接数。

*   `mongodb.driver.pool.waitqueuesize` 报告池中连接的等待队列的当前大小。





默认情况下，每个指标都被标记为以下信息。



<colgroup><col><col></colgroup>
| Tag | 说明 |
| --- | --- |
| `cluster.id` | 连接池所对应的集群的标识符。 |
| `server.address` | 连接池所对应的服务器的地址。 |



要取代默认的度量衡标签，请定义一个 `MongoConnectionPoolTagsProvider` bean。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MyConnectionPoolTagsProviderConfiguration {

    @Bean
    public MongoConnectionPoolTagsProvider customConnectionPoolTagsProvider() {
        return new CustomConnectionPoolTagsProvider();
    }

}

```







要禁用自动配置的连接池度量，请设置以下属性。







Properties

Yaml





```
management.metrics.mongo.connectionpool.enabled=false

```











#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.jetty)7.3.20\. Jetty 指标



自动配置通过使用Micrometer的 `JettyServerThreadPoolMetrics` 为Jetty的 `ThreadPool` 绑定指标。 Jetty的 `Connector` 实例的指标通过使用Micrometer的 `JettyConnectionMetrics` 来绑定，当 `server.ssl.enabled` 被设置为 `true` 时，Micrometer的 `JettySslHandshakeMetrics`。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.timed-annotation)7.3.21\. @Timed 注解的支持



要在Spring Boot不直接支持的地方使用 `@Timed`，请参考 [Micrometer 文档](https://micrometer.io/docs/concepts#_the_timed_annotation)。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.supported.redis)7.3.22\. Redis 指标



自动配置为自动配置的 `LettuceConnectionFactory` 注册了一个 `MicrometerCommandLatencyRecorder`。 更多细节，请参阅Lettuce文档的 [Micrometer Metrics部分](https://lettuce.io/core/6.2.3.RELEASE/reference/index.html#command.latency.metrics.micrometer)。









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.registering-custom)7.4\. 注册自定义指标



要注册自定义度量，请将 `MeterRegistry` 注入你的组件中。







Java

Kotlin





```
@Component
public class MyBean {

    private final Dictionary dictionary;

    public MyBean(MeterRegistry registry) {
        this.dictionary = Dictionary.load();
        registry.gauge("dictionary.size", Tags.empty(), this.dictionary.getWords().size());
    }

}

```







如果你的度量标准依赖于其他Bean，我们建议你使用 `MeterBinder` 来注册它们。







Java

Kotlin





```
public class MyMeterBinderConfiguration {

    @Bean
    public MeterBinder queueSize(Queue queue) {
        return (registry) -> Gauge.builder("queueSize", queue::size).register(registry);
    }

}

```







使用 `MeterBinder` 可以确保建立正确的依赖关系，并且在检索度量值的时候，Bean是可用的。 如果你发现你在组件或应用程序中重复测量一套指标，那么 `MeterBinder` 的实现也会很有用。





|  | 默认情况下，所有 `MeterBinder` Bean的指标都会自动绑定到Spring管理的 `MeterRegistry`。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.customizing)7.5\. 定制个别指标



如果你需要对特定的 `Meter` 实例进行自定义，你可以使用 `io.micrometer.core.instrument.config.MeterFilter` 接口。





例如，如果你想把所有以 `com.example` 开头的仪表ID的 `mytag.region` 标签重命名为 `mytag.area`，你可以做以下工作。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MyMetricsFilterConfiguration {

    @Bean
    public MeterFilter renameRegionTagMeterFilter() {
        return MeterFilter.renameTag("com.example", "mytag.region", "mytag.area");
    }

}

```







|  | 默认情况下，所有的 `MeterFilter` Bean都自动绑定到Spring管理的 `MeterRegistry`。 请确保使用Spring管理的 `MeterRegistry` 来注册你的指标，而不是使用 `Metrics` 的任何静态方法。 这些方法使用的是不被Spring管理的全局注册表。 |
| --- | --- |





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.customizing.common-tags)7.5.1\. 常见标签（Tag）



通用标签一般用于对运行环境进行维度下钻，如主机、实例、区域、堆栈等。 共用标签适用于所有仪表，可以进行配置，如下例所示。







Properties

Yaml





```
management.metrics.tags.region=us-east-1
management.metrics.tags.stack=prod

```







前面的例子为所有值为 `us-east-1` 和 `prod` 的仪表添加了 `region` 和 `stack` 标签。





|  | 如果你使用Graphite，普通标签的顺序是很重要的。 由于使用这种方法不能保证常见标签的顺序，建议Graphite用户定义一个自定义的 `MeterFilter` 来代替。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.customizing.per-meter-properties)7.5.2\. Per-meter Properties



除了 `MeterFilter` Bean，你还可以使用属性在每个表的基础上应用一组有限的自定义功能。 使用Spring Boot的 `PropertiesMeterFilter`，每个表的定制被应用于以给定名称开头的任何表的ID。 下面的例子过滤掉任何ID以 `example.remote` 开头的仪表。







Properties

Yaml





```
management.metrics.enable.example.remote=false

```







下面的属性允许per-meter的定制。



<caption>Table 1\. Per-meter customizations</caption><colgroup><col><col></colgroup>
| Property | 说明 |
| --- | --- |
| `management.metrics.enable` | 是否接受具有特定ID的Meter。 不接受的Meter将从 `MeterRegistry` 中过滤掉。 |
| `management.metrics.distribution.percentiles-histogram` | 是否发布适合计算可聚集（跨维度）的百分位数近似值的直方图。 |
| `management.metrics.distribution.minimum-expected-value`, `management.metrics.distribution.maximum-expected-value` | 通过钳制预期值的范围，发布更少的直方图桶。 |
| `management.metrics.distribution.percentiles` | 发布在你的应用程序中计算的百分位值 |
| `management.metrics.distribution.expiry`, `management.metrics.distribution.buffer-length` | 通过在环形缓冲区中积累最近的样本，给它们更大的权重，环形缓冲区在可配置的过期后旋转，缓冲区长度为 可配置的缓冲区长度。 |
| `management.metrics.distribution.slo` | 发布一个累积直方图，其中的桶由你的服务水平目标定义。 |



关于 `percentiles-histogram` （百分数-直方图）、`percentiles`（百分数）和 `slo` 背后的概念的更多细节，请参见Micrometer文档中的 [“Histograms and percentiles” （直方图和百分数）部分](https://micrometer.io/docs/concepts#_histograms_and_percentiles)。









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.endpoint)7.6\. 指标端点



Spring Boot提供了一个 `metrics` 端点，你可以诊断性地使用它来检查应用程序收集的指标。该端点默认情况下是不可用的，必须公开。更多细节请参见 [暴露端点](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints.exposing)。





导航到 `/actuator/metrics` 会显示一个可用的仪表名称列表。 你可以通过提供名称作为选择器来深入查看某个特定仪表的信息，例如，`/actuator/metrics/jvm.memory.max`。





|  | 你在这里使用的名字应该与代码中使用的名字一致，而不是在它被运往的监控系统中经过命名惯例规范化后的名字。 换句话说，如果 `jvm.memory.max` 在Prometheus中显示为 `jvm_memory_max`，因为它的蛇形命名惯例，你仍然应该使用 `jvm.memory.max` 作为选择器，在 `metrics` 端点中检查仪表。 |
| --- | --- |





你也可以在URL的末尾添加任意数量的 `tag=KEY:VALUE` 查询参数，以对仪表进行维度下钻?—?例如，`/actuator/metrics/jvm.memory.max?tag=area:nonheap`。





|  | 报告的测量值是所有与仪表名称相匹配的仪表和任何已应用的标签的统计量的 _总和_。 在前面的例子中，返回的 `Value` 统计是堆的 “Code Cache”、“Compressed Class Space” 和 “Metaspace” 区域的最大内存“足迹”之和。 如果你想只看到 “Metaspace” 的最大尺寸，你可以添加一个额外的 `tag=id:Metaspace` --即 `/actuator/metrics/jvm.memory.max?tag=area:nonheap&tag=id:Metaspace`。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.metrics.micrometer-observation)7.7\. 整合 Micrometer Observation



一个 `DefaultMeterObservationHandler` 被自动注册在 `ObservationRegistry` 上，它为每个完成的观察（completed observation）创建度量（metric）。











## [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing)8\. 追踪（Tracing）





Spring Boot Actuator 为 Micrometer Tracing 提供了依赖性管理和自动配置， [Micrometer Tracing](https://micrometer.io/docs/tracing) 是流行的追踪器（tracer）库的一个接口（facade）。





|  | 要了解更多关于 Micrometer Tracing 功能的信息，请参阅其 [参考文档](https://micrometer.io/docs/tracing)。 |
| --- | --- |





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.tracers)8.1\. 支持的追踪器



Spring Boot为以下追踪器提供了自动配置。





*   使用 [Zipkin](https://zipkin.io/) 或 [Wavefront](https://docs.wavefront.com/) 的 [OpenTelemetry](https://opentelemetry.io/)

*   使用 [Zipkin](https://zipkin.io/) 或 [Wavefront](https://docs.wavefront.com/) 的 [OpenZipkin Brave](https://github.com/openzipkin/brave)







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.getting-started)8.2\. 入门



我们需要一个可以用来开始追踪的示例应用程序。就我们的目的而言，“[getting-started.html](https://springdoc.cn/spring-boot/getting-started.html#getting-started.first-application)” 部分所涉及的简单的 “Hello World!” web程序就足够了。我们将使用 `OpenTelemetry` 追踪器和 `Zipkin` 作为追踪后端。





回顾一下，我们的主要应用代码看起来是这样的。







```
@RestController
@SpringBootApplication
public class MyApplication {

    private static final Log logger = LogFactory.getLog(MyApplication.class);

    @RequestMapping("/")
    String home() {
        logger.info("home() has been called");
        return "Hello World!";
    }

    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }

}

```







|  | 在 `home()` 方法中，有一个附加的logger语句，这在后面会很重要。 |
| --- | --- |





现在，我们必须添加以下依赖项。





*   `org.springframework.boot:spring-boot-starter-actuator`

*   `io.micrometer:micrometer-tracing-bridge-otel` - 这是连接 Micrometer Observation API 和 OpenTelemetry 的必要条件。

*   `io.opentelemetry:opentelemetry-exporter-zipkin` - 这是向Zipkin报告 [traces](https://micrometer.io/docs/tracing#_glossary) 所需要的。





添加如下的 application properties:







Properties

Yaml





```
management.tracing.sampling.probability=1.0

```







默认情况下，Spring Boot只对10%的请求进行采样，以防止追踪后端不堪重负。此属性将其切换为100%，这样每个请求都会被发送到跟踪后端。





为了收集和可视化跟踪，我们需要一个运行跟踪的后端。我们在这里使用Zipkin作为我们的跟踪后端。 [Zipkin快速入门指南](https://zipkin.io/pages/quickstart) 提供了如何在本地启动Zipkin的说明。





Zipkin运行后，你可以启动你的应用程序。





如果你打开web浏览器访问 `[localhost:8080](http://localhost:8080/)`，你应该看到以下输出。







 Hello World! 







在幕后，已经为HTTP请求创建了一个 observation，它反过来被桥接到 `OpenTelemetry`，后者向Zipkin报告一个新的跟踪（trace）。





现在，在 `[localhost:9411](http://localhost:9411/)` 打开Zipkin用户界面，点击 "Run Query" 按钮，列出所有收集到的跟踪信息。你应该看到一个追踪。点击 "Show" 按钮，查看该追踪的细节。





|  | 你可以通过将 `logging.pattern.level` 属性设置为 `%5p [${spring.application.name:},%X{traceId:-},%X{spanId:-}]`，在日志中包含当前的跟踪（trace）和 span id。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.tracer-implementations)8.3\. 跟踪器（Tracer）的实现



由于Micrometer Tracer支持多种示踪器的实现，因此Spring Boot可能有多种依赖组合。





所有追踪器的实现都需要 `org.springframework.boot:spring-boot-starter-actuator` 依赖。





#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.tracer-implementations.otel-zipkin)8.3.1\. 使用 Zipkin 的 OpenTelemetry



*   `io.micrometer:micrometer-tracing-bridge-otel` - 这是连接 Micrometer Observation API 和 OpenTelemetry 的必要条件。

*   `io.opentelemetry:opentelemetry-exporter-zipkin` - 这是向Zipkin报告trace所需要的。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.tracer-implementations.otel-wavefront)8.3.2\. 使用 Wavefront 的 OpenTelemetry



*   `io.micrometer:micrometer-tracing-bridge-otel` - 这是连接 Micrometer Observation API 和 OpenTelemetry 的必要条件。

*   `io.micrometer:micrometer-tracing-reporter-wavefront` - 这是向Wavefront报告trace所需要的。







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.tracer-implementations.brave-zipkin)8.3.3\. 使用 Zipkin 的 OpenZipkin Brave



*   `io.micrometer:micrometer-tracing-bridge-brave` - 这是连接 Micrometer Observation API 和 Brave 的必要条件。

*   `io.zipkin.reporter2:zipkin-reporter-brave` - 这是向Zipkin报告 trace 所需要的。





|  | 如果你的项目没有使用Spring MVC或Spring WebFlux，也需要使用 `io.zipkin.reporter2:zipkin-sender-urlconnection` 依赖项。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.tracer-implementations.brave-wavefront)8.3.4\. 使用Wavefront的OpenZipkin Brave



*   `io.micrometer:micrometer-tracing-bridge-brave` - 这是连接测Micrometer Observation API和Brave的必要条件。

*   `io.micrometer:micrometer-tracing-reporter-wavefront` - 这是向Wavefront报告trace所需要的。









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.micrometer-tracing.creating-spans)8.4\. 创建自定义跨度（span）



你可以通过启动一个 observation 来创建你自己的span。为此，将 `ObservationRegistry` 注入到你的组件中。







```
@Component
class CustomObservation {

    private final ObservationRegistry observationRegistry;

    CustomObservation(ObservationRegistry observationRegistry) {
        this.observationRegistry = observationRegistry;
    }

    void someOperation() {
        Observation observation = Observation.createNotStarted("some-operation", this.observationRegistry);
        observation.lowCardinalityKeyValue("some-tag", "some-value");
        observation.observe(() -> {
            // Business logic ...
        });
    }

}

```







这将创建一个名为 "some-operation" 的 observation，标签为，标签为 "some-tag=some-value"。





|  | 如果你想在不创建metric的情况下创建一个span，你需要使用 Micrometer 的 [低级Tracer API](https://micrometer.io/docs/tracing#_using_micrometer_tracing_directly)。 |
| --- | --- |











## [](https://springdoc.cn/spring-boot/actuator.html#actuator.auditing)9\. 审计





一旦Spring Security发挥作用，Spring Boot Actuator就有一个灵活的审计框架，可以发布事件（默认为 “authentication success”, “failure” 和 “access denied” 的异常）。 这一功能对于报告和实施基于认证失败的锁定策略非常有用。





你可以通过在应用程序的配置中提供一个 `AuditEventRepository` 类型的bean来启用审计。 为了方便，Spring Boot提供了一个 `InMemoryAuditEventRepository`。 `InMemoryAuditEventRepository` 的功能有限，我们建议只在开发环境中使用它。 对于生产环境，请考虑创建你自己的替代 `AuditEventRepository` 实现。





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.auditing.custom)9.1\. 定制审计



为了定制发布的安全事件，你可以提供你自己的 `AbstractAuthenticationAuditListener` 和 `AbstractAuthorizationAuditListener` 的实现。





你也可以为你自己的业务事件使用审计服务。 要做到这一点，要么将 `AuditEventRepository` bean注入你自己的组件并直接使用它，要么用Spring的 `ApplicationEventPublisher` 发布 `AuditApplicationEvent`（通过实现 `ApplicationEventPublisherAware`）。











## [](https://springdoc.cn/spring-boot/actuator.html#actuator.http-exchanges)10\. 记录 HTTP Exchange





你可以通过在应用程序的配置中提供一个 `HttpExchangeRepository` 类型的 bean 来启用 HTTP exchange 的记录。为了方便起见，Spring Boot 提供了 `InMemoryHttpExchangeRepository`，默认情况下，它存储了最后100个 request/response exchange。与追踪解决方案（tracing solutions）相比，`InMemoryHttpExchangeRepository` 是有限的，我们建议只在开发环境中使用它。对于生产环境，我们建议使用一个生产就绪的跟踪或观察解决方案，如 `Zipkin` 或 `OpenTelemetry`。另外，你也可以创建你自己的 `HttpExchangeRepository`。





你可以使用 `httpexchanges` 端点来获取存储在 `HttpExchangeRepository` 中的 request/response exchange 的信息。





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.http-exchanges.custom)10.1\. 自定义 HTTP Exchange 记录



要自定义包括在每个记录的 exchange 项目，请使用 `management.httpexchanges.recording.include` 配置属性。





要完全禁止重新编码，请将 `management.httpexchanges.recording.enabled` 设置为 `false`。











## [](https://springdoc.cn/spring-boot/actuator.html#actuator.process-monitoring)11\. 进程监控





在 `spring-boot` 模块中，你可以找到两个创建文件的类，这些文件在进程监控中通常很有用。





*   `ApplicationPidFileWriter` 创建一个包含应用程序PID的文件（默认情况下，在应用程序目录下，文件名为 `application.pid`）。

*   `WebServerPortFileWriter` 创建一个（或多个）文件，包含运行中的Web服务器的端口（默认情况下，在应用程序目录下，文件名为 `application.port`）。





默认情况下，这些写手没有被激活，但你可以启用它们。





*   [通过扩展配置](https://springdoc.cn/spring-boot/actuator.html#actuator.process-monitoring.configuration)

*   [以编程方式实现进程监控](https://springdoc.cn/spring-boot/actuator.html#actuator.process-monitoring.programmatically)





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.process-monitoring.configuration)11.1\. 扩展配置



在 `META-INF/spring.factories` 文件中，你可以激活写入PID文件的listener（一个或者多个）。







 org.springframework.context.ApplicationListener=\
org.springframework.boot.context.ApplicationPidFileWriter,\
org.springframework.boot.web.context.WebServerPortFileWriter 









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.process-monitoring.programmatically)11.2\. 以编程方式实现进程监控



你也可以通过调用 `SpringApplication.addListeners(…?)` 方法并传递适当的 `Writer` 对象来激活一个监听器。 这个方法还可以让你在 `Writer` 构造函数中自定义文件名和路径。











## [](https://springdoc.cn/spring-boot/actuator.html#actuator.cloud-foundry)12\. Cloud Foundry 的支持





Spring Boot 的actuator模块包括额外的支持，当您部署到兼容的 Cloud Foundry 实例时，该支持将被激活。 `/cloudfoundryapplication` 路径为所有 `@Endpoint` Bean提供了另一条安全路线。





扩展支持使 Cloud Foundry 管理 UI（例如您可以用来查看已部署的应用程序的 Web 应用程序）得到 Spring Boot 执行器信息的增强。 例如，应用程序状态页面可以包括完整的健康信息，而不是典型的 “running” 或 “stopped” 状态。





|  | 普通用户无法直接访问 `/cloudfoundryapplication` 路径。 要使用该端点，您必须在请求中传递一个有效的 UAA 令牌。 |
| --- | --- |





### [](https://springdoc.cn/spring-boot/actuator.html#actuator.cloud-foundry.disable)12.1\. 禁用扩展的 Cloud Foundry Actuator 支持



如果您想完全禁用 `/cloudfoundryapplication` 端点，您可以在您的 `application.properties` 文件中添加以下设置。







Properties

Yaml





```
management.cloudfoundry.enabled=false

```









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.cloud-foundry.ssl)12.2\. Cloud Foundry自签名证书



默认情况下，`/cloudfoundryapplication` 端点的安全验证会对各种 Cloud Foundry 服务进行 SSL 调用。 如果您的 Cloud Foundry UAA 或 Cloud Controller 服务使用自签名证书，您需要设置以下属性。







Properties

Yaml





```
management.cloudfoundry.skip-ssl-validation=true

```









### [](https://springdoc.cn/spring-boot/actuator.html#actuator.cloud-foundry.custom-context-path)12.3\. 自定义 Context Path



如果服务器的 context-path 被配置为 `/` 以外的任何内容，则 Cloud Foundry 端点在应用程序的根部不可用。 例如，如果 `server.servlet.context-path=/app`，则 Cloud Foundry 端点在 `/app/cloudfoundryapplication/*` 处可用。





如果您希望 Cloud Foundry 端点始终在 `/cloudfoundryapplication/*` 处可用，无论服务器的上下文路径如何，您需要在您的应用程序中明确配置。 该配置因使用的 Web 服务器不同而不同。 对于 Tomcat，您可以添加以下配置。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class MyCloudFoundryConfiguration {

    @Bean
    public TomcatServletWebServerFactory servletWebServerFactory() {
        return new TomcatServletWebServerFactory() {

            @Override
            protected void prepareContext(Host host, ServletContextInitializer[] initializers) {
                super.prepareContext(host, initializers);
                StandardContext child = new StandardContext();
                child.addLifecycleListener(new Tomcat.FixContextListener());
                child.setPath("/cloudfoundryapplication");
                ServletContainerInitializer initializer = getServletContextInitializer(getContextPath());
                child.addServletContainerInitializer(initializer, Collections.emptySet());
                child.setCrossContext(true);
                host.addChild(child);
            }

        };
    }

    private ServletContainerInitializer getServletContextInitializer(String contextPath) {
        return (classes, context) -> {
            Servlet servlet = new GenericServlet() {

                @Override
                public void service(ServletRequest req, ServletResponse res) throws ServletException, IOException {
                    ServletContext context = req.getServletContext().getContext(contextPath);
                    context.getRequestDispatcher("/cloudfoundryapplication").forward(req, res);
                }

            };
            context.addServlet("cloudfoundry", servlet).addMapping("/*");
        };
    }

}

```













## [](https://springdoc.cn/spring-boot/actuator.html#actuator.whats-next)13\. 接下来读什么





你可能想读一下 [Graphite](https://graphiteapp.org/) 等图形工具。





否则，你可以继续阅读 [“部署选项”](https://springdoc.cn/spring-boot/deployment.html#deployment) ，或者跳到前面去了解有关Spring Boot [构建工具插](https://springdoc.cn/spring-boot/build-tool-plugins.html#build-tool-plugins)件的一些深入信息。







