### 1\. servlet 3.0 规范

本系列一开始，先介绍了 `servlet3.0` 规范，通过该规范，可以让我们实现 web 项目的 `0xml` 配置.

*   `servlet3.0` 规范中，servlet 通过 `SPI` 机制提供了一个接口：`ServletContainerInitializer`；
*   spring 实现了该接口，在其实现类 `SpringServletContainerInitializer` 的 `onStartup(...)` 方法中，会执行所有实现了 `WebApplicationInitializer` 接口的类的 `onStartup(...)` 方法，最终我们只需要实现 `WebApplicationInitializer` 接口即可；
*   在我们自主实现 `WebApplicationInitializer` 接口的类中，在 `onStartup(...)` 向 servlet 容器中手动注册了一 个 servlet：`DispatcherServlet`，在这个 servlet 中会启动 spring 容器；

以上整个流程就这样：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0874fa7ef39ca9c405cdf55d99ca891ebf2.png)

### 2\. 启用 webMvc 的方式

我们分析了启用 `webMvc` 的两种方式：

#### 1. `@EnableWebMvc`

这种方式很简单，只需要这样：

```
// 使用@EnableWebMvc注解启用mvc功能
@Component
@EnableWebMvc
public class MvcConfig {
    ...
}

```

如果我们要处理 webMvc 的一些配置时，需要实现 `WebMvcConfigurer`:

```
// 实现 WebMvcConfigurer，添加自定义配置
@Component
public class MyWebMvcConfigurer implements WebMvcConfigurer {

    // 重写WebMvcConfigurer方法，处理自定义配置
}

```

#### 2\. 实现 `WebMvcConfigurationSupport`

还有一种方式启用 `webMvc` 的 方式是实现 `WebMvcConfigurationSupport`：

```
@Component
public class MyWebMvcConfigurationSupport extends WebMvcConfigurationSupport {
    // 重写配置方法，处理自定义配置
    ...

   /**
    * 比如，添加跨域配置，直接重写 addCorsMappings 方法
    */
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        // 添加自己的配置
        ...
    }

}

```

需要注意的是，使用这种方式后，需要处理自定义配置时，就不能再去实现 `WebMvcConfigurer` 接口了，而应该直接重写 `WebMvcConfigurationSupport` 中的相应方法，如上面重写 `addCorsMappings()`.

### 3\. 整个启动流程

用一张图来总结整个启动流程：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-74d675bbae28247726b8d054e8758c3d8b1.png) ![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-aa6bb35d0ab26925c45c62ab4d709d05cdd.png)

### 4\. 请求流程

也用一张图来总结整个请求流程：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-024b75e7f7952dbf1ace7aa5a8cfe3bcb77.png)

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4773418](https://my.oschina.net/funcy/blog/4773418) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_