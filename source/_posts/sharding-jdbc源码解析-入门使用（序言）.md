---
title: sharding-jdbc源码解析-入门使用（序言）
tags:
  - sharding-jdbc
  - sharding-sphere
categories:
  - 中间件
  - 数据库
abbrlink: 1442ae52
date: 2018-05-05 21:04:06
---

## 简介

Sharding-JDBC是一个开源的分布式数据库中间件解决方案。它在Java的JDBC层以对业务应用零侵入的方式额外提供数据分片，读写分离，柔性事务和分布式治理能力。并在其基础上提供封装了MySQL协议的服务端版本，用于完成对异构语言的支持。

(还没写完这个系列的时候就已经更名成sharding-sphere了)

官方文档：http://shardingjdbc.io/docs_cn/00-overview/

http://shardingsphere.io/document/cn/overview/

官方github地址：https://github.com/shardingjdbc/sharding-jdbc

https://github.com/sharding-sphere/sharding-sphere

### 功能列表

#### 1. 数据分片

- 支持分库 + 分表
- 支持聚合，分组，排序，分页，关联查询等复杂查询语句
- 支持常见的DML，DDL，TCL以及数据库管理语句
- 支持=，BETWEEN，IN的分片操作符
- 自定义的灵活分片策略，支持多分片键共用，支持inline表达式
- 基于Hint的强制路由
- 支持分布式主键

#### 2. 读写分离

- 支持一主多从的读写分离
- 支持同一线程内的数据一致性
- 支持分库分表与读写分离共同使用
- 支持基于Hint的强制主库路由

#### 3. 柔性事务

- 最大努力送达型事务
- TCC型事务(TBD)

#### 4. 分布式治理

- 支持配置中心，可动态修改配置
- 支持客户端熔断和失效转移
- 支持Open Tracing协议

<!-- more -->

### 部署架构

#### Sharding-JDBC-Driver

通过客户端分片的方式由应用程序直连数据库，减少二次转发成本，性能最高，适合线上程序使用。

- 可适用于任何基于Java的ORM框架，如：JPA, Hibernate, Mybatis, Spring JDBC Template或直接使用JDBC。

- 可基于任何第三方的数据库连接池，如：DBCP, C3P0, BoneCP, Druid等。

- 可支持任意实现JDBC规范的数据库。目前支持MySQL，Oracle，SQLServer和PostgreSQL。


