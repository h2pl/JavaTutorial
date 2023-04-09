# 目录
* [前言](#前言)
* [准备](#准备)
* [读取](#读取)
* [解析](#解析)
* [注册](#注册)


本文转载自互联网，侵删

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

本文大致地介绍了IOC容器的初始化过程，只列出了比较重要的过程和代码，可以从中看出IOC容器执行的大致流程。

接下来的文章会更加深入剖析Bean容器如何解析xml，注册和初始化bean，以及如何获取bean实例等详细的过程。

转自：[http://www.importnew.com/19243.html](http://www.importnew.com/19243.html)

1\. 初始化

大致单步跟了下Spring IOC的初始化过程，整个脉络很庞大，初始化的过程主要就是读取XML资源，并解析，最终注册到Bean Factory中：

[![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/032179be-070b-11e5-9ecf-d7befc804e9d.png "flow")](https://cloud.githubusercontent.com/assets/1736354/7897341/032179be-070b-11e5-9ecf-d7befc804e9d.png "flow")

在完成初始化的过程后，Bean们就在BeanFactory中蓄势以待地等调用了。下面通过一个具体的例子，来详细地学习一下初始化过程，例如当加载下面一个bean：

```
<bean id="XiaoWang" class="com.springstudy.talentshow.SuperInstrumentalist">
    <property name="instruments">
        <list>
            <ref bean="piano"/>
            <ref bean="saxophone"/>
        </list>
    </property>
</bean>
```

加载时需要读取、解析、注册bean，这个过程具体的调用栈如下所示：
[![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/8a488060-06e6-11e5-9ad9-4ddd3375984f.png "load")](https://cloud.githubusercontent.com/assets/1736354/7896285/8a488060-06e6-11e5-9ad9-4ddd3375984f.png "load")

下面对每一步的关键的代码进行详细分析：

## 准备

保存配置位置，并刷新
在调用ClassPathXmlApplicationContext后，先会将配置位置信息保存到configLocations，供后面解析使用，之后，会调用`AbstractApplicationContext`的refresh方法进行刷新：

```
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh,
        ApplicationContext parent) throws BeansException {

    super(parent);
    // 保存位置信息，比如`com/springstudy/talentshow/talent-show.xml`
    setConfigLocations(configLocations);
    if (refresh) {
        // 刷新
        refresh();
    }
}

public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        prepareRefresh();
        // Tell the subclass to refresh the internal bean factory.
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // Prepare the bean factory for use in this context.
        prepareBeanFactory(beanFactory);
        try {
            // Allows post-processing of the bean factory in context subclasses.
            postProcessBeanFactory(beanFactory);
            // Invoke factory processors registered as beans in the context.
            invokeBeanFactoryPostProcessors(beanFactory);
            // Register bean processors that intercept bean creation.
            registerBeanPostProcessors(beanFactory);
            // Initialize message source for this context.
            initMessageSource();
            // Initialize event multicaster for this context.
            initApplicationEventMulticaster();
            // Initialize other special beans in specific context subclasses.
            onRefresh();
            // Check for listener beans and register them.
            registerListeners();
            // Instantiate all remaining (non-lazy-init) singletons.
            finishBeanFactoryInitialization(beanFactory);
            // Last step: publish corresponding event.
            finishRefresh();
        }
        catch (BeansException ex) {
            // Destroy already created singletons to avoid dangling resources.
            destroyBeans();
            // Reset 'active' flag.
            cancelRefresh(ex);
            // Propagate exception to caller.
            throw ex;
        }
    }
}
```

创建载入BeanFactory

```
protected final void refreshBeanFactory() throws BeansException {
    // ... ...
    DefaultListableBeanFactory beanFactory = createBeanFactory();
    // ... ...
    loadBeanDefinitions(beanFactory);
    // ... ...
}
```

创建XMLBeanDefinitionReader

```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory)
     throws BeansException, IOException {
    // Create a new XmlBeanDefinitionReader for the given BeanFactory.
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    // ... ...
    // Allow a subclass to provide custom initialization of the reader,
    // then proceed with actually loading the bean definitions.
    initBeanDefinitionReader(beanDefinitionReader);
    loadBeanDefinitions(beanDefinitionReader);
```

## 读取

创建处理每一个resource

```
public int loadBeanDefinitions(String location, Set<Resource> actualResources)
     throws BeanDefinitionStoreException {
    // ... ...
    // 通过Location来读取Resource
    Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
    int loadCount = loadBeanDefinitions(resources);
    // ... ...
}

public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int counter = 0;
    for (Resource resource : resources) {
        // 载入每一个resource
        counter += loadBeanDefinitions(resource);
    }
    return counter;
}
```

处理XML每个元素

```
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // ... ...
    NodeList nl = root.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        if (node instanceof Element) {
            Element ele = (Element) node;
            if (delegate.isDefaultNamespace(ele)) {
                // 处理每个xml中的元素，可能是import、alias、bean
                parseDefaultElement(ele, delegate);
            }
            else {
                delegate.parseCustomElement(ele);
            }
        }
    }
    // ... ...
}
```

解析和注册bean

```
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 解析
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // 注册
            // Register the final decorated instance.
            BeanDefinitionReaderUtils.registerBeanDefinition(
                bdHolder, getReaderContext().getRegistry());
        }
        catch (BeanDefinitionStoreException ex) {
            getReaderContext().error("Failed to register bean definition with name '" +
                    bdHolder.getBeanName() + "'", ele, ex);
        }
        // Send registration event.
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```

本步骤中，通过parseBeanDefinitionElement将XML的元素解析为BeanDefinition，然后存在BeanDefinitionHolder中，然后再利用BeanDefinitionHolder将BeanDefinition注册，实质就是把BeanDefinition的实例put进BeanFactory中，和后面将详细的介绍解析和注册过程。

## 解析

[![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/eae02bc6-06e6-11e5-941a-d1f59e3b363f.png "process")](https://cloud.githubusercontent.com/assets/1736354/7896302/eae02bc6-06e6-11e5-941a-d1f59e3b363f.png "process")

处理每个Bean的元素

```
public AbstractBeanDefinition parseBeanDefinitionElement(
        Element ele, String beanName, BeanDefinition containingBean) {

    // ... ...
    // 创建beandefinition
    AbstractBeanDefinition bd = createBeanDefinition(className, parent);

    parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
    bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

    parseMetaElements(ele, bd);
    parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
    parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
    // 处理“Constructor”
    parseConstructorArgElements(ele, bd);
    // 处理“Preperty”
    parsePropertyElements(ele, bd);
    parseQualifierElements(ele, bd);
    // ... ...
}
```

处理属性的值

```
public Object parsePropertyValue(Element ele, BeanDefinition bd, String propertyName) {
    String elementName = (propertyName != null) ?
                    "<property> element for property '" + propertyName + "'" :
                    "<constructor-arg> element";

    // ... ...
    if (hasRefAttribute) {
    // 处理引用
        String refName = ele.getAttribute(REF_ATTRIBUTE);
        if (!StringUtils.hasText(refName)) {
            error(elementName + " contains empty 'ref' attribute", ele);
        }
        RuntimeBeanReference ref = new RuntimeBeanReference(refName);
        ref.setSource(extractSource(ele));
        return ref;
    }
    else if (hasValueAttribute) {
    // 处理值
        TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
        valueHolder.setSource(extractSource(ele));
        return valueHolder;
    }
    else if (subElement != null) {
    // 处理子类型（比如list、map等）
        return parsePropertySubElement(subElement, bd);
    }
    // ... ...
}

```

1.4 注册

```
public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}

public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    // ......

    // 将beanDefinition注册
    this.beanDefinitionMap.put(beanName, beanDefinition);

    // ......
}

```

注册过程中，最核心的一句就是：this.beanDefinitionMap.put(beanName, beanDefinition)，也就是说注册的实质就是以beanName为key，以beanDefinition为value，将其put到HashMap中。

## 注册

```
    public static void registerBeanDefinition(
        BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
        throws BeanDefinitionStoreException {

    // Register bean definition under primary name.
    String beanName = definitionHolder.getBeanName();
    registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

    // Register aliases for bean name, if any.
    String[] aliases = definitionHolder.getAliases();
    if (aliases != null) {
        for (String alias : aliases) {
            registry.registerAlias(beanName, alias);
        }
    }
}

public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
        throws BeanDefinitionStoreException {

    // ......

    // 将beanDefinition注册
    this.beanDefinitionMap.put(beanName, beanDefinition);

    // ......

```

理解了以上两个过程，我们就可以自己实现一个简单的Spring框架了。于是，我根据自己的理解实现了一个简单的IOC框架Simple Spring，有兴趣可以看看。



注册过程中，最核心的一句就是：`this.beanDefinitionMap.put(beanName, beanDefinition)`，也就是说注册的实质就是以beanName为key，以beanDefinition为value，将其put到HashMap中。

### 注入依赖

当完成初始化IOC容器后，如果bean没有设置lazy-init(延迟加载)属性，那么bean的实例就会在初始化IOC完成之后，及时地进行初始化。初始化时会先建立实例，然后根据配置利用反射对实例进行进一步操作，具体流程如下所示：
[![](https://cloud.githubusercontent.com/assets/1736354/7929429/615570ea-0930-11e5-8097-ae982ef7709d.png "bean_flow")](https://cloud.githubusercontent.com/assets/1736354/7929429/615570ea-0930-11e5-8097-ae982ef7709d.png "bean_flow")

创建bean的实例
创建bean的实例过程函数调用栈如下所示：
[![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/cec01bcc-092f-11e5-81ad-88c285f33845.png "create_bean")](https://cloud.githubusercontent.com/assets/1736354/7929379/cec01bcc-092f-11e5-81ad-88c285f33845.png "create_bean")

注入bean的属性
注入bean的属性过程函数调用栈如下所示：
[![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/db58350e-092f-11e5-82a4-caaf349291ea.png "inject_property")](https://cloud.githubusercontent.com/assets/1736354/7929381/db58350e-092f-11e5-82a4-caaf349291ea.png "inject_property")

在创建bean和注入bean的属性时，都是在doCreateBean函数中进行的，我们重点看下：

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd,
            final Object[] args) {
        // Instantiate the bean.
        BeanWrapper instanceWrapper = null;
        if (mbd.isSingleton()) {
            instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
        }
        if (instanceWrapper == null) {
            // 创建bean的实例
            instanceWrapper = createBeanInstance(beanName, mbd, args);
        }

        // ... ...

        // Initialize the bean instance.
        Object exposedObject = bean;
        try {
            // 初始化bean的实例，如注入属性
            populateBean(beanName, mbd, instanceWrapper);
            if (exposedObject != null) {
                exposedObject = initializeBean(beanName, exposedObject, mbd);
            }
        }

        // ... ...
    }

```

理解了以上两个过程，我们就可以自己实现一个简单的Spring框架了。于是，我根据自己的理解实现了一个简单的IOC框架[Simple Spring](https://github.com/Yikun/simple-spring)，有兴趣可以看看。



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
 


