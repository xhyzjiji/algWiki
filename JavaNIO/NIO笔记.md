## 1.常规read()操作

<img src="/Users/panyongfeng/Library/Application Support/typora-user-images/Screen Shot 2020-08-23 at 1.42.15 PM.png" alt="Screen Shot 2020-08-23 at 1.42.15 PM" style="zoom:75%;" />

数据通过DMA从存储介质到内核缓冲区，然后通过CPU把数据拷贝到用户缓冲区。

把数据从内核空间拷贝到用户空间似乎有些多余。<font color='red'>为什么不直接让磁盘控制器把数据送到用户空间的缓冲区呢？</font>这样做有几个问题。首先，硬件通常不能直接访问用户空间 。其次，像磁盘这样基于块存储的硬件设备操作的是固定大小的数据块，而用户进程请求的可能是任意大小的或非对齐的数据块。在数据往来于用户空间与存储设备的过程中，内核负责数据的分解、再组合工作，因此充当着中间人的角色。

由于内核直接与设备打交道，当从块设备读取数据时，一般都是以扇区为最小单位读取，因此内核缓冲区的大小也理所当然的使用最小单位的倍数来存储。而用户空间的缓冲区则与应用相关。

<img src="/Users/panyongfeng/Library/Application Support/typora-user-images/Screen Shot 2020-08-23 at 1.45.40 PM.png" alt="Screen Shot 2020-08-23 at 1.45.40 PM z" style="zoom:75%;" />

内核缓存区对存储数据进行预读，用户缓存可以直接从内核缓冲区读取相应数据，而不必多次执行系统调用。

## 2.虚拟内存

虚拟内存意为使用虚假(或虚拟)地址取代物理(硬件RAM)内存地址。这样做好处颇多，总结起来可分为两大类: 

1. 一个以上的虚拟地址可指向同一个物理内存地址。
2. 虚拟内存空间可大于实际可用的硬件内存。

### 2.1 内存映射

<img src="/Users/panyongfeng/Library/Application Support/typora-user-images/Screen Shot 2020-08-23 at 3.31.06 PM.png" alt="Screen Shot 2020-08-23 at 3.31.06 PM" style="zoom:80%;" />

当内核空间地址与用户空间的虚拟地址映射到同一个物理地址，就可以节省一次从内核缓冲区到用户缓冲区的（通过CPU）的数据拷贝。

但是这样的映射是有条件的<font color='red'>（天下没有免费的午餐）</font>：内核与用户缓冲区必须使用相同的页对齐，缓冲区的大小还必须是磁盘控制器块大小(通常为 512 字节磁盘扇区)的倍 数。操作系统把内存地址空间划分为页，即固定大小的字节组。内存页的大小总是磁盘块大小的倍数，通常为 2 次幂(这样可简化寻址操作)。典型的内存页为 1,024、2,048 和 4,096 字节。

### 2.2 内存页面调度

要得到虚拟空间大于硬件内存空间，就需要进行内存页面调度。

物理内存充当了分页区的高速缓存；而所谓分页区，即从物理内存置换出来，转而存储于磁盘上的内存页面。当进程引用某内存地址时，CPU上的MMU单元（该设备包含虚拟地址向物理内存地址转换时所需映射信息）会先计算地址所在页面号，然后将虚拟页面号转换成屋里页面号，如果物理内存不存在与之映射的页面，则触发一个陷阱调用，从而转入内核态，从设备读取相应数据，成功后更新MMU映射内容。这时候很可能需要将别的物理页移出物理内存，以腾出足够的内存空间。如果此时需要移出的页面内容发生过变化，则先执行页面调出，并把内容拷贝到存储设备的分页区。

### 2.3 内存映射文件（MMAP）

<img src="/Users/panyongfeng/Library/Application Support/typora-user-images/Screen Shot 2020-08-23 at 5.40.53 PM.png" alt="Screen Shot 2020-08-23 at 5.40.53 PM" style="zoom:80%;" />

与内存映射不同的是，内存映射文件把文件数据当做内存的一部分，当进程读取数据页缺失时，系统自动触发陷阱，然后从存储介质读出对应页面的数据，无需使用read()和write()系统调用。系统根据内存负载情况，智能地进行预读和刷新。

<font color='red'>对于高性能存储而言（比如同一时刻大量需要对文件不同位置进行读写），一些定制化需求就不能依赖系统自动缓存，这时候就需要使用DIO来定制自己的页缓存机制。</font>

