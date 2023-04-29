作者 | 宇木木兮
来源 |今日头条

**学习目标**

1.  为什么加一个注解就能实现远程过程调用呢？推导它底层的实现主流程？
2.  OpenFeign怎么实现RPC的基本功能的
3.  通过源码验证
    **第1章 OpenFeign主流程推导**
    要明确OpenFeign的主流程首先我们还是要明确它的核心目标是什么？

说白了，OpenFeign最核心的目标就是让客户端在远程调用过程中不需要做什么多余的操作，只要拿到一个对象，然后调用该对象的方法就好了，剩下的操作都交给OpenFeign去帮你完成，那剩下一些什么操作呢？

1.  首先肯定是保证网络通信，那我们大胆地猜测一下，OpenFeign其实底层帮我们封装了请求的地址、端口、请求参数以及响应的参数。
2.  其次，当我们要用对象去请求方法，那这个对象是远程的服务，这个对象肯定不简单，这里也大胆猜测一下，该对象也是OpenFeign给我们创建的。
3.  然后在调用过程中，如果服务是由多台服务器提供的，那又涉及到负载均衡了，这肯定也是OpenFeign帮我们完成了。
4.  除了上面的问题，再联想一下，既然存在服务端是集群的情况，那服务端的地址和端口还需要一个注册中心来注册，这肯定也不能由客户端来完成，因为客户端只关注业务代码。那想都不用想，也是OpenFeign来完成了。

OK，上面推导了OpenFeign应该完成的主要目标，接下来我们再来分析分析它是怎么做的。
![SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b45564c706ab4113cc23492cc0796907b851f8.jpg "SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区")之前的文章有讲过一个概念，不管是什么组件，只要是集成spring或者springboot的话，那一定是想通过spring或者springboot去管理bean对象的创建的，当通过容器拿到对象之后再去调用对象的核心方法，那OpenFeign在集成springboot的时候理念也应该是这样。

1.  所以第一步，OpenFeign集成springboot，通过springboot拿到核心bean对象，例如上图中的userService对象。
2.  这个对象肯定不简单，不可能只有getUser的功能。那想一想，spring中做功能增强可以用什么来做呢？——代理嘛，那这个对象的类型就呼之欲出了，代理对象。
3.  调用代理对象的方法时，流程先进入到invoke中，在这个invoke中，做的增强包括了负载均衡LoadBalance，因为我在真正调用getUser的时候要知道具体是调用哪台服务器的服务。
4.  负载均衡做完就得拼接具体的http请求参数，请求头，请求地址，请求端口了。
    OK，以上分析了OpenFeign底层要实现的具体功能，也分析了它的处理流程，那么接下来我们通过源码来验证一下，它是不是这么玩的。

**第2章 源码验证**

![SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/93070e354fbacedc2b85274846d2dde7dd170a.jpg "SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区")**2.1 EnableFeignClients**
我们从下面这个注解进行切入，这个注解开启了FeignClient的解析过程。



```
@EnableFeignClients(basePackages = "com.example.client")
```









这个注解的声明如下，它用到了一个@Import注解，我们知道Import是用来导入一个配置类的，接下来去看一下FeignClientsRegistrar的定义。



```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Documented
@Import(FeignClientsRegistrar.class)
public @interface EnableFeignClients {
}
```









FeignClientsRegistrar实现了
ImportBeanDefinitionRegistrar，它是一个动态注入bean的接口，Spring Boot启动的时候，会去调用这个类中的registerBeanDefinitions来实现动态Bean的装载。registerBeanDefinitions是在spring容器启动时执行invokeBeanFactoryPostProcessors方法，然后对相应的类进行解析注册，它的作用类似于ImportSelector。



```
class FeignClientsRegistrar implements ImportBeanDefinitionRegistrar, ResourceLoaderAware,EnvironmentAware {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
        registerDefaultConfiguration(metadata, registry);
        registerFeignClients(metadata, registry);
    }
}
```









**2.1.1 ImportBeanDefinitionRegistrar**
简单给大家演示一下
ImportBeanDefinitionRegistrar的作用。

*   定义一个需要被装载到IOC容器中的类HelloService



