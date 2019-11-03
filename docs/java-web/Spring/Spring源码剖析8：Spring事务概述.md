原文出处： [张开涛](http://sishuok.com/forum/blogPost/list/0/2508.html)

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

## 数据库事务概述

事务首先是一系列操作组成的工作单元，该工作单元内的操作是不可分割的，即要么所有操作都做，要么所有操作都不做，这就是事务。

事务必需满足ACID（原子性、一致性、隔离性和持久性）特性，缺一不可：

*   原子性（Atomicity）：即事务是不可分割的最小工作单元，事务内的操作要么全做，要么全不做；
*   一致性（Consistency）：在事务执行前数据库的数据处于正确的状态，而事务执行完成后数据库的数据还是处于正确的状态，即数据完整性约束没有被破坏；如银行转帐，A转帐给B，必须保证A的钱一定转给B，一定不会出现A的钱转了但B没收到，否则数据库的数据就处于不一致（不正确）的状态。
*   隔离性（Isolation）：并发事务执行之间无影响，在一个事务内部的操作对其他事务是不产生影响，这需要事务隔离级别来指定隔离性；
*   持久性（Durability）：事务一旦执行成功，它对数据库的数据的改变必须是永久的，不会因比如遇到系统故障或断电造成数据不一致或丢失。

在实际项目开发中数据库操作一般都是并发执行的，即有多个事务并发执行，并发执行就可能遇到问题，目前常见的问题如下：

*   丢失更新：两个事务同时更新一行数据，最后一个事务的更新会覆盖掉第一个事务的更新，从而导致第一个事务更新的数据丢失，这是由于没有加锁造成的；
*   脏读：一个事务看到了另一个事务未提交的更新数据；
*   不可重复读：在同一事务中，多次读取同一数据却返回不同的结果；也就是有其他事务更改了这些数据；
*   幻读：一个事务在执行过程中读取到了另一个事务已提交的插入数据；即在第一个事务开始时读取到一批数据，但此后另一个事务又插入了新数据并提交，此时第一个事务又读取这批数据但发现多了一条，即好像发生幻觉一样。

为了解决这些并发问题，需要通过数据库隔离级别来解决，在标准SQL规范中定义了四种隔离级别：

*   未提交读（Read Uncommitted）：最低隔离级别，一个事务能读取到别的事务未提交的更新数据，很不安全，可能出现丢失更新、脏读、不可重复读、幻读；
*   提交读（Read Committed）：一个事务能读取到别的事务提交的更新数据，不能看到未提交的更新数据，不可能可能出现丢失更新、脏读，但可能出现不可重复读、幻读；
*   可重复读（Repeatable Read）：保证同一事务中先后执行的多次查询将返回同一结果，不受其他事务影响，可能可能出现丢失更新、脏读、不可重复读，但可能出现幻读；
*   序列化（Serializable）：最高隔离级别，不允许事务并发执行，而必须串行化执行，最安全，不可能出现更新、脏读、不可重复读、幻读。

隔离级别越高，数据库事务并发执行性能越差，能处理的操作越少。因此在实际项目开发中为了考虑并发性能一般使用提交读隔离级别，它能避免丢失更新和脏读，尽管不可重复读和幻读不能避免，但可以在可能出现的场合使用悲观锁或乐观锁来解决这些问题。

### 事务类型

数据库事务类型有本地事务和分布式事务：

*   本地事务：就是普通事务，能保证单台数据库上的操作的ACID，被限定在一台数据库上；
*   分布式事务：涉及两个或多个数据库源的事务，即跨越多台同类或异类数据库的事务（由每台数据库的本地事务组成的），分布式事务旨在保证这些本地事务的所有操作的ACID，使事务可以跨越多台数据库；

Java事务类型有JDBC事务和JTA事务：

*   JDBC事务：就是数据库事务类型中的本地事务，通过Connection对象的控制来管理事务；
*   JTA事务：JTA指Java事务API(Java Transaction API)，是Java EE数据库事务规范， JTA只提供了事务管理接口，由应用程序服务器厂商（如WebSphere Application Server）提供实现，JTA事务比JDBC更强大，支持分布式事务。

Java EE事务类型有本地事务和全局事务：

*   本地事务：使用JDBC编程实现事务；
*   全局事务：由应用程序服务器提供，使用JTA事务；

按是否通过编程实现事务有声明式事务和编程式事务；

*   声明式事务： 通过注解或XML配置文件指定事务信息；
*   编程式事务：通过编写代码实现事务。

### Spring提供的事务管理

Spring框架最核心功能之一就是事务管理，而且提供一致的事务管理抽象，这能帮助我们：

*   提供一致的编程式事务管理API，不管使用Spring JDBC框架还是集成第三方框架使用该API进行事务编程；
*   无侵入式的声明式事务支持。

Spring支持声明式事务和编程式事务事务类型。

### spring事务特性

spring所有的事务管理策略类都继承自org.springframework.transaction.PlatformTransactionManager接口

其中TransactionDefinition接口定义以下特性：

## 事务隔离级别

  隔离级别是指若干个并发的事务之间的隔离程度。TransactionDefinition 接口中定义了五个表示隔离级别的常量：

*   TransactionDefinition.ISOLATION_DEFAULT：这是默认值，表示使用底层数据库的默认隔离级别。对大部分数据库而言，通常这值就是TransactionDefinition.ISOLATION_READ_COMMITTED。
*   TransactionDefinition.ISOLATION_READ_UNCOMMITTED：该隔离级别表示一个事务可以读取另一个事务修改但还没有提交的数据。该级别不能防止脏读，不可重复读和幻读，因此很少使用该隔离级别。比如PostgreSQL实际上并没有此级别。
*   TransactionDefinition.ISOLATION_READ_COMMITTED：该隔离级别表示一个事务只能读取另一个事务已经提交的数据。该级别可以防止脏读，这也是大多数情况下的推荐值。
*   TransactionDefinition.ISOLATION_REPEATABLE_READ：该隔离级别表示一个事务在整个过程中可以多次重复执行某个查询，并且每次返回的记录都相同。该级别可以防止脏读和不可重复读。
*   TransactionDefinition.ISOLATION_SERIALIZABLE：所有的事务依次逐个执行，这样事务之间就完全不可能产生干扰，也就是说，该级别可以防止脏读、不可重复读以及幻读。但是这将严重影响程序的性能。通常情况下也不会用到该级别。

## 事务传播行为

      所谓事务的传播行为是指，如果在开始当前事务之前，一个事务上下文已经存在，此时有若干选项可以指定一个事务性方法的执行行为。在TransactionDefinition定义中包括了如下几个表示传播行为的常量：

*   TransactionDefinition.PROPAGATION_REQUIRED：如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。这是默认值。
*   TransactionDefinition.PROPAGATION_REQUIRES_NEW：创建一个新的事务，如果当前存在事务，则把当前事务挂起。
*   TransactionDefinition.PROPAGATION_SUPPORTS：如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
*   TransactionDefinition.PROPAGATION_NOT_SUPPORTED：以非事务方式运行，如果当前存在事务，则把当前事务挂起。
*   TransactionDefinition.PROPAGATION_NEVER：以非事务方式运行，如果当前存在事务，则抛出异常。
*   TransactionDefinition.PROPAGATION_MANDATORY：如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。
*   TransactionDefinition.PROPAGATION_NESTED：如果当前存在事务，则创建一个事务作为当前事务的嵌套事务来运行；如果当前没有事务，则该取值等价于TransactionDefinition.PROPAGATION_REQUIRED。

### 事务超时

      所谓事务超时，就是指一个事务所允许执行的最长时间，如果超过该时间限制但事务还没有完成，则自动回滚事务。在 TransactionDefinition 中以 int 的值来表示超时时间，其单位是秒。

  默认设置为底层事务系统的超时值，如果底层数据库事务系统没有设置超时值，那么就是none，没有超时限制。

### 事务只读属性

      只读事务用于客户代码只读但不修改数据的情形，只读事务用于特定情景下的优化，比如使用Hibernate的时候。

默认为读写事务。

### 概述

Spring框架支持事务管理的核心是事务管理器抽象，对于不同的数据访问框架（如Hibernate）通过实现策略接口PlatformTransactionManager，从而能支持各种数据访问框架的事务管理，PlatformTransactionManager接口定义如下：

java代码：

```

public interface PlatformTransactionManager {
       TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
       void commit(TransactionStatus status) throws TransactionException;
       void rollback(TransactionStatus status) throws TransactionException;
}

```

*   getTransaction()：返回一个已经激活的事务或创建一个新的事务（根据给定的TransactionDefinition类型参数定义的事务属性），返回的是TransactionStatus对象代表了当前事务的状态，其中该方法抛出TransactionException（未检查异常）表示事务由于某种原因失败。
*   commit()：用于提交TransactionStatus参数代表的事务，具体语义请参考Spring Javadoc；
*   rollback()：用于回滚TransactionStatus参数代表的事务，具体语义请参考Spring Javadoc。

TransactionDefinition接口定义如下：

java代码：

```

public interface TransactionDefinition {
       int getPropagationBehavior();
       int getIsolationLevel();
       int getTimeout();
       boolean isReadOnly();
       String getName();
}

```

*   getPropagationBehavior()：返回定义的事务传播行为；
*   getIsolationLevel()：返回定义的事务隔离级别；
*   getTimeout()：返回定义的事务超时时间；
*   isReadOnly()：返回定义的事务是否是只读的；
*   getName()：返回定义的事务名字。

TransactionStatus接口定义如下：

java代码：

```
public interface TransactionStatus extends SavepointManager {
       boolean isNewTransaction();
       boolean hasSavepoint();
       void setRollbackOnly();
       boolean isRollbackOnly();
       void flush();
       boolean isCompleted();
}

```

*   isNewTransaction()：返回当前事务状态是否是新事务；
*   hasSavepoint()：返回当前事务是否有保存点；
*   setRollbackOnly()：设置当前事务应该回滚；
*   isRollbackOnly(()：返回当前事务是否应该回滚；
*   flush()：用于刷新底层会话中的修改到数据库，一般用于刷新如Hibernate/JPA的会话，可能对如JDBC类型的事务无任何影响；
*   isCompleted():当前事务否已经完成。

### 内置事务管理器实现

Spring提供了许多内置事务管理器实现：

*   DataSourceTransactionManager：位于org.springframework.jdbc.datasource包中，数据源事务管理器，提供对单个javax.sql.DataSource事务管理，用于Spring JDBC抽象框架、iBATIS或MyBatis框架的事务管理；
*   JdoTransactionManager：位于org.springframework.orm.jdo包中，提供对单个javax.jdo.PersistenceManagerFactory事务管理，用于集成JDO框架时的事务管理；
*   JpaTransactionManager：位于org.springframework.orm.jpa包中，提供对单个javax.persistence.EntityManagerFactory事务支持，用于集成JPA实现框架时的事务管理；
*   HibernateTransactionManager：位于org.springframework.orm.hibernate3包中，提供对单个org.hibernate.SessionFactory事务支持，用于集成Hibernate框架时的事务管理；该事务管理器只支持Hibernate3+版本，且Spring3.0+版本只支持Hibernate 3.2+版本；
*   JtaTransactionManager：位于org.springframework.transaction.jta包中，提供对分布式事务管理的支持，并将事务管理委托给Java EE应用服务器事务管理器；
*   OC4JjtaTransactionManager：位于org.springframework.transaction.jta包中，Spring提供的对OC4J10.1.3+应用服务器事务管理器的适配器，此适配器用于对应用服务器提供的高级事务的支持；
*   WebSphereUowTransactionManager：位于org.springframework.transaction.jta包中，Spring提供的对WebSphere 6.0+应用服务器事务管理器的适配器，此适配器用于对应用服务器提供的高级事务的支持；
*   WebLogicJtaTransactionManager：位于org.springframework.transaction.jta包中，Spring提供的对WebLogic 8.1+应用服务器事务管理器的适配器，此适配器用于对应用服务器提供的高级事务的支持。

Spring不仅提供这些事务管理器，还提供对如JMS事务管理的管理器等，Spring提供一致的事务抽象如图9-1所示。
![](https://segmentfault.com/img/remote/1460000020178870?w=816&h=435)

图9-1 Spring事务管理器

接下来让我们学习一下如何在Spring配置文件中定义事务管理器：

一、声明对本地事务的支持：

a)JDBC及iBATIS、MyBatis框架事务管理器

java代码：

```

<bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>

```

通过dataSource属性指定需要事务管理的单个javax.sql.DataSource对象。

b)Jdo事务管理器

java代码：

```

<bean id="txManager" class="org.springframework.orm.jdo.JdoTransactionManager">
    <property name="persistenceManagerFactory" ref="persistenceManagerFactory"/>
</bean>

```

通过persistenceManagerFactory属性指定需要事务管理的javax.jdo.PersistenceManagerFactory对象。

c)Jpa事务管理器