![image](https://user-images.githubusercontent.com/7789698/39663686-51e91372-50aa-11e8-96b5-d383f1b02f55.png)

#### Sharding-JDBC-Server

通过代理服务器连接数据库(目前仅支持MySQL)，适合其他开发语言或MySQL客户端操作数据。

- 向应用程序完全透明，可直接当做MySQL使用。
- 可适用于任何兼容MySQL协议的的客户端。


![image](https://user-images.githubusercontent.com/7789698/39663693-8c4d6266-50aa-11e8-939a-e8779641a35b.png)

#### Sharding-JDBC-Sidecar(TBD)

通过sidecar分片的方式，由IPC代替RPC，自动代理SQL分片，适合与Kubernetes或Mesos配合使用。

![image](https://user-images.githubusercontent.com/7789698/39663709-c6aa7d68-50aa-11e8-8052-b3b86afba760.png)

### 快速入门

#### 读写分离

为了缓解数据库压力，将写入和读取操作分离为不同数据源，写库称为主库，读库称为从库，一主库可配置多从库。

一主两从，读写分离。

```sql
CREATE SCHEMA IF NOT EXISTS demo_ds_master;
CREATE SCHEMA IF NOT EXISTS demo_ds_slave_0;
CREATE SCHEMA IF NOT EXISTS demo_ds_slave_1;
```



```sql
CREATE TABLE `t_order` (
  `order_id` bigint(20) NOT NULL AUTO_INCREMENT,
  `user_id` int(11) NOT NULL,
  `status` varchar(50) DEFAULT NULL,
  PRIMARY KEY (`order_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `t_order_item` (
  `item_id` bigint(20) NOT NULL AUTO_INCREMENT,
  `order_id` bigint(20) NOT NULL,
  `user_id` int(11) NOT NULL,
  PRIMARY KEY (`item_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

java:

```java
private static DataSource getMasterSlaveDataSource() throws SQLException {
    MasterSlaveRuleConfiguration masterSlaveRuleConfig = new MasterSlaveRuleConfiguration();
    masterSlaveRuleConfig.setName("demo_ds_master_slave");
    //主数据源
    masterSlaveRuleConfig.setMasterDataSourceName("demo_ds_master");
    //从数据源
    masterSlaveRuleConfig.setSlaveDataSourceNames(Arrays.asList("demo_ds_slave_0", "demo_ds_slave_1"));
    //随机
    masterSlaveRuleConfig	.setLoadBalanceAlgorithmType(MasterSlaveLoadBalanceAlgorithmType.RANDOM);      
    return MasterSlaveDataSourceFactory.createDataSource(createDataSourceMap(), masterSlaveRuleConfig);
}

private static Map<String, DataSource> createDataSourceMap() {
    final Map<String, DataSource> result = new HashMap<>(3, 1);
    result.put("demo_ds_master", DataSourceUtil.createDataSource("demo_ds_master"));
    result.put("demo_ds_slave_0", DataSourceUtil.createDataSource("demo_ds_slave_0"));
    result.put("demo_ds_slave_1", DataSourceUtil.createDataSource("demo_ds_slave_1"));
    return result;
}
```

spring:

```xml
<context:component-scan base-package="io.shardingjdbc.example.spring.namespace.mybatis" />

<bean id="demo_ds_master" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/demo_ds_master"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</bean>

<bean id="demo_ds_slave_0" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/demo_ds_slave_0"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</bean>

<bean id="demo_ds_slave_1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/demo_ds_slave_1"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</bean>

<bean id="randomStrategy" class="io.shardingjdbc.core.api.algorithm.masterslave.RandomMasterSlaveLoadBalanceAlgorithm" />

<master-slave:data-source id="masterSlaveDataSource" master-data-source-name="demo_ds_master" slave-data-source-names="demo_ds_slave_0, demo_ds_slave_1" strategy-ref="randomStrategy" />
```

yaml:

```yaml
dataSources:
  ds_master: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/demo_ds_master
    username: root
    password: 123456
  ds_slave_0: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/demo_ds_slave_0
    username: root
    password: 123456
  ds_slave_1: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/demo_ds_slave_1
    username: root
    password: 123456

masterSlaveRule:
  name: 
    ds_ms
  masterDataSourceName:
    ds_master
  slaveDataSourceNames: [ds_slave_0, ds_slave_1]
```

springboot:

```properties
sharding.jdbc.datasource.names=ds_master,ds_slave_0,ds_slave_1

sharding.jdbc.datasource.ds_master.type=org.apache.commons.dbcp.BasicDataSource
sharding.jdbc.datasource.ds_master.driverClassName=com.mysql.jdbc.Driver
sharding.jdbc.datasource.ds_master.url=jdbc:mysql://localhost:3306/demo_ds_master
sharding.jdbc.datasource.ds_master.username=root
sharding.jdbc.datasource.ds_master.password=123456

sharding.jdbc.datasource.ds_slave_0.type=org.apache.commons.dbcp.BasicDataSource
sharding.jdbc.datasource.ds_slave_0.driverClassName=com.mysql.jdbc.Driver
sharding.jdbc.datasource.ds_slave_0.url=jdbc:mysql://localhost:3306/demo_ds_slave_0
sharding.jdbc.datasource.ds_slave_0.username=root
sharding.jdbc.datasource.ds_slave_0.password=123456

sharding.jdbc.datasource.ds_slave_1.type=org.apache.commons.dbcp.BasicDataSource
sharding.jdbc.datasource.ds_slave_1.driverClassName=com.mysql.jdbc.Driver
sharding.jdbc.datasource.ds_slave_1.url=jdbc:mysql://localhost:3306/demo_ds_slave_1
sharding.jdbc.datasource.ds_slave_1.username=root
sharding.jdbc.datasource.ds_slave_1.password=123456

sharding.jdbc.config.masterslave.load-balance-algorithm-type=round_robin
sharding.jdbc.config.masterslave.name=ds_ms
sharding.jdbc.config.masterslave.master-data-source-name=ds_master
sharding.jdbc.config.masterslave.slave-data-source-names=ds_slave_0,ds_slave_1
```

#### 分库分表

- 数据源分片策略DatabaseShardingStrategy：数据被分配的目标数据源

- 表分片策略TableShardingStrategy：数据被分配的目标表，该目标表存在与该数据的目标数据源内

- 分片键  分片键是分片策略的第一个参数。分片键表示的是SQL语句中WHERE中的条件列。分片键可以配置多个。

- 分片算法

  Sharding-JDBC提供了5种分片策略。由于分片算法和业务实现紧密相关，因此Sharding-JDBC并未提供内置分片算法，而是通过分片策略将各种场景提炼出来，提供更高层级的抽象，并提供接口让应用开发者自行实现分片算法。

  - StandardShardingStrategy

  标准分片策略。提供对SQL语句中的=, IN和BETWEEN AND的分片操作支持。StandardShardingStrategy**只支持单分片键**，提供**PreciseShardingAlgorithm**和**RangeShardingAlgorithm**两个分片算法。PreciseShardingAlgorithm是必选的，用于处理**=和IN**的分片。RangeShardingAlgorithm是可选的，用于处理**BETWEEN AND**分片，如果不配置RangeShardingAlgorithm，SQL中的BETWEEN AND将按照全库路由处理。

  - ComplexShardingStrategy

  复合分片策略。提供对SQL语句中的=, IN和BETWEEN AND的分片操作支持。ComplexShardingStrategy支持多分片键，由于多分片键之间的关系复杂，因此Sharding-JDBC并未做过多的封装，而是直接将分片键值组合以及分片操作符交于算法接口，完全由应用开发者实现，提供最大的灵活度。

  - InlineShardingStrategy

  Inline**表达式**分片策略。使用**Groovy**的Inline表达式，提供对SQL语句中的=和IN的分片操作支持。InlineShardingStrategy只支持单分片键，对于简单的分片算法，可以通过简单的配置使用，从而避免繁琐的Java代码开发，如: t*user*${user_id % 8} 表示t_user表按照user_id按8取模分成8个表，表名称为t_user_0到t_user_7。

  - HintShardingStrategy

  通过Hint而非SQL解析的方式分片的策略。

  - NoneShardingStrategy

  不分片的策略。

- 级联绑定表

  级联绑定表代表一组表，这组表的逻辑表与实际表之间的映射关系是相同的。比如t_order与t_order_item就是这样一组绑定表关系，它们的分库与分表策略是完全相同的,那么可以使用它们的表规则将它们配置成级联绑定表。

  那么在进行SQL路由时，如果SQL为：

  ```sql
  SELECT i.* FROM t_order o JOIN t_order_item i ON o.order_id=i.order_id WHERE o.user_id=? AND o.order_id=?
  ```

  其中t_order在FROM的最左侧，Sharding-JDBC将会以它作为整个绑定表的主表。所有路由计算将会只使用主表的策略，那么t_order_item表的分片计算将会使用t_order的条件。故绑定表之间的分区键要完全相同。

```java
private static DataSource getShardingDataSource() throws SQLException {
    ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
    shardingRuleConfig.getTableRuleConfigs().add(getOrderTableRuleConfiguration());
    shardingRuleConfig.getTableRuleConfigs().add(getOrderItemTableRuleConfiguration());
    shardingRuleConfig.getBindingTableGroups().add("t_order, t_order_item");
    //按user_id模2分库，demo_ds_0、demo_ds_1
    shardingRuleConfig.setDefaultDatabaseShardingStrategyConfig(new InlineShardingStrategyConfiguration("user_id", "demo_ds_${user_id % 2}"));
    //按order_id模2分表，实现PreciseShardingAlgorithm
    shardingRuleConfig.setDefaultTableShardingStrategyConfig(new StandardShardingStrategyConfiguration("order_id", ModuloShardingTableAlgorithm.class.getName()));
    //构造ShardingDataSource
    return ShardingDataSourceFactory.createDataSource(createDataSourceMap(), shardingRuleConfig);
}

private static TableRuleConfiguration getOrderTableRuleConfiguration() {
    TableRuleConfiguration orderTableRuleConfig = new TableRuleConfiguration();
    orderTableRuleConfig.setLogicTable("t_order");
    orderTableRuleConfig.setActualDataNodes("demo_ds_${0..1}.t_order_${[0, 1]}");
    //配置自增列
    orderTableRuleConfig.setKeyGeneratorColumnName("order_id");
    return orderTableRuleConfig;
}

private static TableRuleConfiguration getOrderItemTableRuleConfiguration() {
    TableRuleConfiguration orderItemTableRuleConfig = new TableRuleConfiguration();
    orderItemTableRuleConfig.setLogicTable("t_order_item");
    orderItemTableRuleConfig.setActualDataNodes("demo_ds_${0..1}.t_order_item_${[0, 1]}");
    return orderItemTableRuleConfig;
}

private static Map<String, DataSource> createDataSourceMap() {
    Map<String, DataSource> result = new HashMap<>(2, 1);
    result.put("demo_ds_0", DataSourceUtil.createDataSource("demo_ds_0"));
    result.put("demo_ds_1", DataSourceUtil.createDataSource("demo_ds_1"));
    return result;
}
```

```java
public final class ModuloShardingTableAlgorithm implements PreciseShardingAlgorithm<Long> {
    
    @Override
    public String doSharding(final Collection<String> tableNames, final PreciseShardingValue<Long> shardingValue) {
        for (String each : tableNames) {
            //分片键
            if (each.endsWith(shardingValue.getValue() % 2 + "")) {
                return each;
            }
        }
        throw new UnsupportedOperationException();
    }
}
```


想要同时读写分离，只要加入

```java
shardingRuleConfig.setMasterSlaveRuleConfigs(
	Lists.newArrayList(masterSlaveRuleConfig1, masterSlaveRuleConfig2)
);
```

spring:

```xml
<context:component-scan base-package="io.shardingjdbc.example.spring.namespace.mybatis" />

<bean id="demo_ds_0" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/demo_ds_0"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</bean>

<bean id="demo_ds_1" class="org.apache.commons.dbcp.BasicDataSource" destroy-method="close">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/demo_ds_1"/>
    <property name="username" value="root"/>
    <property name="password" value="123456"/>
</bean>

<sharding:standard-strategy id="databaseShardingStrategy" sharding-column="user_id" precise-algorithm-class="io.shardingjdbc.example.spring.namespace.mybatis.algorithm.PreciseModuloDatabaseShardingAlgorithm"/>
<sharding:standard-strategy id="tableShardingStrategy" sharding-column="order_id" precise-algorithm-class="io.shardingjdbc.example.spring.namespace.mybatis.algorithm.PreciseModuloTableShardingAlgorithm"/>

<sharding:data-source id="shardingDataSource">
    <sharding:sharding-rule data-source-names="demo_ds_0, demo_ds_1">
        <sharding:table-rules>
            <sharding:table-rule logic-table="t_order" actual-data-nodes="demo_ds_${0..1}.t_order_${0..1}" database-strategy-ref="databaseShardingStrategy" table-strategy-ref="tableShardingStrategy" generate-key-column="order_id" />
            <sharding:table-rule logic-table="t_order_item" actual-data-nodes="demo_ds_${0..1}.t_order_item_${0..1}" database-strategy-ref="databaseShardingStrategy" table-strategy-ref="tableShardingStrategy" generate-key-column="order_item_id" />
        </sharding:table-rules>
    </sharding:sharding-rule>
</sharding:data-source>
```

#### 分片管理器

com.dangdang.ddframe.rdb.sharding.api.HintManager使用ThreadLocal技术管理分片键值。

- 使用hintManager.addDatabaseShardingValue来添加数据源分片键值
- 使用hintManager.addTableShardingValue来添加表分片键值
- 分片键值保存在ThreadLocal中，所以需要在操作结束时调用hintManager.close()来清除ThreadLocal中的内容。

```java
String sql = "SELECT * FROM t_order";
        
try (
        HintManager hintManager = HintManager.getInstance();
        Connection conn = dataSource.getConnection();
        PreparedStatement preparedStatement = conn.prepareStatement(sql)) {
    hintManager.addDatabaseShardingValue("t_order", "user_id", 1);
    hintManager.addTableShardingValue("t_order", "order_id", 2);
    try (ResultSet rs = preparedStatement.executeQuery()) {
        while (rs.next()) {
            ...
        }
    }
}
```

#### 分布式主键

##### 设置自动生成键

- 配置自增列


```java
 TableRuleConfiguration tableRuleConfig = new TableRuleConfiguration();
  tableRuleConfig.setLogicTable("t_order");
  tableRuleConfig.setKeyGeneratorColumnName("order_id");
```

- 设置Id生成器的实现类

  该类必须实现io.shardingjdbc.core.keygen.KeyGenerator接口。

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

配置全局生成器(com.xx.xx.KeyGenerator):

```java
ShardingRuleConfiguration shardingRuleConfig = new ShardingRuleConfiguration();
shardingRuleConfig.setDefaultKeyGeneratorClass("com.xx.xx.KeyGenerator");
```

#### 事务支持

BASE理论

（1）BA：Basic Available 基本可用

整个系统在某些不可抗力的情况下，仍然能够保证“可用性”，即一定时间内仍然能够返回一个明确的结果。只不过“基本可用”和“高可用”的区别是：

- “一定时间”可以适当延长：当举行大促时，响应时间可以适当延长
- 给部分用户返回一个降级页面：给部分用户直接返回一个降级页面，从而缓解服务器压力。但要注意，返回降级页面仍然是返回明确结果。

（2）S：Soft State：柔性状态

同一数据的不同副本的状态，可以不需要实时一致。

（3）E：Eventual Consisstency：最终一致性

同一数据的不同副本的状态，可以不需要实时一致，但一定要保证经过一定时间后仍然是一致的。

##### 最大努力送达型

适用场景

- 根据主键删除数据。
- 更新记录永久状态，如更新通知送达状态。

使用限制

使用最大努力送达型柔性事务的SQL需要满足幂等性。

- INSERT语句要求必须包含主键，且不能是自增主键。
- UPDATE语句要求幂等，不能是UPDATE xxx SET x=x+1
- DELETE语句无要求。

独立部署作业指南

- 部署用于存储事务日志的数据库。
- 部署用于异步作业使用的Zookeeper。
- 配置YAML文件,参照示例。
- 下载并解压文件sharding-jdbc-transaction-async-job-$VERSION.tar，通过start.sh脚本启动异步作业。

![image](https://user-images.githubusercontent.com/7789698/39670535-9ee904f4-5139-11e8-9d60-e485f3c50f54.png)



```java
	// 1. 配置SoftTransactionConfiguration	
	SoftTransactionConfiguration transactionConfig = new SoftTransactionConfiguration(dataSource);
    transactionConfig.setXXX();
             
    // 2. 初始化SoftTransactionManager
	SoftTransactionManager transactionManager = new SoftTransactionManager(transactionConfig);
	transactionManager.init();
             
	// 3. 获取BEDSoftTransaction
	BEDSoftTransaction transaction = (BEDSoftTransaction) transactionManager.getTransaction(SoftTransactionType.BestEffortsDelivery);
             
	// 4. 开启事务
	transaction.begin(connection);
             
            // 5. 执行JDBC
            /* 
                codes here
            */

	// 6.关闭事务
	transaction.end();
```

异步作业YAML文件配置

```properties
# 目标数据库的数据源.
targetDataSource:
  ds_0: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds_0
    username: root
    password:
  ds_1: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/ds_1
    username: root
    password: 123456

# 事务日志的数据源.
transactionLogDataSource:
  ds_trans: !!org.apache.commons.dbcp.BasicDataSource
    driverClassName: com.mysql.jdbc.Driver
    url: jdbc:mysql://localhost:3306/trans_log
    username: root
    password: 123456
    

#注册中心配置
zkConfig:
  #注册中心的连接地址
  connectionString: localhost:2181
  #作业的命名空间
  namespace: Best-Efforts-Delivery-Job
  #注册中心的等待重试的间隔时间的初始值
  baseSleepTimeMilliseconds: 1000
  #注册中心的等待重试的间隔时间的最大值
  maxSleepTimeMilliseconds: 3000
  #注册中心的最大重试次数
  maxRetries: 3

#作业配置
jobConfig:
  #作业名称
  name: bestEffortsDeliveryJob
  #触发作业的cron表达式
  cron: 0/5 * * * * ?
  #每次作业获取的事务日志最大数量
  transactionLogFetchDataCount: 100
  #事务送达的最大尝试次数.
  maxDeliveryTryTimes: 3
  #执行送达事务的延迟毫秒数,早于此间隔时间的入库事务才会被作业执行
  maxDeliveryTryDelayMillis: 60000
```



##### SoftTransactionConfiguration配置

用于配置事务管理器。

| *名称*                              | *类型*                                    | *必填* | *默认值* | *说明*                                                       |
| ----------------------------------- | ----------------------------------------- | ------ | -------- | ------------------------------------------------------------ |
| shardingDataSource                  | ShardingDataSource                        | 是     |          | 事务管理器管理的数据源                                       |
| syncMaxDeliveryTryTimes             | int                                       | 否     | 3        | 同步的事务送达的最大尝试次数                                 |
| storageType                         | enum                                      | 否     | RDB      | 事务日志存储类型。可选值: RDB,MEMORY。使用RDB类型将自动建表  |
| transactionLogDataSource            | DataSource                                | 否     | null     | 存储事务日志的数据源，如果storageType为RDB则必填             |
| bestEffortsDeliveryJobConfiguration | NestedBestEffortsDeliveryJobConfiguration | 否     | null     | 最大努力送达型内嵌异步作业配置对象。如需使用，请参考NestedBestEffortsDeliveryJobConfiguration配置 |

##### NestedBestEffortsDeliveryJobConfiguration配置 (仅开发环境)

用于配置内嵌的异步作业，仅用于开发环境。生产环境应使用独立部署的作业版本。

| *名称*                         | *类型* | *必填* | *默认值*                  | *说明*                                                       |
| ------------------------------ | ------ | ------ | ------------------------- | ------------------------------------------------------------ |
| zookeeperPort                  | int    | 否     | 4181                      | 内嵌的注册中心端口号                                         |
| zookeeperDataDir               | String | 否     | target/test_zk_data/nano/ | 内嵌的注册中心的数据存放目录                                 |
| asyncMaxDeliveryTryTimes       | int    | 否     | 3                         | 异步的事务送达的最大尝试次数                                 |
| asyncMaxDeliveryTryDelayMillis | long   | 否     | 60000                     | 执行异步送达事务的延迟毫秒数，早于此间隔时间的入库事务才会被异步作业执行 |

