# Table of Contents

  * [打破双亲委派模型](#打破双亲委派模型)
    * [JNDI](#jndi)
    * [[JNDI 的理解](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.cnblogs.com%2Fzhchoutai%2Fp%2F7389089.html)](#[jndi-的理解]httpsyqaliyuncomgoarticlerenderredirecturlhttps3a2f2fwwwcnblogscom2fzhchoutai2fp2f7389089html)
  * [OSGI](#osgi)
    * [1.如何正确的理解和认识OSGI技术？](#1如何正确的理解和认识osgi技术？)
  * [Tomcat类加载器以及应用间class隔离与共享](#tomcat类加载器以及应用间class隔离与共享)
    * [类加载器](#类加载器)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


本文转自互联网，侵删

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章将同步到我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《深入理解JVM虚拟机》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，一步步地学习JVM基础知识，并上手进行JVM调优实战，JVM是每一个Java工程师必须要学习和理解的知识点，你必须要掌握其实现原理，才能更完整地了解整个Java技术体系，形成自己的知识框架。

为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## 打破双亲委派模型

### JNDI

### [JNDI 的理解](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Fwww.cnblogs.com%2Fzhchoutai%2Fp%2F7389089.html)





JNDI是 Java 命名与文件夹接口（Java Naming and Directory Interface），在J2EE规范中是重要的规范之中的一个，不少专家觉得，没有透彻理解JNDI的意义和作用，就没有真正掌握J2EE特别是EJB的知识。 

那么，JNDI究竟起什么作用？//带着问题看文章是最有效的 

要了解JNDI的作用，我们能够从“假设不用JNDI我们如何做？用了JNDI后我们又将如何做？”这个问题来探讨。 

没有JNDI的做法： 

程序猿开发时，知道要开发訪问MySQL数据库的应用，于是将一个对 MySQL JDBC 驱动程序类的引用进行了编码，并通过使用适当的 JDBC URL 连接到数据库。 
就像以下代码这样： 

    
    
    
    1.  Connection conn=null;  
    2.  try {  
    3.    Class.forName("com.mysql.jdbc.Driver",  
    4.                  true, Thread.currentThread().getContextClassLoader());  
    5.    conn=DriverManager.  
    6.      getConnection("jdbc:mysql://MyDBServer?user=qingfeng&password=mingyue");  
    7.    ......  
    8.    conn.close();  
    9.  } catch(Exception e) {  
    10.    e.printStackTrace();  
    11.  } finally {  
    12.    if(conn!=null) {  
    13.      try {  
    14.        conn.close();  
    15.      } catch(SQLException e) {}  
    16.    }  
    17.  }  



这是传统的做法，也是曾经非Java程序猿（如Delphi、VB等）常见的做法。

这种做法一般在小规模的开发过程中不会产生问题，仅仅要程序猿熟悉Java语言、了解JDBC技术和MySQL，能够非常快开发出对应的应用程序。 

没有JNDI的做法存在的问题：

    1、数据库server名称MyDBServer 、username和口令都可能须要改变，由此引发JDBC URL须要改动； 
    2、数据库可能改用别的产品，如改用DB2或者Oracle，引发JDBC驱动程序包和类名须要改动； 
    3、随着实际使用终端的添加，原配置的连接池參数可能须要调整； 
    4、...... 

解决的方法： 

程序猿应该不须要关心“详细的数据库后台是什么？JDBC驱动程序是什么？JDBC URL格式是什么？訪问数据库的username和口令是什么？”等等这些问题。

程序猿编写的程序应该没有对 JDBC驱动程序的引用，没有server名称，没实username称或口令 —— 甚至没有数据库池或连接管理。

而是把这些问题交给J2EE容器（比方weblogic）来配置和管理，程序猿仅仅须要对这些配置和管理进行引用就可以。 

由此，就有了JNDI。

//看的出来。是为了一个最最核心的问题：是为了解耦，是为了开发出更加可维护、可扩展//的系统 

用了JNDI之后的做法： 
首先。在在J2EE容器中配置JNDI參数，定义一个数据源。也就是JDBC引用參数，给这个数据源设置一个名称；然后，在程序中，通过数据源名称引用数据源从而訪问后台数据库。 

//红色的字能够看出。JNDI是由j2ee容器提供的功能 

详细操作例如以下（以JBoss为例）： 
1、配置数据源 
在JBoss 的 D:\jboss420GA\docs\examples\jca 文件夹以下。有非常多不同数据库引用的数据源定义模板。

将当中的 mysql-ds.xml 文件Copy到你使用的server下，如 D:\jboss420GA\server\default\deploy。 
改动 mysql-ds.xml 文件的内容，使之能通过JDBC正确訪问你的MySQL数据库。例如以下： 


    
    
    1.  <?
    
        xml version="1.0" encoding="UTF-8"?>  
    
    2.  <datasources>  
    3.  <local-tx-datasource>  
    4.      <jndi-name>MySqlDS</jndi-name>  
    5.      <connection-url>jdbc:mysql://localhost:3306/lw</connection-url>  
    6.      <driver-class>com.mysql.jdbc.Driver</driver-class>  
    7.      <user-name>root</user-name>  
    8.      <password>rootpassword</password>  
    9.  <exception-sorter-class-name>  
    10.  org.jboss.resource.adapter.jdbc.vendor.MySQLExceptionSorter  
    11.  </exception-sorter-class-name>  
    12.      <metadata>  
    13.         <type-mapping>mySQL</type-mapping>  
    14.      </metadata>  
    15.  </local-tx-datasource>  
    16.  </datasources>  



这里，定义了一个名为MySqlDS的数据源。其參数包含JDBC的URL。驱动类名，username及密码等。 

2、在程序中引用数据源： 




    
    1.  Connection conn=null;  
    2.  try {  
    3.    Context ctx=new InitialContext();  
    4.    Object datasourceRef=ctx.lookup("java:MySqlDS"); //引用数据源  
    5.    DataSource ds=(Datasource)datasourceRef;  
    6.    conn=ds.getConnection();  
    7.    ......  
    8.    c.close();  
    9.  } catch(Exception e) {  
    10.    e.printStackTrace();  
    11.  } finally {  
    12.    if(conn!=null) {  
    13.      try {  
    14.        conn.close();  
    15.      } catch(SQLException e) { }  
    16.    }  
    17.  }  
    


直接使用JDBC或者通过JNDI引用数据源的编程代码量相差无几，可是如今的程序能够不用关心详细JDBC參数了。

//解藕了。可扩展了 
在系统部署后。假设数据库的相关參数变更。仅仅须要又一次配置 mysql-ds.xml 改动当中的JDBC參数，仅仅要保证数据源的名称不变，那么程序源码就无需改动。

由此可见。JNDI避免了程序与数据库之间的紧耦合，使应用更加易于配置、易于部署。

JNDI的扩展： 
JNDI在满足了数据源配置的要求的基础上。还进一步扩充了作用：全部与系统外部的资源的引用，都能够通过JNDI定义和引用。

//注意什么叫资源 

所以，在J2EE规范中，J2EE 中的资源并不局限于 JDBC 数据源。

引用的类型有非常多，当中包含资源引用（已经讨论过）、环境实体和 EJB 引用。

特别是 EJB 引用，它暴露了 JNDI 在 J2EE 中的另外一项关键角色：查找其它应用程序组件。 

EJB 的 JNDI 引用非常相似于 JDBC 资源的引用。在服务趋于转换的环境中，这是一种非常有效的方法。能够对应用程序架构中所得到的全部组件进行这类配置管理，从 EJB 组件到 JMS 队列和主题。再到简单配置字符串或其它对象。这能够降低随时间的推移服务变更所产生的维护成本，同一时候还能够简化部署，降低集成工作。外部资源”。

总结： 

J2EE 规范要求全部 J2EE 容器都要提供 JNDI 规范的实现。//sun 果然喜欢制定规范JNDI 在 J2EE 中的角色就是“交换机” —— J2EE 组件在执行时间接地查找其它组件、资源或服务的通用机制。在多数情况下，提供 JNDI 供应者的容器能够充当有限的数据存储。这样管理员就能够设置应用程序的执行属性，并让其它应用程序引用这些属性（Java 管理扩展（Java Management Extensions，JMX）也能够用作这个目的）。JNDI 在 J2EE 应用程序中的主要角色就是提供间接层，这样组件就能够发现所须要的资源，而不用了解这些间接性。 

在 J2EE 中，JNDI 是把 J2EE 应用程序合在一起的粘合剂。JNDI 提供的间接寻址同意跨企业交付可伸缩的、功能强大且非常灵活的应用程序。

这是 J2EE 的承诺，并且经过一些计划和预先考虑。这个承诺是全然能够实现的。 

 从上面的文章中能够看出： 
1、JNDI 提出的目的是为了解藕，是为了开发更加easy维护，easy扩展。easy部署的应用。 
2、JNDI 是一个sun提出的一个规范(相似于jdbc),详细的实现是各个j2ee容器提供商。sun   仅仅是要求，j2ee容器必须有JNDI这种功能。

3、JNDI 在j2ee系统中的角色是“交换机”，是J2EE组件在执行时间接地查找其它组件、资源或服务的通用机制。 
4、JNDI 是通过资源的名字来查找的，资源的名字在整个j2ee应用中(j2ee容器中)是唯一的。 


   上文提到过双亲委派模型并不是一个强制性的约束模型，而是 Java设计者推荐给开发者的类加载器实现方式。在Java 的世界中大部分的类加载器都遵循这个模型，但也有例外。

   双亲委派模型的一次“被破坏”是由这个模型自身的缺陷所导致的，双亲委派很好地解决了各个类加载器的基础类的统一问题（越基础的类由越上层的加载器进行加载），基础类之所以称为“基础”，是因为它们总是作为被用户代码调用的API ，但世事往往没有绝对的完美，如果基础类又要调用回用户的代码，那该怎么办？

这并非是不可能的事情，一个典型的例子便是JNDI 服务，JNDI现在已经是Java的标准服务，它的代码由启动类加载器去加载（在 JDK 1.3时放进去的rt.jar），但JNDI 的目的就是对资源进行集中管理和查找，它需要调用由独立厂商实现并部署在应用程序的Class Path下的JNDI 接口提供者（SPI，Service Provider Interface）的代码，但启动类加载器不可能“认识” 这些代码 ，因为启动类加载器的搜索范围中找不到用户应用程序类，那该怎么办？


为了解决这个问题，Java设计团队只好引入了一个不太优雅的设计：线程上下文类加载器（Thread Context ClassLoader）。这个类加载器可以通过java.lang.Thread类的setContextClassLoader()方法进行设置，如果创建线程时还未设置，它将会从父线程中继承一个，如果在应用程序的全局范围内都没有设置过的话，那这个类加载器默认就是应用程序类加载器（Application ClassLoader）。

   有了线程上下文类加载器，就可以做一些“舞弊”的事情了，JNDI服务使用这个线程上下文类加载器去加载所需要的 SPI代码，也就是父类加载器请求子类加载器去完成类加载的动作，这种行为实际上就是打通了双亲委派模型的层次结构来逆向使用类加载器 ，实际上已经违背了双亲委派模型的一般性原则，但这也是无可奈何的事情。Java中所有涉及SPI的加载动作基本上都采用这种方式，例如JNDI 、JDBC、JCE、 JAXB 和JBI等。

  

## OSGI

  

目前，业内关于OSGI技术的学习资源或者技术文档还是很少的。我在某宝网搜索了一下“OSGI”的书籍，结果倒是有，但是种类少的可怜，而且几乎没有人购买。
因为工作的原因我需要学习OSGI，所以我不得不想尽办法来主动学习OSGI。我将用文字记录学习OSGI的整个过程，通过整理书籍和视频教程，来让我更加了解这门技术，同时也让需要学习这门技术的同志们有一个清晰的学习路线。

我们需要解决一下几问题:

### 1.如何正确的理解和认识OSGI技术？

我们从外文资料上或者从翻译过来的资料上看到OSGi解释和定义，都是直译过来的，但是OSGI的真实意义未必是中文直译过来的意思。OSGI的解释就是Open Service Gateway Initiative，直译过来就是“开放的服务入口(网关)的初始化”，听起来非常费解，什么是服务入口初始化？

所以我们不去直译这个OSGI，我们换一种说法来描述OSGI技术。

我们来回到我们以前的某些开发场景中去，假设我们使用SSH(struts+spring+hibernate)框架来开发我们的Web项目，我们做产品设计和开发的时候都是分模块的，我们分模块的目的就是实现模块之间的“解耦”，更进一步的目的是方便对一个项目的控制和管理。
我们对一个项目进行模块化分解之后，我们就可以把不同模块交给不同的开发人员来完成开发，然后项目经理把大家完成的模块集中在一起，然后拼装成一个最终的产品。一般我们开发都是这样的基本情况。

那么我们开发的时候预计的是系统的功能，根据系统的功能来进行模块的划分，也就是说，这个产品的功能或客户的需求是划分的重要依据。

但是我们在开发过程中，我们模块之间还要彼此保持联系，比如A模块要从B模块拿到一些数据，而B模块可能要调用C模块中的一些方法(除了公共底层的工具类之外)。所以这些模块只是一种逻辑意义上的划分。

最重要的一点是，我们把最终的项目要去部署到tomcat或者jBoss的服务器中去部署。那么我们启动服务器的时候，能不能关闭项目的某个模块或功能呢？很明显是做不到的，一旦服务器启动，所有模块就要一起启动，都要占用服务器资源，所以关闭不了模块，假设能强制拿掉，就会影响其它的功能。

以上就是我们传统模块式开发的一些局限性。

我们做软件开发一直在追求一个境界，就是模块之间的真正“解耦”、“分离”，这样我们在软件的管理和开发上面就会更加的灵活，甚至包括给客户部署项目的时候都可以做到更加的灵活可控。但是我们以前使用SSH框架等架构模式进行产品开发的时候我们是达不到这种要求的。

所以我们“架构师”或顶尖的技术高手都在为模块化开发努力的摸索和尝试，然后我们的OSGI的技术规范就应运而生。

现在我们的OSGI技术就可以满足我们之前所说的境界:在不同的模块中做到彻底的分离，而不是逻辑意义上的分离，是物理上的分离，也就是说在运行部署之后都可以在不停止服务器的时候直接把某些模块拿下来，其他模块的功能也不受影响。

由此，OSGI技术将来会变得非常的重要，因为它在实现模块化解耦的路上，走得比现在大家经常所用的SSH框架走的更远。这个技术在未来大规模、高访问、高并发的Java模块化开发领域，或者是项目规范化管理中，会大大超过SSH等框架的地位。

现在主流的一些应用服务器，Oracle的weblogic服务器，IBM的WebSphere，JBoss，还有Sun公司的glassfish服务器，都对OSGI提供了强大的支持，都是在OSGI的技术基础上实现的。有那么多的大型厂商支持OSGI这门技术，我们既可以看到OSGI技术的重要性。所以将来OSGI是将来非常重要的技术。 

但是OSGI仍然脱离不了框架的支持，因为OSGI本身也使用了很多spring等框架的基本控件(因为要实现AOP依赖注入等功能)，但是哪个项目又不去依赖第三方jar呢？

  
   双亲委派模型的另一次“被破坏”是由于用户对程序动态性的追求而导致的，这里所说的“ 动态性”指的是当前一些非常“热门”的名词：代码热替换（HotSwap）、模块热部署（HotDeployment）等 ，说白了就是希望应用程序能像我们的计算机外设那样，接上鼠标、U盘，不用重启机器就能立即使用，鼠标有问题或要升级就换个鼠标，不用停机也不用重启。

   对于个人计算机来说，重启一次其实没有什么大不了的，但对于一些生产系统来说，关机重启一次可能就要被列为生产事故，这种情况下热部署就对软件开发者，尤其是企业级软件开发者具有很大的吸引力。Sun 公司所提出的JSR-294、JSR-277规范在与 JCP组织的模块化规范之争中落败给JSR-291（即 OSGi R4.2），虽然Sun不甘失去Java 模块化的主导权，独立在发展 Jigsaw项目，但目前OSGi已经成为了业界“ 事实上” 的Java模块化标准，而OSGi实现模块化热部署的关键则是它自定义的类加载器机制的实现。

   每一个程序模块（ OSGi 中称为Bundle）都有一个自己的类加载器，当需要更换一个Bundle 时，就把Bundle连同类加载器一起换掉以实现代码的热替换。

   在OSGi环境下，类加载器不再是双亲委派模型中的树状结构，而是进一步发展为更加复杂的网状结构，当收到类加载请求时，OSGi 将按照下面的顺序进行类搜索：

    1）将以java.*开头的类委派给父类加载器加载。
    
    2）否则，将委派列表名单内的类委派给父类加载器加载。
    
    3）否则，将Import列表中的类委派给 Export这个类的Bundle的类加载器加载。
    
    4）否则，查找当前Bundle的 Class Path，使用自己的类加载器加载。
    
    5）否则，查找类是否在自己的Fragment Bundle中，如果在，则委派给 Fragment Bundle的类加载器加载。
    
    6）否则，查找Dynamic Import列表的 Bundle，委派给对应Bundle的类加载器加载。
    
    7）否则，类查找失败。

   上面的查找顺序中只有开头两点仍然符合双亲委派规则，其余的类查找都是在平级的类加载器中进行的。

   只要有足够意义和理由，突破已有的原则就可认为是一种创新。正如OSGi中的类加载器并不符合传统的双亲委派的类加载器，并且业界对其为了实现热部署而带来的额外的高复杂度还存在不少争议，但在Java 程序员中基本有一个共识：OSGi中对类加载器的使用是很值得学习的，弄懂了OSGi的实现，就可以算是掌握了类加载器的精髓。


## Tomcat类加载器以及应用间class隔离与共享



Tomcat的用户一定都使用过其应用部署功能，无论是直接拷贝文件到webapps目录，还是修改server.xml以目录的形式部署，或者是增加虚拟主机，指定新的appBase等等。

但部署应用时，不知道你是否曾注意过这几点：

1.  如果在一个Tomcat内部署多个应用，甚至多个应用内使用了某个类似的几个不同版本，但它们之间却互不影响。这是如何做到的。

2.  如果多个应用都用到了某类似的相同版本，是否可以统一提供，不在各个应用内分别提供，占用内存呢。

3.  还有时候，在开发Web应用时，在pom.xml中添加了servlet-api的依赖，那实际应用的class加载时，会加载你的servlet-api 这个jar吗

以上提到的这几点，在Tomcat以及各类的应用服务器中，都是通过类加载器（ClasssLoader）来实现的。通过本文，你可以了解到Tomcat内部提供的各种类加载器，Web应用的class和资源等加载的方式，以及其内部的实现原理。在遇到类似问题时，更胸有成竹。

### 类加载器

Java语言本身，以及现在其它的一些基于JVM之上的语言(Groovy，Jython， Scala...)，都是在将代码编译生成class文件，以实现跨多平台，write once, run anywhere。最终的这些class文件，在应用中，又被加载到JVM虚拟机中，开始工作。而把class文件加载到JVM的组件，就是我们所说的类加载器。而对于类加载器的抽象，能面对更多的class数据提供形式，例如网络、文件系统等。

Java中常见的那个ClassNotFoundException和NoClassDefFoundError就是类加载器告诉我们的。

Servlet规范指出，容器用于加载Web应用内Servlet的class loader, 允许加载位于Web应用内的资源。但不允许重写java.*, javax.*以及容器实现的类。同时

每个应用内使用Thread.currentThread.getContextClassLoader()获得的类加载器，都是该应用区别于其它应用的类加载器等等。

根据Servlet规范，各个应用服务器厂商自行实现。所以像其他的一些应用服务器一样， Tomcat也提供了多种的类加载器，以便应用服务器内的class以及部署的Web应用类文件运行在容器中时，可以使用不同的class repositories。

在Java中，类加载器是以一种父子关系树来组织的。除Bootstrap外，都会包含一个parent 类加载器。(这里写parent 类加载器，而不是父类加载器，不是为了装X，是为了避免和Java里的父类混淆) 一般以类加载器需要加载一个class或者资源文件的时候，他会先委托给他的parent类加载器，让parent类加载器先来加载，如果没有，才再在自己的路径上加载。这就是人们常说的双亲委托，即把类加载的请求委托给parent。

但是...，这里需要注意一下

> 对于Web应用的类加载，和上面的双亲委托是有区别的。

   主流的Java Web服务器（也就是Web容器） ，如Tomcat、Jetty、WebLogic、WebSphere 或其他笔者没有列举的服务器，都实现了自己定义的类加载器（一般都不止一个）。因为一个功能健全的 Web容器，要解决如下几个问题：

   1）部署在同一个Web容器上 的两个Web应用程序所使用的Java类库可以实现相互隔离。这是最基本的需求，两个不同的应用程序可能会依赖同一个第三方类库的不同版本，不能要求一个类库在一个服务器中只有一份，服务器应当保证两个应用程序的类库可以互相独立使用。

   2）部署在同一个Web容器上 的两个Web应用程序所使用的Java类库可以互相共享 。这个需求也很常见，例如，用户可能有10个使用[spring](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Flink.juejin.im%2F%3Ftarget%3Dhttp%253A%252F%252Flib.csdn.net%252Fbase%252Fjavaee "Java EE知识库") 组织的应用程序部署在同一台服务器上，如果把10份Spring分别存放在各个应用程序的隔离目录中，将会是很大的资源浪费——这主要倒不是浪费磁盘空间的问题，而是指类库在使用时都要被加载到Web容器的内存，如果类库不能共享，虚拟机的方法区就会很容易出现过度膨胀的风险。

   3）Web容器需要尽可能地保证自身的安全不受部署的Web应用程序影响。目前，有许多主流的Java Web容器自身也是使用Java语言来实现的。因此，Web容器本身也有类库依赖的问题，一般来说，基于安全考虑，容器所使用的类库应该与应用程序的类库互相独立。

   4）支持JSP应用的Web容器，大多数都需要支持 HotSwap功能。我们知道，JSP文件最终要编译成Java Class才能由虚拟机执行，但JSP文件由于其纯文本存储的特性，运行时修改的概率远远大于第三方类库或程序自身的Class文件 。而且ASP、[PHP](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Flink.juejin.im%2F%3Ftarget%3Dhttp%253A%252F%252Flib.csdn.net%252Fbase%252Fphp "PHP知识库") 和JSP这些网页应用也把修改后无须重启作为一个很大的“优势”来看待 ，因此“主流”的Web容器都会支持JSP生成类的热替换 ，当然也有“非主流”的，如运行在生产模式（Production Mode）下的WebLogic服务器默认就不会处理JSP文件的变化。

   由于存在上述问题，在部署Web应用时，单独的一个Class Path就无法满足需求了，所以各种 Web容都“不约而同”地提供了好几个Class Path路径供用户存放第三方类库，这些路径一般都以“lib”或“classes ”命名。被放置到不同路径中的类库，具备不同的访问范围和服务对象，通常，每一个目录都会有一个相应的自定义类加载器去加载放置在里面的Java类库 。现在，就以Tomcat 容器为例，看一看Tomcat具体是如何规划用户类库结构和类加载器的。

   在Tomcat目录结构中，有3组目录（“/common/*”、“/server/*”和“/shared/*”）可以存放Java类库，另外还可以加上Web 应用程序自身的目录“/WEB-INF/*” ，一共4组，把Java类库放置在这些目录中的含义分别如下：

   ①放置在/common目录中：类库可被Tomcat和所有的 Web应用程序共同使用。

   ②放置在/server目录中：类库可被Tomcat使用，对所有的Web应用程序都不可见。

   ③放置在/shared目录中：类库可被所有的Web应用程序共同使用，但对Tomcat自己不可见。

   ④放置在/WebApp/WEB-INF目录中：类库仅仅可以被此Web应用程序使用，对 Tomcat和其他Web应用程序都不可见。

   为了支持这套目录结构，并对目录里面的类库进行加载和隔离，Tomcat自定义了多个类加载器，这些类加载器按照经典的双亲委派模型来实现，其关系如下图所示。

  

![](https://user-gold-cdn.xitu.io/2017/5/8/0dddae151e8fe1eba5db1a35d2b7b9b2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1) 

  

  



   上图中灰色背景的3个类加载器是JDK默认提供的类加载器，这3个加载器的作用已经介绍过了。而CommonClassLoader、CatalinaClassLoader、SharedClassLoader和WebappClassLoader则是Tomcat自己定义的类加载器，它们分别加载/common/*、/server/*、/shared/*和/WebApp/WEB-INF/*中的Java类库。其中WebApp类加载器和Jsp类加载器通常会存在多个实例，每一个Web应用程序对应一个WebApp类加载器，每一个JSP文件对应一个Jsp类加载器。

   从图中的委派关系中可以看出，CommonClassLoader能加载的类都可以被Catalina ClassLoader和SharedClassLoader使用，而CatalinaClassLoader和Shared  ClassLoader自己能加载的类则与对方相互隔离。WebAppClassLoader可以使用SharedClassLoader加载到的类，但各个WebAppClassLoader实例之间相互隔离。

   而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个.Class文件，它出现的目的就是为了被丢弃：当Web容器检测到JSP文件被修改时，会替换掉目前的JasperLoader的实例，并通过再建立一个新的Jsp类加载器来实现JSP文件的HotSwap功能。

   对于Tomcat的6.x版本，只有指定了tomcat/conf/catalina.properties配置文件的server.loader和share.loader项后才会真正建立Catalina ClassLoader和Shared ClassLoader的实例，否则在用到这两个类加载器的地方都会用Common ClassLoader的实例代替，而默认的配置文件中没有设置这两个loader项，所以Tomcat 6.x顺理成章地把/common、/server和/shared三个目录默认合并到一起变成一个/lib目录，这个目录里的类库相当于以前/common目录中类库的作用。

   这是Tomcat设计团队为了简化大多数的部署场景所做的一项改进，如果默认设置不能满足需要，用户可以通过修改配置文件指定server.loader和share.loader的方式重新启用Tomcat 5.x的加载器[架构](https://yq.aliyun.com/go/articleRenderRedirect?url=https%3A%2F%2Flink.juejin.im%2F%3Ftarget%3Dhttp%253A%252F%252Flib.csdn.net%252Fbase%252Farchitecture "大型网站架构知识库")。

    Tomcat加载器的实现清晰易懂，并且采用了官方推荐的“正统”的使用类加载器的方式。如果读者阅读完上面的案例后，能完全理解Tomcat设计团队这样布置加载器架构的用意，那说明已经大致掌握了类加载器“主流”的使用方式，那么笔者不妨再提一个问题让读者思考一下：前面曾经提到过一个场景，如果有10个Web应用程序都是用Spring来进行组织和管理的话，可以把Spring放到Common或Shared目录下让这些程序共享。

    Spring要对用户程序的类进行管理，自然要能访问到用户程序的类，而用户的程序显然是放在/WebApp/WEB-INF目录中的，那么被CommonClassLoader或SharedClassLoader加载的Spring如何访问并不在其加载范围内的用户程序呢？如果研究过虚拟机类加载器机制中的双亲委派模型，相信读者可以很容易地回答这个问题。

  分析：如果按主流的双亲委派机制，显然无法做到让父类加载器加载的类 去访问子类加载器加载的类，上面在类加载器一节中提到过通过线程上下文方式传播类加载器。

  答案是使用线程上下文类加载器来实现的，使用线程上下文加载器，可以让父类加载器请求子类加载器去完成类加载的动作。

  看spring源码发现，spring加载类所用的Classloader是通过Thread.currentThread().getContextClassLoader()来获取的，而当线程创建时会默认setContextClassLoader(AppClassLoader)，即线程上下文类加载器被设置为 AppClassLoader，spring中始终可以获取到这个AppClassLoader( 在 Tomcat里就是WebAppClassLoader)子类加载器来加载bean ，以后任何一个线程都可以通过 getContextClassLoader()获取到WebAppClassLoader来getbean 了 。

  

本篇博文内容取材自《深入理解Java虚拟机：JVM高级特性与最佳实践》

## 参考文章

<https://segmentfault.com/a/1190000009707894>

<https://www.cnblogs.com/hysum/p/7100874.html>

<http://c.biancheng.net/view/939.html>

<https://www.runoob.com/>

https://blog.csdn.net/android_hl/article/details/53228348

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
