### 1\. 什么是循环依赖？

spring 在依赖注入时，可能会出现相互注入的情况：

```
@Service
public class Service1 {
    @Autowired
    private Service2 service2;

}

@Service
public class Service2 {
    @Autowired
    private Service1 service1;

}

```

如以上代码，在 `Service1` 中通过 `@Autowird` 注入了 `Service2`，在 `Service2` 中通过 `@Autowird` 注入了 `Service1`，这种相互注入的情况，就叫做循环依赖。

### 2\. 循环依赖会有什么问题

实际上，这种 `A持有B对象，B也持有A对象`的情况，java 代码是完全支持的：

```
/**
 * 准备service1
 */
public class Service1 {
    private Service2 service2;

    public void setService2(Service2 service2) {
        this.service2 = service2;
    }

    public Service2 getService2() {
        return this.service2;
    }
}

/**
 * 准备service2
 */
public class Service2 {
    private Service1 service1;

    public void setService1(Service1 service1) {
        this.service1 = service1;
    }

    public Service1 getService1() {
        return this.service1;
    }
}

/**
 * 主方法中调用
 */
public class Main {
    public void main(String[] args) {
        // 准备两个对象
        Service1 service1 = new Service1();
        Service2 service2 = new Service2();
        // 相互设置
        service1.setService2(service2);
        service2.setService1(service1);
    }
}

```

那么，在 spring 中，两个类相互注入对方实例的情况，会有什么问题呢？我们来看 `spring bean` 的创建过程（**注意：这里我们仅分析 `bean` 的 `scope` 为 `singleton` 的情况，也就是 `scope` 为`单例`的情况**）：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-ca36e17077b1b191834645b3c8e588ff6c4.png)

这个过程中有几点需要说明下：

1.  创建对象：这个其实就是使用 jdk 提供的反射机制创建 java 对象，以第 1 节提到的 `Service1` 为例，可简单理解为 `Service1 service = new Service1()`；
2.  注入依赖对象：还是以第 1 节提到的 `Service1` 为例，`Service1` 中通过 `@Autowired` 自动注入 `Service2`，这一步就是给 `Service2` 赋值的过程，可简单理解为 `service1.setService2(service2)`；
3.  `singletonObjects`：经过上面两步后，一个 java 对象就变成了一个 spring bean，然后保存到 `singletonObjects` 了，这是个 `map`，`key` 是 bean 的名称，`value` 是 bean，它只保存 `spring bean`，不会只在 java 实例。

实际上，`java` 对象变成 `spring bean`，不仅仅只是依赖注入，还有初始化、执行 `beanPorcessor` 方法等，**由于本文是分析 `spring bean` 的循环依赖的，因此我们重点关注与循环依赖相关的步骤。**

#### 2.1 循环依赖产生的问题

了解了 spring bean 的产生过程之后，接下来我们就来分析下循环依赖产生的问题，在正式分析前，我们先来明确两个概念：

*   `java对象`：实际上，java 中一切对象都可以称之为 `java` 对象，为了说明方便，以下提到的 `java对象`仅指实例化完成、但未进行 spring bean 的生命周期对象；
*   `spring bean`：是一个 java 对象，并且进行了完整的 spring bean 的生命周期对象；

spring bean 的创建过程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b55a211447b5fabeaa0c3ef0bfee0920c82.png)

对上图说明如下：

1.  在 `service1` 对象创建完成后，`spring` 发现 `service1` 需要注入 `service2`，然后就去 `singletonObjects` 中查找 `service2`，此时是找不到 `service2`，然后就开始了 `service2` 的创建过程；
2.  在 `service2` 对象创建完成后，`spring` 发现 `service2` 需要注入 `service1`，然后就去 `singletonObjects` 中查找 `service1`，此时是找不到 `service1`，因为第一步中 `service1` 并没有创建成功 ，然后就开始了 `service1` 的创建过程；
3.  流程跳回到 `1`，再次开始了 `service1` 的创建、属性注入过程。

到这里，我们惊喜地发现，循环出现了！

#### 2.2 引入 `earlySingletonObjects` 解决循环依赖

我们分析下，循环出现的原因在于，在 `service2` 获取 `service1` 时，由于 `singletonObjects` 中此时并不存在 `service1`，因此会再走 `service1` 的创建过程，重新创建 `service1`，因此，我们有个大胆的想法：如果在 `service1` 实例化后就把它保存起来，后面再再找 `service1` 时，就返回这个未进行依赖注入的 `service1`，像下面这样：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-c0eaecbe82b144a6fcd9fd048f2cca53497.png)

上图中，引入了 `earlySingletonObjects`，这也是个 map，同 `singletonObjects` 一样，`key` 是 bean 的名称，`value` 是一个未完成依赖注入的对象。

对上图说明如下：

1.  在 `service1` 对象创建完成后，先将 `service1` 放入 `earlySingletonObjects`，然后进行依赖注入；
2.  对 `service1` 进行依赖注入时，`spring` 发现 `service1` 需要注入 `service2`，然后先去 `earlySingletonObjects` 查找 `service2`，未找到；再去 `singletonObjects` 中查找 `service2`，还是未找到，于是就开始了 `service2` 的创建过程；
3.  在 `service2` 对象创建完成后，先将 `service2` 放入 `earlySingletonObjects`，然后进行依赖注入；
4.  对 `service2` 进行依赖注入时，`spring` 发现 `service2` 需要注入 `service1`，然后就去 `earlySingletonObjects` 查找 `service1`，找到了，就将 `service1` 注入到 `service2` 中，此时 `service2` 就是一个 `spring bean` 了，将其保存到 `singletonObjects` 中；
5.  经过第 4 步后，我们得到了 `service2`，然后将其注入到 `service1` 中，此时 `service1` 也成了一个 `spring bean`，将其保存到 `singletonObjects` 中。

经过以上步骤，我们发现，循环依赖得到了解决。

#### 2.2 aop 下的循环依赖

经过上面的分析，我们发现只要额外引入一个 `earlySingletonObjects` 后，循环依赖就能得到解决。但是，循环依赖真的得到了解决吗？spring 除了 ioc 外，还有另一个重大功能：aop，我们来看看 aop 情况下出现循环依赖会怎样。

##### 1\. aop 对象的创建过程

在正式介绍 aop 下的循环依赖前，我们先来明确两个个概念：

*   `原始对象`：区别于代理对象，指未进行过 aop 的对象，可以是 java 对象，也可以是未进行 aop 的 spring bean；
*   `代理对象`：进行过 aop 的对象，可以是 java 对象仅进行过 aop 得到的对象 (仅进行过 aop，未进行依赖注入，也未进行初始化)，也可以是进行过 aop 的 `spring bean`.

我们先来看看 aop 是如何创建对象的：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-e92991c1c173bbd5579be3001a977555be7.png)

相比于 `2.1` 中的流程，aop 多了 "生成代理对象" 的操作，并且最终保存到 `singletonObjects` 中的对象也是代理对象。

原始对象与代理对象之间是什么关系呢？用代码示意下，大致如下：

```
public class ProxyObj extends Obj {

    // 原始对象
    private Obj obj;

    ...
}

```

实际上，两者之间的关系并没有这么简单，但为了说明问题，这里对两者关系做了简化，小伙伴们只需要明白，**代理对象持有原始对象的引用**即可。

