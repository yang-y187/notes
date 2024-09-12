# 前言 ZNode

Paxos描述了这样一个场景，有一个叫做Paxos的小岛(Island)上面住了一批居民，岛上面所有的事情由一些特殊的人决定，他们叫做议员(Senator)。议员的总数(Senator Count)是确定的，不能更改。岛上每次环境事务的变更都需要通过一个提议(Proposal)，每个提议都有一个编号(PID)，这个编号是一直增长的，不能倒退。每个提议都需要超过半数((Senator Count)/2 +1)的议员同意才能生效。每个议员只会同意大于当前编号的提议，包括已生效的和未生效的。如果议员收到小于等于当前编号的提议，他会拒绝，并告知对方：你的提议已经有人提过了。这里的当前编号是每个议员在自己记事本上面记录的编号，他不断更新这个编号。整个议会不能保证所有议员记事本上的编号总是相同的。现在议会有一个目标：保证所有的议员对于提议都能达成一致的看法。
好，现在议会开始运作，所有议员一开始记事本上面记录的编号都是0。有一个议员发了一个提议：将电费设定为1元/度。他首先看了一下记事本，嗯，当前提议编号是0，那么我的这个提议的编号就是1，于是他给所有议员发消息：1号提议，设定电费1元/度。其他议员收到消息以后查了一下记事本，哦，当前提议编号是0，这个提议可接受，于是他记录下这个提议并回复：我接受你的1号提议，同时他在记事本上记录：当前提议编号为1。发起提议的议员收到了超过半数的回复，立即给所有人发通知：1号提议生效！收到的议员会修改他的记事本，将1好提议由记录改成正式的法令，当有人问他电费为多少时，他会查看法令并告诉对方：1元/度。
现在看冲突的解决：假设总共有三个议员S1-S3，S1和S2同时发起了一个提议:1号提议，设定电费。S1想设为1元/度, S2想设为2元/度。结果S3先收到了S1的提议，于是他做了和前面同样的操作。紧接着他又收到了S2的提议，结果他一查记事本，咦，这个提议的编号小于等于我的当前编号1，于是他拒绝了这个提议：对不起，这个提议先前提过了。于是S2的提议被拒绝，S1正式发布了提议: 1号提议生效。S2向S1或者S3打听并更新了1号法令的内容，然后他可以选择继续发起2号提议。
好，我觉得Paxos的精华就这么多内容。现在让我们来对号入座，看看在ZK Server里面Paxos是如何得以贯彻实施的。
小岛(Island)——ZK Server Cluster
议员(Senator)——ZK Server
提议(Proposal)——ZNode Change(Create/Delete/SetData…)
提议编号(PID)——Zxid(ZooKeeper Transaction Id)
正式法令——所有ZNode及其数据



# 1、ZooKeeper 简介

- Zookeeper主要服务于分布式系统，用来，统一配置管理、统一命名管理、分布式锁和集群管理
- ZooKeeper作为分布式的中间件解决了分布式系统中无法避免的对结点管理的问题（需要实时感知结点的状态、对节点进行统一管理等等）



使用Zookeeper的项目

- Kafka：主要为Kafka提供Broker和topic的注册一级多个Partiton的负载均衡等功能
- Dubbo：使用Zookeeper为期命名服务，维护各个机器的信息，维护全局的服务地址列表



**Zookeeper 的三种运行模式**

- 单机模式：开发测试环境需要
- 集群模式：Zookeeper集群通常由一组机器组成，一般是3台机器网上，每台机器都在内存中维护了当前的服务状态，并且每台机器进行通信
- 伪集群模式：集群所有的机器都在一台机器上，Zookeeper允许暴露不同的接口，则一台机器可以部署多个Zookeeper服务实例。



**Zookeeper的特点**

![img](zookeeper.assets/cc9a79e4e3d2eeb91daa131960154435.png)

