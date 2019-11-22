# Table of Contents

  * [实战内存溢出异常](#实战内存溢出异常)
  * [1 . 对象的创建过程](#1--对象的创建过程)
  * [2 . 对象的内存布局](#2--对象的内存布局)
  * [3 . 对象的访问定位](#3--对象的访问定位)
  * [4 .实战内存异常](#4-实战内存异常)
    * [Java堆内存异常](#java堆内存异常)
    * [Java栈内存异常](#java栈内存异常)
    * [方法区内存异常](#方法区内存异常)
    * [方法区与运行时常量池OOM](#方法区与运行时常量池oom)
    * [附加-直接内存异常](#附加-直接内存异常)
  * [Java内存泄漏](#java内存泄漏)
  * [Java是如何管理内存？](#java是如何管理内存？)
  * [什么是Java中的内存泄露？](#什么是java中的内存泄露？)
    * [其他常见内存泄漏](#其他常见内存泄漏)
      * [1、静态集合类引起内存泄露：](#1、静态集合类引起内存泄露：)
      * [2、当集合里面的对象属性被修改后，再调用remove（）方法时不起作用。](#2、当集合里面的对象属性被修改后，再调用remove（）方法时不起作用。)
      * [3、监听器](#3、监听器)
      * [4、各种连接](#4、各种连接)
      * [5、内部类和外部模块等的引用](#5、内部类和外部模块等的引用)
      * [6、单例模式](#6、单例模式)
  * [如何检测内存泄漏](#如何检测内存泄漏)
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

## 实战内存溢出异常

大家好，相信大部分Javaer在code时经常会遇到本地代码运行正常，但在生产环境偶尔会莫名其妙的报一些关于内存的异常，StackOverFlowError,OutOfMemoryError异常是最常见的。今天就基于上篇文章JVM系列之Java内存结构详解讲解的各个内存区域重点实战分析下内存溢出的情况。在此之前，我还是想多余累赘一些其他关于对象的问题，具体内容如下:

> 文章结构：
> 对象的创建过程
> 对象的内存布局
> 对象的访问定位
> 实战内存异常

## 1 . 对象的创建过程

关于对象的创建，第一反应是new关键字，那么本文就主要讲解new关键字创建对象的过程。

```
Student stu =new Student("张三"，"18");

```

就拿上面这句代码来说，虚拟机首先会去检查Student这个类有没有被加载，如果没有，首先去加载这个类到方法区，然后根据加载的Class类对象创建stu实例对象，需要注意的是，stu对象所需的内存大小在Student类加载完成后便可完全确定。内存分配完成后，虚拟机需要将分配到的内存空间的实例数据部分初始化为零值,这也就是为什么我们在编写Java代码时创建一个变量不需要初始化。紧接着，虚拟机会对对象的对象头进行必要的设置，如这个对象属于哪个类，如何找到类的元数据(Class对象),对象的锁信息，GC分代年龄等。设置完对象头信息后，调用类的构造函数。
其实讲实话，虚拟机创建对象的过程远不止这么简单，我这里只是把大致的脉络讲解了一下，方便大家理解。

## 2 . 对象的内存布局

刚刚提到的实例数据，对象头，有些小伙伴也许有点陌生，这一小节就详细讲解一下对象的内存布局,对象创建完成后大致可以分为以下几个部分:

*   对象头
*   实例数据
*   对齐填充

**对象头:** 对象头中包含了对象运行时一些必要的信息，如GC分代信息，锁信息，哈希码，指向Class类元信息的指针等，其中对Javaer比较有用的是**锁信息与指向Class对象的指针**，关于锁信息，后期有机会讲解并发编程JUC时再扩展，关于指向Class对象的指针其实很好理解。比如上面那个Student的例子，当我们拿到stu对象时，调用Class stuClass=stu.getClass();的时候，其实就是根据这个指针去拿到了stu对象所属的Student类在方法区存放的Class类对象。虽然说的有点拗口，但这句话我反复琢磨了好几遍，应该是说清楚了。

**实例数据:** 实例数据部分是对象真正存储的有效信息，就是程序代码中所定义的各种类型的字段内容。

**对齐填充:** 虚拟机规范要求对象大小必须是8字节的整数倍。对齐填充其实就是来补全对象大小的。

## 3 . 对象的访问定位

谈到对象的访问，还拿上面学生的例子来说，当我们拿到stu对象时，直接调用stu.getName();时，其实就完成了对对象的访问。但这里要累赘说一下的是，stu虽然通常被认为是一个对象，其实准确来说是不准确的，stu只是一个变量，变量里存储的是指向对象的指针，(如果干过C或者C++的小伙伴应该比较清楚指针这个概念)，当我们调用stu.getName()时，虚拟机会根据指针找到堆里面的对象然后拿到实例数据name.需要注意的是，当我们调用stu.getClass()时，虚拟机会首先根据stu指针定位到堆里面的对象，然后根据对象头里面存储的指向Class类元信息的指针再次到方法区拿到Class对象，进行了两次指针寻找。具体讲解图如下:
![在这里插入图片描述](https://img-blog.csdnimg.cn/201901092235030.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzE5MjczMg==,size_16,color_FFFFFF,t_70)

## 4 .实战内存异常

内存异常是我们工作当中经常会遇到问题，但如果仅仅会通过加大内存参数来解决问题显然是不够的，应该通过一定的手段定位问题，到底是因为参数问题，还是程序问题(无限创建，内存泄露)。定位问题后才能采取合适的解决方案，而不是一内存溢出就查找相关参数加大。

> 概念
> 内存泄露:代码中的某个对象本应该被虚拟机回收，但因为拥有GCRoot引用而没有被回收。关于GCRoot概念，下一篇文章讲解。
> 内存溢出: 虚拟机由于堆中拥有太多不可回收对象没有回收，导致无法继续创建新对象。

在分析问题之前先给大家讲一讲排查内存溢出问题的方法，内存溢出时JVM虚拟机会退出，**那么我们怎么知道JVM运行时的各种信息呢，Dump机制会帮助我们，可以通过加上VM参数-XX:+HeapDumpOnOutOfMemoryError让虚拟机在出现内存溢出异常时生成dump文件，然后通过外部工具(作者使用的是VisualVM)来具体分析异常的原因。**

下面从以下几个方面来配合代码实战演示内存溢出及如何定位:

*   Java堆内存异常
*   Java栈内存异常
*   方法区内存异常

### Java堆内存异常

```
/**
    VM Args:
    //这两个参数保证了堆中的可分配内存固定为20M
    -Xms20m
    -Xmx20m  
    //文件生成的位置，作则生成在桌面的一个目录
    -XX:+HeapDumpOnOutOfMemoryError //文件生成的位置，作则生成在桌面的一个目录
    //文件生成的位置，作则生成在桌面的一个目录
    -XX:HeapDumpPath=/Users/zdy/Desktop/dump/ 
 */
public class HeapOOM {
    //创建一个内部类用于创建对象使用
    static class OOMObject {
    }
    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        //无限创建对象，在堆中
        while (true) {
            list.add(new OOMObject());
        }
    }
}

```

Run起来代码后爆出异常如下:

java.lang.OutOfMemoryError: Java heap space
Dumping heap to /Users/zdy/Desktop/dump/java_pid1099.hprof …

可以看到生成了dump文件到指定目录。并且爆出了OutOfMemoryError，还告诉了你是哪一片区域出的问题:heap space

打开VisualVM工具导入对应的heapDump文件(如何使用请读者自行查阅相关资料)，相应的说明见图:
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190109224456933.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzE5MjczMg==,size_16,color_FFFFFF,t_70)
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190109224531128.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzE5MjczMg==,size_16,color_FFFFFF,t_70)
分析dump文件后，我们可以知道，OOMObject这个类创建了810326个实例。所以它能不溢出吗？接下来就在代码里找这个类在哪new的。排查问题。(我们的样例代码就不用排查了，While循环太凶猛了)分析dump文件后，我们可以知道，OOMObject这个类创建了810326个实例。所以它能不溢出吗？接下来就在代码里找这个类在哪new的。排查问题。(我们的样例代码就不用排查了，While循环太凶猛了)

### Java栈内存异常

老实说，在栈中出现异常(StackOverFlowError)的概率小到和去苹果专卖店买手机，买回来后发现是Android系统的概率是一样的。因为作者确实没有在生产环境中遇到过，除了自己作死写样例代码测试。先说一下异常出现的情况，前面讲到过，方法调用的过程就是方法帧进虚拟机栈和出虚拟机栈的过程，那么有两种情况可以导致StackOverFlowError,当一个方法帧(比如需要2M内存)进入到虚拟机栈(比如还剩下1M内存)的时候，就会报出StackOverFlow.这里先说一个概念，栈深度:指目前虚拟机栈中没有出栈的方法帧。虚拟机栈容量通过参数-Xss来控制,下面通过一段代码，把栈容量人为的调小一点，然后通过递归调用触发异常。

```
/**
 * VM Args：
    //设置栈容量为160K，默认1M
   -Xss160k
 */
public class JavaVMStackSOF {
    private int stackLength = 1;
    public void stackLeak() {
        stackLength++;
        //递归调用，触发异常
        stackLeak();
    }

    public static void main(String[] args) throws Throwable {
        JavaVMStackSOF oom = new JavaVMStackSOF();
        try {
            oom.stackLeak();
        } catch (Throwable e) {
            System.out.println("stack length:" + oom.stackLength);
            throw e;
        }
    }
}

```

> 结果如下:
> stack length:751 Exception in thread “main”
> java.lang.StackOverflowError

可以看到，递归调用了751次，栈容量不够用了。
默认的栈容量在正常的方法调用时，栈深度可以达到1000-2000深度，所以，一般的递归是可以承受的住的。如果你的代码出现了StackOverflowError，首先检查代码，而不是改参数。

这里顺带提一下，很多人在做多线程开发时，当创建很多线程时，**容易出现OOM(OutOfMemoryError),** 这时可以通过具体情况，减少最大堆容量，或者栈容量来解决问题，这是为什么呢。请看下面的公式:

<font color="red">线程数*(最大栈容量)+最大堆值+其他内存(忽略不计或者一般不改动)=机器最大内存</font>

当线程数比较多时，且无法通过业务上削减线程数，那么再不换机器的情况下，**你只能把最大栈容量设置小一点，或者把最大堆值设置小一点。**

### 方法区内存异常

写到这里时，作者本来想写一个无限创建动态代理对象的例子来演示方法区溢出，避开谈论JDK7与JDK8的内存区域变更的过渡，但细想一想，还是把这一块从始致终的说清楚。在上一篇文章中JVM系列之Java内存结构详解讲到方法区时提到，JDK7环境下方法区包括了(运行时常量池),其实这么说是不准确的。因为从JDK7开始，HotSpot团队就想到开始去"永久代",大家首先明确一个概念，**方法区和"永久代"(PermGen space)是两个概念，方法区是JVM虚拟机规范，任何虚拟机实现(J9等)都不能少这个区间，而"永久代"只是HotSpot对方法区的一个实现。** 为了把知识点列清楚，我还是才用列表的形式:

*   [ ]  JDK7之前(包括JDK7)拥有"永久代"(PermGen space),用来实现方法区。但在JDK7中已经逐渐在实现中把永久代中把很多东西移了出来，比如:符号引用(Symbols)转移到了native heap,运行时常量池(interned strings)转移到了java heap；类的静态变量(class statics)转移到了java heap.
*   [ ]  所以这就是为什么我说上一篇文章中说方法区中包含运行时常量池是不正确的，因为已经移动到了java heap;
    **在JDK7之前(包括7)可以通过-XX:PermSize -XX:MaxPermSize来控制永久代的大小.
    JDK8正式去除"永久代",换成Metaspace(元空间)作为JVM虚拟机规范中方法区的实现。**
    元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。因此，默认情况下，元空间的大小仅受本地内存限制，但仍可以通过参数控制:-XX:MetaspaceSize与-XX:MaxMetaspaceSize来控制大小。

### 方法区与运行时常量池OOM

Java 永久代是非堆内存的组成部分，用来存放类名、访问修饰符、常量池、字段描述、方法描述等，因运行时常量池是方法区的一部分，所以这里也包含运行时常量池。我们可以通过 jvm 参数 -XX：PermSize=10M -XX：MaxPermSize=10M 来指定该区域的内存大小，-XX：PermSize 默认为物理内存的 1/64 ,-XX：MaxPermSize 默认为物理内存的 1/4 。String.intern() 方法是一个 Native 方法，它的作用是：如果字符串常量池中已经包含一个等于此 String 对象的字符串，则返回代表池中这个字符串的 String 对象；否则，将此 String 对象包含的字符串添加到常量池中，并且返回此 String 对象的引用。在 JDK 1.6 及之前的版本中，由于常量池分配在永久代内，我们可以通过 -XX：PermSize 和 -XX：MaxPermSize 限制方法区大小，从而间接限制其中常量池的容量,通过运行 java -XX：PermSize=8M -XX：MaxPermSize=8M RuntimeConstantPoolOom 下面的代码我们可以模仿一个运行时常量池内存溢出的情况：

```
import java.util.ArrayList;
import java.util.List;

public class RuntimeConstantPoolOom {
  public static void main(String[] args) {
    List<String> list = new ArrayList<String>();
    int i = 0;
    while (true) {
      list.add(String.valueOf(i++).intern());
    }
  }
}

```

运行结果如下：

```
[root@9683817ada51 oom]# ../jdk1.6.0_45/bin/java -XX:PermSize=8m -XX:MaxPermSize=8m RuntimeConstantPoolOom
Exception in thread "main" java.lang.OutOfMemoryError: PermGen space
    at java.lang.String.intern(Native Method)
    at RuntimeConstantPoolOom.main(RuntimeConstantPoolOom.java:9)

```

还有一种情况就是我们可以通过不停的加载class来模拟方法区内存溢出，《深入理解java虚拟机》中借助 CGLIB 这类字节码技术模拟了这个异常，我们这里使用不同的 classloader 来实现（同一个类在不同的 classloader 中是不同的），代码如下

```
import java.io.File;
import java.net.MalformedURLException;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.HashSet;
import java.util.Set;

public class MethodAreaOom {
  public static void main(String[] args) throws MalformedURLException, ClassNotFoundException {
    Set<Class<?>> classes = new HashSet<Class<?>>();
    URL url = new File("").toURI().toURL();
    URL[] urls = new URL[]{url};
    while (true) {
      ClassLoader loader = new URLClassLoader(urls);
      Class<?> loadClass = loader.loadClass(Object.class.getName());
      classes.add(loadClass);
    }
  }
}

```

```
[root@9683817ada51 oom]# ../jdk1.6.0_45/bin/java -XX:PermSize=2m -XX:MaxPermSize=2m MethodAreaOom
Error occurred during initialization of VM
java.lang.OutOfMemoryError: PermGen space
    at sun.net.www.ParseUtil.<clinit>(ParseUtil.java:31)
    at sun.misc.Launcher.getFileURL(Launcher.java:476)
    at sun.misc.Launcher$ExtClassLoader.getExtURLs(Launcher.java:187)
    at sun.misc.Launcher$ExtClassLoader.<init>(Launcher.java:158)
    at sun.misc.Launcher$ExtClassLoader$1.run(Launcher.java:142)
    at java.security.AccessController.doPrivileged(Native Method)
    at sun.misc.Launcher$ExtClassLoader.getExtClassLoader(Launcher.java:135)
    at sun.misc.Launcher.<init>(Launcher.java:55)
    at sun.misc.Launcher.<clinit>(Launcher.java:43)
    at java.lang.ClassLoader.initSystemClassLoader(ClassLoader.java:1337)
    at java.lang.ClassLoader.getSystemClassLoader(ClassLoader.java:1319)

```

在 jdk1.8 上运行上面的代码将不会出现异常，因为 jdk1.8 已结去掉了永久代，当然 -XX:PermSize=2m -XX:MaxPermSize=2m 也将被忽略，如下

```
[root@9683817ada51 oom]# java -XX:PermSize=2m -XX:MaxPermSize=2m MethodAreaOom
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=2m; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=2m; support was removed in 8.0

```

jdk1.8 使用元空间（ Metaspace ）替代了永久代（ PermSize ），因此我们可以在 1.8 中指定 Metaspace 的大小模拟上述情况

```
[root@9683817ada51 oom]# java -XX:MetaspaceSize=2m -XX:MaxMetaspaceSize=2m RuntimeConstantPoolOom
Error occurred during initialization of VM
java.lang.OutOfMemoryError: Metaspace
    <<no stack trace available>>

```

在JDK8的环境下将报出异常:
Exception in thread “main” java.lang.OutOfMemoryError: Metaspace
这是因为在调用CGLib的创建代理时会生成动态代理类，即Class对象到Metaspace,所以While一下就出异常了。
**提醒一下:虽然我们日常叫"堆Dump",但是dump技术不仅仅是对于"堆"区域才有效，而是针对OOM的，也就是说不管什么区域，凡是能够报出OOM错误的，都可以使用dump技术生成dump文件来分析。**

在经常动态生成大量Class的应用中，需要特别注意类的回收状况，这类场景除了例子中的CGLib技术，常见的还有，大量JSP，反射，OSGI等。需要特别注意，当出现此类异常，应该知道是哪里出了问题，然后看是调整参数，还是在代码层面优化。

### 附加-直接内存异常

直接内存异常非常少见，而且机制很特殊，因为直接内存不是直接向操作系统分配内存，而且通过计算得到的内存不够而手动抛出异常，所以当你发现你的dump文件很小，而且没有明显异常，只是告诉你OOM，你就可以考虑下你代码里面是不是直接或者间接使用了NIO而导致直接内存溢出。

## Java内存泄漏

Java的一个重要优点就是通过垃圾收集器(Garbage Collection，GC)自动管理内存的回收，程序员不需要通过调用函数来释放内存。因此，很多程序员认为Java不存在内存泄漏问题，或者认为即使有内存泄漏也不是程序的责任，而是GC或JVM的问题。其实，这种想法是不正确的，因为Java也存在内存泄露，但它的表现与C++不同。

随着越来越多的服务器程序采用Java技术，例如JSP，Servlet， EJB等，服务器程序往往长期运行。另外，在很多嵌入式系统中，内存的总量非常有限。内存泄露问题也就变得十分关键，即使每次运行少量泄漏，长期运行之后，系统也是面临崩溃的危险。

## Java是如何管理内存？

为了判断Java中是否有内存泄露，我们首先必须了解Java是如何管理内存的。Java的内存管理就是对象的分配和释放问题。在Java中，程序员需要通过关键字new为每个对象申请内存空间 (基本类型除外)，所有的对象都在堆 (Heap)中分配空间。另外，对象的释放是由GC决定和执行的。在Java中，内存的分配是由程序完成的，而内存的释放是有GC完成的，这种收支两条线的方法确实简化了程序员的工作。但同时，它也加重了JVM的工作。这也是Java程序运行速度较慢的原因之一。因为，GC为了能够正确释放对象，GC必须监控每一个对象的运行状态，包括对象的申请、引用、被引用、赋值等，GC都需要进行监控。

监视对象状态是为了更加准确地、及时地释放对象，而释放对象的根本原则就是该对象不再被引用。

为了更好理解GC的工作原理，我们可以将对象考虑为有向图的顶点，将引用关系考虑为图的有向边，有向边从引用者指向被引对象。另外，每个线程对象可以作为一个图的起始顶点，例如大多程序从main进程开始执行，那么该图就是以main进程顶点开始的一棵根树。在这个有向图中，根顶点可达的对象都是有效对象，GC将不回收这些对象。如果某个对象 (连通子图)与这个根顶点不可达(注意，该图为有向图)，那么我们认为这个(这些)对象不再被引用，可以被GC回收。

以下，我们举一个例子说明如何用有向图表示内存管理。对于程序的每一个时刻，我们都有一个有向图表示JVM的内存分配情况。以下右图，就是左边程序运行到第6行的示意图。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110151433131.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzE5MjczMg==,size_16,color_FFFFFF,t_70)
Java使用有向图的方式进行内存管理，可以消除引用循环的问题，例如有三个对象，相互引用，只要它们和根进程不可达的，那么GC也是可以回收它们的。这种方式的优点是管理内存的精度很高，但是效率较低。另外一种常用的内存管理技术是使用计数器，例如COM模型采用计数器方式管理构件，它与有向图相比，精度行低(很难处理循环引用的问题)，但执行效率很高。

## 什么是Java中的内存泄露？

下面，我们就可以描述什么是内存泄漏。在Java中，内存泄漏就是存在一些被分配的对象，这些对象有下面两个特点，首先，这些对象是可达的，即在有向图中，存在通路可以与其相连；其次，这些对象是无用的，即程序以后不会再使用这些对象。如果对象满足这两个条件，这些对象就可以判定为Java中的内存泄漏，这些对象不会被GC所回收，然而它却占用内存。

在C++中，内存泄漏的范围更大一些。有些对象被分配了内存空间，然后却不可达，由于C++中没有GC，这些内存将永远收不回来。在Java中，这些不可达的对象都由GC负责回收，因此程序员不需要考虑这部分的内存泄露。

通过分析，我们得知，对于C++，程序员需要自己管理边和顶点，而对于Java程序员只需要管理边就可以了(不需要管理顶点的释放)。通过这种方式，Java提高了编程的效率。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190110152226472.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzE5MjczMg==,size_16,color_FFFFFF,t_70)
因此，通过以上分析，我们知道在Java中也有内存泄漏，但范围比C++要小一些。因为Java从语言上保证，任何对象都是可达的，所有的不可达对象都由GC管理。

对于程序员来说，GC基本是透明的，不可见的。虽然，我们只有几个函数可以访问GC，例如运行GC的函数System.gc()，但是根据Java语言规范定义， 该函数不保证JVM的垃圾收集器一定会执行。因为，不同的JVM实现者可能使用不同的算法管理GC。通常，GC的线程的优先级别较低。JVM调用GC的策略也有很多种，有的是内存使用到达一定程度时，GC才开始工作，也有定时执行的，有的是平缓执行GC，有的是中断式执行GC。但通常来说，我们不需要关心这些。除非在一些特定的场合，GC的执行影响应用程序的性能，例如对于基于Web的实时系统，如网络游戏等，用户不希望GC突然中断应用程序执行而进行垃圾回收，那么我们需要调整GC的参数，让GC能够通过平缓的方式释放内存，例如将垃圾回收分解为一系列的小步骤执行，Sun提供的HotSpot JVM就支持这一特性。

下面给出了一个简单的内存泄露的例子。在这个例子中，我们循环申请Object对象，并将所申请的对象放入一个Vector中，如果我们仅仅释放引用本身，那么Vector仍然引用该对象，所以这个对象对GC来说是不可回收的。因此，如果对象加入到Vector后，还必须从Vector中删除，最简单的方法就是将Vector对象设置为null。

```
Vector v=new Vector(10);
for (int i=1;i<100; i++)
{
    Object o=new Object();
    v.add(o);
    o=null;
}
//此时，所有的Object对象都没有被释放，因为变量v引用这些对象

```

### 其他常见内存泄漏

#### 1、静态集合类引起内存泄露：

像HashMap、Vector等的使用最容易出现内存泄露，这些静态变量的生命周期和应用程序一致，他们所引用的所有的对象Object也不能被释放，因为他们也将一直被Vector等引用着。
例:

```
Static Vector v = new Vector(10); 
for (int i = 1; i<100; i++) { 
	Object o = new Object(); 
	v.add(o); 
	o = null; 
}// 
在这个例子中，循环申请Object 对象，并将所申请的对象放入一个Vector 中，如果仅仅释放引用本身（o=null），那么Vector 仍然引用该对象，所以这个对象对GC 来说是不可回收的。因此，如果对象加入到Vector 后，还必须从Vector 中删除，最简单的方法就是将Vector对象设置为null。

```

#### 2、当集合里面的对象属性被修改后，再调用remove（）方法时不起作用。

例：

```
public static void main(String[] args) { 
	Set<Person> set = new HashSet<Person>(); 
	Person p1 = new Person("唐僧","pwd1",25); 
	Person p2 = new Person("孙悟空","pwd2",26); 
	Person p3 = new Person("猪八戒","pwd3",27); 
	set.add(p1); 
	set.add(p2); 
	set.add(p3); 
	System.out.println("总共有:"+set.size()+" 个元素!"); //结果：总共有:3 个元素! 
	p3.setAge(2); //修改p3的年龄,此时p3元素对应的hashcode值发生改变 

	set.remove(p3); //此时remove不掉，造成内存泄漏
	set.add(p3); //重新添加，居然添加成功 
	System.out.println("总共有:"+set.size()+" 个元素!"); //结果：总共有:4 个元素! 
	for (Person person : set) { 
		System.out.println(person); 
	} 
}

```

#### 3、监听器

在java 编程中，我们都需要和监听器打交道，通常一个应用当中会用到很多监听器，我们会调用一个控件的诸如addXXXListener()等方法来增加监听器，但往往在释放对象的时候却没有记住去删除这些监听器，从而增加了内存泄漏的机会。

#### 4、各种连接

比如数据库连接（dataSourse.getConnection()），网络连接(socket)和io连接，除非其显式的调用了其close（）方法将其连接关闭，否则是不会自动被GC 回收的。对于Resultset 和Statement 对象可以不进行显式回收，但Connection 一定要显式回收，因为Connection 在任何时候都无法自动回收，而Connection一旦回收，Resultset 和Statement 对象就会立即为NULL。但是如果使用连接池，情况就不一样了，除了要显式地关闭连接，还必须显式地关闭Resultset Statement 对象（关闭其中一个，另外一个也会关闭），否则就会造成大量的Statement 对象无法释放，从而引起内存泄漏。这种情况下一般都会在try里面去的连接，在finally里面释放连接。

#### 5、内部类和外部模块等的引用

内部类的引用是比较容易遗忘的一种，而且一旦没释放可能导致一系列的后继类对象没有释放。此外程序员还要小心外部模块不经意的引用，例如程序员A 负责A 模块，调用了B 模块的一个方法如：
public void registerMsg(Object b);
这种调用就要非常小心了，传入了一个对象，很可能模块B就保持了对该对象的引用，这时候就需要注意模块B 是否提供相应的操作去除引用。

#### 6、单例模式

不正确使用单例模式是引起内存泄露的一个常见问题，单例对象在被初始化后将在JVM的整个生命周期中存在（以静态变量的方式），如果单例对象持有外部对象的引用，那么这个外部对象将不能被jvm正常回收，导致内存泄露，考虑下面的例子：

```
class A{ 
	public A(){ 
		B.getInstance().setA(this); 
	} 
.... 
} 
//B类采用单例模式 
class B{ 
	private A a; 
	private static B instance=new B(); 
	public B(){} 
	public static B getInstance(){ 
		return instance; 
	} 
	public void setA(A a){ 
		this.a=a; 
	} 
	//getter... 
} 

```

**显然B采用singleton模式，它持有一个A对象的引用，而这个A类的对象将不能被回收。想象下如果A是个比较复杂的对象或者集合类型会发生什么情况。**

## 如何检测内存泄漏

最后一个重要的问题，就是如何检测Java的内存泄漏。目前，我们通常使用一些工具来检查Java程序的内存泄漏问题。市场上已有几种专业检查Java内存泄漏的工具，它们的基本工作原理大同小异，都是通过监测Java程序运行时，所有对象的申请、释放等动作，将内存管理的所有信息进行统计、分析、可视化。开发人员将根据这些信息判断程序是否有内存泄漏问题。这些工具包括Optimizeit Profiler，JProbe Profiler，JinSight , Rational 公司的Purify等。

下面，我们将简单介绍Optimizeit的基本功能和工作原理。

Optimizeit Profiler版本4.11支持Application，Applet，Servlet和Romote Application四类应用，并且可以支持大多数类型的JVM，包括SUN JDK系列，IBM的JDK系列，和Jbuilder的JVM等。并且，该软件是由Java编写，因此它支持多种操作系统。Optimizeit系列还包括Thread Debugger和Code Coverage两个工具，分别用于监测运行时的线程状态和代码覆盖面。

当设置好所有的参数了，我们就可以在OptimizeIt环境下运行被测程序，在程序运行过程中，Optimizeit可以监视内存的使用曲线(如下图)，包括JVM申请的堆(heap)的大小，和实际使用的内存大小。另外，在运行过程中，我们可以随时暂停程序的运行，甚至强行调用GC，让GC进行内存回收。通过内存使用曲线，我们可以整体了解程序使用内存的情况。这种监测对于长期运行的应用程序非常有必要，也很容易发现内存泄露。








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
