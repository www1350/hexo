---
title: mysql事务隔离机制
abbrlink: 6275b0e6
date: 2018-11-23 22:32:45
tags:
categories:
---

> Isolation.DEFAULT(TransactionDefinition.ISOLATION_DEFAULT)使用数据库默认的事务隔离级别。

> Isolation.READ_UNCOMMITTED(TransactionDefinition.ISOLATION_READ_UNCOMMITTED),这是事务最低的隔离级别，它允许另外一个事务可以看到这个事务未提交的数据。这种隔离级别会产生脏读，不可重复读和幻像读。
>
> 实现：SET SESSION TRANSACTION ISOLATION LEVEL READ UNCOMMITTED

> Isolation.READ_COMMITTED(TransactionDefinition.ISOLATION_READ_COMMITTED),保证一个事务修改的数据提交后才能被另外一个事务读取。另外一个事务不能读取该事务未提交的数据。这种事务隔离级别可以避免脏读出现，但是可能会出现不可重复读和幻像读。
>
> 实现：SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED

> Isolation.REPEATABLE_READ(TransactionDefinition.ISOLATION_REPEATABLE_READ),这种事务隔离级别可以防止脏读，不可重复读。但是可能出现幻像读。
>
> 实现：SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ

> Isolation.SERIALIZABLE(TransactionDefinition.ISOLATION_SERIALIZABLE);这是花费最高代价但是最可靠的事务隔离级别。事务被处理为顺序执行。除了防止脏读，不可重复读外，还避免了幻读。
>
> 实现：SET SESSION TRANSACTION ISOLATION LEVEL SERIALIZABLE

√: 可能出现    ×: 不会出现

|                  | 脏读 | 不可重复读 | 幻读 |
| ---------------- | ---- | ---------- | ---- |
| Read uncommitted | √    | √          | √    |
| Read committed   | ×    | √          | √    |
| Repeatable read  | ×    | ×          | √    |
| Serializable     | ×    | ×          | ×    |



- 未提交读(Read Uncommitted)：允许脏读，也就是可能读取到其他会话中未提交事务修改的数据
- 提交读(Read Committed)：只能读取到已经提交的数据。Oracle等多数数据库默认都是该级别 (不重复读)
- 可重复读(Repeated Read)：可重复读。在同一个事务内的查询都是事务开始时刻一致的，InnoDB默认级别。在SQL标准中，该隔离级别消除了不可重复读，但是还存在幻象读
- 串行读(Serializable)：完全串行化的读，每次读都需要获得表级共享锁，读写相互都会阻塞

## 为什么RU级别会发生脏读，而其他的隔离级别能够避免？

RU级别的操作其实就是对事务内的每一条更新语句对应的行记录加上读写锁来操作，而不把一个事务当成一个整体来加锁，所以会导致脏读。但是RC和RR能够通过MVCC来保证记录只有在最后COMMIT后才会让别的事务看到。





Read Committed（读取提交内容）

在RC级别中，数据的读取都是不加锁的，但是数据的写入、修改和删除是需要加锁的。



由于MySQL的InnoDB默认是使用的RR级别，所以我们先要将该session开启成RC级别，并且设置binlog的模式



SET session transaction isolation level read committed;

SET SESSION binlog_format = 'ROW';（或者是MIXED）



Repeatable Read（可重读）

这是MySQL中InnoDB默认的隔离级别。我们姑且分“读”和“写”两个模块来讲解。



读

读就是可重读，可重读这个概念是一事务的多个实例在并发读取数据时，会看到同样的数据行，有点抽象，我们来看一下效果。



RC（不可重读）模式下的展现

| 事务A                                                        | 事务B                                                      |      |
| ------------------------------------------------------------ | ---------------------------------------------------------- | ---- |
| begin;                                                       | begin;                                                     |      |
| select id,class_name,teacher_id from class_teacher where teacher_id=1; idclass_nameteacher_id 1初三二班1 2初三一班1 |                                                            |      |
|                                                              | update class_teacher set class_name='初三三班' where id=1; |      |
|                                                              | commit;                                                    |      |
| select id,class_name,teacher_id from class_teacher where teacher_id=1; idclass_nameteacher_id 1初三三班1 2初三一班1 读到了事务B修改的数据，和第一次查询的结果不一样，是不可重读的。 |                                                            |      |
| commit;                                                      |                                                            |      |





