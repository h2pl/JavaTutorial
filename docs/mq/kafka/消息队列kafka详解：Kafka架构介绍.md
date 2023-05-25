## 一. 工作流程

Kafka中消息是以topic进行分类的，Producer生产消息，Consumer消费消息，都是面向topic的。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-b9d626794f6625526598db6627b780e7_720w.webp)

</figure>

Topic是逻辑上的改变，Partition是物理上的概念，每个Partition对应着一个log文件，该log文件中存储的就是producer生产的数据，topic=N*partition；partition=log

Producer生产的数据会被不断的追加到该log文件的末端，且每条数据都有自己的offset，consumer组中的每个consumer，都会实时记录自己消费到了哪个offset，以便出错恢复的时候，可以从上次的位置继续消费。流程：Producer => Topic（Log with offset）=> Consumer.

## 二. 文件存储

Kafka文件存储也是通过本地落盘的方式存储的，主要是通过相应的log与index等文件保存具体的消息文件。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-116ebd7dffd85595d69f080e5b5f6948_720w.webp)

</figure>

生产者不断的向log文件追加消息文件，为了防止log文件过大导致定位效率低下，Kafka的log文件以1G为一个分界点，当.log文件大小超过1G的时候，此时会创建一个新的.log文件，同时为了快速定位大文件中消息位置，Kafka采取了分片和索引的机制来加速定位。

在kafka的存储log的地方，即文件的地方，会存在消费的偏移量以及具体的分区信息，分区信息的话主要包括.index和.log文件组成

<figure data-size="normal">


![](https://pic3.zhimg.com/80/v2-c6de61f43ecbe58d4f3e7aa29541220e_720w.webp)

</figure>

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-8345e4966d8c5274a1e74e29151bd9c6_720w.webp)

</figure>

副本目的是为了备份，所以同一个分区存储在不同的broker上，即当third-2存在当前机器kafka01上，实际上再kafka03中也有这个分区的文件（副本），分区中包含副本，即一个分区可以设置多个副本，副本中有一个是leader，其余为follower。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-6e8de9e7dcbdac0b7bd424eaaf4f8568_720w.webp)

</figure>

如果.log文件超出大小，则会产生新的.log文件。如下所示:



```
00000000000000000000.index
00000000000000000000.log
00000000000000170410.index
00000000000000170410.log
00000000000000239430.index
00000000000000239430.log

```



**此时如何快速定位数据，步骤：**



```
.index文件存储的消息的offset+真实的起始偏移量。.log中存放的是真实的数据。

```



首先通过二分查找.index文件到查找到当前消息具体的偏移，如上图所示，查找为2，发现第二个文件为6，则定位到一个文件中。 然后通过第一个.index文件通过seek定位元素的位置3，定位到之后获取起始偏移量+当前文件大小=总的偏移量。 获取到总的偏移量之后，直接定位到.log文件即可快速获得当前消息大小。

## 三. 生产者分区策略

**分区的原因** 1\. 方便在集群中扩展：每个partition通过调整以适应它所在的机器，而一个Topic又可以有多个partition组成，因此整个集群可以适应适合的数据。 2\. 可以提高并发：以Partition为单位进行读写。类似于多路。

**分区的原则** 1\. 指明partition（这里的指明是指第几个分区）的情况下，直接将指明的值作为partition的值 2\. 没有指明partition的情况下，但是存在值key，此时将key的hash值与topic的partition总数进行取余得到partition值 3\. 值与partition均无的情况下，第一次调用时随机生成一个整数，后面每次调用在这个整数上自增，将这个值与topic可用的partition总数取余得到partition值，即round-robin算法。

## 四. 生产者ISR

为保证producer发送的数据能够可靠的发送到指定的topic中，topic的每个partition收到producer发送的数据后，都需要向producer发送ackacknowledgement，如果producer收到ack就会进行下一轮的发送，否则重新发送数据。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-409ea1af4f66bd2f44398850cc2ba9e2_720w.webp)

</figure>

**发送ack的时机** 确保有follower与leader同步完成，leader再发送ack，这样可以保证在leader挂掉之后，follower中可以选出新的leader（主要是确保follower中数据不丢失）

