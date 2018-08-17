---
title: Java分布式锁四种实现方案
abbrlink: b49cb7a7
date: 2017-05-13 23:11:12
tags:
categories:
---


## 方案一：数据库乐观锁

乐观锁通常实现基于数据版本(version)的记录机制实现的，比如有一张红包表（t_bonus），有一个字段(left_count)记录礼物的剩余个数，用户每领取一个奖品，对应的left_count减1，在并发的情况下如何要保证left_count不为负数，乐观锁的实现方式为在红包表上添加一个版本号字段（version），默认为0。

### 异常实现流程

```sql
-- 可能会发生的异常情况
-- 线程1查询，当前left_count为1，则有记录
select * from t_bonus where id = 10001 and left_count > 0

-- 线程2查询，当前left_count为1，也有记录
select * from t_bonus where id = 10001 and left_count > 0

-- 线程1完成领取记录，修改left_count为0,
update t_bonus set left_count = left_count - 1 where id = 10001

-- 线程2完成领取记录，修改left_count为-1，产生脏数据
update t_bonus set left_count = left_count - 1 where id = 10001
```

#### 通过乐观锁实现

```sql
-- 添加版本号控制字段
ALTER TABLE table ADD COLUMN version INT DEFAULT '0' NOT NULL AFTER t_bonus;

-- 线程1查询，当前left_count为1，则有记录，当前版本号为1234
select left_count, version from t_bonus where id = 10001 and left_count > 0

-- 线程2查询，当前left_count为1，有记录，当前版本号为1234
select left_count, version from t_bonus where id = 10001 and left_count > 0

-- 线程1,更新完成后当前的version为1235，update状态为1，更新成功
update t_bonus set version = 1235, left_count = left_count-1 where id = 10001 and version = 1234

-- 线程2,更新由于当前的version为1235，udpate状态为0，更新失败，再针对相关业务做异常处理
update t_bonus set version = 1235, left_count = left_count-1 where id = 10001 and version = 1234
```

## 方案二：基于Redis的分布式锁

>SETNX命令（SET if Not eXists）
>语法：SETNX key value
>功能：原子性操作，当且仅当 key 不存在，将 key 的值设为 value ，并返回1；若给定的 key 已经存在，则 SETNX 不做任何动作，并返回0。
>
>Expire命令
>语法：expire(key, expireTime)
>功能：key设置过期时间
>
>GETSET命令
>语法：GETSET key value
>功能：将给定 key 的值设为 value ，并返回 key 的旧值 (old value)，当 key 存在但不是字符串类型时，返回一个错误，当key不存在时，返回nil。
>
>GET命令
>语法：GET key
>功能：返回 key 所关联的字符串值，如果 key 不存在那么返回特殊值 nil 。
>
>DEL命令
>语法：DEL key [KEY …]
>功能：删除给定的一个或多个 key ,不存在的 key 会被忽略。
>
>SET命令
>语法：SET key value  \[expiration EX seconds |PX milliseconds][NX|XX]
>功能：如果key已经存在，则覆盖
>解释：`EX` *seconds* -- 指定到期时间，秒
>`PX` *milliseconds* -- 指定到期时间，毫秒
>`NX` --  不存在才设置
>`XX` -- 只有存在才设置
>

### 第一种：使用redis的setnx()、expire()方法，用于分布式锁
1. setnx(lockkey, 1) 如果返回0，则说明占位失败；如果返回1，则说明占位成功
2. expire()命令对lockkey设置超时时间，为的是避免死锁问题。
3. 执行完业务代码后，可以通过delete命令删除key。

*这个方案其实是可以解决日常工作中的需求的，但从技术方案的探讨上来说，可能还有一些可以完善的地方。比如，如果在第一步setnx执行成功后，在expire()命令执行成功前，发生了宕机的现象，那么就依然会出现死锁的问题。这个问题可以通过使用Lua脚本（包含SETNX和EXPIRE两条命令），但是如果Redis仅执行了一条命令后crash或者发生主从切换，依然会出现锁没有过期时间，最终导致无法释放。*

```lua
local r = tonumber(redis.call('SETNX', KEYS[1],ARGV[1]));
if (r == 1) then
	redis.call('PEXPIRE',KEYS[1],ARGV[2]);
end
return r
```

