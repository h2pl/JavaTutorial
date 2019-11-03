本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《走进JavaWeb技术世界》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。
该系列博文会告诉你如何从入门到进阶，从servlet到框架，从ssm再到SpringBoot，一步步地学习JavaWeb基础知识，并上手进行实战，接着了解JavaWeb项目中经常要使用的技术和组件，包括日志组件、Maven、Junit，等等内容，以便让你更完整地了解整个Java Web技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## 前言

学习一个新东西前，如果能对他有一个比较直观的印象与定位，那么接下来的学习过程就会顺畅很多。所以本文主要是我对Mybatis的一个简单入门性的总结介绍（前提还是需要些必要的概念认知）。
_PS:文末有参考列表_

## Mybatis是什么

Mybatis是一个持久层框架，用于数据的持久化。主要表现为将SQL与POJO进行一个映射，将SQL从代码中解耦。基本概念如图：





![](https://upload-images.jianshu.io/upload_images/4226917-700b83c25876d6d0.png?imageMogr2/auto-orient/strip|imageView2/2/w/462/format/webp)





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

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9nc3MwLmJkc3RhdGljLmNvbS8tNG8zZFNhZ194STRraEdrcG9XSzFIRjZoaHkvYmFpa2UvYzAlM0RiYWlrZTgwJTJDNSUyQzUlMkM4MCUyQzI2L3NpZ249NGRmM2FiMTQzOWRiYjZmZDMxNTZlZDc0Njg0ZGMwN2QvMGI0NmYyMWZiZTA5NmI2M2VhMGQ0MWJmMGMzMzg3NDRlYWY4YWNjYy5qcGc)

### 框架结构：

(1)加载配置：配置来源于两个地方，一处是配置文件，一处是Java代码的注解，将SQL的配置信息加载成为一个个MappedStatement对象（包括了传入参数映射配置、执行的SQL语句、结果映射配置），存储在内存中。

(2)SQL解析：当API接口层接收到调用请求时，会接收到传入SQL的ID和传入对象（可以是Map、JavaBean或者基本数据类型），Mybatis会根据SQL的ID找到对应的MappedStatement，然后根据传入参数对象对MappedStatement进行解析，解析后可以得到最终要执行的SQL语句和参数。

(3)SQL执行：将最终得到的SQL和参数拿到数据库进行执行，得到操作数据库的结果。

(4)结果映射：将操作数据库的结果按照映射的配置进行转换，可以转换成HashMap、JavaBean或者基本数据类型，并将最终结果返回。

![](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9nc3MxLmJkc3RhdGljLmNvbS85dm8zZFNhZ194STRraEdrcG9XSzFIRjZoaHkvYmFpa2UvYzAlM0RiYWlrZTgwJTJDNSUyQzUlMkM4MCUyQzI2L3NpZ249ZjZjYzY5NzY4MjI2Y2ZmYzdkMjdiN2UwZDg2ODIxZjUvNjQzODBjZDc5MTIzOTdkZGUwY2Q4N2ViNTk4MmIyYjdkMWEyODdhYy5qcGc)

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

![](https://img-blog.csdnimg.cn/20190821203129674.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2phZGViYWk=,size_16,color_FFFFFF,t_70)


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
## 微信公众号

### 个人公众号：黄小斜

黄小斜是跨考软件工程的 985 硕士，自学 Java 两年，拿到了 BAT 等近十家大厂 offer，从技术小白成长为阿里工程师。

作者专注于 JAVA 后端技术栈，热衷于分享程序员干货、学习经验、求职心得和程序人生，目前黄小斜的CSDN博客有百万+访问量，知乎粉丝2W+，全网已有10W+读者。

黄小斜是一个斜杠青年，坚持学习和写作，相信终身学习的力量，希望和更多的程序员交朋友，一起进步和成长！

**原创电子书:**
关注公众号【黄小斜】后回复【原创电子书】即可领取我原创的电子书《菜鸟程序员修炼手册：从技术小白到阿里巴巴Java工程师》

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

**考研复习资料：** 
计算机考研大礼包，都是我自己考研复习时用的一些复习资料,包括公共课和专业的复习视频，这里也推荐给大家，关注公众号后，后台回复关键字 **“考研”** 即可免费获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)

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


<pre><configuration>
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
</configuration></pre>



### 2.Sql映射文件

