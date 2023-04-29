前面几篇文章分析 springboot 的启动流程，本文来总结下。

文章一开始，从 `SpringApplication.run(Demo01Application.class, args);` 代码入手，着重分析了两个方法：

*   `SpringApplication#SpringApplication(...)`
*   `SpringApplication#run(...)`

这两个方法涵盖了 springboot 启动的整个流程，这里我们逐一总结下 。

### `SpringApplication#SpringApplication(...)`

这个方法的流程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-e9a43f1c523c0f19d37e4741580ed32ca08.png)

其中，

*   `webApplicationType` 会在后面决定创建什么类型的 `applicationContext`；
*   `Initialzers` 来自于 `META-INF/spring.factories`，会在 springboot 启动时做一些初始化操作；
*   `Listteners` 同样来自于 `META-INF/spring.factories`，提供了多个方法，可以方便地监听 springboot 的执行过程。

### `SpringApplication#run(...)`

这部分的流程如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-07a6b491fbe69b8dcbd41e59a8543f06671.png)

其中，

*   `getRunListener()` 会获取所有的 `Listeners`，也就是在 `SpringApplication#SpringApplication(...)` 中 获取的 `Listeners`，`Listeners` 中提供了众多方法，可监听 springboot 的启动流程；
*   准备运行环境时，会根据 `webApplicationType` 的类型来创建、配置，得到相应类型的 `Environment` 对象，这 个对象后面会设置到 spring 容器中，spring 容器中使用的 `Environment` 就是在这里创建及配置的；
*   创建 ioc 容器时，也是根据 `webApplicationType` 的类型来创建对应的 `ApplicationContext`；
*   在准备 ioc 容器的方法中，会对 `ApplicationContext` 做一个配置 ，`Initializers` 也会在这里运行；
*   启动 ioc 容器时，springboot 会注册一个 shutdownhook，用以在项目关闭时处理关闭操作，另外，对于 ioc 的启动流程，springboot 在其扩展中会创建及启动 web 容器；
*   springboot 提供了两种类型的运行器：`ApplicationRunner`、`CommandLineRunner`，在启动后会调用两者的方法。

以上内容讲的比较简略，如果要详细了解，可自行阅读前面的文章。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4906588](https://my.oschina.net/funcy/blog/4906588) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_