```
public class HelloService {
}
```









*   定义一个Registrar的实现，定义一个bean，装载到IOC容器



```
public class FeignImportBeanDefinitionRegistrar implements ImportBeanDefinitionRegistrar {
    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        BeanDefinition beanDefinition = new GenericBeanDefinition();
        beanDefinition.setBeanClassName(HelloService.class.getName());
        registry.registerBeanDefinition("helloService",beanDefinition);
    }
}
```









* 定义一个注解类



  ```
  @Retention(RetentionPolicy.RUNTIME)
  @Target({ElementType.TYPE})
  @Documented
  @Import({FeignImportBeanDefinitionRegistrar.class})
  public @interface EnableFeignTest {
  }
  ```









* 启动类



```
@EnableFeignClients(basePackages = "com.example.clients")
@EnableFeignTest
@SpringBootApplication
public class OpenfeignUserServiceApplication {
    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(OpenfeignUserServiceApplication.class, args);
        System.out.println(context.getBean(HelloService.class));
    }

}
```









*   通过结果演示可以发现，HelloService这个bean 已经装载到了IOC容器。
    这就是动态装载的功能实现，它相比于@Configuration配置注入，会多了很多的灵活性。 ok，再回到FeignClient的解析中来。

**2.1.2 FeignClientsRegistrar**

* registerDefaultConfiguration 方法内部从 SpringBoot 启动类上检查是否有@EnableFeignClients, 有该注解的话， 则完成 Feign 框架相关的一些配置内容注册

* registerFeignClients 方法内部从 classpath 中， 扫描获得 @FeignClient 修饰的类， 将类的内容解析为 BeanDefinition , 最终通过调用 Spring 框架中的BeanDefinitionReaderUtils.resgisterBeanDefinition 将解析处理过的 FeignClientBeanDeifinition 添加到 spring 容器中.



  ```
  @Override
  public void registerBeanDefinitions(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
      //注册@EnableFeignClients中定义defaultConfiguration属性下的类，包装成FeignClientSpecification，注册到Spring容器。
      //在@FeignClient中有一个属性：configuration，这个属性是表示各个FeignClient自定义的配置类，后面也会通过调用registerClientConfiguration方法来注册成FeignClientSpecification到容器。
      //所以，这里可以完全理解在@EnableFeignClients中配置的是做为兜底的配置，在各个@FeignClient配置的就是自定义的情况。
      registerDefaultConfiguration(metadata, registry);
      registerFeignClients(metadata, registry);
  }
  ```









**2.2 registerDefaultConfiguration**



  ```
  private void registerDefaultConfiguration(AnnotationMetadata metadata, BeanDefinitionRegistry registry) {
      // 获取到metadata中关于EnableFeignClients的属性值键值对。
      Map<String, Object> defaultAttrs = metadata.getAnnotationAttributes(EnableFeignClients.class.getName(), true);
      //如果配置了defaultConfiguration 进行配置,如果没有使用默认的configuration
      if (defaultAttrs != null && defaultAttrs.containsKey("defaultConfiguration")) {
          String name;
          if (metadata.hasEnclosingClass()) {
              name = "default." + metadata.getEnclosingClassName();
          } else {
              name = "default." + metadata.getClassName();
          }
          //进行注册
          this.registerClientConfiguration(registry, name, defaultAttrs.get("defaultConfiguration"));
      }
  
  }
  ```











```
private void registerClientConfiguration(BeanDefinitionRegistry registry, Object name, Object configuration) {
    //使用BeanDefinitionBuilder来生成BeanDefinition,并把它进行注册
    BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition(FeignClientSpecification.class);
    builder.addConstructorArgValue(name);
    builder.addConstructorArgValue(configuration);
    registry.registerBeanDefinition(name + "." + FeignClientSpecification.class.getSimpleName(), builder.getBeanDefinition());
}
```









方法的入参BeanDefinitionRegistry是spring框架用于动态注册BeanDefinition信息的接口，调用registerBeanDefinition方法可以将BeanDefinition注册到Spring容器中，其中name属性就是注册的BeanDefinition的名称，在这里它注册了一个FeignClientSpecification的对象。

