## 重要面试知识点

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-75cd70ad3052ba44bf706a3ab39e59d5_720w.webp)


Kafka 消费端确保一个 Partition 在一个消费者组内只能被一个消费者消费。这句话改怎么理解呢？

1.  在同一个消费者组内，一个 Partition 只能被一个消费者消费。
2.  在同一个消费者组内，所有消费者组合起来必定可以消费一个 Topic 下的所有 Partition。
3.  在同一个消费组内，一个消费者可以消费多个 Partition 的信息。
4.  在不同消费者组内，同一个分区可以被多个消费者消费。
5.  每个消费者组一定会完整消费一个 Topic 下的所有 Partition。

### **消费组存在的意义**

了解了消费者与消费组的关系后，有朋友会比较疑惑消费者组有啥实际存在的意义呢？或者说消费组的作用是什么？

作者对消费组的作用归结了如下两点。

1.  在实际生产中，对于同一个 Topic，可能有 A、B、C 等 N 个消费方想要消费。比如一份用户点击日志，A 消费方想用来做一个用户近 N 天点击过哪些商品；B 消费方想用来做一个用户近 N 天点击过前 TopN 个相似的商品；C 消费方想用来做一个根据用户点击过的商品推荐相关周边的商品需求。对于多应用场景，就可以使用消费组来隔离不同的业务使用场景，从而达到一个 Topic 可以被多个消费组重复消费的目的。
2.  消费组与 Partition 的消费进度绑定。当有新的消费者加入或者有消费者从消费组退出时，会触发消费组的 Repartition 操作（后面会详细介绍 Repartition）；在 Repartition 前，Partition1 被消费组的消费者 A 进行消费，Repartition 后，Partition1 消费组的消费者 B 进行消费，为了避免消息被重复消费，需要从消费组记录的 Partition 消费进度读取当前消费到的位置（即 OffSet 位置），然后在继续消费，从而达到消费者的平滑迁移，同时也提高了系统的可用性。

## **Repartition 触发时机**

使用过 Kafka 消费者客户端的同学肯定知道，消费者组内偶尔会触发 Repartition 操作，所谓 Repartition 即 Partition 在某些情况下重新被分配给参与消费的消费者。基本可以分为如下几种情况。

1.  消费组内某消费者宕机，触发 Repartition 操作，如下图所示。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-a9ef6a29cb9ba3456a05ad75cb91cb03_720w.webp)

<figcaption>消费者宕机情况</figcaption>

</figure>

2\. 消费组内新增消费者，触发 Repartition 操作，如下图所示。一般这种情况是为了提高消费端的消费能力，从而加快消费进度。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-8803223d712fdde035b8e7b9170dd3fb_720w.webp)

<figcaption>新增消费者情况</figcaption>

</figure>

3.Topic 下的 Partition 增多，触发 Repartition 操作，如下图所示。一般这种调整 Partition 个数的情况也是为了提高消费端消费速度的，因为当消费者个数大于等于 Partition 个数时，在增加消费者个数是没有用的（原因是：在一个消费组内，消费者:Partition = 1:N，当 N 小于 1 时，相当于消费者过剩了），所以一方面增加 Partition 个数同时增加消费者个数可以提高消费端的消费速度。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-8f1a427c6842d9babf139454ce23cfa3_720w.webp)

<figcaption>新增Partition个数情况</figcaption>

</figure>

## **消费者与 ZK 的关系**

众所周知，ZK 不仅保存了消费者消费 partition 的进度，同时也保存了消费组的成员列表、partition 的所有者。消费者想要消费 Partition，需要从 ZK 中获取该消费者对应的分区信息及当前分区对应的消费进度，即 OffSert 信息。那么 Partition 应该由那个消费者进行消费，决定因素有哪些呢？从之前的图中不难得出，两个重要因素分别是：消费组中存活的消费者列表和 Topic 对应的 Partition 列表。通过这两个因素结合 Partition 分配算法，即可得出消费者与 Partition 的对应关系，然后将信息存储到 ZK 中。Kafka 有高级 API 和低级 API，如果不需要操作 OffSet 偏移量的提交，可通过高级 API 直接使用，从而降低使用者的难度。对于一些比较特殊的使用场景，比如想要消费特定 Partition 的信息，Kafka 也提供了低级 API 可进行手动操作。

## **消费端工作流程**

在介绍消费端工作流程前，先来熟悉一下用到的一些组件。

