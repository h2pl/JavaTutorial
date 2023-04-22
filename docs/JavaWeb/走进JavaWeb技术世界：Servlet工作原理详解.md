# 目录
  * [什么是Servlet](#什么是servlet)
  * [Servlet体系结构](#servlet体系结构)
  * [Servlet工作原理](#servlet工作原理)
  * [Servlet生命周期](#servlet生命周期)
  * [Servlet中的Listener](#servlet中的listener)
* [Cookie与Session](#cookie与session)
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
## 什么是Servlet

Servlet的作用是**为Java程序提供一个统一的web应用的规范**，方便程序员统一的使用这种规范来编写程序，应用容器可以使用提供的规范来实现自己的特性。比如tomcat的代码和jetty的代码就不一样，但作为程序员你只需要了解servlet规范就可以从request中取值，你可以操作session等等。不用在意应用服务器底层的实现的差别而影响你的开发。

HTTP 协议只是一个规范，定义服务请求和响应的大致式样。Java servlet 类**将HTTP中那些低层的结构包装在 Java 类中**，这些类所包含的便利方法使其在 Java 语言环境中更易于处理。

> 正如您正使用的特定 servlet 容器的配置文件中所定义的，当用户通过 URL 发出一个请求时，这些 Java servlet 类就将之转换成一个 HttpServletRequest，并发送给 URL 所指向的目标。当服务器端完成其工作时，Java 运行时环境（Java Runtime Environment）就将结果包装在一个 HttpServletResponse 中，然后将原 HTTP 响应送回给发出该请求的客户机。在与 Web 应用程序进行交互时，通常会发出多个请求并获得多个响应。所有这些都是在一个会话语境中，Java 语言将之包装在一个 HttpSession 对象中。在处理响应时，您可以访问该对象，并在创建响应时向其添加事件。它提供了一些跨请求的语境。

**容器**（如 Tomcat）将为 servlet 管理运行时环境。您可以配置该容器，定制 J2EE 服务器的工作方式，以便将 servlet 暴露给外部世界。正如我们将看到的，通过该容器中的各种配置文件，您在 URL（由用户在浏览器中输入）与服务器端组件之间搭建了一座桥梁，这些组件将处理您需要该 URL 转换的请求。在运行应用程序时，该容器将**加载并初始化 servlet**，**管理其生命周期**。

## Servlet体系结构

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152028.png)
Servlet顶级类关联图



**Servlet**

Servlet的框架是由两个Java包组成的：javax.servlet与javax.servlet.http。在javax.servlet包中定义了所有的Servlet类都必须实现或者扩展的通用接口和类。在javax.servlet.http包中定义了采用Http协议通信的HttpServlet类。Servlet的框架的核心是javax.servlet.Servlet接口，所有的Servlet都必须实现这个接口。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152057.png)


Servlet接口



在Servlet接口中定义了5个方法：

```
1\. init(ServletConfig)方法：负责初始化Servlet对象，在Servlet的生命周期中，该方法执行一次；该方法执行在单线程的环境下，因此开发者不用考虑线程安全的问题；
2\. service(ServletRequest req,ServletResponse res)方法：负责响应客户的请求；为了提高效率，Servlet规范要求一个Servlet实例必须能够同时服务于多个客户端请求，即service()方法运行在多线程的环境下，Servlet开发者必须保证该方法的线程安全性；
3\. destroy()方法：当Servlet对象退出生命周期时，负责释放占用的资源；
4\. getServletInfo：就是字面意思，返回Servlet的描述；
5\. getServletConfig：这个方法返回由Servlet容器传给init方法的ServletConfig。

```

**ServletRequest & ServletResponse**

对于每一个HTTP请求，servlet容器会创建一个封装了HTTP请求的ServletRequest实例传递给servlet的service方法，ServletResponse则表示一个Servlet响应，其隐藏了将响应发给浏览器的复杂性。通过ServletRequest的方法你可以获取一些请求相关的参数，而ServletResponse则可以将设置一些返回参数信息，并且设置返回内容。

**ServletConfig**

ServletConfig封装可以通过@WebServlet或者web.xml传给一个Servlet的配置信息，以这种方式传递的每一条信息都称做初始化信息，初始化信息就是一个个K-V键值对。为了从一个Servlet内部获取某个初始参数的值，init方法中调用ServletConfig的getinitParameter方法或getinitParameterNames方法获取，除此之外，还可以通过getServletContext获取ServletContext对象。

**ServletContext**

ServletContext是代表了Servlet应用程序。每个Web应用程序只有一个context。在分布式环境中，一个应用程序同时部署到多个容器中，并且每台Java虚拟机都有一个ServletContext对象。有了ServletContext对象后，就可以共享能通过应用程序的所有资源访问的信息，促进Web对象的动态注册，共享的信息通过一个内部Map中的对象保存在ServiceContext中来实现。保存在ServletContext中的对象称作属性。操作属性的方法：

**GenericServlet**

前面编写的Servlet应用中通过实现Servlet接口来编写Servlet，但是我们每次都必须为Servlet中的所有方法都提供实现，还需要将ServletConfig对象保存到一个类级别的变量中，GenericServlet抽象类就是为了为我们省略一些模板代码，实现了Servlet和ServletConfig，完成了一下几个工作：

将init方法中的ServletConfig赋给一个类级变量，使的可以通过getServletConfig来获取。

```
public void init(ServletConfig config) throws ServletException {
        this.config = config;
        this.init();
}

```

同时为避免覆盖init方法后在子类中必须调用super.init(servletConfig)，GenericServlet还提供了一个不带参数的init方法，当ServletConfig赋值完成就会被第带参数的init方法调用。这样就可以通过覆盖不带参数的init方法编写初始化代码，而ServletConfig实例依然得以保存

为Servlet接口中的所有方法提供默认实现。

提供方法来包装ServletConfig中的方法。

**HTTPServlet**

在编写Servlet应用程序时，大多数都要用到HTTP，也就是说可以利用HTTP提供的特性，javax.servlet.http包含了编写Servlet应用程序的类和接口，其中很多覆盖了javax.servlet中的类型，我们自己在编写应用时大多时候也是继承的HttpServlet。

## Servlet工作原理

当Web服务器接收到一个HTTP请求时，它会先判断请求内容——如果是静态网页数据，Web服务器将会自行处理，然后产生响应信息；如果牵涉到动态数据，Web服务器会将请求转交给Servlet容器。此时Servlet容器会找到对应的处理该请求的Servlet实例来处理，结果会送回Web服务器，再由Web服务器传回用户端。

针对同一个Servlet，Servlet容器会在第一次收到http请求时建立一个Servlet实例，然后启动一个线程。第二次收到http请求时，Servlet容器无须建立相同的Servlet实例，而是启动第二个线程来服务客户端请求。所以多线程方式不但可以提高Web应用程序的执行效率，也可以降低Web服务器的系统负担。





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152119.png)


Web服务器工作流程



接着我们描述一下Tomcat与Servlet是如何工作的，首先看下面的时序图：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152142.png)


Servlet工作原理时序图



> 1.  Web Client 向Servlet容器（Tomcat）发出Http请求；
>     
>     
> 2.  Servlet容器接收Web Client的请求；
>     
>     
> 3.  Servlet容器创建一个HttpRequest对象，将Web Client请求的信息封装到这个对象中；
>     
>     
> 4.  Servlet容器创建一个HttpResponse对象；
>     
>     
> 5.  Servlet容器调用HttpServlet对象的service方法，把HttpRequest对象与HttpResponse对象作为参数传给 HttpServlet对象；
>     
>     
> 6.  HttpServlet调用HttpRequest对象的有关方法，获取Http请求信息；
>     
>     
> 7.  HttpServlet调用HttpResponse对象的有关方法，生成响应数据；
>     
>     
> 8.  Servlet容器把HttpServlet的响应结果传给Web Client；

## Servlet生命周期

**在Servlet接口中定义了5个方法，其中3个方法代表了Servlet的生命周期：**

```
1\. init(ServletConfig)方法：负责初始化Servlet对象，在Servlet的生命周期中，该方法执行一次；该方法执行在单线程的环境下，因此开发者不用考虑线程安全的问题；
2\. service(ServletRequest req,ServletResponse res)方法：负责响应客户的请求；为了提高效率，Servlet规范要求一个Servlet实例必须能够同时服务于多个客户端请求，即service()方法运行在多线程的环境下，Servlet开发者必须保证该方法的线程安全性；
3\. destroy()方法：当Servlet对象退出生命周期时，负责释放占用的资源；

```

编程注意事项说明：

1.  当Server Thread线程执行Servlet实例的init()方法时，所有的Client Service Thread线程都不能执行该实例的service()方法，更没有线程能够执行该实例的destroy()方法，因此Servlet的init()方法是工作在单线程的环境下，开发者不必考虑任何线程安全的问题。
2.  当服务器接收到来自客户端的多个请求时，服务器会在单独的Client Service Thread线程中执行Servlet实例的service()方法服务于每个客户端。此时会有多个线程同时执行同一个Servlet实例的service()方法，因此必须考虑线程安全的问题。
3.  虽然service()方法运行在多线程的环境下，并不一定要同步该方法。而是要看这个方法在执行过程中访问的资源类型及对资源的访问方式。分析如下：

```
1\. 如果service()方法没有访问Servlet的成员变量也没有访问全局的资源比如静态变量、文件、数据库连接等，而是只使用了当前线程自己的资源，比如非指向全局资源的临时变量、request和response对象等。该方法本身就是线程安全的，不必进行任何的同步控制。

2\. 如果service()方法访问了Servlet的成员变量，但是对该变量的操作是只读操作，该方法本身就是线程安全的，不必进行任何的同步控制。

3\. 如果service()方法访问了Servlet的成员变量，并且对该变量的操作既有读又有写，通常需要加上同步控制语句。

4\. 如果service()方法访问了全局的静态变量，如果同一时刻系统中也可能有其它线程访问该静态变量，如果既有读也有写的操作，通常需要加上同步控制语句。

5\. 如果service()方法访问了全局的资源，比如文件、数据库连接等，通常需要加上同步控制语句。

```

在创建一个 Java servlet 时，一般需要子类 HttpServlet。该类中的方法允许您访问请求和响应包装器（wrapper），您可以用这个包装器来处理请求和创建响应。Servlet的生命周期，简单的概括这就分为四步：

```
Servlet类加载--->实例化--->服务--->销毁；

```

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152215.png)


Servlet生命周期

**创建Servlet对象的时机：**

1.  **默认情况下**，在Servlet容器启动后：客户首次向Servlet发出请求，Servlet容器会判断内存中是否存在指定的Servlet对象，如果没有则创建它，然后根据客户的请求创建HttpRequest、HttpResponse对象，从而调用Servlet对象的service方法；
2.  **Servlet容器启动时**：当web.xml文件中如果<servlet>元素中指定了<load-on-startup>子元素时，Servlet容器在启动web服务器时，将按照顺序创建并初始化Servlet对象；
3.  **Servlet的类文件被更新后，重新创建Servlet**。Servlet容器在启动时自动创建Servlet，这是由在web.xml文件中为Servlet设置的<load-on-startup>属性决定的。从中我们也能看到同一个类型的Servlet对象在Servlet容器中以单例的形式存在；

> 注意：在web.xml文件中，某些Servlet只有<serlvet>元素，没有<servlet-mapping>元素，这样我们无法通过url的方式访问这些Servlet，这种Servlet通常会在<servlet>元素中配置一个<load-on-startup>子元素，让容器在启动的时候自动加载这些Servlet并调用init(ServletConfig config)方法来初始化该Servlet。其中方法参数config中包含了Servlet的配置信息，比如初始化参数，该对象由服务器创建。

**销毁Servlet对象的时机：**

**Servlet容器停止或者重新启动**：Servlet容器调用Servlet对象的destroy方法来释放资源。以上所讲的就是Servlet对象的生命周期。那么Servlet容器如何知道创建哪一个Servlet对象？Servlet对象如何配置？实际上这些信息是通过读取web.xml配置文件来实现的。

```
<servlet>
    <!-- Servlet对象的名称 -->
    <servlet-name>action<servlet-name>
    <!-- 创建Servlet对象所要调用的类 -->
    <servlet-class>org.apache.struts.action.ActionServlet</servlet-class>
    <init-param>
        <!-- 参数名称 -->
        <param-name>config</param-name>
        <!-- 参数值 -->
        <param-value>/WEB-INF/struts-config.xml</param-value>
    </init-param>
    <init-param>
        <param-name>detail</param-name>
        <param-value>2</param-value>
    </init-param>
    <init-param>
        <param-name>debug</param-name>
        <param-value>2</param-value>
    </init-param>
    <!-- Servlet容器启动时加载Servlet对象的顺序 -->
    <load-on-startup>2</load-on-startup>
</servlet>
<!-- 要与servlet中的servlet-name配置节内容对应 -->
<servlet-mapping>
    <servlet-name>action</servlet-name>
    <!-- 客户访问的Servlet的相对URL路径 -->
    <url-pattern>*.do</url-pattern>
</servlet-mapping>

```

> 当Servlet容器启动的时候读取<servlet>配置节信息，根据<servlet-class>配置节信息创建Servlet对象，同时根据<init-param>配置节信息创建HttpServletConfig对象，然后执行Servlet对象的init方法，并且根据<load-on-startup>配置节信息来决定创建Servlet对象的顺序，如果此配置节信息为负数或者没有配置，那么在Servlet容器启动时，将不加载此Servlet对象。当客户访问Servlet容器时，Servlet容器根据客户访问的URL地址，通过<servlet-mapping>配置节中的<url-pattern>配置节信息找到指定的Servlet对象，并调用此Servlet对象的service方法。

在整个Servlet的生命周期过程中，**创建Servlet实例、调用实例的init()和destroy()方法都只进行一次**，当初始化完成后，Servlet容器会将该实例保存在内存中，通过调用它的service()方法，为接收到的请求服务。下面给出Servlet整个生命周期过程的UML序列图，如图所示：


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152252.png)