java代码：

```
bean id="txManager" class="org.springframework.orm.jpa.JpaTransactionManager">
    <property name="entityManagerFactory" ref="entityManagerFactory"/>
</bean>

```

通过entityManagerFactory属性指定需要事务管理的javax.persistence.EntityManagerFactory对象。

还需要为entityManagerFactory对象指定jpaDialect属性，该属性所对应的对象指定了如何获取连接对象、开启事务、关闭事务等事务管理相关的行为。

java代码：

```
<bean id="entityManagerFactory" class="org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean">
        ……
        <property name="jpaDialect" ref="jpaDialect"/>
</bean>
<bean id="jpaDialect" class="org.springframework.orm.jpa.vendor.HibernateJpaDialect"/>

```

d)Hibernate事务管理器

java代码：

```
<bean id="txManager" class="org.springframework.orm.hibernate3.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory"/>
</bean>

```

通过entityManagerFactory属性指定需要事务管理的org.hibernate.SessionFactory对象。

## 声明式事务

### 声明式事务概述

从上节编程式实现事务管理可以深刻体会到编程式事务的痛苦，即使通过代理配置方式也是不小的工作量。

本节将介绍声明式事务支持，使用该方式后最大的获益是简单，事务管理不再是令人痛苦的，而且此方式属于无侵入式，对业务逻辑实现无影响。