FeignClientSpecification实现了
NamedContextFactory.Specification接口，它是Feign实例化的重要一环，在上面的方法中，它持有自定义配置的组件实例，SpringCloud使用NamedContextFactory创建一些列的运行上下文ApplicationContext来让对应的Specification在这些上下文中创建实例对象。

NamedContextFactory有3个功能：

*   创建AnnotationConfigApplicationContext上下文。
*   在上下文中创建并获取bean实例。
*   当上下文销毁时清除其中的feign实例。
    NamedContextFactory有个非常重要的子类FeignContext，用于存储各种OpenFeign的组件实例。



```
public class FeignContext extends NamedContextFactory<FeignClientSpecification> {
    public FeignContext() {
       super(FeignClientsConfiguration.class, "feign", "feign.client.name");
    }
 }
```









FeignContext是哪里构建的呢？

配置见：
pring-cloud-openfeign-core-2.2.3.RELEASE.jar!\META-INF\spring.factories![SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d830f20713aca01438472786f5f84c7058232b.jpg "SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区")**2.2.1 FeignAutoConfiguration**

![SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/894c73791fbf7215782217d7132bc57f004085.jpg "SpringCloud系列—Spring Cloud 源码分析之OpenFeign-开源基础软件社区")

将默认的FeignClientsConfiguration作为参数传递给构造函数

FeignContext创建的时候会将之前FeignClientSpecification通过setConfigurations设置给context上下文。

**2.2.2 createContext**
代码详见：
org.springframework.cloud.context.named.NamedContextFactory#createContext方法。

FeignContext的父类的createContext方法会将创建
AnnotationConfigApplicationContext实例，这实例将作为当前上下文的子上下文，用于关联feign组件的不同实例。在调用FeignClientFactoryBean的getObject方法时调用。（createContext调用在下文会讲解）



```
protected AnnotationConfigApplicationContext createContext(String name) {
    AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext();
	//获取name所对应的configuration,如果有就注册到子context中
    if (this.configurations.containsKey(name)) {
        for (Class<?> configuration : this.configurations.get(name)
             .getConfiguration()) {
            context.register(configuration);
        }
    }
    //注册default的Configuration,也就是 FeignClientsRegistrar类中registerDefaultConfiguration方法中注册的Configuration
    for (Map.Entry<String, C> entry : this.configurations.entrySet()) {
        if (entry.getKey().startsWith("default.")) {
            for (Class<?> configuration : entry.getValue().getConfiguration()) {
                context.register(configuration);
            }
        }
    }
    //注册PropertyPlaceholderAutoConfiguration
    context.register(PropertyPlaceholderAutoConfiguration.class,
                     this.defaultConfigType);
    //置Environment的propertySources属性源
    context.getEnvironment().getPropertySources().addFirst(new MapPropertySource(
        this.propertySourceName,
        Collections.<String, Object>singletonMap(this.propertyName, name)));
    if (this.parent != null) {
        // Uses Environment from parent as well as beans
        context.setParent(this.parent);
        // jdk11 issue
        // https://github.com/spring-cloud/spring-cloud-netflix/issues/3101
        context.setClassLoader(this.parent.getClassLoader());
    }
    context.setDisplayName(generateDisplayName(name));
    context.refresh();
    return context;
}
```









由于NamedContextFactory实现了DisposableBean，所以当实例消亡的时候会调用



```
@Override
public void destroy() {
    Collection<AnnotationConfigApplicationContext> values = this.contexts.values();
    for (AnnotationConfigApplicationContext context : values) {
        // This can fail, but it never throws an exception (you see stack traces
        // logged as WARN).
        context.close();
    }
    this.contexts.clear();
}
```









总结：NamedContextFactory会创建出
AnnotationConfigApplicationContext实例，并以name作为唯一标识，然后每个AnnotationConfigApplicationContext实例都会注册部分配置类，从而可以给出一系列的基于配置类生成的组件实例，这样就可以基于name来管理一系列的组件实例，为不同的FeignClient准备不同配置组件实例。

