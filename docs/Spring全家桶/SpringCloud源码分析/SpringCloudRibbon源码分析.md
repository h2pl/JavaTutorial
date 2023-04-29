**学习目标**

1.  推导Ribbon的核心流程
2.  手写一个简易版的Ribbon
3.  通过源码验证推导的流程
    **第1章 核心流程推导**
    其实Ribbon的核心流程很简单，我们在使用过程中无非就是引入了一个spring-cloud-starter-netflix-ribbon的jar包，然后在程序启动的时候注入了一个RestTemplate对象，在该对象上面增加了一个@LoadBalanced注解，然后在通过RestTemplate对象去调用URL的时候就能根据不同的负载均衡策略去调不同的服务，那这个注解，或者说这个jar包到底做了什么事情呢？

首先我们要明白，spring-cloud-starter-netflix-ribbon这个jar包从名字上来看就知道，它是基于starter组件的，那它肯定是依据springboot的自动装配原理，在容器启动的时候提供了一个自动配置类，将我们所需要用的对象注入到IoC容器里面去了，这点毋庸置疑。

然后当要用RestTemplate对象去请求目标服务的时候，这个时候，我们肯定是要用真实的IP和端口来替换服务名。这一步其实就是核心步骤，它要怎么做呢？怎么在真正请求之前将地址和端口狸猫换太子换成真实的呢？

在我们日常开发中，我们应该知道，有一个过滤器和一个拦截器其实可以做到这个操作，对吧，所以，实际上就是在获取RestTemplate对象的时候，将该对象里面添加了一个拦截器，当RestTemplate对象执行某个方法的时候，都会去拦截器里面执行一遍。然后就完事了。

具体的流程图推导如下：![SpringCloud系列—Ribbon源码分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1790fb324161913dd79098940f8102d652cd86.jpg "SpringCloud系列—Ribbon源码分析-开源基础软件社区")**第2章 简易版Ribbon实现**
根据上面的推导过程，我们接下来来实现一个简易版的Ribbon。

具体的步骤我们要有清晰的思路：

1.首先要实现一个starter组件，集成我们springboot，让springboot在启动的时候可以拿到相应的RestTemplate的bean对象。创建一个Maven的quickstart项目

2.然后引包



```
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>com.example</groupId>
  myribbon-spring-cloud-starter
  <version>1.0-SNAPSHOT</version>

  <name>myribbon-spring-cloud-starter</name>
  <!-- FIXME change it to the project's website -->
  <url>http://www.example.com</url>

  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    <spring-boot.version>2.3.2.RELEASE</spring-boot.version>
  </properties>

  <!-- RestTemplate必须要用到SpringMVC -->
  <dependencies>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      spring-boot-starter-web
      <version>${spring-boot.version}</version>
    </dependency>

    <dependency>
      <groupId>org.springframework.boot</groupId>
      spring-boot-starter-test
      <version>${spring-boot.version}</version>
    </dependency>

    <!-- 这个包里面集成了一些核心接口 -->
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      spring-cloud-commons
      <version>2.2.6.RELEASE</version>
    </dependency>
  </dependencies>

  <build>
    <pluginManagement><!-- lock down plugins versions to avoid using Maven defaults (may be moved to parent pom) -->
      <plugins>
        <!-- clean lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#clean_Lifecycle -->
        <plugin>
          maven-clean-plugin
          <version>3.1.0</version>
        </plugin>
        <!-- default lifecycle, jar packaging: see https://maven.apache.org/ref/current/maven-core/default-bindings.html#Plugin_bindings_for_jar_packaging -->
        <plugin>
          maven-resources-plugin
          <version>3.0.2</version>
        </plugin>
        <plugin>
          maven-compiler-plugin
          <version>3.8.0</version>
        </plugin>
        <plugin>
          maven-surefire-plugin
          <version>2.22.1</version>
        </plugin>
        <plugin>
          maven-jar-plugin
          <version>3.0.2</version>
        </plugin>
        <plugin>
          maven-install-plugin
          <version>2.5.2</version>
        </plugin>
        <plugin>
          maven-deploy-plugin
          <version>2.8.2</version>
        </plugin>
        <!-- site lifecycle, see https://maven.apache.org/ref/current/maven-core/lifecycles.html#site_Lifecycle -->
        <plugin>
          maven-site-plugin
          <version>3.7.1</version>
        </plugin>
        <plugin>
          maven-project-info-reports-plugin
          <version>3.0.0</version>
        </plugin>
      </plugins>
    </pluginManagement>
  </build>
</project>
```









