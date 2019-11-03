本仓库为【Java工程师技术指南】基础是立身之本，力求打造最完整最实用的Java工程师学习指南！

点击关注[微信公众号](#微信公众号)及时获取笔主最新更新文章，并可免费领取Java工程师必备学习资源。

<p align="center">
<a href="https://github.com/h2pl/Java-Tutorial" target="_blank">
    <img src="https://s2.ax1x.com/2019/09/23/uFaedP.jpg" width="200" height="160"/>
</a>
</p>

<p align="center">
  <a href="https://how2playlife.com/"><img src="https://img.shields.io/badge/阅读-read-brightgreen.svg" alt="阅读"></a>
  <a href="#微信公众号"><img src="https://img.shields.io/badge/chat-微信群-blue.svg" alt="微信公众号"></a>
</p>

推荐使用 https://how2playlife.com/ 在线阅读，在线阅读内容本仓库同步一致。这种方式阅读的优势在于：左侧边栏有目录，阅读体验更好。

## 目录

- [Java](#Java)
    - [基础](#基础)
    - [容器](#容器)
    - [并发](#并发)
    - [JVM](#jvm)
    - [Java网络编程](#Java网络编程)
    - [设计模式](#设计模式)
- [JavaWeb](#JavaWeb)
    - [Maven](#Maven)
    - [Srping](#Srping)
    - [SpringMVC](#SpringMVC)
    - [SpringBoot](#SpringBoot)
- [计算机网络](#计算机网络)
- [操作系统](#操作系统)
    - [Linux相关](#linux相关)
- [数据结构与算法](#数据结构与算法)
    - [数据结构](#数据结构)
    - [算法](#算法)
- [数据库](#数据库)
    - [MySQL](#mysql)
- [缓存](#缓存)
    - [Redis](#Redis)
- [消息队列](#消息队列)
    - [Kafka](#Kafka)
- [面试指南](#面试指南)
    - [校招指南](#校招指南)
    - [面经](#面经)
- [工具](#工具)
    - [Git](#git)
- [资料](#资料)
    - [书单](#书单)
- [待办](#待办)
- [说明](#说明)
- [微信公众号](#微信公众号)

## Java

### 基础

* [面向对象基础](docs/java/basic/1、面向对象基础.md)
* [Java基本数据类型](docs/java/basic/2、Java基本数据类型.md)
* [string和包装类](docs/java/basic/3、string和包装类.md)
* [final关键字特性](docs/java/basic/4、final关键字特性.md)
* [Java类和包](docs/java/basic/5、Java类和包.md)
* [抽象类和接口](docs/java/basic/6、抽象类和接口.md)
* [代码块和代码执行顺序](docs/java/basic/7、代码块和代码执行顺序.md)
* [Java自动拆箱装箱里隐藏的秘密](docs/java/basic/8、Java自动拆箱装箱里隐藏的秘密.md)
* [Java中的Class类和Object类](docs/java/basic/9、Java中的Class类和Object类.md)
* [Java异常](docs/java/basic/10、Java异常.md)
* [解读Java中的回调](docs/java/basic/11、解读Java中的回调.md)
* [反射](docs/java/basic/12、反射.md)
* [泛型](docs/java/basic/13、泛型.md)
* [枚举类](docs/java/basic/14、枚举类.md)
* [Java注解和最佳实践](docs/java/basic/15、Java注解和最佳实践.md)
* [JavaIO流](docs/java/basic/16、JavaIO流.md)
* [多线程](docs/java/basic/17、多线程.md)
* [深入理解内部类](docs/java/basic/18、深入理解内部类.md)
* [javac和javap](docs/java/basic/19、Java集合框架梳理.md)
* [Java8新特性终极指南](docs/java/basic/20、javac和javap.md)
* [Java类和包](docs/java/basic/21、Java8新特性终极指南.md)
* [序列化和反序列化](docs/java/basic/22、序列化和反序列化.md)
* [继承、封装、多态的实现原理](docs/java/basic/23、继承、封装、多态的实现原理.md)

### 容器
* [Java集合类总结](docs/java/collection/Java集合类总结.md)
* [Java集合详解1：一文读懂ArrayList,Vector与Stack使用方法和实现原理](docs/java/collection/Java集合详解1：一文读懂ArrayList,Vector与Stack使用方法和实现原理.md)  
* [Java集合详解2：Queue和LinkedList](docs/java/collection/Java集合详解2：Queue和LinkedList.md)
* [Java集合详解3：Iterator，fail-fast机制与比较器](docs/java/collection/Java集合详解3：Iterator，fail-fast机制与比较器.md)
* [Java集合详解5：深入理解LinkedHashMap和LRU缓存](docs/java/collection/Java集合详解4：HashMap和HashTable.md)
* [Java集合详解4：HashMap和HashTable](docs/java/collection/Java集合详解5：深入理解LinkedHashMap和LRU缓存.md)
* [Java集合详解6：TreeMap和红黑树](docs/java/collection/Java集合详解6：TreeMap和红黑树.md)
* [Java集合详解7：HashSet，TreeSet与LinkedHashSet](docs/java/collection/Java集合详解7：HashSet，TreeSet与LinkedHashSet.md)
* [Java集合详解8：Java集合类细节精讲](docs/java/collection/Java集合详解8：Java集合类细节精讲.md)


### 并发
* [Java并发指南1：并发基础与Java多线程](docs/java/currency/Java并发指南1：并发基础与Java多线程.md)
* [Java并发指南2：深入理解Java内存模型JMM](docs/java/currency/Java并发指南2：深入理解Java内存模型JMM.md)
* [Java并发指南3：并发三大问题与volatile关键字，CAS操作](docs/java/currency/Java并发指南3：并发三大问题与volatile关键字，CAS操作.md)
* [Java并发指南4：Java中的锁 Lock和synchronized](docs/java/currency/Java并发指南4：Java中的锁 Lock和synchronized.md)
* [Java并发指南5：JMM中的final关键字解析](docs/java/currency/Java并发指南5：JMM中的final关键字解析.md)
* [Java并发指南6：Java内存模型JMM总结](docs/java/currency/Java并发指南6：Java内存模型JMM总结.md)
* [Java并发指南7：JUC的核心类AQS详解](docs/java/currency/Java并发指南7：JUC的核心类AQS详解.md)
* [Java并发指南8：AQS中的公平锁与非公平锁，Condtion](docs/java/currency/Java并发指南8：AQS中的公平锁与非公平锁，Condtion.md)
* [Java并发指南9：AQS共享模式与并发工具类的实现](docs/java/currency/Java并发指南9：AQS共享模式与并发工具类的实现.md)
* [Java并发指南10：Java 读写锁 ReentrantReadWriteLock 源码分析](docs/java/currency/Java并发指南10：Java 读写锁 ReentrantReadWriteLock 源码分析.md)
* [Java并发指南11：解读 Java 阻塞队列 BlockingQueue](docs/java/currency/Java并发指南11：解读 Java 阻塞队列 BlockingQueue.md)
* [Java并发指南12：深度解读 java 线程池设计思想及源码实现](docs/java/currency/Java并发指南12：深度解读 java 线程池设计思想及源码实现.md)
* [Java并发指南13：Java 中的 HashMap 和 ConcurrentHashMap 全解析](docs/java/currency/Java并发指南13：Java 中的 HashMap 和 ConcurrentHashMap 全解析.md)
* [Java并发指南14：JUC中常用的Unsafe和Locksupport](docs/java/currency/Java并发指南14：JUC中常用的Unsafe和Locksupport.md)
* [Java并发指南15：Fork join并发框架与工作窃取算法剖析](docs/java/currency/Java并发编程指南15：Fork join并发框架与工作窃取算法剖析.md)


### JVM
* [JVM总结](docs/java/jvm/JVM总结.md)


### Java网络编程
* [Java网络与NIO总结](docs/java/network-programing/Java网络与NIO总结.md)

### 设计模式
* [设计模式学习总结](docs/java/design-parttern/设计模式学习总结.md)

## JavaWeb

### Maven

### Spring

### SpringMVC

### SpringBoot

## 计算机网络
* [计算机网络学习总结](docs/network/计算机网络学习总结.md)


## 操作系统
* [操作系统学习总结](docs/operateing-system/操作系统学习总结.md)

### Linux相关
* [Linux内核与基础命令学习总结](docs/operateing-system/Linux内核与基础命令学习总结.md)


## 数据结构与算法

### 数据结构


### 算法
* [剑指offer](docs/algorithms/剑指offer.md)


## 数据库

### MySQL
* [Mysql原理与实践总结](docs/database/Mysql原理与实践总结.md)



## 缓存

### Redis
* [Redis原理与实践总结](docs/cache/Redis原理与实践总结.md)

## 消息队列

### Kafka

## 面试指南

### 校招指南

### 面经

## 工具

## 资料

### 书单

***

## 待办

- [ ] Java集合类
- [ ] Java并发编程
- [ ] Java网络编程
- [ ] JVM
- [ ] 设计面试

## 说明

## 微信公众号

### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

### 个人公众号：程序员黄小斜

作者是 985 硕士，蚂蚁金服 JAVA 工程师，专注于 JAVA 后端技术栈：SpringBoot、MySQL、分布式、中间件、微服务，同时也懂点投资理财，偶尔讲点算法和计算机理论基础，坚持学习和写作，相信终身学习的力量！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。	

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)
