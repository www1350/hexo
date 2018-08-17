---
title: springtransaction 事务
tags:
  - spring
  - 事务
categories:
  - spring
  - 事务
abbrlink: c930c494
date: 2016-07-06 22:26:20
---

事务传播属性  

```java
    @Transactional(propagation=Propagation.REQUIRED) //如果有事务,那么加入事务,没有的话新建一个(不写的情况下)  **默认**
    @Transactional(propagation=Propagation.NOT_SUPPORTED) //容器不为这个方法开启事务  
    @Transactional(propagation=Propagation.REQUIRES_NEW) //不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务  
    @Transactional(propagation=Propagation.MANDATORY) //必须在一个已有的事务中执行,否则抛出异常  
    @Transactional(propagation=Propagation.NEVER) //必须在一个没有的事务中执行,否则抛出异常(与Propagation.MANDATORY相反)  
    @Transactional(propagation=Propagation.SUPPORTS) //如果其他bean调用这个方法,在其他bean中声明事务,那就用事务.如果其他bean没有声明事务,那就不用事务.  
    @Transactional(propagation=Propagation.NESTED)   //如果当前存在事务，则在嵌套事务内执行。如果当前没有事务，则进行与PROPAGATION_REQUIRED类似的操作。 
    @Transactional (propagation = Propagation.REQUIRED,readOnly=true) //readOnly=true只读,不能更新,删除   
    @Transactional (propagation = Propagation.REQUIRED,timeout=30)//设置超时时间   
    @Transactional (propagation = Propagation.REQUIRED,isolation=Isolation.DEFAULT)//设置数据库隔离级别  
```


xml配置：
```xml
  <!-- hibernate事务管理器 -->  
    <bean id="transactionManager"
        class="org.springframework.orm.hibernate3.HibernateTransactionManager">
        <property name="sessionFactory" ref="sessionFactory" />
    </bean>
    
<!-- mybatis配置事务管理 -->  
    <bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">       
          <property name="dataSource" ref="dataSource"></property>  
    </bean> 
```

1.直接配置
```    xml
    <bean id="userDao"  
        class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean">  
           <!-- 配置事务管理器 -->  
           <property name="transactionManager" ref="transactionManager" />     
        <property name="target" ref="userDaoTarget" />  
         <property name="proxyInterfaces" value="com.absurd.xxxDao" />
        <!-- 配置事务属性 -->  
        <property name="transactionAttributes">  
            <props>  
                <prop key="*">PROPAGATION_REQUIRED</prop>
            </props>  
        </property>  
    </bean>  
```

2.共享一个代理基类
```xml
<bean id="transactionBase"  
            class="org.springframework.transaction.interceptor.TransactionProxyFactoryBean"  
            lazy-init="true" abstract="true">  
        <!-- 配置事务管理器 -->  
        <property name="transactionManager" ref="transactionManager" />  
        <!-- 配置事务属性 -->  
        <property name="transactionAttributes">  
            <props>  
                <prop key="*">PROPAGATION_REQUIRED</prop>  
            </props>  
        </property>  
    </bean>    
   
    
    <bean id="userDao" parent="transactionBase" >  
        <property name="target" ref="userDaoTarget" />   
    </bean>
```

3.

 ```xml
  <bean id="transactionInterceptor"  
        class="org.springframework.transaction.interceptor.TransactionInterceptor">  
        <property name="transactionManager" ref="transactionManager" />  
        <!-- 配置事务属性 -->  
        <property name="transactionAttributes">  
            <props>  
                <prop key="*">PROPAGATION_REQUIRED</prop>  
            </props>  
        </property>  
    </bean>
      
    <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">  
        <property name="beanNames">  
            <list>  
                <value>*Dao</value>
            </list>  
        </property>  
        <property name="interceptorNames">  
            <list>  
                <value>transactionInterceptor</value>  
            </list>  
        </property>  
    </bean>  
 ```

