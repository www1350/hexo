---
title: mongodb
tags: mongodb
categories: 数据库
abbrlink: effa6b88
date: 2016-07-04 22:03:26
---

 http://www.mongodb.org/downloads 

mongod.exe --dbpath

创建数据库

```
use DATABASE_NAME
```

删除数据库

```
db.dropDatabase()
```

创建集合

```
db.createCollection(name, options)
```

options:
项​​参数是可选的，所以只需要到指定的集合名称。以下是可以使用的选项列表：

字段  类型  描述
capped  Boolean （可选）如果为true，则启用封顶集合。封顶集合是固定大小的集合，会自动覆盖最早的条目，当它达到其最大大小。如果指定true，则需要也指定尺寸参数。
autoIndexID Boolean （可选）如果为true，自动创建索引_id字段的默认值是false。
size    number  （可选）指定最大大小字节封顶集合。如果封顶如果是 true，那么你还需要指定这个字段。
max number  （可选）指定封顶集合允许在文件的最大数量。

删除集合

```
db.COLLECTION_NAME.drop()
```

数据类型：
- String : 这是最常用的数据类型来存储数据。在MongoDB中的字符串必须是有效的UTF-8。
- Integer : 这种类型是用来存储一个数值。整数可以是32位或64位，这取决于您的服务器。
- Boolean : 此类型用于存储一个布尔值 (true/ false) 。
- Double : 这种类型是用来存储浮点值。
- Min/ Max keys : 这种类型被用来对BSON元素的最低和最高值比较。
- Arrays : 使用此类型的数组或列表或多个值存储到一个键。
- Timestamp : 时间戳。这可以方便记录时的文件已被修改或添加。
- Object : 此数据类型用于嵌入式的文件。
- Null : 这种类型是用来存储一个Null值。
- Symbol : 此数据类型用于字符串相同，但它通常是保留给特定符号类型的语言使用。
- Date : 此数据类型用于存储当前日期或时间的UNIX时间格式。可以指定自己的日期和时间，日期和年，月，日到创建对象。
- Object ID : 此数据类型用于存储文档的ID。
- Binary data : 此数据类型用于存储二进制数据。
- Code : 此数据类型用于存储到文档中的JavaScript代码。
- Regular expression : 此数据类型用于存储正则表达式

基本操作：

```
db.COLLECTION_NAME.insert(document)
db.COLLECTION_NAME.find()

db.mycol.find().pretty()//格式化
db.mycol.find({key1:value1, key2:value2}).pretty()
db.mycol.find(
   {
      $or: [
         {key1: value1}, {key2:value2}
      ]
   }
).pretty()

db.mycol.find("likes": {$gt:10}, $or: [{"by": "yiibai"}, {"title": "MongoDB Overview"}] }).pretty()
// 'where likes>10 AND (by = 'yiibai' OR title = 'MongoDB Overview')'
db.COLLECTION_NAME.update(SELECTIOIN_CRITERIA, UPDATED_DATA)
db.COLLECTION_NAME.save({_id:ObjectId(),NEW_DATA})
db.COLLECTION_NAME.remove(DELLETION_CRITTERIA)
db.COLLECTION_NAME.remove(DELETION_CRITERIA,1)
db.mycol.find({},{"title":1,_id:0})//1表示显示该字段0不显示
db.COLLECTION_NAME.find().limit(NUMBER)
db.COLLECTION_NAME.find().limit(NUMBER).skip(NUMBER)
db.COLLECTION_NAME.find().sort({KEY:1})

db.COLLECTION_NAME.aggregate(AGGREGATE_OPERATION)
```

