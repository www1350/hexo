---
title: sharding-jdbc源码解析-分库分表（四）
tags:
  - sharding-jdbc
categories:
  - 中间件
  - 数据库
abbrlink: 38641c2c
date: 2018-05-17 14:43:01
---

## 分库分表配置

TableRuleConfiguration##build—>TableRule

### TableRuleConfiguration

```java
//逻辑表,表名
//例：订单数据根据主键尾数拆分为10张表,分别是t_order_0到t_order_9，他们的逻辑表名为t_order。
private String logicTable;
//
//如：ds_jdbc.t_order_${0..9}
//
private String actualDataNodes;
//数据库分库策略
private ShardingStrategyConfiguration databaseShardingStrategyConfig;
//分表策略
private ShardingStrategyConfiguration tableShardingStrategyConfig;
//自增列
private String keyGeneratorColumnName;
//id生成器类名
private String keyGeneratorClass;
```

- actualDataNodes

```java
//使用groovy表达式解析出真实数据库节点
List<String> actualDataNodes = new InlineExpressionParser(this.actualDataNodes).evaluate();
```

#### InlineExpressionParser

内联表达式解析器

inlineExpression是它唯一的参数和构造参数。如ds_jdbc.t_order_${[0, 9]}这样的表达式

```java
public List<String> evaluate() {
    if (null == inlineExpression) {
        return Collections.emptyList();
    }
    return flatten(evaluate(split()));
}
// ds_jdbc.t_order_${[0,9]} ,ds_jdbc.t_order_${1..8}  
//result: ["ds_jdbc.t_order_${[0,9]}", "ds_jdbc.t_order_${1..8}"] 
private List<String> split() {
     List<String> result = new ArrayList<>();
     StringBuilder segment = new StringBuilder();
     int bracketsDepth = 0;
     for (int i = 0; i < inlineExpression.length(); i++) {
        char each = inlineExpression.charAt(i);
        switch (each) {
            // ','
            case SPLITTER:
               if (bracketsDepth > 0) {
                    segment.append(each);
                } else {
                    result.add(segment.toString().trim());
                    segment.setLength(0);
                }
                break;
            case '$':
               if ('{' == inlineExpression.charAt(i + 1)) {
                    bracketsDepth++;
               }
               segment.append(each);
               break;
            case '}':
               if (bracketsDepth > 0) {
                   bracketsDepth--;
               }
               segment.append(each);
               break;
            default:
               segment.append(each);
               break;
         }
      }
      if (segment.length() > 0) {
         result.add(segment.toString().trim());
      }
      return result;
}

//Groovy表达式
//inlineExpressions: "ds_jdbc.t_order_${[0,9]} ,ds_jdbc.t_order_${1..8}"    
//result : [ds_jdbc.t_order_[0, 9], ds_jdbc.t_order_[1, 2, 3, 4, 5, 6, 7, 8]]  -> List<GString>
private List<Object> evaluate(final List<String> inlineExpressions) {
        List<Object> result = new ArrayList<>(inlineExpressions.size());
        GroovyShell shell = new GroovyShell();
        for (String each : inlineExpressions) {
            StringBuilder expression = new StringBuilder(each);
            if (!each.startsWith("\"")) {
                expression.insert(0, "\"");
            }
            if (!each.endsWith("\"")) {
                expression.append("\"");
            }
            result.add(shell.evaluate(expression.toString()));
        }
        return result;
    }

//segments: "[ds_jdbc.t_order_[0, 9], ds_jdbc.t_order_[1, 2, 3, 4, 5, 6, 7, 8]]"  -> List<GString>
//result : "ds_jdbc.t_order_0","ds_jdbc.t_order_1","ds_jdbc.t_order_2","ds_jdbc.t_order_3","ds_jdbc.t_order_4","ds_jdbc.t_order_5","ds_jdbc.t_order_6","ds_jdbc.t_order_7","ds_jdbc.t_order_8","ds_jdbc.t_order_9"
private List<String> flatten(final List<Object> segments) {
        List<String> result = new ArrayList<>();
        for (Object each : segments) {
            if (each instanceof GString) {
                result.addAll(assemblyCartesianSegments((GString) each));
            } else {
                result.add(each.toString());
            }
        }
        return result;
    }
```