Servlet生命周期


> 如果需要让Servlet容器在启动时即加载Servlet，可以在web.xml文件中配置<load-on-startup>元素。

## Servlet中的Listener

Listener 使用的非常广泛，它是基于观察者模式设计的，Listener 的设计对开发 Servlet 应用程序提供了一种快捷的手段，能够方便的从另一个纵向维度控制程序和数据。目前 Servlet 中提供了 5 种两类事件的观察者接口，它们分别是：4 个 EventListeners 类型的，ServletContextAttributeListener、ServletRequestAttributeListener、ServletRequestListener、HttpSessionAttributeListener 和 2 个 LifecycleListeners 类型的，ServletContextListener、HttpSessionListener。如下图所示：


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152314.png)

Servlet中的Listener


它们基本上涵盖了整个 Servlet 生命周期中，你感兴趣的每种事件。这些 Listener 的实现类可以配置在 web.xml 中的 <listener> 标签中。当然也可以在应用程序中动态添加 Listener，需要注意的是 ServletContextListener 在容器启动之后就不能再添加新的，因为它所监听的事件已经不会再出现。掌握这些 Listener 的使用，能够让我们的程序设计的更加灵活。

# Cookie与Session

Servlet 能够给我们提供两部分数据，一个是在 Servlet 初始化时调用 init 方法时设置的 ServletConfig，这个类基本上含有了 Servlet 本身和 Servlet 所运行的 Servlet 容器中的基本信息。还有一部分数据是由 ServletRequest 类提供，从提供的方法中发现主要是描述这次请求的 HTTP 协议的信息。关于这一块还有一个让很多人迷惑的 Session 与 Cookie。

