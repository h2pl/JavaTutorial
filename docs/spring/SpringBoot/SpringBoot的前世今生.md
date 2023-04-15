# SpringBoot的前世今生

Spring Boot 2.0 的推出又激起了一阵学习 Spring Boot 热，就单从我个人的博客的访问量大幅增加就可以感受到大家对学习 Spring Boot 的热情，那么在这么多人热衷于学习 Spring Boot 之时，我自己也在思考： Spring Boot 诞生的背景是什么？Spring 企业又是基于什么样的考虑创建 Spring Boot? 传统企业使用 Spring Boot 会给我们带来什么样变革?

带着这些问题，我们一起来了解下 Spring Boot 到底是什么?

## Spring 历史

说起 Spring Boot 我们不得不先了解一下 Spring 这个企业，不仅因为 Spring Boot 来源于 Spirng 大家族，而且 Spring Boot 的诞生和 Sping 框架的发展息息相关。

时间回到2002年，当时正是 Java EE 和 EJB 大行其道的时候，很多知名公司都是采用此技术方案进行项目开发。这时候有一个美国的小伙子认为 EJB 太过臃肿，并不是所有的项目都需要使用 EJB 这种大型框架，应该会有一种更好的方案来解决这个问题。

为了证明他的想法是正确的，于2002年10月甚至写了一本书《 Expert One-on-One J2EE 》，介绍了当时 Java 企业应用程序开发的情况，并指出了 Java EE 和 EJB 组件框架中存在的一些主要缺陷。在这本书中，他提出了一个基于普通 Java 类和依赖注入的更简单的解决方案。

在书中，他展示了如何在不使用 EJB 的情况下构建高质量，可扩展的在线座位预留系统。为了构建应用程序，他编写了超过 30,000 行的基础结构代码，项目中的根包命名为 com.interface21，所以人们最初称这套开源框架为 interface21，也就是 Spring 的前身。

他是谁呢，他就是大名鼎鼎的 Rod Johnson （下图）, Rod Johnson 在悉尼大学不仅获得了计算机学位，同时还获得了音乐学位，更令人吃惊的是在回到软件开发领域之前，他还获得了音乐学的博士学位。现在 Rod Johnson 已经离开了 Spring ，成为了一个天使投资人，同时也是多个公司的董事，早已走上人生巅峰。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springboot-hot.png)

在这本书发布后，一对一的 J2EE 设计和开发一炮而红。这本书免费提供的大部分基础架构代码都是高度可重用的。 2003 年 Rod Johnson 和同伴在此框架的基础上开发了一个全新的框架命名为 Spring ,据 Rod Johnson 介绍 Spring 是传统 J2EE 新的开始。随后 Spring 发展进入快车道。

*   2004 年 03 月，1.0 版发布。
*   2006 年 10 月，2.0 版发布。
*   2007 年 11 月更名为 SpringSource，同时发布了 Spring 2.5。
*   2009 年 12 月，Spring 3.0 发布。
*   2013 年 12 月，Pivotal 宣布发布 Spring 框架 4.0。
*   2017 年 09 月，Spring 5.0 发布。

## Spring Boot 的诞生

随着使用 Spring 进行开发的个人和企业越来越多，Spring 也慢慢从一个单一简洁的小框架变成一个大而全的开源软件，Spring 的边界不断的进行扩充，到了后来 Spring 几乎可以做任何事情了，市面上主流的开源软件、中间件都有 Spring 对应组件支持，人们在享用 Spring 的这种便利之后，也遇到了一些问题。

Spring 每集成一个开源软件，就需要增加一些基础配置，慢慢的随着人们开发的项目越来越庞大，往往需要集成很多开源软件，因此后期使用 Spirng 开发大型项目需要引入很多配置文件，太多的配置非常难以理解，并容易配置出错，到了后来人们甚至称 Spring 为配置地狱。

Spring 似乎也意识到了这些问题，急需有这么一套软件可以解决这些问题，这个时候微服务的概念也慢慢兴起，快速开发微小独立的应用变得更为急迫，Spring 刚好处在这么一个交叉点上，于 2013 年初开始的 Spring Boot 项目的研发，2014年4月，Spring Boot 1.0.0 发布。

Spring Boot 诞生之初，就受到开源社区的持续关注，陆续有一些个人和企业尝试着使用了 Spring Boot，并迅速喜欢上了这款开源软件。直到2016年，在国内 Spring Boot 才被正真使用了起来，期间很多研究 Spring Boot 的开发者在网上写了大量关于 Spring Boot 的文章，同时有一些公司在企业内部进行了小规模的使用，并将使用经验分享了出来。从2016年到2018年，使用 Spring Boot 的企业和个人开发者越来越多，我们从 Spring Boot 关键字的百度指数就可以看出。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springboot-hot.png)

上图为2014年到2018年 Spring Boot 的百度指数，可以看出 Spring Boot 2.0 的推出引发了搜索高峰。

当然 Spring Boot 不是为了取代 Spring ,Spring Boot 基于 Spring 开发，是为了让人们更容易的使用 Spring。看到 Spring Boot 的市场反应，Spring 官方也非常重视 Spring Boot 的后续发展，已经将 Spring Boot 作为公司最顶级的项目来推广，放到了官网上第一的位置，因此后续 Spring Boot 的持续发展也被看好。

# 基本介绍

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230415111020.png)

这是spring官网对于springboot的基本介绍

Spring Boot makes it easy to create stand-alone, production-grade Spring based Applications that you can "just run".

We take an opinionated view of the Spring platform and third-party libraries so you can get started with minimum fuss. Most Spring Boot applications need minimal Spring configuration.

简而言之，就是在告诉我们，springboot可以帮助我们快速构建独立部署的生产级应用

，只需要一个带有@SpringBootApplication 注解的应用入口，即可被识别为一个springboot应用

并且，springboot继承了大量的第三方库，我们仅需很少甚至是完全不需要额外的配置，就可以进行应用搭建，因为springboot框架本身已经帮我们设置好了大量的默认配置。

# 基本特性

Features

*   Create stand-alone Spring applications
*   Embed Tomcat, Jetty or Undertow directly (no need to deploy WAR files)
*   Provide opinionated 'starter' dependencies to simplify your build configuration
*   Automatically configure Spring and 3rd party libraries whenever possible
*   Provide production-ready features such as metrics, health checks, and externalized configuration
*   Absolutely no code generation and no requirement for XML configuration

官方对于springboot几大特性的描述如上

1、创建一个独立部署的spring应用，它常用于定义微服务，或者是一个web应用

2、内置的tomcat容器，我们再也需要把spring应用打包成war包，而是只需要将其打包成jar包进行部署即可。

3、提供starter依赖，更加简化了springboot生态中的依赖和配置，例如当我们使用springweb相关依赖时，我们不需要单独地添加多个springmvc相关的maven依赖，而是直接引用spring-boot-starter-web就可以了，这个starter会自动引入相关的依赖，并且帮你管理版本号，会大量地简化和优化你的maven配置

4、自动装配spring和第三方库，这里主要是指通过注解和配置文件进行自动装配，并且还有一些是基于约定的自动装配机制，这可以帮我们节省大量的额外配置，也可以自动装配外部jar包提供的spring相关bean和配置。

5、提供了生产环境相关的特性，比如监控、指标打点、健康检查等功能，springboot提供了强大的生态组件，这里不仅包括springboot自己的组件，也包括一些外部的生态组件。

6、不不存在代码生成，也不需要xml配置文件，这证明了springboot的框架完整度，以及对于配置简化的极致追求，我们仅需要application.properties进行基本配置参数管理，你也不会在springboot编译器找到动态生成的多余class文件。

# 和Spring的关系

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230415111139.png)

## SpringFramework解决了什么问题？

Spring是Java企业版（Java Enterprise Edition，JEE，也称J2EE）的轻量级代替品。无需开发重量级的EnterpriseJavaBean（EJB），Spring为企业级Java开发提供了一种相对简单的方法，通过依赖注入和面向切面编程，用简单的Java对象（Plain Old Java Object，POJO）实现了EJB的功能。

1.使用Spring的IOC容器,将对象之间的依赖关系交给Spring,降低组件之间的耦合性,让我们更专注于应用逻辑

2.可以提供众多服务,事务管理,WS等。

3.很好地支持AOP,方便面向切面编程。

4.对主流的框架提供了很好的集成支持,如Hibernate,Struts2,JPA等

5.Spring DI机制降低了业务对象替换的复杂性。

6.Spring属于低侵入,代码污染极低。

7.Spring的高度可开放性,并不强制依赖于Spring,开发者可以自由选择Spring部分或全部

## SpringFramework没有解决了什么问题？

