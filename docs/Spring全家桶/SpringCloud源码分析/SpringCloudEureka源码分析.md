Eureka源码分析

**第1章 核心流程**
**1.1 Eureka做了什么事**

首先我们都明白，Eureka是用在做服务注册的，而注册中心要实现什么功能呢？这个必须明确了。

1.  既然是注册中心，那首先要能保存服务的ip、port等信息吧，这是Eureka-server必须提供的基础功能。
2.  当我注册上来之后，还要提供一些动态感知服务上下线的功能吧，如果一个服务上下线，Eureka-server都不知道，那这个玩意儿就一文不值了。
3.  当Eureka-server端感知到服务的变化之后总得通知消费端吧，这里就牵扯到时server端主动通知client端呢还是client端自己去拉取信息呢？这个现在不知道，等会去看源码验证。
4.  OK，上面的功能都实现了，Eureka基本合格了，那还有一步，既然服务的ip和端口都在Eureka-server上面，那我消费端调用服务端的时候，通信是用的OpenFeign，那OpenFeign是怎么知道调用哪个服务的？之前都是写死在application.properties里面的<servicename>.ribbon.listOfServers中，现在Eureka怎么自动写进去的呢？
5.  上面4个功能基本上完成了注册中心该有的功能，那这个时候我们再来思考一下，注册中心用于微服务项目中，那注册中心也作为一个服务，它也需要做集群的，这个时候我们就要想一下，做集群怎么保证数据一致性，该基于什么理论？
    上面是Eureka要实现的最核心的功能，那这些功能提供出来了，我项目怎么去调用呢？不能直接去调API吧，这多麻烦，还得去学一遍Eureka的api，完犊子了。

这个时候我们就会联想到SpringBoot的自动装配和Starter组件。这两个玩意儿帮我们完成了核心bean的自动注入，底层可以直接拿到bean，然后在Starter组件中应该自动帮我们调用了API的；OK，那回过头来我发现，我的Eureka-client就是一个starter组件，嘿嘿，有点东西了。这个client端核心逻辑肯定是帮我们封装了各种bean，然后帮我们调用了核心api了。

这里针对Eureka的核心功能我们在做一个更简便的总结：

1.  实现注册，并存在内存中
2.  动态感知服务的健康状态
3.  服务的发现，及动态感知服务的变化
    **1.2 核心流程推导**
    明确了核心功能，以及如何调用的，接下来我们来大胆地推导一下核心流程图：

![SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f15564313c2406546821713bcaf3eb0e9ac73d.jpg "SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区")大体流程推导出来之后，我们再来通过源码做一个验证。

**第2章 源码分析**
**2.1 服务注册的入口**
服务注册是在spring boot应用启动的时候发起的。具体的执行路径我们暂且不看，先回顾一下前面咱们讲过的知识。

我们说spring cloud是一个生态，它提供了一套标准，这套标准可以通过不同的组件来实现，其中就包含服务注册/发现、熔断、负载均衡等，在spring-cloud-common这个包中，
org.springframework.cloud.client.serviceregistry 路径下，可以看到一个服务注册的接口定 义 ServiceRegistry 。它就是定义了spring cloud中服务注册的一个接口。

我们看一下它的类关系图，这个接口有一个唯一的实现 EurekaServiceRegistry 。表示采用的是Eureka Server作为服务注册中心。

![SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/49d085328febfffcbae4030dac151db2d0bfcb.jpg "SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区")**2.1.1 注册的时机**
服务注册的发起，我们可以猜测一下应该是什么时候完成？大家自要想想其实应该不难猜测到，服务的注册取决于服务是否已经启动好了。而在spring boot中，会等到spring 容器启动并且所有的配置都完成之后来进行注册。而这个动作在spring boot的启动方法中的refreshContext中完成。

我们观察一下finishRefresh这个方法，从名字上可以看到它是用来体现完成刷新的操作，也就是刷新完成之后要做的后置的操作。它主要做几个事情

* 清空缓存

* 初始化一个LifecycleProcessor，在Spring启动的时候启动bean，在spring结束的时候销毁bean

* 调用LifecycleProcessor的onRefresh方法，启动实现了Lifecycle接口的bean

* 发布ContextRefreshedEvent

* 注册Bean，通过JMX进行监控和管理



  ```
  protected void finishRefresh() {
      // Clear context-level resource caches (such as ASM metadata from scanning).
      clearResourceCaches();
      // Initialize lifecycle processor for this context.
      initLifecycleProcessor();
      // Propagate refresh to lifecycle processor first.
      getLifecycleProcessor().onRefresh();
      // Publish the final event.
      publishEvent(new ContextRefreshedEvent(this));
      // Participate in LiveBeansView MBean, if active.
      LiveBeansView.registerApplicationContext(this);
  }
  ```









在这个方法中，我们重点关注 getLifecycleProcessor().onRefresh() ，它是调用生命周期处理器的onrefresh方法，找到SmartLifecycle接口的所有实现类并调用start方法。

**2.1.2 SmartLifeCycle**
我拓展一下SmartLifeCycle这块的知识， SmartLifeCycle是一个接口，当Spring容器加载完所有的Bean并且初始化之后，会继续回调实现了SmartLifeCycle接口的类中对应的方法，比如（start）。

实际上我们自己也可以拓展，比如在springboot工程的main方法同级目录下，写一个测试类，实现SmartLifeCycle接口，并且通过 @Service 声明为一个bean，因为要被spring去加载，首先得是bean。



```
@Service
public class TestSmartLifeCycle implements SmartLifecycle {
    /**
     * 服务启动后执行.无需显示调用start方法.
     * 但是依赖isAutoStartup()返回值,只有isAutoStartup()返回true的时候,start()才会被执行
     */
    @Override
    public void start() {
        System.out.println("----------start-----------");
    }
    /**
     * 服务停止前执行方法
     * 前提条件: isRunning()返回true才会被执行
     */
    @Override
    public void stop() {
        System.out.println("----------stop-----------");
    }
    /**
     * 返回服务运行状态,影响到服务是否调用stop方法
     * @return
     */
    @Override
    public boolean isRunning() {
        return false;
    }
    /**
     * 是否调用start方法,需要注意
     * 当前方法返回false是不会执行start()
     * @return
     */
    @Override
    public boolean isAutoStartup() {
        return true;
    }
    @Override
    public void stop(Runnable runnable) {
        stop();
        runnable.run();
    }
    /**
     * 指定执行顺序
     * 当前容器如果有多个类实现了SmartLifecycle,则按此法方法返回值排序执行
     * @return
     */
    @Override
    public int getPhase() {
        return 0;
    }
}
```









接着，我们启动spring boot应用后，可以看到控制台输出了 start 字符串。

我们在DefaultLifecycleProcessor.startBeans方法上加一个debug，可以很明显的看到我们自己定义的TestSmartLifeCycle被扫描到了，并且最后会调用该bean的start方法。![SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/75b18e528ba1879148c106d9bd3fb61ab9b5bb.jpg "SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区")在startBeans方法中，我们可以看到它首先会获得所有实现了SmartLifeCycle的Bean，然后会循环调用实现了SmartLifeCycle的bean的start方法，代码如下。



```
private void startBeans(boolean autoStartupOnly) {
    Map<String, Lifecycle> lifecycleBeans = this.getLifecycleBeans();
    Map<Integer, DefaultLifecycleProcessor.LifecycleGroup> phases = new HashMap();
    lifecycleBeans.forEach((beanName, bean) -> {
        if (!autoStartupOnly || bean instanceof SmartLifecycle &&
            ((SmartLifecycle)bean).isAutoStartup()) {
            int phase = this.getPhase(bean);
            DefaultLifecycleProcessor.LifecycleGroup group =
                (DefaultLifecycleProcessor.LifecycleGroup)phases.get(phase);
            if (group == null) {
                group = new DefaultLifecycleProcessor.LifecycleGroup(phase,this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
                phases.put(phase, group);
            }
            group.add(beanName, bean);
        }
    });
    if (!phases.isEmpty()) {
        List<Integer> keys = new ArrayList(phases.keySet());
        Collections.sort(keys);
        Iterator var5 = keys.iterator();
        while(var5.hasNext()) {
            Integer key = (Integer)var5.next();
            ((DefaultLifecycleProcessor.LifecycleGroup)phases.get(key)).start();
        }
    }
}
```









