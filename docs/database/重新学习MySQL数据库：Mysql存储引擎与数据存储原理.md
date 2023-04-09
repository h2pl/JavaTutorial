# Table of Contents

  * [**一、 MySQL记录存储（页为单位）**](#一、-mysql记录存储（页为单位）)
    * [**页头**](#页头)
    * [**虚记录**](#虚记录)
    * [**记录堆**](#记录堆)
    * [**自由空间链表**](#自由空间链表)
    * [**未分配空间**](#未分配空间)
    * [**Slot区**](#slot区)
    * [**页内记录维护**](#页内记录维护)
  * [**二、 MySQL InnoDB存储引擎内存管理**](#二、-mysql-innodb存储引擎内存管理)
    * [**预分配内存空间**](#预分配内存空间)
    * [**数据以页为单位加载 （减少io访问次数）**](#数据以页为单位加载-（减少io访问次数）)
    * [**数据内外存交换**](#数据内外存交换)
    * [**页面管理**](#页面管理)
    * [**页面淘汰**](#页面淘汰)
    * [**全表扫描对内存的影响？**](#全表扫描对内存的影响？)
    * [**页面淘汰**](#页面淘汰-1)
    * [**位置移动**](#位置移动)
  * [**三、MySQL事务实现原理**](#三、mysql事务实现原理)
    * [**1、事务特性**](#1、事务特性)
    * [**2、并发问题**](#2、并发问题)
    * [**3、隔离级别**](#3、隔离级别)
    * [**MySQL事务实现原理（事务管理机制）**](#mysql事务实现原理（事务管理机制）)
    * [**1、MVCC 多版本并发控制**](#1、mvcc-多版本并发控制)
    * [**2、undo log**](#2、undo-log)
    * [**3、redo log**](#3、redo-log)
    * [**意义**](#意义)
  * [**四、MySQL锁实现原理**](#四、mysql锁实现原理)


## **一、 MySQL记录存储（页为单位）**

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205306.png)

### **页头**

记录页面的控制信息，共占56字节，包括页的左右兄弟页面指针、页面空间使用情况等。

### **虚记录**

最大虚记录：比页内最大主键还大 最小虚记录：比页内最小主键还小 (作用：比如说我们要查看一个记录是否在这个页面里，就要看这个记录是否在最大最小虚记录范围内)

### **记录堆**

行记录存储区，分为有效记录和已删除记录两种

### **自由空间链表**

已删除记录组成的链表 (重复利用空间)

### **未分配空间**

页面未使用的存储空间；

### **Slot区**

页尾 页面最后部分，占8个字节，主要存储页面的校验信息；

### **页内记录维护**

**顺序保证**

物理有序(利于查询，不利于插入删除)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-db41a4ce70476cb846cbcd92d3cb6efd_720w.webp)

**逻辑有序(插入删除性能高，查询效率低)** 默认

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205330.png)

所以`MySQL`是像下图所示这样子有序的组织数据的。

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-5c927bccdfd6815a65b5bb1aebb13aae_720w.webp)

**2、插入策略**

**自由空间链表**（优先利用自由空间链表）

**未使用空间**

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205347.png)

**3、页内查询**

**遍历**

**二分查找(数据不一样大，不能用二分)**

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-0b09abe7fa1b04a6274003c2dd08f733_720w.webp)

利用槽位做二分，实现近似的二分查找，近似于跳表 。

## **二、 MySQL InnoDB存储引擎内存管理**

### **预分配内存空间**

内存池

### **数据以页为单位加载 （减少io访问次数）**

内存页面管理

- 页面映射（记录哪块磁盘上的数据加载到哪块内存上了）
- 页面数据管理

### **数据内外存交换**

数据淘汰

- 内存页耗尽
- 需要加载新数据

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-369ec920b95324b4f2359cbe505c9a75_720w.webp)

### **页面管理**

- 空闲页
- 数据页
- 脏页（需刷回磁盘）

### **页面淘汰**

LRU(淘汰冷数据)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-065e91777cbc08461d0e5a6d6c5de606_720w.webp)



```
某时刻状态->访问P2->访问新页P7
```

### **全表扫描对内存的影响？**

可能会把内存中的热数据淘汰掉（比如说对一个几乎没有访问量的表进行全表扫描）

所以`MySQL`不是单纯的利用`LRU算法`

解决问题：如何避免热数据被淘汰？

解决方案：访问时间 + 频率(redis)

两个`LRU`表

`MySQL`的解决方案

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-9bf6f401a22185ae7852b596f58c0fa4_720w.webp)

MySQL内存管理—LRU

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205410.png)


**页面装载**

磁盘数据到内存

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-3ad4cc1b1a83d8a7c0ecc8f7b9befd12_720w.webp)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205420.png)

没有空闲页怎么办？`Free list中取 > LRU中淘汰 > LRU Flush`

### **页面淘汰**

`LRU`尾部淘汰`Flush LRU`淘汰

`LRU`链表中将第一个脏页刷盘并“释放”，放到`LRU`尾部？直接放`FreeList`？

### **位置移动**

old 到 new new 到 old

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205431.png)

思考：移动时机是什么？`innodb_old_blocks_time`old区存活时间，大于此值，有机会进入new区

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205437.png)

`LRU_new`的操作 链表操作效率很高，有访问移动到表头？`Lock！！！MySQL`设计思路：减少移动次数

两个重要参考：1、`freed_page_clock：Buffer Pool`淘汰页数 2、`LRU_new`长度1/4

当前`freed_page_clock` - 上次移动到Header时`freed_page_clock>LRU_new长度1/4`



## **三、MySQL事务实现原理**

MySQL事务基本概念。

### **1、事务特性**

A（`Atomicity`原子性）：全部成功或全部失败

I（`Isolation`隔离性）：并行事务之间互不干扰

D（`Durability`持久性）：事务提交后，永久生效

C（`Consistency`一致性）：通过AID保证

### **2、并发问题**

脏读(`Drity Read`)：读取到未提交的数据

不可重复读(`Non-repeatable read`)：两次读取结果不同

幻读(`Phantom Read`)：select 操作得到的结果所表征的数据状态无法支撑后续的业务操作

### **3、隔离级别**

`Read Uncommitted`（未提交读）：最低隔离级别，会读取到其他事务未提交的数据。脏读；

`Read Committed`（提交读）：事务过程中可以读取到其他事务已提交的数据。不可重复读；

`Repeatable Read`（可重复读）：每次读取相同结果集，不管其他事务是否提交，幻读；（两次当前读不会产生幻读）

`Serializable`（串行化）：事务排队，隔离级别最高，性能最差；

### **MySQL事务实现原理（事务管理机制）**

### **1、MVCC 多版本并发控制**

解决读-写冲突 如何工作：隐藏列

–当前读（读在存储引擎中存储的那个数据）

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-241cfc7589ea0de0cac531355324d0a7_720w.webp)

RR级别下

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-70c832b4a5b81c944c99a96557bcddd6_720w.webp)

### **2、undo log**

回滚日志 保证事务原子性 实现数据多版本`delete undo log`：用于回滚，提交即清理；`update undo log`：用于回滚，同时实现快照读，不能随便删除

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-1e9ecbc408164c034a73c3415aa9105d_720w.webp)

思考：undolog如何清理？依据系统活跃的最小活跃事务ID Read view 为什么InnoDB count（*）这么慢？因为

### **3、redo log**

实现事务持久性

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-029212b1ddd7fb111cefdfe9e6648bb7_720w.webp)

写入流程 l 记录页的修改，状态为prepare l 事务提交，讲事务记录为commit状态

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205454.png)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-7d685adf553d18f2e8dcdcd632d8696f_720w.webp)

### **意义**

体积小，记录页的修改，比写入页代价低 末尾追加，随机写变顺序写，发生改变的页不固定

## **四、MySQL锁实现原理**

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-44403b5fcb1bd988f68f72172c95627b_720w.webp)

所有当前读加排他锁，都有哪些是当前读？`SELECT FOR UPDATEUPDATEDELETE`

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-6161f7636962904b6e17d57e1a2540dc_720w.webp)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-c6b3c1d5c628b0c2fc104421ddc0faad_720w.webp)

唯一索引/非唯一索引 `* RC/RR`4种情况逐一分析

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-323f7b0fc7e2bfbdee1bb19e8f7dea0f_720w.webp)

会出现幻读问题，不可重复读了

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-3e9c8bda25b0b6788952f5024819e4ff_720w.webp)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-7f9ba333592f1253ffa6adf619448aaf_720w.webp)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-55b01d4fd841e1c417004c86a80746b7_720w.webp)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-b155fff4fd294066127c2e6c2650f559_720w.webp)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405205509.png)

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-e87ad5dc811f52789e20adb3b53b8ba1_720w.webp)

死锁在库表中有记录，通过kill 那个锁删除。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/%E5%86%B3%E5%AE%9A%E7%89%88.jpeg)