**2.3 registerFeignClients**
这个方法主要是扫描类路径下所有的@FeignClient注解，然后进行动态Bean的注入。它最终会调用 registerFeignClient 方法。



```
public void registerFeignClients(AnnotationMetadata metadata,BeanDefinitionRegistry registry) {
    //省略代码...
    registerFeignClient(registry, annotationMetadata, attributes);
}
```









在这个方法中，就是去组装BeanDefinition，也就是Bean的定义，然后注册到Spring IOC容器。



```
private void registerFeignClient(BeanDefinitionRegistry registry,AnnotationMetadata annotationMetadata, Map<String, Object> attributes) {
    String className = annotationMetadata.getClassName();
    BeanDefinitionBuilder definition = BeanDefinitionBuilder.genericBeanDefinition(FeignClientFactoryBean.class);
    //省略代码...
    BeanDefinitionHolder holder = new BeanDefinitionHolder(beanDefinition,className,new String[] { alias });
    BeanDefinitionReaderUtils.registerBeanDefinition(holder, registry);
}
```









我们关注一下，BeanDefinitionBuilder是用来构建一个BeanDefinition的，它是通过genericBeanDefinition 来构建的，并且传入了一个FeignClientFactoryBean的类。

我们可以发现，FeignClient被动态注册成了一个FactoryBean

> Spring Cloud FengnClient实际上是利用Spring的代理工厂来生成代理类，所以在这里才会把所有的FeignClient的BeanDefinition设置为FeignClientFactoryBean类型，而FeignClientFactoryBean继承自FactoryBean，它是一个工厂Bean。
>
> 在Spring中，FactoryBean是一个工厂Bean，用来创建代理Bean。
>
> 工厂 Bean 是一种特殊的 Bean, 对于 Bean 的消费者来说， 他逻辑上是感知不到这个 Bean 是普通的 Bean 还是工厂 Bean, 只是按照正常的获取 Bean 方式去调用， 但工厂bean 最后返回的实例不是工厂Bean 本身， 而是执行工厂 Bean 的 getObject 逻辑返回的示例。（也就是在实例化工厂Bean的时候会去调用它的getObject方法）



```
public static BeanDefinitionBuilder genericBeanDefinition(Class<?> beanClass) {
    BeanDefinitionBuilder builder = new BeanDefinitionBuilder(new GenericBeanDefinition());
    builder.beanDefinition.setBeanClass(beanClass);
    return builder;
}
```









简单来说，FeignClient标注的这个接口，会通过
FeignClientFactoryBean.getObject()这个方法获得一个代理对象。

**2.3.1 FeignClientFactoryBean.getObject**
getObject调用的是getTarget方法，它从applicationContext取出FeignContext，FeignContext继承了NamedContextFactory，它是用来来统一维护feign中各个feign客户端相互隔离的上下文。

接着，构建feign.builder，在构建时会向FeignContext获取配置的Encoder，Decoder等各种信息。FeignContext在上篇中已经提到会为每个Feign客户端分配了一个容器，它们的父容器就是spring容器

配置完Feign.Builder之后，再判断是否需要LoadBalance，如果需要，则通过LoadBalance的方法来设置。实际上他们最终调用的是Target.target()方法。



```
@Override
public Object getObject() throws Exception {
    return getTarget();
}
<T> T getTarget() {
    //实例化Feign上下文对象FeignContext
    FeignContext context = this.applicationContext.getBean(FeignContext.class);
    Feign.Builder builder = feign(context);//构建Builder对象
    if (!StringUtils.hasText(this.url)) {//如果url为空，则走负载均衡，生成有负载均衡功能的代理类
        if (!this.name.startsWith("http")) {
            this.url = "http://" + this.name;
        }
        else {
            this.url = this.name;
        }
        this.url += cleanPath();
        return (T) loadBalance(builder, context,new HardCodedTarget<>(this.type, this.name,this.url));
    }
    //如果指定了url，则生成默认的代理类
    if (StringUtils.hasText(this.url) && !this.url.startsWith("http")) {
        this.url = "http://" + this.url;
    }
    String url = this.url + cleanPath();
    //调用FeignContext的getInstance方法获取Client对象
    Client client = getOptional(context, Client.class);
    if (client != null) {
        if (client instanceof LoadBalancerFeignClient) {
            // not load balancing because we have a url,
            // but ribbon is on the classpath, so unwrap
            client = ((LoadBalancerFeignClient) client).getDelegate();
        }
        if (client instanceof FeignBlockingLoadBalancerClient) {
            // not load balancing because we have a url,
            // but Spring Cloud LoadBalancer is on the classpath, so unwrap
            client = ((FeignBlockingLoadBalancerClient) client).getDelegate();
        }
        builder.client(client);
    }//生成默认代理类
    Targeter targeter = get(context, Targeter.class);
    return (T) targeter.target(this, builder, context,new HardCodedTarget<>(this.type, this.name,url));
}
```