接下来先来看看声明式事务如何实现吧。

### 声明式实现事务管理

1、定义业务逻辑实现，此处使用ConfigUserServiceImpl和ConfigAddressServiceImpl：

2、定义配置文件（chapter9/service/ applicationContext-service-declare.xml）：

2.1、XML命名空间定义，定义用于事务支持的tx命名空间和AOP支持的aop命名空间：

```
java代码：
<beans xmlns="http://www.springframework.org/schema/beans"
      xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:tx="http://www.springframework.org/schema/tx"
      xmlns:aop="http://www.springframework.org/schema/aop"
      xsi:schemaLocation="

http://www.springframework.org/schema/beans

http://www.springframework.org/schema/beans/spring-beans-3.0.xsd

http://www.springframework.org/schema/tx

http://www.springframework.org/schema/tx/spring-tx-3.0.xsd

http://www.springframework.org/schema/aop

http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
```

2.2、业务实现配置，非常简单，使用以前定义的非侵入式业务实现：

```
java代码：
<bean id="userService" class="cn.javass.spring.chapter9.service.impl.ConfigUserServiceImpl">
    <property name="userDao" ref="userDao"/>
    <property name="addressService" ref="addressService"/>
</bean>
<bean id="addressService" class="cn.javass.spring.chapter9.service.impl.ConfigAddressServiceImpl">
    <property name="addressDao" ref="addressDao"/>
</bean>
```

