## 1 概述
Spring Boot 通过其自动配置功能使配置 Spring 变得更加容易。

在本快速教程中，我们将探索 org.springframework.boot.autoconfigure 和 org.springframework.boot.autoconfigure.condition 包中的注解。

## 2 @SpringBootApplication
我们使用这个注解来标记 Spring Boot 应用程序的主类：

@SpringBootApplication
我们使用这个注解来标记 Spring Boot 应用程序的主类：
````
@SpringBootApplication
class VehicleFactoryApplication {

    public static void main(String[] args) {
        SpringApplication.run(VehicleFactoryApplication.class, args);
    }
}
````
@SpringBootApplication 封装了@Configuration、@EnableAutoConfiguration 和@ComponentScan 注解及其默认属性。

## 3 @EnableAutoConfiguration

@EnableAutoConfiguration，顾名思义，启用自动配置。 这意味着 Spring Boot 在其类路径中查找自动配置 bean 并自动应用它们。

请注意，我们必须将此注释与@Configuration 一起使用：

````
@Configuration
@EnableAutoConfiguration
class VehicleFactoryConfig {}
````

## 4 @Configuration以及相关类

@Configuration的作用：标注在类上，配置spring容器(应用上下文)。

以往我们在spring中配置bean会使用到xml配置文件，定义一个个bean，在springboot中，为了做到零配置，spring框架提供了@Configuration这一注解

相当于把该类作为spring的xml配置文件中的<beans>

@Configuration注解的类中，使用@Bean注解标注的方法，返回的类型都会直接注册为bean。

@Configure注解的定义如下：
````
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
String value() default "";
}
````
从定义来看，底层是含有@Component ，所以@Configuration 具有和 @Component 的作用。因此context:component-scan/或者@ComponentScan都能处理@Configuration注解的类。

通常，当我们编写自定义自动配置时，我们希望 Spring 有条件地使用它们。 我们可以通过本节中的注释来实现这一点。

我们可以将此部分中的注释放在@Configuration 类或@Bean 方法上。

在接下来的部分中，我们将只介绍每个条件背后的基本概念。 欲了解更多信息，请访问这篇文章。

### 4.1 @ConditionalOnClass and @ConditionalOnMissingClass
很明显这是一个带有条件判断的配置注解，要知道，很多时候我们按需加载bean的，甚至是根据外部jar包是否存在来进行加载和判断的。

这种时候，就要根据外部类是否存在来决定是否是否加载该bean

使用这些条件，如果注释参数中的类存在/不存在，Spring 将仅使用标记的自动配置 bean：

````
@Configuration
@ConditionalOnClass(DataSource.class)
class MySQLAutoconfiguration {
//...
}
````

### 4.2 @ConditionalOnBean and @ConditionalOnMissingBean

当我们想要根据特定 bean 的存在或不存在来定义条件时，我们可以使用这些注释：

和上面一个注解稍有些不同，因为我们的判断条件变成了bean

````
@Bean
@ConditionalOnBean(name = "dataSource")
LocalContainerEntityManagerFactoryBean entityManagerFactory() {
// ...
}
````
### 4.3 @ConditionalOnProperty
通过这个注解，我们可以对属性的值设置条件

要注意，这里的属性值来源于application.properties文件中的配置

````
@Bean
@ConditionalOnProperty(
name = "usemysql",
havingValue = "local"
)
DataSource dataSource() {
// ...
}
````
 
### 4.4 @ConditionalOnResource

我们可以让 Spring 仅在存在特定资源时使用定义：
顾名思义，要求classpath存在这个资源文件时才进行加载，这也是很常用的一个注解。

````

@ConditionalOnResource(resources = "classpath:mysql.properties")
Properties  ditionalProperties() {
// ...
}
````
 
### 4.5 @ConditionalOnWebApplication and @ConditionalOnNotWebApplication
这个注解通常用于和web有强关联，或者完全无关联的配置上

使用这些注释，我们可以根据当前应用程序是否是 Web 应用程序来创建条件：
````

@ConditionalOnWebApplication
HealthCheckController healthCheckController() {
// ...
}
````
 
### 4.6 @ConditionalExpression
springboot真的为我们想到了所有情况，如果上面的注解还不能满足你的要求，那么干脆就让你自己写，这下应该没有问题吧？

我们可以在更复杂的情况下使用这个注解。 当 SpEL 表达式被评估为真时，Spring 将使用标记的定义：

````
@Bean
@ConditionalOnExpression("${usemysql} && ${mysqlserver == 'local'}")
DataSource dataSource() {
// ...
}
````

### 4.7 @Conditional
什么，还有问题？
那springboot也不再提供什么表达式给你了，直接让用户写一个判断条件，告诉我true或者false就行了

对于更复杂的条件，我们可以创建一个评估自定义条件的类。 我们告诉 Spring 将这个自定义条件与 @Conditional 一起使用：

````
@Conditional(HibernateCondition.class)
Properties  ditionalProperties() {
//...
}
````

## 5 总结
在本文中，我们概述了如何微调自动配置过程并为自定义自动配置 bean 提供条件。