## 什么是Zab协议？

Zab协议 的全称是 **Zookeeper Atomic Broadcast** （Zookeeper原子广播）。
 **Zookeeper 是通过 Zab 协议来保证分布式事务的最终一致性**。

1. Zab协议是为分布式协调服务Zookeeper专门设计的一种 **支持崩溃恢复** 的 **原子广播协议** ，是Zookeeper保证数据一致性的核心算法。Zab借鉴了Paxos算法，但又不像Paxos那样，是一种通用的分布式一致性算法。**它是特别为Zookeeper设计的支持崩溃恢复的原子广播协议**。
2. 在Zookeeper中主要依赖Zab协议来实现数据一致性，基于该协议，zk实现了一种主备模型（即Leader和Follower模型）的系统架构来保证集群中各个副本之间数据的一致性。
    这里的主备系统架构模型，就是指只有一台客户端（Leader）负责处理外部的写事务请求，然后Leader客户端将数据同步到其他Follower节点。

## Zab协议核心

Zab协议的核心：**定义了事务请求的处理方式**

1）所有的事务请求必须由一个全局唯一的服务器来协调处理，这样的服务器被叫做 **Leader服务器**。其他剩余的服务器则是 **Follower服务器**。

2）Leader服务器 负责将一个客户端事务请求，转换成一个 **事务Proposal**，并将该 Proposal 分发给集群中所有的 Follower 服务器，也就是向所有 Follower 节点发送数据广播请求（或数据复制）

3）分发之后Leader服务器需要等待所有Follower服务器的反馈（Ack请求），**在Zab协议中，只要超过半数的Follower服务器进行了正确的反馈**后（也就是收到半数以上的Follower的Ack请求），那么 Leader 就会再次向所有的 Follower服务器发送 Commit 消息，要求其将上一个 事务proposal 进行提交。

## Zab协议内容

Zab 协议包括两种基本的模式：**崩溃恢复** 和 **消息广播**

#### 协议过程

当整个集群启动过程中，或者当 Leader 服务器出现网络中弄断、崩溃退出或重启等异常时，Zab协议就会 **进入崩溃恢复模式**，选举产生新的Leader。

当选举产生了新的 Leader，同时集群中有过半的机器与该 Leader 服务器完成了状态同步（即数据同步）之后，Zab协议就会退出崩溃恢复模式，**进入消息广播模式**。

这时，如果有一台遵守Zab协议的服务器加入集群，因为此时集群中已经存在一个Leader服务器在广播消息，那么该新加入的服务器自动进入恢复模式：找到Leader服务器，并且完成数据同步。同步完成后，作为新的Follower一起参与到消息广播流程中。

#### 协议状态切换

当Leader出现崩溃退出或者机器重启，亦或是集群中不存在超过半数的服务器与Leader保存正常通信，Zab就会再一次进入崩溃恢复，发起新一轮Leader选举并实现数据同步。同步完成后又会进入消息广播模式，接收事务请求。

#### 保证消息有序

在整个消息广播中，Leader会将每一个事务请求转换成对应的 proposal 来进行广播，并且在广播 事务Proposal 之前，Leader服务器会首先为这个事务Proposal分配一个全局单递增的唯一ID，称之为事务ID（即zxid），由于Zab协议需要保证每一个消息的严格的顺序关系，因此必须将每一个proposal按照其zxid的先后顺序进行排序和处理。

## 消息广播

1）在zookeeper集群中，数据副本的传递策略就是采用消息广播模式。zookeeper中农数据副本的同步方式与二段提交相似，但是却又不同。二段提交要求协调者必须等到所有的参与者全部反馈ACK确认消息后，再发送commit消息。要求所有的参与者要么全部成功，要么全部失败。二段提交会产生严重的阻塞问题。

2）Zab协议中 Leader 等待 Follower 的ACK反馈消息是指“只要半数以上的Follower成功反馈即可，不需要收到全部Follower反馈”