2.3、事务相关配置：

```
java代码：
<tx:advice id="txAdvice" transaction-manager="txManager">
    <tx:attributes>
        <tx:method name="save*" propagation="REQUIRED" isolation="READ_COMMITTED"/>
        <tx:method name="*" propagation="REQUIRED" isolation="READ_COMMITTED" read-only="true"/>
    </tx:attributes>
</tx:advice>
```

java代码：

```

<tx:advice>：事务通知定义，用于指定事务属性，其中“transaction-manager”属性指定事务管理器，并通过< tx:attributes >指定具体需要拦截的方法；
 <tx:method name=”save*”>：表示将拦截以save开头的方法，被拦截的方法将应用配置的事务属性：propagation=”REQUIRED”表示传播行为是Required，isolation=”READ_COMMITTED”表示隔离级别是提交读；
<tx:method name=”*”>：表示将拦截其他所有方法，被拦截的方法将应用配置的事务属性：propagation=”REQUIRED”表示传播行为是Required，isolation=”READ_COMMITTED”表示隔离级别是提交读，read-only=”true”表示事务只读；
：AOP相关配置：
：切入点定义，定义名为”serviceMethod”的aspectj切入点，切入点表达式为”execution(* cn..chapter9.service..*.*(..))”表示拦截cn包及子包下的chapter9\. service包及子包下的任何类的任何方法；
：Advisor定义，其中切入点为serviceMethod，通知为txAdvice。
从配置中可以看出，将对cn包及子包下的chapter9\. service包及子包下的任何类的任何方法应用“txAdvice”通知指定的事务属性。

```

