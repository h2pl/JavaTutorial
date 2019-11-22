# Table of Contents

  * [前言](#前言)
  * [源码环境准备](#源码环境准备)
  * [endpoint](#endpoint)
  * [init 过程分析](#init-过程分析)
  * [start 过程分析](#start-过程分析)
  * [Acceptor](#acceptor)
  * [Poller](#poller)
  * [processKey](#processkey)
  * [总结](#总结)


本文转载 https://www.javadoop.com

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

## 前言

之前写了两篇关于 NIO 的文章，第一篇介绍了 NIO 的 Channel、Buffer、Selector 使用，第二篇介绍了非阻塞 IO 和异步 IO，并展示了简单的用例。

本文将介绍 Tomcat 中的 NIO 使用，使大家对 Java NIO 的生产使用有更加直观的认识。

虽然本文的源码篇幅也不短，但是 Tomcat 的源码毕竟不像 Doug Lea 的并发源码那么“变态”，对于大部分读者来说，阅读难度比之前介绍的其他并发源码要简单一些，所以读者不要觉得有什么压力。

本文基于 Tomcat 当前（2018-03-20）**最新版本 9.0.6**。

先简单画一张图示意一下本文的主要内容：

![0](https://www.javadoop.com/blogimages/tomcat-nio/0.png)

**目录**

## 源码环境准备

Tomcat 9.0.6 下载地址：[https://tomcat.apache.org/download-90.cgi](https://tomcat.apache.org/download-90.cgi)

由于上面下载的 tomcat 的源码并没有使用 maven 进行组织，不方便我们看源码，也不方便我们进行调试。这里我们将使用 maven 仓库中的 tomcat-embed-core，自己编写代码进行启动的方式来进行调试。

首先，创建一个空的 maven 工程，然后添加以下依赖。

```
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    tomcat-embed-core
    <version>9.0.6</version>
</dependency>
```

> 上面的依赖，只会将 tomcat-embed-core-9.0.6.jar 和 tomcat-annotations-api-9.0.6.jar 两个包引进来，对于本文来说，已经足够了，如果你需要其他功能，需要额外引用其他的依赖，如 Jasper。

然后，使用以下启动方法：

```
public static void main(String[] args) throws LifecycleException {

   Tomcat tomcat = new Tomcat();

   Connector connector = new Connector("HTTP/1.1");
   connector.setPort(8080);
   tomcat.setConnector(connector);

   tomcat.start();
   tomcat.getServer().await();
}
```

经过以上的代码，我们的 Tomcat 就启动起来了。

> Tomcat 中的其他接口感兴趣的读者请自行探索，如设置 webapp 目录，设置 resources 等

这里，介绍第一个重要的概念：**Connector**。在 Tomcat 中，使用 Connector 来处理连接，一个 Tomcat 可以配置多个 Connector，分别用于监听不同端口，或处理不同协议。

在 Connector 的构造方法中，我们可以传 `HTTP/1.1` 或 `AJP/1.3` 用于指定协议，也可以传入相应的协议处理类，毕竟协议不是重点，将不同端口进来的连接对应不同处理类才是正道。典型地，我们可以指定以下几个协议处理类：

*   org.apache.coyote.http11.Http11NioProtocol：对应非阻塞 IO
*   org.apache.coyote.http11.Http11Nio2Protocol：对应异步 IO
*   org.apache.coyote.http2.Http2Protocol：对应 http2 协议，对 http2 感兴趣的读者，赶紧看起来吧。

本文的重点当然是非阻塞 IO 了，之前已经介绍过`异步 IO`的基础知识了，读者看完本文后，如果对异步 IO 的处理流程感兴趣，可以自行去分析一遍。

> 如果你使用 9.0 以前的版本，Tomcat 在启动的时候是会自动配置一个 connector 的，我们可以不用显示配置。
> 
> 9.0 版本的 Tomcat#start() 方法：
> 
> ```
> public void start() throws LifecycleException {
>     getServer();
>     server.start();
> }
> ```
> 
> 8.5 及之前版本的 Tomcat#start() 方法：
> 
> ```
> public void start() throws LifecycleException {
>     getServer();
>     // 自动配置一个使用非阻塞 IO 的 connector
>     getConnector();
>     server.start();
> }
> ```

## endpoint

前面我们说过一个 Connector 对应一个协议，当然这描述也不太对，NIO 和 NIO2 就都是处理 HTTP/1.1 的，只不过一个使用非阻塞，一个使用异步。进到指定 protocol 代码，我们就会发现，它们的代码及其简单，只不过是指定了特定的 **endpoint**。

打开 `Http11NioProtocol` 和 `Http11Nio2Protocol`源码，我们可以看到，在构造方法中，它们分别指定了 NioEndpoint 和 Nio2Endpoint。

```
// 非阻塞模式
public class Http11NioProtocol extends AbstractHttp11JsseProtocol<NioChannel> {
    public Http11NioProtocol() {
        // NioEndpoint
        super(new NioEndpoint());
    }
    ...
}
// 异步模式
public class Http11Nio2Protocol extends AbstractHttp11JsseProtocol<Nio2Channel> {

    public Http11Nio2Protocol() {
        // Nio2Endpoint
        super(new Nio2Endpoint());
    }
    ...
}
```

这里介绍第二个重要的概念：**endpoint**。Tomcat 使用不同的 endpoint 来处理不同的协议请求，今天我们的重点是 **NioEndpoint**，其使用**非阻塞 IO** 来进行处理 HTTP/1.1 协议的请求。

**NioEndpoint** 继承 => **AbstractJsseEndpoint** 继承 => **AbstractEndpoint**。中间的 AbstractJsseEndpoint 主要是提供了一些关于 `HTTPS` 的方法，这块我们暂时忽略它，后面所有关于 HTTPS 的我们都直接忽略，感兴趣的读者请自行分析。

## init 过程分析

下面，我们看看从 tomcat.start() 一直到 NioEndpoint 的过程。

**1\. AbstractProtocol** # **init**

```
@Override
public void init() throws Exception {
    ...
    String endpointName = getName();
    endpoint.setName(endpointName.substring(1, endpointName.length()-1));
    endpoint.setDomain(domain);
    // endpoint 的 name=http-nio-8089,domain=Tomcat
    endpoint.init();
}
```

**2\. AbstractEndpoint** # **init**

```
public final void init() throws Exception {
    if (bindOnInit) {
        bind(); // 这里对应的当然是子类 NioEndpoint 的 bind() 方法
        bindState = BindState.BOUND_ON_INIT;
    }
    ...
}
```

**3\. NioEndpoint** # **bind**

这里就到我们的 NioEndpoint 了，要使用到我们之前学习的 NIO 的知识了。

```
@Override
public void bind() throws Exception {
    // initServerSocket(); 原代码是这行，我们 “内联” 过来一起说

    // 开启 ServerSocketChannel
    serverSock = ServerSocketChannel.open();
    socketProperties.setProperties(serverSock.socket());

    // getPort() 会返回我们最开始设置的 8080，得到我们的 address 是 0.0.0.0:8080
    InetSocketAddress addr = (getAddress()!=null?new InetSocketAddress(getAddress(),getPort()):new InetSocketAddress(getPort()));

    // ServerSocketChannel 绑定地址、端口，
    // 第二个参数 backlog 默认为 100，超过 100 的时候，新连接会被拒绝(不过源码注释也说了，这个值的真实语义取决于具体实现)
    serverSock.socket().bind(addr,getAcceptCount());

    // ※※※ 设置 ServerSocketChannel 为阻塞模式 ※※※
    serverSock.configureBlocking(true);

    // 设置 acceptor 和 poller 的数量，至于它们是什么角色，待会说
    // acceptorThreadCount 默认为 1
    if (acceptorThreadCount == 0) {
        // FIXME: Doesn't seem to work that well with multiple accept threads
        // 作者想表达的意思应该是：使用多个 acceptor 线程并不见得性能会更好
        acceptorThreadCount = 1;
    }

    // poller 线程数，默认值定义如下，所以在多核模式下，默认为 2
    // pollerThreadCount = Math.min(2,Runtime.getRuntime().availableProcessors());
    if (pollerThreadCount <= 0) {
        pollerThreadCount = 1;
    }

    // 
    setStopLatch(new CountDownLatch(pollerThreadCount));

    // 初始化 ssl，我们忽略 ssl
    initialiseSsl();

    // 打开 NioSelectorPool，先忽略它
    selectorPool.open();
}
```

1.  ServerSocketChannel 已经打开，并且绑定要了之前指定的 8080 端口，设置成了**阻塞模式**。
2.  设置了 acceptor 的线程数为 1
3.  设置了 poller 的线程数，单核 CPU 为 1，多核为 2
4.  打开了一个 SelectorPool，我们先忽略这个

到这里，我们还不知道 Acceptor 和 Poller 是什么东西，我们只是设置了它们的数量，我们先来看看最后面提到的 SelectorPool。

## start 过程分析

刚刚我们分析完了 init() 过程，下面是启动过程 start() 分析。

AbstractProtocol # start

```
@Override
public void start() throws Exception {
    ...
    // 调用 endpoint 的 start 方法
    endpoint.start();

    // Start async timeout thread
    asyncTimeout = new AsyncTimeout();
    Thread timeoutThread = new Thread(asyncTimeout, getNameInternal() + "-AsyncTimeout");
    int priority = endpoint.getThreadPriority();
    if (priority < Thread.MIN_PRIORITY || priority > Thread.MAX_PRIORITY) {
        priority = Thread.NORM_PRIORITY;
    }
    timeoutThread.setPriority(priority);
    timeoutThread.setDaemon(true);
    timeoutThread.start();
}
```

AbstractEndpoint # start

```
public final void start() throws Exception {
    // 按照我们的流程，刚刚 init 的时候，已经把 bindState 改为 BindState.BOUND_ON_INIT 了，
    // 所以下面的 if 分支我们就不进去了
    if (bindState == BindState.UNBOUND) {
        bind();
        bindState = BindState.BOUND_ON_START;
    }
    // 往里看 NioEndpoint 的实现
    startInternal();
}
```

下面这个方法还是比较重要的，这里会创建前面说过的 acceptor 和 poller。

NioEndpoint # startInternal

```
@Override
public void startInternal() throws Exception {

    if (!running) {
        running = true;
        paused = false;

        // 以下几个是缓存用的，之后我们也会看到很多这样的代码，为了减少 new 很多对象出来
        processorCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getProcessorCache());
        eventCache = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                        socketProperties.getEventCache());
        nioChannels = new SynchronizedStack<>(SynchronizedStack.DEFAULT_SIZE,
                socketProperties.getBufferPool());

        // 创建【工作线程池】，Tomcat 自己包装了一下 ThreadPoolExecutor，
        // 1\. 为了在创建线程池以后，先启动 corePoolSize 个线程(这个属于线程池的知识了，不熟悉的读者可以看看我之前的文章)
        // 2\. 自己管理线程池的增长方式（默认 corePoolSize 10, maxPoolSize 200），不是本文重点，不分析
        if ( getExecutor() == null ) {
            createExecutor();
        }

        // 设置一个栅栏（tomcat 自定义了类 LimitLatch），控制最大的连接数，默认是 10000
        initializeConnectionLatch();

        // 开启 poller 线程
        // 还记得之前 init 的时候，默认地设置了 poller 的数量为 2，所以这里启动 2 个 poller 线程
        pollers = new Poller[getPollerThreadCount()];
        for (int i=0; i<pollers.length; i++) {
            pollers[i] = new Poller();
            Thread pollerThread = new Thread(pollers[i], getName() + "-ClientPoller-"+i);
            pollerThread.setPriority(threadPriority);
            pollerThread.setDaemon(true);
            pollerThread.start();
        }

        // 开启 acceptor 线程，和开启 poller 线程组差不多。
        // init 的时候，默认地，acceptor 的线程数是 1
        startAcceptorThreads();
    }
}
```

到这里，我们启动了**工作线程池**、 **poller 线程组**、**acceptor 线程组**。同时，工作线程池初始就已经启动了 10 个线程。我们用 **jconsole** 来看看此时的线程，请看下图：

![1](https://www.javadoop.com/blogimages/tomcat-nio/1.png)

从 jconsole 中，我们可以看到，此时启动了 BlockPoller、worker、poller、acceptor、AsyncTimeout，大家应该都已经清楚了每个线程是哪里启动的吧。

> Tomcat 中并没有 Worker 这个类，此名字是我瞎编。

此时，我们还是不知道 acceptor、poller 甚至 worker 到底是干嘛的，下面，我们从 acceptor 线程开始看起。

## Acceptor

它的结构非常简单，在构造函数中，已经把 endpoint 传进来了，此外就只有 threadName 和 state 两个简单的属性。

```
private final AbstractEndpoint<?,U> endpoint;
private String threadName;
protected volatile AcceptorState state = AcceptorState.NEW;

public Acceptor(AbstractEndpoint<?,U> endpoint) {
    this.endpoint = endpoint;
}
```

**threadName** 就是一个线程名字而已，Acceptor 的状态 **state** 主要是随着 endpoint 来的。

```
public enum AcceptorState {
    NEW, RUNNING, PAUSED, ENDED
}
```

我们直接来看 acceptor 的 run 方法吧：

Acceptor # run

```
@Override
public void run() {

    int errorDelay = 0;

    // 只要 endpoint 处于 running，这里就一直循环
    while (endpoint.isRunning()) {

        // 如果 endpoint 处于 pause 状态，这边 Acceptor 用一个 while 循环将自己也挂起
        while (endpoint.isPaused() && endpoint.isRunning()) {
            state = AcceptorState.PAUSED;
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                // Ignore
            }
        }
        // endpoint 结束了，Acceptor 自然也要结束嘛
        if (!endpoint.isRunning()) {
            break;
        }
        state = AcceptorState.RUNNING;

        try {
            // 如果此时达到了最大连接数(之前我们说过，默认是10000)，就等待
            endpoint.countUpOrAwaitConnection();

            // Endpoint might have been paused while waiting for latch
            // If that is the case, don't accept new connections
            if (endpoint.isPaused()) {
                continue;
            }

            U socket = null;
            try {
                // 这里就是接收下一个进来的 SocketChannel
                // 之前我们设置了 ServerSocketChannel 为阻塞模式，所以这边的 accept 是阻塞的
                socket = endpoint.serverSocketAccept();
            } catch (Exception ioe) {
                // We didn't get a socket
                endpoint.countDownConnection();
                if (endpoint.isRunning()) {
                    // Introduce delay if necessary
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    // re-throw
                    throw ioe;
                } else {
                    break;
                }
            }
            // accept 成功，将 errorDelay 设置为 0
            errorDelay = 0;

            if (endpoint.isRunning() && !endpoint.isPaused()) {
                // setSocketOptions() 是这里的关键方法，也就是说前面千辛万苦都是为了能到这里进行处理
                if (!endpoint.setSocketOptions(socket)) {
                    // 如果上面的方法返回 false，关闭 SocketChannel
                    endpoint.closeSocket(socket);
                }
            } else {
                // 由于 endpoint 不 running 了，或者处于 pause 了，将此 SocketChannel 关闭
                endpoint.destroySocket(socket);
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            String msg = sm.getString("endpoint.accept.fail");
            // APR specific.
            // Could push this down but not sure it is worth the trouble.
            if (t instanceof Error) {
                Error e = (Error) t;
                if (e.getError() == 233) {
                    // Not an error on HP-UX so log as a warning
                    // so it can be filtered out on that platform
                    // See bug 50273
                    log.warn(msg, t);
                } else {
                    log.error(msg, t);
                }
            } else {
                    log.error(msg, t);
            }
        }
    }
    state = AcceptorState.ENDED;
}
```

大家应该发现了，Acceptor 绕来绕去，都是在调用 NioEndpoint 的方法，我们简单分析一下这个。

在 NioEndpoint init 的时候，我们开启了一个 ServerSocketChannel，后来 start 的时候，我们开启多个 acceptor（实际上，默认是 1 个），每个 acceptor 启动以后就开始循环调用 ServerSocketChannel 的 accept() 方法获取新的连接，然后调用 endpoint.setSocketOptions(socket) 处理新的连接，之后再进入循环 accept 下一个连接。

到这里，大家应该也就知道了，为什么这个叫 acceptor 了吧？接下来，我们来看看 setSocketOptions 方法到底做了什么。

NioEndpoint # setSocketOptions

```
@Override
protected boolean setSocketOptions(SocketChannel socket) {
    try {
        // 设置该 SocketChannel 为非阻塞模式
        socket.configureBlocking(false);
        Socket sock = socket.socket();
        // 设置 socket 的一些属性
        socketProperties.setProperties(sock);

        // 还记得 startInternal 的时候，说过了 nioChannels 是缓存用的。
        // 限于篇幅，这里的 NioChannel 就不展开了，它包括了 socket 和 buffer
        NioChannel channel = nioChannels.pop();
        if (channel == null) {
            // 主要是创建读和写的两个 buffer，默认地，读和写 buffer 都是 8192 字节，8k
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                    socketProperties.getAppReadBufSize(),
                    socketProperties.getAppWriteBufSize(),
                    socketProperties.getDirectBuffer());
            if (isSSLEnabled()) {
                channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
            } else {
                channel = new NioChannel(socket, bufhandler);
            }
        } else {
            channel.setIOChannel(socket);
            channel.reset();
        }

        // getPoller0() 会选取所有 poller 中的一个 poller
        getPoller0().register(channel);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        try {
            log.error("",t);
        } catch (Throwable tt) {
            ExceptionUtils.handleThrowable(tt);
        }
        // Tell to close the socket
        return false;
    }
    return true;
}
```

我们看到，这里又没有进行实际的处理，而是将这个 SocketChannel **注册**到了其中一个 poller 上。因为我们知道，acceptor 应该尽可能的简单，只做 accept 的工作，简单处理下就往后面扔。acceptor 还得回到之前的循环去 accept 新的连接呢。

我们只需要明白，此时，往 poller 中注册了一个 NioChannel 实例，此实例包含客户端过来的 SocketChannel 和一个 SocketBufferHandler 实例。

## Poller

之前我们看到 acceptor 将一个 NioChannel 实例 register 到了一个 poller 中。在看 register 方法之前，我们需要先对 poller 要有个简单的认识。

```
public class Poller implements Runnable {

    public Poller() throws IOException {
        // 每个 poller 开启一个 Selector
        this.selector = Selector.open();
    }
    private Selector selector;
    // events 队列，此类的核心
    private final SynchronizedQueue<PollerEvent> events =
            new SynchronizedQueue<>();

    private volatile boolean close = false;
    private long nextExpiration = 0;//optimize expiration handling

    // 这个值后面有用，记住它的初始值为 0
    private AtomicLong wakeupCounter = new AtomicLong(0);

    private volatile int keyCount = 0;

    ...
}
```

> 敲重点：每个 poller 关联了一个 Selector。

Poller 内部围着一个 events 队列转，来看看其 events() 方法：

```
public boolean events() {
    boolean result = false;

    PollerEvent pe = null;
    for (int i = 0, size = events.size(); i < size && (pe = events.poll()) != null; i++ ) {
        result = true;
        try {
            // 逐个执行 event.run()
            pe.run();
            // 该 PollerEvent 还得给以后用，这里 reset 一下(还是之前说过的缓存)
            pe.reset();
            if (running && !paused) {
                eventCache.push(pe);
            }
        } catch ( Throwable x ) {
            log.error("",x);
        }
    }
    return result;
}
```

events() 方法比较简单，就是取出当前队列中的 PollerEvent 对象，逐个执行 event.run() 方法。

然后，现在来看 Poller 的 run() 方法，该方法会一直循环，直到 poller.destroy() 被调用。

Poller # run

```
public void run() {
    while (true) {

        boolean hasEvents = false;

        try {
            if (!close) {
                // 执行 events 队列中每个 event 的 run() 方法
                hasEvents = events();
                // wakeupCounter 的初始值为 0，这里设置为 -1
                if (wakeupCounter.getAndSet(-1) > 0) {
                    //if we are here, means we have other stuff to do
                    //do a non blocking select
                    keyCount = selector.selectNow();
                } else {
                    // timeout 默认值 1 秒
                    keyCount = selector.select(selectorTimeout);
                }
                wakeupCounter.set(0);
            }
            // 篇幅所限，我们就不说 close 的情况了
            if (close) {
                events();
                timeout(0, false);
                try {
                    selector.close();
                } catch (IOException ioe) {
                    log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            }
        } catch (Throwable x) {
            ExceptionUtils.handleThrowable(x);
            log.error("",x);
            continue;
        }
        //either we timed out or we woke up, process events first
        // 这里没什么好说的，顶多就再执行一次 events() 方法
        if ( keyCount == 0 ) hasEvents = (hasEvents | events());

        // 如果刚刚 select 有返回 ready keys，进行处理
        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        // Walk through the collection of ready keys and dispatch
        // any active event.
        while (iterator != null && iterator.hasNext()) {
            SelectionKey sk = iterator.next();
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            // Attachment may be null if another thread has called
            // cancelledKey()
            if (attachment == null) {
                iterator.remove();
            } else {
                iterator.remove();
                // ※※※※※ 处理 ready key ※※※※※
                processKey(sk, attachment);
            }
        }//while

        //process timeouts
        timeout(keyCount,hasEvents);
    }//while

    getStopLatch().countDown();
}
```

poller 的 run() 方法主要做了调用 events() 方法和处理注册到 Selector 上的 ready key，这里我们暂时不展开 processKey 方法，因为此方法必定是及其复杂的。

我们回过头来看之前从 acceptor 线程中调用的 register 方法。

Poller # register

```
public void register(final NioChannel socket) {
    socket.setPoller(this);
    NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
    socket.setSocketWrapper(ka);
    ka.setPoller(this);
    ka.setReadTimeout(getConnectionTimeout());
    ka.setWriteTimeout(getConnectionTimeout());
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    ka.setSecure(isSSLEnabled());

    PollerEvent r = eventCache.pop();
    ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.

    // 注意第三个参数值 OP_REGISTER
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    else r.reset(socket,ka,OP_REGISTER);

    // 添加 event 到 poller 中
    addEvent(r);
}
```

这里将这个 socket（包含 socket 和 buffer 的 NioChannel 实例） 包装为一个 PollerEvent，然后添加到 events 中，此时调用此方法的 acceptor 结束返回，去处理新的 accepted 连接了。

接下来，我们已经知道了，poller 线程在循环过程中会不断调用 events() 方法，那么 PollerEvent 的 run() 方法很快就会被执行，我们就来看看刚刚这个新的连接被**注册**到这个 poller 后，会发生什么。

PollerEvent # run

```
@Override
public void run() {
    // 对于新来的连接，前面我们说过，interestOps == OP_REGISTER
    if (interestOps == OP_REGISTER) {
        try {
            // 这步很关键！！！
            // 将这个新连接 SocketChannel 注册到该 poller 的 Selector 中，
            // 设置监听 OP_READ 事件，
            // 将 socketWrapper 设置为 attachment 进行传递(这个对象可是什么鬼都有，往上看就知道了)
            socket.getIOChannel().register(
                    socket.getPoller().getSelector(), SelectionKey.OP_READ, socketWrapper);
        } catch (Exception x) {
            log.error(sm.getString("endpoint.nio.registerFail"), x);
        }
    } else {
        /* else 这块不介绍，省得大家头大 */

        final SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
        try {
            if (key == null) {
                // The key was cancelled (e.g. due to socket closure)
                // and removed from the selector while it was being
                // processed. Count down the connections at this point
                // since it won't have been counted down when the socket
                // closed.
                socket.socketWrapper.getEndpoint().countDownConnection();
            } else {
                final NioSocketWrapper socketWrapper = (NioSocketWrapper) key.attachment();
                if (socketWrapper != null) {
                    //we are registering the key to start with, reset the fairness counter.
                    int ops = key.interestOps() | interestOps;
                    socketWrapper.interestOps(ops);
                    key.interestOps(ops);
                } else {
                    socket.getPoller().cancelledKey(key);
                }
            }
        } catch (CancelledKeyException ckx) {
            try {
                socket.getPoller().cancelledKey(key);
            } catch (Exception ignore) {}
        }
    }
}
```

到这里，我们再回顾一下：刚刚在 PollerEvent 的 run() 方法中，我们看到，新的 SocketChannel 注册到了 Poller 内部的 Selector 中，监听 OP_READ 事件，然后我们再回到 Poller 的 run() 看下，一旦该 SocketChannel 是 readable 的状态，那么就会进入到 poller 的 processKey 方法。

## processKey

Poller # processKey

```
protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
    try {
        if ( close ) {
            cancelledKey(sk);
        } else if ( sk.isValid() && attachment != null ) {
            if (sk.isReadable() || sk.isWritable() ) {
                // 忽略 sendfile
                if ( attachment.getSendfileData() != null ) {
                    processSendfile(sk,attachment, false);
                } else {
                    // unregister 相应的 interest set，
                    // 如接下来是处理 SocketChannel 进来的数据，那么就不再监听该 channel 的 OP_READ 事件
                    unreg(sk, attachment, sk.readyOps());
                    boolean closeSocket = false;
                    // Read goes before write
                    if (sk.isReadable()) {
                        // 处理读
                        if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                            closeSocket = true;
                        }
                    }
                    if (!closeSocket && sk.isWritable()) {
                        // 处理写
                        if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                            closeSocket = true;
                        }
                    }
                    if (closeSocket) {
                        cancelledKey(sk);
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error("",t);
    }
}
```

接下来是 processSocket 方法，注意第三个参数，上面进来的时候是 true。

AbstractEndpoint # processSocket

```
public boolean processSocket(SocketWrapperBase<S> socketWrapper,
        SocketEvent event, boolean dispatch) {
    try {
        if (socketWrapper == null) {
            return false;
        }
        SocketProcessorBase<S> sc = processorCache.pop();
        if (sc == null) {
            // 创建一个 SocketProcessor 的实例
            sc = createSocketProcessor(socketWrapper, event);
        } else {
            sc.reset(socketWrapper, event);
        }
        Executor executor = getExecutor();
        if (dispatch && executor != null) {
            // 将任务放到之前建立的 worker 线程池中执行
            executor.execute(sc);
        } else {
            sc.run(); // ps: 如果 dispatch 为 false，那么就当前线程自己执行
        }
    } catch (RejectedExecutionException ree) {
        getLog().warn(sm.getString("endpoint.executor.fail", socketWrapper) , ree);
        return false;
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        // This means we got an OOM or similar creating a thread, or that
        // the pool and its queue are full
        getLog().error(sm.getString("endpoint.process.fail"), t);
        return false;
    }
    return true;
}
```

NioEndpoint # createSocketProcessor

```
@Override
protected SocketProcessorBase<NioChannel> createSocketProcessor(
        SocketWrapperBase<NioChannel> socketWrapper, SocketEvent event) {
    return new SocketProcessor(socketWrapper, event);
}
```

我们看到，提交到 worker 线程池中的是 NioEndpoint.SocketProcessor 的实例，至于它的 run() 方法之后的逻辑，我们就不再继续往里分析了。

## 总结

最后，再祭出文章开始的那张图来总结一下：

![0](https://www.javadoop.com/blogimages/tomcat-nio/0.png)

这里简单梳理下前面我们说的流程，帮大家回忆一下：

1.  指定 Protocol，初始化相应的 Endpoint，我们分析的是 NioEndpoint；
2.  init 过程：在 NioEndpoint 中做 bind 操作；
3.  start 过程：启动 worker 线程池，启动 1 个 Acceptor 和 2 个 Poller，当然它们都是默认值，可配；
4.  Acceptor 获取到新的连接后，getPoller0() 获取其中一个 Poller，然后 register 到 Poller 中；
5.  Poller 循环 selector.select(xxx)，如果有通道 readable，那么在 processKey 中将其放到 worker 线程池中。

后续的流程，感兴趣的读者请自行分析，本文就说到这里了。

（全文完）
