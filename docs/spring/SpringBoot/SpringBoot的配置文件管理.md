## 1.SpringBoot配置管理的背景与基础

为实现快速搭建和开发，项目以Springboot框架搭建，springboot搭建的项目可以将项目直接打成jar包并运行，无需自己安装配置Tomcat或者其他服务器，是一种方便快捷的部署方式。

假设项目以最常规的方式打包成一个整体的jar包部署，即配置文件和第三方依赖包都包含在jar包里，就会有如下两个问题

问题一：项目运行过程中，要改动配置文件的话需要重新打包并部署。

问题二：多个第三方依赖包都相近的项目要部署在同一台服务器时，各自的jar包都包含了相同的第三方依赖包（假设项目jar包有100M，第三方依赖包可能就占用了99M），这样第三方依赖包冗余造成了服务器资源的浪费以及降低了项目部署的效率。

如果将各项目的配置文件、第三方依赖包都提取到jar包外统一管理，这样即提升了项目打包效率又节约了服务器的磁盘消耗，同时项目的运维也是非常方便的，改动了配置文件重启下服务就可以了，无需重新构建部署。

下面是具体的实现方案

### 1.1 **配置文件统一管理**

```
	- springboot核心配置文件
	- Springboot读取核心配置文件（application.properties）的优先级为
	- Jar包同级目录的config目录
	- Jar包同级目录
	- classPath(即resources目录)的config目录
	- classpath目录

```

上面是springboot默认去拿自己的核心配置文件的优先级，还有一种最高优先级的方式是项目启动时通过命令的方式指定项目加载核心配置文件，命令如下
java –jar -Dspring.config.location=xxx/xxx/xxxx.properties xxxx.jar

如果Spring Boot在优先级更高的位置找到了配置，那么它会无视优先级更低的配置

### 1.2 **其他资源配置文件**

上面描述的Springboot核心文件已经能够提取出jar包外进行管理了，但是还有其他一些业务上的配置文件，如数据源配置文件，公共资源定义配置文件（常量，FTP信息等），quartz定时器，日志等配置文件我们如何去提取出来并确保能在代码中引用到呢

我们知道Springboot项目可以通过注解方式来获取相关配置文件，所以我们也是通过注解方式让项目能够引用到jar包外部的配置文件的，如下图：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121170400864.png)

@PropertySource里面的value有两个值，第一个是classpath下config目录下的数据源配置文件，第二个则是根据spring.profiles.path动态获取的目录，spring.profiles.path是我们在核心文件自定义的一个配置项，它的值是我们配置文件统一管理的文件夹路径，后面的ignoreResourceNotFound=true则是设定假如根据前面一个路径没有找到相关配置文件，则根据第二个路径去找。

我们还可以直接根据路径，用FileSystemResource类去加载一个配置文件实例出来，如下图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121170338934.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0dlZUxvb25n,size_16,color_FFFFFF,t_70)

原理类似，根据在核心文件自定义的统一配置目录的路径来加载配置文件

另外logback日志配置文件加载方式如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121170306979.png)

综上所述，我们梳理一下实现方案的思路

```
	- 在springboot核心文件里定义一个spring.profiles.path配置项，它的值指向我们所有配置文件统一放置的目录，包含核心文件自身也是放置在里面的

	- 代码或者配置文件里加载配置文件的地方也应该获取spring.profiles.path配置项来动态加载该路径下的配置文件

	- Pom.xml文件修改打包相关模块，将配置文件排除，这样我们打出的jar包是不含配置文件的，打包配置请参考文档节点3

	- 启动jar包时，通过命令指定加载的核心文件为spring.profiles.path下的核心文件

```
**第三方依赖包统一管理**

通常第三方jar包可以打进jar包里，也可以放在项目jar包同级目录下的lib目录，我们可以根据修改pom.xml打包配置来实现，请参考文档节点3打包配置

**打包配置**

```
<build>
		<resources>
			<resource>
				<directory>src/main/java</directory>
				<includes>
					<include>**/*.properties</include>
					<include>**/*.xml</include>
				</includes>
				<filtering>true</filtering>
			</resource>
			<resource>
				<directory>src/main/resources</directory>
				<!—打包时排除配置文件-->
                                <excludes>
					<exclude>**/*.properties</exclude>
					<exclude>**/*.xml</exclude>
					<exclude>**/*.yml</exclude>
				</excludes>
				<filtering>false</filtering>
			</resource>
		</resources>
		<plugins>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				maven-compiler-plugin
				<configuration>
					<source>1.8</source>
					<target>1.8</target>
					<fork>true</fork>
					<skip>true</skip>
					<executable>
						C:/Program Files/Java/jdk1.8.0_161/bin/javac.exe
					</executable>
				</configuration>
			</plugin>
			<plugin>
				maven-jar-plugin
				<configuration>
					
						<manifest>
							true
							<classpathPrefix>lib/</classpathPrefix>
							<useUniqueVersions>false</useUniqueVersions>
							<mainClass>com.xrq.demo.Application</mainClass>
						</manifest>
						<manifestEntries>
							<Class-Path>./</Class-Path>
						</manifestEntries>
					
					<excludes>
						<exclude>*.properties</exclude>
						<exclude>*.yml</exclude>
						<exclude>*.xml</exclude>
						<exclude>config/**</exclude>
					</excludes>
				</configuration>
			</plugin>
			<plugin>
				<groupId>org.apache.maven.plugins</groupId>
				maven-dependency-plugin
				<executions>
					<execution>
						<id>copy</id>
						<phase>package</phase>
						<goals>
							<goal>copy-dependencies</goal>
						</goals>
						<configuration>
							<outputDirectory>
								${project.build.directory}/lib
							</outputDirectory>
						</configuration>
					</execution>
				</executions>
			</plugin>
		</plugins>
	</build>

```

改好pom.xml的build模块后，就可以通过mvn package 或者mvn install打出我们的jar包了

1.  **项目管理shell脚本编写**
    自定义shell脚本，实现项目的启动，停止，状态，重启操作：

```
#!/bin/bash 
#这里可替换为你自己的执行程序,其他代码无需更改 
APP_NAME=demo1-0.0.1-SNAPSHOT.jar 
JVM="-server -Xms512m -Xmx512m -XX:PermSize=64M -XX:MaxNewSize=128m -XX:MaxPermSize=128m -Djava.awt.headless=true -XX:+CMSClassUnloadingEnabled -XX:+CMSPermGenSweepingEnabled"
APPFILE_PATH="-Dspring.config.location=/usr/local/demo/config/application-demo1.properties"
#使用说明,用来提示输入参数 
usage() { 
echo "Usage: sh 执行脚本.sh [start|stop|restart|status]" 
exit 1 
} 
#检查程序是否在运行 
is_exist(){ 
pid=`ps -ef|grep $APP_NAME|grep -v grep|awk '{print $2}' ` 
#如果不存在返回1,存在返回0 
if [ -z "${pid}" ]; then 
return 1 
else 
return 0 
fi 
} 
#启动方法 
start(){ 
is_exist 
if [ $? -eq "0" ]; then 
echo "${APP_NAME} is already running. pid=${pid} ." 
else 
nohup java $JVM -jar $APPFILE_PATH $APP_NAME > /dev/null 2>&1 
fi
} 
#停止方法 
stop(){ 
is_exist 
if [ $? -eq "0" ]; then 
kill -9 $pid 
else 
echo "${APP_NAME} is not running" 
fi 
} 
#输出运行状态 
status(){ 
is_exist 
if [ $? -eq "0" ]; then 
echo "${APP_NAME} is running. Pid is ${pid}" 
else 
echo "${APP_NAME} is NOT running." 
fi 
} 
#重启 
restart(){ 
stop 
start 
} 
#根据输入参数,选择执行对应方法,不输入则执行使用说明 
case "$1" in 
"start") 
start 
;; 
"stop") 
stop 
;; 
"status") 
status 
;; 
"restart") 
restart 
;; 
*) 
usage 
;; 
esac

```

**部署**

linux服务器上新建个文件夹，将我们打好的项目jar包都丢进去，在jar包的同级目录新建config和lib文件夹，分别将配置文件和第三方依赖包丢进去，其结构如下图，*.sh为自己写的项目启动shell脚本

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190121170223384.png)

打开config内的springboot核心文件（如application-demo1.properties文件），

spring.profiles.path配置项改成当前配置文件所在的目录，假设为/usr/local/demo/config

打开*.sh脚本，修改APPFILE_PATH的值，如下

APPFILE_PATH="-Dspring.config.location=/usr/local/demo/config/application-demo1.properties"

**项目管理**

进入jar包所在目录执行下面命令

sh [demo1.sh](http://demo1.sh/) start 启动项目

sh [demo1.sh](http://demo1.sh/) stop 停止项目

sh [demo1.sh](http://demo1.sh/) restart重启项目

sh [demo1.sh](http://demo1.sh/) status项目状态

## 2\. 外部化的配置





Spring Boot可以让你将配置外部化，这样你就可以在不同的环境中使用相同的应用程序代码。 你可以使用各种外部配置源，包括Java properties 文件、YAML文件、环境变量和命令行参数。





属性值可以通过使用 `@Value` 注解直接注入你的Bean，也可以通过Spring 的 `Environment` 访问，或者通过 `@ConfigurationProperties` [绑定到对象](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties)。





Spring Boot 使用一个非常特别的 `PropertySource` 顺序，旨在允许合理地重写值。 后面的 property source 可以覆盖前面属性源中定义的值。 按以下顺序考虑。





1.  默认属性（通过 `SpringApplication.setDefaultProperties` 指定）。

2.  @Configuration 类上的 [`@PropertySource`](https://docs.spring.io/spring-framework/docs/6.0.5/javadoc-api/org/springframework/context/annotation/PropertySource.html) 注解。请注意，这样的属性源直到application context被刷新时才会被添加到环境中。这对于配置某些属性来说已经太晚了，比如 `logging.*` 和 `spring.main.*` ，它们在刷新开始前就已经被读取了。

3.  配置数据（如 `application.properties` 文件）。

4.  `RandomValuePropertySource`，它只有 `random.*` 属性。

5.  操作系统环境变量

6.  Java System properties (`System.getProperties()`).

7.  `java:comp/env` 中的 JNDI 属性。

8.  `ServletContext` init parameters.

9.  `ServletConfig` init parameters.

10.  来自 `SPRING_APPLICATION_JSON` 的属性（嵌入环境变量或系统属性中的内联JSON）。

11.  命令行参数

12.  你在测试中的 `properties` 属性。在 [`@SpringBootTest`](https://docs.spring.io/spring-boot/docs/3.1.0-SNAPSHOT/api/org/springframework/boot/test/context/SpringBootTest.html) 和测试注解中可用，[用于测试你的应用程序的一个特定片断](https://springdoc.cn/spring-boot/features.html#features.testing.spring-boot-applications.autoconfigured-tests)。

13.  你测试中的https://docs.spring.io/spring-framework/docs/6.0.5/javadoc-api/org/springframework/test/context/TestPropertySource.html[`@TestPropertySource`] 注解.

14.  当devtools处于活动状态时，`$HOME/.config/spring-boot` 目录下的[Devtools全局设置属性](https://springdoc.cn/spring-boot/using.html#using.devtools.globalsettings)。





配置数据文件按以下顺序考虑。





1.  在你的jar中打包的[Application properties](https://springdoc.cn/spring-boot/features.html#features.external-config.files)（application.properties 和 YAML）。

2.  在你的jar中打包的 [特定的 Profile application properties](https://springdoc.cn/spring-boot/features.html#features.external-config.files.profile-specific)（`application-{profile}.properties` 和 YAML）。

3.  在你打包的jar之外的[Application properties](https://springdoc.cn/spring-boot/features.html#features.external-config.files)性（application.properties和YAML）。

4.  在你打包的jar之外的[特定的 Profile application properties](https://springdoc.cn/spring-boot/features.html#features.external-config.files.profile-specific)（ `application-{profile}.properties` 和YAML）。





|  | 建议你在整个应用程序中坚持使用一种格式。 如果你在同一个地方有 `.properties` 和 `.yml` 格式的配置文件，`.properties` 优先。 |
| --- | --- |





为了提供一个具体的例子，假设你开发了一个 `@Component`，使用了一个 `name` 属性，如下面的例子所示。







Java

Kotlin





```
@Component
public class MyBean {

    @Value("${name}")
    private String name;

    // ...

}

```







在你的应用程序的classpath（例如，在你的jar中），你可以有一个 `application.properties` 文件，为 `name` 提供一个合理的默认属性值。当在一个新的环境中运行时，可以在你的jar之外提供一个 `application.properties` 文件来覆盖 `name` 。对于一次性的测试，你可以用一个特定的命令行参数来启动（例如，`java -jar app.jar --name="Spring"`）。





|  | `env` 和 `configprops` 端点在确定一个属性为什么有一个特定的值时非常有用。你可以使用这两个端点来诊断意外的属性值。详见 "[生产就绪功能](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints)" 部分。 |
| --- | --- |





### [](https://springdoc.cn/spring-boot/features.html#features.external-config.command-line-args)2.1\. 访问命令行属性



默认情况下，`SpringApplication` 会将任何命令行选项参数（即以 `--` 开头的参数，如 `--server.port=9000` ）转换为 `property` 并将其添加到Spring `Environment` 中。 如前所述，命令行属性总是优先于基于文件的属性源。





如果你不希望命令行属性被添加到 `Environment` 中，你可以通过 `SpringApplication.setAddCommandLineProperties(false)` 禁用它们。







### [](https://springdoc.cn/spring-boot/features.html#features.external-config.application-json)2.2\. JSON Application Properties



环境变量和系统属性往往有限制，这意味着有些属性名称不能使用。 为了帮助解决这个问题，Spring Boot允许你将一个属性块编码为一个单一的JSON结构。





当你的应用程序启动时，任何 `spring.application.json` 或 `SPRING_APPLICATION_JSON` 属性将被解析并添加到 `Environment` 中。





例如，`SPRING_APPLICATION_JSON` 属性可以在 UN*X shell 的命令行中作为环境变量提供。







```
$ SPRING_APPLICATION_JSON='{"my":{"name":"test"}}' java -jar myapp.jar
```







在前面的例子中，你在Spring的 `Environment` 中最终得到了 `my.name=test`。





同样的JSON也可以作为一个系统属性提供。







```
$ java -Dspring.application.json='{"my":{"name":"test"}}' -jar myapp.jar
```







或者你可以通过使用一个命令行参数来提供JSON。







```
$ java -jar myapp.jar --spring.application.json='{"my":{"name":"test"}}'
```







如果你要部署到一个经典的应用服务器中，你也可以使用一个名为 `java:comp/env/spring.application.json` 的JNDI变量。





|  | 尽管JSON中的 `null` 值将被添加到生成的属性源中，但 `PropertySourcesPropertyResolver` 将 `null` 属性视为缺失值。 这意味着JSON不能用 `null` 值覆盖来自低阶属性源的属性。 |
| --- | --- |







### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files)2.3\. 外部的 Application Properties



当你的应用程序启动时，Spring Boot会自动从以下位置找到并加载 `application.properties` 和 `application.yaml` 文件。





1.  classpath



    1.  classpath 根路径

    2.  classpath 下的 `/config` 包



2.  当前目录



    1.  当前目录下

    2.  当前目录下的 `config/` 子目录

    3.  `config/` 子目录的直接子目录







列表按优先级排序（较低项目的值覆盖较早项目的值）。 加载的文件被作为 `PropertySources` 添加到Spring的 `Environment` 中。





如果你不喜欢 `application` 作为配置文件名称，你可以通过指定 `spring.config.name` 环境属性切换到另一个文件名称。 例如，为了寻找 `myproject.properties` 和 `myproject.yaml` 文件，你可以按以下方式运行你的应用程序。







```
$ java -jar myproject.jar --spring.config.name=myproject
```







你也可以通过使用 `spring.config.location` 环境属性来引用一个明确的位置。 该属性接受一个逗号分隔的列表，其中包含一个或多个要检查的位置。





下面的例子显示了如何指定两个不同的文件。







```
$ java -jar myproject.jar --spring.config.location=\
    optional:classpath:/default.properties,\
    optional:classpath:/override.properties
```







|  | 如果 [配置文件是可选的](https://springdoc.cn/spring-boot/features.html#features.external-config.files.optional-prefix)，并且可以是不存在的，那么请使用 `optional:` 前缀。 |
| --- | --- |





|  | `spring.config.name`, `spring.config.location`, 和 `spring.config.extra-location` 很早就用来确定哪些文件必须被加载。 它们必须被定义为环境属性（通常是操作系统环境变量，系统属性，或命令行参数）。 |
| --- | --- |





如果 `spring.config.location` 包含目录（而不是文件），它们应该以 `/` 结尾。 在运行时，它们将被附加上由 `spring.config.name` 生成的名称，然后被加载。 在 `spring.config.location` 中指定的文件被直接导入。





|  | 目录和文件位置值也被扩展，以检查[特定的配置文件](https://springdoc.cn/spring-boot/features.html#features.external-config.files.profile-specific)。例如，如果你的 `spring.config.location` 是 `classpath:myconfig.properties`，你也会发现适当的 `classpath:myconfig-<profile>.properties` 文件被加载。 |
| --- | --- |





在大多数情况下，你添加的每个 `spring.config.location` 项将引用一个文件或目录。 位置是按照它们被定义的顺序来处理的，后面的位置可以覆盖前面的位置的值。





如果你有一个复杂的位置设置，而且你使用特定的配置文件，你可能需要提供进一步的提示，以便Spring Boot知道它们应该如何分组。一个位置组是一个位置的集合，这些位置都被认为是在同一级别。例如，你可能想把所有classpath位置分组，然后是所有外部位置。一个位置组内的项目应该用 `;` 分隔。更多细节见 “[指定 profile](https://springdoc.cn/spring-boot/features.html#features.external-config.files.profile-specific)” 部分的例子。





通过使用 `spring.config.location` 配置的位置取代默认位置。 例如，如果 `spring.config.location` 被配置为 `optional:classpath:/custom-config/,optional:file:./custom-config/` ，考虑的完整位置集如下。





1.  `optional:classpath:custom-config/`

2.  `optional:file:./custom-config/`





如果你喜欢添加额外的位置，而不是替换它们，你可以使用 `spring.config.extra-location` 。 从附加位置加载的属性可以覆盖默认位置的属性。 例如，如果 `spring.config.extra-location` 被配置为 `optional:classpath:/custom-config/,optional:file:./custom-config/` ，考虑的完整位置集如下。





1.  `optional:classpath:/;optional:classpath:/config/`

2.  `optional:file:./;optional:file:./config/;optional:file:./config/*/`

3.  `optional:classpath:custom-config/`

4.  `optional:file:./custom-config/`





这种搜索排序让你在一个配置文件中指定默认值，然后在另一个文件中选择性地覆盖这些值。 你可以在其中一个默认位置的 `application.properties` （或你用 `spring.config.name` 选择的其他basename）中为你的应用程序提供默认值。 然后，这些默认值可以在运行时被位于其中一个自定义位置的不同文件覆盖。





|  | 如果你使用环境变量而不是系统属性，大多数操作系统不允许使用句点分隔的键名，但你可以使用下划线代替（例如， `SPRING_CONFIG_NAME` 而不是 `spring.config.name` ）。 参见[从环境变量绑定](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables) 了解详情。 |
| --- | --- |





|  | 如果你的应用程序在servlet容器或应用服务器中运行，那么JNDI属性（在 `java:comp/env` 中）或servlet上下文初始化参数可以代替环境变量或系统属性，或者与之一样。 |
| --- | --- |





#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.optional-prefix)2.3.1\. 可选的位置(Optional Locations)



默认情况下，当指定的配置数据位置不存在时，Spring Boot将抛出一个 `ConfigDataLocationNotFoundException` ，你的应用程序将无法启动。





如果你想指定一个位置，但你不介意它并不总是存在，你可以使用 `optional:` 前缀。你可以在 `spring.config.location和spring.config.extra-location` 属性中使用这个前缀，也可以在 [`spring.config.import`](https://springdoc.cn/spring-boot/features.html#features.external-config.files.importing) 声明中使用。





例如，`spring.config.import` 值为 `optional:file:./myconfig.properties` 允许你的应用程序启动，即使 `myconfig.properties` 文件丢失。





如果你想忽略所有的 `ConfigDataLocationNotFoundExceptions` 并始终继续启动你的应用程序，你可以使用 `spring.config.on-not-found` 属性。 使用 `SpringApplication.setDefaultProperties(..)` 或使用系统/环境变量将其值设置为 `ignore`。







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.wildcard-locations)2.3.2\. 通配符地址



如果一个配置文件的位置在最后一个路径段中包含 `*` 字符，它就被认为是一个通配符位置。 通配符在加载配置时被扩展，因此，直接的子目录也被检查。 通配符位置在Kubernetes这种有多个配置属性的来源的环境中特别有用。





例如，如果你有一些Redis配置和一些MySQL配置，你可能想把这两部分配置分开，同时要求这两部分都存在于一个 `application.properties` 文件中。





这可能会导致两个独立的 `application.properties` 文件挂载在不同的位置，如 `/config/redis/application.properties` 和 `/config/mysql/application.properties` 。 在这种情况下，有一个通配符位置 `config/*/` ，将导致两个文件被处理。





默认情况下，Spring Boot将 `config/*/` 列入默认搜索位置。 这意味着你的jar之外的 `/config` 目录的所有子目录都会被搜索到。





你可以在 `spring.config.location` 和 `spring.config.extra-location` 属性中使用通配符位置。





|  | 通配符位置必须只包含一个 `*` 并以 `*/` 结尾，用于搜索属于目录的位置，或 `*/<filename>` 用于搜索属于文件的位置。 带有通配符的位置将根据文件名的绝对路径按字母顺序排序。 |
| --- | --- |





|  | 通配符位置只对外部目录起作用。 你不能在 `classpath:` 位置中使用通配符。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.profile-specific)2.3.3\. 特定文件（Profile Specific Files）



除了 `application` 属性文件，Spring Boot还将尝试使用 `application-{profile}` 的命名惯例加载profile特定的文件。 例如，如果你的应用程序激活了名为 `prod` 的配置文件（`spring.profiles.active=prod`）并使用YAML文件，那么 `application.yml` 和 `application-prod.yml` 都将被考虑。





特定文件(`profiles`)的属性与标准的 `application.properties` 的位置相同，特定文件总是优先于非特定文件。 如果指定了几个配置文件，则采用最后胜出的策略。 例如，如果配置文件 `prod,live` 是由 `spring.profiles.active` 属性指定的，`application-prod.properties` 中的值可以被 `application-live.properties` 中的值所覆盖。





|  | 最后胜出的策略适用于[location group](https://springdoc.cn/spring-boot/features.html#features.external-config.files.location-groups)级别。 `spring.config.location` 的 `classpath:/cfg/,classpath:/ext/` 将不会有与 `classpath:/cfg/;classpath:/ext/` 相同的覆盖规则。例如，拿我们上面的 `prod,live` 例子来说，我们可能有以下文件。 /cfg  application-live.properties/ext  application-live.properties  application-prod.properties 当我们有一个 `spring.config.location` 为 `classpath:/cfg/,classpath:/ext/` 时，我们会在所有 `/ext` 文件之前处理所有 `/cfg` 文件。1.  `/cfg/application-live.properties`        2.  `/ext/application-prod.properties`        3.  `/ext/application-live.properties`        当我们用 `classpath:/cfg/;classpath:/ext/` 代替时（用 `;` 分隔符），我们在同一级别处理 `/cfg` 和 `/ext` 。1.  `/ext/application-prod.properties`        2.  `/cfg/application-live.properties`        3.  `/ext/application-live.properties`         |
| --- | --- |





`Environment` 有一组默认的配置文件（默认为 `[default]` ），如果没有设置活动的配置文件，就会使用这些配置文件。 换句话说，如果没有明确激活的配置文件，那么就会考虑来自 `application-default` 的属性。





|  | 属性文件只被加载一次。 如果你已经直接[导入了](https://springdoc.cn/spring-boot/features.html#features.external-config.files.importing)一个配置文件的特定属性文件，那么它将不会被第二次导入。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.importing)2.3.4\. 导入额外的数据



application properties 中可以使用 `spring.config.import` 属性从其他地方导入更多的配置数据。 导入在被发现时被处理，并被视为紧接着声明导入的文件下面插入的额外文件。





例如，你可能在你的 classpath `application.properties` 文件中有以下内容。







Properties

Yaml





```
spring.application.name=myapp
spring.config.import=optional:file:./dev.properties

```







这将触发导入当前目录下的 `dev.properties` 文件（如果存在这样的文件）。 导入的 `dev.properties` 中的值将优先于触发导入的文件。 在上面的例子中，`dev.properties` 可以将 `spring.application.name` 重新定义为一个不同的值。





一个导入只会被导入一次，无论它被声明多少次。 一个导入在properties/yaml文件内的单个文件中被定义的顺序并不重要。 例如，下面的两个例子产生相同的结果。







Properties

Yaml





```
spring.config.import=my.properties
my.property=value

```









Properties

Yaml





```
my.property=value
spring.config.import=my.properties

```







在上述两个例子中，`my.properties` 文件的值将优先于触发其导入的文件。





在一个单一的 `spring.config.import` 属性下可以指定多个位置。 位置将按照它们被定义的顺序被处理，后来的导入将被优先处理。





|  | 在适当的时候，[特定配置文件的变体](https://springdoc.cn/spring-boot/features.html#features.external-config.files.profile-specific)也被考虑导入。 上面的例子将导入 `my.properties` 以及任何 `my-<profile>.properties` 变体。 |
| --- | --- |





|  | Spring Boot 提供了可插拔的API（插件），允许支持各种不同的位置地址。 默认情况下，你可以导入Java Properties、YAML和 “[配置树](https://springdoc.cn/spring-boot/features.html#features.external-config.files.configtree)” 。第三方jar可以提供对其他技术的支持（不要求必须是本地文件）。 例如，你可以想象配置数据来自外部存储，如Consul、Apache ZooKeeper或Netflix Archaius（包括Nacos）。如果你想支持你自己的位置（实现自己定义的配置加载），请参阅 `org.springframework.boot.context.config` 包中的 `ConfigDataLocationResolver` 和 `ConfigDataLoader` 类。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.importing-extensionless)2.3.5\. 导入无扩展名的文件



有些云平台不能为卷装文件（volume mounted files）添加文件扩展名。 要导入这些无扩展名的文件，你需要给Spring Boot一个提示，以便它知道如何加载它们。 你可以通过把扩展名提示放在方括号里来做到这一点。





例如，假设你有一个 `/etc/config/myconfig` 文件，你希望以yaml形式导入。 你可以用下面的方法从你的 `application.properties` 中导入它。







Properties

Yaml





```
spring.config.import=file:/etc/config/myconfig[.yaml]

```









#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.configtree)2.3.6\. 使用配置树（Configuration Trees）



当在云平台（如Kubernetes）上运行应用程序时，你经常需要读取平台提供的配置值。 将环境变量用于此类目的并不少见，但这可能有缺点，特别是如果该值是 secret 的。





作为环境变量的替代方案，许多云平台现在允许你将配置映射到挂载的数据卷。 例如，Kubernetes 可以卷挂载 [`ConfigMaps`](https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/#populate-a-volume-with-data-stored-in-a-configmap) 和 [`Secrets`](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-files-from-a-pod)。





可以使用两种常见的 volume 挂载模式：





1.  一个文件包含一套完整的属性（通常写成YAML）。

2.  多个文件被写入一个目录树中，文件名成为 ‘key’，内容成为 ‘value’。





对于第一种情况，你可以使用 `spring.config.import` 直接导入YAML或属性文件，[如上所述](https://springdoc.cn/spring-boot/features.html#features.external-config.files.importing)。 对于第二种情况，你需要使用 `configtree:` 前缀，以便Spring Boot知道它需要将所有文件作为属性公开。





举个例子，让我们想象一下，Kubernetes已经挂载了以下volume。







 etc/
  config/
    myapp/
      username
      password 







`username` 文件的内容将是一个配置值，而 `password` 的内容将是一个 secret。





要导入这些属性，你可以在你的 `application.properties` 或 `application.yaml` 文件中添加以下内容。







Properties

Yaml





```
spring.config.import=optional:configtree:/etc/config/

```







然后你可以从 `Environment` 中以常规方式访问或注入 `myapp.username` 和 `myapp.password` 属性。





|  | 配置树下的文件夹构成了属性名称。 在上面的例子中，为了访问属性为 `username` 和 `password`，你可以将 `spring.config.import` 设置为 `optional:configtree:/etc/config/myapp` 。 |
| --- | --- |





|  | 带有点符号的文件名也会被正确映射。 例如，在上面的例子中，`/etc/config` 中名为 `myapp.username` 的文件在 `Environment` 中的属性名将会是 `myapp.username` 。 |
| --- | --- |





|  | 配置树的值可以被绑定到字符串 `String` 和 `byte[]` 类型，这取决于预期的内容。 |
| --- | --- |





如果你有多个配置树要从同一个父文件夹导入，你可以使用通配符快捷方式。 任何以 `/*/` 结尾的 `configtree:` 位置将导入所有直接的子文件夹作为配置树。





例如，给定以下volume：







 etc/
  config/
    dbconfig/
      db/
        username
        password
    mqconfig/
      mq/
        username
        password 







你可以使用 `configtree:/etc/config/*/` 作为导入位置。







Properties

Yaml





```
spring.config.import=optional:configtree:/etc/config/*/

```







这将添加 `db.username`、`db.password`、`mq.username` 和 `mq.password` 属性。





|  | 使用通配符加载的目录是按字母顺序排列的。 如果你需要一个不同的顺序，那么你应该把每个位置作为一个单独的导入列出。 |
| --- | --- |





配置树也可用于Docker secret。 当Docker swarm服务被授予对secret的访问权时，该secret会被装载到容器中。 例如，如果一个名为 `db.password` 的secret。被挂载在 `/run/secrets/` 的位置，你可以用以下方法让 `db.password` 对Spring环境可用。







Properties

Yaml





```
spring.config.import=optional:configtree:/run/secrets/

```









#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.property-placeholders)2.3.7\. 属性占位符



`application.properties` 和 `application.yml` 中的值在使用时通过现有的 `Environment` 过滤，所以你可以参考以前定义的值（例如，来自系统属性或环境变量）。 标准的 `${name}` 属性占位符语法可以用在一个值的任何地方。 属性占位符也可以指定一个默认值，使用 `:` 来分隔默认值和属性名称，例如 `${name:default}` 。





下面的例子显示了带默认值和不带默认值的占位符的使用情况。







Properties

Yaml





```
app.name=MyApp
app.description=${app.name} is a Spring Boot application written by ${username:Unknown}

```







假设 `username` 属性没有在其他地方设置，`app.description` 的值将是 `MyApp is a Spring Boot application written by Unknown`。





|  | 你应该始终使用占位符中的属性名称的规范形式（仅使用小写字母的kebab-case）来引用它们。 这将允许Spring Boot使用与[宽松绑定](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding) `@ConfigurationProperties` 时相同的逻辑。例如，`${demo.item-price}` 将从 `application.properties` 文件中获取 `demo.item-price` 和 `demo.itemPrice` 形式的属性，以及从系统环境中获取 `DEMO_ITEMPRICE` 。 如果你用 `${demo.itemPrice}` 的话， `demo.item-price` 和 `DEMO_ITEMPRICE` 就不会被考虑。 |
| --- | --- |





|  | 你也可以使用这种技术来创建现有Spring Boot属性的 “short” 变体。 详情请参见_[howto.html](https://springdoc.cn/spring-boot/howto.html#howto.properties-and-configuration.short-command-line-arguments)_的方法。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.multi-document)2.3.8\. 使用多文档文件（Working with Multi-Document Files）



Spring Boot允许你将一个物理文件分成多个逻辑文件，每个文件都是独立添加的。 文件是按顺序处理的，从上到下。 后面的文件可以覆盖前面文件中定义的属性。





对于 `application.yml` 文件，使用标准的YAML多文档语法。 三个连续的连字符（`---`）代表一个文件的结束，和下一个文件的开始。





例如，下面的文件有两个逻辑文档。







```
spring:
  application:
    name: "MyApp"
---
spring:
  application:
    name: "MyCloudApp"
  config:
    activate:
      on-cloud-platform: "kubernetes"
```







对于 `application.properties` 文件，一个特殊的 `#---` 或 `!---` 注释被用来标记文件的分割。







```
spring.application.name=MyApp
#---
spring.application.name=MyCloudApp
spring.config.activate.on-cloud-platform=kubernetes
```







|  | properties 文件的分隔符不能有任何前导空白，并且必须正好有三个连字符。 分隔符的前后两行不能是相同的注释前缀。 |
| --- | --- |





|  | 多文档属性文件通常与激活属性一起使用，如 `spring.config.activated.on-profile`。 详见[下一节](https://springdoc.cn/spring-boot/features.html#features.external-config.files.activation-properties)。 |
| --- | --- |





|  | 多文档属性文件不能通过使用 `@PropertySource` 或 `@TestPropertySource` 注解加载。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.files.activation-properties)2.3.9\. 激活属性（Activation Properties）



有时，只在满足某些条件时激活一组特定的属性是很有用的。 例如，你可能有一些属性只有在特定的配置文件被激活时才相关。





你可以使用 `spring.config.activation.*` 有条件地激活一个属性文件。





激活属性有如下。



<caption>Table 2\. activation properties</caption><colgroup><col><col></colgroup>
| 属性 | 说明 |
| --- | --- |
| `on-profile` | 一个必须与之匹配的配置文件表达式，以使文件处于活动状态（激活指定的配置文件时有效）。 |
| `on-cloud-platform` | 必须检测到的 `CloudPlatform`，以使文件处于活动状态。（云平台状态下有效） |



例如，下面指定第二个文件只有在Kubernetes上运行时才有效，并且只有在 “prod” 或 “staging” 配置文件处于活动状态时才有效。







Properties

Yaml





```
myprop=always-set
#---
spring.config.activate.on-cloud-platform=kubernetes
spring.config.activate.on-profile=prod | staging
myotherprop=sometimes-set

```











### [](https://springdoc.cn/spring-boot/features.html#features.external-config.encrypting)2.4\. 加密配置属性（Encrypting Properties）



Spring Boot没有为加密属性值提供任何内置支持，但它提供了Hookm，可以用来修改Spring `Environment` 中包含的值。 `EnvironmentPostProcessor` 接口允许你在应用程序启动前操作 `Environment`。 参见[howto.html](https://springdoc.cn/spring-boot/howto.html#howto.application.customize-the-environment-or-application-context)以了解详情。





如果你需要一种安全的方式来存储凭证和密码， [Spring Cloud Vault](https://cloud.spring.io/spring-cloud-vault/) 项目提供了对在 [HashiCorp Vault](https://www.vaultproject.io/)中存储外部化配置的支持。







### [](https://springdoc.cn/spring-boot/features.html#features.external-config.yaml)2.5\. 使用 YAML



[YAML](https://yaml.org/) 是JSON的超集，因此是指定分层配置数据的方便格式。 只要你的classpath上有 [SnakeYAML](https://github.com/snakeyaml/snakeyaml) 库，`SpringApplication` 类就会自动支持YAML作为properties的替代品。





|  | 如果你使用 “Starter”，SnakeYAML将由 `spring-boot-starter` 自动提供。 |
| --- | --- |





#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.yaml.mapping-to-properties)2.5.1\. 将YAML映射到Properties



YAML 文档需要从其分层格式转换为可与 Spring `Environment` 一起使用的扁平结构。 例如，考虑下面这个YAML文档。







```
environments:
  dev:
    url: "https://dev.example.com"
    name: "Developer Setup"
  prod:
    url: "https://another.example.com"
    name: "My Cool App"
```







为了从 `Environment` 中访问这些属性，它们将被扁平化，如下所示。







```
environments.dev.url=https://dev.example.com
environments.dev.name=Developer Setup
environments.prod.url=https://another.example.com
environments.prod.name=My Cool App
```







同样地，YAML中的列表也需要进行扁平化处理。 它们被表示为带有 `[index]` 索引的key。 例如，考虑下面的YAML。







```
my:
 servers:
 - "dev.example.com"
 - "another.example.com"
```







前面的例子将被转化为如下属性。







```
my.servers[0]=dev.example.com
my.servers[1]=another.example.com
```







|  | 使用 `[index]` 符号的属性可以使用Spring Boot的 `Binder` 类绑定到Java `List` 或 `Set` 对象。 更多细节见下面的 “[类型安全的配置属性](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties)” 部分。 |
| --- | --- |





|  | YAML文件不能通过使用 `@PropertySource` 或 `@TestPropertySource` 注解来加载。 所以，在你需要以这种方式加载值的情况下，你需要使用一个 properties 文件。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.yaml.directly-loading)2.5.2\. 直接加载YAML



Spring Framework提供了两个方便的类，可以用来加载YAML文档。 `YamlPropertiesFactoryBean` 将YAML作为 `Properties` 加载，`YamlMapFactoryBean` 将YAML作为 `Map` 加载。





如果你想把YAML加载为Spring的 `PropertySource` ，你也可以使用 `YamlPropertySourceLoader` 类。









### [](https://springdoc.cn/spring-boot/features.html#features.external-config.random-values)2.6\. 配置随机值



The `RandomValuePropertySource` is useful for injecting random values (for example, into secrets or test cases). It can produce integers, longs, uuids, or strings, as shown in the following example:





`RandomValuePropertySource` 对于注入随机值很有用（例如，注入密码或测试案例）。 它可以产生Integer、Long、UUID，或String，如下面的例子所示。







Properties

Yaml





```
my.secret=${random.value}
my.number=${random.int}
my.bignumber=${random.long}
my.uuid=${random.uuid}
my.number-less-than-ten=${random.int(10)}
my.number-in-range=${random.int[1024,65536]}

```







`random.int*` 的语法是 `OPEN value (,max) CLOSE`，其中 `OPEN,CLOSE` 是任何字符， `value,max` 是整数。 如果提供了 `max`，那么 `value` 是最小值， `max` 是最大值（独占）。







### [](https://springdoc.cn/spring-boot/features.html#features.external-config.system-environment)2.7\. 配置系统环境属性



Spring Boot支持为环境属性设置一个前缀。 如果系统环境被多个具有不同配置要求的Spring Boot应用程序共享，这就很有用。 系统环境属性的前缀可以直接在 `SpringApplication` 上设置。





例如，如果你将前缀设置为 `input` ，诸如 `remote.timeout` 这样的属性在系统环境中也将被解析为 `input.remote.timeout`。







### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties)2.8\. 类型安全的配置属性



使用 `@Value("${property}")` 注解来注入配置属性有时会很麻烦，特别是当你要处理多个属性或你的数据是分层的。 Spring Boot提供了一种处理属性的替代方法，让强类型的Bean管理和验证你的应用程序的配置。





|  | 另请参见[`@Value` 和类型安全配置属性之间的区别](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.vs-value-annotation)。 |
| --- | --- |





#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.java-bean-binding)2.8.1\. JavaBean 属性绑定



如下面的例子所示，可以绑定一个声明了标准JavaBean属性的bean。







Java

Kotlin





```
@ConfigurationProperties("my.service")
public class MyProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    // getters / setters...

    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));

        // getters / setters...

    }

}

```







前面的POJO定义了以下属性。





*   `my.service.enabled`，默认值为`false`。

*   `my.service.remote-address`，其类型可由`String`强制提供。

*   `my.service.security.username`，有一个嵌套的 `security` 对象，其名称由该属性的名称决定。 特别是，那里完全没有使用类型，可以是 `SecurityProperties`。

*   `my.service.security.password`.

*   `my.service.security.role`，有一个 `String` 的集合，默认为 `USER`。





|  | 映射到Spring Boot中可用的 `@ConfigurationProperties` 类的属性，通过properties文件、YAML文件、环境变量和其他机制进行配置，这些属性是公共API，但类本身的 getters/setters 并不意味着可以直接使用（一句话，Spring也是通过getter/setter这些public方法进行设置值的，你别用）。 |
| --- | --- |





|  | 这样的设计依赖于一个默认的无参构造函数，getter和setter通常是必须的，因为绑定是通过标准的Java Beans property descriptor（Java内省）实现的，就像在Spring MVC中一样。 在以下情况下，可以省略setter。*   Map, 只要它们被初始化，就需要一个getter，但不一定需要一个setter，因为它们可以被绑定器突变。        *   Collection和array 可以通过索引（通常用YAML）或使用单个逗号分隔的值（属性）来访问。 在后一种情况下，一个setter是必须的。 我们建议总是为这类类型添加一个setter。 如果你初始化一个集合，确保它不是不可变的（如前面的例子）。        *   如果嵌套的POJO属性被初始化（就像前面例子中的 `Security` 字段），就不需要setter。 如果你想让绑定器通过使用它的默认构造函数来即时创建实例，你需要一个setter。        有些人使用Project Lombok来自动添加getter和setter。 请确保Lombok不会为这样的类型生成任何特定的构造函数，因为它被容器自动用来实例化对象。最后，只考虑标准的Java Bean属性，不支持对静态属性的绑定。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.constructor-binding)2.8.2\. 构造函数绑定



上一节的例子可以用不可变的方式重写，如下例所示。







Java

Kotlin





```
@ConfigurationProperties("my.service")
public class MyProperties {

    // fields...

    public MyProperties(boolean enabled, InetAddress remoteAddress, Security security) {
        this.enabled = enabled;
        this.remoteAddress = remoteAddress;
        this.security = security;
    }

    // getters...

    public static class Security {

        // fields...

        public Security(String username, String password, @DefaultValue("USER") List<String> roles) {
            this.username = username;
            this.password = password;
            this.roles = roles;
        }

        // getters...

    }

}

```







在这种设置中，唯一的“带参数构造函数”的存在意味着应该使用该构造函数进行绑定。 这意味着绑定器会找到一个带有你希望绑定的参数的构造函数。 如果你的类有多个构造函数，可以使用 `@ConstructorBinding` 注解来指定使用哪个构造函数进行构造函数绑定。 如果要为一个只有一个“带参数构造函数”的类选择不绑定构造函数，该构造函数必须用 `@Autowired` 来注解。 构造函数绑定可以与 `Record` 一起使用。 除非你的记录有多个构造函数，否则没有必要使用 `@ConstructorBinding`。





构造函数绑定类的嵌套成员（如上面例子中的 `Security`）也将通过其构造函数被绑定。





默认值可以在构造函数参数和Record组件上使用 `@DefaultValue` 来指定。 转换服务将被应用于将注解的 `String` 值强制转换为缺失属性的目标类型。





参考前面的例子，如果没有属性绑定到 `Security` ， `MyProperties` 实例将包含一个 `security` 类型的 `null` 值。 为了使它包含一个非 null 的 `Security` 实例，即使没有属性与之绑定（当使用Kotlin时，这将要求 `Security` 的 `username` 和 `password` 参数被声明为 nullable，因为它们没有默认值），使用一个空的 `@DefaultValue` 注解。







Java

Kotlin





```
public MyProperties(boolean enabled, InetAddress remoteAddress, @DefaultValue Security security) {
    this.enabled = enabled;
    this.remoteAddress = remoteAddress;
    this.security = security;
}

```







|  | 要使用构造函数绑定，该类必须使用 `@EnableConfigurationProperties` 或配置属性扫描来启用。 你不能对通过常规Spring机制创建的Bean使用构造函数绑定（例如 `@Component` Bean，通过使用 `@Bean` 方法创建的Bean或通过使用 `@Import` 加载的Bean）。 |
| --- | --- |





|  | 要在原生镜像中使用构造函数绑定，必须用 `-parameters` 参数编译该类。如果你使用 Spring Boot 的 Gradle 插件或使用 Maven 和 `spring-boot-starter-parent`，这将自动配置。 |
| --- | --- |





|  | 不建议将 `java.util.Optional` 与 `@ConfigurationProperties` 一起使用，因为它主要是作为一个返回类型使用。 因此，它并不适合配置属性注入。 为了与其他类型的属性保持一致，如果你确实声明了一个 `Optional` 属性，但它没有值，`null` 而不是一个空的 `Optional` 将被绑定。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.enabling-annotated-types)2.8.3\. 启用 @ConfigurationProperties 类



Spring Boot提供了绑定 `@ConfigurationProperties` 类型并将其注册为Bean的基础设施。 你可以在逐个类的基础上启用配置属性，或者启用配置属性扫描，其工作方式与组件扫描类似。





有时，用 `@ConfigurationProperties` 注解的类可能不适合扫描，例如，如果你正在开发你自己的自动配置或者你想有条件地启用它们。 在这些情况下，使用 `@EnableConfigurationProperties` 注解指定要处理的类型列表， 它可以注解在任何 `@Configuration` 类上，如下面的例子所示。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(SomeProperties.class)
public class MyConfiguration {

}

```









Java

Kotlin





```
@ConfigurationProperties("some.properties")
public class SomeProperties {

}

```







要使用配置属性扫描，请向你的application添加 `@ConfigurationPropertiesScan` 注解。 通常，它被添加到用 `@SpringBootApplication` 注解的main类中，但它也可以被添加到任何 `@Configuration` 类上。 默认情况下，扫描会从注解所在的包开始，你如果想自定义扫描其他包，可以参考如下。







Java

Kotlin





```
@SpringBootApplication
@ConfigurationPropertiesScan({ "com.example.app", "com.example.another" })
public class MyApplication {

}

```







|  | 当 `@ConfigurationProperties` Bean使用配置属性扫描或通过 `@EnableConfigurationProperties` 注册时，该Bean有一个常规名称：`<prefix>-<fqn>`，其中 `<prefix>` 是 `@ConfigurationProperties` 注解中指定的环境键前缀， `<fqn>` 是Bean的完全限定名称。 如果注解没有提供任何前缀，则只使用Bean的完全限定名称。假设它在 `com.example.app` 包中，上面的 `SomeProperties` 例子的 bean 名称是 `some.properties-com.example.app.SomeProperties`。 |
| --- | --- |





我们建议 `@ConfigurationProperties` 只处理 environment，特别是不从上下文注入其他Bean。 对于边角案例（特殊情况），可以使用 setter 注入或框架提供的任何 `*Aware` 接口（如 `EnvironmentAware` ，如果你需要访问 `Environment`）。 如果你仍然想使用构造器注入其他Bean，配置属性Bean必须用 `@Component` 来注解，并使用基于JavaBean的属性绑定。







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.using-annotated-types)2.8.4\. 使用 @ConfigurationProperties 类



这种配置方式与 `SpringApplication` 外部YAML配置配合得特别好，如以下例子所示。







```
my:
  service:
    remote-address: 192.168.1.1
    security:
      username: "admin"
      roles:
      - "USER"
      - "ADMIN"
```







要使用 `@ConfigurationProperties` Bean，你可以用与其他Bean相同的方式注入它们，如下例所示。







Java

Kotlin





```
@Service
public class MyService {

    private final MyProperties properties;

    public MyService(MyProperties properties) {
        this.properties = properties;
    }

    public void openConnection() {
        Server server = new Server(this.properties.getRemoteAddress());
        server.start();
        // ...
    }

    // ...

}

```







|  | 使用 `@ConfigurationProperties` 还可以让你生成元数据文件，这些文件可以被IDE用来配置属性的“自动补全”功能。 详情见[附录](https://springdoc.cn/spring-boot/configuration-metadata.html#appendix.configuration-metadata)。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.third-party-configuration)2.8.5\. 第三方配置



除了使用 `@ConfigurationProperties` 来注解一个类之外，你还可以在公共的 `@Bean` 方法上使用它。 当你想把属性绑定到你控制之外的第三方组件时，这样做特别有用。





要从 `Environment` 属性中配置一个Bean，请在其Bean注册中添加 `@ConfigurationProperties` ，如下例所示。







Java

Kotlin





```
@Configuration(proxyBeanMethods = false)
public class ThirdPartyConfiguration {

    @Bean
    @ConfigurationProperties(prefix = "another")
    public AnotherComponent anotherComponent() {
        return new AnotherComponent();
    }

}

```







任何用 `another` 前缀定义的JavaBean属性都会被映射到 `AnotherComponent` Bean上，其方式类似于前面的 `SomeProperties` 例子。







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding)2.8.6\. 宽松的绑定



Spring Boot在将 `Environment` 属性绑定到 `@ConfigurationProperties` bean时使用了一些宽松的规则，因此 `Environment` 属性名称和bean属性名称之间不需要完全匹配。 这很有用，常见的例子包括破折号分隔的属性名称（例如， `context-path` 绑定到 `contextPath` ），和大写的属性名称（例如，`PORT` 绑定到 `port` ）。





演示一个例子，考虑以下 `@ConfigurationProperties` 类。







Java

Kotlin





```
@ConfigurationProperties(prefix = "my.main-project.person")
public class MyPersonProperties {

    private String firstName;

    public String getFirstName() {
        return this.firstName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
    }

}

```







对以上的代码来说，以下的属性名称都可以使用。



<caption>Table 3\. relaxed binding</caption><colgroup><col><col></colgroup>
| Property | Note |
| --- | --- |
| `my.main-project.person.first-name` | Kebab 风格（短横线隔开），建议在 `.properties` 和 `.yml` 文件中使用。 |
| `my.main-project.person.firstName` | 标准的驼峰语法。 |
| `my.main-project.person.first_name` | 下划线，这是一种用于 `.properties` 和 `.yml` 文件的替代格式。 |
| `MY_MAINPROJECT_PERSON_FIRSTNAME` | 大写格式，在使用系统环境变量时建议使用大写格式。 |



|  | 注解的 `prefix` 值 _必须_ 是kebab风格（小写并以 `-` 分隔，如 `my.main-project.person` ）。 |
| --- | --- |



<caption>Table 4\. 每种属性源的宽松绑定规则</caption><colgroup><col><col><col></colgroup>
| 属性源 | 简单的 | 列表 |
| --- | --- | --- |
| Properties 文件 | 驼峰, kebab , 下划线 | 使用 `[ ]` 或逗号分隔值的标准列表语法 |
| YAML 文件 | 驼峰, kebab , 下划线 | 标准YAML列表语法或逗号分隔的值 |
| 环境变量 | 大写，下划线为分隔符(见 [从环境变量绑定](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables)). | Numeric values surrounded by underscores (see [从环境变量绑定](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables)) |
| 系统属性（System properties） | 驼峰, kebab , 下划线 | 使用 `[ ]` 或逗号分隔值的标准列表语法 |



|  | 我们建议，在可能的情况下，属性应以小写的kebab格式存储，例如 `my.person.first-name=Rod` 。 |
| --- | --- |





##### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding.maps)绑定Map



当绑定到 `Map` 属性时，你可能需要使用一个特殊的括号符号，以便保留原始的 `key` 值。 如果key没有被 `[ ]` 包裹，任何非字母数字、`-` 或 `.` 的字符将被删除。





例如，考虑将以下属性绑定到一个 `Map<String,String>`。







Properties

Yaml





```
my.map.[/key1]=value1
my.map.[/key2]=value2
my.map./key3=value3
```







|  | 对于YAML文件，括号需要用引号包裹，以使key被正确解析。 |
| --- | --- |





上面的属性将绑定到一个 `Map` ，`/key1`，`/key2` 和 `key3` 作为map的key。 斜线已经从 `key3` 中删除，因为它没有被方括号包裹。





当绑定到标量值时，带有 `.` 的键不需要用 `[]` 包裹。 标量值包括枚举和所有 `java.lang` 包中的类型，除了 `Object` 。 将 `a.b=c` 绑定到 `Map<String, String>` 将保留键中的 `.` ，并返回一个带有 `{"a.b"="c"}` Entry的Map。 对于任何其他类型，如果你的 `key` 包含 `.` ，你需要使用括号符号。 例如，将 `a.b=c` 绑定到 `Map<String, Object>` 将返回一个带有 `{"a"={"b"="c"}` entry的Map，而 `[a.b]=c` 将返回一个带有 `{"a.b"="c"}` entry 的Map。







##### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding.environment-variables)从环境变量绑定



例如，Linux shell变量只能包含字母（`a` 到 `z` 或 `A` 到 `Z` ）、数字（ `0` 到 `9` ）或下划线字符（ `_` ）。 按照惯例，Unix shell变量的名称也将采用大写字母。





Spring Boot宽松的绑定规则被设计为尽可能地与这些命名限制兼容。





要将规范形式的属性名称转换为环境变量名称，你可以遵循这些规则。





*   用下划线（`_`）替换点（`.`）。

*   删除任何破折号（`-`）。

*   转换为大写字母。





例如，配置属性 `spring.main.log-startup-info` 将是一个名为 `SPRING_MAIN_LOGSTARTUPINFO` 的环境变量。





环境变量也可以在绑定到对象列表（List）时使用。 要绑定到一个 `List`，在变量名称中，元素编号（索引）应该用下划线包裹。





例如，配置属性 `my.service[0].other` 将使用一个名为 `MY_SERVICE_0_OTHER` 的环境变量。









#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.merging-complex-types)2.8.7\. 合并复杂的类型



当List被配置在多个地方时，覆盖的作用是替换整个list。





例如，假设一个 `MyPojo` 对象的 `name` 和 `description` 属性默认为 `null`。 下面的例子从 `MyProperties` 中暴露了一个 `MyPojo` 对象的列表。







Java

Kotlin





```
@ConfigurationProperties("my")
public class MyProperties {

    private final List<MyPojo> list = new ArrayList<>();

    public List<MyPojo> getList() {
        return this.list;
    }

}

```







考虑以下配置。







Properties

Yaml





```
my.list[0].name=my name
my.list[0].description=my description
#---
spring.config.activate.on-profile=dev
my.list[0].name=my another name

```







如果 `dev` 配置文件未被激活，`MyProperties.list` 包含一个 `MyPojo` 条目，如之前定义的那样。 然而，如果 `dev` 配置文件被激活，`list` 仍然只包含一个条目（name 为 `my another name`，description为 `null`）。 这种配置不会在列表中添加第二个 `MyPojo` 实例，也不会合并项目。





当一个 `List` 在多个配置文件中被指定时，将使用具有最高优先级的那个（并且只有那个）。 考虑下面的例子。







Properties

Yaml





```
my.list[0].name=my name
my.list[0].description=my description
my.list[1].name=another name
my.list[1].description=another description
#---
spring.config.activate.on-profile=dev
my.list[0].name=my another name

```







在前面的例子中，如果 `dev` 配置文件是激活的，`MyProperties.list` 包含 _一个_ `MyPojo` 条目（name 是 `my another name`，description是 `null`）。 对于YAML，逗号分隔的列表和YAML列表都可以用来完全覆盖列表的内容。





对于 `Map` 属性，你可以用从多个来源获取的属性值进行绑定。 然而，对于多个来源中的同一属性，使用具有最高优先级的那个。 下面的例子从 `MyProperties` 暴露了一个 `Map<String, MyPojo>`。







Java

Kotlin





```
@ConfigurationProperties("my")
public class MyProperties {

    private final Map<String, MyPojo> map = new LinkedHashMap<>();

    public Map<String, MyPojo> getMap() {
        return this.map;
    }

}

```







考虑以下配置。







Properties

Yaml





```
my.map.key1.name=my name 1
my.map.key1.description=my description 1
#---
spring.config.activate.on-profile=dev
my.map.key1.name=dev name 1
my.map.key2.name=dev name 2
my.map.key2.description=dev description 2

```







如果 `dev` 配置文件没有激活，`MyProperties.map` 包含一个key为 `key1` 的条目（name为 `my name 1` ，description为 `my description 1` ）。 然而，如果 `dev` 配置文件被激活，`map` 包含两个条目，key为 `key1` （name为 `dev name 1`，description为 `my description 1` ）和 `key2`（name为 `dev name 2`，description为 `dev description 2`）。





|  | 前面的合并规则适用于所有属性源的属性，而不仅仅是文件。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.conversion)2.8.8\. 属性（Properties）转换



当Spring Boot与 `@ConfigurationProperties` Bean绑定时，它试图将外部application properties强制改为正确的类型。 如果你需要自定义类型转换，你可以提供一个 `ConversionService` bean（Bean的名称为 `conversionService` ）或自定义属性编辑器（通过 `CustomEditorConfigurer` bean）或自定义 `Converters` Bean（使用 `@ConfigurationPropertiesBinding` 注解）。





|  | 由于这个Bean是在应用程序生命周期的早期被请求的，请确保限制你的 `ConversionService` 所使用的依赖关系。 通常情况下，你所需要的任何依赖关系在创建时可能没有完全初始化。 如果你的自定义 `ConversionService` 不需要配置keys coercion，你可能想重命名它，并且只依赖用 `@ConfigurationPropertiesBinding` 限定的自定义转换器。 |
| --- | --- |





##### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.conversion.durations)转换为 Duration



Spring Boot对表达持续时间有专门的支持。 如果你公开了一个 `java.time.Duration` 属性，application properties中的以下格式就可用。





*   普通的 `long` （使用毫秒作为默认单位，除非指定了 `@DurationUnit` ）。

*   标准的ISO-8601格式 [由 `java.time.Duration` 使用](https://docs.oracle.com/javase/17/docs/api/java/time/Duration.html#parse-java.lang.CharSequence-)。

*   一个更易读的格式，其中值和单位是耦合的（`10s` 表示10秒）。





请考虑以下例子。







Java

Kotlin





```
@ConfigurationProperties("my")
public class MyProperties {

    @DurationUnit(ChronoUnit.SECONDS)
    private Duration sessionTimeout = Duration.ofSeconds(30);

    private Duration readTimeout = Duration.ofMillis(1000);

    // getters / setters...

}

```







要指定一个30秒的会话超时， `30` 、 `PT30S` 和 `30s` 都是等价的。 读取超时为500ms，可以用以下任何一种形式指定。 `500`, `PT0.5S` 和 `500ms`.





你也可以使用如下支持的时间单位。





*   `ns` 纳秒

*   `us` 微秒

*   `ms` 毫秒

*   `s` 秒

*   `m` 分

*   `h` 小时

*   `d` 天





默认单位是毫秒，可以使用 `@DurationUnit` 来重写，如上面的例子所示。





如果你喜欢使用构造函数绑定，同样的属性可以被暴露出来，如下面的例子所示。







Java

Kotlin





```
@ConfigurationProperties("my")
public class MyProperties {

    // fields...

    public MyProperties(@DurationUnit(ChronoUnit.SECONDS) @DefaultValue("30s") Duration sessionTimeout,
            @DefaultValue("1000ms") Duration readTimeout) {
        this.sessionTimeout = sessionTimeout;
        this.readTimeout = readTimeout;
    }

    // getters...

}

```







|  | 如果你要升级一个 `Long` 的属性，如果它不是毫秒，请确保定义单位（使用 `@DurationUnit` ）。 这样做提供了一个透明的升级路径，同时支持更丰富的格式 |
| --- | --- |







##### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.conversion.periods)转换为期间（Period）



除了duration，Spring Boot还可以使用 `java.time.Period` 类型。 以下格式可以在application properties中使用。





*   一个常规的 `int` 表示法（使用天作为默认单位，除非指定了 `@PeriodUnit` ）。

*   标准的ISO-8601格式 [由 `java.time.Period` 使用](https://docs.oracle.com/javase/17/docs/api/java/time/Period.html#parse-java.lang.CharSequence-)。

*   一个更简单的格式，其中值和单位对是耦合的（ `1y3d` 表示1年3天）。





支持下列简单的单位格式。





*   `y` 年

*   `m` 月

*   `w` 周

*   `d` 日





|  | `java.time.Period` 类型实际上从未存储过周数，它是一个快捷方式，意味着 “7天”。 |
| --- | --- |







##### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.conversion.data-sizes)转换为数据大小（Data Sizes）



Spring Framework有一个 `DataSize` 值类型，以字节为单位表达大小。 如果你公开了一个 `DataSize` 属性，application properties中的以下格式就可用。





*   一个常规的 `long` 表示（使用字节作为默认单位，除非指定了 `@DataSizeUnit`）。

*   一个更易读的格式，其中值和单位是耦合的（`10MB` 意味着10兆字节）。





考虑以下例子。







Java

Kotlin





```
@ConfigurationProperties("my")
public class MyProperties {

    @DataSizeUnit(DataUnit.MEGABYTES)
    private DataSize bufferSize = DataSize.ofMegabytes(2);

    private DataSize sizeThreshold = DataSize.ofBytes(512);

    // getters/setters...

}

```







要指定一个10兆字节（Mb）的缓冲区大小， `10` 和 `10MB` 是等价的。 256字节的大小阈值可以指定为 `256` 或 `256B`。





你也可以使用如下这些支持的单位。





*   `B` 字节

*   `KB` KB

*   `MB` MB

*   `GB` GB

*   `TB` TB





默认单位是字节，可以使用 `@DataSizeUnit` 来重写，如上面的例子所示。





如果你喜欢使用构造函数绑定，同样的属性可以被暴露出来，如下面的例子所示。







Java

Kotlin





```
@ConfigurationProperties("my")
public class MyProperties {

    // fields...

    public MyProperties(@DataSizeUnit(DataUnit.MEGABYTES) @DefaultValue("2MB") DataSize bufferSize,
            @DefaultValue("512B") DataSize sizeThreshold) {
        this.bufferSize = bufferSize;
        this.sizeThreshold = sizeThreshold;
    }

    // getters...

}

```







|  | 如果你正在升级一个 `Long` 属性，确保定义单位（使用 `@DataSizeUnit`），如果它不是字节。 这样做提供了一个透明的升级路径，同时支持更丰富的格式。 |
| --- | --- |









#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.validation)2.8.9\. @ConfigurationProperties 校验



只要使用Spring的 `@Validated` 注解，Spring Boot就会尝试验证 `@ConfigurationProperties` 类。 你可以直接在你的配置类上使用JSR-303的 `jakarta.validation` 约束注解。 要做到这一点，请确保你的classpath上有一个兼容的JSR-303实现，然后将约束注解添加到你的字段中，如下面的例子所示。







Java

Kotlin





```
@ConfigurationProperties("my.service")
@Validated
public class MyProperties {

    @NotNull
    private InetAddress remoteAddress;

    // getters/setters...

}

```







|  | 你也可以通过在 configuration properties 的 `@Bean` 方法上注解 `@Validated` 来触发验证。 |
| --- | --- |





为了确保总是为嵌套的属性触发验证，即使没有找到属性，相关的字段必须用 `@Valid` 来注释。 下面的例子建立在前面的 `MyProperties` 的基础上。







Java

Kotlin





```
@ConfigurationProperties("my.service")
@Validated
public class MyProperties {

    @NotNull
    private InetAddress remoteAddress;

    @Valid
    private final Security security = new Security();

    // getters/setters...

    public static class Security {

        @NotEmpty
        private String username;

        // getters/setters...

    }

}

```







你也可以通过创建一个名为 `configurationPropertiesValidator` 的bean定义来添加一个自定义的Spring `Validator`。 `@Bean` 方法应该被声明为 `static`。 配置属性验证器是在应用程序生命周期的早期创建的，将 `@Bean` 方法声明为静态，可以让Bean的创建不需要实例化 `@Configuration` 类。 这样做可以避免过早实例化可能引起的任何问题。





|  | `spring-boot-actuator` 模块包括一个暴露所有 `@ConfigurationProperties` Bean 的端点。 你可以通过浏览器访问 `/actuator/configprops` 或使用相应的JMX端点。 详情见"[生产就绪](https://springdoc.cn/spring-boot/actuator.html#actuator.endpoints)"部分。 |
| --- | --- |







#### [](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.vs-value-annotation)2.8.10\. @ConfigurationProperties vs. @Value



`@Value` 注解是一个核心的容器功能，它不提供与类型安全的配置属性相同的功能。 下表总结了 `@ConfigurationProperties` 和 `@Value` 所支持的功能。



<colgroup><col><col><col></colgroup>
| 功能 | `@ConfigurationProperties` | `@Value` |
| --- | --- | --- |
| [宽松绑定](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding) | Yes | 有限制 (见 [下文注释](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.vs-value-annotation.note)) |
| [支持 Meta-data](https://springdoc.cn/spring-boot/configuration-metadata.html#appendix.configuration-metadata) | Yes | No |
| `SpEL` 表达式 | No | Yes |



|  | 如果你确实想使用 `@Value`，我们建议你使用属性名称的规范形式（仅使用小写字母的kebab-case）来引用属性名称。 这将允许Spring Boot使用与 [宽松绑定](https://springdoc.cn/spring-boot/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding) `@ConfigurationProperties` 时相同的逻辑。例如，`@Value("${demo.item-price}")` 将从 `application.properties` 文件中获取 `demo.item-price` 和 `demo.itemPrice` 形式，以及从系统环境中获取 `DEMO_ITEMPRICE`。 如果你用 `@Value("${demo.itemPrice}")` 代替，`demo.item-price` 和 `DEMO_ITEMPRICE` 将不会被考虑。 |
| --- | --- |





如果你为你自己的组件定义了一组配置键，我们建议你将它们分组在一个用 `@ConfigurationProperties` 注解的POJO中。 这样做将为你提供结构化的、类型安全的对象，你可以将其注入到你自己的bean中。





来自应用[application property](https://springdoc.cn/spring-boot/features.html#features.external-config.files) 文件的 `SpEL` 表达式在解析这些文件和填充environment时不会被处理。 然而，可以在 `@Value` 中写一个 `SpEL` 表达式。 如果来自应用程序属性文件的属性值是一个 `SpEL` 表达式，它将在被 `@Value` 消费时被解析。













## [](https://springdoc.cn/spring-boot/features.html#features.profiles)

