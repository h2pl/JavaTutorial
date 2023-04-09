# 目录
* [前言](#前言)
* [Mybatis是什么](#mybatis是什么)
    * [特点](#特点)
    * [核心类介绍](#核心类介绍)
    * [功能架构：我们把Mybatis的功能架构分为三层](#功能架构：我们把mybatis的功能架构分为三层)
    * [框架结构：](#框架结构：)
    * [执行流程：](#执行流程：)
* [与Hibernate的异同](#与hibernate的异同)
* [参考文章](#参考文章)
* [微信公众号](#微信公众号)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)
* [mybatis新手上路](#mybatis新手上路)
    * [MyBatis简介](#mybatis简介)
    * [MyBatis整体架构及运行流程](#mybatis整体架构及运行流程)
        * [1.数据源配置文件](#1数据源配置文件)
        * [2.Sql映射文件](#2sql映射文件)
        * [3.会话工厂与会话](#3会话工厂与会话)
        * [4.运行流程](#4运行流程)
    * [测试工程搭建](#测试工程搭建)
    * [微信公众号](#微信公众号-1)
        * [个人公众号：程序员黄小斜](#个人公众号：程序员黄小斜)
        * [技术公众号：Java技术江湖](#技术公众号：java技术江湖)


本文转载自互联网，侵删
本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本系列文章将同步到我的个人博客：

> www.how2playlife.com

更多Java技术文章将陆续在微信公众号【Java技术江湖】更新，敬请关注。

本文是《走进JavaWeb技术世界》系列博文的其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从servlet到框架，从ssm再到SpringBoot，一步步地学习JavaWeb基础知识，并上手进行实战，接着了解JavaWeb项目中经常要使用的技术和组件，包括日志组件、Maven、Junit，等等内容，以便让你更完整地了解整个JavaWeb技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## 前言

学习一个新东西前，如果能对他有一个比较直观的印象与定位，那么接下来的学习过程就会顺畅很多。所以本文主要是我对Mybatis的一个简单入门性的总结介绍（前提还是需要些必要的概念认知）。
_PS:文末有参考列表_

## Mybatis是什么

Mybatis是一个持久层框架，用于数据的持久化。主要表现为将SQL与POJO进行一个映射，将SQL从代码中解耦。基本概念如图：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4226917-700b83c25876d6d0.png)


使用时，以User为例，UserMapper定义了`findById`接口，该接口返回一个User对象，接口的实现为一个xml配置文件。该xml文件中定义对应接口中的实现所需要的SQL。从而达到将SQL与代码解耦的目标。

```
<?xml version="1.0" encoding="UTF-8"?>  
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"   
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">  

<mapper namespace="com.mybatis.UserMapper">  

    <select id="findById" parameterType="int" resultType="User">  
        select user_id id,user_name userName,user_age age from t_user where user_id=#{id}  
    </select>  

</mapper> 

```

MyBatis 是Apache的一个Java开源项目，是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。Mybatis可以将Sql语句配置在XML文件中，避免将Sql语句硬编码在Java类中。

### 特点

1.Mybatis通过参数映射方式，可以将参数灵活的配置在SQL语句中的配置文件中，避免在Java类中配置参数（JDBC）

2.Mybatis通过输出映射机制，将结果集的检索自动映射成相应的Java对象，避免对结果集手工检索（JDBC）

3.Mybatis可以通过Xml配置文件对数据库连接进行管理

### 核心类介绍

1.SqlSessionaFactoryBuilder ：该类主要用于创建 SqlSessionFactory, 这个类可以被实例化、使用和丢弃，一旦创建了 SqlSessionFactory，就不再需要它了。 因此 SqlSessionFactoryBuilder 实例的最佳作用域是方法作用域（也就是局部方法变量）。

2.SqlSessionFactory ：该类的作用了创建 SqlSession, 从名字上我们也能看出, 该类使用了工厂模式, 每次应用程序访问数据库, 我们就要通过 SqlSessionFactory 创建 SqlSession, 所以 SqlSessionFactory 和整个 Mybatis 的生命周期是相同的. 这也告诉我们不能创建多个同一个数据的 SqlSessionFactory, 如果创建多个, 会消耗尽数据库的连接资源, 导致服务器夯机. 应当使用单例模式. 避免过多的连接被消耗, 也方便管理。

3.SqlSession ：SqlSession 相当于一个会话, 每次访问数据库都需要这样一个会话, 大家可能会想起了 JDBC 中的 Connection, 很类似, 但还是有区别的, 何况现在几乎所有的连接都是使用的连接池技术, 用完后直接归还而不会像 Session 一样销毁. 注意: 他是一个线程不安全的对象, 在设计多线程的时候我们需要特别的当心, 操作数据库需要注意其隔离级别, 数据库锁等高级特性, 此外, 每次创建的 SqlSession 都必须及时关闭它, 它长期存在就会使数据库连接池的活动资源减少, 对系统性能的影响很大, 我们一般在 finally 块中将其关闭. 还有, SqlSession 存活于一个应用的请求和操作, 可以执行多条 Sql, 保证事务的一致性。SqlSession在执行过程中，有包含了几大对象：

3.1.Executor ：执行器，由它调度 StatementHandler、ParameterHandler、ResultSetHandler 等来执行对应的 SQL。其中 StatementHandler 是最重要的。

3.2.StatementHandler ：作用是使用数据库的 Statement（PreparedStatement）执行操作，它是四大对象的核心，起到承上启下的作用，许多重要的插件都是通过拦截它来实现的。

3.3.ParamentHandler ：用来处理 SQL 参数的。

3.4.ResultSetHandler ：进行数据集的封装返回处理的。

4.Mapper ：映射器是一些由你创建的、绑定你映射的语句的接口。映射器接口的实例是从 SqlSession 中获得的, 他的作用是发送 SQL, 然后返回我们需要的结果. 或者执行 SQL 从而更改数据库的数据, 因此它应该在 SqlSession 的事务方法之内, 在 Spring 管理的 Bean 中, Mapper 是单例的。

### 功能架构：我们把Mybatis的功能架构分为三层

(1)API接口层：提供给外部使用的接口API，开发人员通过这些本地API来操纵数据库。接口层一接收到调用请求就会调用数据处理层来完成具体的数据处理。

(2)数据处理层：负责具体的SQL查找、SQL解析、SQL执行和执行结果映射处理等。它主要的目的是根据调用的请求完成一次数据库操作。

(3)基础支撑层：负责最基础的功能支撑，包括连接管理、事务管理、配置加载和缓存处理，这些都是共用的东西，将他们抽取出来作为最基础的组件。为上层的数据处理层提供最基础的支撑。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405164855.png)

### 框架结构：

(1)加载配置：配置来源于两个地方，一处是配置文件，一处是Java代码的注解，将SQL的配置信息加载成为一个个MappedStatement对象（包括了传入参数映射配置、执行的SQL语句、结果映射配置），存储在内存中。

(2)SQL解析：当API接口层接收到调用请求时，会接收到传入SQL的ID和传入对象（可以是Map、JavaBean或者基本数据类型），Mybatis会根据SQL的ID找到对应的MappedStatement，然后根据传入参数对象对MappedStatement进行解析，解析后可以得到最终要执行的SQL语句和参数。

(3)SQL执行：将最终得到的SQL和参数拿到数据库进行执行，得到操作数据库的结果。

(4)结果映射：将操作数据库的结果按照映射的配置进行转换，可以转换成HashMap、JavaBean或者基本数据类型，并将最终结果返回。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405164952.png)

### 执行流程：

1.获取SqlsessionFactory：根据配置文件（全局、sql映射）初始化configuration对象，

2.获取sqlSession：创建一个DefaultSqlSession对象，包含Configuration及Executor（根据全局配置文件中defaultExecutorType创建对应的Executor）

3.获取接口代理对象MapperProxy：DefaultSqlSession.getMapper拿到Mapper接口对应的MapperProxy

4.执行增删改查

1、调用DefaultSqlSession增删改查

2、创建StatementHandler （同时创建ParameterHandler,ResultSetHandler）

3、调用StatementHandler预编译参数以及设置参数值，使用ParameterHandler给sql设置参数

4、调用StatementHandler增删改查

5、ResultSetHandler封装结果

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20190821203129674.png)


## 与Hibernate的异同

Mybatis开始逐渐流行起来，必然有其原因，简单了解了一下它与同为持久层框架的Hibernate的异同。

*   映射模式
    从上面的简单概念可以知道Mybatis实际上着力点在POJO与SQL的映射。而Hibernate则主要是POJO与数据库表的对象关系映射。前者掌控力度更细，代码量会相对多一点，后者灵活性则差一点，更为自动化一些，与PHP里的`Eloquent`属于同类型。
*   性能
    Mybatis基于原生JDBC，相比于对JDBC进行二次封装的Hibernate性能会更好一点。
*   开发与维护
    Hibernate配置好实体类后，使用起来是比较简洁，舒服的，但是前期学习曲线比较陡，后期调优比较麻烦。Mybatis对SQL掌控的颗粒更细一点，相比较而言看上去简陋些。由于直接映射SQL，迁移性是个问题。

## 参考文章

<https://segmentfault.com/a/1190000009707894>

<https://www.cnblogs.com/hysum/p/7100874.html>

<http://c.biancheng.net/view/939.html>

<https://www.runoob.com/>

https://blog.csdn.net/android_hl/article/details/53228348

# mybatis新手上路

## MyBatis简介

Mybatis是Apache的一个Java开源项目，是一个支持动态Sql语句的持久层框架。Mybatis可以将Sql语句配置在XML文件中，避免将Sql语句硬编码在Java类中。与JDBC相比：

1.  Mybatis通过参数映射方式，可以将参数灵活的配置在SQL语句中的配置文件中，避免在Java类中配置参数（JDBC）
2.  Mybatis通过输出映射机制，将结果集的检索自动映射成相应的Java对象，避免对结果集手工检索（JDBC）
3.  Mybatis可以通过Xml配置文件对数据库连接进行管理。

## MyBatis整体架构及运行流程

Mybatis整体构造由 数据源配置文件、Sql映射文件、会话工厂、会话、执行器和底层封装对象组成。

### 1.数据源配置文件

通过配置的方式将数据库的配置信息从应用程序中独立出来，由独立的模块管理和配置。Mybatis的数据源配置文件包含数据库驱动、数据库连接地址、用户名密码、事务管理等，还可以配置连接池的连接数、空闲时间等。

一个SqlMapConfig.xml基本的配置信息如下：


<configuration>
    <!-- 加载数据库属性文件 -->
    <properties resource="db.properties"></properties>
    <environments default="development">
        <environment id="development">
            <!--使用JDBC实务管理-->
            <transactionManager type="JDBC"></transactionManager>
            <!--连接池 -->
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"></property>
                <property name="url" value="${jdbc.url}"></property>
                <property name="username" value="${jdbc.username}" />
                <property name="password" value="${jdbc.password}" />
            </dataSource>
        </environment>
    </environments>
</configuration>




### 2.Sql映射文件

Mybatis中所有数据库的操作都会基于该映射文件和配置的sql语句，在这个配置文件中可以配置任何类型的sql语句。框架会根据配置文件中的参数配置，完成对sql语句以及输入输出参数的映射配置。

Mapper.xml配置文件大致如下：


````
<!DOCTYPE mapper 
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sl.dao.ProductDao">
    <!-- 根据id查询product表
        resultType:返回值类型，一条数据库记录也就对应实体类的一个对象
        parameterType:参数类型，也就是查询条件的类型 -->
    <select id="selectProductById" resultType="com.sl.po.Product" parameterType="int">
     <!-- 这里和普通的sql 查询语句差不多，对于只有一个参数，后面的 #{id}表示占位符，里面不一定要写id,写啥都可以，但是不要空着，如果有多个参数则必须写pojo类里面的属性 --> select * from products where id = #{id} </select>
</mapper>
````


### 3.会话工厂与会话

Mybatis中会话工厂SqlSessionFactory类可以通过加载资源文件，读取数据源配置SqlMapConfig.xml信息，从而产生一种可以与数据库交互的会话实例SqlSession，会话实例SqlSession根据Mapper.xml文件中配置的sql,对数据库进行操作。

### 4.运行流程

会话工厂SqlSessionFactory通过加载资源文件获取SqlMapConfig.xml配置文件信息，然后生成可以与数据库交互的会话实例SqlSession。

会话实例可以根据Mapper配置文件中的Sql配置去执行相应的增删改查操作。

在SqlSession会话实例内部，通过执行器Executor对数据库进行操作，Executor依靠封装对象Mappered Statement，它分装了从mapper.xml文件中读取的信息（sql语句，参数，结果集类型）。

Mybatis通过执行器与Mappered Statement的结合实现与数据库的交互。

执行流程图：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/577318-20180702181255132-2135681101.png)

## 测试工程搭建

1.新建maven工程

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/577318-20180702181320571-1138935145.png)

2\. 添加依赖pom.xml



````
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.sl</groupId>
    <artifactId>mybatis-demo</artifactId>
    <version>0.0.1-SNAPSHOT</version>
    <properties>
        <junit.version>4.12</junit.version>
        <mybatis.version>3.4.1</mybatis.version>
        <mysql.version>5.1.32</mysql.version>
        <log4j.version>1.2.17</log4j.version>
    </properties>
    <dependencies>
        <!-- 单元测试 -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
            <!-- <scope>test</scope> -->
        </dependency>
        <!-- Mybatis -->
        <dependency>
            <groupId>org.mybatis</groupId>
            <artifactId>mybatis</artifactId>
            <version>${mybatis.version}</version>
        </dependency>
        <!-- mysql -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>${mysql.version}</version>
        </dependency>
        <!-- 日志处理 -->
        <dependency>
            <groupId>log4j</groupId>
            <artifactId>log4j</artifactId>
            <version>${log4j.version}</version>
        </dependency>
    </dependencies>
</project>

````


3.编写数据源配置文件SqlMapConfig.xml

````
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN" "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
<!-- 加载配置文件 -->
    <properties resource="db.properties"></properties>
    <environments default="development">
        <!-- id属性必须和上面的defaut一致 -->
        <environment id="development">
            <transactionManager type="JDBC"></transactionManager>
            <!--dataSource 元素使用标准的 JDBC 数据源接口来配置 JDBC 连接对象源 -->
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"></property>
                <property name="url" value="${jdbc.url}"></property>
                <property name="username" value="${jdbc.username}" />
                <property name="password" value="${jdbc.password}" />
            </dataSource>
        </environment>
    </environments>
     <!—申明mapper文件 -->
        <mappers>
        <!-- xml实现    注册productMapper.xml文件 -->
        <mapper resource="mapper/productMapper.xml"></mapper>
    </mappers>
</configuration>
````


4.编写SQL映射配置文件productMapper.xml


````

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper 
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sl.mapper.ProductMapper">


    <select id="selectAllProduct" resultType="com.sl.po.Product"> select * from products </select>

</mapper>

````
5.编写测试代码TestClient.java

````
//使用productMapper.xml配置文件
public class TestClient { //定义会话SqlSession
    SqlSession session =null;


    @Before public void init() throws IOException { //定义mabatis全局配置文件
        String resource = "SqlMapConfig.xml"; //加载mybatis全局配置文件 //InputStream inputStream = TestClient.class.getClassLoader().getResourceAsStream(resource);

 InputStream inputStream = Resources.getResourceAsStream(resource);
        SqlSessionFactoryBuilder builder = new SqlSessionFactoryBuilder();
        SqlSessionFactory factory = builder.build(inputStream); //根据sqlSessionFactory产生会话sqlsession
        session = factory.openSession();    
    } //查询所有user表所有数据
 @Test public void testSelectAllUser() {
        String statement = "com.sl.mapper.ProductMapper.selectAllProduct";
        List<Product> listProduct =session.selectList(statement); for(Product product:listProduct)
        {
            System.out.println(product);
        } //关闭会话
 session.close();    
    }

}
````

````
public class Product { private int Id; private String Name; private String Description; private BigDecimal UnitPrice; private String ImageUrl; private Boolean IsNew; public int getId() { return Id;
    } public void setId(int id) { this.Id = id;
    } public String getName() { return Name;
    } public void setName(String name) { this.Name = name;
    } public String getDescription() { return Description;
    } public void setDescription(String description) { this.Description = description;
    } public BigDecimal getUnitPrice() { return UnitPrice;
    } public void setUnitPrice(BigDecimal unitprice) { this.UnitPrice = unitprice;
    } public String getImageUrl() { return Name;
    } public void setImageUrl(String imageurl) { this.ImageUrl = imageurl;
    } public boolean getIsNew() { return IsNew;
    } public void setIsNew(boolean isnew) { this.IsNew = isnew;
    }


    @Override public String toString() { return "Product [id=" + Id + ", Name=" + Name + ", Description=" + Description + ", UnitPrice=" + UnitPrice + ", ImageUrl=" + ImageUrl + ", IsNew=" + IsNew+ "]";
    }

}
````
6.运行测试用例

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/577318-20180702181558003-1700599553.png)




                     