虽然Spring的组件代码是轻量级的，但它的配置却是重量级的。一开始，Spring用XML配置，而且是很多XML配置。Spring 2.5引入了基于注解的组件扫描，这消除了大量针对应用程序自身组件的显式XML配置。Spring 3.0引入了基于Java的配置，这是一种类型安全的可重构配置方式，可以代替XML。

所有这些配置都代表了开发时的损耗。因为在思考Spring特性配置和解决业务问题之间需要进行思维切换，所以编写配置挤占了编写应用程序逻辑的时间。和所有框架一样，Spring实用，但与此同时它要求的回报也不少。

除此之外，项目的依赖管理也是一件耗时耗力的事情。在环境搭建时，需要分析要导入哪些库的坐标，而且还需要分析导入与之有依赖关系的其他库的坐标，一旦选错了依赖的版本，随之而来的不兼容问题就会严重阻碍项目的开发进度。

## SpringBoot解决上述Spring的缺点

SpringBoot对上述Spring的缺点进行的改善和优化，基于约定优于配置的思想，可以让开发人员不必在配置与逻辑业务之间进行思维的切换，全身心的投入到逻辑业务的代码编写中，从而大大提高了开发的效率，一定程度上缩短了项目周期。

在使用Spring框架进行开发的过程中，需要配置很多Spring框架包的依赖，如spring-core、spring-bean、spring-context等，而这些配置通常都是重复添加的，而且需要做很多框架使用及环境参数的重复配置，如开启注解、配置日志等。Spring Boot致力于弱化这些不必要的操作，提供默认配置，当然这些默认配置是可以按需修改的，快速搭建、开发和运行Spring应用

# 和SpringMVC的关系

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230415111153.png)

Spring Boot：Spring Boot使快速引导和开始开发基于Spring的应用程序变得容易。 它避免了很多样板代码。 它在幕后隐藏了很多复杂性，因此开发人员可以快速上手并轻松开发基于Spring的应用程序。

Spring MVC：Spring MVC是用于构建Web应用程序的Web MVC框架。 它包含许多用于各种功能的配置文件。 这是一个面向HTTP的Web应用程序开发框架。



Spring Boot和Spring MVC出于不同的目的而存在。 下面讨论了Spring Boot和Spring MVC之间的主要区别：



| Spring Boot | Spring MVC |
| --- | --- |
| Spring Boot用于使用合理的默认值打包基于Spring的应用程序。 | Spring MVC是Spring框架下基于模型视图控制器的Web框架。 |
| 它提供了默认配置来构建Spring支持的框架。 | 它提供了用于构建Web应用程序的即用型功能。 |
| 无需手动构建配置。 | 它需要手动进行构建配置。 |
| 不需要部署描述符。 | 部署描述符是必需的。 |
| 它避免了样板代码，并将依赖项包装在一个单元中。 | 它分别指定每个依赖项。 |
| 它减少了开发时间并提高了生产率。 | 实现相同目的需要更多时间。 |

# 和微服务、SpringCloud的关系

Spring Boot是Spring家族的成员，它是一个全新的框架，它的设计目的是尽可能简单和快速的开发、运行Spring应用程序，简化配置。它为开发者快捷的使用Spring及相关开发框架提供了便利，但是它并不是微服务的框架，它只是为微服务框架的使用也提供了很好的脚手架。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230415111929.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230415111950.png)
上面两张图表明了两者的基本区别，springboot用于构建单个服务应用，也就是我们常说的微服务

而springcloud用于服务治理，也就是串联、管理和协调多个微服务（springboot应用程序）

springcloud中包含了众多服务治理相关的组件：服务API网关Springcloud Gateway，配置中心组件Config Serever，断路器Circuit Breaker、服务注册中心Service Registty，监控组件Sleuth等

# 使用Springboot的8个原因

## 更快的开发

Spring Boot 在 Spring 生态系统上做出了很多决定和固执己见的默认设置。 这种性质有助于开发人员快速设置并投入开发。

例如，Spring MVC 可以通过大量 XML bean 定义和自定义 servlet 类实现。 但是使用 Spring Boot，它就像添加启动器依赖项一样简单。 绝对不需要代码生成 XML 配置。

## 一切皆有先机

Spring Boot Starters 是包含库和它们的一些自动配置的 Maven 描述符。 而且，这些启动器有助于为 Spring Boot 应用程序提供功能。 想要建立数据库连接？ 有一个启动器依赖项。 想与消息队列通话或发送电子邮件？ Spring Boot 涵盖了这一切。

