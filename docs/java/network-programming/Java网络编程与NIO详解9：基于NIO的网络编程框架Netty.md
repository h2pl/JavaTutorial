# Table of Contents

  * [Netty概述](#netty概述)
  * [etty简介](#etty简介)
    * [Netty都有哪些组件？](#netty都有哪些组件？)
    * [Netty是如何处理连接请求和业务逻辑的呢？](#netty是如何处理连接请求和业务逻辑的呢？)
    * [如何配置一个Netty应用？](#如何配置一个netty应用？)
    * [Netty是如何处理数据的？](#netty是如何处理数据的？)
    * [如何处理我们的业务逻辑？](#如何处理我们的业务逻辑？)
    * [ByteBuf](#bytebuf)
    * [Channel](#channel)
    * [ChannelHandler](#channelhandler)
    * [ChannelPipeline](#channelpipeline)
    * [EventLoop](#eventloop)
    * [Bootstrap](#bootstrap)
    * [Echo示例](#echo示例)
  * [参考文献](#参考文献)


本文转自：https://sylvanassun.github.io/2017/11/30/2017-11-30-netty_introduction/

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章将同步到我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《不可轻视的Java网络编程》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从计算机网络的基础知识入手，一步步地学习Java网络基础，从socket到nio、bio、aio和netty等网络编程知识，并且进行实战，网络编程是每一个Java后端工程师必须要学习和理解的知识点，进一步来说，你还需要掌握Linux中的网络编程原理，包括IO模型、网络编程框架netty的进阶原理，才能更完整地了解整个Java网络编程的知识体系，形成自己的知识框架。

为了更好地总结和检验你的学习成果，本系列文章也会提供部分知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## Netty概述

**Netty是一个基于异步与事件驱动的网络应用程序框架，它支持快速与简单地开发可维护的高性能的服务器与客户端。**

所谓**事件驱动就是由通过各种事件响应来决定程序的流程**，在Netty中到处都充满了异步与事件驱动，这种特点使得应用程序可以以任意的顺序响应在任意的时间点产生的事件，它带来了非常高的可伸缩性，让你的应用可以在需要处理的工作不断增长时，通过某种可行的方式或者扩大它的处理能力来适应这种增长。

Netty提供了高性能与易用性，它具有以下特点：

*   **拥有设计良好且统一的API**，支持NIO与OIO（阻塞IO）等多种传输类型，支持真正的无连接UDP Socket。

*   **简单而强大的线程模型**，可高度定制线程（池）。(定制化的Reactor模型)

*   **良好的模块化与解耦**，支持可扩展和灵活的事件模型，可以很轻松地分离关注点以复用逻辑组件（可插拔的）。

*   **性能高效**，拥有比Java核心API更高的吞吐量，通过zero-copy功能以实现最少的内存复制消耗。

*   **内置了许多常用的协议编解码器**，如HTTP、SSL、WebScoket等常见协议可以通过Netty做到开箱即用。用户也可以利用Netty简单方便地实现自己的应用层协议。

大多数人使用Netty主要还是为了**提高应用的性能**，而高性能则离不开非阻塞IO。Netty的非阻塞IO是基于Java NIO的，并且对其进行了封装（直接使用Java NIO API在高复杂度下的应用中是一项非常繁琐且容易出错的操作，而Netty帮你封装了这些复杂操作）。

## etty简介

<pre>读完这一章，我们基本上可以了解到Netty所有重要的组件，对Netty有一个全面的认识，这对下一步深入学习Netty是十分重要的，而学完这一章，我们其实已经可以用Netty解决一些常规的问题了。</pre>

### Netty都有哪些组件？

<pre>为了更好的理解和进一步深入Netty，我们先总体认识一下Netty用到的组件及它们在整个Netty架构中是怎么协调工作的。Netty应用中必不可少的组件：</pre>

*   Bootstrap or ServerBootstrap

*   EventLoop

*   EventLoopGroup

*   ChannelPipeline

*   Channel

*   Future or ChannelFuture

*   ChannelInitializer

*   ChannelHandler

*   Bootstrap，一个Netty应用通常由一个Bootstrap开始，它主要作用是配置整个Netty程序，**串联起各个组件**。

    Handler，为了支持各种协议和处理数据的方式，便诞生了Handler组件。Handler主要用来**处理各种事件**，这里的事件很广泛，比如可以是**连接、数据接收、异常、数据转换**等。

    **ChannelInboundHandler**，一个最常用的Handler。这个Handler的作用就是**处理接收到数据**时的事件，也就是说，我们的业务逻辑一般就是写在这个Handler里面的，ChannelInboundHandler就是用来处理我们的**核心业务逻辑**。

    ChannelInitializer，当一个链接建立时，我们需要知道**怎么来接收或者发送数据**，当然，我们有各种各样的Handler实现来处理它，那么ChannelInitializer便是用来**配置这些Handler，它会提供一个ChannelPipeline，并把Handler加入到ChannelPipeline。**

    ChannelPipeline，一个Netty应用基于ChannelPipeline机制，这种机制需要**依赖于EventLoop和EventLoopGroup，因为它们三个都和事件或者事件处理相关。**

    EventLoops的目的是为Channel处理IO操作，一个EventLoop可以为多个Channel服务。

    EventLoopGroup会包含多个EventLoop。

    Channel代表了一个Socket链接，或者其它和IO操作相关的组件，它和EventLoop一起用来参与IO处理。

    Future，在Netty中所有的IO操作都是**异步的**，因此，你**不能立刻得知消息是否被正确处理**，但是我们**可以过一会等它执行完成或者直接注册一个监听**，具体的实现就是通过Future和ChannelFutures,他们可以注册一个监听，当操作执行成功或失败时监听会自动触发。总之**，所有的操作都会返回一个ChannelFuture**。

### Netty是如何处理连接请求和业务逻辑的呢？

Channels、Events 和 IO

Netty是一个**非阻塞的、事件驱动的、网络编程**框架。当然，我们很容易理解Netty会用线程来处理IO事件，对于熟悉多线程编程的人来说，你或许会想到如何同步你的代码，但是Netty不需要我们考虑这些，具体是这样：

一个Channel会对应一个EventLoop，而一个**EventLoop会对应着一个线程**，也就是说，仅有一个线程在负责一个Channel的IO操作。

关于这些名词之间的关系，可以见下图：

![](https://img-blog.csdn.net/20140606104845234?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

如图所示：当一个连接到达，Netty会注册一个channel，然后EventLoopGroup会**分配一个EventLoop绑定到这个channel**,在这个channel的整个生命周期过程中，都会由绑定的这个EventLoop来为它服务，而这个**EventLoop就是一个线程**。

说到这里，那么EventLoops和EventLoopGroups关系是如何的呢？我们前面说过一个EventLoopGroup包含多个Eventloop，但是我们看一下下面这幅图，这幅图是一个继承树，从这幅图中我们可以看出，EventLoop其实继承自EventloopGroup，也就是说，在某些情况下，我们可以把一个EventLoopGroup当做一个EventLoop来用。

<pre>​</pre>

[![](https://img-blog.csdn.net/20140606104919140?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)](https://img-blog.csdn.net/20140606104919140?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### 如何配置一个Netty应用？

BootsStrapping

我们**利用BootsStrap来配置netty 应用**，它有两种类型，一种用于Client端：BootsStrap，另一种用于Server端：ServerBootstrap，要想区别如何使用它们，你仅需要记住一个用在Client端，一个用在Server端。下面我们来详细介绍一下这两种类型的区别：

1.第一个最明显的区别是，ServerBootstrap用于Server端，通过调用**bind()**方法来绑定到一个端口监听连接；Bootstrap用于Client端，需要调用**connect**()方法来连接服务器端，但我们也可以通过调用**bind()方法返回的ChannelFuture**中获取Channel从而去connect服务器端。

2.**客户端**的Bootstrap一般用**一个EventLoopGroup**，而**服务器**端的ServerBootstrap会用到**两个**（这两个也可以是同一个实例）。为何服务器端要用到两个EventLoopGroup呢？这么设计有明显的好处，如果一个ServerBootstrap有两个EventLoopGroup，那么就可以把**第一个**EventLoopGroup用来**专门负责绑定到端口监听连接事件**，而把**第二个**EventLoopGroup用来**处理每个接收到的连接**，下面我们用一幅图来展现一下这种模式：

![](https://img-blog.csdn.net/20140606104949484?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

PS: 如果仅由一个EventLoopGroup处理所有请求和连接的话，在并发量很大的情况下，这个EventLoopGroup有可能会忙于处理已经接收到的连接而不能及时处理新的连接请求，**用两个的话**，会有专门的线程来处理连接请求，不会导致请求超时的情况，大大**提高了并发处理能力**。

我们知道一个Channel需要由一个EventLoop来绑定，而且两者**一旦绑定就不会再改变**。一般情况下一个EventLoopGroup中的**EventLoop数量会少于Channel数量**，那么就很**有可能出现一个多个Channel公用一个EventLoop**的情况，这就意味着如果一个Channel中的**EventLoop很忙**的话，会**影响**到这个Eventloop**对其它Channel的处理**，**这也就是为什么我们不能阻塞EventLoop的原因。**

当然，我们的Server也可以只用一个EventLoopGroup,由一个实例来处理连接请求和IO事件，请看下面这幅图：

![](https://img-blog.csdn.net/20140606105016890?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

### Netty是如何处理数据的？

Netty核心ChannelHandler

下面我们来看一下netty中是怎样处理数据的，回想一下我们前面讲到的Handler，对了，就是它。说到Handler我们就不得不提ChannelPipeline，ChannelPipeline负责**安排Handler的顺序及其执行**，下面我们就来详细介绍一下他们：

ChannelPipeline and handlers

我们的应用程序中用到的最多的应该就是ChannelHandler，我们可以这么想象，数据在一个ChannelPipeline中流动，而ChannelHandler便是其中的一个个的小阀门，这些数据都会经过每一个ChannelHandler并且被它处理。这里有一个公共接口ChannelHandler:

![](https://img-blog.csdn.net/20140606105045406?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从上图中我们可以看到，ChannelHandler有两个子类ChannelInboundHandler和ChannelOutboundHandler，这两个类对应了两个数据流向，如果数据是**从外部流入**我们的应用程序，我们就看做是**inbound**，相反便是outbound。其实ChannelHandler和Servlet有些类似，一个ChannelHandler处理完接收到的数据会传给下一个Handler，或者什么不处理，直接传递给下一个。下面我们看一下ChannelPipeline是如何安排ChannelHandler的：

![](https://img-blog.csdn.net/20140606105113171?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvc3VpZmVuZzMwNTE=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

从上图中我们可以看到，**一个ChannelPipeline可以把两种Handler（ChannelInboundHandler和ChannelOutboundHandler）混合在一起**，当一个数据流进入ChannelPipeline时，它会从ChannelPipeline头部开始传给第一个ChannelInboundHandler，当第一个处理完后再传给下一个，一直传递到管道的尾部。与之相对应的是，当数据被写出时，它会从管道的尾部开始，先经过管道尾部的“最后”一个ChannelOutboundHandler，当它处理完成后会传递给前一个ChannelOutboundHandler。

**数据在各个Handler之间传递**，这需要调用方法中传递的**ChanneHandlerContext**来操作， 在netty的API中提供了两个基类分**ChannelOutboundHandlerAdapter**和**ChannelInboundHandlerAdapter**，他们**仅仅实现了调用ChanneHandlerContext来把消息传递给下一个Handler**，因为我们只关心处理数据，因此我们的程序中**可以继承这两个基类来帮助我们做这些**，而**我们仅需实现处理数据**的部分即可。

我们知道InboundHandler和OutboundHandler在ChannelPipeline中是**混合在一起**的，那么它们**如何区分**彼此呢？其实很容易，因为**它们各自实现的是不同的接口**，对于inbound event，Netty会自动跳过OutboundHandler,相反若是outbound event，ChannelInboundHandler会被忽略掉。

当一个ChannelHandler被加入到ChannelPipeline中时，它便会获得一个ChannelHandlerContext的引用，而**ChannelHandlerContext可以用来读写Netty中的数据流**。因此，现在可以有两种方式来发送数据，一种是把数据直接写入Channel，一种是把数据写入ChannelHandlerContext，它们的区别是写入Channel的话，数据流会从Channel的头开始传递，而如果写入ChannelHandlerContext的话，数据流会流入管道中的下一个Handler。

### 如何处理我们的业务逻辑？

Encoders, Decoders and Domain Logic

Netty中会有很多Handler，具体是哪种Handler还要看它们继承的是InboundAdapter还是OutboundAdapter。当然，Netty中还提供了一些列的**Adapter来帮助我们简化开发**，我们知道在Channelpipeline中**每一个Handler都负责把Event传递给下一个Handler**，如果有了这些辅助Adapter，这些**额外的工作都可自动完成**，我们只需覆盖实现我们真正关心的部分即可。此外，还有一些Adapter会提供一些额外的功能，比如编码和解码。那么下面我们就来看一下其中的三种常用的ChannelHandler：

Encoders和Decoders

因为我们在网络传输时只能传输字节流，因此，在发送数据之前，我们必须把我们的message型转换为bytes，与之对应，我们在接收数据后，必须把接收到的bytes再转换成message。我们把bytes to message这个过程称作Decode(解码成我们可以理解的)，把message to bytes这个过程成为Encode。

Netty中提供了很多现成的编码/解码器，我们一般从他们的名字中便可知道他们的用途，如**ByteToMessageDecoder**、**MessageToByteEncoder**，如专门用来处理Google Protobuf协议的**ProtobufEncoder**、 **ProtobufDecoder**。

我们前面说过，具体是哪种Handler就要看它们继承的是InboundAdapter还是OutboundAdapter，对于**Decoders**,很容易便可以知道它是继承自**ChannelInboundHandlerAdapter**或 ChannelInboundHandler，因为解码的意思是把ChannelPipeline**传入的bytes解码成我们可以理解的message**（即Java Object），而ChannelInboundHandler正是处理Inbound Event，而Inbound Event中传入的正是字节流。Decoder会覆盖其中的“ChannelRead()”方法，在这个方法中来调用具体的decode方法解码传递过来的字节流，然后通过调用ChannelHandlerContext.fireChannelRead(decodedMessage)方法把编码好的Message传递给下一个Handler。与之类似，Encoder就不必多少了。

Domain Logic

其实我们最最关心的事情就是**如何处理接收到的解码后的数据**，我们真正的业务逻辑便是处理接收到的数据。Netty提供了一个最常用的基类**SimpleChannelInboundHandler**<T>，其中**T就是这个Handler处理的数据的类型**（上一个Handler已经替我们解码好了），消息到达这个Handler时，Netty会自动调用这个Handler中的**channelRead0**(ChannelHandlerContext,T)方法，T是传递过来的数据对象，在这个方法中我们便可以任意写我们的业务逻辑了。

Netty从某方面来说就是一套NIO框架，在Java NIO基础上做了封装，所以要想学好Netty我建议先理解好Java NIO，

NIO可以称为New IO也可以称为Non-blocking IO，它比Java旧的阻塞IO在性能上要高效许多（如果让每一个连接中的IO操作都单独创建一个线程，那么阻塞IO并不会比NIO在性能上落后，但不可能创建无限多的线程，在连接数非常多的情况下会很糟糕）。

*   ByteBuffer：NIO的数据传输是基于缓冲区的，ByteBuffer正是NIO数据传输中所使用的缓冲区抽象。**ByteBuffer支持在堆外分配内存**，并且尝试避免在执行I/O操作中的多余复制。一般的I/O操作都需要进行系统调用，这样会先切换到内核态，内核态要先从文件读取数据到它的缓冲区，只有等数据准备完毕后，才会从内核态把数据写到用户态，所谓的**阻塞IO**其实就是说的**在等待数据准备好的这段时间内进行阻塞**。如果想要避免这个额外的内核操作，可以通过使用mmap（虚拟内存映射）的方式来让用户态直接操作文件。

*   Channel：它类似于(fd)文件描述符，简单地来说它**代表了一个实体**（如一个硬件设备、文件、Socket或者一个能够执行一个或多个不同的I/O操作的程序组件）。你可以从一个Channel中读取数据到缓冲区，也可以将一个缓冲区中的数据写入到Channel。

*   Selector：选择器是NIO实现的关键，NIO采用的是I/O多路复用的方式来实现非阻塞，Selector通过在**一个线程中监听每个Channel的IO事件来确定有哪些已经准备好进行IO操作的Channel**，因此可**以在任何时间检查任意的读操作或写操作的完成状态**。这种方式**避免了等待IO操作准备数据时的阻塞**，使用较少的线程便可以处理许多连接，减少了线程切换与维护的开销。

    [![](http://wx2.sinaimg.cn/large/63503acbly1flys7n7hvaj20h90doglj.jpg)](http://wx2.sinaimg.cn/large/63503acbly1flys7n7hvaj20h90doglj.jpg)

了解了NIO的实现思想之后，我觉得还很有必要了解一下Unix中的I/O模型，Unix中拥有以下5种I/O模型：

*   阻塞I/O（Blocking I/O）

*   非阻塞I/O（Non-blocking I/O）

*   I/O多路复用（I/O multiplexing (select and poll)）

*   信号驱动I/O（signal driven I/O (SIGIO)）

*   异步I/O（asynchronous I/O (the POSIX aio_functions)）

[![](http://wx3.sinaimg.cn/large/63503acbly1flz1e7kzblj20wb0ftq3l.jpg)](http://wx3.sinaimg.cn/large/63503acbly1flz1e7kzblj20wb0ftq3l.jpg)

![](file:///C:/Users/xiaok/Pictures/%E5%A4%8D%E4%B9%A0%E6%96%87%E6%A1%A3%E5%9B%BE%E7%89%87_%E5%8B%BF%E5%88%A0/%E6%AF%94%E7%89%B9%E6%88%AA%E5%9B%BE2019-01-12-16-40-18.png?lastModify=1549709160)

阻塞I/O模型是最常见的I/O模型，通常我们使用的InputStream/OutputStream都是基于阻塞I/O模型。在上图中，我们使用UDP作为例子，recvfrom()函数是UDP协议用于接收数据的函数，它需要**使用系统调用并一直阻塞到内核将数据准备好**，之后再由内核缓冲区复制数据到用户态（即是recvfrom()接收到数据），所谓**阻塞就是在等待内核准备数据的这段时间内什么也不干**。

举个生活中的例子，阻塞I/O就像是你去餐厅吃饭，在等待饭做好的时间段中，你只能在餐厅中坐着干等（如果你在玩手机那么这就是非阻塞I/O了）。

[![](http://wx2.sinaimg.cn/large/63503acbly1flz1e8lh7rj20wb0ft0ty.jpg)](http://wx2.sinaimg.cn/large/63503acbly1flz1e8lh7rj20wb0ft0ty.jpg)

> 在非阻塞I/O模型中，内核在**数据尚未准备好**的情况下回**返回一个错误码`EWOULDBLOCK`**，而**recvfrom**并没有在失败的情况下选择阻塞休眠，而是**不断地向内核询问是否已经准备完毕**，在上图中，前三次内核都返回了`EWOULDBLOCK`，直到第四次询问时，内核数据准备完毕，然后开始将内核中缓存的数据复制到用户态。这种不断询问内核以查看某种状态是否完成的方式被称为**polling（轮询）**。

非阻塞I/O就像是你在点外卖，只不过你非常心急，**每隔一段时间就要打电话问外卖小哥有没有到**。

[![](http://wx3.sinaimg.cn/large/63503acbly1flz1e989dfj20wh0g80tw.jpg)](http://wx3.sinaimg.cn/large/63503acbly1flz1e989dfj20wh0g80tw.jpg)


I/O多路复用的思想跟非阻塞I/O是一样的，只不过在**非阻塞I/O**中，是在**recvfrom的用户态（或一个线程）中去轮询内核**，这种方式会**消耗大量的CPU时间**。而**I/O多路复用**则是通过select()或poll()**系统调用来负责进行轮询**，以实现监听I/O读写事件的状态。如上图中，select监听到一个datagram可读时，就交由recvfrom去发送系统调用将内核中的数据复制到用户态。

这种方式的优点很明显，通过**I/O多路复用**可以**监听多个文件描述符**，且在**内核中完成监控的任务**。但缺点是至少需要两个系统调用（select()与recvfrom()）。

I/O多路复用同样适用于点外卖这个例子，只不过你在等外卖的期间完全可以做自己的事情，当外卖到的时候会**通过外卖APP或者由外卖小哥打电话来通知你**(因为内核会帮你轮询)。

Unix中提供了两种I/O多路复用函数，select()和poll()。select()的兼容性更好，但它在单个进程中所能监控的文件描述符是有限的，这个值与`FD_SETSIZE`相关，32位系统中默认为1024，64位系统中为2048。select()还有一个缺点就是他轮询的方式，它采取了**线性扫描的轮询方式**，每次都要遍历FD_SETSIZE个文件描述符，不管它们是否活不活跃的。poll()本质上与select()的实现**没有区别**，不过在数据结构上区别很大，用户必须分配一个pollfd结构数组，该数组维护在内核态中，正因如此，**poll()并不像select()那样拥有大小上限的限制**，但缺点同样也很明显，**大量的fd数组会在用户态与内核态之间不断复制**，不管这样的复制是否有意义。

还有一种比select()与poll()更加高效的实现叫做epoll()，它是由Linux内核2.6推出的可伸缩的I/O多路复用实现，目的是为了替代select()与poll()。epoll()同样**没有文件描述符上限的限制**，它**使用一个文件描述符来管理多个文件描述符**，并**使用一个红黑树来作为存储结构**。同时它还支持边缘触发（edge-triggered）与水平触发（level-triggered）两种模式（poll()只支持水平触发），在**边缘触发模式**下，**`epoll_wait`仅会在新的事件对象首次被加入到epoll时返回**，而在**水平触发**模式下，**`epoll_wait`会在事件状态未变更前不断地触发**。也就是说，边缘触发模式**只会**在文件描述符**变为就绪状态时通知一次**，水平触发模式会**不断地通知**该文件描述符**直到被处理**。

关于`epoll_wait`请参考如下epoll API。

<pre>// 创建一个epoll对象并返回它的文件描述符。
// 参数flags允许修改epoll的行为，它只有一个有效值EPOLL_CLOEXEC。
int epoll_create1(int flags);
// 配置对象，该对象负责描述监控哪些文件描述符和哪些事件。
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// 等待与epoll_ctl注册的任何事件，直至事件发生一次或超时。
// 返回在events中发生的事件，最多同时返回maxevents个。
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);</pre>

epoll另一亮点是**采用了事件驱动的方式而不是轮询**，在**epoll_ctl**中注册的文件描述符**在事件触发的时候会通过一个回调机制**来激活该文件描述符，**`epoll_wait`便可以收到通知**。这样效率就不会与文件描述符的数量成正比

在Java NIO2（从JDK1.7开始引入）中，只要Linux内核版本在2.6以上，就会采用epoll，如下源码所示（DefaultSelectorProvider.java）。

<pre>public static SelectorProvider create() {
String osname = AccessController.doPrivileged(
new GetPropertyAction("os.name"));
if ("SunOS".equals(osname)) {
return new sun.nio.ch.DevPollSelectorProvider();
}
// use EPollSelectorProvider for Linux kernels >= 2.6
if ("Linux".equals(osname)) {
String osversion = AccessController.doPrivileged(
new GetPropertyAction("os.version"));
String[] vers = osversion.split("\\.", 0);
if (vers.length >= 2) {
try {
int major = Integer.parseInt(vers[0]);
int minor = Integer.parseInt(vers[1]);
if (major > 2 || (major == 2 && minor >= 6)) {
return new sun.nio.ch.EPollSelectorProvider();
}
} catch (NumberFormatException x) {
// format not recognized
}
}
}
return new sun.nio.ch.PollSelectorProvider();
}</pre>

[![](http://wx3.sinaimg.cn/large/63503acbly1flz1e9uk8aj20wb0ft3zn.jpg)](http://wx3.sinaimg.cn/large/63503acbly1flz1e9uk8aj20wb0ft3zn.jpg)

信号驱动I/O模型使用到了**信号**，内核在数据**准备就绪**时会**通过信号来进行通知**。我们首先**开启**了一个信号驱动I/O套接字，并使用sigaction系统调用来安装信号处理程序，内核直接返回，不会阻塞用户态。当datagram准备好时，内核会发送SIGIN信号，recvfrom接收到信号后会发送系统调用开始进行I/O操作。

这种模型的优点是**主进程（线程）不会被阻塞**，当数据准备就绪时，**通过信号处理程序**来通知主进程（线程）准备进行I/O操作与对数据的处理。

[![](http://wx2.sinaimg.cn/large/63503acbly1flz1eai66rj20wb0g8aau.jpg)](http://wx2.sinaimg.cn/large/63503acbly1flz1eai66rj20wb0g8aau.jpg)

我们之前讨论的各种I/O模型无论是阻塞还是非阻塞，它们所说的**阻塞都是指的数据准备阶段**。**异步I/O**模型**同样依赖**于**信号**处理程序来进行通知，但与以上I/O模型都不相同的是，异步I/O模型通知的是**I/O操作**已经完成，而不是**数据准备**完成。

可以说**异步I/O模型才是真正的非阻塞**，主进程只管做自己的事情，然后在I/O操作完成时调用回调函数来完成一些对数据的处理操作即可。

闲扯了这么多，想必大家已经对I/O模型有了一个深刻的认识。之后，我们将会结合部分源码（Netty4.X）来探讨Netty中的各大核心组件，以及如何使用Netty，你会发现实现一个Netty程序是多么简单（而且还伴随了高性能与可维护性）。


![](https://netty.io/images/components.png)

### ByteBuf



* * *



网络传输的基本单位是字节，在Java NIO中提供了ByteBuffer作为字节缓冲区容器，但该类的API使用起来不太方便，所以Netty实现了ByteBuf作为其替代品，下面是使用ByteBuf的优点：

*   相比ByteBuffer使用起来**更加简单**。

*   通过内置的复合缓冲区类型实现了透明的**zero-copy**。

*   **容量**可以**按需增长**。

*   **读和写**使用了**不同的索引指针**。

*   支持**链式调用**。

*   支持**引用计数与池化**。

*   可以被用户**自定义的缓冲区类型**扩展。

在讨论ByteBuf之前，我们先需要了解一下ByteBuffer的实现，这样才能比较深刻地明白它们之间的区别。

ByteBuffer继承于`abstract class Buffer`（所以还有LongBuffer、IntBuffer等其他类型的实现），本质上它只是一个有限的线性的元素序列，包含了三个重要的属性。

*   Capacity：缓冲区中元素的容量大小，你只能将capacity个数量的元素写入缓冲区，一旦缓冲区已满就需要清理缓冲区才能继续写数据。

*   Position：指向下一个写入数据位置的索引指针，初始位置为0，最大为capacity-1。当写模式转换为读模式时，position需要被重置为0。

*   Limit：在写模式中，limit是可以写入缓冲区的最大索引，也就是说它在写模式中等价于缓冲区的容量。在读模式中，limit表示可以读取数据的最大索引。

[![](http://tutorials.jenkov.com/images/java-nio/buffers-modes.png)](http://tutorials.jenkov.com/images/java-nio/buffers-modes.png)

由于Buffer中只维护了position一个索引指针，所以它在读写模式之间的切换需要调用一个flip()方法来重置指针。使用Buffer的流程一般如下：

*   写入数据到缓冲区。

*   调用flip()方法。

*   从缓冲区中读取数据

*   调用buffer.clear()或者buffer.compact()清理缓冲区，以便下次写入数据。

<pre>RandomAccessFile aFile = new RandomAccessFile("data/nio-data.txt", "rw");
FileChannel inChannel = aFile.getChannel();
// 分配一个48字节大小的缓冲区
ByteBuffer buf = ByteBuffer.allocate(48);
int bytesRead = inChannel.read(buf); // 读取数据到缓冲区
while (bytesRead != -1) {
buf.flip(); // 将position重置为0
while(buf.hasRemaining()){
System.out.print((char) buf.get()); // 读取数据并输出到控制台
}
buf.clear(); // 清理缓冲区
bytesRead = inChannel.read(buf);
}
aFile.close();
Buffer中核心方法的实现也非常简单，主要就是在操作指针position。</pre>

Buffer中核心方法的实现也非常简单，主要就是在操作指针position。

<pre>/**
* Sets this buffer's mark at its position.
*
* @return This buffer
*/
public final Buffer mark() {
mark = position; // mark属性是用来标记当前索引位置的
return this;
}
// 将当前索引位置重置为mark所标记的位置
public final Buffer reset() {
int m = mark;
if (m < 0)
throw new InvalidMarkException();
position = m;
return this;
}
// 翻转这个Buffer，将limit设置为当前索引位置，然后再把position重置为0
public final Buffer flip() {
limit = position;
position = 0;
mark = -1;
return this;
}
// 清理缓冲区
// 说是清理,也只是把postion与limit进行重置,之后再写入数据就会覆盖之前的数据了
public final Buffer clear() {
position = 0;
limit = capacity;
mark = -1;
return this;
}
// 返回剩余空间
public final int remaining() {
return limit - position;
}</pre>

Java NIO中的**Buffer API操作的麻烦之处就在于读写转换需要手动重置指针。而ByteBuf没有这种繁琐性，它维护了两个不同的索引，一个用于读取，一个用于写入**。当你从ByteBuf读取数据时，它的readerIndex将会被递增已经被读取的字节数，同样的，当你写入数据时，writerIndex则会递增。readerIndex的最大范围在writerIndex的所在位置，如果试图移动readerIndex超过该值则会触发异常。

ByteBuf中名称以read或write开头的方法将会递增它们其对应的索引，而名称以get或set开头的方法则不会。ByteBuf同样可以指定一个最大容量，试图移动writerIndex超过该值则会触发异常。

<pre>public byte readByte() {
 this.checkReadableBytes0(1); // 检查readerIndex是否已越界
 int i = this.readerIndex;
 byte b = this._getByte(i);
 this.readerIndex = i + 1; // 递增readerIndex
 return b;
}
private void checkReadableBytes0(int minimumReadableBytes) {
 this.ensureAccessible();
 if(this.readerIndex > this.writerIndex - minimumReadableBytes) {
 throw new IndexOutOfBoundsException(String.format("readerIndex(%d) + length(%d) exceeds writerIndex(%d): %s", new Object[]{Integer.valueOf(this.readerIndex), Integer.valueOf(minimumReadableBytes), Integer.valueOf(this.writerIndex), this}));
 }
}
public ByteBuf writeByte(int value) {
 this.ensureAccessible();
 this.ensureWritable0(1); // 检查writerIndex是否会越过capacity
 this._setByte(this.writerIndex++, value);
 return this;
}
private void ensureWritable0(int minWritableBytes) {
 if(minWritableBytes > this.writableBytes()) {
 if(minWritableBytes > this.maxCapacity - this.writerIndex) {
 throw new IndexOutOfBoundsException(String.format("writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s", new Object[]{Integer.valueOf(this.writerIndex), Integer.valueOf(minWritableBytes), Integer.valueOf(this.maxCapacity), this}));
 } else {
 int newCapacity = this.alloc().calculateNewCapacity(this.writerIndex + minWritableBytes, this.maxCapacity);
 this.capacity(newCapacity);
 }
 }
}
// get与set只对传入的索引进行了检查，然后对其位置进行get或set
public byte getByte(int index) {
 this.checkIndex(index);
 return this._getByte(index);
}
public ByteBuf setByte(int index, int value) {
 this.checkIndex(index);
 this._setByte(index, value);
 return this;
}</pre>

ByteBuf同样支持在**堆内和堆外进行分配**。在**堆内分配**也被称为**支撑数组模式**，它能在**没有使用池化**的情况下**提供快速的分配和释放**。

<pre>ByteBuf heapBuf = Unpooled.copiedBuffer(bytes);
if (heapBuf.hasArray()) { // 判断是否有一个支撑数组
byte[] array = heapBuf.array();
// 计算第一个字节的偏移量
int offset = heapBuf.arrayOffset() + heapBuf.readerIndex();
int length = heapBuf.readableBytes(); // 获得可读字节
handleArray(array,offset,length); // 调用你的处理方法
}
</pre>

另一种模式为**堆外分配**，Java NIO ByteBuffer类在JDK1.4时就已经允许JVM实现通过JNI调用来在堆外分配内存（调用malloc()函数在JVM堆外分配内存），这主要是**为了避免额外的缓冲区复制操作**。

<pre>ByteBuf directBuf = Unpooled.directBuffer(capacity);
if (!directBuf.hasArray()) {
int length = directBuf.readableBytes();
byte[] array = new byte[length];
// 将字节复制到数组中
directBuf.getBytes(directBuf.readerIndex(),array);
handleArray(array,0,length);
}
</pre>

ByteBuf还支持第三种模式，它被称为**复合缓冲区**，为多个ByteBuf**提供**了一个**聚合视图**。在这个视图中，你可以根据需要添加或者删除ByteBuf实例，ByteBuf的子类**CompositeByteBuf实现了该模式**。

一个适合使用**复合缓冲区的场景是HTTP协议**，通过HTTP协议传输的消息都会被分成两部分——头部和主体，如果这两部分由应用程序的不同模块产生，将在消息发送时进行组装，并且该应用程序还会为多个消息复用相同的消息主体，这样对于每个消息都将会创建一个新的头部，产生了很多不必要的内存操作。使用CompositeByteBuf是一个很好的选择，它消除了这些额外的复制，以帮助你复用这些消息。

<pre>CompositeByteBuf messageBuf = Unpooled.compositeBuffer();
ByteBuf headerBuf = ....;
ByteBuf bodyBuf = ....;
messageBuf.addComponents(headerBuf,bodyBuf);
for (ByteBuf buf : messageBuf) {
System.out.println(buf.toString());
}
</pre>

CompositeByteBuf透明的实现了**zero-copy**，zero-copy其实就是避免数据在两个内存区域中来回的复制。从操作系统层面上来讲，zero-copy指的是**避免在内核态与用户态之间的数据缓冲区复制（通过mmap避免）**，而Netty中的zero-copy更偏向于在用户态中的数据操作的优化，就像使用CompositeByteBuf来复用多个ByteBuf以避免额外的复制，也可以使用wrap()方法来将一个字节数组包装成ByteBuf，又或者使用ByteBuf的slice()方法把它分割为多个共享同一内存区域的ByteBuf，这些都是为了优化内存的使用率。

那么如何创建ByteBuf呢？在上面的代码中使用到了**Unpooled**，它是Netty提供的一个用于创建与分配ByteBuf的工具类，建议都使用这个工具类来创建你的缓冲区，不要自己去调用构造函数。经常使用的是wrappedBuffer()与copiedBuffer()，它们一个是用于将一个字节数组或ByteBuffer包装为一个ByteBuf，一个是根据传入的字节数组与ByteBuffer/ByteBuf来复制出一个新的ByteBuf。

<pre>// 通过array.clone()来复制一个数组进行包装
public static ByteBuf copiedBuffer(byte[] array) {
return array.length == 0?EMPTY_BUFFER:wrappedBuffer((byte[])array.clone());
}
// 默认是堆内分配
public static ByteBuf wrappedBuffer(byte[] array) {
return (ByteBuf)(array.length == 0?EMPTY_BUFFER:new UnpooledHeapByteBuf(ALLOC, array, array.length));
}
// 也提供了堆外分配的方法
private static final ByteBufAllocator ALLOC;
public static ByteBuf directBuffer(int initialCapacity) {
return ALLOC.directBuffer(initialCapacity);
}
</pre>

相对底层的分配方法是使用ByteBufAllocator，Netty实现了PooledByteBufAllocator和UnpooledByteBufAllocator，前者使用了[jemalloc（一种malloc()的实现）](https://github.com/jemalloc/jemalloc)来分配内存，并且实现了对ByteBuf的池化以提高性能。后者分配的是未池化的ByteBuf，其分配方式与之前讲的一致。

<pre>Channel channel = ...;
ByteBufAllocator allocator = channel.alloc();
ByteBuf buffer = allocator.directBuffer();
do something.......
</pre>

为了优化内存使用率，**Netty提供了一套手动的方式来追踪不活跃对象**，像UnpooledHeapByteBuf这种分配在堆内的对象得益于JVM的GC管理，无需额外操心，而UnpooledDirectByteBuf是在堆外分配的，它的内部基于DirectByteBuffer，DirectByteBuffer会先向Bits类申请一个额度（Bits还拥有一个全局变量totalCapacity，记录了所有DirectByteBuffer总大小），每次申请前都会查看是否已经超过-XX:MaxDirectMemorySize所设置的上限，**如果超限就会尝试调用System.gc()**，**以试图回收一部分内存，然后休眠100毫秒，如果内存还是不足，则只能抛出OOM异常**。堆外内存的回收虽然有了这么一层保障，但为了提高性能与使用率，主动回收也是很有必要的。由于Netty还实现了ByteBuf的池化，像PooledHeapByteBuf和PooledDirectByteBuf就必须**依赖于手动的方式来进行回收**（放回池中）。

Netty使用了**引用计数器的方式来追踪那些不活跃的对象**。引用计数的接口为**ReferenceCounted**，它的思想很简单，只要ByteBuf对象的**引用计数大于0**，就保证该对象**不会被释放回收**，可以通过**手动调用release()与retain()**方法来操作该对象的引用计数值**递减或递增**。用户也可以通过自定义一个ReferenceCounted的实现类，以满足自定义的规则。

<pre>package io.netty.buffer;
public abstract class AbstractReferenceCountedByteBuf extends AbstractByteBuf {
// 由于ByteBuf的实例对象会非常多,所以这里没有将refCnt包装为AtomicInteger
// 而是使用一个全局的AtomicIntegerFieldUpdater来负责操作refCnt
private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater = AtomicIntegerFieldUpdater.newUpdater(AbstractReferenceCountedByteBuf.class, "refCnt");
// 每个ByteBuf的初始引用值都为1
private volatile int refCnt = 1;
public int refCnt() {
return this.refCnt;
}
protected final void setRefCnt(int refCnt) {
this.refCnt = refCnt;
}
public ByteBuf retain() {
return this.retain0(1);
}
// 引用计数值递增increment，increment必须大于0
public ByteBuf retain(int increment) {
return this.retain0(ObjectUtil.checkPositive(increment, "increment"));
}
public static int checkPositive(int i, String name) {
if(i <= 0) {
throw new IllegalArgumentException(name + ": " + i + " (expected: > 0)");
} else {
return i;
} 
}
// 使用CAS操作不断尝试更新值
private ByteBuf retain0(int increment) {
int refCnt;
int nextCnt;
do {
refCnt = this.refCnt;
nextCnt = refCnt + increment;
if(nextCnt <= increment) {
throw new IllegalReferenceCountException(refCnt, increment);
}
} while(!refCntUpdater.compareAndSet(this, refCnt, nextCnt));
return this;
}
public boolean release() {
return this.release0(1);
}
public boolean release(int decrement) {
return this.release0(ObjectUtil.checkPositive(decrement, "decrement"));
}
private boolean release0(int decrement) {
int refCnt;
do {
refCnt = this.refCnt;
if(refCnt < decrement) {
throw new IllegalReferenceCountException(refCnt, -decrement);
}
} while(!refCntUpdater.compareAndSet(this, refCnt, refCnt - decrement));
if(refCnt == decrement) {
this.deallocate();
return true;
} else {
return false;
}
}
protected abstract void deallocate();
}
</pre>

### Channel



* * *



Netty中的Channel与Java NIO的概念一样，都是对**一个实体或连接的抽象**，但**Netty提供了一套更加通用的API**。就以网络套接字为例，在Java中OIO与NIO是截然不同的两套API，假设你之前使用的是OIO而又想更改为NIO实现，那么几乎需要重写所有代码。而在Netty中，只需要更改短短几行代码（更改Channel与EventLoop的实现类，如把OioServerSocketChannel替换为NioServerSocketChannel），就可以完成OIO与NIO（或其他）之间的转换。

[![](http://wx2.sinaimg.cn/large/63503acbly1fm103i127ej20xe0f074y.jpg)](http://wx2.sinaimg.cn/large/63503acbly1fm103i127ej20xe0f074y.jpg)

每个Channel最终都会被分配一个**ChannelPipeline**和**ChannelConfig**，前者持有所有负责**处理入站与出站数据**以及**事件**的ChannelHandler，后者包含了该Channel的**所有配置设置**，并且**支持热更新**，由于不同的传输类型可能具有其特别的配置，所以该类可能会实现为ChannelConfig的不同子类。

**Channel是线程安全的**（与之后要讲的线程模型有关），因此你完全可以在多个线程中复用同一个Channel，就像如下代码所示。

<pre>final Channel channel = ...
final ByteBuf buffer = Unpooled.copiedBuffer("Hello,World!", CharsetUtil.UTF_8).retain();
Runnable writer = new Runnable() {
@Override
public void run() {
channel.writeAndFlush(buffer.duplicate());
}
};
Executor executor = Executors.newCachedThreadPool();
executor.execute(writer);
executor.execute(writer);
.......
</pre>

Netty除了支持常见的NIO与OIO，还内置了其他的传输类型。

| Nmae | Package | Description |
| --- | --- | --- |
| NIO | io.netty.channel.socket.nio | 以Java NIO为基础实现 |
| OIO | io.netty.channel.socket.oio | 以java.net为基础实现，使用阻塞I/O模型 |
| Epoll | io.netty.channel.epoll | 由JNI驱动epoll()实现的更高性能的非阻塞I/O，它**只能使用在Linux** |
| Local | io.netty.channel.local | **本地传输**，在JVM内部通过**管道**进行通信 |
| Embedded | io.netty.channel.embedded | 允许在不需要真实网络传输的环境下使用ChannelHandler，主要用于对ChannelHandler进行**测试** |

NIO、OIO、Epoll我们应该已经很熟悉了，下面主要说说Local与Embedded。

Local传输用于在**同一个JVM中**运行的客户端和服务器程序之间的**异步通信**，与服务器Channel相关联的SocketAddress并没有绑定真正的物理网络地址，它会被存储在注册表中，并在Channel关闭时注销。因此Local传输不会接受真正的网络流量，也就是说它不能与其他传输实现进行互操作。

Embedded传输主要用于对ChannelHandler进行**单元测试**，ChannelHandler是用于**处理消息的逻辑组件**，Netty通过将入站消息与出站消息都写入到EmbeddedChannel中的方式（提供了write/readInbound()与write/readOutbound()来读写入站与出站消息）来实现对ChannelHandler的单元测试。

### ChannelHandler



* * *



ChannelHandler充当了处理**入站**和**出站**数据的应用程序**逻辑的容器**，该类是基于**事件驱动**的，它会**响应相关的事件**然后去**调用其关联的回调函数**，例如当一个新的连接**被建立**时，ChannelHandler的**channelActive**()方法将**会被调用**。

关于入站消息和出站消息的数据流向定义，如果以客户端为主视角来说的话，那么从**客户端**流向**服务器**的数据被称为**出站**，反之为入站。

入站事件是可能被**入站数据或者相关的状态更改而触发的事件**，包括：连接已被激活、连接失活、读取入站数据、用户事件、发生异常等。

出站事件是**未来将会触发的某个动作的结果的事件**，这些动作包括：打开或关闭远程节点的连接、将数据写（或冲刷）到套接字。

ChannelHandler的主要用途包括：

*   对**入站与出站数据**的业务**逻辑处理**

*   **记录日志**

*   **将数据从一种格式转换为另一种格式**，实现编解码器。以一次HTTP协议（或者其他应用层协议）的流程为例，数据在网络传输时的单位为字节，当客户端发送请求到服务器时，服务器需要通过解码器（处理入站消息）将字节解码为协议的消息内容，服务器在发送响应的时候（处理出站消息），还需要通过编码器将消息内容编码为字节。

*   **捕获异常**

*   **提供Channel生命周期内的通知**，如Channel活动时与非活动时

Netty中到处都充满了异步与事件驱动，而**回调函数**正是用于**响应事件之后的操作**。由于异步会直接返回一个结果，所以Netty提供了ChannelFuture（实现了java.util.concurrent.Future）来作为异步调用返回的占位符，真正的结果会在未来的某个时刻完成，到时候就可以通过ChannelFuture对其进行访问，每个Netty的出站I/O操作都将会返回一个ChannelFuture。

Netty还提供了**ChannelFutureListener**接口来**监听ChannelFuture**是否成功，并采取对应的操作。

<pre>Channel channel = ...
ChannelFuture future = channel.connect(new InetSocketAddress("192.168.0.1",6666));
// 注册一个监听器
future.addListener(new ChannelFutureListener() {
@Override
public void operationComplete(ChannelFuture future) {
if (future.isSuccess()) {
// do something....
} else {
// 输出错误信息
Throwable cause = future.cause();
cause.printStackTrace();
// do something....
}
}
});
</pre>

ChannelFutureListener接口中还提供了几个简单的默认实现，方便我们使用。

<pre>package io.netty.channel;
import io.netty.channel.ChannelFuture;
import io.netty.util.concurrent.GenericFutureListener;
public interface ChannelFutureListener extends GenericFutureListener<ChannelFuture> {
// 在Future完成时关闭
ChannelFutureListener CLOSE = new ChannelFutureListener() {
public void operationComplete(ChannelFuture future) {
future.channel().close();
}
};
// 如果失败则关闭
ChannelFutureListener CLOSE_ON_FAILURE = new ChannelFutureListener() {
public void operationComplete(ChannelFuture future) {
if(!future.isSuccess()) {
future.channel().close();
}
}
};
// 将异常信息传递给下一个ChannelHandler
ChannelFutureListener FIRE_EXCEPTION_ON_FAILURE = new ChannelFutureListener() {
public void operationComplete(ChannelFuture future) {
if(!future.isSuccess()) {
future.channel().pipeline().fireExceptionCaught(future.cause());
}
}
};
}
</pre>

ChannelHandler接口**定义了对它生命周期进行监听的回调函数**，在ChannelHandler被添加到ChannelPipeline或者被移除时都会调用这些函数。

<pre>package io.netty.channel;
public interface ChannelHandler {
void handlerAdded(ChannelHandlerContext var1) throws Exception;
void handlerRemoved(ChannelHandlerContext var1) throws Exception;
/** @deprecated */
@Deprecated
void exceptionCaught(ChannelHandlerContext var1, Throwable var2) throws Exception;
// 该注解表明这个ChannelHandler可被其他线程复用
@Inherited
@Documented
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Sharable {
}
}

</pre>

**入站消息与出站消息**由其对应的接口**ChannelInboundHandler与ChannelOutboundHandle**r负责，这两个接口定义了监听Channel的**生命周期的状态改变事件**的回调函数。

<pre>package io.netty.channel;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
public interface ChannelInboundHandler extends ChannelHandler {
// 当channel被注册到EventLoop时被调用
void channelRegistered(ChannelHandlerContext var1) throws Exception;
// 当channel已经被创建，但还未注册到EventLoop（或者从EventLoop中注销）被调用
void channelUnregistered(ChannelHandlerContext var1) throws Exception;
// 当channel处于活动状态（连接到远程节点）被调用
void channelActive(ChannelHandlerContext var1) throws Exception;
// 当channel处于非活动状态（没有连接到远程节点）被调用
void channelInactive(ChannelHandlerContext var1) throws Exception;
// 当从channel读取数据时被调用
void channelRead(ChannelHandlerContext var1, Object var2) throws Exception;
// 当channel的上一个读操作完成时被调用
void channelReadComplete(ChannelHandlerContext var1) throws Exception;
// 当ChannelInboundHandler.fireUserEventTriggered()方法被调用时被调用
void userEventTriggered(ChannelHandlerContext var1, Object var2) throws Exception;
// 当channel的可写状态发生改变时被调用
void channelWritabilityChanged(ChannelHandlerContext var1) throws Exception;
// 当处理过程中发生异常时被调用
void exceptionCaught(ChannelHandlerContext var1, Throwable var2) throws Exception;
}

package io.netty.channel;
import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelPromise;
import java.net.SocketAddress;
public interface ChannelOutboundHandler extends ChannelHandler {
// 当请求将Channel绑定到一个地址时被调用
// ChannelPromise是ChannelFuture的一个子接口，定义了如setSuccess(),setFailure()等方法
void bind(ChannelHandlerContext var1, SocketAddress var2, ChannelPromise var3) throws Exception;
// 当请求将Channel连接到远程节点时被调用
void connect(ChannelHandlerContext var1, SocketAddress var2, SocketAddress var3, ChannelPromise var4) throws Exception;
// 当请求将Channel从远程节点断开时被调用
void disconnect(ChannelHandlerContext var1, ChannelPromise var2) throws Exception;
// 当请求关闭Channel时被调用
void close(ChannelHandlerContext var1, ChannelPromise var2) throws Exception;
// 当请求将Channel从它的EventLoop中注销时被调用
void deregister(ChannelHandlerContext var1, ChannelPromise var2) throws Exception;
// 当请求从Channel读取数据时被调用
void read(ChannelHandlerContext var1) throws Exception;
// 当请求通过Channel将数据写到远程节点时被调用
void write(ChannelHandlerContext var1, Object var2, ChannelPromise var3) throws Exception;
// 当请求通过Channel将缓冲中的数据冲刷到远程节点时被调用
void flush(ChannelHandlerContext var1) throws Exception;
}
</pre>

通过实现ChannelInboundHandler或者ChannelOutboundHandler就可以完成用户自定义的应用逻辑处理程序，不过Netty已经帮你实**现了一些基本操作，用户只需要继承并扩展ChannelInboundHandlerAdapter或ChannelOutboundHandlerAdapter**来作为自定义实现的起始点。

ChannelInboundHandlerAdapter与ChannelOutboundHandlerAdapter都继承于ChannelHandlerAdapter，该抽象类简单实现了ChannelHandler接口。

<pre>public abstract class ChannelHandlerAdapter implements ChannelHandler {
boolean added;
public ChannelHandlerAdapter() {
}
// 该方法不允许将此ChannelHandler共享复用
protected void ensureNotSharable() {
if(this.isSharable()) {
throw new IllegalStateException("ChannelHandler " + this.getClass().getName() + " is not allowed to be shared");
}
}
// 使用反射判断实现类有没有@Sharable注解，以确认该类是否为可共享复用的
public boolean isSharable() {
Class clazz = this.getClass();
Map cache = InternalThreadLocalMap.get().handlerSharableCache();
Boolean sharable = (Boolean)cache.get(clazz);
if(sharable == null) {
sharable = Boolean.valueOf(clazz.isAnnotationPresent(Sharable.class));
cache.put(clazz, sharable);
}
return sharable.booleanValue();
}
public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
}
public void handlerRemoved(ChannelHandlerContext ctx) throws Exception {
}
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
ctx.fireExceptionCaught(cause);
}
}
</pre>

ChannelInboundHandlerAdapter与ChannelOutboundHandlerAdapter**默认只是简单地将请求传递给ChannelPipeline中的下一个ChannelHandler**，源码如下：

<pre>public class ChannelInboundHandlerAdapter extends ChannelHandlerAdapter implements ChannelInboundHandler {
public ChannelInboundHandlerAdapter() {
}
public void channelRegistered(ChannelHandlerContext ctx) throws Exception {
ctx.fireChannelRegistered();
}
public void channelUnregistered(ChannelHandlerContext ctx) throws Exception {
ctx.fireChannelUnregistered();
}
public void channelActive(ChannelHandlerContext ctx) throws Exception {
ctx.fireChannelActive();
}
public void channelInactive(ChannelHandlerContext ctx) throws Exception {
ctx.fireChannelInactive();
}
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
ctx.fireChannelRead(msg);
}
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
ctx.fireChannelReadComplete();
}
public void userEventTriggered(ChannelHandlerContext ctx, Object evt) throws Exception {
ctx.fireUserEventTriggered(evt);
}
public void channelWritabilityChanged(ChannelHandlerContext ctx) throws Exception {
ctx.fireChannelWritabilityChanged();
}
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
ctx.fireExceptionCaught(cause);
}
}
public class ChannelOutboundHandlerAdapter extends ChannelHandlerAdapter implements ChannelOutboundHandler {
public ChannelOutboundHandlerAdapter() {
}
public void bind(ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise) throws Exception {
ctx.bind(localAddress, promise);
}
public void connect(ChannelHandlerContext ctx, SocketAddress remoteAddress, SocketAddress localAddress, ChannelPromise promise) throws Exception {
ctx.connect(remoteAddress, localAddress, promise);
}
public void disconnect(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
ctx.disconnect(promise);
}
public void close(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
ctx.close(promise);
}
public void deregister(ChannelHandlerContext ctx, ChannelPromise promise) throws Exception {
ctx.deregister(promise);
}
public void read(ChannelHandlerContext ctx) throws Exception {
ctx.read();
}
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
ctx.write(msg, promise);
}
public void flush(ChannelHandlerContext ctx) throws Exception {
ctx.flush();
}
}
</pre>

对于处理入站消息，另外一种选择是**继承SimpleChannelInboundHandler**，它是Netty的一个继承于ChannelInboundHandlerAdapter的抽象类，并在其之上实现了**自动释放资源的功能**。

我们在了解ByteBuf时就已经知道了**Netty使用了一套自己实现的引用计数算法来主动释放资源**，假设你的ChannelHandler继承于ChannelInboundHandlerAdapter或ChannelOutboundHandlerAdapter，那么你就有责任去管理你所分配的ByteBuf，一般来说，一个消息对象（**ByteBuf**）已经被消费（或丢弃）了，**并不会传递给ChannelHandler链中的下一个处理器**（如果该消息到达了实际的传输层，那么当它被写入或Channel关闭时，都会被自动释放），所以你就需要去手动释放它。通过一个简单的工具类**ReferenceCountUtil的release方法**，就可以做到这一点。

<pre>// 这个泛型为消息对象的类型
public abstract class SimpleChannelInboundHandler<I> extends ChannelInboundHandlerAdapter {
private final TypeParameterMatcher matcher;
private final boolean autoRelease;
protected SimpleChannelInboundHandler() {
	this(true);
}

protected SimpleChannelInboundHandler(boolean autoRelease) {
        this.matcher = TypeParameterMatcher.find(this, SimpleChannelInboundHandler.class, "I");
	    this.autoRelease = autoRelease;
}

protected SimpleChannelInboundHandler(Class<? extends I> inboundMessageType) {
		this(inboundMessageType, true);
}

protected SimpleChannelInboundHandler(Class<? extends I> inboundMessageType, boolean autoRelease) {
            this.matcher = TypeParameterMatcher.get(inboundMessageType);
            this.autoRelease = autoRelease;
}

public boolean acceptInboundMessage(Object msg) throws Exception {
		return this.matcher.match(msg);
}

// SimpleChannelInboundHandler只是替你做了ReferenceCountUtil.release()
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
boolean release = true;
try {
    if(this.acceptInboundMessage(msg)) {
  		   this.channelRead0(ctx, msg);
    } else {
            release = false;
            ctx.fireChannelRead(msg);
    }
} finally {
        if(this.autoRelease && release) {
            //ByteBuf的释放
        	ReferenceCountUtil.release(msg);
		}
	}
}
// 这个方法才是我们需要实现的方法
protected abstract void channelRead0(ChannelHandlerContext var1, I var2) throws Exception;
}
// ReferenceCountUtil中的源码，release方法对消息对象的类型进行判断然后调用它的release()方法
public static boolean release(Object msg) {
return msg instanceof ReferenceCounted?((ReferenceCounted)msg).release():false;
}
</pre>

### ChannelPipeline



* * *



为了**模块化与解耦合**，不可能由一个ChannelHandler来完成所有应用逻辑，所以Netty采用了拦截器链的设计。ChannelPipeline就是用来**管理ChannelHandler实例链的容器**，它的职责就是**保证实例链的流动**。

每一个新创建的Channel都将会被分配一个新的ChannelPipeline，这种**关联关系是永久**性的，一个Channel一生只能对应一个ChannelPipeline。

[![](http://wx3.sinaimg.cn/large/63503acbly1fm1er9l4jfj213h0fcq3d.jpg)](http://wx3.sinaimg.cn/large/63503acbly1fm1er9l4jfj213h0fcq3d.jpg)

一个入站事件被触发时，它会先从ChannelPipeline的最左端（头部）开始一直传播到ChannelPipeline的最右端（尾部），而出站事件正好与入站事件顺序相反（从最右端一直传播到最左端）。这个**顺序是定死**的，Netty总是将ChannelPipeline的**入站口作为头部**，而将**出站口作为尾部**。在事件传播的过程中，ChannelPipeline会判断下一个ChannelHandler的类型是否和事件的运动方向相匹配，如果不匹配，就跳过该ChannelHandler并继续检查下一个（保证入站事件只会被ChannelInboundHandler处理），**一个**ChannelHandler也可以**同时实现**ChannelInboundHandler与ChannelOutboundHandler，它在**入站事件与出站事件中都会被调用。**

在阅读ChannelHandler的源码时，发现很多方法需要一个ChannelHandlerContext类型的参数，该接口是ChannelPipeline与ChannelHandler之间相关联的关键。ChannelHandlerContext可以通知ChannelPipeline中的当前ChannelHandler的下一个ChannelHandler，还可以动态地改变当前ChannelHandler在ChannelPipeline中的位置（通过调用ChannelPipeline中的各种方法来修改）。

ChannelHandlerContext负责了在同一个ChannelPipeline中的ChannelHandler与其他ChannelHandler之间的交互，每个ChannelHandlerContext都对应了一个ChannelHandler。在DefaultChannelPipeline的源码中，已经表现的很明显了。

<pre>public class DefaultChannelPipeline implements ChannelPipeline {
.........
// 头部节点和尾部节点的引用变量
// ChannelHandlerContext在ChannelPipeline中是以链表的形式组织的
final AbstractChannelHandlerContext head;
final AbstractChannelHandlerContext tail;
.........
// 添加一个ChannelHandler到链表尾部
public final ChannelPipeline addLast(String name, ChannelHandler handler) {
return this.addLast((EventExecutorGroup)null, name, handler);
}
public final ChannelPipeline addLast(EventExecutorGroup group, String name, ChannelHandler handler) {
final AbstractChannelHandlerContext newCtx;
synchronized(this) {
// 检查ChannelHandler是否为一个共享对象(@Sharable)
// 如果该ChannelHandler没有@Sharable注解，并且是已被添加过的那么就抛出异常
checkMultiplicity(handler);
// 返回一个DefaultChannelHandlerContext，注意该对象持有了传入的ChannelHandler
newCtx = this.newContext(group, this.filterName(name, handler), handler);
this.addLast0(newCtx);
// 如果当前ChannelPipeline没有被注册，那么就先加到未决链表中
if(!this.registered) {
newCtx.setAddPending();
this.callHandlerCallbackLater(newCtx, true);
return this;
}
// 否则就调用ChannelHandler中的handlerAdded()
EventExecutor executor = newCtx.executor();
if(!executor.inEventLoop()) {
newCtx.setAddPending();
executor.execute(new Runnable() {
public void run() {
DefaultChannelPipeline.this.callHandlerAdded0(newCtx);
}
});
return this;
}
}
this.callHandlerAdded0(newCtx);
return this;
}
// 将新的ChannelHandlerContext插入到尾部与尾部之前的节点之间
private void addLast0(AbstractChannelHandlerContext newCtx) {
AbstractChannelHandlerContext prev = this.tail.prev;
newCtx.prev = prev;
newCtx.next = this.tail;
prev.next = newCtx;
this.tail.prev = newCtx;
}
.....
}
</pre>

ChannelHandlerContext还定义了许多与Channel和ChannelPipeline重合的方法（像read()、write()、connect()这些用于出站的方法或者如fireChannelXXXX()这样用于**入站的方法**），不同之处在于**调用Channel或者ChannelPipeline上的这些方法，它们将会从头沿着整个ChannelHandler实例链进行传播，而调用位于ChannelHandlerContext上的相同方法，则会从当前所关联的ChannelHandler开始，且只会传播给实例链中的下一个ChannelHandler**。而且，**事件之间的移动**（从一个ChannelHandler到下一个ChannelHandler）也是**通过ChannelHandlerContext中的方法调用完成**的。

<pre>public class DefaultChannelPipeline implements ChannelPipeline {
public final ChannelPipeline fireChannelRead(Object msg) {
// 注意这里将头节点传入了进去
AbstractChannelHandlerContext.invokeChannelRead(this.head, msg);
return this;
}
}
---------------------------------------------------------------

abstract class AbstractChannelHandlerContext extends DefaultAttributeMap implements ChannelHandlerContext, ResourceLeakHint {
static void invokeChannelRead(final AbstractChannelHandlerContext next, Object msg) {
final Object m = next.pipeline.touch(ObjectUtil.checkNotNull(msg, "msg"), next);
EventExecutor executor = next.executor();
if(executor.inEventLoop()) {
next.invokeChannelRead(m);
} else {
executor.execute(new Runnable() {
public void run() {
next.invokeChannelRead(m);
}
});
}
}
private void invokeChannelRead(Object msg) {
if(this.invokeHandler()) {
try {
((ChannelInboundHandler)this.handler()).channelRead(this, msg);
} catch (Throwable var3) {
this.notifyHandlerException(var3);
}
} else {
// 寻找下一个ChannelHandler
this.fireChannelRead(msg);
}
}
public ChannelHandlerContext fireChannelRead(Object msg) {
invokeChannelRead(this.findContextInbound(), msg);
return this;
}
private AbstractChannelHandlerContext findContextInbound() {
AbstractChannelHandlerContext ctx = this;
do {
ctx = ctx.next;
} while(!ctx.inbound); // 直到找到一个ChannelInboundHandler
return ctx;
}
}
</pre>

### EventLoop



* * *



为了最大限度地提供高性能和可维护性，Netty设计了一套强大又易用的线程模型。在一个网络框架中，最重要的能力是能够快速高效地**处理在连接的生命周期内发生的各种事件**，与之相匹配的程序构造被称为**事件循环**，Netty定义了接口EventLoop来负责这项工作。

如果是经常用Java进行多线程开发的童鞋想必经常会使用到线程池，也就是Executor这套API。Netty就是从Executor（java.util.concurrent）之上扩展了自己的EventExecutorGroup（io.netty.util.concurrent），同时为了与Channel的事件进行交互，还扩展了EventLoopGroup接口（io.netty.channel）。在io.netty.util.concurrent包下的EventExecutorXXX负责实现**线程并发**相关的工作，而在io.netty.channel包下的EventLoopXXX负责**实现网络编程**相关的工作（处理Channel中的事件）。

[![](http://wx3.sinaimg.cn/large/63503acbly1fm296hz0p9j20ff0kc3z2.jpg)](http://wx3.sinaimg.cn/large/63503acbly1fm296hz0p9j20ff0kc3z2.jpg)

在Netty的线程模型中，一个EventLoop将由一个永远不会改变的Thread驱动，而一个Channel一生只会使用一个EventLoop（但是一个EventLoop可能会被指派用于服务多个Channel），在Channel中的所有I/O操作和事件都由EventLoop中的线程处理，也就是说**一个Channel的一生之中都只会使用到一个线程**。不过在Netty3，只有入站事件会被EventLoop处理，所有出站事件都会由调用线程处理，这种设计导致了ChannelHandler的线程安全问题。Netty4简化了线程模型，通过在同一个线程处理所有事件，既解决了这个问题，还提供了一个更加简单的架构。

<pre>package io.netty.channel;
public abstract class SingleThreadEventLoop extends SingleThreadEventExecutor implements EventLoop {
protected static final int DEFAULT_MAX_PENDING_TASKS = Math.max(16, SystemPropertyUtil.getInt("io.netty.eventLoop.maxPendingTasks", 2147483647));
    //内部队列
private final Queue<Runnable> tailTasks;
protected SingleThreadEventLoop(EventLoopGroup parent, ThreadFactory threadFactory, boolean addTaskWakesUp) {
this(parent, threadFactory, addTaskWakesUp, DEFAULT_MAX_PENDING_TASKS, RejectedExecutionHandlers.reject());
}
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor, boolean addTaskWakesUp) {
this(parent, executor, addTaskWakesUp, DEFAULT_MAX_PENDING_TASKS, RejectedExecutionHandlers.reject());
}
protected SingleThreadEventLoop(EventLoopGroup parent, ThreadFactory threadFactory, boolean addTaskWakesUp, int maxPendingTasks, RejectedExecutionHandler rejectedExecutionHandler) {
super(parent, threadFactory, addTaskWakesUp, maxPendingTasks, rejectedExecutionHandler);
this.tailTasks = this.newTaskQueue(maxPendingTasks);
}
protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor, boolean addTaskWakesUp, int maxPendingTasks, RejectedExecutionHandler rejectedExecutionHandler) {
super(parent, executor, addTaskWakesUp, maxPendingTasks, rejectedExecutionHandler);
this.tailTasks = this.newTaskQueue(maxPendingTasks);
}
// 返回它所在的EventLoopGroup
public EventLoopGroup parent() {
return (EventLoopGroup)super.parent();
}
public EventLoop next() {
return (EventLoop)super.next();
}
// 注册Channel,这里ChannelPromise和Channel关联到了一起
public ChannelFuture register(Channel channel) {
return this.register((ChannelPromise)(new DefaultChannelPromise(channel, this)));
}
public ChannelFuture register(ChannelPromise promise) {
ObjectUtil.checkNotNull(promise, "promise");
promise.channel().unsafe().register(this, promise);
return promise;
}
// 剩下这些函数都是用于调度任务
public final void executeAfterEventLoopIteration(Runnable task) {
ObjectUtil.checkNotNull(task, "task");
if(this.isShutdown()) {
reject();
}
if(!this.tailTasks.offer(task)) {
this.reject(task);
}
if(this.wakesUpForTask(task)) {
this.wakeup(this.inEventLoop());
}
}
final boolean removeAfterEventLoopIterationTask(Runnable task) {
return this.tailTasks.remove(ObjectUtil.checkNotNull(task, "task"));
}
protected boolean wakesUpForTask(Runnable task) {
return !(task instanceof SingleThreadEventLoop.NonWakeupRunnable);
}
protected void afterRunningAllTasks() {
this.runAllTasksFrom(this.tailTasks);
}
protected boolean hasTasks() {
return super.hasTasks() || !this.tailTasks.isEmpty();
}
public int pendingTasks() {
return super.pendingTasks() + this.tailTasks.size();
}
interface NonWakeupRunnable extends Runnable {
}
}
</pre>

为了确保一个Channel的整个生命周期中的I/O事件会被一个EventLoop负责，Netty通过**inEventLoop()**方法来**判断当前执行的线程的身份**，确定它是否是分配给当前Channel以及它的EventLoop的那一个线程。

如果当前（调用）线程正是EventLoop中的线程，那么所提交的任务将会被**(true)直接执行**，否则，EventLoop将调度该任务以便**(false)稍后执行**，并将它**放入内部的任务队列**（每个EventLoop都有它自己的任务队列，SingleThreadEventLoop的源码就能发现很多用于调度内部任务队列的方法），在下次处理它的事件时，将会执行队列中的那些任务。这种设计可以让任何线程与Channel直接交互，而无需在ChannelHandler中进行额外的同步。

从性能上来考虑，千万**不要将一个需要长时间来运行的任务放入到任务队列中**，它会影响到该队列中的其他任务的执行。**解决方案**是**使用一个专门的EventExecutor来执行它**（ChannelPipeline提供了带有EventExecutorGroup参数的addXXX()方法，该方法可以**将传入的ChannelHandler绑定到你传入的EventExecutor之中**），这样它就会在另一条线程中执行，与其他任务隔离。

<pre>public abstract class SingleThreadEventExecutor extends AbstractScheduledEventExecutor implements OrderedEventExecutor {
.....
public void execute(Runnable task) {
if(task == null) {
throw new NullPointerException("task");
} else {
boolean inEventLoop = this.inEventLoop();
if(inEventLoop) {
this.addTask(task);
} else {
this.startThread();
this.addTask(task);
if(this.isShutdown() && this.removeTask(task)) {
reject();
}
}
if(!this.addTaskWakesUp && this.wakesUpForTask(task)) {
this.wakeup(inEventLoop);
}
}
}
public boolean inEventLoop(Thread thread) {
return thread == this.thread;
}
.....
}
</pre>

EventLoopGroup**负责管理和分配EventLoop（创建EventLoop和为每个新创建的Channel分配EventLoop），根据不同的传输类型，EventLoop的创建和分配方式也不同**。例如，使用NIO传输类型，EventLoopGroup就会只使用较少的EventLoop（一个EventLoop服务于多个Channel），这是因为NIO基于I/O多路复用，一个线程可以处理多个连接，而如果使用的是OIO，那么新创建一个Channel（连接）就需要分配一个EventLoop（线程）。

### Bootstrap



* * *



在深入了解地Netty的核心组件之后，发现它们的**设计**都很**模块化**，如果想要实现你自己的应用程序，就需要**将这些组件组装到一起**。Netty通过Bootstrap类，以对一个Netty应用程序进行配置（**组装各个组件**），并最终使它运行起来。对于客户端程序和服务器程序所使用到的Bootstrap类是不同的，后者需要使用ServerBootstrap，这样设计是因为，在如TCP这样有连接的协议中，服务器程序往往需要一个以上的Channel，通过父Channel来接受来自客户端的连接，然后创建子Channel用于它们之间的通信，而像UDP这样无连接的协议，它不需要每个连接都创建子Channel，只需要一个Channel即可。

一个比较明显的差异就是Bootstrap与ServerBootstrap的group()方法，后者提供了一个接收2个EventLoopGroup的版本。

<pre>// 该方法在Bootstrap的父类AbstractBootstrap中，泛型B为它当前子类的类型（为了链式调用）
public B group(EventLoopGroup group) {
if(group == null) {
throw new NullPointerException("group");
} else if(this.group != null) {
throw new IllegalStateException("group set already");
} else {
this.group = group;
return this;
}
}
// ServerBootstrap中的实现，它也支持只用一个EventLoopGroup
public ServerBootstrap group(EventLoopGroup group) {
return this.group(group, group);
}
public ServerBootstrap group(EventLoopGroup parentGroup, EventLoopGroup childGroup) {
super.group(parentGroup);
if(childGroup == null) {
throw new NullPointerException("childGroup");
} else if(this.childGroup != null) {
throw new IllegalStateException("childGroup set already");
} else {
this.childGroup = childGroup;
return this;
}
}
</pre>

Bootstrap其实没有什么可以好说的，它就只是一个**装配工**，将各个组件拼装组合到一起，然后进行一些配置，有关它的详细API请参考[Netty JavaDoc](http://netty.io/4.1/api/index.html)。

### Echo示例

下面我们将通过一个经典的Echo客户端与服务器的例子，来梳理一遍创建Netty应用的流程。

首先实现的是服务器，我们先实现一个EchoServerInboundHandler，处理入站消息。

<pre>public class EchoServerInboundHandler extends ChannelInboundHandlerAdapter {
@Override
public void channelRead(ChannelHandlerContext ctx, Object msg) throws Exception {
ByteBuf in = (ByteBuf) msg;
System.out.printf("Server received: %s \n", in.toString(CharsetUtil.UTF_8));
// 由于读事件不是一次性就能把完整消息发送过来的，这里并没有调用writeAndFlush
ctx.write(in); // 直接把消息写回给客户端(会被出站消息处理器处理,不过我们的应用没有实现任何出站消息处理器)
}
@Override
public void channelReadComplete(ChannelHandlerContext ctx) throws Exception {
// 等读事件已经完成时,冲刷之前写数据的缓冲区
// 然后添加了一个监听器，它会在Future完成时进行关闭该Channel.
ctx.writeAndFlush(Unpooled.EMPTY_BUFFER)
.addListener(ChannelFutureListener.CLOSE);
}
// 处理异常，输出异常信息，然后关闭Channel
@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
cause.printStackTrace();
ctx.close();
}
}
</pre>

服务器的应用逻辑只有这么多，剩下就是用ServerBootstrap进行配置了。

<pre>public class EchoServer {
private final int port;
public EchoServer(int port) {
this.port = port;
}
public void start() throws Exception {
final EchoServerInboundHandler serverHandler = new EchoServerInboundHandler();
EventLoopGroup group = new NioEventLoopGroup(); // 传输类型使用NIO
try {
ServerBootstrap b = new ServerBootstrap();
b.group(group) // 配置EventLoopGroup
.channel(NioServerSocketChannel.class) // 配置Channel的类型
.localAddress(new InetSocketAddress(port)) // 配置端口号
.childHandler(new ChannelInitializer<SocketChannel>() {
// 实现一个ChannelInitializer，它可以方便地添加多个ChannelHandler
@Override
protected void initChannel(SocketChannel socketChannel) throws Exception {
socketChannel.pipeline().addLast(serverHandler);
}
});
// 绑定地址，同步等待它完成
ChannelFuture f = b.bind().sync();
// 关闭这个Future
f.channel().closeFuture().sync();
} finally {
// 关闭应用程序，一般来说Netty应用只需要调用这个方法就够了
group.shutdownGracefully().sync();
}
}
public static void main(String[] args) throws Exception {
if (args.length != 1) {
System.err.printf(
"Usage: %s <port> \n",
EchoServer.class.getSimpleName()
);
return;
}
int port = Integer.parseInt(args[0]);
new EchoServer(port).start();
}
}
</pre>

接下来实现客户端，同样需要先实现一个入站消息处理器。

<pre>public class EchoClientInboundHandler extends SimpleChannelInboundHandler<ByteBuf> {
/**
* 我们在Channel连接到远程节点直接发送一条消息给服务器
*/
@Override
public void channelActive(ChannelHandlerContext ctx) throws Exception {
ctx.writeAndFlush(Unpooled.copiedBuffer("Hello, Netty!", CharsetUtil.UTF_8));
}
@Override
protected void channelRead0(ChannelHandlerContext channelHandlerContext, ByteBuf byteBuf) throws Exception {
// 输出从服务器Echo的消息
System.out.printf("Client received: %s \n", byteBuf.toString(CharsetUtil.UTF_8));
}
@Override
public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) throws Exception {
cause.printStackTrace();
ctx.close();
}
}
</pre>

然后配置客户端。

<pre>public class EchoClient {
private final String host;
private final int port;
public EchoClient(String host, int port) {
this.host = host;
this.port = port;
}
public void start() throws Exception {
EventLoopGroup group = new NioEventLoopGroup();
try {
Bootstrap b = new Bootstrap();
b.group(group)
.channel(NioSocketChannel.class)
.remoteAddress(new InetSocketAddress(host, port)) // 服务器的地址
.handler(new ChannelInitializer<SocketChannel>() {
@Override
protected void initChannel(SocketChannel socketChannel) throws Exception {
socketChannel.pipeline().addLast(new EchoClientInboundHandler());
}
});
ChannelFuture f = b.connect().sync(); // 连接到服务器
f.channel().closeFuture().sync();
} finally {
group.shutdownGracefully().sync();
}
}
public static void main(String[] args) throws Exception {
if (args.length != 2) {
System.err.printf("Usage: %s <host> <port> \n", EchoClient.class.getSimpleName());
return;
}
String host = args[0];
int port = Integer.parseInt(args[1]);
new EchoClient(host, port).start();
}
}
</pre>

实现一个Netty应用程序就是如此简单，用户大多数都是在编写各种应用逻辑的ChannelHandler（或者使用Netty内置的各种实用ChannelHandler），然后只需要将它们全部添加到ChannelPipeline即可。

## 参考文献



* * *



*   [Netty: Home](https://netty.io/)

*   [Chapter 6\. I/O Multiplexing: The select and poll Functions - Shichao’s Notes](https://notes.shichao.io/unp/ch6/#io-multiplexing-model)

*   [epoll(7) - Linux manual page](http://man7.org/linux/man-pages/man7/epoll.7.html)

*   [Java NIO](http://tutorials.jenkov.com/java-nio/)

*   [Netty: Home](https://netty.io/)

*   [Chapter 6\. I/O Multiplexing: The select and poll Functions - Shichao’s Notes](https://notes.shichao.io/unp/ch6/#io-multiplexing-model)

*   [epoll(7) - Linux manual page](http://man7.org/linux/man-pages/man7/epoll.7.html)

*   [Java NIO](http://tutorials.jenkov.com/java-nio/)