**zookeeper 采用 Zab 协议的核心，就是只要有一台服务器提交了 Proposal，就要确保所有的服务器最终都能正确提交 Proposal。这也是 CAP/BASE 实现最终一致性的一个体现。**

**Leader 服务器与每一个 Follower 服务器之间都维护了一个单独的 FIFO 消息队列进行收发消息，使用队列消息可以做到异步解耦。 Leader 和 Follower 之间只需要往队列中发消息即可。如果使用同步的方式会引起阻塞，性能要下降很多。**

## 崩溃恢复

**一旦 Leader 服务器出现崩溃或者由于网络原因导致 Leader 服务器失去了与过半 Follower 的联系，那么就会进入崩溃恢复模式。**

#### Zab 协议如何保证数据一致性

假设两种异常情况：
 1、一个事务在 Leader 上提交了，并且过半的 Folower 都响应 Ack 了，但是 Leader 在 Commit 消息发出之前挂了。
 2、假设一个事务在 Leader 提出之后，Leader 挂了。

要确保如果发生上述两种情况，数据还能保持一致性，那么 Zab 协议选举算法必须满足以下要求：

**Zab 协议崩溃恢复要求满足以下两个要求**：
 1）**确保已经被 Leader 提交的 Proposal 必须最终被所有的 Follower 服务器提交**。
 2）**确保丢弃已经被 Leader 提出的但是没有被提交的 Proposal**。

根据上述要求
 Zab协议需要保证选举出来的Leader需要满足以下条件：
 1）**新选举出来的 Leader 不能包含未提交的 Proposal** 。
 即新选举的 Leader 必须都是已经提交了 Proposal 的 Follower 服务器节点。
 2）**新选举的 Leader 节点中含有最大的 zxid** 。
 这样做的好处是可以避免 Leader 服务器检查 Proposal 的提交和丢弃工作。

#### Zab 如何数据同步

1）完成 Leader 选举后（新的 Leader 具有最高的zxid），在正式开始工作之前（接收事务请求，然后提出新的 Proposal），Leader 服务器会首先确认事务日志中的所有的 Proposal 是否已经被集群中过半的服务器 Commit。

2）Leader 服务器需要确保所有的 Follower 服务器能够接收到每一条事务的 Proposal ，并且能将所有已经提交的事务 Proposal 应用到内存数据中。等到 Follower 将所有尚未同步的事务 Proposal 都从 Leader 服务器上同步过啦并且应用到内存数据中以后，Leader 才会把该 Follower 加入到真正可用的 Follower 列表中。


#### Zab 数据同步过程中，如何处理需要丢弃的 Proposal

在 Zab 的事务编号 zxid 设计中，zxid是一个64位的数字。

其中低32位可以看成一个简单的单增计数器，针对客户端每一个事务请求，Leader 在产生新的 Proposal 事务时，都会对该计数器加1。而高32位则代表了 Leader 周期的 epoch 编号。

> epoch 编号可以理解为当前集群所处的年代，或者周期。每次Leader变更之后都会在 epoch 的基础上加1，这样旧的 Leader 崩溃恢复之后，其他Follower 也不会听它的了，因为 Follower 只服从epoch最高的 Leader 命令。

每当选举产生一个新的 Leader ，就会从这个 Leader 服务器上取出本地事务日志充最大编号 Proposal 的 zxid，并从 zxid 中解析得到对应的 epoch 编号，然后再对其加1，之后该编号就作为新的 epoch 值，并将低32位数字归零，由0开始重新生成zxid。

**Zab 协议通过 epoch 编号来区分 Leader 变化周期**，能够有效避免不同的 Leader 错误的使用了相同的 zxid 编号提出了不一样的 Proposal 的异常情况。

