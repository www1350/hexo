---
title: sharding-jdbc源码解析-读写分离（二）
tags:
  - sharding-jdbc
categories:
  - 中间件
  - 数据库
abbrlink: 5cbcca12
date: 2018-05-07 19:20:15
---

调用流程

![sequencediagram1](https://user-images.githubusercontent.com/7789698/39745015-a9f08048-52d8-11e8-8a74-9d504e46f3a8.jpg)

执行sql的时候，sql根据词法分析出sql的类型从而判断走主库还是从库

### MasterSlaveDataSource

从`MasterSlaveDataSourceFactory.createDataSource(createDataSourceMap(), masterSlaveRuleConfig)`我们可以找到读写分离实现最重要的在于MasterSlaveDataSource这个数据源的实现。

```java
@Getter
public class MasterSlaveDataSource extends AbstractDataSourceAdapter {
    //数据源映射
    private Map<String, DataSource> dataSourceMap;
    
    private MasterSlaveRule masterSlaveRule;
    
    public MasterSlaveDataSource(final Map<String, DataSource> dataSourceMap, final MasterSlaveRuleConfiguration masterSlaveRuleConfig, final Map<String, Object> configMap) throws SQLException {
        super(getAllDataSources(dataSourceMap, masterSlaveRuleConfig.getMasterDataSourceName(), masterSlaveRuleConfig.getSlaveDataSourceNames()));
        this.dataSourceMap = dataSourceMap;
        this.masterSlaveRule = new MasterSlaveRule(masterSlaveRuleConfig);
        if (!configMap.isEmpty()) {
            ConfigMapContext.getInstance().getMasterSlaveConfig().putAll(configMap);
        }
    }
    
    private static Collection<DataSource> getAllDataSources(final Map<String, DataSource> dataSourceMap, final String masterDataSourceName, final Collection<String> slaveDataSourceNames) {
        Collection<DataSource> result = new LinkedList<>();
        result.add(dataSourceMap.get(masterDataSourceName));
        for (String each : slaveDataSourceNames) {
            result.add(dataSourceMap.get(each));
        }
        return result;
    }
    
    /**
     * Get map of all actual data source name and all actual data sources.
     *
     * @return map of all actual data source name and all actual data sources
     */
    public Map<String, DataSource> getAllDataSources() {
        Map<String, DataSource> result = new HashMap<>(masterSlaveRule.getSlaveDataSourceNames().size() + 1, 1);
        result.put(masterSlaveRule.getMasterDataSourceName(), dataSourceMap.get(masterSlaveRule.getMasterDataSourceName()));
        for (String each : masterSlaveRule.getSlaveDataSourceNames()) {
            result.put(each, dataSourceMap.get(each));
        }
        return result;
    }
    
    /**
     * Renew master-slave data source.
     *
     * @param dataSourceMap data source map
     * @param masterSlaveRuleConfig new master-slave rule configuration
     */
    public void renew(final Map<String, DataSource> dataSourceMap, final MasterSlaveRuleConfiguration masterSlaveRuleConfig) {
        this.dataSourceMap = dataSourceMap;
        this.masterSlaveRule = new MasterSlaveRule(masterSlaveRuleConfig);
    }
    
    @Override
    public MasterSlaveConnection getConnection() {
        return new MasterSlaveConnection(this);
    }
}
```

### MasterSlaveConnection

基本上没啥好看的直接都是跟后面两个相关

### MasterSlaveStatement

```java
@Getter
public final class MasterSlaveStatement extends AbstractStatementAdapter {
    
    private final MasterSlaveConnection connection;
    
    private final MasterSlaveRouter masterSlaveRouter;
    
    private final int resultSetType;
    
    private final int resultSetConcurrency;
    
    private final int resultSetHoldability;
    
    private final Collection<Statement> routedStatements = new LinkedList<>();
    
    public MasterSlaveStatement(final MasterSlaveConnection connection) {
        this(connection, ResultSet.TYPE_FORWARD_ONLY, ResultSet.CONCUR_READ_ONLY, ResultSet.HOLD_CURSORS_OVER_COMMIT);
    }
    
    public MasterSlaveStatement(final MasterSlaveConnection connection, final int resultSetType, final int resultSetConcurrency) {
        this(connection, resultSetType, resultSetConcurrency, ResultSet.HOLD_CURSORS_OVER_COMMIT);
    }
    
    public MasterSlaveStatement(final MasterSlaveConnection connection, final int resultSetType, final int resultSetConcurrency, final int resultSetHoldability) {
        super(Statement.class);
        this.connection = connection;
        masterSlaveRouter = new MasterSlaveRouter(connection.getMasterSlaveDataSource().getMasterSlaveRule());
        this.resultSetType = resultSetType;
        this.resultSetConcurrency = resultSetConcurrency;
        this.resultSetHoldability = resultSetHoldability;
    }
    
    @Override
    public ResultSet executeQuery(final String sql) throws SQLException {
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        Collection<String> dataSourceNames = masterSlaveRouter.route(sqlStatement.getType());
        Preconditions.checkState(1 == dataSourceNames.size(), "Cannot support executeQuery for DML or DDL");
        Statement statement = connection.getConnection(dataSourceNames.iterator().next()).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
        routedStatements.add(statement);
        return statement.executeQuery(sql);
    }
    
    @Override
    public int executeUpdate(final String sql) throws SQLException {
        int result = 0;
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result += statement.executeUpdate(sql);
        }
        return result;
    }
    
    @Override
    public int executeUpdate(final String sql, final int autoGeneratedKeys) throws SQLException {
        int result = 0;
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result += statement.executeUpdate(sql, autoGeneratedKeys);
        }
        return result;
    }
    
    @Override
    public int executeUpdate(final String sql, final int[] columnIndexes) throws SQLException {
        int result = 0;
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result += statement.executeUpdate(sql, columnIndexes);
        }
        return result;
    }
    
    @Override
    public int executeUpdate(final String sql, final String[] columnNames) throws SQLException {
        int result = 0;
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result += statement.executeUpdate(sql, columnNames);
        }
        return result;
    }
    
    @Override
    public boolean execute(final String sql) throws SQLException {
        boolean result = false;
        //根据sql语句判断出是哪种SQLStatement
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result = statement.execute(sql);
        }
        return result;
    }
    
    @Override
    public boolean execute(final String sql, final int autoGeneratedKeys) throws SQLException {
        boolean result = false;
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result = statement.execute(sql, autoGeneratedKeys);
        }
        return result;
    }
    
    @Override
    public boolean execute(final String sql, final int[] columnIndexes) throws SQLException {
        boolean result = false;
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result = statement.execute(sql, columnIndexes);
        }
        return result;
    }
    
    @Override
    public boolean execute(final String sql, final String[] columnNames) throws SQLException {
        boolean result = false;
        SQLStatement sqlStatement = new SQLJudgeEngine(sql).judge();
        for (String each : masterSlaveRouter.route(sqlStatement.getType())) {
            Statement statement = connection.getConnection(each).createStatement(resultSetType, resultSetConcurrency, resultSetHoldability);
            routedStatements.add(statement);
            result = statement.execute(sql, columnNames);
        }
        return result;
    }
    
    @Override
    public ResultSet getGeneratedKeys() throws SQLException {
        Preconditions.checkState(1 == routedStatements.size());
        return routedStatements.iterator().next().getGeneratedKeys();
    }
    
    @Override
    public ResultSet getResultSet() throws SQLException {
        Preconditions.checkState(1 == routedStatements.size());
        return routedStatements.iterator().next().getResultSet();
    }
}
```

### MasterSlaveRouter（4月份新加的）

根据sql类型判断主从

```java
@RequiredArgsConstructor
public final class MasterSlaveRouter {
    
    private final MasterSlaveRule masterSlaveRule;
    
    /**
     * Route Master slave.
     * 
     * @param sqlType SQL type
     * @return data source name
     */
    // TODO for multiple masters may return more than one data source
    public Collection<String> route(final SQLType sqlType) {
        if (isMasterRoute(sqlType)) {
            MasterVisitedManager.setMasterVisited();
            return Collections.singletonList(masterSlaveRule.getMasterDataSourceName());
        } else {
            return Collections.singletonList(masterSlaveRule.getLoadBalanceAlgorithm().getDataSource(
                    masterSlaveRule.getName(), masterSlaveRule.getMasterDataSourceName(), new ArrayList<>(masterSlaveRule.getSlaveDataSourceNames())));
        }
    }
    
    private boolean isMasterRoute(final SQLType sqlType) {
        //1.非SELECT语句2.判断过放到ThreadLocal里3.通过Hint管理器设置强制masterRouteOnly
        return SQLType.DQL != sqlType || MasterVisitedManager.isMasterVisited() || HintManagerHolder.isMasterRouteOnly();
    }
}
```

### SQLJudgeEngine

通过词法分析分析出sql的类型

```java
@RequiredArgsConstructor
public final class SQLJudgeEngine {
    
    private final String sql;
    
    /**
     * judge SQL Type only.
     *
     * @return SQL statement
     */
    public SQLStatement judge() {
        LexerEngine lexerEngine = LexerEngineFactory.newInstance(DatabaseType.MySQL, sql);
        //获取token
        lexerEngine.nextToken();
        while (true) {
            //获取token的type
            TokenType tokenType = lexerEngine.getCurrentToken().getType();
            if (tokenType instanceof Keyword) {
                //select
                if (isDQL(tokenType)) {
                    //SelectStatement
                    return getDQLStatement();
                }
                //INSERT、UPDATE、DELETE
                if (isDML(tokenType)) {
                    //InsertStatement 或者DMLStatement
                    return getDMLStatement(tokenType);
                }
                //CREATE、ALTER、DROP、TRUNCATE
                if (isDDL(tokenType)) {
                    //DDLStatement
                    return getDDLStatement();
                }
                //SET、COMMIT、ROLLBACK、SAVEPOINT、BEGIN
                if (isTCL(tokenType)) {
                    //TCLStatement
                    return getTCLStatement();
                }
                //USE、DESC、DESCRIBE、SHOW
                if (isDAL(tokenType)) {
                    //UseStatement或DescribeStatement或ShowxxxStatement
                    return getDALStatement(tokenType, lexerEngine);
                }
            }
            if (tokenType instanceof Assist && Assist.END == tokenType) {
                throw new SQLParsingException("Unsupported SQL statement: [%s]", sql);
            }
            lexerEngine.nextToken();
        }
    }
    
    private boolean isDAL(final TokenType tokenType) {
        return DefaultKeyword.USE == tokenType || DefaultKeyword.DESC == tokenType || MySQLKeyword.DESCRIBE == tokenType || MySQLKeyword.SHOW == tokenType;
    }
    
  
    private SQLStatement getDALStatement(final TokenType tokenType, final LexerEngine lexerEngine) {
        if (DefaultKeyword.USE == tokenType) {
            return new UseStatement();
        }
        if (DefaultKeyword.DESC == tokenType || MySQLKeyword.DESCRIBE == tokenType) {
            return new DescribeStatement();
        }
        return getShowStatement(lexerEngine);
    }
    
    private SQLStatement getShowStatement(final LexerEngine lexerEngine) {
        lexerEngine.nextToken();
        if (MySQLKeyword.DATABASES == lexerEngine.getCurrentToken().getType()) {
            return new ShowDatabasesStatement();
        }
        if (MySQLKeyword.TABLES == lexerEngine.getCurrentToken().getType()) {
            return new ShowTablesStatement();
        }
        if (MySQLKeyword.COLUMNS == lexerEngine.getCurrentToken().getType()) {
            return new ShowColumnsStatement();
        }
        return new ShowOtherStatement();
    }
}
```

### LexerEngine

LexerEngineFactory会根据数据库来选择适合的分词器

mysql对应MySQLLexer

```java
@NoArgsConstructor(access = AccessLevel.PRIVATE)
public final class LexerEngineFactory {
    
    /**
     * Create lexical analysis engine instance.
     * 
     * @param dbType database type
     * @param sql SQL
     * @return lexical analysis engine instance
     */
    public static LexerEngine newInstance(final DatabaseType dbType, final String sql) {
        switch (dbType) {
            case H2:
            case MySQL:
                return new LexerEngine(new MySQLLexer(sql));
            case Oracle:
                return new LexerEngine(new OracleLexer(sql));
            case SQLServer:
                return new LexerEngine(new SQLServerLexer(sql));
            case PostgreSQL:
                return new LexerEngine(new PostgreSQLLexer(sql));
            default:
                throw new UnsupportedOperationException(String.format("Cannot support database [%s].", dbType));
        }
    }
}
```

关于词法分析，下一章单独分析