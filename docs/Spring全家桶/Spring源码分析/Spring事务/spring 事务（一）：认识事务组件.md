前面分析完了 spring aop 相关功能后，本文将来分析 spring aop 的一个应用 —— 事务管理。

### 1\. 从两个 demo 讲起

在正式分析前，我们先来思考下，如果让我们自己来基于 spring aop 来设计一套事务处理机制，该如何实现呢？如果没有 spring，我们的事务处理代码一般长这样：

```
public void fun() {
    // 开启事务
    start();
    try {
        // 业务处理
        xxx();
        // 提交事务
        commit();
    } catch(Exception e) {
        // 回滚事务
        rollback();
        throw e;
    }
}

```

从上面的代码来看，像开启事务、提交事务、回滚事务，都跟业务代码无关，这些可以使用 spring aop 来实现，因此就有了下面两个 demo.

#### demo01：基于 `@Around` 注解实现事务

咱们可以使用 `@Around` 注解来操作，代码如下：

1.  定义一个注解：`@MyTransactional`

```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface MyTransactional {
}

```

1.  定义 aop 操作

```
@Aspect
@Component
public class MyAopAspectj {
    @Pointcut("@annotation(org.springframework.learn.tx.demo02.MyTransactional)")
    public void testAop(){

    }

    @Around("testAop()")
    public Object around(ProceedingJoinPoint p) throws Throwable {
        System.out.println("执行前，开启事务....");
        try {
            Object o = p.proceed();
            System.out.println("执行完成，提交事务....");
            return o;
        } catch (Throwable e) {
            System.out.println("出现了异常，根据异常类型回滚事务....");
            throw e;
        } finally {
            System.out.println("执行后....");
        }
    }

}

```

1.  config，进行一些必要的配置

```
@Configuration
@ComponentScan("org.springframework.learn.tx.demo02")
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class TxDemo02Config {

}

```

1.  添加一个 service 类，其中一个方法上有 `@MyTransactional` 注解

```
@Service
public class TxTestService {

    @MyTransactional
    public void test01() {
        System.out.println("执行test01方法");
    }

    public void test02() {
        System.out.println("执行test02方法");
    }

}

```

1.  主类

```
public class TxDemo02Main {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext
                = new AnnotationConfigApplicationContext(TxDemo02Config.class);
        TxTestService service = applicationContext.getBean(TxTestService.class);
        System.out.println("-------------------");
        service.test01();
        System.out.println("-------------------");
        service.test02();

    }
}

```

运行，结果如下：

```
-------------------
执行前，开启事务....
执行test01方法
执行完成，提交事务....
执行后....
-------------------
执行test02方法

```

这个 demo 中，我们使用 `@Around` 注解来拦截业务代码执行前后的情况，可以看到，`@Around` 注解可以在代码运行前后甚至是出现异常时处理一些额外的操作。

#### demo02：自定义 `advisor` 实现事务

让我们回忆下 spring aop 对 `@Around` 注解的处理，实际上 `@Around` 最终会封装为 `InstantiationModelAwarePointcutAdvisorImpl` 对象，后面的处理就跟 `@Around` 无关了，`@Around` 到 `InstantiationModelAwarePointcutAdvisorImpl` 对象的过程，可参考 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（上）](https://my.oschina.net/funcy/blog/4678817).

`InstantiationModelAwarePointcutAdvisorImpl` 是个什么东西呢？这是个 `advisor`，具体来说就是可用于方法的增强。关于 spring aop 是如何找到能应用于当前方法的 `advisor` 的，可参考 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)。

通过以上分析，给我们提供了另一个思路：我们可以实现 `advisor` 接口，定制化自己的逻辑，代码如下：

1.  准备 `advice`

```
/**
 * 这个advice就是advisor的一个属性，切面逻辑在这里处理
 */
public class MyAdvice implements MethodInterceptor {

    @Override
    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("执行前，开启事务....");
        try {
            Object val = invocation.proceed();
            System.out.println("执行完成，提交事务....");
            return val;
        } catch (Throwable e) {
            System.out.println("出现了异常，根据异常类型回滚事务....");
            throw e;
        } finally {
            System.out.println("执行后....");
        }
    }
}

```

1.  准备 `pointcut`

