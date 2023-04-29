`BeanPostProcessor` 中文名为 spring bean 的后置处理器，区别于 `BeanFactoryPostProcessor`，`BeanPostProcessor` 可以对 bean 进行操作。

spring `BeanPostProcessor` 在 bean 的创建过程中执行，执行时机如下:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b7a4d4bc1bfbd76537e40cf843b0d18df93.png)

在 bean 的创建过程中，`BeanPostProcessor` 一共执行过 8 次：

1.  尝试生成代理对象
2.  推断构造方法
3.  获取注入的属性
4.  添加三级缓存
5.  是否需要注入属性
6.  填充属性
7.  初始化前
8.  初始化后

本来将会逐一梳理这些过程中执行的 `BeanPostProcessor`。

### 1\. 什么是 `BeanPostProcessor`

梳理 `BeanPostProcessor` 前，我们先来看看什么是 `BeanPostProcessor`，代码如下：

```
public interface BeanPostProcessor {

    /**
     * 初始化前执行
     */
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) 
            throws BeansException {
        return bean;
    }

    /**
     * 初始化后执行
     */
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) 
            throws BeansException {
        return bean;
    }

}

```

`BeanPostProcessor` 是一个接口，定义了 bean 初始前后的一些操作，我们可以实现这个接口，重写它的两个方法，就可以在 bean 初始化前进行一些处理操作了。

`BeanPostProcessor` 在 `AbstractApplicationContext#registerBeanPostProcessors` 中注册。

实际上，`BeanPostProcessor` 还有众多的子接口，这些我们都统称为 `BeanPostProcessor`，本文的主要目的就是梳理这些 `BeanPostProcessor`。

### 2. `BeanPostProcessor` 梳理

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-102ca0d1e4db82a28871661241b05bc3956.png)

#### 2.1 尝试生成代理对象

*   调用位置：`AbstractAutowireCapableBeanFactory#resolveBeforeInstantiation`
*   执行方法：`InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`
*   `AbstractAutoProxyCreator#postProcessBeforeInstantiation`：生成代理对象

#### 2.2 推断构造方法

*   调用位置：`AbstractAutowireCapableBeanFactory#determineConstructorsFromBeanPostProcessors`
*   执行方法：`SmartInstantiationAwareBeanPostProcessor#determineCandidateConstructors`
*   `AutowiredAnnotationBeanPostProcessor#determineCandidateConstructors`：推断构造方法

#### 2.3 获取注入的属性

*   调用位置：`AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors`
*   执行方法：`MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition`
*   `ApplicationListenerDetector#postProcessMergedBeanDefinition`：收集单例的 `ApplicationListener`
*   `AutowiredAnnotationBeanPostProcessor#postProcessMergedBeanDefinition`：查找被 `@Autowired`、`@Value`、`@Inject` 标记的属性与方法
*   `CommonAnnotationBeanPostProcessor#postProcessMergedBeanDefinition`：查找被 `@Resource` 标记的属性与方法
*   `InitDestroyAnnotationBeanPostProcessor#postProcessMergedBeanDefinition`：查找被 `@PostConstruct`、`@PreDestroy` 标记的方法

#### 2.4 添加三级缓存

*   调用位置（方法并没有执行）：`AbstractAutowireCapableBeanFactory#doCreateBean`
*   执行方法：`SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference`
*   `AbstractAutoProxyCreator#getEarlyBeanReference`：提前生成代理对象

#### 2.5 是否需要注入属性

*   调用位置：`AbstractAutowireCapableBeanFactory#populateBean`
*   执行方法：`InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`

#### 2.6 填充属性

*   调用位置：`AbstractAutowireCapableBeanFactory#populateBean`
*   执行方法：`InstantiationAwareBeanPostProcessor#postProcessProperties`
*   `AutowiredAnnotationBeanPostProcessor#postProcessProperties`：填充被 `@Autowired`、`@Value`、`@Inject` 标记的属性与方法
*   `CommonAnnotationBeanPostProcessor#postProcessProperties`：填充被 `@Resource` 标记的属性与方法
*   `ImportAwareBeanPostProcessor#postProcessProperties`：为 `EnhancedConfiguration` 实例设置 `beanFactory`

#### 2.7 初始化前

*   调用位置：`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization`
*   执行方法：`BeanPostProcessor#postProcessBeforeInitialization`
*   `ApplicationContextAwareProcessor#postProcessBeforeInitialization`：调用 `XxxAware` 接口的方法
*   `BeanValidationPostProcessor#postProcessBeforeInitialization`：处理 `JSR-303` 校验
*   `ImportAwareBeanPostProcessor#postProcessBeforeInitialization`：调用 `ImportAware` 接口的方法
*   `InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization`：调用被 `@PostConstruct` 标记的方法
*   `LoadTimeWeaverAwareProcessor#postProcessBeforeInitialization`：调用 `LoadTimeWeaverAware` 接口的方法
*   `ServletContextAwareProcessor#postProcessBeforeInitialization`：调用 `ServletContextAware` 接口的方法，设置 `servletContext` 与 `servletConfig`

#### 2.8 初始化后

*   调用位置：`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization`
*   执行方法：`BeanPostProcessor#postProcessAfterInitialization`
*   `AbstractAdvisingBeanPostProcessor#postProcessAfterInitialization`：处理 `AopInfrastructureBean`
*   `AbstractAutoProxyCreator#postProcessAfterInitialization`：生成代理对象
*   `AdvisorAdapterRegistrationManager#postProcessAfterInitialization`：如果当前 bean 是 `AdvisorAdapter`，则注册
*   `ApplicationListenerDetector#postProcessAfterInitialization`：如果当前 bean 是 `ApplicationListener`，则添加到事件监听器中
*   `BeanPostProcessorChecker#postProcessAfterInitialization`：检查操作，打了个 log
*   `BeanValidationPostProcessor#postProcessAfterInitialization`：处理 `JSR-303` 校验
*   `JmsListenerAnnotationBeanPostProcessor#postProcessAfterInitialization`：处理 `@JmsListener` 注解
*   `ScheduledAnnotationBeanPostProcessor#postProcessAfterInitialization`：处理 `@Scheduled` 注解
*   `SimpleServletPostProcessor#postProcessAfterInitialization`：对 `Servlet` 实例，调用方法 `Servlet#init(ServletConfig)`

### 3\. 总结

最后用一个表格来总结这些 `BeanPostProcessor`:

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b1117b66c4881f366669dab69b332164d8f.png)

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4597551](https://my.oschina.net/funcy/blog/4597551) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_