# 目录
  * [servlet和jsp的区别](#servlet和jsp的区别)
  * [servlet和jsp各自的特点](#servlet和jsp各自的特点)
  * [通过MVC双剑合璧](#通过mvc双剑合璧)
  * [JavaWeb基础知识](#javaweb基础知识)
    * [一、Servlet 是什么？](#一、servlet-是什么？)
    * [二、Servlet的生命周期](#二、servlet的生命周期)
      * [init() 方法](#init-方法)
      * [service() 方法](#service-方法)
      * [destroy() 方法](#destroy-方法)
  * [相关面试题](#相关面试题)
    * [怎样理解Servlet的单实例多线程？**](#怎样理解servlet的单实例多线程？)
    * [JSP的中存在的多线程问题：](#jsp的中存在的多线程问题：)
    * [如何开发线程安全的Servlet](#如何开发线程安全的servlet)
  * [同步对共享数据的操作](#同步对共享数据的操作)
  * [五、servlet与jsp的区别](#五、servlet与jsp的区别)
  * [参考文章](#参考文章)


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

**文末赠送8000G的Java架构师学习资料，需要的朋友可以到文末了解领取方式，资料包括Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源）**
<!-- more -->

jsp作为Servlet技术的扩展，经常会有人将jsp和Servlet搞混。本文，将为大家带来servlet和jsp的区别，希望对大家有所帮助。

## servlet和jsp的区别

1、Servlet在Java代码中可以通过HttpServletResponse对象动态输出HTML内容。

2、JSP是在静态HTML内容中嵌入Java代码，然后Java代码在被动态执行后生成HTML内容。

## servlet和jsp各自的特点

1、Servlet虽然能够很好地组织业务逻辑代码，但是在Java源文件中，因为是通过字符串拼接的方式生成动态HTML内容，这样就容易导致代码维护困难、可读性差。

2、JSP虽然规避了Servlet在生成HTML内容方面的劣势，但是在HTML中混入大量、复杂的业务逻辑。

## 通过MVC双剑合璧

JSP和Servlet都有自身的适用环境，那么有没有什么办法能够让它们发挥各自的优势呢？答案是肯有的，MVC模式就能够完美解决这一问题。

MVC模式，是Model-View-Controller的简称，是软件工程中的一种软件架构模式，分为三个基本部分，分别是：模型（Model）、视图（View）和控制器（Controller）：

Controller——负责转发请求，对请求进行处理

View——负责界面显示

Model——业务功能编写（例如算法实现）、数据库设计以及数据存取操作实现

在JSP/Servlet开发的软件系统中，这三个部分的描述如下所示：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405151458.png)

1、Web浏览器发送HTTP请求到服务端，然后被Controller(Servlet)获取并进行处理（例如参数解析、请求转发）

2、Controller(Servlet)调用核心业务逻辑——Model部分，获得结果

3、Controller(Servlet)将逻辑处理结果交给View（JSP），动态输出HTML内容

4、动态生成的HTML内容返回到浏览器显示

MVC模式在Web开发中有很大的优势，它完美规避了JSP与Servlet各自的缺点，让Servlet只负责业务逻辑部分，而不会生成HTML代码；同时JSP中也不会充斥着大量的业务代码，这样能大提高了代码的可读性和可维护性。

## JavaWeb基础知识

### 一、Servlet 是什么？

[Java](http://lib.csdn.net/base/java "Java 知识库")Servlet 是运行在 Web 服务器或应用服务器上的程序，它是作为来自 Web 浏览器或其他 HTTP 客户端的请求和 HTTP 服务器上的[数据库](http://lib.csdn.net/base/mysql "MySQL知识库")或应用程序之间的中间层。

使用 Servlet，您可以收集来自网页表单的用户输入，呈现来自数据库或者其他源的记录，还可以动态创建网页。

[Java](http://lib.csdn.net/base/javase "Java SE知识库")Servlet 通常情况下与使用 CGI（Common Gateway Interface，公共网关接口）实现的程序可以达到异曲同工的效果。但是相比于 CGI，Servlet 有以下几点优势：

*   **1、性能明显更好。**
*   **2、Servlet 在 Web 服务器的地址空间内执行。这样它就没有必要再创建一个单独的进程来处理每个客户端请求。**
*   **3、Servlet 是独立于平台的，因为它们是用 Java 编写的。**
*   **4、服务器上的 Java 安全管理器执行了一系列限制，以保护服务器计算机上的资源。因此，Servlet 是可信的。**
*   **5、Java 类库的全部功能对 Servlet 来说都是可用的。它可以通过 sockets 和 RMI 机制与 applets、数据库或其他软件进行交互。**

### 二、Servlet的生命周期

Servlet 生命周期可被定义为从创建直到毁灭的整个过程。以下是 Servlet 遵循的过程：

*   **1、Servlet 通过调用init ()方法进行初始化。**
*   **2、Servlet 调用service()方法来处理客户端的请求。**
*   **3、Servlet 通过调用destroy()方法终止（结束）。**
*   **4、最后，Servlet 是由 JVM 的垃圾回收器进行垃圾回收的。**

#### init() 方法

init 方法被设计成只调用一次。它在第一次创建 Servlet 时被调用，在后续每次用户请求时不再调用。因此，它是用于一次性初始化，就像 Applet 的 init 方法一样。

Servlet 创建于用户第一次调用对应于该 Servlet 的 URL 时，但是您也可以指定 Servlet 在服务器第一次启动时被加载。

#### service() 方法

service() 方法是执行实际任务的主要方法。Servlet 容器（即 Web 服务器）调用 service() 方法来处理来自客户端（浏览器）的请求，并把格式化的响应写回给客户端。

每次服务器接收到一个 Servlet 请求时，服务器会产生一个新的线程并调用服务。service() 方法检查 HTTP 请求类型（GET、POST、PUT、DELETE 等），并在适当的时候调用 doGet、doPost、doPut，doDelete 等方法。

#### destroy() 方法

destroy() 方法只会被调用一次，在 Servlet 生命周期结束时被调用。destroy() 方法可以让您的 Servlet 关闭数据库连接、停止后台线程、把 Cookie 列表或点击计数器写入到磁盘，并执行其他类似的清理活动。

在调用 destroy() 方法之后，servlet 对象被标记为垃圾回收。

执行后：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405151559.png)

以后继续请求时：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405151623.png)

可见，就绪请求时只有service()方法执行！
## 相关面试题
### 怎样理解Servlet的单实例多线程？**

**不同的用户同时对同一个业务（如注册）发出请求，那这个时候容器里产生的有是几个servlet实例呢？**

答案是：**只有一个servlet实例**。一个servlet是在第一次被访问时加载到内存并实例化的。同样的业务请求共享一个servlet实例。不同的业务请求一般对应不同的servlet。

由于Servlet/JSP默认是以多线程模式执行的，所以，在编写代码时需要非常细致地考虑多线程的安全性问题。

### JSP的中存在的多线程问题：

当客户端第一次请求某一个JSP文件时，服务端把该JSP编译成一个CLASS文件，并创建一个该类的实例，然后创建一个线程处理CLIENT端的请求。如果有多个客户端同时请求该JSP文件，则服务端会创建多个线程。每个客户端请求对应一个线程。以多线程方式执行可大大降低对系统的资源需求,提高系统的并发量及响应时间。

对JSP中可能用的的变量说明如下:

**实例变量**: 实例变量是在堆中分配的,并被属于该实例的所有线程共享，所以**不是线程安全的。**

JSP系统提供的8个类变量

JSP中用到的**OUT,REQUEST,RESPONSE,SESSION,CONFIG,PAGE,PAGECONXT是线程安全的**(因为每个线程对应的request，respone对象都是不一样的，不存在共享问题),**APPLICATION在整个系统内被使用,所以不是线程安全的。**

**局部变量**: 局部变量在堆栈中分配,因为每个线程都有它自己的堆栈空间,所以**是线程安全的**

**静态类**: 静态类不用被实例化,就可直接使用,也**不是线程安全的**

**外部资源**: 在程序中可能会有多个线程或进程同时操作同一个资源(如:多个线程或进程同时对一个文件进行写操作).此时也要注意同步问题.

Servlet单实例多线程机制：

Servlet采用多线程来处理多个请求同时访问。servlet依赖于一个线程池来服务请求。线程池实际上是一系列的工作者线程集合。Servlet使用一个调度线程来管理工作者线程。

当容器收到一个Servlet请求，调度线程从线程池中选出一个工作者线程,将请求传递给该工作者线程，然后由该线程来执行Servlet的service方法。

当这个线程正在执行的时候,容器收到另外一个请求,调度线程同样从线程池中选出另一个工作者线程来服务新的请求,容器并不关心这个请求是否访问的是同一个Servlet.当容器同时收到对同一个Servlet的多个请求的时候，那么这个Servlet的service()方法将在多线程中并发执行。

Servlet容器默认采用单实例多线程的方式来处理请求，这样减少产生Servlet实例的开销，提升了对请求的响应时间，对于Tomcat可以在server.xml中通过<Connector>元素设置线程池中线程的数目。

### 如何开发线程安全的Servlet

1、实现 SingleThreadModel 接口

该接口指定了系统如何处理对同一个Servlet的调用。如果一个Servlet被这个接口指定,那么在这个Servlet中的service方法将不会有两个线程被同时执行，当然也就不存在线程安全的问题。这种方法只要将前面的Concurrent Test类的类头定义更改为：

````

Public class Concurrent Test extends HttpServlet implements SingleThreadModel { 
………… 
}  
````

## 同步对共享数据的操作
使用synchronized 关键字能保证一次只有一个线程可以访问被保护的区段

避免使用实例变量

本实例中的线程安全问题是由实例变量造成的，只要在Servlet里面的任何方法里面都不使用实例变量，那么该Servlet就是线程安全的。

1) Struts2的Action是原型，非单实例的；会对每一个请求,产生一个Action的实例来处理

Struts1 Action是单实例的

mvc的controller也是如此。因此开发时要求必须是线程安全的，因为仅有Action的一个实例来处理所有的请求。单例策略限制了Struts1 Action能作的事，并且要在开发时特别小心。Action资源必须是线程安全的或同步的。

2) Struts1的Action,Spring的Ioc容器管理的bean 默认是单实例的.

Spring的Ioc容器管理的bean 默认是单实例的。

Struts2 Action对象为每一个请求产生一个实例，因此没有线程安全问题。（实际上，servlet容器给每个请求产生许多可丢弃的对象，并且不会导致性能和垃圾回收问题）。

当Spring管理Struts2的Action时，bean默认是单实例的，可以通过配置参数将其设置为原型。(scope="prototype ）

## 五、servlet与jsp的区别


1.jsp经编译后就变成了Servlet.(JSP的本质就是Servlet，JVM只能识别java的类，不能识别JSP的代码,Web容器将JSP的代码编译成JVM能够识别的java类)

2.jsp更擅长表现于页面显示,servlet更擅长于逻辑控制.

3.Servlet中没有内置对象，内置对象都是必须通过HttpServletRequest对象，HttpServletResponse对象以及HttpServlet对象得到.Jsp是Servlet的一种简化，使用Jsp只需要完成程序员需要输出到客户端的内容，Jsp中的Java脚本如何镶嵌到一个类中，由Jsp容器完成。而Servlet则是个完整的Java类，这个类的Service方法用于生成对客户端的响应。

4.对于静态HTML标签，Servlet都必须使用页面输出流逐行输出


## 参考文章

https://www.w3cschool.cn/servlet/servlet-sxoy2p19.html
https://blog.csdn.net/qq_19782019/article/details/80292110
https://blog.csdn.net/qiuhuang_123/article/details/83617647
https://blog.csdn.net/zt15732625878/article/details/79951933
https://blog.csdn.net/android_hl/article/details/53228348