### TableRule

```java
private final String logicTable;
//静态分库分表数据单元
//"ds_jdbc.t_order_0","ds_jdbc.t_order_1","ds_jdbc.t_order_2","ds_jdbc.t_order_3","ds_jdbc.t_order_4","ds_jdbc.t_order_5","ds_jdbc.t_order_6","ds_jdbc.t_order_7","ds_jdbc.t_order_8","ds_jdbc.t_order_9"
private final List<DataNode> actualDataNodes;

private final ShardingStrategy databaseShardingStrategy;

private final ShardingStrategy tableShardingStrategy;

private final String generateKeyColumn;

private final KeyGenerator keyGenerator;
```

#### ShardingStrategyConfiguration

- StandardShardingStrategyConfiguration 标准分片策略

  ```java
  //分片列名
  private final String shardingColumn;
  //用于处理=和IN的分片
  private final String preciseAlgorithmClassName;
  //处理BETWEEN AND分片
  private final String rangeAlgorithmClassName;
  ```

- ComplexShardingStrategyConfiguration


```java
private final String shardingColumns;
//
private final String algorithmClassName;
```

- InlineShardingStrategyConfiguration

```java
private final String shardingColumn;
//分片表达式，
//如：t_user_${u_id % 8} 表示t_user表按照u_id按8取模分成8个表，表名称为t_user_0到t_user_7。
private final String algorithmExpression;
```

- HintShardingStrategyConfiguration

```java
private final String algorithmClassName;
```

- NoneShardingStrategyConfiguration

#### ShardingStrategy

- StandardShardingStrategy

```java
private final String shardingColumn;
//用于处理=和IN的算法
private final PreciseShardingAlgorithm preciseShardingAlgorithm;
//这个是可选的，处理BETWEEN AND算法
private final Optional<RangeShardingAlgorithm> rangeShardingAlgorithm;
```

- ComplexShardingStrategy  复合分片策略

```java
@Getter
private final Collection<String> shardingColumns;
//复合分片算法
private final ComplexKeysShardingAlgorithm shardingAlgorithm;
```

- InlineShardingStrategy  行表达式分片策略

```java
private final String shardingColumn;
//表达式转化成groovy闭包
private final Closure<?> closure;
```

- HintShardingStrategy 通过Hint而非SQL解析的方式分片的策略

```java
@Getter
private final Collection<String> shardingColumns;

private final HintShardingAlgorithm shardingAlgorithm;
```

- NoneShardingStrategy 不分片的策略

```java
private final Collection<String> shardingColumns = Collections.emptyList();
```

### ShardingRuleConfiguration

```java

private String defaultDataSourceName;
    
private Collection<TableRuleConfiguration> tableRuleConfigs = new LinkedList<>();
    
private Collection<String> bindingTableGroups = new LinkedList<>();
    
private ShardingStrategyConfiguration defaultDatabaseShardingStrategyConfig;
    
private ShardingStrategyConfiguration defaultTableShardingStrategyConfig;
    
private String defaultKeyGeneratorClass;
    
private Collection<MasterSlaveRuleConfiguration> masterSlaveRuleConfigs = 
    new LinkedList<>();
```

### ShardingRule

```java
private final Map<String, DataSource> dataSourceMap;

private final String defaultDataSourceName;
//表规则配置对象集合
private final Collection<TableRule> tableRules;

private final Collection<BindingTableRule> bindingTableRules = new LinkedList<>();
//分库策略
private final ShardingStrategy defaultDatabaseShardingStrategy;
//分表策略
private final ShardingStrategy defaultTableShardingStrategy;

private final KeyGenerator defaultKeyGenerator;
```

## 重要的接口

```java
//分片值规则
public interface ShardingValue {
    //获取逻辑表名
    String getLogicTableName();
    
    //获取列名
    String getColumnName();
}
```



