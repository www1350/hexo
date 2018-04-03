---
title: redis设计与实现 阅读笔记
date: 2018-04-03 22:40:56
tags:
categories:
---

1.redis数据结构

# 简单字符串
redis专门封装了一个叫SDS的数据结构

```
struct sdshdr {
    // 记录 buf 数组中已使用字节的数量
    // 等于 SDS 所保存字符串的长度
    int len;

    // 记录 buf 数组中未使用字节的数量
    int free;

    // 字节数组，用于保存字符串
    char buf[];
};
```


- free 属性的值为 0 ， 表示这个 SDS 没有分配任何未使用空间。
- len 属性的值为 5 ， 表示这个 SDS 保存了一个五字节长的字符串。（strlen的复杂度就从O(n)变为O(1)）
- buf 属性是一个 char 类型的数组， 数组的前五个字节分别保存了 'R' 、 'e' 、 'd' 、 'i' 、 's' 五个字符， 而最后一个字节则保存了空字符 '\0' 。

![image](https://user-images.githubusercontent.com/7789698/30947484-9f4c6148-a43c-11e7-8ea0-7c1772c45dab.png)

![2015-09-13_55f50d86a66ae](https://user-images.githubusercontent.com/7789698/30947884-e653020c-a43e-11e7-8692-0afa793643a5.png)


优点：
- 常数复杂度获取字符串长度
- 杜绝缓冲区溢出
  举个例子， <string.h>/strcat 函数可以将 src 字符串中的内容拼接到 dest 字符串的末尾：

`char *strcat(char *dest, const char *src);`

因为 C 字符串不记录自身的长度， 所以 strcat 假定用户在执行这个函数时， 已经为 dest 分配了足够多的内存， 可以容纳 src 字符串中的所有内容， 而一旦这个假定不成立时， 就会产生缓冲区溢出。
举个例子， 假设程序里有两个在内存中紧邻着的 C 字符串 s1 和 s2 ， 其中 s1 保存了字符串 "Redis" ， 而 s2 则保存了字符串 "MongoDB"， 如图 2-7 所示。
![2015-09-13_55f50e28c1620](https://user-images.githubusercontent.com/7789698/30947756-fd52851e-a43d-11e7-8401-edab070fb25f.png)

`strcat(s1, " Cluster");`

将 s1 的内容修改为 "Redis Cluster" ， 但粗心的他却忘了在执行 strcat 之前为 s1 分配足够的空间， 那么在 strcat 函数执行之后， s1 的数据将溢出到 s2 所在的空间中， 导致 s2 保存的内容被意外地修改， 如图 2-8 所示。
![2015-09-13_55f50e29e04a2](https://user-images.githubusercontent.com/7789698/30947766-13d71c50-a43e-11e7-97c3-79d7595df328.png)

与 C 字符串不同， SDS 的空间分配策略完全杜绝了发生缓冲区溢出的可能性： 当 SDS API 需要对 SDS 进行修改时， API 会先检查 SDS 的空间是否满足修改所需的要求， 如果不满足的话， API 会自动将 SDS 的空间扩展至执行修改所需的大小， 然后才执行实际的修改操作， 所以使用 SDS 既不需要手动修改 SDS 的空间大小， 也不会出现前面所说的缓冲区溢出问题。

举个例子， SDS 的 API 里面也有一个用于执行拼接操作的 sdscat 函数， 它可以将一个 C 字符串拼接到给定 SDS 所保存的字符串的后面， 但是在执行拼接操作之前， sdscat 会先检查给定 SDS 的空间是否足够， 如果不够的话， sdscat 就会先扩展 SDS 的空间， 然后才执行拼接操作。

![image](https://user-images.githubusercontent.com/7789698/30947776-2a5fdf48-a43e-11e7-9280-7da4112f1fd5.png)

![image](https://user-images.githubusercontent.com/7789698/30947789-3d2738a6-a43e-11e7-92f8-214bcc0c63ab.png)

- 减少修改字符串时带来的内存重分配次数
   因为 C 字符串并不记录自身的长度， 所以对于一个包含了 N 个字符的 C 字符串来说， 这个 C 字符串的底层实现总是一个 N+1 个字符长的数组（额外的一个字符空间用于保存空字符）。
   因为 C 字符串的长度和底层数组的长度之间存在着这种关联性， 所以每次增长或者缩短一个 C 字符串， 程序都总要对保存这个 C 字符串的数组进行一次内存重分配操作：

如果程序执行的是增长字符串的操作， 比如拼接操作（append）， 那么在执行这个操作之前， 程序需要先通过内存重分配来扩展底层数组的空间大小 —— 如果忘了这一步就会产生缓冲区溢出。
如果程序执行的是缩短字符串的操作， 比如截断操作（trim）， 那么在执行这个操作之后， 程序需要通过内存重分配来释放字符串不再使用的那部分空间 —— 如果忘了这一步就会产生内存泄漏。
举个例子， 如果我们持有一个值为 "Redis" 的 C 字符串 s ， 那么为了将 s 的值改为 "Redis Cluster" ， 在执行：
`strcat(s, " Cluster");`

为了避免 C 字符串的这种缺陷， SDS 通过未使用空间解除了字符串长度和底层数组长度之间的关联： 在 SDS 中， buf 数组的长度不一定就是字符数量加一， 数组里面可以包含未使用的字节， 而这些字节的数量就由 SDS 的 free 属性记录。
通过未使用空间， SDS 实现了空间预分配和惰性空间释放两种优化策略。

- 空间预分配
  空间预分配用于优化 SDS 的字符串增长操作： 当 SDS 的 API 对一个 SDS 进行修改， 并且需要对 SDS 进行空间扩展的时候， 程序不仅会为 SDS 分配修改所必须要的空间， 还会为 SDS 分配额外的未使用空间。

其中， 额外分配的未使用空间数量由以下公式决定：

如果对 SDS 进行修改之后， SDS 的长度（也即是 len 属性的值）将小于 1 MB ， 那么程序分配和 len 属性同样大小的未使用空间， 这时 SDS len 属性的值将和 free 属性的值相同。 举个例子， 如果进行修改之后， SDS 的 len 将变成 13 字节， 那么程序也会分配13 字节的未使用空间， SDS 的 buf 数组的实际长度将变成 13 + 13 + 1 = 27 字节（额外的一字节用于保存空字符）。

如果对 SDS 进行修改之后， SDS 的长度将大于等于 1 MB ， 那么程序会分配 1 MB 的未使用空间。 举个例子， 如果进行修改之后， SDS 的 len 将变成 30 MB ， 那么程序会分配 1 MB 的未使用空间， SDS 的 buf 数组的实际长度将为 30 MB + 1 MB + 1 byte 。

`sdscat(s, " Cluster");`

那么 sdscat 将执行一次内存重分配操作， 将 SDS 的长度修改为 13 字节， 并将 SDS 的未使用空间同样修改为 13 字节， 如图 2-12 所示。

![2015-09-13_55f50e329713a](https://user-images.githubusercontent.com/7789698/30948189-d2db4002-a440-11e7-8e25-6102a31d740e.png)

![2015-09-13_55f50e33eb3a4](https://user-images.githubusercontent.com/7789698/30948205-e30d41b4-a440-11e7-81c8-233008f63d47.png)



如果这时， 我们再次对 s 执行：
`sdscat(s, " Tutorial");`
那么这次 sdscat 将不需要执行内存重分配： 因为未使用空间里面的 13 字节足以保存 9 字节的 " Tutorial" ， 执行 sdscat 之后的 SDS 如图 2-13 所示。
![image](https://user-images.githubusercontent.com/7789698/30948239-15dd7988-a441-11e7-94a1-4abf8aea7ec3.png)


在扩展 SDS 空间之前， SDS API 会先检查未使用空间是否足够， 如果足够的话， API 就会直接使用未使用空间， 而无须执行内存重分配。

通过这种预分配策略， SDS 将连续增长 N 次字符串所需的内存重分配次数从必定 N 次降低为最多 N 次。


- 惰性空间释放

惰性空间释放用于优化 SDS 的字符串缩短操作： 当 SDS 的 API 需要缩短 SDS 保存的字符串时， 程序并不立即使用内存重分配来回收缩短后多出来的字节， 而是使用 free 属性将这些字节的数量记录起来， 并等待将来使用。
举个例子， sdstrim 函数接受一个 SDS 和一个 C 字符串作为参数， 从 SDS 左右两端分别移除所有在 C 字符串中出现过的字符。

sdstrim(s, "XY");   // 移除 SDS 字符串中的所有 'X' 和 'Y'

会将 SDS 修改成图 2-15 所示的样子。

![image](https://user-images.githubusercontent.com/7789698/30948262-4073db6a-a441-11e7-9a09-397fda3aad4d.png)
![image](https://user-images.githubusercontent.com/7789698/30948275-4ecc74f6-a441-11e7-95dd-3a949dce7868.png)

注意执行 sdstrim 之后的 SDS 并没有释放多出来的 8 字节空间， 而是将这 8 字节空间作为未使用空间保留在了 SDS 里面， 如果将来要对 SDS 进行增长操作的话， 这些未使用空间就可能会派上用场。

- 二进制安全
   SDS 的 API 都是二进制安全的（binary-safe）： 所有 SDS API 都会以处理二进制的方式来处理 SDS 存放在 buf 数组里的数据， 程序不会对其中的数据做任何限制、过滤、或者假设 —— 数据在写入时是什么样的， 它被读取时就是什么样。




- 兼容部分 C 字符串函数

<img width="938" alt="wx20170928-114705 2x" src="https://user-images.githubusercontent.com/7789698/30948471-d65552d4-a442-11e7-9570-3422d7570b40.png">


<img width="937" alt="wx20170928-114913 2x" src="https://user-images.githubusercontent.com/7789698/30948529-1574ae24-a443-11e7-9c56-87f4047aeb57.png">


# 链表
```
redis> LLEN integers
(integer) 1024

redis> LRANGE integers 0 10
1) "1"
2) "2"
3) "3"
4) "4"
5) "5"
6) "6"
7) "7"
8) "8"
9) "9"
10) "10"
11) "11"
```


```
typedef struct listNode {

    // 前置节点
    struct listNode *prev;

    // 后置节点
    struct listNode *next;

    // 节点的值
    void *value;

} listNode;
```

多个 listNode 可以通过 prev 和 next 指针组成双端链表， 如图 3-1 所示。

![2015-09-13_55f50fad082e3](https://user-images.githubusercontent.com/7789698/30948577-86fe8d4e-a443-11e7-8700-e9ab993d2bfb.png)


```
typedef struct list {

    // 表头节点
    listNode *head;

    // 表尾节点
    listNode *tail;

    // 链表所包含的节点数量
    unsigned long len;

    // 节点值复制函数
    void *(*dup)(void *ptr);

    // 节点值释放函数
    void (*free)(void *ptr);

    // 节点值对比函数
    int (*match)(void *ptr, void *key);

} list;
```

list 结构为链表提供了表头指针 head 、表尾指针 tail ， 以及链表长度计数器 len ， 而 dup 、 free 和 match 成员则是用于实现多态链表所需的类型特定函数：

dup 函数用于复制链表节点所保存的值；
free 函数用于释放链表节点所保存的值；
match 函数则用于对比链表节点所保存的值和另一个输入值是否相等。


![2015-09-13_55f50fb39b6cb](https://user-images.githubusercontent.com/7789698/30948645-d81d4d82-a443-11e7-9f03-1141889620b2.png)


Redis 的链表实现的特性可以总结如下：

1. 双端： 链表节点带有 prev 和 next 指针， 获取某个节点的前置节点和后置节点的复杂度都是 O(1) 。
  2.无环： 表头节点的 prev 指针和表尾节点的 next 指针都指向 NULL ， 对链表的访问以 NULL 为终点。
  3.带表头指针和表尾指针： 通过 list 结构的 head 指针和 tail 指针， 程序获取链表的表头节点和表尾节点的复杂度为 O(1) 。
  4.带链表长度计数器： 程序使用 list 结构的 len 属性来对 list 持有的链表节点进行计数， 程序获取链表中节点数量的复杂度为 O(1)。
  5.多态： 链表节点使用 void* 指针来保存节点值， 并且可以通过 list 结构的 dup 、 free 、 match 三个属性为节点值设置类型特定函数， 所以链表可以用于保存各种不同类型的值。

<img width="932" alt="wx20170928-120027 2x" src="https://user-images.githubusercontent.com/7789698/30949299-1988948a-a448-11e7-999d-3022b762da23.png">

<img width="936" alt="wx20170928-120039 2x" src="https://user-images.githubusercontent.com/7789698/30949304-28391518-a448-11e7-89b0-409f58f7b079.png">

# 字典
字典， 又称符号表（symbol table）、关联数组（associative array）或者映射（map）， 是一种用于保存键值对（key-value pair）的抽象数据结构。


```
redis> HLEN website
(integer) 10086

redis> HGETALL website
1) "Redis"
2) "Redis.io"
3) "MariaDB"
4) "MariaDB.org"
5) "MongoDB"
6) "MongoDB.org"
# ...
```

其中一个键值对的键为 "Redis" ， 值为 "Redis.io" 。
另一个键值对的键为 "MariaDB" ， 值为 "MariaDB.org" ；
还有一个键值对的键为 "MongoDB" ， 值为 "MongoDB.org" ；



```
typedef struct dictht {

    // 哈希表数组
    dictEntry **table;

    // 哈希表大小
    unsigned long size;

    // 哈希表大小掩码，用于计算索引值
    // 总是等于 size - 1
    unsigned long sizemask;

    // 该哈希表已有节点的数量
    unsigned long used;

} dictht;
```

table 属性是一个数组， 数组中的每个元素都是一个指向 dict.h/dictEntry 结构的指针， 每个 dictEntry 结构保存着一个键值对。
size 属性记录了哈希表的大小， 也即是 table 数组的大小， 而 used 属性则记录了哈希表目前已有节点（键值对）的数量。
sizemask 属性的值总是等于 size - 1 ， 这个属性和哈希值一起决定一个键应该被放到 table 数组的哪个索引上面。

![2015-09-13_55f511fc9428c](https://user-images.githubusercontent.com/7789698/30949367-7e5f2860-a448-11e7-8339-6cefb372aa0e.png)


- 哈希表节点

```
typedef struct dictEntry {

    // 键
    void *key;

    // 值
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
    } v;

    // 指向下个哈希表节点，形成链表
    struct dictEntry *next;

} dictEntry;
```

key 属性保存着键值对中的键， 而 v 属性则保存着键值对中的值， 其中键值对的值可以是一个指针， 或者是一个 uint64_t 整数， 又或者是一个 int64_t 整数。

next 属性是指向另一个哈希表节点的指针， 这个指针可以将多个哈希值相同的键值对连接在一次， 以此来解决键冲突（collision）的问题。

举个例子， 图 4-2 就展示了如何通过 next 指针， 将两个索引值相同的键 k1 和 k0 连接在一起。

![2015-09-13_55f51205335f9](https://user-images.githubusercontent.com/7789698/30951893-ec80c1f0-a457-11e7-8c93-374095b0234f.png)


- 字典

```
typedef struct dict {

    // 类型特定函数
    dictType *type;

    // 私有数据
    void *privdata;

    // 哈希表
    dictht ht[2];

    // rehash 索引
    // 当 rehash 不在进行时，值为 -1
    int rehashidx; /* rehashing not in progress if rehashidx == -1 */

} dict;
```

type 属性和 privdata 属性是针对不同类型的键值对， 为创建多态字典而设置的：

type 属性是一个指向 dictType 结构的指针， 每个 dictType 结构保存了一簇用于操作特定类型键值对的函数， Redis 会为用途不同的字典设置不同的类型特定函数。

而 privdata 属性则保存了需要传给那些类型特定函数的可选参数。

```
typedef struct dictType {

    // 计算哈希值的函数
    unsigned int (*hashFunction)(const void *key);

    // 复制键的函数
    void *(*keyDup)(void *privdata, const void *key);

    // 复制值的函数
    void *(*valDup)(void *privdata, const void *obj);

    // 对比键的函数
    int (*keyCompare)(void *privdata, const void *key1, const void *key2);

    // 销毁键的函数
    void (*keyDestructor)(void *privdata, void *key);

    // 销毁值的函数
    void (*valDestructor)(void *privdata, void *obj);

} dictType;
```

ht 属性是一个包含两个项的数组， 数组中的每个项都是一个 dictht 哈希表， 一般情况下， 字典只使用 ht[0] 哈希表， ht[1] 哈希表只会在对 ht[0] 哈希表进行 rehash 时使用。

除了 ht[1] 之外， 另一个和 rehash 有关的属性就是 rehashidx ： 它记录了 rehash 目前的进度， 如果目前没有在进行 rehash ， 那么它的值为 -1 。

图 4-3 展示了一个普通状态下（没有进行 rehash）的字典：

![2015-09-13_55f5120772706](https://user-images.githubusercontent.com/7789698/30952389-48202166-a45a-11e7-87f9-7f641db990a8.png)


- 哈希算法

当要将一个新的键值对添加到字典里面时， 程序需要先根据键值对的键计算出哈希值和索引值， 然后再根据索引值， 将包含新键值对的哈希表节点放到哈希表数组的指定索引上面。

Redis 计算哈希值和索引值的方法如下：

```
# 使用字典设置的哈希函数，计算键 key 的哈希值
hash = dict->type->hashFunction(key);

# 使用哈希表的 sizemask 属性和哈希值，计算出索引值
# 根据情况不同， ht[x] 可以是 ht[0] 或者 ht[1]
index = hash & dict->ht[x].sizemask;
```

![2015-09-13_55f5128c90054](https://user-images.githubusercontent.com/7789698/30952777-c41403b8-a45b-11e7-9cfc-d17d250c8f40.png)

举个例子， 对于图 4-4 所示的字典来说， 如果我们要将一个键值对 k0 和 v0 添加到字典里面， 那么程序会先使用语句：
`hash = dict->type->hashFunction(k0);`
计算键 k0 的哈希值。
假设计算得出的哈希值为 8 ， 那么程序会继续使用语句：
`index = hash & dict->ht[0].sizemask = 8 & 3 = 0;`
计算出键 k0 的索引值 0 ， 这表示包含键值对 k0 和 v0 的节点应该被放置到哈希表数组的索引 0 位置上， 如图 4-5 所示。
![2015-09-13_55f51293b4642](https://user-images.githubusercontent.com/7789698/30952800-da4a106e-a45b-11e7-8b36-32757f62cd40.png)

当字典被用作数据库的底层实现， 或者哈希键的底层实现时， Redis 使用 MurmurHash2 算法来计算键的哈希值。

MurmurHash 算法最初由 Austin Appleby 于 2008 年发明， 这种算法的优点在于， 即使输入的键是有规律的， 算法仍能给出一个很好的随机分布性， 并且算法的计算速度也非常快。

MurmurHash 算法目前的最新版本为 MurmurHash3 ， 而 Redis 使用的是 MurmurHash2 ， 关于 MurmurHash 算法的更多信息可以参考该算法的主页： http://code.google.com/p/smhasher/ 


- 解决键冲突

当有两个或以上数量的键被分配到了哈希表数组的同一个索引上面时， 我们称这些键发生了冲突（collision）。

Redis 的哈希表使用链地址法（separate chaining）来解决键冲突： 每个哈希表节点都有一个 next 指针， 多个哈希表节点可以用 next 指针构成一个单向链表， 被分配到同一个索引上的多个节点可以用这个单向链表连接起来， 这就解决了键冲突的问题。
举个例子， 假设程序要将键值对 k2 和 v2 添加到图 4-6 所示的哈希表里面， 并且计算得出 k2 的索引值为 2 ， 那么键 k1 和 k2 将产生冲突， 而解决冲突的办法就是使用 next 指针将键 k2 和 k1 所在的节点连接起来， 如图 4-7 所示。


![2015-09-13_55f512c4d7f99](https://user-images.githubusercontent.com/7789698/30953004-c158d1ca-a45c-11e7-9a67-c8ea8966fcc7.png)

![2015-09-13_55f512c2524e5](https://user-images.githubusercontent.com/7789698/30953007-c2a45824-a45c-11e7-9ac8-55f067e9898b.png)

因为 dictEntry 节点组成的链表没有指向链表表尾的指针， 所以为了速度考虑， 程序总是将新节点添加到链表的表头位置（复杂度为 O(1)）， 排在其他已有节点的前面。

- rehash
  随着操作的不断执行， 哈希表保存的键值对会逐渐地增多或者减少， 为了让哈希表的负载因子（load factor）维持在一个合理的范围之内， 当哈希表保存的键值对数量太多或者太少时， 程序需要对哈希表的大小进行相应的扩展或者收缩。

扩展和收缩哈希表的工作可以通过执行 rehash （重新散列）操作来完成， Redis 对字典的哈希表执行 rehash 的步骤如下：

为字典的 ht[1] 哈希表分配空间， 这个哈希表的空间大小取决于要执行的操作， 以及 ht[0] 当前包含的键值对数量 （也即是ht[0].used 属性的值）：
如果执行的是扩展操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used * 2 的 2^n （2 的 n 次方幂）；
如果执行的是收缩操作， 那么 ht[1] 的大小为第一个大于等于 ht[0].used 的 2^n 。
将保存在 ht[0] 中的所有键值对 rehash 到 ht[1] 上面： rehash 指的是重新计算键的哈希值和索引值， 然后将键值对放置到 ht[1] 哈希表的指定位置上。

当 ht[0] 包含的所有键值对都迁移到了 ht[1] 之后 （ht[0] 变为空表）， 释放 ht[0] ， 将 ht[1] 设置为 ht[0] ， 并在 ht[1] 新创建一个空白哈希表， 为下一次 rehash 做准备。

举个例子， 假设程序要对图 4-8 所示字典的 ht[0] 进行扩展操作， 那么程序将执行以下步骤：
ht[0].used 当前的值为 4 ， 4 * 2 = 8 ， 而 8 （2^3）恰好是第一个大于等于 4 的 2 的 n 次方， 所以程序会将 ht[1] 哈希表的大小设置为 8 。 图 4-9 展示了 ht[1] 在分配空间之后， 字典的样子。
将 ht[0] 包含的四个键值对都 rehash 到 ht[1] ， 如图 4-10 所示。
释放 ht[0] ，并将 ht[1] 设置为 ht[0] ，然后为 ht[1] 分配一个空白哈希表，如图 4-11 所示。
至此， 对哈希表的扩展操作执行完毕， 程序成功将哈希表的大小从原来的 4 改为了现在的 8 。

![2015-09-13_55f5130162f2d](https://user-images.githubusercontent.com/7789698/30953216-7d7ff108-a45d-11e7-9f90-ae3426492096.png)

![2015-09-13_55f51302b6785](https://user-images.githubusercontent.com/7789698/30953264-a6e83a0a-a45d-11e7-8849-c056d07b85a7.png)

![2015-09-13_55f51309b4775](https://user-images.githubusercontent.com/7789698/30953273-b5c7243c-a45d-11e7-8323-b8f25f2b5187.png)

![2015-09-13_55f5130b2ec57](https://user-images.githubusercontent.com/7789698/30953287-c35d0026-a45d-11e7-981d-273db0f853f8.png)


- 哈希表的扩展与收缩

当以下条件中的任意一个被满足时， 程序会自动开始对哈希表执行扩展操作：

服务器目前没有在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 1 ；
服务器目前正在执行 BGSAVE 命令或者 BGREWRITEAOF 命令， 并且哈希表的负载因子大于等于 5 ；
其中哈希表的负载因子可以通过公式：

```
# 负载因子 = 哈希表已保存节点数量 / 哈希表大小
load_factor = ht[0].used / ht[0].size
```

计算得出。

比如说， 对于一个大小为 4 ， 包含 4 个键值对的哈希表来说， 这个哈希表的负载因子为：
`load_factor = 4 / 4 = 1`

又比如说， 对于一个大小为 512 ， 包含 256 个键值对的哈希表来说， 这个哈希表的负载因子为：
`load_factor = 256 / 512 = 0.5`

根据 BGSAVE 命令或 BGREWRITEAOF 命令是否正在执行， 服务器执行扩展操作所需的负载因子并不相同， 这是因为在执行 BGSAVE 命令或BGREWRITEAOF 命令的过程中， Redis 需要创建当前服务器进程的子进程， 而大多数操作系统都采用写时复制（copy-on-write）技术来优化子进程的使用效率， 所以在子进程存在期间， 服务器会提高执行扩展操作所需的负载因子， 从而尽可能地避免在子进程存在期间进行哈希表扩展操作， 这可以避免不必要的内存写入操作， 最大限度地节约内存。
另一方面， 当哈希表的负载因子小于 0.1 时， 程序自动开始对哈希表执行收缩操作。

- 渐进式 rehash
  上一节说过， 扩展或收缩哈希表需要将 ht[0] 里面的所有键值对 rehash 到 ht[1] 里面， 但是， 这个 rehash 动作并不是一次性、集中式地完成的， 而是分多次、渐进式地完成的。

这样做的原因在于， 如果 ht[0] 里只保存着四个键值对， 那么服务器可以在瞬间就将这些键值对全部 rehash 到 ht[1] ； 但是， 如果哈希表里保存的键值对数量不是四个， 而是四百万、四千万甚至四亿个键值对， 那么要一次性将这些键值对全部 rehash 到 ht[1] 的话， 庞大的计算量可能会导致服务器在一段时间内停止服务。

因此， 为了避免 rehash 对服务器性能造成影响， 服务器不是一次性将 ht[0] 里面的所有键值对全部 rehash 到 ht[1] ， 而是分多次、渐进式地将 ht[0] 里面的键值对慢慢地 rehash 到 ht[1] 。
以下是哈希表渐进式 rehash 的详细步骤：

1. 为 ht[1] 分配空间， 让字典同时持有 ht[0] 和 ht[1] 两个哈希表。
2. 在字典中维持一个索引计数器变量 rehashidx ， 并将它的值设置为 0 ， 表示 rehash 工作正式开始。
3. 在 rehash 进行期间， 每次对字典执行添加、删除、查找或者更新操作时， 程序除了执行指定的操作以外， 还会顺带将 ht[0] 哈希表在 rehashidx 索引上的所有键值对 rehash 到 ht[1] ， 当 rehash 工作完成之后， 程序将 rehashidx 属性的值增一。
4. 随着字典操作的不断执行， 最终在某个时间点上， ht[0] 的所有键值对都会被 rehash 至 ht[1] ， 这时程序将 rehashidx 属性的值设为 -1 ， 表示 rehash 操作已完成。
  渐进式 rehash 的好处在于它采取分而治之的方式， 将 rehash 键值对所需的计算工作均滩到对字典的每个添加、删除、查找和更新操作上， 从而避免了集中式 rehash 而带来的庞大计算量。

https://www.kancloud.cn/kancloud/redisbook/63842

<img width="957" alt="wx20170928-150614 2x" src="https://user-images.githubusercontent.com/7789698/30953489-c2f6d85e-a45e-11e7-97e7-543618fad97a.png">


# 跳跃表
跳跃表（skiplist）是一种有序数据结构， 它通过在每个节点中维持多个指向其他节点的指针， 从而达到快速访问节点的目的。

跳跃表支持平均 O(\log N) 最坏 O(N) 复杂度的节点查找， 还可以通过顺序性操作来批量处理节点。
在大部分情况下， 跳跃表的效率可以和平衡树相媲美， 并且因为跳跃表的实现比平衡树要来得更为简单， 所以有不少程序都使用跳跃表来代替平衡树。

Redis 使用跳跃表作为有序集合键的底层实现之一： 如果一个有序集合包含的元素数量比较多， 又或者有序集合中元素的成员（member）是比较长的字符串时， Redis 就会使用跳跃表来作为有序集合键的底层实现。

举个例子， fruit-price 是一个有序集合键， 这个有序集合以水果名为成员， 水果价钱为分值， 保存了 130 款水果的价钱：

```
redis> ZRANGE fruit-price 0 2 WITHSCORES
1) "banana"
2) "5"
3) "cherry"
4) "6.5"
5) "apple"
6) "8"

redis> ZCARD fruit-price
(integer) 130
```

fruit-price 有序集合的所有数据都保存在一个跳跃表里面， 其中每个跳跃表节点（node）都保存了一款水果的价钱信息， 所有水果按价钱的高低从低到高在跳跃表里面排序：

和链表、字典等数据结构被广泛地应用在 Redis 内部不同， Redis 只在两个地方用到了跳跃表， 一个是实现有序集合键， 另一个是在集群节点中用作内部数据结构， 除此之外， 跳跃表在 Redis 里面没有其他用途。

Redis 的跳跃表由 redis.h/zskiplistNode 和 redis.h/zskiplist 两个结构定义， 其中 zskiplistNode 结构用于表示跳跃表节点， 而 zskiplist结构则用于保存跳跃表节点的相关信息， 比如节点的数量， 以及指向表头节点和表尾节点的指针， 等等。

![2015-09-13_55f51478611a6](https://user-images.githubusercontent.com/7789698/30953604-36ae23ba-a45f-11e7-9264-b9b9ba6565dd.png)

图 5-1 展示了一个跳跃表示例， 位于图片最左边的是 zskiplist 结构， 该结构包含以下属性：
header ：指向跳跃表的表头节点。
tail ：指向跳跃表的表尾节点。
level ：记录目前跳跃表内，层数最大的那个节点的层数（表头节点的层数不计算在内）。
length ：记录跳跃表的长度，也即是，跳跃表目前包含节点的数量（表头节点不计算在内）。
位于 zskiplist 结构右方的是四个 zskiplistNode 结构， 该结构包含以下属性：
层（level）：节点中用 L1 、 L2 、 L3 等字样标记节点的各个层， L1 代表第一层， L2 代表第二层，以此类推。每个层都带有两个属性：前进指针和跨度。前进指针用于访问位于表尾方向的其他节点，而跨度则记录了前进指针所指向节点和当前节点的距离。在上面的图片中，连线上带有数字的箭头就代表前进指针，而那个数字就是跨度。当程序从表头向表尾进行遍历时，访问会沿着层的前进指针进行。
后退（backward）指针：节点中用 BW 字样标记节点的后退指针，它指向位于当前节点的前一个节点。后退指针在程序从表尾向表头遍历时使用。
分值（score）：各个节点中的 1.0 、 2.0 和 3.0 是节点所保存的分值。在跳跃表中，节点按各自所保存的分值从小到大排列。
成员对象（obj）：各个节点中的 o1 、 o2 和 o3 是节点所保存的成员对象。
注意表头节点和其他节点的构造是一样的： 表头节点也有后退指针、分值和成员对象， 不过表头节点的这些属性都不会被用到， 所以图中省略了这些部分， 只显示了表头节点的各个层。


```
typedef struct zskiplistNode {

    // 后退指针
    struct zskiplistNode *backward;

    // 分值
    double score;

    // 成员对象
    robj *obj;

    // 层
    struct zskiplistLevel {

        // 前进指针
        struct zskiplistNode *forward;

        // 跨度
        unsigned int span;

    } level[];

} zskiplistNode;
```

- 层
  跳跃表节点的 level 数组可以包含多个元素， 每个元素都包含一个指向其他节点的指针， 程序可以通过这些层来加快访问其他节点的速度， 一般来说， 层的数量越多， 访问其他节点的速度就越快。

每次创建一个新跳跃表节点的时候， 程序都根据幂次定律 （power law，越大的数出现的概率越小） 随机生成一个介于 1 和 32 之间的值作为 level 数组的大小， 这个大小就是层的“高度”。



- 前进指针

每个层都有一个指向表尾方向的前进指针（level[i].forward 属性）， 用于从表头向表尾方向访问节点。
图 5-3 用虚线表示出了程序从表头向表尾方向， 遍历跳跃表中所有节点的路径：

1.迭代程序首先访问跳跃表的第一个节点（表头）， 然后从第四层的前进指针移动到表中的第二个节点。
2.在第二个节点时， 程序沿着第二层的前进指针移动到表中的第三个节点。
3.在第三个节点时， 程序同样沿着第二层的前进指针移动到表中的第四个节点。
4.当程序再次沿着第四个节点的前进指针移动时， 它碰到一个 NULL ， 程序知道这时已经到达了跳跃表的表尾， 于是结束这次遍历。

- 跨度
  层的跨度（level[i].span 属性）用于记录两个节点之间的距离：
  两个节点之间的跨度越大， 它们相距得就越远。
  指向 NULL 的所有前进指针的跨度都为 0 ， 因为它们没有连向任何节点。
  初看上去， 很容易以为跨度和遍历操作有关， 但实际上并不是这样 —— 遍历操作只使用前进指针就可以完成了， 跨度实际上是用来计算排位（rank）的： 在查找某个节点的过程中， 将沿途访问过的所有层的跨度累计起来， 得到的结果就是目标节点在跳跃表中的排位。

举个例子， 图 5-4 用虚线标记了在跳跃表中查找分值为 3.0 、 成员对象为 o3 的节点时， 沿途经历的层： 查找的过程只经过了一个层， 并且层的跨度为 3 ， 所以目标节点在跳跃表中的排位为 3 。

- 后退指针

节点的后退指针（backward 属性）用于从表尾向表头方向访问节点： 跟可以一次跳过多个节点的前进指针不同， 因为每个节点只有一个后退指针， 所以每次只能后退至前一个节点。
图 5-6 用虚线展示了如果从表尾向表头遍历跳跃表中的所有节点： 程序首先通过跳跃表的 tail 指针访问表尾节点， 然后通过后退指针访问倒数第二个节点， 之后再沿着后退指针访问倒数第三个节点， 再之后遇到指向 NULL 的后退指针， 于是访问结束。

- 分值和成员
  节点的分值（score 属性）是一个 double 类型的浮点数， 跳跃表中的所有节点都按分值从小到大来排序。
  节点的成员对象（obj 属性）是一个指针， 它指向一个字符串对象， 而字符串对象则保存着一个 SDS 值。
  在同一个跳跃表中， 各个节点保存的成员对象必须是唯一的， 但是多个节点保存的分值却可以是相同的： 分值相同的节点将按照成员对象在字典序中的大小来进行排序， 成员对象较小的节点会排在前面（靠近表头的方向）， 而成员对象较大的节点则会排在后面（靠近表尾的方向）。

举个例子， 在图 5-7 所示的跳跃表中， 三个跳跃表节点都保存了相同的分值 10086.0 ， 但保存成员对象 o1 的节点却排在保存成员对象 o2和 o3 的节点之前， 而保存成员对象 o2 的节点又排在保存成员对象 o3 的节点之前， 由此可见， o1 、 o2 、 o3 三个成员对象在字典中的排序为 o1 <= o2 <= o3 。

- 跳跃表

```
typedef struct zskiplist {

    // 表头节点和表尾节点
    struct zskiplistNode *header, *tail;

    // 表中节点的数量
    unsigned long length;

    // 表中层数最大的节点的层数
    int level;

} zskiplist;
```

# 整数集合

整数集合（intset）是集合键的底层实现之一： 当一个集合只包含整数值元素， 并且这个集合的元素数量不多时， Redis 就会使用整数集合作为集合键的底层实现。

```
redis> SADD numbers 1 3 5 7 9
(integer) 5

redis> OBJECT ENCODING numbers
"intset"
```

```
typedef struct intset {

    // 编码方式
    uint32_t encoding;

    // 集合包含的元素数量
    uint32_t length;

    // 保存元素的数组
    int8_t contents[];

} intset;
```

contents 数组是整数集合的底层实现： 整数集合的每个元素都是 contents 数组的一个数组项（item）， 各个项在数组中按值的大小从小到大有序地排列， 并且数组中不包含任何重复项。

length 属性记录了整数集合包含的元素数量， 也即是 contents 数组的长度。

虽然 intset 结构将 contents 属性声明为 int8_t 类型的数组， 但实际上 contents 数组并不保存任何 int8_t 类型的值 —— contents 数组的真正类型取决于 encoding 属性的值：

1. 如果 encoding 属性的值为 INTSET_ENC_INT16 ， 那么 contents 就是一个 int16_t 类型的数组， 数组里的每个项都是一个 int16_t 类型的整数值 （最小值为 -32,768 ，最大值为 32,767 ）。
2. 如果 encoding 属性的值为 INTSET_ENC_INT32 ， 那么 contents 就是一个 int32_t 类型的数组， 数组里的每个项都是一个 int32_t 类型的整数值 （最小值为 -2,147,483,648 ，最大值为 2,147,483,647 ）。
3. 如果 encoding 属性的值为 INTSET_ENC_INT64 ， 那么 contents 就是一个 int64_t 类型的数组， 数组里的每个项都是一个 int64_t 类型的整数值 （最小值为 -9,223,372,036,854,775,808 ，最大值为 9,223,372,036,854,775,807 ）。

4. encoding 属性的值为 INTSET_ENC_INT16 ， 表示整数集合的底层实现为 int16_t 类型的数组， 而集合保存的都是 int16_t 类型的整数值。
5. length 属性的值为 5 ， 表示整数集合包含五个元素。
6. contents 数组按从小到大的顺序保存着集合中的五个元素。
7. 因为每个集合元素都是 int16_t 类型的整数值， 所以 contents 数组的大小等于 sizeof(int16_t) * 5 = 16 * 5 = 80 位。

![2015-09-13_55f51a1132a4b](https://user-images.githubusercontent.com/7789698/30954135-a2993f9a-a461-11e7-9eae-d5c8ea98f996.png)


1. encoding 属性的值为 INTSET_ENC_INT64 ， 表示整数集合的底层实现为 int64_t 类型的数组， 而数组中保存的都是 int64_t 类型的整数值。
2. length 属性的值为 4 ， 表示整数集合包含四个元素。
3. contents 数组按从小到大的顺序保存着集合中的四个元素。
4. 因为每个集合元素都是 int64_t 类型的整数值， 所以 contents 数组的大小为 sizeof(int64_t) * 4 = 64 * 4 = 256 位。

![2015-09-13_55f51a139a148](https://user-images.githubusercontent.com/7789698/30954158-baaaab46-a461-11e7-9cb2-8886e331b927.png)


虽然 contents 数组保存的四个整数值中， 只有 -2675256175807981027 是真正需要用 int64_t 类型来保存的， 而其他的 1 、 3 、 5 三个值都可以用 int16_t 类型来保存， 不过根据整数集合的升级规则， 当向一个底层为 int16_t 数组的整数集合添加一个 int64_t 类型的整数值时， 整数集合已有的所有元素都会被转换成 int64_t 类型， 所以 contents 数组保存的四个整数值都是 int64_t 类型的， 不仅仅是-2675256175807981027 。

- 升级
  每当我们要将一个新元素添加到整数集合里面， 并且新元素的类型比整数集合现有所有元素的类型都要长时， 整数集合需要先进行升级（upgrade）， 然后才能将新元素添加到整数集合里面。

升级整数集合并添加新元素共分为三步进行：

1. 根据新元素的类型， 扩展整数集合底层数组的空间大小， 并为新元素分配空间。
2. 将底层数组现有的所有元素都转换成与新元素相同的类型， 并将类型转换后的元素放置到正确的位上， 而且在放置元素的过程中， 需要继续维持底层数组的有序性质不变。
3. 将新元素添加到底层数组里面。

举个例子， 假设现在有一个 INTSET_ENC_INT16 编码的整数集合， 集合中包含三个 int16_t 类型的元素， 如图 6-3 所示。


因为每个元素都占用 16 位空间， 所以整数集合底层数组的大小为 3 * 16 = 48 位， 图 6-4 展示了整数集合的三个元素在这 48 位里的位置。

![2015-09-13_55f51ab6dbe72](https://user-images.githubusercontent.com/7789698/30954310-404deb50-a462-11e7-8bb7-560f7a946b1e.png)

现在， 假设我们要将类型为 int32_t 的整数值 65535 添加到整数集合里面， 因为 65535 的类型 int32_t 比整数集合当前所有元素的类型都要长， 所以在将 65535 添加到整数集合之前， 程序需要先对整数集合进行升级。
升级首先要做的是， 根据新类型的长度， 以及集合元素的数量（包括要添加的新元素在内）， 对底层数组进行空间重分配。

整数集合目前有三个元素， 再加上新元素 65535 ， 整数集合需要分配四个元素的空间， 因为每个 int32_t 整数值需要占用 32 位空间， 所以在空间重分配之后， 底层数组的大小将是 32 * 4 = 128 位， 如图 6-5 所示

![2015-09-13_55f51abfde71b](https://user-images.githubusercontent.com/7789698/30954327-545efdfa-a462-11e7-901f-ef8ea9b90d37.png)

虽然程序对底层数组进行了空间重分配， 但数组原有的三个元素 1 、 2 、 3 仍然是 int16_t 类型， 这些元素还保存在数组的前 48 位里面， 所以程序接下来要做的就是将这三个元素转换成 int32_t 类型， 并将转换后的元素放置到正确的位上面， 而且在放置元素的过程中， 需要维持底层数组的有序性质不变。
首先， 因为元素 3 在 1 、 2 、 3 、 65535 四个元素中排名第三， 所以它将被移动到 contents 数组的索引 2 位置上， 也即是数组 64 位至 95 位的空间内， 如图 6-6 所示。

![2015-09-13_55f51ac25b6a0](https://user-images.githubusercontent.com/7789698/30954364-7456234a-a462-11e7-9d9c-b775c31ebbf5.png)

接着， 因为元素 2 在 1 、 2 、 3 、 65535 四个元素中排名第二， 所以它将被移动到 contents 数组的索引 1 位置上， 也即是数组的 32位至 63 位的空间内， 如图 6-7 所示。

![2015-09-13_55f51ac353bfa](https://user-images.githubusercontent.com/7789698/30954458-d2490634-a462-11e7-8dbb-9f61d8058d17.png)


之后， 因为元素 1 在 1 、 2 、 3 、 65535 四个元素中排名第一， 所以它将被移动到 contents 数组的索引 0 位置上， 也即是数组的 0 位至 31 位的空间内， 如图 6-8 所示。
![2015-09-13_55f51ac466154](https://user-images.githubusercontent.com/7789698/30954472-e529b758-a462-11e7-92d3-eb843fa11494.png)


然后， 因为元素 65535 在 1 、 2 、 3 、 65535 四个元素中排名第四， 所以它将被添加到 contents 数组的索引 3 位置上， 也即是数组的96 位至 127 位的空间内， 如图 6-9 所示。
![2015-09-13_55f51acae92db](https://user-images.githubusercontent.com/7789698/30954508-feab27f2-a462-11e7-9310-a8f06db5899c.png)


最后， 程序将整数集合 encoding 属性的值从 INTSET_ENC_INT16 改为 INTSET_ENC_INT32 ， 并将 length 属性的值从 3 改为 4 ， 设置完成之后的整数集合如图 6-10 所示。
![2015-09-13_55f51ae117896](https://user-images.githubusercontent.com/7789698/30954520-07d5e196-a463-11e7-833c-a950e122e357.png)


因为每次向整数集合添加新元素都可能会引起升级， 而每次升级都需要对底层数组中已有的所有元素进行类型转换， 所以向整数集合添加新元素的时间复杂度为 O(N) 

其他类型的升级操作， 比如从 INTSET_ENC_INT16 编码升级为 INTSET_ENC_INT64 编码， 或者从 INTSET_ENC_INT32 编码升级为 INTSET_ENC_INT64 编码， 升级的过程都和上面展示的升级过程类似。


升级之后新元素的摆放位置

因为引发升级的新元素的长度总是比整数集合现有所有元素的长度都大， 所以这个新元素的值要么就大于所有现有元素， 要么就小于所有现有元素：

在新元素小于所有现有元素的情况下， 新元素会被放置在底层数组的最开头（索引 0 ）；
在新元素大于所有现有元素的情况下， 新元素会被放置在底层数组的最末尾（索引 length-1 ）。

- 降级
  整数集合不支持降级操作， 一旦对数组进行了升级， 编码就会一直保持升级后的状态。

<img width="957" alt="wx20170928-153940 2x" src="https://user-images.githubusercontent.com/7789698/30954573-44344f1a-a463-11e7-9937-f202bd13f84b.png">


# 压缩列表
当一个列表键只包含少量列表项， 并且每个列表项要么就是小整数值， 要么就是长度比较短的字符串， 那么 Redis 就会使用压缩列表来做列表键的底层实现。

```
redis> RPUSH lst 1 3 5 10086 "hello" "world"
(integer) 6

redis> OBJECT ENCODING lst
"ziplist"
```

另外， 当一个哈希键只包含少量键值对， 并且每个键值对的键和值要么就是小整数值， 要么就是长度比较短的字符串， 那么 Redis 就会使用压缩列表来做哈希键的底层实现。

举个例子， 执行以下命令将创建一个压缩列表实现的哈希键：

```
redis> HMSET profile "name" "Jack" "age" 28 "job" "Programmer"
OK

redis> OBJECT ENCODING profile
"ziplist"
```

![2015-09-13_55f51bfccbd83](https://user-images.githubusercontent.com/7789698/30954687-abdd05a8-a463-11e7-81d7-23810c2e3c94.png)

<img width="955" alt="wx20170928-154257 2x" src="https://user-images.githubusercontent.com/7789698/30954715-c2a1bb30-a463-11e7-8ef3-a356a3953cc9.png">

1. 列表 zlbytes 属性的值为 0x50 （十进制 80）， 表示压缩列表的总长为 80 字节。
2. 列表 zltail 属性的值为 0x3c （十进制 60）， 这表示如果我们有一个指向压缩列表起始地址的指针 p ， 那么只要用指针 p 加上偏移量 60 ， 就可以计算出表尾节点 entry3 的地址。
3. 列表 zllen 属性的值为 0x3 （十进制 3）， 表示压缩列表包含三个节点。


- 压缩列表节点的构成

每个压缩列表节点可以保存一个字节数组或者一个整数值， 其中， 字节数组可以是以下三种长度的其中一种：

1. 长度小于等于 63 （2^{6}-1）字节的字节数组；
2. 长度小于等于 16383 （2^{14}-1） 字节的字节数组；
3. 长度小于等于 4294967295 （2^{32}-1）字节的字节数组；

而整数值则可以是以下六种长度的其中一种：

1. 4 位长，介于 0 至 12 之间的无符号整数；
2. 1 字节长的有符号整数；
3. 3 字节长的有符号整数；
4. int16_t 类型整数；
5. int32_t 类型整数；
6. int64_t 类型整数。

每个压缩列表节点都由 previous_entry_length 、 encoding 、 content 三个部分组成

- previous_entry_length

节点的 previous_entry_length 属性以字节为单位， 记录了压缩列表中前一个节点的长度。
previous_entry_length 属性的长度可以是 1 字节或者 5 字节：
1. 如果前一节点的长度小于 254 字节， 那么 previous_entry_length 属性的长度为 1 字节： 前一节点的长度就保存在这一个字节里面。
2. 如果前一节点的长度大于等于 254 字节， 那么 previous_entry_length 属性的长度为 5 字节： 其中属性的第一字节会被设置为 0xFE（十进制值 254）， 而之后的四个字节则用于保存前一节点的长度。

因为节点的 previous_entry_length 属性记录了前一个节点的长度， 所以程序可以通过指针运算， 根据当前节点的起始地址来计算出前一个节点的起始地址。

压缩列表的从表尾向表头遍历操作就是使用这一原理实现的： 只要我们拥有了一个指向某个节点起始地址的指针， 那么通过这个指针以及这个节点的 previous_entry_length 属性， 程序就可以一直向前一个节点回溯， 最终到达压缩列表的表头节点。
图 7-8 展示了一个从表尾节点向表头节点进行遍历的完整过程：
1. 首先，我们拥有指向压缩列表表尾节点 entry4 起始地址的指针 p1 （指向表尾节点的指针可以通过指向压缩列表起始地址的指针加上zltail 属性的值得出）；
2. 通过用 p1 减去 entry4 节点 previous_entry_length 属性的值， 我们得到一个指向 entry4 前一节点 entry3 起始地址的指针 p2 ；
3. 通过用 p2 减去 entry3 节点 previous_entry_length 属性的值， 我们得到一个指向 entry3 前一节点 entry2 起始地址的指针 p3 ；
4. 通过用 p3 减去 entry2 节点 previous_entry_length 属性的值， 我们得到一个指向 entry2 前一节点 entry1 起始地址的指针 p4 ， entry1为压缩列表的表头节点；
5. 最终， 我们从表尾节点向表头节点遍历了整个列表。



- encoding

节点的 encoding 属性记录了节点的 content 属性所保存数据的类型以及长度：
1. 一字节、两字节或者五字节长， 值的最高位为 00 、 01 或者 10 的是字节数组编码： 这种编码表示节点的 content 属性保存着字节数组， 数组的长度由编码除去最高两位之后的其他位记录；
2. 一字节长， 值的最高位以 11 开头的是整数编码： 这种编码表示节点的 content 属性保存着整数值， 整数值的类型和长度由编码除去最高两位之后的其他位记录；

<img width="963" alt="wx20170928-155056 2x" src="https://user-images.githubusercontent.com/7789698/30955209-67be740e-a465-11e7-81d5-727da5f34484.png">


- content


编码的最高两位 00 表示节点保存的是一个字节数组；
编码的后六位 001011 记录了字节数组的长度 11 ；
content 属性保存着节点的值 "hello world" 。

编码 11000000 表示节点保存的是一个 int16_t 类型的整数值；
content 属性保存着节点的值 10086 。


- 连锁更新
  https://www.kancloud.cn/kancloud/redisbook/63859

# 对象

```
typedef struct redisObject {

    // 类型
    unsigned type:4;

    // 编码
    unsigned encoding:4;

    // 指向底层实现数据结构的指针
    void *ptr;

    // ...

} robj;
```

- 类型

| 类型常量     | 对象的名称   |
| ------------ | ------------ |
| REDIS_STRING | 字符串对象   |
| REDIS_LIST   | 列表对象     |
| REDIS_HASH   | 哈希对象     |
| REDIS_SET    | 集合对象     |
| REDIS_ZSET   | 有序集合对象 |



```
# 键为字符串对象，值为字符串对象

redis> SET msg "hello world"
OK

redis> TYPE msg
string

# 键为字符串对象，值为列表对象

redis> RPUSH numbers 1 3 5
(integer) 6

redis> TYPE numbers
list

# 键为字符串对象，值为哈希对象

redis> HMSET profile name Tome age 25 career Programmer
OK

redis> TYPE profile
hash

# 键为字符串对象，值为集合对象

redis> SADD fruits apple banana cherry
(integer) 3

redis> TYPE fruits
set

# 键为字符串对象，值为有序集合对象

redis> ZADD price 8.5 apple 5.0 banana 6.0 cherry
(integer) 3

redis> TYPE price
zset
```


| 对象         | 对象 type 属性的值 | TYPE 命令的输出 |
| ------------ | ------------------ | --------------- |
| 字符串对象   | REDIS_STRING       | "string"        |
| 列表对象     | REDIS_LIST         | "list"          |
| 哈希对象     | REDIS_HASH         | "hash"          |
| 集合对象     | REDIS_SET          | "set"           |
| 有序集合对象 | REDIS_ZSET         | "zset"          |

- 编码和底层实现

| 编码常量                  | 编码所对应的底层数据结构    |
| ------------------------- | --------------------------- |
| REDIS_ENCODING_INT        | long 类型的整数             |
| REDIS_ENCODING_EMBSTR     | embstr 编码的简单动态字符串 |
| REDIS_ENCODING_RAW        | 简单动态字符串              |
| REDIS_ENCODING_HT         | 字典                        |
| REDIS_ENCODING_LINKEDLIST | 双端链表                    |
| REDIS_ENCODING_ZIPLIST    | 压缩列表                    |
| REDIS_ENCODING_INTSET     | 整数集合                    |
| REDIS_ENCODING_SKIPLIST   | 跳跃表和字典                |


| 类型         | 编码                      | 对象                                               |
| ------------ | ------------------------- | -------------------------------------------------- |
| REDIS_STRING | REDIS_ENCODING_INT        | 使用整数值实现的字符串对象。                       |
| REDIS_STRING | REDIS_ENCODING_EMBSTR     | 使用 embstr 编码的简单动态字符串实现的字符串对象。 |
| REDIS_STRING | REDIS_ENCODING_RAW        | 使用简单动态字符串实现的字符串对象。               |
| REDIS_LIST   | REDIS_ENCODING_ZIPLIST    | 使用压缩列表实现的列表对象。                       |
| REDIS_LIST   | REDIS_ENCODING_LINKEDLIST | 使用双端链表实现的列表对象。                       |
| REDIS_HASH   | REDIS_ENCODING_ZIPLIST    | 使用压缩列表实现的哈希对象。                       |
| REDIS_HASH   | REDIS_ENCODING_HT         | 使用字典实现的哈希对象。                           |
| REDIS_SET    | REDIS_ENCODING_INTSET     | 使用整数集合实现的集合对象。                       |
| REDIS_SET    | REDIS_ENCODING_HT         | 使用字典实现的集合对象。                           |
| REDIS_ZSET   | REDIS_ENCODING_ZIPLIST    | 使用压缩列表实现的有序集合对象。                   |
| REDIS_ZSET   | REDIS_ENCODING_SKIPLIST   | 使用跳跃表和字典实现的有序集合对象。               |



```
redis> SET msg "hello wrold"
OK

redis> OBJECT ENCODING msg
"embstr"

redis> SET story "long long long long long long ago ..."
OK

redis> OBJECT ENCODING story
"raw"

redis> SADD numbers 1 3 5
(integer) 3

redis> OBJECT ENCODING numbers
"intset"

redis> SADD numbers "seven"
(integer) 1

redis> OBJECT ENCODING numbers
"hashtable"
```


| 对象所使用的底层数据结构           | 编码常量                  | OBJECT ENCODING 命令输出 |
| ---------------------------------- | ------------------------- | ------------------------ |
| 整数                               | REDIS_ENCODING_INT        | "int"                    |
| embstr 编码的简单动态字符串（SDS） | REDIS_ENCODING_EMBSTR     | "embstr"                 |
| 简单动态字符串                     | REDIS_ENCODING_RAW        | "raw"                    |
| 字典                               | REDIS_ENCODING_HT         | "hashtable"              |
| 双端链表                           | REDIS_ENCODING_LINKEDLIST | "linkedlist"             |
| 压缩列表                           | REDIS_ENCODING_ZIPLIST    | "ziplist"                |
| 整数集合                           | REDIS_ENCODING_INTSET     | "intset"                 |
| 跳跃表和字典                       | REDIS_ENCODING_SKIPLIST   | "skiplist"               |

https://www.kancloud.cn/kancloud/redisbook/63862

- 内存回收
  Redis 在自己的对象系统中构建了一个引用计数（reference counting）技术实现的内存回收机制

```
typedef struct redisObject {

    // ...

    // 引用计数
    int refcount;

    // ...

} robj;
```

1. 在创建一个新对象时， 引用计数的值会被初始化为 1 ；
2. 当对象被一个新程序使用时， 它的引用计数值会被增一；
3. 当对象不再被一个程序使用时， 它的引用计数值会被减一；
4. 当对象的引用计数值变为 0 时， 对象所占用的内存会被释放。



| 函数          | 作用                                                         |
| ------------- | ------------------------------------------------------------ |
| incrRefCount  | 将对象的引用计数值增一。                                     |
| decrRefCount  | 将对象的引用计数值减一， 当对象的引用计数值等于 0 时， 释放对象。 |
| resetRefCount | 将对象的引用计数值设置为 0 ， 但并不释放对象， 这个函数通常在需要重新设置对象的引用计数值时使用。 |

- 对象共享
   对象的引用计数属性还带有对象共享的作用

Redis 只对包含整数值的字符串对象进行共享。

- 对象的空转时长

除了前面介绍过的 type 、 encoding 、 ptr 和 refcount 四个属性之外， redisObject 结构包含的最后一个属性为 lru 属性， 该属性记录了对象最后一次被命令程序访问的时间：

```
typedef struct redisObject {

    // ...

    unsigned lru:22;

    // ...

} robj;
```

# 单机数据库的实现
## 数据库
- 数据库键空间
  Redis 是一个键值对（key-value pair）数据库服务器， 服务器中的每个数据库都由一个 redis.h/redisDb 结构表示， 其中， redisDb 结构的dict 字典保存了数据库中的所有键值对， 我们将这个字典称为键空间（key space）：

```
typedef struct redisDb {

    // ...

    // 数据库键空间，保存着数据库中的所有键值对
    dict *dict;

    // ...

} redisDb;
```

- 添加新键
  添加一个新键值对到数据库， 实际上就是将一个新键值对添加到键空间字典里面， 其中键为字符串对象， 而值则为任意一种类型的 Redis 对象。



```
redis> SET message "hello world"
OK

redis> RPUSH alphabet "a" "b" "c"
(integer) 3

redis> HSET book name "Redis in Action"
(integer) 1

redis> HSET book author "Josiah L. Carlson"
(integer) 1

redis> HSET book publisher "Manning"
(integer) 1
```

```
redis> SET date "2013.12.1"
OK
```

![2015-09-13_55f5227cb91e3](https://user-images.githubusercontent.com/7789698/30955855-c032d600-a467-11e7-8699-9c88744eba2a.png)

- 删除键
  删除数据库中的一个键， 实际上就是在键空间里面删除键所对应的键值对对象。

- 更新键
  对一个数据库键进行更新， 实际上就是对键空间里面键所对应的值对象进行更新， 根据值对象的类型不同， 更新的具体方法也会有所不同。

- 对键取值


- 读写键空间时的维护操作
  当使用 Redis 命令对数据库进行读写时， 服务器不仅会对键空间执行指定的读写操作， 还会执行一些额外的维护操作， 其中包括：

- 在读取一个键之后（读操作和写操作都要对键进行读取）， 服务器会根据键是否存在， 以此来更新服务器的键空间命中（hit）次数或键空间不命中（miss）次数， 这两个值可以在 INFO stats 命令的 keyspace_hits 属性和 keyspace_misses 属性中查看。
- 在读取一个键之后， 服务器会更新键的 LRU （最后一次使用）时间， 这个值可以用于计算键的闲置时间， 使用命令 OBJECT idletime  命令可以查看键 key 的闲置时间。
- 如果服务器在读取一个键时， 发现该键已经过期， 那么服务器会先删除这个过期键， 然后才执行余下的其他操作， 本章稍后对过期键的讨论会详细说明这一点。
- 如果有客户端使用 WATCH 命令监视了某个键， 那么服务器在对被监视的键进行修改之后， 会将这个键标记为脏（dirty）， 从而让事务程序注意到这个键已经被修改过， 《事务》一章会详细说明这一点。
- 服务器每次修改一个键之后， 都会对脏（dirty）键计数器的值增一， 这个计数器会触发服务器的持久化以及复制操作执行， 《RDB 持久化》、《AOF 持久化》和《复制》这三章都会说到这一点。
- 如果服务器开启了数据库通知功能， 那么在对键进行修改之后， 服务器将按配置发送相应的数据库通知， 本章稍后讨论数据库通知功能的实现时会详细说明这一点。


- Redis 服务器的所有数据库都保存在 redisServer.db 数组中， 而数据库的数量则由 redisServer.dbnum 属性保存。
- 客户端通过修改目标数据库指针， 让它指向 redisServer.db 数组中的不同元素来切换不同的数据库。
- 数据库主要由 dict 和 expires 两个字典构成， 其中 dict 字典负责保存键值对， 而 expires 字典则负责保存键的过期时间。
- 因为数据库由字典构成， 所以对数据库的操作都是建立在字典操作之上的。
- 数据库的键总是一个字符串对象， 而值则可以是任意一种 Redis 对象类型， 包括字符串对象、哈希表对象、集合对象、列表对象和有序集合对象， 分别对应字符串键、哈希表键、集合键、列表键和有序集合键。
- expires 字典的键指向数据库中的某个键， 而值则记录了数据库键的过期时间， 过期时间是一个以毫秒为单位的 UNIX 时间戳。
- Redis 使用惰性删除和定期删除两种策略来删除过期的键： 惰性删除策略只在碰到过期键时才进行删除操作， 定期删除策略则每隔一段时间， 主动查找并删除过期键。
- 执行 SAVE 命令或者 BGSAVE 命令所产生的新 RDB 文件不会包含已经过期的键。
- 执行 BGREWRITEAOF 命令所产生的重写 AOF 文件不会包含已经过期的键。
- 当一个过期键被删除之后， 服务器会追加一条 DEL 命令到现有 AOF 文件的末尾， 显式地删除过期键。
- 当主服务器删除一个过期键之后， 它会向所有从服务器发送一条 DEL 命令， 显式地删除过期键。
- 从服务器即使发现过期键， 也不会自作主张地删除它， 而是等待主节点发来 DEL 命令， 这种统一、中心化的过期键删除策略可以保证主从服务器数据的一致性。
- 当 Redis 命令对数据库进行修改之后， 服务器会根据配置， 向客户端发送数据库通知。

## RDB 持久化

- RDB 文件结构
  ![2015-09-13_55f523462fa07](https://user-images.githubusercontent.com/7789698/30956342-98488282-a469-11e7-919d-d0b5af2a405d.png)

RDB 文件的最开头是 REDIS 部分， 这个部分的长度为 5 字节， 保存着 "REDIS" 五个字符。 通过这五个字符， 程序可以在载入文件时， 快速检查所载入的文件是否 RDB 文件。

db_version 长度为 4 字节， 它的值是一个字符串表示的整数， 这个整数记录了 RDB 文件的版本号， 比如 "0006" 就代表 RDB 文件的版本为第六版。

databases 部分包含着零个或任意多个数据库， 以及各个数据库中的键值对数据：

- 如果服务器的数据库状态为空（所有数据库都是空的）， 那么这个部分也为空， 长度为 0 字节。
- 如果服务器的数据库状态为非空（有至少一个数据库非空）， 那么这个部分也为非空， 根据数据库所保存键值对的数量、类型和内容不同， 这个部分的长度也会有所不同。


EOF 常量的长度为 1 字节， 这个常量标志着 RDB 文件正文内容的结束， 当读入程序遇到这个值的时候， 它知道所有数据库的所有键值对都已经载入完毕了

check_sum 是一个 8 字节长的无符号整数， 保存着一个校验和， 这个校验和是程序通过对 REDIS 、 db_version 、 databases 、 EOF 四个部分的内容进行计算得出的。 服务器在载入 RDB 文件时， 会将载入数据所计算出的校验和与 check_sum 所记录的校验和进行对比， 以此来检查 RDB 文件是否有出错或者损坏的情况出现。


- databases 部分
  一个 RDB 文件的 databases 部分可以保存任意多个非空数据库。

每个非空数据库在 RDB 文件中都可以保存为 SELECTDB 、 db_number 、 key_value_pairs 三个部分

SELECTDB 常量的长度为 1 字节， 当读入程序遇到这个值的时候， 它知道接下来要读入的将是一个数据库号码。

db_number 保存着一个数据库号码， 根据号码的大小不同， 这个部分的长度可以是 1 字节、 2 字节或者 5 字节。 当程序读入 db_number 部分之后， 服务器会调用 SELECT 命令， 根据读入的数据库号码进行数据库切换， 使得之后读入的键值对可以载入到正确的数据库中。

key_value_pairs 部分保存了数据库中的所有键值对数据， 如果键值对带有过期时间， 那么过期时间也会和键值对保存在一起。 根据键值对的数量、类型、内容、以及是否有过期时间等条件的不同， key_value_pairs 部分的长度也会有所不同。

- key_value_pairs 部分
  RDB 文件中的每个 key_value_pairs 部分都保存了一个或以上数量的键值对， 如果键值对带有过期时间的话， 那么键值对的过期时间也会被保存在内。

不带过期时间的键值对在 RDB 文件中对由 TYPE 、 key 、 value 三部分组成， 如图 IMAGE_KEY_WITHOUT_EXPIRE_TIME 所示

- REDIS_RDB_TYPE_STRING
- REDIS_RDB_TYPE_LIST
- REDIS_RDB_TYPE_SET
- REDIS_RDB_TYPE_ZSET
- REDIS_RDB_TYPE_HASH
- REDIS_RDB_TYPE_LIST_ZIPLIST
- REDIS_RDB_TYPE_SET_INTSET
- REDIS_RDB_TYPE_ZSET_ZIPLIST
- REDIS_RDB_TYPE_HASH_ZIPLIST


以上列出的每个 TYPE 常量都代表了一种对象类型或者底层编码， 当服务器读入 RDB 文件中的键值对数据时， 程序会根据 TYPE 的值来决定如何读入和解释 value 的数据。
key 和 value 分别保存了键值对的键对象和值对象：
- 其中 key 总是一个字符串对象， 它的编码方式和 REDIS_RDB_TYPE_STRING 类型的 value 一样。 根据内容长度的不同， key 的长度也会有所不同。
- 根据 TYPE 类型的不同， 以及保存内容长度的不同， 保存 value 的结构和长度也会有所不同， 本节稍后会详细说明每种 TYPE 类型的value 结构保存方式。

https://www.kancloud.cn/kancloud/redisbook/63879

## AOF 持久化
AOF 持久化功能的实现可以分为命令追加（append）、文件写入、文件同步（sync）三个步骤。

当 AOF 持久化功能处于打开状态时， 服务器在执行完一个写命令之后， 会以协议格式将被执行的写命令追加到服务器状态的 aof_buf 缓冲区的末尾：

```
struct redisServer {

    // ...

    // AOF 缓冲区
    sds aof_buf;

    // ...
};
```

- AOF 文件的写入与同步
  Redis 的服务器进程就是一个事件循环（loop）， 这个循环中的文件事件负责接收客户端的命令请求， 以及向客户端发送命令回复， 而时间事件则负责执行像 serverCron 函数这样需要定时运行的函数。

因为服务器在处理文件事件时可能会执行写命令， 使得一些内容被追加到 aof_buf 缓冲区里面， 所以在服务器每次结束一个事件循环之前， 它都会调用 flushAppendOnlyFile 函数， 考虑是否需要将 aof_buf 缓冲区中的内容写入和保存到 AOF 文件里面， 这个过程可以用以下伪代码表示：

```
def eventLoop():

    while True:

        # 处理文件事件，接收命令请求以及发送命令回复
        # 处理命令请求时可能会有新内容被追加到 aof_buf 缓冲区中
        processFileEvents()

        # 处理时间事件
        processTimeEvents()

        # 考虑是否要将 aof_buf 中的内容写入和保存到 AOF 文件里面
        flushAppendOnlyFile()
```

| appendfsync 选项的值 | flushAppendOnlyFile 函数的行为                               |
| -------------------- | ------------------------------------------------------------ |
| always               | 将 aof_buf 缓冲区中的所有内容写入并同步到 AOF 文件。         |
| everysec             | 将 aof_buf 缓冲区中的所有内容写入到 AOF 文件， 如果上次同步 AOF 文件的时间距离现在超过一秒钟， 那么再次对 AOF 文件进行同步， 并且这个同步操作是由一个线程专门负责执行的。 |
| no                   | 将 aof_buf 缓冲区中的所有内容写入到 AOF 文件， 但并不对 AOF 文件进行同步， 何时同步由操作系统来决定。 |

如果用户没有主动为 appendfsync 选项设置值， 那么 appendfsync 选项的默认值为 everysec ， 关于 appendfsync 选项的更多信息， 请参考 Redis 项目附带的示例配置文件 redis.conf 。

为了提高文件的写入效率， 在现代操作系统中， 当用户调用 write 函数， 将一些数据写入到文件的时候， 操作系统通常会将写入数据暂时保存在一个内存缓冲区里面， 等到缓冲区的空间被填满、或者超过了指定的时限之后， 才真正地将缓冲区中的数据写入到磁盘里面。

这种做法虽然提高了效率， 但也为写入数据带来了安全问题， 因为如果计算机发生停机， 那么保存在内存缓冲区里面的写入数据将会丢失。为此， 系统提供了 fsync 和 fdatasync 两个同步函数， 它们可以强制让操作系统立即将缓冲区中的数据写入到硬盘里面， 从而确保写入数据的安全性。

如果这时 flushAppendOnlyFile 函数被调用， 假设服务器当前 appendfsync 选项的值为 everysec ， 并且根据 server.aof_last_fsync 属性显示， 距离上次同步 AOF 文件已经超过一秒钟， 那么服务器会先将 aof_buf 中的内容写入到 AOF 文件中， 然后再对 AOF 文件进行同步。

当 appendfsync 的值为 always 时， 服务器在每个事件循环都要将 aof_buf 缓冲区中的所有内容写入到 AOF 文件， 并且同步 AOF 文件， 所以 always 的效率是 appendfsync 选项三个值当中最慢的一个， 但从安全性来说， always 也是最安全的， 因为即使出现故障停机， AOF 持久化也只会丢失一个事件循环中所产生的命令数据。

当 appendfsync 的值为 everysec 时， 服务器在每个事件循环都要将 aof_buf 缓冲区中的所有内容写入到 AOF 文件， 并且每隔超过一秒就要在子线程中对 AOF 文件进行一次同步： 从效率上来讲， everysec 模式足够快， 并且就算出现故障停机， 数据库也只丢失一秒钟的命令数据。

## 事件
Redis 基于 Reactor 模式开发了自己的网络事件处理器： 这个处理器被称为文件事件处理器（file event handler）：
- 文件事件处理器使用 I/O 多路复用（multiplexing）程序来同时监听多个套接字， 并根据套接字目前执行的任务来为套接字关联不同的事件处理器。
- 当被监听的套接字准备好执行连接应答（accept）、读取（read）、写入（write）、关闭（close）等操作时， 与操作相对应的文件事件就会产生， 这时文件事件处理器就会调用套接字之前关联好的事件处理器来处理这些事件。

虽然文件事件处理器以单线程方式运行， 但通过使用 I/O 多路复用程序来监听多个套接字， 文件事件处理器既实现了高性能的网络通信模型， 又可以很好地与 Redis 服务器中其他同样以单线程方式运行的模块进行对接， 这保持了 Redis 内部单线程设计的简单性。

- 文件事件处理器的构成
  图 IMAGE_CONSTRUCT_OF_FILE_EVENT_HANDLER 展示了文件事件处理器的四个组成部分， 它们分别是套接字、 I/O 多路复用程序、 文件事件分派器（dispatcher）、 以及事件处理器。

![2015-09-13_55f524b50e6e6](https://user-images.githubusercontent.com/7789698/30958265-2cdfc2a2-a46f-11e7-9bed-b456ce0760c9.png)

文件事件是对套接字操作的抽象， 每当一个套接字准备好执行连接应答（accept）、写入、读取、关闭等操作时， 就会产生一个文件事件。 因为一个服务器通常会连接多个套接字， 所以多个文件事件有可能会并发地出现。

I/O 多路复用程序负责监听多个套接字， 并向文件事件分派器传送那些产生了事件的套接字。

尽管多个文件事件可能会并发地出现， 但 I/O 多路复用程序总是会将所有产生事件的套接字都入队到一个队列里面， 然后通过这个队列， 以有序（sequentially）、同步（synchronously）、每次一个套接字的方式向文件事件分派器传送套接字： 当上一个套接字产生的事件被处理完毕之后（该套接字为事件所关联的事件处理器执行完毕）， I/O 多路复用程序才会继续向文件事件分派器传送下一个套接字， 如图 IMAGE_DISPATCH_EVENT_VIA_QUEUE 。

![2015-09-13_55f524bd8058c](https://user-images.githubusercontent.com/7789698/30958923-315077b2-a471-11e7-867f-15a4ee350dda.png)

文件事件分派器接收 I/O 多路复用程序传来的套接字， 并根据套接字产生的事件的类型， 调用相应的事件处理器。

服务器会为执行不同任务的套接字关联不同的事件处理器， 这些处理器是一个个函数， 它们定义了某个事件发生时， 服务器应该执行的动作。

- I/O 多路复用程序的实现
  Redis 的 I/O 多路复用程序的所有功能都是通过包装常见的 select 、 epoll 、 evport 和 kqueue 这些 I/O 多路复用函数库来实现的， 每个 I/O 多路复用函数库在 Redis 源码中都对应一个单独的文件， 比如 ae_select.c 、 ae_epoll.c 、 ae_kqueue.c ， 诸如此类。

![2015-09-13_55f524bea64ce](https://user-images.githubusercontent.com/7789698/30959044-950343a2-a471-11e7-925c-f94ce82a3995.png)

Redis 在 I/O 多路复用程序的实现源码中用 #include 宏定义了相应的规则， 程序会在编译时自动选择系统中性能最高的 I/O 多路复用函数库来作为 Redis 的 I/O 多路复用程序的底层实现：

https://www.kancloud.cn/kancloud/redisbook/63885

- 一次完整的客户端与服务器连接事件示例
  让我们来追踪一次 Redis 客户端与服务器进行连接并发送命令的整个过程， 看看在过程中会产生什么事件， 而这些事件又是如何被处理的。

假设一个 Redis 服务器正在运作， 那么这个服务器的监听套接字的 AE_READABLE 事件应该正处于监听状态之下， 而该事件所对应的处理器为连接应答处理器。

如果这时有一个 Redis 客户端向服务器发起连接， 那么监听套接字将产生 AE_READABLE 事件， 触发连接应答处理器执行： 处理器会对客户端的连接请求进行应答， 然后创建客户端套接字， 以及客户端状态， 并将客户端套接字的 AE_READABLE 事件与命令请求处理器进行关联， 使得客户端可以向主服务器发送命令请求。

之后， 假设客户端向主服务器发送一个命令请求， 那么客户端套接字将产生 AE_READABLE 事件， 引发命令请求处理器执行， 处理器读取客户端的命令内容， 然后传给相关程序去执行。

执行命令将产生相应的命令回复， 为了将这些命令回复传送回客户端， 服务器会将客户端套接字的 AE_WRITABLE 事件与命令回复处理器进行关联： 当客户端尝试读取命令回复的时候， 客户端套接字将产生 AE_WRITABLE 事件， 触发命令回复处理器执行， 当命令回复处理器将命令回复全部写入到套接字之后， 服务器就会解除客户端套接字的 AE_WRITABLE 事件与命令回复处理器之间的关联。

图 IMAGE_COMMAND_PROGRESS 总结了上面描述的整个通讯过程， 以及通讯时用到的事件处理器。
![2015-09-13_55f524c5218e4](https://user-images.githubusercontent.com/7789698/30960298-33b7c72c-a475-11e7-8a1b-f2ffe510df96.png)

- 客户端属性
  客户端状态包含的属性可以分为两类：

1. 一类是比较通用的属性， 这些属性很少与特定功能相关， 无论客户端执行的是什么工作， 它们都要用到这些属性。
2. 另外一类是和特定功能相关的属性， 比如操作数据库时需要用到的 db 属性和 dictid 属性， 执行事务时需要用到的 mstate 属性， 以及执行 WATCH 命令时需要用到的 watched_keys 属性， 等等。

- 套接字描述符

```
typedef struct redisClient {

    // ...

    int fd;

    // ...

} redisClient;
```

根据客户端类型的不同， fd 属性的值可以是 -1 或者是大于 -1 的整数：
伪客户端（fake client）的 fd 属性的值为 -1 ： 伪客户端处理的命令请求来源于 AOF 文件或者 Lua 脚本， 而不是网络， 所以这种客户端不需要套接字连接， 自然也不需要记录套接字描述符。 目前 Redis 服务器会在两个地方用到伪客户端， 一个用于载入 AOF 文件并还原数据库状态， 而另一个则用于执行 Lua 脚本中包含的 Redis 命令。

普通客户端的 fd 属性的值为大于 -1 的整数： 普通客户端使用套接字来与服务器进行通讯， 所以服务器会用 fd 属性来记录客户端套接字的描述符。 因为合法的套接字描述符不能是 -1 ， 所以普通客户端的套接字描述符的值必然是大于 -1 的整数。

执行 CLIENT_LIST 命令可以列出目前所有连接到服务器的普通客户端， 命令输出中的 fd 域显示了服务器连接客户端所使用的套接字描述符：
```
redis> CLIENT list

addr=127.0.0.1:53428 fd=6 name= age=1242 idle=0 ...
addr=127.0.0.1:53469 fd=7 name= age=4 idle=4 ...
```

使用 CLIENT_SETNAME 命令可以为客户端设置一个名字， 让客户端的身份变得更清晰。

```
typedef struct redisClient {

    // ...

    robj *name;

    // ...

} redisClient;
```

- 标志
  客户端的标志属性 flags 记录了客户端的角色（role）， 以及客户端目前所处的状态：

```
typedef struct redisClient {

    // ...

    int flags;

    // ...

} redisClient;

```

可以是多个标志的二进制或， 比如：

`flags = <flag1> | <flag2> | ...`

```
# 客户端是一个主服务器
REDIS_MASTER

# 客户端正在被列表命令阻塞
REDIS_BLOCKED

# 客户端正在执行事务，但事务的安全性已被破坏
REDIS_MULTI | REDIS_DIRTY_CAS

# 客户端是一个从服务器，并且版本低于 Redis 2.8
REDIS_SLAVE | REDIS_PRE_PSYNC

# 这是专门用于执行 Lua 脚本包含的 Redis 命令的伪客户端
# 它强制服务器将当前执行的命令写入 AOF 文件，并复制给从服务器
REDIS_LUA_CLIENT | REDIS_FORCE_AOF | REDIS_FORCE_REPL
```


- 输入缓冲区

```
typedef struct redisClient {

    // ...

    sds querybuf;

    // ...

} redisClient;
```

输入缓冲区的大小会根据输入内容动态地缩小或者扩大， 但它的最大大小不能超过 1 GB ， 否则服务器将关闭这个客户端。


- 命令与命令参数

```
typedef struct redisClient {

    // ...

    robj **argv;

    int argc;

    // ...

} redisClient;
```

- 身份验证

```
typedef struct redisClient {

    // ...

    int authenticated;

    // ...

} redisClient;
```
如果 authenticated 的值为 0 ， 那么表示客户端未通过身份验证； 如果 authenticated 的值为 1 ， 那么表示客户端已经通过了身份验证。

```
# authenticated 属性的值从 0 变为 1
redis> AUTH 123321
OK

redis> PING
PONG

redis> SET msg "hello world"
OK
```

- 时间

```
typedef struct redisClient {

    // ...

    time_t ctime;

    time_t lastinteraction;

    time_t obuf_soft_limit_reached_time;

    // ...

} redisClient;
```

- 服务端
- 命令请求的执行过程

那么从客户端发送 SET KEY VALUE 命令到获得回复 OK 期间， 客户端和服务器共需要执行以下操作：
1. 客户端向服务器发送命令请求 SET KEY VALUE 。
2. 服务器接收并处理客户端发来的命令请求 SET KEY VALUE ， 在数据库中进行设置操作， 并产生命令回复 OK 。
3. 服务器将命令回复 OK 发送给客户端。
4. 客户端接收服务器返回的命令回复 OK ， 并将这个回复打印给用户观看。

- 发送命令请求
  Redis 服务器的命令请求来自 Redis 客户端， 当用户在客户端中键入一个命令请求时， 客户端会将这个命令请求转换成协议格式， 然后通过连接到服务器的套接字， 将协议格式的命令请求发送给服务器
  ![2015-09-13_55f526cca2250](https://user-images.githubusercontent.com/7789698/30960377-64b676c0-a475-11e7-9dee-d0edcbcd3a30.png)


- 读取命令请求

当客户端与服务器之间的连接套接字因为客户端的写入而变得可读时， 服务器将调用命令请求处理器来执行以下操作：
1. 读取套接字中协议格式的命令请求， 并将其保存到客户端状态的输入缓冲区里面。
2. 对输入缓冲区中的命令请求进行分析， 提取出命令请求中包含的命令参数， 以及命令参数的个数， 然后分别将参数和参数个数保存到客户端状态的 argv 属性和 argc 属性里面。
3. 调用命令执行器， 执行客户端指定的命令。

https://www.kancloud.cn/kancloud/redisbook/63892

# 多机数据库的实现
## 旧版复制功能的实现
Redis 的复制功能分为同步（sync）和命令传播（command propagate）两个操作：
- 同步操作用于将从服务器的数据库状态更新至主服务器当前所处的数据库状态。
- 而命令传播操作则用于在主服务器的数据库状态被修改， 导致主从服务器的数据库状态出现不一致时， 让主从服务器的数据库重新回到一致状态。

- 同步
  当客户端向从服务器发送 SLAVEOF 命令， 要求从服务器复制主服务器时， 从服务器首先需要执行同步操作， 也即是， 将从服务器的数据库状态更新至主服务器当前所处的数据库状态。

从服务器对主服务器的同步操作需要通过向主服务器发送 SYNC 命令来完成， 以下是 SYNC 命令的执行步骤：
1. 从服务器向主服务器发送 SYNC 命令。
2. 收到 SYNC 命令的主服务器执行 BGSAVE 命令， 在后台生成一个 RDB 文件， 并使用一个缓冲区记录从现在开始执行的所有写命令。
3. 当主服务器的 BGSAVE 命令执行完毕时， 主服务器会将 BGSAVE 命令生成的 RDB 文件发送给从服务器， 从服务器接收并载入这个 RDB 文件， 将自己的数据库状态更新至主服务器执行 BGSAVE 命令时的数据库状态。
4. 主服务器将记录在缓冲区里面的所有写命令发送给从服务器， 从服务器执行这些写命令， 将自己的数据库状态更新至主服务器数据库当前所处的状态。

![2015-09-13_55f5274336096](https://user-images.githubusercontent.com/7789698/30960614-1ed7a902-a476-11e7-817e-3fe42e095c2b.png)



| 时间 | 主服务器                                                     | 从服务器                                                     |
| ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| T0   | 服务器启动。                                                 | 服务器启动。                                                 |
| T1   | 执行 SET k1 v1 。                                            |                                                              |
| T2   | 执行 SET k2 v2 。                                            |                                                              |
| T3   | 执行 SET k3 v3 。                                            |                                                              |
| T4   |                                                              | 向主服务器发送 SYNC 命令。                                   |
| T5   | 接收到从服务器发来的 SYNC 命令， 执行 BGSAVE 命令， 创建包含键 k1 、 k2 、 k3 的 RDB 文件， 并使用缓冲区记录接下来执行的所有写命令。 |                                                              |
| T6   | 执行 SET k4 v4 ， 并将这个命令记录到缓冲区里面。             |                                                              |
| T7   | 执行 SET k5 v5 ， 并将这个命令记录到缓冲区里面。             |                                                              |
| T8   | BGSAVE 命令执行完毕， 向从服务器发送 RDB 文件。              |                                                              |
| T9   |                                                              | 接收并载入主服务器发来的 RDB 文件 ， 获得 k1 、 k2 、 k3 三个键。 |
| T10  | 向从服务器发送缓冲区中保存的写命令 SET k4 v4 和 SET k5v5 。  |                                                              |
| T11  |                                                              | 接收并执行主服务器发来的两个 SET 命令， 得到 k4 和 k5 两个键。 |
| T12  | 同步完成， 现在主从服务器两者的数据库都包含了键 k1 、k2 、 k3 、 k4 和 k5 。 | 同步完成， 现在主从服务器两者的数据库都包含了键 k1 、 k2 、k3 、 k4 和 k5 。 |

- 命令传播
  在同步操作执行完毕之后， 主从服务器两者的数据库将达到一致状态， 但这种一致并不是一成不变的 —— 每当主服务器执行客户端发送的写命令时， 主服务器的数据库就有可能会被修改， 并导致主从服务器状态不再一致。

举个例子， 假设一个主服务器和一个从服务器刚刚完成同步操作， 它们的数据库都保存了相同的五个键 k1 至 k5 ， 如图 IMAGE_CONSISTENT 所示。

![2015-09-13_55f52746e0111](https://user-images.githubusercontent.com/7789698/30960653-3dd78516-a476-11e7-80a9-b96951fb733c.png)

如果这时， 客户端向主服务器发送命令 DEL k3 ， 那么主服务器在执行完这个 DEL 命令之后， 主从服务器的数据库将出现不一致： 主服务器的数据库已经不再包含键 k3 ， 但这个键却仍然包含在从服务器的数据库里面， 如图 IMAGE_INCONSISTENT 所示。

![2015-09-13_55f527488c8ad](https://user-images.githubusercontent.com/7789698/30960690-59c4313e-a476-11e7-8652-1eabd5e35054.png)

为了让主从服务器再次回到一致状态， 主服务器需要对从服务器执行命令传播操作： 主服务器会将自己执行的写命令 —— 也即是造成主从服务器不一致的那条写命令 —— 发送给从服务器执行， 当从服务器执行了相同的写命令之后， 主从服务器将再次回到一致状态。

在上面的例子中， 主服务器因为执行了命令 DEL k3 而导致主从服务器不一致， 所以主服务器将向从服务器发送相同的命令 DEL k3 ： 当从服务器执行完这个命令之后， 主从服务器将再次回到一致状态 —— 现在主从服务器两者的数据库都不再包含键 k3 了， 如图 IMAGE_PROPAGATE_DEL_k3 所示。

![2015-09-13_55f5274a0ca0f](https://user-images.githubusercontent.com/7789698/30960728-7680800c-a476-11e7-9321-7c17ff15f041.png)

## Sentinel
- 启动并初始化 Sentinel
  启动一个 Sentinel 可以使用命令：
  `$ redis-sentinel /path/to/your/sentinel.conf`
  或者命令：
  `$ redis-server /path/to/your/sentinel.conf --sentinel`
  这两个命令的效果完全相同。

当一个 Sentinel 启动时， 它需要执行以下步骤：
1. 初始化服务器。
2. 将普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。
3. 初始化 Sentinel 状态。
4. 根据给定的配置文件， 初始化 Sentinel 的监视主服务器列表。
5. 创建连向主服务器的网络连接。

- 初始化服务器
  因为 Sentinel 本质上只是一个运行在特殊模式下的 Redis 服务器， 所以启动 Sentinel 的第一步， 就是初始化一个普通的 Redis 服务器


| 功能                                                    | 使用情况                                                     |
| ------------------------------------------------------- | ------------------------------------------------------------ |
| 数据库和键值对方面的命令， 比如 SET 、 DEL 、FLUSHDB 。 | 不使用。                                                     |
| 事务命令， 比如 MULTI 和 WATCH 。                       | 不使用。                                                     |
| 脚本命令，比如 EVAL 。                                  | 不使用。                                                     |
| RDB 持久化命令， 比如 SAVE 和 BGSAVE 。                 | 不使用。                                                     |
| AOF 持久化命令， 比如 BGREWRITEAOF 。                   | 不使用。                                                     |
| 复制命令，比如 SLAVEOF 。                               | Sentinel 内部可以使用，但客户端不可以使用。                  |
| 发布与订阅命令， 比如 PUBLISH 和 SUBSCRIBE 。           | SUBSCRIBE 、 PSUBSCRIBE 、 UNSUBSCRIBE PUNSUBSCRIBE 四个命令在 Sentinel 内部和客户端都可以使用， 但 PUBLISH 命令只能在 Sentinel 内部使用。 |
| 文件事件处理器（负责发送命令请求、处理命令回复）。      | Sentinel 内部使用， 但关联的文件事件处理器和普通 Redis 服务器不同。 |
| 时间事件处理器（负责执行 serverCron 函数）。            | Sentinel 内部使用， 时间事件的处理器仍然是 serverCron 函数， serverCron函数会调用 sentinel.c/sentinelTimer 函数， 后者包含了 Sentinel 要执行的所有操作。 |


- 使用 Sentinel 专用代码
  启动 Sentinel 的第二个步骤就是将一部分普通 Redis 服务器使用的代码替换成 Sentinel 专用代码。

比如说， 普通 Redis 服务器使用 redis.h/REDIS_SERVERPORT 常量的值作为服务器端口：
`#define REDIS_SERVERPORT 6379`
而 Sentinel 则使用 sentinel.c/REDIS_SENTINEL_PORT 常量的值作为服务器端口：
`#define REDIS_SENTINEL_PORT 26379`

除此之外， 普通 Redis 服务器使用 redis.c/redisCommandTable 作为服务器的命令表：

```
struct redisCommand redisCommandTable[] = {
    {"get",getCommand,2,"r",0,NULL,1,1,1,0,0},
    {"set",setCommand,-3,"wm",0,noPreloadGetKeys,1,1,1,0,0},
    {"setnx",setnxCommand,3,"wm",0,noPreloadGetKeys,1,1,1,0,0},
    // ...
    {"script",scriptCommand,-2,"ras",0,NULL,0,0,0,0,0},
    {"time",timeCommand,1,"rR",0,NULL,0,0,0,0,0},
    {"bitop",bitopCommand,-4,"wm",0,NULL,2,-1,1,0,0},
    {"bitcount",bitcountCommand,-2,"r",0,NULL,1,1,1,0,0}
}
```

- 初始化 Sentinel 状态
  在应用了 Sentinel 的专用代码之后， 接下来， 服务器会初始化一个 sentinel.c/sentinelState 结构（后面简称“Sentinel 状态”）， 这个结构保存了服务器中所有和 Sentinel 功能有关的状态 （服务器的一般状态仍然由 redis.h/redisServer 结构保存）：

```
struct sentinelState {

    // 当前纪元，用于实现故障转移
    uint64_t current_epoch;

    // 保存了所有被这个 sentinel 监视的主服务器
    // 字典的键是主服务器的名字
    // 字典的值则是一个指向 sentinelRedisInstance 结构的指针
    dict *masters;

    // 是否进入了 TILT 模式？
    int tilt;

    // 目前正在执行的脚本的数量
    int running_scripts;

    // 进入 TILT 模式的时间
    mstime_t tilt_start_time;

    // 最后一次执行时间处理器的时间
    mstime_t previous_time;

    // 一个 FIFO 队列，包含了所有需要执行的用户脚本
    list *scripts_queue;

} sentinel;
```

- 初始化 Sentinel 状态的 masters 属性
  Sentinel 状态中的 masters 字典记录了所有被 Sentinel 监视的主服务器的相关信息， 其中：
- 字典的键是被监视主服务器的名字。
- 而字典的值则是被监视主服务器对应的 sentinel.c/sentinelRedisInstance 结构。
  每个 sentinelRedisInstance 结构（后面简称“实例结构”）代表一个被 Sentinel 监视的 Redis 服务器实例（instance）， 这个实例可以是主服务器、从服务器、或者另外一个 Sentinel 。
  实例结构包含的属性非常多， 以下代码展示了实例结构在表示主服务器时使用的其中一部分属性， 本章接下来将逐步对实例结构中的各个属性进行介绍：

```
typedef struct sentinelRedisInstance {

    // 标识值，记录了实例的类型，以及该实例的当前状态
    int flags;

    // 实例的名字
    // 主服务器的名字由用户在配置文件中设置
    // 从服务器以及 Sentinel 的名字由 Sentinel 自动设置
    // 格式为 ip:port ，例如 "127.0.0.1:26379"
    char *name;

    // 实例的运行 ID
    char *runid;

    // 配置纪元，用于实现故障转移
    uint64_t config_epoch;

    // 实例的地址
    sentinelAddr *addr;

    // SENTINEL down-after-milliseconds 选项设定的值
    // 实例无响应多少毫秒之后才会被判断为主观下线（subjectively down）
    mstime_t down_after_period;

    // SENTINEL monitor <master-name> <IP> <port> <quorum> 选项中的 quorum 参数
    // 判断这个实例为客观下线（objectively down）所需的支持投票数量
    int quorum;

    // SENTINEL parallel-syncs <master-name> <number> 选项的值
    // 在执行故障转移操作时，可以同时对新的主服务器进行同步的从服务器数量
    int parallel_syncs;

    // SENTINEL failover-timeout <master-name> <ms> 选项的值
    // 刷新故障迁移状态的最大时限
    mstime_t failover_timeout;

    // ...

} sentinelRedisInstance;
```

sentinelRedisInstance.addr 属性是一个指向 sentinel.c/sentinelAddr 结构的指针， 这个结构保存着实例的 IP 地址和端口号：

```
typedef struct sentinelAddr {

    char *ip;

    int port;

} sentinelAddr;
```

对 Sentinel 状态的初始化将引发对 masters 字典的初始化， 而 masters 字典的初始化是根据被载入的 Sentinel 配置文件来进行的。

- 创建连向主服务器的网络连接
  初始化 Sentinel 的最后一步是创建连向被监视主服务器的网络连接： Sentinel 将成为主服务器的客户端， 它可以向主服务器发送命令， 并从命令回复中获取相关的信息。

对于每个被 Sentinel 监视的主服务器来说， Sentinel 会创建两个连向主服务器的异步网络连接：

一个是命令连接， 这个连接专门用于向主服务器发送命令， 并接收命令回复。
另一个是订阅连接， 这个连接专门用于订阅主服务器的 __sentinel__:hello 频道。
为什么有两个连接？

在 Redis 目前的发布与订阅功能中， 被发送的信息都不会保存在 Redis 服务器里面， 如果在信息发送时， 想要接收信息的客户端不在线或者断线， 那么这个客户端就会丢失这条信息。

因此， 为了不丢失 __sentinel__:hello 频道的任何信息， Sentinel 必须专门用一个订阅连接来接收该频道的信息。
而另一方面， 除了订阅频道之外， Sentinel 还又必须向主服务器发送命令， 以此来与主服务器进行通讯， 所以 Sentinel 还必须向主服务器创建命令连接。

并且因为 Sentinel 需要与多个实例创建多个网络连接， 所以 Sentinel 使用的是异步连接。

图 IMAGE_SENTINEL_CONNECT_SERVER 展示了一个 Sentinel 向被它监视的两个主服务器 master1 和 master2 创建命令连接和订阅连接的例子。

![2015-09-13_55f5282219b43](https://user-images.githubusercontent.com/7789698/30962201-75404d30-a47b-11e7-8f0c-29ba45be8932.png)


- Sentinel 只是一个运行在特殊模式下的 Redis 服务器， 它使用了和普通模式不同的命令表， 所以 Sentinel 模式能够使用的命令和普通 Redis 服务器能够使用的命令不同。
- Sentinel 会读入用户指定的配置文件， 为每个要被监视的主服务器创建相应的实例结构， 并创建连向主服务器的命令连接和订阅连接， 其中命令连接用于向主服务器发送命令请求， 而订阅连接则用于接收指定频道的消息。
- Sentinel 通过向主服务器发送 INFO 命令来获得主服务器属下所有从服务器的地址信息， 并为这些从服务器创建相应的实例结构， 以及连向这些从服务器的命令连接和订阅连接。
- 在一般情况下， Sentinel 以每十秒一次的频率向被监视的主服务器和从服务器发送 INFO 命令， 当主服务器处于下线状态， 或者 Sentinel 正在对主服务器进行故障转移操作时， Sentinel 向从服务器发送 INFO 命令的频率会改为每秒一次。
- 对于监视同一个主服务器和从服务器的多个 Sentinel 来说， 它们会以每两秒一次的频率， 通过向被监视服务器的 __sentinel__:hello频道发送消息来向其他 Sentinel 宣告自己的存在。
- 每个 Sentinel 也会从 __sentinel__:hello 频道中接收其他 Sentinel 发来的信息， 并根据这些信息为其他 Sentinel 创建相应的实例结构， 以及命令连接。
- Sentinel 只会与主服务器和从服务器创建命令连接和订阅连接， Sentinel 与 Sentinel 之间则只创建命令连接。
- Sentinel 以每秒一次的频率向实例（包括主服务器、从服务器、其他 Sentinel）发送 PING 命令， 并根据实例对 PING 命令的回复来判断实例是否在线： 当一个实例在指定的时长中连续向 Sentinel 发送无效回复时， Sentinel 会将这个实例判断为主观下线。
- 当 Sentinel 将一个主服务器判断为主观下线时， 它会向同样监视这个主服务器的其他 Sentinel 进行询问， 看它们是否同意这个主服务器已经进入主观下线状态。
- 当 Sentinel 收集到足够多的主观下线投票之后， 它会将主服务器判断为客观下线， 并发起一次针对主服务器的故障转移操作。

Sentinel 系统选举领头 Sentinel 的方法是对 Raft 算法的领头选举方法的实现

- 集群

一个 Redis 集群通常由多个节点（node）组成， 在刚开始的时候， 每个节点都是相互独立的， 它们都处于一个只包含自己的集群当中， 要组建一个真正可工作的集群， 我们必须将各个独立的节点连接起来， 构成一个包含多个节点的集群。

连接各个节点的工作可以使用 CLUSTER MEET 命令来完成， 该命令的格式如下：

`CLUSTER MEET <ip> <port>`

向一个节点 node 发送 CLUSTER MEET 命令， 可以让 node 节点与 ip 和 port 所指定的节点进行握手（handshake）， 当握手成功时， node节点就会将 ip 和 port 所指定的节点添加到 node 节点当前所在的集群中。
举个例子， 假设现在有三个独立的节点 127.0.0.1:7000 、 127.0.0.1:7001 、 127.0.0.1:7002 （下文省略 IP 地址，直接使用端口号来区分各个节点）， 我们首先使用客户端连上节点 7000 ， 通过发送 CLUSTER NODE 命令可以看到， 集群目前只包含 7000 自己一个节点：

```
$ redis-cli -c -p 7000
127.0.0.1:7000> CLUSTER NODES
51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected
```

通过向节点 7000 发送以下命令， 我们可以将节点 7001 添加到节点 7000 所在的集群里面：

```
127.0.0.1:7000> CLUSTER MEET 127.0.0.1 7001
OK

127.0.0.1:7000> CLUSTER NODES
68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master - 0 1388204746210 0 connected
51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected
```

继续向节点 7000 发送以下命令， 我们可以将节点 7002 也添加到节点 7000 和节点 7001 所在的集群里面：

```
127.0.0.1:7000> CLUSTER MEET 127.0.0.1 7002
OK

127.0.0.1:7000> CLUSTER NODES
68eef66df23420a5862208ef5b1a7005b806f2ff 127.0.0.1:7001 master - 0 1388204848376 0 connected
9dfb4c4e016e627d9769e4c9bb0d4fa208e65c26 127.0.0.1:7002 master - 0 1388204847977 0 connected
51549e625cfda318ad27423a31e7476fe3cd2939 :0 myself,master - 0 0 0 connected
```

现在， 这个集群里面包含了 7000 、 7001 和 7002 三个节点， 图 IMAGE_CONNECT_NODES_1 至 IMAGE_CONNECT_NODES_5 展示了这三个节点进行握手的整个过程。

![2015-09-13_55f52896e231a](https://user-images.githubusercontent.com/7789698/30962347-f7021b0a-a47b-11e7-9f53-7db8a3818ab6.png)

![2015-09-13_55f52898c2d57](https://user-images.githubusercontent.com/7789698/30962355-fe5ce376-a47b-11e7-8373-a0750e4aac54.png)

![2015-09-13_55f52899dbdb6](https://user-images.githubusercontent.com/7789698/30962363-045127f6-a47c-11e7-805e-2a7568d9e580.png)

![2015-09-13_55f5289b0e82b](https://user-images.githubusercontent.com/7789698/30962380-15c00106-a47c-11e7-944f-162fc1594086.png)

- 启动节点
  一个节点就是一个运行在集群模式下的 Redis 服务器， Redis 服务器在启动时会根据 cluster-enabled 配置选项的是否为 yes 来决定是否开启服务器的集群模式

节点（运行在集群模式下的 Redis 服务器）会继续使用所有在单机模式中使用的服务器组件， 比如说：
- 节点会继续使用文件事件处理器来处理命令请求和返回命令回复。
- 节点会继续使用时间事件处理器来执行 serverCron 函数， 而 serverCron 函数又会调用集群模式特有的 clusterCron 函数： clusterCron函数负责执行在集群模式下需要执行的常规操作， 比如向集群中的其他节点发送 Gossip 消息， 检查节点是否断线； 又或者检查是否需要对下线节点进行自动故障转移， 等等。
- 节点会继续使用数据库来保存键值对数据，键值对依然会是各种不同类型的对象。
- 节点会继续使用 RDB 持久化模块和 AOF 持久化模块来执行持久化工作。
- 节点会继续使用发布与订阅模块来执行 PUBLISH 、 SUBSCRIBE 等命令。
- 节点会继续使用复制模块来进行节点的复制工作。
- 节点会继续使用 Lua 脚本环境来执行客户端输入的 Lua 脚本。

- 集群数据结构
  clusterNode 结构保存了一个节点的当前状态， 比如节点的创建时间， 节点的名字， 节点当前的配置纪元， 节点的 IP 和地址， 等等。

每个节点都会使用一个 clusterNode 结构来记录自己的状态， 并为集群中的所有其他节点（包括主节点和从节点）都创建一个相应的clusterNode 结构， 以此来记录其他节点的状态：

```
struct clusterNode {

    // 创建节点的时间
    mstime_t ctime;

    // 节点的名字，由 40 个十六进制字符组成
    // 例如 68eef66df23420a5862208ef5b1a7005b806f2ff
    char name[REDIS_CLUSTER_NAMELEN];

    // 节点标识
    // 使用各种不同的标识值记录节点的角色（比如主节点或者从节点），
    // 以及节点目前所处的状态（比如在线或者下线）。
    int flags;

    // 节点当前的配置纪元，用于实现故障转移
    uint64_t configEpoch;

    // 节点的 IP 地址
    char ip[REDIS_IP_STR_LEN];

    // 节点的端口号
    int port;

    // 保存连接节点所需的有关信息
    clusterLink *link;

    // ...

};
```

clusterNode 结构的 link 属性是一个 clusterLink 结构， 该结构保存了连接节点所需的有关信息， 比如套接字描述符， 输入缓冲区和输出缓冲区：

```
typedef struct clusterLink {

    // 连接的创建时间
    mstime_t ctime;

    // TCP 套接字描述符
    int fd;

    // 输出缓冲区，保存着等待发送给其他节点的消息（message）。
    sds sndbuf;

    // 输入缓冲区，保存着从其他节点接收到的消息。
    sds rcvbuf;

    // 与这个连接相关联的节点，如果没有的话就为 NULL
    struct clusterNode *node;

} clusterLink;
```

redisClient 结构和 clusterLink 结构的相同和不同之处

redisClient 结构和 clusterLink 结构都有自己的套接字描述符和输入、输出缓冲区， 这两个结构的区别在于， redisClient 结构中的套接字和缓冲区是用于连接客户端的， 而 clusterLink 结构中的套接字和缓冲区则是用于连接节点的。

最后， 每个节点都保存着一个 clusterState 结构， 这个结构记录了在当前节点的视角下， 集群目前所处的状态 —— 比如集群是在线还是下线， 集群包含多少个节点， 集群当前的配置纪元， 诸如此类：

```
typedef struct clusterState {

    // 指向当前节点的指针
    clusterNode *myself;

    // 集群当前的配置纪元，用于实现故障转移
    uint64_t currentEpoch;

    // 集群当前的状态：是在线还是下线
    int state;

    // 集群中至少处理着一个槽的节点的数量
    int size;

    // 集群节点名单（包括 myself 节点）
    // 字典的键为节点的名字，字典的值为节点对应的 clusterNode 结构
    dict *nodes;

    // ...

} clusterState;
```

以前面介绍的 7000 、 7001 、 7002 三个节点为例， 图 IMAGE_CLUSTER_STATE_OF_7000 展示了节点 7000 创建的 clusterState 结构， 这个结构从节点 7000 的角度记录了集群、以及集群包含的三个节点的当前状态 （为了空间考虑，图中省略了 clusterNode 结构的一部分属性）：
- 结构的 currentEpoch 属性的值为 0 ， 表示集群当前的配置纪元为 0 。
- 结构的 size 属性的值为 0 ， 表示集群目前没有任何节点在处理槽： 因此结构的 state 属性的值为 REDIS_CLUSTER_FAIL —— 这表示集群目前处于下线状态。
- 结构的 nodes 字典记录了集群目前包含的三个节点， 这三个节点分别由三个 clusterNode 结构表示： 其中 myself 指针指向代表节点 7000 的 clusterNode 结构， 而字典中的另外两个指针则分别指向代表节点 7001 和代表节点 7002 的 clusterNode 结构， 这两个节点是节点 7000 已知的在集群中的其他节点。
- 三个节点的 clusterNode 结构的 flags 属性都是 REDIS_NODE_MASTER ，说明三个节点都是主节点。

节点 7001 和节点 7002 也会创建类似的 clusterState 结构：
- 不过在节点 7001 创建的 clusterState 结构中， myself 指针将指向代表节点 7001 的 clusterNode 结构， 而节点 7000 和节点 7002 则是集群中的其他节点。
- 而在节点 7002 创建的 clusterState 结构中， myself 指针将指向代表节点 7002 的 clusterNode 结构， 而节点 7000 和节点 7001 则是集群中的其他节点。

- CLUSTER MEET 命令的实现
  通过向节点 A 发送 CLUSTER MEET 命令， 客户端可以让接收命令的节点 A 将另一个节点 B 添加到节点 A 当前所在的集群里面：

`CLUSTER MEET <ip> <port>`

收到命令的节点 A 将与节点 B 进行握手（handshake）， 以此来确认彼此的存在， 并为将来的进一步通信打好基础：
1. 节点 A 会为节点 B 创建一个 clusterNode 结构， 并将该结构添加到自己的 clusterState.nodes 字典里面。
2. 之后， 节点 A 将根据 CLUSTER MEET 命令给定的 IP 地址和端口号， 向节点 B 发送一条 MEET 消息（message）。
3. 如果一切顺利， 节点 B 将接收到节点 A 发送的 MEET 消息， 节点 B 会为节点 A 创建一个 clusterNode 结构， 并将该结构添加到自己的 clusterState.nodes 字典里面。
4. 之后， 节点 B 将向节点 A 返回一条 PONG 消息。
5. 如果一切顺利， 节点 A 将接收到节点 B 返回的 PONG 消息， 通过这条 PONG 消息节点 A 可以知道节点 B 已经成功地接收到了自己发送的 MEET 消息。
6. 之后， 节点 A 将向节点 B 返回一条 PING 消息。
7. 如果一切顺利， 节点 B 将接收到节点 A 返回的 PING 消息， 通过这条 PING 消息节点 B 可以知道节点 A 已经成功地接收到了自己返回的 PONG 消息， 握手完成。

![2015-09-13_55f528a75c69b](https://user-images.githubusercontent.com/7789698/30962518-aaeae3e0-a47c-11e7-9e7e-49d70d108122.png)

之后， 节点 A 会将节点 B 的信息通过 Gossip 协议传播给集群中的其他节点， 让其他节点也与节点 B 进行握手， 最终， 经过一段时间之后， 节点 B 会被集群中的所有节点认识。


# 独立功能的实现
https://www.kancloud.cn/kancloud/redisbook/63905