```
/**
 * 切点
 * 判断哪些方法能用于该advisor
 */
public class MyPointcut extends StaticMethodMatcherPointcut {
    /**
     * 匹配方法，有 @MyTransactional 的类或方法就返回true
     */
    @Override
    public boolean matches(Method method, Class<?> targetClass) {
        return null != AnnotationUtils.getAnnotation(method, MyTransactional.class)
                || null != AnnotationUtils.getAnnotation(targetClass, MyTransactional.class);
    }
}

```

1.  准备 `advisor`

```
/**
 * advisor 可看作是 advice 与 pointcut 的包装
 */
@Component
public class MyAdvisor extends AbstractBeanFactoryPointcutAdvisor {

    private static final long serialVersionUID = 2651364800145442305L;

    private MyPointcut pointcut;

    public MyAdvisor() {
        this.pointcut = new MyPointcut();
        this.setAdvice(new MyAdvice());
    }

    @Override
    public Pointcut getPointcut() {
        return this.pointcut;
    }

}

```

以上是不同于注解的实现方式，接下来的代码就与注解一样了。

1.  准备一个注解：`@MyTransactional`

```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface MyTransactional {
}

```

1.  处理项目配置

```
@Configuration
@ComponentScan("org.springframework.learn.tx.demo01")
@EnableAspectJAutoProxy(proxyTargetClass = true)
public class TxDemo01Config {

}

```

1.  准备一个 service

```
@Service
public class TxTestService {

    @MyTransactional
    public void test01() {
        System.out.println("执行test01方法");
    }

    public void test02() {
        System.out.println("执行test02方法");
    }

}

```

1.  主类

```
public class TxDemo01Main {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext
                = new AnnotationConfigApplicationContext(TxDemo01Config.class);
        TxTestService service = applicationContext.getBean(TxTestService.class);
        System.out.println("-------------------");
        service.test01();
        System.out.println("-------------------");
        service.test02();

    }
}

```

运行，结果如下：

```
-------------------
执行前，开启事务....
执行test01方法
执行完成，提交事务....
执行后....
-------------------
执行test02方法

```

### 2\. 使用 spring 事务管理

有了前面的两个小 demo 作为开胃菜，对于 spring 事务处理想必你已有了一个清晰的认识，spring 在处理事务时，使用的就是第二种方式，即往自定义一个 `advisor` 添加到 spring 容器中。关于 spring 实现任务的具体细节，我们待会分析，这里我们再上一个 demo，体验下我们平时是怎么使用事务的。

为了进行数据库连接，我们需要引入数据库连接池，这里我们使用的是 mysql，需要在 `spring-learn.gradle` 中添加依赖：

```
optional("mysql:mysql-connector-java:5.1.48")

```

接着就是代码了。

1.  配置类

```
@Configuration
@ComponentScan("org.springframework.learn.tx.demo03")
@EnableTransactionManagement(proxyTargetClass = true)
public class TxDemo01Config {

    /**
     * 生成数据源
     * @return
     * @throws Exception
     */
    @Bean
    public DataSource dataSource() throws Exception {
        Driver driver = new com.mysql.jdbc.Driver();
        String url = "jdbc:mysql://localhost:3306/test";
        String username = "root";
        String password = "123";
        return new SimpleDriverDataSource(driver, url, username, password);
    }

    /**
     * 生成jdbcTemplate，后面就是用这个类来处理数据库的操作
     * @param dataSource
     * @return
     */
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    /**
     * 事务管理器
     * @param dataSource
     * @return
     */
    @Bean
    public DataSourceTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

}

```

1.  数据库操作类

```
@Service
public class UserService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 数据库插入操作，使用 @Transactional 开启事务
     * @return
     */
    @Transactional(rollbackFor = Exception.class)
    public int insert() {
        String sql = "insert into `user`(`login_name`, `nick`, `create_time`, `update_time`)"
                + "values (?, ?, ?, ?)";
        int result = jdbcTemplate.update(sql, "test", "test", new Date(), new Date());
        if(true) {
            //throw new RuntimeException("抛出个异常");
        }
        System.out.println(result);
        return result;
    }

}

```

1.  主类

```
public class TxDemo01Main {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext
                = new AnnotationConfigApplicationContext(TxDemo01Config.class);
        UserService userService = applicationContext.getBean(UserService.class);
        userService.insert();

    }
}

```

demo 中，`DataSource` 使用 spring 自带的 `SimpleDriverDataSource`，`orm` 框架也是 spring 提供的 `jdbcTemplate`，使用的 `user` 表 sql 如下：

