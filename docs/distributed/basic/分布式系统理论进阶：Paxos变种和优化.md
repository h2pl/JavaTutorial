# Table of Contents
  * [**引言**](#引言)
  * [**Multi Paxos**](#multi-paxos)
  * [**Fast Paxos**](#fast-paxos)
  * [**EPaxos**](#epaxos)
  * [**小结**](#小结)


本文转自：https://www.cnblogs.com/bangerlee/p/6189646.html

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本文也将同步到我的个人博客：
> www.how2playlife.com

更多Java技术文章将陆续在微信公众号【Java技术江湖】更新，敬请关注。

该系列博文会告诉你什么是分布式系统，这对后端工程师来说是很重要的一门学问，我们会逐步了解分布式理论中的基本概念，常见算法、以及一些较为复杂的分布式原理，同时也需要进一步了解zookeeper的实现，以及CAP、一致性原理等一些常见的分布式理论基础，以便让你更完整地了解分布式理论的基础，为后续学习分布式技术内容做好准备。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->  

## **引言**

[《分布式系统理论进阶 - Paxos》](http://www.cnblogs.com/bangerlee/p/5655754.html)中我们了解了Basic Paxos、Multi Paxos的基本原理，但如果想把Paxos应用于工程实践，了解基本原理还不够。

有很多基于Paxos的优化，在保证一致性协议正确(safety)的前提下，减少Paxos决议通信步骤、避免单点故障、实现节点负载均衡，从而降低时延、增加吞吐量、提升可用性，下面我们就来了解这些Paxos变种。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/116770-20161217185911917-43631009.jpg)

## **Multi Paxos**

首先我们来回顾一下Multi Paxos，Multi Paxos在Basic Paxos的基础上确定一系列值，其决议过程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/116770-20161218102045714-754820695.png)

phase1a: leader提交提议给acceptor

phase1b: acceptor返回最近一次接受的提议(即曾接受的最大的提议ID和对应的value)，未接受过提议则返回空

phase2a: leader收集acceptor的应答，分两种情况处理

phase2a.1: 如果应答内容都为空，则自由选择一个提议value

phase2a.2: 如果应答内容不为空，则选择应答里面ID最大的提议的value

phase2b: acceptor将决议同步给learner

Multi Paxos中leader用于避免活锁，但leader的存在会带来其他问题，一是如何选举和保持唯一leader(虽然无leader或多leader不影响一致性，但影响决议进程progress)，二是充当leader的节点会承担更多压力，如何均衡节点的负载。Mencius<sup>[1]</sup>提出节点轮流担任leader，以达到均衡负载的目的；租约(lease)可以帮助实现唯一leader，但leader故障情况下可导致服务短期不可用。

## **Fast Paxos**

在Multi Paxos中，proposer -> leader -> acceptor -> learner，从提议到完成决议共经过3次通信，能不能减少通信步骤？

对Multi Paxos phase2a，如果可以自由提议value，则可以让proposer直接发起提议、leader退出通信过程，变为proposer -> acceptor -> learner，这就是Fast Paxos<sup>[2]</sup>的由来。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/116770-20161218102011683-1409659558.png)

Multi Paxos里提议都由leader提出，因而不存在一次决议出现多个value，Fast Paxos里由proposer直接提议，一次决议里可能有多个proposer提议、出现多个value，即出现提议冲突(collision)。leader起到初始化决议进程(progress)和解决冲突的作用，当冲突发生时leader重新参与决议过程、回退到3次通信步骤。

Paxos自身隐含的一个特性也可以达到减少通信步骤的目标，如果acceptor上一次确定(chosen)的提议来自proposerA，则当次决议proposerA可以直接提议减少一次通信步骤。如果想实现这样的效果，需要在proposer、acceptor记录上一次决议确定(chosen)的历史，用以在提议前知道哪个proposer的提议上一次被确定、当次决议能不能节省一次通信步骤。

## **EPaxos**

除了从减少通信步骤的角度提高Paxos决议效率外，还有其他方面可以降低Paxos决议时延，比如Generalized Paxos<sup>[3]</sup>提出不冲突的提议(例如对不同key的写请求)可以同时决议、以降低Paxos时延。

更进一步地，EPaxos<sup>[4]</sup>(Egalitarian Paxos)提出一种既支持不冲突提议同时提交降低时延、还均衡各节点负载、同时将通信步骤减少到最少的Paxos优化方法。

为达到这些目标，EPaxos的实现有几个要点。一是EPaxos中没有全局的leader，而是每一次提议发起提议的proposer作为当次提议的leader(command leader)；二是不相互影响(interfere)的提议可以同时提交；三是跳过prepare，直接进入accept阶段。EPaxos决议的过程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/116770-20161218173608104-1507680298.png)

左侧展示了互不影响的两个update请求的决议过程，右侧展示了相互影响的两个update请求的决议。Multi Paxos、Mencius、EPaxos时延和吞吐量对比：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/116770-20161218180622104-945213222.png)

为判断决议是否相互影响，实现EPaxos得记录决议之间的依赖关系。

## **小结**

以上介绍了几个基于Paxos的变种，Mencius中节点轮流做leader、均衡节点负载，Fast Paxos减少一次通信步骤，Generalized Paxos允许互不影响的决议同时进行，EPaxos无全局leader、各节点平等分担负载。

优化无止境，对Paxos也一样，应用在不同场景和不同范围的Paxos变种和优化将继续不断出现。

[1][Mencius: Building Efficient Replicated State Machines for WANs](http://cseweb.ucsd.edu/classes/wi09/cse223a/mencius.pdf),Yanhua Mao,Flavio P. Junqueira,Keith Marzullo, 2018

[2][Fast Paxos](https://www.microsoft.com/en-us/research/wp-content/uploads/2016/02/tr-2005-112.pdf),Leslie Lamport, 2005

[3][Generalized Consensus and Paxos](http://diyhpl.us/~bryan/papers2/distributed/distributed-systems/generalized-consensus-and-paxos.2004.pdf),Leslie Lamport, 2004

[4][There Is More Consensus in Egalitarian Parliaments](http://sigops.org/sosp/sosp13/papers/p358-moraru.pdf),Iulian Moraru, David G. Andersen, Michael Kaminsky, 2013
