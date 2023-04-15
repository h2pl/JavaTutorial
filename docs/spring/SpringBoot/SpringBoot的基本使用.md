# 快速构建SpringBoot应用

这个教程来自于springboot官网，足够权威，也足够简单。

##  第一个 hello world 程序

您将构建一个经典的“Hello World！”任何浏览器都可以连接到的端点。你甚至可以告诉它你的名字，它会以更友好的方式回应。

## 基本环境要求

1、首先你最好有一个趁手的IDE,热门选择包括 IntelliJ IDEA、Spring Tools、Visual Studio Code 或 Eclipse 等等。

2、JDK，至于版本的话，8-17都是不错的选择

3、当然这里还需要导入maven的pom依赖，所以我们也需要maven，maven插件在idea自带了。我们会在接下来的部分进行介绍，需要引入哪些依赖

## 第一步：启动一个新的Spring Boot项目

使用[start.spring.io](http://start.spring.io/)创建一个“web”项目。在“dependencies”对话框中搜索并添加“web”依赖项，如屏幕截图所示。

点击“生成”按钮，下载 zip 文件，并将其解压到您计算机上的一个文件夹中。

![Start.spring.io 快速入门](https://spring.io/img/extra/quickstart-1.png)

[start.spring.io](http://start.spring.io/)创建的项目包含[Spring Boot](https://spring.io/projects/spring-boot)，一个让Spring准备好在您的应用程序内部工作，但不需要太多代码或配置。 Spring Boot 是启动 Spring 项目的最快和最流行的方式。

这里我们选择使用maven作为包管理工具，下载下来的项目里会包含pom文件，pom文件里包含了你的添加的依赖。

````
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.0.5</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
````


## 第二步：添加你的代码

在您的 IDE 中打开项目并在 `src/main/java/com/example/demo` 文件夹中找到 `DemoApplication.java` 文件。

现在通过添加下面代码中显示的额外方法和注释来更改文件的内容。您可以复制并粘贴代码或直接键入代码。
```
package com.example.demo;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
@RestController
public class DemoApplication {
    public static void main(String[] args) {
      SpringApplication.run(DemoApplication.class, args);
    }
    @GetMapping("/hello")
    public String hello(@RequestParam(value = "name", defaultValue = "World") String name) {
      return String.format("Hello %s!", name);
    }
}

```

这是在 Spring Boot 中创建一个简单的“Hello World” Web 服务所需的所有代码。

我们添加的`hello()`方法旨在获取一个名为name的String参数，然后将此参数与代码中的单词`"Hello"`结合起来。

这意味着如果您在请求中将您的名字设置为“Amy”，则响应将是“Hello Amy”。

`@RestController` 注释告诉 Spring，此代码描述了一个端点，该端点应在 Web 上可用。 

@GetMapping(“/hello”) 告诉 Spring 使用我们的 hello() 方法来响应发送到 http://localhost:8080/hello 地址的请求。

最后，@RequestParam 告诉 Spring 在请求中期望一个名称值，但如果不存在，它将默认使用单词“World”。

## 第三步：启动

让我们构建并运行程序。打开命令行（或终端）并导航到您拥有项目文件的文件夹。

我们可以通过发出以下命令来构建和运行应用程序：

**MacOS/Linux:**

```
COPY./gradlew bootRun

```

**Windows:**

```
COPY.\gradlew.bat bootRun

```

您应该会看到一些与此非常相似的输出：
![Quick Start On Start.spring.io](https://spring.io/img/extra/quickstart-2.png)

这里的最后几行告诉我们，Spring应用已经开始运行了。 
Spring Boot 的嵌入式 Apache Tomcat 服务器充当网络服务器，并正在监听“localhost”端口“8080”上的请求。

打开浏览器，在顶部的地址栏中输入`http://localhost:8080/hello`。

你应该得到这样一个友好的回应：
![Quick Start On Start.spring.io](https://spring.io/img/extra/quickstart-3.png)

# 总结
就是如此简单，第一个SpringBoot应用就这么构建完成了，你不需要额外的配置文件，额外的依赖，甚至是额外的服务器。
只需要一个启动类，就可以实现一个最基本的SpringBoot的应用。

这也是为什么springboot可以用来快速构建一个微服务，因为它实在是太方便了。
当然，实际开发中我们需要用到更多springboot的功能和特性，我们将会在接下来的章节中逐渐展开介绍。

# 参考文章
https://spring.io/quickstart