```
CREATE TABLE `user` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT COMMENT '主键id',
  `login_name` varchar(32) NOT NULL DEFAULT '0' COMMENT '登录名',
  `nick` varchar(32) NOT NULL DEFAULT '0' COMMENT '昵称',
  `create_time` datetime DEFAULT NULL COMMENT '创建时间',
  `update_time` datetime DEFAULT NULL COMMENT '更新时间',
  PRIMARY KEY (`id`),
  KEY `create_time` (`create_time`)
) ENGINE=InnoDB AUTO_INCREMENT=1 DEFAULT CHARSET=utf8mb4 COMMENT='用户表';

```

执行结果如下：

第一次不抛出异常，数据库结果：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-30bbe23a8e0491d1f59378469ad04703e03.png)

第二次抛出异常，数据库结果：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-edf98369ccef9735d83813cef7af7ea1dcd.png)

第三次不抛出异常，数据库结果：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-fab4169dbec7661f27bba203c5e77232f65.png)

可以看到，第二次抛出异常时，数据正常回滚了。

我们再来分析下这个 demo，跟事务有关的代码有三处：

*   `@EnableTransactionManagement(proxyTargetClass = true)`：启用事务
*   `DataSourceTransactionManager`：事务管理器
*   `@Transactional`：指定开启事务的方法

类比于 aop 的 `@EnableAspectJAutoProxy`，`@EnableTransactionManagement` 是启动事务的入口，接着我们就从这个注解入手，分析 spring 事务的启用流程。

### 3. `@EnableTransactionManagement` 注解

```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {

    /**
     * 学完aop后，想必对这个属性已经很熟悉
     * true: 表示强制使用cglib代理
     * false：如果目标类实现了接口，则使用jdk动态代理，否则使用cglib代理
     * 仅在 mode 为 PROXY 下生效
     */
    boolean proxyTargetClass() default false;

    /**
     * advice模式，使用代理，还是使用 aspectJ
     */
    AdviceMode mode() default AdviceMode.PROXY;

    /**
     * 执行顺序，当一个代理对象有多个增强时，按什么样的顺序来执行
     */
    int order() default Ordered.LOWEST_PRECEDENCE;

}

```

这个注解本身没什么，就三个属性，注释已经很明确了，我们关键还是看这个注解引入的类：`TransactionManagementConfigurationSelector`：

```
public class TransactionManagementConfigurationSelector extends 
        AdviceModeImportSelector<EnableTransactionManagement> {
    @Override
    protected String[] selectImports(AdviceMode adviceMode) {
        switch (adviceMode) {
            case PROXY:
                // 基于代理的事务管理，会引入两个类
                return new String[] {AutoProxyRegistrar.class.getName(),
                        ProxyTransactionManagementConfiguration.class.getName()};
            case ASPECTJ:
                // 基于aspectJ的事务管理，会引入这个类，本文不分析
                return new String[] {determineTransactionAspectClass()};
            default:
                return null;
        }
    }
    // 省略其他
    ...

}

```

基于代理的事务管理，会引入两个类：`AutoProxyRegistrar`、`ProxyTransactionManagementConfiguration`，接下来我们就来分析这两个类。

#### 3.1 `AutoProxyRegistrar`

从名字上来看，`AutoProxyRegistrar` 是一个注册器，还记得前面 aop 的注册器 `AspectJAutoProxyRegistrar` 吗，两者一样的套路！

我们来看看里面究竟做了啥：

```
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

    private final Log logger = LogFactory.getLog(getClass());

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, 
            BeanDefinitionRegistry registry) {
        boolean candidateFound = false;
        Set<String> annTypes = importingClassMetadata.getAnnotationTypes();
        for (String annType : annTypes) {
            AnnotationAttributes candidate = AnnotationConfigUtils
                    .attributesFor(importingClassMetadata, annType);
            if (candidate == null) {
                continue;
            }
            Object mode = candidate.get("mode");
            Object proxyTargetClass = candidate.get("proxyTargetClass");
            // 满足if条件的，就是 @EnableTransactionManagement 注解
            if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
                    Boolean.class == proxyTargetClass.getClass()) {
                candidateFound = true;
                if (mode == AdviceMode.PROXY) {
                    // 注册操作，最终注册了 InfrastructureAdvisorAutoProxyCreator 类，接下来会继续分析
                    AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
                    if ((Boolean) proxyTargetClass) {
                        // 使用cglib代理
                        AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
                        return;
                    }
                }
            }
        }
        if (!candidateFound && logger.isInfoEnabled()) {
            String name = getClass().getSimpleName();
            logger.info(...);
        }
    }
}

```