关于原始对象如何变成代理对象的，可以参考 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)。

对以上创建过程，用 java 代码模拟如下：

```
/**
 * 准备一个类
 */
public class Obj1 {

}

/**
 * 准备一个类，内部有一个属性 Obj1
 */
public class Obj2 {

    private Obj1 obj1;

    // 省略其他方法
    ...

}

/**
 * 准备Obj2的代理类，内部持有obj2的对象
 */
public class ProxyObj2 extends Obj2 {

    private Obj2 obj2;

    public ProxyObj2(Obj2 obj2) {
        this.obj2 = obj2;
    }

    // 省略其他方法
    ...

}

```

接着，就是模拟 “创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中” 的 流程了：

```
public static main(String[] args) {
     // 准备一个容器，这里保存的是完成上述生命周期的对象
     // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
     // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
     Collection<?> collection = new ArrayList();

     // 开始 Obj2 的创建流程
     // 1\. 创建 Obj2 对象
     Obj2 obj2 = new Obj2();

     // 2\. 往 Obj2 中注入 obj1，但此时并没有obj1，因此先要创建obj1，再将其注入到Obj2中
     Obj1 obj1 = new Obj1();
     obj2.setObj1(obj1);

     // 3\. 生成Obj2的代理对象，代理对象中持有 Obj2的原始对象
     ProxyObj2 proxyObj2 = new ProxyObj2(obj2);

     // 4\. proxyObj2已经走完了完整的生命周期，因此将代理对象添加到容器时
     collection.add(proxyObj2); 

}

```

上述代码中，

*   以 `new Obj2()` 模拟对象的创建
*   以 `obj2.setObj1(xxx)` 模拟依赖注入
*   以 `new ProxyObj2(xxx)` 模拟代理对象的生成
*   以 `collection.add(xxx)` 模拟对象添加到容器中的过程

模拟的流程如下：

1.  创建 `obj2` 对象
2.  往 `Obj2` 中注入 `obj1`，但此时并没有 `obj1`，因此先要创建 `obj1`，再将其注入到 `Obj2` 中
3.  生成 `Obj2` 的代理对象 `proxyObj2`，`proxyObj2` 中持有 `Obj2` 的原始对象
4.  `proxyObj2` 已经走完了完整的生命周期，因此将代理对象添加到容器时

仔细分析上面的步骤，就会发现，上面的第 2 步与第 3 步完全调换顺序也没问题，代码模拟如下：

```
public static main(String[] args) {
     // 准备一个容器，这里保存的是完成上述生命周期的对象
     // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
     // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
     Collection<?> collection = new ArrayList();

     // 开始 Obj2 的创建流程
     // 1\. 创建 Obj2 对象
     Obj2 obj2 = new Obj2();

     // 2\. 生成Obj2的代理对象，代理对象中持有 Obj2的原始对象
     ProxyObj2 proxyObj2 = new ProxyObj2(obj2);

     // 3\. 往 obj2 中注入 obj1，但此时并没有obj1，因此先要创建obj1，再将其注入到Obj2中
     Obj1 obj1 = new Obj1();
     // 这里是注入到原始对象中
     obj2.setObj1(obj1);

     // 4\. proxyObj2已经走完了完整的生命周期，因此将代理对象添加到容器时
     collection.add(proxyObj2); 

}

```

上述代码的流程如下：

1.  创建 obj2 对象
2.  生成 Obj2 的代理对象，代理对象中持有 Obj2 的原始对象
3.  往 Obj2 中注入 obj1，但此时并没有 obj1，因此先要创建 obj1，再将其注入到 Obj2
4.  proxyObj2 已经走完了完整的生命周期，因此将代理对象添加到容器时

从代码上看，`proxyObj2(代理对象)` 中持有 `ob2(原始对象)`，生成代理对象后，继续对原始对象进行属性注入，依然能影响代理对象，最终代理对象持有的原始对象也完成了依赖注入，整个过程用图形示意如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8ff5579425f73dd5c2d321a86d0303390ac.png)

这里我们再次申明，从 java 对象到 spring bean 的步骤有好多，这里我们仅关注与循环依赖相关的步骤，如果想了解 spring bean 详细的初始化过程，可查看 [spring 启动流程之启动流程概览](https://my.oschina.net/funcy/blog/4597493)。

到这里，我们探索到代理对象的生命周期可以有两种：

*   创建 --> 属性注入 --> 生成代理对象 --> 将代理对象保存到容器中
*   创建 (原始对象)--> 生成代理对象 (提前进行 aop)--> 对原始对象进行属性注入 --> 将代理对象保存到容器中

这两种都能达到最终目的，即**保存到容器中的是代理对象，且代理对象对应的原始对象完成了依赖注入**。请牢记这两个创建流程，这是后面解决 aop 下循环依赖问题的核心，说白了，**aop 下的循环依赖问题之所以能解决，就是因为对象可以提前进行 aop 操作**。

##### 2\. 为什么用 `earlySingletonObjects` 无法解决循环依赖？

前面我们主要说明了代理对象的创建过程，接下来我们来看看在 aop 下，使用 `earlySingletonObjects` 来解决循环依赖有什么问题：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d3d00d76ab0c72339faccb7ccd853723d6c.png)

我们来分析上图的流程：

1.  在 `service1` 对象创建完成后，先将 `service1` 放入 `earlySingletonObjects`，然后进行依赖注入；
2.  对 `service1` 进行依赖注入时，`spring` 发现 `service1` 需要注入 `service2`，然后先去 `earlySingletonObjects` 查找 `service2`，未找到；再去 `singletonObjects` 中查找 `service2`，还是未找到，于是就开始了 `service2` 的创建过程；
3.  在 `service2` 对象创建完成后，先将 `service2` 放入 `earlySingletonObjects`，然后进行依赖注入；
4.  对 `service2` 进行依赖注入时，`spring` 发现 `service2` 需要注入 `service1`，然后就去 `earlySingletonObjects` 查找 `service1`，找到了，就将 `service1` 注入到 `service2` 中，然后再进行 aop，此时 `service2` 是一个代理对象，将其保存到 `singletonObjects` 中；
5.  经过第 4 步后，我们得到了 `service2` 的代理对象，然后将其注入到 `service1` 中，接着再对 `service1` 进行 aop，此时 `service1` 也成了一个 `spring bean`，将其保存到 `singletonObjects` 中。

上述步骤有什么问题呢？仔细看第 4 步，就会发现，**注入到 `service2` 的 `service1` 并不是代理对象**！纵观全局，最终得到的 `service1` 与 `service2` 都是代理对象，注入到 `service2` 的 `service1` 应该也是代理对象才对。因此，在 aop 下，循环依赖的问题又出现了！

#### 2.3 spring 的解决方案

前面我们提到，在 aop 下，引入 `earlySingletonObjects` 并不能解决循环依赖的问题，那 spring 是怎么解决的呢？spring 再次引入了一个 `map` 来解决这个问题，这也是人们常说的 **spring 三级缓存**，对这三个 `map` 说明如下：

