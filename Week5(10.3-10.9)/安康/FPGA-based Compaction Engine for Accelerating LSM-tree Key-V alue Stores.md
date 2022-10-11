FPGA-based Compaction Engine for Accelerating LSM-tree Key-V alue Stores
作者xuansun，City University of Hong Kong，©2020 IEEE

本文设计并实现了一个基于FPGA的压缩引擎，以加速基于LSM树的键值存储的压缩。
充分利用FPGA上的流水线机制，利用FPGA芯片的高带宽和并行值传输，compaction性能显著提高，提出了键值分离和索引数据块分离策略（在选择最小键时选择键值分离策略，提高处理效率）。

资料链接：
Sstable：Sorted String Table
https://www.cnblogs.com/Jack47/p/sstable-1.html
https://cloud.tencent.com/developer/article/1531876
Mentable
https://zhuanlan.zhihu.com/p/149794113
一文彻底搞懂leveldb架构_神技圈子的博客-程序员秘密_leveldb
https://cxymm.net/article/songguangfan/124828824

Introduction:
LSM树的键值存储适合写操作，原因是存储冗余数据为代价；将随机写变为顺序写VS传统的RDBMS更适合于以读取请求为主的工作负载。
LSM树compaction对旧数据进行排序合并(类似垃圾处理),
Background:

![](C:\Users\AnKang\Desktop\图片1.png)

图一：传统的基于LSM树的键值存储由两种类型的任务组成。一个是主线程，它负责处理来自用户的写和读请求。另一个是后台线程，负责调度和执行压缩任务。因此，当CPU因大量压缩而过载时，主线程可能会减慢速度，从而导致用户的响应延迟更长。
Immutable(不可变) Memtable：
刚才提到leveldb的一次写入操作并不是直接将数据写入到磁盘文件，而是采用先将数据写入内存的方式。所以,memtable就是一个内存中进行数据组织与维护的结构。在memtable中，数据按用户定义的方法排序之后按序存储。等到其存储内容到达阈值时（4MB）时，便将其转换成一个不可修改的memtable，与此同时创建一个新的memtable来供用户进行读写操作。memtable底层采用跳表，它的大多数操作都是O(logn)。
当memtable的容量达到阈值时，便会转换成一个不可修改的memtable即immutable memtable。它同memtable的结构定义一样。两者的区别只是immutable memtable是只读的。immutable memtable被创建时，leveldb的后台压缩进程便会利用其中的内容创建一个sstable,然后持久化到磁盘中。
LSM两种压缩方式：
(1)存储格式转换,将随机写入转换为顺序写入来实现高写入吞吐量
(2)合并存储在不同SSTable中的数据,每个级别包含有限数量的具有特定大小的SSTable。一旦i级达到极限，后台线程将从i级和i+1级收集相关的SSTable，然后执行数据排序合并操作，以生成i+1级的新SSTable。0level特殊直接压缩，
Motivation：
基于LSM树的键值存储因其高效的写入性能而被广泛应用。压缩在LSM树中起着关键作用，它合并了旧数据，并可以显著降低整个系统的总体吞吐量，特别是对于写密集型工作负载。本文设计并实现了一个基于FPGA的压缩引擎，以加速基于LSM树的键值存储的压缩。
System overview：
级别0的输入数量等于所涉及的SSTable的数量。对于其他级别，SSTables都已排序。最小的密钥必须在第一个文件中。因此，所涉及的一组SSTable可以连接为一个大的SSTable，并且输入的数量是一个。
Hardware implementation：

![](C:\Users\AnKang\Desktop\图片2.png)

解码器、比较器和编码器，先解码再比较将选择最小的有效键值对，并在编码器中进行编码。
1）解码器+编码器分离：为了提高键值解码任务的吞吐量，每个解码器模块被分为索引块解码器和数据块解码器。
2）通过键值分离通过减少不必要的值流传输，所提出的解决方案可以表现得更好。

将Immutable MemTable转储到级别0的第一种压缩类型的优先级高于第二种压缩类型（以FPGA为中心）
对于LevelDB中的0级压缩，在大多数情况下，压缩过程涉及到0级和1级上的8个SSTable

Conclusion：
优化了索引和数据块管道，以减少DRAM延迟的影响，在处理过程中将键值实行分离操作，通过利用FPGA芯片的高带宽和并行值传输，compaction性能显著提高。