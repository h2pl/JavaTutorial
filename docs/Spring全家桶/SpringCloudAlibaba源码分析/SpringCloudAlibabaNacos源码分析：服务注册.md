# 一、 Nacos服务注册源码解析



* * *



## 1.1  源码方式打包



* * *



客户端源码中增加打包方式，将源码打入包中







```<plugin>  <groupId>org.apache.maven.plugins</groupId>  maven-source-plugin  <version>3.2.1</version>  <configuration>  true  </configuration>  <executions>  <execution>  <phase>compile</phase>  <goals>  <goal>jar</goal>  </goals>  </execution>  </executions> </plugin> ```







然后打包：







```mvn install -DskipTests ```







## 1.2 入口



* * *



[github.com/alibaba/nac…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Falibaba%2Fnacos%2Ftree%2F1.4.1)

首先我们会把源码下载下来，我们会通过源码的方式进行启动， 你可以通过debug的方式进行运行来判断他的运行过程。

我们从源码的角度来分析一下：服务启动后他会上报到注册中心：

NacosNamingService 就是服务注册和发现相关的类，他就是在这里将当前启动的服务调用注册实例的方法，我们看一下这个方法干什么了？

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/fc004433c7304147905c088dd3227005.png "image.png")

![image-20211221124947544](E:\BaiduNetdiskWorkspace\springcloud alibaba\img\image-20211221124947544.png)

他就是拼接了一些参数发送http请求，到达服务注册中心进行发现，请求的就是这个路径：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/51f5b45b70ef4a7dbe1258e05314fa61.png "image.png")

好这就是对应的路径，我们回到官方文档的指南当中

[nacos.io/zh-cn/docs/…](https://link.juejin.cn?target=https%3A%2F%2Fnacos.io%2Fzh-cn%2Fdocs%2Fopen-api.html)

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d3b30ba715ab4363b3ee8b4e21a2f3a2.png "image.png")

好，按照我们讲到这里就不用再往里面看了，我们可以点进去看一下，

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/9f4bbf69cc1746e79545831252796e55.png "image.png")

他真正的调用给你是在这里。

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b06c94937d374ddb9fc2c224e5932205.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d0d55d3616ad4109837d9aee36e5a946.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/450565c4c14f4f3bb2097ab1ea69aae7.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/527d2af9bb0c4206b090d3410238a576.png "image.png")

在这里进行调用

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/3f3f39de5aaa4ad2b3ac7dccc33aa574.png "image.png")

好哪有同学问你怎么知道断点就达到这里，那我们看一下怎样查看源码的启动的路径，我们看一下我们订单微服务的路径，我们要集成nacos的服务发现功能，我们要引入我们的discovery的包，他是一个starter，前面我们学过springboot我们知道任何starter里面一定有个spring.factories，作为一个入口

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/be47c62f16f94507a1af84d0169ebf53.png "image.png")

这里面动态加载的类很多，NacosServiceRegistryAutoConfiguration 从这名字我们能发现他是一个nacos服务注册的自动配置类，

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/84017587439846a4b21a696f12775b6c.png "image.png")

这里面实现了三个类，我们看一下这个类NacosAUtoServiceRegistration

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/fe7ee87e88944b689835432c833972c2.png "image.png")

自动注册类，我们可以看一下他的集成关系，是一个ApplicationListener  spring启动完成后都会发送一个消息，applicaitonListener就是通过监听这个消息然后进行执行的。所以我们知道下一步我们应该怎么看：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/2a12ba62dae24aa092f97b0cff5dfaaa.png "image.png")

所以查看他的抽象类。

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1f9253dbbbd349ffb6d0e60d0761b3ec.png "image.png")

查看onApplicationEvent方法：这样在服务发送完就会发送这样一个消息，收到这个消息就会调用这个bind方法

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/41a7323a2e2c49d19aaf5768e66af7d8.png "image.png")

这里有个if return 我们就直接跳过，这一定是分支代码，像这样的分支代码我们就不要看，第一 次要看主线，所以我们直接看这里的start方法，如果后面这里没有对应的代码逻辑我们可以进入这个分支来看。 好，像这样start, begin，init，register方法都是很重要的方法，我们一定要进去看

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/3923ffd589e9488eb1143811a50d28c0.png "image.png")

第一个if就不用看，第二个if需要看，因为后面就没有逻辑了你看这个register()，应该就是这个方法，因为你就是查看注册的流程。

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d5b527e0900a4f1b9abcd76db4bfc138.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/d3501fdf6479417180e881c38638bdac.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/7c0ea798fcd7404d8f350047e947add1.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/0ca3ab1a98f248a0aa3854730eba85fb.png "image.png")

这里需要你知道SpringBoot自动装配的基本知识，其次要知道Spring启动发现的 基本知识。

## 1.3 服务注册



* * *



Nacos高并发支撑异步任务与内存队列剖析

刚才是在服务提供者上面讲的内容，现在我们服务注册中心来看一下

请求的是instance实例：这就是一个springmvc的controller，所有我们可以全文搜索 Controller，我们是instance实例吧，所以我们这里InstanceController

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4e02f16787a84704add5aa41f9d7662a.png "image.png")

那之前我们用的是post请求所以我们查看post方法，这里有delete,update...,他这里是什么风格？ restFul

我们发现里面没有对应的DefaultGroup，在服务注册和发现的情况下这个group是不经常用的。你用的话只是自己的规范和方便管理的。在服务注册和发现中源码中都没有用。，命名空间，服务名，然后将我们参数转化为实例。这就我们服务模型中的三层模型。

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/09f059ba31a7444a88a24b03399d502d.png "image.png")

那我们看一下他的注册实例里面做了什么？ 这里面我们注意我们是注册instance，我们就围绕着他，进行分析，是不是就来到addInstance了

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4b69913abe6f40288a08dee89ecb0772.png "image.png")

createEmptyService

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b8a3063b3c924ee195af64d5d8efa647.png "image.png")

1、获取service 初次获取一定为空，我们可以进去分析一下

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/fdf453ebadee486c9def86560bc3b8de.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/7025fb4edcc247b3aca0f1f851d26203.png "image.png")

这里就是注册表，我们前面说过nacos服务领域模型【可以参考图】，这个map就是对应的注册表

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4f1db03d3fcc4f93aaa17cc485fb59c2.png "image.png")

这里面设置服务和初始化

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/1af3a9015b2c4e1ab06da283e4dc5332.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/cb795dd33b114f84bc51cdbdca5af202.png "image.png")

服务初始化：心跳

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/0d0583abf0184ed49bf5fd93c9effafe.png "image.png")

看这里是个scheduleNamingHealth是一个定时任务，我们只需要看一下task任务就可以

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/51ba8120a9954389b9e2f44d49532a14.png "image.png")

task任务我们需要看一下run方法：

在这里我们看是获取所有的实例【可以点进去看一下】

当前时间 -  上次心跳时间 间隔超过15秒 则将实例设置为非健康， 当超过30秒没有收到心跳就直接剔除

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f9131eea860e410685d8c4cd5beea69e.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/586fd2fcb489414590295ce2dfafd91d.png "image.png")

好，我们回来，这里名字起的特别好，createEmtyService，是创建一个空服务，后面我们的实例，是不是还会注册到里面，

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/9518406856b642298d3ee2f16e36bed0.png "image.png")

> 我们可以看一下服务模型，和我们以前说过的一样
>
> 命名空间 ，cluster 集群概念
>
> ![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/0323827c8249417b8ab3fd6c4bd3e61e.png "image.png")
>
> ![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/044d8f58dc5b4c7e83dab05c62397566.png "image.png")
>
> 集群中对应的实例。
>
> ![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/da6adcbf25f44992b08f919988f81b62.png "image.png")

我们看一下addinstance

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/cb9d4cb8fd4d4f0aa75e5d95c4923aae.png "image.png")

构建对应的key:







```String key = KeyBuilder.buildInstanceListKey(namespaceId, serviceName, ephemeral); ```







![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/6ab491f925e3403fa1377c2649d4721d.png "image.png")







```//获取注册实例的IP端口列表 List<Instance> instanceList = addIpAddresses(service, ephemeral, ips); ```







我们进入简单的看一下，我们发现这个add,remove，这里就是新增和移除实例

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/bebd689b54d04e0dab0e64ac552e965e.png "image.png")

我们主要出里这写注册进来的ips，那我们就点击ips，高亮来显示看看，然后他会进行循环instance，这里我们可以看到如果他是移除就从map里面移除出去，如果是新增就在instanceMap中新增一下，最后将期返回。

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4d797c90012b4c22ac6e999bdfad8d63.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/158d309e86084a76be392ac8b67ccb94.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/04b51596154e4d4b933a39f0eb249587.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/86453757688743df89df30752bbbe0e5.png "image.png")

现类，我们可以猜测一下，或者debug进去，当然这个类，我们点击一下，在声明的地方

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/f7a189074e00495da10119f2a5de7bc5.png "image.png")

它指定了名称：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/c54ca72e12e04d80b189d587a535d834.png "image.png")

好，我们全文搜索看一下：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/2b24cf84302c42669bed76d19a11e301.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/fcf6ffab779548a694a68c443a90a214.png "image.png")

我们知道前面说过ephemeral是true所以选择第一个： 有疑问：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4adb989d26cf46f98c943a114e047679.png "image.png")

我们应该调用EphemeralConsistencyService对应的put方法。但是EphemeralConsistencyService只是一个接口，我们应该调用对应的实现实例。

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/87869280d709441fbb8ceec818d67a97.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/3764ecf8f5904315af85482f3c3ee71d.png "image.png")

我们看一下他的onput方法。

现在放到队列中：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/02a34386b7444f0e8a6e35e57ceedf84.png "image.png")

这里面就是把核心的请求放到blockquene里面，也就是一个阻塞队列中

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/81b1c141753c492493eb095ead9e34da.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/252863f614a24c2da64713af52669ce5.png "image.png")

整个注册的过程就这么简单，一个请求过来，最终将服务注册的请求，放到我们的阻塞队列当中，放到则色队列之后，整个阻塞队列就返回了。那放到阻塞队列之后，哪里有访问这个阻塞队列。

大家注意这个Notifier是一个线程，老师交大家一个技巧：如果遇到一个线程就需要看他的run方法因为run方法才是他 真正执行代码的地方。

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/659d96fac1d445859a6ea6086ab8277c.png "image.png")

在这里进行死循环进行数据处理，不断的处理客户端注册的信息，丢进来就实现后面的异步注册信息，

这个线程会一直在转，一直运行，如果他挂了那说明整个服务就挂掉了，好，你看着里面的异常也吃掉了，所以一直会运行，如果没有数据他会阻塞，阻塞会让出cpu

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/487a2e6d15f642549fe0d145447cc5cf.png "image.png")

注册表结构的分析：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/3eab037db7e2480ba4a9b439b18567da.png "image.png")

首先当他是个change的时候，我们就进入onChange，他是个服务所以我们进入他的service里面去看：

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/4adf69d76188430d860f2003b6c508d6.png "image.png")

这里面就不用看了，先看权重，权重大于多少的时候设置最大值，权重小于多少的时候设置一个最小值。然后就是核心的方法updateIP

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/887332bfdcb64c2f9ad871ee55961b59.png "image.png")

那这个updateIPs是做什么呢？ 做的就是我遍历我所有要注册的实例，然后就放到我们的clusterMap里面

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/87df4508357847f1ab49a785ab464e5b.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b31cf0150533469ebd4fbc462c6a0952.png "image.png")

那到这里大家可能就有疑问了，那什么时候启动这个线程，来实时监听我们消息的阻塞队列呢？ 教大家如何找一下这个方法，我们现在看着个类叫Notifier，因为他本身就是一个线程，他会丢到一个线程池中进行运行，我们看一下他究竟是在哪里实例化的，

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/16223a5f9aab4fdaa004d0698e6cb0a6.png "image.png")

我们看到这个方法：这个注解就是当你的spring的一个类进行初始化之后进行调用的，那我们看一下这个init方法到底是做了什么

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/b9d2a25274834d73ab8f939006e9b075.png "image.png")

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/64f25704efac43a7ae6ee5afda46cdf7.png "image.png")

是一个Scheduled线程池：

![image-20211222222903941](E:\BaiduNetdiskWorkspace\springcloud alibaba\img\image-20211222222903941.png)

也就是在对象初始化的时候就进行启动一个线程池，去运行notifier对应的方法。这个run方法就是这样run的。启动后就会实时监听异步队列。这样写的好处，就是将写和处理完全隔离了。通过监听高性能的内存队列，来处理这个事情，他这样的好处，1、提高性能

![image.png](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/e27c8e72b30845c3ab9346b89932fb42.png "image.png")

# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud
https://github.com/macrozheng/springcloud-learning