3、修改测试方法并测试该配置方式是否好用：

将TransactionTest 类的testServiceTransaction测试方法拷贝一份命名为testDeclareTransaction：

并在testDeclareTransaction测试方法内将：

4、执行测试，测试正常通过，说明该方式能正常工作，当调用save方法时将匹配到事务通知中定义的“<tx:method name=”save_”>”中指定的事务属性，而调用countAll方法时将匹配到事务通知中定义的“<tx:method name=”_”>”中指定的事务属性。

声明式事务是如何实现事务管理的呢？还记不记得TransactionProxyFactoryBean实现配置式事务管理，配置式事务管理是通过代理方式实现，而声明式事务管理同样是通过AOP代理方式实现。

声明式事务通过AOP代理方式实现事务管理，利用环绕通知TransactionInterceptor实现事务的开启及关闭，而TransactionProxyFactoryBean内部也是通过该环绕通知实现的，因此可以认为是<tx:tags/>帮你定义了TransactionProxyFactoryBean，从而简化事务管理。

了解了实现方式后，接下来详细学习一下配置吧：

9.4.4 <tx:advice/>配置详解
声明式事务管理通过配置<tx:advice/>来定义事务属性，配置方式如下所示：

```
java代码：
<tx:advice id="……" transaction-manager="……">
<tx:attributes>
        <tx:method name="……"
                           propagation=" REQUIRED"
                           isolation="READ_COMMITTED"
                           timeout="-1"
                           read-only="false"
                           no-rollback-for=""
                           rollback-for=""/>
        ……
    </tx:attributes>
</tx:advice>
<tx:advice>：id用于指定此通知的名字， transaction-manager用于指定事务管理器，默认的事务管理器名字为“transactionManager”；
<tx:method>：用于定义事务属性即相关联的方法名；

```

name：定义与事务属性相关联的方法名，将对匹配的方法应用定义的事务属性，可以使用“_”通配符来匹配一组或所有方法，如“save_”将匹配以save开头的方法，而“*”将匹配所有方法；

propagation：事务传播行为定义，默认为“REQUIRED”，表示Required，其值可以通过TransactionDefinition的静态传播行为变量的“PROPAGATION_”后边部分指定，如“TransactionDefinition.PROPAGATION_REQUIRED”可以使用“REQUIRED”指定；

isolation：事务隔离级别定义；默认为“DEFAULT”，其值可以通过TransactionDefinition的静态隔离级别变量的“ISOLATION_”后边部分指定，如“TransactionDefinition. ISOLATION_DEFAULT”可以使用“DEFAULT”指定：

timeout：事务超时时间设置，单位为秒，默认-1，表示事务超时将依赖于底层事务系统；

read-only：事务只读设置，默认为false，表示不是只读；

rollback-for：需要触发回滚的异常定义，以“，”分割，默认任何RuntimeException 将导致事务回滚，而任何Checked Exception 将不导致事务回滚；异常名字定义和TransactionProxyFactoryBean中含义一样

no-rollback-for：不被触发进行回滚的 Exception(s)；以“，”分割；异常名字定义和TransactionProxyFactoryBean中含义一样；

记不记得在配置方式中为了解决“自我调用”而导致的不能设置正确的事务属性问题，使用“((IUserService)AopContext.currentProxy()).otherTransactionMethod()”方式解决，在声明式事务要得到支持需要使用来开启。

9.4.5 多事务语义配置及最佳实践
什么是多事务语义？说白了就是为不同的Bean配置不同的事务属性，因为我们项目中不可能就几个Bean，而可能很多，这可能需要为Bean分组，为不同组的Bean配置不同的事务语义。在Spring中，可以通过配置多切入点和多事务通知并通过不同方式组合使用即可。

