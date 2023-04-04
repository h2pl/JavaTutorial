# 目录
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《深入理解JVM虚拟机》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。
该系列博文会告诉你如何从入门到进阶，一步步地学习JVM基础知识，并上手进行JVM调优实战，JVM是每一个Java工程师必须要学习和理解的知识点，你必须要掌握其实现原理，才能更完整地了解整个Java技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## 5\. GC 调优(基础篇) - GC参考手册

> **说明**:
>
> **Capacity**: 性能,能力,系统容量; 文中翻译为”**系统容量**“; 意为硬件配置。

您应该已经阅读了前面的章节:

1.  垃圾收集简介 - GC参考手册
2.  Java中的垃圾收集 - GC参考手册
3.  GC 算法(基础篇) - GC参考手册
4.  GC 算法(实现篇) - GC参考手册

GC调优(Tuning Garbage Collection)和其他性能调优是同样的原理。初学者可能会被 200 多个 GC参数弄得一头雾水, 然后随便调整几个来试试结果,又或者修改几行代码来测试。其实只要参照下面的步骤，就能保证你的调优方向正确:

1.  列出性能调优指标(State your performance goals)
2.  执行测试(Run tests)
3.  检查结果(Measure the results)
4.  与目标进行对比(Compare the results with the goals)
5.  如果达不到指标, 修改配置参数, 然后继续测试(go back to running tests)

第一步, 我们需要做的事情就是: 制定明确的GC性能指标。对所有性能监控和管理来说, 有三个维度是通用的:

*   Latency(延迟)
*   Throughput(吞吐量)
*   Capacity(系统容量)

我们先讲解基本概念,然后再演示如何使用这些指标。如果您对 延迟、吞吐量和系统容量等概念很熟悉, 可以跳过这一小节。

### 核心概念(Core Concepts)

我们先来看一家工厂的装配流水线。工人在流水线将现成的组件按顺序拼接,组装成自行车。通过实地观测, 我们发现从组件进入生产线，到另一端组装成自行车需要4小时。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224347.png)
继续观察,我们还发现,此后每分钟就有1辆自行车完成组装, 每天24小时,一直如此。将这个模型简化, 并忽略维护窗口期后得出结论：**这条流水线每小时可以组装60辆自行车**。

> **说明**: 时间窗口/窗口期，请类比车站卖票的窗口，是一段规定/限定做某件事的时间段。

通过这两种测量方法, 就知道了生产线的相关性能信息：**延迟**与**吞吐量**:

*   生产线的延迟:**4小时**
*   生产线的吞吐量:**60辆/小时**

请注意, 衡量延迟的时间单位根据具体需要而确定 —— 从纳秒(nanosecond)到几千年(millennia)都有可能。系统的吞吐量是每个单位时间内完成的操作。操作(Operations)一般是特定系统相关的东西。在本例中,选择的时间单位是小时, 操作就是对自行车的组装。

掌握了延迟和吞吐量两个概念之后, 让我们对这个工厂来进行实际的调优。自行车的需求在一段时间内都很稳定, 生产线组装自行车有四个小时延迟, 而吞吐量在几个月以来都很稳定: 60辆/小时。假设某个销售团队突然业绩暴涨, 对自行车的需求增加了1倍。客户每天需要的自行车不再是 60 * 24 = 1440辆, 而是 2*1440 = 2880辆/天。老板对工厂的产能不满意，想要做些调整以提升产能。

看起来总经理很容易得出正确的判断, 系统的延迟没法子进行处理 —— 他关注的是每天的自行车生产总量。得出这个结论以后, 假若工厂资金充足, 那么应该立即采取措施, 改善吞吐量以增加产能。

我们很快会看到, 这家工厂有两条相同的生产线。每条生产线一分钟可以组装一辆成品自行车。 可以想象，每天生产的自行车数量会增加一倍。达到 2880辆/天。要注意的是, 不需要减少自行车的装配时间 —— 从开始到结束依然需要 4 小时。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224359.png)

巧合的是，这样进行的性能优化,同时增加了吞吐量和产能。一般来说，我们会先测量当前的系统性能, 再设定新目标, 只优化系统的某个方面来满足性能指标。

在这里做了一个很重要的决定 —— 要增加吞吐量,而不是减小延迟。在增加吞吐量的同时, 也需要增加系统容量。比起原来的情况, 现在需要两条流水线来生产出所需的自行车。在这种情况下, 增加系统的吞吐量并不是免费的, 需要水平扩展, 以满足增加的吞吐量需求。

在处理性能问题时, 应该考虑到还有另一种看似不相关的解决办法。假如生产线的延迟从1分钟降低为30秒,那么吞吐量同样可以增长 1 倍。

或者是降低延迟, 或者是客户非常有钱。软件工程里有一种相似的说法 —— 每个性能问题背后,总有两种不同的解决办法。 可以用更多的机器, 或者是花精力来改善性能低下的代码。

#### Latency(延迟)

GC的延迟指标由一般的延迟需求决定。延迟指标通常如下所述:

*   所有交易必须在10秒内得到响应
*   90%的订单付款操作必须在3秒以内处理完成
*   推荐商品必须在 100 ms 内展示到用户面前

面对这类性能指标时, 需要确保在交易过程中, GC暂停不能占用太多时间，否则就满足不了指标。“不能占用太多” 的意思需要视具体情况而定, 还要考虑到其他因素, 比如外部数据源的交互时间(round-trips), 锁竞争(lock contention), 以及其他的安全点等等。

假设性能需求为:`90%`的交易要在`1000ms`以内完成, 每次交易最长不能超过`10秒`。 根据经验, 假设GC暂停时间比例不能超过10%。 也就是说, 90%的GC暂停必须在`100ms`内结束, 也不能有超过`1000ms`的GC暂停。为简单起见, 我们忽略在同一次交易过程中发生多次GC停顿的可能性。

有了正式的需求,下一步就是检查暂停时间。有许多工具可以使用, 在接下来的6\. GC 调优(工具篇)



```
2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics)
        [PSYoungGen: 93677K->70109K(254976K)] 
        [ParOldGen: 499597K->511230K(761856K)] 
        593275K->581339K(1016832K),
        [Metaspace: 2936K->2936K(1056768K)]
    , 0.0713174 secs]
    [Times: user=0.21 sys=0.02, real=0.07 secs
```



这表示一次GC暂停, 在`2015-06-04T13:34:16`这个时刻触发. 对应于JVM启动之后的`2,578 ms`。

此事件将应用线程暂停了`0.0713174`秒。虽然花费的总时间为 210 ms, 但因为是多核CPU机器, 所以最重要的数字是应用线程被暂停的总时间, 这里使用的是并行GC, 所以暂停时间大约为`70ms`。 这次GC的暂停时间小于`100ms`的阈值，满足需求。

继续分析, 从所有GC日志中提取出暂停相关的数据, 汇总之后就可以得知是否满足需求。

#### Throughput(吞吐量)

吞吐量和延迟指标有很大区别。当然两者都是根据一般吞吐量需求而得出的。一般吞吐量需求(Generic requirements for throughput) 类似这样:

*   解决方案每天必须处理 100万个订单
*   解决方案必须支持1000个登录用户,同时在5-10秒内执行某个操作: A、B或C
*   每周对所有客户进行统计, 时间不能超过6小时，时间窗口为每周日晚12点到次日6点之间。

可以看出,吞吐量需求不是针对单个操作的, 而是在给定的时间内, 系统必须完成多少个操作。和延迟需求类似, GC调优也需要确定GC行为所消耗的总时间。每个系统能接受的时间不同, 一般来说, GC占用的总时间比不能超过`10%`。

现在假设需求为: 每分钟处理 1000 笔交易。同时, 每分钟GC暂停的总时间不能超过6秒(即10%)。

有了正式的需求, 下一步就是获取相关的信息。依然是从GC日志中提取数据, 可以看到类似这样的信息:



```
2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics)
        [PSYoungGen: 93677K->70109K(254976K)] 
        [ParOldGen: 499597K->511230K(761856K)] 
        593275K->581339K(1016832K), 
        [Metaspace: 2936K->2936K(1056768K)], 
     0.0713174 secs] 
     [Times: user=0.21 sys=0.02, real=0.07 secs
```



此时我们对 用户耗时(user)和系统耗时(sys)感兴趣, 而不关心实际耗时(real)。在这里, 我们关心的时间为`0.23s`(user + sys = 0.21 + 0.02 s), 这段时间内, GC暂停占用了 cpu 资源。 重要的是, 系统运行在多核机器上, 转换为实际的停顿时间(stop-the-world)为`0.0713174秒`, 下面的计算会用到这个数字。

提取出有用的信息后, 剩下要做的就是统计每分钟内GC暂停的总时间。看看是否满足需求: 每分钟内总的暂停时间不得超过6000毫秒(6秒)。

#### Capacity(系统容量)

系统容量(Capacity)需求,是在达成吞吐量和延迟指标的情况下,对硬件环境的额外约束。这类需求大多是来源于计算资源或者预算方面的原因。例如:

*   系统必须能部署到小于512 MB内存的Android设备上
*   系统必须部署在Amazon**EC2**实例上, 配置不得超过**c3.xlarge(4核8GB)**。
*   每月的 Amazon EC2 账单不得超过`$12,000`

因此, 在满足延迟和吞吐量需求的基础上必须考虑系统容量。可以说, 假若有无限的计算资源可供挥霍, 那么任何 延迟和吞吐量指标 都不成问题, 但现实情况是, 预算(budget)和其他约束限制了可用的资源。

### 相关示例

介绍完性能调优的三个维度后, 我们来进行实际的操作以达成GC性能指标。

请看下面的代码:



```
//imports skipped for brevity
public class Producer implements Runnable {

  private static ScheduledExecutorService executorService
         = Executors.newScheduledThreadPool(2);

  private Deque<byte[]> deque;
  private int objectSize;
  private int queueSize;

  public Producer(int objectSize, int ttl) {
    this.deque = new ArrayDeque<byte[]>();
    this.objectSize = objectSize;
    this.queueSize = ttl * 1000;
  }

  @Override
  public void run() {
    for (int i = 0; i < 100; i++) { 
        deque.add(new byte[objectSize]); 
        if (deque.size() > queueSize) {
            deque.poll();
        }
    }
  }

  public static void main(String[] args) 
        throws InterruptedException {
    executorService.scheduleAtFixedRate(
        new Producer(200 * 1024 * 1024 / 1000, 5), 
        0, 100, TimeUnit.MILLISECONDS
    );
    executorService.scheduleAtFixedRate(
        new Producer(50 * 1024 * 1024 / 1000, 120), 
        0, 100, TimeUnit.MILLISECONDS);
    TimeUnit.MINUTES.sleep(10);
    executorService.shutdownNow();
  }
}
```



这段程序代码, 每 100毫秒 提交两个作业(job)来。每个作业都模拟特定的生命周期: 创建对象, 然后在预定的时间释放, 接着就不管了, 由GC来自动回收占用的内存。

在运行这个示例程序时，通过以下JVM参数打开GC日志记录:



```
-XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps
```



还应该加上JVM参数`-Xloggc`以指定GC日志的存储位置,类似这样:



```
-Xloggc:C:\\Producer_gc.log
```



*   1
*   2

在日志文件中可以看到GC的行为, 类似下面这样:



```
2015-06-04T13:34:16.119-0200: 1.723: [GC (Allocation Failure) 
        [PSYoungGen: 114016K->73191K(234496K)] 
    421540K->421269K(745984K), 
    0.0858176 secs] 
    [Times: user=0.04 sys=0.06, real=0.09 secs] 

2015-06-04T13:34:16.738-0200: 2.342: [GC (Allocation Failure) 
        [PSYoungGen: 234462K->93677K(254976K)] 
    582540K->593275K(766464K), 
    0.2357086 secs] 
    [Times: user=0.11 sys=0.14, real=0.24 secs] 

2015-06-04T13:34:16.974-0200: 2.578: [Full GC (Ergonomics) 
        [PSYoungGen: 93677K->70109K(254976K)] 
        [ParOldGen: 499597K->511230K(761856K)] 
    593275K->581339K(1016832K), 
        [Metaspace: 2936K->2936K(1056768K)], 
    0.0713174 secs] 
    [Times: user=0.21 sys=0.02, real=0.07 secs]
```