事务B修改id=1的数据提交之后，事务A同样的查询，后一次和前一次的结果不一样，这就是不可重读（重新读取产生的结果不一样）。这就很可能带来一些问题，那么我们来看看在RR级别中MySQL的表现：

| 事务A                                                        | 事务B                                                        | 事务C                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| begin;                                                       | begin;                                                       | begin;                                                       |
| select id,class_name,teacher_id from class_teacher where teacher_id=1; idclass_nameteacher_id 1初三二班1 2初三一班1 |                                                              |                                                              |
|                                                              | update class_teacher set class_name='初三三班' where id=1; commit; |                                                              |
|                                                              |                                                              | insert into class_teacher values (null,'初三三班',1);commit; |
| select id,class_name,teacher_id from class_teacher where teacher_id=1; idclass_nameteacher_id 1初三二班1 2初三一班1 没有读到事务B修改的数据，和第一次sql读取的一样，是可重复读的。 没有读到事务C新添加的数据。 |                                                              |                                                              |
| commit;                                                      |                                                              |                                                              |

我们注意到，当teacher_id=1时，事务A先做了一次读取，事务B中间修改了id=1的数据，并commit之后，事务A第二次读到的数据和第一次完全相同。所以说它是可重读的。那么MySQL是怎么做到的呢？这里姑且卖个关子，我们往下看。



## 为什么RC级别不能重复读，而RR级别能够避免？

在RC事务隔离级别下,每次语句执行都关闭ReadView,然后重新创建一份ReadView。而在RR下,事务开始后第一个读操作创建ReadView,一直到事务结束关闭





### 不可重复读和幻读的区别

很多人容易搞混不可重复读和幻读，确实这两者有些相似。但不可重复读重点在于update和delete，而幻读的重点在于insert。

如果使用锁机制来实现这两种隔离级别，在可重复读中，该sql第一次读取到数据后，就将这些数据加锁，其它事务无法修改这些数据，就可以实现可重复读了。但这种方法却无法锁住insert的数据，所以当事务A先前读取了数据，或者修改了全部数据，事务B还是可以insert数据提交，这时事务A就会发现莫名其妙多了一条之前没有的数据，这就是幻读，不能通过行锁来避免。需要Serializable隔离级别 ，读用读锁，写用写锁，读锁和写锁互斥，这么做可以有效的避免幻读、不可重复读、脏读等问题，但会极大的降低数据库的并发能力。



所以说不可重复读和幻读最大的区别，就在于如何通过锁机制来解决他们产生的问题。



上文说的，是使用悲观锁机制来处理这两种问题，但是MySQL、ORACLE、PostgreSQL等成熟的数据库，出于性能考虑，都是使用了以乐观锁为理论基础的MVCC（多版本并发控制）来避免这两种问题。





- 快照读：就是select

- - select * from table ....;

- 当前读：特殊的读操作，插入/更新/删除操作，属于当前读，处理的都是当前的数据，需要加锁。

- - select * from table where ? lock in share mode;
  - select * from table where ? for update;
  - insert;
  - update ;
  - delete;


GAP间隙锁

RC级别：

| 事务A                                                        | 事务B                                                        |      |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---- |
| begin;                                                       | begin;                                                       |      |
| select id,class_name,teacher_id from class_teacher where teacher_id=30; idclass_nameteacher_id 2初三二班30 |                                                              |      |
| update class_teacher set class_name='初三四班' where teacher_id=30; |                                                              |      |
|                                                              | insert into class_teacher values (null,'初三二班',30); commit; |      |
| select id,class_name,teacher_id from class_teacher where teacher_id=30; idclass_nameteacher_id 2初三四班30 10初三二班30 |                                                              |      |

RR级别：