这行代码关键的就只有 if 里的几行，先说下 if 条件：`mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&Boolean.class == proxyTargetClass.getClass()`，通过上面的对 `@EnableTransactionManagement`，这说的就是它了；对于 `mode == AdviceMode.PROXY`，继续调用 `AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry)`（关于这个方法的调用，我们接下来会继续分析）；然后处理 `proxyTargetClass`，这个属性的作用与 `@EnableAspectJAutoProxy` 中的 `proxyTargetClass` 一致，也是可以强制使用 cglib 代理。

接着我们来分析下 `AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry)` 的过程，跟进代码：

> AopConfigUtils

```
    @Nullable
    public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
        // 继续往下看
        return registerAutoProxyCreatorIfNecessary(registry, null);
    }

    @Nullable
    public static BeanDefinition registerAutoProxyCreatorIfNecessary(
            BeanDefinitionRegistry registry, @Nullable Object source) {
        // 传入 InfrastructureAdvisorAutoProxyCreator 类，继续调用
        return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, 
            registry, source);
    }

```

看到这里，是不是有种深深的熟悉感？aop 中的 `AspectJAnnotationAutoProxyCreator` 也 是这么注册的！进入 `AopConfigUtils#registerOrEscalateApcAsRequired` 方法：

```
// AopConfigUtils 可注册的类都在这里了
private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>(3);

static {
    APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
    APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
    APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
}

/**
 * 注册操作
 */
private static BeanDefinition registerOrEscalateApcAsRequired(
        Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
    Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
    //如果已存在这个bean
    if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
        BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
        //判断优先级，如果优先级较高则替换原先的bean
        if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
            int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
            int requiredPriority = findPriorityForClass(cls);
            // 已存在类 的优先级 小于正在注册的，则使用正在注册的，已存在的三个类的优先级为
            // 0: InfrastructureAdvisorAutoProxyCreator(处理事务)
            // 1: AspectJAwareAdvisorAutoProxyCreator(处理基于xml的aop)
            // 2: AnnotationAwareAspectJAutoProxyCreator(处理基于注解的aop)
            if (currentPriority < requiredPriority) {
                apcDefinition.setBeanClassName(cls.getName());
            }
        }
        return null;
    }
    //注册XxxAutoProxyCreator到容器中
    RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
    beanDefinition.setSource(source);
    beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
    beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
    registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
    return beanDefinition;
}

/**
 * 查找注册类的优先级
 */
private static int findPriorityForClass(@Nullable String className) {
    for (int i = 0; i < APC_PRIORITY_LIST.size(); i++) {
        Class<?> clazz = APC_PRIORITY_LIST.get(i);
        if (clazz.getName().equals(className)) {
            return i;
        }
    }
    throw new IllegalArgumentException(
            "Class name [" + className + "] is not a known auto-proxy creator class");
}

```

`AopConfigUtils` 可注册的类有有三个：

*   `InfrastructureAdvisorAutoProxyCreator`：处理事务
*   `AspectJAwareAdvisorAutoProxyCreator`：处理基于 xml 的 aop
*   `AnnotationAwareAspectJAutoProxyCreator`：处理基于注解的 aop

这三者的优先级为 `AnnotationAwareAspectJAutoProxyCreator` > `AspectJAwareAdvisorAutoProxyCreator` > `InfrastructureAdvisorAutoProxyCreator`，注入时，会判断注入类的优先级，优先级高的最终会被注入到 spring 容器中。这样就导致了一个问题：**如果项目中同时开启了 aop (`@EnableAspectJAutoProxy`) 与事务 (`@EnableTransactionManagement`)，那么最终注入到容器的将是 `AnnotationAwareAspectJAutoProxyCreator`，这也就是说，`AnnotationAwareAspectJAutoProxyCreator` 也能处理事务！** 这句话非常关键，它意味着事务的处理过程，实际上就包含在前面分析的 aop 的过程中了！

我们也来看看 `InfrastructureAdvisorAutoProxyCreator`：

