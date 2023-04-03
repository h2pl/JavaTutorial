# Table of Contents

  * [1多线程的优点](#1多线程的优点)
    * [1.1资源利用率更好案例](#11资源利用率更好案例)
    * [1.2程序响应更快](#12程序响应更快)
  * [2多线程的代价](#2多线程的代价)
    * [2.1设计更复杂](#21设计更复杂)
    * [2.2上下文切换的开销](#22上下文切换的开销)
    * [2.3增加资源消耗](#23增加资源消耗)
  * [3竞态条件与临界区](#3竞态条件与临界区)
  * [**4**线程的运行与创建](#4线程的运行与创建)
  * [5线程的状态和优先级](#5线程的状态和优先级)


本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章同步发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《Java并发指南》其中一篇，本文大部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何全面深入地学习Java并发技术，从Java多线程基础，再到并发编程的基础知识，从Java并发包的入门和实战，再到JUC的源码剖析，一步步地学习Java并发编程，并上手进行实战，以便让你更完整地了解整个Java并发编程知识体系，形成自己的知识框架。

为了更好地总结和检验你的学习成果，本系列文章也会提供一些对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。
<!--more -->

## 1多线程的优点

1.  - 资源利用率更好 
2.  - 程序设计在某些情况下更简单 
3.  - 程序响应更快

### 1.1资源利用率更好案例

**方式1**
从磁盘读取一个文件需要5秒，处理一个文件需要2秒。处理两个文件则需要14秒

```
1    5秒读取文件A2    2秒处理文件A3    5秒读取文件B4    2秒处理文件B5    ---------------------6    总共需要14秒
```

**方式2**
从磁盘中读取文件的时候，大部分的CPU非常的空闲。它可以做一些别的事情。通过改变操作的顺序，就能够更好的使用CPU资源。看下面的顺序：

```
1    5秒读取文件A2    5秒读取文件B + 2秒处理文件A3    2秒处理文件B4    ---------------------5    总共需要12秒
```

**总结：多线程并发效率提高2秒**


### 1.2程序响应更快

设想一个服务器应用，它在某一个端口监听进来的请求。当一个请求到来时，它把请求传递给工作者线程(worker thread)，然后立刻返回去监听。而工作者线程则能够处理这个请求并发送一个回复给客户端。

     while(server is active){
            listenThread for request
           hand request to workerThread
        }


这种方式，服务端线程迅速地返回去监听。因此，更多的客户端能够发送请求给服务端。这个服务也变得响应更快。

## 2多线程的代价

### 2.1设计更复杂

多线程一般都复杂。在多线程访问共享数据的时候，这部分代码需要特别的注意。线程之间的交互往往非常复杂。不正确的线程同步产生的错误非常难以被发现，并且重现以修复。

### 2.2上下文切换的开销

**上下文切换**当CPU从执行一个线程切换到执行另外一个线程的时候，它需要先存储当前线程的本地的数据，程序指针等，然后载入另一个线程的本地数据，程序指针等，最后才开始执行。

CPU会在一个上下文中执行一个线程，然后切换到另外一个上下文中执行另外一个线程。

上下文切换并不廉价。如果没有必要，应该减少上下文切换的发生。

### 2.3增加资源消耗

**每个线程需要消耗的资源：**

CPU，内存（维持它本地的堆栈），操作系统资源（管理线程） 

## 3竞态条件与临界区

当多个线程竞争同一资源时，如果对资源的访问顺序敏感，就称存在竞态条件。导致竞态条件发生的代码区称作临界区。

**多线程同时执行下面的代码可能会出错：**


    public class Counter {
    	protected long count = 0;
     
    	public void add(long value) {
    		this.count = this.count + value;
    	}
    }
想象下线程A和B同时执行同一个Counter对象的add()方法，我们无法知道操作系统何时会在两个线程之间切换。JVM并不是将这段代码视为单条指令来执行的，而是按照下面的顺序


    从内存获取 this.count 的值放到寄存器
    将寄存器中的值增加value
    将寄存器中的值写回内存
     
     
     
    观察线程A和B交错执行会发生什么
     
    	this.count = 0;
       A:	读取 this.count 到一个寄存器 (0)
       B:	读取 this.count 到一个寄存器 (0)
       B: 	将寄存器的值加2
       B:	回写寄存器值(2)到内存. this.count 现在等于 2
       A:	将寄存器的值加3


由于两个线程是交叉执行的，两个线程从内存中读出的初始值都是0。然后各自加了2和3，并分别写回内存。最终的值可能并不是期望的5，而是最后写回内存的那个线程的值，上面例子中最后写回内存可能是线程A，也可能是线程B
 
## **4**线程的运行与创建

**Java 创建线程对象有两种方法：**

*   继承 Thread 类创建线程对象
*   实现 Runnable 接口类创建线程对象

**注意：**

在java中，每次程序运行至少启动2个线程。一个是main线程，一个是垃圾收集线程。因为每当使用java命令执行一个类的时候，实际上都会启动一个jvm，每一个jvm实际上就是在操作系统中启动了一个进程。

![](https://img-blog.csdnimg.cn/20181031104947230.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pob3U5MjA3ODYzMTI=,size_16,color_FFFFFF,t_70)

## 5线程的状态和优先级

**线程优先级**1 到 10 ，其中 1 是最低优先级，10 是最高优先级。 

**状态**

*   new（新建）
*   runnnable（可运行）
*   blocked（阻塞）
*   waiting（等待）
*   time waiting （定时等待）
*   terminated（终止）

**状态转换**

**![](https://img-blog.csdnimg.cn/20181031105627437.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3pob3U5MjA3ODYzMTI=,size_16,color_FFFFFF,t_70)**

**线程状态流程如下：**

*   线程创建后，进入 new 状态
*   调用 start 或者 run 方法，进入 runnable 状态
*   JVM 按照线程优先级及时间分片等执行 runnable 状态的线程。开始执行时，进入 running 状态
*   如果线程执行 sleep、wait、join，或者进入 IO 阻塞等。进入 wait 或者 blocked 状态
*   线程执行完毕后，线程被线程队列移除。最后为 terminated 状态

**代码**


    public class MyThreadInfo extends Thread {
     
    	@Override // 可以省略
    	public void run() {
    		System.out.println("run");
    		// System.exit(1);
    	}
     
    	public static void main(String[] args) {
    		MyThreadInfo thread = new MyThreadInfo();
    		thread.start();
     
    		System.out.println("线程唯一标识符：" + thread.getId());
    		System.out.println("线程名称：" + thread.getName());
    		System.out.println("线程状态：" + thread.getState());
    		System.out.println("线程优先级：" + thread.getPriority());
    	}
    }
     
    结果：
    线程唯一标识符：9
    线程名称：Thread-0
    run
    线程状态：RUNNABLE




