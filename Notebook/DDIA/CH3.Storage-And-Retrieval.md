# 1.数据存储结构

总的来说，数据库只需要做两件事情，当你给他数据的时候，他要把数据存起来，当你随后找他要的时候，它还能正确的把数据吐给你。

本章主要介绍两种存储引擎（基于日志log-structured的存储引擎和面向页page-oriented的存储引擎）的数据存储和查询。

## 1.1 哈希索引

* 存储方式

insert/update：假设我们有一个最简单的数据库，每当有写请求的时候，就把他<font color='red'>顺序追加</font>到一个文件的末尾。

delete：当需要删除行数据时，则需要写入一条特殊的删除记录，并在下次compaction时将数据空间和索引释放。

* 数据索引

为了方便查询，我们给这个数据库建立一个内存KV索引，Key是行主键，而Value指向对应文件中的这个主键的最新数据的文件偏移地址。

![hash-index](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/hash-index.png)

受磁盘空间限制，文件大小无法无限扩充，所以当写入超过一定大小后，对文件进行分段，将新数据写入新的文件，然后对旧文件的重复Key的行进行压缩，不允许再有任何写入。

压缩过程中，读请求依然可以使用未压缩前的段文件，当压缩完成后再修改索引文件引用新的段文件偏移量。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/hash-index-multi-compaction.png" alt="hash-index-multi-compaction" style="zoom:80%;" />

哈希索引的一些工程实现优化：

1. 宕机恢复

   哈希索引尽管在服务重启后重建，但重建时间与数据量成正比，所以当分段文件压缩形成新的文件时（文件不会变化），同时针对这个文件生成一个哈希索引快照，那么哈希索引重建时，只需要对当前可写文件重建索引外，其他文件索引直接加载即可。

2. 并发控制

   顺序写文件需要严格顺序，所以使用主节点单线程是一个常规实现。而读取的时候因为文件是顺序追加或者不可变的，所以并发读没有冲突问题。

哈希索引的局限性：

* 面对磁盘数据量，全内存hash索引可能引起内存溢出；而且很难做到存储层的hash索引，因为随机读写将成为性能瓶颈。
* 当需要范围查询时，只能逐个值扫描文件内容，效率极低。

## 1.2 SSTables和LSM-Tree

* 存储方式

SSTable要求

1. 每一个段文件内，以Key-Value保存记录，而且一个文件内，不会出现重名Key记录；

2. 每一个段文件内，保存的是以Key排序后的数据

   对于超过一定文件大小的段文件，服务不会再写入新数据，此时通过文件压缩合并时，对数据进行排序落盘形成新文件即可；

   对于当前可写的段文件，直接在内存操作（memTable），就不会有随机写文件的性能问题（故障保护，需要增加WAL保证数据完整性）

* 索引方式

由于文件内Key已经按序存储，所以只需要知道文件的最小Key和最大Key，通过一个稀疏索引（文件中的Key-Value不定长，无法直接使用二分法）即可快速查找。

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/SSTable-sparse-index.png" alt="SSTable-sparse-index z" style="zoom:80%;" />

由于数据文件产生后不可修改，所以对于一个文件的稀疏索引可以直接存储于文件之中：

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/SSTable-file-format.png" alt="SSTable-file-format" style="zoom:70%;" />

同时，一个Key可能同时存在于不同的段文件内，但文件越新鲜代表Key也越新鲜，所以从最新文件扫描Key就能得到最新的数据。

### 1.2.1 由SSTable实现LSM树

LSM树在SSTable基础上，以树状结果管理多个层级的SSTable：

1. 当memTable大小到达阈值，会转换成immutable in mem，然后异步刷入Level 0的SStable中

2. 当Level 0的SSTable数量达到合并条件，则从层级最后一个SSTable与Level 1中<font color='red'>有重叠的Key</font>的若干个SSTable进行合并，合并后如果Level 1也达到合并要求，则继续与下一个层级的SSTable合并，如此类推；

   这样保证了只有Level 0上的SSTable不同文件中存在重复的Key，而Level X(X>0)的SSTable，Key不再重复；Key的范围查询效率和穿透查询可以快速停止，而无需扫描所有文件。

3. 由于memTable是内存数据，依旧需要redolog保证宕机恢复后数据完整性。

### 1.2.2 工程调优

1. 查询Key是否存在时，先使用Bloom Filter，只要Filter返回不存在，那表示Key一定不存在，减少了文件扫描的次数；
2. 文件合并压缩机制优化：https://blog.csdn.net/zhangchunminggucas/article/details/7626514

