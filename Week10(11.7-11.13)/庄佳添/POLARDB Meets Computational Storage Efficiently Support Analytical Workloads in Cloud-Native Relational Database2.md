# POLARDB Meets Computational Storage: Efficiently Support Analytical Workloads in Cloud-Native Relational Database

这是一篇阿里和ScaleFlux合作发表在FAST'20上的一篇文章，这是第一篇将CSD部署在云原生数据库上的工作，主要将table scan的操作从cpu上下放到存储结点内，从而减轻主机端cpu的压力和主机端和存储器件之间的IO压力。作者的主要工作为：在存储节点内使用FPGA完成table scan操作，利用FPGA的并行性提升系统性能；对从POLARDB的存储引擎到文件系统到存储节点的软件栈和数据块格式做出相应修改，以支持在POLARDB系统中使用CSD；在FPGA内部设计并实现一种数据格式无关的比较部件，以支持针对不同数据库谓词的高效扫描。实现该工作的主要挑战为：修改POLARDB分析引擎、存储引擎、文件系统到存储器驱动之间的软件栈以支持CSD；使用低成本的方式完成CSD的实现。

### Introduction

1. 为支持可扩展性和可容错性，云原生数据库自然的采用了计算-存储分离架构，云原生数据库目前在TP负载下的性能比肩本地版本数据库。但对于AP负载，更大量的数据使得计算节点和存储结点之间的IO成为稀缺资源，尤其在为了更好的支持TP负载而采用行存储的背景下。因此为提高AP负载下的性能，将数据密集操作如table scan下放到存储节点成为唯一可行的方案。这个简单的idea在云原生数据库上的实际实现很困难：1. 每个存储结点必须配备足够的计算能力 2. 在注重成本效益的云原生数据库上，不能提高存储结点的成本。因此采用cpu与特殊硬件的异构架构成为平衡计算能力和经济效益的方案。
2. 本文工作的核心idea就是将CPU的table scan操作下发并分配到POLARDB的各个存储节点中。相比于将table scan下发到一块独立的专用硬件上，下发到storage drive(存储器驱动)上有两个好处：1. 消除了存储器件到专用硬件之间的IO； 2. 避免了专用硬件的热点问题。 但实际中的实现有两个挑战：1. 如何将table scan经过各个软件层级下发到csd上；2. 如何平衡经济和算力。

### Background & Motivation

1. 云原生数据库：为支持可扩展性和可容错性，采用计算-存储分离架构。与主流数据库如Mysql兼容并以更低的成本比肩传统版本的数据库的TP负载下的性能。
2. POLARDB计算-存储分离架构如Figure 1所示。最上层是应用程序的云服务器端，中间为数据库结点，底下为存储结点。云服务器和数据库结点之间实现读写分离和负载均衡，数据库结点和存储结点之间由RDMA实现。数据库/计算结点被分为2中类型：读写节点和只读结点。

![image-20221113213637878](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211132136964.png)

如下图，每一个POLARDB实例被分配一个volume，每个volume包含多个chunk，volume可以根据需求动态扩容这个volume可以理解为：表示分配给一个polardb实例的空间，映射到存储节点上，实际存储还是在存储节点上？

chunk是数据分配的最小单位，1个chunk不会跨越多个硬盘，并且默认有3个副本，chunk可以在chunkserver访问热的时候移动。

PolarSwitch：部署在计算节点上，libpfs将chunk的请求的偏移量、长度等信息发给polarswitch，由polarswitch分割并转发给所在的leader chunk server。

chunkserver：用来存储chunk和提供对chunk的随机访问服务。1个存储节点上由多个chunkserver进程， 每个chunkserver有用一块独立的nvme ssd和一个cpu core。

polarctrl：维护元数据，chunk所在位置等元数据保存至polarswitch等cache中。

![](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211132231621.png)



