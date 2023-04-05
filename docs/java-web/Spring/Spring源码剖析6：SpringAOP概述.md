# 目录
  * [我们为什么要使用 AOP](#我们为什么要使用-aop)
  * [使用装饰器模式](#使用装饰器模式)
  * [使用代理模式](#使用代理模式)
  * [使用CGLIB](#使用cglib)
  * [使用AOP](#使用aop)
  * [AOP总结](#aop总结)


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
## 我们为什么要使用 AOP


一年半前写了一篇文章Spring3：AOP，是当时学习如何使用Spring AOP的时候写的，比较基础。这篇文章最后的推荐以及回复认为我写的对大家有帮助的评论有很多，但是现在从我个人的角度来看，这篇文章写得并不好，甚至可以说是没有太多实质性的内容，因此这些推荐和评论让我觉得受之有愧。

基于以上原因，更新一篇文章，从最基础的原始代码–>使用设计模式（装饰器模式与代理）–>使用AOP三个层次来讲解一下为什么我们要使用AOP，希望这篇文章可以对网友朋友们有益。

原始代码的写法
既然要通过代码来演示，那必须要有例子，这里我的例子为：


有一个接口Dao有insert、delete、update三个方法，在insert与update被调用的前后，打印调用前的毫秒数与调用后的毫秒数
首先定义一个Dao接口：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public interface Dao {

    public void insert();

    public void delete();

    public void update();

}
```

然后定义一个实现类DaoImpl：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class DaoImpl implements Dao {

    @Override
    public void insert() {
        System.out.println("DaoImpl.insert()");
    }

    @Override
    public void delete() {
        System.out.println("DaoImpl.delete()");
    }

    @Override
    public void update() {
        System.out.println("DaoImpl.update()");
    }

}
```

最原始的写法，我要在调用insert()与update()方法前后分别打印时间，就只能定义一个新的类包一层，在调用insert()方法与update()方法前后分别处理一下，新的类我命名为ServiceImpl，其实现为：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class ServiceImpl {

    private Dao dao = new DaoImpl();

    public void insert() {
        System.out.println("insert()方法开始时间：" + System.currentTimeMillis());
        dao.insert();
        System.out.println("insert()方法结束时间：" + System.currentTimeMillis());
    }

    public void delete() {
        dao.delete();
    }

    public void update() {
        System.out.println("update()方法开始时间：" + System.currentTimeMillis());
        dao.update();
        System.out.println("update()方法结束时间：" + System.currentTimeMillis());
    }

}
```

这是最原始的写法，这种写法的缺点也是一目了然：

方法调用前后输出时间的逻辑无法复用，如果有别的地方要增加这段逻辑就得再写一遍

如果Dao有其它实现类，那么必须新增一个类去包装该实现类，这将导致类数量不断膨胀

## 使用装饰器模式
接着我们使用上设计模式，先用装饰器模式，看看能解决多少问题。装饰器模式的核心就是实现Dao接口并持有Dao接口的引用，我将新增的类命名为LogDao，其实现为：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class LogDao implements Dao {

    private Dao dao;

    public LogDao(Dao dao) {
        this.dao = dao;
    }

    @Override
    public void insert() {
        System.out.println("insert()方法开始时间：" + System.currentTimeMillis());
        dao.insert();
        System.out.println("insert()方法结束时间：" + System.currentTimeMillis());
    }

    @Override
    public void delete() {
        dao.delete();
    }

    @Override
    public void update() {
        System.out.println("update()方法开始时间：" + System.currentTimeMillis());
        dao.update();
        System.out.println("update()方法结束时间：" + System.currentTimeMillis());
    }

}
```

在使用的时候，可以使用”Dao dao = new LogDao(new DaoImpl())”的方式，这种方式的优点为：

透明，对调用方来说，它只知道Dao，而不知道加上了日志功能
类不会无限膨胀，如果Dao的其它实现类需要输出日志，只需要向LogDao的构造函数中传入不同的Dao实现类即可
不过这种方式同样有明显的缺点，缺点为：

输出日志的逻辑还是无法复用
输出日志的逻辑与代码有耦合，如果我要对delete()方法前后同样输出时间，需要修改LogDao
但是，这种做法相比最原始的代码写法，已经有了很大的改进。

## 使用代理模式
接着我们使用代理模式尝试去实现最原始的功能，使用代理模式，那么我们就要定义一个InvocationHandler，我将它命名为LogInvocationHandler，其实现为：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class LogInvocationHandler implements InvocationHandler {

    private Object obj;

    public LogInvocationHandler(Object obj) {
        this.obj = obj;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        String methodName = method.getName();
        if ("insert".equals(methodName) || "update".equals(methodName)) {
            System.out.println(methodName + "()方法开始时间：" + System.currentTimeMillis());
            Object result = method.invoke(obj, args);
            System.out.println(methodName + "()方法结束时间：" + System.currentTimeMillis());

            return result;
        }

        return method.invoke(obj, args);
    }

}
```

其调用方式很简单，我写一个main函数：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public static void main(String[] args) {
    Dao dao = new DaoImpl();

    Dao proxyDao = (Dao)Proxy.newProxyInstance(LogInvocationHandler.class.getClassLoader(), new Class<?>[]{Dao.class}, new LogInvocationHandler(dao));

    proxyDao.insert();
    System.out.println("----------分割线----------");
    proxyDao.delete();
    System.out.println("----------分割线----------");
    proxyDao.update();
}
```

结果就不演示了，这种方式的优点为：

输出日志的逻辑被复用起来，如果要针对其他接口用上输出日志的逻辑，只要在newProxyInstance的时候的第二个参数增加Class<?>数组中的内容即可

这种方式的缺点为：

JDK提供的动态代理只能针对接口做代理，不能针对类做代理
代码依然有耦合，如果要对delete方法调用前后打印时间，得在LogInvocationHandler中增加delete方法的判断

## 使用CGLIB
接着看一下使用CGLIB的方式，使用CGLIB只需要实现MethodInterceptor接口即可：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class DaoProxy implements MethodInterceptor {

    @Override
    public Object intercept(Object object, Method method, Object[] objects, MethodProxy proxy) throws Throwable {
        String methodName = method.getName();

        if ("insert".equals(methodName) || "update".equals(methodName)) {
            System.out.println(methodName + "()方法开始时间：" + System.currentTimeMillis());
            proxy.invokeSuper(object, objects);
            System.out.println(methodName + "()方法结束时间：" + System.currentTimeMillis());

            return object;
        }

        proxy.invokeSuper(object, objects);
        return object;
    }

}
```

代码调用方式为：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public static void main(String[] args) {
    DaoProxy daoProxy = new DaoProxy();

    Enhancer enhancer = new Enhancer();
    enhancer.setSuperclass(DaoImpl.class);
    enhancer.setCallback(daoProxy);

    Dao dao = (DaoImpl)enhancer.create();
    dao.insert();
    System.out.println("----------分割线----------");
    dao.delete();
    System.out.println("----------分割线----------");
    dao.update();
}
```

使用CGLIB解决了JDK的Proxy无法针对类做代理的问题，但是这里要专门说明一个问题：使用装饰器模式可以说是对使用原生代码的一种改进，使用Java代理可以说是对于使用装饰器模式的一种改进，但是使用CGLIB并不是对于使用Java代理的一种改进。

前面的可以说改进是因为使用装饰器模式比使用原生代码更好，使用Java代理又比使用装饰器模式更好，但是Java代理与CGLIb的对比并不能说改进，因为使用CGLIB并不一定比使用Java代理更好，这两种各有优缺点，像Spring框架就同时支持Java Proxy与CGLIB两种方式。

从目前看来代码又更好了一些，但是我认为还有两个缺点：

无论使用Java代理还是使用CGLIB，编写这部分代码都稍显麻烦
代码之间的耦合还是没有解决，像要针对delete()方法加上这部分逻辑就必须修改代码

## 使用AOP

最后来看一下使用AOP的方式，首先定义一个时间处理类，我将它命名为TimeHandler：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class TimeHandler {

    public void printTime(ProceedingJoinPoint pjp) {
        Signature signature = pjp.getSignature();
        if (signature instanceof MethodSignature) {
            MethodSignature methodSignature = (MethodSignature)signature;
            Method method = methodSignature.getMethod();
            System.out.println(method.getName() + "()方法开始时间：" + System.currentTimeMillis());

            try {
                pjp.proceed();
                System.out.println(method.getName() + "()方法结束时间：" + System.currentTimeMillis());
            } catch (Throwable e) {

            }
        }
    }

}
```

到第8行的代码与第12行的代码分别打印方法开始执行时间与方法结束执行时间。我这里写得稍微复杂点，使用了的写法，其实也可以拆分为与两种，这个看个人喜好。

这里多说一句，切面方法printTime本身可以不用定义任何的参数，但是有些场景下需要获取调用方法的类、方法签名等信息，此时可以在printTime方法中定义JointPoint，Spring会自动将参数注入，可以通过JoinPoint获取调用方法的类、方法签名等信息。由于这里我用的，要保证方法的调用，这样才能在方法调用前后输出时间，因此不能直接使用JoinPoint，因为JoinPoint没法保证方法调用。此时可以使用ProceedingJoinPoint，ProceedingPointPoint的proceed()方法可以保证方法调用，但是要注意一点，ProceedingJoinPoint只能和搭配，换句话说，如果aop.xml中配置的是，然后printTime的方法参数又是ProceedingJoinPoint的话，Spring容器启动将报错。

接着看一下aop.xml的配置：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="http://www.springframework.org/schema/beans

http://www.springframework.org/schema/beans/spring-beans-3.0.xsd

http://www.springframework.org/schema/aop

http://www.springframework.org/schema/aop/spring-aop-3.0.xsd">

    <bean id="daoImpl" class="org.xrq.spring.action.aop.DaoImpl" />
    <bean id="timeHandler" class="org.xrq.spring.action.aop.TimeHandler" />


</beans>
```

我不大会写expression，也懒得去百度了，因此这里就拦截Dao下的所有方法了。测试代码很简单：


    /**
    * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
    */
    public class AopTest {
    
    ```
        @Test
        @SuppressWarnings("resource")
        public void testAop() {
            ApplicationContext ac = new ClassPathXmlApplicationContext("spring/aop.xml");
    
            Dao dao = (Dao)ac.getBean("daoImpl");
            dao.insert();
            System.out.println("----------分割线----------");
            dao.delete();
            System.out.println("----------分割线----------");
            dao.update();
        }
    
    }
    ```

## AOP总结

结果就不演示了。到此我总结一下使用AOP的几个优点：

切面的内容可以复用，比如TimeHandler的printTime方法，任何地方需要打印方法执行前的时间与方法执行后的时间，都可以使用TimeHandler的printTime方法
避免使用Proxy、CGLIB生成代理，这方面的工作全部框架去实现，开发者可以专注于切面内容本身
代码与代码之间没有耦合，如果拦截的方法有变化修改配置文件即可
下面用一张图来表示一下AOP的作用：

我们传统的编程方式是垂直化的编程，即A–>B–>C–>D这么下去，一个逻辑完毕之后执行另外一段逻辑。但是AOP提供了另外一种思路，它的作用是在业务逻辑不知情（即业务逻辑不需要做任何的改动）的情况下对业务代码的功能进行增强，这种编程思想的使用场景有很多，例如事务提交、方法执行之前的权限检测、日志打印、方法调用事件等等。

AOP使用场景举例
上面的例子纯粹为了演示使用，为了让大家更加理解AOP的作用，这里以实际场景作为例子。

第一个例子，我们知道MyBatis的事务默认是不会自动提交的，因此在编程的时候我们必须在增删改完毕之后调用SqlSession的commit()方法进行事务提交，这非常麻烦，下面利用AOP简单写一段代码帮助我们自动提交事务（这段代码我个人测试过可用）：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class TransactionHandler {

    public void commit(JoinPoint jp) {
        Object obj = jp.getTarget();
        if (obj instanceof MailDao) {
            Signature signature = jp.getSignature();
            if (signature instanceof MethodSignature) {
                SqlSession sqlSession = SqlSessionThrealLocalUtil.getSqlSession();               

                MethodSignature methodSignature = (MethodSignature)signature;
                Method method = methodSignature.getMethod();

                String methodName = method.getName();
                if (methodName.startsWith("insert") || methodName.startsWith("update") || methodName.startsWith("delete")) {
                    sqlSession.commit();
                }

                sqlSession.close();
            }
        }
    }

}
```

这种场景下我们要使用的aop标签为，即切在方法调用之后。

这里我做了一个SqlSessionThreadLocalUtil，每次打开会话的时候，都通过SqlSessionThreadLocalUtil把当前会话SqlSession放到ThreadLocal中，看到通过TransactionHandler，可以实现两个功能：

insert、update、delete操作事务自动提交
对SqlSession进行close()，这样就不需要在业务代码里面关闭会话了，因为有些时候我们写业务代码的时候会忘记关闭SqlSession，这样可能会造成内存句柄的膨胀，因此这部分切面也一并做了
整个过程，业务代码是不知道的，而TransactionHandler的内容可以充分再多处场景下进行复用。

第二个例子是权限控制的例子，不管是从安全角度考虑还是从业务角度考虑，我们在开发一个Web系统的时候不可能所有请求都对所有用户开放，因此这里就需要做一层权限控制了，大家看AOP作用的时候想必也肯定会看到AOP可以做权限控制，这里我就演示一下如何使用AOP做权限控制。我们知道原生的Spring MVC，Java类是实现Controller接口的，基于此，利用AOP做权限控制的大致代码如下（这段代码纯粹就是一段示例，我构建的Maven工程是一个普通的Java工程，因此没有验证过）：

```
/**
 * @author 五月的仓颉http://www.cnblogs.com/xrq730/p/7003082.html
 */
public class PermissionHandler {

    public void hasPermission(JoinPoint jp) throws Exception {
        Object obj = jp.getTarget();

        if (obj instanceof Controller) {
            Signature signature = jp.getSignature();
            MethodSignature methodSignature = (MethodSignature)signature;

            // 获取方法签名
            Method method = methodSignature.getMethod();
            // 获取方法参数
            Object[] args = jp.getArgs();

            // Controller中唯一一个方法的方法签名ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws Exception;
            // 这里对这个方法做一层判断
            if ("handleRequest".equals(method.getName()) && args.length == 2) {
                Object firstArg = args[0];
                if (obj instanceof HttpServletRequest) {
                    HttpServletRequest request = (HttpServletRequest)firstArg;
                    // 获取用户id
                    long userId = Long.parseLong(request.getParameter("userId"));
                    // 获取当前请求路径
                    String requestUri = request.getRequestURI();

                    if(!PermissionUtil.hasPermission(userId, requestUri)) {
                        throw new Exception("没有权限");
                    }
                }
            }
        }

    }

}
```

毫无疑问这种场景下我们要使用的aop标签为。这里我写得很简单，获取当前用户id与请求路径，根据这两者，判断该用户是否有权限访问该请求，大家明白意思即可。

后记
文章演示了从原生代码到使用AOP的过程，一点一点地介绍了每次演化的优缺点，最后以实际例子分析了AOP可以做什么事情。



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
 
