# ZooKeeper
<!--ts-->
<!--te-->
## ZooKeeper概念

- ZooKeeper是一个分布式协调服务的开源框架。主要用来解决分布式集群中应用系统的一致性问题，例如怎样避免同时操作同一数据造成脏读的问题。ZooKeeper本质上是一个分布式的小文件存储系统。提供基于类似文件系统的目录树方式的数据存储，并且可以对树中的节点进行有效管理，从而来维护和监控你存储的数据的状态变化。将通过监控这些数据的状态的变化，从而可以达到基于数据的集群管理。例如：统一命名服务(dubbo)、分布式配置管理、分布式消息队列、分布式锁、分布式协调等功能。
- Zookeeper是一个典型的分布式数据一致性解决方案，分布式应用程序可以基于ZooKeeper实现诸如数据发布/订阅、负载均衡、命名服务、分布式协调/通知、集群管理、Master选举、分布式锁和分布式队列等功能。ZooKeeper一个最常用的功能就是用于担任服务生产者和服务消费者的注册中心。

## ZooKeeper架构图
1. Leader：ZooKeeper集群工作的核心事务请求的唯一调度和处理者，保证集群事务处理的顺序性；集群内部各个服务的调度者。对于create、setData、delete等有写操作的请求，需要统一转发给leader处理，leader需要决定编号、执行操作、这个过程称为一个事务。
2. Follower：处理客户端非事务，请求转发事务请求给Leader，参与集群leader选举投票2n-1台可以做集群投票。此外对于访问量比较大的zooKeeper集群，还可以新增观察者角色。
3. Observer：观察者角色。观察ZooKeeper集群的最新状态变化并将这些变化同步过来，其对于非事务请求可以进行独立处理，对于事务请求，则转发给Leader服务器处理，不参与任何形式的投票，只提供服务，通常用于在不影响集群事务处理能力的前提下提升集群的非事务处理能力。(增加并发的请求)

## ZooKeeper的特点
1. 全局一致性：每个server都保存一份相同的副本，不论client访问哪台server，访问到的数据都是一致的。
2. 可靠性：如果消息被一台服务器接收，那么将被所有服务器接受
3. 顺序性：全局有序性和偏序：全局有序：如果一台服务器上消息a在消息b前发布，那么所有server上消息a在消息b前发布。偏序性：如果一个消息b在消息a后被同一个发送者发布，则a必须排在b前面。通过单长连接通道交互的，中间通过队列缓存。
4. 数据操作的原子性：一次数据更新要么成功、要么失败。
5. 实时性：ZooKeeper保证客户端将在一段时间间隔内收到服务端的更新信息，或者服务器失效的信息。

## ZAB协议
- ZAB协议是为ZooKeeper设计的崩溃恢复原子广播协议，它保证zooKeeper集群数据的一致性和命令的全局有序性。
### 概念介绍
1. 集群角色：Leader(同一时间内只允许有一个leader存在，提供对客户端的读写功能，负责将数据同步到各个节点)、Follower(提供对客户端的读功能，写请求转发给Leader，当leader失效后参与投票)、Observer(与Follower不同的是不参与投票)。
2. 服务状态：LOOKING，FOLLOWING，LEADING，OBSERVERING。**ZAB的状态分为**:ELECTION，DISCOVERY(连上leader，响应leader心态，并检测leader的角色是否更改，通过此步骤之后选举出来的leader才能执行真正的职务)，SYNCHRONIZATION，BROADCAST。
3. ZXID，是一个long型的整数，分为两部分，epoch和计数器(counter)部分，是有一个全局有效的数字。epoch代表当前集群所属哪个leader。epoch代表当前命令的有效性。counter是一个递增的数字。
### 选举
- 时机：(1)集群当开始启动的时候，还没有leader，需要进行选举。(2)当因为各种意外情况，leader服务器宕机，需要进行leader选举。
- 投票规则：按照epoch(纪元)，zxid和serverId来进行依次选择较大的。由此可以看出尽量把服务性能更大的集群的serverId配置大一些。
- 过程：首先会将票投给自己，然后将自己的投票信息进行广播，当收到其他的投票信息后，决定是否更改投票信息。投票完成后，进行统计投票信息，如果集群中过半的机器都选择了某一台机器，则该机器成为新的leader，选举结束。
### 广播(集群对外提供服务，如何保证各个节点数据的一致性)
- zab协议在广播状态保证以下特征：
1. 可靠传递：如果消息m被一台服务器传递，那么它将被全部服务器传递。
2. 全局有序：如果一个消息a在消息b之前被一台服务器交付，所有服务器都交付了a和b，并且a先于b。
3. 因果有序：如果消息a因果上先于消息b，并且两者都被交付，那么a必须排在b前面。
### 写请求
- 整个写请求类似一个二阶段的提交。首先leader收到客户端的写请求后，会生成一个事务(proposal)，其中包含了zxid。leader开始广播该事务，**需要注意的是所有节点的通讯都是由一个FIFO队列维护的**。follower收到事务后，将事务写入本地磁盘，写入成功后并返回一个ack。当leader收到过半的ack后，会进行事务的提交，并广播事务提交信息。从节点开始提交事务。
- ZooKeeper通过二阶段提交来保证集群中数据的一致性，因为只要收到过半的ack就可以提交事务，所以ZooKeeper不是强一致性的。
- zab协议的有序性保证是通过几个方面来体现的：(1)服务之间用的是TCP通讯，保证在网络中的有序性。(2)通讯使用的是FIFO队列，保证消息的全局一致性。(3)通过全局递增zxid来保证因果有序性。
### 分布式锁

## Watcher监听机制和它的原理
ZooKeeper可以提供分布式数据的发布订阅功能，依赖的就是Watcher监听机制。
客户端可以向服务器端注册Watcher监听，服务端的指定事件触发之后，就会向客户端发送一个事件通知。
有几个特性：
1. 一次性：一旦一个Watcher被触发，Zookeeper就会将它从存储中移除。
2. 客户端串行：客户端的Watcher回调处理是串行同步的过程，不要因为一个Watcher的逻辑阻塞整个客户端。
3. 轻量
主要流程如下：
1. 客户端向服务端注册Watcher监听
2. 保存Watcher对象到客户端本地的WatcherManager中。
3. 服务端Watcher事件触发后，客户端收到服务端通知，从WatcherManager中取出响应Watcher对象执行回调逻辑。
## 如何保持数据一致性
使用ZAB协议来保证数据的最终一致性，类型一个2PC两阶段提交的过程。主要流程与写请求类似。
## 如何进行leader选举
时机：分为启动时的leader选举与运行期间的leader选举
## 数据同步
在选举完成之后，Follower和Observer就会去向leader注册，然后就会开始数据同步工作。
数据同步包含3个主要值和4种形式。
PeerLastZxid：learner(follower和observer)服务器最后处理的ZXID。
minCommitedLog：Leader提议缓存队列中最小的zxid
maxCommitedLog：Leader提议缓存队列中最大的zxid
1. 直接差异化同步，DIFF同步：PeerLastZxid处于min与max之间
2. 回滚再差异化同步：对于只存在于learner中的提议，先回滚至离max最近的位置，再进行同步
3. 仅回滚同步：PeerLastZxid大于max
4. 全量同步：PeerLastZxid小于min
## 数据不一致性问题