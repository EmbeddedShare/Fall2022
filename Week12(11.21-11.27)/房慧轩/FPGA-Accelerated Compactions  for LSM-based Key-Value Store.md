# **FPGA-Accelerated Compactions  for LSM-based Key-Value Store**
Author : Teng Zhang, Jianying Wang, Xuntao Cheng, Hao Xu, Nanlong Yu, Gui Huang, Tieying Zhang,Dengcheng He, Feifei Li, Wei Cao, Zhongdong Huang, and Jianling Sun

Research unit :Alibaba-Zhejiang University Joint Institute of Frontier Technologies, Zhejiang University
{jason.zt,beilou.wjy,xuntao.cxt,haoke.xh,qushan,tieying.zhang,dengcheng.hedc,lifeifei,mingsong.cw}@alibaba-inc.com
{yunanlong,hzd, sunjl}@zju.edu.cn

## **Abstract**

LSM-tree KV store由于其较高的写效率和较低的花费，被广泛应用于各种产品中。LSM-tree依赖于一个后台合并操作来合并数据记录和收集垃圾，以此实现上述优势。但由于LSM-tree KV store在运行时各层的大小会过大，CPU和I/O的资源冲突会增加，导致系统性能下降。同时由于最新的磁盘存储中I/O容量的增大，使得合并操作的瓶颈逐渐变为了CPU，这使得后台合并和前台的查询事物操作互相竞争CPU资源。这篇文章提出把合并操作装载到FPGA上，以此加速合并操作并且减少CPU的资源冲突。

## **Introduction**

LSM-tree KV store对于那些对价格和性能都很敏感的应用很合适，因为其写效率可以满足性能需求，而同时存储开销也弥补了SSD的缺点。

LSM-tree KV store通常会利用其他数据库的结构来提供一个通用的存储服务，以此在读写性能和开销上同时获得最佳效果。
但尽管采取了索引和缓存等措施，LSM-tree在长时间面对WPI工作负载时，性能仍然会下降。因为LSM-tree的包含广阔键值范围的多层结构使得查询操作不得不横跨多层来将分散的记录合并为一个完整的结果，这其中还要滤过无效的被删除记录，进一步增加了开销，所以提出用后台合并操作来维持LSM-tree的形状，但这一后台操作需要大量计算和磁盘I/O资源。

![图1](https://img-blog.csdnimg.cn/f0b23134396a40f5b0b33e0e3933f058.jpeg#pic_center)


如图1所示，当用于处理合并的线程增加到32之后，再增加线程数会使得用于前台的CPU资源减少，反而使得总吞吐量降低，而且即使线程数到达32，依然无法快速解决上述的问题。

先前的研究表明可以通过调度合并操作，使得合并结果在必要以及和其他资源冲突最小的情况下进行，然而在实际过程中这两个条件一般互相冲突。这篇文章提出将合并装载到FPGA上，因为：  
1、这样减少了CPU的I/O密集操作开销，提高CPU吞吐量。  
2、FPGA很适合合并的流水线计算操作。  
3、FPGA的能量利用率高，可降低TCO。

在FPGA中，文中设计并且实现了一个合并操作的三级流水线：解码/译码输入和输出、合并数据以及管理缓存中的中间数据。

文中还实现了一个FPGA driver，这是一个异步合并任务调度器，用于协助装载过程，提高效率。

除此之外，文中还对FPGA的合并吞吐量分析进行了建模，用于预测不同的合并任务的性能。

## **Background and Motivation**

**Motivation**

问题1，破碎的第L0层。在L0层的数据块互相之间的键值域有重叠，因为它们并没有经过排序操作，这使得点查询需要检查多个数据块。随着时间经过，L0层的数据会堆积，进一步增加开销。

![图2](https://img-blog.csdnimg.cn/8584947ab6c64e0898758ea9e20e2c2f.jpeg#pic_center)


问题2，瓶颈的转移。合并操作由解码、合并和译码三部分组成。如图2，当value的大小增加时，计算时间所占比率会降低，但是对于短键值对来说，计算占的时间能达到60%。这表明合并操作的瓶颈在短键值对情况下是计算，而其他情况下才是I/O。

**Design and Implementation**

![图3](https://img-blog.csdnimg.cn/8585e478481e4909b02a61facf04f882.jpeg#pic_center)


图3展示了FPGA装载的合并的设计。这一设计整合了X-Engine。它利用memtable来存储新插入的KV记录，利用缓存来存储常被访问的键值对和数据块。它还有一个全局索引来加速查询，在原始的系统中，所有put,get,delete,multi-get以及flush和合并操作都在CPU上执行。

为了装在合并，设计中提出了一个管理新发出的合并任务的Task Queue、一个在内存中存储合并完成的键值记录的Result Queue和一个软件端的Driver，它的管理内容包括负责将数据从host传送到FPGA。在FPGA中还有多个用于合并KV记录的Compaction Units。

**Driver**

![图4](https://img-blog.csdnimg.cn/7340830af95c48eea7d4ccb97bf9ed44.jpeg#pic_center)


在LSM-tree KV store中，当Li层的数据容量达到一个阈值时，就会触发Li层和Li+1层的合并。文中提出了三种类型的线程来促进装载：builder threads，dispatcher threads和driver threads。这三者都在CPU端，分别用于创建合并任务、调度相关任务给CUs以及将合并好的数据块装载回存储系统。如图4所示。

当有合并被触发时，builder thread会把extent分为多个大小相似的组，每个组形成一个合并任务，同时将其数据装载入内存，这个任务会被送入task queue，等待被dispatcher thread分配给相应CU，builder thread也会检查result queue，在任务成功时将合并好的数据块装入存储系统。如果FPGA处理失败，会启动一个CPU合并线程重试这个任务。

Dispatcher thread会通知driver thread来将数据转移到FPGA上。

Driver thread会将数据转移到FPGA上，并且通知相应CU开始工作，当一个合并任务完成后，driver thread会中断，执行一个callback函数，将合并好的数据块送回host，并且将完成的任务放入result queue。

文中还设计了指令通路和数据通路来加速传输效率。  
Instruction Path用于传输较小且频繁的数据。  
Compaction Data Path使用了DMA，传输用于合并的数据。  
Interruption Mechanism用于在合并任务完成后传输中断。  
Memory Management Unit用于分配FPGA中的存储空间，来存储host的输入数据。

**Compaction Unit**

![图5](https://img-blog.csdnimg.cn/fd356f593bc14d33b60f10b918e15904.jpeg#pic_center)


CU是FPGA中用于合并操作的逻辑实现。同一个FPGA可以装载多个CU，数量与硬件资源有关。其设计如图5所示，其中Decoder，Merger，KV Transfer和Encoder用于流水线的各个阶段，KV Ring Buffer和Key Buffer是为了其后的模组而存储数据，Controller用于协调各个模组的执行。

**Analytical Model for CU**

![图6](https://img-blog.csdnimg.cn/765703221a6b43d0b032aba619382795.jpeg#pic_center)


![图7](https://img-blog.csdnimg.cn/b7007cb1ca2449b892eaf338e74e52d0.jpeg#pic_center)


文中提出了一个CU的分析模型，来指导资源配置。图6展示了模型中的各类概念。

图7的公式表明了吞吐量的计算方式。

## **EVALUATION**

**Evaluating the FPGA-based Compaction**

![图8](https://img-blog.csdnimg.cn/0a5df147add34253bde7603c9639b61b.jpeg#pic_center)


如图8所示，FPGA-offloading的合并的吞吐量相比于CPU的提高了203%到507%，在所有KV大小规格下实现了性能提升，在短键值情况下提升相对较少。

![图9](https://img-blog.csdnimg.cn/f5dab10d125f4f0581c03d66ddcb007f.jpeg#pic_center)


图9表示模型预测结果与实际情况的对比，可见误差不超过13%，最低在5%以内。

**Evaluating a KV Store with FPGAoffloading of Compactions**

![图10](https://img-blog.csdnimg.cn/69ef0dfe5a504900a3ef22afae0c4ec6.jpeg#pic_center)


图10表明FPGA-offloading的合并相比于CPU-only的最好情况也实现了23%的吞吐量提升。

![图11](https://img-blog.csdnimg.cn/de437c4e5e95486e912910cde72e6b9a.jpeg#pic_center)


如图11所示，FPGA-offloading的能源利用率提高了31.7%。

## **Conclusion**

文中表明LSM-tree KV由于资源冲突、各层大小超标等问题而导致合并变慢，将合并装载到FPGA上可以提高23%的系统吞吐量，并且提高31.7%的能源利用率。
