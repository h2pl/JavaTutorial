本文是《事务的隔离级别与传播方式的处理》分析的第 3 篇，接上文，我们继续。

上文中我们提到了事务的开启（`doBegin(...)`）、挂起（`suspend(...)`）与创建保存点（`createAndHoldSavepoint(...)`）的操作，本文将来分析这些操作的实现。

### 1. `doBegin(...)`：启动新的事务

启动新事务的方法为 `DataSourceTransactionManager#doBegin`，代码如下：

```
protected void doBegin(Object transaction, TransactionDefinition definition) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    Connection con = null;

    try {
        // 1\. 获取数据库连接
        if (!txObject.hasConnectionHolder() ||
                txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
            // getConnection(): 获取数据库连接，obtainDataSource()：获取数据源
            Connection newCon = obtainDataSource().getConnection();
            txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
        }
        // 这里将 synchronizedWithTransaction 设置为true
        txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
        con = txObject.getConnectionHolder().getConnection();

        // 2\. 设置事务的隔离级别

        Integer previousIsolationLevel 
                = DataSourceUtils.prepareConnectionForTransaction(con, definition);
        txObject.setPreviousIsolationLevel(previousIsolationLevel);
        // 设置只读属性
        txObject.setReadOnly(definition.isReadOnly());

        // 3\. 开启事务
        if (con.getAutoCommit()) {
            txObject.setMustRestoreAutoCommit(true);
            // 关闭事务的自动提交，也就是开启事务
            con.setAutoCommit(false);
        }
        // 4\. 如果是只读事务，在这里设置
        prepareTransactionalConnection(con, definition);
        // 设置事务的激活标记
        txObject.getConnectionHolder().setTransactionActive(true);
        // 5\. 设置事务的超时时间
        int timeout = determineTimeout(definition);
        if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
            txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
        }

        // 6\. 绑定数据源与连接到当前线程
        if (txObject.isNewConnectionHolder()) {
            TransactionSynchronizationManager.bindResource(
                    obtainDataSource(), txObject.getConnectionHolder());
            }
        }
    } catch (Throwable ex) {
        // 处理异常操作，如果是新连接则关闭该连接
        if (txObject.isNewConnectionHolder()) {
            DataSourceUtils.releaseConnection(con, obtainDataSource());
            txObject.setConnectionHolder(null, false);
        }
        throw new CannotCreateTransactionException(...);
    }
}

```

以上代码还是挺清晰的，注释也很明确了，这里对关键步骤作进一步分析。

#### 1.1 获取数据库连接

数据库连接的获取还是很简单的，代码如下：

```
Connection newCon = obtainDataSource().getConnection();

```

其实就是调用 `javax.sql.DataSource#getConnection()` 方法。

#### 1.2 设置事务的隔离级别

在 `@Transactional` 中，我们可以使用 `isolation` 指定事务的隔离级别：

```
public @interface Transactional {
    /**
     * 指定事务的隔离级别
     */
    Isolation isolation() default Isolation.DEFAULT;
    ...
}

```

如果不指定，就使用默认的隔离级别，也就是使用数据库配置的。

spring 设置事务隔离级别的方法为 `DataSourceUtils#prepareConnectionForTransaction`，代码如下：

```
public static Integer prepareConnectionForTransaction(Connection con, 
        @Nullable TransactionDefinition definition) throws SQLException {
    Assert.notNull(con, "No Connection specified");
    if (definition != null && definition.isReadOnly()) {
        try {
            // 设置为只读模式
            con.setReadOnly(true);
        }
        catch (SQLException | RuntimeException ex) {
            ...
        }
    }

    Integer previousIsolationLevel = null;
    if (definition != null && definition.getIsolationLevel() 
            != TransactionDefinition.ISOLATION_DEFAULT) {
        int currentIsolation = con.getTransactionIsolation();
        if (currentIsolation != definition.getIsolationLevel()) {
            // 拿到之前的隔离级别，事务完成事，需要重置为原来的隔离级别
            previousIsolationLevel = currentIsolation;
            // 在这里设置数据库的隔离级别，调用的是：
            // java.sql.Connection.setTransactionIsolation
            con.setTransactionIsolation(definition.getIsolationLevel());
        }
    }
    return previousIsolationLevel;
}

```

这 个方法处理了两个设置：

1.  设置为只读模式：调用的是 `java.sql.Connection#setReadOnly` 方法
2.  设置隔离级别：调用的是 `java.sql.Connection.setTransactionIsolation` 方法

关于只读模式，也是可以在 `@Transactional` 中设置的：

```
public @interface Transactional {
    /**
     * 设置只读事务
     */
    boolean readOnly() default false;
    ...
}

```

#### 1.3 开启事务

激动人心的时刻终于来了，前面铺垫了那么多，就是为了这一步的操作：开启事务。开启事务的代码如下：

```
if (con.getAutoCommit()) {
    txObject.setMustRestoreAutoCommit(true);
    // 关闭事务的自动提交，也就是开启事务
    con.setAutoCommit(false);
}

```

具体流程为，先判断自动提交是否开启，如果开设置了，就将其设置为 false，调用的也是 `java.sql` 的方法：

*   获取自动提交状态：`java.sql.Connection#getAutoCommit`
*   设置自动提交状态：`java.sql.Connection#setAutoCommit`

#### 1.4 如果是只读事务，在这里设置

在前面分析 `1.2 设置事务的隔离级别`中，通过调用 `java.sql.Connection#setReadOnly` 将连接设置为只读了，这里还会再一次设置，方法为 `DataSourceTransactionManager#prepareTransactionalConnection`：

```
protected void prepareTransactionalConnection(Connection con, TransactionDefinition definition)
        throws SQLException {
    if (isEnforceReadOnly() && definition.isReadOnly()) {
        try (Statement stmt = con.createStatement()) {
            // 设置只读类型要运行sql
            stmt.executeUpdate("SET TRANSACTION READ ONLY");
        }
    }
}

```

这一次是通过执行 sql 语句 `SET TRANSACTION READ ONLY` 将事务设置为只读。

#### 1.5 设置事务的超时时间

在 `@Transactional` 注解中，我们可以使用 `timeout` 来指定事务的超时时间：

```
public @interface Transactional {
    /**
     * 设置超时时间
     */
    int timeout() default TransactionDefinition.TIMEOUT_DEFAULT;
    ...
}

```

这样设置的超时时间会在这里用到，我们进入 `ResourceHolderSupport#setTimeoutInSeconds`：

```
public abstract class ResourceHolderSupport implements ResourceHolder {
    /**
     * 截止时间
     */
    private Date deadline;

    /**
     * 设置时间，单位：秒
     */
    public void setTimeoutInSeconds(int seconds) {
        setTimeoutInMillis(seconds * 1000L);
    }

    /**
     * 设置时间，单位：毫秒
     * 最后会转化为 截止时间
     */
    public void setTimeoutInMillis(long millis) {
        this.deadline = new Date(System.currentTimeMillis() + millis);
    }

    /**
     * 获取截止时间
     */
    @Nullable
    public Date getDeadline() {
        return this.deadline;
    }

    /**
     * 获取剩余时间，单位：秒
     */
    public int getTimeToLiveInSeconds() {
        double diff = ((double) getTimeToLiveInMillis()) / 1000;
        int secs = (int) Math.ceil(diff);
        checkTransactionTimeout(secs <= 0);
        return secs;
    }

    /**
     * 获取剩余时间，单位：毫秒
     */
    public long getTimeToLiveInMillis() throws TransactionTimedOutException{
        if (this.deadline == null) {
            throw new IllegalStateException("No timeout specified for this resource holder");
        }
        long timeToLive = this.deadline.getTime() - System.currentTimeMillis();
        checkTransactionTimeout(timeToLive <= 0);
        return timeToLive;
    }

    ...
}

```

在 `ResourceHolderSupport` 中维护了一个成员变量 `deadline`（截止时间），传入的超时时间最终都会转化为 `deadline`。

获取剩余时间时，也是由 `deadline` 计算得到，返回的剩余时间有秒与毫秒两种。

看到这里，`txObject.getConnectionHolder().setTimeoutInSeconds(timeout)` 只是将超时时间设置到 `ConnectionHolder` 的成员变量中（`ConnectionHolder` 是 `ResourceHolderSupport` 的子类），似乎跟数据库没啥关系，数据库是怎么控制事务超时的呢？

不得不说，这个超时的控制是有点难找，本文是通过调试找到的，超时时间的设置在在 `DataSourceUtils#applyTimeout` 方法中，这其中可谓是经过了千山万水：

![](https://java-tutorial.oss-cn-shanghai.aliyuncs.com/up-9fc1a75dc7b0644269b6dba16dfd5d0e676.png)

感谢调试功能，没有它，天知道要多久才能找到这个时间的设置！我这里使用的是 `jdbcTemplate`，在其他 `orm` 框架下，设置超时时间应该会有所不同 。

我们看看 `DataSourceUtils#applyTimeout` 是怎么设置超时时间的：

```
public static void applyTimeout(Statement stmt, @Nullable DataSource dataSource, int timeout) 
        throws SQLException {
    Assert.notNull(stmt, "No Statement specified");
    ConnectionHolder holder = null;
    if (dataSource != null) {
        holder = (ConnectionHolder) TransactionSynchronizationManager.getResource(dataSource);
    }
    if (holder != null && holder.hasTimeout()) {
        // 这里就是获取剩余的超时时间，由 ConnectionHolder.dateline 计算得到的
        stmt.setQueryTimeout(holder.getTimeToLiveInSeconds());
    }
    else if (timeout >= 0) {
        // jdbcTemplate 自身也可以设置查询超时时间
        stmt.setQueryTimeout(timeout);
    }
}

```

最终调用的是 `java.sql.Statement#setQueryTimeout` 来设置超时时间的。

#### 1.6 绑定数据源与连接到当前线程

以上事情处理完成后，接下来就是绑定数据源与连接了，处理方法为 `TransactionSynchronizationManager#bindResource`:

```
/**
 * resources 存放当前线程中的数据源与连接
 * 其中存放的内容为一个 Map，Map 的 key 为数据源，value 为数据源对应的连接
 */
private static final ThreadLocal<Map<Object, Object>> resources =
        new NamedThreadLocal<>("Transactional resources");

/**
 * 绑定操作
 */
public static void bindResource(Object key, Object value) throws IllegalStateException {
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    Assert.notNull(value, "Value must not be null");
    Map<Object, Object> map = resources.get();
    if (map == null) {
        map = new HashMap<>();
        resources.set(map);
    }
    // 将数据源与连接存放到map中
    Object oldValue = map.put(actualKey, value);
    if (oldValue instanceof ResourceHolder && ((ResourceHolder) oldValue).isVoid()) {
        oldValue = null;
    }
    if (oldValue != null) {
        throw new IllegalStateException("Already value [" + oldValue + "] for key [" +
                actualKey + "] bound to thread [" + Thread.currentThread().getName() + "]");
    }

```

这 一步的操作还是比较简单的，就是将数据源与连接放进 `resources` 中，从而完成与当前线程的绑定操作。

### 2. `suspend(...)`：挂起事务

挂起事务的操作为 `AbstractPlatformTransactionManager#suspend`，代码如下：

```
protected final SuspendedResourcesHolder suspend(@Nullable Object transaction) 
        throws TransactionException {
    // 如果有同步的事务，则优先挂起同步的事务
    if (TransactionSynchronizationManager.isSynchronizationActive()) {
        List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
        try {
            Object suspendedResources = null;
            if (transaction != null) {
                // 挂起操作
                suspendedResources = doSuspend(transaction);
            }
            // 重置事务名称
            String name = TransactionSynchronizationManager.getCurrentTransactionName();
            TransactionSynchronizationManager.setCurrentTransactionName(null);
            // 重置只读状态
            boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
            TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
            // 重置隔离级别
            Integer isolationLevel = TransactionSynchronizationManager
                    .getCurrentTransactionIsolationLevel();
            TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
            // 重置事务激活状态
            boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
            TransactionSynchronizationManager.setActualTransactionActive(false);
            // 返回挂起的事务
            return new SuspendedResourcesHolder(
                    suspendedResources, suspendedSynchronizations, name, readOnly, 
                    isolationLevel, wasActive);
        }
        catch (RuntimeException | Error ex) {
            doResumeSynchronization(suspendedSynchronizations);
            throw ex;
        }
    }
    else if (transaction != null) {
        Object suspendedResources = doSuspend(transaction);
        return new SuspendedResourcesHolder(suspendedResources);
    }
    else {
        return null;
    }
}

```

`suspend(...)` 方法中最重要的就是挂起事务的操作了，也就是 `doSuspend(transaction)`，该方法 位于 `` 中，直接看代码：

```
protected Object doSuspend(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
    txObject.setConnectionHolder(null);
    // 解除绑定
    return TransactionSynchronizationManager.unbindResource(obtainDataSource());
}

```

继续进入 `TransactionSynchronizationManager.unbindResource` 方法：

```
/**
 * 解除绑定操作
 */
public static Object unbindResource(Object key) throws IllegalStateException {
    Object actualKey = TransactionSynchronizationUtils.unwrapResourceIfNecessary(key);
    // 继续调用
    Object value = doUnbindResource(actualKey);
    if (value == null) {
        throw new IllegalStateException(...);
    }
    return value;
}

/**
 * 解除绑定操作
 */
private static Object doUnbindResource(Object actualKey) {
    Map<Object, Object> map = resources.get();
    if (map == null) {
        return null;
    }
    // 移除资源
    Object value = map.remove(actualKey);
    if (map.isEmpty()) {
        resources.remove();
    }
    if (value instanceof ResourceHolder && ((ResourceHolder) value).isVoid()) {
        value = null;
    }
    return value;
}

```

在开启事务时，是将数据源与连接绑定到当前线程，挂起时 ，是将数据源与连接解除与当前线程的绑定关系。

### 3. `createAndHoldSavepoint(...)`：创建保存点

保存点的创建在 `AbstractTransactionStatus#createAndHoldSavepoint` 方法中处理，代码如下：

```
    // 保存点
    private Object savepoint;

    // 创建保存点
    public void createAndHoldSavepoint() throws TransactionException {
        setSavepoint(getSavepointManager().createSavepoint());
    }

    protected void setSavepoint(@Nullable Object savepoint) {
        this.savepoint = savepoint;
    }

```

嗯，看来这个方法里只是做了`保存点`的保存（也就是赋值给 `AbstractTransactionStatus` 的成员变量），要真正了解保存点的创建，还得看 `getSavepointManager().createSavepoint()`，进入到 `JdbcTransactionObjectSupport#createSavepoint`：

```
public Object createSavepoint() throws TransactionException {
    ConnectionHolder conHolder = getConnectionHolderForSavepoint();
    try {
        if (!conHolder.supportsSavepoints()) {
            throw new NestedTransactionNotSupportedException(...);
        }
        if (conHolder.isRollbackOnly()) {
            throw new CannotCreateTransactionException(...);
        }
        // 创建保存点
        return conHolder.createSavepoint();
    }
    catch (SQLException ex) {
        throw new CannotCreateTransactionException("Could not create JDBC savepoint", ex);
    }
}

```

发现最后调用的是 `ConnectionHolder#createSavepoint` 方法，原来保存点是在 `ConnectionHolder` 中创建的啊！继续：

```
// 保存点名称前缀
public static final String SAVEPOINT_NAME_PREFIX = "SAVEPOINT_";

// 保存点数量
private int savepointCounter = 0;

public Savepoint createSavepoint() throws SQLException {
    this.savepointCounter++;
    // 这里创建保存点，调用的是 java.sql.Connection#setSavepoint(java.lang.String) 方法
    return getConnection().setSavepoint(SAVEPOINT_NAME_PREFIX + this.savepointCounter);
}

```

保存点名称前缀为 `SAVEPOINT_`，每创建一个保存点，`savepointCounter` 的计数器就加 1，最终保存点的名称为 `SAVEPOINT_1`、`SAVEPOINT_2`、...

创建保存点最终调用的方法是 `java.sql.Connection#setSavepoint(java.lang.String)`，依然是 jdk 提供的方法，分析到这里，我们会发现事务的大部分操作都是 spring 对 jdk 方法的封装。

好了，本文的分析就先到了，关于事务的提交、回滚、回滚到保存点、恢复挂起的事务等，下篇文章继续分析。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4947826](https://my.oschina.net/funcy/blog/4947826) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_