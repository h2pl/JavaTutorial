### 1\. 什么是 `BeanFactoryPostProcessor`

`BeanFactoryPostProcessor` 中文名叫 spring beanFactory 的后置处理器，可以用来定制化 beanFactory 的一些行为。

spring 为我们提供了两种 `BeanFactoryPostProcessor`：

* `org.springframework.beans.factory.config.BeanFactoryPostProcessor`

  ```
  /**
   * beanFactory 的后置处理器，可以改变 beanFactory 的一些行为
   */
  public interface BeanFactoryPostProcessor {
  
      /**
       * 处理 beanFactory 的方法，参数为 beanFactory，实际类型是 DefaultListableBeanFactory
       */
      void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
              throws BeansException;
  
  }
  
  ```

* `org.springframework.beans.factory.support.BeanDefinitionRegistryPostProcessor`

  ```
  /**
   * BeanDefinition 注册器，从名称来看，就是用来注册 beanDefinition 的
   * 继承了 BeanFactoryPostProcessor 接口，
   * 也可以重写 BeanFactoryPostProcessor#postProcessBeanFactory 方法
   * 
   */
  public interface BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor {
  
      /**
       * 1\. 该方法先于 BeanDefinitionRegistryPostProcessor#postProcessBeanFactory 执行
       * 2\. 传入参数为 registry，实际类是 DefaultListableBeanFactory，也可以使用 beanFactory 的操作
       */
      void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 
              throws BeansException;
  
  }
  
  ```

`BeanFactoryPostProcessor` 在 `AbstractApplicationContext#invokeBeanFactoryPostProcessors` 方法中被执行，执行时先执行 `BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry`，再执行 `BeanFactoryPostProcessor#postProcessBeanFactory`，关于这块的分析，可以参考 [spring 启动流程之执行 BeanFactoryPostProcessor](https://my.oschina.net/funcy/blog/4641114) 一文。

### 2. `BeanFactoryPostProcessor` 提供的功能

本节我们来介绍 `BeanFactoryPostProcessor` 提供的功能。

#### 2.1 `BeanFactoryPostProcessor` 的作用

探讨前，我们先来了解下 `BeanFactoryPostProcessor` 提供方法：

```
void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) 
        throws BeansException;

```

该方法中只有一个参数：`ConfigurableListableBeanFactory`，我们想了解 `BeanFactoryPostProcessor` 能为我们做什么，需要知道这个参数提供了哪些功能：

我们先来看看它的 `set` 方法： ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-63b39c81bcae0b10c60a2f847c6b47af932.png)

除此之外，还有 `register` 方法： ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-50c48da9b50dcf18abcd99db09142644c6c.png)

有了这些方法，我们就可以定制化 `beanFactory` 的一些行为了。

#### 2.2 `BeanDefinitionRegistryPostProcessor` 的作用

我们也来了解下 `BeanDefinitionRegistryPostProcessor` 提供的方法：

```
void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) 
        throws BeansException;

```

这个方法的参数是 `BeanDefinitionRegistry`，从字面意义来看，这是个 `BeanDefinition 注册器`，它提供了如下方法：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-5812a2cac994c5940d57c7e6ab55c23a63e.png)

可以看到，这个参数主要就是围绕 `BeanDefinition` 来操作，比较重要的方法列举如下：

*   `BeanDefinitionRegistry#containsBeanDefinition`：是否包含指定名称的 `BeanDefinition`
*   `BeanDefinitionRegistry#getBeanDefinition`：获取指定名称的 `BeanDefinition`
*   `BeanDefinitionRegistry#registerBeanDefinition`：注册一个 `BeanDefinition`
*   `BeanDefinitionRegistry#removeBeanDefinition`：移除 `BeanDefinition`

### 3\. spring 提供的 `BeanFactoryPostProcessor`

spring 中一共有两个 `BeanFactoryPostProcessor` 的实现类，相关信息及作用如下：

*   `EventListenerMethodProcessor`：

    *   实现了 `BeanFactoryPostProcessor`，`SmartInitializingSingleton` 接口
    *   在 `BeanDefinitionRegistryPostProcessor#postProcessBeanFactory` 方法中获取了 `EventListenerFactory`
    *   在 `SmartInitializingSingleton#afterSingletonsInstantiated` 方法中处理 `@EventListener` 注解
*   `ConfigurationClassPostProcessor`：

    *   在 `AnnotationConfigUtils#registerAnnotationConfigProcessors(BeanDefinitionRegistry, Object)` 方法中注册
    *   实现了 `BeanDefinitionRegistryPostProcessor` 接口
    *   处理 `@Conditional` 注解
    *   处理 `@Component` 注解
    *   处理 `@PropertySource/@PropertySources` 注解
    *   处理 `@ComponentScan/@ComponentScans` 注解
    *   处理 `@Import` 注解
    *   处理 `@ImportResource` 注解
    *   处理 `@Bean` 注解
    *   处理 `@Configuration` 注解

关于 `EventListenerMethodProcessor` 处理 `@EventListener` 的分析，可以参考[【spring 源码分析】spring 探秘之监听器注解 @EventListener](https://my.oschina.net/funcy/blog/4926344).

关于 `ConfigurationClassPostProcessor` 处理各注解的流程，可以参考：

*   [ConfigurationClassPostProcessor（一）：处理 @ComponentScan 注解](https://my.oschina.net/funcy/blog/4836178)
*   [ConfigurationClassPostProcessor（二）：处理 @Bean 注解](https://my.oschina.net/funcy/blog/4492878)
*   [ConfigurationClassPostProcessor（三）：处理 @Import 注解](https://my.oschina.net/funcy/blog/4678152)
*   [ConfigurationClassPostProcessor（四）：处理 @Conditional 注解](https://my.oschina.net/funcy/blog/4873444)

### 4\. 总结

本文介绍了 `BeanFactoryPostProcessor` 的概念，举例说明了 `BeanFactoryPostProcessor` 的使用，以及介绍了 spring 提供的两个 `BeanFactoryPostProcessor` 实现类：`EventListenerMethodProcessor` 与 `ConfigurationClassPostProcessor`。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4597545](https://my.oschina.net/funcy/blog/4597545) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_