# 目录
  * [maven简介](#maven简介)
    * [1.1 Maven是什么](#11-maven是什么)
    * [1.2 Maven发展史](#12-maven发展史)
    * [1.3 为什么要用Maven](#13-为什么要用maven)
  * [Maven 新手入门](#maven-新手入门)
    * [Maven概念](#maven概念)
    * [maven的安装](#maven的安装)
    * [maven目录](#maven目录)
    * [Maven常用命令说明](#maven常用命令说明)
    * [Maven使用](#maven使用)
    * [传递性依赖](#传递性依赖)
    * [依赖范围](#依赖范围)
  * [Maven和Gradle的比较](#maven和gradle的比较)
    * [依赖管理系统](#依赖管理系统)
    * [多模块构建](#多模块构建)
    * [一致的项目结构](#一致的项目结构)
    * [一致的构建模型](#一致的构建模型)
    * [插件机制](#插件机制)
  * [参考文章](#参考文章)



本文转载自互联网，侵删
本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看
> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

本系列文章将同步到我的个人博客：
> www.how2playlife.com

更多Java技术文章将陆续在微信公众号【Java技术江湖】更新，敬请关注。

本文是《走进JavaWeb技术世界》系列博文的其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从入门到进阶，从servlet到框架，从ssm再到SpringBoot，一步步地学习JavaWeb基础知识，并上手进行实战，接着了解JavaWeb项目中经常要使用的技术和组件，包括日志组件、Maven、Junit，等等内容，以便让你更完整地了解整个JavaWeb技术体系，形成自己的知识框架。为了更好地总结和检验你的学习成果，本系列文章也会提供每个知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

**文末赠送8000G的Java架构师学习资料，需要的朋友可以到文末了解领取方式，资料包括Java基础、进阶、项目和架构师等免费学习资料，更有数据库、分布式、微服务等热门技术学习视频，内容丰富，兼顾原理和实践，另外也将赠送作者原创的Java学习指南、Java程序员面试指南等干货资源）**
<!-- more -->
## maven简介
### 1.1 Maven是什么

Maven是一个项目管理和综合工具。 Maven提供了开发人员构建一个完整的生命周期框架。开发者团队可以自动完成项目的基础工具建设， Maven使用标准的目录结构和默认构建生命周期。

在多个开发者团队环境时， Maven可以设置按标准在非常短的时间里完成配置工作。 由于大部分项目的设置都很简单， 并且可重复使用， Maven让开发人员的工作更轻松， 同时创建报表， 检查， 构建和测试自动化设置。

用过GitHub的同学看到这里应该感觉似曾相识，对，Maven和git的作用很相似，都是为了方便项目的创建与管理。

概括地说， Maven简化和标准化项目建设过程。 处理编译， 分配， 文档， 团队协作和其他任务的无缝连接。 Maven增加可重用性并负责建立相关的任务。

### 1.2 Maven发展史

Maven设计之初， 是为了简化Jakarta Turbine项目的建设。 在几个项目， 每个项目包含了不同的Ant构建文件。 JAR检查到CVS。 Apache组织开发Maven可以建立多个项目， 发布项目信息， 项目部署， 在几个项目中JAR文件提供团队合作和帮助。

Maven的经历了Maven-> Maven2 -> Maven3的发展。

### 1.3 为什么要用Maven

Maven之前我们经常使用Ant来进行Java项目的构建， 然后Ant仅是一个构建工具， 它并未对项目的中的工程依赖以及项目本身进行管理， 并且Ant作为构建工具未能消除软件构建的重复性， 因为不同的项目需要编写对应的Ant任务。

Maven作为后来者， 继承了Ant的项目构建功能， 并且提供了依赖关系， 项目管理的功能， 因此它是一个项目管理和综合工具， 其核心的依赖管理， 项目信息管理， 中央仓库， 约定大于配置的核心功能使得Maven成为当前Java项目构建和管理工具的标准选择。

学习Maven的理由是非常多：

主流IDE（Eclipse,IDEA,Netbean） 够内置了Maven

SpringFramework已经不再提供jar的下载， 直接通过Maven进行依赖下载。

在github， 开源社区几乎所有流行的Java项目都是通过Maven进行构建和管理的。
## Maven 新手入门

### Maven概念

Maven作为一个构建工具，不仅能帮我们自动化构建，还能够抽象构建过程，提供构建任务实现;它跨平台，对外提供了一致的操作接口，这一切足以使它成为优秀的、流行的构建工具。

Maven不仅是构建工具，还是一个依赖管理工具和项目管理工具，它提供了中央仓库，能帮我自动下载构件。

### maven的安装

一：因为本人是window系统，所以这里只介绍window下如何安装，在安装Maven之前，先确认已经安装了JDK.
[![image.png](http://www.pianshen.com/images/221/09092452baf3edd653f387516fb8be0d.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-5a7737962f83f677.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

二：接着去[Maven官网](https://maven.apache.org/download.cgi)下载界面下载想要的版本解压到你想要的目录就行
[![image.png](http://www.pianshen.com/images/434/28b5fb0701c54ac4ada5500ed99bdc12.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-16d9fd82c7f938ae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

[![image.png](http://www.pianshen.com/images/370/fb1719c12ec1fec62d766168eb5fb2d2.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-7482108a7ff71031.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

三：最后设置一下环境变量，将Maven安装配置到操作系统环境中，主要就是配置M2_HOME和PATH两项，如图
[![image.png](http://www.pianshen.com/images/162/46a29661ccbce3f798e931c61c9b39aa.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-ffdf167e64415703.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

都搞定后，验证一下，打开doc输入 mvn -v如何得到下面信息就说明配置成功了
[![image.png](http://www.pianshen.com/images/496/373fd8fcc75b3e1af5f038ea33c36aa0.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-c473853017951ebe.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

### maven目录

[![image.png](http://www.pianshen.com/images/307/3244327db95e1096a8f82cf2fc66e62b.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-8a4c77bcc9a4565a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

*   bin目录：
    该目录包含了mvn运行的脚本，这些脚本用来配置java命令，准备好classpath和相关的Java系统属性，然后执行Java命令。
*   boot目录:
    该目录只包含一个文件，该文件为plexus-classworlds-2.5.2.jar。plexus-classworlds是一个类加载器框架，相对于默认的java类加载器，它提供了更加丰富的语法以方便配置，Maven使用该框架加载自己的类库。
*   conf目录:
    该目录包含了一个非常重要的文件settings.xml。直接修改该文件，就能在机器上全局地定制Maven的行为，一般情况下，我们更偏向于复制该文件至~/.m2/目录下（~表示用户目录），然后修改该文件，在用户范围定制Maven的行为。
*   lib目录:
    该目录包含了所有Maven运行时需要的Java类库，Maven本身是分模块开发的，因此用户能看到诸如maven-core-3.0.jar、maven-model-3.0.jar之类的文件，此外这里还包含一些Maven用到的第三方依赖如commons-cli-1.2.jar、commons-lang-2.6.jar等等。

### Maven常用命令说明

    mvn clean：表示运行清理操作（会默认把target文件夹中的数据清理）。
    mvn clean compile：表示先运行清理之后运行编译，会将代码编译到target文件夹中。
    mvn clean test：运行清理和测试。
    mvn clean package：运行清理和打包。
    mvn clean install：运行清理和安装，会将打好的包安装到本地仓库中，以便其他的项目可以调用。
    mvn clean deploy：运行清理和发布（发布到私服上面）。

上面的命令大部分都是连写的，大家也可以拆分分别执行，这是活的，看个人喜好以及使用需求，Eclipse Run as对maven项目会提供常用的命令。

### Maven使用



    <?xml version="1.0" encoding="UTF-8"?>
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
        <modelVersion>4.0.0</modelVersion>
        <groupId>com.tengj</groupId>
        <artifactId>springBootDemo1</artifactId>
        <version>0.0.1-SNAPSHOT</version>
        <name>springBootDemo1</name>
    </project>


代码的第一行是XML头，指定了该xml文档的版本和编码方式。
project是所有pom.xml的根元素，它还声明了一些POM相关的命名空间及xsd元素。
根元素下的第一个子元素modelVersion指定了当前的POM模型的版本，对于Maven3来说，它只能是4.0.0
代码中最重要是包含了groupId,artifactId和version了。这三个元素定义了一个项目基本的坐标，在Maven的世界，任何的jar、pom或者jar都是以基于这些基本的坐标进行区分的。

groupId定义了项目属于哪个组，随意命名，比如谷歌公司的myapp项目，就取名为 com.google.myapp

artifactId定义了当前Maven项目在组中唯一的ID,比如定义hello-world。

version指定了项目当前的版本0.0.1-SNAPSHOT,SNAPSHOT意为快照，说明该项目还处于开发中，是不稳定的。

name元素生命了一个对于用户更为友好的项目名称，虽然这不是必须的，但还是推荐为每个POM声明name,以方便信息交流

## 依赖的配置



    <project>
    ...
    <dependencies>
        <dependency>
            <groupId>实际项目</groupId>
    　　　　 <artifactId>模块</artifactId>
    　　　　 <version>版本</version>
    　　　　 <type>依赖类型</type>
    　　　　 <scope>依赖范围</scope>
    　　　　 <optional>依赖是否可选</optional>
    　　　　 <!—主要用于排除传递性依赖-->
    　　　　 <exclusions>
    　　　　     <exclusion>
    　　　　　　　    <groupId>…</groupId>
    　　　　　　　　　 <artifactId>…</artifactId>
    　　　　　　　</exclusion>
    　　　　 </exclusions>
    　　</dependency>
    <dependencies>
    ...
    </project>

根元素project下的dependencies可以包含一个或者多个dependency元素，以声明一个或者多个项目依赖。每个依赖可以包含的元素有：

*   grounpId、artifactId和version:以来的基本坐标，对于任何一个依赖来说，基本坐标是最重要的，Maven根据坐标才能找到需要的依赖。
*   type:依赖的类型，对于项目坐标定义的packaging。大部分情况下，该元素不必声明，其默认值为jar
*   scope:依赖的范围
*   optional:标记依赖是否可选
*   exclusions:用来排除传递性依赖

## 依赖范围

依赖范围就是用来控制依赖和三种classpath(编译classpath，测试classpath、运行classpath)的关系，Maven有如下几种依赖范围：

*   compile:编译依赖范围。如果没有指定，就会默认使用该依赖范围。使用此依赖范围的Maven依赖，对于编译、测试、运行三种classpath都有效。典型的例子是spring-code,在编译、测试和运行的时候都需要使用该依赖。
*   test:测试依赖范围。使用次依赖范围的Maven依赖，只对于测试classpath有效，在编译主代码或者运行项目的使用时将无法使用此依赖。典型的例子是Jnuit,它只有在编译测试代码及运行测试的时候才需要。
*   provided:已提供依赖范围。使用此依赖范围的Maven依赖，对于编译和测试classpath有效，但在运行时候无效。典型的例子是servlet-api,编译和测试项目的时候需要该依赖，但在运行项目的时候，由于容器以及提供，就不需要Maven重复地引入一遍。
*   runtime:运行时依赖范围。使用此依赖范围的Maven依赖，对于测试和运行classpath有效，但在编译主代码时无效。典型的例子是JDBC驱动实现，项目主代码的编译只需要JDK提供的JDBC接口，只有在执行测试或者运行项目的时候才需要实现上述接口的具体JDBC驱动。
*   system:系统依赖范围。该依赖与三种classpath的关系，和provided依赖范围完全一致，但是，使用system范围的依赖时必须通过systemPath元素显示地指定依赖文件的路径。由于此类依赖不是通过Maven仓库解析的，而且往往与本机系统绑定，可能构成构建的不可移植，因此应该谨慎使用。systemPath元素可以引用环境变量，如：

    
    <dependency>
        <groupId>javax.sql</groupId>
        <artifactId>jdbc-stdext</artifactId>
        <Version>2.0</Version>
        <scope>system</scope>
        <systemPath>${java.home}/lib/rt.jar</systemPath>
    </dependency>

*   import:导入依赖范围。该依赖范围不会对三种classpath产生实际的影响。
    上述除import以外的各种依赖范围与三种classpath的关系如下:

[![image.png](http://www.pianshen.com/images/89/9304bde50143be84e01bb47451e00a99.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-e7cdb7800f523b6b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

### 传递性依赖

比如一个account-email项目为例，account-email有一个compile范围的spring-code依赖，spring-code有一个compile范围的commons-logging依赖，那么commons-logging就会成为account-email的compile的范围依赖，commons-logging是account-email的一个传递性依赖


有了传递性依赖机制，在使用Spring Framework的时候就不用去考虑它依赖了什么，也不用担心引入多余的依赖。Maven会解析各个直接依赖的POM，将那些必要的间接依赖，以传递性依赖的形式引入到当前的项目中。

### 依赖范围

假设A依赖于B,B依赖于C，我们说A对于B是第一直接依赖，B对于C是第二直接依赖，A对于C是传递性依赖。第一直接依赖和第二直接依赖的范围决定了传递性依赖的范围，如下图所示，最左边一行表示第一直接依赖范围，最上面一行表示第二直接依赖范围，中间的交叉单元格则表示传递依赖范围。

[![image.png](http://www.pianshen.com/images/361/15e6b876f6226edf630f3fc9f92c9ec9.png "image.png")](http://upload-images.jianshu.io/upload_images/5811881-9e1e45b117656aac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240 "image.png")

从上图中，我们可以发现这样的规律：

*   当第二直接依赖的范围是compile的时候，传递性依赖的范围与第一直接依赖的范围一致；
*   当第二直接依赖的范围是test的时候，依赖不会得以传递；
*   当第二直接依赖的范围是provided的时候，只传递第一直接依赖范围也为provided的依赖，切传递依赖的范围同样为provided;
*   当第二直接依赖的范围是runtime的时候，传递性依赖的范围与第一直接依赖的范围一致，但compile列外，此时传递性依赖范围为runtime.

## Maven和Gradle的比较

Java生态体系中有三大构建工具：Ant、Maven和Gradle。其中，Ant是由Apache软件基金会维护；Maven这个单词来自于意第绪语（犹太语），意为知识的积累，最初在Jakata Turbine项目中用来简化构建过程；Gradle是一个基于Apache Ant和Apache Maven概念的项目自动化构建开源工具，它使用一种基于Groovy的特定领域语言(DSL)来声明项目设置，抛弃了基于XML的各种繁琐配置。

经过几年的发展，Ant几乎销声匿迹，而Maven由于较为不灵活的配置也渐渐被遗忘，而由于Gradle是基于Ant和Maven的一个优化版本，变得如日中天。

Maven的主要功能主要分为依赖管理系统、多模块构建、一致的项目结构、一致的构建模型和插件机制。这里通过这五个方面介绍两者的不同：

### 依赖管理系统

在Maven的管理体系中，用GroupID、ArtifactID和Version组成的Coordination唯一标识一个依赖项。任何基于Maven构建的项目自身也必须定义这三项属性，生成的包可以是Jar包，也可以是War包或Ear包。

一个典型的引用如下：

```
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-data-jpa
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-thymeleaf
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        spring-boot-starter-test
        <scope>test</scope>
    </dependency>
</dependencies>

```

这里 GroupID类似于C#中的namespace或者Java中的package，而ArtifactID相当于Class，Version相当于不同版本，如果Version忽略掉，将选择最新的版本链接。

同时，存储这些组件的仓库有远程仓库和本地仓库之分，远程仓库可以是使用世界公用的central仓库，也可以使用Apache Nexus自建的私有仓库；本地仓库则在本地计算机上。通过Maven安装目录下的settings.xml文件可以配置本地仓库的路径，以及采用的远程仓库地址。Gradle在设计时沿用了Maven这种依赖管理体系，同时也引入了改进，让依赖变得更加简洁：

    dependencies {
    // This dependency is exported to consumers, that is to say found on their compile classpath.
    api 'org.apache.commons:commons-math3:3.6.1'
    
    
    // This dependency is used internally, and not exposed to consumers on their own compile classpath.
    implementation 'com.google.guava:guava:23.0'
    
    // Use JUnit test framework
    testImplementation 'junit:junit:4.12'
    
    compile 'org.hibernate:hibernate-core:3.6.7.Final'
    testCompile ‘junit:junit:4.+'
    
    
    
    }

另外，Maven和Gradle对依赖项的审视也有所不同。在Maven中，一个依赖项有6种scope，分别是compile、provided、runtime、test、system、import。其中compile为默认。而gradle将其简化为4种，compile、runtime、testCompile、testRuntime。如上述代码“testCompile ‘junit:junit:4.+'”，在Gradle中支持动态的版本依赖，在版本号后面使用+号可以实现动态的版本管理。在解决依赖冲突方面Gradle的实现机制更加明确，两者都采用的是传递性依赖，而如果多个依赖项指向同一个依赖项的不同版本时可能会引起依赖冲突，Maven处理起来较为繁琐，而Gradle先天具有比较明确的策略。

### 多模块构建

在面向服务的架构中，通常将一个项目分解为多个模块。在Maven中需要定义parent POM(Project Object Model)作为一组module的通用配置模型，在POM文件中可以使用<modules>标签来定义一组子模块。parent POM中的build配置以及依赖配置会自动继承给子module。

Gradle也支持多模块构建，在parent的build.gradle中可以使用allprojects和subprojects代码块分别定义应用于所有项目或子项目中的配置。对于子模块中的定义放置在settings.gradle文件中，每一个模块代表project的对象实例，在parent的build.gradle中通过allproject或subprojects对这些对象进行操作，相比Maven更显灵活。

    allprojects {
    task nice << { task -> println "I'm $task.project.name" }
    }

执行命令gradle -q nice会依次打印出各模块的项目名称。

### 一致的项目结构

Maven指定了一套项目目录结构作为标准的java项目结构，Gradle也沿用了这一标准的目录结构。如果在Gradle项目中使用了Maven项目结构的话，在Gradle中无需进行多余的配置，只需在文件中包括apply plugin:'java'，系统会自动识别source、resource、test source、test resource等相应资源。

同时，Gradle作为JVM上的构建工具，也支持Groovy、Scala等源代码的构建，同样功能Maven通过一些插件也能达到目的，但配置方面Gradle更灵活。

### 一致的构建模型

为了解决Ant中对项目构建缺乏标准化的问题，Maven设置了标准的项目周期，构建周期：验证、初始化、生成原始数据、处理原始数据、生成资源、处理资源、编译、处理类、生成测试原始数据、处理测试原始数据、生成测试资源、处理测试资源、测试编译、处理测试类、测试、预定义包、生成包文件、预集成测试、集成测试、后集成测试、核实、安装、部署。但这种构建周期也是Maven应用的劣势。因为Maven将项目的构建周期限制过严，无法在构建周期中添加新的阶段，只能将插件绑定到已有的阶段上。而Gradle在构建模型上非常灵活，可以创建一个task，并随时通过depends建立与已有task的依赖关系。

### 插件机制

两者都采用了插件机制，Maven是基于XML进行配置，而在Gradle中更加灵活。

## 参考文章
http://www.pianshen.com/article/4537698845
https://www.jianshu.com/p/7248276d3bb5
https://www.cnblogs.com/lykbk/p/erwerwerwerwerwerwe.html
https://blog.csdn.net/u012131888/article/details/78209514
https://blog.csdn.net/belvine/article/details/81073365
https://blog.csdn.net/u012131888/article/details/78209514



                     
