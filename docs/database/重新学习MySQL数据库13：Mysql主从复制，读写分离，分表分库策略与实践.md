# 目录

* [二、分表实现策略](#二、分表实现策略)
* [三、分库实现策略](#三、分库实现策略)
* [四、分库与分表实现策略](#四、分库与分表实现策略)
* [五、分库分表总结](#五、分库分表总结)
* [六、总结](#六、总结)
* [Mycat实现主从复制，读写分离，以及分库分表的实践](#mycat实现主从复制，读写分离，以及分库分表的实践)
    * [Mycat是什么](#mycat是什么)
    * [一、分区分表](#一、分区分表)
    * [二、Mycat 数据分片的种类](#二、mycat-数据分片的种类)
    * [三、Mycat 垂直切分、水平切分实战](#三、mycat-垂直切分、水平切分实战)
        * [1、垂直切分](#1、垂直切分)
        * [2、水平切分](#2、水平切分)
* [range start-end ,data node index](#range-start-end-data-node-index)
* [K=1000,M=10000.](#k1000m10000)
  * [为什么需要读写分离](#为什么需要读写分离)
  * [MySQL主从复制](#mysql主从复制)
  * [Mycat读写分离设置](#mycat读写分离设置)
  * [配置Mycat用户](#配置mycat用户)
  * [配置Mycat逻辑库](#配置mycat逻辑库)
  * [schema](#schema)
  * [dataNode](#datanode)
  * [dataHost](#datahost)


本文转自互联网

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本也将整理到我的个人博客：

> www.how2playlife.com

更多Java技术文章将陆续在微信公众号【Java技术江湖】更新，敬请关注。

本文是《重新学习MySQL数据库》系列其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从sql基本的使用方法，从MySQL执行引擎再到索引、事务等知识，一步步地学习MySQL相关技术的实现原理，更好地了解如何基于这些知识来优化sql，减少SQL执行时间，通过执行计划对SQL性能进行分析，再到MySQL的主从复制、主备部署等内容，以便让你更完整地了解整个MySQL方面的技术体系，形成自己的知识框架。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

一、MySQL扩展具体的实现方式

随着业务规模的不断扩大，需要选择合适的方案去应对数据规模的增长，以应对逐渐增长的访问压力和数据量。

关于数据库的扩展主要包括：业务拆分、主从复制、读写分离、数据库分库与分表等。这篇文章主要讲述数据库分库与分表

（1）业务拆分

在[大型网站应用之海量数据和高并发解决方案总结一二](http://blog.csdn.net/xlgen157387/article/details/53230138)一篇文章中也具体讲述了为什么要对业务进行拆分。

业务起步初始，为了加快应用上线和快速迭代，很多应用都采用集中式的架构。随着业务系统的扩大，系统变得越来越复杂，越来越难以维护，开发效率变得越来越低，并且对资源的消耗也变得越来越大，通过硬件提高系统性能的方式带来的成本也越来越高。

因此，在选型初期，一个优良的架构设计是后期系统进行扩展的重要保障。

例如：电商平台，包含了用户、商品、评价、订单等几大模块，最简单的做法就是在一个数据库中分别创建users、shops、comment、order四张表。

![这里写图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171414385-1549042254.png)

但是，随着业务规模的增大，访问量的增大，我们不得不对业务进行拆分。每一个模块都使用单独的数据库来进行存储，不同的业务访问不同的数据库，将原本对一个数据库的依赖拆分为对4个数据库的依赖，这样的话就变成了4个数据库同时承担压力，系统的吞吐量自然就提高了。

![这里写图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171414592-1710110308.png)

（2）主从复制

一般是主写从读，一主多从

1、[MySQL5.6 数据库主从（Master/Slave）同步安装与配置详解](http://blog.csdn.net/xlgen157387/article/details/51331244)

2、[MySQL主从复制的常见拓扑、原理分析以及如何提高主从复制的效率总结](http://blog.csdn.net/xlgen157387/article/details/52451613)

3、[使用mysqlreplicate命令快速搭建 Mysql 主从复制](http://blog.csdn.net/xlgen157387/article/details/52452394)

上述三篇文章中，讲述了如何配置主从数据库，以及如何实现数据库的读写分离，这里不再赘述，有需要的选择性点击查看。

![这里写图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171415114-995239105.png)

上图是网上的一张关于MySQL的Master和Slave之间数据同步的过程图。

主要讲述了MySQL主从复制的原理：数据复制的实际就是Slave从Master获取Binary log文件，然后再本地镜像的执行日志中记录的操作。由于主从复制的过程是异步的，因此Slave和Master之间的数据有可能存在延迟的现象，此时只能保证数据最终的一致性。

（3）数据库分库与分表

我们知道每台机器无论配置多么好它都有自身的物理上限，所以当我们应用已经能触及或远远超出单台机器的某个上限的时候，我们惟有寻找别的机器的帮助或者继续升级的我们的硬件，但常见的方案还是通过添加更多的机器来共同承担压力。

我们还得考虑当我们的业务逻辑不断增长，我们的机器能不能通过线性增长就能满足需求？因此，使用数据库的分库分表，能够立竿见影的提升系统的性能，关于为什么要使用数据库的分库分表的其他原因这里不再赘述，主要讲具体的实现策略。请看下边章节。

## 二、分表实现策略

关键字：用户ID、表容量

对于大部分数据库的设计和业务的操作基本都与用户的ID相关，因此使用用户ID是最常用的分库的路由策略。用户的ID可以作为贯穿整个系统用的重要字段。因此，使用用户的ID我们不仅可以方便我们的查询，还可以将数据平均的分配到不同的数据库中。（当然，还可以根据类别等进行分表操作，分表的路由策略还有很多方式）

接着上述电商平台假设，订单表order存放用户的订单数据，sql脚本如下（只是为了演示，省略部分细节）：

```
CREATE TABLE `order` (
  `order_id` bigint(32) primary key auto_increment,
  `user_id` bigint(32),
   ...
) 
```

当数据比较大的时候，对数据进行分表操作，首先要确定需要将数据平均分配到多少张表中，也就是：表容量。

这里假设有100张表进行存储，则我们在进行存储数据的时候，首先对用户ID进行取模操作，根据`user_id%100`获取对应的表进行存储查询操作，示意图如下：

![这里写图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171415339-1589002733.png)

例如，`user_id = 101`那么，我们在获取值的时候的操作，可以通过下边的sql语句：

```
select * from order_1 where user_id= 101
```

其中，`order_1`是根据`101%100`计算所得，表示分表之后的第一章order表。

注意：

在实际的开发中，如果你使用MyBatis做持久层的话，MyBatis已经提供了很好得支持数据库分表的功能，例如上述sql用MyBatis实现的话应该是：

接口定义：

```
/**
  * 获取用户相关的订单详细信息
  * @param tableNum 具体某一个表的编号
  * @param userId 用户ID
  * @return 订单列表
  */
public List<Order> getOrder(@Param("tableNum") int tableNum,@Param("userId") int userId);
```

xml配置映射文件：

```
<select id="getOrder" resultMap="BaseResultMap">
    select * from order_${tableNum}
    where user_id = #{userId}
  </select>
```

其中`${tableNum}`含义是直接让参数加入到sql中，这是MyBatis支持的特性。

注意：

```
另外，在实际的开发中，我们的用户ID更多的可能是通过UUID生成的，这样的话，我们可以首先将UUID进行hash获取到整数值，然后在进行取模操作。
```

## 三、分库实现策略

数据库分表能够解决单表数据量很大的时候数据查询的效率问题，但是无法给数据库的并发操作带来效率上的提高，因为分表的实质还是在一个数据库上进行的操作，很容易受数据库IO性能的限制。

因此，如何将数据库IO性能的问题平均分配出来，很显然将数据进行分库操作可以很好地解决单台数据库的性能问题。

分库策略与分表策略的实现很相似，最简单的都是可以通过取模的方式进行路由。

还是上例，将用户ID进行取模操作，这样的话获取到具体的某一个数据库，同样关键字有：

用户ID、库容量

路由的示意图如下：

![这里写图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171415519-516549009.png)

上图中库容量为100。

同样，如果用户ID为UUID请先hash然后在进行取模。

## 四、分库与分表实现策略

上述的配置中，数据库分表可以解决单表海量数据的查询性能问题，分库可以解决单台数据库的并发访问压力问题。

有时候，我们需要同时考虑这两个问题，因此，我们既需要对单表进行分表操作，还需要进行分库操作，以便同时扩展系统的并发处理能力和提升单表的查询性能，就是我们使用到的分库分表。

分库分表的策略相对于前边两种复杂一些，一种常见的路由策略如下：

```
１、中间变量　＝ user_id%（库数量*每个库的表数量）;
２、库序号　＝　取整（中间变量／每个库的表数量）;
３、表序号　＝　中间变量％每个库的表数量;
```

例如：数据库有256 个，每一个库中有1024个数据表，用户的user_id＝262145，按照上述的路由策略，可得：

```
１、中间变量　＝ 262145%（256*1024）= 1;
２、库序号　＝　取整（1／1024）= 0;
３、表序号　＝　1％1024 = 1;
```

这样的话，对于user_id＝262145，将被路由到第０个数据库的第１个表中。

示意图如下：

![这里写图片描述](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171416563-1109228018.png)

## 五、分库分表总结

关于分库分表策略的选择有很多种，上文中根据用户ID应该是比较简单的一种。其他方式比如使用号段进行分区或者直接使用hash进行路由等。有兴趣的可以自行查找学习。

关于上文中提到的，如果用户的ID是通过UUID的方式生成的话，我们需要单独的进行一次hash操作，然后在进行取模操作等，其实hash本身就是一种分库分表的策略，使用hash进行路由策略的时候，我们需要知道的是，也就是hash路由策略的优缺点，优点是：数据分布均匀；缺点是：数据迁移的时候麻烦，不能按照机器性能分摊数据。

上述的分库和分表操作，查询性能和并发能力都得到了提高，但是还有一些需要注意的就是，例如：原本跨表的事物变成了分布式事物；由于记录被切分到不同的数据库和不同的数据表中，难以进行多表关联查询，并且不能不指定路由字段对数据进行查询。分库分表之后，如果我们需要对系统进行进一步的扩阵容（路由策略变更），将变得非常不方便，需要我们重新进行数据迁移。

* * *

最后需要指出的是，分库分表目前有很多的中间件可供选择，最常见的是使用淘宝的中间件Cobar。

GitHub地址：[https://github.com/alibaba/cobara](https://github.com/alibaba/cobara)

文档地址为：[https://github.com/alibaba/cobar/wiki](https://github.com/alibaba/cobar/wiki)

关于淘宝的中间件Cobar本篇内容不具体介绍，会在后边的学习中在做介绍。

另外Spring也可以实现数据库的读写分离操作，后边的文章，会进一步学习。

## 六、总结

上述中，我们学到了如何进行数据库的读写分离和分库分表，那么，是不是可以实现一个可扩展、高性能、高并发的网站那？很显然还不可以!一个大型的网站使用到的技术远不止这些，可以说，这些都是其中的最基础的一个环节，因为还有很多具体的细节我们没有掌握到，比如：数据库的集群控制，集群的负载均衡，灾难恢复，故障自动切换，事务管理等等技术。因此，还有很多需要去学习去研究的地方。

总之：

```
路漫漫其修远兮，吾将上下而求索。
```

前方道路美好而光明，2017年新征程，不泄步！

## Mycat实现主从复制，读写分离，以及分库分表的实践

### Mycat是什么

一个彻底开源的，面向企业应用开发的大数据库集群

支持事务、ACID、可以替代MySQL的加强版数据库

一个可以视为MySQL集群的企业级数据库，用来替代昂贵的Oracle集群

一个融合内存缓存技术、NoSQL技术、HDFS大数据的新型SQL Server

结合传统数据库和新型分布式数据仓库的新一代企业级数据库产品

一个新颖的数据库中间件产品

以上内容来自[Mycat官网](http://www.mycat.io/)，简单来说，Mycat就是一个数据库中间件，对于我们开发来说，就像是一个代理，当我们需要使用到多个数据库和需要进行分库分表的时候，我们只需要在mycat里面配置好相关规则，程序无需做任何修改，只是需要将原本的数据源链接到mycat而已，当然如果以前有多个数据源，需要将数据源切换为单个数据源，这样有个好处就是当我们的数据量已经很大的时候，需要开始分库分表或者做读写分离的时候，不用修改代码（只需要改一下数据源的链接地址）

**使用Mycat分表分库实践**

haha,首先这不是一篇入门Mycat的博客但小编感觉又很入门的博客!这篇博客主要讲解Mycat中数据分片的相关知识，同时小编将会在本机数据库上进行测试验证，图文并茂展示出来。

数据库分区分表，咋一听非常地高大上，总有一种高高在上，望尘莫及的感觉，但小编想说的是，其实，作为一个开发人员，该来的总是会来，该学的东西你还是得学，区别只是时间先后顺序的问题。

### 一、分区分表

分区就是把一个数据表的文件和索引分散存储在不同的物理文件中。

mysql支持的分区类型包括Range、List、Hash、Key，其中Range比较常用：

RANGE分区：基于属于一个给定连续区间的列值，把多行分配给分区。

LIST分区：类似于按RANGE分区，区别在于LIST分区是基于列值匹配一个离散值集合中的某个值来进行选择。

HASH分区：基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL 中有效的、产生非负整数值的任何表达式。

KEY分区：类似于按HASH分区，区别在于KEY分区只支持计算一列或多列，且MySQL服务器提供其自身的哈希函数。必须有一列或多列包含整数值。

分表是指在逻辑上将一个表拆分成多个逻辑表，在整体上看是一张表，分表有水平拆分和垂直拆分两种,举个例子，将一张大的存储商户信息的表按照商户号的范围进行分表，将不同范围的记录分布到不同的表中。

### 二、Mycat 数据分片的种类

Mycat 的分片其实和分表差不多意思，就是当数据库过于庞大，尤其是写入过于频繁且很难由一台主机支撑是，这时数据库就会面临瓶颈。我们将存放在同一个数据库实例中的数据分散存放到多个数据库实例（主机）上，进行多台设备存取以提高性能，在切分数据的同时可以提高系统的整体性。

数据分片是指将数据全局地划分为相关的逻辑片段，有水平切分、垂直切分、混合切分三种类型，下面主要讲下Mycat的水平和垂直切分。有一点很重要，那就是Mycat是分布式的，因此分出来的数据片分布到不同的物理机上是正常的，靠网络通信进行协作。

水平切分

就是按照某个字段的某种规则分散到多个节点库中，每个节点中包含一部分数据。可以将数据水平切分简单理解为按照数据行进行切分，就是将表中的某些行切分到一个节点，将另外某些行切分到其他节点，从分布式的整体来看它们是一个整体的表。

垂直切分

一个数据库由很多表构成，每个表对应不同的业务，垂直切分是指按照业务将表进行分类并分不到不同的节点上。垂直拆分简单明了，拆分规则明确，应用程序模块清晰、明确、容易整合，但是某个表的数据量达到一定程度后扩展起来比较困难。

混合切分

为水平切分和垂直切分的结合。

### 三、Mycat 垂直切分、水平切分实战

#### 1、垂直切分

上面说到，垂直切分主要是根据具体业务来进行拆分的，那么，我们可以想象这么一个场景，假设我们有一个非常大的电商系统，那么我们需要将订单表、流水表、用户表、用户评论表等分别分不到不同的数据库中来提高吞吐量，架构图大概如下：



![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171417040-1009330909.webp)


由于小编是在一台机器上测试，因此就只有host1这个节点，但不同的表还是依旧对应不同的数据库，只不过是所有数据库属于同一个数据库实例（主机）而已，后期不同主机只需增加`<dataHost>`节点即可。

mycat配置文件如下：

`server.xml`

```
<user name="root">
    <property name="password">root</property>
    // 对应四个逻辑库
    <property name="schemas">order,trade,user,comment</property>
</user>

```

`schema.xml`

```
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

    <!-- 4个逻辑库，对应4个不同的分片节点 -->
    <schema name="order" checkSQLschema="false" sqlMaxLimit="100" dataNode="database1" />
    <schema name="trade" checkSQLschema="false" sqlMaxLimit="100" dataNode="database2" />
    <schema name="user" checkSQLschema="false" sqlMaxLimit="100" dataNode="database3" />
    <schema name="comment" checkSQLschema="false" sqlMaxLimit="100" dataNode="database4" />

    <!-- 四个分片，对应四个不同的数据库 -->
    <dataNode name="database1" dataHost="localhost1" database="database1" />
    <dataNode name="database2" dataHost="localhost1" database="database2" />
    <dataNode name="database3" dataHost="localhost1" database="database3" />
    <dataNode name="database4" dataHost="localhost1" database="database4" />

    <!-- 实际物理主机，只有这一台 -->
    <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
                writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <writeHost host="hostM1" url="localhost:3306" user="root"
                password="root">
        </writeHost>
    </dataHost>
</mycat:schema>

```

登陆本机mysql，创建`order,trade,user,comment`4个数据库:

```
create database database1 character set utf8;
create database database2 character set utf8;
create database database3 character set utf8;
create database database4 character set utf8;

```

执行`bin`目录下的`startup_nowrap.bat`文件，如果输出下面内容，则说明已经启动mycat成功，如果没有，请检查`order,trade,user,comment`4个数据库是否已经创建。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171417782-1932957742.webp)


采用下面语句登陆Mycat服务器：

`mysql -uroot -proot -P8066 -h127.0.0.1`

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171418203-391358451.webp)


在`comment`数据库中创建`Comment`表，并插入一条数据

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1092007-20190824171418351-1853467698.webp)


上图1处新建一个`Comment`表，2处插入一条记录，3处查看记录插入到哪个数据节点中，即`database4`。

#### 2、水平切分

`server.xml`

```
<user name="root">
    <property name="password">root</property>
    <property name="schemas">TESTDB</property>
</user>

```

`schema.xml`

```
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
    <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100">
        <table name="travelrecord" dataNode="dn1,dn2,dn3" rule="auto-sharding-long" />
    </schema>

    <dataNode name="dn1" dataHost="localhost1" database="db1" />
    <dataNode name="dn2" dataHost="localhost1" database="db2" />
    <dataNode name="dn3" dataHost="localhost1" database="db3" />

    <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
                writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
    <heartbeat>select user()</heartbeat>
    <!-- can have multi write hosts -->
    <writeHost host="hostM1" url="localhost:3306" user="root"
       password="root">
    </writeHost>
    </dataHost>
</mycat:schema>

```

`rule.xml`

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
    <tableRule name="auto-sharding-long">
        <rule>
            <columns>id</columns>
            rang-long
        </rule>
    </tableRule>

    <function name="rang-long"
            class="io.mycat.route.function.AutoPartitionByLong">
        <property name="mapFile">autopartition-long.txt</property>
    </function>
</mycat:rule>

```

`conf`目录下的`autopartition-long.txt`

```
# range start-end ,data node index
# K=1000,M=10000.
0-500M=0
500M-1000M=1
1000M-1500M=2

```

上面的配置创建了一个名为`TESTDB`的逻辑库，并指定了需要切分的表`<table>`标签，表名为`travelrecord`,分区的策略采用`rang-long`算法，即根据`id`数据列值的范围进行切分，具体的规则在`autopartition-long.txt`文件中定义，即`id`在`0-500*10000`范围内的记录存放在`db1`的`travelrecord`表中，`id`在`500*10000 - 1000*10000`范围内的记录存放在`db2`数据库的`travelrecord`表中，下面我们插入两条数据，验证是否和分片规则一致。

创建`db1,db2,db3`数据库

```
create database db1 character set utf8;
create database db2 character set utf8;
create database db3 character set utf8;

```

确实是这样的，到此我们就完成了mycat数据库的水平切分，这个例子只是演示按照id列值得范围进行切分，mycat还支持很多的分片算法，如取模、一致性哈希算法、按日期分片算法等等，大家可以看《分布式数据库架构及企业实战----基于Mycat中间件》这本书深入学习。

#### 为什么需要读写分离

至于为什么需要读写分离，在我之前的文章有介绍过了，相信看到这篇文章的人也知道为什么需要读写分离了，当然如果你也需要了解一下，那么欢迎查看我之前的文章[SpringBoot Mybatis 读写分离配置](http://raye.wang/2018/02/03/springboot-mybatis-du-xie-fen-chi-pei-zhi/),顺便也可以了解一下怎么通过代码进行读写分离的

#### MySQL主从复制

主从复制是读写分离的关键，不管通过什么方式进行读写分离，前提就是MySQL有主从复制，当前双机主从也行，但是关键的关键，是要能保证2个库的数据能一致（出掉刚写入主库从库还未能及时反应过来的情况），如果2个库的数据不一致，那么读写分离也有没有任何意义了，具体MySQL怎么做主从复制可以查看我之前的文章[MySQL主从复制搭建，基于日志（binlog）](http://raye.wang/2017/04/14/mysqlzhu-cong-fu-zhi-da-jian-ji-yu-ri-zhi-binlog/)

#### Mycat读写分离设置

##### 配置Mycat用户

Mycat的用户就跟MySQL用户是同一个意思，主要配置链接到Mycat的用户名以及密码，以及能使用的逻辑库，用户信息主要在server.xml中配置的，具体如下

```
<?xml version="1.0" encoding="UTF-8"?>  
<!-- - - Licensed under the Apache License, Version 2.0 (the "License");  
    - you may not use this file except in compliance with the License. - You 
    may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0 
    - - Unless required by applicable law or agreed to in writing, software - 
    distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT 
    WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the 
    License for the specific language governing permissions and - limitations 
    under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">  
<mycat:server xmlns:mycat="http://io.mycat/">  
    <system>
    <property name="defaultSqlParser">druidparser</property>
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
    <!-- <property name="processorBufferChunk">40960</property> -->
    <!-- 
    <property name="processors">1</property> 
    <property name="processorExecutor">32</property> 
     -->
        <!--默认是65535 64K 用于sql解析时最大文本长度 -->
        <!--<property name="maxStringLiteralLength">65535</property>-->
        <!--<property name="sequnceHandlerType">0</property>-->
        <!--<property name="backSocketNoDelay">1</property>-->
        <!--<property name="frontSocketNoDelay">1</property>-->
        <!--<property name="processorExecutor">16</property>-->
        <!-- 
            <property name="mutiNodeLimitType">1</property> 0：开启小数量级（默认） ；1：开启亿级数据排序
            <property name="mutiNodePatchSize">100</property> 亿级数量排序批量
            <property name="processors">32</property> <property name="processorExecutor">32</property> 
            <property name="serverPort">8066</property> <property name="managerPort">9066</property> 
            <property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property> 
            <property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
    </system>
    <user name="raye">
        <property name="password">rayewang</property>
        <property name="schemas">separate</property>
    </user>

        </host> 
</mycat:server>  

```

其中`<user name="raye">`定义了一个名为raye的用户，标签user中的`<property name="password">rayewang</property>`定义了用户的密码，`<property name="schemas">separate</property>`定义了用户可以使用的逻辑库

##### 配置Mycat逻辑库

Mycat的配置有很多，不过因为我们只是使用Mycat的读写分类的功能，所以用到的配置并不多，只需要配置一些基本的，当然本文也只是会介绍到读写分离相关的配置，其他配置建议读者自己查看一下文档，或者通过其他方式了解，逻辑库是在`schema.xml`中配置的

首先介绍Mycat逻辑库中的一些配置标签

###### schema

`schema`标签是用来定义逻辑库的，`schema`有四个属性`dataNode`,`checkSQLschema`,`sqlMaxLimit`,`name`

`dataNode`标签属性用于绑定逻辑库到某个具体的 database 上，1.3 版本如果配置了 dataNode，则不可以配置分片表，1.4 可以配置默认分片，只需要配置需要分片的表即可

`name`是定义当前逻辑库的名字的，方便`server.xml`中定义用户时的引用

`checkSQLschema`当该值设置为 true 时，如果我们执行语句select * from separate.users;则 MyCat 会把语句修改 为select * from users;。即把表示 schema 的字符去掉，避免发送到后端数据库执行时报（ERROR 1146 (42S02): Table ‘separate.users’ doesn’t exist）。不过，即使设置该值为 true ，如果语句所带的是并非是 schema 指定的名字，例如：select * from db1.users;那么 MyCat 并不会删除 db1 这个字段，如果没有定义该库的话则会报错，所以在提供 SQL语句的最好是不带这个字段。

`sqlMaxLimit`当该值设置为某个数值时。每条执行的 SQL 语句，如果没有加上 limit 语句，MyCat 也会自动的加上所对应的值。例如设置值为 100，执行select * from users;的效果为和执行select * from users limit 100;相同。设置该值的话，MyCat 默认会把查询到的信息全部都展示出来，造成过多的输出。所以，在正常使用中，还是建议加上一个值，用于减少过多的数据返回。当然 SQL 语句中也显式的指定 limit 的大小，不受该属性的约束。需要注意的是，如果运行的 schema 为非拆分库的，那么该属性不会生效。需要手动添加 limit 语句。

`schema`标签中有标签`table`用于定义不同的表分片信息，不过我们只是做读写分离，并不会用到，所以这里就不多介绍了

###### dataNode

`dataNode`dataNode 标签定义了 MyCat 中的数据节点，也就是我们通常说所的数据分片。一个 dataNode 标签就是一个独立的数据分片,`dataNode`有3个属性:`name`,`dataHost`,`database`。

`name`定义数据节点的名字，这个名字需要是唯一的，此名字是用于`table`标签和`schema`标签中引用的

`dataHost`该属性用于定义该分片属于哪个数据库实例的，属性值是引用 dataHost 标签上定义的 name 属性

`database`该属性用于定义该分片属性哪个具体数据库实例上的具体库，因为这里使用两个纬度来定义分片，就是：实例+具体的库。因为每个库上建立的表和表结构是一样的。所以这样做就可以轻松的对表进行水平拆分

###### dataHost

`dataHost`是定义真实的数据库连接的标签，该标签在 mycat 逻辑库中也是作为最底层的标签存在，直接定义了具体的数据库实例、读写分离配置和心跳语句，`dataHost`有7个属性：`name`,`maxCon`,`minCon`,`balance`,`writeType`,`dbType`,`dbDriver`,有2个标签`heartbeat`,`writeHost`,其中`writeHost`标签中又包含一个`readHost`标签

`name`唯一标识 dataHost 标签，供`dataNode`标签使用

`maxCon`指定每个读写实例连接池的最大连接。也就是说，标签内嵌套的 writeHost、readHost 标签都会使用这个属性的值来实例化出连接池的最大连接数

`minCon`指定每个读写实例连接池的最小连接，初始化连接池的大小

`balance`读取负载均衡类型

1.  balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的 writeHost 上。

2.  balance="1"，全部的 readHost 与 stand by writeHost 参与 select 语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且 M1 与 M2 互为主备)，正常情况下，M2,S1,S2 都参与 select 语句的负载均衡。

3.  balance="2"，所有读操作都随机的在 writeHost、readhost 上分发。

4.  balance="3"，所有读请求随机的分发到 wiriterHost 对应的 readhost 执行，writerHost 不负担读压力

`writeType`写入负载均衡类型，目前的取值有 3 种：

1.  writeType="0", 所有写操作发送到配置的第一个 writeHost，第一个挂了切到还生存的第二个writeHost，重新启动后已切换后的为准，切换记录在配置文件中:dnindex.properties .

2.  writeType="1"，所有写操作都随机的发送到配置的 writeHost

`dbType`指定后端连接的数据库类型，目前支持二进制的 mysql 协议，还有其他使用 JDBC 连接的数据库。例如：mongodb、oracle、spark 等

`dbDriver`指定连接后端数据库使用的 Driver，目前可选的值有 native 和 JDBC。使用 native 的话，因为这个值执行的 是二进制的 mysql 协议，所以可以使用 mysql 和 maridb。其他类型的数据库则需要使用 JDBC 驱动来支持。从 1.6 版本开始支持 postgresql 的 native 原始协议。 如果使用 JDBC 的话需要将符合 JDBC 4 标准的驱动 JAR 包放到 MYCAT\lib 目录下，并检查驱动 JAR 包中包括如下目录结构的文件：META-INF\services\java.sql.Driver。在这个文件内写上具体的 Driver 类名，例如： com.mysql.jdbc.Driver。

`heartbeat`这个标签内指明用于和后端数据库进行心跳检查的语句。例如,MYSQL 可以使用 select user()，Oracle 可以使用 select 1 from dual 等。 这个标签还有一个 connectionInitSql 属性，主要是当使用 Oracla 数据库时，需要执行的初始化 SQL 语句就这个放到这里面来。例如：alter session set nlsdateformat='yyyy-mm-dd hh24:mi:ss'

`writeHost`，`readHost`这两个标签都指定后端数据库的相关配置给 mycat，用于实例化后端连接池。唯一不同的是，writeHost 指定写实例、readHost 指定读实例，组着这些读写实例来满足系统的要求。 在一个 dataHost 内可以定义多个 writeHost 和 readHost。但是，如果 writeHost 指定的后端数据库宕机，那么这个 writeHost 绑定的所有 readHost 都将不可用。另一方面，由于这个 writeHost 宕机系统会自动的检测到，并切换到备用的 writeHost 上去,这2个标签属性都一致，拥有`host`,`url`,`password`,`user`,`weight`,`usingDecrypt`等属性

`host`用于标识不同实例，一般 writeHost 我们使用M1，readHost 我们用S1

`url`真实数据库的实例的链接地址，如果是使用 native 的 dbDriver，则一般为 address:port 这种形式。用 JDBC 或其他的dbDriver，则需要特殊指定。当使用 JDBC 时则可以这么写：jdbc:mysql://localhost:3306/

`user`真实数据库实例的链接用户名

`password`真实数据库实例的链接密码

`weight`权重 配置在 readhost 中作为读节点的权重,主要用于多台读取的数据库实例机器配置不同的情况，可以根据权重调整访问量

`usingDecrypt`是否对密码加密默认 0 否 如需要开启配置 1，同时使用加密程序对密码加密

注意，readHost是在writeHost标签内的，不是单独的

以下是我的读写分离配置文件

```
<?xml version="1.0"?>  
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">  
<mycat:schema xmlns:mycat="http://io.mycat/">

    <schema name="separate" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn1"/>
    <dataNode name="dn1" dataHost="localhost1" database="test" />

    <dataHost name="localhost1" maxCon="1000" minCon="10" balance="3"
              writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
        <heartbeat>select user()</heartbeat>
        <!-- can have multi write hosts -->
        <writeHost host="hostM1" url="192.168.1.126:3307" user="root"
                   password="123456">
            <!-- can have multi read hosts -->
            <readHost host="hostS2" url="192.168.1.126:3308" user="root" password="123456" />
        </writeHost>

    </dataHost>

</mycat:schema>  

```

前面已经差不多都解释清楚了，因为我只是用的基本的主从复制，所以我的将`dataHost`的`balance`设置成了3

启动mycat，然后用数据库连接工具连接到mycat，可以测试是否配置成功，最简单的就是通过修改从库的数据，这样方便查看到底是运行到哪个库上面了，另外由于我是基于docker启动的mycat，所以如果是直接在系统中运行的mycat的，可以去看官方文档，看看到底怎么启动mycat