###  第二种：使用redis的setnx()、get()、getset()方法，用于分布式锁，解决死锁问题

1. setnx(lockkey, 当前时间+过期超时时间) ，如果返回1，则获取锁成功；如果返回0则没有获取到锁，转向2。
2. get(lockkey)获取值oldExpireTime ，并将这个value值与当前的系统时间进行比较，如果小于当前系统时间，则认为这个锁已经超时，可以允许别的请求重新获取，转向3。
3. 计算newExpireTime=当前时间+过期超时时间，然后getset(lockkey, newExpireTime) 会返回当前lockkey的值currentExpireTime。
4. 判断currentExpireTime与oldExpireTime 是否相等，如果相等，说明当前getset设置成功，获取到了锁。如果不相等，说明这个锁又被别的请求获取走了，那么当前请求可以直接返回失败，或者继续重试。
5. 在获取到锁之后，当前线程可以开始自己的业务处理，当处理完毕后，比较自己的处理时间和对于锁设置的超时时间，如果小于锁设置的超时时间，则直接执行delete释放锁；如果大于锁设置的超时时间，则不需要再锁进行处理。


**注意**：这个版本去掉了EXPIRE命令，改为通过Value时间戳值来判断过期

问题：

    1. 在锁竞争较高的情况下，会出现Value不断被覆盖，但是没有一个Client获取到锁。
    2. 在获取锁的过程中不断的修改原有锁的数据，设想一种场景C1，C2竞争锁，C1获取到了锁，C2锁执行了GETSET操作修改了C1锁的过期时间，如果C1没有正确释放锁，锁的过期时间被延长，其它Client需要等待更久的时间。

### 第三种：使用 SET resource_name my_random_value NX PX 30000

redis 2.6.12版本以后支持了set带过期时间的写法，官方的意思是以后会逐步用setnx取代set

1. Redis客户端为了**获取锁**，向Redis节点发送如下命令：

   ```
   SET resource_name my_random_value NX PX 30000
   ```

   上面的命令如果执行成功，则客户端成功获取到了锁，接下来就可以**访问共享资源**了；而如果上面的命令执行失败，则说明获取锁失败。

   注意，在上面的`SET`命令中：

   - `my_random_value`是由客户端生成的一个随机字符串，它要保证在足够长的一段时间内在所有客户端的所有获取锁的请求中都是唯一的。
   - `NX`表示只有当`resource_name`对应的key值不存在的时候才能`SET`成功。这保证了只有第一个请求的客户端才能获得锁，而其它客户端在锁被释放之前都无法获得锁。
   - `PX 30000`表示这个锁有一个30秒的自动过期时间。当然，这里30秒只是一个例子，客户端可以选择合适的过期时间。

2. 当客户端完成了对共享资源的操作之后，执行下面的Redis Lua脚本来**释放锁**：

   ```lua
   if redis.call("get",KEYS[1]) == ARGV[1] then
       return redis.call("del",KEYS[1])
   else
       return 0
   end
   ```


   这段Lua脚本在执行的时候要把前面的`my_random_value`作为`ARGV[1]`的值传进去，把`resource_name`作为`KEYS[1]`的值传进去。

问题：

1. 这个锁必须要设置一个过期时间。否则的话，当一个客户端获取锁成功之后，假如它崩溃了，或者由于发生了网络分割（network partition）导致它再也无法和Redis节点通信了，那么它就会一直持有这个锁，而其它客户端永远无法获得锁了。antirez在后面的分析中也特别强调了这一点，而且把这个过期时间称为锁的有效时间(lock validity time)。获得锁的客户端必须在这个时间之内完成对共享资源的访问。

2. 第一步**获取锁**的操作，网上不少文章把它实现成了两个Redis命令：

   ```
   SETNX resource_name my_random_value
   EXPIRE resource_name 30
   ```

   虽然这两个命令和前面算法描述中的一个`SET`命令执行效果相同，但却不是原子的。如果客户端在执行完`SETNX`后崩溃了，那么就没有机会执行`EXPIRE`了，导致它一直持有这个锁。

