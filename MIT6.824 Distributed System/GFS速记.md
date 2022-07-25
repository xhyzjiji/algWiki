GFS速记

1. 分布式系统的挑战
   
   |     | 需求               | 具体目标            | 实现思路                                    | 引入问题                  |
   | --- | ---------------- | --------------- | --------------------------------------- | --------------------- |
   | 1   | Performance      | 读写流量负载均衡，容量水平扩展 | 单点转多点，多节点提供读写服务，数据分片Sharding            | 节点数量与服务发生 故障概率成正比     |
   | 2   | Faults Tolerance | 数据容灾，服务迁移       | 数据复制（副本容灾）Replication，分布式协调算法实现自动故障自动恢复 | 数据一致性问题               |
   | 3   | Consistency      | 副本数据一致性         | 分布式一致性算法                                | 降低性能， Low Performance |

2. GFS应用场景
   
   | 特性               | 表现                                                    | 注意项                                                  |
   | ---------------- | ----------------------------------------------------- | ---------------------------------------------------- |
   | Performance      | 大型数据（TB级别数据）的顺序读写（而非随机访问），分chunk存储，chunk在MB级别以降低元数据数量 | 小文件会占用一个chunk导致存储空间浪费和元数据激增                          |
   | Faults Tolerance | 自动故障恢复                                                |                                                      |
   | Consistency      | 弱一致性（具体与复制协议、客户端实现相关）                                 | client会尝试从相同rack或者idc甚至region上的数据副本中读取数据（降低网络IO的RTT） |

3. GFS的部署和特点
   
   1. 针对单数据中心（而非跨中心构建集群）
   
   2. 文件appendable，一旦一个chunk填满就不可变

