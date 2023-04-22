本文是《事务的隔离级别与传播方式的处理》分析的第 2 篇，接上文，我们继续。

#### 3.5 获取事务信息

事务的信息会在 `TransactionAspectSupport#createTransactionIfNecessary` 方法中获取，这个方法非常重要，前面介绍隔离级别、传播方式都会在这个方法里处理。该方法代码如下：

```
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {
    // 如果未指定名称，则将方法名当做事务名称
    if (txAttr != null && txAttr.getName() == null) {
        txAttr = new DelegatingTransactionAttribute(txAttr) {
            @Override
            public String getName() {
                return joinpointIdentification;
            }
        };
    }

    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            // 获取事务状态，如果当前没有事务，可能会创建事务
            status = tm.getTransaction(txAttr);
        }
    }
    // 准备事务信息，就是将前面得到的信息封装成 TransactionInfo
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}

```

这个方法主要是两个操作：

1.  获取事务状态
2.  准备事务信息

先来看下获取事务状态的流程，方法为 `AbstractPlatformTransactionManager#getTransaction`：

```
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    TransactionDefinition def = (definition != null ? 
            definition : TransactionDefinition.withDefaults());

    // 获取事务对象
    Object transaction = doGetTransaction();
    boolean debugEnabled = logger.isDebugEnabled();

    // 是否存在事务，存在则返回
    if (isExistingTransaction(transaction)) {
        return handleExistingTransaction(def, transaction, debugEnabled);
    }
    // 运行到了这里，表明当前没有事务

    // 检查超时时间的设置是否合理
    if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
    }

    // PROPAGATION_MANDATORY：必须在事务中运行，这里没有事务，直接抛异常
    // No existing transaction found -> check propagation behavior to find out how to proceed.
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(...);
    }
    // 挂起当前事务，创建新事务
    else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        // suspend(...) 传入null：如果有同步事务，则挂起同步事务，否则什么也不做
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            // 创建事务对象
            DefaultTransactionStatus status = newTransactionStatus(
                    def, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // 启动事务
            doBegin(transaction, def);
            // 设置 TransactionSynchronizationManager 的属性
            prepareSynchronization(status, def);
            return status;
        }
        catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    else {
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}

```

这个方法有点长，我们慢慢分析。

##### 1. `doGetTransaction(...)`：获取事务对象

获取事务对象的方法为 `DataSourceTransactionManager#doGetTransaction`：

```
protected Object doGetTransaction() {
    DataSourceTransactionObject txObject = new DataSourceTransactionObject();
    txObject.setSavepointAllowed(isNestedTransactionAllowed());
    // 获取连接信息，obtainDataSource()：获取数据源
    ConnectionHolder conHolder = 
            (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
    txObject.setConnectionHolder(conHolder, false);
    return txObject;
}

```

这里有两个操作：

1.  获取数据源
2.  获取 `ConnectionHolder`

我们先来看看数据源是如何获取的：

```
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager
        implements ResourceTransactionManager, InitializingBean {

    @Nullable
    private DataSource dataSource;

    /**
     * 构造方法传入了数据源
     */
    public DataSourceTransactionManager(DataSource dataSource) {
        this();
        setDataSource(dataSource);
        afterPropertiesSet();
    }

    /**
     * 设置数据源
     */
    public void setDataSource(@Nullable DataSource dataSource) {
        if (dataSource instanceof TransactionAwareDataSourceProxy) {
            this.dataSource = ((TransactionAwareDataSourceProxy) dataSource)
                    .getTargetDataSource();
        }
        else {
            this.dataSource = dataSource;
        }
    }

    @Nullable
    public DataSource getDataSource() {
        return this.dataSource;
    }

    /**
     * 获取数据源
     */
    protected DataSource obtainDataSource() {
        DataSource dataSource = getDataSource();
        Assert.state(dataSource != null, "No DataSource set");
        return dataSource;
    }

    ...
}

```

`obtainDataSource()` 实际上是调用了 `getDataSource()` 方法，返回的是 `dataSource` 成员变量，而 `dataSource` 又是在 `DataSourceTransactionManager` 的构造方法里传入的，因此，得到的结论是，这里获取的数据源就是我们在设置 `DataSourceTransactionManager` 时传入的：

```
@Configuration
public class TxDemo03Config {

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
     * 事务管理器
     * @param dataSource
     * @return
     */
    @Bean
    public DataSourceTransactionManager transactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    ...

}

```

而这个数据源，正是 `SimpleDriverDataSource`.

我们再来看看 `ConnectionHolder` 的获取，该方法为 `TransactionSynchronizationManager#getResource` ，代码如下：

```
// 用 ThreadLocal 来存放  ConnectionHolder 信息
private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

/**
 * 获取 ConnectionHolder
 */
public static Object getResource(Object key) {
    // 包装下传入的 key
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    // 在这里获取连接信息
    Object value = doGetResource(actualKey);
    return value;
}

/**
 * 具体的获取操作
 */
private static Object doGetResource(Object actualKey) {
    // 从ThreadLocal中获取
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    Object value = map.get(actualKey);
    if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
        map.remove(actualKey);
        if (map.isEmpty()) {
            resources.remove();
        }
        value = null;
    }
    return value;
}

```

从代码来看，`TransactionSynchronizationManager` 持有一个 `ThreadLocal` 的实例，其中存放了一个 `Map`，该 `Map` 的 `key` 为 `datasource`，`value` 为 `ConnectionHolder`.

那么这个 `ConnectionHolder` 是什么呢？可以简单地将其理解为 `Connection`(数据库连接) 的包装类，其中最重要的属性就是 `Connection` 了：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-d8c4ae3884177f485fbb95d1828fdb39ae2.png)

好了，到这里就把 `doGetTransaction(xxx)` 方法分析完了，再来看看这个方法返回的结果：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-0944429e31a6c0b67e121674202dd5ec0fd.png)

##### 2. `isExistingTransaction(...)`：是否存在事务

获取到事务对象 `DataSourceTransactionObject` 后，接下来就是判断是否存在事务了，判断方法在 `DataSourceTransactionManager#isExistingTransaction`，代码如下：

```
protected boolean isExistingTransaction(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    return (txObject.hasConnectionHolder() 
            && txObject.getConnectionHolder().isTransactionActive());
}

```

`ConnectionHolder` 中有一个成员变量 `transactionActive`，它表明当前 `ConnectionHolder` 的事务是否处于激活状态，`isExistingTransaction(...)` 方法主要是根据它来判断当前事务对象是否存在事务的。

##### 3\. 处理已存在的事务：`handleExistingTransaction(...)`

这里我们来看看如果当前存在事务，spring 是怎么处理的，处理已存在事务的方法为 `AbstractPlatformTransactionManager#handleExistingTransaction`，代码如下：

```
private TransactionStatus handleExistingTransaction(TransactionDefinition definition, 
        Object transaction, boolean debugEnabled) throws TransactionException {
    // 当传播方式为【不使用事务】时，抛出异常
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
        throw new IllegalTransactionStateException(
                "Existing transaction found for transaction marked with propagation 'never'");
    }
    // 当传播方式为【不支持事务】时，挂起当前事务，然后在无事务的状态中运行
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
        // 1\. suspend()：挂起事务操作
        Object suspendedResources = suspend(transaction);
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(
                definition, null, false, newSynchronization, debugEnabled, suspendedResources);
    }

    // 当传播方式为【在新的事务中运行】时，挂起当前事务，然后启动新的事务
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
        // 挂起事务操作
        SuspendedResourcesHolder suspendedResources = suspend(transaction);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(definition, transaction, true,
                    newSynchronization, debugEnabled, suspendedResources);
            // 2\. doBegin()：启动新的事务
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
        catch (RuntimeException | Error beginEx) {
            resumeAfterBeginException(transaction, suspendedResources, beginEx);
            throw beginEx;
        }
    }

    // 当传播方式为【嵌套执行】时， 设置事务的保存点
    // 存在事务，将该事务标注保存点，形成嵌套事务。
    // 嵌套事务中的子事务出现异常不会影响到父事务保存点之前的操作。
    if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        if (!isNestedTransactionAllowed()) {
            throw new NestedTransactionNotSupportedException(...);
        }
        // 3\. createAndHoldSavepoint(...)：创建保存点，回滚时只回滚到该保存点
        if (useSavepointForNestedTransaction()) {
            DefaultTransactionStatus status = prepareTransactionStatus(definition, transaction,
                    false, false, debugEnabled, null);
            status.createAndHoldSavepoint();
            return status;
        }
        else {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            DefaultTransactionStatus status = newTransactionStatus(
                    definition, transaction, true, newSynchronization, debugEnabled, null);
            // 如果不支持保存点，就启动新的事务
            doBegin(transaction, definition);
            prepareSynchronization(status, definition);
            return status;
        }
    }
    if (isValidateExistingTransaction()) {
        // 处理验证操作，不作分析
        ...
    }
    boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
    return prepareTransactionStatus(definition, transaction, false, 
            newSynchronization, debugEnabled, null);
}

```

可以看到，这个方法里就处理了事务的隔离级别的逻辑，相关的代码已经作了注释，这里就不多说了，不过这里有几个方法需要特别提出：

1.  `suspend()`：挂起事务操作
2.  `doBegin()`：启动新的事务
3.  `createAndHoldSavepoint(...)`：创建保存点，回滚时只回滚到该保存点

这几个操作包含了事务处理操作，后面我们再统一分析。

##### 4\. 继续 `AbstractPlatformTransactionManager#getTransaction`

让我们再回到 `AbstractPlatformTransactionManager#getTransaction` 方法，继续剩下的流程：

```
public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition)
        throws TransactionException {

    // 前面已经分析过了，省略
    ...

    // 运行到了这里，表明当前没有事务

    // 检查超时时间的设置是否合理
    if (def.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
        throw new InvalidTimeoutException("Invalid transaction timeout", def.getTimeout());
    }

    // PROPAGATION_MANDATORY：必须在事务中运行，这里没有事务，直接抛异常
    if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
        throw new IllegalTransactionStateException(...);
    }
    // 挂起当前事务，创建新事务
    else if (def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
            def.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
        // suspend(...) 传入null：如果有同步事务，则挂起同步事务，否则什么也不做
        SuspendedResourcesHolder suspendedResources = suspend(null);
        try {
            boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
            // 创建事务对象
            DefaultTransactionStatus status = newTransactionStatus(
                    def, transaction, true, newSynchronization, debugEnabled, suspendedResources);
            // 启动事务
            doBegin(transaction, def);
            // 设置 TransactionSynchronizationManager 的属性
            prepareSynchronization(status, def);
            return status;
        }
        catch (RuntimeException | Error ex) {
            resume(null, suspendedResources);
            throw ex;
        }
    }
    else {
        boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
        return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);
    }
}

```

`handleExistingTransaction(...)` 方法的功能是**当事务存在时，那些个传播类型要怎么处理**，`getTransaction(...)` 方法余下部分的功能是，**当事务存在时，那些个传播类型又要怎么处理**。可以看到，这里面依然还是有 `suspend(...)`、`doBegin(...)` 等方法，这些个方法我们一会也统一分析。

##### 5\. 准备返回结果：`prepareTransactionStatus(...)`

`handleExistingTransaction(...)` 方法与 `getTransaction(...)` 方法在处理返回结果时，都使用了 `prepareTransactionStatus(...)` 方法：

```
// `handleExistingTransaction(...)`方法
return prepareTransactionStatus(definition, transaction, false, 
            newSynchronization, debugEnabled, null);

// `getTransaction(...)`方法
return prepareTransactionStatus(def, null, true, newSynchronization, debugEnabled, null);

```

我们来分析下这个方法是做了啥，进入 `AbstractPlatformTransactionManager#prepareTransactionStatus`：

```
protected final DefaultTransactionStatus prepareTransactionStatus(
        TransactionDefinition definition, @Nullable Object transaction, boolean newTransaction,
        boolean newSynchronization, boolean debug, @Nullable Object suspendedResources) {

    // 创建了一个 DefaultTransactionStatus 对象
    DefaultTransactionStatus status = newTransactionStatus(
            definition, transaction, newTransaction, newSynchronization, debug, suspendedResources);
    // 准备 Synchronization
    prepareSynchronization(status, definition);
    return status;
}

/**
 *创建一个 TransactionStatus 实例
 */
protected DefaultTransactionStatus newTransactionStatus(
        TransactionDefinition definition, @Nullable Object transaction, boolean newTransaction,
        boolean newSynchronization, boolean debug, @Nullable Object suspendedResources) {

    boolean actualNewSynchronization = newSynchronization &&
            !TransactionSynchronizationManager.isSynchronizationActive();
    // 调用 DefaultTransactionStatus 的构造方法
    return new DefaultTransactionStatus(
            transaction, newTransaction, actualNewSynchronization,
            definition.isReadOnly(), debug, suspendedResources);
}

```

所以这个方法主要就是为了创建 `DefaultTransactionStatus` 对象！我们来看下这一步的运行结果：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-a491ee7ca22238b2d9c698c55ae9dd6d005.png)

##### 6\. 准备事务信息：`TransactionAspectSupport#prepareTransactionInfo`

回到 `TransactionAspectSupport#createTransactionIfNecessary` 方法：

```
protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, final String joinpointIdentification) {
    ...
    TransactionStatus status = null;
    if (txAttr != null) {
        if (tm != null) {
            // 获取事务状态，如果当前没有事务，可能会创建事务
            status = tm.getTransaction(txAttr);
        }
    }
    // 准备事务信息，就是将前面得到的信息封装成 TransactionInfo
    return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
}

```

前面分析了那么多，只是得到了 `TransactionStatus`，我们再接再厉，继续分析准备事务信息的方法 `prepareTransactionInfo(...)`：

```
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
        @Nullable TransactionAttribute txAttr, String joinpointIdentification,
        @Nullable TransactionStatus status) {
    TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);

    if (txAttr != null) {
        txInfo.newTransactionStatus(status);
    }

    // 省略log的打印
    ...

    // 与线程绑定
    txInfo.bindToThread();
    return txInfo;
}

```

嗯，同 `prepareTransactionStatus(...)` 类似，这个方法也是创建了一个 `TransactionInfo` 对象，并且将 `TransactionInfo` 与当前线程绑定，绑定的代码如下：

```
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {

    // 存放当前使用的事物信息
    private static final ThreadLocal<TransactionInfo> transactionInfoHolder =
            new NamedThreadLocal<>("Current aspect-driven transaction");

    // 重置为旧的事务信息
    protected void cleanupTransactionInfo(@Nullable TransactionInfo txInfo) {
        if (txInfo != null) {
            txInfo.restoreThreadLocalStatus();
        }
    }

    /**
     * TransactionInfo: 保存事务信息
     */
    protected static final class TransactionInfo {

        // 当前的事务状态对象
        @Nullable
        private TransactionStatus transactionStatus;

        // 旧的事务信息（也就是挂起的事务信息）
        @Nullable
        private TransactionInfo oldTransactionInfo;

        /**
         * 将事务信息绑定到当前线程
         */
        private void bindToThread() {
            // 拿到旧的事务信息
            this.oldTransactionInfo = transactionInfoHolder.get();
            // 设置成最新的事务信息
            transactionInfoHolder.set(this);
        }

        /**
         * 事务完成后，会将旧的事务信息绑定到当前线程
         */
        private void restoreThreadLocalStatus() {
            // 设置为旧的事务信息
            transactionInfoHolder.set(this.oldTransactionInfo);
        }

        ...
    }

}

```

`TransactionAspectSupport` 中有一个 `ThreadLocal`，用来存放当前的 `TransactionInfo` 对象，进行线程绑定时，会先拿到旧的事务信息，保存在 `TransactionInfo` 的成员变量 `oldTransactionInfo` 中，然后将新的 `TransactionInfo` 放入 `ThreadLocal` 中；当事务执行完成后，会从 `TransactionInfo` 的成员变量 `oldTransactionInfo` 中拿到旧的事务信息，再将旧的事务信息放入 `ThreadLocal` 中，这就完成了事务 "旧 - 新 - 旧" 的切换.

这一步得到的 `TransactionInfo` 如下：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-ab882613ce67ba3bf7afcbd2eab01c6cf27.png)

对于 `TransactionInfo` 的结构，这里作一些说明：

*   类型是 `TransactionAspectSupport.TransactionInfo`，是 `TransactionAspectSupport` 的一个内部类，封装了事务的一些信息
*   `transactionManager`: 事务管理器，就是我们设置的 `DataSourceTransactionManager`
*   `transactionAttribute`: 事务属性值，用来保存 `@Transactional` 注解的属性值
*   `joinpointIdentification`: 方法的全限定名（格式为："包名。类型。方法名"）
*   `transactionStatus`: 从名称上看，是记录事务的状态，实际这个对象不仅记录了事务的状态，包括的重大功能如下：
    *   `complete`: 事务的完成状态
    *   `connectionHolder`: 当前持有的数据库连接
    *   `suspendedResources`: 挂起的数据库连接，当需要恢复挂起的事务时，可以能过该对象拿到挂起的数据库连接
*   `oldTransactionInfo`: 上一个事务（也就是挂起的事务）的信息，执行完当前事务后，会恢复到上一个事务的执行

一不小心又写了这么长了，本文就先分析到这里了，`suspend(...)`、`doBegin(...)` 等方法下篇再分析吧。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4947799](https://my.oschina.net/funcy/blog/4947799) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_