```
   1、首先看下声明式事务配置的最佳实践吧：

<tx:advice id="txAdvice" transaction-manager="txManager">
<tx:attributes>
           <tx:method name="save*" propagation="REQUIRED" />
           <tx:method name="add*" propagation="REQUIRED" />
           <tx:method name="create*" propagation="REQUIRED" />
           <tx:method name="insert*" propagation="REQUIRED" />
           <tx:method name="update*" propagation="REQUIRED" />
           <tx:method name="merge*" propagation="REQUIRED" />
           <tx:method name="del*" propagation="REQUIRED" />
           <tx:method name="remove*" propagation="REQUIRED" />
           <tx:method name="put*" propagation="REQUIRED" />
           <tx:method name="get*" propagation="SUPPORTS" read-only="true" />
           <tx:method name="count*" propagation="SUPPORTS" read-only="true" />
          <tx:method name="find*" propagation="SUPPORTS" read-only="true" />
          <tx:method name="list*" propagation="SUPPORTS" read-only="true" />
          <tx:method name="*" propagation="SUPPORTS" read-only="true" />
       </tx:attributes>
</tx:advice>

       
       

```

该声明式事务配置可以应付常见的CRUD接口定义，并实现事务管理，我们只需修改切入点表达式来拦截我们的业务实现从而对其应用事务属性就可以了，如果还有更复杂的事务属性直接添加即可，即

如果我们有一个batchSaveOrUpdate方法需要“REQUIRES_NEW”事务传播行为，则直接添加如下配置即可：

java代码：
1
<tx:method name="batchSaveOrUpdate" propagation="REQUIRES_NEW" />
2、接下来看一下多事务语义配置吧，声明式事务最佳实践中已经配置了通用事务属性，因此可以针对需要其他事务属性的业务方法进行特例化配置：

```
java代码：
<tx:advice id="noTxAdvice" transaction-manager="txManager">
    <tx:attributes>
           <tx:method name="*" propagation="NEVER" />
    </tx:attributes>
</tx:advice>

       
       

```

该声明将对切入点匹配的方法所在事务应用“Never”传播行为。

多事务语义配置时，切入点一定不要叠加，否则将应用两次事务属性，造成不必要的错误及麻烦。

### @Transactional实现事务管理

对声明式事务管理，Spring提供基于@Transactional注解方式来实现，但需要Java 5+。

注解方式是最简单的事务配置方式，可以直接在Java源代码中声明事务属性，且对于每一个业务类或方法如果需要事务都必须使用此注解。

接下来学习一下注解事务的使用吧：

1、定义业务逻辑实现：

```
package cn.javass.spring.chapter9.service.impl;
//省略import
public class AnnotationUserServiceImpl implements IUserService {
    private IUserDao userDao;
    private IAddressService addressService;
    public void setUserDao(IUserDao userDao) {
        this.userDao = userDao;
    }
    public void setAddressService(IAddressService addressService) {
        this.addressService = addressService;
    }
    @Transactional(propagation=Propagation.REQUIRED, isolation=Isolation.READ_COMMITTED)
    @Override
    public void save(final UserModel user) {
        userDao.save(user);
        user.getAddress().setUserId(user.getId());
        addressService.save(user.getAddress());
    }
    @Transactional(propagation=Propagation.REQUIRED, isolation=Isolation.READ_COMMITTED, readOnly=true)
    @Override
    public int countAll() {
        return userDao.countAll();
    }
}
```

2、定义配置文件（chapter9/service/ applicationContext-service-annotation.xml）：

2.1、XML命名空间定义，定义用于事务支持的tx命名空间和AOP支持的aop命名空间：

java代码：

```
    <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns:tx="http://www.springframework.org/schema/tx"
          xmlns:aop="http://www.springframework.org/schema/aop"
          xsi:schemaLocation="

    http://www.springframework.org/schema/beans

    http://www.springframework.org/schema/beans/spring-beans-3.0.xsd

    http://www.springframework.org/schema/tx

    http://www.springframework.org/schema/tx/spring-tx-3.0.xsd

    http://www.springframework.org/schema/aop

    http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">
```

