前面几篇文章我们分析了 spring aop 的相关代码，这里来做个总结。

### 1\. spring 启用 aop 功能

在 [spring aop（一）：示例 demo 及 @EnableAspectJAutoProxy](https://my.oschina.net/funcy/blog/4678093 "spring aop（一）：示例 demo 及 @EnableAspectJAutoProxy") 一文中，我们分析了 spring 通过 `@EnableAspectJAutoProxy` 注解启用 aop 功能，而这个注解实际上是向 spring 中导入了 `AnnotationAwareAspectJAutoProxyCreator`，而这个类是一个 `BeanPostProcessor`，最终在 bean 初始化前后完成代理对象的生成。

### 2\. 代理对象的生成

spring aop 基于注解方式的实现是通过 `AnnotationAwareAspectJAutoProxyCreator` 类来操作的，这个类是一个 `BeanPostProcessor`，在 bean 的初始化前后执行的操作如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-c634a0bda86d94cce68aaa46ac74f57d41d.png)

### 3\. 切面方法的执行

当调用代理对象的方法，jdk 会根据代理的类型而选择执行 `InvocationHandler#invoke`(jdk 动态代理) 还是 `MethodInterceptor#intercept`(cglib 代理)，这一步在创建代理对象时，已经与代理对象结合了，开发者无法干涉调用哪个方法，不过这两个方法里的内容开发者可以自由发挥。在这两个方法中，spring 会获取可用用当前方法的所有 Advisors，然后执行 Advisors 里的切面方法，整个过程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-96bb9ba4b77e60a85a1da1c2cec3858edf7.png) ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-013fb0c06e03fbe5044c211497df8ce306a.png)

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4701587](https://my.oschina.net/funcy/blog/4701587) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_