```
// 继承了 AbstractAdvisorAutoProxyCreator，这个类非常关键
public class InfrastructureAdvisorAutoProxyCreator extends AbstractAdvisorAutoProxyCreator {

    @Nullable
    private ConfigurableListableBeanFactory beanFactory;

    @Override
    protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
        super.initBeanFactory(beanFactory);
        this.beanFactory = beanFactory;
    }

    @Override
    protected boolean isEligibleAdvisorBean(String beanName) {
        return (this.beanFactory != null && 
                this.beanFactory.containsBeanDefinition(beanName) 
                &&  this.beanFactory.getBeanDefinition(beanName).getRole() 
                                == BeanDefinition.ROLE_INFRASTRUCTURE);
    }

}

```

`InfrastructureAdvisorAutoProxyCreator` 其实并没有做什么与 aop 相关的事，但它继承了一个关键的类：`AbstractAdvisorAutoProxyCreator`，这个类可是大有来头：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-2881f63ac07afc5095c449ddb9a0df2bb55.png)

从继承关系来看，这个类继承了 `AbstractAutoProxyCreator`，而 `AbstractAutoProxyCreator` 正是我们在 - [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（上）](https://my.oschina.net/funcy/blog/4678817) 与 [spring aop 之 AnnotationAwareAspectJAutoProxyCreator 分析（下）](https://my.oschina.net/funcy/blog/4687961)中重点分析的、代理对象的产生所在！

我们再来看下 `AnnotationAwareAspectJAutoProxyCreator`、`AspectJAwareAdvisorAutoProxyCreator` 、`InfrastructureAdvisorAutoProxyCreator` 这三者的关系：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-acd72335503eeb686abe81d930b45f1f3f0.png)

可以看到，`AspectJAwareAdvisorAutoProxyCreator` 、`InfrastructureAdvisorAutoProxyCreator` 都继承了 `AbstractAdvisorAutoProxyCreator`，`AnnotationAwareAspectJAutoProxyCreator` 又继承了 `AspectJAwareAdvisorAutoProxyCreator`。

通过以上分析，`AutoProxyRegistrar` 最终向 spring 容器注册了 `InfrastructureAdvisorAutoProxyCreator`(`aop` 未启用的情况下)，如果启用了 `aop`，则会注册 `AspectJAwareAdvisorAutoProxyCreator`(基于 `xml` 的 `aop`) 或 `AnnotationAwareAspectJAutoProxyCreator`(基于 `annotation` 的 `aop`)。

#### 3.2 `ProxyTransactionManagementConfiguration`

接下来我们来看看 `ProxyTransactionManagementConfiguration` 类。名字上来看，这是个配置类：

```
@Configuration(proxyBeanMethods = false)
public class ProxyTransactionManagementConfiguration 
        extends AbstractTransactionManagementConfiguration {

    /**
     * 读取Spring的 @Transactional 注解，并将相应的事务属性公开给Spring的事务基础结构
     */
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionAttributeSource transactionAttributeSource() {
        return new AnnotationTransactionAttributeSource();
    }

    /**
     * TransactionInterceptor继承了Advice，这个类是个advice，用来处理事务的执行操作
     * @param transactionAttributeSource：来自于上面的 transactionAttributeSource() 方法
     */
    @Bean
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public TransactionInterceptor transactionInterceptor(
            TransactionAttributeSource transactionAttributeSource) {
        TransactionInterceptor interceptor = new TransactionInterceptor();
        // 设置事务属性处理对象，就是用来处理 @Transactional 注解 的读取
        interceptor.setTransactionAttributeSource(transactionAttributeSource);
        if (this.txManager != null) {
            interceptor.setTransactionManager(this.txManager);
        }
        return interceptor;
    }

    /**
     * 事务增强器.
     * @param transactionAttributeSource：来自于上面的 transactionAttributeSource() 方法
     * @param transactionInterceptor：来自于上面的 transactionInterceptor(...) 方法
     */
    @Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor(
            TransactionAttributeSource transactionAttributeSource,
            TransactionInterceptor transactionInterceptor) {
        BeanFactoryTransactionAttributeSourceAdvisor advisor 
                = new BeanFactoryTransactionAttributeSourceAdvisor();
        // 事务属性类，用来保存 @Transactional 的属性
        advisor.setTransactionAttributeSource(transactionAttributeSource);
        // 配置advice，在advice里处理事务
        advisor.setAdvice(transactionInterceptor);
        if (this.enableTx != null) {
            advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
        }
        return advisor;
    }

}

```

可以看到这个类引入了一 些 `bean`：

