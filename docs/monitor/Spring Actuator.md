## 前言

在微服务系统架构中，服务的监控是必不可少的。目前大多数微服务应用又是基于Spring Cloud系列，也可以说是基于Spring Boot系列的。此时使用Spring Boot Actuator来进行微服务的监控，不仅功能全面，而且非常方便。

在上篇文章《[Spring Boot Actuator集成，难的是灵活运用！](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FBaNQWygQb8UXxktrXetOcw "https://mp.weixin.qq.com/s/BaNQWygQb8UXxktrXetOcw")》中我们已经介绍了如何将Actuator集成到Spring Boot项目中，并且介绍了如何自定义Endpoint（端点）。有朋友留言说不够深入，那么，本篇文章呢，我们将介绍Actuator原生端点的功能及基本使用场景。

## Endpoints 介绍

Actuator中所谓的 Endpoints （翻译为端点）提供了外部来与应用程序进行访问和交互的功能。 比如说/health端点提供了应用健康情况的信息，metrics 端点提供了应用程序的指标（JVM 内存使用、系统CPU使用等）信息。

Actuator原生的端点可分为三大类：

*   应用配置类：获取应用程序中加载的应用配置、环境变量、自动化配置报告等与Spring Boot应用密切相关的配置类信息。
*   度量指标类：获取应用程序运行过程中用于监控的度量指标，比如：内存信息、线程池信息、HTTP请求统计等。
*   操作控制类：提供了对应用的关闭等操作类功能。

不同版本的Actuator提供的原生端点有所出入，在使用的过程中最好以所使用版本的官方文档为准。同时，每个原生的端点都可以通过配置来单独的禁用或启用。

而在Actuator 2.x 中默认端点增加了/actuator前缀，同时默认只暴露的两个端点为/actuator/health和 /actuator/info。关于端点暴露的配置，可参考前一篇文章。下面基于Spring Boot 2.2.2.RELEASE版本来重点讲解每个端点的功能和应用场景。

## actuator端点

Actuator 2.x新增的默认端点，用于展示目前应用中暴露出来的端点汇总，你可以理解为可用端点的目录。

访问URL：[http://localhost:8080/actuator](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator "http://localhost:8080/actuator") ，对应展示结果如下图：

![image-20230530233537559](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233537559.png)

上述只展示了一部分的端点，返回结果为，这里采用了浏览器的插件-Handler进行了格式美化。通过actuator可以直观的看出目前开放了哪些端点，以及这些端点的名称和请求路径。

下面我们就按照显示actuator端点展示的列表逐一介绍。

## auditevents端点

auditevents端点用于显示应用暴露的审计事件 (比如认证进入、订单失败)，即使我们打开了所有端点，默认情况下也是看不到这个端点的。因为使用它的前提是需要在Spring容器中存在一个类型为AuditEventRepository的Bean的。

查看了网络上大多数教程，基本上都是介绍了auditevents端点功能，而未展示具体实例。笔者经过多方尝试，终于给大家写了一个案例出来。

首先涉及到权限认证，需要先引入spring-boot-starter-security依赖：

````
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-security</artifactId>
</dependency>` 
````

单纯添加这个依赖还是不够的，还需要加入security的配置，不然AuthorizationAuditListener,AuthenticationAuditListener 监听什么事件呢? 因此,我们加入如下代码：


````
@Configuration
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

 @Override
 protected void configure(AuthenticationManagerBuilder auth) throws Exception {

 auth.inMemoryAuthentication()
 .withUser("admin")
 .password(bcryptPasswordEncoder().encode("admin"))
 .roles("admin");
 }
````

````
 @Bean
 public PasswordEncoder bcryptPasswordEncoder() {
 return new BCryptPasswordEncoder();
 }
}
````

这里采用了security默认的登录界面和权限控制，也就是说所有的访问都需要进行登录。而登录的用户名和密码均为admin。

另外，前面提到需要用到AuditEventRepository的Bean，这里初始化一个对应的Bean：




````
@Configuration
public class AuditEventConfig {

 @Bean
 public InMemoryAuditEventRepository repository(){
 return new InMemoryAuditEventRepository();
 }
}
````

InMemoryAuditEventRepository是AuditEventRepository接口的唯一实现类。

重启项目，auditevents端点便可用了。访问[http://localhost:8080/actuator](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator "http://localhost:8080/actuator") ,此时会跳转到Security提供的登录页面：

![image-20230530233604253](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233604253.png)

输入代码中指定的用户名和密码，登录成功，跳转到/actuator页面：

![image-20230530233625068](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233625068.png)

可以看到auditevents端点已经成功显示出来了。新开页面访问[http://localhost:8080/actuator/auditevents](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fauditevents "http://localhost:8080/actuator/auditevents") ，展示内容如下：

![image-20230530233716752](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233716752.png)

可以看到已经记录下了权限相关的事件，其中第一次事件是我们直接访问actuator端点时，由于之前为做权限认真，所以事件类型为"AUTHORIZATION_FAILURE"，也就是认证失败。此时跳转到登录页面，然后在登录页面输入用户名和密码，登录成功，对应的事件为"AUTHENTICATION_SUCCESS"。

也就是说auditevents记录了用户认证登录系统相关的事件信息，包括时间戳、认证用户、事件类型、访问地址、sessionId等。

示例源码地址：[github.com/secbr/sprin…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fsecbr%2Fspringboot-all%2Ftree%2Fmaster%2Fspringboot-actuator-auditevents "https://github.com/secbr/springboot-all/tree/master/springboot-actuator-auditevents") 。

## beans端点

/beans端点会返回Spring容器中所有bean的别名、类型、是否单例、依赖等信息。

访问路径为[http://localhost:8080/actuator/beans](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fbeans "http://localhost:8080/actuator/beans") ，范围结果如下：

![image-20230530233748286](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233748286.png)

这个端点会展示目前Spring容器中初始化的所有Bean，试想一下，如果你配置了一个Bean，但不确定是否成功实例化，是不是就可以通过这个端口查询一下呢？

我们在项目中定义一个TestController，并注入一个UserService：


````
@Controller
public class TestController {

 @Resource
 private UserService userService;
}
````

重新启动并访问该端点，会看到如下信息：

![image-20230530233805161](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233805161.png)

可以看到TestController被实例化了，而且依赖于UserService。

## caches端点

caches端点主要用于暴露应用程序中的缓冲。这里以Spring Boot提供的Cache组件来展示一下实例。

在项目中集成spring-boot-starter-cache，引入依赖：


````
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
````

然后在启动类上添加@EnableCaching，开启缓存功能。

定义一个CacheController，对其方法queryAll使用缓存机制：


````
@RestController
public class CacheController {

 @RequestMapping("/queryAll")
 @Cacheable(value = "queryAll")
 public Map<String, String> queryAll() {
 Map<String, String> map = new HashMap<>();
 map.put("1", "Tom");
 map.put("2", "Steven");
 return map;
 }
}
````

这里使用@Cacheable注解来实现缓存功能，缓存的key为queryAll。此时，访问[http://localhost:8080/actuator/caches](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fcaches "http://localhost:8080/actuator/caches") ，会展示缓存的根内容，但里面并没有缓存。

访问一下[http://localhost:8080/queryAll](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2FqueryAll "http://localhost:8080/queryAll") ，也就是触发一下缓存内容的生成。此时再访问上面的链接，便可以看到应用程序中的缓存信息了：

![image-20230530233852486](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233852486.png)

可以看到，返回的内容不仅展示了应用程序的缓存管理器，同时也展示了缓存的Key和缓存数据存储类型信息。

## caches-cache端点

caches-cache端点是对上面caches端点的扩展，caches端点展示的所有的缓存信息。如果想直接看出具体的一个缓存信息，则可以使用caches-cache端点。

访问的URL为：[http://localhost:8080/actuator/caches/{cache}](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fcaches%2F%257Bcache%257D "http://localhost:8080/actuator/caches/%7Bcache%7D") ，其中大括号内的值可以替换为缓存的key。




`http://localhost:8080/actuator/caches/queryAll`

将上面的占位符换成queryAll（缓存的key），执行结果如下：

![image-20230530233906164](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233906164.png)

可以看出，只查询除了指定的缓存信息，包括缓存管理器、缓存名称（key），缓存的存储类型。

## health端点

health端点用来检查应用的运行状态，最高频使用的一个端点。检查应用实例的运行状态以及应用不”健康“的原因，比如数据库连接、磁盘空间不够等。

访问地址：[http://localhost:8080/actuator/health](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fhealth "http://localhost:8080/actuator/health")

展示结果：

`{
"status": "UP"
}`

上面的实例过于简单，在项目中把数据库给集成进去：

`<!--数据库连接相关-->

````
<dependency>
 <groupId>org.springframework.boot</groupId>
 <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>
<dependency>
 <groupId>mysql</groupId>
 <artifactId>mysql-connector-java</artifactId>
</dependency>` 
````

然后在application配置文件中进行配置：



```
spring:
 datasource:
 url: jdbc:mysql://xxx:3333/xxx?characterEncoding=utf8&serverTimezone=Asia/Shanghai
 username: root
 password: root
 driver-class-name: com.mysql.cj.jdbc.Driver
```

同时，我们要在application配置文件中配置一下management.endpoint.health.show-details的值。该属性有三个可选项：

*   never ：不展示详细信息，up 或者 down 的状态，默认配置；
*   when-authorized：详细信息将会展示给通过认证的用户。授权的角色可以通过management.endpoint.health.roles 配置；
*   always：对所有用户暴露详细信息。

默认值是never，所以我们直接访问看到的只有UP或DOWN。现在集成了数据库，同时把该项值配置为always，看一下详情：

![image-20230530233934501](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233934501.png)

可以看到整体状态为UP，其中下面的三个组件均为UP，而数据库是MYSQL，检查数据库的语句为“SELECT 1”。同时，还展示了磁盘信息和ping的状态。

现在我们把数据库的用户名和密码故意改错，重启访问可得：

![image-20230530233951145](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530233951145.png)

状态为DOWN，组件中db出现了问题，状态为DOWN。问题的详情在error中展示，可以看出是建立连接时出错了。在实践中，我们可以通过health端口监控数据库、Redis、MongoDB、磁盘等健康情况。Actuator预定义的处理类为：DataSourceHealthIndicator, DiskSpaceHealthIndicator, MongoHealthIndicator, RedisHealthIndicator等。

而且每个指标都可以单独的进行开启和关闭，以数据库的为例：


````
management:
 health:
 db:
 enabled: true` 
````

## info端点

/info 端点用来查看配置文件 application中以info开头的配置信息，默认情况下 application中并没有 info 节点配置，所以默认为空。

application中添加如下配置：



````
info:
 user:
 type: 公众号
 name: 程序新视界
 wechat: zhuan2quan
````

访问[http://localhost:8080/actuator/info](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Finfo "http://localhost:8080/actuator/info") ，展示结果如下：

![image-20230530234019487](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234019487.png)

## conditions端点

Spring Boot提供了自动配置功能，使用起来非常方便。但这些自动配置类是什么情况下生效的，是否生效是比较难排查的。此时，可以使用 conditions 在应用运行时查看某个配置类在什么条件下生效，或为什么没有生效。

访问URL：[http://localhost:8080/actuator/conditions](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fconditions "http://localhost:8080/actuator/conditions") ，部分返回信息如下：

![image-20230530234053134](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234053134.png)

可以看到某个自动配置类对应的生效条件和提示信息。

## shutdown端点

shutdown端点属于操作控制类端点，可以优雅关闭 Spring Boot 应用。需要在配置文件中开启：


````
management:
 endpoint:
 shutdown:
 enabled: true
````

该端点只支持POST请求，执行命令及返回结果如下：


```
curl -X POST "http://localhost:8080/actuator/shutdown" 
{
 "message": "Shutting down, bye..."
}
```

执行之后，会发现应用程序已经被关闭了。由于该端点会关闭应用程序，因此使用是需要小心。

## configprops端点

在Spring Boot项目中，我们经常会用到@ConfigurationProperties注解来批量注入一些属性，而configprops端点就是用来显示这些被该注解标注的配置类。

比如前面的info配置，我们就可以定义一个类InfoProperties：


````
@Component
@ConfigurationProperties(prefix = "info")
public class InfoProperties {

 private String type;

 private String name;

 private String wechat;
  
 // 省略getter/setter 
}
````

访问URL：[http://localhost:8080/actuator/configprops](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fconfigprops "http://localhost:8080/actuator/configprops") ，部分信息如下：

![image-20230530234110515](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234110515.png)

不仅可以看到系统中默认集成的配置类信息，还可以看到我们自定义的配置类信息。这里需要注意的是对应的类需要进行实例化（@Component）这里才能够看到。

我们自定义的类中返回了Bean的名称、配置前缀。上面的ProjectInfoProperties还返回了属性信息。

## env端点

env端点用于获取全部环境属性，包括application配置文件中的内容、系统变量等。

访问URL：[http://localhost:8080/actuator/env，返回部分信息：](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fenv%25EF%25BC%258C%25E8%25BF%2594%25E5%259B%259E%25E9%2583%25A8%25E5%2588%2586%25E4%25BF%25A1%25E6%2581%25AF%25EF%25BC%259A "http://localhost:8080/actuator/env%EF%BC%8C%E8%BF%94%E5%9B%9E%E9%83%A8%E5%88%86%E4%BF%A1%E6%81%AF%EF%BC%9A")

![image-20230530234200949](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234200949.png)

## env-toMatch端点

env-toMatch端点与caches和caches-cache类似，一个是获取所有的，一个是获取指定的。这里的env-toMatch端点是获取指定key的环境变量属性。

基本格式为：[http://localhost:8080/actuator/env/{toMatch}。](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fenv%2F%257BtoMatch%257D%25E3%2580%2582 "http://localhost:8080/actuator/env/%7BtoMatch%7D%E3%80%82") 实例URL：[http://localhost:8080/actuator/env/info.user.name](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fenv%2Finfo.user.name "http://localhost:8080/actuator/env/info.user.name") ，返回结果如下图：

![image-20230530234238073](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234238073.png)

返回数据信息包括该属性的来源、value值等信息。

## loggers端点

/loggers 端点暴露了程序内部配置的所有 logger 的信息，包括不同的package、不同的类的日志级别信息。

访问URL：[http://localhost:8080/actuator/loggers](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Floggers "http://localhost:8080/actuator/loggers") ，部分返回结果：

![image-20230530234301625](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234301625.png)

## loggers-name端点

loggers-name端点也是logger端点的细分，可以通过name访问某一个logger。

基本请求格式：[http://localhost:8080/actuator/loggers/{name}](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Floggers%2F%257Bname%257D "http://localhost:8080/actuator/loggers/%7Bname%7D") 示例请求URL：[http://localhost:8080/actuator/loggers/com.secbro2.SpringbootActuatorApplication](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Floggers%2Fcom.secbro2.SpringbootActuatorApplication "http://localhost:8080/actuator/loggers/com.secbro2.SpringbootActuatorApplication") ，返回结果如下：



`{
"configuredLevel": null,
"effectiveLevel": "INFO"
}`

可以看出，启动类的日志级别为INFO。

## heapdump端点

heapdump端点会返回一个JVM 堆dump，通过JVM自带的监控工具VisualVM可打开此文件查看内存快照。这是内存优化，基于堆栈层面进行排查的利器。

访问URL：[http://localhost:8080/actuator/heapdump](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fheapdump "http://localhost:8080/actuator/heapdump") 。Mac操作系统下浏览器访问会下载一个名字为heapdump的文件，无后缀，30M。

命令行执行jvisualvm命令，打开VisualVM，依次点击“文件”、“装入”，记得文件类型要选择“堆Dump(_.hprof,_.*)”，然后选择heapdump。

![image-20230530234346098](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234346098.png)

此时便可以通过工具来进行堆栈信息的分析了。对于线上问题的分析提供了极为便利的方式。

## threaddump端点

/threaddump 端点会生成当前线程活动的快照。在日常定位问题的时候查看线程的情况非常有用，主要展示了线程名、线程ID、线程的状态、是否等待锁资源等信息。

访问URL：[http://localhost:8080/actuator/threaddump](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fthreaddump "http://localhost:8080/actuator/threaddump") ，部分返回结果：

![image-20230530234405331](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234405331.png)

我们可通过线程快照来排查生产环境的问题。

## metrics端点

/metrics 端点用来暴露当前应用的各类重要度量指标，比如：内存信息、线程信息、垃圾回收信息、tomcat、数据库连接池等。2.x版本这里只显示了一个指标的列表。

访问URL：[http://localhost:8080/actuator/metrics](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fmetrics "http://localhost:8080/actuator/metrics") 。

````
{
 "names": [
 "jvm.memory.max",
 "jvm.threads.states",
 "jvm.gc.pause",
 "http.server.requests",
 "process.files.max",
 "jvm.gc.memory.promoted",
 "system.load.average.1m",
 "jvm.memory.used",
 "jvm.gc.max.data.size",
 "jvm.memory.committed",
 "system.cpu.count",
 "logback.events",
 "jvm.buffer.memory.used",
 "tomcat.sessions.created",
 "jvm.threads.daemon",
 "system.cpu.usage",
 "jvm.gc.memory.allocated",
 "tomcat.sessions.expired",
 "jvm.threads.live",
 "jvm.threads.peak",
 "process.uptime",
 "tomcat.sessions.rejected",
 "process.cpu.usage",
 "jvm.classes.loaded",
 "jvm.classes.unloaded",
 "tomcat.sessions.active.current",
 "tomcat.sessions.alive.max",
 "jvm.gc.live.data.size",
 "process.files.open",
 "jvm.buffer.count",
 "jvm.buffer.total.capacity",
 "tomcat.sessions.active.max",
 "process.start.time"
 ]
}
````

/metrics端点可以提供应用运行状态的完整度量指标报告，这项功能非常的实用，但是对于监控系统中的各项监控功能，它们的监控内容、数据收集频率都有所不同，如果我们每次都通过全量获取报告的方式来收集，略显粗暴。官方也是可能是处于此方面的考虑，在Spring Boot 2.x之后，/metrics端点只显示了指标的列表。

如果需要查看具体的某项指标，则可通过/metrics-requiredMetricName端点来实现。

## metrics-requiredMetricName端点

metrics-requiredMetricName端点，用于访问指定指标的报告，一般会配合/metrics端点先查出指标列表，然后再查询具体的某个指标。

基本格式：[http://localhost:8080/actuator/metrics/{requiredMetricName}。](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fmetrics%2F%257BrequiredMetricName%257D%25E3%2580%2582 "http://localhost:8080/actuator/metrics/%7BrequiredMetricName%7D%E3%80%82") 实例URL：[http://localhost:8080/actuator/metrics/jvm.memory.max](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fmetrics%2Fjvm.memory.max "http://localhost:8080/actuator/metrics/jvm.memory.max") ，返回结果如下：


````
{
 "name": "jvm.memory.max",
 "description": "The maximum amount of memory in bytes that can be used for memory management",
 "baseUnit": "bytes",
 "measurements": [
 {
 "statistic": "VALUE",
 "value": 5606211583
 }
 ],
 "availableTags": [
 {
 "tag": "area",
 "values": [
 "heap",
 "nonheap"
 ]
 },
 {
 "tag": "id",
 "values": [
 "Compressed Class Space",
 "PS Survivor Space",
 "PS Old Gen",
 "Metaspace",
 "PS Eden Space",
 "Code Cache"
 ]
 }
 ]
}
````

上述结果展示了最大可用内存的情况。其他相关指标的展示替换对应的名字进行查看即可。

## scheduledtasks端点

/scheduledtasks端点用于展示应用中的定时任务信息。

先在项目中构建两个定时任务，首先在启动类上添加@EnableScheduling开启定时任务功能。然后创建定时任务类：

````
@Component
public class MyTask {

 @Scheduled(cron = "0/10 * * * * *")
 public void work() {
 System.out.println("I am a cron job.");
 }

 @Scheduled(fixedDelay = 10000)
 public void work1() {
 System.out.println("I am a fixedDelay job.");
 }
}
````

其中定义了两种类型的定时任务，work是基于cron实现的定时任务，work1是基于fixedDelay实现的定时任务。

访问URL：[http://localhost:8080/actuator/scheduledtasks](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fscheduledtasks "http://localhost:8080/actuator/scheduledtasks") ，返回结果信息如下：

```
{
 "cron": [
 {
 "runnable": {
 "target": "com.secbro2.job.MyTask.work"
 },
 "expression": "0/10 * * * * *"
 }
 ],
 "fixedDelay": [
 {
 "runnable": {
 "target": "com.secbro2.job.MyTask.work1"
 },
 "initialDelay": 0,
 "interval": 10000
 }
 ],
 "fixedRate": [],
 "custom": []
}
```

可以看到，通过该端点可以明确的知道当前应用中定义的定时任务，以及执行模式和频次。

## mappings端点

/mappings端点用于描述全部的 URI 路径，以及和控制器的映射关系。这个功能算是比较常用的了，如果想系统的查看URL对应的Controller及方法，可以使用此端点。

访问URL：[http://localhost:8080/actuator/mappings](https://link.juejin.cn?target=http%3A%2F%2Flocalhost%3A8080%2Factuator%2Fmappings "http://localhost:8080/actuator/mappings") ，部分返回结果如下：

![image-20230530234501440](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230530234501440.png)

至此关于Spring Boot Actuator提供的所有端点介绍完毕。

## 小结

本文通过对Spring Boot Actuator提供所有端点构建实例并演示结果，本文大多数内容和实例都是全网第一手资料。该框架对排查线上问题，性能优化等都有极大的帮助。而在写本文的过程中也越来越惊叹Actuator的功能之强大，强烈推荐用起来。

## 参考链接

作者：程序新视界
链接：https://juejin.cn/post/6984550846876876814
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