2.2、业务实现配置，非常简单，使用以前定义的非侵入式业务实现：

```
java代码：
<bean id="userService" class="cn.javass.spring.chapter9.service.impl.ConfigUserServiceImpl">
    <property name="userDao" ref="userDao"/>
    <property name="addressService" ref="addressService"/>
</bean>
<bean id="addressService" class="cn.javass.spring.chapter9.service.impl.ConfigAddressServiceImpl">
    <property name="addressDao" ref="addressDao"/>
</bean>
```

2.3、事务相关配置：

```

java代码：
1
<tx:annotation-driven transaction-manager="txManager"/>
使用如上配置已支持声明式事务。

3、修改测试方法并测试该配置方式是否好用：

将TransactionTest 类的testServiceTransaction测试方法拷贝一份命名为testAnntationTransactionTest：

classpath:chapter9/service/applicationContext-service-annotation.xml"

userService.save(user);
try {
    userService.save(user);
    Assert.fail();
} catch (RuntimeException e) {
}
Assert.assertEquals(0, userService.countAll());
Assert.assertEquals(0, addressService.countAll());
```

4、执行测试，测试正常通过，说明该方式能正常工作，因为在AnnotationAddressServiceImpl类的save方法中抛出异常，因此事务需要回滚，所以两个countAll操作都返回0。

9.4.7 @Transactional配置详解
Spring提供的<tx:annotation-driven/>用于开启对注解事务管理的支持，从而能识别Bean类上的@Transactional注解元数据，其具有以下属性：

transaction-manager：指定事务管理器名字，默认为transactionManager，当使用其他名字时需要明确指定；
proxy-target-class：表示将使用的代码机制，默认false表示使用JDK代理，如果为true将使用CGLIB代理
order：定义事务通知顺序，默认Ordered.LOWEST_PRECEDENCE，表示将顺序决定权交给AOP来处理。
Spring使用@Transactional 来指定事务属性，可以在接口、类或方法上指定，如果类和方法上都指定了@Transactional ，则方法上的事务属性被优先使用，具体属性如下：

value：指定事务管理器名字，默认使用<tx:annotation-driven/>指定的事务管理器，用于支持多事务管理器环境；
propagation：指定事务传播行为，默认为Required，使用Propagation.REQUIRED指定；
isolation：指定事务隔离级别，默认为“DEFAULT”，使用Isolation.DEFAULT指定；
readOnly：指定事务是否只读，默认false表示事务非只读；
timeout：指定事务超时时间，以秒为单位，默认-1表示事务超时将依赖于底层事务系统；
rollbackFor：指定一组异常类，遇到该类异常将回滚事务；
rollbackForClassname：指定一组异常类名字，其含义与<tx:method>中的rollback-for属性语义完全一样；
noRollbackFor：指定一组异常类，即使遇到该类异常也将提交事务，即不回滚事务；
noRollbackForClassname：指定一组异常类名字，其含义与<tx:method>中的no-rollback-for属性语义完全一样；
Spring提供的@Transactional 注解事务管理内部同样利用环绕通知TransactionInterceptor实现事务的开启及关闭。

使用@Transactional注解事务管理需要特别注意以下几点：

如果在接口、实现类或方法上都指定了@Transactional 注解，则优先级顺序为方法>实现类>接口；
建议只在实现类或实现类的方法上使用@Transactional，而不要在接口上使用，这是因为如果使用JDK代理机制是没问题，因为其使用基于接口的代理；而使用使用CGLIB代理机制时就会遇到问题，因为其使用基于类的代理而不是接口，这是因为接口上的@Transactional注解是“不能继承的”；

```
具体请参考基于JDK动态代理和CGLIB动态代理的实现Spring注解管理事务（@Trasactional）到底有什么区别。
```

在Spring代理机制下(不管是JDK动态代理还是CGLIB代理)，“自我调用”同样不会应用相应的事务属性，其语义和<tx:tags>中一样；


默认只对RuntimeException异常回滚；

在使用Spring代理时，默认只有在public可见度的方法的@Transactional 注解才是有效的，其它可见度（protected、private、包可见）的方法上即使有@Transactional
注解也不会应用这些事务属性的，Spring也不会报错，如果你非要使用非公共方法注解事务管理的话，可考虑使用AspectJ。