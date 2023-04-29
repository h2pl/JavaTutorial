## 简介

我们都知道 Seata 是一个分布式事务的解决方案，今天我们就来带大家了解一下什么是分布式事务，首先我们先来了解一下基础的知识——事务，我们先来了解一下事务的概念是什么。

### 基本概念

事务四部分构成— ACID：

*   A(Atomic)：原子性，构成事务的所有操作，要么全部执行成功，要么全部执行失败，不会出现部分成功或者部分失败的情况。
*   C(Consistency)：一致性，在事务执行前后，数据库的一致性约束没有被破坏，比如，小勇去银行取100块钱，取之前是600，取之后应该是400，取之前和取之后的数据为正确数值为一致性，如果取出100，而银行里面的钱没有减少，要么小勇要笑醒了，这个就没有达到一致性的要求。
*   I(Isolation):隔离性，数据库中的事务一般都是并发的，隔离性是只在并发的两个事务执行过程互不干扰，一个事务在执行过程中不能看到其他事务运行过程的中间状态，通过配置事务隔离级别可以避免脏读，重复读等问题。
*   D(Durability)：持久性，当事务完成之后，事务对数据的更改会被持久化到数据库，且不会回滚。

事务分为两部分：本地事务和分布式事务。

#### 本地事务：

在计算机系统中，比较多的是通过关系型数据库来控制事务，这是利用数据库本身的事务特性进行实现的，因为应用主要靠关系型数据库来维持事务，加上数据库和应用都在同一个服务器，所以基于关系型数据的事务又被称为本地事务。

#### 分布式事务：

分布式事务是指事务的参与者、支持事务的服务器、资源服务器以及事务管理者分别位于不同的分布式系统的不同节点之上，且属于不同的应用，分布式事务需要保证这些操作要么全部成功，要么全部失败，分布式事务就是为了保证在不同服务器上数据库数据的一致性。

Seata 的设计思路是将多个服务器的本地事务组成一个全局事务，下面若干个本地事务，都能满足ACID，最好形成一个整的分布式事务，操作分布式事务就像是操作本地事务一样。

分布式系统会把一个应用拆分为多个可独立部署的服务，服务于服务之间通常需要远程协作才能完成事务的操作，这种分布式系统环境下由于不同的服务之间通过网络远程协作完成的事务被称为分布式事务，例如供应链系统中，订单创建（生成订单、扣减库存、履约通知发货）等。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/6471b8f38c5920f6ace85580abd1567bc0f021.png "图片")

在上图中我们可以看出，只要涉及到操作多个数据源，就会产生事务的问题，我们在实际开发中应该要避免这这个你问题的出现，但是虽然系统的拓展，应用和应用之间必然会产生应用之间事务的分离，当微服务架构中，主要有MQ和Seata，在了解他们之前，我们先来了解一下分布式事务是怎样组成，以及如何实现的。

## 分布式事务

#### 分布式事务是什么？

分布式事务指的是事务的参与者，支持事务的服务器，资源服务器分别位于分布式系统的不同节点之上，通常一个分布式事物中会涉及到对多个数据源或业务系统的操作。

随着互联网的发展，从之前的单一项目逐渐向分布式服务做转换，现如今微服务在各个公司已经普遍存在，而当时的本地事务已经无法满足分布式应用的要求，因此分布式服务之间事务的协调就产生了问题，如果做到多个服务之间事务的操作，能够像本地事务一样遵循ACID原则，成为一个难题，但是在大牛们不断的探索下，终于找到了分布式事务存在两大理论依据：CAP定律和BASE理论。

### CAP定律

CAP定律由一致性(C)、可用性(A)、分区容错性(P)组成，在分布式系统中，不可能同时满足Consistency(一致性)/Availability(可用性)/Partition tolerance(分区容错性) 三个特性，最多只能同时满足其中两项。

*   一致性(C)：在分布式系统中所有的数据备份，在同一时刻保持一致的特性，所有的应用节点访问的都是同一份最新的数据副本。
*   可用性(A): 当集群中一部分节点故障以后，集群整体能够响应客户端的读写请求，对数据更新具备高可用性。
*   分区容错性(P): 如果系统在规定时间限制内不能达成数据的一致性，就表示要发生分区的情况，当前操作需要在C和A之间做出选择，让系统能够在遇到网络故障等情况的时候，任然能够保证对外提供满足一致性或者可用性的服务。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/a5ec7fb950052bd1116452eb0ca9f52fd4e5a5.png "图片")

在上图中我们可以看到，当我们用户去购物车里面点击下单结算的时候，首先会经过我们库存服务，判断库存是否足够，当库存满足，扣减库存以后，我们需要将数据同步到其他服务器上，这一步是为了保证数据的结果的一致性，这个时候如果网络产生波动了，我们的系统需要保证分区容错性，也就是我们必须容忍网络所带来的一些问题，此时想保证一致性，就需要舍弃可用性。

但是如果为了保证高可用性，那么在高并发的情况下，是无法保证在限定时间内给出响应，由于网络的不可靠，我们的订单服务可能无法拿到最新的数据，但是我们要给用户做出响应，那么也无法保证一致性，所以AP是无法保证强一致性的。

如果既想要保证高可用又想要保证一致性，必须在网络良好的情况下才能实现，那么解决方法只有一个，那就是需要将库存、订单、履约放到一起，但是这个就上去了我们微服务的作用，也就不再是分布式系统了。

在分布式系统中，分区容错性是必须存在的，我们只能在一致性和可用性上取舍，在这种条件下就诞生了BASE理论。

### BASE理论

BASE由 基本可用 (Basically Available)、软状态 (Soft state)和 最终一致性 (Eventually consistent) 三个构建而成，是对CAP中一致性和可用性权衡的结果，来源于对互联网系统分布式实践的总结，是基于CAP定理逐步演化而来的，核心四系那个是及时无法做到强一致性，但是每个应用都可以根据自身的业务特点，采用适当的方式来使系统达到最终一致性。

1.  基本可用：基本可用是指当分布式系统出现不可预知故障的时候，允许损失部分可用性，但是这里并不是说表示系统不可以用，主要体现为以下几点:

*   响应时间上的损失，在正常情况下，一个在线搜索引擎需要在0.5秒之内返回给用户响应的查询结果，但是由于出现故障，查询结果的响应时间增加了1-2秒。
*   系统功能上的损失，在正常情况下，一个电子商务网站上进行购物，消费者几乎能够顺利的完成每一单操作，但是在一些节日大促销购物高峰期的时候，由于网站上购买量的猛增，为了保证系统的稳定性，部分消费者可能会引导到一个临时降级处理的页面或者提示。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/a7dac0d93bc82c6c6c7076b77061ed1ebed415.png "图片")

基本可用的意思是，对于我们的核心服务是可以使用的，其他的服务可以适当的降低响应时间，甚至是进行服务降级处理，在当前中，库存和订单肯定是核心服务，至于我们的发货系统在当时只要保证基本可用就行，它的同步可以慢一点或者延迟更高，等待流量高峰过去以后，在进行恢复。

1.  软状态：软状态是指允许系统中的数据存在中间状态，并认为该中间状态的存在不会影响系统的整体可用性，即允许系统不用节点的数据副本之间进行数据同步的过程存在延时。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/610c63492ee5e0f1d30493958b54430c85a9f3.png "图片")

软状态的意思是说，当我们大量下单的时候，扣减库存时，流量激增，这个时候如果大量访问到库存或者订单中，可能会将系统弄垮，这个过程中我们可以允许数据的同步存在延迟，不影响整体系统的使用。

1.  最终一致性：最终一致性强调的是所有数据副本，在经过一段时间的同步之后，最终都能够达到一个一致的状态，因此，最终一致性的本质是需要系统保证最终数据能够达到一致，而不是需要实时保证系统的强一致性。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/a23602e9776d2c7bd3d507582538c32a471c8c.png "图片")

经过流量高峰期以后，经过一段时间的同步，从中间状态最后变成数据最终一致性，保证各个服务数据的一致性。

### 二阶段提交(2PC)

2PC即两阶段提交协议，是将整个事务流程分为两个阶段，P是指准备阶段，C是指提交阶段。

就好比我们去KCC买冰淇淋吃，那刚好有活动，第二杯半价，但是你是一个人，这个时候刚好有个小姐姐过来，正在考虑买不买冰淇淋吃，这个时候你和她提出了AA，也就会说只有当你和她都同意买这个的时候，才能购买到，如果两个人中有一个不同意那么就不能买这个冰淇淋吃。

**阶段一：**准备阶段 老板要求你先进行付款，你同意付款完成后，再要求女方付款，女方同意付款完成。

**阶段二：**提交阶段 都付款完成，老板出餐，两个人都吃到冰淇淋。

这个例子就组成了一个事务。如果男女双方有一个人拒绝付款，那么老板就不会出餐，并且会把已收取的钱原路退回。

整个事务过程是由事务管理器和参与者组成的，店老板就是事务管理器，你和那个女孩就是参与者，事务管理器决策整个分布式事务在计算机中关系数据支持两阶段提交协议：

*   准备阶段(Prepare phase)：事务管理器给每个参与者发送Prepare 消息，每个数据库参与者在本地执行事务，并写本地的Undo/Redo 日志，此时事务没有提交。

> undo日志是记录修改前的数据，用于数据库回滚。
>
> Redo 日志是记录修改后的数据，用于提交事务写入数据文件。

*   提交阶段(commit phase)：如果事务管理器收到了参与者的执行失败或者超时消息时，直接给每个参与者发送(Rollback) 消息，如果收到参与者都成功，发送(Commit) 参与者根据事务管理器的指令执行提交或者回滚操作，并释放事务处理过程中使用的资源。

#### 成功提交：

事务管理器向所有参与者发送事务内容，询问是否准备好了，等待参与者的响应，各个参与者事务节点执行事务操作，并将 Undo和Redo 信息记入事务日志中。如果参与者成功执行事务操作，反馈事务管理器YES操作，表示事务可以执行，假如协调者从所有的参与者或得反馈都是Yes响应，那么就会执行事务提交。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/48e1a2907174a1ed035541e548755d39490441.png "图片")

#### 失败：

假如任何一个参与者向事务管理器反馈了No指令，或者等待超时之后，事务管理器无法接收到所有参与者的反馈响应，那么中断事务，发送回滚请求，事务管理器向所有参与者节点发送 RollBack 请求，参与者接收到 RollBack 请求后，会利用在阶段一记录的Undo信息执行事务的回滚操作，在完成回滚之后释放事务执行期间占用的资源，参与者在完成事务回滚之后，向协调者发送ACK消息，事务管理器在接受到所有参与者反馈的ACK消息之后，完成事务中断。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/0928e8933a69a75eff4968f6989ca78ceaeb47.png "图片")

### 三阶段提交(3PC)

3PC 主要是为了解决两阶段提交协议的单点故障问题和缩小参与者阻塞范围。是二阶段提交（2PC）的改进版本，引入参与节点的超时机制之外，3PC把2PC的准备阶段分成事务询问（该阶段不会阻塞）和事务预提交,则三个阶段分别为 CanCommit、PreCommit、DoCommit。

#### CanCommit 询问状态

CanCommit阶段 协调者(Coordinator)会向参与者(Participant) 发送CanCommit消息，询问是否可以执行操作，参与者收到消息后，表示能够执行，会返回给协调者能够执行的(yes)命令。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c9cc9e8959b9a5393cb59672bc05a9121825ff.png "图片")

如果参与者不能执行，会返回No命令,释放资源，结束事务。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/61bf61d800a42fb1583689ccbc1b1e862fec29.png "图片")

#### PreCommit 预提交

PreCommit 阶段如果协调者收到参与者返回的状态值为YES，那么就证明它们都有能力去执行这个操作，那么协调者就会向所有参与者 发送 PreCommit 消息，协调者收到 PreCommit消息后，回去执行本地事务，如果执行成功会将本地事务保存到 undo和redo 后，再返回给协调者YES指令，如果执行本地事务失败，返回协调者No,只要协调者收到一个执行失败，给所有参与者发送中断事务消息，参与者收到消息后，对事务进行回滚操作。

