## [分布式服务协调员zookeeper - 应用场景和监控](https://www.cnblogs.com/bangerlee/p/4427331.html)

zookeeper在分布式系统中作为协调员的角色，可应用于Leader选举、分布式锁、配置管理等服务的实现。以下我们从zookeeper供的API、应用场景和监控三方面学习和了解zookeeper（以下简称ZK）。





**ZK API**

ZK以Unix文件系统树结构的形式管理存储的数据，图示如下：

![](https://images0.cnblogs.com/blog2015/116770/201504/142337054797921.png)

其中每个树节点被称为**znode**，每个znode类似一个文件，包含文件元信息(meta data)和数据。

以下我们用**server**表示ZK服务的提供方，**client**表示ZK服务的使用方，当client连接ZK时，相应创建**session**会话信息。

有两种类型的znode：

**Regular**: 该类型znode只能由client端显式创建或删除

**Ephemeral**: client端可创建或删除该类型znode；当session终止时，ZK亦会删除该类型znode

znode创建时还可以被打上**sequential**标志，被打上该标志的znode，将自行加上自增的数字后缀

ZK提供了以下API，供client操作znode和znode中存储的数据：

*   create(path, data, flags)：创建路径为path的znode，在其中存储data[]数据，flags可设置为Regular或Ephemeral，并可选打上sequential标志。
*   delete(path, version)：删除相应path/version的znode
*   exists(path,watch)：如果存在path对应znode，则返回true；否则返回false，watch标志可设置监听事件
*   getData(path, watch)：返回对应znode的数据和元信息（如version等）
*   setData(path, data, version)：将data[]数据写入对应path/version的znode
*   getChildren(path, watch)：返回指定znode的子节点集合

**ZK应用场景**

基于以上ZK提供的znode和znode数据的操作，可轻松实现Leader选举、分布式锁、配置管理等服务。

**Leader选举**

利用打上sequential标志的Ephemeral，我们可以实现Leader选举。假设需要从三个client中选取Leader，实现过程如下：

**1**、各自创建Ephemeral类型的znode，并打上sequential标志：



<pre>[zk: localhost:2181(CONNECTED) 4] ls /master
[lock-0000000241, lock-0000000243, lock-0000000242]</pre>



**2**、检查 /master 路径下的所有znode，如果自己创建的znode序号最小，则认为自己是Leader；否则记录序号比自己次小的znode

**3**、非Leader在次小序号znode上设置监听事件，并重复执行以上步骤2

假如以上 /master/lock-0000000241节点被删除（相应client服务异常或网络异常等原因），那么 /master/lock-0000000242相应的znode将提升自己为Leader。client只关心自己创建的znode和序号次小的znode，这避免了惊群效应(Herd Effect)。

分布式锁的实现与以上Leader选举的实现相同，稍作修改，我们还可以基于ZK实现lease机制（有期限的授权服务）。

**配置管理**

znode可以存储数据，基于这一点，我们可以用ZK实现分布式系统的配置管理，假设有服务A，A扩容设备时需要将相应新增的ip/port同步到全网服务器的A.conf配置，实现过程如下：

**1**、A扩容时，相应在ZK上新增znode，该znode数据形式如下：



<pre>[zk: localhost:2181(CONNECTED) 30] get /A/blk-0000340369 {"svr_info": [{"ip": "1.1.1.1.", "port": "11000"}]}
cZxid = 0x2ffdeda3be ……</pre>



**2**、全网机器监听 /A，当该znode下有新节点加入时，调用相应处理函数，将服务A的新增ip/port加入A.conf

**3**、完成步骤2后，继续设置对 /A监听

服务缩容的步骤类似，机器下线时将ZK相应节点删除，全网机器监听到该事件后将配置中的设备剔除。

**ZK监控**

ZK自身提供了一些“四字命令”，通过这些四字命令，我们可以获得ZK集群中，某台ZK的角色、znode数、健康状态等信息：




<pre># echo "mntr" | /usr/bin/netcat 127.0.0.1 2181 zk_version 3.4.3-1240972, built on 02/06/2012 10:48 GMT
zk_packets_received 267044485 zk_packets_sent 267069992 zk_outstanding_requests 0 zk_server_state follower
zk_znode_count 16216</pre>




常用的四字命令有：

*   **mntr**：显示自身角色、znode数、平均调用耗时、收包发包数等信息
*   **ruok**：诊断自身状态是否ok
*   **cons**：展示当前的client连接

像不能问一个醉酒的人是否喝醉一样，我们也不能确信一台回复"imok"的ZK就是真的ok，我们可以通过ZK自带的zkCli.sh模拟client创建/删除znode：



<pre>/usr/local/zookeeper/bin/zkCli.sh create /zookeeper/test 'test' >/dev/null 2>&1
/usr/local/zookeeper/bin/zkCli.sh delete /zookeeper/test >/dev/null 2>&1</pre>



再根据返回值判断添加、删除znode是否成功，从而判断该台ZK状态是否正常。

**小结**

zookeeper以目录树的形式管理数据，提供znode监听、数据设置等接口，基于这些接口，我们可以实现Leader选举、配置管理、命名服务等功能。结合四字命令，加上模拟zookeeper client 创建/删除znode，我们可以实现对zookeeper的有效监控。在各种分布式系统中，我们经常可以看到zookeeper的身影。



