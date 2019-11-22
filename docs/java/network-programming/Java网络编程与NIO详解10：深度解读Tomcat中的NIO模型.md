# Table of Contents

  * [一、I/O复用模型解读](#一、io复用模型解读)
  * [二、TOMCAT对IO模型的支持](#二、tomcat对io模型的支持)
  * [三、TOMCAT中NIO的配置与使用](#三、tomcat中nio的配置与使用)
  * [四、NioEndpoint组件关系图解读](#四、nioendpoint组件关系图解读)
  * [五、NioEndpoint执行序列图](#五、nioendpoint执行序列图)
  * [六、NioEndpoint源码解读](#六、nioendpoint源码解读)
  * [七、关于性能](#七、关于性能)
  * [八、总结](#八、总结)


本文转自：http://www.sohu.com/a/203838233_827544

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
> 摘要: I/O复用模型，是同步非阻塞，这里的非阻塞是指I/O读写，对应的是recvfrom操作，因为数据报文已经准备好，无需阻塞。

说它是同步，是因为，这个执行是在一个线程里面执行的。有时候，还会说它又是阻塞的，实际上是指阻塞在select上面，必须等到读就绪、写就绪等网络事件。

## 一、I/O复用模型解读

Tomcat的NIO是基于I/O复用来实现的。对这点一定要清楚，不然我们的讨论就不在一个逻辑线上。下面这张图学习过I/O模型知识的一般都见过，出自《UNIX网络编程》，I/O模型一共有阻塞式I/O，非阻塞式I/O，I/O复用(select/poll/epoll)，信号驱动式I/O和异步I/O。这篇文章讲的是I/O复用。

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/0ab0b70be5a64b0e9014de867235da73.png)

IO复用.png

这里先来说下用户态和内核态，直白来讲，如果线程执行的是用户代码，当前线程处在用户态，如果线程执行的是内核里面的代码，当前线程处在内核态。更深层来讲，操作系统为代码所处的特权级别分了4个级别。

不过现代操作系统只用到了0和3两个级别。0和3的切换就是用户态和内核态的切换。更详细的可参照《深入理解计算机操作系统》。I/O复用模型，是同步非阻塞，这里的非阻塞是指I/O读写，对应的是recvfrom操作，因为数据报文已经准备好，无需阻塞。

说它是同步，是因为，这个执行是在一个线程里面执行的。有时候，还会说它又是阻塞的，实际上是指阻塞在select上面，必须等到读就绪、写就绪等网络事件。有时候我们又说I/O复用是多路复用，这里的多路是指N个连接，每一个连接对应一个channel，或者说多路就是多个channel。

复用，是指多个连接复用了一个线程或者少量线程(在Tomcat中是Math.min(2,Runtime.getRuntime().availableProcessors()))。

上面提到的网络事件有连接就绪，接收就绪，读就绪，写就绪四个网络事件。I/O复用主要是通过Selector复用器来实现的，可以结合下面这个图理解上面的叙述。

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/3a18d371c9c848479ca39d8eb4e57ce4.jpeg)

Selector图解.png

## 二、TOMCAT对IO模型的支持

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/983c122c2a924e01b9f898279e2bd0b5.jpeg)

tomcat支持IO类型图.png

tomcat从6以后开始支持NIO模型，实现是基于JDK的java.nio包。这里可以看到对read body 和response body是Blocking的。关于这点在第6.3节源代码阅读有重点介绍。

## 三、TOMCAT中NIO的配置与使用

在Connector节点配置protocol="org.apache.coyote.http11.Http11NioProtocol"，Http11NioProtocol协议下默认最大连接数是10000，也可以重新修改maxConnections的值，同时我们可以设置最大线程数maxThreads，这里设置的最大线程数就是Excutor的线程池的大小。

在BIO模式下实际上是没有maxConnections，即使配置也不会生效，BIO模式下的maxConnections是保持跟maxThreads大小一致，因为它是一请求一线程模式。

## 四、NioEndpoint组件关系图解读

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/4e1a924a60974c76ab5115007562e563.jpeg)

tomcatnio组成.png

我们要理解tomcat的nio最主要就是对NioEndpoint的理解。它一共包含LimitLatch、Acceptor、Poller、SocketProcessor、Excutor5个部分。

LimitLatch是连接控制器，它负责维护连接数的计算，nio模式下默认是10000，达到这个阈值后，就会拒绝连接请求。Acceptor负责接收连接，默认是1个线程来执行，将请求的事件注册到事件列表。

有Poller来负责轮询，Poller线程数量是cpu的核数Math.min(2,Runtime.getRuntime().availableProcessors())。由Poller将就绪的事件生成SocketProcessor同时交给Excutor去执行。Excutor线程池的大小就是我们在Connector节点配置的maxThreads的值。

在Excutor的线程中，会完成从socket中读取http request，解析成HttpServletRequest对象，分派到相应的servlet并完成逻辑，然后将response通过socket发回client。

在从socket中读数据和往socket中写数据的过程，并没有像典型的非阻塞的NIO的那样，注册OP_READ或OP_WRITE事件到主Selector，而是直接通过socket完成读写，这时是阻塞完成的，但是在timeout控制上，使用了NIO的Selector机制，但是这个Selector并不是Poller线程维护的主Selector，而是BlockPoller线程中维护的Selector，称之为辅Selector。详细源代码可以参照 第6.3节。

## 五、NioEndpoint执行序列图

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/d69c2ef5110d4706aa7284c616d62927.jpeg)

tomcatnio序列图.png

在下一小节NioEndpoint源码解读中我们将对步骤1-步骤11依次找到对应的代码来说明。

## 六、NioEndpoint源码解读

6.1、初始化

无论是BIO还是NIO，开始都会初始化连接限制，不可能无限增大，NIO模式下默认是10000。

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/332338f6d2c8488e9b35cd4fd76f078a.png)

**6.2、步骤解读**

下面我们着重叙述跟NIO相关的流程，共分为11个步骤，分别对应上面序列图中的步骤。

**步骤1**：绑定IP地址及端口，将ServerSocketChannel设置为阻塞。

这里为什么要设置成阻塞呢，我们一直都在说非阻塞。Tomcat的设计初衷主要是为了操作方便。这样这里就跟BIO模式下一样了。只不过在BIO下这里返回的是

Socket，NIO下这里返回的是SocketChannel。

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/ca01395590944a35b4717b15c984849e.png)

**步骤2**：启动接收线程

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/e7c40dc6f84b48f4915ca854c2a3b2cc.png)

