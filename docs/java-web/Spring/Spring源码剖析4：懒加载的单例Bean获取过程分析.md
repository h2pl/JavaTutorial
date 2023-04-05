# 目录

* [前言](#前言)
* [step1:](#step1)
* [step2:](#step2)
* [step3:](#step3)


本文转自五月的仓颉 https://www.cnblogs.com/xrq730

本系列文章将整理到我在GitHub上的《Java面试指南》仓库，更多精彩内容请到我的仓库里查看

> https://github.com/h2pl/Java-Tutorial

喜欢的话麻烦点下Star哈

文章将同步到我的个人博客：

> www.how2playlife.com

本文是微信公众号【Java技术江湖】的《Spring和SpringMVC源码分析》其中一篇，本文部分内容来源于网络，为了把本文主题讲得清晰透彻，也整合了很多我认为不错的技术博客内容，引用其中了一些比较好的博客文章，如有侵权，请联系作者。

该系列博文会告诉你如何从spring基础入手，一步步地学习spring基础和springmvc的框架知识，并上手进行项目实战，spring框架是每一个Java工程师必须要学习和理解的知识点，进一步来说，你还需要掌握spring甚至是springmvc的源码以及实现原理，才能更完整地了解整个spring技术体系，形成自己的知识框架。

后续还会有springboot和springcloud的技术专题，陆续为大家带来，敬请期待。

为了更好地总结和检验你的学习成果，本系列文章也会提供部分知识点对应的面试题以及参考答案。

如果对本系列文章有什么建议，或者是有什么疑问的话，也可以关注公众号【Java技术江湖】联系作者，欢迎你参与本系列博文的创作和修订。

<!-- more -->

## 前言

xml的读取应该是Spring的重要功能，因为Spring的大部分功能都是以配置做为切入点的。

我们在静态代码块中读取配置文件可以这样做：

```
   //这样来加载配置文件    
   XmlBeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml")); 
```

（1）XmlBeanFactory 继承 AbstractBeanDefinitionReader ，使用ResourceLoader 将资源文件路径转换为对应的Resource文件。

（2）通过DocumentLoader 对 Resource 文件进行转换，将 Resource 文件转换为 Document 文件。

（3）通过实现接口 BeanDefinitionDocumentReader 的 DefaultBeanDefinitionDocumentReader 类对Document 进行解析，并且使用 BeanDefinitionParserDelegate对Element进行解析。

## step1:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/bb0bf7543226c4ada238d93363f864d39da8e3e8.png)

在平常开发中，我们也可以使用Resource 获取 资源文件：

```
  Resource resource = new ClassPathResource("application.xml");
  InputStream in = resource.getInputStream();
```

## step2:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/13bd511377c0957e4ef8daebdf457585a9acabea.png)

在资源实现加载之前，调用了 super(parentBeanFactory) -- /**Ignore the given dependency interface for autowiring.(忽略接口的自动装配功能)*/

调用XmlBeanDefinitionReader 的 loadBeanDefinitions（）方法进行加载资源：

（1） 对Resource资源进行编码

（2） 通过SAX读取XML文件来创建InputSource对象

（3） 核心处理

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185226.png)

可以很直观的看出来是这个function是在解析xml文件从而获得对应的Document对象。

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185240.png)

在doLoadDocument方法里面还存一个方法getValidationModeForResource（）用来读取xml的验证模式。（和我关心的没什么关系，暂时不看了~）

转换成document也是最常用的方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185253.png)

## step3

/**Register the bean definitions contained in the given DOM document*/

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185301.png)

参数doc是doLoadBeanDefinitions（）方法传进来的 loadDocument 加载过来的。这边就很好的体现出了面向对象的单一全责原则，将逻辑处理委托給单一的类去处理。

在这边单一逻辑处理类是： BeanDefinitionDocumentReader

核心方法：<font color="#FF0000">documentReader.registerBeanDefinitions(doc, createReaderContext(resource));</font>

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185308.png)

<font color="#FF0000">开始解析：</font>

![image-20230405185319712](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230405185319712.png)

-------------

在Spring的xml配置中有两种方式来声明bean:

一种是默认的： <bean id = " " class = " " />

还有一种是自定义的： < tx : annotation-driven / >

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185328.png)

通过xml配置文件的默认配置空间来判断：http://www.springframework.org/schema/beans

对于默认标签的解析：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185340.png)

对Bean 配置的解析：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185347.png)

**BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele); 返回BeanDefinitionHolder**

![image-20230405185356720](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/image-20230405185356720.png)

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185402.png)

这边代码大致看下来：

1.  提取元素中的id和name属性
2.  进一步解析将其他属性封装到 BeanDefinition 的实现类中
3.  如果没有指定beanName 变使用默认规则生成beanName
4.  封装类BeanDefinitionHolder

可以先了解一下 BeanDefinition 这个类的作用。

BeanDefinition是一个接口，对应着配置文件中<bean>里面的所有配置，在Spring中存在着三个实现类：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185457.png)

在配置文件中，可以定义父<bean>和子<bean>，父<bean>是用RootDefinition来表示，子<bean>是用ChildBeanDefinition来表示。

Spring 通过BeanDefiniton将配置文件中的<bean>配置信息转换为容器内部表示，并且将这些BeanDefinition注册到BeanDefinitonRegistry中。

Spring容器的BeanDefinitonRegistry就像是Spring配置信息的内存数据库，主要是以map的形式保存的。

因此解析属性首先要创建用于承载属性的实例：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185505.png)

然后就是各种对属性的解析的具体方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/20230405185520.png)


## 微信公众号

### 个人公众号：黄小斜

黄小斜是跨考软件工程的 985 硕士，自学 Java 两年，拿到了 BAT 等近十家大厂 offer，从技术小白成长为阿里工程师。

作者专注于 JAVA 后端技术栈，热衷于分享程序员干货、学习经验、求职心得和程序人生，目前黄小斜的CSDN博客有百万+访问量，知乎粉丝2W+，全网已有10W+读者。

黄小斜是一个斜杠青年，坚持学习和写作，相信终身学习的力量，希望和更多的程序员交朋友，一起进步和成长！

**原创电子书:**
关注公众号【黄小斜】后回复【原创电子书】即可领取我原创的电子书《菜鸟程序员修炼手册：从技术小白到阿里巴巴Java工程师》

**程序员3T技术学习资源：** 一些程序员学习技术的资源大礼包，关注公众号后，后台回复关键字 **“资料”** 即可免费无套路获取。

**考研复习资料：**
计算机考研大礼包，都是我自己考研复习时用的一些复习资料,包括公共课和专业的复习视频，这里也推荐给大家，关注公众号后，后台回复关键字 **“考研”** 即可免费获取。

![](https://img-blog.csdnimg.cn/20190829222750556.jpg)


### 技术公众号：Java技术江湖

如果大家想要实时关注我更新的文章以及分享的干货的话，可以关注我的公众号【Java技术江湖】一位阿里 Java 工程师的技术小站，作者黄小斜，专注 Java 相关技术：SSM、SpringBoot、MySQL、分布式、中间件、集群、Linux、网络、多线程，偶尔讲点Docker、ELK，同时也分享技术干货和学习经验，致力于Java全栈开发！

**Java工程师必备学习资源:** 一些Java工程师常用学习资源，关注公众号后，后台回复关键字 **“Java”** 即可免费无套路获取。

![我的公众号](https://img-blog.csdnimg.cn/20190805090108984.jpg)