*   一级缓存 `singletonObjects`：类型为 `ConcurrentHashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是完整的 `spring bean`，即完成属性注入、初始化的 bean，如果 bean 需要 aop，存储的就是代理对象；
*   二级缓存 `earlySingletonObjects`：类型为 `HashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是实例化完成，但未进行依赖注入的 `bean`，如果 `bean` 需要 `aop`，这里存储的就是代理对象，只不过代理对象所持有的原始对象并未进行依赖注入；
*   三级缓存 `singletonFactories`：类型为 `HashMap<String, ObjectFactory>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 存储的是一个 `lambda` 表达式：`() -> getEarlyBeanReference(beanName, mbd, bean)`， `getEarlyBeanReference` 中的 `bean` 是刚创建完成的 `java bean`，没有进行 spring 依赖注入，也没进行 aop (关于这个 `lambda` 表达式，后面会继续分析)。

为了说明方便，下面对 `singletonObjects`、`earlySingletonObjects` 和 `singletonFactories` 分别称为**一级缓存**、**二级缓存**和**三级缓存**。

spring 解决 aop 下的循环依赖流程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-dc325a87b321e4c246a1b2f14169821a75a.png)

这个图看着比较复杂，其实分开来看就比较简单了，上述操作中，`1~8` 是获取 `service1` 的流程，`5.1~5.8` 是获取 `service2` 的流程，`5.5.1` 是再次获取 `service1` 的流程，只不过在处理 `service1` 的初始化过程中，会触发 `service2` 的初始化过程，而 `service2` 的初始化时，又会依赖到 `service1`，因此才看着像是连在一起，比较复杂。

对上图的过程，这里说明如下（建议：如果觉得流程比较复杂，可以先看 `1~8` 的操作，再看 `5.1~5.8` 的操作，最后两者联合起来看，这样会清晰很多）：

*   1.  `service1`：获取 `service1`，从一级缓存中获取，此时是获取不到的；
*   1.  `service1`：创建 `service1` 的实例；
*   1.  `service1`：获取需要注入的属性与方法（在原始对象上进行获取）；
*   1.  `service1`：如果开启了支持循环依赖的配置，就将 `service1` 放到三级缓存中（是否支持循环依赖，是可以配置的）；
*   1.  `service1`：对 `service1` 进行依赖注入，需要 `service2`，然后就开始了 `service2` 的获取流程；
*   5.1 `service2`：获取 `service2`，从一级缓存中获取，此时是获取不到的；
*   5.2 `service2`：创建 `service2` 的实例；
*   5.3 `service2`：获取需要注入的属性与方法（在原始对象上进行获取）；
*   5.4 `service2`：如果开启了支持循环依赖的配置，就将 `service2` 放到三级缓存中（是否支持循环依赖，是可以配置的）；
*   5.5 `service2`：对 `service2` 进行依赖注入，需要 `service1`，然后就开始了 `service1` 的获取流程；
*   5.5.1 `service1`: 获取 `service1`，从一级缓存中获取，获取不到；此时发现 `service1` 正在创建中，于是继续从二、三级缓存中获取，最终从三级缓存中获取到了，将其放入二级缓存。从三级缓存获取的过程中，**会判断 `service1` 是否需要进行 aop，然后开始 aop 操作**，因此放入二级缓存中的是 `service1` 代理代理，提前进行 aop 是解决循环依赖的关键；
*   5.6 `service2`：得到了 `service1` 后（这里的 `service1` 是代理对象），将其注入到 `service2` 中，接着对 `service2` 进行 aop，得到 `service2` 的代理对象；
*   5.7 `service2`：如果支持循环依赖，先从一、二级缓存中再次获取 `service2`，都未获取到，就使用当前 `service2`（当前 `service2` 是代理对象)；
*   5.8 `service2`：将 service2 的代理对象放入一级缓存中，删除二、三级缓存，至此，`service2` 初始化完成，注入的 `service1` 是代理对象，一级缓存中的 `service2` 也是代理对象；
*   1.  `service1`：回到 `service1` 的生命周期，拿到 `service2`（这里的 `service2` 是代理对象）后，将其注入到 `service1`，`service1` 的依赖注入完成，进行初始化，这里会判断 `service1` 是否需要进行 aop，虽然 `service1` 是需要进行 aop 的，但由于在 `5.5.1` 已经进行过 aop 了，因此，这里直接返回（到这一步，`service1` 还是原始对象）；
*   1.  `service1`：如果支持循环依赖，先从一级缓存中获取 `service1`，获取不到；再从二缓存中获取 `service1`，可以获取到（从 `5.5.1` 可知，二级缓存里是 `service1` 代理对象），返回；
*   1.  `service1`：将二级缓存中获取的对象注册到一级缓存中，删除二、三级缓存，至此，`service1` 初始化完成，注入的 `service2` 是代理对象，一级缓存中的 `service1` 也是代理对象。

以上流程，虽然步骤较多，但 `service1` 与 `service2` 的获取步骤是相同的，只要弄清了其中之一的获取流程，另一个 bean 的获取流程就很雷同了。

在上述流程中，还有两个数据结构需要说明下：

*   `singletonsCurrentlyInCreation`：类型为 `SetFromMap<String>`，位于 `DefaultSingletonBeanRegistry`，创建方式为 `Collections.newSetFromMap(new ConcurrentHashMap<>(16))`，表明这是个由 `ConcurrentHashMap` 实现的 set，存储的是正在创建中的对象，**判断当前对象是否在创建中就是通过查找当前对象是否在这个 set 中**做到的；
*   `earlyProxyReferences`：类型为 `ConcurrentHashMap<Object, Object>`，位于 `AbstractAutoProxyCreator`，存储的是提前进行 aop 的对象，**如果一个对象提前进行了 aop，在后面再次 aop 时，会通过判断对象是否在 `earlyProxyReferences` 中而确定要不要进行 aop，以此来保证每个对象只进行一次 aop**。

至此，spring 一共提供了 5 个数据结构来辅助解决循环依赖问题，总结如下：

| 结构                            | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `singletonObjects`              | **一级缓存**，类型为 `ConcurrentHashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是完整的 `spring bean`，即完成属性注入、初始化的 bean，如果 bean 需要 aop，存储的就是代理对象 |
| `earlySingletonObjects`         | **二级缓存**，类型为 `HashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是实例化完成，但未进行依赖注入的 `bean`，**如果 `bean` 需要 `aop`，这里存储的就是代理对象，只不过代理对象所持有的原始对象并未进行依赖注入** |
| `singletonFactories`            | **三级缓存**，类型为 `HashMap<String, ObjectFactory>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 存储的是一个 `lambda` 表达式：`() -> getEarlyBeanReference(beanName, mbd, bean)`，`getEarlyBeanReference(xxx)` 中的 `bean` 是刚创建完成的 `java bean`，没有进行 spring 依赖注入，也没进行 aop |
| `singletonsCurrentlyInCreation` | 类型为 `SetFromMap<String>`，位于 `DefaultSingletonBeanRegistry`，创建方式为 `Collections.newSetFromMap(new ConcurrentHashMap<>(16))`，表明这是个由 `ConcurrentHashMap` 实现的 set，存储的是正在创建中的对象，可以**用来判断当前对象是否在创建中** |
| `earlyProxyReferences`          | 类型为 `ConcurrentHashMap<Object, Object>`，位于 `AbstractAutoProxyCreator`，存储的是提前进行 aop 的对象，可以**用来判断 bean 是否进行过 aop，保证每个对象只进行一次 aop** |

