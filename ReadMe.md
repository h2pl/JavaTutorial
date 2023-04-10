本仓库为【Java工程师技术指南】力求打造最完整最实用的Java工程师学习指南！

这些文章和总结都是我近几年学习Java总结和整理出来的，非常实用，对于学习Java后端的朋友来说应该是最全面最完整的技术仓库。
我靠着这些内容进行复习，拿到了BAT等大厂的offer，这个仓库也已经帮助了很多的Java学习者，如果对你有用，希望能给个star支持我，谢谢！

为了更好地讲清楚每个知识模块，我们也参考了很多网上的优质博文，力求不漏掉每一个知识点，所有参考博文都将声明转载来源，如有侵权，请联系我。

点击关注[微信公众号](#微信公众号)及时获取笔主最新更新文章，并可免费领取Java工程师必备学习资源

<p align="center">
<a href="https://github.com/h2pl/Java-Tutorial" target="_blank">
    <img src="https://imgconvert.csdnimg.cn/aHR0cHM6Ly9ub3RlLnlvdWRhby5jb20veXdzL3B1YmxpYy9yZXNvdXJjZS8wNjk1ODIwMzc5MjhhMGU0OWViY2UyYjVhODVmM2UwZi94bWxub3RlLzRCNjhFODVCMzA4MzQwMjJCNzBGMTBBMTRDOEVENDVDLzg4MDQ3?x-oss-process=image/format,png" width="200" height="160"/>
</a>
</p>


推荐使用 https://how2playlife.com/ 在线阅读，在线阅读内容本仓库同步一致。这种方式阅读的优势在于：左侧边栏有目录，阅读体验更好。


## 目录

- [Java基础](#Java基础)
    - [基础知识](#基础知识)
    - [容器](#容器)
    - [设计模式](#设计模式)
- [JavaWeb](#JavaWeb)
    - [Spring](#Spring)
    - [SpringMVC](#SpringMVC)
    - [SpringBoot](#SpringBoot)
- [Java进阶](#Java进阶)
    - [并发](#并发)
    - [JVM](#JVM)
    - [Java网络编程](#Java网络编程)
- [计算机基础](#计算机基础)
    - [计算机网络](#计算机网络)
    - [操作系统](#操作系统)
        - [Linux相关](#linux相关)
    - [数据结构与算法](#数据结构与算法)
        - [数据结构](#数据结构)
        - [算法](#算法)
- [数据库](#数据库)
    - [MySQL](#MySQL)
- [缓存](#缓存)
    - [Redis](#Redis)
- [消息队列](#消息队列)
    - [Kafka](#Kafka)
- [大后端](#大后端)
- [分布式](#分布式)
    - [理论](#理论)
    - [实战](#实战)
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

## Java基础

### 基础知识

* [1、面向对象基础](docs/java/basic/1、面向对象基础.md)
* [2、Java基本数据类型](docs/java/basic/2、Java基本数据类型.md)
* [3、string和包装类](docs/java/basic/3、string和包装类.md)
* [4、final关键字特性](docs/java/basic/4、final关键字特性.md)
* [5、Java类和包](docs/java/basic/5、Java类和包.md)
* [6、抽象类和接口](docs/java/basic/6、抽象类和接口.md)
* [7、代码块和代码执行顺序](docs/java/basic/7、代码块和代码执行顺序.md)
* [8、Java自动拆箱装箱里隐藏的秘密](docs/java/basic/8、Java自动拆箱装箱里隐藏的秘密.md)
* [9、Java中的Class类和Object类](docs/java/basic/9、Java中的Class类和Object类.md)
* [10、Java异常](docs/java/basic/10、Java异常.md)
* [11、解读Java中的回调](docs/java/basic/11、解读Java中的回调.md)
* [12、反射](docs/java/basic/12、反射.md)
* [13、泛型](docs/java/basic/13、泛型.md)
* [14、枚举类](docs/java/basic/14、枚举类.md)
* [15、Java注解和最佳实践](docs/java/basic/15、Java注解和最佳实践.md)
* [16、JavaIO流](docs/java/basic/16、JavaIO流.md)
* [17、多线程](docs/java/basic/17、多线程.md)
* [18、深入理解内部类](docs/java/basic/18、深入理解内部类.md)
* [20、javac和javap](docs/java/basic/20、javac和javap.md)
* [21、Java8新特性终极指南](docs/java/basic/21、Java8新特性终极指南.md)
* [22、序列化和反序列化](docs/java/basic/22、序列化和反序列化.md)
* [23、继承、封装、多态的实现原理](docs/java/basic/23、继承、封装、多态的实现原理.md)

### 容器

* [Java集合类总结](docs/java/collection/Java集合类总结.md)
* [Java集合详解1：一文读懂ArrayList,Vector与Stack使用方法和实现原理](docs/java/collection/Java集合详解1：一文读懂ArrayList,Vector与Stack使用方法和实现原理.md)  
* [Java集合详解2：Queue和LinkedList](docs/java/collection/Java集合详解2：Queue和LinkedList.md)
* [Java集合详解3：Iterator，fail-fast机制与比较器](docs/java/collection/Java集合详解3：Iterator，fail-fast机制与比较器.md)
* [Java集合详解4：HashMap和HashTable](docs/java/collection/Java集合详解4：HashMap和HashTable.md)
* [Java集合详解5：深入理解LinkedHashMap和LRU缓存](docs/java/collection/Java集合详解5：深入理解LinkedHashMap和LRU缓存.md)
* [Java集合详解6：TreeMap和红黑树](docs/java/collection/Java集合详解6：TreeMap和红黑树.md)
* [Java集合详解7：HashSet，TreeSet与LinkedHashSet](docs/java/collection/Java集合详解7：HashSet，TreeSet与LinkedHashSet.md)
* [Java集合详解8：Java集合类细节精讲](docs/java/collection/Java集合详解8：Java集合类细节精讲.md)

### 设计模式

* [设计模式学习总结](docs/java/design-parttern/设计模式学习总结.md)
* [初探Java设计模式1：创建型模式（工厂，单例等）.md](docs/java/design-parttern/初探Java设计模式1：创建型模式（工厂，单例等）.md)
* [初探Java设计模式2：结构型模式（代理模式，适配器模式等）.md](docs/java/design-parttern/初探Java设计模式2：结构型模式（代理模式，适配器模式等）.md)
* [初探Java设计模式3：行为型模式（策略，观察者等）.md](docs/java/design-parttern/初探Java设计模式3：行为型模式（策略，观察者等）.md)
* [初探Java设计模式4：JDK中的设计模式.md](docs/java/design-parttern/初探Java设计模式4：JDK中的设计模式.md)
* [初探Java设计模式5：Spring涉及到的9种设计模式.md](docs/java/design-parttern/初探Java设计模式5：Spring涉及到的9种设计模式.md)

## JavaWeb

* [走进JavaWeb技术世界1：JavaWeb的由来和基础知识](docs/java-web/走进JavaWeb技术世界1：JavaWeb的由来和基础知识.md)
* [走进JavaWeb技术世界2：JSP与Servlet的曾经与现在](docs/java-web/走进JavaWeb技术世界2：JSP与Servlet的曾经与现在.md)
* [走进JavaWeb技术世界3：JDBC的进化与连接池技术](docs/java-web/走进JavaWeb技术世界3：JDBC的进化与连接池技术.md)
* [走进JavaWeb技术世界4：Servlet工作原理详解](docs/java-web/走进JavaWeb技术世界4：Servlet工作原理详解.md)
* [走进JavaWeb技术世界5：初探Tomcat的HTTP请求过程](docs/java-web/走进JavaWeb技术世界5：初探Tomcat的HTTP请求过程.md)
* [走进JavaWeb技术世界6：Tomcat5总体架构剖析](docs/java-web/走进JavaWeb技术世界6：Tomcat5总体架构剖析.md)
* [走进JavaWeb技术世界7：Tomcat和其他WEB容器的区别](docs/java-web/走进JavaWeb技术世界7：Tomcat和其他WEB容器的区别.md)
* [走进JavaWeb技术世界8：浅析Tomcat9请求处理流程与启动部署过程](docs/java-web/走进JavaWeb技术世界8：浅析Tomcat9请求处理流程与启动部署过程.md)
* [走进JavaWeb技术世界9：Java日志系统的诞生与发展](docs/java-web/走进JavaWeb技术世界9：Java日志系统的诞生与发展.md)
* [走进JavaWeb技术世界10：从JavaBean讲到Spring](docs/java-web/走进JavaWeb技术世界10：从JavaBean讲到Spring.md)
* [走进JavaWeb技术世界11：单元测试框架Junit](docs/java-web/走进JavaWeb技术世界11：单元测试框架Junit.md)
* [走进JavaWeb技术世界12：从手动编译打包到项目构建工具Maven](docs/java-web/走进JavaWeb技术世界12：从手动编译打包到项目构建工具Maven.md)
* [走进JavaWeb技术世界13：Hibernate入门经典与注解式开发](docs/java-web/走进JavaWeb技术世界13：Hibernate入门经典与注解式开发.md)
* [走进JavaWeb技术世界14：Mybatis入门](docs/java-web/走进JavaWeb技术世界14：Mybatis入门.md)
* [走进JavaWeb技术世界15：深入浅出Mybatis基本原理](docs/java-web/走进JavaWeb技术世界15：深入浅出Mybatis基本原理.md)
* [走进JavaWeb技术世界16：极简配置的SpringBoot](docs/java-web/走进JavaWeb技术世界16：极简配置的SpringBoot.md)

### Spring

* [Spring源码剖析1：Spring概述](docs/java-web/Spring/Spring源码剖析1：Spring概述.md)
* [Spring源码剖析2：初探Spring IOC核心流程](docs/java-web/Spring/Spring源码剖析2：初探Spring%20IOC核心流程.md)
* [Spring源码剖析3：Spring IOC容器的加载过程 ](docs/java-web/Spring/Spring源码剖析3：Spring%20IOC容器的加载过程.md)
* [Spring源码剖析4：懒加载的单例Bean获取过程分析](docs/java-web/Spring/Spring源码剖析4：懒加载的单例Bean获取过程分析.md)
* [Spring源码剖析5：JDK和cglib动态代理原理详解 ](docs/java-web/Spring/Spring源码剖析5：JDK和cglib动态代理原理详解.md)
* [Spring源码剖析6：Spring AOP概述](docs/java-web/Spring/Spring源码剖析6：Spring%20AOP概述.md)
* [Spring源码剖析7：AOP实现原理详解 ](docs/java-web/Spring/Spring源码剖析7：AOP实现原理详解.md)
* [Spring源码剖析8：Spring事务概述](docs/java-web/Spring/Spring源码剖析8：Spring事务概述.md)
* [Spring源码剖析9：Spring事务源码剖析](docs/java-web/Spring/Spring源码剖析9：Spring事务源码剖析.md)

### SpringMVC

* [SpringMVC源码分析1：SpringMVC概述](docs/java-web/SpringMVC/SpringMVC源码分析1：SpringMVC概述.md)
* [SpringMVC源码分析2：SpringMVC设计理念与DispatcherServlet](docs/java-web/SpringMVC/SpringMVC源码分析2：SpringMVC设计理念与DispatcherServlet.md)
* [SpringMVC源码分析3：DispatcherServlet的初始化与请求转发 ](docs/java-web/SpringMVC/SpringMVC源码分析3：DispatcherServlet的初始化与请求转发.md)
* [SpringMVC源码分析4：DispatcherServlet如何找到正确的Controller ](docs/java-web/SpringMVC/SpringMVC源码分析4：DispatcherServlet如何找到正确的Controller.md)
* [SpringMVC源码剖析5：消息转换器HttpMessageConverter与@ResponseBody注解](docs/java-web/SpringMVC/SpringMVC源码剖析5：消息转换器HttpMessageConverter与@ResponseBody注解.md)
* [SpringMVC源码分析6：SpringMVC的视图解析原理 ](docs/java-web/SpringMVC/SpringMVC源码分析6：SpringMVC的视图解析原理.md)

### SpringBoot

todo

### SpringCloud

todo

## Java进阶

### 并发

* [Java并发指南1：并发基础与Java多线程](docs/java/concurrency/Java并发指南1：并发基础与Java多线程.md)
* [Java并发指南2：深入理解Java内存模型JMM](docs/java/concurrency/Java并发指南2：深入理解Java内存模型JMM.md)
* [Java并发指南3：并发三大问题与volatile关键字，CAS操作](docs/java/concurrency/Java并发指南3：并发三大问题与volatile关键字，CAS操作.md)
* [Java并发指南4：Java中的锁Lock和synchronized](docs/java/concurrency/Java并发指南4：Java中的锁Lock和synchronized.md)
* [Java并发指南5：JMM中的final关键字解析](docs/java/concurrency/Java并发指南5：JMM中的final关键字解析.md)
* [Java并发指南6：Java内存模型JMM总结](docs/java/concurrency/Java并发指南6：Java内存模型JMM总结.md)
* [Java并发指南7：JUC的核心类AQS详解](docs/java/concurrency/Java并发指南7：JUC的核心类AQS详解.md)
* [Java并发指南8：AQS中的公平锁与非公平锁，Condtion](docs/java/concurrency/Java并发指南8：AQS中的公平锁与非公平锁，Condtion.md)
* [Java并发指南9：AQS共享模式与并发工具类的实现](docs/java/concurrency/Java并发指南9：AQS共享模式与并发工具类的实现.md)
* [Java并发指南10：Java读写锁ReentrantReadWriteLock源码分析](docs/java/concurrency/Java并发指南10：Java读写锁ReentrantReadWriteLock源码分析.md)
* [Java并发指南11：解读Java阻塞队列BlockingQueue](docs/java/concurrency/Java并发指南11：解读Java阻塞队列BlockingQueue.md)
* [Java并发指南12：深度解读java线程池设计思想及源码实现](docs/java/concurrency/Java并发指南12：深度解读Java线程池设计思想及源码实现.md)
* [Java并发指南13：Java中的HashMap和ConcurrentHashMap全解析](docs/java/concurrency/Java并发指南13：Java中的HashMap和ConcurrentHashMap全解析.md)
* [Java并发指南14：JUC中常用的Unsafe和Locksupport](docs/java/concurrency/Java并发指南14：JUC中常用的Unsafe和Locksupport.md)
* [Java并发指南15：ForkJoin并发框架与工作窃取算法剖析](docs/java/concurrency/Java并发指南15：ForkJoin并发框架与工作窃取算法剖析.md)
* [Java并发编程学习总结](docs/java/concurrency/Java并发编程学习总结.md)

### JVM

* [JVM总结](docs/java/jvm/JVM总结.md)
* [深入理解JVM虚拟机1：JVM内存的结构与消失的永久代](docs/java/jvm/深入理解JVM虚拟机1：JVM内存的结构与消失的永久代.md)
* [深入理解JVM虚拟机2：JVM垃圾回收基本原理和算法](docs/java/jvm/深入理解JVM虚拟机2：JVM垃圾回收基本原理和算法.md)
* [深入理解JVM虚拟机3：垃圾回收器详解](docs/java/jvm/深入理解JVM虚拟机3：垃圾回收器详解.md)
* [深入理解JVM虚拟机4：Javaclass介绍与解析实践](docs/java/jvm/深入理解JVM虚拟机4：Java字节码介绍与解析实践.md)
* [深入理解JVM虚拟机5：虚拟机字节码执行引擎](docs/java/jvm/深入理解JVM虚拟机5：虚拟机字节码执行引擎.md)
* [深入理解JVM虚拟机6：深入理解JVM类加载机制](docs/java/jvm/深入理解JVM虚拟机6：深入理解JVM类加载机制.md)
* [深入理解JVM虚拟机7：JNDI，OSGI，Tomcat类加载器实现](docs/java/jvm/深入理解JVM虚拟机7：JNDI，OSGI，Tomcat类加载器实现.md)
* [深入了解JVM虚拟机8：Java的编译期优化与运行期优化](docs/java/jvm/深入理解JVM虚拟机8：Java的编译期优化与运行期优化.md)
* [深入理解JVM虚拟机9：JVM监控工具与诊断实践](docs/java/jvm/深入理解JVM虚拟机9：JVM监控工具与诊断实践.md)
* [深入理解JVM虚拟机10：JVM常用参数以及调优实践](docs/java/jvm/深入理解JVM虚拟机10：JVM常用参数以及调优实践.md)
* [深入理解JVM虚拟机11：Java内存异常原理与实践](docs/java/jvm/深入理解JVM虚拟机11：Java内存异常原理与实践.md)
* [深入理解JVM虚拟机12：JVM性能管理神器VisualVM介绍与实战](docs/java/jvm/深入理解JVM虚拟机12：JVM性能管理神器VisualVM介绍与实战.md)
* [深入理解JVM虚拟机13：再谈四种引用及GC实践](docs/java/jvm/深入理解JVM虚拟机13：再谈四种引用及GC实践.md)
* [深入理解JVM虚拟机14：GC调优思路与常用工具](docs/java/jvm/深入理解JVM虚拟机14：GC调优思路与常用工具.md)

### Java网络编程

* [Java网络编程和NIO详解1：JAVA 中原生的 socket 通信机制](docs/java/network-programming/Java网络编程与NIO详解1：JAVA中原生的socket通信机制.md)
* [Java网络编程与NIO详解2：JAVA NIO 一步步构建IO多路复用的请求模型](docs/java/network-programming/Java网络编程与NIO详解2：JavaNIO一步步构建IO多路复用的请求模型.md) 
* [Java网络编程和NIO详解3：IO模型与Java网络编程模型](docs/java/network-programming/Java网络编程与NIO详解3：IO模型与Java网络编程模型.md) 
* [Java网络编程与NIO详解4：浅析NIO包中的Buffer、Channel和Selector](docs/java/network-programming/Java网络编程与NIO详解4：浅析NIO包中的Buffer、Channel和Selector.md) 
* [Java网络编程和NIO详解5：Java非阻塞IO和异步IO](docs/java/network-programming/Java网络编程与NIO详解5：Java非阻塞IO和异步IO.md)
* [Java网络编程与NIO详解6：LinuxEpoll实现原理详解](docs/java/network-programming/Java网络编程与NIO详解6：LinuxEpoll实现原理详解.md.md) 
* [Java网络编程与NIO详解7：浅谈Linux中Selector的实现原理](docs/java/network-programming/Java网络编程与NIO详解7：浅谈Linux中Selector的实现原理.md)
* [Java网络编程与NIO详解8：浅析mmap和DirectBuffer](docs/java/network-programming/Java网络编程与NIO详解8：浅析mmap和DirectBuffer.md)
* [Java网络编程与NIO详解9：基于NIO的网络编程框架Netty](docs/java/network-programming/Java网络编程与NIO详解9：基于NIO的网络编程框架Netty.md)
* [Java网络编程与NIO详解10：Java网络编程与NIO详解10](docs/java/network-programming/Java网络编程与NIO详解10：深度解读Tomcat中的NIO模型.md)
* [Java网络编程与NIO详解11：Tomcat中的Connector源码分析（NIO）](docs/java/network-programming/Java网络编程与NIO详解11：Tomcat中的Connector源码分析（NIO）.md)

## 计算机基础

### 计算机网络
todo


### 操作系统
todo

#### Linux相关
todo


### 数据结构与算法
todo

#### 数据结构
todo

#### 算法
todo

## 数据库
todo

### MySQL
* [Mysql原理与实践总结](docs/database/Mysql原理与实践总结.md)
* [重新学习Mysql数据库1：无废话MySQL入门](docs/database/重新学习MySQL数据库1：无废话MySQL入门.md)
* [重新学习Mysql数据库2：『浅入浅出』MySQL和InnoDB](docs/database/重新学习MySQL数据库2：『浅入浅出』MySQL和InnoDB.md)
* [重新学习Mysql数据库3：Mysql存储引擎与数据存储原理](docs/database/重新学习MySQL数据库3：Mysql存储引擎与数据存储原理.md)
* [重新学习Mysql数据库4：Mysql索引实现原理和相关数据结构算法](docs/database/重新学习MySQL数据库4：Mysql索引实现原理和相关数据结构算法.md)
* [重新学习Mysql数据库5：根据MySQL索引原理进行分析与优化](docs/database/重新学习MySQL数据库5：根据MySQL索引原理进行分析与优化.md)
* [重新学习MySQL数据库6：浅谈MySQL的中事务与锁](docs/database/重新学习MySQL数据库6：浅谈MySQL的中事务与锁.md) 
* [重新学习Mysql数据库7：详解MyIsam与InnoDB引擎的锁实现](docs/database/重新学习MySQL数据库7：详解MyIsam与InnoDB引擎的锁实现.md) 
* [重新学习Mysql数据库8：MySQL的事务隔离级别实战](docs/database/重新学习MySQL数据库8：MySQL的事务隔离级别实战.md)
* [重新学习MySQL数据库9：Innodb中的事务隔离级别和锁的关系](docs/database/重新学习MySQL数据库9：Innodb中的事务隔离级别和锁的关系.md) 
* [重新学习MySQL数据库10：MySQL里的那些日志们](docs/database/重新学习MySQL数据库10：MySQL里的那些日志们.md) 
* [重新学习MySQL数据库11：以Java的视角来聊聊SQL注入](docs/database/重新学习MySQL数据库11：以Java的视角来聊聊SQL注入.md) 
* [重新学习MySQL数据库12：从实践sql语句优化开始](docs/database/重新学习MySQL数据库12：从实践sql语句优化开始.md) 
* [重新学习Mysql数据库13：Mysql主从复制，读写分离，分表分库策略与实践](docs/database/重新学习MySQL数据库13：Mysql主从复制，读写分离，分表分库策略与实践.md)


## 缓存

### Redis
* [Redis原理与实践总结](docs/cache/Redis原理与实践总结.md)
* [探索Redis设计与实现开篇：什么是Redis](docs/cache/探索Redis设计与实现开篇：什么是Redis.md)
* [探索Redis设计与实现1：Redis的基础数据结构概览](docs/cache/探索Redis设计与实现1：Redis的基础数据结构概览.md)
* [探索Redis设计与实现2：Redis内部数据结构详解——dict](docs/cache/探索Redis设计与实现2：Redis内部数据结构详解——dict.md)
* [探索Redis设计与实现3：Redis内部数据结构详解——sds](docs/cache/探索Redis设计与实现3：Redis内部数据结构详解——sds.md)
* [探索Redis设计与实现4：Redis内部数据结构详解——ziplist](docs/cache/探索Redis设计与实现4：Redis内部数据结构详解——ziplist.md)
* [探索Redis设计与实现5：Redis内部数据结构详解——quicklist](docs/cache/探索Redis设计与实现5：Redis内部数据结构详解——quicklist.md)
* [探索Redis设计与实现6：Redis内部数据结构详解——skiplist](docs/cache/探索Redis设计与实现6：Redis内部数据结构详解——skiplist.md)
* [探索Redis设计与实现7：Redis内部数据结构详解——intset](docs/cache/探索Redis设计与实现7：Redis内部数据结构详解——intset.md)
* [探索Redis设计与实现8：连接底层与表面的数据结构robj](docs/cache/探索Redis设计与实现8：连接底层与表面的数据结构robj.md)
* [探索Redis设计与实现9：数据库redisDb与键过期删除策略](docs/cache/探索Redis设计与实现9：数据库redisDb与键过期删除策略.md)
* [探索Redis设计与实现10：Redis的事件驱动模型与命令执行过程](docs/cache/探索Redis设计与实现10：Redis的事件驱动模型与命令执行过程.md)
* [探索Redis设计与实现11：使用快照和AOF将Redis数据持久化到硬盘中](docs/cache/探索Redis设计与实现11：使用快照和AOF将Redis数据持久化到硬盘中.md)
* [探索Redis设计与实现12：浅析Redis主从复制](docs/cache/探索Redis设计与实现12：浅析Redis主从复制.md)
* [探索Redis设计与实现13：Redis集群机制及一个Redis架构演进实例](docs/cache/探索Redis设计与实现13：Redis集群机制及一个Redis架构演进实例.md)
* [探索Redis设计与实现14：Redis事务浅析与ACID特性介绍](docs/cache/探索Redis设计与实现14：Redis事务浅析与ACID特性介绍.md)
* [探索Redis设计与实现15：Redis分布式锁进化史 ](docs/cache/探索Redis设计与实现15：Redis分布式锁进化史.md )

## 消息队列

### Kafka

## 大后端
* [后端技术杂谈开篇：云计算，大数据与AI的故事](docs/big-backEnd/后端技术杂谈开篇：云计算，大数据与AI的故事.md)
* [后端技术杂谈1：搜索引擎基础倒排索引](docs/big-backEnd/后端技术杂谈1：搜索引擎基础倒排索引.md)
* [后端技术杂谈2：搜索引擎工作原理](docs/big-backEnd/后端技术杂谈2：搜索引擎工作原理.md)
* [后端技术杂谈3：Lucene基础原理与实践](docs/big-backEnd/后端技术杂谈3：Lucene基础原理与实践.md)
* [后端技术杂谈4：Elasticsearch与solr入门实践](docs/big-backEnd/后端技术杂谈4：Elasticsearch与solr入门实践.md)
* [后端技术杂谈5：云计算的前世今生](docs/big-backEnd/后端技术杂谈5：云计算的前世今生.md)
* [后端技术杂谈6：白话虚拟化技术](docs/big-backEnd/后端技术杂谈6：白话虚拟化技术.md )
* [后端技术杂谈7：OpenStack的基石KVM](docs/big-backEnd/后端技术杂谈7：OpenStack的基石KVM.md)
* [后端技术杂谈8：OpenStack架构设计](docs/big-backEnd/后端技术杂谈8：OpenStack架构设计.md)
* [后端技术杂谈9：先搞懂Docker核心概念吧](docs/big-backEnd/后端技术杂谈9：先搞懂Docker核心概念吧.md)
* [后端技术杂谈10：Docker 核心技术与实现原理](docs/big-backEnd/后端技术杂谈10：Docker%20核心技术与实现原理.md)
* [后端技术杂谈11：十分钟理解Kubernetes核心概念](docs/big-backEnd/后端技术杂谈11：十分钟理解Kubernetes核心概念.md)
* [后端技术杂谈12：捋一捋大数据研发的基本概念](docs/big-backEnd/后端技术杂谈12：捋一捋大数据研发的基本概念.md)

## 分布式
### 理论
* [分布式系统理论基础1：一致性、2PC和3PC ](docs/distributed/basic/分布式系统理论基础1：一致性、2PC和3PC.md)
* [分布式系统理论基础2：CAP ](docs/distributed/basic/分布式系统理论基础2：CAP.md)
* [分布式系统理论基础3：时间、时钟和事件顺序](docs/distributed/basic/分布式系统理论基础3：时间、时钟和事件顺序.md)
* [分布式系统理论基础4：Paxos](docs/distributed/basic/分布式系统理论基础4：Paxos.md)
* [分布式系统理论基础5：选举、多数派和租约](docs/distributed/basic/分布式系统理论基础5：选举、多数派和租约.md)
* [分布式系统理论基础6：Raft、Zab ](docs/distributed/basic/分布式系统理论基础6：Raft、Zab.md)
* [分布式系统理论进阶7：Paxos变种和优化 ](docs/distributed/basic/分布式系统理论进阶7：Paxos变种和优化.md)
* [分布式系统理论基础8：zookeeper分布式协调服务 ](docs/distributed/basic/分布式系统理论基础8：zookeeper分布式协调服务.md)

* [分布式技术实践总结](docs/distributed/分布式理论总结.md)

### 技术
* [搞懂分布式技术1：分布式系统的一些基本概念](docs/distributed/practice/搞懂分布式技术1：分布式系统的一些基本概念.md )
* [搞懂分布式技术2：分布式一致性协议与Paxos，Raft算法](docs/distributed/practice/搞懂分布式技术2：分布式一致性协议与Paxos，Raft算法.md)
* [搞懂分布式技术3：初探分布式协调服务zookeeper](docs/distributed/practice/搞懂分布式技术3：初探分布式协调服务zookeeper.md )
* [搞懂分布式技术4：ZAB协议概述与选主流程详解](docs/distributed/practice/搞懂分布式技术4：ZAB协议概述与选主流程详解.md )
* [搞懂分布式技术5：Zookeeper的配置与集群管理实战](docs/distributed/practice/搞懂分布式技术5：Zookeeper的配置与集群管理实战.md)
* [搞懂分布式技术6：Zookeeper典型应用场景及实践](docs/distributed/practice/搞懂分布式技术6：Zookeeper典型应用场景及实践.md )

[//]: # (* [搞懂分布式技术7：负载均衡概念与主流方案]&#40;docs/distributed/practice/搞懂分布式技术7：负载均衡概念与主流方案.md&#41;)

[//]: # (* [搞懂分布式技术8：负载均衡原理剖析 ]&#40;docs/distributed/practice/搞懂分布式技术8：负载均衡原理剖析.md &#41;)

[//]: # (* [搞懂分布式技术9：Nginx负载均衡原理与实践 ]&#40;docs/distributed/practice/搞懂分布式技术9：Nginx负载均衡原理与实践.md&#41;)
* [搞懂分布式技术10：LVS实现负载均衡的原理与实践 ](docs/distributed/practice/搞懂分布式技术10：LVS实现负载均衡的原理与实践.md )
* [搞懂分布式技术11：分布式session解决方案与一致性hash](docs/distributed/practice/搞懂分布式技术11：分布式session解决方案与一致性hash.md)
* [搞懂分布式技术12：分布式ID生成方案 ](docs/distributed/practice/搞懂分布式技术12：分布式ID生成方案.md )
* [搞懂分布式技术13：缓存的那些事](docs/distributed/practice/搞懂分布式技术13：缓存的那些事.md)
* [搞懂分布式技术14：SpringBoot使用注解集成Redis缓存](docs/distributed/practice/搞懂分布式技术14：SpringBoot使用注解集成Redis缓存.md)
* [搞懂分布式技术15：缓存更新的套路 ](docs/distributed/practice/搞懂分布式技术15：缓存更新的套路.md )
* [搞懂分布式技术16：浅谈分布式锁的几种方案 ](docs/distributed/practice/搞懂分布式技术16：浅谈分布式锁的几种方案.md )
* [搞懂分布式技术17：浅析分布式事务](docs/distributed/practice/搞懂分布式技术17：浅析分布式事务.md )
* [搞懂分布式技术18：分布式事务常用解决方案 ](docs/distributed/practice/搞懂分布式技术18：分布式事务常用解决方案.md )
* [搞懂分布式技术19：使用RocketMQ事务消息解决分布式事务 ](docs/distributed/practice/搞懂分布式技术19：使用RocketMQ事务消息解决分布式事务.md )
* [搞懂分布式技术20：消息队列因何而生](docs/distributed/practice/搞懂分布式技术20：消息队列因何而生.md)
* [搞懂分布式技术21：浅谈分布式消息技术Kafka](docs/distributed/practice/搞懂分布式技术21：浅谈分布式消息技术Kafka.md )

* [分布式理论总结](docs/distributed/分布式技术实践总结.md)
## 面试指南

todo
### 校招指南
todo

### 面经
todo

## 工具
todo

## 资料
todo

### 书单
todo

## 待办
springboot和springcloud

## 微信公众号


### Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】前阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师技术学习资料:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

**Java进阶架构师资料:** 关注公众号后回复 **”架构师“** 即可领取 Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)
