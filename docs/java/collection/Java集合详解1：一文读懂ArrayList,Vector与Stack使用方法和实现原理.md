# Table of Contents

  * [ArrayList](#arraylist)
    * [ArrayList概述](#arraylist概述)
    * [ArrayList的继承关系](#arraylist的继承关系)
    * [底层数据结构](#底层数据结构)
    * [增删改查](#增删改查)
    * [modCount](#modcount)
    * [初始容量和扩容方式](#初始容量和扩容方式)
    * [线程安全](#线程安全)
  * [Vector](#vector)
    * [Vector简介](#vector简介)
    * [增删改查](#增删改查-1)
    * [初始容量和扩容](#初始容量和扩容)
    * [线程安全](#线程安全-1)
  * [Stack](#stack)
* [Stack](#stack-1)
  * [三个集合类之间的区别](#三个集合类之间的区别)
  * [参考文章](#参考文章)
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


    //一般讨论集合类无非就是。这里的两种数组类型更是如此
    // 1底层数据结构
    // 2增删改查方式
    // 3初始容量，扩容方式，扩容时机。
    // 4线程安全与否
    // 5是否允许空，是否允许重复，是否有序 

## ArrayList

### ArrayList概述

  ArrayList是实现List接口的动态数组，所谓动态就是它的大小是可变的。实现了所有可选列表操作，并允许包括 null 在内的所有元素。除了实现 List 接口外，此类还提供一些方法来操作内部用来存储列表的数组的大小。

  每个ArrayList实例都有一个容量，该容量是指用来存储列表元素的数组的大小。默认初始容量为10。随着ArrayList中元素的增加，它的容量也会不断的自动增长。

  在每次添加新的元素时，ArrayList都会检查是否需要进行扩容操作，扩容操作带来数据向新数组的重新拷贝，所以如果我们知道具体业务数据量，在构造ArrayList时可以给ArrayList指定一个初始容量，这样就会减少扩容时数据的拷贝问题。当然在添加大量元素前，应用程序也可以使用ensureCapacity操作来增加ArrayList实例的容量，这可以减少递增式再分配的数量。

  注意，ArrayList实现不是同步的。如果多个线程同时访问一个ArrayList实例，而其中至少一个线程从结构上修改了列表，那么它必须保持外部同步。所以为了保证同步，最好的办法是在创建时完成，以防止意外对列表进行不同步的访问：

        List list = Collections.synchronizedList(new ArrayList(...)); 
        
### ArrayList的继承关系

ArrayList继承AbstractList抽象父类，实现了List接口（规定了List的操作规范）、RandomAccess（可随机访问）、Cloneable（可拷贝）、Serializable（可序列化）。

![](https://img-blog.csdn.net/2018081819553095?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zNjM3ODkxNw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

 ### 底层数据结构

 ArrayList的底层是一个object数组，并且由trasient修饰。

    //transient Object[] elementData; //


non-private to simplify nested class access
//ArrayList底层数组不会参与序列化，而是使用另外的序列化方式。

//使用writeobject方法进行序列化,具体为什么这么做欢迎查看我之前的关于序列化的文章

//总结一下就是只复制数组中有值的位置，其他未赋值的位置不进行序列化，可以节省空间。


    //        private void writeObject(java.io.ObjectOutputStream s)
    //        throws java.io.IOException{
    //            // Write out element count, and any hidden stuff
    //            int expectedModCount = modCount;
    //            s.defaultWriteObject();
    //
    //            // Write out size as capacity for behavioural compatibility with clone()
    //            s.writeInt(size);
    //
    //            // Write out all elements in the proper order.
    //            for (int i=0; i<size; i++) {
    //                s.writeObject(elementData[i]);
    //            }
    //
    //            if (modCount != expectedModCount) {
    //                throw new ConcurrentModificationException();
    //            }
    //        }


### 增删改查

    //增删改查

添加元素时，首先判断索引是否合法，然后检测是否需要扩容，最后使用System.arraycopy方法来完成数组的复制。

这个方法无非就是使用System.arraycopy()方法将C集合(先准换为数组)里面的数据复制到elementData数组中。这里就稍微介绍下System.arraycopy()，因为下面还将大量用到该方法

。该方法的原型为：

    public static void arraycopy(Object src, int srcPos, Object dest, int destPos, int length)。

它的根本目的就是进行数组元素的复制。即从指定源数组中复制一个数组，复制从指定的位置开始，到目标数组的指定位置结束。

将源数组src从srcPos位置开始复制到dest数组中，复制长度为length，数据从dest的destPos位置开始粘贴。

    //        public void add(int index, E element) {
    //            rangeCheckForAdd(index);
    //
    //            ensureCapacityInternal(size + 1);  // Increments modCount!!
    //            System.arraycopy(elementData, index, elementData, index + 1,
    //                    size - index);
    //            elementData[index] = element;
    //            size++;
    //        }
    //

删除元素时，同样判断索引是否和法，删除的方式是把被删除元素右边的元素左移，方法同样是使用System.arraycopy进行拷贝。

    //        public E remove(int index) {
    //            rangeCheck(index);
    //
    //            modCount++;
    //            E oldValue = elementData(index);
    //
    //            int numMoved = size - index - 1;
    //            if (numMoved > 0)
    //                System.arraycopy(elementData, index+1, elementData, index,
    //                        numMoved);
    //            elementData[--size] = null; // clear to let GC do its work
    //
    //            return oldValue;
    //        }

ArrayList提供一个清空数组的办法，方法是将所有元素置为null，这样就可以让GC自动回收掉没有被引用的元素了。

    //
    //        /**
    //         * Removes all of the elements from this list.  The list will
    //         * be empty after this call returns.
    //         */
    //        public void clear() {
    //            modCount++;
    //
    //            // clear to let GC do its work
    //            for (int i = 0; i < size; i++)
    //                elementData[i] = null;
    //
    //            size = 0;
    //        }

修改元素时，只需要检查下标即可进行修改操作。

    //        public E set(int index, E element) {
    //            rangeCheck(index);
    //
    //            E oldValue = elementData(index);
    //            elementData[index] = element;
    //            return oldValue;
    //        }
    //
    //        public E get(int index) {
    //            rangeCheck(index);
    //
    //            return elementData(index);
    //        }
    //

上述方法都使用了rangeCheck方法，其实就是简单地检查下标而已。

    //        private void rangeCheck(int index) {
    //            if (index >= size)
    //                throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    //        }

### modCount

    //        protected transient int modCount = 0;

由以上代码可以看出，在一个迭代器初始的时候会赋予它调用这个迭代器的对象的mCount，如何在迭代器遍历的过程中，一旦发现这个对象的mcount和迭代器中存储的mcount不一样那就抛异常 

> 好的，下面是这个的完整解释 
> Fail-Fast 机制 
> 我们知道 java.util.ArrayList 不是线程安全的，ArrayList，那么将抛出ConcurrentModificationException，这就是所谓fail-fast策略。
>
> 这一策略在源码中的实现是通过 modCount 域，modCount 顾名思义就是修改次数，对ArrayList 内容的修改都将增加这个值，那么在迭代器初始化过程中会将这个值赋给迭代器的 expectedModCount。
>
> 在迭代过程中，判断 modCount 跟 expectedModCount 是否相等，如果不相等就表示已经有其他线程修改了 ArrayList。
>
> 所以在这里和大家建议，当大家遍历那些非线程安全的数据结构时，尽量使用迭代器

### 初始容量和扩容方式

初始容量是10，下面是扩容方法。
首先先取

    //        private static final int DEFAULT_CAPACITY = 10;
    
    扩容发生在add元素时，传入当前元素容量加一
       public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }


    这里给出初始化时的数组
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    
    这说明：如果数组还是初始数组，那么最小的扩容大小就是size+1和初始容量中较大的一个，初始容量为10。
    因为addall方法也会调用该函数，所以此时需要做判断。
    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }
    
        ensureExplicitCapacity(minCapacity);
    }
    
    //开始精确地扩容
    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;
    
        // overflow-conscious code
            如果此时扩容容量大于数组长度吗，执行grow，否则不执行。
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }

真正执行扩容的方法grow
               
扩容方式是让新容量等于旧容量的1.5被。

当新容量大于最大数组容量时，执行大数扩容

    //        private void grow(int minCapacity) {
    //            // overflow-conscious code
    //            int oldCapacity = elementData.length;
    //            int newCapacity = oldCapacity + (oldCapacity >> 1);
    //            if (newCapacity - minCapacity < 0)
    //                newCapacity = minCapacity;
    //            if (newCapacity - MAX_ARRAY_SIZE > 0)
    //                newCapacity = hugeCapacity(minCapacity);
    //            // minCapacity is usually close to size, so this is a win:
    //            elementData = Arrays.copyOf(elementData, newCapacity);
    //        }

当新容量大于最大数组长度，有两种情况，一种是溢出，抛异常，一种是没溢出，返回整数的最大值。

    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }


在这里有一个疑问，为什么每次扩容处理会是1.5倍，而不是2.5、3、4倍呢？通过google查找，发现1.5倍的扩容是最好的倍数。因为一次性扩容太大(例如2.5倍)可能会浪费更多的内存(1.5倍最多浪费33%，而2.5被最多会浪费60%，3.5倍则会浪费71%……)。但是一次性扩容太小，需要多次对数组重新分配内存，对性能消耗比较严重。所以1.5倍刚刚好，既能满足性能需求，也不会造成很大的内存消耗。

  处理这个ensureCapacity()这个扩容数组外，ArrayList还给我们提供了将底层数组的容量调整为当前列表保存的实际元素的大小的功能。它可以通过trimToSize()方法来实现。该方法可以最小化ArrayList实例的存储量。

    public void trimToSize() {
        modCount++;
        int oldCapacity = elementData.length;
        if (size < oldCapacity) {
            elementData = Arrays.copyOf(elementData, size);
        }
    }
### 线程安全

ArrayList是线程不安全的。在其迭代器iteator中，如果有多线程操作导致modcount改变，会执行fastfail。抛出异常。

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }

## Vector

### Vector简介

Vector可以实现可增长的对象数组。与数组一样，它包含可以使用整数索引进行访问的组件。不过，Vector的大小是可以增加或者减小的，以便适应创建Vector后进行添加或者删除操作。

Vector实现List接口，继承AbstractList类，所以我们可以将其看做队列，支持相关的添加、删除、修改、遍历等功能。

Vector实现RandmoAccess接口，即提供了随机访问功能，提供提供快速访问功能。在Vector我们可以直接访问元素。

Vector 实现了Cloneable接口，支持clone()方法，可以被克隆。

vector底层数组不加transient，序列化时会全部复制

     protected Object[] elementData;


 
    //        private void writeObject(java.io.ObjectOutputStream s)
    //            throws java.io.IOException {
    //            final java.io.ObjectOutputStream.PutField fields = s.putFields();
    //            final Object[] data;
    //            synchronized (this) {
    //                fields.put("capacityIncrement", capacityIncrement);
    //                fields.put("elementCount", elementCount);
    //                data = elementData.clone();
    //            }
    //            fields.put("elementData", data);
    //            s.writeFields();
    //        }

Vector除了iterator外还提供Enumeration枚举方法，不过现在比较过时。

    //        public Enumeration<E> elements() {
    //            return new Enumeration<E>() {
    //                int count = 0;
    //
    //                public boolean hasMoreElements() {
    //                    return count < elementCount;
    //                }
    //
    //                public E nextElement() {
    //                    synchronized (Vector.this) {
    //                        if (count < elementCount) {
    //                            return elementData(count++);
    //                        }
    //                    }
    //                    throw new NoSuchElementException("Vector Enumeration");
    //                }
    //            };
    //        }
    //


### 增删改查

vector的增删改查既提供了自己的实现，也继承了abstractList抽象类的部分方法。
下面的方法是vector自己实现的。

    //
    //    public synchronized E elementAt(int index) {
    //        if (index >= elementCount) {
    //            throw new ArrayIndexOutOfBoundsException(index + " >= " + elementCount);
    //        }
    //
    //        return elementData(index);
    //    }
    //
    //
    
    //    public synchronized void setElementAt(E obj, int index) {
    //        if (index >= elementCount) {
    //            throw new ArrayIndexOutOfBoundsException(index + " >= " +
    //                    elementCount);
    //        }
    //        elementData[index] = obj;
    //    }
    //

 
    //    public synchronized void removeElementAt(int index) {
    //        modCount++;
    //        if (index >= elementCount) {
    //            throw new ArrayIndexOutOfBoundsException(index + " >= " +
    //                    elementCount);
    //        }
    //        else if (index < 0) {
    //            throw new ArrayIndexOutOfBoundsException(index);
    //        }
    //        int j = elementCount - index - 1;
    //        if (j > 0) {
    //            System.arraycopy(elementData, index + 1, elementData, index, j);
    //        }
    //        elementCount--;
    //        elementData[elementCount] = null; /* to let gc do its work */
    //    }

   
    //    public synchronized void insertElementAt(E obj, int index) {
    //        modCount++;
    //        if (index > elementCount) {
    //            throw new ArrayIndexOutOfBoundsException(index
    //                    + " > " + elementCount);
    //        }
    //        ensureCapacityHelper(elementCount + 1);
    //        System.arraycopy(elementData, index, elementData, index + 1, elementCount - index);
    //        elementData[index] = obj;
    //        elementCount++;
    //    }
    //
    
    //    public synchronized void addElement(E obj) {
    //        modCount++;
    //        ensureCapacityHelper(elementCount + 1);
    //        elementData[elementCount++] = obj;
    //    }

### 初始容量和扩容  
扩容方式与ArrayList基本一样，但是扩容时不是1.5倍扩容，而是有一个扩容增量。


    //    protected int elementCount;
    
    //    protected int capacityIncrement;
    //
    //
    //    }
    //    public Vector() {
    //        this(10);
    //    }

capacityIncrement：向量的大小大于其容量时，容量自动增加的量。如果在创建Vector时，指定了capacityIncrement的大小；则，每次当Vector中动态数组容量增加时>，增加的大小都是capacityIncrement。如果容量的增量小于等于零，则每次需要增大容量时，向量的容量将增大一倍。



    //        public synchronized void ensureCapacity(int minCapacity) {
    //            if (minCapacity > 0) {
    //                modCount++;
    //                ensureCapacityHelper(minCapacity);
    //            }
    //        }
    //        private void ensureCapacityHelper(int minCapacity) {
    //            // overflow-conscious code
    //            if (minCapacity - elementData.length > 0)
    //                grow(minCapacity);
    //        }
    //
    //        private void grow(int minCapacity) {
    //            // overflow-conscious code
    //            int oldCapacity = elementData.length;
    //            int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
    //                    capacityIncrement : oldCapacity);
    //            if (newCapacity - minCapacity < 0)
    //                newCapacity = minCapacity;
    //            if (newCapacity - MAX_ARRAY_SIZE > 0)
    //                newCapacity = hugeCapacity(minCapacity);
    //            elementData = Arrays.copyOf(elementData, newCapacity);
    //        }


下面是扩容过程示意图


![](https://img-blog.csdn.net/20180818200637720?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zNjM3ODkxNw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![](https://img-blog.csdn.net/20180818200704724?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zNjM3ODkxNw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

![](https://img-blog.csdn.net/20180818200735561?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl8zNjM3ODkxNw==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)

### 线程安全

vector大部分方法都使用了synchronized修饰符，所以他是线层安全的集合类。

## Stack
我们最常用的数据结构之一大概就是stack了。在实际的程序执行，方法调用的过程中都离不开stack。那么，在一个成熟的类库里面，它的实现是怎么样的呢？也许平时我们实践的时候也会尝试着去写一个stack的实现玩玩。这里，我们就仔细的分析一下jdk里的详细实现。

# Stack

    如果我们去查jdk的文档，我们会发现stack是在java.util这个包里。它对应的一个大致的类关系图如下：

![](http://dl.iteye.com/upload/attachment/0081/2496/006da63f-388e-3669-b57f-1cdd1909d5f8.jpg)

    通过继承Vector类，Stack类可以很容易的实现他本身的功能。因为大部分的功能在Vector里面已经提供支持了。
在Java中Stack类表示后进先出（LIFO）的对象堆栈。栈是一种非常常见的数据结构，它采用典型的先进后出的操作方式完成的。


Stack通过五个操作对Vector进行扩展，允许将向量视为堆栈。这个五个操作如下：

> empty()
>
> 测试堆栈是否为空。
>
> peek()
>
> 查看堆栈顶部的对象，但不从堆栈中移除它。
>
> pop()
>
> 移除堆栈顶部的对象，并作为此函数的值返回该对象。
>
> push(E item)
>
> 把项压入堆栈顶部。
>
> search(Object o)
>
> 返回对象在堆栈中的位置，以 1 为基数。

Stack继承Vector，他对Vector进行了简单的扩展：

public class Stack<E> extends Vector<E>
  Stack的实现非常简单，仅有一个构造方法，五个实现方法（从Vector继承而来的方法不算与其中），同时其实现的源码非常简单

    /**
     * 构造函数
     */
    public Stack() {
    }
    
    /**
     *  push函数：将元素存入栈顶
     */
    public E push(E item) {
        // 将元素存入栈顶。
        // addElement()的实现在Vector.java中
        addElement(item);
    
        return item;
    }
    
    /**
     * pop函数：返回栈顶元素，并将其从栈中删除
     */
    public synchronized E pop() {
        E    obj;
        int    len = size();
    
        obj = peek();
        // 删除栈顶元素，removeElementAt()的实现在Vector.java中
        removeElementAt(len - 1);
    
        return obj;
    }
    
    /**
     * peek函数：返回栈顶元素，不执行删除操作
     */
    public synchronized E peek() {
        int    len = size();
    
        if (len == 0)
            throw new EmptyStackException();
        // 返回栈顶元素，elementAt()具体实现在Vector.java中
        return elementAt(len - 1);
    }
    
    /**
     * 栈是否为空
     */
    public boolean empty() {
        return size() == 0;
    }
    
    /**
     *  查找“元素o”在栈中的位置：由栈底向栈顶方向数
     */
    public synchronized int search(Object o) {
        // 获取元素索引，elementAt()具体实现在Vector.java中
        int i = lastIndexOf(o);
    
        if (i >= 0) {
            return size() - i;
        }
        return -1;
    }

Stack的源码很多都是基于Vector，所以这里不再累述

## 三个集合类之间的区别

ArrayList的优缺点

从上面的几个过程总结一下ArrayList的优缺点。ArrayList的优点如下：
>
> 1、ArrayList底层以数组实现，是一种随机访问模式，再加上它实现了RandomAccess接口，因此查找也就是get的时候非常快
>
> 2、ArrayList在顺序添加一个元素的时候非常方便，只是往数组里面添加了一个元素而已

不过ArrayList的缺点也十分明显：
>
> 1、删除元素的时候，涉及到一次元素复制，如果要复制的元素很多，那么就会比较耗费性能
>
> 2、插入元素的时候，涉及到一次元素复制，如果要复制的元素很多，那么就会比较耗费性能
>
> 因此，ArrayList比较适合顺序添加、随机访问的场景。

 

ArrayList和Vector的区别

> ArrayList是线程非安全的，这很明显，因为ArrayList中所有的方法都不是同步的，在并发下一定会出现线程安全问题。那么我们想要使用ArrayList并且让它线程安全怎么办？一个方法是用Collections.synchronizedList方法把你的ArrayList变成一个线程安全的List，比如：
>
>     List<String> synchronizedList = Collections.synchronizedList(list);
>     synchronizedList.add("aaa");
>     synchronizedList.add("bbb");
>     for (int i = 0; i < synchronizedList.size(); i++)
>     {
>         System.out.println(synchronizedList.get(i));
>     }

另一个方法就是Vector，它是ArrayList的线程安全版本，其实现90%和ArrayList都完全一样，区别在于：

> 1、Vector是线程安全的，ArrayList是线程非安全的
>
> 2、Vector可以指定增长因子，如果该增长因子指定了，那么扩容的时候会每次新的数组大小会在原数组的大小基础上加上增长因子；如果不指定增长因子，那么就给原数组大小*2，源代码是这样的：

    int newCapacity = oldCapacity + ((capacityIncrement > 0) ?
                                     capacityIncrement : oldCapacity);


## 参考文章

https://www.cnblogs.com/williamjie/p/11158523.html

https://www.cnblogs.com/shenzhichipingguo/p/10075212.html

https://www.cnblogs.com/rnmb/p/6553711.html

https://blog.csdn.net/u011419651/article/details/83831156

https://www.jianshu.com/p/c4027084ac43

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
