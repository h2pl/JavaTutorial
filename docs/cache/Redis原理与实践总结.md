# 目录
  * [使用和基础数据结构（外观）](#使用和基础数据结构（外观）)
  * [底层数据结构](#底层数据结构)
  * [redis server结构和数据库redisDb](#redis-server结构和数据库redisdb)
  * [redis的事件模型](#redis的事件模型)
  * [备份方式](#备份方式)
  * [redis主从复制](#redis主从复制)
  * [分布式锁实现](#分布式锁实现)
    * [使用setnx加expire实现加锁和时限](#使用setnx加expire实现加锁和时限)
    * [使用getset加锁和获取过期时间](#使用getset加锁和获取过期时间)
    * [2.0的setnx可以配置过期时间。](#20的setnx可以配置过期时间。)
    * [使用sentx将值设为时间戳，通过lua脚本进行cas比较和删除操作](#使用sentx将值设为时间戳，通过lua脚本进行cas比较和删除操作)
    * [分布式Redis锁：Redlock](#分布式redis锁：redlock)
    * [总结](#总结)
  * [分布式方案](#分布式方案)
  * [redis事务](#redis事务)
    * [redis脚本事务](#redis脚本事务)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


[toc]


本文主要对Redis的设计和实现原理做了一个介绍很总结，有些东西我也介绍的不是很详细准确，尽量在自己的理解范围内把一些知识点和关键性技术做一个描述。如有错误，还望见谅，欢迎指出。
这篇文章主要还是参考我之前的技术专栏总结而来的。欢迎查看：

重新学习Redis

https://blog.csdn.net/column/details/21877.html
<!-- more -->

## 使用和基础数据结构（外观）

redis的基本使用方式是建立在redis提供的数据结构上的。

字符串
REDIS_STRING (字符串)是 Redis 使用得最为广泛的数据类型,它除了是 SET 、GET 等命令 的操作对象之外,数据库中的所有键,以及执行命令时提供给 Redis 的参数,都是用这种类型 保存的。

字符串类型分别使用 REDIS_ENCODING_INT 和 REDIS_ENCODING_RAW 两种编码

只有能表示为 long 类型的值,才会以整数的形式保存,其他类型 的整数、小数和字符串,都是用 sdshdr 结构来保存

哈希表
REDIS_HASH (哈希表)是HSET 、HLEN 等命令的操作对象

它使用 REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_HT 两种编码方式

Redis 中每个hash可以存储232-1键值对（40多亿）

列表
REDIS_LIST(列表)是LPUSH 、LRANGE等命令的操作对象

它使用 REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_LINKEDLIST 这两种方式编码

一个列表最多可以包含232-1 个元素(4294967295, 每个列表超过40亿个元素)。

集合
REDIS_SET (集合) 是 SADD 、 SRANDMEMBER 等命令的操作对象

它使用 REDIS_ENCODING_INTSET 和 REDIS_ENCODING_HT 两种方式编码

Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。

集合中最大的成员数为 232 - 1 (4294967295, 每个集合可存储40多亿个成员)

有序集
REDIS_ZSET (有序集)是ZADD 、ZCOUNT 等命令的操作对象

它使用 REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_SKIPLIST 两种方式编码

不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。

有序集合的成员是唯一的,但分数(score)却可以重复。

集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。 集合中最大的成员数为 232 - 1 (4294967295, 每个集合可存储40多亿个成员)

## 底层数据结构

下面讨论redis底层数据结构


1 SDS动态字符串

sds字符串是字符串的实现

动态字符串是一个结构体，内部有一个buf数组，以及字符串长度，剩余长度等字段，优点是通过长度限制写入，避免缓冲区溢出，另外剩余长度不足时会自动扩容，扩展性较好，不需要频繁分配内存。

并且sds支持写入二进制数据，而不一定是字符。

2 dict字典

dict字典是哈希表的实现。

dict字典与Java中的哈希表实现简直如出一辙，首先都是数组+链表组成的结构，通过dictentry保存节点。

其中dict同时保存两个entry数组，当需要扩容时，把节点转移到第二个数组即可，平时只使用一个数组。

![image](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/redis_dict_structure.png)

3 压缩链表ziplist

3.1 ziplist是一个经过特殊编码的双向链表，它的设计目标就是为了提高存储效率。ziplist可以用于存储字符串或整数，其中整数是按真正的二进制表示进行编码的，而不是编码成字符串序列。它能以O(1)的时间复杂度在表的两端提供push和pop操作。

3.2 实际上，ziplist充分体现了Redis对于存储效率的追求。一个普通的双向链表，链表中每一项都占用独立的一块内存，各项之间用地址指针（或引用）连接起来。这种方式会带来大量的内存碎片，而且地址指针也会占用额外的内存。

3.3 而ziplist却是将表中每一项存放在前后连续的地址空间内，一个ziplist整体占用一大块内存。它是一个表（list），但其实不是一个链表（linked list）。

3.4 另外，ziplist为了在细节上节省内存，对于值的存储采用了变长的编码方式，大概意思是说，对于大的整数，就多用一些字节来存储，而对于小的整数，就少用一些字节来存储。

实际上。redis的字典一开始的数据比较少时，会使用ziplist的方式来存储，也就是key1，value1，key2，value2这样的顺序存储，对于小数据量来说，这样存储既省空间，查询的效率也不低。

当数据量超过阈值时，哈希表自动膨胀为之前我们讨论的dict。

4 quicklist

quicklist是结合ziplist存储优势和链表灵活性与一身的双端链表。

quicklist的结构为什么这样设计呢？总结起来，大概又是一个空间和时间的折中：

4.1 双向链表便于在表的两端进行push和pop操作，但是它的内存开销比较大。

首先，它在每个节点上除了要保存数据之外，还要额外保存两个指针；其次，双向链表的各个节点是单独的内存块，地址不连续，节点多了容易产生内存碎片。

4.2 ziplist由于是一整块连续内存，所以存储效率很高。

但是，它不利于修改操作，每次数据变动都会引发一次内存的realloc。特别是当ziplist长度很长的时候，一次realloc可能会导致大批量的数据拷贝，进一步降低性能。

![image](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/redis_quicklist_structure.png)

5 zset
zset其实是两种结构的合并。也就是dict和skiplist结合而成的。dict负责保存数据对分数的映射，而skiplist用于根据分数进行数据的查询（相辅相成）

6 skiplist

sortset数据结构使用了ziplist+zset两种数据结构。

Redis里面使用skiplist是为了实现sorted set这种对外的数据结构。sorted set提供的操作非常丰富，可以满足非常多的应用场景。这也意味着，sorted set相对来说实现比较复杂。

sortedset是由skiplist，dict和ziplist组成的。

当数据较少时，sorted set是由一个ziplist来实现的。
当数据多的时候，sorted

set是由一个叫zset的数据结构来实现的，这个zset包含一个dict + 一个skiplist。dict用来查询数据到分数(score)的对应关系，而skiplist用来根据分数查询数据（可能是范围查找）。

在本系列前面关于ziplist的文章里，我们介绍过，ziplist就是由很多数据项组成的一大块连续内存。由于sorted set的每一项元素都由数据和score组成，因此，当使用zadd命令插入一个(数据, score)对的时候，底层在相应的ziplist上就插入两个数据项：数据在前，score在后。

![image](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/redis_skiplist_example.png)

skiplist的节点中存着节点值和分数。并且跳表是根据节点的分数进行排序的，所以可以根据节点分数进行范围查找。

7inset

inset是一个数字结合，他使用灵活的数据类型来保持数字。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230406202310.png)

新创建的intset只有一个header，总共8个字节。其中encoding = 2, length = 0。
添加13, 5两个元素之后，因为它们是比较小的整数，都能使用2个字节表示，所以encoding不变，值还是2。
当添加32768的时候，它不再能用2个字节来表示了（2个字节能表达的数据范围是-215~215-1，而32768等于215，超出范围了），因此encoding必须升级到INTSET_ENC_INT32（值为4），即用4个字节表示一个元素。

8总结

sds是一个灵活的字符串数组，并且支持直接存储二进制数据，同时提供长度和剩余空间的字段来保证伸缩性和防止溢出。

dict是一个字典结构，实现方式就是Java中的hashmap实现，同时持有两个节点数组，但只使用其中一个，扩容时换成另外一个。

ziplist是一个压缩链表，他放弃内存不连续的连接方式，而是直接分配连续内存进行存储，减少内存碎片。提高利用率，并且也支持存储二进制数据。

quicklist是ziplist和传统链表的中和形成的链表结果，每个链表节点都是一个ziplist。

skiplist一般有ziplist和zset两种实现方法，根据数据量来决定。zset本身是由skiplist和dict实现的。

inset是一个数字集合，他根据插入元素的数据类型来决定数组元素的长度。并自动进行扩容。

9 他们实现了哪些结构

字符串由sds实现

list由ziplist和quicklist实现

sortset由ziplist和zset实现

hash表由dict实现

集合由inset实现。


## redis server结构和数据库redisDb

1 redis服务器中维护着一个数据库名为redisdb，实际上他是一个dict结构。

Redis的数据库使用字典作为底层实现，数据库的增、删、查、改都是构建在字典的操作之上的。

2 redis服务器将所有数据库都保存在服务器状态结构redisServer(redis.h/redisServer)的db数组（应该是一个链表）里：

同理也有一个redis client结构，通过指针可以选择redis client访问的server是哪一个。

3 redisdb的键空间

    typedef struct redisDb {
        // 数据库键空间，保存着数据库中的所有键值对
        dict *dict;                 /* The keyspace for this DB */
        // 键的过期时间，字典的键为键，字典的值为过期事件 UNIX 时间戳
        dict *expires;              /* Timeout of keys with a timeout set */
        // 数据库号码
        int id;                     /* Database ID */
        // 数据库的键的平均 TTL ，统计信息
        long long avg_ttl;          /* Average TTL, just for stats */
        //..
    } redisDb

这部分的代码说明了，redisdb除了维护一个dict组以外，还需要对应地维护一个expire的字典数组。

大的dict数组中有多个小的dict字典，他们共同负责存储redisdb的所有键值对。

同时，对应的expire字典则负责存储这些键的过期时间
![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230406201529.png)

4 过期键的删除策略

2、过期键删除策略
通过前面的介绍，大家应该都知道数据库键的过期时间都保存在过期字典里，那假如一个键过期了，那么这个过期键是什么时候被删除的呢？现在来看看redis的过期键的删除策略：

a、定时删除：在设置键的过期时间的同时，创建一个定时器，在定时结束的时候，将该键删除；

b、惰性删除：放任键过期不管，在访问该键的时候，判断该键的过期时间是否已经到了，如果过期时间已经到了，就执行删除操作；

c、定期删除：每隔一段时间，对数据库中的键进行一次遍历，删除过期的键。

## redis的事件模型

redis处理请求的方式基于reactor线程模型，即一个线程处理连接，并且注册事件到IO多路复用器，复用器触发事件以后根据不同的处理器去执行不同的操作。总结以下客户端到服务端的请求过程

总结

    远程客户端连接到 redis 后，redis服务端会为远程客户端创建一个 redisClient 作为代理。
    
    redis 会读取嵌套字中的数据，写入 querybuf 中。
    
    解析 querybuf 中的命令，记录到 argc 和 argv 中。
    
    根据 argv[0] 查找对应的 recommand。
    
    执行 recommend 对应的执行函数。
    
    执行以后将结果存入 buf & bufpos & reply 中。
    
    返回给调用方。返回数据的时候，会控制写入数据量的大小，如果过大会分成若干次。保证 redis 的相应时间。
    
    Redis 作为单线程应用，一直贯彻的思想就是，每个步骤的执行都有一个上限（包括执行时间的上限或者文件尺寸的上限）一旦达到上限，就会记录下当前的执行进度，下次再执行。保证了 Redis 能够及时响应不发生阻塞。

## 备份方式

快照（RDB）：就是我们俗称的备份，他可以在定期内对数据进行备份，将Redis服务器中的数据持久化到硬盘中；

只追加文件（AOF）：他会在执行写命令的时候，将执行的写命令复制到硬盘里面，后期恢复的时候，只需要重新执行一下这个写命令就可以了。类似于我们的MySQL数据库在进行主从复制的时候，使用的是binlog二进制文件，同样的是执行一遍写命令；

appendfsync同步频率的区别如下图：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230406201549.png)

## redis主从复制

Redis复制工作过程：

slave向master发送sync命令。

master开启子进程来讲dataset写入rdb文件，同时将子进程完成之前接收到的写命令缓存起来。

子进程写完，父进程得知，开始将RDB文件发送给slave。

master发送完RDB文件，将缓存的命令也发给slave。

master增量的把写命令发给slave。

    注意有两步操作，一个是写入rdb的时候要缓存写命令，防止数据不一致。发完rdb后还要发写命令给salve，以后增量发命令就可以了

## 分布式锁实现

### 使用setnx加expire实现加锁和时限

加锁时使用setnx设置key为1并设置超时时间，解锁时删除键

    tryLock(){  
        SETNX Key 1
        EXPIRE Key Seconds
    }
    release(){  
      DELETE Key
    }

这个方案的一个问题在于每次提交一个Redis请求，如果执行完第一条命令后应用异常或者重启，锁将无法过期，一种改善方案就是使用Lua脚本（包含SETNX和EXPIRE两条命令），但是如果Redis仅执行了一条命令后crash或者发生主从切换，依然会出现锁没有过期时间，最终导致无法释放。

### 使用getset加锁和获取过期时间

针对锁无法释放问题的一个解决方案基于GETSET命令来实现


    思路：
    
    SETNX(Key,ExpireTime)获取锁
    
    如果获取锁失败，通过GET(Key)返回的时间戳检查锁是否已经过期
    
    GETSET(Key,ExpireTime)修改Value为NewExpireTime
    
    检查GETSET返回的旧值，如果等于GET返回的值，则认为获取锁成功
    
    注意：这个版本去掉了EXPIRE命令，改为通过Value时间戳值来判断过期




### 2.0的setnx可以配置过期时间。

    V2.0 基于SETNX
    
    tryLock(){  
        SETNX Key 1 Seconds
    }
    release(){  
      DELETE Key
    }

Redis 2.6.12版本后SETNX增加过期时间参数，这样就解决了两条命令无法保证原子性的问题。但是设想下面一个场景：

1. C1成功获取到了锁，之后C1因为GC进入等待或者未知原因导致任务执行过长，最后在锁失效前C1没有主动释放锁 2. C2在C1的锁超时后获取到锁，并且开始执行，这个时候C1和C2都同时在执行，会因重复执行造成数据不一致等未知情况 3. C1如果先执行完毕，则会释放C2的锁，此时可能导致另外一个C3进程获取到了锁

### 使用sentx将值设为时间戳，通过lua脚本进行cas比较和删除操作

    V3.0
    tryLock(){  
        SETNX Key UnixTimestamp Seconds
    }
    release(){  
        EVAL(
          //LuaScript
          if redis.call("get",KEYS[1]) == ARGV[1] then
              return redis.call("del",KEYS[1])
          else
              return 0
          end
        )
    }

这个方案通过指定Value为时间戳，并在释放锁的时候检查锁的Value是否为获取锁的Value，避免了V2.0版本中提到的C1释放了C2持有的锁的问题；另外在释放锁的时候因为涉及到多个Redis操作，并且考虑到Check And Set 模型的并发问题，所以使用Lua脚本来避免并发问题。

如果在并发极高的场景下，比如抢红包场景，可能存在UnixTimestamp重复问题，另外由于不能保证分布式环境下的物理时钟一致性，也可能存在UnixTimestamp重复问题，只不过极少情况下会遇到。

### 分布式Redis锁：Redlock

redlock的思想就是要求一个节点获取集群中N/2 + 1个节点
上的锁才算加锁成功。


### 总结

不论是基于SETNX版本的Redis单实例分布式锁，还是Redlock分布式锁，都是为了保证下特性

    1. 安全性：在同一时间不允许多个Client同时持有锁
    2. 活性

    死锁：锁最终应该能够被释放，即使Client端crash或者出现网络分区（通常基于超时机制）
    容错性：只要超过半数Redis节点可用，锁都能被正确获取和释放

## 分布式方案

1 主从复制，优点是备份简易使用。缺点是不能故障切换，并且不易扩展。

2 使用sentinel哨兵工具监控和实现自动切换。

3 codis集群方案

首先codis使用代理的方式隐藏底层redis，这样可以完美融合以前的代码，不需要更改redis访问操作。

然后codis使用了zookeeper进行监控和自动切换。同时使用了redis-group的概念，保证一个group里是一主多从的主从模型，基于此来进行切换。

4 redis cluster集群

该集群是一个p2p方式部署的集群

Redis cluster是一个去中心化、多实例Redis间进行数据共享的集群。

每个节点上都保存着其他节点的信息，通过任一节点可以访问正常工作的节点数据，因为每台机器上的保留着完整的分片信息，某些机器不正常工作不影响整体集群的工作。并且每一台redis主机都会配备slave，通过sentinel自动切换。

## redis事务

事务
MULTI 、 EXEC 、 DISCARD 和 WATCH 是 Redis 事务相关的命令。事务可以一次执行多个命令， 并且带有以下两个重要的保证：

事务是一个单独的隔离操作：事务中的所有命令都会序列化、按顺序地执行。事务在执行的过程中，不会被其他客户端发送来的命令请求所打断。

事务是一个原子操作：事务中的命令要么全部被执行，要么全部都不执行。

redis事务有一个特点，那就是在2.6以前，事务的一系列操作，如果有的成功有的失败，仍然会提交成功的那部分，后来改为全部不提交了。

但是Redis事务不支持回滚，提交以后不能执行回滚操作。

    为什么 Redis 不支持回滚（roll back）
    如果你有使用关系式数据库的经验， 那么 “Redis 在事务失败时不进行回滚，而是继续执行余下的命令”这种做法可能会让你觉得有点奇怪。
    
    以下是这种做法的优点：
    
    Redis 命令只会因为错误的语法而失败（并且这些问题不能在入队时发现），或是命令用在了错误类型的键上面：这也就是说，从实用性的角度来说，失败的命令是由编程错误造成的，而这些错误应该在开发的过程中被发现，而不应该出现在生产环境中。
    因为不需要对回滚进行支持，所以 Redis 的内部可以保持简单且快速。




### redis脚本事务

Redis 脚本和事务
从定义上来说， Redis 中的脚本本身就是一种事务， 所以任何在事务里可以完成的事， 在脚本里面也能完成。 并且一般来说， 使用脚本要来得更简单，并且速度更快。

因为脚本功能是 Redis 2.6 才引入的， 而事务功能则更早之前就存在了， 所以 Redis 才会同时存在两种处理事务的方法。

redis事务的ACID特性
在传统的关系型数据库中,尝尝用ACID特质来检测事务功能的可靠性和安全性。
在redis中事务总是具有原子性(Atomicity),一致性(Consistency)和隔离性(Isolation),并且当redis运行在某种特定的持久化
模式下,事务也具有耐久性(Durability).

①原子性

事务具有原子性指的是,数据库将事务中的多个操作当作一个整体来执行,服务器要么就执行事务中的所有操作,要么就一个操作也不执行。
但是对于redis的事务功能来说,事务队列中的命令要么就全部执行,要么就一个都不执行,因此redis的事务是具有原子性的。

②一致性

    事务具有一致性指的是,如果数据库在执行事务之前是一致的,那么在事务执行之后,无论事务是否执行成功,数据库也应该仍然一致的。
    ”一致“指的是数据符合数据库本身的定义和要求,没有包含非法或者无效的错误数据。redis通过谨慎的错误检测和简单的设计来保证事务一致性。

③隔离性

    事务的隔离性指的是,即使数据库中有多个事务并发在执行,各个事务之间也不会互相影响,并且在并发状态下执行的事务和串行执行的事务产生的结果完全
    相同。
    因为redis使用单线程的方式来执行事务(以及事务队列中的命令),并且服务器保证,在执行事务期间不会对事物进行中断,因此,redis的事务总是以串行
    的方式运行的,并且事务也总是具有隔离性的

④持久性

    事务的耐久性指的是,当一个事务执行完毕时,执行这个事务所得的结果已经被保持到永久存储介质里面。
    因为redis事务不过是简单的用队列包裹起来一组redis命令,redis并没有为事务提供任何额外的持久化功能,所以redis事务的耐久性由redis使用的模式
    决定
