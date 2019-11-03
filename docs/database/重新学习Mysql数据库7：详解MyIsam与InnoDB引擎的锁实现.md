本文转自互联网

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《重新学习MySQL数据库》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从sql基本的使用方法，从MySQL执行引擎再到索引、事务等知识，一步步地学习MySQL相关技术的实现原理，更好地了解如何基于这些知识来优化sql，减少SQL执行时间，通过执行计划对SQL性能进行分析，再到MySQL的主从复制、主备部署等内容，以便让你更完整地了解整个MySQL方面的技术体系，形成自己的知识框架。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->


说到锁机制之前，先来看看Mysql的存储引擎，毕竟不同的引擎的锁机制也随着不同。

## 三类常见引擎：

MyIsam ：不支持事务，不支持外键，所以访问速度快。锁机制是表锁，支持全文索引

InnoDB ：支持事务、支持外键，所以对比MyISAM，InnoDB的处理效率差一些，并要占更多的磁盘空间保留数据和索引。锁机制是行锁，不支持全文索引

Memory：数据是存放在内存中的，默认哈希索引，非常适合存储临时数据，服务器关闭后，数据会丢失掉。

  



### 如何选择存储引擎：

MyISAM：应用是以读操作和插入操作为主，只有很少的更新和删除操作，并且对事务的完整性、并发性要求不是很高。

InnoDB：用于事务处理应用程序，支持外键，如果应用对事务的完整性有比较高的要求，在并发条件下要求数据的一致性。更新删除等频繁（InnoDB可以有效的降低由于删除和更新导致的锁定），对于数据准确性要求比较高的，此引擎适合。

Memory：通常用于更新不太频繁的小表，用以快速得到访问结果。

### Mysql中的锁

如果熟悉多线程，那么对锁肯定是有概念的，锁是计算机协调多个进程或线程对某一资源并发访问的机制。

Mysql中的锁分为表锁和行锁：

顾名思义，表锁就是锁住一张表，而行锁就是锁住一行。

表锁的特点：开销小，不会产生死锁，发生锁冲突的概率高，并且并发度低。

行锁的特点：开销大，会产生死锁，发生锁冲突的概率低，并发度高。

因此MyISAM和Memory引擎采用的是表锁，而InnoDB存储引擎采用的是行锁。

### MyISAM的锁机制：

分为共享读锁和独占写锁。

读锁是：当某一进程对某张表进行读操作时（select），其他线程也可以读，但是不能写。简单的理解就是，我读的时候你不能写。

写锁是：当某一进程对某种表某张表的写时（insert，update，，delete），其他线程不能写也不能读。可以理解为，我写的时候，你不能读，也不能写。

因此MyISAM的读操作和写操作，以及写操作之间是串行的！MyISAM在执行读写操作的时候会自动给表加相应的锁（也就是说不用显示的使用lock table命令），MyISAM总是一次获得SQL语句所需要的全部锁，这也是MyISAM不会出现死锁的原因。

下面分别举关于写锁和读锁的例子：

写锁：



| 事务1 | 事务2 |
| --- | --- |
| 取得first_test表的写锁：mysql> lock table first_test write;Query OK, 0 rows affected (0.00 sec) |  |
| 当前事务对查询、更新和插入操作都可以执行mysql> select * from first_test ;+----+------+| id  | age   |+----+------+|  1   |   10  ||  2   |   11  ||  3   |   12  ||  4   |   13  |+----+------+4 rows in set (0.00 sec)mysql> insert into first_test(age) values(14);Query OK, 1 row affected (0.11 sec) | 其他事务对锁定表的查询被阻塞，需要等到锁被释放，才可以执行mysql> select * from first_test;等待...... |
| mysql> unlock table;Query OK, 0 rows affected (0.00 sec) | 等待 |
|   | mysql> select * from first_test;+----+------+| id  | age  |+----+------+|  1  |   10  ||  2  |    11 ||  3  |    12 ||  4  |    13 ||  5  |    14 |+----+------+5 rows in set (9 min 45.02 sec) |



