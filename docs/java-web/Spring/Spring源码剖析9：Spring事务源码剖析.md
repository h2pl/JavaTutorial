# Table of Contents

  * [声明式事务使用](#声明式事务使用)
  * [TxNamespaceHandler](#txnamespacehandler)
  * [注册事务功能bean](#注册事务功能bean)
  * [使用bean的后处理方法获取增强器](#使用bean的后处理方法获取增强器)
  * [Spring获取匹配的增强器](#spring获取匹配的增强器)
  * [Transactional注解](#transactional注解)
  * [开启事务过程](#开启事务过程)


转自：http://www.linkedkeeper.com/detail/blog.action?bid=1045

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
## 声明式事务使用

Spring事务是我们日常工作中经常使用的一项技术，Spring提供了编程、注解、aop切面三种方式供我们使用Spring事务，其中编程式事务因为对代码入侵较大所以不被推荐使用，注解和aop切面的方式可以基于需求自行选择，我们以注解的方式为例来分析Spring事务的原理和源码实现。

首先我们简单看一下Spring事务的使用方式，配置：

```
<tx:annotation-driven transaction-manager="transactionManager"/>
 <bean id="transactionManager" 
         class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
     <property name="dataSource" ref="dataSource"/>
 </bean>
```

在需要开启事务的方法上加上@Transactional注解即可，这里需要注意的是，当<tx:annotation-driven>标签在不指定transaction-manager属性的时候，会默认寻找id固定名为transactionManager的bean作为事务管理器，如果没有id为transactionManager的bean并且在使用@Transactional注解时也没有指定value（事务管理器），程序就会报错。当我们在配置两个以上的<tx:annotation-driven>标签时，如下：</tx:annotation-driven></tx:annotation-driven>

```
    <tx:annotation-driven transaction-manager="transactionManager1"/>
<bean id="transactionManager1" 
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource1"/>
</bean>
<tx:annotation-driven transaction-manager="transactionManager2"/>
<bean id="transactionManager2" 
        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource2"/>
</bean>
```

这时第一个<tx:annotation-driven>会生效，也就是当我们使用@Transactional注解时不指定事务管理器，默认使用的事务管理器是transactionManager1，后文分析源码时会具体提到这些注意点。</tx:annotation-driven>

下面我们开始分析Spring的相关源码，首先看一下对<tx:annotation-driven>标签的解析，这里需要读者对Spring自定义标签解析的过程有一定的了解，笔者后续也会出相关的文章。锁定TxNamespaceHandler：</tx:annotation-driven>

## TxNamespaceHandler

(右键可查看大图)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142033158-801757952.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142035853-1216561547.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142050711-1788774364.jpg)

## 注册事务功能bean

这个方法比较长，关键的部分做了标记，最外围的if判断限制了<tx:annotation-driven>标签只能被解析一次，所以只有第一次被解析的标签会生效。蓝色框的部分分别注册了三个BeanDefinition，分别为AnnotationTransactionAttributeSource、TransactionInterceptor和BeanFactoryTransactionAttributeSourceAdvisor，并将前两个BeanDefinition添加到第三个BeanDefinition的属性当中，这三个bean支撑了整个事务功能，后面会详细说明。我们先来看红色框的第个方法：</tx:annotation-driven>

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142052962-1382540518.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142056340-212555152.jpg)

还记得当<tx:annotation-driven>标签在不指定transaction-manager属性的时候，会默认寻找id固定名为transactionManager的bean作为事务管理器这个注意事项么，就是在这里实现的。下面我们来看红色框的第二个方法：</tx:annotation-driven>

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142058845-60551057.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142100099-521338949.jpg)

这两个方法的主要目的是注册InfrastructureAdvisorAutoProxyCreator，注册这个类的目的是什么呢？我们看下这个类的层次：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142103625-1672142980.jpg)

## 使用bean的后处理方法获取增强器

我们发现这个类间接实现了BeanPostProcessor接口，我们知道，Spring会保证所有bean在实例化的时候都会调用其postProcessAfterInitialization方法，我们可以使用这个方法包装和改变bean，而真正实现这个方法是在其父类AbstractAutoProxyCreator类中：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142105124-1636877804.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142112838-373935120.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142114588-713887840.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142118835-2064313605.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142119676-1959254739.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142131238-2064733556.jpg)

上面这个方法相信大家已经看出了它的目的，先找出所有对应Advisor的类的beanName，再通过beanFactory.getBean方法获取这些bean并返回。不知道大家还是否记得在文章开始的时候提到的三个类，其中BeanFactoryTransactionAttributeSourceAdvisor实现了Advisor接口，所以这个bean就会在此被提取出来，而另外两个bean被织入了BeanFactoryTransactionAttributeSourceAdvisor当中，所以也会一起被提取出来，下图为BeanFactoryTransactionAttributeSourceAdvisor类的层次：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142136219-556507874.jpg)

## Spring获取匹配的增强器

下面让我们来看Spring如何在所有候选的增强器中获取匹配的增强器：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142138356-655521572.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142145927-1806310557.jpg)

