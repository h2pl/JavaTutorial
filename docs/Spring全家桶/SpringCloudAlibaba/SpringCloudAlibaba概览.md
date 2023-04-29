Spring Cloud Alibaba致力于提供微服务开发的一站式解决方案，它是Spring Cloud组件被植入Alibaba元素之后的产物。利用Spring Cloud Alibaba，可以快速搭建微服务架构并完成技术升级。中小企业如果需要快速落地业务中台和技术中台，并向数字化业务转型，那Spring Cloud Alibaba绝对是一个“神器”。

# 什么是Spring Cloud Alibaba

好吧我们先看看Spring Cloud Alibaba的官网，如下图所示。

![image-20230423165959115](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423165959115.png)

从图中就可以看出，Spring Cloud Alibaba是Spring Cloud的子项目，好吧，这两项目一开始的定位就是父子关系。

看来官方还是没打算颠覆Spring Cloud的架构思想，只是想做一次能力的增强和扩展。


# 为什么要有Spring Cloud Alibaba


我们先看看Spring Cloud Alibaba有哪些能力？



Spring Cloud Alibaba 的组件功能既有免费版本，也有收费版本。

Sentinel：以“流”为切入点，在流量控制、并发性、容错降级和负载保护等方面提供解决方案，以保护服务的稳定性。



Nacos：一个具备动态服务发现和分布式配置等功能的管理平台，主要用于构建云原生应用程序。



RocketMQ：一个高性能、高可用、高吞吐量的金融级消息中间件。Spring Cloud Alibaba 将RocketMQ 定制化封装，开发人员可“开箱即用”。



Dubbo：一个基于Java的高性能开源RPC框架。



Seata：一个高性能且易于使用的分布式事务解决方案，可用于微服务架构。

阿里云OSS（阿里云对象存储服务）：一种加密的安全云存储服务，可以存储、处理和访问来自世界任何地方的大量数据。



阿里云SchedulerX：一款分布式任务调度产品，支持定期任务和在指定时间点触发任务。



阿里云SMS：一种覆盖全球的消息服务，提供便捷、高效和智能的通信功能，可帮助企业快速联系其客户。



我们再来看看Spring Cloud Alibaba的各个版本的对比，如下图所示。

![image-20230423170034102](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423170034102.png)

我们再来看看，没有Spring Cloud Alibaba之前，我们如何使用Spring Cloud进行微服务开发。



Spring Cloud支持多种注册中心，比如Eureka、ZooKeeper、Consul等。



如果软件开发人员需要采用Eureka作为注册中心，则需要搭建一个Eureka Server集群，用于管理注册中心服务的元数据，然后应用服务需要接入Eureka，就需要使用对应的注解将服务提供者以及服务订阅者注册到Eureka注册中心。同理ZooKeeper和Consul也是采用同样的方式，使用对应注册中心的注解完成服务的注册和订阅。



Spring Cloud为了方便软件开发人员快速的接入不同的注册中心，统一使用注解@EnableDiscoveryClient+对应注册中心的Starter组件。当然Eureka还是沿用老的使用方式@EnableEurekaClient+对应注册中心的Starter组件，主要是由于Spring Cloud已经停止了对Eureka的维护。



好吧问题来了，Spring Cloud已经将ZooKeeper和Consul的使用方式统一起来，软件开发人员非常愉快的将应用接入Spring Cloud，但是目前市面上又出了一个新的注册中心，比如Nacos，它的性能非常高，并且支持CP和AP模式，但是Spring Cloud不支持。



我们再来看看在没有Spring Cloud Alibaba之前，我们如何使用Nacos。



好吧，Nacos是一款既支持分布式注册中心和分布式配置中心的神器。Nacos官方提供了很多接入模式，比如Spring Framework、Spring Boot等，但是其底层本质上是依赖Nacos提供的SDK,比如Nacos Client。



如果采用Spring Framework+Nacos Client（比如nacos-spring-context），则需要开发人员自己维护NacosNamingServce和NacosConfigService实例对象，也就是说开发人员需要自己依赖Nacos Client做二次开发，成本非常大。



如果采用Spring Boot+Nacos Starter组件（比如nacos-discovery-spring-boot-starter），则开发人员可以高效的接入Nacos配置中心，并且可以使用Spring Boot提供的各种Starter组件。



如果纯碎采用Spring Cloud作为基础框架，则不能使用Nacos作为注册中心，好纠结啊。



那么有没有一个框架既可以使用Spring Cloud，又可以使用Spring Boot，还能兼容各种注册中心呢，很高兴的告诉大家，Spring Cloud Alibaba就是这个神器，完美的解决了开发人员微服务架构框架选型的问题，用它就是了。

# Spring Cloud Alibaba的核心架构思想


Spring Cloud Alibaba整体架构，如下图。

![image-20230423170108696](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423170108696.png)

Spring Cloud Alibaba架构的木目标是将开源产品和云厂商融合，做到业务云上和云下的无缝兼容，业务开发人员只需要关心业务项目开发，底层技术细节就交给Spring Cloud Alibaba。



如何快速上手并使用




好吧，说了那么多我们软件开发人员怎么快速上手Spring Cloud Alibaba呢？

其实最简单的方式是用问题驱动，然后自己亲自动手使用Spring Cloud Alibaba的功能，再去分析源码这样才能更加深刻的熟悉原理。



也许会有一些人会认为，我看了源码就算是我现在熟悉了，过了一个月之后就忘记了，有什么用呢。但是你这样想，就算是不看源码，你做其他的事情，一个月之后也会忘记的。只有通过理论和实战结合，你才会长期的保持习惯去练习，做到温故而知新。



