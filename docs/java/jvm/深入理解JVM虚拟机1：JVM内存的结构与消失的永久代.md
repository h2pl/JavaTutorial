# Table of Contents

  * [前言](#前言)
  * [Java堆（Heap）](#java堆（heap）)
  * [方法区（Method Area）](#方法区（method-area）)
  * [程序计数器（Program Counter Register）](#程序计数器（program-counter-register）)
  * [JVM栈（JVM Stacks）](#jvm栈（jvm-stacks）)
  * [本地方法栈（Native Method Stacks）](#本地方法栈（native-method-stacks）)
  * [哪儿的OutOfMemoryError](#哪儿的outofmemoryerror)
* [[JDK8-废弃永久代（PermGen）迎来元空间（Metaspace）](https://www.cnblogs.com/yulei126/p/6777323.html)](#[jdk8-废弃永久代（permgen）迎来元空间（metaspace）]httpswwwcnblogscomyulei126p6777323html)
  * [一、背景](#一、背景)
    * [1.1 永久代（PermGen）在哪里？](#11-永久代（permgen）在哪里？)
    * [1.2 JDK8永久代的废弃](#12-jdk8永久代的废弃)
  * [ 二、为什么废弃永久代（PermGen）](# 二、为什么废弃永久代（permgen）)
    * [ 2.1 官方说明](# 21-官方说明)
  * [Motivation](#motivation)
    * [ 2.2 现实使用中易出问题](# 22-现实使用中易出问题)
  * [三、深入理解元空间（Metaspace）](#三、深入理解元空间（metaspace）)
    * [3.1元空间的内存大小](#31元空间的内存大小)
    * [3.2常用配置参数](#32常用配置参数)
    * [3.3测试并追踪元空间大小](#33测试并追踪元空间大小)
      * [ 3.3.1.测试字符串常量](# 331测试字符串常量)
      * [3.3.2.测试元空间溢出](#332测试元空间溢出)
  * [ 四、总结](# 四、总结)
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

## 前言

所有的Java开发人员可能会遇到这样的困惑？我该为堆内存设置多大空间呢？OutOfMemoryError的异常到底涉及到运行时数据的哪块区域？该怎么解决呢？

其实如果你经常解决服务器性能问题，那么这些问题就会变的非常常见，了解JVM内存也是为了服务器出现性能问题的时候可以快速的了解那块的内存区域出现问题，以便于快速的解决生产故障。



先看一张图，这张图能很清晰的说明JVM内存结构布局。

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036730192392.png)

JVM内存结构主要有三大块：堆内存、方法区和栈。堆内存是JVM中最大的一块由年轻代和老年代组成，而年轻代内存又被分成三部分，Eden空间、From Survivor空间、To Survivor空间,默认情况下年轻代按照8:1:1的比例来分配；

方法区存储类信息、常量、静态变量等数据，是线程共享的区域，为与Java堆区分，方法区还有一个别名Non-Heap(非堆)；栈又分为java虚拟机栈和本地方法栈主要用于方法的执行。

在通过一张图来了解如何通过参数来控制各区域的内存大小

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036730949557.png)

控制参数

*   -Xms设置堆的最小空间大小。
*   -Xmx设置堆的最大空间大小。
*   -XX:NewSize设置新生代最小空间大小。
*   -XX:MaxNewSize设置新生代最大空间大小。
*   -XX:PermSize设置永久代最小空间大小。
*   -XX:MaxPermSize设置永久代最大空间大小。
*   -Xss设置每个线程的堆栈大小。

没有直接设置老年代的参数，但是可以设置堆空间大小和新生代空间大小两个参数来间接控制。

> 老年代空间大小=堆空间大小-年轻代大空间大小

从更高的一个维度再次来看JVM和系统调用之间的关系

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036732641186.png)

方法区和对是所有线程共享的内存区域；而java栈、本地方法栈和程序员计数器是运行是线程私有的内存区域。

下面我们详细介绍每个区域的作用

## Java堆（Heap）

对于大多数应用来说，Java堆（Java Heap）是Java虚拟机所管理的内存中最大的一块。Java堆是被所有线程共享的一块内存区域，在虚拟机启动时创建。此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例都在这里分配内存。

Java堆是垃圾收集器管理的主要区域，因此很多时候也被称做“GC堆”。如果从内存回收的角度看，由于现在收集器基本都是采用的分代收集算法，所以Java堆中还可以细分为：新生代和老年代；再细致一点的有Eden空间、From Survivor空间、To Survivor空间等。

根据Java虚拟机规范的规定，Java堆可以处于物理上不连续的内存空间中，只要逻辑上是连续的即可，就像我们的磁盘空间一样。在实现时，既可以实现成固定大小的，也可以是可扩展的，不过当前主流的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。

如果在堆中没有内存完成实例分配，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

## 方法区（Method Area）

方法区（Method Area）与Java堆一样，是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据。虽然Java虚拟机规范把方法区描述为堆的一个逻辑部分，但是它却有一个别名叫做Non-Heap（非堆），目的应该是与Java堆区分开来。

对于习惯在HotSpot虚拟机上开发和部署程序的开发者来说，很多人愿意把方法区称为“永久代”（Permanent Generation），本质上两者并不等价，仅仅是因为HotSpot虚拟机的设计团队选择把GC分代收集扩展至方法区，或者说使用永久代来实现方法区而已。

Java虚拟机规范对这个区域的限制非常宽松，除了和Java堆一样不需要连续的内存和可以选择固定大小或者可扩展外，还可以选择不实现垃圾收集。相对而言，垃圾收集行为在这个区域是比较少出现的，但并非数据进入了方法区就如永久代的名字一样“永久”存在了。这个区域的内存回收目标主要是针对常量池的回收和对类型的卸载，一般来说这个区域的回收“成绩”比较难以令人满意，尤其是类型的卸载，条件相当苛刻，但是这部分区域的回收确实是有必要的。

根据Java虚拟机规范的规定，当方法区无法满足内存分配需求时，将抛出OutOfMemoryError异常。

方法区有时被称为持久代（PermGen）。

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036749972827.png)

所有的对象在实例化后的整个运行周期内，都被存放在堆内存中。堆内存又被划分成不同的部分：伊甸区(Eden)，幸存者区域(Survivor Sapce)，老年代（Old Generation Space）。

方法的执行都是伴随着线程的。原始类型的本地变量以及引用都存放在线程栈中。而引用关联的对象比如String，都存在在堆中。为了更好的理解上面这段话，我们可以看一个例子：





<pre>import java.text.SimpleDateFormat;import java.util.Date;import org.apache.log4j.Logger;
 public class HelloWorld {
    private static Logger LOGGER = Logger.getLogger(HelloWorld.class.getName());
    public void sayHello(String message) {
        SimpleDateFormat formatter = new SimpleDateFormat("dd.MM.YYYY");
        String today = formatter.format(new Date());
        LOGGER.info(today + ": " + message);
    }}</pre>





这段程序的数据在内存中的存放如下：

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036766406898.png)

通过JConsole工具可以查看运行中的Java程序（比如Eclipse）的一些信息：堆内存的分配，线程的数量以及加载的类的个数；

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036768475136.png)

## 程序计数器（Program Counter Register）

程序计数器（Program Counter Register）是一块较小的内存空间，它的作用可以看做是当前线程所执行的字节码的行号指示器。在虚拟机的概念模型里（仅是概念模型，各种虚拟机可能会通过一些更高效的方式去实现），字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等基础功能都需要依赖这个计数器来完成。

由于Java虚拟机的多线程是通过线程轮流切换并分配处理器执行时间的方式来实现的，在任何一个确定的时刻，一个处理器（对于多核处理器来说是一个内核）只会执行一条线程中的指令。因此，为了线程切换后能恢复到正确的执行位置，每条线程都需要有一个独立的程序计数器，各条线程之间的计数器互不影响，独立存储，我们称这类内存区域为“线程私有”的内存。

如果线程正在执行的是一个Java方法，这个计数器记录的是正在执行的虚拟机字节码指令的地址；如果正在执行的是Natvie方法，这个计数器值则为空（Undefined）。

此内存区域是唯一一个在Java虚拟机规范中没有规定任何OutOfMemoryError情况的区域。

## JVM栈（JVM Stacks）

与程序计数器一样，Java虚拟机栈（Java Virtual Machine Stacks）也是线程私有的，它的生命周期与线程相同。虚拟机栈描述的是Java方法执行的内存模型：每个方法被执行的时候都会同时创建一个栈帧（Stack Frame）用于存储局部变量表、操作栈、动态链接、方法出口等信息。每一个方法被调用直至执行完成的过程，就对应着一个栈帧在虚拟机栈中从入栈到出栈的过程。

局部变量表存放了编译期可知的各种基本数据类型（boolean、byte、char、short、int、float、long、double）、对象引用（reference类型，它不等同于对象本身，根据不同的虚拟机实现，它可能是一个指向对象起始地址的引用指针，也可能指向一个代表对象的句柄或者其他与此对象相关的位置）和returnAddress类型（指向了一条字节码指令的地址）。

其中64位长度的long和double类型的数据会占用2个局部变量空间（Slot），其余的数据类型只占用1个。局部变量表所需的内存空间在编译期间完成分配，当进入一个方法时，这个方法需要在帧中分配多大的局部变量空间是完全确定的，在方法运行期间不会改变局部变量表的大小。

在Java虚拟机规范中，对这个区域规定了两种异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果虚拟机栈可以动态扩展（当前大部分的Java虚拟机都可动态扩展，只不过Java虚拟机规范中也允许固定长度的虚拟机栈），当扩展时无法申请到足够的内存时会抛出OutOfMemoryError异常。

## 本地方法栈（Native Method Stacks）

本地方法栈（Native Method Stacks）与虚拟机栈所发挥的作用是非常相似的，其区别不过是虚拟机栈为虚拟机执行Java方法（也就是字节码）服务，而本地方法栈则是为虚拟机使用到的Native方法服务。虚拟机规范中对本地方法栈中的方法使用的语言、使用方式与数据结构并没有强制规定，因此具体的虚拟机可以自由实现它。甚至有的虚拟机（譬如Sun HotSpot虚拟机）直接就把本地方法栈和虚拟机栈合二为一。与虚拟机栈一样，本地方法栈区域也会抛出StackOverflowError和OutOfMemoryError异常。

## 哪儿的OutOfMemoryError

对内存结构清晰的认识同样可以帮助理解不同OutOfMemoryErrors：





<pre>Exception in thread “main”: java.lang.OutOfMemoryError: Java heap space</pre>





原因：对象不能被分配到堆内存中





<pre>Exception in thread “main”: java.lang.OutOfMemoryError: PermGen space</pre>





原因：类或者方法不能被加载到持久代。它可能出现在一个程序加载很多类的时候，比如引用了很多第三方的库；





<pre>Exception in thread “main”: java.lang.OutOfMemoryError: Requested array size exceeds VM limit</pre>





原因：创建的数组大于堆内存的空间





<pre>Exception in thread “main”: java.lang.OutOfMemoryError: request <size> bytes for <reason>. Out of swap space?</pre>





原因：分配本地分配失败。JNI、本地库或者Java虚拟机都会从本地堆中分配内存空间。





<pre>Exception in thread “main”: java.lang.OutOfMemoryError: <reason> <stack trace>（Native method）</pre>





原因：同样是本地方法内存分配失败，只不过是JNI或者本地方法或者Java虚拟机发现

# [JDK8-废弃永久代（PermGen）迎来元空间（Metaspace）](https://www.cnblogs.com/yulei126/p/6777323.html)





1.背景

2.为什么废弃永久代（PermGen）

3.深入理解元空间（Metaspace）

4.总结

========正文分割线=====

## 一、背景

### 1.1 永久代（PermGen）在哪里？

根据，hotspot jvm结构如下(虚拟机栈和本地方法栈合一起了)：

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036768860207.png)

上图引自网络，但有个问题：方法区和heap堆都是线程共享的内存区域。

关于方法区和永久代：

在HotSpot JVM中，这次讨论的永久代，就是上图的方法区（JVM规范中称为方法区）。《Java虚拟机规范》只是规定了有方法区这么个概念和它的作用，并没有规定如何去实现它。在其他JVM上不存在永久代。

### 1.2 JDK8永久代的废弃

JDK8 永久代变化如下图：

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036768834803.jpg)

1.新生代：Eden+From Survivor+To Survivor

2.老年代：OldGen

3.永久代（方法区的实现） : PermGen----->替换为Metaspace(本地内存中)

##  二、为什么废弃永久代（PermGen）

###  2.1 官方说明

参照JEP122：http://openjdk.java.net/jeps/122，原文截取：

## Motivation

This is part of the JRockit and Hotspot convergence effort. JRockit customers do not need to configure the permanent generation (since JRockit does not have a permanent generation) and are accustomed to not configuring the permanent generation.

 即：移除永久代是为融合HotSpot JVM与 JRockit VM而做出的努力，因为JRockit没有永久代，不需要配置永久代。

###  2.2 现实使用中易出问题

由于永久代内存经常不够用或发生内存泄露，爆出异常java.lang.OutOfMemoryError: PermGen

其实在JDK7时就已经逐步把永久代的内容移动到其他区域了，比如移动到native区，移动到堆区等，而JDK8则是则是废除了永久代，改用元数据。

## 三、深入理解元空间（Metaspace）

### 3.1元空间的内存大小

元空间是方法区的在HotSpot jvm 中的实现，方法区主要用于存储类的信息、常量池、方法数据、方法代码等。方法区逻辑上属于堆的一部分，但是为了与堆进行区分，通常又叫“非堆”。

元空间的本质和永久代类似，都是对JVM规范中方法区的实现。不过元空间与永久代之间最大的区别在于：元空间并不在虚拟机中，而是使用本地内存。，理论上取决于32位/64位系统可虚拟的内存大小。可见也不是无限制的，需要配置参数。

### 3.2常用配置参数

1.MetaspaceSize

初始化的Metaspace大小，控制元空间发生GC的阈值。GC后，动态增加或降低MetaspaceSize。在默认情况下，这个值大小根据不同的平台在12M到20M浮动。使用[Java](http://lib.csdn.net/base/javase "Java SE知识库") -XX:+PrintFlagsInitial命令查看本机的初始化参数

2.MaxMetaspaceSize

限制Metaspace增长的上限，防止因为某些情况导致Metaspace无限的使用本地内存，影响到其他程序。在本机上该参数的默认值为4294967295B（大约4096MB）。

3.MinMetaspaceFreeRatio

当进行过Metaspace GC之后，会计算当前Metaspace的空闲空间比，如果空闲比小于这个参数（即实际非空闲占比过大，内存不够用），那么虚拟机将增长Metaspace的大小。默认值为40，也就是40%。设置该参数可以控制Metaspace的增长的速度，太小的值会导致Metaspace增长的缓慢，Metaspace的使用逐渐趋于饱和，可能会影响之后类的加载。而太大的值会导致Metaspace增长的过快，浪费内存。

4.MaxMetasaceFreeRatio

当进行过Metaspace GC之后， 会计算当前Metaspace的空闲空间比，如果空闲比大于这个参数，那么虚拟机会释放Metaspace的部分空间。默认值为70，也就是70%。

5.MaxMetaspaceExpansion

Metaspace增长时的最大幅度。在本机上该参数的默认值为5452592B（大约为5MB）。

6.MinMetaspaceExpansion

Metaspace增长时的最小幅度。在本机上该参数的默认值为340784B（大约330KB为）。

### 3.3测试并追踪元空间大小

####  3.3.1.测试字符串常量






<pre> 1 public class StringOomMock {
 2     static String  base = "string";
 3     
 4     public static void main(String[] args) {
 5         List<String> list = new ArrayList<String>();
 6         for (int i=0;i< Integer.MAX_VALUE;i++){
 7             String str = base + base;
 8             base = str;
 9             list.add(str.intern());
10         }
11     }
12 }</pre>






在eclipse中选中类--》run configuration-->java application--》new 参数如下：

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036770114377.png)

 由于设定了最大内存20M，很快就溢出，如下图：

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036770308300.png)

 可见在jdk8中：

1.字符串常量由永久代转移到堆中。

2.持久代已不存在，PermSize MaxPermSize参数已移除。（看图中最后两行）

#### 3.3.2.测试元空间溢出

根据定义，我们以加载类来测试元空间溢出，代码如下：






<pre> 1 package jdk8;
 2 
 3 import java.io.File;
 4 import java.lang.management.ClassLoadingMXBean;
 5 import java.lang.management.ManagementFactory;
 6 import java.net.URL;
 7 import java.net.URLClassLoader;
 8 import java.util.ArrayList;
 9 import java.util.List;
10 
11 /**
12  * 
13  * @ClassName:OOMTest
14  * @Description:模拟类加载溢出（元空间oom）
15  * @author diandian.zhang
16  * @date 2017年4月27日上午9:45:40
17  */
18 public class OOMTest {  
19     public static void main(String[] args) {  
20         try {  
21             //准备url  
22             URL url = new File("D:/58workplace/11study/src/main/java/jdk8").toURI().toURL();  
23             URL[] urls = {url};  
24             //获取有关类型加载的JMX接口  
25             ClassLoadingMXBean loadingBean = ManagementFactory.getClassLoadingMXBean();  
26             //用于缓存类加载器  
27             List<ClassLoader> classLoaders = new ArrayList<ClassLoader>();  
28             while (true) {  
29                 //加载类型并缓存类加载器实例  
30                 ClassLoader classLoader = new URLClassLoader(urls);  
31                 classLoaders.add(classLoader);  
32                 classLoader.loadClass("ClassA");  
33                 //显示数量信息（共加载过的类型数目，当前还有效的类型数目，已经被卸载的类型数目）  
34                 System.out.println("total: " + loadingBean.getTotalLoadedClassCount());  
35                 System.out.println("active: " + loadingBean.getLoadedClassCount());  
36                 System.out.println("unloaded: " + loadingBean.getUnloadedClassCount());  
37             }  
38         } catch (Exception e) {  
39             e.printStackTrace();  
40         }  
41     }  
42 }  </pre>






为了快速溢出，设置参数：-XX:MetaspaceSize=8m -XX:MaxMetaspaceSize=80m，运行结果如下：

![](http://blog.itpub.net/ueditor/php/upload/image/20190817/1566036772261654.png)

 上图证实了，我们的JDK8中类加载（方法区的功能）已经不在永久代PerGem中了，而是Metaspace中。可以配合JVisualVM来看，更直观一些。

##  四、总结

本文讲解了元空间（Metaspace）的由来和本质，常用配置，以及监控测试。元空间的大小是动态变更的，但不是无限大的，最好也时常关注一下大小，以免影响服务器内存。








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
