![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-be7f7797a27a2dc5ab1ad8d11327b140c90.png)

接上文，我们继续分析。

### 6\. 注册 `BeanPostProcessor`: `registerBeanPostProcessors(beanFactory)`

在正式分析前，需要明确两个概念：

*   `BeanFactoryPostProcessor`，被称为 `BeanFactory` 的后置处理器，可以对 `BeanFactory` 进行一些操作；
*   `BeanPostProcessor`，被称为 `Bean` 的后置处理器，可以对 `Bean` 进行一些操作。

不要将本文的 `BeanPostProcessor` 与前面的 `BeanFactoryPostProcessor` 弄混了。

本文主要是对 `BeanPostProcessor` 的 `register` 操作 (`registerBeanPostProcessors(beanFactory)`)，即将 `BeanPostProcessor` 注册到 `BeanFactory` 中，那么调用是在什么时候呢？既然是对 `Bean` 的操作，当然是有了 bean 之后再运行了。

> `BeanPostProcessor` 也是 spring 中一个重要组件，关于该组件的详细分析，可以参考 [spring 组件之 BeanPostProcessors](https://my.oschina.net/funcy/blog/4597551)

废话不多说，直接上代码，同样地，对不重要的方法我们只给出调用链：

```
|-AbstractApplicationContext#refresh
 |-AbstractApplicationContext#registerBeanPostProcessors
  |-PostProcessorRegistrationDelegate
    #registerBeanPostProcessors(ConfigurableListableBeanFactory, AbstractApplicationContext)

```

最终调用到了 `PostProcessorRegistrationDelegate#registerBeanPostProcessors`，代码如下：

```
public static void registerBeanPostProcessors(
        ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

    // 获取spring中所有的 BeanPostProcessor，这里仅有一个bean: 
    // org.springframework.context.annotation.internalAutowiredAnnotationProcessor，
    // 即 AutowiredAnnotationBeanPostProcessor
    String[] postProcessorNames = beanFactory
            .getBeanNamesForType(BeanPostProcessor.class, true, false);

    int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() 
            + 1 + postProcessorNames.length;
    beanFactory.addBeanPostProcessor(
            new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

    List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
    List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
    List<String> orderedPostProcessorNames = new ArrayList<>();
    List<String> nonOrderedPostProcessorNames = new ArrayList<>();

    // 先获取实现了PriorityOrdered的BeanPostProcessor
    // 再获取实现了Ordered的BeanPostProcessor
    // 最后再获取不满足以上条件的BeanPostProcessor
    for (String ppName : postProcessorNames) {
        if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
            BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
            priorityOrderedPostProcessors.add(pp);
            if (pp instanceof MergedBeanDefinitionPostProcessor) {
                internalPostProcessors.add(pp);
            }
        }
        else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
            orderedPostProcessorNames.add(ppName);
        }
        else {
            nonOrderedPostProcessorNames.add(ppName);
        }
    }

    // 处理priorityOrderedPostProcessor：排序，然后添加到beanFactory中
    sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

    List<BeanPostProcessor> orderedPostProcessors 
            = new ArrayList<>(orderedPostProcessorNames.size());
    for (String ppName : orderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        orderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    // 处理orderedPostProcessor：排序，然后添加到beanFactory中
    sortPostProcessors(orderedPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, orderedPostProcessors);

    List<BeanPostProcessor> nonOrderedPostProcessors 
            = new ArrayList<>(nonOrderedPostProcessorNames.size());
    for (String ppName : nonOrderedPostProcessorNames) {
        BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
        nonOrderedPostProcessors.add(pp);
        if (pp instanceof MergedBeanDefinitionPostProcessor) {
            internalPostProcessors.add(pp);
        }
    }
    // 处理余下的BeanPostProcessor：排序，然后添加到beanFactory中
    registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);    
    // 处理internalPostProcessor：排序，然后添加到beanFactory中
    // AutowiredAnnotationBeanPostProcessor实现了MergedBeanDefinitionPostProcessor，
    // 因此这里会再次注册
    sortPostProcessors(internalPostProcessors, beanFactory);
    registerBeanPostProcessors(beanFactory, internalPostProcessors);

    beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
}

```

这块代码主要是对 `BeanFactoryPostProcessor` 进行注册操作，步骤如下：

*   将实现了 `PriorityOrdered` 的 `BeanPostProcessor` 注册到 `beanFactory` 中；
*   将实现了 `Ordered` 的 `BeanPostProcessor` 注册到 `beanFactory` 中；
*   将不满足上述两个条件的 `BeanPostProcessor` 注册到 `beanFactory` 中；
*   将所有实现了 `MergedBeanDefinitionPostProcessor` 的 `BeanPostProcessor` 再次注册到 `beanFactory` 中。

事实上，对 demo01 而言，这里注册的 bean 只有一个：`AutowiredAnnotationBeanPostProcessor`，它同时实现了 `MergedBeanDefinitionPostProcessor` 与 `PriorityOrdered`，因此会注册两次。

尽管注册了多次 `AutowiredAnnotationBeanPostProcessor`，但最终只会存在一个，让我们进入 `registerBeanPostProcessors` 看看 spring 是如何注册的，一路跟下去，代码到了 `AbstractBeanFactory#addBeanPostProcessor`:

> AbstractBeanFactory#addBeanPostProcessor

```
private final List<BeanPostProcessor> beanPostProcessors = new CopyOnWriteArrayList<>();

@Override
public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
    Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
    // 先进行移除，因此多次注册beanPostProcessors也只存在一个
    this.beanPostProcessors.remove(beanPostProcessor);
    if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
        this.hasInstantiationAwareBeanPostProcessors = true;
    }
    if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
        this.hasDestructionAwareBeanPostProcessors = true;
    }
    this.beanPostProcessors.add(beanPostProcessor);
}

```

可以看到，所谓的注册到 `BeanFactory`，其实就是把 `beanPostProcessor` 加入到 `BeanFactory` 的 `beanPostProcessors` 中。

本文仅绍了 `beanPostProcessor` 的注册，关于 `beanPostProcessor` 的调用，后续的分析再提到，本文的分析就先到这里吧！

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4657181](https://my.oschina.net/funcy/blog/4657181) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_