以上就是 spring 解决循环依赖的完整流程了。

### 3\. 代码模拟

在正式分析源码前，我们首先模拟循环下依赖解决的过程，代码如下：

```
/**
 * 准备一个类，内部有一个属性 Obj2
 */
public class Obj1 {
    // 需要注入 obj2
    private Obj2 obj2;

    // 省略其他方法
    ...
}

/**
 * 准备一个类，内部有一个属性 Obj1
 */
public class Obj2 {
    // 需要注入 ob1
    private Obj1 obj1;

    // 省略其他方法
    ...

}

/**
 * 准备Obj2的代理类，内部持有obj2的对象
 */
public class ProxyObj2 extends Obj2 {
    // obj2代理类内部持有obj2的原始对象
    private Obj2 obj2;

    public ProxyObj2(Obj2 obj2) {
        this.obj2 = obj2;
    }

    // 省略其他方法
    ...

}

/**
 * 准备Obj1的代理类，内部持有obj1的对象
 */
public class ProxyObj1 extends Obj1 {
    // obj2代理类内部持有obj1的原始对象
    private Obj1 obj1;

    public ProxyObj1(Obj1 obj1) {
        this.obj1 = obj1;
    }

    // 省略其他方法
    ...

}

```

*   首先准备了两个类：`Obj1` 与 `Obj2`， 其中 `Obj1` 有个属性为 `Obj2`，`Obj2` 中有个属性为 `Obj1`；
*   接着准备了 `Obj1` 与 `Obj2` 的代理类 `ProxyObj1`、`ProxyObj2`，并且 `ProxyObj1`、`ProxyObj2` 分别有一个属性：`Obj1` 、 `Obj2`；
*   我们依旧以 `new ObjX()` 模拟对象的创建；
*   我们依旧以 `objX.setObjX(xxx)` 模拟依赖注入；
*   我们依旧以 `new ProxyObjX(xxx)` 模拟代理对象的生成；
*   我们依旧以 `collection.add(xxx)` 模拟对象添加到容器中的过程；

我们模拟最终得到的结果为：

*   最终放入容器的对象分别是 `proxyObj1`，`proxyObj2`
*   注入到 `obj1` 中的是 `proxyObj2`，注入到 `obj2` 中的是 `proxyObj2`

准备工作已经完成了，接下来我们就开始进行模拟了。

#### 3.1 模拟 1

要求：

*   Obj1 与 Obj2 必须严格按照 “创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中” 的流程创建
*   两个对象的创建流程可以交替进行

目标：

*   最终放入容器的对象分别是 `proxyObj1`，`proxyObj2`
*   注入到 `obj1` 中的是 `proxyObj2`，注入到 `obj2` 中的是 `proxyObj2`

代码如下：

```
public static main(String[] args) {
     // 准备一个容器，这里保存的是完成上述生命周期的对象
     // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
     // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
     Collection<?> collection = new ArrayList();

     // 1\. 创建 Obj1 对象
     Obj1 obj1 = new Obj1();

     // 接下来需要将obj2的代理对象注入到obj1中，但此时容器中并没有obj2的代理对象，于是切换到obj2的创建流程
     // 一. 创建 Obj2 对象
     Obj2 obj2 = new Obj2();

     // 到这里，obj2需要注入obj1的代理对象，但此时容器中并没有obj2的代理对象，于是又要切到obj1的创建流程

}

```

在执行以上流程中 ，发现创建 Obj2 对象后，流程就进行不下去了：

*   `obj1` 需要注入 `obj2` 的代理对象，但找不到，于是切换到 `obj2` 的创建流程；
*   `obj2` 需要注入 `obj1` 的代理对象，但找不到，于是切换到 `obj1` 的创建流程；
*   `obj1` 需要注入 `obj2` 的代理对象，但找不到，于是切换到 `obj2` 的创建流程；
*   ...

如此循环往复。

模拟结果：未达到预期目标，本次模拟宣告失败。

#### 3.1 模拟 2

要求：

*   Obj1 与 Obj2 必须以下两种流程之一创建：
    *   “创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中” 的流程创建
    *   “创建 (原始对象)--> 生成代理对象 --> 对原始对象进行属性注入 --> 将代理对象保存到容器中” 的流程创建
*   两个对象的创建流程可以交替进行

目标：

*   最终放入容器的对象分别是 `proxyObj1`，`proxyObj2`
*   注入到 `obj1` 中的是 `proxyObj2`，注入到 `obj2` 中的是 `proxyObj2`### 1\. 什么是循环依赖？

spring 在依赖注入时，可能会出现相互注入的情况：

```
@Service
public class Service1 {
    @Autowired
    private Service2 service2;

}

@Service
public class Service2 {
    @Autowired
    private Service1 service1;

}

```

如以上代码，在 `Service1` 中通过 `@Autowird` 注入了 `Service2`，在 `Service2` 中通过 `@Autowird` 注入了 `Service1`，这种相互注入的情况，就叫做循环依赖。

### 2\. 循环依赖会有什么问题

实际上，这种 `A持有B对象，B也持有A对象`的情况，java 代码是完全支持的：

```
/**
 * 准备service1
 */
public class Service1 {
    private Service2 service2;

    public void setService2(Service2 service2) {
        this.service2 = service2;
    }

    public Service2 getService2() {
        return this.service2;
    }
}

/**
 * 准备service2
 */
public class Service2 {
    private Service1 service1;

    public void setService1(Service1 service1) {
        this.service1 = service1;
    }

    public Service1 getService1() {
        return this.service1;
    }
}

/**
 * 主方法中调用
 */
public class Main {
    public void main(String[] args) {
        // 准备两个对象
        Service1 service1 = new Service1();
        Service2 service2 = new Service2();
        // 相互设置
        service1.setService2(service2);
        service2.setService1(service1);
    }
}

```

那么，在 spring 中，两个类相互注入对方实例的情况，会有什么问题呢？我们来看 `spring bean` 的创建过程（**注意：这里我们仅分析 `bean` 的 `scope` 为 `singleton` 的情况，也就是 `scope` 为`单例`的情况**）：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-ca36e17077b1b191834645b3c8e588ff6c4.png)

这个过程中有几点需要说明下：

1.  创建对象：这个其实就是使用 jdk 提供的反射机制创建 java 对象，以第 1 节提到的 `Service1` 为例，可简单理解为 `Service1 service = new Service1()`；
2.  注入依赖对象：还是以第 1 节提到的 `Service1` 为例，`Service1` 中通过 `@Autowired` 自动注入 `Service2`，这一步就是给 `Service2` 赋值的过程，可简单理解为 `service1.setService2(service2)`；
3.  `singletonObjects`：经过上面两步后，一个 java 对象就变成了一个 spring bean，然后保存到 `singletonObjects` 了，这是个 `map`，`key` 是 bean 的名称，`value` 是 bean，它只保存 `spring bean`，不会只在 java 实例。

实际上，`java` 对象变成 `spring bean`，不仅仅只是依赖注入，还有初始化、执行 `beanPorcessor` 方法等，**由于本文是分析 `spring bean` 的循环依赖的，因此我们重点关注与循环依赖相关的步骤。**

