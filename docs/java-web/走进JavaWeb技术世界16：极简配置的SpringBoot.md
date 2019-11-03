本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章首发于我的个人博客：
> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《走进JavaWeb技术世界》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从servlet到框架，从ssm再到SpringBoot，一步步地学习JavaWeb基础知识，并上手进行实战，接着了解JavaWeb项目中经常要使用的技术和组件，包括日志组件、Maven、Junit，等等内容，以便让你更完整地了解整个JavaWeb技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

**文末赠送8000G的Java架构师学习资料，需要的朋友可以到文末了解领取方式，资料包括Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源）**
<!-- more -->


![](https://upload-images.jianshu.io/upload_images/7896890-9323041dd0ce883e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Spring Boot 概述

> **Build Anything with Spring Boot：**Spring Boot is the starting point for building all Spring-based applications. Spring Boot is designed to get you up and running as quickly as possible, with minimal upfront configuration of Spring.

上面是引自官网的一段话，大概是说： Spring Boot 是所有基于 Spring 开发的项目的起点。Spring Boot 的设计是为了让你尽可能快的跑起来 Spring 应用程序并且尽可能减少你的配置文件。

#### 什么是 Spring Boot

*   它使用 “习惯优于配置” （项目中存在大量的配置，此外还内置一个习惯性的配置，让你无须）的理念让你的项目快速运行起来。
*   它并不是什么新的框架，而是默认配置了很多框架的使用方式，就像 Maven 整合了所有的 jar 包一样，Spring Boot 整合了所有框架（引自：[springboot(一)：入门篇——纯洁的微笑](http://www.ityouknow.com/springboot/2016/01/06/springboot(%E4%B8%80)-%E5%85%A5%E9%97%A8%E7%AF%87.html)）

#### 使用 Spring Boot 有什么好处

回顾我们之前的 SSM 项目，搭建过程还是比较繁琐的，需要：

*   1）配置 web.xml，加载 spring 和 spring mvc
*   2）配置数据库连接、配置日志文件
*   3）配置家在配置文件的读取，开启注解
*   4）配置mapper文件
*   **.....**

而使用 Spring Boot 来开发项目则只需要非常少的几个配置就可以搭建起来一个 Web 项目，并且利用 IDEA 可以自动生成生成，这简直是太爽了...

*   划重点：简单、快速、方便地搭建项目；对主流开发框架的无配置集成；极大提高了开发、部署效率。

* * *

## Spring Boot 快速搭建

#### 第一步：新建项目

选择 Spring Initializr ，然后选择默认的 url 点击【Next】：

![](https://upload-images.jianshu.io/upload_images/7896890-3e2c9c5742c10c86.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

然后修改一下项目的信息：

![](https://upload-images.jianshu.io/upload_images/7896890-28dbe478ff25a3a0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

勾选上 Web 模板：

![](https://upload-images.jianshu.io/upload_images/7896890-532868b7e6760e03.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

选择好项目的位置，点击【Finish】：

![](https://upload-images.jianshu.io/upload_images/7896890-931cc2fb5c8964e9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如果是第一次配置 Spring Boot 的话可能需要等待一会儿 IDEA 下载相应的 依赖包，默认创建好的项目结构如下：

![](https://upload-images.jianshu.io/upload_images/7896890-9ac7acc56d5a32f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

项目结构还是看上去挺清爽的，少了很多配置文件，我们来了解一下默认生成的有什么：

*   SpringbootApplication： 一个带有 main() 方法的类，用于启动应用程序
*   SpringbootApplicationTests：一个空的 Junit 测试了，它加载了一个使用 Spring Boot 字典配置功能的 Spring 应用程序上下文
*   application.properties：一个空的 properties 文件，可以根据需要添加配置属性
*   pom.xml： Maven 构建说明文件

#### 第二步：HelloController

在【cn.wmyskxz.springboot】包下新建一个【HelloController】：

```
package cn.wmyskxz.springboot;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

/**
 * 测试控制器
 *
 * @author: @我没有三颗心脏
 * @create: 2018-05-08-下午 16:46
 */
@RestController
public class HelloController {

    @RequestMapping("/hello")
    public String hello() {
        return "Hello Spring Boot!";
    }
}
```

*   **@RestController 注解：** 该注解是 @Controller 和 @ResponseBody 注解的合体版

#### 第三步：利用 IDEA 启动 Spring Boot

我们回到 SpringbootApplication 这个类中，然后右键点击运行：

![](https://upload-images.jianshu.io/upload_images/7896890-bf1aa6ed5c0db7b4.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   **注意**：我们之所以在上面的项目中没有手动的去配置 Tomcat 服务器，是因为 Spring Boot 内置了 Tomcat

等待一会儿就会看到下方的成功运行的提示信息：

![](https://upload-images.jianshu.io/upload_images/7896890-63e43dc6a277de3e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到我们的 Tomcat 运行在 8080 端口，我们来访问 “`/hello`” 地址试一下：

![](https://upload-images.jianshu.io/upload_images/7896890-6111e1913c5bf6d6.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

可以看到页面成功显示出我们返回的信息。

* * *

## 解析 Spring Boot 项目

> 这一部分参考自：[Spring Boot干货系列（一）优雅的入门篇 ——嘟嘟独立博客](http://tengj.top/2017/02/26/springboot1/)

#### 解析 pom.xml 文件

让我们来看看默认生成的 pom.xml 文件中到底有一些什么特别：

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>cn.wmyskxz</groupId>
    springboot
    <version>0.0.1-SNAPSHOT</version>
    <packaging>jar</packaging>

    <name>springboot</name>
    <description>Demo project for Spring Boot</description>

    <parent>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-parent
        <version>2.0.1.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-web
        </dependency>

        <dependency>
            <groupId>org.springframework.boot</groupId>
            spring-boot-starter-test
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                spring-boot-maven-plugin
            </plugin>
        </plugins>
    </build>
</project>
```

我们可以看到一个比较陌生一些的标签 `<parent>` ，这个标签是在配置 Spring Boot 的父级依赖：

```
<parent>
    <groupId>org.springframework.boot</groupId>
    spring-boot-starter-parent
    <version>2.0.1.RELEASE</version>
    <relativePath/> <!-- lookup parent from repository -->
</parent>
```

有了这个，当前的项目才是 Spring Boot 项目，spring-boot-starter-parent 是一个特殊的 starter ，它用来提供相关的 Maven 默认依赖，**使用它之后，常用的包依赖就可以省去 version 标签。**

关于具体 Spring Boot 提供了哪些 jar 包的依赖，我们可以查看本地 Maven 仓库下：\repository\org\springframework\boot\spring-boot-dependencies\2.0.1.RELEASE\spring-boot-dependencies-2.0.1.RELEASE.pom 文件来查看，挺长的...

#### 应用入口类

Spring Boot 项目通常有一个名为 *Application 的入口类，入口类里有一个 main 方法， **这个 main 方法其实就是一个标准的 Javay 应用的入口方法。**

**@SpringBootApplication** 是 Spring Boot 的核心注解，它是一个组合注解，该注解组合了：**@Configuration、@EnableAutoConfiguration、@ComponentScan；** 若不是用 @SpringBootApplication 注解也可以使用这三个注解代替。

*   其中，**@EnableAutoConfiguration 让 Spring Boot 根据类路径中的 jar 包依赖为当前项目进行自动配置**，例如，添加了 spring-boot-starter-web 依赖，会自动添加 Tomcat 和 Spring MVC 的依赖，那么 Spring Boot 会对 Tomcat 和 Spring MVC 进行自动配置。
*   **Spring Boot 还会自动扫描 @SpringBootApplication 所在类的同级包以及下级包里的 Bean** ，所以入口类建议就配置在 grounpID + arctifactID 组合的包名下（这里为 cn.wmyskxz.springboot 包）

#### Spring Boot 的配置文件

Spring Boot 使用一个全局的配置文件 application.properties 或 application.yml，放置在【src/main/resources】目录或者类路径的 /config 下。

Spring Boot 不仅支持常规的 properties 配置文件，还支持 yaml 语言的配置文件。yaml 是以数据为中心的语言，在配置数据的时候具有面向对象的特征。

Spring Boot 的全局配置文件的作用是对一些默认配置的配置值进行修改。

> *   简单实例一下

![](https://upload-images.jianshu.io/upload_images/7896890-bcd65f7469b06608.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们同样的将 Tomcat 默认端口设置为 8080 ，并将默认的访问路径从 “`/`” 修改为 “`/hello`” 时，使用 properties 文件和 yml 文件的区别如上图。

*   注意： yml 需要在 “`:`” 后加一个空格，幸好 IDEA 很好地支持了 yml 文件的格式有良好的代码提示；

> *   我们可以自己配置多个属性

我们直接把 .properties 后缀的文件删掉，使用 .yml 文件来进行简单的配置，然后使用 @Value 来获取配置属性：

![](https://upload-images.jianshu.io/upload_images/7896890-0e808a82254d6a4b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

重启 Spring Boot ，输入地址：localhost:8080/hello 能看到正确的结果：

![](https://upload-images.jianshu.io/upload_images/7896890-c85216e8ea7910f0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   **注意：** 我们并没有在 yml 文件中注明属性的类型，而是在使用的时候定义的。

你也可以在配置文件中使用当前配置：

![](https://upload-images.jianshu.io/upload_images/7896890-37e91abbc4550982.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

仍然可以得到正确的结果：

![](https://upload-images.jianshu.io/upload_images/7896890-eabc3cd39b44fd0d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   **问题：** 这样写配置文件繁琐而且可能会造成类的臃肿，因为有许许多多的 @Value 注解。

> *   封装配置信息

![](https://upload-images.jianshu.io/upload_images/7896890-2599817d8f2f50d2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们可以把配置信息封装成一个类，首先在我们的 name 和 age 前加一个 student 前缀，然后新建一个 StudentProperties 的类用来封装这些信息，并用上两个注解：

*   @Component：表明当前类是一个 Java Bean
*   @ConfigurationProperties(prefix = "student")：表示获取前缀为 sutdent 的配置信息

这样我们就可以在控制器中使用，重启得到正确信息：

![](https://upload-images.jianshu.io/upload_images/7896890-84dc1215d01f3fa9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### Spring Boot 热部署

在目前的 Spring Boot 项目中，当发生了任何修改之后我们都需要重新启动才能够正确的得到效果，这样会略显麻烦，Spring Boot 提供了热部署的方式，当发现任何类发生了改变，就会通过 JVM 类加载的方式，加载最新的类到虚拟机中，这样就不需要重新启动也能看到修改后的效果了。

> *   做法也很简单，修改 pom.xml 即可！

我们往 pom.xml 中添加一个依赖就可以了：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    spring-boot-devtools
    <optional>true</optional> <!-- 这个需要为 true 热部署才有效 -->
</dependency>
```

重新启动 Spring Boot ，然后修改任意代码，就能观察到控制台的自动重启现象：

![](https://upload-images.jianshu.io/upload_images/7896890-cec869956c3cf158.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 关于如何在 IDEA 中配置热部署：[传送门](https://blog.csdn.net/xusheng_Mr/article/details/78771746)

* * *

## Spring Boot 使用

上面已经完成了 Spring Boot 项目的简单搭建，我们仅仅需要进行一些简单的设置，写一个 HelloController 就能够直接运行了，不要太简单...接下来我们再深入了解一下 Spring Boot 的使用。

#### Spring Boot 支持 JSP

Spring Boot 的默认视图支持是 Thymeleaf 模板引擎，但是这个我们不熟悉啊，我们还是想要使用 JSP 怎么办呢？

> *   第一步：修改 pom.xml 增加对 JSP 文件的支持

```
<!-- servlet依赖. -->
<dependency>
    <groupId>javax.servlet</groupId>
    javax.servlet-api
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>javax.servlet</groupId>
    jstl
</dependency>

<!-- tomcat的支持.-->
<dependency>
    <groupId>org.apache.tomcat.embed</groupId>
    tomcat-embed-jasper
    <scope>provided</scope>
</dependency>
```

> *   第二步：配置试图重定向 JSP 文件的位置

修改 application.yml 文件，将我们的 JSP 文件重定向到 /WEB-INF/views/ 目录下：

![](https://upload-images.jianshu.io/upload_images/7896890-7c17f7e10cfb2629.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> *   第三步：修改 HelloController

修改 @RestController 注解为 @Controller ，然后将 hello 方法修改为：

![](https://upload-images.jianshu.io/upload_images/7896890-2dc2c39cd962edc1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> *   第四步：新建 hello.jsp 文件

在【src/main】目录下依次创建 webapp、WEB-INF、views 目录，并创建一个 hello.jsp 文件：

![](https://upload-images.jianshu.io/upload_images/7896890-a180556d7ead9605.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> *   第五步：刷新网页

因为我们部署了热部署功能，所以只需要等待控制台重启信息完成之后再刷新网页就可以看到正确效果了：

![](https://upload-images.jianshu.io/upload_images/7896890-cfd20f747ffca978.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

*   关于 404，使用 spring-boot:run 运行项目可以解决：

![](https://upload-images.jianshu.io/upload_images/7896890-27c1bf46487ba5eb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 集成 MyBatis

> *   第一步：修改 pom.xml 增加对 MySql和 MyBatis 的支持

```

<!-- mybatis -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    mybatis-spring-boot-starter
    <version>1.1.1</version>
</dependency>
<!-- mysql -->
<dependency>
    <groupId>mysql</groupId>
    mysql-connector-java
    <version>5.1.21</version>
</dependency>
```

> *   第二步：新增数据库链接参数

这里我们就直接使用之前创建好的 student 表了吧：

![](https://upload-images.jianshu.io/upload_images/7896890-1eda563cfdfbae65.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> *   第三步：创建 Student 实体类和 StudentMapper 映射类

在【cn.wmyskxz.springboot】下新建一个【pojo】包，然后在其下创建一个 Student 类：

```
public class Student {

    private Integer id;
    private Integer student_id;
    private String name;
    private Integer age;
    private String sex;
    private Date birthday;

    /* getter and setter */
}
```

在【cn.wmyskxz.springboot】下新建一个【mapper】包，然后在其下创建一个 StudentMapper 映射类：

```
package cn.wmyskxz.springboot.mapper;

import cn.wmyskxz.springboot.pojo.Student;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Select;

import java.util.List;

@Mapper
public interface StudentMapper {

    @Select("SELECT * FROM student")
    List<Student> findAll();
}
```

> *   第四步：编写 StudentController

在【cn.wmyskxz.springboot】下新建一个【controller】包，然后在其下创建一个 StudentController ：

```
package cn.wmyskxz.springboot.controller;

import cn.wmyskxz.springboot.mapper.StudentMapper;
import cn.wmyskxz.springboot.pojo.Student;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;

import java.util.List;

/**
 * Student 控制器
 *
 * @author: @我没有三颗心脏
 * @create: 2018-05-08-下午 20:25
 */
@Controller
public class StudentController {

    @Autowired
    StudentMapper studentMapper;

    @RequestMapping("/listStudent")
    public String listStudent(Model model) {
        List<Student> students = studentMapper.findAll();
        model.addAttribute("students", students);
        return "listStudent";
    }
}
```

> 第五步：编写 listStudent.jsp 文件

我们简化一下 JSP 的文件，仅显示两个字段的数据：

```
<%@ page language="java" contentType="text/html; charset=UTF-8"
         pageEncoding="UTF-8"%>

<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c"%>

<table align='center' border='1' cellspacing='0'>
    <tr>
        <td>id</td>
        <td>name</td>
    </tr>
    <c:forEach items="${students}" var="s" varStatus="st">
        <tr>
            <td>${s.id}</td>
            <td>${s.name}</td>
        </tr>
    </c:forEach>
</table>
```

> *   第六步：重启服务器运行

因为往 pom.xml 中新增加了依赖的包，所以自动重启服务器没有作用，我们需要手动重启一次，然后在地址输入：localhost:8080/listStudent 查看效果：

![](https://upload-images.jianshu.io/upload_images/7896890-5fd3c075d07b5840.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 以上。

## springMVC和springboot的区别

Spring 框架就像一个家族，有众多衍生产品例如 boot、security、jpa等等。但他们的基础都是Spring 的 ioc和 aop ioc 提供了依赖注入的容器 aop ，解决了面向横切面的编程，然后在此两者的基础上实现了其他延伸产品的高级功能。Spring MVC是基于 Servlet 的一个 MVC 框架 主要解决 WEB 开发的问题，因为 Spring 的配置非常复杂，各种XML、 JavaConfig、hin处理起来比较繁琐。于是为了简化开发者的使用，从而创造性地推出了Spring boot，约定优于配置，简化了spring的配置流程。



说得更简便一些：Spring 最初利用“工厂模式”（DI）和“代理模式”（AOP）解耦应用组件。大家觉得挺好用，于是按照这种模式搞了一个 MVC框架（一些用Spring 解耦的组件），用开发 web 应用（ SpringMVC ）。然后有发现每次开发都写很多样板代码，为了简化工作流程，于是开发出了一些“懒人整合包”（starter），这套就是 Spring Boot。

**Spring MVC的功能**

Spring MVC提供了一种轻度耦合的方式来开发web应用。

Spring MVC是Spring的一个模块，式一个web框架。通过Dispatcher Servlet, ModelAndView 和 View Resolver，开发web应用变得很容易。解决的问题领域是网站应用程序或者服务开发——URL路由、Session、模板引擎、静态Web资源等等。

**Spring Boot的功能**

Spring Boot实现了自动配置，降低了项目搭建的复杂度。

众所周知Spring框架需要进行大量的配置，Spring Boot引入自动配置的概念，让项目设置变得很容易。Spring Boot本身并不提供Spring框架的核心特性以及扩展功能，只是用于快速、敏捷地开发新一代基于Spring框架的应用程序。也就是说，它并不是用来替代Spring的解决方案，而是和Spring框架紧密结合用于提升Spring开发者体验的工具。同时它集成了大量常用的第三方库配置(例如Jackson, JDBC, Mongo, Redis, Mail等等)，Spring Boot应用中这些第三方库几乎可以零配置的开箱即用(out-of-the-box)，大部分的Spring Boot应用都只需要非常少量的配置代码，开发者能够更加专注于业务逻辑。

Spring Boot只是承载者，辅助你简化项目搭建过程的。如果承载的是WEB项目，使用Spring MVC作为MVC框架，那么工作流程和你上面描述的是完全一样的，因为这部分工作是Spring MVC做的而不是Spring Boot。

对使用者来说，换用Spring Boot以后，项目初始化方法变了，配置文件变了，另外就是不需要单独安装Tomcat这类容器服务器了，maven打出jar包直接跑起来就是个网站，但你最核心的业务逻辑实现与业务流程实现没有任何变化。

所以，用最简练的语言概括就是：

Spring 是一个“引擎”；

Spring MVC 是基于Spring的一个 MVC 框架 ；

Spring Boot 是基于Spring4的条件注册的一套快速开发整合包。

## 微信公众号

### 个人公众号：程序员黄小斜

​
黄小斜是 985 硕士，阿里巴巴Java工程师，在自学编程、技术求职、Java学习等方面有丰富经验和独到见解，希望帮助到更多想要从事互联网行业的程序员们。
​
作者专注于 JAVA 后端技术栈，热衷于分享程序员干货、学习经验、求职心得，以及自学编程和Java技术栈的相关干货。
​
黄小斜是一个斜杠青年，坚持学习和写作，相信终身学习的力量，希望和更多的程序员交朋友，一起进步和成长！

**原创电子书:**
关注微信公众号【程序员黄小斜】后回复【原创电子书】即可领取我原创的电子书《菜鸟程序员修炼手册：从技术小白到阿里巴巴Java工程师》这份电子书总结了我2年的Java学习之路，包括学习方法、技术总结、求职经验和面试技巧等内容，已经帮助很多的程序员拿到了心仪的offer！

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取，包括Java、python、C++、大数据、机器学习、前端、移动端等方向的技术资料。


![](https://img-blog.csdnimg.cn/20190829222750556.jpg)


### 技术公众号：Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的微信公众号【Java技术江湖】

这是一位阿里 Java 工程师的技术小站。作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！


**Java工程师必备学习资源:** 
关注公众号后回复”Java“即可领取 Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源


![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

​                     