## 1.3 B-Trees

之前的基于日志索引把数据库拆分为若干个段数据块(*segment*)，每段数据块顺序写，大小大约是几M或更大，但是B树却不同，他把数据库打散成一个个小块(*block*/*page*)，每个小块一般是4K，一次读写一个小块。这种设计更贴近底层的硬件结构，因为物理磁盘实际上也是按固定大小的扇区为最小单元进行读写，而页大小则是扇区大小的倍数关系。

每页数据有一个唯一的地址，这样数据块就可以有一个像指针一样的东西，指向另一页数据，只不过这个指针是在磁盘上。我们除了可以通过这些页的引用构建一棵关于页的树，还可以在<font color='red'>同一高度的页之间增加引用，形成类似双向链表的关系</font>，数据相邻页在磁盘上就不需要严格相邻。

![B-Tree-Index](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/B-Tree-Index.png)

<center>树结点和数据页的引用</center>

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/B-Tree-Index2.png" alt="B-Tree-Index2" style="zoom:50%;" />

<center>页之间的引用形成链表结构</center>

* 存储和索引

insert/delete：insert与delete类似，在树中新增或摘除相应结点，释放空间，如果页面大小变化足以触发页的分裂和收缩，则在后台异步进行，不过delete操作比insert稍复杂一些，因为delete还需要处理删除结点下的孩子结点的归宿问题。

update：通过树找到指定Key所在的页，加载到内存，然后更新指定字段的数据（如果更新后页大小超过阈值，则触发页分裂，反正，如果更新后存在空页，则触发页的合并）

不管是页刷盘、页的分裂合并还是树引用的变化，为了保证故障恢复时的数据一致性，存储引擎需要增加WAL日志，记录物理页的相应操作和doublewrite保证页数据的完整性。

---

:seedling:一个可变长度的字段，在update的时候所需的存储长度变了，数据页应该如何变化？

<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/page-split.jpg" alt="page-split" style="zoom:15%;" />

参考文档：

https://www.percona.com/blog/2017/04/10/innodb-page-merging-and-page-splitting/

译文：https://blog.2014bduck.com/archives/260

---

:seedling:离散写加速页的分裂合并，对性能的影响测试：https://mp.weixin.qq.com/s/vAoA1T6MFJDQK-YcdZf6hw

---

:seedling: 一颗高度为4，页大小为4KB，结点引用孩子页数量因子为500，理论上可以支撑256TB的数据。
$$
TotalDataSize = branchingFactor^{treeHeight} * pageSize = 500^4 * 4KB = 256TB
$$
<img src="/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/tree-datasize-map.jpg" alt="tree-datasize-map" style="zoom:15%;" />

---

## 1.3 LSM-Tree和BTree的比较

1. LSM由于重复执行SSTable的压缩合并，会造成数据的写放大（原文提到SSD会降低写放大，不是很理解，BTree的页分裂合并的时候不是也有写放大吗？），不过由于SSTable文件顺序写的特性，理论上LSMTree比BTree拥有更高的写吞吐；
2. LSM-Tree比BTree节省存储空间，因为周期的压缩，可以降低数据页的碎片化；
3. LSM-Tree在文件压缩期间可能影响读写吞吐量，磁盘的读写带宽被压缩操作所瓜分；
4. 如果LSM-Tree的压缩速率比不上写入速率，那么segment文件就会堆积，继而严重影响读效率；
5. BTree读更快，因为不像LSMTree那样，需要遍历Level 0的SSTable确保Key的存在性（尽管BloomFilter已经减少了大部分查询）

## 1.4 其他的索引结构

* 二次索引
* 联合索引
* 全文索引

:joy:内容太多了，后面慢慢补



# 2. OLTP和OLAP

| 特点         | OLTP                         | OLAP                                              |
| ------------ | ---------------------------- | ------------------------------------------------- |
| 读           | 每次根据key查很少数据        | <font color='red'>查询大量数据，算统计数字</font> |
| 写           | 基于用户输入，随机写，低延迟 | 批量导入，或者基于时间流                          |
| 用途         | 通过应用服务用户             | 给内部决策提供分析                                |
| 数据呈现方式 | 数据的最终状态               | <font color='red'>数据的全历史状态</font>         |
| 数据规模     | GB 到TB                      | <font color='red'>TB 到PB</font>                  |

下图是一个零售商的数据仓库。仓库的核心是一张事实表(*Fact Table*)，例子中对应fact_salesb表，每行记录了一个事件，这个例子中每行表示一个顾客在特定时间买了某一样商品。一般来说事实表是每一个事件一行记录，这样可以给分析员最大的灵活性。但这就意味着事实表无比巨大，像沃尔玛，苹果他们都有几十PB的数据仓库，绝大部分都是这类事件数据。

![Start-Schema](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/Start-Schema.png)

<center>表间的星状拓扑结构</center>

如果你的数据仓库有超过PB级的数据以及有超过几千亿行记录，那如何高效的存储和查询对你来说就是一个技术活了。属性表相对来说就要小很多了，往往也就是百万级。所以我们主要把精力集中在如何处理事实表上。
尽管事实表往往有超过100列，但是一般来说一次查询也就是其中的4、5列，像SELECT * 这种查询分析员一般很少用，因此间接催生了面向列的存储方式(Column-Oriented Storage)。

# 3. 列存储

在绝大多数OLTP系统中，数据是以行的方式存储的。一条数据的所有字段在存储的时候也是彼此相邻的。当你要查询的时候，即使你有索引，你也需要把整行数据从内存或者磁盘当中读出来，解析过滤，这个过程相当耗时间。而且对于分析员来说，往往要查的数据要么甚至没索引。面向列的存储思路与之相反，不是把表中每行数据存在一起，而是让每一列的数据存在一起。如果每列数据存成一个文件，那只需要查对应列的文件就很快了。

![row-to-column-storage](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/row-to-column-storage.png)

<center>行存储到列存储视图</center>

## 3.1 列存储优化

### 3.1.1 压缩

简单来说，可能每个列值的取值范围要比行数小得多得多，比如一个超市可能有几十亿的销售记录，但只有10万的商品。所以我们可以把n种商品用n个比特位来表示。第i种商品的表示方法就是第i个比特为1，其余为0。如果n很小，比如全世界国家只有不到200个，那可以简单按照上面的存储方法每位都存进去，但是如果n很大，就会发现有很多0，导致很稀疏。那就可以再给bitmap做一个*run-length*编码，就像下图一样，比如29就可以用(9,1)表示前面有9个0，然后1个1，后面全是0。这样压缩后就很小了。

另外就是bitmap索引机制也能够很好的应对数据仓库当中的查询操作，举个例子
比如查询WHERE product_sk IN (30, 68, 69), 就可以找到3个数字对应的bit位，然后将每条product_sk按位或，如果不为0，就表示在这个范围内，然后定位到拥有这些product_sk的行。

![Bitmap-Index](/Users/panyongfeng/Documents/basic_framework/wiki/Notebook/DDIA/pics/Bitmap-Index.png)

### 3.1.2 字段排序

其实一般来说不对数据的顺序做要求，最简单的方法就是数据按照插入顺序排序。这样的话插入对于每个列文件来说都是追加操作，比较快。如果有必要其实我们也可以引入一个排序的策略，像之前讲的SSTable一样，用他作为索引。但是有一点需要注意，我们不能每个列都排一个序，因为这样我们就没法知道哪些数据属于同一行了。所以我们只能用一列来对一行数据的所有列排序。选哪一列就需要管理员根据查询的特征来看了，如果查询一般都带有时间特征，比如查上个月的数据，那日期就应该用来排序。这样就只需要在排序后的数据中扫描对应的一段数据就可以了，而不用去扫全表。当第一列相同的时候，我们可以再选一个列作为*第二排序字段*，以此类推。

排序的一个好处是数据当中会有大量的重复，这对压缩特别有用，可能会有连续很多数字一模一样，这样用前面讲到的*run-length*编码就可以用几k的大小表示甚至十亿行的数据。（假设这10亿行的日期都是20180209， 那用20180209，10亿就表示完了，又或者是前缀匹配来压缩）。但是这种好处只有在第一个用于排序的字段对应的文件上特别有效，*第二排序字段*的数据相对来说就会比较跳跃，效果就没有第一个字段好。但是就算这样收益也已经足够大了。

如果需要同时对多个字段进行排序，那么可能需要建立类似辅维度的冗余数据来支持。

## 3.2 列存储的实现

列存储针对OLAP场景优化了读性能，但是列存储的写入就更为困难（不同列写不同的文件）。

上面提到的LSM-Tree反而比较适合列存储的写入需求。