上面的方法中提到引介增强的概念，在此做简要说明，引介增强是一种比较特殊的增强类型，它不是在目标方法周围织入增强，而是为目标类创建新的方法和属性，所以引介增强的连接点是类级别的，而非方法级别的。通过引介增强，我们可以为目标类添加一个接口的实现，即原来目标类未实现某个接口，通过引介增强可以为目标类创建实现该接口的代理，使用方法可以参考文末的引用链接。另外这个方法用两个重载的canApply方法为目标类寻找匹配的增强器，其中第一个canApply方法会调用第二个canApply方法并将第三个参数传为false：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142150013-1860771847.jpg)

在上面BeanFactoryTransactionAttributeSourceAdvisor类的层次中我们看到它实现了PointcutAdvisor接口，所以会调用红框中的canApply方法进行判断，第一个参数pca.getPointcut()也就是调用BeanFactoryTransactionAttributeSourceAdvisor的getPointcut方法：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142156819-1296874034.jpg)

这里的transactionAttributeSource也就是我们在文章开始看到的为BeanFactoryTransactionAttributeSourceAdvisor织入的两个bean中的AnnotationTransactionAttributeSource，我们以TransactionAttributeSourcePointcut作为第一个参数继续跟踪canApply方法：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142204505-259300568.jpg)

我们跟踪pc.getMethodMatcher()方法也就是TransactionAttributeSourcePointcut的getMethodMatcher方法是在它的父类中实现：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142208118-661649181.jpg)

发现方法直接返回this，也就是下面methodMatcher.matches方法就是调用TransactionAttributeSourcePointcut的matches方法：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142210658-52846138.jpg)

在上面我们看到其实这个tas就是AnnotationTransactionAttributeSource，这里的目的其实也就是判断我们的业务方法或者类上是否有@Transactional注解，跟踪AnnotationTransactionAttributeSource的getTransactionAttribute方法：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142219647-1040649746.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142229849-1651021110.jpg)

方法中的事务声明优先级最高，如果方法上没有声明则在类上寻找：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142231850-1851923561.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142233762-127153561.jpg)

this.annotationParsers是在AnnotationTransactionAttributeSource类初始化的时候初始化的：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142238658-1305833682.jpg)

所以annotationParser.parseTransactionAnnotation就是调用SpringTransactionAnnotationParser的parseTransactionAnnotation方法：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142241626-1898573511.jpg)

至此，我们终于看到的Transactional注解，下面无疑就是解析注解当中声明的属性了：

## Transactional注解

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142250086-1593469718.jpg)

在这个方法中我们看到了在Transactional注解中声明的各种常用或者不常用的属性的解析，至此，事务的初始化工作算是完成了，下面开始真正的进入执行阶段。

在上文AbstractAutoProxyCreator类的wrapIfNecessary方法中，获取到目标bean匹配的增强器之后，会为bean创建代理，这部分内容我们会在Spring AOP的文章中进行详细说明，在此简要说明方便大家理解，在执行代理类的目标方法时，会调用Advisor的getAdvice获取MethodInterceptor并执行其invoke方法，而我们本文的主角BeanFactoryTransactionAttributeSourceAdvisor的getAdvice方法会返回我们在文章开始看到的为其织入的另外一个bean，也就是TransactionInterceptor，它实现了MethodInterceptor，所以我们分析其invoke方法：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142251964-1897702955.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142253742-81370531.jpg)

这个方法很长，但是整体逻辑还是非常清晰的，首选获取事务属性，这里的getTransactionAttrubuteSource()方法的返回值同样是在文章开始我们看到的被织入到TransactionInterceptor中的AnnotationTransactionAttributeSource，在事务准备阶段已经解析过事务属性并保存到缓存中，所以这里会直接从缓存中获取，接下来获取配置的TransactionManager，也就是determineTransactionManager方法，这里如果配置没有指定transaction-manager并且也没有默认id名为transactionManager的bean，就会报错，然后是针对声明式事务和编程式事务的不同处理，创建事务信息，执行目标方法，最后根据执行结果进行回滚或提交操作，我们先分析创建事务的过程。在分析之前希望大家能先去了解一下Spring的事务传播行为，有助于理解下面的源码，这里做一个简要的介绍，更详细的信息请大家自行查阅Spring官方文档，里面有更新详细的介绍。

