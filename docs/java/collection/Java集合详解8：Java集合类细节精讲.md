# Table of Contents

  * [初始容量](#初始容量)
  * [asList的缺陷](#aslist的缺陷)
    * [避免使用基本数据类型数组转换为列表](#避免使用基本数据类型数组转换为列表)
    * [asList产生的列表不可操作](#aslist产生的列表不可操作)
  * [subList的缺陷](#sublist的缺陷)
    * [subList返回仅仅只是一个视图](#sublist返回仅仅只是一个视图)
    * [subList生成子列表后，不要试图去操作原列表](#sublist生成子列表后，不要试图去操作原列表)
    * [推荐使用subList处理局部列表](#推荐使用sublist处理局部列表)
  * [保持compareTo和equals同步](#保持compareto和equals同步)
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

今天我们来探索一下Java集合类中的一些技术细节。主要是对一些比较容易被遗漏和误解的知识点做一些讲解和补充。可能不全面，还请谅解。

## 初始容量

集合是我们在Java编程中使用非常广泛的，它就像大海，海纳百川，像万能容器，盛装万物，而且这个大海，万能容器还可以无限变大（如果条件允许）。当这个海、容器的量变得非常大的时候，它的初始容量就会显得很重要了，因为挖海、扩容是需要消耗大量的人力物力财力的。

同样的道理，Collection的初始容量也显得异常重要。所以：对于已知的情景，请为集合指定初始容量。

    public static void main(String[] args) {
        StudentVO student = null;
        long begin1 = System.currentTimeMillis();
        List<StudentVO> list1 = new ArrayList<>();
        for(int i = 0 ; i < 1000000; i++){
            student = new StudentVO(i,"chenssy_"+i,i);
            list1.add(student);
        }
        long end1 = System.currentTimeMillis();
        System.out.println("list1 time：" + (end1 - begin1));
        
        long begin2 = System.currentTimeMillis();
        List<StudentVO> list2 = new ArrayList<>(1000000);
        for(int i = 0 ; i < 1000000; i++){
            student = new StudentVO(i,"chenssy_"+i,i);
            list2.add(student);
        }
        long end2 = System.currentTimeMillis();
        System.out.println("list2 time：" + (end2 - begin2));
    }
上面代码两个list都是插入1000000条数据，只不过list1没有没有申请初始化容量，而list2初始化容量1000000。那运行结果如下：

    list1 time：1638
    list2 time：921

从上面的运行结果我们可以看出list2的速度是list1的两倍左右。在前面LZ就提过，ArrayList的扩容机制是比较消耗资源的。我们先看ArrayList的add方法：

    public boolean add(E e) {  
            ensureCapacity(size + 1);   
            elementData[size++] = e;  
            return true;  
        }  
    
    public void ensureCapacity(int minCapacity) {  
        modCount++;         //修改计数器
        int oldCapacity = elementData.length;    
        //当前需要的长度超过了数组长度，进行扩容处理
        if (minCapacity > oldCapacity) {  
            Object oldData[] = elementData;  
            //新的容量 = 旧容量 * 1.5 + 1
            int newCapacity = (oldCapacity * 3)/2 + 1;  
                if (newCapacity < minCapacity)  
                    newCapacity = minCapacity;  
          //数组拷贝，生成新的数组 
          elementData = Arrays.copyOf(elementData, newCapacity);  
        }  
    }
ArrayList每次新增一个元素，就会检测ArrayList的当前容量是否已经到达临界点，如果到达临界点则会扩容1.5倍。然而ArrayList的扩容以及数组的拷贝生成新的数组是相当耗资源的。所以若我们事先已知集合的使用场景，知道集合的大概范围，我们最好是指定初始化容量，这样对资源的利用会更加好，尤其是大数据量的前提下，效率的提升和资源的利用会显得更加具有优势。

## asList的缺陷

在实际开发过程中我们经常使用asList讲数组转换为List，这个方法使用起来非常方便，但是asList方法存在几个缺陷：

### 避免使用基本数据类型数组转换为列表

使用8个基本类型数组转换为列表时会存在一个比较有味的缺陷。先看如下程序：

    public static void main(String[] args) {
            int[] ints = {1,2,3,4,5};
            List list = Arrays.asList(ints);
            System.out.println("list'size：" + list.size());
        }
    ------------------------------------
    outPut：
    list'size：1
程序的运行结果并没有像我们预期的那样是5而是逆天的1，这是什么情况？先看源码：

    public static <T> List<T> asList(T... a) {
            return new ArrayList<>(a);
        }

asList接受的参数是一个泛型的变长参数，我们知道基本数据类型是无法发型化的，也就是说8个基本类型是无法作为asList的参数的， 要想作为泛型参数就必须使用其所对应的包装类型。但是这个这个实例中为什么没有出错呢？

因为该实例是将int类型的数组当做其参数，而在Java中数组是一个对象，它是可以泛型化的。所以该例子是不会产生错误的。既然例子是将整个int类型的数组当做泛型参数，那么经过asList转换就只有一个int 的列表了。如下：

    public static void main(String[] args) {
        int[] ints = {1,2,3,4,5};
        List list = Arrays.asList(ints);
        System.out.println("list 的类型:" + list.get(0).getClass());
        System.out.println("list.get(0) == ints：" + list.get(0).equals(ints));
    }
--------------------------------------------
outPut:
list 的类型:class [I
list.get(0) == ints：true
从这个运行结果我们可以充分证明list里面的元素就是int数组。弄清楚这点了，那么修改方法也就一目了然了：将int 改变为Integer。

    public static void main(String[] args) {
            Integer[] ints = {1,2,3,4,5};
            List list = Arrays.asList(ints);
            System.out.println("list'size：" + list.size());
            System.out.println("list.get(0) 的类型:" + list.get(0).getClass());
            System.out.println("list.get(0) == ints[0]：" + list.get(0).equals(ints[0]));
        }
    ----------------------------------------
    outPut:
    list'size：5
    list.get(0) 的类型:class java.lang.Integer
    list.get(0) == ints[0]：true



### asList产生的列表不可操作

对于上面的实例我们再做一个小小的修改：

    public static void main(String[] args) {
            Integer[] ints = {1,2,3,4,5};
            List list = Arrays.asList(ints);
            list.add(6);
        }

该实例就是讲ints通过asList转换为list 类别，然后再通过add方法加一个元素，这个实例简单的不能再简单了，但是运行结果呢？打出我们所料：

    Exception in thread "main" java.lang.UnsupportedOperationException
        at java.util.AbstractList.add(Unknown Source)
        at java.util.AbstractList.add(Unknown Source)
        at com.chenssy.test.arrayList.AsListTest.main(AsListTest.java:10)

运行结果尽然抛出UnsupportedOperationException异常，该异常表示list不支持add方法。这就让我们郁闷了，list怎么可能不支持add方法呢？难道jdk脑袋堵塞了？我们再看asList的源码：

    public static <T> List<T> asList(T... a) {
            return new ArrayList<>(a);
        }
asList接受参数后，直接new 一个ArrayList，到这里看应该是没有错误的啊？别急，再往下看:

    private static class ArrayList<E> extends AbstractList<E>
        implements RandomAccess, java.io.Serializable{
            private static final long serialVersionUID = -2764017481108945198L;
            private final E[] a;
    
            ArrayList(E[] array) {
                if (array==null)
                    throw new NullPointerException();
                a = array;
            }
            //.................
        }
这是ArrayList的源码,从这里我们可以看出,此ArrayList不是java.util.ArrayList，他是Arrays的内部类。

该内部类提供了size、toArray、get、set、indexOf、contains方法，而像add、remove等改变list结果的方法从AbstractList父类继承过来，同时这些方法也比较奇葩，它直接抛出UnsupportedOperationException异常：

    public boolean add(E e) {
            add(size(), e);
            return true;
        }
        
        public E set(int index, E element) {
            throw new UnsupportedOperationException();
        }
        
        public void add(int index, E element) {
            throw new UnsupportedOperationException();
        }
        
        public E remove(int index) {
            throw new UnsupportedOperationException();
        }

通过这些代码可以看出asList返回的列表只不过是一个披着list的外衣，它并没有list的基本特性（变长）。该list是一个长度不可变的列表，传入参数的数组有多长，其返回的列表就只能是多长。所以：：不要试图改变asList返回的列表，否则你会自食苦果。

## subList的缺陷
我们经常使用subString方法来对String对象进行分割处理，同时我们也可以使用subList、subMap、subSet来对List、Map、Set进行分割处理，但是这个分割存在某些瑕疵。

### subList返回仅仅只是一个视图

首先我们先看如下实例：

public static void main(String[] args) {
        List<Integer> list1 = new ArrayList<Integer>();
        list1.add(1);
        list1.add(2);
        
        //通过构造函数新建一个包含list1的列表 list2
        List<Integer> list2 = new ArrayList<Integer>(list1);
        
        //通过subList生成一个与list1一样的列表 list3
        List<Integer> list3 = list1.subList(0, list1.size());
        
        //修改list3
        list3.add(3);
        
        System.out.println("list1 == list2：" + list1.equals(list2));
        System.out.println("list1 == list3：" + list1.equals(list3));
    }

这个例子非常简单，无非就是通过构造函数、subList重新生成一个与list1一样的list，然后修改list3，最后比较list1 == list2?、list1 == list3?。

按照我们常规的思路应该是这样的：因为list3通过add新增了一个元素，那么它肯定与list1不等，而list2是通过list1构造出来的，所以应该相等，所以结果应该是：

    list1 == list2：true
    list1 == list3: false
首先我们先不论结果的正确与否，我们先看subList的源码：

    public List<E> subList(int fromIndex, int toIndex) {
            subListRangeCheck(fromIndex, toIndex, size);
            return new SubList(this, 0, fromIndex, toIndex);
    }

subListRangeCheck方式是判断fromIndex、toIndex是否合法，如果合法就直接返回一个subList对象，注意在产生该new该对象的时候传递了一个参数 this ，该参数非常重要，因为他代表着原始list。

/**
     * 继承AbstractList类，实现RandomAccess接口
     */
    private class SubList extends AbstractList<E> implements RandomAccess {
        private final AbstractList<E> parent;    //列表
        private final int parentOffset;   
        private final int offset;
        int size;

        //构造函数
        SubList(AbstractList<E> parent,
                int offset, int fromIndex, int toIndex) {
            this.parent = parent;
            this.parentOffset = fromIndex;
            this.offset = offset + fromIndex;
            this.size = toIndex - fromIndex;
            this.modCount = ArrayList.this.modCount;
        }
    
        //set方法
        public E set(int index, E e) {
            rangeCheck(index);
            checkForComodification();
            E oldValue = ArrayList.this.elementData(offset + index);
            ArrayList.this.elementData[offset + index] = e;
            return oldValue;
        }
    
        //get方法
        public E get(int index) {
            rangeCheck(index);
            checkForComodification();
            return ArrayList.this.elementData(offset + index);
        }
    
        //add方法
        public void add(int index, E e) {
            rangeCheckForAdd(index);
            checkForComodification();
            parent.add(parentOffset + index, e);
            this.modCount = parent.modCount;
            this.size++;
        }
    
        //remove方法
        public E remove(int index) {
            rangeCheck(index);
            checkForComodification();
            E result = parent.remove(parentOffset + index);
            this.modCount = parent.modCount;
            this.size--;
            return result;
        }
    }

该SubLsit是ArrayList的内部类，它与ArrayList一样，都是继承AbstractList和实现RandomAccess接口。同时也提供了get、set、add、remove等list常用的方法。但是它的构造函数有点特殊，在该构造函数中有两个地方需要注意：

1、this.parent = parent;而parent就是在前面传递过来的list，也就是说this.parent就是原始list的引用。

2、this.offset = offset + fromIndex;this.parentOffset = fromIndex;。同时在构造函数中它甚至将modCount（fail-fast机制）传递过来了。

我们再看get方法，在get方法中return ArrayList.this.elementData(offset + index);

这段代码可以清晰表明get所返回就是原列表offset + index位置的元素。同样的道理还有add方法里面的：

parent.add(parentOffset + index, e);
this.modCount = parent.modCount;
remove方法里面的

E result = parent.remove(parentOffset + index);
this.modCount = parent.modCount;

诚然，到了这里我们可以判断subList返回的SubList同样也是AbstractList的子类，同时它的方法如get、set、add、remove等都是在原列表上面做操作，它并没有像subString一样生成一个新的对象。

所以subList返回的只是原列表的一个视图，它所有的操作最终都会作用在原列表上。

那么从这里的分析我们可以得出上面的结果应该恰恰与我们上面的答案相反：

list1 == list2：false
list1 == list3：true



### subList生成子列表后，不要试图去操作原列表

从上面我们知道subList生成的子列表只是原列表的一个视图而已，如果我们操作子列表它产生的作用都会在原列表上面表现，但是如果我们操作原列表会产生什么情况呢？

public static void main(String[] args) {
        List<Integer> list1 = new ArrayList<Integer>();
        list1.add(1);
        list1.add(2);
        
        //通过subList生成一个与list1一样的列表 list3
        List<Integer> list3 = list1.subList(0, list1.size());
        //修改list1
        list1.add(3);
        
        System.out.println("list1'size：" + list1.size());
        System.out.println("list3'size：" + list3.size());
    }
该实例如果不产生意外，那么他们两个list的大小都应该都是3，但是偏偏事与愿违，事实上我们得到的结果是这样的：

    list1'size：3
    Exception in thread "main" java.util.ConcurrentModificationException
        at java.util.ArrayList$SubList.checkForComodification(Unknown Source)
        at java.util.ArrayList$SubList.size(Unknown Source)
        at com.chenssy.test.arrayList.SubListTest.main(SubListTest.java:17)
list1正常输出，但是list3就抛出ConcurrentModificationException异常，看过我另一篇博客的同仁肯定对这个异常非常，fail-fast？不错就是fail-fast机制，在fail-fast机制中，LZ花了很多力气来讲述这个异常，所以这里LZ就不对这个异常多讲了。我们再看size方法：

    public int size() {
                checkForComodification();
                return this.size;
            }
size方法首先会通过checkForComodification验证，然后再返回this.size。

    private void checkForComodification() {
                if (ArrayList.this.modCount != this.modCount)
                    throw new ConcurrentModificationException();
            }
该方法表明当原列表的modCount与this.modCount不相等时就会抛出ConcurrentModificationException。

同时我们知道modCount 在new的过程中 "继承"了原列表modCount，只有在修改该列表（子列表）时才会修改该值（先表现在原列表后作用于子列表）。

而在该实例中我们是操作原列表，原列表的modCount当然不会反应在子列表的modCount上啦，所以才会抛出该异常。

对于子列表视图，它是动态生成的，生成之后就不要操作原列表了，否则必然都导致视图的不稳定而抛出异常。最好的办法就是将原列表设置为只读状态，要操作就操作子列表：

//通过subList生成一个与list1一样的列表 list3

    List<Integer> list3 = list1.subList(0, list1.size());

//对list1设置为只读状态

    list1 = Collections.unmodifiableList(list1);

### 推荐使用subList处理局部列表

在开发过程中我们一定会遇到这样一个问题：获取一堆数据后，需要删除某段数据。例如，有一个列表存在1000条记录，我们需要删除100-200位置处的数据，可能我们会这样处理：

    for(int i = 0 ; i < list1.size() ; i++){
       if(i >= 100 && i <= 200){
           list1.remove(i);
           /*
            * 当然这段代码存在问题，list remove之后后面的元素会填充上来，
             * 所以需要对i进行简单的处理，当然这个不是这里讨论的问题。
             */
       }
    }
这个应该是我们大部分人的处理方式吧，其实还有更好的方法，利用subList。在前面LZ已经讲过，子列表的操作都会反映在原列表上。所以下面一行代码全部搞定：

    list1.subList(100, 200).clear();
简单而不失华丽！！！！！


## 保持compareTo和equals同步
在Java中我们常使用Comparable接口来实现排序，其中compareTo是实现该接口方法。我们知道compareTo返回0表示两个对象相等，返回正数表示大于，返回负数表示小于。同时我们也知道equals也可以判断两个对象是否相等，那么他们两者之间是否存在关联关系呢？

    public class Student implements Comparable<Student>{
        private String id;
        private String name;
        private int age;
        
        public Student(String id,String name,int age){
            this.id = id;
            this.name = name;
            this.age = age;
        }
    
        public boolean equals(Object obj){
            if(obj == null){
                return false;
            }
            
            if(this == obj){
                return true;
            }
            
            if(obj.getClass() != this.getClass()){
                return false;
            }
            
            Student student = (Student)obj;
            if(!student.getName().equals(getName())){
                return false;
            }
            
            return true;
        }
        
        public int compareTo(Student student) {
            return this.age - student.age;
        }
    
        /** 省略getter、setter方法 */
    }
Student类实现Comparable接口和实现equals方法，其中compareTo是根据age来比对的，equals是根据name来比对的。

    public static void main(String[] args){
            List<Student> list = new ArrayList<>();
            list.add(new Student("1", "chenssy1", 24));
            list.add(new Student("2", "chenssy1", 26));
            
            Collections.sort(list);   //排序
            
            Student student = new Student("2", "chenssy1", 26);
            
            //检索student在list中的位置
            int index1 = list.indexOf(student);
            int index2 = Collections.binarySearch(list, student);
            
            System.out.println("index1 = " + index1);
            System.out.println("index2 = " + index2);
        }

按照常规思路来说应该两者index是一致的，因为他们检索的是同一个对象，但是非常遗憾，其运行结果：

index1 = 0
index2 = 1

> 为什么会产生这样不同的结果呢？这是因为indexOf和binarySearch的实现机制不同。
>
> indexOf是基于equals来实现的只要equals返回TRUE就认为已经找到了相同的元素。
>
> 而binarySearch是基于compareTo方法的，当compareTo返回0 时就认为已经找到了该元素。
>
> 在我们实现的Student类中我们覆写了compareTo和equals方法，但是我们的compareTo、equals的比较依据不同，一个是基于age、一个是基于name。

比较依据不同那么得到的结果很有可能会不同。所以知道了原因，我们就好修改了：将两者之间的比较依据保持一致即可。

对于compareTo和equals两个方法我们可以总结为：compareTo是判断元素在排序中的位置是否相等，equals是判断元素是否相等，既然一个决定排序位置，一个决定相等，所以我们非常有必要确保当排序位置相同时，其equals也应该相等。

使其相等的方式就是两者应该依附于相同的条件。当compareto相等时equals也应该相等，而compareto不相等时equals不应该相等，并且compareto依据某些属性来决定排序。


## 参考文章
https://www.cnblogs.com/galibujianbusana/p/6600226.html

http://blog.itpub.net/69906029/viewspace-2641300/

https://www.cnblogs.com/itxiaok/p/10356553.html


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
