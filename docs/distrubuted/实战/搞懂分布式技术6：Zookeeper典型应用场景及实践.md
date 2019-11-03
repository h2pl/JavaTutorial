# 一.ZooKeeper典型应用场景实践

ZooKeeper是一个`高可用的分布式数据管理与系统协调框架`。`基于对Paxos算法的实现，使该框架保证了分布式环境中数据的强一致性`，也正是基于这样的特性，使得ZooKeeper解决很多分布式问题。网上对ZK的应用场景也有不少介绍，本文将介绍比较常用的项目例子，系统地对ZK的应用场景进行一个分门归类的介绍。

值得注意的是，`ZK并非天生就是为这些应用场景设计的，都是后来众多开发者根据其框架的特性`，利用其提供的一系列API接口（或者称为原语集），`摸索出来的典型使用方法`。因此，也非常欢迎读者分享你在ZK使用上的奇技淫巧。

# 1 Zookeeper数据模型

Zookeeper 会维护`一个具有层次关系的数据结构`，它非常类似于一个标准的文件系统，如图所示：

[![](https://static.oschina.net/uploads/img/201511/17163447_w7k1.png)](https://static.oschina.net/uploads/img/201511/17163447_w7k1.png)

**图中的每个节点称为一个znode. 每个znode由3部分组成：**

1.  stat. 此为状态信息, 描述该`znode的版本, 权限等信息`；

2.  data. 与该znode`关联的数据`；

3.  children. 该znode下的`子节点`；

**Zookeeper 这种数据结构有如下这些特点：**

1.  每个子目录项如 NameService 都被称作为 znode，这个 znode 是被它所在的路径唯一标识，如 Server1 这个 znode 的标识为 /NameService/Server1；

2.  znode 可以有子节点目录，并且每个 znode 可以存储数据，注意 `EPHEMERAL 类型的目录节点不能有子节点目录`；

3.  znode 是有版本的，每个 znode 中存储的数据可以有多个版本，`也就是一个访问路径中可以存储多份数据`；

4.  znode 可以是临时节点，`一旦创建这个 znode 的客户端与服务器失去联系，这个 znode 也将自动删除`，Zookeeper 的客户端和服务器通信`采用长连接方式，每个客户端和服务器通过心跳来保持连接`，这个连接状态称为 session，如果 znode 是临时节点，这个 session 失效，znode 也就删除了；

5.  znode 的`目录名可以自动编号`，如 App1 已经存在，再创建的话，将会自动命名为 App2；

6.  znode `可以被监控，包括这个目录节点中存储的数据的修改，子节点目录的变化等`，一旦变化可以通知设置监控的客户端，`这个是 Zookeeper 的核心特性`，Zookeeper 的很多功能都是基于这个特性实现的，后面在典型的应用场景中会有实例介绍；

**znode节点的状态信息：**

使用get命令获取指定节点的数据时, `同时也将返回该节点的状态信息, 称为Stat`. 其包含如下字段:

<pre>czxid. 节点创建时的zxid；
mzxid. 节点最新一次更新发生时的zxid；
ctime. 节点创建时的时间戳；
mtime. 节点最新一次更新发生时的时间戳；
dataVersion. 节点数据的更新次数；
cversion. 其子节点的更新次数；
aclVersion. 节点ACL(授权信息)的更新次数；
ephemeralOwner. 如果该节点为ephemeral节点, ephemeralOwner值表示与该节点绑定的session id. 如果该节点不是              ephemeral节点, ephemeralOwner值为0\. 至于什么是ephemeral节点；
dataLength. 节点数据的字节数；
numChildren. 子节点个数；
​</pre>

**zxid：**

znode节点的状态信息中包含czxid和mzxid, 那么什么是zxid呢?

`ZooKeeper状态的每一次改变, 都对应着一个递增的Transaction id, 该id称为zxid`. 由于zxid的递增性质, 如果zxid1小于zxid2, 那么zxid1肯定先于zxid2发生. `创建任意节点, 或者更新任意节点的数据, 或者删除任意节点, 都会导致Zookeeper状态发生改变, 从而导致zxid的值增加`.

**session：**

在client和server通信之前, 首先需要建立连接, 该连接称为session. 连接建立后, 如果发生连接超时, 授权失败, 或者显式关闭连接, 连接便处于CLOSED状态, 此时session结束.

**节点类型：**

讲述节点状态的ephemeralOwner字段时, 提到过有的节点是ephemeral节点, 而有的并不是. 那么节点都具有哪些类型呢? 每种类型的节点又具有哪些特点呢?

`persistent. persistent节点不和特定的session绑定`, 不会随着创建该节点的session的结束而消失, 而是**一直存在, 除非该节点被显式删除**.

`ephemeral. ephemeral(临时)节点是临时性的, 如果创建该节点的session结束了, 该节点就会被自动删除`. `ephemeral节点不能拥有子节点`. 虽然ephemeral节点与创建它的session绑定, 但只要该节点没有被删除, 其他session就可以读写该节点中关联的数据. `使用-e参数指定创建ephemeral节点`.

<pre>create -e /xing/ei world</pre>

`sequence. 严格的说, sequence(顺序)并非节点类型中的一种`. **sequence节点既可以是ephemeral的, 也可以是persistent的**. `创建sequence节点时, ZooKeeper server会在指定的节点名称后加上一个数字序列, 该数字序列是递增的`. 因此可以**多次创建相同的sequence节点, 而得到不同的节点**. `使用-s参数指定创建sequence节点`.

<pre>[zk: localhost:4180(CONNECTED) 0] create -s /xing/item world  
Created /xing/item0000000001  
[zk: localhost:4180(CONNECTED) 1] create -s /xing/item world  
Created /xing/item0000000002  
[zk: localhost:4180(CONNECTED) 2] create -s /xing/item world  
Created /xing/item0000000003  
[zk: localhost:4180(CONNECTED) 3] create -s /xing/item world  
Created /xing/item0000000004
​</pre>

**watch：**

`watch的意思是监听感兴趣的事件`. 在命令行中, 以下几个命令可以指定是否监听相应的事件.

ls命令. ls命令的第一个参数指定znode, 第二个参数如果为true, 则说明监听该znode的**子节点的增减**, 以及该znode**本身的删除**事件.

<pre>[zk: localhost:4180(CONNECTED) 21] ls /xing true
[]
[zk: localhost:4180(CONNECTED) 22] create /xing/item item000
WATCHER::
 WatchedEvent state:SyncConnected type:NodeChildrenChanged path:/xing
Created /xing/item</pre>

`get命令. get命令的第一个参数指定znode, 第二个参数如果为true, 则说明监听该znode的更新和删除事件`.

<pre>[zk: localhost:4180(CONNECTED) 39] get /xing true
world
cZxid = 0x100000066
ctime = Fri May 17 22:30:01 CST 2013
mZxid = 0x100000066
mtime = Fri May 17 22:30:01 CST 2013
pZxid = 0x100000066
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 5
numChildren = 0
[zk: localhost:4180(CONNECTED) 40] create /xing/item item000
Created /xing/item
[zk: localhost:4180(CONNECTED) 41] rmr /xing
WATCHER::
 WatchedEvent state:SyncConnected type:NodeDeleted path:/xing
​</pre>

# 2 如何使用Zookeeper

Zookeeper 作为一个分布式的服务框架，`主要用来解决分布式集群中应用系统的一致性问题`，它能提供基于类似于文件系统的目录节点树方式的数据存储，但是 `Zookeeper 并不是用来专门存储数据的，它的作用主要是用来维护和监控你存储的数据的状态变化`。`通过监控这些数据状态的变化，从而可以达到基于数据的集群管理`，后面将会详细介绍 Zookeeper 能够解决的一些典型问题，这里先介绍一下，Zookeeper 的操作接口和简单使用示例。

## 2.1 常用接口操作

客户端要连接 Zookeeper 服务器可以通过创建 `org.apache.zookeeper.ZooKeeper` 的一个实例对象，然后调用这个类提供的接口来和服务器交互。

前面说了 `ZooKeeper 主要是用来维护和监控一个目录节点树中存储的数据的状态`，所有我们能够操作 ZooKeeper 的也和操作目录节点树大体一样，如创建一个目录节点，给某个目录节点设置数据，获取某个目录节点的所有子目录节点，给某个目录节点设置权限和监控这个目录节点的状态变化。

**ZooKeeper 基本的操作示例：**

<pre>public class ZkDemo {
 public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
 // 创建一个与服务器的连接
 ZooKeeper zk = new ZooKeeper("127.0.0.1:2180", 60000, new Watcher() {
 // 监控所有被触发的事件
 // 当对目录节点监控状态打开时，一旦目录节点的状态发生变化，Watcher 对象的 process 方法就会被调用。
 public void process(WatchedEvent event) {
 System.out.println("EVENT:" + event.getType());
 }
 });
 // 查看根节点
 // 获取指定 path 下的所有子目录节点，同样 getChildren方法也有一个重载方法可以设置特定的 watcher 监控子节点的状态
 System.out.println("ls / => " + zk.getChildren("/", true));
 // 判断某个 path 是否存在，并设置是否监控这个目录节点，这里的 watcher 是在创建 ZooKeeper 实例时指定的 watcher；
 // exists方法还有一个重载方法，可以指定特定的 watcher
 if (zk.exists("/node", true) == null) {
 // 创建一个给定的目录节点 path, 并给它设置数据；
 // CreateMode 标识有四种形式的目录节点，分别是：
 //     PERSISTENT：持久化目录节点，这个目录节点存储的数据不会丢失；
 //     PERSISTENT_SEQUENTIAL：顺序自动编号的目录节点，这种目录节点会根据当前已近存在的节点数自动加 1，然后返回给客户端已经成功创建的目录节点名；
 //     EPHEMERAL：临时目录节点，一旦创建这个节点的客户端与服务器端口也就是 session 超时，这种节点会被自动删除；
 //     EPHEMERAL_SEQUENTIAL：临时自动编号节点
 zk.create("/node", "conan".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
 System.out.println("create /node conan");
 // 查看/node节点数据
 System.out.println("get /node => " + new String(zk.getData("/node", false, null)));
 // 查看根节点
 System.out.println("ls / => " + zk.getChildren("/", true));
 }
 // 创建一个子目录节点
 if (zk.exists("/node/sub1", true) == null) {
 zk.create("/node/sub1", "sub1".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
 System.out.println("create /node/sub1 sub1");
 // 查看node节点
 System.out.println("ls /node => " + zk.getChildren("/node", true));
 }
 // 修改节点数据
 if (zk.exists("/node", true) != null) {
 // 给 path 设置数据，可以指定这个数据的版本号，如果 version 为 -1 怎可以匹配任何版本
 zk.setData("/node", "changed".getBytes(), -1);
 // 查看/node节点数据
 // 获取这个 path 对应的目录节点存储的数据，数据的版本等信息可以通过 stat 来指定，同时还可以设置是否监控这个目录节点数据的状态
 System.out.println("get /node => " + new String(zk.getData("/node", false, null)));
 }
 // 删除节点
 if (zk.exists("/node/sub1", true) != null) {
 // 删除 path 对应的目录节点，version 为 -1 可以匹配任何版本，也就删除了这个目录节点所有数据
 zk.delete("/node/sub1", -1);
 zk.delete("/node", -1);
 // 查看根节点
 System.out.println("ls / => " + zk.getChildren("/", true));
 }
 // 关闭连接
 zk.close();
 }
}
​</pre>

# 3 ZooKeeper 典型的应用场景

Zookeeper 从设计模式角度来看，是一个`基于观察者模式设计的分布式服务管理框架，它负责存储和管理大家都关心的数据，然后接受观察者的注册，一旦这些数据的状态发生变化，Zookeeper 就将负责通知已经在 Zookeeper 上注册的那些观察者做出相应的反应，从而实现集群中类似 Master/Slave 管理模式`，关于 Zookeeper 的详细架构等内部细节可以阅读 Zookeeper 的源码。

下面详细介绍这些典型的应用场景，也就是 Zookeeper 到底能帮我们解决那些问题？下面将给出答案。

## 3.1 统一命名服务（Name Service）

分布式应用中，通常需要有一套完整的命名规则，既能够产生唯一的名称又便于人识别和记住，通常情况下用树形的名称结构是一个理想的选择，**树形的名称结构是一个有层次的目录结构，既对人友好又不会重复**。说到这里你可能想到了 JNDI，没错 Zookeeper 的 Name Service **与 JNDI 能够完成的功能是差不多的**，它们都是将有层次的目录结构关联到一定资源上，但是 Zookeeper 的 Name Service 更加是广泛意义上的关联，也许你并不需要将名称关联到特定资源上，你可能只需要一个不会重复名称，**就像数据库中产生一个唯一的数字主键一样**。

`Name Service 已经是 Zookeeper 内置的功能`，你只要调用 Zookeeper 的 API 就能实现。如调用 create 接口就可以很容易创建一个目录节点。

命名服务也是分布式系统中比较常见的一类场景。在分布式系统中，**通过使用命名服务，客户端应用能够根据指定名字来获取资源或服务的地址，提供者等信息**`。`被命名的实体通常可以是集群中的机器，提供的服务地址，远程对象等等——这些我们都可以统称他们为名字（Name）`。其中较为常见的就是一些分布式服务框架中的服务地址列表。通过调用ZK提供的创建节点的API，能够很容易创建一个全局唯一的path，这个path就可以作为一个名称。

**命名服务实例：**

阿里巴巴集团开源的分布式服务框架**Dubbo中使用ZooKeeper来作为其命名服务，维护全局的服务地址列表**，在Dubbo实现中：

`服务提供者在启动的时候`，向ZK上的指定节点`/dubbo/${serviceName}/providers`目录下写入自己的URL地址，这个操作就完成了服务的发布。

`服务消费者启动的时候`，`订阅/dubbo/${serviceName}/providers`目录下的提供者URL地址， 并向`/dubbo/${serviceName} /consumers`目录下写入自己的URL地址。

注意，**所有向ZK上注册的地址都是临时节点**，这样就能够保证服务提供者和消费者能够**自动感应资源的变化**。 另外，Dubbo还有**针对服务粒度的监控**，方法是订阅/dubbo/${serviceName}目录下所有提供者和消费者的信息。

## 3.2 配置管理（Configuration Management）

配置的管理在分布式应用环境中很常见，例如同一个应用系统需要多台 PC Server 运行，但是它们运行的应用系统的某些配置项是相同的，如果要修改这些相同的配置项，那么就必须同时修改每台运行这个应用系统的 PC Server，这样非常麻烦而且容易出错。

像这样的配置信息完全可以交给 Zookeeper 来管理，`将配置信息保存在 Zookeeper 的某个目录节点中，然后将所有需要修改的应用机器监控配置信息的状态，一旦配置信息发生变化，每台应用机器就会收到 Zookeeper 的通知，然后从 Zookeeper 获取新的配置信息应用到系统中`。

[![](https://static.oschina.net/uploads/img/201511/18093533_GnwN.png)](https://static.oschina.net/uploads/img/201511/18093533_GnwN.png)

**发布与订阅模型，即所谓的配置中心**，顾名思义就是**发布者将数据发布到ZK节点上，供订阅者动态获取数据**，`实现配置信息的集中式管理和动态更新`。例如全局的配置信息，服务式服务框架的服务地址列表等就非常适合使用。

**配置管理实例：**

1.  `应用中用到的一些配置信息放到ZK上进行集中管理`。这类场景通常是这样：**应用在启动的时候会主动来获取一次配置`，同时，`在节点上注册一个Watcher，这样一来，以后每次配置有更新的时候，都会实时通知到订阅的客户端，从而达到获取最新配置信息的目的。**

2.  `分布式搜索服务中`，索引的元信息和服务器集群机器的节点状态存放在ZK的一些指定节点，供各个客户端订阅使用。

3.  `分布式日志收集系统`。这个系统的核心工作是收集分布在不同机器的日志。收集器通常是按照应用来分配收集任务单元，因此需要在ZK上创建一个以应用名作为path的节点P，并将这个应用的所有机器ip，以子节点的形式注册到节点P上，这样一来就能够实现机器变动的时候，能够实时通知到收集器调整任务分配。

4.  `系统中有些信息需要动态获取，并且还会存在人工手动去修改这个信息的发问`。通常是暴露出接口，例如JMX接口，来获取一些运行时的信息。引入ZK之后，就不用自己实现一套方案了，只要将这些信息存放到指定的ZK节点上即可。

注意：在上面提到的应用场景中，有个默认前提是：`数据量很小，但是数据更新可能会比较快的场景`。

## 3.3 集群管理（Group Membership）

Zookeeper 能够很容易的实现集群管理的功能，如有多台 Server 组成一个服务集群，那么`必须要一个“总管”知道当前集群中每台机器的服务状态，一旦有机器不能提供服务，集群中其它集群必须知道`，从而做出调整重新分配服务策略。同样`当增加集群的服务能力时，就会增加一台或多台 Server，同样也必须让“总管”知道`。

Zookeeper 不仅能够帮你维护当前的集群中机器的服务状态，而且能够帮你选出一个“总管”，让这个总管来管理集群，这就是 `Zookeeper 的另一个功能 Leader Election`。

它们的实现方式都是在 **Zookeeper 上创建一个 EPHEMERAL 类型的目录节点`，然后`每个 Server 在它们创建目录节点的父目录节点上调用 getChildren(String path, boolean watch) 方法并设置 watch 为 true`，由于是 EPHEMERAL 目录节点，当创建它的 Server 死去，这个目录节点也随之被删除，所以 Children 将会变化，这时 getChildren上的 Watch 将会被调用，所以其它 Server 就知道已经有某台 Server 死去了**。新增 Server 也是同样的原理。

Zookeeper 如何实现 Leader Election，也就是选出一个 Master Server。和前面的一样`每台 Server 创建一个 EPHEMERAL 目录节点，不同的是它还是一个 SEQUENTIAL 目录节点，所以它是个 EPHEMERAL_SEQUENTIAL 目录节点`。之所以它是 **EPHEMERAL_SEQUENTIAL** 目录节点，是因为我们可以给每台 Server 编号，我们可以`选择当前是最小编号的 Server 为 Master`，假如这个最小编号的 Server 死去，由于是 EPHEMERAL 节点，`死去的 Server 对应的节点也被删除，所以当前的节点列表中又出现一个最小编号的节点，我们就选择这个节点为当前 Master`。这样就实现了动态选择 Master，避免了传统意义上单 Master 容易出现单点故障的问题。

[![](https://static.oschina.net/uploads/img/201511/18095931_70Ol.png)](https://static.oschina.net/uploads/img/201511/18095931_70Ol.png)

**1\. 集群机器监控**

这通常用于那种`对集群中机器状态，机器在线率有较高要求的场景`，能够快速对集群中机器变化作出响应。这样的场景中，往往有一个监控系统，实时检测集群机器是否存活。过去的做法通常是：监控系统通过某种手段（比如ping）定时检测每个机器，或者每个机器自己定时向监控系统汇报“我还活着”。 这种做法可行，但是存在两个比较明显的问题：

1.  集群中机器有变动的时候，牵连修改的东西比较多。

2.  有一定的延时。

利用ZooKeeper有两个特性，就可以实现另一种集群机器存活性监控系统：

1.  `客户端在节点 x 上注册一个Watcher，那么如果 x 的子节点变化了，会通知该客户端`。

2.  `创建EPHEMERAL类型的节点，一旦客户端和服务器的会话结束或过期，那么该节点就会消失`。

`例如`：监控系统在 /clusterServers 节点上注册一个Watcher，以后每动态加机器，那么就往 /clusterServers 下创建一个 EPHEMERAL类型的节点：/clusterServers/{hostname}. 这样，监控系统就能够实时知道机器的增减情况，至于后续处理就是监控系统的业务了。

**2\. Master选举则是zookeeper中最为经典的应用场景了**

在分布式环境中，相同的业务应用分布在不同的机器上，`有些业务逻辑（例如一些耗时的计算，网络I/O处理），往往只需要让整个集群中的某一台机器进行执行，其余机器可以共享这个结果`，这样可以大大减少重复劳动，提高性能，于是`这个master选举便是这种场景下的碰到的主要问题`。

**利用ZooKeeper的强一致性，能够保证在分布式高并发情况下节点创建的全局唯一性**，即：同时有多个客户端请求创建 /currentMaster 节点，最终一定只有一个客户端请求能够创建成功。利用这个特性，就能很轻易的在分布式环境中进行集群选取了。

另外，这种场景演化一下，就是`动态Master选举`。这就要用到`EPHEMERAL_SEQUENTIAL类型节点的特性了`。

上文中提到，所有客户端创建请求，最终只有一个能够创建成功。在这里稍微变化下，就是**允许所有请求都能够创建成功，但是得有个创建顺序**，于是所有的请求最终在ZK上创建结果的一种可能情况是这样： /currentMaster/{sessionId}-1 ,/currentMaster/{sessionId}-2,/currentMaster/{sessionId}-3 ….. `每次选取序列号最小的那个机器作为Master，如果这个机器挂了，由于他创建的节点会马上消失，那么之后最小的那个机器就是Master了`。

**3\. 在搜索系统中，如果集群中每个机器都生成一份全量索引，不仅耗时，而且不能保证彼此之间索引数据一致。**因此让集群中的Master来进行全量索引的生成，然后同步到集群中其它机器。另外，Master选举的容灾措施是，可以随时进行手动指定master，就是说应用在zk在无法获取master信息时，可以通过比如http方式，向一个地方获取master。

**4\. 在Hbase中，也是使用ZooKeeper来实现动态HMaster的选举。**在Hbase实现中，会在ZK上存储一些ROOT表的地址和HMaster的地址，HRegionServer也会把自己以临时节点（Ephemeral）的方式注册到Zookeeper中，使得HMaster可以随时感知到各个HRegionServer的存活状态，同时，一旦HMaster出现问题，会重新选举出一个HMaster来运行，从而避免了HMaster的单点问题。

[![](https://static.oschina.net/uploads/img/201511/18102032_cFyc.png)](https://static.oschina.net/uploads/img/201511/18102032_cFyc.png)

## 3.4 共享锁（Locks）

共享锁在同一个进程中很容易实现，但是在跨进程或者在不同 Server 之间就不好实现了。Zookeeper 却很容易实现这个功能，实现方式也是`需要获得锁的 Server 创建一个 EPHEMERAL_SEQUENTIAL 目录节点，然后调用 getChildren方法获取当前的目录节点列表中最小的目录节点是不是就是自己创建的目录节点，如果正是自己创建的，那么它就获得了这个锁，如果不是那么它就调用 exists(String path, boolean watch) 方法并监控 Zookeeper 上目录节点列表的变化，一直到自己创建的节点是列表中最小编号的目录节点，从而获得锁，释放锁很简单，只要删除前面它自己所创建的目录节点就行了`。

分布式锁，这个主要得益于ZooKeeper为我们保证了数据的强一致性。锁服务可以分为两类，`一个是保持独占，另一个是控制时序`。

1.  **所谓保持独占，就是所有试图来获取这个锁的客户端，最终只有一个可以成功获得这把锁**。通常的做法是把zk上的一个znode看作是一把锁，通过create znode的方式来实现。`所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁`。

2.  **控制时序，就是所有视图来获取这个锁的客户端，最终都是会被安排执行，只是有个全局时序了**。做法和上面基本类似，只是这里 /distribute_lock 已经预先存在，客户端在它下面创建临时有序节点（这个可以通过节点的属性控制：CreateMode.EPHEMERAL_SEQUENTIAL来指定）。**Zk的父节点（/distribute_lock）维持一份sequence,保证子节点创建的时序性，从而也形成了每个客户端的全局时序。**

[![](https://static.oschina.net/uploads/img/201511/18103433_BJs7.png)](https://static.oschina.net/uploads/img/201511/18103433_BJs7.png)

[![](https://static.oschina.net/uploads/img/201511/18103709_c4c3.png)](https://static.oschina.net/uploads/img/201511/18103709_c4c3.png)

## 3.5 队列管理

Zookeeper 可以处理两种类型的队列：

1.  `当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达`，这种是同步队列。

2.  `队列按照 FIFO 方式进行入队和出队操作`，例如实现生产者和消费者模型。

**同步队列用 Zookeeper 实现的实现思路如下：**

创建一个父目录 /synchronizing，每个成员都监控标志（Set Watch）位目录 /synchronizing/start 是否存在，然后每个成员都加入这个队列，**加入队列的方式**就是创建 /synchronizing/member_i 的**临时目录节点**，然后每个成员获取 / synchronizing 目录的所有目录节点，也就是 member_i。判断 i 的值是否已经是成员的个数，如果小于成员个数等待 /synchronizing/start 的出现，`如果已经相等就创建 /synchronizing/start`。

[![](https://static.oschina.net/uploads/img/201511/18104311_7kNc.png)](https://static.oschina.net/uploads/img/201511/18104311_7kNc.png)

[![](https://static.oschina.net/uploads/img/201511/18104433_Rdxx.png)](https://static.oschina.net/uploads/img/201511/18104433_Rdxx.png)

**FIFO 队列用 Zookeeper 实现思路如下：**

实现的思路也非常简单，就是在特定的目录下**创建 SEQUENTIAL 类型的子目录 /queue_i**，这样就能保证所有成员加入队列时都是有编号的`，出队列时通过 getChildren( ) 方法可以返回当前所有的队列中的元素，然后消费其中最小的一个，这样就能保证 FIFO。

[![](https://static.oschina.net/uploads/img/201511/18104614_ncNj.png)](https://static.oschina.net/uploads/img/201511/18104614_ncNj.png)

## 3.6 负载均衡

这里说的负载均衡是指**软负载均衡**。在分布式环境中，为了保证高可用性，通常同一个应用或同一个服务的提供方都会部署多份，达到对等服务。而消费者就须要在这些对等的服务器中选择一个来执行相关的业务逻辑，`其中比较典型的是消息中间件中的生产者，消费者负载均衡`。

`消息中间件中发布者和订阅者的负载均衡`，linkedin开源的KafkaMQ和阿里开源的metaq都是通过zookeeper**来做到生产者、消费者的负载均衡**`。这里以metaq为例如讲下：

`生产者负载均衡`：metaq发送消息的时候，生产者在发送消息的时候必须选择一台broker上的一个分区来发送消息，因此metaq在运行过程中，会把所有broker和对应的分区信息全部注册到ZK指定节点上，默认的策略是一个依次轮询的过程，生产者在通过ZK获取分区列表之后，会按照brokerId和partition的顺序排列组织成一个有序的分区列表，发送的时候按照从头到尾循环往复的方式选择一个分区来发送消息。

`消费负载均衡`： 在消费过程中，一个消费者会消费一个或多个分区中的消息，但是一个分区只会由一个消费者来消费。MetaQ的消费策略是：

1.  每个分区针对同一个group只挂载一个消费者。

2.  如果同一个group的消费者数目大于分区数目，则多出来的消费者将不参与消费。

3.  如果同一个group的消费者数目小于分区数目，则有部分消费者需要额外承担消费任务。

    在某个消费者故障或者重启等情况下，其他消费者会感知到这一变化（通过 zookeeper watch消费者列表），然后重新进行负载均衡，保证所有的分区都有消费者进行消费。

## 3.7 分布式通知/协调

`ZooKeeper中特有watcher注册与异步通知机制，能够很好的实现分布式环境下不同系统之间的通知与协调，实现对数据变更的实时处理`。使用方法通常是不同系统都对ZK上同一个znode进行注册，监听znode的变化（包括znode本身内容及子节点的），其中一个系统update了znode，那么另一个系统能够收到通知，并作出相应处理。

1.  `另一种心跳检测机制`：检测系统和被检测系统之间并不直接关联起来，而是通过zk上某个节点关联，大大减少系统耦合。

2.  `另一种系统调度模式`：某系统有控制台和推送系统两部分组成，控制台的职责是控制推送系统进行相应的推送工作。管理人员在控制台作的一些操作，实际上是修改了ZK上某些节点的状态，而ZK就把这些变化通知给他们注册Watcher的客户端，即推送系统，于是，作出相应的推送任务。

3.  `另一种工作汇报模式`：一些类似于任务分发系统，子任务启动后，到zk来注册一个临时节点，并且定时将自己的进度进行汇报（将进度写回这个临时节点），这样任务管理者就能够实时知道任务进度。

总之，使用zookeeper来进行分布式通知和协调能够大大降低系统之间的耦合。

## 二:典型场景描述总结

#### **数据发布与订阅**(配置管理)

发布与订阅即所谓的配置管理，顾名思义就是将数据发布到zk节点上，供订阅者动态获取数据，实现配置信息的集中式管理和动态更新。

例如全局的配置信息，地址列表等就非常适合使用。

发布/订阅系统一般有两种设计模式，分别是推(Push)模式和拉(Pull)模式。

推模式

服务端主动将数据更新发送给所有订阅的客户端。

拉模式

客户端通过采用定时轮询拉取。

ZooKeeper采用的是推拉相结合的方式：客户端向服务端注册自己需要关注的节点，一旦该节点的数据发生变更，那么服务端就会向相应的客户端发送Watcher事件通知，客户端接收到这个消息通知之后，需要主动到服务端获取最新的数据。

#### **负载均衡**

负载均衡(Load Balance)是一种相当常见的计算机网络技术，用来对多个计算机(计算机集群)、网络连接、CPU、硬盘驱动器或其他资源进行分配负载，以达到优化资源使用、最大化吞吐率、最小化响应时间和避免过载的目的。通常，负载均衡可以分为硬件和软件负载均衡两类

#### **分布通知/协调**

分布式协调/通知是将不同的分布式组件有机结合起来的关键所在。对于一个在多台机器上部署运行的应用而言，通常需要一个协调者(Coordinator)来控制整个系统的运行流程，例如分布式事务的处理、机器间的相互协调等。同时，引入这样一个协调者，便于将分布式协调的职责从应用中分离出来，从而大大减少系统之间的耦合性，而且能够显著提高系统的可扩展性。

ZooKeeper 中特有watcher注册与异步通知机制，能够很好的实现分布式环境下不同系统之间的通知与协调，实现对数据变更的实时处理。

使用方法通常是不同系统都对 ZK上同一个znode进行注册，监听znode的变化（包括znode本身内容及子节点的），其中一个系统update了znode，那么另一个系统能 够收到通知，并作出相应处理。

\1\. 另一种心跳检测机制：检测系统和被检测系统之间并不直接关联起来，而是通过zk上某个节点关联，大大减少系统耦合。

\2\. 另一种系统调度模式：某系统有控制台和推送系统两部分组成，控制台的职责是控制推送系统进行相应的推送工作。管理人员在控制台作的一些操作，实际上是修改 了ZK上某些节点的状态，而zk就把这些变化通知给他们注册Watcher的客户端，即推送系统，于是，作出相应的推送任务。

\3\. 另一种工作汇报模式：一些类似于任务分发系统，子任务启动后，到zk来注册一个临时节点，并且定时将自己的进度进行汇报（将进度写回这个临时节点），这样任务管理者就能够实时知道任务进度。总之，使用zookeeper来进行分布式通知和协调能够大大降低系统之间的耦合。

#### **命名服务**

在分布式系统中，被命名的实体通常是集群中的机器、提供的服务地址或远程对象等--这些我们都可以统称他们为名字，其中比较常见的就是一些分布式服务框架(RPC、RMI)中的服务地址列表，通过使用命名服务，客户端应用能够根据指定名字来获取资源的实体、服务地址和提供者信息等。

ZooKeeper提供的命名服务功能与JNDI技术有类似的地方，都能够帮助应用系统通过一个资源引用的方式来实现对资源的定位与使用。另外，广义上命名服务的资源定位都不是真正意义的实体资源--在分布式环境中，上层应用仅仅需要一个全局唯一的名字，类似于数据库的唯一主键。，通过调用zk的create node api，能够很容易创建一个全局唯一的path，这个path就可以作为一个名称。所谓ID，就是一个能唯一标识某个对象的标识符。

#### **分布式锁**

分布式锁，这个主要得益于ZooKeeper为我们保证了数据的强一致性，即用户只要完全相信每时每刻，zk集群中任意节点（一个zk server）上的相同znode的数据是一定是相同的。锁服务可以分为两类，一个是保持独占，另一个是控制时序。

保持独占:就是所有试图来获取这个锁的客户端，最终只有一个可以成功获得这把锁。通常的做法是把zk上的一个znode看作是一把锁，通过create znode的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁。

控制时序:就是所有视图来获取这个锁的客户端，最终都是会被安排执行，只是有个全局时序了。

做法和上面基本类似，只是这里 /distribute_lock 已经预先存在，客户端在它下面创建临时有序节点

（这个可以通过节点的属性控制：CreateMode.EPHEMERAL_SEQUENTIAL来指定）。Zk的父节点（/distribute_lock）维持一份sequence,保证子节点创建的时序性，从而也形成了每个客户端的全局时序。

原理:(借助Zookeeper 可以实现这种分布式锁：需要获得锁的 Server 创建一个 EPHEMERAL_SEQUENTIAL 目录节点，然后调用 getChildren()方法获取列表中最小的目录节点，如果最小节点就是自己创建的目录节点，那么它就获得了这个锁，如果不是那么它就调用 exists() 方法并监控前一节点的变化，一直到自己创建的节点成为列表中最小编号的目录节点，从而获得锁。释放锁很简单，只要删除它自己所创建的目录节点就行了。)

#### **集群管理**

\1\. 集群机器监控：这通常用于那种对集群中机器状态，机器在线率有较高要求的场景，能够快速对集群中机器变化作出响应。这样的场景中，往往有一个监控系统，实时检测集群机器是否存活。过去的做法通常是：监控系统通过某种手段（比如ping）定时检测每个机器，或者每个机器自己定时向监控系统汇报“我还活着”。 这种做法可行，但是存在两个比较明显的问题：

\1\. 集群中机器有变动的时候，牵连修改的东西比较多。

\2\. 有一定的延时。利用ZooKeeper有两个特性，就可以实时另一种集群机器存活性监控系统：

a. 客户端在节点 x 上注册一个Watcher，那么如果 x 的子节点变化了，会通知该客户端。

b. 创建EPHEMERAL类型的节点，一旦客户端和服务器的会话结束或过期，那么该节点就会消失。

应用:

Master选举

则是zookeeper中最为经典的使用场景了。

在分布式环境中，相同的业务应用分布在不同的机器上，有些业务逻辑（例如一些耗时的计算，网络I/O处理），往往只需要让整个集群中的某一台机器进行执行， 其余机器可以共享这个结果，这样可以大大减少重复劳动，提高性能，于是这个master选举便是这种场景下的碰到的主要问题。

利用ZooKeeper的强一致性，能够保证在分布式高并发情况下节点创建的全局唯一性，即：同时有多个客户端请求创建 /currentMaster 节点，最终一定只有一个客户端请求能够创建成功。

#### **分布式队列**

两种类型的队列：

1、 同步队列，当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达。

2、队列按照 FIFO 方式进行入队和出队操作。

第一类，在约定目录下创建临时目录节点，监听节点数目是否是我们要求的数目。

第二类，和分布式锁服务中的控制时序场景基本原理一致，入列有编号，出列按编号。

同步队列。一个job由多个task组成，只有所有任务完成后，job才运行完成。可为job创建一个/job目录，然后在该目录下，为每个完成的task创建一个临时znode，一旦临时节点数目达到task总数，则job运行完成。