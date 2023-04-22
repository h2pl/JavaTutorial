到了这里，spring 容器的启动流程终于是分析完成了，这里使用一张图来总结整个启动流程：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-72a4008f2ad3401de6b4f2d5c7f697923a3.png)

本系列从 `AnnotationConfigApplicationContext#AnnotationConfigApplicationContext(String...)` 方法出发，分析了 `AnnotationConfigApplicationContext` 的无参构造方法、包扫描流程、容器的启动流程等，这部分内容比较多，重点流程如下：

1.  包扫描流程
2.  运行 `beanFactoryProcessor`
3.  初始化单例 bean

在分析过程中，我们忽略了许多细节，仅关注了主要流程，对细节感兴趣的小伙伴可以根据提供的链接进入相应文章阅读。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4659519](https://my.oschina.net/funcy/blog/4659519) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_