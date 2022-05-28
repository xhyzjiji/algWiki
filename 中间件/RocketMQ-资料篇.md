RocketMQ内部队列一览

```java
public static final String RMQ_SYS_SCHEDULE_TOPIC = "SCHEDULE_TOPIC_XXXX";
public static final String RMQ_SYS_BENCHMARK_TOPIC = "BenchmarkTest";
public static final String RMQ_SYS_TRANS_HALF_TOPIC = "RMQ_SYS_TRANS_HALF_TOPIC";
public static final String RMQ_SYS_TRACE_TOPIC = "RMQ_SYS_TRACE_TOPIC";
public static final String RMQ_SYS_TRANS_OP_HALF_TOPIC = "RMQ_SYS_TRANS_OP_HALF_TOPIC";
public static final String RMQ_SYS_TRANS_CHECK_MAX_TIME_TOPIC = "TRANS_CHECK_MAX_TIME_TOPIC";
public static final String RMQ_SYS_SELF_TEST_TOPIC = "SELF_TEST_TOPIC";
public static final String RMQ_SYS_OFFSET_MOVED_EVENT = "OFFSET_MOVED_EVENT";
```



事务发送，采用2PC协议

2PC协调者是producer，参与者是producer本地事务和broker事务消息，broker半消息会先进入内部队列<font  color='red'>RMQ_SYS_TRANS_HALF_TOPIC</font>，在2PC的commit阶段，将内部队列的事务消息写入业务topic。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/中间件/pics/RocketMQ_Transaction.png" alt="RocketMQ_Transaction" style="zoom:50%;" />

broker处理半消息的逻辑，会在消息中记录真实的topic和queueId，然后再写入事务队列，详细过程见：

org.apache.rocketmq.broker.processor.SendMessageProcessor#sendMessage：这是接收到事务发送请求后broker的处理逻辑

```java
// org.apache.rocketmq.broker.transaction.queue.TransactionalMessageServiceImpl#parseHalfMessageInner
private MessageExtBrokerInner parseHalfMessageInner(MessageExtBrokerInner msgInner) {
  MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_TOPIC, msgInner.getTopic());
  MessageAccessor.putProperty(msgInner, MessageConst.PROPERTY_REAL_QUEUE_ID,
                              String.valueOf(msgInner.getQueueId()));
  msgInner.setSysFlag(
    MessageSysFlag.resetTransactionValue(msgInner.getSysFlag(), MessageSysFlag.TRANSACTION_NOT_TYPE));
  msgInner.setTopic(TransactionalMessageUtil.buildHalfTopic());
  msgInner.setQueueId(0);
  msgInner.setPropertiesString(MessageDecoder.messageProperties2String(msgInner.getProperties()));
  return msgInner;
}
```

如果步骤3的commit/rollback丢失，Broker会定时轮询Producer事务结果，当轮询次数达到transactionCheckMax（默认15次）时，就会按rollback处理消息。

> :question: 如果多个进程同时发送半消息，消息按顺序持久化到RMQ_SYS_TRANS_HALF_TOPIC队列，那怎么对某个消息进行commit？
>
> 比如半消息队列状态：HalfMsg1、HalfMsg2、HalfMsg3，先对HalfMsg2进行commit，消息是如何出队的？
>
> ```java
> // org.apache.rocketmq.broker.transaction.queue.TransactionalMessageServiceImpl#commitMessage
> public OperationResult commitMessage(EndTransactionRequestHeader requestHeader) {
> 	return getHalfMessageByOffset(requestHeader.getCommitLogOffset());
> }
> ```
>
> 可以看出是通过offset直接取出数据（怎么更新消息状态？）



源码阅读心得：

1. 消息驱动
   broker server的网络处理器：org.apache.rocketmq.broker.processor
   producer的网络处理器：
   consumer的网络处理器：

2. 数据存储和读取

   * 元数据的存取

   MQ一般的元数据有哪些？
   1.资源分配关系：
   分区数量、分区与consumer的绑定关系、ConsumerGroup内consumer负载均衡、broker副本主从和路由关系
   2.consumer消费位点：
   consumer offset相关数据

   相关的代码路径：

   ```
   org.apache.rocketmq.namesrv.routeinfo.RouteInfoManager：Topic、Consumer路由信息
   
   org.apache.rocketmq.client.consumer.store.RemoteBrokerOffsetStore：consumer消费offset存储
   ```

   * 消息的存取

   ```
   org.apache.rocketmq.store.MappedFile：MMAP映射文件，不管是数据还是索引，都是可选FileChannel（写入pageCache）或者MappedByteBuffer（写入PageCache）方式写入文件
   
   org.apache.rocketmq.store.CommitLog：所有topic的数据日志，每个Msg都有topic和queueId；RocketMQ是如何避免pageCache被频繁淘汰的?
   参考：https://www.jianshu.com/p/62f2c9514e2d
   org.apache.rocketmq.store.ConsumeQueue：单个topic的单个分区的数据索引，数据在commitLog中的文件号和偏移量，在写入commitLog后由CommitLogDispatcherBuildConsumeQueue分发到对应topic的consumeQueue索引中
   ```

   > 个MQ对PageCache的使用方式：
   >
   > <img src="/Users/panyongfeng/Documents/basic_framework/wiki/中间件/pics/MQ_PageCache.png" alt="MQ_PageCache" style="zoom:50%;" />
   >
   > 不同写入字节数下的性能区别：https://www.jianshu.com/p/d0b4ac90dbcb
   >
   > 总体来说：
   >
   > （1）每次写入量大用FileChannel，量小用MMAP更佳。
   >
   > （2）如果你需要经常执行 force，即使是异步的，也请一定不要使用 mmap，请使用 FileChannel。
   >
   > <img src="/Users/panyongfeng/Documents/basic_framework/wiki/中间件/pics/MQ_FileChannel_VS_MMAP.png" alt="MQ_FileChannel_VS_MMAP" style="zoom:50%;" />
   >
   > <font color='red'>补充一个benchmark实现</font>

3. 故障处理和恢复

   

