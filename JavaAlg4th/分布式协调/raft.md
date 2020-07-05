# 1. Raft 简介

* 学习 Raft 的理由

Paxos 在 Raft 算法出现前，一直作为共识算法的代表。Raft 设计者给出了两个 Paxos 的主要缺点：

1. Paxos 算法难以理解，也很难实现。

   一个Paxos拥有3中角色，proposer、acceptor、learner

   proposer：负责发起提议；acceptor：与所有proposer一起选举一个提议；learner：学习已通过的提议；

   我们可以体验一下Basic Paxos的一次协调流程：

   1. Proposer 提出提案，广播prepare请求到所有Acceptor；
   2. Acceptor 回复收到的最大proposal提议id；
   3. Proposer 根据多数（过半）的Acceptor回复的提议，发送给所有Acceptor最终提议；
   4. 在多数（过半）Acceptor响应提议后，Proposer形成<font color='red'>决议</font>，并广播到所有Learner；

   整个写入主要有两个阶段：（1）prepare 选主提议（步骤1~3）（2）提议 commit（步骤3~4）；两次的quorum选举，期间出现延时、分区、节点宕机等等情况下，因为故障恢复的必要性，给整个算法的复杂度和限制条件是很难把控的。

   整个流程需要两次quorum选举，效率肯定会比较低下，所以就出现了Multi-Paxos算法（这个不是很了解，后面看了再补充）

   Paxos 也有很多变种算法，针对不同场景的优化：

   Disk Paxos(2002); Cheap Paxos(2003); Fast Paxos(2004); Generalized Paxos(2005); Stoppable Paxos(2008); Vertical Paxos(2009)

   具体的实现也有很多：

   Google Chubby；Google Spanner；Ceph；Neo4j；Amazon Elastic Container Service

   

* Raft 工作原理

Raft 是一个基于”日志复制同步“来实现的一种共识算法。

![Raft1](file:///Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/%E5%88%86%E5%B8%83%E5%BC%8F%E5%8D%8F%E8%B0%83/pics/Raft1.png?lastModify=1593936969)

<center>图 1 - Raft 工作原理，选主（Leader Election) -> 日志提交 -> 日志本地应用</center>

Client 的所有变更可以视为一个的指令（command），当指令到达 Server 后先形成操作日志（log），然后将操作日志按顺序应用到状态机（State Machine)，称为”可复制的状态机架构“。

只要各个 Server 上的操作日志在顺序和内容上一致，我们就可以认为节点之间的状态机是一致的，所以问题就转换成：如何保证节点之间的日志是顺序一致的。

Raft 算法比 Paxos 容易理解的主要原因，在于Raft 算法工作流程是可以拆解的（包括故障恢复）

<center>选主（Leader Election），日志复制（Log replication），安全性，集群节点变化（Membership Changes）</center>

下面就 Raft 切分的各个流程进行介绍。



# 2. Raft 选主

Raft 集群节点跟 Paxos 类似，分为3个角色：Leader、Candidate、Follower

* Leader：Raft 集群一个时刻有且最多存在一个Leader节点，所有变更操作都由Leader节点提交；
* Candidate：只有意识到Leader不存在的时候，Follower节点自发将自己提升为Candidate角色，对外发起选举；正常工作状态下不会存在这类角色的节点；
* Follower：负责接收和提交Leader广播的日志和客户端的请求，集群初始化的时候，所有节点都是出于Follower角色；

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Server-States.png" alt="Raft-Server-States" style="zoom:50%;" />

<center>图 2 - Raft 节点状态转移图</center>

## 2.1 Raft-Leader 任期（Term）

只要操作能在全局定序，那么操作之间的先后关系就很容易得到。在分布式系统中，事件定序可以依赖<font color='red'>物理时钟</font>和<font color='red'>逻辑时钟</font>。

* 事件的定序方式

偏序：就是发送在同一节点上的事件（对于同一个Object）必定存在因果关系，可以定序；而不同节点上的事件（对于同一个Object）则视为无序。实现的例子有：Lamport的矢量时钟