| 事务A                                                        | 事务B                                                        |      |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ---- |
| begin;                                                       | begin;                                                       |      |
| select id,class_name,teacher_id from class_teacher where teacher_id=30; idclass_nameteacher_id 2初三二班30 |                                                              |      |
| update class_teacher set class_name='初三四班' where teacher_id=30; |                                                              |      |
|                                                              | insert into class_teacher values (null,'初三二班',30); waiting.... |      |
| select id,class_name,teacher_id from class_teacher where teacher_id=30; idclass_nameteacher_id 2初三四班30 |                                                              |      |
| commit;                                                      | 事务Acommit后，事务B的insert执行。                           |      |

通过对比我们可以发现，在RC级别中，事务A修改了所有teacher_id=30的数据，但是当事务Binsert进新数据后，事务A发现莫名其妙多了一行teacher_id=30的数据，而且没有被之前的update语句所修改，这就是“当前读”的幻读。



RR级别中，事务A在update后加锁，事务B无法插入新数据，这样事务A在update前后读的数据保持一致，避免了幻读。这个锁，就是Gap锁。



MySQL是这么实现的：

在class_teacher这张表中，teacher_id是个索引，那么它就会维护一套B+树的数据关系，为了简化，我们用链表结构来表达（实际上是个树形结构，但原理相同）

![image](https://user-images.githubusercontent.com/7789698/48948565-de03cf00-ef6f-11e8-8b15-400d86f16bf4.png)如图所示，InnoDB使用的是聚集索引，teacher_id身为二级索引，就要维护一个索引字段和主键id的树状结构（这里用链表形式表现），并保持顺序排列。



Innodb将这段数据分成几个个区间



- (negative infinity, 5],
- (5,30],
- (30,positive infinity)；

update class_teacher set class_name='初三四班' where teacher_id=30;不仅用行锁，锁住了相应的数据行；同时也在两边的区间，（5,30]和（30，positive infinity），都加入了gap锁。这样事务B就无法在这个两个区间insert进新数据。



受限于这种实现方式，Innodb很多时候会锁住不需要锁的区间。如下所示：

| 事务A                                                        | 事务B                                                        | 事务C                                                  |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------ |
| begin;                                                       | begin;                                                       | begin;                                                 |
| select id,class_name,teacher_id from class_teacher; idclass_nameteacher_id 1初三一班5  2初三二班30 |                                                              |                                                        |
| update class_teacher set class_name='初一一班' where teacher_id=20; |                                                              |                                                        |
|                                                              | insert into class_teacher values (null,'初三五班',10); waiting ..... | insert into class_teacher values (null,'初三五班',40); |
| commit;                                                      | 事务A commit之后，这条语句才插入成功                         | commit;                                                |
|                                                              | commit;                                                      |                                                        |

update的teacher_id=20是在(5，30]区间，即使没有修改任何数据，Innodb也会在这个区间加gap锁，而其它区间不会影响，事务C正常插入。



如果使用的是没有索引的字段，比如update class_teacher set teacher_id=7 where class_name='初三八班（即使没有匹配到任何数据）',那么会给全表加入gap锁。同时，它不能像上文中行锁一样经过MySQL Server过滤自动解除不满足条件的锁，因为没有索引，则这些字段也就没有排序，也就没有区间。除非该事务提交，否则其它事务无法插入任何数据。



行锁防止别的事务修改或删除，GAP锁防止别的事务新增，行锁和GAP锁结合形成的的Next-Key锁共同解决了RR级别在写数据时的幻读问题。



### Read uncommitted 读未提交

公司发工资了，领导把5000元打到singo的账号上，但是该事务并未提交，而singo正好去查看账户，发现工资已经到账，是5000元整，非常高兴。可是不幸的是，领导发现发给singo的工资金额不对，是2000元，于是迅速回滚了事务，修改金额后，将事务提交，最后singo实际的工资只有2000元，singo空欢喜一场。

![image](https://user-images.githubusercontent.com/7789698/48948598-fe338e00-ef6f-11e8-86d1-1aceac364782.png)

出现上述情况，即我们所说的脏读，两个并发的事务，“事务A：领导给singo发工资”、“事务B：singo查询工资账户”，事务B读取了事务A尚未提交的数据。

当隔离级别设置为Read uncommitted时，就可能出现脏读，如何避免脏读，请看下一个隔离级别。



#### Read committed 读提交

singo拿着工资卡去消费，系统读取到卡里确实有2000元，而此时她的老婆也正好在网上转账，把singo工资卡的2000元转到另一账户，并在singo之前提交了事务，当singo扣款时，系统检查到singo的工资卡已经没有钱，扣款失败，singo十分纳闷，明明卡里有钱，为何......



出现上述情况，即我们所说的不可重复读，两个并发的事务，“事务A：singo消费”、“事务B：singo的老婆网上转账”，事务A事先读取了数据，事务B紧接了更新了数据，并提交了事务，而事务A再次读取该数据时，数据已经发生了改变。



当隔离级别设置为Read committed时，避免了脏读，但是可能会造成不可重复读。



大多数数据库的默认级别就是Read committed，比如Sql Server , Oracle。如何解决不可重复读这一问题，请看下一个隔离级别。



#### Repeatable read 重复读

当隔离级别设置为Repeatable read时，可以避免不可重复读。当singo拿着工资卡去消费时，一旦系统开始读取工资卡信息（即事务开始），singo的老婆就不可能对该记录进行修改，也就是singo的老婆不能在此时转账。



虽然Repeatable read避免了不可重复读，但还有可能出现幻读。



singo的老婆工作在银行部门，她时常通过银行内部系统查看singo的信用卡消费记录。有一天，她正在查询到singo当月信用卡的总消费金额（select sum(amount) from transaction where month = 本月）为80元，而singo此时正好在外面胡吃海塞后在收银台买单，消费1000元，即新增了一条1000元的消费记录（insert transaction ... ），并提交了事务，随后singo的老婆将singo当月信用卡消费的明细打印到A4纸上，却发现消费总额为1080元，singo的老婆很诧异，以为出现了幻觉，幻读就这样产生了。

注：Mysql的默认隔离级别就是Repeatable read。



#### Serializable 序列化

Serializable是最高的事务隔离级别，同时代价也花费最高，性能很低，一般很少使用，在该级别下，事务顺序执行，不仅可以避免脏读、不可重复读，还避免了幻像读。



# MySql ACID如何保证？

ACID，指数据库事务正确执行的四个基本要素的缩写。包含：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability） 



## 为什么InnoDB能够保证原子性？

在事务里任何对数据的修改都会写一个Undo log，然后进行数据的修改，如果出现错误或者用户需要回滚的时候可以利用Undo log的备份数据恢复到事务开始之前的状态。



## 为什么InnoDB能够保证持久性？

在一个事务中的每一次SQL操作之后都会写入一个redo log到buffer中，在最后COMMIT的时候，必须先将该事务的所有日志写入到redo log file进行持久化（这里的写入是顺序写的），待事务的COMMIT操作完成才算完成。即使COMMIT后数据库有任何的问题，在下次重启后依然能够通过redo log的checkpoint进行恢复。



## 为什么InnoDB能够保证一致性？

在事务处理的ACID属性中，一致性是最基本的属性，其它的三个属性都为了保证一致性而存在的。



首先回顾一下一致性的定义。所谓一致性，指的是数据处于一种有意义的状态，这种状态是语义上的而不是语法上的。最常见的例子是转帐。例如从帐户A转一笔钱到帐户B上，如果帐户A上的钱减少了，而帐户B上的钱却没有增加，那么我们认为此时数据处于不一致的状态。



在数据库实现的场景中，一致性可以分为数据库外部的一致性和数据库内部的一致性。前者由外部应用的编码来保证，即某个应用在执行转帐的数据库操作时，必须在同一个事务内部调用对帐户A和帐户B的操作。如果在这个层次出现错误，这不是数据库本身能够解决的，也不属于我们需要讨论的范围。后者由数据库来保证，即在同一个事务内部的一组操作必须全部执行成功（或者全部失败）。这就是事务处理的原子性。（上面说过了是用Undo log来保证的）



但是，原子性并不能完全保证一致性。在多个事务并行进行的情况下，即使保证了每一个事务的原子性，仍然可能导致数据不一致的结果，比如丢失更新问题。



为了保证并发情况下的一致性，引入了隔离性，即保证每一个事务能够看到的数据总是一致的，就好象其它并发事务并不存在一样。用术语来说，就是多个事务并发执行后的状态，和它们串行执行后的状态是等价的。