**follower同步完成多少才发送ack** 1\. 半数以上的follower同步完成，即可发送ack 2\. 全部的follower同步完成，才可以发送ack

## 4.1 副本数据同步策略

### 4.1.1 半数follower同步完成即发送ack

优点是延迟低

缺点是选举新的leader的时候，容忍n台节点的故障，需要2n+1个副本（因为需要半数同意，所以故障的时候，能够选举的前提是剩下的副本超过半数），容错率为1/2

### 4.1.2 全部follower同步完成完成发送ack

优点是容错率搞，选举新的leader的时候，容忍n台节点的故障只需要n+1个副本即可，因为只需要剩下的一个人同意即可发送ack了

缺点是延迟高，因为需要全部副本同步完成才可

### 4.1.3 kafka的选择

kafka选择的是第二种，因为在容器率上面更加有优势，同时对于分区的数据而言，每个分区都有大量的数据，第一种方案会造成大量数据的冗余。虽然第二种网络延迟较高，但是网络延迟对于Kafka的影响较小。

## 4.2 ISR(同步副本集)

**猜想** 采用了第二种方案进行同步ack之后，如果leader收到数据，所有的follower开始同步数据，但有一个follower因为某种故障，迟迟不能够与leader进行同步，那么leader就要一直等待下去，直到它同步完成，才可以发送ack，此时需要如何解决这个问题呢？

**解决** leader中维护了一个动态的ISR（in-sync replica set），即与leader保持同步的follower集合，当ISR中的follower完成数据的同步之后，给leader发送ack，如果follower长时间没有向leader同步数据，则该follower将从ISR中被踢出，该之间阈值由replica.lag.time.max.ms参数设定。当leader发生故障之后，会从ISR中选举出新的leader。

## 五. 生产者ack机制

对于某些不太重要的数据，对数据的可靠性要求不是很高，能够容忍数据的少量丢失，所以没有必要等到ISR中所有的follower全部接受成功。

Kafka为用户提供了三种可靠性级别，用户根据可靠性和延迟的要求进行权衡选择不同的配置。

**ack参数配置** 0：producer不等待broker的ack，这一操作提供了最低的延迟，broker接收到还没有写入磁盘就已经返回，当broker故障时有可能丢失数据

1：producer等待broker的ack，partition的leader落盘成功后返回ack，如果在follower同步成功之前leader故障，那么将丢失数据。（只是leader落盘）

<figure data-size="normal">


![](https://pic1.zhimg.com/80/v2-a219d261edd97432347f4edf5794e170_720w.webp)

</figure>

-1(all)：producer等待broker的ack，partition的leader和ISR的follower全部落盘成功才返回ack，但是如果在follower同步完成后，broker发送ack之前，如果leader发生故障，会造成数据重复。(这里的数据重复是因为没有收到，所以继续重发导致的数据重复)

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-c9741a10f418f7ea4eed929f0f266bbb_720w.webp)

</figure>

producer返ack，0无落盘直接返，1只leader落盘然后返，-1全部落盘然后返

## 六. 数据一致性问题

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-031d84a2012f64b122dd64ab67a4e52a_720w.webp)

</figure>

LEO(Log End Offset)：每个副本最后的一个offset HW(High Watermark)：高水位，指代消费者能见到的最大的offset，ISR队列中最小的LEO。

**follower故障和leader故障** 1\. follower故障：follower发生故障后会被临时踢出ISR，等待该follower恢复后，follower会读取本地磁盘记录的上次的HW，并将log文件高于HW的部分截取掉，从HW开始向leader进行同步，等待该follower的LEO大于等于该partition的HW，即follower追上leader之后，就可以重新加入ISR了。 2\. leader故障：leader发生故障之后，会从ISR中选出一个新的leader，为了保证多个副本之间的数据的一致性，其余的follower会先将各自的log文件高于HW的部分截掉，然后从新的leader中同步数据。

**这只能保证副本之间的数据一致性，并不能保证数据不丢失或者不重复**

## 七. ExactlyOnce

将服务器的ACK级别设置为-1（all），可以保证producer到Server之间不会丢失数据，即At Least Once至少一次语义。将服务器ACK级别设置为0，可以保证生产者每条消息只会被发送一次，即At Most Once至多一次。

At Least Once可以保证数据不丢失，但是不能保证数据不重复，而At Most Once可以保证数据不重复，但是不能保证数据不丢失，对于重要的数据，则要求数据不重复也不丢失，即Exactly Once即精确的一次。

在0.11版本的Kafka之前，只能保证数据不丢失，在下游对数据的重复进行去重操作，多余多个下游应用的情况，则分别进行全局去重，对性能有很大影响。

0.11版本的kafka，引入了一项重大特性：幂等性，幂等性指代Producer不论向Server发送了多少次重复数据，Server端都只会持久化一条数据。幂等性结合At Least Once语义就构成了Kafka的Exactly Once语义。

启用幂等性，即在Producer的参数中设置enable.idempotence=true即可，Kafka的幂等性实现实际是将之前的去重操作放在了数据上游来做，开启幂等性的Producer在初始化的时候会被分配一个PID，发往同一个Partition的消息会附带Sequence Number，而Broker端会对做缓存，当具有相同主键的消息的时候，Broker只会持久化一条。

但PID在重启之后会发生变化，同时不同的Partition也具有不同的主键，所以幂等性无法保证跨分区跨会话的Exactly Once。

要解决跨分区跨会话的Exactly Once，就引入了生产者事务的概念。

## 八. Kafka消费者分区分配策略

**消费方式:** consumer采用pull拉的方式来从broker中读取数据。

push推的模式很难适应消费速率不同的消费者，因为消息发送率是由broker决定的，它的目标是尽可能以最快的速度传递消息，但是这样容易造成consumer来不及处理消息，典型的表现就是拒绝服务以及网络拥塞。而pull方式则可以让consumer根据自己的消费处理能力以适当的速度消费消息。

pull模式不足在于如果Kafka中没有数据，消费者可能会陷入循环之中 (因为消费者类似监听状态获取数据消费的)，一直返回空数据，针对这一点，Kafka的消费者在消费数据时会传入一个时长参数timeout，如果当前没有数据可供消费，consumer会等待一段时间之后再返回，时长为timeout。

## 8.1\. 分区分配策略

一个consumer group中有多个consumer，一个topic有多个partition，所以必然会涉及到partition的分配问题，即确定那个partition由那个consumer消费的问题。

**Kafka的两种分配策略：** 1\. round-robin循环 2\. range

**Round-Robin** 主要采用轮询的方式分配所有的分区，该策略主要实现的步骤： 假设存在三个topic：t0/t1/t2，分别拥有1/2/3个分区，共有6个分区，分别为t0-0/t1-0/t1-1/t2-0/t2-1/t2-2，这里假设我们有三个Consumer，C0、C1、C2，订阅情况为C0：t0，C1：t0、t1，C2：t0/t1/t2。

此时round-robin采取的分配方式，则是按照分区的字典对分区和消费者进行排序，然后对分区进行循环遍历，遇到自己订阅的则消费，否则向下轮询下一个消费者。即按照分区轮询消费者，继而消息被消费。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-21eed325191d7d72c9d4c39455c4cae5_720w.webp)

</figure>

分区在循环遍历消费者，自己被当前消费者订阅，则消息与消费者共同向下（消息被消费），否则消费者向下消息继续遍历（消息没有被消费）。轮询的方式会导致每个Consumer所承载的分区数量不一致，从而导致各个Consumer压力不均。上面的C2因为订阅的比较多，导致承受的压力也相对较大。

**Range** Range的重分配策略，首先计算各个Consumer将会承载的分区数量，然后将指定数量的分区分配给该Consumer。假设存在两个Consumer，C0和C1，两个Topic，t0和t1，这两个Topic分别都有三个分区，那么总共的分区有6个，t0-0，t0-1，t0-2，t1-0，t1-1，t1-2。分配方式如下：

