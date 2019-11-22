# Table of Contents

  * [HashSet](#hashset)
    * [定义](#定义)
    * [方法](#方法)
  * [TreeSet](#treeset)
    * [TreeSet定义](#treeset定义)
    * [TreeSet主要方法](#treeset主要方法)
  * [最后](#最后)
  * [LinkedHashSet](#linkedhashset)
    * [LinkedHashSet内部是如何工作的](#linkedhashset内部是如何工作的)
    * [LinkedHashSet是如何维护插入顺序的](#linkedhashset是如何维护插入顺序的)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)

本文参考 http://cmsblogs.com/?p=599

《Java集合详解系列》是我在完成夯实Java基础篇的系列博客后准备开始整理的新系列文章。
为了更好地诠释知识点，形成体系文章，本系列文章整理了很多优质的博客内容，如有侵权请联系我，一定删除。

这些文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star、fork哈

本系列文章将整理于我的个人博客：

> www.how2playlife.com

今天我们来探索一下HashSet，TreeSet与LinkedHashSet的基本原理与源码实现，由于这三个set都是基于之前文章的三个map进行实现的，所以推荐大家先看一下前面有关map的文章，结合使用味道更佳。



## HashSet

### 定义

    public class HashSet<E>
        extends AbstractSet<E>
        implements Set<E>, Cloneable, java.io.Serializable

HashSet继承AbstractSet类，实现Set、Cloneable、Serializable接口。其中AbstractSet提供 Set 接口的骨干实现，从而最大限度地减少了实现此接口所需的工作。
==Set接口是一种不包括重复元素的Collection，它维持它自己的内部排序，所以随机访问没有任何意义。==

本文基于1.8jdk进行源码分析。

 基本属性

基于HashMap实现，底层使用HashMap保存所有元素

    private transient HashMap<E,Object> map;
    
    //定义一个Object对象作为HashMap的value
    private static final Object PRESENT = new Object();

 构造函数

    /**
         * 默认构造函数
         * 初始化一个空的HashMap，并使用默认初始容量为16和加载因子0.75。
         */
        public HashSet() {
            map = new HashMap<>();
        }
    
        /**
         * 构造一个包含指定 collection 中的元素的新 set。
         */
        public HashSet(Collection<? extends E> c) {
            map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
            addAll(c);
        }
    
        /**
         * 构造一个新的空 set，其底层 HashMap 实例具有指定的初始容量和指定的加载因子
         */
        public HashSet(int initialCapacity, float loadFactor) {
            map = new HashMap<>(initialCapacity, loadFactor);
        }
    
        /**
         * 构造一个新的空 set，其底层 HashMap 实例具有指定的初始容量和默认的加载因子（0.75）。
         */
        public HashSet(int initialCapacity) {
           map = new HashMap<>(initialCapacity);
        }
    
        /**
         * 在API中我没有看到这个构造函数，今天看源码才发现（原来访问权限为包权限，不对外公开的）
         * 以指定的initialCapacity和loadFactor构造一个新的空链接哈希集合。
         * dummy 为标识 该构造函数主要作用是对LinkedHashSet起到一个支持作用
         */
        HashSet(int initialCapacity, float loadFactor, boolean dummy) {
           map = new LinkedHashMap<>(initialCapacity, loadFactor);
        }
     从构造函数中可以看出HashSet所有的构造都是构造出一个新的HashMap，其中最后一个构造函数，为包访问权限是不对外公开，仅仅只在使用LinkedHashSet时才会发生作用。

### 方法

 既然HashSet是基于HashMap，那么对于HashSet而言，其方法的实现过程是非常简单的。

    public Iterator<E> iterator() {
            return map.keySet().iterator();
        }

> iterator()方法返回对此 set 中元素进行迭代的迭代器。返回元素的顺序并不是特定的。
>
> 底层调用HashMap的keySet返回所有的key，这点反应了HashSet中的所有元素都是保存在HashMap的key中，value则是使用的PRESENT对象，该对象为static final。
>
>     public int size() {
>             return map.size();
>         }
>        size()返回此 set 中的元素的数量（set 的容量）。底层调用HashMap的size方法，返回HashMap容器的大小。

    public boolean isEmpty() {
            return map.isEmpty();
        }
        isEmpty()，判断HashSet()集合是否为空，为空返回 true，否则返回false。
    
    public boolean contains(Object o) {
            return map.containsKey(o);
    }
    
    public boolean containsKey(Object key) {
        return getNode(hash(key), key) != null;
    }
    
    //最终调用该方法进行节点查找
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //先检查桶的头结点是否存在
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            if (first.hash == hash && // always check first node
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
                //不是头结点，则遍历链表，如果是树节点则使用树节点的方法遍历，直到找到，或者为null
            if ((e = first.next) != null) {
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }

contains()，判断某个元素是否存在于HashSet()中，存在返回true，否则返回false。更加确切的讲应该是要满足这种关系才能返回true：(o==null ? e==null : o.equals(e))。底层调用containsKey判断HashMap的key值是否为空。
    
    public boolean add(E e) {
            return map.put(e, PRESENT)==null;
    }
    
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }
    
    map的put方法：
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        
        //确认初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
            
        //如果桶为空，直接插入新元素，也就是entry
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            //如果冲突，分为三种情况
            //key相等时让旧entry等于新entry即可
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            //红黑树情况
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                //如果key不相等，则连成链表
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        return null;
    }

> 这里注意一点，hashset只是不允许重复的元素加入，而不是不允许元素连成链表，因为只要key的equals方法判断为true时它们是相等的，此时会发生value的替换，因为所有entry的value一样，所以和没有插入时一样的。
>
> 而当两个hashcode相同但key不相等的entry插入时，仍然会连成一个链表，长度超过8时依然会和hashmap一样扩展成红黑树，看完源码之后笔者才明白自己之前理解错了。所以看源码还是蛮有好处的。hashset基本上就是使用hashmap的方法再次实现了一遍而已，只不过value全都是同一个object，让你以为相同元素没有插入，事实上只是value替换成和原来相同的值而已。

当add方法发生冲突时，如果key相同，则替换value，如果key不同，则连成链表。       

add()如果此 set 中尚未包含指定元素，则添加指定元素。如果此Set没有包含满足(e==null ? e2==null : e.equals(e2)) 的e2时，则将e2添加到Set中，否则不添加且返回false。

由于底层使用HashMap的put方法将key = e，value=PRESENT构建成key-value键值对，当此e存在于HashMap的key中，则value将会覆盖原有value，但是key保持不变，所以如果将一个已经存在的e元素添加中HashSet中，新添加的元素是不会保存到HashMap中，所以这就满足了HashSet中元素不会重复的特性。

    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }
remove如果指定元素存在于此 set 中，则将其移除。底层使用HashMap的remove方法删除指定的Entry。

    public void clear() {
        map.clear();
    }

clear从此 set 中移除所有元素。底层调用HashMap的clear方法清除所有的Entry。

    public Object clone() {
            try {
                HashSet<E> newSet = (HashSet<E>) super.clone();
                newSet.map = (HashMap<E, Object>) map.clone();
                return newSet;
            } catch (CloneNotSupportedException e) {
                throw new InternalError();
            }
        }

clone返回此 HashSet 实例的浅表副本：并没有复制这些元素本身。

后记：

> 由于HashSet底层使用了HashMap实现，使其的实现过程变得非常简单，如果你对HashMap比较了解，那么HashSet简直是小菜一碟。有两个方法对HashMap和HashSet而言是非常重要的，下篇将详细讲解hashcode和equals。


## TreeSet

与HashSet是基于HashMap实现一样，TreeSet同样是基于TreeMap实现的。在《Java提高篇（二七）-----TreeMap》中LZ详细讲解了TreeMap实现机制，如果客官详情看了这篇博文或者多TreeMap有比较详细的了解，那么TreeSet的实现对您是喝口水那么简单。

### TreeSet定义

我们知道TreeMap是一个有序的二叉树，那么同理TreeSet同样也是一个有序的，它的作用是提供有序的Set集合。通过源码我们知道TreeSet基础AbstractSet，实现NavigableSet、Cloneable、Serializable接口。

其中AbstractSet提供 Set 接口的骨干实现，从而最大限度地减少了实现此接口所需的工作。

NavigableSet是扩展的 SortedSet，具有了为给定搜索目标报告最接近匹配项的导航方法，这就意味着它支持一系列的导航方法。比如查找与指定目标最匹配项。Cloneable支持克隆，Serializable支持序列化。

    public class TreeSet<E> extends AbstractSet<E>
        implements NavigableSet<E>, Cloneable, java.io.Serializable
同时在TreeSet中定义了如下几个变量。

    private transient NavigableMap<E,Object> m;
        
    //PRESENT会被当做Map的value与key构建成键值对
     private static final Object PRESENT = new Object();
其构造方法：

    //默认构造方法，根据其元素的自然顺序进行排序
    
    public TreeSet() {
        this(new TreeMap<E,Object>());
    }
    
    //构造一个包含指定 collection 元素的新 TreeSet，它按照其元素的自然顺序进行排序。
    public TreeSet(Comparator<? super E> comparator) {
            this(new TreeMap<>(comparator));
    }
    
    //构造一个新的空 TreeSet，它根据指定比较器进行排序。
    public TreeSet(Collection<? extends E> c) {
        this();
        addAll(c);
    }
    
    //构造一个与指定有序 set 具有相同映射关系和相同排序的新 TreeSet。
    public TreeSet(SortedSet<E> s) {
        this(s.comparator());
        addAll(s);
    }
    
    TreeSet(NavigableMap<E,Object> m) {
        this.m = m;
    }


### TreeSet主要方法

1、add：将指定的元素添加到此 set（如果该元素尚未存在于 set 中）。
    
    public boolean add(E e) {
            return m.put(e, PRESENT)==null;
        }
     
    public V put(K key, V value) {
        Entry<K,V> t = root;
        if (t == null) {
        //空树时，判断节点是否为空
            compare(key, key); // type (and possibly null) check
    
            root = new Entry<>(key, value, null);
            size = 1;
            modCount++;
            return null;
        }
        int cmp;
        Entry<K,V> parent;
        // split comparator and comparable paths
        Comparator<? super K> cpr = comparator;
        //非空树，根据传入比较器进行节点的插入位置查找
        if (cpr != null) {
            do {
                parent = t;
                //节点比根节点小，则找左子树，否则找右子树
                cmp = cpr.compare(key, t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                    //如果key的比较返回值相等，直接更新值（一般compareto相等时equals方法也相等）
                else
                    return t.setValue(value);
            } while (t != null);
        }
        else {
        //如果没有传入比较器，则按照自然排序
            if (key == null)
                throw new NullPointerException();
            @SuppressWarnings("unchecked")
                Comparable<? super K> k = (Comparable<? super K>) key;
            do {
                parent = t;
                cmp = k.compareTo(t.key);
                if (cmp < 0)
                    t = t.left;
                else if (cmp > 0)
                    t = t.right;
                else
                    return t.setValue(value);
            } while (t != null);
        }
        //查找的节点为空，直接插入，默认为红节点
        Entry<K,V> e = new Entry<>(key, value, parent);
        if (cmp < 0)
            parent.left = e;
        else
            parent.right = e;
            //插入后进行红黑树调整
        fixAfterInsertion(e);
        size++;
        modCount++;
        return null;
    }    

2、get：获取元素

    public V get(Object key) {
        Entry<K,V> p = getEntry(key);
        return (p==null ? null : p.value);
    }

该方法与put的流程类似，只不过是把插入换成了查找
    
3、ceiling：返回此 set 中大于等于给定元素的最小元素；如果不存在这样的元素，则返回 null。

    public E ceiling(E e) {
            return m.ceilingKey(e);
        }
4、clear：移除此 set 中的所有元素。

    public void clear() {
            m.clear();
        }
5、clone：返回 TreeSet 实例的浅表副本。属于浅拷贝。

    public Object clone() {
            TreeSet<E> clone = null;
            try {
                clone = (TreeSet<E>) super.clone();
            } catch (CloneNotSupportedException e) {
                throw new InternalError();
            }
    
            clone.m = new TreeMap<>(m);
            return clone;
        }

6、comparator：返回对此 set 中的元素进行排序的比较器；如果此 set 使用其元素的自然顺序，则返回 null。

    public Comparator<? super E> comparator() {
            return m.comparator();
        }
7、contains：如果此 set 包含指定的元素，则返回 true。

    public boolean contains(Object o) {
            return m.containsKey(o);
        }
8、descendingIterator：返回在此 set 元素上按降序进行迭代的迭代器。

    public Iterator<E> descendingIterator() {
            return m.descendingKeySet().iterator();
        }
9、descendingSet：返回此 set 中所包含元素的逆序视图。

    public NavigableSet<E> descendingSet() {
            return new TreeSet<>(m.descendingMap());
        }
10、first：返回此 set 中当前第一个（最低）元素。

    public E first() {
            return m.firstKey();
        }
11、floor：返回此 set 中小于等于给定元素的最大元素；如果不存在这样的元素，则返回 null。
    
    public E floor(E e) {
            return m.floorKey(e);
        }
12、headSet：返回此 set 的部分视图，其元素严格小于 toElement。

    public SortedSet<E> headSet(E toElement) {
            return headSet(toElement, false);
        }
13、higher：返回此 set 中严格大于给定元素的最小元素；如果不存在这样的元素，则返回 null。

    public E higher(E e) {
            return m.higherKey(e);
        }
14、isEmpty：如果此 set 不包含任何元素，则返回 true。

    public boolean isEmpty() {
            return m.isEmpty();
        }
15、iterator：返回在此 set 中的元素上按升序进行迭代的迭代器。

    public Iterator<E> iterator() {
            return m.navigableKeySet().iterator();
        }
16、last：返回此 set 中当前最后一个（最高）元素。

    public E last() {
            return m.lastKey();
        }
17、lower：返回此 set 中严格小于给定元素的最大元素；如果不存在这样的元素，则返回 null。
    
    public E lower(E e) {
            return m.lowerKey(e);
        }
18、pollFirst：获取并移除第一个（最低）元素；如果此 set 为空，则返回 null。

    public E pollFirst() {
            Map.Entry<E,?> e = m.pollFirstEntry();
            return (e == null) ? null : e.getKey();
        }
19、pollLast：获取并移除最后一个（最高）元素；如果此 set 为空，则返回 null。

    public E pollLast() {
            Map.Entry<E,?> e = m.pollLastEntry();
            return (e == null) ? null : e.getKey();
        }
20、remove：将指定的元素从 set 中移除（如果该元素存在于此 set 中）。

    public boolean remove(Object o) {
            return m.remove(o)==PRESENT;
        }

该方法与put类似，只不过把插入换成了删除，并且要进行删除后调整

21、size：返回 set 中的元素数（set 的容量）。

    public int size() {
            return m.size();
        }
22、subSet：返回此 set 的部分视图

    /**
         * 返回此 set 的部分视图，其元素范围从 fromElement 到 toElement。
         */
         public NavigableSet<E> subSet(E fromElement, boolean fromInclusive,
                 E toElement,   boolean toInclusive) {
                 return new TreeSet<>(m.subMap(fromElement, fromInclusive,
                      toElement,   toInclusive));
         }
         
         /**
          * 返回此 set 的部分视图，其元素从 fromElement（包括）到 toElement（不包括）。
          */
         public SortedSet<E> subSet(E fromElement, E toElement) {
             return subSet(fromElement, true, toElement, false);
         }
23、tailSet：返回此 set 的部分视图

    /**
         * 返回此 set 的部分视图，其元素大于（或等于，如果 inclusive 为 true）fromElement。
         */
        public NavigableSet<E> tailSet(E fromElement, boolean inclusive) {
            return new TreeSet<>(m.tailMap(fromElement, inclusive));
        }
        
        /**
         * 返回此 set 的部分视图，其元素大于等于 fromElement。
         */
        public SortedSet<E> tailSet(E fromElement) {
            return tailSet(fromElement, true);
        }


## 最后

由于TreeSet是基于TreeMap实现的，所以如果我们对treeMap有了一定的了解，对TreeSet那是小菜一碟，我们从TreeSet中的源码可以看出，其实现过程非常简单，几乎所有的方法实现全部都是基于TreeMap的。

## LinkedHashSet

### LinkedHashSet内部是如何工作的

LinkedHashSet是HashSet的一个“扩展版本”，HashSet并不管什么顺序，不同的是LinkedHashSet会维护“插入顺序”。HashSet内部使用HashMap对象来存储它的元素，而LinkedHashSet内部使用LinkedHashMap对象来存储和处理它的元素。这篇文章，我们将会看到LinkedHashSet内部是如何运作的及如何维护插入顺序的。

我们首先着眼LinkedHashSet的构造函数。在LinkedHashSet类中一共有4个构造函数。这些构造函数都只是简单地调用父类构造函数（如HashSet类的构造函数）。
下面看看LinkedHashSet的构造函数是如何定义的。

    //Constructor - 1
     
    public LinkedHashSet(int initialCapacity, float loadFactor)
    {
          super(initialCapacity, loadFactor, true);              //Calling super class constructor
    }
     
    //Constructor - 2
     
    public LinkedHashSet(int initialCapacity)
    {
            super(initialCapacity, .75f, true);             //Calling super class constructor
    }
     
    //Constructor - 3
     
    public LinkedHashSet()
    {
            super(16, .75f, true);                //Calling super class constructor
    }
     
    //Constructor - 4
     
    public LinkedHashSet(Collection<? extends E> c)
    {
            super(Math.max(2*c.size(), 11), .75f, true);          //Calling super class constructor
            addAll(c);
    }
    
在上面的代码片段中，你可能注意到4个构造函数调用的是同一个父类的构造函数。这个构造函数（父类的，译者注）是一个包内私有构造函数（见下面的代码，HashSet的构造函数没有使用public公开，译者注），它只能被LinkedHashSet使用。

这个构造函数需要初始容量，负载因子和一个boolean类型的哑值（没有什么用处的参数，作为标记，译者注）等参数。这个哑参数只是用来区别这个构造函数与HashSet的其他拥有初始容量和负载因子参数的构造函数，下面是这个构造函数的定义，

    HashSet(int initialCapacity, float loadFactor, boolean dummy)
    {
            map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }
显然，这个构造函数内部初始化了一个LinkedHashMap对象，这个对象恰好被LinkedHashSet用来存储它的元素。

LinkedHashSet并没有自己的方法，所有的方法都继承自它的父类HashSet，因此，对LinkedHashSet的所有操作方式就好像对HashSet操作一样。

唯一的不同是内部使用不同的对象去存储元素。在HashSet中，插入的元素是被当做HashMap的键来保存的，而在LinkedHashSet中被看作是LinkedHashMap的键。

这些键对应的值都是常量PRESENT（PRESENT是HashSet的静态成员变量，译者注）。

### LinkedHashSet是如何维护插入顺序的
> LinkedHashSet使用LinkedHashMap对象来存储它的元素，插入到LinkedHashSet中的元素实际上是被当作LinkedHashMap的键保存起来的。
>
> LinkedHashMap的每一个键值对都是通过内部的静态类Entry<K, V>实例化的。这个 Entry<K, V>类继承了HashMap.Entry类。
>
> 这个静态类增加了两个成员变量，before和after来维护LinkedHasMap元素的插入顺序。这两个成员变量分别指向前一个和后一个元素，这让LinkedHashMap也有类似双向链表的表现。

    private static class Entry<K,V> extends HashMap.Entry<K,V>
    {
            // These fields comprise the doubly linked list used for iteration.
            Entry<K,V> before, after;
     
            Entry(int hash, K key, V value, HashMap.Entry<K,V> next) {
                super(hash, key, value, next);
            }
    }
从上面代码看到的LinkedHashMap内部类的前面两个成员变量——before和after负责维护LinkedHashSet的插入顺序。LinkedHashMap定义的成员变量header保存的是
这个双向链表的头节点。header的定义就像下面这样，

接下来看一个例子就知道LinkedHashSet内部是如何工作的了。

    public class LinkedHashSetExample
    {
        public static void main(String[] args)
        {
            //Creating LinkedHashSet
     
            LinkedHashSet<String> set = new LinkedHashSet<String>();
     
            //Adding elements to LinkedHashSet
     
            set.add("BLUE");
     
            set.add("RED");
     
            set.add("GREEN");    
     
            set.add("BLACK");
        }
    }


如果你知道LinkedHashMap内部是如何工作的，就非常容易明白LinkedHashSet内部是如何工作的。看一遍LinkedHashSet和LinkedHashMap的源码，
你就能够准确地理解在Java中LinkedHashSet内部是如何工作的。


## 参考文章
http://cmsblogs.com/?p=599

https://www.cnblogs.com/one-apple-pie/p/11036309.html

https://blog.csdn.net/learningcoding/article/details/79983248
## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)



​                     
