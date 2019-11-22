本文转自https://www.xilidou.com/2018/03/12/redis-data/
作者：犀利豆

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

系列文章将整理在我的个人博客：
> www.how2playlife.com

本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，Redis基本的使用方法，Redis的基本数据结构，以及一些进阶的使用方法，同时也需要进一步了解Redis的底层数据结构，再接着，还会带来Redis主从复制、集群、分布式锁等方面的相关内容，以及作为缓存的一些使用方法和注意事项，以便让你更完整地了解整个Redis相关的技术体系，形成自己的知识框架。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

原文地址：https://www.xilidou.com/2018/03/22/redis-event/

Redis 是一个事件驱动的内存数据库，服务器需要处理两种类型的事件。

文件事件
时间事件
下面就会介绍这两种事件的实现原理。

文件事件
Redis 服务器通过 socket 实现与客户端（或其他redis服务器）的交互,文件事件就是服务器对 socket 操作的抽象。 Redis 服务器，通过监听这些 socket 产生的文件事件并处理这些事件，实现对客户端调用的响应。

Reactor
Redis 基于 Reactor 模式开发了自己的事件处理器。

这里就先展开讲一讲 Reactor 模式。看下图：
![image](https://img.xilidou.com/img/Reactor.jpg)
reactor

“I/O 多路复用模块”会监听多个 FD ，当这些FD产生，accept，read，write 或 close 的文件事件。会向“文件事件分发器（dispatcher）”传送事件。

文件事件分发器（dispatcher）在收到事件之后，会根据事件的类型将事件分发给对应的 handler。

我们顺着图，从上到下的逐一讲解 Redis 是怎么实现这个 Reactor 模型的。

I/O 多路复用模块
Redis 的 I/O 多路复用模块，其实是封装了操作系统提供的 select，epoll，avport 和 kqueue 这些基础函数。向上层提供了一个统一的接口，屏蔽了底层实现的细节。

一般而言 Redis 都是部署到 Linux 系统上，所以我们就看看使用 Redis 是怎么利用 linux 提供的 epoll 实现I/O 多路复用。

首先看看 epoll 提供的三个方法：
    
    /*
     * 创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
     */
    int epoll_create(int size)；
    
    /*
     * 可以理解为，增删改 fd 需要监听的事件
     * epfd 是 epoll_create() 创建的句柄。
     * op 表示 增删改
     * epoll_event 表示需要监听的事件，Redis 只用到了可读，可写，错误，挂断 四个状态
     */
    int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
    
    /*
     * 可以理解为查询符合条件的事件
     * epfd 是 epoll_create() 创建的句柄。
     * epoll_event 用来存放从内核得到事件的集合
     * maxevents 获取的最大事件数
     * timeout 等待超时时间
     */
    int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
    再看 Redis 对文件事件，封装epoll向上提供的接口：
    
    
    /*
     * 事件状态
     */
    typedef struct aeApiState {
    
        // epoll_event 实例描述符
        int epfd;
    
        // 事件槽
        struct epoll_event *events;
    
    } aeApiState;
    
    /*
     * 创建一个新的 epoll 
     */
    static int  aeApiCreate(aeEventLoop *eventLoop)
    /*
     * 调整事件槽的大小
     */
    static int  aeApiResize(aeEventLoop *eventLoop, int setsize)
    /*
     * 释放 epoll 实例和事件槽
     */
    static void aeApiFree(aeEventLoop *eventLoop)
    /*
     * 关联给定事件到 fd
     */
    static int  aeApiAddEvent(aeEventLoop *eventLoop, int fd, int mask)
    /*
     * 从 fd 中删除给定事件
     */
    static void aeApiDelEvent(aeEventLoop *eventLoop, int fd, int mask)
    /*
     * 获取可执行事件
     */
    static int  aeApiPoll(aeEventLoop *eventLoop, struct timeval *tvp)
所以看看这个ae_peoll.c 如何对 epoll 进行封装的：

aeApiCreate() 是对 epoll.epoll_create() 的封装。
aeApiAddEvent()和aeApiDelEvent() 是对 epoll.epoll_ctl()的封装。
aeApiPoll() 是对 epoll_wait()的封装。
这样 Redis 的利用 epoll 实现的 I/O 复用器就比较清晰了。

再往上一层次我们需要看看 ea.c 是怎么封装的？

首先需要关注的是事件处理器的数据结构：


    typedef struct aeFileEvent {
    
        // 监听事件类型掩码，
        // 值可以是 AE_READABLE 或 AE_WRITABLE ，
        // 或者 AE_READABLE | AE_WRITABLE
        int mask; /* one of AE_(READABLE|WRITABLE) */
    
        // 读事件处理器
        aeFileProc *rfileProc;
    
        // 写事件处理器
        aeFileProc *wfileProc;
    
        // 多路复用库的私有数据
        void *clientData;
    
    } aeFileEvent;
mask 就是可以理解为事件的类型。

除了使用 ae_peoll.c 提供的方法外,ae.c 还增加 “增删查” 的几个 API。

增:aeCreateFileEvent
删:aeDeleteFileEvent
查: 查包括两个维度 aeGetFileEvents 获取某个 fd 的监听类型和aeWait等待某个fd 直到超时或者达到某个状态。
事件分发器（dispatcher）
Redis 的事件分发器 ae.c/aeProcessEvents 不但处理文件事件还处理时间事件，所以这里只贴与文件分发相关的出部分代码，dispather 根据 mask 调用不同的事件处理器。

    //从 epoll 中获关注的事件
    numevents = aeApiPoll(eventLoop, tvp);
    for (j = 0; j < numevents; j++) {
        // 从已就绪数组中获取事件
        aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
    
        int mask = eventLoop->fired[j].mask;
        int fd = eventLoop->fired[j].fd;
        int rfired = 0;
    
        // 读事件
        if (fe->mask & mask & AE_READABLE) {
            // rfired 确保读/写事件只能执行其中一个
            rfired = 1;
            fe->rfileProc(eventLoop,fd,fe->clientData,mask);
        }
        // 写事件
        if (fe->mask & mask & AE_WRITABLE) {
            if (!rfired || fe->wfileProc != fe->rfileProc)
                fe->wfileProc(eventLoop,fd,fe->clientData,mask);
        }
    
        processed++;
    }
可以看到这个分发器，根据 mask 的不同将事件分别分发给了读事件和写事件。

文件事件处理器的类型
Redis 有大量的事件处理器类型，我们就讲解处理一个简单命令涉及到的三个处理器：

acceptTcpHandler 连接应答处理器，负责处理连接相关的事件，当有client 连接到Redis的时候们就会产生 AE_READABLE 事件。引发它执行。
readQueryFromClinet 命令请求处理器，负责读取通过 sokect 发送来的命令。
sendReplyToClient 命令回复处理器，当Redis处理完命令，就会产生 AE_WRITEABLE 事件，将数据回复给 client。
文件事件实现总结
我们按照开始给出的 Reactor 模型，从上到下讲解了文件事件处理器的实现，下面将会介绍时间时间的实现。

时间事件
Reids 有很多操作需要在给定的时间点进行处理，时间事件就是对这类定时任务的抽象。

先看时间事件的数据结构：

    /* Time event structure
     *
     * 时间事件结构
     */
    typedef struct aeTimeEvent {
    
        // 时间事件的唯一标识符
        long long id; /* time event identifier. */
    
        // 事件的到达时间
        long when_sec; /* seconds */
        long when_ms; /* milliseconds */
    
        // 事件处理函数
        aeTimeProc *timeProc;
    
        // 事件释放函数
        aeEventFinalizerProc *finalizerProc;
    
        // 多路复用库的私有数据
        void *clientData;
    
        // 指向下个时间事件结构，形成链表
        struct aeTimeEvent *next;
    
    } aeTimeEvent;
看见 next 我们就知道这个 aeTimeEvent 是一个链表结构。看图：
![image](https://img.xilidou.com/img/timeEvent.jpg)

timeEvent

注意这是一个按照id倒序排列的链表，并没有按照事件顺序排序。

processTimeEvent
Redis 使用这个函数处理所有的时间事件，我们整理一下执行思路：

记录最新一次执行这个函数的时间，用于处理系统时间被修改产生的问题。
遍历链表找出所有 when_sec 和 when_ms 小于现在时间的事件。
执行事件对应的处理函数。
检查事件类型，如果是周期事件则刷新该事件下一次的执行事件。
否则从列表中删除事件。
综合调度器（aeProcessEvents）
综合调度器是 Redis 统一处理所有事件的地方。我们梳理一下这个函数的简单逻辑：

    // 1. 获取离当前时间最近的时间事件
    shortest = aeSearchNearestTimer(eventLoop);
    
    // 2. 获取间隔时间
    timeval = shortest - nowTime;
    
    // 如果timeval 小于 0，说明已经有需要执行的时间事件了。
    if(timeval < 0){
        timeval = 0
    }
    
    // 3. 在 timeval 时间内，取出文件事件。
    numevents = aeApiPoll(eventLoop, timeval);
    
    // 4.根据文件事件的类型指定不同的文件处理器
    if (AE_READABLE) {
        // 读事件
        rfileProc(eventLoop,fd,fe->clientData,mask);
    }
        // 写事件
    if (AE_WRITABLE) {
        wfileProc(eventLoop,fd,fe->clientData,mask);
    }
以上的伪代码就是整个 Redis 事件处理器的逻辑。

我们可以再看看谁执行了这个 aeProcessEvents:


    void aeMain(aeEventLoop *eventLoop) {
    
        eventLoop->stop = 0;
    
        while (!eventLoop->stop) {
    
            // 如果有需要在事件处理前执行的函数，那么运行它
            if (eventLoop->beforesleep != NULL)
                eventLoop->beforesleep(eventLoop);
    
            // 开始处理事件
            aeProcessEvents(eventLoop, AE_ALL_EVENTS);
        }
    }
然后我们再看看是谁调用了 eaMain:

    int main(int argc, char **argv) {
        //一些配置和准备
        ...
        aeMain(server.el);
        
        //结束后的回收工作
        ...
    }
我们在 Redis 的 main 方法中找个了它。

这个时候我们整理出的思路就是:

Redis 的 main() 方法执行了一些配置和准备以后就调用 eaMain() 方法。

eaMain() while(true) 的调用 aeProcessEvents()。

所以我们说 Redis 是一个事件驱动的程序，期间我们发现，Redis 没有 fork 过任何线程。所以也可以说 Redis 是一个基于事件驱动的单线程应用。

总结
在后端的面试中 Redis 总是一个或多或少会问到的问题。

读完这篇文章你也许就能回答这几个问题：

为什么 Redis 是一个单线程应用？
为什么 Redis 是一个单线程应用，却有如此高的性能？
如果你用本文提供的知识点回答这两个问题，一定会在面试官心中留下一个高大的形象。

大家还可以阅读我的 Redis 相关的文章：
