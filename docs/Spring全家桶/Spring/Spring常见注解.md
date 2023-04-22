# Spring注解
## 1 概述

我们都知道Spring最核心的特性就是IOC+AOP，IOC的原理就是实现了一个Spring容器，用来管理所有Spring Bean实例
DI，也就是依赖注入，是我们作为开发者需要关心的核心话题，如何注入依赖，注入哪个依赖，是我们需要明确知道的。
在以前，我们习惯于用xml配置文件来声明一个bean，但现在，我们更多地使用注解和代码来完成DI的过程


我们可以使用 org.springframework.beans.factory.annotation 和 org.springframework.context.annotation 包中的注释来启用 Spring DI 引擎的功能。

我们通常将这些称为“Spring 核心注释”，我们将在本教程中对其进行回顾。


## 2 DI相关注解

### 2.1 @Autowired

我们可以使用 @Autowired 来标记 Spring 将要解析和注入的依赖项。
我们可以将此注释与构造函数、setter 或字段注入一起使用。
Constructor injection:

**构造器注入**

````
class Car {
    Engine engine;

    @Autowired
    Car(Engine engine) {
        this.engine = engine;
    }
}
````

**Setter注入**
````
class Car {
    Engine engine;

    @Autowired
    void setEngine(Engine engine) {
        this.engine = engine;
    }
}
````
**变量注入**
````
class Car {
    @Autowired
    Engine engine;
}
````

@Autowired 有一个名为 required 的布尔参数，默认值为 true。

当它找不到合适的 bean 来连接时，它会调整 Spring 的行为。 当为 true 时，将抛出异常，否则不连接任何内容。

请注意，如果我们使用构造函数注入，则所有构造函数参数都是强制性的。

从 4.3 版本开始，我们不需要显式地使用 @Autowired 注解构造函数，除非我们声明至少两个构造函数。

### 2.2 @Bean

@Bean 标记实例化 Spring bean 的工厂方法：