咱们现在软件开发人员，尤其是Java开发人员，应该都用IDEA来做项目开发，我觉得就可以利用它来快速的生成一个Spring Cloud Alibaba项目，或者利用Spring Framework官方的脚手架项目Spring Initializr等。



还有就是要多总结一些最佳实践，比如要从Spring Cloud Alibaba的配置文件开始，反撸一些配置的业务场景，这样才能更加深刻的了解Spring Cloud Alibaba的设计思想。


Copyright ? 俊逸 Link: [https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html](https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html)

这块像官方文档一样，简单的介绍Spring Cloud Alibaba包含了那些主要部分，简单有个概念和印象，后面当然会详细的介绍，再去深入理解。

- **服务限流和降级**：支持`WebServlet`、`WebFlux`、`OpenFeign`、`RestTemplate`、`Dubbo`访问限流和降级，可以通过`console`实时修改限流降级策略，支持监控限流降级指标
- **服务注册和发现**：可以注册服务，clients可以通过Spring管理的bean发现实例，并融合了Ribbon
- **分布式配置**：支持分布式系统的配置扩展，配置改变时自动刷新
- **Rpc 服务**：扩展Spring Cloud的RestTemplate和OpenFeign支持调用Dubbo RPC服务
- **事件驱动**：支持构建通过共享消息系统连接的高度可伸缩的事件驱动的微服务
- **分布式事务**：支持高性能、容易使用的分布式事务解决方案
- 阿里云对象存储、阿里云任务调度、阿里云短信服务

上面基本上是把官方文档翻译过来的，可以发现Spring Cloud Alibaba非常强大，还不心动的往下看~



# 整体大纲

## ![springCloud-Alibaba-1](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springCloud-Alibaba-1.png)[#](#版本关系) 版本关系

[官方版本说明(opens new window)](https://github.com/alibaba/spring-cloud-alibaba/wiki/%E7%89%88%E6%9C%AC%E8%AF%B4%E6%98%8E)

### 2021.x 分支

| Spring Cloud Alibaba Version | Spring Cloud Version  | Spring Boot Version |
| ---------------------------- | --------------------- | ------------------- |
| 2021.0.1.0*                  | Spring Cloud 2021.0.1 | 2.6.3               |
| 2021.1                       | Spring Cloud 2020.0.1 | 2.4.2               |

### 2.2.x 分支

| Spring Cloud Alibaba Version      | Spring Cloud Version        | Spring Boot Version |
| --------------------------------- | --------------------------- | ------------------- |
| 2.2.8.RELEASE*                    | Spring Cloud Hoxton.SR12    | 2.3.12.RELEASE      |
| 2.2.7.RELEASE                     | Spring Cloud Hoxton.SR12    | 2.3.12.RELEASE      |
| 2.2.6.RELEASE                     | Spring Cloud Hoxton.SR9     | 2.3.2.RELEASE       |
| 2.1.4.RELEASE                     | Spring Cloud Greenwich.SR6  | 2.1.13.RELEASE      |
| 2.2.1.RELEASE                     | Spring Cloud Hoxton.SR3     | 2.2.5.RELEASE       |
| 2.2.0.RELEASE                     | Spring Cloud Hoxton.RELEASE | 2.2.X.RELEASE       |
| 2.1.2.RELEASE                     | Spring Cloud Greenwich      | 2.1.X.RELEASE       |
| 2.0.4.RELEASE(停止维护，建议升级) | Spring Cloud Finchley       | 2.0.X.RELEASE       |
| 1.5.1.RELEASE(停止维护，建议升级) | Spring Cloud Edgware        | 1.5.X.RELEASE       |

### 组件版本关系

| Spring Cloud Alibaba Version                              | Sentinel Version | Nacos Version | RocketMQ Version | Dubbo Version | Seata Version |
| --------------------------------------------------------- | ---------------- | ------------- | ---------------- | ------------- | ------------- |
| 2.2.8.RELEASE                                             | 1.8.4            | 2.1.0         | 4.9.3            | ~             | 1.5.1         |
| 2021.0.1.0                                                | 1.8.3            | 1.4.2         | 4.9.2            | ~             | 1.4.2         |
| 2.2.7.RELEASE                                             | 1.8.1            | 2.0.3         | 4.6.1            | 2.7.13        | 1.3.0         |
| 2.2.6.RELEASE                                             | 1.8.1            | 1.4.2         | 4.4.0            | 2.7.8         | 1.3.0         |
| 2021.1 or 2.2.5.RELEASE or 2.1.4.RELEASE or 2.0.4.RELEASE | 1.8.0            | 1.4.1         | 4.4.0            | 2.7.8         | 1.3.0         |
| 2.2.3.RELEASE or 2.1.3.RELEASE or 2.0.3.RELEASE           | 1.8.0            | 1.3.3         | 4.4.0            | 2.7.8         | 1.3.0         |
| 2.2.1.RELEASE or 2.1.2.RELEASE or 2.0.2.RELEASE           | 1.7.1            | 1.2.1         | 4.4.0            | 2.7.6         | 1.2.0         |
| 2.2.0.RELEASE                                             | 1.7.1            | 1.1.4         | 4.4.0            | 2.7.4.1       | 1.0.0         |
| 2.1.1.RELEASE or 2.0.1.RELEASE or 1.5.1.RELEASE           | 1.7.0            | 1.1.4         | 4.4.0            | 2.7.3         | 0.9.0         |
| 2.1.0.RELEASE or 2.0.0.RELEASE or 1.5.0.RELEASE           | 1.6.3            | 1.1.1         | 4.4.0            | 2.7.3         | 0.7.1         |

# 总结



本文就是从全局聊了一些Spring Cloud Alibaba的一些架构思想，但是没有从细节聊，具体细节可以参考“Spring Cloud Alibaba系列”。


# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud