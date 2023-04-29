在上一篇文章的最后，我们提到事务的执行在 `TransactionAspectSupport#invokeWithinTransaction` 方法中，本文将从这个方法出发，探究 spring 的事务机制。

注：本文所探讨的数据库是 `mysql`，其他数据库可能会有所差异。

### 1. 事务相关概念

在正式分析事务源码前，一些前置知识还是要了解一下的，这里主要介绍与 spring 相关的事务概念。

#### 1.1 事务的隔离级别

事务有四大特性 `ACID`，列举如下：

- 原子性（`Atomicity`）
- 一致性（`Consistency`）
- 隔离性（`Isolation`）
- 持久性（`Durability`）

而事务的隔离级别正是对隔离性（`Isolation`）的进一步划分，这些隔离级别如下：

- `读未提交`
- `读提交`
- `可重复读`
- `串行化`

关于这些概念本文并不会深入解读，我们重点还是关注 spring 相关的内容，spring 对事务隔离级别的定义在 `org.springframework.transaction.annotation.Isolation` 中，代码如下：

```
public enum Isolation {

    /**
    * 默认值，不设置隔离级别，最终使用的是数据库设置的隔离级别
    */
    DEFAULT(TransactionDefinition.ISOLATION_DEFAULT),

    /**
    * 读未提交
    */
    READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),

    /**
    * 读提交
    */
    READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),

    /**
    * 可重复读
    */
    REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),

    /**
    * 串行化
    */
    SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);

    ...
}
```

我们可以使用 `@Transactional` 注解的 `isolation` 来设置隔离级别

#### 1.2 事务的超时时间

- 可以给事务指定一个执行时间，如果执行消耗的时间超过了指定时间，事务就会抛出异常从而回滚
- 可以在 `@Transactional` 注解的 `timeout` 来设置超时时间

#### 1.3 只读事务

- 可以将事务设置为`只读模式`，这个我平时用基本没用到，查到一些文章说，将事务设置为`只读模式`后，其他事务中的更新只读事务看不到，且只读事务中不能有写操作，真实性有待验证
- 我们可以使用 `@Transactional` 注解的 `readOnly` 来设置只读模式

#### 1.4 事务的传播类型

我们试想一种情况：`方法A` 与`方式B` 都开启了事务，在`方法B` 中调用`方法A`，如果在`方法A` 执行完成后，`方法B` 报错了，代码示意如下：

```
class A {
    // 开启了事务
    @Transactional
    public void methdA() {
        // 处理一些操作
        ...
    }
}

class B {
    // 开启了事务
    @Transactional
    public void methodB() {
        // 1. 处理一些操作
        ...
        // 2. 调用 methodA()
        a.methodA();
        // 3. 这里报个错
        throw new RuntimeException();
    }
}
```

由于开启了事务，`方法B` 是一定会回滚的，那么`方法A` 要不要回滚呢？

- 如果我们把`方法A` 与`方法B` 的事务看成是同一个事务，`方法A` 应该也是要回滚的
- 如果我们把`方法A` 与`方法B` 看成是在两个独立的事务中执行，`方法A` 的执行与`方法B` 的报错无关，`方法A` 就不应该回滚

为了处理这种纠纷，spring 引入了`事务的传播类型`的概念，我们可以使用 `@Transactional` 注解的 `propagation` 来设置只读模式：

```
public @interface Transactional {
    ...

    // 默认的级别为 Propagation.REQUIRED
    Propagation propagation() default Propagation.REQUIRED;

}
```

spring 一共定义了 7 种事务的传播类型，列举如下：

| 事务传播行为                | 描述                                                         |
| --------------------------- | ------------------------------------------------------------ |
| `PROPAGATION_REQUIRED`      | 【默认值：必需】当前方法必须在事务中运行，如果当前线程中没有事务，则开启一个新的事务；如果当前线程中已经存在事务，则方法将会在该事务中运行。 |
| `PROPAGATION_MANDATORY`     | 【强制】当前方法必须在事务中运行，如果当前线程中不存在事务，则**抛出异常** |
| `PROPAGATION_SUPPORTS`      | 【支持】当前方法单独运行时不需要事务，但如果当前线程中存在事务时，方法会在事务中运行 |
| `PROPAGATION_REQUIRES_NEW`  | 【新事务】当前方法必须在独立的事务中运行，如果当前线程中已经存在事务，则将该事务挂起，重新开启一个事务，直到方法运行结束再恢复之前的事务 |
| `PROPAGATION_NESTED`        | 【嵌套】当前方法必须在事务中运行，如果当前线程中存在事务，则将该事务标注**保存点**，形成嵌套事务。嵌套事务中的子事务出现异常不会影响到父事务保存点之前的操作。 |
| `PROPAGATION_NOT_SUPPORTED` | 【不支持】当前方法不会在事务中运行，如果当前线程中存在事务，则将事务挂起，直到方法运行结束 |
| `PROPAGATION_NEVER`         | 【不允许】当前方法不允许在事务中运行，如果当前线程中存在事务，则**抛出异常** |

注意体会以下类型的区别：

1. `PROPAGATION_REQUIRED` 与 `PROPAGATION_MANDATORY`
    - `PROPAGATION_REQUIRED`：必须要在事务中运行，没有事务就**开启新事务**
    - `PROPAGATION_MANDATORY`：必须要在事务中运行，没有事务就**抛异常**
2. `PROPAGATION_NOT_SUPPORTED` 与 `PROPAGATION_NEVER`
    - `PROPAGATION_NOT_SUPPORTED`：不能在事务中运行，有事务就**挂起事务**
    - `PROPAGATION_NEVER`：不能在事务中运行，有事务就**抛异常**
3. `PROPAGATION_REQUIRES_NEW` 与 `PROPAGATION_NESTED`
    - `PROPAGATION_REQUIRES_NEW`：新事务执行完成后，旧事务报错，只回滚旧事务，新事务不回滚；新事务执行报错，新旧事务一起回滚
    - `PROPAGATION_NESTED`：子事务执行完成后，父事务报错，回滚到保存点；子事务执行报错，也是回滚到保存点

### 2. demo 准备

明确了以上概念后，接下来就开始分析了，我们先准备个简单的 demo：

先是准备一些配置：

```
@Configuration
@ComponentScan("org.springframework.learn.tx.demo03")
@EnableTransactionManagement(proxyTargetClass = true)
public class TxDemo03Config {

    /**
     * 生成数据源
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
     */
    @Bean
    public JdbcTemplate jdbcTemplate(DataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }

    /**
     * 事务管理器
     */
    @Bean
    public DataSourceTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

}
```

对以上代码说明如下：

- 数据源：使用的是 spring 提供的 `SimpleDriverDataSource`，这个数据源功能不多，适合这个简单的 demo
- 处理 jdbc 相关操作：也是使用 spring 提供的 `jdbcTemplate`，作为一个简单的 demo，不想引入 `mybatis`，`jpa` 等
- 事务管理器：使用的也是 spring 提供的 `DataSourceTransactionManager` ，对单数据源来说，这个完全够用了

准备一个 mysql 的操作，要开启事务：

```
@Service
public class UserService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    /**
     * 数据库插入操作，使用 @Transactional 开启事务
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

最后是主类：

```
public class TxDemo03Main {

    public static void main(String[] args) {
        AnnotationConfigApplicationContext applicationContext
                = new AnnotationConfigApplicationContext(TxDemo03Config.class);
        UserService userService = applicationContext.getBean(UserService.class);
        userService.insert();

    }
}
```

这个 demo 十分简单，就不多做分析了，我们接下来会通过这个 demo 进行一些调度操作，探究下 spring 对事务的隔离级别、传播方式的处理。

### 3. `TransactionAspectSupport#invokeWithinTransaction`

本文一开篇我们就说过，事务的处理在 `TransactionAspectSupport#invokeWithinTransaction` 方法，接下来我们将重点分析这个方法。

上代码：

```
protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass,
        final InvocationCallback invocation) throws Throwable {
    TransactionAttributeSource tas = getTransactionAttributeSource();

    // 1. 获取 @Transactional 的属性配置
    final TransactionAttribute txAttr = (tas != null 
            ? tas.getTransactionAttribute(method, targetClass) : null);

    // 2. 获取事务管理器（IOC容器中获取）
    final TransactionManager tm = determineTransactionManager(txAttr);

    // 这部分的代码是针对 TransactionManager 是 ReactiveTransactionManager 的情况，不作分析
    ...

    // 3. 将 TransactionManager 转换为 PlatformTransactionManager
    PlatformTransactionManager ptm = asPlatformTransactionManager(tm);
    // 4. 获取方法的全限定名，格式为："包名.类型.方法名"
    final String joinpointIdentification 
            = methodIdentification(method, targetClass, txAttr);
    if (txAttr == null || !(ptm instanceof CallbackPreferringPlatformTransactionManager)) {
        // 5. 获取事务信息，会在这里开启事务
        TransactionInfo txInfo = createTransactionIfNecessary(
                ptm, txAttr, joinpointIdentification);
        Object retVal;
        try {
            // 6. 执行具体的业务
            retVal = invocation.proceedWithInvocation();
        }
        catch (Throwable ex) {
            // 7. 异常回滚
            completeTransactionAfterThrowing(txInfo, ex);
            throw ex;
        }
        finally {
            // 8. 重置事务信息，就是将事务信息设置为旧的
            cleanupTransactionInfo(txInfo);
        }

        if (vavrPresent && VavrDelegate.isVavrTry(retVal)) {
            TransactionStatus status = txInfo.getTransactionStatus();
            if (status != null && txAttr != null) {
                retVal = VavrDelegate.evaluateTryFailure(retVal, txAttr, status);
            }
        }
        // 9. 提交事务
        commitTransactionAfterReturning(txInfo);
        return retVal;
    }
    else {
        // 处理 CallbackPreferringPlatformTransactionManager 类型的 TransactionManager，不作分析
        ...
    }
}
```

对 `TransactionAspectSupport#invokeWithinTransaction` 方法的内容，注释中已详细说明，接下来为我们来分析这些执行过程。

#### 3.1 获取 `@Transactional` 的属性配置

这个就是获取 `UserService#insert` 方法上标记的 `@Transactional` 的属性配置，得到的结果如下：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-aefbe56da2db53982f73092a191e587fcc8.png)

#### 3.2 获取事务管理器

获取事务管理器的方法为 `TransactionAspectSupport#determineTransactionManager`，直接看代码：

```
protected TransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
    if (txAttr == null || this.beanFactory == null) {
        return getTransactionManager();
    }

    // 如果在 @Transaction 注解上指定了事务管理器，就从spring容器中获取这个事务管理器
    String qualifier = txAttr.getQualifier();
    if (StringUtils.hasText(qualifier)) {
        return determineQualifiedTransactionManager(this.beanFactory, qualifier);
    }
    // 指定事务管理器名称，最终也是从spring容器中获取
    else if (StringUtils.hasText(this.transactionManagerBeanName)) {
        return determineQualifiedTransactionManager(
            this.beanFactory, this.transactionManagerBeanName);
    }
    else {
        // 指定事务管理器对象，直接返回
        TransactionManager defaultTransactionManager = getTransactionManager();
        if (defaultTransactionManager == null) {
            // 从缓存中获取默认的
            defaultTransactionManager = this.transactionManagerCache
                    .get(DEFAULT_TRANSACTION_MANAGER_KEY);
            if (defaultTransactionManager == null) {
                // 根据类型从 spring 容器中获取一个事务管理器
                defaultTransactionManager = this.beanFactory.getBean(TransactionManager.class);
                this.transactionManagerCache.putIfAbsent(
                        DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
            }
        }
        return defaultTransactionManager;
    }
}
```

代码虽然有点长，但逻辑非常清晰，这里对该方法的流程总结如下：

1. 如果在 `@Transaction` 注解上指定了事务管理器，就从 spring 容器中获取该事务管理器
2. 如果指定事务管理器名称，就从 spring 容器中获取该事务管理器
3. 如果指定事务管理器对象，直接返回
4. 以上都不满足，直接从 spring 容器中获取类型为 `TransactionManager` 的 bean

在 `TxDemo03Config` 中，我们配置事务管理器为 `DataSourceTransactionManager`：

```
public DataSourceTransactionManager transactionManager(DataSource dataSource) {
    return new DataSourceTransactionManager(dataSource);
}
```

这里得到的也是 `DataSourceTransactionManager`：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-34ac74501b184c44f95432b31b21a041619.png)

#### 3.3 将 `TransactionManager` 转换为 `PlatformTransactionManager`

这个没啥好说的，`DataSourceTransactionManager` 就是 `PlatformTransactionManager` 的子类，代码里做了一个类型转换。

```
private PlatformTransactionManager asPlatformTransactionManager(
        @Nullable Object transactionManager) {
    if (transactionManager == null || transactionManager instanceof PlatformTransactionManager) {
        return (PlatformTransactionManager) transactionManager;
    } else {
        // 抛个异常
        ...
    }
}
```

#### 3.4 获取方法的全限定名

这一步会拿到方法的全限定名，格式为："包名。类型。方法名"，这也没啥好说的，这一步得到的结果如下：

![img](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-656a73b9eb1cd3120d3586fe4f1302de373.png)

限于篇幅，本文就先分析到这里了，下篇我们继续。

------

*本文原文链接：https://my.oschina.net/funcy/blog/4773459 ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。*