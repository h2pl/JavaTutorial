# 目录

  * [JVM优化的必要性](#jvm优化的必要性)
  * [JVM调优原则](#jvm调优原则)
    * [JVM运行参数设置](#jvm运行参数设置)
  * [JVM性能调优工具](#jvm性能调优工具)
  * [常用调优策略](#常用调优策略)
  * [六、JVM调优实例](#六、jvm调优实例)
  * [七、一个具体的实战案例分析](#七、一个具体的实战案例分析)
  * [参考资料](#参考资料)
  * [参考文章](#参考文章)




![](https://pic.rmb.bdstatic.com/bjh/down/97522789c423e19931adafa65bd5424d.png)

## JVM优化的必要性

**1.1： 项目上线后，什么原因使得我们需要进行jvm调优**

1)、垃圾太多（java线程，对象占满内存），内存占满了，程序跑不动了！！
2)、垃圾回收线程太多，频繁地回收垃圾（垃圾回收线程本身也会占用资源： 占用内存，cpu资源），导致程序性能下降
3)、回收垃圾频繁导致STW

因此基于以上的原因，程序上线后，必须进行调优，否则程序性能就无法提升；也就是程序上线后，必须设置合理的垃圾回收策略；

**1.2： jvm调优的本质是什么？？**

答案： 回收垃圾，及时回收没有用垃圾对象，及时释放掉内存空间

**1.3： 基于服务器环境，jvm堆内存到底应用设置多少内存？**

1、32位的操作系统 --- 寻址能力 2^32 = 4GB ,最大的能支持4gb; jvm可以分配 2g+

2、64位的操作系统 --- 寻址能力 2^64 = 16384PB , 高性能计算机（IBM Z unix 128G 200+）

![](https://pic.rmb.bdstatic.com/bjh/down/aae367a929e2d1361975942091ffadfe.png)

Jvm堆内存不能设置太大，否则会导致寻址垃圾的时间过长，也就是导致整个程序STW, 也不能设置太小，否则会导致回收垃圾过于频繁；

**1.4：总结**

如果你遇到以下情况，就需要考虑进行JVM调优了：

*   Heap内存（老年代）持续上涨达到设置的最大内存值；

*   Full GC 次数频繁；

*   GC 停顿时间过长（超过1秒）；

*   应用出现OutOfMemory 等内存异常；

*   应用中有使用本地缓存且占用大量内存空间；

*   系统吞吐量与响应性能不高或下降。

## JVM调优原则

![](https://pic.rmb.bdstatic.com/bjh/down/7cd4f5c19ab4f5f5d3087422097ee931.png)

JVM调优是一个手段，但并不一定所有问题都可以通过JVM进行调优解决，因此，在进行JVM调优时，我们要遵循一些原则：

*   大多数的Java应用不需要进行JVM优化；

*   大多数导致GC问题的原因是代码层面的问题导致的（代码层面）；

*   上线之前，应先考虑将机器的JVM参数设置到最优；

*   减少创建对象的数量（代码层面）；

*   减少使用全局变量和大对象（代码层面）；

*   优先架构调优和代码调优，JVM优化是不得已的手段（代码、架构层面）；

*   分析GC情况优化代码比优化JVM参数更好（代码层面）；

通过以上原则，我们发现，其实最有效的优化手段是架构和代码层面的优化，而JVM优化则是最后不得已的手段，也可以说是对服务器配置的最后一次“压榨”。

**2.1 JVM调优目标**

调优的最终目的都是为了令应用程序使用最小的硬件消耗来承载更大的吞吐。jvm调优主要是针对垃圾收集器的收集性能优化，令运行在虚拟机上的应用能够使用更少的内存以及延迟获取更大的吞吐量。

*   延迟：GC低停顿和GC低频率；

*   低内存占用；

*   高吞吐量;

其中，任何一个属性性能的提高，几乎都是以牺牲其他属性性能的损为代价的，不可兼得。具体根据在业务中的重要性确定。

**2.2 JVM调优量化目标**

下面展示了一些JVM调优的量化目标参考实例：

*   Heap 内存使用率 <= 70%;

*   Old generation内存使用率<= 70%;

*   avgpause <= 1秒;

*   Full gc 次数0 或 avg pause interval >= 24小时 ;

注意：不同应用的JVM调优量化目标是不一样的。

**2.3 JVM调优的步骤**

一般情况下，JVM调优可通过以下步骤进行：

*   分析GC日志及dump文件，判断是否需要优化，确定瓶颈问题点；

*   确定JVM调优量化目标；

*   确定JVM调优参数（根据历史JVM参数来调整）；

*   依次调优内存、延迟、吞吐量等指标；

*   对比观察调优前后的差异；

*   不断的分析和调整，直到找到合适的JVM参数配置；

*   找到最合适的参数，将这些参数应用到所有服务器，并进行后续跟踪。

以上操作步骤中，某些步骤是需要多次不断迭代完成的。一般是从满足程序的内存使用需求开始的，之后是时间延迟的要求，最后才是吞吐量的要求，要基于这个步骤来不断优化，每一个步骤都是进行下一步的基础，不可逆行之。

![](https://pic.rmb.bdstatic.com/bjh/down/c271890a3714d538ed3362073c66893d.png)

**调优原则总结**

JVM的自动内存管理本来就是为了将开发人员从内存管理的泥潭里拉出来。JVM调优不是常规手段，性能问题一般第一选择是优化程序，最后的选择才是进行JVM调优。

即使不得不进行JVM调优，也绝对不能拍脑门就去调整参数，一定要全面监控，详细分析性能数据。

**附录：系统性能优化指导**

![](https://pic.rmb.bdstatic.com/bjh/down/00d90cef8369568f8581a9850dcd42e6.png)

### JVM运行参数设置

**3.1、堆参数设置**

**-XX:+PrintGC**使用这个参数，虚拟机启动后，只要遇到GC就会打印日志

**-XX:+UseSerialGC**配置串行回收器

**-XX:+PrintGCDetails**可以查看详细信息，包括各个区的情况

**-Xms**设置Java程序启动时初始化堆大小

**-Xmx**设置Java程序能获得最大的堆大小

**-Xmx20m -Xms5m -XX:+PrintCommandLineFlags**可以将隐式或者显示传给虚拟机的参数输出

**3.2、新生代参数配置**

**-Xmn**可以设置新生代的大小，设置一个比较大的新生代会减少老年代的大小，这个参数对系统性能以及GC行为有很大的影响，新生代大小一般会设置整个堆空间的1/3到1/4左右

**-XX:SurvivorRatio**用来设置新生代中eden空间和from/to空间的比例。含义：-XX:SurvivorRatio=eden/from**/**eden/to

不同的堆分布情况，对系统执行会产生一定的影响，在实际工作中，应该根据系统的特点做出合理的配置，基本策略：尽可能将对象预留在新生代，减少老年代的GC次数

除了可以设置新生代的绝对大小（-Xmn），还可以使用（-XX:NewRatio）设置新生代和老年代的比例：-XX：NewRatio=老年代/新生代

**配置运行时参数：**

-Xms20m -Xmx20m -Xmn1m -XX:SurvivorRatio=2 -XX:+PrintGCDetails -XX:+UseSerialGC

**3.3、堆溢出参数配置**

在Java程序在运行过程中，如果对空间不足，则会抛出内存溢出的错误（Out Of Memory）OOM，一旦这类问题发生在生产环境，则可能引起严重的业务中断，Java虚拟机提供了**-XX:+
HeapDumpOnOutOfMemoryError**，使用该参数可以在内存溢出时导出整个堆信息，与之配合使用的还有参数**-XX:HeapDumpPath**，可以设置导出堆的存放路径

内存分析工具：Memory Analyzer

**配置运行时参数**-Xms1m -Xmx1m -XX:+
HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=d:/Demo3.dump

**3.4、栈参数配置**

Java虚拟机提供了参数**-Xss**来指定线程的最大栈空间，整个参数也直接决定了函数可调用的最大深度。

**配置运行时参数：**-Xss1m

3.5、方法区参数配置

和Java堆一样，方法区是一块所有线程共享的内存区域，它用于保存系统的类信息，方法区（永久区）可以保存多少信息可以对其进行配置，在默认情况下，**-XX:MaxPermSize**为64M，如果系统运行时生产大量的类，就需要设置一个相对合适的方法区，以免出现永久区内存溢出的问题

-XX:PermSize=64M -XX:MaxPermSize=64M

**3.6、直接内存参数配置**

直接内存也是Java程序中非常重要的组成部分，特别是广泛用在NIO中，直接内存跳过了Java堆，使用Java程序可以直接访问原生堆空间，因此在一定程度上加快了内存空间的访问速度

但是说直接内存一定就可以提高内存访问速度也不见得，具体情况具体分析

**相关配置参数：-XX:MaxDirectMemorySize**，如果不设置，默认值为最大堆空间，即-Xmx。直接内存使用达到上限时，就会触发垃圾回收，如果不能有效的释放空间，就会引起系统的OOM

**3.7、对象进入老年代的参数配置**

一般而言，对象首次创建会被放置在新生代的eden区，如果没有GC介入，则对象不会离开eden区，那么eden区的对象如何进入老年代呢？

通常情况下，只要对象的年龄达到一定的大小，就会自动离开年轻代进入老年代，对象年龄是由对象经历数次GC决定的，在新生代每次GC之后如果对象没有被回收，则年龄加1

虚拟机提供了一个参数来控制新生代对象的最大年龄，当超过这个年龄范围就会晋升老年代

**-XX:MaxTenuringThreshold**，默认情况下为15

**配置运行时参数：**-Xmx64M -Xms64M -XX:+PrintGCDetails

**结论**：对象首次创建会被放置在新生代的eden区，因此输出结果中from和to区都为0%

根据设置MaxTenuringThreshold参数，可以指定新生代对象经过多少次回收后进入老年代。另外，大对象新生代eden区无法装入时，也会直接进入老年代。

JVM里有个参数可以设置对象的大小超过在指定的大小之后，直接晋升老年代 **-XX:PretenureSizeThreshold=15**

参数：-Xmx1024M -Xms1024M -XX:+UseSerialGC -XX:MaxTenuringThreshold=15 -XX:+PrintGCDetails

使用PretenureSizeThreshold可以进行指定进入老年代的对象大小，但是要注意TLAB区域优先分配空间。虚拟机对于体积不大的对象 会优先把数据分配到TLAB区域中，因此就失去了在老年代分配的机会

参数：-Xmx30M -Xms30M -XX:+UseSerialGC -XX:+PrintGCDetails
-XX:PretenureSizeThreshold=1000 -XX:-UseTLAB

**3.8、TLAB参数配置**

TLAB全称是Thread Local Allocation Buffer，即线程本地分配缓存，从名字上看是一个线程专用的内存分配区域，是为了加速对象分配对象而生的。每一个线程都会产生一个TLAB，该线程独享的工作区域，Java虚拟机使用这种TLAB区来避免多线程冲突问题，提高了对象分配的效率

TLAB空间一般不会太大，当大对象无法在TLAB分配时，则会直接分配到堆上

**-XX:+UseTLAB**使用TLAB

**-XX:+TLABSize**设置TLAB大小

**-XX:TLABRefillWasteFraction**设置维护进入TLAB空间的单个对象大小，它是一个比例值，默认为64，即如果对象大于整个空间的1/64，则在堆创建对象

**-XX:+PrintTLAB**查看TLAB信息

**-XX:ResizeTLAB**自调整TLABRefillWasteFraction阈值

参数：-XX:+UseTLAB -XX:+PrintTLAB -XX:+PrintGC -XX:TLABSize=102400 -XX:-ResizeTLAB
-XX:TLABRefillWasteFraction=100 -XX:-DoEscapeAnalysis -server

内存参数





![](https://pics7.baidu.com/feed/bba1cd11728b47102f9e1e7af46fe7f6fd03237b.png@f_auto?token=ca9d5541411861cfb09e792849a82a06)





## JVM性能调优工具

这个篇幅在这里就不过多介绍了，可以参照：

深入理解JVM虚拟机——Java虚拟机的监控及诊断工具大全

## 常用调优策略

这里还是要提一下，及时确定要进行JVM调优，也不要陷入“知见障”，进行分析之后，发现可以通过优化程序提升性能，仍然首选优化程序。

**5.1、选择合适的垃圾回收器**

CPU单核，那么毫无疑问Serial 垃圾收集器是你唯一的选择。

CPU多核，关注吞吐量 ，那么选择PS+PO组合。

CPU多核，关注用户停顿时间，JDK版本1.6或者1.7，那么选择CMS。

CPU多核，关注用户停顿时间，JDK1.8及以上，JVM可用内存6G以上，那么选择G1。

参数配置：

> //设置Serial垃圾收集器（新生代）
>
> 开启：-XX:+UseSerialGC
>
> //设置PS+PO,新生代使用功能Parallel Scavenge 老年代将会使用Parallel Old收集器
>
> 开启 -XX:+UseParallelOldGC
>
> //CMS垃圾收集器（老年代）
>
> 开启 -XX:+UseConcMarkSweepGC
>
> //设置G1垃圾收集器
>
> 开启 -XX:+UseG1GC

**5.2、调整内存大小**

现象：垃圾收集频率非常频繁。

原因：如果内存太小，就会导致频繁的需要进行垃圾收集才能释放出足够的空间来创建新的对象，所以增加堆内存大小的效果是非常显而易见的。

注意：如果垃圾收集次数非常频繁，但是每次能回收的对象非常少，那么这个时候并非内存太小，而可能是内存泄露导致对象无法回收，从而造成频繁GC。

参数配置：

> //设置堆初始值
>
> 指令1：-Xms2g
>
> 指令2：-XX:InitialHeapSize=2048m
>
> //设置堆区最大值
>
> 指令1：`-Xmx2g`
>
> 指令2： -XX:MaxHeapSize=2048m
>
> //新生代内存配置
>
> 指令1：-Xmn512m
>
> 指令2：-XX:MaxNewSize=512m

**5.3、设置符合预期的停顿时间**

**现象**：程序间接性的卡顿

**原因**：如果没有确切的停顿时间设定，垃圾收集器以吞吐量为主，那么垃圾收集时间就会不稳定。

**注意**：不要设置不切实际的停顿时间，单次时间越短也意味着需要更多的GC次数才能回收完原有数量的垃圾.

**参数配置**：

> //GC停顿时间，垃圾收集器会尝试用各种手段达到这个时间
>
> -XX:MaxGCPauseMillis

**5.4、调整内存区域大小比率**

现象：某一个区域的GC频繁，其他都正常。

原因：如果对应区域空间不足，导致需要频繁GC来释放空间，在JVM堆内存无法增加的情况下，可以调整对应区域的大小比率。

注意：也许并非空间不足，而是因为内存泄造成内存无法回收。从而导致GC频繁。

参数配置：

> //survivor区和Eden区大小比率
>
> 指令：-XX:SurvivorRatio=6 //S区和Eden区占新生代比率为1:6,两个S区2:6
>
> //新生代和老年代的占比
>
> -XX:NewRatio=4 //表示新生代:老年代 = 1:4 即老年代占整个堆的4/5；默认值=2

**5.5、调整对象升老年代的年龄**

**现象**：老年代频繁GC，每次回收的对象很多。

**原因**：如果升代年龄小，新生代的对象很快就进入老年代了，导致老年代对象变多，而这些对象其实在随后的很短时间内就可以回收，这时候可以调整对象的升级代年龄，让对象不那么容易进入老年代解决老年代空间不足频繁GC问题。

**注意**：增加了年龄之后，这些对象在新生代的时间会变长可能导致新生代的GC频率增加，并且频繁复制这些对象新生的GC时间也可能变长。

配置参数：

> //进入老年代最小的GC年龄,年轻代对象转换为老年代对象最小年龄值，默认值7
>
> -XX:InitialTenuringThreshol=7

**5.6、调整大对象的标准**

**现象**：老年代频繁GC，每次回收的对象很多,而且单个对象的体积都比较大。

**原因**：如果大量的大对象直接分配到老年代，导致老年代容易被填满而造成频繁GC，可设置对象直接进入老年代的标准。

**注意**：这些大对象进入新生代后可能会使新生代的GC频率和时间增加。

配置参数：

> //新生代可容纳的最大对象,大于则直接会分配到老年代，0代表没有限制。
>
> -XX:PretenureSizeThreshold=1000000

**5.7、调整GC的触发时机**

**现象**：CMS，G1 经常 Full GC，程序卡顿严重。

**原因**：G1和CMS 部分GC阶段是并发进行的，业务线程和垃圾收集线程一起工作，也就说明垃圾收集的过程中业务线程会生成新的对象，所以在GC的时候需要预留一部分内存空间来容纳新产生的对象，如果这个时候内存空间不足以容纳新产生的对象，那么JVM就会停止并发收集暂停所有业务线程（STW）来保证垃圾收集的正常运行。这个时候可以调整GC触发的时机（比如在老年代占用60%就触发GC），这样就可以预留足够的空间来让业务线程创建的对象有足够的空间分配。

**注意**：提早触发GC会增加老年代GC的频率。

配置参数：

> //使用多少比例的老年代后开始CMS收集，默认是68%，如果频繁发生SerialOld卡顿，应该调小
>
> -XX:CMSInitiatingOccupancyFraction
>
> //G1混合垃圾回收周期中要包括的旧区域设置占用率阈值。默认占用率为 65%
>
> -XX:G1MixedGCLiveThresholdPercent=65

5.8、调整 JVM本地内存大小

**现象**：GC的次数、时间和回收的对象都正常，堆内存空间充足，但是报OOM

**原因**： JVM除了堆内存之外还有一块堆外内存，这片内存也叫本地内存，可是这块内存区域不足了并不会主动触发GC，只有在堆内存区域触发的时候顺带会把本地内存回收了，而一旦本地内存分配不足就会直接报OOM异常。

**注意**： 本地内存异常的时候除了上面的现象之外，异常信息可能是OutOfMemoryError：Direct buffer memory。 解决方式除了调整本地内存大小之外，也可以在出现此异常时进行捕获，手动触发GC（System.gc()）。

配置参数：

> XX:MaxDirectMemorySize

## 六、JVM调优实例

整理的一些JVM调优实例：

**6.1、网站流量浏览量暴增后，网站反应页面响很慢**

> 1、问题推测：在测试环境测速度比较快，但是一到生产就变慢，所以推测可能是因为垃圾收集导致的业务线程停顿。
>
> 2、定位：为了确认推测的正确性，在线上通过jstat -gc 指令 看到JVM进行GC 次数频率非常高，GC所占用的时间非常长，所以基本推断就是因为GC频率非常高，所以导致业务线程经常停顿，从而造成网页反应很慢。
>
> 3、解决方案：因为网页访问量很高，所以对象创建速度非常快，导致堆内存容易填满从而频繁GC，所以这里问题在于新生代内存太小，所以这里可以增加JVM内存就行了，所以初步从原来的2G内存增加到16G内存。
>
> 4、第二个问题：增加内存后的确平常的请求比较快了，但是又出现了另外一个问题，就是不定期的会间断性的卡顿，而且单次卡顿的时间要比之前要长很多。
>
> 5、问题推测：练习到是之前的优化加大了内存，所以推测可能是因为内存加大了，从而导致单次GC的时间变长从而导致间接性的卡顿。
>
> 6、定位：还是通过jstat -gc 指令 查看到 的确FGC次数并不是很高，但是花费在FGC上的时间是非常高的,根据GC日志 查看到单次FGC的时间有达到几十秒的。
>
> 7、解决方案： 因为JVM默认使用的是PS+PO的组合，PS+PO垃圾标记和收集阶段都是STW，所以内存加大了之后，需要进行垃圾回收的时间就变长了，所以这里要想避免单次GC时间过长，所以需要更换并发类的收集器，因为当前的JDK版本为1.7，所以最后选择CMS垃圾收集器，根据之前垃圾收集情况设置了一个预期的停顿的时间，上线后网站再也没有了卡顿问题。

**6.2、后台导出数据引发的OOM**

**问题描述：**公司的后台系统，偶发性的引发OOM异常，堆内存溢出。

> 1、因为是偶发性的，所以第一次简单的认为就是堆内存不足导致，所以单方面的加大了堆内存从4G调整到8G。
>
> 2、但是问题依然没有解决，只能从堆内存信息下手，通过开启了-XX:+
> HeapDumpOnOutOfMemoryError参数 获得堆内存的dump文件。
>
> 3、VisualVM 对 堆dump文件进行分析，通过VisualVM查看到占用内存最大的对象是String对象，本来想跟踪着String对象找到其引用的地方，但dump文件太大，跟踪进去的时候总是卡死，而String对象占用比较多也比较正常，最开始也没有认定就是这里的问题，于是就从线程信息里面找突破点。
>
> 4、通过线程进行分析，先找到了几个正在运行的业务线程，然后逐一跟进业务线程看了下代码，发现有个引起我注意的方法，导出订单信息。
>
> 5、因为订单信息导出这个方法可能会有几万的数据量，首先要从数据库里面查询出来订单信息，然后把订单信息生成excel，这个过程会产生大量的String对象。
>
> 6、为了验证自己的猜想，于是准备登录后台去测试下，结果在测试的过程中发现到处订单的按钮前端居然没有做点击后按钮置灰交互事件，结果按钮可以一直点，因为导出订单数据本来就非常慢，使用的人员可能发现点击后很久后页面都没反应，结果就一直点，结果就大量的请求进入到后台，堆内存产生了大量的订单对象和EXCEL对象，而且方法执行非常慢，导致这一段时间内这些对象都无法被回收，所以最终导致内存溢出。
>
> 7、知道了问题就容易解决了，最终没有调整任何JVM参数，只是在前端的导出订单按钮上加上了置灰状态，等后端响应之后按钮才可以进行点击，然后减少了查询订单信息的非必要字段来减少生成对象的体积，然后问题就解决了。

**6.3、单个缓存数据过大导致的系统CPU飚高**

> 1、系统发布后发现CPU一直飚高到600%，发现这个问题后首先要做的是定位到是哪个应用占用CPU高，通过top 找到了对应的一个java应用占用CPU资源600%。
>
> 2、如果是应用的CPU飚高，那么基本上可以定位可能是锁资源竞争，或者是频繁GC造成的。
>
> 3、所以准备首先从GC的情况排查，如果GC正常的话再从线程的角度排查，首先使用jstat -gc PID 指令打印出GC的信息，结果得到得到的GC 统计信息有明显的异常，应用在运行了才几分钟的情况下GC的时间就占用了482秒，那么问这很明显就是频繁GC导致的CPU飚高。
>
> 4、定位到了是GC的问题，那么下一步就是找到频繁GC的原因了，所以可以从两方面定位了，可能是哪个地方频繁创建对象，或者就是有内存泄露导致内存回收不掉。
>
> 5、根据这个思路决定把堆内存信息dump下来看一下，使用jmap -dump 指令把堆内存信息dump下来（堆内存空间大的慎用这个指令否则容易导致会影响应用，因为我们的堆内存空间才2G所以也就没考虑这个问题了）。
>
> 6、把堆内存信息dump下来后，就使用visualVM进行离线分析了，首先从占用内存最多的对象中查找，结果排名第三看到一个业务VO占用堆内存约10%的空间，很明显这个对象是有问题的。
>
> 7、通过业务对象找到了对应的业务代码，通过代码的分析找到了一个可疑之处，这个业务对象是查看新闻资讯信息生成的对象，由于想提升查询的效率，所以把新闻资讯保存到了redis缓存里面，每次调用资讯接口都是从缓存里面获取。
>
> 8、把新闻保存到redis缓存里面这个方式是没有问题的，有问题的是新闻的50000多条数据都是保存在一个key里面，这样就导致每次调用查询新闻接口都会从redis里面把50000多条数据都拿出来，再做筛选分页拿出10条返回给前端。50000多条数据也就意味着会产生50000多个对象，每个对象280个字节左右，50000个对象就有13.3M，这就意味着只要查看一次新闻信息就会产生至少13.3M的对象，那么并发请求量只要到10，那么每秒钟都会产生133M的对象，而这种大对象会被直接分配到老年代，这样的话一个2G大小的老年代内存，只需要几秒就会塞满，从而触发GC。
>
> 9、知道了问题所在后那么就容易解决了，问题是因为单个缓存过大造成的，那么只需要把缓存减小就行了，这里只需要把缓存以页的粒度进行缓存就行了，每个key缓存10条作为返回给前端1页的数据，这样的话每次查询新闻信息只会从缓存拿出10条数据，就避免了此问题的 产生。

**6.4、CPU经常100% 问题定位**

问题分析：CPU高一定是某个程序长期占用了CPU资源。

1、所以先需要找出那个进行占用CPU高。

> top 列出系统各个进程的资源占用情况。

2、然后根据找到对应进行里哪个线程占用CPU高。

> top -Hp 进程ID 列出对应进程里面的线程占用资源情况

3、找到对应线程ID后，再打印出对应线程的堆栈信息

> printf "%x\n" PID 把线程ID转换为16进制。
>
> jstack PID 打印出进程的所有线程信息，从打印出来的线程信息中找到上一步转换为16进制的线程ID对应的线程信息。

4、最后根据线程的堆栈信息定位到具体业务方法,从代码逻辑中找到问题所在。

> 查看是否有线程长时间的watting 或blocked
>
> 如果线程长期处于watting状态下， 关注watting on xxxxxx，说明线程在等待这把锁，然后根据锁的地址找到持有锁的线程。

**6.5、内存飚高问题定位**

分析： 内存飚高如果是发生在java进程上，一般是因为创建了大量对象所导致，持续飚高说明垃圾回收跟不上对象创建的速度，或者内存泄露导致对象无法回收。

1、先观察垃圾回收的情况

> jstat -gcPID 1000查看GC次数，时间等信息，每隔一秒打印一次。
>
> jmap -histo PID | head -20 查看堆内存占用空间最大的前20个对象类型,可初步查看是哪个对象占用了内存。

如果每次GC次数频繁，而且每次回收的内存空间也正常，那说明是因为对象创建速度快导致内存一直占用很高；如果每次回收的内存非常少，那么很可能是因为内存泄露导致内存一直无法被回收。

2、导出堆内存文件快照

> jmap -dump:live,format=b,file=/home/myheapdump.hprof PID dump堆内存信息到文件。

3、使用visualVM对dump文件进行离线分析,找到占用内存高的对象，再找到创建该对象的业务代码位置，从代码和业务场景中定位具体问题。

**6.6、数据分析平台系统频繁 Full GC**

平台主要对用户在 App 中行为进行定时分析统计，并支持报表导出，使用 CMS GC 算法。

数据分析师在使用中发现系统页面打开经常卡顿，通过 jstat 命令发现系统每次 Young GC 后大约有 10% 的存活对象进入老年代。

原来是因为 Survivor 区空间设置过小，每次 Young GC 后存活对象在 Survivor 区域放不下，提前进入老年代。

通过调大 Survivor 区，使得 Survivor 区可以容纳 Young GC 后存活对象，对象在 Survivor 区经历多次 Young GC 达到年龄阈值才进入老年代。

调整之后每次 Young GC 后进入老年代的存活对象稳定运行时仅几百 Kb，Full GC 频率大大降低。

**6.7、业务对接网关 OOM**

网关主要消费 Kafka 数据，进行数据处理计算然后转发到另外的 Kafka 队列，系统运行几个小时候出现 OOM，重启系统几个小时之后又 OOM。

通过 jmap 导出堆内存，在 eclipse MAT 工具分析才找出原因：代码中将某个业务 Kafka 的 topic 数据进行日志异步打印，该业务数据量较大，大量对象堆积在内存中等待被打印，导致 OOM。

**6.8、鉴权系统频繁长时间 Full GC**

系统对外提供各种账号鉴权服务，使用时发现系统经常服务不可用，通过 Zabbix 的监控平台监控发现系统频繁发生长时间 Full GC，且触发时老年代的堆内存通常并没有占满，发现原来是业务代码中调用了

## 七、一个具体的实战案例分析

**7.1 典型调优参数设置**

> 服务器配置： 4cpu,8GB内存 ---- jvm调优实际上是设置一个合理大小的jvm堆内存（既不能太大，也不能太小）

> -Xmx3550m 设置jvm堆内存最大值 (经验值设置： 根据压力测试，根据线上程序运行效果情况)
>
> -Xms3550m 设置jvm堆内存初始化大小，一般情况下必须设置此值和最大的最大的堆内存空间保持一致，防止内存抖动，消耗性能
>
> -Xmn2g 设置年轻代占用的空间大小
>
> -Xss256k 设置线程堆栈的大小；jdk5.0以后默认线程堆栈大小为1MB; 在相同的内存情况下，减小堆栈大小，可以使得操作系统创建更多的业务线程；

jvm堆内存设置:

> nohup java -Xmx3550m -Xms3550m -Xmn2g -Xss256k -jar jshop-web-1.0-SNAPSHOT.jar --spring.addition-location=application.yaml > jshop.log 2>&1 ><

TPS性能曲线：

![](https://pic.rmb.bdstatic.com/bjh/down/ee3949fc51c360330ece8f6729cd0560.png)

**7.2 分析gc日志**

如果需要分析gc日志，就必须使得服务gc输入gc详情到log日志文件中，然后使用相应gc日志分析工具来对日志进行分析即可；

把gc详情输出到一个gc.log日志文件中，便于gc分析

> -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log

Throughput: 业务线程执行时间 / (gc时间+业务线程时间)

![](https://pic.rmb.bdstatic.com/bjh/down/25529b786b50b8b30067e6f721101369.png)

分析gc日志，发现，一开始就发生了3次fullgc，很明显jvm优化参数的设置是有问题的；

![](https://pic.rmb.bdstatic.com/bjh/down/e70f3ffedc03bc0b387ff860cb8c948b.png)

查看fullgc发生问题原因： jstat -gcutil pid

![](https://pic.rmb.bdstatic.com/bjh/down/0e135e59c1e140aaded9223edd2f0177.png)

Metaspace持久代： 初始化分配大小20m , 当metaspace被占满后，必须对持久代进行扩容，如果metaspace每进行一次扩容，fullgc就需要执行一次；（fullgc回收整个堆空间，非常占用时间）

调整gc配置： 修改永久代空间初始化大小:

> nohup java -Xmx3550m -Xms3550m -Xmn2g -Xss256k -XX:MetaspaceSize=256m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log -jar jshop-web-1.0-SNAPSHOT.jar --spring.addition-location=application.yaml > jshop.log 2>&1 ><

经过调优后，fullgc现象已经消失了：

![](https://pic.rmb.bdstatic.com/bjh/down/0f8e8b90a8f9d810759ae43b3b0ed1b9.png)

**7.3 Young&Old比例**

年轻代和老年代比例：1:2 参数：-XX:NewRetio = 4 , 表示年轻代（eden,s0,s1）和老年代所占比值为1:4

1） -XX:NewRetio = 4

![](https://pic.rmb.bdstatic.com/bjh/down/f8353c852a2f65626b5b3a23f9616208.png)

年轻代分配的内存大小变小了，这样YGC次数变多了，虽然fullgc不发生了，但是YGC花费的时间更多了！

2） -XX:NewRetio = 2 YGC发生的次数必然会减少；因为eden区域的大小变大了，因此YGC就会变少；

![](https://pic.rmb.bdstatic.com/bjh/down/11ca145ee5a93f511feda69e749582cb.png)

7.4 Eden&S0S1

为了进一步减少YGC, 可以设置 enden ,s 区域的比值大小； 设置方式： -XX:SurvivorRatio=8

1） 设置比值：8:1:1

![](https://pic.rmb.bdstatic.com/bjh/down/6330cbdd9acd6a5c18e91c5bea13cca2.png)

2） Xmn2g 8:1:1

nohup java -Xmx3550m -Xms3550m -Xmn2g -XX:SurvivorRatio=8 -Xss256k -XX:MetaspaceSize=256m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log -jar jshop-web-1.0-SNAPSHOT.jar --spring.addition-location=application.yaml > jshop.log 2>&1 ><

根据gc调优，垃圾回收次数，时间，吞吐量都是一个比较优的一个配置；

![](https://pic.rmb.bdstatic.com/bjh/down/68edd8bfceb7d5e85d18a53e150cad2a.png)

**7.5 吞吐量优先**

使用并行的垃圾回收器，可以充分利用多核心cpu来帮助进行垃圾回收；这样的gc方式，就叫做吞吐量优先的调优方式

垃圾回收器组合： ps(parallel scavenge) + po (parallel old) 此垃圾回收器是Jdk1.8 默认的垃圾回收器组合；

> nohup java -Xmx3550m -Xms3550m -Xmn2g -XX:SurvivorRatio=8 -Xss256k -XX:+UseParallelGC -XX:UseParallelOldGC -XX:MetaspaceSize=256m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log -jar jshop-web-1.0-SNAPSHOT.jar --spring.addition-location=application.yaml > jshop.log 2>&1 ><

**7.6 响应时间优先**

使用cms垃圾回收器，就是一个响应时间优先的组合； cms垃圾回收器（垃圾回收和业务线程交叉执行，不会让业务线程进行停顿stw）尽可能的减少stw的时间，因此使用cms垃圾回收器组合，是响应时间优先组合

> nohup java -Xmx3550m -Xms3550m -Xmn2g -XX:SurvivorRatio=8 -Xss256k -XX:+UseParNewGC -XX:UseConcMarkSweepGC -XX:MetaspaceSize=256m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log -jar jshop-web-1.0-SNAPSHOT.jar --spring.addition-location=application.yaml > jshop.log 2>&1 ><

可以发现，cms垃圾回收器时间变长；

**7.7 g1**

配置方式如下所示：

> nohup java -Xmx3550m -Xms3550m -Xmn2g -XX:SurvivorRatio=8 -Xss256k -XX:+UseG1GC -XX:MetaspaceSize=256m -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintHeapAtGC -Xloggc:gc.log -jar jshop-web-1.0-SNAPSHOT.jar --spring.addition-location=application.yaml > jshop.log 2>&1 ><

![](https://pic.rmb.bdstatic.com/bjh/down/4146281952d45f65334e7ba97c6ba873.png)


## 参考资料

*   [Java HotSpot™ Virtual Machine Performance Enhancements](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/performance-enhancements-7.html)
*   [Java HotSpot Virtual Machine Garbage Collection Tuning Guide](http://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/index.html)
*   [[HotSpot VM] JVM调优的”标准参数”的各种陷阱](http://hllvm.group.iteye.com/group/topic/27945)


## 参考文章

<https://segmentfault.com/a/1190000009707894>

<https://www.cnblogs.com/hysum/p/7100874.html>

<http://c.biancheng.net/view/939.html>

<https://www.runoob.com/>

https://blog.csdn.net/android_hl/article/details/53228348