#### 2.1 循环依赖产生的问题

了解了 spring bean 的产生过程之后，接下来我们就来分析下循环依赖产生的问题，在正式分析前，我们先来明确两个概念：

*   `java对象`：实际上，java 中一切对象都可以称之为 `java` 对象，为了说明方便，以下提到的 `java对象`仅指实例化完成、但未进行 spring bean 的生命周期对象；
*   `spring bean`：是一个 java 对象，并且进行了完整的 spring bean 的生命周期对象；

spring bean 的创建过程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-b55a211447b5fabeaa0c3ef0bfee0920c82.png)

对上图说明如下：

1.  在 `service1` 对象创建完成后，`spring` 发现 `service1` 需要注入 `service2`，然后就去 `singletonObjects` 中查找 `service2`，此时是找不到 `service2`，然后就开始了 `service2` 的创建过程；
2.  在 `service2` 对象创建完成后，`spring` 发现 `service2` 需要注入 `service1`，然后就去 `singletonObjects` 中查找 `service1`，此时是找不到 `service1`，因为第一步中 `service1` 并没有创建成功 ，然后就开始了 `service1` 的创建过程；
3.  流程跳回到 `1`，再次开始了 `service1` 的创建、属性注入过程。

到这里，我们惊喜地发现，循环出现了！

#### 2.2 引入 `earlySingletonObjects` 解决循环依赖

我们分析下，循环出现的原因在于，在 `service2` 获取 `service1` 时，由于 `singletonObjects` 中此时并不存在 `service1`，因此会再走 `service1` 的创建过程，重新创建 `service1`，因此，我们有个大胆的想法：如果在 `service1` 实例化后就把它保存起来，后面再再找 `service1` 时，就返回这个未进行依赖注入的 `service1`，像下面这样：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-c0eaecbe82b144a6fcd9fd048f2cca53497.png)

上图中，引入了 `earlySingletonObjects`，这也是个 map，同 `singletonObjects` 一样，`key` 是 bean 的名称，`value` 是一个未完成依赖注入的对象。

对上图说明如下：

1.  在 `service1` 对象创建完成后，先将 `service1` 放入 `earlySingletonObjects`，然后进行依赖注入；
2.  对 `service1` 进行依赖注入时，`spring` 发现 `service1` 需要注入 `service2`，然后先去 `earlySingletonObjects` 查找 `service2`，未找到；再去 `singletonObjects` 中查找 `service2`，还是未找到，于是就开始了 `service2` 的创建过程；
3.  在 `service2` 对象创建完成后，先将 `service2` 放入 `earlySingletonObjects`，然后进行依赖注入；
4.  对 `service2` 进行依赖注入时，`spring` 发现 `service2` 需要注入 `service1`，然后就去 `earlySingletonObjects` 查找 `service1`，找到了，就将 `service1` 注入到 `service2` 中，此时 `service2` 就是一个 `spring bean` 了，将其保存到 `singletonObjects` 中；
5.  经过第 4 步后，我们得到了 `service2`，然后将其注入到 `service1` 中，此时 `service1` 也成了一个 `spring bean`，将其保存到 `singletonObjects` 中。

经过以上步骤，我们发现，循环依赖得到了解决。

#### 2.2 aop 下的循环依赖

经过上面的分析，我们发现只要额外引入一个 `earlySingletonObjects` 后，循环依赖就能得到解决。但是，循环依赖真的得到了解决吗？spring 除了 ioc 外，还有另一个重大功能：aop，我们来看看 aop 情况下出现循环依赖会怎样。

##### 1\. aop 对象的创建过程

在正式介绍 aop 下的循环依赖前，我们先来明确两个个概念：

*   `原始对象`：区别于代理对象，指未进行过 aop 的对象，可以是 java 对象，也可以是未进行 aop 的 spring bean；
*   `代理对象`：进行过 aop 的对象，可以是 java 对象仅进行过 aop 得到的对象 (仅进行过 aop，未进行依赖注入，也未进行初始化)，也可以是进行过 aop 的 `spring bean`.

我们先来看看 aop 是如何创建对象的：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-e92991c1c173bbd5579be3001a977555be7.png)

相比于 `2.1` 中的流程，aop 多了 "生成代理对象" 的操作，并且最终保存到 `singletonObjects` 中的对象也是代理对象。

原始对象与代理对象之间是什么关系呢？用代码示意下，大致如下：

```
public class ProxyObj extends Obj {

    // 原始对象
    private Obj obj;

    ...
}

```

实际上，两者之间的关系并没有这么简单，但为了说明问题，这里对两者关系做了简化，小伙伴们只需要明白，**代理对象持有原始对象的引用**即可。

关于原始对象如何变成代理对象的，可以参考 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)。

对以上创建过程，用 java 代码模拟如下：

```
/**
 * 准备一个类
 */
public class Obj1 {

}

/**
 * 准备一个类，内部有一个属性 Obj1
 */
public class Obj2 {

    private Obj1 obj1;

    // 省略其他方法
    ...

}

/**
 * 准备Obj2的代理类，内部持有obj2的对象
 */
public class ProxyObj2 extends Obj2 {

    private Obj2 obj2;

    public ProxyObj2(Obj2 obj2) {
        this.obj2 = obj2;
    }

    // 省略其他方法
    ...

}

```

接着，就是模拟 “创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中” 的 流程了：

```
public static main(String[] args) {
     // 准备一个容器，这里保存的是完成上述生命周期的对象
     // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
     // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
     Collection<?> collection = new ArrayList();

     // 开始 Obj2 的创建流程
     // 1\. 创建 Obj2 对象
     Obj2 obj2 = new Obj2();

     // 2\. 往 Obj2 中注入 obj1，但此时并没有obj1，因此先要创建obj1，再将其注入到Obj2中
     Obj1 obj1 = new Obj1();
     obj2.setObj1(obj1);

     // 3\. 生成Obj2的代理对象，代理对象中持有 Obj2的原始对象
     ProxyObj2 proxyObj2 = new ProxyObj2(obj2);

     // 4\. proxyObj2已经走完了完整的生命周期，因此将代理对象添加到容器时
     collection.add(proxyObj2); 

}

```

上述代码中，

*   以 `new Obj2()` 模拟对象的创建
*   以 `obj2.setObj1(xxx)` 模拟依赖注入
*   以 `new ProxyObj2(xxx)` 模拟代理对象的生成
*   以 `collection.add(xxx)` 模拟对象添加到容器中的过程

模拟的流程如下：

1.  创建 `obj2` 对象
2.  往 `Obj2` 中注入 `obj1`，但此时并没有 `obj1`，因此先要创建 `obj1`，再将其注入到 `Obj2` 中
3.  生成 `Obj2` 的代理对象 `proxyObj2`，`proxyObj2` 中持有 `Obj2` 的原始对象
4.  `proxyObj2` 已经走完了完整的生命周期，因此将代理对象添加到容器时

仔细分析上面的步骤，就会发现，上面的第 2 步与第 3 步完全调换顺序也没问题，代码模拟如下：

