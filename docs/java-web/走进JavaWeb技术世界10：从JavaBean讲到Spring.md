# Table of Contents

  * [**Java Bean**](#java-bean)
  * [**JSP + Java Bean**](#jsp--java-bean)
  * [Enterprise Java bean](#enterprise-java-bean)
  * [Spring](#spring)
  * [JavaBean 和 Spring中Bean的区别](#javabean-和-spring中bean的区别)
    * [Jave bean](#jave-bean)
    * [spring bean](#spring bean)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
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


## **Java Bean**

Java语言欠缺属性、事件、多重继承功能。所以，如果要在Java程序中实现一些面向对象编程的常见需求，只能手写大量胶水代码。Java Bean正是编写这套胶水代码的惯用模式或约定。这些约定包括getXxx、setXxx、isXxx、addXxxListener、XxxEvent等。遵守上述约定的类可以用于若干工具或库。

举个例子，假如有人要用Java实现一个单向链表类，可能会这样写：





![](https://upload-images.jianshu.io/upload_images/9915352-8a35b591c3c78ed3.png)





上述实现为了能够快速获取链表的大小，把链表大小缓存在size变量中。用法如下：

JavaIntList myList = new JavaIntList( );

System.out.println(myList.size);

要节省内存，不要缓存size变量了，把代码改成这样：





![](https://upload-images.jianshu.io/upload_images/9915352-55f1e5d91f50eb68.png)





发现找不到什么size变量。如果要找到size变量，你就必须保持向后兼容性。所以Java标准库中，绝对不会出现public int size这样的代码，而一定会一开始就写成：

private int size;

public int getSize( ){return size;}

让用户一开始就使用getSize，以便有朝一日修改getSize实现时，不破坏向后兼容性。这种public int getSize() { return size; }的惯用手法，就是Java Bean。

## **JSP + Java Bean**

在jsp上，  可以用java bean 来封装业务逻辑，保存数据到数据库， 像这样：





![](https://upload-images.jianshu.io/upload_images/9915352-8793c939542d7910.png)





其中jsp 直接用来接受用户的请求， 然后通过java bean 来处理业务， 具体的使用方法是：

这就能把HTTP request中的所有参数都设置到 user 这个java bean 对应的属性上去。 

只要保证 http request中的参数名和 java bean 中的属性名是一样的。 

这个叫做JSP Model 1 的模型受到了很多Java程序员的欢迎 ,  因为他们的应用规模都很小， 用Model 1 使得开发很快速，实际上， 这种方式和微软的asp , 以及和开源的php 几乎一样。 

但在项目中频繁使用了Model 1 导致整个系统的崩溃，因为系统中有好几千个jsp， 这些jsp互相调用(通过GET/POST), 到了最后调用关系无人能搞懂。 

为了解决这个问题，又推出了 ：JSP Model 2 ,    这是个模型真正的体现了Model-View-Controller的思想：





![](https://upload-images.jianshu.io/upload_images/9915352-b5fe81e22fca9339.png)





Servlet 充当Controller ,  jsp 充当 View ，Java bean 当然就是Model 了！

业务逻辑， 页面显示， 和处理过程做了很好的分离。 

基于这个模型的扩展和改进，  很多Web开发框架开始如雨后春笋一样出现， 其中最著名的就是 SpringMVC了。

## Enterprise Java bean

越来越多企业程序员提出诉求：要分布式、要安全、要事务、要高可用性。

诉求可以归结为：“我们只想关注我们的业务逻辑， 我们不想， 也不应该由我们来处理‘低级’的事务， 多线程，连接池，以及其他各种各种的‘低级’API， 此外Java帝国一定得提供集群功能， 这样我们的一台机器死机以后，整个系统还能运转。 ”

于是推出了J2EE， 像Java bean 一样， 这还是一个规范， 但是比Java bean 复杂的多， 其中有：

**JDBC**:  Java 数据库连接

**JNDI** :  Java 命名和目录接口， 通过一个名称就可以定位到一个数据源， 连jdbc连接都不用了

**RMI**：  远程过程调用，  让一个机器上的java 对象可以调用另外一个机器上的java 对象 

**JMS** :   Java 消息服务，  可以使用消息队列了

**JTA**：  Java 事务管理， 支持分布式事务， 能在访问、更新多个数据库的时候，仍然保证事务， 还是分布式。

**Java mail** : 收发邮件

J2EE 后来改成了Java EE。

当然最重要的是， java bean 变成了 **Enterprise Java bean **, 简称 **EJB**。

使用了EJB， 你就可以把精力只放在业务上了， 那些烦人的事务管理， 安全管理，线程 统统交给容器（应用服务器）来处理吧。 

我们还提供了额外的福利， 只要你的应用服务器是由多个机器组成的集群， EJB就可以无缝的运行在这个集群上， 你完全不用考虑一个机器死掉了应用该怎么办。我们都帮你搞定了。 

使用Session Bean ， 可以轻松的处理你的业务。

使用实体Bean (Entity bean ) , 你和数据库打交道会变得极为轻松， 甚至sql 都不用写了。

使用消息驱动Bean(Message Driven bean ) , 你可以轻松的和一个消息队列连接， 处理消息。

## Spring

然而，大部分的程序员就发现，  EJB中用起来极为繁琐和笨重， 性能也不好， 为了获得所谓的分布式，反而背上了沉重的枷锁。 

实体Bean很快没人用了， 就连简单的无状态Session bean 也被大家所诟病， 其中一条罪状就是“代码的侵入性”。

在定义EJB的时候没考虑那么多，程序员在定义一个Session bean的时候，需要写一大堆和业务完全没有关系的类。 

还需要被迫实现一些根本不应该实现的接口及其方法： 





![](https://upload-images.jianshu.io/upload_images/9915352-535d649b27315b2a.png)





他们希望这个样子：

public class HelloworldBean{

    public String hello(){

        return "hello world"

   }

}

与此同时，他们还过分的要求保留事务、 安全这些必备的东西。 

Spring 框架顺应了POJO的潮流， 提供了一个spring 的容器来管理这些POJO, 也叫bean 。

对于一个Bean 来说，如果你依赖别的Bean , 只需要声明即可， spring 容器负责把依赖的bean 给“注入进去“， 起初大家称之为控制反转(IoC)。

后来 Martin flower 给这种方式起来个更好的名字，叫“依赖注入”（DI）。

如果一个Bean 需要一些像事务，日志，安全这样的通用的服务， 也是只需要声明即可， spring 容器在运行时能够动态的“织入”这些服务， 这叫面向切面（AOP）。 

总之，spring和spring mvc极大的增加了Java对web开发领地的统治力。

## JavaBean 和 Spring中Bean的区别
先了解一下各自是什么吧!

### Jave bean
javaBean简单的讲就是实体类，用来封装对象，这个类里面全部都是属性值，和get，set方法。简单笼统的说就是一个类，一个可复用的类。javaBean在MVC设计模型中是model，又称模型层，在一般的程序中，我们称它为数据层，就是用来设置数据的属性和一些行为，然后我会提供获取属性和设置属性的get/set方法JavaBean是一种JAVA语言写成的可重用组件。为写成JavaBean，类必须是具体的和公共的，并且具有无参数的构造器。

### spring bean
对于使用Spring框架的开发人员来说，我们主要做的主要有两件事情：①开发Bean;②配置Bean;而Spring帮我们做的就是根据配置文件来创建Bean实例，并调用Bean实例的方法来完成“依赖注入”，可以把Spring容器理解成一个大型工厂，Bean就是该工厂的产品，工厂(Spirng容器)里能生产出来什么样的产品（Bean），完全取决于我们在配置文件中的配置。其实就是根据配置文件产生对象,而不需要人为的手动去创造对象,降低了耦合.

用处不同：传统javabean更多地作为值传递参数，而spring中的bean用处几乎无处不在，任何组件都可以被称为bean。

写法不同：传统javabean作为值对象，要求每个属性都提供getter和setter方法；但spring中的bean只需为接受设值注入的属性提供setter方法。

javabean的写法:

    public class A{
    private String a;
    private void setA(String a){
    this.a = a;
    }
    private String getA(){
    return a;
    }
    }
    spring bean的写法
    
    <bean id="p1" class="com.zking.Pojo.Person" scope="prototype">
    //及时加载 加载你的xml配置文件
    ApplicationContext applicationContext = new ClassPathXmlApplicationContext("ApplicationContext.xml");
    //getbean输入你配置类的别名得到 person对象
     Person p = (Person) applicationContext.getBean("p1");
     
id是给这个对象定的别名 class是这个实体类的全路径名 根据配置文件来创建Bean实例，并调用Bean实例的方法 bean里面还有很多属性

生命周期不同：传统javabean作为值对象传递，不接受任何容器管理其生命周期；spring中的bean有spring管理其生命周期行为。

所有可以被spring容器实例化并管理的java类都可以称为bean。

原来服务器处理页面返回的值都是直接使用request对象，后来增加了javabean来管理对象，所有页面值只要是和javabean对应，就可以用类.GET属性方法来获取值。javabean不只可以传参数，也可以处理数据，相当与把一个服务器执行的类放到了页面上，使对象管理相对不那么乱（对比asp的时候所有内容都在页面上完成）。

spring中的bean，是通过配置文件、javaconfig等的设置，有spring自动实例化，用完后自动销毁的对象。让我们只需要在用的时候使用对象就可以，不用考虑如果创建类对象（这就是spring的注入）。一般是用在服务器端代码的执行上。

## 参考文章
微信公众号【码农翻身】
https://blog.csdn.net/hmh13548571896/article/details/100628104
https://www.cnblogs.com/xll1025/p/11366413.html
https://blog.csdn.net/qqqnzhky/article/details/82747333
https://www.cnblogs.com/mike-mei/p/9712836.html
https://blog.csdn.net/qq_42245219/article/details/82748460

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