range按照topic一次进行分配，即消费者遍历topic，t0，含有三个分区，同时有两个订阅了该topic的消费者，将这些分区和消费者按照字典序排列。 按照平均分配的方式计算每个Consumer会得到多少个分区，如果没有除尽，多出来的分区则按照字典序挨个分配给消费者。按照此方式以此分配每一个topic给订阅的消费者，最后完成topic分区的分配。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-d642ed5512a4abdca9a8f35f2d27c277_720w.webp)

</figure>

## 8.2\. 消费者offset的存储

由于Consumer在消费过程中可能会出现断电宕机等故障，Consumer恢复以后，需要从故障前的位置继续消费，所以Consumer需要实时记录自己消费到了那个offset，以便故障恢复后继续消费。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-f2a50fd7f054821e36a80b1f6d99ecb0_720w.webp)

</figure>

Kafka0.9版本之前，consumer默认将offset保存在zookeeper中，从0.9版本之后，consumer默认将offset保存在kafka一个内置的topic中，该topic为__consumer_offsets

## 九. 高效读写&Zookeeper作用

## 9.1 高效读写

**顺序写磁盘** Kafka的producer生产数据，需要写入到log文件中，写的过程是追加到文件末端，顺序写的方式，官网有数据表明，同样的磁盘，顺序写能够到600M/s，而随机写只有200K/s，这与磁盘的机械结构有关，顺序写之所以快，是因为其省去了大量磁头寻址的时间。

**零复制技术**

NIC：Network Interface Controller网络接口控制器

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-2807d010381d304949bf8cea16ba1744_720w.webp)

</figure>

这是常规的读取操作： 1\. 操作系统将数据从磁盘文件中读取到内核空间的页面缓存 2\. 应用程序将数据从内核空间读入到用户空间缓冲区 3\. 应用程序将读到的数据写回内核空间并放入到socket缓冲区 4\. 操作系统将数据从socket缓冲区复制到网卡接口，此时数据通过网络发送给消费者

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-390cbabdeaba6f9e79a8ba6f4d08d75f_720w.webp)

</figure>

零拷贝技术只用将磁盘文件的数据复制到页面缓存中一次，然后将数据从页面缓存直接发送到网络中（发送给不同的订阅者时，都可以使用同一个页面缓存），从而避免了重复复制的操作。

如果有10个消费者，传统方式下，数据复制次数为4*10=40次，而使用“零拷贝技术”只需要1+10=11次，一次为从磁盘复制到页面缓存，10次表示10个消费者各自读取一次页面缓存。

## 9.2 zookeeper作用

Kafka集群中有一个broker会被选举为Controller，负责管理集群broker的上下线、所有topic的分区副本分配和leader的选举等工作。Controller的工作管理是依赖于zookeeper的。

**Partition的Leader的选举过程**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-fc64ea72cba32e702b15344767bdace9_720w.webp)

</figure>

## 十. 事务

kafka从0.11版本开始引入了事务支持，事务可以保证Kafka在Exactly Once语义的基础上，生产和消费可以跨分区的会话，要么全部成功，要么全部失败。

## 10.1 Producer事务

为了按跨分区跨会话的事务，需要引入一个全局唯一的Transaction ID，并将Producer获得的PID(可以理解为Producer ID)和Transaction ID进行绑定，这样当Producer重启之后就可以通过正在进行的Transaction ID获得原来的PID。

为了管理Transaction，Kafka引入了一个新的组件Transaction Coordinator，Producer就是通过有和Transaction Coordinator交互获得Transaction ID对应的任务状态，Transaction Coordinator还负责将事务信息写入内部的一个Topic中，这样即使整个服务重启，由于事务状态得到保存，进行中的事务状态可以恢复，从而继续进行。

## 10.2 Consumer事务

对于Consumer而言，事务的保证相比Producer相对较弱，尤其是无法保证Commit的信息被精确消费，这是由于Consumer可以通过offset访问任意信息，而且不同的Segment File声明周期不同，同一事务的消息可能会出现重启后被删除的情况。

## 参考文章

https://blog.csdn.net/cao131502
https://zhuanlan.zhihu.com/p/137811719