在这个阶段参与者和协调者都引入了超时机制，如果参与者没有收到，协调者的消息，或者协调者没有收到参与者返回的预执行结果状态，在等待超时之后，事务会中断，避免了事务的阻塞。

协调者向参与者发送PreCommit，如果参与者执行成功，返回yes![图片](https://s2.51cto.com/oss/202206/18/91b004f982c531b8a641821cf1a49be547f9b3.gif "图片")。

如果参与者执行失败，只有有一个返回No到协调者，协调者会向参与者发送中断事务的消息，参与者回滚事务。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/996ca048361577f6ba066998a6b3ed9ac0450b.png "图片")

#### DoCommit 提交

协调者收到所有参与者返回的状态都是YES，这时协调者会向所有的参与者都发送 DoCommit ，参与者收到 DoCommit 后，会真正的提交事务，当事务提交成功后，返回协调者YES状态，表示我已经完成事务的提交了，协调者收到所有参与者都返回YES状态后，那么就完成了本次事务。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f4ab1eb14668202d8be313dc1b754833689c37.png "图片")

如果某个参与者返回No消息，协调者发送中断事务消息(abort)，给参与者们，参与者回滚事务。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/42ab62f59f9e71c66f7213a6833c2c7e436193.png "图片")

3PC是2PC的升级版，引入了超时机制，解决了单点故障引起的事务阻塞问题，但是3PC依然不能解决事务一致性的问题，因为在DoCommit阶段，如果由于网络或者超时等原则导致参与者收不到协调者发送过来的 中断事务消息(abort) ，过了这个时间后，参与者会提交事务，本来是应该进行回滚，提交事务后，会导致数据不一致的问题出现，2PC虽然在网络故障情况下存在强一致性被破坏的问题，但是故障恢复以后能保证最终一致性，3PC虽然有超时时间，解决了阻塞，提高了可用性，但是牺牲了一致性，如果针对网络波动问题导致数据问题这一点上，2PC是优于3PC的。

## Seata

官网：https://seata.io/zh-cn/docs/overview/what-is-seata.html。

Seata 是一款开源的分布式事务解决方案，致力于提供高性能和简单易用的分布式事务服务。Seata 将为用户提供了 AT、TCC、SAGA 和 XA 事务模式，为用户打造一站式的分布式解决方案。

![图片](https://s6.51cto.com/oss/202206/18/22906d30493df9861728958921200a254ef5d5.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/e335a42151fa17eb25e848defd449317e036d3.png "图片")

在微服务系统中，一般业务会被拆分成独立的模块，在官方提供的结构图中，我们可以看到当前主要分为三个模块。

*   库存服务：对于商品库存信息进行增加或者减少操作。
*   订单服务：根据用户指定商品生成订单。
*   账户服务：从用户账户中扣除余额，增加积分，维护地址信息等等。

在当前架构中，用户挑选心仪的商品下单后，需要三个服务来完成操作，每一个服务的内部都拥有一个独立的本地事务来保证当前服务数据的强一致性，但是三个服务组成的全局事务一致性就没办法进行保证，那么Seata就是来解决这个问题的。

### Seata术语

官网地址：https://seata.io/zh-cn/docs/overview/terminology.html。

在了解Seata之前，我们先来了解一下 Seata 几个关键的概念：

1.  TC(Transaction Coordinator)事务协调者：维护全局和分支事务的状态，驱动全局事务提交或者回滚。
2.  TM(Transaction Manager) 事务管理者： 发起者，同时一个RM的一种，定义全局事务的范围，开始全局事务，提交或回滚全局事务。
3.  RM(Resource Manager) 资源管理器：  参与事务的微服务，管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

### Seata 2PC

**一阶段：** 业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。

**二阶段：** 提交异步化，非常快速地完成。回滚通过一阶段的回滚日志进行反向补偿。

一阶段本地事务提交前，需要确保先拿到 全局锁 。拿不到全局锁 ，不能提交本地事务。拿全局锁的尝试被限制在一定范围内，超出范围将放弃，并回滚本地事务，释放本地锁。

在数据库本地事务隔离级别读已提交或以上的基础上，Seata（AT 模式）的默认全局隔离级别是 读未提交。

如果应用在特定场景下，必需要求全局的 读已提交 ，目前 Seata 的方式是通过 SELECT FOR UPDATE 语句的代理。

Seata执行流程分析：

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c7e10e315ebb9d8effa798730cebde46ec741a.png "图片")

每个RM 使用 DataSourceProxy 链接数据路，目的是使用 ConnectionProxy ，使用数据源和数据代理的目的是在第一阶段将 undo和业务数据放在一个本地事务中提交，这样就保存了只要有业务操作就一定会有dudo日志。

在第一阶段中，undo存放了数据修改前后修改的值，是为了事务回滚做好准备，在第一阶段完成就已经将分支事务提交了，也就释放了锁资源。

TM开启全局事务开始，将XID全局事务ID放在事务上下文中，通过feign调用将XID传入下游服务器中，每个分支事务将自己的 Branch ID分支事务ID和XID进行关联。

在第二阶段全局事务提交，TC会通知各个分支参与者提交分支事务，在第一阶段已经提交了分支事务，在这里各参与者只需要删除undo即可，并且可以异步执行。

如果某一个分支事务异常了，第二阶段全局事务回滚操作，TC会通知各个分支参与者回滚分支事务，通过XID和Branch-ID找到对应的回滚日志，通过回滚日志生成的反向SQL执行，完成分支事务回滚到之前的状态。

### Seata 下载安装

下载地址：https://github.com/seata/seata/releases。

![图片](https://s7.51cto.com/oss/202206/18/97b6c0b520c17b43b883003d10ef8e22d933a2.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/019d53972f031fdf1767425abf92f907f0f1c8.png "图片")

解压后找到conf目录。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/88a3fa94605fb2567a64954540711876f4a6df.png "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/6998821822b0d8ed7a44318b8b3390bc421303.png "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d157544494550bb1c865815e31ae990fbac40b.png "图片")

