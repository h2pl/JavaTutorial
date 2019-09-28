# Table of Contents

  * [Colletion，iterator，comparable](#colletion，iterator，comparable)
  * [List](#list)
  * [Map](#map)
  * [CHM](#chm)
  * [Set](#set)
  * [Linkedhashmap](#linkedhashmap)
  * [collections和Arrays工具类](#collections和arrays工具类)
  * [comparable和comparator](#comparable和comparator)
  * [treemap和treeset](#treemap和treeset)
  * [微信公众号](#微信公众号)
    * [Java技术江湖](#java技术江湖)
    * [个人公众号：黄小斜](#个人公众号：黄小斜)


---
title: Java集合框架学习总结
date: 2018-07-08 22:03:44
tags:
	- Java集合
categories:
	- 后端
	- 技术总结
---



这篇总结是基于之前博客内容的一个整理和回顾。



这里先简单地总结一下，更多详细内容请参考我的专栏：深入浅出Java核心技术

https://blog.csdn.net/column/details/21930.html

里面有包括Java集合类在内的众多Java核心技术系列文章。


以下总结不保证全对，如有错误，还望能够指出。谢谢
<!-- more -->



## Colletion，iterator，comparable


一般认为Collection是最上层接口，但是hashmap实际上实现的是Map接口。iterator是迭代器，是实现iterable接口的类必须要提供的一个东西，能够使用for(i : A) 这种方式实现的类型能提供迭代器，以前有一个enumeration，现在早弃用了。


## List


List接口下的实现类有ArrayList，linkedlist，vector等等，一般就是用这两个，用法不多说，老生常谈。
ArrayList的扩容方式是1.5倍扩容，这样扩容避免2倍扩容可能浪费空间，是一种折中的方案。
另外他不是线程安全，vector则是线程安全的，它是两倍扩容的。


linkedlist没啥好说的，多用于实现链表。




## Map


map永远都是重头戏。


hashmap是数组和链表的组合结构，数组是一个Entry数组，entry是k-V键值对类型，所以一个entry数组存着很entry节点，一个entry的位置通过key的hashcode方法，再进行hash（移位等操作），最后与表长-1进行相与操作，其实就是取hash值到的后n - 1位，n代表表长是2的n次方。


hashmap的默认负载因子是0.75，阈值是16 * 0.75 = 12；初始长度为16；


hashmap的增删改查方式比较简单，都是遍历，替换。有一点要注意的是key相等时，替换元素，不相等时连成链表。


除此之外，1.8jdk改进了hashmap，当链表上的元素个数超过8个时自动转化成红黑树，节点变成树节点，以提高搜索效率和插入效率到logn。


还有一点值得一提的是，hashmap的扩容操作，由于hashmap非线程安全，扩容时如果多线程并发进行操作，则可能有两个线程分别操作新表和旧表，导致节点成环，查询时会形成死锁。chm避免了这个问题。


另外，扩容时会将旧表元素移到新表，原来的版本移动时会有rehash操作，每个节点都要rehash，非常不方便，而1.8改成另一种方式，对于同一个index下的链表元素，由于一个元素的hash值在扩容后只有两种情况，要么是hash值不变，要么是hash值变为原来值+2^n次方，这是因为表长翻倍，所以hash值取后n位，第一位要么是0要么是1，所以hash值也只有两种情况。这两种情况的元素分别加到两个不同的链表。这两个链表也只需要分别放到新表的两个位置即可，是不是很酷。


最后有一个比较冷门的知识点，hashmap1.7版本链表使用的是节点的头插法，扩容时转移链表仍然使用头插法，这样的结果就是扩容后链表会倒置，而hashmap.1.8在插入时使用尾插法，扩容时使用头插法，这样可以保证顺序不变。


## CHM


concurrenthashmap也稍微提一下把，chm1.7使用分段锁来控制并发，每个segment对应一个segmentmask，通过key的hash值相与这个segmentmask得到segment位置，然后在找到具体的entry数组下标。所以chm需要维护多个segment，每个segment对应一个数组。分段锁使用的是reetreetlock可重入锁实现。查询时不加锁。


1.8则放弃使用分段锁，改用cas+synchronized方式实现并发控制，查询时不加锁，插入时如果没有冲突直接cas到成功为止，有冲突则使用synchronized插入。




## Set


set就是hashmap将value固定为一个object，只存key元素包装成一个entry即可，其他不变。


## Linkedhashmap


在原来hashmap基础上将所有的节点依据插入的次序另外连成一个链表。用来保持顺序，可以使用它实现lru缓存，当访问命中时将节点移到队头，当插入元素超过长度时，删除队尾元素即可。


## collections和Arrays工具类
两个工具类分别操作集合和数组，可以进行常用的排序，合并等操作。


## comparable和comparator
实现comparable接口可以让一个类的实例互相使用compareTo方法进行比较大小，可以自定义比较规则，comparator则是一个通用的比较器，比较指定类型的两个元素之间的大小关系。


## treemap和treeset

主要是基于红黑树实现的两个数据结构，可以保证key序列是有序的，获取sortedset就可以顺序打印key值了。其中涉及到红黑树的插入和删除，调整等操作，比较复杂，这里就不细说了。

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
