**前言**

上篇文章讲解了 Kafka 的基础概念和架构，了解了基本概念之后，必须得实践一波了，所谓“实践才是检验真理的唯一办法”，后续系列关于 Kafka 的文章都以 kafka_2.11-0.9.0.0 为例；另外为了让大家快速入门，本文只提供单机版的安装实战教程，如果有想尝试集群方案的，后面在出一篇集群安装的教程，废话不多说了，直接开干。

## **安装**

### **1\. 下载**

版本号：kafka_2.11-0.9.0.0

下载地址：[http://kafka.apache.org/downloads](https://link.zhihu.com/?target=http%3A//kafka.apache.org/downloads)

### **2\. 安装**



```
# 安装目录
$ pwd
/Users/my/software/study

# 减压
$ sudo tar -zxvf kafka_2.11-0.9.0.0.tgz

# 重命名
$ sudo mv kafka_2.11-0.9.0.0.tgz kafka-0.9

# 查看目录结构
$ cd kafka-0.9 && ls
LICENSE   NOTICE    bin       config    libs      site-docs

# 目录结构介绍：
# bin: 			存放kafka 客户端和服务端的执行脚本
# config:		存放kafka的一些配置文件
# libs:			存放kafka运行的的jar包
# site-docs:	存放kafka的配置文档说明

# 配置环境变量，方便在任意目录下运行kafka命令
# 博主使用的Mac，所以配置在了 ~/.bash_profile文件中，
# Linux中则配置在 ~/.bashrc 或者  ~/.zshrc文件中
$ vim ~/.bash_profile

export KAFKA_HOME=/Users/haikuan1/software/study/kafka-0.9
export PATH=$PATH:$JAVA_HOME:$KAFKA_HOME/bin

# 使得环境变量生效
$ source ~/.bash_profile

```



### **3.运行**

### **3.1 启动 zookeeper**



```
# 启动zookeeper，因为kafka的元数据需要保存到zookeeper中
$ bin/zookeeper-server-start.sh config/zookeeper.properties

# 若出现如下信息，则证明zookeeper启动成功了
[2020-04-25 16:23:44,493] INFO Server environment:user.dir=/Users/haikuan1/software/study/kafka-0.10 (org.apache.zookeeper.server.ZooKeeperServer)
[2020-04-25 16:23:44,505] INFO tickTime set to 3000 (org.apache.zookeeper.server.ZooKeeperServer)
[2020-04-25 16:23:44,505] INFO minSessionTimeout set to -1 (org.apache.zookeeper.server.ZooKeeperServer)
[2020-04-25 16:23:44,505] INFO maxSessionTimeout set to -1 (org.apache.zookeeper.server.ZooKeeperServer)
[2020-04-25 16:23:44,548] INFO binding to port 0.0.0.0/0.0.0.0:2181 (org.apache.zookeeper.server.NIOServerCnxnFactory)

```



### **3.2 启动 Kafka server**



```
# 以守护进程的方式启动kafka服务端，去掉 -daemon 参数则关闭当前窗口服务端自动退出
$ bin/kafka-server-start.sh -daemon config/server.properties

```



### **3.3 kafka 基础命令使用**



```
# 1\. 创建一个topic
# --replication-factor：指定副本个数
# --partition：指定partition个数
# --topic：指定topic的名字
$ bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 1 --partition 1 --topic mytopic

# 2\. 查看创建成功的topic
$ kafka-topics.sh --list --zookeeper localhost:2181

# 3\. 创建生产者和消费者

# 3.1 启动kafka消费端
# --from-beginning：从头开始消费，该特性也表明kafka消息具有持久性
$ bin/kafka-console-consumer.sh --zookeeper localhost:2181 --topic mytopic --from-beginning

# 3.2 启动kafka生产端
# --broker-list：当前的Broker列表，即提供服务的列表
$ bin/kafka-console-producer.sh --broker-list localhost:9092 --topic mytopic

```



<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-a1e0c6db02c2822b2ad88db1c3b0b8a7_720w.webp)

</figure>

### **4.使用 Java 连接 kafka 进行测试**

### **4.1 创建一个 maven 工程，引入如下 pom 依赖**



```
<dependency>
    <groupId>org.apache.kafka</groupId>
    kafka-clients
    <version>0.9.0.0</version>
</dependency>

<dependency>
    <groupId>org.apache.kafka</groupId>
    kafka_2.11
    <version>0.9.0.0</version>
</dependency>

```



### **4.2 消费者端代码**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-5e9876ca0dc733fe8c2df51d2e42d1ce_720w.webp)

</figure>

### **4.3 生产者端代码**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-d1e6bfdf23c2b42e23f30d4430c587e2_720w.webp)

</figure>

### **4.4 消费者端效果图**

<figure data-size="normal">


![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/v2-1912f5b2b12ac766d746d88a04b9bd28_720w.webp)

</figure>

### **5.总结**

本文介绍了 kafka 单机版安装及简单命令使用，然后使用 Java 实现了生产者和消费者的简单功能，虽然内容可能比较简单，但还是**强烈建议大家手动去实践一下**，从而对 kafka 的架构有一个更深入的理解。