全序：每一个事件都可以按一定逻辑进行全局定序，比如：节点间按 IP 有确定的优先级，加上节点本地时间存着因果关系，所以一个事件就可以在全局进行定序；也可以利用类似于Google的True Time，利用卫星时间+最大网络延时预估对每个事件进行一个时间发配，来表示事件的全局顺序。实现的例子有：Spanner的True Time，单Leader主从架构

* *逻辑时钟*

  不同节点之间，由于地理位置不同、时钟晶振漂移等问题，很难保证节点之间时钟的一致性，导致在不能节点上处理的事件，无法真实反映出真实世界中事件发生的顺序，这时候就出现了逻辑时钟，常见的逻辑时钟有：

  （1）矢量时钟（Vector Clock)

  （2）逻辑时钟（Lamport Timestamp）

* *物理时钟*

  （1）Spanner True Time

Raft 的日志定序依据： Term + logIndex，我们先看下 Term 的概念。

Raft Term也可以算是逻辑时钟的一种，在一个Leader任期内，Term的值不会变化；当Candidate发起选主时，根据本地的Term值加1。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Term.png" alt="Raft-Term z" style="zoom:50%;" />

<center>图 3 - 单调递增的Term</center>

## 2.2 Raft 选举

### 2.2.1 选举行为触发条件

不管是 日志复制请求AppendEntries 还是 选举请求RequestVote，都会带上发起请求节点本地的Term

1. 当Leader收到Candidate的RequestVote，且请求的Term大于本地的Term，Leader马上转为Follower（这个时候就没有Leader发心跳了）；
2. 当Follower在一定时间内没有收到Leader心跳（<font color='red'>AppendEntries除了提交日志也有心跳的作用</font>，心跳检查消息也只是一些不含任何数据信息的 AppendEntries 远程调用），主动切换到Candidate角色发起选举；
3. Candidate节点发起选举后，经过一定时间得不到多数（过半）的选票，会重新发起新一轮（上一次选举Term+1）选举；

当节点发送请求并收到应答时，应答Term比本地Term大，也会转换为Follower角色。（回顾角色状态转移图）

### 2.2.2 选举机制

1. Candidate、Follower都可以参与选举投票；
2. 每个Candidate节点在投自己票后，向其他节点广播Request Vote请求，所以Candidate节点至少有一票；
3. 如果节点接收到RequestVote中的Term比自己本地Term小，则拒绝投票；
4. 每个节点投票后会记录投票时请求的Term，同一个Term下仅可以给自己除外的一个节点投票；如果RequestVote中的Term比自己曾经投过票的Term小（或等于），则拒绝投票；
5. 节点收到的RequestVote请求中，请求中的日志包含节点本地的所有日志时（最新日志「Term + LastLogIndex」大于节点的「Term + LastLogIndex」）则投票，反之拒绝，详见 ***3.日志复制***

## 2.3 Raft 如何防止Split Vote

在一次选举期间，可能选票被平均到各个节点，使得所有Candidate角色得不到过半选票，只有等待选举超时，Candidate节点重新发起下一轮投票。

比如，在5个节点情况下，

|       | A    | B    | C    | D    | E    | 总票数 |
| ----- | ---- | ---- | ---- | ---- | ---- | ------ |
| 节点A | 投A  | 投B  | -    | -    | -    | 2      |
| 节点B | -    | 投B  | 投C  | -    | -    | 2      |
| 节点C | -    | -    | 投C  | 投D  | -    | 2      |
| 节点D | -    | -    | -    | 投D  | 投E  | 2      |
| 节点E | 投A  | -    | -    | -    | 投E  | 2      |

所有节点都没过半，均需要等待选举超时后发起下一轮选举，本轮选举就被称为Split Vote。

Raft 使用随机选举超时时间，防止一次选举中存在过多的Candidate瓜分选票，而且在Leader选出来后，Leader需要立即发送心跳事件，防止其他Candidate触发下一轮选举。<font color='red'>通常集群将这个时间定为 100ms 到 500ms</font>。

可见，心跳事件超时时间应该要求远小于选举超时时间
$$
Timeout_{心跳} << Timeout_{选举} \\
Timeout_{选举} \in [100ms, 500ms] \\
Random(Timeout_{选举}) \in [Timeout_{选举}, 2Timeout_{选举}]
$$


