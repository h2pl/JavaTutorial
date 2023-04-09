# Table of Contents

  * [Replication（主从复制）](#replication（主从复制）)
    * [配置主服务器](#配置主服务器)
    * [配置从服务器](#配置从服务器)
    * [测试](#测试)
  * [Sentinel（哨兵）](#sentinel（哨兵）)
    * [配置Sentinel](#配置sentinel)
    * [启动 Sentinel](#启动-sentinel)
    * [测试](#测试-1)
  * [Twemproxy](#twemproxy)
  * [Codis](#codis)
  * [Redis 3.0集群](#redis-30集群)
    * [环境搭建](#环境搭建)
* [根据实际情况修改](#根据实际情况修改)
* [允许redis支持集群模式](#允许redis支持集群模式)
* [节点配置文件，由redis自动维护](#节点配置文件，由redis自动维护)
* [节点超时毫秒](#节点超时毫秒)
* [开启AOF同步模式](#开启aof同步模式)
    * [创建集群](#创建集群)


## Replication（主从复制）

Redis的replication机制允许slave从master那里通过网络传输拷贝到完整的数据备份，从而达到主从机制。为了实现主从复制，我们准备三个redis服务，依次命名为master，slave1，slave2。

### 配置主服务器

为了测试效果，我们先修改主服务器的配置文件redis.conf的端口信息



1.  port 6300


### 配置从服务器

replication相关的配置比较简单，只需要把下面一行加到slave的配置文件中。你只需要把ip地址和端口号改一下。

1.  slaveof 192.168.1.1  6379



我们先修改从服务器1的配置文件redis.conf的端口信息和从服务器配置。




1.  port 6301
2.  slaveof 127.0.0.1  6300



我们再修改从服务器2的配置文件redis.conf的端口信息和从服务器配置。




1.  port 6302
2.  slaveof 127.0.0.1  6300



值得注意的是，从redis2.6版本开始，slave支持只读模式，而且是默认的。可以通过配置项slave-read-only来进行配置。
此外，如果master通过requirepass配置项设置了密码，slave每次同步操作都需要验证密码，可以通过在slave的配置文件中添加以下配置项




1.  masterauth <password>



### 测试

分别启动主服务器，从服务器，我们来验证下主从复制。我们在主服务器写入一条消息，然后再其他从服务器查看是否成功复制了。

## Sentinel（哨兵）

主从机制，上面的方案中主服务器可能存在单点故障，万一主服务器宕机，这是个麻烦事情，所以Redis提供了Redis-Sentinel，以此来实现主从切换的功能，类似与zookeeper。

Redis-Sentinel是Redis官方推荐的高可用性(HA)解决方案，当用Redis做master-slave的高可用方案时，假如master宕机了，Redis本身(包括它的很多客户端)都没有实现自动进行主备切换，而Redis-Sentinel本身也是一个独立运行的进程，它能监控多个master-slave集群，发现master宕机后能进行自动切换。

它的主要功能有以下几点

*   监控（Monitoring）：不断地检查redis的主服务器和从服务器是否运作正常。
*   提醒（Notification）：如果发现某个redis服务器运行出现状况，可以通过 API 向管理员或者其他应用程序发送通知。
*   自动故障迁移（Automatic failover）：能够进行自动切换。当一个主服务器不能正常工作时，会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器； 当客户端试图连接失效的主服务器时， 集群也会向客户端返回新主服务器的地址， 使得集群可以使用新主服务器代替失效服务器。

Redis Sentinel 兼容 Redis 2.4.16 或以上版本， 推荐使用 Redis 2.8.0 或以上的版本。

### 配置Sentinel

必须指定一个sentinel的配置文件sentinel.conf，如果不指定将无法启动sentinel。首先，我们先创建一个配置文件sentinel.conf


````

port 26379
sentinel monitor mymaster 127.0.0.1  6300  2
````


官方典型的配置如下


````

sentinel monitor mymaster 127.0.0.1  6379  2
sentinel down-after-milliseconds mymaster 60000
sentinel failover-timeout mymaster 180000
sentinel parallel-syncs mymaster 1

sentinel monitor resque 192.168.1.3  6380  4
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
````


配置文件只需要配置master的信息就好啦，不用配置slave的信息，因为slave能够被自动检测到(master节点会有关于slave的消息)。

需要注意的是，配置文件在sentinel运行期间是会被动态修改的，例如当发生主备切换时候，配置文件中的master会被修改为另外一个slave。这样，之后sentinel如果重启时，就可以根据这个配置来恢复其之前所监控的redis集群的状态。

接下来我们将一行一行地解释上面的配置项：


````

sentinel monitor mymaster 127.0.0.1  6379  2
````


这行配置指示 Sentinel 去监视一个名为 mymaster 的主服务器， 这个主服务器的 IP 地址为 127.0.0.1 ， 端口号为 6300， 而将这个主服务器判断为失效至少需要 2 个 Sentinel 同意，只要同意 Sentinel 的数量不达标，自动故障迁移就不会执行。

不过要注意， 无论你设置要多少个 Sentinel 同意才能判断一个服务器失效， 一个 Sentinel 都需要获得系统中多数（majority） Sentinel 的支持， 才能发起一次自动故障迁移， 并预留一个给定的配置纪元 （configuration Epoch ，一个配置纪元就是一个新主服务器配置的版本号）。换句话说， 在只有少数（minority） Sentinel 进程正常运作的情况下， Sentinel 是不能执行自动故障迁移的。sentinel集群中各个sentinel也有互相通信，通过gossip协议。

除了第一行配置，我们发现剩下的配置都有一个统一的格式:



````
sentinel <option_name>  <master_name>  <option_value>
````


接下来我们根据上面格式中的option_name一个一个来解释这些配置项：

*   down-after-milliseconds 选项指定了 Sentinel 认为服务器已经断线所需的毫秒数。
*   parallel-syncs 选项指定了在执行故障转移时， 最多可以有多少个从服务器同时对新的主服务器进行同步， 这个数字越小， 完成故障转移所需的时间就越长。

### 启动 Sentinel

对于 redis-sentinel 程序， 你可以用以下命令来启动 Sentinel 系统


````

redis-sentinel sentinel.conf
````


对于 redis-server 程序， 你可以用以下命令来启动一个运行在 Sentinel 模式下的 Redis 服务器



````
redis-server sentinel.conf --sentinel
````


以上两种方式，都必须指定一个sentinel的配置文件sentinel.conf， 如果不指定将无法启动sentinel。sentinel默认监听26379端口，所以运行前必须确定该端口没有被别的进程占用。
![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/sentinel01.png)

### 测试

此时，我们开启两个Sentinel，关闭主服务器，我们来验证下Sentinel。发现，服务器发生切换了。
![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/sentinel02.png)
当6300端口的这个服务重启的时候，他会变成6301端口服务的slave。

## Twemproxy

Twemproxy是由Twitter开源的Redis代理， Redis客户端把请求发送到Twemproxy，Twemproxy根据路由规则发送到正确的Redis实例，最后Twemproxy把结果汇集返回给客户端。

Twemproxy通过引入一个代理层，将多个Redis实例进行统一管理，使Redis客户端只需要在Twemproxy上进行操作，而不需要关心后面有多少个Redis实例，从而实现了Redis集群。
![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/twemproxy01.png)
Twemproxy本身也是单点，需要用Keepalived做高可用方案。

这么些年来，Twenproxy作为应用范围最广、稳定性最高、最久经考验的分布式中间件，在业界广泛使用。

但是，Twemproxy存在诸多不方便之处，最主要的是，Twemproxy无法平滑地增加Redis实例，业务量突增，需增加Redis服务器；业务量萎缩，需要减少Redis服务器。但对Twemproxy而言，基本上都很难操作。其次，没有友好的监控管理后台界面，不利于运维监控。

## Codis

Codis解决了Twemproxy的这两大痛点，由豌豆荚于2014年11月开源，基于Go和C开发、现已广泛用于豌豆荚的各种Redis业务场景。

Codis 3.x 由以下组件组成：

*   Codis Server：基于 redis-2.8.21 分支开发。增加了额外的数据结构，以支持 slot 有关的操作以及数据迁移指令。具体的修改可以参考文档 redis 的修改。
*   Codis Proxy：客户端连接的 Redis 代理服务, 实现了 Redis 协议。 除部分命令不支持以外(不支持的命令列表)，表现的和原生的 Redis 没有区别（就像 Twemproxy）。对于同一个业务集群而言，可以同时部署多个 codis-proxy 实例；不同 codis-proxy 之间由 codis-dashboard 保证状态同步。
*   Codis Dashboard：集群管理工具，支持 codis-proxy、codis-server 的添加、删除，以及据迁移等操作。在集群状态发生改变时，codis-dashboard 维护集群下所有 codis-proxy 的状态的一致性。对于同一个业务集群而言，同一个时刻 codis-dashboard 只能有 0个或者1个；所有对集群的修改都必须通过 codis-dashboard 完成。
*   Codis Admin：集群管理的命令行工具。可用于控制 codis-proxy、codis-dashboard 状态以及访问外部存储。
*   Codis FE：集群管理界面。多个集群实例共享可以共享同一个前端展示页面；通过配置文件管理后端 codis-dashboard 列表，配置文件可自动更新。
*   Codis HA：为集群提供高可用。依赖 codis-dashboard 实例，自动抓取集群各个组件的状态；会根据当前集群状态自动生成主从切换策略，并在需要时通过 codis-dashboard 完成主从切换。
*   Storage：为集群状态提供外部存储。提供 Namespace 概念，不同集群的会按照不同 product name 进行组织；目前仅提供了 Zookeeper 和 Etcd 两种实现，但是提供了抽象的 interface 可自行扩展。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/codis02.png)

Codis引入了Group的概念，每个Group包括1个Redis Master及一个或多个Redis Slave，这是和Twemproxy的区别之一，实现了Redis集群的高可用。当1个Redis Master挂掉时，Codis不会自动把一个Slave提升为Master，这涉及数据的一致性问题，Redis本身的数据同步是采用主从异步复制，当数据在Maste写入成功时，Slave是否已读入这个数据是没法保证的，需要管理员在管理界面上手动把Slave提升为Master。

Codis使用，可以参考官方文档[https://github.com/CodisLabs/codis/blob/release3.0/doc/tutorial_zh.md](https://github.com/CodisLabs/codis/blob/release3.0/doc/tutorial_zh.md)

## Redis 3.0集群

Redis 3.0集群采用了P2P的模式，完全去中心化。支持多节点数据集自动分片，提供一定程度的分区可用性，部分节点挂掉或者无法连接其他节点后，服务可以正常运行。Redis 3.0集群采用Hash Slot方案，而不是一致性哈希。Redis把所有的Key分成了16384个slot，每个Redis实例负责其中一部分slot。集群中的所有信息（节点、端口、slot等），都通过节点之间定期的数据交换而更新。

Redis客户端在任意一个Redis实例发出请求，如果所需数据不在该实例中，通过重定向命令引导客户端访问所需的实例。

Redis 3.0集群，目前支持的cluster特性

*   节点自动发现
*   slave->master 选举,集群容错
*   Hot resharding:在线分片
*   集群管理:cluster xxx
*   基于配置(nodes-port.conf)的集群管理
*   ASK 转向/MOVED 转向机制

* ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/redis3-cluster01.png)

如上图所示，所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽。节点的fail是通过集群中超过半数的节点检测失效时才生效。客户端与redis节点直连，不需要中间proxy层。客户端不需要连接集群所有节点，连接集群中任何一个可用节点即可。redis-cluster把所有的物理节点映射到[0-16383]slot上cluster负责维护node<->slot<->value。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/redis3-cluster02.png)

选举过程是集群中所有master参与，如果半数以上master节点与master节点通信超时，认为当前master节点挂掉。

当集群不可用时，所有对集群的操作做都不可用，收到((error) CLUSTERDOWN The cluster is down)错误。如果集群任意master挂掉，且当前master没有slave，集群进入fail状态，也可以理解成进群的slot映射[0-16383]不完成时进入fail状态。如果进群超过半数以上master挂掉，无论是否有slave集群进入fail状态。

### 环境搭建

现在，我们进行集群环境搭建。集群环境至少需要3个主服务器节点。本次测试，使用另外3个节点作为从服务器的节点，即3个主服务器，3个从服务器。

修改配置文件，其它的保持默认即可。



````
# 根据实际情况修改

port 7000  

# 允许redis支持集群模式

cluster-enabled yes 

# 节点配置文件，由redis自动维护

cluster-config-file nodes.conf 

# 节点超时毫秒

cluster-node-timeout 5000  

# 开启AOF同步模式

appendonly yes

````

### 创建集群

目前这些实例虽然都开启了cluster模式，但是彼此还不认识对方，接下来可以通过Redis集群的命令行工具redis-trib.rb来完成集群创建。
首先，下载 [https://raw.githubusercontent.com/antirez/redis/unstable/src/redis-trib.rb](https://raw.githubusercontent.com/antirez/redis/unstable/src/redis-trib.rb "redis-trib.rb")。

然后，搭建Redis 的 Ruby 支持环境。这里，不进行扩展，参考相关文档。

现在，接下来运行以下命令。这个命令在这里用于创建一个新的集群, 选项–replicas 1 表示我们希望为集群中的每个主节点创建一个从节点。

1.  redis-trib.rb create --replicas 1  127.0.0.1:7001  127.0.0.1:7002  127.0.0.1:7003  127.0.0.1:7004  127.0.0.1:7005  127.0.0.1:7006

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/redis3-cluster03.png)

5.3、测试
![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/redis3-cluster04.png)

*   版权声明：本文由 **梁桂钊** 发表于 [梁桂钊的博客](http://blog.720ui.com/)
*   转载声明：转载请联系公众号【服务端思维】。
*   文章标题：[Redis实战（四） 集群机制 | 梁桂钊的博客](http://blog.720ui.com/2016/redis_action_04_cluster/)
*   文章链接：[http://blog.720ui.com/2016/redis_action_04_cluster/](http://blog.720ui.com/2016/redis_action_04_cluster/)