读锁例子如下：



| 事务1 | 事务2 |
| --- | --- |
| 获得表first_read的锁定mysql> lock table first_test read;Query OK, 0 rows affected (0.00 sec) |   |
| 当前事务可以查询该表记录：mysql> select * from first_test;+----+------+| id   |  age |+----+------+|  1   |    10 ||  2   |    11 ||  3   |    12 ||  4   |    13 ||  5   |    14 |+----+------+5 rows in set (0.00 sec) | 其他事务也可以查到该表信息mysql> select * from first_test;+----+------+| id   |  age  |+----+------+|  1   |    10 ||  2   |    11 ||  3   |    12 ||  4   |    13 ||  5   |    14 |+----+------+5 rows in set (0.00 sec) |
| 但是当前事务不能查询没有锁定的表：mysql> select * from goods;ERROR 1100 (HY000): Table 'goods' was not locked with LOCK TABLES | 其他事务可以查询或更新未锁定的表：mysql> select * from goods;+----+------------+------+| id   | name     | num |+----+------------+------+|  1  | firstGoods  |   11 ||  3 | ThirdGoods |   11 ||  4 | fourth            |   11 |+----+------------+------+10 rows in set (0.00 sec) |
| 而且插入更新锁定的表都会报错：mysql> insert into first_test(age) values(15);ERROR 1099 (HY000): Table 'first_test' was locked with a READ lock and can't be updatedmysql> update first_test set age=100 where id =1;ERROR 1099 (HY000): Table 'first_test' was locked with a READ lock and can't be updated | 当更新被锁定的表时会等待：mysql> update first_test set age=100 where id =1;等待...... |
| mysql> unlock table;Query OK, 0 rows affected (0.00 sec) | mysql> update first_test set age=100 where id =1;Query OK, 1 row affected (38.82 sec)Rows matched: 1  Changed: 1  Warnings: 0 |



### 并发插入

刚说到Mysql在插入和修改的时候都是串行的，但是MyISAM也支持查询和插入的并发操作。

MyISAM中有一个系统变量concurrent_insert（默认为1），用以控制并发插入（用户在表尾插入数据）行为。

当concurrent_insert为0时，不允许并发插入。

当concurrent_insert为1时，如果表中没有空洞（中间没有被删除的行），MyISAM允许一个进程在读表的同时，另一个进程从表尾插入记录。

当concurrent_insert为2时，无论MyISAM表中有没有空洞，都可以在末尾插入记录



| 事务1    | 事务2 |
| --- | --- |
| mysql> lock table first_test read local;Query OK, 0 rows affected (0.00 sec)--加入local选项是说明，在表满足并发插入的前提下，允许在末尾插入数据 |   |
| 当前进程不能进行插入和更新操作mysql> insert into first_test(age) values(15);ERROR 1099 (HY000): Table 'first_test' was locked with a READ lock and can't be updatedmysql> update first_test set age=200 where id =1;ERROR 1099 (HY000): Table 'first_test' was locked with a READ lock and can't be updated | 其他进程可以进行插入，但是更新会等待：mysql> insert into first_test(age) values(15);Query OK, 1 row affected (0.00 sec)mysql> update first_test set age=200 where id =2;等待..... |
| 当前进程不能不能访问其他进程插入的数据mysql> select * from first_test;+----+------+| id | age  |+----+------+|  1 |  100 ||  2 |   11 ||  3 |   12 ||  4 |   13 ||  5 |   14 ||  6 |   14 |+----+------+6 rows in set (0.00 sec) |   |
| 释放锁以后皆大欢喜mysql> unlock table;Query OK, 0 rows affected (0.00 sec) | 等待 |
| 插入的和更新的都出来的：mysql> select * from first_test;+----+------+| id | age  |+----+------+|  1 |  100 ||  2 |  200 ||  3 |   12 ||  4 |   13 ||  5 |   14 ||  6 |   14 ||  7 |   15 |+----+------+7 rows in set (0.00 sec) | mysql> update first_test set age=200 where id =2;Query OK, 1 row affected (1 min 39.75 sec)Rows matched: 1  Changed: 1  Warnings: 0 |