对于几乎所有的 Spring 模块，都有一个启动器依赖项可以为您配置所有内容。 甚至一些第三方库也通过他们的启动模块提供对 Spring 的支持。 如果没有这些启动器，您作为开发人员将不得不维护依赖项和 XML 配置。 这是您应该使用 Spring Boot 的另一个原因。

## 嵌入式服务器

Spring Boot 为嵌入式 Tomcat、Jetty 和 Undertow 服务器提供开箱即用的支持。 这样，开发人员就不必担心在传统应用服务器中部署 Web 应用程序。 通过适当的启动器依赖项，您甚至可以将一种服务器技术与其他服务器技术交换。 所以你实际上最终得到了一个可以像任何 JAR 一样运行的 JAR 文件。 在启动时，JAR 包含足够的库和配置以作为应用程序服务器启动并侦听请求。

如果您对嵌入式服务器不感兴趣，您可以随时将 Spring Boot 应用程序从 JAR 转换为 WAR，并将它们部署到传统服务器。

## IDE 对 Spring Boot 的支持

所有主要的 IDE 都提供对 Spring Boot 代码帮助的支持。 例如，IntelliJ IDEA Ultimate 为 Spring Boot 项目提供了出色的代码完成和导航功能。除此之外，VSCode 和 Eclipse 也对其提供了丰富的功能支持。

## 生产环境常用功能

Spring Boot 提供了生产就绪的特性，例如监控、指标和开箱即用的注销。 有了这些特性，开发人员可以避免额外的配置。 例如，健康执行器端点等功能使应用程序状态监控成为可能。 例如，

您可以让像 Prometheus 这样的工具收集应用程序指标

在您的 Kubernetes 或 Openshift 环境中使用就绪性和活跃度健康端点。

只需添加其他属性或通过 /actuator/logging 端点即可更改日志记录级别。

此外，开发人员可以使用自己的自定义健康端点配置这些执行器端点。

## 开箱即用的 JUnit 支持

默认情况下，所有 Spring Boot 项目都带有 JUnit 5。 此外，Spring Boot 提供了@SpringBootTest 注解来在我们需要时初始化测试上下文。 所以开发人员只需要编写测试用例。 他们不必再担心测试用例的复杂 spring 上下文。

例如，下面自动生成的测试将检查上下文是否正确加载。

````
@SpringBootTest
class SpringBootDerbyDatabaseApplicationTests {

     @测试
     void contextLoads() {
     }

}
````

## Spring Profiles

Spring Profiles 是 spring Boot 的一个强大特性，有助于隔离应用程序中的不同组件。 使用配置文件，您可以在特定环境中启用或禁用组件。 当您必须根据特定条件使用不同的组件时，这可能会派上用场。
````
@Profile(value = {"prod","uat"})
class RabbitMQConfig {

// 听众

}
````

在上面的代码中，上下文将限制兔子侦听器仅在具有 prod 或 uat 作为活动配置文件的环境中运行。

## 多种打包和部署选项

该框架提供了多种方式来打包您的应用程序。 正如我们之前所说，应用程序可以是 JAR 或 WAR 文件。 通过一些额外的配置和参数，您还可以创建开箱即用的高性能 docker 镜像。

启动和停止 Spring Boot 应用程序非常简单。 此外，您可以通过几个额外的步骤将这些 JAR 文件部署为 linux 服务。 JAR 文件称为 FAT jar，它们包含与应用程序相关的所有依赖项。 这使得部署过程不那么复杂。 实际上，这些构建可以在任何装有 Java 8 或更高版本的机器上运行。

# 参考文章

[https://spring.io/](https://spring.io/)

[https://pdai.tech/md/spring/springboot/springboot-x-overview.html](https://pdai.tech/md/spring/springboot/springboot-x-overview.html)

[https://springhow.com/why-use-spring-boot/](https://springhow.com/why-use-spring-boot/)

[https://dzone.com/articles/why-springboot](https://dzone.com/articles/why-springboot)

[https://scand.com/company/blog/pros-and-cons-of-using-spring-boot/](https://scand.com/company/blog/pros-and-cons-of-using-spring-boot/)

[https://cloud.tencent.com/developer/article/1620255](https://cloud.tencent.com/developer/article/1620255)

[https://www.yiibai.com/spring-boot/spring-vs-spring-boot-vs-spring-mvc.html](https://www.yiibai.com/spring-boot/spring-vs-spring-boot-vs-spring-mvc.html)