1. Zookeeper的支持集群部署的，集群由一个领导者和多个跟随者组成
2. **高可用性：**集群中只要有半数以上节点存活，则Zookeeper集群就能正常服务
3. **全局一致性：**每个Service保存相同的数据副本，Client无论连接到哪个service，数据均是一致的
4. **更新请求顺序执行**：来自同一个client的更新请求按期发送顺序依次执行
5. **数据更新原子性：**更新数据要么都成功，要么都失败
6. **实时性**：Client能读取到最新的数据
7. 从设计模式的角度来说，Zk是一个观察者设计模式的框架，它负责管理和存储数据，给接收观察者的注册，数据发生变化会通知各个观察者做出反应。
8. Zookeeper保证的**CP（数据一致性）**





# 2、CAP和BASE理论

CAP定理：系统需要在系统可用性和数据一致性中做出权衡。 C：一致性   A： 可用性   P： 分区容忍性

分布式系统都是接收P的，因此主要的区别是CP（数据一致性）还是AP（可用性）。**Zookeeper保证的是CP（数据一致性）**。Eureka保证了AP（可用性）。



**BASE理论**：无法做到强一致性，分布式系统可通过自己的业务特点采取适当的方式来使得系统达到最终的一致性。分为三个部分

- **基本可用（Basically Available）**：分布式系统在出现故障的时候，允许损失部分可用性，保证核心功能的高可用。电商大促时，需要先加入购物车再下单
- **软状态（Soft State）**：允许系统存在中间状态，但最终是一致的，且中间状态不影响系统整体的可用性。分布式中存在副本，副本同步的延时则是中间态
- **最终一致性（Eventual Consistency）**：经过软状态后，最终达到一致装填，这是**弱一致性**，强一致性则不允许有中间态

平时要求系统基本可用，运行有可容忍的延迟装填，但经过一段时间后，达到最终的一致性。

**Zookeeper是让系统尽可能的高可用，而且数据能达到最终的一致性。**



# 3、一致性协议 TAB

Zookeeper为解决分布式数据一致性问题，自己定制了一致性协议 ZAB（Zookeeper Automic Broadcast）原子广播协议，能够支持崩溃恢复



## 3.1 ZAB的三个角色

Leader 领导者、Follower 跟随者、Observer 观察者。统称为zkService

- Leader：集群中唯一的写请求处理者，能够发起投票
- Follower：能够接收客户端的请求，如果是读请求则，自己处理，若是写请求，转发给Leader。选举过程中，会参与投票，有选举和被选举权
- Observer：没有选举权和被选举权的Follower

ZAB的协议，则是对该三种角色的协议，分为 **消息传播**和**崩溃恢复**



## 3.2 ZXID和myid

**zxid**

zookeeper采用全局递增的事物Id来标识。所有的变更发生时都会有一个Zookeeper Transaction Id。ZXID 是64位的Long类型。**各个角色根据该Id保证事务顺序一致性。**ZXID 高32位表示纪元epoch，低32位表示xid。

![在这里插入图片描述](zookeeper.assets/cfb646b6c1cf54c061c7435942411d80.png)

- 每个leader 都会有不同的epoch值。表示一个朝代。从1开始，每次新的选举，选出新的leader后，epoch递增1。并且将该值更新到所有zkService的epoch。
- xid是一个递增的事务编号。数值越大说明数据越新。每次epoch变化，都将低32位的xid重置。

**myid**

每个Zookeeper服务器都会在数据文件夹下创建myid的文件，这个文件包含整个Zookeeper集群的唯一性ID（整数）.



## 3.3 历史队列

每个follower节点都会有一个先进先出（FIFO）队列用来存放收到的事物请求，保证事物的顺序。

- 可靠提交由ZAB的事务一致性协议保证
- 全局有序由TCP协议保证
- 因果有序由follower的历史队列(history queue)保证



## 3.4 消息广播模式

消息广播就是ZkService如何处理写请求和数据同步。

**ZAB协议两种模式：消息广播模式和崩溃恢复模式。**

![在这里插入图片描述](zookeeper.assets/eafeea1c853a67ac5c6181ac81b2f1a1.png)

- ### 写请求

ZkService收到写请求后，转发到Leader。Leader根据写请求，询问所有的Follower是否同意更新，要是超过半数同意，则将Follower和Observer更新。**数据同步是保证事务的顺序一致性的。**根据版本的Id  ZXID进行有序同步

![在这里插入图片描述](zookeeper.assets/8daca995415d58496bfa389f4203a655.png)