```
@Bean
Engine engine() {
    return new Engine();
}
````

当需要返回类型的新实例时，Spring 会调用这些方法。

生成的 bean 与工厂方法同名。 如果我们想以不同的方式命名，我们可以使用此注释的名称或值参数（参数值是参数名称的别名）：

````
@Bean("engine")
Engine getEngine() {
    return new Engine();
}
````
这是一种非常常见的bean声明方式，因为很多的bean并不是我们一开始就在代码里定义好的，它可能需要基于运行时环境来进行按需构建。

我们可以自由地声明和定义Bean，并且也可以赋予它自定义的bean名称。

注意，所有用@Bean 注释的方法都必须在@Configuration 类中。

### 2.3 @Qualifier

我们使用@Qualifier 和@Autowired 来提供我们想要在不明确情况下使用的bean id 或bean 名称。

例如，以下两个 bean 实现相同的接口：
````
class Bike implements Vehicle {}

class Car implements Vehicle {}

````

如果 Spring 需要注入一个 Vehicle bean，它会以多个匹配定义结束。 在这种情况下，我们可以使用 @Qualifier 注释显式提供 bean 的名称。

**构造器注入**
````
@Autowired
Biker(@Qualifier("bike") Vehicle vehicle) {
this.vehicle = vehicle;
}
````

**Setter注入**

````
@Autowired
void setVehicle(@Qualifier("bike") Vehicle vehicle) {
this.vehicle = vehicle;
}
````
或者:

````
@Autowired
@Qualifier("bike")
void setVehicle(Vehicle vehicle) {
this.vehicle = vehicle;
````
**变量注入**

````
@Autowired
@Qualifier("bike")
Vehicle vehicle;
````
这个注解我们可能平常用的不多，但是当一个接口有多个实现类时，它就会经常派上用场！

### 2.4 @Required

@Required 在 setter 方法上标记我们想要通过 XML 填充的依赖项：
````
@Required
void setColor(String color) {
this.color = color;
}
````
xml
````
<bean class="com.baeldung.annotations.Bike">
    <property name="color" value="green" />
</bean>
````
否则，将抛出 BeanInitializationException。
非常少见的用法，知道一下就行了

### 2.5 @Value
我们可以使用 @Value 将属性值注入 bean。 它与构造函数、setter 和字段注入兼容。

这也是非常常用的一个注解，因为我们很多时候都需要从application.properties或者其他配置文件中来读取配置属性值。

**构造器注入**
````
Engine(@Value("8") int cylinderCount) {
this.cylinderCount = cylinderCount;
}
````

**setter注入**

````
@Autowired
void setCylinderCount(@Value("8") int cylinderCount) {
this.cylinderCount = cylinderCount;
}
````

或者:
````

@Value("8")
void setCylinderCount(int cylinderCount) {
this.cylinderCount = cylinderCount;
}
````

**变量注入**
````
@Value("8")
int cylinderCount;
````

当然，注入静态值是没有用的。 因此，我们可以在 @Value 中使用占位符字符串来连接在外部源中定义的值，例如在 .properties 或 .yaml 文件中。
````

engine.fuelType=petrol
````

我们可以通过以下方式注入 engine.fuelType 的值：

````
@Value("${engine.fuelType}")
String fuelType;
````

我们甚至可以使用 SpEL 来使用@Value。 更多高级示例可以在我们关于@Value 的文章中找到。

### 2.6 @DependsOn
我们可以使用此注解让 Spring 在注解的 bean 之前初始化其他 bean。 通常，此行为是自动的，基于 bean 之间的显式依赖关系。

我们只有在依赖是隐式的时候才需要这个注解，比如JDBC驱动加载或者静态变量初始化。

我们可以在指定依赖 bean 名称的依赖类上使用 @DependsOn。 注释的值参数需要一个包含依赖 bean 名称的数组：

````
@DependsOn("engine")
class Car implements Vehicle {}
````
Alternatively, if we define a bean with the @Bean annotation, the factory method should be annotated with @DependsOn:
````
@Bean
@DependsOn("fuel")
Engine engine() {
return new Engine();
}
````
### 2.7 @Lazy
当我们想懒惰地初始化我们的 bean 时，我们使用 @Lazy。 默认情况下，Spring 在应用程序上下文的启动/引导时急切地创建所有单例 bean。

但是，有些情况下我们需要在请求时创建 bean，而不是在应用程序启动时创建。

这个注解的行为会根据我们放置它的确切位置而有所不同。 我们可以把它放在：

一个 @Bean 注释的 bean 工厂方法，以延迟方法调用（因此创建 bean）
@Configuration 类和所有包含的@Bean 方法都会受到影响

一个 @Component 类，它不是 @Configuration 类，这个 bean 将被延迟初始化

@Autowired 构造函数、setter 或字段，用于延迟加载依赖项本身（通过代理）

````
@Configuration
@Lazy
class VehicleFactoryConfig {

    @Bean
    @Lazy(false)
    Engine engine() {
        return new Engine();
    }
}
````
这同样不是一个常用的注解。

当我们维护一个有大量bean的项目时，会发现有很多bean可能都是按需使用的，不一定需要在容器初始化时就进行初始化，这可以帮我们节省很多时间和性能。


### 2.8 @Lookup
同样是一个比较少用到的注解

@Lookup 会告诉 Spring 在我们调用它时返回方法返回类型的实例。

本质上，Spring 将覆盖我们带注释的方法并使用我们方法的返回类型和参数作为 BeanFactory#getBean 的参数。

@Lookup 适用于：

将原型 bean 注入我们自己的bean（类似于 Provider）

如果我们碰巧决定拥有一个原型 Spring bean，那么我们几乎会立即面临这样的问题：

我们的单例 Spring bean 将如何访问这些原型 Spring bean？

现在，Provider 肯定是一种方式，尽管 @Lookup 在某些方面更加通用。

要注意的是，spring默认使用的单例bean，所以如果我们要注入原型bean，我们才需要做这样的额外工作

首先，让我们创建一个原型 bean，稍后我们将其注入到单例 bean 中：
````
@Component
@Scope("prototype")
public class SchoolNotification {
// ... prototype-scoped state
}
````
使用@Lookup，我们可以通过单例 bean 获取 SchoolNotification 的实例：

````
@Component
public class StudentServices {

    // ... member variables, etc.

    @Lookup
    public SchoolNotification getNotification() {
        return null;
    }

    // ... getters and setters
}
````
Using @Lookup, we can get an instance of SchoolNotification through our singleton bean:
````
@Test
public void whenLookupMethodCalled_thenNewInstanceReturned() {
// ... initialize context
StudentServices first = this.context.getBean(StudentServices.class);
StudentServices second = this.context.getBean(StudentServices.class);

    assertEquals(first, second); 
    assertNotEquals(first.getNotification(), second.getNotification()); 
}
````
请注意，在 StudentServices 中，我们将 getNotification 方法保留为存根。

这是因为 Spring 通过调用 beanFactory.getBean(StudentNotification.class) 覆盖了该方法，因此我们可以将其留空。


### 2.9 @Primary
有时我们需要定义多个相同类型的bean。 在这些情况下，注入将不成功，因为 Spring 不知道我们需要哪个 bean。

我们已经看到了处理这种情况的选项：用@Qualifier 标记所有接线点并指定所需 bean 的名称。

然而，大多数时候我们需要一个特定的 bean，很少需要其他 bean。

我们可以使用@Primary 来简化这种情况：如果我们用@Primary 标记最常用的bean，它将在unqualified的注入点上被选择：

````
@Component
@Primary
class Car implements Vehicle {}

@Component
class Bike implements Vehicle {}

@Component
class Driver {
@Autowired
Vehicle vehicle;
}

@Component
class Biker {
@Autowired
@Qualifier("bike")
Vehicle vehicle;
}
````
在前面的示例中，汽车是主要车辆。 因此，在 Driver 类中，Spring 注入了一个 Car bean。 当然，在 Biker bean 中，字段 vehicle 的值将是一个 Bike 对象，因为它是qualified的。

### 2.10 @Scope

对于通常的应用来说，我们bean的scope默认都是单例的，但实际上spring bean可以支持多种多样的作用范围，它们有着不同的生命周期。

我们使用@Scope 来定义@Component 类或@Bean 定义的范围。 它可以是单例、原型、请求、会话、globalSession 或一些自定义范围。

对应的枚举值为
````
singleton
prototype
request
session
application
websocket
````

例子
````
@Component
@Scope("prototype")
class Engine {}
````
我们可以想象一些场景，比如request、session、websocket这类作用域的bean，通常应该是和网络请求相关的，比如用存储用户信息、session之类的bean。

具体怎么使用，就要根据你的具体场景来选择了，这会是一个很大的话题，这里就先不展开了，以后可以在单独的文章里继续介绍。

## 3 上下文配置注解
我们可以使用本节中描述的注释来配置应用程序上下文。


### 3.1 @Profile

如果我们希望 Spring 仅在特定配置文件处于活动状态时使用@Component 类或@Bean 方法，我们可以使用@Profile 对其进行标记。

我们可以使用注释的值参数配置配置文件的名称：

我们通常这个注解来配置不同环境的配置。
比如下面这个例子

````
public interface DatasourceConfig {
public void setup();
}
````

下面是开发环境的配置：

````
@Component
@Profile("dev")
public class DevDatasourceConfig implements DatasourceConfig {
@Override
public void setup() {
System.out.println("Setting up datasource for DEV environment. ");
}
}
````
下面是生产环境的配置：

````
@Component
@Profile("production")
public class ProductionDatasourceConfig implements DatasourceConfig {
@Override
public void setup() {
System.out.println("Setting up datasource for PRODUCTION environment. ");
}
}
````
当然我们也可以使用xml或者其他类型的配置文件来描述这个配置bean

xml
````
<beans profile="local">
    <bean id="localDatasourceConfig" 
      class="org.test.profiles.LocalDatasourceConfig" />
</beans>
````
### 3.2 @Import

我们可以使用特定的 @Configuration 类，而无需使用此注解进行组件扫描。 我们可以为这些类提供@Import 的值参数

正常来说，如果要用到@Configuration注解的bean，那么spring应用必须要扫描到该目录，如果恰好没有对该路径进行扫描，或者我们只是想使用路径下的单个配置类，那么我们就可以使用@Import来导入这个配置类了。

这个注解还是非常常用的，看一下下面这个例子

````
@Import(VehiclePartSupplier.class)
class VehicleFactoryConfig {}

@Configuration
class VehiclePartSupplier{
}
````

### 3.3 @ImportResource

比如说：现在有一个 bean.xml 的配置文件，需要将该 beans.xml 中定义的 bean对象 都导入到 Spring Boot 环境的容器中，该如何操作呢？

1.Spring 方式的配置文件 bean.xml 此处随便举个示例，比如说 xml 中配置了一个 helloService，如下所示
````
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <!--将 HelloService 以xml的方式,注入到容器中-->
    <bean id="helloService" class="com.demo.springboot.service.HelloService"></bean>
</beans>
````
2.使用@ImportResource注解，引入 xml 配置
````
/**
 * Spring Boot里面没有Spring的配置文件，我们自己编写的配置文件，也不能自动识别；
 * 如果想让Spring的配置文件生效，加载到Spring 容器中来；
 * 使用@ImportResource注解，将其标注在一个配置类上(此处配置在启动类)
 */
@SpringBootApplication
@ImportResource(locations = {"classpath:beans.xml"})
public class BootApplication {

    public static void main(String[] args) {
        // Spring应用启动起来         
        SpringApplication.run(BootApplication.class,args);

    }
}

````
### 3.4 @PropertySource
通过这个注解，我们可以为应用程序设置定义属性文件：

@PropertySource 注解提供了一种方便的声明性机制，用于将 PropertySource 添加到 Spring 的 Environment 中，与 @Configuration 类一起使用。

你可以使用 @Value 去引用定义的属性，例如：@Value("testbean.name")；也可以指定默认值，如：@Value("testbean.name:defaultValue")。

用法示例

给定一个配置文件app.properties
````
testbean.name=myTestBean
````
以下 @Configuration 类使用 @PropertySource 将 app.properties 设置给 Environment 的 PropertySources 集合。
````
@Configuration
@PropertySource("classpath:/com/myco/app.properties")
public class AppConfig {

    @Autowired
    Environment env;
 
    @Bean
    public TestBean testBean() {
        TestBean testBean = new TestBean();
        testBean.setName(env.getProperty("testbean.name"));
        return testBean;
    }
}
````

注意：使用 @Autowired 将 Environment 对象注入到配置类中，然后在 testBean() 方法中使用。
以上配置中，调用 testBean.getName() 方法将返回“myTestBean”字符串。

@PropertySource 利用了 Java 8 的重复注解特性，这意味着我们可以用它多次标记一个类：

````
@Configuration
@PropertySource("classpath:/annotations.properties")
@PropertySource("classpath:/vehicle-factory.properties")
class VehicleFactoryConfig {}
````

### 3.5 @PropertySources
用法同上，只不过，这一次我们可以使用这个注解指定多个@PropertySource 配置：
````
@Configuration
@PropertySources({
@PropertySource("classpath:/annotations.properties"),
@PropertySource("classpath:/vehicle-factory.properties")
})
class VehicleFactoryConfig {}
````
请注意，自 Java 8 以来，我们可以通过上述重复注释功能实现相同的功能。

## 4.结论

在本文中，我们看到了最常见的 Spring 核心注释的概述。 我们看到了如何配置 bean 连接和应用上下文，以及如何为组件扫描标记类。

spring体系中的常见注解还有很多，一篇文章不可能全部覆盖，如有遗漏，欢迎补充。

# Spring Bean注解

## 1 概述
在本教程中，我们将讨论用于定义不同类型 bean 的最常见的 Spring bean 注释。

有几种方法可以在 Spring 容器中配置 bean。 首先，我们可以使用 XML 配置声明它们。 我们还可以在配置类中使用@Bean 注解来声明 bean。

最后，我们可以使用 org.springframework.stereotype 包中的注释之一来标记该类，并将其余部分留给组件扫描。

## 2 @ComponentScan
这是我们经常会使用的一个注解，在我们的应用中，有时候不一定会扫描所有的包，特别是当我们要扫描外部jar包中的bean时，它非常有用。

它可以加在SpringBootApplication上，也可以加在@configuration注解上的配置类上

如果启用了组件扫描，Spring 可以自动扫描包中的 bean。

@ComponentScan 配置使用注解配置扫描哪些包的类。 

我们可以直接使用 basePackages 或 value 参数之一指定基本包名称（value 是 basePackages 的别名）

````
@Configuration
@ComponentScan(basePackages = "com.baeldung.annotations")
class VehicleFactoryConfig {}
````
此外，我们可以使用 basePackageClasses 参数指向基础包中的类：

````
@Configuration
@ComponentScan(basePackageClasses = VehicleFactoryConfig.class)
class VehicleFactoryConfig {}
````

这两个参数都是数组，因此我们可以为每个参数提供多个包。

如果未指定参数，则扫描发生在存在 @ComponentScan 注释类的同一包中。

@ComponentScan 利用了 Java 8 的重复注解特性，这意味着我们可以用它多次标记一个类：

````
@Configuration
@ComponentScan(basePackages = "com.baeldung.annotations")
@ComponentScan(basePackageClasses = VehicleFactoryConfig.class)
class VehicleFactoryConfig {}
````

或者，我们可以使用 @ComponentScans 指定多个 @ComponentScan 配置：

````
@Configuration
@ComponentScans({
@ComponentScan(basePackages = "com.baeldung.annotations"),
@ComponentScan(basePackageClasses = VehicleFactoryConfig.class)
})
````
````
class VehicleFactoryConfig {
}
````
使用 XML 配置时，配置组件扫描同样简单：

````
<context:component-scan base-package="com.baeldung"/>
````

### 3 @Component

@Component 是类级别的注解。 在组件扫描期间，Spring Framework 会自动检测使用@Component 注解的类：
````
@Component
class CarUtility {
// ...
}
````

默认情况下，此类的 bean 实例与类名同名，首字母小写。 此外，我们可以使用此注释的可选值参数指定不同的名称。

由于@Repository、@Service、@Configuration 和@Controller 都是带有@Component 的注解，它们共享相同的bean 命名行为。 

Spring 还会在组件扫描过程中自动检测它们。

通常来说，我们会在mvc应用中我们会用到上述几种注解，而我们在非web应用中更多地可以使用@component来注解bean

### 4 @Repository

DAO or Repository classes usually represent the database access layer in an application, and should be annotated with @Repository:
````
@Repository
class VehicleRepository {
// ...
}
````
使用此注释的一个优点是它启用了自动持久性异常转换。 当使用持久性框架（如 Hibernate）时，在使用 @Repository 注释的类中抛出的本机异常将自动转换为 Spring 的 DataAccessExeption 的子类。

要启用异常转换，我们需要声明我们自己的 PersistenceExceptionTranslationPostProcessor bean：
````
@Bean
public PersistenceExceptionTranslationPostProcessor exceptionTranslation() {
return new PersistenceExceptionTranslationPostProcessor();
}
````
请注意，在大多数情况下，Spring 会自动执行上述步骤。

或者通过 XML 配置：
````
<bean class=
"org.springframework.dao.annotation.PersistenceExceptionTranslationPostProcessor"/>
````

### 5 @Service
应用程序的业务逻辑通常驻留在服务层中，因此我们将使用@Service 注释来指示一个类属于该层：

````
@Service
public class VehicleService {
// ...    
}
````
### 6 @Controller
@Controller 是一个类级别的注解，它告诉 Spring Framework 这个类作为 Spring MVC 中的控制器：

spring会对@Controller 注解的bean做很多事情，具体内容我们会在SpringMVC相关的内容来讲述

````
@Controller
public class VehicleController {
// ...
}

````
## 7 @Configuration

配置类可以包含用@Bean 注释的 bean 定义方法：
````
@Configuration
class VehicleFactoryConfig {

    @Bean
    Engine engine() {
        return new Engine();
    }

}
````
## 8 AOP注解
当我们使用 Spring 构造型注释时，很容易创建一个切入点，该切入点以所有具有特定构造型的类为目标。

例如，假设我们想测量 DAO 层方法的执行时间。 我们将创建以下方面（使用 AspectJ 注释），利用 @Repository 构造型：

```
@Aspect
@Component
public class PerformanceAspect {
@Pointcut("within(@org.springframework.stereotype.Repository *)")
public void repositoryClassMethods() {};