需要注意的：

并发插入是解决对同一表中的查询和插入的锁争用。

如果对有空洞的表进行并发插入会产生碎片，所以在空闲时可以利用optimize table命令回收因删除记录产生的空洞。

### 锁调度

在MyISAM中当一个进程请求某张表的读锁，而另一个进程同时也请求写锁，Mysql会先让后者获得写锁。即使读请求比写请求先到达锁等待队列，写锁也会插入到读锁之前。

因为Mysql总是认为写请求一般比读请求重要，这也就是MyISAM不太适合有大量的读写操作的应用的原因，因为大量的写请求会让查询操作很难获取到读锁，有可能永远阻塞。

处理办法：

1、指定Insert、update、delete语句的low_priority属性，降低其优先级。

2、指定启动参数low-priority-updates，使得MyISAM默认给读请求优先的权利。

3、执行命令set low_priority_updates=1，使该连接发出的请求降低。

4、指定max_write_lock_count设置一个合适的值，当写锁达到这个值后，暂时降低写请求的优先级，让读请求获取锁。

但是上面的处理办法造成的原因就是当遇到复杂的查询语句时，写请求可能很难获取到锁，这是一个很纠结的问题，所以我们一般避免使用复杂的查询语句，如果如法避免，则可以再数据库空闲阶段（深夜）执行。

我们知道mysql在以前，存储引擎默认是MyISAM，但是随着对事务和并发的要求越来越高，便引入了InnoDB引擎，它具有支持事务安全等一系列特性。



### InnoDB锁模式

InnoDB实现了两种类型的行锁。

共享锁（S）：允许一个事务去读一行，阻止其他事务获得相同的数据集的排他锁。

排他锁（X）：允许获得排他锁的事务更新数据，但是组织其他事务获得相同数据集的共享锁和排他锁。

可以这么理解：

共享锁就是我读的时候，你可以读，但是不能写。排他锁就是我写的时候，你不能读也不能写。其实就是MyISAM的读锁和写锁，但是针对的对象不同了而已。

除此之外InnoDB还有两个表锁：

意向共享锁（IS）：表示事务准备给数据行加入共享锁，也就是说一个数据行加共享锁前必须先取得该表的IS锁

意向排他锁（IX）：类似上面，表示事务准备给数据行加入排他锁，说明事务在一个数据行加排他锁前必须先取得该表的IX锁。



InnoDB行锁模式兼容列表：

