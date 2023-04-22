

<header>

# Spring Boot调度

以下内容仅是站长或网友个人学习笔记、总结和研究收藏。不保证正确性，因使用而带来的风险与本站无关！

</header>



<script>( adsbygoogle = window.adsbygoogle || []).push({});</script>



调度是执行特定时间段的任务的过程。Spring Boot为在Spring应用程序上编写调度程序提供了很好的支持。

## Java Cron表达式

Java Cron表达式用于配置CronTrigger的实例，它是`org.quartz.Trigger`的子类。 有关Java cron表达式的更多信息，请参阅此链接 -

*   [https://docs.oracle.com/cd/E12058_01/doc/doc.1014/e12030/cron_expressions.html](https://docs.oracle.com/cd/E12058_01/doc/doc.1014/e12030/cron_expressions.html)

`[@EnableScheduling](https://github.com/EnableScheduling "@EnableScheduling")`注解用于为应用程序启用调度程序。将此批注添加到主Spring Boot应用程序类文件中。

```
@SpringBootApplication
@EnableScheduling

public class DemoApplication {
   public static void main(String[] args) {
      SpringApplication.run(DemoApplication.class, args);
   }
}

```

`[@Scheduled](https://github.com/Scheduled "@Scheduled")`注解用于在特定时间段内触发调度程序。

```
@Scheduled(cron = "0 * 9 * * ?")
public void cronJobSch() throws Exception {
}

```

以下是一个示例代码，演示如何在每天上午9:00开始到每天上午9:59结束执行任务。

```
package com.yiibai.demo.scheduler;

import java.text.SimpleDateFormat;
import java.util.Date;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class Scheduler {
   @Scheduled(cron = "0 * 9 * * ?")
   public void cronJobSch() {
      SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
      Date now = new Date();
      String strDate = sdf.format(now);
      System.out.println("Java cron job expression:: " + strDate);
   }
}

```

以下屏幕截图显示了应用程序如何在`09:03:23`启动，并且从那时起每隔一分钟执行一次cron作业调度程序任务。

![](/uploads/images/2018/10/05/103218_77311.jpg)

## 固定速率

固定速率调度程序用于在特定时间执行任务。它不等待前一个任务的完成。 值是以毫秒为单位。 示例代码显示在此处 -

```
@Scheduled(fixedRate = 1000)
public void fixedRateSch() { 
}

```

此处显示了应用程序启动时每秒执行任务的示例代码 -

```
package com.yiibai.demo.scheduler;

import java.text.SimpleDateFormat;
import java.util.Date;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class Scheduler {
   @Scheduled(fixedRate = 1000)
   public void fixedRateSch() {
      SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");

      Date now = new Date();
      String strDate = sdf.format(now);
      System.out.println("Fixed Rate scheduler:: " + strDate);
   }
}

```

请注意以下屏幕截图，其中显示了在`09:12:00`启动的应用程序，之后每隔一个固定速率调度程序执行任务。

![](/uploads/images/2018/10/05/103355_72877.jpg)

## 固定延迟

固定延迟调度程序用于在特定时间执行任务。 它应该等待上一个任务完成。 值应以毫秒为单位。 此处显示示例代码 -

```
@Scheduled(fixedDelay = 1000, initialDelay = 1000)
public void fixedDelaySch() {
}

```

这里，`initialDelay`是在初始延迟值之后第一次执行任务的时间。

从应用程序启动完成`3`秒后每秒执行一次任务的示例如下所示 -

```
package com.yiibai.demo.scheduler;

import java.text.SimpleDateFormat;
import java.util.Date;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@Component
public class Scheduler {
   @Scheduled(fixedDelay = 1000, initialDelay = 3000)
   public void fixedDelaySch() {
      SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS");
      Date now = new Date();
      String strDate = sdf.format(now);
      System.out.println("Fixed Delay scheduler:: " + strDate);
   }
}

```

执行上面代码后，它会显示在`09:18:39`开始的应用程序，每`3`秒后，固定延迟计划程序任务(每秒执行一次)。





//更多请阅读：https://www.yiibai.com/spring-boot/spring_boot_scheduling.html

@EnableAsync 注解
要使用 @Async，首先需要使用 @EnableAsync 注解开启 Spring Boot 中的异步特性。

@Configuration
@EnableAsync
public class AppConfig {
}
更详细的配置说明，可以参考：AsyncConfigurer(opens new window)

#@Async 注解
#支持的用法
（1）无入参无返回值方法

您可以用 @Async 注解修饰方法，这表明这个方法是异步方式调用。换句话说，程序在调用此方法时会立即返回，而方法的实际执行发生在已提交给 Spring TaskExecutor 的任务中。在最简单的情况下，您可以将注解应用于返回 void 的方法，如以下示例所示：

@Async
void doSomething() {
// this will be executed asynchronously
}
（2）有入参无返回值方法

与使用 @Scheduled 注释注释的方法不同，这些方法可以指定参数，因为它们在运行时由调用者以“正常”方式调用，而不是由容器管理的调度任务调用。例如，以下代码是 @Async 注解的合法应用：

@Async
void doSomething(String s) {
// this will be executed asynchronously
}
（3）有入参有返回值方法

甚至可以异步调用返回值的方法。但是，这些方法需要具有 Future 类型的返回值。这仍然提供了异步执行的好处，以便调用者可以在调用 Future 上的 get() 之前执行其他任务。以下示例显示如何在返回值的方法上使用@Async：

@Async
Future<String> returnSomething(int i) {
// this will be executed asynchronously
}
#不支持的用法
@Async 不能与生命周期回调一起使用，例如 @PostConstruct。

要异步初始化 Spring bean，必须使用单独的初始化 Spring bean，然后在目标上调用 @Async 带注释的方法，如以下示例所示：

public class SampleBeanImpl implements SampleBean {

    @Async
    void doSomething() {
        // ...
    }

}

public class SampleBeanInitializer {

    private final SampleBean bean;

    public SampleBeanInitializer(SampleBean bean) {
        this.bean = bean;
    }

    @PostConstruct
    public void initialize() {
        bean.doSomething();
    }

}
#明确指定执行器
默认情况下，在方法上指定 @Async 时，使用的执行器是在启用异步支持时配置的执行器，即如果使用 XML 或 AsyncConfigurer 实现（如果有），则为 annotation-driven 元素。但是，如果需要指示在执行给定方法时应使用默认值以外的执行器，则可以使用 @Async 注解的 value 属性。以下示例显示了如何执行此操作：

@Async("otherExecutor")
void doSomething(String s) {
// this will be executed asynchronously by "otherExecutor"
}
在这种情况下，“otherExecutor”可以是 Spring 容器中任何 Executor bean 的名称，也可以是与任何 Executor 关联的限定符的名称（例如，使用 <qualifier> 元素或 Spring 的 @Qualifier 注释指定） ）。

#管理 @Async 的异常
当 @Async 方法的返回值类型为 Future 型时，很容易管理在方法执行期间抛出的异常，因为在调用 get 结果时会抛出此异常。但是，对于返回值类型为 void 型的方法，异常不会被捕获且无法传输。您可以提供 AsyncUncaughtExceptionHandler 来处理此类异常。以下示例显示了如何执行此操作：

public class MyAsyncUncaughtExceptionHandler implements AsyncUncaughtExceptionHandler {

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        // handle exception
    }
}
默认情况下，仅记录异常。您可以使用 AsyncConfigurer 或 <task：annotation-driven /> XML 元素定义自定义 AsyncUncaughtExceptionHandler。