操作  语法  例子  RDBMS 等同
Equality    {<key>:<value>} db.mycol.find({"by":"tutorials yiibai"}).pretty()   where by = 'tutorials yiibai'
Less Than   {<key>:{$lt:<value>}}   db.mycol.find({"likes":{$lt:50}}).pretty()  where likes < 50
Less Than Equals    {<key>:{$lte:<value>}}  db.mycol.find({"likes":{$lte:50}}).pretty() where likes <= 50
Greater Than    {<key>:{$gt:<value>}}   db.mycol.find({"likes":{$gt:50}}).pretty()  where likes > 50
Greater Than Equals {<key>:{$gte:<value>}}  db.mycol.find({"likes":{$gte:50}}).pretty() where likes >= 50
Not Equals  {<key>:{$ne:<value>}}   db.mycol.find({"likes":{$ne:50}}).pretty()  where likes != 50

```
db.COLLECTION_NAME.ensureIndex({KEY:1})//索引
```

ensureIndex() 方法也可以接受的选项列表（可选），其下面给出的列表：

参数  类型  描述
background  Boolean 在后台建立索引，以便建立索引并不能阻止其他数据库活动。指定true建立在后台。默认值是 false.
unique  Boolean 创建唯一索引，以便收集不会接受插入索引键或键匹配现有的值存储在索引文档。指定创建唯一索引。默认值是 false.
name    string  索引的名称。如果未指定，MongoDB中都生成一个索引名索引字段的名称和排序顺序串联.
dropDups    Boolean 创建一个唯一索引的字段，可能有重复。 MongoDB的索引只有第一次出现的一个键，从集合中删除的所有文件包含该键的后续出现的。指定创建唯一索引。默认值是 false.
sparse  Boolean 如果为true，指数只引用文档指定的字段。这些索引使用更少的空间，但在某些情况下，特别是各种不同的表现。默认值是 false.
expireAfterSeconds  integer 指定一个值，以秒为TTL控制多久MongoDB的文档保留在此集合.
v   index version   索引版本号。默认的索引版本取决于mongodb 运行的版本在创建索引时.
weights document    权重是从1到99999范围内的数，表示该字段的意义，相对于其他的索引字段分数.
default_language    string  对于文本索引时，决定停止词和词干分析器和标记生成规则列表的语言。默认值是 english.
language_override   string  对于文本索引时，指定的名称在文档中包含覆盖默认的语言，语言字段中。默认值是语言。

```
db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$sum : 1}}}])
```

表达式   描述  实例
$sum    总结从集合中的所有文件所定义的值.   db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$sum : "$likes"}}}])
$avg    从所有文档集合中所有给定值计算的平均. db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$avg : "$likes"}}}])
$min    获取集合中的所有文件中的相应值最小.    db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$min : "$likes"}}}])
$max    获取集合中的所有文件中的相应值的最大. db.mycol.aggregate([{$group : {_id : "$by_user", num_tutorial : {$max : "$likes"}}}])
$push   值插入到一个数组生成文档中.    db.mycol.aggregate([{$group : {_id : "$by_user", url : {$push: "$url"}}}])
$addToSet   值插入到一个数组中所得到的文档，但不会创建重复.  db.mycol.aggregate([{$group : {_id : "$by_user", url : {$addToSet : "$url"}}}])
$first  根据分组从源文档中获取的第一个文档。通常情况下，这才有意义，连同以前的一些应用 “$sort”-stage.    db.mycol.aggregate([{$group : {_id : "$by_user", first_url : {$first : "$url"}}}])
$last   根据分组从源文档中获取最后的文档。通常，这才有意义，连同以前的一些应用 “$sort”-stage.    db.mycol.aggregate([{$group : {_id : "$by_user", last_url : {$last : "$url"}}}])

设置一个副本集
在本教程中，我们将mongod实例转换成独立的副本集。要转换到副本设置遵循以下步骤：

关闭停止已经运行的MongoDB服务器。
现在启动MongoDB服务器通过指定  --replSet 选项。 --replSet 基本语法如下：

```
mongod --port "PORT" --dbpath "YOUR_DB_DATA_PATH" --replSet 
```

"REPLICA_SET_INSTANCE_NAME"
例子

```
mongod --port 27017 --dbpath "D:set upmongodbdata" --replSet rs0
```

它会启动一个mongod 实例名称rs0 ，端口为27017。启动命令提示符 rs.initiate()，并连接到这个mongod实例。在mongod客户端执行命令rs.initiate()启动一个新的副本集。要检查副本集的配置执行命令rs.conf()。要检查的状态副本sete执行命令：rs.status()。

将成员添加到副本集
将成员添加到副本集，在多台机器上启动mongod 实例。现在开始一个mongod 客户和发出命令 rs.add().

```
rs.add("mongod1.net:27017")
```

java
建立连接

```
import com.mongodb.MongoClient;
import com.mongodb.MongoException;
import com.mongodb.WriteConcern;
import com.mongodb.DB;
import com.mongodb.DBCollection;
import com.mongodb.BasicDBObject;
import com.mongodb.DBObject;
import com.mongodb.DBCursor;
import com.mongodb.ServerAddress;

import java.util.Arrays;

// To connect to mongodb server
MongoClient mongoClient = new MongoClient( "localhost" , 27017 );
// Now connect to your databases
DB db = mongoClient.getDB( "test" );
boolean auth = db.authenticate(myUserName, myPassword);
```

获取一个集合列表

```
Set colls = db.getCollectionNames();
for (String s : colls) {
   System.out.println(s);
}
```

获取/选择一个集合

```
DBCollection coll = db.getCollection("mycol");
```

插入文档

```
BasicDBObject doc = new BasicDBObject("title", "MongoDB").
   append("description", "database").
   append("likes", 100).
   append("url", "http://www.yiibai.com/mongodb/").
   append("by", "yiibai.com").
   ;
coll.insert(doc);
```

查找第一个文档

```
DBObject myDoc = coll.findOne();
System.out.println(myDoc);
```

和spring配合

```
<mongo:mongo id="mongo" replica-set="${mongodb.url}">
    <mongo:options connections-per-host="${mongo.connectionsPerHost}"
            threads-allowed-to-block-for-connection-multiplier="${mongo.threadsAllowedToBlockForConnectionMultiplier}"
            connect-timeout="${mongo.connectTimeout}"
            max-wait-time="${mongo.maxWaitTime}"
            auto-connect-retry="${mongo.autoConnectRetry}"
            socket-keep-alive="${mongo.socketKeepAlive}"
            socket-timeout="${mongo.socketTimeout}" slave-ok="${mongo.slaveOk}"
            write-number="1" write-timeout="0" write-fsync="true" />
        <mongo:options write-number="1" write-timeout="0"
            write-fsync="true" />
    </mongo:mongo>
<mongo:db-factory dbname="${mongodb.dbname}" mongo-ref="mongo" />
    <bean id="mongoTemplate" class="org.springframework.data.mongodb.core.MongoTemplate">
        <constructor-arg name="mongoDbFactory" ref="mongoDbFactory" />
    </bean>
```

使用 MongoTemplate

```
  Person p = new Person("Joe", 34);

  // Insert is used to initially store the object into the database.
  mongoOps.insert(p);
  log.info("Insert: " + p);

  // Find
  p = mongoOps.findById(p.getId(), Person.class);   
  log.info("Found: " + p);

  // Update
  mongoOps.updateFirst(query(where("name").is("Joe")), update("age", 35), Person.class);    
  p = mongoOps.findOne(query(where("name").is("Joe")), Person.class);
  log.info("Updated: " + p);

  // Delete
  mongoOps.remove(p);

  // Check that deletion worked
  List<Person> people =  mongoOps.findAll(Person.class);
  log.info("Number of people = : " + people.size());

  mongoOps.dropCollection(Person.class);
```

```
Query query=new Query(
    Criteria.where("AAA").is(XXobj.getAAA()).
    orOperator(Criteria.where("BBB").is(XXobj.getBBB()))
    );
List<XXObject> result = mongoTemplate.find(query, XXObject.class);
if(result!=null && !result.isEmpty()){
    return result.get(0);
}

XXObject obj = mongoTemplate.findOne(query, XXObject.class);
if(obj!=null){
    return obj;
}

```