4. GFS的读写流程
   
   1. 写流程（primary相当于2PC协调者，所有chunk server都是参与者，但是又不是2PC，因为这里没有回滚，只有做了还是没做之分）
      
      * client 向 primary chunk server 请求文件 append 的 offset
        
        :question: primary是否需要将offset持久化，否则primary故障部分从副本已经占用offset这块数据？(我觉得理论上是需要的，所以对于小量写多频繁更新offset的场景，对性能有一定影响)
      
      * client 广播数据到所有 chunk server
        
        :leaves: 工程优化上，client仅就近发送数据到一个chunk server，并由chunk server进行链式复制(chain replication)，当链尾的chunk server写入完成，所有chunk server就都已经写入完成了。
        
        > chain replication有不同的一致性：
        > 
        > 1. 如果client可以任意读取一个节点的数据，可能出现不可重复读，第一次读到数据，然后第二次可能读不到，只能是最终一致（类似主从异步复制）；
        > 
        > 2. 如果client只读链尾节点，那就是线性读；
        > 
        > 如果大量线性读，势必对链尾节点造成负载（单点瓶颈）
        > 
        > 变种chain replication：Chain Replication with Apportioned Queries
        > 
        > [CRAQ 论文笔记 - 楷哥 - 博客园](https://www.cnblogs.com/zzk0/p/13504600.html)
        > 
        > 原理类似etcd，链表内节点维护数据版本号，client请求先从链头节点获取最新版本号，然后随机请求节点，并等待节点直到数据更新为知道版本。
      
      * 一旦从 chunk server 完成数据传输，会向 primary chunk server 响应（类似2PC） 
      
      * primary 收到所有从 chunk server 的ack后，返回client写入成功，超时或者发现错误都会返回写入失败
      
      > 故障场景分析：
      > 
      > 1. client写入，部分从chunk server写入失败
      > 
      > 2. client写入，primary分配offset后故障
      > 
      > 3. client写入，primary分配offset前故障
      > 
      > 4. client请求过期primary
      >    
      >    client会从master获取最新的primary，但master回应后但消息到达client前，可能原primary出现故障，导致client请求旧primary（client缓存chunk server拓扑结构也会有这种问题）
      >    
      >    此时primary可能与master网络隔离，但与client可以通信，所以当master无法通过与primary以心跳维持通信，需要先等待lease过期后，才能分配新的primary（GFS默认60s）
   
   2. 读流程

5. 回顾GFS的一致性
   
   即使client写入成功，可能存在数据不一致的情况，具体以GFS一致性模型来分析。



GFS核心问题

1. Lease租约机制如何防止脑裂
   
   > Even if the master loses communication with a primary, it can safely grant a new lease to another replica after the old lease expires.
   > 
   > Lease设计出来的目的，是为了降低master的请求负载（防止中心化负载瓶颈）
   
   * 对client颁布lease
     
     client向master请求chunk server list和主从信息时，master向client颁布lease，表示，在lease期间，master承诺不会修改应答client的元信息，这时候client可以安全的缓存这些元信息。
     
     > :question: 那么如果master需要紧急修改元信息，只能等待最后一个承诺client的lease过期，将会大大降低系统的响应速度。
     > 
     > 工程优化：在master修改元信息前，可以立即告知client，让client快速失效本地cache的数据（会增加系统设计的复杂度）
   
   * 对chunk server颁布lease
     
     master颁布lease给一个chunk server节点，在lease期间，权力下放到该节点，以释放master算力。client可以直接访问primary节点，在record append操作下，primary负责计算offset和保证所有secondary节点的按固定顺序持久化数据。
   
   Lease租约机制根据时间确定是否过期，这就要求Lease的发布和持有者时钟同步。最常见的是使用NTP协议对时钟进行同步。
   
   > 场景分析
   > 
   > ----
   > 
   > 1. Lease发布者本地时钟比持有者快
   >    
   >    根据时钟同步误差，Lease持有者的租约有效时间比发布者短（让Lease持有者更早失效）
   > 
   > 2. Lease发布者本地时钟比持有者慢
   >    
   >    Lease持有者会过早失效
   >    
   >    （1）针对client的lease，会促使client重新请求master新的chunk server元数据，如果master在更新元数据，则仅发送最新数据而不提供lease承诺
   >    
   >    （2）针对chunk server的lease，自主失效primary权力，client请求时则返回client自己已经不是primary，client重新请求master并选出新primary
   > 
   > 工程优化：
   > 
   > 除了依赖NAT时钟同步外，Lease发布者和持有者通信过程，可以根据统计的网络RT，交互信息带有双方时钟，来做时钟的调整。（其实NTP也是这么做）

2. GFS的一致性模型
   
   ![GFS一致性定义.png](/Users/bytedance/Documents/bytedance_framework/algWiki/MIT6.824%20Distributed%20System/pics/GFS一致性定义.png)
   
   (1)写操作
   
   * write操作
     
     > A write causes data to be written at an application-specified file offset.
   
   * record append操作
     
     > A record append causes data (the “record”) to be appended atomically at least once even in the presence of concurrent mutations, but at an offset of GFS’s choosing
   
   (2)副本数据差异
   
   * consistent
     
     > A file region is consistent if all clients will always see the same data, regardless of which replicas they read from.
   
   * defined
     
     > A region is defined after a file data mutation if it is consistent and clients will see what the mutation writes in its entirety.
     
     defined定义除了包含consistent定义外，还要求client可以读出写入数据的完整性。
   
   (3)场景
   
   * consistent but undefined
     
     > 在并发write的时候（API要求client指定offset和写入的数据），会有一下场景引发consistent but undefined
     > 
     > 1. 跨chunk并发写，均成功
     >    
     >    比如：两个client同时要在62M的某个chunk写入4M数据（API自动计算得到）
     >    RequestA：chunk-1,62M写A1(2M), chunk-2,0M写A2(2M)
     >    RequestB：chunk-1,62M写B1(2M), chunk-2,0M写B2(2M)
     >    每个请求无法保证Primary执行顺序，所以可能出现：
     >    A1,B1,B2,A2，最终得到的数据是chunk-1,62M是B1, chunk-2,0M是A2，这种混合的不完整数据
     > 
     > 2. 跨chunk串行写，部分成功，部分失败
     >    
     >    情况类似场景1，在跨chunk拆分写请求时，有其中一个chunk写入失败，client再次读取，只能读到上次写入的部分数据。
   
   * defined interspersed with inconsistent
     
     > defined既然包含consistent，为什么会有defined interspersed with inconsistent呢？
     > 
     > -------------
     > 
     > Record append allows multiple clients to append data to the same file concurrently while guaranteeing the atomicity of each individual client’s append.
     > 
     > File namespace mutations (e.g., file creation) are atomic.
     > 
     > 首先，append操作是原子的，create chunk也是原子的。
     > 
     > 先看下面的场景：
     > 
     > 如果client要在现有的chunk写入数据，并且这个chunk剩余容量不足容纳写入的数据长度，那么这个操作就要拆分成两次写请求执行。
     > 
     > 对于record append写操作（数据长度<=16MB），GFS可以根据请求控制新增数据要写入的offset，并且写入操作在一个chunk上进行。原理很简单，当目前文件最后一个chunk容量小于要写入的数据，就创建一个新的chunk来装填数据，老chunk多余的空间可以填充一些空洞数据。
     > 
     > ----
     > 
     > 因为record append不跨chunk写，所以是原子的，要么写入成功要么失败，数据是defined的。
     > 
     > 但是，在写入失败的时候，client会重试写入数据（GFS会分配一个新的offset），然后写入成功，但文件里有一段数据在各个节点并非一致（失败节点可能是损坏的数据片段），所以就有了 defined interspersed with inconsistent 这种状态。
     > 
     > primary:             D1
     > secondary-1:     D1
     > secondary-2:     error
     > 
     > client重试
     > 
     > primary:             D1，    D1
     > secondary-1:     D1，    D1
     > secondary-2:     \*，       D1
     > 
     > ------
     > 
     > client在读取时，可能会遇到像secondary-1的重复数据，或者secondary-2的一段错误数据，所以应用层需要考虑处理这两种情况：
     > 
     > secondary-1的重复数据：数据段增加uid，对数据进行去重处理
     > 
     > secondary-2的错误数据：数据增加checksum，处理前先校验数据正确性
   
   * inconsistent
