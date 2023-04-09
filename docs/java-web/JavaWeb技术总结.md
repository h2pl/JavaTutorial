# 目录
  * [Servlet及相关类](#servlet及相关类)
  * [Jsp和ViewResolver](#jsp和viewresolver)
  * [filter，listener](#filter，listener)
  * [web.xml](#webxml)
  * [war包](#war包)
  * [tomcat基础](#tomcat基础)
  * [log4j](#log4j)
  * [数据库驱动和连接池](#数据库驱动和连接池)
  * [单元测试](#单元测试)
  * [Maven](#maven)
  * [Git](#git)
  * [Json和xml](#json和xml)
  * [hibernate和mybatis](#hibernate和mybatis)


    
这篇总结主要是基于我之前两个系列的文章而来。主要是把重要的知识点用自己的话说了一遍，可能会有一些错误，还望见谅和指点。谢谢

更多详细内容可以查看我的专栏文章：

JavaWeb技术世界

https://blog.csdn.net/column/details/21850.html


<!-- more -->

## Servlet及相关类

servlet是一个接口，它的实现类有GenericServlet，而httpservlet是GenericServlet的一个子类，一般我们都会使用这个类。

servletconfig是用于保存servlet配置信息的数据结构，而servletcontext则负责保持servlet的上下文，web应用启动时加载web.xml信息于servletconfig中。

## Jsp和ViewResolver

jsp页面需要编译成class文件并通过tomcat的类加载器进行加载，形成servlet实例，请求到来时实际上执行的是servlet代码，然后最终再通过viewresolver渲染成页面。

## filter，listener

filter是过滤器，也需要在web.xml中配置，是责任链式的调用，在servlet执行service方法前执行。
listener则是监听器，由于容器组件都实现了lifecycle接口，所以可以在组件上添加监听器来控制生命周期。

## web.xml

web.xml用来配置servlet和servlet的配置信息，listener和filter。也可以配置静态文件的目录等。

## war包

waWAR包
WAR(Web Archive file)网络应用程序文件，是与平台无关的文件格式，它允许将许多文件组合成一个压缩文件。war专用在web方面 。

JAVA WEB工程，都是打成WAR包进行发布。

典型的war包内部结构如下：

webapp.war
````
  |    index.jsp

  |

  |— images

  |— META-INF

  |— WEB-INF

          |   web.xml                   // WAR包的描述文件
    
          |
    
          |— classes
    
          |          action.class       // java类文件
    
          |
    
          |— lib
    
                    other.jar             // 依赖的jar包
    
                    share.jar
````
## tomcat基础

上一篇文章关于网络编程和NIO已经讲过了，这里按住不表。

## log4j

log4j是非常常用的日志组件，不过现在为了使用更通用的日志组件，一般使用slf4j来配置日志管理器，然后再介入日志源，比如log4j这样的日志组件。

## 数据库驱动和连接池

一般我们会使用class.forname加载数据库驱动，但是随着Spring的发展，现在一般会进行数据源DataSource这个bean的配置，bean里面填写你的数据来源信息即可，并且在实现类中可以选择支持连接池的数据源实现类，比如c3poDataSource，非常方便。

数据库连接池本身和线程池类似，就是为了避免频繁建立数据库连接，保存了一部分连接并存放在集合里，一般可以用队列来存放。

除此之外，还可以使用tomcat的配置文件来管理数据库连接池，只需要简单的一些配置，就可以让tomcat自动管理数据库的连接池了。
应用需要使用的时候，通过jndi的方式访问即可，具体方法就是调用jndi命名服务的look方法。

## 单元测试

单元测试是工程中必不可少的组件，maven项目在打包期间会自动运行所有单元测试。一般我们使用junit做单元测试，统一地在test包中分别测试service和dao层，并且使用mock方法来构造假的数据，以便跳过数据库或者其他外部资源来完成测试。

## Maven

maven是一个项目构建工具，基于约定大于配置的方式，规定了一个工程各个目录的用途，并且根据这些规则进行编译，测试和打包。
同时他提供了方便的包管理方式，以及快速部署的优势。

## Git

git是分布式的代码管理工具，比起svn有着分布式的优势。太过常见了，略了。

## Json和xml
数据描述形式不同，json更简洁。

## hibernate和mybatis

由于jdbc方式的数据库连接和语句执行太过繁琐，重复代码太多，后来提出了jdbctemplate对数据进行bean转换。

但是还是差强人意，于是转而出现了hibernate这类的持久化框架。可以做到数据表和bean一一映射，程序只需要操作bean就可以完成数据库的curd。

mybatis比hibernate更轻量级，mybatis支持原生sql查询，并且也可以使用bean映射，同时还可以自定义地配置映射对象，更加灵活，并且在多表查询上更有优势。


