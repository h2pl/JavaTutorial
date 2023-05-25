## **启动zookeeper**

bin/zookeeper-server-start.sh config/zookeeper.properties &

## **启动kafka：**

bin/kafka-server-start.sh config/server.properties

这样启动又一个坏处，就是kafka启动完毕之后，不能关闭终端，为此，我们可以运行这条命令：

nohup bin/kafka-server-start.sh config/server.properties > ./dev/null 2>&1 &

![](https://img2022.cnblogs.com/blog/796632/202208/796632-20220812161146385-332776455.png)

多个kafka的话，在各个虚拟机上运行kafka启动命令多次即可。

当然这个是单机的命令，集群的命令后面再讲。

## **查看是否启动**

jps -lm

![](https://img2022.cnblogs.com/blog/796632/202208/796632-20220812161210221-836644701.png)

说明没有启动kfka

![](https://img2022.cnblogs.com/blog/796632/202208/796632-20220812161224734-562363764.png)

说明启动kafka了

## 查看kafka版本

find ./libs/ -name \*kafka_\* | head -1 | grep -o '\kafka[^\n]*'

kafka_2.12-2.4.1.jar

结果:

就可以看到kafka的具体版本了。

其中，2.12为scala版本，2.4.1为kafka版本。

## **停止kafka**

bin/kafka-server-stop.sh

## **停止zookeeper**

bin/zookeeper-server-stop.sh

## **创建topic**

bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partitions 1 --topic test

多集群创建，执行这个需要搭建多机器的kafka集群环境，zkq1/zkq2/zkq3分别代表了3台zookeeper集群的三台机器

/bin/kafka-topics.sh —create —zookeeper zkq1:2181,zkq2:2181,zkq3:2181 -replication-factor 6 —partition 6 —topic test

解释：

--topic后面的test0是topic的名称

--zookeeper应该和server.properties文件中的zookeeper.connect一样

--config指定当前topic上有效的参数值

--partitions指定topic的partition数量，如果不指定该数量，默认是server.properties文件中的num.partitions配置值

--replication-factor指定每个partition的副本个数，默认是1个

也可以向没有的topic发送消息的时候创建topic

需要

开启自动创建配置：auto.create.topics.enable=true

使用程序直接往kafka中相应的topic发送数据，如果topic不存在就会按默认配置进行创建。

## **展示topic**

bin/kafka-topics.sh --list --zookeeper localhost:2181

## **描述topic**

bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic

![](https://img2022.cnblogs.com/blog/796632/202208/796632-20220812161250801-1389051022.png)

解释：

要查看多个topic用逗号分割

**leader**:

是该partitons所在的所有broker中担任leader的broker id，每个broker都有可能成为leader，负责处理消息的读和写，leader是从所有节点中随机选择的.

-1表示此broker移除了

**Replicas**:

显示该partiton所有副本所在的broker列表，包括leader，不管该broker是否是存活，不管是否和leader保持了同步。列出了所有的副本节点，不管节点是否在服务中.

**Isr**:

in-sync replicas的简写，表示存活且副本都已同步的的broker集合，是replicas的子集，是正在服务中的节点.

举例：

比如上面结果的第一行：Topic: test0 .Partition:0 ...Leader: 0 ......Replicas: 0,2,1 Isr: 1,0,2

Partition: 0[该partition编号是0]

Replicas: 0,2,1[代表partition0 在broker0，broker1，broker2上保存了副本]

Isr: 1,0,2 [代表broker0，broker1，broker2都存活而且目前都和leader保持同步]

Leader: 0

代表保存在broker0，broker1，broker2上的这三个副本中，leader是broker0

leader负责读写，broker1、broker2负责从broker0同步信息，平时没他俩什么事

## **查看topic的partition及增加partition**

/kafka-topics.sh –zookeeper 10.2.1.1:2181 –topic mcc-logs –describe.

## **删除Topic**

/bin/kafka-topics.sh --zookeeper localhost:2181 --delete --topic test

如果你的server.properties内没有配置相关的配置的话，会出现如下错误：

Topic test is marked for deletion.

Note: This will have no impact if delete.topic.enable is not set to true.

这边是说，你的Topic已经被标记为待删除的Topic，但是呢，你配置文件的开关没有打开，所以只是给它添加了一个标记，实际上，这个Topic并没有被删除。只有，你打开开关之后，会自动删除被标记删除的Topic。

解决办法：

设置server.properties文件内的“delete.topic.enable=true”，并且重启Kafka就可以了。

如果不想修改配置也可以完全删除

1、删除kafka存储目录（server.propertiewenjian log.dirs配置，默认为“/tmp/kafka-logs”）下对应的topic。(不同broker下存储的topic不一定相同，所有broker都要看一下)

2、进入zookeeper客户端删掉对应topic

zkCli.sh .-server 127.0.0.1:42182

找到topic目录:

ls ../brokers/topics

删掉对应topic

rmr ./brokers/topic/topic-name

找到目录:

ls .../config/topics

删掉对应topic

rmr ./config/topics/topic-name .

这样就完全删除了

## **删除topic中存储的内容**

在config/server.properties中找到如下的位置

![](https://img2022.cnblogs.com/blog/796632/202208/796632-20220812161312458-550425542.png)

删除log.dirs指定的文件目录，

登录zookeeper client。

命令：

/home/ZooKeeper/bin/zkCli.sh

删除zookeeper中该topic相关的目录

命令：

rm -r /kafka/config/topics/test0

rm -r /kafka/brokers/topics/test0

rm -r /kafka/admin/delete_topics/test0 （topic被标记为marked for deletion时需要这个命令）

重启zookeeper和broker

## **生产者发送消息：**

bin/kafka-console-producer.sh --broker-list 130.51.23.95:9092 --topic my-replicated-topic

这里的ip和端口是broker的ip及端口，根据自己kafka机器的ip和端口写就可以

## **消费者消费消息：**

bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --new-consumer --from-beginning --consumer.config config/consumer.properties

## **查看topic某分区偏移量最大（小）值**

bin/kafka-run-class.sh kafka.tools.GetOffsetShell --topic hive-mdatabase-hostsltable .--time -1 --broker-list node86:9092 --partitions 0

注： time为-1时表示最大值，time为-2时表示最小值

不指定--partitions 就是指这个topic整体的情况

## 查看指定group的消费情况

kafka-consumer-groups.sh --bootstrap-server 172.20.72.93:9092 --describe --group mygroup

运行结果：

![](https://img2022.cnblogs.com/blog/796632/202208/796632-20220816164455794-344440282.png)

*   GROUP:消费者组
*   TOPIC：topic名字
*   PARTITION ：partition id
*   CURRENT-OFFSET： .当前消费到的offset . . . . . . . .
*   LOG-END-OFFSETSIZE ：最新的offset
*   LAG:未消费的条数
*   CONSUMER-ID:消费者组中消费者的id,为—代表没有active的消费者
*   HOST：消费者的机器ip，为—代表没有active的消费者
*   CLIENT-ID:消费者clientID，为—代表没有active的消费者

## .查看所有group的消费情况

kafka-consumer-groups.sh --bootstrap-server 172.20.72.93:9092 --all-groups --all-topics --describe

![](https://img2022.cnblogs.com/blog/796632/202208/796632-20220816172442100-1560497638.png)

## 修改group消费的offset

kafka-consumer-groups.sh --bootstrap-server 172.20.72.93:9092 --group mygroup --reset-offsets --topic mytopic --to-offset 61 --execute

上面就是把mygroup在mytopic的消费offset修改到了61

重设位移有几种选项:

--to-earliest：. .设置到最早位移处，也就是0

--to-latest：. . .设置到最新处，也就是主题分区HW的位置

--to-offset NUM： 指定具体的位移位置

--shift-by NUM：. 基于当前位移向前回退多少

--by-duration：. .回退到多长时间

## 查看指定group中活跃的消费者

kafka-consumer-groups.sh --bootstrap-server 172.20.72.93:9092 --describe --group mygroup --members

## **增加topic分区数**

（只能增加不能减少）

为topic t_cdr 增加10个分区

bin/kafka-topics.sh --zookeeper node01:2181 .--alter --topic t_cdr --partitions 10

## **常用配置及说明**

kafka 常见重要配置说明，分为四部分

*   Broker Config：kafka 服务端的配置
*   Producer Config：生产端的配置
*   Consumer Config：消费端的配置
*   Kafka Connect Config：kafka 连接相关的配置

### **Broker Config**

1.  **zookeeper.connect**

连接 zookeeper 集群的地址，用于将 kafka 集群相关的元数据信息存储到指定的 zookeeper 集群中

**2\. advertised.port**

注册到 zookeeper 中的地址端口信息，在 IaaS 环境中，默认注册到 zookeeper 中的是内网地址，通过该配置指定外网访问的地址及端口，advertised.host.name 和 advertised.port 作用和 advertised.port 差不多，在 0.10.x 之后，直接配置 advertised.port 即可，前两个参数被废弃掉了。

**3\. auto.create.topics.enable**

自动创建 topic，默认为 true。其作用是当向一个还没有创建的 topic 发送消息时，此时会自动创建一个 topic，并同时设置 -num.partition 1 (partition 个数) 和 default.replication.factor (副本个数，默认为 1) 参数。

一般该参数需要手动关闭，因为自动创建会影响 topic 的管理，我们可以通过 kafka-topic.sh 脚本手动创建 topic，通常也是建议使用这种方式创建 topic。在 0.10.x 之后提供了 kafka-admin 包，可以使用其来创建 topic。

**4\. auto.leader.rebalance.enable**

自动 rebalance，默认为 true。其作用是通过后台线程定期扫描检查，在一定条件下触发重新 leader 选举；在生产环境中，不建议开启，因为替换 leader 在性能上没有什么提升。

**5\. background.threads**

后台线程数，默认为 10。用于后台操作的线程，可以不用改动。

**6\. broker.id**

Broker 的唯一标识，用于区分不同的 Broker。kafka 的检查就是基于此 id 是否在 zookeeper 中/brokers/ids 目录下是否有相应的 id 目录来判定 Broker 是否健康。

**7\. compression.type**

压缩类型。此配置可以接受的压缩类型有 gzip、snappy、lz4。另外可以不设置，即保持和生产端相同的压缩格式。

**8\. delete.topic.enable**

启用删除 topic。如果关闭，则无法使用 admin 工具进行 topic 的删除操作。

**9\. leader.imbalance.check.interval.seconds**

partition 检查重新 rebalance 的周期时间

**10\. leader.imbalance.per.broker.percentage**

标识每个 Broker 失去平衡的比率，如果超过改比率，则执行重新选举 Broker 的 leader

**11\. log.dir / log.dirs**

保存 kafka 日志数据的位置。如果 log.dirs 没有设置，则使用 log.dir 指定的目录进行日志数据存储。

**12\. log.flush.interval.messages**

partition 分区的数据量达到指定大小时，对数据进行一次刷盘操作。比如设置了 1024k 大小，当 partition 积累的数据量达到这个数值时则将数据写入到磁盘上。

**13\. log.flush.interval.ms**

数据写入磁盘时间间隔，即内存中的数据保留多久就持久化一次，如果没有设置，则使用 log.flush.scheduler.interval.ms 参数指定的值。

**14\. log.retention.bytes**

表示 topic 的容量达到指定大小时，则对其数据进行清除操作，默认为-1，永远不删除。

**15\. log.retention.hours**

标示 topic 的数据最长保留多久，单位是小时

**16\. log.retention.minutes**

表示 topic 的数据最长保留多久，单位是分钟，如果没有设置该参数，则使用 log.retention.hours 参数

**17\. log.retention.ms**

表示 topic 的数据最长保留多久，单位是毫秒，如果没有设置该参数，则使用 log.retention.minutes 参数

**18\. log.roll.hours**

新的 segment 创建周期，单位小时。kafka 数据是以 segment 存储的，当周期时间到达时，就创建一个新的 segment 来存储数据。

**19\. log.segment.bytes**

segment 的大小。当 segment 大小达到指定值时，就新创建一个 segment。

**20\. message.max.bytes**

topic 能够接收的最大文件大小。需要注意的是 producer 和 consumer 端设置的大小需要一致。

**21\. min.insync.replicas**

最小副本同步个数。当 producer 设置了 request.required.acks 为-1 时，则 topic 的副本数要同步至该参数指定的个数，如果达不到，则 producer 端会产生异常。

**22\. num.io.threads**

指定 io 操作的线程数

**23\. num.network.threads**

执行网络操作的线程数

**24\. num.recovery.threads.per.data.dir**

每个数据目录用于恢复数据的线程数

**25\. num.replica.fetchers**

从 leader 备份数据的线程数

**26\. offset.metadata.max.bytes**

允许消费者端保存 offset 的最大个数

**27\. offsets.commit.timeout.ms**

offset 提交的延迟时间

**28\. offsets.topic.replication.factor**

topic 的 offset 的备份数量。该参数建议设置更高保证系统更高的可用性

**29\. port**

端口号，Broker 对外提供访问的端口号。

**30\. request.timeout.ms**

Broker 接收到请求后的最长等待时间，如果超过设定时间，则会给客户端发送错误信息

**31\. zookeeper.connection.timeout.ms**

客户端和 zookeeper 建立连接的超时时间，如果没有设置该参数，则使用 zookeeper.session.timeout.ms 值

**32\. connections.max.idle.ms**

空连接的超时时间。即空闲连接超过该时间时则自动销毁连接。

### **Producer Config**

1.  **bootstrap.servers**

服务端列表。即接收生产消息的服务端列表

**2\. key.serializer**

消息键的序列化方式。指定 key 的序列化类型

3..**value.serializer**

消息内容的序列化方式。指定 value 的序列化类型

4..**acks**

消息写入 Partition 的个数。通常可以设置为 0，1，all；当设置为 0 时，只需要将消息发送完成后就完成消息发送功能；当设置为 1 时，即 Leader Partition 接收到数据并完成落盘；当设置为 all 时，即主从 Partition 都接收到数据并落盘。

5..**buffer.memory**

客户端缓存大小。即 Producer 生产的数据量缓存达到指定值时，将缓存数据一次发送的 Broker 上。

6..**compression.type**

压缩类型。指定消息发送前的压缩类型，通常有 none, gzip, snappy, or, lz4 四种。不指定时消息默认不压缩。

7..**retries**

消息发送失败时重试次数。当该值设置大于 0 时，消息因为网络异常等因素导致消息发送失败进行重新发送的次数。

### **Consumer Config**

1.  **bootstrap.servers**

服务端列表。即消费端从指定的服务端列表中拉取消息进行消费。

2..**key.deserializer**

消息键的反序列化方式。指定 key 的反序列化类型，与序列化时指定的类型相对应。

3..**value.deserializer**

消息内容的反序列化方式。指定 value 的反序列化类型，与序列化时指定的类型相对应。

4..**fetch.min.bytes**

抓取消息的最小内容。指定每次向服务端拉取的最小消息量。

5..**group.id**

消费组中每个消费者的唯一表示。

6..**heartbeat.interval.ms**

心跳检查周期。即在周期性的向 group coordinator 发送心跳，当服务端发生 rebalance 时，会将消息发送给各个消费者。该参数值需要小于 session.timeout.ms，通常为后者的 1/3。

7..**max.partition.fetch.bytes**

Partition 每次返回的最大数据量大小。

**8\. session.timeout.ms**

consumer 失效的时间。即 consumer 在指定的时间后，还没有响应则认为该 consumer 已经发生故障了。

**9\. auto.offset.reset**

当 kafka 中没有初始偏移量或服务器上不存在偏移量时，指定从哪个位置开始消息消息。earliest：指定从头开始；latest：从最新的数据开始消费。

### **Kafka Connect Config**

1.  **group.id**

消费者在消费组中的唯一标识

**2\. internal.key.converter**

内部 key 的转换类型。

**3\. internal.value.converter**

内部 value 的转换类型。

**4\. key.converter**

服务端接收到 key 时指定的转换类型。

5..**value.converter**

服务端接收到 value 时指定的转换类型。

**6\. bootstrap.servers**

服务端列表。

**7\. heartbeat.interval.ms**

心跳检测，与 consumer 中指定的配置含义相同。

**8\. session.timeout.ms**

session 有效时间，与 consumer 中指定的配置含义相同。

## **总结**

本文总结了平时经常用到的一些 Kafka 配置及命令说明，方便随时查看；喜欢的朋友可以收藏以备不时之需。


## 参考文章
https://blog.csdn.net/cao131502
https://zhuanlan.zhihu.com/p/137811719