# Table of Contents

  * [一、背景](#一、背景)
  * [二、简介](#二、简介)
    * [1.强引用 StrongReference](#1强引用-strongreference)
    * [2.弱引用 WeakReference](#2弱引用-weakreference)
    * [3.软引用 SoftReference](#3软引用-softreference)
    * [4.虚引用 PhantomReference](#4虚引用-phantomreference)
  * [三、小结](#三、小结)
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

## 一、背景

Java的内存回收不需要程序员负责，JVM会在必要时启动Java GC完成垃圾回收。Java以便我们控制对象的生存周期，提供给了我们四种引用方式，引用强度从强到弱分别为：强引用、软引用、弱引用、虚引用。 

## 二、简介

### 1.强引用 StrongReference

StrongReference是Java的默认引用形式，使用时不需要显示定义。任何通过强引用所使用的对象不管系统资源有多紧张，Java GC都不会主动回收具有强引用的对象。

<pre>public class StrongReferenceTest {

	public static int M = 1024*1024;

	public static void printlnMemory(String tag){
		Runtime runtime = Runtime.getRuntime();
		int M = StrongReferenceTest.M;
		System.out.println("\n"+tag+":");
		System.out.println(runtime.freeMemory()/M+"M(free)/" + runtime.totalMemory()/M+"M(total)");
	}

	public static void main(String[] args){
		StrongReferenceTest.printlnMemory("1.原可用内存和总内存");

		//实例化10M的数组并与strongReference建立强引用
		byte[] strongReference = new byte[10*StrongReferenceTest.M];
		StrongReferenceTest.printlnMemory("2.实例化10M的数组,并建立强引用");
		System.out.println("strongReference : "+strongReference);

		System.gc();
		StrongReferenceTest.printlnMemory("3.GC后");
		System.out.println("strongReference : "+strongReference);

		//strongReference = null;后,强引用断开了
		strongReference = null;
		StrongReferenceTest.printlnMemory("4.强引用断开后");
		System.out.println("strongReference : "+strongReference);

		System.gc();
		StrongReferenceTest.printlnMemory("5.GC后");
		System.out.println("strongReference : "+strongReference);
		}
}
</pre>

运行结果：

![](https://user-gold-cdn.xitu.io/2018/1/7/160cd0dc536b2384?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 2.弱引用 WeakReference

如果一个对象只具有弱引用，无论内存充足与否，Java GC后对象如果只有弱引用将会被自动回收。

<pre>public class WeakReferenceTest {

	public static int M = 1024*1024;

	public static void printlnMemory(String tag){
		Runtime runtime = Runtime.getRuntime();
		int M = WeakReferenceTest.M;
		System.out.println("\n"+tag+":");
		System.out.println(runtime.freeMemory()/M+"M(free)/" + runtime.totalMemory()/M+"M(total)");
	}

	public static void main(String[] args){  
		WeakReferenceTest.printlnMemory("1.原可用内存和总内存");

		//创建弱引用
		WeakReference<Object> weakRerference = new WeakReference<Object>(new byte[10*WeakReferenceTest.M]);   
		WeakReferenceTest.printlnMemory("2.实例化10M的数组,并建立弱引用");
		System.out.println("weakRerference.get() : "+weakRerference.get());

		System.gc();
		StrongReferenceTest.printlnMemory("3.GC后");
		System.out.println("weakRerference.get() : "+weakRerference.get());
	}   
}
</pre>

运行结果：

![](https://user-gold-cdn.xitu.io/2018/1/7/160cd0f1ead8184e?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 3.软引用 SoftReference

软引用和弱引用的特性基本一致， 主要的区别在于软引用在内存不足时才会被回收。如果一个对象只具有软引用，Java GC在内存充足的时候不会回收它，内存不足时才会被回收。

<pre>public class SoftReferenceTest {

	public static int M = 1024*1024;

	public static void printlnMemory(String tag){
		Runtime runtime = Runtime.getRuntime();
		int M = StrongReferenceTest.M;
		System.out.println("\n"+tag+":");
		System.out.println(runtime.freeMemory()/M+"M(free)/" + runtime.totalMemory()/M+"M(total)");
	}

	public static void main(String[] args){
		SoftReferenceTest.printlnMemory("1.原可用内存和总内存");

		//建立软引用
		SoftReference<Object> softRerference = new SoftReference<Object>(new byte[10*SoftReferenceTest.M]);
		SoftReferenceTest.printlnMemory("2.实例化10M的数组,并建立软引用");
		System.out.println("softRerference.get() : "+softRerference.get());

		System.gc();  
		SoftReferenceTest.printlnMemory("3.内存可用容量充足，GC后");
		System.out.println("softRerference.get() : "+softRerference.get());  

		//实例化一个4M的数组,使内存不够用,并建立软引用
		//free=10M=4M+10M-4M,证明内存可用量不足时，GC后byte[10*m]被回收
		SoftReference<Object> softRerference2 = new SoftReference<Object>(new byte[4*SoftReferenceTest.M]);
		SoftReferenceTest.printlnMemory("4.实例化一个4M的数组后");
		System.out.println("softRerference.get() : "+softRerference.get());
		System.out.println("softRerference2.get() : "+softRerference2.get());  
	 } 
}
</pre>

运行结果：

![](https://user-gold-cdn.xitu.io/2018/1/7/160cd1023a8f3de2?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 4.虚引用 PhantomReference

从PhantomReference类的源代码可以知道，它的get()方法无论何时返回的都只会是null。所以单独使用虚引用时，没有什么意义，需要和引用队列ReferenceQueue类联合使用。当执行Java GC时如果一个对象只有虚引用，就会把这个对象加入到与之关联的ReferenceQueue中。

<pre>public class PhantomReferenceTest {

	public static int M = 1024*1024;

	public static void printlnMemory(String tag){
		Runtime runtime = Runtime.getRuntime();
		int M = PhantomReferenceTest.M;
		System.out.println("\n"+tag+":");
		System.out.println(runtime.freeMemory()/M+"M(free)/" + runtime.totalMemory()/M+"M(total)");
	}

	public static void main(String[] args) throws InterruptedException {

		PhantomReferenceTest.printlnMemory("1.原可用内存和总内存");
		byte[] object = new byte[10*PhantomReferenceTest.M];		
		PhantomReferenceTest.printlnMemory("2.实例化10M的数组后");

	    //建立虚引用
	    ReferenceQueue<Object> referenceQueue = new ReferenceQueue<Object>();
	    PhantomReference<Object> phantomReference = new PhantomReference<Object>(object,referenceQueue);  

	    PhantomReferenceTest.printlnMemory("3.建立虚引用后");
	    System.out.println("phantomReference : "+phantomReference); 
	    System.out.println("phantomReference.get() : "+phantomReference.get());
	    System.out.println("referenceQueue.poll() : "+referenceQueue.poll());

	    //断开byte[10*PhantomReferenceTest.M]的强引用
	    object = null;  
	    PhantomReferenceTest.printlnMemory("4.执行object = null;强引用断开后");

	    System.gc();
	    PhantomReferenceTest.printlnMemory("5.GC后");
	    System.out.println("phantomReference : "+phantomReference); 
	    System.out.println("phantomReference.get() : "+phantomReference.get());
	    System.out.println("referenceQueue.poll() : "+referenceQueue.poll());	    

	    //断开虚引用
	    phantomReference = null;
		System.gc(); 
		PhantomReferenceTest.printlnMemory("6.断开虚引用后GC");
	    System.out.println("phantomReference : "+phantomReference);
	    System.out.println("referenceQueue.poll() : "+referenceQueue.poll());	    	
	}
}
</pre>

运行结果：

![](https://user-gold-cdn.xitu.io/2018/1/7/160cd110cba23d84?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

## 三、小结

强引用是 Java 的默认引用形式，使用时不需要显示定义，是我们平时最常使用到的引用方式。不管系统资源有多紧张，Java GC都不会主动回收具有强引用的对象。 弱引用和软引用一般在引用对象为非必需对象的时候使用。它们的区别是被弱引用关联的对象在垃圾回收时总是会被回收，被软引用关联的对象只有在内存不足时才会被回收。 虚引用的get()方法获取的永远是null，无法获取对象实例。Java GC会把虚引用的对象放到引用队列里面。可用来在对象被回收时做额外的一些资源清理或事物回滚等处理。 由于无法从虚引获取到引用对象的实例。它的使用情况比较特别，所以这里不把虚引用放入表格进行对比。这里对强引用、弱引用、软引用进行对比：



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
