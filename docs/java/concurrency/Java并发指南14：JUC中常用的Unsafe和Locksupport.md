# Table of Contents

  * [前言](#前言)
  * [Unsafe类是啥？](#unsafe类是啥？)
  * [为什么叫Unsafe？](#为什么叫unsafe？)
  * [JAVA高并发—LockSupport的学习及简单使用](#java高并发locksupport的学习及简单使用)


本文转自网络，侵删

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
## 前言

最近在看Java并发包的源码，发现了神奇的Unsafe类，仔细研究了一下，在这里跟大家分享一下。

![](http://p1.pstatp.com/large/c5e00061c7c63b28923)

Unsafe类是在sun.misc包下，不属于Java标准。但是很多Java的基础类库，包括一些被广泛使用的高性能开发库都是基于Unsafe类开发的，比如Netty、Cassandra、Hadoop、Kafka等。Unsafe类在提升Java运行效率，增强Java语言底层操作能力方面起了很大的作用。

![](http://p3.pstatp.com/large/ca4000030d1f375caf1)

![](http://p3.pstatp.com/large/ca4000030319e3adb8f)

Unsafe类使Java拥有了像C语言的指针一样操作内存空间的能力，同时也带来了指针的问题。过度的使用Unsafe类会使得出错的几率变大，因此Java官方并不建议使用的，官方文档也几乎没有。Oracle正在计划从Java 9中去掉Unsafe类，如果真是如此影响就太大了。

![](http://p3.pstatp.com/large/c5e00061cf0b59ac2ac)

通常我们最好也不要使用Unsafe类，除非有明确的目的，并且也要对它有深入的了解才行。要想使用Unsafe类需要用一些比较tricky的办法。Unsafe类使用了单例模式，需要通过一个静态方法getUnsafe()来获取。但Unsafe类做了限制，如果是普通的调用的话，它会抛出一个SecurityException异常；只有由主类加载器加载的类才能调用这个方法。其源码如下：


    1 public static Unsafe getUnsafe() {
    2     Class var0 = Reflection.getCallerClass();
    3     if(!VM.isSystemDomainLoader(var0.getClassLoader())) {
    4         throw new SecurityException("Unsafe");
    5     } else {
    6         return theUnsafe;
    7     }
    8 }

网上也有一些办法来用主类加载器加载用户代码，比如设置bootclasspath参数。但更简单方法是利用Java反射，方法如下：

    1 Field f = Unsafe.class.getDeclaredField("theUnsafe");
    2 f.setAccessible(true);
    3 Unsafe unsafe = (Unsafe) f.get(null);

获取到Unsafe实例之后，我们就可以为所欲为了。Unsafe类提供了以下这些功能：

一、内存管理。包括分配内存、释放内存等。

该部分包括了allocateMemory（分配内存）、reallocateMemory（重新分配内存）、copyMemory（拷贝内存）、freeMemory（释放内存 ）、getAddress（获取内存地址）、addressSize、pageSize、getInt（获取内存地址指向的整数）、getIntVolatile（获取内存地址指向的整数，并支持volatile语义）、putInt（将整数写入指定内存地址）、putIntVolatile（将整数写入指定内存地址，并支持volatile语义）、putOrderedInt（将整数写入指定内存地址、有序或者有延迟的方法）等方法。getXXX和putXXX包含了各种基本类型的操作。

利用copyMemory方法，我们可以实现一个通用的对象拷贝方法，无需再对每一个对象都实现clone方法，当然这通用的方法只能做到对象浅拷贝。

二、非常规的对象实例化。

allocateInstance()方法提供了另一种创建实例的途径。通常我们可以用new或者反射来实例化对象，使用allocateInstance()方法可以直接生成对象实例，且无需调用构造方法和其它初始化方法。

这在对象反序列化的时候会很有用，能够重建和设置final字段，而不需要调用构造方法。

三、操作类、对象、变量。

这部分包括了staticFieldOffset（静态域偏移）、defineClass（定义类）、defineAnonymousClass（定义匿名类）、ensureClassInitialized（确保类初始化）、objectFieldOffset（对象域偏移）等方法。

通过这些方法我们可以获取对象的指针，通过对指针进行偏移，我们不仅可以直接修改指针指向的数据（即使它们是私有的），甚至可以找到JVM已经认定为垃圾、可以进行回收的对象。

四、数组操作。

这部分包括了arrayBaseOffset（获取数组第一个元素的偏移地址）、arrayIndexScale（获取数组中元素的增量地址）等方法。arrayBaseOffset与arrayIndexScale配合起来使用，就可以定位数组中每个元素在内存中的位置。

由于Java的数组最大值为Integer.MAX_VALUE，使用Unsafe类的内存分配方法可以实现超大数组。实际上这样的数据就可以认为是C数组，因此需要注意在合适的时间释放内存。

五、多线程同步。包括锁机制、CAS操作等。

这部分包括了monitorEnter、tryMonitorEnter、monitorExit、compareAndSwapInt、compareAndSwap等方法。

其中monitorEnter、tryMonitorEnter、monitorExit已经被标记为deprecated，不建议使用。

Unsafe类的CAS操作可能是用的最多的，它为Java的锁机制提供了一种新的解决办法，比如AtomicInteger等类都是通过该方法来实现的。compareAndSwap方法是原子的，可以避免繁重的锁机制，提高代码效率。这是一种乐观锁，通常认为在大部分情况下不出现竞态条件，如果操作失败，会不断重试直到成功。

六、挂起与恢复。

这部分包括了park、unpark等方法。

将一个线程进行挂起是通过park方法实现的，调用 park后，线程将一直阻塞直到超时或者中断等条件出现。unpark可以终止一个挂起的线程，使其恢复正常。整个并发框架中对线程的挂起操作被封装在 LockSupport类中，LockSupport类中有各种版本pack方法，但最终都调用了Unsafe.park()方法。

七、内存屏障。

这部分包括了loadFence、storeFence、fullFence等方法。这是在Java 8新引入的，用于定义内存屏障，避免代码重排序。

loadFence() 表示该方法之前的所有load操作在内存屏障之前完成。同理storeFence()表示该方法之前的所有store操作在内存屏障之前完成。fullFence()表示该方法之前的所有load、store操作在内存屏障之前完成。

## Unsafe类是啥？

Java最初被设计为一种安全的受控环境。尽管如此，Java HotSpot还是包含了一个“后门”，提供了一些可以直接操控内存和线程的低层次操作。这个后门类——sun.misc.Unsafe——被JDK广泛用于自己的包中，如java.nio和java.util.concurrent。但是丝毫不建议在生产环境中使用这个后门。因为这个API十分不安全、不轻便、而且不稳定。这个不安全的类提供了一个观察HotSpot JVM内部结构并且可以对其进行修改。有时它可以被用来在不适用C++调试的情况下学习虚拟机内部结构，有时也可以被拿来做性能监控和开发工具。

## 为什么叫Unsafe？

Java官方不推荐使用Unsafe类，因为官方认为，这个类别人很难正确使用，非正确使用会给JVM带来致命错误。而且未来Java可能封闭丢弃这个类。


1、简单介绍
   LockSupport是JDK中比较底层的类，用来创建锁和其他同步工具类的基本线程阻塞原语。可以做到与join() 、wait()/notifyAll() 功能一样，使线程自由的阻塞、释放。
   Java锁和同步器框架的核心AQS(AbstractQueuedSynchronizer 抽象队列同步器)，就是通过调用LockSupport.park()和LockSupport.unpark()实现线程的阻塞和唤醒的。

补充：AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，
如常用的ReentrantLock/Semaphore/CountDownLatch...。

2、简单原理
  LockSupport方法底层都是调用Unsafe的方法实现。全名sun.misc.Unsafe，该类可以直接操控内存，被JDK广泛用于自己的包中，如java.nio和java.util.concurrent。但是不建议在生产环境中使用这个类。因为这个API十分不安全、不轻便、而且不稳定。

   LockSupport提供park()和unpark()方法实现阻塞线程和解除线程阻塞，LockSupport和每个使用它的线程都与一个许可(permit)关联。permit是相当于1，0的开关，默认是0，调用一次unpark就加1变成1，调用一次park会消费permit, 也就会将1变成0，同时park立即返回。

   再次调用park会变成block（因为permit为0了，会阻塞在这里，直到permit变为1）, 这时调用unpark会把permit置为1。每个线程都有一个相关的permit, permit最多只有一个，重复调用unpark也不会积累。意思就是说unpark 之后，如果permit 已经变为1，之后，再执行unpark ,permit 依旧是1。下边有例子会说到。

3、简单例子
  以下边的做饭例子，正常来说，做饭 之前，要有锅、有菜才能开始做饭 。具体如下：
（1）先假设已经有了锅 ，那只需要买菜就可以做饭。如下，即注释掉了买锅的步骤：

      public class LockSupportTest {
          public static void main(String[] args) throws InterruptedException {
            //买锅
      //      Thread t1 = new Thread(new BuyGuo(Thread.currentThread()));
      //      t1.start();
    
            //买菜
              Thread t2 = new Thread(new BuyCai(Thread.currentThread()));
              t2.start();
      //        LockSupport.park();
      //        System.out.println("锅买回来了...");
              LockSupport.park();
              System.out.println("菜买回来 了...");
              System.out.println("开始做饭");
          }
      }
      class BuyGuo implements Runnable{
          private Object threadObj;
          public BuyGuo(Object threadObj) {
              this.threadObj = threadObj;
          }
    
          @Override
          public void run() {
              System.out.println("去买锅...");
              LockSupport.unpark((Thread)threadObj);
    
          }
      }
      class BuyCai implements Runnable{
          private Object threadObj;
          public BuyCai(Object threadObj) {
              this.threadObj = threadObj;
          }
    
          @Override
          public void run() {
              System.out.println("买菜去...");
              LockSupport.unpark((Thread)threadObj);
          }
      }
  

执行后，可出现下面的结果：

买菜去...
菜买回来了...
开始做饭

   如上所述，可以达到阻塞主线程等到买完菜之后才开始做饭。这即是park()、unpark() 的用法。简单解释一下上述的步骤：

main 方法启动后，主线程 和 买菜线程 同时开始执行。
因为两者同时进行，当主线程 走到park() 时，发现permit 还为0 ，即会等待在这里。
当买菜线程执行进去后，走到unpark() 会将permit 变为1 。
主线程 park() 处发现permit 已经变成1 ，就可以继续往下执行了，同时消费掉permit ，重新变成0 。
   以上permit 只是park/unpark 执行的一种逻辑开关，执行的步骤大致如此。

4、注意点及思考
（1）必须将park()与uppark() 配对使用才更高效。
  如果上边也把买锅的线程放开，main 方法改为如下：

       //买锅
      Thread t1 = new Thread(new BuyGuo(Thread.currentThread()));
       t1.start();
      //买菜
        Thread t2 = new Thread(new BuyCai(Thread.currentThread()));
        t2.start();
        LockSupport.park();
        System.out.println("锅买回来了...");
        LockSupport.park();
        System.out.println("菜买回来了...");
        System.out.println("开始做饭");

  即调用了两次park() 和unpark() ，发现有时候可以，有时候会使线程卡在那里，然后我又换了下顺序，如下：

       //买锅
      Thread t1 = new Thread(new BuyGuo(Thread.currentThread()));
       t1.start();
          LockSupport.park();
        System.out.println("锅买回来了...");
      //买菜
        Thread t2 = new Thread(new BuyCai(Thread.currentThread()));
        t2.start();
        LockSupport.park();
        System.out.println("菜买回来了...");
        System.out.println("开始做饭");

  原理没有详细去研究,不过想了想，上边两种其实并无区别，只是执行顺序有了影响，park() 和unpark() 既然是成对配合使用，通过标识permit 来控制，如果像前边那个例子那样，出现阻塞的情况原因，我分析可能是这么个原因：

  当买锅的时候，通过unpark()将permit 置为1，但是还没等到外边的main方法执行第一个park() ,买菜的线程又调了一次unpark(),但是这时候permit 还是从1变成了1，等回到主线程调用park()的时候，因为还有两个park()需要执行，也就是需要两个消费permit ,因为permit 只有1个，所以，可能会剩下一个park()卡在那里了。

（2）使用park(Object blocker) 方法更能明确问题
  其实park() 有个重载方法park(Object blocker) ,这俩方法效果差不多，但是有blocker的可以传递给开发人员更多的现场信息，可以查看到当前线程的阻塞对象，方便定位问题。所以java6新增加带blocker入参的系列park方法，替代原有的park方法。

5、与wait()/notifyAll() 的比较
LockSupport 的 park/unpark 方法，虽然与平时Object 中wait/notify 同样达到阻塞线程的效果。但是它们之间还是有区别的。

面向的对象主体不同。LockSupport() 操作的是线程对象，直接传入的就是Thread ,而wait() 属于具体对象，notifyAll() 也是针对所有线程进行唤醒。
wait/notify 需要获取对象的监视器，即synchronized修饰，而park/unpark 不需要获取对象的监视器。
实现的机制不同，因此两者没有交集。也就是说 LockSupport 阻塞的线程，notify/notifyAll 没法唤醒。但是 park 之后，同样可以被中断(interrupt()) !


## JAVA高并发—LockSupport的学习及简单使用
1、简单介绍
   LockSupport是JDK中比较底层的类，用来创建锁和其他同步工具类的基本线程阻塞原语。可以做到与join() 、wait()/notifyAll() 功能一样，使线程自由的阻塞、释放。
   

   Java锁和同步器框架的核心AQS(AbstractQueuedSynchronizer 抽象队列同步器)，就是通过调用LockSupport.park()和LockSupport.unpark()实现线程的阻塞和唤醒的。

补充：AQS定义了一套多线程访问共享资源的同步器框架，许多同步类实现都依赖于它，
如常用的ReentrantLock/Semaphore/CountDownLatch...。

2、简单原理
  LockSupport方法底层都是调用Unsafe的方法实现。全名sun.misc.Unsafe，该类可以直接操控内存，被JDK广泛用于自己的包中，如java.nio和java.util.concurrent。但是不建议在生产环境中使用这个类。因为这个API十分不安全、不轻便、而且不稳定。

   LockSupport提供park()和unpark()方法实现阻塞线程和解除线程阻塞，LockSupport和每个使用它的线程都与一个许可(permit)关联。

permit是相当于1，0的开关，默认是0，调用一次unpark就加1变成1，调用一次park会消费permit, 也就会将1变成0，同时park立即返回。再次调用park会变成block（因为permit为0了，会阻塞在这里，直到permit变为1）, 这时调用unpark会把permit置为1。

   每个线程都有一个相关的permit, permit最多只有一个，重复调用unpark也不会积累。意思就是说unpark 之后，如果permit 已经变为1，之后，再执行unpark ,permit 依旧是1。下边有例子会说到。

3、简单例子
  以下边的做饭例子，正常来说，做饭 之前，要有锅、有菜才能开始做饭 。具体如下：
（1）先假设已经有了锅 ，那只需要买菜就可以做饭。如下，即注释掉了买锅的步骤：
    
    public class LockSupportTest {
        public static void main(String[] args) throws InterruptedException {
          //买锅
    //      Thread t1 = new Thread(new BuyGuo(Thread.currentThread()));
    //      t1.start();
    
          //买菜
            Thread t2 = new Thread(new BuyCai(Thread.currentThread()));
            t2.start();
    //        LockSupport.park();
    //        System.out.println("锅买回来了...");
            LockSupport.park();
            System.out.println("菜买回来 了...");
            System.out.println("开始做饭");
        }
    }
    class BuyGuo implements Runnable{
        private Object threadObj;
        public BuyGuo(Object threadObj) {
            this.threadObj = threadObj;
        }
    
        @Override
        public void run() {
            System.out.println("去买锅...");
            LockSupport.unpark((Thread)threadObj);
    
        }
    }
    class BuyCai implements Runnable{
        private Object threadObj;
        public BuyCai(Object threadObj) {
            this.threadObj = threadObj;
        }
    
        @Override
        public void run() {
            System.out.println("买菜去...");
            LockSupport.unpark((Thread)threadObj);
        }
    }
    
  执行后，可出现下面的结果：

买菜去...
菜买回来了...
开始做饭

   如上所述，可以达到阻塞主线程等到买完菜之后才开始做饭。这即是park()、unpark() 的用法。简单解释一下上述的步骤：

main 方法启动后，主线程 和 买菜线程 同时开始执行。
因为两者同时进行，当主线程 走到park() 时，发现permit 还为0 ，即会等待在这里。
当买菜线程执行进去后，走到unpark() 会将permit 变为1 。
主线程 park() 处发现permit 已经变成1 ，就可以继续往下执行了，同时消费掉permit ，重新变成0 。
   以上permit 只是park/unpark 执行的一种逻辑开关，执行的步骤大致如此。

4、注意点及思考
（1）必须将park()与uppark() 配对使用才更高效。
  如果上边也把买锅的线程放开，main 方法改为如下：

       //买锅
      Thread t1 = new Thread(new BuyGuo(Thread.currentThread()));
       t1.start();
      //买菜
        Thread t2 = new Thread(new BuyCai(Thread.currentThread()));
        t2.start();
        LockSupport.park();
        System.out.println("锅买回来了...");
        LockSupport.park();
        System.out.println("菜买回来了...");
        System.out.println("开始做饭");

  即调用了两次park() 和unpark() ，发现有时候可以，有时候会使线程卡在那里，然后我又换了下顺序，如下：

       //买锅
      Thread t1 = new Thread(new BuyGuo(Thread.currentThread()));
       t1.start();
          LockSupport.park();
        System.out.println("锅买回来了...");
      //买菜
        Thread t2 = new Thread(new BuyCai(Thread.currentThread()));
        t2.start();
        LockSupport.park();
        System.out.println("菜买回来了...");
        System.out.println("开始做饭");

  原理没有详细去研究,不过想了想，上边两种其实并无区别，只是执行顺序有了影响，park() 和unpark() 既然是成对配合使用，通过标识permit 来控制，如果像前边那个例子那样，出现阻塞的情况原因，我分析可能是这么个原因：

  当买锅的时候，通过unpark()将permit 置为1，但是还没等到外边的main方法执行第一个park() ,买菜的线程又调了一次unpark(),但是这时候permit 还是从1变成了1，等回到主线程调用park()的时候，因为还有两个park()需要执行，也就是需要两个消费permit ,因为permit 只有1个，所以，可能会剩下一个park()卡在那里了。

（2）使用park(Object blocker) 方法更能明确问题
  其实park() 有个重载方法park(Object blocker) ,这俩方法效果差不多，但是有blocker的可以传递给开发人员更多的现场信息，可以查看到当前线程的阻塞对象，方便定位问题。所以java6新增加带blocker入参的系列park方法，替代原有的park方法。

5、与wait()/notifyAll() 的比较
LockSupport 的 park/unpark 方法，虽然与平时Object 中wait/notify 同样达到阻塞线程的效果。但是它们之间还是有区别的。

面向的对象主体不同。LockSupport() 操作的是线程对象，直接传入的就是Thread ,而wait() 属于具体对象，notifyAll() 也是针对所有线程进行唤醒。
wait/notify 需要获取对象的监视器，即synchronized修饰，而park/unpark 不需要获取对象的监视器。
实现的机制不同，因此两者没有交集。也就是说 LockSupport 阻塞的线程，notify/notifyAll 没法唤醒。但是 park 之后，同样可以被中断(interrupt()) !