3. **POLARDB中的table scan下推**

   根据上文所讲，POLARDB的数据密集型计算的下推关键在于 如何 在低经济成本的前提下为存储结点配备足够的计算能力。

   1. scale up存储节点。scale up应该是指提高存储节点的cpu性能。存在如下问题：1. 行存在cpu下会产生严重的资源利用不充分，如缓存、SIMD。因此通过增加cpu性能来弥补cpu的低效是不经济也不可取的。

   2. 采用中心化的cpu-特殊硬件异构架构，特殊硬件通常为fpga/gpu的pcid板卡。存在如下不足：1. 对于table scan这样的数据密集操作，几乎所有的数据都要从存储器件搬到特殊硬件上，造成pcie通道的压力，能源消耗，负载间的干扰。2. 数据处理热点，多个存储节点的多个nvme ssd并发，将数据交由特殊硬件处理，造成特殊硬件成为数据处理的热点，对pcie板子的io带宽是个挑战。

      ![image-20221113232214873](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211132322929.png)

   3. CSD没有广泛被使用的很大原因在于成本：硬件实现成本和开发必要的软、硬件以支持csd的成本呢。作者选用FPGA，从两个方面减少成本：1. FPGA同时实现flash memory control和计算，FPGA的电路级编程降低开发周期和成本。 2. FPGA采用host-management策略，包括地址映射、请求调度、垃圾回收等都有host管理。**这里的host指的是啥？**host-management可以促使CSD集成到现有软件栈中。使得为设计和优化CSD的API提供灵活性，应用程序可以利用CSD可配置的计算资源。同时，CSD接入Linux IO栈中以实现普通的IO的操作。**所以推测这里的host指的是存储节点**，通过存储结点的cpu管理。因为之前的table scan是通过软件栈下发到cpu，现在将csd接入到cpu下一层，用cpu管理csd。或者host指的是软件栈的 **计算存储驱动器**？具有调度、地址映射的功能。

### Design and Implementation

1. 如上文所述，面临2大挑战
   1. 将table scan通过整个软件层级结构下推到CSD。 table scan操作在用户侧polardb引擎是通过指定File中的offset（1个SSTable是1个File，通过offset读一个block，offset通过index block维护），CSD是通过LBA拿数据的，所以对存储引擎到CSD中间的所有软件栈都需要修改或增强。
   2. FPGA开发成本低但贵，且始终频率远小于cpu，因此需要大量实现并行。
2. 支持将表扫描下推通过整个软件栈
   1. ![image-20221114105224215](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211141052275.png)
   2. MMP是前端的分析处理引擎，负载接卸、优化、重写sql并组织成DAG执行计划。mmp不需要修改。
   3. **POLARDB Storage Engine。**POLARDB存储引擎原始是利用存储节点的cpu来处理table scan，现改成产csd，所以现在需要修改在他之下的存储IO栈。 每个table scan请求包含：1. schema(对于csd无法实现的扫描谓词存储引擎还要进一步筛选) 2. offset in file ；3. 谓词。csd无法处理所有谓词，因此修改后的存储引擎提取出谓词子集，并在返回数据后用完全谓词检查。4. 并发发出多个table scan请求，以充分利用多个存储节点。
   4. **PolarFS**。1. 由于对blcok进行压缩，因此block不能是固定大小对其如4-KBaligned，POLARFS实现粗粒度的条带分割，以确保1个block不会跨越2个csd。对于少数跨域2个csd的情况使用存储结点的cpu进行表扫描。2. POLARFS在**并行性**方面的修改。同一个table scan可能涉及多个csd，所以POLARDB将设计多个csd的请求分割成多个子请求给对应的csd。3**. 将数据地址信息转化成LBA中的偏移量**。？？
   5. Computational Storage Driver。CSD的host-management中指的是 computational storage driver管理(driver是在内核态，且直到物理地址，所以**应该是在存储节点上？**)。其工作：1. 重新安排谓词顺序以便更好地流水线处理；2. 将LBA转成PBA；3.将请求分割成更小的请求，一与其他io请求共同调度，避免其他IO请求等待过长，二是提高并行性降低硬件缓存的开销。 4.调度GC进程，避免干扰table scan，在数据迸发时放弃GC。
3. 降低硬件实现成本。
   1. 核心思路在于最大化FPGA硬件资源利用率和使用效率
   
   2. **数据格式**。
   
      1. FPGA难以同时支持多种数据类型的比较，因此作者修改POLARDB存储引擎将数据以**memory-comparable format的形式保存**，数据可以通过memcmp()函数直接比较而不用解码，因此FPGA上只需要实现memcmp()功能的1种比较算子就行，不用考虑数据类型，从而降低FPGA上table scan的计算资源。
      2. 原有的block format适合cpu处理不适合用FPGA处理，因此作者如figure b增加block head。restart key用于方便在有前缀压缩的场景下进行key搜索。2个好处：1. csd可以直接解压block并检查crc校验码，而不用要求POLARDB存储引擎发送块的大小信息(这里应该不是csd向存储引擎请求，而是存储引擎下发table scan请求的时候带上块大小信息？)。 2. # of keys/restart keys可以帮助FPGA方便的处理block种的restart和检测每个块的尾部。![image-20221114160757517](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211141607586.png)
   
   3. FPGA实现。采用FPGA实现flash memory control和table scan。FLASH controle模块中有1个soft LDPC来进一步支持3D TLC/QLC颗粒(BCH下寿命过短，LDPC可以提升擦写次数)。table scan模块采用**并行和流水线**的方式。每个scan引擎包含1个memcmp模块和结果评估RE模块。$P=\sum_{i=1}^{m}\left(\prod_{j=1}^{n_{i}} c_{i, j}\right)$为谓词总和(析取范式)，RE不断地去判断当前 已被比较的$c_{i, j}$结果足够判断P的真值，当可以判断P真值就停止并处理下一行。
   
      ![image-20221114165323222](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211141653265.png)