```java
//表映射单元
@RequiredArgsConstructor
@Getter
@EqualsAndHashCode
@ToString
public final class TableUnit {
    //真实数据源名
    private final String dataSourceName;
    //逻辑表名
    private final String logicTableName;
    //真实表名
    private final String actualTableName;
}
```

```java
//SQL执行单元
@RequiredArgsConstructor
@Getter
@EqualsAndHashCode
@ToString
public final class SQLExecutionUnit {
    //真实数据源
    private final String dataSource;
    //真实sql
    private final String sql;
}
```

```java
//SQL路由结果
@RequiredArgsConstructor
@Getter
public final class SQLRouteResult {
    
    private final SQLStatement sqlStatement;
    
    private final Set<SQLExecutionUnit> executionUnits = new LinkedHashSet<>();
    
    private final List<Number> generatedKeys = new LinkedList<>();
}
```

## 执行流程

![121212](https://user-images.githubusercontent.com/7789698/40271273-97779b9c-5bcd-11e8-8a63-f7736a1d752b.png)

## 路由

### SQLRouter

SQL 路由器接口

```java
public interface SQLRouter {
    //解析sql
    SQLStatement parse(String logicSQL, int parametersSize);
    
    //路由sql
    SQLRouteResult route(String logicSQL, List<Object> parameters, SQLStatement sqlStatement);
}
```

有两种实现：

- DatabaseHintSQLRouter：通过提示且仅路由至数据库的SQL路由器

```java
@Override
public SQLStatement parse(final String logicSQL, final int parametersSize) {
    //通过词法分析分析出sql的类型
    return new SQLJudgeEngine(logicSQL).judge();
}
```

```java
@Override
// TODO insert SQL need parse gen key
public SQLRouteResult route(final String logicSQL, final List<Object> parameters, final SQLStatement sqlStatement) {
    //SQL路由结果
    SQLRouteResult result = new SQLRouteResult(sqlStatement);
    //路由
    RoutingResult routingResult = new DatabaseHintRoutingEngine(shardingRule.getDataSourceMap(), (HintShardingStrategy) shardingRule.getDefaultDatabaseShardingStrategy()).route();
    for (TableUnit each : routingResult.getTableUnits().getTableUnits()) {
        result.getExecutionUnits().add(new SQLExecutionUnit(each.getDataSourceName(), logicSQL));
    }
    //打印sql
    if (showSQL) {
        SQLLogger.logSQL(logicSQL, sqlStatement, result.getExecutionUnits(), parameters);
    }
    return result;
}
```

DatabaseHintRoutingEngine 数据库提示路由引擎

```java
private final Map<String, DataSource> dataSourceMap;
//hint路由策略
private final HintShardingStrategy databaseShardingStrategy;

@Override
public RoutingResult route() {
    //获取分片键值
    Optional<ShardingValue> shardingValue = HintManagerHolder.getDatabaseShardingValue(new ShardingKey(HintManagerHolder.DB_TABLE_NAME, HintManagerHolder.DB_COLUMN_NAME));
    Preconditions.checkState(shardingValue.isPresent());
    log.debug("Before database sharding only db:{} sharding values: {}", dataSourceMap.keySet(), shardingValue.get());
    //路由
    Collection<String> routingDataSources;
    routingDataSources = databaseShardingStrategy.doSharding(dataSourceMap.keySet(), Collections.singletonList(shardingValue.get()));
    Preconditions.checkState(!routingDataSources.isEmpty(), "no database route info");
    log.debug("After database sharding only result: {}", routingDataSources);
    RoutingResult result = new RoutingResult();
    //TableUnit
    for (String each : routingDataSources) {
        result.getTableUnits().getTableUnits().add(new TableUnit(each, "", ""));
    }
    return result;
}
```

HintShardingStrategy  hint路由策略

```java
@Override
public Collection<String> doSharding(final Collection<String> availableTargetNames, final Collection<ShardingValue> shardingValues) {
    //通过HintShardingAlgorithm接口路由
    Collection<String> shardingResult = shardingAlgorithm.doSharding(availableTargetNames, shardingValues.iterator().next());
    Collection<String> result = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
    result.addAll(shardingResult);
    return result;
}
```

- ParsingSQLRouter：需要解析的SQL路由器

  解析

```java
@Override
public SQLStatement parse(final String logicSQL, final int parametersSize) {
    SQLParsingEngine parsingEngine = new SQLParsingEngine(databaseType, logicSQL, shardingRule);
    SQLStatement result = parsingEngine.parse();
    if (result instanceof InsertStatement) {
        ((InsertStatement) result).appendGenerateKeyToken(shardingRule, parametersSize);
    }
    return result;
}
```

SQLParsingEngine 是sql解析引擎，下一章讲到。经过sql解析引擎解析后得到SQLStatement，如果是InsertStatement会改写sql处理 GenerateKeyToken。关于sql改写后面会讲到。

### InsertStatement

被sql解析引擎解析后，最重要的是返回了SQLStatement，拿insert举例就是InsertStatement

```java
//sql类型，比如DML
private final SQLType type;
    
private final Tables tables = new Tables();
//条件<列名，值> 
private final Conditions conditions = new Conditions();
//所有token，比如TableToken（t_order）、ItemToken(order_id)、GeneratedKeyToken(57)
private final List<SQLToken> sqlTokens = new LinkedList<>();
    
private int parametersIndex;
//所有列
private final Collection<Column> columns = new LinkedList<>();
//批量条件<列名，值>
private final List<Conditions> multipleConditions = new LinkedList<>();
//最后一列结束位置
private int columnsListLastPosition;
//自增列是第几个，-1表示没有
private int generateKeyColumnIndex = -1;
//VALUES或VALUE后面值开始的位置
private int afterValuesPosition;
//VALUES或VALUE后面值结束位置 
private int valuesListLastPosition;

private GeneratedKey generatedKey;
```

  路由

```java
@Override
public SQLRouteResult route(final String logicSQL, final List<Object> parameters, final SQLStatement sqlStatement) {
    //SQL路由结果
    SQLRouteResult result = new SQLRouteResult(sqlStatement);
    if (sqlStatement instanceof InsertStatement && null != ((InsertStatement) sqlStatement).getGeneratedKey()) {
        //处理插入sql的主键
        processGeneratedKey(parameters, (InsertStatement) sqlStatement, result);
    }
    //路由
    RoutingResult routingResult = route(parameters, sqlStatement);
    //SQL重写引擎
    SQLRewriteEngine rewriteEngine = new SQLRewriteEngine(shardingRule, logicSQL, databaseType, sqlStatement);
    boolean isSingleRouting = routingResult.isSingleRouting();
    if (sqlStatement instanceof SelectStatement && null != ((SelectStatement) sqlStatement).getLimit()) {
        // 处理分页
        processLimit(parameters, (SelectStatement) sqlStatement, isSingleRouting);
    }
    //重写
    SQLBuilder sqlBuilder = rewriteEngine.rewrite(!isSingleRouting);
    // 笛卡尔积结果生成 ExecutionUnit
    if (routingResult instanceof CartesianRoutingResult) {
        for (CartesianDataSource cartesianDataSource : ((CartesianRoutingResult) routingResult).getRoutingDataSources()) {
           
            for (CartesianTableReference cartesianTableReference : cartesianDataSource.getRoutingTableReferences()) {
                result.getExecutionUnits().add(new SQLExecutionUnit(cartesianDataSource.getDataSource(), rewriteEngine.generateSQL(cartesianTableReference, sqlBuilder)));
            }
        }
    } else {
         //将每个逻辑表名转化成真实表名
        for (TableUnit each : routingResult.getTableUnits().getTableUnits()) {
            result.getExecutionUnits().add(new SQLExecutionUnit(each.getDataSourceName(), rewriteEngine.generateSQL(each, sqlBuilder)));
        }
    }
    // 打印 SQL
    if (showSQL) {
        SQLLogger.logSQL(logicSQL, sqlStatement, result.getExecutionUnits(), parameters);
    }
    return result;
}


 private RoutingResult route(final List<Object> parameters, final SQLStatement sqlStatement) {
     //获取所有sql语句中的逻辑表名
        Collection<String> tableNames = sqlStatement.getTables().getTableNames();
        RoutingEngine routingEngine;
        if (tableNames.isEmpty()) {
            routingEngine = new DatabaseAllRoutingEngine(shardingRule.getDataSourceMap());
            //1.只有一个逻辑表名2.是否表名全为BindingTable3.所有逻辑表名都在默认数据库
        } else if (1 == tableNames.size() || shardingRule.isAllBindingTables(tableNames) || shardingRule.isAllInDefaultDataSource(tableNames)) {
            //使用第一个表名做路由。
            //如：SELECT * FROM t_order o join t_order_item i ON o.order_id = i.order_id 则使用t_order
            routingEngine = new SimpleRoutingEngine(shardingRule, parameters, tableNames.iterator().next(), sqlStatement);
        } else {
            // TODO config for cartesian set
            //可配置是否执行笛卡尔积
            routingEngine = new ComplexRoutingEngine(shardingRule, parameters, tableNames, sqlStatement);
        }
     //路由
        return routingEngine.route();
    }
```

## 路由引擎

### RoutingEngine

```java
public interface RoutingEngine {
    
    /**
     * Route.
     *
     * @return routing result
     */
    RoutingResult route();
}
```

#### SimpleRoutingEngine

```java
private final ShardingRule shardingRule;
//参数
private final List<Object> parameters;
//逻辑表名
private final String logicTableName;

private final SQLStatement sqlStatement;
```



```java
@Override
public RoutingResult route() {
    //获取规则
    TableRule tableRule = shardingRule.getTableRule(logicTableName);
    //Hint则使用分片管理器获取，否则使用数据库分片算法。获取逻辑表、列名、条件值
    List<ShardingValue> databaseShardingValues = getDatabaseShardingValues(tableRule);
    //表分片算法。获取逻辑表、列名、条件值
    List<ShardingValue> tableShardingValues = getTableShardingValues(tableRule);
    //根据逻辑表、列名、条件值路由出真实数据库节点
    Collection<String> routedDataSources = routeDataSources(tableRule, databaseShardingValues);
    Collection<DataNode> routedDataNodes = new LinkedList<>();
    for (String each : routedDataSources) {
        routedDataNodes.addAll(routeTables(tableRule, each, tableShardingValues));
    }
    //生成RoutingResult。包含TableUnit集合。TableUnit包括真实数据源、逻辑表名、真实表名
    return generateRoutingResult(routedDataNodes);
}

    private Collection<String> routeDataSources(final TableRule tableRule, final List<ShardingValue> databaseShardingValues) {
        //获取所有真实数据库名
        Collection<String> availableTargetDatabases = tableRule.getActualDatasourceNames();
        if (databaseShardingValues.isEmpty()) {
            return availableTargetDatabases;
        }
        //ShardingStrategy#doSharding
        Collection<String> result = shardingRule.getDatabaseShardingStrategy(tableRule).doSharding(availableTargetDatabases, databaseShardingValues);
        Preconditions.checkState(!result.isEmpty(), "no database route info");
        return result;
    }
```

#### 分库规则

假如user_id模2分库，demo_ds_0、demo_ds_1    `shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(new InlineShardingStrategyConfiguration("user_id", "demo_ds_${user_id % 2}"));`

如SELECT o.* FROM t_order o  WHERE o.user_id= 10

则返回ListShardingValue ,t_order（逻辑表）、user_id（列名）、10（值）

```java
private List<ShardingValue> getDatabaseShardingValues(final TableRule tableRule) {
    //获取分库规则
    ShardingStrategy strategy = shardingRule.getDatabaseShardingStrategy(tableRule);
    //分库的列名
    return HintManagerHolder.isUseShardingHint() ? getDatabaseShardingValuesFromHint(strategy.getShardingColumns()) : getShardingValues(strategy.getShardingColumns());
}
```



```java
private List<ShardingValue> getShardingValues(final Collection<String> shardingColumns) {
    List<ShardingValue> result = new ArrayList<>(shardingColumns.size());
    //查看sql的条件里面是否能找到分库规则使用的列名
    for (String each : shardingColumns) {
        Optional<Condition> condition = sqlStatement.getConditions().find(new Column(each, logicTableName));
        if (condition.isPresent()) {
            result.add(condition.get().getShardingValue(parameters));
        }
    }
    return result;
}
```
Condition

```java
public ShardingValue getShardingValue(final List<Object> parameters) {
    List<Comparable<?>> conditionValues = getValues(parameters);
    switch (operator) {
            //'='或者"IN"返回 ListShardingValue
        case EQUAL:
        case IN:
            return new ListShardingValue<>(column.getTableName(), column.getName(), conditionValues);
            //"BETWEEN"返回RangeShardingValue
        case BETWEEN:
            return new RangeShardingValue<>(column.getTableName(), column.getName(), Range.range(conditionValues.get(0), BoundType.CLOSED, conditionValues.get(1), BoundType.CLOSED));
        default:
            throw new UnsupportedOperationException(operator.getExpression());
    }
}
```

#### 分表规则

假如按order_id模2分表   `shardingRuleConfig.setDefaultTableShardingStrategyConfig(new StandardShardingStrategyConfiguration("order_id", ModuloShardingTableAlgorithm.class.getName()));`

```java
private List<ShardingValue> getTableShardingValues(final TableRule tableRule) {
    ShardingStrategy strategy = shardingRule.getTableShardingStrategy(tableRule);
    return HintManagerHolder.isUseShardingHint() ? getTableShardingValuesFromHint(strategy.getShardingColumns()) : getShardingValues(strategy.getShardingColumns());
}
```

### 分片策略实现

```java
public interface ShardingStrategy {
    //availableTargetNames真实数据库名集合
    //SQL 的逻辑表、列名、条件（分片值）集合
    Collection<String> doSharding(Collection<String> availableTargetNames, Collection<ShardingValue> shardingValues);
}
```

#### StandardShardingStrategy

```java
@Override
public Collection<String> doSharding(final Collection<String> availableTargetNames, final Collection<ShardingValue> shardingValues) {
    //获取第一个分片值
    ShardingValue shardingValue = shardingValues.iterator().next();
    Collection<String> shardingResult = shardingValue instanceof ListShardingValue
            ? doSharding(availableTargetNames, (ListShardingValue) shardingValue) : doSharding(availableTargetNames, (RangeShardingValue) shardingValue);
    Collection<String> result = new TreeSet<>(String.CASE_INSENSITIVE_ORDER);
    result.addAll(shardingResult);
    return result;
}


    private Collection<String> doSharding(final Collection<String> availableTargetNames, final ListShardingValue<?> shardingValue) {
        Collection<String> result = new LinkedList<>();
        //ListShardingValue-》List<PreciseShardingValue>
        //每个值都划出来，使用preciseShardingAlgorithm策略，获取真实数据库名
        for (PreciseShardingValue<?> each : transferToPreciseShardingValues(shardingValue)) {
            result.add(preciseShardingAlgorithm.doSharding(availableTargetNames, each));
        }
        return result;
    }


    private Collection<String> doSharding(final Collection<String> availableTargetNames, final RangeShardingValue<?> shardingValue) {
        if (!rangeShardingAlgorithm.isPresent()) {
            throw new UnsupportedOperationException("Cannot find range sharding strategy in sharding rule.");
        }
        //rangeShardingAlgorithm策略获取真实数据库名
        return rangeShardingAlgorithm.get().doSharding(availableTargetNames, shardingValue);
    }
```





```java
public final class ModuloShardingAlgorithm implements PreciseShardingAlgorithm<Integer> {
    
    @Override
    public String doSharding(final Collection<String> availableTargetNames, final PreciseShardingValue<Integer> shardingValue) {
        for (String each : availableTargetNames) {
            if (each.endsWith(shardingValue.getValue() % 2 + "")) {
                return each;
            }
        }
        throw new IllegalArgumentException();
    }
}
```

## SQL重写

### SQLRewriteEngine

```java
private final ShardingRule shardingRule;
//原始sql
private final String originalSQL;
//数据库类型
private final DatabaseType databaseType;
//所有的SQL词根
private final List<SQLToken> sqlTokens = new LinkedList<>();

private final SQLStatement sqlStatement;
```



```java
public SQLBuilder rewrite(final boolean isRewriteLimit) {
    SQLBuilder result = new SQLBuilder();
    if (sqlTokens.isEmpty()) {
        result.appendLiterals(originalSQL);
        return result;
    }
    int count = 0;
    //sqlTokens按beginPos次序排序
    sortByBeginPosition();
    for (SQLToken each : sqlTokens) {
        if (0 == count) {
            //第一个词素加入
            result.appendLiterals(originalSQL.substring(0, each.getBeginPosition()));
        }
        TableToken（t_order）、ItemToken(order_id)、GeneratedKeyToken(57)
        //表名
        if (each instanceof TableToken) {
            appendTableToken(result, (TableToken) each, count, sqlTokens);
        //ItemToken 如列名
        } else if (each instanceof ItemsToken) {
            appendItemsToken(result, (ItemsToken) each, count, sqlTokens);
        //limit 后面的rowCount、offset
        } else if (each instanceof RowCountToken) {
            appendLimitRowCount(result, (RowCountToken) each, count, sqlTokens, isRewriteLimit);
          //limit 后面的offset
        } else if (each instanceof OffsetToken) {
            appendLimitOffsetToken(result, (OffsetToken) each, count, sqlTokens, isRewriteLimit);
          //转换order
        } else if (each instanceof OrderByToken) {
            appendOrderByToken(result, count, sqlTokens);
        }
        count++;
    }
    return result;
}
```





```java
public String generateSQL(final TableUnit tableUnit, final SQLBuilder sqlBuilder) {
    return sqlBuilder.toSQL(getTableTokens(tableUnit));
}
```



根据TableUnit转换逻辑表名和真实表名

```java
private Map<String, String> getTableTokens(final TableUnit tableUnit) {
    Map<String, String> tableTokens = new HashMap<>();
    tableTokens.put(tableUnit.getLogicTableName(), tableUnit.getActualTableName());
    Optional<BindingTableRule> bindingTableRule = shardingRule.findBindingTableRule(tableUnit.getLogicTableName());
    if (bindingTableRule.isPresent()) {
        tableTokens.putAll(getBindingTableTokens(tableUnit, bindingTableRule.get()));
    }
    return tableTokens;
}
```

## 处理自增键

### ParsingSQLRouter

```java
private void processGeneratedKey(final List<Object> parameters, final InsertStatement insertStatement, final SQLRouteResult sqlRouteResult) {
    GeneratedKey generatedKey = insertStatement.getGeneratedKey();
    if (parameters.isEmpty()) {
        sqlRouteResult.getGeneratedKeys().add(generatedKey.getValue());
    } else if (parameters.size() == generatedKey.getIndex()) {
        Number key = shardingRule.generateKey(insertStatement.getTables().getSingleTableName());
        parameters.add(key);
        setGeneratedKeys(sqlRouteResult, key);
    } else if (-1 != generatedKey.getIndex()) {
        setGeneratedKeys(sqlRouteResult, (Number) parameters.get(generatedKey.getIndex()));
    }
}
```





### ShardingRule

```java
public Number generateKey(final String logicTableName) {
    Optional<TableRule> tableRule = tryFindTableRule(logicTableName);
    if (!tableRule.isPresent()) {
        throw new ShardingJdbcException("Cannot find strategy for generate keys.");
    }
    if (null != tableRule.get().getKeyGenerator()) {
        return tableRule.get().getKeyGenerator().generateKey();
    }
    return defaultKeyGenerator.generateKey();
}
```





```java
public interface KeyGenerator {
    
    /**
     * Generate key.
     * 
     * @return generated key
     */
    Number generateKey();
}
```