    @Around("repositoryClassMethods()")
    public Object measureMethodExecutionTime(ProceedingJoinPoint joinPoint) 
      throws Throwable {
        long start = System.nanoTime();
        Object returnValue = joinPoint.proceed();
        long end = System.nanoTime();
        String methodName = joinPoint.getSignature().getName();
        System.out.println(
          "Execution of " + methodName + " took " + 
          TimeUnit.NANOSECONDS.toMillis(end - start) + " ms");
        return returnValue;
    }
}
````

在此示例中，我们创建了一个切入点，该切入点匹配使用@Repository 注释的类中的所有方法。 然后我们使用@Around 通知来定位那个切入点，并确定拦截方法调用的执行时间。

此外，使用这种方法，我们可以为每个应用程序层添加日志记录、性能管理、审计和其他行为。

当然了，aspectJ的相关注解还很多，可以用来声明切面，未来我们也会单独写文介绍。

## 9 结论

在本文中，我们检查了 Spring 构造型注释并讨论了它们各自代表的语义类型。

我们还学习了如何使用组件扫描来告诉容器在哪里可以找到带注释的类。

最后，我们了解了这些注释如何导致干净、分层的设计以及应用程序关注点之间的分离。 它们还使配置更小，因为我们不再需要手动显式定义 bean。

# 参考文章
https://www.baeldung.com/spring-annotations