![](https://img-blog.csdn.net/20150809115556064?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)


注意：

当一个事务请求的锁模式与当前的锁兼容，InnoDB就将请求的锁授予该事务；反之如果请求不兼容，则该事务就等待锁释放。

意向锁是InnoDB自动加的，不需要用户干预。

对于insert、update、delete，InnoDB会自动给涉及的数据加排他锁（X）；对于一般的Select语句，InnoDB不会加任何锁，事务可以通过以下语句给显示加共享锁或排他锁。

共享锁：select * from table_name where .....lock in share mode

排他锁：select * from table_name where .....for update

加入共享锁的例子：

![](https://blog.csdn.net/u014307117/article/details/47374531)

![](https://img-blog.csdn.net/20150809115652652)

利用select ....for update加入排他锁

![](https://blog.csdn.net/u014307117/article/details/47374531)![](https://img-blog.csdn.net/20150809115737527)

### 锁的实现方式：

InnoDB行锁是通过给索引项加锁实现的，如果没有索引，InnoDB会通过隐藏的聚簇索引来对记录加锁。

也就是说：如果不通过索引条件检索数据，那么InnoDB将对表中所有数据加锁，实际效果跟表锁一样。

行锁分为三种情形：

Record lock ：对索引项加锁，即锁定一条记录。

Gap lock：对索引项之间的‘间隙’、对第一条记录前的间隙或最后一条记录后的间隙加锁，即锁定一个范围的记录，不包含记录本身

Next-key Lock：锁定一个范围的记录并包含记录本身（上面两者的结合）。

注意：InnoDB默认级别是repeatable-read级别，所以下面说的都是在RR级别中的。

  

之前一直搞不懂Gap Lock和Next-key Lock的区别，直到在网上看到一句话豁然开朗，希望对各位有帮助。

Next-Key Lock是行锁与间隙锁的组合，这样，当InnoDB扫描索引记录的时候，会首先对选中的索引记录加上行锁（Record Lock），再对索引记录两边的间隙加上间隙锁（Gap Lock）。如果一个间隙被事务T1加了锁，其它事务是不能在这个间隙插入记录的。

干巴巴的说没意思，我们来看看具体实例：

假设我们有一张表：

+----+------+

| id | age  |

+----+------+

|  1 |    3 |

|  2 |    6 |

|  3 |    9 |

+----+------+

表结构如下：

CREATE TABLE `test` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `age` int(11) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `keyname` (`age`)
) ENGINE=InnoDB AUTO_INCREMENT=302 DEFAULT CHARSET=gbk ;

这样我们age段的索引就分为

(negative infinity, 3],

(3,6],

(6,9],

(9,positive infinity)；

  



我们来看一下几种情况：

1、当事务A执行以下语句：

mysql> select * from fenye where age=6for update ;

不仅使用行锁锁住了相应的数据行，同时也在两边的区间，（5,6]和（6，9] 都加入了gap锁。

这样事务B就无法在这个两个区间insert进新数据,但是事务B可以在两个区间外的区间插入数据。

2、当事务A执行

select * from fenye where age=7 for update ;

那么就会给(6,9]这个区间加锁，别的事务无法在此区间插入或更新数据。

3、如果查询的数据不再范围内，

比如事务A执行 select * from fenye where age=100 for update ;

那么加锁区间就是(9,positive infinity)。

小结：

行锁防止别的事务修改或删除，GAP锁防止别的事务新增，行锁和GAP锁结合形成的的Next-Key锁共同解决了RR级别在写数据时的幻读问题。


### 何时在InnoDB中使用表锁：

InnoDB在绝大部分情况会使用行级锁，因为事务和行锁往往是我们选择InnoDB的原因，但是有些情况我们也考虑使用表级锁。

1、当事务需要更新大部分数据时，表又比较大，如果使用默认的行锁，不仅效率低，而且还容易造成其他事务长时间等待和锁冲突。

2、事务比较复杂，很可能引起死锁导致回滚。

### 死锁：

我们说过MyISAM中是不会产生死锁的，因为MyISAM总是一次性获得所需的全部锁，要么全部满足，要么全部等待。而在InnoDB中，锁是逐步获得的，就造成了死锁的可能。

在上面的例子中我们可以看到，当两个事务都需要获得对方持有的锁才能够继续完成事务，导致双方都在等待，产生死锁。

发生死锁后，InnoDB一般都可以检测到，并使一个事务释放锁回退，另一个获取锁完成事务。

### 避免死锁：

有多种方法可以避免死锁，这里只介绍常见的三种：

1、如果不同程序会并发存取多个表，尽量约定以相同的顺序访问表，可以大大降低死锁机会。

2、在同一个事务中，尽可能做到一次锁定所需要的所有资源，减少死锁产生概率；

3、对于非常容易产生死锁的业务部分，可以尝试使用升级锁定颗粒度，通过表级锁定来减少死锁产生的概率；