基于以上策略
 **当一个包含了上一个 Leader 周期中尚未提交过的事务 Proposal 的服务器启动时，当这台机器加入集群中，以 Follower 角色连上 Leader 服务器后，Leader 服务器会根据自己服务器上最后提交的 Proposal 来和 Follower 服务器的 Proposal 进行比对，比对的结果肯定是 Leader 要求 Follower 进行一个回退操作，回退到一个确实已经被集群中过半机器 Commit 的最新 Proposal**。

## 协议实现

协议的 Java 版本实现跟上面的定义略有不同，选举阶段使用的是 Fast Leader Election（FLE），它包含了步骤1的发现指责。因为FLE会选举拥有最新提议的历史节点作为 Leader，这样就省去了发现最新提议的步骤。

**Fast Leader Election（快速选举）**
 前面提到的 FLE 会选举拥有最新Proposal history （lastZxid最大）的节点作为 Leader，这样就省去了发现最新提议的步骤。**这是基于拥有最新提议的节点也拥有最新的提交记录**

-  **成为 Leader 的条件：**
   1）选 epoch 最大的
   2）若 epoch 相等，选 zxid 最大的
   3）若 epoch 和 zxid 相等，选择 server_id 最大的（zoo.cfg中的myid）

## 2.2 不建议使用zookeeper 作为注册中心的场景和原因

