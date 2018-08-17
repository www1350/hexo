---
title: Paxos 协议
tags:
  - 笔记
  - Paxos
  - 分布式
  - Raft
  - zookeeper
  - ZAB
  - CAP
  - 2PC
  - 3PC
categories: 一致性
abbrlink: 410c3782
date: 2018-04-12 10:13:02
---

## CAP

### 数据一致性(consistency)

如果系统对一个写操作返回成功，那么之后的读请求都必须读到这个新数据；如果返回失败，那么所有读操作都不能读到这个数据，对调用者而言数据具有强一致性(strong consistency) (又叫原子性 atomic、线性一致性 linearizable consistency)

### 服务可用性(availability)

所有读写请求在一定时间内得到响应，可终止、不会一直等待

### 分区容错性(partition-tolerance)

在网络分区的情况下，被分隔的节点仍能正常对外服务



根据定理，分布式系统只能满足三项中的两项而不可能满足全部三项[[4\]](https://zh.wikipedia.org/wiki/CAP%E5%AE%9A%E7%90%86#cite_note-4)。理解CAP理论的最简单方式是想象两个节点分处分区两侧。允许至少一个节点更新状态会导致数据不一致，即丧失了C性质。如果为了保证数据一致性，将分区一侧的节点设置为不可用，那么又丧失了A性质。除非两个节点可以互相通信，才能既保证C又保证A，这又会导致丧失P性质。

<!-- more -->

## BASE理论

### 基本可用（Basically Available）

基本可用是指分布式系统在出现故障的时候，允许损失部分可用性，即保证核心可用。

- 响应时间上的损失：正常情况下，一个在线搜索引擎需要0.5秒内返回给用户相应的查询结果，但由于出现异常（比如系统部分机房发生断电或断网故障），查询结果的响应时间增加到了1~2秒。
- 功能上的损失：正常情况下，在一个电子商务网站上进行购物，消费者几乎能够顺利地完成每一笔订单，但是在一些节日大促购物高峰的时候，由于消费者的购物行为激增，为了保护购物系统的稳定性，部分消费者可能会被引导到一个降级页面。

### 软状态（Soft state）

软状态是指允许系统存在中间状态，而该中间状态不会影响系统整体可用性。分布式存储中一般一份数据至少会有三个副本，允许不同节点间副本同步的延时就是软状态的体现。

### 最终一致性（Eventually consistent）

最终一致性是指系统中的所有数据副本经过一定时间后，最终能够达到一致的状态。弱一致性和强一致性相反，最终一致性是弱一致性的一种特殊情况。

## 集群脑裂

集群的脑裂通常是发生在集群中部分节点之间不可达而引起的（或者因为节点请求压力较大，导致其他节点与该节点的心跳检测不可用）。当上述情况发生时，不同分裂的小集群会自主的选择出master节点，造成原本的集群会同时存在多个master节点。

## 两阶段提交（Two-Phase Commit）

为了使基于[分布式系统](https://zh.wikipedia.org/wiki/%E5%88%86%E5%B8%83%E5%BC%8F%E7%B3%BB%E7%BB%9F)架构下的所有节点在进行[事务](https://zh.wikipedia.org/wiki/%E6%95%B0%E6%8D%AE%E5%BA%93%E4%BA%8B%E5%8A%A1)提交时保持一致性而设计的一种[算法](https://zh.wikipedia.org/wiki/%E7%AE%97%E6%B3%95)(Algorithm)。

通常，**二阶段提交**也被称为是一种**协议**(Protocol)。在分布式系统中，每个节点虽然可以知晓自己的操作时成功或者失败，却无法知道其他节点的操作的成功或失败。当一个事务跨越多个节点时，为了保持事务的[ACID](https://zh.wikipedia.org/wiki/ACID)特性，需要引入一个作为**协调者**的组件来统一掌控所有节点(称作**参与者**)的操作结果并最终指示这些节点是否要把操作结果进行真正的提交(比如将更新后的数据写入磁盘等等)。

因此，二阶段提交的算法思路可以概括为： 参与者将操作成败通知协调者，再由协调者根据所有参与者的反馈情报决定各参与者是否要提交操作还是中止操作。

### 基本算法

- 第一阶段(提交请求阶段)

  1. **协调者**节点向**所有参与者**节点询问是否可以执行提交操作，并开始等待各参与者节点的响应。
  2. 参与者节点**执行**询问发起为止的**所有事务**操作，并将[Undo信息](https://zh.wikipedia.org/w/index.php?title=Undo%E4%BF%A1%E6%81%AF&action=edit&redlink=1)和[Redo信息](https://zh.wikipedia.org/w/index.php?title=Redo%E4%BF%A1%E6%81%AF&action=edit&redlink=1)写入日志。
  - 各参与者节点**响应**协调者节点发起的询问。如果参与者节点的事务操作实际执行成功，则它返回一个"同意"消息；如果参与者节点的事务操作实际执行失败，则它返回一个"中止"消息。

  有时候，第一阶段也被称作**投票阶段**，即各参与者投票是否要继续接下来的提交操作。


- 第二阶段(提交执行阶段)

  - 成功

    当协调者节点从所有参与者节点获得的相应消息都为"同意"时：

    1. 协调者节点向所有参与者节点发出"**正式提交**"的请求。
    2. 参与者节点正式完成操作（如commit），并释放在整个事务期间内占用的资源。
    - 参与者节点向协调者节点发送"完成"消息。

    - 协调者节点收到**所有**参与者节点**反馈**的"完成"消息后，完成事务。

   - 失败

     如果任一参与者节点在第一阶段返回的响应消息为"终止"，或者 协调者节点在第一阶段的询问超时之前无法获取所有参与者节点的响应消息时：

     1. 协调者节点向所有参与者节点发出"回滚操作"的请求。
     2. 参与者节点利用之前写入的Undo信息执行回滚，并释放在整个事务期间内占用的资源。
     - 参与者节点向协调者节点发送"回滚完成"消息。

     - 协调者节点收到所有参与者节点反馈的"回滚完成"消息后，取消事务。

   有时候，第二阶段也被称作**完成阶段**，因为无论结果怎样，协调者都必须在此阶段结束当前事务。

### 缺点

二阶段提交算法的最大缺点就在于它的执行过程中间，节点都处于**阻塞**状态。即节点之间在等待对方的相应消息时，它将什么也做不了。特别是，当一个节点在已经占有了某项资源的情况下，为了等待其他节点的响应消息而陷入阻塞状态时，当第三个节点尝试访问该节点占有的资源时，这个节点也将连带陷入阻塞状态。

另外，协调者节点指示参与者节点进行提交等操作时，如有**参与者节点出现了崩溃**等情况而导致协调者始终无法获取所有参与者的响应信息，这时协调者将只能依赖协调者自身的**超时机制**来生效。但往往超时机制生效时，协调者都会指示参与者进行回滚操作。这样的策略显得比较保守。

## 三阶段提交（Three-Phase Commit）

在计算机网络及数据库的范畴下，使得一个分布式系统内的所有节点能够执行事务的提交的一种分布式算法。三阶段提交是为解决[两阶段提交协议](https://zh.wikipedia.org/wiki/%E4%BA%8C%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4)的缺点而设计的。

与两阶段提交不同的是，三阶段提交是“非阻塞”协议。三阶段提交在两阶段提交的**第一阶段与第二阶段之间插入了**一个**准备阶段**，使得原先在两阶段提交中，参与者在投票之后，由于协调者发生崩溃或错误，而导致参与者处于无法知晓是否提交或者中止的“不确定状态”所产生的可能相当长的延时的问题[[1\]](https://zh.wikipedia.org/wiki/%E4%B8%89%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4#cite_note-1)得以解决。 举例来说，假设有一个决策小组由一个主持人负责与多位组员以电话联络方式协调是否通过一个提案，以两阶段提交来说，主持人收到一个提案请求，打电话跟每个组员询问是否通过并统计回复，然后将最后决定打电话通知各组员。要是主持人在跟第一位组员通完电话后失忆，而第一位组员在得知结果并执行后老人痴呆，那么即使重新选出主持人，也没人知道最后的提案决定是什么，也许是通过，也许是驳回，不管大家选择哪一种决定，都有可能与第一位组员已执行过的真实决定不一致，老板就会不开心认为决策小组沟通有问题而解雇。三阶段提交即是引入了另一个步骤，主持人打电话跟组员通知请准备通过提案，以避免没人知道真实决定而造成决定不一致的失业危机。为什么能够解决二阶段提交的问题呢？回到刚刚提到的状况，在主持人通知完第一位组员请准备通过后两人意外失忆，即使没人知道全体在第一阶段的决定为何，全体决策组员仍可以重新协调过程或直接否决，不会有不一致决定而失业。那么当主持人通知完全体组员请准备通过并得到大家的再次确定后进入第三阶段，当主持人通知第一位组员请通过提案后两人意外失忆，这时候其他组员再重新选出主持人后，仍可以知道目前至少是处于准备通过提案阶段，表示第一阶段大家都已经决定要通过了，此时便可以直接通过。

![image](https://user-images.githubusercontent.com/7789698/38654220-cd1a8610-3e40-11e8-8027-ea351e5bab41.png)

### 基本算法

- 阶段一CanCommit

  1. 事务询问

  ```
  协调者向各参与者发送CanCommit的请求，询问是否可以执行事务提交操作，并开始等待各参与者的响应
  ```

  2. 参与者向协调者反馈询问的响应

  参与者收到CanCommit请求后，正常情况下，如果自身认为可以顺利执行事务，那么会反馈Yes响应，并进入预备状态，否则反馈No。

- 阶段二PreCommit

  1. 执行事务预提交

    ```
    如果协调者接收到各参与者反馈都是Yes，那么执行事务预提交
    ```

    - A、发送预提交请求
       协调者向各参与者发送preCommit请求，并进入prepared阶段
    - B、事务预提交
       参与者接收到preCommit请求后，会执行事务操作，并将Undo和Redo信息记录到事务日记中
    - C、各参与者向协调者反馈事务执行的响应
       如果各参与者都成功执行了事务操作，那么反馈给协调者Ack响应，同时等待最终指令，提交commit或者终止abort

  2. 中断事务

    如果**任何一个**参与者向协调者反馈了No响应，或者在**等待超时**后，协调者无法接收到所有参与者的反馈，那么就会中断事务。

    - A、发送中断请求

      ```
      协调者向所有参与者发送abort请求
      ```

    - B、中断事务

      ```
      无论是收到来自协调者的abort请求，还是等待超时，参与者都中断事务
      ```

  3. 阶段三doCommit

    - 执行提交

      A、发送提交请求

      假设协调者正常工作，接收到了所有参与者的ack响应，那么它将从预提交阶段进入提交状态，并向所有参与者发送doCommit请求

      B、事务提交

      参与者收到doCommit请求后，正式提交事务，并在完成事务提交后释放占用的资源

      C、反馈事务提交结果

       参与者完成事务提交后，向协调者发送ACK信息

       D、完成事务
       协调者接收到所有参与者ack信息，完成事务

    - 中断事务

    假设协调者正常工作，并且有任一参与者反馈No，或者在等待超时后无法接收所有参与者的反馈，都会中断事务

     - A、发送中断请求

       协调者向所有参与者节点发送abort请求

       B、事务回滚
       参与者接收到abort请求后，利用undo日志执行事务回滚，并在完成事务回滚后释放占用的资源

       C、反馈事务回滚结果
       参与者在完成事务回滚之后，向协调者发送ack信息

       D、中断事务
       协调者接收到所有参与者反馈的ack信息后，中断事务。

  阶段三可能出现的问题：
    **协调者出现问题**、**协调者与参与者之间网络出现故障**。不论出现哪种情况，最终都会导致参与者无法及时接收到来自协调者的doCommit或是abort请求，针对这种情况，**参与者都会在等待超时后，继续进行事务提交（timeout后中断事务）**。

### 优点

降低参与者阻塞范围，并能够在出现单点故障后继续达成一致

### 缺点

引入preCommit阶段，在这个阶段如果出现网络分区，协调者无法与参与者正常通信，参与者依然会进行事务提交，造成数据不一致。

## 租约

租约就是在**一定期限内**给予持有者**特定权力**的协议。每个租约都有一个期限，正是这个期限可以保证租约机制容忍机器失效和网络分割。

### 租约可以用来干什么？

- 进行故障检测。这类似于ZooKeeper中master 与 slaver 之间发送的心跳包的作用。在ZK中， master 和 slaver之间通过交换**心跳包**来检测它们是否还存活。
- 维护缓存一致性。第一种办法是轮询：每次读取数据时都先询问服务器数据是不是最新的，若不是，则先让服务器传输新数据，然后再读取该新数据。第二种方法是回调：由服务器记录有哪些客户端读取了数据，当服务器对数据做修改时先通知记录下来的这些客户端，上次读取过的数据已经失效。这二种方法都有一定的缺陷。因此，可以引入租约机制。在租约期限内，可以保证客户端缓存的数据是最新的。当租约过期后，客户端需要重新向服务器询问数据，重新续约。

## 两将军问题

为了引入该算法，首先提出一种场景，即两将军问题：

> 有两支军队，它们分别有一位将军领导，现在准备攻击一座修筑了防御工事的城市。这两支军队都驻扎在那座城市的附近，分占一座山头。一道山谷把两座山分隔开来，并且两位将军唯一的通信方式就是派各自的信使来往于山谷两边。不幸的是，这个山谷已经被那座城市的保卫者占领，并且存在一种可能，那就是任何被派出的信使通过山谷是会被捕。 请注意，虽然两位将军已经就攻击那座城市达成共识，但在他们各自占领山头阵地之前，并没有就进攻时间达成共识。两位将军必须让自己的军队同时进攻城市才能取得成功。因此，他们必须互相沟通，以确定一个时间来攻击，并同意就在那时攻击。如果只有一个将军进行攻击，那么这将是一个灾难性的失败。

### 三军问题

![image](https://user-images.githubusercontent.com/7789698/38786841-12c90a14-415d-11e8-980b-eede0b8ec1f5.png)

> 1） 1支红军在山谷里扎营，在周围的山坡上驻扎着3支蓝军；
>
> 2） 红军比任意1支蓝军都要强大；如果1支蓝军单独作战，红军胜；如果2支或以上蓝军同时进攻，蓝军胜；
>
> 3） 三支蓝军需要同步他们的进攻时间；但他们惟一的通信媒介是派通信兵步行进入山谷，在那里他们可能被俘虏，从而将信息丢失；或者为了避免被俘虏，可能在山谷停留很长时间；
>
> 4） 每支军队有1个参谋负责提议进攻时间；每支军队也有1个将军批准参谋提出的进攻时间；很明显，1个参谋提出的进攻时间需要获得至少2个将军的批准才有意义；
>
> 5） 问题：是否存在一个协议，能够使得蓝军同步他们的进攻时间？

接下来以两个假设的场景来演绎BasicPaxos；参谋和将军需要遵循一些基本的规则

> 1） 参谋以两阶段提交（prepare/commit）的方式来发起提议，在prepare阶段需要给出一个编号；
>
> 2） 在prepare阶段产生冲突，将军以编号大小来裁决，编号大的参谋胜出；
>
> 3） 参谋在prepare阶段如果收到了将军返回的已接受进攻时间，在commit阶段必须使用这个返回的进攻时间；

## 拜占庭将军问题（Byzantine Generals Problem）

> 一组拜占庭将军分别各率领一支军队共同围困一座城市。为了简化问题，将各支军队的行动策略限定为进攻或撤离两种。因为部分军队进攻部分军队撤离可能会造成灾难性后果，因此各位将军必须通过投票来达成一致策略，即所有军队一起进攻或所有军队一起撤离。因为各位将军分处城市不同方向，他们只能通过信使互相联系。在投票过程中每位将军都将自己投票给进攻还是撤退的信息通过信使分别通知其他所有将军，这样一来每位将军根据自己的投票和其他所有将军送来的信息就可以知道共同的投票结果而决定行动策略。

系统的问题在于，将军中可能出现[叛徒](https://zh.wikipedia.org/wiki/%E5%8F%9B%E5%BE%92)，他们不仅可能向较为糟糕的策略投票，还可能选择性地发送投票信息。假设有9位将军投票，其中1名叛徒。8名忠诚的将军中出现了4人投进攻，4人投撤离的情况。这时候叛徒可能故意给4名投进攻的将领送信表示投票进攻，而给4名投撤离的将领送信表示投撤离。这样一来在4名投进攻的将领看来，投票结果是5人投进攻，从而发起进攻；而在4名投撤离的将军看来则是5人投撤离。这样各支军队的一致协同就遭到了破坏。

由于将军之间需要通过信使通讯，叛变将军可能通过伪造信件来以其他将军的身份发送假投票。而即使在保证所有将军忠诚的情况下，也不能排除信使被敌人截杀，甚至被敌人间谍替换等情况。因此很难通过保证人员可靠性及通讯可靠性来解决问题。

假始那些忠诚（或是没有出错）的将军仍然能通过多数决定来决定他们的战略，便称达到了拜占庭容错。在此，票都会有一个默认值，若消息（票）没有被收到，则使用此默认值来投票。

上述的故事映射到计算机系统里，将军便成了计算机，而信差就是通信系统。虽然上述的问题涉及了电子化的决策支持与信息安全，却没办法单纯的用[密码学](https://zh.wikipedia.org/wiki/%E5%AF%86%E7%A2%BC%E5%AD%B8)与[数字签名](https://zh.wikipedia.org/wiki/%E6%95%B8%E4%BD%8D%E7%B0%BD%E7%AB%A0)来解决。因为不正常的[电压](https://zh.wikipedia.org/wiki/%E9%9B%BB%E5%A3%93)仍可能影响整个加密过程，这不是密码学与数字签名算法在解决的问题。因此计算机就有可能将错误的结果提交去，亦可能导致错误的决策。

## paxos 协议

> 一个叫做Paxos的希腊城邦，这个岛按照议会民主制的政治模式制订法律，但是没有人愿意将自己的全部时间和精力放在这种事情上。所以无论是议员，议长或者传递纸条的服务员都不能承诺别人需要时一定会出现，也无法承诺批准决议或者传递消息的时间。但是这里假设没有拜占庭将军问题（Byzantine failure，即虽然有可能一个消息被传递了两次，但是绝对不会出现错误的消息）；只要等待足够的时间，消息就会被传到。另外，Paxos岛上的议员是不会反对其他议员提出的决议的。

用于达成共识性问题，即对多个节点产生的值，该算法能保证只选出唯一一个值。

### 角色

主要有三类节点：

1. **提议者（Proposer）**：提议一个值；

2. **接受者（Acceptor）**：对每个提议进行投票；

3. **告知者（Learner）**：被告知投票的结果，不参与投票的过程。

   ![image](https://user-images.githubusercontent.com/7789698/38655116-5ac4c404-3e45-11e8-910a-cc69ae55787a.png)

### 定义

1. 决议（value）只有在被proposers提出后才能被批准（未经批准的决议称为“提案（proposal）”）；

2. 在一次Paxos算法的执行实例中，只批准（chosen）一个value；

3. learners只能获得被批准（chosen）的value。


### 约束

> p1: 一个acceptor必须接受（accept）第一次收到的提案。

> p1a：当且仅当acceptor没有回应过编号大于n的prepare请求时，acceptor接受（accept）编号为n的提案。


> p2: 一旦一个具有value v的提案被批准（chosen），那么之后批准（chosen）的提案必须具有value v。
>

> p2a: 一旦一个具有value v的提案被批准（chosen），那么之后任何acceptor再次接受（accept）的提案必须具有value v。
>

> p2b: 一旦一个具有value v的提案被批准（chosen），那么以后任何proposer提出的提案必须具有value v。
>

> p2c: 如果一个编号为n的提案具有value v，那么存在一个多数派，要么他们中所有人都没有接受（accept）编号小于n 的任何提案，要么他们已经接受（accept）的所有编号小于n的提案中编号最大的那个提案具有value v。
>

### Basic-Paxos

#### 执行过程

规定一个提议包含两个字段：[n, v]，其中 n 为序号（具有唯一性），v 为提议值。

1. 下图演示了两个 Proposer 和三个 Acceptor 的系统中运行该算法的初始过程，每个 Proposer 都会**向每个** Acceptor **发送提议请求**。

   ![image](https://user-images.githubusercontent.com/7789698/38655245-285a36ec-3e46-11e8-91ee-44a133d8fb10.png)

2. 当 Acceptor **接收**到一个**提议**请求，包含的提议为 [n1, v1]，并且之前还未接收过提议请求，那么发送一个提议响应，**设置当前接收的提议为 [n1, v1]**，并且保证以后**不会再接受提议值小于 n1 的提议**。

   如下图，Acceptor X 在收到 [n=2, v=8] 的提议请求时，由于之前没有接收过提议，因此就发送一个 [no previous] 的提议响应，并且设置当前接收的提议为 [n=2, v=8]，并且保证以后不会再接受提议值小于 2 的提议。其它的 Acceptor 类似。

   ![image](https://user-images.githubusercontent.com/7789698/38655263-44041b1a-3e46-11e8-848d-a5299cede0be.png)

3. 如果 Acceptor 接受到一个提议请求，包含的提议为 [n2, v2]，并且**之前**已经接收过提议 [**n1**, v1]。如果 **n1 > n2**，那么就**丢弃**该提议请求；否则（**n2>n1**），发送提议响应，该提议响应包含之前已经接收过的提议 [n1, v1]，**设置当前接收的提议为 [n2, v2]**，并且保证以后不会再接受提议值小于 n2 的提议。

   如下图，Acceptor Z 收到 Proposer A 发来的 [n=2, v=8] 的提议请求，由于之前已经接收过 [n=4, v=5] 的提议，并且 n > 2，因此就抛弃该提议请求；Acceptor X 收到 Proposer B 发来的 [n=4, v=5] 的提议请求，因为之前接收的提议为 [n=2, v=8]，并且 2 <= 4，因此就发送 [n=2, v=8] 的提议响应，设置当前接收的提议为 [n=4, v=5]，并且保证以后不会再接受提议值小于 4 的提议。Acceptor Y 类似。

   ![image](https://user-images.githubusercontent.com/7789698/38656437-1e225b4e-3e4d-11e8-8687-3d3dbb8a61e8.png)

4. 当一个 Proposer 接收到**超过一半 Acceptor 的提议响应**时，就可以发送接受请求。

   如下图，Proposer A 接受到两个提议响应之后，就发送 [n=2, v=8] 接受请求。该接受请求会被所有 Acceptor 丢弃，因为此时所有 Acceptor 都保证不接受提议值小于 4 的提议。Proposer B 过后也收到了两个提议响应，因此也开始发送接受请求。需要注意的是，接受请求的 v 需要取它收到的最大 v 值，也就是 8。因此它发送 [n=4, v=8] 的接受请求。

   ![image](https://user-images.githubusercontent.com/7789698/38656170-5df52db6-3e4b-11e8-8479-613a7f8c6589.png)

5. Acceptor 接收到接受请求时，如果**提议号大于等于该 Acceptor 承诺的最小提议号**，那么就发送**通知给所有的 Learner**。当 Learner 发现有大多数的 Acceptor 接收了某个提议，那么该提议的提议值就被 Paxos 选择出来。

   ![image](https://user-images.githubusercontent.com/7789698/38658036-e02e53a2-3e55-11e8-9a86-cd7455d0da33.png)


#### 约束条件

##### 1. 正确性

只有一个提议值会生效。

因为 Paxos 协议要求每个生效的提议被多数 Acceptor 接收，并且 Acceptor 不会接受两个不同的提议，因此可以保证正确性。

##### 2. 可终止性

最后总会有一个提议生效。

Paxos 协议能够让 Proposer 发送的提议朝着能被大多数 Acceptor 接受的那个提议靠拢，因此能够保证可终止性。

#### 缺点

1. **活锁问题**。在base-paxos算法中，不存在leader这样的角色，于是存在这样一种情况，即P1提交了一个proposal n1并且通过了prepare阶段；此时P2提交了一个proposal n2(n2>n1)并且也通过了prepare阶段；P1在commit时因为已经通过了n2而被拒绝；于是P1继续提交一个proposal n3并且通过prepare阶段；巧的是此时P2开始commit了，由于n2<n3再次被拒绝……如此循环往复。这种情况被称为活锁。即整个系统都没死，但由于互相请求资源而被互相锁死。为了不发生活锁的情况，最简单的方式当然是缩减proposer到一个，这样就不会发生互相请求锁死的情况，也即退化。事实上很多后来的工业级协议，都是paxos协议的退化或者变种。


2. **复杂度问题**。base-paxos协议中还存在这样那样的问题，于是各种变种paxos出现了，比如为了解决活锁问题，出现了multi-paxos；为了解决通信次数较多的问题，出现了fast-paxos；为了尽量减少冲突，出现了epaxos。可以看到，工业级实现需要考虑更多的方面，诸如性能，异常等等。这也是为啥许多分布式的一致性框架并非真正基于paxos来实现的原因。
3. **全序问题**。对于paxos算法来说，不能保证两次提交最终的顺序，而zookeeper需要做到这点

信息流：

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(1,Vn)
   |         |<---------X--X--X------>|->|  Accepted(1,Vn)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

Vn = last of (Va,Vb,Vc)

####  Acceptor接收失败的情况

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |          |  |  !       |  |  !! FAIL !!
   |         |<---------X--X          |  |  Promise(1,{Va, Vb, null})
   |         X--------->|->|          |  |  Accept!(1,V)
   |         |<---------X--X--------->|->|  Accepted(1,V)
   |<---------------------------------X--X  Response
   |         |          |  |          |  |
```

#### Learner接收失败

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  |
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(1)
   |         |<---------X--X--X       |  |  Promise(1,{null,null,null})
   |         X--------->|->|->|       |  |  Accept!(1,V)
   |         |<---------X--X--X------>|->|  Accepted(1,V)
   |         |          |  |  |       |  !  !! FAIL !!
   |<---------------------------------X     Response
   |         |          |  |  |       |
```



####  Proposer宕机或发送失败

```
Client  Proposer        Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null, null, null})
   |      |             |  |  |       |  |
   |      |             |  |  |       |  |  !! Leader fails during broadcast !!
   |      X------------>|  |  |       |  |  Accept!(1,V)
   |      !             |  |  |       |  |
   |         |          |  |  |       |  |  !! NEW LEADER !!
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{V, null, null})
   |         X--------->|->|->|       |  |  Accept!(2,V)
   |         |<---------X--X--X------>|->|  Accepted(2,V)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

#### Proposers竞争

```
Client   Leader         Acceptor     Learner
   |      |             |  |  |       |  |
   X----->|             |  |  |       |  |  Request
   |      X------------>|->|->|       |  |  Prepare(1)
   |      |<------------X--X--X       |  |  Promise(1,{null,null,null})
   |      !             |  |  |       |  |  !! LEADER FAILS
   |         |          |  |  |       |  |  !! NEW LEADER (knows last number was 1)
   |         X--------->|->|->|       |  |  Prepare(2)
   |         |<---------X--X--X       |  |  Promise(2,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER recovers
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 2, denied
   |      X------------>|->|->|       |  |  Prepare(2)
   |      |<------------X--X--X       |  |  Nack(2)
   |      |  |          |  |  |       |  |  !! OLD LEADER tries 3
   |      X------------>|->|->|       |  |  Prepare(3)
   |      |<------------X--X--X       |  |  Promise(3,{null,null,null})
   |      |  |          |  |  |       |  |  !! NEW LEADER proposes, denied
   |      |  X--------->|->|->|       |  |  Accept!(2,Va)
   |      |  |<---------X--X--X       |  |  Nack(3)
   |      |  |          |  |  |       |  |  !! NEW LEADER tries 4
   |      |  X--------->|->|->|       |  |  Prepare(4)
   |      |  |<---------X--X--X       |  |  Promise(4,{null,null,null})
   |      |  |          |  |  |       |  |  !! OLD LEADER proposes, denied
   |      X------------>|->|->|       |  |  Accept!(3,Vb)
   |      |<------------X--X--X       |  |  Nack(4)
   |      |  |          |  |  |       |  |  ... and so on ...
```

### Multi-Paxos

唯一的propser

如果leader是相对稳定的，可以跳过第一个阶段。在同一个leader下，每一轮还会递增一个整数。

Multi-Paxos减少了无故障消息延迟（proposal到learning），从4次延迟减少到2次延迟

#### 开始阶段

```
Client   Proposer      Acceptor     Learner
   |         |          |  |  |       |  | --- First Request ---
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Prepare(N)
   |         |<---------X--X--X       |  |  Promise(N,I,{Va,Vb,Vc})
   |         X--------->|->|->|       |  |  Accept!(N,I,Vm)
   |         |<---------X--X--X------>|->|  Accepted(N,I,Vm)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

Vm = last of (Va, Vb, Vc)

#### 稳定状态

```
Client   Proposer       Acceptor     Learner
   |         |          |  |  |       |  |  --- Following Requests ---
   X-------->|          |  |  |       |  |  Request
   |         X--------->|->|->|       |  |  Accept!(N,I+1,W)
   |         |<---------X--X--X------>|->|  Accepted(N,I+1,W)
   |<---------------------------------X--X  Response
   |         |          |  |  |       |  |
```

#### 简化角色

#### 开始阶段

```
Client      Servers
   |         |  |  | --- First Request ---
   X-------->|  |  |  Request
   |         X->|->|  Prepare(N)
   |         |<-X--X  Promise(N,I,{Va,Vb})
   |         X->|->|  Accept!(N,I,Vn)
   |         X<>X<>X  Accepted(N,I)
   |<--------X  |  |  Response
   |         |  |  |
```

#### 稳定阶段

```
Client      Servers
   X-------->|  |  |  Request
   |         X->|->|  Accept!(N,I+1,W)
   |         X<>X<>X  Accepted(N,I+1)
   |<--------X  |  |  Response
   |         |  |  |
```

### Fast-Paxos

Basic Paxos里，客户端到Learner的请求有三个消息的延迟。Fast Paxos允许两个消息的延迟。但是要求1、系统由3f+1个acceptor构成，容许f个错误（取代2f+1个）；2、客户端发送请求到多个目的地

如果leader没有值给Acceptor，Client可以直接给Acceptor发送一个Accept，接下来类似Basic Paxos，Acceptor会回复Leader一个Accepted信息，这样以来就只需要两个消息的延迟久就可以发送给Learner。

如果leader检测到碰撞，leader将会重新发送消息来解决碰撞。这种协调恢复结束将会耗费4个消息延迟

#### 非冲突

```
Client    Leader         Acceptor      Learner
   |         |          |  |  |  |       |  |
   |         X--------->|->|->|->|       |  |  Any(N,I,Recovery)
   |         |          |  |  |  |       |  |
   X------------------->|->|->|->|       |  |  Accept!(N,I,W)
   |         |<---------X--X--X--X------>|->|  Accepted(N,I,W)
   |<------------------------------------X--X  Response(W)
   |         |          |  |  |  |       |  |
```

#### proposals冲突

```
Client   Leader      Acceptor     Learner
 |  |      |        |  |  |  |      |  |
 |  |      |        |  |  |  |      |  |
 |  |      |        |  |  |  |      |  |  !! Concurrent conflicting proposals
 |  |      |        |  |  |  |      |  |  !!   received in different order
 |  |      |        |  |  |  |      |  |  !!   by the Acceptors
 |  X--------------?|-?|-?|-?|      |  |  Accept!(N,I,V)
 X-----------------?|-?|-?|-?|      |  |  Accept!(N,I,W)
 |  |      |        |  |  |  |      |  |
 |  |      |        |  |  |  |      |  |  !! Acceptors disagree on value
 |  |      |<-------X--X->|->|----->|->|  Accepted(N,I,V)
 |  |      |<-------|<-|<-X--X----->|->|  Accepted(N,I,W)
 |  |      |        |  |  |  |      |  |
 |  |      |        |  |  |  |      |  |  !! Detect collision & recover
 |  |      X------->|->|->|->|      |  |  Accept!(N+1,I,W)
 |  |      |<-------X--X--X--X----->|->|  Accepted(N+1,I,W)
 |<---------------------------------X--X  Response(W)
 |  |      |        |  |  |  |      |  |
```

proposals冲突，不协调恢复

```
Client   Leader      Acceptor     Learner
 |  |      |        |  |  |  |      |  |
 |  |      X------->|->|->|->|      |  |  Any(N,I,Recovery)
 |  |      |        |  |  |  |      |  |
 |  |      |        |  |  |  |      |  |  !! Concurrent conflicting proposals
 |  |      |        |  |  |  |      |  |  !!   received in different order
 |  |      |        |  |  |  |      |  |  !!   by the Acceptors
 |  X--------------?|-?|-?|-?|      |  |  Accept!(N,I,V)
 X-----------------?|-?|-?|-?|      |  |  Accept!(N,I,W)
 |  |      |        |  |  |  |      |  |
 |  |      |        |  |  |  |      |  |  !! Acceptors disagree on value
 |  |      |<-------X--X->|->|----->|->|  Accepted(N,I,V)
 |  |      |<-------|<-|<-X--X----->|->|  Accepted(N,I,W)
 |  |      |        |  |  |  |      |  |
 |  |      |        |  |  |  |      |  |  !! Detect collision & recover
 |  |      |<-------X--X--X--X----->|->|  Accepted(N+1,I,W)
 |<---------------------------------X--X  Response(W)
 |  |      |        |  |  |  |      |  |
```

####  不协调恢复、减少角色

```
Client         Servers
 |  |         |  |  |  |
 |  |         X->|->|->|  Any(N,I,Recovery)
 |  |         |  |  |  |
 |  |         |  |  |  |  !! Concurrent conflicting proposals
 |  |         |  |  |  |  !!   received in different order
 |  |         |  |  |  |  !!   by the Servers
 |  X--------?|-?|-?|-?|  Accept!(N,I,V)
 X-----------?|-?|-?|-?|  Accept!(N,I,W)
 |  |         |  |  |  |
 |  |         |  |  |  |  !! Servers disagree on value
 |  |         X<>X->|->|  Accepted(N,I,V)
 |  |         |<-|<-X<>X  Accepted(N,I,W)
 |  |         |  |  |  |
 |  |         |  |  |  |  !! Detect collision & recover
 |  |         X<>X<>X<>X  Accepted(N+1,I,W)
 |<-----------X--X--X--X  Response(W)
 |  |         |  |  |  |
```

## Zab 原子广播协议

Zab的全称是Zookeeper atomic broadcast protocol，是Zookeeper内部用到的一致性协议。相比Paxos，Zab最大的特点是保证强一致性(strong consistency，或叫线性一致性linearizable consistency)。

#### 术语解释

- **quorum**：集群中超过半数的节点集合

ZAB 中的节点有三种状态

- **following**：当前节点是跟随者，服从 leader 节点的命令
- **leading**：当前节点是 leader，负责协调事务
- **election/looking**：节点处于选举状态

*代码实现中多了一种：observing 状态，这是 Zookeeper 引入 Observer 之后加入的，Observer 不参与选举，是只读节点，跟 ZAB 协议没有关系*

节点的持久状态

- **history**：当前节点接收到事务提议的 log
- **acceptedEpoch**：follower 已经接受的 leader 更改年号的 NEWEPOCH 提议
- **currentEpoch**：当前所处的年代
- **zxid**：事务请求的唯一标记，由leader服务器负责进行分配。由2部分构成，高32位是上述的peerEpoch，低32位是请求的计数，从0开始。所以由zxid我们就可以知道该请求是哪个轮次的，并且是该轮次的第几个请求。
- **lastZxid**：history 中最近接收到的提议的 zxid （最大的）
- **lastProcessedZxid**：最后一次commit的事务请求的zxid
- **electionEpoch**：每执行一次leader选举，electionEpoch就会自增，用来标记leader选举的轮次
- **peerEpoch**：每次leader选举完成之后，都会选举出一个新的peerEpoch，用来标记事务请求所属的轮次d

> 在 ZAB 协议的事务编号 Zxid 设计中，Zxid 是一个 64 位的数字，其中低 32 位是一个简单的单调递增的计数器，针对客户端每一个事务请求，计数器加 1；而高 32 位则代表 Leader 周期 epoch 的编号，每个当选产生一个新的 Leader 服务器，就会从这个 Leader 服务器上取出其本地日志中最大事务的ZXID，并从中读取 epoch 值，然后加 1，以此作为新的 epoch，并将低 32 位从 0 开始计数。
>
> epoch：可以理解为当前集群所处的年代或者周期，每个 leader 就像皇帝，都有自己的年号，所以每次改朝换代，leader 变更之后，都会在前一个年代的基础上加 1。这样就算旧的 leader 崩溃恢复之后，也没有人听他的了，因为 follower 只听从当前年代的 leader 的命令。

![image](https://user-images.githubusercontent.com/7789698/38787192-1f444b76-415f-11e8-8e4a-434b73380f99.png)

#### Phase 0: Leader election（选举阶段）

节点在一开始都处于选举阶段，只要有**一个节点得到超半数节点的票数**，它就可以当选**准leader**。只有到达 Phase 3 准 leader 才会成为真正的 leader。这一阶段的目的是就是为了选出一个准 leader，然后进入下一个阶段。

协议并没有规定详细的选举算法，后面我们会提到实现中使用的 Fast Leader Election。

#### Phase 1: Discovery（发现阶段）

在这个阶段，**followers 跟准leader进行通信**，**同步followers最近接收的事务提议**。这个一阶段的主要目的是发现当前大多数节点接收的最新提议，并且**准leader生成新的epoch**，让followers接受，更新它们的acceptedEpoch

![image](https://user-images.githubusercontent.com/7789698/39752060-dc9269ee-52ec-11e8-8782-3287f2829583.png)

**一个 follower 只会连接一个 leader**，如果有一个节点 f 认为另一个 follower p 是 leader，f 在尝试连接 p 时会被拒绝，f 被拒绝之后，就会进入 Phase 0。

#### Phase 2: Synchronization（同步阶段）

同步阶段主要是利用 leader 前一阶段获得的最新提议历史，同步集群中所有的副本。只有当 quorum 都同步完成，准 leader 才会成为真正的 leader。follower 只会接收 zxid 比自己的 lastZxid 大的提议。
![image](https://user-images.githubusercontent.com/7789698/39752118-0e6473a4-52ed-11e8-81c9-242536d38d7f.png)

#### Phase 3: Broadcast（广播阶段）

到了这个阶段，Zookeeper 集群才能正式对外提供事务服务，并且 leader 可以进行消息广播。同时如果有新的节点加入，还需要对新节点进行同步。
![image](https://user-images.githubusercontent.com/7789698/39752157-2888a2dc-52ed-11e8-929e-6887e95fefcc.png)
值得注意的是，ZAB 提交事务并不像 2PC 一样需要全部follower都 ACK，只需要得到quorum（超过半数的节点）的 ACK 就可以了。

### 协议实现

协议的 Java 版本实现跟上面的定义有些不同，选举阶段使用的是 Fast Leader Election（FLE），它包含了 Phase 1 的发现职责。因为 FLE 会选举拥有最新提议历史的节点作为 leader，这样就省去了发现最新提议的步骤。实际的实现将 Phase 1 和 Phase 2 合并为 Recovery Phase（恢复阶段）。所以，ZAB 的实现只有三个阶段：

- **Fast Leader Election**
- **Recovery Phase**
- **Broadcast Phase**

#### Fast Leader Election

前面提到 FLE 会选举拥有最新提议历史（lastZixd最大）的节点作为 leader，这样就省去了发现最新提议的步骤。这是基于拥有最新提议的节点也有最新提交记录的前提。

##### 成为 leader 的条件

1. 选`epoch`最大的
2. `epoch`相等，选 zxid 最大的
3. `epoch`和`zxid`都相等，选择`server id`最大的（就是我们配置`zoo.cfg`中的`myid`）

节点在选举开始都默认投票给自己，当接收其他节点的选票时，会根据上面的条件更改自己的选票并重新发送选票给其他节点，当有一个节点的得票超过半数，该节点会设置自己的状态为 leading，其他节点会设置自己的状态为 following。

##### 选举过程

![image](https://user-images.githubusercontent.com/7789698/39752377-d84a49fa-52ed-11e8-83fb-b4ef3875f52a.png)

##### Recovery Phase （恢复阶段）

这一阶段 follower 发送它们的 lastZixd 给 leader，leader 根据 lastZixd 决定如何同步数据。这里的实现跟前面 Phase 2 有所不同：Follower 收到 TRUNC 指令会中止 L.lastCommittedZxid 之后的提议，收到 DIFF 指令会接收新的提议。

> history.lastCommittedZxid：最近被提交的提议的 zxid
> history:oldThreshold：被认为已经太旧的已提交提议的 zxid

![image](https://user-images.githubusercontent.com/7789698/39752393-e46e974a-52ed-11e8-8c05-d0db59b82a9a.png)



## Raft 协议

在一个由 Raft 协议组织的集群中有三类角色：

- Leader（领袖）
- Follower（群众）
- Candidate（候选人）

### Leader 选举过程

在极简的思维下，一个最小的 Raft 民主集群需要三个参与者（如下图：A、B、C），这样才可能投出多数票。初始状态 ABC 都是 Follower，然后发起选举这时有三种可能情形发生。下图中前二种都能选出 Leader，第三种则表明本轮投票无效（Split Votes），每方都投给了自己，结果没有任何一方获得多数票。之后每个参与方随机休息一阵（Election Timeout）重新发起投票直到一方获得多数票。这里的关键就是随机 timeout，最先从 timeout 中恢复发起投票的一方向还在 timeout 中的另外两方请求投票，这时它们就只能投给对方了，很快达成一致。

![image](https://user-images.githubusercontent.com/7789698/38790076-2c6de066-4171-11e8-82f3-4310c7cb3873.png)

选出 Leader 后，Leader 通过定期向所有 Follower 发送心跳信息维持其统治。若 Follower 一段时间未收到 Leader 的心跳则认为 Leader 可能已经挂了再次发起选主过程。

### Leader 节点对一致性的影响

Raft 协议强依赖 Leader 节点的可用性来确保集群数据的一致性。数据的流向只能从 Leader 节点向 Follower 节点转移。当 Client 向集群 Leader 节点提交数据后，Leader 节点接收到的数据处于未提交状态（Uncommitted），接着 Leader 节点会并发向所有 Follower 节点复制数据并等待接收响应，确保至少集群中超过半数节点已接收到数据后再向 Client 确认数据已接收。一旦向 Client 发出数据接收 Ack 响应后，表明此时数据状态进入已提交（Committed），Leader 节点再向 Follower 节点发通知告知该数据状态已提交。

![image](https://user-images.githubusercontent.com/7789698/38790297-bf18a51c-4172-11e8-974d-3449e0790e56.png)



在这个过程中，主节点可能在任意阶段挂掉，看下 Raft 协议如何针对不同阶段保障数据一致性的。

#### 1. 数据到达 Leader 节点前

这个阶段 Leader 挂掉不影响一致性，不多说。

![image](https://user-images.githubusercontent.com/7789698/39752667-bb36f452-52ee-11e8-981e-6c7373837ad4.png)

#### 2. 数据到达 Leader 节点，但未复制到 Follower 节点

这个阶段 Leader 挂掉，数据属于未提交状态，Client 不会收到 Ack 会认为**超时失败**可安全发起重试。Follower 节点上没有该数据，**重新选主**后 Client **重试重新提交**可成功。原来的 Leader 节点恢复后作为 Follower 加入集群重新从当前任期的新 Leader 处同步数据，强制保持和 Leader 数据一致。

![image](https://user-images.githubusercontent.com/7789698/39752699-ce243002-52ee-11e8-8fc2-2c17081470f0.png)

#### 3. 数据到达 Leader 节点，成功复制到 Follower 所有节点，但还未向 Leader 响应接收

这个阶段 Leader 挂掉，虽然数据在 Follower 节点处于未提交状态（Uncommitted）但保持一致，重新选出 Leader 后可完成数据提交，此时 Client 由于不知到底提交成功没有，可**重试提交**。针对这种情况 Raft 要求 RPC 请求实现幂等性，也就是要实现内部去重机制。

![image](https://user-images.githubusercontent.com/7789698/39752713-dbf0420c-52ee-11e8-8c06-e801a03cd2a4.png)

#### 4. 数据到达 Leader 节点，成功复制到 Follower 部分节点，但还未向 Leader 响应接收

这个阶段 Leader 挂掉，数据在 Follower 节点处于未提交状态（Uncommitted）且不一致，Raft 协议要求投票只能投给拥有最新数据的节点。所以拥有最新数据的节点会被选为 Leader 再强制同步数据到 Follower，数据不会丢失并最终一致。

![image](https://user-images.githubusercontent.com/7789698/39752738-e92947d4-52ee-11e8-999c-f53e42de92ea.png)

#### 5. 数据到达 Leader 节点，成功复制到 Follower 所有或多数节点，数据在 Leader 处于已提交状态，但在 Follower 处于未提交状态

这个阶段 Leader 挂掉，重新选出新 Leader 后的处理流程和阶段 3 一样。

![image](https://user-images.githubusercontent.com/7789698/39752753-f84889fa-52ee-11e8-9e3d-2a0fa6952882.png)

#### 6. 数据到达 Leader 节点，成功复制到 Follower 所有或多数节点，数据在所有节点都处于已提交状态，但还未响应 Client

这个阶段 Leader 挂掉，Cluster 内部数据其实已经是一致的，Client 重复重试基于幂等策略对一致性无影响。

![image](https://user-images.githubusercontent.com/7789698/39752773-06c7f826-52ef-11e8-8e3c-a0815ab94efc.png)

#### 7. 网络分区导致的脑裂情况，出现双 Leader

网络分区将原先的 Leader 节点和 Follower 节点分隔开，Follower 收不到 Leader 的心跳将发起选举产生新的 Leader。这时就产生了双 Leader，原先的 Leader 独自在一个区，向它提交数据不可能复制到多数节点所以永远提交不成功。向新的 Leader 提交数据可以提交成功，网络恢复后旧的 Leader 发现集群中有更新任期（Term）的新 Leader 则自动降级为 Follower 并从新 Leader 处同步数据达成集群数据一致。

![image](https://user-images.githubusercontent.com/7789698/39753101-300ada22-52f0-11e8-95e5-d19fc3955ed6.png)

### 图解

1. 刚开始所有的节点都是follower。

![image](https://user-images.githubusercontent.com/7789698/38790429-dc07722e-4173-11e8-8ebb-6298a3a46af8.png)

2. 如果follower不能接收到leader的消息就成为candidate。这里有个选举超时时间election timeout，这个时间是随机在150ms到300ms之间，第一个到达超时时间恢复的节点将会认为自己是candidate，并且投自己。



![image](https://user-images.githubusercontent.com/7789698/38790454-f736b474-4173-11e8-8c99-1a5cb429bae9.png)

3. candidate发送投票请求，请求选举自己为leader。如果follow节点尚未投票则返回响应给候选人投票且重置自己的election timeout。

![image](https://user-images.githubusercontent.com/7789698/38790495-30cb3bf6-4174-11e8-9312-9cc50da7ef79.png)

![image](https://user-images.githubusercontent.com/7789698/38790385-8af85808-4173-11e8-9e15-e392d86a5fc7.png)



4. 候选人获得超过半数的票则被选举为leader。

![image](https://user-images.githubusercontent.com/7789698/38790512-5afcc674-4174-11e8-91f8-e10c60f9bbdf.png)

5. leader停止将会触发重新选举

![image](https://user-images.githubusercontent.com/7789698/38790657-4bb35f24-4175-11e8-9916-5e5ed1dcc821.png)

![image](https://user-images.githubusercontent.com/7789698/38790664-54980a86-4175-11e8-9d37-0fc915b4c945.png)

6. 当有两个节点同时成为候选人。节点已经投票的时候将不会在接受后续投票请求。一旦没有超过半数投票，就会等待超时重新投票。

![image](https://user-images.githubusercontent.com/7789698/38790720-9c80060a-4175-11e8-950b-cb2c2a379e2e.png)![image](https://user-images.githubusercontent.com/7789698/38790726-a3766aee-4175-11e8-89dd-3b4f3d21deb4.png)

![image](https://user-images.githubusercontent.com/7789698/38790739-b76b6608-4175-11e8-994c-2db0082a89a1.png)



7. leader将会采用两段式提交的方法发送log，这一个阶段也叫做log replication。这时候每个节点将会有一个被指定的heartbeat timeout。首先，客户端发送的变更将会增加到leader的log里面。然后，这个log将会被在heartbeat timeout结束之前发送到followers。接下来，超过半数的节点响应接受了这个请求则leader会进入commit状态，并且会返回给client响应。最后，leader会提交给followers commit状态。

![image](https://user-images.githubusercontent.com/7789698/38792973-e7a9c204-4181-11e8-8cba-c954e2b35ea2.png)![image](https://user-images.githubusercontent.com/7789698/38792979-edbc5bac-4181-11e8-89dc-bf8d00973869.png)![image](https://user-images.githubusercontent.com/7789698/38793077-4865746c-4182-11e8-9dd9-46af5e615b92.png)
![image](https://user-images.githubusercontent.com/7789698/38793087-4ffb7f78-4182-11e8-8324-d9e1a4762e53.png)

![image](https://user-images.githubusercontent.com/7789698/38793134-82a6b3a2-4182-11e8-8097-d912dda1e05e.png)

![image](https://user-images.githubusercontent.com/7789698/38793177-b650267a-4182-11e8-88e5-2cd39ec3e0ff.png)

![image](https://user-images.githubusercontent.com/7789698/38793222-f709e782-4182-11e8-94b6-fbc177386855.png)

![image](https://user-images.githubusercontent.com/7789698/38793251-172ea7be-4183-11e8-89cb-78c2a9e21760.png)







8. 当raft节点面临网络分区的时候，也能够保持一致。比如我们现在把节点A、B和节点C、D、E分区。这时候节点C、D、E将会重新选举出节点E。这时候我们尝试用两个客户端分别与两个分区通信：1.第一个客户端向第一个分区发送set 3的请求的时候，由于节点B没法得到超过半数的响应所以将会停留在uncommitted状态 2.第二个客户端发送set 8 请求的时候，由于节点E能得到超过半数的响应（包括自己3个），所以将会形成一次完整的复制。当网络分区请求得到恢复之后：节点B做为leader接收到了term更高的leader的消息，便下线成为follow

![image](https://user-images.githubusercontent.com/7789698/38793354-82d6dbf8-4183-11e8-8845-fdf31420b8c6.png)![image](https://user-images.githubusercontent.com/7789698/38793367-904b8676-4183-11e8-9d8e-1604b54eebef.png)

![image](https://user-images.githubusercontent.com/7789698/38793437-e12c9f30-4183-11e8-8500-aae5f3cab84d.png)

![image](https://user-images.githubusercontent.com/7789698/38793564-6377404e-4184-11e8-9b60-2ce830d11ed0.png)![image](https://user-images.githubusercontent.com/7789698/38793763-1a5d0ca8-4185-11e8-99dc-f3cd935312d7.png)



参考：

http://mp.weixin.qq.com/s?__biz=MzI4NDMyNTU2Mw==&mid=2247483815&idx=1&sn=74ee0b591ada2c24b6bd13f8d3171670&chksm=ebfc6273dc8beb65113c585ece8f5feda02e4be29684d9d8f8a173b25986374cc755c80b6ca5&scene=21#wechat_redirect

http://www.cnblogs.com/hapjin/p/4748603.html

https://segmentfault.com/a/1190000004474543

http://blog.csdn.net/liweisnake/article/details/69253206

https://my.oschina.net/pingpangkuangmo/blog/778927

https://zh.wikipedia.org/wiki/%E6%8B%9C%E5%8D%A0%E5%BA%AD%E5%B0%86%E5%86%9B%E9%97%AE%E9%A2%98

https://mp.weixin.qq.com/s?__biz=MzI4NDMyNTU2Mw==&mid=2247483815&idx=1&sn=74ee0b591ada2c24b6bd13f8d3171670&chksm=ebfc6273dc8beb65113c585ece8f5feda02e4be29684d9d8f8a173b25986374cc755c80b6ca5&scene=21#wechat_redirect

http://blog.jobbole.com/104985/

http://www.infoq.com/cn/articles/raft-paper

[In search of an Understandable Consensus Algorithm (Extended Version)](https://ramcloud.atlassian.net/wiki/download/attachments/6586375/raft.pdf)

https://www.cnblogs.com/mindwind/p/5231986.html

http://thesecretlivesofdata.com/raft/