**步骤3**：ServerSocketChannel.accept()接收新连接

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/50ddbfe042514ba29a8bcd606b125f76.jpeg)

**步骤4**：将接收到的链接通道设置为非阻塞

**步骤5**：构造NioChannel对象

**步骤6**：register注册到轮询线程

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/f3f5130bcf7a4348bc9752bd009c1b72.png)

**步骤7**：构造PollerEvent，并添加到事件队列

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/65de3c5cebea42c59ff19e8a168cf406.png)

**步骤8**：启动轮询线程

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/5c097feaa2324a2da082a81287f9e862.png)

**步骤9**：取出队列中新增的PollerEvent并注册到Selector

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/8ac5e80b9aa84f57b4c3f7ecd4e2ea1d.png)

**步骤10**：Selector.select()

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/82d20de2a4614eab9fe14e58234db552.jpeg)

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/75641a4b8d444f8ab7a033bee9497c2b.png)

**步骤11**：根据选择的SelectionKey构造SocketProcessor提交到请求处理线程

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/b8ec9bbcbdcd4088841741acb20152f8.png)

**6.3、NioBlockingSelector和BlockPoller介绍**

上面的序列图有个地方我没有描述，就是NioSelectorPool这个内部类，是因为在整体理解tomcat的nio上面在序列图里面不包括它更好理解。

在有了上面的基础后，我们在来说下NioSelectorPool这个类，对更深层了解Tomcat的NIO一定要知道它的作用。NioEndpoint对象中维护了一个NioSelecPool对象，这个NioSelectorPool中又维护了一个BlockPoller线程，这个线程就是基于辅Selector进行NIO的逻辑。

以执行servlet后，得到response，往socket中写数据为例，最终写的过程调用NioBlockingSelector的write方法。代码如下：

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/4ac6ac8fdb0a4b24bdcb646a09cbb13e.jpeg)

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/549283c8059b4e3c8798077b654dc3d1.png)

也就是说当socket.write()返回0时，说明网络状态不稳定，这时将socket注册OP_WRITE事件到辅Selector，由BlockPoller线程不断轮询这个辅Selector，直到发现这个socket的写状态恢复了，通过那个倒数计数器，通知Worker线程继续写socket动作。看一下BlockSelector线程的代码逻辑：

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/626817ca25fc4a439dbf5d23c2c08d0e.jpeg)

使用这个辅Selector主要是减少线程间的切换，同时还可减轻主Selector的负担。

## 七、关于性能

下面这份报告是我们压测的一个结果，跟想象的是不是不太一样？几乎没有差别，实际上NIO优化的是I/O的读写，如果瓶颈不在这里的话，比如传输字节数很小的情况下，BIO和NIO实际上是没有差别的。

NIO的优势更在于用少量的线程hold住大量的连接。还有一点，我们在压测的过程中，遇到在NIO模式下刚开始的一小段时间内容，会有错误，这是因为一般的压测工具是基于一种长连接，也就是说比如模拟1000并发，那么同时建立1000个连接，下一时刻再发送请求就是基于先前的这1000个连接来发送，还有TOMCAT的NIO处理是有POLLER线程来接管的，它的线程数一般等于CPU的核数，如果一瞬间有大量并发过来，POLLER也会顿时处理不过来。

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/7d62f8792e1c41f090d945c755bba6c7.jpeg)

压测1.jpeg

![](http://5b0988e595225.cdn.sohucs.com/images/20171112/e8ec450685d64e5785db44a0bb60660c.jpeg)

压测2.jpeg

## 八、总结

NIO只是优化了网络IO的读写，如果系统的瓶颈不在这里，比如每次读取的字节说都是500b，那么BIO和NIO在性能上没有区别。NIO模式是最大化压榨CPU，把时间片都更好利用起来。

对于操作系统来说，线程之间上下文切换的开销很大，而且每个线程都要占用系统的一些资源如内存，有关线程资源可参照这篇文章《一台java服务器可以跑多少个线程》。

因此，使用的线程越少越好。而I/O复用模型正是利用少量的线程来管理大量的连接。在对于维护大量长连接的应用里面更适合用基于I/O复用模型NIO，比如web qq这样的应用。所以我们要清楚系统的瓶颈是I/O还是CPU的计算