*   `transactionAttributeSource`：类型为 `AnnotationTransactionAttributeSource`，用来解析 `@Transactional` 注解；
*   `transactionInterceptor`：类型为 `TransactionInterceptor`，`Advice` 的子类，处理事务的逻辑在这个类里；
*   `transactionAdvisor`：类型为 `BeanFactoryTransactionAttributeSourceAdvisor`，这是个 `Advisor`，用来处理切面逻辑，内部集成了上面两个对象：`transactionAttributeSource` 与 `transactionInterceptor`；

`ProxyTransactionManagementConfiguration` 继承了 `AbstractTransactionManagementConfiguration`，而 `AbstractTransactionManagementConfiguration` 中也引入了一些 `bean`：

```
@Configuration
public abstract class AbstractTransactionManagementConfiguration implements ImportAware {

    @Nullable
    protected AnnotationAttributes enableTx;

    /**
     * 保存事务管理器
     */
    @Nullable
    protected TransactionManager txManager;

    /**
     * 来自于 ImportAware 接口的方法
     */
    @Override
    public void setImportMetadata(AnnotationMetadata importMetadata) {
        this.enableTx = AnnotationAttributes.fromMap(importMetadata
                .getAnnotationAttributes(EnableTransactionManagement.class.getName(), false));
        if (this.enableTx == null) {
            throw new IllegalArgumentException(
                    "@EnableTransactionManagement is not present on importing class " 
                    + importMetadata.getClassName());
        }
    }

    /**
     * 配置事务管理器.
     * 注入spring容器中所有的 TransactionManagementConfigurer 对象
     * TransactionManagementConfigurer就只有一个方法：
     *  TransactionManager annotationDrivenTransactionManager()
     * 这个方法用来返回一个事务管理器
     */
    @Autowired(required = false)
    void setConfigurers(Collection<TransactionManagementConfigurer> configurers) {
        if (CollectionUtils.isEmpty(configurers)) {
            return;
        }
        if (configurers.size() > 1) {
            throw new IllegalStateException("Only one TransactionManagementConfigurer may exist");
        }
        TransactionManagementConfigurer configurer = configurers.iterator().next();
        this.txManager = configurer.annotationDrivenTransactionManager();
    }

    /**
     * 处理事件监听，用来处理 @TransactionalEventListener 注解的方法.
     */
    @Bean(name = TransactionManagementConfigUtils.TRANSACTIONAL_EVENT_LISTENER_FACTORY_BEAN_NAME)
    @Role(BeanDefinition.ROLE_INFRASTRUCTURE)
    public static TransactionalEventListenerFactory transactionalEventListenerFactory() {
        return new TransactionalEventListenerFactory();
    }

}

```

*   `void setConfigurers(Collection<TransactionManagementConfigurer> configurers)`：注入 `TransactionManagementConfigurer` 对象，具体作用已在代码注释中说明；
*   `TransactionalEventListenerFactory`：类型为 `TransactionalEventListenerFactory`，用来处理事务事件（主要是 `@TransactionalEventListener` 注解的方法，关于这部分的内容，本文就不展开了）。

好了，有了这些对象后，spring 就可以进行事务处理了，这些我们留到下篇文章再分析。

### 4\. 总结

本文先是从两个 demo 入手，示范了如果是由我们自己开发一个基于 spring aop 的事务管理功能是如何进行的，接着又用一个 demo 示范了如何使用 spring 提供的事务管理功能，然后就具体分析了 spring 事务启用注解 `@EnableTransactionManagement` 的功能。

`@EnableTransactionManagement` 是 spring 中用来启用事务管理功能的，在 `AdviceMode` 为 `proxy` 模式下，该注解向 spring 中引入了两个类：`AutoProxyRegistrar`、`ProxyTransactionManagementConfiguration`，作用如下：

*   `AutoProxyRegistrar`：`aop` 未启用的情况下，会向 spring 容器中注册 `InfrastructureAdvisorAutoProxyCreator`；如果启用了 `aop`，则会注册 `AspectJAwareAdvisorAutoProxyCreator`(基于 `xml` 的 `aop`) 或 `AnnotationAwareAspectJAutoProxyCreator`(基于 `annotation` 的 `aop`)。这三个类都是 `AbstractAdvisorAutoProxyCreator` 的子类，用来生成代理对象。

*   `ProxyTransactionManagementConfiguration`：这是一个配置类，通过带有 `@Bean` 注解的方法向容器中引入了一系列的 bean，用来处理事务逻辑，对这些 bean，本文只需大概了解即可。

本文就先到这里了，下篇文章继续分析事务处理机制。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4773454](https://my.oschina.net/funcy/blog/4773454) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_