4.使用tx标签配置的拦截器
 ```xml
  <tx:advice id="txAdvice" transaction-manager="transactionManager">
        <tx:attributes>
            <tx:method name="*" propagation="REQUIRED" />
        </tx:attributes>
    </tx:advice>
    
    <aop:config>
        <aop:pointcut id="interceptorPointCuts"
            expression="execution(* com.absurd.*.*(..))" />
        <aop:advisor advice-ref="txAdvice"
            pointcut-ref="interceptorPointCuts" />        
    </aop:config>      
 ```

5.全注解

 ` <tx:annotation-driven transaction-manager="transactionManager"/>`

6.编程：
```java
DefaultTransactionDefinition def = new DefaultTransactionDefinition();
def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

TransactionStatus status = txManager.getTransaction(def);
try {
  userMapper.insertUser(user);
}
catch (MyException ex) {
  txManager.rollback(status);
  throw ex;
}
txManager.commit(status);
```

TransactionStatus
```
	boolean isNewTransaction();

	boolean hasSavepoint();

	void setRollbackOnly();

	boolean isRollbackOnly();

	void flush();

	boolean isCompleted();
```

PlatformTransactionManager
```java
	TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;

	void commit(TransactionStatus status) throws TransactionException;

	void rollback(TransactionStatus status) throws TransactionException;
```


源码解析：
从TxNamespaceHandler入手

```java
	@Override
	public void init() {
		registerBeanDefinitionParser("advice", new TxAdviceBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-driven", new AnnotationDrivenBeanDefinitionParser());
		registerBeanDefinitionParser("jta-transaction-manager", new JtaTransactionManagerBeanDefinitionParser());
	}
```

可以看到分别用不同的解析器去解析xml，
TxAdviceBeanDefinitionParser用来解析`<tx:advice>`并被解析为RuleBasedTransactionAttribute。

parse(Element element, ParserContext parserContext)->parseInternal(Element element, ParserContext parserContext)->getBeanClass(Element element)->TransactionInterceptor
解析到的bean被设置为TransactionInterceptor统一拦截