Spring的事务传播行为定义在Propagation这个枚举类中，一共有七种，分别为：

REQUIRED：业务方法需要在一个容器里运行。如果方法运行时，已经处在一个事务中，那么加入到这个事务，否则自己新建一个新的事务，是默认的事务传播行为。

NOT_SUPPORTED：声明方法不需要事务。如果方法没有关联到一个事务，容器不会为他开启事务，如果方法在一个事务中被调用，该事务会被挂起，调用结束后，原先的事务会恢复执行。

REQUIRESNEW：不管是否存在事务，该方法总汇为自己发起一个新的事务。如果方法已经运行在一个事务中，则原有事务挂起，新的事务被创建。

MANDATORY：该方法只能在一个已经存在的事务中执行，业务方法不能发起自己的事务。如果在没有事务的环境下被调用，容器抛出例外。

SUPPORTS：该方法在某个事务范围内被调用，则方法成为该事务的一部分。如果方法在该事务范围外被调用，该方法就在没有事务的环境下执行。

NEVER：该方法绝对不能在事务范围内执行。如果在就抛例外。只有该方法没有关联到任何事务，才正常执行。

NESTED：如果一个活动的事务存在，则运行在一个嵌套的事务中。如果没有活动事务，则按REQUIRED属性执行。它使用了一个单独的事务，这个事务拥有多个可以回滚的保存点。内部事务的回滚不会对外部事务造成影响。它只对DataSourceTransactionManager事务管理器起效。

## 开启事务过程

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142254272-1473500584.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142256723-2146158353.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142257278-1878221643.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142257769-599399453.jpg)

判断当前线程是否存在事务就是判断记录的数据库连接是否为空并且transactionActive状态为true。

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142259232-1015967872.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142301922-198146239.jpg)

REQUIRESNEW会开启一个新事务并挂起原事务，当然开启一个新事务就需要一个新的数据库连接：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142303284-1121880076.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142303540-922946646.jpg)

suspend挂起操作主要目的是将当前connectionHolder置为null，保存原有事务信息，以便于后续恢复原有事务，并将当前正在进行的事务信息进行重置。下面我们看Spring如何开启一个新事务：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142304817-1631199195.jpg)

这里我们看到了数据库连接的获取，如果是新事务需要获取新一个新的数据库连接，并为其设置了隔离级别、是否只读等属性，下面就是将事务信息记录到当前线程中：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142305129-2058808100.jpg)

接下来就是记录事务状态并返回事务信息：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142306070-647878384.jpg)

然后就是我们目标业务方法的执行了，根据执行结果的不同做提交或回滚操作，我们先看一下回滚操作：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142307229-1348831577.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142307451-1327327647.jpg)

其中回滚条件默认为RuntimeException或Error，我们也可以自行配置。

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142307619-1218769460.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142309139-1058770763.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142310021-1140102079.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142310191-1124633997.jpg)

保存点一般用于嵌入式事务，内嵌事务的回滚不会引起外部事务的回滚。下面我们来看新事务的回滚：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142310897-1920299450.jpg)

很简单，就是获取当前线程的数据库连接并调用其rollback方法进行回滚，使用的是底层数据库连接提供的API。最后还有一个清理和恢复挂起事务的操作：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142311558-94141948.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142312884-396652627.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142313851-102126465.jpg)

如果事务执行前有事务挂起，那么当前事务执行结束后需要将挂起的事务恢复，挂起事务时保存了原事务信息，重置了当前事务信息，所以恢复操作就是将当前的事务信息设置为之前保存的原事务信息。到这里事务的回滚操作就结束了，下面让我们来看事务的提交操作：

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142314316-220782134.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142315709-533683607.jpg)

在上文分析回滚流程中我们提到了如果当前事务不是独立的事务，也没有保存点，在回滚的时候只是设置一个回滚标记，由外部事务提交时统一进行整体事务的回滚。

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142316748-2061067152.jpg)

![](https://img2018.cnblogs.com/blog/1092007/201908/1092007-20190825142317597-660647393.jpg)

提交操作也是很简单的调用数据库连接底层API的commit方法。

参考链接：

http://blog.163.com/asd_wll/blog/static/2103104020124801348674/

https://docs.spring.io/spring/docs/current/spring-framework-reference/data-access.html#spring-data-tier