```
public static main(String[] args) {
     // 准备一个容器，这里保存的是完成上述生命周期的对象
     // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
     // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
     Collection<?> collection = new ArrayList();

     // 开始 Obj2 的创建流程
     // 1\. 创建 Obj2 对象
     Obj2 obj2 = new Obj2();

     // 2\. 生成Obj2的代理对象，代理对象中持有 Obj2的原始对象
     ProxyObj2 proxyObj2 = new ProxyObj2(obj2);

     // 3\. 往 obj2 中注入 obj1，但此时并没有obj1，因此先要创建obj1，再将其注入到Obj2中
     Obj1 obj1 = new Obj1();
     // 这里是注入到原始对象中
     obj2.setObj1(obj1);

     // 4\. proxyObj2已经走完了完整的生命周期，因此将代理对象添加到容器时
     collection.add(proxyObj2); 

}

```

上述代码的流程如下：

1.  创建 obj2 对象
2.  生成 Obj2 的代理对象，代理对象中持有 Obj2 的原始对象
3.  往 Obj2 中注入 obj1，但此时并没有 obj1，因此先要创建 obj1，再将其注入到 Obj2
4.  proxyObj2 已经走完了完整的生命周期，因此将代理对象添加到容器时

从代码上看，`proxyObj2(代理对象)` 中持有 `ob2(原始对象)`，生成代理对象后，继续对原始对象进行属性注入，依然能影响代理对象，最终代理对象持有的原始对象也完成了依赖注入，整个过程用图形示意如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-8ff5579425f73dd5c2d321a86d0303390ac.png)

这里我们再次申明，从 java 对象到 spring bean 的步骤有好多，这里我们仅关注与循环依赖相关的步骤，如果想了解 spring bean 详细的初始化过程，可查看 [spring 启动流程之启动流程概览](https://my.oschina.net/funcy/blog/4597493)。

到这里，我们探索到代理对象的生命周期可以有两种：

*   创建 --> 属性注入 --> 生成代理对象 --> 将代理对象保存到容器中
*   创建 (原始对象)--> 生成代理对象 (提前进行 aop)--> 对原始对象进行属性注入 --> 将代理对象保存到容器中

这两种都能达到最终目的，即**保存到容器中的是代理对象，且代理对象对应的原始对象完成了依赖注入**。请牢记这两个创建流程，这是后面解决 aop 下循环依赖问题的核心，说白了，**aop 下的循环依赖问题之所以能解决，就是因为对象可以提前进行 aop 操作**。

##### 2\. 为什么用 `earlySingletonObjects` 无法解决循环依赖？

前面我们主要说明了代理对象的创建过程，接下来我们来看看在 aop 下，使用 `earlySingletonObjects` 来解决循环依赖有什么问题：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d3d00d76ab0c72339faccb7ccd853723d6c.png)

我们来分析上图的流程：

1.  在 `service1` 对象创建完成后，先将 `service1` 放入 `earlySingletonObjects`，然后进行依赖注入；
2.  对 `service1` 进行依赖注入时，`spring` 发现 `service1` 需要注入 `service2`，然后先去 `earlySingletonObjects` 查找 `service2`，未找到；再去 `singletonObjects` 中查找 `service2`，还是未找到，于是就开始了 `service2` 的创建过程；
3.  在 `service2` 对象创建完成后，先将 `service2` 放入 `earlySingletonObjects`，然后进行依赖注入；
4.  对 `service2` 进行依赖注入时，`spring` 发现 `service2` 需要注入 `service1`，然后就去 `earlySingletonObjects` 查找 `service1`，找到了，就将 `service1` 注入到 `service2` 中，然后再进行 aop，此时 `service2` 是一个代理对象，将其保存到 `singletonObjects` 中；
5.  经过第 4 步后，我们得到了 `service2` 的代理对象，然后将其注入到 `service1` 中，接着再对 `service1` 进行 aop，此时 `service1` 也成了一个 `spring bean`，将其保存到 `singletonObjects` 中。

上述步骤有什么问题呢？仔细看第 4 步，就会发现，**注入到 `service2` 的 `service1` 并不是代理对象**！纵观全局，最终得到的 `service1` 与 `service2` 都是代理对象，注入到 `service2` 的 `service1` 应该也是代理对象才对。因此，在 aop 下，循环依赖的问题又出现了！

#### 2.3 spring 的解决方案

前面我们提到，在 aop 下，引入 `earlySingletonObjects` 并不能解决循环依赖的问题，那 spring 是怎么解决的呢？spring 再次引入了一个 `map` 来解决这个问题，这也是人们常说的 **spring 三级缓存**，对这三个 `map` 说明如下：