*   `KakfaConsumer`：消费端，用于启动消费者进程来消费消息。
*   `ConsumerConfig`：消费端配置管理，用于给消费端配置相关参数，比如指定 Kafka 集群，设置自动提交和自动提交时间间隔等等参数，都由其来管理。
*   `ConsumerConnector`：消费者连接器，通过消费者连接器可以获得 Kafka 消息流，然后通过消息流就能获得消息从而使得客户端开始消费消息。

以上三者之间的关系可以概括为：消费端使用消费者配置管理创建出了消费者连接器，通过消费者连接器创建队列（这个队列的作用也是为了缓存数据），其中队列中的消息由专门的拉取线程从服务端拉取然后写入，最后由消费者客户端轮询队列中的消息进行消费。具体操作流程如下图所示。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-122b4a706de39655d257928005a83ff1_720w.webp)

<figcaption>消费端工作流程</figcaption>

</figure>

我们在从消费者与 ZK 的角度来看看其工作流程是什么样的？

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-4ed25ebb9236986b2084ce8a042f65b9_720w.webp)

<figcaption>消费端与ZK之间的工作流程</figcaption>

</figure>

从上图可以看出，首先拉取线程每拉取一次消息，同步更新一次拉取状态，其作用是为了下一次拉取消息时能够拉取到最新产生的消息；拉取线程将拉取到的消息写入到队列中等待消费消费线程去真正读取处理。消费线程以轮询的方式持续读取队列中的消息，只要发现队列中有消息就开始消费，消费完消息后更新消费进度，此处需要注意的是，消费线程不是每次都和 ZK 同步消费进度，而是将消费进度暂时写入本地。这样做的目的是为了减少消费者与 ZK 的频繁同步消息，从而降低 ZK 的压力。

## **消费者的三种消费情况**

消费者从服务端的 Partition 上拉取到消息，消费消息有三种情况，分别如下：

1.  至少一次。即一条消息至少被消费一次，消息不可能丢失，但是可能会被重复消费。
2.  至多一次。即一条消息最多可以被消费一次，消息不可能被重复消费，但是消息有可能丢失。
3.  正好一次。即一条消息正好被消费一次，消息不可能丢失也不可能被重复消费。

### **1.至少一次**

消费者读取消息，先处理消息，在保存消费进度。消费者拉取到消息，先消费消息，然后在保存偏移量，当消费者消费消息后还没来得及保存偏移量，则会造成消息被重复消费。如下图所示：

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-1a047ed616ba44daebdb4b6ce786a61a_720w.webp)

<figcaption>先消费后保存消费进度</figcaption>

</figure>

### **2.至多一次**

消费者读取消息，先保存消费进度，在处理消息。消费者拉取到消息，先保存了偏移量，当保存了偏移量后还没消费完消息，消费者挂了，则会造成未消费的消息丢失。如下图所示：

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-1f9f91ae54396c5e5d93ae89251eb1ed_720w.webp)

<figcaption>先保存消费进度后消费消息</figcaption>

</figure>

### **3.正好一次**

正好消费一次的办法可以通过将消费者的消费进度和消息处理结果保存在一起。只要能保证两个操作是一个原子操作，就能达到正好消费一次的目的。通常可以将两个操作保存在一起，比如 HDFS 中。正好消费一次流程如下图所示。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-a0bbb114e2ad551227f81c1f26d4bd5d_720w.webp)

<figcaption>正好消费一次</figcaption>

</figure>

## Partition、Replica、Log 和 LogSegment 的关系

假设有一个 Kafka 集群，Broker 个数为 3，Topic 个数为 1，Partition 个数为 3，Replica 个数为 2。Partition 的物理分布如下图所示。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-f8f21631b138321f25c8821c677c5579_720w.webp)

<figcaption>Partition分布图</figcaption>

</figure>

从上图可以看出，该 Topic 由三个 Partition 构成，并且每个 Partition 由主从两个副本构成。每个 Partition 的主从副本分布在不同的 Broker 上，通过这点也可以看出，当某个 Broker 宕机时，可以将分布在其他 Broker 上的从副本设置为主副本，因为只有主副本对外提供读写请求，当然在最新的 2.x 版本中从副本也可以对外读请求了。将主从副本分布在不同的 Broker 上从而提高系统的可用性。

