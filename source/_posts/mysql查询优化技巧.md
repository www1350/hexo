---
title: mysql查询优化技巧
abbrlink: c2da5a2b
date: 2018-04-03 22:39:01
tags:
categories:
---

http://blog.jobbole.com/24006/

# 索引区分度

> 区分度: 指字段在数据库中的不重复比
> 区分度在新建索引时有着非常重要的参考价值,在MySQL中,区分度的计算规则如下:

> 字段去重后的总数与全表总记录数的商。

例如:

`select count(distinct(name))/count(*) from t_base_user;`

count(distinct(name))/count(*)
--
1.0000

其中区分度最大值为1.000,最小为0.0000,区分度的值越大,也就是数据不重复率越大，新建索引效果也越好,在主键以及唯一键上面的区分度是最高的,为1.0000,在状态,性别等字段上面的区分度值是最小的。 (这个就要看数据量了,如果只有几条数据,这时区分度还挺高的,如果数据量多,区分度基本为0.0000。也就是在这些字段上添加索引后,效果也不佳的原因。)

值得注意的是: 如果表中没有任何记录时,计算区分度的结果是为空值，其他情况下,区分度值均分布在0.0000-1.0000之间。

个人强烈建议, 建索引时,一定要先计算该字段的区分度,原因如下:

1、单列索引

可以查看该字段的区分度,根据区分度的大小,也能大概知道在该字段上的新建索引是否有效，以及效果如何。区分度越大,索引效果越明显。

2、多列索引(联合索引)

多列索引中其实还有一个字段的先后顺序问题,一般是将区分度较高的放在前面,这样联合索引才更有效,例如:

`select * from t_base_user where name="" and status=1;`

像上述语句,如果建联合索引的话,就应该是:

`alter table t_base_user add index idx_name_status(name,status);`

而不是:

`alter table t_base_user add index idx_status_name(status,name)；`

# 最左前缀匹配原则
  MySQL会一直向右匹配直到遇到范围查询(>、<、between、like)就停止匹配，比如
`select * from t_base_user where type="10" and created_at<"2017-11-03" and status=1`, (该语句仅作为演示)

在上述语句中,status就不会走索引,因为遇到<时,MySQL已经停止匹配,此时走的索引为:(type,created_at),其先后顺序是可以调整的,而走不到status索引,此时需要修改语句为:

`select * from t_base_user where type=10 and status=1 and created_at<"2017-11-03"`

举例：
CREATE TABLE `titles` (
  `id` varchar(50) NOT NULL DEFAULT '',
  `emp_no` varchar(50) NOT NULL DEFAULT '',
  `title` varchar(50) NOT NULL DEFAULT '',
  `from_date` datetime DEFAULT NULL COMMENT 'from',
  `to_date` datetime DEFAULT NULL,
  `date_create` datetime DEFAULT NULL,
  `date_update` datetime DEFAULT NULL,
  `date_delete` datetime DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `idx_emp` (`emp_no`,`title`,`from_date`) USING BTREE
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

一：全列匹配
`EXPLAIN SELECT * FROM titles WHERE emp_no='10001' AND title='Senior Engineer' AND from_date='1986-06-26';`


| id   | select_type | table  | type | possible_keys |   key   | key_len |        ref        | rows | Extra |
| :--- | :---------: | :----: | :--: | :-----------: | :-----: | :-----: | :---------------: | :--: | ----: |
| 1    |   SIMPLE    | titles | ref  |    idx_emp    | idx_emp |   310   | const,const,const |  1   |       |


次序调换也是一样
`EXPLAIN SELECT * FROM titles WHERE from_date='1986-06-26' AND emp_no='10001' AND title='Senior Engineer';`


二：最左前缀匹配

`EXPLAIN SELECT * FROM titles WHERE emp_no='10001';`

| id   | select_type | table  | type | possible_keys |   key   | key_len |  ref  | rows | Extra |
| :--- | :---------: | :----: | :--: | :-----------: | :-----: | :-----: | :---: | :--: | ----: |
| 1    |   SIMPLE    | titles | ref  |    idx_emp    | idx_emp |   152   | const |  1   |       |


三：查询条件用到了索引中列的精确匹配，但是中间某个条件未提供
`EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND from_date='1986-06-26';`

| id   | select_type | table  | type | possible_keys |   key   | key_len |  ref  | rows |                 Extra |
| :--- | :---------: | :----: | :--: | :-----------: | :-----: | :-----: | :---: | :--: | --------------------: |
| 1    |   SIMPLE    | titles | ref  |    idx_emp    | idx_emp |   152   | const |  1   | Using index condition |




```
EXPLAIN SELECT * FROM titles
WHERE emp_no='10001'
AND title IN ('Senior Engineer', 'Staff', 'Engineer', 'Senior Staff', 'Assistant Engineer', 'Technique Leader', 'Manager')
AND from_date='1986-06-26';
```

| id   | select_type | table  | type | possible_keys | key  | key_len | ref  | rows |       Extra |
| :--- | :---------: | :----: | :--: | :-----------: | :--: | :-----: | :--: | :--: | ----------: |
| 1    |   SIMPLE    | titles | ALL  |    idx_emp    |      |         | NULL |  7   | Using where |


四：查询条件没有指定索引第一列

`EXPLAIN SELECT * FROM titles WHERE from_date='1986-06-26';`

| id   | select_type | table  | type | possible_keys | key  | key_len | ref  |  rows  |       Extra |
| :--- | :---------: | :----: | :--: | :-----------: | :--: | :-----: | :--: | :----: | ----------: |
| 1    |   SIMPLE    | titles | ALL  |     NULL      | NULL |  NULL   | NULL | 443308 | Using where |



五：匹配某列的前缀字符串
`EXPLAIN SELECT * FROM employees.titles WHERE emp_no='10001' AND title LIKE 'Senior%';`

| id   | select_type | table  | type  | possible_keys |   key   | key_len | ref  | rows |       Extra |
| :--- | :---------: | :----: | :---: | :-----------: | :-----: | :-----: | :--: | :--: | ----------: |
| 1    |   SIMPLE    | titles | range |    PRIMARY    | PRIMARY |   56    | NULL |  1   | Using where |


# 避免全表扫描
1. 应尽量避免在 where 子句中使用!=或<>操作符，否则将引擎放弃使用索引而进行全表扫描
2. 对查询进行优化，应尽量避免全表扫描，首先应考虑在 where 及 order by 涉及的列上建立索引。
3. 应尽量避免在 where 子句中对字段进行 null 值判断，否则将导致引擎放弃使用索引而进行全表扫描
   如：

```
select id from t where num is null
```

可以在num上设置默认值0，确保表中num列没有null值，然后这样查询：

```
select id from t where num=0
```
1. 尽量避免在 where 子句中使用 or 来连接条件，否则将导致引擎放弃使用索引而进行全表扫描，如：

```
 select id from t where num=10 or num=20

```

可以这样查询：

```
     select id from t where num=10
     union all
     select id from t where num=20
```

5.不能前置百分号

```
  select id from t where name like ‘hkjh%’
```

 若要提高效率，可以考虑全文检索。
1. in 和 not in 也要慎用，如：

```
 select id from t where num in(1,2,3)
```

对于连续的数值，能用 between 就不要用 in 了：

```
 select id from t where num between 1 and 3
```

7.如果在 where 子句中使用参数，也会导致全表扫描。因为SQL只有在运行时才会解析局部变量，但优化程序不能将访问计划的选择推迟到运行时；它必须在编译时进行选择。然 而，如果在编译时建立访问计划，变量的值还是未知的，因而无法作为索引选择的输入项。如下面语句将进行全表扫描：

```
select id from t where num=@num
```

可以改为强制查询使用索引：

```
select id from t with(index(索引名)) where num=@num
```

8.应尽量避免在where子句中对字段进行函数操作，这将导致引擎放弃使用索引而进行全表扫描。如：

```
     select id from t where substring(name,1,3)=’abc’–name以abc开头的id
     select id from t where datediff(day,createdate,’2005-11-30′)=0–’2005-11-30′生成的id
```

应改为:

```
     select id from t where name like ‘abc%’
     select id from t where createdate>=’2005-11-30′ and createdate<’2005-12-1′
```
# 用 exists 代替 in

很多时候用 exists 代替 in 是一个好的选择：

```
     select num from a where num in(select num from b)
```

用下面的语句替换：

```
   select num from a where exists(select 1 from b where num=a.num)
```
# 使用数字型字段

1.尽量使用数字型字段，若只含数值信息的字段尽量不要设计为字符型，这会降低查询和连接的性能，并会增加存储开销。这是因为引擎在处理查询和连接时会 逐个比较字符串中每一个字符，而对于数字型而言只需要比较一次就够了。

2.尽可能的使用 varchar/nvarchar 代替 char/nchar ，因为首先变长字段存储空间小，可以节省存储空间，其次对于查询来说，在一个相对较小的字段内搜索效率显然要高些。
# 临时表
- 尽量使用表变量来代替临时表。如果表变量包含大量数据，请注意索引非常有限（只有主键索引）。
- 在新建临时表时，如果一次性插入数据量很大，那么可以使用 select into 代替 create table，避免造成大量 log ，以提高速度；如果数据量不大，为了缓和系统表的资源，应先create table，然后insert。
- 如果使用到了临时表，在存储过程的最后务必将所有的临时表显式删除，先 truncate table ，然后 drop table ，这样可以避免系统表的较长时间锁定。
# 重构查询的方式
1. 将一个复杂查询拆分为数个小且简单的查询，数据返回也快。
2. 切分查询，如删除10万条数据，可以切分为10次，每次删除1万条。
3. 分解关联查询：

```
SELECT * FROM tag
    JOIN tag_post ON tag_post.tag_id = tag.id
    JOIN post ON tag_post.post_id = post.id
WHERE tag.name = 'mysql';
```

分解为

```
SELECT * FROM tag WHERE name = 'mysql';
SELECT * FROM tag_post WHERE tag_id = 1234;
SELECT * FROM post WHERE post.id in (123,456,789,818);
```

4.当只要一行数据时使用 LIMIT 1

一个实例，如何对单表查询优化：

`select * from   product   limit `    866613,20 37.44秒

`select id from  product   limit  866613,20  `   0.2秒(主键索引)

`SELECT *  FROM  product  WHERE    ID   >=  (select   id   from  product  limit   866613,1)  limit  20`
0.2秒
# 慢查询基础

 优化数据访问，就是优化访问的数据，操作对象是要访问的数据，两方面，是否向服务器请求了大量不需要的数据，二是是否逼迫MySQL扫描额外的记录（没有必要扫描）。

  请求不需要数据的典型案例：不加LIMIT（返回全部数据，只取10条）、多表关联Select \* 返回全部列（多表关联查询时*返回多个表的全部列）、还是Select *（可能写程序方面或代码复用方面有好处，但还要权衡）、重复查询相同数据（真需要这样，可以缓存下来，移动开发这个很有必要本地存储）。

标志额外扫描的三个指标：响应时间（自己判断是否合理值）、扫描的行数、返回的行数，一般扫描行数>返回行数。

扫描的行数需要与一个“访问类型”概念关联，就是 Explain 中的 type，explain的type结果由差到优分别是：ALL（全表扫描）、index（索引扫描）、range（范围扫描）、ref（唯一索引查询 key_col=xx）、const（常数引用）等。从“访问类型”可以明白，索引让 MySQL 以最高效、扫描行数最少的方式找到需要的记录。

书中有个例子，说明在where中使用已是索引的列和取消该列的索引后两种结果，type由ref变为All，预估要访问的rows从10变为5073，差异非常明显。
# MySQL查询优化器的局限性

一个UNION限制，无法将限制条件从外层下推到内层，改造例子如下

```
(SELECT first_name,last_name
FROM sak.actor
ORDER BY last_name)
UNION ALL
(SELECT first_name,last_name
FROM sak.customer
ORDER BY last_name)
LIMIT 20;
```

优化后

```
(SELECT first_name,last_name
FROM sak.actor
ORDER BY last_name
LIMIT 20)
UNION ALL
(SELECT first_name,last_name
FROM sak.customer
ORDER BY last_name
LIMIT 20)
LIMIT 20;
```

等值传递：讲的IN列表，MySQL会将IN列表的值传到各个过滤子句，如果IN列表太大，会造成额外消耗，优化和执行都很慢。

最大值和最小值，MySQL对 MIN()和MAX()做得不好

```
SELECT MIN(actor_id) FROM sak.actor WHERE first_name = 'EE';
```

改造后（first_name 不是索引，原来必须全表查询）

```
SELECT actor_id FROM sak.actor USE INDEX(PRIMARY)
WHERE first_name = 'EE' LIMIT 1;
```



 