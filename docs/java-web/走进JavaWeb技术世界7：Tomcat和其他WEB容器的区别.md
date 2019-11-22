# Table of Contents

  * [Tomcat和物理服务器的区别](#tomcat和物理服务器的区别)
    * [Tomcat：](#tomcat：)
    * [物理服务器：](#物理服务器：)
  * [详解tomcat 与 nginx，apache的区别及优缺点](#详解tomcat-与-nginx，apache的区别及优缺点)
    * [定义：](#定义：)
    * [区别](#区别)
    * [总结](#总结)
  * [微信公众号](#微信公众号)
    * [个人公众号：程序员黄小斜](#个人公众号：程序员黄小斜)
    * [技术公众号：Java技术江湖](#技术公众号：java技术江湖)


本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com


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
下面主要介绍下tomcat 与 nginx，apache的定义、区别及优缺点。

## Tomcat和物理服务器的区别

### Tomcat：

1.     本质：软件 Web 应用服务器----一个免费的开放源代码的Web 应用服务器，属于轻量级应用服务器，在中小型系统和并发访问用户不是很多的场合下被普遍使用，是开发和调试JSP 程序的首选。

2.     用途：

a．  当在一台机器（即物理服务器，也就是物理机）上配置好Apache 服务器，可利用它响应HTML页面的访问请求。实际上Tomcat是Apache 服务器的扩展，但运行时它是独立运行的，所以当你运行tomcat 时，它实际上作为一个与Apache 独立的进程单独运行的，Tomcat 实际上运行JSP 页面和Servlet

b． Tomcat和IIS等Web服务器一样，具有处理HTML页面的功能，另外它还是一个Servlet和JSP容器，独立的Servlet容器是Tomcat的默认模式。

### 物理服务器：

1．本质：硬件，也就是我们经常讲的服务器或者物理机，我们的PC就是一台性能较低的网络服务器，常见的有 云服务器（例如阿里云ECS）等

2．组成：处理器、硬盘、内存、系统总线等，和通用的计算机架构类似，但是由于需要提供高可靠的服务，因此在处理能力、稳定性、可靠性、安全性、可扩展性、可管理性等方面要求较高。


## 详解tomcat 与 nginx，apache的区别及优缺点


### 定义：

1\. Apache

Apache HTTP服务器是一个模块化的服务器，可以运行在几乎所有广泛使用的计算机平台上。其属于应用服务器。Apache支持支持模块多，性能稳定，Apache本身是静态解析，适合静态HTML、图片等，但可以通过扩展脚本、模块等支持动态页面等。

(Apche可以支持PHPcgiperl,但是要使用Java的话，你需要Tomcat在Apache后台支撑，将Java请求由Apache转发给Tomcat处理。) 缺点：配置相对复杂，自身不支持动态页面。

2\. Tomcat：

Tomcat是应用(Java)服务器，它只是一个Servlet(JSP也翻译成Servlet)容器，可以认为是Apache的扩展，但是可以独立于Apache运行。

3\. Nginx

Nginx是俄罗斯人编写的十分轻量级的HTTP服务器,Nginx，它的发音为“engine X”，是一个高性能的HTTP和反向代理服务器，同时也是一个IMAP/POP3/SMTP 代理服务器。

[![](https://server.zzidc.com/uploads/allimg/181211/1-1Q211140I2H2.jpg)](http://s1.51cto.com/oss/201812/11/0b70cbd49f22c2bfd4bf6cd5da29335f.jpg-wh_651x-s_227048833.jpg)

### 区别

1\. Apache与Tomcat的比较

相同点：

两者都是Apache组织开发的两者都有HTTP服务的功能两者都是免费的 不同点：

Apache是专门用了提供HTTP服务的，以及相关配置的(例如虚拟主机、URL转发等等)，而Tomcat是Apache组织在符合Java EE的JSP、Servlet标准下开发的一个JSP服务器.

[![](https://server.zzidc.com/uploads/allimg/181211/140P921b-0.jpg)](https://server.zzidc.com/uploads/allimg/181211/140P921b-0.jpg)

Apache是一个Web服务器环境程序,启用他可以作为Web服务器使用,不过只支持静态网页如(ASP,PHP,CGI,JSP)等动态网页的就不行。如果要在Apache环境下运行JSP的话就需要一个解释器来执行JSP网页,而这个JSP解释器就是Tomcat。

Apache:侧重于HTTPServer ，Tomcat:侧重于Servlet引擎，如果以Standalone方式运行，功能上与Apache等效，支持JSP，但对静态网页不太理想;

Apache是Web服务器，Tomcat是应用(Java)服务器，它只是一个Servlet(JSP也翻译成Servlet)容器，可以认为是Apache的扩展，但是可以独立于Apache运行。

实际使用中Apache与Tomcat常常是整合使用：

如果客户端请求的是静态页面，则只需要Apache服务器响应请求。 如果客户端请求动态页面，则是Tomcat服务器响应请求。 因为JSP是服务器端解释代码的，这样整合就可以减少Tomcat的服务开销。

可以理解Tomcat为Apache的一种扩展。

2\. Nginx与Apache比较

1) nginx相对于apache的优点

轻量级，同样起web 服务，比apache占用更少的内存及资源 抗并发，nginx 处理请求是异步非阻塞的，而apache 则是阻塞型的，在高并发下nginx 能保持低资源低消耗高性能高度模块化的设计，编写模块相对简单提供负载均衡

社区活跃，各种高性能模块出品迅速

2) apache 相对于nginx 的优点

apache的 rewrite 比nginx 的强大 ;

支持动态页面;

支持的模块多，基本涵盖所有应用;

性能稳定，而nginx相对bug较多。

3) 两者优缺点比较

Nginx 配置简洁, Apache 复杂 ;

Nginx 静态处理性能比 Apache 高 3倍以上 ;

Apache 对 PHP 支持比较简单，Nginx 需要配合其他后端用;Apache 的组件比 Nginx 多 ;

apache是同步多进程模型，一个连接对应一个进程;nginx是异步的，多个连接(万级别)可以对应一个进程;

nginx处理静态文件好,耗费内存少;

动态请求由apache去做，nginx只适合静态和反向;

Nginx适合做前端服务器，负载性能很好;

Nginx本身就是一个反向代理服务器 ，且支持负载均衡

### 总结

Nginx优点：负载均衡、反向代理、处理静态文件优势。nginx处理静态请求的速度高于apache;

Apache优点：相对于Tomcat服务器来说处理静态文件是它的优势，速度快。Apache是静态解析，适合静态HTML、图片等。

Tomcat：动态解析容器，处理动态请求，是编译JSPServlet的容器，Nginx有动态分离机制，静态请求直接就可以通过Nginx处理，动态请求才转发请求到后台交由Tomcat进行处理。

Apache在处理动态有优势，Nginx并发性比较好，CPU内存占用低，如果rewrite频繁，那还是Apache较适合。

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