Partition 的实际物理存储是以 Log 文件的形式展示的，而每个 Log 文件又以多个 LogSegment 组成。Kafka 为什么要这么设计呢？其实原因比较简单，随着消息的不断写入，Log 文件肯定是越来越大，Kafka 为了方便管理，将一个大文件切割成一个一个的 LogSegment 来进行管理；每个 LogSegment 由数据文件和索引文件构成，数据文件是用来存储实际的消息内容，而索引文件是为了加快消息内容的读取。

可能又有朋友会问，Kafka 本身消费是以 Partition 维度顺序消费消息的，磁盘在顺序读的时候效率很高完全没有必要使用索引啊。其实 Kafka 为了满足一些特殊业务需求，比如要随机消费 Partition 中的消息，此时可以先通过索引文件快速定位到消息的实际存储位置，然后进行处理。

总结一下 Partition、Replica、Log 和 LogSegment 之间的关系。消息是以 Partition 维度进行管理的，为了提高系统的可用性，每个 Partition 都可以设置相应的 Replica 副本数，一般在创建 Topic 的时候同时指定 Replica 的个数；Partition 和 Replica 的实际物理存储形式是通过 Log 文件展现的，为了防止消息不断写入，导致 Log 文件大小持续增长，所以将 Log 切割成一个一个的 LogSegment 文件。

**注意：** 在同一时刻，每个主 Partition 中有且只有一个 LogSegment 被标识为可写入状态，当一个 LogSegment 文件大小超过一定大小后（比如当文件大小超过 1G，这个就类似于 HDFS 存储的数据文件，HDFS 中数据文件达到 128M 的时候就会被分出一个新的文件来存储数据），就会新创建一个 LogSegment 来继续接收新写入的消息。

## 写入消息流程分析

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-eb66e4ecf7cf07fcb6b12029bfdd9b71_720w.webp)

<figcaption>消息写入及落盘流程</figcaption>

</figure>

流程解析

在第 3 篇文章讲过，生产者客户端对于每个 Partition 一次会发送一批消息到服务端，服务端收到一批消息后写入相应的 Partition 上。上图流程主要分为如下几步：

1.  **客户端消息收集器收集属于同一个分区的消息，并对每条消息设置一个偏移量，且每一批消息总是从 0 开始单调递增。比如第一次发送 3 条消息，则对三条消息依次编号 [0,1,2]，第二次发送 4 条消息，则消息依次编号为 [0,1,2,3]。注意此处设置的消息偏移量是相对偏移量。**
2.  **客户端将消息发送给服务端，服务端拿到下一条消息的绝对偏移量，将传到服务端的这批消息的相对偏移量修改成绝对偏移量。**
3.  **将修改后的消息以追加的方式追加到当前活跃的 LogSegment 后面，然后更新绝对偏移量。**
4.  **将消息集写入到文件通道。**
5.  **文件通道将消息集 flush 到磁盘，完成消息的写入操作。**

了解以上过程后，我们在来看看消息的具体构成情况。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-6e993c95decd5d274b032cd423936504_720w.webp)

<figcaption>消息构成细节图</figcaption>

</figure>

一条消息由如下三部分构成：

*   **OffSet：偏移量，消息在客户端发送前将相对偏移量存储到该位置，当消息存储到 LogSegment 前，先将其修改为绝对偏移量在写入磁盘。**
*   **Size：本条 Message 的内容大小**
*   **Message：消息的具体内容，其具体又由 7 部分组成，crc 用于校验消息，Attribute 代表了属性，key-length 和 value-length 分别代表 key 和 value 的长度，key 和 value 分别代表了其对应的内容。**

### 消息偏移量的计算过程

通过以上流程可以看出，每条消息在被实际存储到磁盘时都会被分配一个绝对偏移量后才能被写入磁盘。在同一个分区内，消息的绝对偏移量都是从 0 开始，且单调递增；在不同分区内，消息的绝对偏移量是没有任何关系的。接下来讨论下消息的绝对偏移量的计算规则。

确定消息偏移量有两种方式，一种是顺序读取每一条消息来确定，此种方式代价比较大，实际上我们并不想知道消息的内容，而只是想知道消息的偏移量；第二种是读取每条消息的 Size 属性，然后计算出下一条消息的起始偏移量。比如第一条消息内容为 “abc”，写入磁盘后的偏移量为：8（OffSet）+ 4（Message 大小）+ 3（Message 内容的长度）= 15。第二条写入的消息内容为“defg”，其起始偏移量为 15，下一条消息的起始偏移量应该是：15+8+4+4=31，以此类推。