基于日志中的信息, 可以通过三个优化目标来提升性能:

1.  确保最坏情况下,GC暂停时间不超过预定阀值
2.  确保线程暂停的总时间不超过预定阀值
3.  在确保达到延迟和吞吐量指标的情况下, 降低硬件配置以及成本。

为此, 用三种不同的配置, 将代码运行10分钟, 得到了三种不同的结果, 汇总如下:

<colgroup data-id="c7104f7d-gSmaHJLh"><col data-id="cdf32f61-8J2cfg0C" span="1" width="239"><col data-id="cdf32f61-AWERG8eA" span="1" width="239"><col data-id="cdf32f61-NaGTMlPG" span="1" width="239"><col data-id="cdf32f61-dRpWPkiD" span="1" width="239"></colgroup>
| **堆内存大小(Heap)** | **GC算法(GC Algorithm)** | **有效时间比(Useful work)** | **最长停顿时间(Longest pause)** |
| -Xmx12g | -XX:+UseConcMarkSweepGC | 89.8% | **560 ms** |
| -Xmx12g | -XX:+UseParallelGC | 91.5% | 1,104 ms |
| -Xmx8g | -XX:+UseConcMarkSweepGC | 66.3% | 1,610 ms |

使用不同的GC算法,和不同的内存配置,运行相同的代码, 以测量GC暂停时间与 延迟、吞吐量的关系。实验的细节和结果在后面章节详细介绍。

注意, 为了尽量简单, 示例中只改变了很少的输入参数, 此实验也没有在不同CPU数量或者不同的堆布局下进行测试。

#### Tuning for Latency(调优延迟指标)

假设有一个需求,**每次作业必须在 1000ms 内处理完成**。我们知道, 实际的作业处理只需要100 ms，简化后， 两者相减就可以算出对 GC暂停的延迟要求。现在需求变成:**GC暂停不能超过900ms**。这个问题很容易找到答案, 只需要解析GC日志文件, 并找出GC暂停中最大的那个暂停时间即可。

再来看测试所用的三个配置:

<colgroup data-id="c7104f7d-odLOoE9j"><col data-id="cdf32f61-NXanPMa5" span="1" width="239"><col data-id="cdf32f61-agXUo5B7" span="1" width="239"><col data-id="cdf32f61-1a1qqOLf" span="1" width="239"><col data-id="cdf32f61-k2joL2XZ" span="1" width="239"></colgroup>
| **堆内存大小(Heap)** | **GC算法(GC Algorithm)** | **有效时间比(Useful work)** | **最长停顿时间(Longest pause)** |
| -Xmx12g | -XX:+UseConcMarkSweepGC | 89.8% | **560 ms** |
| -Xmx12g | -XX:+UseParallelGC | 91.5% | 1,104 ms |
| -Xmx8g | -XX:+UseConcMarkSweepGC | 66.3% | 1,610 ms |

可以看到,其中有一个配置达到了要求。运行的参数为:



```
java -Xmx12g -XX:+UseConcMarkSweepGC Producer
```



对应的GC日志中,暂停时间最大为`560 ms`, 这达到了延迟指标`900 ms`的要求。如果还满足吞吐量和系统容量需求的话,就可以说成功达成了GC调优目标, 调优结束。

#### Tuning for Throughput(吞吐量调优)

假定吞吐量指标为:**每小时完成 1300万次操作处理**。同样是上面的配置, 其中有一种配置满足了需求:

<colgroup data-id="c7104f7d-XWFk01FQ"><col data-id="cdf32f61-bRSQkdW2" span="1" width="239"><col data-id="cdf32f61-AX9oFR9r" span="1" width="239"><col data-id="cdf32f61-6dlMCOOR" span="1" width="239"><col data-id="cdf32f61-rnF2Ml61" span="1" width="239"></colgroup>
| **堆内存大小(Heap)** | **GC算法(GC Algorithm)** | **有效时间比(Useful work)** | **最长停顿时间(Longest pause)** |
| -Xmx12g | -XX:+UseConcMarkSweepGC | 89.8% | 560 ms |
| -Xmx12g | -XX:+UseParallelGC | **91.5%** | 1,104 ms |
| -Xmx8g | -XX:+UseConcMarkSweepGC | 66.3% | 1,610 ms |

此配置对应的命令行参数为:



```
java -Xmx12g -XX:+UseParallelGC Producer
```



*   可以看到,GC占用了 8.5%的CPU时间,剩下的`91.5%`是有效的计算时间。为简单起见, 忽略示例中的其他安全点。现在需要考虑:

1.  每个CPU核心处理一次作业需要耗时`100ms`
2.  因此, 一分钟内每个核心可以执行 60,000 次操作(**每个job完成100次操作**)
3.  一小时内, 一个核心可以执行 360万次操作
4.  有四个CPU内核, 则每小时可以执行: 4 x 3.6M = 1440万次操作

理论上，通过简单的计算就可以得出结论, 每小时可以执行的操作数为:`14.4 M * 91.5% = 13,176,000`次, 满足需求。

值得一提的是, 假若还要满足延迟指标, 那就有问题了, 最坏情况下, GC暂停时间为`1,104 ms`, 最大延迟时间是前一种配置的两倍。

#### Tuning for Capacity(调优系统容量)

假设需要将软件部署到服务器上(commodity-class hardware), 配置为`4核10G`。这样的话, 系统容量的要求就变成: 最大的堆内存空间不能超过`8GB`。有了这个需求, 我们需要调整为第三套配置进行测试:

<colgroup data-id="c7104f7d-CM3OYb8V"><col data-id="cdf32f61-2q4eRIn9" span="1" width="239"><col data-id="cdf32f61-e9evDkSA" span="1" width="239"><col data-id="cdf32f61-32S42uAd" span="1" width="239"><col data-id="cdf32f61-VjMFsW5B" span="1" width="239"></colgroup>
| **堆内存大小(Heap)** | **GC算法(GC Algorithm)** | **有效时间比(Useful work)** | **最长停顿时间(Longest pause)** |
| -Xmx12g | -XX:+UseConcMarkSweepGC | 89.8% | 560 ms |
| -Xmx12g | -XX:+UseParallelGC | 91.5% | 1,104 ms |
| **-Xmx8g** | -XX:+UseConcMarkSweepGC | 66.3% | 1,610 ms |

程序可以通过如下参数执行:



```
java -Xmx8g -XX:+UseConcMarkSweepGC Producer
```



*   测试结果是延迟大幅增长, 吞吐量同样大幅降低:
*   现在,GC占用了更多的CPU资源, 这个配置只有`66.3%`的有效CPU时间。因此,这个配置让吞吐量从最好的情况**13,176,000 操作/小时**下降到**不足 9,547,200次操作/小时**.
*   最坏情况下的延迟变成了**1,610 ms**, 而不再是**560ms**。

通过对这三个维度的介绍, 你应该了解, 不是简单的进行“性能(performance)”优化, 而是需要从三种不同的维度来进行考虑, 测量, 并调优延迟和吞吐量, 此外还需要考虑系统容量的约束。

请继续阅读下一章:6\. GC 调优(工具篇) - GC参考手册

原文链接:[GC Tuning: Basics](https://plumbr.eu/handbook/gc-tuning)

翻译时间: 2016年02月06日

## 6\. GC 调优(工具篇) - GC参考手册

2017年02月23日 18:56:02

阅读数：6469

进行GC性能调优时, 需要明确了解, 当前的GC行为对系统和用户有多大的影响。有多种监控GC的工具和方法, 本章将逐一介绍常用的工具。

您应该已经阅读了前面的章节:

1.  垃圾收集简介 - GC参考手册
2.  Java中的垃圾收集 - GC参考手册
3.  GC 算法(基础篇) - GC参考手册
4.  GC 算法(实现篇) - GC参考手册
5.  GC 调优(基础篇) - GC参考手册

JVM 在程序执行的过程中, 提供了GC行为的原生数据。那么, 我们就可以利用这些原生数据来生成各种报告。原生数据(_raw data_) 包括:

*   各个内存池的当前使用情况,
*   各个内存池的总容量,
*   每次GC暂停的持续时间,
*   GC暂停在各个阶段的持续时间。

可以通过这些数据算出各种指标, 例如: 程序的内存分配率, 提升率等等。本章主要介绍如何获取原生数据。 后续的章节将对重要的派生指标(derived metrics)展开讨论, 并引入GC性能相关的话题。

## JMX API

从 JVM 运行时获取GC行为数据, 最简单的办法是使用标准[JMX API 接口](https://docs.oracle.com/javase/tutorial/jmx/index.html). JMX是获取 JVM内部运行时状态信息 的标准API. 可以编写程序代码, 通过 JMX API 来访问本程序所在的JVM，也可以通过JMX客户端执行(远程)访问。

最常见的 JMX客户端是[JConsole](http://docs.oracle.com/javase/7/docs/technotes/guides/management/jconsole.html)和[JVisualVM](http://docs.oracle.com/javase/7/docs/technotes/tools/share/jvisualvm.html)(可以安装各种插件,十分强大)。两个工具都是标准JDK的一部分, 而且很容易使用. 如果使用的是 JDK 7u40 及更高版本, 还可以使用另一个工具:[Java Mission Control](http://www.oracle.com/technetwork/java/javaseproducts/mission-control/java-mission-control-1998576.html)( 大致翻译为 Java控制中心,`jmc.exe`)。

> JVisualVM安装MBeans插件的步骤: 通过 工具(T) – 插件(G) – 可用插件 – 勾选VisualVM-MBeans – 安装 – 下一步 – 等待安装完成…… 其他插件的安装过程基本一致。

所有 JMX客户端都是独立的程序,可以连接到目标JVM上。目标JVM可以在本机, 也可能是远端JVM. 如果要连接远端JVM, 则目标JVM启动时必须指定特定的环境变量,以开启远程JMX连接/以及端口号。 示例如下:



```
java -Dcom.sun.management.jmxremote.port=5432 com.yourcompany.YourApp
```



在此处, JVM 打开端口`5432`以支持JMX连接。

通过 JVisualVM 连接到某个JVM以后, 切换到 MBeans 标签, 展开 “java.lang/GarbageCollector” . 就可以看到GC行为信息, 下图是 JVisualVM 中的截图:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224430.png)

下图是Java Mission Control 中的截图:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224439.png)

从以上截图中可以看到两款垃圾收集器。其中一款负责清理年轻代(**PS Scavenge**)，另一款负责清理老年代(**PS MarkSweep**); 列表中显示的就是垃圾收集器的名称。可以看到 , jmc 的功能和展示数据的方式更强大。

对所有的垃圾收集器, 通过 JMX API 获取的信息包括:

*   **CollectionCount**: 垃圾收集器执行的GC总次数,
*   **CollectionTime**: 收集器运行时间的累计。这个值等于所有GC事件持续时间的总和,
*   **LastGcInfo**: 最近一次GC事件的详细信息。包括 GC事件的持续时间(duration), 开始时间(startTime) 和 结束时间(endTime), 以及各个内存池在最近一次GC之前和之后的使用情况,
*   **MemoryPoolNames**: 各个内存池的名称,
*   **Name**: 垃圾收集器的名称
*   **ObjectName**: 由JMX规范定义的 MBean的名字,,
*   **Valid**: 此收集器是否有效。本人只见过 “`true`“的情况 (^_^)

根据经验, 这些信息对GC的性能来说,不能得出什么结论. 只有编写程序, 获取GC相关的 JMX 信息来进行统计和分析。 在下文可以看到, 一般也不怎么关注 MBean , 但 MBean 对于理解GC的原理倒是挺有用的。

## JVisualVM

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224451.png)

Visual GC 插件常用来监控本机运行的Java程序, 比如开发者和性能调优专家经常会使用此插件, 以快速获取程序运行时的GC信息。

