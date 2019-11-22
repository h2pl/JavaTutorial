# Table of Contents

  * [Java中的锁机制及Lock类](#java中的锁机制及lock类)
    * [锁的释放-获取建立的happens before 关系](#锁的释放-获取建立的happens-before-关系)
    * [锁释放和获取的内存语义](#锁释放和获取的内存语义)
    * [锁内存语义的实现](#锁内存语义的实现)
            * [LOCK_IF_MP(mp) __asm cmp mp, 0  \](#lock_if_mpmp-__asm-cmp-mp-0--)
  * [concurrent包的实现](#concurrent包的实现)
  * [synchronized实现原理](#synchronized实现原理)
    * [****1、实现原理****](#1、实现原理)
    * [**2、Java对象头**](#2、java对象头)
    * [**3、Monitor**](#3、monitor)
    * [**4、锁优化**](#4、锁优化)
    * [**5、自旋锁**](#5、自旋锁)
    * [**6、适应自旋锁**](#6、适应自旋锁)
    * [**7、锁消除**](#7、锁消除)
    * [**8、锁粗化**](#8、锁粗化)
    * [**9、轻量级锁**](#9、轻量级锁)
    * [**10、偏向锁**](#10、偏向锁)
  * [**11、重量级锁**](#11、重量级锁)
  * [参考资料](#参考资料)


**本文转载自并发编程网，侵删**

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

## Java中的锁机制及Lock类

### 锁的释放-获取建立的happens before 关系

锁是java并发编程中最重要的同步机制。锁除了让临界区互斥执行外，还可以让释放锁的线程向获取同一个锁的线程发送消息。

下面是锁释放-获取的示例代码：
    class MonitorExample {
        int a = 0;
     
        public synchronized void writer() {  //1
            a++;                             //2
        }                                    //3
     
        public synchronized void reader() {  //4
            int i = a;                       //5
            ……
        }                                    //6
    }
根据程序次序规则，1 happens before 2, 2 happens before 3; 4 happens before 5, 5 happens before 6。假设线程A执行writer()方法，随后线程B执行reader()方法。根据happens before规则，这个过程包含的happens before 关系可以分为两类：

1.  根据监视器锁规则，3 happens before 4。
2.  根据happens before 的传递性，2 happens before 5。

上述happens before 关系的图形化表现形式如下：

![](http://blog.itpub.net/ueditor/php/upload/image/20190811/1565506398748401.png)

在上图中，每一个箭头链接的两个节点，代表了一个happens before 关系。黑色箭头表示程序顺序规则；橙色箭头表示监视器锁规则；蓝色箭头表示组合这些规则后提供的happens before保证。

上图表示在线程A释放了锁之后，随后线程B获取同一个锁。在上图中，2 happens before 5。因此，线程A在释放锁之前所有可见的共享变量，在线程B获取同一个锁之后，将立刻变得对B线程可见。

### 锁释放和获取的内存语义

当线程释放锁时，JMM会把该线程对应的本地内存中的共享变量刷新到主内存中。以上面的MonitorExample程序为例，A线程释放锁后，共享数据的状态示意图如下：

![](http://blog.itpub.net/ueditor/php/upload/image/20190811/1565506402638665.png)

当线程获取锁时，JMM会把该线程对应的本地内存置为无效。从而使得被监视器保护的临界区代码必须要从主内存中去读取共享变量。下面是锁获取的状态示意图：

![](http://blog.itpub.net/ueditor/php/upload/image/20190811/1565506405253375.png)

对比锁释放-获取的内存语义与volatile写-读的内存语义，可以看出：锁释放与volatile写有相同的内存语义；锁获取与volatile读有相同的内存语义。

下面对锁释放和锁获取的内存语义做个总结：

*   线程A释放一个锁，实质上是线程A向接下来将要获取这个锁的某个线程发出了（线程A对共享变量所做修改的）消息。
*   线程B获取一个锁，实质上是线程B接收了之前某个线程发出的（在释放这个锁之前对共享变量所做修改的）消息。
*   线程A释放锁，随后线程B获取这个锁，这个过程实质上是线程A通过主内存向线程B发送消息。

### 锁内存语义的实现

本文将借助ReentrantLock的源代码，来分析锁内存语义的具体实现机制。

请看下面的示例代码：

    class ReentrantLockExample {
    int a = 0;
    ReentrantLock lock = new ReentrantLock();
     
    public void writer() {
        lock.lock();         //获取锁
        try {
            a++;
        } finally {
            lock.unlock();  //释放锁
        }
    }
     
    public void reader () {
        lock.lock();        //获取锁
        try {
            int i = a;
            ……
        } finally {
            lock.unlock();  //释放锁
        }
    }
    }
    
在ReentrantLock中，调用lock()方法获取锁；调用unlock()方法释放锁。

ReentrantLock的实现依赖于java同步器框架AbstractQueuedSynchronizer（本文简称之为AQS）。AQS使用一个整型的volatile变量（命名为state）来维护同步状态，马上我们会看到，这个volatile变量是ReentrantLock内存语义实现的关键。 下面是ReentrantLock的类图（仅画出与本文相关的部分）：

![](http://blog.itpub.net/ueditor/php/upload/image/20190811/1565506414141304.png)

ReentrantLock分为公平锁和非公平锁，我们首先分析公平锁。

使用公平锁时，加锁方法lock()的方法调用轨迹如下：

1.  ReentrantLock : lock()
2.  FairSync : lock()
3.  AbstractQueuedSynchronizer : acquire(int arg)
4.  ReentrantLock : tryAcquire(int acquires)

在第4步真正开始加锁，下面是该方法的源代码：


    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();   //获取锁的开始，首先读volatile变量state
        if (c == 0) {
            if (isFirst(current) &&
                compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)  
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }


从上面源代码中我们可以看出，加锁方法首先读volatile变量state。

在使用公平锁时，解锁方法unlock()的方法调用轨迹如下：

1.  ReentrantLock : unlock()
2.  AbstractQueuedSynchronizer : release(int arg)
3.  Sync : tryRelease(int releases)

在第3步真正开始释放锁，下面是该方法的源代码：


    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);           //释放锁的最后，写volatile变量state
        return free;
    }


从上面的源代码我们可以看出，在释放锁的最后写volatile变量state。

公平锁在释放锁的最后写volatile变量state；在获取锁时首先读这个volatile变量。根据volatile的happens-before规则，释放锁的线程在写volatile变量之前可见的共享变量，在获取锁的线程读取同一个volatile变量后将立即变的对获取锁的线程可见。

现在我们分析非公平锁的内存语义的实现。

非公平锁的释放和公平锁完全一样，所以这里仅仅分析非公平锁的获取。

使用公平锁时，加锁方法lock()的方法调用轨迹如下：

1.  ReentrantLock : lock()
2.  NonfairSync : lock()
3.  AbstractQueuedSynchronizer : compareAndSetState(int expect, int update)

在第3步真正开始加锁，下面是该方法的源代码：

<pre name="code">protected final boolean compareAndSetState(int expect, int update) {return unsafe.compareAndSwapInt(this, stateOffset, expect, update);}</pre>

该方法以原子操作的方式更新state变量，本文把java的compareAndSet()方法调用简称为CAS。JDK文档对该方法的说明如下：如果当前状态值等于预期值，则以原子方式将同步状态设置为给定的更新值。此操作具有 volatile 读和写的内存语义。

这里我们分别从编译器和处理器的角度来分析,CAS如何同时具有volatile读和volatile写的内存语义。

前文我们提到过，编译器不会对volatile读与volatile读后面的任意内存操作重排序；编译器不会对volatile写与volatile写前面的任意内存操作重排序。组合这两个条件，意味着为了同时实现volatile读和volatile写的内存语义，编译器不能对CAS与CAS前面和后面的任意内存操作重排序。

下面我们来分析在常见的intel x86处理器中，CAS是如何同时具有volatile读和volatile写的内存语义的。

下面是sun.misc.Unsafe类的compareAndSwapInt()方法的源代码：

    protected final boolean compareAndSetState(int expect, int update) {
        return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
    }

可以看到这是个本地方法调用。这个本地方法在openjdk中依次调用的c++代码为：unsafe.cpp，atomic.cpp和atomicwindowsx86.inline.hpp。这个本地方法的最终实现在openjdk的如下位置：openjdk-7-fcs-src-b147-27jun2011\openjdk\hotspot\src\oscpu\windowsx86\vm\ atomicwindowsx86.inline.hpp（对应于windows操作系统，X86处理器）。下面是对应于intel x86处理器的源代码的片段：

    // Adding a lock prefix to an instruction on MP machine
    // VC++ doesn't like the lock prefix to be on a single line
    // so we can't insert a label after the lock prefix.
    // By emitting a lock prefix, we can define a label after it.
    #define LOCK_IF_MP(mp) __asm cmp mp, 0  \
                           __asm je L0      \
                           __asm _emit 0xF0 \
                           __asm L0:
     
    inline jint     Atomic::cmpxchg    (jint     exchange_value, volatile jint*     dest, jint     compare_value) {
      // alternative for InterlockedCompareExchange
      int mp = os::is_MP();
      __asm {
        mov edx, dest
        mov ecx, exchange_value
        mov eax, compare_value
        LOCK_IF_MP(mp)
        cmpxchg dword ptr [edx], ecx
      }
    }

如上面源代码所示，程序会根据当前处理器的类型来决定是否为cmpxchg指令添加lock前缀。如果程序是在多处理器上运行，就为cmpxchg指令加上lock前缀（lock cmpxchg）。反之，如果程序是在单处理器上运行，就省略lock前缀（单处理器自身会维护单处理器内的顺序一致性，不需要lock前缀提供的内存屏障效果）。

intel的手册对lock前缀的说明如下：

1.  确保对内存的读-改-写操作原子执行。在Pentium及Pentium之前的处理器中，带有lock前缀的指令在执行期间会锁住总线，使得其他处理器暂时无法通过总线访问内存。很显然，这会带来昂贵的开销。
 
2.  从Pentium 4，Intel Xeon及P6处理器开始，intel在原有总线锁的基础上做了一个很有意义的优化：如果要访问的内存区域（area of memory）在lock前缀指令执行期间已经在处理器内部的缓存中被锁定（即包含该内存区域的缓存行当前处于独占或以修改状态），并且该内存区域被完全包含在单个缓存行（cache line）中，那么处理器将直接执行该指令。

3.  由于在指令执行期间该缓存行会一直被锁定，其它处理器无法读/写该指令要访问的内存区域，因此能保证指令执行的原子性。这个操作过程叫做缓存锁定（cache locking），缓存锁定将大大降低lock前缀指令的执行开销，但是当多处理器之间的竞争程度很高或者指令访问的内存地址未对齐时，仍然会锁住总线。

4.  禁止该指令与之前和之后的读和写指令重排序。
  
5.  把写缓冲区中的所有数据刷新到内存中。

上面的第2点和第3点所具有的内存屏障效果，足以同时实现volatile读和volatile写的内存语义。

经过上面的这些分析，现在我们终于能明白为什么JDK文档说CAS同时具有volatile读和volatile写的内存语义了。

现在对公平锁和非公平锁的内存语义做个总结：

*   公平锁和非公平锁释放时，最后都要写一个volatile变量state。
*   公平锁获取时，首先会去读这个volatile变量。
*   非公平锁获取时，首先会用CAS更新这个volatile变量,这个操作同时具有volatile读和volatile写的内存语义。

从本文对ReentrantLock的分析可以看出，锁释放-获取的内存语义的实现至少有下面两种方式：

1.  利用volatile变量的写-读所具有的内存语义。
2.  利用CAS所附带的volatile读和volatile写的内存语义。

## concurrent包的实现

由于java的CAS同时具有 volatile 读和volatile写的内存语义，因此Java线程之间的通信现在有了下面四种方式：

1.  A线程写volatile变量，随后B线程读这个volatile变量。
2.  A线程写volatile变量，随后B线程用CAS更新这个volatile变量。
3.  A线程用CAS更新一个volatile变量，随后B线程用CAS更新这个volatile变量。
4.  A线程用CAS更新一个volatile变量，随后B线程读这个volatile变量。

Java的CAS会使用现代处理器上提供的高效机器级别原子指令，这些原子指令以原子方式对内存执行读-改-写操作，这是在多处理器中实现同步的关键（从本质上来说，能够支持原子性读-改-写指令的计算机器，是顺序计算图灵机的异步等价机器，因此任何现代的多处理器都会去支持某种能对内存执行原子性读-改-写操作的原子指令）。同时，volatile变量的读/写和CAS可以实现线程之间的通信。把这些特性整合在一起，就形成了整个concurrent包得以实现的基石。如果我们仔细分析concurrent包的源代码实现，会发现一个通用化的实现模式：

1.  首先，声明共享变量为volatile；
2.  然后，使用CAS的原子条件更新来实现线程之间的同步；
3.  同时，配合以volatile的读/写和CAS所具有的volatile读和写的内存语义来实现线程之间的通信。

AQS，非阻塞数据结构和原子变量类（java.util.concurrent.atomic包中的类），这些concurrent包中的基础类都是使用这种模式来实现的，而concurrent包中的高层类又是依赖于这些基础类来实现的。从整体来看，concurrent包的实现示意图如下：

![](http://blog.itpub.net/ueditor/php/upload/image/20190811/1565506416227137.png)

## synchronized实现原理

转自：https://blog.csdn.net/chenssy/article/details/54883355


记得刚刚开始学习Java的时候，一遇到多线程情况就是synchronized。对于当时的我们来说，synchronized是如此的神奇且强大。我们赋予它一个名字“同步”，也成为我们解决多线程情况的良药，百试不爽。但是，随着学习的深入，我们知道synchronized是一个重量级锁，相对于Lock，它会显得那么笨重，以至于我们认为它不是那么的高效，并慢慢抛弃它。 

　　诚然，随着Javs SE 1.6对synchronized进行各种优化后，synchronized不会显得那么重。

 　　下面跟随LZ一起来探索**synchronized的实现机制、Java是如何对它进行了优化、锁优化机制、锁的存储结构和升级过程。**

### ****1、实现原理****

 　　synchronized可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性。

Java中每一个对象都可以作为锁，这是synchronized实现同步的基础：

1.  **普通同步方法，锁是当前实例对象；**

2.  **静态同步方法，锁是当前类的class对象；**

3.  **同步方法块，锁是括号里面的对象。**

　　当一个线程访问同步代码块时，它首先是需要得到锁才能执行同步代码，**当退出或者抛出异常时必须要释放锁，那么它是如何来实现这个机制的呢？**

 我们先看一段简单的代码：



<pre>public class SynchronizedTest{ public synchronized void test1(){

　　} public void test2(){
　　　　synchronized(this){

       }
    }
}</pre>



**利用Javap工具查看生成的class文件信息来分析Synchronize的实现：**

![](https://images2018.cnblogs.com/blog/1169376/201807/1169376-20180726111452385-496687429.png)

　　从上面可以看出，同步代码块是使用monitorenter和monitorexit指令实现的，同步方法（在这看不出来需要看JVM底层实现）依靠的是方法修饰符上的ACCSYNCHRONIZED实现。

**同步代码块：**

　　monitorenter指令插入到同步代码块的开始位置，monitorexit指令插入到同步代码块的结束位置，JVM需要保证每一个monitorenter都有一个monitorexit与之相对应。任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁；

**同步方法**

　　synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的**accessflags字段中的synchronized标志位置1**，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象。

(摘自：http://www.cnblogs.com/javaminer/p/3889023.html)

下面我们来继续分析，但是在深入之前我们需要了解两个重要的概念：**Java对象头、Monitor。**

 **Java对象头、monitor：Java对象头和monitor是实现synchronized的基础！下面就这两个概念来做详细介绍。**

### **2、Java对象头**

synchronized用的锁是存在Java对象头里的，那么什么是Java对象头呢？

Hotspot虚拟机的对象头主要包括两部分数据：**Mark Word（标记字段）、Klass Pointer（类型指针）**。其中**Klass Point是是对象指向它的类元数据的指针**，虚拟机通过这个指针来确定这个对象是哪个类的实例，**Mark Word用于存储对象自身的运行时数据，它是实现轻量级锁和偏向锁的关键。**

所以下面将重点阐述。

*   **Mark Word**

    Mark Word用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。Java对象头一般占有两个机器码（在32位虚拟机中，1个机器码等于4字节，也就是32bit），但是如果对象是数组类型，则需要三个机器码，因为JVM虚拟机可以通过Java对象的元数据信息确定Java对象的大小，但是无法从数组的元数据来确认数组的大小，所以用一块来记录数组长度。

　　下图是Java对象头的存储结构（32位虚拟机）：

　　![](https://images2018.cnblogs.com/blog/1169376/201807/1169376-20180726111729662-408733474.png)

　　对象头信息是与对象自身定义的数据无关的额外存储成本，但是考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据，它会根据对象的状态复用自己的存储空间，也就是说，Mark Word会随着程序的运行发生变化，变化状态如下（32位虚拟机）：

![](https://images2018.cnblogs.com/blog/1169376/201807/1169376-20180726111757632-1279497345.png)

简单介绍了Java对象头，我们下面再看Monitor。

### **3、Monitor**

什么是Monitor？

　　我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。 

　　与一切皆对象一样，所有的Java对象是天生的Monitor，**每一个Java对象都有成为Monitor的潜质**，因为在Java的设计中 ，每一个Java对象自打娘胎里出来**就带了一把看不见的锁，它叫做内部锁或者Monitor锁**。 

　　Monitor 是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联（对象头的MarkWord中的LockWord指向monitor的起始地址），**同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用**。　　

其结构如下：

 ![](https://images2018.cnblogs.com/blog/1169376/201807/1169376-20180726111912718-1995783204.png)

*   **Owner：**初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL。

*   **EntryQ：**关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程。

*   **RcThis：**表示blocked或waiting在该monitor record上的所有线程的个数。

*   **Nest：**用来实现重入锁的计数。HashCode:保存从对象头拷贝过来的HashCode值（可能还包含GC age）。

*   **Candidate：用来避免不必要的阻塞或等待线程唤醒**，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。

    Candidate只有两种可能的值0表示没有需要唤醒的线程1表示要唤醒一个继任线程来竞争锁。

我们知道synchronized是重量级锁，效率不怎么滴，同时这个观念也一直存在我们脑海里，不过在**JDK 1.6中对synchronize的实现进行了各种优化，使得它显得不是那么重了，那么JVM采用了那些优化手段呢？**

### **4、锁优化**

　　JDK1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。 

**　　锁主要存在四中状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态。**他们会随着竞争的激烈而逐渐升级。注意锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。

### **5、自旋锁**

　　线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。同时我们发现在许多应用上面，**对象锁的锁状态只会持续很短一段时间**，**为了这一段很短的时间频繁地阻塞和唤醒线程**是非常不值得的。

所以引入自旋锁。 

何谓自旋锁？

**　　所谓自旋锁，就是让该线程等待一段时间，不会被立即挂起（就是不让前来获取该锁（已被占用）的线程立即阻塞），看持有锁的线程是否会很快释放锁。**

**怎么等待呢？**

执行一段无意义的循环即可（自旋）。

　　自旋等待不能替代阻塞，先不说对处理器数量的要求（多核，貌似现在没有单核的处理器了），虽然它可以避免线程切换带来的开销，但是它占用了处理器的时间。如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好；反之，自旋的线程就会白白消耗掉处理的资源，它不会做任何有意义的工作，典型的占着茅坑不拉屎，这样反而会带来性能上的浪费。

　　所以说，自旋等待的时间（自旋的次数）必须要有一个限度，如果自旋超过了定义的时间仍然没有获取到锁，则应该被挂起。自旋锁在JDK 1.4.2中引入，默认关闭，但是可以使用-XX:+UseSpinning开开启，在JDK1.6中默认开启。同时自旋的默认次数为10次，可以通过参数-XX:PreBlockSpin来调整。

　　如果通过参数-XX:preBlockSpin来调整自旋锁的自旋次数，会带来诸多不便。假如我将参数调整为10，但是系统很多线程都是等你刚刚退出的时候就释放了锁（假如你多自旋一两次就可以获取锁），你是不是很尴尬？于是JDK1.6引入自适应的自旋锁，让虚拟机会变得越来越聪明。

### **6、适应自旋锁**

　　JDK 1.6引入了更加聪明的自旋锁，即自适应自旋锁。所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。

　　它怎么做呢？

　　线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。**有了自适应自旋锁，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测会越来越准确，虚拟机会变得越来越聪明。** 

### **7、锁消除**

　　为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制，但是在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。锁消除的依据是逃逸分析的数据支持。 

**　　如果不存在竞争，为什么还需要加锁呢？**

　　所以锁消除可以节省毫无意义的请求锁的时间。变量是否逃逸，对于虚拟机来说需要使用数据流分析来确定，但是对于我们程序员来说这还不清楚么？我们会在明明知道不存在数据竞争的代码块前加上同步吗？但是有时候程序并不是我们所想的那样？

　　我们虽然没有显示使用锁，但是我们在使用一些JDK的内置API时，如StringBuffer、Vector、HashTable等，这个时候会存在隐形的加锁操作。

**　　比如StringBuffer的append()方法，Vector的add()方法：**


<pre>public void vectorTest(){
    Vector<String> vector = new Vector<String>(); for(int i = 0 ; i < 10 ; i++){
        vector.add(i + "");
     } 

    System.out.println(vector);
}</pre>


在运行这段代码时，JVM可以明显检测到变量vector没有逃逸出方法vectorTest()之外，所以JVM可以大胆地将vector内部的加锁操作消除。

### **8、锁粗化**

　　我们知道在使用同步锁的时候，需要让同步块的作用范围尽可能小，仅在共享数据的实际作用域中才进行同步。这样做的目的是为了使需要同步的操作数量尽可能缩小，如果存在锁竞争，那么等待锁的线程也能尽快拿到锁。 

　　在大多数的情况下，上述观点是正确的，LZ也一直坚持着这个观点。但是如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入锁粗化的概念。 

　　**那什么是锁粗化？**

**就是将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。**

　　**如上面实例：vector每次add的时候都需要加锁操作，JVM检测到对同一个对象（vector）连续加锁、解锁操作，会合并一个更大范围的加锁、解锁操作，即加锁解锁操作会移到for循环之外。**

### **9、轻量级锁**

　　引入轻量级锁的主要目的是在多没有多线程竞争的前提下，**减少传统的重量级锁使用操作系统互斥量产生的性能消耗**。

**当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁，其步骤如下：****获取锁。**

1.  判断当前对象是否处于无锁状态（hashcode、0、01），若是，则JVM首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了一个Displaced前缀，即Displaced Mark Word）；否则执行步骤（3）；

2.  JVM利用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指正，如果成功表示竞争到锁，则将锁标志位变成00（表示此对象处于轻量级锁状态），执行同步操作；如果失败则执行步骤（3）；

3.  判断当前对象的Mark Word是否指向当前线程的栈帧，如果是则表示当前线程已经持有当前对象的锁，则直接执行同步代码块；否则只能说明该锁对象已经被其他线程抢占了，这时轻量级锁需要膨胀为重量级锁，锁标志位变成10，后面等待的线程将会进入阻塞状态； 

**释放锁轻量级锁的释放也是通过CAS操作来进行的，主要步骤如下：**

1.  取出在获取轻量级锁保存在Displaced Mark Word中的数据；

2.  用CAS操作将取出的数据替换当前对象的Mark Word中，如果成功，则说明释放锁成功，否则执行（3）；

3.  如果CAS操作替换失败，说明有其他线程尝试获取该锁，则需要在释放锁的同时需要唤醒被挂起的线程。

　　轻量级锁能提升程序同步性能的依据是“对于绝大部分的锁，在整个同步周期内都是不存在竞争的”，这是一个经验数据。轻量级锁在当前线程的栈帧中建立一个名为锁记录的空间，用于存储锁对象目前的指向和状态。如果没有竞争，轻量级锁使用CAS操作避免了使用互斥量的开销，但如果存在锁竞争，除了互斥量的开销外，还额外发生了CAS操作，因此**在有竞争的情况下，轻量级锁会比传统的重量级锁更慢。**

**什么是CAS操作？**

compare and swap,CAS操作需要输入两个数值，一个旧值（期望操作前的值）和一个新值，在操作期间先比较旧值有没有发生变化，如果没有发生变化，才交换成新值，发生了变化则不交换。

CAS详解：https://mp.weixin.qq.com/s__biz=MzIxMjE5MTE1Nw==&mid=2653192625&idx=1&sn=cbabbd806e4874e8793332724ca9d454&chksm=8c99f36bbbee7a7d169581dedbe09658d0b0edb62d2cbc9ba4c40f706cb678c7d8c768afb666&scene=21#wechat_redirect

https://blog.csdn.net/qq_35357656/article/details/78657373

下图是轻量级锁的获取和释放过程：

![](https://images2018.cnblogs.com/blog/1169376/201807/1169376-20180726104054182-532040033.png)

### **10、偏向锁**

　　引入偏向锁主要目的是：为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径。上面提到了轻量级锁的加锁解锁操作是需要依赖多次CAS原子指令的。**那么偏向锁是如何来减少不必要的CAS操作呢**？我们可以查看Mark work的结构就明白了。

**只需要检查是否为偏向锁、锁标识为以及ThreadID即可，处理流程如下：****获取锁。**

1.  检测Mark Word是否为可偏向状态，即是否为偏向锁1，锁标识位为01；

2.  若为可偏向状态，则测试线程ID是否为当前线程ID，如果是，则执行步骤（5），否则执行步骤（3）；

3.  如果线程ID不为当前线程ID，则通过CAS操作竞争锁，竞争成功，则将Mark Word的线程ID替换为当前线程ID，否则执行线程（4）；

4.  通过CAS竞争锁失败，证明当前存在多线程竞争情况，当到达全局安全点，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码块；

5.  执行同步代码块。 

释放锁偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。偏向锁的撤销需要等待全局安全点（这个时间点是上没有正在执行的代码）。

其步骤如下：

1.  **暂停拥有偏向锁的线程，判断锁对象石是否还处于被锁定状态；**

2.  **撤销偏向苏，恢复到无锁状态（01）或者轻量级锁的状态。**

 下图是偏向锁的获取和释放流程：

![](https://images2018.cnblogs.com/blog/1169376/201807/1169376-20180726110935396-89753255.png)

## **11、重量级锁**

　　重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。

## 参考资料

1.  周志明：《深入理解Java虚拟机》
2.  方腾飞：《Java并发编程的艺术》
3.  Java中synchronized的实现原理与应用