# 3. 日志复制

一次Client写入请求流程大致如下：（以实际实现为准）

​				![Raft-Commit1](/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Commit1.png		)				![Raft-Commit2](/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Commit2.png)

<center>图 4 - 写入请求处理流程</center>

## 3.1 日志结构

![Raft-Log](/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Log.png)

<center>图 5 - 一个可能的集群日志状态</center>

一个 LogEntry 由Term、LogIndex和数据组成，一个 Term + LogIndex 可以表示唯一一个LogEntry的位置。

节点在接收到 AppendEntries 请求后，即将数据追加到日志文件中，以防止节点宕机导致数据丢失。（其实每次fsync会对性能有影响，有合并写的优化，这个后面补充）

<font color='red'>当一个LogEntry被持久化的节点数过半时，这个时间就可以被称为”提交“</font>。

## 3.2 日志对选举的影响

* 当一个LogEntry日志被提交，未来所选举出来的Leader必定包含已提交的所有LogEntry。

证明：（反证法）

当一个LogEntry已经被集群认定为提交（客观认定），那么，这个LogEntry必定存在于过半数的节点上。

假设原集群总的节点数量为N，当发生新一轮选举时，假设选出的新Leader不包含这个LogEntry，那么这个Leader选举中就不可能获得过半选票。

比如：

![Raft-Log2](/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Log2.png)

当Leader A发生故障，B、C、D、E节点选举，因为最大已提交日志在[Term=3, LogIndex=6]，所以，节点D、E是不可能获得节点B、C的选票的。

节点C当选的可能投票情况：

|               | A    | B    | C    | D    | E    | 总票数    |
| ------------- | ---- | ---- | ---- | ---- | ---- | --------- |
| 节点A（宕机） | -    | -    | -    | -    | -    | 0         |
| 节点B         | -    | 投B  | 投C  | -    | -    | 2         |
| 节点C         | -    | -    | 投C  | -    | -    | 3（过半） |
| 节点D         | -    | -    | 投C  | 投D  | -    | 1         |
| 节点E         | -    | 投B  | -    | -    | 投E  | 1         |

节点B当选的可能投票情况：（因为[Term=3, LogIndex={7,8}]的LogEntries不算已提交，所以节点B是有资格成为Leader的）

|               | A    | B    | C    | D    | E    | 总票数    |
| ------------- | ---- | ---- | ---- | ---- | ---- | --------- |
| 节点A（宕机） | -    | -    | -    | -    | -    | 0         |
| 节点B         | -    | 投B  | 投C  | -    | -    | 3（过半） |
| 节点C         | -    | -    | 投C  | -    | -    | 2         |
| 节点D         | -    | 投B  | -    | 投D  | -    | 1         |
| 节点E         | -    | 投B  | -    | -    | 投E  | 1         |

## 3.3 日志覆盖

一种可能的集群日志状态：

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Log-Inconsistency.png" alt="Raft-Log-Inconsistency" style="zoom:50%;" />

日志不一致一般由以下原因造成：

1. 网络延时。因为Leader广播日志追加请求，只需要等待过半数的节点返回，则可以认为日志已经提交，无需等待剩余节点的返回结果，所以有部分节点可能因负载或者延时，导致日志复制延时；
2. 网络分区。当网络分区发生时，低于半数的子集群无法不可写入，则不再对外提供服务，那么就不会再产生新的”已提交“日志；多余半数的子集群则可能会重新选举，并在新的Term中继续提供服务；

* 延时导致的日志不一致：

  ![Raft-Log2](/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Log2.png)

* 网络分区导致的日志不一致：

  ![Raft-Network-Partition-Log](/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Network-Partition-Log.png)

只要日志在客观上不是”已提交“，那么在新Term下，就有可能被覆盖。

所以，在新Leader上任之后，Follower需要找到上一次无争议的日志位置，并从该位置开始复制。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/JavaAlg4th/分布式协调/pics/Raft-Follower-Replication.png" alt="Raft-Follower-Replication" style="zoom:50%;" />









