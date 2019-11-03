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


这周开始学习 Redis，看看Redis是怎么实现的。所以会写一系列关于 Redis的文章。这篇文章关于 Redis 的基础数据。阅读这篇文章你可以了解：

*   动态字符串（SDS）
*   链表
*   字典

三个数据结构 Redis 是怎么实现的。



# [](https://www.xilidou.com/2018/03/12/redis-data/#SDS "SDS")SDS

SDS （Simple Dynamic String）是 Redis 最基础的数据结构。直译过来就是”简单的动态字符串“。Redis 自己实现了一个动态的字符串，而不是直接使用了 C 语言中的字符串。

sds 的数据结构：


    struct sdshdr {   
    // buf 中已占用空间的长度 int len; 
    // buf 中剩余可用空间的长度 int free; 
    // 数据空间 
    char buf[];
        
    }



所以一个 SDS 的就如下图：

![sds](https://img.xilidou.com/img/2019-04-25-022158.jpg)

所以我们看到，sds 包含3个参数。buf 的长度 len，buf 的剩余长度，以及buf。

为什么这么设计呢？

*   可以直接获取字符串长度。
    C 语言中，获取字符串的长度需要用指针遍历字符串，时间复杂度为 O(n)，而 SDS 的长度，直接从len 获取复杂度为 O(1)。

*   杜绝缓冲区溢出。
    由于C 语言不记录字符串长度，如果增加一个字符传的长度，如果没有注意就可能溢出，覆盖了紧挨着这个字符的数据。对于SDS 而言增加字符串长度需要验证 free的长度，如果free 不够就会扩容整个 buf，防止溢出。

*   减少修改字符串长度时造成的内存再次分配。
    redis 作为高性能的内存数据库，需要较高的相应速度。字符串也很大概率的频繁修改。 SDS 通过未使用空间这个参数，将字符串的长度和底层buf的长度之间的额关系解除了。buf的长度也不是字符串的长度。基于这个分设计 SDS 实现了空间的预分配和惰性释放。

    1.  预分配
        如果对 SDS 修改后，如果 len 小于 1MB 那 len = 2 * len + 1byte。 这个 1 是用于保存空字节。
        如果 SDS 修改后 len 大于 1MB 那么 len = 1MB + len + 1byte。
    2.  惰性释放
        如果缩短 SDS 的字符串长度，redis并不是马上减少 SDS 所占内存。只是增加 free 的长度。同时向外提供 API 。真正需要释放的时候，才去重新缩小 SDS 所占的内存
*   二进制安全。
    C 语言中的字符串是以 ”\0“ 作为字符串的结束标记。而 SDS 是使用 len 的长度来标记字符串的结束。所以SDS 可以存储字符串之外的任意二进制流。因为有可能有的二进制流在流中就包含了”\0“造成字符串提前结束。也就是说 SDS 不依赖 “\0” 作为结束的依据。

*   兼容C语言
    SDS 按照惯例使用 ”\0“ 作为结尾的管理。部分普通C 语言的字符串 API 也可以使用。

# [](https://www.xilidou.com/2018/03/12/redis-data/#%E9%93%BE%E8%A1%A8 "链表")链表

C语言中并没有链表这个数据结构所以 Redis 自己实现了一个。Redis 中的链表是：

    typedef struct listNode { 
    // 前置节点 struct listNode *prev; 
    // 后置节点 struct listNode *next; 
    // 节点的值 void *value;} listNode;


非常典型的双向链表的数据结构。

同时为双向链表提供了如下操作的函数：


    /* * 双端链表迭代器 */typedef struct listIter { 
    // 当前迭代到的节点 listNode *next; 
    // 迭代的方向 int direction;} listIter;
    
    /* * 双端链表结构 
    
    */typedef struct list { 
    // 表头节点 listNode *head; 
    // 表尾节点 listNode *tail; 
    // 节点值复制函数 void *(*dup)(void *ptr); 
    // 节点值释放函数 void (*free)(void *ptr); 
    // 节点值对比函数 int (*match)(void *ptr, void *key); 
    // 链表所包含的节点数量 unsigned long len;} list;


链表的结构比较简单，数据结构如下：

![list](https://img.xilidou.com/img/2019-04-25-22159.jpg)

总结一下性质：

*   双向链表，某个节点寻找上一个或者下一个节点时间复杂度 O(1)。
*   list 记录了 head 和 tail，寻找 head 和 tail 的时间复杂度为 O(1)。
*   获取链表的长度 len 时间复杂度 O(1)。

# [](https://www.xilidou.com/2018/03/12/redis-data/#%E5%AD%97%E5%85%B8 "字典")字典

字典数据结构极其类似 java 中的 Hashmap。

Redis的字典由三个基础的数据结构组成。最底层的单位是哈希表节点。结构如下：

    typedef struct dictEntry {
        
        // 键
        void *key;
    
        // 值
        union {
            void *val;
            uint64_t u64;
            int64_t s64;
        } v;
    
        // 指向下个哈希表节点，形成链表
        struct dictEntry *next;
    
    } dictEntry;

实际上哈希表节点就是一个单项列表的节点。保存了一下下一个节点的指针。 key 就是节点的键，v是这个节点的值。这个 v 既可以是一个指针，也可以是一个 `uint64_t`或者 `int64_t` 整数。*next 指向下一个节点。

通过一个哈希表的数组把各个节点链接起来：
    typedef struct dictht {
        
        // 哈希表数组
        dictEntry **table;
    
        // 哈希表大小
        unsigned long size;
        
        // 哈希表大小掩码，用于计算索引值
        // 总是等于 size - 1
        unsigned long sizemask;
    
        // 该哈希表已有节点的数量
        unsigned long used;
    
    } dictht;
dictht

通过图示我们观察：

![dictht.png](https://img.xilidou.com/img/2019-04-25-022159.jpg)

实际上，如果对java 的基本数据结构了解的同学就会发现，这个数据结构和 java 中的 HashMap 是很类似的，就是数组加链表的结构。

字典的数据结构：

    typedef struct dict {
    
        // 类型特定函数
        dictType *type;
    
        // 私有数据
        void *privdata;
    
        // 哈希表
        dictht ht[2];
    
        // rehash 索引
        // 当 rehash 不在进行时，值为 -1
        int rehashidx; /* rehashing not in progress if rehashidx == -1 */
    
        // 目前正在运行的安全迭代器的数量
        int iterators; /* number of iterators currently running */
    
    } dict;

其中的dictType 是一组方法，代码如下：

<figure>

    /*
     * 字典类型特定函数
     */
    typedef struct dictType {
    
        // 计算哈希值的函数
        unsigned int (*hashFunction)(const void *key);
    
        // 复制键的函数
        void *(*keyDup)(void *privdata, const void *key);
    
        // 复制值的函数
        void *(*valDup)(void *privdata, const void *obj);
    
        // 对比键的函数
        int (*keyCompare)(void *privdata, const void *key1, const void *key2);
    
        // 销毁键的函数
        void (*keyDestructor)(void *privdata, void *key);
        
        // 销毁值的函数
        void (*valDestructor)(void *privdata, void *obj);
    
    } dictType;

字典的数据结构如下图：

![dict](https://img.xilidou.com/img/2019-04-25-022200.jpg)

这里我们可以看到一个dict 拥有两个 dictht。一般来说只使用 ht[0],当扩容的时候发生了rehash的时候，ht[1]才会被使用。

当我们观察或者研究一个hash结构的时候偶我们首先要考虑的这个 dict 如何插入一个数据？

我们梳理一下插入数据的逻辑。

*   计算Key 的 hash 值。找到 hash 映射到 table 数组的位置。

*   如果数据已经有一个 key 存在了。那就意味着发生了 hash 碰撞。新加入的节点，就会作为链表的一个节点接到之前节点的 next 指针上。

*   如果 key 发生了多次碰撞，造成链表的长度越来越长。会使得字典的查询速度下降。为了维持正常的负载。Redis 会对 字典进行 rehash 操作。来增加 table 数组的长度。所以我们要着重了解一下 Redis 的 rehash。步骤如下：

    1.  根据ht[0] 的数据和操作的类型（扩大或缩小），分配 ht[1] 的大小。
    2.  将 ht[0] 的数据 rehash 到 ht[1] 上。
    3.  rehash 完成以后，将ht[1] 设置为 ht[0]，生成一个新的ht[1]备用。
*   渐进式的 rehash 。
    其实如果字典的 key 数量很大，达到千万级以上，rehash 就会是一个相对较长的时间。所以为了字典能够在 rehash 的时候能够继续提供服务。Redis 提供了一个渐进式的 rehash 实现，rehash的步骤如下：

    1.  分配 ht[1] 的空间，让字典同时持有 ht[1] 和 ht[0]。
    2.  在字典中维护一个 rehashidx，设置为 0 ，表示字典正在 rehash。
    3.  在rehash期间，每次对字典的操作除了进行指定的操作以外，都会根据 ht[0] 在 rehashidx 上对应的键值对 rehash 到 ht[1]上。
    4.  随着操作进行， ht[0] 的数据就会全部 rehash 到 ht[1] 。设置ht[0] 的 rehashidx 为 -1，渐进的 rehash 结束。

这样保证数据能够平滑的进行 rehash。防止 rehash 时间过久阻塞线程。

*   在进行 rehash 的过程中，如果进行了 delete 和 update 等操作，会在两个哈希表上进行。如果是 find 的话优先在ht[0] 上进行，如果没有找到，再去 ht[1] 中查找。如果是 insert 的话那就只会在 ht[1]中插入数据。这样就会保证了 ht[1] 的数据只增不减，ht[0]的数据只减不增。

