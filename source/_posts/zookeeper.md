---
title: zookeeper
abbrlink: 3ad834c9
date: 2017-11-11 23:17:51
tags: [zookeeper,笔记]
categories: 一致性
---

**CAP** 

> CAP由Eric Brewer在2000年PODC会议上提出[1][2]，是Eric Brewer在Inktomi[3]期间研发搜索引擎、分布式web缓存时得出的关于数据一致性(consistency)、服务可用性(availability)、分区容错性(partition-tolerance)的猜想：

- 数据一致性(consistency)：如果系统对一个写操作返回成功，那么之后的读请求都必须读到这个新数据；如果返回失败，那么所有读操作都不能读到这个数据，对调用者而言数据具有强一致性(strong consistency) (又叫原子性 atomic、线性一致性 linearizable consistency)[5]
- 服务可用性(availability)：所有读写请求在一定时间内得到响应，可终止、不会一直等待
- 分区容错性(partition-tolerance)：在网络分区的情况下，被分隔的节点仍能正常对外服务

在某时刻如果满足AP，分隔的节点同时对外服务但不能相互通信，将导致状态不一致，即不能满足C；如果满足CP，网络分区的情况下为达成C，请求只能一直等待，即不满足A；如果要满足CA，在一定时间内要达到节点状态一致，要求不能出现网络分区，则不能满足P。

C、A、P三者最多只能满足其中两个，和FLP定理一样，CAP定理也指示了一个不可达的结果(impossibility result)。

在满足分区容错的前提下，没有算法能同时满足数据一致性和服务可用性[11]：

P 是必选项，那3选2的选择题不就变成数据一致性(consistency)、服务可用性(availability) 2选1？工程实践中一致性有不同程度，可用性也有不同等级，在保证分区容错性的前提下，放宽约束后可以兼顾一致性和可用性，两者不是非此即彼[12]。

<!-- more -->

- 序列一致性(sequential consistency)[13]：不要求时序一致，A操作先于B操作，在B操作后如果所有调用端读操作得到A操作的结果，满足序列一致性
- 最终一致性(eventual consistency)[14]：放宽对时间的要求，在被调完成操作响应后的某个时间点，被调多个节点的数据最终达成一致

可用性在CAP定理里指所有读写操作必须要能终止，实际应用中从主调、被调两个不同的视角，可用性具有不同的含义。当P(网络分区)出现时，主调可以只支持读操作，通过牺牲部分可用性达成数据一致。

工程实践中，较常见的做法是通过异步拷贝副本(asynchronous replication)、quorum/NRW，实现在调用端看来数据强一致、被调端最终一致，在调用端看来服务可用、被调端允许部分节点不可用(或被网络分隔)的效果[15]。

例如延时(latency)，它是衡量系统可用性、与用户体验直接相关的一项重要指标[16]。CAP理论中的可用性要求操作能终止、不无休止地进行，除此之外，我们还关心到底需要多长时间能结束操作，这就是延时，它值得我们设计、实现分布式系统时单列出来考虑。

# Zookeeper
临时结点（ephemeral）：1.创建该node的客户端会话超时或主动关闭2.某客户端删除

持久化结点（persistent）

有序结点（sequential）：自增整数会被加到路径之后

监听（watch）和通知（notification）：客户端向zk注册需要接收通知的znode，通过对znode设置监视点来接收通知

###  使用：
前台运行并查看输出
`bin/zkServer.sh start-foreground`

![image](https://user-images.githubusercontent.com/7789698/32696391-7873f59a-c73c-11e7-975c-5a8489cc0e75.png)



`create /absurd '"'`

`delete /absurd`

## 会话的状态和声明周期

CONNECTING、CONNECTED、CLOSED、NOT_CONNECTED

![image](https://user-images.githubusercontent.com/7789698/32696552-b3a6a82a-c740-11e7-9c24-4d0af715b5e0.png)

![image](https://user-images.githubusercontent.com/7789698/32696435-d422efd0-c73d-11e7-975b-04c227aa24c2.png)

### 会话超时时间
如果经过T时间后服务接收不到会话任何消息，服务就会声明会话过期。客户端侧，如果经过T/3时间未接收到任何消息，客户端将发送心跳消息。经过2T/3时间后，开始寻找其他服务器。当尝试连接到不同服务器时，服务器的zk要与最后连接的服务器的zk状态保持最新。客户端不能连接到 一个未发现更新而客户端已经发现的更新，Zookeeper通过在服务中排序更新操作来决定新鲜程度(Freshness)。每一个对Zookeeper布局状态的改动操作相对于所有其它执行的更新操作都是全序的，所以如果一个客户端已经在位置i观察到一个更新，它不能连接一个仅看到i' < i的服务器。在ZooKeeper的实现中，系统分配给每个更新操作一个事务ID来建立这个顺序。


![image](https://user-images.githubusercontent.com/7789698/32696557-cda6df88-c740-11e7-84c5-f9344ab2ab26.png)
客户端因为超时和s1断开连接后，它尝试连接s2，但是s2已经落后了，并不能反应客户端已知的更新。而s3已经看到和客户端一样看到的更新，所以它被安全连接。

> server.1=127.0.0.1:2222:2223
> server.2=127.0.0.1:3333:3334
> server.3=127.0.0.1:4444:4445

ip(hostname):port(仲裁):port(群首选举)

> mkdir z1
> mkdir z1/data
> echo 1> z1/data/myid
> mkdir z2
> mkdir z2/data
> echo 2> z2/data/myid
> mkdir z3
> mkdir z3/data
> echo 3> z3/data/myid


设置监视点：
getData、getChildren、exists都可以在读取的znode设置监视点
会话状态（KeeperState）：Disconnected (0)、 SyncConnected (3)、AuthFailed (4)、 ConnectedReadOnly (5)、SaslAuthenticated(6)、Expired (-112);
事件状态（EventType）：None (-1)、 NodeCreated (1)、 NodeDeleted (2)、NodeDataChanged (3)、NodeChildrenChanged (4);

监视点一旦设置就无法移除，1.触发这个监视点2.使其会话关闭或者过期

如果想无限监听，使用ZkClient的subscribeDataChanges或者CuratorFramework的TreeCache.getListenable().addListener


 ```
       //监听指定节点的数据变化
        zk.subscribeDataChanges(nodeName, new IZkDataListener() {
            @Override
            public void handleDataChange(String s, Object o) throws Exception {
                logger.info("node data changed!");
                logger.info("node=>" + s);
                logger.info("data=>" + o);
                logger.info("--------------");
            }

            @Override
            public void handleDataDeleted(String s) throws Exception {
                logger.info("node data deleted!");
                logger.info("s=>" + s);
                logger.info("--------------");

            }
        });
 ```
```
        TreeCache treeCache = new TreeCache(client,path);
        treeCache.getListenable().addListener( new TreeCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
                ChildData data = event.getData();
                if(data != null){
                    switch (event.getType()){
                        case NODE_ADDED:
                        case NODE_REMOVED:
                        case NODE_UPDATED:
                            System.out.println(event.getType().name()+":"+new String(event.getData().getData()));
                            break;
                        default:
                            break;
                    }
                }
            }
        });
        treeCache.start();
```