![image](https://user-images.githubusercontent.com/7789698/34664349-5bba05ca-f496-11e7-94f1-11355666c5d3.png)


AnnotationDrivenBeanDefinitionParser是用来解析`<annotation-driven>`为RootBeanDefinition

![image](https://user-images.githubusercontent.com/7789698/34664424-c8565878-f496-11e7-9456-1b8b3bcb93f3.png)


DataSourceTransactionManager 和DataSourceTransactionManager
![image](https://user-images.githubusercontent.com/7789698/34664675-fa586a9a-f497-11e7-997e-7196bdbc63e2.png)

TransactionInterceptor
```
	@Override
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		// Work out the target class: may be {@code null}.
		// The TransactionAttributeSource should be passed the target class
		// as well as the method, which may be from an interface.
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// Adapt to TransactionAspectSupport's invokeWithinTransaction...
		return invokeWithinTransaction(invocation.getMethod(), targetClass, new InvocationCallback() {
			@Override
			public Object proceedWithInvocation() throws Throwable {
				return invocation.proceed();
			}
		});
	}



protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)
			throws Throwable {
		// If the transaction attribute is null, the method is non-transactional.
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
//选择事务管理器
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
//切面方法标识
		final String joinpointIdentification = methodIdentification(method, targetClass);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
//原有逻辑执行
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
//清理TransactionInfo信息
				cleanupTransactionInfo(txInfo);
			}
//提交事务
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		else {
			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr,
						new TransactionCallback<Object>() {
							@Override
							public Object doInTransaction(TransactionStatus status) {
								TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
								try {
									return invocation.proceedWithInvocation();
								}
								catch (Throwable ex) {
									if (txAttr.rollbackOn(ex)) {
										// A RuntimeException: will lead to a rollback.
										if (ex instanceof RuntimeException) {
											throw (RuntimeException) ex;
										}
										else {
											throw new ThrowableHolderException(ex);
										}
									}
									else {
										// A normal return value: will lead to a commit.
										return new ThrowableHolder(ex);
									}
								}
								finally {
									cleanupTransactionInfo(txInfo);
								}
							}
						});

				// Check result: It might indicate a Throwable to rethrow.
				if (result instanceof ThrowableHolder) {
					throw ((ThrowableHolder) result).getThrowable();
				}
				else {
					return result;
				}
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
		}
	}


protected TransactionInfo createTransactionIfNecessary(
			PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
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
//获取事务状态
				status = tm.getTransaction(txAttr);
			}
			else {
//debug..
			}
		}
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}

```



接下来看下：AbstractPlatformTransactionManager#getTransaction

```
@Override
	public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
		Object transaction = doGetTransaction();

//...
                //如果存在事务
		if (isExistingTransaction(transaction)) {
			
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

		// Check definition settings for new transaction.
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		//不存在事务，Propagation.MANDATORY，抛出异常
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException(
					"No existing transaction found for transaction marked with propagation 'mandatory'");
		}
//PROPAGATION_REQUIRED、PROPAGATION_REQUIRES_NEW、PROPAGATION_NESTED
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
//不用挂起
			SuspendedResourcesHolder suspendedResources = suspend(null);
//debug...
			}
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException ex) {
				resume(null, suspendedResources);
				throw ex;
			}
			catch (Error err) {
				resume(null, suspendedResources);
				throw err;
			}
		}
		else {
			//创建空事务，同步
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT && logger.isWarnEnabled()) {
				logger.warn("Custom isolation level specified but no actual transaction initiated; " +
						"isolation level will effectively be ignored: " + definition);
			}
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}
```


```java
private TransactionStatus handleExistingTransaction(
			TransactionDefinition definition, Object transaction, boolean debugEnabled)
			throws TransactionException {
//PROPAGATION_NEVER抛出异常(Propagation.NEVER)
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NEVER) {
			throw new IllegalTransactionStateException(
					"Existing transaction found for transaction marked with propagation 'never'");
		}
//PROPAGATION_NOT_SUPPORTED不为这个方法开启事务（Propagation.NOT_SUPPORTED）
//线程存在事务则挂起（挂起实现其实就是断开连接下次进行重连）
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NOT_SUPPORTED) {
//debug...
			Object suspendedResources = suspend(transaction);
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
 //创建非事务的事务状态，让方法非事务地执行  
			return prepareTransactionStatus(
					definition, null, false, newSynchronization, debugEnabled, suspendedResources);
		}
//PROPAGATION_REQUIRES_NEW不管是否存在事务,都创建一个新的事务,原来的挂起,新的执行完毕,继续执行老的事务(Propagation.REQUIRES_NEW )
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW) {
			//...
//挂起原来事务
			SuspendedResourcesHolder suspendedResources = suspend(transaction);
			try {
//只要不是never就执行完毕新的
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException beginEx) {
//异常了恢复原来事务
				resumeAfterBeginException(transaction, suspendedResources, beginEx);
				throw beginEx;
			}
			catch (Error beginErr) {
				resumeAfterBeginException(transaction, suspendedResources, beginErr);
				throw beginErr;
			}
		}
//NESTED如果是嵌套事务（TransactionDefinition.PROPAGATION_NESTED）
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
//如果不允许事务嵌套，则抛出异常
			if (!isNestedTransactionAllowed()) {
				throw new NestedTransactionNotSupportedException(
						"Transaction manager does not allow nested transactions by default - " +"specify 'nestedTransactionAllowed' property with value 'true'");
			}
//debug...
			}
//如果允许使用savepoint保存点保存嵌套事务  
			if (useSavepointForNestedTransaction()) {
				   //为当前事务创建一个保存点  
				DefaultTransactionStatus status =
						prepareTransactionStatus(definition, transaction, false, false, debugEnabled, null);
				status.createAndHoldSavepoint();
				return status;
			}
			else {
				 //如果不允许使用savepoint保存点保存嵌套事务 ,使用JTA的嵌套commit/rollback调用  
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, null);
				doBegin(transaction, definition);
				prepareSynchronization(status, definition);
				return status;
			}
		}

//debug...
//PROPAGATION_SUPPORTS or PROPAGATION_REQUIRED.
 //校验已存在的事务，如果已有事务与事务属性配置不一致，则抛出异常  
		if (isValidateExistingTransaction()) {
  //如果事务隔离级别不是默认隔离级别 ，获取到的当前事务隔离级别为null 或 获取不等于事务属性配置的隔离级别
			if (definition.getIsolationLevel() != TransactionDefinition.ISOLATION_DEFAULT) {
				Integer currentIsolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
				if (currentIsolationLevel == null || currentIsolationLevel != definition.getIsolationLevel()) {
					Constants isoConstants = DefaultTransactionDefinition.constants;
					throw new IllegalTransactionStateException("Participating transaction with definition [" +
							definition + "] specifies isolation level which is incompatible with existing transaction: " +
							(currentIsolationLevel != null ?
									isoConstants.toCode(currentIsolationLevel, DefaultTransactionDefinition.PREFIX_ISOLATION) :
									"(unknown)"));
				}
			}
                //如果当前已有事务是只读 ，抛出异常
			if (!definition.isReadOnly()) {
				if (TransactionSynchronizationManager.isCurrentTransactionReadOnly()) {
					throw new IllegalTransactionStateException("Participating transaction with definition [" +
							definition + "] is not marked as read-only but existing transaction is");
				}
			}
		}
		boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);
		return prepareTransactionStatus(definition, transaction, false, newSynchronization, debugEnabled, null);
	}
```

```java
	@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;
		try {
			if (txObject.getConnectionHolder() == null ||
					txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
				Connection newCon = this.dataSource.getConnection();
//debug...
				}
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}

			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();

			Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
			txObject.setPreviousIsolationLevel(previousIsolationLevel);

			
			if (con.getAutoCommit()) {
				txObject.setMustRestoreAutoCommit(true);
				//debug...
				con.setAutoCommit(false);
			}
			txObject.getConnectionHolder().setTransactionActive(true);

			int timeout = determineTimeout(definition);
			if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
				txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
			}

			// Bind the session holder to the thread.
			if (txObject.isNewConnectionHolder()) {
				TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
			}
		}

		catch (Throwable ex) {
			if (txObject.isNewConnectionHolder()) {
				DataSourceUtils.releaseConnection(con, this.dataSource);
				txObject.setConnectionHolder(null, false);
			}
			throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
		}
	}
```

```java
protected final SuspendedResourcesHolder suspend(Object transaction) throws TransactionException {
//如果事务是激活的，且当前线程事务同步机制也是激活状态  
		if (TransactionSynchronizationManager.isSynchronizationActive()) {
//挂起当前线程中所有同步的事务  TransactionSynchronization#suspend   还有 TransactionSynchronizationManager#unbindResource
			List<TransactionSynchronization> suspendedSynchronizations = doSuspendSynchronization();
			try {
				Object suspendedResources = null;
				if (transaction != null) {
					suspendedResources = doSuspend(transaction);
				}
				String name = TransactionSynchronizationManager.getCurrentTransactionName();
				TransactionSynchronizationManager.setCurrentTransactionName(null);
				boolean readOnly = TransactionSynchronizationManager.isCurrentTransactionReadOnly();
				TransactionSynchronizationManager.setCurrentTransactionReadOnly(false);
				Integer isolationLevel = TransactionSynchronizationManager.getCurrentTransactionIsolationLevel();
				TransactionSynchronizationManager.setCurrentTransactionIsolationLevel(null);
				boolean wasActive = TransactionSynchronizationManager.isActualTransactionActive();
				TransactionSynchronizationManager.setActualTransactionActive(false);
				return new SuspendedResourcesHolder(
						suspendedResources, suspendedSynchronizations, name, readOnly, isolationLevel, wasActive);
			}
			catch (RuntimeException ex) {
				// doSuspend failed - original transaction is still active...
				doResumeSynchronization(suspendedSynchronizations);
				throw ex;
			}
			catch (Error err) {
				// doSuspend failed - original transaction is still active...
				doResumeSynchronization(suspendedSynchronizations);
				throw err;
			}
		}
		else if (transaction != null) {
			// Transaction active but no synchronization active.
			Object suspendedResources = doSuspend(transaction);
			return new SuspendedResourcesHolder(suspendedResources);
		}
		else {
			// Neither transaction nor synchronization active.
			return null;
		}
	}



//doSuspendSynchronization方法将逐个挂起当前线程中的TransactionSynchronization
private List<TransactionSynchronization> doSuspendSynchronization() {
		List<TransactionSynchronization> suspendedSynchronizations =
				TransactionSynchronizationManager.getSynchronizations();
		for (TransactionSynchronization synchronization : suspendedSynchronizations) {
			synchronization.suspend();
		}
		TransactionSynchronizationManager.clearSynchronization();
		return suspendedSynchronizations;
	}

//ConnectionSynchronization#suspend
		public void suspend() {
			if (this.holderActive) {
//解绑数据源
				TransactionSynchronizationManager.unbindResource(this.dataSource);
// 如果存在连接，且处于打开状态
				if (this.connectionHolder.hasConnection() && !this.connectionHolder.isOpen()) {
 // 当挂起的时候如果没有句柄连接到该connection，将释放该连接
 // 当resume的时候 会重开打开一个连接参与到原来的事务中
					releaseConnection(this.connectionHolder.getConnection(), this.dataSource);
					this.connectionHolder.setConnection(null);
				}
			}
		}

//恢复事务
	private void doResumeSynchronization(List<TransactionSynchronization> suspendedSynchronizations) {
		TransactionSynchronizationManager.initSynchronization();
		for (TransactionSynchronization synchronization : suspendedSynchronizations) {
			synchronization.resume();
			TransactionSynchronizationManager.registerSynchronization(synchronization);
		}
	}

//SqlSessionSynchronization#resume
    @Override
    public void resume() {
      if (this.holderActive) {
       //debug...
        TransactionSynchronizationManager.bindResource(this.sessionFactory, this.holder);
      }
    }


//DataSourceTransactionManager#doSuspend
	@Override
	protected Object doSuspend(Object transaction) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		txObject.setConnectionHolder(null);
		ConnectionHolder conHolder = (ConnectionHolder)
				TransactionSynchronizationManager.unbindResource(this.dataSource);
		return conHolder;
	}
```
## 事务提交
AbstractPlatformTransactionManager##commit
```java
	@Override
	public final void commit(TransactionStatus status) throws TransactionException {
//如果事务状态是已完成，再次提交会抛出“Transaction is already completed - do not call commit or rollback more than once per transaction”
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException(
					"Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
//rollback only
		if (defStatus.isLocalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Transactional code has requested rollback");
			}
//回滚
			processRollback(defStatus);
			return;
		}
		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			if (defStatus.isDebug()) {
				logger.debug("Global transaction is marked as rollback-only but transactional code requested commit");
			}
			processRollback(defStatus);
			// Throw UnexpectedRollbackException only at outermost transaction boundary
			// or if explicitly asked to.
			if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
				throw new UnexpectedRollbackException(
						"Transaction rolled back because it has been marked as rollback-only");
			}
			return;
		}

		processCommit(defStatus);
	}

private void processRollback(DefaultTransactionStatus status) {
		try {
			try {
//回调TransactionSynchronization对象的beforeCompletion方法。
				triggerBeforeCompletion(status);
//有保存点
				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Rolling back transaction to savepoint");
					}
//回滚到保存点
					status.rollbackToHeldSavepoint();
				}
//如果是一个新事务
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction rollback");
					}
//rollback
					doRollback(status);
				}
				else if (status.hasTransaction()) {
//如果RollbackOnly或者globalRollbackOnParticipationFailure（部分失败）
					if (status.isLocalRollbackOnly() || isGlobalRollbackOnParticipationFailure()) {
						if (status.isDebug()) {
							logger.debug("Participating transaction failed - marking existing transaction as rollback-only");
						}
						doSetRollbackOnly(status);
					}
					else {
						if (status.isDebug()) {
							logger.debug("Participating transaction failed - letting transaction originator decide on rollback");
						}
					}
				}
				else {
					logger.debug("Should roll back transaction but cannot - no transaction available");
				}
			}
			catch (RuntimeException ex) {
//TransactionSynchronization 的afterCompletion
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				throw ex;
			}
			catch (Error err) {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				throw err;
			}
			triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
		}
		finally {
			cleanupAfterCompletion(status);
		}
	}


private void processCommit(DefaultTransactionStatus status) throws TransactionException {
		try {
			boolean beforeCompletionInvoked = false;
			try {
				prepareForCommit(status);
//TransactionSynchronization 的beforeCommit 提交前提示
				triggerBeforeCommit(status);
//TransactionSynchronization 的beforeCompletion 完成前提示
				triggerBeforeCompletion(status);
				beforeCompletionInvoked = true;
				boolean globalRollbackOnly = false;
				if (status.isNewTransaction() || isFailEarlyOnGlobalRollbackOnly()) {
					globalRollbackOnly = status.isGlobalRollbackOnly();
				}
				if (status.hasSavepoint()) {
					if (status.isDebug()) {
						logger.debug("Releasing transaction savepoint");
					}
					status.releaseHeldSavepoint();
				}
				else if (status.isNewTransaction()) {
					if (status.isDebug()) {
						logger.debug("Initiating transaction commit");
					}
//提交事务
					doCommit(status);
				}
				// Throw UnexpectedRollbackException if we have a global rollback-only
				// marker but still didn't get a corresponding exception from commit.
				if (globalRollbackOnly) {
					throw new UnexpectedRollbackException(
							"Transaction silently rolled back because it has been marked as rollback-only");
				}
			}
			catch (UnexpectedRollbackException ex) {
				// can only be caused by doCommit
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_ROLLED_BACK);
				throw ex;
			}
			catch (TransactionException ex) {
				// can only be caused by doCommit
				if (isRollbackOnCommitFailure()) {
					doRollbackOnCommitException(status, ex);
				}
				else {
					triggerAfterCompletion(status, TransactionSynchronization.STATUS_UNKNOWN);
				}
				throw ex;
			}
			catch (RuntimeException ex) {
				if (!beforeCompletionInvoked) {
					triggerBeforeCompletion(status);
				}
				doRollbackOnCommitException(status, ex);
				throw ex;
			}
			catch (Error err) {
				if (!beforeCompletionInvoked) {
					triggerBeforeCompletion(status);
				}
				doRollbackOnCommitException(status, err);
				throw err;
			}

			// Trigger afterCommit callbacks, with an exception thrown there
			// propagated to callers but the transaction still considered as committed.
			try {
				triggerAfterCommit(status);
			}
			finally {
				triggerAfterCompletion(status, TransactionSynchronization.STATUS_COMMITTED);
			}

		}
		finally {
			cleanupAfterCompletion(status);
		}
	}
```

隔离级别（其实最后是通过jdbc的隔离级别做的）：
> Isolation.DEFAULT(TransactionDefinition.ISOLATION_DEFAULT)
> 使用数据库默认的事务隔离级别。

> Isolation.READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),
> 这是事务最低的隔离级别，它允许另外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。
> 实现：`SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED`

> Isolation.READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),
> 保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。这种事务隔离级别可以避免脏读出现，但是可能会出现不可重复读和幻像读。
> 实现：` SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED`

> Isolation.REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),
> 这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻像读。
> 实现：`SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ`

> Isolation.SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);
> 这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻读。
> 实现：`SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE`

详细：http://www1350.github.io/#post/64
