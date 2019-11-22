# Table of Contents

  * [概述](#概述)
  * [Selector的中的重要属性](#selector的中的重要属性)
  * [Selector 源码解析](#selector-源码解析)
    * [1、Selector的构建](#1、selector的构建)
          * [接下来看下 selector.open()：](#接下来看下-selectoropen：)
    * [EPollSelectorImpl](#epollselectorimpl)
    * [EPollArrayWrapper](#epollarraywrapper)
    * [ServerSocketChannel的构建](#serversocketchannel的构建)
    * [将ServerSocketChannel注册到Selector](#将serversocketchannel注册到selector)
    * [EPollSelectorImpl. implRegister](#epollselectorimpl-implregister)
    * [Selection操作](#selection操作)
    * [epoll原理](#epoll原理)
  * [后记](#后记)
  * [参考文章](#参考文章)


本文转自互联网

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

## 概述

Selector是NIO中实现I/O多路复用的关键类。Selector实现了通过一个线程管理多个Channel，从而管理多个网络连接的目的。

Channel代表这一个网络连接通道，我们可以将Channel注册到Selector中以实现Selector对其的管理。一个Channel可以注册到多个不同的Selector中。

当Channel注册到Selector后会返回一个SelectionKey对象，该SelectionKey对象则代表这这个Channel和它注册的Selector间的关系。并且SelectionKey中维护着两个很重要的属性：interestOps、readyOps
interestOps是我们希望Selector监听Channel的哪些事件。

我们将我们感兴趣的事件设置到该字段，这样在selection操作时，当发现该Channel有我们所感兴趣的事件发生时，就会将我们感兴趣的事件再设置到readyOps中，这样我们就能得知是哪些事件发生了以做相应处理。

## Selector的中的重要属性

Selector中维护3个特别重要的SelectionKey集合，分别是

*   keys：所有注册到Selector的Channel所表示的SelectionKey都会存在于该集合中。keys元素的添加会在Channel注册到Selector时发生。
*   selectedKeys：该集合中的每个SelectionKey都是其对应的Channel在上一次操作selection期间被检查到至少有一种SelectionKey中所感兴趣的操作已经准备好被处理。该集合是keys的一个子集。
*   cancelledKeys：执行了取消操作的SelectionKey会被放入到该集合中。该集合是keys的一个子集。

下面的源码解析会说明上面3个集合的用处

## Selector 源码解析

下面我们通过一段对Selector的使用流程讲解来进一步深入其实现原理。
首先先来段Selector最简单的使用片段

```
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        int port = 5566;          
        serverChannel.socket().bind(new InetSocketAddress(port));
        Selector selector = Selector.open();
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        while(true){
            int n = selector.select();
            if(n > 0) {
                Iterator<SelectionKey> iter = selector.selectedKeys().iterator();
                while (iter.hasNext()) {
                    SelectionKey selectionKey = iter.next();
                    ......
                    iter.remove();
                }
            }
        }

```

### 1、Selector的构建

SocketChannel、ServerSocketChannel和Selector的实例初始化都通过SelectorProvider类实现。

ServerSocketChannel.open();

```
    public static ServerSocketChannel open() throws IOException {
        return SelectorProvider.provider().openServerSocketChannel();
    }

```

SocketChannel.open();

```
    public static SocketChannel open() throws IOException {
        return SelectorProvider.provider().openSocketChannel();
    }

```

Selector.open();

```
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

```

我们来进一步的了解下SelectorProvider.provider()

```
    public static SelectorProvider provider() {
        synchronized (lock) {
            if (provider != null)
                return provider;
            return AccessController.doPrivileged(
                new PrivilegedAction<>() {
                    public SelectorProvider run() {
                            if (loadProviderFromProperty())
                                return provider;
                            if (loadProviderAsService())
                                return provider;
                            provider = sun.nio.ch.DefaultSelectorProvider.create();
                            return provider;
                        }
                    });
        }
    }

```

① 如果配置了“java.nio.channels.spi.SelectorProvider”属性，则通过该属性值load对应的SelectorProvider对象，如果构建失败则抛异常。
② 如果provider类已经安装在了对系统类加载程序可见的jar包中，并且该jar包的源码目录META-INF/services包含有一个java.nio.channels.spi.SelectorProvider提供类配置文件，则取文件中第一个类名进行load以构建对应的SelectorProvider对象，如果构建失败则抛异常。
③ 如果上面两种情况都不存在，则返回系统默认的SelectorProvider，即，sun.nio.ch.DefaultSelectorProvider.create();
④ 随后在调用该方法，即SelectorProvider.provider()。则返回第一次调用的结果。

不同系统对应着不同的sun.nio.ch.DefaultSelectorProvider





![](https://upload-images.jianshu.io/upload_images/4235178-a02c498e08979aff.png?imageMogr2/auto-orient/strip|imageView2/2/w/640/format/webp)





这里我们看linux下面的sun.nio.ch.DefaultSelectorProvider

```
public class DefaultSelectorProvider {

    /**
     * Prevent instantiation.
     */
    private DefaultSelectorProvider() { }

    /**
     * Returns the default SelectorProvider.
     */
    public static SelectorProvider create() {
        return new sun.nio.ch.EPollSelectorProvider();
    }

}

```

可以看见，linux系统下sun.nio.ch.DefaultSelectorProvider.create(); 会生成一个sun.nio.ch.EPollSelectorProvider类型的SelectorProvider，这里对应于linux系统的epoll

###### 接下来看下 selector.open()：

```
    /**
     * Opens a selector.
     *
     * <p> The new selector is created by invoking the {@link
     * java.nio.channels.spi.SelectorProvider#openSelector openSelector} method
     * of the system-wide default {@link
     * java.nio.channels.spi.SelectorProvider} object.  </p>
     *
     * @return  A new selector
     *
     * @throws  IOException
     *          If an I/O error occurs
     */
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

```

在得到sun.nio.ch.EPollSelectorProvider后调用openSelector()方法构建Selector，这里会构建一个EPollSelectorImpl对象。

### EPollSelectorImpl

```
class EPollSelectorImpl
    extends SelectorImpl
{

    // File descriptors used for interrupt
    protected int fd0;
    protected int fd1;

    // The poll object
    EPollArrayWrapper pollWrapper;

    // Maps from file descriptors to keys
    private Map<Integer,SelectionKeyImpl> fdToKey;

```

```
EPollSelectorImpl(SelectorProvider sp) throws IOException {
        super(sp);
        long pipeFds = IOUtil.makePipe(false);
        fd0 = (int) (pipeFds >>> 32);
        fd1 = (int) pipeFds;
        try {
            pollWrapper = new EPollArrayWrapper();
            pollWrapper.initInterrupt(fd0, fd1);
            fdToKey = new HashMap<>();
        } catch (Throwable t) {
            try {
                FileDispatcherImpl.closeIntFD(fd0);
            } catch (IOException ioe0) {
                t.addSuppressed(ioe0);
            }
            try {
                FileDispatcherImpl.closeIntFD(fd1);
            } catch (IOException ioe1) {
                t.addSuppressed(ioe1);
            }
            throw t;
        }
    }

```

EPollSelectorImpl构造函数完成：
① EPollArrayWrapper的构建，EpollArrayWapper将Linux的epoll相关系统调用封装成了native方法供EpollSelectorImpl使用。
② 通过EPollArrayWrapper向epoll注册中断事件

```
    void initInterrupt(int fd0, int fd1) {
        outgoingInterruptFD = fd1;
        incomingInterruptFD = fd0;
        epollCtl(epfd, EPOLL_CTL_ADD, fd0, EPOLLIN);
    }

```

③ fdToKey：构建文件描述符-SelectionKeyImpl映射表，所有注册到selector的channel对应的SelectionKey和与之对应的文件描述符都会放入到该映射表中。

### EPollArrayWrapper

EPollArrayWrapper完成了对epoll文件描述符的构建，以及对linux系统的epoll指令操纵的封装。维护每次selection操作的结果，即epoll_wait结果的epoll_event数组。
EPollArrayWrapper操纵了一个linux系统下epoll_event结构的本地数组。

```
* typedef union epoll_data {
*     void *ptr;
*     int fd;
*     __uint32_t u32;
*     __uint64_t u64;
*  } epoll_data_t;
*
* struct epoll_event {
*     __uint32_t events;
*     epoll_data_t data;
* };

```

epoll_event的数据成员(epoll_data_t data)包含有与通过epoll_ctl将文件描述符注册到epoll时设置的数据相同的数据。这里data.fd为我们注册的文件描述符。这样我们在处理事件的时候持有有效的文件描述符了。

EPollArrayWrapper将Linux的epoll相关系统调用封装成了native方法供EpollSelectorImpl使用。

```
    private native int epollCreate();
    private native void epollCtl(int epfd, int opcode, int fd, int events);
    private native int epollWait(long pollAddress, int numfds, long timeout,
                                 int epfd) throws IOException;

```

上述三个native方法就对应Linux下epoll相关的三个系统调用

```
    // The fd of the epoll driver
    private final int epfd;

     // The epoll_event array for results from epoll_wait
    private final AllocatedNativeObject pollArray;

    // Base address of the epoll_event array
    private final long pollArrayAddress;

```

```
    // 用于存储已经注册的文件描述符和其注册等待改变的事件的关联关系。在epoll_wait操作就是要检测这里文件描述法注册的事件是否有发生。
    private final byte[] eventsLow = new byte[MAX_UPDATE_ARRAY_SIZE];
    private final Map<Integer,Byte> eventsHigh = new HashMap<>();

```

```
    EPollArrayWrapper() throws IOException {
        // creates the epoll file descriptor
        epfd = epollCreate();

        // the epoll_event array passed to epoll_wait
        int allocationSize = NUM_EPOLLEVENTS * SIZE_EPOLLEVENT;
        pollArray = new AllocatedNativeObject(allocationSize, true);
        pollArrayAddress = pollArray.address();
    }

```

EPoolArrayWrapper构造函数，创建了epoll文件描述符。构建了一个用于存放epoll_wait返回结果的epoll_event数组。

### ServerSocketChannel的构建

ServerSocketChannel.open();

返回ServerSocketChannelImpl对象，构建linux系统下ServerSocket的文件描述符。

```
    // Our file descriptor
    private final FileDescriptor fd;

    // fd value needed for dev/poll. This value will remain valid
    // even after the value in the file descriptor object has been set to -1
    private int fdVal;

```

```
    ServerSocketChannelImpl(SelectorProvider sp) throws IOException {
        super(sp);
        this.fd =  Net.serverSocket(true);
        this.fdVal = IOUtil.fdVal(fd);
        this.state = ST_INUSE;
    }

```

### 将ServerSocketChannel注册到Selector

serverChannel.register(selector, SelectionKey.OP_ACCEPT);

```
    public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            SelectionKey k = findKey(sel);
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
            }
            return k;
        }
    }

```

```
    protected final SelectionKey register(AbstractSelectableChannel ch,
                                          int ops,
                                          Object attachment)
    {
        if (!(ch instanceof SelChImpl))
            throw new IllegalSelectorException();
        SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
        k.attach(attachment);
        synchronized (publicKeys) {
            implRegister(k);
        }
        k.interestOps(ops);
        return k;
    }

```

① 构建代表channel和selector间关系的SelectionKey对象
② implRegister(k)将channel注册到epoll中
③ k.interestOps(int) 完成下面两个操作：
a) 会将注册的感兴趣的事件和其对应的文件描述存储到EPollArrayWrapper对象的eventsLow或eventsHigh中，这是给底层实现epoll_wait时使用的。
b) 同时该操作还会将设置SelectionKey的interestOps字段，这是给我们程序员获取使用的。

### EPollSelectorImpl. implRegister

```
    protected void implRegister(SelectionKeyImpl ski) {
        if (closed)
            throw new ClosedSelectorException();
        SelChImpl ch = ski.channel;
        int fd = Integer.valueOf(ch.getFDVal());
        fdToKey.put(fd, ski);
        pollWrapper.add(fd);
        keys.add(ski);
    }

```

① 将channel对应的fd(文件描述符)和对应的SelectionKeyImpl放到fdToKey映射表中。
② 将channel对应的fd(文件描述符)添加到EPollArrayWrapper中，并强制初始化fd的事件为0 ( 强制初始更新事件为0，因为该事件可能存在于之前被取消过的注册中。)
③ 将selectionKey放入到keys集合中。

### Selection操作

selection操作有3中类型：
① select()：该方法会一直阻塞直到至少一个channel被选择(即，该channel注册的事件发生了)为止，除非当前线程发生中断或者selector的wakeup方法被调用。
② select(long time)：该方法和select()类似，该方法也会导致阻塞直到至少一个channel被选择(即，该channel注册的事件发生了)为止，除非下面3种情况任意一种发生：a) 设置的超时时间到达；b) 当前线程发生中断；c) selector的wakeup方法被调用
③ selectNow()：该方法不会发生阻塞，如果没有一个channel被选择也会立即返回。

我们主要来看看select()的实现 ：int n = selector.select();

```
    public int select() throws IOException {
        return select(0);
    }

```

最终会调用到EPollSelectorImpl的doSelect

```
    protected int doSelect(long timeout) throws IOException {
        if (closed)
            throw new ClosedSelectorException();
        processDeregisterQueue();
        try {
            begin();
            pollWrapper.poll(timeout);
        } finally {
            end();
        }
        processDeregisterQueue();
        int numKeysUpdated = updateSelectedKeys();
        if (pollWrapper.interrupted()) {
            // Clear the wakeup pipe
            pollWrapper.putEventOps(pollWrapper.interruptedIndex(), 0);
            synchronized (interruptLock) {
                pollWrapper.clearInterrupted();
                IOUtil.drain(fd0);
                interruptTriggered = false;
            }
        }
        return numKeysUpdated;
    }

```

① 先处理注销的selectionKey队列
② 进行底层的epoll_wait操作
③ 再次对注销的selectionKey队列进行处理
④ 更新被选择的selectionKey

先来看processDeregisterQueue():

```
    void processDeregisterQueue() throws IOException {
        Set var1 = this.cancelledKeys();
        synchronized(var1) {
            if (!var1.isEmpty()) {
                Iterator var3 = var1.iterator();

                while(var3.hasNext()) {
                    SelectionKeyImpl var4 = (SelectionKeyImpl)var3.next();

                    try {
                        this.implDereg(var4);
                    } catch (SocketException var12) {
                        IOException var6 = new IOException("Error deregistering key");
                        var6.initCause(var12);
                        throw var6;
                    } finally {
                        var3.remove();
                    }
                }
            }

        }
    }

```

从cancelledKeys集合中依次取出注销的SelectionKey，执行注销操作，将处理后的SelectionKey从cancelledKeys集合中移除。执行processDeregisterQueue()后cancelledKeys集合会为空。

```
    protected void implDereg(SelectionKeyImpl ski) throws IOException {
        assert (ski.getIndex() >= 0);
        SelChImpl ch = ski.channel;
        int fd = ch.getFDVal();
        fdToKey.remove(Integer.valueOf(fd));
        pollWrapper.remove(fd);
        ski.setIndex(-1);
        keys.remove(ski);
        selectedKeys.remove(ski);
        deregister((AbstractSelectionKey)ski);
        SelectableChannel selch = ski.channel();
        if (!selch.isOpen() && !selch.isRegistered())
            ((SelChImpl)selch).kill();
    }

```

注销会完成下面的操作：
① 将已经注销的selectionKey从fdToKey( 文件描述与SelectionKeyImpl的映射表 )中移除
② 将selectionKey所代表的channel的文件描述符从EPollArrayWrapper中移除
③ 将selectionKey从keys集合中移除，这样下次selector.select()就不会再将该selectionKey注册到epoll中监听
④ 也会将selectionKey从对应的channel中注销
⑤ 最后如果对应的channel已经关闭并且没有注册其他的selector了，则将该channel关闭
完成👆的操作后，注销的SelectionKey就不会出现先在keys、selectedKeys以及cancelKeys这3个集合中的任何一个。

接着我们来看EPollArrayWrapper.poll(timeout)：

```
    int poll(long timeout) throws IOException {
        updateRegistrations();
        updated = epollWait(pollArrayAddress, NUM_EPOLLEVENTS, timeout, epfd);
        for (int i=0; i<updated; i++) {
            if (getDescriptor(i) == incomingInterruptFD) {
                interruptedIndex = i;
                interrupted = true;
                break;
            }
        }
        return updated;
    }

```

updateRegistrations()方法会将已经注册到该selector的事件(eventsLow或eventsHigh)通过调用epollCtl(epfd, opcode, fd, events); 注册到linux系统中。
这里epollWait就会调用linux底层的epoll_wait方法，并返回在epoll_wait期间有事件触发的entry的个数

再看updateSelectedKeys()：

```
    private int updateSelectedKeys() {
        int entries = pollWrapper.updated;
        int numKeysUpdated = 0;
        for (int i=0; i<entries; i++) {
            int nextFD = pollWrapper.getDescriptor(i);
            SelectionKeyImpl ski = fdToKey.get(Integer.valueOf(nextFD));
            // ski is null in the case of an interrupt
            if (ski != null) {
                int rOps = pollWrapper.getEventOps(i);
                if (selectedKeys.contains(ski)) {
                    if (ski.channel.translateAndSetReadyOps(rOps, ski)) {
                        numKeysUpdated++;
                    }
                } else {
                    ski.channel.translateAndSetReadyOps(rOps, ski);
                    if ((ski.nioReadyOps() & ski.nioInterestOps()) != 0) {
                        selectedKeys.add(ski);
                        numKeysUpdated++;
                    }
                }
            }
        }
        return numKeysUpdated;
    }

```

该方法会从通过EPollArrayWrapper pollWrapper 以及 fdToKey( 构建文件描述符-SelectorKeyImpl映射表 )来获取有事件触发的SelectionKeyImpl对象，然后将SelectionKeyImpl放到selectedKey集合( 有事件触发的selectionKey集合，可以通过selector.selectedKeys()方法获得 )中，即selectedKeys。并重新设置SelectionKeyImpl中相关的readyOps值。


但是，这里要注意两点：


① 如果SelectionKeyImpl已经存在于selectedKeys集合中，并且发现触发的事件已经存在于readyOps中了，则不会使numKeysUpdated++；这样会使得我们无法得知该事件的变化。


👆这点说明了为什么我们要在每次从selectedKey中获取到Selectionkey后，将其从selectedKey集合移除，就是为了当有事件触发使selectionKey能正确到放入selectedKey集合中，并正确的通知给调用者。


再者，如果不将已经处理的SelectionKey从selectedKey集合中移除，那么下次有新事件到来时，在遍历selectedKey集合时又会遍历到这个SelectionKey，这个时候就很可能出错了。比如，如果没有在处理完OP_ACCEPT事件后将对应SelectionKey从selectedKey集合移除，那么下次遍历selectedKey集合时，处理到到该SelectionKey，相应的ServerSocketChannel.accept()将返回一个空(null)的SocketChannel。


② 如果发现channel所发生I/O事件不是当前SelectionKey所感兴趣，则不会将SelectionKeyImpl放入selectedKeys集合中，也不会使numKeysUpdated++

### epoll原理

select，poll，epoll都是IO多路复用的机制。I/O多路复用就是通过一种机制，一个进程可以监视多个描述符，一旦某个描述符就绪（一般是读就绪或者写就绪），能够通知程序进行相应的读写操作。但select，poll，epoll本质上都是同步I/O，因为他们都需要在读写事件就绪后自己负责进行读写，也就是说这个读写过程是阻塞的，而异步I/O则无需自己负责进行读写，异步I/O的实现会负责把数据从内核拷贝到用户空间。

epoll是Linux下的一种IO多路复用技术，可以非常高效的处理数以百万计的socket句柄。

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而epoll事先通过epoll_ctl()来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知。(此处去掉了遍历文件描述符，而是通过监听回调的的机制。这正是epoll的魅力所在。)
如果没有大量的idle -connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle- connection，就会发现epoll的效率大大高于select/poll。

注意：linux下Selector底层是通过epoll来实现的，当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

先看看使用c封装的3个epoll系统调用:

*   **int epoll_create(int size)**
    epoll_create建立一个epoll对象。参数size是内核保证能够正确处理的最大句柄数，多于这个最大数时内核可不保证效果。
*   **int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)**
    epoll_ctl可以操作epoll_create创建的epoll，如将socket句柄加入到epoll中让其监控，或把epoll正在监控的某个socket句柄移出epoll。
*   **int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout)**
    epoll_wait在调用时，在给定的timeout时间内，所监控的句柄中有事件发生时，就返回用户态的进程。

    大概看看epoll内部是怎么实现的：

1.  epoll初始化时，会向内核注册一个文件系统，用于存储被监控的句柄文件，调用epoll_create时，会在这个文件系统中创建一个file节点。同时epoll会开辟自己的内核高速缓存区，以红黑树的结构保存句柄，以支持快速的查找、插入、删除。还会再建立一个list链表，用于存储准备就绪的事件。
2.  当执行epoll_ctl时，除了把socket句柄放到epoll文件系统里file对象对应的红黑树上之外，还会给内核中断处理程序注册一个回调函数，告诉内核，如果这个句柄的中断到了，就把它放到准备就绪list链表里。所以，当一个socket上有数据到了，内核在把网卡上的数据copy到内核中后，就把socket插入到就绪链表里。
3.  当epoll_wait调用时，仅仅观察就绪链表里有没有数据，如果有数据就返回，否则就sleep，超时时立刻返回。

    epoll的两种工作模式：

*   LT：level-trigger，水平触发模式，只要某个socket处于readable/writable状态，无论什么时候进行epoll_wait都会返回该socket。
*   ET：edge-trigger，边缘触发模式，只有某个socket从unreadable变为readable或从unwritable变为writable时，epoll_wait才会返回该socket。

socket读数据





![](https://upload-images.jianshu.io/upload_images/4235178-55ea1cf846c7d84c.png?imageMogr2/auto-orient/strip|imageView2/2/w/540/format/webp)





socket写数据





![](https://upload-images.jianshu.io/upload_images/4235178-39c86c1d52d6abce.png?imageMogr2/auto-orient/strip|imageView2/2/w/585/format/webp)





最后顺便说下在Linux系统中JDK NIO使用的是 LT ，而Netty epoll使用的是 ET。

## 后记

因为本人对计算机系统组成以及C语言等知识比较欠缺，因为文中相关知识点的表示也相当“肤浅”，如有不对不妥的地方望读者指出。同时我也会继续加强对该方面知识点的学习~

## 参考文章

[http://www.jianshu.com/p/0d497fe5484a](https://www.jianshu.com/p/0d497fe5484a)
[http://remcarpediem.com/2017/04/02/Netty](https://link.jianshu.com/?t=http://remcarpediem.com/2017/04/02/Netty)源码-三-I-O模型和Java-NIO底层原理/
圣思园netty课程
