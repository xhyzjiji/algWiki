1. Flink的快照保存机制和Exactly-Once思想

   首先科普下Flink的概念：
   Flink作业可以抽象成有向图（DAG）表示，图的顶点是算子（operator），边是数据流（data stream）。
   Flink可以分为3种算子：Source、Transformation、Sink，其中Transformation属于计算节点，分有状态和无状态两种算子。

   >  快照保存机制要求：
   >
   > 1. 所有算子产生的快照必须保证一致性
   > 2. 快照过程不能影响系统正常运行，更不能stop the world
   >
   > Exactly-Once新思路：
   >
   > 当系统链路上发生故障，只需要保证系统从某一个快照状态后，继续消费数据，来达到数据精确一次消费的目的
   >
   > 当然，这需要系统具有“幂等”或者“回滚”的特性。

   算子的状态受入边输入的数据影响而发生改变，出边不影响算子状态。

   快照算法：Chandy-Lamport和Asynchronous Barrier Snapshotting

   1. Chandy-Lamport 算法（源自论文：Chandy Lamport-Determining Global States of Distributed Systems）

      论文中将使分布式系统状态发生变化的因素叫做事件（event），并给出了它的形式化定义，即e=<p, s, s', M, c>。p是进程、M是进程p对外发送的消息M、c是两个进程间通信的边，而s和s'分别是进程p在事件e发生之前及发生之后的状态。

      ```
      // 进程p行为，通过向q发出Marker，发起snapshot
      begin
             p record its state；
      then
             send one Marker along c after p records its state and before p sends further messages along c
      end
      
      //进程q接受Marker后的行为，q记录自身状态，并记录通道c的状态
      if q has not recorded its state then
              begin
                    q records its state;
                    q records the state c as the empty sequence
              end
      else q records the state of c as the sequence of messages received along c after q’s state was recorded and before q received the marker along c. 
      ```

      **算法过程：**

      进程p启动这个算法，记录自身状态，并发出Marker。随着Marker不断的沿着分布式系统的相连通道逐渐传输到所有的进程，所有的进程都会执行算法以记录自身状态和入射通道的状态，待到所有进程执行完该算法，一个分布式Snapshot就完成了记录。Marker相当于是一个信使，它随着消息流流经所有的进程，通知每个进程记录自身状态。且Marker对整个分布式系统的计算过程没有任何影响。只要保证Marker能在有限时间内通过通道传输到进程，每个进程能够在有限时间内完成自身状态的记录，这个算法就能在有限的时间内执行完成。

   2. Asynchronous Barrier Snapshotting算法

      Asynchronous Barrier Snapshotting算法在Chandy-Lamport算法基础上进行实现，Barrier则是上述的Marker。JobManager会定期从Source中放入Barrier，Barrier沿着算子下发。

      **算法过程：**

      1. Barrier周期性的被注入到所有的Source中，Source节点看到Barrier后，会立即记录自己的状态，然后将Barrier发送到Transformation Operator。

      2. 当Transformation Operator从某个input channel收到Barrier后，它会立刻Block住这条通道，<font color='red'>直到所有的input channel都收到Barrier</font>（这一过程叫对齐），此时该Operator就会记录自身状态，并向自己的所有output channel广播Barrier。

         ![Flink_ABS](/Users/panyongfeng/Documents/basic_framework/wiki/中间件/pics/Flink_ABS.png)

      3. Sink接受Barrier的操作流程与Transformation Oper一样。当所有的Barrier都到达Sink之后，并且所有的Sink也完成了Checkpoint，这一轮Snapshot就完成了。

      > 注意：
      >
      > 1. 如果一个算子有多个入边，如果有入边的Barrier长时间未触达算子，可能会阻塞流的正常吞吐，Flink提供的解决方案是：（1）输入缓冲（2）可选的Block模式，如果选择不阻塞，则在第一个Barrier到达时立即保存快照，这个时候语义从Exactly-Once降级为At-Least-Once
      >
      > 2. Asynchronous Barrier Snapshotting，除了Barrier，还有一个是Asynchronous，即快照保存期间，数据可以继续处理，所以这时候，即使所有Barrier都到达Sink，不代表所有的快照已经产生，还需要所有算子对快照动作做出响应才行。
      >
      > 3. 一旦有算子在完成快照前发生故障，表示本轮快照没有完成，但Flink不会立即报错，如果快照故障持续发送，Flink才会中止作业，这也是在不影响实时处理与故障恢复影响间的一个折衷。
      >
      > 4. 对于DAG图，快照算法比较容易理解，不过Asynchronous Barrier Snapshotting同样适用于DCG图（有环）。
      >
      >    ![Flink_ABS_DCG](/Users/panyongfeng/Documents/basic_framework/wiki/中间件/pics/Flink_ABS_DCG.png)
      >
      >    **算法过程：**
      >
      >    1. 当第一个Barrier到达后，block住这个入边通道，直到所有入边（不包含回环的入边）的Barrier到达后，保持快照、向下一个算子输出Barrier，并继续处理通道数据。
      >
      >    2. 在保存快照到回环入边的Barrier到达，这段期间，需要保存回环入边的所有输入数据（各个回环入边单独存储），并在恢复时，重放这些数据进入算子参与计算。
      >
      >       <img src="/Users/panyongfeng/Documents/basic_framework/wiki/中间件/pics/Flink_ABS_DCG_Example.png" alt="Flink_ABS_DCG_Example" style="zoom:50%;" />

2. WaterMark和窗口生命周期

3. Trigger触发器和数据输出时间

4. 