Mybatis中所有数据库的操作都会基于该映射文件和配置的sql语句，在这个配置文件中可以配置任何类型的sql语句。框架会根据配置文件中的参数配置，完成对sql语句以及输入输出参数的映射配置。

Mapper.xml配置文件大致如下：



<pre><!DOCTYPE mapper 
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sl.dao.ProductDao">
    <!-- 根据id查询product表
        resultType:返回值类型，一条数据库记录也就对应实体类的一个对象
        parameterType:参数类型，也就是查询条件的类型 -->
    <select id="selectProductById" resultType="com.sl.po.Product" parameterType="int">
     <!-- 这里和普通的sql 查询语句差不多，对于只有一个参数，后面的 #{id}表示占位符，里面不一定要写id,写啥都可以，但是不要空着，如果有多个参数则必须写pojo类里面的属性 --> select * from products where id = #{id} </select>
</mapper></pre>


### 3.会话工厂与会话

Mybatis中会话工厂SqlSessionFactory类可以通过加载资源文件，读取数据源配置SqlMapConfig.xml信息，从而产生一种可以与数据库交互的会话实例SqlSession，会话实例SqlSession根据Mapper.xml文件中配置的sql,对数据库进行操作。

### 4.运行流程

会话工厂SqlSessionFactory通过加载资源文件获取SqlMapConfig.xml配置文件信息，然后生成可以与数据库交互的会话实例SqlSession。

会话实例可以根据Mapper配置文件中的Sql配置去执行相应的增删改查操作。

在SqlSession会话实例内部，通过执行器Executor对数据库进行操作，Executor依靠封装对象Mappered Statement，它分装了从mapper.xml文件中读取的信息（sql语句，参数，结果集类型）。

Mybatis通过执行器与Mappered Statement的结合实现与数据库的交互。

执行流程图：

![](https://images2018.cnblogs.com/blog/577318/201807/577318-20180702181255132-2135681101.png)

## 测试工程搭建

 1. 新建maven工程

![](https://images2018.cnblogs.com/blog/577318/201807/577318-20180702181320571-1138935145.png)

2\. 添加依赖pom.xml




<pre><project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
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
</project></pre>



3.编写数据源配置文件SqlMapConfig.xml


<pre><?xml version="1.0" encoding="UTF-8"?>
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
</configuration></pre>


4.编写SQL映射配置文件productMapper.xml

![](https://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)



<pre><?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper 
    PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.sl.mapper.ProductMapper">

    <select id="selectAllProduct" resultType="com.sl.po.Product"> select * from products </select>

</mapper></pre>


5.编写测试代码TestClient.java


<pre>//使用productMapper.xml配置文件
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

}</pre>

<pre>public class Product { private int Id; private String Name; private String Description; private BigDecimal UnitPrice; private String ImageUrl; private Boolean IsNew; public int getId() { return Id;
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
}</pre>

6.运行测试用例

![](https://images2018.cnblogs.com/blog/577318/201807/577318-20180702181558003-1700599553.png)


## 微信公众号

### 个人公众号：程序员黄小斜

​
黄小斜是 985 硕士，阿里巴巴Java工程师，在自学编程、技术求职、Java学习等方面有丰富经验和独到见解，希望帮助到更多想要从事互联网行业的程序员们。
​
作者专注于 JAVA 后端技术栈，热衷于分享程序员干货、学习经验、求职心得，以及自学编程和Java技术栈的相关干货。
​
黄小斜是一个斜杠青年，坚持学习和写作，相信终身学习的力量，希望和更多的程序员交朋友，一起进步和成长！

**原创电子书:**
关注微信公众号【程序员黄小斜】后回复【原创电子书】即可领取我原创的电子书《菜鸟程序员修炼手册：从技术小白到阿里巴巴Java工程师》这份电子书总结了我2年的Java学习之路，包括学习方法、技术总结、求职经验和面试技巧等内容，已经帮助很多的程序员拿到了心仪的offer！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取，包括Java、python、C++、大数据、机器学习、前端、移动端等方向的技术资料。


![](https://img-blog.csdnimg.cn/20190829222750556.jpg)


### 技术公众号：Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的微信公众号【Java技术江湖】

这是一位阿里 Java 工程师的技术小站。作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！


**Java工程师必备学习资源:** 
关注公众号后回复”Java“即可领取 Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源


![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

​                     