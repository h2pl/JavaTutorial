# 目录
  * [redis 学习笔记](#redis-学习笔记)
    * [redis 是什么?](#redis-是什么)
    * [Redis 数据结构](#redis-数据结构)
    * [Redis 数据类型](#redis-数据类型)
    * [过期时间](#过期时间)
    * [应用场景](#应用场景)
    * [内存优化](#内存优化)
  * [天下无难试之Redis面试刁难大全](#天下无难试之redis面试刁难大全)


本文转自互联网

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《探索Redis设计与实现》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，Redis基本的使用方法，Redis的基本数据结构，以及一些进阶的使用方法，同时也需要进一步了解Redis的底层数据结构，再接着，还会带来Redis主从复制、集群、分布式锁等方面的相关内容，以及作为缓存的一些使用方法和注意事项，以便让你更完整地了解整个Redis相关的技术体系，形成自己的知识框架。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## redis 学习笔记

> 这篇 redis 学习笔记主要介绍 redis 的数据结构和数据类型，并讨论数据结构的选择以及应用场景的优化。

### redis 是什么?

> Redis是一种面向“键/值”对类型数据的分布式NoSQL数据库系统，特点是高性能，持久存储，适应高并发的应用场景。

### Redis 数据结构

*   动态字符串 (Sds)
*   双端列表 (LINKEDLIST)
*   字典
*   跳跃表 (SKIPLIST)
*   整数集合 (INTSET)
*   压缩列表 (ZIPLIST)

HUGOMORE42

[动态字符串](https://link.juejin.im/?target=http%3A%2F%2Forigin.redisbook.com%2Finternal-datastruct%2Fsds.html)

Sds (Simple Dynamic String,简单动态字符串)是 Redis 底层所使用的字符串表示,它被用 在几乎所有的 Redis 模块中

Redis 是一个键值对数据库(key-value DB),数据库的值可以是字符串、集合、列表等多种类 型的对象,而数据库的键则总是字符串对象

在 Redis 中, 一个字符串对象除了可以保存字符串值之外,还可以保存 long 类型的值当字符串对象保存的是字符串时,它包含的才是 sds 值,否则的话,它就 是一个 long 类型的值

动态字符串主要有两个作用:

1.  实现字符串对象(StringObject)
2.  在 Redis 程序内部用作 char * 类型的替代品

[双端列表](https://link.juejin.im/?target=http%3A%2F%2Forigin.redisbook.com%2Finternal-datastruct%2Fadlist.html)

双端链表还是 Redis 列表类型的底层实现之一，当对列表类型的键进行操作——比如执行 RPUSH 、LPOP 或 LLEN 等命令时,程序在底层操作的可能就是双端链表

双端链表主要有两个作用:

*   作为 Redis 列表类型的底层实现之一;
*   作为通用数据结构,被其他功能模块所使用;

[字典](https://link.juejin.im/?target=http%3A%2F%2Forigin.redisbook.com%2Finternal-datastruct%2Fdict.html)

字典(dictionary),又名映射(map)或关联数组(associative array), 它是一种抽象数据结 构,由一集键值对(key-value pairs)组成,各个键值对的键各不相同,程序可以将新的键值对 添加到字典中,或者基于键进行查找、更新或删除等操作

字典的应用

1.  实现数据库键空间(key space);
2.  用作 Hash 类型键的其中一种底层实现;

> Redis 是一个键值对数据库,数据库中的键值对就由字典保存:每个数据库都有一个与之相对应的字典,这个字典被称之为键空间(key space)。

Redis 的 Hash 类型键使用**字典和压缩列表**两种数据结构作为底层实现

[跳跃表](https://link.juejin.im/?target=http%3A%2F%2Forigin.redisbook.com%2Finternal-datastruct%2Fskiplist.html)

跳跃表(skiplist)是一种随机化的数据,由 William Pugh 在论文《Skip lists: a probabilistic alternative to balanced trees》中提出,这种数据结构以有序的方式在层次化的链表中保存元素,它的效率可以和平衡树媲美——查找、删除、添加等操作都可以在对数期望时间下完成, 并且比起平衡树来说,跳跃表的实现要简单直观得多

和字典、链表或者字符串这几种在 Redis 中大量使用的数据结构不同,跳跃表在 Redis 的唯一作用,就是实现有序集数据类型
跳跃表将指向有序集的 score 值和 member 域的指针作为元素,并以 score 值为索引,对有序集元素进行排序。

[整数集合](https://link.juejin.im/?target=http%3A%2F%2Forigin.redisbook.com%2Fcompress-datastruct%2Fintset.html)

整数集合(intset)用于有序、无重复地保存多个整数值,它会根据元素的值,自动选择该用什么长度的整数类型来保存元素

Intset 是集合键的底层实现之一,如果一个集合:

1.  只保存着整数元素;
2.  元素的数量不多;
    那么 Redis 就会使用 intset 来保存集合元素。

[压缩列表](https://link.juejin.im/?target=http%3A%2F%2Forigin.redisbook.com%2Fcompress-datastruct%2Fziplist.html)

Ziplist 是由一系列特殊编码的内存块构成的列表,一个 ziplist 可以包含多个节点(entry),每个节点可以保存一个长度受限的字符数组(不以 \0 结尾的 char 数组)或者整数

### Redis 数据类型

[RedisObject](https://link.juejin.im/?target=http%3A%2F%2Forigin.redisbook.com%2Fdatatype%2Fobject.html%23redisobject-redis)

redisObject 是 Redis 类型系统的核心,数据库中的每个键、值,以及 Redis 本身处理的参数,都表示为这种数据类型

redisObject 的定义位于 redis.h :

```
/** Redis 对象*/typedef struct redisObject {    // 类型    unsigned type:4;    // 对齐位    unsigned notused:2;    // 编码方式    unsigned encoding:4;    // LRU 时间(相对于 server.lruclock)    unsigned lru:22;    // 引用计数    int refcount;    // 指向对象的值    void *ptr;} robj;
```

type 、encoding 和 ptr 是最重要的三个属性。

type 记录了对象所保存的值的类型,它的值可能是以下常量的其中一个

```
/** 对象类型*/#define REDIS_STRING 0 // 字符串#define REDIS_LIST 1   // 列表#define REDIS_SET 2    // 集合#define REDIS_ZSET 3   // 有序集#define REDIS_HASH 4   // 哈希表
```

encoding 记录了对象所保存的值的编码,它的值可能是以下常量的其中一个

```
/** 对象编码*/#define REDIS_ENCODING_RAW 0    // 编码为字符串#define REDIS_ENCODING_INT 1    // 编码为整数#define REDIS_ENCODING_HT 2     // 编码为哈希表#define REDIS_ENCODING_ZIPMAP 3 // 编码为 zipmap(2.6 后不再使用)#define REDIS_ENCODING_LINKEDLIST 4 // 编码为双端链表#define REDIS_ENCODING_ZIPLIST 5    // 编码为压缩列表#define REDIS_ENCODING_INTSET 6     // 编码为整数集合#define REDIS_ENCODING_SKIPLIST 7    // 编码为跳跃表
```

ptr 是一个指针,指向实际保存值的数据结构,这个数据结构由 type 属性和 encoding 属性决定。

当执行一个处理数据类型的命令时,Redis 执行以下步骤:

1.  根据给定key,在数据库字典中查找和它像对应的redisObject,如果没找到,就返回 NULL 。
2.  检查redisObject的type属性和执行命令所需的类型是否相符,如果不相符,返回类 型错误。
3.  根据redisObject的encoding属性所指定的编码,选择合适的操作函数来处理底层的 数据结构。
4.  返回数据结构的操作结果作为命令的返回值。

[字符串](https://link.juejin.im/?target=http%3A%2F%2Fredisdoc.com%2Fstring%2Findex.html)

REDIS_STRING (字符串)是 Redis 使用得最为广泛的数据类型,它除了是 SET 、GET 等命令 的操作对象之外,数据库中的所有键,以及执行命令时提供给 Redis 的参数,都是用这种类型 保存的。

字符串类型分别使用 REDIS_ENCODING_INT 和 REDIS_ENCODING_RAW 两种编码

> 只有能表示为 long 类型的值,才会以整数的形式保存,其他类型 的整数、小数和字符串,都是用 sdshdr 结构来保存

[哈希表](https://link.juejin.im/?target=http%3A%2F%2Fredisdoc.com%2Fhash%2Findex.html)

REDIS_HASH (哈希表)是HSET 、HLEN 等命令的操作对象

它使用 REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_HT 两种编码方式

Redis 中每个hash可以存储232-1键值对（40多亿）

[列表](https://link.juejin.im/?target=http%3A%2F%2Fredisdoc.com%2Flist%2Findex.html)

REDIS_LIST(列表)是LPUSH 、LRANGE等命令的操作对象

它使用 REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_LINKEDLIST 这两种方式编码

一个列表最多可以包含232-1 个元素(4294967295, 每个列表超过40亿个元素)。

[集合](https://link.juejin.im/?target=http%3A%2F%2Fredisdoc.com%2Fset%2Findex.html)

REDIS_SET (集合) 是 SADD 、 SRANDMEMBER 等命令的操作对象

它使用 REDIS_ENCODING_INTSET 和 REDIS_ENCODING_HT 两种方式编码

Redis 中集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。

集合中最大的成员数为 232 - 1 (4294967295, 每个集合可存储40多亿个成员)

[有序集](https://link.juejin.im/?target=http%3A%2F%2Fredisdoc.com%2Fsorted_set%2Findex.html)

REDIS_ZSET (有序集)是ZADD 、ZCOUNT 等命令的操作对象

它使用 REDIS_ENCODING_ZIPLIST和REDIS_ENCODING_SKIPLIST 两种方式编码

不同的是每个元素都会关联一个double类型的分数。redis正是通过分数来为集合中的成员进行从小到大的排序。

有序集合的成员是唯一的,但分数(score)却可以重复。

集合是通过哈希表实现的，所以添加，删除，查找的复杂度都是O(1)。 集合中最大的成员数为 232 - 1 (4294967295, 每个集合可存储40多亿个成员)

Redis各种数据类型_以及它们的编码方式

![Redis各种数据类型_以及它们的编码方式](https://user-gold-cdn.xitu.io/2017/9/17/2c71cff03efc96d2280d12602cc2aa92?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)Redis各种数据类型_以及它们的编码方式

### 过期时间

在数据库中,所有键的过期时间都被保存在 redisDb 结构的 expires 字典里:

```
typedef struct redisDb {    // ...    dict *expires;    // ...} redisDb;
```

expires 字典的键是一个指向 dict 字典(键空间)里某个键的指针,而字典的值则是键所指 向的数据库键的到期时间,这个值以 long long 类型表示

过期时间设置

Redis 有四个命令可以设置键的生存时间(可以存活多久)和过期时间(什么时候到期):

*   EXPIRE 以秒为单位设置键的生存时间;
*   PEXPIRE 以毫秒为单位设置键的生存时间;
*   EXPIREAT 以秒为单位,设置键的过期 UNIX 时间戳;
*   PEXPIREAT 以毫秒为单位,设置键的过期 UNIX 时间戳。

> 虽然有那么多种不同单位和不同形式的设置方式,但是 expires 字典的值只保存“以毫秒为单位的过期 UNIX 时间戳” ,这就是说,通过进行转换,所有命令的效果最后都和 PEXPIREAT 命令的效果一样。

**如果一个键是过期的,那它什么时候会被删除?**

下边是参考答案

1.  定时删除:在设置键的过期时间时,创建一个定时事件,当过期时间到达时,由事件处理 器自动执行键的删除操作。
2.  惰性删除:放任键过期不管,但是在每次从 dict 字典中取出键值时,要检查键是否过 期,如果过期的话,就删除它,并返回空;如果没过期,就返回键值。
3.  定期删除:每隔一段时间,对expires字典进行检查,删除里面的过期键

Redis 使用的过期键删除策略是惰性删除加上定期删除

### 应用场景

*   缓存
*   队列
*   需要精准设定过期时间的应用

> 比如你可以把上面说到的sorted set的score值设置成过期时间的时间戳，那么就可以简单地通过过期时间排序，定时清除过期数据了，不仅是清除Redis中的过期数据，你完全可以把Redis里这个过期时间当成是对数据库中数据的索引，用Redis来找出哪些数据需要过期删除，然后再精准地从数据库中删除相应的记录

*   排行榜应用，取TOP N操作

    > 这个需求与上面需求的不同之处在于，前面操作以时间为权重，这个是以某个条件为权重，比如按顶的次数排序，这时候就需要我们的sorted set出马了，将你要排序的值设置成sorted set的score，将具体的数据设置成相应的value，每次只需要执行一条ZADD命令即可

*   统计页面访问次数

> 使用 incr 命令 定时使用 getset 命令 读取数据 并设置新的值 0

*   使用set 设置标签

例如假设我们的话题D 1000被加了三个标签tag 1,2,5和77，就可以设置下面两个集合：

```
$ redis-cli sadd topics:1000:tags 1(integer) 1$ redis-cli sadd topics:1000:tags 2(integer) 1$ redis-cli sadd topics:1000:tags 5(integer) 1$ redis-cli sadd topics:1000:tags 77(integer) 1$ redis-cli sadd tag:1:objects 1000(integer) 1$ redis-cli sadd tag:2:objects 1000(integer) 1$ redis-cli sadd tag:5:objects 1000(integer) 1$ redis-cli sadd tag:77:objects 1000(integer) 1
```

要获取一个对象的所有标签：

```
$ redis-cli smembers topics:1000:tags1. 52. 13. 774. 2
```

获得一份同时拥有标签1, 2,10和27的对象列表。
这可以用SINTER命令来做，他可以在不同集合之间取出交集

### 内存优化

`问题`: Instagram的照片数量已经达到3亿，而在Instagram里，我们需要知道每一张照片的作者是谁，下面就是Instagram团队如何使用Redis来解决这个问题并进行内存优化的。

具体方法，参考下边这篇文章：[节约内存：Instagram的Redis实践](https://link.juejin.im/?target=http%3A%2F%2Fblog.nosqlfan.com%2Fhtml%2F3379.html)。

## 天下无难试之Redis面试刁难大全

Redis在互联网技术存储方面使用如此广泛，几乎所有的后端技术面试官都要在Redis的使用和原理方面对小伙伴们进行各种刁难。作为一名在互联网技术行业打击过成百上千名【请允许我夸张一下】的资深技术面试官，看过了无数落寞的身影失望的离开，略感愧疚，故献上此文，希望各位读者以后面试势如破竹，永无失败！    

Redis有哪些数据结构？

字符串String、字典Hash、列表List、集合Set、有序集合SortedSet。

如果你是Redis中高级用户，还需要加上下面几种数据结构HyperLogLog、Geo、Pub/Sub。

如果你说还玩过Redis Module，像BloomFilter，RedisSearch，Redis-ML，面试官得眼睛就开始发亮了。

使用过Redis分布式锁么，它是什么回事？

先拿setnx来争抢锁，抢到之后，再用expire给锁加一个过期时间防止锁忘记了释放。

这时候对方会告诉你说你回答得不错，然后接着问如果在setnx之后执行expire之前进程意外crash或者要重启维护了，那会怎么样？

这时候你要给予惊讶的反馈：唉，是喔，这个锁就永远得不到释放了。紧接着你需要抓一抓自己得脑袋，故作思考片刻，好像接下来的结果是你主动思考出来的，然后回答：我记得set指令有非常复杂的参数，这个应该是可以同时把setnx和expire合成一条指令来用的！对方这时会显露笑容，心里开始默念：摁，这小子还不错。

假如Redis里面有1亿个key，其中有10w个key是以某个固定的已知的前缀开头的，如果将它们全部找出来？

使用keys指令可以扫出指定模式的key列表。

对方接着追问：如果这个redis正在给线上的业务提供服务，那使用keys指令会有什么问题？

这个时候你要回答redis关键的一个特性：redis的单线程的。keys指令会导致线程阻塞一段时间，线上服务会停顿，直到指令执行完毕，服务才能恢复。这个时候可以使用scan指令，scan指令可以无阻塞的提取出指定模式的key列表，但是会有一定的重复概率，在客户端做一次去重就可以了，但是整体所花费的时间会比直接用keys指令长。

使用过Redis做异步队列么，你是怎么用的？

一般使用list结构作为队列，rpush生产消息，lpop消费消息。当lpop没有消息的时候，要适当sleep一会再重试。

如果对方追问可不可以不用sleep呢？list还有个指令叫blpop，在没有消息的时候，它会阻塞住直到消息到来。

如果对方追问能不能生产一次消费多次呢？使用pub/sub主题订阅者模式，可以实现1:N的消息队列。

如果对方追问pub/sub有什么缺点？在消费者下线的情况下，生产的消息会丢失，得使用专业的消息队列如rabbitmq等。

如果对方追问redis如何实现延时队列？我估计现在你很想把面试官一棒打死如果你手上有一根棒球棍的话，怎么问的这么详细。但是你很克制，然后神态自若的回答道：使用sortedset，拿时间戳作为score，消息内容作为key调用zadd来生产消息，消费者用zrangebyscore指令获取N秒之前的数据轮询进行处理。

到这里，面试官暗地里已经对你竖起了大拇指。但是他不知道的是此刻你却竖起了中指，在椅子背后。

如果有大量的key需要设置同一时间过期，一般需要注意什么？

如果大量的key过期时间设置的过于集中，到过期的那个时间点，redis可能会出现短暂的卡顿现象。一般需要在时间上加一个随机值，使得过期时间分散一些。

Redis如何做持久化的？

bgsave做镜像全量持久化，aof做增量持久化。因为bgsave会耗费较长时间，不够实时，在停机的时候会导致大量丢失数据，所以需要aof来配合使用。在redis实例重启时，会使用bgsave持久化文件重新构建内存，再使用aof重放近期的操作指令来实现完整恢复重启之前的状态。

对方追问那如果突然机器掉电会怎样？取决于aof日志sync属性的配置，如果不要求性能，在每条写指令时都sync一下磁盘，就不会丢失数据。但是在高性能的要求下每次都sync是不现实的，一般都使用定时sync，比如1s1次，这个时候最多就会丢失1s的数据。

对方追问bgsave的原理是什么？你给出两个词汇就可以了，fork和cow。fork是指redis通过创建子进程来进行bgsave操作，cow指的是copy on write，子进程创建后，父子进程共享数据段，父进程继续提供读写服务，写脏的页面数据会逐渐和子进程分离开来。

Pipeline有什么好处，为什么要用pipeline？

可以将多次IO往返的时间缩减为一次，前提是pipeline执行的指令之间没有因果相关性。使用redis-benchmark进行压测的时候可以发现影响redis的QPS峰值的一个重要因素是pipeline批次指令的数目。

Redis的同步机制了解么？

Redis可以使用主从同步，从从同步。第一次同步时，主节点做一次bgsave，并同时将后续修改操作记录到内存buffer，待完成后将rdb文件全量同步到复制节点，复制节点接受完成后将rdb镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点进行重放就完成了同步过程。

是否使用过Redis集群，集群的原理是什么？

Redis Sentinal着眼于高可用，在master宕机时会自动将slave提升为master，继续提供服务。

Redis Cluster着眼于扩展性，在单个redis内存不足时，使用Cluster进行分片存储。
