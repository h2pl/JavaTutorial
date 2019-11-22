# 目录
  * [集合类大图](#集合类大图)
  * [Collection接口](#collection接口)
  * [List接口](#list接口)
  * [Set接口](#set接口)
  * [Map接口](#map接口)
  * [Queue](#queue)
  * [关于Java集合的小抄](#关于java集合的小抄)
    * [List](#list)
      * [ArrayList](#arraylist)
      * [LinkedList](#linkedlist)
      * [CopyOnWriteArrayList](#copyonwritearraylist)
      * [遗憾](#遗憾)
    * [Map](#map)
      * [HashMap](#hashmap)
      * [LinkedHashMap](#linkedhashmap)
      * [TreeMap](#treemap)
      * [EnumMap](#enummap)
      * [ConcurrentHashMap](#concurrenthashmap)
      * [ConcurrentSkipListMap](#concurrentskiplistmap)
    * [Set](#set)
    * [Queue](#queue-1)
      * [普通队列](#普通队列)
      * [PriorityQueue](#priorityqueue)
      * [线程安全的队列](#线程安全的队列)
      * [线程安全的阻塞队列](#线程安全的阻塞队列)
      * [同步队列](#同步队列)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)
---
title: 夯实Java基础系列19：一文搞懂Java集合类框架，以及常见面试题
date: 2019-9-19  15:56:26 # 文章生成时间，一般不改
categories:
    - Java技术江湖
    - Java基础
tags:
    - Java集合类
---

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《夯实Java基础系列博文》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。
该系列博文会告诉你如何从入门到进阶，一步步地学习Java基础知识，并上手进行实战，接着了解每个Java知识点背后的实现原理，更完整地了解整个Java技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。


<!-- more -->

本文参考 https://www.cnblogs.com/chenssy/p/3495238.html

## 集合类大图

在编写java程序中，我们最常用的除了八种基本数据类型，String对象外还有一个集合类，在我们的的程序中到处充斥着集合类的身影！

![image](https://img-blog.csdnimg.cn/20190222094403579.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3p6dzE1MzE0MzkwOTA=,size_16,color_FFFFFF,t_70)

java中集合大家族的成员实在是太丰富了，有常用的ArrayList、HashMap、HashSet，也有不常用的Stack、Queue，有线程安全的Vector、HashTable，也有线程不安全的LinkedList、TreeMap等等！

![](https://images0.cnblogs.com/blog/381060/201312/28124707-3a873160808e457686d67c118af6fa70.png)

上面的图展示了整个集合大家族的成员以及他们之间的关系。下面就上面的各个接口、基类做一些简单的介绍(主要介绍各个集合的特点。区别)。

下面几张图更清晰地介绍了结合类接口间的关系：

> Collections和Collection。
> Arrays和Collections。
>
> ![](https://www.programcreek.com/wp-content/uploads/2009/02/CollectionVsCollections.jpeg)

> Collection的子接口

![](https://www.programcreek.com/wp-content/uploads/2009/02/java-collection-hierarchy.jpeg)
> map的实现类

![](https://www.programcreek.com/wp-content/uploads/2009/02/MapClassHierarchy-600x354.jpg)

## Collection接口

  Collection接口是最基本的集合接口，它不提供直接的实现，Java SDK提供的类都是继承自Collection的“子接口”如List和Set。Collection所代表的是一种规则，它所包含的元素都必须遵循一条或者多条规则。如有些允许重复而有些则不能重复、有些必须要按照顺序插入而有些则是散列，有些支持排序但是有些则不支持。

  在Java中所有实现了Collection接口的类都必须提供两套标准的构造函数，一个是无参，用于创建一个空的Collection，一个是带有Collection参数的有参构造函数，用于创建一个新的Collection，这个新的Collection与传入进来的Collection具备相同的元素。
//要求实现基本的增删改查方法，并且需要能够转换为数组类型


    public class Collection接口 {
        class collect implements Collection {
    
            @Override
            public int size() {
                return 0;
            }
    
            @Override
            public boolean isEmpty() {
                return false;
            }
    
            @Override
            public boolean contains(Object o) {
                return false;
            }
    
            @Override
            public Iterator iterator() {
                return null;
            }
    
            @Override
            public Object[] toArray() {
                return new Object[0];
            }
    
            @Override
            public boolean add(Object o) {
                return false;
            }
    
            @Override
            public boolean remove(Object o) {
                return false;
            }
    
            @Override
            public boolean addAll(Collection c) {
                return false;
            }
    
            @Override
            public void clear() {
    
            }
    //省略部分代码  
    
            @Override
            public Object[] toArray(Object[] a) {
                return new Object[0];
            }
        }
    }

## List接口

>   List接口为Collection直接接口。List所代表的是有序的Collection，即它用某种特定的插入顺序来维护元素顺序。用户可以对列表中每个元素的插入位置进行精确地控制，同时可以根据元素的整数索引（在列表中的位置）访问元素，并搜索列表中的元素。实现List接口的集合主要有：ArrayList、LinkedList、Vector、Stack。

2.1、ArrayList

>   ArrayList是一个动态数组，也是我们最常用的集合。它允许任何符合规则的元素插入甚至包括null。每一个ArrayList都有一个初始容量（10），该容量代表了数组的大小。随着容器中的元素不断增加，容器的大小也会随着增加。在每次向容器中增加元素的同时都会进行容量检查，当快溢出时，就会进行扩容操作。所以如果我们明确所插入元素的多少，最好指定一个初始容量值，避免过多的进行扩容操作而浪费时间、效率。
>
>   size、isEmpty、get、set、iterator 和 listIterator 操作都以固定时间运行。add 操作以分摊的固定时间运行，也就是说，添加 n 个元素需要 O(n) 时间（由于要考虑到扩容，所以这不只是添加元素会带来分摊固定时间开销那样简单）。
>
>   ArrayList擅长于随机访问。同时ArrayList是非同步的。
>
>   2.2、LinkedList

>   同样实现List接口的LinkedList与ArrayList不同，ArrayList是一个动态数组，而LinkedList是一个双向链表。所以它除了有ArrayList的基本操作方法外还额外提供了get，remove，insert方法在LinkedList的首部或尾部。
>
>   由于实现的方式不同，LinkedList不能随机访问，它所有的操作都是要按照双重链表的需要执行。在列表中索引的操作将从开头或结尾遍历列表（从靠近指定索引的一端）。这样做的好处就是可以通过较低的代价在List中进行插入和删除操作。
>
>   与ArrayList一样，LinkedList也是非同步的。如果多个线程同时访问一个List，则必须自己实现访问同步。一种解决方法是在创建List时构造一个同步的List： 
>   List list = Collections.synchronizedList(new LinkedList(...));

> 2.3、Vector
>    与ArrayList相似，但是Vector是同步的。所以说Vector是线程安全的动态数组。它的操作与ArrayList几乎一样。
>
> 2.4、Stack
>    Stack继承自Vector，实现一个后进先出的堆栈。Stack提供5个额外的方法使得Vector得以被当作堆栈使用。基本的push和pop 方法，还有peek方法得到栈顶的元素，empty方法测试堆栈是否为空，search方法检测一个元素在堆栈中的位置。Stack刚创建后是空栈。。

    public class List接口 {
        //下面是List的继承关系，由于List接口规定了包括诸如索引查询，迭代器的实现，所以实现List接口的类都会有这些方法。
        //所以不管是ArrayList和LinkedList底层都可以使用数组操作，但一般不提供这样外部调用方法。
        //    public interface Iterable<T>
    //    public interface Collection<E> extends Iterable<E>
    //    public interface List<E> extends Collection<E>
        class MyList implements List {
    
            @Override
            public int size() {
                return 0;
            }
    
            @Override
            public boolean isEmpty() {
                return false;
            }
    
            @Override
            public boolean contains(Object o) {
                return false;
            }
    
            @Override
            public Iterator iterator() {
                return null;
            }
    
            @Override
            public Object[] toArray() {
                return new Object[0];
            }
    
            @Override
            public boolean add(Object o) {
                return false;
            }
    
            @Override
            public boolean remove(Object o) {
                return false;
            }
    
            @Override
            public void clear() {
    
            }
    
           //省略部分代码
           
            @Override
            public Object get(int index) {
                return null;
            }
    
            @Override
            public ListIterator listIterator() {
                return null;
            }
    
            @Override
            public ListIterator listIterator(int index) {
                return null;
            }
    
            @Override
            public List subList(int fromIndex, int toIndex) {
                return null;
            }
    
            @Override
            public Object[] toArray(Object[] a) {
                return new Object[0];
            }
        }
    }

## Set接口

> Set是一种不包括重复元素的Collection。它维持它自己的内部排序，所以随机访问没有任何意义。与List一样，它同样运行null的存在但是仅有一个。由于Set接口的特殊性，所有传入Set集合中的元素都必须不同，同时要注意任何可变对象，如果在对集合中元素进行操作时，导致e1.equals(e2)==true，则必定会产生某些问题。实现了Set接口的集合有：EnumSet、HashSet、TreeSet。
>
> 3.1、EnumSet
>    是枚举的专用Set。所有的元素都是枚举类型。
>
> 3.2、HashSet
>    HashSet堪称查询速度最快的集合，因为其内部是以HashCode来实现的。它内部元素的顺序是由哈希码来决定的，所以它不保证set 的迭代顺序；特别是它不保证该顺序恒久不变。

    public class Set接口 {
        // Set接口规定将set看成一个集合，并且使用和数组类似的增删改查方式，同时提供iterator迭代器
        //    public interface Set<E> extends Collection<E>
        //    public interface Collection<E> extends Iterable<E>
        //    public interface Iterable<T>
        class MySet implements Set {
    
            @Override
            public int size() {
                return 0;
            }
    
            @Override
            public boolean isEmpty() {
                return false;
            }
    
            @Override
            public boolean contains(Object o) {
                return false;
            }
    
            @Override
            public Iterator iterator() {
                return null;
            }
    
            @Override
            public Object[] toArray() {
                return new Object[0];
            }
    
            @Override
            public boolean add(Object o) {
                return false;
            }
    
            @Override
            public boolean remove(Object o) {
                return false;
            }
    
            @Override
            public boolean addAll(Collection c) {
                return false;
            }
    
            @Override
            public void clear() {
    
            }
    
            @Override
            public boolean removeAll(Collection c) {
                return false;
            }
    
            @Override
            public boolean retainAll(Collection c) {
                return false;
            }
    
            @Override
            public boolean containsAll(Collection c) {
                return false;
            }
    
            @Override
            public Object[] toArray(Object[] a) {
                return new Object[0];
            }
        }
    }

## Map接口

>   Map与List、Set接口不同，它是由一系列键值对组成的集合，提供了key到Value的映射。同时它也没有继承Collection。在Map中它保证了key与value之间的一一对应关系。也就是说一个key对应一个value，所以它不能存在相同的key值，当然value值可以相同。实现map的有：HashMap、TreeMap、HashTable、Properties、EnumMap。

> 4.1、HashMap
>    以哈希表数据结构实现，查找对象时通过哈希函数计算其位置，它是为快速查询而设计的，其内部定义了一个hash表数组（Entry[] table），元素会通过哈希转换函数将元素的哈希地址转换成数组中存放的索引，如果有冲突，则使用散列链表的形式将所有相同哈希地址的元素串起来，可能通过查看HashMap.Entry的源码它是一个单链表结构。
>
> 4.2、TreeMap
>    键以某种排序规则排序，内部以red-black（红-黑）树数据结构实现，实现了SortedMap接口
>
> 4.3、HashTable
>    也是以哈希表数据结构实现的，解决冲突时与HashMap也一样也是采用了散列链表的形式，不过性能比HashMap要低

    public class Map接口 {
        //Map接口是最上层接口，Map接口实现类必须实现put和get等哈希操作。
        //并且要提供keyset和values，以及entryset等查询结构。
        //public interface Map<K,V>
        class MyMap implements Map {
    
            @Override
            public int size() {
                return 0;
            }
    
            @Override
            public boolean isEmpty() {
                return false;
            }
    
            @Override
            public boolean containsKey(Object key) {
                return false;
            }
    
            @Override
            public boolean containsValue(Object value) {
                return false;
            }
    
            @Override
            public Object get(Object key) {
                return null;
            }
    
            @Override
            public Object put(Object key, Object value) {
                return null;
            }
    
            @Override
            public Object remove(Object key) {
                return null;
            }
    
            @Override
            public void putAll(Map m) {
    
            }
    
            @Override
            public void clear() {
    
            }
    
            @Override
            public Set keySet() {
                return null;
            }
    
            @Override
            public Collection values() {
                return null;
            }
    
            @Override
            public Set<Entry> entrySet() {
                return null;
            }
        }
    }

## Queue

>   队列，它主要分为两大类，一类是阻塞式队列，队列满了以后再插入元素则会抛出异常，主要包括ArrayBlockQueue、PriorityBlockingQueue、LinkedBlockingQueue。另一种队列则是双端队列，支持在头、尾两端插入和移除元素，主要包括：ArrayDeque、LinkedBlockingDeque、LinkedList。

    public class Queue接口 {
        //queue接口是对队列的一个实现，需要提供队列的进队出队等方法。一般使用linkedlist作为实现类
        class MyQueue implements Queue {
    
            @Override
            public int size() {
                return 0;
            }
    
            @Override
            public boolean isEmpty() {
                return false;
            }
    
            @Override
            public boolean contains(Object o) {
                return false;
            }
    
            @Override
            public Iterator iterator() {
                return null;
            }
    
            @Override
            public Object[] toArray() {
                return new Object[0];
            }
    
            @Override
            public Object[] toArray(Object[] a) {
                return new Object[0];
            }
    
            @Override
            public boolean add(Object o) {
                return false;
            }
    
            @Override
            public boolean remove(Object o) {
                return false;
            }
    
            //省略部分代码
            @Override
            public boolean offer(Object o) {
                return false;
            }
    
            @Override
            public Object remove() {
                return null;
            }
    
            @Override
            public Object poll() {
                return null;
            }
    
            @Override
            public Object element() {
                return null;
            }
    
            @Override
            public Object peek() {
                return null;
            }
        }
    }


## 关于Java集合的小抄

这部分内容转自我偶像 江南白衣 的博客：http://calvin1978.blogcn.com/articles/collection.html
在尽可能短的篇幅里，将所有集合与并发集合的特征、实现方式、性能捋一遍。适合所有"精通Java"，其实还不那么自信的人阅读。

期望能不止用于面试时，平时选择数据结构，也能考虑一下其成本与效率，不要看着API合适就用了。

### List

#### ArrayList
以数组实现。节约空间，但数组有容量限制。超出限制时会增加50%容量，用System.arraycopy（）复制到新的数组。因此最好能给出数组大小的预估值。默认第一次插入元素时创建大小为10的数组。

按数组下标访问元素－get（i）、set（i,e） 的性能很高，这是数组的基本优势。

如果按下标插入元素、删除元素－add（i,e）、 remove（i）、remove（e），则要用System.arraycopy（）来复制移动部分受影响的元素，性能就变差了。

越是前面的元素，修改时要移动的元素越多。直接在数组末尾加入元素－常用的add（e），删除最后一个元素则无影响。

 

#### LinkedList
以双向链表实现。链表无容量限制，但双向链表本身使用了更多空间，每插入一个元素都要构造一个额外的Node对象，也需要额外的链表指针操作。

 

按下标访问元素－get（i）、set（i,e） 要悲剧的部分遍历链表将指针移动到位 （如果i>数组大小的一半，会从末尾移起）。

插入、删除元素时修改前后节点的指针即可，不再需要复制移动。但还是要部分遍历链表的指针才能移动到下标所指的位置。

只有在链表两头的操作－add（）、addFirst（）、removeLast（）或用iterator（）上的remove（）倒能省掉指针的移动。

Apache Commons 有个TreeNodeList，里面是棵二叉树，可以快速移动指针到位。

 

#### CopyOnWriteArrayList
并发优化的ArrayList。基于不可变对象策略，在修改时先复制出一个数组快照来修改，改好了，再让内部指针指向新数组。

因为对快照的修改对读操作来说不可见，所以读读之间不互斥，读写之间也不互斥，只有写写之间要加锁互斥。但复制快照的成本昂贵，典型的适合读多写少的场景。

虽然增加了addIfAbsent（e）方法，会遍历数组来检查元素是否已存在，性能可想像的不会太好。

 

#### 遗憾
无论哪种实现，按值返回下标contains（e）, indexOf（e）, remove（e） 都需遍历所有元素进行比较，性能可想像的不会太好。

没有按元素值排序的SortedList。

除了CopyOnWriteArrayList，再没有其他线程安全又并发优化的实现如ConcurrentLinkedList。凑合着用Set与Queue中的等价类时，会缺少一些List特有的方法如get（i）。如果更新频率较高，或数组较大时，还是得用Collections.synchronizedList（list），对所有操作用同一把锁来保证线程安全。

### Map
#### HashMap


以Entry[]数组实现的哈希桶数组，用Key的哈希值取模桶数组的大小可得到数组下标。

插入元素时，如果两条Key落在同一个桶（比如哈希值1和17取模16后都属于第一个哈希桶），我们称之为哈希冲突。

JDK的做法是链表法，Entry用一个next属性实现多个Entry以单向链表存放。查找哈希值为17的key时，先定位到哈希桶，然后链表遍历桶里所有元素，逐个比较其Hash值然后key值。

在JDK8里，新增默认为8的阈值，当一个桶里的Entry超过閥值，就不以单向链表而以红黑树来存放以加快Key的查找速度。

当然，最好还是桶里只有一个元素，不用去比较。所以默认当Entry数量达到桶数量的75%时，哈希冲突已比较严重，就会成倍扩容桶数组，并重新分配所有原来的Entry。扩容成本不低，所以也最好有个预估值。

取模用与操作（hash & （arrayLength-1））会比较快，所以数组的大小永远是2的N次方， 你随便给一个初始值比如17会转为32。默认第一次放入元素时的初始值是16。

iterator（）时顺着哈希桶数组来遍历，看起来是个乱序。

 

#### LinkedHashMap
扩展HashMap，每个Entry增加双向链表，号称是最占内存的数据结构。

支持iterator（）时按Entry的插入顺序来排序（如果设置accessOrder属性为true，则所有读写访问都排序）。

插入时，Entry把自己加到Header Entry的前面去。如果所有读写访问都要排序，还要把前后Entry的before/after拼接起来以在链表中删除掉自己，所以此时读操作也是线程不安全的了。

 

#### TreeMap
以红黑树实现，红黑树又叫自平衡二叉树：

对于任一节点而言，其到叶节点的每一条路径都包含相同数目的黑结点。
上面的规定，使得树的层数不会差的太远，使得所有操作的复杂度不超过 O（lgn），但也使得插入，修改时要复杂的左旋右旋来保持树的平衡。

支持iterator（）时按Key值排序，可按实现了Comparable接口的Key的升序排序，或由传入的Comparator控制。可想象的，在树上插入/删除元素的代价一定比HashMap的大。

支持SortedMap接口，如firstKey（），lastKey（）取得最大最小的key，或sub（fromKey, toKey）, tailMap（fromKey）剪取Map的某一段。

 

#### EnumMap
EnumMap的原理是，在构造函数里要传入枚举类，那它就构建一个与枚举的所有值等大的数组，按Enum. ordinal（）下标来访问数组。性能与内存占用俱佳。

美中不足的是，因为要实现Map接口，而 V get（Object key）中key是Object而不是泛型K，所以安全起见，EnumMap每次访问都要先对Key进行类型判断，在JMC里录得不低的采样命中频率。

 

#### ConcurrentHashMap
并发优化的HashMap。

在JDK5里的经典设计，默认16把写锁（可以设置更多），有效分散了阻塞的概率。数据结构为Segment[]，每个Segment一把锁。Segment里面才是哈希桶数组。Key先算出它在哪个Segment里，再去算它在哪个哈希桶里。

也没有读锁，因为put/remove动作是个原子动作（比如put的整个过程是一个对数组元素/Entry 指针的赋值操作），读操作不会看到一个更新动作的中间状态。

但在JDK8里，Segment[]的设计被抛弃了，改为精心设计的，只在需要锁的时候加锁。

支持ConcurrentMap接口，如putIfAbsent（key，value）与相反的replace（key，value）与以及实现CAS的replace（key, oldValue, newValue）。

 

####  ConcurrentSkipListMap
JDK6新增的并发优化的SortedMap，以SkipList结构实现。Concurrent包选用它是因为它支持基于CAS的无锁算法，而红黑树则没有好的无锁算法。

原理上，可以想象为多个链表组成的N层楼，其中的元素从稀疏到密集，每个元素有往右与往下的指针。从第一层楼开始遍历，如果右端的值比期望的大，那就往下走一层，继续往前走。

 



典型的空间换时间。每次插入，都要决定在哪几层插入，同时，要决定要不要多盖一层楼。

它的size（）同样不能随便调，会遍历来统计。

 

### Set


所有Set几乎都是内部用一个Map来实现, 因为Map里的KeySet就是一个Set，而value是假值，全部使用同一个Object即可。

Set的特征也继承了那些内部的Map实现的特征。

HashSet：内部是HashMap。

LinkedHashSet：内部是LinkedHashMap。

TreeSet：内部是TreeMap的SortedSet。

ConcurrentSkipListSet：内部是ConcurrentSkipListMap的并发优化的SortedSet。

CopyOnWriteArraySet：内部是CopyOnWriteArrayList的并发优化的Set，利用其addIfAbsent（）方法实现元素去重，如前所述该方法的性能很一般。

好像少了个ConcurrentHashSet，本来也该有一个内部用ConcurrentHashMap的简单实现，但JDK偏偏没提供。Jetty就自己简单封了一个，Guava则直接用java.util.Collections.newSetFromMap（new ConcurrentHashMap（）） 实现。

 


### Queue
Queue是在两端出入的List，所以也可以用数组或链表来实现。

#### 普通队列

LinkedList
是的，以双向链表实现的LinkedList既是List，也是Queue。

ArrayDeque
以循环数组实现的双向Queue。大小是2的倍数，默认是16。

为了支持FIFO，即从数组尾压入元素（快），从数组头取出元素（超慢），就不能再使用普通ArrayList的实现了，改为使用循环数组。

有队头队尾两个下标：弹出元素时，队头下标递增；加入元素时，队尾下标递增。如果加入元素时已到数组空间的末尾，则将元素赋值到数组[0]，同时队尾下标指向0，再插入下一个元素则赋值到数组[1]，队尾下标指向1。如果队尾的下标追上队头，说明数组所有空间已用完，进行双倍的数组扩容。

#### PriorityQueue
用平衡二叉最小堆实现的优先级队列，不再是FIFO，而是按元素实现的Comparable接口或传入Comparator的比较结果来出队，数值越小，优先级越高，越先出队。但是注意其iterator（）的返回不会排序。

平衡最小二叉堆，用一个简单的数组即可表达，可以快速寻址，没有指针什么的。最小的在queue[0] ，比如queue[4]的两个孩子，会在queue[2*4+1] 和 queue[2*（4+1）]，即queue[9]和queue[10]。

入队时，插入queue[size]，然后二叉地往上比较调整堆。

出队时，弹出queue[0]，然后把queque[size]拿出来二叉地往下比较调整堆。

初始大小为11，空间不够时自动50%扩容。

 

#### 线程安全的队列
ConcurrentLinkedQueue/Deque
无界的并发优化的Queue，基于链表，实现了依赖于CAS的无锁算法。

ConcurrentLinkedQueue的结构是单向链表和head/tail两个指针，因为入队时需要修改队尾元素的next指针，以及修改tail指向新入队的元素两个CAS动作无法原子，所以需要的特殊的算法。

#### 线程安全的阻塞队列
BlockingQueue，一来如果队列已空不用重复的查看是否有新数据而会阻塞在那里，二来队列的长度受限，用以保证生产者与消费者的速度不会相差太远。当入队时队列已满，或出队时队列已空，不同函数的效果见下表


ArrayBlockingQueue
定长的并发优化的BlockingQueue，也是基于循环数组实现。有一把公共的锁与notFull、notEmpty两个Condition管理队列满或空时的阻塞状态。

LinkedBlockingQueue/Deque
可选定长的并发优化的BlockingQueue，基于链表实现，所以可以把长度设为Integer.MAX_VALUE成为无界无等待的。

利用链表的特征，分离了takeLock与putLock两把锁，继续用notEmpty、notFull管理队列满或空时的阻塞状态。

PriorityBlockingQueue
无界的PriorityQueue，也是基于数组存储的二叉堆（见前）。一把公共的锁实现线程安全。因为无界，空间不够时会自动扩容，所以入列时不会锁，出列为空时才会锁。


DelayQueue
内部包含一个PriorityQueue，同样是无界的，同样是出列时才会锁。一把公共的锁实现线程安全。元素需实现Delayed接口，每次调用时需返回当前离触发时间还有多久，小于0表示该触发了。

pull（）时会用peek（）查看队头的元素，检查是否到达触发时间。ScheduledThreadPoolExecutor用了类似的结构。



#### 同步队列
SynchronousQueue同步队列本身无容量，放入元素时，比如等待元素被另一条线程的消费者取走再返回。JDK线程池里用它。

JDK7还有个LinkedTransferQueue，在普通线程安全的BlockingQueue的基础上，增加一个transfer（e） 函数，效果与SynchronousQueue一样。

## 参考文章

https://blog.csdn.net/zzw1531439090/article/details/87872424
https://blog.csdn.net/weixin_40374341/article/details/86496343
https://www.cnblogs.com/uodut/p/7067162.html
https://www.jb51.net/article/135672.htm
https://www.cnblogs.com/suiyue-/p/6052456.html

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。 

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