**2.3.2 loadBalance**
生成具备负载均衡能力的feign客户端，为feign客户端构建起绑定负载均衡客户端.

Client client = (Client)this.getOptional(context, Client.class); 从上下文中获取一个Client，默认是LoadBalancerFeignClient。

它是在FeignRibbonClientAutoConfiguration这个自动装配类中，通过Import实现的



```
@Import({ HttpClientFeignLoadBalancedConfiguration.class,OkHttpFeignLoadBalancedConfiguration.class,DefaultFeignLoadBalancedConfiguration.class })
```











```
protected <T> T loadBalance(Builder builder, FeignContext context,
                            HardCodedTarget<T> target) {
    Client client = (Client)this.getOptional(context, Client.class);
    if (client != null) {
        builder.client(client);
        Targeter targeter = (Targeter)this.get(context, Targeter.class);
        return targeter.target(this, builder, context, target);
    } else {
        throw new IllegalStateException("No Feign Client for loadBalancing defined. Did you forget to include spring-cloud-starter-netflix-ribbon?");
    }
}
```









**2.3.3 DefaultTarget.target**



```
@Override
public <T> T target(FeignClientFactoryBean factory, Feign.Builder feign,FeignContext context, Target.HardCodedTarget<T> target) {
    return feign.target(target);
}
```









**2.3.4 ReflectiveFeign.newInstance**
这个方法是用来创建一个动态代理的方法，在生成动态代理之前，会根据Contract协议（协议解析规则，解析接口类的注解信息，解析成内部的MethodHandler的处理方式。

从实现的代码中可以看到熟悉的Proxy.newProxyInstance方法产生代理类。而这里需要对每个定义的接口方法进行特定的处理实现，所以这里会出现一个MethodHandler的概念，就是对应方法级别的InvocationHandler。



```
public <T> T newInstance(Target<T> target) {
    //根据接口类和Contract协议解析方式，解析接口类上的方法和注解，转换成内部的MethodHandler处理方式
    Map<String, MethodHandler> nameToHandler = this.[targetToHandlersByName.apply(target)];
    Map<Method, MethodHandler> methodToHandler = new LinkedHashMap();
    List<DefaultMethodHandler> defaultMethodHandlers = new LinkedList();
    Method[] var5 = target.type().getMethods();
    int var6 = var5.length;
    for(int var7 = 0; var7 < var6; ++var7) {
        Method method = var5[var7];
        if (method.getDeclaringClass() != Object.class) {
            if (Util.isDefault(method)) {
                DefaultMethodHandler handler = new DefaultMethodHandler(method);
                defaultMethodHandlers.add(handler);
                methodToHandler.put(method, handler);
            } else {
                methodToHandler.put(method,nameToHandler.get(Feign.configKey(target.type(), method)));
            }
        }
    }
    InvocationHandler handler = this.factory.create(target, methodToHandler);
    // 基于Proxy.newProxyInstance 为接口类创建动态实现，将所有的请求转换给InvocationHandler 处理。
    T proxy = Proxy.newProxyInstance(target.type().getClassLoader(), new Class[]{target.type()}, handler);
    Iterator var12 = defaultMethodHandlers.iterator();
    while(var12.hasNext()) {
        DefaultMethodHandler defaultMethodHandler = (DefaultMethodHandler)var12.next();
        defaultMethodHandler.bindTo(proxy);
    }
    return proxy;
}
```









