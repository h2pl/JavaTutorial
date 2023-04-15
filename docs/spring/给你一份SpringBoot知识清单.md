# 目录
* [一、抛砖引玉：探索Spring IoC容器](#一、抛砖引玉：探索spring-ioc容器)
    * [1.1、Spring IoC容器](#11、spring-ioc容器)
    * [1.2、Spring容器扩展机制](#12、spring容器扩展机制)
* [二、夯实基础：JavaConfig与常见Annotation](#二、夯实基础：javaconfig与常见annotation)
    * [2.1、JavaConfig](#21、javaconfig)
    * [2.2、@ComponentScan](#22、componentscan)
    * [2.3、@Import](#23、import)
    * [2.4、@Conditional](#24、conditional)
    * [2.5、@ConfigurationProperties与@EnableConfigurationProperties](#25、configurationproperties与enableconfigurationproperties)
* [三、削铁如泥：SpringFactoriesLoader详解](#三、削铁如泥：springfactoriesloader详解)
* [四、另一件武器：Spring容器的事件监听机制](#四、另一件武器：spring容器的事件监听机制)
    * [Spring容器内的事件监听机制](#spring容器内的事件监听机制)
* [五、出神入化：揭秘自动配置原理](#五、出神入化：揭秘自动配置原理)
* [六、启动引导：Spring Boot应用启动的秘密](#六、启动引导：spring-boot应用启动的秘密)
    * [6.1 SpringApplication初始化](#61-springapplication初始化)
    * [6.2 Spring Boot启动流程](#62-spring-boot启动流程)
* [参考文章](#参考文章)


本文转自互联网，侵删

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章将同步到我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《Spring和SpringMVC源码分析》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从spring基础入手，一步步地学习spring基础和springmvc的框架知识，并上手进行项目实战，spring框架是每一个Java工程师必须要学习和理解的知识点，进一步来说，你还需要掌握spring甚至是springmvc的源码以及实现原理，才能更完整地了解整个spring技术体系，形成自己的知识框架。

后续还会有springboot和springcloud的技术专题，陆续为大家带来，敬请期待。

为了更好地总结和检验你的学习成果，本系列文章也会提供部分知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->  
<!-- more -->  


> 1.  预警：本文非常长，建议先mark后看，也许是最后一次写这么长的文章
> 2.  说明：前面有4个小节关于Spring的基础知识，分别是：IOC容器、JavaConfig、事件监听、SpringFactoriesLoader详解，它们占据了本文的大部分内容，虽然它们之间可能没有太多的联系，但这些知识对于理解Spring Boot的核心原理至关重要，如果你对Spring框架烂熟于心，完全可以跳过这4个小节。正是因为这个系列的文章是由这些看似不相关的知识点组成，因此取名知识清单。

转自：https://www.jianshu.com/p/83693d3d0a65

在过去两三年的Spring生态圈，最让人兴奋的莫过于Spring Boot框架。或许从命名上就能看出这个框架的设计初衷：快速的启动Spring应用。因而Spring Boot应用本质上就是一个基于Spring框架的应用，它是Spring对“约定优先于配置”理念的最佳实践产物，它能够帮助开发者更快速高效地构建基于Spring生态圈的应用。

那Spring Boot有何魔法？**自动配置**、**起步依赖**、**Actuator**、**命令行界面(CLI)**是Spring Boot最重要的4大核心特性，其中CLI是Spring Boot的可选特性，虽然它功能强大，但也引入了一套不太常规的开发模型，因而这个系列的文章仅关注其它3种特性。如文章标题，本文是这个系列的第一部分，将为你打开Spring Boot的大门，重点为你剖析其启动流程以及自动配置实现原理。要掌握这部分核心内容，理解一些Spring框架的基础知识，将会让你事半功倍。

## 一、抛砖引玉：探索Spring IoC容器

如果有看过`SpringApplication.run()`方法的源码，Spring Boot冗长无比的启动流程一定会让你抓狂，透过现象看本质，SpringApplication只是将一个典型的Spring应用的启动流程进行了扩展，因此，透彻理解Spring容器是打开Spring Boot大门的一把钥匙。

### 1.1、Spring IoC容器

可以把Spring IoC容器比作一间餐馆，当你来到餐馆，通常会直接招呼服务员：点菜！至于菜的原料是什么？如何用原料把菜做出来？可能你根本就不关心。IoC容器也是一样，你只需要告诉它需要某个bean，它就把对应的实例（instance）扔给你，至于这个bean是否依赖其他组件，怎样完成它的初始化，根本就不需要你关心。

作为餐馆，想要做出菜肴，得知道菜的原料和菜谱，同样地，IoC容器想要管理各个业务对象以及它们之间的依赖关系，需要通过某种途径来记录和管理这些信息。`BeanDefinition`对象就承担了这个责任：容器中的每一个bean都会有一个对应的BeanDefinition实例，该实例负责保存bean对象的所有必要信息，包括bean对象的class类型、是否是抽象类、构造方法和参数、其它属性等等。当客户端向容器请求相应对象时，容器就会通过这些信息为客户端返回一个完整可用的bean实例。

原材料已经准备好（把BeanDefinition看着原料），开始做菜吧，等等，你还需要一份菜谱，`BeanDefinitionRegistry`和`BeanFactory`就是这份菜谱，BeanDefinitionRegistry抽象出bean的注册逻辑，而BeanFactory则抽象出了bean的管理逻辑，而各个BeanFactory的实现类就具体承担了bean的注册以及管理工作。它们之间的关系就如下图：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/175724-911dc459ef9cec28.png)



BeanFactory、BeanDefinitionRegistry关系图（来自：Spring揭秘）



`DefaultListableBeanFactory`作为一个比较通用的BeanFactory实现，它同时也实现了BeanDefinitionRegistry接口，因此它就承担了Bean的注册管理工作。从图中也可以看出，BeanFactory接口中主要包含getBean、containBean、getType、getAliases等管理bean的方法，而BeanDefinitionRegistry接口则包含registerBeanDefinition、removeBeanDefinition、getBeanDefinition等注册管理BeanDefinition的方法。

下面通过一段简单的代码来模拟BeanFactory底层是如何工作的：

```  
// 默认容器实现  
DefaultListableBeanFactory beanRegistry = new DefaultListableBeanFactory();  
// 根据业务对象构造相应的BeanDefinition  
AbstractBeanDefinition definition = new RootBeanDefinition(Business.class,true);  
// 将bean定义注册到容器中  
beanRegistry.registerBeanDefinition("beanName",definition);  
// 如果有多个bean，还可以指定各个bean之间的依赖关系  
// ........  
  
// 然后可以从容器中获取这个bean的实例  
// 注意：这里的beanRegistry其实实现了BeanFactory接口，所以可以强转，  
// 单纯的BeanDefinitionRegistry是无法强制转换到BeanFactory类型的  
BeanFactory container = (BeanFactory)beanRegistry;  
Business business = (Business)container.getBean("beanName");  
  
```  

这段代码仅为了说明BeanFactory底层的大致工作流程，实际情况会更加复杂，比如bean之间的依赖关系可能定义在外部配置文件(XML/Properties)中、也可能是注解方式。Spring IoC容器的整个工作流程大致可以分为两个阶段：

①、容器启动阶段

容器启动时，会通过某种途径加载`Configuration MetaData`。除了代码方式比较直接外，在大部分情况下，容器需要依赖某些工具类，比如：`BeanDefinitionReader`，BeanDefinitionReader会对加载的`Configuration MetaData`进行解析和分析，并将分析后的信息组装为相应的BeanDefinition，最后把这些保存了bean定义的BeanDefinition，注册到相应的BeanDefinitionRegistry，这样容器的启动工作就完成了。这个阶段主要完成一些准备性工作，更侧重于bean对象管理信息的收集，当然一些验证性或者辅助性的工作也在这一阶段完成。

来看一个简单的例子吧，过往，所有的bean都定义在XML配置文件中，下面的代码将模拟BeanFactory如何从配置文件中加载bean的定义以及依赖关系：

```  
// 通常为BeanDefinitionRegistry的实现类，这里以DeFaultListabeBeanFactory为例  
BeanDefinitionRegistry beanRegistry = new DefaultListableBeanFactory(); // XmlBeanDefinitionReader实现了BeanDefinitionReader接口，用于解析XML文件  
XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReaderImpl(beanRegistry);  
// 加载配置文件  
beanDefinitionReader.loadBeanDefinitions("classpath:spring-bean.xml");  
  
// 从容器中获取bean实例  
BeanFactory container = (BeanFactory)beanRegistry;  
Business business = (Business)container.getBean("beanName");  
  
```  

②、Bean的实例化阶段

经过第一阶段，所有bean定义都通过BeanDefinition的方式注册到BeanDefinitionRegistry中，当某个请求通过容器的getBean方法请求某个对象，或者因为依赖关系容器需要隐式的调用getBean时，就会触发第二阶段的活动：容器会首先检查所请求的对象之前是否已经实例化完成。如果没有，则会根据注册的BeanDefinition所提供的信息实例化被请求对象，并为其注入依赖。当该对象装配完毕后，容器会立即将其返回给请求方法使用。

BeanFactory只是Spring IoC容器的一种实现，如果没有特殊指定，它采用采用延迟初始化策略：只有当访问容器中的某个对象时，才对该对象进行初始化和依赖注入操作。而在实际场景下，我们更多的使用另外一种类型的容器：`ApplicationContext`，它构建在BeanFactory之上，属于更高级的容器，除了具有BeanFactory的所有能力之外，还提供对事件监听机制以及国际化的支持等。它管理的bean，在容器启动时全部完成初始化和依赖注入操作。

### 1.2、Spring容器扩展机制

IoC容器负责管理容器中所有bean的生命周期，而在bean生命周期的不同阶段，Spring提供了不同的扩展点来改变bean的命运。在容器的启动阶段，`BeanFactoryPostProcessor`允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做一些额外的操作，比如修改bean定义的某些属性或者增加其他信息等。

如果要自定义扩展类，通常需要实现`org.springframework.beans.factory.config.BeanFactoryPostProcessor`接口，与此同时，因为容器中可能有多个BeanFactoryPostProcessor，可能还需要实现`org.springframework.core.Ordered`接口，以保证BeanFactoryPostProcessor按照顺序执行。Spring提供了为数不多的BeanFactoryPostProcessor实现，我们以`PropertyPlaceholderConfigurer`来说明其大致的工作流程。

在Spring项目的XML配置文件中，经常可以看到许多配置项的值使用占位符，而将占位符所代表的值单独配置到独立的properties文件，这样可以将散落在不同XML文件中的配置集中管理，而且也方便运维根据不同的环境进行配置不同的值。这个非常实用的功能就是由PropertyPlaceholderConfigurer负责实现的。

根据前文，当BeanFactory在第一阶段加载完所有配置信息时，BeanFactory中保存的对象的属性还是以占位符方式存在的，比如`${jdbc.mysql.url}`。当PropertyPlaceholderConfigurer作为BeanFactoryPostProcessor被应用时，它会使用properties配置文件中的值来替换相应的BeanDefinition中占位符所表示的属性值。当需要实例化bean时，bean定义中的属性值就已经被替换成我们配置的值。当然其实现比上面描述的要复杂一些，这里仅说明其大致工作原理，更详细的实现可以参考其源码。

与之相似的，还有`BeanPostProcessor`，其存在于对象实例化阶段。跟BeanFactoryPostProcessor类似，它会处理容器内所有符合条件并且已经实例化后的对象。简单的对比，BeanFactoryPostProcessor处理bean的定义，而BeanPostProcessor则处理bean完成实例化后的对象。BeanPostProcessor定义了两个接口：

```  
public interface BeanPostProcessor {  
    // 前置处理  
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;    // 后置处理  
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;}  
  
```  

为了理解这两个方法执行的时机，简单的了解下bean的整个生命周期：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/175724-0d50d67d980161b4.png)



Bean的实例化过程（来自：Spring揭秘）



`postProcessBeforeInitialization()`方法与`postProcessAfterInitialization()`分别对应图中前置处理和后置处理两个步骤将执行的方法。这两个方法中都传入了bean对象实例的引用，为扩展容器的对象实例化过程提供了很大便利，在这儿几乎可以对传入的实例执行任何操作。注解、AOP等功能的实现均大量使用了`BeanPostProcessor`，比如有一个自定义注解，你完全可以实现BeanPostProcessor的接口，在其中判断bean对象的脑袋上是否有该注解，如果有，你可以对这个bean实例执行任何操作，想想是不是非常的简单？

再来看一个更常见的例子，在Spring中经常能够看到各种各样的Aware接口，其作用就是在对象实例化完成以后将Aware接口定义中规定的依赖注入到当前实例中。比如最常见的`ApplicationContextAware`接口，实现了这个接口的类都可以获取到一个ApplicationContext对象。当容器中每个对象的实例化过程走到BeanPostProcessor前置处理这一步时，容器会检测到之前注册到容器的ApplicationContextAwareProcessor，然后就会调用其postProcessBeforeInitialization()方法，检查并设置Aware相关依赖。看看代码吧，是不是很简单：

```  
// 代码来自：org.springframework.context.support.ApplicationContextAwareProcessor  
// 其postProcessBeforeInitialization方法调用了invokeAwareInterfaces方法  
private void invokeAwareInterfaces(Object bean) {  
    if (bean instanceof EnvironmentAware) {        ((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());    }    if (bean instanceof ApplicationContextAware) {        ((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);    }    // ......}  
  
```  

最后总结一下，本小节内容和你一起回顾了Spring容器的部分核心内容，限于篇幅不能写更多，但理解这部分内容，足以让您轻松理解Spring Boot的启动原理，如果在后续的学习过程中遇到一些晦涩难懂的知识，再回过头来看看Spring的核心知识，也许有意想不到的效果。也许Spring Boot的中文资料很少，但Spring的中文资料和书籍有太多太多，总有东西能给你启发。

## 二、夯实基础：JavaConfig与常见Annotation

### 2.1、JavaConfig

我们知道`bean`是Spring IOC中非常核心的概念，Spring容器负责bean的生命周期的管理。在最初，Spring使用XML配置文件的方式来描述bean的定义以及相互间的依赖关系，但随着Spring的发展，越来越多的人对这种方式表示不满，因为Spring项目的所有业务类均以bean的形式配置在XML文件中，造成了大量的XML文件，使项目变得复杂且难以管理。

后来，基于纯Java Annotation依赖注入框架`Guice`出世，其性能明显优于采用XML方式的Spring，甚至有部分人认为，`Guice`可以完全取代Spring（`Guice`仅是一个轻量级IOC框架，取代Spring还差的挺远）。正是这样的危机感，促使Spring及社区推出并持续完善了`JavaConfig`子项目，它基于Java代码和Annotation注解来描述bean之间的依赖绑定关系。比如，下面是使用XML配置方式来描述bean的定义：

```  
<bean id="bookService" class="cn.moondev.service.BookServiceImpl"></bean>  
  
```  

而基于JavaConfig的配置形式是这样的：

```  
@Configuration  
public class MoonBookConfiguration {  
  
    // 任何标志了@Bean的方法，其返回值将作为一个bean注册到Spring的IOC容器中  
    // 方法名默认成为该bean定义的id  
    @Bean    public BookService bookService() {        return new BookServiceImpl();    }}  
  
```  

如果两个bean之间有依赖关系的话，在XML配置中应该是这样：

```  
<bean id="bookService" class="cn.moondev.service.BookServiceImpl">  
    <property name="dependencyService" ref="dependencyService"/></bean>  
  
<bean id="otherService" class="cn.moondev.service.OtherServiceImpl">  
    <property name="dependencyService" ref="dependencyService"/></bean>  
  
<bean id="dependencyService" class="DependencyServiceImpl"/>  
  
```  

而在JavaConfig中则是这样：

```  
@Configuration  
public class MoonBookConfiguration {  
  
    // 如果一个bean依赖另一个bean，则直接调用对应JavaConfig类中依赖bean的创建方法即可  
    // 这里直接调用dependencyService()  
    @Bean    public BookService bookService() {        return new BookServiceImpl(dependencyService());    }  
    @Bean    public OtherService otherService() {        return new OtherServiceImpl(dependencyService());    }  
    @Bean    public DependencyService dependencyService() {        return new DependencyServiceImpl();    }}  
  
```  

你可能注意到这个示例中，有两个bean都依赖于dependencyService，也就是说当初始化bookService时会调用`dependencyService()`，在初始化otherService时也会调用`dependencyService()`，那么问题来了？这时候IOC容器中是有一个dependencyService实例还是两个？这个问题留着大家思考吧，这里不再赘述。

### 2.2、@ComponentScan

`@ComponentScan`注解对应XML配置形式中的`<context:component-scan>`元素，表示启用组件扫描，Spring会自动扫描所有通过注解配置的bean，然后将其注册到IOC容器中。我们可以通过`basePackages`等属性来指定`@ComponentScan`自动扫描的范围，如果不指定，默认从声明`@ComponentScan`所在类的`package`进行扫描。正因为如此，SpringBoot的启动类都默认在`src/main/java`下。

### 2.3、@Import

`@Import`注解用于导入配置类，举个简单的例子：

```  
@Configuration  
public class MoonBookConfiguration {  
    @Bean    public BookService bookService() {        return new BookServiceImpl();    }}  
  
```  

现在有另外一个配置类，比如：`MoonUserConfiguration`，这个配置类中有一个bean依赖于`MoonBookConfiguration`中的bookService，如何将这两个bean组合在一起？借助`@Import`即可：

```  
@Configuration  
// 可以同时导入多个配置类，比如：@Import({A.class,B.class})  
@Import(MoonBookConfiguration.class)  
public class MoonUserConfiguration {  
    @Bean    public UserService userService(BookService bookService) {        return new BookServiceImpl(bookService);    }}  
  
```  

需要注意的是，在4.2之前，`@Import`注解只支持导入配置类，但是在4.2之后，它支持导入普通类，并将这个类作为一个bean的定义注册到IOC容器中。

### 2.4、@Conditional

`@Conditional`注解表示在满足某种条件后才初始化一个bean或者启用某些配置。它一般用在由`@Component`、`@Service`、`@Configuration`等注解标识的类上面，或者由`@Bean`标记的方法上。如果一个`@Configuration`类标记了`@Conditional`，则该类中所有标识了`@Bean`的方法和`@Import`注解导入的相关类将遵从这些条件。

在Spring里可以很方便的编写你自己的条件类，所要做的就是实现`Condition`接口，并覆盖它的`matches()`方法。举个例子，下面的简单条件类表示只有在`Classpath`里存在`JdbcTemplate`类时才生效：

```  
public class JdbcTemplateCondition implements Condition {  
  
    @Override    public boolean matches(ConditionContext conditionContext, AnnotatedTypeMetadata annotatedTypeMetadata) {        try {        conditionContext.getClassLoader().loadClass("org.springframework.jdbc.core.JdbcTemplate");            return true;        } catch (ClassNotFoundException e) {            e.printStackTrace();        }        return false;    }}  
  
```  

当你用Java来声明bean的时候，可以使用这个自定义条件类：

```  
@Conditional(JdbcTemplateCondition.class)  
@Service  
public MyService service() {  
    ......}  
  
```  

这个例子中只有当`JdbcTemplateCondition`类的条件成立时才会创建MyService这个bean。也就是说MyService这bean的创建条件是`classpath`里面包含`JdbcTemplate`，否则这个bean的声明就会被忽略掉。

`Spring Boot`定义了很多有趣的条件，并把他们运用到了配置类上，这些配置类构成了`Spring Boot`的自动配置的基础。`Spring Boot`运用条件化配置的方法是：定义多个特殊的条件化注解，并将它们用到配置类上。下面列出了`Spring Boot`提供的部分条件化注解：

| 条件化注解 | 配置生效条件 |  
| --- | --- |  
| @ConditionalOnBean | 配置了某个特定bean |  
| @ConditionalOnMissingBean | 没有配置特定的bean |  
| @ConditionalOnClass | Classpath里有指定的类 |  
| @ConditionalOnMissingClass | Classpath里没有指定的类 |  
| @ConditionalOnExpression | 给定的Spring Expression Language表达式计算结果为true |  
| @ConditionalOnJava | Java的版本匹配特定指或者一个范围值 |  
| @ConditionalOnProperty | 指定的配置属性要有一个明确的值 |  
| @ConditionalOnResource | Classpath里有指定的资源 |  
| @ConditionalOnWebApplication | 这是一个Web应用程序 |  
| @ConditionalOnNotWebApplication | 这不是一个Web应用程序 |  

### 2.5、@ConfigurationProperties与@EnableConfigurationProperties

当某些属性的值需要配置的时候，我们一般会在`application.properties`文件中新建配置项，然后在bean中使用`@Value`注解来获取配置的值，比如下面配置数据源的代码。

```  
// jdbc config  
jdbc.mysql.url=jdbc:mysql://localhost:3306/sampledb  
jdbc.mysql.username=root  
jdbc.mysql.password=123456  
......  
  
// 配置数据源  
@Configuration  
public class HikariDataSourceConfiguration {  
  
    @Value("jdbc.mysql.url")    public String url;    @Value("jdbc.mysql.username")    public String user;    @Value("jdbc.mysql.password")    public String password;  
    @Bean    public HikariDataSource dataSource() {        HikariConfig hikariConfig = new HikariConfig();        hikariConfig.setJdbcUrl(url);        hikariConfig.setUsername(user);        hikariConfig.setPassword(password);        // 省略部分代码  
        return new HikariDataSource(hikariConfig);    }}  
  
```  

使用`@Value`注解注入的属性通常都比较简单，如果同一个配置在多个地方使用，也存在不方便维护的问题（考虑下，如果有几十个地方在使用某个配置，而现在你想改下名字，你改怎么做？）。对于更为复杂的配置，Spring Boot提供了更优雅的实现方式，那就是`@ConfigurationProperties`注解。我们可以通过下面的方式来改写上面的代码：

```  
@Component  
//  还可以通过@PropertySource("classpath:jdbc.properties")来指定配置文件  
@ConfigurationProperties("jdbc.mysql")  
// 前缀=jdbc.mysql，会在配置文件中寻找jdbc.mysql.*的配置项  
pulic class JdbcConfig {  
    public String url;    public String username;    public String password;}  
  
@Configuration  
public class HikariDataSourceConfiguration {  
  
    @AutoWired    public JdbcConfig config;  
    @Bean    public HikariDataSource dataSource() {        HikariConfig hikariConfig = new HikariConfig();        hikariConfig.setJdbcUrl(config.url);        hikariConfig.setUsername(config.username);        hikariConfig.setPassword(config.password);        // 省略部分代码  
        return new HikariDataSource(hikariConfig);    }}  
  
```  

`@ConfigurationProperties`对于更为复杂的配置，处理起来也是得心应手，比如有如下配置文件：

```  
#App  
app.menus[0].title=Home  
app.menus[0].name=Home  
app.menus[0].path=/  
app.menus[1].title=Login  
app.menus[1].name=Login  
app.menus[1].path=/login  
  
app.compiler.timeout=5  
app.compiler.output-folder=/temp/  
  
app.error=/error/  
  
```  

可以定义如下配置类来接收这些属性

```  
@Component  
@ConfigurationProperties("app")  
public class AppProperties {  
  
    public String error;    public List<Menu> menus = new ArrayList<>();    public Compiler compiler = new Compiler();  
    public static class Menu {        public String name;        public String path;        public String title;    }  
    public static class Compiler {        public String timeout;        public String outputFolder;    }}  
  
```  

`@EnableConfigurationProperties`注解表示对`@ConfigurationProperties`的内嵌支持，默认会将对应Properties Class作为bean注入的IOC容器中，即在相应的Properties类上不用加`@Component`注解。

## 三、削铁如泥：SpringFactoriesLoader详解

JVM提供了3种类加载器：`BootstrapClassLoader`、`ExtClassLoader`、`AppClassLoader`分别加载Java核心类库、扩展类库以及应用的类路径(`CLASSPATH`)下的类库。JVM通过双亲委派模型进行类的加载，我们也可以通过继承`java.lang.classloader`实现自己的类加载器。

何为双亲委派模型？当一个类加载器收到类加载任务时，会先交给自己的父加载器去完成，因此最终加载任务都会传递到最顶层的BootstrapClassLoader，只有当父加载器无法完成加载任务时，才会尝试自己来加载。

采用双亲委派模型的一个好处是保证使用不同类加载器最终得到的都是同一个对象，这样就可以保证Java 核心库的类型安全，比如，加载位于rt.jar包中的`java.lang.Object`类，不管是哪个加载器加载这个类，最终都是委托给顶层的BootstrapClassLoader来加载的，这样就可以保证任何的类加载器最终得到的都是同样一个Object对象。查看ClassLoader的源码，对双亲委派模型会有更直观的认识：

```  
protected Class<?> loadClass(String name, boolean resolve) {  
    synchronized (getClassLoadingLock(name)) {    // 首先，检查该类是否已经被加载，如果从JVM缓存中找到该类，则直接返回  
    Class<?> c = findLoadedClass(name);    if (c == null) {        try {            // 遵循双亲委派的模型，首先会通过递归从父加载器开始找，  
            // 直到父类加载器是BootstrapClassLoader为止  
            if (parent != null) {                c = parent.loadClass(name, false);            } else {                c = findBootstrapClassOrNull(name);            }        } catch (ClassNotFoundException e) {}        if (c == null) {            // 如果还找不到，尝试通过findClass方法去寻找  
            // findClass是留给开发者自己实现的，也就是说  
            // 自定义类加载器时，重写此方法即可  
           c = findClass(name);        }    }    if (resolve) {        resolveClass(c);    }    return c;    }}  
  
```  

但双亲委派模型并不能解决所有的类加载器问题，比如，Java 提供了很多服务提供者接口(`Service Provider Interface`，SPI)，允许第三方为这些接口提供实现。常见的 SPI 有 JDBC、JNDI、JAXP 等，这些SPI的接口由核心类库提供，却由第三方实现，这样就存在一个问题：SPI 的接口是 Java 核心库的一部分，是由BootstrapClassLoader加载的；SPI实现的Java类一般是由AppClassLoader来加载的。BootstrapClassLoader是无法找到 SPI 的实现类的，因为它只加载Java的核心库。它也不能代理给AppClassLoader，因为它是最顶层的类加载器。也就是说，双亲委派模型并不能解决这个问题。

线程上下文类加载器(`ContextClassLoader`)正好解决了这个问题。从名称上看，可能会误解为它是一种新的类加载器，实际上，它仅仅是Thread类的一个变量而已，可以通过`setContextClassLoader(ClassLoader cl)`和`getContextClassLoader()`来设置和获取该对象。如果不做任何的设置，Java应用的线程的上下文类加载器默认就是AppClassLoader。在核心类库使用SPI接口时，传递的类加载器使用线程上下文类加载器，就可以成功的加载到SPI实现的类。线程上下文类加载器在很多SPI的实现中都会用到。但在JDBC中，你可能会看到一种更直接的实现方式，比如，JDBC驱动管理`java.sql.Driver`中的`loadInitialDrivers()`方法中，你可以直接看到JDK是如何加载驱动的：

```  
for (String aDriver : driversList) {  
    try {        // 直接使用AppClassLoader  
        Class.forName(aDriver, true, ClassLoader.getSystemClassLoader());    } catch (Exception ex) {        println("DriverManager.Initialize: load failed: " + ex);    }}  
  
```  

其实讲解线程上下文类加载器，最主要是让大家在看到`Thread.currentThread().getClassLoader()`和`Thread.currentThread().getContextClassLoader()`时不会一脸懵逼，这两者除了在许多底层框架中取得的ClassLoader可能会有所不同外，其他大多数业务场景下都是一样的，大家只要知道它是为了解决什么问题而存在的即可。

类加载器除了加载class外，还有一个非常重要功能，就是加载资源，它可以从jar包中读取任何资源文件，比如，`ClassLoader.getResources(String name)`方法就是用于读取jar包中的资源文件，其代码如下：

```  
public Enumeration<URL> getResources(String name) throws IOException {  
    Enumeration<URL>[] tmp = (Enumeration<URL>[]) new Enumeration<?>[2];    if (parent != null) {        tmp[0] = parent.getResources(name);    } else {        tmp[0] = getBootstrapResources(name);    }    tmp[1] = findResources(name);    return new CompoundEnumeration<>(tmp);}  
  
```  

是不是觉得有点眼熟，不错，它的逻辑其实跟类加载的逻辑是一样的，首先判断父类加载器是否为空，不为空则委托父类加载器执行资源查找任务，直到BootstrapClassLoader，最后才轮到自己查找。而不同的类加载器负责扫描不同路径下的jar包，就如同加载class一样，最后会扫描所有的jar包，找到符合条件的资源文件。

类加载器的`findResources(name)`方法会遍历其负责加载的所有jar包，找到jar包中名称为name的资源文件，这里的资源可以是任何文件，甚至是.class文件，比如下面的示例，用于查找Array.class文件：

```  
// 寻找Array.class文件  
public static void main(String[] args) throws Exception{  
    // Array.class的完整路径  
    String name = "java/sql/Array.class";    Enumeration<URL> urls = Thread.currentThread().getContextClassLoader().getResources(name);    while (urls.hasMoreElements()) {        URL url = urls.nextElement();        System.out.println(url.toString());    }}  
  
```  

运行后可以得到如下结果：

```  
$JAVA_HOME/jre/lib/rt.jar!/java/sql/Array.class  
  
```  

根据资源文件的URL，可以构造相应的文件来读取资源内容。

看到这里，你可能会感到挺奇怪的，你不是要详解`SpringFactoriesLoader`吗？上来讲了一堆ClassLoader是几个意思？看下它的源码你就知道了：

```  
public static final String FACTORIES_RESOURCE_LOCATION = "META-INF/spring.factories";  
// spring.factories文件的格式为：key=value1,value2,value3  
// 从所有的jar包中找到META-INF/spring.factories文件  
// 然后从文件中解析出key=factoryClass类名称的所有value值  
public static List<String> loadFactoryNames(Class<?> factoryClass, ClassLoader classLoader) {  
    String factoryClassName = factoryClass.getName();    // 取得资源文件的URL  
    Enumeration<URL> urls = (classLoader != null ? classLoader.getResources(FACTORIES_RESOURCE_LOCATION) : ClassLoader.getSystemResources(FACTORIES_RESOURCE_LOCATION));    List<String> result = new ArrayList<String>();    // 遍历所有的URL  
    while (urls.hasMoreElements()) {        URL url = urls.nextElement();        // 根据资源文件URL解析properties文件  
        Properties properties = PropertiesLoaderUtils.loadProperties(new UrlResource(url));        String factoryClassNames = properties.getProperty(factoryClassName);        // 组装数据，并返回  
        result.addAll(Arrays.asList(StringUtils.commaDelimitedListToStringArray(factoryClassNames)));    }    return result;}  
  
```  

有了前面关于ClassLoader的知识，再来理解这段代码，是不是感觉豁然开朗：从`CLASSPATH`下的每个Jar包中搜寻所有`META-INF/spring.factories`配置文件，然后将解析properties文件，找到指定名称的配置后返回。需要注意的是，其实这里不仅仅是会去ClassPath路径下查找，会扫描所有路径下的Jar包，只不过这个文件只会在Classpath下的jar包中。来简单看下`spring.factories`文件的内容吧：

```  
// 来自 org.springframework.boot.autoconfigure下的META-INF/spring.factories  
// EnableAutoConfiguration后文会讲到，它用于开启Spring Boot自动配置功能  
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\  
org.springframework.boot.autoconfigure.admin.SpringApplicationAdminJmxAutoConfiguration,\  
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\  
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration\  
  
```  

执行`loadFactoryNames(EnableAutoConfiguration.class, classLoader)`后，得到对应的一组`@Configuration`类，  
我们就可以通过反射实例化这些类然后注入到IOC容器中，最后容器里就有了一系列标注了`@Configuration`的JavaConfig形式的配置类。

这就是`SpringFactoriesLoader`，它本质上属于Spring框架私有的一种扩展方案，类似于SPI，Spring Boot在Spring基础上的很多核心功能都是基于此，希望大家可以理解。

## 四、另一件武器：Spring容器的事件监听机制

过去，事件监听机制多用于图形界面编程，比如：**点击**按钮、在文本框**输入**内容等操作被称为事件，而当事件触发时，应用程序作出一定的响应则表示应用监听了这个事件，而在服务器端，事件的监听机制更多的用于异步通知以及监控和异常处理。Java提供了实现事件监听机制的两个基础类：自定义事件类型扩展自`java.util.EventObject`、事件的监听器扩展自`java.util.EventListener`。来看一个简单的实例：简单的监控一个方法的耗时。

首先定义事件类型，通常的做法是扩展EventObject，随着事件的发生，相应的状态通常都封装在此类中：

```  
public class MethodMonitorEvent extends EventObject {  
    // 时间戳，用于记录方法开始执行的时间  
    public long timestamp;  
    public MethodMonitorEvent(Object source) {        super(source);    }}  
  
```  

事件发布之后，相应的监听器即可对该类型的事件进行处理，我们可以在方法开始执行之前发布一个begin事件，在方法执行结束之后发布一个end事件，相应地，事件监听器需要提供方法对这两种情况下接收到的事件进行处理：

```  
// 1、定义事件监听接口  
public interface MethodMonitorEventListener extends EventListener {  
    // 处理方法执行之前发布的事件  
    public void onMethodBegin(MethodMonitorEvent event);    // 处理方法结束时发布的事件  
    public void onMethodEnd(MethodMonitorEvent event);}  
// 2、事件监听接口的实现：如何处理  
public class AbstractMethodMonitorEventListener implements MethodMonitorEventListener {  
  
    @Override    public void onMethodBegin(MethodMonitorEvent event) {        // 记录方法开始执行时的时间  
        event.timestamp = System.currentTimeMillis();    }  
    @Override    public void onMethodEnd(MethodMonitorEvent event) {        // 计算方法耗时  
        long duration = System.currentTimeMillis() - event.timestamp;        System.out.println("耗时：" + duration);  
    }}  
  
```  

事件监听器接口针对不同的事件发布实际提供相应的处理方法定义，最重要的是，其方法只接收MethodMonitorEvent参数，说明这个监听器类只负责监听器对应的事件并进行处理。有了事件和监听器，剩下的就是发布事件，然后让相应的监听器监听并处理。通常情况，我们会有一个事件发布者，它本身作为事件源，在合适的时机，将相应的事件发布给对应的事件监听器：

```  
public class MethodMonitorEventPublisher {  
  
    private List<MethodMonitorEventListener> listeners = new ArrayList<MethodMonitorEventListener>();  
    public void methodMonitor() {        MethodMonitorEvent eventObject = new MethodMonitorEvent(this);        publishEvent("begin",eventObject);        // 模拟方法执行：休眠5秒钟  
        TimeUnit.SECONDS.sleep(5);        publishEvent("end",eventObject);  
    }  
    private void publishEvent(String status,MethodMonitorEvent event) {        // 避免在事件处理期间，监听器被移除，这里为了安全做一个复制操作  
        List<MethodMonitorEventListener> copyListeners = ➥ new ArrayList<MethodMonitorEventListener>(listeners);        for (MethodMonitorEventListener listener : copyListeners) {            if ("begin".equals(status)) {                listener.onMethodBegin(event);            } else {                listener.onMethodEnd(event);            }        }    }  
    public static void main(String[] args) {        MethodMonitorEventPublisher publisher = new MethodMonitorEventPublisher();        publisher.addEventListener(new AbstractMethodMonitorEventListener());        publisher.methodMonitor();    }    // 省略实现  
    public void addEventListener(MethodMonitorEventListener listener) {}    public void removeEventListener(MethodMonitorEventListener listener) {}    public void removeAllListeners() {}  
```  

对于事件发布者（事件源）通常需要关注两点：

1.  在合适的时机发布事件。此例中的methodMonitor()方法是事件发布的源头，其在方法执行之前和结束之后两个时间点发布MethodMonitorEvent事件，每个时间点发布的事件都会传给相应的监听器进行处理。在具体实现时需要注意的是，事件发布是顺序执行，为了不影响处理性能，事件监听器的处理逻辑应尽量简单。
2.  事件监听器的管理。publisher类中提供了事件监听器的注册与移除方法，这样客户端可以根据实际情况决定是否需要注册新的监听器或者移除某个监听器。如果这里没有提供remove方法，那么注册的监听器示例将一直被MethodMonitorEventPublisher引用，即使已经废弃不用了，也依然在发布者的监听器列表中，这会导致隐性的内存泄漏。

#### Spring容器内的事件监听机制

Spring的ApplicationContext容器内部中的所有事件类型均继承自`org.springframework.context.AppliationEvent`，容器中的所有监听器都实现`org.springframework.context.ApplicationListener`接口，并且以bean的形式注册在容器中。一旦在容器内发布ApplicationEvent及其子类型的事件，注册到容器的ApplicationListener就会对这些事件进行处理。

你应该已经猜到是怎么回事了。

ApplicationEvent继承自EventObject，Spring提供了一些默认的实现，比如：`ContextClosedEvent`表示容器在即将关闭时发布的事件类型，`ContextRefreshedEvent`表示容器在初始化或者刷新的时候发布的事件类型......

容器内部使用ApplicationListener作为事件监听器接口定义，它继承自EventListener。ApplicationContext容器在启动时，会自动识别并加载EventListener类型的bean，一旦容器内有事件发布，将通知这些注册到容器的EventListener。

ApplicationContext接口继承了ApplicationEventPublisher接口，该接口提供了`void publishEvent(ApplicationEvent event)`方法定义，不难看出，ApplicationContext容器担当的就是事件发布者的角色。如果有兴趣可以查看`AbstractApplicationContext.publishEvent(ApplicationEvent event)`方法的源码：ApplicationContext将事件的发布以及监听器的管理工作委托给`ApplicationEventMulticaster`接口的实现类。在容器启动时，会检查容器内是否存在名为applicationEventMulticaster的ApplicationEventMulticaster对象实例。如果有就使用其提供的实现，没有就默认初始化一个SimpleApplicationEventMulticaster作为实现。

最后，如果我们业务需要在容器内部发布事件，只需要为其注入ApplicationEventPublisher依赖即可：实现ApplicationEventPublisherAware接口或者ApplicationContextAware接口(Aware接口相关内容请回顾上文)。

## 五、出神入化：揭秘自动配置原理

典型的Spring Boot应用的启动类一般均位于`src/main/java`根路径下，比如`MoonApplication`类：

```  
@SpringBootApplication  
public class MoonApplication {  
  
    public static void main(String[] args) {        SpringApplication.run(MoonApplication.class, args);    }}  
  
```  

其中`@SpringBootApplication`开启组件扫描和自动配置，而`SpringApplication.run`则负责启动引导应用程序。`@SpringBootApplication`是一个复合`Annotation`，它将三个有用的注解组合在一起：

```  
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@SpringBootConfiguration  
@EnableAutoConfiguration  
@ComponentScan(excludeFilters = {  
        @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),        @Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })public @interface SpringBootApplication {  
    // ......}  
  
```  

`@SpringBootConfiguration`就是`@Configuration`，它是Spring框架的注解，标明该类是一个`JavaConfig`配置类。而`@ComponentScan`启用组件扫描，前文已经详细讲解过，这里着重关注`@EnableAutoConfiguration`。

`@EnableAutoConfiguration`注解表示开启Spring Boot自动配置功能，Spring Boot会根据应用的依赖、自定义的bean、classpath下有没有某个类 等等因素来猜测你需要的bean，然后注册到IOC容器中。那`@EnableAutoConfiguration`是如何推算出你的需求？首先看下它的定义：

```  
@Target(ElementType.TYPE)  
@Retention(RetentionPolicy.RUNTIME)  
@Documented  
@Inherited  
@AutoConfigurationPackage  
@Import(EnableAutoConfigurationImportSelector.class)  
public @interface EnableAutoConfiguration {  
    // ......}  
  
```  

你的关注点应该在`@Import(EnableAutoConfigurationImportSelector.class)`上了，前文说过，`@Import`注解用于导入类，并将这个类作为一个bean的定义注册到容器中，这里它将把`EnableAutoConfigurationImportSelector`作为bean注入到容器中，而这个类会将所有符合条件的@Configuration配置都加载到容器中，看看它的代码：

```  
public String[] selectImports(AnnotationMetadata annotationMetadata) {  
    // 省略了大部分代码，保留一句核心代码  
    // 注意：SpringBoot最近版本中，这句代码被封装在一个单独的方法中  
    // SpringFactoriesLoader相关知识请参考前文  
    List<String> factories = new ArrayList<String>(new LinkedHashSet<String>(        SpringFactoriesLoader.loadFactoryNames(EnableAutoConfiguration.class, this.beanClassLoader)));  
}  
  
```  

这个类会扫描所有的jar包，将所有符合条件的@Configuration配置类注入的容器中，何为符合条件，看看`META-INF/spring.factories`的文件内容：

```  
// 来自 org.springframework.boot.autoconfigure下的META-INF/spring.factories  
// 配置的key = EnableAutoConfiguration，与代码中一致  
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\  
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\  
org.springframework.boot.autoconfigure.aop.AopAutoConfiguration,\  
org.springframework.boot.autoconfigure.amqp.RabbitAutoConfiguration\  
.....  
  
```  

以`DataSourceAutoConfiguration`为例，看看Spring Boot是如何自动配置的：

```  
@Configuration  
@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })  
@EnableConfigurationProperties(DataSourceProperties.class)  
@Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })  
public class DataSourceAutoConfiguration {  
}  
  
```  

分别说一说：

*   `@ConditionalOnClass({ DataSource.class, EmbeddedDatabaseType.class })`：当Classpath中存在DataSource或者EmbeddedDatabaseType类时才启用这个配置，否则这个配置将被忽略。
*   `@EnableConfigurationProperties(DataSourceProperties.class)`：将DataSource的默认配置类注入到IOC容器中，DataSourceproperties定义为：

```  
// 提供对datasource配置信息的支持，所有的配置前缀为：spring.datasource  
@ConfigurationProperties(prefix = "spring.datasource")  
public class DataSourceProperties  {  
    private ClassLoader classLoader;    private Environment environment;    private String name = "testdb";    ......}  
  
```  

*   `@Import({ Registrar.class, DataSourcePoolMetadataProvidersConfiguration.class })`：导入其他额外的配置，就以`DataSourcePoolMetadataProvidersConfiguration`为例吧。

```  
@Configuration  
public class DataSourcePoolMetadataProvidersConfiguration {  
  
    @Configuration    @ConditionalOnClass(org.apache.tomcat.jdbc.pool.DataSource.class)    static class TomcatDataSourcePoolMetadataProviderConfiguration {        @Bean        public DataSourcePoolMetadataProvider tomcatPoolDataSourceMetadataProvider() {            .....        }    }  ......}  
  
```  

DataSourcePoolMetadataProvidersConfiguration是数据库连接池提供者的一个配置类，即Classpath中存在`org.apache.tomcat.jdbc.pool.DataSource.class`，则使用tomcat-jdbc连接池，如果Classpath中存在`HikariDataSource.class`则使用Hikari连接池。

这里仅描述了DataSourceAutoConfiguration的冰山一角，但足以说明Spring Boot如何利用条件话配置来实现自动配置的。回顾一下，`@EnableAutoConfiguration`中导入了EnableAutoConfigurationImportSelector类，而这个类的`selectImports()`通过SpringFactoriesLoader得到了大量的配置类，而每一个配置类则根据条件化配置来做出决策，以实现自动配置。

整个流程很清晰，但漏了一个大问题：`EnableAutoConfigurationImportSelector.selectImports()`是何时执行的？其实这个方法会在容器启动过程中执行：`AbstractApplicationContext.refresh()`，更多的细节在下一小节中说明。

## 六、启动引导：Spring Boot应用启动的秘密

### 6.1 SpringApplication初始化

SpringBoot整个启动流程分为两个步骤：初始化一个SpringApplication对象、执行该对象的run方法。看下SpringApplication的初始化流程，SpringApplication的构造方法中调用initialize(Object[] sources)方法，其代码如下：

```  
private void initialize(Object[] sources) {  
     if (sources != null && sources.length > 0) {         this.sources.addAll(Arrays.asList(sources));     }     // 判断是否是Web项目  
     this.webEnvironment = deduceWebEnvironment();     setInitializers((Collection) getSpringFactoriesInstances(ApplicationContextInitializer.class));     setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));     // 找到入口类  
     this.mainApplicationClass = deduceMainApplicationClass();}  
  
```  

初始化流程中最重要的就是通过SpringFactoriesLoader找到`spring.factories`文件中配置的`ApplicationContextInitializer`和`ApplicationListener`两个接口的实现类名称，以便后期构造相应的实例。`ApplicationContextInitializer`的主要目的是在`ConfigurableApplicationContext`做refresh之前，对ConfigurableApplicationContext实例做进一步的设置或处理。ConfigurableApplicationContext继承自ApplicationContext，其主要提供了对ApplicationContext进行设置的能力。

实现一个ApplicationContextInitializer非常简单，因为它只有一个方法，但大多数情况下我们没有必要自定义一个ApplicationContextInitializer，即便是Spring Boot框架，它默认也只是注册了两个实现，毕竟Spring的容器已经非常成熟和稳定，你没有必要来改变它。

而`ApplicationListener`的目的就没什么好说的了，它是Spring框架对Java事件监听机制的一种框架实现，具体内容在前文Spring事件监听机制这个小节有详细讲解。这里主要说说，如果你想为Spring Boot应用添加监听器，该如何实现？

Spring Boot提供两种方式来添加自定义监听器：

*   通过`SpringApplication.addListeners(ApplicationListener<?>... listeners)`或者`SpringApplication.setListeners(Collection<? extends ApplicationListener<?>> listeners)`两个方法来添加一个或者多个自定义监听器
*   既然SpringApplication的初始化流程中已经从`spring.factories`中获取到`ApplicationListener`的实现类，那么我们直接在自己的jar包的`META-INF/spring.factories`文件中新增配置即可：

```  
org.springframework.context.ApplicationListener=\  
cn.moondev.listeners.xxxxListener\  
  
```  

关于SpringApplication的初始化，我们就说这么多。

### 6.2 Spring Boot启动流程

Spring Boot应用的整个启动流程都封装在SpringApplication.run方法中，其整个流程真的是太长太长了，但本质上就是在Spring容器启动的基础上做了大量的扩展，按照这个思路来看看源码：

```  
public ConfigurableApplicationContext run(String... args) {  
        StopWatch stopWatch = new StopWatch();        stopWatch.start();        ConfigurableApplicationContext context = null;        FailureAnalyzers analyzers = null;        configureHeadlessProperty();        // ①  
        SpringApplicationRunListeners listeners = getRunListeners(args);        listeners.starting();        try {            // ②  
            ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);            ConfigurableEnvironment environment = prepareEnvironment(listeners,applicationArguments);            // ③  
            Banner printedBanner = printBanner(environment);            // ④  
            context = createApplicationContext();            // ⑤  
            analyzers = new FailureAnalyzers(context);            // ⑥  
            prepareContext(context, environment, listeners, applicationArguments,printedBanner);            // ⑦   
            refreshContext(context);  
            // ⑧  
            afterRefresh(context, applicationArguments);            // ⑨  
            listeners.finished(context, null);            stopWatch.stop();            return context;        }        catch (Throwable ex) {            handleRunFailure(context, listeners, analyzers, ex);            throw new IllegalStateException(ex);        }    }  
```  

① 通过SpringFactoriesLoader查找并加载所有的`SpringApplicationRunListeners`，通过调用starting()方法通知所有的SpringApplicationRunListeners：应用开始启动了。SpringApplicationRunListeners其本质上就是一个事件发布者，它在SpringBoot应用启动的不同时间点发布不同应用事件类型(ApplicationEvent)，如果有哪些事件监听者(ApplicationListener)对这些事件感兴趣，则可以接收并且处理。还记得初始化流程中，SpringApplication加载了一系列ApplicationListener吗？这个启动流程中没有发现有发布事件的代码，其实都已经在SpringApplicationRunListeners这儿实现了。

简单的分析一下其实现流程，首先看下SpringApplicationRunListener的源码：

```  
public interface SpringApplicationRunListener {  
  
    // 运行run方法时立即调用此方法，可以用户非常早期的初始化工作  
    void starting();  
    // Environment准备好后，并且ApplicationContext创建之前调用  
    void environmentPrepared(ConfigurableEnvironment environment);  
    // ApplicationContext创建好后立即调用  
    void contextPrepared(ConfigurableApplicationContext context);  
    // ApplicationContext加载完成，在refresh之前调用  
    void contextLoaded(ConfigurableApplicationContext context);  
    // 当run方法结束之前调用  
    void finished(ConfigurableApplicationContext context, Throwable exception);  
}  
  
```  

SpringApplicationRunListener只有一个实现类：`EventPublishingRunListener`。①处的代码只会获取到一个EventPublishingRunListener的实例，我们来看看starting()方法的内容：

```  
public void starting() {  
    // 发布一个ApplicationStartedEvent  
    this.initialMulticaster.multicastEvent(new ApplicationStartedEvent(this.application, this.args));}  
  
```  

顺着这个逻辑，你可以在②处的`prepareEnvironment()`方法的源码中找到`listeners.environmentPrepared(environment);`即SpringApplicationRunListener接口的第二个方法，那不出你所料，`environmentPrepared()`又发布了另外一个事件`ApplicationEnvironmentPreparedEvent`。接下来会发生什么，就不用我多说了吧。

② 创建并配置当前应用将要使用的`Environment`，Environment用于描述应用程序当前的运行环境，其抽象了两个方面的内容：配置文件(profile)和属性(properties)，开发经验丰富的同学对这两个东西一定不会陌生：不同的环境(eg：生产环境、预发布环境)可以使用不同的配置文件，而属性则可以从配置文件、环境变量、命令行参数等来源获取。因此，当Environment准备好后，在整个应用的任何时候，都可以从Environment中获取资源。

总结起来，②处的两句代码，主要完成以下几件事：

*   判断Environment是否存在，不存在就创建（如果是web项目就创建`StandardServletEnvironment`，否则创建`StandardEnvironment`）
*   配置Environment：配置profile以及properties
*   调用SpringApplicationRunListener的`environmentPrepared()`方法，通知事件监听者：应用的Environment已经准备好

③、SpringBoot应用在启动时会输出这样的东西：

```  
  .   ____          _            __ _ _ /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \  
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )  '  |____| .__|_| |_|_| |_\__, | / / / / =========|_|==============|___/=/_/_/_/ :: Spring Boot ::        (v1.5.6.RELEASE)  
```  

如果想把这个东西改成自己的涂鸦，你可以研究以下Banner的实现，这个任务就留给你们吧。

④、根据是否是web项目，来创建不同的ApplicationContext容器。

⑤、创建一系列`FailureAnalyzer`，创建流程依然是通过SpringFactoriesLoader获取到所有实现FailureAnalyzer接口的class，然后在创建对应的实例。FailureAnalyzer用于分析故障并提供相关诊断信息。

⑥、初始化ApplicationContext，主要完成以下工作：

*   将准备好的Environment设置给ApplicationContext
*   遍历调用所有的ApplicationContextInitializer的`initialize()`方法来对已经创建好的ApplicationContext进行进一步的处理
*   调用SpringApplicationRunListener的`contextPrepared()`方法，通知所有的监听者：ApplicationContext已经准备完毕
*   将所有的bean加载到容器中
*   调用SpringApplicationRunListener的`contextLoaded()`方法，通知所有的监听者：ApplicationContext已经装载完毕

⑦、调用ApplicationContext的`refresh()`方法，完成IoC容器可用的最后一道工序。从名字上理解为刷新容器，那何为刷新？就是插手容器的启动，联系一下第一小节的内容。那如何刷新呢？且看下面代码：

```  
// 摘自refresh()方法中一句代码  
invokeBeanFactoryPostProcessors(beanFactory);  
  
```  

看看这个方法的实现：

```  
protected void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory) {  
    PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());    ......}  
  
```  

获取到所有的`BeanFactoryPostProcessor`来对容器做一些额外的操作。BeanFactoryPostProcessor允许我们在容器实例化相应对象之前，对注册到容器的BeanDefinition所保存的信息做一些额外的操作。这里的getBeanFactoryPostProcessors()方法可以获取到3个Processor：

```  
ConfigurationWarningsApplicationContextInitializer$ConfigurationWarningsPostProcessor  
SharedMetadataReaderFactoryContextInitializer$CachingMetadataReaderFactoryPostProcessor  
ConfigFileApplicationListener$PropertySourceOrderingPostProcessor  
  
```  

不是有那么多BeanFactoryPostProcessor的实现类，为什么这儿只有这3个？因为在初始化流程获取到的各种ApplicationContextInitializer和ApplicationListener中，只有上文3个做了类似于如下操作：

```  
public void initialize(ConfigurableApplicationContext context) {  
    context.addBeanFactoryPostProcessor(new ConfigurationWarningsPostProcessor(getChecks()));}  
  
```  

然后你就可以进入到`PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors()`方法了，这个方法除了会遍历上面的3个BeanFactoryPostProcessor处理外，还会获取类型为`BeanDefinitionRegistryPostProcessor`的bean：`org.springframework.context.annotation.internalConfigurationAnnotationProcessor`，对应的Class为`ConfigurationClassPostProcessor`。`ConfigurationClassPostProcessor`用于解析处理各种注解，包括：@Configuration、@ComponentScan、@Import、@PropertySource、@ImportResource、@Bean。当处理`@import`注解的时候，就会调用<自动配置>这一小节中的`EnableAutoConfigurationImportSelector.selectImports()`来完成自动配置功能。其他的这里不再多讲，如果你有兴趣，可以查阅参考资料6。

⑧、查找当前context中是否注册有CommandLineRunner和ApplicationRunner，如果有则遍历执行它们。

⑨、执行所有SpringApplicationRunListener的finished()方法。

这就是Spring Boot的整个启动流程，其核心就是在Spring容器初始化并启动的基础上加入各种扩展点，这些扩展点包括：ApplicationContextInitializer、ApplicationListener以及各种BeanFactoryPostProcessor等等。你对整个流程的细节不必太过关注，甚至没弄明白也没有关系，你只要理解这些扩展点是在何时如何工作的，能让它们为你所用即可。

整个启动流程确实非常复杂，可以查询参考资料中的部分章节和内容，对照着源码，多看看，我想最终你都能弄清楚的。言而总之，Spring才是核心，理解清楚Spring容器的启动流程，那Spring Boot启动流程就不在话下了。

## 参考文章
[1][王福强 著；SpringBoot揭秘：快速构建微服务体系; 机械工业出版社, 2016](https://link.jianshu.com/?t=http%3A%2F%2Funion-click.jd.com%2Fjdc%3Fd%3D4jESQ9)  
[2][王福强 著；Spring揭秘; 人民邮件出版社, 2009](https://link.jianshu.com/?t=http%3A%2F%2Funion-click.jd.com%2Fjdc%3Fd%3DyzfgeF)  
[3][Craig Walls 著；丁雪丰 译；Spring Boot实战；中国工信出版集团 人民邮电出版社，2016](https://link.jianshu.com/?t=http%3A%2F%2Funion-click.jd.com%2Fjdc%3Fd%3DAQ6oHO)  
[4][深入探讨 Java 类加载器](https://link.jianshu.com/?t=https%3A%2F%2Fwww.ibm.com%2Fdeveloperworks%2Fcn%2Fjava%2Fj-lo-classloader%2F):[https://www.ibm.com/developerworks/cn/java/j-lo-classloader/](https://link.jianshu.com/?t=https%3A%2F%2Fwww.ibm.com%2Fdeveloperworks%2Fcn%2Fjava%2Fj-lo-classloader%2F)  
[5][spring boot实战：自动配置原理分析](https://link.jianshu.com/?t=http%3A%2F%2Fblog.csdn.net%2Fliaokailin%2Farticle%2Fdetails%2F49559951):[http://blog.csdn.net/liaokailin/article/details/49559951](https://link.jianshu.com/?t=http%3A%2F%2Fblog.csdn.net%2Fliaokailin%2Farticle%2Fdetails%2F49559951)  
[6][spring boot实战：Spring boot Bean加载源码分析](https://link.jianshu.com/?t=http%3A%2F%2Fblog.csdn.net%2Fliaokailin%2Farticle%2Fdetails%2F49107209)：[http://blog.csdn.net/liaokailin/article/details/49107209](https://link.jianshu.com/?t=http%3A%2F%2Fblog.csdn.net%2Fliaokailin%2Farticle%2Fdetails%2F49107209)