![06_03_jvmsualvm-garbage-collection-monitoring.png](https://s4.51cto.com/images/blog/202106/25/eabd68ba262d004c4919475f00d8ec9c.png?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk= "06_03_jvmsualvm-garbage-collection-monitoring.png")

左侧的图表展示了各个内存池的使用情况: Metaspace/永久代, 老年代, Eden区以及两个存活区。

在右边, 顶部的两个图表与 GC无关, 显示的是 JIT编译时间 和 类加载时间。下面的6个图显示的是内存池的历史记录, 每个内存池的GC次数,GC总时间, 以及最大值，峰值, 当前使用情况。

再下面是 HistoGram, 显示了年轻代对象的年龄分布。至于对象的年龄监控(objects tenuring monitoring), 本章不进行讲解。

与纯粹的JMX工具相比, VisualGC 插件提供了更友好的界面, 如果没有其他趁手的工具, 请选择VisualGC. 本章接下来会介绍其他工具, 这些工具可以提供更多的信息, 以及更好的视角. 当然, 在“Profilers(分析器)”一节中，也会介绍 JVisualVM 的适用场景 —— 如: 分配分析(allocation profiling), 所以我们绝不会贬低哪一款工具, 关键还得看实际情况。

## jstat

[jstat](https://docs.oracle.com/javase/8/docs/technotes/tools/unix/jstat.html)也是标准JDK提供的一款监控工具(Java Virtual Machine statistics monitoring tool),可以统计各种指标。既可以连接到本地JVM,也可以连到远程JVM. 查看支持的指标和对应选项可以执行 “`jstat -options`” 。例如:



```
+-----------------+---------------------------------------------------------------+
|     Option      |                          Displays...                          |
+-----------------+---------------------------------------------------------------+
|class            | Statistics on the behavior of the class loader                |
|compiler         | Statistics  on  the behavior of the HotSpot Just-In-Time com- |
|                 | piler                                                         |
|gc               | Statistics on the behavior of the garbage collected heap      |
|gccapacity       | Statistics of the capacities of  the  generations  and  their |
|                 | corresponding spaces.                                         |
|gccause          | Summary  of  garbage collection statistics (same as -gcutil), |
|                 | with the cause  of  the  last  and  current  (if  applicable) |
|                 | garbage collection events.                                    |
|gcnew            | Statistics of the behavior of the new generation.             |
|gcnewcapacity    | Statistics of the sizes of the new generations and its corre- |
|                 | sponding spaces.                                              |
|gcold            | Statistics of the behavior of the old and  permanent  genera- |
|                 | tions.                                                        |
|gcoldcapacity    | Statistics of the sizes of the old generation.                |
|gcpermcapacity   | Statistics of the sizes of the permanent generation.          |
|gcutil           | Summary of garbage collection statistics.                     |
|printcompilation | Summary of garbage collection statistics.                     |
+-----------------+---------------------------------------------------------------+
```



*   jstat 对于快速确定GC行为是否健康非常有用。启动方式为: “`jstat -gc -t PID 1s`” , 其中,PID 就是要监视的Java进程ID。可以通过`jps`命令查看正在运行的Java进程列表。



```
jps

jstat -gc -t 2428 1s
```



以上命令的结果, 是 jstat 每秒向标准输出输出一行新内容, 比如:



```
Timestamp  S0C    S1C    S0U    S1U      EC       EU        OC         OU       MC     MU    CCSC   CCSU   YGC     YGCT    FGC    FGCT     GCT   
200.0    8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169344.0  21248.0 20534.3 3072.0 2807.7     34    0.720  658   133.684  134.404
201.0    8448.0 8448.0 8448.0  0.0   67712.0  67712.0   169344.0   169343.2  21248.0 20534.3 3072.0 2807.7     34    0.720  662   134.712  135.432
202.0    8448.0 8448.0 8102.5  0.0   67712.0  67598.5   169344.0   169343.6  21248.0 20534.3 3072.0 2807.7     34    0.720  667   135.840  136.559
203.0    8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
204.0    8448.0 8448.0 8126.3  0.0   67712.0  67702.2   169344.0   169343.6  21248.0 20547.2 3072.0 2807.7     34    0.720  669   136.178  136.898
205.0    8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
206.0    8448.0 8448.0 8134.6  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  671   136.234  136.954
207.0    8448.0 8448.0 8154.8  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  673   136.289  137.009
208.0    8448.0 8448.0 8154.8  0.0   67712.0  67712.0   169344.0   169343.5  21248.0 20547.2 3072.0 2807.7     34    0.720  673   136.289  137.009
```



稍微解释一下上面的内容。参考[jstat manpage](http://www.manpagez.com/man/1/jstat/), 我们可以知道:

*   jstat 连接到 JVM 的时间, 是JVM启动后的 200秒。此信息从第一行的 “**Timestamp**” 列得知。继续看下一行, jstat 每秒钟从JVM 接收一次信息, 也就是命令行参数中 “`1s`” 的含义。
*   从第一行的 “**YGC**” 列得知年轻代共执行了34次GC, 由 “**FGC**” 列得知整个堆内存已经执行了 658次 full GC。
*   年轻代的GC耗时总共为`0.720 秒`, 显示在“**YGCT**” 这一列。
*   Full GC 的总计耗时为`133.684 秒`, 由“**FGCT**”列得知。 这立马就吸引了我们的目光, 总的JVM 运行时间只有 200 秒,**但其中有 66% 的部分被 Full GC 消耗了**。

再看下一行, 问题就更明显了。

*   在接下来的一秒内共执行了 4 次 Full GC。参见 “**FGC**” 列.
*   这4次 Full GC 暂停占用了差不多 1秒的时间(根据**FGCT**列的差得知)。与第一行相比, Full GC 耗费了`928 毫秒`, 即`92.8%`的时间。
*   根据 “**OC**和 “**OU**” 列得知,**整个老年代的空间**为`169,344.0 KB`(“OC“), 在 4 次 Full GC 后依然占用了`169,344.2 KB`(“OU“)。用了`928ms`的时间却只释放了 800 字节的内存, 怎么看都觉得很不正常。

只看这两行的内容, 就知道程序出了很严重的问题。继续分析下一行, 可以确定问题依然存在,而且变得更糟。

JVM几乎完全卡住了(stalled), 因为GC占用了90%以上的计算资源。GC之后, 所有的老代空间仍然还在占用。事实上, 程序在一分钟以后就挂了, 抛出了 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” 错误。

可以看到, 通过 jstat 能很快发现对JVM健康极为不利的GC行为。一般来说, 只看 jstat 的输出就能快速发现以下问题:

*   最后一列 “**GCT**”, 与JVM的总运行时间 “**Timestamp**” 的比值, 就是GC 的开销。如果每一秒内, “**GCT**” 的值都会明显增大, 与总运行时间相比, 就暴露出GC开销过大的问题. 不同系统对GC开销有不同的容忍度, 由性能需求决定, 一般来讲, 超过`10%`的GC开销都是有问题的。
*   “**YGC**” 和 “**FGC**” 列的快速变化往往也是有问题的征兆。频繁的GC暂停会累积,并导致更多的线程停顿(stop-the-world pauses), 进而影响吞吐量。
*   如果看到 “**OU**” 列中,老年代的使用量约等于老年代的最大容量(**OC**), 并且不降低的话, 就表示虽然执行了老年代GC, 但基本上属于无效GC。

## GC日志(GC logs)

通过日志内容也可以得到GC相关的信息。因为GC日志模块内置于JVM中, 所以日志中包含了对GC活动最全面的描述。 这就是事实上的标准, 可作为GC性能评估和优化的最真实数据来源。

GC日志一般输出到文件之中, 是纯 text 格式的, 当然也可以打印到控制台。有多个可以控制GC日志的JVM参数。例如,可以打印每次GC的持续时间, 以及程序暂停时间(`-XX:+PrintGCApplicationStoppedTime`), 还有GC清理了多少引用类型(`-XX:+PrintReferenceGC`)。

要打印GC日志, 需要在启动脚本中指定以下参数:



```
-XX:+PrintGCTimeStamps -XX:+PrintGCDateStamps -XX:+PrintGCDetails -Xloggc:<filename>
```



以上参数指示JVM: 将所有GC事件打印到日志文件中, 输出每次GC的日期和时间戳。不同GC算法输出的内容略有不同. ParallelGC 输出的日志类似这样:



```
199.879: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1473386 secs] [Times: user=0.43 sys=0.01, real=0.15 secs]
200.027: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1567794 secs] [Times: user=0.41 sys=0.00, real=0.16 secs]
200.184: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1621946 secs] [Times: user=0.43 sys=0.00, real=0.16 secs]
200.346: [Full GC (Ergonomics) [PSYoungGen: 64000K->63998K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1547695 secs] [Times: user=0.41 sys=0.00, real=0.15 secs]
200.502: [Full GC (Ergonomics) [PSYoungGen: 64000K->63999K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1563071 secs] [Times: user=0.42 sys=0.01, real=0.16 secs]
200.659: [Full GC (Ergonomics) [PSYoungGen: 64000K->63999K(74240K)] [ParOldGen: 169318K->169318K(169472K)] 233318K->233317K(243712K), [Metaspace: 20427K->20427K(1067008K)], 0.1538778 secs] [Times: user=0.42 sys=0.00, real=0.16 secs]
```



在 “04\. GC算法:实现篇” 中详细介绍了这些格式, 如果对此不了解, 可以先阅读该章节。

分析以上日志内容, 可以得知:

*   这部分日志截取自JVM启动后200秒左右。
*   日志片段中显示, 在`780毫秒`以内, 因为垃圾回收 导致了5次 Full GC 暂停(去掉第六次暂停,这样更精确一些)。
*   这些暂停事件的总持续时间是`777毫秒`, 占总运行时间的**99.6%**。
*   在GC完成之后, 几乎所有的老年代空间(`169,472 KB`)依然被占用(`169,318 KB`)。

通过日志信息可以确定, 该应用的GC情况非常糟糕。JVM几乎完全停滞, 因为GC占用了超过`99%`的CPU时间。 而GC的结果是, 老年代空间仍然被占满, 这进一步肯定了我们的结论。 示例程序和jstat 小节中的是同一个, 几分钟之后系统就挂了, 抛出 “[java.lang.OutOfMemoryError: GC overhead limit exceeded](https://plumbr.eu/outofmemoryerror/gc-overhead-limit-exceeded)” 错误, 不用说, 问题是很严重的.

从此示例可以看出, GC日志对监控GC行为和JVM是否处于健康状态非常有用。一般情况下, 查看 GC 日志就可以快速确定以下症状:

*   GC开销太大。如果GC暂停的总时间很长, 就会损害系统的吞吐量。不同的系统允许不同比例的GC开销, 但一般认为, 正常范围在`10%`以内。
*   极个别的GC事件暂停时间过长。当某次GC暂停时间太长, 就会影响系统的延迟指标. 如果延迟指标规定交易必须在`1,000 ms`内完成, 那就不能容忍任何超过`1000毫秒`的GC暂停。
*   老年代的使用量超过限制。如果老年代空间在 Full GC 之后仍然接近全满, 那么GC就成为了性能瓶颈, 可能是内存太小, 也可能是存在内存泄漏。这种症状会让GC的开销暴增。

可以看到,GC日志中的信息非常详细。但除了这些简单的小程序, 生产系统一般都会生成大量的GC日志, 纯靠人工是很难阅读和进行解析的。

## GCViewer

我们可以自己编写解析器, 来将庞大的GC日志解析为直观易读的图形信息。 但很多时候自己写程序也不是个好办法, 因为各种GC算法的复杂性, 导致日志信息格式互相之间不太兼容。那么神器来了:[GCViewer](https://github.com/chewiebug/GCViewer)。

[GCViewer](https://github.com/chewiebug/GCViewer)是一款开源的GC日志分析工具。项目的 GitHub 主页对各项指标进行了完整的描述. 下面我们介绍最常用的一些指标。

第一步是获取GC日志文件。这些日志文件要能够反映系统在性能调优时的具体场景. 假若运营部门(operational department)反馈: 每周五下午,系统就运行缓慢, 不管GC是不是主要原因, 分析周一早晨的日志是没有多少意义的。

获取到日志文件之后, 就可以用 GCViewer 进行分析, 大致会看到类似下面的图形界面:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224526.png)

使用的命令行大致如下:



```
java -jar gcviewer_1.3.4.jar gc.log
```



当然, 如果不想打开程序界面,也可以在后面加上其他参数,直接将分析结果输出到文件。

命令大致如下:



```
java -jar gcviewer_1.3.4.jar gc.log summary.csv chart.png
```



以上命令将信息汇总到当前目录下的 Excel 文件`summary.csv`之中, 将图形信息保存为`chart.png`文件。

点击下载:gcviewer的jar包及使用示例

上图中, Chart 区域是对GC事件的图形化展示。包括各个内存池的大小和GC事件。上图中, 只有两个可视化指标: 蓝色线条表示堆内存的使用情况, 黑色的Bar则表示每次GC暂停时间的长短。

从图中可以看到, 内存使用量增长很快。一分钟左右就达到了堆内存的最大值. 堆内存几乎全部被消耗, 不能顺利分配新对象, 并引发频繁的 Full GC 事件. 这说明程序可能存在内存泄露, 或者启动时指定的内存空间不足。

从图中还可以看到 GC暂停的频率和持续时间。`30秒`之后, GC几乎不间断地运行,最长的暂停时间超过`1.4秒`。

在右边有三个选项卡。“`**Summary**`(摘要)” 中比较有用的是 “`Throughput`”(吞吐量百分比) 和 “`Number of GC pauses`”(GC暂停的次数), 以及“`Number of full GC pauses`”(Full GC 暂停的次数). 吞吐量显示了有效工作的时间比例, 剩下的部分就是GC的消耗。

以上示例中的吞吐量为`**6.28%**`。这意味着有`**93.72%**`

下一个有意思的地方是“**Pause**”(暂停)选项卡:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224540.png)

“`Pause`” 展示了GC暂停的总时间,平均值,最小值和最大值, 并且将 total 与minor/major 暂停分开统计。如果要优化程序的延迟指标, 这些统计可以很快判断出暂停时间是否过长。另外, 我们可以得出明确的信息: 累计暂停时间为`634.59 秒`, GC暂停的总次数为`3,938 次`, 这在`11分钟/660秒`的总运行时间里那不是一般的高。

更详细的GC暂停汇总信息, 请查看主界面中的 “**Event details**” 标签:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224553.png)

从“**Event details**” 标签中, 可以看到日志中所有重要的GC事件汇总:`普通GC停顿`和`Full GC 停顿次数`, 以及`并发执行数`,`非 stop-the-world 事件`等。此示例中, 可以看到一个明显的地方, Full GC 暂停严重影响了吞吐量和延迟, 依据是:`3,928 次 Full GC`, 暂停了`634秒`。

可以看到, GCViewer 能用图形界面快速展现异常的GC行为。一般来说, 图像化信息能迅速揭示以下症状:

*   低吞吐量。当应用的吞吐量下降到不能容忍的地步时, 有用工作的总时间就大量减少. 具体有多大的 “容忍度”(tolerable) 取决于具体场景。按照经验, 低于 90% 的有效时间就值得警惕了, 可能需要好好优化下GC。
*   单次GC的暂停时间过长。只要有一次GC停顿时间过长,就会影响程序的延迟指标. 例如, 延迟需求规定必须在 1000 ms以内完成交易, 那就不能容忍任何一次GC暂停超过1000毫秒。
*   堆内存使用率过高。如果老年代空间在 Full GC 之后仍然接近全满, 程序性能就会大幅降低, 可能是资源不足或者内存泄漏。这种症状会对吞吐量产生严重影响。

业界良心 —— 图形化展示的GC日志信息绝对是我们重磅推荐的。不用去阅读冗长而又复杂的GC日志,通过容易理解的图形, 也可以得到同样的信息。

## 分析器(Profilers)

下面介绍分析器([profilers](http://zeroturnaround.com/rebellabs/developer-productivity-report-2015-java-performance-survey-results/3/), Oracle官方翻译是:`抽样器`)。相对于前面的工具, 分析器只关心GC中的一部分领域. 本节我们也只关注分析器相关的GC功能。

首先警告 —— 不要认为分析器适用于所有的场景。分析器有时确实作用很大, 比如检测代码中的CPU热点时。但某些情况使用分析器不一定是个好方案。

对GC调优来说也是一样的。要检测是否因为GC而引起延迟或吞吐量问题时, 不需要使用分析器. 前面提到的工具(`jstat`或 原生/可视化GC日志)就能更好更快地检测出是否存在GC问题. 特别是从生产环境中收集性能数据时, 最好不要使用分析器, 因为性能开销非常大。

如果确实需要对GC进行优化, 那么分析器就可以派上用场了, 可以对 Object 的创建信息一目了然. 换个角度看, 如果GC暂停的原因不在某个内存池中, 那就只会是因为创建对象太多了。 所有分析器都能够跟踪对象分配(via allocation profiling), 根据内存分配的轨迹, 让你知道**实际驻留在内存中的是哪些对象**。

分配分析能定位到在哪个地方创建了大量的对象. 使用分析器辅助进行GC调优的好处是, 能确定哪种类型的对象最占用内存, 以及哪些线程创建了最多的对象。

下面我们通过实例介绍3种分配分析器:`**hprof**`,`**JVisualV**`**M**和`**AProf**`。实际上还有很多分析器可供选择, 有商业产品,也有免费工具, 但其功能和应用基本上都是类似的。

### hprof

[hprof 分析器](http://docs.oracle.com/javase/8/docs/technotes/samples/hprof.html)内置于JDK之中。 在各种环境下都可以使用, 一般优先使用这款工具。

要让`hprof`和程序一起运行, 需要修改启动脚本, 类似这样:



```
java -agentlib:hprof=heap=sites com.yourcompany.YourApplication
```



在程序退出时,会将分配信息dump(转储)到工作目录下的`java.hprof.txt`文件中。使用文本编辑器打开, 并搜索 “**SITES BEGIN**” 关键字, 可以看到:



```
SITES BEGIN (ordered by live bytes) Tue Dec  8 11:16:15 2015
          percent          live          alloc'ed  stack class
 rank   self  accum     bytes objs     bytes  objs trace name
    1  64.43% 4.43%   8370336 20121  27513408 66138 302116 int[]
    2  3.26% 88.49%    482976 20124   1587696 66154 302104 java.util.ArrayList
    3  1.76% 88.74%    241704 20121   1587312 66138 302115 eu.plumbr.demo.largeheap.ClonableClass0006
    ... 部分省略 ...

SITES END
```



从以上片段可以看到, allocations 是根据每次创建的对象数量来排序的。第一行显示所有对象中有`**64.43%**`的对象是整型数组(`int[]`), 在标识为`302116`的位置创建。搜索 “**TRACE 302116**” 可以看到:



```
TRACE 302116:   
    eu.plumbr.demo.largeheap.ClonableClass0006.<init>(GeneratorClass.java:11)
    sun.reflect.GeneratedConstructorAccessor7.newInstance(<Unknown Source>:Unknown line)
    sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
    java.lang.reflect.Constructor.newInstance(Constructor.java:422)
```



现在, 知道有`64.43%`的对象是整数数组, 在`ClonableClass0006`类的构造函数中, 第11行的位置, 接下来就可以优化代码, 以减少GC的压力。

### Java VisualVM

本章前面的第一部分, 在监控 JVM 的GC行为工具时介绍了 JVisualVM , 本节介绍其在分配分析上的应用。

JVisualVM 通过GUI的方式连接到正在运行的JVM。 连接上目标JVM之后 :

1.  打开 “工具” –> “选项” 菜单, 点击**性能分析(Profiler)**标签, 新增配置, 选择 Profiler 内存, 确保勾选了 “Record allocations stack traces”(记录分配栈跟踪)。
2.  勾选 “Settings”(设置) 复选框, 在内存设置标签下,修改预设配置。
3.  点击 “Memory”(内存) 按钮开始进行内存分析。
4.  让程序运行一段时间,以收集关于对象分配的足够信息。
5.  单击下方的 “Snapshot”(快照) 按钮。可以获取收集到的快照信息。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224729.png)

完成上面的步骤后, 可以得到类似这样的信息:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224743.png)

上图按照每个类被创建的对象数量多少来排序。看第一行可以知道, 创建的最多的对象是`int[]`数组. 鼠标右键单击这行, 就可以看到这些对象都在哪些地方创建的:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230404224753.png)

与`hprof`相比, JVisualVM 更加容易使用 —— 比如上面的截图中, 在一个地方就可以看到所有`int[]`的分配信息, 所以多次在同一处代码进行分配的情况就很容易发现。

### AProf

最重要的一款分析器,是由 Devexperts 开发的[**AProf**](https://code.devexperts.com/display/AProf/About+Aprof)。 内存分配分析器 AProf 也被打包为 Java agent 的形式。

用 AProf 分析应用程序, 需要修改 JVM 启动脚本,类似这样:



```
java -javaagent:/path-to/aprof.jar com.yourcompany.YourApplication
```



重启应用之后, 工作目录下会生成一个`aprof.txt`文件。此文件每分钟更新一次, 包含这样的信息:



```
========================================================================================================================
TOTAL allocation dump for 91,289 ms (0h01m31s)
Allocated 1,769,670,584 bytes in 24,868,088 objects of 425 classes in 2,127 locations
========================================================================================================================

Top allocation-inducing locations with the data types allocated from them
------------------------------------------------------------------------------------------------------------------------
eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 1,423,675,776 (80.44%) bytes in 17,113,721 (68.81%) objects (avg size 83 bytes)
    int[]: 711,322,976 (40.19%) bytes in 1,709,911 (6.87%) objects (avg size 416 bytes)
    char[]: 369,550,816 (20.88%) bytes in 5,132,759 (20.63%) objects (avg size 72 bytes)
    java.lang.reflect.Constructor: 136,800,000 (7.73%) bytes in 1,710,000 (6.87%) objects (avg size 80 bytes)
    java.lang.Object[]: 41,079,872 (2.32%) bytes in 1,710,712 (6.87%) objects (avg size 24 bytes)
    java.lang.String: 41,063,496 (2.32%) bytes in 1,710,979 (6.88%) objects (avg size 24 bytes)
    java.util.ArrayList: 41,050,680 (2.31%) bytes in 1,710,445 (6.87%) objects (avg size 24 bytes)
          ... cut for brevity ...
```



上面的输出是按照`size`进行排序的。可以看出,`80.44%`的 bytes 和`68.81%`的 objects 是在`ManyTargetsGarbageProducer.newRandomClassObject()`方法中分配的。 其中,**int[]**数组占用了`40.19%`的内存, 是最大的一个。

继续往下看, 会发现`allocation traces`(分配痕迹)相关的内容, 也是以 allocation size 排序的:



```
Top allocated data types with reverse location traces
------------------------------------------------------------------------------------------------------------------------
int[]: 725,306,304 (40.98%) bytes in 1,954,234 (7.85%) objects (avg size 371 bytes)
    eu.plumbr.demo.largeheap.ClonableClass0006.: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
        java.lang.reflect.Constructor.newInstance: 38,357,696 (2.16%) bytes in 92,206 (0.37%) objects (avg size 416 bytes)
            eu.plumbr.demo.largeheap.ManyTargetsGarbageProducer.newRandomClassObject: 38,357,280 (2.16%) bytes in 92,205 (0.37%) objects (avg size 416 bytes)
            java.lang.reflect.Constructor.newInstance: 416 (0.00%) bytes in 1 (0.00%) objects (avg size 416 bytes)
... cut for brevity ...
```



可以看到,`int[]`数组的分配, 在`ClonableClass0006`构造函数中继续增大。

和其他工具一样,`AProf`揭露了 分配的大小以及位置信息(`allocation size and locations`), 从而能够快速找到最耗内存的部分。在我们看来,**AProf**是最有用的分配分析器, 因为它只专注于内存分配, 所以做得最好。 当然, 这款工具是开源免费的, 资源开销也最小。

请继续阅读下一章:7\. GC 调优(实战篇) - GC参考手册

原文链接:[GC Tuning: Tooling](https://plumbr.eu/handbook/gc-tuning-measuring)

翻译时间: 2016年02月06日

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
