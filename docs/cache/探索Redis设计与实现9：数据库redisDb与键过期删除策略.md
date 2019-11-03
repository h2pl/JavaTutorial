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

一. 数据库
Redis的数据库使用字典作为底层实现，数据库的增、删、查、改都是构建在字典的操作之上的。
redis服务器将所有数据库都保存在服务器状态结构redisServer(redis.h/redisServer)的db数组（应该是一个链表）里：

    struct redisServer {
      //..
      // 数据库数组，保存着服务器中所有的数据库
        redisDb *db;
      //..
    }

在初始化服务器时，程序会根据服务器状态的dbnum属性来决定应该创建多少个数据库：

    struct redisServer {
        // ..
        //服务器中数据库的数量
        int dbnum;
        //..
    }

dbnum属性的值是由服务器配置的database选项决定的，默认值为16；

二、切换数据库原理
每个Redis客户端都有自己的目标数据库，每当客户端执行数据库的读写命令时，目标数据库就会成为这些命令的操作对象。

    127.0.0.1:6379> set msg 'Hello world'
    OK
    127.0.0.1:6379> get msg
    "Hello world"
    127.0.0.1:6379> select 2
    OK
    127.0.0.1:6379[2]> get msg
    (nil)
    127.0.0.1:6379[2]>

在服务器内部，客户端状态redisClient结构(redis.h/redisClient)的db属性记录了客户端当前的目标数据库，这个属性是一个指向redisDb结构(redis.h/redisDb)的指针：

    typedef struct redisClient {
        //..
        // 客户端当前正在使用的数据库
        redisDb *db;
        //..
    } redisClient;

redisClient.db指针指向redisServer.db数组中的一个元素，而被指向的元素就是当前客户端的目标数据库。
我们就可以通过修改redisClient指针，让他指向服务器中的不同数据库，从而实现切换数据库的功能–这就是select命令的实现原理。
实现代码：

    int selectDb(redisClient *c, int id) {
        // 确保 id 在正确范围内
        if (id < 0 || id >= server.dbnum)
            return REDIS_ERR;
        // 切换数据库（更新指针）
        c->db = &server.db[id];
        return REDIS_OK;
    }

三、数据库的键空间
1、数据库的结构（我们只分析键空间和键过期时间）
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


上图是一个RedisDb的示例，该数据库存放有五个键值对，分别是sRedis，INums，hBooks，SortNum和sNums，它们各自都有自己的值对象，另外，其中有三个键设置了过期时间，当前数据库是服务器的第0号数据库。现在，我们就从源码角度分析这个数据库结构：
我们知道，Redis是一个键值对数据库服务器，服务器中的每一个数据库都是一个redis.h/redisDb结构，其中，结构中的dict字典保存了数据库中所有的键值对，我们就将这个字典成为键空间。
Redis数据库的数据都是以键值对的形式存在，其充分利用了字典高效索引的特点。
a、键空间的键就是数据库中的键，一般都是字符串对象；
b、键空间的值就是数据库中的值，可以是5种类型对象（字符串、列表、哈希、集合和有序集合）之一。
数据库的键空间结构分析完了，我们先看看数据库的初始化。

2、键空间的初始化
在redis.c中，我们可以找到键空间的初始化操作：

    //创建并初始化数据库结构
     for (j = 0; j < server.dbnum; j++) {
        // 创建每个数据库的键空间
        server.db[j].dict = dictCreate(&dbDictType,NULL);
        // ...
        // 设定当前数据库的编号
        server.db[j].id = j;
    }

初始化之后就是对键空间的操作了。