### Evaluation

1. Evaluation主要分为两部分：1. 单个CSD的评估 2. 应用CSD后POLARDB的性能评估。

2. 实验环境和CSD基本性能：1. 64-layer 3D TLC NAND flash，配上PCIe3*4接口，性能比肩商用版的NVMe SSD；2. Xilinx UltraScale+ KU15p FPGA chip  3. a POLARDB instance with seven database nodes and three storage nodes)

3. 单个CSD的table scan性能评估。FPGA中解压缩引擎的吞吐量和数据的压缩情况相关，当压缩率为60%和30%时两个解压缩引擎的总吞吐量为2.3GB/S和2.8GB/S。表扫描引擎的吞吐量也和数据信息相关，如行大小、table schema、扫描条件谓词。使用TPC-H进行测试，6个表扫描的选择率分别1.25%, 15.17%, 30.34%, 54.04%, 63.22%, 和100.00%。压缩率为0.5，snappy压缩库。![image-20221114192540866](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211141925927.png)

   单个结果由10次测量取平均值得到。结果表明1. 相比CPU，CSD的table scan降低扫描延迟和cpu利用率；2.TS-6获益最少因为他的谓词最简单(100%选择率)，比如4个table scan引擎就无法流水线，因为没有谓词，实际上利用不到scan引擎； 3. cpu-based的表扫描cpu使用率基本不变，csd-based的cpu使用率与选择率成正相关，因为数据越多CPU处理调度等信息也就越多；4. 轻量级压缩可以显著提高cpu-base的table scan性能，但要付出更多的cpu使用率。

   ![image-20221114193014905](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211141930967.png)

   Figure7 a表示csd到host内存的传输数据量，b表示host内存的全部传输数据量。结果表示1. csd-based显著减少存储到内存的数据量。

   ![image-20221114200018530](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211142000569.png)

4. 系统级评估

   1. 1个由32个sql引擎容器的POLARDB实例，分布7个数据库结点和3个存储节点，每个存储节点有12个csd，每个csd有3.7TB大小。实验表明：1. 随着请求数的增多，csd-based计算下推比cpu-based计算下推的优势更加明显，如32并发请求时只有4组提升大于30%，但128并发时有11组提升大于30%。这是因为并发请求多时csd能更好的利用硬件资源和并行调度。2. csd-based 计算下推在snappy压缩时性能提升更多，这是因为cpu架构下同时处理解压和table scan使cpu性能成为瓶颈，但fpga可以充分利用解压引擎。3. 少数情况下cpu-based性能由于csd-based，这可能是由调度的次优行为造成的硬件资源使用不充分。
   2. ![image-20221114202922308](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211142029366.png)
   3. Figure11显示了存储节点内的数据传输总量和存储节点和数据库节点之间的数据传输总量。![image-20221114204025223](https://raw.githubusercontent.com/Lia0104/BlogImg/main/imgs/202211142040279.png)

### ELSE

1. 云原生数据库：https://blog.csdn.net/weixin_50843918/article/details/125412192

2. RDMA:  Remote Direct Memory Acces。RDMA允许用户态的应用程序直接读取或写入远程内存，不经过操作系统，无内核干预和内存拷贝发生，节省了大量 CPU 资源，提高了系统吞吐量、降低了系统的网络通信延迟。

3. YourSQL是in storage 还是near storage

4. FPGA是host management的，这里的host指的是什么？如何管理？

5. nand flash和ssd的关系：ssd的存储单元是nand flash颗粒，如SLC、TLC颗粒等

6. POLARFS 粗粒度的条带化防止1个block分布在2个csd上？什么原理？

7. memory comparable format
   * 保证编码前和编码后的**比较关系不变** 的方案我们称为 Memcomparable
   * TiDB的 codec库 、RocksDB
   * 使用 memcomparable 模式编码的数据可以不经过解码而直接使用 C 的 memcmp 函数来比较数据的大小，可以保证数据在底层存储的顺序是和结构化层看到的数据顺序是一致的。这样的编码方式可以大大的简化和加速大量数据排序的流程。
   
8. 增加block header的作用？为什么能帮助fpga进行table scan

9. data compression和block compression的区别

10. 关系型数据库行存储和LSM-tree的关系？ key为主键，data为row？？

11. CSD是一个ssd/chunkserver有一个csd还是一个存储节点有一个CSD。

    1个存储结点可以有多个csd，应该是1个chunkserver1个csd。

12. host的cpu有什么额外的工作：存储节点的同步等等