*   一级缓存 `singletonObjects`：类型为 `ConcurrentHashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是完整的 `spring bean`，即完成属性注入、初始化的 bean，如果 bean 需要 aop，存储的就是代理对象；
*   二级缓存 `earlySingletonObjects`：类型为 `HashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是实例化完成，但未进行依赖注入的 `bean`，如果 `bean` 需要 `aop`，这里存储的就是代理对象，只不过代理对象所持有的原始对象并未进行依赖注入；
*   三级缓存 `singletonFactories`：类型为 `HashMap<String, ObjectFactory>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 存储的是一个 `lambda` 表达式：`() -> getEarlyBeanReference(beanName, mbd, bean)`， `getEarlyBeanReference` 中的 `bean` 是刚创建完成的 `java bean`，没有进行 spring 依赖注入，也没进行 aop (关于这个 `lambda` 表达式，后面会继续分析)。

为了说明方便，下面对 `singletonObjects`、`earlySingletonObjects` 和 `singletonFactories` 分别称为**一级缓存**、**二级缓存**和**三级缓存**。

spring 解决 aop 下的循环依赖流程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-dc325a87b321e4c246a1b2f14169821a75a.png)

这个图看着比较复杂，其实分开来看就比较简单了，上述操作中，`1~8` 是获取 `service1` 的流程，`5.1~5.8` 是获取 `service2` 的流程，`5.5.1` 是再次获取 `service1` 的流程，只不过在处理 `service1` 的初始化过程中，会触发 `service2` 的初始化过程，而 `service2` 的初始化时，又会依赖到 `service1`，因此才看着像是连在一起，比较复杂。

对上图的过程，这里说明如下（建议：如果觉得流程比较复杂，可以先看 `1~8` 的操作，再看 `5.1~5.8` 的操作，最后两者联合起来看，这样会清晰很多）：

*   1.  `service1`：获取 `service1`，从一级缓存中获取，此时是获取不到的；
*   1.  `service1`：创建 `service1` 的实例；
*   1.  `service1`：获取需要注入的属性与方法（在原始对象上进行获取）；
*   1.  `service1`：如果开启了支持循环依赖的配置，就将 `service1` 放到三级缓存中（是否支持循环依赖，是可以配置的）；
*   1.  `service1`：对 `service1` 进行依赖注入，需要 `service2`，然后就开始了 `service2` 的获取流程；
*   5.1 `service2`：获取 `service2`，从一级缓存中获取，此时是获取不到的；
*   5.2 `service2`：创建 `service2` 的实例；
*   5.3 `service2`：获取需要注入的属性与方法（在原始对象上进行获取）；
*   5.4 `service2`：如果开启了支持循环依赖的配置，就将 `service2` 放到三级缓存中（是否支持循环依赖，是可以配置的）；
*   5.5 `service2`：对 `service2` 进行依赖注入，需要 `service1`，然后就开始了 `service1` 的获取流程；
*   5.5.1 `service1`: 获取 `service1`，从一级缓存中获取，获取不到；此时发现 `service1` 正在创建中，于是继续从二、三级缓存中获取，最终从三级缓存中获取到了，将其放入二级缓存。从三级缓存获取的过程中，**会判断 `service1` 是否需要进行 aop，然后开始 aop 操作**，因此放入二级缓存中的是 `service1` 代理代理，提前进行 aop 是解决循环依赖的关键；
*   5.6 `service2`：得到了 `service1` 后（这里的 `service1` 是代理对象），将其注入到 `service2` 中，接着对 `service2` 进行 aop，得到 `service2` 的代理对象；
*   5.7 `service2`：如果支持循环依赖，先从一、二级缓存中再次获取 `service2`，都未获取到，就使用当前 `service2`（当前 `service2` 是代理对象)；
*   5.8 `service2`：将 service2 的代理对象放入一级缓存中，删除二、三级缓存，至此，`service2` 初始化完成，注入的 `service1` 是代理对象，一级缓存中的 `service2` 也是代理对象；
*   1.  `service1`：回到 `service1` 的生命周期，拿到 `service2`（这里的 `service2` 是代理对象）后，将其注入到 `service1`，`service1` 的依赖注入完成，进行初始化，这里会判断 `service1` 是否需要进行 aop，虽然 `service1` 是需要进行 aop 的，但由于在 `5.5.1` 已经进行过 aop 了，因此，这里直接返回（到这一步，`service1` 还是原始对象）；
*   1.  `service1`：如果支持循环依赖，先从一级缓存中获取 `service1`，获取不到；再从二缓存中获取 `service1`，可以获取到（从 `5.5.1` 可知，二级缓存里是 `service1` 代理对象），返回；
*   1.  `service1`：将二级缓存中获取的对象注册到一级缓存中，删除二、三级缓存，至此，`service1` 初始化完成，注入的 `service2` 是代理对象，一级缓存中的 `service1` 也是代理对象。

以上流程，虽然步骤较多，但 `service1` 与 `service2` 的获取步骤是相同的，只要弄清了其中之一的获取流程，另一个 bean 的获取流程就很雷同了。

在上述流程中，还有两个数据结构需要说明下：

*   `singletonsCurrentlyInCreation`：类型为 `SetFromMap<String>`，位于 `DefaultSingletonBeanRegistry`，创建方式为 `Collections.newSetFromMap(new ConcurrentHashMap<>(16))`，表明这是个由 `ConcurrentHashMap` 实现的 set，存储的是正在创建中的对象，**判断当前对象是否在创建中就是通过查找当前对象是否在这个 set 中**做到的；
*   `earlyProxyReferences`：类型为 `ConcurrentHashMap<Object, Object>`，位于 `AbstractAutoProxyCreator`，存储的是提前进行 aop 的对象，**如果一个对象提前进行了 aop，在后面再次 aop 时，会通过判断对象是否在 `earlyProxyReferences` 中而确定要不要进行 aop，以此来保证每个对象只进行一次 aop**。

至此，spring 一共提供了 5 个数据结构来辅助解决循环依赖问题，总结如下：

| 结构                            | 说明                                                         |
| ------------------------------- | ------------------------------------------------------------ |
| `singletonObjects`              | **一级缓存**，类型为 `ConcurrentHashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是完整的 `spring bean`，即完成属性注入、初始化的 bean，如果 bean 需要 aop，存储的就是代理对象 |
| `earlySingletonObjects`         | **二级缓存**，类型为 `HashMap<String, Object>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 是实例化完成，但未进行依赖注入的 `bean`，**如果 `bean` 需要 `aop`，这里存储的就是代理对象，只不过代理对象所持有的原始对象并未进行依赖注入** |
| `singletonFactories`            | **三级缓存**，类型为 `HashMap<String, ObjectFactory>`，位于 `DefaultSingletonBeanRegistry` 类中，`key` 为 `beanName`，`value` 存储的是一个 `lambda` 表达式：`() -> getEarlyBeanReference(beanName, mbd, bean)`，`getEarlyBeanReference(xxx)` 中的 `bean` 是刚创建完成的 `java bean`，没有进行 spring 依赖注入，也没进行 aop |
| `singletonsCurrentlyInCreation` | 类型为 `SetFromMap<String>`，位于 `DefaultSingletonBeanRegistry`，创建方式为 `Collections.newSetFromMap(new ConcurrentHashMap<>(16))`，表明这是个由 `ConcurrentHashMap` 实现的 set，存储的是正在创建中的对象，可以**用来判断当前对象是否在创建中** |
| `earlyProxyReferences`          | 类型为 `ConcurrentHashMap<Object, Object>`，位于 `AbstractAutoProxyCreator`，存储的是提前进行 aop 的对象，可以**用来判断 bean 是否进行过 aop，保证每个对象只进行一次 aop** |

以上就是 spring 解决循环依赖的完整流程了。

### 3\. 代码模拟

在正式分析源码前，我们首先模拟循环下依赖解决的过程，代码如下：

```
/**
 * 准备一个类，内部有一个属性 Obj2
 */
public class Obj1 {
    // 需要注入 obj2
    private Obj2 obj2;

    // 省略其他方法
    ...
}

/**
 * 准备一个类，内部有一个属性 Obj1
 */
public class Obj2 {
    // 需要注入 ob1
    private Obj1 obj1;

    // 省略其他方法
    ...

}

/**
 * 准备Obj2的代理类，内部持有obj2的对象
 */
public class ProxyObj2 extends Obj2 {
    // obj2代理类内部持有obj2的原始对象
    private Obj2 obj2;

    public ProxyObj2(Obj2 obj2) {
        this.obj2 = obj2;
    }

    // 省略其他方法
    ...

}

/**
 * 准备Obj1的代理类，内部持有obj1的对象
 */
public class ProxyObj1 extends Obj1 {
    // obj2代理类内部持有obj1的原始对象
    private Obj1 obj1;

    public ProxyObj1(Obj1 obj1) {
        this.obj1 = obj1;
    }

    // 省略其他方法
    ...

}

```

*   首先准备了两个类：`Obj1` 与 `Obj2`， 其中 `Obj1` 有个属性为 `Obj2`，`Obj2` 中有个属性为 `Obj1`；
*   接着准备了 `Obj1` 与 `Obj2` 的代理类 `ProxyObj1`、`ProxyObj2`，并且 `ProxyObj1`、`ProxyObj2` 分别有一个属性：`Obj1` 、 `Obj2`；
*   我们依旧以 `new ObjX()` 模拟对象的创建；
*   我们依旧以 `objX.setObjX(xxx)` 模拟依赖注入；
*   我们依旧以 `new ProxyObjX(xxx)` 模拟代理对象的生成；
*   我们依旧以 `collection.add(xxx)` 模拟对象添加到容器中的过程；

我们模拟最终得到的结果为：

*   最终放入容器的对象分别是 `proxyObj1`，`proxyObj2`
*   注入到 `obj1` 中的是 `proxyObj2`，注入到 `obj2` 中的是 `proxyObj2`

准备工作已经完成了，接下来我们就开始进行模拟了。

#### 3.1 模拟 1

要求：

*   Obj1 与 Obj2 必须严格按照 “创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中” 的流程创建
*   两个对象的创建流程可以交替进行

目标：

*   最终放入容器的对象分别是 `proxyObj1`，`proxyObj2`
*   注入到 `obj1` 中的是 `proxyObj2`，注入到 `obj2` 中的是 `proxyObj2`

代码如下：

```
public static main(String[] args) {
     // 准备一个容器，这里保存的是完成上述生命周期的对象
     // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
     // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
     Collection<?> collection = new ArrayList();

     // 1\. 创建 Obj1 对象
     Obj1 obj1 = new Obj1();

     // 接下来需要将obj2的代理对象注入到obj1中，但此时容器中并没有obj2的代理对象，于是切换到obj2的创建流程
     // 一. 创建 Obj2 对象
     Obj2 obj2 = new Obj2();

     // 到这里，obj2需要注入obj1的代理对象，但此时容器中并没有obj2的代理对象，于是又要切到obj1的创建流程

}

