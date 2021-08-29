# 1. 一致性背景

分布式存储系统中，不管是文件系统还是数据库，只要数据存在多个副本，都涉及一致性问题其中一致性包括内部一致性和副本一致性，内部一致性即单机版数据库中的数据满足一定的约束条件。副本一致性表示同一数据的多个副本的值相同。GFS作为一种分布式文件系统，采用了多副本机制，自然也会有一致性问题。



# 2. 影响一致性的操作

修改元数据
写数据：向一个文件块中写数据，客户端指定要写的数据和要写的数据在一个文件中的偏移量。
追加数据：客户端指定要将哪些数据追加到哪个文件后，系统返回追加成功的数据的起始位置。
负载情况
这里只讨论写请求。而且并发的意思是针对同一个文件块的写操作

无并发：假设一个写请求处理完后，下个写请求再来。
并发：GFS同时执行多个写请求到一个文件块上。



# 3. GFS一致性

## 3.1 元数据的一致性

元数据只有一份（不考虑hidden master和日志），不存在副本一致性问题，只考虑传统数据库中的内部一致性问题，即符合数据库的内部约束，GFS对元数据的修改都要加锁，隔离各个操作。

* 修改元数据+无并发
  操作只有成功和不成功两情况，元数据永远是一致的。

* 修改元数据+并发
  GFS存在锁机制，会将各个操作依次执行，与无并发一样。只不过并发的修改后值是多少不确定（是我不确定，不是GFS不确定，也许有一定的机制来对操作排序，也许是自由竞争）。元数据还是一致的。

:key:可以看到，如果数据只有一份，总是一致的。只有并发会产生语义的问题，需要根据应用逻辑进行并发处理。



## 3.2 一个文件块（chunk）的一致性

![img](https://images0.cnblogs.com/blog/422033/201306/09113934-0769397dafac49b7950284959c0fcc38.png)

> 定义：
>
> 1. defined/undefined：客户端写入文件后，与再读出的内容一致，则为defined，反之，则为undefined
> 2. consistent/inconsistent：如果所有客户端读到的文件内容是一致的，就是consistent，反之，则为inconsistent
> 3. undefined but consistent：多个客户端对文件同一个偏移量写数据，由GFS确定顺序和最终写入结果，客户端无法确认再读出的内容是自己写入的，但GFS Chunk Server个副本数据是一致的，这时候为undefined but consistent
> 4. defined interspersed with inconsistent：

这里只讨论一个chunk ，也就是一个文件块的写操作，不涉及整个文件的写流程中数据和元数据的流程，原论文里好像也没介绍文件的写流程。

每个chunk默认有3个副本，不同副本会存在不同节点上，master会设置1个主副本（primary），2个二级（secondary）副本。

当写操作和追加操作失败，数据会出现部分被修改的情况，于是肯定会出现副本不一致的情况，这时就依赖master的重备份来将好的副本备份成N份。以下只考虑操作成功的情况。

* 写一个chunk+无并发

  写一个chunk时，客户端向primary发送写请求（一个chunk对应几个写请求不确定，这里不影响理解，当做一个看就可以了）。primary确定写操作的顺序，由于没有并发，只有一个写请求，直接执行这个写请求，然后再命令secondary副本执行这个写请求。其他secondary都按照这个顺序执行写操作，保证了全局有序，并且只有当所有副本都写成功，才返回成功，用系统延迟保证了数据强一致，即 consistent（所有副本的值都一样） 。

  这个强一致指每个写成功后，所有客户端都能看到这个修改。即论文中说的 defined 。defined 的意思是知道这个文件是谁写的（那么谁知道呢？肯定是自己知道，其他客户端看不到文件的创建者）。也就是当前客户端在写完之后，再读数据，肯定能读到刚才自己写的。

* 写一个chunk+并发

  这时primary可能同时接受到多个客户端对自己的写操作。举个例子，两个客户端同时写一个chunk。w1或w2代表（写操作+数据）。下边表示client1想将这个chunk写成w1，client2想将这个chunk写成w2。

  client1：w1

  client2：w2

  于是primary要将这些写操作按某个机制排个顺序：

  primary：w2，w1
  然后在primary本地执行，于是这个chunk首先被写成w2，之后被覆盖成w1。

  之后所有secondary副本都会按照这个顺序来执行操作，于是所有副本都是w1，这时数据是 consistent 的，也就是副本一致的。因为所有操作都正确执行了，所以两个client都收到写成功了。但是谁也不能保证数据一定是自己刚才写的，也就是 undefined 。这与最终一致性有点像（系统保证所有副本最终都一样，但是不保证是什么值）。

* 追加数据+无并发

  追加数据时，会追加到最后一个chunk，其实和写一个chunk+无并发基本一样。
  但由于追加操作和写文件不一样，追加操作不是幂等的，当一次追加操作没有成功，客户端重试时，不同副本可能被追加了不同次数。

  假设追加了一个数据a

  client：追加a。
  第一次追加请求执行了一半失败了，这个chunk的所有副本现在是这样：

  primary：原始数据，offset1：a
  second1：原始数据，offset1：a
  second2：原始数据

  于是客户端重新发送追加请求，因为primary会先执行操作再将请求发给secondary，所以primary当前文件是最长的（先不考虑primary改变的情况）。primary继续往offset2（当前文件末尾）追加，并通知所有secondary往offset2追加，但是secondary2的offset2不是末尾，所以会先补空。如果这次追加操作成功，数据最终会是这样：

  primary：原始数据，offset1：a，offset2：a
  second1：原始数据，offset1：a，offset2：a
  second2：原始数据，offset1：*，offset2：a
  并且给客户端返回 offset2 。

  于是数据中间一部分是 inconsistent。但是对于追加的数据是 defined 。客户端再读offset2，不管从可以确定读到a。
  这就是追加操作的defined interspersed with inconsistent。

* 追加数据+并发
  两个客户端分别向同一个文件追加数据a和b

  client1：追加a
  client2：追加b
  最后一个文件块的primary接收到追加操作后进行序列化

  primary：b，a
  然后执行，b失败了一次，于是client2再发送一次追b。primary再追加一次。

  primary：原始数据，off1：b，off2：a，off3：b
  second1：原始数据，off1：b，off2：a，off3：b
  second2：原始数据，off1： ，off2：a，off3：b
  client1收到GFS返回的off2（表示a追加到了文件的off2位置），client2收到off3

  也满足off2和off3是 defined ，off1是 inconsistent ，所以总体来说是 defined interspersed with inconsistent

可以看到，不管有没有并发，追加数据都不能保证数据全部 defined，只能保证有 defined ，但是可能会与 inconsistent 相互交叉。