1. Zkserver收到写请求转发到 leader
2. leader生成一个新的事务，并生成ZXID。
3. Leader 将事务发给所有的Follower，携带ZXID作为一个提案（propose）分发给所有Follower
4. follower节点将收到的事务请求加到历史队列（history queue）中，当follower收到提案propose，现将propose写到磁盘，完成后再回复leader一个ACK。
5. 当leader收到超过半数的follower的ack消息，leader向follower发送commit请求（leader自身也需要提交事务）
6. 当follower收到commit请求后，判断当前事务的ZXID是否比历史队列里的事务ZXID都小。如果是则执行该事务。若不是则等待比它更小的事务的执行（保证有序性）
7. Leader将执行结果返给客户端



**过半成功策略**

Leader节点接收到写请求后，Leader将会把写请求广播给各个server。各个Server将该写请求加入到历史队列，并向Leader发送Ack信息。收到超过半数的Ack信息后，说明该操作可以执行。Leader会向各个Server提交commit消息，各个Server收到消息后执行操作

- Leader不需要得到Observer的Ack（Observer无投票权）
- Leader不需要得到所有Follower的Ack，只需要得到一半即可（Leader本身对自己就有一个Ack）
- Observer虽然无投票权，但仍同步Leader数据，从而在处理读请求时，可以返回新数据



- Follower 和Observer都有可能收到写请求，zkserver收到请求后，将该请求转发到Leader
- 无论是Follower和Observer，都会执行commit命令，即写请求

读请求的话，任何zkServer都可以处理，在本地内存中读取



## 3.5 崩溃恢复模式

 崩溃恢复模式分为四步：选举、发现、同步、广播

1. 选举阶段：当Leader崩溃后，集群进度选举阶段，选出准leader
2. 发现阶段：从各个节点中发现最新的ZXID和事务日志。准Leader接收所有Follower的各自的epoch值。Leader从中选取最大的epoch + 1，将新值发给各个Follower。Follower回应Ack给Leader，带上各自最大的ZXID和存储的信息。Leader选取最大的ZXID，并更新自身历史日志。Leader有了最新的提议历史（每次epoch变化时，ZXID重新从0开始）
3. 同步阶段：Leader获取到最新的提议历史，则同步所有Follower。超过半数Follower同步成功，准Leader才成为正式Leader。（再同步提议时，follower只同步zxid比自己lastZxid大的提议）
4. 广播阶段：集群恢复到广播模式，开始接受客户端的写请求。



- **Leader被选为主节点后，已经是最新的数据，为什么还要从各节点寻找最新的事务？**

  - **确保已经被Leader提交的提案最终被所有Follower提交**

    - Leader(server2)发送了commit请求，然后发给server3，还没发给server1时，挂了。若选举server1作为新Leader，则server2收到的commit请求，则会被丢弃。
    - **在选举时，比较ZXID**，只有最新的server3当选Leader。当前Leader后，将最新的Leader同步到Follower。若是server2恢复，则作为Follower加入集群

    ![在这里插入图片描述](zookeeper.assets/941001e25663ddf3a304190acf12f5c1.png)

  - **确保跳过那些已经被丢弃的提案**
    - Leader（server2）统一了提案N1,自身提交了提议，没提交到server1和server3时挂了。无论是server1还是server3选举成Leader。server2恢复后作为Follower加入集群，同步Leader的提案历史，sever2的提案N1会被丢弃。
      ![在这里插入图片描述](zookeeper.assets/1322c36948cb00bfa4a3d22673f687d2.png)

## 3.6 脑裂问题

当前集群需要选举leader，当所有节点通信没有问题时，会选举出一个master。若通信出现了问题，至少两个节点被选为leader，所以一个集群有两个及以上个leader。

ZAB为解决脑裂问题，要求集群内的结点数量为2N+1，当网络分裂后，始终有一个集群的结点数量超过半数，而另一个集群节点数量小于N+1（小于半数），**选举需要过半数节点同意。**所以不会有两个leader。

因此，过半机制，对于Zookeeper集群，要么没有leader，要么只有一个1个leader，由此避免了脑裂问题。





























































