# **SpanDB: A Fast, Cost-Effective LSM-tree Based KV Store on Hybrid Storage**
Author : Hao Chen, Chaoyi Ruan, Cheng Li,  Xiaosong Ma and Yinlong Xu

Research unit :University of Science and Technology of China,Qatar Computing Research Institute, HBKU,Anhui Province Key Laboratory of High Performance Computing

KV存储在内存内部的处理过程相当迅速，但在I/O的性能表现上有所限制。最新的高速NVMe SSD可以利用其低延迟和高带宽改善这个问题，但价格开销太大。为了平衡这一问题，文中提出了SpanDB，一种兼容RocksDB的LSM-tree based KV store，带有可选的高速SSD部署。这种存储允许用户将大量数据存储在较便宜的SSD上，同时将写前日志(WAL)和LSM-tree的顶部数层放入更小但更快的NVMe SSD中。为了完全利用快速硬盘，SpanDB提供使用SPDK的并行WAL高速写，使能异步请求处理。SpanDB相比于RocksDB吞吐量提升了8.8倍，延迟减少了9.5%到58.3%。

## **Introduction**

在写性能方面表现很好的LSM-tree已经被广泛应用，而在生产KV环境下，这种结构依然有很大吸引力，因为大量的内存缓存经常导致写密集的环境。  
这篇文章致力于将主流的LSM-tree based KV设计调整至适应NVMe SSD和I/O接口，特别关注cost-effective deployment。现今的LSM-tree based KV store不能完全开发出NVMe SSD的潜能，特别是I/O path使得NVMe SSD的低延迟完全得不到发挥。这种特性尤其对WAL不利，而WAL对于数据的有效性和事物原子性都很关键。同时，现存的KV请求处理都假设设备速度很慢，带来很大的软件开销，浪费了CPU的周期。除此之外，NVMe的接口也有一定限制，使得利用NVMe的KV设计更加复杂，导致当前的同步请求处理更加低效。最后，先进的SSD对于大规模部署来说过于昂贵。文中基于这些挑战，做出了以下成果：  
1、同时部署一个容量小但速度快的speed disk和一或多个容量大但便宜的capacity disks来加快写请求和读请求的处理过程。  
2、利用SPDK实现快速并行的访问，来更好的利用SD，绕过Linux I/O stack并且允许高速WAL写。  
3、为轮询I/O设备设计一个异步请求处理流水线，减少了不必要的同步性，使得I/O等待尽量于内存内处理重叠，并自适应地调节前台和后台的I/O。  
4、根据实际工作负载策略性且自适应地调节数据分布。

## **Background and Motivation**

**LSM-tree based KV Stores**

LSM-tree中，为了避免数据丢失和不持续性，存在一个序列write-ahead-log文件，一般有几十GB大小，在执行更新操作前，必须先将其写入日志，直到一个写操作或一个事务完成为止。在请求写入日志后，相关改变可以被应用于MemTable。  
后台操作包括flush和compaction，两者都会产生大量的I/O请求，这会导致前台操作发生I/O冲突和写阻塞。现存的前台后台coordination并不适合polling-based I/O接口。

**Group WAL Writes**

