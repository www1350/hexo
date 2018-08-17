---
title: sharding-jdbc源码解析-JDBC实现（一）
tags:
  - sharding-jdbc
categories:
  - 中间件
  - 数据库
abbrlink: b792e0dc
date: 2018-05-06 22:37:12
---

## JDBC接口

```java
public interface DataSource  extends CommonDataSource, Wrapper {
  //建立和数据源的连接
  Connection getConnection() throws SQLException;

  Connection getConnection(String username, String password)
    throws SQLException;
}
```

![image](https://user-images.githubusercontent.com/7789698/39674488-3df76816-517f-11e8-97b6-343b9894a332.png)

```java
public interface Connection  extends Wrapper, AutoCloseable {
    //创建一个Statement对象，可以用来发送sql语句
    Statement createStatement() throws SQLException;

    //创建一个PreparedStatement对象，发送参数化的sql语句。sql语句将会被预编译存储在对象
    PreparedStatement prepareStatement(String sql)
        throws SQLException;
    
    //创建一个CallableStatement对象，发送调用存储过程语句
    CallableStatement prepareCall(String sql) throws SQLException;

    //将给定的SQL语句转换成系统的原生SQL语法
    String nativeSQL(String sql) throws SQLException;

    //设置自动提交参数
    void setAutoCommit(boolean autoCommit) throws SQLException;

    boolean getAutoCommit() throws SQLException;

    //发送commit请求
    void commit() throws SQLException;

    //发送rollback请求
    void rollback() throws SQLException;

    //不等到连接自动释放，马上关闭连接
    void close() throws SQLException;

    boolean isClosed() throws SQLException;

    //获取数据库元数据
    DatabaseMetaData getMetaData() throws SQLException;

    //设置只读
    void setReadOnly(boolean readOnly) throws SQLException;

    boolean isReadOnly() throws SQLException;

    //设置catalog（数据库名）
    void setCatalog(String catalog) throws SQLException;

    String getCatalog() throws SQLException;

    /**
     * A constant indicating that transactions are not supported.
     */
    int TRANSACTION_NONE             = 0;

    //读未提交级别
    int TRANSACTION_READ_UNCOMMITTED = 1;

    //读提交级别rc
    int TRANSACTION_READ_COMMITTED   = 2;

    //可重复读rr
    int TRANSACTION_REPEATABLE_READ  = 4;

    //串行化
    int TRANSACTION_SERIALIZABLE     = 8;

    //设置连接的事务隔离级别
    void setTransactionIsolation(int level) throws SQLException;

    int getTransactionIsolation() throws SQLException;

    //获取此 Connection 对象上的调用报告的第一个警告。如果有多个警告，则后续警告将被链接到第一个警告，可以通过对之前获得的警告调用 SQLWarning.getNextWarning 方法获取。
    SQLWarning getWarnings() throws SQLException;

    //清除为此 Connection 对象报告的所有警告。调用此方法后，在为此 Connection 对象报告新的警告前，getWarnings 方法将返回 null。
    void clearWarnings() throws SQLException;


    //--------------------------JDBC 2.0-----------------------------

    //创建一个 Statement 对象，该对象将生成具有给定类型和并发性的 ResultSet 对象。此方法与上述 createStatement 方法相同，但它允许重写默认结果集类型和并发性。
    //resultSetType - 结果集类型，它是 ResultSet.TYPE_FORWARD_ONLY、ResultSet.TYPE_SCROLL_INSENSITIVE 或 ResultSet.TYPE_SCROLL_SENSITIVE 之一
    //resultSetConcurrency - 并发类型；它是 ResultSet.CONCUR_READ_ONLY 或 ResultSet.CONCUR_UPDATABLE 之一
    Statement createStatement(int resultSetType, int resultSetConcurrency)
        throws SQLException;

    //创建一个 PreparedStatement 对象，该对象将生成具有给定类型和并发性的 ResultSet 对象。此方法与上述 prepareStatement 方法相同，但它允许重写默认结果集类型和并发性。
	//resultSetType - 结果集类型，它是 ResultSet.TYPE_FORWARD_ONLY、ResultSet.TYPE_SCROLL_INSENSITIVE 或 ResultSet.TYPE_SCROLL_SENSITIVE 之一
	//resultSetConcurrency - 并发类型，它是 ResultSet.CONCUR_READ_ONLY 或 ResultSet.CONCUR_UPDATABLE 之一
    PreparedStatement prepareStatement(String sql, int resultSetType,
                                       int resultSetConcurrency)
        throws SQLException;

    CallableStatement prepareCall(String sql, int resultSetType,
                                  int resultSetConcurrency) throws SQLException;

    //
    java.util.Map<String,Class<?>> getTypeMap() throws SQLException;

  
    void setTypeMap(java.util.Map<String,Class<?>> map) throws SQLException;

    //--------------------------JDBC 3.0-----------------------------

    // ResultSet.HOLD_CURSORS_OVER_COMMIT or ResultSet.CLOSE_CURSORS_AT_COMMIT
    void setHoldability(int holdability) throws SQLException;

    int getHoldability() throws SQLException;

    //有时候一个事务可能是一组复杂的语句，因此可能想要回滚到事务中某个特殊的点。JDBC Savepoint帮我们在事务中创建检查点（checkpoint），这样就可以回滚到指定点。当事务提交或者整个事务回滚后，为事务产生的任何保存点都会自动释放并变为无效。把事务回滚到一个保存点，会使其他所有保存点自动释放并变为无效。
    Savepoint setSavepoint() throws SQLException;

    Savepoint setSavepoint(String name) throws SQLException;

    //回滚到检查点
    void rollback(Savepoint savepoint) throws SQLException;

    //移除检查点
    void releaseSavepoint(Savepoint savepoint) throws SQLException;

    Statement createStatement(int resultSetType, int resultSetConcurrency,
                              int resultSetHoldability) throws SQLException;

    PreparedStatement prepareStatement(String sql, int resultSetType,
                                       int resultSetConcurrency, int resultSetHoldability)
        throws SQLException;

    CallableStatement prepareCall(String sql, int resultSetType,
                                  int resultSetConcurrency,
                                  int resultSetHoldability) throws SQLException;

    //autoGeneratedKeys - 标志是否自增key应该返回Statement.RETURN_GENERATED_KEYS or Statement.NO_GENERATED_KEYS
    PreparedStatement prepareStatement(String sql, int autoGeneratedKeys)
        throws SQLException;

    //columnIndexes - 列下标数组指示插入行哪些列应该返回
    PreparedStatement prepareStatement(String sql, int columnIndexes[])
        throws SQLException;

    PreparedStatement prepareStatement(String sql, String columnNames[])
        throws SQLException;

    Clob createClob() throws SQLException;

    Blob createBlob() throws SQLException;

    NClob createNClob() throws SQLException;

    SQLXML createSQLXML() throws SQLException;

        //
         boolean isValid(int timeout) throws SQLException;

        //
         void setClientInfo(String name, String value)
                throws SQLClientInfoException;

         void setClientInfo(Properties properties)
                throws SQLClientInfoException;

         String getClientInfo(String name)
                throws SQLException;

         Properties getClientInfo()
                throws SQLException;

//
 Array createArrayOf(String typeName, Object[] elements) throws
SQLException;

//
 Struct createStruct(String typeName, Object[] attributes)
throws SQLException;

   //--------------------------JDBC 4.1 -----------------------------

   //schema
    void setSchema(String schema) throws SQLException;

    String getSchema() throws SQLException;

    //
    void abort(Executor executor) throws SQLException;

    //
    void setNetworkTimeout(Executor executor, int milliseconds) throws SQLException;

    int getNetworkTimeout() throws SQLException;
}
```

![image](https://user-images.githubusercontent.com/7789698/39674502-7550a67e-517f-11e8-9f11-f925695e2111.png)

```java
public interface Statement extends Wrapper, AutoCloseable {

    //执行sql语句返回一个ResultSet
    ResultSet executeQuery(String sql) throws SQLException;

    //执行一个增删改语句，或者没返回值的ddl语句。返回更新行数
    int executeUpdate(String sql) throws SQLException;

    //
    void close() throws SQLException;

    //返回列字符最大字节或二进制最大字节，0表示没有限制
    int getMaxFieldSize() throws SQLException;

    void setMaxFieldSize(int max) throws SQLException;

    //返回最大行数
    int getMaxRows() throws SQLException;

    void setMaxRows(int max) throws SQLException;

    //true以允许对转义处理。 否则为 false
    void setEscapeProcessing(boolean enable) throws SQLException;

    //返回语句的等待超时时间，0表示一直等待
    int getQueryTimeout() throws SQLException;

    void setQueryTimeout(int seconds) throws SQLException;

    //
    void cancel() throws SQLException;

    SQLWarning getWarnings() throws SQLException;

    void clearWarnings() throws SQLException;

    //设置游标名
    void setCursorName(String name) throws SQLException;

    //----------------------- Multiple Results --------------------------

    //返回true代表是一个ResultSet对象，false无返回或者返回的是更新行数
    boolean execute(String sql) throws SQLException;

    ResultSet getResultSet() throws SQLException;

    int getUpdateCount() throws SQLException;

    //无返回：((stmt.getMoreResults() == false) && (stmt.getUpdateCount() == -1))
    boolean getMoreResults() throws SQLException;


    //--------------------------JDBC 2.0-----------------------------


    //ResultSet.FETCH_FORWARD、ResultSet.FETCH_REVERSE、ResultSet.FETCH_UNKNOWN
    void setFetchDirection(int direction) throws SQLException;

    int getFetchDirection() throws SQLException;

    //游标拿的行数
    void setFetchSize(int rows) throws SQLException;

    int getFetchSize() throws SQLException;

    //ResultSet.CONCUR_READ_ONLY、ResultSet.CONCUR_UPDATABLE
    int getResultSetConcurrency() throws SQLException;

    //ResultSet.TYPE_FORWARD_ONLY、ResultSet.TYPE_SCROLL_INSENSITIVE、ResultSet.TYPE_SCROLL_SENSITIVE
    int getResultSetType()  throws SQLException;

    //添加sql语句，executeBatch将批量执行
    void addBatch( String sql ) throws SQLException;

    void clearBatch() throws SQLException;

    //批量执行返回每一句成功行数
    int[] executeBatch() throws SQLException;

    Connection getConnection()  throws SQLException;

  //--------------------------JDBC 3.0-----------------------------

    /**
     * The constant indicating that the current <code>ResultSet</code> object
     * should be closed when calling <code>getMoreResults</code>.
     *
     * @since 1.4
     */
    int CLOSE_CURRENT_RESULT = 1;

    /**
     * The constant indicating that the current <code>ResultSet</code> object
     * should not be closed when calling <code>getMoreResults</code>.
     *
     * @since 1.4
     */
    int KEEP_CURRENT_RESULT = 2;

    /**
     * The constant indicating that all <code>ResultSet</code> objects that
     * have previously been kept open should be closed when calling
     * <code>getMoreResults</code>.
     *
     * @since 1.4
     */
    int CLOSE_ALL_RESULTS = 3;

    /**
     * The constant indicating that a batch statement executed successfully
     * but that no count of the number of rows it affected is available.
     *
     * @since 1.4
     */
    int SUCCESS_NO_INFO = -2;

    /**
     * The constant indicating that an error occurred while executing a
     * batch statement.
     *
     * @since 1.4
     */
    int EXECUTE_FAILED = -3;

    /**
     * The constant indicating that generated keys should be made
     * available for retrieval.
     *
     * @since 1.4
     */
    int RETURN_GENERATED_KEYS = 1;

    /**
     * The constant indicating that generated keys should not be made
     * available for retrieval.
     *
     * @since 1.4
     */
    int NO_GENERATED_KEYS = 2;

    //Statement.CLOSE_CURRENT_RESULT、Statement.KEEP_CURRENT_RESULT、Statement.CLOSE_ALL_RESULTS
    boolean getMoreResults(int current) throws SQLException;

    //获取自增key
    ResultSet getGeneratedKeys() throws SQLException;

    //update，Statement.RETURN_GENERATED_KEYS、Statement.NO_GENERATED_KEY
    int executeUpdate(String sql, int autoGeneratedKeys) throws SQLException;

    //带参数
    int executeUpdate(String sql, int columnIndexes[]) throws SQLException;

    //
    int executeUpdate(String sql, String columnNames[]) throws SQLException;

    //Statement.RETURN_GENERATED_KEYS、Statement.NO_GENERATED_KEY
    boolean execute(String sql, int autoGeneratedKeys) throws SQLException;

    //带参数
    boolean execute(String sql, int columnIndexes[]) throws SQLException;

    //
    boolean execute(String sql, String columnNames[]) throws SQLException;

   //
    int getResultSetHoldability() throws SQLException;

    boolean isClosed() throws SQLException;

        //
        void setPoolable(boolean poolable)
                throws SQLException;

        boolean isPoolable()
                throws SQLException;
}
```

![image](https://user-images.githubusercontent.com/7789698/39674533-f0594a88-517f-11e8-8fc4-568fb50f671f.png)

```java
public interface PreparedStatement extends Statement {
    ResultSet executeQuery() throws SQLException;

    int executeUpdate() throws SQLException;

    //setxxx

    void clearParameters() throws SQLException;

    void setObject(int parameterIndex, Object x, int targetSqlType)
      throws SQLException;

    void setObject(int parameterIndex, Object x) throws SQLException;

    boolean execute() throws SQLException;

    //--------------------------JDBC 2.0-----------------------------
    void addBatch() throws SQLException;

    //setxxxx

    ResultSetMetaData getMetaData() throws SQLException;

    //------------------------- JDBC 3.0 -----------------------------------
    void setURL(int parameterIndex, java.net.URL x) throws SQLException;

    ParameterMetaData getParameterMetaData() throws SQLException;

    //------------------------- JDBC 4.0 -----------------------------------

    void setRowId(int parameterIndex, RowId x) throws SQLException;

     //setxxx
    //-----
}
```

![image](https://user-images.githubusercontent.com/7789698/39674527-dd1cb2c0-517f-11e8-8f7f-65fe642ddd39.png)



![image](https://user-images.githubusercontent.com/7789698/39683937-1a735c1c-51eb-11e8-8701-b9e984d585b7.png)

```java
//此接口描述访问哪些由代理代表的包装资源的标准机制，允许对资源代理的直接访问
public interface Wrapper {
    //返回一个对象，该对象实现给定接口，以允许访问非标准方法或代理未公开的标准方法
        <T> T unwrap(java.lang.Class<T> iface) throws java.sql.SQLException;

    //如果调用此方法的对象实现接口参数，或者是实现接口参数的对象的直接或间接包装器，则返回 true
    boolean isWrapperFor(java.lang.Class<?> iface) throws java.sql.SQLException;
}
```

## unsupported包

sharding-jdbc-core/java.io.shardingjdbc.core.jdbc.unsupported下都是一些抽象类，对不支持的方法返回SQLFeatureNotSupportedException

![image](https://user-images.githubusercontent.com/7789698/39684009-8b8f4ae6-51eb-11e8-9e67-949bb46b54d7.png)

## adapter包

sharding-jdbc-core/java.io.shardingjdbc.core.jdbc.adapter里面都是一些适配类

### WrapperAdapter

实现wrapper接口unwrap和isWrapperFor。另外提供给子类recordMethodInvocation、replayMethodsInvocation、throwSQLExceptionIfNecessary

```java
public class WrapperAdapter implements Wrapper {
    
    private final Collection<JdbcMethodInvocation> jdbcMethodInvocations = new ArrayList<>();
    
    @SuppressWarnings("unchecked")
    @Override
    public final <T> T unwrap(final Class<T> iface) throws SQLException {
        if (isWrapperFor(iface)) {
            return (T) this;
        }
        throw new SQLException(String.format("[%s] cannot be unwrapped as [%s]", getClass().getName(), iface.getName()));
    }
    
    @Override
    public final boolean isWrapperFor(final Class<?> iface) {
        return iface.isInstance(this);
    }
    
    /**
     * 记录方法调用.
     * 
     * @param targetClass 目标类
     * @param methodName 方法名称
     * @param argumentTypes 参数类型
     * @param arguments 参数
     */
    public final void recordMethodInvocation(final Class<?> targetClass, final String methodName, final Class<?>[] argumentTypes, final Object[] arguments) {
        try {
            jdbcMethodInvocations.add(new JdbcMethodInvocation(targetClass.getMethod(methodName, argumentTypes), arguments));
        } catch (final NoSuchMethodException ex) {
            throw new ShardingJdbcException(ex);
        }
    }
    
    /**
     * 回放记录的方法调用.
     * 
     * @param target 目标对象
     */
    public final void replayMethodsInvocation(final Object target) {
        for (JdbcMethodInvocation each : jdbcMethodInvocations) {
            each.invoke(target);
        }
    }
    
    protected void throwSQLExceptionIfNecessary(final Collection<SQLException> exceptions) throws SQLException {
        if (exceptions.isEmpty()) {
            return;
        }
        SQLException ex = new SQLException();
        for (SQLException each : exceptions) {
            ex.setNextException(each);
        }
        throw ex;
    }
}
```

#### JdbcMethodInvocation

```java
@RequiredArgsConstructor
public class JdbcMethodInvocation {
    
    @Getter
    private final Method method;
    
    @Getter
    private final Object[] arguments;
    
    /**
     * Invoke JDBC method.
     * 
     * @param target target object
     */
    public void invoke(final Object target) {
        try {
            method.invoke(target, arguments);
        } catch (final IllegalAccessException | InvocationTargetException ex) {
            throw new ShardingJdbcException("Invoke jdbc method exception", ex);
        }
    }
}
```

### AbstractDataSourceAdapter

```java
public abstract class AbstractDataSourceAdapter extends AbstractUnsupportedOperationDataSource {
    @Getter
    private final DatabaseType databaseType;
    
    private PrintWriter logWriter = new PrintWriter(System.out);
    
    public AbstractDataSourceAdapter(final Collection<DataSource> dataSources) throws SQLException {
        databaseType = getDatabaseType(dataSources);
    }
    
    protected DatabaseType getDatabaseType(final Collection<DataSource> dataSources) throws SQLException {
        DatabaseType result = null;
        for (DataSource each : dataSources) {
            DatabaseType databaseType = getDatabaseType(each);
            Preconditions.checkState(null == result || result.equals(databaseType), String.format("Database type inconsistent with '%s' and '%s'", result, databaseType));
            result = databaseType;
        }
        return result;
    }
    
    private DatabaseType getDatabaseType(final DataSource dataSource) throws SQLException {
        if (dataSource instanceof AbstractDataSourceAdapter) {
            return ((AbstractDataSourceAdapter) dataSource).databaseType;
        }
        try (Connection connection = dataSource.getConnection()) {
            return DatabaseType.valueFrom(connection.getMetaData().getDatabaseProductName());
        }
    }
    
    @Override
    public final PrintWriter getLogWriter() {
        return logWriter;
    }
    
    @Override
    public final void setLogWriter(final PrintWriter out) {
        this.logWriter = out;
    }
    
    @Override
    public final Logger getParentLogger() {
        return Logger.getLogger(Logger.GLOBAL_LOGGER_NAME);
    }
    
    @Override
    public final Connection getConnection(final String username, final String password) throws SQLException {
        return getConnection();
    }
}
```



### AbstractConnectionAdapter

```java
public abstract class AbstractConnectionAdapter extends AbstractUnsupportedOperationConnection {
    //缓存数据源
    private final Map<String, Connection> cachedConnections = new HashMap<>();
    
    private boolean autoCommit = true;
    
    private boolean readOnly = true;
    
    private boolean closed;
    
    private int transactionIsolation = TRANSACTION_READ_UNCOMMITTED;
    
    /**
     * Get database connection.
     *
     * @param dataSourceName data source name
     * @return database connection
     * @throws SQLException SQL exception
     */
    public final Connection getConnection(final String dataSourceName) throws SQLException {
        if (cachedConnections.containsKey(dataSourceName)) {
            return cachedConnections.get(dataSourceName);
        }
        //从子类获取数据源
        DataSource dataSource = getDataSourceMap().get(dataSourceName);
        Preconditions.checkState(null != dataSource, "Missing the data source name: '%s'", dataSourceName);
        //获取连接
        Connection result = dataSource.getConnection();
        cachedConnections.put(dataSourceName, result);
        //回放记录的方法调用
        replayMethodsInvocation(result);
        return result;
    }
    
    protected abstract Map<String, DataSource> getDataSourceMap();
    
    protected void removeCache(final Connection connection) {
        cachedConnections.values().remove(connection);
    }
    
    @Override
    public final boolean getAutoCommit() {
        return autoCommit;
    }
    
    @Override
    public final void setAutoCommit(final boolean autoCommit) throws SQLException {
        this.autoCommit = autoCommit;
        //记录方法调用
        recordMethodInvocation(Connection.class, "setAutoCommit", new Class[] {boolean.class}, new Object[] {autoCommit});
        for (Connection each : cachedConnections.values()) {
            each.setAutoCommit(autoCommit);
        }
    }
    
    @Override
    public final void commit() throws SQLException {
        Collection<SQLException> exceptions = new LinkedList<>();
        for (Connection each : cachedConnections.values()) {
            try {
                each.commit();
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        throwSQLExceptionIfNecessary(exceptions);
    }
    
    @Override
    public final void rollback() throws SQLException {
        Collection<SQLException> exceptions = new LinkedList<>();
        for (Connection each : cachedConnections.values()) {
            try {
                each.rollback();
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        throwSQLExceptionIfNecessary(exceptions);
    }
    
    @Override
    public void close() throws SQLException {
        closed = true;
        HintManagerHolder.clear();
        MasterVisitedManager.clear();
        Collection<SQLException> exceptions = new LinkedList<>();
        for (Connection each : cachedConnections.values()) {
            try {
                each.close();
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        throwSQLExceptionIfNecessary(exceptions);
    }
    
    @Override
    public final boolean isClosed() {
        return closed;
    }
    
    @Override
    public final boolean isReadOnly() {
        return readOnly;
    }
    
    @Override
    public final void setReadOnly(final boolean readOnly) throws SQLException {
        this.readOnly = readOnly;
        recordMethodInvocation(Connection.class, "setReadOnly", new Class[] {boolean.class}, new Object[] {readOnly});
        for (Connection each : cachedConnections.values()) {
            each.setReadOnly(readOnly);
        }
    }
    
    @Override
    public final int getTransactionIsolation() {
        return transactionIsolation;
    }
    
    @Override
    public final void setTransactionIsolation(final int level) throws SQLException {
        transactionIsolation = level;
        //记录
        recordMethodInvocation(Connection.class, "setTransactionIsolation", new Class[] {int.class}, new Object[] {level});
        for (Connection each : cachedConnections.values()) {
            each.setTransactionIsolation(level);
        }
    }
    
    // ------- Consist with MySQL driver implementation -------
    
    @Override
    public SQLWarning getWarnings() {
        return null;
    }
    
    @Override
    public void clearWarnings() {
    }
    
    @Override
    public final int getHoldability() {
        return ResultSet.CLOSE_CURSORS_AT_COMMIT;
    }
    
    @Override
    public final void setHoldability(final int holdability) {
    }
}
```

### AbstractStatementAdapter

```java
@RequiredArgsConstructor
public abstract class AbstractStatementAdapter extends AbstractUnsupportedOperationStatement {
    
    private final Class<? extends Statement> targetClass;
    
    private boolean closed;
    
    private boolean poolable;
    
    private int fetchSize;
    
    @Override
    public final void close() throws SQLException {
        closed = true;
        Collection<SQLException> exceptions = new LinkedList<>();
        for (Statement each : getRoutedStatements()) {
            try {
                each.close();
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        getRoutedStatements().clear();
        throwSQLExceptionIfNecessary(exceptions);
    }
    
    @Override
    public final boolean isClosed() {
        return closed;
    }
    
    @Override
    public final boolean isPoolable() {
        return poolable;
    }
    
    @Override
    public final void setPoolable(final boolean poolable) throws SQLException {
        this.poolable = poolable;
        //记录调用
        recordMethodInvocation(targetClass, "setPoolable", new Class[] {boolean.class}, new Object[] {poolable});
        for (Statement each : getRoutedStatements()) {
            each.setPoolable(poolable);
        }
    }
    
    @Override
    public final int getFetchSize() {
        return fetchSize;
    }
    
    @Override
    public final void setFetchSize(final int rows) throws SQLException {
        this.fetchSize = rows;
        recordMethodInvocation(targetClass, "setFetchSize", new Class[] {int.class}, new Object[] {rows});
        for (Statement each : getRoutedStatements()) {
            each.setFetchSize(rows);
        }
    }
    
    @Override
    public final void setEscapeProcessing(final boolean enable) throws SQLException {
        recordMethodInvocation(targetClass, "setEscapeProcessing", new Class[] {boolean.class}, new Object[] {enable});
        for (Statement each : getRoutedStatements()) {
            each.setEscapeProcessing(enable);
        }
    }
    
    @Override
    public final void cancel() throws SQLException {
        for (Statement each : getRoutedStatements()) {
            each.cancel();
        }
    }
    
    @Override
    public final int getUpdateCount() throws SQLException {
        long result = 0;
        boolean hasResult = false;
        for (Statement each : getRoutedStatements()) {
            if (each.getUpdateCount() > -1) {
                hasResult = true;
            }
            //累加起来
            result += each.getUpdateCount();
        }
        if (result > Integer.MAX_VALUE) {
            result = Integer.MAX_VALUE;
        }
        return hasResult ? Long.valueOf(result).intValue() : -1;
    }
    
    @Override
    public SQLWarning getWarnings() {
        return null;
    }
    
    @Override
    public void clearWarnings() {
    }
    
    @Override
    public final boolean getMoreResults() {
        return false;
    }
    
    @Override
    public final boolean getMoreResults(final int current) {
        return false;
    }
    
    @Override
    public final int getMaxFieldSize() throws SQLException {
        return getRoutedStatements().isEmpty() ? 0 : getRoutedStatements().iterator().next().getMaxFieldSize();
    }
    
    @Override
    public final void setMaxFieldSize(final int max) throws SQLException {
        recordMethodInvocation(targetClass, "setMaxFieldSize", new Class[] {int.class}, new Object[] {max});
        for (Statement each : getRoutedStatements()) {
            each.setMaxFieldSize(max);
        }
    }
    
    // TODO Confirm MaxRows for multiple databases is need special handle. eg: 10 statements maybe MaxRows / 10
    @Override
    public final int getMaxRows() throws SQLException {
        return getRoutedStatements().isEmpty() ? -1 : getRoutedStatements().iterator().next().getMaxRows();
    }
    
    @Override
    public final void setMaxRows(final int max) throws SQLException {
        recordMethodInvocation(targetClass, "setMaxRows", new Class[] {int.class}, new Object[] {max});
        for (Statement each : getRoutedStatements()) {
            each.setMaxRows(max);
        }
    }
    
    @Override
    public final int getQueryTimeout() throws SQLException {
        return getRoutedStatements().isEmpty() ? 0 : getRoutedStatements().iterator().next().getQueryTimeout();
    }
    
    @Override
    public final void setQueryTimeout(final int seconds) throws SQLException {
        recordMethodInvocation(targetClass, "setQueryTimeout", new Class[] {int.class}, new Object[] {seconds});
        for (Statement each : getRoutedStatements()) {
            each.setQueryTimeout(seconds);
        }
    }
    //抽象，子类实现，路由语句对象集合
    protected abstract Collection<? extends Statement> getRoutedStatements();
}
```

### AbstractShardingPreparedStatementAdapter

```java
public abstract class AbstractShardingPreparedStatementAdapter extends AbstractUnsupportedOperationPreparedStatement {
    //记录的设置参数方法数组
    private final List<SetParameterMethodInvocation> setParameterMethodInvocations = new LinkedList<>();
    
    @Getter
    private final List<Object> parameters = new ArrayList<>();
    
    //记录占位符参数
    private void setParameter(final int parameterIndex, final Object value) {
        if (parameters.size() == parameterIndex - 1) {
            parameters.add(value);
            return;
        }
        for (int i = parameters.size(); i <= parameterIndex - 1; i++) {
            parameters.add(null);
        }
        parameters.set(parameterIndex - 1, value);
    }
    //记录设置参数方法调用
    private void recordSetParameter(final String methodName, final Class[] argumentTypes, final Object... arguments) {
        try {
            setParameterMethodInvocations.add(new SetParameterMethodInvocation(PreparedStatement.class.getMethod(methodName, argumentTypes), arguments, arguments[1]));
        } catch (final NoSuchMethodException ex) {
            throw new ShardingJdbcException(ex);
        }
    }
    //回放记录的设置参数方法调用
    protected void replaySetParameter(final PreparedStatement preparedStatement, final List<Object> parameters) {
        setParameterMethodInvocations.clear();
        addParameters(parameters);
        for (SetParameterMethodInvocation each : setParameterMethodInvocations) {
            each.invoke(preparedStatement);
        }
    }

    private void addParameters(final List<Object> parameters) {
        for (int i = 0; i < parameters.size(); i++) {
            recordSetParameter("setObject", new Class[]{int.class, Object.class}, i + 1, parameters.get(i));
        }
    }
    
    @Override
    public final void clearParameters() {
        parameters.clear();
        setParameterMethodInvocations.clear();
    }
}
```



```java
public final class SetParameterMethodInvocation extends JdbcMethodInvocation {
    
    @Getter
    private final int index;
    
    @Getter
    private final Object value;
    
    public SetParameterMethodInvocation(final Method method, final Object[] arguments, final Object value) {
        super(method, arguments);
        this.index = (int) arguments[0];
        this.value = value;
    }
    
    /**
     * Set argument.
     * 
     * @param value argument value
     */
    public void changeValueArgument(final Object value) {
        getArguments()[1] = value;
    }
}
```



### AbstractResultSetAdapter

```java
@Slf4j
public abstract class AbstractResultSetAdapter extends AbstractUnsupportedOperationResultSet {
    @Getter
    private final List<ResultSet> resultSets;
    
    @Getter
    private final Statement statement;
    
    private boolean closed;
    
    public AbstractResultSetAdapter(final List<ResultSet> resultSets, final Statement statement) {
        Preconditions.checkArgument(!resultSets.isEmpty());
        this.resultSets = resultSets;
        this.statement = statement;
    }
    
    @Override
    public final ResultSetMetaData getMetaData() throws SQLException {
        return resultSets.get(0).getMetaData();
    }
    
    @Override
    public int findColumn(final String columnLabel) throws SQLException {
        return resultSets.get(0).findColumn(columnLabel);
    }
    
    @Override
    public final void close() throws SQLException {
        closed = true;
        Collection<SQLException> exceptions = new LinkedList<>();
        for (ResultSet each : resultSets) {
            try {
                each.close();
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        throwSQLExceptionIfNecessary(exceptions);
    }
    
    @Override
    public final boolean isClosed() {
        return closed;
    }
    
    @Override
    public final void setFetchDirection(final int direction) throws SQLException {
        Collection<SQLException> exceptions = new LinkedList<>();
        for (ResultSet each : resultSets) {
            try {
                each.setFetchDirection(direction);
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        throwSQLExceptionIfNecessary(exceptions);
    }
    
    @Override
    public final int getFetchDirection() throws SQLException {
        return resultSets.get(0).getFetchDirection();
    }
    
    @Override
    public final void setFetchSize(final int rows) throws SQLException {
        Collection<SQLException> exceptions = new LinkedList<>();
        for (ResultSet each : resultSets) {
            try {
                each.setFetchSize(rows);
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        throwSQLExceptionIfNecessary(exceptions);
    }
    
    @Override
    public final int getFetchSize() throws SQLException {
        return resultSets.get(0).getFetchSize();
    }
    
    @Override
    public final int getType() throws SQLException {
        return resultSets.get(0).getType();
    }
    
    @Override
    public final int getConcurrency() throws SQLException {
        return resultSets.get(0).getConcurrency();
    }
    
    @Override
    public final SQLWarning getWarnings() throws SQLException {
        return resultSets.get(0).getWarnings();
    }
    
    @Override
    public final void clearWarnings() throws SQLException {
        Collection<SQLException> exceptions = new LinkedList<>();
        for (ResultSet each : resultSets) {
            try {
                each.clearWarnings();
            } catch (final SQLException ex) {
                exceptions.add(ex);
            }
        }
        throwSQLExceptionIfNecessary(exceptions);
    }
}
```

