# Table of Contents

  * [Java IO](#java-io)
  * [Socket编程](#socket编程)
  * [客户端，服务端的线程模型](#客户端，服务端的线程模型)
  * [IO模型](#io模型)
  * [NIO](#nio)
  * [AIO](#aio)
  * [Tomcat中的NIO模型](#tomcat中的nio模型)
  * [Tomcat的container](#tomcat的container)
  * [netty](#netty)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


---
title: Java网络编程与NIO学习总结
date: 2018-07-08 22:08:22
tags:
	- Java网络编程
	- NIO
categories:
	- 后端
	- 技术总结
---
这篇总结主要是基于我之前Java网络编程与NIO系列文章而形成的的。主要是把重要的知识点用自己的话说了一遍，可能会有一些错误，还望见谅和指点。谢谢

更多详细内容可以查看我的专栏文章：Java网络编程与NIO

https://blog.csdn.net/column/details/21963.html
<!-- more -->

## Java IO

Java IO的基础知识已在前面讲过

## Socket编程

socket是操作系统提供的网络编程接口，他封装了对于TCP/IP协议栈的支持，用于进程间的通信，当有连接接入主机以后，操作系统自动为其分配一个socket套接字，套接字绑定着一个IP与端口号。通过socket接口，可以获取tcp连接的输入流和输出流，并且通过他们进行读取和写入此操作。

Java提供了net包用于socket编程，同时支持像Inetaddress，URL等工具类，使用socket绑定一个endpoint（ip+端口号），可以用于客户端的请求处理和发送，使用serversocket绑定本地ip和端口号，可以用于服务端接收TCP请求。

## 客户端，服务端的线程模型

一般客户端使用单线程模型即可，当有数据到来时启动线程读取，需要写入数据时开启线程进行数据写入。

服务端一般使用多线程模型，一个线程负责接收tcp连接请求，每当接收到请求后开启一个线程处理它的读写请求。

udp的客户端和服务端就比较简单了，由于udp数据报长度是确定的，只需要写入一个固定的缓存和读取一个固定的缓存空间即可。

一般通过DatagramPacket包装一个udp数据报，然后通过DatagramSocket发送

## IO模型

上述的socket在处理IO请求时使用的是阻塞模型。

于是我们还是得来探讨一下IO模型。

一般认为，应用程序处理IO请求需要将内核缓存区中的数据拷贝到用户缓冲区。这个步骤可以通过系统调用来完成，而用户程序处理IO请求的时候，需要先检查用户缓冲区是否准备好了数据，这个操作是系统调用recevfrom，如果数据没有准备好，默认会阻塞调用该方法的线程。

这样就导致了线程处理IO请求需要频繁进行阻塞，特别是并发量大的时候，线程切换的开销巨大。

一般认为有几种IO模型

1 阻塞IO ：就是线程会阻塞在系统调用recevfrom上，并且等待数据准备就绪以后才会返回。

2 非阻塞IO : 不阻塞在系统调用recevfrom，而是通过自旋忙等的方式不断询问缓冲区数据是否准备就绪，避免线程阻塞的开销。

3 IO多路复用 ：使用IO多路复用器管理socket，由于每个socket是一个文件描述符，操作系统可以维护socket和它的连接状态，一般分为可连接，可读和可写等状态。

每当用户程序接受到socket请求，将请求托管给多路复用器进行监控，当程序对请求感兴趣的事件发生时，多路复用器以某种方式通知或是用户程序自己轮询请求，以便获取就绪的socket，然后只需使用一个线程进行轮询，多个线程处理就绪请求即可。

IO多路复用避免了每个socket请求都需要一个线程去处理，而是使用事件驱动的方式，让少数的线程去处理多数socket的IO请求。

Linux操作系统对IO多路复用提供了较好的支持，select，poll，epoll是Linux提供的支持IO多路复用的API。一般用户程序基于这个API去开发自己的IO复用模型。比如NIO的非阻塞模型，就是采用了IO多路复用的方式，是基于epoll实现的。

3.1 select方式主要是使用数组来存储socket描述符，系统将发生事件的描述符做标记，然后IO复用器在轮询描述符数组的时候，就可以知道哪些请求是就绪了的。缺点是数组的长度只能到1024，并且需要不断地在内核空间和用户空间之间拷贝数组。

3.2 poll方式不采用数组存储描述符，而是使用独立的数据结构来描述，并且使用id来表示描述符，能支持更多的请求数量，缺点和select方式有点类似，就是轮询的效率很低，并且需要拷贝数据。

当然，上述两种方法适合在请求总数较少，并且活跃请求数较多的情况，这种场景下他们的性能还是不错的。

3.3 epoll

epoll函数会在内核空间开辟一个特殊的数据结构，红黑树，树节点中存放的是一个socket描述符以及用户程序感兴趣的事件类型。同时epoll还会维护一个链表。用于存储已经就绪的socket描述符节点。

由Linux内核完成对红黑树的维护，当事件到达时，内核将就绪的socket节点加入链表中，用户程序可以直接访问这个链表以便获取就绪的socket。

当然了，这些操作都linux包装在epoll的api中了。

epoll_create函数会执行红黑树的创建操作。

epoll_ctl函数会将socket和感兴趣的事件注册到红黑树中。

epoll_wait函数会等待内核空间发来的链表，从而执行IO请求。

epoll的水平触发和边缘触发有所区别，水平触发的意思是，如果用户程序没有执行就绪链表里的任务，epoll仍会不断通知程序。

而边缘触发只会通知程序一次，之后socket的状态不发生改变epoll就不会再通知程序了。

4 信号驱动
略

5 异步非阻塞

用户进程发起read操作之后，立刻就可以开始去做其它的事。而另一方面，从kernel的角度，当它受到一个asynchronous read之后，首先它会立刻返回，所以不会对用户进程产生任何block。然后，kernel会等待数据准备完成，然后将数据拷贝到用户内存，当这一切都完成之后，kernel会给用户进程发送一个signal，告诉它read操作完成了。

事实上就是，用户提交IO请求，然后直接返回，并且内核自动完成将数据从内核缓冲区复制到用户缓冲区，完成后再通知用户。

当然，内核通知我们以后我们还需要执行剩余的操作，但是我们的代码已经继续往下运行了，所以AIO采用了回调的机制，为每个socket注册一个回调事件或者是回调处理器，在处理器中完成数据的操作，也就是内核通知到用户的时候，会自动触发回调函数，完成剩余操作。
这样的方式就是异步的网络编程。

但是，想要让操作系统支持这样的功能并非易事，windows的IOCP可以支持AIO方式，但是Linux的AIO支持并不是很好
## NIO

由于Java原生的socket只支持阻塞方式处理IO

所以Java后来推出了新版IO 也叫New IO = NIO

NIO提出了socketChannel，serversocketchannel，bytebuffer，selector和selectedkey等概念。

1 socketchannel其实就是socket的替代品，他的好处是多个socket可以复用同一个bytebuffer，因为socket是从channel里打开的，所以多个socket都可以访问channel绑定着的buffer。

2 serversocketchannel顾名思义，是用在服务端的channel。

3 bytebuffer以前对用户是透明的，用户直接操作io流即可，所以之前的socket io操作都是阻塞的，引入bytebuffer以后，用户可以更灵活地进行io操作。

buffer可以分为不同数据类型的buffer，但是常用的还是bytebuffer。写入数据时按顺序写入，写入完使用flip方法反转缓冲区，让接收端反向读取。这个操作比较麻烦，后来的netty对缓冲区进行了重新封装，封装了这个经常容易出错的方法。

4 selector其实就是对io多路复用器的封装，一般基于linux的epoll来实现。
socket把感兴趣的事件和描述符注册到selector上，然后通过遍历selectedKey来获取感兴趣的请求，进行IO操作。
selectedkey应该就是epoll中就绪链表的实现了。

5 所以一般的流程是：
新建一个serversocket，启动一个线程进行while循环，当有请求接入时，使用accept方法阻塞获取socket，然后将socket和感兴趣的事件注册到selector上。再开启一个线程轮询selectoredKey，当请求就绪时开启一个线程去处理即可。

## AIO

后来NIO发展到2.0，Java又推出了AIO 的API，与上面描述的异步非阻塞模型类似。

AIO使用回调的方式处理IO请求，在socket上注册一个回调函数，然后提交请求后直接返回。由操作系统完成数据拷贝操作，需要操作系统对AIO的支持。

AIO的具体使用方式还是比较复杂的，感兴趣的可以自己查阅资料。

## Tomcat中的NIO模型

Tomcat作为一个应用服务器，分为connector和container两个部分，connector负责接收请求，而container负责解析请求。

一般connector负责接收http请求，当然首先要建立tcp连接，所以涉及到了如何处理连接和IO请求。

Tomcat使用endpoint的概念来绑定一个ip+port，首先，使用acceptor循环等待连接请求。然后开启一个线程池，也叫poller池，每个请求绑定一个poller进行后续处理，poller将socket请求封装成一个事件，并且将这个事件注册到selector中。

poller还需要维护一个事件列表，以便获取selector上就绪的事件。然后poller再去列表中获取就绪的请求，将其封装成processor，交给后续的worker线程池，会有worker将其提交给container流程中进行处理。

当然，到达container之后还有非常复杂的处理过程，稍微提几个点。

## Tomcat的container

container是一个多级容器，最外层到最内层依次是engine，host，context和wrapper

下面是个server.xml文件实例，Tomcat根据该文件进行部署

<Server>                                                //顶层类元素，可以包括多个Service   
    <Service>                                           //顶层类元素，可包含一个Engine，多个Connecter
        <Connector>                                     //连接器类元素，代表通信接口
                <Engine>                                //容器类元素，为特定的Service组件处理客户请求，要包含多个Host
                        <Host>                          //容器类元素，为特定的虚拟主机组件处理客户请求，可包含多个Context
                                <Context>               //容器类元素，为特定的Web应用处理所有的客户请求
                                </Context>
                        </Host>
                </Engine>
        </Connector>
    </Service>
</Server>


根据配置文件初始化容器信息，当请求到达时进行容器间的请求传递，事实上整个链条被称作pipeline，pipeline连接了各个容器的入口，由于每个容器和组件都实现了lifecycle接口。

tomcat可以在任意流程中通过加监听器的方式监听组件的生命周期，也就能够控制整个运行的流程，通过在pipeline上增加valve可以增加一些自定义的操作。

一般到wrapper层才开始真正的请求解析，因为wrapper其实就是对servlet的简单封装，此时进来的请求和响应已经是httprequest和httpresponse，很多信息已经解析完毕，只需要按照service方法执行业务逻辑即可，当然在执行service方法之前，会调用filter链先执行过滤操作。

## netty

netty我也不是很在行，这里简单总结一下

netty是一个基于事件驱动的网络编程框架。

因为直接基于Java NIO编程复杂度太高，而且容易出错，于是netty对NIO进行了改造和封装。形成了一个比较完整的网络框架，可以通过他实现rpc，http服务。

先了解一下两种线程模型。reactor和proactor。

1 reactor就是netty采用的模型，首先也是使用一个acceptor线程接收连接请求，然后开启一个线程组reactor thread pool。

server会事先在endpoint上注册一系列的回调方法，然后接收socket请求后交给底层的selector进行管理，当selector对应的事件响应以后，会通知用户进程，然后reactor工作线程会执行接下来的IO请求，执行操作是写在回调处理器中的。


其实netty 支持三种reactor模型
1.1.Reactor单线程模型：Reactor单线程模型，指的是所有的I/O操作都在同一个NIO线程上面完成。对于一些小容量应用场景，可以使用单线程模型。

1.2.Reactor多线程模型：Rector多线程模型与单线程模型最大的区别就是有一组NIO线程处理I/O操作。主要用于高并发、大业务量场景。

1.3.主从Reactor多线程模型：主从Reactor线程模型的特点是服务端用于接收客户端连接的不再是个1个单独的NIO线程，而是一个独立的NIO线程池。利用主从NIO线程模型，可以解决1个服务端监听线程无法有效处理所有客户端连接的性能不足问题

2 proactor模型其实是基于异步非阻塞IO模型的，当accpetor接收到请求以后，直接提交异步的io请求给linux内核，内核完成io请求后会回写消息到proactor提供的事件队列中，此时工作线程查看到IO请求已完成，则会继续剩余的工作，也是通过回调处理器来进行的。

所以两者最大的差别是，前者基于epoll的IO多路复用，后者基于AIO实现。

3 netty的核心组件：

bytebuf

bytebuf是对NIO中Bytebuffer的优化和扩展，并且支持堆外内存分配，堆外内存避免gc，可以更好地与内核空间进行交换数据。

channel和NIO的channel类似，但是NIO的socket代码改成nio实现非常麻烦，所以netty优化了这个过程，只需替换几个类就可以实现不更新太多代码就完成旧IO和新IO的切换。

channelhandler就是任务的处理器了，使用回调函数的方式注册到channel中，更准确来说是注册到channelpipeline里。

channelpipeline是用来管理和连接多个channelhandler的容器，执行任务时，会根据channelpipeline的调用链完成处理器的顺序调用，启动服务器时只需要将需要的channelhandler注册在上面就可以了。
    
eventloop
在Netty的线程模型中，一个EventLoop将由一个永远不会改变的Thread驱动，而一个Channel一生只会使用一个EventLoop（但是一个EventLoop可能会被指派用于服务多个Channel），在Channel中的所有I/O操作和事件都由EventLoop中的线程处理，也就是说一个Channel的一生之中都只会使用到一个线程。


bootstrap

在深入了解地Netty的核心组件之后，发现它们的设计都很模块化，如果想要实现你自己的应用程序，就需要将这些组件组装到一起。Netty通过Bootstrap类，以对一个Netty应用程序进行配置（组装各个组件），并最终使它运行起来。

对于客户端程序和服务器程序所使用到的Bootstrap类是不同的，后者需要使用ServerBootstrap，这样设计是因为，在如TCP这样有连接的协议中，服务器程序往往需要一个以上的Channel，通过父Channel来接受来自客户端的连接，然后创建子Channel用于它们之间的通信，而像UDP这样无连接的协议，它不需要每个连接都创建子Channel，只需要一个Channel即可。

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
