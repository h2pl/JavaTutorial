# Table of Contents

  * [Overview](#overview)
  * [Connector Init and Start](#connector-init-and-start)
  * [Request Process](#request-process)
    * [Acceptor](#acceptor)
    * [Poller](#poller)
    * [Worker](#worker)
    * [Container](#container)
  * [Reference](#reference)
  * [微信公众号](#微信公众号)
    * [个人公众号：程序员黄小斜](#个人公众号：程序员黄小斜)
    * [技术公众号：Java技术江湖](#技术公众号：java技术江湖)


本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《走进JavaWeb技术世界》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从servlet到框架，从ssm再到SpringBoot，一步步地学习JavaWeb基础知识，并上手进行实战，接着了解JavaWeb项目中经常要使用的技术和组件，包括日志组件、Maven、Junit，等等内容，以便让你更完整地了解整个Java Web技术体系，形成自己的知识框架。

为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

**文末赠送8000G的Java架构师学习资料，需要的朋友可以到文末了解领取方式，资料包括Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源）**
<!-- more -->

很多东西在时序图中体现的已经非常清楚了，没有必要再一步一步的作介绍，所以本文以图为主，然后对部分内容加以简单解释。

绘制图形使用的工具是 PlantUML + Visual Studio Code + PlantUML Extension

本文对 Tomcat 的介绍以 Tomcat-9.0.0.M22 为标准。

Tomcat-9.0.0.M22 是 Tomcat 目前最新的版本，但尚未发布，它实现了 Servlet4.0 及 JSP2.3 并提供了很多新特性，需要 1.8 及以上的 JDK 支持等等，详情请查阅 Tomcat-9.0-doc。

> https://tomcat.apache.org/tomcat-9.0-doc/index.html

## Overview

![](https://img-blog.csdnimg.cn/20190808094540456.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FsYmVuWGll,size_16,color_FFFFFF,t_70)

Connector 启动以后会启动一组线程用于不同阶段的请求处理过程。

1.  Acceptor 线程组。用于接受新连接，并将新连接封装一下，选择一个 Poller 将新连接添加到 Poller 的事件队列中。

2.  Poller 线程组。用于监听 Socket 事件，当 Socket 可读或可写等等时，将 Socket 封装一下添加到 worker 线程池的任务队列中。

3.  worker 线程组。用于对请求进行处理，包括分析请求报文并创建 Request 对象，调用容器的 pipeline 进行处理。

Acceptor、Poller、worker 所在的 ThreadPoolExecutor 都维护在 NioEndpoint 中。

## Connector Init and Start

![](https://img-blog.csdnimg.cn/20190808094650927.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FsYmVuWGll,size_16,color_FFFFFF,t_70)

1.  initServerSocket()，通过 ServerSocketChannel.open() 打开一个 ServerSocket，默认绑定到 8080 端口，默认的连接等待队列长度是 100， 当超过 100 个时会拒绝服务。我们可以通过配置 conf/server.xml 中 Connector 的 acceptCount 属性对其进行定制。

2.  createExecutor() 用于创建 Worker 线程池。默认会启动 10 个 Worker 线程，Tomcat 处理请求过程中，Woker 最多不超过 200 个。我们可以通过配置 conf/server.xml 中 Connector 的 minSpareThreads 和 maxThreads 对这两个属性进行定制。

3.  Pollor 用于检测已就绪的 Socket。默认最多不超过 2 个，Math.min(2,Runtime.getRuntime().availableProcessors());。我们可以通过配置 pollerThreadCount 来定制。

4.  Acceptor 用于接受新连接。默认是 1 个。我们可以通过配置 acceptorThreadCount 对其进行定制。

## Request Process

### Acceptor

![](https://img-blog.csdnimg.cn/20190808094725483.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FsYmVuWGll,size_16,color_FFFFFF,t_70)

1.  Acceptor 在启动后会阻塞在 ServerSocketChannel.accept(); 方法处，当有新连接到达时，该方法返回一个 SocketChannel。

2.  配置完 Socket 以后将 Socket 封装到 NioChannel 中，并注册到 Poller,值的一提的是，我们一开始就启动了多个 Poller 线程，注册的时候，连接是公平的分配到每个 Poller 的。NioEndpoint 维护了一个 Poller 数组，当一个连接分配给 pollers[index] 时，下一个连接就会分配给 pollers[(index+1)%pollers.length].

3.  addEvent() 方法会将 Socket 添加到该 Poller 的 PollerEvent 队列中。到此 Acceptor 的任务就完成了。

### Poller

![](https://img-blog.csdnimg.cn/20190808094749113.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FsYmVuWGll,size_16,color_FFFFFF,t_70)

1.  selector.select(1000)。当 Poller 启动后因为 selector 中并没有已注册的 Channel，所以当执行到该方法时只能阻塞。所有的 Poller 共用一个 Selector，其实现类是 sun.nio.ch.EPollSelectorImpl

2.  events() 方法会将通过 addEvent() 方法添加到事件队列中的 Socket 注册到 EPollSelectorImpl，当 Socket 可读时，Poller 才对其进行处理

3.  createSocketProcessor() 方法将 Socket 封装到 SocketProcessor 中，SocketProcessor 实现了 Runnable 接口。worker 线程通过调用其 run() 方法来对 Socket 进行处理。

4.  execute(SocketProcessor) 方法将 SocketProcessor 提交到线程池，放入线程池的 workQueue 中。workQueue 是 BlockingQueue 的实例。到此 Poller 的任务就完成了。

### Worker

![](https://img-blog.csdnimg.cn/20190808094814420.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FsYmVuWGll,size_16,color_FFFFFF,t_70)

*   worker 线程被创建以后就执行 ThreadPoolExecutor 的 runWorker() 方法，试图从 workQueue 中取待处理任务，但是一开始 workQueue 是空的，所以 worker 线程会阻塞在 workQueue.take() 方法。

*   当新任务添加到 workQueue后，workQueue.take() 方法会返回一个 Runnable，通常是 SocketProcessor,然后 worker 线程调用 SocketProcessor 的 run() 方法对 Socket 进行处理。

*   createProcessor() 会创建一个 Http11Processor, 它用来解析 Socket，将 Socket 中的内容封装到 Request 中。注意这个 Request 是临时使用的一个类，它的全类名是 org.apache.coyote.Request，

*   postParseRequest() 方法封装一下 Request，并处理一下映射关系(从 URL 映射到相应的 Host、Context、Wrapper)。

1.  CoyoteAdapter 将 Rquest 提交给 Container 处理之前，并将 org.apache.coyote.Request 封装到 org.apache.catalina.connector.Request，传递给 Container 处理的 Request 是 org.apache.catalina.connector.Request。

2.  connector.getService().getMapper().map()，用来在 Mapper 中查询 URL 的映射关系。映射关系会保留到 org.apache.catalina.connector.Request 中，Container 处理阶段 request.getHost() 是使用的就是这个阶段查询到的映射主机，以此类推 request.getContext()、request.getWrapper() 都是。

*   connector.getService().getContainer().getPipeline().getFirst().invoke() 会将请求传递到 Container 处理，当然了 Container 处理也是在 Worker 线程中执行的，但是这是一个相对独立的模块，所以单独分出来一节。

### Container

![](https://img-blog.csdnimg.cn/20190808094835789.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FsYmVuWGll,size_16,color_FFFFFF,t_70)

*   需要注意的是，基本上每一个容器的 StandardPipeline 上都会有多个已注册的 Valve，我们只关注每个容器的 Basic Valve。其他 Valve 都是在 Basic Valve 前执行。

*   request.getHost().getPipeline().getFirst().invoke() 先获取对应的 StandardHost，并执行其 pipeline。

*   request.getContext().getPipeline().getFirst().invoke() 先获取对应的 StandardContext,并执行其 pipeline。

*   request.getWrapper().getPipeline().getFirst().invoke() 先获取对应的 StandardWrapper，并执行其 pipeline。

*   最值得说的就是 StandardWrapper 的 Basic Valve，StandardWrapperValve

1.  allocate() 用来加载并初始化 Servlet，值的一提的是 Servlet 并不都是单例的，当 Servlet 实现了 SingleThreadModel 接口后，StandardWrapper 会维护一组 Servlet 实例，这是享元模式。当然了 SingleThreadModel在 Servlet 2.4 以后就弃用了。

2.  createFilterChain() 方法会从 StandardContext 中获取到所有的过滤器，然后将匹配 Request URL 的所有过滤器挑选出来添加到 filterChain 中。

3.  doFilter() 执行过滤链,当所有的过滤器都执行完毕后调用 Servlet 的 service() 方法。

## Reference

1.  《How Tomcat works》

    https://www.amazon.com/How-Tomcat-Works-Budi-Kurniawan/dp/097521280X

2.  《Tomcat 架构解析》– 刘光瑞

    http://product.dangdang.com/25084132.html

3.  Tomcat-9.0-doc

    https://tomcat.apache.org/tomcat-9.0-doc/index.html

4.  apache-tomcat-9.0.0.M22-src

    http://www-eu.apache.org/dist/tomcat/tomcat-9/v9.0.0.M22/src/

5.  tomcat架构分析 (connector NIO 实现)

    http://gearever.iteye.com/blog/1844203



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
