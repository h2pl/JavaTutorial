# Table of Contents

  * [Java日志系统的演变史](#java日志系统的演变史)
    * [阶段一](#阶段一)
    * [阶段二](#阶段二)
    * [阶段三](#阶段三)
    * [阶段四](#阶段四)
    * [阶段五](#阶段五)
  * [一、日志框架的分类](#一、日志框架的分类)
  * [二、发展历程](#二、发展历程)
    * [Log4j](#log4j)
    * [J.U.L](#jul)
    * [JCL（commons-logging）](#jcl（commons-logging）)
    * [SLF4J & Logback](#slf4j--logback)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [个人公众号：程序员黄小斜](#个人公众号：程序员黄小斜)
    * [技术公众号：Java技术江湖](#技术公众号：java技术江湖)


本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《走进JavaWeb技术世界》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从servlet到框架，从ssm再到SpringBoot，一步步地学习JavaWeb基础知识，并上手进行实战，接着了解JavaWeb项目中经常要使用的技术和组件，包括日志组件、Maven、Junit，等等内容，以便让你更完整地了解整个JavaWeb技术体系，形成自己的知识框架。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

**文末赠送8000G的Java架构师学习资料，需要的朋友可以到文末了解领取方式，资料包括Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源）**
<!-- more -->

##  Java日志系统的演变史

我们先看一个故事。项目经理A带着一帮兄弟开发了一套复杂的企业ERP系统，这个系统一连开发了好几年，开发人员也换了好几拨。

### 阶段一
最开始的时候，项目经理A安排小B在系统中添加日志功能，在控制台上打印一些必要的信息。最开始的时候，由于项目的功能比较少，于是小B就是用System.out.println的方式打印日志信息。经理A感觉这样使用比较方便，也便于项目小组人员的使用，于是就沿用了下来。

### 阶段二
此时小B被借调到其他项目，小C加入到了项目组中。此时项目经理A要求改造日志系统，要求能把日志写到一个文件中，方便以后分析用户行为。小C在查看了以前的日志方式之后，感觉特别low，于是自己写了一个日志框架，命名为xiaoC-logging.jar，此举收到了项目经理A的好评。





![](https://upload-images.jianshu.io/upload_images/11968147-7072fc51b9dfc62d.png?imageMogr2/auto-orient/strip|imageView2/2/w/472/format/webp)




### 阶段三
项目组中加入了一个大牛老D，老D发现xiaoC-logging.jar这个日志框架虽然可以满足基本的日志要求，但是还不够高大上，没有一些诸如自动归档，异步写入文件，把日志文件写入NoSQL数据库中等功能。于是老D开发了一个更高级的日志框架叫oldD-logging.jar。

### 阶段四
oldD-logging.jar开发完成之后，需要把原来的xiaoC-logging.jar中的日志API做修改，把之前的日志实现写下来，换上高大上的oldD-logging.jar。

### 阶段五
在这个卸载与上新的过程中，老D的工作量陡增，他感觉很累。不过姜还是老的辣，他参考了JDBC和spring中面向接口的编程方式，制定了一个日志的门面（一系列的接口），以后所有的日志的记录，都只面向接口编程，至于今后怎么去实现，都要遵循这个接口就可以了。 


那么在JAVA开发中，这正的日志系统是怎么演变的呢？简短地描述下日志发展，最先出现的是apache开源社区的log4j，这个日志确实是应用最广泛的日志工具，成为了java日志的事实上的标准。然而，当时Sun公司在jdk1.4中增加了JUL日志实现，企图对抗log4j，但是却造成了混乱，这个也是被人诟病的一点。当然也有其他日志工具的出现，这样必然造成开发者的混乱，因为这些日志系统互相没有关联，替换和统一也就变成了比较棘手的一件事。想象下你的应用使用log4j，然后使用了一个其他团队的库，他们使用了JUL，你的应用就得使用两个日志系统了，然后又有第二个库出现了，使用了simplelog。





![](https://upload-images.jianshu.io/upload_images/11968147-b042d013f85e993f.png?imageMogr2/auto-orient/strip|imageView2/2/w/441/format/webp)





这个时候估计让你崩溃了，这是要闹哪样？这个状况交给你来想想办法，你该如何解决呢？进行抽象，抽象出一个接口层，对每个日志实现都适配或者转接，这样这些提供给别人的库都直接使用抽象层即可。不错，开源社区提供了commons-logging抽象，被称为JCL，也就是日志框架了，确实出色地完成了兼容主流的日志实现（log4j、JUL、simplelog），基本一统江湖，就连顶顶大名的spring也是依赖了JCL。

看起来事物确实是美好，但是美好的日子不长，接下来另一个优秀的日志框架slf4j的加入导致了更加混乱的场面。比较巧的是slf4j的作者(Ceki Gülcü)就是log4j的作者，他觉得JCL不够优秀，所以他要自己搞一套更优雅的出来，于是slf4j日志体系诞生了，并为slf4j实现了一个亲子——logback，确实更加优雅，但是由于之前很多代码库已经使用JCL，虽然出现slf4j和JCL之间的桥接转换，但是集成的时候问题依然多多，对很多新手来说确实会很懊恼，因为比单独的log4j时代“复杂”多了，抱怨声确实很多。

到此本来应该完了，但是Ceki Gülcü觉得还是得回头拯救下自己的“大阿哥”——log4j，于是log4j2诞生了，同样log4j2也参与到了slf4j日志体系中，想必将来会更加混乱。接下来详细解读日志系统的配合使用问题。slf4j的设计确实比较优雅，采用比较熟悉的方式——接口和实现分离，有个纯粹的接口层——slf4j-api工程，这个里边基本完全定义了日志的接口，所以对于开发来说，只需要使用这个即可。

有接口就要有实现，比较推崇的实现是logback，logback完全实现了slf4j-api的接口，并且性能也比log4j更好，同时实现了变参占位符日志输出方式等等新特性。刚刚也提到log4j的使用比较普遍，所以支持这批用户依然是必须的，slf4j-log4j12也实现了slf4j-api，这个算是对log4j的适配器。同样推理，也会有对JUL的适配器slf4j-jdk14等等。为了使使用JCL等等其他日志系统后者实现的用户可以很简单地切换到slf4j上来，给出了各种桥接工程，比如：jcl-over-slf4j会把对JCL的调用都桥接到slf4j上来，可以看出jcl-over-slf4j的api和JCL是相同的，所以这两个jar是不能共存的。jul-to-slf4j是把对jul的调用桥接到slf4j上，log4j-over-slf4j是把对log4j的调用桥接到slf4j。


## 一、日志框架的分类

*   门面型日志框架：

1.  JCL：　　Apache基金会所属的项目，是一套Java日志接口，之前叫Jakarta Commons Logging，后更名为Commons Logging
2.  SLF4J：  是一套简易Java日志门面，**本身并无日志的实现**。（Simple Logging Facade for Java，缩写Slf4j）

*   记录型日志框架:

1.  JUL：　　JDK中的日志记录工具，也常称为JDKLog、jdk-logging，自Java1.4以来的官方日志实现。
2.  Log4j：　 一个具体的日志实现框架。
3.  Log4j2：   一个具体的日志实现框架，是LOG4J1的下一个版本，与Log4j 1发生了很大的变化，Log4j 2不兼容Log4j 1。
4.  Logback：一个具体的日志实现框架，和Slf4j是同一个作者，但其性能更好。

　　　　　　　　　　　　　　![](https://img2018.cnblogs.com/blog/1577453/201908/1577453-20190801222005588-1535811596.png)







## 二、发展历程

要搞清楚它们的关系，就要从它们是在什么情况下产生的说起。我们按照时间的先后顺序来介绍。

### Log4j

在JDK 1.3及以前，Java打日志依赖System.out.println(), System.err.println()或者e.printStackTrace()，Debug日志被写到STDOUT流，错误日志被写到STDERR流。这样打日志有一个非常大的缺陷，即无法定制化，且日志粒度不够细。
于是， Gülcü 于2001年发布了Log4j，后来成为Apache 基金会的顶级项目。Log4j 在设计上非常优秀，对后续的 Java Log 框架有长久而深远的影响，它定义的Logger、Appender、Level等概念如今已经被广泛使用。Log4j 的短板在于性能，在Logback 和 Log4j2 出来之后，Log4j的使用也减少了。

### J.U.L

受Logj启发，Sun在Java1.4版本中引入了java.util.logging，但是j.u.l功能远不如log4j完善，开发者需要自己编写Appenders（Sun称之为Handlers），且只有两个Handlers可用（Console和File），j.u.l在Java1.5以后性能和可用性才有所提升。

### JCL（commons-logging）

由于项目的日志打印必然选择两个框架中至少一个，这时候，Apache的JCL（commons-logging）诞生了。JCL 是一个Log Facade，只提供 Log API，不提供实现，然后有 Adapter 来使用 Log4j 或者 JUL 作为Log Implementation。
在程序中日志创建和记录都是用JCL中的接口，在真正运行时，会看当前ClassPath中有什么实现，如果有Log4j 就是用 Log4j, 如果啥都没有就是用 JDK 的 JUL。
这样，在你的项目中，还有第三方的项目中，大家记录日志都使用 JCL 的接口，然后最终运行程序时，可以按照自己的需求(或者喜好)来选择使用合适的Log Implementation。如果用Log4j, 就添加 Log4j 的jar包进去，然后写一个 Log4j 的配置文件；如果喜欢用JUL，就只需要写个 JUL 的配置文件。如果有其他的新的日志库出现，也只需要它提供一个Adapter，运行的时候把这个日志库的 jar 包加进去。
不过，commons-logging对Log4j和j.u.l的配置问题兼容的并不好，使用commons-loggings还可能会遇到类加载问题，导致NoClassDefFoundError的错误出现。



　　　　　　　　　　　　　　　　　　　　　　　　![](https://img2018.cnblogs.com/blog/1577453/201908/1577453-20190801215840541-1005764017.png)





到这个时候一切看起来都很简单，很美好。接口和实现做了良好的分离，在统一的JCL之下，不改变任何代码，就可以通过配置就换用功能更强大，或者性能更好的日志库实现。

这种简单美好一直持续到SLF4J出现。

### SLF4J & Logback

SLF4J（Simple Logging Facade for Java）和 Logback 也是Gülcü 创立的项目，目的是为了提供更高性能的实现。
从设计模式的角度说，SLF4J 是用来在log和代码层之间起到门面作用，类似于 JCL 的 Log Facade。对于用户来说只要使用SLF4J提供的接口，即可隐藏日志的具体实现，SLF4J提供的核心API是一些接口和一个LoggerFactory的工厂类，用户只需按照它提供的统一纪录日志接口，最终日志的格式、纪录级别、输出方式等可通过具体日志系统的配置来实现，因此可以灵活的切换日志系统。

Logback是log4j的升级版，当前分为三个目标模块：

*   logback-core：核心模块，是其它两个模块的基础模块
*   logback-classic：是log4j的一个改良版本，同时完整实现 SLF4J API 使你可以很方便地更换成其它日记系统如log4j 或 JDK14 Logging
*   logback-access：访问模块与Servlet容器集成提供通过Http来访问日记的功能，是logback不可或缺的组成部分

Logback相较于log4j有更多的优点：

*   更快的执行速度
*   更充分的测试
*   logback-classic 非常自然的实现了SLF4J
*   使用XML配置文件或者Groovy
*   自动重新载入配置文件
*   优雅地从I/O错误中恢复
*   自动清除旧的日志归档文件
*   自动压缩归档日志文件
*   谨慎模式
*   Lilith
*   配置文件中的条件处理
*   更丰富的过滤

更详细的解释参见官网：[https://logback.qos.ch/reasonsToSwitch.html](https://link.jianshu.com/?t=https%3A%2F%2Flogback.qos.ch%2FreasonsToSwitch.html)

到这里，你可能会问：Apache 已经有了个JCL，用来做各种Log lib统一的接口，如果 Gülcü 要搞一个更好的 Log 实现的话，直接写一个实现就好了，为啥还要搞一个和SLF4J呢?

原因是Gülcü 认为 JCL 的 API 设计得不好，容易让使用者写出性能有问题的代码。关于这点，你可以参考这篇文章获得更详细的介绍：[https://zhuanlan.zhihu.com/p/24272450](https://link.jianshu.com/?t=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F24272450)

现在事情就变复杂了。我们有了两个流行的 Log Facade，以及三个流行的 Log Implementation。Gülcü 是个追求完美的人，他决定让这些Log之间都能够方便的互相替换，所以做了各种 Adapter 和 Bridge 来连接:



　　　　　　　　　　　　　　![](https://img2018.cnblogs.com/blog/1577453/201908/1577453-20190801220018444-1557580371.png)





可以看到甚至 Log4j 和 JUL 都可以桥接到SLF4J，再通过 SLF4J 适配到到 Logback！需要注意的是不能有循环的桥接，比如下面这些依赖就不能同时存在:

*   jcl-over-slf4j 和 slf4j-jcl
*   log4j-over-slf4j 和 slf4j-log4j12
*   jul-to-slf4j 和 slf4j-jdk14

然而，事情在变得更麻烦！

Log4j2

现在有了更好的 SLF4J 和 Logback，慢慢取代JCL 和 Log4j ，事情到这里总该大统一圆满结束了吧。然而维护 Log4j 的人不这样想，他们不想坐视用户一点点被 SLF4J / Logback 蚕食，继而搞出了 Log4j2。

Log4j2 和 Log4j1.x 并不兼容，设计上很大程度上模仿了 SLF4J/Logback，性能上也获得了很大的提升。Log4j2 也做了 Facade/Implementation 分离的设计，分成了 log4j-api 和 log4j-core。

现在好了，我们有了三个流行的Log 接口和四个流行的Log实现，如果画出桥接关系的图来回事什么样子呢?



　　　　　　　　　　　　![](https://img2018.cnblogs.com/blog/1577453/201908/1577453-20190801220108556-715466336.png)
看到这里是不是感觉有点晕呢？是的，我也有这种感觉。同样，在添加依赖的时候，要小心不要有循环依赖。









## 参考文章

<https://segmentfault.com/a/1190000009707894>

<https://www.cnblogs.com/hysum/p/7100874.html>

<http://c.biancheng.net/view/939.html>

<https://www.runoob.com/>

https://blog.csdn.net/android_hl/article/details/53228348

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