Session 与 Cookie 的作用都是为了保持访问用户与后端服务器的交互状态。它们有各自的优点也有各自的缺陷。然而具有讽刺意味的是它们优点和它们的使用场景又是矛盾的，例如使用 Cookie 来传递信息时，随着 Cookie 个数的增多和访问量的增加，它占用的网络带宽也也会越来越大。所以大访问量的时候希望用 Session，但是 Session 的致命弱点是不容易在多台服务器之间共享，所以这也限制了 Session 的使用。

不管 Session 和 Cookie 有什么不足，我们还是要用它们。下面详细讲一下，Session 如何基于 Cookie 来工作。实际上有三种方式能可以让 Session 正常工作：

*   基于 URL Path Parameter，默认就支持
*   基于 Cookie，如果你没有修改 Context 容器个 cookies 标识的话，默认也是支持的
*   基于 SSL，默认不支持，只有 connector.getAttribute("SSLEnabled") 为 TRUE 时才支持

第一种情况下，当浏览器不支持 Cookie 功能时，浏览器会将用户的 SessionCookieName 重写到用户请求的 URL 参数中，它的传递格式如：

```
 /path/Servlet?name=value&name2=value2&JSESSIONID=value3

```

接着 Request 根据这个 JSESSIONID 参数拿到 Session ID 并设置到 request.setRequestedSessionId 中。

