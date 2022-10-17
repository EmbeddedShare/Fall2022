# FLIN: Enabling Fairness and Enhancing Performance in Modern NVMe Solid State Drives
[原文链接](https://ieeexplore.ieee.org/document/8416843)
>A. Tavakkol et al., "FLIN: Enabling Fairness and Enhancing Performance in Modern NVMe Solid State Drives," 2018 ACM/IEEE 45th Annual International Symposium on Computer Architecture (ISCA), 2018, pp. 397-410, doi: 10.1109/ISCA.2018.00041.

随着现代高性能SSD的发展以及主机接口协议的发展（如NVMe），传统单队列的调度不再适合，由此衍生了MQ-SSD的发展，而MQ-SSD是直接分配到SSD中的，这将具有公平性机制的调度算法删除了。论文基于MQ-SSD造成不公平的现象的干扰源进行了研究，发现了四个主要的干扰源:(1)每个应用程序发送的请求的强度，(2)请求访问模式的差异，(3)读与写的比率，以及(4)垃圾收集。基于这四个干扰源，论文设计了一种新的公平调度-FLIN，对SSD中的TSU(transaction scheduling unit)进行了重新设计，提出三阶段调度，以一种显著减少四种干扰源的方式调度和重新排序事务，以便在并发运行的流之间提供公平性。FLIN完全在SSD控制器固件中实现，不需要新的硬件，存储成本可以忽略不(<0.06%)。实验结果显示出性能和公平性都得到了显著提升。
## 1.背景和动机
### 1.1 MQ-SSD
![在这里插入图片描述](https://img-blog.csdnimg.cn/2856ce08f3db414daea8be5794f0d208.png)

- 主机接口逻辑(HIL)，它实现用于与主机通信的协议，并以循环的方式从主机端请求队列中获取I/O请求;
-  flash转换层(FTL)，它管理SSD资源并使用嵌入式处理器和DRAM处理I/O请求;
- 闪存通道控制器(FCCs)，用于其他前端模块与后端芯片之间的通信
MQ-SSD的一个I/O请求的处理流程：
（1）	主机生成请求并将请求插入到主机端专用的dram内I/O队列中
（2）	MQ-SSD中的HIL从主机端队列获取请求，并将其插入SSD中的设备级队列中
（3）	HIL解析请求并将其分解为多个事务
（4）	FTL中的地址转换模块为每个事务执行从逻辑到物理的地址转换
（5）	FTL中的事务调度单元(TSU)对服务的每个事务进行调度，并将事务放入芯片级软件队列中
（6）	TSU从芯片级队列中选择一个事务，并将其分发给相应的FCC
（7）	FCC向目标后端芯片发出命令以执行事务
对于TSU，它负责调度I/O事务和GC事务。对于I/O事务，它为每个后端芯片在DRAM中维护一个读队列(图1中的RDQ)和一个写队列(WRQ)。在DRAM中，GC事务保存在一组独立的读(GC- rdq)和写(GC- wrq)队列中。TSU通常采用先到先得事务调度策略，并通常将I/O事务优先于GC事务。

### 1.2 SSD设备对公平控制的需求
当并发服务多个I/O流时，一个流的请求行为可能会对另一个流的请求产生负面影响。
一个流的slowdown计算如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/6f8294bd0d9546e9b91887e3c5f2a81f.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/bebab539527843ceba910129ec756eeb.png)

S越小越好。
将公平性(F)定义为系统中任何流所经历的最大和最小减速的比值:

![在这里插入图片描述](https://img-blog.csdnimg.cn/743e62c7eb154df09939544a607c6c90.png)

F越大越公平。
对MQ-SSD做了大量实验，下图是高强度的io流对一个中等强度的io流的影响，黑色是中等强度的io流，红色是高强度的io流。可以看到中等强度的io流slowdown明显减小了，且缺乏公平性

![在这里插入图片描述](https://img-blog.csdnimg.cn/25795d0c87ec47fea945afd61f7c4896.png)

### 1.3  对mq - ssd的不公平性干扰源的分析
论文基于实验给出了四种主要的干扰源：
 -  具有不同I/O强度的流
由于高强度流的干扰，低强度流的平均响应时间大幅增加。
>阶段性统计一个流的事务数，在单个周期的结尾计算到达率，若小于给定阈值，则定为低强度，反之高
 - 具有不同访问模式的流
具有并行友好访问模式的流容易受到不利用并行性的访问模式的流的干扰。
>因为不利用并行性的I/O流只将事务插入到少量的每个芯片队列中，这阻止了高度并行流的事务在同一时间完成
 - 具有不同读/写比率的流
一个具有更大比例写请求的流不公平地减慢了一个同时运行的具有更小比例写请求的流，即一个读密集的流。
>因为SSD的写通常比读慢10 - 40倍
 - 具有不同GC（垃圾回收）需求的流
当一个I/O流需要频繁的GC操作时，这些操作可以阻塞第二个只很少执行GC的流的I/O请求.

## 2. 设计
论文设计了一种新的MQ-SSD事务调度单元(TSU)，称为flash级干涉感知调度器(FLIN，Flash-Level INterference-aware scheduler)。
![在这里插入图片描述](https://img-blog.csdnimg.cn/b30a25be0b0c43bab40636c3c9e75608.png)

- 1.Fairness-Aware Queue Insertion
减轻由于并发运行流的强度和访问模式而产生的干扰。
每个优先级的流对应一个芯片的读写事务队列。
![算法流程](https://img-blog.csdnimg.cn/3b62c6c40bcb42a8b6669bf1a014a17d.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/6f73eb185c58444ebe82366043181e7c.png)

首先根据流的强度大致确定位置，然后再根据公平性调整具体的位置。
关于具体怎么确定位置参考论文5.1, 5.2, 5.3。
- 2.Priority-Aware Queue Arbitration
根据优先级队列，从准备好事务的（高）优先级读写队列取出请求进入下一阶段。
不止一个队列的事务准备好时，则利用WRR策略，从优先级i的队列选出2^i个事务到下一阶段。
- 3.Wait-Balancing Transaction Selection
最小化由于读写比率和并发运行流的垃圾收集需求而产生的干扰。
![在这里插入图片描述](https://img-blog.csdnimg.cn/94b75006e9d148b0b94cf678161db5f4.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/c2640766442a453d82790a00184e125c.png)

首先确定读写槽的proportional wait time ，然后谁大那么先将谁分配。
>读的优先级高那么直接分配，如果写的优先级高，则先得考虑垃圾回收。如果空闲页数量低于阈值，则执行垃圾回收，垃圾回收会伴随一对读写，因为回收的页可能有有效数据。一旦回收完成，立即分配写。

具体怎么计算参考论文5.4

## 3. 评估
### 3.1 实验配置
![在这里插入图片描述](https://img-blog.csdnimg.cn/12a3230cf37d4b70bab2f583904dbcaf.png)

![在这里插入图片描述](https://img-blog.csdnimg.cn/eff758ff89f342c8aec6d7869d7d0c5f.png)

具体参数配置见论文第6大节

### 3.2 实验结果
![在这里插入图片描述](https://img-blog.csdnimg.cn/6f7a8a1ed6b14135b5b6de9debdb5f63.png)

与最先进的设备级调度器相比，FLIN的公平性控制机制显著提高了weighted speedup（WS），这是MQ-SSD总体性能的一个重要指标。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7e0f71601bf447869b6a5b49c84a5bdf.png)

## 4. 总结
论文提出了FLIN，这是一种用于现代多队列ssd (mq - ssd)的轻量级事务调度器，它在并发运行的流之间提供公平性。FLIN使用三阶段设计极大程度上减少了实际mq - ssd中存在的所有四种主要干扰源的影响，同时执行由主机分配的应用程序级优先级。论文广泛的评估表明，与最先进的设备级调度器相比，FLIN有效地提高了公平性和系统性能。