**2.1.3 doStart**



```
private void doStart(Map<String, ? extends Lifecycle> lifecycleBeans, String beanName, boolean autoStartupOnly) {
    Lifecycle bean = (Lifecycle)lifecycleBeans.remove(beanName);
    if (bean != null && bean != this) {
        String[] dependenciesForBean = this.getBeanFactory().getDependenciesForBean(beanName);
        String[] var6 = dependenciesForBean;
        int var7 = dependenciesForBean.length;
        for(int var8 = 0; var8 < var7; ++var8) {
            String dependency = var6[var8];
            this.doStart(lifecycleBeans, dependency, autoStartupOnly);
        }
        if (!bean.isRunning() && (!autoStartupOnly || !(bean instanceof SmartLifecycle) || ((SmartLifecycle)bean).isAutoStartup())) {
            if (this.logger.isTraceEnabled()) {
                this.logger.trace("Starting bean '" + beanName + "' of type [" + bean.getClass().getName() + "]");
            }
            try {
                bean.start(); //此时 Bean的实例应该是EurekaAutoServiceRegistration
            } catch (Throwable var10) {
                throw new ApplicationContextException("Failed to start bean '" + beanName + "'", var10);
            }
            if (this.logger.isDebugEnabled()) {
                this.logger.debug("Successfully started bean '" + beanName + "'");
            }
        }
    }
}
```









此时，bean.start()，调用的可能是
EurekaAutoServiceRegistration中的start方法，因为很显然，它实现了SmartLifeCycle接口。



```
public class EurekaAutoServiceRegistration implements AutoServiceRegistration,SmartLifecycle, Ordered, SmartApplicationListener {
    @Override
    public void start() {
        // only set the port if the nonSecurePort or securePort is 0 and this.port != 0
        if (this.port.get() != 0) {
            if (this.registration.getNonSecurePort() == 0) {
                this.registration.setNonSecurePort(this.port.get());
            }
            if (this.registration.getSecurePort() == 0 &&
                this.registration.isSecure()) {
                this.registration.setSecurePort(this.port.get());
            }
        }
        // only initialize if nonSecurePort is greater than 0 and it isn't already running
        // because of containerPortInitializer below
        if (!this.running.get() && this.registration.getNonSecurePort() > 0) {
            this.serviceRegistry.register(this.registration);
            this.context.publishEvent(new InstanceRegisteredEvent<>(this,
                                                                    this.registration.getInstanceConfig()));
            this.running.set(true);
        }
    }
}
```









在start方法中，我们可以看到
this.serviceRegistry.register 这个方法，它实际上就是发起服务注册的机制。

此时this.serviceRegistry的实例，应该是 EurekaServiceRegistry ， 原因是
EurekaAutoServiceRegistration的构造方法中，会有一个赋值操作，而这个构造方法是在EurekaClientAutoConfiguration 这个自动装配类中被装配和初始化的，代码如下。



```
@Bean
@ConditionalOnBean(AutoServiceRegistrationProperties.class)
@ConditionalOnProperty(value = "spring.cloud.service-registry.auto-registration.enabled", matchIfMissing = true)
public EurekaAutoServiceRegistration eurekaAutoServiceRegistration(
    ApplicationContext context, EurekaServiceRegistry registry,
    EurekaRegistration registration) {
    return new EurekaAutoServiceRegistration(context, registry, registration);
}
```









**2.2 服务的注册流程**
接下来我们分析服务注册的流程



```
public class EurekaAutoServiceRegistration implements AutoServiceRegistration,
SmartLifecycle, Ordered, SmartApplicationListener {
    @Override
    public void start() {
        //省略...
        this.serviceRegistry.register(this.registration);
        this.context.publishEvent(new InstanceRegisteredEvent<> this,this.registration.getInstanceConfig()));
    }
}
```









this.serviceRegistry.register(this.registration); 方法最终会调用

EurekaServiceRegistry 类中的 register 方法来实现服务注册

**2.2.1 register**



```
@Override
public void register(EurekaRegistration reg) {
    maybeInitializeClient(reg);
    if (log.isInfoEnabled()) {
        log.info("Registering application "
                 + reg.getApplicationInfoManager().getInfo().getAppName()
                 + " with eureka with status "
                 + reg.getInstanceConfig().getInitialStatus());
    }
    //设置当前实例的状态，一旦这个实例的状态发生变化，只要状态不是DOWN，那么就会被监听器监听并且执行服务注册。
    reg.getApplicationInfoManager().setInstanceStatus(reg.getInstanceConfig().getInitialStatus());
    //设置健康检查的处理
    reg.getHealthCheckHandler().ifAvailable(healthCheckHandler -> reg.getEurekaClient().registerHealthCheck(healthCheckHandler));
}
```









从上述代码来看，注册方法中并没有真正调用Eureka的方法去执行注册，而是仅仅设置了一个状态以及设置健康检查处理器。我们继续看一下
reg.getApplicationInfoManager().setInstanceStatus方法。



```
public synchronized void setInstanceStatus(InstanceStatus status) {
    InstanceStatus next = instanceStatusMapper.map(status);
    if (next == null) {
        return;
    }
    InstanceStatus prev = instanceInfo.setStatus(next);
    if (prev != null) {
        for (StatusChangeListener listener : listeners.values()) {
            try {
                listener.notify(new StatusChangeEvent(prev, next));
            } catch (Exception e) {
                logger.warn("failed to notify listener: {}", listener.getId(),e);
            }
        }
    }
}
```









在这个方法中，它会通过监听器来发布一个状态变更事件。ok，此时listener的实例是StatusChangeListener ，也就是调用 StatusChangeListener 的notify方法。这个事件是触发一个服务状态变更，应该是有地方会监听这个事件，然后基于这个事件进行注册。

这个时候我们以为找到了方向，然后点击进去一看，发现它是一个接口。而且我们发现它是静态的内部接口，还无法直接看到它的实现类。

依我多年源码阅读经验，于是又往回找，因为我基本上能猜测到一定是在某个地方做了初始化的工作，于是，我想找到
EurekaServiceRegistry.register方法中的 reg.getApplicationInfoManager 这个实例是什么，而且我们发现ApplicationInfoManager是来自于EurekaRegistration这个类中的属性。而EurekaRegistration又是在EurekaAutoServiceRegistration这个类中实例化的。那我在想，是不是在自动装配中做了什么东西。于是找到EurekaClientAutoConfiguration这个类，果然看到了Bean的一些自动装配，其中包含 EurekaClient 、 ApplicationInfoMangager 、 EurekaRegistration 等。

**2.2.2 EurekaClientConfiguration**



```
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingRefreshScope
protected static class EurekaClientConfiguration {
    @Autowired
    private ApplicationContext context;
    @Autowired
    private AbstractDiscoveryClientOptionalArgs<?> optionalArgs;
    @Bean(destroyMethod = "shutdown")
    @ConditionalOnMissingBean(value = EurekaClient.class,search = SearchStrategy.CURRENT)
    public EurekaClient eurekaClient(ApplicationInfoManager manager,EurekaClientConfig config) {
        return new CloudEurekaClient(manager, config, this.optionalArgs,this.context);
    }
    @Bean
    @ConditionalOnMissingBean(value = ApplicationInfoManager.class,search = SearchStrategy.CURRENT)
    public ApplicationInfoManager eurekaApplicationInfoManager(
        EurekaInstanceConfig config) {
        InstanceInfo instanceInfo = new InstanceInfoFactory().create(config);
        return new ApplicationInfoManager(config, instanceInfo);
    }
    @Bean
    @ConditionalOnBean(AutoServiceRegistrationProperties.class)
    @ConditionalOnProperty(
        value = "spring.cloud.service-registry.auto-registration.enabled",
        matchIfMissing = true)
    public EurekaRegistration eurekaRegistration(EurekaClient eurekaClient,CloudEurekaInstanceConfig
                                                 instanceConfig,ApplicationInfoManager applicationInfoManager, @Autowired(required = false)
                                                 ObjectProvider<HealthCheckHandler> healthCheckHandler) {
        return EurekaRegistration.builder(instanceConfig).with(applicationInfoManager).with(eurekaClient).with(healthCheckHandler).build();
    }
}
```









不难发现，我们似乎看到了一个很重要的Bean在启动的时候做了自动装配，也就是CloudEurekaClient 。从名字上来看，我可以很容易的识别并猜测出它是Eureka客户端的一个工具类，用来实现和服务端的通信以及处理。这个是很多源码一贯的套路，要么在构造方法里面去做很多的初始化和一些后台执行的程序操作，要么就是通过异步事件的方式来处理。接着，我们看一下CloudEurekaClient的初始化过程，它的构造方法中会通过 super 调用父类的构造方法。也就是DiscoveryClient的构造。

**2.2.3 CloudEurekaClient**
super(applicationInfoManager, config, args);调用父类的构造方法，而CloudEurekaClient的父类是DiscoveryClient.



```
public CloudEurekaClient(ApplicationInfoManager applicationInfoManager,EurekaClientConfig config,AbstractDiscoveryClientOptionalArgs<?> args,ApplicationEventPublisher publisher) {
    super(applicationInfoManager, config, args);
    this.applicationInfoManager = applicationInfoManager;
    this.publisher = publisher;
    this.eurekaTransportField = ReflectionUtils.findField(DiscoveryClient.class,"eurekaTransport");
    ReflectionUtils.makeAccessible(this.eurekaTransportField);
}
```









**2.2.4 DiscoveryClient构造**
我们可以看到在最终的DiscoveryClient构造方法中，有非常长的代码。其实很多代码可以不需要关心，大部分都是一些初始化工作，比如初始化了几个定时任务

* scheduler

* heartbeatExecutor 心跳定时任务

* cacheRefreshExecutor 定时去同步服务端的实例列表



  ```
  DiscoveryClient(ApplicationInfoManager applicationInfoManager,EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,Provider<BackupRegistry> backupRegistryProvider,EndpointRandomizer endpointRandomizer) {
      //省略部分代码...
      //是否要从eureka server上获取服务地址信息
      if (config.shouldFetchRegistry()) {
          this.registryStalenessMonitor = new ThresholdLevelsMetric(this,METRIC_REGISTRY_PREFIX + "lastUpdateSec_", new long[]{15L, 30L, 60L, 120L, 240L,480L});
      } else {
          this.registryStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
      }
      //是否要注册到eureka server上
      if (config.shouldRegisterWithEureka()) {
          this.heartbeatStalenessMonitor = new ThresholdLevelsMetric(this,METRIC_REGISTRATION_PREFIX + "lastHeartbeatSec_", new long[]{15L, 30L, 60L,120L, 240L, 480L});
      } else {
          this.heartbeatStalenessMonitor = ThresholdLevelsMetric.NO_OP_METRIC;
      }
      //如果不需要注册并且不需要更新服务地址
      if (!config.shouldRegisterWithEureka() && !config.shouldFetchRegistry()) {
  
          return;  // no need to setup up an network tasks and we are done
      }
      try {
          // default size of 2 - 1 each for heartbeat and cacheRefresh
          scheduler = Executors.newScheduledThreadPool(2,new ThreadFactoryBuilder()       .setNameFormat("DiscoveryClient-%d")
                                                       .setDaemon(true)
                                                       .build());
          heartbeatExecutor = new ThreadPoolExecutor(1, clientConfig.getHeartbeatExecutorThreadPoolSize(), 0,
                                                     TimeUnit.SECONDS,
                                                     new SynchronousQueue<Runnable>(),
                                                     new ThreadFactoryBuilder()
                                                     .setNameFormat("DiscoveryClient-HeartbeatExecutor-%d")
                                                     .setDaemon(true)
                                                     .build()
                                                    );  // use direct handoff
          cacheRefreshExecutor = new ThreadPoolExecutor(
              1, clientConfig.getCacheRefreshExecutorThreadPoolSize(), 0,
              TimeUnit.SECONDS,
              new SynchronousQueue<Runnable>(),
              new ThreadFactoryBuilder()
              .setNameFormat("DiscoveryClient-CacheRefreshExecutor-%d")
              .setDaemon(true)
              .build()
          );  // use direct handoff
          eurekaTransport = new EurekaTransport();
          scheduleServerEndpointTask(eurekaTransport, args);
          AzToRegionMapper azToRegionMapper;
          if (clientConfig.shouldUseDnsForFetchingServiceUrls()) {
              azToRegionMapper = new DNSBasedAzToRegionMapper(clientConfig);
          } else {
              azToRegionMapper = new PropertyBasedAzToRegionMapper(clientConfig);
          }
          if (null != remoteRegionsToFetch.get()) {
  
              azToRegionMapper.setRegionsToFetch(remoteRegionsToFetch.get().split(","));
          }
          instanceRegionChecker = new InstanceRegionChecker(azToRegionMapper,
                                                            clientConfig.getRegion());
      } catch (Throwable e) {
          throw new RuntimeException("Failed to initialize DiscoveryClient!", e);
      }
      //如果需要注册到Eureka server并且是开启了初始化的时候强制注册，则调用register()发起服务注册
      if (clientConfig.shouldRegisterWithEureka() &&
          clientConfig.shouldEnforceRegistrationAtInit()) {
          try {
              if (!register() ) {
                  throw new IllegalStateException("Registration error at startup.Invalid server response.");
              }
          } catch (Throwable th) {
              logger.error("Registration error at startup: {}", th.getMessage());
              throw new IllegalStateException(th);
          }
      }
      // finally, init the schedule tasks (e.g. cluster resolvers, heartbeat,instanceInfo replicator, fetch
      initScheduledTasks();
  }
  ```









**2.2.5 initScheduledTasks**
initScheduledTasks 去启动一个定时任务。

*   如果配置了开启从注册中心刷新服务列表，则会开启cacheRefreshExecutor这个定时任务
*   如果开启了服务注册到Eureka，则通过需要做几个事情.

1.  建立心跳检测机制

通过内部类来实例化StatusChangeListener 实例状态监控接口，这个就是前面我们在分析启动过程中所看到的，调用notify的方法，实际上会在这里体现。



```
private void initScheduledTasks() {
    //如果配置了开启从注册中心刷新服务列表，则会开启cacheRefreshExecutor这个定时任务
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        int registryFetchIntervalSeconds =
            clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound =
            clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        scheduler.schedule(
            new TimedSupervisorTask(
                "cacheRefresh",
                scheduler,
                cacheRefreshExecutor,
                registryFetchIntervalSeconds,
                TimeUnit.SECONDS,
                expBackOffBound,
                new CacheRefreshThread()
            ),
            registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }
    //如果开启了服务注册到Eureka，则通过需要做几个事情
    if (clientConfig.shouldRegisterWithEureka()) {
        int renewalIntervalInSecs =
            instanceInfo.getLeaseInfo().getRenewalIntervalInSecs();
        int expBackOffBound =
            clientConfig.getHeartbeatExecutorExponentialBackOffBound();
        logger.info("Starting heartbeat executor: " + "renew interval is: {}",
                    renewalIntervalInSecs);
        // Heartbeat timer
        scheduler.schedule(
            new TimedSupervisorTask(
                "heartbeat",
                scheduler,
                heartbeatExecutor,
                renewalIntervalInSecs,
                TimeUnit.SECONDS,
                expBackOffBound,
                new HeartbeatThread()
            ),
            renewalIntervalInSecs, TimeUnit.SECONDS);
        // InstanceInfo replicator 初始化一个:instanceInfoReplicator
        instanceInfoReplicator = new InstanceInfoReplicator(
            this,
            instanceInfo,
            clientConfig.getInstanceInfoReplicationIntervalSeconds(),
            2); // burstSize
        statusChangeListener = new ApplicationInfoManager.StatusChangeListener()
        {
            @Override
            public String getId() {
                return "statusChangeListener";
            }
            @Override
            public void notify(StatusChangeEvent statusChangeEvent) {
                if (InstanceStatus.DOWN == statusChangeEvent.getStatus() ||
                    InstanceStatus.DOWN ==
                    statusChangeEvent.getPreviousStatus()) {
                    // log at warn level if DOWN was involved
                    logger.warn("Saw local status change event {}",
                                statusChangeEvent);
                } else {
                    logger.info("Saw local status change event {}",
                                statusChangeEvent);
                }
                instanceInfoReplicator.onDemandUpdate();
            }
        };
        //注册实例状态变化的监听
        if (clientConfig.shouldOnDemandUpdateStatusChange()) {
            applicationInfoManager.registerStatusChangeListener(statusChangeListener);
        }
        //启动一个实例信息复制器，主要就是为了开启一个定时线程，每40秒判断实例信息是否变更，如果变更了则重新注册
        instanceInfoReplicator.start(clientConfig.getInitialInstanceInfoReplicationInte
                                     rvalSeconds());
    } else {
        logger.info("Not registering with Eureka server per configuration");
    }
}
```









**2.2.6 onDemandUpdate**
这个方法的主要作用是根据实例数据是否发生变化，来触发服务注册中心的数据。



```
public boolean onDemandUpdate() {
    //限流判断
    if (rateLimiter.acquire(burstSize, allowedRatePerMinute)) {
        if (!scheduler.isShutdown()) {
            //提交一个任务
            scheduler.submit(new Runnable() {
                @Override
                public void run() {
                    logger.debug("Executing on-demand update of local InstanceInfo");
                    //取出之前已经提交的任务，也就是在start方法中提交的更新任务，如果任务还没有执行完成，则取消之前的任务。
                    Future latestPeriodic = scheduledPeriodicRef.get();
                    if (latestPeriodic != null && !latestPeriodic.isDone()) {
                        logger.debug("Canceling the latest scheduled update, it will be rescheduled at the end of on demand update");
                        latestPeriodic.cancel(false);//如果此任务未完成，就立即取消
                    }
                    //通过调用run方法，令任务在延时后执行，相当于周期性任务中的一次
                    InstanceInfoReplicator.this.run();
                }
            });
            return true;
        } else {
            logger.warn("Ignoring onDemand update due to stopped scheduler");
            return false;
        }
    } else {
        logger.warn("Ignoring onDemand update due to rate limiter");
        return false;
    }
}
```









**2.2.7 run**
run方法实际上和前面自动装配所执行的服务注册方法是一样的，也就是调用 register 方法进行服务注册，并且在finally中，每30s会定时执行一下当前的run 方法进行检查。



```
public void run() {
    try {
        discoveryClient.refreshInstanceInfo();
        Long dirtyTimestamp = instanceInfo.isDirtyWithTime();
        if (dirtyTimestamp != null) {
            discoveryClient.register();
            instanceInfo.unsetIsDirty(dirtyTimestamp);
        }
    } catch (Throwable t) {
        logger.warn("There was a problem with the instance info replicator", t);
    } finally {
        Future next = scheduler.schedule(this, replicationIntervalSeconds,
                                         TimeUnit.SECONDS);
        scheduledPeriodicRef.set(next);
    }
}
```









**2.2.8 register**
最终，我们终于找到服务注册的入口了，
eurekaTransport.registrationClient.register 最终调用的是 AbstractJerseyEurekaHttpClient#register(...)`， 当然大家如果自己去看代码，就会发现去调用之前有很多绕来绕去的代码，比如工厂模式、装饰器模式等。



```
boolean register() throws Throwable {
    logger.info(PREFIX + "{}: registering service...", appPathIdentifier);
    EurekaHttpResponse<Void> httpResponse;
    try {
        httpResponse = eurekaTransport.registrationClient.register(instanceInfo);
    } catch (Exception e) {
        logger.warn(PREFIX + "{} - registration failed {}", appPathIdentifier,e.getMessage(), e);
        throw e;
    }
    if (logger.isInfoEnabled()) {
        logger.info(PREFIX + "{} - registration status: {}", appPathIdentifier,httpResponse.getStatusCode());
    }
    return httpResponse.getStatusCode() == Status.NO_CONTENT.getStatusCode();
}
```









很显然，这里是发起了一次http请求，访问Eureka-Server的apps/${APP_NAME}接口，将当前服务实例的信息发送到Eureka Server进行保存。

至此，我们基本上已经知道Spring Cloud Eureka 是如何在启动的时候把服务信息注册到Eureka Server上的了。



```
public EurekaHttpResponse<Void> register(InstanceInfo info) {
    String urlPath = "apps/" + info.getAppName();
    ClientResponse response = null;
    try {
        Builder resourceBuilder =
            jerseyClient.resource(serviceUrl).path(urlPath).getRequestBuilder();
        addExtraHeaders(resourceBuilder);
        response = resourceBuilder
            .header("Accept-Encoding", "gzip")
            .type(MediaType.APPLICATION_JSON_TYPE)
            .accept(MediaType.APPLICATION_JSON)
            .post(ClientResponse.class, info);
        return
            anEurekaHttpResponse(response.getStatus()).headers(headersOf(response)).build();
    } finally {
        if (logger.isDebugEnabled()) {
            logger.debug("Jersey HTTP POST {}/{} with instance {}; statusCode={}", serviceUrl, urlPath, info.getId(),
                         response == null ? "N/A" : response.getStatus());
        }
        if (response != null) {
            response.close();
        }
    }
}
```









但是，似乎最开始的问题还没有解决，也就是Spring Boot应用在启动时，会调用start方法，最终调用
StatusChangeListener.notify 去更新服务的一个状态，并没有直接调用register方法注册。所以我们继续去看一下 statusChangeListener.notify 方法。

**2.2.9 服务总结**
至此，我们知道Eureka Client发起服务注册时，有两个地方会执行服务注册的任务

1.  在Spring Boot启动时，由于自动装配机制将CloudEurekaClient注入到了容器，并且执行了构造方法，而在构造方法中有一个定时任务每40s会执行一次判断，判断实例信息是否发生了变化，如果是则会发起服务注册的流程
2.  在Spring Boot启动时，通过refresh方法，最终调用StatusChangeListener.notify进行服务状态变更的监听，而这个监听的方法受到事件之后会去执行服务注册。
    **2.3 Server端逻辑**
    在没分析源码实现之前，我们一定知道它肯定对请求过来的服务实例数据进行了存储。那么我们去Eureka Server端看一下处理流程。

请求入口在：
com.netflix.eureka.resources.ApplicationResource.addInstance() 。

大家可以发现，这里所提供的REST服务，采用的是jersey来实现的。Jersey是基于JAX-RS标准，提供REST的实现的支持，这里就不展开分析了。

**2.3.1 addInstance()**
当EurekaClient调用register方法发起注册时，会调用
ApplicationResource.addInstance方法。

服务注册就是发送一个POST请求带上当前实例信息到类 ApplicationResource 的 addInstance方法进行服务注册。



```
@POST
@Consumes({"application/json", "application/xml"})
public Response addInstance(InstanceInfo info, @HeaderParam("x-netflix-discovery-replication") String isReplication) {
    logger.debug("Registering instance {} (replication={})", info.getId(),
                 isReplication);
    DataCenterInfo dataCenterInfo = info.getDataCenterInfo();
    if (dataCenterInfo instanceof UniqueIdentifier) {
        String dataCenterInfoId =
            ((UniqueIdentifier)dataCenterInfo).getId();
        if (this.isBlank(dataCenterInfoId)) {
            boolean experimental =            "true".equalsIgnoreCase(this.serverConfig.getExperimental("registration.validation.dataCenterInfoId"));
            if (experimental) {
                String entity = "DataCenterInfo of type " +
                    dataCenterInfo.getClass() + " must contain a valid id";
                return Response.status(400).entity(entity).build();
            }
            if (dataCenterInfo instanceof AmazonInfo) {
                AmazonInfo amazonInfo = (AmazonInfo)dataCenterInfo;
                String effectiveId = amazonInfo.get(MetaDataKey.instanceId);
                if (effectiveId == null) {
                    amazonInfo.getMetadata().put(MetaDataKey.instanceId.getName(), info.getId());
                }
            } else {
                logger.warn("Registering DataCenterInfo of type {} without an appropriate id", dataCenterInfo.getClass());
            }
        }
    }
    this.registry.register(info, "true".equals(isReplication));
    return Response.status(204).build();
}

```









**2.3.2 register**
我们先来看
PeerAwareInstanceRegistryImpl的类关系图，从类关系图可以看出，PeerAwareInstanceRegistry的最顶层接口为LeaseManager与LookupService,

*   其中LookupService定义了最基本的发现示例的行为
*   LeaseManager定义了处理客户端注册，续约，注销等操作

![SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/897d97d444664479bda0430417ec683f1f079a.jpg "SpringCloud系列—Spring Cloud 源码分析之Eureka-开源基础软件社区")在 addInstance 方法中，最终调用的是
PeerAwareInstanceRegistryImpl.register 方法。

* leaseDuration 表示租约过期时间，默认是90s，也就是当服务端超过90s没有收到客户端的心跳，则主动剔除该节点

* 调用super.register发起节点注册

* 将信息复制到Eureka Server集群中的其他机器上，同步的实现也很简单，就是获得集群中的所有节点，然后逐个发起注册



  ```
  public void register(final InstanceInfo info, final boolean isReplication) {
      int leaseDuration = Lease.DEFAULT_DURATION_IN_SECS;
      if (info.getLeaseInfo() != null && info.getLeaseInfo().getDurationInSecs() >0) {
          leaseDuration = info.getLeaseInfo().getDurationInSecs(); //如果客户端有自己定义心跳超时时间，则采用客户端的时间
      }
      super.register(info, leaseDuration, isReplication); //节点注册
      //复制到Eureka Server集群中的其他节点
      replicateToPeers(Action.Register, info.getAppName(), info.getId(), info,
                       null, isReplication);
  }
  ```









**2.3.3 AbstractInstanceRegistry.register**
简单来说，Eureka-Server的服务注册，实际上是将客户端传递过来的实例数据保存到Eureka-Server中的ConcurrentHashMap中。



```
public void register(InstanceInfo registrant, int leaseDuration, boolean
                     isReplication) {
    try {
        read.lock();
        //从registry中获得当前实例信息，根据appName
        Map<String, Lease<InstanceInfo>> gMap =
            registry.get(registrant.getAppName());
        REGISTER.increment(isReplication); //增加注册次数到监控信息中
        if (gMap == null) {//如果当前appName是第一次注册，则初始化一个ConcurrentHashMap
            final ConcurrentHashMap<String, Lease<InstanceInfo>> gNewMap = new
                ConcurrentHashMap<String, Lease<InstanceInfo>>();
            gMap = registry.putIfAbsent(registrant.getAppName(), gNewMap);
            if (gMap == null) {
                gMap = gNewMap;
            }
        }
        //从gMap中查询已经存在的Lease信息，Lease中文翻译为租约，实际上它把服务提供者的实例信息包装成了一个lease，里面提供了对于改服务实例的租约管理
        Lease<InstanceInfo> existingLease = gMap.get(registrant.getId());
        // 当instance已经存在是，和客户端的instance的信息做比较，时间最新的那个，为有效instance信息
        if (existingLease != null && (existingLease.getHolder() != null)) {
            Long existingLastDirtyTimestamp =
                existingLease.getHolder().getLastDirtyTimestamp();
            Long registrationLastDirtyTimestamp =
                registrant.getLastDirtyTimestamp();
            logger.debug("Existing lease found (existing={}, provided={}",
                         existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
            // this is a > instead of a >= because if the timestamps are equal,we still take the remote transmitted
            // InstanceInfo instead of the server local copy.
            if (existingLastDirtyTimestamp > registrationLastDirtyTimestamp) {
                logger.warn("There is an existing lease and the existing lease's dirty timestamp {} is greater" +
                            " than the one that is being registered {}",
                            existingLastDirtyTimestamp, registrationLastDirtyTimestamp);
                logger.warn("Using the existing instanceInfo instead of the new instanceInfo as the registrant");
                registrant = existingLease.getHolder();
            }
        } else {
            //当lease不存在时，进入到这段代码，
            synchronized (lock) {
                if (this.expectedNumberOfClientsSendingRenews > 0) {
                    // Since the client wants to register it, increase the number of clients sending renews
                    this.expectedNumberOfClientsSendingRenews =
                        this.expectedNumberOfClientsSendingRenews + 1;
                    updateRenewsPerMinThreshold();
                }
            }
            logger.debug("No previous lease information found; it is new registration");
        }
        //构建一个lease
        Lease<InstanceInfo> lease = new Lease<InstanceInfo>(registrant,
                                                            leaseDuration);
        if (existingLease != null) {
            // 当原来存在Lease的信息时，设置serviceUpTimestamp, 保证服务启动的时间一直是第一次注册的那个
            lease.setServiceUpTimestamp(existingLease.getServiceUpTimestamp());
        }
        gMap.put(registrant.getId(), lease);
        synchronized (recentRegisteredQueue) {//添加到最近注册的队列中
            recentRegisteredQueue.add(new Pair<Long, String>(
                System.currentTimeMillis(),
                registrant.getAppName() + "(" + registrant.getId() + ")"));
        }
        // 检查实例状态是否发生变化，如果是并且存在，则覆盖原来的状态
        if (!InstanceStatus.UNKNOWN.equals(registrant.getOverriddenStatus())) {
            logger.debug("Found overridden status {} for instance {}. Checking to see if needs to be add to the "
                         + "overrides", registrant.getOverriddenStatus(),
                         registrant.getId());
            if (!overriddenInstanceStatusMap.containsKey(registrant.getId())) {
                logger.info("Not found overridden id {} and hence adding it",
                            registrant.getId());
                overriddenInstanceStatusMap.put(registrant.getId(),
                                                registrant.getOverriddenStatus());
            }
        }
        InstanceStatus overriddenStatusFromMap =
            overriddenInstanceStatusMap.get(registrant.getId());
        if (overriddenStatusFromMap != null) {
            logger.info("Storing overridden status {} from map",
                        overriddenStatusFromMap);
            registrant.setOverriddenStatus(overriddenStatusFromMap);
        }
        // Set the status based on the overridden status rules
        InstanceStatus overriddenInstanceStatus =
            getOverriddenInstanceStatus(registrant, existingLease, isReplication);
        registrant.setStatusWithoutDirty(overriddenInstanceStatus);
        // 得到instanceStatus，判断是否是UP状态，
        if (InstanceStatus.UP.equals(registrant.getStatus())) {
            lease.serviceUp();
        }
        // 设置注册类型为添加
        registrant.setActionType(ActionType.ADDED);
        // 租约变更记录队列，记录了实例的每次变化， 用于注册信息的增量获取
        recentlyChangedQueue.add(new RecentlyChangedItem(lease));
        registrant.setLastUpdatedTimestamp();
        //让缓存失效
        invalidateCache(registrant.getAppName(), registrant.getVIPAddress(),
                        registrant.getSecureVipAddress());
        logger.info("Registered instance {}/{} with status {} (replication={})",
                    registrant.getAppName(), registrant.getId(),
                    registrant.getStatus(), isReplication);
    } finally {
        read.unlock();
    }
}
```









**2.3.4 小结**
至此，我们就把服务注册在客户端和服务端的处理过程做了一个详细的分析，实际上在Eureka Server端，会把客户端的地址信息保存到ConcurrentHashMap中存储。并且服务提供者和注册中心之间，会建立一个心跳检测机制。用于监控服务提供者的健康状态。

**2.4 Eureka 的多级缓存设计**
Eureka Server存在三个变量：(registry、readWriteCacheMap、readOnlyCacheMap)保存服务注册信息，默认情况下定时任务每30s将readWriteCacheMap同步至readOnlyCacheMap，每60s清理超过90s未续约的节点，Eureka Client每30s从readOnlyCacheMap更新服务注册信息，而客户端服务的注册则从registry更新服务注册信息。

**2.4.1 多级缓存的意义**
这里为什么要设计多级缓存呢？原因很简单，就是当存在大规模的服务注册和更新时，如果只是修改一个ConcurrentHashMap数据，那么势必因为锁的存在导致竞争，影响性能。

而Eureka又是AP模型，只需要满足最终可用就行。所以它在这里用到多级缓存来实现读写分离。注册方法写的时候直接写内存注册表，写完表之后主动失效读写缓存。

获取注册信息接口先从只读缓存取，只读缓存没有再去读写缓存取，读写缓存没有再去内存注册表里取（不只是取，此处较复杂）。并且，读写缓存会更新回写只读缓存

*   responseCacheUpdateIntervalMs ： readOnlyCacheMap 缓存更新的定时器时间间隔，默认为30秒
*   responseCacheAutoExpirationInSeconds : readWriteCacheMap 缓存过期时间，默认为 180 秒。
    **2.4.2 服务注册的缓存失效**
    在AbstractInstanceRegistry.register方法的最后，会调用invalidateCache(registrant.getAppName(), registrant.getVIPAddress(),registrant.getSecureVipAddress()); 方法，使得读写缓存失效。



```
public void invalidate(Key... keys) {
    for (Key key : keys) {
        logger.debug("Invalidating the response cache key : {} {} {} {}, {}",
                     key.getEntityType(), key.getName(), key.getVersion(),
                     key.getType(), key.getEurekaAccept());
        readWriteCacheMap.invalidate(key);
        Collection<Key> keysWithRegions = regionSpecificKeys.get(key);
        if (null != keysWithRegions && !keysWithRegions.isEmpty()) {
            for (Key keysWithRegion : keysWithRegions) {
                logger.debug("Invalidating the response cache key : {} {} {} {} {}",
                             key.getEntityType(), key.getName(),
                             key.getVersion(), key.getType(), key.getEurekaAccept());
                readWriteCacheMap.invalidate(keysWithRegion);
            }
        }
    }
}
```









**2.4.3 定时同步缓存**
ResponseCacheImpl的构造方法中，会启动一个定时任务，这个任务会定时检查写缓存中的数据变化，进行更新和同步。



```
private TimerTask getCacheUpdateTask() {
    return new TimerTask() {
        @Override
        public void run() {
            logger.debug("Updating the client cache from response cache");
            for (Key key : readOnlyCacheMap.keySet()) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Updating the client cache from response cache for key : {} {} {} {}",
                                 key.getEntityType(), key.getName(),
                                 key.getVersion(), key.getType());
                }
                try {
                    CurrentRequestVersion.set(key.getVersion());
                    Value cacheValue = readWriteCacheMap.get(key);
                    Value currentCacheValue = readOnlyCacheMap.get(key);
                    if (cacheValue != currentCacheValue) {
                        readOnlyCacheMap.put(key, cacheValue);
                    }
                } catch (Throwable th) {
                    logger.error("Error while updating the client cache from response cache for key {}", key.toStringCompact(), th);
                } finally {
                    CurrentRequestVersion.remove();
                }
            }
        }
    };
}
```









**2.5 服务续约**
所谓的服务续约，其实就是一种心跳检查机制。客户端会定期发送心跳来续约。那么简单给大家看一下代码的实现

**2.5.1 initScheduledTasks**
客户端会在
DiscoveryClient.initScheduledTasks 中，创建一个心跳检测的定时任务



```
// Heartbeat timer
scheduler.schedule(
    new TimedSupervisorTask(
        "heartbeat",
        scheduler,
        heartbeatExecutor,
        renewalIntervalInSecs,
        TimeUnit.SECONDS,
        expBackOffBound,
        new HeartbeatThread()
    ),
    renewalIntervalInSecs, TimeUnit.SECONDS);
```









**2.5.2 HeartbeatThread**
然后这个定时任务中，会执行一个 HearbeatThread 的线程，这个线程会定时调用renew()来做续约。



```
//每隔30s发送一个心跳请求到
private class HeartbeatThread implements Runnable {
    public void run() {
        if (renew()) {
            lastSuccessfulHeartbeatTimestamp = System.currentTimeMillis();
        }
    }
}
```









**2.5.3 服务端收到心跳请求的处理**
在ApplicationResource.getInstanceInfo这个接口中，会返回一个InstanceResource的实例，在该实例下，定义了一个statusUpdate的接口来更新状态



```
@Path("{id}")
public InstanceResource getInstanceInfo(@PathParam("id") String id) {
    return new InstanceResource(this, id, serverConfig, registry);
}
```









**2.5.4 InstanceResource.statusUpdate()**
在该方法中，我们重点关注 registry.statusUpdate 这个方法，它会调用
AbstractInstanceRegistry.statusUpdate来更新指定服务提供者在服务端存储的信息中的变化。



```
@PUT
@Path("status")
public Response statusUpdate(
    @QueryParam("value") String newStatus,
    @HeaderParam(PeerEurekaNode.HEADER_REPLICATION) String isReplication,
    @QueryParam("lastDirtyTimestamp") String lastDirtyTimestamp) {
    try {
        if (registry.getInstanceByAppAndId(app.getName(), id) == null) {
            logger.warn("Instance not found: {}/{}", app.getName(), id);
            return Response.status(Status.NOT_FOUND).build();
        }
        boolean isSuccess = registry.statusUpdate(app.getName(), id,                      
                                                  InstanceStatus.valueOf(newStatus), lastDirtyTimestamp,
                                                  "true".equals(isReplication));
        if (isSuccess) {
            logger.info("Status updated: {} - {} - {}", app.getName(), id,
                        newStatus);
            return Response.ok().build();
        } else {
            logger.warn("Unable to update status: {} - {} - {}", app.getName(),
                        id, newStatus);
            return Response.serverError().build();
        }
    } catch (Throwable e) {
        logger.error("Error updating instance {} for status {}", id,
                     newStatus);
        return Response.serverError().build();
    }
}
```









**2.5.5 AbstractInstanceRegistry.statusUpdate**
在这个方法中，会拿到应用对应的实例列表，然后调用Lease.renew()去进行心跳续约。



```
public boolean statusUpdate(String appName, String id,
                            InstanceStatus newStatus, String
                            lastDirtyTimestamp,
                            boolean isReplication) {
    try {
        read.lock();
        // 更新状态的次数 状态统计
        STATUS_UPDATE.increment(isReplication);
        // 从本地数据里面获取实例信息，
        Map<String, Lease<InstanceInfo>> gMap = registry.get(appName);
        Lease<InstanceInfo> lease = null;
        if (gMap != null) {
            lease = gMap.get(id);
        }
        // 实例不存在，则直接返回，表示失败
        if (lease == null) {
            return false;
        } else {
            // 执行一下lease的renew方法，里面主要是更新了这个instance的最后更新时间。
            lease.renew();
            // 获取instance实例信息
            InstanceInfo info = lease.getHolder();
            // Lease is always created with its instance info object.
            // This log statement is provided as a safeguard, in case this invariant is violated.
            if (info == null) {
                logger.error("Found Lease without a holder for instance id {}",
                             id);
            }
            // 当instance信息不为空时，并且实例状态发生了变化
            if ((info != null) && !(info.getStatus().equals(newStatus))) {
                // 如果新状态是UP的状态，那么启动一下serviceUp() , 主要是更新服务的注册时
                间
                    if (InstanceStatus.UP.equals(newStatus)) {
                        lease.serviceUp();
                    }
                // 将instance Id 和这个状态的映射信息放入覆盖缓存MAP里面去
                overriddenInstanceStatusMap.put(id, newStatus);
                // Set it for transfer of overridden status to replica on
                // 设置覆盖状态到实例信息里面去
                info.setOverriddenStatus(newStatus);
                long replicaDirtyTimestamp = 0;
                info.setStatusWithoutDirty(newStatus);
                if (lastDirtyTimestamp != null) {
                    replicaDirtyTimestamp = Long.valueOf(lastDirtyTimestamp);
                }
                // If the replication's dirty timestamp is more than the existing one, just update
                // it to the replica's.
                // 如果replicaDirtyTimestamp 的时间大于instance的getLastDirtyTimestamp() ,则更新

                if (replicaDirtyTimestamp > info.getLastDirtyTimestamp()) {
                    info.setLastDirtyTimestamp(replicaDirtyTimestamp);
                }
                info.setActionType(ActionType.MODIFIED);
                recentlyChangedQueue.add(new RecentlyChangedItem(lease));
                info.setLastUpdatedTimestamp();
                //更新写缓存
                invalidateCache(appName, info.getVIPAddress(),
                                info.getSecureVipAddress());
            }
            return true;
        }
    } finally {
        read.unlock();
    }
}
```









至此，心跳续约功能就分析完成了。

**2.6 服务发现**
我们继续来研究服务的发现过程，就是客户端需要能够满足两个功能

在启动的时候获取指定服务提供者的地址列表
Eureka server端地址发生变化时，需要动态感知
**2.6.1 DiscoveryClient构造时查询**
构造方法中，如果当前的客户端默认开启了fetchRegistry，则会从eureka-server中拉取数据。



```
DiscoveryClient(ApplicationInfoManager applicationInfoManager,
                EurekaClientConfig config, AbstractDiscoveryClientOptionalArgs args,
                Provider<BackupRegistry> backupRegistryProvider,
                EndpointRandomizer endpointRandomizer) {
    if (clientConfig.shouldFetchRegistry() && !fetchRegistry(false)) {
        fetchRegistryFromBackup();
    }
}
```









**2.6.2 fetchRegistry**



```
private boolean fetchRegistry(boolean forceFullRegistryFetch) {
    Stopwatch tracer = FETCH_REGISTRY_TIMER.start();
    try {
        // If the delta is disabled or if it is the first time, get all
        // applications
        Applications applications = getApplications();
        if (clientConfig.shouldDisableDelta()
            ||
            (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
            || forceFullRegistryFetch
            || (applications == null)
            || (applications.getRegisteredApplications().size() == 0)
            || (applications.getVersion() == -1)) //Client application does not have latest library supporting delta
        {
            logger.info("Disable delta property : {}",
                        clientConfig.shouldDisableDelta());
            logger.info("Single vip registry refresh property : {}",
                        clientConfig.getRegistryRefreshSingleVipAddress());
            logger.info("Force full registry fetch : {}",
                        forceFullRegistryFetch);
            logger.info("Application is null : {}", (applications == null));
            logger.info("Registered Applications size is zero : {}",
                        (applications.getRegisteredApplications().size() == 0));
            logger.info("Application version is -1: {}",
                        (applications.getVersion() == -1));
            getAndStoreFullRegistry();
        } else {
            getAndUpdateDelta(applications);
        }
        applications.setAppsHashCode(applications.getReconcileHashCode());
        logTotalInstances();
    } catch (Throwable e) {
        logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
        return false;
    } finally {
        if (tracer != null) {
            tracer.stop();
        }
    }
    // Notify about cache refresh before updating the instance remote status
    onCacheRefreshed();
    // Update remote status based on refreshed data held in the cache
    updateInstanceRemoteStatus();
    // registry was fetched successfully, so return true
    return true;
}
```









**2.6.3 定时刷新本地地址列表**
任务每隔30s更新一次
在DiscoveryClient构造的时候，会初始化一些任务，这个在前面咱们分析过了。其中有一个任务动态更新本地服务地址列表，叫 cacheRefreshTask 。

这个任务最终执行的是CacheRefreshThread这个线程。它是一个周期性执行的任务，具体我们来看一下。



```
private void initScheduledTasks() {
    if (clientConfig.shouldFetchRegistry()) {
        // registry cache refresh timer
        int registryFetchIntervalSeconds =
            clientConfig.getRegistryFetchIntervalSeconds();
        int expBackOffBound =
            clientConfig.getCacheRefreshExecutorExponentialBackOffBound();
        cacheRefreshTask = new TimedSupervisorTask(
            "cacheRefresh",
            scheduler,
            cacheRefreshExecutor,
            registryFetchIntervalSeconds,
            TimeUnit.SECONDS,
            expBackOffBound,
            new CacheRefreshThread()
        );
        scheduler.schedule(
            cacheRefreshTask,
            registryFetchIntervalSeconds, TimeUnit.SECONDS);
    }
```









**2.6.4 TimedSupervisorTask**
从整体上看，TimedSupervisorTask是固定间隔的周期性任务，一旦遇到超时就会将下一个周期的间隔时间调大，如果连续超时，那么每次间隔时间都会增大一倍，一直到达外部参数设定的上限为止，一旦新任务不再超时，间隔时间又会自动恢复为初始值。这种设计还是值得学习的。



```
public void run() {
      Future future = null;
  try {
    //使用Future，可以设定子线程的超时时间，这样当前线程就不用无限等待了
    future = executor.submit(task);
    threadPoolLevelGauge.set((long) executor.getActiveCount());
    //指定等待子线程的最长时间
    future.get(timeoutMillis, TimeUnit.MILLISECONDS);  // block until done or timeout
    //delay是个很有用的变量，后面会用到，这里记得每次执行任务成功都会将delay重置
    delay.set(timeoutMillis);
    threadPoolLevelGauge.set((long) executor.getActiveCount());
 } catch (TimeoutException e) {
    logger.error("task supervisor timed out", e);
    timeoutCounter.increment();
    long currentDelay = delay.get();
    //任务线程超时的时候，就把delay变量翻倍，但不会超过外部调用时设定的最大延时时间
    long newDelay = Math.min(maxDelay, currentDelay * 2);
    //设置为最新的值，考虑到多线程，所以用了CAS
    delay.compareAndSet(currentDelay, newDelay);
 } catch (RejectedExecutionException e) {
    //一旦线程池的阻塞队列中放满了待处理任务，触发了拒绝策略，就会将调度器停掉
    if (executor.isShutdown() || scheduler.isShutdown()) {
      logger.warn("task supervisor shutting down, reject the task", e);
   } else {
      logger.error("task supervisor rejected the task", e);
   }
    rejectedCounter.increment();
 } catch (Throwable e) {
    //一旦出现未知的异常，就停掉调度器
    if (executor.isShutdown() || scheduler.isShutdown()) {
      logger.warn("task supervisor shutting down, can't accept the task");
   } else {
      logger.error("task supervisor threw an exception", e);
   }
    throwableCounter.increment();
 } finally {
    //这里任务要么执行完毕，要么发生异常，都用cancel方法来清理任务；
    if (future != null) {
      future.cancel(true);
   }
    //只要调度器没有停止，就再指定等待时间之后在执行一次同样的任务
    if (!scheduler.isShutdown()) {
      //这里就是周期性任务的原因：只要没有停止调度器，就再创建一次性任务，执行时间时dealy的值，
      //假设外部调用时传入的超时时间为30秒（构造方法的入参timeout），最大间隔时间为50秒(构造方法的入参expBackOffBound)
      //如果最近一次任务没有超时，那么就在30秒后开始新任务，
      //如果最近一次任务超时了，那么就在50秒后开始新任务（异常处理中有个乘以二的操作，乘以二后的60秒超过了最大间隔50秒）
      scheduler.schedule(this, delay.get(), TimeUnit.MILLISECONDS);
   }
 }
}
```









**2.6.5 refreshRegistry**
这段代码主要两个逻辑

* 判断remoteRegions是否发生了变化

* 调用fetchRegistry获取本地服务地址缓存



  ```
  @VisibleForTesting
  void refreshRegistry() {
      try {
          boolean isFetchingRemoteRegionRegistries =
              isFetchingRemoteRegionRegistries();
          boolean remoteRegionsModified = false;
          //如果部署在aws环境上，会判断最后一次远程区域更新的信息和当前远程区域信息进行比较，如果不想等，则更新
          String latestRemoteRegions =
              clientConfig.fetchRegistryForRemoteRegions();
          if (null != latestRemoteRegions) {
              String currentRemoteRegions = remoteRegionsToFetch.get();
              if (!latestRemoteRegions.equals(currentRemoteRegions)) {
                  //判断最后一次
              }
              boolean success = fetchRegistry(remoteRegionsModified);
              if (success) {
                  registrySize = localRegionApps.get().size();
                  lastSuccessfulRegistryFetchTimestamp =
                      System.currentTimeMillis();
              }
              // 省略
          } catch (Throwable e) {
              logger.error("Cannot fetch registry from server", e);
          }
      }
  ```









**2.6.6 fetchRegistry**



  ```
  private boolean fetchRegistry(boolean forceFullRegistryFetch) {
      Stopwatch tracer = FETCH_REGISTRY_TIMER.start();
      try {
          // If the delta is disabled or if it is the first time, get all
          // applications
          // 取出本地缓存的服务列表信息
          Applications applications = getApplications();
          //判断多个条件，确定是否触发全量更新，如下任一个满足都会全量更新：
          //1\. 是否禁用增量更新；
          //2\. 是否对某个region特别关注；
          //3\. 外部调用时是否通过入参指定全量更新；
          //4\. 本地还未缓存有效的服务列表信息；
          if (clientConfig.shouldDisableDelta()
              ||
              (!Strings.isNullOrEmpty(clientConfig.getRegistryRefreshSingleVipAddress()))
              || forceFullRegistryFetch
              || (applications == null)
              || (applications.getRegisteredApplications().size() == 0)
              || (applications.getVersion() == -1)) //Client application does not
              have latest library supporting delta
          {
              //调用全量更新
              getAndStoreFullRegistry();
          } else {
              //调用增量更新
              getAndUpdateDelta(applications);
          }
          //重新计算和设置一致性hash码
          applications.setAppsHashCode(applications.getReconcileHashCode());
          logTotalInstances(); //日志打印所有应用的所有实例数之和
      } catch (Throwable e) {
          logger.error(PREFIX + "{} - was unable to refresh its cache! status = {}", appPathIdentifier, e.getMessage(), e);
          return false;
      } finally {
          if (tracer != null) {
              tracer.stop();
          }
      }
      //将本地缓存更新的事件广播给所有已注册的监听器，注意该方法已被CloudEurekaClient类重写
      onCacheRefreshed();
      // Update remote status based on refreshed data held in the cache
      //检查刚刚更新的缓存中，有来自Eureka server的服务列表，其中包含了当前应用的状态，
      //当前实例的成员变量lastRemoteInstanceStatus，记录的是最后一次更新的当前应用状态，
      //上述两种状态在updateInstanceRemoteStatus方法中作比较 ，如果不一致，就更新lastRemoteInstanceStatus，并且广播对应的事件
      updateInstanceRemoteStatus();
      // registry was fetched successfully, so return true
      return true;
  }
  ```









**2.6.7 getAndStoreFullRegistry**
从eureka server端获取服务注册中心的地址信息，然后更新并设置到本地缓存 localRegionApps 。



```
private void getAndStoreFullRegistry() throws Throwable {
    long currentUpdateGeneration = fetchRegistryGeneration.get();
    logger.info("Getting all instance registry info from the eureka server");
    Applications apps = null;
    EurekaHttpResponse<Applications> httpResponse =
        clientConfig.getRegistryRefreshSingleVipAddress() == null
        ? eurekaTransport.queryClient.getApplications(remoteRegionsRef.get())
        :
    eurekaTransport.queryClient.getVip(clientConfig.getRegistryRefreshSingleVipAddre
                                       ss(), remoteRegionsRef.get());
    if (httpResponse.getStatusCode() == Status.OK.getStatusCode()) {
        apps = httpResponse.getEntity();
    }
    logger.info("The response status is {}", httpResponse.getStatusCode());
    if (apps == null) {
        logger.error("The application is null for some reason. Not storing this information");
    } else if (fetchRegistryGeneration.compareAndSet(currentUpdateGeneration,
                                                     currentUpdateGeneration + 1)) {
        localRegionApps.set(this.filterAndShuffle(apps));
        logger.debug("Got full registry with apps hashcode {}",
                     apps.getAppsHashCode());
    } else {
        logger.warn("Not updating applications as another thread is updating it already");
    }
}
```









**2.6.8 服务端查询服务地址流程**
前面我们知道，客户端发起服务地址的查询有两种，一种是全量、另一种是增量。对于全量查询的请求，会调用Eureka-server的ApplicationsResource的getContainers方法。

而对于增量请求，会调用
ApplicationsResource.getContainerDifferential。

**2.6.9 ApplicationsResource.getContainers**
接收客户端发送的获取全量注册信息请求。



```
@GET
public Response getContainers(@PathParam("version") String version,
                              @HeaderParam(HEADER_ACCEPT) String acceptHeader,
                              @HeaderParam(HEADER_ACCEPT_ENCODING) String
                              acceptEncoding,
                              @HeaderParam(EurekaAccept.HTTP_X_EUREKA_ACCEPT)
                              String eurekaAccept,
                              @Context UriInfo uriInfo,
                              @Nullable @QueryParam("regions") String
                              regionsStr) {
    boolean isRemoteRegionRequested = null != regionsStr &&
        !regionsStr.isEmpty();
    String[] regions = null;
    if (!isRemoteRegionRequested) {
        EurekaMonitors.GET_ALL.increment();
    } else {
        regions = regionsStr.toLowerCase().split(",");
        Arrays.sort(regions); // So we don't have different caches for same regions queried in different order.
        EurekaMonitors.GET_ALL_WITH_REMOTE_REGIONS.increment();
    }
    // EurekaServer无法提供服务，返回403
    if (!registry.shouldAllowAccess(isRemoteRegionRequested)) {
        return Response.status(Status.FORBIDDEN).build();
    }
    CurrentRequestVersion.set(Version.toEnum(version));
    KeyType keyType = Key.KeyType.JSON;// 设置返回数据格式，默认JSON
    String returnMediaType = MediaType.APPLICATION_JSON;
    if (acceptHeader == null || !acceptHeader.contains(HEADER_JSON_VALUE)) {
        // 如果接收到的请求头部没有具体格式信息，则返回格式为XML
        keyType = Key.KeyType.XML;
        returnMediaType = MediaType.APPLICATION_XML;
    }
    // 构建缓存键
    Key cacheKey = new Key(Key.EntityType.Application,
                           ResponseCacheImpl.ALL_APPS,
                           keyType, CurrentRequestVersion.get(),
                           EurekaAccept.fromString(eurekaAccept), regions
                          );
    // 返回不同的编码类型的数据，去缓存中取数据的方法基本一致
    Response response;
    if (acceptEncoding != null && acceptEncoding.contains(HEADER_GZIP_VALUE)) {
        response = Response.ok(responseCache.getGZIP(cacheKey))
            .header(HEADER_CONTENT_ENCODING, HEADER_GZIP_VALUE)
            .header(HEADER_CONTENT_TYPE, returnMediaType)
            .build();
    } else {
        response = Response.ok(responseCache.get(cacheKey))
            .build();
    }
    CurrentRequestVersion.remove();
    return response;
}
```









**2.6.10 responseCache.getGZIP**
从缓存中读取数据。



```
public byte[] getGZIP(Key key) {
    Value payload = getValue(key, shouldUseReadOnlyResponseCache);
    if (payload == null) {
        return null;
    }
    return payload.getGzipped();
}
Value getValue(final Key key, boolean useReadOnlyCache) {
    Value payload = null;
    try {
        if (useReadOnlyCache) {
            final Value currentPayload = readOnlyCacheMap.get(key);
            if (currentPayload != null) {
                payload = currentPayload;
            } else {
                payload = readWriteCacheMap.get(key);
                readOnlyCacheMap.put(key, payload);
            }
        } else {
            payload = readWriteCacheMap.get(key);
        }
    } catch (Throwable t) {
        logger.error("Cannot get value for key : {}", key, t);
    }
    return payload;
}
```

# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning