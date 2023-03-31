本文转自：https://www.cnblogs.com/bangerlee/p/5328888.html

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本文也将同步到我的个人博客：
> www.how2playlife.com

更多Java技术文章将陆续在微信公众号【Java技术江湖】更新，敬请关注。

该系列博文会告诉你什么是分布式系统，这对后端工程师来说是很重要的一门学问，我们会逐步了解分布式理论中的基本概念，常见算法、以及一些较为复杂的分布式原理，同时也需要进一步了解zookeeper的实现，以及CAP、一致性原理等一些常见的分布式理论基础，以便让你更完整地了解分布式理论的基础，为后续学习分布式技术内容做好准备。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->
**引言**

CAP是分布式系统、特别是分布式存储领域中被讨论最多的理论，“[什么是CAP定理？](https://www.quora.com/What-Is-CAP-Theorem-1)”在Quora 分布式系统分类下排名 FAQ 的 No.1。CAP在程序员中也有较广的普及，它不仅仅是“C、A、P不能同时满足，最多只能3选2”，以下尝试综合各方观点，从发展历史、工程实践等角度讲述CAP理论。希望大家透过本文对CAP理论有更多地了解和认识。

**CAP定理**

CAP由[Eric Brewer](https://en.wikipedia.org/wiki/Eric_Brewer_(scientist))在2000年PODC会议上提出<sup>[1][2]</sup>，是Eric Brewer在Inktomi<sup>[3]</sup>期间研发搜索引擎、分布式web缓存时得出的关于数据一致性(consistency)、服务可用性(availability)、分区容错性(partition-tolerance)的猜想：

> It is impossible for a web service to provide the three following guarantees : Consistency, Availability and Partition-tolerance.

该猜想在提出两年后被证明成立<sup>[4]</sup>，成为我们熟知的CAP定理：

*   **数据一致性**(consistency)：如果系统对一个写操作返回成功，那么之后的读请求都必须读到这个新数据；如果返回失败，那么所有读操作都不能读到这个数据，对调用者而言数据具有强一致性(strong consistency) (又叫原子性 atomic、线性一致性 linearizable consistency)<sup>[5]</sup>
*   **服务可用性**(availability)：所有读写请求在一定时间内得到响应，可终止、不会一直等待
*   **分区容错性**(partition-tolerance)：在网络分区的情况下，被分隔的节点仍能正常对外服务

在某时刻如果满足AP，分隔的节点同时对外服务但不能相互通信，将导致状态不一致，即不能满足C；如果满足CP，网络分区的情况下为达成C，请求只能一直等待，即不满足A；如果要满足CA，在一定时间内要达到节点状态一致，要求不能出现网络分区，则不能满足P。

C、A、P三者最多只能满足其中两个，和FLP定理一样，CAP定理也指示了一个不可达的结果(impossibility result)。

![](https://images2015.cnblogs.com/blog/116770/201603/116770-20160329205542613-1908405713.jpg)

**CAP的工程启示**

CAP理论提出7、8年后，NoSql圈将CAP理论当作对抗传统关系型数据库的依据、阐明自己放宽对数据一致性(consistency)要求的正确性<sup>[6]</sup>，随后引起了大范围关于CAP理论的讨论。

CAP理论看似给我们出了一道3选2的选择题，但在工程实践中存在很多现实限制条件，需要我们做更多地考量与权衡，避免进入CAP认识误区<sup>[7]</sup>。

**1、关于 P 的理解**

Partition字面意思是网络分区，即因网络因素将系统分隔为多个单独的部分，有人可能会说，网络分区的情况发生概率非常小啊，是不是不用考虑P，保证CA就好<sup>[8]</sup>。要理解P，我们看回CAP证明<sup>[4]</sup>中P的定义：

> In order to model partition tolerance, the network will be allowed to lose arbitrarily many messages sent from one node to another.

网络分区的情况符合该定义，网络丢包的情况也符合以上定义，另外节点宕机，其他节点发往宕机节点的包也将丢失，这种情况同样符合定义。现实情况下我们面对的是一个不可靠的网络、有一定概率宕机的设备，这两个因素都会导致Partition，因而分布式系统实现中 P 是一个必须项，而不是可选项<sup>[9][10]</sup>。

对于分布式系统工程实践，CAP理论更合适的描述是：在满足分区容错的前提下，没有算法能同时满足数据一致性和服务可用性<sup>[11]</sup>：

> In a network subject to communication failures, it is impossible for any web service to implement an atomic read/write shared memory that guarantees a response to every request.

**2、CA非0/1的选择**

P 是必选项，那3选2的选择题不就变成数据一致性(consistency)、服务可用性(availability) 2选1？工程实践中一致性有不同程度，可用性也有不同等级，在保证分区容错性的前提下，放宽约束后可以兼顾一致性和可用性，两者不是非此即彼<sup>[12]</sup>。

![](https://images2015.cnblogs.com/blog/116770/201604/116770-20160401221124957-2025686892.jpg)

CAP定理证明中的一致性指强一致性，强一致性要求多节点组成的被调要能像单节点一样运作、操作具备原子性，数据在时间、时序上都有要求。如果放宽这些要求，还有其他一致性类型：

*   序列一致性(sequential consistency)<sup>[13]</sup>：不要求时序一致，A操作先于B操作，在B操作后如果所有调用端读操作得到A操作的结果，满足序列一致性
*   最终一致性(eventual consistency)<sup>[14]</sup>：放宽对时间的要求，在被调完成操作响应后的某个时间点，被调多个节点的数据最终达成一致

可用性在CAP定理里指所有读写操作必须要能终止，实际应用中从主调、被调两个不同的视角，可用性具有不同的含义。当P(网络分区)出现时，主调可以只支持读操作，通过牺牲部分可用性达成数据一致。

工程实践中，较常见的做法是通过异步拷贝副本(asynchronous replication)、quorum/NRW，实现在调用端看来数据强一致、被调端最终一致，在调用端看来服务可用、被调端允许部分节点不可用(或被网络分隔)的效果<sup>[15]</sup>。

**3、跳出CAP**

CAP理论对实现分布式系统具有指导意义，但CAP理论并没有涵盖分布式工程实践中的所有重要因素。

例如延时(latency)，它是衡量系统可用性、与用户体验直接相关的一项重要指标<sup>[16]</sup>。CAP理论中的可用性要求操作能终止、不无休止地进行，除此之外，我们还关心到底需要多长时间能结束操作，这就是延时，它值得我们设计、实现分布式系统时单列出来考虑。

延时与数据一致性也是一对“冤家”，如果要达到强一致性、多个副本数据一致，必然增加延时。加上延时的考量，我们得到一个CAP理论的修改版本PACELC<sup>[17]</sup>：如果出现P(网络分区)，如何在A(服务可用性)、C(数据一致性)之间选择；否则，如何在L(延时)、C(数据一致性)之间选择。

**小结**

以上介绍了CAP理论的源起和发展，介绍了CAP理论给分布式系统工程实践带来的启示。

CAP理论对分布式系统实现有非常重大的影响，我们可以根据自身的业务特点，在数据一致性和服务可用性之间作出倾向性地选择。通过放松约束条件，我们可以实现在不同时间点满足CAP(此CAP非CAP定理中的CAP，如C替换为最终一致性)<sup>[18][19][20]</sup>。

有非常非常多文章讨论和研究CAP理论，希望这篇对你认识和了解CAP理论有帮助。

[1] [Harvest, Yield, and Scalable Tolerant Systems](https://cs.uwaterloo.ca/~brecht/servers/readings-new2/harvest-yield.pdf), Armando Fox , Eric Brewer, 1999

[2] [Towards Robust Distributed Systems](http://www.cs.berkeley.edu/~brewer/cs262b-2004/PODC-keynote.pdf), Eric Brewer, 2000

[3] [Inktomi's wild ride - A personal view of the Internet bubble](https://www.youtube.com/watch?v=E91oEn1bnXM), Eric Brewer, 2004

[4] [Brewer’s Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web](https://pdfs.semanticscholar.org/24ce/ce61e2128780072bc58f90b8ba47f624bc27.pdf), Seth Gilbert, Nancy Lynch, 2002

[5] [Linearizability: A Correctness Condition for Concurrent Objects](http://cs.brown.edu/~mph/HerlihyW90/p463-herlihy.pdf), Maurice P. Herlihy,Jeannette M. Wing, 1990

[6] [Brewer's CAP Theorem - The kool aid Amazon and Ebay have been drinking](http://julianbrowne.com/article/viewer/brewers-cap-theorem), Julian Browne, 2009

[7] [CAP Theorem between Claims and Misunderstandings: What is to be Sacrificed?](http://www.sersc.org/journals/IJAST/vol56/1.pdf), Balla Wade Diack,Samba Ndiaye,Yahya Slimani, 2013

[8] [Errors in Database Systems, Eventual Consistency, and the CAP Theorem](http://cacm.acm.org/blogs/blog-cacm/83396-errors-in-database-systems-eventual-consistency-and-the-cap-theorem/fulltext), Michael Stonebraker, 2010

[9] [CAP Confusion: Problems with 'partition tolerance'](http://blog.cloudera.com/blog/2010/04/cap-confusion-problems-with-partition-tolerance/), Henry Robinson, 2010

[10] [You Can’t Sacrifice Partition Tolerance](https://codahale.com/you-cant-sacrifice-partition-tolerance/), Coda Hale, 2010

[11] [Perspectives on the CAP Theorem](https://groups.csail.mit.edu/tds/papers/Gilbert/Brewer2.pdf), Seth Gilbert, Nancy Lynch, 2012

[12] [CAP Twelve Years Later: How the "Rules" Have Changed](https://www.computer.org/cms/Computer.org/ComputingNow/homepage/2012/0512/T_CO2_CAP12YearsLater.pdf), Eric Brewer, 2012

[13] [How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Programs](http://research.microsoft.com/en-us/um/people/lamport/pubs/multi.pdf), Lamport Leslie, 1979

[14] [Eventual Consistent Databases: State of the Art](http://www.ronpub.com/publications/OJDB-v1i1n03_Elbushra.pdf), Mawahib Elbushra , Jan Lindström, 2014

[15] [Eventually Consistent](http://www.allthingsdistributed.com/2008/12/eventually_consistent.html), Werner Vogels, 2008

[16] [Speed Matters for Google Web Search](http://www.isaacsunyer.com/wp-content/uploads/2009/09/test_velocidad_google.pdf), Jake Brutlag, 2009

[17] [Consistency Tradeoffs in Modern Distributed Database System Design](http://cs-www.cs.yale.edu/homes/dna/papers/abadi-pacelc.pdf), Daniel J. Abadi, 2012

[18] [A CAP Solution (Proving Brewer Wrong)](http://guysblogspot.blogspot.com/2008/09/cap-solution-proving-brewer-wrong.html), Guy's blog, 2008

[19] [How to beat the CAP theorem](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html), nathanmarz , 2011

[20] [The CAP FAQ](https://github.com/henryr/cap-faq), Henry Robinson