```

在执行以上流程中 ，发现创建 Obj2 对象后，流程就进行不下去了：

*   `obj1` 需要注入 `obj2` 的代理对象，但找不到，于是切换到 `obj2` 的创建流程；
*   `obj2` 需要注入 `obj1` 的代理对象，但找不到，于是切换到 `obj1` 的创建流程；
*   `obj1` 需要注入 `obj2` 的代理对象，但找不到，于是切换到 `obj2` 的创建流程；
*   ...

如此循环往复。

模拟结果：未达到预期目标，本次模拟宣告失败。

#### 3.1 模拟 2

要求：

*   Obj1 与 Obj2 必须以下两种流程之一创建：
    *   “创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中” 的流程创建
    *   “创建 (原始对象)--> 生成代理对象 --> 对原始对象进行属性注入 --> 将代理对象保存到容器中” 的流程创建
*   两个对象的创建流程可以交替进行

目标：

*   最终放入容器的对象分别是 `proxyObj1`，`proxyObj2`
*   注入到 `obj1` 中的是 `proxyObj2`，注入到 `obj2` 中的是 `proxyObj2`

示例代码如下：

```
 public static main(String[] args) {
      // 准备一个容器，这里保存的是完成上述生命周期的对象
      // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
      // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
      Collection<?> collection = new ArrayList();

      // 1\. 创建 Obj1 对象
      Obj1 obj1 = new Obj1();

      // 接下来需要将obj2的代理对象注入到obj1中，但此时容器中并没有obj2的代理对象，于是切换到obj2的创建流程
      // 一. 创建 Obj2 对象
      Obj2 obj2 = new Obj2();

      // 2\. 对 Obj1 提前代理
      ProxyObj1 proxyObj1 = new ProxyObj1(obj1);

      // 二. 将 proxyObj1 注入到 obj2 中
      obj2.setObj1(proxyObj1);

      // 三. 生成 obj2的代理对象
      ProxyObj2 proxyObj2 = new ProxyObj2(obj2);

      // 四. proxyObj2 已经走完了完整的生命周期，将代理对象添加到容器时
      collection.add(proxyObj2);

      // 此时容器中已经有 obj2 的代理对象了，继续obj1的生命周期
      // 3\. 将 proxyObj2 注入到 obj1 中
      obj1.setObj2(proxyObj2);

      // 4\. proxyObj1 已经走完了完整的生命周期，将代理对象添加到容器时
      collection.add(proxyObj1);
 }

```

上面的代码中，obj1 的流程用 “1，2，3，4” 标识，obj2 的流程用 “一，二，三，四” 标识，两者流程如下：

*   obj1：“创建 (原始对象)--> 生成代理对象 --> 对原始对象进行属性注入 --> 将代理对象保存到容器中”
*   obj2：“创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中”

最终两者都存入了容器中，达到了预期的目标。

#### 3.3 从模拟中得到的结论

对比上面两个模拟代码，发现模拟 2 之 所以能达到预期目标，主要是因为在注入 `obj2` 的 `obj1` 属性时，提前生成了 `obj1` 的代理对象 `proxyObj1`，使得 `obj2` 能完成整个创建流程。这里再次证明，提供进行 aop 对循环依赖的解决起到至关重要的作用！

限于篇幅，本文就先到这里了，本文主要分析了循环依赖的产生，介绍了 spring 解决循环依赖的步骤，最后通过两段代码模拟了循环依赖的解决，下一篇文章我们将从 spring 源码分析 spring 是如何解决循环依赖的。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4659555](https://my.oschina.net/funcy/blog/4659555) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_

示例代码如下：

```
 public static main(String[] args) {
      // 准备一个容器，这里保存的是完成上述生命周期的对象
      // 1\. 如果元素是原始对象，则该对象已经完成了属性注入 
      // 2\. 如果元素是代理对象，则该对象持有的原有对象已经完成了属性注入 
      Collection<?> collection = new ArrayList();

      // 1\. 创建 Obj1 对象
      Obj1 obj1 = new Obj1();

      // 接下来需要将obj2的代理对象注入到obj1中，但此时容器中并没有obj2的代理对象，于是切换到obj2的创建流程
      // 一. 创建 Obj2 对象
      Obj2 obj2 = new Obj2();

      // 2\. 对 Obj1 提前代理
      ProxyObj1 proxyObj1 = new ProxyObj1(obj1);

      // 二. 将 proxyObj1 注入到 obj2 中
      obj2.setObj1(proxyObj1);

      // 三. 生成 obj2的代理对象
      ProxyObj2 proxyObj2 = new ProxyObj2(obj2);

      // 四. proxyObj2 已经走完了完整的生命周期，将代理对象添加到容器时
      collection.add(proxyObj2);

      // 此时容器中已经有 obj2 的代理对象了，继续obj1的生命周期
      // 3\. 将 proxyObj2 注入到 obj1 中
      obj1.setObj2(proxyObj2);

      // 4\. proxyObj1 已经走完了完整的生命周期，将代理对象添加到容器时
      collection.add(proxyObj1);
 }

```

上面的代码中，obj1 的流程用 “1，2，3，4” 标识，obj2 的流程用 “一，二，三，四” 标识，两者流程如下：

*   obj1：“创建 (原始对象)--> 生成代理对象 --> 对原始对象进行属性注入 --> 将代理对象保存到容器中”
*   obj2：“创建 --> 属性注入 --> 生成代理对象 --> 保存到容器中”

最终两者都存入了容器中，达到了预期的目标。

#### 3.3 从模拟中得到的结论

对比上面两个模拟代码，发现模拟 2 之 所以能达到预期目标，主要是因为在注入 `obj2` 的 `obj1` 属性时，提前生成了 `obj1` 的代理对象 `proxyObj1`，使得 `obj2` 能完成整个创建流程。这里再次证明，提供进行 aop 对循环依赖的解决起到至关重要的作用！

限于篇幅，本文就先到这里了，本文主要分析了循环依赖的产生，介绍了 spring 解决循环依赖的步骤，最后通过两段代码模拟了循环依赖的解决，下一篇文章我们将从 spring 源码分析 spring 是如何解决循环依赖的。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4659555](https://my.oschina.net/funcy/blog/4659555) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_