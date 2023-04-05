 
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

Tomcat 是 Java WEB 开发接触最多的 Servlet 容器，但它不仅仅是一个 Servlet 容器，它还是一个 WEB 应用服务器，在微服务架构体系下，为了降低部署成本，减少资源的开销，追求的是轻量化与稳定，而 Tomcat 是一个轻量级应用服务器，自然被很多开发人员所接受。

Tomcat 里面藏着很多值得我们每个 Java WEB 开发者学习的知识，可以这么说，当你弄懂了 Tomcat 的设计原理，Java WEB 开发对你来说已经没有什么秘密可言了。本篇文章主要是跟大家聊聊 Tomcat 的内部架构体系，让大家对 Tomcat 有个整体的认知。

前面我也说了，Tomcat 的本质其实就是一个 WEB 服务器 + 一个 Servlet 容器，那么它必然需要处理网络的连接与 Servlet 的管理，因此，Tomcat 设计了两个核心组件来实现这两个功能，分别是连接器和容器，连接器用来处理外部网络连接，容器用来处理内部 Servlet，我用一张图来表示它们的关系：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405153506.png)

一个 Tomcat 代表一个 Server 服务器，一个 Server 服务器可以包含多个 Service 服务，Tomcat 默认的 Service 服务是 Catalina，而一个 Service 服务可以包含多个连接器，因为 Tomcat 支持多种网络协议，包括 HTTP/1.1、HTTP/2、AJP 等等，一个 Service 服务还会包括一个容器，容器外部会有一层 Engine 引擎所包裹，负责与处理连接器的请求与响应，连接器与容器之间通过 ServletRequest 和 ServletResponse 对象进行交流。

也可以从 server.xml 的配置结构可以看出 tomcat 整体的内部结构：

<section>

```
<Server port="8005" shutdown="SHUTDOWN">
```

</section>

## 连接器（Connector）

连接器负责将各种网络协议封装起来，对外部屏蔽了网络连接与 IO 处理的细节，将处理得到的 Request 对象传递给容器处理，Tomcat 将处理请求的细节封装到 ProtocolHandler，ProtocolHandler 是一个接口类型，通过实现 ProtocolHandler 来实现各种协议的处理，如 Http11AprProtocol：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405153520.png)

ProtocolHandler 采用组件模式的设计，将处理网络连接，字节流封装成 Request 对象，再将 Request 适配成 Servlet 处理 ServletRequest 对象这几个动作，用组件封装起来了，ProtocolHandler 包括了三个组件：Endpoint、Processor、Adapter。

Endpoint 在 ProtocolHandler 实现类的构造方法中创建，如下：

<section>

```
public Http11AprProtocol() {
```

</section>

**Endpoint 组件用来处理底层的 Socket 网络连接**，AprEndpoint 里面有个叫 SocketProcessor 的内部类，它负责为 AprEndpoint 将接收到的 Socket 请求转化成 Request 对象，SocketProcessor 实现了 Runnable 接口，它会有一个专门的线程池来处理，后面我会单独从源码的角度分析 Endpoint 组件的设计原理。

org.apache.tomcat.util.net.AprEndpoint.SocketProcessor#doRun：

<section>

```
// Process the request from this socket
```

</section>

process 方法会创建一个 processor 对象，**调用它的 process 方法将 Socket 字节流封装成 Request 对象**，在创建 Processor 组件时，会将 Adapter 组件添加到 Processor 组件中：

org.apache.coyote.http11.AbstractHttp11Protocol#createProcessor：

<section>

```
protected Processor createProcessor() {
```

</section>

而 Adapter 组件在连接器初始化时就已经创建好了：

org.apache.catalina.connector.Connector#initInternal：

<section>

```
// Initialize adapter
```

</section>

目前为止，Tomcat 只有一个 Adapter 实现类，就是 CoyoteAdapter。**Adapter 的主要作用是将 Request 对象适配成容器能够识别的 Request 对象**，比如 Servlet 容器，它的只能识别 ServletRequest 对象，这时候就需要 Adapter 适配器类作一层适配。

以上连接器的各个组件，我用一张图说明它们直接的关系：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405153534.png)

## 容器（Container）

在 Tomcat 中一共设计了 4 种容器，它们分别为 Engine、Host、Context、Wrapper，它们的关系如下图所示：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405153549.png)

•**Engine：表示一个虚拟主机的引擎，一个 Tomcat Server 只有一个 引擎，连接器所有的请求都交给引擎处理，而引擎则会交给相应的虚拟主机去处理请求；**•**Host：表示虚拟主机，一个容器可以有多个虚拟主机，每个主机都有对应的域名，在 Tomcat 中，一个 webapps 就代表一个虚拟主机，当然 webapps 可以配置多个；**•**Context：表示一个应用容器，一个虚拟主机可以拥有多个应用，webapps 中每个目录都代表一个 Context，每个应用可以配置多个 Servlet。**

从上图可看出，**各个容器组件之间的关系是由大到小，即父子关系，它们之间关系形成一个树状的结构**，它们的实现类都实现了 Container 接口，它有如下方法来控制容器组件之间的关系：

<section>

```
ublic interface Container extends Lifecycle {
```

</section>

容器组件之间通过以上几个方法，即可实现它们之间的父子关系，有没有发现，Container 接口还继承了 Lifecycle 接口，它有如下方法：

<section>

```
public interface Lifecycle {   
```

</section>

Tomcat 中有很多组件，组件通过实现 Lifecycle 接口，Tomcat 通过事件机制来实现对这些组件生命周期的管理。

**Tomcat 的这种容器设计思想，其实是运用了组合设计模式的思想，组合设计模式最大的优点是可以自由添加节点，这样也就使得 Tomcat 的容器组件非常地容易进行扩展，符合设计模式中的开闭原则。**

现在我们知道了 Tomcat 的容器组件的组合方式，那我们现在就来想一个问题：

当一个请求过来时，Tomcat 是如何识别请求并将它交给特定 Servlet 来处理呢？

从容器的组合关系可以看出，它们调用顺序必定是：

<section>

```
Engine -> Host -> Context -> Wrapper -> Servlet
```

</section>

那么 Tomcat 是如何来定位 Servlet 的呢？答案是利用 Mapper 组件来完成定位的工作。

**Mapper 最主要的核心功能是保存容器组件之间访问路径的映射关系**，它是如何做到这点的呢？

我们不妨先从源码入手：

org.apache.catalina.core.StandardService：

<section>

```
protected final Mapper mapper = new Mapper();
```

</section>

Service 实现类中，已经初始化了 Mapper 组件以及它的监听类 MapperListener，这里先说明一下，在 Tomcat 组件中，标准的实现组件类前缀会有 Standard，比如：

<section>

```
org.apache.catalina.core.StandardServer
```

</section>

在 Service 服务启动的时候，会调用 MapperListener.start() 方法，最终会执行 MapperListener 的 startInternal 方法：

org.apache.catalina.mapper.MapperListener#startInternal：

<section>

```
Container[] conHosts = engine.findChildren();
```

</section>

该方法会注册新的虚拟主机，接着 registerHost() 方法会注册 context，以此类推，从而将容器组件直接的访问的路径都注册到 Mapper 中。

定位 Servlet 的流程图：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405153609.png)