> 请注意如果客户端也支持 Cookie 的话，Tomcat 仍然会解析 Cookie 中的 Session ID，并会覆盖 URL 中的 Session ID。

如果是第三种情况的话将会根据 javax.servlet.request.ssl_session 属性值设置 Session ID。

有了 Session ID 服务器端就可以创建 HttpSession 对象了，第一次触发是通过 request. getSession() 方法，如果当前的 Session ID 还没有对应的 HttpSession 对象那么就创建一个新的，并将这个对象加到 org.apache.catalina. Manager 的 sessions 容器中保存，Manager 类将管理所有 Session 的生命周期，Session 过期将被回收，服务器关闭，Session 将被序列化到磁盘等。只要这个 HttpSession 对象存在，用户就可以根据 Session ID 来获取到这个对象，也就达到了状态的保持。





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152357.png)


Session相关类图



上从图中可以看出从 request.getSession 中获取的 HttpSession 对象实际上是 StandardSession 对象的门面对象，这与前面的 Request 和 Servlet 是一样的原理。下图是 Session 工作的时序图：





![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405152416.png)


Session工作的时序图



还有一点与 Session 关联的 Cookie 与其它 Cookie 没有什么不同，这个配置的配置可以通过 web.xml 中的 session-config 配置项来指定。


## 参考文章

<https://segmentfault.com/a/1190000009707894>

<https://www.cnblogs.com/hysum/p/7100874.html>

<http://c.biancheng.net/view/939.html>

<https://www.runoob.com/>

https://blog.csdn.net/android_hl/article/details/53228348



                     