不建议使用zookeeper 的原因是当它没满足A带来的影响。
![img](https://img2018.cnblogs.com/blog/1271798/202002/1271798-20200204174327871-473885430.png)

当机房 3 出现网络分区 (Network Partitioned) 的时候，即机房 3 在网络上成了孤岛，我们知道虽然整体 ZooKeeper 服务是可用的，但是节点 ZK5 是不可写的，因为联系不上 Leader。

也就是说，这时候机房 3 的应用服务 svcB 是不可以新部署，重新启动，扩容或者缩容的，但是站在网络和服务调用的角度看，机房 3 的 svcA 虽然无法调用机房 1 和机房 2 的 svcB, **但是与机房 3 的 svcB 之间的网络明明是 OK 的啊，为什么不让我调用本机房的服务**？

现在因为注册中心自身为了保脑裂 (P) 下的数据一致性（C）而放弃了可用性，导致了同机房的服务之间出现了无法调用，这是绝对不允许的！可以说**在实践中，注册中心不能因为自身的任何原因破坏服务之间本身的可连通性**，这是注册中心设计应该遵循的铁律

#### 2.3 zookeeper 的拓展

ZooKeeper 的写并不是可扩展的，不可以通过加节点解决水平扩展性问题。

要想在 ZooKeeper 基础上硬着头皮解决服务规模的增长问题，一个实践中可以考虑的方法是想办法梳理业务，垂直划分业务域，将其划分到多个 ZooKeeper 注册中心，但是作为提供通用服务的平台机构组，因自己提供的服务能力不足要业务按照技术的指挥棒配合划分治理业务，真的可行么？

而且这又违反了因为注册中心自身的原因（能力不足）破坏了服务的可连通性，举个简单的例子，1 个搜索业务，1 个地图业务，1 个大文娱业务，1 个游戏业务，他们之间的服务就应该老死不相往来么？也许今天是肯定的，那么明天呢，1 年后呢，10 年后呢？谁知道未来会要打通几个业务域去做什么奇葩的业务创新？注册中心作为基础服务，无法预料未来的时候当然不能妨碍业务服务对未来固有联通性的需求。

#### 2.4 zookeeper 的持久化存储

ZooKeeper 的 ZAB 协议对每一个写请求，会在每个 ZooKeeper 节点上保持写一个事务日志，同时再加上定期的将内存数据镜像（Snapshot）到磁盘来保证数据的一致性和持久性，以及宕机之后的数据可恢复，这是非常好的特性，但是我们要问，在服务发现场景中，其最核心的数据 - 实时的健康的服务的地址列表是不需要数据持久化的

需要持久化存储的地方在于一个完整的生产可用的注册中心，除了服务的实时地址列表以及实时的健康状态之外，还会存储一些服务的元数据信息，例如服务的版本，分组，所在的数据中心，权重，鉴权策略信息，service label 等元信息，这些数据需要持久化存储，并且注册中心应该提供对这些元信息的检索的能力。

#### 2.5 容灾能力

如果注册中心（Registry）本身完全宕机了，服务A 调用 服务B 链路应该受到影响么？
![img](https://img2018.cnblogs.com/blog/1271798/202002/1271798-20200204174343965-1238148315.png)

是的，不应该受到影响。

服务调用（请求响应流）链路应该是弱依赖注册中心，必须仅在服务发布，机器上下线，服务扩缩容等必要时才依赖注册中心。

这需要注册中心仔细的设计自己提供的客户端，客户端中应该有针对注册中心服务完全不可用时做容灾的手段，例如设计客户端缓存数据机制（我们称之为 client snapshot）就是行之有效的手段。另外，注册中心的 health check 机制也要仔细设计以便在这种情况不会出现诸如推空等情况的出现。

ZooKeeper 的原生客户端并没有这种能力，所以利用 ZooKeeper 实现注册中心的时候我们一定要问自己，如果把 ZooKeeper 所有节点全干掉，你生产上的所有服务调用链路能不受任何影响么？而且应该定期就这一点做故障演练。

#### zookeeper 的健康检查

使用 ZooKeeper 作为服务注册中心时，服务的健康检测常利用 ZooKeeper 的 Session 活性 Track 机制 以及结合 Ephemeral ZNode 的机制，简单而言，就是将服务的健康监测绑定在了 ZooKeeper 对于 Session 的健康监测上，或者说绑定在 TCP 长链接活性探测上了。

这在很多时候也会造成致命的问题，ZK 与服务提供者机器之间的 TCP 长链接活性探测正常的时候，该服务就是健康的么？答案当然是否定的！注册中心应该提供更丰富的健康监测方案，服务的健康与否的逻辑应该开放给服务提供方自己定义，而不是一刀切搞成了 TCP 活性检测！

健康检测的一大基本设计原则就是尽可能真实的反馈服务本身的真实健康状态，否则一个不敢被服务调用者相信的健康状态判定结果还不如没有健康检测。

## zookeeper适合的场景

## 我们能用zookeeper做什么

### 1、 命名服务

​        这个似乎最简单，在zookeeper的文件系统里创建一个目录，即有唯一的path。在我们使用tborg无法确定上游程序的部署机器时即可与下游程序约定好path，通过path即能互相探索发现，不见不散了。

 

### 2、 配置管理

​        程序总是需要配置的，如果程序分散部署在多台机器上，要逐个改变配置就变得困难。好吧，现在把这些配置全部放到zookeeper上去，保存在 Zookeeper 的某个目录节点中，然后所有相关应用程序对这个目录节点进行监听，一旦配置信息发生变化，每个应用程序就会收到 Zookeeper 的通知，然后从 Zookeeper 获取新的配置信息应用到系统中就好。

​                                         ![zookeeper简介](http://static.open-open.com/lib/uploadImg/20141108/20141108213345_625.png)

 

### 3、 集群管理

所谓集群管理无在乎两点：是否有机器退出和加入、选举master。

​        对于第一点，所有机器约定在父目录GroupMembers下创建临时目录节点，然后监听父目录节点的子节点变化消息。一旦有机器挂掉，该机器与 zookeeper的连接断开，其所创建的临时目录节点被删除，所有其他机器都收到通知：某个兄弟目录被删除，于是，所有人都知道：它上船了。新机器加入 也是类似，所有机器收到通知：新兄弟目录加入，highcount又有了。

​        对于第二点，我们稍微改变一下，所有机器创建临时顺序编号目录节点，每次选取编号最小的机器作为master就好。

​                                 ![zookeeper简介](http://static.open-open.com/lib/uploadImg/20141108/20141108213345_947.png)

 

### 4、  分布式锁

​        有了zookeeper的一致性文件系统，锁的问题变得容易。锁服务可以分为两类，一个是保持独占，另一个是控制时序。

​        对于第一类，我们将zookeeper上的一个znode看作是一把锁，通过createznode的方式来实现。所有客户端都去创建 /distribute_lock 节点，最终成功创建的那个客户端也即拥有了这把锁。厕所有言：来也冲冲，去也冲冲，用完删除掉自己创建的distribute_lock 节点就释放出锁。

​        对于第二类， /distribute_lock 已经预先存在，所有客户端在它下面创建临时顺序编号目录节点，和选master一样，编号最小的获得锁，用完删除，依次方便。

​                                         ![zookeeper简介](http://static.open-open.com/lib/uploadImg/20141108/20141108213345_5.png)

### 5、队列管理

两种类型的队列：

1、 同步队列，当一个队列的成员都聚齐时，这个队列才可用，否则一直等待所有成员到达。

2、队列按照 FIFO 方式进行入队和出队操作。

第一类，在约定目录下创建临时目录节点，监听节点数目是否是我们要求的数目。

第二类，和分布式锁服务中的控制时序场景基本原理一致，入列有编号，出列按编号

## 关于zookeeper的脑裂和假死

假死：由于心跳超时（网络原因导致的）认为master死了，但其实master还存活着。
脑裂：由于假死会发起新的master选举，选举出一个新的master，但旧的master网络又通了，导致出现了两个master ，有的客户端连接到老的master 有的客户端链接到新的master。

### 一般解决脑裂问题方案：

- 法定人数（Quorums）：比如3个节点的集群，Quorums = 2, 也就是说集群可以容忍1个节点失效，这时候还能选举出1个lead，集群还可用。比如4个节点的集群，它的Quorums = 3，Quorums要超过3，相当于集群的容忍度还是1，如果2个节点失效，那么整个集群还是无效的。
- 冗余通信的方式，集群中采用多种通信方式，防止一种通信方式失效导致集群中的节点无法通信。
-  共享资源的方式（Fencing）：比如能看到共享资源就表示在集群中，能够获得**共享资源的锁**的就是Leader，看不到共享资源的，就不在集群中。

### zookeeper解决方案：

​	**zooKeeper默认采用了Quorums这种方式来防止"脑裂"现象**。即只有集群中超过半数节点投票才能选举出Leader。这样的方式可以确保leader的唯一性,要么选出唯一的一个leader,要么选举失败。在zookeeper中Quorums作用如下：

- 集群中最少的节点数用来选举leader保证集群可用。
-  通知客户端数据已经安全保存前集群中最少数量的节点数已经保存了该数据。一旦这些节点保存了该数据，客户端将被通知已经安全保存了，可以继续其他任务。而集群中剩余的节点将会最终也保存了该数据。

**假设某个leader假死，其余的followers选举出了一个新的leader。这时，旧的leader复活并且仍然认为自己是leader，这个时候它向其他followers发出写请求也是会被拒绝的**。因为每当新leader产生时，会生成一个epoch标号(标识当前属于那个leader的统治时期)，这个epoch是递增的，followers如果确认了新的leader存在，知道其epoch，就会拒绝epoch小于现任leader epoch的所有请求。那有没有follower不知道新的leader存在呢，有可能，但肯定不是大多数，否则新leader无法产生。**Zookeeper的写也遵循quorum机制，因此，得不到大多数支持的写是无效的，旧leader即使各种认为自己是leader，依然没有什么作用。**

zookeeper除了可以采用上面默认的Quorums方式来避免出现"脑裂"，还可以可采用下面的预防措施：
1、添加冗余的心跳线，例如双线条线，尽量减少“裂脑”发生机会。
2、启用磁盘锁。正在服务一方锁住共享磁盘，"裂脑"发生时，让对方完全"抢不走"共享磁盘资源。但使用锁磁盘也会有一个不小的问题，如果占用共享盘的一方不主动"解锁"，另一方就永远得不到共享磁盘。现实中假如服务节点突然死机或崩溃，就不可能执行解锁命令。后备节点也就接管不了共享资源和应用服务。于是有人在HA中设计了"智能"锁。即正在服务的一方只在发现心跳线全部断开（察觉不到对端）时才启用磁盘锁。平时就不上锁了。
3、设置仲裁机制。例如设置参考IP（如网关IP），当心跳线完全断开时，2个节点都各自ping一下 参考IP，不通则表明断点就出在本端，不仅"心跳"、还兼对外"服务"的本端网络链路断了，即使启动（或继续）应用服务也没有用了，那就主动放弃竞争，让能够ping通参考IP的一端去起服务。更保险一些，ping不通参考IP的一方干脆就自我重启，以彻底释放有可能还占用着的那些共享资源。

## 关于zookeeper分布式锁

zk节点有个唯⼀的特性，就是我们创建过这个节点了，你再创建zk是会报错的，那我们就利⽤⼀下他的
唯⼀性去实现分布式锁。

层层深入：

1. 如果只是用上述去创建分布式锁，那么如何确保释放锁？
   	用finnally释放？

2. 那如何达到阻塞效果呢？
          其他的线程全部死循环去创建节点？

3. 那如果加锁后程序宕机呢，如何释放锁？

   ​	你又说用临时节点，client端开就删除？

4. 怎么知道前⾯的⽼哥删除节点了嗯？
           监听节点删除事件？

5. 但是仍然有一个不好的地方，就是多个线程都监听该节点的话，一旦删除触发，会导致瞬间很多线程请求创建，服务器也受不了
         故以上方式不可行！

因此最终的zk的分布式锁形态应该是**临时顺序节点**！

- 加锁时创建一个临时顺序节点，并获取当前目录所有节点，若自己为最小的节点则加锁成功，否则加锁失败
- 第一次加锁失败时，后续只需要比对自己节点是否是当前最小节点，如果是则加锁成功
- 和之前监听⼀个永久节点的区别就在于，这⾥每个节点只监听了⾃⼰的前⼀个节点，释放当然也是⼀个
  个释放下去，就不会出现⽺群效应了。

**基于curator封装的zookeper的实现主要有下面四类:**

InterProcessMutex：分布式可重入排它锁
InterProcessSemaphoreMutex：分布式不可重入排它锁
InterProcessReadWriteLock：分布式读写锁
InterProcessMultiLock：将多个锁作为单个实体管理的容器

```
如：InterProcessLock lock = new InterProcessMutex(client, lockPath);
```

 

```java
//用curator自实现分布式锁
public class DistributeLock {

    private final static String LOCK_PARENT = "/chronos";
    private final static String LOCK_PATH = "/lock";

    private String path;

    private String beforePath;

    private CountDownLatch countDownLatch = new CountDownLatch(1);

    private static ExponentialBackoffRetry retry = new ExponentialBackoffRetry(1000, 3);
    private static CuratorFramework client = CuratorFrameworkFactory.builder()
            .connectString("10.0.60.26:2181")
            .connectionTimeoutMs(10000)
            .sessionTimeoutMs(10000)
            .retryPolicy(retry).build();

    static {
        client.start();
    }

    public Boolean lock() {

        try {
            System.out.println("进入获取锁创建节点方法");
            //第一步先建立自己的临时顺序节点
            path = client.create().creatingParentsIfNeeded().withMode(CreateMode.EPHEMERAL_SEQUENTIAL)
                    .forPath(LOCK_PARENT + LOCK_PATH, UUID.randomUUID().toString().getBytes());
            Thread.sleep(20000);
            List<String> list = client.getChildren().forPath(LOCK_PARENT);
            Collections.sort(list);
            int i;
            for (i = 0; i < list.size(); i++) {
                if (path.equalsIgnoreCase(LOCK_PARENT + "/" + list.get(i))) {
                    break;
                }
            }
            if (i == 0) {
                //当前节点为第一个节点
                return true;
            } else {
                //当前节点不为第一个节点，且记录前一个节点
                beforePath = LOCK_PARENT + "/" + list.get(i - 1);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return false;
    }

    public void waitForLock() throws Exception {
        //通过NodeCache 来监控节点值的变化。
        NodeCache nodeCache = new NodeCache(client, beforePath, false);
        nodeCache.start();
        nodeCache.getListenable().addListener(() -> {
            ChildData currentData = nodeCache.getCurrentData();
            if (currentData == null) {//表示节点被删除
                System.out.println(path + "监听到前节点" + beforePath + "发生变化,变化后的值为" + currentData + ",开始获取锁");
                countDownLatch.countDown();
            }
        });

        if (client.checkExists().forPath(beforePath) != null) {
            System.out.println(path + "前一个节点存在，获取锁失败，开始阻塞");
            countDownLatch.await();
        }
    }

    public void unlock() throws Exception {
        System.out.println(path + "开始释放锁");
        client.delete().forPath(path);
    }


    public static void main(String[] args) throws Exception {
        Runnable task = () -> {
            try {
                DistributeLock zkLock = new DistributeLock();
                Boolean flag = zkLock.lock();
                System.out.println("线程" + Thread.currentThread().getName() + "加锁返回：" + flag);
                //加锁成功则执行逻辑，否则就重试
                if (flag) {
                    System.out.println("当前创建的节点值为" + zkLock.path);
                    Thread.sleep(20000);
                } else {
                    System.out.println("当前创建的节点值为" + zkLock.path);
                    //否则进行等待
                    zkLock.waitForLock();
                }
                zkLock.unlock();
            } catch (Exception e) {
                e.printStackTrace();
            }
        };

        for (int i = 0; i < 10; i++) {
            new Thread(task, "线程" + i).start();
        }
    }
}
```

运行日志如下：

```shell
线程线程1加锁返回：true
当前创建的节点值为/chronos/lock0000000074
线程线程8加锁返回：false
当前创建的节点值为/chronos/lock0000000077
线程线程4加锁返回：false
当前创建的节点值为/chronos/lock0000000075
线程线程6加锁返回：false
当前创建的节点值为/chronos/lock0000000076
线程线程7加锁返回：false
当前创建的节点值为/chronos/lock0000000079
线程线程0加锁返回：false
当前创建的节点值为/chronos/lock0000000080
线程线程2加锁返回：false
当前创建的节点值为/chronos/lock0000000078
线程线程3加锁返回：false
当前创建的节点值为/chronos/lock0000000081
线程线程9加锁返回：false
当前创建的节点值为/chronos/lock0000000082
线程线程5加锁返回：false
当前创建的节点值为/chronos/lock0000000083
/chronos/lock0000000075前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000081前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000079前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000078前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000080前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000076前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000077前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000082前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000083前一个节点存在，获取锁失败，开始阻塞
/chronos/lock0000000074开始释放锁
/chronos/lock0000000075监听到前节点/chronos/lock0000000074发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000075开始释放锁
/chronos/lock0000000076监听到前节点/chronos/lock0000000075发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000076开始释放锁
/chronos/lock0000000077监听到前节点/chronos/lock0000000076发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000077开始释放锁
/chronos/lock0000000078监听到前节点/chronos/lock0000000077发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000078开始释放锁
/chronos/lock0000000079监听到前节点/chronos/lock0000000078发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000079开始释放锁
/chronos/lock0000000080监听到前节点/chronos/lock0000000079发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000080开始释放锁
/chronos/lock0000000081监听到前节点/chronos/lock0000000080发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000081开始释放锁
/chronos/lock0000000082监听到前节点/chronos/lock0000000081发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000082开始释放锁
/chronos/lock0000000083监听到前节点/chronos/lock0000000082发生变化,变化后的值为null,开始获取锁
/chronos/lock0000000083开始释放锁

```

