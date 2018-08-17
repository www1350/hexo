---
title: sharding-jdbc源码解析-sql解析（五）
tags:
  - sharding-jdbc
categories:
  - 中间件
  - 数据库
abbrlink: 8ce3a370
date: 2018-05-20 01:54:35
---

## SQL解析流程



![sequencediagram01](https://user-images.githubusercontent.com/7789698/40620293-e0708094-62ca-11e8-8fa9-df9a742d7673.jpg)

## SQL解析引擎

### SQLParsingEngine

sql分析引擎

会先使用词法分析引擎进行词法分析

```java
@RequiredArgsConstructor
public final class SQLParsingEngine {
    //数据库类型，如mysql
    private final DatabaseType dbType;
    
    private final String sql;
    
    private final ShardingRule shardingRule;
    
    /**
     * Parse SQL.
     * 
     * @return parsed SQL statement
     */
    public SQLStatement parse() {
        //根据sql和数据库类型创建词法分析引擎
        LexerEngine lexerEngine = LexerEngineFactory.newInstance(dbType, sql);
        //读入第一个标记
        lexerEngine.nextToken();
        //使用sql解析工厂创建sql解析器并解析
        return SQLParserFactory.newInstance(dbType, lexerEngine.getCurrentToken().getType(), shardingRule, lexerEngine).parse();
    }
}
```

### SQLParserFactory

sql解析器工厂，负责根据sql第一个标记类型创建出相应的解析器

- 解析器工厂（负责根据数据库类型选择具体解析器）分类：

  SELECT<--->SelectParserFactory、INSERT<--->InsertParserFactory、UPDATE<--->UpdateParserFactory、DELETE<--->DeleteParserFactory、CREATE<--->CreateParserFactory、ALTER<--->AlterParserFactory、DROP<--->DropParserFactory、TRUNCATE<--->TruncateParserFactory

- 解析器分类：

  SELECT<--->AbstractSelectParser、INSERT<--->AbstractInsertParser、UPDATE<--->AbstractUpdateParser、DELETE<--->AbstractDeleteParser、CREATE<--->AbstractCreateParser、ALTER<--->AbstractAlterParser、DROP<--->AbstractDropParser、TRUNCATE<--->AbstractTruncateParser



## SQL解析器

### SQLParser

解析器

![diagram](https://user-images.githubusercontent.com/7789698/40277466-dbb0dcae-5c51-11e8-8e48-eb1f17b7d257.jpg)



```java
public interface SQLParser {
    
    /**
     * Parse SQL.
     *
     * @return SQL statement
     */
    SQLStatement parse();
}
```



## 插入

以插入为例

![2121212](https://user-images.githubusercontent.com/7789698/40279805-b948e74a-5c7b-11e8-8c09-d184ca02db14.png)

### 插入语句解析器

```java
public abstract class AbstractInsertParser implements SQLParser {
    
    @Getter(AccessLevel.PROTECTED)
    private final ShardingRule shardingRule;
    
    @Getter(AccessLevel.PROTECTED)
    private final LexerEngine lexerEngine;
    
    private final AbstractInsertClauseParserFacade insertClauseParserFacade;
    
    public AbstractInsertParser(final ShardingRule shardingRule, final LexerEngine lexerEngine, final AbstractInsertClauseParserFacade insertClauseParserFacade) {
        this.shardingRule = shardingRule;
        this.lexerEngine = lexerEngine;
        this.insertClauseParserFacade = insertClauseParserFacade;
    }
    
    @Override
    public final DMLStatement parse() {
        //读取下一个标记，比如INSERT INTO t_order (user_id, status) VALUES (10, 'INIT')
        //就读取到INTO
        lexerEngine.nextToken();
        //创建InsertStatement
        InsertStatement result = new InsertStatement();
        //读取INTO后面的表名
        insertClauseParserFacade.getInsertIntoClauseParser().parse(result);
        //读取插入的列
        insertClauseParserFacade.getInsertColumnsClauseParser().parse(result);
        //不支持INSERT SELECT
        if (lexerEngine.equalAny(DefaultKeyword.SELECT, Symbol.LEFT_PAREN)) {
            throw new UnsupportedOperationException("Cannot INSERT SELECT");
        }
        //读取VALUES 后面
        insertClauseParserFacade.getInsertValuesClauseParser().parse(result);
         //读取SET 后面
        insertClauseParserFacade.getInsertSetClauseParser().parse(result);
        //处理自增键转化为GeneratedKeyToken
        appendGenerateKey(result);
        return result;
    }
    
    private void appendGenerateKey(final InsertStatement insertStatement) {
        String tableName = insertStatement.getTables().getSingleTableName();
        //获取自增列列名
        Optional<String> generateKeyColumn = shardingRule.getGenerateKeyColumn(tableName);
        if (!generateKeyColumn.isPresent() || null != insertStatement.getGeneratedKey()) {
            return;
        } 
        //拿到刚才解析的所有列名
        ItemsToken columnsToken = new ItemsToken(insertStatement.getColumnsListLastPosition());
        columnsToken.getItems().add(generateKeyColumn.get());
        insertStatement.getSqlTokens().add(columnsToken);
        //处理自增id
        insertStatement.getSqlTokens().add(new GeneratedKeyToken(insertStatement.getValuesListLastPosition()));
    }
}
```



![diagram](https://user-images.githubusercontent.com/7789698/40279597-c877da7c-5c77-11e8-9ddd-6840e7800550.png)

### AbstractInsertClauseParserFacade

门面模式

```java
@RequiredArgsConstructor
@Getter
public abstract class AbstractInsertClauseParserFacade {
    
    private final InsertIntoClauseParser insertIntoClauseParser;
    
    private final InsertColumnsClauseParser insertColumnsClauseParser;
    
    private final InsertValuesClauseParser insertValuesClauseParser;
    
    private final InsertSetClauseParser insertSetClauseParser;
}
```



![diagram](https://user-images.githubusercontent.com/7789698/40279583-adb78570-5c77-11e8-8c00-d7f582f5be96.png)



### InsertIntoClauseParser

INTO 部分解析

```java
public void parse(final InsertStatement insertStatement) {
    lexerEngine.unsupportedIfEqual(getUnsupportedKeywordsBeforeInto());
    //一直读取直到结束或者 "INTO"
    lexerEngine.skipUntil(DefaultKeyword.INTO);
    //读取"INTO"下一个标记
    lexerEngine.nextToken();
    //解析表
    tableReferencesClauseParser.parse(insertStatement, true);
    skipBetweenTableAndValues(insertStatement);
}
```



#### 表解析器MySQLTableReferencesClauseParser

```java
public final void parse(final SQLStatement sqlStatement, final boolean isSingleTableOnly) {
    do {
        parseTableReference(sqlStatement, isSingleTableOnly);
        //','分割
    } while (lexerEngine.skipIfEqual(Symbol.COMMA));
}

    @Override
    protected void parseTableReference(final SQLStatement sqlStatement, final boolean isSingleTableOnly) {
        parseTableFactor(sqlStatement, isSingleTableOnly);
        //解析PARTITION，Mysql不支持。
        parsePartition();
        //解析使用索引
        parseIndexHint(sqlStatement);
    }

 protected final void parseTableFactor(final SQLStatement sqlStatement, final boolean isSingleTableOnly) {
     //"INTO"下一个标记开始的下标
     //如：INSERT INTO t_order (user_id, status) VALUES (10, 'INIT') 
     //是12
        final int beginPosition = lexerEngine.getCurrentToken().getEndPosition() - lexerEngine.getCurrentToken().getLiterals().length();
     //"INTO"下一个字面量，就是逻辑表名
     //如：INSERT INTO t_order (user_id, status) VALUES (10, 'INIT') 
     //是t_order
        String literals = lexerEngine.getCurrentToken().getLiterals();
     //下一个标记，如(、AS等，
        lexerEngine.nextToken();
     //不能支持`schema.table`
        if (lexerEngine.equalAny(Symbol.DOT)) {
            throw new UnsupportedOperationException("Cannot support SQL for `schema.table`");
        }
     //移除 '`'和 '"'
        String tableName = SQLUtil.getExactlyValue(literals);
     //解析 AS ，拿到别名，并跳到别名下一个标记
        Optional<String> alias = aliasClauseParser.parse();
     
        if (isSingleTableOnly || shardingRule.tryFindTableRule(tableName).isPresent() || shardingRule.findBindingTableRule(tableName).isPresent()
                || shardingRule.getDataSourceMap().containsKey(shardingRule.getDefaultDataSourceName())) {
            //添加sqlToken （12，t_order）
            sqlStatement.getSqlTokens().add(new TableToken(beginPosition, literals));
            //添加表名和别名
            sqlStatement.getTables().add(new Table(tableName, alias));
        }
     //解析join
        parseJoinTable(sqlStatement);
        if (isSingleTableOnly && !sqlStatement.getTables().isSingleTable()) {
            throw new UnsupportedOperationException("Cannot support Multiple-Table.");
        }
     
 private void parseIndexHint(final SQLStatement sqlStatement) {
     //USE、IGNORE、FORCE
        if (getLexerEngine().skipIfEqual(DefaultKeyword.USE, MySQLKeyword.IGNORE, MySQLKeyword.FORCE)) {
            //INDEX、KEY、FOR、JOIN、ORDER、GROUP、BY
            getLexerEngine().skipAll(DefaultKeyword.INDEX, DefaultKeyword.KEY, DefaultKeyword.FOR, DefaultKeyword.JOIN, DefaultKeyword.ORDER, DefaultKeyword.GROUP, DefaultKeyword.BY);
            getLexerEngine().skipParentheses(sqlStatement);
        }
    }
```

#### 别名解析器AliasClauseParser

```java
public Optional<String> parse() {
    //解析到AS了，就在往下读一个标记
    if (lexerEngine.skipIfEqual(DefaultKeyword.AS)) {
        //读到符号返回不存在
        if (lexerEngine.equalAny(Symbol.values())) {
            return Optional.absent();
        }
        //接下来的字面量去 '`'和 '"'
        String result = SQLUtil.getExactlyValue(lexerEngine.getCurrentToken().getLiterals());
        //往下读
        lexerEngine.nextToken();
        //返回别名
        return Optional.of(result);
    }
    //直接别名的
    if (lexerEngine.equalAny(
            Literals.IDENTIFIER, Literals.CHARS, DefaultKeyword.USER, DefaultKeyword.END, DefaultKeyword.CASE, DefaultKeyword.KEY, DefaultKeyword.INTERVAL, DefaultKeyword.CONSTRAINT)) {
        String result = SQLUtil.getExactlyValue(lexerEngine.getCurrentToken().getLiterals());
        lexerEngine.nextToken();
        //返回别名
        return Optional.of(result);
    }
    return Optional.absent();
}
```



#### InsertColumnsClauseParser

列 部分解析

```java
public void parse(final InsertStatement insertStatement) {
    Collection<Column> result = new LinkedList<>();
    // "("开头
    if (lexerEngine.equalAny(Symbol.LEFT_PAREN)) {
        //刚才解析出来的表名
        String tableName = insertStatement.getTables().getSingleTableName();
        //获取该表的分片规则列的列名
        Optional<String> generateKeyColumn = shardingRule.getGenerateKeyColumn(tableName);
        int count = 0;
        //读取INTO 的所有列名
        do {
            lexerEngine.nextToken();
            String columnName = SQLUtil.getExactlyValue(lexerEngine.getCurrentToken().getLiterals());
            result.add(new Column(columnName, tableName));
            lexerEngine.nextToken();
            if (generateKeyColumn.isPresent() && generateKeyColumn.get().equalsIgnoreCase(columnName)) {
                //记下需要自增的列的位置
                insertStatement.setGenerateKeyColumnIndex(count);
            }
            count++;
        } while (!lexerEngine.equalAny(Symbol.RIGHT_PAREN) && !lexerEngine.equalAny(Assist.END));      
//记录最后一列结束位置
        insertStatement.setColumnsListLastPosition(lexerEngine.getCurrentToken().getEndPosition() - lexerEngine.getCurrentToken().getLiterals().length());
        //跳过")"
        lexerEngine.nextToken();
    }
    //设置列名
    insertStatement.getColumns().addAll(result);
}
```



#### InsertValuesClauseParser

```java
public void parse(final InsertStatement insertStatement) {
    Collection<Keyword> valueKeywords = new LinkedList<>();
    //VALUES
    valueKeywords.add(DefaultKeyword.VALUES);
    //mysql是VALUE
    valueKeywords.addAll(Arrays.asList(getSynonymousKeywordsForValues()));
    //读到VALUES或VALUE，接着读下一个
    if (lexerEngine.skipIfEqual(valueKeywords.toArray(new Keyword[valueKeywords.size()]))) {
        //记录VALUES或VALUE后面开始的位置
        insertStatement.setAfterValuesPosition(lexerEngine.getCurrentToken().getEndPosition() - lexerEngine.getCurrentToken().getLiterals().length());
        //VALUES或VALUE的值和表名组成Condition
        parseValues(insertStatement);
        //如果是","表示批量插入的写法
        if (lexerEngine.equalAny(Symbol.COMMA)) {
            parseMultipleValues(insertStatement);
        }
    }
}


private void parseValues(final InsertStatement insertStatement) {
    //跳过"("
        lexerEngine.accept(Symbol.LEFT_PAREN);
        List<SQLExpression> sqlExpressions = new LinkedList<>();
        do {
            //表达式，就是每一个值，逗号隔开
            sqlExpressions.add(expressionClauseParser.parse(insertStatement));
        } while (lexerEngine.skipIfEqual(Symbol.COMMA));
  //记录结束位置      
  insertStatement.setValuesListLastPosition(lexerEngine.getCurrentToken().getEndPosition() - lexerEngine.getCurrentToken().getLiterals().length());
        int count = 0;
    //列名和值组装成条件Condition
        for (Column each : insertStatement.getColumns()) {
            SQLExpression sqlExpression = sqlExpressions.get(count);
            insertStatement.getConditions().add(new Condition(each, sqlExpression), shardingRule);
            if (insertStatement.getGenerateKeyColumnIndex() == count) {
                insertStatement.setGeneratedKey(createGeneratedKey(each, sqlExpression));
            }
            count++;
        }
    //跳过")"
        lexerEngine.accept(Symbol.RIGHT_PAREN);
    }
```



## 查询

select语法

```SQL
SELECT
    [ALL | DISTINCT | DISTINCTROW ]
      [HIGH_PRIORITY]
      [STRAIGHT_JOIN]
      [SQL_SMALL_RESULT][SQL_BIG_RESULT] [SQL_BUFFER_RESULT]
      [SQL_CACHE | SQL_NO_CACHE][SQL_CALC_FOUND_ROWS]
    select_expr [, select_expr ...]
    [FROM table_references
      [PARTITION partition_list]
    [WHERE where_condition]
    [GROUP BY {col_name | expr | position}
      [ASC | DESC], ... [WITH ROLLUP]]
    [HAVING where_condition]
    [ORDER BY {col_name | expr | position}
      [ASC | DESC], ...]
    [LIMIT {[offset,] row_count | row_count OFFSET offset}]
    [PROCEDURE procedure_name(argument_list)]
    [INTO OUTFILE 'file_name'
        [CHARACTER SET charset_name]
        export_options
      | INTO DUMPFILE 'file_name'
      | INTO var_name [, var_name]]
    [FOR UPDATE | LOCK IN SHARE MODE]]
```



*select_expr* 指的是你想获取的列，至少要有一个

*table_references* 指的是表或者表中的行

```SQL
tbl_name [[AS] alias] [index_hint_list]

index_hint_list:
    index_hint [index_hint] ...

index_hint:
    USE {INDEX|KEY}
      [FOR {JOIN|ORDER BY|GROUP BY}] ([index_list])
  | IGNORE {INDEX|KEY}
      [FOR {JOIN|ORDER BY|GROUP BY}] (index_list)
  | FORCE {INDEX|KEY}
      [FOR {JOIN|ORDER BY|GROUP BY}] (index_list)

index_list:
    index_name [, index_name] ...
```

*select … partition* 分区 。https://dev.mysql.com/doc/refman/5.7/en/partitioning-selection.html



*Expression Syntax* 

```SQL
expr:
    expr OR expr
  | expr || expr
  | expr XOR expr
  | expr AND expr
  | expr && expr
  | NOT expr
  | ! expr
  | boolean_primary IS [NOT] {TRUE | FALSE | UNKNOWN}
  | boolean_primary

boolean_primary:
    boolean_primary IS [NOT] NULL
  | boolean_primary <=> predicate
  | boolean_primary comparison_operator predicate
  | boolean_primary comparison_operator {ALL | ANY} (subquery)
  | predicate

comparison_operator: = | >= | > | <= | < | <> | !=

predicate:
    bit_expr [NOT] IN (subquery)
  | bit_expr [NOT] IN (expr [, expr] ...)
  | bit_expr [NOT] BETWEEN bit_expr AND predicate
  | bit_expr SOUNDS LIKE bit_expr
  | bit_expr [NOT] LIKE simple_expr [ESCAPE simple_expr]
  | bit_expr [NOT] REGEXP bit_expr
  | bit_expr

bit_expr:
    bit_expr | bit_expr
  | bit_expr & bit_expr
  | bit_expr << bit_expr
  | bit_expr >> bit_expr
  | bit_expr + bit_expr
  | bit_expr - bit_expr
  | bit_expr * bit_expr
  | bit_expr / bit_expr
  | bit_expr DIV bit_expr
  | bit_expr MOD bit_expr
  | bit_expr % bit_expr
  | bit_expr ^ bit_expr
  | bit_expr + interval_expr
  | bit_expr - interval_expr
  | simple_expr

simple_expr:
    literal
  | identifier
  | function_call
  | simple_expr COLLATE collation_name
  | param_marker
  | variable
  | simple_expr || simple_expr
  | + simple_expr
  | - simple_expr
  | ~ simple_expr
  | ! simple_expr
  | BINARY simple_expr
  | (expr [, expr] ...)
  | ROW (expr, expr [, expr] ...)
  | (subquery)
  | EXISTS (subquery)
  | {identifier expr}
  | match_expr
  | case_expr
  | interval_expr
```

*Operator Precedence* 

运算符的优先级

```SQL
INTERVAL
BINARY, COLLATE
!
- (unary minus), ~ (unary bit inversion)
^
*, /, DIV, %, MOD
-, +
<<, >>
&
|
= (comparison), <=>, >=, >, <=, <, <>, !=, IS, LIKE, REGEXP, IN
BETWEEN, CASE, WHEN, THEN, ELSE
NOT
AND, &&
XOR
OR, ||
= (assignment), :=
```

  







### 查询语句解析器

```java
@RequiredArgsConstructor
@Getter(AccessLevel.PROTECTED)
public abstract class AbstractSelectParser implements SQLParser {
    
    private static final String DERIVED_COUNT_ALIAS = "AVG_DERIVED_COUNT_%s";
    
    private static final String DERIVED_SUM_ALIAS = "AVG_DERIVED_SUM_%s";
    
    private static final String ORDER_BY_DERIVED_ALIAS = "ORDER_BY_DERIVED_%s";
    
    private static final String GROUP_BY_DERIVED_ALIAS = "GROUP_BY_DERIVED_%s";
    
    private final ShardingRule shardingRule;
    
    private final LexerEngine lexerEngine;
    
    private final AbstractSelectClauseParserFacade selectClauseParserFacade;
    
    private final List<SelectItem> items = new LinkedList<>();
    
    @Override
    public final SelectStatement parse() {
        //解析成SelectStatement
        SelectStatement result = parseInternal();
        //是否包含子查询，包含的话合并子查询语句
        if (result.containsSubQuery()) {
            result = result.mergeSubQueryStatement();
        }
        // TODO move to rewrite
        appendDerivedColumns(result);
        appendDerivedOrderBy(result);
        return result;
    }  
    
   private SelectStatement parseInternal() {
        SelectStatement result = new SelectStatement();
       //跳过第一个SELECT
        lexerEngine.nextToken();
        parseInternal(result);
        return result;
    }
    

}
```



#### MySQLSelectParser

```java
@Override
protected void parseInternal(final SelectStatement selectStatement) {
    //解析distinct
    parseDistinct();
    //解析Option
    parseSelectOption();
    //解析列
    parseSelectList(selectStatement, getItems());
    //解析from
    parseFrom(selectStatement);
    //解析WHERE
    parseWhere(getShardingRule(), selectStatement, getItems());
    //解析Group By
    parseGroupBy(selectStatement);
    //解析Having
    parseHaving();
    //解析Order By
    parseOrderBy(selectStatement);
    //解析Limit
    parseLimit(selectStatement);
    //解析
    parseSelectRest();
}
```



### AbstractSelectClauseParserFacade

```java
@RequiredArgsConstructor
@Getter
public abstract class AbstractSelectClauseParserFacade {
    
    private final DistinctClauseParser distinctClauseParser;
    
    private final SelectListClauseParser selectListClauseParser;
    
    private final TableReferencesClauseParser tableReferencesClauseParser;
    
    private final WhereClauseParser whereClauseParser;
    
    private final GroupByClauseParser groupByClauseParser;
    
    private final HavingClauseParser havingClauseParser;
    
    private final OrderByClauseParser orderByClauseParser;
    
    private final SelectRestClauseParser selectRestClauseParser;
}
```

#### 解析Distinct

```java
@RequiredArgsConstructor
public class DistinctClauseParser implements SQLClauseParser {
    
    private final LexerEngine lexerEngine;
    
    /**
     * Parse distinct.
     */
    public final void parse() {
        lexerEngine.skipAll(DefaultKeyword.ALL);
        Collection<Keyword> distinctKeywords = new LinkedList<>();
        distinctKeywords.add(DefaultKeyword.DISTINCT);
        distinctKeywords.addAll(Arrays.asList(getSynonymousKeywordsForDistinct()));
        lexerEngine.unsupportedIfEqual(distinctKeywords.toArray(new Keyword[distinctKeywords.size()]));
    }
    
    protected Keyword[] getSynonymousKeywordsForDistinct() {
        return new Keyword[0];
    }
}
```



#### 解析Option

```java
@RequiredArgsConstructor
public final class MySQLSelectOptionClauseParser implements SQLClauseParser {
    
    private final LexerEngine lexerEngine;
    
    /**
     * 解析Option.
     */
    public void parse() {
        //HIGH_PRIORITY、STRAIGHT_JOIN、SQL_SMALL_RESULT、SQL_BIG_RESULT、SQL_BUFFER_RESULT、SQL_CACHE、SQL_NO_CACHE、SQL_CALC_FOUND_ROWS
        lexerEngine.skipAll(MySQLKeyword.HIGH_PRIORITY, MySQLKeyword.STRAIGHT_JOIN, 
                MySQLKeyword.SQL_SMALL_RESULT, MySQLKeyword.SQL_BIG_RESULT, MySQLKeyword.SQL_BUFFER_RESULT, MySQLKeyword.SQL_CACHE, MySQLKeyword.SQL_NO_CACHE, MySQLKeyword.SQL_CALC_FOUND_ROWS);
    }
}
```



#### 解析列

SelectListClauseParser

```java
public void parse(final SelectStatement selectStatement, final List<SelectItem> items) {
    do {
        selectStatement.getItems().add(parseSelectItem(selectStatement));
     //','分割
    } while (lexerEngine.skipIfEqual(Symbol.COMMA));
    selectStatement.setSelectListLastPosition(lexerEngine.getCurrentToken().getEndPosition() - lexerEngine.getCurrentToken().getLiterals().length());
    items.addAll(selectStatement.getItems());
}

    //解析列
    private SelectItem parseSelectItem(final SelectStatement selectStatement) {
        //跳过CONNECT_BY_ROOT
        lexerEngine.skipIfEqual(getSkippedKeywordsBeforeSelectItem());
        SelectItem result;
        //Mysql是false，没有行号。sqlserver是ROW_NUMBER
        if (isRowNumberSelectItem()) {
            result = parseRowNumberSelectItem(selectStatement);
        // 解析'*'
        } else if (isStarSelectItem()) {
            selectStatement.setContainStar(true);
            //StarSelectItem
            result = parseStarSelectItem();
        //解析MAX, MIN, SUM, AVG, COUNT
        } else if (isAggregationSelectItem()) {
            //转化AggregationSelectItem
            result = parseAggregationSelectItem(selectStatement);
            parseRestSelectItem(selectStatement);
        //其他情况，也就是直接列名的
        } else {
            result = new CommonSelectItem(SQLUtil.getExactlyValue(parseCommonSelectItem(selectStatement) + parseRestSelectItem(selectStatement)), aliasClauseParser.parse());
        }
        return result;
    }
```





```java
private String parseCommonSelectItem(final SelectStatement selectStatement) {
    String literals = lexerEngine.getCurrentToken().getLiterals();
    int position = lexerEngine.getCurrentToken().getEndPosition() - literals.length();
    StringBuilder result = new StringBuilder();
    result.append(literals);
    lexerEngine.nextToken();
    //'('开头
    if (lexerEngine.equalAny(Symbol.LEFT_PAREN)) {
        //跳过括号内所有令牌,并返回括号内内容
        result.append(lexerEngine.skipParentheses(selectStatement));
    //字面量带'.'
    } else if (lexerEngine.equalAny(Symbol.DOT)) {
        //解析表名
        String tableName = SQLUtil.getExactlyValue(literals);
        //看绑定的逻辑表名有没有
        if (shardingRule.tryFindTableRule(tableName).isPresent() || shardingRule.findBindingTableRule(tableName).isPresent()) {
            //添加到TableToken
            selectStatement.getSqlTokens().add(new TableToken(position, literals));
        }
        
        result.append(lexerEngine.getCurrentToken().getLiterals());
        lexerEngine.nextToken();
        //列名
        result.append(lexerEngine.getCurrentToken().getLiterals());
        lexerEngine.nextToken();
    }
    return result.toString();
}

private String parseRestSelectItem(final SelectStatement selectStatement) {
    StringBuilder result = new StringBuilder();
    while (lexerEngine.equalAny(Symbol.getOperators())) {
        result.append(lexerEngine.getCurrentToken().getLiterals());
        lexerEngine.nextToken();
        result.append(parseCommonSelectItem(selectStatement));
    }
    return result.toString();
}
```

#### 解析FROM

```java
protected final void parseFrom(final SelectStatement selectStatement) {
    //不支持SELECT * INTO
    lexerEngine.unsupportedIfEqual(DefaultKeyword.INTO);
    if (lexerEngine.skipIfEqual(DefaultKeyword.FROM)) {
        parseTable(selectStatement);
    }
}


private void parseTable(final SelectStatement selectStatement) {
    //'('，解析子查询
    if (lexerEngine.skipIfEqual(Symbol.LEFT_PAREN)) {
        //子查询，在调用一次parseInternal
         selectStatement.setSubQueryStatement(parseInternal());
        //跳过WHERE 
         if (lexerEngine.equalAny(DefaultKeyword.WHERE, Assist.END)) {
            return;
         }
   }
    //解析表名。TableReferencesClauseParser已经分析过了
   selectClauseParserFacade.getTableReferencesClauseParser().parse(selectStatement, false);
}
```

#### 解析Where

```java
protected final void parseWhere(final ShardingRule shardingRule, final SelectStatement selectStatement, final List<SelectItem> items) {
    selectClauseParserFacade.getWhereClauseParser().parse(shardingRule, selectStatement, items);
}

public void parse(final ShardingRule shardingRule, final SQLStatement sqlStatement, final List<SelectItem> items) {
    //AliasClauseParser 别名解析器，分析过了
   aliasClauseParser.parse();
    //Where
   if (lexerEngine.skipIfEqual(DefaultKeyword.WHERE)) {
        parseConditions(shardingRule, sqlStatement, items);
        }
    }
```



```java
private void parseConditions(final ShardingRule shardingRule, final SQLStatement sqlStatement, final List<SelectItem> items) {
    do {
        parseComparisonCondition(shardingRule, sqlStatement, items);
    } while (lexerEngine.skipIfEqual(DefaultKeyword.AND));
    //不支持OR，3.X支持了。。。我这里源码是2.x的所以没有
    lexerEngine.unsupportedIfEqual(DefaultKeyword.OR);
}
```





```java
private void parseComparisonCondition(final ShardingRule shardingRule, final SQLStatement sqlStatement, final List<SelectItem> items) {
    //跳过'('
    lexerEngine.skipIfEqual(Symbol.LEFT_PAREN);
    //表达式解析器
    SQLExpression left = expressionClauseParser.parse(sqlStatement);
    //'='
    if (lexerEngine.skipIfEqual(Symbol.EQ)) {
        parseEqualCondition(shardingRule, sqlStatement, left);
        lexerEngine.skipIfEqual(Symbol.RIGHT_PAREN);
        return;
    }
    //IN
    if (lexerEngine.skipIfEqual(DefaultKeyword.IN)) {
        parseInCondition(shardingRule, sqlStatement, left);
        lexerEngine.skipIfEqual(Symbol.RIGHT_PAREN);
        return;
    }
    //BETWEEN
    if (lexerEngine.skipIfEqual(DefaultKeyword.BETWEEN)) {
        parseBetweenCondition(shardingRule, sqlStatement, left);
        lexerEngine.skipIfEqual(Symbol.RIGHT_PAREN);
        return;
    }
    //处理rowNumber
    if (sqlStatement instanceof SelectStatement && isRowNumberCondition(items, left)) {
        if (lexerEngine.skipIfEqual(Symbol.LT, Symbol.LT_EQ)) {
            parseRowCountCondition((SelectStatement) sqlStatement);
            return;
        }
        if (lexerEngine.skipIfEqual(Symbol.GT, Symbol.GT_EQ)) {
            parseOffsetCondition((SelectStatement) sqlStatement);
            return;
        }
    }
    //处理其他自定义条件，mysql就是REGEXP
    List<Keyword> otherConditionOperators = new LinkedList<>(Arrays.asList(getCustomizedOtherConditionOperators()));
    //'<'、'<='、'>'、'>='、'<>'、'!='、'!>'、'!<'、'LIKE'、'IS'
    otherConditionOperators.addAll(
            Arrays.asList(Symbol.LT, Symbol.LT_EQ, Symbol.GT, Symbol.GT_EQ, Symbol.LT_GT, Symbol.BANG_EQ, Symbol.BANG_GT, Symbol.BANG_LT, DefaultKeyword.LIKE, DefaultKeyword.IS));
    //处理条件
    if (lexerEngine.skipIfEqual(otherConditionOperators.toArray(new Keyword[otherConditionOperators.size()]))) {
        parseOtherCondition(sqlStatement);
    }
    //处理NOT
    if (lexerEngine.skipIfEqual(DefaultKeyword.NOT)) {
        lexerEngine.nextToken();
        lexerEngine.skipIfEqual(Symbol.LEFT_PAREN);
        parseOtherCondition(sqlStatement);
        lexerEngine.skipIfEqual(Symbol.RIGHT_PAREN);
    }
    lexerEngine.skipIfEqual(Symbol.RIGHT_PAREN);
}
```





#### 解析Group By

```java
public final void parse(final SelectStatement selectStatement) {
    if (!lexerEngine.skipIfEqual(DefaultKeyword.GROUP)) {
        return;
    }
    lexerEngine.accept(DefaultKeyword.BY);
    while (true) {
        addGroupByItem(expressionClauseParser.parse(selectStatement), selectStatement);
        //不是','，也就是不是多个就跳出
        if (!lexerEngine.equalAny(Symbol.COMMA)) {
            break;
        }
        //处理下一个
        lexerEngine.nextToken();
    }
    //WITH、ROLLUP
    lexerEngine.skipAll(getSkippedKeywordAfterGroupBy());
    selectStatement.setGroupByLastPosition(lexerEngine.getCurrentToken().getEndPosition() - lexerEngine.getCurrentToken().getLiterals().length());
}
```

关于ROLLUP https://dev.mysql.com/doc/refman/5.7/en/group-by-modifiers.html

解析Group By后面的东西

```java
private void addGroupByItem(final SQLExpression sqlExpression, final SelectStatement selectStatement) {
    //如果是Oracle不支持ROLLUP、CUBE、GROUPING
    lexerEngine.unsupportedIfEqual(getUnsupportedKeywordBeforeGroupByItem());
    OrderType orderByType = OrderType.ASC;
    //ASC,默认ASC
    if (lexerEngine.equalAny(DefaultKeyword.ASC)) {
        lexerEngine.nextToken();
    //DESC
    } else if (lexerEngine.skipIfEqual(DefaultKeyword.DESC)) {
        orderByType = OrderType.DESC;
    }
    OrderItem orderItem;
    if (sqlExpression instanceof SQLPropertyExpression) {
        SQLPropertyExpression sqlPropertyExpression = (SQLPropertyExpression) sqlExpression;
        orderItem = new OrderItem(SQLUtil.getExactlyValue(sqlPropertyExpression.getOwner().getName()), SQLUtil.getExactlyValue(sqlPropertyExpression.getName()), orderByType, OrderType.ASC,
                selectStatement.getAlias(SQLUtil.getExactlyValue(sqlPropertyExpression.getOwner() + "." + SQLUtil.getExactlyValue(sqlPropertyExpression.getName()))));
    } else if (sqlExpression instanceof SQLIdentifierExpression) {
        SQLIdentifierExpression sqlIdentifierExpression = (SQLIdentifierExpression) sqlExpression;
        orderItem = new OrderItem(
                SQLUtil.getExactlyValue(sqlIdentifierExpression.getName()), orderByType, OrderType.ASC, selectStatement.getAlias(SQLUtil.getExactlyValue(sqlIdentifierExpression.getName())));
    } else if (sqlExpression instanceof SQLIgnoreExpression) {
        SQLIgnoreExpression sqlIgnoreExpression = (SQLIgnoreExpression) sqlExpression;
        orderItem = new OrderItem(sqlIgnoreExpression.getExpression(), orderByType, OrderType.ASC, selectStatement.getAlias(sqlIgnoreExpression.getExpression()));
    } else {
        return;
    }
    selectStatement.getGroupByItems().add(orderItem);
}
```

#### 合并子查询

```java
public SelectStatement mergeSubQueryStatement() {
    SelectStatement result = processLimitForSubQuery();
    processItems(result);
    processOrderByItems(result);
    return result;
}
```