## 消费消息及副本同步流程分析

和写入消息流程不同，读取消息流程分为两种情况，分别是消费端消费消息和从副本（备份副本）同步主副本的消息。在开始分析读取流程之前，需要先明白几个用到的变量，不然流程分析可能会看的比较糊涂。

*   **BaseOffSet**：基准偏移量，每个 Partition 由 N 个 LogSegment 组成，每个 LogSegment 都有基准偏移量，大概由如下构成，数组中每个数代表一个 LogSegment 的基准偏移量：[0,200,400,600, ...]。
*   **StartOffSet**：起始偏移量，由消费端发起读取消息请求时，指定从哪个位置开始消费消息。
*   **MaxLength**：拉取大小，由消费端发起读取消息请求时，指定本次最大拉取消息内容的数据大小。该参数可以通过[max.partition.fetch.bytes](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/draft/3020%23)来指定，默认大小为 1M。
*   **MaxOffSet**：最大偏移量，消费端拉取消息时，最高可拉取消息的位置，即俗称的“高水位”。该参数由服务端指定，其作用是为了防止生产端还未写入的消息就被消费端进行消费。此参数对于从副本同步主副本不会用到。
*   **MaxPosition**：LogSegment 的最大位置，确定了起始偏移量在某个 LogSegment 上开始，读取 MaxLength 后，不能超过 MaxPosition。MaxPosition 是一个实际的物理位置，而非偏移量。

假设消费端从 000000621 位置开始消费消息，关于几个变量的关系如下图所示。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-cd9c62a71cddccd7bc8a5d810d5af216_720w.webp)

<figcaption>位置关系图</figcaption>

</figure>

消费端和从副本拉取流程如下：

1.  **客户端确定拉取的位置，即 StartOffSet 的值，找到主副本对应的 LogSegment。**
2.  **LogSegment 由索引文件和数据文件构成，由于索引文件是从小到大排列的，首先从索引文件确定一个小于等于 StartOffSet 最近的索引位置。**
3.  **根据索引位置找到对应的数据文件位置，由于数据文件也是从小到大排列的，从找到的数据文件位置顺序向后遍历，直到找到和 StartOffSet 相等的位置，即为消费或拉取消息的位置。**
4.  **从 StartOffSet 开始向后拉取 MaxLength 大小的数据，返回给消费端或者从副本进行消费或备份操作。**

假设拉取消息起始位置为 00000313，消息拉取流程图如下：

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-9417ca60a0c5e9474ec49a77fff18b1b_720w.webp)

<figcaption>消息拉取流程图</figcaption>

</figure>

## kafka 如何保证系统的高可用、数据的可靠性和数据的一致性的？

### kafka 的高可用性：

1.  **Kafka 本身是一个分布式系统，同时采用了 Zookeeper 存储元数据信息，提高了系统的高可用性。**
2.  **Kafka 使用多副本机制，当状态为 Leader 的 Partition 对应的 Broker 宕机或者网络异常时，Kafka 会通过选举机制从对应的 Replica 列表中重新选举出一个 Replica 当做 Leader，从而继续对外提供读写服务（当然，需要注意的一点是，在新版本的 Kafka 中，Replica 也可以对外提供读请求了），利用多副本机制在一定程度上提高了系统的容错性，从而提升了系统的高可用。**

### Kafka 的可靠性：

1.  **从 Producer 端来看，可靠性是指生产的消息能够正常的被存储到 Partition 上且消息不会丢失。Kafka 通过 [request.required.acks](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23)和[min.insync.replicas](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23) 两个参数配合，在一定程度上保证消息不会丢失。**
2.  **[request.required.acks](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23) 可设置为 1、0、-1 三种情况。**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-7946f258c85fb8ca3d4aa423269c483a_720w.webp)

<figcaption>request.required.acks=1</figcaption>

</figure>

设置为 1 时代表当 Leader 状态的 Partition 接收到消息并持久化时就认为消息发送成功，如果 ISR 列表的 Replica 还没来得及同步消息，Leader 状态的 Partition 对应的 Broker 宕机，则消息有可能丢失。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-382c9f37f644feb37dd975c67bc1038f_720w.webp)

<figcaption>request.required.acks=0</figcaption>