3.创建配置类



```
@Configuration
public class MyRibbonAutoConfiguration {
    //简易版的Ribbon是靠这个类去完成负载均衡算法以及真实的ip和端口替换的
    @Bean
    public LoadBalancerClient loadBalancerClient(){
        return new MyLoadBalancerClient();
    }
    //收集所有带MyLoadBalanced注解的RestTemplate对象
    @MyLoadBalanced
    @Autowired(required = false)
    private List<RestTemplate> restTemplates = Collections.emptyList();
    @Bean
    @ConditionalOnMissingBean
    public LoadBalancerRequestFactory loadBalancerRequestFactory(
            LoadBalancerClient loadBalancerClient) {
        return new LoadBalancerRequestFactory(loadBalancerClient);
    }
    //这就是核心的拦截器
    @Bean
    public MyLoadBalancerInterceptor myLoadBalancerInterceptor(
            LoadBalancerClient loadBalancerClient,
            LoadBalancerRequestFactory requestFactory){
        return new MyLoadBalancerInterceptor(loadBalancerClient,requestFactory);
    }
    //收集到的RestTemplate对象，在这里都会配置一个拦截器
    @Bean
    public SmartInitializingSingleton smartInitializingSingleton(
            final MyLoadBalancerInterceptor myLoadBalancerInterceptor){
        return ()->{
            for (RestTemplate restTemplate : MyRibbonAutoConfiguration.this.restTemplates) {
                List<ClientHttpRequestInterceptor> list = new ArrayList<>(
                        restTemplate.getInterceptors());
                list.add(myLoadBalancerInterceptor);
                restTemplate.setInterceptors(list);
            }
        };
    }
}
```









4.创建MyLoadBalancerClient



```
public class MyLoadBalancerClient implements LoadBalancerClient {
    @Autowired
    AbstractEnvironment environment;
    //1.在拦截器里面会调用这个方法
    @Override
    public <T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException {
        ServiceInstance server = this.choose(serviceId);
        return execute(serviceId, server, request);
    }	
    //3.真正执行Http请求
    @Override
    public <T> T execute(String serviceId, ServiceInstance serviceInstance, LoadBalancerRequest<T> request) throws IOException {
        T returnVal = null;
        try {
            returnVal = request.apply(serviceInstance);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return returnVal;
    } 
	//4.这一步就是用真实的ip和port替换服务名
    @Override
    public URI reconstructURI(ServiceInstance instance, URI original) {
        String host = instance.getHost();
        int port = instance.getPort();
        if (host.equals(original.getHost())
                && port == original.getPort()
                ) {
            return original;
        }
        try {
            StringBuilder sb = new StringBuilder();
            sb.append("http").append("://");
            if (!Strings.isNullOrEmpty(original.getRawUserInfo())) {
                sb.append(original.getRawUserInfo()).append("@");
            }
            sb.append(host);
            if (port >= 0) {
                sb.append(":").append(port);
            }
            sb.append(original.getRawPath());
            if (!Strings.isNullOrEmpty(original.getRawQuery())) {
                sb.append("?").append(original.getRawQuery());
            }
            if (!Strings.isNullOrEmpty(original.getRawFragment())) {
                sb.append("#").append(original.getRawFragment());
            }
            URI newURI = new URI(sb.toString());
            return newURI;
        }catch (URISyntaxException e){
            throw new RuntimeException(e);
        }
    }
    //2.负载均衡算法在这一步进行服务的ip和端口选择，我这里用的是随机算法
    @Override
    public ServiceInstance choose(String serviceId) {
        Server instance = new Server(serviceId,null,"127.0.0.1",8080);
        String sr = environment.getProperty(serviceId+".ribbon.listOfServers");
        if (!StringUtils.isEmpty(sr)){
            String[] arr = sr.split(",",-1);
            Random selector = new Random();
            int next = selector.nextInt(arr.length);
            String a = arr[next];
            String[] srr = a.split(":",-1);
            instance.setHost(srr[0]);
            instance.setPort(Integer.parseInt(srr[1]));
        }
        return instance;
    }
}
```









5.拦截器的逻辑其实很简单，就是在真正发送http请求之前先来执行我的逻辑



```
public class MyLoadBalancerInterceptor implements ClientHttpRequestInterceptor {
    private LoadBalancerClient loadBalancerClient;
    private LoadBalancerRequestFactory requestFactory;
    public MyLoadBalancerInterceptor(LoadBalancerClient loadBalancerClient,
                                   LoadBalancerRequestFactory requestFactory) {
        this.loadBalancerClient = loadBalancerClient;
        this.requestFactory = requestFactory;
    }
    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body, ClientHttpRequestExecution execution) throws IOException {
        final URI originalUri = request.getURI();
        String serviceName = originalUri.getHost();
        return this.loadBalancerClient.execute(serviceName,
                this.requestFactory.createRequest(request, body, execution));
    }
}
```









6.定义自己的注解



```
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface MyLoadBalanced {
}
```









7.定义一个自己的Server实体类



```
public class Server implements ServiceInstance {
    private String serviceId;
    private String instanceId;
    private String host;
    private int port;
    public Server(String serviceId, String instanceId, String host, int port) {
        this.serviceId = serviceId;
        this.instanceId = instanceId;
        this.host = host;
        this.port = port;
    }
    public Server() {
    }
    public void setServiceId(String serviceId) {
        this.serviceId = serviceId;
    }
    public void setInstanceId(String instanceId) {
        this.instanceId = instanceId;
    }
    public void setHost(String host) {
        this.host = host;
    }
    public void setPort(int port) {
        this.port = port;
    }
    @Override
    public String getInstanceId() {
        return null;
    }
    @Override
    public String getServiceId() {
        return null;
    }
    @Override
    public String getHost() {
        return host;
    }
    @Override
    public int getPort() {
        return port;
    }
    @Override
    public boolean isSecure() {
        return false;
    }
    @Override
    public URI getUri() {
        return null;
    }
    @Override
    public Map<String, String> getMetadata() {
        return null;
    }
    @Override
    public String getScheme() {
        return null;
    }
}
```









8.写spring.factories文件



```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
    com.example.config.MyRibbonAutoConfiguration
```









9.打成jar包，测试，测试的时候，需要加配置文件



```
<serverName>.ribbon.listOfServers=127.0.0.1:2223,127.0.0.1:2222
```









**第3章 源码验证**
**3.1 @LoadBalanced**
从上节课的代码看，我们只是在RestTemplate上面加了一个@LoadBalance,就可以实现负载均衡了，可是我们点击进入@LoadBalance看了一下，在这个注解里面有一个@Qualifier注解。该注解限定哪个bean应该被自动注入。当Spring无法判断出哪个bean应该被注入时，@Qualifier注解有助于消除歧义bean的自动注入，例子见代码。



```
/**
 * Annotation to mark a RestTemplate or WebClient bean to be configured to use a
 * LoadBalancerClient.
 * @author Spencer Gibb
 */
@Target({ ElementType.FIELD, ElementType.PARAMETER, ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@Qualifier
public @interface LoadBalanced {

}
```









从注释中可以知道，这个注解是用来给RestTemplate做标记，以使用负载均衡客户端（LoadBalancerClient）来配置它。所以，我们在生成的RestTemplate的bean上添加这么一个注解，这个bean就会配置LoadBalancerClient。

**3.2 LoadBalancerClient**
那么，就再看下LoadBalancerClient的代码：



```
public interface LoadBalancerClient extends ServiceInstanceChooser {
	<T> T execute(String serviceId, LoadBalancerRequest<T> request) throws IOException;
	<T> T execute(String serviceId, ServiceInstance serviceInstance,
			LoadBalancerRequest<T> request) throws IOException;
	URI reconstructURI(ServiceInstance instance, URI original);
}
public interface ServiceInstanceChooser {
	ServiceInstance choose(String serviceId);
}
```









LoadBalancerClient是一个接口，里面有三个方法。

ServiceInstance choose(String serviceId);从方法名上就可以看出，是根据传入的serviceId（服务名），从负载均衡器中选择一个服务实例，服务实例通过ServiceInstance类来表示。
execute方法，使用从负载均衡器中选择的服务实例来执行请求内容。
URI reconstructURI(ServiceInstance instance, URI original);方法，是重新构建一个URI的，还记得我们在代码中，通过RestTemplate请求服务时，写的是服务名吧，这个方法就会把这个请求的URI进行转换，返回host+port，通过host+port的形式去请求服务。
**3.3 自动装配**
当springboot启动之后，会通过自动装配自动去
spring-cloud-netflix-ribbon这个jar包的META-INF目录找spring.factories文件，并且将RibbonAutoConfiguration配置类进行注入。而在RibbonAutoConfiguration配置类中因为存在@AutoConfigureBefore注解，所以又会加载LoadBalancerAutoConfiguration配置类。在LoadBalancerAutoConfiguration类中，spring容器会将所有被@LoadBalance注解修饰的bean注入到IOC容器中



```
@LoadBalanced
@Autowired(required = false)
private List<RestTemplate> restTemplates = Collections.emptyList();
```









同时，在LoadBalancerAutoConfiguration配置类中还会为每个RestTemplate实例添加LoadBalancerInterceptor拦截器。

在RibbonAutoConfiguration类中注入了LoadBalancerClient接口的实现类RibbonLoadBalancerClient



```
@Bean
@ConditionalOnMissingBean(LoadBalancerClient.class)
public LoadBalancerClient loadBalancerClient() {
    return new RibbonLoadBalancerClient(springClientFactory());
}
```









**3.4 拦截器**
由于在自动配置类中，对restTemplate实例添加了LoadBalancerInterceptor拦截器，所以，当用restTemplate发送http请求时，就会执行这个拦截器的intercept方法。

intercept方法中，会根据request.getURI()，获取请求的uri，再获取host，我们在发送http请求的时候，是用的服务名作为host，所以，这里就会拿到服务名，再调用具体LoadBalancerClient实例的execute方法，发送请求。

LoadBalancerClient的实现类为RibbonLoadBalancerClient，最终的负载均衡请求由它来执行，所以，还需要再梳理下RibbonLoadBalancerClient的逻辑。

先看下RibbonLoadBalancerClient中的execute方法：



```
public <T> T execute(String serviceId, LoadBalancerRequest<T> request, Object hint)
    throws IOException {
    ILoadBalancer loadBalancer = getLoadBalancer(serviceId);
    Server server = getServer(loadBalancer, hint);
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }
    RibbonServer ribbonServer = new RibbonServer(serviceId, server,
                                                 isSecure(server, serviceId),
                                                 serverIntrospector(serviceId).getMetadata(server));

    return execute(serviceId, ribbonServer, request);
}

@Override
public <T> T execute(String serviceId, ServiceInstance serviceInstance,
                     LoadBalancerRequest<T> request) throws IOException {
    Server server = null;
    if (serviceInstance instanceof RibbonServer) {
        server = ((RibbonServer) serviceInstance).getServer();
    }
    if (server == null) {
        throw new IllegalStateException("No instances available for " + serviceId);
    }

    RibbonLoadBalancerContext context = this.clientFactory
        .getLoadBalancerContext(serviceId);
    RibbonStatsRecorder statsRecorder = new RibbonStatsRecorder(context, server);
    try {
        T returnVal = request.apply(serviceInstance);
        statsRecorder.recordStats(returnVal);
        return returnVal;
    }
    // catch IOException and rethrow so RestTemplate behaves correctly
    catch (IOException ex) {
        statsRecorder.recordStats(ex);
        throw ex;
    }
    catch (Exception ex) {
        statsRecorder.recordStats(ex);
        ReflectionUtils.rethrowRuntimeException(ex);
    }
    return null;
}
```









服务名作为serviceId字段传进来，先通过getLoadBalancer获取loadBalancer，再根据loadBalancer获取server，下面是getServer的代码：



```
protected Server getServer(ILoadBalancer loadBalancer, Object hint) {
    if (loadBalancer == null) {
        return null;
    }
    // Use 'default' on a null hint, or just pass it on?
    return loadBalancer.chooseServer(hint != null ? hint : "default");
}
```









如果loadBalancer为空，就直接返回空，否则就调用loadBalancer的chooseServer方法，获取相应的server。

看一下ILoadBalancer是一个接口，里面声明了一系列负载均衡实现的方法：



```
public interface ILoadBalancer {
	public void addServers(List<Server> newServers);
	public Server chooseServer(Object key);
	public void markServerDown(Server server);
	@Deprecated
	public List<Server> getServerList(boolean availableOnly);
    public List<Server> getReachableServers();
	public List<Server> getAllServers();
}
```









这些方法名比较直观，很容易就能猜出是干啥的，addServers是用来添加一个server集合，chooseServer是选择一个server，markServerDown用来标记某个服务下线，getReachableServers获取可用的Server集合，getAllServers是获取所有的server集合。
ILoadBalancer有很多实现，那具体是用的哪个类呢，在RibbonAutoConfiguration类中注入SpringClientFactory，通过RibbonClientConfiguration类看到，这个配置类在初始化的时候，返回了ZoneAwareLoadBalancer作为负载均衡器。



```
@Bean
@ConditionalOnMissingBean
public ILoadBalancer ribbonLoadBalancer(IClientConfig config,
                                        ServerList<Server> serverList, ServerListFilter<Server> serverListFilter,
                                        IRule rule, IPing ping, ServerListUpdater serverListUpdater) {
    if (this.propertiesFactory.isSet(ILoadBalancer.class, name)) {
        return this.propertiesFactory.get(ILoadBalancer.class, config, name);
    }
    return new ZoneAwareLoadBalancer<>(config, rule, ping, serverList,
                                       serverListFilter, serverListUpdater);
}
```









**3.5 ZoneAwareLoadBalancer**
ZoneAwareLoadBalancer从名字中可以看出来，这个负载均衡器和zone是有关系的。下面看下ZoneAwareLoadBalancer中的chooseServer方法：

> eureka提供了region和zone两个概念来进行分区，这两个概念均来自于亚马逊的AWS：
> region：可以简单理解为地理上的分区，比如亚洲地区，或者华北地区，再或者北京等等，没有具体大小的限制。根据项目具体的情况，可以自行合理划分region。
> zone：可以简单理解为region内的具体机房，比如说region划分为北京，然后北京有两个机房，就可以在此region之下划分出zone1,zone2两个zone。



```
@Override
public Server chooseServer(Object key) {
    //只有当负载均衡器中维护的实例所属的Zone区域的个数大于1的时候才会执行选择策略
    //否则还是使用父类的实现
    if (!ENABLED.get() || getLoadBalancerStats().getAvailableZones().size() <= 1) {
        logger.debug("Zone aware logic disabled or there is only one zone");
        return super.chooseServer(key);
    }
    Server server = null;
    try {
        LoadBalancerStats lbStats = getLoadBalancerStats();
        //为当前负载均衡器中的所有Zone区域分别创建快照，保存在zoneSnapshot中，这些快照中的数据用于后续的算法
        Map<String, ZoneSnapshot> zoneSnapshot = ZoneAvoidanceRule.createSnapshot(lbStats);
        logger.debug("Zone snapshots: {}", zoneSnapshot);
        if (triggeringLoad == null) {
            triggeringLoad = DynamicPropertyFactory.getInstance().getDoubleProperty(
                "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".triggeringLoadPerServerThreshold", 0.2d);
        }

        if (triggeringBlackoutPercentage == null) {
            triggeringBlackoutPercentage = DynamicPropertyFactory.getInstance().getDoubleProperty(
                "ZoneAwareNIWSDiscoveryLoadBalancer." + this.getName() + ".avoidZoneWithBlackoutPercetage", 0.99999d);
        }
        //获得可用Zone区域的集合，getAvailableZones会通过zoneSnapshot实现可用区域挑选
        Set<String> availableZones = ZoneAvoidanceRule.getAvailableZones(zoneSnapshot, triggeringLoad.get(), triggeringBlackoutPercentage.get());
        logger.debug("Available zones: {}", availableZones);
        if (availableZones != null &&  availableZones.size() < zoneSnapshot.keySet().size()) {
            //随机选择一个Zone区域
            String zone = ZoneAvoidanceRule.randomChooseZone(zoneSnapshot, availableZones);
            logger.debug("Zone chosen: {}", zone);
            if (zone != null) {
                //获得对应区域的负载均衡器
                BaseLoadBalancer zoneLoadBalancer = getLoadBalancer(zone);
                //选择具体的服务实例
                //在chooseServer中将会使用IRule接口的choose函数来选择具体服务实例。在这里，IRule接口的实现会实现ZoneAvoidanceRule来挑选具体的服务实例。
                server = zoneLoadBalancer.chooseServer(key);
            }
        }
    } catch (Exception e) {
        logger.error("Error choosing server using zone aware logic for load balancer={}", name, e);
    }
    if (server != null) {
        return server;
    } else {
        logger.debug("Zone avoidance logic is not invoked.");
        return super.chooseServer(key);
    }
}
```









这个方法会根据server的zone和可用性来选择具体的实例，返回一个Server对象。

这个类里面的几个方法：

1.  setServerListForZones : 这个基于Zone进行服务划分
2.  chooseServer这里也是主要跟zone有关的计算，当然默认的Zone只有一个，所以直接是调用的父类的chooseServer（key)
3.  getLoadBalancer(String zone) 基于zone去获取LoadBalancer
4.  setRule(IRule rule) 为每个负载均衡器设置规则。
    这里可以看到，其实这个主要是针对Zone做了一些分类处理，就是将原来属于同一服务的服务实例，再根据地区进行划分。这也是为了能够快速响应而设置的。

**3.5.1 DynamicServerListLoadBalancer**

> ZoneAwareLoadBalancer的父类

这个类按照名称来说就是动态加载服务列表使用的。其中有几个比较重要的方法

1.  updateListOfServers:服务列表并更新本地缓存的服务列表
2.  enableAndInitLearnNewServersFeature:开启服务列表更新定时任务
3.  开启监听:@Monitor

DynamicServerListLoadBalancer的核心就是获取服务列表，在Eureka中默认是通过DomainExtractingServerList来获取，这个类是在org.springframework.cloud.netflix.ribbon.eureka.EurekaRibbonClientConfiguration#ribbonServerList，这里没有集成Eureka，暂时不讲

**3.5.2 BaseLoadBalancer**

> DynamicServerListLoadBalancer的父类

核心默认值

*   默认的负载均衡策略RoundRobinRule
*   默认的Ping策略SerialPingStrategy
*   所有服务实例容器：allServerList
*   在线服务实例容器：upServerList
    ![SpringCloud系列—Ribbon源码分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b2db0bc66c3045288a762907ff72e01883bf20.jpg "SpringCloud系列—Ribbon源码分析-开源基础软件社区")从子类构造中将对应的负载均衡规则，ping策略，ping等传递过来

**ping做了些什么？**

PingTask作为一个线程任务，就是定期检查服务是否都存活，跟ServerListUpdater服务更新机制不冲突。这是ribbon自己维护的一套服务检测机制，主要是为了降低访问失败的概率。默认在使用eureka时，ping是使用的是NIWSDiscoveryPing来完成服务保活检测。由eureka 和 ServerListUpdater来刷新服务列表。![SpringCloud系列—Ribbon源码分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/27db4c518aa7c100d427761832a9fa19fa8f04.jpg "SpringCloud系列—Ribbon源码分析-开源基础软件社区")这里有个常用的定时任务快速退出的方法，我觉得在我们自己写的时候也可以使用。

就是在同一个定时任务如果执行时间超过了定时周期，那么下一个定时任务发现上一个定时任务还没有执行完时，就先取消。![SpringCloud系列—Ribbon源码分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f2d669b11f17530d2e62603d45949e934a5931.jpg "SpringCloud系列—Ribbon源码分析-开源基础软件社区")这里也用了很多锁机制，比如所有服务实例到一个新的对象时使用的是读锁，就是告诉allServers现在只能读不能写。![SpringCloud系列—Ribbon源码分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/711c81e67b2c1cd3fb91005f53ea1e1352ad9a.png "SpringCloud系列—Ribbon源码分析-开源基础软件社区")在发送ping后，将检测通过的服务放入newUpList中，最后通过写锁，将upServerList锁住。

这里就是只能有一个写，且不能读。![SpringCloud系列—Ribbon源码分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/047543857174aa1130b0747d1d149ffd6d5632.jpg "SpringCloud系列—Ribbon源码分析-开源基础软件社区")上面是ping在检测过程中关于读写锁和原子类的使用。

主要流程就是：

1.  读取全部服务实例列表
2.  检测服务实例是否存活pingServers
3.  将服务状态发生改变的放入changedServers
4.  将服务在线的放入newUpList
5.  将newUpList赋值到upServerList 在线服务实例列表中

这里面pingServers就是检查心跳的

**BaseLoadBalancer的其他功能简述**

1.  对allServerList和upServerList的读写锁方法
2.  提供对allServerList和upServerList的增删改功能
3.  提供了PingTask（ping的定时任务），Pinger（ping的执行器）
4.  基于负载均衡策略选择服务rule.choose(key)
5.  提供默认的ping策略SerialPingStrategy
    **3.6 LoadBalancerRequest**
    通过ZoneAwareLoadBalancer选择具体的Server之后，再包装成RibbonServer对象，之前返回的server是该对象中的一个字段，除此之外，还有服务名serviceId，是否需要使用https等信息。最后，通过LoadBalancerRequest的apply方法，向具体的server发请求，从而实现了负载均衡。

下面是apply方法的定义：



```
public interface LoadBalancerRequest<T> {
    T apply(ServiceInstance instance) throws Exception;
}
```









在请求时，传入的ribbonServer对象，被当成ServiceInstance类型的对象进行接收。ServiceInstance是一个接口，定义了服务治理系统中，每个实例需要提供的信息，比如serviceId，host，port等。

LoadBalancerRequest是一个接口，最终会通过实现类的apply方法去执行，实现类是在LoadBalancerInterceptor中调用RibbonLoadBalancerClient的execute方法时，传进来的一个匿名类，可以通过查看LoadBalancerInterceptor的代码看到。

创建LoadBalancerRequest匿名类的时候，就重写了apply方法，apply方法中，还新建了一个ServiceRequestWrapper的内部类，这个类中，就重写了getURI方法，getURI方法会调用loadBalancer的reconstructURI方法来构建uri。

看到这里，已经可以大体知道Ribbon实现负载均衡的流程了，我们在RestTemplate上添加注解，就会有LoadBalancerClient的对象来配置它，也就是RibbonLoadBalancerClient。同时，
LoadBalancerAutoConfiguration会进行配置，创建一个LoadBalancerInterceptor，并且拿到我们声明的所有restTemplate，在这些restTemplate中添加LoadBalancerInterceptor拦截器。

当通过restTemplate发送请求时，就会经过这个拦截器，在拦截器中，就会调用RibbonLoadBalancerClient中的方法，获取到根据服务名，通过负载均衡方法获取到服务实例，然后去请求这个实例。

**3.7 获取服务列表**
上面说的这些，是如何对请求进行负载均衡的，但是还有个问题，我们请求的实例，是从Eureka Server上获取到的，那这个实例列表是如何获取的呢？怎么保证这个实例列表中的实例是可用的呢？

在RibbonLoadBalancerClient选择实例的时候，是通过ILoadBalancer的实现类根据负载均衡算法选择服务实例的，也就是ZoneAwareLoadBalancer的chooseServer中的逻辑，那就在这里找线索。查看ZoneAwareLoadBalancer的继承关系，可以看到如下图所示。![SpringCloud系列—Ribbon源码分析-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/425d8ff567d9ac6171f6036dab1726a7da04a3.jpg "SpringCloud系列—Ribbon源码分析-开源基础软件社区")可以看到，最上面是ILoadBalancer接口，AbstractLoadBalancer类继承了这个接口，BaseLoadBalancer继承了AbstractLoadBalancer类，
DynamicServerListLoadBalancer继承了BaseLoadBalancer，ZoneAwareLoadBalancer继承了DynamicServerListLoadBalancer。

ILoadBalancer接口的代码已经看过了，现在看下AbstractLoadBalancer的代码：



```
public abstract class AbstractLoadBalancer implements ILoadBalancer {
    public enum ServerGroup{
        ALL,
        STATUS_UP,
        STATUS_NOT_UP        
    }    
    /**
     * delegate to {@link #chooseServer(Object)} with parameter null.
     */
    public Server chooseServer() {
    	return chooseServer(null);
    }
    /**
     * List of servers that this Loadbalancer knows about
     * 
     * @param serverGroup Servers grouped by status, e.g., {@link ServerGroup#STATUS_UP}
     */
    public abstract List<Server> getServerList(ServerGroup serverGroup);
    /**
     * Obtain LoadBalancer related Statistics
     */
    public abstract LoadBalancerStats getLoadBalancerStats();    
}
```









这是一个抽象类，里面加了一个枚举，增加了两个抽象方法。定义的chooseServer方法。

下面再看BaseLoadBalancer类，BaseLoadBalancer类就算是负载均衡器的一个基础实现类，在里面可以看到定义了两个list：



```
@Monitor(name = PREFIX + "AllServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> allServerList = Collections
    .synchronizedList(new ArrayList<Server>());
@Monitor(name = PREFIX + "UpServerList", type = DataSourceType.INFORMATIONAL)
protected volatile List<Server> upServerList = Collections
    .synchronizedList(new ArrayList<Server>());
```









从名字上看，这就是维护所有服务的实例列表，和维护状态为up的实例列表。
而且还可以看到BaseLoadBalancer中实现的ILoadBalancer接口中的方法，比如下面这两个，获取可用的服务列表，就会把upServerList返回，获取所有的服务列表，就会把allServerList返回。



```
@Override
public List<Server> getReachableServers() {
    return Collections.unmodifiableList(upServerList);
}
@Override
public List<Server> getAllServers() {
    return Collections.unmodifiableList(allServerList);
}
```









接下来，再看DynamicServerListLoadBalancer类。从类头上的注释可以知道，这个类可以动态的获取服务列表，并且利用filter对服务列表进行过滤。

在DynamicServerListLoadBalancer类中，能看到定义了一个ServerList类型的serverListImpl字段，ServerList是一个接口，里面有两个方法：



```
public interface ServerList<T extends Server> {
    public List<T> getInitialListOfServers();
    /**
     * Return updated list of servers. This is called say every 30 secs
     * (configurable) by the Loadbalancer's Ping cycle
     * 
     */
    public List<T> getUpdatedListOfServers();   
}
```









getInitialListOfServers是获取初始化的服务列表。
getUpdatedListOfServers是获取更新的服务列表。
ServerList有多个实现类，具体用的哪个呢，可以在
EurekaRibbonClientConfiguration类中找到，这是Ribbon和Eureka结合的自动配置类，但是目前我们没有整合Eureka，是通过配置文件配置，所以会走ConfigurationBasedServerList类。

# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning