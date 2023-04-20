## 4\. 日志





Spring Boot在所有内部日志中使用 [Commons Logging](https://commons.apache.org/logging) ，但对底层日志的实现保持开放。 为 [Java Util Logging](https://docs.oracle.com/javase/17/docs/api/java/util/logging/package-summary.html) 、 [Log4j2](https://logging.apache.org/log4j/2.x/) 、 [Logback](https://logback.qos.ch/) 提供了默认配置。 在每一种情况下，记录器（logger）都被预设为使用控制台输出，也可以选择输出到文件。





默认情况下，如果你使用 “Starter”，则默认使用Logback。 适当的Logback路由也包括在内，以确保使用Java Util Logging、Commons Logging、Log4J或SLF4J的依赖库都能正确工作。





|  | 有很多适用于Java的日志框架。 如果上面的列表看起来很混乱，请不要担心。 一般来说，你不需要改变你的日志依赖，Spring Boot的默认值就很好用。 |
| --- | --- |





|  | 当你把你的应用程序部署到一个servlet容器或应用服务器时，用Java Util Logging API执行的日志不会被传送到你的应用程序的日志中。 这可以防止由容器或其他已经部署到它的应用程序执行的日志出现在你的应用程序的日志中。 |
| --- | --- |





### [](https://springdoc.cn/spring-boot/features.html#features.logging.log-format)4.1\. 日志格式



Spring Boot的默认的日志输出格式类似于下面的例子。







 2023-03-03T21:18:18.827+08:00  INFO 19388 --- [           main] o.s.b.d.f.s.MyApplication                : Starting MyApplication using Java 17 with PID 19388 (/opt/apps/myapp.jar started by myuser in /opt/apps/)
2023-03-03T21:18:18.834+08:00  INFO 19388 --- [           main] o.s.b.d.f.s.MyApplication                : No active profile set, falling back to 1 default profile: "default"
2023-03-03T21:18:20.439+08:00  INFO 19388 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2023-03-03T21:18:20.461+08:00  INFO 19388 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2023-03-03T21:18:20.461+08:00  INFO 19388 --- [           main] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.1.5]
2023-03-03T21:18:20.600+08:00  INFO 19388 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2023-03-03T21:18:20.602+08:00  INFO 19388 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 1685 ms
2023-03-03T21:18:21.078+08:00  INFO 19388 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2023-03-03T21:18:21.093+08:00  INFO 19388 --- [           main] o.s.b.d.f.s.MyApplication                : Started MyApplication in 2.998 seconds (process running for 3.601) 







输出的项目如下。





*   Date和时Time：精确到毫秒，易于排序。

*   日志级别: `ERROR`, `WARN`, `INFO`, `DEBUG`, 或 `TRACE`.

*   进程ID。

*   一个 `---` 分隔符，以区分实际日志信息的开始。

*   线程名称：包含在方括号中（对于控制台输出可能会被截断）。

*   记录器名称：这通常是源类的名称（通常是缩写）。

*   日志消息。





|  | Logback没有 `FATAL` 级别。 它被映射到 `ERROR`。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/features.html#features.logging.console-output)4.2\. 控制台输出



默认情况下，日志会输出 `ERROR`、`WARN` 和 `INFO` 级别的消息到控制台。 你也可以通过用 `--debug` 标志启动你的应用程序，来启用 `debug` 模式。







```
$ java -jar myapp.jar --debug
```







|  | 你也可以在你的 `application.properties` 中指定 `debug=true`。 |
| --- | --- |





当debug模式被启用时，一些核心记录器（嵌入式容器、Hibernate和Spring Boot）被配置为输出更多信息。 启用debug模式并不意味着将你的应用程序配置为以 `DEBUG` 级别记录所有信息。





另外，你可以通过在启动应用程序时使用 `--trace` 标志（或在 `application.properties` 中使用 `trace=true` ）来启用 “trace” 模式。 这样做可以对一些核心记录器（嵌入式容器、Hibernate schema生成和整个Spring组合）进行跟踪记录。





#### [](https://springdoc.cn/spring-boot/features.html#features.logging.console-output.color-coded)4.2.1\. 彩色编码的输出



如果你的终端支持ANSI，就会使用彩色输出来帮助阅读。 你可以将 `spring.output.ansi.enabled` 设置为 [支持的值](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/api/org/springframework/boot/ansi/AnsiOutput.Enabled.html)，以覆盖自动检测。





颜色编码是通过使用 `%clr` 转换关键字来配置的。 在其最简单的形式中，转换器根据日志级别对输出进行着色，如下面的例子中所示。







```
%clr(%5p)
```







下表描述了日志级别与颜色的映射关系。



<colgroup><col><col></colgroup>
| 日志级别 | 颜色 |
| --- | --- |
| `FATAL` | 红 |
| `ERROR` | 红 |
| `WARN` | 黄 |
| `INFO` | 绿 |
| `DEBUG` | 绿 |
| `TRACE` | 绿 |



另外，你也可以通过为转换提供一个选项来指定应该使用的颜色或样式。 例如，要使文本为黄色，请使用以下设置。







```
%clr(%d{yyyy-MM-dd'T'HH:mm:ss.SSSXXX}){yellow}
```







支持以下颜色和样式。





*   `blue`

*   `cyan`

*   `faint`

*   `green`

*   `magenta`

*   `red`

*   `yellow`









### [](https://springdoc.cn/spring-boot/features.html#features.logging.file-output)4.3\. 输出到文件



默认情况下，Spring Boot只向控制台记录日志，不写日志文件。 如果你想在控制台输出之外写日志文件，你需要设置 `logging.file.name` 或 `logging.file.path` 属性（例如，在你的 `application.properties` 中）。





下表显示了 `logging.*` 属性如何被一起使用。



<caption>Table 5\. Logging properties</caption><colgroup><col><col><col><col></colgroup>
| `logging.file.name` | `logging.file.path` | Example | Description |
| --- | --- | --- | --- |
| _(none)_ | _(none)_ |  | 只在控制台进行记录。 |
| 指定文件 | _(none)_ | `my.log` | 写入指定的日志文件。 名称可以是一个确切的位置，也可以是与当前目录的相对位置。 |
| _(none)_ | 指定目录 | `/var/log` | 将 `spring.log` 写到指定目录。 名称可以是一个确切的位置，也可以是与当前目录的相对位置。 |



日志文件在达到10MB时就会轮换，与控制台输出一样，默认情况下会记录 `ERROR` 、`WARN` 级和 `INFO` 级别的信息。





|  | 日志属性独立于实际的日志基础设施。 因此，特定的配置属性（如Logback的 `logback.configurationFile` ）不由spring Boot管理。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/features.html#features.logging.file-rotation)4.4\. 文件轮换（滚动日志）



如果你使用Logback，可以使用你的 `application.properties` 或 `application.yaml` 文件来微调日志轮换设置。 对于所有其他的日志系统，你将需要自己直接配置轮换设置（例如，如果你使用Log4J2，那么你可以添加一个 `log4j2.xml` 或 `log4j2-spring.xml` 文件）。





支持以下轮换策略属性。



<colgroup><col><col></colgroup>
| 属性 | 说明 |
| --- | --- |
| `logging.logback.rollingpolicy.file-name-pattern` | 用于创建日志归档的文件名模式。 |
| `logging.logback.rollingpolicy.clean-history-on-start` | 应用程序启动时，是否行日志归档清理。 |
| `logging.logback.rollingpolicy.max-file-size` | 日志文件归档前的最大尺寸（文件最大体积，达到这个体积就会归档）。 |
| `logging.logback.rollingpolicy.total-size-cap` | 日志档案在被删除前的最大尺寸（归档文件最大占用大小，超过这个大小后会被删除）。 |
| `logging.logback.rollingpolicy.max-history` | 要保留的归档日志文件的最大数量（默认为7）。 |





### [](https://springdoc.cn/spring-boot/features.html#features.logging.log-levels)4.5\. 日志级别



所有支持的日志系统都可以通过使用 `logging.level.<logger-name>=<level>` 在Spring的 `Environment`（例如，在 `application.properties`）中设置日志级别，其中 `level` 是 `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`, `FATAL`, 或 `OFF` 之一。 `root` 记录器（logger）的级别可以通过 `logging.level.root` 来配置。





下面的例子显示了 `application.properties` 中潜在的日志设置。







Properties

Yaml





```
logging.level.root=warn
logging.level.org.springframework.web=debug
logging.level.org.hibernate=error
```







也可以使用环境变量来设置日志级别。 例如，`LOGGING_LEVEL_ORG_SPRINGFRAMEWORK_WEB=DEBUG` 将设置 `org.springframework.web` 为 `DEBUG` 。





|  | 上述方法只适用于包级日志。 由于宽松绑定总是将环境变量转换为小写字母，所以不可能用这种方式为单个类配置日志。 如果你需要为一个类配置日志，你可以使用[`SPRING_APPLICATION_JSON`](https://springdoc.cn/spring-boot/features.html#features.external-config.application-json)变量。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/features.html#features.logging.log-groups)4.6\. 日志组（Log Groups）



能够将相关的日志记录器分组，以便同时对它们进行配置，这通常很有用。 例如，你可能经常改变 _所有_ 与Tomcat相关的记录器的记录级别，但你不容易记住最高级别的包。





为了帮助解决这个问题，Spring Boot允许你在Spring `Environment` 中定义日志组。 例如，你可以通过在 `application.properties` 中加入 “tomcat” group 来定义它。







Properties

Yaml





```
logging.group.tomcat=org.apache.catalina,org.apache.coyote,org.apache.tomcat

```







一旦定义好后，就可以用一行代码来改变组中所有logger的级别。







Properties

Yaml





```
logging.level.tomcat=trace

```







Spring Boot包括以下预定义的日志组，可以开箱即用。



<colgroup><col><col></colgroup>
| 组名 | 组中的logger |
| --- | --- |
| web | `org.springframework.core.codec`, `org.springframework.http`, `org.springframework.web`, `org.springframework.boot.actuate.endpoint.web`, `org.springframework.boot.web.servlet.ServletContextInitializerBeans` |
| sql | `org.springframework.jdbc.core`, `org.hibernate.SQL`, `org.jooq.tools.LoggerListener` |





### [](https://springdoc.cn/spring-boot/features.html#features.logging.shutdown-hook)4.7\. 使用日志 Shutdown Hook



为了在你的应用程序终止时释放日志资源，我们提供了一个Shutdown Hook，它将在JVM退出时触发日志系统清理。 除非你的应用程序是以war文件的形式部署的，否则这个Shutdown Hook会自动注册。 如果你的应用程序有复杂的上下文层次结构，Shutdown Hook可能无法满足你的需求。 如果不能，请禁用关机钩子，并研究底层日志系统直接提供的选项。 例如，Logback提供了 [context selectors](https://logback.qos.ch/manual/loggingSeparation.html)，允许每个记录器在它自己的上下文中被创建。 你可以使用 `logging.register-shutdown-hook` 属性来禁用Shutdown Hook。 将其设置为 `false` 将禁用注册。 你可以在你的 `application.properties` 或 `application.yaml` 文件中设置该属性。







Properties

Yaml





```
logging.register-shutdown-hook=false

```









### [](https://springdoc.cn/spring-boot/features.html#features.logging.custom-log-configuration)4.8\. 自定义日志配置



各种日志系统可以通过在classpath上包含适当的库来激活，并且可以通过在classpath的根目录下或在 Spring `Environment` 属性指定的位置提供一个合适的配置文件来进一步定制： `logging.config`。





您可以通过使用 `org.springframework.boot.logging.LoggingSystem` 系统属性，强制Spring Boot使用特定的日志系统。 该值应该是 `LoggingSystem` 实现的全类名。 你也可以通过使用 `none` 的值来完全禁用Spring Boot的日志配置。





|  | S由于日志是在创建 `ApplicationContext` 之前初始化的，所以不可能从Spring `@Configuration` 文件中的 `@PropertySources` 控制日志。 改变日志系统或完全停用它的唯一方法是通过System properties。 |
| --- | --- |





根据你的日志系统，会加载以下文件。



<colgroup><col><col></colgroup>
| 日志系统 | 配置文件 |
| --- | --- |
| Logback | `logback-spring.xml`, `logback-spring.groovy`, `logback.xml` 或者 `logback.groovy` |
| Log4j2 | `log4j2-spring.xml` 或者 `log4j2.xml` |
| JDK (Java Util Logging) | `logging.properties` |



|  | 在可能的情况下，我们建议你使用 `-spring` 变体来进行日志配置（例如， `logback-spring.xml` 而不是 `logback.xml` ）。 如果你使用标准配置位置，Spring不能完全控制日志初始化。 |
| --- | --- |





|  | 当从 "可执行的jar "中运行时，Java Util Logging有一些已知的类加载问题，会导致问题。 如果可能的话，我们建议你在从 "可执行的jar" 中运行时避免使用它。 |
| --- | --- |





为了帮助定制，其他一些属性从Spring的 `Environment` 转移到System properties，如下表所示。



| Spring Environment | System Property | 备注 |
| --- | --- | --- |
| `logging.exception-conversion-word` | `LOG_EXCEPTION_CONVERSION_WORD` | 记录异常时使用的转换词。 |
| `logging.file.name` | `LOG_FILE` | 如果定义了，它将用于默认的日志配置中。 |
| `logging.file.path` | `LOG_PATH` | 如果定义了，它将用于默认的日志配置中。 |
| `logging.pattern.console` | `CONSOLE_LOG_PATTERN` | 在控制台（stdout）使用的日志输出模式。 |
| `logging.pattern.dateformat` | `LOG_DATEFORMAT_PATTERN` | date 格式化. |
| `logging.charset.console` | `CONSOLE_LOG_CHARSET` | 控制台输出日志的字符编码。 |
| `logging.threshold.console` | `CONSOLE_LOG_THRESHOLD` | 用于控制台日志记录的日志级别。 |
| `logging.pattern.file` | `FILE_LOG_PATTERN` | 要在文件中使用的日志模式（如果 `LOG_FILE` 被启用）。 |
| `logging.charset.file` | `FILE_LOG_CHARSET` | 文件日志的字符编码（如果 `LOG_FILE` 被启用）。 |
| `logging.threshold.file` | `FILE_LOG_THRESHOLD` | 用于文件日志记录的日志级别。 |
| `logging.pattern.level` | `LOG_LEVEL_PATTERN` | 渲染日志级别时使用的格式（默认为 `%5p` ）。 |
| `PID` | `PID` | 当前的进程ID |



如果你使用Logback，以下属性也会被转移。



| Spring Environment | System Property | 备注 |
| --- | --- | --- |
| `logging.logback.rollingpolicy.file-name-pattern` | `LOGBACK_ROLLINGPOLICY_FILE_NAME_PATTERN` | 滚动日志文件名的模式（默认为 `${LOG_FILE}.%d{yyyy-MM-dd}.%i.gz` ）。 |
| `logging.logback.rollingpolicy.clean-history-on-start` | `LOGBACK_ROLLINGPOLICY_CLEAN_HISTORY_ON_START` | 是否在启动时清理归档日志文件。 |
| `logging.logback.rollingpolicy.max-file-size` | `LOGBACK_ROLLINGPOLICY_MAX_FILE_SIZE` | 最大日志文件大小。 |
| `logging.logback.rollingpolicy.total-size-cap` | `LOGBACK_ROLLINGPOLICY_TOTAL_SIZE_CAP` | 要保留的日志备份的总大小。 |
| `logging.logback.rollingpolicy.max-history` | `LOGBACK_ROLLINGPOLICY_MAX_HISTORY` | 要保留的最大归档日志文件数量。 |



所有支持的日志系统在解析其配置文件时都可以从 System properties 中获取属性。 例子见 `spring-boot.jar` 中的默认配置。





*   [Logback](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot/src/main/resources/org/springframework/boot/logging/logback/defaults.xml)

*   [Log4j 2](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot/src/main/resources/org/springframework/boot/logging/log4j2/log4j2.xml)

*   [Java Util logging](https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot/src/main/resources/org/springframework/boot/logging/java/logging-file.properties)





|  | 如果你想在日志属性中使用占位符，你应该使用[Spring Boot的语法](https://springdoc.cn/spring-boot/features.html#features.external-config.files.property-placeholders)而不是底层框架的语法。 值得注意的是，如果你使用Logback，你应该使用 `:` 作为属性名和其默认值之间的分隔符，而不是使用 `:-` 。 |
| --- | --- |





|  | 你可以通过只覆盖 `LOG_LEVEL_PATTERN` （或使用Logback的 `logging.pattern.level` ）来向日志行添加MDC和其他临时内容。 例如，如果你使用 `logging.pattern.level=user:%X{user} %5p` ，那么默认的日志格式包含一个 "user" 的MDC条目，如果它存在的话，如下例所示。 2019-08-30 12:30:04.031 user:someone INFO 22174 --- [  nio-8080-exec-0] demo.ControllerHandling authenticated request  |
| --- | --- |







### [](https://springdoc.cn/spring-boot/features.html#features.logging.logback-extensions)4.9\. Logback 扩展



Spring Boot包括一些对Logback的扩展，可以帮助进行高级配置。 你可以在你的 `logback-spring.xml` 配置文件中使用这些扩展。





|  | 因为标准的 `logback.xml` 配置文件被过早加载，你不能在其中使用扩展。 你需要使用 `logback-spring.xml` 或者定义一个 `logging.config` 属性。 |
| --- | --- |





|  | 扩展程序不能与 [Logback的配置扫描](https://logback.qos.ch/manual/configuration.html#autoScan) 一起使用。 如果你试图这样做，对配置文件进行修改会导致类似于以下的错误被记录下来。 |
| --- | --- |







 ERROR in ch.qos.logback.core.joran.spi.Interpreter@4:71 - no applicable action for [springProperty], current ElementPath is [[configuration][springProperty]]
ERROR in ch.qos.logback.core.joran.spi.Interpreter@4:71 - no applicable action for [springProfile], current ElementPath is [[configuration][springProfile]] 







#### [](https://springdoc.cn/spring-boot/features.html#features.logging.logback-extensions.profile-specific)4.9.1\. 特定的配置文件



`<springProfile>` 标签让你可以根据活动的Spring配置文件选择性地包括或排除配置的部分， 支持在 `<configuration>` 元素的任何地方定义它。 使用 `name` 属性来指定接受配置的配置文件。 `<springProfile>` 标签可以包含一个配置文件名称（例如 `staging` ）或一个配置文件表达式。 配置文件表达式允许表达更复杂的配置文件逻辑，例如 `production & (eu-central | eu-west)` 。 查看 [Spring 框架参考指南](https://docs.spring.io/spring-framework/docs/6.0.5/reference/html/core.html#beans-definition-profiles-java) 以了解更多细节。 下面的列表显示了三个样本配置文件。







```
<springProfile name="staging">
    <!-- configuration to be enabled when the "staging" profile is active -->
</springProfile>

<springProfile name="dev | staging">
    <!-- configuration to be enabled when the "dev" or "staging" profiles are active -->
</springProfile>

<springProfile name="!production">
    <!-- configuration to be enabled when the "production" profile is not active -->
</springProfile>
```









#### [](https://springdoc.cn/spring-boot/features.html#features.logging.logback-extensions.environment-properties)4.9.2\. （环境属性）Environment Properties



`<springProperty>` 标签可以访问 Spring `Environment` 中的属性，以便在Logback中使用。 如果你想在Logback配置中访问 `application.properties` 文件中的值，这样做会很有用。 该标签的工作方式与Logback的标准 `<property>` 标签类似。 然而，你不是直接指定一个 `value` ，而是指定属性的 `source` （来自 `Environment` ）。 如果你需要在 `local` 范围以外的地方存储该属性，你可以使用 `scope` 属性。 如果你需要一个后备值（默认值）（万一该属性没有在 `Environment` 中设置），你可以使用 `defaultValue` 属性。 下面的例子显示了如何公开属性以便在Logback中使用。







```
<springProperty scope="context" name="fluentHost" source="myapp.fluentd.host"
        defaultValue="localhost"/>

    <remoteHost>${fluentHost}</remoteHost>
    ...

```







|  | `source` 必须以kebab风格指定（如 `my.property-name` ）。 然而，属性可以通过使用宽松的规则添加到 `Environment` 中。 |
| --- | --- |









### [](https://springdoc.cn/spring-boot/features.html#features.logging.log4j2-extensions)4.10\. Log4j2 扩展



Spring Boot包括一些对Log4j2的扩展，可以帮助进行高级配置。你可以在任何 `log4j2-spring.xml` 配置文件中使用这些扩展。





|  | 因为标准的 `log4j2.xml` 配置文件被过早加载，你不能在其中使用扩展。你需要使用 `log4j2-spring.xml` 或者定义一个 ``logging.config`` 属性。 |
| --- | --- |





|  | 这些扩展取代了Log4J提供的 [Spring Boot支持](https://logging.apache.org/log4j/2.x/log4j-spring-boot/index.html)。 你应该确保在你的构建中不包括 `org.apache.logging.log4j:log4j-spring-boot` 模块。 |
| --- | --- |





#### [](https://springdoc.cn/spring-boot/features.html#features.logging.log4j2-extensions.profile-specific)4.10.1\. 特定配置文件配置



`<SpringProfile>` 标签让你可以根据活动的Spring配置文件选择性地包括或排除配置的部分。配置文件部分被支持在 `<Configuration>` 元素的任何地方。使用 `name` 属性来指定哪个配置文件接受配置。 `<SpringProfile>` 标签可以包含一个配置文件名称（例如 `staging`）或一个配置文件表达式。 配置文件表达式允许表达更复杂的配置文件逻辑，例如 `production & (eu-central | eu-west)`。查看 [Spring框架参考指南](https://docs.spring.io/spring-framework/docs/6.0.5/reference/html/core.html#beans-definition-profiles-java) 以了解更多细节。 下面的列表显示了三个样本配置文件。







```
<SpringProfile name="staging">
    <!-- configuration to be enabled when the "staging" profile is active -->
</SpringProfile>

<SpringProfile name="dev | staging">
    <!-- configuration to be enabled when the "dev" or "staging" profiles are active -->
</SpringProfile>

<SpringProfile name="!production">
    <!-- configuration to be enabled when the "production" profile is not active -->
</SpringProfile>
```









#### [](https://springdoc.cn/spring-boot/features.html#features.logging.log4j2-extensions.environment-properties-lookup)4.10.2\. 环境（Environment）属性查找



如果你想在Log4j2配置中引用Spring `Environment` 中的属性，你可以使用 `spring:` 前缀 [查找](https://logging.apache.org/log4j/2.x/manual/lookups.html)。如果你想在Log4j2配置中访问 `application.properties` 文件中的值，这样做会很有用。





下面的例子显示了如何设置一个名为 `applicationName` 的Log4j2属性，它从Spring `Environment` 中读取 `spring.application.name`。







```
<Properties>
    <Property name="applicationName">${spring:spring.application.name}</Property>
</Properties>
```







|  | 查询key应以kebabf风格指定（如 `my.property-name`）。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.logging.log4j2-extensions.environment-property-source)4.10.3\. Log4j2 系统属性（System Properties）



Log4j2支持一些 [System Properties](https://logging.apache.org/log4j/2.x/manual/configuration.html#SystemProperties)，可以用来配置各种项目。例如，`log4j2.skipJansi` 系统属性可以用来配置 `ConsoleAppender` 是否会在Windows上尝试使用 [Jansi](https://github.com/fusesource/jansi) 输出流。





Log4j2 初始化后加载的所有系统属性都可以从Spring `Environment` 中获得。例如，你可以在 `application.properties` 文件中添加 `log4j2.skipJansi=false`，让 `ConsoleAppender` 在Windows上使用Jansi。





|  | 只有当系统属性（system properties）和操作系统环境变量不包含正在加载的值时，才会考虑Spring `Environment`。 |
| --- | --- |





|  | 在早期Log4j2初始化期间加载的系统属性不能引用Spring `Environment`。例如，Log4j2用于允许选择默认Log4j2实现的属性是在 Spring Environment 可用之前使用的。 |
| --- | --- |