**2.4 接口定义的参数解析**
根据FeignClient接口的描述解析出对应的请求数据。

**2.4.1 targetToHandlersByName.apply(target)**
根据Contract协议规则，解析接口类的注解信息，解析成内部表现：

targetToHandlersByName.apply(target);会解析接口方法上的注解，从而解析出方法粒度的特定的配置信息，然后生产一个SynchronousMethodHandler 然后需要维护一个<method，MethodHandler>的map，放入InvocationHandler的实现FeignInvocationHandler中。



```
public Map<String, MethodHandler> apply(Target target) {
    List<MethodMetadata> metadata = contract.parseAndValidateMetadata(target.type());
    Map<String, MethodHandler> result = new LinkedHashMap<String,MethodHandler>();
    for (MethodMetadata md : metadata) {
        BuildTemplateByResolvingArgs buildTemplate;
        if (!md.formParams().isEmpty() && md.template().bodyTemplate() == null)
        {
            buildTemplate = new BuildFormEncodedTemplateFromArgs(md, encoder, queryMapEncoder,target);
        } else if (md.bodyIndex() != null) {
            buildTemplate = new BuildEncodedTemplateFromArgs(md, encoder,queryMapEncoder, target);
        } else {
            buildTemplate = new BuildTemplateByResolvingArgs(md, queryMapEncoder,target);
        }
        if (md.isIgnored()) {
            result.put(md.configKey(), args -> {
                throw new IllegalStateException(md.configKey() + " is not a method handled by feign");
            });
        } else {
            result.put(md.configKey(),
                       factory.create(target, md, buildTemplate, options, decoder,errorDecoder));
        }
    }
    return result;
}
```









**2.4.2 SpringMvcContract**
当前Spring Cloud 微服务解决方案中，为了降低学习成本，采用了Spring MVC的部分注解来完成 请求议解析，也就是说 ，写客户端请求接口和像写服务端代码一样：客户端和服务端可以通过SDK的方式进行约定，客户端只需要引入服务端发布的SDK API，就可以使用面向接口的编码方式对接服务。

该类继承了Contract.BaseContract并实现了ResourceLoaderAware接口，

其作用就是对RequestMapping、RequestParam、RequestHeader等注解进行解析的。

**2.5 OpenFeign调用过程**
在前面的分析中，我们知道OpenFeign最终返回的是一个#
ReflectiveFeign.FeignInvocationHandler的对象。

那么当客户端发起请求时，会进入到
FeignInvocationHandler.invoke方法中，这个大家都知道，它是一个动态代理的实现。



```
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (!"equals".equals(method.getName())) {
        if ("hashCode".equals(method.getName())) {
            return this.hashCode();
        } else {
            return "toString".equals(method.getName()) ? this.toString() :
            ((MethodHandler)this.dispatch.get(method)).invoke(args);
        }
    } else {
        try {
            Object otherHandler = args.length > 0 && args[0] != null ?
                Proxy.getInvocationHandler(args[0]) : null;
            return this.equals(otherHandler);
        } catch (IllegalArgumentException var5) {
            return false;
        }
    }
}
```









而接着，在invoke方法中，会调用 this.dispatch.get(method)).invoke(args) 。this.dispatch.get(method) 会返回一个SynchronousMethodHandler,进行拦截处理。

这个方法会根据参数生成完成的RequestTemplate对象，这个对象是Http请求的模版，代码如下。



```
public Object invoke(Object[] argv) throws Throwable {
    RequestTemplate template = this.buildTemplateFromArgs.create(argv);
    Options options = this.findOptions(argv);
    Retryer retryer = this.retryer.clone();
    while(true) {
        try {
            return this.executeAndDecode(template, options);
        } catch (RetryableException var9) {
            RetryableException e = var9;
            try {
                retryer.continueOrPropagate(e);
            } catch (RetryableException var8) {
                Throwable cause = var8.getCause();
                if (this.propagationPolicy == ExceptionPropagationPolicy.UNWRAP
                    && cause != null) {
                    throw cause;
                }
                throw var8;
            }
            if (this.logLevel != Level.NONE) {
                this.logger.logRetry(this.metadata.configKey(), this.logLevel);
            }
        }
    }
}
```