![图1.RocksDB group WAL workflow](https://img-blog.csdnimg.cn/36e8b378728e428f87001bcaf3b4958a.jpeg#pic_center)


现今常见的WAL写是group logging，即在一个日志数据写中包含多个写请求。其工作流程如图1所示，WAL的写操作是线性的，任何时候至多只有一组可以写日志，当一个写操作正在进行时，worker线程会处理来自新组的写请求，将它们加入一个共享队列，首个入队的会被视为leader，这个leader会从同组线程处收集log entry，然后等待前一组的leader写完成后通知它们继续执行。当leader完成写日志后，会通知同组成员做出实际更新，最终它会做出commit操作并解散该组，通知下一组继续执行。  
在这一操作中，各类线程的等待时间带来了软件开销。

**High-Performance SSDs Interfaces**

Inter提供了Storage Performance Development Kit，这是一组访问NVMe设备的用户空间套件，它将驱动移动到用户空间，避免了系统调用，提供了zero-copy访问，它对于硬件的访问方式是轮询而不是中断。

![图2.4KB顺序写的并写分析](https://img-blog.csdnimg.cn/0cb0e6063442469fa8fbbba1eaec3890.jpeg#pic_center)


如图2所示，SPDK可以为KV I/O带来显著的提升。  
NVMe SSD确实提供了一定程度的并发性，但超过3个并行写操作后不会带来更高的SPDK IOPS。
使用SPDK有一定的限制，当一个SSD与SPDK相联系后，其他处理不能访问它，如果不与其联系，则会带来I/O性能损失。

**SpanDB Overview**

![图4.SpanDB总体架构](https://img-blog.csdnimg.cn/f2162d4950a540c09ee30c0e37922962.jpeg#pic_center)


SpanDB的整体架构如图3所示，在DRAM上，和RocksDB一样，SpanDB也维护多个immutable memtable和一个mutable memtable，SpanDB兼容RocksDB，没有引入额外的数据结构算法和新的操作语义。主要的不同在于异步处理模型，减少了同步开销，且自适应地调度任务。  
磁盘上的数据被分为CD和SD部分，SD部分会进一步被分为WAL区域和data区域。SpanDB重新设计了RocksDB的group WAL写。数据区域管理LSM-tree的顶部数层。为了减少对RocksDB的修改，SpanDB引入了一个轻量级的文件系统TopFS。CD部分的管理方式与RocksDB无异。  
图3还展示了SpanDB的I/O流，SD中的WAL区域专注于日志管理，data区域接收所有flush操作。SD的data区域和CD都会接纳用户的读操作和compaction操作，SpanDB使得两个分区可以同时进行compaction，并自动调节前台后台任务。SpanDB也能实时动态根据带宽调整tree level placement。  
SpanDB通过SPDK和SD加速了WAL写，同时因为SD也可以存储部分数据，使得SD的带宽得到最大利用，而SD-CD的数据分配也使得I/O资源得到合理规划。而因为部分I/O操作被装载到了SD上，SpanDB也减少了读密集工作负载的尾延迟。最后，通过减少同步性，平衡前台后台I/O需求，使得可以在节省CPU资源情况下使用轮询I/O设备。  
而SpanDB的限制在于SD必须同一个操作相联系，使得很难共享其资源，并且在只有读的工作负载下，SpanDB带来少量的异步处理开销。

## **Design and Implementation**

**Asynchronous Request Processing**

![图4.异步请求处理工作流程](https://img-blog.csdnimg.cn/a209f753e98e44ef9c01587713017de2.jpeg#pic_center)


SpanDB的异步请求处理过程如图4所示。在一个具有n个CPU核的机器上，每个线程占用一个核，用户将客户端线程数配置为Nclient，内部线程数为n-Nclient个。内部线程分为logger和worker，一个叫head-server thread的管理线程会动态调整两种线程的数量。  
SpanDB在RocksDB基础上引入了一个读队列Qread和三个写队列QProLog、QLog和QEpiLog。
在执行读请求时，client线程会先从memtable中查找，命中则直接返回，未命中则需要将读请求放入Qread，之后worker线程会读取Qread中的读请求，并给出读取结果，而client线程通过获取请求状态接口获取结果。  
在执行写请求时，client线程直接将写请求放入QProLog队列，之后通过获取请求状态接口获取结果。worker线程读取QProLog中的请求，将WAL entry序列化，放入QLog。logger线程之后一次性读取QLog队列中的所有请求，执行group logging，并将写请求放入QEpiLog。worker线程从QEpiLog中读取请求，执行实际更新。  
在执行过程中，为减少上下文切换，SpanDB默认启动一个logger，根据工作负载在1-3个logger之间浮动。

**High-speed Logging via SPDK**

![图5.SpanDB的并行WAL日志机制](https://img-blog.csdnimg.cn/bebdf58128464d4c9c25b9a408064594.jpeg#pic_center)


如图5所示，SpanDB的WAL写入仍是group logging，多个logger线程从QLog中获取所有请求，并发执行。  
SpanDB在SD的WAL区域分配多个逻辑page，每个page有唯一Log Page Number，其中一个page是metadata page。多个连续page组成一个log page group，每个group对应一个memtable，容量足够容纳一个WAL。当对应的memtable flush后，group被回收重利用。metadata page记录每个group的起始LPN。当memtable写满时，metadata page记录结束LPN。  
SpanDB配置多个logger并发执行，每个logger有单独的WAL buffer，logger的下一个写入位置通过原子操作分配得到。

**Offloading LSM-tree Levels to SD**

SpanDB将树顶部的数层放入SD，来充分利用SD的性能。SpanDB加入了一个叫做TopFS的文件系统来减少对RocksDB的修改。文件系统的管理类似于WAL，由一个metadata page和多个data page组成，metadata page通过hash table记录每个文件的起始LPN。  
对于KV store，写延迟更加明显，为了实现WAL优先写，采取以下措施：  
1、设置专门的logger请求队列  
2、将flush/compaction默认请求大小从1MB变为64KB  
3、限制处理flush/compaction的worker数量  
由于SPDK接口绕过内核，因此无法使用系统的cache，无法充分利用局部性原理，因此在TopFS之上，SpanDB实现了一个自己的cache。  
head-server thread会监控SD的带宽使用情况，动态调整SST文件的分配，其调整的是新创建的文件，因此，同一个level的文件可能会同时分布在CD和SD上。

## **Evaluation**

![图6](https://img-blog.csdnimg.cn/ae596c08297f42bf9b04b40e84f167ff.jpeg#pic_center)


如图6所示，基于文件系统的顺序log写入不能充分发挥高端SSD的性能，SpanDB在除去读密集的工作负载下，不同硬件组合均可以显著提升I/O吞吐量并降低延迟，而高端硬件在SpanDB上收益更高。

![图7](https://img-blog.csdnimg.cn/16dd7aedbd324b6c8bc4111d5cbe765c.jpeg#pic_center)


如图7所示，SpanDB相比于RocksDB性能得到提升，高端硬件更加明显。

## **Conclusion**

文中设计了一种较为便宜的方式，通过将一块小的昂贵的高速SSD部署在最需要的地方来加速设备，而不需要更改其他大部分的便宜硬件。结果表明主流LSM-tree based设计的表现可以通过这样部分的硬件升级得到明显改善。