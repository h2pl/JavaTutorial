# Table of Contents

  * [JVM GC基本原理与GC算法](#jvm-gc基本原理与gc算法)
  * [Java关键术语](#java关键术语)
  * [Java HotSpot 虚拟机](#java-hotspot-虚拟机)
  * [Java堆内存](#java堆内存)
  * [启动Java垃圾回收](#启动java垃圾回收)
  * [各种GC的触发时机(When)](#各种gc的触发时机when)
    * [GC类型](#gc类型)
    * [触发时机](#触发时机)
    * [FULL GC触发条件详解](#full-gc触发条件详解)
    * [总结](#总结)
    * [什么是Stop the world](#什么是stop-the-world)
  * [Java垃圾回收过程](#java垃圾回收过程)
  * [垃圾回收中实例的终结](#垃圾回收中实例的终结)
  * [对象什么时候符合垃圾回收的条件？](#对象什么时候符合垃圾回收的条件？)
    * [GC Scope 示例程序](#gc-scope-示例程序)
  * [[JVM GC算法](https://www.cnblogs.com/wupeixuan/p/8670341.html)](#[jvm-gc算法]httpswwwcnblogscomwupeixuanp8670341html)
  * [JVM垃圾判定算法](#jvm垃圾判定算法)
    * [引用计数算法(Reference Counting)](#引用计数算法reference-counting)
    * [可达性分析算法（根搜索算法）](#可达性分析算法（根搜索算法）)
  * [四种引用](#四种引用)
  * [JVM垃圾回收算法](#jvm垃圾回收算法)
  * [标记—清除算法（Mark-Sweep）](#标记清除算法（mark-sweep）)
  * [复制算法（Copying）](#复制算法（copying）)
  * [标记—整理算法（Mark-Compact）](#标记整理算法（mark-compact）)
  * [分代收集算法(Generational Collection)](#分代收集算法generational-collection)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


本文转自互联网，侵删

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章将同步到我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《深入理解JVM虚拟机》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，一步步地学习JVM基础知识，并上手进行JVM调优实战，JVM是每一个Java工程师必须要学习和理解的知识点，你必须要掌握其实现原理，才能更完整地了解整个Java技术体系，形成自己的知识框架。

为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## JVM GC基本原理与GC算法



Java的内存分配与回收全部由JVM垃圾回收进程自动完成。与C语言不同，Java开发者不需要自己编写代码实现垃圾回收。这是Java深受大家欢迎的众多特性之一，能够帮助程序员更好地编写Java程序。

下面四篇教程是了解Java 垃圾回收（GC）的基础：

1.  [垃圾回收简介](http://www.importnew.com/13504.html)
2.  [圾回收是如何工作的？](http://www.importnew.com/13493.html)
3.  [垃圾回收的类别](http://www.importnew.com/13827.html)

这篇教程是系列第一部分。首先会解释基本的术语，比如JDK、JVM、JRE和HotSpotVM。接着会介绍JVM结构和Java 堆内存结构。理解这些基础对于理解后面的垃圾回收知识很重要。

## Java关键术语

*   JavaAPI：一系列帮助开发者创建Java应用程序的封装好的库。
*   Java 开发工具包 （JDK）：一系列工具帮助开发者创建Java应用程序。JDK包含工具编译、运行、打包、分发和监视Java应用程序。
*   Java 虚拟机（JVM）：JVM是一个抽象的计算机结构。Java程序根据JVM的特性编写。JVM针对特定于操作系统并且可以将Java指令翻译成底层系统的指令并执行。JVM确保了Java的平台无关性。
*   Java 运行环境（JRE）：JRE包含JVM实现和Java API。

## Java HotSpot 虚拟机

每种JVM实现可能采用不同的方法实现垃圾回收机制。在收购SUN之前，Oracle使用的是JRockit JVM，收购之后使用HotSpot JVM。目前Oracle拥有两种JVM实现并且一段时间后两个JVM实现会合二为一。

HotSpot JVM是目前Oracle SE平台标准核心组件的一部分。在这篇垃圾回收教程中，我们将会了解基于HotSpot虚拟机的垃圾回收原则。

## Java堆内存

我们有必要了解堆内存在JVM内存模型的角色。在运行时，Java的实例被存放在堆内存区域。当一个对象不再被引用时，满足条件就会从堆内存移除。在垃圾回收进程中，这些对象将会从堆内存移除并且内存空间被回收。堆内存以下三个主要区域：

1.  新生代（Young Generation）
    *   Eden空间（Eden space，任何实例都通过Eden空间进入运行时内存区域）
    *   S0 Survivor空间（S0 Survivor space，存在时间长的实例将会从Eden空间移动到S0 Survivor空间）
    *   S1 Survivor空间 （存在时间更长的实例将会从S0 Survivor空间移动到S1 Survivor空间）
2.  老年代（Old Generation）实例将从S1提升到Tenured（终身代）
3.  永久代（Permanent Generation）包含类、方法等细节的元信息


永久代空间[在Java SE8特性](http://javapapers.com/java/java-8-features/)中已经被移除。

Java 垃圾回收是一项自动化的过程，用来管理程序所使用的运行时内存。通过这一自动化过程，JVM 解除了程序员在程序中分配和释放内存资源的开销。

## 启动Java垃圾回收

作为一个自动的过程，程序员不需要在代码中显示地启动垃圾回收过程。`System.gc()`和`Runtime.gc()`用来请求JVM启动垃圾回收。

虽然这个请求机制提供给程序员一个启动 GC 过程的机会，但是启动由 JVM负责。JVM可以拒绝这个请求，所以并不保证这些调用都将执行垃圾回收。启动时机的选择由JVM决定，并且取决于堆内存中Eden区是否可用。JVM将这个选择留给了Java规范的实现，不同实现具体使用的算法不尽相同。

毋庸置疑，我们知道垃圾回收过程是不能被强制执行的。我刚刚发现了一个调用`System.gc()`有意义的场景。通过这篇文章了解一下[适合调用System.gc() ](http://javapapers.com/core-java/system-gc-invocation-a-suitable-scenario/)这种极端情况。

## 各种GC的触发时机(When)

### GC类型

说到GC类型，就更有意思了，为什么呢，因为业界没有统一的严格意义上的界限，也没有严格意义上的GC类型，都是左边一个教授一套名字，右边一个作者一套名字。为什么会有这个情况呢，因为GC类型是和收集器有关的，不同的收集器会有自己独特的一些收集类型。所以作者在这里引用**R大**关于GC类型的介绍，作者觉得还是比较妥当准确的。如下:

*   Partial GC：并不收集整个GC堆的模式
    *   Young GC(Minor GC)：只收集young gen的GC
    *   Old GC：只收集old gen的GC。只有CMS的concurrent collection是这个模式
    *   Mixed GC：收集整个young gen以及部分old gen的GC。只有G1有这个模式
*   Full GC(Major GC)：收集整个堆，包括young gen、old gen、perm gen（如果存在的话）等所有部分的模式。

### 触发时机

上面大家也看到了，GC类型分分类是和收集器有关的，那么当然了，对于不同的收集器，GC触发时机也是不一样的，作者就针对默认的serial GC来说:

*   young GC：当young gen中的eden区分配满的时候触发。注意young GC中有部分存活对象会晋升到old gen，所以young GC后old gen的占用量通常会有所升高。
*   full GC：当准备要触发一次young GC时，如果发现统计数据说之前young GC的平均晋升大小比目前old gen剩余的空间大，则不会触发young GC而是转为触发full GC（因为HotSpot VM的GC里，除了CMS的concurrent collection之外，其它能收集old gen的GC都会同时收集整个GC堆，包括young gen，所以不需要事先触发一次单独的young GC）；或者，如果有perm gen的话，要在perm gen分配空间但已经没有足够空间时，也要触发一次full GC；或者System.gc()、heap dump带GC，默认也是触发full GC。

### FULL GC触发条件详解

除直接调用System.gc外，触发Full GC执行的情况有如下四种。

1. 旧生代空间不足

旧生代空间只有在新生代对象转入及创建为大对象、大数组时才会出现不足的现象，当执行Full GC后空间仍然不足，则抛出如下错误：

java.lang.OutOfMemoryError: Java heap space 

为避免以上两种状况引起的Full GC，调优时应尽量做到让对象在Minor GC阶段被回收、让对象在新生代多存活一段时间及不要创建过大的对象及数组。

2\. Permanet Generation空间满

Permanet Generation中存放的为一些class的信息等，当系统中要加载的类、反射的类和调用的方法较多时，Permanet Generation可能会被占满，在未配置为采用CMS GC的情况下会执行Full GC。如果经过Full GC仍然回收不了，那么JVM会抛出如下错误信息：

java.lang.OutOfMemoryError: PermGen space 

为避免Perm Gen占满造成Full GC现象，可采用的方法为增大Perm Gen空间或转为使用CMS GC。

3\. CMS GC时出现promotion failed和concurrent mode failure

对于采用CMS进行旧生代GC的程序而言，尤其要注意GC日志中是否有promotion failed和concurrent mode failure两种状况，当这两种状况出现时可能会触发Full GC。

promotion failed是在进行Minor GC时，survivor space放不下、对象只能放入旧生代，而此时旧生代也放不下造成的；concurrent mode failure是在执行CMS GC的过程中同时有对象要放入旧生代，而此时旧生代空间不足造成的。

应对措施为：增大survivor space、旧生代空间或调低触发并发GC的比率，但在JDK 5.0+、6.0+的版本中有可能会由于JDK的bug29导致CMS在remark完毕后很久才触发sweeping动作。对于这种状况，可通过设置-XX: CMSMaxAbortablePrecleanTime=5（单位为ms）来避免。

4. 统计得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间

这是一个较为复杂的触发情况，Hotspot为了避免由于新生代对象晋升到旧生代导致旧生代空间不足的现象，在进行Minor GC时，做了一个判断，如果之前统计所得到的Minor GC晋升到旧生代的平均大小大于旧生代的剩余空间，那么就直接触发Full GC。

例如程序第一次触发Minor GC后，有6MB的对象晋升到旧生代，那么当下一次Minor GC发生时，首先检查旧生代的剩余空间是否大于6MB，如果小于6MB，则执行Full GC。

当新生代采用PS GC时，方式稍有不同，PS GC是在Minor GC后也会检查，例如上面的例子中第一次Minor GC后，PS GC会检查此时旧生代的剩余空间是否大于6MB，如小于，则触发对旧生代的回收。

除了以上4种状况外，对于使用RMI来进行RPC或管理的Sun JDK应用而言，默认情况下会一小时执行一次Full GC。可通过在启动时通过- java -Dsun.rmi.dgc.client.gcInterval=3600000来设置Full GC执行的间隔时间或通过-XX:+ DisableExplicitGC来禁止RMI调用System.gc。

### 总结

** Minor GC ，Full GC 触发条件**

Minor GC触发条件：当Eden区满时，触发Minor GC。

Full GC触发条件：

（1）调用System.gc时，系统建议执行Full GC，但是不必然执行

（2）老年代空间不足

（3）方法去空间不足

（4）通过Minor GC后进入老年代的平均大小大于老年代的可用内存

（5）由Eden区、From Space区向To Space区复制时，对象大小大于To Space可用内存，则把该对象转存到老年代，且老年代的可用内存小于该对象大小

### 什么是Stop the world

Java中Stop-The-World机制简称STW，是在执行垃圾收集算法时，Java应用程序的其他所有线程都被挂起（除了垃圾收集帮助器之外）。Java中一种全局暂停现象，全局停顿，所有Java代码停止，native代码可以执行，但不能与JVM交互；这些现象多半是由于gc引起。

GC时的Stop the World(STW)是大家最大的敌人。但可能很多人还不清楚，除了GC，JVM下还会发生停顿现象。

JVM里有一条特殊的线程－－VM Threads，专门用来执行一些特殊的VM Operation，比如分派GC，thread dump等，这些任务，都需要整个Heap，以及所有线程的状态是静止的，一致的才能进行。所以JVM引入了安全点(Safe Point)的概念，想办法在需要进行VM Operation时，通知所有的线程进入一个静止的安全点。

除了GC，其他触发安全点的VM Operation包括：

1\. JIT相关，比如Code deoptimization, Flushing code cache ；

2\. Class redefinition (e.g. javaagent，AOP代码植入的产生的instrumentation) ；

3\. Biased lock revocation 取消偏向锁 ；

4\. Various debug operation (e.g. thread dump or deadlock check)；

## Java垃圾回收过程

垃圾回收是一种回收无用内存空间并使其对未来实例可用的过程。

Eden 区：当一个实例被创建了，首先会被存储在堆内存年轻代的 Eden 区中。

注意：如果你不能理解这些词汇，我建议你阅读这篇 [垃圾回收介绍](http://javapapers.com/java/java-garbage-collection-introduction/) ，这篇教程详细地介绍了内存模型、JVM 架构以及这些术语。

Survivor 区（S0 和 S1）：作为年轻代 GC（Minor GC）周期的一部分，存活的对象（仍然被引用的）从 Eden 区被移动到 Survivor 区的 S0 中。类似的，垃圾回收器会扫描 S0 然后将存活的实例移动到 S1 中。

（译注：此处不应该是Eden和S0中存活的都移到S1么，为什么会先移到S0再从S0移到S1？）

死亡的实例（不再被引用）被标记为垃圾回收。根据垃圾回收器（有四种常用的垃圾回收器，将在下一教程中介绍它们）选择的不同，要么被标记的实例都会不停地从内存中移除，要么回收过程会在一个单独的进程中完成。

老年代： 老年代（Old or tenured generation）是堆内存中的第二块逻辑区。当垃圾回收器执行 Minor GC 周期时，在 S1 Survivor 区中的存活实例将会被晋升到老年代，而未被引用的对象被标记为回收。

老年代 GC（Major GC）：相对于 Java 垃圾回收过程，老年代是实例生命周期的最后阶段。Major GC 扫描老年代的垃圾回收过程。如果实例不再被引用，那么它们会被标记为回收，否则它们会继续留在老年代中。

内存碎片：一旦实例从堆内存中被删除，其位置就会变空并且可用于未来实例的分配。这些空出的空间将会使整个内存区域碎片化。为了实例的快速分配，需要进行碎片整理。基于垃圾回收器的不同选择，回收的内存区域要么被不停地被整理，要么在一个单独的GC进程中完成。

## 垃圾回收中实例的终结

在释放一个实例和回收内存空间之前，Java 垃圾回收器会调用实例各自的 `finalize()` 方法，从而该实例有机会释放所持有的资源。虽然可以保证 `finalize()` 会在回收内存空间之前被调用，但是没有指定的顺序和时间。多个实例间的顺序是无法被预知，甚至可能会并行发生。程序不应该预先调整实例之间的顺序并使用 `finalize()` 方法回收资源。

*   任何在 finalize过程中未被捕获的异常会自动被忽略，然后该实例的 finalize 过程被取消。
*   JVM 规范中并没有讨论关于弱引用的垃圾回收机制，也没有很明确的要求。具体的实现都由实现方决定。
*   垃圾回收是由一个守护线程完成的。

## 对象什么时候符合垃圾回收的条件？

*   所有实例都没有活动线程访问。
*   没有被其他任何实例访问的循环引用实例。

[Java 中有不同的引用类型](http://javapapers.com/core-java/java-weak-reference/)。判断实例是否符合垃圾收集的条件都依赖于它的引用类型。

| 引用类型 | 垃圾收集 |
| --- | --- |
| 强引用（Strong Reference） | 不符合垃圾收集 |
| 软引用（Soft Reference） | 垃圾收集可能会执行，但会作为最后的选择 |
| 弱引用（Weak Reference） | 符合垃圾收集 |
| 虚引用（Phantom Reference） | 符合垃圾收集 |

在编译过程中作为一种优化技术，Java 编译器能选择给实例赋 `null` 值，从而标记实例为可回收。

    class Animal {
    
        public static void main(String[] args) {
    
            Animal lion = new Animal();
    
            System.out.println("Main is completed.");
    
        }
    
     
    
        protected void finalize() {
    
            System.out.println("Rest in Peace!");
    
        }
    
    }

在上面的类中，`lion` 对象在实例化行后从未被使用过。因此 Java 编译器作为一种优化措施可以直接在实例化行后赋值`lion = null`。因此，即使在 SOP 输出之前， finalize 函数也能够打印出 `'Rest in Peace!'`。我们不能证明这确定会发生，因为它依赖JVM的实现方式和运行时使用的内存。然而，我们还能学习到一点：如果编译器看到该实例在未来再也不会被引用，能够选择并提早释放实例空间。

*   关于对象什么时候符合垃圾回收有一个更好的例子。实例的所有属性能被存储在寄存器中，随后寄存器将被访问并读取内容。无一例外，这些值将被写回到实例中。虽然这些值在将来能被使用，这个实例仍然能被标记为符合垃圾回收。这是一个很经典的例子，不是吗？
*   当被赋值为null时，这是很简单的一个符合垃圾回收的示例。当然，复杂的情况可以像上面的几点。这是由 JVM 实现者所做的选择。目的是留下尽可能小的内存占用，加快响应速度，提高吞吐量。为了实现这一目标， JVM 的实现者可以选择一个更好的方案或算法在垃圾回收过程中回收内存空间。
*   当 `finalize()` 方法被调用时，JVM 会释放该线程上的所有同步锁。

### GC Scope 示例程序
    Class GCScope {
    
        GCScope t;
    
        static int i = 1;
    
     
    
        public static void main(String args[]) {
    
            GCScope t1 = new GCScope();
    
            GCScope t2 = new GCScope();
    
            GCScope t3 = new GCScope();
    
     
    
            // No Object Is Eligible for GC
    
     
    
            t1.t = t2; // No Object Is Eligible for GC
    
            t2.t = t3; // No Object Is Eligible for GC
    
            t3.t = t1; // No Object Is Eligible for GC
    
     
    
            t1 = null;
    
            // No Object Is Eligible for GC (t3.t still has a reference to t1)
    
     
    
            t2 = null;
    
            // No Object Is Eligible for GC (t3.t.t still has a reference to t2)
    
     
    
            t3 = null;
    
            // All the 3 Object Is Eligible for GC (None of them have a reference.
    
            // only the variable t of the objects are referring each other in a
    
            // rounded fashion forming the Island of objects with out any external
    
            // reference)
    
        }
    
     
    
        protected void finalize() {
    
            System.out.println("Garbage collected from object" + i);
    
            i++;
    
        }
    
     
    
    class GCScope {
    
        GCScope t;
    
        static int i = 1;
    
     
    
        public static void main(String args[]) {
    
            GCScope t1 = new GCScope();
    
            GCScope t2 = new GCScope();
    
            GCScope t3 = new GCScope();
    
     
    
            // 没有对象符合GC
    
            t1.t = t2; // 没有对象符合GC
    
            t2.t = t3; // 没有对象符合GC
    
            t3.t = t1; // 没有对象符合GC
    
     
    
            t1 = null;
    
            // 没有对象符合GC (t3.t 仍然有一个到 t1 的引用)
    
     
    
            t2 = null;
    
            // 没有对象符合GC (t3.t.t 仍然有一个到 t2 的引用)
    
     
    
            t3 = null;
    
            // 所有三个对象都符合GC (它们中没有一个拥有引用。
    
            // 只有各对象的变量 t 还指向了彼此，
    
            // 形成了一个由对象组成的环形的岛，而没有任何外部的引用。)
    
        }
    
     
    
        protected void finalize() {
    
            System.out.println("Garbage collected from object" + i);
    
            i++;
    
        }
## [JVM GC算法](https://www.cnblogs.com/wupeixuan/p/8670341.html)

在判断哪些内存需要回收和什么时候回收用到GC 算法，本文主要对GC 算法进行讲解。

## JVM垃圾判定算法

常见的JVM垃圾判定算法包括：引用计数算法、可达性分析算法。

### 引用计数算法(Reference Counting)

引用计数算法是通过判断对象的引用数量来决定对象是否可以被回收。

给对象中添加一个引用计数器，每当有一个地方引用它时，计数器值就加1；当引用失效时，计数器值就减1；任何时刻计数器为0的对象就是不可能再被使用的。

优点：简单，高效，现在的objective-c用的就是这种算法。

缺点：很难处理循环引用，相互引用的两个对象则无法释放。因此目前主流的Java虚拟机都摒弃掉了这种算法。

举个简单的例子，对象objA和objB都有字段instance，赋值令objA.instance=objB及objB.instance=objA，除此之外，这两个对象没有任何引用，实际上这两个对象已经不可能再被访问，但是因为互相引用，导致它们的引用计数都不为0，因此引用计数算法无法通知GC收集器回收它们。

```

public class ReferenceCountingGC {
    public Object instance = null;

    public static void main(String[] args) {
        ReferenceCountingGC objA = new ReferenceCountingGC();
        ReferenceCountingGC objB = new ReferenceCountingGC();
        objA.instance = objB;
        objB.instance = objA;

        objA = null;
        objB = null;

        System.gc();//GC
    }
}
```

运行结果

```

[GC (System.gc()) [PSYoungGen: 3329K->744K(38400K)] 3329K->752K(125952K), 0.0341414 secs] [Times: user=0.00 sys=0.00, real=0.06 secs] 
[Full GC (System.gc()) [PSYoungGen: 744K->0K(38400K)] [ParOldGen: 8K->628K(87552K)] 752K->628K(125952K), [Metaspace: 3450K->3450K(1056768K)], 0.0060728 secs] [Times: user=0.05 sys=0.00, real=0.01 secs] 
Heap
 PSYoungGen      total 38400K, used 998K [0x00000000d5c00000, 0x00000000d8680000, 0x0000000100000000)
  eden space 33280K, 3% used [0x00000000d5c00000,0x00000000d5cf9b20,0x00000000d7c80000)
  from space 5120K, 0% used [0x00000000d7c80000,0x00000000d7c80000,0x00000000d8180000)
  to   space 5120K, 0% used [0x00000000d8180000,0x00000000d8180000,0x00000000d8680000)
 ParOldGen       total 87552K, used 628K [0x0000000081400000, 0x0000000086980000, 0x00000000d5c00000)
  object space 87552K, 0% used [0x0000000081400000,0x000000008149d2c8,0x0000000086980000)
 Metaspace       used 3469K, capacity 4496K, committed 4864K, reserved 1056768K
  class space    used 381K, capacity 388K, committed 512K, reserved 1048576K

Process finished with exit code 0
```

从运行结果看，GC日志中包含“3329K->744K”,意味着虚拟机并没有因为这两个对象互相引用就不回收它们，说明虚拟机不是通过引用技术算法来判断对象是否存活的。

### 可达性分析算法（根搜索算法）

可达性分析算法是通过判断对象的引用链是否可达来决定对象是否可以被回收。

从GC Roots（每种具体实现对GC Roots有不同的定义）作为起点，向下搜索它们引用的对象，可以生成一棵引用树，树的节点视为可达对象，反之视为不可达。

![可达性分析算法](https://images.cnblogs.com/cnblogs_com/wupeixuan/1186116/o_4240985.jpg)

在Java语言中，可以作为GC Roots的对象包括下面几种：

*   虚拟机栈（栈帧中的本地变量表）中的引用对象。
*   方法区中的类静态属性引用的对象。
*   方法区中的常量引用的对象。
*   本地方法栈中JNI（Native方法）的引用对象

真正标记以为对象为可回收状态至少要标记两次。

## 四种引用

强引用就是指在程序代码之中普遍存在的，类似"Object obj = new Object()"这类的引用，只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象。

```

Object obj = new Object();
```

软引用是用来描述一些还有用但并非必需的对象，对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK1.2之后，提供了SoftReference类来实现软引用。

```

Object obj = new Object();
SoftReference<Object> sf = new SoftReference<Object>(obj);
```

弱引用也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象，只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在JDK1.2之后，提供了WeakReference类来实现弱引用。

```

Object obj = new Object();
WeakReference<Object> wf = new WeakReference<Object>(obj);
```

虚引用也成为幽灵引用或者幻影引用，它是最弱的一中引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。在JDK1.2之后，提供给了PhantomReference类来实现虚引用。

```

Object obj = new Object();
PhantomReference<Object> pf = new PhantomReference<Object>(obj);
```

## JVM垃圾回收算法

常见的垃圾回收算法包括：标记-清除算法，复制算法，标记-整理算法，分代收集算法。

在介绍JVM垃圾回收算法前，先介绍一个概念。

Stop-the-World

Stop-the-world意味着 JVM由于要执行GC而停止了应用程序的执行，并且这种情形会在任何一种GC算法中发生。当Stop-the-world发生时，除了GC所需的线程以外，所有线程都处于等待状态直到GC任务完成。事实上，GC优化很多时候就是指减少Stop-the-world发生的时间，从而使系统具有高吞吐 、低停顿的特点。

## 标记—清除算法（Mark-Sweep）

之所以说标记/清除算法是几种GC算法中最基础的算法，是因为后续的收集算法都是基于这种思路并对其不足进行改进而得到的。标记/清除算法的基本思想就跟它的名字一样，分为“标记”和“清除”两个阶段：首先标记出所有需要回收的对象，在标记完成后统一回收所有被标记的对象。

标记阶段：标记的过程其实就是前面介绍的可达性分析算法的过程，遍历所有的GC Roots对象，对从GC Roots对象可达的对象都打上一个标识，一般是在对象的header中，将其记录为可达对象；

清除阶段：清除的过程是对堆内存进行遍历，如果发现某个对象没有被标记为可达对象（通过读取对象header信息），则将其回收。

不足：

*   标记和清除过程效率都不高
*   会产生大量碎片，内存碎片过多可能导致无法给大对象分配内存。

![标记-清除](https://images.cnblogs.com/cnblogs_com/wupeixuan/1186116/o_a4248c4b-6c1d-4fb8-a557-86da92d3a294.jpg)

## 复制算法（Copying）

将内存划分为大小相等的两块，每次只使用其中一块，当这一块内存用完了就将还存活的对象复制到另一块上面，然后再把使用过的内存空间进行一次清理。

现在的商业虚拟机都采用这种收集算法来回收新生代，但是并不是将内存划分为大小相等的两块，而是分为一块较大的 Eden 空间和两块较小的 Survior 空间，每次使用 Eden 空间和其中一块 Survivor。在回收时，将 Eden 和 Survivor 中还存活着的对象一次性复制到另一块 Survivor 空间上，最后清理 Eden 和 使用过的那一块 Survivor。HotSpot 虚拟机的 Eden 和 Survivor 的大小比例默认为 8:1，保证了内存的利用率达到 90 %。如果每次回收有多于 10% 的对象存活，那么一块 Survivor 空间就不够用了，此时需要依赖于老年代进行分配担保，也就是借用老年代的空间。

不足：

*   将内存缩小为原来的一半，浪费了一半的内存空间，代价太高；如果不想浪费一半的空间，就需要有额外的空间进行分配担保，以应对被使用的内存中所有对象都100%存活的极端情况，所以在老年代一般不能直接选用这种算法。
*   复制收集算法在对象存活率较高时就要进行较多的复制操作，效率将会变低。

![复制](https://images.cnblogs.com/cnblogs_com/wupeixuan/1186116/o_e6b733ad-606d-4028-b3e8-83c3a73a3797.jpg)

## 标记—整理算法（Mark-Compact）

标记—整理算法和标记—清除算法一样，但是标记—整理算法不是把存活对象复制到另一块内存，而是把存活对象往内存的一端移动，然后直接回收边界以外的内存，因此其不会产生内存碎片。标记—整理算法提高了内存的利用率，并且它适合在收集对象存活时间较长的老年代。

不足：

效率不高，不仅要标记存活对象，还要整理所有存活对象的引用地址，在效率上不如复制算法。

![标记—整理](https://images.cnblogs.com/cnblogs_com/wupeixuan/1186116/o_902b83ab-8054-4bd2-898f-9a4a0fe52830.jpg)

## 分代收集算法(Generational Collection)

分代回收算法实际上是把复制算法和标记整理法的结合，并不是真正一个新的算法，一般分为：老年代（Old Generation）和新生代（Young Generation），老年代就是很少垃圾需要进行回收的，新生代就是有很多的内存空间需要回收，所以不同代就采用不同的回收算法，以此来达到高效的回收算法。

新生代：由于新生代产生很多临时对象，大量对象需要进行回收，所以采用复制算法是最高效的。

老年代：回收的对象很少，都是经过几次标记后都不是可回收的状态转移到老年代的，所以仅有少量对象需要回收，故采用标记清除或者标记整理算法。

## 参考文章

<https://segmentfault.com/a/1190000009707894>

<https://www.cnblogs.com/hysum/p/7100874.html>

<http://c.biancheng.net/view/939.html>

<https://www.runoob.com/>

https://blog.csdn.net/android_hl/article/details/53228348

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