**2.5.1 executeAndDecode**
经过上述的代码，我们已经将restTemplate拼装完成，上面的代码中有一个 executeAndDecode() 方法，该方法通过RequestTemplate生成Request请求对象，然后利用Http Client获取response，来获取响应信息。



```
Object executeAndDecode(RequestTemplate template, Options options) throws Throwable {
    //转化为Http请求报文
    Request request = this.targetRequest(template);
    if (this.logLevel != Level.NONE) {
        this.logger.logRequest(this.metadata.configKey(), this.logLevel,request);
    }
    long start = System.nanoTime();
    Response response;
    try {
        //发起远程通信
        response = this.client.execute(request, options);
        //获取返回结果
        response = response.toBuilder().request(request).requestTemplate(template).build();
    } catch (IOException var16) {
        if (this.logLevel != Level.NONE) {
            this.logger.logIOException(this.metadata.configKey(), this.logLevel,var16, this.elapsedTime(start));
        }
        throw FeignException.errorExecuting(request, var16);
    }
    long elapsedTime = TimeUnit.NANOSECONDS.toMillis(System.nanoTime() - start);
    boolean shouldClose = true;
    Response var10;
    try {
        if (this.logLevel != Level.NONE) {
            response = this.logger.logAndRebufferResponse(this.metadata.configKey(), this.logLevel,response, elapsedTime);
        }
        if (Response.class != this.metadata.returnType()) {
            Object result;
            Object var21;
            if (response.status() >= 200 && response.status() < 300) {
                if (Void.TYPE == this.metadata.returnType()) {
                    var10 = null;
                    return var10;
                }
                result = this.decode(response);
                shouldClose = this.closeAfterDecode;
                var21 = result;
                return var21;
            }
            if (this.decode404 && response.status() == 404 && Void.TYPE != this.metadata.returnType()) {
                result = this.decode(response);
                shouldClose = this.closeAfterDecode;
                var21 = result;
                return var21;
            }
            throw this.errorDecoder.decode(this.metadata.configKey(), response);
        }
        if (response.body() == null) {
            var10 = response;
            return var10;
        }
        if (response.body().length() != null && (long)response.body().length() <= 8192L) {
            byte[] bodyData = Util.toByteArray(response.body().asInputStream());
            Response var11 = response.toBuilder().body(bodyData).build();
            return var11;
        }
        shouldClose = false;
        var10 = response;
    } catch (IOException var17) {
        if (this.logLevel != Level.NONE) {
            this.logger.logIOException(this.metadata.configKey(), this.logLevel,var17, elapsedTime);
        }
        throw FeignException.errorReading(request, response, var17);
    } finally {
        if (shouldClose) {
            Util.ensureClosed(response.body());
        }
    }
    return var10;
}
```









**2.5.2 Client.execute**
默认采用JDK的 HttpURLConnection 发起远程调用。



```
@Override
public Response execute(Request request, Options options) throws IOException {
    HttpURLConnection connection = convertAndSend(request, options);
    return convertResponse(connection, request);
}
Response convertResponse(HttpURLConnection connection, Request request) throws IOException {
    int status = connection.getResponseCode();
    String reason = connection.getResponseMessage();
    if (status < 0) {
        throw new IOException(format("Invalid status(%s) executing %s %s",status,connection.getRequestMethod(),connection.getURL()));
    }
    Map<String, Collection<String>> headers = new LinkedHashMap<>();
    for (Map.Entry<String, List<String>> field :
         connection.getHeaderFields().entrySet()) {
        // response message
        if (field.getKey() != null) {
            headers.put(field.getKey(), field.getValue());
        }
    }
    Integer length = connection.getContentLength();
    if (length == -1) {
        length = null;
    }
    InputStream stream;
    if (status >= 400) {
        stream = connection.getErrorStream();
    } else {
        stream = connection.getInputStream();
    }
    return Response.builder()
        .status(status)
        .reason(reason)
        .headers(headers)
        .request(request)
        .body(stream, length)
        .build();
}
```



# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning