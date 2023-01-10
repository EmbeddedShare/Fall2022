## In-storage Processing of I/O Intensive Applications on Computational Storage Drives

论文链接：https://ieeexplore.ieee.org/stamp/stamp.jsp?tp=&arnumber=9806270

此工作由加州大学欧文分校的Ali HeydariGorji和NGD的Mahdi Torabzadehkashi等人发表在ISQED(2022)。常见的可计算存储驱动器（CSD）例如带有通用处理器的SSD，其上的通用处理器可以用来执行存内计算。通过将计算从主机端移动到数据存储端从而减少数据搬运成本，CSD可以提高大数据分析的性能以及优化能耗。本文介绍了作者团队实现的Solana，首个E1.s形态的12-TB CSD。三个不同NLP应用场景下相较于常规商用SSDs，Solana可以提速3.1x，减少67%能耗以及68%数据搬运。

### 1 Background

文本、图像、音乐、视频等海量数据的产生以及深度学习的发展使得人们对于高性能存储和高性能计算的需求不断增加。存储方面，硬盘驱动器( HDD )的容量呈指数增长，可靠性高，成本极具竞争力。然而，HDD在性能上已趋于平稳，其功耗最终受到机械运动的限制。固态硬盘( Solid State Drives，SSD )基于NAND - Flash存储器，移动部件，在低功耗下具有更好的性能。在计算方面，通用图形处理单元( GPGPU )为运行深度学习和密码挖掘等计算密集型应用的传统PC带来了海量并行处理，但是GPU能耗高，不适合存储系统。数据中心现在最关心的不仅是电费，还包括冷却开销，此外将数据从存储端搬运到内存的开销也不可忽略。NLP任务的输入数据规模较大，存储端和主机端之间的数据移动在延迟和能耗上占据重要部分，为此存内计算（ISP）应运而生。为了支持ISP，存储系统需要一类称为计算存储驱动器( CSD )的新兴设备，这些设备通过处理能力来增加存储。下图所示，带有CSD的应用服务器中CSD可以处理部分请求从而提高吞吐量。本文作者提出的CSD原型Solana采用E1.s的形态，内置12-TB NAND flash SSD，嵌入式处理器以及SSD控制器。内置处理器运行Linux操作系统，允许用户运行Docker等基于Linux的应用。

![请添加图片描述](https://img-blog.csdnimg.cn/7630c3d75bb24c74a59ebc5a7b659f65.png)

通常有两种方式实现CSD，一种是使用SSD内部运行SSD控制器的低性能M系列或R系列ARM处理器来进行存内计算，第二种是使用与SSD控制器耦合的外部引擎运行存内计算任务，存储模块和ISP模块分离使得此种方式实现较为简单，但是需要考虑存储模块和计算模块时间的通信开销。外部ISP引擎可以采用混合的ARM A系列或者FPGA。

Solana采用ASIC设计，其中SSD控制器和运行通用应用程序的ARM处理器位于同一chip上。Solana的64位四核ARM处理器支持运行Linux操作系统，从而能够在不修改源代码或重新编译的情况下运行可执行文件。我们还实现了一个基于TCP / IP的隧道系统，允许存储内计算引擎连接到网络，包括互联网和其他处理节点。Solana原型如下图所示：

![请添加图片描述](https://img-blog.csdnimg.cn/fef9eca47a5a4938a8351ab262e22497.png)

### 2 Design

#### 2.1 CSD Hardware

<img src="https://img-blog.csdnimg.cn/29cad8dfa1c9477eb7e32e4068c32f50.png" alt="请添加图片描述" style="zoom:67%;" />

Solana通过NVMe over 4-lane PCIe与主机端相连。硬件架构主要包括两个子系统：flash控制单元（FCU）以及存内计算（ISP）引擎，两者通过片内高速数据总线共享6-GB的DRAM。

- **FCU子系统：**所提出的CSD解决方案应该能够充当具有NVMe接口的标准SSD，因此FCU子系统有许多与传统SSD类似的部分。FCU子系统包括前端（FE）和后端（BE）两部分。
  - FE前端：接受主机端的IO请求，检查完整性和正确性并解析，然后将指令传给BE执行。具有NVMe/PCIe的ASIC接口模块；
  - BE后端：需要和flash存储芯片通信，执行IO操作，ECC差错检测，负责实现闪存管理例程，如磨损均衡、地址翻译和垃圾回收。
- **In-Storage Processing子系统：**包括四核ARM Cortex A-53处理器以及NEON单指令多数据（SIMD）引擎；

#### 2.2 System Software on the CSD

在CSD上移植了一个完整版本的嵌入式Linux操作系统作为ISP引擎运行。运行一个传统的基于Linux的计算系统，使其能够兼容广泛的应用程序、编程语言和命令行而不需要修改。Linux也是可扩展的，有自己的自定义特性，包括设备驱动和文件系统。开发了一个定制块设备驱动程序( Customized Block Device Driver，CBDD )，使存储单元的访问优化到特定的片上通信链路/协议，这不同于处理单元和存储设备(例如PCIe和SATA)之间的通用协议。CBDD使用基于命令的机制与SSD控制器通信来发起数据传输。CBDD支持ISP应用程序和主机对文件系统的访问。

#### 2.3 Multi-Path Software Stack

![请添加图片描述](https://img-blog.csdnimg.cn/0034401b2fd04b20a4b2fc868490dc2e.png)

Solana支持三种不同的访问路径（上图中的a、b、c）

- Flash-to-Host：允许主机通过NVMe协议访问NAND Flash中存储的数据，NVMe控制器可以处理并响应NVMe指令，与flash控制器通信管理数据移动；

- Flash-to-ISP：为ISP引擎提供基于文件系统的数据访问。CBDD通过与闪存媒体控制器的通信实现了基于文件系统的数据访问接口。闪存媒体控制器负责处理来自ISP引擎和主机的请求。
- TCP/IP tunnelling over the PCIe/NVMe:主机(以及世界范围内的网络)与ISP系统的通信。两个用户级应用程序，一个在主机上，一个在后台运行的ISP上，负责管理基于NVMe的数据移动，并通过NVMe数据包封装/解封装TCP/IP协议族数据。

### 3 Evaluation

- 测试环境：
  - AIC FB128-LX equipped with an 8-core, 16-thread Intel® Xeon® Silver 4108 CPU running at 2.1 GHz and 64 GB of DRAM；
  - support up to 36 E1.S drives in the front bay, 12 TB each, for a total capacity of 432 TB on a 1U class server；

为了将处理任务发放到各个CSD，Solana还有一个任务调度器（scheduler），调度器运行在主机上的单独线程上，每0.2秒唤醒一次，检查是否有来自节点的新消息。调度器的两个重要参数是批大小（batch size），即一次分配给每个节点的数据块的大小，以及批比率（batch ratio），即主机和CSD的批大小之比。

- 性能测试结果：

  主要在Speech to text、Movie Recommender、以及Sentiment Analysis三个不同的NLP任务上进行测试，Baseline为只开起存储功能的CSD（作者称条件受限无法与其他CSD进行比较）。

![请添加图片描述](https://img-blog.csdnimg.cn/c7123593926e460b99d4bffd8d2d55b0.png)

<img src="https://img-blog.csdnimg.cn/14049550b1be44b48ce6802fe588190a.png" alt="请添加图片描述" style="zoom:67%;" />

在语音文字转换和电影推荐两个NLP任务上随着CSD数量增加，处理的查询（在语音转换任务中为每秒处理的单词数量）也随之增加，不同batch size对处理速度的影响不大。在情感分析任务中，系统性能也随着CSD数量增加增加，但是Batch size越大也会提升每秒查询数量，但是某些查询的延迟也会增加。由于输入是顺序输入到节点的，一旦一批查询被分配到一个处理节点，每个查询都要等待同一批中的先前查询完成，而不能迁移到其他一些空闲节点；在较小的批处理规模下，不同批次的查询可以分配给其他可用节点，因此总等待时间较少。

- 能耗测试：

  总的能耗包括主处理器，存储系统以及外设（如冷却系统），由于市场上没有可比较的具有12TB的容量的E1S驱动器，作者将相同服务器的ISP功能取消作为基线测试。在三个不同NLP任务中Solana对于能耗的减少都很显著且随CSD数量的增加效果越明显。

<img src="https://img-blog.csdnimg.cn/f46029eed35045568819ac8bead05ca3.png" alt="请添加图片描述" style="zoom:67%;" />

<img src="https://img-blog.csdnimg.cn/170a5d51dccd4772b793af88319dcc53.png" alt="请添加图片描述" style="zoom:67%;" />

### 4 Conclusion

本文提出了一个E1形态的12-TB CSD Solana，ISP引擎和SSD控制器在同一块芯片上集成，实验表明对于某些NLP任务提速可达3.1x，能耗和数据搬运可减少67%和68%。未来工作包括开发一个数据感知的分布式系统，通过将查询分类为分类组并将其重定向到关联节点，该系统可以从数据的时间局部性和数据的空间局部性中获益。