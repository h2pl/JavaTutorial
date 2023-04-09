# 目录
* [数据库原理](#数据库原理)
  * [范式 反范式](#范式-反范式)
  * [主键 外键](#主键-外键)
  * [锁 共享锁和排它锁](#锁-共享锁和排它锁)
  * [存储过程与视图](#存储过程与视图)
  * [事务与隔离级别](#事务与隔离级别)
  * [索引](#索引)
* [mysql原理](#mysql原理)
  * [mysql客户端，服务端，存储引擎，文件系统](#mysql客户端，服务端，存储引擎，文件系统)
  * [mysql常用语法](#mysql常用语法)
  * [MySQL的存储原理](#mysql的存储原理)
    * [数据页page](#数据页page)
  * [mysql的索引，b树，聚集索引](#mysql的索引，b树，聚集索引)
  * [mysql的explain 慢查询日志](#mysql的explain-慢查询日志)
  * [mysql的binlog,redo log和undo log。](#mysql的binlogredo-log和undo-log。)
  * [mysql的数据类型](#mysql的数据类型)
  * [mysql的sql优化。](#mysql的sql优化。)
  * [MySQL的事务实现和锁](#mysql的事务实现和锁)
  * [分库分表](#分库分表)
  * [主从复制，读写分离](#主从复制，读写分离)
  * [分布式数据库](#分布式数据库)



本文根据自己对MySQL的学习和实践以及各类文章与书籍总结而来。
囊括了MySQL数据库的基本原理和技术。本文主要是我的一个学习总结，基于之前的系列文章做了一个概括，如有错误，还望指出，谢谢。

详细内容请参考我的系列文章：

重新学习MySQL与Redis

https://blog.csdn.net/column/details/21877.html
<!-- more -->

# 数据库原理
Mysql是关系数据库。

## 范式 反范式
范式设计主要是避免冗余，以及数据不一致。反范式设计主要是避免多表连接，增加了冗余。

## 主键 外键
主键是一个表中一行数据的唯一标识。
外键则是值某一列的键值是其他表的主键，外键的作用一般用来作为两表连接的键，并且保证数据的一致性。

## 锁 共享锁和排它锁
数据库的锁用来进行并发控制，排它锁也叫写锁，共享锁也叫行锁，根据不同粒度可以分为行锁和表锁。

## 存储过程与视图
存储过程是对sql语句进行预编译并且以文件形式包装为一个可以快速执行的程序。但是缺点是不易修改，稍微改动语句就需要重新开发储存过程，优点是执行效率快。视图就是对其他一个或多个表进行重新包装，是一个外观模式，对视图数据的改动也会影响到数据报本身。

## 事务与隔离级别   
事务的四个性质：原子性，一致性，持久性，隔离性。

原子性：一个事务中的操作要么全部成功要么全部失败。

一致性：事务执行成功的状态都是一致的，即使失败回滚了，也应该和事务执行前的状态是一致的。

隔离性：两个事务之间互不相干，不能互相影响。

事务的隔离级别
读未提交：事务A和事务B，A事务中执行的操作，B也可以看得到，因为级别是未提交读，别人事务中还没提交的数据你也看得到。这是没有任何并发措施的级别，也是默认级别。这个问题叫做脏读，为了解决这个问题，提出了读已提交。

读已提交：事务A和B，A中的操作B看不到，只有A提交后，在B中才看得到。虽然A的操作B看不到，但是B可以修改A用到的数据，导致A读两次的数据结果不同。这就是不可重读问题。

可重复读：事务A和B，事务A和B，A在数据行上加读锁，B虽然看得到但是改不了。所以是可重复读的，但是A的其他行仍然会被B访问并修改，所以导致了幻读问题。

序列化：数据库强制事务A和B串行化操作，避免了并发问题，但是效率比较低。

后面可以看一下mysql对隔离级别的实现。

## 索引

索引的作用就和书的目录类似，比如根据书名做索引，然后我们通过书名就可以直接翻到某一页。数据表中我们要找一条数据，也可以根据它的主键来找到对应的那一页。当然数据库的搜索不是翻书，如果一页一页翻书，就相当于是全表扫描了，效率很低，所以人翻书肯定也是跳着翻。数据库也会基于类似的原理"跳着”翻书，快速地找到索引行。

# mysql原理

MySQL是oracle公司的免费数据库，作为关系数据库火了很久了。所以我们要学他。

## mysql客户端，服务端，存储引擎，文件系统

MySQL数据库的架构可以分为客户端，服务端，存储引擎和文件系统。

详细可以看下架构图，我稍微总结下

    最高层的客户端，通过tcp连接mysql的服务器，然后执行sql语句，其中涉及了查询缓存，执行计划处理和优化，接下来再到存储引擎层执行查询，底层实际上访问的是主机的文件系统。

![image](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/39b96aa41090f9bb.png)
## mysql常用语法

1 登录mysql

mysql -h 127.0.0.1 -u 用户名 -p

2 创建表
语法还是比较复杂的，之前有腾讯面试官问这个，然后答不上来。
````    
    CREATE TABLE `user_accounts` (
      `id`             int(100) unsigned NOT NULL AUTO_INCREMENT primary key,
      `password`       varchar(32)       NOT NULL DEFAULT '' COMMENT '用户密码',
      `reset_password` tinyint(32)       NOT NULL DEFAULT 0 COMMENT '用户类型：0－不需要重置密码；1-需要重置密码',
      `mobile`         varchar(20)       NOT NULL DEFAULT '' COMMENT '手机',
      `create_at`      timestamp(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6),
      `update_at`      timestamp(6)      NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
      -- 创建唯一索引，不允许重复
      UNIQUE INDEX idx_user_mobile(`mobile`)
    )
    ENGINE=InnoDB DEFAULT CHARSET=utf8
````

3 crud比较简单，不谈

4 join用于多表连接，查询的通常是两个表的字段。

union用于组合同一种格式的多个select查询。

6 聚合函数，一般和group by一起使用，比如查找某部门员工的工资平均值。
就是select AVE(money) from departmentA group by department

7 建立索引


唯一索引(UNIQUE)
语法：ALTER TABLE 表名字 ADD UNIQUE (字段名字)

添加多列索引
语法：

ALTER TABLE table_name ADD INDEX index_name ( column1, column2, column3)

8 修改添加列

添加列
语法：alter table 表名 add 列名 列数据类型 [after 插入位置];

删除列
语法：alter table 表名 drop 列名称;

9 清空表数据
方法一：delete from 表名;
方法二：truncate from "表名";

DELETE:1. DML语言;2. 可以回退;3. 可以有条件的删除;

TRUNCATE:1. DDL语言;2. 无法回退;3. 默认所有的表内容都删除;4. 删除速度比delete快。

## MySQL的存储原理

下面我们讨论的是innodb的存储原理

innodb的存储引擎将数据存储单元分为多层。按此不表

MySQL中的逻辑数据库只是一个shchme。事实上物理数据库只有一个。

mysql使用两个文件分别存储数据库的元数据和数据库的真正数据。
### 数据页page

数据页结构
页是 InnoDB 存储引擎管理数据的最小磁盘单位，而 B-Tree 节点就是实际存放表中数据的页面，我们在这里将要介绍页是如何组织和存储记录的；首先，一个 InnoDB 页有以下七个部分：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405195759.png)
每一个页中包含了两对 header/trailer：内部的 Page Header/Page Directory 关心的是页的状态信息，而 Fil Header/Fil Trailer 关心的是记录页的头信息。

    也就是说，外部的h-t对用来和其他页形成联系，而内部的h-t用来是保存内部记录的状态。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405195900.png)
User Records 就是整个页面中真正用于存放行记录的部分，而 Free Space 就是空余空间了，它是一个链表的数据结构，为了保证插入和删除的效率，整个页面并不会按照主键顺序对所有记录进行排序，它会自动从左侧向右寻找空白节点进行插入，行记录在物理存储上并不是按照顺序的，它们之间的顺序是由 next_record 这一指针控制的。
    
    也就是说，一个页中存了非常多行的数据，而每一行数据和相邻行使用指针进行链表连接。

## mysql的索引，b树，聚集索引

1 MySQL的innodb支持聚簇索引，myisam不支持聚簇索引。

innodb在建表时自动按照第一个非空字段或者主键建立聚簇索引。mysql使用B+树建立索引。

每一个非叶子结点只存储主键值，而叶子节点则是一个数据页，这个数据页就是上面所说的存储数据的page页。

一个节点页对应着多行数据，每个节点按照顺序使用指针连成一个链表。mysql使用索引访问一行数据时，先通过log2n的时间访问到叶子节点，然后在数据页中按照行数链表执行顺序查找，直到找到那一行数据。

2 b+树索引可以很好地支持范围搜索，因为叶子节点通过指针相连。

## mysql的explain 慢查询日志

explain主要用于检查sql语句的执行计划，然后分析sql是否使用到索引，是否进行了全局扫描等等。

mysql慢查询日志可以在mysql的,my.cnf文件中配置开启，然后执行操作超过设置时间就会记录慢日志。

比如分析一个sql：

    explain查看执行计划
````    
    id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
    
    1	SIMPLE	vote_record	\N	ALL	votenum,vote	\N	\N	\N	996507	50.00	Using where
````

    
    
    还是没用到索引，因为不符合最左前缀匹配。查询需要3.5秒左右


    
    
    最后修改一下sql语句
````    
    EXPLAIN SELECT * FROM vote_record WHERE id > 0 AND vote_num > 1000;
    
    id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
    
    1	SIMPLE	vote_record	\N	range	PRIMARY,votenum,vote	PRIMARY	4	\N	498253	50.00	Using where
````

    
    
    用到了索引，但是只用到了主键索引。再修改一次


    
````    
    EXPLAIN SELECT * FROM vote_record WHERE id > 0 AND vote_num = 1000;


    
    
    id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
    
    1	SIMPLE	vote_record	\N	index_merge	PRIMARY,votenum,vote	votenum,PRIMARY	8,4	\N	51	100.00	Using intersect(votenum,PRIMARY); Using where


    
````    
    用到了两个索引，votenum,PRIMARY。

## mysql的binlog,redo log和undo log。

binlog就是二进制日志，用于记录用户数据操作的日志。用于主从复制。

redolog负责事务的重做，记录事务中的每一步操作，记录完再执行操作，并且在数据刷入磁盘前刷入磁盘，保证可以重做成功。

undo日志负责事务的回滚，记录事务操作中的原值，记录完再执行操作，在事务提交前刷入磁盘，保证可以回滚成功。

这两个日志也是实现分布式事务的基础。

## mysql的数据类型

mysql一般提供多种数据类型，int，double，varchar，tinyint，datatime等等。文本的话有fulltext，mediumtext等。没啥好说的。

## mysql的sql优化。

sql能优化的点是在有点多。

比如基本的，不使用null判断，不使用><
分页的时候利用到索引，查询的时候注意顺序。

如果是基于索引的优化，则要注意索引列是否能够使用到

    1 索引列不要使用>< != 以及 null，还有exists等。
    
    2 索引列不要使用聚集函数。
    
    3 如果是联合索引，排在第一位的索引一定要用到，否则后面的也会失效，为什么呢，因为第一列索引不同时才会找第二列，如果没有第一列索引，后续的索引页没有意义。
    
    举个例子。联合索引A,B,C。查询时必须要用到A，但是A的位置无所谓，只要用到就行，A,B,C或者C,B,A都可以。
    
    4 分页时直接limit n 5可能用不到索引，假设索引列是ID，那么我们使用where id > n limit 5就可以实现上述操作了。

## MySQL的事务实现和锁    


innodb支持行级锁和事务，而myisam只支持表锁，它的所有操作都需要加锁。

1 锁

    锁可以分为共享锁和排它锁，也叫读锁和写锁。
    
    select操作默认不加锁，需要加锁时会用for update加排它锁，或者用in share mode表示加共享锁。
    
    这里的锁都是行锁。
    innodb会使用行锁配合mvcc一同完成事务的实现。
    并且使用next-key lock来实现可重复读，而不必加表锁或者串行化执行。

2   MVCC

    MVCC是多版本控制协议。
    
    通过时间戳来判断先后顺序，并且是无锁的。但是需要额外存一个字段。
    
    读操作比较自己的版本号，自动读取比自己版本号新的版本。不读。
    
    写操作自动覆盖写版本号比自己的版本号早的版本。否则不写。
    
    这样保证一定程度上的一致性。
    
    MVCC比较好地支持读多写少的情景。
    
    但是偶尔需要加锁时才会进行加锁。

3 事务

所以看看innodb如何实现事务的。

首先，innodb的行锁是加在索引上的，因为innodb默认有聚簇索引，但实际上的行锁是对整个索引节点进行加锁，锁了该节点所有的行。

看看innodb如何实现隔离级别以及解决一致问题

    未提交读，会导致脏读，没有并发措施
    
    已提交读，写入时需要加锁，使用行级写锁锁加锁指定行，其他事务就看不到未提交事务的数据了。但是会导致不可重读，
    
    可重复读：在原来基础上，在读取行时也需要加行级读锁，这样其他事务不能修改这些数据。就避免了不可重读。
    但是这样会导致幻读。
    
    序列化：序列化会串行化读写操作来避免幻读，事实上就是事务在读取数据时加了表级读锁。

但是实际上。mysql的新版innodb引擎已经解决了幻读的问题，并且使用的是可重复读级别就能解决幻读了。

实现的原理是next-key lock。是gap lock的加强版。不会锁住全表，只会锁住被读取行前后的间隙行。


    
    
## 分库分表

分库分表的方案比较多，首先看下分表。

当一个大表没办法继续优化的时候，可以使用分表，横向拆分的方案就是把一个表的数据放到多个表中。一般可以按照某个键来分表。比如最常用的id，1-100w放在表一。100w-200w在表二，以此类推。

如果是纵向分表，则可以按列拆分，比如用户信息的字段放在一个表，用户使用数据放在另一个表，这其实就是一次性拆表了。

分库的话就是把数据表存到多个库中了，和横向分表的效果差不多。

如果只是单机的分表分库，其性能瓶颈在于主机。

我们需要考虑扩展性，所以需要使用分布式的数据库。

分布式数据库解决方案mycat
    
    mycat是一款支持分库分表的数据库中间件，支持单机也支持分布式。
    
    首先部署mycat，mycat的访问方式和一个mysqlserver是类似的。里面可以配置数据库和数据表。
    
    然后在mycat的配置文件中，我们可以指定分片，比如按照id分片，然后在每个分片下配置mysql节点，可以是本地的数据库实例也可以是其他主机上的数据库。
    
    这样的话，每个分片都能找到对应机器上的数据库和表了。
    
    用户连接mycat执行数据库操作，实际上会根据id映射到对应的数据库和表中，

## 主从复制，读写分离

主从复制大法好，为了避免单点mysql宕机和丢失数据，我们一般使用主从部署，主节点将操作日志写入binlog，然后日志文件通过一个连接传给从节点的relaylog。从节点定时去relaylog读取日志，并且执行操作。这样保证了主从的同步。

读写分离大法好，为了避免主库的读写压力太大，由于业务以读操作为主，所以主节点一般作为主库，读节点作为从库，从库负责读，主库负责写，写入主库的数据通过日志同步给从库。这样的部署就是读写分离。

使用mycat中间件也可以配置读写分离，只需在分片时指定某个主机是读节点还是写节点即可。

## 分布式数据库

分布式关系数据库无非就是关系数据库的分布式部署方案。

真正的分布式数据库应该是nosql数据库，比如基于hdfs的hbase数据库。底层就是分布式的。

redis的分布式部署方案也比较成熟。


