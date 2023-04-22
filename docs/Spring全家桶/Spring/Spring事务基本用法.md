# Spring事务核心接口

[Spring](http://www.voidme.com/spring) 的事务管理是基于 AOP 实现的，而 AOP 是以方法为单位的。Spring 的事务属性分别为传播行为、隔离级别、只读和超时属性，这些属性提供了事务应用的方法和描述策略。

在 [Java](http://www.voidme.com/java) EE 开发经常采用的分层模式中，Spring 的事务处理位于业务逻辑层，它提供了针对事务的解决方案。

在 Spring 解压包的 libs 目录中，包含一个名称为 spring-tx-3.2.13.RELEASE.jar 的文件，该文件是 Spring 提供的用于事务管理的 JAR 包，其中包括事务管理的三个核心接口：PlatformTransactionManager、TransactionDefinition 和 TransactionStatus。

将该 JAR 包的后缀名 jar 改成 zip 的形式后，解压压缩包，进入解压文件夹中的 \org\springframework\transaction 目录后，该目录中的文件如图 1 所示。

![事务管理核心接口](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/760_a12_405.png)  
图 1  事务管理核心接口

在图 1 中，方框所标注的三个文件就是本节将要讲解的核心接口。这三个核心接口的作用及其提供的方法如下。

#### 1\. PlatformTransactionManager

PlatformTransactionManager 接口是 Spring 提供的平台事务管理器，用于管理事务。该接口中提供了三个事务操作方法，具体如下。

*   TransactionStatus getTransaction（TransactionDefinition definition）：用于获取事务状态信息。
*   void commit（TransactionStatus status）：用于提交事务。
*   void rollback（TransactionStatus status）：用于回滚事务。

在项目中，Spring 将 xml 中配置的事务详细信息封装到对象 TransactionDefinition 中，然后通过事务管理器的 getTransaction() 方法获得事务的状态（TransactionStatus），并对事务进行下一步的操作。

#### 2\. TransactionDefinition

TransactionDefinition 接口是事务定义（描述）的对象，它提供了事务相关信息获取的方法，其中包括五个操作，具体如下。

*   String getName()：获取事务对象名称。
*   int getIsolationLevel()：获取事务的隔离级别。
*   int getPropagationBehavior()：获取事务的传播行为。
*   int getTimeout()：获取事务的超时时间。
*   boolean isReadOnly()：获取事务是否只读。

在上述五个方法的描述中，事务的传播行为是指在同一个方法中，不同操作前后所使用的事务。传播行为的种类如表 1 所示。

| 属性名称 | 值 | 描  述 |  
| --- | --- | --- |  
| PROPAGATION_REQUIRED | required | 支持当前事务。如果 A 方法已经在事务中，则 B 事务将直接使用。否则将创建新事务 |  
| PROPAGATION_SUPPORTS | supports | 支持当前事务。如果 A 方法已经在事务中，则 B 事务将直接使用。否则将以非事务状态执行 |  
| PROPAGATION_MANDATORY | mandatory | 支持当前事务。如果 A 方法没有事务，则抛出异常 |  
| PROPAGATION_REQUIRES_NEW | requires_new | 将创建新的事务，如果 A 方法已经在事务中，则将 A 事务挂起 |  
| PROPAGATION_NOT_SUPPORTED | not_supported | 不支持当前事务，总是以非事务状态执行。如果 A 方法已经在事务中，则将其挂起 |  
| PROPAGATION_NEVER | never | 不支持当前事务，如果 A 方法在事务中，则抛出异常 |  
| PROPAGATION.NESTED | nested | 嵌套事务，底层将使用 Savepoint 形成嵌套事务 |  

在事务管理过程中，传播行为可以控制是否需要创建事务以及如何创建事务。

通常情况下，数据的查询不会改变原数据，所以不需要进行事务管理，而对于数据的增加、修改和删除等操作，必须进行事务管理。如果没有指定事务的传播行为，则 Spring3 默认的传播行为是 required。

#### 3\. TransactionStatus

TransactionStatus 接口是事务的状态，它描述了某一时间点上事务的状态信息。其中包含六个操作，具体如表 2 所示。

<caption>表 2  事务的操作</caption>  
| 名称 | 说明 |  
| --- | --- |  
| void flush() | 刷新事务 |  
| boolean hasSavepoint() | 获取是否存在保存点 |  
| boolean isCompleted() | 获取事务是否完成 |  
| boolean isNewTransaction() | 获取是否是新事务 |  
| boolean isRollbackOnly() | 获取是否回滚 |  
| void setRollbackOnly() | 设置事务回滚 |  

# Spring声明式事务管理（基于XML方式实现）

[Spring](http://www.voidme.com/spring) 的事务管理有两种方式：一种是传统的编程式事务管理，即通过编写代码实现的事务管理；另一种是基于 AOP 技术实现的声明式事务管理。由于在实际开发中，编程式事务管理很少使用，所以我们只对 Spring 的声明式事务管理进行详细讲解。

Spring 声明式事务管理在底层采用了 AOP 技术，其最大的优点在于无须通过编程的方式管理事务，只需要在配置文件中进行相关的规则声明，就可以将事务规则应用到业务逻辑中。

Spring 实现声明式事务管理主要有两种方式：

*   基于 XML 方式的声明式事务管理。
*   通过 Annotation 注解方式的事务管理。

本节通过银行转账的案例讲解如何使用 XML 的方式实现 Spring 的声明式事务处理。

#### 1\. 创建项目

在 MyEclipse 中创建一个名为 springDemo03 的 Web 项目，将 Spring 支持和依赖的 JAR 包复制到 Web 项目的 lib 目录中，并添加到类路径下。所添加的 JAR 包如图 1 所示。

![需要导入的JAR包](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/761_732_fcf.png)  
图 1  需要导入的JAR包

从图 1 中可以看出，这里增加导入了 spring-tx-3.2.13.RELEASE.jar（事务管理），以及 [MySQL](http://www.voidme.com/mysql) 驱动、JDBC 和 C3P0 的 JAR 包。

#### 2\. 创建数据库、表以及插入数据

在 MySQL 中创建一个名为 spring 的数据库，然后在该数据库中创建一个 account 表，并向表中插入两条数据，其 SQL 执行语句如下所示：

CREATE DATABASE spring;  
USE spring;  
CREATE TABLE account (  
id INT (11) PRIMARY KEY AUTO_INCREMENT,  
username VARCHAR(20) NOT NULL,  
money INT DEFAULT NULL  
);  
INSERT INTO account VALUES (1,'zhangsan',1000);  
INSERT INTO account VALUES (2,'lisi',1000);

执行后的 account 表中的数据如图 2 所示。

![执行结果](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/762_276_8a5.PNG)  
图 2  执行结果

#### 3\. 创建 c3p0-db.properties

在项目的 src 下创建一个名为 c3p0-db.properties 的配置文件，这里使用 C3P0 数据源，需要在该文件中添加如下配置：
````  
jdbc.driverClass = com.mysql.jdbc.Driver  
jdbc.jdbcUrl = jdbc:mysql://localhost:3306/spring  
jdbc.user = root  
jdbc.password = root  
````  
#### 4\. 实现 DAO

#### 1）创建 AccountDao 接口

在项目的 src 目录下创建一个名为 com.mengma.dao 的包，在该包下创建一个接口 AccountDao，并在接口中创建汇款和收款的方法，如下所示。
````  
package com.mengma.dao;  
public interface AccountDao {  
    // 汇款  
    public void out(String outUser, int money);  
    // 收款  
    public void in(String inUser, int money);} ````  
上述代码中，定义了 out() 和 in() 两个方法，分别用于表示汇款和收款。  
  
#### 2）创建DAO层接口实现类  
  
在项目的 src 目录下创建一个名为 com.mengma.dao.impl 的包，在该包下创建实现类 AccountDaoImpl，如下所示。  
````  
package com.mengma.dao.impl;

import org.springframework.jdbc.core.JdbcTemplate;  
import com.mengma.dao.AccountDao;

public class AccountDaoImpl implements AccountDao {  
private JdbcTemplate jdbcTemplate;  
public void setJdbcTemplate(JdbcTemplate jdbcTemplate) {        this.jdbcTemplate = jdbcTemplate;    }  
// 汇款的实现方法  
public void out(String outUser, int money) {        this.jdbcTemplate.update("update account set money =money-?"                + "where username =?", money, outUser);    }  
// 收款的实现方法  
public void in(String inUser, int money) {        this.jdbcTemplate.update("update account set money =money+?"                + "where username =?", money, inUser);    }} ````  
上述代码中，使用 JdbcTemplate 类的 update() 方法实现了更新操作。

#### 5\. 实现 Service

#### 1）创建 Service 层接口

在项目的 src 目录下创建一个名为 com.mengma.service 的包，在该包下创建接口 AccountService，如下所示。
````  
package com.mengma.service;  
  
public interface AccountService {  
    // 转账  
    public void transfer(String outUser, String inUser, int money);} ````  
#### 2）创建 Service 层接口实现类  
  
在项目的 src 目录下创建一个名为 com.mengma.service.impl 的包，在该包下创建实现类 AccountServiceImpl，如下所示。  
````  
package com.mengma.service.impl;

import com.mengma.dao.AccountDao;

public class AccountServiceImpl {  
private AccountDao accountDao;  
public void setAccountDao(AccountDao accountDao) {        this.accountDao = accountDao;    }  
public void transfer(String outUser, String inUser, int money) {        this.accountDao.out(outUser, money);        this.accountDao.in(inUser, money);    }} ````  
上述代码中可以看出，该类实现了 AccountService 接口，并对转账的方法进行了实现，根据参数的不同调用 DAO 层相应的方法。

#### 6\. 创建 Spring 配置文件

在项目的 src 目录下创建 Spirng 配置文件 applicationContext.xml，编辑后如下所示。
````  
 <?xml version="1.0" encoding="UTF-8"?><beans xmlns="http://www.springframework.org/schema/beans"  
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"    xmlns:context="http://www.springframework.org/schema/context"    xmlns:tx="http://www.springframework.org/schema/tx"    xmlns:aop="http://www.springframework.org/schema/aop"    xsi:schemaLocation="http://www.springframework.org/schema/beans            http://www.springframework.org/schema/beans/spring-beans-2.5.xsd            http://www.springframework.org/schema/context  
            http://www.springframework.org/schema/context/spring-context.xsd            http://www.springframework.org/schema/tx            http://www.springframework.org/schema/tx/spring-tx-2.5.xsd            http://www.springframework.org/schema/aop            http://www.springframework.org/schema/aop/spring-aop-2.5.xsd">    <!-- 加载properties文件 -->    <context:property-placeholder location="classpath:c3p0-db.properties" />    <!-- 配置数据源，读取properties文件信息 -->    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">        <property name="driverClass" value="${jdbc.driverClass}" />        <property name="jdbcUrl" value="${jdbc.jdbcUrl}" />        <property name="user" value="${jdbc.user}" />        <property name="password" value="${jdbc.password}" />    </bean>    <!-- 配置jdbc模板 -->    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">        <property name="dataSource" ref="dataSource" />    </bean>    <!-- 配置dao -->  
    <bean id="accountDao" class="com.mengma.dao.impl.AccountDaoImpl">        <property name="jdbcTemplate" ref="jdbcTemplate" />    </bean>    <!-- 配置service -->  
    <bean id="accountService" class="com.mengma.service.impl.AccountServiceImpl">        <property name="accountDao" ref="accountDao" />    </bean>    <!-- 事务管理器，依赖于数据源 -->    <bean id="txManager"        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">        <property name="dataSource" ref="dataSource" />    </bean>    <!-- 编写通知：对事务进行增强（通知），需要编写切入点和具体执行事务的细节 -->    <tx:advice id="txAdvice" transaction-manager="txManager">        <tx:attributes>            <!-- 给切入点方法添加事务详情，name表示方法名称，*表示任意方法名称，propagation用于设置传播行为，read-only表示隔离级别，是否只读 -->            <tx:method name="find*" propagation="SUPPORTS"                rollback-for="Exception" />            <tx:method name="*" propagation="REQUIRED" isolation="DEFAULT"                read-only="false" />        </tx:attributes>    </tx:advice>    <!-- aop编写，让Spring自动对目标生成代理，需要使用AspectJ的表达式 -->    <aop:config>        <!-- 切入点 -->        <aop:pointcut expression="execution(* com.mengma.service.*.*(..))"            id="txPointCut" />        <!-- 切面：将切入点与通知整合 -->        <aop:advisor pointcut-ref="txPointCut" advice-ref="txAdvice" />    </aop:config></beans> ````  
上述代码中，首先在 <beans> 标记的第 6、13 和 14 行代码分别添加了 AOP 所需的命名空间声明。第 42～50 行代码使用 <tx:advice> 标记配置事务通知内容。  
  
第 52～58 行代码使用  标记定义切面，其中第 54 行代码应用了 AspectJ 表达式，代表 com.mengma.service 包下所有类的所有方法都应用事务规则，第 57 行代码使用  标记将切入点与事务通知整合，基于 AOP 的声明式事务配置完成。  
  
#### 7\. 创建测试类  
  
在项目的 src 目录下创建 com.mengma.test 的包，在该包下创建测试类 AccountTest，如下所示。  
````  
package com.mengma.test;  
import org.junit.Test;  
import org.springframework.context.ApplicationContext;  
import org.springframework.context.support.ClassPathXmlApplicationContext;  
import com.mengma.service.AccountService;  
public class AccountTest {  
@Test    public void test() {        // 获得Spring容器，并操作  
String xmlPath = "applicationContext.xml";        ApplicationContext applicationContext = new ClassPathXmlApplicationContext(                xmlPath);        AccountService accountService = (AccountService) applicationContext                .getBean("accountService");        accountService.transfer("zhangsan", "lisi", 100);    }} ````  
上述代码中模拟了银行转账业务，从 zhangsan 的账户向 lisi 的账户中转入 100 元。使用 JUnit 测试运行 test() 方法，运行成功后，查询 account 表，如图 3 所示。

从图 3 的查询结果中可以看出，zhangsan 成功向 lisi 转账 100 元。

![查询结果](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/763_9ae_de5.PNG)  
图 3  查询结果

下面通过修改案例模拟转账失败的情况，在的 transfer() 方法中添加一行代码“int i=1/0；”模拟系统断电的情况，具体代码如下所示：
````  
 public void transfer(String outUser, String inUser, int money) {    this.accountDao.out(outUser, money);    //模拟断电  
    int i = 1/0;    this.accountDao.in(inUser, money);
 } 
````  
    
重新测试运行 test() 方法，JUnit 控制台输出的信息如图 4 所示。  
  
![控制台输出结果](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/764_7a3_f52.png)  
图 4  控制台输出结果  
  
从图 4 中可以看出，在执行测试方法时，出现了除以 0 的异常信息。此时再次查询 account 表，其查询结果如图 5 所示。  
  
从图 5 的查询结果中可以看出，表中的数据并没有发生变化。由于程序在执行过程中抛出了异常，事务不能正常被提交，所以转账失败。由此可知，Spring 的事务管理生效了。  
  
![查询结果](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/763_9ae_de5.PNG)  
图 5  查询结果  
  
# Spring声明式事务管理（基于Annotation注解方式实现）  
  
在 [Spring](http://www.voidme.com/spring) 中，除了使用基于 XML 的方式可以实现声明式事务管理以外，还可以通过 Annotation 注解的方式实现声明式事务管理。  
  
使用 Annotation 的方式非常简单，只需要在项目中做两件事，具体如下。  
  
#### 1）在 Spring 容器中注册驱动，代码如下所示：  
````  
<tx:annotation-driven transaction-manager="txManager"/>
````  
#### 2）在需要使用事务的业务类或者方法中添加注解 @Transactional，并配置 @Transactional 的参数。关于 @Transactional 的参数如图 1 所示。  
  
![@Transactional参数列表](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/765_7af_ff7.png)  
图 1  @Transactional参数列表  
  
下面通过修改《 [Spring基于XML实现事务管理](http://www.voidme.com/spring/spring-transaction-management-by-xml)》教程中银行转账的案例讲解如何使用 Annotation 注解的方式实现 Spring 声明式事务管理。  
  
#### 1\. 注册驱动  
  
修改 Spring 配置文件 applicationContext.xml，修改后如下所示。  


````  
<?xml version="1.0" encoding="UTF-8"?>  
<beans xmlns="http://www.springframework.org/schema/beans"  
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"    xmlns:context="http://www.springframework.org/schema/context"    xmlns:tx="http://www.springframework.org/schema/tx"    xmlns:aop="http://www.springframework.org/schema/aop"    xsi:schemaLocation="http://www.springframework.org/schema/beans            http://www.springframework.org/schema/beans/spring-beans-2.5.xsd            http://www.springframework.org/schema/context  
http://www.springframework.org/schema/context/spring-context.xsd            http://www.springframework.org/schema/tx            http://www.springframework.org/schema/tx/spring-tx-2.5.xsd            http://www.springframework.org/schema/aop            http://www.springframework.org/schema/aop/spring-aop-2.5.xsd">    <!-- 加载properties文件 -->    <context:property-placeholder location="classpath:c3p0-db.properties" />    <!-- 配置数据源，读取properties文件信息 -->    <bean id="dataSource" class="com.mchange.v2.c3p0.ComboPooledDataSource">        <property name="driverClass" value="${jdbc.driverClass}" />        <property name="jdbcUrl" value="${jdbc.jdbcUrl}" />        <property name="user" value="${jdbc.user}" />        <property name="password" value="${jdbc.password}" />    </bean>    <!-- 配置jdbc模板 -->    <bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">        <property name="dataSource" ref="dataSource" />    </bean>    <!-- 配置dao -->  
<bean id="accountDao" class="com.mengma.dao.impl.AccountDaoImpl">        <property name="jdbcTemplate" ref="jdbcTemplate" />    </bean>    <!-- 配置service -->  
<bean id="accountService" class="com.mengma.service.impl.AccountServiceImpl">        <property name="accountDao" ref="accountDao" />    </bean>    <!-- 事务管理器，依赖于数据源 -->    <bean id="txManager"        class="org.springframework.jdbc.datasource.DataSourceTransactionManager">        <property name="dataSource" ref="dataSource" />    </bean>    <!-- 注册事务管理驱动 -->    <tx:annotation-driven transaction-manager="txManager"/></beans> ````  
````
上述代码中可以看出，与原来的配置文件相比，这里只修改了事务管理器部分，新添加并注册了事务管理器的驱动。

需要注意的是，在学习 AOP 注解方式开发时，需要在配置文件中开启注解处理器，指定扫描哪些包下的注解，这里没有开启注解处理器是因为在第 33～35 行手动配置了 AccountServiceImpl，而 @Transactional 注解就配置在该类中，所以会直接生效。

#### 2\. 添加 @Transactional 注解

修改 AccountServiceImpl，在文件中添加 @Transactional 注解及参数，添加后如下所示。
````  
package com.mengma.service.impl;  
  
import org.springframework.transaction.annotation.Isolation;  
import org.springframework.transaction.annotation.Propagation;  
import org.springframework.transaction.annotation.Transactional;  
  
import com.mengma.dao.AccountDao;  
  
@Transactional(propagation = Propagation.REQUIRED, isolation = Isolation.DEFAULT, readOnly = false)  
public class AccountServiceImpl {  
    private AccountDao accountDao;  
    public void setAccountDao(AccountDao accountDao) {        this.accountDao = accountDao;    }  
    public void transfer(String outUser, String inUser, int money) {        this.accountDao.out(outUser, money);        // 模拟断电  
        int i = 1 / 0;        this.accountDao.in(inUser, money);    
}}
````

需要注意的是，在使用 @Transactional 注解时，参数之间用“，”进行分隔。  
  
使用 JUnit 测试再次运行 test() 方法时，控制台同样会输出如图 2 所示的异常信息，这说明使用基于 Annotation 注解的方式同样实现了 Spring 的声明式事务管理。如果注释掉模拟断电的代码进行测试，则转账操作可以正常完成。  
  
![运行结果](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/764_7a3_f52.png)  
图 2  运行结果  
  
# 参考文章  
https://www.w3cschool.cn/wkspring  
https://www.runoob.com/w3cnote/basic-knowledge-summary-of-spring.html  
http://codepub.cn/2015/06/21/Basic-knowledge-summary-of-Spring  
https://dunwu.github.io/spring-tutorial  
https://mszlu.com/java/spring  
http://c.biancheng.net/spring/aop-module.html