</figure>

设置为 0 时代表 Producer 发送消息后就认为成功，消息有可能丢失。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-592996f264baadc64967d6f4b28f4d23_720w.webp)

<figcaption>request.required.acks=-1</figcaption>

</figure>

设置为-1 时，代表 ISR 列表中的所有 Replica 将消息同步完成后才认为消息发送成功；但是如果只存在主 Partition 的时候，Broker 异常时同样会导致消息丢失。所以此时就需要[min.insync.replicas](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23)参数的配合，该参数需要设定值大于等于 2，当 Partition 的个数小于设定的值时，Producer 发送消息会直接报错。

上面这个过程看似已经很完美了，但是假设如果消息在同步到部分从 Partition 上时，主 Partition 宕机，此时消息会重传，虽然消息不会丢失，但是会造成同一条消息会存储多次。在新版本中 Kafka 提出了幂等性的概念，通过给每条消息设置一个唯一 ID，并且该 ID 可以唯一映射到 Partition 的一个固定位置，从而避免消息重复存储的问题（作者到目前还没有使用过该特性，感兴趣的朋友可以自行在深入研究一下）。

### Kafka 的一致性：

1.  **从 Consumer 端来看，同一条消息在多个 Partition 上读取到的消息是一直的，Kafka 通过引入 HW（High Water）来实现这一特性。**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-9975539d98bf1a4e1a3038f2eceb2bb9_720w.webp)

<figcaption>消息同步图</figcaption>

</figure>

从上图可以看出，假设 Consumer 从主 Partition1 上消费消息，由于 Kafka 规定只允许消费 HW 之前的消息，所以最多消费到 Message2。假设当 Partition1 异常后，Partition2 被选举为 Leader，此时依旧可以从 Partition2 上读取到 Message2。其实 HW 的意思利用了木桶效应，始终保持最短板的那个位置。

从上面我们也可以看出，使用 HW 特性后会使得消息只有被所有副本同步后才能被消费，所以在一定程度上降低了消费端的性能，可以通过设置[replica.lag.time.max.ms](https://link.zhihu.com/?target=https%3A//xie.infoq.cn/edit/49a133ad2b2f2671aa60706b0%23)参数来保证消息同步的最大时间。

## kafka 为什么那么快？

kafka 使用了顺序写入和“零拷贝”技术，来达到每秒钟 200w（Apache 官方给出的数据） 的磁盘数据写入量，另外 Kafka 通过压缩数据，降低 I/O 的负担。

1.  **顺序写入**

大家都知道，对于磁盘而已，如果是随机写入数据的话，每次数据在写入时要先进行寻址操作，该操作是通过移动磁头完成的，极其耗费时间，而顺序读写就能够避免该操作。

1.  **“零拷贝”技术**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-6930901956f341f1ab4a6e5650a0680b_720w.webp)

<figcaption>普通数据拷贝流程图</figcaption>

</figure>

普通的数据拷贝流程如上图所示，数据由磁盘 copy 到内核态，然后在拷贝到用户态，然后再由用户态拷贝到 socket，然后由 socket 协议引擎，最后由协议引擎将数据发送到网络中。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-9e44873a63d8addca917e658667f0b61_720w.webp)

<figcaption>&amp;quot;零拷贝&amp;quot;流程图</figcaption>

</figure>

采用了“零拷贝”技术后可以看出，数据不在经过用户态传输，而是直接在内核态完成操作，减少了两次 copy 操作。从而大大提高了数据传输速度。

1.  **压缩**

Kafka 官方提供了多种压缩协议，包括 gzip、snappy、lz4 等等，从而降低了数据传输的成本。

## Kafka 中的消息是否会丢失和重复消费？

1.  **Kafka 是否会丢消息，答案相信仔细看过前面两个问题的同学都比较清楚了，这里就不在赘述了。**
2.  **在低版本中，比如作者公司在使用的 Kafka0.8 版本中，还没有幂等性的特性的时候，消息有可能会重复被存储到 Kafka 上（原因见上一个问题的），在这种情况下消息肯定是会被重复消费的。**

**这里给大家一个解决重复消费的思路，作者公司使用了 Redis 记录了被消费的 key，并设置了过期时间，在 key 还没有过期内，对于同一个 key 的消息全部当做重复消息直接抛弃掉。** 在网上看到过另外一种解决方案，使用 HDFS 存储被消费过的消息，是否具有可行性存疑（需要读者朋友自行探索），读者朋友们可以根据自己的实际情况选择相应的策略，如果朋友们还有其他比较好的方案，欢迎留言交流。

## 为什么要使用 kafka，为什么要使用消息队列？

### 先来说说为什么要使用消息队列？

这道题比较主观一些（自认为没有网上其他文章写得话，轻喷），但是都相信大家使用消息队列无非就是为了 **解耦**、**异步**、**消峰**。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-f7c1bb87ab46ddd03255c58109ce360f_720w.webp)

<figcaption>系统调用图</figcaption>

</figure>

随着业务的发展，相信有不少朋友公司遇到过如上图所示的情况，系统 A 处理的结构被 B、C、D 系统所依赖，当新增系统 E 时，也需要系统 A 配合进行联调和上线等操作；还有当系统 A 发生变更时同样需要告知 B、C、D、E 系统需要同步升级改造。

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-0f0c8f9531a38f6d79b2cbb2973bfbfc_720w.webp)

<figcaption>引入消息队列图</figcaption>

</figure>

引入消息队列后有两个好处：

1.  **各个系统进行了解耦，从上图也可以看出，当系统 A 突然发生热点事件时，同一时间产生大量结果，MQ 充当了消息暂存的效果，防止 B、C、D、E 系统也跟着崩溃。**
2.  **当新系统 E 需要接入系统 A 的数据，只需要和 MQ 对接就可以了，从而避免了与系统 A 的调试上线等操作。**

引入消息队列的坏处：

万事皆具备两面性，看似引入消息队列这件事情很美好，但是同时也增加了系统的复杂度、系统的维护成本提高（如果 MQ 挂了怎么办）、引入了一致性等等问题需要去解决。

## 为什么要使用 Kafka?

作者认为采用 Kafka 的原因有如下几点：

1.  **Kafka 目前在业界被广泛使用，社区活跃度高，版本更新迭代速度也快。**
2.  **Kafka 的生产者和消费者都用 Java 语言进行了重写，在一定程度降低了系统的维护成本（作者的主观意见，因为当下 Java 的使用群体相当庞大）。**
3.  **Kafka 系统的吞吐量高，达到了每秒 10w 级别的处理速度。**
4.  **Kafka 可以和很多当下优秀的大数据组件进行集成，包括 Spark、Flink、Flume、Storm 等等。**

## 为什么 Kafka 不支持读写分离？

这个问题有个先决条件，我们只讨论 Kafka0.9 版本的情况。对于高版本，从 Partition 也可以承担读请求了，这里不多赘述。

Kafka 如果支持读写分离的话，有如下几个问题。

1.  **系统设计的复杂度会比较大，当然这个比较牵强，毕竟高版本的 Kafka 已经实现了。**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-98093ad82970feb7a0c52954c6942aa1_720w.webp)

</figure>

**2\. 从上图可以看出，从从 Partition 上读取数据会有两个问题。一、数据从主 Partition 上同步到从 Partition 有数据延迟问题，因为数据从生产到消费会经历 3 次网络传输才能够被消费，对于时效性要求比较高的场景本身就不适合了。二、数据一致性问题，假设主 Partition 将数据第一次修改成了 A，然后又将该数据修改成了 B，由于从主 Partition 同步到从 Partition 会有延迟问题，所以也就会产生数据一致性问题。**

分析得出，通过解决以上问题来换取从 Partition 承担读请求，成本可想而知，而且对于写入压力大，读取压力小的场景，本身也就没有什么意义了。

## 总结

本文介绍了几个常见的 Kafka 的面试题

### 常见面试题一览

#### 1.1 Kafka 中的 ISR(InSyncRepli)、 OSR(OutSyncRepli)、 AR(AllRepli)代表什么？

ISR：速率和leader相差低于10s的follower的集合

OSR：速率和leader相差大于10s的follwer

AR：所有分区的follower

#### 1.2 Kafka 中的 HW、 LEO 等分别代表什么？

HW：High Water高水位，根据同一分区中最低的LEO决定（Log End Offset）

LEO：每个分区最大的Offset

#### 1.3 Kafka 中是怎么体现消息顺序性的？

在每个分区内，每条消息都有offset，所以消息在同一分区内有序，无法做到全局有序性

#### 1.4 Kafka 中的分区器、序列化器、拦截器是否了解？它们之间的处理顺序是什么？

