# Table of Contents

  * [Iterator](#iterator)
    * [java.util.Iterator](#javautiliterator)
    * [各个集合的Iterator的实现](#各个集合的iterator的实现)
    * [fail-fast机制](#fail-fast机制)
      * [fail-fast示例](#fail-fast示例)
      * [fail-fast产生原因](#fail-fast产生原因)
      * [fail-fast解决办法](#fail-fast解决办法)
* [Comparable 和 Comparator](#comparable-和-comparator)
  * [Comparable](#comparable)
    * [Comparator](#comparator)
    * [Java8中使用lambda实现比较器](#java8中使用lambda实现比较器)
  * [总结](#总结)
  * [参考文章](#参考文章)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)

本文参考 cmsblogs.com/p=1185

《Java集合详解系列》是我在完成夯实Java基础篇的系列博客后准备开始整理的新系列文章。
为了更好地诠释知识点，形成体系文章，本系列文章整理了很多优质的博客内容，如有侵权请联系我，一定删除。

这些文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star、fork哈

本系列文章将整理于我的个人博客：

> www.how2playlife.com

今天我们来探索一下LIterator，fail-fast机制与比较器的源码。


## Iterator


迭代对于我们搞Java的来说绝对不陌生。我们常常使用JDK提供的迭代接口进行Java集合的迭代。

    Iterator iterator = list.iterator();
            while(iterator.hasNext()){
                String string = iterator.next();
                do something
            }
迭代其实我们可以简单地理解为遍历，是一个标准化遍历各类容器里面的所有对象的方法类，它是一个很典型的设计模式。Iterator模式是用于遍历集合类的标准访问方法。

它可以把访问逻辑从不同类型的集合类中抽象出来，从而避免向客户端暴露集合的内部结构。 在没有迭代器时我们都是这么进行处理的。如下：

对于数组我们是使用下标来进行处理的

    int[] arrays = new int[10];
       for(int i = 0 ; i  arrays.length ; i++){
           int a = arrays[i];
           do something
       }

对于ArrayList是这么处理的

    ListString list = new ArrayListString();
       for(int i = 0 ; i  list.size() ;  i++){
          String string = list.get(i);
          do something
       }

对于这两种方式，我们总是都事先知道集合的内部结构，访问代码和集合本身是紧密耦合的，无法将访问逻辑从集合类和客户端代码中分离出来。同时每一种集合对应一种遍历方法，客户端代码无法复用。

在实际应用中如何需要将上面将两个集合进行整合是相当麻烦的。所以为了解决以上问题，Iterator模式腾空出世，它总是用同一种逻辑来遍历集合。

使得客户端自身不需要来维护集合的内部结构，所有的内部状态都由Iterator来维护。客户端从不直接和集合类打交道，它总是控制Iterator，向它发送向前，向后，取当前元素的命令，就可以间接遍历整个集合。

上面只是对Iterator模式进行简单的说明，下面我们看看Java中Iterator接口，看他是如何来进行实现的。

### java.util.Iterator

在Java中Iterator为一个接口，它只提供了迭代了基本规则，在JDK中他是这样定义的：对 collection 进行迭代的迭代器。迭代器取代了 Java Collections Framework 中的 Enumeration。迭代器与枚举有两点不同：

    1、迭代器允许调用者利用定义良好的语义在迭代期间从迭代器所指向的 collection 移除元素。
    
    2、方法名称得到了改进。

其接口定义如下：

    public interface Iterator {
    　　boolean hasNext();
    　　Object next();
    　　void remove();
    }
其中：

    Object next()：返回迭代器刚越过的元素的引用，返回值是Object，需要强制转换成自己需要的类型
    
    boolean hasNext()：判断容器内是否还有可供访问的元素
    
    void remove()：删除迭代器刚越过的元素

对于我们而言，我们只一般只需使用next()、hasNext()两个方法即可完成迭代。如下：

    for(Iterator it = c.iterator(); it.hasNext(); ) {
    　　Object o = it.next();
    　　 do something
    }

==前面阐述了Iterator有一个很大的优点,就是我们不必知道集合的内部结果,集合的内部结构、状态由Iterator来维持，通过统一的方法hasNext()、next()来判断、获取下一个元素，至于具体的内部实现我们就不用关心了。==

但是作为一个合格的程序员我们非常有必要来弄清楚Iterator的实现。下面就ArrayList的源码进行分析分析。

### 各个集合的Iterator的实现

下面就ArrayList的Iterator实现来分析，其实如果我们理解了ArrayList、Hashset、TreeSet的数据结构，内部实现，对于他们是如何实现Iterator也会胸有成竹的。因为ArrayList的内部实现采用数组，所以我们只需要记录相应位置的索引即可，其方法的实现比较简单。

ArrayList的Iterator实现

在ArrayList内部首先是定义一个内部类Itr，该内部类实现Iterator接口，如下：

    private class Itr implements IteratorE {
        do something
    }
    而ArrayList的iterator()方法实现：
    
    public IteratorE iterator() {
            return new Itr();
        }

所以通过使用ArrayList.iterator()方法返回的是Itr()内部类，所以现在我们需要关心的就是Itr()内部类的实现：

在Itr内部定义了三个int型的变量：cursor、lastRet、expectedModCount。其中cursor表示下一个元素的索引位置，lastRet表示上一个元素的索引位置

            int cursor;             
            int lastRet = -1;     
            int expectedModCount = modCount;

从cursor、lastRet定义可以看出，lastRet一直比cursor少一所以hasNext()实现方法异常简单，只需要判断cursor和lastRet是否相等即可。

    public boolean hasNext() {
        return cursor != size;
    }

对于next()实现其实也是比较简单的，只要返回cursor索引位置处的元素即可，然后修改cursor、lastRet即可。

    public E next() {
        checkForComodification();
        int i = cursor;    记录索引位置
        if (i = size)    如果获取元素大于集合元素个数，则抛出异常
            throw new NoSuchElementException();
        Object[] elementData = ArrayList.this.elementData;
        if (i = elementData.length)
            throw new ConcurrentModificationException();
        cursor = i + 1;      cursor + 1
        return (E) elementData[lastRet = i];  lastRet + 1 且返回cursor处元素
    }

 checkForComodification()主要用来判断集合的修改次数是否合法，即用来判断遍历过程中集合是否被修改过。

 。modCount用于记录ArrayList集合的修改次数，初始化为0，，每当集合被修改一次（结构上面的修改，内部update不算），如add、remove等方法，modCount + 1，所以如果modCount不变，则表示集合内容没有被修改。

 该机制主要是用于实现ArrayList集合的快速失败机制，在Java的集合中，较大一部分集合是存在快速失败机制的，这里就不多说，后面会讲到。

 所以要保证在遍历过程中不出错误，我们就应该保证在遍历过程中不会对集合产生结构上的修改（当然remove方法除外），出现了异常错误，我们就应该认真检查程序是否出错而不是catch后不做处理。

    final void checkForComodification() {
                if (modCount != expectedModCount)
                    throw new ConcurrentModificationException();
            }
    对于remove()方法的是实现，它是调用ArrayList本身的remove()方法删除lastRet位置元素，然后修改modCount即可。
    
    public void remove() {
        if (lastRet  0)
            throw new IllegalStateException();
        checkForComodification();
    
        try {
            ArrayList.this.remove(lastRet);
            cursor = lastRet;
            lastRet = -1;
            expectedModCount = modCount;
        } catch (IndexOutOfBoundsException ex) {
            throw new ConcurrentModificationException();
        }
    }

这里就对ArrayList的Iterator实现讲解到这里，对于Hashset、TreeSet等集合的Iterator实现，各位如果感兴趣可以继续研究，个人认为在研究这些集合的源码之前，有必要对该集合的数据结构有清晰的认识，这样会达到事半功倍的效果！！！！

### fail-fast机制
这部分参考http://cmsblogs.com/p=1220

在JDK的Collection中我们时常会看到类似于这样的话：

例如，ArrayList
    
 注意，迭代器的快速失败行为无法得到保证，因为一般来说，不可能对是否出现不同步并发修改做出任何硬性保证。快速失败迭代器会尽最大努力抛出ConcurrentModificationException。
 因此，为提高这类迭代器的正确性而编写一个依赖于此异常的程序是错误的做法：迭代器的快速失败行为应该仅用于检测 bug。

HashMap中：

 注意，迭代器的快速失败行为不能得到保证，一般来说，存在非同步的并发修改时，不可能作出任何坚决的保证。快速失败迭代器尽最大努力抛出 ConcurrentModificationException。因此，编写依赖于此异常的程序的做法是错误的，正确做法是：迭代器的快速失败行为应该仅用于检测程序错误。

在这两段话中反复地提到”快速失败”。那么何为”快速失败”机制呢？

 “快速失败”也就是fail-fast，它是Java集合的一种错误检测机制。当多个线程对集合进行结构上的改变的操作时，有可能会产生fail-fast机制。

 记住是有可能，而不是一定。例如：假设存在两个线程（线程1、线程2），线程1通过Iterator在遍历集合A中的元素，在某个时候线程2修改了集合A的结构（是结构上面的修改，而不是简单的修改集合元素的内容），那么这个时候程序就会抛出 ConcurrentModificationException异常，从而产生fail-fast机制。

#### fail-fast示例

    public class FailFastTest {
        private static ListInteger list = new ArrayList();


​        
          @desc线程one迭代list
          @Projecttest
          @fileFailFastTest.java
          @Authrochenssy
          @data2014年7月26日
         
        private static class threadOne extends Thread{
            public void run() {
                IteratorInteger iterator = list.iterator();
                while(iterator.hasNext()){
                    int i = iterator.next();
                    System.out.println(ThreadOne 遍历 + i);
                    try {
                        Thread.sleep(10);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }


​        
          @desc当i == 3时，修改list
          @Projecttest
          @fileFailFastTest.java
          @Authrochenssy
          @data2014年7月26日
         
        private static class threadTwo extends Thread{
            public void run(){
                int i = 0 ; 
                while(i  6){
                    System.out.println(ThreadTwo run： + i);
                    if(i == 3){
                        list.remove(i);
                    }
                    i++;
                }
            }
        }
        
        public static void main(String[] args) {
            for(int i = 0 ; i  10;i++){
                list.add(i);
            }
            new threadOne().start();
            new threadTwo().start();
        }
    }
运行结果：

    ThreadOne 遍历0
    ThreadTwo run：0
    ThreadTwo run：1
    ThreadTwo run：2
    ThreadTwo run：3
    ThreadTwo run：4
    ThreadTwo run：5
    Exception in thread Thread-0 java.util.ConcurrentModificationException
        at java.util.ArrayList$Itr.checkForComodification(Unknown Source)
        at java.util.ArrayList$Itr.next(Unknown Source)
        at test.ArrayListTest$threadOne.run(ArrayListTest.java23)

#### fail-fast产生原因

通过上面的示例和讲解，我初步知道fail-fast产生的原因就在于程序在对 collection 进行迭代时，某个线程对该 collection 在结构上对其做了修改，这时迭代器就会抛出 ConcurrentModificationException 异常信息，从而产生 fail-fast。

 要了解fail-fast机制，我们首先要对ConcurrentModificationException 异常有所了解。当方法检测到对象的并发修改，但不允许这种修改时就抛出该异常。同时需要注意的是，该异常不会始终指出对象已经由不同线程并发修改，如果单线程违反了规则，同样也有可能会抛出改异常。


诚然，迭代器的快速失败行为无法得到保证，它不能保证一定会出现该错误，但是快速失败操作会尽最大努力抛出ConcurrentModificationException异常，所以因此，为提高此类操作的正确性而编写一个依赖于此异常的程序是错误的做法，正确做法是：ConcurrentModificationException 应该仅用于检测 bug。下面我将以ArrayList为例进一步分析fail-fast产生的原因。

 从前面我们知道fail-fast是在操作迭代器时产生的。现在我们来看看ArrayList中迭代器的源代码：

    private class Itr implements IteratorE {
            int cursor;
            int lastRet = -1;
            int expectedModCount = ArrayList.this.modCount;
    
            public boolean hasNext() {
                return (this.cursor != ArrayList.this.size);
            }
    
            public E next() {
                checkForComodification();
                 省略此处代码 
            }
    
            public void remove() {
                if (this.lastRet  0)
                    throw new IllegalStateException();
                checkForComodification();
                 省略此处代码 
            }
    
            final void checkForComodification() {
                if (ArrayList.this.modCount == this.expectedModCount)
                    return;
                throw new ConcurrentModificationException();
            }
        }

从上面的源代码我们可以看出，迭代器在调用next()、remove()方法时都是调用checkForComodification()方法，该方法主要就是检测modCount == expectedModCount  若不等则抛出ConcurrentModificationException 异常，从而产生fail-fast机制。所以要弄清楚为什么会产生fail-fast机制我们就必须要用弄明白为什么modCount != expectedModCount ，他们的值在什么时候发生改变的。

expectedModCount 是在Itr中定义的：int expectedModCount = ArrayList.this.modCount;所以他的值是不可能会修改的，所以会变的就是modCount。modCount是在 AbstractList 中定义的，为全局变量：

protected transient int modCount = 0;
那么他什么时候因为什么原因而发生改变呢？请看ArrayList的源码：

    public boolean add(E paramE) {
        ensureCapacityInternal(this.size + 1);
         省略此处代码 
    }
    
    private void ensureCapacityInternal(int paramInt) {
        if (this.elementData == EMPTY_ELEMENTDATA)
            paramInt = Math.max(10, paramInt);
        ensureExplicitCapacity(paramInt);
    }
    
    private void ensureExplicitCapacity(int paramInt) {
        this.modCount += 1;    修改modCount
         省略此处代码 
    }

   public boolean remove(Object paramObject) {
        int i;
        if (paramObject == null)
            for (i = 0; i  this.size; ++i) {
                if (this.elementData[i] != null)
                    continue;
                fastRemove(i);
                return true;
            }
        else
            for (i = 0; i  this.size; ++i) {
                if (!(paramObject.equals(this.elementData[i])))
                    continue;
                fastRemove(i);
                return true;
            }
        return false;
    }

    private void fastRemove(int paramInt) {
        this.modCount += 1;   修改modCount
         省略此处代码 
    }
    
    public void clear() {
        this.modCount += 1;    修改modCount
         省略此处代码 
    }
 从上面的源代码我们可以看出，ArrayList中无论add、remove、clear方法只要是涉及了改变ArrayList元素的个数的方法都会导致modCount的改变。

所以我们这里可以初步判断由于expectedModCount 得值与modCount的改变不同步，导致两者之间不等从而产生fail-fast机制。知道产生fail-fast产生的根本原因了，我们可以有如下场景：

有两个线程（线程A，线程B），其中线程A负责遍历list、线程B修改list。线程A在遍历list过程的某个时候（此时expectedModCount = modCount=N），线程启动，同时线程B增加一个元素，这是modCount的值发生改变（modCount + 1 = N + 1）。

线程A继续遍历执行next方法时，通告checkForComodification方法发现expectedModCount  = N  ，而modCount = N + 1，两者不等，这时就抛出ConcurrentModificationException 异常，从而产生fail-fast机制。

所以，直到这里我们已经完全了解了fail-fast产生的根本原因了。知道了原因就好找解决办法了。

#### fail-fast解决办法

通过前面的实例、源码分析，我想各位已经基本了解了fail-fast的机制，下面我就产生的原因提出解决方案。这里有两种解决方案：

 方案一：在遍历过程中所有涉及到改变modCount值得地方全部加上synchronized或者直接使用Collections.synchronizedList，这样就可以解决。但是不推荐，因为增删造成的同步锁可能会阻塞遍历操作。

 方案二：使用CopyOnWriteArrayList来替换ArrayList。推荐使用该方案。

CopyOnWriteArrayList为何物？ArrayList 的一个线程安全的变体，其中所有可变操作（add、set 等等）都是通过对底层数组进行一次新的复制来实现的。 该类产生的开销比较大，但是在两种情况下，它非常适合使用。

 1：在不能或不想进行同步遍历，但又需要从并发线程中排除冲突时。

 2：当遍历操作的数量大大超过可变操作的数量时。遇到这两种情况使用CopyOnWriteArrayList来替代ArrayList再适合不过了。那么为什么CopyOnWriterArrayList可以替代ArrayList呢？


第一、CopyOnWriterArrayList的无论是从数据结构、定义都和ArrayList一样。它和ArrayList一样，同样是实现List接口，底层使用数组实现。在方法上也包含add、remove、clear、iterator等方法。

第二、CopyOnWriterArrayList根本就不会产生ConcurrentModificationException异常，也就是它使用迭代器完全不会产生fail-fast机制。请看：

    private static class COWIteratorE implements ListIteratorE {
             省略此处代码 
            public E next() {
                if (!(hasNext()))
                    throw new NoSuchElementException();
                return this.snapshot[(this.cursor++)];
            }
    
             省略此处代码 
        }
CopyOnWriterArrayList的方法根本就没有像ArrayList中使用checkForComodification方法来判断expectedModCount 与 modCount 是否相等。它为什么会这么做，凭什么可以这么做呢？我们以add方法为例：

    public boolean add(E paramE) {
            ReentrantLock localReentrantLock = this.lock;
            localReentrantLock.lock();
            try {
                Object[] arrayOfObject1 = getArray();
                int i = arrayOfObject1.length;
                Object[] arrayOfObject2 = Arrays.copyOf(arrayOfObject1, i + 1);
                arrayOfObject2[i] = paramE;
                setArray(arrayOfObject2);
                int j = 1;
                return j;
            } finally {
                localReentrantLock.unlock();
            }
        }


      
        final void setArray(Object[] paramArrayOfObject) {
            this.array = paramArrayOfObject;
        }

CopyOnWriterArrayList的add方法与ArrayList的add方法有一个最大的不同点就在于，下面三句代码：

    Object[] arrayOfObject2 = Arrays.copyOf(arrayOfObject1, i + 1);
    arrayOfObject2[i] = paramE;
    setArray(arrayOfObject2);
就是这三句代码使得CopyOnWriterArrayList不会抛ConcurrentModificationException异常。他们所展现的魅力就在于copy原来的array，再在copy数组上进行add操作，这样做就完全不会影响COWIterator中的array了。

 所以CopyOnWriterArrayList所代表的核心概念就是：任何对array在结构上有所改变的操作（add、remove、clear等），CopyOnWriterArrayList都会copy现有的数据，再在copy的数据上修改，这样就不会影响COWIterator中的数据了，修改完成之后改变原有数据的引用即可。同时这样造成的代价就是产生大量的对象，同时数组的copy也是相当有损耗的。

# Comparable 和 Comparator

Java 中为我们提供了两种比较机制：Comparable 和 Comparator，他们之间有什么区别呢？今天来了解一下。

## Comparable

Comparable 在 java.lang包下，是一个接口，内部只有一个方法 compareTo()：

    public interface ComparableT {
        public int compareTo(T o);
    }

Comparable 可以让实现它的类的对象进行比较，具体的比较规则是按照 compareTo 方法中的规则进行。这种顺序称为 自然顺序。

compareTo 方法的返回值有三种情况：

    e1.compareTo(e2)  0 即 e1  e2
    e1.compareTo(e2) = 0 即 e1 = e2
    e1.compareTo(e2)  0 即 e1  e2

注意：

 1.由于 null 不是一个类，也不是一个对象，因此在重写 compareTo 方法时应该注意 e.compareTo(null) 的情况，即使 e.equals(null) 返回 false，compareTo 方法也应该主动抛出一个空指针异常 NullPointerException。

 2.Comparable 实现类重写 compareTo 方法时一般要求 e1.compareTo(e2) == 0 的结果要和 e1.equals(e2) 一致。这样将来使用 SortedSet 等根据类的自然排序进行排序的集合容器时可以保证保存的数据的顺序和想象中一致。
 有人可能好奇上面的第二点如果违反了会怎样呢？

举个例子，如果你往一个 SortedSet 中先后添加两个对象 a 和 b，a b 满足 (!a.equals(b) && a.compareTo(b) == 0)，同时也没有另外指定个 Comparator，那当你添加完 a 再添加 b 时会添加失败返回 false, SortedSet 的 size 也不会增加，因为在 SortedSet 看来它们是相同的，而 SortedSet 中是不允许重复的。

 实际上所有实现了 Comparable 接口的 Java 核心类的结果都和 equlas 方法保持一致。 
 实现了 Comparable 接口的 List 或则数组可以使用 Collections.sort() 或者 Arrays.sort() 方法进行排序。

 实现了 Comparable 接口的对象才能够直接被用作 SortedMap (SortedSet) 的 key，要不然得在外边指定 Comparator 排序规则。

因此自己定义的类如果想要使用有序的集合类，需要实现 Comparable 接口，比如：


  description 测试用的实体类 书, 实现了 Comparable 接口，自然排序
  
      author shixinzhang
      br
      data 1052016
    
    public class BookBean implements Serializable, Comparable {
        private String name;
        private int count;


    public BookBean(String name, int count) {
        this.name = name;
        this.count = count;
    }
    
    public String getName() {
        return name;
    }
    
    public void setName(String name) {
        this.name = name;
    }
    
    public int getCount() {
        return count;
    }
    
    public void setCount(int count) {
        this.count = count;
    }


​    
      重写 equals
      @param o
      @return
     
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof BookBean)) return false;
    
        BookBean bean = (BookBean) o;
    
        if (getCount() != bean.getCount()) return false;
        return getName().equals(bean.getName());
    
    }


​    
      重写 hashCode 的计算方法
      根据所有属性进行 迭代计算，避免重复
      计算 hashCode 时 计算因子 31 见得很多，是一个质数，不能再被除
      @return
     
    @Override
    public int hashCode() {
        调用 String 的 hashCode(), 唯一表示一个字符串内容
        int result = getName().hashCode();
        乘以 31, 再加上 count
        result = 31  result + getCount();
        return result;
    }
    
    @Override
    public String toString() {
        return BookBean{ +
                name=' + name + ''' +
                , count= + count +
                '}';
    }


​    
      当向 TreeSet 中添加 BookBean 时，会调用这个方法进行排序
      @param another
      @return
     
    @Override
    public int compareTo(Object another) {
        if (another instanceof BookBean){
            BookBean anotherBook = (BookBean) another;
            int result;
    
            比如这里按照书价排序
            result = getCount() - anotherBook.getCount();     
    
          或者按照 String 的比较顺序
          result = getName().compareTo(anotherBook.getName());
    
            if (result == 0){   当书价一致时，再对比书名。 保证所有属性比较一遍
                result = getName().compareTo(anotherBook.getName());
            }
            return result;
        }
         一样就返回 0
        return 0;
    }

上述代码还重写了 equlas(), hashCode() 方法，自定义的类将来可能会进行比较时，建议重写这些方法。

 这里我想表达的是在有些场景下 equals 和 compareTo 结果要保持一致，这时候不重写 equals，使用 Object.equals 方法得到的结果会有问题，比如说 HashMap.put() 方法，会先调用 key 的 equals 方法进行比较，然后才调用 compareTo。

 后面重写 compareTo 时，要判断某个相同时对比下一个属性，把所有属性都比较一次。
 
### Comparator
首先认识一下Comparator：

Comparator 是javase中的接口，位于java.util包下，该接口抽象度极高，有必要掌握该接口的使用
大多数文章告诉大家Comparator是用来排序，但我想说排序是Comparator能实现的功能之一，他不仅限于排序

排序例子：
题目描述
输入一个正整数数组，把数组里所有数字拼接起来排成一个数，打印能拼接出的所有数字中最小的一个。例如输入数组{3，32，321}，则打印出这三个数字能排成的最小数字为321323。

代码实现：
    
    import java.util.ArrayList;
    import java.util.Collections;
    import java.util.Comparator;
     
    public class Solution {
        public String PrintMinNumber(int [] s) {
            if(s==null) return null;
            String s1="";
            ArrayList<Integer> list=new ArrayList<Integer>();
            for(int i=0;i<s.length;i++){
                 list.add(s[i]);
            }
            Collections.sort(list,new Comparator<Integer>(){
                public int compare(Integer str1,Integer str2){
                    String s1=str1+""+str2;
                    String s2=str2+""+str1;
                    return s1.compareTo(s2);
                }
            });
             for(int j:list){
                    s1+=j;
                 }
            return s1;
        }
    }
    

 

一般需要做比较的逻辑都可以使用的上Comparator，最常用的场景就是排序和分组，排序常使用Arrays和Collections的sort方法，而分组则可以使用提供的divider方法。

排序和分组的区别在于: 
排序时，两个对象比较的结果有三种：大于，等于，小于。 
分组时，两个对象比较的结果只有两种：等于(两个对象属于同一组)，不等于(两个对象属于不同组)


### Java8中使用lambda实现比较器

今天先看看Lambda 表达式的简单使用：
首先：Lambda表达式的基本语法：(parameters) -> expression或（请注意语句的花括号）
(parameters) -> { statements; }

第一感觉就是这个箭头感觉有点怪，不过多用几次习惯就好，它主要是为了把参数列表与Lambda主体分隔开，箭头左边的是参数列表，右边的是Lambda主体。注意：Lambda表达式可以包含多行语句。
在用Lambda 之前，我们先看看之前写比较器的写法

    Comparator<Developer> byName = new Comparator<Developer>() {
        @Override
        public int compare(Developer o1, Developer o2) {
            return o1.getName().compareTo(o2.getName());
        }
    };

感觉也不是很复杂，没几行代码，再来看看Lambda 表达式的写法：

    Comparator<Developer> byName =
        (Developer o1, Developer o2)->o1.getName().compareTo(o2.getName());

比之前要简单许多有木有。
下面再来看看排序功能示例：
先用Collections.sort如下：

    public class TestSorting {
        public static void main(String[] args) {
            List<Developer> listDevs = getDevelopers();
            System.out.println("Before Sort");
            for (Developer developer : listDevs) {
                System.out.println(developer);
            }
            //安装年龄排序
            Collections.sort(listDevs, new Comparator<Developer>() {
                @Override
                public int compare(Developer o1, Developer o2) {
                    return o1.getAge() - o2.getAge();
                }
            });
            System.out.println("After Sort");
            for (Developer developer : listDevs) {
                System.out.println(developer);
            }
        }
        private static List<Developer> getDevelopers() {
            List<Developer> result = new ArrayList<Developer>();
            result.add(new Developer("mkyong", new BigDecimal("70000"), 33));
            result.add(new Developer("alvin", new BigDecimal("80000"), 20));
            result.add(new Developer("jason", new BigDecimal("100000"), 10));
            result.add(new Developer("iris", new BigDecimal("170000"), 55));
            return result;
        }
    }

    输出结果：
    
    Before Sort
    Developer [name=mkyong, salary=70000, age=33]
    Developer [name=alvin, salary=80000, age=20]
    Developer [name=jason, salary=100000, age=10]
    Developer [name=iris, salary=170000, age=55]
     
    After Sort
    Developer [name=jason, salary=100000, age=10]
    Developer [name=alvin, salary=80000, age=20]
    Developer [name=mkyong, salary=70000, age=33]
    Developer [name=iris, salary=170000, age=55]

看起来整个流程完全没毛病，下面再来看看Lambda的方式:

    public class TestSorting {
        public static void main(String[] args) {
            List<Developer> listDevs = getDevelopers();
            System.out.println("Before Sort");
            for (Developer developer : listDevs) {
                System.out.println(developer);
            }
            System.out.println("After Sort");
            //对比上面的代码
            listDevs.sort((Developer o1, Developer o2)->o1.getAge()-o2.getAge());
            //这样打印感觉也不错
            listDevs.forEach((developer)->System.out.println(developer));
        }
        private static List<Developer> getDevelopers() {
            List<Developer> result = new ArrayList<Developer>();
            result.add(new Developer("mkyong", new BigDecimal("70000"), 33));
            result.add(new Developer("alvin", new BigDecimal("80000"), 20));
            result.add(new Developer("jason", new BigDecimal("100000"), 10));
            result.add(new Developer("iris", new BigDecimal("170000"), 55));
            return result;
        }
    }

输出结果：

    Before Sort
    Developer [name=mkyong, salary=70000, age=33]
    Developer [name=alvin, salary=80000, age=20]
    Developer [name=jason, salary=100000, age=10]
    Developer [name=iris, salary=170000, age=55]
    
    After Sort
    Developer [name=jason, salary=100000, age=10]
    Developer [name=alvin, salary=80000, age=20]
    Developer [name=mkyong, salary=70000, age=33]
    Developer [name=iris, salary=170000, age=55]

总体来说，写法与之前有较大的改变，写的代码更少，更简便，感觉还不错。
后续会带来更多有关Java8相关的东西，毕竟作为一只程序狗，得不停的学习才能不被淘汰。Java语言都在不停的改进更新，我们有啥理由不跟上节奏呢？
由于时间问题这里只是一个简单的应用，想了解更多可到官网查找相关示例。

## 总结

Java 中的两种排序方式：

    Comparable 自然排序。（实体类实现）
    Comparator 是定制排序。（无法修改实体类时，直接在调用方创建）
    同时存在时采用 Comparator（定制排序）的规则进行比较。

对于一些普通的数据类型（比如 String, Integer, Double…），它们默认实现了Comparable 接口，实现了 compareTo 方法，我们可以直接使用。

而对于一些自定义类，它们可能在不同情况下需要实现不同的比较策略，我们可以新创建 Comparator 接口，然后使用特定的 Comparator 实现进行比较。

这就是 Comparable 和 Comparator 的区别。


## 参考文章

https://blog.csdn.net/weixin_30363263/article/details/80867590

https://www.cnblogs.com/shizhijie/p/7657049.html

https://www.cnblogs.com/xiaweicn/p/8688216.html

https://cmsblogs.com/p=1185

https://blog.csdn.net/android_hl/article/details/53228348

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
