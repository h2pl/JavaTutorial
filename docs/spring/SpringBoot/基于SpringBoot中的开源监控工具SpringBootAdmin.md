Spring Boot Admin(SBA)是一个开源的社区项目，用于管理和监控 Spring Boot 应用程序。应用程序可以通过 http 的方式，或 Spring Cloud 服务发现机制注册到 SBA 中，然后就可以实现对 Spring Boot 项目的可视化管理和查看了。

Spring Boot Admin 可以监控 Spring Boot 单机或集群项目，它提供详细的健康 (Health)信息、内存信息、JVM 系统和环境属性、垃圾回收信息、日志设置和查看、定时任务查看、Spring Boot 缓存查看和管理等功能。接下来我们一起来搭建和使用吧。

它最终的展示效果如下：

[![](https://s5.51cto.com/oss/202201/14/5d142e8c6b544f7b981b3eff8099b3d8.png)](https://s5.51cto.com/oss/202201/14/5d142e8c6b544f7b981b3eff8099b3d8.png)

## 1.搭建SBA监控端

我们需要创建一个 Spring Boot Admin 项目，用来监控和管理我们的 Spring Boot 项目，搭建的方式和创建普通的 Spring Boot 项目类似，具体步骤如下。使用 Idea 创建一个 Spring Boot 项目：

[![](https://s5.51cto.com/oss/202201/14/d97c492785db6ff2ded49175184ceda9.png)](https://s5.51cto.com/oss/202201/14/d97c492785db6ff2ded49175184ceda9.png)

[![](https://s3.51cto.com/oss/202201/14/8bb1f0389b95e174b56ac01ba313ec7b.png)](https://s3.51cto.com/oss/202201/14/8bb1f0389b95e174b56ac01ba313ec7b.png)

这里需要注意，需要添加 Spring Boot Admin(Server)服务端框架的支持，如下图所示：

[![](https://s4.51cto.com/oss/202201/14/122e9f0726fde8ac0c8936c79ef12f5f.png)](https://s4.51cto.com/oss/202201/14/122e9f0726fde8ac0c8936c79ef12f5f.png)

也就是创建的 Spring Boot 项目需要添加以下两个重要的框架支持：



<button data-clipboard-target="#code_id_0">复制</button>

```
<dependency> 
   <groupId>org.springframework.boot</groupId> 
   spring-boot-starter-web 
</dependency> 
<dependency> 
  <groupId>de.codecentric</groupId> 
  spring-boot-admin-starter-server 
</dependency> 

```





<dependency></dependency>

### 1.1 开启SBA服务

创建完项目之后，需要在启动类上开启 SBA 服务：



<button data-clipboard-target="#code_id_1">复制</button>

```
import de.codecentric.boot.admin.server.config.EnableAdminServer; 
import org.springframework.boot.SpringApplication; 
import org.springframework.boot.autoconfigure.SpringBootApplication; 

@EnableAdminServer // 添加此行代码 
@SpringBootApplication  
public class SbaserverApplication { 
    public static void main(String[] args) { 
        SpringApplication.run(SbaserverApplication.class, args); 
    } 
} 

```





### 1.2 配置SBA端口号

在 application.properties 中配置一个项目的端口号就可以直接启动了，我配置的端口号是 9001：



<button data-clipboard-target="#code_id_2">复制</button>

```
server.port=9001 

```





PS：配置端口号的主要目的是为了不和其他 Spring Boot 项目冲突，如果 SBA 是单独部署此步骤可以忽略。

启动项目之后，就可以看到 SBA 的主页了，如下图所示：

[![](https://s5.51cto.com/oss/202201/14/20418ce88230b20b234f9e9c15e07f47.png)](https://s5.51cto.com/oss/202201/14/20418ce88230b20b234f9e9c15e07f47.png)

此时 SBA 中还没有添加任何需要监控的项目，接下来我们再创建一个 Spring Boot 项目，加入到 SBA 中来进行监控和管理吧。

## 2.创建一个普通SpringBoot项目

首先，我们需要创建一个普通的 Spring Boot 项目，具体的创建步骤这里就不演示了。当创建好 Spring Boot 项目之后，需要在这个 Spring Boot 项目中需要添加 SBA 客户端框架的支持，也就是在 pom.xml 中配置如下内容：

<dependency></dependency>



<button data-clipboard-target="#code_id_3">复制</button>

```
<dependency> 
  <groupId>de.codecentric</groupId> 
  spring-boot-admin-starter-client 
</dependency> 

```





然后在 application.properties 文件中配置 SBA 服务器端地址，也就是咱们第一步创建 SBA 项目的地址，配置内容如下：



<button data-clipboard-target="#code_id_4">复制</button>

```
# 当前项目端口号 
server.port=8080 
# Spring Boot Admin 监控服务器端地址 
spring.boot.admin.client.url=http://localhost:9001 

```





其中“spring.boot.admin.client.url”为 SBA 监控地址。

## 3.SpringBootAdmin监控总览

配置完以上信息之后，此时查看 Spring Boot Admin 页面中就有被监控的 Spring Boot 项目了，如下图所示：

[![](https://s5.51cto.com/oss/202201/14/c010770a5cdfe5fad0ad1e8f0c3b07dc.png)](https://s5.51cto.com/oss/202201/14/c010770a5cdfe5fad0ad1e8f0c3b07dc.png)

也可以点击“应用墙”查看 Spring Boot Admin 中所有被监控的 Spring Boot 项目，如下图所示：

[![](https://s4.51cto.com/oss/202201/14/d88e33f87e116000f9717e8c19c43cc4.png)](https://s4.51cto.com/oss/202201/14/d88e33f87e116000f9717e8c19c43cc4.png)

点击应用进入详情页面，如下图所示：

[![](https://s2.51cto.com/oss/202201/14/4c3609840ea5cb45779eafbc2b260324.png)](https://s2.51cto.com/oss/202201/14/4c3609840ea5cb45779eafbc2b260324.png)

[![](https://s4.51cto.com/oss/202201/14/2912ed434d97f8dd49c27ce73252d34c.png)](https://s4.51cto.com/oss/202201/14/2912ed434d97f8dd49c27ce73252d34c.png)

事件日志中包含 Spring Boot 各种状态的展示(UP 为正常、OFFLINE 为异常)和发生的时间，如下图所示：

[![](https://s5.51cto.com/oss/202201/14/5792a62fbcafe6978bfe3bd26cf1e3ab.png)](https://s5.51cto.com/oss/202201/14/5792a62fbcafe6978bfe3bd26cf1e3ab.png)

## 4.SpringBoot异常监控

当我们手动把被监控的 Spring Boot 项目停止之后，在 Spring Boot Admin 中就可以查看到一个应用已经被停掉了，如下图所示：

[![](https://s5.51cto.com/oss/202201/14/47569a3fe09e62b2364c26bdbd7da4bc.png)](https://s5.51cto.com/oss/202201/14/47569a3fe09e62b2364c26bdbd7da4bc.png)

我们也可以通过事件日志查看 Spring Boot 宕机的具体时间，如下图所示：

[![](https://s2.51cto.com/oss/202201/14/b63f631561fa646f85ccf3e1e4321939.png)](https://s2.51cto.com/oss/202201/14/b63f631561fa646f85ccf3e1e4321939.png)

## 5.配置查看更多监控项

通过上面的内容我们可以看出，监控的 Spring Boot 选项还是比较少的，怎么才能查看更多的监控项呢?要解决这个问题，我们需要在被监控的 Spring Boot 项目中添加 spring-boot-starter-actuator 框架的支持，并开启查看所有监控项的配置才行，最终展示效果如下：

[![](https://s4.51cto.com/oss/202201/14/03938ac0bded4487b6720fc4657f9e99.png)](https://s4.51cto.com/oss/202201/14/03938ac0bded4487b6720fc4657f9e99.png)

接下来我们来配置一下这些监控项。

### 5.1 添加actuator框架支持

在被监控的 Spring Boot 项目中添加 actuator 框架支持，也就是在 pom.xml 中添加以下配置：

<dependency></dependency>



<button data-clipboard-target="#code_id_5">复制</button>

```
<dependency> 
    <groupId>org.springframework.boot</groupId> 
    spring-boot-starter-actuator 
</dependency> 

```





手动点击 Maven 导入依赖包(如果开启了自动导入，此步骤可忽略)。

### 5.2 配置开放所有监控项

在被监控的 Spring Boot 项目中添加以下配置：



<button data-clipboard-target="#code_id_6">复制</button>

```
# 开启监控所有项 
management.endpoints.web.exposure.include=* 

```





以上的配置是开放监控所有选项，配置完之后，重启此 Spring Boot 项目，然后再刷新 Spring Boot Admin 更多的监控项就展示出来了，如下图所示：

[![](https://s6.51cto.com/oss/202201/14/2cc24e05bc6185ba1869872db5a864a5.png)](https://s6.51cto.com/oss/202201/14/2cc24e05bc6185ba1869872db5a864a5.png)

### 5.3 监控项目预览

将 Spring Boot 的所有监控项都开启之后，通过 SBA 就可以查看以下内容了：

*   启动时间、累计运行时间;
*   进程和线程数量和占用的 CPU 资源;
*   垃圾回收详情信息，回收次数和花费时间;
*   JVM 线程转储、内存转储详情和对应的文件下载;
*   可以查看和配置 Spring Boot 项目中的日志级别;
*   查看 Spring Boot 项目性能监控;
*   查看 Spring Boot 运行环境信息;
*   查看 Spring Boot 所有类信息;
*   查看 Spring Boot 中的定时任务;
*   查看和管理 Spring Boot 项目中的所有缓存。

以下是几个重要页面的截图，我们一起来看。

### 5.3.1 查看运行环境

[![](https://s5.51cto.com/oss/202201/14/a9db77b1b0e378450086edd1ab438df5.png)](https://s5.51cto.com/oss/202201/14/a9db77b1b0e378450086edd1ab438df5.png)

[![](https://s2.51cto.com/oss/202201/14/17604f4f5eb388a2a6c08f55e1e050ac.png)](https://s2.51cto.com/oss/202201/14/17604f4f5eb388a2a6c08f55e1e050ac.png)

### 5.3.2 查看定时任务

[![](https://s5.51cto.com/oss/202201/14/113d94b9bd488e239967915aededc89c.png)](https://s5.51cto.com/oss/202201/14/113d94b9bd488e239967915aededc89c.png)

### 5.3.3 项目日志级别配置

[![](https://s3.51cto.com/oss/202201/14/8da60846eecbdbd9472ae6dbf17d951f.png)](https://s3.51cto.com/oss/202201/14/8da60846eecbdbd9472ae6dbf17d951f.png)

我们可以通过 Spring Boot Admin 来动态的配置项目中的日志级别。

### 5.3.4 JVM线程和内存查看

[![](https://s2.51cto.com/oss/202201/14/d3f98228a8b19675475c863457821034.png)](https://s2.51cto.com/oss/202201/14/d3f98228a8b19675475c863457821034.png)

### 5.3.5 查看SpringBoot所有缓存

[![](https://s5.51cto.com/oss/202201/14/ae811102080c26b11135be50cd889710.png)](https://s5.51cto.com/oss/202201/14/ae811102080c26b11135be50cd889710.png)

当然我们还可以对这些缓存进行删除操作。

## 6.查看项目实时日志

想要查看监控项目中的日志信息，有一个前提条件，前提条件是你被监控的 Spring Boot 项目，必须配置了日志的保存路径或者日志保存文件名，只有配置这两项中的任意一项，你的 Spring Boot 项目才会将日志保存到磁盘上，这样才能通过 SBA 查看到，我配置的是日志路径，在 Spring Boot 的 application.properties 配置文件中添加以下配置：



<button data-clipboard-target="#code_id_7">复制</button>

```
# 设置日志保存路径 
logging.file.path=C:\\work\\log 

```





设置完成之后，重启你的 Spring Boot 项目，然后刷新 SBA 页面，最终展示效果如下：

[![](https://s6.51cto.com/oss/202201/14/3f03c6402cc8a2532ed45ab43be156ac.png)](https://s6.51cto.com/oss/202201/14/3f03c6402cc8a2532ed45ab43be156ac.png)

此时我们就可以查看实时的日志信息了，当然你也可以随时下载日志，如果需要的话。

## 总结

Spring Boot Admin(SBA)是一个社区开源项目，用于管理和监视 Spring Boot 应用程序，它提供详细的健康 (Health)信息、内存信息、JVM 系统和环境属性、垃圾回收信息、日志设置和查看、定时任务查看、Spring Boot 缓存查看和管理等功能。

我们需要创建一个 SBA 服务器端用来监控一个或多个 Spring Boot 项目，被监控的 Spring Boot 项目要添加 SBA Client 框架的支持，且添加 actuator 框架和相应的配置，就可以实现对 Spring Boot 项目的完美监控了。