3. 设置一个随机字符串`my_random_value`是很有必要的，它保证了一个客户端释放的锁必须是自己持有的那个锁。假如获取锁时`SET`的不是一个随机字符串，而是一个固定值，那么可能会发生下面的执行序列：

   1. 客户端1获取锁成功。
   2. 客户端1在某个操作上阻塞了很长时间。
   3. 过期时间到了，锁自动释放了。
   4. 客户端2获取到了对应同一个资源的锁。
   5. 客户端1从阻塞中恢复过来，释放掉了客户端2持有的锁。

   之后，客户端2在访问共享资源的时候，就没有锁为它提供保护了。

4. 释放锁的操作必须使用Lua脚本来实现。释放锁其实包含三步操作：'GET'、判断和'DEL'，用Lua脚本来实现能保证这三步的原子性。否则，如果把这三步操作放到客户端逻辑中去执行的话，就有可能发生与前面第三个问题类似的执行序列：

   1. 客户端1获取锁成功。
   2. 客户端1访问共享资源。
   3. 客户端1为了释放锁，先执行'GET'操作获取随机字符串的值。
   4. 客户端1判断随机字符串的值，与预期的值相等。
   5. 客户端1由于某个原因阻塞住了很长时间。
   6. 过期时间到了，锁自动释放了。
   7. 客户端2获取到了对应同一个资源的锁。
   8. 客户端1从阻塞中恢复过来，执行`DEL`操纵，释放掉了客户端2持有的锁。

   实际上，在上述第三个问题和第四个问题的分析中，如果不是客户端阻塞住了，而是出现了大的网络延迟，也有可能导致类似的执行序列发生。

5. 假如Redis节点宕机了，那么所有客户端就都无法获得锁了，服务变得不可用。为了提高可用性，我们可以给这个Redis节点挂一个Slave，当Master节点不可用的时候，系统自动切到Slave上（failover）。但由于Redis的主从复制（replication）是异步的，这可能导致在failover过程中丧失锁的安全性。考虑下面的执行序列：

   1. 客户端1从Master获取了锁。
   2. Master宕机了，存储锁的key还没有来得及同步到Slave上。
   3. Slave升级为Master。
   4. 客户端2从新的Master获取到了对应同一个资源的锁。

   于是，客户端1和客户端2同时持有了同一个资源的锁。

### 第四种：Redlock算法

在Redis的分布式环境中，我们假设有N个Redis master。这些节点完全互相独立，不存在主从复制或者其他集群协调机制。

1. 获取当前Unix时间，以毫秒为单位。

2. 依次尝试从N个实例，使用相同的key和随机值获取锁。在步骤2，当向Redis设置锁时,客户端应该设置一个网络连接和响应超时时间，这个超时时间应该小于锁的失效时间。例如你的锁自动失效时间为10秒，则超时时间应该在5-50毫秒之间。这样可以避免服务器端Redis已经挂掉的情况下，客户端还在死死地等待响应结果。如果服务器端没有在规定时间内响应，客户端应该尽快尝试另外一个Redis实例。

3. 客户端使用当前时间减去开始获取锁时间（步骤1记录的时间）就得到获取锁使用的时间。当且仅当从大多数（这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。

4. 如果取到了锁，key的真正有效时间等于有效时间减去获取锁所使用的时间（步骤3计算的结果）。

5. 如果因为某些原因，获取锁失败（*没有*在至少N/2+1个Redis实例取到锁或者取锁时间已经超过了有效时间），客户端应该在所有的Redis实例上进行解锁（即便某些Redis实例根本就没有加锁成功）。



https://github.com/www1350/redislock

## 方案三：基于Zookeeper的分布式锁

### 利用节点名称的唯一性来实现独占锁

>ZooKeeper机制规定同一个目录下只能有一个唯一的文件名，zookeeper上的一个znode看作是一把锁，通过createznode的方式来实现。所有客户端都去创建/lock/${lock_name}_lock节点，最终成功创建的那个客户端也即拥有了这把锁，创建失败的可以选择监听继续等待，还是放弃抛出异常实现独占锁。

### 利用临时顺序节点控制时序实现

>/lock已经预先存在，所有客户端在它下面创建临时顺序编号目录节点，和选master一样，编号最小的获得锁，用完删除，依次方便。
>算法思路：对于加锁操作，可以让所有客户端都去/lock目录下创建临时顺序节点，如果创建的客户端发现自身创建节点序列号是/lock/目录下最小的节点，则获得锁。否则，监视比自己创建节点的序列号小的节点（比自己创建的节点小的最大节点），进入等待。 对于解锁操作，只需要将自身创建的节点删除即可。

1. 客户端调用create()方法创建名为“_locknode_/guid-lock-”的节点，需要注意的是，这里节点的创建类型需要设置为EPHEMERAL_SEQUENTIAL。
2. 客户端调用getChildren(“_locknode_”)方法来获取所有已经创建的子节点，**同时在这个节点上注册上子节点变更通知的Watcher**。
3. 客户端获取到所有子节点path之后，如果发现自己在步骤1中创建的节点是所有节点中序号最小的，那么就认为这个客户端获得了锁。
4. 如果在步骤3中发现自己并非是所有子节点中最小的，说明自己还没有获取到锁，就开始等待，直到下次子节点变更通知的时候，再进行子节点的获取，判断是否获取锁。

**释放锁**的过程相对比较简单，就是删除自己创建的那个子节点即可。

**问题所在**

上面这个分布式锁的实现中，大体能够满足了一般的分布式集群竞争锁的需求。这里说的一般性场景是指集群规模不大，一般在10台机器以内。

不过，细想上面的实现逻辑，我们很容易会发现一个问题，步骤4，“即获取所有的子点，判断自己创建的节点是否已经是序号最小的节点”，这个过程，在整个分布式锁的竞争过程中，大量重复运行，并且绝大多数的运行结果都是判断出自己并非是序号最小的节点，从而继续等待下一次通知——这个显然看起来不怎么科学。客户端无端的接受到过多的和自己不相关的事件通知，这如果在集群规模大的时候，会对Server造成很大的性能影响，并且如果一旦同一时间有多个节点的客户端断开连接，这个时候，服务器就会像其余客户端发送大量的事件通知——这就是所谓的羊群效应。而这个问题的根源在于，没有找准客户端真正的关注点。

我们再来回顾一下上面的分布式锁竞争过程，它的核心逻辑在于：判断自己是否是所有节点中序号最小的。于是，很容易可以联想的到的是，每个节点的创建者只需要关注比自己序号小的那个节点。

**改进后的分布式锁实现**

下面是改进后的分布式锁实现，和之前的实现方式唯一不同之处在于，这里设计成每个锁竞争者，只需要关注”_locknode_”节点下序号比自己小的那个节点是否存在即可。实现如下：

1. 客户端调用create()方法创建名为“_locknode_/guid-lock-”的节点，需要注意的是，这里节点的创建类型需要设置为EPHEMERAL_SEQUENTIAL。
2. 客户端调用getChildren(“_locknode_”)方法来获取所有已经创建的子节点，注意，这里不注册任何Watcher。
3. 客户端获取到所有子节点path之后，如果发现自己在步骤1中创建的节点序号最小，那么就认为这个客户端获得了锁。
4. 如果在步骤3中发现自己并非所有子节点中最小的，说明自己还没有获取到锁。此时客户端需要找到比自己小的那个节点，然后对其调用exist()方法，同时注册事件监听。
5. 之后当这个被关注的节点被移除了，客户端会收到相应的通知。这个时候客户端需要再次调用getChildren(“_locknode_”)方法来获取所有已经创建的子节点，确保自己确实是最小的节点了，然后进入步骤3。

## 方案四：基于memcached的分布式锁

### 利用add操作实现独占锁

memcache中**Memcache::add()**方法在缓存服务器之前不存在`key`时， 以`key`作为key存储一个变量`var`到缓存服务器，该操作是原子操作，可以设置一个超时时间。del用来解锁。





参考：

http://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html

https://mp.weixin.qq.com/s/JTsJCDuasgIJ0j95K8Ay8w

https://mp.weixin.qq.com/s/4CUe7OpM6y1kQRK8TOC_qQ

http://redis.cn/topics/distlock.html

http://tech.dianwoda.com/2018/04/11/redisfen-bu-shi-suo-jin-hua-shi/