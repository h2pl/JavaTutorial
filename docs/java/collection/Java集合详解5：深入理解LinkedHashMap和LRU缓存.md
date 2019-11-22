# Table of Contents

  * [LinkedHashMap 概述](#linkedhashmap-概述)
  * [LinkedHashMap 在 JDK 中的定义](#linkedhashmap-在-jdk-中的定义)
    * [类结构定义](#类结构定义)
    * [成员变量定义](#成员变量定义)
    * [成员方法定义](#成员方法定义)
    * [基本元素 Entry](#基本元素-entry)
  * [LinkedHashMap 的构造函数](#linkedhashmap-的构造函数)
    * [LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder)](#linkedhashmapint-initialcapacity-float-loadfactor-boolean-accessorder)
    * [LinkedHashMap(Map<? extends K, ? extends V> m)](#linkedhashmapmap-extends-k--extends-v-m)
    * [init 方法](#init-方法)
  * [LinkedHashMap 的数据结构](#linkedhashmap-的数据结构)
  * [LinkedHashMap 的快速存取](#linkedhashmap-的快速存取)
    * [LinkedHashMap 的存储实现 : put(key, vlaue)](#linkedhashmap-的存储实现--putkey-vlaue)
    * [LinkedHashMap 的扩容操作 : resize()](#linkedhashmap-的扩容操作--resize)
    * [LinkedHashMap 的读取实现 ：get(Object key)](#linkedhashmap-的读取实现-：getobject-key)
    * [LinkedHashMap 存取小结](#linkedhashmap-存取小结)
  * [LinkedHashMap 与 LRU(Least recently used，最近最少使用)算法](#linkedhashmap-与-lruleast-recently-used，最近最少使用算法)
    * [put操作与标志位accessOrder](#put操作与标志位accessorder)
    * [get操作与标志位accessOrder](#get操作与标志位accessorder)
    * [LinkedListMap与LRU小结](#linkedlistmap与lru小结)
  * [使用LinkedHashMap实现LRU算法](#使用linkedhashmap实现lru算法)
  * [LinkedHashMap 有序性原理分析](#linkedhashmap-有序性原理分析)
  * [JDK1.8的改动](#jdk18的改动)
  * [总结](#总结)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)

本文参考多篇优质技术博客，参考文章请在文末查看

《Java集合详解系列》是我在完成夯实Java基础篇的系列博客后准备开始整理的新系列文章。
为了更好地诠释知识点，形成体系文章，本系列文章整理了很多优质的博客内容，如有侵权请联系我，一定删除。

这些文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star、fork哈

本系列文章将整理于我的个人博客：

> www.how2playlife.com


摘要：

>HashMap和双向链表合二为一即是LinkedHashMap。所谓LinkedHashMap，其落脚点在HashMap，因此更准确地说，它是一个将所有Entry节点链入一个双向链表的HashMap。

 >由于LinkedHashMap是HashMap的子类，所以LinkedHashMap自然会拥有HashMap的所有特性。比如，LinkedHashMap的元素存取过程基本与HashMap基本类似，只是在细节实现上稍有不同。当然，这是由LinkedHashMap本身的特性所决定的，因为它额外维护了一个双向链表用于保持迭代顺序。

>此外，LinkedHashMap可以很好的支持LRU算法，笔者在第七节便在LinkedHashMap的基础上实现了一个能够很好支持LRU的结构。

友情提示：

> 　　本文所有关于 LinkedHashMap 的源码都是基于 JDK 1.6 的，不同 JDK 版本之间也许会有些许差异，但不影响我们对 LinkedHashMap 的数据结构、原理等整体的把握和了解。后面会讲解1.8对于LinkedHashMap的改动。
>
> 　　由于 LinkedHashMap 是 HashMap 的子类，所以其具有HashMap的所有特性，这一点在源码共用上体现的尤为突出。因此，读者在阅读本文之前，最好对 HashMap 有一个较为深入的了解和回顾，否则很可能会导致事倍功半。可以参考我之前关于hashmap的文章。

## LinkedHashMap 概述
>
> 　　笔者曾提到，HashMap 是 Java Collection Framework 的重要成员，也是Map族(如下图所示)中我们最为常用的一种。不过遗憾的是，HashMap是无序的，也就是说，迭代HashMap所得到的元素顺序并不是它们最初放置到HashMap的顺序。

>
> 　　HashMap的这一缺点往往会造成诸多不便，因为在有些场景中，我们确需要用到一个可以保持插入顺序的Map。庆幸的是，JDK为我们解决了这个问题，它为HashMap提供了一个子类 —— LinkedHashMap。虽然LinkedHashMap增加了时间和空间上的开销，但是它通过维护一个额外的双向链表保证了迭代顺序。
> 　　

> 　　特别地，该迭代顺序可以是插入顺序，也可以是访问顺序。因此，根据链表中元素的顺序可以将LinkedHashMap分为：保持插入顺序的LinkedHashMap和保持访问顺序的LinkedHashMap，其中LinkedHashMap的默认实现是按插入顺序排序的。

　　　　　![image](https://img-blog.csdn.net/20170317181610752?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)　　　　　　

> 　 本质上，HashMap和双向链表合二为一即是LinkedHashMap。所谓LinkedHashMap，其落脚点在HashMap，因此更准确地说，它是一个将所有Entry节点链入一个双向链表双向链表的HashMap。

> 　 在LinkedHashMapMap中，所有put进来的Entry都保存在如下面第一个图所示的哈希表中，但由于它又额外定义了一个以head为头结点的双向链表(如下面第二个图所示)，因此对于每次put进来Entry，除了将其保存到哈希表中对应的位置上之外，还会将其插入到双向链表的尾部。

![image](https://img-blog.csdn.net/20170317181650025?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
    
　　更直观地，下图很好地还原了LinkedHashMap的原貌：HashMap和双向链表的密切配合和分工合作造就了LinkedHashMap。特别需要注意的是，next用于维护HashMap各个桶中的Entry链，before、after用于维护LinkedHashMap的双向链表，虽然它们的作用对象都是Entry，但是各自分离，是两码事儿。

![这里写图片描述](https://img-blog.csdn.net/20170512160734275?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
　　 
　　其中，HashMap与LinkedHashMap的Entry结构示意图如下图所示：

![这里写图片描述](https://img-blog.csdn.net/20170512155609530?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

　　特别地，由于LinkedHashMap是HashMap的子类，所以LinkedHashMap自然会拥有HashMap的所有特性。比如，==LinkedHashMap也最多只允许一条Entry的键为Null(多条会覆盖)，但允许多条Entry的值为Null。==
　　
　　此外，LinkedHashMap 也是 Map 的一个非同步的实现。此外，LinkedHashMap还可以用来实现LRU (Least recently used, 最近最少使用)算法，这个问题会在下文的特别谈到。

## LinkedHashMap 在 JDK 中的定义

### 类结构定义

　　LinkedHashMap继承于HashMap，其在JDK中的定义为：

    public class LinkedHashMap<K,V> extends HashMap<K,V>
        implements Map<K,V> {
    
        ...
    }

### 成员变量定义

　　与HashMap相比，LinkedHashMap增加了两个属性用于保证迭代顺序，分别是 双向链表头结点header 和 标志位accessOrder (值为true时，表示按照访问顺序迭代；值为false时，表示按照插入顺序迭代)。

    /**
     * The head of the doubly linked list.
     */
    private transient Entry<K,V> header;  // 双向链表的表头元素
    
    /**
     * The iteration ordering method for this linked hash map: <tt>true</tt>
     * for access-order, <tt>false</tt> for insertion-order.
     *
     * @serial
     */
    private final boolean accessOrder;  //true表示按照访问顺序迭代，false时表示按照插入顺序 

### 成员方法定义

　　从下图我们可以看出，LinkedHashMap中并增加没有额外方法。也就是说，LinkedHashMap与HashMap在操作上大致相同，只是在实现细节上略有不同罢了。

[外链图片转存失败(img-C2vYmjQ7-1567839753833)(http://static.zybuluo.com/Rico123/nvojgv4s0o0ciieibz1tbakc/LinkedHashMap_Outline.png)]

### 基本元素 Entry

　　LinkedHashMap采用的hash算法和HashMap相同，但是它重新定义了Entry。LinkedHashMap中的Entry增加了两个指针 before 和 after，它们分别用于维护双向链接列表。特别需要注意的是，next用于维护HashMap各个桶中Entry的连接顺序，before、after用于维护Entry插入的先后顺序的，源代码如下：

    private static class Entry<K,V> extends HashMap.Entry<K,V> {
    
        // These fields comprise the doubly linked list used for iteration.
        Entry<K,V> before, after;
    
        Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
            super(hash, key, value, next);
        }
        ...
    }

　　形象地，HashMap与LinkedHashMap的Entry结构示意图如下图所示：

![这里写图片描述](https://img-blog.csdn.net/20170512155609530?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## LinkedHashMap 的构造函数
　　LinkedHashMap 一共提供了五个构造函数，它们都是在HashMap的构造函数的基础上实现的，除了默认空参数构造方法，下面这个构造函数包含了大部分其他构造方法使用的参数，就不一一列举了。

### LinkedHashMap(int initialCapacity, float loadFactor, boolean accessOrder)

　　该构造函数意在构造一个指定初始容量和指定负载因子的具有指定迭代顺序的LinkedHashMap，其源码如下：

/**
     * Constructs an empty <tt>LinkedHashMap</tt> instance with the
     * specified initial capacity, load factor and ordering mode.
     *
     * @param  initialCapacity the initial capacity
     * @param  loadFactor      the load factor
     * @param  accessOrder     the ordering mode - <tt>true</tt> for
     *         access-order, <tt>false</tt> for insertion-order
     * @throws IllegalArgumentException if the initial capacity is negative
     *         or the load factor is nonpositive
     */
    public LinkedHashMap(int initialCapacity,
             float loadFactor,
                         boolean accessOrder) {
        super(initialCapacity, loadFactor);   // 调用HashMap对应的构造函数
        this.accessOrder = accessOrder;    // 迭代顺序的默认值
    }

初始容量 和负载因子是影响HashMap性能的两个重要参数。同样地，它们也是影响LinkedHashMap性能的两个重要参数。此外，LinkedHashMap 增加了双向链表头结点 header和标志位 accessOrder两个属性用于保证迭代顺序。
　　
### LinkedHashMap(Map<? extends K, ? extends V> m)

　　该构造函数意在构造一个与指定 Map 具有相同映射的 LinkedHashMap，其 初始容量不小于 16 (具体依赖于指定Map的大小)，负载因子是 0.75，是 Java Collection Framework 规范推荐提供的，其源码如下：

    /**
     * Constructs an insertion-ordered <tt>LinkedHashMap</tt> instance with
     * the same mappings as the specified map.  The <tt>LinkedHashMap</tt>
     * instance is created with a default load factor (0.75) and an initial
     * capacity sufficient to hold the mappings in the specified map.
     *
     * @param  m the map whose mappings are to be placed in this map
     * @throws NullPointerException if the specified map is null
     */
    public LinkedHashMap(Map<? extends K, ? extends V> m) {
        super(m);       // 调用HashMap对应的构造函数
        accessOrder = false;    // 迭代顺序的默认值
    }

### init 方法

　　从上面的五种构造函数我们可以看出，无论采用何种方式创建LinkedHashMap，其都会调用HashMap相应的构造函数。事实上，不管调用HashMap的哪个构造函数，HashMap的构造函数都会在最后调用一个init()方法进行初始化，只不过这个方法在HashMap中是一个空实现，而在LinkedHashMap中重写了它用于初始化它所维护的双向链表。例如，HashMap的参数为空的构造函数以及init方法的源码如下：

    /**
     * Constructs an empty <tt>HashMap</tt> with the default initial capacity
     * (16) and the default load factor (0.75).
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
        threshold = (int)(DEFAULT_INITIAL_CAPACITY * DEFAULT_LOAD_FACTOR);
        table = new Entry[DEFAULT_INITIAL_CAPACITY];
        init();
    }

   /**
     * Initialization hook for subclasses. This method is called
     * in all constructors and pseudo-constructors (clone, readObject)
     * after HashMap has been initialized but before any entries have
     * been inserted.  (In the absence of this method, readObject would
     * require explicit knowledge of subclasses.)
     */
    void init() {
    }

　　在LinkedHashMap中，它重写了init方法以便初始化双向列表，源码如下：

 /**
     * Called by superclass constructors and pseudoconstructors (clone,
     * readObject) before any entries are inserted into the map.  Initializes
     * the chain.
     */
    void init() {
        header = new Entry<K,V>(-1, null, null, null);
        header.before = header.after = header;
    }

　　因此，我们在创建LinkedHashMap的同时就会不知不觉地对双向链表进行初始化。

## LinkedHashMap 的数据结构

> 本质上，LinkedHashMap = HashMap + 双向链表，也就是说，HashMap和双向链表合二为一即是LinkedHashMap。

> 也可以这样理解，LinkedHashMap 在不对HashMap做任何改变的基础上，给HashMap的任意两个节点间加了两条连线(before指针和after指针)，使这些节点形成一个双向链表。

> 在LinkedHashMapMap中，所有put进来的Entry都保存在HashMap中，但由于它又额外定义了一个以head为头结点的空的双向链表，因此对于每次put进来Entry还会将其插入到双向链表的尾部。

　　　　　　　　　　　　![这里写图片描述](https://img-blog.csdn.net/20170512160734275?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvanVzdGxvdmV5b3Vf/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

## LinkedHashMap 的快速存取

　　我们知道，在HashMap中最常用的两个操作就是：put(Key,Value) 和 get(Key)。同样地，在 LinkedHashMap 中最常用的也是这两个操作。

对于put(Key,Value)方法而言，LinkedHashMap完全继承了HashMap的 put(Key,Value) 方法，只是对put(Key,Value)方法所调用的recordAccess方法和addEntry方法进行了重写；对于get(Key)方法而言，LinkedHashMap则直接对它进行了重写。

下面我们结合JDK源码看 LinkedHashMap 的存取实现。

### LinkedHashMap 的存储实现 : put(key, vlaue)

　　上面谈到，LinkedHashMap没有对 put(key,vlaue) 方法进行任何直接的修改，完全继承了HashMap的 put(Key,Value) 方法，其源码如下：


    public V put(K key, V value) {
    
        //当key为null时，调用putForNullKey方法，并将该键值对保存到table的第一个位置 
        if (key == null)
            return putForNullKey(value); 
    
        //根据key的hashCode计算hash值
        int hash = hash(key.hashCode());           
    
        //计算该键值对在数组中的存储位置（哪个桶）
        int i = indexFor(hash, table.length);              
    
        //在table的第i个桶上进行迭代，寻找 key 保存的位置
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {      
            Object k;
            //判断该条链上是否存在hash值相同且key值相等的映射，若存在，则直接覆盖 value，并返回旧value
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this); // LinkedHashMap重写了Entry中的recordAccess方法--- (1)    
                return oldValue;    // 返回旧值
            }
        }
    
        modCount++; //修改次数增加1，快速失败机制
    
        //原Map中无该映射，将该添加至该链的链头
        addEntry(hash, key, value, i);  // LinkedHashMap重写了HashMap中的createEntry方法 ---- (2)    
        return null;
    }

　　上述源码反映了LinkedHashMap与HashMap保存数据的过程。特别地，在LinkedHashMap中，它对addEntry方法和Entry的recordAccess方法进行了重写。下面我们对比地看一下LinkedHashMap 和HashMap的addEntry方法的具体实现：

    /**
     * This override alters behavior of superclass put method. It causes newly
     * allocated entry to get inserted at the end of the linked list and
     * removes the eldest entry if appropriate.
     *
     * LinkedHashMap中的addEntry方法
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {   
    
        //创建新的Entry，并插入到LinkedHashMap中  
        createEntry(hash, key, value, bucketIndex);  // 重写了HashMap中的createEntry方法
    
        //双向链表的第一个有效节点（header后的那个节点）为最近最少使用的节点，这是用来支持LRU算法的
        Entry<K,V> eldest = header.after;  
        //如果有必要，则删除掉该近期最少使用的节点，  
        //这要看对removeEldestEntry的覆写,由于默认为false，因此默认是不做任何处理的。  
        if (removeEldestEntry(eldest)) {  
            removeEntryForKey(eldest.key);  
        } else {  
            //扩容到原来的2倍  
            if (size >= threshold)  
                resize(2 * table.length);  
        }  
    } 
    
    -------------------------------我是分割线------------------------------------
    
     /**
     * Adds a new entry with the specified key, value and hash code to
     * the specified bucket.  It is the responsibility of this
     * method to resize the table if appropriate.
     *
     * Subclass overrides this to alter the behavior of put method.
     * 
     * HashMap中的addEntry方法
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {
        //获取bucketIndex处的Entry
        Entry<K,V> e = table[bucketIndex];
    
        //将新创建的 Entry 放入 bucketIndex 索引处，并让新的 Entry 指向原来的 Entry 
        table[bucketIndex] = new Entry<K,V>(hash, key, value, e);
    
        //若HashMap中元素的个数超过极限了，则容量扩大两倍
        if (size++ >= threshold)
            resize(2 * table.length);
    }

　　由于LinkedHashMap本身维护了插入的先后顺序，因此其可以用来做缓存，14~19行的操作就是用来支持LRU算法的，这里暂时不用去关心它。此外，在LinkedHashMap的addEntry方法中，它重写了HashMap中的createEntry方法，我们接着看一下createEntry方法：

    void createEntry(int hash, K key, V value, int bucketIndex) { 
        // 向哈希表中插入Entry，这点与HashMap中相同 
        //创建新的Entry并将其链入到数组对应桶的链表的头结点处， 
        HashMap.Entry<K,V> old = table[bucketIndex];  
        Entry<K,V> e = new Entry<K,V>(hash, key, value, old);  
        table[bucketIndex] = e;     
    
        //在每次向哈希表插入Entry的同时，都会将其插入到双向链表的尾部，  
        //这样就按照Entry插入LinkedHashMap的先后顺序来迭代元素(LinkedHashMap根据双向链表重写了迭代器)
        //同时，新put进来的Entry是最近访问的Entry，把其放在链表末尾 ，也符合LRU算法的实现  
        e.addBefore(header);  
        size++;  
    }  

　　由以上源码我们可以知道，在LinkedHashMap中向哈希表中插入新Entry的同时，还会通过Entry的addBefore方法将其链入到双向链表中。其中，addBefore方法本质上是一个双向链表的插入操作，其源码如下：

    //在双向链表中，将当前的Entry插入到existingEntry(header)的前面  
    private void addBefore(Entry<K,V> existingEntry) {  
        after  = existingEntry;  
        before = existingEntry.before;  
        before.after = this;  
        after.before = this;  
    }  

　　到此为止，我们分析了在LinkedHashMap中put一条键值对的完整过程。总的来说，相比HashMap而言，LinkedHashMap在向哈希表添加一个键值对的同时，也会将其链入到它所维护的双向链表中，以便设定迭代顺序。

### LinkedHashMap 的扩容操作 : resize()

在HashMap中，我们知道随着HashMap中元素的数量越来越多，发生碰撞的概率将越来越大，所产生的子链长度就会越来越长，这样势必会影响HashMap的存取速度。

为了保证HashMap的效率，系统必须要在某个临界点进行扩容处理，该临界点就是HashMap中元素的数量在数值上等于threshold（table数组长度*加载因子）。

但是，不得不说，扩容是一个非常耗时的过程，因为它需要重新计算这些元素在新table数组中的位置并进行复制处理。所以，如果我们能够提前预知HashMap中元素的个数，那么在构造HashMap时预设元素的个数能够有效的提高HashMap的性能。 

同样的问题也存在于LinkedHashMap中，因为LinkedHashMap本来就是一个HashMap，只是它还将所有Entry节点链入到了一个双向链表中。LinkedHashMap完全继承了HashMap的resize()方法，只是对它所调用的transfer方法进行了重写。我们先看resize()方法源码：

    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
    
        // 若 oldCapacity 已达到最大值，直接将 threshold 设为 Integer.MAX_VALUE
        if (oldCapacity == MAXIMUM_CAPACITY) {  
            threshold = Integer.MAX_VALUE;
            return;             // 直接返回
        }
    
        // 否则，创建一个更大的数组
        Entry[] newTable = new Entry[newCapacity];
    
        //将每条Entry重新哈希到新的数组中
        transfer(newTable);  //LinkedHashMap对它所调用的transfer方法进行了重写
    
        table = newTable;
        threshold = (int)(newCapacity * loadFactor);  // 重新设定 threshold
    }

　　从上面代码中我们可以看出，Map扩容操作的核心在于重哈希。所谓重哈希是指重新计算原HashMap中的元素在新table数组中的位置并进行复制处理的过程。鉴于性能和LinkedHashMap自身特点的考量，LinkedHashMap对重哈希过程(transfer方法)进行了重写，源码如下：

    /**
     * Transfers all entries to new table array.  This method is called
     * by superclass resize.  It is overridden for performance, as it is
     * faster to iterate using our linked list.
     */
    void transfer(HashMap.Entry[] newTable) {
        int newCapacity = newTable.length;
        // 与HashMap相比，借助于双向链表的特点进行重哈希使得代码更加简洁
        for (Entry<K,V> e = header.after; e != header; e = e.after) {
            int index = indexFor(e.hash, newCapacity);   // 计算每个Entry所在的桶
            // 将其链入桶中的链表
            e.next = newTable[index];
            newTable[index] = e;   
        }
    }

　　如上述源码所示，LinkedHashMap借助于自身维护的双向链表轻松地实现了重哈希操作。 

### LinkedHashMap 的读取实现 ：get(Object key)

　　相对于LinkedHashMap的存储而言，读取就显得比较简单了。LinkedHashMap中重写了HashMap中的get方法，源码如下：


    public V get(Object key) {
        // 根据key获取对应的Entry，若没有这样的Entry，则返回null
        Entry<K,V> e = (Entry<K,V>)getEntry(key); 
        if (e == null)      // 若不存在这样的Entry，直接返回
            return null;
        e.recordAccess(this);
        return e.value;
    }
    
    /**
         * Returns the entry associated with the specified key in the
         * HashMap.  Returns null if the HashMap contains no mapping
         * for the key.
         * 
         * HashMap 中的方法
         *     
         */
        final Entry<K,V> getEntry(Object key) {
            if (size == 0) {
                return null;
            }
    
            int hash = (key == null) ? 0 : hash(key);
            for (Entry<K,V> e = table[indexFor(hash, table.length)];
                 e != null;
                 e = e.next) {
                Object k;
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            }
            return null;
        }

　　在LinkedHashMap的get方法中，通过HashMap中的getEntry方法获取Entry对象。注意这里的recordAccess方法，如果链表中元素的排序规则是按照插入的先后顺序排序的话，该方法什么也不做；如果链表中元素的排序规则是按照访问的先后顺序排序的话，则将e移到链表的末尾处，笔者会在后文专门阐述这个问题。

另外，同样地，调用LinkedHashMap的get(Object key)方法后，若返回值是 NULL，则也存在如下两种可能：

该 key 对应的值就是 null;
HashMap 中不存在该 key。

### LinkedHashMap 存取小结

> LinkedHashMap的存取过程基本与HashMap基本类似，只是在细节实现上稍有不同，这是由LinkedHashMap本身的特性所决定的，因为它要额外维护一个双向链表用于保持迭代顺序。
>
> 在put操作上，虽然LinkedHashMap完全继承了HashMap的put操作，但是在细节上还是做了一定的调整，比如，在LinkedHashMap中向哈希表中插入新Entry的同时，还会通过Entry的addBefore方法将其链入到双向链表中。
>
> 在扩容操作上，虽然LinkedHashMap完全继承了HashMap的resize操作，但是鉴于性能和LinkedHashMap自身特点的考量，LinkedHashMap对其中的重哈希过程(transfer方法)进行了重写。在读取操作上，LinkedHashMap中重写了HashMap中的get方法，通过HashMap中的getEntry方法获取Entry对象。在此基础上，进一步获取指定键对应的值。

## LinkedHashMap 与 LRU(Least recently used，最近最少使用)算法

　　到此为止，我们已经分析完了LinkedHashMap的存取实现，这与HashMap大体相同。LinkedHashMap区别于HashMap最大的一个不同点是，前者是有序的，而后者是无序的。为此，LinkedHashMap增加了两个属性用于保证顺序，分别是双向链表头结点header和标志位accessOrder。

我们知道，header是LinkedHashMap所维护的双向链表的头结点，而accessOrder用于决定具体的迭代顺序。实际上，accessOrder标志位的作用可不像我们描述的这样简单，我们接下来仔细分析一波~ 

> 我们知道，当accessOrder标志位为true时，表示双向链表中的元素按照访问的先后顺序排列，可以看到，虽然Entry插入链表的顺序依然是按照其put到LinkedHashMap中的顺序，但put和get方法均有调用recordAccess方法（put方法在key相同时会调用）。
>
> recordAccess方法判断accessOrder是否为true，如果是，则将当前访问的Entry（put进来的Entry或get出来的Entry）移到双向链表的尾部（key不相同时，put新Entry时，会调用addEntry，它会调用createEntry，该方法同样将新插入的元素放入到双向链表的尾部，既符合插入的先后顺序，又符合访问的先后顺序，因为这时该Entry也被访问了）；
>
> 当标志位accessOrder的值为false时，表示双向链表中的元素按照Entry插入LinkedHashMap到中的先后顺序排序，即每次put到LinkedHashMap中的Entry都放在双向链表的尾部，这样遍历双向链表时，Entry的输出顺序便和插入的顺序一致，这也是默认的双向链表的存储顺序。

因此，当标志位accessOrder的值为false时，虽然也会调用recordAccess方法，但不做任何操作。

### put操作与标志位accessOrder

    / 将key/value添加到LinkedHashMap中      
    public V put(K key, V value) {      
        // 若key为null，则将该键值对添加到table[0]中。      
        if (key == null)      
            return putForNullKey(value);      
        // 若key不为null，则计算该key的哈希值，然后将其添加到该哈希值对应的链表中。      
        int hash = hash(key.hashCode());      
        int i = indexFor(hash, table.length);      
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {      
            Object k;      
            // 若key对已经存在，则用新的value取代旧的value     
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {      
                V oldValue = e.value;      
                e.value = value;      
                e.recordAccess(this);      
                return oldValue;      
            }      
        }      
    
        // 若key不存在，则将key/value键值对添加到table中      
        modCount++;    
        //将key/value键值对添加到table[i]处    
        addEntry(hash, key, value, i);      
        return null;      
    }      

　　从上述源码我们可以看到，当要put进来的Entry的key在哈希表中已经在存在时，会调用Entry的recordAccess方法；当该key不存在时，则会调用addEntry方法将新的Entry插入到对应桶的单链表的头部。我们先来看recordAccess方法：

    /**
    * This method is invoked by the superclass whenever the value
    * of a pre-existing entry is read by Map.get or modified by Map.set.
    * If the enclosing Map is access-ordered, it moves the entry
    * to the end of the list; otherwise, it does nothing.
    */
    void recordAccess(HashMap<K,V> m) {  
        LinkedHashMap<K,V> lm = (LinkedHashMap<K,V>)m;  
        //如果链表中元素按照访问顺序排序，则将当前访问的Entry移到双向循环链表的尾部，  
        //如果是按照插入的先后顺序排序，则不做任何事情。  
        if (lm.accessOrder) {  
            lm.modCount++;  
            //移除当前访问的Entry  
            remove();  
            //将当前访问的Entry插入到链表的尾部  
            addBefore(lm.header);  
          }  
      } 

　　LinkedHashMap重写了HashMap中的recordAccess方法（HashMap中该方法为空），当调用父类的put方法时，在发现key已经存在时，会调用该方法；当调用自己的get方法时，也会调用到该方法。

该方法提供了LRU算法的实现，它将最近使用的Entry放到双向循环链表的尾部。也就是说，当accessOrder为true时，get方法和put方法都会调用recordAccess方法使得最近使用的Entry移到双向链表的末尾；当accessOrder为默认值false时，从源码中可以看出recordAccess方法什么也不会做。我们反过头来，再看一下addEntry方法：

   /**
     * This override alters behavior of superclass put method. It causes newly
     * allocated entry to get inserted at the end of the linked list and
     * removes the eldest entry if appropriate.
     *
     * LinkedHashMap中的addEntry方法
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {   

        //创建新的Entry，并插入到LinkedHashMap中  
        createEntry(hash, key, value, bucketIndex);  // 重写了HashMap中的createEntry方法
    
        //双向链表的第一个有效节点（header后的那个节点）为最近最少使用的节点，这是用来支持LRU算法的
        Entry<K,V> eldest = header.after;  
        //如果有必要，则删除掉该近期最少使用的节点，  
        //这要看对removeEldestEntry的覆写,由于默认为false，因此默认是不做任何处理的。  
        if (removeEldestEntry(eldest)) {  
            removeEntryForKey(eldest.key);  
        } else {  
            //扩容到原来的2倍  
            if (size >= threshold)  
                resize(2 * table.length);  
        }  
    } 
    
    void createEntry(int hash, K key, V value, int bucketIndex) { 
        // 向哈希表中插入Entry，这点与HashMap中相同 
        //创建新的Entry并将其链入到数组对应桶的链表的头结点处， 
        HashMap.Entry<K,V> old = table[bucketIndex];  
        Entry<K,V> e = new Entry<K,V>(hash, key, value, old);  
        table[bucketIndex] = e;     
    
        //在每次向哈希表插入Entry的同时，都会将其插入到双向链表的尾部，  
        //这样就按照Entry插入LinkedHashMap的先后顺序来迭代元素(LinkedHashMap根据双向链表重写了迭代器)
        //同时，新put进来的Entry是最近访问的Entry，把其放在链表末尾 ，也符合LRU算法的实现  
        e.addBefore(header);  
        size++;  
    }

　　同样是将新的Entry链入到table中对应桶中的单链表中，但可以在createEntry方法中看出，同时也会把新put进来的Entry插入到了双向链表的尾部。
　　
从插入顺序的层面来说，新的Entry插入到双向链表的尾部可以实现按照插入的先后顺序来迭代Entry，而从访问顺序的层面来说，新put进来的Entry又是最近访问的Entry，也应该将其放在双向链表的尾部。在上面的addEntry方法中还调用了removeEldestEntry方法，该方法源码如下：

    /**
     * Returns <tt>true</tt> if this map should remove its eldest entry.
     * This method is invoked by <tt>put</tt> and <tt>putAll</tt> after
     * inserting a new entry into the map.  It provides the implementor
     * with the opportunity to remove the eldest entry each time a new one
     * is added.  This is useful if the map represents a cache: it allows
     * the map to reduce memory consumption by deleting stale entries.
     *
     * <p>Sample use: this override will allow the map to grow up to 100
     * entries and then delete the eldest entry each time a new entry is
     * added, maintaining a steady state of 100 entries.
     * <pre>
     *     private static final int MAX_ENTRIES = 100;
     *
     *     protected boolean removeEldestEntry(Map.Entry eldest) {
     *        return size() > MAX_ENTRIES;
     *     }
     * </pre>
     *
     * <p>This method typically does not modify the map in any way,
     * instead allowing the map to modify itself as directed by its
     * return value.  It <i>is</i> permitted for this method to modify
     * the map directly, but if it does so, it <i>must</i> return
     * <tt>false</tt> (indicating that the map should not attempt any
     * further modification).  The effects of returning <tt>true</tt>
     * after modifying the map from within this method are unspecified.
     *
     * <p>This implementation merely returns <tt>false</tt> (so that this
     * map acts like a normal map - the eldest element is never removed).
     *
     * @param    eldest The least recently inserted entry in the map, or if
     *           this is an access-ordered map, the least recently accessed
     *           entry.  This is the entry that will be removed it this
     *           method returns <tt>true</tt>.  If the map was empty prior
     *           to the <tt>put</tt> or <tt>putAll</tt> invocation resulting
     *           in this invocation, this will be the entry that was just
     *           inserted; in other words, if the map contains a single
     *           entry, the eldest entry is also the newest.
     * @return   <tt>true</tt> if the eldest entry should be removed
     *           from the map; <tt>false</tt> if it should be retained.
     */
    protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
        return false;
    }
}

　　该方法是用来被重写的，一般地，如果用LinkedHashmap实现LRU算法，就要重写该方法。比如可以将该方法覆写为如果设定的内存已满，则返回true，这样当再次向LinkedHashMap中putEntry时，在调用的addEntry方法中便会将近期最少使用的节点删除掉（header后的那个节点）。在第七节，笔者便重写了该方法并实现了一个名副其实的LRU结构。

### get操作与标志位accessOrder

    public V get(Object key) {
        // 根据key获取对应的Entry，若没有这样的Entry，则返回null
        Entry<K,V> e = (Entry<K,V>)getEntry(key); 
        if (e == null)      // 若不存在这样的Entry，直接返回
            return null;
        e.recordAccess(this);
        return e.value;
    }

　　在LinkedHashMap中进行读取操作时，一样也会调用recordAccess方法。上面笔者已经表述的很清楚了，此不赘述。

### LinkedListMap与LRU小结

　　使用LinkedHashMap实现LRU的必要前提是将accessOrder标志位设为true以便开启按访问顺序排序的模式。我们可以看到，无论是put方法还是get方法，都会导致目标Entry成为最近访问的Entry，因此就把该Entry加入到了双向链表的末尾：get方法通过调用recordAccess方法来实现；

put方法在覆盖已有key的情况下，也是通过调用recordAccess方法来实现，在插入新的Entry时，则是通过createEntry中的addBefore方法来实现。这样，我们便把最近使用的Entry放入到了双向链表的后面。多次操作后，双向链表前面的Entry便是最近没有使用的，这样当节点个数满的时候，删除最前面的Entry(head后面的那个Entry)即可，因为它就是最近最少使用的Entry。

## 使用LinkedHashMap实现LRU算法
　　如下所示，笔者使用LinkedHashMap实现一个符合LRU算法的数据结构，该结构最多可以缓存6个元素，但元素多余六个时，会自动删除最近最久没有被使用的元素，如下所示：


    public class LRU<K,V> extends LinkedHashMap<K, V> implements Map<K, V>{
    
        private static final long serialVersionUID = 1L;
    
        public LRU(int initialCapacity,
                 float loadFactor,
                            boolean accessOrder) {
            super(initialCapacity, loadFactor, accessOrder);
        }
    
        /** 
         * @description 重写LinkedHashMap中的removeEldestEntry方法，当LRU中元素多余6个时，
         *              删除最不经常使用的元素
         * @author rico       
         * @created 2017年5月12日 上午11:32:51      
         * @param eldest
         * @return     
         * @see java.util.LinkedHashMap#removeEldestEntry(java.util.Map.Entry)     
         */  
        @Override
        protected boolean removeEldestEntry(java.util.Map.Entry<K, V> eldest) {
            // TODO Auto-generated method stub
            if(size() > 6){
                return true;
            }
            return false;
        }
    
        public static void main(String[] args) {
    
            LRU<Character, Integer> lru = new LRU<Character, Integer>(
                    16, 0.75f, true);
    
            String s = "abcdefghijkl";
            for (int i = 0; i < s.length(); i++) {
                lru.put(s.charAt(i), i);
            }
            System.out.println("LRU中key为h的Entry的值为： " + lru.get('h'));
            System.out.println("LRU的大小 ：" + lru.size());
            System.out.println("LRU ：" + lru);
        }
    }

　　下图是程序的运行结果： 
![](http://static.zybuluo.com/Rico123/gjz8mjvhkkhwjlzr5o8b27yv/LRU.png)
## LinkedHashMap 有序性原理分析

如前文所述，LinkedHashMap 增加了双向链表头结点header 和 标志位accessOrder两个属性用于保证迭代顺序。但是要想真正实现其有序性，还差临门一脚，那就是重写HashMap 的迭代器，其源码实现如下：

    private abstract class LinkedHashIterator<T> implements Iterator<T> {
        Entry<K,V> nextEntry    = header.after;
        Entry<K,V> lastReturned = null;
    
        /**
         * The modCount value that the iterator believes that the backing
         * List should have.  If this expectation is violated, the iterator
         * has detected concurrent modification.
         */
        int expectedModCount = modCount;
    
        public boolean hasNext() {         // 根据双向列表判断 
                return nextEntry != header;
        }
    
        public void remove() {
            if (lastReturned == null)
            throw new IllegalStateException();
            if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    
                LinkedHashMap.this.remove(lastReturned.key);
                lastReturned = null;
                expectedModCount = modCount;
        }
    
        Entry<K,V> nextEntry() {        // 迭代输出双向链表各节点
            if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
                if (nextEntry == header)
                    throw new NoSuchElementException();
    
                Entry<K,V> e = lastReturned = nextEntry;
                nextEntry = e.after;
                return e;
        }
    }
    
    // Key 迭代器，KeySet
    private class KeyIterator extends LinkedHashIterator<K> {   
        public K next() { return nextEntry().getKey(); }
    }
    
       // Value 迭代器，Values(Collection)
    private class ValueIterator extends LinkedHashIterator<V> {
        public V next() { return nextEntry().value; }
    }
    
    // Entry 迭代器，EntrySet
    private class EntryIterator extends LinkedHashIterator<Map.Entry<K,V>> {
        public Map.Entry<K,V> next() { return nextEntry(); }
    }

　 从上述代码中我们可以知道，LinkedHashMap重写了HashMap 的迭代器，它使用其维护的双向链表进行迭代输出。
　 
## JDK1.8的改动

原文是基于JDK1.6的实现，实际上JDK1.8对其进行了改动。
首先它删除了addentry，createenrty等方法（事实上是hashmap的改动影响了它而已）。

linkedhashmap同样使用了大部分hashmap的增删改查方法。
新版本linkedhashmap主要是通过对hashmap内置几个方法重写来实现lru的。

hashmap不提供实现：

     void afterNodeAccess(Node<K,V> p) { }
        void afterNodeInsertion(boolean evict) { }
        void afterNodeRemoval(Node<K,V> p) { }

linkedhashmap的实现：
    
处理元素被访问后的情况

    void afterNodeAccess(Node<K,V> e) { // move node to last
            LinkedHashMap.Entry<K,V> last;
            if (accessOrder && (last = tail) != e) {
                LinkedHashMap.Entry<K,V> p =
                    (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
                p.after = null;
                if (b == null)
                    head = a;
                else
                    b.after = a;
                if (a != null)
                    a.before = b;
                else
                    last = b;
                if (last == null)
                    head = p;
                else {
                    p.before = last;
                    last.after = p;
                }
                tail = p;
                ++modCount;
            }
        }

处理元素插入后的情况


        void afterNodeInsertion(boolean evict) { // possibly remove eldest
        LinkedHashMap.Entry<K,V> first;
        if (evict && (first = head) != null && removeEldestEntry(first)) {
            K key = first.key;
            removeNode(hash(key), key, null, false, true);
        }

处理元素被删除后的情况

         void afterNodeRemoval(Node<K,V> e) { // unlink
        LinkedHashMap.Entry<K,V> p =
            (LinkedHashMap.Entry<K,V>)e, b = p.before, a = p.after;
        p.before = p.after = null;
        if (b == null)
            head = a;
        else
            b.after = a;
        if (a == null)
            tail = b;
        else
            a.before = b;
    }
    }

另外1.8的hashmap在链表长度超过8时自动转为红黑树，会按顺序插入链表中的元素，可以自定义比较器来定义节点的插入顺序。

1.8的linkedhashmap同样会使用这一特性，当变为红黑树以后，节点的先后顺序同样是插入红黑树的顺序，其双向链表的性质没有改表，只是原来hashmap的链表变成了红黑树而已，在此不要混淆。

## 总结
> 本文从linkedhashmap的数据结构，以及源码分析，到最后的LRU缓存实现，比较深入地剖析了linkedhashmap的底层原理。
> 总结以下几点：
> 
> 1 linkedhashmap在hashmap的数组加链表结构的基础上，将所有节点连成了一个双向链表。
> 
> 2 当主动传入的accessOrder参数为false时, 使用put方法时，新加入元素不会被加入双向链表，get方法使用时也不会把元素放到双向链表尾部。
> 
> 3 当主动传入的accessOrder参数为true时，使用put方法新加入的元素，如果遇到了哈希冲突，并且对key值相同的元素进行了替换，就会被放在双向链表的尾部，当元素超过上限且removeEldestEntry方法返回true时，直接删除最早元素以便新元素插入。如果没有冲突直接放入，同样加入到链表尾部。使用get方法时会把get到的元素放入双向链表尾部。
> 
> 4 linkedhashmap的扩容比hashmap来的方便，因为hashmap需要将原来的每个链表的元素分别在新数组进行反向插入链化，而linkedhashmap的元素都连在一个链表上，可以直接迭代然后插入。
> 
> 5 linkedhashmap的removeEldestEntry方法默认返回false，要实现lru很重要的一点就是集合满时要将最久未访问的元素删除，在linkedhashmap中这个元素就是头指针指向的元素。实现LRU可以直接实现继承linkedhashmap并重写removeEldestEntry方法来设置缓存大小。jdk中实现了LRUCache也可以直接使用。

## 参考文章
http://cmsblogs.com/?p=176

https://www.jianshu.com/p/8f4f58b4b8ab

https://blog.csdn.net/wang_8101/article/details/83067860

https://www.cnblogs.com/create-and-orange/p/11237072.html

https://www.cnblogs.com/ganchuanpu/p/8908093.html
## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)


