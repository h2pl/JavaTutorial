本文是《事务的隔离级别与传播方式的处理》分析的第 4 篇，接上文，我们继续。

#### 3.6 执行具体的业务

这块代码如下：

```
retVal = invocation.proceedWithInvocation();

```

这里最终会调用到业务方法 `UserService#insert`，本文并不会探究这其中是如何一步步调用过去的，要了解调用过程的小伙伴可以参考 aop 相关操作：

*   [spring aop 之 jdk 动态代理](https://my.oschina.net/funcy/blog/4696654)
*   [spring aop 之 cglib 代理](https://my.oschina.net/funcy/blog/4696655)

#### 3.7 异常回滚

处理异常的方法为 `TransactionAspectSupport#completeTransactionAfterThrowing`，代码如下：

```
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        // 异常符合才回滚
        if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
            try {
                txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
            }
            catch (...) {
                ...
            }
        }
        else {
            try {
                // 异常不符合，即使执行出错也会提交
                txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
            }
            catch (...) {
                ...
            }
        }
    }
}

```

从这个方法来看，出现了异常并不会直接进行回滚的，而是先要判断异常类型，对于需要回滚的异常才回滚。

##### 判断当前异常是否要回滚

判断异常是否符合的方法为 `RuleBasedTransactionAttribute#rollbackOn`：

```
public boolean rollbackOn(Throwable ex) {
    RollbackRuleAttribute winner = null;
    int deepest = Integer.MAX_VALUE;
    if (this.rollbackRules != null) {
        for (RollbackRuleAttribute rule : this.rollbackRules) {
            // 获取异常的深度
            int depth = rule.getDepth(ex);
            // 深度满足条件，表示当前异常需要回滚
            if (depth >= 0 && depth < deepest) {
                deepest = depth;
                winner = rule;
            }
        }
    }
    if (winner == null) {
        return super.rollbackOn(ex);
    }
    return !(winner instanceof NoRollbackRuleAttribute);
}

```

获取树深的方法为 `RollbackRuleAttribute#getDepth(Throwable)`，代码如下：

```
public int getDepth(Throwable ex) {
    return getDepth(ex.getClass(), 0);
}

private int getDepth(Class<?> exceptionClass, int depth) {
    if (exceptionClass.getName().contains(this.exceptionName)) {
        // Found it!
        return depth;
    }
    // If we've gone as far as we can go and haven't found it...
    if (exceptionClass == Throwable.class) {
        return -1;
    }
    // 递归获取
    return getDepth(exceptionClass.getSuperclass(), depth + 1);
}

```

这个实现很简单，就是递归获取 `exception` 的父类，找到了就返回递归的次数，如果最终找到的是 `Throwable`，就返回 - 1.

之所以使用树深来判断是否需要回滚，原因是设置回滚的异常时，可以设置异常名称：

```
public @interface Transactional {
    ...

    // 注意这个类型是字符串
    String[] rollbackForClassName() default {};
}

```

因此不能使用 `ex instanceof Exception` 的方式来判断能否回滚。

`RuleBasedTransactionAttribute#rollbackOn` 中的 `RollbackRuleAttribute` 与 `NoRollbackRuleAttribute` 又是啥呢？它们 `rollbackFor` 与 `noRollbackFor` 的包装类，在 `SpringTransactionAnnotationParser#parseTransactionAnnotation(AnnotationAttributes)` 方法中设置，代码如下：

```
protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
    RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();
    ...
    // 处理回滚异常
    List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
    for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
        rollbackRules.add(new RollbackRuleAttribute(rbRule));
    }
    for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
        rollbackRules.add(new RollbackRuleAttribute(rbRule));
    }
    // 处理不回滚异常
    for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
        rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
    }
    for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
        rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
    }
    // 处理规则
    rbta.setRollbackRules(rollbackRules);
    return rbta;
}

```

##### 回滚操作

回滚操作在 `AbstractPlatformTransactionManager#rollback` 方法中处理，除了数据库的回滚操作外，这个方法还处理了一些回调处理，这里我们就不分析了，直接看关键的回滚代码。在前面的分析事务的传播类型时，大多传播类型是回滚事务，不过 `PROPAGATION_NESTED` 例外，它是回滚到保存点，这里我们分别来看看关键代码：

1. 事务回滚

   处理事务回滚的方法为 `DataSourceTransactionManager#doRollback`，最终调用的是 `java.sql.Connection` 的方法：

   ```
   protected void doRollback(DefaultTransactionStatus status) {
       DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
       // 获取连接，Connection 为 java.sql.Connection
       Connection con = txObject.getConnectionHolder().getConnection();
       try {
           con.rollback();
       }
       catch (SQLException ex) {
           throw new TransactionSystemException("Could not roll back JDBC transaction", ex);
       }
   }
   
   ```

2. 回滚到保存点 处理回滚到保存点的操作在 `AbstractTransactionStatus#rollbackToHeldSavepoint` 方法中：

   ```
   public void rollbackToHeldSavepoint() throws TransactionException {
       Object savepoint = getSavepoint();
       if (savepoint == null) {
           throw new TransactionUsageException(...);
       }
       // 回滚到保存点
       getSavepointManager().rollbackToSavepoint(savepoint);
       // 释放保存点
       getSavepointManager().releaseSavepoint(savepoint);
       // 将保存点置为null
       setSavepoint(null);
   }
   
   ```

   这个方法主要有两个操作：回滚到保存点与释放保存点。

   回滚到保存点的操作在 `JdbcTransactionObjectSupport#rollbackToSavepoint` 方法：

   ```
   public void rollbackToSavepoint(Object savepoint) throws TransactionException {
       ConnectionHolder conHolder = getConnectionHolderForSavepoint();
       try {
           conHolder.getConnection().rollback((Savepoint) savepoint);
           conHolder.resetRollbackOnly();
       }
       catch (Throwable ex) {
           throw new TransactionSystemException("Could not roll back to JDBC savepoint", ex);
       }
   }
   
   ```

   释放保存点的操作在 `JdbcTransactionObjectSupport#releaseSavepoint` 方法：

   ```
   public void releaseSavepoint(Object savepoint) throws TransactionException {
       ConnectionHolder conHolder = getConnectionHolderForSavepoint();
       try {
           conHolder.getConnection().releaseSavepoint((Savepoint) savepoint);
       }
       catch (Throwable ex) {
           logger.debug("Could not explicitly release JDBC savepoint", ex);
       }
   }
   
   ```

   最终都是调用 `java.sql.Connection` 提供的方法来完成操作。

##### 提交操作

再来看看提交操作，处理提交操作的代码为

```
txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());

```

我们跟进这个方法，一直跟到 `AbstractPlatformTransactionManager#processCommit`：

```
private void processCommit(DefaultTransactionStatus status) throws TransactionException {
    try {
        ...

        try {
            if (status.hasSavepoint()) {
                ...
                unexpectedRollback = status.isGlobalRollbackOnly();
                // 1\. 释放保存点
                status.releaseHeldSavepoint();
            }
            else if (status.isNewTransaction()) {
                ...
                unexpectedRollback = status.isGlobalRollbackOnly();
                // 2\. 处理提交操作
                doCommit(status);
            }
            else if (isFailEarlyOnGlobalRollbackOnly()) {
                ...
            }

            ...
        }
        catch (...) {
            ...
        }
    }
    finally {
        // 3\. 处理完成操作，在这里会恢复挂起的事务(恢复数据库连接)
        cleanupAfterCompletion(status);
    }
}

```

以上方法省略了大量代码，大多是与 `TransactionSynchronization` 回调相关的，我们聚集主要操作：

1.  释放保存点：这个操作在上面已经分析过了，这里就不再分析了
2.  处理提交操作：提交在这里进行，一会分析，最后我们会发现它是调用了 `java.sql.Connection` 提供的方法
3.  处理完成操作：这个操作比较重要，重置连接的信息、恢复挂起事务连接就是在这里进行的

我们先来看提交操作，直接进入最终代码：`DataSourceTransactionManager#doCommit`

```
protected void doCommit(DefaultTransactionStatus status) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
    // 获取连接，Connection 为 java.sql.Connection
    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        // 提交事务
        con.commit();
    }
    catch (SQLException ex) {
        throw new TransactionSystemException("Could not commit JDBC transaction", ex);
    }
}

```

最终也是调用 `java.sql.Connection` 提供的方法。

再来看下完成操作的处理，进入 `AbstractPlatformTransactionManager#cleanupAfterCompletion` 方法：

```
private void cleanupAfterCompletion(DefaultTransactionStatus status) {
    status.setCompleted();
    if (status.isNewSynchronization()) {
        TransactionSynchronizationManager.clear();
    }
    if (status.isNewTransaction()) {
        // 这里会重置连接，如果是新连接，在这里会关闭连接
        doCleanupAfterCompletion(status.getTransaction());
    }
    // 如果有挂起的事务，在这里进行恢复
    if (status.getSuspendedResources() != null) {
        Object transaction = (status.hasTransaction() ? status.getTransaction() : null);
        // 恢复挂起的事务
        resume(transaction, (SuspendedResourcesHolder) status.getSuspendedResources());
    }
}

```

我们先来看 `DataSourceTransactionManager#doCleanupAfterCompletion` 方法:

```
protected void doCleanupAfterCompletion(Object transaction) {
    DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;

    // 移除数据源与连接的绑定关系
    if (txObject.isNewConnectionHolder()) {
        TransactionSynchronizationManager.unbindResource(obtainDataSource());
    }

    // 重置连接，就是将连接信息恢复到执行事务前的状态
    Connection con = txObject.getConnectionHolder().getConnection();
    try {
        if (txObject.isMustRestoreAutoCommit()) {
            con.setAutoCommit(true);
        }
        DataSourceUtils.resetConnectionAfterTransaction(
                con, txObject.getPreviousIsolationLevel(), txObject.isReadOnly());
    }
    catch (Throwable ex) {
        logger.debug("Could not reset JDBC Connection after transaction", ex);
    }
    // 如果是新连接，用完就在这里关闭连接
    if (txObject.isNewConnectionHolder()) {
        // 最终调用的是 java.sql.Connection#close
        DataSourceUtils.releaseConnection(con, this.dataSource);
    }

    txObject.getConnectionHolder().clear();
}

```

再来看看事务的恢复操作，也就是 `resume(...)` 方法，跟进这个方法后，发现最终调用的是 `DataSourceTransactionManager#doResume` 方法，代码如下：

```
@Override
protected void doResume(@Nullable Object transaction, Object suspendedResources) {
    // 将数据源、挂起的数据库连接与当前线程绑定
    TransactionSynchronizationManager.bindResource(obtainDataSource(), suspendedResources);
}

```

处理完这一步后，此时的数据源就是之前挂起的数据源了。

#### 3.8 重置事务信息

在某些传播方式下，比如 `PROPAGATION_REQUIRES_NEW`，我们需要挂起当前事务，然后创建新的事务，在新事务执行完成后，需要恢复原来的事务，这里的重置事务信息就是将当前事务信息恢复为挂起的事务的信息（只是恢复了事务信息，挂起的数据库连接不是在这里恢复）：

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

        ...

        private void restoreThreadLocalStatus() {
            // 设置为旧的事务信息
            transactionInfoHolder.set(this.oldTransactionInfo);
        }

        ...
    }

}

```

可以看到，`TransactionInfo` 对换持有上一个事务的信息（`oldTransactionInfo` 的成员变量），重置回上一个事务信息时，只是简单地将 `oldTransactionInfo` 设置到名为 `transactionInfoHolder` 的 `ThreadLocal` 里。

#### 3.9 提交事务

处理事务提交的代码为 `TransactionAspectSupport#commitTransactionAfterReturning`，代码如下：

```
protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
    // 判断下事务的状态
    if (txInfo != null && txInfo.getTransactionStatus() != null) {
        // 处理提交操作，这前面已经分析过了
        txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
    }
}

```

在该方法中，我们又看到了这行代码：

```
txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());

```

这个已经在由**异常引起的事务提交**中分析过了，就不再分析了。

* * *

_本文原文链接：[https://my.oschina.net/funcy/blog/4947800](https://my.oschina.net/funcy/blog/4947800) ，限于作者个人水平，文中难免有错误之处，欢迎指正！原创不易，商业转载请联系作者获得授权，非商业转载请注明出处。_