3、键空间的操作
我先把一些常见的键空间操作函数列出来：

    // 从数据库中取出键key的值对象，若不存在就返回NULL
    robj *lookupKey(redisDb *db, robj *key);
    
    /* 先删除过期键，以读操作的方式从数据库中取出指定键对应的值对象
     * 并根据是否成功找到值，更新服务器的命中或不命中信息,
     * 如不存在则返回NULL，底层调用lookupKey函数 */
    robj *lookupKeyRead(redisDb *db, robj *key);

    /* 先删除过期键，以写操作的方式从数据库中取出指定键对应的值对象
     * 如不存在则返回NULL，底层调用lookupKey函数，
     * 不会更新服务器的命中或不命中信息
     */
    robj *lookupKeyWrite(redisDb *db, robj *key);
    
    /* 先删除过期键，以读操作的方式从数据库中取出指定键对应的值对象
     * 如不存在则返回NULL，底层调用lookupKeyRead函数
     * 此操作需要向客户端回复
     */
    robj *lookupKeyReadOrReply(redisClient *c, robj *key, robj *reply);

    /* 先删除过期键，以写操作的方式从数据库中取出指定键对应的值对象
     * 如不存在则返回NULL，底层调用lookupKeyWrite函数
     * 此操作需要向客户端回复
     */
    robj *lookupKeyWriteOrReply(redisClient *c, robj *key, robj *reply);
    
    /* 添加元素到指定数据库 */
    void dbAdd(redisDb *db, robj *key, robj *val);
    /* 重写指定键的值 */
    void dbOverwrite(redisDb *db, robj *key, robj *val);
    /* 设定指定键的值 */
    void setKey(redisDb *db, robj *key, robj *val);
    /* 判断指定键是否存在 */
    int dbExists(redisDb *db, robj *key);
    /* 随机返回数据库中的键 */
    robj *dbRandomKey(redisDb *db);
    /* 删除指定键 */
    int dbDelete(redisDb *db, robj *key);
    /* 清空所有数据库，返回键值对的个数 */
    long long emptyDb(void(callback)(void*));

下面我选取几个比较典型的操作函数分析一下：

查找键值对函数–lookupKey
robj *lookupKey(redisDb *db, robj *key) {
    // 查找键空间
    dictEntry *de = dictFind(db->dict,key->ptr);
    // 节点存在
    if (de) {
        // 取出该键对应的值
        robj *val = dictGetVal(de);
        // 更新时间信息
        if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
            val->lru = LRU_CLOCK();
        // 返回值
        return val;
    } else {
        // 节点不存在
        return NULL;
    }
}

添加键值对–dbAdd
添加键值对使我们经常使用到的函数，底层由dbAdd()函数实现，传入的参数是待添加的数据库，键对象和值对象，源码如下：

void dbAdd(redisDb *db, robj *key, robj *val) {
    // 复制键名
    sds copy = sdsdup(key->ptr);
    // 尝试添加键值对
    int retval = dictAdd(db->dict, copy, val);
    // 如果键已经存在，那么停止
    redisAssertWithInfo(NULL,key,retval == REDIS_OK);
    // 如果开启了集群模式，那么将键保存到槽里面
    if (server.cluster_enabled) slotToKeyAdd(key);
 }

好了，关于键空间操作函数就分析到这，其他函数(在文件db.c中)大家可以自己去分析，有问题的话可以回帖，我们可以一起讨论！

四、数据库的过期键操作
在前面我们说到，redisDb结构中有一个expires指针（概况图可以看上图），该指针指向一个字典结构，字典中保存了所有键的过期时间，该字典称为过期字典。
过期字典的初始化：

// 创建并初始化数据库结构
 for (j = 0; j < server.dbnum; j++) {
        // 创建每个数据库的过期时间字典
        server.db[j].expires = dictCreate(&keyptrDictType,NULL);
        // 设定当前数据库的编号
        server.db[j].id = j;
        // ..
    }

a、过期字典的键是一个指针，指向键空间中的某一个键对象（就是某一个数据库键）；
b、过期字典的值是一个long long类型的整数，这个整数保存了键所指向的数据库键的时间戳–一个毫秒精度的unix时间戳。
下面我们就来分析过期键的处理函数：

1、过期键处理函数
设置键的过期时间–setExpire()
/*
 * 将键 key 的过期时间设为 when
 */
void setExpire(redisDb *db, robj *key, long long when) {
    dictEntry *kde, *de;
    // 从键空间中取出键key
    kde = dictFind(db->dict,key->ptr);
    // 如果键空间找不到该键，报错
    redisAssertWithInfo(NULL,key,kde != NULL);
    // 向过期字典中添加该键
    de = dictReplaceRaw(db->expires,dictGetKey(kde));
    // 设置键的过期时间
    // 这里是直接使用整数值来保存过期时间，不是用 INT 编码的 String 对象
    dictSetSignedIntegerVal(de,when);
}

获取键的过期时间–getExpire()
long long getExpire(redisDb *db, robj *key) {
    dictEntry *de;
    // 如果过期键不存在，那么直接返回
    if (dictSize(db->expires) == 0 ||
       (de = dictFind(db->expires,key->ptr)) == NULL) return -1;
    redisAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);
    // 返回过期时间
    return dictGetSignedIntegerVal(de);
}

删除键的过期时间–removeExpire()
// 移除键 key 的过期时间
int removeExpire(redisDb *db, robj *key) {
    // 确保键带有过期时间
    redisAssertWithInfo(NULL,key,dictFind(db->dict,key->ptr) != NULL);
    // 删除过期时间
    return dictDelete(db->expires,key->ptr) == DICT_OK;
}

2、过期键删除策略
通过前面的介绍，大家应该都知道数据库键的过期时间都保存在过期字典里，那假如一个键过期了，那么这个过期键是什么时候被删除的呢？现在来看看redis的过期键的删除策略：
a、定时删除：在设置键的过期时间的同时，创建一个定时器，在定时结束的时候，将该键删除；
b、惰性删除：放任键过期不管，在访问该键的时候，判断该键的过期时间是否已经到了，如果过期时间已经到了，就执行删除操作；
c、定期删除：每隔一段时间，对数据库中的键进行一次遍历，删除过期的键。
其中定时删除可以及时删除数据库中的过期键，并释放过期键所占用的内存，但是它为每一个设置了过期时间的键都开了一个定时器，使的cpu的负载变高，会对服务器的响应时间和吞吐量造成影响。
惰性删除有效的克服了定时删除对CPU的影响，但是，如果一个过期键很长时间没有被访问到，且若存在大量这种过期键时，势必会占用很大的内存空间，导致内存消耗过大。
定时删除可以算是上述两种策略的折中。设定一个定时器，每隔一段时间遍历数据库，删除其中的过期键，有效的缓解了定时删除对CPU的占用以及惰性删除对内存的占用。
在实际应用中，Redis采用了惰性删除和定时删除两种策略来对过期键进行处理，上面提到的lookupKeyWrite等函数中就利用到了惰性删除策略，定时删除策略则是在根据服务器的例行处理程序serverCron来执行删除操作，该程序每100ms调用一次。

惰性删除函数–expireIfNeeded()
源码如下：

/* 检查key是否已经过期，如果是的话，将它从数据库中删除 
 * 并将删除命令写入AOF文件以及附属节点(主从复制和AOF持久化相关)
 * 返回0代表该键还没有过期，或者没有设置过期时间
 * 返回1代表该键因为过期而被删除
 */
int expireIfNeeded(redisDb *db, robj *key) {
    // 获取该键的过期时间
    mstime_t when = getExpire(db,key);
    mstime_t now;
    // 该键没有设定过期时间
    if (when < 0) return 0;
    // 服务器正在加载数据的时候，不要处理
    if (server.loading) return 0;
    // lua脚本相关
    now = server.lua_caller ? server.lua_time_start : mstime();
    // 主从复制相关，附属节点不主动删除key
    if (server.masterhost != NULL) return now > when;
    // 该键还没有过期
    if (now <= when) return 0;
    // 删除过期键
    server.stat_expiredkeys++;
    // 将删除命令传播到AOF文件和附属节点
    propagateExpire(db,key);
    // 发送键空间操作时间通知
    notifyKeyspaceEvent(NOTIFY_EXPIRED,
        "expired",key,db->id);
    // 将该键从数据库中删除
    return dbDelete(db,key);
}

定期删除策略
过期键的定期删除策略由redis.c/activeExpireCycle()函数实现，服务器周期性地操作redis.c/serverCron()（每隔100ms执行一次）时，会调用activeExpireCycle()函数，分多次遍历服务器中的各个数据库，从数据库中的expires字典中随机检查一部分键的过期时间，并删除其中的过期键。
删除过期键的操作由activeExpireCycleTryExpire函数(activeExpireCycle()调用了该函数)执行，其源码如下：

/* 检查键的过期时间，如过期直接删除*/
int activeExpireCycleTryExpire(redisDb *db, dictEntry *de, long long now) {
    // 获取过期时间
    long long t = dictGetSignedIntegerVal(de);
    if (now > t) {
        // 执行到此说明过期
        // 创建该键的副本
        sds key = dictGetKey(de);
        robj *keyobj = createStringObject(key,sdslen(key));
        // 将删除命令传播到AOF和附属节点
        propagateExpire(db,keyobj);
        // 在数据库中删除该键
        dbDelete(db,keyobj);
        // 发送事件通知
        notifyKeyspaceEvent(NOTIFY_EXPIRED,
            "expired",keyobj,db->id);
        // 临时键对象的引用计数减1
        decrRefCount(keyobj);
        // 服务器的过期键计数加1
        // 该参数影响每次处理的数据库个数
        server.stat_expiredkeys++;
        return 1;
    } else {
        return 0;
    }
}

删除过期键对AOF、RDB和主从复制都有影响，等到了介绍相关功能时再讨论。
今天就先到这里~
