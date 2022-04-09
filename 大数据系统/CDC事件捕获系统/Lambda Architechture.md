# 1. Lambda Architecture

在大数据处理系统里面，数据的准确性和实时性是一对矛盾的目标，鱼与熊掌不可兼得。

* 针对数据的准确性，业界通常会使用批处理系统，如：Hadoop MapReduce，批处理系统要求处理的数据集是“有界”的，“不可变化”的；

* 针对数据的实时性，业界也会使用对应的流处理系统，如：Storm、Spark等，这些系统针对“无界”的数据集，进行“单行”或者“窗口化”处理；

在金融股票市场，“分分钟几百万上下”的交易平台，一般要求数据系统提供“既实时又准确”的数值，以便用户或者企业做出正确的判决。但是，实时系统意味着可以处理的信息容量是有限的，所以，就逐渐成：先提供“实时但不那么准确”的数据，而后续提供数据修正来弥补数据的“准确性”的服务（产生了LA架构）。

## 1.1 LA架构

![LA-1.png](/Users/panyongfeng/Documents/basic_framework/wiki/大数据系统/CDC事件捕获系统/pics/LA-1.png)

LA架构主要分为三个部分：

* Batch Layer
  
  一个能处理大批量数据，而且输出精确结果的系统（完整的数据视图下数据之间关联性是准确的）。系统输入、输出都是“只读”，如果需要更新数据，需要重新计算并覆盖输出结果。
  
  代表系统：Hadoop, Snowflake, Redshift, Synapse, Big Query

* Speed Layer
  
  Speed Layer不要求输出跟Batch Layer一样准确的结果，其目标是尽快输出近期数据的处理视图，等Batch Layer结果输出后，可以替换Speed Layer输出结果。Speed Layer输出结果一般存储于No-SQL类型数据库中。
  
  代表系统： [Apache Storm](https://en.wikipedia.org/wiki/Storm_(event_processor) "Storm (event processor)"), [SQLstream](https://en.wikipedia.org/wiki/Sqlstream "Sqlstream"), [Apache Samza](https://en.wikipedia.org/wiki/Apache_Samza "Apache Samza"), [Apache Spark](https://en.wikipedia.org/wiki/Apache_Spark "Apache Spark"), [Azure Stream Analytics](https://en.wikipedia.org/wiki/Azure_Stream_Analytics "Azure Stream Analytics")(微软)
  
  > :leaves: Speed Layer不准确计算，是源于Data Source在传输过程中可能出现的延时、丢失，最终造成：关系数据视图不对齐（如：由多个业务线的实体表聚合得到的事实表，或者由多个实体表或者事实表，通过一定维度，如：时间周期、数量、数据特征聚合而成的维度表）

* Serving Layer（类似于数据中台？）
  
  通过adhoc查询，分别从Batch Layer和Speed Layer中获取数据，并进行合并返回最终确认的数据给查询方。

## 1.2 LA架构工作原理

![LA-2.png](/Users/panyongfeng/Documents/basic_framework/wiki/大数据系统/CDC事件捕获系统/pics/LA-2.png)

LA工作原理很简单，就是Batch Layer处理T+N数据，Speed Layer处理最近N天的数据，Batch Layer在批处理完成时，即可替换数据视图，并丢弃Speed Layer处理结果。

> :leaves: T+N的单位根据实际场景而定，可以灵活从延时、吞吐和容错方面做出权衡。







参考文档：

1. https://www.voltdb.com/blog/2014/12/simplifying-complex-lambda-architecture/

2. 