我们在启动seata之前，首先要启动nacos，其实也很简单，只需要下载nacos后启动就行，不知道nacos怎么操作的看这里的介绍nacos基础介绍，启动好之后，我们再来启动seata，bin目录下seata-server.bat![图片](https://s5.51cto.com/oss/202206/18/b8605b968056145dc6c740b5c95f346a98eaae.gif "图片")。

如果我们看到8091端口在监听，并且在nacos看到服务注册上去了，就表示我们seata启动成功了。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/96829da176e4fcca80a0230d5c14b601ac549b.png "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c253ebd69820f7d2daa259c97f27cdc70fd160.png "图片")

## 总结

到这里我们关于分布式事务的和seata的介绍就讲完了，其实关于分布式还有MQ实现可靠消息最终一致性，MQ主要解决了两个功能：本地事务与消息发送的原子性问题。事务参与方接收消息的可靠性。

## 前言

在上一节中我们讲解了，关于分布式事务和seata的基本介绍和使用，感兴趣的小伙伴可以回顾一下[《别再说你不知道分布式事务了!》](https://developer.51cto.com/article/711909.html) 最后小农也说了，下期会带给大家关于Seata中关于seata中AT、TCC、SAGA 和 XA 模式的介绍和使用，今天就来讲解关于Seata中分布式四种模型的介绍。

Seata分为三大模块，分别是 TM、RM 和 TC。

**TC (Transaction Coordinator) - 事务协调者：**维护全局和分支事务的状态，驱动全局事务提交或回滚。

**TM (Transaction Manager) - 事务管理器：**定义全局事务的范围：开始全局事务、提交或回滚全局事务。

**RM (Resource Manager) - 资源管理器：**管理分支事务处理的资源，与TC交谈以注册分支事务和报告分支事务的状态，并驱动分支事务提交或回滚。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c1ba09f1965be00dd7b817c1942fccfa2d7d25.png "图片")

在 Seata 中，分布式事务的执行流程：

*   TM 开启分布式事务（TM 向 TC 注册全局事务记录）。
*   按业务场景，编排数据库、服务等事务内资源（RM 向 TC 汇报资源准备状态 ）。
*   TM 结束分布式事务，事务一阶段结束（TM 通知 TC 提交/回滚分布式事务）。
*   TC 汇总事务信息，决定分布式事务是提交还是回滚。
*   TC 通知所有 RM 提交/回滚 资源，事务二阶段结束。

TM 和 RM 是作为 Seata 的客户端与业务系统集成在一起，TC 作为 Seata 的服务端独立部署。

服务端存储模式支持三种：

**file：** 单机模式，全局事务会话信息内存中读写并持久化本地文件root.data，性能较高（默认）。

**DB:**  高可用模式，全局事务会话信息通过DB共享，相对性能差一些。

**redis：** Seata-Server1.3及以上版本支持，性能较高，存在事务信息丢失风险，需要配合实际场景使用。

### TC环境搭建详解

这里我们使用DB高可用模式，找到conf/file.conf文件。

![图片](https://s9.51cto.com/oss/202207/03/84e02c1536f8c1431cc833b98958c1168f8658.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d9e1e2d06e2f4e53344238a2806fbc9bdbaf7b.png "图片")

修改以上中的信息，找到对应的db配置，修改其中的jdbc连接，要注意其中涉及到三个表（global_table，branch_table，lock_table），同时 mysql5和mysql8的驱动是不一样的。

> mysql5：com.mysql.jdbc.Driver。
>
> mysql8：com.mysql.cj.jdbc.Driver。

建表语句地址：https://github.com/seata/seata/blob/develop/script/server/db/mysql.sql。

**global_table：** 全局事务表，每当有一个全局事务发起后，就会在该表中记录全局事务的ID。

**branch_table：** 分支事务表，记录每一个分支事务的 ID，分支事务操作的哪个数据库等信息。

**lock_table：** 全局锁。

当上述配置好以后，重启Seata即可生效。

### Seata 配置 Nacos

Seata支持注册服务到Nacos，以及支持Seata所有配置放到Nacos配置中心，在Nacos中统一维护；在 高可用模式下就需要配合Nacos来完成

首先找到 conf/registry.conf，修改registry信息。









```
registry {
  # file 、nacos 、eureka、redis、zk、consul、etcd3、sofa
  type = "nacos"
  nacos {
    application = "seata-server" # 这里的配置要和客户端保持一致
    serverAddr = "127.0.0.1:8848"
    group = "SEATA_GROUP"  # 这里的配置要和客户端保持一致
    namespace = ""
    cluster = "default"
    username = "nacos"
    password = "nacos"
  }
  config {
  # file、nacos 、apollo、zk、consul、etcd3
  type = "nacos"
  nacos {
    serverAddr = "127.0.0.1:8848"
    namespace = ""
    group = "SEATA_GROUP"
    username = "nacos"
    password = "nacos"
    dataId = "seataServer.properties"
  }
  ......
}
```











修改好后，将seata中的一些配置上传到Nacos中，因为配置项比较多，所以官方提供了一个config.txt，只下载并且修改其中某些参数后，上传到Nacos中即可。

**下载地址：**https://github.com/seata/seata/tree/develop/script/config-center。

修改项如下：











```
service.vgroupMapping.mygroup=default # 事务分组
store.mode=db
store.db.driverClassName=com.mysql.cj.jdbc.Driver
store.db.url=jdbc:mysql://127.0.0.1:3306/seata?useUnicode=true&characterEncoding=UTF-8&serverTimezone=Asia/Shanghai
store.db.user=root
store.db.password=123456
```











修改好这个文件以后，把这个文件放到seata目录下：

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b42aeca797a2bcb8fb5791e836d0f6011e9848.png "图片")

把这些配置都加入到Nacos配置中，要借助一个脚本来进行执行，官方已经提供好。

**地址为：**https://github.com/seata/seata/blob/develop/script/config-center/nacos/nacos-config.sh。

新建一个nacos-config.sh文件，将脚本内容复制进去，修改congfig.txt的路径。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b876340233c7ac1623731359ea33164becafc1.png "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f563db6014ec0cb7f5a13952c63552c916da05.png "图片")

上述文件修改好后，打开git工具，将nacos-config.sh工具拖拽到窗体中即可或者使用命令。

> sh nacos-config.sh -h 127.0.0.1 -p 8848 -g SEATA_GROUP -t 88b8f583-43f9-4272-bd46-78a9f89c56e8 -u nacos -w nacos。
>
> -h：nacos地址。
>
> -p：端口，默认8848。
>
> -g：seata的服务列表分组名称。
>
> -t：nacos命名空间id。
>
> -u和-w：nacos的用户名和密码。

![图片](https://s3.51cto.com/oss/202207/03/b685868586971f9c3b1490e3fcf50041fc2f6b.gif "图片")![图片](https://s4.51cto.com/oss/202207/03/965180d92863b00defe5550f051ade40ac60e8.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/764bfaf385a67b25f732035cde02cc76bbcd20.png "图片")

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/110a73f54f3f0e565cd21630cf0e0cf4b80ee2.png "图片")

最后会有四个执行失败，是因为redis报错的关系，这个可以忽略，不影响正常使用。最后可以看到在Nacos中有很多的配置项，说明导入成功。再重新其中seata，成功监听到8091端口，表示前置工作都已经准备完成。

![图片](https://s7.51cto.com/oss/202207/03/573560212aa718ea82c317737eefc492908086.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f71535950882c017bbd05116c54c16ccb20f0e.png "图片")

### Seata的事务模式

Seata 定义了全局事务的框架，主要分为以下几步：

1.  TM 向 TC请求 发起(Begin)、提交(Commit)、回滚(Rollback)等全局事务。
2.  TM把代表全局事务的XID绑定到分支事务上。
3.  RM向TC注册，把分支事务关联到XID代表的全局事务中。
4.  RM把分支事务的执行结果上报给TC。
5.  TC发送分支提交（Branch Commit）或分支回滚（Branch Rollback）命令给RM。

![图片](https://s9.51cto.com/oss/202207/03/410c3b93872ba8a55a9532876f3afbe1853d88.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/a30dd41045c8db4e0c5757095c6d1ab52fd57e.png "图片")

Seata 的 全局事务 处理过程，分为两个阶段：

*   执行阶段 ：执行分支事务，并保证执行结果满足是_可回滚的（Rollbackable）_和_持久化的（Durable）_。
*   完成阶段：根据 执行阶段 结果形成的决议，应用通过 TM 发出的全局提交或回滚的请求给 TC，TC 命令 RM 驱动 分支事务 进行 Commit 或 Rollback。

Seata 的所谓事务模式是指：运行在 Seata 全局事务框架下的 分支事务 的行为模式。准确地讲，应该叫作 分支事务模式。

不同的 事务模式 区别在于 分支事务 使用不同的方式达到全局事务两个阶段的目标。即，回答以下两个问题：

*   执行阶段 ：如何执行并 保证 执行结果满足是_可回滚的（Rollbackable）_和_持久化的（Durable）_。
*   完成阶段：收到 TC 的命令后，如何做到分支的提交或回滚？

我们以AT模式为例：

*   执行阶段：

*   可回滚：根据 SQL 解析结果，记录回滚日志
*   持久化：回滚日志和业务 SQL 在同一个本地事务中提交到数据库

*   完成阶段：
*   分支提交：异步删除回滚日志记录
*   分支回滚：依据回滚日志进行反向补偿更新

接下来就进入重头戏，Seata四大模式的介绍。

## Seata-XA模式

Seata 1.2.0 版本发布了新的事务模型：XA模式，实现了对XA协议的支持。对于XA模式我们需要从三个点去解析它。

*   XA模式是什么。
*   为什么支持XA。
*   XA模式如何实现和使用。

### XA模式简介

首先需要知道XA模型是什么，XA 规范早在上世纪 90 年代初就被提出，用于解决分布式事务领域的问题，他也是最早的分布式事务处理方案，因为需要数据库内部也是支持XA模式的，比如MYSQL，XA模式具有强一致性的特点，因此他对数据库占用时间比较长，所以性能比较低。

XA模式属于两阶段提交。

1.  第一阶段进行事务注册，将事务注册到TC中，执行SQL语句。
2.  第二阶段TC判断无事务出错，通知所有事务提交，否则回滚。
3.  在第一到第二阶段过程中，事务一直占有数据库锁，因此性能比较低，但是所有事务要么一起提交，要么一起回滚，所以能实现强一致性。

无论是AT模式、TCC还是SAGA，这些模式的提出，都是源于XA规范对某些业务场景无法满足。

### 什么是XA协议

XA规范是X/OPEN组织定义的分布式事务处理（DTP，Distributed Transaction Processing）标准，XA规范描述了全局事务管理器和局部资源管理器之间的接口，XA规范的目的是允许多个资源(如数据库，应用服务器，消息队列等）在同一事务中访问，这样可以使 ACID 属性跨越应用程序而保持有效。

XA 规范 使用两阶段提交（2PC，Two-Phase Commit）来保证所有资源同时提交或回滚任何特定的事务。因为XA规范最早被提出，所以几乎所有的主流数据库都保有对XA规范的支持。

分布式事务DTP模型定义的角色如下：

*   AP：即应用程序，可以理解为使用DTP分布式事务的程序，例如订单服务、库存服务。
*   RM：资源管理器，可以理解为事务的参与者，一般情况下是指一个数据库的实例（MySql），通过资源管理器对该数据库进行控制，资源管理器控制着分支事务。
*   TM：事务管理器，负责协调和管理事务，事务管理器控制着全局事务，管理实务生命周期，并协调各个RM。全局事务是指分布式事务处理环境中，需要操作多个数据库共同完成一个工作，这个工作即是一个全局事务。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/85cb437444a4debfdbb835519a1b2fdd16ee7b.png "图片")

DTP模式定义TM和RM之间通讯的接口规范叫XA，简单理解为数据库提供的2PC接口协议，基于数据库的XA协议来实现的2PC又称为XA方案。

现在有应用程序(AP)持有订单库和库存库，应用程序(AP)通过TM通知订单库(RM)和库存库(RM)，进行扣减库存和生成订单，这个时候RM并没有提交事务，而且锁定资源。

当TM收到执行消息，如果有一方RM执行失败，分别向其他RM也发送回滚事务，回滚完毕，释放锁资源。

当TM收到执行消息，RM全部成功，向所有RM发起提交事务，提交完毕，释放锁资源。

![图片](https://s7.51cto.com/oss/202207/03/d264d0e03f64cdd85f1600518e2db31ee0b8a5.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/8298749403719fbb05d917a501c3dd3b1cc449.png "图片")

分布式通信协议XA规范，具体执行流程如下所示：

![图片](https://s7.51cto.com/oss/202207/03/3609cd043edf05941832252b7e7ce838b95837.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/9235cb212a65d21b72549719fdc8b7e4cdd340.png "图片")

第一步：AP创建了RM1,RM2的JDBC连接。

第二步：AP通知生成全局事物ID，并把RM1，RM2注册到全局事务ID。

第三步：执行二阶段协议中的第一阶段prepare。

第四步：根据prepare请求，决定整体提交或回滚。

但是对于XA而言，如果一个参与全局事务的资源“失联”了，那么就意味着TM收不到分支事务结束的命令，那么它锁定的数据，将会一直被锁定，从而产生死锁，这个也是Seata需要重点解决的问题。

在Seata定义的分布式事务架构中，利用事务资源(数据局、消息)等对XA协议进行支持，用XA协议的机制来管理分支事务。

![图片](https://s2.51cto.com/oss/202207/03/a93391c180ce2293be8525a483594c701ebcbd.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/52b22c57810685deea7412204aa069e494f863.png "图片")

*   执行阶段：

*   可回滚：业务SQL操作在XA分支中进行，有资源管理器对XA协议的支持来保证可回滚。
*   持久化：ZA分支完成以后，执行 XA prepare，同样，由资源对XA协议的支持来保证持久化。

*   完成阶段：

### XA存在的意义

*   分支提交：执行XA分支的commit。
*   分支回滚：执行XA分支的rollback。

Seata 已经支持了三大事务模式：AT\TCC\SAGA，这三个都是补偿型事务，补偿型事务处理你机制构建在 事务资源 之上(要么中间件层面，要么应用层)，事务资源本身对于分布式的事务是无感知的，这种对于分布式事务的无感知存在有一个根本性的问题，无法做到真正的全局一致性。

例如一个库存记录，在补偿型事务处理过程中，用80扣减为60，这个时候仓库管理员查询数据结果，看到的是60，之后因为异常回滚，库存回滚到原来的80，那么这个时候库存管理员看到的60，其实就是脏数据，而这个中间状态就是补偿型事务存在的脏数据。

和补偿型事务不同，XA协议要求事务资源 本身提供对规范和协议的支持，因为事务资源感知并参与分布式事务处理过程中，所以事务资源可以保证从任意视角对数据的访问有效隔离性，满足全局数据的一致性。

### XA模式的使用

**官方案例：**https://github.com/seata/seata-samples。

**项目名：**seata-samples。

**业务开始：**business-xa库存服务：stock-xa订单服务：order-xa账号服务：account-xa。

把这个项目案例下载下来以后，找到项目名为seata-xa的目录，里面有测试数据库的链接，如果不想用测试数据库，只需要修改官方文档中数据库配置信息即可。

首先关注的是 business-xa项目，更多的关注BusinessService.purchase()方法。










```
@GlobalTransactional
    public void purchase(String userId, String commodityCode, int orderCount, boolean rollback) {
        String xid = RootContext.getXID();
        LOGGER.info("New Transaction Begins: " + xid);       
        //调用库存减库存
        String result = stockFeignClient.deduct(commodityCode, orderCount);
        if (!SUCCESS.equals(result)) {
            throw new RuntimeException("库存服务调用失败,事务回滚!");
        }
        //生成订单
        result = orderFeignClient.create(userId, commodityCode, orderCount);
        if (!SUCCESS.equals(result)) {
            throw new RuntimeException("订单服务调用失败,事务回滚!");
        }
        if (rollback) {
            throw new RuntimeException("Force rollback ... ");
        }
    }
```











其实现方法较之前差不多，我们只需要在order-xa里面(OrderService.create)，添加人为错误(int i = 1/0;)。











```
public void create(String userId, String commodityCode, Integer count) {
        String xid = RootContext.getXID();
        LOGGER.info("create order in transaction: " + xid);
        int i = 1/0;
        // 定单总价 = 订购数量(count) * 商品单价(100)
        int orderMoney = count * 100;
        // 生成订单
        jdbcTemplate.update("insert order_tbl(user_id,commodity_code,count,money) values(?,?,?,?)",
            new Object[] {userId, commodityCode, count, orderMoney});
        // 调用账户余额扣减
        String result = accountFeignClient.reduce(userId, orderMoney);
        if (!SUCCESS.equals(result)) {
            throw new RuntimeException("Failed to call Account Service. ");
        }

    }
```











里面有一个方法可以进行XA模式和AT模式的转换OrderXADataSourceConfiguration.dataSource。










```
@Bean("dataSourceProxy")
    public DataSource dataSource(DruidDataSource druidDataSource) {
        // DataSourceProxy for AT mode
        // return new DataSourceProxy(druidDataSource);
        // DataSourceProxyXA for XA mode
        return new DataSourceProxyXA(druidDataSource);
    }
```











我们启动这四个服务，访问地址 http://localhost:8084/purchase。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d82378852259a962ae041377e8433a31eb8602.png "图片")

我们可以其中报错，然后再去看对应数据库的数据，没有发生更改，说明我们的XA模式生效了，当你dubug去看里面的库存服务的时候，当操作数据更改的时候，数据库里面其实也是没有记录的，因为XA是强一致性，只有当事务结束完成以后，才会更改其中的数据。

XA模式的加入，补齐了Seata在全局一致性场景下的缺口，形成了AT、TCC、Saga、XA 四大事务模式的版图，基本满足了所有场景分布式事务处理的需求。

其中XA和AT是无业务侵入的，而TCC和Saga是有一定业务侵入的。

## Seata-AT模式

先来介绍一下AT模式，AT模式是一种没有侵入的分布式事务的解决方案，在AT模式下，用户只需关注自己的业务SQL，用户的业务SQL作为一阶段，Seata框架会自动生成事务进行二阶段提交和回滚操作。

**两阶段提交协议的演变：**

*   一阶段：业务数据和回滚日志记录在同一个本地事务中提交，释放本地锁和连接资源。
*   二阶段：
    提交异步化，非常快速地完成。
    回滚通过一阶段的回滚日志进行反向补偿。

#### AT模式主要特点

1.  最终一致性。
2.  性能较XA高。
3.  只在第一阶段获取锁，在第一阶段进行提交后释放锁。

在一阶段中，Seata会拦截 业务SQL ，首先解析SQL语义，找到要操作的业务数据，在数据被操作前，保存下来记录 undo log，然后执行 业务SQL 更新数据，更新之后再次保存数据 redo log，最后生成行锁，这些操作都在本地数据库事务内完成，这样保证了一阶段的原子性。

相对一阶段，二阶段比较简单，负责整体的回滚和提交，如果之前的一阶段中有本地事务没有通过，那么就执行全局回滚，否在执行全局提交，回滚用到的就是一阶段记录的 undo Log ，通过回滚记录生成反向更新SQL并执行，以完成分支的回滚。当然事务完成后会释放所有资源和删除所有日志。

AT流程分为两阶段，主要逻辑全部在第一阶段，第二阶段主要做回滚或日志清理的工作。流程如下：

![图片](https://s9.51cto.com/oss/202207/03/c7d9f9e7001e9fca3d8309b146f3014ecfbf06.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/02880c46589d54a3f12449afa76a44e9b8e73a.png "图片")

从上图中我们可以看到，订单服务中TM向TC申请开启一个全局事务，一般通过@GlobalTransactional标注开启，TC会返回一个全局事务ID(XID)，订单服务在执行本地事务之前，RM会先向TC注册一个分支事务， 订单服务依次生成undo log 执行本地事务，生成redo log 提交本地事务，向TC汇报，事务执行OK。

订单服务发起远程调用，将事务ID传递给库存服务，库存服务在执行本地事务之前，先向TC注册分支事务，库存服务同样生成undo Log和redo Log，向TC汇报，事务状态成功。

如果正常全局提交，TC通知RM一步清理掉本地undo和redo日志，如果存在一个服务执行失败，那么发起回滚请求。通过undo log进行回滚。

在这里还会存在一个问题，因为每个事务从本地提交到通知回滚这段时间里面，可能这条数据已经被其他事务进行修改，如果直接用undo log进行回滚，可能会导致数据不一致的情况，

这个时候 RM会用 redo log进行验证，对比数据是否一样，从而得知数据是否有别的事务进行修改过，undo log是用于被修改前的数据，可以用来回滚，redolog是用于被修改后的数据，用于回滚校验。

如果数据没有被其他事务修改过，可以直接进行回滚，如果是脏数据，redolog校验后进行处理。

### 实战

了解了AT模型的基本操作，接下来就来实战操作一下，关于AT模型具体是如何实现的。首先设计两个服务 cloud-alibaba-seata-order 和 cloud-alibaba-seata-stock。

表结构t_order、t_stock和undo_log三张表，项目源码和表结构，加上undo_log表，此表用于数据的回滚，文末有链接。

cloud-alibaba-seata-order核心代码如下：

controller：










```
@RestController
public class OrderController {
    @Autowired
    private OrderService orderService;
    @GetMapping("order/create")
    @GlobalTransactional //开启分布式事务
    public String create(){
        orderService.create();
        return "订单创建成功！";
    }
}
```











OrderService：










```
public interface OrderService {
    void create();
}
```











StockClient：










```
@FeignClient(value = "seata-stock")
public interface StockClient {
    @GetMapping("/stock/reduce")
    String reduce();

}
```











OrderServiceImpl：










```
@Service
public class OrderServiceImpl implements OrderService{
    @Autowired
    private OrderMapper orderMapper;
    @Autowired
    private StockClient stockClient;
    @Override
    public void create() {
        //扣减库存
        stockClient.reduce();
        System.out.println("扣减库存成功！");
        //手工异常 用于回滚库存信息
        int i = 1/0;
        System.err.println("异常！");
        //创建订单
        orderMapper.createOrder();
        System.out.println("创建订单成功！");
    }
}
```











OrderMapper：










```
@Mapper
public interface OrderMapper {
    @Insert("insert into t_order (order_no,order_num) value (order_no+1,1)")
    void createOrder();
}
```











cloud-alibaba-seata-stock核心代码如下：










```
@RestController
public class StockController {
    @Autowired
    private StockService stockService;
    @GetMapping("stock/reduce")
    public String reduce(){
        stockService.reduce();
        return "库存数量已扣减："+ new Date();
    }
}
```




















```
public interface StockService {
    void reduce();
}
```




















```
@Service
public class StockServiceImpl implements StockService{
    @Autowired
    StockMapper stockMapper;
    @Override
    public void reduce() {
        stockMapper.reduce();
    }
}
```




















```
@Mapper
@Repository
public interface StockMapper {
    @Update("update t_stock set order_num = order_num - 1 where order_no = 1 ")
    void reduce();

}
```











代码都比较简单，我们就不做过多的描述，基本注释也都有，，首先我们需要将order和stock服务都跑起来，在之前我们的Nacos和Seata都要启动起来，这个时候我们访问order的Rest接口，http://localhost:8087/order/create,为了验证undo_log的表是用于存储回滚数据，我们在OrderServiceImpl.create()中添加断点，用debug的方式启动。

![图片](https://s5.51cto.com/oss/202207/03/e9df0649789b2ad3b0a9120c1992f9a7e55cb9.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/64e370d42b8375a8f7a1285cc1bc47b1437bff.png "图片")

然后访问http://localhost:8087/order/create，当程序卡在这个节点的时间，我们去看undo_log和库存表，会发现，库存确实减少了，而且undo_log也出现了对应的快照记录修改当前的数据信息，这个数据就是用来回滚的数据。

![图片](https://s8.51cto.com/oss/202207/03/9361fc3173dcb931a615608893af1f48fbeed0.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/567b564906d2c473d3d770087b745a5218587f.png "图片")

但是当我们F9通过以后，库存数量恢复，并且undo_log表的数据行也没有了，这个时候证明我们的Seata事务生效，回滚成功。

![图片](https://s8.51cto.com/oss/202207/03/66c11ca96212ce62540349442f14ccff8b87a4.gif "图片")![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/75cd0d103bd57d300e5194f66aa826f8494b94.png "图片")

到这里我们就验证了AT事务的执行过程，相比于XA和TCC等事务模型，Seata的AT模型可以应对大多数的业务场景，并且可以做到无业务侵入，开发者无感知，对于整个事务的协调、提交或者回滚操作，都可以通过AOP完成，开发者只需要关注业务即可。

由于Seata需要在不同的服务之间传递全局唯一的事务ID，和Dubbo等框架集成会比较友好，例如Dubbo可以用过隐士传参来进行事务ID的传递，整个事务ID的传播过程对开发者也可以做到无感知。

## Seata-TCC模式

具体使用案例：https://seata.io/zh-cn/blog/integrate-seata-tcc-mode-with-spring-cloud.html。

### 什么是TCC

TCC 是分布式事务中的二阶段提交协议，它的全称为 Try-Confirm-Cancel，即资源预留（Try）、确认操作（Confirm）、取消操作（Cancel），他们的具体含义如下：

1.  Try：对业务资源的检查并预留。
2.  Confirm：对业务处理进行提交，即 commit 操作，只要 Try 成功，那么该步骤一定成功。
3.  Cancel：对业务处理进行取消，即回滚操作，该步骤回对 Try 预留的资源进行释放。

TCC 是一种侵入式的分布式事务解决方案，以上三个操作都需要业务系统自行实现，对业务系统有着非常大的入侵性，设计相对复杂，但优点是 TCC 完全不依赖数据库，能够实现跨数据库、跨应用资源管理，对这些不同数据访问通过侵入式的编码方式实现一个原子操作，更好地解决了在各种复杂业务场景下的分布式事务问题。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1431eaa67529de68d4f9506ae25dc17c3882b8.png "图片")

### TCC和AT区别

AT 模式基于 支持本地 ACID 事务 的 关系型数据库：

*   一阶段 prepare 行为：在本地事务中，一并提交业务数据更新和相应回滚日志记录。
*   二阶段 commit 行为：马上成功结束，**自动**异步批量清理回滚日志。
*   二阶段 rollback 行为：通过回滚日志，**自动**生成补偿操作，完成数据回滚。

相应的，TCC 模式，不依赖于底层数据资源的事务支持：

*   一阶段 prepare 行为：调用**自定义**的 prepare 逻辑。
*   二阶段 commit 行为：调用**自定义**的 commit 逻辑。
*   二阶段 rollback 行为：调用**自定义**的 rollback 逻辑。

所谓 TCC 模式，是指支持把 **自定义** 的分支事务纳入到全局事务的管理中。

#### 特点：

1.  侵入性比较强，并且需要自己实现相关事务控制逻辑
2.  在整个过程基本没有锁，性能较强

## Seata-Saga模式

Saga模式是SEATA提供的长事务解决方案，在Saga模式中，业务流程中每个参与者都提交本地事务，当出现某一个参与者失败则补偿前面已经成功的参与者，一阶段正向服务和二阶段补偿服务（执行处理时候出错了，给一个修复的机会）都由业务开发实现。

Saga 模式下分布式事务通常是由事件驱动的，各个参与者之间是异步执行的，Saga 模式是一种长事务解决方案。

之前我们学习的Seata分布式三种操作模型中所使用的的微服务全部可以根据开发者的需求进行修改，但是在一些特殊环境下，比如老系统，封闭的系统（无法修改，同时没有任何分布式事务引入），那么AT、XA、TCC模型将全部不能使用，为了解决这样的问题，才引用了Saga模型。

比如：事务参与者可能是其他公司的服务或者是遗留系统，无法改造，可以使用Saga模式。

![图片](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/e923ef663d8c0a9de06072d96c6debbac49795.png "图片")

Saga模式是Seata提供的长事务解决方案，提供了异构系统的事务统一处理模型。在Saga模式中，所有的子业务都不在直接参与整体事务的处理（只负责本地事务的处理），而是全部交由了最终调用端来负责实现，而在进行总业务逻辑处理时，在某一个子业务出现问题时，则自动补偿全面已经成功的其他参与者，这样一阶段的正向服务调用和二阶段的服务补偿处理全部由总业务开发实现。

### Saga状态机

目前Seata提供的Saga模式只能通过状态机引擎来实现，需要开发者手工的进行Saga业务流程绘制，并且将其转换为Json配置文件，而后在程序运行时，将依据子配置文件实现业务处理以及服务补偿处理，而要想进行Saga状态图的绘制，一般需要通过Saga状态机来实现。

#### 基本原理：

*   通过状态图来定义服务调用的流程并生成json定义文件。
*   状态图中一个节点可以调用一个服务，节点可以配置它的补偿节点。
*   状态图 json 由状态机引擎驱动执行，当出现异常时状态引擎反向执行已成功节点对应的补偿节点将事务回滚。
*   可以实现服务编排需求，支持单项选择、并发、子流程、参数转换、参数映射、服务执行状态判断、异常捕获等功能。

#### Saga状态机的应用

**官方文档地址：**https://seata.io/zh-cn/docs/user/saga.html。

Seata Safa状态机可视化图形设计器使用地址：https://github.com/seata/seata/blob/develop/saga/seata-saga-statemachine-designer/README.zh-CN.md。

## 总结

总的来说在Seata的中AT模式基本可以满足百分之80的分布式事务的业务需求，AT模式实现的是最终一致性，所以可能存在中间状态，而XA模式实现的强一致性，所以效率较低一点，而Saga可以用来处理不同开发语言之间的分布式事务，所以关于分布式事务的四大模型，基本可以满足所有的业务场景，其中XA和AT没有业务侵入性，而Saga和TCC具有一定的业务侵入。

# 参考文章
https://www.51cto.com/article/713007.html
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud