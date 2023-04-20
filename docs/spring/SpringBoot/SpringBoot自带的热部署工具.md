# SpringBoot入门 - 配置热部署devtools工具

> 在SpringBoot开发调试中，如果我每行代码的修改都需要重启启动再调试，可能比较费时间；SpringBoot团队针对此问题提供了spring-boot-devtools（简称devtools）插件，它试图提升开发调试的效率。@pdai

*   [SpringBoot入门 - 配置热部署devtools工具](#springboot%E5%85%A5%E9%97%A8---%E9%85%8D%E7%BD%AE%E7%83%AD%E9%83%A8%E7%BD%B2devtools%E5%B7%A5%E5%85%B7)
    *   [准备知识点](#%E5%87%86%E5%A4%87%E7%9F%A5%E8%AF%86%E7%82%B9)
        *   [什么是热部署和热加载？](#%E4%BB%80%E4%B9%88%E6%98%AF%E7%83%AD%E9%83%A8%E7%BD%B2%E5%92%8C%E7%83%AD%E5%8A%A0%E8%BD%BD)
        *   [什么是LiveLoad？](#%E4%BB%80%E4%B9%88%E6%98%AFliveload)
    *   [配置devtools实现热部署](#%E9%85%8D%E7%BD%AEdevtools%E5%AE%9E%E7%8E%B0%E7%83%AD%E9%83%A8%E7%BD%B2)
        *   [POM配置](#pom%E9%85%8D%E7%BD%AE)
        *   [IDEA配置](#idea%E9%85%8D%E7%BD%AE)
        *   [application.yml配置](#applicationyml%E9%85%8D%E7%BD%AE)
        *   [使用LiveLoad](#%E4%BD%BF%E7%94%A8liveload)
    *   [进一步理解](#%E8%BF%9B%E4%B8%80%E6%AD%A5%E7%90%86%E8%A7%A3)
        *   [devtool的原理？为何会自动重启？](#devtool%E7%9A%84%E5%8E%9F%E7%90%86%E4%B8%BA%E4%BD%95%E4%BC%9A%E8%87%AA%E5%8A%A8%E9%87%8D%E5%90%AF)
        *   [devtool是否会被打包进Jar？](#devtool%E6%98%AF%E5%90%A6%E4%BC%9A%E8%A2%AB%E6%89%93%E5%8C%85%E8%BF%9Bjar)
        *   [devtool为何会默认禁用缓存选项？](#devtool%E4%B8%BA%E4%BD%95%E4%BC%9A%E9%BB%98%E8%AE%A4%E7%A6%81%E7%94%A8%E7%BC%93%E5%AD%98%E9%80%89%E9%A1%B9)
        *   [devtool是否可以给所有Springboot应用做全局的配置？](#devtool%E6%98%AF%E5%90%A6%E5%8F%AF%E4%BB%A5%E7%BB%99%E6%89%80%E6%9C%89springboot%E5%BA%94%E7%94%A8%E5%81%9A%E5%85%A8%E5%B1%80%E7%9A%84%E9%85%8D%E7%BD%AE)
        *   [如果我不用devtool，还有什么选择？](#%E5%A6%82%E6%9E%9C%E6%88%91%E4%B8%8D%E7%94%A8devtool%E8%BF%98%E6%9C%89%E4%BB%80%E4%B9%88%E9%80%89%E6%8B%A9)
    *   [示例源码](#%E7%A4%BA%E4%BE%8B%E6%BA%90%E7%A0%81)
    *   [参考文章](#%E5%8F%82%E8%80%83%E6%96%87%E7%AB%A0)

## [#](#准备知识点) 准备知识点

### [#](#什么是热部署和热加载) 什么是热部署和热加载？

> 热部署和热加载是在应用正在运行的时候，自动更新（重新加载或者替换class等）应用的一种能力。（PS：spring-boot-devtools提供的方案也是要重启的，只是无需手动重启能实现自动加载而已。）

严格意义上，我们需要区分下热部署和热加载, 对于Java项目而言：

*   **热部署**

    *   在服务器运行时重新部署项目
    *   它是直接重新加载整个应用，这种方式会释放内存，比热加载更加干净彻底，但同时也更费时间。
*   **热加载**

    *   在在运行时重新加载class，从而升级应用。
    *   热加载的实现原理主要依赖[java的类加载机制](/md/java/jvm/java-jvm-classload.html)，在实现方式可以概括为在容器启动的时候起一条后台线程，定时的检测类文件的时间戳变化，如果类的时间戳变掉了，则将类重新载入。
    *   对比反射机制，反射是在运行时获取类信息，通过动态的调用来改变程序行为； 热加载则是在运行时通过重新加载改变类信息，直接改变程序行为。

### [#](#什么是liveload) 什么是LiveLoad？

LiveLoad是提供浏览器客户端自动加载更新的工具，分为LiveLoad服务器和Liveload浏览器插件两部分； devtools中已经集成了LiveLoad服务器，所以如果我们开发的是web应用，并且期望浏览器自动刷新， 这时候可以考虑LiveLoad.

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416205152.png)

同一时间只能运行一个LiveReload服务器。 开始应用程序之前，请确保没有其他LiveReload服务器正在运行。如果从IDE启动多个应用程序，则只有第一个应用程序将支持LiveReload。

## [#](#配置devtools实现热部署) 配置devtools实现热部署

> 我们通过如下配置来实现自动重启方式的热部署。

### [#](#pom配置) POM配置

添加spring-boot-devtools的依赖



```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        spring-boot-devtools
        <optional>true</optional> <!-- 可以防止将devtools依赖传递到其他模块中 -->
    </dependency>
</dependencies>

```



### [#](#idea配置) IDEA配置

> 如果你使用IDEA开发工具，通常有如下两种方式：

*   方式一： **无任何配置时，手动触发重启更新（Ctrl+F9）**

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416205320.png)
（也可以用`mvn compile`编译触发重启更新）

*   方式二： **IDEA需开启运行时编译，自动重启更新**

**设置1**：

File->Setting->Build,Execution,Deployment->Compile

勾选：Make project automatically

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416205335.png)

**设置2**：

快捷键：ctrl+alt+shift+/

选择：Registry

勾选：compiler.automake.allow.when.app.running

新版本的IDEA可以在File->setting->Advanced Setttings里面的第一个设置：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416205405.png)
### [#](#application-yml配置) application.yml配置



```
spring:
  devtools:
    restart:
      enabled: true  #设置开启热部署
      additional-paths: src/main/java #重启目录
      exclude: WEB-INF/**
  thymeleaf:
    cache: false #使用Thymeleaf模板引擎，关闭缓存

```



### [#](#使用liveload) 使用LiveLoad

spring-boot-devtools模块包含**嵌入式LiveReload服务器**，可以在资源更改时用于触发浏览器刷新。 LiveReload浏览器扩展程序支持Chrome，Firefox和Safari，你可以从livereload.com免费下载。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416205417.png)
或者从浏览器插件中心下载，比如firefox:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416205426.png)

安装完之后，可以通过如下图标管理

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416205442.png)

如果你不想在应用程序运行时启动LiveReload服务器，则可以将spring.devtools.livereload.enabled属性设置为false 。

同一时间只能运行一个LiveReload服务器。 开始应用程序之前，请确保没有其他LiveReload服务器正在运行。如果从IDE启动多个应用程序，则只有第一个应用程序将支持LiveReload。

## [#](#进一步理解) 进一步理解

> 虽然一些开发者会使用devtool工具，但是很少有能够深入理解的；让我们理解如下几个问题，帮助你进一步理解。@pdai

### [#](#devtool的原理-为何会自动重启) devtool的原理？为何会自动重启？

> 为什么同样是重启应用，为什么不手动重启，而是建议使用spring-boot-devtools进行热部署重启？

spring-boot-devtools使用了两个类加载器ClassLoader，一个ClassLoader加载不会发生更改的类（第三方jar包），另一个ClassLoader（restart ClassLoader）加载会更改的类（自定义的类）。

后台启动一个**文件监听线程（File Watcher）**，**监测的目录中的文件发生变动时， 原来的restart ClassLoader被丢弃，将会重新加载新的restart ClassLoader**。

因为文件变动后，第三方jar包不再重新加载，只加载自定义的类，加载的类比较少，所以重启比较快。

这也是为什么，同样是重启应用，为什么不手动重启，建议使用spring-boot-devtools进行热部署重启。

在自动重启中有几点需要注意:

*   **自动重启会记录日志的**

（记录在什么情况下重启的日志）

可以通过如下关闭



```
spring:
  devtools:
    restart:
      log-condition-evaluation-delta: false

```



*   **排除一些不需要自动重启的资源**

某些资源在更改时不一定需要触发重新启动。默认情况下，改变资源/META-INF/maven，/META-INF/resources，/resources，/static，/public，或/templates不触发重新启动，但确会触发现场重装。如果要自定义这些排除项，可以使用该spring.devtools.restart.exclude属性。例如，要仅排除/static，/public你将设置以下属性：



```
spring:
  devtools:
    restart:
      exclude: "static/**,public/**"

```



如果要保留这些默认值并添加其他排除项，请改用该spring.devtools.restart.additional-exclude属性。

*   **自定义重启类加载器**

重启功能是通过使用两个类加载器来实现的。对于大多数应用程序，这种方法效果很好。但是，它有时会导致类加载问题。

默认情况下，IDE 中的任何打开项目都使用“重启”类加载器加载，任何常规.jar文件都使用“基本”类加载器加载。如果你处理一个多模块项目，并且不是每个模块都导入到你的 IDE 中，你可能需要自定义一些东西。为此，你可以创建一个META-INF/spring-devtools.properties文件。

该spring-devtools.properties文件可以包含以restart.exclude和为前缀的属性restart.include。该include元素是应该被拉高到“重启”的类加载器的项目，以及exclude要素是应该向下推入“Base”类加载器的项目。该属性的值是应用于类路径的正则表达式模式，如以下示例所示：



```
restart:
  exclude:
    companycommonlibs: "/mycorp-common-[\\w\\d-\\.]+\\.jar"
  include:
    projectcommon: "/mycorp-myproj-[\\w\\d-\\.]+\\.jar"

```



更多相关的信息可以在[这里在新窗口打开](https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools)查看。

### [#](#devtool是否会被打包进jar) devtool是否会被打包进Jar？

> devtool原则上来说应该是只在开发调试的时候使用，而在生产环境运行jar包时是不需要的，所以Spring打包会不会把它打进JAR吗？

*   **默认情况下，不会被打包进JAR**

运行打包的应用程序时，开发人员工具**会自动禁用**。如果你通过 java -jar或者其他特殊的类加载器进行启动时，都会被认为是“生产环境的应用”。

*   **如果我们期望远程调试应用**

（_生产环境勿用，只有在受信任的网络上运行或使用 SSL 进行保护时，才应启用它_）

在这种情况下，devtool也具备远程调试的能力：远程客户端应用程序旨在从你的 IDE 中运行。你需要org.springframework.boot.devtools.RemoteSpringApplication使用与你连接的远程项目相同的类路径运行。应用程序的唯一必需参数是它连接到的远程 URL。

例如，如果使用 Eclipse 或 Spring Tools，并且你有一个my-app已部署到 Cloud Foundry 的名为的项目，执行以下操作：

1.  选择Run Configurations…?从Run菜单。
2.  创建一个新的Java Application“启动配置”。
3.  浏览my-app项目。
4.  使用org.springframework.boot.devtools.RemoteSpringApplication作为主类。
5.  添加https://myapp.cfapps.io到Program arguments（或任何你的远程 URL）。

正在运行的远程客户端可能类似于以下列表：



```
  .   ____          _                                              __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _          ___               _      \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` |        | _ \___ _ __  ___| |_ ___ \ \ \ \
 \\/  ___)| |_)| | | | | || (_| []::::::[]   / -_) '  \/ _ \  _/ -_) ) ) ) )
  '  |____| .__|_| |_|_| |_\__, |        |_|_\___|_|_|_\___/\__\___|/ / / /
 =========|_|==============|___/===================================/_/_/_/
 :: Spring Boot Remote :: 2.5.4

2015-06-10 18:25:06.632  INFO 14938 --- [           main] o.s.b.devtools.RemoteSpringApplication   : Starting RemoteSpringApplication on pwmbp with PID 14938 (/Users/pwebb/projects/spring-boot/code/spring-boot-project/spring-boot-devtools/target/classes started by pwebb in /Users/pwebb/projects/spring-boot/code)
2015-06-10 18:25:06.671  INFO 14938 --- [           main] s.c.a.AnnotationConfigApplicationContext : Refreshing org.springframework.context.annotation.AnnotationConfigApplicationContext@2a17b7b6: startup date [Wed Jun 10 18:25:06 PDT 2015]; root of context hierarchy
2015-06-10 18:25:07.043  WARN 14938 --- [           main] o.s.b.d.r.c.RemoteClientConfiguration    : The connection to http://localhost:8080 is insecure. You should use a URL starting with 'https://'.
2015-06-10 18:25:07.074  INFO 14938 --- [           main] o.s.b.d.a.OptionalLiveReloadServer       : LiveReload server is running on port 35729
2015-06-10 18:25:07.130  INFO 14938 --- [           main] o.s.b.devtools.RemoteSpringApplication   : Started RemoteSpringApplication in 0.74 seconds (JVM running for 1.105)

```



### [#](#devtool为何会默认禁用缓存选项) devtool为何会默认禁用缓存选项？

> Spring Boot 支持的一些库**使用缓存来提高性能**。例如，模板引擎缓存已编译的模板以避免重复解析模板文件。此外，Spring MVC 可以在提供静态资源时向响应添加 HTTP 缓存标头。

虽然缓存**在生产中非常有益，但在开发过程中可能会适得其反**，使你无法看到刚刚在应用程序中所做的更改。出于这个原因， spring-boot-devtools 默认禁用缓存选项。

比如Thymeleaf 提供了spring.thymeleaf.cache来设置模板引擎的缓存，使用spring-boot-devtools模块时是不需要手动设置这些属性的，因为spring-boot-devtools会自动进行设置。

那么会自动设置哪些配置呢？你可以在DevToolsPropertyDefaultsPostProcessor类找到对应的默认配置。



```
public class DevToolsPropertyDefaultsPostProcessor implements EnvironmentPostProcessor {

	static {
		Map<String, Object> properties = new HashMap<>();
		properties.put("spring.thymeleaf.cache", "false");
		properties.put("spring.freemarker.cache", "false");
		properties.put("spring.groovy.template.cache", "false");
		properties.put("spring.mustache.cache", "false");
		properties.put("server.servlet.session.persistent", "true");
		properties.put("spring.h2.console.enabled", "true");
		properties.put("spring.web.resources.cache.period", "0");
		properties.put("spring.web.resources.chain.cache", "false");
		properties.put("spring.template.provider.cache", "false");
		properties.put("spring.mvc.log-resolved-exception", "true");
		properties.put("server.error.include-binding-errors", "ALWAYS");
		properties.put("server.error.include-message", "ALWAYS");
		properties.put("server.error.include-stacktrace", "ALWAYS");
		properties.put("server.servlet.jsp.init-parameters.development", "true");
		properties.put("spring.reactor.debug", "true");
		PROPERTIES = Collections.unmodifiableMap(properties);
	}

```



当然如果你不想被应用属性被spring-boot-devtools默认设置， 可以通过spring.devtools.add-properties到false你application.yml中。

### [#](#devtool是否可以给所有springboot应用做全局的配置) devtool是否可以给所有Springboot应用做全局的配置？

> 可以通过将spring-boot-devtools.yml文件添加到$HOME/.config/spring-boot目录来**配置全局 devtools 设置**。

添加到这些文件的任何属性都适用于你机器上使用 devtools 的所有Spring Boot 应用程序。例如，要将重新启动配置为始终使用触发器文件，你需要将以下属性添加到你的spring-boot-devtools文件中：



```
spring:
  devtools:
    restart:
      trigger-file: ".reloadtrigger"

```



### [#](#如果我不用devtool-还有什么选择) 如果我不用devtool，还有什么选择？

> 如果我不用devtool，还有什么选择？

**在实际的开发过程中，我也不会去使用devtool工具**, 因为：

*   devtool本身基于重启方式，这种仍然不是真正的热替换方案，JRebel才是（它是收费的）
*   开发调试最重要的还是一种权衡
    *   自动重启的开销如果和手动重启没有什么太大差别，那么还不如手动重启（按需重启）
    *   多数情况下，如果是**方法内部的修改或者静态资源的修改**，在IDEA中是可以通过Rebuild（Ctrl + Shift + F9）进行热更的

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230416211745.png)

*   此外还有一个工具spring loaded， 可实现修改类文件的热部署，具体可看其[github地址在新窗口打开](https://github.com/spring-projects/spring-loaded)上的说明。

## [#](#示例源码) 示例源码

https://github.com/realpdai/tech-pdai-spring-demos

## [#](#参考文章) 参考文章

https://docs.spring.io/spring-boot/docs/current/reference/html/using.html#using.devtools

https://liayun.blog.csdn.net/article/details/116541775

* * *

著作权归@pdai所有 原文链接：https://pdai.tech/md/spring/springboot/springboot-x-hello-devtool.html