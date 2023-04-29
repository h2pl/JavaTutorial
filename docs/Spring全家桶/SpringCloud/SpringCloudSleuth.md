> Spring Cloud Sleuth 是分布式系统中跟踪服务间调用的工具，它可以直观地展示出一次请求的调用过程，本文将对其用法进行详细介绍。

## [#](https://www.macrozheng.com/cloud/sleuth.html#spring-cloud-sleuth-%E7%AE%80%E4%BB%8B)Spring Cloud Sleuth 简介

随着我们的系统越来越庞大，各个服务间的调用关系也变得越来越复杂。当客户端发起一个请求时，这个请求经过多个服务后，最终返回了结果，经过的每一个服务都有可能发生延迟或错误，从而导致请求失败。这时候我们就需要请求链路跟踪工具来帮助我们，理清请求调用的服务链路，解决问题。

## [#](https://www.macrozheng.com/cloud/sleuth.html#%E7%BB%99%E6%9C%8D%E5%8A%A1%E6%B7%BB%E5%8A%A0%E8%AF%B7%E6%B1%82%E9%93%BE%E8%B7%AF%E8%B7%9F%E8%B8%AA)给服务添加请求链路跟踪

> 我们将通过user-service和ribbon-service之间的服务调用来演示该功能，这里我们调用ribbon-service的接口时，ribbon-service会通过RestTemplate来调用user-service提供的接口。

*   首先给user-service和ribbon-service添加请求链路跟踪功能的支持；

*   在user-service和ribbon-service中添加相关依赖：



```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    spring-cloud-starter-zipkin
</dependency>

```




*   修改application.yml文件，配置收集日志的zipkin-server访问地址：



```
spring:
  zipkin:
    base-url: http://localhost:9411
  sleuth:
    sampler:
      probability: 0.1 #设置Sleuth的抽样收集概率

```



## [#](https://www.macrozheng.com/cloud/sleuth.html#%E6%95%B4%E5%90%88zipkin%E8%8E%B7%E5%8F%96%E5%8F%8A%E5%88%86%E6%9E%90%E6%97%A5%E5%BF%97)整合Zipkin获取及分析日志

> Zipkin是Twitter的一个开源项目，可以用来获取和分析Spring Cloud Sleuth 中产生的请求链路跟踪日志，它提供了Web界面来帮助我们直观地查看请求链路跟踪信息。

*   SpringBoot 2.0以上版本已经不需要自行搭建zipkin-server，我们可以从该地址下载zipkin-server：https://repo1.maven.org/maven2/io/zipkin/java/zipkin-server/2.12.9/zipkin-server-2.12.9-exec.jar

*   下载完成后使用以下命令运行zipkin-server：



```
java -jar zipkin-server-2.12.9-exec.jar

```

1




*   Zipkin页面访问地址：http://localhost:9411

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springcloud_sleuth_01.625f37c3.png)

*   启动eureka-sever，ribbon-service，user-service：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springcloud_sleuth_02.baf7b77c.png)

*   多次调用（Sleuth为抽样收集）ribbon-service的接口[http://localhost:8301/user/1open in new window](http://localhost:8301/user/1) ，调用完后查看Zipkin首页发现已经有请求链路跟踪信息了；

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springcloud_sleuth_03.a71d1cf1.png)

*   点击查看详情可以直观地看到请求调用链路和通过每个服务的耗时：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springcloud_sleuth_04.d13e3d99.png)

## [#](https://www.macrozheng.com/cloud/sleuth.html#%E4%BD%BF%E7%94%A8elasticsearch%E5%AD%98%E5%82%A8%E8%B7%9F%E8%B8%AA%E4%BF%A1%E6%81%AF)使用Elasticsearch存储跟踪信息

> 如果我们把zipkin-server重启一下就会发现刚刚的存储的跟踪信息全部丢失了，可见其是存储在内存中的，有时候我们需要将所有信息存储下来，这里以存储到Elasticsearch为例，来演示下该功能。

### [#](https://www.macrozheng.com/cloud/sleuth.html#%E5%AE%89%E8%A3%85elasticsearch)安装Elasticsearch

*   下载Elasticsearch6.2.2的zip包，并解压到指定目录，下载地址：[https://www.elastic.co/cn/downloads/past-releases/elasticsearch-6-2-2open in new window](https://www.elastic.co/cn/downloads/past-releases/elasticsearch-6-2-2)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/arch_screen_25.48daf958.png)

*   运行bin目录下的elasticsearch.bat启动Elasticsearch

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/arch_screen_27.ba3cb8e0.png)

### [#](https://www.macrozheng.com/cloud/sleuth.html#%E4%BF%AE%E6%94%B9%E5%90%AF%E5%8A%A8%E5%8F%82%E6%95%B0%E5%B0%86%E4%BF%A1%E6%81%AF%E5%AD%98%E5%82%A8%E5%88%B0elasticsearch)修改启动参数将信息存储到Elasticsearch

*   使用以下命令运行，就可以把跟踪信息存储到Elasticsearch里面去了，重新启动也不会丢失；



```
# STORAGE_TYPE：表示存储类型 ES_HOSTS：表示ES的访问地址
java -jar zipkin-server-2.12.9-exec.jar --STORAGE_TYPE=elasticsearch --ES_HOSTS=localhost:9200 

```


*   之后需要重新启动user-service和ribbon-service才能生效，重启后多次调用ribbon-service的接口[http://localhost:8301/user/1open in new window](http://localhost:8301/user/1)；

*   如果安装了Elasticsearch的可视化工具Kibana的话，可以看到里面已经存储了跟踪信息：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/springcloud_sleuth_05.9929ce6a.png)

### [#](https://www.macrozheng.com/cloud/sleuth.html#%E6%9B%B4%E5%A4%9A%E5%90%AF%E5%8A%A8%E5%8F%82%E6%95%B0%E5%8F%82%E8%80%83)更多启动参数参考

https://github.com/openzipkin/zipkin/tree/master/zipkin-server#elasticsearch-storage

## [#](https://www.macrozheng.com/cloud/sleuth.html#%E4%BD%BF%E7%94%A8%E5%88%B0%E7%9A%84%E6%A8%A1%E5%9D%97)使用到的模块



```
springcloud-learning
├── eureka-server -- eureka注册中心
├── user-service -- 提供User对象CRUD接口的服务
└── ribbon-service -- ribbon服务调用测试服务

```




## [#](https://www.macrozheng.com/cloud/sleuth.html#%E9%A1%B9%E7%9B%AE%E6%BA%90%E7%A0%81%E5%9C%B0%E5%9D%80)项目源码地址

[https://github.com/macrozheng/springcloud-learning](https://github.com/macrozheng/springcloud-learning)