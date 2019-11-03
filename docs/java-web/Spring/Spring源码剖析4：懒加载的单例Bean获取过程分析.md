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

![bb0bf7543226c4ada238d93363f864d39da8e3e8](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/bb0bf7543226c4ada238d93363f864d39da8e3e8.png)

     在平常开发中，我们也可以使用Resource 获取 资源文件：

```
  Resource resource = new ClassPathResource("application.xml");
  InputStream in = resource.getInputStream();
```

## step2:

![13bd511377c0957e4ef8daebdf457585a9acabea](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/13bd511377c0957e4ef8daebdf457585a9acabea.png)

      在资源实现加载之前，调用了 super(parentBeanFactory) --  /**Ignore the given dependency interface for autowiring.(忽略接口的自动装配功能)*/

      调用XmlBeanDefinitionReader 的 loadBeanDefinitions（）方法进行加载资源：

      （1） 对Resource资源进行编码

      （2） 通过SAX读取XML文件来创建InputSource对象

      （3） 核心处理

![7613f54877fef111ccbe68f2c3a96a9588029fb3](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/7613f54877fef111ccbe68f2c3a96a9588029fb3.png)

       可以很直观的看出来是这个function是在解析xml文件从而获得对应的Document对象。

![4b3425c37260bbb7e68ace81867259089871a0db](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/4b3425c37260bbb7e68ace81867259089871a0db.png)

      在doLoadDocument方法里面还存一个方法getValidationModeForResource（）用来读取xml的验证模式。（和我关心的没什么关系，暂时不看了~）

      转换成document也是最常用的方法：

     ![869effccb2e4f7b69e0b53d17fe0a2b50044d61b](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/869effccb2e4f7b69e0b53d17fe0a2b50044d61b.png)

## step3 : 我们已经step by step 的看到了如何将xml文件转换成Document的，现在就要分析是如何提取和注册bean的。

            /**Register the bean definitions contained in the given DOM document*/

![2daf08bfd105a15d3c5eaf411fdb0083b3969f81](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/2daf08bfd105a15d3c5eaf411fdb0083b3969f81.png)

参数doc是doLoadBeanDefinitions（）方法传进来的  loadDocument 加载过来的。这边就很好的体现出了面向对象的单一全责原则，将逻辑处理委托給单一的类去处理。

在这边单一逻辑处理类是：  BeanDefinitionDocumentReader

核心方法：  <font color="#FF0000">documentReader.registerBeanDefinitions(doc, createReaderContext(resource));</font>

<font color="#FF0000">![46ca5d8a7167fb010024f79e1f334820d7d2080a](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/46ca5d8a7167fb010024f79e1f334820d7d2080a.png)</font>

<font color="#FF0000">开始解析：</font>

<font color="#FF0000">![43eb5d219f00c7b5c99c0eed0828b9ff2550af41](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/43eb5d219f00c7b5c99c0eed0828b9ff2550af41.png)</font> 

-------------

在Spring的xml配置中有两种方式来声明bean:

     一种是默认的：  <bean id = " " class = " " />

     还有一种是自定义的：  < tx : annotation-driven / >

![fecfb37a9f121df42d5754f6fdf99367539936c6](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/fecfb37a9f121df42d5754f6fdf99367539936c6.png)

通过xml配置文件的默认配置空间来判断：http://www.springframework.org/schema/beans

对于默认标签的解析：

![2b3bba761875a27d4ca52d72e76de934a90e51a1](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/2b3bba761875a27d4ca52d72e76de934a90e51a1.png)

对Bean 配置的解析：

![8dac08836a4c27f90d15355991774186886ef141](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/8dac08836a4c27f90d15355991774186886ef141.png)

**BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);  返回BeanDefinitionHolder**

**![f163a5df0d4ea8e105526fa7ef39547a1c188047](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/f163a5df0d4ea8e105526fa7ef39547a1c188047.png)**

**![b93c6e1209359777b877f17e203e6226a269f4e0](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/b93c6e1209359777b877f17e203e6226a269f4e0.png)**

这边代码大致看下来：

1.  提取元素中的id和name属性
2.  进一步解析将其他属性封装到 BeanDefinition 的实现类中
3.  如果没有指定beanName 变使用默认规则生成beanName
4.  封装类BeanDefinitionHolder

可以先了解一下  BeanDefinition  这个类的作用。

      BeanDefinition是一个接口，对应着配置文件中<bean>里面的所有配置，在Spring中存在着三个实现类：

![917b789f984dee75d3b2748d885dcdd6541df8fe](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/917b789f984dee75d3b2748d885dcdd6541df8fe.png)

      在配置文件中，可以定义父<bean>和子<bean>，父<bean>是用RootDefinition来表示，子<bean>是用ChildBeanDefinition来表示。

      Spring 通过BeanDefiniton将配置文件中的<bean>配置信息转换为容器内部表示，并且将这些BeanDefinition注册到BeanDefinitonRegistry中。

Spring容器的BeanDefinitonRegistry就像是Spring配置信息的内存数据库，主要是以map的形式保存的。

     因此解析属性首先要创建用于承载属性的实例：

![83cf6bccba49fb369d0221e7970187041da349f0](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/83cf6bccba49fb369d0221e7970187041da349f0.png)

然后就是各种对属性的解析的具体方法：

![89c0d5422e0495347f18fa03110bfb2afa255493](https://oss-cn-hangzhou.aliyuncs.com/yqfiles/89c0d5422e0495347f18fa03110bfb2afa255493.png)