分区器Partitioner用来对分区进行处理的，即消息发送到哪一个分区的问题。序列化器，这个是对数据进行序列化和反序列化的工具。拦截器，即对于消息发送进行一个提前处理和收尾处理的类Interceptor，处理顺利首先通过拦截器=>序列化器=>分区器

#### 1.5 Kafka 生产者客户端的整体结构是什么样子的？使用了几个线程来处理？分别是什么？

使用两个线程：main和sender 线程，main线程会一次经过拦截器、序列化器、分区器将数据发送到RecoreAccumulator线程共享变量，再由sender线程从共享变量中拉取数据发送到kafka broker

batch.size达到此规模消息才发送，linger.ms未达到规模，等待当前时长就发送数据。

#### 1.6 消费组中的消费者个数如果超过 topic 的分区，那么就会有消费者消费不到数据”这句 话是否正确？

这句话是对的，超过分区个数的消费者不会在接收数据，主要原因是一个分区的消息只能够被一个消费者组中的一个消费者消费。

#### 1.7 消费者提交消费位移时提交的是当前消费到的最新消息的 offset 还是 offset+1？

生产者发送数据的offset是从0开始的，消费者消费的数据的offset是从1开始，故最新消息是offset+1

#### 1.8 有哪些情形会造成重复消费？

先消费后提交offset，如果消费完宕机了，则会造成重复消费

#### 1.9 那些情景会造成消息漏消费？

先提交offset，还没消费就宕机了，则会造成漏消费

#### 1.10 当你使用 kafka-topics.sh 创建（删除）了一个 topic 之后， Kafka 背后会执行什么逻辑？

会在 zookeeper 中的/brokers/topics 节点下创建一个新的 topic 节点，如：/brokers/topics/first 触发 Controller 的监听程序 kafka Controller 负责 topic 的创建工作，并更新 metadata cache

#### 1.11 topic 的分区数可不可以增加？如果可以怎么增加？如果不可以，那又是为什么？

可以增加，修改分区个数--alter可以修改分区个数

#### 1.12 topic 的分区数可不可以减少？如果可以怎么减少？如果不可以，那又是为什么？

不可以减少，减少了分区之后，之前的分区中的数据不好处理

#### 1.13 Kafka 有内部的 topic 吗？如果有是什么？有什么所用？

有，__consumer_offsets主要用来在0.9版本以后保存消费者消费的offset

#### 1.14 Kafka 分区分配的概念？

Kafka分区对于Kafka集群来说，分区可以做到负载均衡，对于消费者来说分区可以提高并发度，提高读取效率

#### 1.15 简述 Kafka 的日志目录结构？

每一个分区对应着一个文件夹，命名为topic-0/topic-1…，每个文件夹内有.index和.log文件。

#### 1.16 如果我指定了一个 offset， Kafka Controller 怎么查找到对应的消息？

offset表示当前消息的编号，首先可以通过二分法定位当前消息属于哪个.index文件中，随后采用seek定位的方法查找到当前offset在.index中的位置，此时可以拿到初始的偏移量。通过初始的偏移量再通过seek定位到.log中的消息即可找到。

#### 1.17 聊一聊 Kafka Controller 的作用？

Kafka集群中有一个broker会被选举为Controller，负责管理集群broker的上下线、所有topic的分区副本分配和leader的选举等工作。Controller的工作管理是依赖于zookeeper的。

#### 1.18 Kafka 中有那些地方需要选举？这些地方的选举策略又有哪些？

在ISR中需要选举出Leader，选择策略为先到先得。在分区中需要选举，需要选举出Leader和follower。

#### 1.19 失效副本是指什么？有那些应对措施？

失效副本为速率比leader相差大于10s的follower，ISR会将这些失效的follower踢出，等速率接近leader的10s内，会重新加入ISR

#### 1.20 Kafka 的哪些设计让它有如此高的性能？

1.  Kafka天生的分布式架构
2.  对log文件进行了分segment，并对segment建立了索引
3.  对于单节点使用了顺序读写，顺序读写是指的文件的顺序追加，减少了磁盘寻址的开销，相比随机写速度提升很多
4.  使用了零拷贝技术，不需要切换到用户态，在内核态即可完成读写操作，且数据的拷贝次数也更少。

## 参考文章
https://blog.csdn.net/cao131502
https://zhuanlan.zhihu.com/p/137811719