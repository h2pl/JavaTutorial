# 1\. SkyWalking 简介

> Skywalking 是由国内开源爱好者吴晟（原 OneAPM 工程师，目前在华为）开源并提交到 Apache 孵化器的产品，它同时吸收了 Zipkin/Pinpoint/CAT 的设计思路，支持非侵入式埋点。是一款基于分布式跟踪的应用程序性能监控系统。另外社区还发展出了一个叫 OpenTracing 的组织，旨在推进调用链监控的一些规范和标准工作。

*   SkyWalking 是一个开源监控平台，用于从服务和云原生基础设施收集、分析、聚合和可视化数据。
*   SkyWalking 提供了一种简单的方法来维护分布式系统的清晰视图，甚至可以跨云查看。它是一种现代APM，专门为云原生、基于容器的分布式系统设计。
*   SkyWalking 从三个维度对应用进行监视：service（服务）, service instance（实例）, endpoint（端点）。服务和实例就不多说了，端点是服务中的某个路径或者说URI。
*   SkyWalking 允许用户了解服务和端点之间的拓扑关系，查看每个服务/服务实例/端点的度量，并设置警报规则。

## SkyWalking的组成

SkyWalking主要的几个组成模块:

1.  Agent 主要负责从系统中采集各种指标，链路数据，发送给 oap 服务。
2.  oap 服务接收 Agent 发送过来的数据，存储，执行分析，提供查询和报警功能。
3.  Storage 和 UI 负责存储数据以及查看数据。

# 2\. 使用 Docker 快速搭建 SkyWalking 8.0

1.  **在 linux 服务器上选择并建立目录**；

```
mkdir skywalking-docker
复制代码
```

1.  **进入 skywalking-docker 目录，建立一个名为 skywalking.yaml 的脚本文件，内容如下**：

```
version: '3'
services:
  elasticsearch7:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.5.0
    container_name: elasticsearch7
    restart: always
    ports:
      - 9023:9200
    environment:
      - discovery.type=single-node
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - TZ=Asia/Shanghai
    ulimits:
      memlock:
        soft: -1
        hard: -1
    networks:
      - skywalking
    volumes:
      - elasticsearch7:/usr/share/elasticsearch/data
  oap:
    image: apache/skywalking-oap-server:8.0.1-es7
    container_name: oap
    depends_on:
      - elasticsearch7
    links:
      - elasticsearch7
    restart: always
    ports:
      - 9022:11800
      - 9021:12800
    networks:
      - skywalking
    volumes:
      - ./ext-config:/skywalking/ext-config
  ui:
    image: apache/skywalking-ui:8.0.1
    container_name: ui
    depends_on:
      - oap
    links:
      - oap
    restart: always
    ports:
      - 9020:8080
    environment:
      SW_OAP_ADDRESS: oap:12800
    networks:
      - skywalking

networks:
  skywalking:
    driver: bridge

volumes:
  elasticsearch7:
    driver: local
复制代码
```

**注意**：如果我们想覆盖 oap 镜像中的 /skywalking/config 目录下的配置文件，我们可以在 docker 中挂载一个 /skywalking/ext-config 目录，将配置文件丢到此目录中即可。

![image-20230423174422281](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174422281.png)

1.  **执行 skywalking.yaml 脚本启动容器**：

```
docker-compose -f skywalking.yaml up
复制代码
```

1.  **进入 skywalking 的控制台，发现各种仪表盘，开始当然是空的**：

```
http://(安装SkyWalking机器的IP):9020
复制代码
```

![image-20230423174444272](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174444272.png)

# 3\. 在 Spring 项目中引入 SkyWalking 客户端

全局日志追踪 traceId 的使用：

1.  **添加 pom 文件依赖**：

```
        <dependency>
            <groupId>org.apache.skywalking</groupId>
            apm-toolkit-logback-1.x
            <version>8.0.1</version>
        </dependency>
        <dependency>
            <groupId>org.apache.skywalking</groupId>
            apm-toolkit-trace
            <version>8.0.1</version>
        </dependency>
复制代码
```

1.  **在 resources 目录下 添加 logback-spring.xml 文件，内容如下**:

```
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="logger.path" value="/mnt/logs"/>

    <!-- 彩色日志 -->
    <!-- 彩色日志依赖的渲染类 -->
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter"/>
    <conversionRule conversionWord="wex"
                    converterClass="org.springframework.boot.logging.logback.WhitespaceThrowableProxyConverter"/>
    <conversionRule conversionWord="wEx"
                    converterClass="org.springframework.boot.logging.logback.ExtendedWhitespaceThrowableProxyConverter"/>
    <!-- 彩色日志格式 -->
    <property name="CONSOLE_LOG_PATTERN"
              value="${CONSOLE_LOG_PATTERN:-%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}}"/>

    <!-- 输出到控制台 -->
    
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter">
            <level>info</level>
        </filter>
        <encoder>
            <Pattern>${CONSOLE_LOG_PATTERN}</Pattern>
            <charset>UTF-8</charset>
        </encoder>
    

    <!-- ConsoleAppender：把日志输出到控制台 -->
    
        <encoder class="ch.qos.logback.core.encoder.LayoutWrappingEncoder">
            <layout class="org.apache.skywalking.apm.toolkit.log.logback.v1.x.mdc.TraceIdMDCPatternLogbackLayout">
                <Pattern>
                    <![CDATA[%clr(%d{${LOG_DATEFORMAT_PATTERN:-yyyy-MM-dd HH:mm:ss.SSS}}){faint} [%X{tid}] %clr(${LOG_LEVEL_PATTERN:-%5p}) %clr(${PID:- }){magenta} %clr(---){faint} %clr([%15.15t]){faint} %clr(%-40.40logger{39}){cyan} %clr(:){faint} %m%n${LOG_EXCEPTION_CONVERSION_WORD:-%wEx}]]></Pattern>
            </layout>
        </encoder>
    

    <!-- 输出到文件 -->
    <!-- 时间滚动输出 level为 DEBUG 日志 -->
    <!-- 
        <file>${logger.path}/log_debug.log</file>
        &lt;!&ndash;日志文件输出格式&ndash;&gt;
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> &lt;!&ndash; 设置字符集 &ndash;&gt;
        </encoder>
        &lt;!&ndash; 日志记录器的滚动策略，按日期，按大小记录 &ndash;&gt;
        <rollingPolicy >
            &lt;!&ndash; 日志归档 &ndash;&gt;
            <fileNamePattern>${logger.path}/debug/log-debug-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy >
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            &lt;!&ndash;日志文件保留天数&ndash;&gt;
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        &lt;!&ndash; 此日志文件只记录debug级别的 &ndash;&gt;
        <filter >
            <level>debug</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
     -->

    <!-- 时间滚动输出 level为 INFO 日志 -->
    
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${logger.path}/log_info.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 每天日志归档路径以及格式 -->
            <fileNamePattern>${logger.path}/info/log-info-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录info级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>info</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    

    <!-- 时间滚动输出 level为 WARN 日志 -->
    <!-- 
        &lt;!&ndash; 正在记录的日志文件的路径及文件名 &ndash;&gt;
        <file>${logger.path}/log_warn.log</file>
        &lt;!&ndash;日志文件输出格式&ndash;&gt;
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> &lt;!&ndash; 此处设置字符集 &ndash;&gt;
        </encoder>
        &lt;!&ndash; 日志记录器的滚动策略，按日期，按大小记录 &ndash;&gt;
        <rollingPolicy >
            <fileNamePattern>${logger.path}/warn/log-warn-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy >
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            &lt;!&ndash;日志文件保留天数&ndash;&gt;
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        &lt;!&ndash; 此日志文件只记录warn级别的 &ndash;&gt;
        <filter >
            <level>warn</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
     -->

    <!-- 时间滚动输出 level为 ERROR 日志 -->
    
        <!-- 正在记录的日志文件的路径及文件名 -->
        <file>${logger.path}/log_error.log</file>
        <!--日志文件输出格式-->
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
            <charset>UTF-8</charset> <!-- 此处设置字符集 -->
        </encoder>
        <!-- 日志记录器的滚动策略，按日期，按大小记录 -->
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>${logger.path}/error/log-error-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
            <!--日志文件保留天数-->
            <maxHistory>15</maxHistory>
        </rollingPolicy>
        <!-- 此日志文件只记录ERROR级别的 -->
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
    

    <!--
        root节点是必选节点，用来指定最基础的日志输出级别，只有一个level属性
        level:用来设置打印级别，大小写无关：TRACE, DEBUG, INFO, WARN, ERROR, ALL 和 OFF，
        不能设置为INHERITED或者同义词NULL。默认是DEBUG
        可以包含零个或多个元素，标识这个appender将会添加到这个logger。
    -->
    <root level="info">
        
        
        <!---->
        
        <!---->
        
    </root>

</configuration>
复制代码
```

**注意**：其他都是日志常规配置，主要是这部分 `` 的配置。

1.  **进入 skywalking 官网下载 SkyWalking APM，主要是要用到 agent**：

skywalking 官网下载地址：[skywalking.apache.org/downloads/](https://link.juejin.cn?target=http%3A%2F%2Fskywalking.apache.org%2Fdownloads%2F "http://skywalking.apache.org/downloads/")

![image-20230423174506711](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174506711.png)

1.  **解压下载的 apache-skywalking-apm-es7-8.0.1.tar.gz 包，目录结构如图**：

![image-20230423174522153](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174522153.png)

我们只要其中的 agent 目录就行，agent 里的东西大概有这些：

![image-20230423174533023](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174533023.png)

把 agent 目录复制到一个妥善的目录下，一会儿需要配置 JVM 启动参数目录，当然作者直接放到了项目里：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/8deaee160419423e9a71e710d3b2c3dd~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

1.  **在 idea 程序启动命令增加如下 JVM 启动参数**：

```
-javaagent:(agent文件夹所在的目录)\agent\skywalking-agent.jar -Dskywalking.agent.service_name=(服务名)-service -Dskywalking.agent.instance_name=(服务名)-instance -Dskywalking.collector.backend_service=(安装SkyWalking机器的IP):9022
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/96e9d5a3aa5c44c3b0b948929609ae1f~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

因为 skywalking 是非侵入式埋点实现分布式链路跟踪和性能监控，所以一般采用 javaagent 的方式。

> **Javaagent 是什么**（JVM 启动前静态 Instrument）？
>
> Javaagent 是 java 命令的一个参数。参数 javaagent 可以用于指定一个 jar 包，并且对该 java 包有两个要求：
>
> 1.  这个 jar 包的 MANIFEST.MF 文件必须指定 Premain-Class 项。
> 2.  Premain-Class 指定的那个类必须实现 premain() 方法。
>
> premain() 方法，从字面上理解，就是运行在 main() 函数之前的的类。当 Java 虚拟机启动时，在执行 main() 函数之前，jvm 会先运行 -javaagent 所指定 jar 包内 Premain-Class 这个类的 premain() 方法 。

1.  **我们前几篇文章中搭建了完整的项目，在这些项目中全部按照上边的配置一遍，启动看下效果**：

*   网关服务：herring-gateway，zuul 统一网关微服务。
*   认证服务：herring-oauth2，oauth2 认证中心微服务。
*   会员服务：herring-member-service，微服务之一，接收到请求后会到认证中心验证。
*   订单服务：herring-orders-service，微服务之二，接收到请求后会到认证中心验证。
*   商品服务：herring-product-service，微服务之三，接收到请求后会到认证中心验证。

![image-20230423174600279](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174600279.png)

1.  **我们来测试一下，先请求下 token，然后再请求 /api/member/update**：

```
#### 

POST http://localhost:8080/oauth2-service/oauth/token?grant_type=password&username=admin&password=123456&client_id=app-client&client_secret=client-secret-8888&scope=all
Accept: */*
Cache-Control: no-cache
复制代码
```

得到返回结果 token：

```
{
  "access_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiZ2F0ZXdheS1zZXJ2aWNlIl0sInVzZXJfbmFtZSI6ImFkbWluIiwiand0LWV4dCI6IkpXVCDmianlsZXkv6Hmga8iLCJzY29wZSI6WyJhbGwiXSwiZXhwIjoxNjEzOTcwMDk2LCJhdXRob3JpdGllcyI6WyJST0xFX0FETUlOIl0sImp0aSI6IjU4MDY5ODlhLWUyNDQtNGQyMy04YTU5LTBjODRiYzE0Yjk5OSIsImNsaWVudF9pZCI6ImFwcC1jbGllbnQifQ.EP4acam0tkJQ9kSGRGk_mQsfi1y4M_hhiBL0H931v60",
  "token_type": "bearer",
  "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiZ2F0ZXdheS1zZXJ2aWNlIl0sInVzZXJfbmFtZSI6ImFkbWluIiwiand0LWV4dCI6IkpXVCDmianlsZXkv6Hmga8iLCJzY29wZSI6WyJhbGwiXSwiYXRpIjoiNTgwNjk4OWEtZTI0NC00ZDIzLThhNTktMGM4NGJjMTRiOTk5IiwiZXhwIjoxNjE0MDM0ODk2LCJhdXRob3JpdGllcyI6WyJST0xFX0FETUlOIl0sImp0aSI6IjQxZGM1ZDc1LTZmZDgtNDU3My04YmRjLWI4ZTMwNWEzMThmMyIsImNsaWVudF9pZCI6ImFwcC1jbGllbnQifQ.CGmGx_msqJBHxa95bBROY2SAO14RyeRklVPYrRxZ7pQ",
  "expires_in": 7199,
  "scope": "all",
  "jwt-ext": "JWT 扩展信息",
  "jti": "5806989a-e244-4d23-8a59-0c84bc14b999"
}
复制代码
```

请求执行 /api/member/update

```
####

GET http://localhost:8080/member-service/api/member/update
Accept: */*
Cache-Control: no-cache
Authorization: bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiZ2F0ZXdheS1zZXJ2aWNlIl0sInVzZXJfbmFtZSI6ImFkbWluIiwiand0LWV4dCI6IkpXVCDmianlsZXkv6Hmga8iLCJzY29wZSI6WyJhbGwiXSwiZXhwIjoxNjEzOTcwMDk2LCJhdXRob3JpdGllcyI6WyJST0xFX0FETUlOIl0sImp0aSI6IjU4MDY5ODlhLWUyNDQtNGQyMy04YTU5LTBjODRiYzE0Yjk5OSIsImNsaWVudF9pZCI6ImFwcC1jbGllbnQifQ.EP4acam0tkJQ9kSGRGk_mQsfi1y4M_hhiBL0H931v60
复制代码
```

**仪表盘结果展示**:

![image-20230423174639471](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174639471.png)

![image-20230423174721822](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174721822.png)

![image-20230423174742703](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174742703.png)

**拓扑图结果展示**：

![image-20230423174809108](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174809108.png)

![image-20230423174831290](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174831290.png)

**链路追踪结果展示**：

![image-20230423174845526](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230423174845526.png)



作者：白菜说技术
链接：https://juejin.cn/post/6931922457741770760
来源：稀土掘金
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

# 参考文章
https://lijunyi.xyz/docs/SpringCloud/SpringCloud.html#_2-2-x-%E5%88%86%E6%94%AF
https://mp.weixin.qq.com/s/2jeovmj77O9Ux96v3A0NtA
https://juejin.cn/post/6931922457741770760
https://github.com/D2C-Cai/herring
http://c.biancheng.net/springcloud