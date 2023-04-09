# Table of Contents

  * [文件系统和VFS](#文件系统和vfs)
  * [进程和线程](#进程和线程)
  * [fork方法](#fork方法)
  * [写时复制](#写时复制)
  * [父子进程，僵尸进程，孤儿进程，守护进程](#父子进程，僵尸进程，孤儿进程，守护进程)
  * [进程组和会话](#进程组和会话)
  * [守护进程](#守护进程)
  * [硬连接和软连接](#硬连接和软连接)
  * [线程](#线程)
  * [线程模型](#线程模型)
  * [内核线程实现](#内核线程实现)
  * [文件系统](#文件系统)
  * [IO操作](#io操作)
* [文件描述符](#文件描述符)
* [write函数](#write函数)
* [read函数](#read函数)
* [**read/write的语义：为什么会阻塞？**](#readwrite的语义：为什么会阻塞？)
  * [Linux常用命令和基础知识](#linux常用命令和基础知识)
    * [查看进程](#查看进程)
* [ps -l](#ps--l)
* [ps aux](#ps-aux)
* [ps aux | grep threadx](#ps-aux--grep-threadx)
* [top -d 2](#top--d-2)
* [pstree -A](#pstree--a)
* [netstat -anp | grep port](#netstat--anp--grep-port)
    * [文件操作](#文件操作)
    * [权限操作](#权限操作)
  * [连接操作](#连接操作)
* [ln /etc/crontab .](#ln-etccrontab-)
* [ll -i /etc/crontab crontab](#ll--i-etccrontab-crontab)
* [ll -i /etc/crontab /root/crontab2](#ll--i-etccrontab-rootcrontab2)
  * [获取内容](#获取内容)
  * [搜索和定位](#搜索和定位)
* [locate [-ir] keyword](#locate-[-ir]-keyword)
* [find [basedir] [option]](#find-[basedir]-[option])
  * [压缩](#压缩)
  * [管道指令](#管道指令)
  * [正则](#正则)
  * [linux指令实践和常见场景](#linux指令实践和常见场景)
  * [查看进程状态](#查看进程状态)
  * [strace](#strace)
  * [tcpdump](#tcpdump)
  * [nc](#nc)
  * [curl](#curl)
  * [lsof](#lsof)
  * [ss](#ss)
  * [awk/sed](#awksed)
  * [vim](#vim)
  * [crontab](#crontab)
  * [service](#service)
  * [free](#free)
  * [top](#top)
  * [df](#df)
  * [kill](#kill)
  * [mount](#mount)
  * [chmod](#chmod)
  * [chown](#chown)
  * [ifconfig](#ifconfig)
  * [uname](#uname)
  * [实际场景问题](#实际场景问题)



---
title: Linux内核与基础命令学习总结
date: 2018-07-09 22:33:14
tags:
	- Linux
categories:
	- 后端
	- 技术总结
---
这部分内容主要是基于一些关于Linux系统的内核基础和基本命令的学习总结，内容不全面，只讲述了其中的一小部分，后续会再补充，如有错误，还请见谅。


Linux操作系统



Linux操作系统博大精深，其中对线程，IO，文件系统等概念的实现都很有借鉴意义。
<!-- more -->


​        
## 文件系统和VFS

文件系统的inode上面讲过了。VFS主要用于屏蔽底层的不同文件系统，比如接入网络中的nfs文件系统，亦或是windows文件系统，正常情况下难以办到，而vfs通过使用IO操作的posix规范来规定所有文件读写操作，每个文件系统只需要实现这些操作就可以接入VFS，不需要重新安装文件系统。
## 进程和线程

    > 进程、程序与线程
    > 
    > 程序
    > 
    >  程序，简单的来说就是存在磁盘上的二进制文件，是可以内核所执行的代码 
    > 
    > 进程
    > 
    >  当一个用户启动一个程序，将会在内存中开启一块空间，这就创造了一个进程，一个进程包含一个独一无二的PID，和执行者的权限属性参数，以及程序所需代码与相关的资料。
    >  进程是系统分配资源的基本单位。
    >  一个进程可以衍生出其他的子进程，子进程的相关权限将会沿用父进程的相关权限。
    > 
    > 线程
    > 
    >  每个进程包含一个或多个线程，线程是进程内的活动单元，是负责执行代码和管理进程运行状态的抽象。
    >  线程是独立运行和调度的基本单位。



>  子进程和父进程
> 进程的层次结构（父进程与子进程）在进程执行的过程中可能会衍生出其他的进程，称之为子进程，子进程拥有一个指明其父进程PID的PPID。子进程可以继承父进程的环境变量和权限参数。
> 
> 于是，linux系统中就诞生了进程的层次结构——进程树。
> 进程树的根是第一个进程（init进程）。
> 
> 过程调用的流程： fork & exec一个进程生成子进程的过程是，系统首先复制(fork)一份父进程，生成一个暂存进程，这个暂存进程和父进程的区别是pid不一样，而且拥有一个ppid，这时候系统再去执行(exec)这个暂存进程，让他加载实际要运行的程序，最终成为一个子进程的存在。
> 
> 服务与进程
> 
> 简单的说服务(daemon)就是常驻内存的进程，通常服务会在开机时通过init.d中的一段脚本被启动。
> 
> 进程通信
> 
> 进程通信的几种基本方式：管道，信号量，消息队列，共享内存，快速用户控件互斥。 
> 
## fork方法
  一个进程，包括代码、数据和分配给进程的资源。fork（）函数通过系统调用创建一个与原来进程几乎完全相同的进程，

也就是两个进程可以做完全相同的事，但如果初始参数或者传入的变量不同，两个进程也可以做不同的事。


    一个进程调用fork（）函数后，系统先给新的进程分配资源，例如存储数据和代码的空间。然后把原来的进程的所有值都

复制到新的新进程中，只有少数值与原来的进程的值不同。相当于克隆了一个自己。

    fork调用的一个奇妙之处就是它仅仅被调用一次，却能够返回两次，它可能有三种不同的返回值：
        1）在父进程中，fork返回新创建子进程的进程ID；
        2）在子进程中，fork返回0；
        3）如果出现错误，fork返回一个负值；

如何理解pid在父子进程中不同？

其实就相当于链表，进程形成了链表，父进程的pid指向了子进程的pid，因为子进程没有子进程，所以pid为0。

## 写时复制

    传统的fork机制是，调用fork时，内核会复制所有的内部数据结构，复制进程的页表项，然后把父进程的地址空间按页复制给子进程（非常耗时）。
    
    现代的fork机制采用了一种惰性算法的优化策略。
    
    为了避免复制时系统开销，就尽可能的减少“复制”操作，当多个进程需要读取他们自己那部分资源的副本时，并不复制多个副本出来，而是为每个进程设定一个文件指针，让它们读取同一个实际文件。
    
    显然这样的方式会在写入时产生冲突（类似并发），于是当某个进程想要修改自己的那个副本时，再去复制该资源，（只有写入时才复制，所以叫写时复制）这样就减少了复制的频率。

## 父子进程，僵尸进程，孤儿进程，守护进程

父进程通过fork产生子进程。

孤儿进程：当子进程未结束时父进程异常退出，原本需要由父进程进行处理的子进程变成了孤儿进程，init系统进程会把这些进程领养，避免他们成为孤儿。

僵尸进程：当子进程结束时，会在内存中保留一部分数据结构等待父亲进程显式结束，如果父进程没有执行结束操作，则会导致子进程的剩余结构无法被释放，占用空间造成严重后果。

守护进程：守护进程用于监控其他进程，当发现大量僵尸进程时，会找到他们的父节点并杀死，同时让init线程认养他们以便释放这些空间。

僵尸进程是有害的，孤儿进程由于内核进程的认养不会造成危害。

## 进程组和会话

> 会话和进程组进程组每个进程都属于某个进程组，进程组就是由一个或者多个为了实现作业控制而相互关联的进程组成的。
> 
> 一个进程组的id是进程组首进程的pid（如果一个进程组只有一个进程，那进程组和进程其实没啥区别）。
> 
> 进程组的意义在于，信号可以发送给进程组中的所有进程。这样可以实现对多个进程的同时操作。
> 会话会话是一个或者多个进程组的集合。
> 
> 一般来说，会话(session)和shell没有什么本质上的区别。
> 我们通常使用用户登录一个终端进行一系列操作这样的例子来描述一次会话。

举例

$cat ship-inventory.txt | grep

booty|sort上面就是在某次会话中的一个shell命令，它会产生一个由3个进程组成的进程组。
## 守护进程
守护进程（服务）守护进程(daemon)运行在后台，不与任何控制终端相关联。通常在系统启动时通过init脚本被调用而开始运行。

在linux系统中，守护进程和服务没有什么区别。
对于一个守护进程，有两个基本的要求：其一：必须作为init进程的子进程运行，其二：不与任何控制终端交互。


## 硬连接和软连接

硬链接指的是不同的文件名指向同一个inode节点，比如某个目录下的a和另一个目录下的b，建立一个软连接让a指向b，则a和b共享同一个inode。

软连接是指一个文件的inode节点不存数据，而是存储着另一个文件的绝对路径，访问文件内容时实际上是去访问对应路径下的文件inode，这样的话文件发生改动或者移动都会导致软连接失效。

## 线程

线程基础概念线程是进程内的执行单元（比进程更低一层的概念），具体包括 虚拟处理器，堆栈，程序状态等。
可以认为 线程是操作系统调度的最小执行单元。

现代操作系统对用户空间做两个基础抽象:虚拟内存和虚拟处理器。这使得进程内部“感觉”自己独占机器资源。

虚拟内存系统会为每个进程分配独立的内存空间，这会让进程以为自己独享全部的RAM。

但是同一个进程内的所有线程共享该进程的内存空间。
虚拟处理器这是一个针对线程的概念，它让每个线程都“感觉”自己独享CPU。实际上对于进程也是一样的。

## 线程模型

线程模型线程的概念同时存在于内核和用户空间中。下面介绍三种线程模型。

    内核级线程模型每个内核线程直接转换成用户空间的线程。即内核线程：用户空间线程=1：1
    
    用户级线程模型这种模型下，一个保护了n个线程的用户进程只会映射到一个内核进程。即n:1。
    可以减少上下文切换的成本，但在linux下没什么意义，因为linux下进程间的上下文切换本身就没什么消耗，所以很少使用。
    
    混合式线程模型上述两种模型的混合，即n:m型。
    很难实现。
## 内核线程实现

系统线程实现：PThreads
原始的linux系统调用中，没有像C++11或者是Java那样完整的线程库。

整体看来pthread的api比较冗余和复杂，但是基本操作也主要是 创建、退出等。

1.创建线程

      int pthread_create
      
      (若线程创建成功，则返回0。若线程创建失败，则返回出错编号)
      
    　　注意:线程创建者和新建线程之间没有fork()调用那样的父子关系，它们是对等关系。调用pthread_create()创建线程后，线程创建者和新建线程哪个先运行是不确定的，特别是在多处理机器上。

2.终止线程
    
      void pthread_exit(void *value_ptr);
      
         线程调用pthread_exit()结束自己，参数value_ptr作为线程的返回值被调用pthread_join的线程使用。由于一个进程中的多个线程是共享数据段的，因此通常在线程退出之后，退出线程所占用的资源并不会随着线程的终止而得到释放，但是可以用pthread_join()函数来同步并释放资源


3.取消线程

      int pthread_cancel(pthread_t thread);
      
    　　注意：若是在整个程序退出时，要终止各个线程，应该在成功发送 CANCEL指令后，使用 pthread_join函数，等待指定的线程已经完全退出以后，再继续执行；否则，很容易产生 “段错误”。

4.连接线程（阻塞）

      int pthread_join(pthread_t thread, void **value_ptr);
      
    　　等待线程thread结束，并设置*value_ptr为thread的返回值。pthread_join阻塞调用者，一直到线程thread结束为止。当函数返回时，被等待线程的资源被收回。如果进程已经结束，那么该函数会立即返回。并且thread指定的线程必须是joinable的。
    
    需要留意的一点是linux机制下，线程存在一个被称为joinable的状态。下面简要了解一下：

Join和Detach
这块的概念，非常类似于之前父子进程那部分，等待子进程退出的内容（一系列的wait函数）。

linux机制下，线程存在两种不同的状态：joinable和unjoinable。

    如果一个线程被标记为joinable时，即便它的线程函数执行完了，或者使用了pthread_exit()结束了该线程，它所占用的堆栈资源和进程描述符都不会被释放（类似僵尸进程），这种情况应该由线程的创建者调用pthread_join()来等待线程的结束并回收其资源（类似wait系函数）。默认情况下创建的线程都是这种状态。
    
    如果一个线程被标记成unjoinable，称它被分离(detach)了，这时候如果该线程结束，所有它的资源都会被自动回收。省去了给它擦屁股的麻烦。
    
    因为创建的线程默认都是joinable的，所以要么在父线程调用pthread_detach(thread_id)将其分离，要么在线程内部，调用pthread_detach(pthread_self())来把自己标记成分离的。



## 文件系统

    文件描述符在linux内核中，文件是用一个整数来表示的，称为 文件描述符，通俗的来说，你可以理解它是文件的id（唯一标识符）
    
    普通文件
    普通文件就是字节流组织的数据。
    文件并不是通过和文件名关联来实现的，而是通过关联索引节点来实现的，文件节点拥有文件系统为普通文件分配的唯一整数值(ino)，并且存放着一些文件的相关元数据。
    
    目录与链接
    正常情况下文件是通过文件名来打开的。
    目录是可读名称到索引编号之间的映射，名称和索引节点之间的配对称为链接。
    可以把目录看做普通文件，只是它包含着文件名称到索引节点的映射（链接）



文件系统是基于底层存储建立的一个树形文件结构。比较经典的是Linux的文件系统，首先在硬盘的超级块中安装文件系统，磁盘引导时会加载文件系统的信息。

linux使用inode来标识任意一个文件。inode存储除了文件名以外的文件信息，包括创建时间，权限，以及一个指向磁盘存储位置的指针，那里才是真正存放数据的地方。

一个目录也是一个inode节点。

详细阐述一次文件访问的过程：
    
    首先用户ls查看目录。由于一个目录也是一个文件，所以相当于是看目录文件下有哪些东西。
    
    实际上目录文件是一个特殊的inode节点，它不需要存储实际数据，而只是维护一个文件名到inode的映射表。
    
    于是我们ls到另一个目录。同理他也是一个inode。我们在这个inode下执行vi操作打开某个文件，于是linux通过inode中的映射表找到了我们请求访问的文件名对应的inode。
    
    然后寻道到对应的磁盘位置，读取内容到缓冲区，通过系统调用把内容读到内存中，最后进行访问。
## IO操作

# 文件描述符

　　对于内核而言，所有打开的文件都通过文件描述符引用。文件描述符是一个非负整数。当打开一个现有文件或创建一个新文件时，内核向进程返回一个文件描述符。当读或写一个文件时，使用open或create返回的文件描述符表示该文件，将其作为参数传给read或write函数。

# write函数

 　　write函数定义如下：



#include <unistd> ssize_t write(int filedes, void *buf, size_t nbytes); // 返回：若成功则返回写入的字节数，若出错则返回-1 // filedes：文件描述符 // buf:待写入数据缓存区 // nbytes:要写入的字节数目录



　　同样，为了保证写入数据的完整性，在《UNIX网络编程 卷1》中，作者将该函数进行了封装，具体程序如下：

![](http://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)



![复制代码](http://common.cnblogs.com/images/copycode.gif)

 1 ssize_t                        /* Write "n" bytes to a descriptor. */
 2 writen(int fd, const void *vptr, size_t n)
 3 {
 4     size_t nleft;
 5     ssize_t nwritten;
 6     const char *ptr;
 7 
 8     ptr = vptr; 9     nleft = n; 10     while (nleft > 0) { 11         if ( (nwritten = write(fd, ptr, nleft)) <= 0) { 12             if (nwritten < 0 && errno == EINTR) 13                 nwritten = 0;        /* and call write() again */
14             else
15                 return(-1);            /* error */
16 } 17 
18         nleft -= nwritten; 19         ptr   += nwritten; 20 } 21     return(n); 22 } 23 /* end writen */
24 
25 void
26 Writen(int fd, void *ptr, size_t nbytes) 27 { 28     if (writen(fd, ptr, nbytes) != nbytes) 29         err_sys("writen error"); 30 }目录

![复制代码](http://common.cnblogs.com/images/copycode.gif)





# read函数

　　read函数定义如下：



#include <unistd> ssize_t read(int filedes, void *buf, size_t nbytes); // 返回：若成功则返回读到的字节数，若已到文件末尾则返回0，若出错则返回-1 // filedes：文件描述符 // buf:读取数据缓存区 // nbytes:要读取的字节数目录



 　　有几种情况可使实际读到的字节数少于要求读的字节数：

　　1）读普通文件时，在读到要求字节数之前就已经达到了文件末端。例如，若在到达文件末端之前还有30个字节，而要求读100个字节，则read返回30，下一次再调用read时，它将返回0（文件末端）。

　　2）当从终端设备读时，通常一次最多读一行。

　　3）当从网络读时，网络中的缓存机构可能造成返回值小于所要求读的字结束。

　　4）当从管道或FIFO读时，如若管道包含的字节少于所需的数量，那么read将只返回实际可用的字节数。

　　5）当从某些面向记录的设备（例如磁带）读时，一次最多返回一个记录。

　　6）当某一个信号造成中断，而已经读取了部分数据。

　　在《UNIX网络编程 卷1》中，作者将该函数进行了封装，以确保数据读取的完整，具体程序如下：

![](http://images.cnblogs.com/OutliningIndicators/ExpandedBlockStart.gif)



![复制代码](http://common.cnblogs.com/images/copycode.gif)

 1 ssize_t                        /* Read "n" bytes from a descriptor. */
 2 readn(int fd, void *vptr, size_t n)
 3 {
 4     size_t nleft;
 5     ssize_t nread;
 6     char *ptr;
 7 
 8     ptr = vptr; 9     nleft = n; 10     while (nleft > 0) { 11         if ( (nread = read(fd, ptr, nleft)) < 0) { 12             if (errno == EINTR) 13                 nread = 0;        /* and call read() again */
14             else
15                 return(-1); 16         } else if (nread == 0) 17             break;                /* EOF */
18 
19         nleft -= nread; 20         ptr   += nread; 21 } 22     return(n - nleft);        /* return >= 0 */
23 } 24 /* end readn */
25 
26 ssize_t 27 Readn(int fd, void *ptr, size_t nbytes) 28 { 29 ssize_t        n; 30 
31     if ( (n = readn(fd, ptr, nbytes)) < 0) 32         err_sys("readn error"); 33     return(n); 34 }目录

![复制代码](http://common.cnblogs.com/images/copycode.gif)





本文下半部分摘自博文[浅谈TCP/IP网络编程中socket的行为](http://www.cnblogs.com/promise6522/archive/2012/03/03/2377935.html)。

# **read/write的语义：为什么会阻塞？**

　　先从write说起：



#include <unistd.h> ssize_t write(int fd, const void *buf, size_t count);目录



　　首先，write成功返回，**只是buf中的数据被复制到了kernel中的TCP发送缓冲区。**至于数据什么时候被发往网络，什么时候被对方主机接收，什么时候被对方进程读取，系统调用层面不会给予任何保证和通知。

　　write在什么情况下会阻塞？当kernel的该socket的发送缓冲区已满时。对于每个socket，拥有自己的send buffer和receive buffer。从Linux 2.6开始，两个缓冲区大小都由系统来自动调节（autotuning），但一般在default和max之间浮动。



# 获取socket的发送/接受缓冲区的大小：（后面的值是在Linux 2.6.38 x86_64上测试的结果）目录

sysctl net.core.wmem_default       #126976
sysctl net.core.wmem_max　　　　    #131071目录



　　已经发送到网络的数据依然需要暂存在send buffer中，只有收到对方的ack后，kernel才从buffer中清除这一部分数据，为后续发送数据腾出空间。接收端将收到的数据暂存在receive buffer中，自动进行确认。但如果socket所在的进程不及时将数据从receive buffer中取出，最终导致receive buffer填满，由于TCP的滑动窗口和拥塞控制，接收端会阻止发送端向其发送数据。这些控制皆发生在TCP/IP栈中，对应用程序是透明的，应用程序继续发送数据，最终导致send buffer填满，write调用阻塞。

　　一般来说，由于**接收端进程从socket读数据的速度**跟不上**发送端进程向socket写数据的速度**，最终导致**发送端write调用阻塞。**

　　而read调用的行为相对容易理解，从socket的receive buffer中拷贝数据到应用程序的buffer中。read调用阻塞，通常是发送端的数据没有到达。

## Linux常用命令和基础知识

### 查看进程

    1. ps
    查看某个时间点的进程信息
    
    示例一：查看自己的进程
    
    # ps -l
    示例二：查看系统所有进程
    
    # ps aux
    示例三：查看特定的进程
    
    # ps aux | grep threadx
    
    2. top
    实时显示进程信息
    
    示例：两秒钟刷新一次
    
    # top -d 2
    3. pstree
    查看进程树
    
    示例：查看所有进程树
    
    # pstree -A
    4. netstat
    查看占用端口的进程
    
    示例：查看特定端口的进程
    
    # netstat -anp | grep port

### 文件操作
ls -a  ,all列出全部文件包括隐藏

ls -l，list显示文件的全部属性

ls -d,仅列出目录本身

cd mkdir rmdir 常用不解释 rm -rf永久删除 cp复制 mv移动或改名

touch，更新文件时间或者建立新文件。


### 权限操作

    chmod rwx 分别对应 421
    
     chmod 754 .bashrc 将权限改为rwxr-xr--
    
    对应权限分配是对于 拥有者，所属群组，以及其他人。


文件默认权限

文件默认权限：文件默认没有可执行权限，因此为 666，也就是 -rw-rw-rw- 。

目录默认权限：目录必须要能够进入，也就是必须拥有可执行权限，因此为 777 ，也就是 drwxrwxrwx。


目录的权限

ps:拥有目录权限才能修改文件名，拥有文件权限是没用的

    文件名不是存储在一个文件的内容中，而是存储在一个文件所在的目录中。因此，拥有文件的 w 权限并不能对文件名进行修改。

目录存储文件列表，一个目录的权限也就是对其文件列表的权限。因此，目录的 r 权限表示可以读取文件列表；w 权限表示可以修改文件列表，具体来说，就是添加删除文件，对文件名进行修改；x 权限可以让该目录成为工作目录，x 权限是 r 和 w 权限的基础，如果不能使一个目录成为工作目录，也就没办法读取文件列表以及对文件列表进行修改了。


## 连接操作


硬链接：

    使用ln建立了一个硬连接，通过ll -i获得他们的inode节点。发现他们的inode节点是相同的。符合硬连接规定。


​    
    # ln /etc/crontab .
    # ll -i /etc/crontab crontab
    
    34474855 -rw-r--r--. 2 root root 451 Jun 10 2014 crontab
    34474855 -rw-r--r--. 2 root root 451 Jun 10 2014 /etc/crontab

软连接：

    符号链接文件保存着源文件所在的绝对路径，在读取时会定位到源文件上，可以理解为 Windows 的快捷方式。
    
    当源文件被删除了或者被移动到其他位置了，链接文件就打不开了。
    
    可以为目录建立链接。
    
    # ll -i /etc/crontab /root/crontab2
    
    34474855 -rw-r--r--. 2 root root 451 Jun 10 2014 /etc/crontab
    53745909 lrwxrwxrwx. 1 root root 12 Jun 23 22:31 /root/crontab2 -> /etc/crontab

## 获取内容

cat 读取内容 加上-n 按行打印

tac是cat的反向操作

more允许翻页查看，而不像cat一次显示全部内容

less可以先前翻页和向后翻页，more只能向前翻页

head 和tail 负责取得文件的前几行和后几行

## 搜索和定位

    1 which负责指令搜索，并显示第一条 比如which pwd，会找到pwd对应的程序。加-a 打印全部。
    
    2 whereis负责搜索文件， 后面接上dirname/filename
    
    文件搜索。速度比较快，因为它只搜索几个特定的目录。
    比如 whereis /bin hello.c
    
    3 locate
    文件搜索。可以用关键字或者正则表达式进行搜索。
    
    locate 使用 /var/lib/mlocate/ 这个数据库来进行搜索，它存储在内存中，并且每天更新一次，所以无法用 locate 搜索新建的文件。可以使用 updatedb 来立即更新数据库。
    
    # locate [-ir] keyword
    -r：正则表达式
    
    locate hello 
    locate he*
    vi heeee
    updatedb
    locate he?
    
    4. find
    文件搜索。可以使用文件的属性和权限进行搜索。
    
    # find [basedir] [option]
    example: find . -name "shadow*"
    
    find -name "hike"
    find +属性后缀 "属性"
    
    （一）与时间有关的选项
    
    -mtime  n ：列出在 n 天前的那一天修改过内容的文件
    
    （二）与文件拥有者和所属群组有关的选项
    
    -uid n
    -gid n
    -user name
    
    （三）与文件权限和名称有关的选项
    
    -name filename
    -size [+-]SIZE：搜寻比 SIZE 还要大 (+) 或小 (-) 的文件。这个 SIZE 的规格有：c: 代表 byte，k: 代表 1024bytes。所以，要找比 50KB 还要大的文件，就是 -size +50k
    -type TYPE

## 压缩
    gzip压缩和解压，还有bzip，xz等压缩
    
    而tar可以用打包，打包的时候也可以执行压缩
    
    压缩指令只能对一个文件进行压缩，而打包能够将多个文件打包成一个大文件。tar 不仅可以用于打包，也可以使用 gip、bzip2、xz 将打包文件进行压缩。
    
    $ tar [-z|-j|-J] [cv] [-f 新建的 tar 文件] filename...  ==打包压缩
    $ tar [-z|-j|-J] [tv] [-f 已有的 tar 文件]              ==查看
    $ tar [-z|-j|-J] [xv] [-f 已有的 tar 文件] [-C 目录]    ==解压缩


## 管道指令

1 |

2 cut切分数据，分成多列，last显示登陆者信息

## 正则


grep

    g/re/p（globally search a regular expression and print)，使用正则表示式进行全局查找并打印。
    
    $ grep [-acinv] [--color=auto] 搜寻字符串 filename
    -c ： 计算找到个数
    -i ： 忽略大小写
    -n ： 输出行号
    -v ： 反向选择，亦即显示出没有 搜寻字符串 内容的那一行
    --color=auto ：找到的关键字加颜色显示

awk

    $ awk '条件类型 1 {动作 1} 条件类型 2 {动作 2} ...' filename
    示例 2：/etc/passwd 文件第三个字段为 UID，对 UID 小于 10 的数据进行处理。
    
    $ cat /etc/passwd | awk 'BEGIN {FS=":"} $3 < 10 {print $1 "\t " $3}'
    root 0
    bin 1
    daemon 2
    sed
    
    示例 3：输出正在处理的行号，并显示每一行有多少字段
    
    $ last -n 5 | awk '{print $1 "\t lines: " NR "\t columns: " NF}'
    dmtsai lines: 1 columns: 10
    dmtsai lines: 2 columns: 10
    dmtsai lines: 3 columns: 10
    dmtsai lines: 4 columns: 10
    dmtsai lines: 5 columns: 9

sed:

    awk用于匹配每一行中的内容并打印
    而sed负责把文件内容重定向到输出，所以sed读取完文件并重定向到输出并且通过awk匹配这些内容并打印。
    
    他们俩经常搭配使用。

## linux指令实践和常见场景

## 查看进程状态

Linux进程状态(ps stat)之R、S、D、T、Z、X



    D    不可中断     Uninterruptible sleep (usually IO)
    R    正在运行，或在队列中的进程
    S    处于休眠状态
    T    停止或被追踪
    Z    僵尸进程
    W    进入内存交换（从内核2.6开始无效）
    X    死掉的进程


    <    高优先级
    N    低优先级
    L    有些页被锁进内存
    s    包含子进程
    +    位于后台的进程组；
    l    多线程，克隆线程  multi-threaded (using CLONE_THREAD, like NPTL pthreads do)


ps aux

![](https://img-blog.csdn.net/20180703221736541?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2E3MjQ4ODg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

## strace
strace用于跟踪程序执行过程中的系统调用，如跟踪test进程，只需要：

strace -p [test_pid] 或直接strace ./test

比如，跟踪pid为12345的进程中所有线程的read和write系统调用，输出字符串的长度限制为1024：

strace -s 1024 -f -e trace=read,write -p 12345

## tcpdump
tcpdump是Linux上的抓包工具，如抓取eth0网卡上的包，使用:

sudo tcpdump -i eth0

比如，抓取80端口的HTTP报文，以文本形式展示：

sudo tcpdump -i any port 80 -A
这样你就可以清楚看到GET、POST请求的内容了。

## nc

nc可以在Linux上开启TCP Server、TCP Client、UDP Server、UDP Client。

如在端口号12345上开启TCP Server和Client模拟TCP通信：

Server:  nc -l 127.0.0.1 12345
Client:  nc 127.0.0.1 12345 
在端口号12345上开启UDP Server和Client模拟TCP通信：

Server:  nc -ul 127.0.0.1 12345
Client:  nc -u 127.0.0.1 12345
Unix Socket通信示例:

Server:  nc -Ul /tmp/1.sock
Client:  nc -U /tmp/1.sock

## curl
curl用于模拟HTTP请求，在终端模拟请求时常用，如最基本的用法：

curl http://www.baidu.com

## lsof

lsof命令主要用法包括：

sudo lsof -i :[port] 查看端口占用进程信息，经常用于端口绑定失败时确认端口被哪个进程占用

sudo lsof -p [pid] 查看进程打开了哪些文件或套接字

## ss
Linux上的ss命令可以用于替换netstat，ss直接读取解析/proc/net下的统计信息，相比netstat遍历/proc下的每个PID目录，速度快很多。

## awk/sed
awk和sed在文本处理方面十分强大，其中，awk按列进行处理，sed按行进行处理。

如采用冒号分隔数据，输出第一列数据（$0代表行全部列数据，$1代表第一列，$2代表第二列...）

awk -F ":" '{print $1}'
在awk的结果基础上，结合sort、uniq和head等命令可以轻松完成频率统计等功能

查看文件的第100行到第200行：
sed -n '100,200p' log.txt
替换字符串中的特定子串
echo "int charset=gb2312 float"|sed "s/charset=gb2312/charset=UTF-8/g"
替换test文件每行匹配ab的部分为cd
sed -i 's/ab/cd/g' test

## vim
打开文件并跳到第10行

$ vim +10 filename.txt
打开文件跳到第一个匹配的行

$ vim +/search-term filename.txt
以只读模式打开文件

$ vim -R /etc/passwd

## crontab
查看某个用户的crontab入口

$ crontab -u john -l
设置一个每十分钟执行一次的计划任务

*/10 * * * * /home/ramesh/check-disk-space
更多示例：Linux Crontab: 15 Awesome Cron Job Examples

## service
service命令用于运行System V init脚本，这些脚本一般位于/etc/init.d文件下，这个命令可以直接运行这个文件夹里面的脚本，而不用加上路径

查看服务状态

$ service ssh status
查看所有服务状态

$ service --status-all
重启服务

$ service ssh restart

## free
这个命令用于显示系统当前内存的使用情况，包括已用内存、可用内存和交换内存的情况

默认情况下free会以字节为单位输出内存的使用量

    $ free
                 total       used       free     shared    buffers     cached
    Mem:       3566408    1580220    1986188          0     203988     902960
    -/+ buffers/cache:     473272    3093136
    Swap:      4000176          0    4000176

如果你想以其他单位输出内存的使用量，需要加一个选项，-g为GB，-m为MB，-k为KB，-b为字节

    $ free -g
                 total       used       free     shared    buffers     cached
    Mem:             3          1          1          0          0          0
    -/+ buffers/cache:          0          2
    Swap:            3          0          3

如果你想查看所有内存的汇总，请使用-t选项，使用这个选项会在输出中加一个汇总行
    
    ramesh@ramesh-laptop:~$ free -t
                 total       used       free     shared    buffers     cached
    Mem:       3566408    1592148    1974260          0     204260     912556
    -/+ buffers/cache:     475332    3091076
    Swap:      4000176          0    4000176
    Total:     7566584    1592148    5974436

## top

top命令会显示当前系统中占用资源最多的一些进程（默认以CPU占用率排序）如果你想改变排序方式，可以在结果列表中点击O（大写字母O）会显示所有可用于排序的列，这个时候你就可以选择你想排序的列

    Current Sort Field:  P  for window 1:Def
    Select sort field via field letter, type any other key to return
    
      a: PID        = Process Id              v: nDRT       = Dirty Pages count
      d: UID        = User Id                 y: WCHAN      = Sleeping in Function
      e: USER       = User Name               z: Flags      = Task Flags
      ........

如果只想显示某个特定用户的进程，可以使用-u选项

$ top -u oracle

## df
显示文件系统的磁盘使用情况，默认情况下df -k 将以字节为单位输出磁盘的使用量

$ df -k

    Filesystem           1K-blocks      Used Available Use% Mounted on
    /dev/sda1             29530400   3233104  24797232  12% /
    /dev/sda2            120367992  50171596  64082060  44% /home

使用-h选项可以以更符合阅读习惯的方式显示磁盘使用量

$ df -h

    Filesystem                  Size   Used  Avail Capacity  iused      ifree %iused  Mounted on
    /dev/disk0s2               232Gi   84Gi  148Gi    37% 21998562   38864868   36%   /
    devfs                      187Ki  187Ki    0Bi   100%      648          0  100%   /dev
    map -hosts                   0Bi    0Bi    0Bi   100%        0          0  100%   /net
    map auto_home                0Bi    0Bi    0Bi   100%        0          0  100%   /home
    /dev/disk0s4               466Gi   45Gi  421Gi    10%   112774  440997174    0%   /Volumes/BOOTCAMP
    //app@izenesoft.cn/public  2.7Ti  1.3Ti  1.4Ti    48% 

## kill
kill用于终止一个进程。一般我们会先用ps -ef查找某个进程得到它的进程号，然后再使用kill -9 进程号终止该进程。你还可以使用killall、pkill、xkill来终止进程

$ ps -ef | grep vim
ramesh    7243  7222  9 22:43 pts/2    00:00:00 vim

$ kill -9 7243

## mount
如果要挂载一个文件系统，需要先创建一个目录，然后将这个文件系统挂载到这个目录上

mkdir /u01
mount /dev/sdb1 /u01
也可以把它添加到fstab中进行自动挂载，这样任何时候系统重启的时候，文件系统都会被加载

/dev/sdb1 /u01 ext2 defaults 0 2
## chmod
chmod用于改变文件和目录的权限

给指定文件的属主和属组所有权限(包括读、写、执行)

$ chmod ug+rwx file.txt
删除指定文件的属组的所有权限

$ chmod g-rwx file.txt
修改目录的权限，以及递归修改目录下面所有文件和子目录的权限

$ chmod -R ug+rwx file.txt
更多示例：7 Chmod Command Examples for Beginners

## chown
chown用于改变文件属主和属组

同时将某个文件的属主改为oracle，属组改为db

$ chown oracle:dba dbora.sh
使用-R选项对目录和目录下的文件进行递归修改

$ chown -R oracle:dba /home/oracle

## ifconfig
ifconfig用于查看和配置Linux系统的网络接口

## uname
uname可以显示一些重要的系统信息，例如内核名称、主机名、内核版本号、处理器类型之类的信息 

## 实际场景问题

    1 cpu占用率
    
    top可以看
    ps看不了
    但是ps -aux可以看到各个线程的cpu和内存占用
    
    2 进程状态：
    
    ps -ef看不了
    ps aux可以看进程状态S R之类
    
    3 IO
    iostat查看io状态
    
    4网络
    netstat查看tcp连接状态和socket情况，
    
    ipconfig查看网络设备
    
    lsof可以查看端口使用情况
    
    5内存
    free

