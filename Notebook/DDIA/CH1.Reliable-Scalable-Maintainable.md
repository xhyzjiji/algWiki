[TOC]

# 0.背景

任何一个应用程序或者服务离不开数据，我们既是服务开发者，更是一个数据系统的设计者（数据的提取、聚合、转换）。

![ArchitectureForADataSystem](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/ArchitectureForADataSystem.png)

<center>图 1 - 常见的服务架构</center>

如上图所示，一个常见的服务架构可能有：

* databases —— 数据存储和查询
* caches —— 记录昂贵操作的结果，提升读取速度
* search indexes —— 从分词、词频统计，到关键字匹配度的全文索引，如elasticsearch
* batch processing —— 周期性处理一批大量数据，如spark
* stream processing —— 流式数据处理，如flink
* message queue —— 消息、异步任务推送，如mafka
* capture data changes —— 捕获存储数据变化，数据路由，如databus

这些软件系统的设计开发都离不开三个主题：可靠性、可扩展性、可维护性。

# 1.可靠性

软件系统通常都需要满足下面的要求：

1. 输入输出符合预期，输入错误不会引起系统非预期异常；
2. 在一定范围内的请求负载或数据量下，系统性能足够地好；
3. 系统能阻挡非法访问和攻击。

当上述要求没有被正确执行时，”错误“可能正在发生。

在系统出现”错误“时，能承受的容错范围或弹性处理的能力，我们称为<font color='red'>可靠性</font>。

## 1.1 硬件故障

常见的硬件故障有：（1）电力故障（2）本地硬件故障（硬盘、内存错误、宕机）（3）网络故障（网络隔离、网络阻塞）。

常见的硬件处理手段：

1. 硬件容错：如配备UPS电源预防电力中断；配置RAID阵列硬盘做数据备份
2. 软件容错：硬件容错的辅助，如Canal Server，在主节点需要停机升级时，热备节点接管任务并继续对外提供服务

## 1.2 软件错误

> 硬件故障特征是故障率与机器数量成正比，一个时刻可能有若干个节点随机故障。
> 
> 软件故障特征是一个时刻，整个集群所有节点或大部分节点可能同时发生故障。

引发软件错误的可能性：（1）硬件故障或代码bug（2）CPU、内存、磁盘空间、网络带宽的资源耗尽（3）服务响应慢或返回错误内容（4）系统级联故障

软件错误预防手段：（1）单元测试和压力测试（2）进程隔离或服务解耦（3）监控和分析系统指标判断健康程度。

+++

:zap: 忽视软件错误，可能最终引发多个服务的雪崩效应，如下图

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/xuebeng1-1.jpg" alt="xuebeng1-1 z" style="zoom:67%;" /><img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/xuebeng1-2.jpg" alt="xuebeng1-1 z" style="zoom:67%;" /><img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/xuebeng 1-3.jpg" alt="xuebeng 1-3 z" style="zoom:67%;" /><img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/xuebeng1-4.jpg" alt="xuebeng1-4 z" style="zoom:67%;" />

应对雪崩的手段：

* 硬件故障：多机房容灾、异地多活
* 缓存穿透：多级缓存，缓存本地预加载、缓存异步加载
* 流量激增：服务弹性扩容、流量控制（限流、熔断[打开/半开/关闭]、关闭重试）
* 同步等待：资源隔离、MQ解耦、不可用服务调用快速失败等。资源隔离通常指不同服务调用采用不同的线程池；不可用服务调用快速失败一般通过熔断器模式结合超时机制实现。

+++

## 1.3 人为操作失误

降低人为失误的方法：

* 运维接口要尽可能的简单和抽象，运维参数和动作应该有一定的约束条件
* 不影响线上真实服务下，提供沙盒环境进行实验
* 从单元测试向系统自动化集成测试发展
* 提供灰度系统、支持运维操作回滚能力，以及出错后的补偿工具
* 监控服务诊断服务异常状态

# 2.可扩展性

可扩展性用于描述系统应对上涨负载的能力。当系统面临负载溢出边缘，可扩展性限制了我们在保持系统可靠性的可用策略。

## 2.1 负载表现

以tweeter用户发布动态为例

发布动态：用户发布动态请求为平均4.6K/s，峰值是12K/s

浏览动态：用户刷新动态平均请求为300K/s

---

tweet-Ver1.0：仅维护发布者动态时间线。发布动态直接记录“发布者”--“粉丝”--“tweet文档”关系

![tweet1](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/tweet1.png)

在“发布者”发布动态的时候，仅需要在tweet文档记录一条数据即可，而用户刷新动态时，则需要根据三表关联关系，查询出订阅对象的tweet文档，并以时间进行排序。写入速度非常快，但很快由于读流量是写流量的2个数量级，所以读取动态立即成为系统第一负载。

---

tweet-Ver2.0：为每个用户维护动态时间线。发布动态，发送到“粉丝”邮箱，“粉丝”各自查看自己的邮箱信息即可。

![tweet2](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/tweet2.png)

用户上线后仅读取属于待收的动态信息即可，问题是对于一些“明星用户“，由于”粉丝“数量庞大，导致写放大，系统扇出成为新的瓶颈。

---

tweet-Ver3.0：分用户类型维护动态时间线，推拉结合。”普通用户“的动态发布直接推送，”明星用户“通过流式推送和”粉丝“读（上线）时拉取相结合。

这里有更详细的演进过程：https://www.infoq.com/presentations/Twitter-Timeline-Scalability/

系统的负载的表象随着系统架构的变化而不同。

## 2.2 性能表现

一般性能的评估方法：

1. 系统资源（CPU核数、内存、网络带宽等）不变情况下，负载增大时性能的变化；
2. 要求系统性能（服务响应时间、批量处理的吞吐量等）不变的情况下，负载增大和资源增加的关系。

系统支持可扩展性要求：

1. 服务无状态
2. 服务架构需要提前对高频次、消耗资源的操作做出假设，并做出应对策略。

# 3.可运维性

运维操作设计原则：

1. 运维操作简单，可以平滑执行
2. 运维要直观，容易理解，尽可能避免有严格时序要求或非线性操作
3. 运维接口的可扩展性

可运维性的宗旨是：Making Life Easier.

系统设计者需要做的准备：

* 可视化的系统内部视图，如系统监控指标、服务关联视图、数据流向等
* 提供集成的标准化工具
* 操作有明确的SOP
* 运维自动修复（如：巡检）
